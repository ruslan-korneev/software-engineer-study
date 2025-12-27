# Клиент-серверная архитектура (Client-Server Architecture)

## Определение

**Клиент-серверная архитектура** — это распределённый архитектурный стиль, в котором задачи и нагрузка распределяются между поставщиками услуг (серверами) и потребителями услуг (клиентами). Клиент инициирует запросы, сервер обрабатывает их и возвращает результаты.

```
┌─────────────────┐         Request          ┌─────────────────┐
│                 │ ─────────────────────────▶│                 │
│     CLIENT      │                          │     SERVER      │
│   (Consumer)    │         Response         │   (Provider)    │
│                 │ ◀─────────────────────────│                 │
└─────────────────┘                          └─────────────────┘
        │                                            │
        │                                            │
   ┌────▼────┐                               ┌───────▼───────┐
   │   UI    │                               │   Database    │
   │  Logic  │                               │   Business    │
   │  Input  │                               │    Logic      │
   └─────────┘                               └───────────────┘
```

Это один из старейших и наиболее фундаментальных архитектурных паттернов, лежащий в основе современного интернета.

## Ключевые характеристики

### 1. Распределение ролей

| Компонент | Роль | Ответственность |
|-----------|------|-----------------|
| **Client** | Потребитель | Запрос данных, отображение UI, локальная обработка |
| **Server** | Поставщик | Обработка запросов, бизнес-логика, хранение данных |

### 2. Модели взаимодействия

#### 2-tier Architecture (Двухуровневая)
```
┌──────────┐          ┌──────────┐
│  Client  │ ◀──────▶ │  Server  │
│   (UI)   │          │ (DB + BL)│
└──────────┘          └──────────┘

BL = Business Logic
```

#### 3-tier Architecture (Трёхуровневая)
```
┌──────────┐       ┌──────────┐       ┌──────────┐
│  Client  │ ◀───▶ │   App    │ ◀───▶ │ Database │
│   (UI)   │       │  Server  │       │  Server  │
└──────────┘       └──────────┘       └──────────┘
```

#### N-tier Architecture (Многоуровневая)
```
┌────────┐    ┌──────────┐    ┌──────────┐    ┌────────┐    ┌──────┐
│ Client │◀──▶│   Web    │◀──▶│   App    │◀──▶│ Cache  │◀──▶│  DB  │
│        │    │  Server  │    │  Server  │    │ Server │    │      │
└────────┘    └──────────┘    └──────────┘    └────────┘    └──────┘
```

### 3. Типы клиентов

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT TYPES                             │
├───────────────┬───────────────────┬─────────────────────────────┤
│  Thin Client  │  Thick/Fat Client │      Hybrid Client          │
├───────────────┼───────────────────┼─────────────────────────────┤
│ • Минимальная │ • Основная        │ • Баланс между              │
│   логика      │   логика на       │   клиентом и сервером       │
│ • Зависим от  │   клиенте         │ • Offline capabilities      │
│   сервера     │ • Независимость   │ • PWA, SPA                  │
│ • Web browser │ • Desktop apps    │                             │
└───────────────┴───────────────────┴─────────────────────────────┘
```

### 4. Протоколы взаимодействия

- **HTTP/HTTPS** — RESTful APIs, Web applications
- **WebSocket** — Real-time двунаправленная связь
- **gRPC** — Высокопроизводительный RPC
- **GraphQL** — Гибкие запросы данных
- **TCP/UDP** — Низкоуровневое сокетное взаимодействие

## Когда использовать

### Идеальные сценарии

- **Web-приложения** — браузер как клиент, бэкенд как сервер
- **Мобильные приложения** — App + Backend API
- **Desktop приложения с сетевым функционалом**
- **Системы с централизованным хранением данных**
- **Многопользовательские системы** — shared data access
- **Системы с требованием безопасности** — централизованный контроль

### Примеры использования

| Домен | Клиент | Сервер |
|-------|--------|--------|
| Email | Outlook, Gmail UI | Mail server (SMTP, IMAP) |
| Banking | Mobile app, Web | Banking backend + DB |
| Gaming | Game client | Game server |
| E-commerce | Browser/App | E-commerce platform |
| Messaging | WhatsApp, Telegram | Messaging backend |

### Не подходит для

- Полностью автономных приложений
- Peer-to-peer систем (торренты)
- Систем с децентрализованным управлением
- Offline-first приложений без серверной части

## Преимущества и недостатки

### Преимущества

| Преимущество | Описание |
|--------------|----------|
| **Централизация** | Единая точка управления данными и логикой |
| **Безопасность** | Контроль доступа на сервере, защита данных |
| **Масштабируемость** | Сервер можно масштабировать независимо |
| **Maintainability** | Обновления на сервере без изменения клиентов |
| **Resource sharing** | Множество клиентов используют общие ресурсы |
| **Data integrity** | Целостность данных контролируется сервером |

### Недостатки

| Недостаток | Описание |
|------------|----------|
| **Single point of failure** | Падение сервера = недоступность системы |
| **Сетевая зависимость** | Требуется стабильное соединение |
| **Стоимость сервера** | Инфраструктура, поддержка, масштабирование |
| **Latency** | Задержки из-за сетевого взаимодействия |
| **Bottleneck** | Сервер может стать узким местом |
| **Vendor lock-in** | Зависимость от серверной инфраструктуры |

## Примеры реализации

### REST API Server (Python/FastAPI)

```python
# server/main.py
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel
from typing import List, Optional
from datetime import datetime
import uvicorn

