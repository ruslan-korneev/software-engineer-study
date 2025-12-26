# Контроль доступа (Access Control)

## Что такое контроль доступа?

**Контроль доступа (Authorization)** — это процесс определения того, какие действия разрешены аутентифицированному пользователю. Отвечает на вопрос: "Что вам разрешено делать?"

**Аутентификация** → Кто вы?
**Авторизация** → Что вам можно?

---

## Модели контроля доступа

### 1. RBAC (Role-Based Access Control)

Права доступа назначаются ролям, а не пользователям напрямую.

```
User → Role → Permissions
```

**Пример структуры:**

```python
from enum import Enum
from typing import Set

class Permission(str, Enum):
    READ_USERS = "users:read"
    WRITE_USERS = "users:write"
    DELETE_USERS = "users:delete"
    READ_ORDERS = "orders:read"
    WRITE_ORDERS = "orders:write"
    ADMIN_PANEL = "admin:access"

class Role(str, Enum):
    VIEWER = "viewer"
    EDITOR = "editor"
    ADMIN = "admin"

# Маппинг ролей на права
ROLE_PERMISSIONS: dict[Role, Set[Permission]] = {
    Role.VIEWER: {
        Permission.READ_USERS,
        Permission.READ_ORDERS,
    },
    Role.EDITOR: {
        Permission.READ_USERS,
        Permission.WRITE_USERS,
        Permission.READ_ORDERS,
        Permission.WRITE_ORDERS,
    },
    Role.ADMIN: {
        Permission.READ_USERS,
        Permission.WRITE_USERS,
        Permission.DELETE_USERS,
        Permission.READ_ORDERS,
        Permission.WRITE_ORDERS,
        Permission.ADMIN_PANEL,
    },
}
```

**Реализация FastAPI:**

```python
from fastapi import Depends, HTTPException
from functools import wraps

class User:
    def __init__(self, id: str, roles: list[Role]):
        self.id = id
        self.roles = roles

    def has_permission(self, permission: Permission) -> bool:
        for role in self.roles:
            if permission in ROLE_PERMISSIONS.get(role, set()):
                return True
        return False

def require_permission(permission: Permission):
    """Декоратор для проверки прав доступа"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, current_user: User = Depends(get_current_user), **kwargs):
            if not current_user.has_permission(permission):
                raise HTTPException(
                    status_code=403,
                    detail=f"Permission denied: {permission.value}"
                )
            return await func(*args, current_user=current_user, **kwargs)
        return wrapper
    return decorator

# Использование
@app.delete("/users/{user_id}")
@require_permission(Permission.DELETE_USERS)
async def delete_user(user_id: str, current_user: User = Depends(get_current_user)):
    return {"message": f"User {user_id} deleted"}
```

**Альтернативная реализация через Depends:**

```python
from fastapi import Security

def PermissionChecker(required_permission: Permission):
    async def check_permission(current_user: User = Depends(get_current_user)):
        if not current_user.has_permission(required_permission):
            raise HTTPException(status_code=403, detail="Permission denied")
        return current_user
    return check_permission

@app.delete("/users/{user_id}")
async def delete_user(
    user_id: str,
    current_user: User = Security(PermissionChecker(Permission.DELETE_USERS))
):
    return {"message": f"User {user_id} deleted"}
```

---

### 2. ABAC (Attribute-Based Access Control)

Решения принимаются на основе атрибутов пользователя, ресурса и контекста.

```
if user.department == resource.department
   and user.clearance >= resource.classification
   and time.is_business_hours()
   then ALLOW
```

**Пример реализации:**

