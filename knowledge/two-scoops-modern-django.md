# Two Scoops of Django: Modern Edition (Django 6.x)

> A comprehensive, authoritative reference for building production Django applications.
> Updated for Django 6.0.3, Python 3.13, and the modern Python ecosystem (2026).

---

## 1. Coding Style

### 1.1 PEP 8 and Modern Python 3.13 Conventions

Follow PEP 8 as the baseline, with these modern Python 3.13 conventions:

- **Line length**: 88 characters (Black/Ruff standard, not strict 79).
- **Trailing commas**: Always use trailing commas in multi-line structures. This produces cleaner diffs.
- **f-strings**: Prefer f-strings over `format()` or `%` formatting. Python 3.12+ supports f-string nesting.
- **Type unions**: Use `X | Y` syntax instead of `Union[X, Y]` (Python 3.10+).
- **Built-in generics**: Use `list[str]`, `dict[str, int]` instead of `List[str]`, `Dict[str, int]`.
- **`Self` type**: Use `typing.Self` for methods returning the same class (Python 3.11+).

```python
# MODERN (Python 3.13)
def get_active_users(
    self,
    role: str | None = None,
    limit: int = 100,
) -> list["User"]:
    """Return active users, optionally filtered by role."""
    qs = self.filter(is_active=True)
    if role is not None:
        qs = qs.filter(role=role)
    return list(qs[:limit])

# DEPRECATED STYLE - do not use
from typing import List, Optional, Union
def get_active_users(
    self,
    role: Optional[str] = None,  # Use `str | None` instead
    limit: int = 100,
) -> List["User"]:  # Use `list["User"]` instead
    ...
```

### 1.2 Ruff: The Modern Standard Linter/Formatter

**Ruff replaces flake8, isort, black, pyflakes, pycodestyle, and many more.** It's written in Rust and is 10-100x faster than the tools it replaces.

> **DEPRECATED**: Do not use flake8, isort, black, autopep8, or pyflakes separately. Use Ruff for all linting and formatting.

Configure Ruff in `pyproject.toml`:

```toml
[tool.ruff]
target-version = "py313"
line-length = 88

[tool.ruff.lint]
select = [
    "E",      # pycodestyle errors
    "W",      # pycodestyle warnings
    "F",      # pyflakes
    "I",      # isort
    "B",      # flake8-bugbear
    "C4",     # flake8-comprehensions
    "UP",     # pyupgrade
    "DJ",     # flake8-django
    "S",      # flake8-bandit (security)
    "SIM",    # flake8-simplify
    "T20",    # flake8-print
    "RUF",    # Ruff-specific rules
]
ignore = [
    "E501",   # line too long (handled by formatter)
    "S101",   # assert (needed in tests)
]

[tool.ruff.lint.isort]
known-first-party = ["myproject"]
section-order = ["future", "standard-library", "third-party", "first-party", "local-folder"]

[tool.ruff.lint.per-file-ignores]
"**/tests/**" = ["S101"]  # Allow assert in tests
"**/migrations/**" = ["E501"]  # Allow long lines in migrations

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
docstring-code-format = true
```

Pre-commit integration:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.14.14
    hooks:
      - id: ruff-check
        args: [--fix]
      - id: ruff-format
```

Run Ruff:

```bash
# Lint
ruff check .

# Lint and auto-fix
ruff check --fix .

# Format (like Black)
ruff format .

# Check formatting without modifying
ruff format --check .
```

### 1.3 Type Hints in Django (Modern Approach)

Use type hints throughout your Django code. While Django's own type stubs (`django-stubs`) continue to improve, type hints are invaluable for documentation and IDE support.

```python
from django.db import models
from django.http import HttpRequest, HttpResponse, JsonResponse


# Models with type hints
class Article(models.Model):
    title: models.CharField = models.CharField(max_length=200)
    slug: models.SlugField = models.SlugField(unique=True)
    published: models.BooleanField = models.BooleanField(default=False)

    class Meta:
        ordering: list[str] = ["-created"]

    def __str__(self) -> str:
        return self.title

    def get_absolute_url(self) -> str:
        return f"/articles/{self.slug}/"


# Views with type hints
def article_detail(request: HttpRequest, slug: str) -> HttpResponse:
    article = get_object_or_404(Article, slug=slug)
    return render(request, "articles/detail.html", {"article": article})


# Service layer with type hints
def publish_article(article_id: int, user: "User") -> Article:
    article = Article.objects.get(id=article_id)
    article.published = True
    article.published_by = user
    article.save(update_fields=["published", "published_by"])
    return article
```

For `django-stubs` integration with mypy:

```toml
# pyproject.toml
[tool.mypy]
plugins = ["mypy_django_plugin.main"]
strict = true

[tool.django-stubs]
django_settings_module = "config.settings.local"
```

### 1.4 Django Coding Conventions

- **Imports order** (handled by Ruff `isort`):
  1. Future imports
  2. Standard library
  3. Third-party (Django first, then others alphabetically)
  4. First-party (your project)
  5. Local folder (relative imports)

```python
from __future__ import annotations

import logging
from datetime import datetime, timedelta

from django.conf import settings
from django.db import models
from django.utils import timezone
import requests

from myproject.core.models import TimeStampedModel

from .constants import STATUS_CHOICES
```

- **Use relative imports within an app**, absolute imports across apps.
- **Model field order**: Follow Django convention — database fields, then `Meta`, then `__str__`, then `save()`, then `get_absolute_url()`, then custom methods.
- **Avoid wildcard imports** (`from module import *`). Always be explicit.
- **Use `gettext_lazy`** for any string that may need translation:

```python
from django.utils.translation import gettext_lazy as _

class Article(models.Model):
    title = models.CharField(_("title"), max_length=200)

    class Meta:
        verbose_name = _("article")
        verbose_name_plural = _("articles")
```

---

## 2. Environment Setup

### 2.1 uv: The Modern Python Package Manager

**uv replaces pip, pip-tools, virtualenv, and pyenv.** Written in Rust by Astral (same team as Ruff), it's 10-100x faster.

> **DEPRECATED**: Do not use `pip`, `pip-tools`, `virtualenv`, `pyenv`, or `poetry` for new projects. Use `uv`.

Install uv:

```bash
# macOS/Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Common uv commands:

```bash
# Create a new project
uv init myproject
cd myproject

# Pin Python version
uv python pin 3.13

# Add dependencies
uv add django~=6.0
uv add psycopg[binary]
uv add django-environ django-allauth whitenoise

# Add development dependencies
uv add --dev pytest pytest-django factory-boy django-debug-toolbar ruff

# Install all dependencies (creates/updates .venv)
uv sync

# Run commands in the virtual environment
uv run python manage.py runserver
uv run pytest

# Lock dependencies (automatic on `uv add` and `uv sync`)
# uv.lock is the lockfile — always commit it

# Install only production dependencies
uv sync --no-dev

# Upgrade a dependency
uv add django --upgrade
```

### 2.2 pyproject.toml as the Standard

`pyproject.toml` is the single configuration file for your project and tools. It replaces `setup.py`, `setup.cfg`, `requirements.txt`, and individual tool configs.

```toml
[project]
name = "myproject"
version = "1.0.0"
description = "My Django Application"
readme = "README.md"
requires-python = ">=3.13"
dependencies = [
    "django~=6.0",
    "psycopg[binary]>=3.2",
    "django-environ>=0.12",
    "django-allauth>=65.0",
    "django-filter>=24.0",
    "django-cors-headers>=4.6",
    "whitenoise>=6.8",
    "gunicorn>=23.0",
    "uvicorn[standard]>=0.34",
    "sentry-sdk>=2.0",
    "redis>=5.2",
    "celery>=5.4",
    "structlog>=24.0",
]

[dependency-groups]
dev = [
    "pytest>=8.0",
    "pytest-django>=4.9",
    "pytest-cov>=6.0",
    "factory-boy>=3.3",
    "faker>=33.0",
    "django-debug-toolbar>=5.0",
    "django-silk>=5.3",
    "ruff>=0.14",
    "mypy>=1.14",
    "django-stubs>=5.1",
    "pre-commit>=4.0",
    "ipdb>=0.13",
]

[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "config.settings.test"
python_files = ["tests.py", "test_*.py"]
addopts = [
    "--strict-markers",
    "--tb=short",
    "--reuse-db",
    "-q",
]
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "integration: marks integration tests",
]

[tool.coverage.run]
source = ["myproject"]
omit = [
    "*/migrations/*",
    "*/tests/*",
    "manage.py",
]
branch = true

[tool.coverage.report]
show_missing = true
fail_under = 90

[tool.ruff]
target-version = "py313"
line-length = 88

[tool.ruff.lint]
select = ["E", "W", "F", "I", "B", "C4", "UP", "DJ", "S", "SIM", "T20", "RUF"]
ignore = ["E501", "S101"]

[tool.ruff.lint.isort]
known-first-party = ["myproject"]

[tool.mypy]
plugins = ["mypy_django_plugin.main"]
python_version = "3.13"
warn_return_any = true
warn_unused_configs = true

[tool.django-stubs]
django_settings_module = "config.settings.local"
```

### 2.3 Docker Compose for Development

Every developer on the team should use Docker Compose for local development to ensure identical environments:

```yaml
# docker-compose.yml
services:
  django:
    build:
      context: .
      dockerfile: Dockerfile
      target: development
    command: uv run python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  postgres:
    image: postgres:17-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: myproject
      POSTGRES_USER: myproject
      POSTGRES_PASSWORD: myproject
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myproject"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  mailpit:
    image: axllent/mailpit
    ports:
      - "8025:8025"  # Web UI
      - "1025:1025"  # SMTP

  celery_worker:
    build:
      context: .
      dockerfile: Dockerfile
      target: development
    command: uv run celery -A config worker -l info
    volumes:
      - .:/app
    env_file:
      - .env
    depends_on:
      - postgres
      - redis

volumes:
  postgres_data:
```

Development Dockerfile:

```dockerfile
# Dockerfile
FROM python:3.13-slim AS base

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    UV_COMPILE_BYTECODE=1

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Install uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

# ---- Development ----
FROM base AS development
COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen
COPY . .

# ---- Production ----
FROM base AS production
COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev
COPY . .
RUN uv run python manage.py collectstatic --noinput
CMD ["uv", "run", "gunicorn", "config.asgi:application", \
     "-k", "uvicorn_worker.UvicornWorker", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "4"]
```

### 2.4 PostgreSQL 17 as the Standard

Always use PostgreSQL. Never use SQLite in production. Use PostgreSQL in development too to maintain dev/prod parity.

**Why PostgreSQL 17:**
- Best Django support (full feature set)
- `RETURNING` clause for dynamic field refresh (Django 6.0)
- Connection pooling support (Django 5.1+)
- `EXPLAIN` improvements for `QuerySet.explain()`
- JSON fields, full-text search, array fields, range fields
- Composite primary key support

```python
# settings/base.py
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": env("DATABASE_NAME", default="myproject"),
        "USER": env("DATABASE_USER", default="myproject"),
        "PASSWORD": env("DATABASE_PASSWORD", default="myproject"),
        "HOST": env("DATABASE_HOST", default="localhost"),
        "PORT": env("DATABASE_PORT", default="5432"),
        "OPTIONS": {
            "pool": {  # Connection pooling (Django 5.1+)
                "min_size": 2,
                "max_size": 4,
                "timeout": 10,
            },
        },
        "CONN_MAX_AGE": 600,
        "CONN_HEALTH_CHECKS": True,
    }
}
```

---

## 3. Project Layout

### 3.1 Modern Cookiecutter-Django Layout

The modern Django project follows a three-level structure:

```
myproject/                    # <-- Repository Root
├── .github/
│   └── workflows/
│       └── ci.yml
├── .pre-commit-config.yaml
├── pyproject.toml            # All project & tool config
├── uv.lock                   # Locked dependencies
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── .gitignore
├── README.md
│
├── config/                   # <-- Configuration Root (Django project)
│   ├── __init__.py
│   ├── asgi.py
│   ├── wsgi.py
│   ├── urls.py
│   └── settings/
│       ├── __init__.py
│       ├── base.py           # Shared settings
│       ├── local.py          # Development settings
│       ├── production.py     # Production settings
│       └── test.py           # Test settings
│
├── myproject/                # <-- Project Root (your apps)
│   ├── __init__.py
│   ├── users/                # Custom user app
│   │   ├── __init__.py
│   │   ├── admin.py
│   │   ├── apps.py
│   │   ├── forms.py
│   │   ├── managers.py
│   │   ├── models.py
│   │   ├── services.py       # Business logic (service layer)
│   │   ├── selectors.py      # Complex queries
│   │   ├── urls.py
│   │   ├── views.py
│   │   ├── tasks.py          # Background tasks
│   │   ├── api/
│   │   │   ├── __init__.py
│   │   │   ├── serializers.py
│   │   │   └── views.py
│   │   ├── tests/
│   │   │   ├── __init__.py
│   │   │   ├── test_models.py
│   │   │   ├── test_views.py
│   │   │   ├── test_services.py
│   │   │   └── factories.py
│   │   ├── migrations/
│   │   └── templates/
│   │       └── users/
│   │
│   ├── articles/             # Another app
│   │   └── ...
│   │
│   ├── core/                 # Shared utilities
│   │   ├── __init__.py
│   │   ├── models.py         # TimeStampedModel, etc.
│   │   ├── services.py       # Shared service utilities
│   │   └── utils.py
│   │
│   └── templates/            # Project-level templates
│       ├── base.html
│       ├── 400.html
│       ├── 403.html
│       ├── 404.html
│       └── 500.html
│
├── static/                   # Project-level static files
│   ├── css/
│   ├── js/
│   └── images/
│
├── media/                    # User uploads (dev only)
│
└── docs/                     # Documentation
```

### 3.2 Three-Level Structure

1. **Repository Root** (`myproject/`): Contains Docker files, CI config, `pyproject.toml`, documentation. Not a Python package.
2. **Configuration Root** (`config/`): Django project settings, URL configuration, WSGI/ASGI entry points.
3. **Project Root** (`myproject/myproject/`): Your actual Django apps and shared code.

### 3.3 The apps/ Directory Pattern

For large projects, group apps under a namespace:

```
myproject/
├── config/
└── myproject/
    ├── __init__.py
    ├── core/
    ├── users/
    ├── billing/
    ├── orders/
    └── products/
```

Each app follows the same internal structure. The `core/` app holds shared models, utilities, and base classes.

### 3.4 Modern Tooling Files

**`.pre-commit-config.yaml`**:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-toml
      - id: check-merge-conflict
      - id: debug-statements
      - id: check-added-large-files

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.14.14
    hooks:
      - id: ruff-check
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.14.0
    hooks:
      - id: mypy
        additional_dependencies:
          - django-stubs>=5.1
```

**`.env.example`**:

```bash
# Django
DJANGO_SETTINGS_MODULE=config.settings.local
DJANGO_SECRET_KEY=change-me-in-production
DJANGO_DEBUG=True
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1

# Database
DATABASE_URL=postgres://myproject:myproject@localhost:5432/myproject

