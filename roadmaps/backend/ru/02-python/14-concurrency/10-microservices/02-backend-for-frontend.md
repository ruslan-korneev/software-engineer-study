# Паттерн backend-for-frontend

[prev: 01-why-microservices.md](./01-why-microservices.md) | [next: 03-product-api.md](./03-product-api.md)

---

## Введение

**Backend for Frontend (BFF)** — это архитектурный паттерн, при котором для каждого типа клиента (веб, мобильное приложение, IoT-устройства) создаётся отдельный backend-сервис. Этот сервис агрегирует данные из нескольких микросервисов и адаптирует их под потребности конкретного клиента.

## Проблема, которую решает BFF

### Без BFF: Общий API для всех клиентов

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│   Web    │  │  Mobile  │  │   IoT    │
│  Client  │  │   App    │  │  Device  │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │              │
     └─────────────┼──────────────┘
                   │
           ┌───────▼───────┐
           │  General API   │  <- Один API для всех
           └───────┬───────┘
                   │
     ┌─────────────┼─────────────┐
     │             │             │
┌────▼────┐  ┌─────▼────┐  ┌─────▼────┐
│  Users  │  │  Orders  │  │ Products │
└─────────┘  └──────────┘  └──────────┘
```

**Проблемы:**
- Мобильному приложению не нужны все поля, которые нужны вебу
- IoT-устройству нужен минимальный payload
- Разные версии API для разных клиентов сложно поддерживать

### С BFF: Специализированные API

```
┌──────────┐       ┌──────────┐       ┌──────────┐
│   Web    │       │  Mobile  │       │   IoT    │
│  Client  │       │   App    │       │  Device  │
└────┬─────┘       └────┬─────┘       └────┬─────┘
     │                  │                   │
┌────▼─────┐      ┌─────▼─────┐      ┌──────▼─────┐
│  Web BFF │      │Mobile BFF │      │  IoT BFF   │
└────┬─────┘      └─────┬─────┘      └──────┬─────┘
     │                  │                   │
     └──────────────────┼───────────────────┘
                        │
          ┌─────────────┼─────────────┐
          │             │             │
     ┌────▼────┐  ┌─────▼────┐  ┌─────▼────┐
     │  Users  │  │  Orders  │  │ Products │
     └─────────┘  └──────────┘  └──────────┘
```

## Реализация BFF на Python с asyncio

### Базовая структура проекта

```
project/
├── services/
│   ├── user_service/
│   ├── product_service/
│   └── order_service/
├── bff/
│   ├── web_bff/
│   │   └── main.py
│   ├── mobile_bff/
│   │   └── main.py
│   └── common/
│       ├── clients.py
│       └── models.py
└── docker-compose.yml
```

### Общие клиенты для микросервисов

```python
# bff/common/clients.py
import aiohttp
import asyncio
from typing import Optional, Dict, Any
from dataclasses import dataclass
from contextlib import asynccontextmanager


@dataclass
class ServiceConfig:
    host: str
    port: int
    timeout: int = 10

    @property
    def base_url(self) -> str:
        return f"http://{self.host}:{self.port}"


class BaseServiceClient:
    """Базовый клиент для взаимодействия с микросервисами"""

    def __init__(self, config: ServiceConfig):
        self.config = config
        self.timeout = aiohttp.ClientTimeout(total=config.timeout)
        self._session: Optional[aiohttp.ClientSession] = None

    async def _ensure_session(self) -> aiohttp.ClientSession:
        if self._session is None or self._session.closed:
            self._session = aiohttp.ClientSession(
                base_url=self.config.base_url,
                timeout=self.timeout
            )
        return self._session

    async def get(self, path: str, params: Optional[Dict] = None) -> Dict[str, Any]:
        session = await self._ensure_session()
        async with session.get(path, params=params) as response:
            if response.status == 404:
                return None
            response.raise_for_status()
            return await response.json()

    async def post(self, path: str, data: Dict[str, Any]) -> Dict[str, Any]:
        session = await self._ensure_session()
        async with session.post(path, json=data) as response:
            response.raise_for_status()
            return await response.json()

    async def close(self):
        if self._session and not self._session.closed:
            await self._session.close()


