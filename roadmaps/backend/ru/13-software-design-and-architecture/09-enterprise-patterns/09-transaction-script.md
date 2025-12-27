# Transaction Script

## Что такое Transaction Script?

**Transaction Script** — это паттерн организации бизнес-логики, при котором каждая бизнес-операция реализуется как отдельная процедура (скрипт). Каждый скрипт обрабатывает один запрос от презентационного слоя, выполняя все необходимые действия: валидацию, бизнес-правила, работу с БД.

> Паттерн описан Мартином Фаулером в "Patterns of Enterprise Application Architecture" и является простейшим способом организации бизнес-логики.

## Основная идея

Transaction Script — это процедурный подход к бизнес-логике:

```
Входящий запрос
      │
      v
┌─────────────────────────────────┐
│      Transaction Script         │
│  ┌───────────────────────────┐  │
│  │ 1. Получить данные        │  │
│  │ 2. Валидировать           │  │
│  │ 3. Применить бизнес-логику│  │
│  │ 4. Сохранить в БД         │  │
│  │ 5. Вернуть результат      │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
      │
      v
   Результат
```

## Зачем нужен Transaction Script?

### Основные преимущества:

1. **Простота** — понятная процедурная логика
2. **Быстрая разработка** — минимум абстракций
3. **Легко понять** — весь код в одном месте
4. **Хорош для CRUD** — типичные операции просты
5. **Независимость** — скрипты изолированы друг от друга

## Базовая реализация

### Простой Transaction Script

```python
from typing import Optional
from dataclasses import dataclass
from datetime import datetime
import sqlite3


@dataclass
class TransferResult:
    success: bool
    message: str
    transaction_id: Optional[int] = None


def transfer_money(
    from_account_id: int,
    to_account_id: int,
    amount: float,
    connection: sqlite3.Connection
) -> TransferResult:
    """
    Transaction Script для перевода денег между счетами.

    Это один скрипт, который выполняет всю бизнес-операцию:
    1. Валидация
    2. Проверка баланса
    3. Обновление счетов
    4. Запись транзакции
    """
    cursor = connection.cursor()

    try:
        # Начинаем транзакцию
        connection.execute("BEGIN")

        # 1. Получаем данные счёта отправителя
        cursor.execute(
            "SELECT balance FROM accounts WHERE id = ?",
            (from_account_id,)
        )
        from_account = cursor.fetchone()

        if not from_account:
            return TransferResult(False, "Source account not found")

        # 2. Проверяем баланс
        if from_account[0] < amount:
            return TransferResult(False, "Insufficient funds")

        # 3. Получаем данные счёта получателя
        cursor.execute(
            "SELECT id FROM accounts WHERE id = ?",
            (to_account_id,)
        )
        to_account = cursor.fetchone()

        if not to_account:
            return TransferResult(False, "Destination account not found")

        # 4. Выполняем перевод
        cursor.execute(
            "UPDATE accounts SET balance = balance - ? WHERE id = ?",
            (amount, from_account_id)
        )
        cursor.execute(
            "UPDATE accounts SET balance = balance + ? WHERE id = ?",
            (amount, to_account_id)
        )

        # 5. Записываем транзакцию
        cursor.execute(
            """INSERT INTO transactions
               (from_account_id, to_account_id, amount, created_at)
               VALUES (?, ?, ?, ?)""",
            (from_account_id, to_account_id, amount, datetime.now())
        )
        transaction_id = cursor.lastrowid

        # Фиксируем транзакцию
        connection.commit()

        return TransferResult(
            success=True,
            message="Transfer completed successfully",
            transaction_id=transaction_id
        )

    except Exception as e:
        connection.rollback()
        return TransferResult(False, f"Transfer failed: {str(e)}")
```

### Организация в классы

