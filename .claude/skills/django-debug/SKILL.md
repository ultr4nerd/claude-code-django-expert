---
name: django-debug
description: Debug Django issues - ORM queries, migrations, template errors, async problems. Use when debugging Django applications.
allowed-tools: Read, Bash, Grep, Glob
---

# Django Debugging Guide

Use this guide when encountering Django errors or unexpected behavior.

## Step 1: Identify the Error Category

### Migration Errors
| Error | Cause | Fix |
|-------|-------|-----|
| `InconsistentMigrationHistory` | Migration applied before its dependency | Reset migration state or fake the dependency |
| `CircularDependencyError` | Two apps' migrations depend on each other | Break the cycle with `SeparateDatabaseAndState` |
| `django.db.utils.ProgrammingError: relation does not exist` | Migration not applied | Run `python manage.py migrate` |
| `Conflicting migrations detected` | Two migrations with same parent | Run `python manage.py makemigrations --merge` |

**Debug steps for migration issues:**
```bash
# Check migration status
python manage.py showmigrations

# Check what SQL a migration generates
python manage.py sqlmigrate app_name 0001

# Fake a migration (mark as applied without running)
python manage.py migrate app_name 0001 --fake

# Reset all migrations for an app
python manage.py migrate app_name zero
```

### ORM / Query Errors
| Error | Cause | Fix |
|-------|-------|-----|
| `FieldError: Cannot resolve keyword` | Wrong field name in filter/query | Check model field names, use `__` for relations |
| `MultipleObjectsReturned` | `.get()` returned more than 1 | Add more filters or use `.filter().first()` |
| `RelatedObjectDoesNotExist` | Accessing reverse FK that doesn't exist | Check if related object was created, use `hasattr()` |
| `TypeError: Field 'id' expected a number but got '...'` | Passing wrong type to filter | Cast the value to the correct type |

**Debug N+1 queries:**
```python
# In settings (local.py)
LOGGING = {
    ...
    "loggers": {
        "django.db.backends": {
            "level": "DEBUG",
            "handlers": ["console"],
        },
    },
}

# Or use django-debug-toolbar
INSTALLED_APPS += ["debug_toolbar"]
MIDDLEWARE += ["debug_toolbar.middleware.DebugToolbarMiddleware"]
INTERNAL_IPS = ["127.0.0.1"]

# Or count queries in tests
from django.test.utils import override_settings

@pytest.mark.django_db
def test_no_n_plus_one(django_assert_num_queries, authenticated_client):
    OrderFactory.create_batch(10)
    with django_assert_num_queries(2):  # 1 auth + 1 list query
        authenticated_client.get("/orders/")
```

### Template Errors
| Error | Cause | Fix |
|-------|-------|-----|
| `TemplateDoesNotExist` | Wrong template path | Check `TEMPLATES` dirs config, check app's `templates/` dir |
| `TemplateSyntaxError` | Invalid template tag/syntax | Check tag spelling, ensure `{% load ... %}` is present |
| `NoReverseMatch` | URL name doesn't exist | Check `app_name:view_name`, check url patterns |
| Variable shows as empty | Context not passed from view | Check `get_context_data()` or `render()` context dict |

**Debug template loading:**
```bash
# Check which template dirs Django searches
python manage.py shell -c "
from django.template.loader import get_template
try:
    t = get_template('orders/order_list.html')
    print(f'Found: {t.origin}')
except Exception as e:
    print(f'Error: {e}')
"
```

### Async Errors
| Error | Cause | Fix |
|-------|-------|-----|
| `SynchronousOnlyOperation` | Sync ORM call in async context | Use async ORM methods (`aget`, `afilter`) or `sync_to_async` |
| `RuntimeError: Event loop is closed` | Improper async cleanup | Use `async with` for connections, ensure proper teardown |
| `django.core.exceptions.SynchronousOnlyOperation` | Accessing lazy attributes in async | Use `select_related()` to prefetch, or wrap in `sync_to_async` |

**Fix SynchronousOnlyOperation:**
```python
# BAD
async def my_view(request):
    orders = Order.objects.filter(user=request.user)  # SYNC call in async view!

# GOOD — use async ORM
async def my_view(request):
    orders = [order async for order in Order.objects.filter(user=request.user)]

# GOOD — use sync_to_async for complex queries
from asgiref.sync import sync_to_async

async def my_view(request):
    orders = await sync_to_async(
        lambda: list(Order.objects.filter(user=request.user).select_related("user"))
    )()
```

### Import / App Errors
| Error | Cause | Fix |
|-------|-------|-----|
| `AppRegistryNotReady` | Accessing models before `django.setup()` | Ensure `django.setup()` in scripts, check import order |
| `ImproperlyConfigured` | Missing settings or misconfiguration | Check the specific setting mentioned in the error |
| Circular import | Two modules importing each other | Use lazy imports, move imports inside functions |

### CORS / CSRF Errors
```python
# CORS setup (django-cors-headers)
INSTALLED_APPS += ["corsheaders"]
MIDDLEWARE = ["corsheaders.middleware.CorsMiddleware", ...] + MIDDLEWARE
CORS_ALLOWED_ORIGINS = env.list("CORS_ALLOWED_ORIGINS", default=[])

# CSRF for API endpoints
# Option 1: Use TokenAuthentication (no CSRF needed)
# Option 2: Include CSRF token in AJAX headers
# Option 3: Use @csrf_exempt on webhook endpoints (with custom auth)
```

## Step 2: General Debugging Workflow

1. **Read the full traceback** — the last frame is usually the most relevant.
2. **Reproduce consistently** — write a failing test if possible.
3. **Check the Django version** — some features/behaviors change between versions.
4. **Use Django shell** to test queries interactively:
   ```bash
   python manage.py shell_plus  # django-extensions, auto-imports models
   ```
5. **Check logs** — look for SQL queries, middleware output, signal handlers.
6. **Bisect** — if the issue is recent, use `git bisect` to find the offending commit.

## Step 3: Diagnostic Commands
```bash
# Check for common issues
python manage.py check --deploy

# Validate models
python manage.py validate

# Show all URL patterns
python manage.py show_urls  # django-extensions

# Show all settings
python manage.py diffsettings

# Database shell
python manage.py dbshell

# Check pending migrations
python manage.py showmigrations --list | grep "\[ \]"
```

## Step 4: Performance Debugging
```bash
# Profile a management command
python -m cProfile -o output.prof manage.py some_command

# Analyze with snakeviz
pip install snakeviz
snakeviz output.prof
```

```python
# Count queries in code
from django.db import connection, reset_queries
from django.conf import settings

settings.DEBUG = True
reset_queries()

# ... run your code ...

print(f"Queries: {len(connection.queries)}")
for q in connection.queries:
    print(f"  {q['time']}s: {q['sql'][:100]}")
```

## Step 5: Report Format
When reporting a debugged issue, include:
1. **Error**: The exact error message and traceback
2. **Cause**: Root cause analysis
3. **Fix**: The specific code change
4. **Prevention**: How to prevent this in the future (test, rule, etc.)