class UserServiceClient(BaseServiceClient):
    """Клиент для User Service"""

    async def get_user(self, user_id: int) -> Optional[Dict]:
        return await self.get(f'/users/{user_id}')

    async def get_users_batch(self, user_ids: list[int]) -> list[Dict]:
        """Получение нескольких пользователей одним запросом"""
        return await self.post('/users/batch', {'ids': user_ids})


class ProductServiceClient(BaseServiceClient):
    """Клиент для Product Service"""

    async def get_product(self, product_id: int) -> Optional[Dict]:
        return await self.get(f'/products/{product_id}')

    async def get_products(
        self,
        category: Optional[str] = None,
        limit: int = 20,
        offset: int = 0
    ) -> list[Dict]:
        params = {'limit': limit, 'offset': offset}
        if category:
            params['category'] = category
        return await self.get('/products', params=params)


class OrderServiceClient(BaseServiceClient):
    """Клиент для Order Service"""

    async def get_user_orders(self, user_id: int) -> list[Dict]:
        return await self.get(f'/orders/user/{user_id}')

    async def get_order(self, order_id: int) -> Optional[Dict]:
        return await self.get(f'/orders/{order_id}')
```

### Web BFF — полнофункциональный API

```python
# bff/web_bff/main.py
import asyncio
from aiohttp import web
from typing import Dict, Any, Optional
import os

# Импорты из common
from ..common.clients import (
    ServiceConfig,
    UserServiceClient,
    ProductServiceClient,
    OrderServiceClient
)


