---
name: django-security-audit
description: Perform a security audit on a Django project
disable-model-invocation: true
allowed-tools: Read, Bash, Grep, Glob
---

# Django Security Audit

Execute each section of this audit systematically. Report findings by severity.

## 1. Settings Audit

### Run Django's Built-in Check
```bash
python manage.py check --deploy
```
This checks for common security misconfigurations. Fix all warnings.

### Critical Settings
Check `config/settings/production.py` for these settings:

```bash
# Search for security-related settings
grep -rn "SECRET_KEY\|DEBUG\|ALLOWED_HOSTS\|SECURE_\|CSRF_\|SESSION_\|HSTS" config/settings/
```

**Must verify:**
- [ ] `SECRET_KEY` loaded from environment (not hardcoded)
- [ ] `DEBUG = False` in production
- [ ] `ALLOWED_HOSTS` is explicitly set (not `["*"]`)
- [ ] `SECURE_SSL_REDIRECT = True`
- [ ] `SECURE_HSTS_SECONDS >= 31536000` (1 year)
- [ ] `SECURE_HSTS_INCLUDE_SUBDOMAINS = True`
- [ ] `SECURE_HSTS_PRELOAD = True`
- [ ] `SESSION_COOKIE_SECURE = True`
- [ ] `CSRF_COOKIE_SECURE = True`
- [ ] `SECURE_CONTENT_TYPE_NOSNIFF = True`
- [ ] `CSRF_TRUSTED_ORIGINS` explicitly set
- [ ] `X_FRAME_OPTIONS = "DENY"` or `"SAMEORIGIN"`

### CSP Configuration (Django 6.0+)
- [ ] CSP middleware is enabled in production
- [ ] `default-src` is set to `'self'`
- [ ] No `unsafe-inline` or `unsafe-eval` in script-src (use nonces if needed)
- [ ] Report-uri configured for CSP violations

### Password Configuration
```bash
grep -rn "AUTH_PASSWORD_VALIDATORS" config/settings/
```
- [ ] At least 4 password validators configured
- [ ] `MinimumLengthValidator` set to >= 10 characters
- [ ] Consider `Argon2PasswordHasher` as first hasher

### CORS Configuration
```bash
grep -rn "CORS_" config/settings/
```
- [ ] `CORS_ALLOWED_ORIGINS` explicitly listed (not `CORS_ALLOW_ALL_ORIGINS = True`)
- [ ] `CORS_ALLOW_CREDENTIALS` only if needed (and origins are restricted)

## 2. Code Vulnerability Scan

### XSS (Cross-Site Scripting)
```bash
# Find mark_safe usage
grep -rn "mark_safe" --include="*.py" .

# Find |safe filter in templates
grep -rn "|safe" --include="*.html" .

# Find format_html_join with user data
grep -rn "format_html" --include="*.py" .
```
- [ ] Zero `mark_safe()` calls (replace with `format_html()`)
- [ ] Zero `|safe` on user-provided data in templates
- [ ] `format_html()` used correctly (user data as arguments, not in format string)

### SQL Injection
```bash
# Find raw SQL
grep -rn "\.raw(" --include="*.py" .
grep -rn "cursor\.execute" --include="*.py" .
grep -rn "\.extra(" --include="*.py" .
grep -rn "RawSQL" --include="*.py" .
```
- [ ] All raw SQL uses parameterized queries (`%s` placeholders)
- [ ] No string formatting/f-strings in SQL queries
- [ ] No user input in `.extra()` without parameterization

### CSRF
```bash
# Find csrf_exempt usage
grep -rn "csrf_exempt" --include="*.py" .

# Check forms for csrf_token
grep -rn "method=\"post\"\|method='post'" --include="*.html" . | while read line; do
    file=$(echo "$line" | cut -d: -f1)
    grep -L "csrf_token" "$file" 2>/dev/null
done
```
- [ ] No `@csrf_exempt` without documented justification
- [ ] All POST forms include `{% csrf_token %}`

### Mass Assignment
```bash
# Find __all__ in serializers
grep -rn 'fields.*=.*"__all__"\|fields.*=.*__all__' --include="*.py" .

# Find exclude in ModelForm/Serializer
grep -rn "exclude\s*=" --include="*.py" .
```
- [ ] No `fields = "__all__"` in serializers or forms
- [ ] No `exclude` pattern (use explicit `fields` list instead)
- [ ] Sensitive fields are `read_only` in serializers

