
# Part II — Django REST Framework

---

## 18. DRF Setup & Configuration

### Installation

```bash
pip install djangorestframework
pip install djangorestframework-simplejwt  # for JWT auth
pip install django-filter                  # for filtering
pip install Markdown                       # for browsable API
```

```python
# settings.py
INSTALLED_APPS = [
    ...
    'rest_framework',
    'rest_framework.authtoken',
    'django_filters',
]

REST_FRAMEWORK = {
    # Default authentication
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
        'rest_framework.authentication.SessionAuthentication',  # for browsable API
    ],
    # Default permissions
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    # Default pagination
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
    # Default filtering
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
    # Throttling
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour',
    },
    # Renderers
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',  # remove in production if desired
    ],
    # Exception handling
    'EXCEPTION_HANDLER': 'core.exceptions.custom_exception_handler',
}
```

```python
# myproject/urls.py
urlpatterns = [
    path('api/v1/', include('api.urls')),
    path('api-auth/', include('rest_framework.urls')),  # browsable API login/logout
]
```

---

## 19. Serializers

### ModelSerializer Basics

```python
# products/serializers.py
from rest_framework import serializers
from .models import Product, Category
from django.contrib.auth import get_user_model

User = get_user_model()


class CategorySerializer(serializers.ModelSerializer):
    class Meta:
        model  = Category
        fields = ['id', 'name', 'slug']
        read_only_fields = ['slug']


class ProductSerializer(serializers.ModelSerializer):
    # Nested serializer — read-only representation of FK
    category = CategorySerializer(read_only=True)
    # Write field for FK (accepts an ID)
    category_id = serializers.PrimaryKeyRelatedField(
        queryset=Category.objects.all(),
        source='category',
        write_only=True
    )
    # Computed field from model property
    is_in_stock = serializers.BooleanField(read_only=True)
    # Source field — map serializer field name to model field
    product_name = serializers.CharField(source='name', read_only=True)

    class Meta:
        model  = Product
        fields = [
            'id', 'product_name', 'slug', 'description',
            'price', 'stock', 'is_in_stock',
            'category', 'category_id',
            'status', 'is_featured',
            'created_at', 'updated_at',
        ]
        read_only_fields = ['id', 'slug', 'created_at', 'updated_at']
        extra_kwargs = {
            'description': {'required': False, 'allow_blank': True},
            'price': {'min_value': 0},
        }
```

### SerializerMethodField — Adding Custom Fields

```python
class ProductSerializer(serializers.ModelSerializer):

    # Add any computed value
    discount_price = serializers.SerializerMethodField()
    created_by_name = serializers.SerializerMethodField()
    image_url = serializers.SerializerMethodField()

    def get_discount_price(self, obj):
        # obj is the Product instance
        if obj.is_featured:
            return float(obj.price) * 0.9  # 10% off featured items
        return float(obj.price)

    def get_created_by_name(self, obj):
        if obj.created_by:
            return obj.created_by.get_full_name() or obj.created_by.username
        return None

    def get_image_url(self, obj):
        request = self.context.get('request')
        if obj.image and request:
            return request.build_absolute_uri(obj.image.url)
        return None

    class Meta:
        model = Product
        fields = ['id', 'name', 'price', 'discount_price',
                  'created_by_name', 'image_url', ...]
```

> **Tricky:** `SerializerMethodField` methods receive the request through `self.context.get('request')`. When using serializers outside of views (e.g., in tasks or management commands), the request won't be in context — always guard with `if request`.

### Field-Level & Object-Level Validation