app = FastAPI(title="Task Manager API")

# In-memory storage (в реальном проекте — база данных)
tasks_db: dict = {}

# Models
class TaskCreate(BaseModel):
    title: str
    description: Optional[str] = None
    due_date: Optional[datetime] = None

class Task(BaseModel):
    id: int
    title: str
    description: Optional[str]
    completed: bool = False
    created_at: datetime
    due_date: Optional[datetime]

class TaskUpdate(BaseModel):
    title: Optional[str] = None
    description: Optional[str] = None
    completed: Optional[bool] = None

# API Endpoints
@app.post("/tasks", response_model=Task, status_code=201)
def create_task(task: TaskCreate):
    """Создание новой задачи"""
    task_id = len(tasks_db) + 1
    new_task = Task(
        id=task_id,
        title=task.title,
        description=task.description,
        created_at=datetime.utcnow(),
        due_date=task.due_date
    )
    tasks_db[task_id] = new_task
    return new_task

@app.get("/tasks", response_model=List[Task])
def get_tasks(completed: Optional[bool] = None):
    """Получение списка задач с фильтрацией"""
    tasks = list(tasks_db.values())
    if completed is not None:
        tasks = [t for t in tasks if t.completed == completed]
    return tasks

@app.get("/tasks/{task_id}", response_model=Task)
def get_task(task_id: int):
    """Получение задачи по ID"""
    if task_id not in tasks_db:
        raise HTTPException(status_code=404, detail="Task not found")
    return tasks_db[task_id]

@app.patch("/tasks/{task_id}", response_model=Task)
def update_task(task_id: int, update: TaskUpdate):
    """Обновление задачи"""
    if task_id not in tasks_db:
        raise HTTPException(status_code=404, detail="Task not found")

    task = tasks_db[task_id]
    update_data = update.dict(exclude_unset=True)

    for field, value in update_data.items():
        setattr(task, field, value)

    return task

@app.delete("/tasks/{task_id}", status_code=204)
def delete_task(task_id: int):
    """Удаление задачи"""
    if task_id not in tasks_db:
        raise HTTPException(status_code=404, detail="Task not found")
    del tasks_db[task_id]

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Python Client

