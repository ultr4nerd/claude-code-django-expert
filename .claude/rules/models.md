---
paths:
  - "**/models.py"
  - "**/models/**/*.py"
---

# Django Model Rules

## Field Defaults
- Use `BigAutoField` as the default PK (`DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"`).
- Use `db_default` for database-level defaults instead of Python-level `default=` where possible:
  ```python
  created_at = models.DateTimeField(db_default=Now())
  is_active = models.BooleanField(db_default=True)
  ```
- Use `GeneratedField` for computed columns stored in the database:
  ```python
  full_name = models.GeneratedField(
      expression=Concat("first_name", Value(" "), "last_name"),
      output_field=models.CharField(max_length=255),
      db_persist=True,
  )
  ```

## Base Model
All models should inherit from a `TimeStampedModel` base:
```python
class TimeStampedModel(models.Model):
    created = models.DateTimeField(db_default=Now(), editable=False)
    modified = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True
```

## String Fields
- **Never** use `null=True` on `CharField` or `TextField`. Use `blank=True, default=""` instead.
- This avoids two possible "no data" states (NULL and empty string).

## Enum Fields
Use `TextChoices` or `IntegerChoices`:
```python
class Status(models.TextChoices):
    DRAFT = "draft", "Draft"
    PUBLISHED = "published", "Published"
    ARCHIVED = "archived", "Archived"

status = models.CharField(max_length=20, choices=Status, db_default=Status.DRAFT)
```

## Meta Class (always define)
```python
class Meta:
    ordering = ["-created"]
    verbose_name = "order"
    verbose_name_plural = "orders"
    indexes = [
        models.Index(fields=["status", "-created"]),
        models.Index(fields=["user", "status"]),
    ]
    constraints = [
        models.CheckConstraint(
            check=models.Q(price__gte=0),
            name="%(app_label)s_%(class)s_price_non_negative",
        ),
        models.UniqueConstraint(
            fields=["user", "slug"],
            name="%(app_label)s_%(class)s_unique_user_slug",
        ),
    ]
```

## Always Define `__str__`
```python
def __str__(self):
    return self.title  # or a meaningful representation
```

## Model Managers
Use custom managers for reusable query logic:
```python
class PublishedManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(status=Article.Status.PUBLISHED)

class Article(TimeStampedModel):
    objects = models.Manager()  # default
    published = PublishedManager()
```

## Relationships
- Always set `related_name` on ForeignKey/M2M fields.
- Use `on_delete=models.CASCADE` deliberately — consider `PROTECT` or `SET_NULL` for important references.
- Use `db_index=True` on ForeignKey fields used in frequent lookups (Django adds this by default for FK).

## Model Methods
- Keep business logic in `services.py`, not in model methods.
- Model methods should be limited to: `__str__`, properties for computed display values, `clean()` for validation.
- Use `@property` for simple computed values that don't hit the DB.

## Composite Primary Keys (Django 5.2+)
For junction/through tables, consider composite PKs:
```python
class Meta:
    pk = models.CompositePrimaryKey("order_id", "product_id")
```

## Migration Safety
- After changing models, always run `python manage.py makemigrations` and review the generated migration.
- Never add a non-nullable field without a default or `db_default`.
- For large tables, avoid `ALTER TABLE` that rewrites the entire table.
