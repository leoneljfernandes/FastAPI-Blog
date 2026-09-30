# FastAPI Blog
> Proyecto: Blog fullstack con API REST + renderizado de templates Jinja2.

---

## Tabla de Contenidos
1. [Tecnologías y Arquitectura](#1-tecnologías-y-arquitectura)
2. [Análisis de Clases: Modelos vs Esquemas](#2-análisis-de-clases-modelos-vs-esquemas)
3. [Flujo de las Funcionalidades Principales](#3-flujo-de-las-funcionalidades-principales)
4. [Testing y Mocking](#4-testing-y-mocking)
5. [Flujo de un Request — Ejemplo de Entrevista](#5-flujo-de-un-request--ejemplo-de-entrevista)

---

## 1. Tecnologías y Arquitectura

### Stack completo y justificación

| Tecnología | Versión mínima | Rol en el proyecto |
|---|---|---|
| **FastAPI** | 0.141 | Framework web ASGI, genera la API REST y sirve las vistas Jinja2 |
| **Pydantic v2** | (incluida en FastAPI) | Validación de datos de entrada/salida + configuración |
| **SQLAlchemy 2.x** | 2.0.52 | ORM async para la capa de acceso a datos |
| **asyncpg / psycopg** | — | Drivers async de PostgreSQL |
| **PyJWT** | 2.13 | Generación y verificación de tokens JWT |
| **pwdlib[argon2]** | 0.3.1 | Hashing de contraseñas (Argon2, el algoritmo recomendado actualmente) |
| **Boto3** | 1.43 | SDK de AWS para interactuar con S3 |
| **Pillow** | 12.3 | Procesamiento de imágenes (resize, conversión de formato) |
| **aiosmtplib** | 5.1.2 | Cliente SMTP asíncrono para envío de emails |
| **Alembic** | 1.19 | Migraciones de base de datos |
| **Pytest + anyio** | 9.1 | Testing asíncrono |
| **Moto[s3]** | 5.2 | Mock de servicios AWS en tests |
| **pydantic-settings** | 2.15 | Gestión de configuración desde variables de entorno |
| **Starlette** | (incluida en FastAPI) | Manejo de excepciones HTTP, archivos estáticos, concurrencia |

---

### ¿Por qué esta arquitectura?

#### FastAPI como núcleo
FastAPI está construido sobre **Starlette** (para el ASGI y el manejo de requests) y **Pydantic** (para validación). Esto significa que tienes tres capas bien definidas:

```
Request HTTP → Starlette (routing/middleware) → FastAPI (inyección de dependencias, validación Pydantic) → Tu lógica → SQLAlchemy → PostgreSQL
```

El proyecto tiene un diseño **híbrido** inusual pero correcto: la misma aplicación sirve:
- **API REST** (`/api/users`, `/api/posts`) — responde JSON, usada por el frontend JavaScript.
- **Server-Side Rendering** (`/`, `/login`, `/account`) — responde HTML via Jinja2, para el navegador directamente.

Esto se controla con el manejador de excepciones inteligente en `main.py`:

```python
@app.exception_handler(StarletteHTTPException)
async def general_http_exception_handler(request: Request, exception: StarletteHTTPException):
    if request.url.path.startswith("/api"):
        return await http_exception_handler(request, exception)  # → JSON
    # De lo contrario → renderiza error.html (HTML)
    return templates.TemplateResponse(request, "error.html", {...})
```

#### Asincronía total (Asyncio)
Toda la stack es async de punta a punta:
- `create_async_engine` + `AsyncSession` de SQLAlchemy → las queries a la BD no bloquean el event loop.
- `aiosmtplib` → los emails se envían sin bloquear.
- `BackgroundTasks` de FastAPI → el envío de emails ocurre **después** de que se retorna la respuesta HTTP al cliente.

La excepción deliberada es `process_profile_image` (Pillow), que **es síncrona** porque hace CPU-bound work. Se envuelve correctamente con:

```python
# En users.py:
processed_bytes, new_filename = await run_in_threadpool(process_profile_image, content)
```

`run_in_threadpool` (de Starlette) ejecuta la función en un thread pool separado, evitando que el procesamiento de imagen bloquee el event loop. Este es un patrón fundamental en FastAPI.

#### Pydantic Settings para configuración
En lugar de leer `os.environ` manualmente, `config.py` usa `pydantic-settings`:

```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env")
    secret_key: SecretStr  # ← Tipo especial: no se imprime en logs accidentalmente
    database_url: str
    s3_bucket_name: str
```

**Ventajas clave**:
1. Validación automática al arrancar: si falta una variable requerida, la app falla inmediatamente (fail-fast).
2. `SecretStr` previene que secrets aparezcan en logs o trazas de error.
3. Un único objeto `settings` importado en todo el proyecto — consistencia.

---

## 2. Análisis de Clases: Modelos vs Esquemas

> [!IMPORTANT]
> Esta es probablemente **la pregunta más frecuente en entrevistas de FastAPI**. La distinción entre modelos ORM y esquemas Pydantic es fundamental.

### La regla de oro

| | `models.py` (SQLAlchemy ORM) | `schemas.py` (Pydantic) |
|---|---|---|
| **Representa** | Una tabla en la base de datos | Un contrato de datos (entrada o salida de la API) |
| **Vive en** | La capa de persistencia | La capa de presentación/validación |
| **Habla con** | La base de datos (SQL) | El cliente HTTP (JSON) |
| **Valida datos** | No (confía en que ya llegaron validados) | Sí, es su propósito principal |
| **Expone info sensible** | Contiene `password_hash` | Nunca expone `password_hash` |

### `models.py` — Análisis detallado

#### `class User`
```python
class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, index=True)
    username: Mapped[str] = mapped_column(String(50), unique=True, nullable=False)
    email: Mapped[str] = mapped_column(String(120), unique=True, nullable=False)
    password_hash: Mapped[str] = mapped_column(String(255), nullable=False)
    image_file: Mapped[str | None] = mapped_column(String(200), nullable=True, default=None)

    posts: Mapped[list[Post]] = relationship(back_populates="author", cascade='all, delete-orphan')
    reset_tokens: Mapped[list[PasswordResetToken]] = relationship(back_populates="user", cascade="all, delete-orphan")
```

**Puntos clave para entrevista:**
- `Mapped[str | None]` usa la sintaxis moderna de SQLAlchemy 2.x con type hints. El tipo `Python` y la columna SQL están alineados.
- `cascade='all, delete-orphan'` en `posts`: si borras un `User`, todos sus `Post` se borran automáticamente. Esto evita registros huérfanos.
- **`@property image_path`**: este es un patrón muy elegante. La columna `image_file` guarda solo el nombre del archivo (ej: `"abc123.jpg"`). La propiedad reconstruye la URL completa de S3 en tiempo de ejecución, sin persistirla en la BD:

```python
@property
def image_path(self) -> str:
    if self.image_file:
        return f"https://{settings.s3_bucket_name}.s3.{settings.s3_region}.amazonaws.com/profile_pics/{self.image_file}"
    return "/static/profile_pics/default.jpg"
```

Esto es importante porque si cambias el bucket de S3, solo cambias la configuración, no los datos en la base de datos.

#### `class Post`
```python
class Post(Base):
    __tablename__ = "posts"

    id, title, content, user_id, date_posted, likes ...
    author: Mapped[User] = relationship(back_populates="posts")
```

- `user_id` es la **clave foránea** (FK) — la columna real en la tabla SQL.
- `author` es la **relación ORM** — no existe como columna, es un objeto `User` cargado por SQLAlchemy. Cuando haces `post.author.username` en Python, SQLAlchemy genera un `JOIN` o un `SELECT` adicional.
- `date_posted` usa `lambda: datetime.now(UTC)` como `default`. Importante: es una función lambda, no `datetime.now(UTC)` directamente (que se evaluaría una sola vez al definir la clase y todos los posts tendrían la misma fecha).

#### `class PasswordResetToken`
Tabla separada para tokens de restablecimiento de contraseña.
- `token_hash`: **nunca se guarda el token en texto plano**, solo su hash SHA-256. Si alguien roba la base de datos, no puede usar los tokens.
- `expires_at`: la caducidad se verifica explícitamente en la lógica del router, no solo a nivel de BD.

---

### `schemas.py` — Análisis detallado

#### Jerarquía de clases de Usuario

```
UserBase (username, email, image_file)
├── UserCreate (+ password)          ← Entrada: POST /api/users
└── UserPublic (+ id, image_path)    ← Salida pública (sin email)
    └── UserPrivate (+ email)        ← Salida para el propio usuario
```

**¿Por qué esta jerarquía?**

- `UserCreate` tiene `password` (en texto plano) porque el usuario lo envía. Esta clase **nunca se retorna** como respuesta.
- `UserPublic` se usa cuando queremos mostrar al autor de un post a cualquier visitante. No expone el email.
- `UserPrivate` hereda `UserPublic` y agrega el `email`. Se usa en `GET /me` y en acciones del propio usuario.
- **`password_hash` no existe en ningún schema de salida** — si no está en el schema, no puede filtrarse accidentalmente en la respuesta JSON.

#### `ConfigDict(from_attributes=True)`
```python
class UserPublic(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    image_path: str  # ← Esta es una @property del modelo ORM
```

Sin `from_attributes=True`, Pydantic solo puede construirse desde diccionarios. Con esa config, puede leer atributos directamente de un objeto ORM (incluyendo `@property`). Esto es lo que permite `PostResponse.model_validate(post)` donde `post` es un objeto SQLAlchemy.

#### Jerarquía de clases de Post

```
PostBase (title, content)
├── PostCreate (sin cambios)          ← Entrada: POST /api/posts
└── PostUpdate (ambos opcionales)     ← Entrada: PATCH /api/posts/{id}
    PostResponse (+ id, user_id, date_posted, author: UserPublic) ← Salida
        └── PaginatedPostResponse    ← Salida con metadatos de paginación
```

El uso de `PostUpdate` con campos opcionales (`str | None = Field(default=None, ...)`) es el patrón correcto para `PATCH`. Permite al cliente enviar solo los campos que quiere actualizar:

```python
# En posts.py:
update_data = post_data.model_dump(exclude_unset=True)  # Solo los campos enviados
for field, value in update_data.items():
    setattr(post, field, value)  # Aplica solo los cambios
```

`model_dump(exclude_unset=True)` es la clave: si el cliente manda `{"title": "Nuevo"}`, `exclude_unset=True` solo incluye `title` en el dict, dejando `content` sin tocar.

---

## 3. Flujo de las Funcionalidades Principales

### 3.1 Autenticación: Registro y JWT

#### Registro (`POST /api/users`)

```
Cliente → POST /api/users con { username, email, password }
    ↓
FastAPI valida con UserCreate (Pydantic)
    ↓
create_user() en users.py
    ↓
Verifica duplicados (username y email) — case-insensitive con func.lower()
    ↓
hash_password(user.password) → password_hash.hash(password) [Argon2 via pwdlib]
    ↓
models.User(username=..., email=email.lower(), password_hash=...)
    ↓
db.add(new_user) → await db.commit() → await db.refresh(new_user)
    ↓
Retorna UserPrivate (nunca el password_hash)
```

**¿Por qué Argon2?** Es el algoritmo ganador del Password Hashing Competition (2015). Es resistente a ataques de GPU y FPGA gracias a su diseño memory-hard. `pwdlib.PasswordHash.recommended()` selecciona automáticamente el mejor algoritmo disponible.

#### Login y generación de JWT (`POST /api/users/token`)

```python
# users.py
@router.post("/token", response_model=Token)
async def login_for_access_token(form_data: OAuth2PasswordRequestForm, ...):
    # 1. Busca usuario por email (el campo "username" del form OAuth2 se usa como email)
    result = await db.execute(select(models.User).where(
        func.lower(models.User.email) == form_data.username.lower()
    ))
    user = result.scalars().first()

    # 2. Verifica existencia Y contraseña en una sola condición (evita timing attacks)
    if not user or not verify_password(form_data.password, user.password_hash):
        raise HTTPException(status_code=401, detail="Incorrect email or password")

    # 3. Crea el token con el ID del usuario como "subject"
    access_token = create_access_token(data={"sub": str(user.id)})
    return Token(access_token=access_token, token_type="bearer")
```

**Por qué `"sub": str(user.id)`**: el estándar JWT usa el claim `sub` (subject) para identificar al sujeto. Se usa el ID (entero) en vez del username o email porque es inmutable.

#### Estructura del JWT

```python
# auth.py
def create_access_token(data: dict, expires_delta: timedelta | None = None) -> str:
    to_encode = data.copy()
    expire = datetime.now(UTC) + timedelta(minutes=settings.access_token_expire_minutes)
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, settings.secret_key.get_secret_value(), algorithm="HS256")
```

El token JWT tiene esta estructura (decodificada):
```json
Header: { "alg": "HS256", "typ": "JWT" }
Payload: { "sub": "42", "exp": 1720000000 }
Signature: HMACSHA256(base64(header) + "." + base64(payload), secret_key)
```

#### Validación del JWT en cada request protegido

```python
# auth.py
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="api/users/token")

async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],  # Extrae el Bearer token del header
    db: Annotated[AsyncSession, Depends(get_db)],
) -> models.User:
    user_id = verify_access_token(token)  # Decodifica y valida firma + expiración
    if user_id is None:
        raise HTTPException(status_code=401, ...)
    
    # Busca el usuario en la BD para confirmar que aún existe
    user = await db.execute(select(models.User).where(models.User.id == int(user_id)))
    return user.scalars().first()

# Alias conveniente, declarado al final del archivo:
CurrentUser = Annotated[models.User, Depends(get_current_user)]
```

`CurrentUser` es un tipo alias de `Annotated`. Cuando un endpoint declara `current_user: CurrentUser`, FastAPI ejecuta automáticamente toda la cadena: extrae el header → decodifica JWT → consulta la BD → retorna el objeto `User`. Es la **inyección de dependencias** de FastAPI en acción.

---

### 3.2 Gestión de Posts: CRUD y Paginación

#### Paginación con cursor offset

```python
# posts.py
@router.get("", response_model=PaginatedPostResponse)
async def get_posts(
    db: ...,
    skip: Annotated[int, Query(ge=0)] = 0,       # offset
    limit: Annotated[int, Query(ge=1, le=100)] = 20,  # max 100 por página
):
    # Query 1: Cuenta total (sin LIMIT ni OFFSET)
    count_result = await db.execute(select(func.count()).select_from(models.Post))
    total = count_result.scalar() or 0

    # Query 2: Los posts de la página actual
    result = await db.execute(
        select(models.Post)
        .options(selectinload(models.Post.author))  # Eager loading del autor
        .order_by(models.Post.date_posted.desc())
        .offset(skip)
        .limit(limit)
    )
    posts = result.scalars().all()

    has_more = skip + len(posts) < total  # ¿Hay más páginas?
```

**`selectinload` vs lazy loading**: por defecto, SQLAlchemy en modo async no puede hacer lazy loading (cargaría el autor en un contexto fuera de la sesión). `selectinload(models.Post.author)` hace un segundo `SELECT` eficiente trayendo todos los autores necesarios **dentro de la misma sesión**. Si no lo usaras, acceder a `post.author` fuera de la sesión lanzaría un `MissingGreenlet` error.

#### CREATE — `POST /api/posts`

```python
async def create_post(post: PostCreate, current_user: CurrentUser, db: ...):
    new_post = models.Post(
        title=post.title,
        content=post.content,
        user_id=current_user.id,  # El usuario viene del JWT, no del request body
    )
    db.add(new_post)
    await db.commit()
    await db.refresh(new_post, attribute_names=["author"])  # Recarga el autor
    return new_post
```

`db.refresh(new_post, attribute_names=["author"])` es necesario porque después del `commit`, el objeto `new_post` está "stale" (desactualizado). `refresh` recarga desde la BD los atributos especificados, incluyendo la relación `author`.

#### UPDATE completo vs parcial (PUT vs PATCH)

| Método | Schema usado | Comportamiento |
|---|---|---|
| `PUT /{post_id}` | `PostCreate` (todos obligatorios) | Reemplaza título y contenido completos |
| `PATCH /{post_id}` | `PostUpdate` (todos opcionales) | Solo actualiza los campos enviados |

```python
# PATCH — el patrón correcto
update_data = post_data.model_dump(exclude_unset=True)  # Solo los enviados
for field, value in update_data.items():
    setattr(post, field, value)  # Aplica dinámicamente
```

En ambos casos, antes de modificar se verifica autorización:
```python
if post.user_id != current_user.id:
    raise HTTPException(status_code=403, detail="Not authorized")
```

---

### 3.3 Subida de Imágenes: Pillow + Boto3 + S3

El flujo está dividido en tres capas con responsabilidades claras:

```
[Router: users.py]                 [image_utils.py]            [AWS S3]
upload_profile_picture()
    ↓ Lee bytes del UploadFile
    ↓ Valida tamaño (< 5MB)
    ↓ run_in_threadpool(process_profile_image, content)
                                    process_profile_image()
                                    ├── Image.open(BytesIO(content))
                                    ├── ImageOps.exif_transpose()  ← Corrige rotación
                                    ├── ImageOps.fit(300x300, LANCZOS)  ← Recorta y redimensiona
                                    ├── img.convert("RGB")  ← Si era RGBA/PNG
                                    └── Retorna (bytes_jpeg, "uuid4.jpg")
    ↓ await upload_profile_image(processed_bytes, new_filename)
                                    _upload_to_s3()
                                    ├── _get_s3_client()  ← Crea cliente Boto3
                                    └── s3.upload_fileobj(BytesIO, bucket, "profile_pics/uuid.jpg")
                                                                        ↑ Guardado en S3
    ↓ Guarda new_filename en user.image_file (BD)
    ↓ Borra imagen anterior si existía (await delete_profile_image(old_filename))
    ↓ Retorna UserPrivate con la nueva image_path
```

**Detalles importantes:**

- **`_get_s3_client()`**: crea el cliente Boto3 en cada llamada. Si `s3_endpoint_url` está configurado, lo usa (permite usar LocalStack o Moto). Si no hay credenciales explícitas (`None`), Boto3 usa las credenciales del entorno (IAM roles, `~/.aws/credentials`).

- **`ImageOps.exif_transpose`**: las fotos tomadas con celulares tienen metadatos EXIF que indican la orientación real. Sin este paso, las fotos "giradas" se subirían mal.

- **`ImageOps.fit` vs `resize`**: `fit` recorta Y redimensiona para llenar exactamente 300x300 sin deformar la imagen. `resize` simplemente escala (puede deformar).

- **`run_in_threadpool`**: como Pillow es síncrono y hace trabajo CPU-intensivo, se corre en un thread pool. Si se llamara directamente `await process_profile_image(...)`, bloquearía el event loop de asyncio.

- **`qualiti=85`**: hay un typo en el código real (`qualiti` en vez de `quality`). Pillow lo ignora silenciosamente y usa su valor por defecto. Esto **no produce un error**, pero la optimización de calidad no se aplica. Es un buen punto a mencionar en una entrevista como una mejora potencial.

---

## 4. Testing y Mocking

### Arquitectura del suite de tests

#### El problema fundamental: tests async + BD + S3

Los tests necesitan:
1. Una BD de test aislada (para no contaminar producción).
2. Que las queries async funcionen con pytest.
3. Que cada test empiece con una BD limpia.
4. Que Boto3/S3 no haga requests reales a AWS.

#### Solución: `conftest.py` en detalle

##### Paso 1 — Variables de entorno ANTES de importar la app

```python
# conftest.py — líneas 4-16
os.environ["DATABASE_URL"] = "postgresql+psycopg://bloguser:blogpass@localhost/test_blog"
os.environ["S3_BUCKET_NAME"] = "test-bucket"
os.environ["SECRET_KEY"] = "test-secret-key-for-testing-only"
# ...

# DESPUÉS de setear envs:
from ..main import app
from ..database import Base, get_db
```

Esto es **crítico**. `pydantic-settings` lee las variables al importar el módulo `config.py`. Si importaras la app antes de setear las variables, usaría los valores del `.env` de producción.

##### Paso 2 — Engine de tests con `NullPool`

```python
@pytest.fixture(scope="session")
def test_engine():
    engine = create_async_engine(
        os.environ["DATABASE_URL"],
        poolclass=NullPool  # ← Sin pool de conexiones
    )
    yield engine
```

`NullPool` crea una conexión nueva para cada operación y la cierra inmediatamente. Es menos eficiente, pero garantiza que no haya conexiones "prestadas" que interfieran entre tests.

##### Paso 3 — Base de datos creada una vez por sesión

```python
@pytest.fixture(scope="session")
async def setup_database(test_engine):
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)  # Crea todas las tablas
    yield
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)   # Las borra al final
    await test_engine.dispose()
```

`scope="session"` significa que las tablas se crean **una sola vez** para todos los tests de la sesión.

##### Paso 4 — Aislamiento por test con SAVEPOINT (el patrón más importante)

```python
@pytest.fixture
async def db_session(test_engine, setup_database) -> AsyncGenerator[AsyncSession]:
    conn = await test_engine.connect()
    trans = await conn.begin()  # Abre una transacción "externa"

    test_async_session = async_sessionmaker(
        bind=conn,
        class_=AsyncSession,
        expire_on_commit=False,
        join_transaction_mode="create_savepoint",  # ← La clave
    )

    async with test_async_session() as session:
        yield session  # El test usa esta sesión

    # Teardown: SIEMPRE hace rollback, sin importar si el test pasó
    await trans.rollback()  # ← Deshace TODO lo que hizo el test
    await conn.close()
```

**`join_transaction_mode="create_savepoint"`** es el corazón del sistema:
- Cuando el código de la app llama a `await db.commit()`, en lugar de hacer un `COMMIT` real, SQLAlchemy crea un **SAVEPOINT** (un punto de guardado dentro de la transacción).
- Al final del test, la transacción externa hace `ROLLBACK`, deshaciendo todo, **incluyendo todos los "commits"** que hizo el código bajo prueba.
- Resultado: cada test empieza con una BD perfectamente limpia, sin necesidad de borrar tablas entre tests.

##### Paso 5 — Mock de AWS con Moto

```python
@pytest.fixture
def mocked_aws():
    with mock_aws():  # Activa el mock de Moto (intercepta todas las llamadas Boto3)
        s3 = boto3.client("s3", region_name="us-east-1")
        s3.create_bucket(Bucket=os.environ["S3_BUCKET_NAME"])  # Crea bucket virtual
        yield s3  # El test recibe el cliente para hacer aserciones
```

`mock_aws()` de Moto intercepta **todas** las llamadas al SDK de Boto3 mientras está activo. No sale ningún request a internet. Todo se simula en memoria.

##### Paso 6 — Override de dependencias en el cliente HTTP

```python
@pytest.fixture
async def client(db_session: AsyncSession, mocked_aws) -> AsyncGenerator[AsyncClient]:
    async def override_get_db():
        yield db_session  # Reemplaza la BD real con la de tests

    app.dependency_overrides[get_db] = override_get_db  # ← Override de FastAPI

    async with AsyncClient(
        transport=ASGITransport(app=app),  # Cliente en proceso (sin HTTP real)
        base_url="http://test",
    ) as ac:
        yield ac

    app.dependency_overrides.clear()  # Limpia los overrides
```

`app.dependency_overrides` es el mecanismo de FastAPI para reemplazar dependencias en tests. Cuando el código de la app llama a `Depends(get_db)`, en los tests usa `override_get_db`, que retorna la sesión de test con el SAVEPOINT.

`ASGITransport` hace que `httpx` comunique directamente con la app FastAPI en memoria, sin abrir un puerto TCP real. Los tests son mucho más rápidos y estables.

---

### Ejemplo: `test_upload_profile_picture`

```python
@pytest.mark.anyio
async def test_upload_profile_picture(client: AsyncClient, mocked_aws):
    user = await create_test_user(client)
    token = await login_user(client)

    image_bytes = Path(__file__).parent / "test_image.jpg").read_bytes()

    response = await client.patch(
        f"/api/users/{user['id']}/picture",
        files={"file": ("profile.jpg", BytesIO(image_bytes), "image/jpeg")},
        headers=auth_header(token),
    )

    assert response.status_code == 200
    assert response.json()["image_file"].endswith(".jpg")
    assert "s3" in response.json()["image_path"]

    # Verifica directamente en el S3 mockeado que el archivo existe
    s3_objects = mocked_aws.list_objects_v2(Bucket="test-bucket")
    assert len(s3_objects["Contents"]) == 1
```

Este test verifica todo el flujo de extremo a extremo: autenticación, procesamiento de imagen, subida a S3 (mockeado) y actualización de la BD.

---

## 5. Flujo de un Request — Ejemplo de Entrevista

> **Pregunta típica**: "Explícame qué pasa exactamente cuando un usuario autenticado hace `POST /api/posts` con `{title: 'Mi Post', content: 'Hola mundo'}`."

### Recorrido completo

```
[1] Cliente HTTP
    POST /api/posts
    Headers: { Authorization: "Bearer eyJhbGci..." }
    Body: { "title": "Mi Post", "content": "Hola mundo" }
```

---

#### Fase 1: Routing en `main.py`

```python
# main.py
app.include_router(posts.router, prefix="/api/posts", tags=["posts"])
```

FastAPI recibe el request y, basándose en el método `POST` y la ruta `/api/posts`, lo delega al router de posts. No hay middleware de auth a nivel de app; la autenticación se hace por dependencia en el endpoint.

---

#### Fase 2: Resolución de dependencias — `posts.py`

```python
# posts.py
@router.post("", response_model=PostResponse, status_code=201)
async def create_post(
    post: PostCreate,        # ← Dependencia implícita: cuerpo JSON validado por Pydantic
    current_user: CurrentUser,  # ← Dependencia declarada: requiere autenticación
    db: Annotated[AsyncSession, Depends(get_db)],  # ← Dependencia: sesión de BD
):
```

FastAPI resuelve las dependencias en este orden:

**`post: PostCreate`** — FastAPI lee el body JSON y lo valida contra `PostCreate`:
```python
class PostCreate(PostBase):  # PostBase tiene title y content
    pass
# Validaciones: title min_length=1 max_length=100, content min_length=1 max_length=5000
```
Si el JSON falla la validación → **422 Unprocessable Entity** inmediatamente.

**`db: Depends(get_db)`** — Ejecuta el generador:
```python
async def get_db():
    async with AsyncSessionLocal() as session:
        yield session  # Abre sesión, la "pausa" y la cierra al terminar el request
```

**`current_user: CurrentUser`** — Esto es `Annotated[models.User, Depends(get_current_user)]`. FastAPI ejecuta `get_current_user`:

```python
async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],  # Extrae el Bearer token
    db: Annotated[AsyncSession, Depends(get_db)],
) -> models.User:
```

---

#### Fase 3: Autenticación — `auth.py`

**`oauth2_scheme`** extrae el token del header `Authorization: Bearer <token>`. Si no hay header → **401 Unauthorized**.

**`verify_access_token(token)`**:
```python
def verify_access_token(token: str) -> str | None:
    payload = jwt.decode(
        token,
        settings.secret_key.get_secret_value(),
        algorithms=["HS256"],
        options={"require": ["exp", "sub"]}  # Exige que existan estos claims
    )
    return payload.get("sub")  # Retorna "42" (el user id como string)
```

Si el token está expirado, tiene firma inválida, o le faltan claims → retorna `None` → **401**.

**Consulta a la BD** para confirmar que el usuario existe:
```python
result = await db.execute(select(models.User).where(models.User.id == 42))
user = result.scalars().first()
```

Si el usuario fue borrado después de emitir el token → **401**.

---

#### Fase 4: Lógica del endpoint — de vuelta en `posts.py`

Con `post` (validado por Pydantic) y `current_user` (objeto User autenticado):

```python
new_post = models.Post(
    title=post.title,       # "Mi Post"
    content=post.content,   # "Hola mundo"
    user_id=current_user.id,  # 42 — el ID viene del JWT, no del cliente
)
db.add(new_post)          # Registra el objeto en la sesión (aún no va a la BD)
await db.commit()         # SQL: INSERT INTO posts (title, content, user_id) VALUES (...)
                          #      La BD asigna el id y date_posted automáticamente
await db.refresh(new_post, attribute_names=["author"])
# SQL: SELECT * FROM users WHERE id = 42
# Carga el objeto User como new_post.author (necesario para PostResponse)
```

---

#### Fase 5: Serialización de la respuesta

FastAPI toma el objeto `new_post` y lo convierte a JSON usando `PostResponse`:

```python
class PostResponse(PostBase):
    model_config = ConfigDict(from_attributes=True)
    id: int
    user_id: int
    date_posted: datetime
    author: UserPublic  # ← Se serializa el autor también (sin email)
```

`from_attributes=True` permite que Pydantic lea los atributos del objeto ORM incluyendo `author.image_path` (que es un `@property`).

**Respuesta final:**
```json
HTTP 201 Created
{
  "id": 1,
  "title": "Mi Post",
  "content": "Hola mundo",
  "user_id": 42,
  "date_posted": "2026-09-11T09:32:00Z",
  "author": {
    "id": 42,
    "username": "leonel",
    "image_file": null,
    "image_path": "/static/profile_pics/default.jpg"
  }
}
```

Notar que `password_hash` **no aparece en ningún lugar** de la respuesta, aunque el objeto `User` en memoria sí lo tiene. El schema actúa como un filtro de seguridad perfecto.

---

### Diagrama de capas del request completo

```
Request POST /api/posts
    │
    ├─[Starlette] → Routing: /api/posts → posts.router
    │
    ├─[FastAPI DI] → Resuelve dependencias en paralelo donde es posible:
    │   ├─ PostCreate  ← valida body JSON (Pydantic)
    │   ├─ get_db()    ← abre AsyncSession
    │   └─ get_current_user()
    │       ├─ oauth2_scheme ← extrae Bearer token del header
    │       ├─ verify_access_token() ← decodifica JWT (PyJWT)
    │       └─ SELECT * FROM users WHERE id=... (SQLAlchemy async)
    │
    ├─[Endpoint] create_post()
    │   ├─ models.Post(...) ← crea objeto ORM
    │   ├─ db.add() + await db.commit() → INSERT SQL (asyncpg)
    │   └─ await db.refresh() → SELECT SQL (carga autor)
    │
    └─[FastAPI] → Serializa con PostResponse (Pydantic)
        └─ Respuesta JSON 201 Created
```

---

> [!TIP]
> **Puntos extra para mencionar en entrevista:**
> - El `user_id` siempre se toma del JWT validado, **nunca del body**. Esto impide que un usuario cree posts a nombre de otro.
> - La sesión de BD se maneja con `yield` (context manager). Si el endpoint lanza una excepción, la sesión igualmente se cierra limpiamente.
> - `expire_on_commit=False` en `AsyncSessionLocal` evita que los atributos del objeto queden "expirados" (y lancen un lazy-load error) después del commit.
