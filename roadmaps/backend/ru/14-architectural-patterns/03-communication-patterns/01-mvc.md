# Model-View-Controller (MVC)

## Определение

**Model-View-Controller (MVC)** — это архитектурный паттерн, который разделяет приложение на три взаимосвязанных компонента:

- **Model (Модель)** — управляет данными, бизнес-логикой и правилами приложения
- **View (Представление)** — отвечает за отображение данных пользователю
- **Controller (Контроллер)** — обрабатывает пользовательский ввод и координирует взаимодействие между Model и View

Этот паттерн был впервые описан в 1979 году Тригве Реенскаугом для языка Smalltalk и с тех пор стал одним из самых распространённых архитектурных паттернов в разработке программного обеспечения.

```
┌─────────────────────────────────────────────────────────────┐
│                         Пользователь                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                        Controller                            │
│  • Принимает пользовательский ввод                          │
│  • Интерпретирует действия                                  │
│  • Вызывает методы Model                                    │
│  • Выбирает View для отображения                            │
└─────────────────────────────────────────────────────────────┘
           │                                    │
           ▼                                    ▼
┌─────────────────────┐              ┌─────────────────────────┐
│       Model         │              │          View           │
│                     │◄─────────────│                         │
│ • Данные            │   запрос     │ • Отображает данные     │
│ • Бизнес-логика     │   данных     │ • Форматирует вывод     │
│ • Валидация         │              │ • Принимает ввод        │
│ • Персистентность   │─────────────►│ • Уведомляет Controller │
│                     │   данные     │                         │
└─────────────────────┘              └─────────────────────────┘
```

## Ключевые характеристики

### 1. Разделение ответственности (Separation of Concerns)

Каждый компонент имеет чётко определённую роль:

```python
# Model - только данные и бизнес-логика
class User:
    def __init__(self, name: str, email: str):
        self.name = name
        self.email = email
        self._validate()

    def _validate(self):
        if not self.email or '@' not in self.email:
            raise ValueError("Invalid email")

    def change_email(self, new_email: str):
        self.email = new_email
        self._validate()

# View - только отображение
class UserView:
    def render_user(self, user: User) -> str:
        return f"User: {user.name} ({user.email})"

    def render_user_list(self, users: list) -> str:
        return "\n".join(self.render_user(u) for u in users)

# Controller - только координация
class UserController:
    def __init__(self, user_repository, view: UserView):
        self.repository = user_repository
        self.view = view

    def show_user(self, user_id: int) -> str:
        user = self.repository.find(user_id)
        return self.view.render_user(user)

    def update_email(self, user_id: int, new_email: str) -> str:
        user = self.repository.find(user_id)
        user.change_email(new_email)
        self.repository.save(user)
        return self.view.render_user(user)
```

### 2. Слабая связанность (Loose Coupling)

Компоненты взаимодействуют через абстракции:

```python
from abc import ABC, abstractmethod

# Абстракции для слабой связанности
class UserRepositoryInterface(ABC):
    @abstractmethod
    def find(self, user_id: int) -> User:
        pass

    @abstractmethod
    def save(self, user: User) -> None:
        pass

class ViewInterface(ABC):
    @abstractmethod
    def render(self, data: dict) -> str:
        pass

# Controller зависит от абстракций, не от конкретных реализаций
class UserController:
    def __init__(self, repository: UserRepositoryInterface, view: ViewInterface):
        self.repository = repository
        self.view = view
```

### 3. Независимость представления

View может быть заменён без изменения Model:

```python
# JSON View
class JsonUserView(ViewInterface):
    def render(self, data: dict) -> str:
        import json
        return json.dumps(data)

# HTML View
class HtmlUserView(ViewInterface):
    def render(self, data: dict) -> str:
        return f"<div class='user'><h1>{data['name']}</h1></div>"

# XML View
class XmlUserView(ViewInterface):
    def render(self, data: dict) -> str:
        return f"<user><name>{data['name']}</name></user>"

# Один Controller, разные View
controller_json = UserController(repository, JsonUserView())
controller_html = UserController(repository, HtmlUserView())
```

### 4. Наблюдатель (Observer Pattern) в классическом MVC

В классической реализации Model уведомляет View об изменениях:

