# Реализация API списка товаров

[prev: 02-backend-for-frontend.md](./02-backend-for-frontend.md) | [next: ../11-synchronization/readme.md](../11-synchronization/readme.md)

---

## Введение

В этом разделе мы создадим полноценный **Product API** — микросервис для управления каталогом товаров. Это практический пример, демонстрирующий паттерны микросервисной архитектуры с использованием asyncio.

## Архитектура Product Service

```
┌─────────────────────────────────────────────────┐
│                 Product Service                  │
│  ┌─────────────────────────────────────────┐    │
│  │              API Layer                   │    │
│  │   (aiohttp handlers + validation)       │    │
│  └────────────────┬────────────────────────┘    │
│                   │                              │
│  ┌────────────────▼────────────────────────┐    │
│  │            Service Layer                 │    │
│  │   (business logic + aggregation)        │    │
│  └────────────────┬────────────────────────┘    │
│                   │                              │
│  ┌────────────────▼────────────────────────┐    │
│  │          Repository Layer                │    │
│  │   (database operations)                  │    │
│  └────────────────┬────────────────────────┘    │
│                   │                              │
│  ┌────────────────▼────────────────────────┐    │
│  │           PostgreSQL                     │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
```

## Структура проекта

```
product_service/
├── main.py              # Точка входа
├── config.py            # Конфигурация
├── models.py            # Pydantic модели
├── repository.py        # Работа с БД
├── service.py           # Бизнес-логика
├── handlers.py          # HTTP handlers
├── middleware.py        # Middleware (logging, errors)
└── requirements.txt
```

## Модели данных

```python
# models.py
from pydantic import BaseModel, Field, validator
from typing import Optional, List
from decimal import Decimal
from datetime import datetime
from enum import Enum


class ProductStatus(str, Enum):
    ACTIVE = 'active'
    INACTIVE = 'inactive'
    OUT_OF_STOCK = 'out_of_stock'


class ProductBase(BaseModel):
    """Базовая модель товара"""
    name: str = Field(..., min_length=1, max_length=255)
    description: Optional[str] = Field(None, max_length=2000)
    price: Decimal = Field(..., gt=0, decimal_places=2)
    category_id: int = Field(..., gt=0)
    stock: int = Field(default=0, ge=0)

    @validator('price')
    def validate_price(cls, v):
        if v <= 0:
            raise ValueError('Price must be positive')
        return v


class ProductCreate(ProductBase):
    """Модель для создания товара"""
    sku: str = Field(..., min_length=3, max_length=50)
    images: List[str] = Field(default_factory=list)
    specifications: dict = Field(default_factory=dict)


class ProductUpdate(BaseModel):
    """Модель для обновления товара (все поля опциональны)"""
    name: Optional[str] = Field(None, min_length=1, max_length=255)
    description: Optional[str] = Field(None, max_length=2000)
    price: Optional[Decimal] = Field(None, gt=0)
    category_id: Optional[int] = Field(None, gt=0)
    stock: Optional[int] = Field(None, ge=0)
    status: Optional[ProductStatus] = None
    images: Optional[List[str]] = None
    specifications: Optional[dict] = None


class ProductResponse(ProductBase):
    """Модель ответа с товаром"""
    id: int
    sku: str
    status: ProductStatus
    images: List[str]
    specifications: dict
    created_at: datetime
    updated_at: datetime

    class Config:
        from_attributes = True


class ProductListResponse(BaseModel):
    """Модель ответа со списком товаров"""
    items: List[ProductResponse]
    total: int
    page: int
    page_size: int
    has_more: bool


class ProductFilter(BaseModel):
    """Фильтры для списка товаров"""
    category_id: Optional[int] = None
    min_price: Optional[Decimal] = None
    max_price: Optional[Decimal] = None
    status: Optional[ProductStatus] = None
    in_stock: Optional[bool] = None
    search: Optional[str] = None
```

## Repository Layer — работа с базой данных