```python
# client/task_client.py
import httpx
from typing import Optional, List
from dataclasses import dataclass
from datetime import datetime

@dataclass
class Task:
    id: int
    title: str
    description: Optional[str]
    completed: bool
    created_at: datetime
    due_date: Optional[datetime]

class TaskClient:
    """HTTP клиент для взаимодействия с Task Manager API"""

    def __init__(self, base_url: str = "http://localhost:8000"):
        self.base_url = base_url
        self.client = httpx.Client(timeout=30.0)

    def create_task(self, title: str, description: str = None) -> Task:
        """Создание задачи на сервере"""
        response = self.client.post(
            f"{self.base_url}/tasks",
            json={"title": title, "description": description}
        )
        response.raise_for_status()
        return self._parse_task(response.json())

    def get_tasks(self, completed: Optional[bool] = None) -> List[Task]:
        """Получение списка задач"""
        params = {}
        if completed is not None:
            params["completed"] = completed

        response = self.client.get(f"{self.base_url}/tasks", params=params)
        response.raise_for_status()
        return [self._parse_task(t) for t in response.json()]

    def get_task(self, task_id: int) -> Task:
        """Получение задачи по ID"""
        response = self.client.get(f"{self.base_url}/tasks/{task_id}")
        response.raise_for_status()
        return self._parse_task(response.json())

    def complete_task(self, task_id: int) -> Task:
        """Отметить задачу как выполненную"""
        response = self.client.patch(
            f"{self.base_url}/tasks/{task_id}",
            json={"completed": True}
        )
        response.raise_for_status()
        return self._parse_task(response.json())

    def delete_task(self, task_id: int) -> None:
        """Удаление задачи"""
        response = self.client.delete(f"{self.base_url}/tasks/{task_id}")
        response.raise_for_status()

    def _parse_task(self, data: dict) -> Task:
        """Парсинг ответа в объект Task"""
        return Task(
            id=data["id"],
            title=data["title"],
            description=data.get("description"),
            completed=data["completed"],
            created_at=datetime.fromisoformat(data["created_at"]),
            due_date=datetime.fromisoformat(data["due_date"]) if data.get("due_date") else None
        )

    def close(self):
        """Закрытие HTTP клиента"""
        self.client.close()

# Использование
if __name__ == "__main__":
    client = TaskClient()

    # Создание задачи
    task = client.create_task("Learn Client-Server", "Study architecture patterns")
    print(f"Created: {task.title}")

    # Получение всех задач
    tasks = client.get_tasks()
    print(f"Total tasks: {len(tasks)}")

    # Завершение задачи
    completed = client.complete_task(task.id)
    print(f"Completed: {completed.completed}")

    client.close()
```

### WebSocket пример (Real-time)

```python
# server/websocket_server.py
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import List
import json

app = FastAPI()

class ConnectionManager:
    """Менеджер WebSocket соединений"""

    def __init__(self):
        self.active_connections: List[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)

    async def broadcast(self, message: dict):
        """Отправка сообщения всем подключённым клиентам"""
        for connection in self.active_connections:
            await connection.send_json(message)

manager = ConnectionManager()

@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: str):
    await manager.connect(websocket)

    # Уведомление о подключении
    await manager.broadcast({
        "type": "user_joined",
        "client_id": client_id
    })

    try:
        while True:
            # Получение сообщения от клиента
            data = await websocket.receive_json()

            # Broadcast всем клиентам
            await manager.broadcast({
                "type": "message",
                "client_id": client_id,
                "content": data.get("content")
            })
    except WebSocketDisconnect:
        manager.disconnect(websocket)
        await manager.broadcast({
            "type": "user_left",
            "client_id": client_id
        })
```

### Диаграмма взаимодействия

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     CLIENT-SERVER COMMUNICATION                         │
└─────────────────────────────────────────────────────────────────────────┘

   CLIENT                                              SERVER
   ──────                                              ──────
      │                                                   │
      │  1. HTTP POST /tasks {"title": "New Task"}        │
      │ ─────────────────────────────────────────────────▶│
      │                                                   │
      │                     [Server processes request]    │
      │                     [Validates data]              │
      │                     [Saves to database]           │
      │                                                   │
      │  2. HTTP 201 Created {"id": 1, "title": "..."}   │
      │ ◀─────────────────────────────────────────────────│
      │                                                   │
      │  3. HTTP GET /tasks                               │
      │ ─────────────────────────────────────────────────▶│
      │                                                   │
      │  4. HTTP 200 OK [{"id": 1, ...}, {"id": 2, ...}] │
      │ ◀─────────────────────────────────────────────────│
      │                                                   │
```

## Best practices и антипаттерны

### Best Practices

#### 1. Stateless сервер
```python
# ХОРОШО: каждый запрос содержит всю необходимую информацию
@app.get("/users/me")
def get_current_user(token: str = Depends(oauth2_scheme)):
    user = verify_token(token)  # Токен содержит информацию о пользователе
    return user

# ПЛОХО: сервер хранит состояние сессии
sessions = {}  # Проблема при масштабировании!

@app.get("/users/me")
def get_current_user(session_id: str):
    return sessions.get(session_id)  # Не будет работать с load balancer
