---
name: django-new-api
description: Create a new REST API endpoint following modern DRF best practices
disable-model-invocation: true
argument-hint: "[ModelName or endpoint-name]"
allowed-tools: Read, Write, Edit, Bash, Grep, Glob
---

# Create a New REST API Endpoint

Follow these steps to create a production-quality DRF API endpoint.

## Step 1: Create Serializers

### Input Serializer (for create/update)
```python
# <app_name>/serializers.py
from rest_framework import serializers

from .models import <Model>


class <Model>CreateSerializer(serializers.Serializer):
    """Validates input for creating a <Model>."""

    title = serializers.CharField(max_length=255)
    description = serializers.CharField(required=False, default="")
    status = serializers.ChoiceField(
        choices=<Model>.Status.choices,
        default=<Model>.Status.DRAFT,
    )

    def validate_title(self, value):
        """Custom field-level validation."""
        if len(value.strip()) < 3:
            raise serializers.ValidationError("Title must be at least 3 characters.")
        return value.strip()

    def validate(self, attrs):
        """Cross-field validation."""
        # Add cross-field validation here if needed
        return attrs


class <Model>UpdateSerializer(serializers.Serializer):
    """Validates input for updating a <Model>."""

    title = serializers.CharField(max_length=255, required=False)
    description = serializers.CharField(required=False)
    status = serializers.ChoiceField(choices=<Model>.Status.choices, required=False)
```

### Output Serializer (for responses)
```python
class <Model>ListSerializer(serializers.ModelSerializer):
    """Serializes <Model> for list responses (minimal fields)."""

    class Meta:
        model = <Model>
        fields = [
            "id",
            "title",
            "status",
            "created",
        ]


class <Model>DetailSerializer(serializers.ModelSerializer):
    """Serializes <Model> for detail responses (full fields)."""

    user = serializers.SerializerMethodField()

    class Meta:
        model = <Model>
        fields = [
            "id",
            "title",
            "slug",
            "description",
            "status",
            "amount",
            "user",
            "created",
            "modified",
        ]

    def get_user(self, obj):
        return {
            "id": obj.user.id,
            "email": obj.user.email,
            "name": obj.user.get_full_name(),
        }
```

## Step 2: Create API Views

### Option A: ViewSet (for full CRUD)
```python
# <app_name>/views.py (or <app_name>/api_views.py)
from rest_framework import status, viewsets
from rest_framework.decorators import action
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response

from .models import <Model>
from .selectors import <model>_selectors
from .serializers import (
    <Model>CreateSerializer,
    <Model>DetailSerializer,
    <Model>ListSerializer,
    <Model>UpdateSerializer,
)
from .services import <model>_service


class <Model>ViewSet(viewsets.ViewSet):
    """
    API endpoint for <Model> CRUD operations.

    list:   GET    /api/v1/<models>/
    create: POST   /api/v1/<models>/
    read:   GET    /api/v1/<models>/{id}/
    update: PUT    /api/v1/<models>/{id}/
    delete: DELETE /api/v1/<models>/{id}/
    """

    permission_classes = [IsAuthenticated]

    def list(self, request):
        queryset = <model>_selectors.get_list(user=request.user)
        serializer = <Model>ListSerializer(queryset, many=True)
        return Response(serializer.data)

    def create(self, request):
        serializer = <Model>CreateSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        instance = <model>_service.create(
            user=request.user,
            **serializer.validated_data,
        )
        output = <Model>DetailSerializer(instance)
        return Response(output.data, status=status.HTTP_201_CREATED)

    def retrieve(self, request, pk=None):
        instance = <model>_selectors.get_by_id(pk=pk, user=request.user)
        serializer = <Model>DetailSerializer(instance)
        return Response(serializer.data)

    def update(self, request, pk=None):
        instance = <model>_selectors.get_by_id(pk=pk, user=request.user)
        serializer = <Model>UpdateSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        instance = <model>_service.update(
            instance=instance,
            **serializer.validated_data,
        )
        output = <Model>DetailSerializer(instance)
        return Response(output.data)

    def destroy(self, request, pk=None):
        instance = <model>_selectors.get_by_id(pk=pk, user=request.user)
        <model>_service.delete(instance=instance)
        return Response(status=status.HTTP_204_NO_CONTENT)

    @action(detail=True, methods=["post"], url_path="archive")
    def archive(self, request, pk=None):
        """Custom action: POST /api/v1/<models>/{id}/archive/"""
        instance = <model>_selectors.get_by_id(pk=pk, user=request.user)
        instance = <model>_service.archive(instance=instance)
        output = <Model>DetailSerializer(instance)
        return Response(output.data)
```

### Option B: APIView (for simple endpoints)
```python
from rest_framework.views import APIView

class <Model>ListView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request):
        queryset = <model>_selectors.get_list(user=request.user)
        serializer = <Model>ListSerializer(queryset, many=True)
        return Response(serializer.data)

    def post(self, request):
        serializer = <Model>CreateSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        instance = <model>_service.create(
            user=request.user, **serializer.validated_data
        )
        output = <Model>DetailSerializer(instance)
        return Response(output.data, status=status.HTTP_201_CREATED)
```

## Step 3: Create URL Configuration

### For ViewSet
```python
# <app_name>/api_urls.py
from rest_framework.routers import DefaultRouter

from .api_views import <Model>ViewSet

router = DefaultRouter()
router.register("<models>", <Model>ViewSet, basename="<model>")

urlpatterns = router.urls
```

