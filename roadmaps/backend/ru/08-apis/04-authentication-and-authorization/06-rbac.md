# RBAC (Role-Based Access Control)

## Что такое RBAC?

**RBAC (Role-Based Access Control)** — это модель управления доступом, в которой права доступа назначаются не напрямую пользователям, а ролям. Пользователи получают роли, и через них — соответствующие разрешения. RBAC — один из самых распространённых подходов к авторизации в современных приложениях.

## Ключевые концепции

### Основные элементы RBAC

| Элемент | Описание | Пример |
|---------|----------|--------|
| **User** | Субъект, выполняющий действия | john@example.com |
| **Role** | Набор разрешений | admin, editor, viewer |
| **Permission** | Право на выполнение действия | read:articles, write:articles |
| **Resource** | Объект, над которым выполняется действие | Article, User, Order |
| **Action** | Тип операции | create, read, update, delete |

### Иерархия RBAC (NIST)

| Уровень | Название | Описание |
|---------|----------|----------|
| RBAC0 | Flat RBAC | Базовый: пользователи, роли, разрешения |
| RBAC1 | Hierarchical RBAC | + иерархия ролей (наследование) |
| RBAC2 | Constrained RBAC | + ограничения (SoD, кардинальность) |
| RBAC3 | Symmetric RBAC | RBAC1 + RBAC2 |

### Диаграмма связей

```
┌─────────────┐     ┌─────────────┐     ┌─────────────────┐
│    Users    │────>│    Roles    │────>│   Permissions   │
└─────────────┘     └─────────────┘     └─────────────────┘
                          │                      │
                          │ наследование         │ resource + action
                          ▼                      ▼
                    ┌───────────┐         ┌─────────────┐
                    │ Parent    │         │ articles:   │
                    │ Roles     │         │   - read    │
                    └───────────┘         │   - write   │
                                          │   - delete  │
                                          └─────────────┘
```

## Практические примеры кода

### Python — базовая реализация

```python
from enum import Enum
from dataclasses import dataclass, field
from typing import Set, Dict, Optional

class Action(Enum):
    CREATE = "create"
    READ = "read"
    UPDATE = "update"
    DELETE = "delete"

@dataclass
class Permission:
    resource: str
    action: Action

    def __hash__(self):
        return hash((self.resource, self.action))

    def __str__(self):
        return f"{self.action.value}:{self.resource}"

@dataclass
class Role:
    name: str
    permissions: Set[Permission] = field(default_factory=set)
    parent: Optional['Role'] = None  # Для иерархии

    def get_all_permissions(self) -> Set[Permission]:
        """Получает все разрешения, включая унаследованные"""
        perms = set(self.permissions)
        if self.parent:
            perms.update(self.parent.get_all_permissions())
        return perms

    def has_permission(self, resource: str, action: Action) -> bool:
        """Проверяет наличие разрешения"""
        required = Permission(resource, action)
        return required in self.get_all_permissions()

@dataclass
class User:
    id: int
    username: str
    roles: Set[Role] = field(default_factory=set)

    def has_permission(self, resource: str, action: Action) -> bool:
        """Проверяет, есть ли у пользователя разрешение"""
        for role in self.roles:
            if role.has_permission(resource, action):
                return True
        return False

    def has_role(self, role_name: str) -> bool:
        """Проверяет наличие роли"""
        return any(role.name == role_name for role in self.roles)

# Создание ролей и разрешений
def setup_rbac():
    # Разрешения
    read_articles = Permission("articles", Action.READ)
    write_articles = Permission("articles", Action.UPDATE)
    create_articles = Permission("articles", Action.CREATE)
    delete_articles = Permission("articles", Action.DELETE)

    read_users = Permission("users", Action.READ)
    manage_users = Permission("users", Action.UPDATE)

    # Роли с иерархией
    viewer = Role("viewer", {read_articles})
    editor = Role("editor", {read_articles, write_articles, create_articles})
    admin = Role(
        "admin",
        {read_users, manage_users, delete_articles},
        parent=editor  # Admin наследует от Editor
    )

    return {"viewer": viewer, "editor": editor, "admin": admin}

# Использование
roles = setup_rbac()

user = User(id=1, username="john", roles={roles["editor"]})
print(user.has_permission("articles", Action.READ))    # True
print(user.has_permission("articles", Action.DELETE))  # False
print(user.has_permission("users", Action.READ))       # False

admin = User(id=2, username="admin", roles={roles["admin"]})
print(admin.has_permission("articles", Action.READ))   # True (унаследовано)
print(admin.has_permission("articles", Action.DELETE)) # True
print(admin.has_permission("users", Action.READ))      # True
```

