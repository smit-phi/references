# Django & DRF Relations — A Complete Intuition Guide

> **The core idea:** A database is a collection of tables with rows and columns. Relations are how you connect those tables so that data in one table can *refer to* data in another — without duplicating it. Django's ORM and DRF's serializers are two separate layers that each handle this in their own way. This guide builds intuition for both, and how they connect.

---

## Table of Contents

1. [The Relational Intuition — Before Any Code](#1-the-relational-intuition--before-any-code)
2. [ForeignKey — Many-to-One](#2-foreignkey--many-to-one)
3. [OneToOneField](#3-onetoonefield)
4. [ManyToManyField](#4-manytomanyfield)
5. [The `related_name` Argument — Reverse Relations](#5-the-related_name-argument--reverse-relations)
6. [ORM Queries Across Relations](#6-orm-queries-across-relations)
7. [The N+1 Problem — The Silent Performance Killer](#7-the-n1-problem--the-silent-performance-killer)
8. [select_related vs prefetch_related](#8-select_related-vs-prefetch_related)
9. [Annotate, Aggregate Across Relations](#9-annotate-aggregate-across-relations)
10. [Through Models — M2M with Extra Data](#10-through-models--m2m-with-extra-data)
11. [Self-Referential Relations](#11-self-referential-relations)
12. [Generic Relations (ContentTypes)](#12-generic-relations-contenttypes)
13. [DRF Serializer Relations — The Full Picture](#13-drf-serializer-relations--the-full-picture)
14. [PrimaryKeyRelatedField](#14-primarykeyrelatedfield)
15. [StringRelatedField](#15-stringrelatedfield)
16. [SlugRelatedField](#16-slugrelatedfield)
17. [HyperlinkedRelatedField](#17-hyperlinkedrelatedfield)
18. [Nested Serializers — Read](#18-nested-serializers--read)
19. [Nested Serializers — Write (The Hard Part)](#19-nested-serializers--write-the-hard-part)
20. [The Read/Write Serializer Split Pattern](#20-the-readwrite-serializer-split-pattern)
21. [SerializerMethodField for Relation Data](#21-serializermethodfield-for-relation-data)
22. [Writable M2M Relations](#22-writable-m2m-relations)
23. [Depth — Automatic Nesting](#23-depth--automatic-nesting)
24. [Source Argument — Renaming and Traversing](#24-source-argument--renaming-and-traversing)
25. [Handling Reverse Relations in Serializers](#25-handling-reverse-relations-in-serializers)
26. [Serializer Context — Passing Request into Relations](#26-serializer-context--passing-request-into-relations)
27. [Performance in Serializers — select_related meets DRF](#27-performance-in-serializers--select_related-meets-drf)
28. [Mental Model Cheat Sheet](#28-mental-model-cheat-sheet)

---

## 1. The Relational Intuition — Before Any Code

Imagine a blogging platform. You have:
- Users
- Posts written by users
- Comments on posts by users
- Tags applied to posts
- Likes given by users to posts

If you stored everything in one table, you'd get this nightmare:

```
| post_title | post_content | author_name | author_email | comment_text | comment_author | tag1  | tag2   |
|------------|--------------|-------------|--------------|--------------|----------------|-------|--------|
| Hello      | ...          | Alice       | a@x.com      | Great post!  | Bob            | django| python |
| Hello      | ...          | Alice       | a@x.com      | Me too!      | Carol          | django| python |
```

Alice's email is duplicated on every comment row. If Alice changes her email, you have to update every row. This is called an **update anomaly**. If you delete all posts by Alice, you lose Alice's email entirely — a **deletion anomaly**.

The solution is **normalization**: store each thing once, in its own table, and connect tables with references (foreign keys).

```
Users table:       id | name  | email
                   1  | Alice | a@x.com

Posts table:       id | title | author_id (→ Users.id)
                   1  | Hello | 1

Comments table:    id | text        | post_id (→ Posts.id) | author_id (→ Users.id)
                   1  | Great post! | 1                    | 2

Tags table:        id | name
                   1  | django

PostTags table:    post_id | tag_id   ← join table for M2M
                   1       | 1
```

Each table has one job. Relations connect them. This is the database design that Django's ORM is built to work with.

### The Three Fundamental Relationships

**Many-to-One (ForeignKey):** Many posts can belong to one author. One author can have many posts. The "many" side holds the foreign key.

**One-to-One:** One user has exactly one profile. One profile belongs to exactly one user. Essentially a FK with a uniqueness constraint.

**Many-to-Many:** A post can have many tags. A tag can apply to many posts. Requires a junction/join table.

---

## 2. ForeignKey — Many-to-One

### The Database Reality

When you define a `ForeignKey` in Django, it creates a column in the "many" table that stores the `id` of the related row in the "one" table.

```python
# models.py
from django.db import models
from django.contrib.auth.models import User

class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey(
        User,
        on_delete=models.CASCADE,
        related_name='posts'
    )
    created_at = models.DateTimeField(auto_now_add=True)
```

The generated SQL column is `author_id` (Django appends `_id` to the field name). In Python you access it two ways:

```python
post.author      # Returns the full User object — triggers a DB query if not prefetched
post.author_id   # Returns just the integer id — no extra query, already in the row
```

Knowing this distinction matters for performance. If you only need the id, use `post.author_id` — not `post.author.id` (which loads the entire User object for nothing).

### `on_delete` — What Happens When the Parent is Deleted

This is one of the most important decisions you make when defining a FK.

```python
# CASCADE — delete all posts when the author is deleted (most common)
author = models.ForeignKey(User, on_delete=models.CASCADE)

# PROTECT — prevent deleting the author if they have posts
author = models.ForeignKey(User, on_delete=models.PROTECT)

# SET_NULL — set author to NULL when user is deleted (field must be nullable)
author = models.ForeignKey(User, on_delete=models.SET_NULL, null=True, blank=True)

# SET_DEFAULT — set to a default value
author = models.ForeignKey(User, on_delete=models.SET_DEFAULT, default=1)

# DO_NOTHING — do nothing in Django (dangerous — may break DB constraints)
author = models.ForeignKey(User, on_delete=models.DO_NOTHING)

# RESTRICT — like PROTECT but allows deletion if there's a cascading path
author = models.ForeignKey(User, on_delete=models.RESTRICT)
```

**Rule of thumb:**
- `CASCADE` for ownership (post belongs to user — delete user, delete posts)
- `PROTECT` for reference data you can't afford to orphan (order references a product — don't let product be deleted)
- `SET_NULL` for optional associations (post has an optional editor — editor can leave)

### Accessing Related Objects

```python
# Forward — from the "many" side to the "one" side
post = Post.objects.get(pk=1)
post.author          # The User object
post.author.email    # Traverse the relation

# Reverse — from the "one" side to the "many" side
user = User.objects.get(pk=1)
user.posts.all()           # All posts by this user (uses related_name='posts')
user.posts.count()
user.posts.filter(published=True)
```

### `null` vs `blank` on ForeignKey

```python
# null=True — the DB column allows NULL (the FK can be absent)
# blank=True — Django forms allow empty input
# Both are needed for a truly optional FK

editor = models.ForeignKey(
    User,
    on_delete=models.SET_NULL,
    null=True,
    blank=True,
    related_name='edited_posts'
)
```

---

## 3. OneToOneField

### The Intuition

A `OneToOneField` is a `ForeignKey` with `unique=True`. Each row in table A corresponds to at most one row in table B.

The canonical Django pattern: extending the built-in `User` model without replacing it.

```python
class UserProfile(models.Model):
    user = models.OneToOneField(
        User,
        on_delete=models.CASCADE,
        related_name='profile'
    )
    bio = models.TextField(blank=True)
    avatar_url = models.URLField(blank=True)
    location = models.CharField(max_length=100, blank=True)
    website = models.URLField(blank=True)
```

### Access Pattern

```python
# Forward
profile = UserProfile.objects.get(user=request.user)

# Reverse — the related_name makes this natural
user = User.objects.get(pk=1)
user.profile          # The UserProfile object
user.profile.bio      # Traverse

# The danger: RelatedObjectDoesNotExist
# If the profile doesn't exist yet, user.profile raises an exception
try:
    profile = user.profile
except UserProfile.DoesNotExist:
    profile = None

# Or use hasattr
if hasattr(user, 'profile'):
    bio = user.profile.bio
```

### Auto-creating with Signals

In real projects, you want the profile to be created automatically when a user registers:

```python
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender=User)
def create_user_profile(sender, instance, created, **kwargs):
    if created:
        UserProfile.objects.create(user=instance)

@receiver(post_save, sender=User)
def save_user_profile(sender, instance, **kwargs):
    instance.profile.save()
```

---

## 4. ManyToManyField

### The Database Reality

Django creates a **junction table** (also called a join table or through table) automatically. For `Post.tags`, Django creates a table called `appname_post_tags` with columns `post_id` and `tag_id`.

```python
class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)
    slug = models.SlugField(unique=True)

    def __str__(self):
        return self.name

class Post(models.Model):
    title = models.CharField(max_length=200)
    tags = models.ManyToManyField(
        Tag,
        related_name='posts',
        blank=True    # blank=True because a post can have no tags
    )
```

Note: `ManyToManyField` never gets `null=True` — the junction table simply has no rows when there are no relations. `null=True` on M2M has no effect.

### Managing M2M Relations

```python
post = Post.objects.get(pk=1)
tag_django = Tag.objects.get(slug='django')
tag_python = Tag.objects.get(slug='python')

# Add tags
post.tags.add(tag_django)
post.tags.add(tag_django, tag_python)    # multiple at once

# Remove a tag
post.tags.remove(tag_django)

# Replace all tags at once
post.tags.set([tag_django, tag_python])  # replaces entirely

# Clear all
post.tags.clear()

# Check membership
post.tags.filter(slug='django').exists()

# Querying
post.tags.all()
tag_django.posts.all()   # reverse: all posts with this tag
```

### Where to Define the M2M Field

You can define it on either side — it creates the same junction table. Convention: define it on the model that "owns" the relationship conceptually.

```python
# Equivalent — Django creates the same table either way
class Post(models.Model):
    tags = models.ManyToManyField(Tag)  # defined on Post

# OR

class Tag(models.Model):
    posts = models.ManyToManyField(Post)  # defined on Tag
```

---

## 5. The `related_name` Argument — Reverse Relations

### Why It Matters

Every FK and M2M creates a **reverse accessor** on the target model. By default, Django names it `<modelname>_set`.

```python
user.post_set.all()    # default — ugly, unclear
user.posts.all()       # with related_name='posts' — clean, readable
```

Always set `related_name` explicitly. It documents your intent and keeps your code readable.

### Naming Convention

Use the plural lowercase name of the model that holds the FK:

```python
class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='comments')
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name='comments')
```

Wait — two `related_name='comments'` on different models targeting different models is fine. `post.comments` and `user.comments` are different accessors on different models.

But this would be a problem:

```python
class Post(models.Model):
    author = models.ForeignKey(User, related_name='posts')
    editor = models.ForeignKey(User, related_name='posts')  # CLASH — same name on User
```

Fix:

```python
author = models.ForeignKey(User, related_name='authored_posts')
editor = models.ForeignKey(User, related_name='edited_posts')
```

### `related_name='+'` — Disabling the Reverse Accessor

If you never need to traverse the relation backwards, disable it:

```python
thumbnail = models.ForeignKey(Image, on_delete=models.SET_NULL, null=True, related_name='+')
```

This tells Django not to create a reverse accessor at all — cleaner when the reverse direction is meaningless.

---

## 6. ORM Queries Across Relations

### The Double-Underscore Traversal Syntax

Django ORM uses `__` (double underscore) to traverse relations in filter/query expressions. This is one of the most powerful features in Django.

```python
# Find all posts by a user with a specific email
Post.objects.filter(author__email='alice@example.com')

# Find all posts tagged 'django'
Post.objects.filter(tags__slug='django')

# Find all posts where the author's profile has a location set
Post.objects.filter(author__profile__location__isnull=False)

# You can chain as deep as needed
Comment.objects.filter(post__author__profile__location='Mumbai')
```

The `__` traversal works with any lookup expression at the end:

```python
Post.objects.filter(author__username__startswith='a')
Post.objects.filter(author__date_joined__year=2024)
Post.objects.filter(tags__name__in=['django', 'python'])
Post.objects.filter(title__icontains='rest')
```

### Filtering on Reverse Relations

The traversal works in both directions:

```python
# Users who have written at least one post
User.objects.filter(posts__isnull=False).distinct()

# Users who have a comment on a post tagged 'django'
User.objects.filter(comments__post__tags__slug='django').distinct()

# The .distinct() is important — without it, a user with 5 posts appears 5 times
```

### Spanning to Aggregated Values

```python
from django.db.models import Count, Avg

# Users with more than 10 posts
User.objects.annotate(post_count=Count('posts')).filter(post_count__gt=10)

# Posts with their comment count, ordered
Post.objects.annotate(comment_count=Count('comments')).order_by('-comment_count')
```

### `values()` and `values_list()` Across Relations

```python
# Get flat list of author emails for all posts
Post.objects.values_list('author__email', flat=True)

# Dict output across FK
Post.objects.values('title', 'author__username', 'author__email')
# → [{'title': 'Hello', 'author__username': 'alice', 'author__email': 'a@x.com'}, ...]
```

---

## 7. The N+1 Problem — The Silent Performance Killer

This is the most important performance concept in any ORM. You will encounter it constantly.

### The Problem

```python
posts = Post.objects.all()  # 1 query: SELECT * FROM posts

for post in posts:
    print(post.author.name)  # 1 query PER POST: SELECT * FROM users WHERE id = ?
```

If there are 100 posts, this is **101 queries**. For 1000 posts it's 1001 queries. The database does `N+1` round trips — 1 for the list, N for each related object. This is invisible in development (fast local DB) but catastrophic in production.

The name comes from the pattern: 1 query for the collection + N queries for each item's relation.

```python
# How to spot it: add this to settings.py for development
import logging
LOGGING = {
    'version': 1,
    'handlers': {'console': {'class': 'logging.StreamHandler'}},
    'loggers': {
        'django.db.backends': {
            'handlers': ['console'],
            'level': 'DEBUG',
        },
    },
}
```

### Why It Happens

Django's ORM is lazy. When you access `post.author`, it executes a fresh `SELECT` against the database at that moment — unless the related data was already loaded. The ORM doesn't know you're about to loop over all posts and access `.author` on each.

---

## 8. `select_related` vs `prefetch_related`

These are the two tools that eliminate N+1 queries. They work differently and are appropriate for different relationship types.

### `select_related` — SQL JOIN for Forward FK/O2O

Uses a SQL `JOIN` to fetch the main object and its FK/O2O related objects in a **single query**.

```python
# Without select_related: N+1
posts = Post.objects.all()
for post in posts:
    print(post.author.name)  # query per post

# With select_related: 1 query using JOIN
posts = Post.objects.select_related('author').all()
for post in posts:
    print(post.author.name)  # no extra query — already loaded
```

Generated SQL:
```sql
SELECT posts.*, auth_user.*
FROM posts
INNER JOIN auth_user ON posts.author_id = auth_user.id
```

**Multi-level traversal:**
```python
# Load post + author + author's profile in one JOIN query
Post.objects.select_related('author', 'author__profile')

# All FK/O2O in both directions (use with caution — can over-fetch)
Post.objects.select_related()
```

**When to use:** Forward FK relationships (`post.author`) and OneToOne (`user.profile`). Works only for FK and O2O — not M2M or reverse FK.

### `prefetch_related` — Separate Queries for M2M and Reverse FK

Does **two queries**: one for the main queryset, one for the related objects. Then Python joins them in memory. This is necessary for M2M and reverse FK because a SQL JOIN would produce duplicate rows.

```python
# Without prefetch: N+1
posts = Post.objects.all()
for post in posts:
    print(post.tags.all())  # query per post

# With prefetch_related: 2 queries total
posts = Post.objects.prefetch_related('tags').all()
for post in posts:
    print(post.tags.all())  # no extra query — already prefetched
```

Generated SQL (two separate queries):
```sql
SELECT * FROM posts;
SELECT * FROM tags
INNER JOIN post_tags ON tags.id = post_tags.tag_id
WHERE post_tags.post_id IN (1, 2, 3, 4, 5, ...);
```

**Reverse FK example:**
```python
# Each post with all its comments — 2 queries
posts = Post.objects.prefetch_related('comments').all()
for post in posts:
    post.comments.all()  # already cached, no extra query
```

### `Prefetch` Object — Fine-Grained Control

```python
from django.db.models import Prefetch

# Prefetch only published comments, with custom queryset
posts = Post.objects.prefetch_related(
    Prefetch(
        'comments',
        queryset=Comment.objects.select_related('author').filter(approved=True),
        to_attr='approved_comments'  # stored as a list, not a manager
    )
)

for post in posts:
    post.approved_comments   # a Python list, not a queryset
```

### Combining Both

```python
# The complete pattern for a blog list endpoint:
posts = Post.objects.select_related(
    'author',           # FK — use JOIN
    'author__profile',  # FK through FK — same JOIN
    'category',         # another FK
).prefetch_related(
    'tags',                           # M2M — separate query
    Prefetch('comments',              # reverse FK with filter
        queryset=Comment.objects.select_related('author').order_by('-created_at')[:3],
        to_attr='recent_comments'
    )
)
```

### The Decision Rule

```
Relationship type              → Tool
─────────────────────────────────────────────────
ForeignKey (forward)           → select_related
OneToOneField (forward)        → select_related
ForeignKey (reverse)           → prefetch_related
ManyToManyField                → prefetch_related
FK through FK (author__profile)→ select_related
Reverse FK with filtering      → Prefetch() object
```

---

## 9. Annotate, Aggregate Across Relations

You can compute values across related tables and attach them to queryset results — without loading related objects into Python.

### `annotate` — Per-Row Computed Values

```python
from django.db.models import Count, Avg, Sum, Max, Min, Q

# Add comment_count to each post in the queryset
posts = Post.objects.annotate(comment_count=Count('comments'))
posts[0].comment_count   # available as an attribute

# Count only approved comments
posts = Post.objects.annotate(
    approved_comments=Count('comments', filter=Q(comments__approved=True))
)

# Multiple annotations
posts = Post.objects.annotate(
    comment_count=Count('comments', distinct=True),
    tag_count=Count('tags', distinct=True),
    latest_comment=Max('comments__created_at')
)

# distinct=True is crucial when annotating across M2M/multi-FK
# Without it, the JOINs can inflate counts
```

### `aggregate` — Single Value Across the Whole Queryset

```python
from django.db.models import Count, Avg

# Total posts in the database
Post.objects.aggregate(total=Count('id'))
# → {'total': 4231}

# Average number of comments per post
Post.objects.aggregate(avg_comments=Avg('comments'))
# → {'avg_comments': 3.7}
```

### Filtering on Annotated Values

```python
# Posts with more than 5 comments
Post.objects.annotate(comment_count=Count('comments')).filter(comment_count__gt=5)

# Users who have made more than 3 posts this year
from django.utils import timezone
User.objects.annotate(
    recent_posts=Count(
        'posts',
        filter=Q(posts__created_at__year=timezone.now().year)
    )
).filter(recent_posts__gt=3)
```

---

## 10. Through Models — M2M with Extra Data

### The Problem

A plain `ManyToManyField` creates a junction table with only `id`, `model_a_id`, and `model_b_id`. What if you need extra data on the relationship itself?

Example: A user can "enroll" in a course. The enrollment has a `date_enrolled`, `grade`, and `status`.

### Defining a Through Model

```python
class Course(models.Model):
    title = models.CharField(max_length=200)
    students = models.ManyToManyField(
        User,
        through='Enrollment',
        related_name='courses'
    )

class Enrollment(models.Model):
    student = models.ForeignKey(User, on_delete=models.CASCADE, related_name='enrollments')
    course = models.ForeignKey(Course, on_delete=models.CASCADE, related_name='enrollments')
    date_enrolled = models.DateTimeField(auto_now_add=True)
    grade = models.DecimalField(max_digits=5, decimal_places=2, null=True, blank=True)
    status = models.CharField(
        max_length=20,
        choices=[('active', 'Active'), ('completed', 'Completed'), ('dropped', 'Dropped')],
        default='active'
    )

    class Meta:
        unique_together = ('student', 'course')  # A student can only enroll once
```

### Querying Through Models

```python
# With a through model, you CANNOT use .add()/.remove()/.set()
# You must create/delete Enrollment objects directly

# Enroll a student
Enrollment.objects.create(student=user, course=course)

# All courses for a user (via M2M)
user.courses.all()

# All courses for a user WITH enrollment data
user.enrollments.select_related('course').all()
for e in user.enrollments.all():
    print(e.course.title, e.grade, e.status)

# All active students in a course
course.enrollments.filter(status='active').select_related('student')

# Query through the through model for aggregate data
from django.db.models import Avg
Course.objects.annotate(avg_grade=Avg('enrollments__grade'))
```

### Another Classic Example: Post Likes

```python
class Like(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='likes')
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='likes')
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = ('user', 'post')

# Toggle a like
def toggle_like(user, post):
    like, created = Like.objects.get_or_create(user=user, post=post)
    if not created:
        like.delete()
        return False
    return True
```

---

## 11. Self-Referential Relations

A model that relates to itself. The canonical examples are:

- A user who follows other users (M2M self)
- A comment that can have replies (FK self)
- A category with subcategories (FK self, tree structure)
- An employee with a manager (FK self)

### FK Self-Referential

```python
class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='comments')
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    content = models.TextField()
    parent = models.ForeignKey(
        'self',           # 'self' is the magic string
        on_delete=models.CASCADE,
        null=True,
        blank=True,
        related_name='replies'
    )

# Top-level comments (no parent)
Comment.objects.filter(post=post, parent=None)

# Replies to a comment
comment.replies.all()

# Deep nesting (be careful with recursion)
def get_tree(comment):
    return {
        'comment': comment,
        'replies': [get_tree(r) for r in comment.replies.all()]
    }
```

### M2M Self-Referential (Followers)

```python
class UserProfile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name='profile')
    following = models.ManyToManyField(
        'self',
        symmetrical=False,    # CRITICAL for directed graphs — A follows B ≠ B follows A
        related_name='followers',
        blank=True
    )

# Follow a user
alice.profile.following.add(bob.profile)

# Alice's following list
alice.profile.following.all()

# Alice's followers
alice.profile.followers.all()
```

`symmetrical=True` (the default for `'self'` M2M) means if A is related to B, B is automatically related to A — like Facebook friends. `symmetrical=False` is for directed relationships — like Twitter follows.

### Employee Hierarchy

```python
class Employee(models.Model):
    name = models.CharField(max_length=100)
    manager = models.ForeignKey(
        'self',
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name='direct_reports'
    )

# Get all direct reports
ceo.direct_reports.all()

# Find all subordinates recursively
def get_all_reports(employee):
    reports = list(employee.direct_reports.all())
    for report in list(reports):
        reports.extend(get_all_reports(report))
    return reports
```

---

## 12. Generic Relations (ContentTypes)

### The Problem

Sometimes you need a model to relate to *any other model* — not a fixed target. Example: a `Comment` that can be on a `Post`, a `Photo`, or a `Video`. Without generic relations, you'd need:

```python
class Comment(models.Model):
    post = models.ForeignKey(Post, null=True, blank=True)
    photo = models.ForeignKey(Photo, null=True, blank=True)
    video = models.ForeignKey(Video, null=True, blank=True)
    # Add a new FK every time you add a commentable model — not scalable
```

### GenericForeignKey

Django's `contenttypes` framework lets you build a generic pointer to any model:

```python
from django.contrib.contenttypes.fields import GenericForeignKey, GenericRelation
from django.contrib.contenttypes.models import ContentType

class Comment(models.Model):
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    # These two fields together form the generic FK
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.PositiveIntegerField()
    content_object = GenericForeignKey('content_type', 'object_id')
    # content_object is not a real DB column — it's a Python descriptor
```

```python
# Add a comment to a post
post = Post.objects.get(pk=1)
Comment.objects.create(
    author=request.user,
    content="Great post!",
    content_object=post    # Django sets content_type and object_id automatically
)

# Add a comment to a photo
photo = Photo.objects.get(pk=5)
Comment.objects.create(
    author=request.user,
    content="Beautiful!",
    content_object=photo
)
```

### `GenericRelation` — Reverse Access

```python
class Post(models.Model):
    title = models.CharField(max_length=200)
    comments = GenericRelation(Comment, related_query_name='post')
    # Now you can do post.comments.all()
```

```python
post.comments.all()
post.comments.count()

# Cascading delete: GenericRelation on Post means deleting a post
# automatically deletes all its comments (no on_delete needed)
```

### When to Use Generic Relations

Use them for truly polymorphic features:
- Comments on any content type
- Likes/bookmarks on any model  
- Tags applied to any model (vs. a dedicated M2M per model)
- Activity/audit logs

Avoid them when you always know the target model — regular FKs have better performance and clearer semantics.

---

## 13. DRF Serializer Relations — The Full Picture

### The Conceptual Boundary

Django ORM handles relations at the **database layer** — defining, querying, traversing. DRF serializers handle relations at the **API boundary** — how related objects appear in JSON responses and how the API accepts them in requests.

The two layers are distinct but deeply connected. A DRF serializer field for a FK can output:
- The raw integer id
- A URL to the related object
- A string representation
- The full nested object (another serializer embedded inside)
- A custom computed value

Each choice has different tradeoffs in terms of API ergonomics, bandwidth, and client complexity.

Here's the full map of options:

```
Django ForeignKey field on a model
        │
        ▼
DRF Serializer — how to represent it in the API?
        │
        ├── PrimaryKeyRelatedField      → { "author": 1 }
        ├── StringRelatedField          → { "author": "alice" }
        ├── SlugRelatedField            → { "author": "alice-smith" }
        ├── HyperlinkedRelatedField     → { "author": "http://api/users/1/" }
        ├── Nested Serializer           → { "author": { "id": 1, "name": "Alice" } }
        └── SerializerMethodField       → { "author": <anything you compute> }
```

---

## 14. PrimaryKeyRelatedField

The default behavior of `ModelSerializer` for FK fields. Outputs/accepts the primary key of the related object.

```python
class PostSerializer(serializers.ModelSerializer):
    # This is implicit — ModelSerializer does this automatically for FKs
    author = serializers.PrimaryKeyRelatedField(queryset=User.objects.all())

    class Meta:
        model = Post
        fields = ['id', 'title', 'author', 'tags']
```

Output:
```json
{
    "id": 1,
    "title": "Hello Django",
    "author": 3,
    "tags": [1, 4, 7]
}
```

Input (POST/PUT):
```json
{
    "title": "New Post",
    "author": 3,
    "tags": [1, 4]
}
```

**`read_only=True`** when you never want the client to set this field:
```python
author = serializers.PrimaryKeyRelatedField(read_only=True)
```

**`many=True`** for M2M:
```python
tags = serializers.PrimaryKeyRelatedField(queryset=Tag.objects.all(), many=True)
```

**Limiting the queryset** (critical for security and correctness):
```python
# Only allow assigning tags that belong to the current user's workspace
def get_fields(self):
    fields = super().get_fields()
    request = self.context.get('request')
    if request:
        fields['tags'].queryset = Tag.objects.filter(workspace=request.user.workspace)
    return fields
```

---

## 15. StringRelatedField

Uses the related object's `__str__` method. Always read-only — makes no sense for write because a string can't uniquely identify an object (unless it's also unique).

```python
class PostSerializer(serializers.ModelSerializer):
    author = serializers.StringRelatedField()
    tags = serializers.StringRelatedField(many=True)

    class Meta:
        model = Post
        fields = ['id', 'title', 'author', 'tags']
```

Output (assuming `User.__str__` returns `username`):
```json
{
    "id": 1,
    "title": "Hello Django",
    "author": "alice",
    "tags": ["django", "python", "rest"]
}
```

Use when: you want a human-readable label and the field is output-only.

---

## 16. SlugRelatedField

Uses a specific field on the related object (not necessarily the PK) for both read and write. The field must be unique on the target model.

```python
class PostSerializer(serializers.ModelSerializer):
    author = serializers.SlugRelatedField(
        slug_field='username',         # use this field instead of pk
        queryset=User.objects.all()
    )
    tags = serializers.SlugRelatedField(
        slug_field='slug',
        queryset=Tag.objects.all(),
        many=True
    )

    class Meta:
        model = Post
        fields = ['id', 'title', 'author', 'tags']
```

Output:
```json
{
    "id": 1,
    "title": "Hello Django",
    "author": "alice",
    "tags": ["django", "python"]
}
```

Input: `{"title": "New Post", "author": "alice", "tags": ["django"]}` — DRF looks up `User.objects.get(username='alice')`.

Use when: clients should identify related objects by a natural key (slug, username, code) rather than an opaque integer ID.

---

## 17. HyperlinkedRelatedField

Returns a full URL instead of a PK. This is HATEOAS — the response itself tells the client where to go to get more information.

```python
class PostSerializer(serializers.HyperlinkedModelSerializer):
    author = serializers.HyperlinkedRelatedField(
        view_name='user-detail',   # the URL name from your router
        read_only=True
    )

    class Meta:
        model = Post
        fields = ['url', 'title', 'author']
```

Output:
```json
{
    "url": "http://api.example.com/posts/1/",
    "title": "Hello Django",
    "author": "http://api.example.com/users/3/"
}
```

Requires: the request object in serializer context (DRF generic views pass this automatically), and the related model must have a detail view registered.

---

## 18. Nested Serializers — Read

Embedding the full representation of a related object inside the parent response.

```python
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email']

class TagSerializer(serializers.ModelSerializer):
    class Meta:
        model = Tag
        fields = ['id', 'name', 'slug']

class PostSerializer(serializers.ModelSerializer):
    author = UserSerializer(read_only=True)     # nested FK
    tags = TagSerializer(many=True, read_only=True)  # nested M2M

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'tags']
```

Output:
```json
{
    "id": 1,
    "title": "Hello Django",
    "content": "...",
    "author": {
        "id": 3,
        "username": "alice",
        "email": "alice@example.com"
    },
    "tags": [
        {"id": 1, "name": "Django", "slug": "django"},
        {"id": 4, "name": "Python", "slug": "python"}
    ]
}
```

**Why `read_only=True`?** Because nested serializers by default don't know how to write back. If you allow writes, you have to handle the nested creation/update logic yourself. `read_only=True` is the safe default.

**Performance note:** Nested serializers make N+1 very easy to trigger. You must pair them with `select_related`/`prefetch_related` in your view. More on this in section 27.

---

## 19. Nested Serializers — Write (The Hard Part)

This is where most developers hit a wall. When you accept nested data in a write operation, you have to manage the creation/update of related objects yourself. DRF doesn't do this automatically because there are too many valid ways to handle it (create-or-get, create-new, update-existing, etc.).

### Writable Nested FK — Creating Related Objects

```python
class TagSerializer(serializers.ModelSerializer):
    class Meta:
        model = Tag
        fields = ['name', 'slug']

class PostSerializer(serializers.ModelSerializer):
    tags = TagSerializer(many=True)   # writable — no read_only

    class Meta:
        model = Post
        fields = ['title', 'content', 'tags']

    def create(self, validated_data):
        # Pop nested data before creating the parent
        tags_data = validated_data.pop('tags')

        # Create the post
        post = Post.objects.create(**validated_data)

        # Create or get tags and link them
        for tag_data in tags_data:
            tag, _ = Tag.objects.get_or_create(**tag_data)
            post.tags.add(tag)

        return post

    def update(self, instance, validated_data):
        tags_data = validated_data.pop('tags', None)

        # Update the post's own fields
        for attr, value in validated_data.items():
            setattr(instance, attr, value)
        instance.save()

        # Replace tags if provided
        if tags_data is not None:
            instance.tags.clear()
            for tag_data in tags_data:
                tag, _ = Tag.objects.get_or_create(**tag_data)
                instance.tags.add(tag)

        return instance
```

Input:
```json
{
    "title": "New Post",
    "content": "...",
    "tags": [
        {"name": "Django", "slug": "django"},
        {"name": "REST", "slug": "rest"}
    ]
}
```

### Writable Nested OneToOne — Creating Profile with User

```python
class ProfileSerializer(serializers.ModelSerializer):
    class Meta:
        model = UserProfile
        fields = ['bio', 'location', 'avatar_url']

class UserSerializer(serializers.ModelSerializer):
    profile = ProfileSerializer()

    class Meta:
        model = User
        fields = ['username', 'email', 'password', 'profile']
        extra_kwargs = {'password': {'write_only': True}}

    def create(self, validated_data):
        profile_data = validated_data.pop('profile')
        password = validated_data.pop('password')

        user = User.objects.create(**validated_data)
        user.set_password(password)    # hash the password
        user.save()

        UserProfile.objects.create(user=user, **profile_data)

        return user

    def update(self, instance, validated_data):
        profile_data = validated_data.pop('profile', None)

        # Update User fields
        for attr, value in validated_data.items():
            setattr(instance, attr, value)
        instance.save()

        # Update Profile fields
        if profile_data:
            profile = instance.profile
            for attr, value in profile_data.items():
                setattr(profile, attr, value)
            profile.save()

        return instance
```

### The Key Pattern for All Writable Nested Serializers

```
1. In create()/update(), pop nested data from validated_data FIRST
2. Create/update the parent object with the remaining validated_data
3. Handle the nested objects separately
4. Link them to the parent
```

---

## 20. The Read/Write Serializer Split Pattern

This is one of the most important production patterns in DRF. The shape of data you want to *output* (GET) is often very different from what you want to *accept* (POST/PUT/PATCH).

```
GET /api/posts/1/ → return full nested objects (author name, tag names)
POST /api/posts/  → accept author_id and tag_ids (flat, simple)
```

Trying to do both in one serializer creates a mess of conditional logic. The cleaner pattern: two serializers.

```python
class PostReadSerializer(serializers.ModelSerializer):
    """Used for GET — rich, nested output."""
    author = UserSerializer(read_only=True)
    tags = TagSerializer(many=True, read_only=True)
    comment_count = serializers.IntegerField(read_only=True)  # from annotation

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'tags', 'comment_count', 'created_at']


class PostWriteSerializer(serializers.ModelSerializer):
    """Used for POST/PUT/PATCH — simple, flat input."""
    tags = serializers.PrimaryKeyRelatedField(
        queryset=Tag.objects.all(),
        many=True
    )

    class Meta:
        model = Post
        fields = ['title', 'content', 'tags']

    def create(self, validated_data):
        tags = validated_data.pop('tags')
        post = Post.objects.create(**validated_data)
        post.tags.set(tags)
        return post

    def update(self, instance, validated_data):
        tags = validated_data.pop('tags', None)
        for attr, value in validated_data.items():
            setattr(instance, attr, value)
        instance.save()
        if tags is not None:
            instance.tags.set(tags)
        return instance
```

```python
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.select_related('author').prefetch_related('tags')

    def get_serializer_class(self):
        if self.action in ('create', 'update', 'partial_update'):
            return PostWriteSerializer
        return PostReadSerializer

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

This is clean, testable, and maps perfectly to REST semantics.

---

## 21. SerializerMethodField for Relation Data

When you need custom logic to represent a relation — counts, computed booleans, conditional output — `SerializerMethodField` gives you full Python control.

```python
class PostSerializer(serializers.ModelSerializer):
    like_count = serializers.SerializerMethodField()
    is_liked_by_me = serializers.SerializerMethodField()
    top_tags = serializers.SerializerMethodField()
    author_name = serializers.SerializerMethodField()

    def get_like_count(self, obj):
        return obj.likes.count()   # or from annotation: obj.like_count

    def get_is_liked_by_me(self, obj):
        request = self.context.get('request')
        if not request or not request.user.is_authenticated:
            return False
        return obj.likes.filter(user=request.user).exists()

    def get_top_tags(self, obj):
        # Only return the first 3 tags
        return [tag.name for tag in obj.tags.all()[:3]]

    def get_author_name(self, obj):
        # Traverse relation, return formatted string
        return f"{obj.author.first_name} {obj.author.last_name}".strip() or obj.author.username

    class Meta:
        model = Post
        fields = ['id', 'title', 'like_count', 'is_liked_by_me', 'top_tags', 'author_name']
```

**Performance warning:** `get_like_count` hits the database once per post if `likes` isn't prefetched. Always pair `SerializerMethodField` that accesses relations with proper prefetching. Better pattern: use annotations:

```python
# In the view's queryset:
Post.objects.annotate(like_count=Count('likes'))

# In the serializer — reads from the annotation, not a query:
def get_like_count(self, obj):
    return getattr(obj, 'like_count', obj.likes.count())  # falls back if no annotation
```

---

## 22. Writable M2M Relations

Several patterns exist depending on what the client sends and what the API should do.

### Pattern 1: Accept List of PKs (most common)

```python
class PostSerializer(serializers.ModelSerializer):
    tag_ids = serializers.PrimaryKeyRelatedField(
        queryset=Tag.objects.all(),
        many=True,
        source='tags',    # maps to the actual 'tags' field on the model
        write_only=True
    )
    tags = TagSerializer(many=True, read_only=True)  # for output

    class Meta:
        model = Post
        fields = ['id', 'title', 'tag_ids', 'tags']

    def create(self, validated_data):
        tags = validated_data.pop('tags')  # source='tags' so it's under 'tags' key
        post = Post.objects.create(**validated_data)
        post.tags.set(tags)
        return post
```

### Pattern 2: Accept List of Slugs

```python
tag_ids = serializers.SlugRelatedField(
    slug_field='slug',
    queryset=Tag.objects.all(),
    many=True,
    source='tags'
)
```

### Pattern 3: create-or-get by name (accept new tag names)

```python
class TagField(serializers.ListField):
    child = serializers.CharField()

    def to_internal_value(self, data):
        names = super().to_internal_value(data)
        tags = []
        for name in names:
            tag, _ = Tag.objects.get_or_create(
                name=name,
                defaults={'slug': name.lower().replace(' ', '-')}
            )
            tags.append(tag)
        return tags

class PostSerializer(serializers.ModelSerializer):
    tags = TagField()
    # accepts: ["django", "python", "new tag"]
```

---

## 23. Depth — Automatic Nesting

`Meta.depth` tells `ModelSerializer` how many levels deep to automatically nest related objects. A quick way to get nested output without writing nested serializers.

```python
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['id', 'title', 'author', 'tags', 'comments']
        depth = 1   # automatically nest one level deep
```

Output: `author` becomes a nested object with all its fields; `tags` becomes a list of tag objects; `comments` becomes a list of comment objects.

```python
depth = 2  # goes two levels: post → author → profile (if author has a FK to profile)
```

### Why Depth Is Rarely Used in Production

- It automatically nests **all** FK fields — you can't pick and choose
- It makes all nested fields `read_only` — you can't write through them
- It exposes fields you may not want to expose
- You lose control over which serializer represents each nested object
- Performance: you get nested data but can't easily pair it with `select_related`

Use `depth` for quick prototyping. Use explicit nested serializers for production.

---

## 24. `source` Argument — Renaming and Traversing

The `source` argument tells the serializer field where to *actually* get its data from, independent of what the field is named in the API.

### Renaming a Field

```python
class PostSerializer(serializers.ModelSerializer):
    # API exposes "author_id", but internally it's "author"
    author_id = serializers.IntegerField(source='author.id', read_only=True)
    author_name = serializers.CharField(source='author.username', read_only=True)

    class Meta:
        model = Post
        fields = ['id', 'title', 'author_id', 'author_name']
```

Output:
```json
{"id": 1, "title": "Hello", "author_id": 3, "author_name": "alice"}
```

### Traversing Relations

```python
class PostSerializer(serializers.ModelSerializer):
    # Traverse FK → O2O in a single source expression
    author_bio = serializers.CharField(source='author.profile.bio', read_only=True)
    author_location = serializers.CharField(source='author.profile.location', read_only=True)

    class Meta:
        model = Post
        fields = ['id', 'title', 'author_bio', 'author_location']
```

### `source='*'` — Pass the Whole Object

When `source='*'`, the field receives the entire model instance. Used for custom fields that need access to multiple attributes:

```python
class PostLocationField(serializers.Field):
    def to_representation(self, value):
        # value is the entire Post instance
        return f"{value.author.profile.location} — {value.category.name}"

class PostSerializer(serializers.ModelSerializer):
    location_info = PostLocationField(source='*')
```

---

## 25. Handling Reverse Relations in Serializers

Reverse relations (from the "one" side of a FK) are not included in `ModelSerializer` by default — you have to declare them explicitly.

### Adding Reverse FK to a Serializer

```python
# A User serializer that includes all their posts

class PostMinimalSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['id', 'title', 'created_at']

class UserSerializer(serializers.ModelSerializer):
    # 'posts' matches the related_name on Post.author
    posts = PostMinimalSerializer(many=True, read_only=True)

    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'posts']
```

Output:
```json
{
    "id": 1,
    "username": "alice",
    "email": "alice@example.com",
    "posts": [
        {"id": 1, "title": "Hello", "created_at": "2024-01-01T..."},
        {"id": 2, "title": "World", "created_at": "2024-01-02T..."}
    ]
}
```

### When the `related_name` Doesn't Match the Field Name

```python
class UserSerializer(serializers.ModelSerializer):
    # The related_name on Post.author is 'posts'
    # But we want to call it 'authored_posts' in the API
    authored_posts = PostMinimalSerializer(
        many=True,
        read_only=True,
        source='posts'   # source points to the actual related_name
    )

    class Meta:
        model = User
        fields = ['id', 'username', 'authored_posts']
```

### Reverse OneToOne

```python
class UserSerializer(serializers.ModelSerializer):
    # Profile is a OneToOne with related_name='profile'
    profile = ProfileSerializer(read_only=True)

    class Meta:
        model = User
        fields = ['id', 'username', 'profile']
```

---

## 26. Serializer Context — Passing Request into Relations

Context is how you pass information (the `request`, the current user, flags) into a serializer. DRF's generic views pass the request automatically. When you manually instantiate a serializer, pass it yourself.

```python
# DRF does this automatically in generic views:
# serializer = self.get_serializer(instance)
# which calls: serializer_class(instance, context={'request': request, 'view': self, ...})

# Manual instantiation — you must pass context yourself:
serializer = PostSerializer(post, context={'request': request})
```

### Using Context in a Nested Serializer

Context is passed down automatically through nested serializers:

```python
class TagSerializer(serializers.ModelSerializer):
    is_followed = serializers.SerializerMethodField()

    def get_is_followed(self, obj):
        request = self.context.get('request')  # available even in nested serializer
        if not request or not request.user.is_authenticated:
            return False
        return request.user.profile.following_tags.filter(pk=obj.pk).exists()

    class Meta:
        model = Tag
        fields = ['id', 'name', 'is_followed']

class PostSerializer(serializers.ModelSerializer):
    tags = TagSerializer(many=True, read_only=True)
    # TagSerializer receives the same context dict as PostSerializer
```

### Dynamically Limiting Fields via Context

```python
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'internal_notes', 'published']

    def to_representation(self, instance):
        data = super().to_representation(instance)
        request = self.context.get('request')
        # Hide internal notes from non-staff users
        if not (request and request.user.is_staff):
            data.pop('internal_notes', None)
        return data
```

---

## 27. Performance in Serializers — `select_related` meets DRF

Nested serializers and `SerializerMethodField` that access relations will trigger N+1 queries unless you optimize the queryset in the view. This is the most common performance problem in DRF applications.

### The Problem

```python
class PostSerializer(serializers.ModelSerializer):
    author = UserSerializer(read_only=True)           # accesses post.author
    tags = TagSerializer(many=True, read_only=True)   # accesses post.tags.all()

    class Meta:
        model = Post
        fields = ['id', 'title', 'author', 'tags']

class PostViewSet(viewsets.ModelViewSet):
    serializer_class = PostSerializer
    queryset = Post.objects.all()  # ← problem: no prefetching
```

For 50 posts: 1 query for posts + 50 for authors + 50 for tags = **101 queries**.

### The Fix

```python
class PostViewSet(viewsets.ModelViewSet):
    serializer_class = PostSerializer
    queryset = Post.objects.select_related(
        'author',           # FK → use JOIN
        'author__profile',  # FK through FK → same JOIN
    ).prefetch_related(
        'tags',             # M2M → separate query
    )
```

Now: **3 queries total**, regardless of how many posts.

### Dynamic Optimization Based on Action

Detail views often serialize more relations than list views. Optimize accordingly:

```python
class PostViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        if self.action == 'list':
            return Post.objects.select_related('author').prefetch_related('tags')
        elif self.action == 'retrieve':
            return Post.objects.select_related(
                'author', 'author__profile', 'category'
            ).prefetch_related(
                'tags',
                Prefetch('comments', queryset=Comment.objects.select_related('author').order_by('-created_at'))
            )
        return Post.objects.all()
```

### Using Annotations Instead of Serializer Queries

```python
# Instead of SerializerMethodField that hits the DB per object:
def get_comment_count(self, obj):
    return obj.comments.count()  # N queries

# Annotate in the view and read the annotation in the serializer:
queryset = Post.objects.annotate(comment_count=Count('comments'))

# In serializer — reads from the annotated attribute:
comment_count = serializers.IntegerField(read_only=True)  # reads obj.comment_count
```

This moves the aggregation into SQL (1 query) instead of Python (N queries).

### `only()` and `defer()` — Column-Level Optimization

```python
# For list views, you might not need all columns
Post.objects.only('id', 'title', 'created_at').select_related('author')

# Defer expensive columns (large text fields)
Post.objects.defer('content').select_related('author')
```

Be careful: `only()` causes an extra query if you access a deferred field. Only use it when you're certain which fields you need.

---

## 28. Mental Model Cheat Sheet

### Django Relationship Decision Tree

```
Two models need to be related...
│
├── Each A has exactly one B, and each B has exactly one A?
│   → OneToOneField (on the model that "belongs to" the other)
│
├── Many A can belong to one B (each A has one B, each B has many A)?
│   → ForeignKey on A (the "many" side), pointing to B
│
├── Many A can relate to many B?
│   ├── Relation has no extra data → ManyToManyField
│   └── Relation has extra data (date, status, etc.) → through model
│
└── A relates to different model types (Post, Photo, Video)?
    → GenericForeignKey + ContentTypes
```

### ORM Query Reference

```python
# FK traversal in filter
Model.objects.filter(fk_field__related_field=value)

# Reverse FK traversal
Parent.objects.filter(child_related_name__field=value)

# Prevent N+1 for FK/O2O
.select_related('fk_field', 'fk1__fk2')

# Prevent N+1 for M2M/reverse FK
.prefetch_related('m2m_field', 'reverse_related_name')

# Custom prefetch
.prefetch_related(Prefetch('field', queryset=..., to_attr='attr_name'))

# Compute per-row aggregates
.annotate(count=Count('related_name'))

# Compute across whole queryset
.aggregate(total=Sum('field'))
```

### DRF Serializer Relations — When to Use What

| Need | Use |
|------|-----|
| Output just the FK id | `PrimaryKeyRelatedField` or default ModelSerializer |
| Accept FK id on write | `PrimaryKeyRelatedField(queryset=...)` |
| Output `__str__` of related | `StringRelatedField(read_only=True)` |
| Output/accept by natural key (slug, username) | `SlugRelatedField(slug_field=...)` |
| Output full URL (HATEOAS) | `HyperlinkedRelatedField(view_name=...)` |
| Output full nested object | Nested serializer with `read_only=True` |
| Accept nested objects on write | Nested serializer + custom `create()`/`update()` |
| Computed/conditional relation data | `SerializerMethodField` |
| Rename or traverse relations | `source` argument |
| Field changes based on GET vs POST | Read/Write serializer split + `get_serializer_class()` |

### The Golden Rules

1. **Set `related_name` on every FK and M2M.** Always. The default `_set` syntax is ambiguous and noisy.
2. **Never access a relation in a loop without prefetching.** That's N+1.
3. **`select_related` for FK/O2O. `prefetch_related` for M2M and reverse FK.**
4. **Use `post.author_id`, not `post.author.id`, when you only need the integer.**
5. **Pair nested serializers with `select_related`/`prefetch_related` in the view's queryset.**
6. **Use annotations for aggregated relation data — not `SerializerMethodField` with `.count()`.**
7. **Use the read/write serializer split when GET and POST shapes differ significantly.**
8. **`through` models when M2M relationships have data of their own.**
9. **`symmetrical=False` for directed self-referential M2M (followers, not friends).**
10. **Context flows into nested serializers automatically — use it for `request.user` checks.**
