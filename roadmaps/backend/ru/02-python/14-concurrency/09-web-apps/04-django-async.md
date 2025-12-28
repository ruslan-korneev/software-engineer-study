# Асинхронные представления Django

[prev: 03-starlette.md](./03-starlette.md) | [next: ../10-microservices/readme.md](../10-microservices/readme.md)

---

## Введение

Начиная с версии 3.0, Django поддерживает асинхронное программирование. Версия 3.1 добавила асинхронные представления (views), а версии 4.1+ существенно расширили поддержку async в ORM и middleware. Это позволяет создавать высокопроизводительные приложения, эффективно обрабатывающие множество одновременных соединений.

## ASGI в Django

### Настройка ASGI

Django автоматически создает файл `asgi.py` при создании проекта:

```python
# myproject/asgi.py
import os
from django.core.asgi import get_asgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings')

application = get_asgi_application()
```

### Запуск с ASGI сервером

```bash
# Uvicorn
pip install uvicorn
uvicorn myproject.asgi:application --host 0.0.0.0 --port 8000

# Для продакшена с несколькими workers
uvicorn myproject.asgi:application --workers 4 --host 0.0.0.0 --port 8000

# Daphne (рекомендуется для Django Channels)
pip install daphne
daphne myproject.asgi:application -b 0.0.0.0 -p 8000

# Hypercorn
pip install hypercorn
hypercorn myproject.asgi:application --bind 0.0.0.0:8000
```

### WSGI vs ASGI режим

```python
# Проверка режима в коде
import asyncio

def is_async_context():
    try:
        asyncio.get_running_loop()
        return True
    except RuntimeError:
        return False
```

## Асинхронные представления (Views)

### Function-based views

```python
# views.py
from django.http import JsonResponse, HttpResponse
import asyncio
import httpx

# Асинхронное представление
async def async_view(request):
    """Простое асинхронное представление."""
    await asyncio.sleep(1)  # Имитация асинхронной операции
    return JsonResponse({'message': 'Hello from async view!'})

# Параллельные запросы к внешним API
async def fetch_multiple_apis(request):
    """Параллельные запросы к нескольким API."""
    async with httpx.AsyncClient() as client:
        # Параллельное выполнение
        tasks = [
            client.get('https://api.example.com/users'),
            client.get('https://api.example.com/products'),
            client.get('https://api.example.com/orders'),
        ]
        responses = await asyncio.gather(*tasks)

    return JsonResponse({
        'users': responses[0].json(),
        'products': responses[1].json(),
        'orders': responses[2].json(),
    })

# Синхронное представление (для сравнения)
def sync_view(request):
    """Синхронное представление."""
    import time
    time.sleep(1)  # Блокирует весь поток!
    return JsonResponse({'message': 'Hello from sync view!'})
```

### Class-based views

```python
from django.views import View
from django.http import JsonResponse
import asyncio

class AsyncView(View):
    """Асинхронный class-based view."""

    async def get(self, request, *args, **kwargs):
        data = await self.fetch_data()
        return JsonResponse(data)

    async def post(self, request, *args, **kwargs):
        # Обработка POST запроса
        import json
        body = json.loads(request.body)
        result = await self.process_data(body)
        return JsonResponse(result, status=201)

    async def fetch_data(self):
        await asyncio.sleep(0.1)
        return {'items': [1, 2, 3]}

    async def process_data(self, data):
        await asyncio.sleep(0.1)
        return {'processed': data}

# urls.py
from django.urls import path
from .views import AsyncView

urlpatterns = [
    path('async/', AsyncView.as_view(), name='async-view'),
]
```

### Миксин для асинхронных операций

```python
from django.views import View
from django.http import JsonResponse
from asgiref.sync import sync_to_async

class AsyncMixin:
    """Миксин с утилитами для async views."""

    @sync_to_async
    def get_object_sync(self, model, **kwargs):
        """Получение объекта из БД (обертка для sync ORM)."""
        return model.objects.get(**kwargs)

    @sync_to_async
    def filter_objects_sync(self, model, **kwargs):
        """Фильтрация объектов (обертка для sync ORM)."""
        return list(model.objects.filter(**kwargs))

class UserDetailView(AsyncMixin, View):
    async def get(self, request, user_id):
        from .models import User
        try:
            user = await self.get_object_sync(User, id=user_id)
            return JsonResponse({
                'id': user.id,
                'username': user.username,
                'email': user.email,
            })
        except User.DoesNotExist:
            return JsonResponse({'error': 'User not found'}, status=404)
```