# Redis
REDIS_URL=redis://localhost:6379/0

# Email
EMAIL_URL=smtp://localhost:1025

# Sentry
SENTRY_DSN=
```

---

## 4. App Design

### 4.1 The Golden Rule of Django App Design

> "Each app should do one thing and do it well."

The golden rule from "Two Scoops" still holds in Django 6.x. An app should be describable in a single sentence without using "and." If you need "and," you probably need two apps.

**Good app names**: `users`, `articles`, `payments`, `notifications`, `analytics`
**Bad app names**: `utils` (too vague), `main` (meaningless), `stuff`

### 4.2 Service Layer Pattern (Modern Approach)

The service layer pattern has become the de facto standard for Django projects beyond simple CRUD. It separates business logic from models and views.

**Pattern**: Models define data, views handle HTTP, services contain business logic.

```python
# myproject/orders/services.py
from __future__ import annotations

from decimal import Decimal

from django.db import transaction
from django.utils import timezone

from myproject.notifications.tasks import send_order_confirmation_email
from myproject.orders.models import Order, OrderItem
from myproject.products.models import Product


class OrderService:
    """Business logic for order processing."""

    @staticmethod
    @transaction.atomic
    def create_order(
        *,  # Force keyword-only arguments
        user: "User",
        items: list[dict[str, int | Decimal]],
        shipping_address: str,
    ) -> Order:
        """
        Create an order with line items and update inventory.

        Args:
            user: The user placing the order.
            items: List of dicts with 'product_id' and 'quantity'.
            shipping_address: Shipping address string.

        Returns:
            The created Order instance.

        Raises:
            ValueError: If any product is out of stock.
        """
        order = Order.objects.create(
            user=user,
            shipping_address=shipping_address,
            status=Order.Status.PENDING,
        )

        total = Decimal("0.00")
        for item_data in items:
            product = Product.objects.select_for_update().get(
                id=item_data["product_id"]
            )
            quantity = item_data["quantity"]

            if product.stock < quantity:
                raise ValueError(
                    f"Insufficient stock for {product.name}. "
                    f"Available: {product.stock}, requested: {quantity}"
                )

            product.stock -= quantity
            product.save(update_fields=["stock"])

            line_total = product.price * quantity
            OrderItem.objects.create(
                order=order,
                product=product,
                quantity=quantity,
                unit_price=product.price,
                line_total=line_total,
            )
            total += line_total

        order.total = total
        order.save(update_fields=["total"])

        # Enqueue background task (Django 6.0 Tasks framework)
        send_order_confirmation_email.enqueue(
            order_id=order.id,
            user_email=user.email,
        )

        return order

    @staticmethod
    def cancel_order(*, order: Order, reason: str = "") -> Order:
        """Cancel an order and restore inventory."""
        if order.status not in (Order.Status.PENDING, Order.Status.CONFIRMED):
            raise ValueError(
                f"Cannot cancel order in '{order.get_status_display()}' status."
            )

        with transaction.atomic():
            # Restore inventory
            for item in order.items.select_related("product"):
                product = Product.objects.select_for_update().get(id=item.product_id)
                product.stock += item.quantity
                product.save(update_fields=["stock"])

            order.status = Order.Status.CANCELLED
            order.cancelled_at = timezone.now()
            order.cancellation_reason = reason
            order.save(update_fields=["status", "cancelled_at", "cancellation_reason"])

        return order
```

```python
# myproject/orders/selectors.py
"""
Selectors: read-only query logic separated from services (write logic).
"""
from __future__ import annotations

from django.db.models import QuerySet, Sum, F

from myproject.orders.models import Order


def get_user_orders(
    *,
    user: "User",
    status: str | None = None,
) -> QuerySet[Order]:
    """Return orders for a user, optionally filtered by status."""
    qs = Order.objects.filter(user=user).select_related("user")
    if status is not None:
        qs = qs.filter(status=status)
    return qs.order_by("-created_at")


def get_order_with_items(*, order_id: int) -> Order:
    """Return a single order with prefetched items and products."""
    return (
        Order.objects
        .select_related("user")
        .prefetch_related("items__product")
        .get(id=order_id)
    )


def get_revenue_by_period(*, year: int, month: int | None = None) -> dict:
    """Calculate total revenue for a given period."""
    qs = Order.objects.filter(
        status=Order.Status.COMPLETED,
        created_at__year=year,
    )
    if month is not None:
        qs = qs.filter(created_at__month=month)

    return qs.aggregate(
        total_revenue=Sum("total"),
        order_count=models.Count("id"),
    )
```

Usage in views:

```python
# myproject/orders/views.py
from django.contrib.auth.decorators import login_required
from django.http import HttpRequest, HttpResponse
from django.shortcuts import redirect, render

from myproject.orders.selectors import get_user_orders
from myproject.orders.services import OrderService


@login_required
def order_list(request: HttpRequest) -> HttpResponse:
    orders = get_user_orders(user=request.user)
    return render(request, "orders/list.html", {"orders": orders})


@login_required
def create_order(request: HttpRequest) -> HttpResponse:
    if request.method == "POST":
        form = OrderForm(request.POST)
        if form.is_valid():
            order = OrderService.create_order(
                user=request.user,
                items=form.cleaned_data["items"],
                shipping_address=form.cleaned_data["shipping_address"],
            )
            return redirect("orders:detail", pk=order.pk)
    else:
        form = OrderForm()
    return render(request, "orders/create.html", {"form": form})
```

### 4.3 Domain-Driven Design with Django

For large projects, align your apps with business domains:

```
myproject/
├── identity/          # Authentication, profiles, permissions
│   ├── users/
│   └── permissions/
├── catalog/           # Product catalog
│   ├── products/
│   └── categories/
├── commerce/          # Order processing
│   ├── orders/
│   ├── payments/
│   └── shipping/
└── engagement/        # User engagement
    ├── notifications/
    └── analytics/
```

Key DDD principles in Django:
- **Bounded contexts** map to groups of apps
- **Aggregates** map to models with related objects
- **Services** handle cross-aggregate operations
- **Domain events** can use Django signals or the Tasks framework
- Keep inter-domain communication through well-defined interfaces (service methods, not direct ORM access across domains)

### 4.4 When to Use Single App vs Multiple Apps

**Use a single app when:**
- The domain is small and cohesive
- All models are tightly coupled
- Fewer than 5-6 models that all relate to the same concept

**Split into multiple apps when:**
- You can describe separate responsibilities
- Models could logically exist independently
- You want reusability (an app could be extracted as a package)
- Different team members own different parts

---

## 5. Settings and Configuration

### 5.1 django-environ for 12-Factor Config

Use `django-environ` to load configuration from environment variables and `.env` files:

```python
# config/settings/base.py
import environ

# Build paths
BASE_DIR = environ.Path(__file__) - 3  # Three levels up from settings file
APPS_DIR = BASE_DIR.path("myproject")

# Initialize environ
env = environ.Env(
    # Set casting and defaults
    DJANGO_DEBUG=(bool, False),
    DJANGO_SECRET_KEY=(str, "CHANGEME"),
    DJANGO_ALLOWED_HOSTS=(list, []),
)

# Read .env file if it exists
env.read_env(str(BASE_DIR.path(".env")))

# SECURITY
SECRET_KEY = env("DJANGO_SECRET_KEY")
DEBUG = env("DJANGO_DEBUG")
ALLOWED_HOSTS = env("DJANGO_ALLOWED_HOSTS")
```

### 5.2 Multiple Settings Files Pattern

```
config/settings/
├── __init__.py       # Empty
├── base.py           # Shared settings (installed apps, middleware, etc.)
├── local.py          # Development (DEBUG=True, debug toolbar, etc.)
├── production.py     # Production (security, caching, email, etc.)
└── test.py           # Testing (fast passwords, in-memory cache, etc.)
```

**`base.py`** — Shared settings:

```python
# config/settings/base.py
import environ

BASE_DIR = environ.Path(__file__) - 3
APPS_DIR = BASE_DIR.path("myproject")
env = environ.Env()

# APPS
DJANGO_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
]
THIRD_PARTY_APPS = [
    "allauth",
    "allauth.account",
    "allauth.socialaccount",
    "corsheaders",
    "django_filters",
    "rest_framework",
]
LOCAL_APPS = [
    "myproject.core",
    "myproject.users",
    "myproject.articles",
]
INSTALLED_APPS = DJANGO_APPS + THIRD_PARTY_APPS + LOCAL_APPS

# MIDDLEWARE
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",
    "corsheaders.middleware.CorsMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.auth.middleware.LoginRequiredMiddleware",  # Django 5.1+
    "django.middleware.csp.ContentSecurityPolicyMiddleware",   # Django 6.0+
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
    "allauth.account.middleware.AccountMiddleware",
]

# DATABASE
DATABASES = {
    "default": env.db("DATABASE_URL", default="postgres:///myproject"),
}
DATABASES["default"]["ATOMIC_REQUESTS"] = True
DATABASES["default"]["CONN_MAX_AGE"] = env.int("CONN_MAX_AGE", default=60)
DATABASES["default"]["CONN_HEALTH_CHECKS"] = True

# Default primary key
DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"

# AUTH
AUTH_USER_MODEL = "users.User"
LOGIN_REDIRECT_URL = "/"
LOGIN_URL = "/accounts/login/"

# TEMPLATES
TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [str(APPS_DIR.path("templates"))],
        "APP_DIRS": True,
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.debug",
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",
                "django.contrib.messages.context_processors.messages",
                "django.template.context_processors.csp",  # Django 6.0+ CSP nonces
            ],
        },
    },
]

# STATIC
STATIC_URL = "/static/"
STATIC_ROOT = str(BASE_DIR.path("staticfiles"))
STATICFILES_DIRS = [str(APPS_DIR.path("static"))]

# MEDIA
MEDIA_URL = "/media/"
MEDIA_ROOT = str(APPS_DIR.path("media"))

# STORAGES (Django 4.2+)
STORAGES = {
    "default": {
        "BACKEND": "django.core.files.storage.FileSystemStorage",
    },
    "staticfiles": {
        "BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage",
    },
}

# CACHES
CACHES = {
    "default": env.cache("REDIS_URL", default="redis://localhost:6379/0"),
}

# EMAIL
EMAIL_BACKEND = env(
    "DJANGO_EMAIL_BACKEND",
    default="django.core.mail.backends.smtp.EmailBackend",
)

# INTERNATIONALIZATION
TIME_ZONE = "UTC"
USE_I18N = True
USE_TZ = True

# SECURITY
CSRF_COOKIE_HTTPONLY = True
SESSION_COOKIE_HTTPONLY = True
X_FRAME_OPTIONS = "DENY"

# CONTENT SECURITY POLICY (Django 6.0+)
from django.utils.csp import CSP

SECURE_CSP = {
    "default-src": [CSP.SELF],
    "script-src": [CSP.SELF, CSP.NONCE],
    "style-src": [CSP.SELF, CSP.UNSAFE_INLINE],
    "img-src": [CSP.SELF, "data:", "https:"],
    "font-src": [CSP.SELF],
    "connect-src": [CSP.SELF],
    "frame-ancestors": [CSP.NONE],
}

# TASKS (Django 6.0+)
TASKS = {
    "default": {
        "BACKEND": "django.tasks.backends.immediate.ImmediateBackend",
    },
}
```

**`local.py`** — Development settings:

```python
# config/settings/local.py
from .base import *  # noqa: F401,F403
from .base import env

DEBUG = True
SECRET_KEY = env("DJANGO_SECRET_KEY", default="local-secret-key-not-for-production")
ALLOWED_HOSTS = ["localhost", "0.0.0.0", "127.0.0.1"]

# Debug toolbar
INSTALLED_APPS += ["debug_toolbar"]  # noqa: F405
MIDDLEWARE.insert(0, "debug_toolbar.middleware.DebugToolbarMiddleware")  # noqa: F405
INTERNAL_IPS = ["127.0.0.1", "10.0.2.2"]

# Email - use Mailpit
EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
EMAIL_HOST = "localhost"
EMAIL_PORT = 1025

# Disable connection pooling in dev (not compatible with DEBUG mode)
DATABASES["default"]["OPTIONS"] = {}  # noqa: F405

# CSP - relaxed for dev
SECURE_CSP_REPORT_ONLY = SECURE_CSP  # noqa: F405
SECURE_CSP = {}  # Don't enforce in dev

# Celery
CELERY_TASK_ALWAYS_EAGER = True
```

**`production.py`** — Production settings:

```python
# config/settings/production.py
from .base import *  # noqa: F401,F403
from .base import env

# SECURITY
SECRET_KEY = env("DJANGO_SECRET_KEY")
ALLOWED_HOSTS = env.list("DJANGO_ALLOWED_HOSTS")
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000  # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SECURE_CONTENT_TYPE_NOSNIFF = True

# DATABASE - with connection pooling
DATABASES["default"]["OPTIONS"] = {  # noqa: F405
    "pool": {
        "min_size": 2,
        "max_size": 10,
        "timeout": 10,
    },
}

# STATIC FILES - S3 example (alternative to Whitenoise)
# STORAGES = {
#     "default": {
#         "BACKEND": "storages.backends.s3boto3.S3Boto3Storage",
#         "OPTIONS": {
#             "bucket_name": env("AWS_STORAGE_BUCKET_NAME"),
#             "region_name": env("AWS_S3_REGION_NAME"),
#         },
#     },
#     "staticfiles": {
#         "BACKEND": "storages.backends.s3boto3.S3StaticStorage",
#         "OPTIONS": {
#             "bucket_name": env("AWS_STATIC_BUCKET_NAME"),
#         },
#     },
# }

# EMAIL
EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
EMAIL_HOST = env("EMAIL_HOST")
EMAIL_PORT = env.int("EMAIL_PORT", default=587)
EMAIL_USE_TLS = True
EMAIL_HOST_USER = env("EMAIL_HOST_USER")
EMAIL_HOST_PASSWORD = env("EMAIL_HOST_PASSWORD")

# LOGGING
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "json": {
            "()": "structlog.stdlib.ProcessorFormatter",
            "processor": "structlog.dev.ConsoleRenderer",
        },
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "json",
        },
    },
    "root": {
        "handlers": ["console"],
        "level": "INFO",
    },
    "loggers": {
        "django": {
            "handlers": ["console"],
            "level": "INFO",
            "propagate": False,
        },
        "myproject": {
            "handlers": ["console"],
            "level": "INFO",
            "propagate": False,
        },
    },
}

# SENTRY
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration
from sentry_sdk.integrations.celery import CeleryIntegration

sentry_sdk.init(
    dsn=env("SENTRY_DSN"),
    integrations=[DjangoIntegration(), CeleryIntegration()],
    traces_sample_rate=env.float("SENTRY_TRACES_SAMPLE_RATE", default=0.1),
    send_default_pii=True,
)
```

**`test.py`** — Test settings:

```python
# config/settings/test.py
from .base import *  # noqa: F401,F403

SECRET_KEY = "test-secret-key-not-for-production"
DEBUG = False

