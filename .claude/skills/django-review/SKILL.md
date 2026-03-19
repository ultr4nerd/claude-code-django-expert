---
name: django-review
description: Review Django code for best practices, security, performance, and modern patterns. Automatically activated when reviewing Django files.
allowed-tools: Read, Grep, Glob
---

# Django Code Review Checklist

Use this comprehensive checklist when reviewing Django code. Check every applicable section.

## 1. Security Review

### XSS Prevention
- [ ] No usage of `mark_safe()` — should use `format_html()` instead
- [ ] No `|safe` filter on user-provided data in templates
- [ ] Custom template tags use `format_html()` for HTML output

### SQL Injection
- [ ] No string formatting/concatenation in raw SQL queries
- [ ] All `.raw()` and `cursor.execute()` use parameterized queries (`%s` placeholders)
- [ ] No user input in `.extra()` calls without parameterization

### CSRF
- [ ] All POST forms include `{% csrf_token %}`
- [ ] No `@csrf_exempt` without strong justification
- [ ] AJAX requests include CSRF header

### Authentication & Authorization
- [ ] Views check appropriate permissions before acting
- [ ] Object-level permissions verified (not just class-level)
- [ ] No information leakage through error messages
- [ ] Users cannot access other users' data through ID manipulation

### Secrets
- [ ] No hardcoded secrets, API keys, or passwords
- [ ] Secrets loaded from environment variables
- [ ] `.env` files in `.gitignore`

## 2. Performance Review

### N+1 Queries
- [ ] `select_related()` used for ForeignKey/OneToOne accessed in loops/serializers
- [ ] `prefetch_related()` used for reverse FK/M2M accessed in loops/serializers
- [ ] Check templates for attribute access that triggers queries
  ```python
  # BAD: N+1 in template
  {% for order in orders %}
      {{ order.user.email }}  {# triggers a query per order #}
  {% endfor %}

  # GOOD: prefetched in view
  orders = Order.objects.select_related("user").all()
  ```

### Database Indexes
- [ ] Fields used in `filter()`, `order_by()`, `get()` have indexes
- [ ] Composite indexes for common multi-field queries
- [ ] No unnecessary indexes (they slow down writes)

### QuerySet Optimization
- [ ] Use `.only()` or `.defer()` for large models when only a few fields are needed
- [ ] Use `.values()` or `.values_list()` when full model instances aren't needed
- [ ] Use `.exists()` instead of `len(qs) > 0` or `bool(qs)`
- [ ] Use `.count()` instead of `len(qs)`
- [ ] Use `.iterator()` for large querysets that don't need caching
- [ ] Use `bulk_create()` / `bulk_update()` for batch operations

### Caching
- [ ] Expensive computed values are cached where appropriate
- [ ] Cache invalidation is handled correctly
- [ ] Template fragment caching for expensive sections

## 3. Django Conventions

### Models
- [ ] Inherits from `TimeStampedModel` (or has `created`/`modified` timestamps)
- [ ] `__str__` method defined
- [ ] `Meta` class with `ordering`, `indexes`, `constraints`
- [ ] No `null=True` on `CharField`/`TextField`
- [ ] `TextChoices`/`IntegerChoices` for enum fields
- [ ] `db_default` used for database-level defaults
- [ ] `related_name` on all ForeignKey/M2M fields
- [ ] `on_delete` deliberately chosen (not blindly CASCADE)

### Views
- [ ] Views are thin — business logic in `services.py`
- [ ] Query logic in `selectors.py` (not inline in views)
- [ ] Proper HTTP methods (GET for reads, POST for writes)
- [ ] PRG (Post/Redirect/Get) pattern for form submissions
- [ ] `@login_not_required` only on genuinely public views

### Serializers (DRF)
- [ ] Explicit field lists (no `fields = "__all__"`)
- [ ] Separate input/output serializers when appropriate
- [ ] Validation logic in serializers, business logic in services
- [ ] Read-only fields marked correctly

### Templates
- [ ] Template inheritance used (not copy-paste)
- [ ] No complex logic in templates
- [ ] `{% partialdef %}` used for reusable fragments (Django 6.0+)
- [ ] `{% querystring %}` used for URL params (Django 5.1+)

### Tests
- [ ] Tests exist for new/changed code
- [ ] Using `pytest` + `factory_boy` (not JSON fixtures)
- [ ] Tests are focused (one assertion per test concept)
- [ ] External services are mocked
- [ ] Edge cases and error conditions tested

## 4. 12-Factor Compliance
- [ ] Configuration from environment variables (not hardcoded)
- [ ] No file-based state (sessions, cache, uploads in stateless mode)
- [ ] Database connection via URL
- [ ] Logs to stdout/stderr
- [ ] Static files served via CDN/Whitenoise (not Django in production)

## 5. Code Quality

### Type Hints
- [ ] Function signatures have type hints
- [ ] Return types annotated
- [ ] Complex types use `typing` module

### Documentation
- [ ] Module docstrings on services, selectors, complex views
- [ ] Complex business logic has inline comments
- [ ] API endpoints have docstrings (shown in DRF browsable API)

### Import Order (Ruff enforced)
- [ ] `__future__` → stdlib → third-party → first-party → local

### Error Handling
- [ ] Specific exceptions caught (not bare `except`)
- [ ] `get_object_or_404()` for missing objects
- [ ] Meaningful error messages returned to users
- [ ] Unexpected errors logged with context

## 6. Migration Safety
- [ ] New fields have defaults or are nullable
- [ ] No destructive changes without multi-step deploy plan
- [ ] Data migrations have reverse functions
- [ ] Large table operations are non-blocking

## Review Output Format
When reporting findings, categorize as:
- **🔴 Critical**: Security vulnerabilities, data loss risks
- **🟡 Warning**: Performance issues, convention violations
- **🔵 Suggestion**: Improvements, modern patterns, code style
