---
paths:
  - "**/serializers.py"
  - "**/serializers/**/*.py"
---

# DRF Serializer Rules

## Explicit Field Declaration
**Never** use `fields = "__all__"`. Always list fields explicitly:
```python
class OrderSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = [
            "id",
            "user",
            "status",
            "total",
            "created",
            "modified",
        ]
        read_only_fields = ["id", "created", "modified"]
```
This prevents accidental exposure of sensitive fields.

## Separate Read/Write Serializers
Use distinct serializers for input and output when they differ:
```python
class OrderCreateSerializer(serializers.Serializer):
    """Input serializer — validates creation data."""
    product_id = serializers.IntegerField()
    quantity = serializers.IntegerField(min_value=1)

class OrderDetailSerializer(serializers.ModelSerializer):
    """Output serializer — shapes API response."""
    user = UserMinimalSerializer(read_only=True)
    items = OrderItemSerializer(many=True, read_only=True)

    class Meta:
        model = Order
        fields = ["id", "user", "status", "items", "total", "created"]
```

## Validation
- Use field-level validators for single-field validation:
  ```python
  def validate_quantity(self, value):
      if value > 100:
          raise serializers.ValidationError("Maximum quantity is 100.")
      return value
  ```
- Use `validate()` for cross-field validation:
  ```python
  def validate(self, attrs):
      if attrs["start_date"] >= attrs["end_date"]:
          raise serializers.ValidationError("End date must be after start date.")
      return attrs
  ```
- Use model-level `UniqueTogetherValidator` and `UniqueValidator` where appropriate.

## SerializerMethodField — Use Sparingly
- Prefer model properties or annotations over `SerializerMethodField`.
- If you use it, keep it simple and read-only:
  ```python
  total_display = serializers.SerializerMethodField()

  def get_total_display(self, obj):
      return f"${obj.total:.2f}"
  ```
- For complex computed fields, annotate in the queryset instead.

## Nested Serializers
- Use nested serializers for read operations:
  ```python
  class OrderDetailSerializer(serializers.ModelSerializer):
      items = OrderItemSerializer(many=True, read_only=True)
  ```
- For write operations, accept PKs or IDs, not nested objects:
  ```python
  class OrderCreateSerializer(serializers.Serializer):
      item_ids = serializers.ListField(child=serializers.IntegerField())
  ```

## Business Logic
- **Never** put business logic in serializers.
- Serializers validate and transform data. Business logic belongs in `services.py`.
- Use `serializer.validated_data` and pass it to service functions:
  ```python
  # In the view
  serializer = OrderCreateSerializer(data=request.data)
  serializer.is_valid(raise_exception=True)
  order = order_service.create_order(user=request.user, **serializer.validated_data)
  return Response(OrderDetailSerializer(order).data, status=201)
  ```

## Performance
- Use `select_related` / `prefetch_related` in the view's queryset, not in the serializer.
- Avoid N+1 queries from nested serializers — always prefetch.
- Use `source` parameter to map serializer fields to model fields without extra queries.

## Error Responses
DRF's default error format is fine. Ensure consistent error structure:
```json
{
    "field_name": ["Error message."],
    "non_field_errors": ["Cross-field error."]
}
```

## File Upload Serializers
```python
class DocumentUploadSerializer(serializers.Serializer):
    file = serializers.FileField()
    description = serializers.CharField(max_length=255, required=False, default="")

    def validate_file(self, value):
        if value.size > 10 * 1024 * 1024:  # 10MB
            raise serializers.ValidationError("File size must be under 10MB.")
        allowed = ["application/pdf", "image/jpeg", "image/png"]
        if value.content_type not in allowed:
            raise serializers.ValidationError("Unsupported file type.")
        return value
```

## Pagination
- Don't handle pagination in serializers. Let the view/viewset handle it.
- Use DRF's built-in pagination classes.