```python
class ProductSerializer(serializers.ModelSerializer):

    def validate_price(self, value):
        """Field-level validation."""
        if value <= 0:
            raise serializers.ValidationError('Price must be greater than 0.')
        if value > 1_000_000:
            raise serializers.ValidationError('Price cannot exceed 1,000,000.')
        return value

    def validate_name(self, value):
        # Check uniqueness, excluding current instance on updates
        qs = Product.objects.filter(name__iexact=value)
        if self.instance:
            qs = qs.exclude(pk=self.instance.pk)
        if qs.exists():
            raise serializers.ValidationError('A product with this name already exists.')
        return value

    def validate(self, attrs):
        """Object-level validation — attrs contains all validated fields."""
        if attrs.get('status') == 'published' and attrs.get('stock', 0) == 0:
            raise serializers.ValidationError(
                'Cannot publish a product with zero stock.'
            )
        return attrs

    def create(self, validated_data):
        """Override to customize creation."""
        request = self.context.get('request')
        validated_data['created_by'] = request.user
        return super().create(validated_data)

    def update(self, instance, validated_data):
        """Override to customize updates."""
        # Log changes
        if 'price' in validated_data and validated_data['price'] != instance.price:
            PriceHistory.objects.create(
                product=instance,
                old_price=instance.price,
                new_price=validated_data['price']
            )
        return super().update(instance, validated_data)
```

### Nested Writable Serializers

```python
class OrderItemSerializer(serializers.ModelSerializer):
    class Meta:
        model = OrderItem
        fields = ['product', 'quantity', 'unit_price']

class OrderSerializer(serializers.ModelSerializer):
    items = OrderItemSerializer(many=True)

    class Meta:
        model  = Order
        fields = ['id', 'status', 'items', 'total_amount', 'created_at']
        read_only_fields = ['id', 'total_amount', 'created_at']

    def create(self, validated_data):
        items_data = validated_data.pop('items')
        order = Order.objects.create(**validated_data)
        for item_data in items_data:
            OrderItem.objects.create(order=order, **item_data)
        return order

    def update(self, instance, validated_data):
        items_data = validated_data.pop('items', None)
        instance = super().update(instance, validated_data)
        if items_data is not None:
            instance.items.all().delete()  # replace all items
            for item_data in items_data:
                OrderItem.objects.create(order=instance, **item_data)
        return instance
```

### Different Serializers for Read vs Write

```python
class ProductReadSerializer(serializers.ModelSerializer):
    category = CategorySerializer(read_only=True)
    created_by = UserSummarySerializer(read_only=True)

    class Meta:
        model  = Product
        fields = '__all__'


class ProductWriteSerializer(serializers.ModelSerializer):
    class Meta:
        model  = Product
        fields = ['name', 'description', 'price', 'stock', 'category', 'status']

# Use in ViewSet:
def get_serializer_class(self):
    if self.action in ['create', 'update', 'partial_update']:
        return ProductWriteSerializer
    return ProductReadSerializer
```

---

## 20. DRF Views — Function-Based API Views

### `@api_view` Decorator

```python
from rest_framework.decorators import api_view, permission_classes, authentication_classes
from rest_framework.permissions import IsAuthenticated, AllowAny
from rest_framework.response import Response
from rest_framework import status
from .models import Product
from .serializers import ProductSerializer

@api_view(['GET'])
@permission_classes([AllowAny])
def product_list(request):
    """List all published products."""
    products = Product.objects.published().select_related('category')
    serializer = ProductSerializer(products, many=True, context={'request': request})
    return Response(serializer.data)


@api_view(['GET'])
def product_detail(request, pk):
    """Get a single product."""
    product = get_object_or_404(Product, pk=pk)
    serializer = ProductSerializer(product, context={'request': request})
    return Response(serializer.data)


@api_view(['POST'])
@permission_classes([IsAuthenticated])
def product_create(request):
    """Create a new product."""
    serializer = ProductSerializer(data=request.data, context={'request': request})
    if serializer.is_valid():
        serializer.save(created_by=request.user)
        return Response(serializer.data, status=status.HTTP_201_CREATED)
    return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


@api_view(['GET', 'PUT', 'PATCH', 'DELETE'])
@permission_classes([IsAuthenticated])
def product_detail_update_delete(request, pk):
    """GET, UPDATE, DELETE a product."""
    product = get_object_or_404(Product, pk=pk)

    if request.method == 'GET':
        serializer = ProductSerializer(product, context={'request': request})
        return Response(serializer.data)

    elif request.method in ('PUT', 'PATCH'):
        partial = request.method == 'PATCH'
        serializer = ProductSerializer(
            product, data=request.data, partial=partial, context={'request': request}
        )
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    elif request.method == 'DELETE':
        product.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

> **Tricky:** Always pass `context={'request': request}` to serializers when `SerializerMethodField` methods or `HyperlinkedRelatedField` need to build absolute URLs. Forgetting this is a common source of bugs when generating image/file URLs.

---

## 21. DRF Class-Based Views & ViewSets

### APIView

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class ProductListCreateView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request):
        products = Product.objects.published()
        serializer = ProductSerializer(products, many=True, context={'request': request})
        return Response(serializer.data)

    def post(self, request):
        serializer = ProductSerializer(data=request.data, context={'request': request})
        if serializer.is_valid():
            serializer.save(created_by=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

### Generic Views

```python
from rest_framework import generics
from rest_framework.permissions import IsAuthenticated, IsAuthenticatedOrReadOnly

