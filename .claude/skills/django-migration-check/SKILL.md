---
name: django-migration-check
description: Check and validate Django migrations for safety and correctness before deployment
disable-model-invocation: true
---

# Django Migration Safety Check

Run this checklist on every migration before committing. Execute each step.

## Step 1: Identify New Migrations
```bash
# List uncommitted migration files
git status --short | grep migrations

# Or list all pending migrations
python manage.py showmigrations --list | grep "\[ \]"
```

## Step 2: Review Migration SQL
For each new migration, inspect the generated SQL:
```bash
python manage.py sqlmigrate <app_name> <migration_number>
```

Check the SQL for:
- `ALTER TABLE ... ADD COLUMN ... NOT NULL` without a DEFAULT — **dangerous on large tables**
- `CREATE INDEX` without `CONCURRENTLY` on large tables — **locks the table**
- `ALTER TABLE ... DROP COLUMN` — **data loss, ensure multi-step deploy**
- `ALTER TABLE ... ALTER COLUMN TYPE` — **may rewrite the table**
- `ALTER TABLE ... RENAME COLUMN` — **breaks running code during deploy**

## Step 3: Safety Checklist

### Adding a New Field
- [ ] Field is nullable (`null=True`) or has `db_default` or `default`
- [ ] If `db_default` is used, verify it's supported by the database
- [ ] For large tables: prefer `db_default` over Python `default` (avoids table rewrite on older PG)
- [ ] If `BooleanField`, use `db_default=True/False` instead of `null=True`

### Removing a Field
- [ ] **Deploy 1**: Remove the field from model code, keep column in DB (no migration)
- [ ] **Deploy 2**: Create migration to drop the column
- [ ] Ensure no running code references the removed field between deploys
- [ ] Check for any raw SQL, annotations, or `.values()` calls referencing this field

### Renaming a Field
- [ ] **Never** rename in a single step on production
- [ ] **Deploy 1**: Add new field, copy data
- [ ] **Deploy 2**: Update code to use new field
- [ ] **Deploy 3**: Remove old field
- [ ] Or use `db_column` to keep the same column name:
  ```python
  new_name = models.CharField(db_column="old_name", max_length=255)
  ```

### Adding an Index
- [ ] For large tables (>1M rows), use `CONCURRENTLY`:
  ```python
  class Migration(migrations.Migration):
      atomic = False  # Required for CONCURRENTLY
      operations = [
          migrations.AddIndex(
              model_name="order",
              index=models.Index(fields=["status"], name="orders_order_status_idx"),
          ),
      ]
  ```
- [ ] Verify the index name follows convention: `%(app_label)s_%(model)s_%(field)s_idx`
- [ ] Check if a covering/composite index already handles this query

### Adding a Constraint
- [ ] `CheckConstraint`: Verify existing data satisfies the constraint
  ```bash
  python manage.py shell -c "
  from app.models import Model
  violations = Model.objects.filter(amount__lt=0).count()
  print(f'Violations: {violations}')
  "
  ```
- [ ] `UniqueConstraint`: Verify no existing duplicates
  ```bash
  python manage.py shell -c "
  from django.db.models import Count
  from app.models import Model
  dupes = Model.objects.values('user', 'slug').annotate(c=Count('id')).filter(c__gt=1)
  print(f'Duplicates: {dupes.count()}')
  "
  ```

### Data Migration (RunPython)
- [ ] Forward function provided and tested
- [ ] Reverse function provided (at minimum `migrations.RunPython.noop`)
- [ ] Uses `apps.get_model()` — never imports models directly
- [ ] Handles empty tables gracefully
- [ ] Batches updates for large tables:
  ```python
  def populate_status(apps, schema_editor):
      Order = apps.get_model("orders", "Order")
      batch_size = 1000
      while True:
          batch = list(Order.objects.filter(status="").values_list("pk", flat=True)[:batch_size])
          if not batch:
              break
          Order.objects.filter(pk__in=batch).update(status="draft")
  ```

### Removing a Model
- [ ] No foreign keys reference this model (check reverse relations)
- [ ] No code references this model (grep the codebase)
- [ ] Content types and permissions cleaned up after removal

## Step 4: Test Migrations

### Forward Test
```bash
# Apply the migration
python manage.py migrate <app_name> <new_migration>

# Verify the app still works
python manage.py check
pytest <app_name>/tests/ -x
```

### Backward Test
```bash
# Roll back the migration
python manage.py migrate <app_name> <previous_migration>

# Verify rollback worked
python manage.py check
```

### Full Reset Test
```bash
# Test from zero (CI should do this)
python manage.py migrate <app_name> zero
python manage.py migrate <app_name>
```

## Step 5: Pre-Deployment Checklist
- [ ] All migrations apply cleanly on a fresh database
- [ ] All migrations roll back cleanly
- [ ] No migration takes more than 30 seconds on production-size data
- [ ] No migration locks tables for writes on production-size data
- [ ] Data migrations handle edge cases (empty tables, null values)
- [ ] Tests pass after migration
- [ ] Migration plan documented if multi-step deploy is needed

## Step 6: Report Format
```
## Migration Review: <app_name>/<migration_file>

**Operations:**
- AddField: `status` to `Order` (CharField, db_default="draft") ✅ Safe
- AddIndex: `orders_order_status_idx` on `Order.status` ⚠️ Use CONCURRENTLY for large tables

**Risk Level:** Low / Medium / High
**Requires Multi-Step Deploy:** Yes / No
**Estimated Lock Time:** None / <1s / >1s (investigate)
**Backward Compatible:** Yes / No

**Recommendations:**
1. ...
```
