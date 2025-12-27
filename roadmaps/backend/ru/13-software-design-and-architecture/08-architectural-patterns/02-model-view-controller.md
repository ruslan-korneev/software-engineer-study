# Model-View-Controller (MVC)

## Что такое MVC?

Model-View-Controller (MVC) — это архитектурный паттерн, который разделяет приложение на три взаимосвязанных компонента: Model (модель), View (представление) и Controller (контроллер). Паттерн был впервые описан в 1979 году Трюгве Реенскаугом для языка Smalltalk.

```
┌─────────────────────────────────────────────────────────────────┐
│                         MVC Pattern                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                        ┌─────────────┐                          │
│           Запрос ────► │  Controller │                          │
│                        └──────┬──────┘                          │
│                               │                                  │
│              ┌────────────────┼────────────────┐                │
│              │                │                │                │
│              ▼                ▼                ▼                │
│       ┌───────────┐    ┌───────────┐    ┌───────────┐          │
│       │   Model   │◄───│ Controller│────►│   View    │          │
│       │  (Данные) │    │ (Логика)  │    │(Отображ.) │          │
│       └───────────┘    └───────────┘    └─────┬─────┘          │
│              │                                 │                │
│              │         Данные                  │                │
│              └─────────────────────────────────┘                │
│                                                                  │
│                        Ответ ◄───────────────────               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Компоненты MVC

### Model (Модель)

Model отвечает за:
- Хранение данных и бизнес-логики
- Работу с базой данных
- Валидацию данных
- Уведомление об изменениях состояния

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional, List
import re


@dataclass
class User:
    """
    Model представляет данные и бизнес-логику.
    Не знает о View или Controller.
    """
    id: Optional[int] = None
    email: str = ""
    username: str = ""
    password_hash: str = ""
    created_at: datetime = field(default_factory=datetime.now)
    is_active: bool = True

    def validate(self) -> List[str]:
        """Валидация данных модели"""
        errors = []

        if not self.email:
            errors.append("Email is required")
        elif not self._is_valid_email(self.email):
            errors.append("Invalid email format")

        if not self.username:
            errors.append("Username is required")
        elif len(self.username) < 3:
            errors.append("Username must be at least 3 characters")

        return errors

    def _is_valid_email(self, email: str) -> bool:
        pattern = r'^[\w\.-]+@[\w\.-]+\.\w+$'
        return bool(re.match(pattern, email))

    def activate(self) -> None:
        """Бизнес-логика: активация пользователя"""
        self.is_active = True

    def deactivate(self) -> None:
        """Бизнес-логика: деактивация пользователя"""
        self.is_active = False


class UserRepository:
    """
    Часть Model слоя — работа с хранилищем данных.
    """

    def __init__(self, database):
        self._db = database
        self._observers = []

    def add_observer(self, observer):
        """Добавить наблюдателя для уведомлений об изменениях"""
        self._observers.append(observer)

    def _notify_observers(self, event: str, data):
        for observer in self._observers:
            observer.on_model_changed(event, data)

    def save(self, user: User) -> User:
        """Сохранить пользователя"""
        if user.id is None:
            user.id = self._db.insert("users", user.__dict__)
            self._notify_observers("user_created", user)
        else:
            self._db.update("users", user.id, user.__dict__)
            self._notify_observers("user_updated", user)
        return user

    def find_by_id(self, user_id: int) -> Optional[User]:
        """Найти пользователя по ID"""
        data = self._db.find_one("users", {"id": user_id})
        if data:
            return User(**data)
        return None

    def find_by_email(self, email: str) -> Optional[User]:
        """Найти пользователя по email"""
        data = self._db.find_one("users", {"email": email})
        if data:
            return User(**data)
        return None

    def find_all(self, filters: dict = None) -> List[User]:
        """Получить список пользователей"""
        data_list = self._db.find_many("users", filters or {})
        return [User(**data) for data in data_list]

    def delete(self, user_id: int) -> bool:
        """Удалить пользователя"""
        result = self._db.delete("users", user_id)
        if result:
            self._notify_observers("user_deleted", {"id": user_id})
        return result
```

### View (Представление)

View отвечает за:
- Отображение данных пользователю
- Форматирование вывода
- Не содержит бизнес-логики