class ProductListCreateView(generics.ListCreateAPIView):
    queryset            = Product.objects.published().select_related('category')
    serializer_class    = ProductSerializer
    permission_classes  = [IsAuthenticatedOrReadOnly]
    filterset_fields    = ['category', 'status']
    search_fields       = ['name', 'description']
    ordering_fields     = ['price', 'created_at', 'name']
    ordering            = ['-created_at']

    def perform_create(self, serializer):
        serializer.save(created_by=self.request.user)

    def get_serializer_context(self):
        context = super().get_serializer_context()
        context['request'] = self.request
        return context


class ProductRetrieveUpdateDestroyView(generics.RetrieveUpdateDestroyAPIView):
    queryset         = Product.objects.all()
    serializer_class = ProductSerializer

    def get_permissions(self):
        if self.request.method in ('PUT', 'PATCH', 'DELETE'):
            return [IsAuthenticated(), IsOwner()]
        return [AllowAny()]

    def perform_destroy(self, instance):
        # Override to soft delete instead of hard delete
        instance.soft_delete()
```

### ViewSets & Routers — The Power Combo

```python
from rest_framework import viewsets, mixins
from rest_framework.decorators import action
from rest_framework.response import Response

class ProductViewSet(viewsets.ModelViewSet):
    """
    Provides: list, create, retrieve, update, partial_update, destroy
    GET    /products/           → list()
    POST   /products/           → create()
    GET    /products/{id}/      → retrieve()
    PUT    /products/{id}/      → update()
    PATCH  /products/{id}/      → partial_update()
    DELETE /products/{id}/      → destroy()
    """
    queryset           = Product.objects.all().select_related('category', 'created_by')
    serializer_class   = ProductSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    filterset_fields   = ['category', 'status', 'is_featured']
    search_fields      = ['name', 'description', 'category__name']
    ordering_fields    = ['price', 'created_at', 'stock']
    ordering           = ['-created_at']

    def get_serializer_class(self):
        if self.action in ['create', 'update', 'partial_update']:
            return ProductWriteSerializer
        return ProductReadSerializer

    def get_queryset(self):
        qs = super().get_queryset()
        if self.request.user.is_authenticated and self.request.query_params.get('mine'):
            return qs.filter(created_by=self.request.user)
        return qs.filter(status='published')

    def perform_create(self, serializer):
        serializer.save(created_by=self.request.user)

    # Custom actions — extra endpoints beyond CRUD
    @action(detail=True, methods=['post'], permission_classes=[IsAuthenticated])
    def toggle_featured(self, request, pk=None):
        """POST /products/{id}/toggle_featured/"""
        product = self.get_object()
        product.is_featured = not product.is_featured
        product.save(update_fields=['is_featured'])
        return Response({'is_featured': product.is_featured})

    @action(detail=False, methods=['get'])
    def featured(self, request):
        """GET /products/featured/ — list all featured products"""
        products   = self.get_queryset().filter(is_featured=True)
        page       = self.paginate_queryset(products)
        serializer = self.get_serializer(page, many=True)
        return self.get_paginated_response(serializer.data)

    @action(detail=True, methods=['post'], permission_classes=[IsAuthenticated])
    def add_to_cart(self, request, pk=None):
        product  = self.get_object()
        quantity = request.data.get('quantity', 1)
        # ... cart logic ...
        return Response({'message': f'Added {quantity} x {product.name} to cart'})
```

```python
# api/urls.py
from rest_framework.routers import DefaultRouter
from products.views import ProductViewSet
from users.views import UserViewSet