class WebBFF:
    """
    BFF для веб-клиента.

    Особенности:
    - Полные данные с детальной информацией
    - Поддержка пагинации
    - Расширенные фильтры
    - Агрегация данных из нескольких сервисов
    """

    def __init__(self):
        # Инициализация клиентов сервисов
        self.user_client = UserServiceClient(
            ServiceConfig(
                host=os.getenv('USER_SERVICE_HOST', 'user-service'),
                port=int(os.getenv('USER_SERVICE_PORT', 8001))
            )
        )
        self.product_client = ProductServiceClient(
            ServiceConfig(
                host=os.getenv('PRODUCT_SERVICE_HOST', 'product-service'),
                port=int(os.getenv('PRODUCT_SERVICE_PORT', 8002))
            )
        )
        self.order_client = OrderServiceClient(
            ServiceConfig(
                host=os.getenv('ORDER_SERVICE_HOST', 'order-service'),
                port=int(os.getenv('ORDER_SERVICE_PORT', 8003))
            )
        )

    async def get_user_dashboard(self, request: web.Request) -> web.Response:
        """
        Агрегированный dashboard пользователя.

        Объединяет:
        - Информацию о пользователе
        - Последние заказы
        - Рекомендуемые товары
        """
        user_id = int(request.match_info['user_id'])

        # Параллельные запросы к разным сервисам
        user_task = asyncio.create_task(
            self.user_client.get_user(user_id)
        )
        orders_task = asyncio.create_task(
            self.order_client.get_user_orders(user_id)
        )
        products_task = asyncio.create_task(
            self.product_client.get_products(limit=5)
        )

        # Ожидаем все результаты
        user, orders, recommended = await asyncio.gather(
            user_task, orders_task, products_task,
            return_exceptions=True
        )

        # Обработка ошибок
        if isinstance(user, Exception) or user is None:
            return web.json_response(
                {'error': 'User not found'},
                status=404
            )

        # Формируем ответ для веб-клиента (полные данные)
        dashboard = {
            'user': {
                'id': user['id'],
                'name': user['name'],
                'email': user['email'],
                'avatar': user.get('avatar'),
                'created_at': user.get('created_at'),
                'preferences': user.get('preferences', {})
            },
            'orders': {
                'recent': orders[:5] if not isinstance(orders, Exception) else [],
                'total_count': len(orders) if not isinstance(orders, Exception) else 0,
                'has_more': len(orders) > 5 if not isinstance(orders, Exception) else False
            },
            'recommendations': recommended if not isinstance(recommended, Exception) else []
        }

        return web.json_response(dashboard)

    async def get_product_details(self, request: web.Request) -> web.Response:
        """
        Детальная информация о товаре для веба.
        Включает все поля, изображения, отзывы.
        """
        product_id = int(request.match_info['product_id'])

        product = await self.product_client.get_product(product_id)

        if not product:
            return web.json_response(
                {'error': 'Product not found'},
                status=404
            )

        # Веб-версия включает все данные
        return web.json_response({
            'id': product['id'],
            'name': product['name'],
            'description': product['description'],
            'full_description': product.get('full_description', ''),
            'price': product['price'],
            'currency': product.get('currency', 'RUB'),
            'images': product.get('images', []),
            'thumbnails': product.get('thumbnails', []),
            'specifications': product.get('specifications', {}),
            'reviews': product.get('reviews', []),
            'rating': product.get('rating'),
            'stock': product.get('stock', 0),
            'category': product.get('category'),
            'related_products': product.get('related', [])
        })

    async def get_order_with_details(self, request: web.Request) -> web.Response:
        """
        Заказ с полной информацией о товарах и пользователе.
        """
        order_id = int(request.match_info['order_id'])

        order = await self.order_client.get_order(order_id)

        if not order:
            return web.json_response({'error': 'Order not found'}, status=404)

        # Получаем детали товаров и пользователя параллельно
        product_ids = [item['product_id'] for item in order['items']]

        tasks = [
            self.user_client.get_user(order['user_id']),
            *[self.product_client.get_product(pid) for pid in product_ids]
        ]

        results = await asyncio.gather(*tasks, return_exceptions=True)
        user = results[0]
        products = {
            pid: prod for pid, prod in zip(product_ids, results[1:])
            if not isinstance(prod, Exception) and prod
        }

        # Обогащаем данные заказа
        enriched_items = []
        for item in order['items']:
            product = products.get(item['product_id'], {})
            enriched_items.append({
                'product_id': item['product_id'],
                'product_name': product.get('name', 'Unknown'),
                'product_image': product.get('images', [None])[0],
                'quantity': item['quantity'],
                'price': item['price'],
                'total': item['quantity'] * item['price']
            })

        return web.json_response({
            'id': order['id'],
            'status': order['status'],
            'created_at': order['created_at'],
            'updated_at': order.get('updated_at'),
            'user': {
                'id': user['id'],
                'name': user['name'],
                'email': user['email']
            } if not isinstance(user, Exception) and user else None,
            'items': enriched_items,
            'subtotal': sum(item['total'] for item in enriched_items),
            'shipping': order.get('shipping', {}),
            'payment': order.get('payment', {})
        })

    async def cleanup(self):
        """Закрытие всех клиентов при остановке"""
        await asyncio.gather(
            self.user_client.close(),
            self.product_client.close(),
            self.order_client.close()
        )


def create_web_bff_app() -> web.Application:
    bff = WebBFF()
    app = web.Application()

    # Routes
    app.router.add_get('/dashboard/{user_id}', bff.get_user_dashboard)
    app.router.add_get('/products/{product_id}', bff.get_product_details)
    app.router.add_get('/orders/{order_id}', bff.get_order_with_details)

    # Cleanup
    app.on_cleanup.append(lambda app: bff.cleanup())

    return app


if __name__ == '__main__':
    web.run_app(create_web_bff_app(), port=9001)
```

### Mobile BFF — оптимизированный API

```python
# bff/mobile_bff/main.py
import asyncio
from aiohttp import web
from typing import Dict, Any
import os

from ..common.clients import (
    ServiceConfig,
    UserServiceClient,
    ProductServiceClient,
    OrderServiceClient
)