```python
from typing import List, Callable

class Observable:
    def __init__(self):
        self._observers: List[Callable] = []

    def add_observer(self, observer: Callable):
        self._observers.append(observer)

    def notify_observers(self):
        for observer in self._observers:
            observer(self)

class UserModel(Observable):
    def __init__(self, name: str):
        super().__init__()
        self._name = name

    @property
    def name(self):
        return self._name

    @name.setter
    def name(self, value: str):
        self._name = value
        self.notify_observers()  # Уведомляем View об изменении

class UserView:
    def __init__(self, model: UserModel):
        self.model = model
        model.add_observer(self.on_model_changed)

    def on_model_changed(self, model: UserModel):
        print(f"View updated: User name is now '{model.name}'")
```

## Когда использовать

### Идеальные случаи применения

1. **Веб-приложения** — классический use case для MVC
2. **Desktop приложения с GUI** — разделение логики и интерфейса
3. **Приложения с несколькими представлениями одних данных**
4. **Системы, требующие параллельной разработки** — команды могут работать над разными слоями
5. **Приложения с частой сменой UI** при стабильной бизнес-логике

### Примеры из реального мира

```python
# Веб-фреймворк (Django-style)
# urls.py - маршрутизация к Controller
urlpatterns = [
    path('users/', UserController.as_view(), name='user-list'),
    path('users/<int:pk>/', UserDetailController.as_view(), name='user-detail'),
]

# models.py - Model
class User(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField(unique=True)

    def activate(self):
        self.is_active = True
        self.save()

# views.py - Controller (в Django называется View, но по сути Controller)
class UserController(View):
    def get(self, request):
        users = User.objects.all()
        return render(request, 'users/list.html', {'users': users})

# templates/users/list.html - View
# {% for user in users %}
#   <div>{{ user.name }}</div>
# {% endfor %}
```

## Преимущества и недостатки

### Преимущества

| Преимущество | Описание |
|-------------|----------|
| **Разделение ответственности** | Каждый компонент имеет одну задачу |
| **Тестируемость** | Model можно тестировать изолированно |
| **Параллельная разработка** | Команды могут работать независимо |
| **Переиспользование** | Model может использоваться с разными View |
| **Гибкость** | Легко менять UI без изменения логики |
| **Поддерживаемость** | Понятная структура кода |

### Недостатки

| Недостаток | Описание |
|-----------|----------|
| **Сложность** | Избыточен для простых приложений |
| **Толстый Controller** | Тенденция к разрастанию контроллеров |
| **Связанность View-Model** | В классическом MVC View знает о Model |
| **Навигация по коду** | Логика распределена по файлам |
| **Boilerplate** | Много шаблонного кода |

## Примеры реализации

### Веб-приложение на Flask

```python
from flask import Flask, render_template, request, redirect, url_for
from dataclasses import dataclass
from typing import List, Optional

app = Flask(__name__)

# ==================== MODEL ====================
@dataclass
class Task:
    id: int
    title: str
    completed: bool = False

    def toggle(self):
        self.completed = not self.completed

class TaskRepository:
    _tasks: List[Task] = []
    _next_id: int = 1

    @classmethod
    def all(cls) -> List[Task]:
        return cls._tasks

    @classmethod
    def find(cls, task_id: int) -> Optional[Task]:
        return next((t for t in cls._tasks if t.id == task_id), None)

    @classmethod
    def create(cls, title: str) -> Task:
        task = Task(id=cls._next_id, title=title)
        cls._next_id += 1
        cls._tasks.append(task)
        return task

    @classmethod
    def delete(cls, task_id: int) -> bool:
        task = cls.find(task_id)
        if task:
            cls._tasks.remove(task)
            return True
        return False

# ==================== CONTROLLER ====================
@app.route('/')
def index():
    """Controller: список всех задач"""
    tasks = TaskRepository.all()
    return render_template('tasks/index.html', tasks=tasks)

@app.route('/tasks', methods=['POST'])
def create_task():
    """Controller: создание задачи"""
    title = request.form.get('title')
    if title:
        TaskRepository.create(title)
    return redirect(url_for('index'))

@app.route('/tasks/<int:task_id>/toggle', methods=['POST'])
def toggle_task(task_id: int):
    """Controller: переключение статуса"""
    task = TaskRepository.find(task_id)
    if task:
        task.toggle()
    return redirect(url_for('index'))

@app.route('/tasks/<int:task_id>/delete', methods=['POST'])
def delete_task(task_id: int):
    """Controller: удаление задачи"""
    TaskRepository.delete(task_id)
    return redirect(url_for('index'))

# ==================== VIEW (templates/tasks/index.html) ====================
"""
<!DOCTYPE html>
<html>
<head><title>Tasks</title></head>
<body>
    <h1>Task List</h1>

    <form action="{{ url_for('create_task') }}" method="post">
        <input type="text" name="title" placeholder="New task...">
        <button type="submit">Add</button>
    </form>

    <ul>
    {% for task in tasks %}
        <li>
            <form action="{{ url_for('toggle_task', task_id=task.id) }}" method="post" style="display:inline;">
                <button type="submit">{{ '✓' if task.completed else '○' }}</button>
            </form>
            {{ task.title }}
            <form action="{{ url_for('delete_task', task_id=task.id) }}" method="post" style="display:inline;">
                <button type="submit">Delete</button>
            </form>
        </li>
    {% endfor %}
    </ul>
</body>
</html>
"""
```