```python
# repository.py
import asyncpg
from typing import Optional, List, Dict, Any
from decimal import Decimal
from datetime import datetime

from models import ProductFilter, ProductStatus


class ProductRepository:
    """Репозиторий для работы с товарами в PostgreSQL"""

    def __init__(self, pool: asyncpg.Pool):
        self.pool = pool

    async def create(self, data: Dict[str, Any]) -> Dict[str, Any]:
        """Создание нового товара"""
        query = '''
            INSERT INTO products (
                sku, name, description, price, category_id,
                stock, status, images, specifications
            )
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
            RETURNING *
        '''

        async with self.pool.acquire() as conn:
            row = await conn.fetchrow(
                query,
                data['sku'],
                data['name'],
                data.get('description'),
                data['price'],
                data['category_id'],
                data.get('stock', 0),
                ProductStatus.ACTIVE.value,
                data.get('images', []),
                data.get('specifications', {})
            )
            return dict(row)

    async def get_by_id(self, product_id: int) -> Optional[Dict[str, Any]]:
        """Получение товара по ID"""
        query = 'SELECT * FROM products WHERE id = $1'

        async with self.pool.acquire() as conn:
            row = await conn.fetchrow(query, product_id)
            return dict(row) if row else None

    async def get_by_sku(self, sku: str) -> Optional[Dict[str, Any]]:
        """Получение товара по SKU"""
        query = 'SELECT * FROM products WHERE sku = $1'

        async with self.pool.acquire() as conn:
            row = await conn.fetchrow(query, sku)
            return dict(row) if row else None

    async def get_list(
        self,
        filters: ProductFilter,
        page: int = 1,
        page_size: int = 20
    ) -> tuple[List[Dict[str, Any]], int]:
        """
        Получение списка товаров с фильтрацией и пагинацией.
        Возвращает (items, total_count).
        """
        conditions = []
        params = []
        param_idx = 1

        # Построение условий фильтрации
        if filters.category_id:
            conditions.append(f'category_id = ${param_idx}')
            params.append(filters.category_id)
            param_idx += 1

        if filters.min_price is not None:
            conditions.append(f'price >= ${param_idx}')
            params.append(filters.min_price)
            param_idx += 1

        if filters.max_price is not None:
            conditions.append(f'price <= ${param_idx}')
            params.append(filters.max_price)
            param_idx += 1

        if filters.status:
            conditions.append(f'status = ${param_idx}')
            params.append(filters.status.value)
            param_idx += 1

        if filters.in_stock is not None:
            if filters.in_stock:
                conditions.append('stock > 0')
            else:
                conditions.append('stock = 0')

        if filters.search:
            conditions.append(f'''
                (name ILIKE ${param_idx} OR description ILIKE ${param_idx})
            ''')
            params.append(f'%{filters.search}%')
            param_idx += 1

        where_clause = ' AND '.join(conditions) if conditions else '1=1'

        # Запрос на получение данных
        query = f'''
            SELECT * FROM products
            WHERE {where_clause}
            ORDER BY created_at DESC
            LIMIT ${param_idx} OFFSET ${param_idx + 1}
        '''
        params.extend([page_size, (page - 1) * page_size])

        # Запрос на подсчёт общего количества
        count_query = f'''
            SELECT COUNT(*) FROM products WHERE {where_clause}
        '''

        async with self.pool.acquire() as conn:
            # Выполняем оба запроса параллельно
            rows, total = await asyncio.gather(
                conn.fetch(query, *params),
                conn.fetchval(count_query, *params[:-2])  # Без LIMIT/OFFSET
            )

            return [dict(row) for row in rows], total

    async def update(
        self,
        product_id: int,
        data: Dict[str, Any]
    ) -> Optional[Dict[str, Any]]:
        """Обновление товара"""
        # Формируем SET clause динамически
        set_parts = []
        params = []
        param_idx = 1

        for key, value in data.items():
            if value is not None:
                set_parts.append(f'{key} = ${param_idx}')
                params.append(value)
                param_idx += 1

        if not set_parts:
            return await self.get_by_id(product_id)

        # Добавляем updated_at
        set_parts.append(f'updated_at = ${param_idx}')
        params.append(datetime.utcnow())
        param_idx += 1

        # Добавляем ID в конец
        params.append(product_id)

        query = f'''
            UPDATE products
            SET {', '.join(set_parts)}
            WHERE id = ${param_idx}
            RETURNING *
        '''

        async with self.pool.acquire() as conn:
            row = await conn.fetchrow(query, *params)
            return dict(row) if row else None

    async def delete(self, product_id: int) -> bool:
        """Удаление товара"""
        query = 'DELETE FROM products WHERE id = $1 RETURNING id'

        async with self.pool.acquire() as conn:
            result = await conn.fetchval(query, product_id)
            return result is not None

    async def update_stock(
        self,
        product_id: int,
        quantity_change: int
    ) -> Optional[Dict[str, Any]]:
        """
        Атомарное изменение остатка товара.
        quantity_change может быть положительным (пополнение) или отрицательным (списание).
        """
        query = '''
            UPDATE products
            SET stock = stock + $2,
                status = CASE
                    WHEN stock + $2 <= 0 THEN 'out_of_stock'
                    ELSE status
                END,
                updated_at = NOW()
            WHERE id = $1 AND stock + $2 >= 0
            RETURNING *
        '''

        async with self.pool.acquire() as conn:
            row = await conn.fetchrow(query, product_id, quantity_change)
            return dict(row) if row else None

    async def get_by_ids(self, product_ids: List[int]) -> List[Dict[str, Any]]:
        """Получение нескольких товаров по списку ID"""
        if not product_ids:
            return []

        query = 'SELECT * FROM products WHERE id = ANY($1)'

        async with self.pool.acquire() as conn:
            rows = await conn.fetch(query, product_ids)
            return [dict(row) for row in rows]


# SQL для создания таблицы
CREATE_TABLE_SQL = '''
CREATE TABLE IF NOT EXISTS products (
    id SERIAL PRIMARY KEY,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    category_id INTEGER NOT NULL,
    stock INTEGER DEFAULT 0,
    status VARCHAR(20) DEFAULT 'active',
    images TEXT[] DEFAULT '{}',
    specifications JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_products_category ON products(category_id);
CREATE INDEX IF NOT EXISTS idx_products_status ON products(status);
CREATE INDEX IF NOT EXISTS idx_products_price ON products(price);
CREATE INDEX IF NOT EXISTS idx_products_name_search ON products USING gin(to_tsvector('russian', name));
'''
```