# Fast password hashing in tests
PASSWORD_HASHERS = ["django.contrib.auth.hashers.MD5PasswordHasher"]

# In-memory cache for tests
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.locmem.LocMemCache",
    }
}

# Email
EMAIL_BACKEND = "django.core.mail.backends.locmem.EmailBackend"

# Tasks - use dummy backend in tests
TASKS = {
    "default": {
        "BACKEND": "django.tasks.backends.dummy.DummyBackend",
    },
}

# Celery
CELERY_TASK_ALWAYS_EAGER = True
```

### 5.3 Environment Variables for ALL Secrets

**Never hardcode secrets.** The following must always come from environment variables:
- `SECRET_KEY`
- `DATABASE_URL` / database credentials
- API keys (Stripe, AWS, etc.)
- Email credentials
- `SENTRY_DSN`
- OAuth client IDs/secrets
- Any third-party service credentials

---

## 6. Models

### 6.1 Always Set default_auto_field

Since Django 3.2, you should explicitly configure the default primary key type. Use `BigAutoField` to avoid integer overflow issues on large tables:

```python
# config/settings/base.py
DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"
```

> **Note**: As of Django 6.0, new projects created with `startproject` no longer set this in settings because `BigAutoField` is the default. However, existing projects that migrated from older Django versions may still have `AutoField` as the implicit default. Always verify.

### 6.2 db_default for Database-Computed Defaults (5.0+)

`db_default` sets default values at the database level, ensuring consistency even when rows are inserted outside Django's ORM:

```python
from django.db import models
from django.db.models.functions import Now, Pi


class Event(models.Model):
    name = models.CharField(max_length=200)
    # Database-computed default — the DB sets the timestamp
    created_at = models.DateTimeField(db_default=Now())
    # Literal default at DB level
    status = models.CharField(max_length=20, db_default="draft")
    # You can combine db_default with default
    priority = models.IntegerField(
        default=0,       # Used when creating instances in Python
        db_default=0,    # Used when inserting via raw SQL or migrations
    )
```

**When to use `db_default` vs `default`:**
- `db_default`: For values that should be set by the database (timestamps, UUIDs, sequences). Ensures consistency across all insert paths.
- `default`: For values that should be available immediately in Python before `save()`.
- Both together: Python-level default takes precedence for ORM-created objects; DB default applies for raw SQL and migration `AddField`.

### 6.3 GeneratedField for Computed Columns (5.0+)

`GeneratedField` creates database-computed columns that are automatically maintained by the database engine:

```python
from django.db import models
from django.db.models import F, Q, Value
from django.db.models.functions import Concat, Lower, Now, ExtractWeekDay


class Employee(models.Model):
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100)
    hire_date = models.DateField()
    salary = models.DecimalField(max_digits=10, decimal_places=2)
    bonus_rate = models.DecimalField(max_digits=5, decimal_places=4, default=0.1)

    # Auto-computed full name
    full_name = models.GeneratedField(
        expression=Concat(
            F("first_name"), Value(" "), F("last_name"),
        ),
        output_field=models.CharField(max_length=201),
        db_persist=True,  # Stored on disk (not virtual)
    )

    # Auto-computed lowercase email slug
    email_slug = models.GeneratedField(
        expression=Lower(
            Concat(F("first_name"), Value("."), F("last_name"))
        ),
        output_field=models.CharField(max_length=201),
        db_persist=True,
    )

    # Auto-computed total compensation
    total_compensation = models.GeneratedField(
        expression=F("salary") * (1 + F("bonus_rate")),
        output_field=models.DecimalField(max_digits=12, decimal_places=2),
        db_persist=True,
    )
```

**Key points:**
- `db_persist=True`: Column is stored on disk (recommended for indexed/queried fields).
- `db_persist=False`: Virtual column computed on each read (not supported by all backends).
- GeneratedFields are **read-only** — you cannot assign values to them.
- In Django 6.0+, GeneratedFields are automatically refreshed after `save()` on backends supporting `RETURNING` (PostgreSQL, SQLite, Oracle).

### 6.4 Composite Primary Keys (5.2+)

Django 5.2 introduced composite primary keys for tables where a single column PK doesn't make sense:

```python
class Product(models.Model):
    name = models.CharField(max_length=100)


class Order(models.Model):
    reference = models.CharField(max_length=20, primary_key=True)


class OrderLineItem(models.Model):
    pk = models.CompositePrimaryKey("product_id", "order_id")
    product = models.ForeignKey(Product, on_delete=models.CASCADE)
    order = models.ForeignKey(Order, on_delete=models.CASCADE)
    quantity = models.IntegerField()
```

The composite PK is represented as a tuple:

```python
>>> item = OrderLineItem.objects.get(pk=(1, "ORD-001"))
>>> item.pk
(1, "ORD-001")
```

**Limitations:**
- Cannot be used as a `ForeignKey` target from another model.
- Excluded from `ModelForm` by default.
- Primary key fields are read-only after creation.

### 6.5 Model Constraints and Validation

Use database-level constraints for data integrity:

```python
from django.db import models
from django.db.models import Q, F, CheckConstraint, UniqueConstraint


class Event(models.Model):
    name = models.CharField(max_length=200)
    start_date = models.DateTimeField()
    end_date = models.DateTimeField()
    capacity = models.PositiveIntegerField()
    registered = models.PositiveIntegerField(default=0)

    class Meta:
        constraints = [
            # End date must be after start date
            CheckConstraint(
                check=Q(end_date__gt=F("start_date")),
                name="event_end_after_start",
                violation_error_message="End date must be after start date.",
            ),
            # Cannot over-register
            CheckConstraint(
                check=Q(registered__lte=F("capacity")),
                name="event_not_overbooked",
                violation_error_message="Registered count exceeds capacity.",
            ),
            # Unique event name per date
            UniqueConstraint(
                fields=["name", "start_date"],
                name="unique_event_per_date",
            ),
            # Unique name for active events only (partial unique index)
            UniqueConstraint(
                fields=["name"],
                condition=Q(is_cancelled=False),
                name="unique_active_event_name",
            ),
        ]
```

### 6.6 Enumeration Types for Choices

Always use `TextChoices` or `IntegerChoices` instead of raw tuples:

```python
from django.db import models


class Article(models.Model):
    class Status(models.TextChoices):
        DRAFT = "draft", "Draft"
        REVIEW = "review", "In Review"
        PUBLISHED = "published", "Published"
        ARCHIVED = "archived", "Archived"

    class Priority(models.IntegerChoices):
        LOW = 1, "Low"
        MEDIUM = 2, "Medium"
        HIGH = 3, "High"
        URGENT = 4, "Urgent"

    title = models.CharField(max_length=200)
    status = models.CharField(
        max_length=20,
        choices=Status,
        default=Status.DRAFT,
        db_default=Status.DRAFT,
    )
    priority = models.IntegerField(
        choices=Priority,
        default=Priority.MEDIUM,
    )

# Usage
article = Article(title="Test", status=Article.Status.PUBLISHED)
article.status == Article.Status.PUBLISHED  # True
Article.Status.PUBLISHED.label  # "Published"
Article.Status.PUBLISHED.value  # "published"
```

### 6.7 TimeStampedModel Pattern

Create a base model for timestamp tracking:

```python
# myproject/core/models.py
from django.db import models
from django.db.models.functions import Now


class TimeStampedModel(models.Model):
    """Abstract base model with created/modified timestamps."""
    created_at = models.DateTimeField(db_default=Now(), editable=False)
    modified_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True
        ordering = ["-created_at"]


class SoftDeleteModel(models.Model):
    """Abstract model for soft deletion."""
    is_deleted = models.BooleanField(default=False, db_index=True)
    deleted_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        abstract = True

    def soft_delete(self) -> None:
        from django.utils import timezone
        self.is_deleted = True
        self.deleted_at = timezone.now()
        self.save(update_fields=["is_deleted", "deleted_at"])


# Usage
class Article(TimeStampedModel, SoftDeleteModel):
    title = models.CharField(max_length=200)
    body = models.TextField()
```

### 6.8 Fat Models with Service Layer for Complex Logic

The "fat models" philosophy says models should encapsulate their own behavior. However, when logic spans multiple models or involves external services, use a service layer:

**Keep in the model:**
- Simple computed properties
- Validation logic (`clean()`)
- String representation (`__str__`)
- URL generation (`get_absolute_url`)
- Simple state transitions

**Move to service layer:**
- Logic involving multiple models/aggregates
- External API calls
- Complex transactions
- Business rules that change frequently
- Side effects (sending emails, triggering tasks)

```python
class Order(TimeStampedModel):
    """Model contains simple, self-contained logic."""

    class Status(models.TextChoices):
        PENDING = "pending"
        CONFIRMED = "confirmed"
        SHIPPED = "shipped"
        DELIVERED = "delivered"
        CANCELLED = "cancelled"

    user = models.ForeignKey("users.User", on_delete=models.CASCADE)
    status = models.CharField(max_length=20, choices=Status, default=Status.PENDING)
    total = models.DecimalField(max_digits=10, decimal_places=2, default=0)

    @property
    def is_cancellable(self) -> bool:
        """Simple property stays on the model."""
        return self.status in (self.Status.PENDING, self.Status.CONFIRMED)

    def __str__(self) -> str:
        return f"Order #{self.pk} ({self.get_status_display()})"

    # Complex cancel logic with side effects -> goes in OrderService.cancel_order()
```

---

## 7. Queries and Database

### 7.1 Async ORM (aget, acreate, afilter, etc.)

Since Django 4.1, async versions of all ORM methods are available. In Django 6.x, async ORM is mature and production-ready.

```python
# Async view using async ORM
from django.http import JsonResponse


async def article_list(request):
    articles = []
    async for article in Article.objects.filter(published=True)[:10]:
        articles.append({
            "id": article.id,
            "title": article.title,
        })
    return JsonResponse({"articles": articles})


# Async ORM methods
article = await Article.objects.aget(pk=1)
article = await Article.objects.acreate(title="New Article")
exists = await Article.objects.filter(status="published").aexists()
count = await Article.objects.acount()
await article.asave()
await article.adelete()

# Async iteration
async for article in Article.objects.filter(published=True):
    print(article.title)

# Async aggregation
from django.db.models import Count
result = await Article.objects.aaggregate(total=Count("id"))
```

**When to use async ORM:**
- In async views served by ASGI
- When calling multiple independent queries (can be parallelized)
- In WebSocket consumers
- When integrating with external async APIs

**When NOT to use async ORM:**
- In standard WSGI deployments (adds overhead without benefit)
- For simple CRUD with few queries per request
- When all queries are sequential anyway (async won't help)

### 7.2 select_related and prefetch_related Best Practices

These are your primary tools for solving the N+1 query problem:

```python
# BAD: N+1 queries
articles = Article.objects.all()
for article in articles:
    print(article.author.name)  # Hits DB for each article!

# GOOD: 1 query with JOIN
articles = Article.objects.select_related("author").all()
for article in articles:
    print(article.author.name)  # No extra queries

# select_related: for ForeignKey and OneToOneField (SQL JOIN)
Article.objects.select_related("author", "category")

# prefetch_related: for ManyToManyField and reverse ForeignKey (separate query)
Article.objects.prefetch_related("tags", "comments")

# Combine them
Article.objects.select_related("author").prefetch_related(
    "tags",
    "comments__author",  # Nested prefetch
)

# Custom Prefetch for filtering
from django.db.models import Prefetch

Article.objects.prefetch_related(
    Prefetch(
        "comments",
        queryset=Comment.objects.filter(is_approved=True).select_related("author"),
        to_attr="approved_comments",  # Access as article.approved_comments
    )
)
```

**Rules of thumb:**
- `select_related`: Use for ForeignKey and OneToOne — produces a JOIN (single query).
- `prefetch_related`: Use for ManyToMany and reverse ForeignKey — produces a separate query with `IN` clause.
- Always check queries in Django Debug Toolbar during development.
- Use `only()` and `defer()` to limit columns fetched when you don't need all fields.

### 7.3 Database Functions and Expressions

Django provides a rich set of database functions that translate to SQL:

```python
from django.db.models import F, Q, Value, Case, When, Count, Sum, Avg
from django.db.models.functions import (
    Coalesce, Concat, Length, Lower, Upper, Now, TruncMonth,
    ExtractYear, Greatest, Least,
)

# F expressions for database-level comparisons
Product.objects.filter(stock__lt=F("reorder_level"))

# Annotate with computed values
Article.objects.annotate(
    title_length=Length("title"),
    word_count=Length("body") / 5,  # Rough word count
)

# Conditional expressions
Order.objects.annotate(
    urgency=Case(
        When(priority=Order.Priority.URGENT, then=Value("ASAP")),
        When(priority=Order.Priority.HIGH, then=Value("Today")),
        default=Value("Normal"),
        output_field=models.CharField(),
    )
)

# Aggregation with filtering
Order.objects.aggregate(
    total_revenue=Sum("total"),
    avg_order_value=Avg("total"),
    completed_count=Count("id", filter=Q(status="completed")),
    pending_count=Count("id", filter=Q(status="pending")),
)

# Group by with annotations
from django.db.models.functions import TruncMonth

Order.objects.annotate(
    month=TruncMonth("created_at")
).values("month").annotate(
    revenue=Sum("total"),
    count=Count("id"),
).order_by("month")

# Subqueries
from django.db.models import Subquery, OuterRef

latest_comment = Comment.objects.filter(
    article=OuterRef("pk")
).order_by("-created_at")

Article.objects.annotate(
    latest_comment_text=Subquery(latest_comment.values("text")[:1]),
)
```

### 7.4 Indexes Strategy

Proper indexing is critical for performance:

```python
from django.db import models


class Article(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)  # Unique implies index
    status = models.CharField(max_length=20, db_index=True)  # Simple index
    author = models.ForeignKey("users.User", on_delete=models.CASCADE)  # FK auto-indexed
    created_at = models.DateTimeField(auto_now_add=True)
    published_at = models.DateTimeField(null=True, blank=True)
    category = models.CharField(max_length=50)

    class Meta:
        indexes = [
            # Composite index for common query patterns
            models.Index(
                fields=["status", "published_at"],
                name="idx_article_status_published",
            ),
            # Index with ordering for common sorts
            models.Index(
                fields=["author", "-created_at"],
                name="idx_article_author_created",
            ),
            # Partial index (only index published articles)
            models.Index(
                fields=["published_at"],
                condition=Q(status="published"),
                name="idx_article_published_date",
            ),
            # Covering index (include extra columns to avoid table lookups)
            models.Index(
                fields=["slug"],
                include=["title", "status"],
                name="idx_article_slug_covering",
            ),
            # GIN index for full-text search (PostgreSQL)
            # GinIndex(fields=["search_vector"], name="idx_article_search"),
        ]
```

**Index guidelines:**
- Index columns used in `WHERE`, `ORDER BY`, and `JOIN` clauses.
- Use composite indexes for queries that filter on multiple columns.
- Use partial indexes to reduce index size when you only query a subset.
- Use covering indexes to satisfy queries entirely from the index.
- Don't over-index — each index slows down writes.
- Use `QuerySet.explain()` to verify index usage.

### 7.5 Transaction Management

```python
from django.db import transaction

