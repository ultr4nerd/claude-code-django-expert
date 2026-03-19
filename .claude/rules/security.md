---
paths:
  - "**/*.py"
---

# Django Security Rules (Global)

## XSS Prevention
- **Never** use `mark_safe()`. Use `format_html()` instead:
  ```python
  # BAD
  from django.utils.safestring import mark_safe
  return mark_safe(f"<strong>{user_input}</strong>")

  # GOOD
  from django.utils.html import format_html
  return format_html("<strong>{}</strong>", user_input)
  ```
- Django templates auto-escape by default. Never disable with `|safe` on user input.

## SQL Injection Prevention
- **Never** use string formatting in raw queries:
  ```python
  # BAD — SQL injection vulnerability
  Model.objects.raw(f"SELECT * FROM app_model WHERE name = '{name}'")
  cursor.execute(f"SELECT * FROM app_model WHERE id = {user_id}")

  # GOOD — parameterized queries
  Model.objects.raw("SELECT * FROM app_model WHERE name = %s", [name])
  cursor.execute("SELECT * FROM app_model WHERE id = %s", [user_id])
  ```
- Prefer the ORM over raw SQL. Use `extra()`, `RawSQL()`, and `raw()` only when absolutely necessary.

## CSRF Protection
- Always include `{% csrf_token %}` in POST forms.
- For AJAX requests, include the CSRF token in headers.
- Never use `@csrf_exempt` unless you have a very specific reason (e.g., webhook endpoints with their own auth).

## Authentication & Authorization
- Always check permissions before performing actions.
- Use Django's permission system or DRF's `permission_classes`.
- Never expose internal IDs that allow enumeration without authorization checks.
- Use `get_object_or_404()` with queryset filtering by user/permissions.

## Secrets Management
- **Never** hardcode secrets, API keys, passwords, or tokens in source code.
- Use environment variables via `django-environ`.
- Never commit `.env` files to version control.
- Rotate `SECRET_KEY` if it's ever exposed.

## File Uploads
- Always validate file type (check MIME type and extension).
- Enforce maximum file size.
- Store uploads outside the webroot.
- Never use user-provided filenames directly — sanitize or generate UUIDs.
- Serve user uploads via a separate domain or with proper Content-Disposition headers.

## Content Security Policy (Django 6.0+)
- Enable CSP middleware in production.
- Start strict and relax only as needed:
  ```python
  CONTENT_SECURITY_POLICY = {
      "default-src": ["'self'"],
      "script-src": ["'self'"],
      "style-src": ["'self'"],
      "img-src": ["'self'", "data:", "https:"],
      "font-src": ["'self'"],
      "connect-src": ["'self'"],
  }
  ```

## Session Security
- Use database or cache-backed sessions, not cookie-based.
- Set `SESSION_COOKIE_SECURE = True` in production.
- Set `SESSION_COOKIE_HTTPONLY = True` (default).
- Consider `SESSION_COOKIE_AGE` for session expiry.

## Password Security
- Use Django's default password hashers (Argon2 preferred, PBKDF2 as fallback).
- Enforce strong password validation via `AUTH_PASSWORD_VALIDATORS`.
- Never store or log plaintext passwords.

## HTTP Security Headers
In production settings, ensure:
```python
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_SSL_REDIRECT = True
SECURE_CONTENT_TYPE_NOSNIFF = True  # default True in Django
```

## Admin Security
- Change the default admin URL from `/admin/` to something non-obvious.
- Use strong admin passwords.
- Consider IP-restricting admin access in production.
- Enable 2FA for admin users (django-allauth or django-otp).

## Logging Sensitive Data
- Never log passwords, tokens, credit card numbers, or PII.
- Sanitize request data in error reporting (Sentry, etc.).
- Use Django's `sensitive_variables` and `sensitive_post_parameters` decorators.

## Dependency Security
- Run `pip-audit` or `safety check` regularly.
- Keep Django and all dependencies updated.
- Subscribe to Django's security mailing list.