```python
from dataclasses import dataclass
from datetime import datetime

@dataclass
class Resource:
    id: str
    owner_id: str
    department: str
    classification: str  # public, internal, confidential, secret

@dataclass
class User:
    id: str
    department: str
    clearance_level: int  # 1-4
    is_manager: bool

@dataclass
class Context:
    ip_address: str
    time: datetime
    device_type: str

class ABACPolicy:
    CLASSIFICATION_LEVELS = {
        "public": 1,
        "internal": 2,
        "confidential": 3,
        "secret": 4,
    }

    def can_access(self, user: User, resource: Resource, context: Context) -> bool:
        # Правило 1: Владелец всегда имеет доступ
        if user.id == resource.owner_id:
            return True

        # Правило 2: Проверка уровня допуска
        required_level = self.CLASSIFICATION_LEVELS.get(resource.classification, 4)
        if user.clearance_level < required_level:
            return False

        # Правило 3: Confidential и secret — только свой отдел
        if resource.classification in ("confidential", "secret"):
            if user.department != resource.department and not user.is_manager:
                return False

        # Правило 4: Secret — только в рабочее время и из офисной сети
        if resource.classification == "secret":
            if not self._is_business_hours(context.time):
                return False
            if not self._is_office_network(context.ip_address):
                return False

        return True

    def _is_business_hours(self, time: datetime) -> bool:
        return 9 <= time.hour < 18 and time.weekday() < 5

    def _is_office_network(self, ip: str) -> bool:
        return ip.startswith("10.0.") or ip.startswith("192.168.1.")
```

**Использование в FastAPI:**

```python
abac = ABACPolicy()

async def check_resource_access(
    resource_id: str,
    current_user: User = Depends(get_current_user),
    request: Request = None
):
    resource = await get_resource(resource_id)
    if not resource:
        raise HTTPException(status_code=404)

    context = Context(
        ip_address=request.client.host,
        time=datetime.utcnow(),
        device_type=request.headers.get("User-Agent", "")
    )

    if not abac.can_access(current_user, resource, context):
        raise HTTPException(status_code=403, detail="Access denied")

    return resource

@app.get("/documents/{doc_id}")
async def get_document(resource: Resource = Depends(check_resource_access)):
    return resource
```

---

### 3. ACL (Access Control Lists)

Список прав для каждого ресурса.

```python
from enum import Enum

class AccessLevel(str, Enum):
    READ = "read"
    WRITE = "write"
    ADMIN = "admin"

# ACL в базе данных
"""
CREATE TABLE resource_acl (
    resource_id VARCHAR(36),
    principal_type VARCHAR(20),  -- 'user' или 'group'
    principal_id VARCHAR(36),
    access_level VARCHAR(20),
    PRIMARY KEY (resource_id, principal_type, principal_id)
);
"""

class ACLService:
    def __init__(self, db):
        self.db = db

    async def check_access(
        self,
        resource_id: str,
        user: User,
        required_level: AccessLevel
    ) -> bool:
        # Проверяем прямой доступ пользователя
        user_access = await self.db.fetch_one(
            """
            SELECT access_level FROM resource_acl
            WHERE resource_id = $1
            AND principal_type = 'user'
            AND principal_id = $2
            """,
            resource_id, user.id
        )

        if user_access and self._level_sufficient(user_access["access_level"], required_level):
            return True

        # Проверяем доступ через группы
        for group_id in user.group_ids:
            group_access = await self.db.fetch_one(
                """
                SELECT access_level FROM resource_acl
                WHERE resource_id = $1
                AND principal_type = 'group'
                AND principal_id = $2
                """,
                resource_id, group_id
            )
            if group_access and self._level_sufficient(group_access["access_level"], required_level):
                return True

        return False

    def _level_sufficient(self, actual: str, required: AccessLevel) -> bool:
        levels = {AccessLevel.READ: 1, AccessLevel.WRITE: 2, AccessLevel.ADMIN: 3}
        return levels.get(actual, 0) >= levels.get(required, 0)

    async def grant_access(
        self,
        resource_id: str,
        principal_type: str,
        principal_id: str,
        level: AccessLevel
    ):
        await self.db.execute(
            """
            INSERT INTO resource_acl (resource_id, principal_type, principal_id, access_level)
            VALUES ($1, $2, $3, $4)
            ON CONFLICT (resource_id, principal_type, principal_id)
            DO UPDATE SET access_level = $4
            """,
            resource_id, principal_type, principal_id, level.value
        )
```

---

## Принцип наименьших привилегий (Least Privilege)