```python
from dataclasses import dataclass
from typing import Optional, List
from datetime import datetime


@dataclass
class OrderDTO:
    customer_id: int
    product_ids: List[int]
    quantities: List[int]


@dataclass
class OrderResult:
    success: bool
    message: str
    order_id: Optional[int] = None
    total: float = 0.0


class OrderScripts:
    """Коллекция Transaction Scripts для заказов."""

    def __init__(self, db_connection):
        self.db = db_connection

    def create_order(self, dto: OrderDTO) -> OrderResult:
        """
        Transaction Script: Создание заказа.
        """
        cursor = self.db.cursor()

        try:
            self.db.execute("BEGIN")

            # 1. Проверяем клиента
            cursor.execute(
                "SELECT id, is_active FROM customers WHERE id = ?",
                (dto.customer_id,)
            )
            customer = cursor.fetchone()

            if not customer:
                return OrderResult(False, "Customer not found")

            if not customer[1]:
                return OrderResult(False, "Customer is inactive")

            # 2. Проверяем продукты и считаем сумму
            total = 0.0
            order_items = []

            for product_id, quantity in zip(dto.product_ids, dto.quantities):
                cursor.execute(
                    "SELECT id, name, price, stock FROM products WHERE id = ?",
                    (product_id,)
                )
                product = cursor.fetchone()

                if not product:
                    return OrderResult(False, f"Product {product_id} not found")

                if product[3] < quantity:
                    return OrderResult(
                        False,
                        f"Insufficient stock for {product[1]}"
                    )

                item_total = product[2] * quantity
                total += item_total
                order_items.append({
                    "product_id": product_id,
                    "quantity": quantity,
                    "price": product[2],
                    "total": item_total
                })

            # 3. Создаём заказ
            cursor.execute(
                """INSERT INTO orders (customer_id, total, status, created_at)
                   VALUES (?, ?, 'pending', ?)""",
                (dto.customer_id, total, datetime.now())
            )
            order_id = cursor.lastrowid

            # 4. Добавляем позиции заказа
            for item in order_items:
                cursor.execute(
                    """INSERT INTO order_items
                       (order_id, product_id, quantity, price)
                       VALUES (?, ?, ?, ?)""",
                    (order_id, item["product_id"], item["quantity"], item["price"])
                )

                # 5. Уменьшаем запас
                cursor.execute(
                    "UPDATE products SET stock = stock - ? WHERE id = ?",
                    (item["quantity"], item["product_id"])
                )

            self.db.commit()

            return OrderResult(
                success=True,
                message="Order created successfully",
                order_id=order_id,
                total=total
            )

        except Exception as e:
            self.db.rollback()
            return OrderResult(False, f"Order creation failed: {str(e)}")

    def cancel_order(self, order_id: int) -> OrderResult:
        """
        Transaction Script: Отмена заказа.
        """
        cursor = self.db.cursor()

        try:
            self.db.execute("BEGIN")

            # 1. Получаем заказ
            cursor.execute(
                "SELECT id, status FROM orders WHERE id = ?",
                (order_id,)
            )
            order = cursor.fetchone()

            if not order:
                return OrderResult(False, "Order not found")

            if order[1] == 'cancelled':
                return OrderResult(False, "Order is already cancelled")

            if order[1] == 'shipped':
                return OrderResult(False, "Cannot cancel shipped order")

            # 2. Получаем позиции заказа
            cursor.execute(
                "SELECT product_id, quantity FROM order_items WHERE order_id = ?",
                (order_id,)
            )
            items = cursor.fetchall()

            # 3. Возвращаем товары на склад
            for product_id, quantity in items:
                cursor.execute(
                    "UPDATE products SET stock = stock + ? WHERE id = ?",
                    (quantity, product_id)
                )

            # 4. Обновляем статус заказа
            cursor.execute(
                "UPDATE orders SET status = 'cancelled' WHERE id = ?",
                (order_id,)
            )

            self.db.commit()

            return OrderResult(
                success=True,
                message="Order cancelled successfully",
                order_id=order_id
            )

        except Exception as e:
            self.db.rollback()
            return OrderResult(False, f"Cancellation failed: {str(e)}")
```

## Transaction Script с SQLAlchemy

