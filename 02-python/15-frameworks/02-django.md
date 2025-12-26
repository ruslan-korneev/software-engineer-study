# Django

"Batteries included" — ORM, админка, аутентификация из коробки.

```bash
pip install django
django-admin startproject myproject
python manage.py runserver
```

---

## Структура

```
myproject/
├── manage.py
├── myproject/
│   ├── settings.py
│   └── urls.py
└── myapp/
    ├── models.py
    ├── views.py
    ├── urls.py
    └── admin.py
```

```bash
python manage.py startapp myapp
```

---

## Models (ORM)

```python
from django.db import models

class User(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField(unique=True)
    created_at = models.DateTimeField(auto_now_add=True)

class Post(models.Model):
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    title = models.CharField(max_length=200)
    content = models.TextField()
```

```bash
python manage.py makemigrations
python manage.py migrate
```

### Запросы

```python
User.objects.create(name="Анна", email="a@b.com")
User.objects.all()
User.objects.get(id=1)
User.objects.filter(name__contains="Анна")
user.posts.all()  # Related
```

---

## Views

```python
from django.http import JsonResponse
from django.shortcuts import get_object_or_404

def user_list(request):
    users = list(User.objects.values('id', 'name'))
    return JsonResponse(users, safe=False)

def user_detail(request, user_id):
    user = get_object_or_404(User, id=user_id)
    return JsonResponse({"id": user.id, "name": user.name})
```

---

## URLs

```python
# myproject/urls.py
from django.urls import path, include
urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('myapp.urls')),
]

# myapp/urls.py
urlpatterns = [
    path('users/', views.user_list),
    path('users/<int:user_id>/', views.user_detail),
]
```

---

## Admin

```python
from django.contrib import admin
from .models import User

@admin.register(User)
class UserAdmin(admin.ModelAdmin):
    list_display = ['name', 'email']
    search_fields = ['name']
```

```bash
python manage.py createsuperuser
```

---

## Django REST Framework

```bash
pip install djangorestframework
```

```python
from rest_framework import serializers, viewsets

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'name', 'email']

class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer
```

---

## Резюме

| Фича | Описание |
|------|----------|
| ORM | Мощный, миграции |
| Admin | Автоматическая админка |
| DRF | REST API |
| Auth | Аутентификация из коробки |

**Когда:** Полноценные веб-приложения, админки, CRUD.
