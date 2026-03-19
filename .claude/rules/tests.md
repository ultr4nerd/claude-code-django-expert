---
paths:
  - "**/tests/**/*.py"
  - "**/test_*.py"
  - "**/*_test.py"
---

# Django Testing Rules

## Framework
- **pytest** + **pytest-django** as the test runner (not unittest).
- **factory_boy** + **Faker** for test data (no JSON fixtures, no `setUp` data).
- Config in `pyproject.toml`:
  ```toml
  [tool.pytest.ini_options]
  DJANGO_SETTINGS_MODULE = "config.settings.test"
  python_files = ["test_*.py", "*_test.py"]
  addopts = "-v --reuse-db --strict-markers"
  markers = [
      "slow: marks tests as slow",
  ]
  ```

## File Structure
```
app/
  tests/
    __init__.py
    factories.py       # factory_boy factories
    test_models.py
    test_views.py
    test_services.py
    test_selectors.py
    test_serializers.py
    test_admin.py
    conftest.py         # app-level fixtures
```

## Factory Pattern
```python
import factory
from factory.django import DjangoModelFactory

from myapp.models import Order
from users.tests.factories import UserFactory

class OrderFactory(DjangoModelFactory):
    class Meta:
        model = Order

    user = factory.SubFactory(UserFactory)
    title = factory.Faker("sentence", nb_words=4)
    status = Order.Status.DRAFT
    total = factory.Faker("pydecimal", left_digits=4, right_digits=2, positive=True)
```

## Test Structure: One Assertion Per Test
```python
@pytest.mark.django_db
class TestOrderModel:
    def test_str_returns_title(self):
        order = OrderFactory(title="My Order")
        assert str(order) == "My Order"

    def test_default_status_is_draft(self):
        order = OrderFactory()
        assert order.status == Order.Status.DRAFT

    def test_total_cannot_be_negative(self):
        with pytest.raises(IntegrityError):
            OrderFactory(total=-1)
```

## Database Access
- **Always** mark tests that access the DB with `@pytest.mark.django_db`.
- Use `@pytest.mark.django_db(transaction=True)` for tests that need transaction control.
- Prefer `conftest.py` fixtures over class-level setUp.

## Fixtures via conftest.py
```python
# conftest.py
import pytest
from users.tests.factories import UserFactory

@pytest.fixture
def user():
    return UserFactory()

@pytest.fixture
def authenticated_client(client, user):
    client.force_login(user)
    return client

@pytest.fixture
def api_client():
    from rest_framework.test import APIClient
    return APIClient()

@pytest.fixture
def authenticated_api_client(api_client, user):
    api_client.force_authenticate(user=user)
    return api_client
```

## View Testing
- Use `RequestFactory` for unit-testing views (no middleware, faster).
- Use `Client`/`APIClient` for integration tests (full request/response cycle).
```python
@pytest.mark.django_db
class TestOrderListView:
    def test_returns_200_for_authenticated_user(self, authenticated_client):
        response = authenticated_client.get(reverse("orders:list"))
        assert response.status_code == 200

    def test_unauthenticated_redirects_to_login(self, client):
        response = client.get(reverse("orders:list"))
        assert response.status_code == 302
```

## API Testing
```python
@pytest.mark.django_db
class TestOrderAPI:
    def test_create_order(self, authenticated_api_client):
        payload = {"product_id": 1, "quantity": 2}
        response = authenticated_api_client.post("/api/v1/orders/", payload)
        assert response.status_code == 201
        assert response.data["quantity"] == 2

    def test_create_order_unauthenticated(self, api_client):
        response = api_client.post("/api/v1/orders/", {"product_id": 1})
        assert response.status_code == 403
```

## Parametrize for Multiple Cases
```python
@pytest.mark.django_db
@pytest.mark.parametrize("status,expected", [
    (Order.Status.DRAFT, False),
    (Order.Status.PUBLISHED, True),
    (Order.Status.ARCHIVED, False),
])
def test_is_visible(status, expected):
    order = OrderFactory(status=status)
    assert order.is_visible == expected
```

## Mocking External Services
```python
from unittest.mock import patch

@pytest.mark.django_db
def test_send_notification(user):
    with patch("orders.services.notification_client.send") as mock_send:
        mock_send.return_value = {"status": "sent"}
        result = order_service.complete_order(order)
        mock_send.assert_called_once()
        assert result.status == Order.Status.COMPLETED
```

## Performance
- Use `--reuse-db` to avoid recreating the DB each run.
- Mark slow tests with `@pytest.mark.slow` and exclude in quick runs: `pytest -m "not slow"`.
- Use `pytest-xdist` for parallel tests: `pytest -n auto`.

## What to Test
- Models: `__str__`, constraints, properties, managers, custom methods.
- Services: business logic, edge cases, error conditions.
- Views: status codes, redirects, context data, permissions.
- Serializers: validation, output format.
- Do NOT test Django internals (ORM, auth, admin defaults).