### For APIView
```python
# <app_name>/api_urls.py
from django.urls import path

from .api_views import <Model>DetailView, <Model>ListView

app_name = "<app_name>-api"

urlpatterns = [
    path("<models>/", <Model>ListView.as_view(), name="list"),
    path("<models>/<int:pk>/", <Model>DetailView.as_view(), name="detail"),
]
```

### Register in Root URLs
```python
# config/urls.py
urlpatterns = [
    ...
    path("api/v1/", include("<app_name>.api_urls")),
]
```

## Step 4: Add Pagination
```python
# In DRF settings (config/settings/base.py)
REST_FRAMEWORK = {
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.LimitOffsetPagination",
    "PAGE_SIZE": 25,
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.SessionAuthentication",
        "rest_framework.authentication.TokenAuthentication",
    ],
}
```

## Step 5: Add Filtering (optional)
```python
# Using django-filter
from django_filters.rest_framework import DjangoFilterBackend
from rest_framework.filters import OrderingFilter, SearchFilter

class <Model>ViewSet(viewsets.ViewSet):
    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_fields = ["status", "is_active"]
    search_fields = ["title", "description"]
    ordering_fields = ["created", "title"]
    ordering = ["-created"]
```

## Step 6: Service Layer
```python
# <app_name>/services.py
from django.db import transaction
from django.shortcuts import get_object_or_404

from .models import <Model>


@transaction.atomic
def create(*, user, title, description="", **kwargs):
    return <Model>.objects.create(user=user, title=title, description=description, **kwargs)


@transaction.atomic
def update(*, instance, **kwargs):
    for field, value in kwargs.items():
        setattr(instance, field, value)
    instance.full_clean()
    instance.save(update_fields=list(kwargs.keys()) + ["modified"])
    return instance


@transaction.atomic
def delete(*, instance):
    instance.delete()


@transaction.atomic
def archive(*, instance):
    instance.status = <Model>.Status.ARCHIVED
    instance.save(update_fields=["status", "modified"])
    return instance
```

## Step 7: Selector Layer
```python
# <app_name>/selectors.py
from django.shortcuts import get_object_or_404

from .models import <Model>


def get_list(*, user, filters=None):
    qs = <Model>.objects.filter(user=user).select_related("user")
    if filters:
        qs = qs.filter(**filters)
    return qs.order_by("-created")


def get_by_id(*, pk, user):
    return get_object_or_404(
        <Model>.objects.select_related("user"),
        pk=pk,
        user=user,
    )
```

## Step 8: Write API Tests
```python
import pytest
from django.urls import reverse
from rest_framework import status

from .factories import <Model>Factory


@pytest.mark.django_db
class Test<Model>API:
    endpoint = "/api/v1/<models>/"

    def test_list_authenticated(self, authenticated_api_client, user):
        <Model>Factory.create_batch(3, user=user)
        response = authenticated_api_client.get(self.endpoint)
        assert response.status_code == status.HTTP_200_OK
        assert len(response.data["results"]) == 3

    def test_list_unauthenticated(self, api_client):
        response = api_client.get(self.endpoint)
        assert response.status_code == status.HTTP_403_FORBIDDEN

    def test_create(self, authenticated_api_client):
        payload = {"title": "New Item", "description": "A description"}
        response = authenticated_api_client.post(self.endpoint, payload)
        assert response.status_code == status.HTTP_201_CREATED
        assert response.data["title"] == "New Item"

    def test_create_invalid(self, authenticated_api_client):
        response = authenticated_api_client.post(self.endpoint, {})
        assert response.status_code == status.HTTP_400_BAD_REQUEST

    def test_retrieve(self, authenticated_api_client, user):
        obj = <Model>Factory(user=user)
        response = authenticated_api_client.get(f"{self.endpoint}{obj.pk}/")
        assert response.status_code == status.HTTP_200_OK
        assert response.data["id"] == obj.pk

    def test_update(self, authenticated_api_client, user):
        obj = <Model>Factory(user=user)
        response = authenticated_api_client.put(
            f"{self.endpoint}{obj.pk}/",
            {"title": "Updated"},
        )
        assert response.status_code == status.HTTP_200_OK
        assert response.data["title"] == "Updated"

    def test_delete(self, authenticated_api_client, user):
        obj = <Model>Factory(user=user)
        response = authenticated_api_client.delete(f"{self.endpoint}{obj.pk}/")
        assert response.status_code == status.HTTP_204_NO_CONTENT

    def test_cannot_access_other_users_data(self, authenticated_api_client):
        other_obj = <Model>Factory()  # different user
        response = authenticated_api_client.get(f"{self.endpoint}{other_obj.pk}/")
        assert response.status_code == status.HTTP_404_NOT_FOUND
```

## Checklist
- [ ] Input serializer(s) with validation
- [ ] Output serializer(s) with explicit fields
- [ ] View (ViewSet or APIView) with permission_classes
- [ ] URL configuration registered in root urls
- [ ] Service layer for write operations
- [ ] Selector layer for read operations
- [ ] Pagination configured
- [ ] Filtering configured (if needed)
- [ ] Tests for all CRUD operations
- [ ] Tests for authentication/authorization
- [ ] Tests for validation errors
- [ ] Tests for cross-user data isolation
