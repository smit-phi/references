## Django Signals — Complete Guide

### What Are Signals?

Signals are Django's **observer/event pattern** implementation. They let one part of your application **notify** other parts when something happens — without those parts being directly coupled to each other.

```
Something happens          Django dispatches          Receivers react
(model saved, user  ──▶   a Signal with      ──▶    (send email, create
 logged in, etc.)          sender + kwargs            profile, log audit...)
```

Think of it as a **pub/sub system** built into Django — the sender doesn't need to know who's listening, and the receiver doesn't need to be in the same file or even the same app.

---

### Why Do They Exist? (The Core Need)

Without signals, cross-app communication forces tight coupling:

```python
# ❌ Without signals — BlogPost directly knows about 3 other systems
class BlogPost(models.Model):
    def save(self, *args, **kwargs):
        super().save(*args, **kwargs)
        NotificationService.notify_subscribers(self)   # coupled
        SearchIndex.update_index(self)                 # coupled
        AuditLog.record_change(self)                   # coupled
```

```python
# ✅ With signals — BlogPost knows nothing about other systems
class BlogPost(models.Model):
    pass   # clean model

# Each concern lives in its own app, listens independently
@receiver(post_save, sender=BlogPost)
def notify_subscribers(sender, instance, created, **kwargs): ...

@receiver(post_save, sender=BlogPost)
def update_search_index(sender, instance, **kwargs): ...

@receiver(post_save, sender=BlogPost)
def log_audit_trail(sender, instance, **kwargs): ...
```

---

### The Three Parts of Every Signal

```
┌─────────────┐       ┌──────────────┐       ┌──────────────┐
│   SIGNAL    │       │    SENDER    │       │   RECEIVER   │
│             │       │              │       │              │
│ The event   │  +    │ Who fired it │  +    │ What to do   │
│ definition  │       │ (a model,    │       │ when it fires │
│             │       │  a class...) │       │              │
└─────────────┘       └──────────────┘       └──────────────┘
     post_save              BlogPost            send_email()
```

---

## Built-in Django Signals

### Model Signals (`django.db.models.signals`)

```python
from django.db.models.signals import (
    pre_save,       # fires BEFORE model.save()
    post_save,      # fires AFTER model.save()
    pre_delete,     # fires BEFORE model.delete()
    post_delete,    # fires AFTER model.delete()
    pre_init,       # fires BEFORE model.__init__()
    post_init,      # fires AFTER model.__init__()
    m2m_changed,    # fires when ManyToMany field changes
)
```

### Request / Auth Signals (`django.contrib.auth.signals`)

```python
from django.contrib.auth.signals import (
    user_logged_in,     # after successful login
    user_logged_out,    # after logout
    user_login_failed,  # after failed login attempt
)
```

### Management Signals (`django.db.models.signals`)

```python
from django.db.models.signals import post_migrate   # after migrate command runs
from django.test.signals import setting_changed     # when settings change in tests
```

---

## Signal Arguments — What You Actually Receive

### `post_save` kwargs

```python
@receiver(post_save, sender=MyModel)
def handle(sender, instance, created, raw, using, update_fields, **kwargs):
    # sender        → the model class itself (MyModel)
    # instance      → the actual saved object
    # created       → True if INSERT, False if UPDATE
    # raw           → True if loaded via fixture (loaddata)
    # using         → the DB alias used ('default', 'replica', etc.)
    # update_fields → set of fields passed to .save(update_fields=[...]), or None
    pass
```

### `pre_save` kwargs

```python
@receiver(pre_save, sender=MyModel)
def handle(sender, instance, raw, using, update_fields, **kwargs):
    # Same as post_save but no `created` — record doesn't exist yet
    # instance.pk is None if it's a new object
    pass
```

### `post_delete` kwargs

```python
@receiver(post_delete, sender=MyModel)
def handle(sender, instance, using, **kwargs):
    # instance still has its data, but pk may be set to None
    # use instance.pk carefully here
    pass
```

### `m2m_changed` kwargs

```python
@receiver(m2m_changed, sender=Article.tags.through)
def handle(sender, instance, action, reverse, model, pk_set, using, **kwargs):
    # action → 'pre_add', 'post_add', 'pre_remove', 'post_remove',
    #           'pre_clear', 'post_clear'
    # pk_set → set of PKs being added/removed (None for clear)
    # model  → the model on the other side of the M2M
    pass
```

### Auth Signal kwargs

```python
@receiver(user_logged_in)
def handle(sender, request, user, **kwargs):
    # request → the HttpRequest
    # user    → the User instance

@receiver(user_login_failed)
def handle(sender, credentials, request, **kwargs):
    # credentials → dict with 'username' (password is stripped for security)
    pass
```

---

## How to Connect Signals — 3 Ways

### Method 1: `@receiver` Decorator (most common)

