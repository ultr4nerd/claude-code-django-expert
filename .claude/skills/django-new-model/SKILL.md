---
name: django-new-model
description: Create a new Django model following modern best practices
disable-model-invocation: true
argument-hint: "[ModelName]"
allowed-tools: Read, Write, Edit, Bash, Grep, Glob
---

# Create a New Django Model

Follow these steps to create a production-quality Django model.

## Step 1: Determine Model Location
- If the app has a single `models.py`: add the model there.
- If the app uses a `models/` package: create `models/<model_name>.py` and import in `models/__init__.py`.

## Step 2: Full Model Template
```python
from django.conf import settings
from django.db import models
from django.db.models import Q
from django.db.models.functions import Now
from django.urls import reverse

from core.models import TimeStampedModel  # or django_extensions.db.models


class <Model>Manager(models.Manager):
    """Custom manager for <Model>."""

    def active(self):
        return self.filter(is_active=True)

    def for_user(self, user):
        return self.filter(user=user)


class <Model>(TimeStampedModel):
    """
    <One-line description of what this model represents.>

    Attributes:
        user: The owner of this <model>.
        title: Human-readable title.
        status: Current lifecycle status.
    """

    class Status(models.TextChoices):
        DRAFT = "draft", "Draft"
        ACTIVE = "active", "Active"
        ARCHIVED = "archived", "Archived"

    # === Relationships ===
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name="<model_plural>",
    )

    # === Fields ===
    title = models.CharField(max_length=255)
    slug = models.SlugField(max_length=255)
    description = models.TextField(blank=True, default="")
    status = models.CharField(
        max_length=20,
        choices=Status,
        db_default=Status.DRAFT,
    )
    is_active = models.BooleanField(db_default=True)
    amount = models.DecimalField(
        max_digits=10,
        decimal_places=2,
        db_default=0,
    )

    # === Database-computed fields (Django 5.0+) ===
    # full_name = models.GeneratedField(
    #     expression=Concat("first_name", Value(" "), "last_name"),
    #     output_field=models.CharField(max_length=255),
    #     db_persist=True,
    # )

    # === Timestamps (inherited from TimeStampedModel) ===
    # created, modified

    # === Managers ===
    objects = <Model>Manager()

    class Meta:
        ordering = ["-created"]
        verbose_name = "<model>"
        verbose_name_plural = "<model_plural>"
        indexes = [
            models.Index(fields=["status", "-created"]),
            models.Index(fields=["user", "status"]),
            models.Index(fields=["slug"]),
        ]
        constraints = [
            models.CheckConstraint(
                check=Q(amount__gte=0),
                name="%(app_label)s_%(class)s_amount_non_negative",
            ),
            models.UniqueConstraint(
                fields=["user", "slug"],
                name="%(app_label)s_%(class)s_unique_user_slug",
            ),
        ]

    def __str__(self):
        return self.title

    def get_absolute_url(self):
        return reverse("<app_name>:detail", kwargs={"pk": self.pk})

    # === Properties ===
    @property
    def is_draft(self):
        return self.status == self.Status.DRAFT

    @property
    def is_editable(self):
        return self.status in {self.Status.DRAFT, self.Status.ACTIVE}
```

## Step 3: Field Type Quick Reference

| Data | Field | Notes |
|------|-------|-------|
| Short text | `CharField(max_length=N)` | Always set max_length |
| Long text | `TextField()` | `blank=True, default=""` if optional |
| Integer | `IntegerField()` or `PositiveIntegerField()` | Use Positive for counts |
| Decimal (money) | `DecimalField(max_digits=M, decimal_places=N)` | Never use Float for money |
| Boolean | `BooleanField(db_default=True)` | Use db_default |
| Date | `DateField()` | |
| DateTime | `DateTimeField()` | Use `db_default=Now()` |
| Email | `EmailField()` | Has built-in validation |
| URL | `URLField()` | Has built-in validation |
| UUID | `UUIDField(default=uuid.uuid4)` | Good for public-facing IDs |
| File | `FileField(upload_to="path/")` | Always set upload_to |
| Image | `ImageField(upload_to="path/")` | Requires Pillow |
| JSON | `JSONField(default=dict)` | Use default=dict or default=list |
| IP | `GenericIPAddressField()` | |
| Slug | `SlugField(max_length=255)` | |
| Enum | `CharField(choices=TextChoices)` | Use TextChoices |