# APPROACH 1: ATOMIC_REQUESTS (recommended as default)
# In settings: DATABASES["default"]["ATOMIC_REQUESTS"] = True
# Every view is wrapped in a transaction. If the view raises an exception,
# the transaction is rolled back.

# To exclude a specific view from ATOMIC_REQUESTS:
from django.db import transaction

@transaction.non_atomic_requests
def my_view(request):
    ...

# APPROACH 2: Explicit transactions
@transaction.atomic
def transfer_funds(from_account, to_account, amount):
    from_account.balance -= amount
    from_account.save(update_fields=["balance"])
    to_account.balance += amount
    to_account.save(update_fields=["balance"])

# APPROACH 3: Transaction as context manager (for partial transactions)
def complex_view(request):
    # This part is NOT in a transaction
    log_access(request)

    with transaction.atomic():
        # This part IS in a transaction
        order = Order.objects.create(user=request.user)
        OrderItem.objects.create(order=order, product=product)

    # This part is NOT in a transaction
    send_confirmation.enqueue(order_id=order.id)

# Savepoints
with transaction.atomic():
    order = Order.objects.create(user=request.user)
    try:
        with transaction.atomic():  # Creates a savepoint
            apply_discount(order)  # Might fail
    except DiscountError:
        pass  # Savepoint rolled back, order still exists

# on_commit - run code after transaction commits
from functools import partial

with transaction.atomic():
    order = Order.objects.create(user=request.user)
    transaction.on_commit(
        partial(send_notification.enqueue, order_id=order.id)
    )
```

### 7.6 Connection Pooling (Django 5.1+)

Django 5.1 introduced native PostgreSQL connection pooling via psycopg 3:

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "myproject",
        "USER": "myproject",
        "PASSWORD": "myproject",
        "HOST": "localhost",
        "PORT": "5432",
        "OPTIONS": {
            "pool": {
                "min_size": 2,    # Minimum connections to keep open
                "max_size": 10,   # Maximum connections in the pool
                "timeout": 10,    # Seconds to wait for a connection
            },
        },
    },
}
```

**Key considerations:**
- Connection pooling is per-process, not per-server. With 4 Gunicorn workers, each gets its own pool.
- For small/medium apps, Django's built-in pooling replaces PgBouncer.
- For large-scale deployments (50+ average connections), consider PgBouncer at the infrastructure level.
- **Disable pooling when `DEBUG=True`** — it can cause issues with the development server.

### 7.7 QuerySet.explain() for Debugging

Use `explain()` to see the database execution plan:

```python
# Basic explain
print(Article.objects.filter(status="published").explain())

# Verbose explain with analysis (PostgreSQL)
print(
    Article.objects.filter(status="published")
    .explain(verbose=True, analyze=True)
)

# Check if index is used
qs = Article.objects.filter(status="published", category="tech")
plan = qs.explain(format="json")
print(plan)
```

---

## 8. Views (FBVs and CBVs)

### 8.1 LoginRequiredMiddleware (Django 5.1+)

Django 5.1 introduced `LoginRequiredMiddleware` to require login by default for all views:

```python
# config/settings/base.py
MIDDLEWARE = [
    # ...
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.auth.middleware.LoginRequiredMiddleware",  # After AuthenticationMiddleware
    # ...
]
```

Then mark public views explicitly:

```python
from django.contrib.auth.decorators import login_not_required
from django.http import HttpResponse


@login_not_required
def public_homepage(request):
    """This view is accessible without login."""
    return HttpResponse("Welcome!")


@login_not_required
def health_check(request):
    """Health check endpoint for load balancers."""
    return HttpResponse("OK")


# All other views automatically require login
def dashboard(request):
    """This requires login without any decorator."""
    return render(request, "dashboard.html")
```

For class-based views:

```python
from django.utils.decorators import method_decorator
from django.contrib.auth.decorators import login_not_required
from django.views import View


@method_decorator(login_not_required, name="dispatch")
class PublicView(View):
    ...
```

> **Note**: When using `LoginRequiredMiddleware` with Django REST Framework, be aware that DRF's authentication happens at the view level, after Django middleware. You may need to mark API views as `@login_not_required` and rely on DRF's own permission classes.

### 8.2 When to Use FBVs vs CBVs

**Use Function-Based Views (FBVs) when:**
- Simple, one-off views with clear logic
- Custom behavior that doesn't fit CBV patterns
- Views that process forms in a non-standard way
- API views with complex logic

```python
@login_not_required
def article_detail(request: HttpRequest, slug: str) -> HttpResponse:
    article = get_object_or_404(Article, slug=slug, status="published")
    comments = article.comments.filter(is_approved=True).select_related("author")
    return render(request, "articles/detail.html", {
        "article": article,
        "comments": comments,
    })
```

**Use Class-Based Views (CBVs) when:**
- Standard CRUD operations (ListView, DetailView, CreateView, UpdateView, DeleteView)
- You need mixins for shared behavior across views
- RESTful API views (DRF ViewSets)

```python
from django.views.generic import ListView, DetailView, CreateView, UpdateView


class ArticleListView(ListView):
    model = Article
    template_name = "articles/list.html"
    context_object_name = "articles"
    paginate_by = 20

    def get_queryset(self):
        return (
            Article.objects
            .filter(status=Article.Status.PUBLISHED)
            .select_related("author")
            .order_by("-published_at")
        )


class ArticleCreateView(CreateView):
    model = Article
    form_class = ArticleForm
    template_name = "articles/create.html"

    def form_valid(self, form):
        form.instance.author = self.request.user
        return super().form_valid(form)
```

### 8.3 Async Views (Fully Mature in Django 6.x)

Async views are production-ready in Django 6.x:

```python
import httpx
from django.http import JsonResponse


# Async FBV
async def weather_dashboard(request):
    """Fetch weather data from external API without blocking."""
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.weather.com/current")
        weather_data = response.json()

    # Async ORM
    user_prefs = await UserPreference.objects.aget(user=request.user)

    return JsonResponse({
        "weather": weather_data,
        "unit": user_prefs.temperature_unit,
    })


# Parallel async queries
import asyncio


async def dashboard(request):
    """Run multiple queries in parallel."""
    orders, notifications, stats = await asyncio.gather(
        sync_to_async(get_recent_orders)(request.user),
        Notification.objects.filter(user=request.user, read=False).acount(),
        sync_to_async(get_dashboard_stats)(request.user),
    )
    return render(request, "dashboard.html", {
        "orders": orders,
        "notification_count": notifications,
        "stats": stats,
    })
```

### 8.4 Keep Business Logic Out of Views

Views should be thin — they handle HTTP concerns and delegate to services:

```python
# BAD: Business logic in the view
def create_order(request):
    if request.method == "POST":
        form = OrderForm(request.POST)
        if form.is_valid():
            order = form.save(commit=False)
            order.user = request.user
            order.total = sum(item.price * item.qty for item in form.cleaned_data["items"])
            order.save()
            for item in form.cleaned_data["items"]:
                product = Product.objects.get(id=item.product_id)
                product.stock -= item.qty
                product.save()
                OrderItem.objects.create(order=order, product=product, ...)
            send_mail(...)  # Blocking!
            return redirect("orders:detail", pk=order.pk)

# GOOD: Delegate to service layer
def create_order(request):
    if request.method == "POST":
        form = OrderForm(request.POST)
        if form.is_valid():
            try:
                order = OrderService.create_order(
                    user=request.user,
                    items=form.cleaned_data["items"],
                    shipping_address=form.cleaned_data["shipping_address"],
                )
            except ValueError as e:
                form.add_error(None, str(e))
            else:
                return redirect("orders:detail", pk=order.pk)
    else:
        form = OrderForm()
    return render(request, "orders/create.html", {"form": form})
```

---

## 9. Forms

### 9.1 Modern Form Rendering (Template-Based Since 4.0)

Django 4.0 replaced programmatic form rendering with template-based rendering:

```html
<!-- Old way (still works but not recommended for new code) -->
{{ form.as_p }}
{{ form.as_table }}

<!-- Modern way: render individual fields with full control -->
<form method="post">
    {% csrf_token %}

    {% for field in form %}
    <div class="form-group {% if field.errors %}has-error{% endif %}">
        <label for="{{ field.id_for_label }}">{{ field.label }}</label>
        {{ field }}
        {% if field.help_text %}
            <small class="form-text text-muted">{{ field.help_text }}</small>
        {% endif %}
        {% for error in field.errors %}
            <div class="invalid-feedback">{{ error }}</div>
        {% endfor %}
    </div>
    {% endfor %}

    <button type="submit">Submit</button>
</form>
```

### 9.2 as_field_group (Django 5.0+)

`as_field_group` renders a complete field group (label + widget + help text + errors) using a customizable template:

```html
<!-- Renders each field using its field group template -->
<form method="post">
    {% csrf_token %}
    {{ form.title.as_field_group }}
    {{ form.body.as_field_group }}
    {{ form.status.as_field_group }}
    <button type="submit">Save</button>
</form>
```

You can override the field group template:

```python
# In your form
class ArticleForm(forms.ModelForm):
    class Meta:
        model = Article
        fields = ["title", "body", "status"]

    # Override template for a specific field
    title = forms.CharField(
        template_name="forms/custom_field.html",
    )
```

```html
<!-- templates/forms/custom_field.html -->
<div class="mb-3">
    {% if field.label %}
        <label class="form-label" for="{{ field.id_for_label }}">
            {{ field.label }}
            {% if field.field.required %}<span class="text-danger">*</span>{% endif %}
        </label>
    {% endif %}
    {{ field }}
    {% if field.help_text %}
        <div class="form-text">{{ field.help_text }}</div>
    {% endif %}
    {% for error in field.errors %}
        <div class="invalid-feedback d-block">{{ error }}</div>
    {% endfor %}
</div>
```

### 9.3 New Widgets: ColorInput, SearchInput, TelInput (Django 5.2+)

```python
from django import forms


class ProfileForm(forms.Form):
    # Renders as <input type="color">
    favorite_color = forms.CharField(
        widget=forms.ColorInput,
        required=False,
    )

    # Renders as <input type="tel">
    phone_number = forms.CharField(
        widget=forms.TelInput(attrs={
            "pattern": r"[0-9]{10}",
            "placeholder": "1234567890",
        }),
    )

    # Renders as <input type="search">
    search_query = forms.CharField(
        widget=forms.SearchInput(attrs={
            "placeholder": "Search...",
        }),
        required=False,
    )
```

### 9.4 Always Validate Incoming Data with Forms

**Never trust raw request data.** Always use forms for validation:

```python
# BAD: Using raw POST data
def update_profile(request):
    user = request.user
    user.email = request.POST["email"]  # No validation!
    user.save()

# GOOD: Validate with a form
def update_profile(request):
    if request.method == "POST":
        form = ProfileForm(request.POST, instance=request.user)
        if form.is_valid():
            form.save()
            return redirect("profile")
    else:
        form = ProfileForm(instance=request.user)
    return render(request, "profile/edit.html", {"form": form})
```

Even for API data, validate:

```python
# For API views without DRF, use forms
def api_create_article(request):
    form = ArticleForm(json.loads(request.body))
    if form.is_valid():
        article = form.save(commit=False)
        article.author = request.user
        article.save()
        return JsonResponse({"id": article.id}, status=201)
    return JsonResponse({"errors": form.errors}, status=400)
```

### 9.5 CSRF Protection Patterns

```python
# Standard form submission - include {% csrf_token %}
```

```html
<form method="post">
    {% csrf_token %}
    {{ form.as_field_group }}
    <button type="submit">Save</button>
</form>
```

For AJAX requests:

```javascript
// Read the CSRF token from the cookie
function getCookie(name) {
    let cookieValue = null;
    if (document.cookie && document.cookie !== '') {
        const cookies = document.cookie.split(';');
        for (let i = 0; i < cookies.length; i++) {
            const cookie = cookies[i].trim();
            if (cookie.substring(0, name.length + 1) === (name + '=')) {
                cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                break;
            }
        }
    }
    return cookieValue;
}

// Include in fetch requests
fetch('/api/articles/', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-CSRFToken': getCookie('csrftoken'),
    },
    body: JSON.stringify(data),
});
```

---

## 10. Templates

### 10.1 Template Partials with {% partialdef %} (Django 6.0+)

Template partials allow you to define reusable template fragments within a file — a major improvement for component-based development:

```html
<!-- templates/articles/list.html -->
{% extends "base.html" %}

{% block content %}
<h1>Articles</h1>

{# Define a partial — it won't render here unless 'inline' is used #}
{% partialdef article_card %}
<div class="card mb-3">
    <div class="card-body">
        <h5 class="card-title">{{ article.title }}</h5>
        <p class="card-text">{{ article.excerpt }}</p>
        <a href="{{ article.get_absolute_url }}" class="btn btn-primary">Read More</a>
    </div>
</div>
{% endpartialdef %}

{# Render the partial for each article #}
{% for article in articles %}
    {% partial article_card %}
{% endfor %}

{% endblock %}
```

**Inline partials** — define and render immediately, but also make available for later use:

```html
{% partialdef sidebar_widget inline %}
<div class="widget">
    <h3>{{ widget_title }}</h3>
    {{ widget_content }}
</div>
{% endpartialdef %}

{# Can be reused elsewhere in the template #}
{% partial sidebar_widget %}
```

**Using partials for HTMX partial responses:**

```html
<!-- templates/articles/list.html -->
{% partialdef article_list_body %}
{% for article in articles %}
<tr>
    <td>{{ article.title }}</td>
    <td>{{ article.author.name }}</td>
    <td>{{ article.published_at|date:"M d, Y" }}</td>
</tr>
{% endfor %}
{% endpartialdef %}

{% if not request.htmx %}
{% extends "base.html" %}
{% block content %}
<table>
    <thead>
        <tr><th>Title</th><th>Author</th><th>Date</th></tr>
    </thead>
    <tbody id="article-list">
        {% partial article_list_body %}
    </tbody>
</table>
{% endblock %}
{% endif %}

{# When called via HTMX, only render the partial #}
{% if request.htmx %}
{% partial article_list_body %}
{% endif %}
```

### 10.2 {% querystring %} Tag (Django 5.1+)

The `{% querystring %}` tag simplifies pagination and filter URL generation:

```html
<!-- Pagination with preserved filters -->
<nav>
    {% if page_obj.has_previous %}
        <a href="{% querystring page=page_obj.previous_page_number %}">Previous</a>
    {% endif %}

    <span>Page {{ page_obj.number }} of {{ page_obj.paginator.num_pages }}</span>

    {% if page_obj.has_next %}
        <a href="{% querystring page=page_obj.next_page_number %}">Next</a>
    {% endif %}
</nav>

<!-- Filter links that preserve other query parameters -->
<div class="filters">
    <a href="{% querystring status='published' %}">Published</a>
    <a href="{% querystring status='draft' %}">Drafts</a>
    <a href="{% querystring status=None %}">All</a>  {# Removes the status param #}
</div>

<!-- Sort links -->
<th>
    <a href="{% querystring sort='title' %}">Title</a>
</th>
<th>
    <a href="{% querystring sort='-created_at' %}">Date</a>
</th>

<!-- Store in a variable -->
{% querystring page=1 as reset_url %}
<a href="{{ reset_url }}">Back to first page</a>
```

