---
name: django-performance
description: Analyze and optimize Django application performance
disable-model-invocation: true
allowed-tools: Read, Bash, Grep, Glob
---

# Django Performance Analysis & Optimization

Execute each section systematically to identify and fix performance issues.

## 1. Identify N+1 Queries

### Find Missing select_related / prefetch_related
```bash
# Find views/serializers that access related objects
grep -rn "\.user\.\|\.author\.\|\.category\.\|\.profile\." --include="*.py" --include="*.html" .

# Find querysets without select_related
grep -rn "objects\.\(filter\|all\|get\)" --include="*.py" . | grep -v "select_related\|prefetch_related"
```

### Measure Query Count
```python
# In tests — use django_assert_num_queries
@pytest.mark.django_db
def test_order_list_queries(django_assert_num_queries, authenticated_client):
    OrderFactory.create_batch(20)
    with django_assert_num_queries(2):  # 1 auth check + 1 list query
        response = authenticated_client.get("/orders/")
    assert response.status_code == 200
```

### Fix Patterns
```python
# BAD: N+1 — each order.user triggers a query
orders = Order.objects.all()
for order in orders:
    print(order.user.email)  # N extra queries!

# GOOD: select_related for ForeignKey/OneToOne (JOIN)
orders = Order.objects.select_related("user").all()

# GOOD: prefetch_related for reverse FK/M2M (separate query)
users = User.objects.prefetch_related("orders").all()

# GOOD: Prefetch with custom queryset
from django.db.models import Prefetch
orders = Order.objects.prefetch_related(
    Prefetch(
        "items",
        queryset=OrderItem.objects.select_related("product"),
    )
)
```

## 2. Database Index Analysis

### Find Missing Indexes
```bash
# Check fields used in filter/order_by/get
grep -rn "\.filter(\|\.exclude(\|\.order_by(\|\.get(" --include="*.py" . | head -50

# Check model Meta for existing indexes
grep -rn "indexes\s*=" --include="*.py" */models*.py
```

### Common Index Candidates
```python
class Meta:
    indexes = [
        # Fields used in WHERE clauses
        models.Index(fields=["status"]),
        # Composite for common query patterns
        models.Index(fields=["user", "status", "-created"]),
        # Partial index (PostgreSQL) — only index active orders
        models.Index(
            fields=["status"],
            name="%(app_label)s_%(class)s_active_idx",
            condition=models.Q(is_active=True),
        ),
        # Index for text search
        GinIndex(
            fields=["search_vector"],
            name="%(app_label)s_%(class)s_search_idx",
        ),
    ]
```

### Verify Index Usage (PostgreSQL)
```sql
-- Check if a query uses indexes
EXPLAIN ANALYZE SELECT * FROM orders_order WHERE status = 'active';

-- Find unused indexes
SELECT schemaname, relname, indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY schemaname, relname;
```

## 3. QuerySet Optimization

### Use Efficient QuerySet Methods
```python
# BAD: loads all objects into memory
if len(Order.objects.filter(status="active")) > 0:
    ...
count = len(Order.objects.all())

# GOOD: database-level operations
if Order.objects.filter(status="active").exists():
    ...
count = Order.objects.count()

# BAD: loads full objects when you only need a few fields
emails = [u.email for u in User.objects.all()]

# GOOD: only fetch needed fields
emails = list(User.objects.values_list("email", flat=True))

# BAD: loads everything into memory
for order in Order.objects.all():
    process(order)

# GOOD: iterate in chunks for large tables
for order in Order.objects.all().iterator(chunk_size=1000):
    process(order)
```

### Bulk Operations
```python
# BAD: N individual INSERT statements
for item in items:
    OrderItem.objects.create(**item)

# GOOD: single bulk INSERT
OrderItem.objects.bulk_create([OrderItem(**item) for item in items], batch_size=1000)

# BAD: N individual UPDATE statements
for order in orders:
    order.status = "archived"
    order.save()

# GOOD: single UPDATE statement
Order.objects.filter(id__in=order_ids).update(status="archived")

# GOOD: bulk_update for varied changes
for order in orders:
    order.status = compute_status(order)
Order.objects.bulk_update(orders, ["status"], batch_size=1000)
```

