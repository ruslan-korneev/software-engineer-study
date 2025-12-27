# Tell, Don't Ask — Говори, не спрашивай

## Определение

**Tell, Don't Ask** — принцип объектно-ориентированного проектирования, который гласит:

> "Вместо того чтобы запрашивать у объекта данные для принятия решений, скажите объекту, что делать."

Другими словами: объекты должны **командовать** другим объектам выполнять действия, а не **спрашивать** их состояние для принятия решений снаружи.

---

## Суть принципа

### Проблема "Ask" подхода

Когда мы запрашиваем данные у объекта и принимаем решения снаружи:
1. Логика размазывается по коду
2. Нарушается инкапсуляция
3. Код становится процедурным, а не объектно-ориентированным
4. Дублируется логика принятия решений

### Решение "Tell" подхода

Объект сам содержит логику и принимает решения на основе своего состояния. Клиенты просто говорят ему, что делать.

---

## Примеры

### 1. Базовый пример

**Плохо — Ask (спрашиваем и решаем снаружи):**

```python
class Account:
    def __init__(self, balance: float):
        self.balance = balance


def withdraw(account: Account, amount: float):
    # Спрашиваем состояние
    if account.balance >= amount:
        # Принимаем решение снаружи
        account.balance -= amount
        return True
    else:
        return False


# Использование
account = Account(1000)
if withdraw(account, 500):
    print("Success")
else:
    print("Insufficient funds")
```

**Хорошо — Tell (говорим объекту, что делать):**

```python
class Account:
    def __init__(self, balance: float):
        self._balance = balance  # Приватное!

    def withdraw(self, amount: float) -> bool:
        """Объект сам принимает решение"""
        if self._balance >= amount:
            self._balance -= amount
            return True
        return False

    @property
    def balance(self) -> float:
        """Только для чтения"""
        return self._balance


# Использование
account = Account(1000)
if account.withdraw(500):  # Говорим: "сними деньги"
    print("Success")
else:
    print("Insufficient funds")
```

---

### 2. Управление заказом

**Плохо — Ask:**

```python
class Order:
    def __init__(self):
        self.status = "pending"
        self.items = []
        self.total = 0


def process_order(order: Order):
    # Много вопросов к объекту
    if order.status == "pending":
        if len(order.items) > 0:
            if order.total > 0:
                order.status = "processing"
                # Отправка уведомления
                send_notification(order)
            else:
                raise ValueError("Order total must be positive")
        else:
            raise ValueError("Order must have items")
    else:
        raise ValueError("Order is not pending")


def cancel_order(order: Order):
    if order.status in ["pending", "processing"]:
        order.status = "cancelled"
        # Возврат товаров
        for item in order.items:
            return_to_inventory(item)
    else:
        raise ValueError("Cannot cancel order")
```

**Хорошо — Tell:**

```python
class Order:
    def __init__(self):
        self._status = "pending"
        self._items = []

    def add_item(self, item):
        if self._status != "pending":
            raise ValueError("Cannot modify non-pending order")
        self._items.append(item)

    def process(self):
        """Говорим: 'обработай заказ'"""
        self._validate_for_processing()
        self._status = "processing"
        self._send_notification()

    def cancel(self):
        """Говорим: 'отмени заказ'"""
        if self._status not in ["pending", "processing"]:
            raise ValueError("Cannot cancel order")
        self._status = "cancelled"
        self._return_items_to_inventory()

    def _validate_for_processing(self):
        if self._status != "pending":
            raise ValueError("Order is not pending")
        if not self._items:
            raise ValueError("Order must have items")
        if self.total <= 0:
            raise ValueError("Order total must be positive")

    def _send_notification(self):
        # Внутренняя логика
        pass

    def _return_items_to_inventory(self):
        for item in self._items:
            item.return_to_inventory()

    @property
    def total(self) -> float:
        return sum(item.price for item in self._items)


# Использование — чистый Tell
order = Order()
order.add_item(item)
order.process()  # Говорим, что делать
```