## Service Layer — бизнес-логика

```python
# service.py
import asyncio
from typing import Optional, List, Dict, Any
from decimal import Decimal

from models import (
    ProductCreate, ProductUpdate, ProductResponse,
    ProductListResponse, ProductFilter, ProductStatus
)
from repository import ProductRepository


class ProductServiceError(Exception):
    """Базовый класс ошибок сервиса"""
    pass


class ProductNotFoundError(ProductServiceError):
    """Товар не найден"""
    pass


class ProductAlreadyExistsError(ProductServiceError):
    """Товар с таким SKU уже существует"""
    pass


class InsufficientStockError(ProductServiceError):
    """Недостаточно товара на складе"""
    pass


class ProductService:
    """Сервис для работы с товарами"""

    def __init__(self, repository: ProductRepository):
        self.repository = repository

    async def create_product(self, data: ProductCreate) -> ProductResponse:
        """Создание нового товара"""
        # Проверяем уникальность SKU
        existing = await self.repository.get_by_sku(data.sku)
        if existing:
            raise ProductAlreadyExistsError(
                f"Product with SKU '{data.sku}' already exists"
            )

        product = await self.repository.create(data.model_dump())
        return ProductResponse(**product)

    async def get_product(self, product_id: int) -> ProductResponse:
        """Получение товара по ID"""
        product = await self.repository.get_by_id(product_id)
        if not product:
            raise ProductNotFoundError(f"Product {product_id} not found")
        return ProductResponse(**product)

    async def get_products(
        self,
        filters: ProductFilter,
        page: int = 1,
        page_size: int = 20
    ) -> ProductListResponse:
        """Получение списка товаров с фильтрацией"""
        # Ограничиваем page_size
        page_size = min(page_size, 100)

        items, total = await self.repository.get_list(
            filters=filters,
            page=page,
            page_size=page_size
        )

        return ProductListResponse(
            items=[ProductResponse(**item) for item in items],
            total=total,
            page=page,
            page_size=page_size,
            has_more=(page * page_size) < total
        )

    async def update_product(
        self,
        product_id: int,
        data: ProductUpdate
    ) -> ProductResponse:
        """Обновление товара"""
        # Проверяем существование
        existing = await self.repository.get_by_id(product_id)
        if not existing:
            raise ProductNotFoundError(f"Product {product_id} not found")

        # Обновляем только переданные поля
        update_data = data.model_dump(exclude_unset=True)

        # Преобразуем Enum в строку для БД
        if 'status' in update_data and update_data['status']:
            update_data['status'] = update_data['status'].value

        product = await self.repository.update(product_id, update_data)
        return ProductResponse(**product)

    async def delete_product(self, product_id: int) -> bool:
        """Удаление товара"""
        deleted = await self.repository.delete(product_id)
        if not deleted:
            raise ProductNotFoundError(f"Product {product_id} not found")
        return True

    async def reserve_stock(
        self,
        product_id: int,
        quantity: int
    ) -> ProductResponse:
        """
        Резервирование товара (уменьшение остатка).
        Используется при создании заказа.
        """
        if quantity <= 0:
            raise ValueError("Quantity must be positive")

        product = await self.repository.update_stock(product_id, -quantity)
        if not product:
            # Либо товар не найден, либо недостаточно на складе
            existing = await self.repository.get_by_id(product_id)
            if not existing:
                raise ProductNotFoundError(f"Product {product_id} not found")
            raise InsufficientStockError(
                f"Insufficient stock for product {product_id}. "
                f"Available: {existing['stock']}, requested: {quantity}"
            )

        return ProductResponse(**product)

    async def release_stock(
        self,
        product_id: int,
        quantity: int
    ) -> ProductResponse:
        """
        Возврат товара на склад (увеличение остатка).
        Используется при отмене заказа.
        """
        if quantity <= 0:
            raise ValueError("Quantity must be positive")

        product = await self.repository.update_stock(product_id, quantity)
        if not product:
            raise ProductNotFoundError(f"Product {product_id} not found")

        return ProductResponse(**product)

    async def get_products_batch(
        self,
        product_ids: List[int]
    ) -> List[ProductResponse]:
        """Получение нескольких товаров по списку ID"""
        products = await self.repository.get_by_ids(product_ids)
        return [ProductResponse(**p) for p in products]
```

