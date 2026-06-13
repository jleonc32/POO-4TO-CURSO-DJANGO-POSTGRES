# Guía de Laboratorio 02 — Modelos, Vistas, Templates (MVT)

> **Parte 2 de 3** · ⏱ Duración estimada: **2 – 2.5 horas**
> **Asignatura:** Programación Orientada a Objetos (4to curso)
> **Prerrequisito:** [Parte 1 — Configuración base](./guia-laboratorio-01.md) completada.
> **Alcance:** crear apps con `startapp`, modelo User personalizado con Manager, modelo Note, CRUD completo con templates Bootstrap y autenticación.

| ⬅️ Anterior | 📘 Esta guía | ➡️ Siguiente |
|---|---|---|
| [01 — Configuración base](./guia-laboratorio-01.md) | **02** Backend MVT | [03 — UML + Verificación](./guia-laboratorio-03.md) |

---

## 1. Fase 4 — Apps `accounts` y `core`

### 1.1 Activar entorno y crear carpetas base

```bash
cd "D:/UNEMI/2026/PERIODO-ABRIL-JUNIO/POO/POO-4TO-CURSO-DJANGO-POSTGRES-REACT/backend"
source .venv/Scripts/activate
```

Cree las carpetas para templates y archivos estáticos:

```bash
mkdir -p templates static/css
```

### 1.2 Crear apps con `startapp`

Django genera automáticamente la estructura de cada app. Las apps se crean al mismo nivel que `config/` (estructura plana):

```bash
python manage.py startapp accounts
python manage.py startapp core
```

Resultado:

```
backend/
├── accounts/          ← App de usuarios
│   ├── migrations/
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── models.py
│   ├── tests.py
│   └── views.py
├── core/              ← App de notas
│   └── (misma estructura)
├── config/            ← Configuración del proyecto
├── templates/
├── static/
└── manage.py
```

Cree archivos adicionales que Django no genera:

```bash
touch accounts/urls.py accounts/forms.py accounts/managers.py
touch core/urls.py core/forms.py
mkdir -p accounts/templates/accounts core/templates/core
```

### 1.3 Registrar apps en `settings.py`

Edite `INSTALLED_APPS` en `config/settings.py`:

```python
INSTALLED_APPS = [
    ...
    "django.contrib.staticfiles",
    # Apps locales
    "accounts",
    "core",
]
```

---

## 2. Fase 5 — Modelo User personalizado

> Se usa `AbstractBaseUser` + `PermissionsMixin` + `CustomUserManager`, patrón profesional que ofrece control total sobre el modelo de usuario.

### 2.1 Manager

📄 **`accounts/managers.py`**

```python
from django.contrib.auth.base_user import BaseUserManager
from django.core.exceptions import ValidationError
from django.core.validators import validate_email


class CustomUserManager(BaseUserManager):
    def email_validator(self, email):
        try:
            validate_email(email)
        except ValidationError:
            raise ValueError("Debe proporcionar un email válido")

    def create_user(self, username, email, password=None, **extra_fields):
        if not username:
            raise ValueError("El usuario es obligatorio")
        if email:
            email = self.normalize_email(email)
            self.email_validator(email)
        else:
            raise ValueError("El email es obligatorio")

        user = self.model(username=username, email=email, **extra_fields)
        user.set_password(password)
        extra_fields.setdefault("is_staff", False)
        extra_fields.setdefault("is_superuser", False)
        user.save(using=self._db)
        return user

    def create_superuser(self, username, email, password=None, **extra_fields):
        extra_fields.setdefault("is_staff", True)
        extra_fields.setdefault("is_superuser", True)
        extra_fields.setdefault("is_active", True)

        if extra_fields.get("is_staff") is not True:
            raise ValueError("Superuser debe tener is_staff=True")
        if extra_fields.get("is_superuser") is not True:
            raise ValueError("Superuser debe tener is_superuser=True")

        return self.create_user(username, email, password, **extra_fields)
```

### 2.2 Modelo User

📄 **`accounts/models.py`**

