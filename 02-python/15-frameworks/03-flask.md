# Flask

Микрофреймворк — минимум из коробки, максимум гибкости.

```bash
pip install flask
flask run
```

---

## Минимальное приложение

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    return {"message": "Hello"}

@app.route("/users/<int:user_id>")
def get_user(user_id):
    return {"user_id": user_id}
```

---

## Методы HTTP

```python
from flask import request

@app.route("/users", methods=["GET", "POST"])
def users():
    if request.method == "POST":
        return {"created": request.json}, 201
    return {"users": []}
```

---

## Request данные

```python
# Query: /search?q=python
q = request.args.get("q")

# JSON body
data = request.json

# Form
name = request.form.get("name")

# Files
file = request.files.get("avatar")
```

---

## Blueprints

```python
# blueprints/users.py
from flask import Blueprint

users_bp = Blueprint("users", __name__, url_prefix="/users")

@users_bp.route("/")
def list_users():
    return {"users": []}

# app.py
app.register_blueprint(users_bp)
```

---

## Error Handlers

```python
@app.errorhandler(404)
def not_found(error):
    return {"error": "Not found"}, 404

# Custom
class APIError(Exception):
    def __init__(self, message, status=400):
        self.message = message
        self.status = status

@app.errorhandler(APIError)
def handle_error(e):
    return {"error": e.message}, e.status
```

---

## Context

```python
from flask import g

@app.before_request
def before():
    g.db = get_db()

@app.teardown_request
def teardown(exc):
    if hasattr(g, 'db'):
        g.db.close()
```

---

## Flask-SQLAlchemy

```python
from flask_sqlalchemy import SQLAlchemy

app.config["SQLALCHEMY_DATABASE_URI"] = "sqlite:///app.db"
db = SQLAlchemy(app)

class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100))

User.query.all()
User.query.filter_by(name="Анна").first()
```

---

## Резюме

| Фича | Описание |
|------|----------|
| Blueprints | Модульность |
| g, context | Данные запроса |
| Extensions | Flask-SQLAlchemy, Flask-Login... |

**Когда:** Простые API, прототипы, гибкость важнее.

---

## Сравнение фреймворков

| | FastAPI | Django | Flask |
|---|---------|--------|-------|
| Тип | Async API | Full-stack | Micro |
| ORM | Нет (SQLAlchemy) | Встроен | Extension |
| Админка | Нет | Встроена | Extension |
| Async | Да | Частично | Нет |
| Скорость | Высокая | Средняя | Средняя |
| Когда | API, микросервисы | Большие проекты | Простые API |