router = DefaultRouter()
router.register('products', ProductViewSet, basename='product')
router.register('users', UserViewSet, basename='user')

urlpatterns = router.urls
# Generates all CRUD URLs + custom actions automatically
```

### Read-Only ViewSet

```python
class CategoryViewSet(
    mixins.ListModelMixin,
    mixins.RetrieveModelMixin,
    viewsets.GenericViewSet
):
    """Only exposes GET /categories/ and GET /categories/{id}/"""
    queryset           = Category.objects.all()
    serializer_class   = CategorySerializer
    permission_classes = [AllowAny]
```

---

## 22. Token Authentication for Mobile Apps

### Token Authentication Setup

```python
# settings.py
INSTALLED_APPS += ['rest_framework.authtoken']

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ],
}
```

```bash
python manage.py migrate  # creates authtoken_token table
```

### User Registration with Token

```python
# users/serializers.py
from django.contrib.auth import get_user_model
from rest_framework import serializers
from rest_framework.authtoken.models import Token

User = get_user_model()

class RegisterSerializer(serializers.ModelSerializer):
    password  = serializers.CharField(write_only=True, min_length=8,
                                      style={'input_type': 'password'})
    password2 = serializers.CharField(write_only=True,
                                      style={'input_type': 'password'})
    token     = serializers.SerializerMethodField(read_only=True)

    class Meta:
        model  = User
        fields = ['id', 'email', 'username', 'password', 'password2', 'token']

    def validate_email(self, value):
        if User.objects.filter(email=value).exists():
            raise serializers.ValidationError('This email is already in use.')
        return value

    def validate(self, attrs):
        if attrs['password'] != attrs['password2']:
            raise serializers.ValidationError({'password2': 'Passwords do not match.'})
        return attrs

    def create(self, validated_data):
        validated_data.pop('password2')
        user = User.objects.create_user(**validated_data)
        return user

    def get_token(self, obj):
        token, _ = Token.objects.get_or_create(user=obj)
        return token.key
```

```python
# users/views.py
from rest_framework import generics, status
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.permissions import AllowAny, IsAuthenticated
from rest_framework.authtoken.models import Token
from django.contrib.auth import authenticate

class RegisterView(generics.CreateAPIView):
    serializer_class   = RegisterSerializer
    permission_classes = [AllowAny]

    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        user = serializer.save()
        token, _ = Token.objects.get_or_create(user=user)
        return Response({
            'user': {
                'id': user.pk,
                'email': user.email,
                'username': user.username,
            },
            'token': token.key
        }, status=status.HTTP_201_CREATED)


class LoginView(APIView):
    permission_classes = [AllowAny]

    def post(self, request):
        email    = request.data.get('email')
        password = request.data.get('password')

        if not email or not password:
            return Response(
                {'error': 'Email and password are required.'},
                status=status.HTTP_400_BAD_REQUEST
            )

        user = authenticate(request, username=email, password=password)

        if not user:
            return Response(
                {'error': 'Invalid credentials.'},
                status=status.HTTP_401_UNAUTHORIZED
            )

        token, _ = Token.objects.get_or_create(user=user)
        return Response({
            'token': token.key,
            'user': {
                'id': user.pk,
                'email': user.email,
                'username': user.username,
            }
        })


class LogoutView(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request):
        # Delete the token — invalidates all sessions for this user
        request.user.auth_token.delete()
        return Response({'message': 'Logged out successfully.'}, status=status.HTTP_200_OK)


class ProfileView(generics.RetrieveUpdateAPIView):
    serializer_class   = UserProfileSerializer
    permission_classes = [IsAuthenticated]

    def get_object(self):
        return self.request.user
```

```python
# api/urls.py
urlpatterns = [
    path('auth/register/', RegisterView.as_view(), name='register'),
    path('auth/login/', LoginView.as_view(), name='login'),
    path('auth/logout/', LogoutView.as_view(), name='logout'),
    path('auth/profile/', ProfileView.as_view(), name='profile'),
    ...
]
```

### Using the Token on the Client

```bash
# All authenticated API calls must include the token header:
curl -H "Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b" \
     https://api.mysite.com/api/v1/products/