### 10.3 forloop.length (Django 6.0+)

Django 6.0 adds `forloop.length` to access the total number of iterations:

```html
{% for item in items %}
    <div class="item">
        {{ item.name }}
        <span class="badge">{{ forloop.counter }} of {{ forloop.length }}</span>
    </div>
{% endfor %}

<!-- Useful for conditional rendering -->
{% for article in articles %}
    <article>
        <h2>{{ article.title }}</h2>
        {% if forloop.length > 5 %}
            {# Show compact view when many articles #}
            <p>{{ article.excerpt|truncatewords:20 }}</p>
        {% else %}
            {# Show full excerpt when few articles #}
            <p>{{ article.excerpt }}</p>
        {% endif %}
    </article>
{% endfor %}
```

### 10.4 Template Architecture: 2-Tier vs 3-Tier

**2-Tier (Simple projects):**

```
templates/
├── base.html              # Master template
├── articles/
│   ├── list.html          # Extends base.html
│   └── detail.html        # Extends base.html
```

**3-Tier (Complex projects — recommended):**

```
templates/
├── base.html              # Site-wide: HTML structure, CSS/JS includes
├── layouts/
│   ├── dashboard.html     # Extends base.html: sidebar layout
│   ├── marketing.html     # Extends base.html: full-width layout
│   └── auth.html          # Extends base.html: centered card layout
├── articles/
│   ├── list.html          # Extends layouts/dashboard.html
│   └── detail.html        # Extends layouts/marketing.html
```

```html
<!-- base.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>{% block title %}My Site{% endblock %}</title>
    {% block css %}{% endblock %}
</head>
<body>
    {% block navbar %}{% include "includes/navbar.html" %}{% endblock %}
    {% block content %}{% endblock %}
    {% block javascript %}{% endblock %}
</body>
</html>

<!-- layouts/dashboard.html -->
{% extends "base.html" %}
{% block content %}
<div class="container-fluid">
    <div class="row">
        <nav class="col-md-2 sidebar">
            {% block sidebar %}{% include "includes/sidebar.html" %}{% endblock %}
        </nav>
        <main class="col-md-10">
            {% block page_content %}{% endblock %}
        </main>
    </div>
</div>
{% endblock %}

<!-- articles/list.html -->
{% extends "layouts/dashboard.html" %}
{% block page_content %}
<h1>Articles</h1>
...
{% endblock %}
```

### 10.5 Limit Processing in Templates

Templates should render data, not compute it:

```python
# BAD: Complex logic in templates
# template: {% if user.orders.count > 10 and user.orders.last.total > 100 %}

# GOOD: Compute in the view or model
# view:
context = {
    "is_premium_customer": user.orders.count() > 10
        and user.orders.last().total > 100,
}
# template: {% if is_premium_customer %}
```

Rules:
- No complex filtering or aggregation in templates
- No business logic in templates
- Use template tags/filters for reusable display logic only
- Pre-compute everything in views or context processors

### 10.6 Jinja2 Considerations

Django supports Jinja2 as an alternative template engine. Use it when:
- You need template logic that Django's template language doesn't support
- Performance is critical (Jinja2 is faster for complex templates)
- You're already familiar with Jinja2 from Flask/other projects

```python
# config/settings/base.py
TEMPLATES = [
    {
        "BACKEND": "django.template.backends.jinja2.Jinja2",
        "DIRS": [str(APPS_DIR.path("jinja2"))],
        "APP_DIRS": True,
        "OPTIONS": {
            "environment": "myproject.core.jinja2_env.environment",
        },
    },
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [str(APPS_DIR.path("templates"))],
        "APP_DIRS": True,
        "OPTIONS": {
            "context_processors": [...],
        },
    },
]
```

```python
# myproject/core/jinja2_env.py
from django.templatetags.static import static
from django.urls import reverse
from jinja2 import Environment


def environment(**options):
    env = Environment(**options)
    env.globals.update({
        "static": static,
        "url": reverse,
    })
    return env
```

> **Caveat**: Jinja2 templates cannot use Django's template tags (including the new `{% partialdef %}` and `{% querystring %}`). The admin always uses Django templates. Most teams stick with Django templates for consistency.

---

## 11. REST APIs

### 11.1 Django REST Framework Best Practices

DRF remains the standard for building REST APIs with Django:

```python
# myproject/articles/api/serializers.py
from rest_framework import serializers
from myproject.articles.models import Article


class ArticleSerializer(serializers.ModelSerializer):
    author_name = serializers.CharField(source="author.get_full_name", read_only=True)
    word_count = serializers.SerializerMethodField()

    class Meta:
        model = Article
        fields = [
            "id", "title", "slug", "body", "status",
            "author", "author_name", "word_count",
            "created_at", "published_at",
        ]
        read_only_fields = ["id", "slug", "author", "created_at"]

    def get_word_count(self, obj) -> int:
        return len(obj.body.split())

    def validate_title(self, value: str) -> str:
        if len(value) < 5:
            raise serializers.ValidationError("Title must be at least 5 characters.")
        return value


class ArticleCreateSerializer(serializers.ModelSerializer):
    """Separate serializer for creation with different field set."""

    class Meta:
        model = Article
        fields = ["title", "body", "status"]

    def create(self, validated_data):
        validated_data["author"] = self.context["request"].user
        return super().create(validated_data)
```

```python
# myproject/articles/api/views.py
from rest_framework import viewsets, permissions, filters
from rest_framework.decorators import action
from rest_framework.response import Response
from django_filters.rest_framework import DjangoFilterBackend

from myproject.articles.models import Article
from .serializers import ArticleSerializer, ArticleCreateSerializer


class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.select_related("author").all()
    permission_classes = [permissions.IsAuthenticated]
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_fields = ["status", "author"]
    search_fields = ["title", "body"]
    ordering_fields = ["created_at", "title"]
    ordering = ["-created_at"]

    def get_serializer_class(self):
        if self.action == "create":
            return ArticleCreateSerializer
        return ArticleSerializer

    def get_queryset(self):
        qs = super().get_queryset()
        if not self.request.user.is_staff:
            qs = qs.filter(status=Article.Status.PUBLISHED)
        return qs

    @action(detail=True, methods=["post"])
    def publish(self, request, pk=None):
        article = self.get_object()
        article.status = Article.Status.PUBLISHED
        article.save(update_fields=["status"])
        return Response(ArticleSerializer(article).data)
```

```python
# config/urls.py
from rest_framework.routers import DefaultRouter
from myproject.articles.api.views import ArticleViewSet

router = DefaultRouter()
router.register("articles", ArticleViewSet, basename="article")

urlpatterns = [
    path("api/v1/", include(router.urls)),
]
```

### 11.2 Django Ninja as a Modern Alternative

Django Ninja is a newer option that offers FastAPI-like syntax with Pydantic schemas:

```python
# myproject/articles/api.py
from ninja import Router, Schema
from datetime import datetime

from myproject.articles.models import Article

router = Router()


class ArticleOut(Schema):
    id: int
    title: str
    slug: str
    status: str
    created_at: datetime


class ArticleIn(Schema):
    title: str
    body: str
    status: str = "draft"


@router.get("/", response=list[ArticleOut])
def list_articles(request, status: str | None = None):
    qs = Article.objects.all()
    if status:
        qs = qs.filter(status=status)
    return qs


@router.post("/", response=ArticleOut)
def create_article(request, payload: ArticleIn):
    article = Article.objects.create(
        **payload.dict(),
        author=request.user,
    )
    return article


@router.get("/{article_id}", response=ArticleOut)
def get_article(request, article_id: int):
    return get_object_or_404(Article, id=article_id)
```

**DRF vs Django Ninja:**
- **DRF**: More mature, larger ecosystem, better for complex APIs with nested resources, permissions, pagination. The community standard.
- **Django Ninja**: Simpler, faster, auto-generated OpenAPI docs, Pydantic validation. Good for new microservices.

### 11.3 API Versioning Strategies

```python
# URL-based versioning (recommended)
urlpatterns = [
    path("api/v1/", include("myproject.api.v1.urls")),
    path("api/v2/", include("myproject.api.v2.urls")),
]

# DRF versioning settings
REST_FRAMEWORK = {
    "DEFAULT_VERSIONING_CLASS": "rest_framework.versioning.URLPathVersioning",
    "DEFAULT_VERSION": "v1",
    "ALLOWED_VERSIONS": ["v1", "v2"],
}
```

### 11.4 Rate Limiting

```python
# config/settings/base.py
REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.AnonRateThrottle",
        "rest_framework.throttling.UserRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {
        "anon": "100/hour",
        "user": "1000/hour",
    },
}
```

### 11.5 Authentication

```python
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.SessionAuthentication",  # For browser clients
        "rest_framework.authentication.TokenAuthentication",     # For API clients
        # Or use JWT:
        # "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
}

# For JWT (using djangorestframework-simplejwt)
from datetime import timedelta

SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=30),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=7),
    "ROTATE_REFRESH_TOKENS": True,
    "BLACKLIST_AFTER_ROTATION": True,
}
```

### 11.6 OpenAPI/Swagger Documentation

```python
# Using drf-spectacular (recommended)
INSTALLED_APPS += ["drf_spectacular"]

REST_FRAMEWORK = {
    "DEFAULT_SCHEMA_CLASS": "drf_spectacular.openapi.AutoSchema",
}

SPECTACULAR_SETTINGS = {
    "TITLE": "My Project API",
    "DESCRIPTION": "API documentation for My Project",
    "VERSION": "1.0.0",
    "SERVE_INCLUDE_SCHEMA": False,
}

# URLs
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView

urlpatterns = [
    path("api/schema/", SpectacularAPIView.as_view(), name="schema"),
    path("api/docs/", SpectacularSwaggerView.as_view(url_name="schema"), name="swagger-ui"),
]
```

---

## 12. Async Django

### 12.1 ASGI Deployment (Uvicorn/Daphne)

Django supports ASGI for async views, middleware, and WebSocket connections:

```python
# config/asgi.py
import os
from django.core.asgi import get_asgi_application

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.production")
application = get_asgi_application()
```

Deploy with Uvicorn via Gunicorn:

```bash
# Development
uvicorn config.asgi:application --reload --host 0.0.0.0 --port 8000

# Production (Gunicorn managing Uvicorn workers)
gunicorn config.asgi:application \
    -k uvicorn_worker.UvicornWorker \
    --bind 0.0.0.0:8000 \
    --workers 4 \
    --timeout 120 \
    --keep-alive 5 \
    --access-logfile - \
    --error-logfile -
```

### 12.2 Async Views, Middleware, and ORM

```python
# Async view
import httpx
from django.http import JsonResponse


async def external_data(request):
    async with httpx.AsyncClient() as client:
        # These requests happen concurrently
        import asyncio
        weather, news = await asyncio.gather(
            client.get("https://api.weather.com/data"),
            client.get("https://api.news.com/latest"),
        )
    return JsonResponse({
        "weather": weather.json(),
        "news": news.json(),
    })


# Async middleware
class TimingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    async def __call__(self, request):
        import time
        start = time.perf_counter()
        response = await self.get_response(request)
        duration = time.perf_counter() - start
        response["X-Request-Duration"] = f"{duration:.4f}"
        return response
```

### 12.3 WebSockets with Django Channels

```python
# Install: uv add channels channels-redis

# config/asgi.py
import os
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.auth import AuthMiddlewareStack
from django.core.asgi import get_asgi_application

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.production")

django_asgi_app = get_asgi_application()

from myproject.chat.routing import websocket_urlpatterns

application = ProtocolTypeRouter({
    "http": django_asgi_app,
    "websocket": AuthMiddlewareStack(
        URLRouter(websocket_urlpatterns)
    ),
})
```

```python
# myproject/chat/consumers.py
import json
from channels.generic.websocket import AsyncWebsocketConsumer


class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.room_name = self.scope["url_route"]["kwargs"]["room_name"]
        self.room_group_name = f"chat_{self.room_name}"

        await self.channel_layer.group_add(self.room_group_name, self.channel_name)
        await self.accept()

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(self.room_group_name, self.channel_name)

    async def receive(self, text_data):
        data = json.loads(text_data)
        await self.channel_layer.group_send(
            self.room_group_name,
            {
                "type": "chat.message",
                "message": data["message"],
                "username": self.scope["user"].username,
            },
        )

    async def chat_message(self, event):
        await self.send(text_data=json.dumps({
            "message": event["message"],
            "username": event["username"],
        }))
```

### 12.4 Background Tasks Framework (Django 6.0+)

Django 6.0 introduced a built-in Tasks framework that provides a standard way to define and enqueue background work:

```python
# myproject/notifications/tasks.py
from django.core.mail import send_mail
from django.tasks import task


@task
def send_order_confirmation_email(order_id: int, user_email: str):
    """Send order confirmation email in the background."""
    from myproject.orders.models import Order
    order = Order.objects.get(id=order_id)
    send_mail(
        subject=f"Order Confirmation #{order.id}",
        message=f"Your order has been placed. Total: ${order.total}",
        from_email=None,
        recipient_list=[user_email],
    )


@task(priority=2, queue_name="high")
def process_payment(payment_id: int):
    """Process payment with higher priority."""
    from myproject.payments.models import Payment
    payment = Payment.objects.get(id=payment_id)
    payment.process()


@task(takes_context=True)
def generate_report(context, report_type: str, user_id: int):
    """Task that uses context for logging/debugging."""
    import logging
    logger = logging.getLogger(__name__)
    logger.info(
        f"Generating {report_type} report. "
        f"Task ID: {context.task_result.id}, "
        f"Attempt: {context.attempt}"
    )
    # ... generate report ...
```

Enqueueing tasks:

```python
# In a view or service
from myproject.notifications.tasks import send_order_confirmation_email

# Simple enqueue
result = send_order_confirmation_email.enqueue(
    order_id=order.id,
    user_email=user.email,
)

# Modify priority before enqueueing
send_order_confirmation_email.using(priority=10).enqueue(
    order_id=order.id,
    user_email=user.email,
)

# Enqueue after transaction commits
from functools import partial
from django.db import transaction

with transaction.atomic():
    order = Order.objects.create(...)
    transaction.on_commit(
        partial(
            send_order_confirmation_email.enqueue,
            order_id=order.id,
            user_email=user.email,
        )
    )

# Async enqueue
result = await send_order_confirmation_email.aenqueue(
    order_id=order.id,
    user_email=user.email,
)

# Check result status
result.refresh()  # or await result.arefresh()
print(result.status)  # "SUCCESSFUL", "FAILED", "RUNNING", etc.
print(result.return_value)  # If successful
```

Configuration:

```python
# config/settings/base.py
TASKS = {
    "default": {
        # Development: runs tasks immediately (synchronously)
        "BACKEND": "django.tasks.backends.immediate.ImmediateBackend",
    },
}

# config/settings/production.py
TASKS = {
    "default": {
        # Use a third-party backend for production
        # e.g., django-tasks-scheduler, or a Celery-compatible backend
        "BACKEND": "path.to.production.TaskBackend",
    },
}

# config/settings/test.py
TASKS = {
    "default": {
        # Dummy backend: stores tasks without executing
        "BACKEND": "django.tasks.backends.dummy.DummyBackend",
    },
}
```

**Important**: Django's Tasks framework defines the API but does not ship a production-ready backend. For actual background execution, use a third-party backend like `django-tasks-local` (thread-based, good for low volume) or a database-backed worker system. For complex needs, Celery remains the standard.

### 12.5 When Async Helps and When It Doesn't

**Async helps when:**
- Making multiple external HTTP API calls (parallel requests)
- Handling WebSocket connections
- Serving high-concurrency APIs with many slow I/O operations
- Long-polling or server-sent events (SSE)
- Streaming responses

**Async doesn't help when:**
- Most of your time is spent in the database (Django ORM calls are wrapped with `sync_to_async` internally)
- Simple CRUD apps with low concurrency
- CPU-bound work (use multiprocessing instead)
- Your existing WSGI deployment works fine

**Practical guideline**: If your views mostly query the database and render templates, stay with WSGI (Gunicorn). Switch to ASGI only when you have a clear use case for concurrent I/O or WebSockets.

---

## 13. Background Tasks

### 13.1 Django's Built-in Tasks Framework (6.0+)

See Section 12.4 above for the full API. Key points:

- **Decorator API**: Use `@task` to define tasks as regular functions.
- **Arguments must be JSON-serializable**: No model instances, datetimes, or tuples as arguments. Pass IDs and re-fetch in the task.
- **Built-in backends are for development only**: `ImmediateBackend` (runs synchronously) and `DummyBackend` (does nothing).
- **Production requires a third-party backend**: The framework is intentionally minimal.

### 13.2 Celery for Complex Workflows

Celery remains essential for complex background processing:

```python
# config/celery.py
import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.production")

app = Celery("myproject")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()
```

```python
# config/settings/base.py
CELERY_BROKER_URL = env("CELERY_BROKER_URL", default="redis://localhost:6379/0")
CELERY_RESULT_BACKEND = env("CELERY_RESULT_BACKEND", default="redis://localhost:6379/0")
CELERY_ACCEPT_CONTENT = ["json"]
CELERY_TASK_SERIALIZER = "json"
CELERY_RESULT_SERIALIZER = "json"
CELERY_TIMEZONE = "UTC"
CELERY_TASK_TRACK_STARTED = True
CELERY_TASK_TIME_LIMIT = 300  # 5 minutes
CELERY_TASK_SOFT_TIME_LIMIT = 240  # 4 minutes (soft limit for graceful shutdown)
CELERY_WORKER_MAX_TASKS_PER_CHILD = 1000  # Restart worker after 1000 tasks (prevents memory leaks)

# Beat schedule for periodic tasks
CELERY_BEAT_SCHEDULE = {
    "cleanup-old-sessions": {
        "task": "myproject.core.tasks.cleanup_sessions",
        "schedule": 3600.0,  # Every hour
    },
    "send-daily-digest": {
        "task": "myproject.notifications.tasks.send_daily_digest",
        "schedule": crontab(hour=8, minute=0),  # Every day at 8 AM
    },
}
```

```python
# myproject/articles/tasks.py
from celery import shared_task
from celery.utils.log import get_task_logger

logger = get_task_logger(__name__)


@shared_task(
    bind=True,
    max_retries=3,
    default_retry_delay=60,
    autoretry_for=(ConnectionError,),
    retry_backoff=True,
)
def process_article_images(self, article_id: int):
    """Process and optimize article images with retry logic."""
    from myproject.articles.models import Article

    try:
        article = Article.objects.get(id=article_id)
        # ... process images ...
        logger.info(f"Processed images for article {article_id}")
    except Article.DoesNotExist:
        logger.warning(f"Article {article_id} not found")
    except Exception as exc:
        logger.error(f"Failed to process article {article_id}: {exc}")
        raise self.retry(exc=exc)


# Celery chains and groups
from celery import chain, group, chord

# Sequential pipeline
result = chain(
    fetch_data.s(url),
    process_data.s(),
    store_results.s(),
)()

# Parallel execution
result = group(
    process_image.s(img_id) for img_id in image_ids
)()

# Parallel then aggregate
result = chord(
    [process_chunk.s(chunk) for chunk in chunks],
    aggregate_results.s(),
)()
```

### 13.3 Django-Q2 as Alternative

Django-Q2 (successor to Django-Q) is a simpler alternative to Celery that uses Django's ORM as the broker:

```python
# Install: uv add django-q2

# settings
Q_CLUSTER = {
    "name": "myproject",
    "workers": 4,
    "recycle": 500,
    "timeout": 120,
    "django_redis": "default",
    "retry": 180,
}
```

### 13.4 Task Design Patterns

```python
# 1. Always pass IDs, not objects
# BAD
@task
def send_email(user):  # Object won't serialize to JSON
    ...

# GOOD
@task
def send_email(user_id: int):
    user = User.objects.get(id=user_id)
    ...


# 2. Idempotent tasks (can be safely retried)
@task
def charge_subscription(subscription_id: int):
    subscription = Subscription.objects.get(id=subscription_id)
    if subscription.is_paid_for_current_period():
        return  # Already processed — idempotent!
    process_payment(subscription)


# 3. Small, focused tasks
# BAD: One giant task
@task
def process_order(order_id):
    validate_payment(...)
    update_inventory(...)
    send_confirmation(...)
    notify_warehouse(...)

# GOOD: Composed tasks
@task
def process_order(order_id: int):
    validate_payment.enqueue(order_id=order_id)

@task
def validate_payment(order_id: int):
    # ... validate ...
    update_inventory.enqueue(order_id=order_id)

# Or with Celery chains:
chain(
    validate_payment.s(order_id),
    update_inventory.s(),
    send_confirmation.s(),
)()
```

---

## 14. Security

### 14.1 Content Security Policy (CSP) Built-In (Django 6.0+)

Django 6.0 provides native CSP support, replacing the third-party `django-csp` package:

```python
# config/settings/base.py
from django.utils.csp import CSP

# Add CSP middleware
MIDDLEWARE = [
    # ...
    "django.middleware.csp.ContentSecurityPolicyMiddleware",
    # ...
]

# Add CSP context processor for nonce support
TEMPLATES = [
    {
        "OPTIONS": {
            "context_processors": [
                # ...
                "django.template.context_processors.csp",
            ],
        },
    },
]

# Enforced CSP policy
SECURE_CSP = {
    "default-src": [CSP.SELF],
    "script-src": [CSP.SELF, CSP.NONCE],          # Allow self + nonced scripts
    "style-src": [CSP.SELF, CSP.UNSAFE_INLINE],    # Allow self + inline styles
    "img-src": [CSP.SELF, "data:", "https:"],       # Allow self + data URIs + HTTPS images
    "font-src": [CSP.SELF, "https://fonts.gstatic.com"],
    "connect-src": [CSP.SELF],
    "frame-ancestors": [CSP.NONE],                  # Prevent framing (replaces X-Frame-Options)
    "base-uri": [CSP.SELF],
    "form-action": [CSP.SELF],
}

# Optional: report-only mode for testing
SECURE_CSP_REPORT_ONLY = {
    "default-src": [CSP.SELF],
    "report-uri": "/csp-report/",
}
```

Using nonces in templates:

```html
<!-- Use {{ csp_nonce }} for inline scripts that need to execute -->
<script nonce="{{ csp_nonce }}">
    // This script will be allowed by CSP
    console.log("Hello from nonced script");
</script>

<!-- External scripts from allowed sources -->
<script src="{% static 'js/app.js' %}" nonce="{{ csp_nonce }}"></script>
```

Per-view CSP overrides:

```python
from django.utils.csp import CSP
from django.views.decorators.csp import csp_override, csp_report_only_override


@csp_override({
    "default-src": [CSP.SELF],
    "script-src": [CSP.SELF, "https://cdn.example.com"],
})
def widget_view(request):
    """This view allows scripts from an additional CDN."""
    ...


@csp_override({})  # Disables CSP for this view
def legacy_view(request):
    """This view has no CSP enforcement (use sparingly!)."""
    ...
```

### 14.2 HTTPS Everywhere

```python
# config/settings/production.py

# Redirect all HTTP to HTTPS
SECURE_SSL_REDIRECT = True

# Tell Django to trust the X-Forwarded-Proto header from your reverse proxy
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")

# Cookies only sent over HTTPS
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

### 14.3 HSTS Configuration

```python
# config/settings/production.py

# HTTP Strict Transport Security
SECURE_HSTS_SECONDS = 31536000          # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True   # Apply to all subdomains
SECURE_HSTS_PRELOAD = True              # Allow preloading in browsers

# Content type sniffing prevention
SECURE_CONTENT_TYPE_NOSNIFF = True
```

> **Warning**: Start with a short `SECURE_HSTS_SECONDS` (e.g., 3600) and increase once you're confident HTTPS works correctly. HSTS is difficult to undo once browsers have cached it.

### 14.4 CSRF Protection

- Always include `{% csrf_token %}` in POST forms.
- For AJAX, include the `X-CSRFToken` header (see Section 9.5).
- Never use `@csrf_exempt` unless absolutely necessary (e.g., webhook endpoints with their own auth).
- Use `CSRF_COOKIE_HTTPONLY = True` (prevents JavaScript from reading the cookie, requiring the token from the DOM).

### 14.5 XSS Prevention with format_html

```python
from django.utils.html import format_html, escape

# BAD: Direct string concatenation (XSS vulnerability)
def get_link(self):
    return f'<a href="{self.url}">{self.name}</a>'  # UNSAFE!

# GOOD: Use format_html
def get_link(self):
    return format_html('<a href="{}">{}</a>', self.url, self.name)

# GOOD: Use escape for user input
def get_display(self):
    return format_html(
        '<span class="badge">{}</span>',
        escape(self.user_provided_text),
    )

# In templates, auto-escaping is ON by default
# {{ user_input }} — automatically escaped
# {{ user_input|safe }} — NEVER use |safe with user input!
# {% autoescape off %} — NEVER disable autoescaping for user content!
```

### 14.6 SQL Injection Prevention

Django's ORM prevents SQL injection by default. Never use raw SQL with user input:

```python
# BAD: SQL injection vulnerability
User.objects.raw(f"SELECT * FROM users WHERE name = '{name}'")

# GOOD: Parameterized queries
User.objects.raw("SELECT * FROM users WHERE name = %s", [name])

# BEST: Use the ORM
User.objects.filter(name=name)

# If you must use raw SQL, always use parameters
from django.db import connection
with connection.cursor() as cursor:
    cursor.execute("SELECT * FROM users WHERE name = %s", [name])
```

### 14.7 Security Headers (SecurityMiddleware)

Django's `SecurityMiddleware` handles most security headers:

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    # Must be near the top
    ...
]

# Security settings (production)
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SECURE_CONTENT_TYPE_NOSNIFF = True
SECURE_REFERRER_POLICY = "same-origin"
X_FRAME_OPTIONS = "DENY"
```

### 14.8 django-allauth for Authentication

```python
# Install: uv add django-allauth

# config/settings/base.py
INSTALLED_APPS = [
    # ...
    "allauth",
    "allauth.account",
    "allauth.socialaccount",
    # Social providers you want to support:
    # "allauth.socialaccount.providers.google",
    # "allauth.socialaccount.providers.github",
]

MIDDLEWARE = [
    # ...
    "allauth.account.middleware.AccountMiddleware",
]

AUTHENTICATION_BACKENDS = [
    "django.contrib.auth.backends.ModelBackend",
    "allauth.account.auth_backends.AuthenticationBackend",
]

# allauth settings
ACCOUNT_AUTHENTICATION_METHOD = "email"
ACCOUNT_EMAIL_REQUIRED = True
ACCOUNT_USERNAME_REQUIRED = False
ACCOUNT_EMAIL_VERIFICATION = "mandatory"
ACCOUNT_SIGNUP_PASSWORD_ENTER_TWICE = False
ACCOUNT_SESSION_REMEMBER = True
ACCOUNT_UNIQUE_EMAIL = True
ACCOUNT_LOGIN_ON_EMAIL_CONFIRMATION = True
```

### 14.9 Two-Factor Authentication

```python
# Install: uv add django-allauth[mfa]

INSTALLED_APPS += ["allauth.mfa"]

MFA_ADAPTER = "allauth.mfa.adapter.DefaultMFAAdapter"
MFA_SUPPORTED_TYPES = ["totp", "recovery_codes", "webauthn"]
```

### 14.10 Password Validation

```python
AUTH_PASSWORD_VALIDATORS = [
    {
        "NAME": "django.contrib.auth.password_validation.UserAttributeSimilarityValidator",
    },
    {
        "NAME": "django.contrib.auth.password_validation.MinimumLengthValidator",
        "OPTIONS": {"min_length": 12},
    },
    {
        "NAME": "django.contrib.auth.password_validation.CommonPasswordValidator",
    },
    {
        "NAME": "django.contrib.auth.password_validation.NumericPasswordValidator",
    },
]
```

---

## 15. Testing

### 15.1 pytest + pytest-django as the Standard

> **DEPRECATED**: Do not use Django's built-in `unittest`-based test runner for new projects. Use `pytest`.

```toml
# pyproject.toml
[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "config.settings.test"
python_files = ["tests.py", "test_*.py"]
addopts = [
    "--strict-markers",
    "--tb=short",
    "--reuse-db",
    "-q",
]
```

```python
# myproject/articles/tests/test_models.py
import pytest
from myproject.articles.models import Article
from .factories import ArticleFactory


@pytest.mark.django_db
class TestArticleModel:
    def test_str_representation(self):
        article = ArticleFactory(title="Test Article")
        assert str(article) == "Test Article"

    def test_default_status_is_draft(self):
        article = ArticleFactory()
        assert article.status == Article.Status.DRAFT

    def test_published_article_has_published_at(self):
        article = ArticleFactory(status=Article.Status.PUBLISHED)
        assert article.published_at is not None

    def test_slug_is_unique(self):
        ArticleFactory(slug="test-slug")
        with pytest.raises(Exception):
            ArticleFactory(slug="test-slug")
```

### 15.2 Factory Boy Instead of Fixtures

