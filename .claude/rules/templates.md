---
paths:
  - "**/templates/**/*.html"
---

# Django Template Rules

## Template Partials (Django 6.0+)
Use `{% partialdef %}` to define reusable template fragments within a template:
```html
{% partialdef order-card %}
<div class="card">
    <h3>{{ order.title }}</h3>
    <p>{{ order.status }}</p>
</div>
{% endpartialdef %}

{# Use the partial elsewhere in the same template or include from other templates #}
{% for order in orders %}
    {% partial order-card %}
{% endfor %}
```
Partials are ideal for HTMX fragments — define once, return via a partial-rendering view.

## Querystring Tag (Django 5.1+)
Use `{% querystring %}` instead of manual URL parameter manipulation:
```html
{# Add or update query parameters #}
<a href="{% querystring page=page_obj.next_page_number %}">Next</a>
<a href="{% querystring sort='name' direction='asc' %}">Sort by Name</a>

{# Remove a parameter #}
<a href="{% querystring page=None %}">Clear page</a>
```

## forloop.length (Django 6.0+)
Access the total iteration count directly:
```html
{% for item in items %}
    Item {{ forloop.counter }} of {{ forloop.length }}
{% endfor %}
```

## Template Inheritance
Use a three-level hierarchy:
```
base.html              → site-wide layout (head, nav, footer)
  app/base.html        → app-specific layout (sidebar, breadcrumbs)
    app/model_list.html → page-specific content
```
```html
{# base.html #}
<!DOCTYPE html>
<html>
<head>{% block head %}{% endblock %}</head>
<body>
    {% block content %}{% endblock %}
    {% block extra_js %}{% endblock %}
</body>
</html>

{# orders/order_list.html #}
{% extends "base.html" %}
{% block content %}
    <h1>Orders</h1>
    {% for order in orders %}
        {% partial order-card %}
    {% empty %}
        <p>No orders found.</p>
    {% endfor %}
{% endblock %}
```

## Naming Convention
- Path: `app_name/model_action.html` (e.g., `orders/order_list.html`, `orders/order_detail.html`).
- Partials: prefix with `_` or use `partialdef` (e.g., `orders/_order_card.html`).
- Email templates: `emails/order_confirmation.html`, `emails/order_confirmation.txt`.

## Logic Restrictions
- **No raw SQL** in templates. Ever.
- **No complex queries** — pass pre-computed data from views.
- **No business logic** — use template tags/filters for display formatting only.
- Limit `{% if %}` nesting to 2 levels maximum.
- Move complex conditionals to model properties or template tags.

## Filters
- Use built-in filters: `{{ value|date:"Y-m-d" }}`, `{{ text|truncatewords:20 }}`, `{{ price|floatformat:2 }}`.
- Create custom filters in `app/templatetags/app_tags.py` for reusable display logic.
- Always `{% load app_tags %}` at the top of templates that use custom tags.

## Security in Templates
- Django auto-escapes by default. **Never** disable with `|safe` unless absolutely necessary.
- Use `format_html()` in Python code, not `mark_safe()`.
- For user-generated HTML, use a bleach/sanitize step before storage.

## Static Files
```html
{% load static %}
<link rel="stylesheet" href="{% static 'css/app.css' %}">
<img src="{% static 'img/logo.svg' %}" alt="Logo">
```

## Forms in Templates
Use Django 5.0+ field group rendering:
```html
<form method="post">
    {% csrf_token %}
    {{ form.as_div }}
    <button type="submit">Save</button>
</form>

{# Or render individual fields #}
{% for field in form %}
    {{ field.as_field_group }}
{% endfor %}
```

## HTMX Integration
When using HTMX, combine with `partialdef`:
```html
<div id="order-list" hx-get="{% url 'orders:list' %}" hx-trigger="load">
    {% partial order-list-content %}
</div>
```
Return only the partial from HTMX views, not the full page.

## Accessibility
- Always include `alt` on images.
- Use semantic HTML (`<nav>`, `<main>`, `<article>`, `<section>`).
- Include `aria-label` on interactive elements where needed.