```python
# accounts/signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.contrib.auth import get_user_model

User = get_user_model()

@receiver(post_save, sender=User)
def create_user_profile(sender, instance, created, **kwargs):
    if created:
        Profile.objects.create(user=instance)
```

### Method 2: `signal.connect()` (explicit, good for dynamic wiring)

```python
from django.db.models.signals import post_save

def create_user_profile(sender, **kwargs):
    ...

# Connect manually
post_save.connect(create_user_profile, sender=User)

# Disconnect when needed
post_save.disconnect(create_user_profile, sender=User)
```

### Method 3: `dispatch_uid` (prevents duplicate receivers)

```python
# If your app is imported multiple times (e.g. in tests),
# the receiver can get registered twice — dispatch_uid prevents that
@receiver(post_save, sender=User, dispatch_uid="accounts.create_user_profile")
def create_user_profile(sender, instance, created, **kwargs):
    if created:
        Profile.objects.create(user=instance)
```

---

## Where to Put Signals — The Right Structure

### The `AppConfig.ready()` Pattern (Django's official recommendation)

```
myapp/
  ├── __init__.py
  ├── apps.py         ← import signals here
  ├── models.py
  └── signals.py      ← define all receivers here
```

```python
# myapp/apps.py
from django.apps import AppConfig

class MyAppConfig(AppConfig):
    name = 'myapp'

    def ready(self):
        import myapp.signals   # ← registers all receivers when app loads
```

```python
# myapp/__init__.py
default_app_config = 'myapp.apps.MyAppConfig'
```

```python
# myapp/signals.py
from django.db.models.signals import post_save, post_delete
from django.dispatch import receiver
from .models import Order, Product

@receiver(post_save, sender=Order)
def on_order_created(sender, instance, created, **kwargs):
    if created:
        instance.send_confirmation_email()

@receiver(post_delete, sender=Product)
def on_product_deleted(sender, instance, **kwargs):
    instance.cleanup_media_files()
```

> ⚠️ Never import signals in `models.py` or at module level — that causes circular imports and double registration.

---

## Custom Signals

You can define your own signals for events Django doesn't cover:

```python
# myapp/signals.py
from django.dispatch import Signal

# Define custom signals
order_placed = Signal()           # will pass: order, user
payment_confirmed = Signal()      # will pass: payment, amount
inventory_low = Signal()          # will pass: product, quantity

# --- Fire them from your business logic ---
# myapp/services.py
from .signals import order_placed, payment_confirmed

class OrderService:
    def place_order(self, cart, user):
        order = Order.objects.create(user=user, ...)

        # Fire the signal — any listener will react
        order_placed.send(sender=self.__class__, order=order, user=user)

        return order

    def confirm_payment(self, payment):
        payment.status = 'confirmed'
        payment.save()

        payment_confirmed.send(
            sender=self.__class__,
            payment=payment,
            amount=payment.amount
        )

# --- Listen to them from any app ---
# notifications/signals.py
from django.dispatch import receiver
from orders.signals import order_placed, payment_confirmed

@receiver(order_placed)
def send_order_email(sender, order, user, **kwargs):
    EmailService.send(user.email, template='order_placed', context={'order': order})

@receiver(payment_confirmed)
def update_inventory(sender, payment, **kwargs):
    for item in payment.order.items.all():
        item.product.reduce_stock(item.quantity)
```

---

## Real-World Use Cases

### 1. Auto-create Profile on User creation

```python
@receiver(post_save, sender=User)
def create_profile(sender, instance, created, **kwargs):
    if created:
        Profile.objects.create(user=instance)

@receiver(post_save, sender=User)
def save_profile(sender, instance, **kwargs):
    instance.profile.save()
```

### 2. Audit Logging

```python
from django.db.models.signals import post_save, post_delete

@receiver(post_save, sender=Invoice)
def log_invoice_change(sender, instance, created, **kwargs):
    AuditLog.objects.create(
        model='Invoice',
        object_id=instance.pk,
        action='CREATE' if created else 'UPDATE',
        changed_by=get_current_user(),     # needs django-crum or similar
    )

@receiver(post_delete, sender=Invoice)
def log_invoice_delete(sender, instance, **kwargs):
    AuditLog.objects.create(
        model='Invoice',
        object_id=instance.pk,
        action='DELETE',
    )
```

### 3. Login Tracking & Security Alerts