```python
from django.db import models
from django.contrib.auth.models import AbstractBaseUser, PermissionsMixin
from django.utils import timezone
from django.utils.translation import gettext_lazy as _
from .managers import CustomUserManager


class User(AbstractBaseUser, PermissionsMixin):
    username = models.CharField(_("usuario"), max_length=150, unique=True)
    email = models.EmailField(_("correo electrónico"), unique=True)
    first_name = models.CharField(_("nombres"), max_length=50, blank=True)
    last_name = models.CharField(_("apellidos"), max_length=50, blank=True)
    is_staff = models.BooleanField(default=False)
    is_active = models.BooleanField(default=True)
    date_joined = models.DateTimeField(default=timezone.now)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    USERNAME_FIELD = "email"
    REQUIRED_FIELDS = ["username"]

    objects = CustomUserManager()

    class Meta:
        verbose_name = _("usuario")
        verbose_name_plural = _("usuarios")
        ordering = ("-date_joined",)

    def __str__(self):
        return self.username
```

> 💡 **USERNAME_FIELD = "email"**: el login se hace con email, no con username. `username` sigue siendo obligatorio (está en `REQUIRED_FIELDS`).

### 2.3 Configurar AUTH_USER_MODEL

En `config/settings.py`, agregue antes de `LOGIN_URL`:

```python
AUTH_USER_MODEL = "accounts.User"
```

### 2.4 Apps config

📄 **`accounts/apps.py`**

```python
from django.apps import AppConfig

class AccountsConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "accounts"
```

📄 **`core/apps.py`**

```python
from django.apps import AppConfig

class CoreConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "core"
    verbose_name = "Notas"
```

### 2.5 Migrar

```bash
python manage.py makemigrations
python manage.py migrate
```

✅ **Checkpoint:** migraciones de `accounts` aplicadas sin errores.

---

## 3. Fase 6 — Autenticación (login, registro, logout)

### 3.1 Admin

📄 **`accounts/admin.py`**

```python
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin as BaseUserAdmin
from django.utils.translation import gettext_lazy as _
from .models import User


@admin.register(User)
class UserAdmin(BaseUserAdmin):
    ordering = ("email",)
    list_display = ("id", "email", "username", "first_name", "last_name", "is_staff", "is_active")
    list_filter = ("is_staff", "is_active")
    search_fields = ("email", "username", "first_name", "last_name")
    fieldsets = (
        (_("Credenciales"), {"fields": ("email", "password")}),
        (_("Información personal"), {"fields": ("username", "first_name", "last_name")}),
        (_("Permisos"), {"fields": ("is_active", "is_staff", "is_superuser", "groups", "user_permissions")}),
        (_("Fechas"), {"fields": ("last_login", "date_joined")}),
    )
    add_fieldsets = (
        (None, {"classes": ("wide",), "fields": ("email", "username", "password1", "password2")}),
    )
```

### 3.2 Formulario de registro

📄 **`accounts/forms.py`**

```python
from django.contrib.auth.forms import UserCreationForm
from .models import User

class UserRegisterForm(UserCreationForm):
    class Meta:
        model = User
        fields = ("email", "username", "first_name", "last_name")
```

### 3.3 Vistas

📄 **`accounts/views.py`**

```python
from django.urls import reverse_lazy
from django.views.generic import CreateView
from .forms import UserRegisterForm

class RegisterView(CreateView):
    form_class = UserRegisterForm
    template_name = "accounts/register.html"
    success_url = reverse_lazy("login")
```

### 3.4 URLs

📄 **`accounts/urls.py`**

```python
from django.contrib.auth.views import LoginView, LogoutView
from django.urls import path
from .views import RegisterView

app_name = "accounts"
urlpatterns = [
    path("login/", LoginView.as_view(template_name="accounts/login.html"), name="login"),
    path("logout/", LogoutView.as_view(), name="logout"),
    path("register/", RegisterView.as_view(), name="register"),
]
```

### 3.5 Templates de accounts

📄 **`accounts/templates/accounts/register.html`**