---

### 3. Валидация пользователя

**Плохо — Ask:**

```python
class User:
    def __init__(self, name: str, email: str, age: int):
        self.name = name
        self.email = email
        self.age = age
        self.is_active = True


def can_purchase_alcohol(user: User) -> bool:
    # Спрашиваем и решаем снаружи
    if user.is_active:
        if user.age >= 21:
            return True
    return False


def can_access_premium_content(user: User) -> bool:
    # Дублирование логики проверки активности
    if user.is_active:
        if user.subscription_type == "premium":
            return True
    return False
```

**Хорошо — Tell:**

```python
class User:
    def __init__(self, name: str, email: str, age: int):
        self._name = name
        self._email = email
        self._age = age
        self._is_active = True
        self._subscription_type = "free"

    def can_purchase_alcohol(self) -> bool:
        """Объект сам знает свои правила"""
        return self._is_active and self._age >= 21

    def can_access_premium_content(self) -> bool:
        """Логика в одном месте"""
        return self._is_active and self._subscription_type == "premium"

    def upgrade_to_premium(self):
        """Говорим: 'перейди на премиум'"""
        if not self._is_active:
            raise ValueError("Inactive users cannot upgrade")
        self._subscription_type = "premium"

    def deactivate(self):
        """Говорим: 'деактивируй аккаунт'"""
        self._is_active = False


# Использование
user = User("Alice", "alice@example.com", 25)

if user.can_purchase_alcohol():
    process_alcohol_sale(user)

user.upgrade_to_premium()  # Tell, not ask
```

---

### 4. Коллекции и агрегаты

**Плохо — Ask:**

```python
class ShoppingCart:
    def __init__(self):
        self.items = []


def calculate_discount(cart: ShoppingCart) -> float:
    # Лезем внутрь корзины
    total = sum(item.price * item.quantity for item in cart.items)

    # Логика скидок снаружи
    if total > 1000:
        return total * 0.1
    elif total > 500:
        return total * 0.05
    return 0


def get_expensive_items(cart: ShoppingCart):
    # Опять лезем внутрь
    return [item for item in cart.items if item.price > 100]
```

**Хорошо — Tell:**

```python
class ShoppingCart:
    def __init__(self):
        self._items = []

    def add_item(self, item):
        """Tell: добавь товар"""
        self._items.append(item)

    def remove_item(self, item_id: int):
        """Tell: удали товар"""
        self._items = [i for i in self._items if i.id != item_id]

    @property
    def total(self) -> float:
        return sum(item.price * item.quantity for item in self._items)

    def calculate_discount(self) -> float:
        """Логика скидок внутри корзины"""
        total = self.total
        if total > 1000:
            return total * 0.1
        elif total > 500:
            return total * 0.05
        return 0

    def get_expensive_items(self, threshold: float = 100):
        """Запрос данных — допустимо, когда нужно вернуть данные"""
        return [item for item in self._items if item.price > threshold]

    def apply_discount_to_expensive_items(self, discount_rate: float):
        """Tell: примени скидку к дорогим товарам"""
        for item in self._items:
            if item.price > 100:
                item.apply_discount(discount_rate)


# Использование
cart = ShoppingCart()
cart.add_item(item)
discount = cart.calculate_discount()  # Корзина сама считает
cart.apply_discount_to_expensive_items(0.1)  # Tell
```

---

### 5. Состояние объекта (State Pattern)

**Плохо — Ask состояние:**

```python
class Document:
    def __init__(self):
        self.state = "draft"


def publish(doc: Document):
    if doc.state == "draft":
        doc.state = "pending_review"
        notify_reviewers(doc)
    elif doc.state == "pending_review":
        doc.state = "published"
        notify_subscribers(doc)
    elif doc.state == "published":
        raise ValueError("Already published")


def reject(doc: Document):
    if doc.state == "pending_review":
        doc.state = "draft"
        notify_author(doc)
    else:
        raise ValueError("Can only reject from review")
```