class MobileBFF:
    """
    BFF для мобильного приложения.

    Особенности:
    - Минимизированный payload (экономия трафика)
    - Сжатые изображения (thumbnails вместо full images)
    - Пагинация с меньшим page size
    - Кэширование на уровне BFF
    """

    def __init__(self):
        self.user_client = UserServiceClient(
            ServiceConfig(
                host=os.getenv('USER_SERVICE_HOST', 'user-service'),
                port=int(os.getenv('USER_SERVICE_PORT', 8001))
            )
        )
        self.product_client = ProductServiceClient(
            ServiceConfig(
                host=os.getenv('PRODUCT_SERVICE_HOST', 'product-service'),
                port=int(os.getenv('PRODUCT_SERVICE_PORT', 8002))
            )
        )
        self.order_client = OrderServiceClient(
            ServiceConfig(
                host=os.getenv('ORDER_SERVICE_HOST', 'order-service'),
                port=int(os.getenv('ORDER_SERVICE_PORT', 8003))
            )
        )

        # Простой in-memory кэш
        self._cache: Dict[str, Any] = {}

    async def get_user_dashboard(self, request: web.Request) -> web.Response:
        """
        Упрощённый dashboard для мобильного приложения.
        Только критичные данные.
        """
        user_id = int(request.match_info['user_id'])

        # Параллельные запросы
        user, orders = await asyncio.gather(
            self.user_client.get_user(user_id),
            self.order_client.get_user_orders(user_id),
            return_exceptions=True
        )

        if isinstance(user, Exception) or user is None:
            return web.json_response({'error': 'User not found'}, status=404)

        # Минимальный ответ для мобильного клиента
        return web.json_response({
            'user': {
                'id': user['id'],
                'name': user['name'],
                'avatar_small': user.get('avatar_thumb')  # Маленький аватар
            },
            'orders_count': len(orders) if not isinstance(orders, Exception) else 0,
            'pending_orders': sum(
                1 for o in (orders if not isinstance(orders, Exception) else [])
                if o.get('status') == 'pending'
            )
        })

    async def get_product_list(self, request: web.Request) -> web.Response:
        """
        Список товаров для мобильного — только thumbnails и основная инфо.
        """
        category = request.query.get('category')
        page = int(request.query.get('page', 1))
        limit = 10  # Меньший page size для мобильных
        offset = (page - 1) * limit

        products = await self.product_client.get_products(
            category=category,
            limit=limit,
            offset=offset
        )

        # Минимизируем данные для мобильного
        mobile_products = [
            {
                'id': p['id'],
                'name': p['name'][:50],  # Обрезаем длинные названия
                'price': p['price'],
                'thumb': p.get('thumbnails', [None])[0],  # Только thumbnail
                'rating': p.get('rating'),
                'in_stock': p.get('stock', 0) > 0
            }
            for p in products
        ]

        return web.json_response({
            'products': mobile_products,
            'page': page,
            'has_more': len(products) == limit
        })

    async def get_product_details(self, request: web.Request) -> web.Response:
        """
        Детали товара для мобильного — без лишних данных.
        """
        product_id = int(request.match_info['product_id'])

        product = await self.product_client.get_product(product_id)

        if not product:
            return web.json_response({'error': 'Product not found'}, status=404)

        # Мобильная версия — компактные данные
        return web.json_response({
            'id': product['id'],
            'name': product['name'],
            'description': product.get('description', '')[:200],  # Короткое описание
            'price': product['price'],
            'images': product.get('thumbnails', [])[:3],  # Максимум 3 изображения
            'rating': product.get('rating'),
            'in_stock': product.get('stock', 0) > 0,
            # Спецификации в виде простого списка
            'specs': [
                f"{k}: {v}"
                for k, v in list(product.get('specifications', {}).items())[:5]
            ]
        })

    async def get_order_summary(self, request: web.Request) -> web.Response:
        """
        Краткая информация о заказе для мобильного.
        """
        order_id = int(request.match_info['order_id'])

        order = await self.order_client.get_order(order_id)

        if not order:
            return web.json_response({'error': 'Order not found'}, status=404)

        return web.json_response({
            'id': order['id'],
            'status': order['status'],
            'items_count': len(order.get('items', [])),
            'total': sum(
                item['quantity'] * item['price']
                for item in order.get('items', [])
            ),
            'created': order['created_at']
        })

    async def cleanup(self):
        await asyncio.gather(
            self.user_client.close(),
            self.product_client.close(),
            self.order_client.close()
        )


def create_mobile_bff_app() -> web.Application:
    bff = MobileBFF()
    app = web.Application()

    # Routes
    app.router.add_get('/dashboard/{user_id}', bff.get_user_dashboard)
    app.router.add_get('/products', bff.get_product_list)
    app.router.add_get('/products/{product_id}', bff.get_product_details)
    app.router.add_get('/orders/{order_id}', bff.get_order_summary)

    app.on_cleanup.append(lambda app: bff.cleanup())

    return app