```python
from abc import ABC, abstractmethod
from typing import List


class View(ABC):
    """Базовый класс для представлений"""

    @abstractmethod
    def render(self, data) -> str:
        pass


class UserListView(View):
    """Представление списка пользователей"""

    def render(self, users: List[User]) -> str:
        if not users:
            return "<p>No users found</p>"

        html = "<table><thead><tr>"
        html += "<th>ID</th><th>Username</th><th>Email</th><th>Status</th>"
        html += "</tr></thead><tbody>"

        for user in users:
            status = "Active" if user.is_active else "Inactive"
            html += f"<tr>"
            html += f"<td>{user.id}</td>"
            html += f"<td>{user.username}</td>"
            html += f"<td>{user.email}</td>"
            html += f"<td>{status}</td>"
            html += f"</tr>"

        html += "</tbody></table>"
        return html


class UserDetailView(View):
    """Представление деталей пользователя"""

    def render(self, user: User) -> str:
        if not user:
            return "<p>User not found</p>"

        status = "Active" if user.is_active else "Inactive"
        html = f"""
        <div class="user-detail">
            <h2>{user.username}</h2>
            <p><strong>Email:</strong> {user.email}</p>
            <p><strong>Status:</strong> {status}</p>
            <p><strong>Member since:</strong> {user.created_at.strftime('%Y-%m-%d')}</p>
        </div>
        """
        return html


class UserFormView(View):
    """Представление формы пользователя"""

    def render(self, user: User = None, errors: List[str] = None) -> str:
        user = user or User()
        errors = errors or []

        error_html = ""
        if errors:
            error_html = "<ul class='errors'>"
            error_html += "".join(f"<li>{e}</li>" for e in errors)
            error_html += "</ul>"

        html = f"""
        {error_html}
        <form method="POST" action="/users">
            <input type="hidden" name="id" value="{user.id or ''}">
            <div>
                <label>Username:</label>
                <input type="text" name="username" value="{user.username}">
            </div>
            <div>
                <label>Email:</label>
                <input type="email" name="email" value="{user.email}">
            </div>
            <button type="submit">Save</button>
        </form>
        """
        return html


class JsonUserView(View):
    """JSON представление для API"""

    def render(self, data) -> str:
        import json

        if isinstance(data, list):
            return json.dumps([self._user_to_dict(u) for u in data])
        elif isinstance(data, User):
            return json.dumps(self._user_to_dict(data))
        else:
            return json.dumps(data)

    def _user_to_dict(self, user: User) -> dict:
        return {
            "id": user.id,
            "username": user.username,
            "email": user.email,
            "is_active": user.is_active,
            "created_at": user.created_at.isoformat()
        }
```

### Controller (Контроллер)

Controller отвечает за:
- Обработку входящих запросов
- Взаимодействие с Model
- Выбор подходящего View
- Передачу данных между Model и View