### File Upload Security
```bash
# Find FileField/ImageField
grep -rn "FileField\|ImageField" --include="*.py" .
```
- [ ] File type validation (check MIME type, not just extension)
- [ ] File size limits enforced
- [ ] Upload path doesn't allow directory traversal
- [ ] User-uploaded filenames are sanitized or UUIDs
- [ ] Files stored outside webroot

## 3. Authentication Audit

### User Model
```bash
grep -rn "AUTH_USER_MODEL" config/settings/
```
- [ ] Custom user model is defined (not default `auth.User`)
- [ ] Email-based authentication (not username) if appropriate
- [ ] Account lockout after N failed attempts (django-axes or similar)

### Session Management
- [ ] Session backend is database or cache (not cookie-based for sensitive data)
- [ ] Session expiry configured (`SESSION_COOKIE_AGE`)
- [ ] Sessions invalidated on password change

### Admin Security
```bash
grep -rn "admin.site.urls\|admin/" config/urls.py
```
- [ ] Admin URL changed from default `/admin/`
- [ ] Admin accessible only to staff users
- [ ] Consider IP restriction for admin in production
- [ ] 2FA enabled for admin users

### API Authentication
```bash
grep -rn "DEFAULT_AUTHENTICATION_CLASSES\|authentication_classes" --include="*.py" .
```
- [ ] Token/JWT authentication for API endpoints
- [ ] Token rotation/expiry configured
- [ ] No hardcoded API tokens in code

## 4. Dependency Audit
```bash
# Check for known vulnerabilities
pip-audit

# Or using safety
safety check

# Check Django version
python -c "import django; print(django.VERSION)"
```
- [ ] Django is on a supported version (check EOL dates)
- [ ] No known CVEs in dependencies
- [ ] Dependencies are pinned (not floating)

## 5. Information Leakage
```bash
# Check for DEBUG mode indicators
grep -rn "DEBUG" --include="*.py" config/settings/production.py

# Check for verbose error pages
grep -rn "handler404\|handler500" --include="*.py" .

# Check for exposed internal URLs
grep -rn "debug_toolbar\|silk\|profiling" --include="*.py" config/
```
- [ ] Custom 404/500 error pages (no Django debug pages in production)
- [ ] Debug toolbar/profiling tools not enabled in production
- [ ] Stack traces not exposed to end users
- [ ] Sensitive headers removed from responses
- [ ] `.env`, `.git/`, `requirements.txt` not accessible via web

## 6. Secrets in Version Control
```bash
# Check git history for secrets
git log --all --diff-filter=A -- "*.env" ".env*" "*secret*" "*password*" "*token*" "*key*"

# Check current files for potential secrets
grep -rn "password\s*=\s*['\"]" --include="*.py" . | grep -v "test\|factory\|fixture"
grep -rn "token\s*=\s*['\"]" --include="*.py" . | grep -v "test\|factory\|fixture"
grep -rn "api_key\s*=\s*['\"]" --include="*.py" . | grep -v "test\|factory\|fixture"
```
- [ ] No secrets in current codebase
- [ ] No secrets in git history (if found, rotate them)
- [ ] `.env` in `.gitignore`

## Severity Levels
- **CRITICAL**: Exploitable vulnerability (SQL injection, exposed secrets, DEBUG=True in prod)
- **HIGH**: Security misconfiguration (missing HSTS, no CSP, weak passwords)
- **MEDIUM**: Best practice violation (mark_safe usage, no rate limiting)
- **LOW**: Improvement opportunity (dependency updates, additional headers)

## Report Template
```markdown
# Security Audit Report — [Project Name]
**Date:** YYYY-MM-DD
**Auditor:** Claude Code (django-security-audit)

## Summary
- Critical: N findings
- High: N findings
- Medium: N findings
- Low: N findings

## Findings

### [CRITICAL] Finding Title
**Location:** `path/to/file.py:42`
**Description:** ...
**Impact:** ...
**Remediation:** ...

### [HIGH] Finding Title
...
```