## Асинхронный ORM (Django 4.1+)

### Основные асинхронные методы

```python
from django.http import JsonResponse
from .models import User, Article

async def async_orm_examples(request):
    """Примеры использования асинхронного ORM."""

    # Получение одного объекта
    user = await User.objects.aget(id=1)

    # Получение или создание
    user, created = await User.objects.aget_or_create(
        username='john',
        defaults={'email': 'john@example.com'}
    )

    # Обновление или создание
    user, created = await User.objects.aupdate_or_create(
        username='john',
        defaults={'email': 'john_new@example.com'}
    )

    # Проверка существования
    exists = await User.objects.filter(username='john').aexists()

    # Подсчет
    count = await User.objects.acount()

    # Первый/последний элемент
    first_user = await User.objects.afirst()
    last_user = await User.objects.alast()

    # Получение списка (итерация)
    users = [user async for user in User.objects.all()]

    # Фильтрация с итерацией
    active_users = [
        user async for user in User.objects.filter(is_active=True)
    ]

    return JsonResponse({'count': count})
```

### Асинхронная итерация по QuerySet

```python
async def list_users_async(request):
    """Асинхронная итерация по QuerySet."""
    from .models import User

    users_data = []

    # Асинхронная итерация
    async for user in User.objects.filter(is_active=True):
        users_data.append({
            'id': user.id,
            'username': user.username,
            'email': user.email,
        })

    return JsonResponse({'users': users_data})

async def paginated_users(request):
    """Пагинация с асинхронным ORM."""
    page = int(request.GET.get('page', 1))
    per_page = int(request.GET.get('per_page', 10))

    offset = (page - 1) * per_page

    users = []
    async for user in User.objects.all()[offset:offset + per_page]:
        users.append({
            'id': user.id,
            'username': user.username,
        })

    total = await User.objects.acount()

    return JsonResponse({
        'users': users,
        'page': page,
        'per_page': per_page,
        'total': total,
    })
```

### Асинхронные операции создания/обновления

```python
async def create_user_async(request):
    """Асинхронное создание объекта."""
    import json
    from .models import User

    data = json.loads(request.body)

    # Создание
    user = await User.objects.acreate(
        username=data['username'],
        email=data['email'],
    )

    return JsonResponse({
        'id': user.id,
        'username': user.username,
    }, status=201)

async def update_user_async(request, user_id):
    """Асинхронное обновление объекта."""
    import json
    from .models import User

    data = json.loads(request.body)

    # Массовое обновление
    updated_count = await User.objects.filter(id=user_id).aupdate(
        email=data.get('email')
    )

    if updated_count == 0:
        return JsonResponse({'error': 'User not found'}, status=404)

    user = await User.objects.aget(id=user_id)
    return JsonResponse({
        'id': user.id,
        'email': user.email,
    })

async def delete_user_async(request, user_id):
    """Асинхронное удаление объекта."""
    from .models import User

    deleted_count, _ = await User.objects.filter(id=user_id).adelete()

    if deleted_count == 0:
        return JsonResponse({'error': 'User not found'}, status=404)

    return JsonResponse({'deleted': True})
```

## sync_to_async и async_to_sync

### Обертка синхронного кода

```python
from asgiref.sync import sync_to_async, async_to_sync
from django.contrib.auth import authenticate

# Обертка синхронной функции для использования в async коде
@sync_to_async
def authenticate_user(username, password):
    """Синхронная аутентификация, обернутая для async."""
    return authenticate(username=username, password=password)

async def login_view(request):
    """Асинхронное представление с синхронной аутентификацией."""
    import json
    data = json.loads(request.body)

    user = await authenticate_user(
        data['username'],
        data['password']
    )

    if user:
        return JsonResponse({'authenticated': True, 'user_id': user.id})
    return JsonResponse({'authenticated': False}, status=401)

# Вызов асинхронной функции из синхронного кода
async def fetch_external_data():
    import httpx
    async with httpx.AsyncClient() as client:
        response = await client.get('https://api.example.com/data')
        return response.json()

def sync_view_with_async_call(request):
    """Синхронное представление, вызывающее async функцию."""
    data = async_to_sync(fetch_external_data)()
    return JsonResponse(data)
```

### Thread-sensitive обертка