```python
class UserController:
    """
    Controller обрабатывает запросы и координирует Model и View.
    """

    def __init__(self, user_repository: UserRepository):
        self._repository = user_repository
        self._list_view = UserListView()
        self._detail_view = UserDetailView()
        self._form_view = UserFormView()
        self._json_view = JsonUserView()

    def index(self, request) -> str:
        """GET /users — список пользователей"""
        filters = {}

        # Фильтрация по параметрам запроса
        if request.get("status") == "active":
            filters["is_active"] = True
        elif request.get("status") == "inactive":
            filters["is_active"] = False

        users = self._repository.find_all(filters)

        # Выбор View в зависимости от формата
        if request.get("format") == "json":
            return self._json_view.render(users)
        return self._list_view.render(users)

    def show(self, request, user_id: int) -> str:
        """GET /users/{id} — детали пользователя"""
        user = self._repository.find_by_id(user_id)

        if not user:
            raise NotFoundError(f"User {user_id} not found")

        if request.get("format") == "json":
            return self._json_view.render(user)
        return self._detail_view.render(user)

    def new(self, request) -> str:
        """GET /users/new — форма создания"""
        return self._form_view.render()

    def create(self, request) -> str:
        """POST /users — создание пользователя"""
        user = User(
            username=request.get("username", ""),
            email=request.get("email", "")
        )

        # Валидация
        errors = user.validate()
        if errors:
            return self._form_view.render(user, errors)

        # Проверка уникальности email
        existing = self._repository.find_by_email(user.email)
        if existing:
            errors.append("Email already exists")
            return self._form_view.render(user, errors)

        # Сохранение
        self._repository.save(user)

        # Редирект на список
        return redirect("/users")

    def edit(self, request, user_id: int) -> str:
        """GET /users/{id}/edit — форма редактирования"""
        user = self._repository.find_by_id(user_id)

        if not user:
            raise NotFoundError(f"User {user_id} not found")

        return self._form_view.render(user)

    def update(self, request, user_id: int) -> str:
        """PUT /users/{id} — обновление пользователя"""
        user = self._repository.find_by_id(user_id)

        if not user:
            raise NotFoundError(f"User {user_id} not found")

        # Обновление данных
        user.username = request.get("username", user.username)
        user.email = request.get("email", user.email)

        # Валидация
        errors = user.validate()
        if errors:
            return self._form_view.render(user, errors)

        self._repository.save(user)
        return redirect(f"/users/{user_id}")

    def delete(self, request, user_id: int) -> str:
        """DELETE /users/{id} — удаление пользователя"""
        success = self._repository.delete(user_id)

        if not success:
            raise NotFoundError(f"User {user_id} not found")

        if request.get("format") == "json":
            return self._json_view.render({"success": True})
        return redirect("/users")

    def activate(self, request, user_id: int) -> str:
        """POST /users/{id}/activate — активация пользователя"""
        user = self._repository.find_by_id(user_id)

        if not user:
            raise NotFoundError(f"User {user_id} not found")

        user.activate()
        self._repository.save(user)

        return redirect(f"/users/{user_id}")

    def deactivate(self, request, user_id: int) -> str:
        """POST /users/{id}/deactivate — деактивация пользователя"""
        user = self._repository.find_by_id(user_id)

        if not user:
            raise NotFoundError(f"User {user_id} not found")

        user.deactivate()
        self._repository.save(user)

        return redirect(f"/users/{user_id}")
```

## Вариации MVC

### MVP (Model-View-Presenter)

В MVP View пассивен и только отображает данные. Presenter управляет всей логикой представления.

```
┌─────────────────────────────────────────────────────────────────┐
│                         MVP Pattern                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌───────────┐         ┌───────────┐         ┌───────────┐    │
│   │   Model   │◄────────│ Presenter │────────►│   View    │    │
│   │  (Данные) │         │ (Логика)  │         │(Пассивное)│    │
│   └───────────┘         └─────┬─────┘         └─────┬─────┘    │
│                               │                     │          │
│                               │   User Events       │          │
│                               │◄────────────────────┘          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

```python
from abc import ABC, abstractmethod


class UserViewInterface(ABC):
    """Интерфейс View для MVP"""

    @abstractmethod
    def show_users(self, users: List[dict]) -> None:
        pass

    @abstractmethod
    def show_error(self, message: str) -> None:
        pass

    @abstractmethod
    def show_loading(self) -> None:
        pass

    @abstractmethod
    def hide_loading(self) -> None:
        pass

    @abstractmethod
    def get_search_query(self) -> str:
        pass


class UserPresenter:
    """
    Presenter в MVP — управляет View и Model.
    """

    def __init__(self, view: UserViewInterface, repository: UserRepository):
        self._view = view
        self._repository = repository

    def load_users(self) -> None:
        """Загрузить список пользователей"""
        self._view.show_loading()

        try:
            users = self._repository.find_all()
            user_data = [
                {
                    "id": u.id,
                    "username": u.username,
                    "email": u.email,
                    "status": "Active" if u.is_active else "Inactive"
                }
                for u in users
            ]
            self._view.show_users(user_data)
        except Exception as e:
            self._view.show_error(str(e))
        finally:
            self._view.hide_loading()

    def search_users(self) -> None:
        """Поиск пользователей"""
        query = self._view.get_search_query()
        self._view.show_loading()

        try:
            users = self._repository.find_all({"username__contains": query})
            user_data = [
                {
                    "id": u.id,
                    "username": u.username,
                    "email": u.email
                }
                for u in users
            ]
            self._view.show_users(user_data)
        except Exception as e:
            self._view.show_error(str(e))
        finally:
            self._view.hide_loading()