## HTTP Handlers — API endpoints

```python
# handlers.py
import asyncio
from aiohttp import web
from typing import Optional
import json
from decimal import Decimal

from models import (
    ProductCreate, ProductUpdate, ProductFilter, ProductStatus
)
from service import (
    ProductService, ProductNotFoundError,
    ProductAlreadyExistsError, InsufficientStockError
)


class ProductHandlers:
    """HTTP handlers для Product API"""

    def __init__(self, service: ProductService):
        self.service = service

    async def create_product(self, request: web.Request) -> web.Response:
        """
        POST /products

        Создание нового товара.
        """
        try:
            data = await request.json()
            product_data = ProductCreate(**data)
        except Exception as e:
            return web.json_response(
                {'error': 'Invalid request data', 'details': str(e)},
                status=400
            )

        try:
            product = await self.service.create_product(product_data)
            return web.json_response(
                product.model_dump(mode='json'),
                status=201
            )
        except ProductAlreadyExistsError as e:
            return web.json_response(
                {'error': str(e)},
                status=409
            )

    async def get_product(self, request: web.Request) -> web.Response:
        """
        GET /products/{id}

        Получение товара по ID.
        """
        try:
            product_id = int(request.match_info['id'])
        except ValueError:
            return web.json_response(
                {'error': 'Invalid product ID'},
                status=400
            )

        try:
            product = await self.service.get_product(product_id)
            return web.json_response(product.model_dump(mode='json'))
        except ProductNotFoundError:
            return web.json_response(
                {'error': f'Product {product_id} not found'},
                status=404
            )

    async def list_products(self, request: web.Request) -> web.Response:
        """
        GET /products

        Список товаров с фильтрацией и пагинацией.

        Query params:
        - page: номер страницы (default: 1)
        - page_size: размер страницы (default: 20, max: 100)
        - category_id: фильтр по категории
        - min_price, max_price: фильтр по цене
        - status: фильтр по статусу
        - in_stock: только в наличии (true/false)
        - search: поиск по названию/описанию
        """
        query = request.query

        # Парсинг параметров
        page = int(query.get('page', 1))
        page_size = int(query.get('page_size', 20))

        # Построение фильтров
        filters = ProductFilter(
            category_id=int(query['category_id']) if 'category_id' in query else None,
            min_price=Decimal(query['min_price']) if 'min_price' in query else None,
            max_price=Decimal(query['max_price']) if 'max_price' in query else None,
            status=ProductStatus(query['status']) if 'status' in query else None,
            in_stock=query.get('in_stock', '').lower() == 'true' if 'in_stock' in query else None,
            search=query.get('search')
        )

        result = await self.service.get_products(filters, page, page_size)
        return web.json_response(result.model_dump(mode='json'))

    async def update_product(self, request: web.Request) -> web.Response:
        """
        PATCH /products/{id}

        Обновление товара.
        """
        try:
            product_id = int(request.match_info['id'])
        except ValueError:
            return web.json_response(
                {'error': 'Invalid product ID'},
                status=400
            )

        try:
            data = await request.json()
            update_data = ProductUpdate(**data)
        except Exception as e:
            return web.json_response(
                {'error': 'Invalid request data', 'details': str(e)},
                status=400
            )

        try:
            product = await self.service.update_product(product_id, update_data)
            return web.json_response(product.model_dump(mode='json'))
        except ProductNotFoundError:
            return web.json_response(
                {'error': f'Product {product_id} not found'},
                status=404
            )

    async def delete_product(self, request: web.Request) -> web.Response:
        """
        DELETE /products/{id}

        Удаление товара.
        """
        try:
            product_id = int(request.match_info['id'])
        except ValueError:
            return web.json_response(
                {'error': 'Invalid product ID'},
                status=400
            )

        try:
            await self.service.delete_product(product_id)
            return web.Response(status=204)
        except ProductNotFoundError:
            return web.json_response(
                {'error': f'Product {product_id} not found'},
                status=404
            )

    async def reserve_stock(self, request: web.Request) -> web.Response:
        """
        POST /products/{id}/reserve

        Резервирование товара для заказа.
        Body: {"quantity": 5}
        """
        try:
            product_id = int(request.match_info['id'])
            data = await request.json()
            quantity = int(data['quantity'])
        except (ValueError, KeyError) as e:
            return web.json_response(
                {'error': 'Invalid request', 'details': str(e)},
                status=400
            )

        try:
            product = await self.service.reserve_stock(product_id, quantity)
            return web.json_response(product.model_dump(mode='json'))
        except ProductNotFoundError:
            return web.json_response(
                {'error': f'Product {product_id} not found'},
                status=404
            )
        except InsufficientStockError as e:
            return web.json_response(
                {'error': str(e)},
                status=409
            )

    async def get_products_batch(self, request: web.Request) -> web.Response:
        """
        POST /products/batch

        Получение нескольких товаров по списку ID.
        Body: {"ids": [1, 2, 3]}
        """
        try:
            data = await request.json()
            product_ids = [int(id) for id in data['ids']]
        except (ValueError, KeyError) as e:
            return web.json_response(
                {'error': 'Invalid request', 'details': str(e)},
                status=400
            )

        products = await self.service.get_products_batch(product_ids)
        return web.json_response(
            [p.model_dump(mode='json') for p in products]
        )

    async def health_check(self, request: web.Request) -> web.Response:
        """
        GET /health

        Health check endpoint для оркестраторов.
        """
        return web.json_response({
            'status': 'healthy',
            'service': 'product-service'
        })


def setup_routes(app: web.Application, handlers: ProductHandlers):
    """Настройка маршрутов"""
    app.router.add_post('/products', handlers.create_product)
    app.router.add_get('/products', handlers.list_products)
    app.router.add_get('/products/{id}', handlers.get_product)
    app.router.add_patch('/products/{id}', handlers.update_product)
    app.router.add_delete('/products/{id}', handlers.delete_product)
    app.router.add_post('/products/{id}/reserve', handlers.reserve_stock)
    app.router.add_post('/products/batch', handlers.get_products_batch)
    app.router.add_get('/health', handlers.health_check)
```