### Python (FastAPI) — полная реализация

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from pydantic import BaseModel
from typing import List, Set, Optional
from enum import Enum
from functools import wraps
import jwt

app = FastAPI()
security = HTTPBearer()

SECRET_KEY = "your-secret-key"

# Enums и модели
class Action(str, Enum):
    CREATE = "create"
    READ = "read"
    UPDATE = "update"
    DELETE = "delete"

class RoleName(str, Enum):
    VIEWER = "viewer"
    EDITOR = "editor"
    ADMIN = "admin"
    SUPER_ADMIN = "super_admin"

# Определение разрешений для ролей
ROLE_PERMISSIONS = {
    RoleName.VIEWER: [
        ("articles", Action.READ),
        ("comments", Action.READ),
    ],
    RoleName.EDITOR: [
        ("articles", Action.READ),
        ("articles", Action.CREATE),
        ("articles", Action.UPDATE),
        ("comments", Action.READ),
        ("comments", Action.CREATE),
        ("comments", Action.UPDATE),
        ("comments", Action.DELETE),
    ],
    RoleName.ADMIN: [
        ("articles", Action.READ),
        ("articles", Action.CREATE),
        ("articles", Action.UPDATE),
        ("articles", Action.DELETE),
        ("comments", Action.READ),
        ("comments", Action.CREATE),
        ("comments", Action.UPDATE),
        ("comments", Action.DELETE),
        ("users", Action.READ),
        ("users", Action.UPDATE),
    ],
    RoleName.SUPER_ADMIN: [
        ("*", Action.READ),    # Все ресурсы
        ("*", Action.CREATE),
        ("*", Action.UPDATE),
        ("*", Action.DELETE),
    ],
}

# Иерархия ролей
ROLE_HIERARCHY = {
    RoleName.SUPER_ADMIN: [RoleName.ADMIN],
    RoleName.ADMIN: [RoleName.EDITOR],
    RoleName.EDITOR: [RoleName.VIEWER],
    RoleName.VIEWER: [],
}

def get_all_permissions(role: RoleName) -> Set[tuple]:
    """Получает все разрешения роли, включая унаследованные"""
    permissions = set(ROLE_PERMISSIONS.get(role, []))

    for parent_role in ROLE_HIERARCHY.get(role, []):
        permissions.update(get_all_permissions(parent_role))

    return permissions

def has_permission(roles: List[RoleName], resource: str, action: Action) -> bool:
    """Проверяет, есть ли разрешение у любой из ролей"""
    for role in roles:
        perms = get_all_permissions(role)
        # Проверяем точное совпадение или wildcard
        if (resource, action) in perms or ("*", action) in perms:
            return True
    return False

# Модели для API
class User(BaseModel):
    id: int
    username: str
    roles: List[RoleName]

class TokenPayload(BaseModel):
    sub: str
    roles: List[RoleName]

# Имитация БД пользователей
USERS_DB = {
    "viewer": User(id=1, username="viewer", roles=[RoleName.VIEWER]),
    "editor": User(id=2, username="editor", roles=[RoleName.EDITOR]),
    "admin": User(id=3, username="admin", roles=[RoleName.ADMIN]),
    "super": User(id=4, username="super", roles=[RoleName.SUPER_ADMIN]),
}

# Dependencies
async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security)
) -> User:
    """Извлекает пользователя из JWT токена"""
    try:
        payload = jwt.decode(
            credentials.credentials,
            SECRET_KEY,
            algorithms=["HS256"]
        )
        username = payload.get("sub")
        user = USERS_DB.get(username)

        if not user:
            raise HTTPException(status_code=401, detail="User not found")

        return user
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