```python
from sqlalchemy import create_engine, Column, Integer, String, Float, Boolean, DateTime, ForeignKey
from sqlalchemy.orm import declarative_base, Session, sessionmaker, relationship
from dataclasses import dataclass
from typing import Optional, List
from datetime import datetime

Base = declarative_base()


# Модели (простые, без бизнес-логики)
class Customer(Base):
    __tablename__ = "customers"
    id = Column(Integer, primary_key=True)
    name = Column(String(100))
    email = Column(String(100))
    is_active = Column(Boolean, default=True)


class Product(Base):
    __tablename__ = "products"
    id = Column(Integer, primary_key=True)
    name = Column(String(100))
    price = Column(Float)
    stock = Column(Integer)


class Order(Base):
    __tablename__ = "orders"
    id = Column(Integer, primary_key=True)
    customer_id = Column(Integer, ForeignKey("customers.id"))
    total = Column(Float)
    status = Column(String(20))
    created_at = Column(DateTime)


class OrderItem(Base):
    __tablename__ = "order_items"
    id = Column(Integer, primary_key=True)
    order_id = Column(Integer, ForeignKey("orders.id"))
    product_id = Column(Integer, ForeignKey("products.id"))
    quantity = Column(Integer)
    price = Column(Float)


# Transaction Scripts
@dataclass
class CreateOrderInput:
    customer_id: int
    items: List[tuple]  # [(product_id, quantity), ...]


@dataclass
class CreateOrderOutput:
    success: bool
    message: str
    order_id: Optional[int] = None


def create_order_script(session: Session, input_data: CreateOrderInput) -> CreateOrderOutput:
    """
    Transaction Script для создания заказа.
    """
    try:
        # 1. Проверяем клиента
        customer = session.query(Customer).filter_by(id=input_data.customer_id).first()

        if not customer:
            return CreateOrderOutput(False, "Customer not found")

        if not customer.is_active:
            return CreateOrderOutput(False, "Customer is inactive")

        # 2. Проверяем продукты и считаем сумму
        total = 0.0
        items_to_add = []

        for product_id, quantity in input_data.items:
            product = session.query(Product).filter_by(id=product_id).first()

            if not product:
                return CreateOrderOutput(False, f"Product {product_id} not found")

            if product.stock < quantity:
                return CreateOrderOutput(False, f"Insufficient stock for {product.name}")

            total += product.price * quantity
            items_to_add.append((product, quantity))

        # 3. Создаём заказ
        order = Order(
            customer_id=input_data.customer_id,
            total=total,
            status="pending",
            created_at=datetime.now()
        )
        session.add(order)
        session.flush()  # Получаем ID

        # 4. Добавляем позиции и уменьшаем запас
        for product, quantity in items_to_add:
            order_item = OrderItem(
                order_id=order.id,
                product_id=product.id,
                quantity=quantity,
                price=product.price
            )
            session.add(order_item)
            product.stock -= quantity

        session.commit()

        return CreateOrderOutput(
            success=True,
            message="Order created successfully",
            order_id=order.id
        )

    except Exception as e:
        session.rollback()
        return CreateOrderOutput(False, f"Order creation failed: {str(e)}")


def ship_order_script(session: Session, order_id: int) -> CreateOrderOutput:
    """
    Transaction Script для отправки заказа.
    """
    try:
        order = session.query(Order).filter_by(id=order_id).first()

        if not order:
            return CreateOrderOutput(False, "Order not found")

        if order.status != "pending":
            return CreateOrderOutput(False, f"Cannot ship order with status {order.status}")

        order.status = "shipped"
        session.commit()

        return CreateOrderOutput(
            success=True,
            message="Order shipped successfully",
            order_id=order.id
        )

    except Exception as e:
        session.rollback()
        return CreateOrderOutput(False, f"Shipping failed: {str(e)}")
```

## Transaction Script в FastAPI