```python
from asgiref.sync import sync_to_async

# По умолчанию sync_to_async запускает код в потоке из пула
@sync_to_async
def default_sync_function():
    # Выполняется в потоке из пула потоков
    pass

# thread_sensitive=True - код выполняется в главном потоке
@sync_to_async(thread_sensitive=True)
def thread_sensitive_function():
    # Выполняется в главном потоке
    # Необходимо для некоторых операций Django (например, работа с сессиями)
    pass

# Пример с доступом к сессии
@sync_to_async(thread_sensitive=True)
def get_session_data(request):
    return dict(request.session)

async def async_view_with_session(request):
    session_data = await get_session_data(request)
    return JsonResponse(session_data)
```

## Асинхронные Middleware

### Создание async middleware

```python
# middleware.py
import time
import logging

logger = logging.getLogger(__name__)

class AsyncTimingMiddleware:
    """Асинхронный middleware для измерения времени запроса."""

    def __init__(self, get_response):
        self.get_response = get_response
        # Проверяем, является ли get_response корутиной
        self.async_mode = asyncio.iscoroutinefunction(get_response)

    async def __call__(self, request):
        start_time = time.time()

        # Вызов следующего middleware или view
        response = await self.get_response(request)

        duration = time.time() - start_time
        logger.info(f"{request.method} {request.path} - {duration:.3f}s")
        response['X-Request-Duration'] = str(duration)

        return response

class AsyncAuthMiddleware:
    """Асинхронный middleware для аутентификации."""

    def __init__(self, get_response):
        self.get_response = get_response

    async def __call__(self, request):
        # Пропускаем публичные URL
        public_paths = ['/api/login/', '/api/register/', '/health/']
        if request.path in public_paths:
            return await self.get_response(request)

        # Проверка токена
        auth_header = request.headers.get('Authorization', '')
        if not auth_header.startswith('Bearer '):
            from django.http import JsonResponse
            return JsonResponse({'error': 'Unauthorized'}, status=401)

        token = auth_header[7:]
        user = await self.validate_token(token)

        if not user:
            from django.http import JsonResponse
            return JsonResponse({'error': 'Invalid token'}, status=401)

        request.user = user
        return await self.get_response(request)

    async def validate_token(self, token):
        """Асинхронная валидация токена."""
        from .models import User
        # Здесь должна быть реальная логика валидации
        try:
            return await User.objects.aget(auth_token=token)
        except User.DoesNotExist:
            return None
```

### Регистрация middleware

```python
# settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'myapp.middleware.AsyncTimingMiddleware',  # Кастомный async middleware
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'myapp.middleware.AsyncAuthMiddleware',  # Кастомный async middleware
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
]
```

## Django Channels для WebSocket

### Установка и настройка

```bash
pip install channels channels-redis
```

```python
# settings.py
INSTALLED_APPS = [
    'channels',
    # ... другие приложения
]

ASGI_APPLICATION = 'myproject.asgi.application'

# Настройка channel layers (для broadcast между workers)
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

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings')

# Инициализация Django до импорта роутов
django_asgi_app = get_asgi_application()

from myapp.routing import websocket_urlpatterns

application = ProtocolTypeRouter({
    'http': django_asgi_app,
    'websocket': AuthMiddlewareStack(
        URLRouter(websocket_urlpatterns)
    ),
})
```

### WebSocket Consumer

```python
# myapp/consumers.py
import json
from channels.generic.websocket import AsyncWebsocketConsumer

class ChatConsumer(AsyncWebsocketConsumer):
    """WebSocket consumer для чата."""

    async def connect(self):
        """Обработка подключения."""
        self.room_name = self.scope['url_route']['kwargs']['room_name']
        self.room_group_name = f'chat_{self.room_name}'

        # Присоединение к группе
        await self.channel_layer.group_add(
            self.room_group_name,
            self.channel_name
        )

        await self.accept()

        # Уведомление о новом пользователе
        await self.channel_layer.group_send(
            self.room_group_name,
            {
                'type': 'user_join',
                'username': self.scope['user'].username,
            }
        )

    async def disconnect(self, close_code):
        """Обработка отключения."""
        await self.channel_layer.group_discard(
            self.room_group_name,
            self.channel_name
        )

    async def receive(self, text_data):
        """Получение сообщения от клиента."""
        data = json.loads(text_data)
        message = data['message']

        # Отправка сообщения в группу
        await self.channel_layer.group_send(
            self.room_group_name,
            {
                'type': 'chat_message',
                'message': message,
                'username': self.scope['user'].username,
            }
        )

    async def chat_message(self, event):
        """Обработчик сообщения чата."""
        await self.send(text_data=json.dumps({
            'type': 'message',
            'message': event['message'],
            'username': event['username'],
        }))

    async def user_join(self, event):
        """Обработчик присоединения пользователя."""
        await self.send(text_data=json.dumps({
            'type': 'user_join',
            'username': event['username'],
        }))

# myapp/routing.py
from django.urls import re_path
from . import consumers

websocket_urlpatterns = [
    re_path(r'ws/chat/(?P<room_name>\w+)/$', consumers.ChatConsumer.as_asgi()),
]
```