## Middleware — обработка ошибок и логирование

```python
# middleware.py
import asyncio
import logging
import time
from aiohttp import web
from typing import Callable, Awaitable

logger = logging.getLogger(__name__)


@web.middleware
async def error_middleware(
    request: web.Request,
    handler: Callable[[web.Request], Awaitable[web.Response]]
) -> web.Response:
    """Middleware для обработки непредвиденных ошибок"""
    try:
        return await handler(request)
    except web.HTTPException:
        raise
    except Exception as e:
        logger.exception(f"Unhandled error: {e}")
        return web.json_response(
            {'error': 'Internal server error'},
            status=500
        )


@web.middleware
async def logging_middleware(
    request: web.Request,
    handler: Callable[[web.Request], Awaitable[web.Response]]
) -> web.Response:
    """Middleware для логирования запросов"""
    start_time = time.time()

    try:
        response = await handler(request)
        return response
    finally:
        duration = time.time() - start_time
        logger.info(
            f"{request.method} {request.path} - "
            f"{response.status if 'response' in dir() else 'ERROR'} - "
            f"{duration:.3f}s"
        )


@web.middleware
async def request_id_middleware(
    request: web.Request,
    handler: Callable[[web.Request], Awaitable[web.Response]]
) -> web.Response:
    """Middleware для добавления request ID"""
    import uuid

    request_id = request.headers.get('X-Request-ID', str(uuid.uuid4()))
    request['request_id'] = request_id

    response = await handler(request)
    response.headers['X-Request-ID'] = request_id

    return response
```