class ConsoleUserView(UserViewInterface):
    """Консольная реализация View"""

    def __init__(self):
        self._search_query = ""

    def show_users(self, users: List[dict]) -> None:
        print("\n=== Users ===")
        for user in users:
            print(f"[{user['id']}] {user['username']} - {user['email']}")

    def show_error(self, message: str) -> None:
        print(f"\nError: {message}")

    def show_loading(self) -> None:
        print("Loading...")

    def hide_loading(self) -> None:
        print("Done.")

    def get_search_query(self) -> str:
        return self._search_query

    def set_search_query(self, query: str) -> None:
        self._search_query = query
```

### MVVM (Model-View-ViewModel)

В MVVM ViewModel предоставляет данные для View через привязку данных (data binding).

```
┌─────────────────────────────────────────────────────────────────┐
│                        MVVM Pattern                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌───────────┐         ┌───────────┐         ┌───────────┐    │
│   │   Model   │◄────────│ ViewModel │◄═══════►│   View    │    │
│   │  (Данные) │         │ (Состоян.)│  Binding │(Отображ.)│    │
│   └───────────┘         └───────────┘         └───────────┘    │
│                                                                  │
│   ═══════► Data Binding (двусторонняя привязка)                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

```python
from typing import Callable, Any


class Observable:
    """Базовый класс для реактивных свойств"""

    def __init__(self, initial_value=None):
        self._value = initial_value
        self._subscribers: List[Callable] = []

    @property
    def value(self):
        return self._value

    @value.setter
    def value(self, new_value):
        if self._value != new_value:
            self._value = new_value
            self._notify()

    def subscribe(self, callback: Callable) -> None:
        self._subscribers.append(callback)

    def _notify(self) -> None:
        for callback in self._subscribers:
            callback(self._value)


class UserViewModel:
    """
    ViewModel в MVVM — хранит состояние и логику представления.
    """

    def __init__(self, repository: UserRepository):
        self._repository = repository

        # Observable свойства
        self.users = Observable([])
        self.selected_user = Observable(None)
        self.search_query = Observable("")
        self.is_loading = Observable(False)
        self.error_message = Observable("")

        # Команды (actions)
        self.load_command = self._create_load_command()
        self.search_command = self._create_search_command()
        self.select_command = self._create_select_command()

    def _create_load_command(self):
        def execute():
            self.is_loading.value = True
            self.error_message.value = ""

            try:
                users = self._repository.find_all()
                self.users.value = [
                    {
                        "id": u.id,
                        "username": u.username,
                        "email": u.email,
                        "is_active": u.is_active
                    }
                    for u in users
                ]
            except Exception as e:
                self.error_message.value = str(e)
            finally:
                self.is_loading.value = False

        return execute

    def _create_search_command(self):
        def execute():
            query = self.search_query.value
            if not query:
                self.load_command()
                return

            self.is_loading.value = True
            try:
                users = self._repository.find_all()
                filtered = [
                    {
                        "id": u.id,
                        "username": u.username,
                        "email": u.email,
                        "is_active": u.is_active
                    }
                    for u in users
                    if query.lower() in u.username.lower()
                ]
                self.users.value = filtered
            except Exception as e:
                self.error_message.value = str(e)
            finally:
                self.is_loading.value = False

        return execute

    def _create_select_command(self):
        def execute(user_id: int):
            user = self._repository.find_by_id(user_id)
            if user:
                self.selected_user.value = {
                    "id": user.id,
                    "username": user.username,
                    "email": user.email,
                    "is_active": user.is_active,
                    "created_at": user.created_at.isoformat()
                }

        return execute


# Пример использования с привязкой к UI
class UserListUI:
    """Простой UI с привязкой к ViewModel"""

    def __init__(self, view_model: UserViewModel):
        self._vm = view_model

        # Подписка на изменения
        self._vm.users.subscribe(self._on_users_changed)
        self._vm.is_loading.subscribe(self._on_loading_changed)
        self._vm.error_message.subscribe(self._on_error_changed)

    def _on_users_changed(self, users: List[dict]) -> None:
        print("\n=== Users Updated ===")
        for user in users:
            status = "Active" if user["is_active"] else "Inactive"
            print(f"  [{user['id']}] {user['username']} ({status})")

    def _on_loading_changed(self, is_loading: bool) -> None:
        if is_loading:
            print("Loading...")
        else:
            print("Ready.")

    def _on_error_changed(self, error: str) -> None:
        if error:
            print(f"ERROR: {error}")

    def search(self, query: str) -> None:
        self._vm.search_query.value = query
        self._vm.search_command()

    def load(self) -> None:
        self._vm.load_command()
```