**Хорошо — Tell с инкапсуляцией состояния:**

```python
from abc import ABC, abstractmethod


class DocumentState(ABC):
    @abstractmethod
    def publish(self, document: "Document"):
        pass

    @abstractmethod
    def reject(self, document: "Document"):
        pass


class DraftState(DocumentState):
    def publish(self, document: "Document"):
        document._state = PendingReviewState()
        document._notify_reviewers()

    def reject(self, document: "Document"):
        raise ValueError("Cannot reject draft")


class PendingReviewState(DocumentState):
    def publish(self, document: "Document"):
        document._state = PublishedState()
        document._notify_subscribers()

    def reject(self, document: "Document"):
        document._state = DraftState()
        document._notify_author()


class PublishedState(DocumentState):
    def publish(self, document: "Document"):
        raise ValueError("Already published")

    def reject(self, document: "Document"):
        raise ValueError("Cannot reject published")


class Document:
    def __init__(self):
        self._state: DocumentState = DraftState()

    def publish(self):
        """Tell: опубликуй"""
        self._state.publish(self)

    def reject(self):
        """Tell: отклони"""
        self._state.reject(self)

    def _notify_reviewers(self):
        pass

    def _notify_subscribers(self):
        pass

    def _notify_author(self):
        pass


# Использование — чистый Tell
doc = Document()
doc.publish()  # Переход draft -> pending_review
doc.publish()  # Переход pending_review -> published
```

---

## Когда "Ask" допустим

### 1. Отображение данных (View)

```python
# Получение данных для отображения — это нормально
user = get_user(user_id)
print(f"Name: {user.name}")
print(f"Email: {user.email}")
```

### 2. Проверка прав доступа

```python
# Иногда нужно спросить перед выполнением
if user.has_permission("edit"):
    show_edit_button()
```

### 3. Сериализация

```python
# Для JSON/XML нужно получить данные
data = {
    "id": user.id,
    "name": user.name,
    "email": user.email,
}
json.dumps(data)
```

---

## Best Practices

1. **Размещайте поведение рядом с данными** — методы в том же классе, что и данные
2. **Используйте приватные атрибуты** — `_balance` вместо `balance`
3. **Возвращайте результат операции** — `bool` или объект результата
4. **Исключения для ошибок** — не возвращайте коды ошибок
5. **Один публичный метод = одна команда** — атомарные операции

## Типичные ошибки

1. **Анемичные модели** — классы только с геттерами/сеттерами
2. **Процедурный код** — вся логика в сервисах, не в объектах
3. **Чрезмерное применение** — иногда Ask необходим
4. **Feature Envy** — метод больше работает с чужими данными

---

## Связь с другими принципами

| Принцип | Связь |
|---------|-------|
| Encapsulation | Tell защищает внутреннее состояние |
| Law of Demeter | Оба направлены на уменьшение связности |
| Single Responsibility | Tell помогает локализовать логику |
| Information Hiding | Tell скрывает детали реализации |

---

## Feature Envy — антипаттерн

**Признак нарушения Tell, Don't Ask:**

```python
class OrderService:
    def calculate_total(self, order):
        # Этот метод "завидует" данным Order
        total = 0
        for item in order.items:
            total += item.price * item.quantity

        if order.customer.is_premium:
            total *= 0.9

        if order.shipping.is_express:
            total += 20

        return total


# Решение — переместить логику в Order
class Order:
    def calculate_total(self) -> float:
        total = sum(item.subtotal for item in self._items)
        total = self._apply_customer_discount(total)
        total = self._add_shipping_cost(total)
        return total
```

---

## Резюме

Tell, Don't Ask учит нас:
- Объекты должны отвечать за своё поведение
- Логика должна быть рядом с данными
- Клиенты командуют, объекты решают
- Инкапсуляция важнее удобства доступа

> "Procedural code gets information then makes decisions. Object-oriented code tells objects to do things." — Martin Fowler