### Отправка сообщений из view в WebSocket

```python
from channels.layers import get_channel_layer
from asgiref.sync import async_to_sync

# Из синхронного кода
def send_notification_sync(room_name, message):
    channel_layer = get_channel_layer()
    async_to_sync(channel_layer.group_send)(
        f'chat_{room_name}',
        {
            'type': 'chat_message',
            'message': message,
            'username': 'system',
        }
    )

# Из асинхронного кода
async def send_notification_async(room_name, message):
    channel_layer = get_channel_layer()
    await channel_layer.group_send(
        f'chat_{room_name}',
        {
            'type': 'chat_message',
            'message': message,
            'username': 'system',
        }
    )
```

## Тестирование асинхронного кода

### Тестирование async views

```python
import pytest
from django.test import AsyncClient
from django.urls import reverse

@pytest.mark.asyncio
@pytest.mark.django_db
async def test_async_view():
    """Тест асинхронного представления."""
    client = AsyncClient()

    response = await client.get('/api/async/')

    assert response.status_code == 200
    data = response.json()
    assert 'message' in data

@pytest.mark.asyncio
@pytest.mark.django_db
async def test_async_create_user():
    """Тест создания пользователя через async view."""
    client = AsyncClient()

    response = await client.post(
        '/api/users/',
        data={'username': 'testuser', 'email': 'test@example.com'},
        content_type='application/json'
    )

    assert response.status_code == 201
    data = response.json()
    assert data['username'] == 'testuser'
```

### Тестирование WebSocket с Channels

```python
import pytest
from channels.testing import WebsocketCommunicator
from myapp.consumers import ChatConsumer

@pytest.mark.asyncio
async def test_chat_consumer():
    """Тест WebSocket consumer."""
    communicator = WebsocketCommunicator(
        ChatConsumer.as_asgi(),
        '/ws/chat/testroom/'
    )

    # Подключение
    connected, _ = await communicator.connect()
    assert connected

    # Отправка сообщения
    await communicator.send_json_to({
        'message': 'Hello, World!'
    })

    # Получение ответа
    response = await communicator.receive_json_from()
    assert response['message'] == 'Hello, World!'

    # Закрытие
    await communicator.disconnect()
```

## Best Practices

### 1. Избегайте блокирующих операций

```python
# Плохо - блокирует event loop
async def bad_view(request):
    import time
    time.sleep(1)  # Блокирует!
    return JsonResponse({'status': 'done'})

# Хорошо - используйте asyncio.sleep или sync_to_async
async def good_view(request):
    await asyncio.sleep(1)
    return JsonResponse({'status': 'done'})

# Или оберните синхронный код
@sync_to_async
def blocking_operation():
    import time
    time.sleep(1)
    return 'result'

async def better_view(request):
    result = await blocking_operation()
    return JsonResponse({'result': result})
```

### 2. Используйте connection pooling

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'CONN_MAX_AGE': 60,  # Поддержание соединений
        'CONN_HEALTH_CHECKS': True,  # Django 4.1+
    }
}
```

### 3. Правильная обработка ошибок

```python
import logging
from django.http import JsonResponse

logger = logging.getLogger(__name__)

async def robust_async_view(request):
    try:
        result = await some_async_operation()
        return JsonResponse({'data': result})
    except SomeExpectedException as e:
        logger.warning(f"Expected error: {e}")
        return JsonResponse({'error': str(e)}, status=400)
    except Exception as e:
        logger.exception("Unexpected error in async view")
        return JsonResponse({'error': 'Internal server error'}, status=500)
```

## Частые ошибки

1. **Смешивание sync и async без обертки** - используйте `sync_to_async`

2. **Блокирующий код в async view** - весь I/O должен быть асинхронным

3. **Неправильное использование ORM** - в Django < 4.1 ORM синхронный, нужна обертка

4. **Забыть про thread_sensitive** - для некоторых операций Django нужен главный поток

5. **Игнорирование совместимости middleware** - проверяйте, что middleware поддерживает async

---

[prev: 03-starlette.md](./03-starlette.md) | [next: ../10-microservices/readme.md](../10-microservices/readme.md)