```

### JWT Authentication (Preferred for Mobile)

```bash
pip install djangorestframework-simplejwt
```

```python
# settings.py
from datetime import timedelta

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
}

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME':  timedelta(minutes=60),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=30),
    'ROTATE_REFRESH_TOKENS':  True,   # issue new refresh token on each refresh
    'BLACKLIST_AFTER_ROTATION': True,  # blacklist old refresh tokens
    'AUTH_HEADER_TYPES': ('Bearer',),
}
```

```python
# urls.py
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView

urlpatterns = [
    path('auth/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('auth/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
]
```

```bash
# Client: POST /auth/token/ → {access, refresh}
# Client: POST /auth/token/refresh/ with {refresh} → {access}
# Requests: Authorization: Bearer <access_token>
```

> **Tricky:** Token auth (DRF's built-in) issues one permanent token per user — logging out on one device logs out all devices (one token per user). JWT gives you short-lived access tokens and long-lived refresh tokens, so you can revoke refresh tokens per device. For mobile apps, JWT is almost always the right choice.

---

## 23. Permissions & Access Control

### Built-in Permission Classes

```python
from rest_framework.permissions import (
    AllowAny,               # anyone can access
    IsAuthenticated,        # must be logged in
    IsAdminUser,            # must be staff/admin
    IsAuthenticatedOrReadOnly,  # read = anyone, write = authenticated
    DjangoModelPermissions, # follows Django model permissions (add/change/delete)
    DjangoObjectPermissions,# per-object permissions
)
```

### Custom Permission Classes

```python
# core/permissions.py
from rest_framework.permissions import BasePermission, SAFE_METHODS

class IsOwnerOrReadOnly(BasePermission):
    """
    Object-level permission. Read for anyone; write only for the owner.
    """
    def has_permission(self, request, view):
        # View-level: allow reads for all, writes for authenticated only
        if request.method in SAFE_METHODS:
            return True
        return request.user and request.user.is_authenticated

    def has_object_permission(self, request, view, obj):
        # Object-level: reads allowed, writes only for owner
        if request.method in SAFE_METHODS:
            return True
        return obj.created_by == request.user


class IsOwner(BasePermission):
    """Only the owner can access — reads and writes."""
    def has_object_permission(self, request, view, obj):
        return obj.created_by == request.user or request.user.is_staff


class IsAdminOrReadOnly(BasePermission):
    def has_permission(self, request, view):
        if request.method in SAFE_METHODS:
            return True
        return request.user and request.user.is_staff


class IsSameUserOrAdmin(BasePermission):
    """Users can only access/modify their own records; admins can access all."""
    def has_object_permission(self, request, view, obj):
        return obj == request.user or request.user.is_staff
```

### Per-Action Permissions in ViewSets

```python
class ProductViewSet(viewsets.ModelViewSet):

    def get_permissions(self):
        if self.action in ['list', 'retrieve']:
            permission_classes = [AllowAny]
        elif self.action in ['create']:
            permission_classes = [IsAuthenticated]
        elif self.action in ['update', 'partial_update', 'destroy']:
            permission_classes = [IsAuthenticated, IsOwnerOrReadOnly]
        else:
            permission_classes = [IsAdminUser]
        return [permission() for permission in permission_classes]
```

### Throttling

```python
from rest_framework.throttling import UserRateThrottle, AnonRateThrottle

class BurstRateThrottle(UserRateThrottle):
    scope = 'burst'

class SustainedRateThrottle(UserRateThrottle):
    scope = 'sustained'

# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'burst': '60/min',
        'sustained': '1000/day',
        'anon': '20/hour',
    }
}

# On a view:
class LoginView(APIView):
    throttle_classes = [AnonRateThrottle]
```

---

## 24. Pagination, Filtering, Search & Ordering

### Pagination

```python
# core/pagination.py
from rest_framework.pagination import PageNumberPagination, CursorPagination

class StandardPagination(PageNumberPagination):
    page_size             = 20
    page_size_query_param = 'page_size'  # ?page_size=50
    max_page_size         = 100
    page_query_param      = 'page'       # ?page=2

    def get_paginated_response(self, data):
        return Response({
            'count':    self.page.paginator.count,
            'next':     self.get_next_link(),
            'previous': self.get_previous_link(),
            'total_pages': self.page.paginator.num_pages,
            'current_page': self.page.number,
            'results': data,
        })