## MVC в веб-фреймворках

### Flask (Python)

```python
from flask import Flask, render_template, request, redirect, url_for, jsonify

app = Flask(__name__)


# Model
class Article:
    _articles = []
    _next_id = 1

    def __init__(self, title: str, content: str, article_id: int = None):
        self.id = article_id or Article._next_id
        if article_id is None:
            Article._next_id += 1
        self.title = title
        self.content = content

    def save(self):
        existing = next((a for a in Article._articles if a.id == self.id), None)
        if existing:
            existing.title = self.title
            existing.content = self.content
        else:
            Article._articles.append(self)
        return self

    @classmethod
    def find_all(cls):
        return cls._articles

    @classmethod
    def find_by_id(cls, article_id: int):
        return next((a for a in cls._articles if a.id == article_id), None)

    @classmethod
    def delete(cls, article_id: int):
        article = cls.find_by_id(article_id)
        if article:
            cls._articles.remove(article)
            return True
        return False


# Controller (routes)
@app.route('/articles')
def article_list():
    """GET /articles"""
    articles = Article.find_all()

    if request.args.get('format') == 'json':
        return jsonify([{'id': a.id, 'title': a.title} for a in articles])

    # View (template)
    return render_template('articles/list.html', articles=articles)


@app.route('/articles/<int:article_id>')
def article_detail(article_id):
    """GET /articles/{id}"""
    article = Article.find_by_id(article_id)

    if not article:
        return "Not Found", 404

    if request.args.get('format') == 'json':
        return jsonify({
            'id': article.id,
            'title': article.title,
            'content': article.content
        })

    return render_template('articles/detail.html', article=article)


@app.route('/articles/new')
def article_new():
    """GET /articles/new"""
    return render_template('articles/form.html', article=None)


@app.route('/articles', methods=['POST'])
def article_create():
    """POST /articles"""
    article = Article(
        title=request.form.get('title', ''),
        content=request.form.get('content', '')
    )
    article.save()
    return redirect(url_for('article_detail', article_id=article.id))


@app.route('/articles/<int:article_id>/edit')
def article_edit(article_id):
    """GET /articles/{id}/edit"""
    article = Article.find_by_id(article_id)
    if not article:
        return "Not Found", 404
    return render_template('articles/form.html', article=article)


@app.route('/articles/<int:article_id>', methods=['PUT', 'POST'])
def article_update(article_id):
    """PUT /articles/{id}"""
    article = Article.find_by_id(article_id)
    if not article:
        return "Not Found", 404

    article.title = request.form.get('title', article.title)
    article.content = request.form.get('content', article.content)
    article.save()

    return redirect(url_for('article_detail', article_id=article.id))


@app.route('/articles/<int:article_id>', methods=['DELETE'])
def article_delete(article_id):
    """DELETE /articles/{id}"""
    if Article.delete(article_id):
        return redirect(url_for('article_list'))
    return "Not Found", 404
```

### Django (Python)