## Step 4: Relationship Patterns

### ForeignKey
```python
user = models.ForeignKey(
    settings.AUTH_USER_MODEL,
    on_delete=models.CASCADE,  # or PROTECT, SET_NULL, SET_DEFAULT
    related_name="orders",
)
```

### ManyToMany
```python
tags = models.ManyToManyField("tags.Tag", related_name="articles", blank=True)
```

### ManyToMany with Through Model
```python
class OrderItem(TimeStampedModel):
    order = models.ForeignKey(Order, on_delete=models.CASCADE, related_name="items")
    product = models.ForeignKey(Product, on_delete=models.PROTECT, related_name="order_items")
    quantity = models.PositiveIntegerField(db_default=1)
    unit_price = models.DecimalField(max_digits=10, decimal_places=2)
```

### OneToOne
```python
profile = models.OneToOneField(
    settings.AUTH_USER_MODEL,
    on_delete=models.CASCADE,
    related_name="profile",
)
```

## Step 5: Register in Admin
```python
@admin.register(<Model>)
class <Model>Admin(admin.ModelAdmin):
    list_display = ["__str__", "user", "status", "created"]
    list_filter = ["status", "created"]
    search_fields = ["title", "user__email"]
    readonly_fields = ["created", "modified"]
    prepopulated_fields = {"slug": ("title",)}
    raw_id_fields = ["user"]
    ordering = ["-created"]
```

## Step 6: Create and Review Migration
```bash
python manage.py makemigrations <app_name>
# REVIEW the migration file before proceeding
python manage.py migrate
```

## Step 7: Create Factory
```python
class <Model>Factory(DjangoModelFactory):
    class Meta:
        model = <Model>

    user = factory.SubFactory(UserFactory)
    title = factory.Faker("sentence", nb_words=4)
    slug = factory.LazyAttribute(lambda o: slugify(o.title))
    status = <Model>.Status.DRAFT
    amount = factory.Faker("pydecimal", left_digits=4, right_digits=2, positive=True)
```

## Step 8: Write Core Tests
```python
@pytest.mark.django_db
class Test<Model>:
    def test_str(self):
        obj = <Model>Factory(title="Test")
        assert str(obj) == "Test"

    def test_get_absolute_url(self):
        obj = <Model>Factory()
        assert obj.get_absolute_url() == f"/<app>/{obj.pk}/"

    def test_amount_non_negative_constraint(self):
        with pytest.raises(IntegrityError):
            <Model>Factory(amount=-1)

    def test_unique_user_slug_constraint(self):
        obj = <Model>Factory(slug="test-slug")
        with pytest.raises(IntegrityError):
            <Model>Factory(user=obj.user, slug="test-slug")

    def test_default_status_is_draft(self):
        obj = <Model>Factory()
        assert obj.status == <Model>.Status.DRAFT

    def test_manager_active(self):
        active = <Model>Factory(is_active=True)
        <Model>Factory(is_active=False)
        assert list(<Model>.objects.active()) == [active]
```

## Checklist
- [ ] Model inherits from `TimeStampedModel`
- [ ] All fields have appropriate types and constraints
- [ ] `TextChoices`/`IntegerChoices` used for enum fields
- [ ] No `null=True` on `CharField`/`TextField`
- [ ] `db_default` used where appropriate
- [ ] `Meta` class with `ordering`, `indexes`, `constraints`
- [ ] `__str__` defined
- [ ] `get_absolute_url` defined (if applicable)
- [ ] `related_name` on all FK/M2M fields
- [ ] Custom manager created (if needed)
- [ ] Registered in admin
- [ ] Migration created and reviewed
- [ ] Factory created
- [ ] Core tests written