class CursorBasedPagination(CursorPagination):
    """
    Better for real-time feeds — stable even when items are added/removed.
    Returns an opaque cursor instead of page number.
    """
    page_size    = 20
    ordering     = '-created_at'
    cursor_query_param = 'cursor'
```

```python
# Apply globally or per view
class ProductViewSet(viewsets.ModelViewSet):
    pagination_class = StandardPagination
```

### Filtering with django-filter

```python
# products/filters.py
import django_filters
from .models import Product

class ProductFilter(django_filters.FilterSet):
    min_price = django_filters.NumberFilter(field_name='price', lookup_expr='gte')
    max_price = django_filters.NumberFilter(field_name='price', lookup_expr='lte')
    name      = django_filters.CharFilter(field_name='name', lookup_expr='icontains')
    in_stock  = django_filters.BooleanFilter(method='filter_in_stock')
    created_after = django_filters.DateFilter(field_name='created_at', lookup_expr='date__gte')

    def filter_in_stock(self, queryset, name, value):
        if value:
            return queryset.filter(stock__gt=0)
        return queryset.filter(stock=0)

    class Meta:
        model  = Product
        fields = {
            'category': ['exact'],
            'status':   ['exact', 'in'],
            'is_featured': ['exact'],
        }
```

```python
# In viewset
class ProductViewSet(viewsets.ModelViewSet):
    filterset_class = ProductFilter
    filter_backends = [
        DjangoFilterBackend,
        filters.SearchFilter,
        filters.OrderingFilter,
    ]
    search_fields   = ['name', 'description', 'category__name']
    ordering_fields = ['price', 'created_at', 'name', 'stock']
    ordering        = ['-created_at']
```

```bash
# Client usage
GET /api/products/?min_price=10&max_price=100&in_stock=true
GET /api/products/?search=laptop
GET /api/products/?ordering=-price
GET /api/products/?ordering=price,-created_at
GET /api/products/?status__in=published,draft&category=3
```

---

## 25. Advanced Patterns & Tricky Gotchas

### Custom Exception Handler

```python
# core/exceptions.py
from rest_framework.views import exception_handler
from rest_framework.response import Response
from rest_framework import status
import logging

logger = logging.getLogger(__name__)

def custom_exception_handler(exc, context):
    # Call DRF's default exception handler first
    response = exception_handler(exc, context)

    if response is not None:
        # Standardize error format
        error_data = {
            'status': 'error',
            'status_code': response.status_code,
            'errors': response.data,
        }
        # Flatten common structures
        if isinstance(response.data, dict):
            if 'detail' in response.data:
                error_data['message'] = str(response.data['detail'])
        response.data = error_data
    else:
        # Unhandled exceptions
        logger.exception('Unhandled exception', exc_info=exc)
        response = Response({
            'status': 'error',
            'status_code': 500,
            'message': 'An unexpected error occurred.',
        }, status=status.HTTP_500_INTERNAL_SERVER_ERROR)

    return response
```

### Versioning APIs

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
    'VERSION_PARAM': 'version',
}

# urls.py
urlpatterns = [
    path('api/<str:version>/', include('api.urls')),
]
# Requests: GET /api/v1/products/ or /api/v2/products/

# In view:
class ProductViewSet(viewsets.ModelViewSet):
    def get_serializer_class(self):
        if self.request.version == 'v2':
            return ProductV2Serializer
        return ProductSerializer
```

### Content Negotiation & Multiple Formats

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
        'rest_framework_csv.renderers.CSVRenderer',  # pip install djangorestframework-csv
        'rest_framework.renderers.BrowsableAPIRenderer',
    ],
}

# In view — override per view:
from rest_framework.renderers import JSONRenderer
from rest_framework_csv.renderers import CSVRenderer

class ProductExportView(generics.ListAPIView):
    renderer_classes = [CSVRenderer, JSONRenderer]
    # GET /products/export/ → JSON by default
    # GET /products/export/?format=csv → CSV
    # Or: Accept: text/csv header