```html
{% extends "base.html" %}
{% block title %}Registro{% endblock %}
{% block content %}
<div class="row justify-content-center">
  <div class="col-md-5">
    <h2 class="mb-4">Crear cuenta</h2>
    <form method="post">
      {% csrf_token %}
      {{ form.as_p }}
      <button type="submit" class="btn btn-primary w-100">Registrarme</button>
    </form>
    <p class="mt-3 text-center">¿Ya tienes cuenta? <a href="{% url 'accounts:login' %}">Inicia sesión</a></p>
  </div>
</div>
{% endblock %}
```

📄 **`accounts/templates/accounts/login.html`**

```html
{% extends "base.html" %}
{% block title %}Iniciar sesión{% endblock %}
{% block content %}
<div class="row justify-content-center">
  <div class="col-md-5">
    <h2 class="mb-4">Iniciar sesión</h2>
    <form method="post">
      {% csrf_token %}
      {{ form.as_p }}
      <button type="submit" class="btn btn-primary w-100">Entrar</button>
    </form>
    <p class="mt-3 text-center">¿No tienes cuenta? <a href="{% url 'accounts:register' %}">Regístrate</a></p>
  </div>
</div>
{% endblock %}
```

✅ **Checkpoint:** `/auth/register/`, `/auth/login/`, `/auth/logout/` funcionan.

---

## 4. Fase 7 — Modelo Note y CRUD

### 4.1 Modelo

📄 **`core/models.py`**

```python
from django.conf import settings
from django.db import models


class Note(models.Model):
    owner = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE,
        related_name="notes", verbose_name="propietario",
    )
    title = models.CharField("título", max_length=200)
    body = models.TextField("contenido", blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ("-created_at",)
        verbose_name = "nota"
        verbose_name_plural = "notas"

    def __str__(self):
        return self.title
```

```bash
python manage.py makemigrations core
python manage.py migrate
```

### 4.2 Admin

📄 **`core/admin.py`**

```python
from django.contrib import admin
from .models import Note

@admin.register(Note)
class NoteAdmin(admin.ModelAdmin):
    list_display = ("id", "title", "owner", "created_at")
    list_filter = ("owner",)
    search_fields = ("title", "body", "owner__username")
```

### 4.3 Formulario

📄 **`core/forms.py`**

```python
from django import forms
from .models import Note

class NoteForm(forms.ModelForm):
    class Meta:
        model = Note
        fields = ("title", "body")
        widgets = {
            "title": forms.TextInput(attrs={"class": "form-control", "placeholder": "Título"}),
            "body": forms.Textarea(attrs={"class": "form-control", "placeholder": "Contenido", "rows": 3}),
        }
```

### 4.4 Vistas

📄 **`core/views.py`**

```python
from django.contrib.auth.mixins import LoginRequiredMixin
from django.urls import reverse_lazy
from django.views.generic import CreateView, DeleteView, ListView
from .forms import NoteForm
from .models import Note


class DashboardView(LoginRequiredMixin, ListView):
    model = Note
    template_name = "core/dashboard.html"
    context_object_name = "notes"

    def get_queryset(self):
        return Note.objects.filter(owner=self.request.user)


class NoteCreateView(LoginRequiredMixin, CreateView):
    model = Note
    form_class = NoteForm
    template_name = "core/note_form.html"
    success_url = reverse_lazy("dashboard")

    def form_valid(self, form):
        form.instance.owner = self.request.user
        return super().form_valid(form)


class NoteDeleteView(LoginRequiredMixin, DeleteView):
    model = Note
    success_url = reverse_lazy("dashboard")

    def get_queryset(self):
        return Note.objects.filter(owner=self.request.user)
```

### 4.5 URLs

📄 **`core/urls.py`**

```python
from django.urls import path
from .views import DashboardView, NoteCreateView, NoteDeleteView

app_name = "core"
urlpatterns = [
    path("", DashboardView.as_view(), name="dashboard"),
    path("nueva/", NoteCreateView.as_view(), name="note_create"),
    path("eliminar/<int:pk>/", NoteDeleteView.as_view(), name="note_delete"),
]
```

### 4.6 Templates de core

