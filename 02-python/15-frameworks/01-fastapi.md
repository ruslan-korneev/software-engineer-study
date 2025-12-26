# FastAPI

Современный async веб-фреймворк с автодокументацией.

```bash
pip install fastapi uvicorn
uvicorn main:app --reload
```

---

## Минимальное приложение

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello"}

@app.get("/items/{item_id}")
async def get_item(item_id: int):
    return {"item_id": item_id}
```

- `/docs` — Swagger UI
- `/redoc` — ReDoc

---

## Path и Query параметры

```python
@app.get("/users/{user_id}")
async def get_user(user_id: int):  # Path
    return {"user_id": user_id}

@app.get("/items")
async def list_items(skip: int = 0, limit: int = 10):  # Query
    return {"skip": skip, "limit": limit}
```

---

## Pydantic модели

```python
from pydantic import BaseModel

class UserCreate(BaseModel):
    name: str
    email: str

class UserResponse(BaseModel):
    id: int
    name: str

@app.post("/users", response_model=UserResponse)
async def create_user(user: UserCreate):
    return {"id": 1, **user.model_dump()}
```

---

## Dependency Injection

```python
from fastapi import Depends, Header, HTTPException

async def get_current_user(token: str = Header()):
    user = decode_token(token)
    if not user:
        raise HTTPException(401, "Invalid token")
    return user

@app.get("/me")
async def read_me(user: User = Depends(get_current_user)):
    return user
```

---

## Роутеры

```python
# routers/users.py
from fastapi import APIRouter

router = APIRouter(prefix="/users", tags=["users"])

@router.get("/")
async def list_users(): ...

# main.py
app.include_router(router)
```

---

## Middleware

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
)

@app.middleware("http")
async def timing(request, call_next):
    start = time.time()
    response = await call_next(request)
    response.headers["X-Time"] = str(time.time() - start)
    return response
```

---

## Background Tasks

```python
from fastapi import BackgroundTasks

@app.post("/send")
async def send(bg: BackgroundTasks):
    bg.add_task(send_email, "user@example.com")
    return {"status": "queued"}
```

---

## Lifespan

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    await db.connect()      # Startup
    yield
    await db.disconnect()   # Shutdown

app = FastAPI(lifespan=lifespan)
```

---

## Резюме

| Фича | Описание |
|------|----------|
| Pydantic | Автовалидация |
| Depends | Dependency Injection |
| APIRouter | Модульность |
| BackgroundTasks | Фоновые задачи |
| Swagger/ReDoc | Автодокументация |

**Когда:** REST API, микросервисы, async приложения.