### Database Annotations
```python
# BAD: compute in Python
orders = Order.objects.all()
for order in orders:
    order.item_count = order.items.count()  # N+1!

# GOOD: annotate at database level
from django.db.models import Count, Sum
orders = Order.objects.annotate(
    item_count=Count("items"),
    total_amount=Sum("items__price"),
)
```

## 4. Caching Strategy

### View-Level Caching
```python
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)  # 15 minutes
def product_list(request):
    ...
```

### Template Fragment Caching
```html
{% load cache %}
{% cache 900 sidebar request.user.id %}
    {# Expensive sidebar content #}
{% endcache %}
```

### Low-Level Caching
```python
from django.core.cache import cache

def get_popular_products():
    cache_key = "popular_products_v1"
    products = cache.get(cache_key)
    if products is None:
        products = list(
            Product.objects.annotate(order_count=Count("orderitem"))
            .order_by("-order_count")[:10]
            .values("id", "name", "order_count")
        )
        cache.set(cache_key, products, timeout=60 * 30)  # 30 min
    return products
```

### Cache Invalidation
```python
# Signal-based invalidation
from django.db.models.signals import post_save, post_delete

def invalidate_product_cache(sender, **kwargs):
    cache.delete("popular_products_v1")

post_save.connect(invalidate_product_cache, sender=OrderItem)
post_delete.connect(invalidate_product_cache, sender=OrderItem)
```

## 5. Database Connection Pooling (Django 5.1+)
```python
# config/settings/base.py
DATABASES = {
    "default": {
        ...
        "OPTIONS": {
            "pool": True,  # Enable built-in connection pooling
        },
    }
}
```
For high-traffic: consider PgBouncer for external connection pooling.

## 6. Async Opportunities

### Identify I/O-Bound Views
```bash
# Find views that call external APIs
grep -rn "requests\.\|httpx\.\|urllib" --include="*.py" */views*.py

# Find views with multiple DB queries
grep -rn "objects\." --include="*.py" */views*.py | cut -d: -f1 | sort | uniq -c | sort -rn
```

### Convert to Async
```python
# Before: sync view with external API call
def dashboard(request):
    weather = requests.get("https://api.weather.com/current").json()
    orders = Order.objects.filter(user=request.user)
    return render(request, "dashboard.html", {"weather": weather, "orders": orders})

# After: async view with concurrent I/O
import httpx

async def dashboard(request):
    async with httpx.AsyncClient() as client:
        weather_task = client.get("https://api.weather.com/current")
        orders = [o async for o in Order.objects.filter(user=request.user)]
        weather = (await weather_task).json()
    return render(request, "dashboard.html", {"weather": weather, "orders": orders})
```

## 7. Static Files & Media

### Whitenoise Configuration
```python
STORAGES = {
    "staticfiles": {
        "BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage",
    },
}
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",  # Right after SecurityMiddleware
    ...
]
```

### CDN for Media Files
```python
# For production — use S3/GCS with CloudFront/CDN
STORAGES = {
    "default": {
        "BACKEND": "storages.backends.s3boto3.S3Boto3Storage",
    },
}
AWS_S3_CUSTOM_DOMAIN = "cdn.example.com"
```

## 8. Report Format
```markdown
# Performance Analysis Report

## Query Analysis
- **Total queries on page load:** X
- **N+1 issues found:** Y
- **Missing indexes:** Z

## Findings

### [HIGH] N+1 Query in Order List View
**Location:** `orders/views.py:42`
**Queries:** 102 queries for 100 orders
**Fix:** Add `select_related("user")` to queryset
**Impact:** Reduces queries from 102 to 2

### [MEDIUM] Missing Index on Order.status
**Location:** `orders/models.py`
**Evidence:** Sequential scan on 500K rows
**Fix:** Add `models.Index(fields=["status", "-created"])`
**Impact:** Query time from 200ms to 5ms

## Recommendations
1. ...
2. ...
```
