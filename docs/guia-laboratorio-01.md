# Guía de Laboratorio 01 — Configuración base (Monolito MVT)

> **Parte 1 de 3** · ⏱ Duración estimada: **45 min – 1 hora**
> **Asignatura:** Programación Orientada a Objetos (4to curso)
> **Alcance:** preparar el entorno, crear el proyecto Django con arquitectura MVT y conectar MySQL.

| 📘 Esta guía | ➡️ Siguiente |
|---|---|
| **01** Configuración base | [02 — Backend Django (Modelos, Vistas, Templates)](./guia-laboratorio-02.md) |

---

## 1. Objetivos

- Crear entorno virtual y activarlo en Git Bash.
- Crear proyecto Django desde cero.
- Conectar Django a MySQL y verificar su funcionamiento.

---

## 2. Prerrequisitos

| Herramienta | Verificar |
|---|---|
| Python 3.12.x | `python --version` |
| pip | `pip --version` |
| Git Bash | `bash --version` |
| MySQL 8.0+ | `mysql --version` |

> 💡 ¿Falta Python? <https://www.python.org/downloads/windows/> (marque *Add Python to PATH*).
> 💡 ¿Falta MySQL? <https://dev.mysql.com/downloads/installer/>. Recuerde la contraseña de `root`.

```bash
python -m pip install --upgrade pip
```

---

## 3. Fase 1 — Proyecto Django

### 3.1 Abrir Git Bash en la carpeta del proyecto

```bash
cd "D:/UNEMI/2026/PERIODO-ABRIL-JUNIO/POO/POO-4TO-CURSO-DJANGO-POSTGRES-REACT"
```

Cree la carpeta `backend/` con el Explorador de Windows si no existe, luego:

```bash
cd backend
```

### 3.2 Crear y activar entorno virtual

```bash
python -m venv .venv
source .venv/Scripts/activate
```

> ⚠️ ¿Error? Pruebe: `source .venv/scripts/activate`

Verifique: el prompt debe mostrar `(.venv)`.

```bash
python -c "import sys; print(sys.prefix)"  # → .../backend/.venv
```

### 3.3 Instalar Django y crear proyecto

```bash
pip install Django
django-admin startproject config .
```

Verifique:

```bash
ls -la
```

Debe ver: `.venv/`, `config/`, `manage.py`.

### 3.4 Probar servidor

```bash
python manage.py migrate
python manage.py runserver
```

Abra `http://127.0.0.1:8000/` — debe ver la página de inicio de Django.

> Detener con `Ctrl + C`.

✅ **Checkpoint:** servidor arranca y muestra página de inicio.

---

## 4. Fase 2 — MySQL y configuración

### 4.1 Instalar dependencias

```bash
pip install python-decouple mysqlclient
```

### 4.2 Crear `.env`

📄 **`backend/.env`**

```ini
DJANGO_SECRET_KEY=genera-una-clave-aqui
DJANGO_DEBUG=True
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1

DB_ENGINE=django.db.backends.mysql
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=xxx
DB_NAME=ventas_db_local
DB_PORT=3306
```

Genere la clave:

```bash
python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
```

Copie el resultado en `DJANGO_SECRET_KEY`. Reemplace `xxx` en `DB_PASSWORD` con su contraseña de MySQL.

### 4.3 Reemplazar `settings.py`

📄 **`backend/config/settings.py`**

```python
from pathlib import Path
from decouple import Csv, config

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = config("DJANGO_SECRET_KEY")
DEBUG = config("DJANGO_DEBUG", default=False, cast=bool)
ALLOWED_HOSTS = config("DJANGO_ALLOWED_HOSTS", default="localhost,127.0.0.1", cast=Csv())

INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]

ROOT_URLCONF = "config.urls"

TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [BASE_DIR / "templates"],
        "APP_DIRS": True,
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.debug",
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",
                "django.contrib.messages.context_processors.messages",
            ],
        },
    },
]

WSGI_APPLICATION = "config.wsgi.application"

DATABASES = {
    "default": {
        "ENGINE": config("DB_ENGINE", default="django.db.backends.mysql"),
        "NAME": config("DB_NAME", default="ventas_db_local"),
        "USER": config("DB_USER", default="root"),
        "PASSWORD": config("DB_PASSWORD", default=""),
        "HOST": config("DB_HOST", default="localhost"),
        "PORT": config("DB_PORT", default="3306"),
    }
}

AUTH_PASSWORD_VALIDATORS = [
    {"NAME": "django.contrib.auth.password_validation.UserAttributeSimilarityValidator"},
    {"NAME": "django.contrib.auth.password_validation.MinimumLengthValidator"},
    {"NAME": "django.contrib.auth.password_validation.CommonPasswordValidator"},
    {"NAME": "django.contrib.auth.password_validation.NumericPasswordValidator"},
]

LANGUAGE_CODE = "es-ec"
TIME_ZONE = "America/Guayaquil"
USE_I18N = True
USE_TZ = True

STATIC_URL = "static/"
STATICFILES_DIRS = [BASE_DIR / "static"]
DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"

LOGIN_URL = "/auth/login/"
LOGIN_REDIRECT_URL = "/dashboard/"
LOGOUT_REDIRECT_URL = "/auth/login/"
```

### 4.4 Verificar

```bash
python manage.py check
```

Debe mostrar: `System check identified no issues (0 silenced).`

```bash
pip freeze > requirements.txt
```

✅ **Checkpoint:** `check` sin errores, `.env` existe.

---

## 5. Fase 3 — Base de datos

### 5.1 Crear BD en MySQL

```sql
CREATE DATABASE ventas_db_local CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### 5.2 Migrar y crear superusuario

```bash
python manage.py migrate
python manage.py createsuperuser
# Username: admin | Email: admin@example.com | Password: ********
```

✅ **Checkpoint:** BD creada, migrado, superusuario OK.

---

## Cierre

- [x] Entorno virtual + Django instalado.
- [x] MySQL conectado (`.env` + `settings.py`).
- [x] Superusuario creado.

**➡️ [Parte 2 — Modelos, Vistas, Templates](./guia-laboratorio-02.md)**