def require_permission(resource: str, action: Action):
    """Decorator factory для проверки разрешений"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, user: User = Depends(get_current_user), **kwargs):
            if not has_permission(user.roles, resource, action):
                raise HTTPException(
                    status_code=status.HTTP_403_FORBIDDEN,
                    detail=f"Permission denied: {action.value}:{resource}"
                )
            return await func(*args, user=user, **kwargs)
        return wrapper
    return decorator

def require_role(required_role: RoleName):
    """Decorator factory для проверки роли"""
    async def role_checker(user: User = Depends(get_current_user)):
        # Проверяем роль с учётом иерархии
        def has_role(roles: List[RoleName], target: RoleName) -> bool:
            if target in roles:
                return True
            for role in roles:
                if target in ROLE_HIERARCHY.get(role, []):
                    return True
                # Рекурсивная проверка
                for parent in ROLE_HIERARCHY.get(role, []):
                    if has_role([parent], target):
                        return True
            return False

        if not has_role(user.roles, required_role):
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Role '{required_role.value}' required"
            )
        return user
    return role_checker

# API endpoints
@app.get("/articles")
async def list_articles(user: User = Depends(get_current_user)):
    """Доступно всем аутентифицированным пользователям"""
    if not has_permission(user.roles, "articles", Action.READ):
        raise HTTPException(status_code=403, detail="Permission denied")
    return {"articles": ["Article 1", "Article 2"]}

@app.post("/articles")
async def create_article(user: User = Depends(get_current_user)):
    """Требует разрешение на создание статей"""
    if not has_permission(user.roles, "articles", Action.CREATE):
        raise HTTPException(status_code=403, detail="Permission denied")
    return {"message": "Article created"}

@app.delete("/articles/{id}")
async def delete_article(id: int, user: User = Depends(get_current_user)):
    """Требует разрешение на удаление статей (только admin+)"""
    if not has_permission(user.roles, "articles", Action.DELETE):
        raise HTTPException(status_code=403, detail="Permission denied")
    return {"message": f"Article {id} deleted"}

@app.get("/users")
async def list_users(user: User = Depends(require_role(RoleName.ADMIN))):
    """Только для администраторов"""
    return {"users": list(USERS_DB.keys())}

@app.get("/admin/settings")
async def admin_settings(user: User = Depends(require_role(RoleName.SUPER_ADMIN))):
    """Только для super admin"""
    return {"settings": {"debug": False, "maintenance": False}}

# Генерация токена для тестирования
@app.post("/auth/login")
async def login(username: str):
    user = USERS_DB.get(username)
    if not user:
        raise HTTPException(status_code=401, detail="User not found")

    token = jwt.encode(
        {"sub": username, "roles": [r.value for r in user.roles]},
        SECRET_KEY,
        algorithm="HS256"
    )
    return {"access_token": token}
```

### Python — RBAC с базой данных

```python
from sqlalchemy import Column, Integer, String, Table, ForeignKey, create_engine
from sqlalchemy.orm import relationship, sessionmaker
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

# Таблицы связей многие-ко-многим
user_roles = Table(
    'user_roles',
    Base.metadata,
    Column('user_id', Integer, ForeignKey('users.id'), primary_key=True),
    Column('role_id', Integer, ForeignKey('roles.id'), primary_key=True)
)

role_permissions = Table(
    'role_permissions',
    Base.metadata,
    Column('role_id', Integer, ForeignKey('roles.id'), primary_key=True),
    Column('permission_id', Integer, ForeignKey('permissions.id'), primary_key=True)
)

class Permission(Base):
    __tablename__ = 'permissions'

    id = Column(Integer, primary_key=True)
    resource = Column(String(100), nullable=False)
    action = Column(String(50), nullable=False)

    def __repr__(self):
        return f"{self.action}:{self.resource}"

class Role(Base):
    __tablename__ = 'roles'

    id = Column(Integer, primary_key=True)
    name = Column(String(50), unique=True, nullable=False)
    parent_id = Column(Integer, ForeignKey('roles.id'), nullable=True)

    permissions = relationship('Permission', secondary=role_permissions)
    parent = relationship('Role', remote_side=[id], backref='children')

    def get_all_permissions(self):
        """Получает все разрешения, включая унаследованные"""
        perms = set(self.permissions)
        if self.parent:
            perms.update(self.parent.get_all_permissions())
        return perms

class User(Base):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True)
    username = Column(String(100), unique=True, nullable=False)
    password_hash = Column(String(255), nullable=False)

    roles = relationship('Role', secondary=user_roles)

    def has_permission(self, resource: str, action: str) -> bool:
        """Проверяет наличие разрешения"""
        for role in self.roles:
            for perm in role.get_all_permissions():
                if perm.resource == resource and perm.action == action:
                    return True
                # Wildcard check
                if perm.resource == "*" and perm.action == action:
                    return True
        return False

    def has_role(self, role_name: str) -> bool:
        """Проверяет наличие роли (с учётом иерархии)"""
        def check_role(role, target_name):
            if role.name == target_name:
                return True
            if role.parent:
                return check_role(role.parent, target_name)
            return False

        return any(check_role(role, role_name) for role in self.roles)

# Инициализация БД
def init_db():
    engine = create_engine('postgresql://user:pass@localhost/mydb')
    Base.metadata.create_all(engine)
    Session = sessionmaker(bind=engine)
    session = Session()

    # Создание разрешений
    perms = {
        "read_articles": Permission(resource="articles", action="read"),
        "write_articles": Permission(resource="articles", action="write"),
        "delete_articles": Permission(resource="articles", action="delete"),
        "read_users": Permission(resource="users", action="read"),
        "manage_users": Permission(resource="users", action="manage"),
    }

    for perm in perms.values():
        session.add(perm)

    # Создание ролей
    viewer = Role(name="viewer")
    viewer.permissions = [perms["read_articles"]]

    editor = Role(name="editor", parent=viewer)
    editor.permissions = [perms["write_articles"]]

    admin = Role(name="admin", parent=editor)
    admin.permissions = [
        perms["delete_articles"],
        perms["read_users"],
        perms["manage_users"]
    ]

    session.add_all([viewer, editor, admin])
    session.commit()

    return session
```

### Node.js (Express)

```javascript
const express = require('express');
const app = express();

// Определение ролей и разрешений
const PERMISSIONS = {
    viewer: [
        'articles:read',
        'comments:read'
    ],
    editor: [
        'articles:read',
        'articles:create',
        'articles:update',
        'comments:read',
        'comments:create',
        'comments:update',
        'comments:delete'
    ],
    admin: [
        'articles:*',      // Все действия над статьями
        'comments:*',
        'users:read',
        'users:update'
    ],
    super_admin: [
        '*:*'              // Полный доступ
    ]
};

// Иерархия ролей
const ROLE_HIERARCHY = {
    super_admin: ['admin'],
    admin: ['editor'],
    editor: ['viewer'],
    viewer: []
};

// Получение всех разрешений роли с учётом иерархии
function getAllPermissions(role) {
    const permissions = new Set(PERMISSIONS[role] || []);

    const parentRoles = ROLE_HIERARCHY[role] || [];
    for (const parentRole of parentRoles) {
        const parentPerms = getAllPermissions(parentRole);
        parentPerms.forEach(p => permissions.add(p));
    }

    return permissions;
}

// Проверка разрешения
function hasPermission(userRoles, resource, action) {
    const required = `${resource}:${action}`;

    for (const role of userRoles) {
        const perms = getAllPermissions(role);

        for (const perm of perms) {
            // Точное совпадение
            if (perm === required) return true;

            // Wildcard для действия (articles:*)
            if (perm === `${resource}:*`) return true;

            // Полный wildcard (*:*)
            if (perm === '*:*') return true;
        }
    }

    return false;
}

// Middleware для проверки разрешений
function requirePermission(resource, action) {
    return (req, res, next) => {
        const userRoles = req.user?.roles || [];

        if (!hasPermission(userRoles, resource, action)) {
            return res.status(403).json({
                error: `Permission denied: ${action}:${resource}`
            });
        }

        next();
    };
}

// Middleware для проверки роли
function requireRole(requiredRole) {
    return (req, res, next) => {
        const userRoles = req.user?.roles || [];

        const hasRole = userRoles.some(role => {
            if (role === requiredRole) return true;

            // Проверка иерархии
            const checkHierarchy = (currentRole) => {
                if (currentRole === requiredRole) return true;
                const parents = ROLE_HIERARCHY[currentRole] || [];
                return parents.some(parent => checkHierarchy(parent));
            };

            return checkHierarchy(role);
        });

        if (!hasRole) {
            return res.status(403).json({
                error: `Role '${requiredRole}' required`
            });
        }

        next();
    };
}

// Эндпоинты
app.get('/articles',
    requirePermission('articles', 'read'),
    (req, res) => res.json({ articles: [] })
);

app.post('/articles',
    requirePermission('articles', 'create'),
    (req, res) => res.json({ message: 'Created' })
);

app.delete('/articles/:id',
    requirePermission('articles', 'delete'),
    (req, res) => res.json({ message: 'Deleted' })
);

app.get('/admin/users',
    requireRole('admin'),
    (req, res) => res.json({ users: [] })
);

app.listen(3000);
```

## Плюсы RBAC

| Преимущество | Описание |
|--------------|----------|
| **Простота** | Легко понять и реализовать |
| **Управляемость** | Права назначаются ролям, не пользователям |
| **Аудит** | Легко отследить, кто имеет доступ |
| **Масштабируемость** | Добавление пользователей не усложняет систему |
| **Стандартизация** | Широко поддерживается (NIST, ISO) |

## Минусы RBAC

| Недостаток | Описание |
|------------|----------|
| **Role explosion** | Много ролей для сложных сценариев |
| **Статичность** | Нет динамических правил |
| **Грубая гранулярность** | Сложно сделать исключения |
| **Не контекстный** | Не учитывает время, место, и т.д. |

## Когда использовать

### Подходит для:

- Корпоративных приложений с чёткой иерархией
- Систем с фиксированными ролями (admin, editor, viewer)
- Когда права редко меняются
- Compliance-ориентированных систем

### НЕ подходит для:

- Динамических правил доступа ("автор может редактировать свой контент")
- Контекстно-зависимого доступа
- Сложных многотенантных систем
- Систем с тысячами уникальных ролей

## Best Practices

1. **Принцип наименьших привилегий** — давай минимально необходимые права
2. **Избегай role explosion** — группируй похожие разрешения
3. **Используй иерархию** — для наследования разрешений
4. **Регулярный аудит** — проверяй актуальность ролей
5. **Separation of Duties** — критичные операции требуют нескольких ролей
6. **Документируй роли** — чёткое описание каждой роли
7. **Централизованное управление** — одно место для управления ролями

## Типичные ошибки

### Ошибка 1: Слишком много ролей

```python
# ПЛОХО: отдельная роль для каждого сценария
roles = ["article_reader", "article_writer", "article_deleter",
         "comment_reader", "comment_writer", ...]

# ХОРОШО: обобщённые роли с разрешениями
roles = ["viewer", "editor", "admin"]
```

### Ошибка 2: Hardcoded проверки ролей

```python
# ПЛОХО: проверка конкретной роли
if user.role == "admin":
    allow_delete()

# ХОРОШО: проверка разрешения
if user.has_permission("articles", "delete"):
    allow_delete()
```

### Ошибка 3: Нет иерархии

```python
# ПЛОХО: дублирование разрешений
admin_permissions = ["read", "write", "delete", "manage_users"]
editor_permissions = ["read", "write"]  # Дублирование

# ХОРОШО: иерархия
editor_permissions = ["read", "write"]
admin = Role("admin", parent=editor, additional=["delete", "manage_users"])
```

## Альтернативы RBAC

| Модель | Описание | Когда использовать |
|--------|----------|-------------------|
| **ABAC** | Атрибутный контроль | Динамические правила |
| **PBAC** | Policy-based | Сложные бизнес-правила |
| **ReBAC** | Relationship-based | Социальные графы |
| **DAC** | Discretionary | Владелец контролирует доступ |

## Резюме

RBAC — это простая и эффективная модель авторизации для большинства приложений. Она хорошо работает, когда роли пользователей чётко определены и редко меняются. Для более сложных сценариев с динамическими правилами рассмотри ABAC или комбинацию моделей.