```

#### 2. Версионирование API
```python
# Версия в URL
app_v1 = FastAPI(prefix="/api/v1")
app_v2 = FastAPI(prefix="/api/v2")

# Или в заголовках
@app.get("/users")
def get_users(api_version: str = Header("1.0")):
    if api_version == "2.0":
        return get_users_v2()
    return get_users_v1()
```

#### 3. Обработка ошибок
```python
from fastapi import HTTPException
from fastapi.responses import JSONResponse

@app.exception_handler(Exception)
async def global_exception_handler(request, exc):
    return JSONResponse(
        status_code=500,
        content={
            "error": "Internal Server Error",
            "message": str(exc) if DEBUG else "Something went wrong"
        }
    )

# Специфичные ошибки с понятными сообщениями
class TaskNotFoundError(HTTPException):
    def __init__(self, task_id: int):
        super().__init__(
            status_code=404,
            detail=f"Task with id {task_id} not found"
        )
```

#### 4. Graceful degradation на клиенте
```python
class ResilientTaskClient:
    """Клиент с обработкой ошибок и retry"""

    def __init__(self, base_url: str, max_retries: int = 3):
        self.base_url = base_url
        self.max_retries = max_retries

    def get_tasks_with_fallback(self) -> List[Task]:
        """Получение задач с fallback на кэш"""
        for attempt in range(self.max_retries):
            try:
                return self._fetch_tasks()
            except httpx.RequestError:
                if attempt == self.max_retries - 1:
                    # Возврат кэшированных данных
                    return self._get_cached_tasks()
                time.sleep(2 ** attempt)  # Exponential backoff
```

### Антипаттерны

#### 1. Chatty Client
```python
# ПЛОХО: много мелких запросов
for user_id in user_ids:
    user = client.get(f"/users/{user_id}")  # N запросов!

# ХОРОШО: batch запрос
users = client.post("/users/batch", json={"ids": user_ids})  # 1 запрос
```

#### 2. Fat Server / Thin Client (или наоборот)
```python
# ПЛОХО: вся логика на клиенте
def calculate_order_total(items, discounts, taxes, shipping):
    # 100 строк бизнес-логики на клиенте
    # Проблема: разные клиенты = разная логика
    pass

# ХОРОШО: бизнес-логика на сервере
response = client.post("/orders/calculate", json={
    "items": items,
    "coupon_code": coupon
})
total = response.json()["total"]
```

#### 3. Игнорирование timeout
```python
# ПЛОХО: бесконечное ожидание
response = requests.get(url)  # Может зависнуть навсегда

# ХОРОШО: явный timeout
response = requests.get(url, timeout=(5, 30))  # connect=5s, read=30s
```

## Связанные паттерны

| Паттерн | Связь |
|---------|-------|
| **RESTful API** | Стиль взаимодействия клиент-сервер |
| **Microservices** | Множество серверов (сервисов) |
| **API Gateway** | Единая точка входа для клиентов |
| **Load Balancer** | Распределение нагрузки между серверами |
| **Circuit Breaker** | Защита клиента от недоступного сервера |
| **Proxy Pattern** | Промежуточный слой клиент-сервер |
| **Peer-to-Peer** | Альтернатива клиент-серверу |

## Ресурсы для изучения

### Книги
- **"RESTful Web Services"** — Leonard Richardson, Sam Ruby
- **"Designing Data-Intensive Applications"** — Martin Kleppmann
- **"Building Microservices"** — Sam Newman

### Статьи и документация
- [HTTP/1.1 Specification (RFC 2616)](https://datatracker.ietf.org/doc/html/rfc2616)
- [REST API Design Best Practices](https://restfulapi.net/)
- [WebSocket Protocol (RFC 6455)](https://datatracker.ietf.org/doc/html/rfc6455)

### Видео
- "Client-Server Architecture Explained" — ByteByteGo
- "REST vs GraphQL vs gRPC" — Fireship

---

## Резюме

Клиент-серверная архитектура — это фундамент современных распределённых систем:

- **Ключевой принцип**: разделение на потребителя (client) и поставщика (server)
- **Преимущества**: централизация, безопасность, масштабируемость
- **Недостатки**: single point of failure, сетевая зависимость
- **Современные реализации**: REST API, GraphQL, gRPC, WebSocket

Понимание этого паттерна критически важно для разработки любых сетевых приложений.
