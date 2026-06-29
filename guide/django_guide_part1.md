# Django & Django REST Framework — Comprehensive Developer Guide

> A deep-dive reference covering Django and DRF from fundamentals to production-grade patterns — including the tricky parts that take experience to internalize.

---

## Table of Contents

### Part I — Django

1. [Project Structure & Multiple Apps](#1-project-structure--multiple-apps)
2. [Django Models](#2-django-models)
3. [Django ORM](#3-django-orm)
4. [Django Forms — Handling, Validation & Crispy Forms](#4-django-forms--handling-validation--crispy-forms)
5. [Django Views — Function-Based Views](#5-django-views--function-based-views)
6. [Django Class-Based Views (CBVs)](#6-django-class-based-views-cbvs)
7. [Jinja2 Templates](#7-jinja2-templates)
8. [Custom Template Tags, Built-in Tags & Filters](#8-custom-template-tags-built-in-tags--filters)
9. [Django Admin — Customization & Power Features](#9-django-admin--customization--power-features)
10. [Authentication — Signup, Sign-in & Sign-out](#10-authentication--signup-sign-in--sign-out)
11. [Image & File Upload](#11-image--file-upload)
12. [Custom Responses, Exceptions & Mixins](#12-custom-responses-exceptions--mixins)
13. [Multi-Tenancy](#13-multi-tenancy)
14. [Django Channels — WebSockets & Real-Time](#14-django-channels--websockets--real-time)
15. [Django with Celery Beat — Background Tasks](#15-django-with-celery-beat--background-tasks)
16. [Django Test Cases](#16-django-test-cases)
17. [Deployment — AWS Linux & Heroku](#17-deployment--aws-linux--heroku)

### Part II — Django REST Framework

18. [DRF Setup & Configuration](#18-drf-setup--configuration)
19. [Serializers](#19-serializers)
20. [DRF Views — Function-Based API Views](#20-drf-views--function-based-api-views)
21. [DRF Class-Based Views & ViewSets](#21-drf-class-based-views--viewsets)
22. [Token Authentication for Mobile Apps](#22-token-authentication-for-mobile-apps)
23. [Permissions & Access Control](#23-permissions--access-control)
24. [Pagination, Filtering, Search & Ordering](#24-pagination-filtering-search--ordering)
25. [Advanced Patterns & Tricky Gotchas](#25-advanced-patterns--tricky-gotchas)

---

# Part I — Django

---

## 1. Project Structure & Multiple Apps

### What Is a Django App?

A Django **project** is the entire web application. A Django **app** is a self-contained module within the project that handles a specific domain. One project can have many apps, and apps can be reused across projects.

```
myproject/
├── manage.py
├── myproject/              ← project package (settings, urls, wsgi, asgi)
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   ├── wsgi.py
│   └── asgi.py
├── users/                  ← app: handles user accounts
│   ├── migrations/
│   ├── templates/users/
│   ├── static/users/
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── forms.py
│   ├── models.py
│   ├── serializers.py
│   ├── tests.py
│   ├── urls.py
│   └── views.py
├── products/               ← app: handles product catalog
│   └── ...
├── orders/                 ← app: handles orders
│   └── ...
└── requirements.txt
```

### Creating and Registering an App

```bash
python manage.py startapp users
```

Then register in `settings.py`:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    # ...
    'users.apps.UsersConfig',   # preferred — uses AppConfig
    'products',                 # shorthand — works but less explicit
]
```

### AppConfig — the Right Way

```python
# users/apps.py
from django.apps import AppConfig

class UsersConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'users'

    def ready(self):
        # Import signals here — not at module level
        import users.signals  # noqa
```

> **Tricky:** Never import models at module level in `apps.py`. The `ready()` hook runs after the app registry is fully populated — importing models before that causes `AppRegistryNotReady` errors.

### URL Routing Across Apps

```python
# myproject/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('users/', include('users.urls', namespace='users')),
    path('products/', include('products.urls', namespace='products')),
    path('api/', include('api.urls')),
]
```

```python
# users/urls.py
from django.urls import path
from . import views

app_name = 'users'   # must match namespace in include()

urlpatterns = [
    path('login/', views.login_view, name='login'),
    path('register/', views.register_view, name='register'),
    path('profile/<int:pk>/', views.profile_view, name='profile'),
]
```

```django
{# Reverse with namespace in templates #}
<a href="{% url 'users:login' %}">Login</a>
<a href="{% url 'users:profile' pk=user.pk %}">Profile</a>
```

```python
# Reverse in Python code
from django.urls import reverse
url = reverse('users:profile', kwargs={'pk': user.pk})
```

### Cross-App Model Relationships

```python
# orders/models.py
from django.conf import settings

class Order(models.Model):
    # Always use AUTH_USER_MODEL — never import User directly
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='orders'
    )
    product = models.ForeignKey(
        'products.Product',   # string reference avoids circular imports
        on_delete=models.SET_NULL,
        null=True
    )
```

> **Tricky:** Always reference `settings.AUTH_USER_MODEL` for user FK relationships, not `from django.contrib.auth.models import User`. If you ever swap to a custom User model, string references keep working; direct imports break.

### Settings Management (Dev vs Production)

```python
# myproject/settings/
#   base.py      ← shared settings
#   development.py
#   production.py
```

```python
# base.py
BASE_DIR = Path(__file__).resolve().parent.parent.parent

# development.py
from .base import *
DEBUG = True
DATABASES = { 'default': { 'ENGINE': 'django.db.backends.sqlite3', ... } }

# production.py
from .base import *
DEBUG = False
ALLOWED_HOSTS = ['yourdomain.com']
```

```bash
# Run with specific settings
python manage.py runserver --settings=myproject.settings.development
```

Or set `DJANGO_SETTINGS_MODULE` in `.env`:

```bash
DJANGO_SETTINGS_MODULE=myproject.settings.production
```

---

## 2. Django Models

### Model Basics & Field Types

```python
from django.db import models
from django.contrib.auth import get_user_model
from django.utils import timezone

User = get_user_model()

class Category(models.Model):
    name = models.CharField(max_length=100, unique=True)
    slug = models.SlugField(max_length=100, unique=True)

    class Meta:
        verbose_name_plural = 'categories'
        ordering = ['name']

    def __str__(self):
        return self.name


class Product(models.Model):

    class Status(models.TextChoices):
        DRAFT     = 'draft',     'Draft'
        PUBLISHED = 'published', 'Published'
        ARCHIVED  = 'archived',  'Archived'

    name        = models.CharField(max_length=255)
    slug        = models.SlugField(max_length=255, unique=True)
    description = models.TextField(blank=True)
    price       = models.DecimalField(max_digits=10, decimal_places=2)
    stock       = models.PositiveIntegerField(default=0)
    status      = models.CharField(
                    max_length=10,
                    choices=Status.choices,
                    default=Status.DRAFT
                )
    category    = models.ForeignKey(
                    Category,
                    on_delete=models.SET_NULL,
                    null=True, blank=True,
                    related_name='products'
                )
    created_by  = models.ForeignKey(
                    User,
                    on_delete=models.SET_NULL,
                    null=True,
                    related_name='products'
                )
    image       = models.ImageField(upload_to='products/%Y/%m/', blank=True, null=True)
    is_featured = models.BooleanField(default=False)
    created_at  = models.DateTimeField(auto_now_add=True)
    updated_at  = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['status', '-created_at']),
            models.Index(fields=['slug']),
        ]
        constraints = [
            models.CheckConstraint(
                check=models.Q(price__gte=0),
                name='product_price_non_negative'
            )
        ]

    def __str__(self):
        return self.name

    def get_absolute_url(self):
        from django.urls import reverse
        return reverse('products:detail', kwargs={'slug': self.slug})

    @property
    def is_in_stock(self):
        return self.stock > 0
```

### Abstract Base Models — DRY Timestamps

```python
# core/models.py
class TimeStampedModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True  # No DB table created for this


class SoftDeleteModel(models.Model):
    deleted_at = models.DateTimeField(null=True, blank=True)

    def soft_delete(self):
        self.deleted_at = timezone.now()
        self.save(update_fields=['deleted_at'])

    @property
    def is_deleted(self):
        return self.deleted_at is not None

    class Meta:
        abstract = True


# products/models.py
class Product(TimeStampedModel, SoftDeleteModel):
    # inherits created_at, updated_at, deleted_at, soft_delete()
    name = models.CharField(max_length=255)
    ...
```

### Model Inheritance Types

```python
# 1. Abstract (most common — shared fields, no shared table)
class Animal(models.Model):
    name = models.CharField(max_length=100)
    class Meta:
        abstract = True

class Dog(Animal):  # gets its own table with `name` column
    breed = models.CharField(max_length=100)


# 2. Multi-table inheritance (separate tables, implicit OneToOne)
class Place(models.Model):
    name = models.CharField(max_length=100)

class Restaurant(Place):  # Restaurant table + Place table; Restaurant has a place_ptr_id
    serves_pizza = models.BooleanField()


# 3. Proxy models (same table, different Python class / behavior)
class PublishedProduct(Product):
    objects = PublishedManager()  # custom manager only

    class Meta:
        proxy = True
        ordering = ['-created_at']
```

> **Tricky:** Multi-table inheritance creates a hidden `OneToOneField` (the `place_ptr`). Accessing `restaurant.name` hits TWO database tables via a JOIN. Be careful with performance in large datasets — abstract inheritance is usually the better choice.

### Custom Managers & QuerySets

```python
class ProductQuerySet(models.QuerySet):
    def published(self):
        return self.filter(status=Product.Status.PUBLISHED)

    def in_stock(self):
        return self.filter(stock__gt=0)

    def featured(self):
        return self.filter(is_featured=True)


class ProductManager(models.Manager):
    def get_queryset(self):
        return ProductQuerySet(self.model, using=self._db)

    def published(self):
        return self.get_queryset().published()

    def in_stock(self):
        return self.get_queryset().in_stock()


class Product(models.Model):
    ...
    objects = ProductManager()

# Usage — chainable!
Product.objects.published().in_stock().filter(category__name='Electronics')
```

### Model `save()` Override vs Signals

```python
# Option 1: Override save() — good for logic intrinsic to the model
class Product(models.Model):
    name = models.CharField(max_length=255)
    slug = models.SlugField(blank=True)

    def save(self, *args, **kwargs):
        if not self.slug:
            from django.utils.text import slugify
            self.slug = slugify(self.name)
        super().save(*args, **kwargs)
```

> **Tricky:** `save()` overrides are NOT called by `QuerySet.update()` or `bulk_create()`. If you need logic to run on bulk operations, use signals or handle it at the application layer. This is a very common source of bugs.

### Model Signals

```python
# users/signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.contrib.auth import get_user_model
from .models import Profile

User = get_user_model()

@receiver(post_save, sender=User)
def create_user_profile(sender, instance, created, **kwargs):
    if created:
        Profile.objects.create(user=instance)

@receiver(post_save, sender=User)
def save_user_profile(sender, instance, **kwargs):
    instance.profile.save()
```

```python
# users/apps.py
class UsersConfig(AppConfig):
    name = 'users'

    def ready(self):
        import users.signals  # register signal handlers
```

> **Tricky:** Signals fire inside the same database transaction as the triggering operation. If your signal handler raises an exception, it rolls back the original save too. Also, `post_save` fires even on `update()` calls if you use `save()`, but NOT if you use `QuerySet.update()`.

---

## 3. Django ORM

### Query Basics

```python
# All objects
Product.objects.all()

# Filter (AND conditions)
Product.objects.filter(status='published', stock__gt=0)

# Exclude
Product.objects.exclude(status='archived')

# Get single object — raises DoesNotExist or MultipleObjectsReturned
product = Product.objects.get(pk=1)

# get_or_create — returns (instance, created)
product, created = Product.objects.get_or_create(
    slug='my-product',
    defaults={'name': 'My Product', 'price': 9.99}
)

# update_or_create
product, created = Product.objects.update_or_create(
    slug='my-product',
    defaults={'price': 14.99, 'stock': 100}
)
```

### Field Lookups

```python
# Exact match (default)
Product.objects.filter(name='Widget')         # name__exact='Widget'

# Case-insensitive
Product.objects.filter(name__iexact='widget')

# Contains
Product.objects.filter(name__contains='Widget')
Product.objects.filter(name__icontains='widget')  # case-insensitive

# Comparison
Product.objects.filter(price__lt=100)          # less than
Product.objects.filter(price__lte=100)         # less than or equal
Product.objects.filter(price__gt=50, price__lt=100)
Product.objects.filter(price__range=(50, 100)) # BETWEEN 50 AND 100

# In list
Product.objects.filter(status__in=['published', 'draft'])

# Null checks
Product.objects.filter(image__isnull=True)

# Date/time
Product.objects.filter(created_at__date=date.today())
Product.objects.filter(created_at__year=2024)
Product.objects.filter(created_at__month=12)

# Starts/ends with
Product.objects.filter(name__startswith='Pro')
Product.objects.filter(name__endswith='Plus')
```

### Q Objects — Complex Queries

```python
from django.db.models import Q

# OR condition
Product.objects.filter(Q(status='published') | Q(is_featured=True))

# Negation
Product.objects.filter(~Q(status='archived'))

# Combined
Product.objects.filter(
    (Q(status='published') | Q(is_featured=True)) &
    Q(stock__gt=0) &
    ~Q(category__isnull=True)
)
```

### Annotations & Aggregations

```python
from django.db.models import Count, Sum, Avg, Max, Min, F, Value
from django.db.models.functions import Coalesce

# Aggregate across entire queryset
Product.objects.aggregate(
    total=Count('id'),
    avg_price=Avg('price'),
    max_price=Max('price'),
    total_stock=Sum('stock')
)
# Returns a dict: {'total': 42, 'avg_price': Decimal('29.99'), ...}

# Annotate — adds a computed value to each object
from django.db.models import Count

Category.objects.annotate(
    product_count=Count('products'),
    published_count=Count('products', filter=Q(products__status='published'))
)

# F expressions — reference another field in the same row (no Python round-trip)
# Find products where stock < reorder_level
Product.objects.filter(stock__lt=F('reorder_level'))

# F expression in update (atomic, race-condition safe)
Product.objects.filter(pk=1).update(stock=F('stock') - 1)

# Coalesce — first non-null value
Product.objects.annotate(
    display_price=Coalesce('sale_price', 'price')
)
```

> **Tricky:** `F()` expressions are evaluated at the database level — they're atomic and race-condition safe for operations like incrementing counters. Using Python (`product.stock -= 1; product.save()`) is NOT atomic and will cause data races in concurrent environments.

### select_related & prefetch_related — The N+1 Problem

```python
# BAD — N+1 query problem
products = Product.objects.all()
for product in products:
    print(product.category.name)  # Each iteration hits the DB!
# 1 query for products + N queries for categories = N+1 total

# GOOD — select_related for ForeignKey / OneToOne (SQL JOIN)
products = Product.objects.select_related('category', 'created_by').all()
for product in products:
    print(product.category.name)  # No extra queries!
# 1 query total

# GOOD — prefetch_related for ManyToMany / reverse FK (separate query + Python join)
categories = Category.objects.prefetch_related('products').all()
for category in categories:
    print(category.products.all())  # No extra queries — data already prefetched
# 2 queries total

# Prefetch with custom queryset
from django.db.models import Prefetch

categories = Category.objects.prefetch_related(
    Prefetch(
        'products',
        queryset=Product.objects.filter(status='published').order_by('-created_at'),
        to_attr='published_products'  # stores result in a list attribute
    )
)
for category in categories:
    category.published_products  # list of published products, no extra query
```

> **Tricky:** `select_related` follows FK chains using a SQL JOIN — good for one-to-one or many-to-one. `prefetch_related` does a second query and joins in Python — necessary for many-to-many or reverse FK. Mixing them incorrectly (e.g., using `select_related` on a M2M) raises an error.

### `only()` and `defer()` — Partial Loading

```python
# Only load specific fields (others are deferred)
Product.objects.only('id', 'name', 'price')

# Defer specific fields (load everything else)
Product.objects.defer('description')  # skip large text fields

# Accessing a deferred field triggers a new query per object!
product = Product.objects.only('name').first()
print(product.price)  # triggers a NEW query — be careful
```

### Bulk Operations

```python
# bulk_create — single INSERT for many objects
products = [Product(name=f'Product {i}', price=9.99) for i in range(1000)]
Product.objects.bulk_create(products, batch_size=100)
# NOTE: signals (pre_save, post_save) are NOT fired
# NOTE: save() overrides are NOT called

# bulk_update
for product in products:
    product.price = 14.99
Product.objects.bulk_update(products, ['price'], batch_size=100)

# QuerySet update — single UPDATE statement
Product.objects.filter(status='draft').update(status='published')
# NOTE: signals are NOT fired; save() overrides are NOT called

# delete
Product.objects.filter(stock=0).delete()
# Returns (count, {model: count}) tuple
```

### Transactions

```python
from django.db import transaction

# Option 1: Decorator
@transaction.atomic
def create_order(user, cart):
    order = Order.objects.create(user=user)
    for item in cart:
        OrderItem.objects.create(order=order, product=item.product, qty=item.qty)
        item.product.stock -= item.qty
        item.product.save()
    # If any step fails, the whole thing rolls back

# Option 2: Context manager
def process_payment(order_id, amount):
    with transaction.atomic():
        order = Order.objects.select_for_update().get(pk=order_id)
        # select_for_update() locks the row until transaction ends
        payment = Payment.objects.create(order=order, amount=amount)
        order.status = 'paid'
        order.save()

# Savepoints — nested atomics
def complex_operation():
    with transaction.atomic():
        do_step_one()
        try:
            with transaction.atomic():  # creates a savepoint
                risky_step()
        except Exception:
            pass  # rolls back to savepoint, outer transaction still alive
        do_step_three()
```

> **Tricky:** `select_for_update()` acquires a row-level lock. If you forget it in concurrent write scenarios (e.g., decrementing stock), two requests can read the same value and both think they can complete the purchase — classic race condition. Always use `select_for_update()` inside `transaction.atomic()` for critical write paths.

### Raw SQL When You Need It

```python
# Raw queryset — maps to model instances
products = Product.objects.raw('SELECT * FROM products_product WHERE price < %s', [100])

# Cursor — for non-ORM queries, aggregations, reporting
from django.db import connection

with connection.cursor() as cursor:
    cursor.execute("""
        SELECT category_id, COUNT(*) as cnt, AVG(price) as avg_price
        FROM products_product
        WHERE status = %s
        GROUP BY category_id
        HAVING COUNT(*) > 5
    """, ['published'])
    rows = cursor.fetchall()
```

---

## 4. Django Forms — Handling, Validation & Crispy Forms

### Form Basics

```python
# forms.py
from django import forms
from .models import Product

# Plain Form
class ContactForm(forms.Form):
    name    = forms.CharField(max_length=100)
    email   = forms.EmailField()
    message = forms.CharField(widget=forms.Textarea(attrs={'rows': 5}))
    subject = forms.ChoiceField(choices=[
        ('support', 'Support'),
        ('billing', 'Billing'),
        ('general', 'General'),
    ])

# ModelForm — fields auto-generated from model
class ProductForm(forms.ModelForm):
    class Meta:
        model  = Product
        fields = ['name', 'description', 'price', 'stock', 'category', 'image']
        # or: exclude = ['created_by', 'created_at']
        widgets = {
            'description': forms.Textarea(attrs={'rows': 4}),
            'price': forms.NumberInput(attrs={'step': '0.01'}),
        }
        labels = {
            'name': 'Product Name',
        }
        help_texts = {
            'slug': 'Auto-generated from name if left blank.',
        }
```

### Handling Forms in Views

```python
# views.py
from django.shortcuts import render, redirect
from django.contrib import messages
from .forms import ProductForm

def product_create(request):
    if request.method == 'POST':
        form = ProductForm(request.POST, request.FILES)  # request.FILES for file uploads
        if form.is_valid():
            product = form.save(commit=False)  # don't hit DB yet
            product.created_by = request.user  # set fields not in form
            product.save()
            form.save_m2m()  # required after commit=False if there are M2M fields
            messages.success(request, 'Product created successfully.')
            return redirect('products:detail', slug=product.slug)
    else:
        form = ProductForm()

    return render(request, 'products/create.html', {'form': form})


def product_edit(request, pk):
    product = get_object_or_404(Product, pk=pk)
    if request.method == 'POST':
        form = ProductForm(request.POST, request.FILES, instance=product)
        if form.is_valid():
            form.save()
            return redirect('products:detail', slug=product.slug)
    else:
        form = ProductForm(instance=product)  # pre-populated with existing data
    return render(request, 'products/edit.html', {'form': form, 'product': product})
```

> **Tricky:** `commit=False` with M2M fields requires manually calling `form.save_m2m()` after saving the instance. Django can't save the M2M relationship until the instance has a primary key.

### Form Validation

```python
class RegistrationForm(forms.Form):
    username        = forms.CharField(max_length=50)
    email           = forms.EmailField()
    password        = forms.CharField(widget=forms.PasswordInput)
    confirm_password = forms.CharField(widget=forms.PasswordInput)
    age             = forms.IntegerField()

    # 1. Field-level validation — clean_<fieldname>()
    def clean_username(self):
        username = self.cleaned_data.get('username')
        if User.objects.filter(username=username).exists():
            raise forms.ValidationError('This username is already taken.')
        if len(username) < 3:
            raise forms.ValidationError('Username must be at least 3 characters.')
        return username  # MUST return the cleaned value

    def clean_age(self):
        age = self.cleaned_data.get('age')
        if age < 18:
            raise forms.ValidationError('You must be at least 18 years old.')
        return age

    # 2. Cross-field validation — clean()
    def clean(self):
        cleaned_data = super().clean()
        password         = cleaned_data.get('password')
        confirm_password = cleaned_data.get('confirm_password')

        if password and confirm_password and password != confirm_password:
            # attach error to specific field
            self.add_error('confirm_password', 'Passwords do not match.')
            # or raise for non-field error:
            # raise forms.ValidationError('Passwords do not match.')

        return cleaned_data
```

### Crispy Forms

```bash
pip install crispy-bootstrap5
```

```python
# settings.py
INSTALLED_APPS = [
    ...
    'crispy_forms',
    'crispy_bootstrap5',
]
CRISPY_ALLOWED_TEMPLATE_PACKS = 'bootstrap5'
CRISPY_TEMPLATE_PACK = 'bootstrap5'
```

```python
# forms.py
from crispy_forms.helper import FormHelper
from crispy_forms.layout import Layout, Submit, Row, Column, Field, HTML, Div

class ProductForm(forms.ModelForm):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.helper = FormHelper()
        self.helper.form_method = 'post'
        self.helper.form_enctype = 'multipart/form-data'  # needed for file uploads
        self.helper.layout = Layout(
            Row(
                Column('name', css_class='col-md-8'),
                Column('price', css_class='col-md-4'),
            ),
            Row(
                Column('stock', css_class='col-md-6'),
                Column('category', css_class='col-md-6'),
            ),
            Field('description', rows=4),
            Field('image'),
            HTML('<hr>'),
            Submit('submit', 'Save Product', css_class='btn btn-primary')
        )
```

```django
{# template — only 3 lines needed #}
{% load crispy_forms_tags %}
<form method="post" enctype="multipart/form-data">
    {% csrf_token %}
    {% crispy form %}
</form>
```

---

## 5. Django Views — Function-Based Views

### The Basic Pattern

```python
from django.shortcuts import render, get_object_or_404, redirect
from django.http import HttpResponse, JsonResponse, Http404
from django.contrib.auth.decorators import login_required, permission_required
from django.views.decorators.http import require_http_methods, require_POST, require_GET

def product_list(request):
    products = Product.objects.published().select_related('category')
    return render(request, 'products/list.html', {'products': products})

@login_required(login_url='/users/login/')
def product_create(request):
    ...

@require_POST  # returns 405 if not POST
def delete_product(request, pk):
    product = get_object_or_404(Product, pk=pk, created_by=request.user)
    product.delete()
    return redirect('products:list')

@require_http_methods(['GET', 'POST'])
def product_form(request, pk=None):
    ...
```

### Handling GET Parameters & Pagination

```python
from django.core.paginator import Paginator, EmptyPage, PageNotAnInteger

def product_list(request):
    queryset = Product.objects.published().select_related('category')

    # Filtering
    category = request.GET.get('category')
    search   = request.GET.get('q', '').strip()
    sort     = request.GET.get('sort', '-created_at')

    if category:
        queryset = queryset.filter(category__slug=category)
    if search:
        queryset = queryset.filter(
            Q(name__icontains=search) | Q(description__icontains=search)
        )

    allowed_sorts = ['name', '-name', 'price', '-price', '-created_at']
    if sort in allowed_sorts:
        queryset = queryset.order_by(sort)

    # Pagination
    paginator = Paginator(queryset, per_page=12)
    page_number = request.GET.get('page', 1)

    try:
        page = paginator.page(page_number)
    except (PageNotAnInteger, EmptyPage):
        page = paginator.page(1)

    return render(request, 'products/list.html', {
        'products': page,
        'paginator': paginator,
        'page': page,
        'categories': Category.objects.all(),
    })
```

```django
{# Pagination template snippet — preserves other GET params #}
<nav>
  {% if page.has_previous %}
    <a href="?page={{ page.previous_page_number }}&q={{ request.GET.q }}">Previous</a>
  {% endif %}
  <span>Page {{ page.number }} of {{ paginator.num_pages }}</span>
  {% if page.has_next %}
    <a href="?page={{ page.next_page_number }}&q={{ request.GET.q }}">Next</a>
  {% endif %}
</nav>
```

### JsonResponse & AJAX Views

```python
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt
import json

def product_stock_api(request, pk):
    product = get_object_or_404(Product, pk=pk)
    return JsonResponse({
        'id': product.pk,
        'name': product.name,
        'stock': product.stock,
        'in_stock': product.is_in_stock,
    })

@login_required
@require_POST
def toggle_featured(request, pk):
    product = get_object_or_404(Product, pk=pk)
    product.is_featured = not product.is_featured
    product.save(update_fields=['is_featured'])
    return JsonResponse({'is_featured': product.is_featured})
```

---

## 6. Django Class-Based Views (CBVs)

### Why CBVs?

CBVs let you organize view logic by HTTP method (get, post, put) and reuse behavior through inheritance and mixins. The tradeoff: more power and DRY code, but the dispatch and MRO (method resolution order) magic can be confusing.

### Generic CBVs

```python
from django.views.generic import (
    ListView, DetailView, CreateView, UpdateView, DeleteView, TemplateView, View
)
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin
from django.urls import reverse_lazy

class ProductListView(ListView):
    model = Product
    template_name = 'products/list.html'
    context_object_name = 'products'  # name in template (default: 'object_list')
    paginate_by = 12
    ordering = ['-created_at']

    def get_queryset(self):
        qs = super().get_queryset().select_related('category')
        q = self.request.GET.get('q')
        if q:
            qs = qs.filter(name__icontains=q)
        return qs

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['categories'] = Category.objects.all()
        context['search_query'] = self.request.GET.get('q', '')
        return context


class ProductDetailView(DetailView):
    model = Product
    template_name = 'products/detail.html'
    context_object_name = 'product'
    slug_field = 'slug'
    slug_url_kwarg = 'slug'


class ProductCreateView(LoginRequiredMixin, CreateView):
    model = Product
    form_class = ProductForm
    template_name = 'products/form.html'
    success_url = reverse_lazy('products:list')
    login_url = '/users/login/'

    def form_valid(self, form):
        form.instance.created_by = self.request.user
        return super().form_valid(form)


class ProductUpdateView(LoginRequiredMixin, UpdateView):
    model = Product
    form_class = ProductForm
    template_name = 'products/form.html'

    def get_success_url(self):
        return self.object.get_absolute_url()

    def get_queryset(self):
        # Ensure users can only edit their own products
        return super().get_queryset().filter(created_by=self.request.user)


class ProductDeleteView(LoginRequiredMixin, DeleteView):
    model = Product
    template_name = 'products/confirm_delete.html'
    success_url = reverse_lazy('products:list')

    def get_queryset(self):
        return super().get_queryset().filter(created_by=self.request.user)
```

### URLs for CBVs

```python
# products/urls.py
from django.urls import path
from .views import ProductListView, ProductDetailView, ProductCreateView, ...

urlpatterns = [
    path('', ProductListView.as_view(), name='list'),
    path('create/', ProductCreateView.as_view(), name='create'),
    path('<slug:slug>/', ProductDetailView.as_view(), name='detail'),
    path('<slug:slug>/edit/', ProductUpdateView.as_view(), name='update'),
    path('<slug:slug>/delete/', ProductDeleteView.as_view(), name='delete'),
]
```

### Custom Mixins

```python
# core/mixins.py
class OwnershipMixin:
    """Restricts access to the object's owner."""

    def get_queryset(self):
        return super().get_queryset().filter(created_by=self.request.user)

    def handle_no_permission(self):
        from django.core.exceptions import PermissionDenied
        raise PermissionDenied


class AjaxResponseMixin:
    """Responds with JSON when the request is AJAX."""

    def form_invalid(self, form):
        if self.request.headers.get('x-requested-with') == 'XMLHttpRequest':
            from django.http import JsonResponse
            return JsonResponse({'errors': form.errors}, status=400)
        return super().form_invalid(form)

    def form_valid(self, form):
        response = super().form_valid(form)
        if self.request.headers.get('x-requested-with') == 'XMLHttpRequest':
            from django.http import JsonResponse
            return JsonResponse({'redirect': self.get_success_url()})
        return response


# Usage
class ProductCreateView(LoginRequiredMixin, AjaxResponseMixin, CreateView):
    ...
```

> **Tricky:** Mixin order in multiple inheritance matters. Python's MRO (C3 linearization) resolves method calls left to right. `LoginRequiredMixin` must come before `CreateView` so its `dispatch()` check runs first. If you see weird behavior, always check the MRO: `ProductCreateView.__mro__`.

### The `View` Base Class — Explicit HTTP Methods

```python
class ProductAPI(LoginRequiredMixin, View):

    def get(self, request, pk):
        product = get_object_or_404(Product, pk=pk)
        return JsonResponse({'name': product.name, 'price': str(product.price)})

    def post(self, request, pk):
        product = get_object_or_404(Product, pk=pk, created_by=request.user)
        data = json.loads(request.body)
        product.price = data.get('price', product.price)
        product.save()
        return JsonResponse({'status': 'updated'})

    def delete(self, request, pk):
        product = get_object_or_404(Product, pk=pk, created_by=request.user)
        product.delete()
        return JsonResponse({'status': 'deleted'})
```

---

## 7. Jinja2 Templates

### Django Template Language vs Jinja2

Django ships with its own template engine (DTL). Jinja2 is a separate, more powerful template engine. Django supports both since 1.8.

| Feature | DTL | Jinja2 |
|---|---|---|
| Speed | Slower | ~2x faster |
| Python expressions | No | Yes |
| Inline conditionals | No | Yes (`x if cond else y`) |
| Function calls in templates | Limited | Full |
| Custom filters | Via `register` | Regular Python functions |

### Setup

```bash
pip install Jinja2
```

```python
# settings.py
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.jinja2.Jinja2',
        'DIRS': [BASE_DIR / 'templates' / 'jinja2'],
        'APP_DIRS': True,  # looks for jinja2/ subfolder in each app
        'OPTIONS': {
            'environment': 'myproject.jinja2.environment',
        },
    },
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```

```python
# myproject/jinja2.py
from django.contrib.staticfiles.storage import staticfiles_storage
from django.urls import reverse
from jinja2 import Environment

def environment(**options):
    env = Environment(**options)
    env.globals.update({
        'static': staticfiles_storage.url,
        'url': reverse,
    })
    return env
```

### Jinja2 Template Syntax

```jinja2
{# Base template — templates/jinja2/base.html #}
<!DOCTYPE html>
<html>
<head>
  <title>{% block title %}My Site{% endblock %}</title>
  <link rel="stylesheet" href="{{ static('css/main.css') }}">
</head>
<body>
  <nav>
    <a href="{{ url('products:list') }}">Products</a>
    {% if request.user.is_authenticated %}
      <a href="{{ url('users:logout') }}">Logout</a>
    {% else %}
      <a href="{{ url('users:login') }}">Login</a>
    {% endif %}
  </nav>

  <main>
    {% for message in messages %}
      <div class="alert alert-{{ message.tags }}">{{ message }}</div>
    {% endfor %}
    {% block content %}{% endblock %}
  </main>
</body>
</html>
```

```jinja2
{# Child template #}
{% extends "base.html" %}

{% block title %}Products — {{ super() }}{% endblock %}

{% block content %}
  <h1>Products</h1>
  {% for product in products %}
    <div class="product-card">
      <h2>{{ product.name }}</h2>
      <p>{{ product.price | round(2) }}</p>
      {# Inline expression — not available in DTL #}
      <span class="{{ 'in-stock' if product.is_in_stock else 'out-of-stock' }}">
        {{ 'In Stock' if product.is_in_stock else 'Out of Stock' }}
      </span>
    </div>
  {% else %}
    <p>No products found.</p>
  {% endfor %}
{% endblock %}
```

---

## 8. Custom Template Tags, Built-in Tags & Filters

### Built-in Template Tags

```django
{# if / elif / else #}
{% if user.is_authenticated and user.is_staff %}
  <a href="/admin/">Admin</a>
{% elif user.is_authenticated %}
  <a href="/profile/">Profile</a>
{% else %}
  <a href="/login/">Login</a>
{% endif %}

{# for loop with forloop variable #}
{% for product in products %}
  <div class="{% if forloop.first %}first{% endif %} {% if forloop.last %}last{% endif %}">
    {{ forloop.counter }}. {{ product.name }}  {# 1-based counter #}
    {{ forloop.counter0 }}                     {# 0-based counter #}
    {{ forloop.revcounter }}                   {# countdown #}
  </div>
{% empty %}
  <p>No items.</p>
{% endfor %}

{# url #}
<a href="{% url 'products:detail' slug=product.slug %}">View</a>

{# static #}
{% load static %}
<img src="{% static 'images/logo.png' %}" alt="Logo">

{# with — assign a variable in template #}
{% with total=products|length %}
  <p>{{ total }} product{{ total|pluralize }}</p>
{% endwith %}

{# include — render another template #}
{% include 'partials/product_card.html' with product=product only %}
{# 'only' restricts context to what you pass explicitly #}

{# block + extends — covered in templates section #}

{# csrf_token — required in all POST forms #}
<form method="post">
  {% csrf_token %}
  ...
</form>
```

### Built-in Filters

```django
{{ product.name|lower }}
{{ product.name|upper }}
{{ product.name|title }}
{{ product.name|truncatewords:10 }}
{{ product.description|truncatechars:200 }}
{{ product.description|linebreaksbr }}    {# \n → <br> #}
{{ product.description|striptags }}        {# strip HTML #}

{{ product.price|floatformat:2 }}          {# 9.9 → "9.90" #}
{{ order_total|intcomma }}                 {# 10000 → "10,000" #}

{{ product.created_at|date:"M d, Y" }}    {# Jan 15, 2024 #}
{{ product.created_at|timesince }}         {# "3 days ago" #}
{{ product.created_at|timeuntil }}         {# "in 2 hours" #}

{{ products|length }}
{{ products|first }}
{{ products|last }}

{{ value|default:"N/A" }}                  {# if falsy, show default #}
{{ value|default_if_none:"N/A" }}          {# only if None #}

{{ html_content|safe }}                    {# mark as safe — don't escape #}
{{ user_input|escape }}                    {# escape HTML (default behavior) #}

{% load humanize %}
{{ 1000000|intword }}                      {# "1.0 million" #}
```

### Custom Template Tags

```
# Directory structure required:
products/
├── templatetags/
│   ├── __init__.py
│   └── product_tags.py
```

```python
# products/templatetags/product_tags.py
from django import template
from django.utils.html import format_html, mark_safe
from products.models import Product, Category

register = template.Library()

# 1. Simple tag — returns a value
@register.simple_tag
def get_featured_products(count=4):
    return Product.objects.filter(is_featured=True)[:count]

@register.simple_tag(takes_context=True)
def current_url_with_param(context, key, value):
    """Adds/updates a GET param to the current URL."""
    request = context['request']
    params = request.GET.copy()
    params[key] = value
    return f"?{params.urlencode()}"


# 2. Filter — transforms a value
@register.filter(name='currency')
def currency_format(value, symbol='$'):
    try:
        return f"{symbol}{float(value):,.2f}"
    except (ValueError, TypeError):
        return value

@register.filter
def subtract(value, arg):
    return value - arg

@register.filter
def multiply(value, arg):
    return value * arg


# 3. Inclusion tag — renders a template with context
@register.inclusion_tag('partials/product_card.html', takes_context=True)
def product_card(context, product, show_price=True):
    return {
        'product': product,
        'show_price': show_price,
        'request': context['request'],
    }


# 4. Simple tag returning HTML
@register.simple_tag
def stock_badge(product):
    if product.stock > 10:
        css = 'success'
        text = 'In Stock'
    elif product.stock > 0:
        css = 'warning'
        text = f'Only {product.stock} left'
    else:
        css = 'danger'
        text = 'Out of Stock'
    return format_html('<span class="badge bg-{}">{}</span>', css, text)
```

```django
{# Usage in templates #}
{% load product_tags %}

{# simple_tag #}
{% get_featured_products as featured %}
{% for product in featured %}...{% endfor %}

{# simple_tag with assignment #}
{% current_url_with_param 'sort' 'price' as sort_url %}
<a href="{{ sort_url }}">Sort by Price</a>

{# filter #}
{{ product.price|currency:"£" }}
{{ 100|subtract:product.discount|multiply:0.9 }}

{# inclusion_tag #}
{% product_card product show_price=True %}

{# html tag #}
{% stock_badge product %}
```

---

## 9. Django Admin — Customization & Power Features

### Basic Admin Registration

```python
# products/admin.py
from django.contrib import admin
from .models import Product, Category

@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    # List page configuration
    list_display    = ('name', 'price', 'stock', 'status', 'is_featured', 'created_at')
    list_display_links = ('name',)
    list_editable   = ('price', 'stock', 'is_featured')
    list_filter     = ('status', 'category', 'is_featured', 'created_at')
    search_fields   = ('name', 'description', 'category__name')
    date_hierarchy  = 'created_at'
    ordering        = ('-created_at',)
    list_per_page   = 25
    list_select_related = ('category', 'created_by')  # avoid N+1

    # Detail page configuration
    readonly_fields = ('created_at', 'updated_at', 'slug')
    prepopulated_fields = {'slug': ('name',)}  # auto-fill slug from name in JS

    fieldsets = (
        ('Basic Information', {
            'fields': ('name', 'slug', 'description', 'category')
        }),
        ('Pricing & Stock', {
            'fields': ('price', 'stock'),
        }),
        ('Publishing', {
            'fields': ('status', 'is_featured'),
        }),
        ('Metadata', {
            'fields': ('created_by', 'created_at', 'updated_at'),
            'classes': ('collapse',),  # collapsible section
        }),
    )

    # Actions
    actions = ['publish_products', 'archive_products']

    def publish_products(self, request, queryset):
        updated = queryset.update(status='published')
        self.message_user(request, f'{updated} product(s) published.')
    publish_products.short_description = 'Mark selected as Published'

    def archive_products(self, request, queryset):
        queryset.update(status='archived')
    archive_products.short_description = 'Archive selected products'

    # Custom list_display columns
    def stock_status(self, obj):
        from django.utils.html import format_html
        color = 'green' if obj.stock > 10 else 'orange' if obj.stock > 0 else 'red'
        return format_html('<span style="color: {};">{}</span>', color, obj.stock)
    stock_status.short_description = 'Stock'
    stock_status.admin_order_field = 'stock'  # make the column sortable

    # Restrict queryset based on user
    def get_queryset(self, request):
        qs = super().get_queryset(request)
        if request.user.is_superuser:
            return qs
        return qs.filter(created_by=request.user)

    # Auto-set created_by
    def save_model(self, request, obj, form, change):
        if not change:  # only on creation
            obj.created_by = request.user
        super().save_model(request, obj, form, change)
```

### Inline Admin — Related Objects on Same Page

```python
class OrderItemInline(admin.TabularInline):  # or StackedInline for vertical layout
    model = OrderItem
    fields = ('product', 'quantity', 'unit_price')
    readonly_fields = ('unit_price',)
    extra = 0       # don't show empty forms by default
    min_num = 1
    max_num = 20
    can_delete = True


@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    inlines = [OrderItemInline]
    list_display  = ('id', 'user', 'total_amount', 'status', 'created_at')
    list_filter   = ('status', 'created_at')
    search_fields = ('user__email', 'user__username', 'id')
    raw_id_fields = ('user',)          # replaces dropdown with search widget for large tables
```

### Custom Admin Actions with Intermediate Page

```python
from django.contrib.admin import helpers
from django.shortcuts import render

def export_to_csv(modeladmin, request, queryset):
    import csv
    from django.http import HttpResponse

    response = HttpResponse(content_type='text/csv')
    response['Content-Disposition'] = 'attachment; filename="products.csv"'

    writer = csv.writer(response)
    writer.writerow(['Name', 'Price', 'Stock', 'Status'])
    for obj in queryset.values_list('name', 'price', 'stock', 'status'):
        writer.writerow(obj)
    return response

export_to_csv.short_description = 'Export selected to CSV'
```

### Customizing the Admin Site

```python
# admin.py or apps.py
from django.contrib import admin

admin.site.site_header  = 'My Store Administration'
admin.site.site_title   = 'My Store Admin'
admin.site.index_title  = 'Dashboard'
```

---

## 10. Authentication — Signup, Sign-in & Sign-out

### Custom User Model — Do This First

Always define a custom user model before your first migration. Changing it later is painful.

```python
# users/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    email      = models.EmailField(unique=True)
    bio        = models.TextField(blank=True)
    avatar     = models.ImageField(upload_to='avatars/', blank=True, null=True)
    phone      = models.CharField(max_length=20, blank=True)

    USERNAME_FIELD  = 'email'      # use email for login instead of username
    REQUIRED_FIELDS = ['username'] # required for createsuperuser (besides email)

    def __str__(self):
        return self.email
```

```python
# settings.py
AUTH_USER_MODEL = 'users.User'  # must be set before first migration
```

### Registration View

```python
# users/forms.py
from django.contrib.auth import get_user_model
from django.contrib.auth.forms import UserCreationForm

User = get_user_model()

class RegisterForm(UserCreationForm):
    email = forms.EmailField(required=True)

    class Meta(UserCreationForm.Meta):
        model  = User
        fields = ('email', 'username', 'password1', 'password2')

    def clean_email(self):
        email = self.cleaned_data.get('email')
        if User.objects.filter(email=email).exists():
            raise forms.ValidationError('An account with this email already exists.')
        return email
```

```python
# users/views.py
from django.contrib.auth import login, logout, authenticate
from django.contrib.auth.decorators import login_required
from django.contrib import messages

def register_view(request):
    if request.user.is_authenticated:
        return redirect('home')

    if request.method == 'POST':
        form = RegisterForm(request.POST)
        if form.is_valid():
            user = form.save()
            login(request, user)  # log in immediately after registration
            messages.success(request, 'Account created! Welcome.')
            return redirect('home')
    else:
        form = RegisterForm()

    return render(request, 'users/register.html', {'form': form})


def login_view(request):
    if request.user.is_authenticated:
        return redirect('home')

    if request.method == 'POST':
        email    = request.POST.get('email')
        password = request.POST.get('password')
        user     = authenticate(request, username=email, password=password)

        if user is not None:
            login(request, user)
            # Redirect to 'next' param if present
            next_url = request.GET.get('next') or request.POST.get('next', 'home')
            return redirect(next_url)
        else:
            messages.error(request, 'Invalid email or password.')

    return render(request, 'users/login.html', {
        'next': request.GET.get('next', '')
    })


@login_required
def logout_view(request):
    if request.method == 'POST':  # POST-only logout — protects against CSRF-based logout
        logout(request)
        messages.info(request, 'You have been logged out.')
        return redirect('users:login')
    return render(request, 'users/logout_confirm.html')
```

### Using Django's Built-in Auth Views

```python
# urls.py
from django.contrib.auth import views as auth_views

urlpatterns = [
    path('login/', auth_views.LoginView.as_view(
        template_name='users/login.html',
        redirect_authenticated_user=True
    ), name='login'),
    path('logout/', auth_views.LogoutView.as_view(next_page='/'), name='logout'),
    path('password-change/', auth_views.PasswordChangeView.as_view(
        template_name='users/password_change.html',
        success_url=reverse_lazy('users:password_change_done')
    ), name='password_change'),
    path('password-reset/', auth_views.PasswordResetView.as_view(
        template_name='users/password_reset.html',
        email_template_name='users/password_reset_email.html',
        success_url=reverse_lazy('users:password_reset_done')
    ), name='password_reset'),
]
```

```python
# settings.py
LOGIN_URL          = '/users/login/'
LOGIN_REDIRECT_URL = '/dashboard/'
LOGOUT_REDIRECT_URL = '/users/login/'
```

---

## 11. Image & File Upload

### Settings

```python
# settings.py
import os

MEDIA_URL  = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'
```

```python
# myproject/urls.py
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    ...
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
# Only serve media via Django in development; use S3/Nginx in production
```

### Model with File Fields

```python
# Required: pip install Pillow  (for ImageField)

import os
from django.db import models
from django.utils import timezone

def product_image_path(instance, filename):
    """Generates upload path: products/2024/01/15/filename.jpg"""
    ext  = filename.split('.')[-1].lower()
    name = f"{instance.slug}.{ext}"
    return os.path.join('products', str(timezone.now().year),
                        str(timezone.now().month), name)

class Product(models.Model):
    image    = models.ImageField(upload_to=product_image_path, blank=True, null=True)
    document = models.FileField(upload_to='documents/%Y/%m/', blank=True, null=True)
```

### View Handling

```python
def product_upload(request):
    if request.method == 'POST':
        form = ProductForm(request.POST, request.FILES)  # request.FILES is critical!
        if form.is_valid():
            product = form.save()
            return redirect('products:detail', slug=product.slug)
    else:
        form = ProductForm()
    return render(request, 'products/form.html', {'form': form})
```

```django
{# Template — enctype is required for file uploads #}
<form method="post" enctype="multipart/form-data">
  {% csrf_token %}
  {{ form.as_p }}
  <button type="submit">Upload</button>
</form>
```

### File Validation

```python
from django.core.exceptions import ValidationError

def validate_image_size(image):
    max_size_mb = 5
    if image.size > max_size_mb * 1024 * 1024:
        raise ValidationError(f'Image must be smaller than {max_size_mb} MB.')

def validate_image_type(image):
    import imghdr
    valid_types = ['jpeg', 'png', 'gif', 'webp']
    img_type = imghdr.what(image)
    if img_type not in valid_types:
        raise ValidationError(f'Unsupported image type. Use: {", ".join(valid_types)}')

class Product(models.Model):
    image = models.ImageField(
        upload_to='products/',
        validators=[validate_image_size, validate_image_type],
        blank=True, null=True
    )
```

### Processing Uploads (Thumbnail with Pillow)

```python
from PIL import Image as PILImage

class Product(models.Model):
    image = models.ImageField(upload_to='products/', blank=True, null=True)
    thumbnail = models.ImageField(upload_to='products/thumbnails/', blank=True, null=True)

    def save(self, *args, **kwargs):
        super().save(*args, **kwargs)
        if self.image:
            self._create_thumbnail()

    def _create_thumbnail(self, size=(300, 300)):
        import io
        from django.core.files.base import ContentFile

        img = PILImage.open(self.image.path)
        img.thumbnail(size, PILImage.LANCZOS)

        thumb_io = io.BytesIO()
        fmt = 'JPEG' if img.format != 'PNG' else 'PNG'
        img.save(thumb_io, format=fmt, quality=85)

        thumb_name = f"thumb_{os.path.basename(self.image.name)}"
        self.thumbnail.save(thumb_name, ContentFile(thumb_io.getvalue()), save=False)
        super().save(update_fields=['thumbnail'])
```

### S3 Storage for Production

```bash
pip install boto3 django-storages
```

```python
# settings.py
INSTALLED_APPS += ['storages']

AWS_ACCESS_KEY_ID     = os.environ.get('AWS_ACCESS_KEY_ID')
AWS_SECRET_ACCESS_KEY = os.environ.get('AWS_SECRET_ACCESS_KEY')
AWS_STORAGE_BUCKET_NAME = os.environ.get('AWS_S3_BUCKET')
AWS_S3_REGION_NAME    = 'us-east-1'
AWS_S3_CUSTOM_DOMAIN  = f'{AWS_STORAGE_BUCKET_NAME}.s3.amazonaws.com'
AWS_DEFAULT_ACL       = None  # use bucket-level policy

DEFAULT_FILE_STORAGE  = 'storages.backends.s3boto3.S3Boto3Storage'
MEDIA_URL             = f'https://{AWS_S3_CUSTOM_DOMAIN}/media/'
```

---

## 12. Custom Responses, Exceptions & Mixins

### Custom Exception Handling

```python
# core/exceptions.py
from django.http import JsonResponse
from django.core.exceptions import PermissionDenied
from django.http import Http404

def custom_404(request, exception):
    if request.headers.get('Accept') == 'application/json':
        return JsonResponse({'error': 'Not found', 'status': 404}, status=404)
    return render(request, 'errors/404.html', status=404)

def custom_500(request):
    if request.headers.get('Accept') == 'application/json':
        return JsonResponse({'error': 'Server error', 'status': 500}, status=500)
    return render(request, 'errors/500.html', status=500)

def custom_403(request, exception):
    return render(request, 'errors/403.html', status=403)
```

```python
# myproject/urls.py
handler404 = 'core.exceptions.custom_404'
handler500 = 'core.exceptions.custom_500'
handler403 = 'core.exceptions.custom_403'
```

### Custom Middleware

```python
# core/middleware.py
import time
import logging

logger = logging.getLogger(__name__)

class RequestTimingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        start = time.monotonic()
        response = self.get_response(request)
        duration = time.monotonic() - start
        response['X-Request-Duration'] = f'{duration:.3f}s'
        if duration > 1.0:
            logger.warning(f'Slow request: {request.path} took {duration:.3f}s')
        return response


class MaintenanceModeMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        from django.conf import settings
        if getattr(settings, 'MAINTENANCE_MODE', False):
            if not request.path.startswith('/admin/'):
                return render(request, 'maintenance.html', status=503)
        return self.get_response(request)
```

```python
# settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'core.middleware.RequestTimingMiddleware',  # add custom middleware
    ...
]
```

---

## 13. Multi-Tenancy

### Schema-Based Multi-Tenancy (django-tenants)

```bash
pip install django-tenants
```

Each tenant gets its own PostgreSQL schema. `public` schema holds shared data (tenant list, domains); each tenant schema holds isolated data.

```python
# settings.py
DATABASE_ROUTERS = ['django_tenants.routers.TenantSyncRouter']
TENANT_MODEL     = 'tenants.Tenant'
TENANT_DOMAIN_MODEL = 'tenants.Domain'
SHARED_APPS = [
    'django_tenants',
    'tenants',
    'django.contrib.contenttypes',
    'django.contrib.auth',
    ...
]
TENANT_APPS = [
    'products',
    'orders',
    ...
]
INSTALLED_APPS = list(SHARED_APPS) + [app for app in TENANT_APPS if app not in SHARED_APPS]
```

```python
# tenants/models.py
from django_tenants.models import TenantMixin, DomainMixin

class Tenant(TenantMixin):
    name       = models.CharField(max_length=100)
    created_at = models.DateTimeField(auto_now_add=True)
    auto_create_schema = True  # auto-create schema on save

class Domain(DomainMixin):
    pass
```

```bash
# Migrations
python manage.py migrate_schemas --shared    # migrates public schema
python manage.py create_tenant               # interactive tenant creation
```

### Row-Based Multi-Tenancy (simpler approach)

```python
# models.py — each model has a tenant FK
class Organization(models.Model):
    name = models.CharField(max_length=255)
    slug = models.SlugField(unique=True)

class TenantModel(models.Model):
    organization = models.ForeignKey(Organization, on_delete=models.CASCADE)

    class Meta:
        abstract = True

class Product(TenantModel):
    name = models.CharField(max_length=255)
    # Always filter by organization in views/managers
```

```python
# middleware.py — inject current tenant into request
class TenantMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        if request.user.is_authenticated:
            try:
                request.organization = request.user.profile.organization
            except Exception:
                request.organization = None
        return self.get_response(request)
```

---

## 14. Django Channels — WebSockets & Real-Time

### Setup

```bash
pip install channels channels-redis
```

```python
# settings.py
INSTALLED_APPS += ['channels']

ASGI_APPLICATION = 'myproject.asgi.application'

CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {
            'hosts': [('127.0.0.1', 6379)],
        },
    },
}
```

```python
# myproject/asgi.py
import os
from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.auth import AuthMiddlewareStack
import chat.routing

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings')

application = ProtocolTypeRouter({
    'http': get_asgi_application(),
    'websocket': AuthMiddlewareStack(
        URLRouter(chat.routing.websocket_urlpatterns)
    ),
})
```

### Consumer (WebSocket Handler)

```python
# chat/consumers.py
import json
from channels.generic.websocket import AsyncWebsocketConsumer
from channels.db import database_sync_to_async
from .models import Message, ChatRoom

class ChatConsumer(AsyncWebsocketConsumer):

    async def connect(self):
        self.room_name  = self.scope['url_route']['kwargs']['room_name']
        self.room_group  = f'chat_{self.room_name}'
        self.user        = self.scope['user']

        if not self.user.is_authenticated:
            await self.close()
            return

        # Join room group
        await self.channel_layer.group_add(self.room_group, self.channel_name)
        await self.accept()

        # Send connection message
        await self.channel_layer.group_send(
            self.room_group,
            {'type': 'user_join', 'username': self.user.username}
        )

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(self.room_group, self.channel_name)

    async def receive(self, text_data):
        data    = json.loads(text_data)
        message = data.get('message', '').strip()

        if not message:
            return

        # Save to DB
        await self.save_message(message)

        # Broadcast to group
        await self.channel_layer.group_send(
            self.room_group,
            {
                'type': 'chat_message',
                'message': message,
                'username': self.user.username,
            }
        )

    # Handlers (type field maps to method name, dots → underscores)
    async def chat_message(self, event):
        await self.send(text_data=json.dumps({
            'type': 'message',
            'message': event['message'],
            'username': event['username'],
        }))

    async def user_join(self, event):
        await self.send(text_data=json.dumps({
            'type': 'join',
            'username': event['username'],
        }))

    @database_sync_to_async
    def save_message(self, content):
        room = ChatRoom.objects.get(name=self.room_name)
        return Message.objects.create(room=room, user=self.user, content=content)
```

```python
# chat/routing.py
from django.urls import re_path
from . import consumers

websocket_urlpatterns = [
    re_path(r'ws/chat/(?P<room_name>\w+)/$', consumers.ChatConsumer.as_asgi()),
]
```

> **Tricky:** Django Channels consumers run in async context. Calling ORM methods directly will block the event loop. Always wrap DB calls with `database_sync_to_async`. Similarly, never use `time.sleep()` in an async consumer — use `asyncio.sleep()`.

---

## 15. Django with Celery Beat — Background Tasks

### Setup

```bash
pip install celery redis django-celery-beat
```

```python
# myproject/celery.py
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings')

app = Celery('myproject')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()   # finds tasks.py in all INSTALLED_APPS
```

```python
# myproject/__init__.py
from .celery import app as celery_app
__all__ = ('celery_app',)
```

```python
# settings.py
INSTALLED_APPS += ['django_celery_beat', 'django_celery_results']

CELERY_BROKER_URL      = 'redis://localhost:6379/0'
CELERY_RESULT_BACKEND  = 'django-db'  # store results in DB
CELERY_BEAT_SCHEDULER  = 'django_celery_beat.schedulers:DatabaseScheduler'

CELERY_TASK_SERIALIZER   = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_ACCEPT_CONTENT    = ['json']
CELERY_TIMEZONE          = 'UTC'
```

### Defining Tasks

```python
# products/tasks.py
from celery import shared_task
from celery.utils.log import get_task_logger

logger = get_task_logger(__name__)

@shared_task(bind=True, max_retries=3, default_retry_delay=60)
def send_order_confirmation(self, order_id):
    """Send order confirmation email. Retries on failure."""
    try:
        from orders.models import Order
        from django.core.mail import send_mail

        order = Order.objects.select_related('user').get(pk=order_id)
        send_mail(
            subject=f'Order #{order.id} Confirmed',
            message=f'Your order has been confirmed. Total: {order.total}',
            from_email='noreply@mystore.com',
            recipient_list=[order.user.email],
        )
        logger.info(f'Order confirmation sent for order {order_id}')
    except Exception as exc:
        logger.error(f'Failed to send order confirmation: {exc}')
        raise self.retry(exc=exc)


@shared_task
def generate_daily_report():
    """Runs every night — aggregates sales data."""
    from django.utils import timezone
    from orders.models import Order
    from django.db.models import Sum, Count

    today   = timezone.now().date()
    summary = Order.objects.filter(created_at__date=today).aggregate(
        total_sales=Sum('total_amount'),
        order_count=Count('id'),
    )
    # Save or email the report
    logger.info(f"Daily report: {summary}")
    return summary


@shared_task
def cleanup_old_sessions():
    from django.contrib.sessions.models import Session
    from django.utils import timezone
    deleted, _ = Session.objects.filter(expire_date__lt=timezone.now()).delete()
    logger.info(f'Deleted {deleted} expired sessions')
    return deleted
```

### Calling Tasks

```python
# In views or signals
from products.tasks import send_order_confirmation

# Async (fire and forget)
send_order_confirmation.delay(order.id)

# With countdown (delay in seconds)
send_order_confirmation.apply_async(args=[order.id], countdown=30)

# With eta (specific time)
from datetime import datetime, timedelta
eta = datetime.utcnow() + timedelta(hours=1)
send_order_confirmation.apply_async(args=[order.id], eta=eta)
```

### Celery Beat — Scheduled Tasks

```python
# settings.py — static schedule (alternatively manage via Admin UI with django-celery-beat)
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    'generate-daily-report': {
        'task': 'orders.tasks.generate_daily_report',
        'schedule': crontab(hour=23, minute=59),  # every day at 11:59 PM
    },
    'cleanup-sessions': {
        'task': 'core.tasks.cleanup_old_sessions',
        'schedule': crontab(hour=2, minute=0, day_of_week=0),  # Sunday 2 AM
    },
    'sync-inventory': {
        'task': 'products.tasks.sync_inventory',
        'schedule': 300.0,  # every 5 minutes
    },
}
```

### Running

```bash
# Start worker
celery -A myproject worker --loglevel=info

# Start beat scheduler
celery -A myproject beat --loglevel=info --scheduler django_celery_beat.schedulers:DatabaseScheduler

# Or combined (development only)
celery -A myproject worker --beat --loglevel=info
```

> **Tricky:** Never run `worker --beat` in production — if the worker process dies and restarts, Beat restarts too and can trigger immediate duplicate task runs. Always run worker and beat as separate processes in production.

---

## 16. Django Test Cases

### Unit Tests

```python
# products/tests.py
from django.test import TestCase, Client
from django.urls import reverse
from django.contrib.auth import get_user_model
from .models import Product, Category
from .forms import ProductForm

User = get_user_model()

class ProductModelTest(TestCase):

    @classmethod
    def setUpTestData(cls):
        # Runs once for the class — not rolled back between tests
        # Use for read-only data
        cls.category = Category.objects.create(name='Electronics', slug='electronics')
        cls.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )

    def setUp(self):
        # Runs before each test — rolled back after each
        # Use for data that tests may modify
        self.product = Product.objects.create(
            name='Test Product',
            slug='test-product',
            price=99.99,
            stock=10,
            status='published',
            category=self.category,
            created_by=self.user
        )

    def test_product_str(self):
        self.assertEqual(str(self.product), 'Test Product')

    def test_product_is_in_stock(self):
        self.assertTrue(self.product.is_in_stock)
        self.product.stock = 0
        self.product.save()
        self.assertFalse(self.product.is_in_stock)

    def test_product_absolute_url(self):
        expected = reverse('products:detail', kwargs={'slug': self.product.slug})
        self.assertEqual(self.product.get_absolute_url(), expected)

    def test_slug_auto_generated(self):
        product = Product.objects.create(
            name='Another Product', price=9.99, created_by=self.user
        )
        self.assertEqual(product.slug, 'another-product')
```

### View Tests

```python
class ProductViewTest(TestCase):

    def setUp(self):
        self.client = Client()
        self.user = User.objects.create_user(username='viewtester', password='pass123')
        self.product = Product.objects.create(
            name='Widget', slug='widget', price=9.99, status='published',
            created_by=self.user
        )

    def test_product_list_view(self):
        response = self.client.get(reverse('products:list'))
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, 'products/list.html')
        self.assertContains(response, 'Widget')
        self.assertIn('products', response.context)

    def test_product_create_requires_login(self):
        response = self.client.get(reverse('products:create'))
        self.assertRedirects(response, f"/users/login/?next={reverse('products:create')}")

    def test_product_create_authenticated(self):
        self.client.login(username='viewtester', password='pass123')
        data = {
            'name': 'New Product',
            'price': '49.99',
            'stock': '5',
            'status': 'published',
        }
        response = self.client.post(reverse('products:create'), data)
        self.assertEqual(response.status_code, 302)  # redirect after success
        self.assertTrue(Product.objects.filter(name='New Product').exists())
```

### Form Tests

```python
class ProductFormTest(TestCase):

    def test_valid_form(self):
        data = {'name': 'Widget', 'price': '9.99', 'stock': '5'}
        form = ProductForm(data)
        self.assertTrue(form.is_valid())

    def test_negative_price_invalid(self):
        data = {'name': 'Widget', 'price': '-5', 'stock': '5'}
        form = ProductForm(data)
        self.assertFalse(form.is_valid())
        self.assertIn('price', form.errors)
```

### Testing with Mocks

```python
from unittest.mock import patch, MagicMock

class OrderEmailTest(TestCase):

    @patch('orders.tasks.send_mail')
    def test_order_confirmation_sends_email(self, mock_send_mail):
        order = Order.objects.create(user=self.user, total_amount=99.99)
        send_order_confirmation(order.id)
        mock_send_mail.assert_called_once()
        call_args = mock_send_mail.call_args
        self.assertIn(str(order.id), call_args.kwargs['subject'])

    @patch('products.views.ProductForm')
    def test_form_error_shown(self, MockForm):
        mock_form = MagicMock()
        mock_form.is_valid.return_value = False
        MockForm.return_value = mock_form
        ...
```

### Running Tests

```bash
python manage.py test                           # all tests
python manage.py test products                  # single app
python manage.py test products.tests.ProductModelTest  # single class
python manage.py test products.tests.ProductModelTest.test_str  # single test
python manage.py test --verbosity=2 --keepdb   # reuse test DB for speed
```

---

## 17. Deployment — AWS Linux & Heroku

### AWS EC2 + Nginx + Gunicorn

```bash
# 1. Server setup (Ubuntu 22.04)
sudo apt update && sudo apt upgrade -y
sudo apt install python3-pip python3-venv nginx postgresql postgresql-contrib -y

# 2. Create project user
sudo adduser django
sudo usermod -aG www-data django

# 3. Clone project
sudo mkdir -p /var/www/myproject
sudo chown django:django /var/www/myproject
cd /var/www/myproject
git clone https://github.com/youruser/myproject.git .

# 4. Virtual environment
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
pip install gunicorn

# 5. Environment variables
cp .env.example .env
nano .env   # set SECRET_KEY, DATABASE_URL, etc.
```

```ini
; /etc/systemd/system/gunicorn.service
[Unit]
Description=Gunicorn daemon for Django
After=network.target

[Service]
User=django
Group=www-data
WorkingDirectory=/var/www/myproject
EnvironmentFile=/var/www/myproject/.env
ExecStart=/var/www/myproject/venv/bin/gunicorn \
    --access-logfile - \
    --workers 3 \
    --bind unix:/run/gunicorn.sock \
    myproject.wsgi:application

[Install]
WantedBy=multi-user.target
```

```nginx
# /etc/nginx/sites-available/myproject
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    location = /favicon.ico { access_log off; log_not_found off; }

    location /static/ {
        root /var/www/myproject;
    }

    location /media/ {
        root /var/www/myproject;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
    }
}
```

```bash
# Enable and start
sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled/
sudo systemctl daemon-reload
sudo systemctl start gunicorn
sudo systemctl enable gunicorn
sudo systemctl restart nginx

# SSL with Certbot
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

### Heroku Deployment

```
# Procfile
web: gunicorn myproject.wsgi --log-file -
worker: celery -A myproject worker --loglevel=info
beat: celery -A myproject beat --loglevel=info
```

```bash
pip install gunicorn whitenoise dj-database-url
```

```python
# settings.py (production additions)
import dj_database_url

DEBUG = os.environ.get('DEBUG', 'False') == 'True'
ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', 'localhost').split(',')

SECRET_KEY = os.environ['SECRET_KEY']

DATABASES = {
    'default': dj_database_url.config(conn_max_age=600)
}

# Whitenoise for static files
MIDDLEWARE.insert(1, 'whitenoise.middleware.WhiteNoiseMiddleware')
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
STATIC_ROOT = BASE_DIR / 'staticfiles'
```

```bash
# Deploy
heroku create myapp
heroku addons:create heroku-postgresql:hobby-dev
heroku addons:create heroku-redis:hobby-dev
heroku config:set SECRET_KEY='your-secret-key'
heroku config:set DEBUG=False
git push heroku main
heroku run python manage.py migrate
heroku run python manage.py createsuperuser
```

---