```python
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel
from typing import List, Optional
from sqlalchemy.orm import Session

app = FastAPI()


# Pydantic модели
class OrderItemInput(BaseModel):
    product_id: int
    quantity: int


class CreateOrderRequest(BaseModel):
    customer_id: int
    items: List[OrderItemInput]


class OrderResponse(BaseModel):
    success: bool
    message: str
    order_id: Optional[int] = None
    total: Optional[float] = None


# Transaction Script как функция
def create_order_transaction(
    db: Session,
    customer_id: int,
    items: List[OrderItemInput]
) -> OrderResponse:
    """Transaction Script для создания заказа."""

    # 1. Валидация клиента
    customer = db.query(Customer).filter_by(id=customer_id).first()
    if not customer:
        return OrderResponse(success=False, message="Customer not found")

    if not customer.is_active:
        return OrderResponse(success=False, message="Customer is inactive")

    # 2. Валидация товаров и расчёт
    total = 0.0
    validated_items = []

    for item in items:
        product = db.query(Product).filter_by(id=item.product_id).first()

        if not product:
            return OrderResponse(
                success=False,
                message=f"Product {item.product_id} not found"
            )

        if product.stock < item.quantity:
            return OrderResponse(
                success=False,
                message=f"Insufficient stock for {product.name}"
            )

        total += product.price * item.quantity
        validated_items.append((product, item.quantity))

    # 3. Создание заказа
    try:
        order = Order(
            customer_id=customer_id,
            total=total,
            status="pending",
            created_at=datetime.now()
        )
        db.add(order)
        db.flush()

        for product, quantity in validated_items:
            db.add(OrderItem(
                order_id=order.id,
                product_id=product.id,
                quantity=quantity,
                price=product.price
            ))
            product.stock -= quantity

        db.commit()

        return OrderResponse(
            success=True,
            message="Order created",
            order_id=order.id,
            total=total
        )

    except Exception as e:
        db.rollback()
        return OrderResponse(success=False, message=str(e))


# Endpoint
@app.post("/orders", response_model=OrderResponse)
def create_order(
    request: CreateOrderRequest,
    db: Session = Depends(get_db)
):
    """Создание заказа."""
    result = create_order_transaction(
        db=db,
        customer_id=request.customer_id,
        items=request.items
    )

    if not result.success:
        raise HTTPException(status_code=400, detail=result.message)

    return result
```

## Сравнение с Domain Model

| Аспект | Transaction Script | Domain Model |
|--------|-------------------|--------------|
| **Сложность** | Низкая | Высокая |
| **Бизнес-логика** | В процедурах | В объектах |
| **Повторное использование** | Сложнее | Проще |
| **Тестирование** | Требует БД | Изолированно |
| **Подходит для** | Простой CRUD | Сложный домен |

## Когда использовать Transaction Script?

### Рекомендуется:
- Простые CRUD-приложения
- Небольшие проекты
- Прототипы и MVP
- Когда бизнес-логика проста
- Административные панели

### Не рекомендуется:
- Сложная бизнес-логика
- Много дублирования между скриптами
- Когда нужна высокая тестируемость
- DDD-проекты

## Плюсы и минусы

### Преимущества

1. **Простота** — понятный процедурный код
2. **Быстрота разработки** — минимум абстракций
3. **Легко отлаживать** — всё в одном месте
4. **Низкий порог входа** — не нужно знать паттерны

### Недостатки

1. **Дублирование** — код повторяется между скриптами
2. **Сложность роста** — становится запутанным при усложнении
3. **Тестирование** — требует БД для тестов
4. **Нет инкапсуляции** — бизнес-логика разбросана

## Эволюция Transaction Script

```python
# Этап 1: Простой скрипт
def transfer_money(from_id, to_id, amount, db):
    # Вся логика в одной функции
    pass


# Этап 2: Выделение валидации
def validate_transfer(from_id, to_id, amount, db):
    pass

def execute_transfer(from_id, to_id, amount, db):
    if not validate_transfer(from_id, to_id, amount, db):
        return False
    # Выполнение
    pass


# Этап 3: Переход к Domain Model
class Account:
    def transfer_to(self, target, amount):
        if not self.can_transfer(amount):
            raise InsufficientFunds()
        self.balance -= amount
        target.balance += amount
```

## Лучшие практики

1. **Один скрипт — одна операция** — не смешивайте бизнес-операции
2. **Чёткие входы и выходы** — используйте DTO/dataclass
3. **Обрабатывайте ошибки** — возвращайте Result объекты
4. **Транзакции БД** — оборачивайте в try/except с rollback
5. **Знайте границы** — при усложнении переходите к Domain Model

## Заключение

Transaction Script — это простой и эффективный подход для несложных приложений. Он позволяет быстро реализовать бизнес-логику без overhead паттернов DDD. Однако при росте сложности важно вовремя распознать момент, когда нужно переходить к более структурированным подходам, таким как Domain Model или Use Cases.
