---
name: django-new-app
description: Create a new Django app following modern best practices and cookiecutter-django conventions
disable-model-invocation: true
---

# Create a New Django App

Follow these steps exactly to scaffold a new Django app.

## Step 1: Determine App Name
- Use short, lowercase names (no underscores if possible): `orders`, `payments`, `notifications`.
- The app name should be a **plural noun** describing the domain.

## Step 2: Create the App
```bash
# From the project root
mkdir -p <app_name>
python manage.py startapp <app_name> ./<app_name>
```

## Step 3: Create Full Directory Structure
```
<app_name>/
├── __init__.py
├── admin.py
├── apps.py
├── models.py
├── views.py
├── urls.py
├── serializers.py      # if API app
├── services.py         # business logic
├── selectors.py        # query logic
├── forms.py            # if template-based
├── signals.py          # if needed
├── tasks.py            # background tasks
├── managers.py         # custom model managers (or keep in models.py)
├── constants.py        # app-level constants
├── migrations/
│   └── __init__.py
├── templates/
│   └── <app_name>/
│       └── .gitkeep
├── static/
│   └── <app_name>/
│       └── .gitkeep
└── tests/
    ├── __init__.py
    ├── conftest.py
    ├── factories.py
    ├── test_models.py
    ├── test_views.py
    ├── test_services.py
    └── test_serializers.py  # if API app
```

## Step 4: Configure apps.py
```python
from django.apps import AppConfig


class <AppName>Config(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "<app_name>"
    verbose_name = "<Human Readable Name>"

    def ready(self):
        # Import signals if used
        try:
            import <app_name>.signals  # noqa: F401
        except ImportError:
            pass
```

## Step 5: Register in Settings
Add to `config/settings/base.py`:
```python
LOCAL_APPS = [
    ...
    "<app_name>",
]
```

## Step 6: Create urls.py
```python
from django.urls import path

from . import views

app_name = "<app_name>"

urlpatterns = [
    # path("", views.<Model>ListView.as_view(), name="list"),
    # path("<int:pk>/", views.<Model>DetailView.as_view(), name="detail"),
    # path("create/", views.<Model>CreateView.as_view(), name="create"),
    # path("<int:pk>/update/", views.<Model>UpdateView.as_view(), name="update"),
    # path("<int:pk>/delete/", views.<Model>DeleteView.as_view(), name="delete"),
]
```

## Step 7: Include in Root URLs
In `config/urls.py`:
```python
urlpatterns = [
    ...
    path("<app_name>/", include("<app_name>.urls")),
    # For API:
    # path("api/v1/<app_name>/", include("<app_name>.api_urls")),
]
```

## Step 8: Create services.py Template
```python
"""
Business logic for <app_name>.

All write operations and complex business logic should live here.
Views call these functions — they do not contain logic directly.
"""
from django.db import transaction


# Example service function:
# @transaction.atomic
# def create_<model>(*, user, **kwargs):
#     """Create a new <model> instance."""
#     <model> = <Model>.objects.create(user=user, **kwargs)
#     # Trigger side effects (notifications, etc.)
#     return <model>
```

## Step 9: Create selectors.py Template
```python
"""
Query logic for <app_name>.

All read operations and complex queries should live here.
Views and serializers call these functions for data retrieval.
"""

# Example selector function:
# def get_<model>_list(*, user, filters=None):
#     """Return filtered queryset of <model> for a user."""
#     qs = <Model>.objects.filter(user=user)
#     if filters:
#         qs = qs.filter(**filters)
#     return qs.select_related("user").order_by("-created")
```

## Step 10: Create admin.py Template
```python
from django.contrib import admin

# from .models import <Model>


# @admin.register(<Model>)
# class <Model>Admin(admin.ModelAdmin):
#     list_display = ["__str__", "status", "created"]
#     list_filter = ["status", "created"]
#     search_fields = ["title", "user__email"]
#     readonly_fields = ["created", "modified"]
#     ordering = ["-created"]
```

## Step 11: Create tests/conftest.py
```python
import pytest

# App-specific fixtures go here
# from .factories import <Model>Factory

# @pytest.fixture
# def <model>():
#     return <Model>Factory()
```

## Step 12: Create tests/factories.py
```python
import factory
from factory.django import DjangoModelFactory

# from <app_name>.models import <Model>
# from users.tests.factories import UserFactory


# class <Model>Factory(DjangoModelFactory):
#     class Meta:
#         model = <Model>
#
#     user = factory.SubFactory(UserFactory)
#     title = factory.Faker("sentence", nb_words=4)
```

## Step 13: Run Verification
```bash
# Check the app is recognized
python manage.py check

# Create initial migrations (if models are defined)
python manage.py makemigrations <app_name>

# Run the migration
python manage.py migrate

# Run tests
pytest <app_name>/tests/ -v
```

## Checklist
- [ ] App directory created with full structure
- [ ] `apps.py` configured with `verbose_name`
- [ ] App added to `LOCAL_APPS` in settings
- [ ] `urls.py` created with `app_name`
- [ ] Root URLs include the app
- [ ] `services.py` and `selectors.py` created
- [ ] `admin.py` scaffolded
- [ ] Test directory with `conftest.py` and `factories.py`
- [ ] `python manage.py check` passes
- [ ] Initial migration created (if models exist)