**Правило:** Давайте пользователям минимально необходимые права.

```python
# ПЛОХО: Все права по умолчанию
def create_user(username: str):
    return User(
        username=username,
        roles=[Role.ADMIN]  # ❌ Опасно!
    )

# ХОРОШО: Минимальные права по умолчанию
def create_user(username: str, role: Role = Role.VIEWER):
    return User(
        username=username,
        roles=[role]  # ✅ Только viewer по умолчанию
    )
```

**Примеры применения:**
- Новые пользователи получают роль `viewer`
- API keys создаются только на чтение
- Сервисные аккаунты имеют доступ только к нужным ресурсам
- Права запрашиваются по мере необходимости

---

## Защита от IDOR (Insecure Direct Object Reference)

**IDOR** — уязвимость, когда пользователь может получить доступ к чужим ресурсам, изменив ID в запросе.

```
# Атакующий меняет order_id
GET /api/orders/12345  →  GET /api/orders/12346
```

### ПЛОХО: Нет проверки владельца

```python
@app.get("/orders/{order_id}")
async def get_order(order_id: str):
    order = await db.get_order(order_id)
    return order  # ❌ Любой может получить любой заказ!
```

### ХОРОШО: Проверка владельца

```python
@app.get("/orders/{order_id}")
async def get_order(order_id: str, current_user: User = Depends(get_current_user)):
    order = await db.get_order(order_id)

    if not order:
        raise HTTPException(status_code=404)

    # Проверяем, что заказ принадлежит текущему пользователю
    if order.user_id != current_user.id:
        # Возвращаем 404, а не 403 — чтобы не раскрывать существование ресурса
        raise HTTPException(status_code=404)

    return order
```

### Лучше: Универсальная защита

```python
class ResourceChecker:
    """Универсальный проверщик доступа к ресурсам"""

    async def check_ownership(
        self,
        resource_id: str,
        resource_type: str,
        user: User
    ) -> Any:
        """Получает ресурс только если пользователь имеет к нему доступ"""

        resource = await self._get_resource(resource_type, resource_id)

        if not resource:
            raise HTTPException(status_code=404)

        if not self._user_can_access(user, resource):
            raise HTTPException(status_code=404)  # Не 403!

        return resource

    def _user_can_access(self, user: User, resource: Any) -> bool:
        # Владелец
        if hasattr(resource, 'user_id') and resource.user_id == user.id:
            return True

        # Админ
        if Role.ADMIN in user.roles:
            return True

        # Проверка через ACL
        return self.acl.check(user, resource)

# Dependency для FastAPI
async def get_user_order(
    order_id: str,
    current_user: User = Depends(get_current_user),
    checker: ResourceChecker = Depends()
):
    return await checker.check_ownership(order_id, "order", current_user)

@app.get("/orders/{order_id}")
async def get_order(order: Order = Depends(get_user_order)):
    return order
```

---

## Иерархия ресурсов

При работе с вложенными ресурсами проверяйте всю цепочку.

```python
# /organizations/{org_id}/projects/{project_id}/tasks/{task_id}

@app.get("/organizations/{org_id}/projects/{project_id}/tasks/{task_id}")
async def get_task(
    org_id: str,
    project_id: str,
    task_id: str,
    current_user: User = Depends(get_current_user)
):
    # 1. Проверяем доступ к организации
    org = await db.get_organization(org_id)
    if not org or not user_belongs_to_org(current_user, org):
        raise HTTPException(status_code=404)

    # 2. Проверяем, что проект принадлежит организации
    project = await db.get_project(project_id)
    if not project or project.org_id != org_id:
        raise HTTPException(status_code=404)

    # 3. Проверяем, что задача принадлежит проекту
    task = await db.get_task(task_id)
    if not task or task.project_id != project_id:
        raise HTTPException(status_code=404)

    return task
```

---

## Scopes (области доступа)

Ограничение прав токена независимо от прав пользователя.

