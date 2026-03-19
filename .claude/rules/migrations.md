---
paths:
  - "**/migrations/**/*.py"
---

# Django Migration Rules

## Never Edit Auto-Generated Migrations
- Do **not** modify auto-generated migration operations (field additions, removals, alterations).
- The only acceptable manual edits:
  - Adding `RunPython` operations for data migrations.
  - Adding `RunSQL` for raw SQL operations.
  - Adjusting `dependencies` to fix ordering issues.
  - Setting `atomic = False` for non-atomic migrations on PostgreSQL.

## Review Before Committing
Always review migration files before committing:
```bash
# Check what migrations would be generated
python manage.py makemigrations --dry-run --verbosity 2

# Review the generated migration file
cat app/migrations/0005_add_status_field.py
```

## Data Migrations
Write data migrations with both forward and reverse functions:
```python
from django.db import migrations

def populate_status(apps, schema_editor):
    Order = apps.get_model("orders", "Order")
    Order.objects.filter(status="").update(status="draft")

def reverse_populate_status(apps, schema_editor):
    pass  # or actual reverse logic

class Migration(migrations.Migration):
    dependencies = [
        ("orders", "0004_add_status_field"),
    ]

    operations = [
        migrations.RunPython(
            populate_status,
            reverse_populate_status,
        ),
    ]
```
- **Always** provide a `reverse` function (even if it's a no-op) for reversibility.
- Use `apps.get_model()` inside `RunPython` — never import models directly.

## Dangerous Operations on Large Tables
Avoid these on large tables in production:
- Adding a column with a non-null default (rewrites the table on older PG). Use `db_default` instead.
- Renaming columns (can break running code during deploy).
- Adding unique constraints (requires full table scan).
- Removing fields (can break running code — use a multi-step deploy).

Safe approach for removing a field:
1. Deploy 1: Remove field from code but keep in DB (migration that removes from model only).
2. Deploy 2: Drop the column in a separate migration.

## Squashing Migrations
Periodically squash migrations to reduce migration count:
```bash
python manage.py squashmigrations app_name 0001 0020
```
- Review the squashed migration and clean up any `RunPython` operations.
- Django 6.x allows re-squashing already-squashed migrations.

## Testing Migrations
```bash
# Test forward migration
python manage.py migrate app_name 0005

# Test backward migration
python manage.py migrate app_name 0004

# Test from scratch
python manage.py migrate app_name zero
python manage.py migrate app_name
```

## Migration Conflicts
When two developers create migrations for the same app:
```bash
# Merge conflicting migrations
python manage.py makemigrations --merge
```
Review the merge migration to ensure it's correct.

## Naming Conventions
Let Django auto-name migrations. For custom data migrations, use descriptive names:
```bash
python manage.py makemigrations --empty app_name -n populate_order_status
```

## Zero-Downtime Deployment Checklist
1. New columns: must be nullable or have `db_default`.
2. Removing columns: remove from code first, drop column in next deploy.
3. Renaming: add new column → copy data → update code → drop old column.
4. Adding indexes: use `AddIndex` with `CREATE INDEX CONCURRENTLY` on PostgreSQL:
   ```python
   class Migration(migrations.Migration):
       atomic = False  # Required for CONCURRENTLY

       operations = [
           AddIndex(
               model_name="order",
               index=models.Index(fields=["status"], name="orders_order_status_idx"),
           ),
       ]
   ```