### API с FastAPI

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional

app = FastAPI()

# ==================== MODEL ====================
class ProductModel(BaseModel):
    id: Optional[int] = None
    name: str
    price: float
    stock: int = 0

    def is_available(self) -> bool:
        return self.stock > 0

    def reduce_stock(self, quantity: int):
        if quantity > self.stock:
            raise ValueError("Insufficient stock")
        self.stock -= quantity

class ProductRepository:
    _products: dict = {}
    _next_id: int = 1

    @classmethod
    def all(cls) -> List[ProductModel]:
        return list(cls._products.values())

    @classmethod
    def find(cls, product_id: int) -> Optional[ProductModel]:
        return cls._products.get(product_id)

    @classmethod
    def save(cls, product: ProductModel) -> ProductModel:
        if product.id is None:
            product.id = cls._next_id
            cls._next_id += 1
        cls._products[product.id] = product
        return product

# ==================== VIEW (Response Schemas) ====================
class ProductResponse(BaseModel):
    id: int
    name: str
    price: float
    available: bool

class ProductListResponse(BaseModel):
    products: List[ProductResponse]
    total: int

def product_to_response(product: ProductModel) -> ProductResponse:
    return ProductResponse(
        id=product.id,
        name=product.name,
        price=product.price,
        available=product.is_available()
    )

# ==================== CONTROLLER ====================
@app.get("/products", response_model=ProductListResponse)
def list_products():
    products = ProductRepository.all()
    return ProductListResponse(
        products=[product_to_response(p) for p in products],
        total=len(products)
    )

@app.get("/products/{product_id}", response_model=ProductResponse)
def get_product(product_id: int):
    product = ProductRepository.find(product_id)
    if not product:
        raise HTTPException(status_code=404, detail="Product not found")
    return product_to_response(product)

@app.post("/products", response_model=ProductResponse)
def create_product(product: ProductModel):
    saved = ProductRepository.save(product)
    return product_to_response(saved)
```

## Best Practices и антипаттерны

### Best Practices

```python
# 1. Тонкие контроллеры - логика в Model или Service
class OrderController:
    def __init__(self, order_service: OrderService):
        self.order_service = order_service

    def create_order(self, request):
        # Controller только делегирует
        order = self.order_service.create_order(
            user_id=request.user_id,
            items=request.items
        )
        return self.view.render(order)

# 2. Model не знает о View (в современных реализациях)
class User:
    # Model содержит только бизнес-логику
    def can_purchase(self, product) -> bool:
        return self.balance >= product.price

# 3. View не содержит логики
class UserView:
    # Только форматирование, никакой бизнес-логики
    def render(self, user, can_purchase: bool):
        return {
            "name": user.name,
            "can_purchase": can_purchase  # Уже вычислено в Controller
        }

# 4. Dependency Injection для тестируемости
class UserController:
    def __init__(
        self,
        repository: UserRepositoryInterface,
        view: UserViewInterface,
        notification_service: NotificationServiceInterface
    ):
        self.repository = repository
        self.view = view
        self.notifications = notification_service