if __name__ == '__main__':
    web.run_app(create_mobile_bff_app(), port=9002)
```

## Сравнение ответов Web BFF vs Mobile BFF

### Endpoint: `/products/{id}`

**Web BFF Response (~2KB):**
```json
{
  "id": 123,
  "name": "Механическая клавиатура Keychron K8",
  "description": "Беспроводная механическая клавиатура...",
  "full_description": "Полное описание на 500 слов...",
  "price": 8990,
  "currency": "RUB",
  "images": [
    "https://cdn.example.com/full/img1.jpg",
    "https://cdn.example.com/full/img2.jpg",
    "https://cdn.example.com/full/img3.jpg",
    "https://cdn.example.com/full/img4.jpg"
  ],
  "specifications": {
    "layout": "TKL",
    "switches": "Gateron Brown",
    "connection": "Bluetooth/USB-C",
    "battery": "4000mAh"
  },
  "reviews": [...],
  "related_products": [...]
}
```

**Mobile BFF Response (~400 bytes):**
```json
{
  "id": 123,
  "name": "Механическая клавиатура Keychron K8",
  "description": "Беспроводная механическая клавиатура...",
  "price": 8990,
  "images": [
    "https://cdn.example.com/thumb/img1.jpg",
    "https://cdn.example.com/thumb/img2.jpg"
  ],
  "rating": 4.7,
  "in_stock": true,
  "specs": ["layout: TKL", "switches: Gateron Brown"]
}
```

## Best Practices

### 1. Параллельные запросы с обработкой ошибок

```python
async def aggregate_with_fallback(self):
    """Агрегация с graceful degradation"""

    results = await asyncio.gather(
        self.user_client.get_user(user_id),
        self.product_client.get_products(),
        self.order_client.get_orders(user_id),
        return_exceptions=True  # Не падаем при ошибке одного сервиса
    )

    user, products, orders = results

    return {
        'user': user if not isinstance(user, Exception) else None,
        'products': products if not isinstance(products, Exception) else [],
        'orders': orders if not isinstance(orders, Exception) else [],
        # Флаги для клиента
        'partial_data': any(isinstance(r, Exception) for r in results)
    }
```

### 2. Кэширование на уровне BFF

```python
from functools import lru_cache
from datetime import datetime, timedelta


class CachedBFF:
    def __init__(self):
        self._cache: Dict[str, tuple[Any, datetime]] = {}
        self._cache_ttl = timedelta(minutes=5)

    async def get_cached_or_fetch(
        self,
        key: str,
        fetch_func,
        ttl: Optional[timedelta] = None
    ):
        ttl = ttl or self._cache_ttl

        if key in self._cache:
            data, cached_at = self._cache[key]
            if datetime.now() - cached_at < ttl:
                return data

        data = await fetch_func()
        self._cache[key] = (data, datetime.now())
        return data

    async def get_product(self, product_id: int):
        return await self.get_cached_or_fetch(
            f'product:{product_id}',
            lambda: self.product_client.get_product(product_id),
            ttl=timedelta(minutes=10)
        )
```

### 3. Версионирование BFF API

```python
# Разные версии для разных версий мобильного приложения
app.router.add_get('/v1/products/{id}', bff.get_product_v1)
app.router.add_get('/v2/products/{id}', bff.get_product_v2)

# Или через header
async def get_product(self, request: web.Request):
    api_version = request.headers.get('X-API-Version', '1')

    if api_version == '2':
        return await self._get_product_v2(request)
    return await self._get_product_v1(request)
```

## Распространённые ошибки

1. **Слишком много BFF** — один BFF на каждый экран приложения
2. **Дублирование логики** — бизнес-логика должна быть в сервисах, не в BFF
3. **Синхронные цепочки** — используйте `asyncio.gather()` для параллельных запросов
4. **Отсутствие таймаутов** — BFF должен иметь таймауты для всех внешних вызовов

---

[prev: 01-why-microservices.md](./01-why-microservices.md) | [next: 03-product-api.md](./03-product-api.md)