```

### Signals + DRF — Watch for Double Triggers

> **Tricky:** When a DRF serializer calls `save()`, it triggers model `post_save` signals just like a regular ORM save. If your signal does async work (e.g., sends a task to Celery), make sure the task reads fresh DB state — the signal may fire before related data (M2M, nested creates) is committed.

```python
# WRONG — signal may fire before order items are saved
@receiver(post_save, sender=Order)
def notify_on_order_created(sender, instance, created, **kwargs):
    if created:
        send_order_email.delay(instance.id)  # items might not exist yet!

# RIGHT — use transaction.on_commit
@receiver(post_save, sender=Order)
def notify_on_order_created(sender, instance, created, **kwargs):
    if created:
        from django.db import transaction
        transaction.on_commit(lambda: send_order_email.delay(instance.id))
```

### Optimizing DRF Serializer Queries

```python
class ProductViewSet(viewsets.ModelViewSet):

    def get_queryset(self):
        return Product.objects.all()\
            .select_related('category', 'created_by')\
            .prefetch_related(
                Prefetch('tags', queryset=Tag.objects.only('id', 'name')),
                'images'
            )\
            .only('id', 'name', 'slug', 'price', 'stock', 'status',
                  'category_id', 'created_by_id', 'created_at')
            # .only() restricts columns fetched — huge win for wide tables
```

### Partial Updates — PUT vs PATCH

```python
# PUT — replaces the entire object; all required fields must be sent
# PATCH — partial update; send only fields you want to change

# In serializer:
class ProductSerializer(serializers.ModelSerializer):
    name = serializers.CharField(required=True)  # required on PUT, not PATCH

# In viewset — partial_update() automatically passes partial=True
def partial_update(self, request, *args, **kwargs):
    kwargs['partial'] = True
    return self.update(request, *args, **kwargs)
```

> **Tricky:** When using `partial=True`, the serializer skips validation for missing fields. But validators that check relationships between fields (cross-field `validate()`) only receive the fields present in the request — they won't see unchanged fields unless you explicitly include them.

### Handling File Uploads in DRF

```python
class ProductImageView(APIView):
    parser_classes  = [MultiPartParser, FormParser]
    permission_classes = [IsAuthenticated]

    def post(self, request, pk):
        product = get_object_or_404(Product, pk=pk, created_by=request.user)
        file_obj = request.FILES.get('image')
        if not file_obj:
            return Response({'error': 'No image provided.'}, status=400)

        product.image = file_obj
        product.save(update_fields=['image'])
        return Response({'image_url': request.build_absolute_uri(product.image.url)})
```

```python
# settings.py — DRF default parsers
REST_FRAMEWORK = {
    'DEFAULT_PARSER_CLASSES': [
        'rest_framework.parsers.JSONParser',
        'rest_framework.parsers.MultiPartParser',  # for file uploads
        'rest_framework.parsers.FormParser',
    ],
}
```

### Caching API Responses

```python
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page
from django.views.decorators.vary import vary_on_cookie

class CategoryListView(generics.ListAPIView):
    queryset = Category.objects.all()
    serializer_class = CategorySerializer
    permission_classes = [AllowAny]

    @method_decorator(cache_page(60 * 15))  # cache for 15 minutes
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)


# Or use DRF extensions
# pip install drf-extensions
from rest_framework_extensions.cache.decorators import cache_response

class ProductViewSet(viewsets.ReadOnlyModelViewSet):
    @cache_response(timeout=60 * 5)
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)
```

### Testing DRF APIs

```python
from rest_framework.test import APITestCase, APIClient
from rest_framework.authtoken.models import Token
from django.urls import reverse