```python
from fastapi import Security
from fastapi.security import OAuth2PasswordBearer, SecurityScopes

oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="token",
    scopes={
        "users:read": "Чтение профилей пользователей",
        "users:write": "Редактирование профилей",
        "orders:read": "Просмотр заказов",
        "orders:write": "Создание и редактирование заказов",
    }
)

async def get_current_user(
    security_scopes: SecurityScopes,
    token: str = Depends(oauth2_scheme)
) -> User:
    # Декодируем токен
    payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])

    # Получаем scopes из токена
    token_scopes = payload.get("scopes", [])

    # Проверяем, что токен имеет все требуемые scopes
    for scope in security_scopes.scopes:
        if scope not in token_scopes:
            raise HTTPException(
                status_code=403,
                detail=f"Not enough permissions. Required scope: {scope}",
                headers={"WWW-Authenticate": f'Bearer scope="{scope}"'}
            )

    return await get_user_from_db(payload["sub"])

# Использование
@app.get("/users/me", dependencies=[Security(get_current_user, scopes=["users:read"])])
async def read_current_user(current_user: User = Depends(get_current_user)):
    return current_user

@app.put("/users/me", dependencies=[Security(get_current_user, scopes=["users:write"])])
async def update_current_user(data: UserUpdate, current_user: User = Depends(get_current_user)):
    return await update_user(current_user.id, data)
```

---

## Best Practices

### 1. Защита по умолчанию (Deny by Default)

```python
# ПЛОХО: Blacklist — легко забыть добавить новый эндпоинт
PROTECTED_PATHS = ["/admin", "/users"]

# ХОРОШО: Whitelist — новые эндпоинты защищены по умолчанию
PUBLIC_PATHS = ["/health", "/docs", "/auth/login"]

@app.middleware("http")
async def auth_middleware(request: Request, call_next):
    if request.url.path not in PUBLIC_PATHS:
        await verify_authentication(request)
    return await call_next(request)
```

### 2. Централизованная проверка прав

```python
# ПЛОХО: Проверки разбросаны по коду
@app.delete("/users/{user_id}")
async def delete_user(user_id: str, current_user: User = Depends(get_current_user)):
    if "admin" not in current_user.roles:  # ❌ Дублирование
        raise HTTPException(status_code=403)
    # ...

# ХОРОШО: Единый сервис авторизации
class AuthorizationService:
    async def authorize(self, user: User, action: str, resource: Any) -> bool:
        # Вся логика в одном месте
        pass

authz = AuthorizationService()

@app.delete("/users/{user_id}")
async def delete_user(user_id: str, current_user: User = Depends(get_current_user)):
    if not await authz.authorize(current_user, "delete", f"user:{user_id}"):
        raise HTTPException(status_code=403)
    # ...
```

### 3. Логирование решений

```python
import logging

logger = logging.getLogger("authorization")

class AuthorizationService:
    async def authorize(self, user: User, action: str, resource: str) -> bool:
        allowed = self._check_permissions(user, action, resource)

        logger.info(
            "Authorization decision",
            extra={
                "user_id": user.id,
                "action": action,
                "resource": resource,
                "allowed": allowed,
            }
        )

        return allowed
```

---

## Типичные ошибки

| Ошибка | Последствие | Решение |
|--------|-------------|---------|
| Нет проверки владельца ресурса | IDOR — доступ к чужим данным | Всегда проверять user_id |
| Проверка прав на клиенте | Обход авторизации | Проверять на сервере |
| Избыточные права по умолчанию | Утечка данных | Least privilege |
| Возврат 403 вместо 404 | Раскрытие существования ресурса | Всегда 404 для чужих ресурсов |
| Нет проверки scopes токена | Токен с минимальными правами всё может | Проверять scopes |

---

## Чек-лист контроля доступа

- [ ] Используется модель RBAC/ABAC/ACL
- [ ] Все эндпоинты требуют авторизации (whitelist для публичных)
- [ ] Реализована защита от IDOR
- [ ] Применяется принцип наименьших привилегий
- [ ] Проверяется вся цепочка вложенных ресурсов
- [ ] Логируются все решения авторизации
- [ ] API tokens имеют ограниченные scopes
- [ ] Роли и права регулярно пересматриваются