```python
# myproject/articles/tests/factories.py
import factory
from factory.django import DjangoModelFactory
from django.utils import timezone

from myproject.articles.models import Article
from myproject.users.tests.factories import UserFactory


class ArticleFactory(DjangoModelFactory):
    class Meta:
        model = Article

    title = factory.Sequence(lambda n: f"Article {n}")
    slug = factory.LazyAttribute(lambda obj: obj.title.lower().replace(" ", "-"))
    body = factory.Faker("paragraphs", nb=3, ext_word_list=None)
    author = factory.SubFactory(UserFactory)
    status = Article.Status.DRAFT
    created_at = factory.LazyFunction(timezone.now)

    class Params:
        published = factory.Trait(
            status=Article.Status.PUBLISHED,
            published_at=factory.LazyFunction(timezone.now),
        )


class UserFactory(DjangoModelFactory):
    class Meta:
        model = "users.User"
        django_get_or_create = ("email",)

    email = factory.Sequence(lambda n: f"user{n}@example.com")
    first_name = factory.Faker("first_name")
    last_name = factory.Faker("last_name")
    password = factory.PostGenerationMethodCall("set_password", "testpass123")

    class Params:
        admin = factory.Trait(
            is_staff=True,
            is_superuser=True,
        )
```

Usage:

```python
# Create a single article
article = ArticleFactory()

# Create a published article
article = ArticleFactory(published=True)

# Create with custom attributes
article = ArticleFactory(title="My Custom Title", author=admin_user)

# Create a batch
articles = ArticleFactory.create_batch(10, published=True)

# Build without saving to DB
article = ArticleFactory.build()
```

### 15.3 Test Structure

```
myproject/articles/
├── tests/
│   ├── __init__.py
│   ├── conftest.py          # Shared fixtures for this app
│   ├── factories.py         # Factory Boy factories
│   ├── test_models.py       # Model tests
│   ├── test_views.py        # View tests
│   ├── test_services.py     # Service layer tests
│   ├── test_selectors.py    # Selector tests
│   ├── test_forms.py        # Form tests
│   ├── test_api.py          # API tests
│   └── test_tasks.py        # Background task tests
```

```python
# myproject/articles/tests/conftest.py
import pytest
from .factories import ArticleFactory, UserFactory


@pytest.fixture
def user():
    return UserFactory()


@pytest.fixture
def admin_user():
    return UserFactory(admin=True)


@pytest.fixture
def published_article(user):
    return ArticleFactory(published=True, author=user)


@pytest.fixture
def api_client():
    from rest_framework.test import APIClient
    return APIClient()


@pytest.fixture
def authenticated_api_client(user, api_client):
    api_client.force_authenticate(user=user)
    return api_client
```

### 15.4 Testing Services and Views

```python
# test_services.py
import pytest
from decimal import Decimal
from myproject.orders.services import OrderService
from myproject.orders.tests.factories import OrderFactory
from myproject.products.tests.factories import ProductFactory
from myproject.users.tests.factories import UserFactory


@pytest.mark.django_db
class TestOrderService:
    def test_create_order_success(self):
        user = UserFactory()
        product = ProductFactory(price=Decimal("10.00"), stock=5)

        order = OrderService.create_order(
            user=user,
            items=[{"product_id": product.id, "quantity": 2}],
            shipping_address="123 Main St",
        )

        assert order.total == Decimal("20.00")
        assert order.items.count() == 1
        product.refresh_from_db()
        assert product.stock == 3

    def test_create_order_insufficient_stock(self):
        user = UserFactory()
        product = ProductFactory(price=Decimal("10.00"), stock=1)

        with pytest.raises(ValueError, match="Insufficient stock"):
            OrderService.create_order(
                user=user,
                items=[{"product_id": product.id, "quantity": 5}],
                shipping_address="123 Main St",
            )

    def test_cancel_order(self):
        order = OrderFactory(status="pending")
        result = OrderService.cancel_order(order=order, reason="Changed mind")
        assert result.status == "cancelled"
```

```python
# test_views.py
import pytest
from django.urls import reverse


@pytest.mark.django_db
class TestArticleViews:
    def test_article_list_requires_login(self, client):
        # With LoginRequiredMiddleware, unauthenticated users get redirected
        response = client.get(reverse("articles:list"))
        assert response.status_code == 302

    def test_article_list_authenticated(self, client, user, published_article):
        client.force_login(user)
        response = client.get(reverse("articles:list"))
        assert response.status_code == 200
        assert published_article.title in response.content.decode()

    def test_article_create(self, client, user):
        client.force_login(user)
        response = client.post(reverse("articles:create"), {
            "title": "New Article",
            "body": "Article body content",
        })
        assert response.status_code == 302  # Redirect on success
```

### 15.5 Async Test Support

```python
import pytest
from django.test import AsyncClient


@pytest.mark.django_db
@pytest.mark.asyncio
async def test_async_view():
    client = AsyncClient()
    response = await client.get("/async-endpoint/")
    assert response.status_code == 200
```

### 15.6 Coverage Targets

Cookiecutter-django starts with 100% test configuration. Aim for:
- **90%+ overall coverage** for established projects
- **100% coverage** for service layer and critical business logic
- **80%+ coverage** for views (test happy path + key error cases)
- **Skip testing** Django internals, migrations, admin configuration

```bash
# Run tests with coverage
uv run pytest --cov=myproject --cov-report=term-missing --cov-report=html

# Fail if coverage drops below threshold
uv run pytest --cov=myproject --cov-fail-under=90
```

---

## 16. Deployment

### 16.1 Docker + Docker Compose for Production

Multi-stage production Dockerfile:

```dockerfile
# Dockerfile
FROM python:3.13-slim AS base

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Install uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

# ---- Builder ----
FROM base AS builder
COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev

# ---- Production ----
FROM base AS production

# Create non-root user
RUN groupadd --gid 1000 appuser && \
    useradd --uid 1000 --gid appuser --shell /bin/bash appuser

# Copy virtual environment from builder
COPY --from=builder /app/.venv /app/.venv
ENV PATH="/app/.venv/bin:$PATH"

# Copy application code
COPY --chown=appuser:appuser . .

# Collect static files
RUN python manage.py collectstatic --noinput

# Switch to non-root user
USER appuser

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health/')"

CMD ["gunicorn", "config.asgi:application", \
     "-k", "uvicorn_worker.UvicornWorker", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "4", \
     "--max-requests", "10000", \
     "--max-requests-jitter", "1000", \
     "--timeout", "120", \
     "--keep-alive", "5", \
     "--access-logfile", "-", \
     "--error-logfile", "-"]
```

Production Docker Compose:

```yaml
# docker-compose.production.yml
services:
  django:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    restart: unless-stopped
    env_file:
      - .env.production
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "python -c \"import urllib.request; urllib.request.urlopen('http://localhost:8000/health/')\""]
      interval: 30s
      timeout: 10s
      retries: 3

  postgres:
    image: postgres:17-alpine
    restart: unless-stopped
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: ${DATABASE_NAME}
      POSTGRES_USER: ${DATABASE_USER}
      POSTGRES_PASSWORD: ${DATABASE_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DATABASE_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  celery_worker:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    command: celery -A config worker -l info --concurrency=4
    restart: unless-stopped
    env_file:
      - .env.production
    depends_on:
      - postgres
      - redis

  celery_beat:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    command: celery -A config beat -l info --scheduler django_celery_beat.schedulers:DatabaseScheduler
    restart: unless-stopped
    env_file:
      - .env.production
    depends_on:
      - postgres
      - redis

  traefik:
    image: traefik:v3.2
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.letsencrypt.acme.email=${ACME_EMAIL}"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik_data:/letsencrypt

volumes:
  postgres_data:
  redis_data:
  traefik_data:
```

### 16.2 Gunicorn + Uvicorn for WSGI/ASGI

```python
# gunicorn.conf.py
import multiprocessing

# Bind
bind = "0.0.0.0:8000"

# Workers
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = "uvicorn_worker.UvicornWorker"  # For ASGI
# worker_class = "gthread"  # For WSGI (if not using async)
worker_connections = 1000
max_requests = 10000
max_requests_jitter = 1000

# Timeouts
timeout = 120
graceful_timeout = 30
keepalive = 5

# Logging
accesslog = "-"
errorlog = "-"
loglevel = "info"

# Process naming
proc_name = "myproject"
```

### 16.3 Static Files: Whitenoise vs S3/CDN

**Whitenoise (Simple, self-hosted static files):**

```python
# config/settings/base.py
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",  # Right after SecurityMiddleware
    # ...
]

STORAGES = {
    "staticfiles": {
        "BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage",
    },
}
```

**S3/CDN (For larger deployments):**

```python
# Install: uv add django-storages boto3

STORAGES = {
    "default": {
        "BACKEND": "storages.backends.s3boto3.S3Boto3Storage",
        "OPTIONS": {
            "bucket_name": env("AWS_STORAGE_BUCKET_NAME"),
            "region_name": env("AWS_S3_REGION_NAME"),
            "custom_domain": env("AWS_CLOUDFRONT_DOMAIN", default=None),
        },
    },
    "staticfiles": {
        "BACKEND": "storages.backends.s3boto3.S3StaticStorage",
        "OPTIONS": {
            "bucket_name": env("AWS_STATIC_BUCKET_NAME"),
        },
    },
}
```

### 16.4 Database Migrations in CI/CD

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_pass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v5

      - name: Set up Python
        run: uv python install 3.13

      - name: Install dependencies
        run: uv sync --frozen

      - name: Run linting
        run: uv run ruff check .

      - name: Run formatting check
        run: uv run ruff format --check .

      - name: Check migrations
        run: uv run python manage.py makemigrations --check --dry-run
        env:
          DATABASE_URL: postgres://test_user:test_pass@localhost:5432/test_db

      - name: Run migrations
        run: uv run python manage.py migrate
        env:
          DATABASE_URL: postgres://test_user:test_pass@localhost:5432/test_db

      - name: Run tests
        run: uv run pytest --cov=myproject --cov-report=xml
        env:
          DATABASE_URL: postgres://test_user:test_pass@localhost:5432/test_db

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Deploy
        run: |
          # Build and push Docker image
          docker build -t myproject:latest .
          # Run migrations
          docker run myproject:latest python manage.py migrate --noinput
          # Deploy (varies by platform)
```

### 16.5 Health Checks

```python
# myproject/core/views.py
from django.contrib.auth.decorators import login_not_required
from django.db import connection
from django.http import HttpResponse, JsonResponse


@login_not_required
def health_check(request):
    """Basic health check for load balancers."""
    return HttpResponse("OK")


@login_not_required
def readiness_check(request):
    """Detailed readiness check including database connectivity."""
    checks = {}

    # Database check
    try:
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")
        checks["database"] = "ok"
    except Exception as e:
        checks["database"] = f"error: {e}"

    # Cache check
    try:
        from django.core.cache import cache
        cache.set("health_check", "ok", timeout=10)
        assert cache.get("health_check") == "ok"
        checks["cache"] = "ok"
    except Exception as e:
        checks["cache"] = f"error: {e}"

    all_ok = all(v == "ok" for v in checks.values())
    status = 200 if all_ok else 503

    return JsonResponse({"status": "ok" if all_ok else "degraded", "checks": checks}, status=status)


# config/urls.py
urlpatterns = [
    path("health/", health_check, name="health-check"),
    path("ready/", readiness_check, name="readiness-check"),
    # ...
]
```

### 16.6 Zero-Downtime Deployments

Key practices:
1. **Separate migration and deployment**: Run `migrate` before deploying new code.
2. **Backwards-compatible migrations**: New code should work with both old and new schema.
3. **Rolling updates**: Deploy new containers gradually while old ones still serve traffic.
4. **Health checks**: Load balancer only sends traffic to healthy containers.
5. **Graceful shutdown**: Gunicorn's `--graceful-timeout` allows in-flight requests to complete.

Migration safety rules:
- Never rename a column in a single migration (add new, migrate data, remove old).
- Never remove a column that old code still reads.
- Add columns as nullable or with a default first.
- Add constraints in a later migration after backfilling data.

---

## 17. Performance

### 17.1 Django Debug Toolbar

Essential for development — shows queries, templates, cache hits, and more:

```python
# config/settings/local.py
INSTALLED_APPS += ["debug_toolbar"]
MIDDLEWARE.insert(0, "debug_toolbar.middleware.DebugToolbarMiddleware")
INTERNAL_IPS = ["127.0.0.1"]

# For Docker:
import socket
hostname, _, ips = socket.gethostbyname_ex(socket.gethostname())
INTERNAL_IPS += [".".join(ip.split(".")[:-1] + ["1"]) for ip in ips]
```

### 17.2 Database Query Optimization

```python
# 1. Use only/defer for partial field loading
Article.objects.only("id", "title", "slug")  # Only fetch these columns
Article.objects.defer("body")                 # Fetch everything except body

# 2. Use values/values_list for lightweight queries
Article.objects.values("id", "title")         # Returns dicts
Article.objects.values_list("id", flat=True)  # Returns flat list of IDs

# 3. Use exists() instead of count() for boolean checks
if Article.objects.filter(status="published").exists():  # Faster than count() > 0
    ...

# 4. Use iterator() for large querysets to reduce memory
for article in Article.objects.all().iterator(chunk_size=1000):
    process(article)

# 5. Use bulk operations
Article.objects.bulk_create([
    Article(title=f"Article {i}") for i in range(1000)
], batch_size=100)

Article.objects.bulk_update(articles, ["status"], batch_size=100)

# 6. Use update() instead of save() for mass updates
Article.objects.filter(status="draft").update(status="archived")

# 7. Analyze queries
print(Article.objects.filter(status="published").explain(analyze=True))
```

### 17.3 Caching Strategies

```python
# config/settings/base.py
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": env("REDIS_URL", default="redis://localhost:6379/1"),
        "OPTIONS": {
            "db": 1,
        },
    },
}

# Per-view caching
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)  # Cache for 15 minutes
def article_list(request):
    ...

# Template fragment caching
# {% load cache %}
# {% cache 500 sidebar request.user.id %}
#     ... expensive sidebar content ...
# {% endcache %}

# Low-level caching
from django.core.cache import cache

def get_popular_articles():
    cache_key = "popular_articles"
    articles = cache.get(cache_key)
    if articles is None:
        articles = list(
            Article.objects
            .filter(status="published")
            .order_by("-view_count")[:10]
            .values("id", "title", "slug")
        )
        cache.set(cache_key, articles, timeout=300)  # 5 minutes
    return articles

# Cache invalidation
def publish_article(article):
    article.status = "published"
    article.save()
    cache.delete("popular_articles")
    cache.delete(f"article_{article.slug}")
```

### 17.4 Connection Pooling

See Section 7.6. Key points:
- Use Django 5.1+ built-in connection pooling for small/medium deployments.
- Use PgBouncer for large deployments with many application servers.
- Set `CONN_MAX_AGE` appropriately (600 seconds is a good default).
- Enable `CONN_HEALTH_CHECKS = True` to detect stale connections.

### 17.5 CDN for Static/Media

Configure a CDN (CloudFront, Cloudflare, etc.) to cache static and media files:

```python
# When using S3 + CloudFront
STORAGES = {
    "staticfiles": {
        "BACKEND": "storages.backends.s3boto3.S3StaticStorage",
        "OPTIONS": {
            "bucket_name": env("AWS_STATIC_BUCKET_NAME"),
            "custom_domain": env("CDN_DOMAIN"),  # e.g., d1234.cloudfront.net
        },
    },
}
STATIC_URL = f"https://{env('CDN_DOMAIN')}/static/"
```

### 17.6 Compression Middleware

```python
# Django's built-in GZip middleware
MIDDLEWARE = [
    "django.middleware.gzip.GZipMiddleware",  # First in middleware stack
    # ...
]