📄 **`core/templates/core/dashboard.html`**

```html
{% extends "base.html" %}
{% block title %}Dashboard{% endblock %}
{% block content %}
<div class="d-flex justify-content-between align-items-center mb-4">
  <h2>Mis notas</h2>
  <a href="{% url 'core:note_create' %}" class="btn btn-success">+ Nueva nota</a>
</div>
{% if notes %}
  <div class="row">
    {% for note in notes %}
      <div class="col-md-4 mb-3">
        <div class="card">
          <div class="card-body">
            <h5 class="card-title">{{ note.title }}</h5>
            <p class="card-text">{{ note.body|linebreaksbr }}</p>
            <p class="text-muted small">{{ note.created_at|date:"d/m/Y H:i" }}</p>
            <form method="post" action="{% url 'core:note_delete' note.pk %}" style="display:inline">
              {% csrf_token %}
              <button type="submit" class="btn btn-sm btn-danger">Eliminar</button>
            </form>
          </div>
        </div>
      </div>
    {% endfor %}
  </div>
{% else %}
  <div class="alert alert-info">No tienes notas aún. ¡Crea tu primera nota!</div>
{% endif %}
{% endblock %}
```

📄 **`core/templates/core/note_form.html`**

```html
{% extends "base.html" %}
{% block title %}Nueva nota{% endblock %}
{% block content %}
<div class="row justify-content-center">
  <div class="col-md-6">
    <h2 class="mb-4">Nueva nota</h2>
    <form method="post">
      {% csrf_token %}
      {{ form.as_p }}
      <button type="submit" class="btn btn-primary">Guardar</button>
      <a href="{% url 'core:dashboard' %}" class="btn btn-secondary">Cancelar</a>
    </form>
  </div>
</div>
{% endblock %}
```

✅ **Checkpoint:** dashboard muestra notas, crear y eliminar funcionan.

---

## 5. Fase 8 — Template base (Bootstrap 5)

📄 **`templates/base.html`**

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{% block title %}Notas App{% endblock %}</title>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body>
  <nav class="navbar navbar-expand-lg navbar-dark bg-dark mb-4">
    <div class="container">
      <a class="navbar-brand" href="{% url 'core:dashboard' %}">Notas App</a>
      <div class="navbar-nav ms-auto">
        {% if user.is_authenticated %}
          <span class="nav-link text-light">{{ user.username }}</span>
          <form method="post" action="{% url 'accounts:logout' %}" class="d-inline">
            {% csrf_token %}
            <button type="submit" class="btn nav-link text-danger">Salir</button>
          </form>
        {% else %}
          <a class="nav-link" href="{% url 'accounts:login' %}">Login</a>
          <a class="nav-link" href="{% url 'accounts:register' %}">Registro</a>
        {% endif %}
      </div>
    </div>
  </nav>
  <div class="container">{% block content %}{% endblock %}</div>
  <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
```

### 5.1 Enrutamiento principal

📄 **`config/urls.py`**

```python
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path("admin/", admin.site.urls),
    path("auth/", include("accounts.urls")),
    path("dashboard/", include("core.urls")),
    path("", include("core.urls")),  # raíz → dashboard
]
```

---

## 6. Probar el sistema completo

```bash
python manage.py runserver
```

| Prueba | URL |
|---|---|
| Admin | `http://127.0.0.1:8000/admin/` |
| Registro | `http://127.0.0.1:8000/auth/register/` |
| Login | `http://127.0.0.1:8000/auth/login/` |
| Dashboard | `http://127.0.0.1:8000/dashboard/` |

**Flujo:** registrar usuario → login → crear nota → eliminar nota → logout.

---

## Cierre

- [x] Apps `accounts` y `core` creadas con `startapp`.
- [x] Modelo User personalizado (AbstractBaseUser + CustomUserManager).
- [x] Autenticación completa (registro, login, logout).
- [x] CRUD de notas con Bootstrap 5.
- [x] Template base reutilizable.

**➡️ [Parte 3 — UML + Verificación Final](./guia-laboratorio-03.md)**