```

### Антипаттерны

```python
# ❌ АНТИПАТТЕРН: Fat Controller (толстый контроллер)
class BadUserController:
    def create_user(self, request):
        # Вся логика в контроллере - ПЛОХО!
        email = request.email.lower().strip()
        if not re.match(r'^[\w\.-]+@[\w\.-]+\.\w+$', email):
            raise ValueError("Invalid email")

        password_hash = hashlib.sha256(request.password.encode()).hexdigest()

        user = User(email=email, password=password_hash)
        self.db.execute("INSERT INTO users ...")

        # Отправка email прямо в контроллере
        smtp = smtplib.SMTP('localhost')
        smtp.send_message(...)

        return {"user": user}

# ✅ ПРАВИЛЬНО: Тонкий контроллер
class GoodUserController:
    def __init__(self, user_service, view):
        self.user_service = user_service
        self.view = view

    def create_user(self, request):
        user = self.user_service.register(request.email, request.password)
        return self.view.render_user(user)


# ❌ АНТИПАТТЕРН: View содержит бизнес-логику
class BadProductView:
    def render(self, product, user):
        # Логика расчёта скидки в View - ПЛОХО!
        if user.is_premium:
            discount = 0.2
        elif user.orders_count > 10:
            discount = 0.1
        else:
            discount = 0

        final_price = product.price * (1 - discount)
        return f"Price: ${final_price}"

# ✅ ПРАВИЛЬНО: View только отображает
class GoodProductView:
    def render(self, product, final_price):
        return f"Price: ${final_price}"


# ❌ АНТИПАТТЕРН: Model знает о HTTP
class BadUser:
    def save(self, request):  # Model зависит от HTTP - ПЛОХО!
        self.name = request.POST['name']
        self.session = request.session

# ✅ ПРАВИЛЬНО: Model независим от инфраструктуры
class GoodUser:
    def __init__(self, name: str, email: str):
        self.name = name
        self.email = email
```

## Связанные паттерны

### MVP (Model-View-Presenter)

```
┌─────────┐     ┌───────────┐     ┌──────┐
│  View   │◄───►│ Presenter │◄───►│Model │
└─────────┘     └───────────┘     └──────┘
     │
     ▼
  Пользователь

Отличие: View полностью пассивен, Presenter управляет всем
```

### MVVM (Model-View-ViewModel)

```
┌─────────┐     ┌───────────┐     ┌──────┐
│  View   │◄═══►│ ViewModel │◄───►│Model │
└─────────┘     └───────────┘     └──────┘
     Data Binding

Отличие: Двустороннее связывание данных между View и ViewModel
```

### Сравнение

| Аспект | MVC | MVP | MVVM |
|--------|-----|-----|------|
| Связь View-Model | View знает о Model | View не знает о Model | Через ViewModel |
| Роль посредника | Controller координирует | Presenter управляет | ViewModel преобразует |
| Тестируемость View | Средняя | Высокая | Высокая |
| Data Binding | Нет | Нет | Да |
| Сложность | Низкая | Средняя | Высокая |

## Ресурсы для изучения

### Книги
- **"Patterns of Enterprise Application Architecture"** — Martin Fowler
- **"Clean Architecture"** — Robert C. Martin
- **"Design Patterns: Elements of Reusable Object-Oriented Software"** — Gang of Four

### Онлайн-ресурсы
- [Original MVC Paper by Trygve Reenskaug](http://heim.ifi.uio.no/~trygver/themes/mvc/mvc-index.html)
- [Martin Fowler - GUI Architectures](https://martinfowler.com/eaaDev/uiArchs.html)
- [Django Documentation - MTV Pattern](https://docs.djangoproject.com/en/stable/faq/general/#django-appears-to-be-a-mvc-framework-but-you-call-the-controller-the-view-and-the-view-the-template-how-come-you-don-t-use-the-standard-names)

### Практика
- Реализовать TODO-приложение с нуля используя MVC
- Рефакторинг существующего кода в MVC структуру
- Сравнить реализации MVC в разных фреймворках (Django, Rails, Spring MVC)

---

**Следующая тема**: [Publish-Subscribe](./02-pub-sub.md)