class ProductAPITest(APITestCase):

    def setUp(self):
        self.user = User.objects.create_user(
            username='apiuser', email='api@test.com', password='pass123'
        )
        self.token = Token.objects.create(user=self.user)
        self.client = APIClient()

    def authenticate(self):
        self.client.credentials(HTTP_AUTHORIZATION=f'Token {self.token.key}')

    def test_list_products_unauthenticated(self):
        response = self.client.get(reverse('product-list'))
        self.assertEqual(response.status_code, 200)
        self.assertIn('results', response.data)

    def test_create_product_requires_auth(self):
        response = self.client.post(reverse('product-list'), {'name': 'Widget'})
        self.assertEqual(response.status_code, 401)

    def test_create_product_authenticated(self):
        self.authenticate()
        data = {
            'name': 'Test Widget',
            'price': '29.99',
            'stock': 10,
            'status': 'published',
        }
        response = self.client.post(reverse('product-list'), data, format='json')
        self.assertEqual(response.status_code, 201)
        self.assertEqual(response.data['name'], 'Test Widget')
        self.assertTrue(Product.objects.filter(name='Test Widget').exists())

    def test_update_product_not_owner(self):
        other_user = User.objects.create_user(username='other', password='pass123')
        product    = Product.objects.create(
            name='Other Widget', price=9.99, created_by=other_user
        )
        self.authenticate()
        response = self.client.patch(
            reverse('product-detail', kwargs={'pk': product.pk}),
            {'price': '5.00'},
            format='json'
        )
        self.assertEqual(response.status_code, 403)

    def test_file_upload(self):
        self.authenticate()
        import io
        from PIL import Image as PILImage

        img = PILImage.new('RGB', (100, 100), color='red')
        img_io = io.BytesIO()
        img.save(img_io, 'JPEG')
        img_io.seek(0)

        response = self.client.post(
            reverse('product-upload-image', kwargs={'pk': self.product.pk}),
            {'image': img_io},
            format='multipart'
        )
        self.assertEqual(response.status_code, 200)
        self.assertIn('image_url', response.data)
```

---

## Quick Reference Cheat Sheet

### ORM Query Patterns

| Intent | Code |
|---|---|
| Get or 404 | `get_object_or_404(Model, pk=pk)` |
| Get or create | `Model.objects.get_or_create(slug=s, defaults={...})` |
| Atomic increment | `Model.objects.filter(pk=pk).update(count=F('count') + 1)` |
| Bulk insert | `Model.objects.bulk_create(objs, batch_size=100)` |
| N+1 FK | `qs.select_related('fk_field')` |
| N+1 M2M | `qs.prefetch_related('m2m_field')` |
| Partial load | `qs.only('id', 'name')` |
| Complex OR | `qs.filter(Q(a=1) \| Q(b=2))` |
| Exclude | `qs.exclude(status='archived')` |
| Annotation | `qs.annotate(count=Count('related'))` |

### DRF Status Codes

| Scenario | Status |
|---|---|
| Successful GET | `200 OK` |
| Resource created | `201 Created` |
| Update with no content | `204 No Content` |
| Validation error | `400 Bad Request` |
| Not authenticated | `401 Unauthorized` |
| Authenticated but forbidden | `403 Forbidden` |
| Not found | `404 Not Found` |
| Method not allowed | `405 Method Not Allowed` |
| Rate limited | `429 Too Many Requests` |
| Server error | `500 Internal Server Error` |

### Serializer Field Quick Reference

| Need | Field |
|---|---|
| Read-only computed field | `SerializerMethodField` |
| FK as nested object (read) | `NestedSerializer(read_only=True)` |
| FK as ID (write) | `PrimaryKeyRelatedField(queryset=...)` |
| FK as URL | `HyperlinkedRelatedField(view_name=..., read_only=True)` |
| Hidden write field | `serializers.HiddenField(default=...)` |
| Current user auto-set | `serializers.HiddenField(default=CurrentUserDefault())` |
| Current time auto-set | `serializers.DateTimeField(default=timezone.now)` |

### Common Permissions Combo

```python
# Read = anyone, Write = authenticated + owner
permission_classes = [IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly]

# All = admin only
permission_classes = [IsAdminUser]

# Per action in ViewSet
def get_permissions(self):
    if self.action in SAFE_METHODS:
        return [AllowAny()]
    return [IsAuthenticated(), IsOwnerOrReadOnly()]
```

---

*Django & DRF Comprehensive Guide — covering Models, ORM, Forms, Views, CBVs, Admin, Auth, Channels, Celery, DRF Serializers, ViewSets, Token Auth, Permissions, Filtering, and production deployment.*