```python
# models.py (Model)
from django.db import models


class Article(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_published = models.BooleanField(default=False)

    class Meta:
        ordering = ['-created_at']

    def publish(self):
        """Бизнес-логика: публикация статьи"""
        self.is_published = True
        self.save()

    def unpublish(self):
        """Бизнес-логика: снятие с публикации"""
        self.is_published = False
        self.save()


# views.py (Controller в терминах Django)
from django.shortcuts import render, get_object_or_404, redirect
from django.http import JsonResponse
from .models import Article
from .forms import ArticleForm


def article_list(request):
    """GET /articles/"""
    articles = Article.objects.all()

    if request.GET.get('published') == 'true':
        articles = articles.filter(is_published=True)

    if request.headers.get('Accept') == 'application/json':
        data = [{'id': a.id, 'title': a.title} for a in articles]
        return JsonResponse(data, safe=False)

    return render(request, 'articles/list.html', {'articles': articles})


def article_detail(request, article_id):
    """GET /articles/{id}/"""
    article = get_object_or_404(Article, pk=article_id)

    if request.headers.get('Accept') == 'application/json':
        return JsonResponse({
            'id': article.id,
            'title': article.title,
            'content': article.content,
            'is_published': article.is_published
        })

    return render(request, 'articles/detail.html', {'article': article})


def article_create(request):
    """GET/POST /articles/new/"""
    if request.method == 'POST':
        form = ArticleForm(request.POST)
        if form.is_valid():
            article = form.save()
            return redirect('article_detail', article_id=article.id)
    else:
        form = ArticleForm()

    return render(request, 'articles/form.html', {'form': form})


def article_update(request, article_id):
    """GET/POST /articles/{id}/edit/"""
    article = get_object_or_404(Article, pk=article_id)

    if request.method == 'POST':
        form = ArticleForm(request.POST, instance=article)
        if form.is_valid():
            form.save()
            return redirect('article_detail', article_id=article.id)
    else:
        form = ArticleForm(instance=article)

    return render(request, 'articles/form.html', {
        'form': form,
        'article': article
    })


def article_delete(request, article_id):
    """POST /articles/{id}/delete/"""
    article = get_object_or_404(Article, pk=article_id)
    article.delete()
    return redirect('article_list')


def article_publish(request, article_id):
    """POST /articles/{id}/publish/"""
    article = get_object_or_404(Article, pk=article_id)
    article.publish()
    return redirect('article_detail', article_id=article.id)


# urls.py (роутинг)
from django.urls import path
from . import views

urlpatterns = [
    path('articles/', views.article_list, name='article_list'),
    path('articles/new/', views.article_create, name='article_create'),
    path('articles/<int:article_id>/', views.article_detail, name='article_detail'),
    path('articles/<int:article_id>/edit/', views.article_update, name='article_update'),
    path('articles/<int:article_id>/delete/', views.article_delete, name='article_delete'),
    path('articles/<int:article_id>/publish/', views.article_publish, name='article_publish'),
]
```

## Плюсы и минусы MVC

### Плюсы

1. **Разделение ответственности** — каждый компонент имеет чёткую роль
2. **Повторное использование** — Model и View можно переиспользовать
3. **Параллельная разработка** — команды могут работать независимо
4. **Тестируемость** — компоненты легко тестировать изолированно
5. **Поддерживаемость** — изменения в одном компоненте не затрагивают другие

### Минусы

1. **Сложность** — для простых приложений MVC может быть избыточен
2. **"Толстые" контроллеры** — логика часто скапливается в контроллерах
3. **Множество файлов** — даже простая функция требует нескольких файлов
4. **Кривая обучения** — требует понимания взаимодействия компонентов
5. **Не подходит для всего** — не оптимален для real-time приложений

## Когда использовать

### Используйте MVC когда:

- Веб-приложение с серверным рендерингом
- Нужно разделение между UI и бизнес-логикой
- Команда работает параллельно над разными частями
- Приложение будет расти и развиваться
- Нужна поддержка нескольких View (web, API, mobile)

### Не используйте MVC когда:

- Очень простое приложение (несколько страниц)
- Real-time приложения с сложным состоянием на клиенте
- Одностраничные приложения (SPA) — лучше использовать MVVM
- Микросервисы с одной функцией

## Best Practices

1. **Тонкие контроллеры** — выносите логику в Model и сервисы
2. **Не обращайтесь к БД из View** — только через Model
3. **Используйте DTO** — не передавайте доменные модели во View напрямую
4. **Один контроллер — один ресурс** — следуйте RESTful принципам
5. **Валидация в Model** — бизнес-правила должны быть в Model

## Заключение

MVC — проверенный временем паттерн, который помогает структурировать код веб-приложений. Его вариации (MVP, MVVM) адаптируют идеи MVC для разных контекстов. Выбор конкретного паттерна зависит от типа приложения и требований проекта.
