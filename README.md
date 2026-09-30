# FastAPI Blog

> Blog fullstack: API REST + renderizado server-side con Jinja2, autenticación JWT, imágenes de perfil en AWS S3 y recuperación de contraseña por email.

![Python](https://img.shields.io/badge/python-3.14-blue)
![FastAPI](https://img.shields.io/badge/FastAPI-0.141-009688)
![SQLAlchemy](https://img.shields.io/badge/SQLAlchemy-2.0_async-red)
![Tests](https://img.shields.io/badge/tests-pytest-informational)

> 🚧 **Próximamente:** integración continua con **CircleCI** y reporte de cobertura con **Coveralls** (ver [Roadmap](#roadmap)).

---

## Tabla de Contenidos
1. [Descripción del proyecto](#1-descripción-del-proyecto)
2. [Tecnologías](#2-tecnologías)
3. [Estructura del proyecto](#3-estructura-del-proyecto)
4. [Clases: modelos y esquemas](#4-clases-modelos-y-esquemas)
5. [Cómo funciona](#5-cómo-funciona)
6. [Endpoints](#6-endpoints)
7. [Instalación y ejecución](#7-instalación-y-ejecución)
8. [Variables de entorno](#8-variables-de-entorno)
9. [Tests](#9-tests)
10. [Roadmap](#roadmap)

---

## 1. Descripción del proyecto

**FastAPI Blog** es una aplicación de blogging construida con un diseño **híbrido**: la misma app sirve

- una **API REST** bajo `/api/...` que responde JSON (consumida por el JavaScript del frontend), y
- **páginas HTML** renderizadas en el servidor con Jinja2 (`/`, `/login`, `/register`, `/account`, ...).

Funcionalidades principales:

- Registro, login y autenticación con **JWT** (contraseñas hasheadas con **Argon2**).
- CRUD de **posts** con paginación (`PUT` completo y `PATCH` parcial).
- Perfil de usuario con **foto de perfil**: se recorta a 300×300, se convierte a JPEG y se sube a **S3**.
- **Recuperación de contraseña** por email con token de un solo uso (solo se guarda su hash SHA-256).
- Cambio de contraseña, edición y borrado de cuenta.
- Endpoint de **health check** (`/health`) que verifica la conexión a la BD.
- Migraciones con **Alembic** y script para poblar la BD con datos de ejemplo.
- Suite de tests asíncronos con BD aislada por transacción y S3 mockeado.

> Para una explicación profunda de las decisiones de diseño (flujo de un request, SAVEPOINTs en tests, etc.) ver [documentacion_tecnica.md](documentacion_tecnica.md).

---

## 2. Tecnologías

| Tecnología | Versión mínima | Rol |
|---|---|---|
| **Python** | 3.14 | Lenguaje |
| **FastAPI** (`[standard]`) | 0.141.1 | Framework web ASGI, API REST y vistas |
| **Pydantic v2 / pydantic-settings** | 2.15 | Validación de datos y configuración por entorno |
| **SQLAlchemy (async)** | 2.0.52 | ORM |
| **PostgreSQL** (`asyncpg`, `psycopg`) | — | Base de datos principal (`aiosqlite` disponible para desarrollo) |
| **Alembic** | 1.19.2 | Migraciones |
| **PyJWT** | 2.13 | Tokens JWT |
| **pwdlib[argon2]** | 0.3.1 | Hashing de contraseñas |
| **Boto3** | 1.43 | Cliente de AWS S3 |
| **Pillow** | 12.3 | Procesamiento de imágenes |
| **aiosmtplib** | 5.1.2 | Envío asíncrono de emails |
| **Jinja2** | (incluida en `fastapi[standard]`) | Templates HTML |
| **Pytest + httpx + Moto[s3]** | 9.1 / — / 5.2 | Testing y mock de AWS |
| **uv** | — | Gestor de paquetes y entornos |

---

## 3. Estructura del proyecto

```
Curso_FastApi/
├── pyproject.toml              # Dependencias y metadata (gestionado con uv)
├── uv.lock
├── .python-version             # 3.14
├── .env                        # Variables de entorno (NO se versiona)
├── documentacion_tecnica.md    # Documentación técnica en profundidad
├── README.md
│
├── src/
│   └── fastapi_blog/
│       ├── main.py             # App FastAPI, rutas HTML, manejadores de excepciones
│       ├── config.py           # Settings (pydantic-settings)
│       ├── database.py         # Engine async, sesión y get_db()
│       ├── models.py           # Modelos ORM: User, Post, PasswordResetToken
│       ├── schemas.py          # Esquemas Pydantic de entrada/salida
│       ├── auth.py             # Hashing, JWT, get_current_user / CurrentUser
│       ├── email_utils.py      # Envío de emails (reset de contraseña)
│       ├── image_utils.py      # Procesado con Pillow y subida/borrado en S3
│       ├── populate_db.py      # Carga de datos de ejemplo
│       ├── check_s3.py         # Utilidad para verificar el acceso a S3
│       ├── aws_iam_policy.json / aws_bucket_policy.json
│       ├── alembic.ini
│       ├── alembic/            # Migraciones (initial_schema, add_likes_to_posts)
│       ├── routers/
│       │   ├── users.py        # /api/users/...
│       │   └── posts.py        # /api/posts/...
│       └── test/
│           ├── conftest.py     # Fixtures: BD, cliente HTTP, mock de AWS
│           ├── test_users_.py
│           └── test_posts.py
│
├── templates/                  # Jinja2: layout, home, post, login, register,
│   └── email/                  #   account, forgot/reset password, error...
├── static/                     # css, js (auth.js, utils.js), iconos, foto por defecto
└── media/                      # Imágenes de perfil locales (legado)
```

---

## 4. Clases: modelos y esquemas

El proyecto separa estrictamente la **capa de persistencia** (SQLAlchemy) de la **capa de contrato con el cliente** (Pydantic).

### Modelos ORM — [models.py](src/fastapi_blog/models.py)

| Clase | Tabla | Descripción |
|---|---|---|
| `User` | `users` | `id`, `username` (único), `email` (único), `password_hash`, `image_file`. Relaciones a `posts` y `reset_tokens` con `cascade="all, delete-orphan"`. La propiedad `image_path` construye la URL completa de S3 (o la imagen por defecto). |
| `Post` | `posts` | `id`, `title`, `content`, `user_id` (FK), `date_posted` (UTC), `likes`. Relación `author` hacia `User`. |
| `PasswordResetToken` | `password_reset_tokens` | `user_id` (FK), `token_hash` (SHA-256, único), `expires_at`, `created_at`. Nunca se guarda el token en claro. |

```
User 1 ──── * Post
User 1 ──── * PasswordResetToken
```

### Esquemas Pydantic — [schemas.py](src/fastapi_blog/schemas.py)

```
UserBase (username, email, image_file)
├── UserCreate (+ password)            ← entrada: POST /api/users
UserPublic (id, username, image_file, image_path)   ← salida pública (sin email)
└── UserPrivate (+ email)              ← salida para el propio usuario
UserUpdate                             ← entrada: PATCH /api/users/{id}
Token                                  ← salida: login

PostBase (title, content)
├── PostCreate                         ← entrada: POST / PUT
PostUpdate (campos opcionales)         ← entrada: PATCH
PostResponse (+ id, user_id, date_posted, author: UserPublic)
└── PaginatedPostResponse              ← posts + total + skip + limit + has_more

ForgotPasswordRequest · ResetPasswordRequest · ChangePasswordRequest
```

`password_hash` **no existe en ningún esquema de salida**, por lo que no puede filtrarse en una respuesta JSON.

---

## 5. Cómo funciona

### Arquitectura general

```
Request HTTP
   → Starlette (routing / estáticos)
   → FastAPI (inyección de dependencias + validación Pydantic)
   → Router (users.py / posts.py)  ó  ruta HTML (main.py + Jinja2)
   → SQLAlchemy async
   → PostgreSQL
```

### Autenticación

1. **Registro** (`POST /api/users`): valida con `UserCreate`, comprueba duplicados de username/email sin distinguir mayúsculas, hashea con Argon2 y guarda.
2. **Login** (`POST /api/users/token`): formulario OAuth2; el campo `username` se usa como **email**. Devuelve un JWT (`sub` = id del usuario, expira según `ACCESS_TOKEN_EXPIRE_MINUTES`).
3. **Endpoints protegidos**: declaran `current_user: CurrentUser`. FastAPI extrae el `Bearer token`, valida firma y expiración, y carga el usuario desde la BD. El `user_id` siempre sale del token, **nunca del body**.

### Posts

- `GET /api/posts` pagina con `skip` / `limit` (máx. 100) y usa `selectinload` para traer autores sin lazy loading (incompatible con async).
- `PUT` exige todos los campos; `PATCH` usa `model_dump(exclude_unset=True)` para actualizar solo lo enviado.
- Editar o borrar un post ajeno devuelve **403**.

### Imagen de perfil

```
PATCH /api/users/{id}/picture
  → valida tamaño (MAX_UPLOAD_SIZE_BYTES, 5 MB por defecto)
  → run_in_threadpool(process_profile_image)   # Pillow es síncrono / CPU-bound
       exif_transpose → fit 300×300 → RGB → JPEG (nombre uuid4)
  → upload a S3 (profile_pics/<uuid>.jpg)
  → guarda el nombre en user.image_file y borra la imagen anterior
```

### Recuperación de contraseña

1. `POST /api/users/forgot-password` genera un token aleatorio, guarda solo su **hash SHA-256** con caducidad y envía el email en una `BackgroundTask` (la respuesta no espera al SMTP).
2. El email enlaza a `FRONTEND_URL/reset-password?token=...`.
3. `POST /api/users/reset-password` valida el token, cambia la contraseña e invalida el token.

### Errores: JSON o HTML según la ruta

Los manejadores de excepciones de [main.py](src/fastapi_blog/main.py) devuelven JSON si la ruta empieza por `/api` y la plantilla `error.html` en cualquier otro caso.

---

## 6. Endpoints

### API REST (`/api`)

| Método | Ruta | Auth | Descripción |
|---|---|---|---|
| `POST` | `/api/users` | — | Registrar usuario |
| `POST` | `/api/users/token` | — | Login, devuelve JWT |
| `GET` | `/api/users/me` | ✅ | Usuario autenticado |
| `PATCH` | `/api/users/me/password` | ✅ | Cambiar contraseña |
| `POST` | `/api/users/forgot-password` | — | Solicitar reset por email |
| `POST` | `/api/users/reset-password` | — | Restablecer con token |
| `GET` | `/api/users/{id}` | — | Perfil público |
| `PATCH` | `/api/users/{id}` | ✅ | Actualizar usuario |
| `DELETE` | `/api/users/{id}` | ✅ | Borrar usuario |
| `GET` | `/api/users/{id}/posts` | — | Posts de un usuario (paginado) |
| `PATCH` | `/api/users/{id}/picture` | ✅ | Subir foto de perfil |
| `DELETE` | `/api/users/{id}/picture` | ✅ | Eliminar foto de perfil |
| `GET` | `/api/posts` | — | Listar posts (paginado) |
| `POST` | `/api/posts` | ✅ | Crear post |
| `GET` | `/api/posts/posts/{id}` | — | Obtener un post |
| `PUT` | `/api/posts/{id}` | ✅ | Reemplazar post |
| `PATCH` | `/api/posts/{id}` | ✅ | Actualizar parcialmente |
| `DELETE` | `/api/posts/{id}` | ✅ | Borrar post |
| `GET` | `/health` | — | Estado de la app y la BD |

La documentación interactiva se genera automáticamente en `/docs` (Swagger) y `/redoc`.

### Páginas HTML

`/`, `/posts`, `/posts/{id}`, `/users/{id}/posts`, `/login`, `/register`, `/account`, `/forgot-password`, `/reset-password`.

---

## 7. Instalación y ejecución

### Requisitos

- **Python 3.14**
- [**uv**](https://docs.astral.sh/uv/) (recomendado)
- **PostgreSQL** en ejecución
- Un **bucket de S3** (o un endpoint compatible como LocalStack) para las imágenes de perfil
- Un servidor SMTP (opcional; solo para recuperación de contraseña)

### Pasos

```bash
# 1. Clonar el repositorio
git clone <url-del-repo>
cd Curso_FastApi

# 2. Instalar dependencias (crea .venv automáticamente)
uv sync

# 3. Crear la base de datos
createdb blog            # o el nombre que prefieras

# 4. Configurar el entorno
cp .env.example .env     # si no existe, crear .env a mano (ver sección 8)

# 5. Aplicar migraciones
uv run alembic -c src/fastapi_blog/alembic.ini upgrade head

# 6. (Opcional) Cargar datos de ejemplo — la app debe poder importarse
uv run python -m fastapi_blog.populate_db

# 7. Levantar el servidor de desarrollo
uv run fastapi dev src/fastapi_blog/main.py
```

La app queda disponible en <http://localhost:8000> y la documentación de la API en <http://localhost:8000/docs>.

> ⚠️ Ejecutá siempre los comandos desde la **raíz del proyecto**: `config.py` busca el archivo `.env` relativo al directorio actual.

### Producción

```bash
uv run fastapi run src/fastapi_blog/main.py --host 0.0.0.0 --port 8000
```

Detrás de un proxy inverso (Nginx/Caddy) con HTTPS, y con `FRONTEND_URL` apuntando al dominio público.

---

## 8. Variables de entorno

Crear un archivo `.env` en la raíz:

```env
# Obligatorias
DATABASE_URL=postgresql+asyncpg://usuario:password@localhost:5432/blog
SECRET_KEY=una-clave-larga-y-aleatoria
S3_BUCKET_NAME=mi-bucket

# S3
S3_REGION=us-east-1
S3_ACCESS_KEY_ID=
S3_SECRET_ACCESS_KEY=
# S3_ENDPOINT_URL=http://localhost:4566   # LocalStack / Moto

# Email (recuperación de contraseña)
MAIL_SERVER=smtp.ejemplo.com
MAIL_PORT=587
MAIL_USERNAME=
MAIL_PASSWORD=
MAIL_FROM=noreply@fastapiblog.com
MAIL_USE_TLS=true
FRONTEND_URL=http://localhost:8000
```

| Variable | Obligatoria | Default | Descripción |
|---|---|---|---|
| `DATABASE_URL` | ✅ | — | URL async de SQLAlchemy |
| `SECRET_KEY` | ✅ | — | Clave de firma de los JWT (`SecretStr`) |
| `S3_BUCKET_NAME` | ✅ | — | Bucket de imágenes |
| `S3_REGION` | | `us-east-1` | Región de AWS |
| `S3_ACCESS_KEY_ID` / `S3_SECRET_ACCESS_KEY` | | `None` | Si faltan, Boto3 usa el entorno (IAM role, `~/.aws`) |
| `S3_ENDPOINT_URL` | | `None` | Endpoint personalizado |
| `ALGORITHM` | | `HS256` | Algoritmo JWT |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | | `30` | Vida del JWT |
| `RESET_ACCESS_TOKEN_EXPIRE_MINUTES` | | `60` | Vida del token de reset |
| `MAX_UPLOAD_SIZE_BYTES` | | `5242880` | Tamaño máximo de imagen |
| `POST_PER_PAGE` | | `10` | Posts por página en las vistas HTML |
| `MAIL_*` | | ver `config.py` | Configuración SMTP |
| `FRONTEND_URL` | | `http://localhost:8000` | Base de los enlaces de los emails |

Si falta una variable obligatoria, la app falla al arrancar (*fail-fast*). Las políticas de IAM y del bucket están en [aws_iam_policy.json](src/fastapi_blog/aws_iam_policy.json) y [aws_bucket_policy.json](src/fastapi_blog/aws_bucket_policy.json).

---

## 9. Tests

La suite usa **pytest** con tests asíncronos (`anyio`), una BD PostgreSQL de test y **Moto** para simular S3:

- Cada test corre dentro de una transacción con `join_transaction_mode="create_savepoint"` y se revierte al terminar: BD limpia sin recrear tablas.
- `app.dependency_overrides[get_db]` inyecta la sesión de test.
- `httpx.AsyncClient` + `ASGITransport` llama a la app en memoria, sin abrir puertos.

Requiere una base `test_blog` accesible con la URL definida en [conftest.py](src/fastapi_blog/test/conftest.py).

```bash
createdb test_blog
uv run pytest src/fastapi_blog/test -v
```

---

## Roadmap

- [ ] **CircleCI**: pipeline de CI (instalar con `uv`, levantar PostgreSQL de servicio, correr `pytest`).
- [ ] **Coveralls**: reporte de cobertura publicado desde CI y badge en este README.
- [ ] Ampliar la cobertura de tests (recuperación de contraseña, emails, borrado de imagen).
- [ ] Endpoint y UI para *likes* (la columna `likes` ya existe en `posts`).
- [ ] Corregir el typo `qualiti=85` → `quality=85` en [image_utils.py](src/fastapi_blog/image_utils.py), si aún está presente.

---

## Autor

**leoneljfernandes**