# With Whitenoise, compression is handled automatically via
# CompressedManifestStaticFilesStorage (Brotli + GZip)
```

---

## 18. Twelve-Factor Compliance

### I. Codebase — One Codebase, Many Deploys

```bash
# Single Git repository, multiple environments
git remote -v
# origin  git@github.com:myorg/myproject.git

# Branches:
# main -> production
# develop -> staging
# feature/* -> development
```

### II. Dependencies — Explicitly Declare and Isolate

```toml
# pyproject.toml declares ALL dependencies
[project]
dependencies = [
    "django~=6.0",
    "psycopg[binary]>=3.2",
    # ... every single dependency listed
]

# uv.lock pins exact versions
# Always committed to version control
```

```bash
# Install exactly what's declared
uv sync --frozen
```

### III. Config — Store Config in the Environment

```python
# config/settings/base.py
import environ
env = environ.Env()

SECRET_KEY = env("DJANGO_SECRET_KEY")          # From environment
DATABASE_URL = env("DATABASE_URL")              # From environment
REDIS_URL = env("REDIS_URL")                    # From environment
# NEVER hardcode secrets, URLs, or credentials
```

### IV. Backing Services — Treat as Attached Resources

```python
# Database, cache, email, storage — all configured via URLs
DATABASES = {"default": env.db("DATABASE_URL")}
CACHES = {"default": env.cache("REDIS_URL")}
EMAIL_CONFIG = env.email("EMAIL_URL")

# Swap Redis for Memcached? Change the URL, not the code.
# Swap PostgreSQL for another? Change the URL (and ENGINE).
```

### V. Build, Release, Run — Strictly Separate Build and Run Stages

```dockerfile
# Build stage
FROM python:3.13-slim AS builder
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

# Release stage (includes app code + built dependencies)
FROM python:3.13-slim AS production
COPY --from=builder /app/.venv /app/.venv
COPY . .
RUN python manage.py collectstatic --noinput

# Run stage
CMD ["gunicorn", "config.asgi:application", "-k", "uvicorn_worker.UvicornWorker"]
```

### VI. Processes — Execute App as Stateless Processes

```python
# NO local file storage for state
# Session data: store in database or Redis
SESSION_ENGINE = "django.contrib.sessions.backends.cache"

# File uploads: store in S3, not local filesystem
STORAGES = {
    "default": {"BACKEND": "storages.backends.s3boto3.S3Boto3Storage"},
}

# Cache: use Redis (shared across processes)
CACHES = {"default": {"BACKEND": "django.core.cache.backends.redis.RedisCache"}}
```

### VII. Port Binding — Export Services via Port Binding

```bash
# Gunicorn binds to a port and serves HTTP directly
gunicorn config.asgi:application --bind 0.0.0.0:8000
# A reverse proxy (Traefik/Nginx) sits in front
```

### VIII. Concurrency — Scale Out via the Process Model

```bash
# Web processes
gunicorn config.asgi:application --workers 4

# Worker processes
celery -A config worker --concurrency=4

# Beat process (scheduler)
celery -A config beat

# Scale by adding more containers/processes, not threads
```

### IX. Disposability — Fast Startup and Graceful Shutdown

```python
# gunicorn.conf.py
graceful_timeout = 30  # Finish in-flight requests on shutdown
timeout = 120          # Kill stuck workers
max_requests = 10000   # Recycle workers to prevent memory leaks
```

### X. Dev/Prod Parity — Keep Environments Similar

```yaml
# Same PostgreSQL version in dev and production
# docker-compose.yml (dev)
services:
  postgres:
    image: postgres:17-alpine  # Same as production

# Same Redis version
  redis:
    image: redis:7-alpine      # Same as production
```

### XI. Logs — Treat Logs as Event Streams

```python
# config/settings/production.py
import structlog

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.StackInfoRenderer(),
        structlog.dev.set_exc_info,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),  # JSON output
    ],
    logger_factory=structlog.PrintLoggerFactory(),  # Print to stdout
)

# In Django's LOGGING config
LOGGING = {
    "version": 1,
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",  # stdout
        },
    },
    "root": {
        "handlers": ["console"],
        "level": "INFO",
    },
}
# Container orchestrators (Docker, K8s) collect stdout logs automatically.
```

### XII. Admin Processes — Run Admin Tasks as One-Off Processes

```bash
# Run management commands as one-off processes
docker exec myproject-django python manage.py migrate
docker exec myproject-django python manage.py createsuperuser
docker exec myproject-django python manage.py clearsessions

# Or in Kubernetes:
kubectl exec deployment/django -- python manage.py migrate

# Custom management commands for admin tasks
# myproject/core/management/commands/cleanup_data.py
from django.core.management.base import BaseCommand

class Command(BaseCommand):
    help = "Clean up expired data"

    def handle(self, *args, **options):
        # ... cleanup logic ...
        self.stdout.write(self.style.SUCCESS("Cleanup complete"))
```

---

## 19. Modern Django Ecosystem

### 19.1 Essential Packages

| Package | Purpose | Notes |
|---------|---------|-------|
| `django-environ` | Environment-based configuration | 12-factor compliance |
| `django-allauth` | Authentication (social, email, MFA) | Best-in-class auth |
| `django-filter` | QuerySet filtering for views/APIs | Integrates with DRF |
| `django-extensions` | Management commands, shell_plus | `show_urls`, `graph_models` |
| `django-cors-headers` | CORS handling for APIs | Required for SPA frontends |
| `dj-rest-auth` | Auth endpoints for DRF | Login, register, password reset |
| `whitenoise` | Static file serving | Simple, no CDN needed for small apps |
| `celery` | Distributed task queue | Complex background processing |
| `sentry-sdk` | Error tracking | Production monitoring |
| `structlog` | Structured logging | JSON logs for production |
| `psycopg[binary]` | PostgreSQL adapter | v3 with async support |
| `redis` | Redis client | For caching and Celery broker |

### 19.2 Development Tools

| Package | Purpose |
|---------|---------|
| `django-debug-toolbar` | Query inspection, profiling |
| `django-silk` | Request profiling, SQL analysis |
| `factory-boy` | Test data factories |
| `faker` | Fake data generation |
| `ruff` | Linting and formatting |
| `mypy` + `django-stubs` | Static type checking |
| `pre-commit` | Git hook management |
| `pytest` + `pytest-django` | Testing framework |
| `pytest-cov` | Test coverage |
| `ipdb` | Better Python debugger |

### 19.3 Package Management with uv

```bash
# Add a package
uv add django-allauth

# Add a dev-only package
uv add --dev django-debug-toolbar

# Remove a package
uv remove django-debug-toolbar

# Update a specific package
uv add django --upgrade

# Update all packages
uv sync --upgrade

# Show installed packages
uv pip list

# Show dependency tree
uv tree
```

### 19.4 Pre-commit Hooks

Complete `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-toml
      - id: check-merge-conflict
      - id: debug-statements
      - id: check-added-large-files
        args: ['--maxkb=1000']
      - id: no-commit-to-branch
        args: ['--branch', 'main']

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.14.14
    hooks:
      - id: ruff-check
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.14.0
    hooks:
      - id: mypy
        additional_dependencies:
          - django-stubs>=5.1
          - types-requests

  - repo: https://github.com/adamchainz/django-upgrade
    rev: "1.23.0"
    hooks:
      - id: django-upgrade
        args: [--target-version, "6.0"]
```

Setup:

```bash
uv add --dev pre-commit
uv run pre-commit install
uv run pre-commit run --all-files  # Initial run
```

---

## 20. Admin Best Practices

### 20.1 Not for End Users

The Django admin is a **developer/staff tool**, not a customer-facing interface. Do not:
- Give customers admin access
- Customize admin extensively to build a user-facing dashboard
- Rely on admin for complex workflows

Instead, build custom views for user-facing features and keep admin for internal use.

### 20.2 Custom Admin Actions

```python
# myproject/articles/admin.py
from django.contrib import admin
from django.utils import timezone
from myproject.articles.models import Article


@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):
    list_display = ["title", "author", "status", "created_at", "published_at"]
    list_filter = ["status", "created_at", "author"]
    search_fields = ["title", "body"]
    date_hierarchy = "created_at"
    readonly_fields = ["created_at", "modified_at"]
    prepopulated_fields = {"slug": ("title",)}
    raw_id_fields = ["author"]  # Better than dropdown for ForeignKey with many rows
    list_per_page = 50

    fieldsets = (
        (None, {
            "fields": ("title", "slug", "body"),
        }),
        ("Publishing", {
            "fields": ("status", "published_at", "author"),
        }),
        ("Metadata", {
            "classes": ("collapse",),
            "fields": ("created_at", "modified_at"),
        }),
    )

    actions = ["make_published", "make_draft", "export_as_csv"]

    @admin.action(description="Mark selected articles as published")
    def make_published(self, request, queryset):
        updated = queryset.update(
            status=Article.Status.PUBLISHED,
            published_at=timezone.now(),
        )
        self.message_user(request, f"{updated} articles published.")

    @admin.action(description="Mark selected articles as draft")
    def make_draft(self, request, queryset):
        queryset.update(status=Article.Status.DRAFT, published_at=None)

    @admin.action(description="Export selected as CSV")
    def export_as_csv(self, request, queryset):
        import csv
        from django.http import HttpResponse

        response = HttpResponse(content_type="text/csv")
        response["Content-Disposition"] = "attachment; filename=articles.csv"
        writer = csv.writer(response)
        writer.writerow(["Title", "Author", "Status", "Created"])
        for article in queryset.select_related("author"):
            writer.writerow([
                article.title,
                article.author.email,
                article.status,
                article.created_at.isoformat(),
            ])
        return response

    def get_queryset(self, request):
        """Optimize admin queries."""
        return super().get_queryset(request).select_related("author")
```

### 20.3 Admin Security

```python
# 1. Change the admin URL (obscurity as one layer of defense)
# config/urls.py
urlpatterns = [
    path("backstage/", admin.site.urls),  # Not /admin/
    # ...
]

# 2. HTTPS only (handled by SECURE_SSL_REDIRECT in production)

# 3. IP restrictions via middleware or reverse proxy
# In Nginx:
# location /backstage/ {
#     allow 10.0.0.0/8;
#     deny all;
#     proxy_pass http://django;
# }

# 4. Admin-specific two-factor auth
# Use django-allauth MFA or django-otp

# 5. Audit logging
# Install: uv add django-auditlog
INSTALLED_APPS += ["auditlog"]
MIDDLEWARE += ["auditlog.middleware.AuditlogMiddleware"]
```

### 20.4 list_display, list_filter, search_fields Optimization

```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    # list_display: columns shown in the list view
    list_display = [
        "id",
        "user_email",       # Custom method instead of ForeignKey traversal
        "status",
        "total",
        "created_at",
    ]

    # list_filter: sidebar filters
    list_filter = [
        "status",
        ("created_at", admin.DateFieldListFilter),
        ("total", admin.EmptyFieldListFilter),
    ]

    # search_fields: searchable columns
    search_fields = [
        "id",
        "user__email",       # Can search across relations
        "user__first_name",
    ]

    # Optimization: avoid N+1 queries in list view
    list_select_related = ["user"]

    @admin.display(description="User Email", ordering="user__email")
    def user_email(self, obj):
        return obj.user.email
```

**Performance tips:**
- Always set `list_select_related` to avoid N+1 queries.
- Use `raw_id_fields` for ForeignKey fields with many records (instead of dropdown).
- Use `autocomplete_fields` for a searchable ForeignKey widget.
- Limit `search_fields` to indexed columns.
- Avoid complex annotations in `get_queryset` that run on every page load.

---

## Quick Reference: What Changed from Django 3.x to 6.x

| Feature | Old Way (3.x) | Modern Way (6.x) |
|---------|---------------|-------------------|
| **Package manager** | pip + virtualenv | uv |
| **Linter/Formatter** | flake8 + isort + black | Ruff |
| **Config format** | requirements.txt + setup.py | pyproject.toml + uv.lock |
| **Timezone handling** | pytz | zoneinfo (built-in) |
| **Form rendering** | `form.as_p()` | Template-based, `as_field_group()` |
| **File storage config** | `DEFAULT_FILE_STORAGE` | `STORAGES` dict |
| **Cache backend (Redis)** | `django-redis` | Built-in `django.core.cache.backends.redis.RedisCache` |
| **Default PK** | `AutoField` (implicit) | `BigAutoField` (explicit) |
| **Auth middleware** | `@login_required` everywhere | `LoginRequiredMiddleware` + `@login_not_required` |
| **CSP** | `django-csp` package | Built-in `ContentSecurityPolicyMiddleware` |
| **Background tasks** | Celery for everything | Django Tasks framework + Celery for complex |
| **Template reuse** | `{% include %}` only | `{% partialdef %}` + `{% partial %}` |
| **Query string URLs** | Manual URL building | `{% querystring %}` tag |
| **DB defaults** | `default=` only | `db_default=` for database-level |
| **Computed columns** | Annotations only | `GeneratedField` |
| **Composite PKs** | Not supported (workarounds) | `CompositePrimaryKey` |
| **Connection pooling** | PgBouncer always | Built-in with `pool` option |
| **Test runner** | Django unittest | pytest + pytest-django |
| **Async ORM** | Not available | Full async methods (`aget`, `acreate`, etc.) |
| **Type checking** | Optional | `mypy` + `django-stubs` standard |

---

## Appendix: Deprecated Patterns to Avoid

1. **`pytz`** — Use `zoneinfo` (Python 3.9+ built-in). Django removed pytz dependency in 4.0.
2. **`DEFAULT_FILE_STORAGE` / `STATICFILES_STORAGE`** — Use `STORAGES` dict (Django 4.2+).
3. **`HttpRequest.is_ajax()`** — Removed in Django 4.0. Check headers manually.
4. **`{% ifequal %}` / `{% ifnotequal %}`** — Removed in Django 4.0. Use `{% if a == b %}`.
5. **`setup.py` / `requirements.txt`** — Use `pyproject.toml` + `uv.lock`.
6. **`flake8` / `black` / `isort`** — Use Ruff (single tool, 100x faster).
7. **`pip` + `virtualenv`** — Use `uv`.
8. **`django-csp`** — Use Django 6.0's built-in CSP middleware.
9. **`DEFAULT_HASHING_ALGORITHM` setting** — Removed in Django 4.0.
10. **`DjangoDivFormRenderer`** — Removed in Django 6.0.
11. **`cx_Oracle`** — Removed in Django 6.0. Use `oracledb`.
12. **`ChoicesMeta` alias** — Removed in Django 6.0.
13. **`@login_required` on every view** — Use `LoginRequiredMiddleware` (Django 5.1+) and `@login_not_required` for public views.
14. **SQLite in production** — Always use PostgreSQL.
15. **Manual `select_for_update()` without `transaction.atomic()`** — Always wrap in a transaction.