```python
from django.contrib.auth.signals import user_logged_in, user_login_failed
from django.utils import timezone

@receiver(user_logged_in)
def track_login(sender, request, user, **kwargs):
    UserLoginHistory.objects.create(
        user=user,
        ip_address=request.META.get('REMOTE_ADDR'),
        user_agent=request.META.get('HTTP_USER_AGENT', ''),
        timestamp=timezone.now(),
    )

@receiver(user_login_failed)
def alert_on_failed_login(sender, credentials, request, **kwargs):
    username = credentials.get('username', '')
    ip = request.META.get('REMOTE_ADDR')
    FailedLoginAttempt.objects.create(username=username, ip=ip)

    # Alert if too many failures
    recent_failures = FailedLoginAttempt.objects.filter(
        ip=ip,
        timestamp__gte=timezone.now() - timedelta(minutes=15)
    ).count()
    if recent_failures >= 5:
        SecurityAlert.send(ip=ip, reason='brute_force_attempt')
```

### 4. Cache Invalidation

```python
from django.core.cache import cache

@receiver(post_save, sender=Article)
@receiver(post_delete, sender=Article)
def invalidate_article_cache(sender, instance, **kwargs):
    cache.delete(f'article_{instance.pk}')
    cache.delete('article_list')
    cache.delete(f'author_{instance.author_id}_articles')
```

### 5. File Cleanup on Delete

```python
import os
from django.db.models.signals import post_delete, pre_save

@receiver(post_delete, sender=Product)
def delete_product_image(sender, instance, **kwargs):
    if instance.image:
        if os.path.isfile(instance.image.path):
            os.remove(instance.image.path)

@receiver(pre_save, sender=Product)
def delete_old_image_on_update(sender, instance, **kwargs):
    if not instance.pk:
        return   # new object, skip
    try:
        old = Product.objects.get(pk=instance.pk)
    except Product.DoesNotExist:
        return
    if old.image and old.image != instance.image:
        if os.path.isfile(old.image.path):
            os.remove(old.image.path)
```

---

## Critical Things to Keep in Mind

### 🔴 Pitfalls

```python
# ❌ 1. Infinite loops — post_save calling save() again
@receiver(post_save, sender=Order)
def update_total(sender, instance, **kwargs):
    instance.total = calculate_total(instance)
    instance.save()   # ← triggers post_save again → infinite loop!

# ✅ Fix: use update_fields to avoid re-triggering
@receiver(post_save, sender=Order)
def update_total(sender, instance, **kwargs):
    if 'total' not in (kwargs.get('update_fields') or []):
        Order.objects.filter(pk=instance.pk).update(
            total=calculate_total(instance)
        )
        # OR:
        instance.total = calculate_total(instance)
        instance.save(update_fields=['total'])   # won't re-trigger this check
```

```python
# ❌ 2. Signals don't fire on bulk operations
Order.objects.filter(status='pending').update(status='cancelled')  # no signal!
Order.objects.all().delete()                                        # no signal!

# ✅ Fix: loop individually (costly) or handle in service layer
for order in Order.objects.filter(status='pending'):
    order.status = 'cancelled'
    order.save()   # signals fire
```

```python
# ❌ 3. Heavy work inside signals blocks the request
@receiver(post_save, sender=Order)
def send_email(sender, instance, created, **kwargs):
    if created:
        send_confirmation_email(instance)   # blocks for 2-3 seconds!

# ✅ Fix: offload to Celery
@receiver(post_save, sender=Order)
def send_email(sender, instance, created, **kwargs):
    if created:
        send_confirmation_email_task.delay(instance.pk)   # async
```

### 🟡 Design Considerations

```
✅ Use signals for:
   → Cross-app communication (app A reacts to app B's events)
   → Audit logging, cache invalidation, cleanup tasks
   → Things that are truly "side effects" of another action
   → When you can't/shouldn't modify the sender's code

❌ Don't use signals for:
   → Logic within the SAME app (use model methods or service layer)
   → Complex business logic that needs to be tested in isolation
   → When execution order matters and must be guaranteed
   → Replacing a simple method call in the same class
```

### 🔵 Testing Signals

```python
# Disconnect signals in tests to isolate behavior
from django.test import TestCase
from django.db.models.signals import post_save
from myapp.signals import create_profile

class UserTests(TestCase):
    def setUp(self):
        post_save.disconnect(create_profile, sender=User)

    def tearDown(self):
        post_save.connect(create_profile, sender=User)

    def test_user_creation(self):
        user = User.objects.create_user('test', 'test@test.com', 'pass')
        # profile signal won't fire — isolated test

# Or use a context manager
from unittest.mock import patch

with patch('myapp.signals.create_profile') as mock_signal:
    User.objects.create_user(...)
    mock_signal.assert_called_once()
```

---

## Mental Model Summary

```
Signal = a named moment in time ("something just happened")
Sender  = who caused it         (a model, a class)
Receiver = who cares about it   (any function, in any app)

Django's built-in signals cover the model lifecycle.
Custom signals cover YOUR application's lifecycle.

Use them to decouple apps.
Avoid them for in-app logic — that belongs in methods or services.
```

Signals are one of Django's most powerful decoupling tools — the key is knowing when they're the right tool and when a plain method call is simpler and clearer.