## Точка входа — main.py

```python
# main.py
import asyncio
import asyncpg
import logging
from aiohttp import web

from config import Config
from repository import ProductRepository
from service import ProductService
from handlers import ProductHandlers, setup_routes
from middleware import error_middleware, logging_middleware, request_id_middleware

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


async def create_app() -> web.Application:
    """Создание и настройка приложения"""
    config = Config()

    # Создаём пул подключений к БД
    db_pool = await asyncpg.create_pool(
        host=config.db_host,
        port=config.db_port,
        user=config.db_user,
        password=config.db_password,
        database=config.db_name,
        min_size=5,
        max_size=20
    )

    # Инициализация слоёв
    repository = ProductRepository(db_pool)
    service = ProductService(repository)
    handlers = ProductHandlers(service)

    # Создаём приложение с middleware
    app = web.Application(middlewares=[
        request_id_middleware,
        logging_middleware,
        error_middleware
    ])

    # Настраиваем маршруты
    setup_routes(app, handlers)

    # Сохраняем ресурсы для cleanup
    app['db_pool'] = db_pool

    # Cleanup при завершении
    async def cleanup(app):
        await db_pool.close()
        logger.info("Database pool closed")

    app.on_cleanup.append(cleanup)

    logger.info("Application created successfully")
    return app


def main():
    """Запуск сервиса"""
    app = asyncio.get_event_loop().run_until_complete(create_app())
    web.run_app(app, host='0.0.0.0', port=8002)


if __name__ == '__main__':
    main()
```

## Конфигурация

```python
# config.py
import os
from dataclasses import dataclass


@dataclass
class Config:
    """Конфигурация сервиса из переменных окружения"""

    db_host: str = os.getenv('DB_HOST', 'localhost')
    db_port: int = int(os.getenv('DB_PORT', 5432))
    db_user: str = os.getenv('DB_USER', 'postgres')
    db_password: str = os.getenv('DB_PASSWORD', 'postgres')
    db_name: str = os.getenv('DB_NAME', 'products')

    service_port: int = int(os.getenv('SERVICE_PORT', 8002))
    log_level: str = os.getenv('LOG_LEVEL', 'INFO')
```

## Примеры использования API

```bash
# Создание товара
curl -X POST http://localhost:8002/products \
  -H "Content-Type: application/json" \
  -d '{
    "sku": "KB-001",
    "name": "Механическая клавиатура",
    "description": "Игровая клавиатура с RGB подсветкой",
    "price": 5990.00,
    "category_id": 1,
    "stock": 100,
    "images": ["https://example.com/img1.jpg"],
    "specifications": {"switches": "Cherry MX Red", "layout": "Full"}
  }'

# Получение товара
curl http://localhost:8002/products/1

# Список товаров с фильтрами
curl "http://localhost:8002/products?category_id=1&min_price=1000&in_stock=true&page=1&page_size=10"

# Обновление товара
curl -X PATCH http://localhost:8002/products/1 \
  -H "Content-Type: application/json" \
  -d '{"price": 4990.00, "stock": 50}'

# Резервирование товара
curl -X POST http://localhost:8002/products/1/reserve \
  -H "Content-Type: application/json" \
  -d '{"quantity": 5}'

# Batch запрос
curl -X POST http://localhost:8002/products/batch \
  -H "Content-Type: application/json" \
  -d '{"ids": [1, 2, 3]}'
```

## Best Practices

1. **Слоистая архитектура** — разделение на handlers, service, repository
2. **Валидация с Pydantic** — типизация и валидация на входе
3. **Идемпотентность** — использование SKU для предотвращения дубликатов
4. **Атомарные операции** — update_stock с проверкой в одном запросе
5. **Graceful shutdown** — корректное закрытие соединений с БД
6. **Health checks** — endpoint для мониторинга состояния сервиса

---

[prev: 02-backend-for-frontend.md](./02-backend-for-frontend.md) | [next: ../11-synchronization/readme.md](../11-synchronization/readme.md)
