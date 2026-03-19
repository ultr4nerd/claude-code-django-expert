# Django Project — Claude Code Instructions

## Stack
- **Django 6.x** · Python 3.13 · PostgreSQL 17 · Redis 7
- **Package manager**: `uv` (NOT pip). Use `uv add`, `uv sync`, `uv run`.
- **Layout**: cookiecutter-django — `config/` for settings/urls/wsgi, apps in project root or `apps/` dir.
- **API**: Django REST Framework 3.15+ for REST endpoints.
- **Auth**: django-allauth. Custom user model from day one (`users` app).
- **Async**: Use async views/ORM for I/O-bound work. Django 6.x has mature async support.

## 12-Factor Compliance
- **Config**: All secrets and config via environment variables. Use `django-environ` (`env = environ.Env()`).
- **Never** hardcode `SECRET_KEY`, database URLs, API keys, or credentials.
- **Backing services**: DATABASE_URL, REDIS_URL, EMAIL_URL — always env-based.
- **Logs**: Stream to stdout. Never write to files in production.
- **Processes**: Stateless. No in-memory sessions; use DB or Redis.

## Settings Structure
```
config/
  settings/
    __init__.py
    base.py        # Shared settings
    local.py       # Development (DEBUG=True)
    production.py  # Production (security hardened)
    test.py        # Fast test settings
```

## Key Settings
```python
DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"
AUTH_USER_MODEL = "users.User"
MIDDLEWARE = [..., "django.contrib.auth.middleware.LoginRequiredMiddleware", ...]
```
Use the `STORAGES` dict (not legacy `DEFAULT_FILE_STORAGE`/`STATICFILES_STORAGE`).

## Import Order (enforced by Ruff)
```python
# 1. __future__
# 2. stdlib
# 3. third-party (django, rest_framework, etc.)
# 4. first-party (project apps)
# 5. local (relative imports within app)
```

## Naming Conventions
- Apps: short, lowercase, no underscores if possible (`users`, `orders`, `payments`).
- Models: singular PascalCase (`Order`, `LineItem`).
- URLs: kebab-case (`/api/v1/line-items/`).
- Templates: `app_name/model_action.html` (`orders/order_detail.html`).
- Services: `app/services.py` or `app/services/order_service.py`.

## Models
- Inherit from `TimeStampedModel` (django-extensions or custom) for `created`/`modified`.
- Use `db_default` for database-level defaults. Use `GeneratedField` for computed columns.
- Use `TextChoices`/`IntegerChoices` for enum fields.
- Never `null=True` on `CharField`/`TextField` — use `blank=True, default=""`.
- Always define `__str__`, `Meta.ordering`, `Meta.indexes`, `Meta.constraints`.
- Use model managers for reusable query patterns.

## Views
- Keep views thin. Business logic goes in `services.py`, query logic in `selectors.py`.
- Prefer CBVs for CRUD, FBVs for simple/custom logic.
- Use `async def` for views doing external API calls or heavy I/O.
- `LoginRequiredMiddleware` is active — use `@login_not_required` for public views.

## Templates (Django 6.x)
- Use `{% partialdef name %}...{% endpartialdef %}` for reusable template fragments.
- Use `{% querystring %}` tag (5.1+) instead of manual URL parameter building.
- Use `{{ forloop.length }}` (6.0+) for total iteration count.
- Minimize logic in templates. No raw queries, no complex Python.

## REST APIs (DRF)
- Explicit serializer fields — never `fields = "__all__"`.
- Use `@action` decorator for non-CRUD endpoints on ViewSets.
- Versioned URLs: `/api/v1/`.
- Pagination: `LimitOffsetPagination` or `CursorPagination` for large datasets.
- Always return consistent error format.

## Testing
- **Runner**: `pytest` + `pytest-django`. Config in `pyproject.toml`.
- **Fixtures**: `factory_boy` + `faker`. No JSON fixtures.
- **Structure**: `app/tests/test_models.py`, `test_views.py`, `test_services.py`, etc.
- Use `RequestFactory` for view unit tests, `APIClient` for integration tests.
- `@pytest.mark.django_db` on tests that hit the database.
- Mock external services. Never call real APIs in tests.

## Commands
```bash
# Run dev server
uv run python manage.py runserver

# Run tests
uv run pytest

# Run tests with coverage
uv run pytest --cov --cov-report=term-missing

# Migrations
uv run python manage.py makemigrations
uv run python manage.py migrate

# Create superuser
uv run python manage.py createsuperuser

# Lint & format
uv run ruff check . --fix
uv run ruff format .

# Docker
docker compose -f docker-compose.local.yml up -d
docker compose -f docker-compose.local.yml exec django python manage.py migrate

# Type checking
uv run mypy .
```

## Security Essentials
- Use `format_html()` — never `mark_safe()`.
- Never raw SQL without parameterized queries (`%s` placeholders or `Func` expressions).
- CSP headers: Use Django 6.x built-in CSP middleware.
- Always validate via Django forms or DRF serializers before saving.
- File uploads: validate type, size, and store outside webroot.

## Docker (Development)
```yaml
# docker-compose.local.yml services:
# django, postgres, redis, mailpit, node (if frontend)
```
Always use the same PostgreSQL version in dev and prod.

## Background Tasks
- Django 6.x built-in background tasks for simple async work.
- Celery + Redis for complex task queues, periodic tasks, retries.

## Git Conventions
- Never commit `.env`, `*.pyc`, `__pycache__/`, `media/`, `staticfiles/`.
- Review migrations before committing. Squash when migration count grows.
- Pre-commit hooks: ruff, mypy, django-upgrade.

## Skills & Agents
Use `/skill:django-new-app` when creating a new app.
Use `/skill:django-new-model` when creating a new model.
Use `/skill:django-new-api` when creating a new API endpoint.
Use `/skill:django-migration-check` before committing migrations.
Use `/skill:django-security-audit` for security reviews.
Use `/skill:django-performance` for performance optimization.
Use `@django-reviewer` for code reviews.
Use `@django-tester` to generate tests.
Use `@django-debugger` to debug issues.

## Rules
Path-specific rules auto-activate for: models, views, serializers, tests, templates, settings, migrations, and security (global). See `.claude/rules/` for details.
