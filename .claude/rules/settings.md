---
paths:
  - "**/settings/**/*.py"
  - "**/settings.py"
---

# Django Settings Rules

## django-environ
Always use `django-environ` for configuration. Never hardcode secrets.
```python
import environ

env = environ.Env(
    DEBUG=(bool, False),
    ALLOWED_HOSTS=(list, []),
)

# Read .env file if it exists
environ.Env.read_env(BASE_DIR / ".env")

SECRET_KEY = env("DJANGO_SECRET_KEY")
DEBUG = env("DEBUG")
ALLOWED_HOSTS = env("ALLOWED_HOSTS")
```

## Settings File Structure
```
config/
  settings/
    __init__.py      # empty or imports local
    base.py          # shared settings for all environments
    local.py         # development (imports base)
    production.py    # production (imports base)
    test.py          # test overrides (imports base)
```
```python
# local.py
from .base import *  # noqa: F401,F403

DEBUG = True
ALLOWED_HOSTS = ["localhost", "127.0.0.1", "0.0.0.0"]
```

## Required Base Settings
```python
# base.py
DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"
AUTH_USER_MODEL = "users.User"
WSGI_APPLICATION = "config.wsgi.application"
ROOT_URLCONF = "config.urls"
```

## Database — Always via URL
```python
DATABASES = {
    "default": env.db("DATABASE_URL", default="postgres:///myproject"),
}
DATABASES["default"]["ATOMIC_REQUESTS"] = True
# PostgreSQL connection pooling (Django 5.1+)
DATABASES["default"]["OPTIONS"] = {
    "pool": True,
}
```

## STORAGES (Django 4.2+)
Use the unified `STORAGES` dict. Never use legacy `DEFAULT_FILE_STORAGE` or `STATICFILES_STORAGE`:
```python
STORAGES = {
    "default": {
        "BACKEND": "django.core.files.storage.FileSystemStorage",
    },
    "staticfiles": {
        "BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage",
    },
}
```

## Caching
```python
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": env("REDIS_URL", default="redis://localhost:6379/0"),
    }
}
```

## Security Settings (production.py)
```python
SECURE_HSTS_SECONDS = 31536000  # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SECURE_SSL_REDIRECT = True
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
CSRF_TRUSTED_ORIGINS = env.list("CSRF_TRUSTED_ORIGINS", default=[])
```

## CSP (Django 6.0+)
Use Django's built-in Content Security Policy middleware:
```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.middleware.csp.CSPMiddleware",
    ...
]

CONTENT_SECURITY_POLICY = {
    "default-src": ["'self'"],
    "script-src": ["'self'"],
    "style-src": ["'self'"],
    "img-src": ["'self'", "data:"],
}
```

## LoginRequiredMiddleware (Django 5.1+)
```python
MIDDLEWARE = [
    ...
    "django.contrib.auth.middleware.LoginRequiredMiddleware",
]
```
All views require login by default. Use `@login_not_required` for public views.

## Email
```python
EMAIL_BACKEND = env(
    "DJANGO_EMAIL_BACKEND",
    default="django.core.mail.backends.smtp.EmailBackend",
)
EMAIL_URL = env.email_url("EMAIL_URL", default="smtp://localhost:1025")
```

## Logging
```python
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "verbose": {
            "format": "{levelname} {asctime} {module} {message}",
            "style": "{",
        },
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "verbose",
        },
    },
    "root": {
        "handlers": ["console"],
        "level": "INFO",
    },
    "loggers": {
        "django.db.backends": {
            "level": env("DJANGO_DB_LOG_LEVEL", default="WARNING"),
        },
    },
}
```

## Test Settings (test.py)
```python
from .base import *  # noqa: F401,F403

PASSWORD_HASHERS = ["django.contrib.auth.hashers.MD5PasswordHasher"]
EMAIL_BACKEND = "django.core.mail.backends.locmem.EmailBackend"
CACHES = {"default": {"BACKEND": "django.core.cache.backends.locmem.LocMemCache"}}
```

## Never Do
- Never commit `.env` files.
- Never set `DEBUG = True` in production settings.
- Never use `*` in `ALLOWED_HOSTS` in production.
- Never hardcode `SECRET_KEY` — always read from environment.
- Never use SQLite in production or as a test DB if prod uses PostgreSQL.
