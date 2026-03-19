---
paths:
  - "**/views.py"
  - "**/views/**/*.py"
---

# Django View Rules

## Thin Views Principle
Views should only:
1. Parse and validate the request (forms/serializers).
2. Call a service or selector function.
3. Return a response.

**Never** put business logic, complex queries, or side effects directly in views.

## Architecture Pattern
```python
# views.py — thin orchestration
from .services import order_service
from .selectors import order_selectors

def create_order(request):
    form = OrderForm(request.POST)
    if form.is_valid():
        order = order_service.create_order(user=request.user, **form.cleaned_data)
        return redirect("orders:detail", pk=order.pk)
    return render(request, "orders/order_form.html", {"form": form})
```

## Class-Based vs Function-Based
- **CBVs**: Use for standard CRUD (`ListView`, `DetailView`, `CreateView`, `UpdateView`, `DeleteView`).
- **FBVs**: Use for custom logic, simple endpoints, or when CBV mixins become unwieldy.
- **API Views**: Use DRF `APIView` or `ViewSet` for REST endpoints.

## LoginRequiredMiddleware (Django 5.1+)
The project uses `LoginRequiredMiddleware`. All views require login by default.
- To make a view public, use the `@login_not_required` decorator:
  ```python
  from django.contrib.auth.decorators import login_not_required

  @login_not_required
  def public_landing(request):
      return render(request, "pages/landing.html")
  ```
- For CBVs: `LoginNotRequiredMixin` or apply decorator in `urls.py`.

## Async Views
Use `async def` when the view performs I/O-bound operations:
```python
import httpx

async def fetch_external_data(request):
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
    return JsonResponse(response.json())
```
- Use `await` with async ORM methods: `await Model.objects.aget(pk=pk)`, `async for obj in Model.objects.afilter(...)`.
- Do **not** mix sync ORM calls in async views — use `sync_to_async` if unavoidable.

## Error Handling
- Use Django's `Http404` or `get_object_or_404()` for missing objects.
- Handle `ValidationError` from services and re-render forms with errors.
- Never catch bare `Exception` — catch specific exceptions.
- Return proper HTTP status codes (400, 403, 404, 409, 422, 500).

## Permissions
- Use `@permission_required("app.action_model")` for granular permissions.
- For DRF: define `permission_classes` on every view.
- Check object-level permissions explicitly when needed.

## Response Patterns
```python
# Template response
return render(request, "app/template.html", context)

# Redirect after POST (PRG pattern)
return redirect("app:view-name", pk=obj.pk)

# JSON response
return JsonResponse({"status": "ok", "data": serializer.data})

# Streaming (large files)
return StreamingHttpResponse(file_iterator, content_type="text/csv")
```

## View Mixins (CBVs)
- Keep mixin chains short (≤3 mixins).
- MRO order: your mixins first, then Django's generic view last.
  ```python
  class OrderListView(LoginRequiredMixin, PermissionRequiredMixin, ListView):
      ...
  ```

## Pagination
- Always paginate list views. Use `paginate_by` on `ListView`.
- For APIs: configure pagination in DRF settings or per-view.

## File Uploads
- Validate file type and size in the form/serializer, not in the view.
- Use `STORAGES` config for storage backends.
- Never serve uploaded files directly from the web server in production.

## Context Data
- Override `get_context_data()` in CBVs to add extra context.
- Keep context dictionaries flat — avoid deeply nested structures.
- Prefer passing querysets (lazy) over evaluated lists.
