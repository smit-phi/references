# Django Class-Based Views: Complete Guide

## Table of Contents
1. [Fundamentals of CBVs](#fundamentals)
2. [View Hierarchy](#view-hierarchy)
3. [Generic Views](#generic-views)
4. [Built-in Mixins](#built-in-mixins)
5. [Custom Mixins](#custom-mixins)
6. [Custom Response Handling](#custom-responses)
7. [Exception Handling](#exception-handling)
8. [Real-World Examples](#real-world-examples)

---

## <a name="fundamentals"></a>1. Fundamentals of Class-Based Views

### What are CBVs?

Class-based views are Python classes instead of functions. They provide an object-oriented approach to handling HTTP requests.

### Basic Structure

```python
# models.py
from django.db import models

class BlogPost(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)
    is_published = models.BooleanField(default=False)
    
    def __str__(self):
        return self.title

# views.py
from django.views import View
from django.http import HttpResponse
from django.views.decorators.http import require_http_methods

class BasicBlogView(View):
    """Basic class-based view"""
    
    def get(self, request):
        return HttpResponse("This is a GET request")
    
    def post(self, request):
        return HttpResponse("This is a POST request")
    
    def put(self, request):
        return HttpResponse("This is a PUT request")

# urls.py
from django.urls import path
from . import views

urlpatterns = [
    path('basic/', views.BasicBlogView.as_view(), name='basic_blog'),
]
```

### Key Concept: as_view()

```python
# as_view() is a class method that returns a view function
# It instantiates the view class and dispatches requests to appropriate methods

class MyView(View):
    def get(self, request):
        return HttpResponse('GET')
    
    def post(self, request):
        return HttpResponse('POST')

# In urls.py - as_view() converts the class to a callable view
urlpatterns = [
    path('my-view/', MyView.as_view()),  # as_view() is crucial!
]
```

### Dispatching HTTP Methods

```python
from django.views import View
from django.http import JsonResponse

class MethodDispatchView(View):
    """Demonstrates how CBVs dispatch based on HTTP method"""
    
    def dispatch(self, request, *args, **kwargs):
        """
        The dispatch method is called first.
        It routes to appropriate HTTP method handlers.
        """
        print(f"Dispatch called for {request.method}")
        return super().dispatch(request, *args, **kwargs)
    
    def get(self, request):
        return JsonResponse({'method': 'GET', 'data': 'List all items'})
    
    def post(self, request):
        return JsonResponse({'method': 'POST', 'data': 'Create new item'})
    
    def put(self, request, pk):
        return JsonResponse({'method': 'PUT', 'pk': pk, 'data': 'Update item'})
    
    def delete(self, request, pk):
        return JsonResponse({'method': 'DELETE', 'pk': pk})

# urls.py
urlpatterns = [
    path('api/<int:pk>/', MethodDispatchView.as_view()),
]
```

---

## <a name="view-hierarchy"></a>2. View Hierarchy

```
View (Base Class)
├── RedirectView
├── TemplateView
│   └── TemplateResponseMixin
├── ListView
├── DetailView
├── FormView
│   ├── CreateView
│   ├── UpdateView
│   └── DeleteView
└── [Custom Views]
```

### View Inheritance Flow

```python
from django.views.generic import View

# Level 1: Base View class
class View:
    def as_view(cls, **initkwargs):
        """Entry point for URL patterns"""
        pass
    
    def dispatch(self, request, *args, **kwargs):
        """Routes HTTP methods to handlers"""
        pass

# Level 2: Mixin-based views
from django.views.generic.base import TemplateResponseMixin, ContextMixin

class TemplateView(TemplateResponseMixin, ContextMixin, View):
    """Adds template rendering capability"""
    template_name = None
    
    def get_context_data(self, **kwargs):
        """Provides context to template"""
        pass

# Level 3: Model-based generic views
from django.views.generic import ListView

class ListView(MultipleObjectMixin, TemplateResponseMixin, View):
    """Lists model instances with pagination"""
    model = None
    paginate_by = None
    
    def get_queryset(self):
        """Returns queryset of objects"""
        pass
```

---

## <a name="generic-views"></a>3. Generic Views

### 3.1 TemplateView

```python
from django.views.generic import TemplateView
from django.utils.decorators import method_decorator
from django.contrib.auth.decorators import login_required

class HomePageView(TemplateView):
    """Simple template rendering without model interaction"""
    template_name = 'home.html'
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['site_name'] = 'My Blog'
        context['year'] = 2024
        return context

# urls.py
urlpatterns = [
    path('', HomePageView.as_view(), name='home'),
]

# template: home.html
"""
<h1>{{ site_name }}</h1>
<p>Year: {{ year }}</p>
"""
```

### 3.2 ListView

```python
from django.views.generic import ListView
from django.paginate_queryset import Paginator

class BlogPostListView(ListView):
    """Lists all blog posts with pagination"""
    model = BlogPost
    template_name = 'blog/post_list.html'
    context_object_name = 'posts'  # Default is 'blogpost_list'
    paginate_by = 10
    ordering = ['-created_at']  # Newest first
    
    def get_queryset(self):
        """Override to filter queryset"""
        queryset = super().get_queryset()
        # Only show published posts
        queryset = queryset.filter(is_published=True)
        
        # Filter by search query
        search = self.request.GET.get('search')
        if search:
            queryset = queryset.filter(
                title__icontains=search
            ) | queryset.filter(
                content__icontains=search
            )
        
        return queryset
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['search_query'] = self.request.GET.get('search', '')
        context['total_posts'] = self.get_queryset().count()
        return context

# urls.py
urlpatterns = [
    path('posts/', BlogPostListView.as_view(), name='post_list'),
]

# template: blog/post_list.html
"""
<h1>Blog Posts</h1>

<form method="get">
    <input type="text" name="search" value="{{ search_query }}">
    <button type="submit">Search</button>
</form>

<ul>
    {% for post in posts %}
        <li>{{ post.title }} - {{ post.created_at }}</li>
    {% endfor %}
</ul>

<!-- Pagination -->
<div class="pagination">
    {% if page_obj.has_previous %}
        <a href="?page=1">First</a>
        <a href="?page={{ page_obj.previous_page_number }}">Previous</a>
    {% endif %}
    
    <span>Page {{ page_obj.number }} of {{ page_obj.paginator.num_pages }}</span>
    
    {% if page_obj.has_next %}
        <a href="?page={{ page_obj.next_page_number }}">Next</a>
        <a href="?page={{ page_obj.paginator.num_pages }}">Last</a>
    {% endif %}
</div>
"""
```

### 3.3 DetailView

```python
from django.views.generic import DetailView
from django.shortcuts import get_object_or_404
from django.http import Http404

class BlogPostDetailView(DetailView):
    """Display single blog post with related data"""
    model = BlogPost
    template_name = 'blog/post_detail.html'
    context_object_name = 'post'
    slug_field = 'slug'  # URL parameter
    slug_url_kwarg = 'slug'
    
    def get_object(self, queryset=None):
        """
        Override to add custom logic for retrieving object.
        Raises Http404 if not found.
        """
        queryset = queryset or self.get_queryset()
        
        slug = self.kwargs.get(self.slug_url_kwarg)
        queryset = queryset.filter(slug=slug)
        
        try:
            obj = queryset.get()
        except queryset.model.DoesNotExist:
            raise Http404(f"Post with slug '{slug}' not found")
        
        return obj
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        post = self.object
        # Add related comments
        context['comments'] = post.comments.filter(is_approved=True)
        # Add next/previous posts
        context['next_post'] = BlogPost.objects.filter(
            created_at__gt=post.created_at,
            is_published=True
        ).first()
        context['previous_post'] = BlogPost.objects.filter(
            created_at__lt=post.created_at,
            is_published=True
        ).last()
        return context

# urls.py
urlpatterns = [
    path('posts/<slug:slug>/', BlogPostDetailView.as_view(), name='post_detail'),
]

# template: blog/post_detail.html
"""
<article>
    <h1>{{ post.title }}</h1>
    <p>By {{ post.author }} on {{ post.created_at }}</p>
    <div>{{ post.content }}</div>
    
    <h3>Comments ({{ comments|length }})</h3>
    {% for comment in comments %}
        <div class="comment">
            <strong>{{ comment.author }}</strong>
            <p>{{ comment.text }}</p>
        </div>
    {% endfor %}
    
    <nav>
        {% if previous_post %}
            <a href="{{ previous_post.get_absolute_url }}">← {{ previous_post.title }}</a>
        {% endif %}
        {% if next_post %}
            <a href="{{ next_post.get_absolute_url }}">{{ next_post.title }} →</a>
        {% endif %}
    </nav>
</article>
"""
```

### 3.4 CreateView

```python
from django.views.generic import CreateView
from django.urls import reverse_lazy
from django.contrib.auth.mixins import LoginRequiredMixin
from django import forms

class BlogPostForm(forms.ModelForm):
    class Meta:
        model = BlogPost
        fields = ['title', 'content']
        widgets = {
            'title': forms.TextInput(attrs={'class': 'form-control'}),
            'content': forms.Textarea(attrs={'class': 'form-control'}),
        }

class BlogPostCreateView(LoginRequiredMixin, CreateView):
    """Create a new blog post (author only)"""
    model = BlogPost
    form_class = BlogPostForm
    template_name = 'blog/post_form.html'
    success_url = reverse_lazy('post_list')
    
    def form_valid(self, form):
        """
        Called when form is valid.
        Set the current user as author before saving.
        """
        form.instance.author = self.request.user
        response = super().form_valid(form)
        # You can add messages or logging here
        return response
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['action'] = 'Create'
        return context

# urls.py
urlpatterns = [
    path('posts/create/', BlogPostCreateView.as_view(), name='post_create'),
]
```

### 3.5 UpdateView

```python
from django.views.generic import UpdateView
from django.core.exceptions import PermissionDenied

class BlogPostUpdateView(LoginRequiredMixin, UpdateView):
    """Update existing blog post (author only)"""
    model = BlogPost
    form_class = BlogPostForm
    template_name = 'blog/post_form.html'
    slug_field = 'slug'
    slug_url_kwarg = 'slug'
    
    def dispatch(self, request, *args, **kwargs):
        """Check permissions before processing request"""
        self.object = self.get_object()
        
        if self.object.author != request.user:
            raise PermissionDenied("You can only edit your own posts")
        
        return super().dispatch(request, *args, **kwargs)
    
    def form_valid(self, form):
        """Optional: add timestamp for last edit"""
        response = super().form_valid(form)
        # Log the update
        print(f"Post updated: {self.object.title}")
        return response
    
    def get_success_url(self):
        """Redirect to the updated post"""
        return self.object.get_absolute_url()
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['action'] = 'Edit'
        return context

# urls.py
urlpatterns = [
    path('posts/<slug:slug>/edit/', BlogPostUpdateView.as_view(), name='post_update'),
]
```

### 3.6 DeleteView

```python
from django.views.generic import DeleteView
from django.contrib import messages

class BlogPostDeleteView(LoginRequiredMixin, DeleteView):
    """Delete blog post (author only)"""
    model = BlogPost
    template_name = 'blog/post_confirm_delete.html'
    success_url = reverse_lazy('post_list')
    slug_field = 'slug'
    slug_url_kwarg = 'slug'
    
    def dispatch(self, request, *args, **kwargs):
        """Verify ownership before deletion"""
        self.object = self.get_object()
        
        if self.object.author != request.user:
            raise PermissionDenied("You can only delete your own posts")
        
        return super().dispatch(request, *args, **kwargs)
    
    def delete(self, request, *args, **kwargs):
        """Log deletion and show message"""
        post_title = self.object.title
        response = super().delete(request, *args, **kwargs)
        messages.success(request, f"Post '{post_title}' deleted successfully")
        return response

# template: blog/post_confirm_delete.html
"""
<div class="alert alert-danger">
    <h2>Delete Post?</h2>
    <p>Are you sure you want to delete "{{ object.title }}"?</p>
    <p>This action cannot be undone.</p>
    
    <form method="post">
        {% csrf_token %}
        <button type="submit" class="btn btn-danger">Delete</button>
        <a href="{{ object.get_absolute_url }}" class="btn btn-secondary">Cancel</a>
    </form>
</div>
"""

# urls.py
urlpatterns = [
    path('posts/<slug:slug>/delete/', BlogPostDeleteView.as_view(), name='post_delete'),
]
```

---

## <a name="built-in-mixins"></a>4. Built-in Mixins

### 4.1 Authentication Mixins

```python
from django.contrib.auth.mixins import (
    LoginRequiredMixin,
    UserPassesTestMixin,
    PermissionRequiredMixin
)
from django.core.exceptions import PermissionDenied
from django.shortcuts import redirect

# Basic Login Required
class ProtectedView(LoginRequiredMixin, ListView):
    """Requires user to be logged in"""
    model = BlogPost
    login_url = 'login'  # Redirect URL if not logged in
    # or
    # login_url = '/accounts/login/'
    
    # Optional: redirect to next page after login
    redirect_field_name = 'next'

# Example: POST request handling
class ProtectedPostView(LoginRequiredMixin, CreateView):
    model = BlogPost
    fields = ['title', 'content']
    
    # If LoginRequiredMixin fails for POST, returns 403 instead of redirect
    # Override this behavior:
    def post(self, request, *args, **kwargs):
        if not request.user.is_authenticated:
            return redirect('login')
        return super().post(request, *args, **kwargs)
```

### 4.2 Permission and Test Mixins

```python
from django.contrib.auth.mixins import (
    PermissionRequiredMixin,
    UserPassesTestMixin
)

# Check specific permission
class AdminOnlyView(PermissionRequiredMixin, ListView):
    """Requires specific permission"""
    model = BlogPost
    permission_required = 'blog.change_blogpost'  # Single permission
    # permission_required = ['blog.view_blogpost', 'blog.add_blogpost']  # Multiple
    
    # Customize error behavior
    raise_exception = True  # Raises PermissionDenied (403)
    # or
    permission_denied_message = "You don't have access to this page"

# Custom test
class AuthorOnlyView(UserPassesTestMixin, UpdateView):
    """Only the post author can edit"""
    model = BlogPost
    fields = ['title', 'content']
    
    def test_func(self):
        """Return True if user passes test, False otherwise"""
        post = self.get_object()
        return post.author == self.request.user
    
    def handle_no_permission(self):
        """Called when test_func returns False"""
        raise PermissionDenied("You can only edit your own posts")

# Multiple conditions
class ModeratorView(UserPassesTestMixin, DeleteView):
    """Multiple test conditions"""
    model = BlogPost
    
    def test_func(self):
        user = self.request.user
        return user.is_staff or user.groups.filter(name='Moderator').exists()
```

### 4.3 Response Mixin

```python
from django.views.generic import View
from django.http import JsonResponse
from django.views.generic.base import View

# JSON Response Mixin
class JSONResponseMixin:
    """Allows returning JSON responses"""
    
    def render_to_json_response(self, context, **response_kwargs):
        return JsonResponse(context, **response_kwargs)

class APIListView(JSONResponseMixin, ListView):
    """API endpoint that returns JSON"""
    model = BlogPost
    
    def get(self, request, *args, **kwargs):
        self.object_list = self.get_queryset()
        data = {
            'count': self.object_list.count(),
            'results': [
                {
                    'id': post.id,
                    'title': post.title,
                    'author': str(post.author),
                    'created_at': post.created_at.isoformat(),
                }
                for post in self.object_list
            ]
        }
        return self.render_to_json_response(data)
```

### 4.4 Context Mixin

```python
from django.views.generic.base import ContextMixin

class NavigationMixin(ContextMixin):
    """Adds navigation data to context"""
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['navigation'] = {
            'home': '/',
            'about': '/about/',
            'contact': '/contact/',
        }
        return context

class BlogPostListWithNav(NavigationMixin, ListView):
    model = BlogPost
    # Now all posts will have 'navigation' in context
```

### 4.5 SingleObjectMixin

```python
from django.views.generic.detail import SingleObjectMixin

class MultiObjectMixin:
    """Already used in ListView, DetailView"""
    pass

# Example of custom mixin combining single and list
class PostWithCommentsView(SingleObjectMixin, ListView):
    """Show post AND its comments"""
    paginate_by = 10
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['post'] = self.get_object(
            self.get_queryset()
        )
        return context
    
    def get_queryset(self):
        # Get comments for this post
        post = self.get_object(
            BlogPost.objects.all()
        )
        return post.comments.all()
```

---

## <a name="custom-mixins"></a>5. Custom Mixins

### 5.1 Timestamp Mixin

```python
from django.utils import timezone

class TimestampMixin:
    """Automatically update timestamps on save"""
    
    def form_valid(self, form):
        response = super().form_valid(form)
        print(f"Object saved at {timezone.now()}")
        return response

class BlogPostCreateWithTimestamp(TimestampMixin, CreateView):
    model = BlogPost
    fields = ['title', 'content']
```

### 5.2 Slugify Mixin

```python
from django.utils.text import slugify

class SlugifyMixin:
    """Auto-generate slug from title"""
    
    def form_valid(self, form):
        if not form.instance.slug:
            form.instance.slug = slugify(form.instance.title)
        return super().form_valid(form)

class BlogPostCreateWithSlug(SlugifyMixin, CreateView):
    model = BlogPost
    fields = ['title', 'content']
```

### 5.3 Audit Trail Mixin

```python
from django.contrib.auth.mixins import LoginRequiredMixin
import logging

logger = logging.getLogger(__name__)

class AuditTrailMixin:
    """Log all modifications"""
    
    def dispatch(self, request, *args, **kwargs):
        response = super().dispatch(request, *args, **kwargs)
        
        logger.info(
            f"User {request.user} accessed {self.__class__.__name__} "
            f"via {request.method} at {timezone.now()}"
        )
        
        return response
    
    def form_valid(self, form):
        response = super().form_valid(form)
        
        logger.info(
            f"User {self.request.user} modified {self.model.__name__}: "
            f"{form.instance.id}"
        )
        
        return response

class AuditedBlogPostUpdate(LoginRequiredMixin, AuditTrailMixin, UpdateView):
    model = BlogPost
    fields = ['title', 'content']
```

### 5.4 Cache Mixin

```python
from django.views.decorators.cache import cache_page
from django.utils.decorators import method_decorator
from django.core.cache import cache

class CacheMixin:
    """Cache view results"""
    cache_timeout = 300  # 5 minutes
    
    @method_decorator(cache_page(cache_timeout))
    def dispatch(self, *args, **kwargs):
        return super().dispatch(*args, **kwargs)

class CachedBlogListView(CacheMixin, ListView):
    model = BlogPost
    cache_timeout = 600  # 10 minutes

# Or without decorator:
class CustomCacheMixin:
    """Custom cache implementation"""
    cache_key_prefix = 'view'
    cache_timeout = 300
    
    def get_cache_key(self):
        return f"{self.cache_key_prefix}:{self.request.path}"
    
    def get(self, request, *args, **kwargs):
        cache_key = self.get_cache_key()
        response = cache.get(cache_key)
        
        if response is None:
            response = super().get(request, *args, **kwargs)
            cache.set(cache_key, response, self.cache_timeout)
        
        return response
```

### 5.5 Pagination Mixin

```python
class PaginationMixin:
    """Enhanced pagination with page size options"""
    default_paginate_by = 10
    
    def get_paginate_by(self, queryset):
        """Allow user to specify page size via GET param"""
        page_size = self.request.GET.get('page_size')
        if page_size and page_size.isdigit():
            page_size = int(page_size)
            if page_size <= 100:  # Max 100 items
                return page_size
        return self.default_paginate_by
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['page_size_options'] = [10, 25, 50, 100]
        return context

class FlexiblePaginationListView(PaginationMixin, ListView):
    model = BlogPost
    default_paginate_by = 15
```

### 5.6 Search Mixin

```python
from django.db.models import Q

class SearchMixin:
    """Add search functionality to list views"""
    search_fields = []  # Override in subclass
    search_param = 'q'
    
    def get_queryset(self):
        queryset = super().get_queryset()
        search_query = self.request.GET.get(self.search_param)
        
        if search_query and self.search_fields:
            q_objects = Q()
            for field in self.search_fields:
                q_objects |= Q(**{f'{field}__icontains': search_query})
            queryset = queryset.filter(q_objects)
        
        return queryset
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['search_query'] = self.request.GET.get(self.search_param, '')
        return context

class SearchableBlogListView(SearchMixin, ListView):
    model = BlogPost
    search_fields = ['title', 'content', 'author__username']
```

### 5.7 Filter Mixin

```python
class FilterMixin:
    """Add filtering to list views"""
    filter_fields = {}  # {'field_name': 'param_name'}
    
    def get_queryset(self):
        queryset = super().get_queryset()
        
        for field, param in self.filter_fields.items():
            value = self.request.GET.get(param)
            if value:
                queryset = queryset.filter(**{field: value})
        
        return queryset
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['filters'] = {}
        for field, param in self.filter_fields.items():
            context['filters'][param] = self.request.GET.get(param, '')
        return context

class FilteredBlogListView(FilterMixin, ListView):
    model = BlogPost
    filter_fields = {
        'is_published': 'published',
        'author__id': 'author_id'
    }
```

### 5.8 Owner Check Mixin

```python
from django.core.exceptions import PermissionDenied

class OwnerCheckMixin:
    """Verify user owns the object"""
    owner_field = 'author'  # Override if needed
    
    def dispatch(self, request, *args, **kwargs):
        self.object = self.get_object()
        
        owner = getattr(self.object, self.owner_field)
        if owner != request.user:
            raise PermissionDenied(
                "You don't have permission to access this resource"
            )
        
        return super().dispatch(request, *args, **kwargs)

class OwnerOnlyBlogUpdateView(LoginRequiredMixin, OwnerCheckMixin, UpdateView):
    model = BlogPost
    fields = ['title', 'content']
    owner_field = 'author'
```

---

## <a name="custom-responses"></a>6. Custom Response Handling

### 6.1 JSON Response

```python
from django.http import JsonResponse
from django.views.generic import DetailView
from django.core import serializers
import json

class BlogPostJSONView(DetailView):
    """Return post data as JSON"""
    model = BlogPost
    
    def render_to_response(self, context, **response_kwargs):
        post = self.object
        data = {
            'id': post.id,
            'title': post.title,
            'content': post.content,
            'author': {
                'id': post.author.id,
                'username': post.author.username,
                'email': post.author.email,
            },
            'created_at': post.created_at.isoformat(),
            'updated_at': post.updated_at.isoformat() if hasattr(post, 'updated_at') else None,
            'is_published': post.is_published,
        }
        return JsonResponse(data)

# urls.py
urlpatterns = [
    path('api/posts/<int:pk>/', BlogPostJSONView.as_view(), name='post_json'),
]
```

### 6.2 CSV Response

```python
import csv
from django.http import HttpResponse
from django.views.generic import ListView

class BlogPostCSVView(ListView):
    """Export posts as CSV"""
    model = BlogPost
    
    def render_to_response(self, context, **response_kwargs):
        response = HttpResponse(content_type='text/csv')
        response['Content-Disposition'] = 'attachment; filename="posts.csv"'
        
        writer = csv.writer(response)
        writer.writerow(['ID', 'Title', 'Author', 'Created At'])
        
        for post in self.get_queryset():
            writer.writerow([
                post.id,
                post.title,
                post.author.username,
                post.created_at.strftime('%Y-%m-%d %H:%M'),
            ])
        
        return response

# urls.py
urlpatterns = [
    path('posts/export/csv/', BlogPostCSVView.as_view(), name='posts_csv'),
]
```

### 6.3 PDF Response

```python
from reportlab.lib.pagesizes import letter
from reportlab.pdfgen import canvas
from django.http import HttpResponse
from django.views.generic import DetailView

class BlogPostPDFView(DetailView):
    """Export post as PDF"""
    model = BlogPost
    
    def render_to_response(self, context, **response_kwargs):
        post = self.object
        
        response = HttpResponse(content_type='application/pdf')
        response['Content-Disposition'] = f'attachment; filename="post_{post.id}.pdf"'
        
        # Create PDF
        p = canvas.Canvas(response, pagesize=letter)
        width, height = letter
        
        # Title
        p.setFont("Helvetica-Bold", 16)
        p.drawString(50, height - 50, post.title)
        
        # Author and date
        p.setFont("Helvetica", 10)
        p.drawString(50, height - 70, f"By {post.author} on {post.created_at}")
        
        # Content
        p.setFont("Helvetica", 11)
        y = height - 100
        for line in post.content.split('\n'):
            if y < 50:
                p.showPage()
                y = height - 50
            p.drawString(50, y, line[:80])  # Wrap text
            y -= 20
        
        p.save()
        return response

# urls.py
urlpatterns = [
    path('posts/<int:pk>/pdf/', BlogPostPDFView.as_view(), name='post_pdf'),
]
```

### 6.4 Redirect Response

```python
from django.shortcuts import redirect
from django.views.generic import View

class RedirectToBlogView(View):
    """Redirect to blog"""
    
    def get(self, request):
        return redirect('post_list')  # Redirect to named URL

class SmartRedirectView(View):
    """Conditional redirect"""
    
    def get(self, request):
        if request.user.is_authenticated:
            return redirect('dashboard')
        else:
            return redirect('login')
```

### 6.5 File Download Response

```python
from django.http import FileResponse
from django.views.generic import DetailView

class BlogPostDownloadView(DetailView):
    """Download post as text file"""
    model = BlogPost
    
    def render_to_response(self, context, **response_kwargs):
        post = self.object
        content = f"Title: {post.title}\nAuthor: {post.author}\n\n{post.content}"
        
        response = HttpResponse(content, content_type='text/plain')
        response['Content-Disposition'] = f'attachment; filename="post_{post.id}.txt"'
        
        return response
```

---

## <a name="exception-handling"></a>7. Exception Handling

### 7.1 Built-in Exceptions

```python
from django.http import Http404, HttpResponseForbidden
from django.core.exceptions import PermissionDenied
from django.views.generic import DetailView

class SafeBlogDetailView(DetailView):
    """Proper exception handling"""
    model = BlogPost
    
    def get_object(self, queryset=None):
        """Raises Http404 if object not found"""
        queryset = queryset or self.get_queryset()
        pk = self.kwargs.get('pk')
        
        try:
            obj = queryset.get(pk=pk)
            return obj
        except queryset.model.DoesNotExist:
            raise Http404(f"Blog post with ID {pk} does not exist")
        except queryset.model.MultipleObjectsReturned:
            # Rare but handle it
            return queryset.first()

# Exception during update
class SafeBlogUpdateView(UpdateView):
    model = BlogPost
    fields = ['title', 'content']
    
    def form_valid(self, form):
        try:
            return super().form_valid(form)
        except Exception as e:
            # Log the error
            import logging
            logger = logging.getLogger(__name__)
            logger.error(f"Error updating post: {str(e)}")
            
            # Re-raise with custom message
            form.add_error(None, "An error occurred. Please try again.")
            return self.form_invalid(form)
```

### 7.2 Custom Exception Handling

```python
class PostNotPublishedException(Exception):
    """Custom exception for unpublished posts"""
    pass

class PublishedPostOnlyView(DetailView):
    """Only show published posts"""
    model = BlogPost
    
    def get_object(self, queryset=None):
        obj = super().get_object(queryset)
        
        if not obj.is_published:
            raise Http404("This post is not published yet")
        
        return obj

class StrictBlogDetailView(DetailView):
    """Raise custom exceptions"""
    model = BlogPost
    
    def get_object(self, queryset=None):
        obj = super().get_object(queryset)
        
        if not obj.is_published:
            raise PostNotPublishedException("Post is not published")
        
        return obj
    
    def get(self, request, *args, **kwargs):
        try:
            self.object = self.get_object()
        except PostNotPublishedException:
            return HttpResponse("This post is not available yet", status=403)
        except Http404:
            return HttpResponse("Post not found", status=404)
        
        context = self.get_context_data(object=self.object)
        return self.render_to_response(context)
```

### 7.3 Global Exception Handler

```python
# views.py
from django.views.generic import View
from django.http import JsonResponse

class ExceptionHandlingMixin:
    """Centralized exception handling"""
    
    def dispatch(self, request, *args, **kwargs):
        try:
            return super().dispatch(request, *args, **kwargs)
        except Http404 as e:
            return self.handle_404(e)
        except PermissionDenied as e:
            return self.handle_403(e)
        except ValueError as e:
            return self.handle_400(e)
        except Exception as e:
            return self.handle_500(e)
    
    def handle_404(self, exception):
        return JsonResponse({'error': str(exception)}, status=404)
    
    def handle_403(self, exception):
        return JsonResponse({'error': str(exception)}, status=403)
    
    def handle_400(self, exception):
        return JsonResponse({'error': str(exception)}, status=400)
    
    def handle_500(self, exception):
        import logging
        logger = logging.getLogger(__name__)
        logger.error(f"Unhandled exception: {str(exception)}", exc_info=True)
        return JsonResponse({'error': 'Internal server error'}, status=500)

class SafeAPIView(ExceptionHandlingMixin, ListView):
    model = BlogPost
```

---

## <a name="real-world-examples"></a>8. Real-World Examples

### 8.1 Complete Blog System

```python
# models.py
from django.db import models
from django.contrib.auth.models import User
from django.utils.text import slugify

class BlogPost(models.Model):
    STATUS_CHOICES = [
        ('draft', 'Draft'),
        ('published', 'Published'),
        ('archived', 'Archived'),
    ]
    
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    content = models.TextField()
    excerpt = models.TextField(max_length=500, blank=True)
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name='posts')
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='draft')
    featured_image = models.ImageField(upload_to='posts/', blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    published_at = models.DateTimeField(null=True, blank=True)
    view_count = models.IntegerField(default=0)
    
    class Meta:
        ordering = ['-published_at', '-created_at']
        indexes = [
            models.Index(fields=['slug']),
            models.Index(fields=['author', 'status']),
        ]
    
    def __str__(self):
        return self.title
    
    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)
    
    def get_absolute_url(self):
        from django.urls import reverse
        return reverse('post_detail', kwargs={'slug': self.slug})

# views.py
from django.views.generic import ListView, DetailView, CreateView, UpdateView, DeleteView
from django.contrib.auth.mixins import LoginRequiredMixin, UserPassesTestMixin
from django.shortcuts import redirect
from django.utils import timezone
from django.db.models import Q
from django.http import JsonResponse
from django.core.paginator import Paginator
import logging

logger = logging.getLogger(__name__)

class BlogListView(ListView):
    """List published blog posts with search and filter"""
    model = BlogPost
    template_name = 'blog/post_list.html'
    context_object_name = 'posts'
    paginate_by = 10
    
    def get_queryset(self):
        queryset = BlogPost.objects.filter(status='published').select_related('author')
        
        # Search
        search = self.request.GET.get('search', '').strip()
        if search:
            queryset = queryset.filter(
                Q(title__icontains=search) |
                Q(content__icontains=search) |
                Q(excerpt__icontains=search)
            )
        
        # Filter by author
        author_id = self.request.GET.get('author')
        if author_id:
            queryset = queryset.filter(author_id=author_id)
        
        return queryset.order_by('-published_at')
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['search_query'] = self.request.GET.get('search', '')
        context['authors'] = User.objects.filter(posts__status='published').distinct()
        return context

class BlogDetailView(DetailView):
    """Show published blog post"""
    model = BlogPost
    template_name = 'blog/post_detail.html'
    context_object_name = 'post'
    slug_field = 'slug'
    slug_url_kwarg = 'slug'
    
    def get_queryset(self):
        return BlogPost.objects.filter(status='published')
    
    def get(self, request, *args, **kwargs):
        self.object = self.get_object()
        self.object.view_count += 1
        self.object.save(update_fields=['view_count'])
        return super().get(request, *args, **kwargs)
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        post = self.object
        
        # Related posts
        context['related_posts'] = BlogPost.objects.filter(
            status='published'
        ).exclude(id=post.id).order_by('-published_at')[:3]
        
        # Author's other posts
        context['author_posts'] = post.author.posts.filter(
            status='published'
        ).exclude(id=post.id)[:5]
        
        return context

class BlogCreateView(LoginRequiredMixin, CreateView):
    """Create new blog post"""
    model = BlogPost
    fields = ['title', 'excerpt', 'content', 'featured_image', 'status']
    template_name = 'blog/post_form.html'
    
    def form_valid(self, form):
        form.instance.author = self.request.user
        if form.instance.status == 'published':
            form.instance.published_at = timezone.now()
        
        logger.info(f"Post created by {self.request.user}: {form.instance.title}")
        return super().form_valid(form)
    
    def get_success_url(self):
        return self.object.get_absolute_url()

class BlogUpdateView(LoginRequiredMixin, UserPassesTestMixin, UpdateView):
    """Update blog post (author only)"""
    model = BlogPost
    fields = ['title', 'excerpt', 'content', 'featured_image', 'status']
    template_name = 'blog/post_form.html'
    slug_field = 'slug'
    slug_url_kwarg = 'slug'
    
    def test_func(self):
        post = self.get_object()
        return post.author == self.request.user
    
    def form_valid(self, form):
        if form.cleaned_data['status'] == 'published' and not self.object.published_at:
            form.instance.published_at = timezone.now()
        
        logger.info(f"Post updated by {self.request.user}: {form.instance.title}")
        return super().form_valid(form)
    
    def get_success_url(self):
        return self.object.get_absolute_url()

class BlogDeleteView(LoginRequiredMixin, UserPassesTestMixin, DeleteView):
    """Delete blog post (author only)"""
    model = BlogPost
    template_name = 'blog/post_confirm_delete.html'
    slug_field = 'slug'
    slug_url_kwarg = 'slug'
    
    def test_func(self):
        post = self.get_object()
        return post.author == self.request.user or self.request.user.is_staff
    
    def delete(self, request, *args, **kwargs):
        post = self.get_object()
        logger.info(f"Post deleted by {request.user}: {post.title}")
        return super().delete(request, *args, **kwargs)
    
    def get_success_url(self):
        from django.urls import reverse
        return reverse('post_list')

# API Views
class BlogAPIListView(ListView):
    """JSON API for blog posts"""
    model = BlogPost
    
    def get_queryset(self):
        return BlogPost.objects.filter(status='published')
    
    def render_to_response(self, context, **response_kwargs):
        posts = self.get_queryset()
        
        # Pagination
        page = self.request.GET.get('page', 1)
        paginator = Paginator(posts, 10)
        page_obj = paginator.get_page(page)
        
        data = {
            'count': paginator.count,
            'total_pages': paginator.num_pages,
            'current_page': int(page),
            'results': [
                {
                    'id': post.id,
                    'title': post.title,
                    'slug': post.slug,
                    'excerpt': post.excerpt,
                    'author': post.author.username,
                    'created_at': post.created_at.isoformat(),
                    'published_at': post.published_at.isoformat() if post.published_at else None,
                    'view_count': post.view_count,
                    'url': post.get_absolute_url(),
                }
                for post in page_obj
            ]
        }
        
        return JsonResponse(data)

# urls.py
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    path('', views.BlogListView.as_view(), name='post_list'),
    path('post/<slug:slug>/', views.BlogDetailView.as_view(), name='post_detail'),
    path('create/', views.BlogCreateView.as_view(), name='post_create'),
    path('post/<slug:slug>/edit/', views.BlogUpdateView.as_view(), name='post_update'),
    path('post/<slug:slug>/delete/', views.BlogDeleteView.as_view(), name='post_delete'),
    path('api/posts/', views.BlogAPIListView.as_view(), name='api_posts'),
]
```

### 8.2 Advanced Mixin Combination

```python
from django.views.generic import ListView, DetailView
from django.db.models import Q, Count
from django.http import JsonResponse
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page
import logging

logger = logging.getLogger(__name__)

# Custom Mixins
class SearchFilterMixin:
    """Search and filter functionality"""
    search_fields = []
    filter_fields = {}
    
    def get_queryset(self):
        queryset = super().get_queryset()
        
        # Search
        search = self.request.GET.get('search')
        if search and self.search_fields:
            q = Q()
            for field in self.search_fields:
                q |= Q(**{f'{field}__icontains': search})
            queryset = queryset.filter(q)
        
        # Filter
        for field, param in self.filter_fields.items():
            value = self.request.GET.get(param)
            if value:
                queryset = queryset.filter(**{field: value})
        
        return queryset

class PaginationMixin:
    """Advanced pagination"""
    default_page_size = 10
    
    def get_page_size(self):
        size = self.request.GET.get('page_size')
        if size and size.isdigit() and int(size) <= 100:
            return int(size)
        return self.default_page_size
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['page_size'] = self.get_page_size()
        context['page_sizes'] = [10, 25, 50]
        return context

class CacheMixin:
    """Cache responses"""
    cache_timeout = 300
    
    @method_decorator(cache_page(cache_timeout))
    def dispatch(self, *args, **kwargs):
        return super().dispatch(*args, **kwargs)

class LoggingMixin:
    """Log view access"""
    
    def dispatch(self, request, *args, **kwargs):
        logger.info(
            f"Accessing {self.__class__.__name__} - "
            f"User: {request.user}, Method: {request.method}"
        )
        return super().dispatch(request, *args, **kwargs)

class JSONMixin:
    """Support JSON responses"""
    
    def render_to_response(self, context, **response_kwargs):
        if self.request.headers.get('Accept') == 'application/json':
            return self.render_to_json(context)
        return super().render_to_response(context, **response_kwargs)
    
    def render_to_json(self, context):
        raise NotImplementedError("Implement render_to_json()")

# Combined usage
class AdvancedBlogListView(
    SearchFilterMixin,
    PaginationMixin,
    CacheMixin,
    LoggingMixin,
    ListView
):
    """Advanced blog list with all features"""
    model = BlogPost
    template_name = 'blog/advanced_list.html'
    context_object_name = 'posts'
    search_fields = ['title', 'content', 'author__username']
    filter_fields = {
        'status': 'status',
        'author_id': 'author'
    }
    default_page_size = 15
    cache_timeout = 600
    
    def get_queryset(self):
        queryset = super().get_queryset()
        return queryset.select_related('author').annotate(
            comment_count=Count('comments')
        )
    
    def paginate_queryset(self, queryset, page_size):
        page_size = self.get_page_size()
        return super().paginate_queryset(queryset, page_size)
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['total_views'] = BlogPost.objects.aggregate(
            total=Sum('view_count')
        )['total'] or 0
        return context
```

---

## Key Takeaways

| Concept | Purpose | Example |
|---------|---------|---------|
| **View** | Base class for all views | `class MyView(View):` |
| **as_view()** | Converts class to URL-callable | `path('url/', MyView.as_view())` |
| **dispatch()** | Routes HTTP method to handler | `def dispatch(self, request):` |
| **Generic Views** | Pre-built views for common patterns | `ListView`, `DetailView`, `CreateView` |
| **Mixins** | Reusable functionality | `LoginRequiredMixin`, custom mixins |
| **get_queryset()** | Filter objects | Override to customize query |
| **get_context_data()** | Add template variables | Override to add more context |
| **form_valid()** | Process valid forms | Override to customize save logic |
| **render_to_response()** | Change response type | Override for JSON, CSV, PDF |
| **dispatch()** | Handle exceptions | Try-catch to handle errors |

---

## Best Practices

1. **Use Generic Views**: Reduces code, follows conventions
2. **Leverage Mixins**: DRY - Don't Repeat Yourself
3. **Override Strategically**: Only override what you need
4. **Test Permissions**: Always verify user access
5. **Log Important Actions**: Track who did what and when
6. **Cache When Appropriate**: Improve performance
7. **Handle Exceptions**: Provide meaningful error messages
8. **Use get_queryset()**: Filter at database level
9. **Document Custom Mixins**: Explain inheritance order
10. **Keep Views Thin**: Move logic to models/utilities
