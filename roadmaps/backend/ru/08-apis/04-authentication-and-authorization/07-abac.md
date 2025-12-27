# ABAC (Attribute-Based Access Control)

## Что такое ABAC?

**ABAC (Attribute-Based Access Control)** — это модель управления доступом, в которой решения об авторизации принимаются на основе атрибутов субъекта (пользователя), объекта (ресурса), действия и окружения. ABAC обеспечивает гибкий, динамический и контекстно-зависимый контроль доступа.

## Ключевые концепции

### Типы атрибутов

| Тип | Описание | Примеры |
|-----|----------|---------|
| **Subject Attributes** | Атрибуты пользователя | department, role, clearance_level, location |
| **Object Attributes** | Атрибуты ресурса | owner, classification, created_at, status |
| **Action Attributes** | Атрибуты действия | read, write, approve, delete |
| **Environment Attributes** | Контекст выполнения | time, ip_address, device_type, threat_level |

### Архитектура ABAC (XACML)

```
                    ┌─────────────────────────────────────────┐
                    │              PEP                        │
                    │   (Policy Enforcement Point)            │
                    └─────────────────────────────────────────┘
                                      │
                                      │ Authorization Request
                                      ▼
                    ┌─────────────────────────────────────────┐
                    │              PDP                        │
                    │   (Policy Decision Point)               │
                    │                                         │
                    │   Evaluates policies against attributes │
                    └─────────────────────────────────────────┘
                        │                  │                │
             ┌──────────┘                  │                └──────────┐
             ▼                             ▼                           ▼
┌─────────────────────┐    ┌─────────────────────┐    ┌─────────────────────┐
│         PIP         │    │         PAP         │    │        PRP          │
│ (Policy Info Point) │    │(Policy Admin Point) │    │(Policy Retrieval    │
│                     │    │                     │    │       Point)        │
│ Provides attributes │    │ Manages policies    │    │ Stores policies     │
└─────────────────────┘    └─────────────────────┘    └─────────────────────┘
```

### Структура ABAC Policy

```
IF
    subject.department == "finance" AND
    subject.clearance >= resource.classification AND
    action == "read" AND
    environment.time BETWEEN "09:00" AND "18:00" AND
    environment.location == "office"
THEN
    PERMIT
```

## Практические примеры кода

### Python — базовая реализация

```python
from dataclasses import dataclass, field
from typing import Dict, Any, Callable, List, Optional
from datetime import datetime, time
from enum import Enum

class Decision(Enum):
    PERMIT = "permit"
    DENY = "deny"
    NOT_APPLICABLE = "not_applicable"
    INDETERMINATE = "indeterminate"

@dataclass
class Attributes:
    """Контейнер для атрибутов"""
    subject: Dict[str, Any] = field(default_factory=dict)
    resource: Dict[str, Any] = field(default_factory=dict)
    action: Dict[str, Any] = field(default_factory=dict)
    environment: Dict[str, Any] = field(default_factory=dict)

@dataclass
class Policy:
    """ABAC Policy"""
    name: str
    description: str
    target: Callable[[Attributes], bool]  # Применима ли политика
    condition: Callable[[Attributes], bool]  # Условие политики
    effect: Decision  # PERMIT или DENY

    def evaluate(self, attrs: Attributes) -> Decision:
        """Оценивает политику"""
        # Проверяем, применима ли политика к запросу
        if not self.target(attrs):
            return Decision.NOT_APPLICABLE

        # Проверяем условие
        try:
            if self.condition(attrs):
                return self.effect
            return Decision.NOT_APPLICABLE
        except Exception:
            return Decision.INDETERMINATE

class PolicyDecisionPoint:
    """PDP — принимает решения на основе политик"""

    def __init__(self, combining_algorithm: str = "deny_unless_permit"):
        self.policies: List[Policy] = []
        self.combining_algorithm = combining_algorithm

    def add_policy(self, policy: Policy):
        self.policies.append(policy)

    def evaluate(self, attrs: Attributes) -> Decision:
        """Оценивает все политики и комбинирует результаты"""
        results = [policy.evaluate(attrs) for policy in self.policies]

        if self.combining_algorithm == "deny_unless_permit":
            # Deny по умолчанию, если нет явного Permit
            if Decision.PERMIT in results:
                if Decision.DENY not in results:
                    return Decision.PERMIT
            return Decision.DENY

        elif self.combining_algorithm == "permit_unless_deny":
            # Permit по умолчанию, если нет явного Deny
            if Decision.DENY in results:
                return Decision.DENY
            return Decision.PERMIT

        elif self.combining_algorithm == "first_applicable":
            # Первая применимая политика
            for result in results:
                if result in [Decision.PERMIT, Decision.DENY]:
                    return result
            return Decision.NOT_APPLICABLE

        return Decision.INDETERMINATE

class PolicyEnforcementPoint:
    """PEP — точка применения политик"""

    def __init__(self, pdp: PolicyDecisionPoint):
        self.pdp = pdp

    def check_access(
        self,
        subject: Dict[str, Any],
        resource: Dict[str, Any],
        action: str,
        environment: Optional[Dict[str, Any]] = None
    ) -> bool:
        """Проверяет доступ"""
        attrs = Attributes(
            subject=subject,
            resource=resource,
            action={"name": action},
            environment=environment or self._get_environment_attrs()
        )

        decision = self.pdp.evaluate(attrs)
        return decision == Decision.PERMIT

    def _get_environment_attrs(self) -> Dict[str, Any]:
        """Получает атрибуты окружения"""
        now = datetime.now()
        return {
            "current_time": now.time(),
            "current_date": now.date(),
            "day_of_week": now.strftime("%A"),
        }

# Создание политик
def setup_abac() -> PolicyEnforcementPoint:
    pdp = PolicyDecisionPoint(combining_algorithm="deny_unless_permit")

    # Политика 1: Сотрудники финансового отдела могут читать финансовые документы
    finance_read_policy = Policy(
        name="finance_department_read",
        description="Finance department can read financial documents",
        target=lambda attrs: (
            attrs.action.get("name") == "read" and
            attrs.resource.get("type") == "financial_document"
        ),
        condition=lambda attrs: (
            attrs.subject.get("department") == "finance"
        ),
        effect=Decision.PERMIT
    )

    # Политика 2: Владелец ресурса имеет полный доступ
    owner_policy = Policy(
        name="owner_full_access",
        description="Resource owner has full access",
        target=lambda attrs: True,  # Применяется ко всему
        condition=lambda attrs: (
            attrs.subject.get("user_id") == attrs.resource.get("owner_id")
        ),
        effect=Decision.PERMIT
    )

    # Политика 3: Доступ только в рабочие часы
    working_hours_policy = Policy(
        name="working_hours_only",
        description="Sensitive documents only during working hours",
        target=lambda attrs: (
            attrs.resource.get("classification") == "sensitive"
        ),
        condition=lambda attrs: (
            time(9, 0) <= attrs.environment.get("current_time", time(0, 0)) <= time(18, 0)
        ),
        effect=Decision.PERMIT
    )

    # Политика 4: Запрет доступа для заблокированных пользователей
    blocked_users_policy = Policy(
        name="block_suspended_users",
        description="Deny access to suspended users",
        target=lambda attrs: True,
        condition=lambda attrs: (
            attrs.subject.get("status") == "suspended"
        ),
        effect=Decision.DENY
    )

    # Политика 5: Уровень допуска должен быть >= уровню секретности
    clearance_policy = Policy(
        name="clearance_level_check",
        description="User clearance must match or exceed resource classification",
        target=lambda attrs: (
            attrs.resource.get("classification_level") is not None
        ),
        condition=lambda attrs: (
            attrs.subject.get("clearance_level", 0) >=
            attrs.resource.get("classification_level", 0)
        ),
        effect=Decision.PERMIT
    )

    pdp.add_policy(blocked_users_policy)  # Deny политики первыми
    pdp.add_policy(owner_policy)
    pdp.add_policy(finance_read_policy)
    pdp.add_policy(working_hours_policy)
    pdp.add_policy(clearance_policy)

    return PolicyEnforcementPoint(pdp)

# Использование
pep = setup_abac()

# Тест 1: Сотрудник финансов читает финансовый документ
result = pep.check_access(
    subject={"user_id": 1, "department": "finance", "status": "active"},
    resource={"type": "financial_document", "owner_id": 2},
    action="read"
)
print(f"Finance employee read financial doc: {result}")  # True

# Тест 2: Сотрудник IT читает финансовый документ
result = pep.check_access(
    subject={"user_id": 3, "department": "IT", "status": "active"},
    resource={"type": "financial_document", "owner_id": 2},
    action="read"
)
print(f"IT employee read financial doc: {result}")  # False

# Тест 3: Владелец документа
result = pep.check_access(
    subject={"user_id": 2, "department": "IT", "status": "active"},
    resource={"type": "financial_document", "owner_id": 2},
    action="delete"
)
print(f"Owner delete own doc: {result}")  # True

# Тест 4: Заблокированный пользователь
result = pep.check_access(
    subject={"user_id": 1, "department": "finance", "status": "suspended"},
    resource={"type": "financial_document", "owner_id": 2},
    action="read"
)
print(f"Suspended user access: {result}")  # False
```

### Python (FastAPI) — полная реализация

```python
from fastapi import FastAPI, Depends, HTTPException, Request
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from pydantic import BaseModel
from typing import Dict, Any, List, Optional, Callable
from datetime import datetime, time
from functools import wraps
import jwt

app = FastAPI()
security = HTTPBearer()

SECRET_KEY = "your-secret-key"

# Модели
class User(BaseModel):
    id: int
    username: str
    department: str
    role: str
    clearance_level: int
    status: str
    location: Optional[str] = None

class Resource(BaseModel):
    id: int
    type: str
    owner_id: int
    classification_level: int = 0
    department: Optional[str] = None

# ABAC Engine
class ABACEngine:
    def __init__(self):
        self.policies: List[Dict] = []

    def add_policy(
        self,
        name: str,
        target: Callable,
        condition: Callable,
        effect: str = "permit"
    ):
        self.policies.append({
            "name": name,
            "target": target,
            "condition": condition,
            "effect": effect
        })

    def check_access(
        self,
        subject: Dict[str, Any],
        resource: Dict[str, Any],
        action: str,
        environment: Dict[str, Any]
    ) -> bool:
        context = {
            "subject": subject,
            "resource": resource,
            "action": action,
            "environment": environment
        }

        # Сначала проверяем deny политики
        for policy in self.policies:
            if policy["effect"] == "deny":
                if policy["target"](context) and policy["condition"](context):
                    return False

        # Затем проверяем permit политики
        for policy in self.policies:
            if policy["effect"] == "permit":
                if policy["target"](context) and policy["condition"](context):
                    return True

        return False  # Deny by default

# Инициализация ABAC
abac = ABACEngine()

# Политика: Заблокированные пользователи не имеют доступа
abac.add_policy(
    name="deny_suspended",
    target=lambda ctx: True,
    condition=lambda ctx: ctx["subject"].get("status") == "suspended",
    effect="deny"
)

# Политика: Владелец имеет полный доступ
abac.add_policy(
    name="owner_access",
    target=lambda ctx: True,
    condition=lambda ctx: ctx["subject"].get("id") == ctx["resource"].get("owner_id"),
    effect="permit"
)

# Политика: Доступ по отделу
abac.add_policy(
    name="department_access",
    target=lambda ctx: ctx["action"] in ["read", "update"],
    condition=lambda ctx: (
        ctx["subject"].get("department") == ctx["resource"].get("department")
    ),
    effect="permit"
)

# Политика: Уровень допуска
abac.add_policy(
    name="clearance_check",
    target=lambda ctx: ctx["resource"].get("classification_level", 0) > 0,
    condition=lambda ctx: (
        ctx["subject"].get("clearance_level", 0) >=
        ctx["resource"].get("classification_level", 0)
    ),
    effect="permit"
)

# Политика: Рабочие часы для sensitive ресурсов
abac.add_policy(
    name="working_hours",
    target=lambda ctx: ctx["resource"].get("classification_level", 0) >= 3,
    condition=lambda ctx: (
        9 <= ctx["environment"].get("hour", 0) <= 18 and
        ctx["environment"].get("day_of_week", 0) < 5  # Пн-Пт
    ),
    effect="permit"
)

# Политика: Admin имеет доступ ко всему
abac.add_policy(
    name="admin_access",
    target=lambda ctx: True,
    condition=lambda ctx: ctx["subject"].get("role") == "admin",
    effect="permit"
)

# Имитация БД
USERS_DB = {
    "alice": User(
        id=1, username="alice", department="finance",
        role="manager", clearance_level=3, status="active"
    ),
    "bob": User(
        id=2, username="bob", department="IT",
        role="developer", clearance_level=2, status="active"
    ),
    "charlie": User(
        id=3, username="charlie", department="finance",
        role="admin", clearance_level=5, status="active"
    ),
}

RESOURCES_DB = {
    1: Resource(
        id=1, type="document", owner_id=1,
        classification_level=2, department="finance"
    ),
    2: Resource(
        id=2, type="document", owner_id=2,
        classification_level=4, department="IT"
    ),
}

def get_environment_context() -> Dict[str, Any]:
    """Получает контекст окружения"""
    now = datetime.now()
    return {
        "hour": now.hour,
        "day_of_week": now.weekday(),
        "date": now.date().isoformat(),
        "time": now.time().isoformat(),
    }

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security)
) -> User:
    """Получает текущего пользователя из токена"""
    try:
        payload = jwt.decode(credentials.credentials, SECRET_KEY, algorithms=["HS256"])
        username = payload.get("sub")
        user = USERS_DB.get(username)
        if not user:
            raise HTTPException(status_code=401, detail="User not found")
        return user
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

def require_abac(action: str):
    """Декоратор для ABAC проверки на уровне ресурса"""
    def decorator(func):
        @wraps(func)
        async def wrapper(
            resource_id: int,
            user: User = Depends(get_current_user),
            **kwargs
        ):
            resource = RESOURCES_DB.get(resource_id)
            if not resource:
                raise HTTPException(status_code=404, detail="Resource not found")

            has_access = abac.check_access(
                subject=user.model_dump(),
                resource=resource.model_dump(),
                action=action,
                environment=get_environment_context()
            )

            if not has_access:
                raise HTTPException(
                    status_code=403,
                    detail=f"Access denied for action '{action}' on resource {resource_id}"
                )

            return await func(resource_id=resource_id, user=user, resource=resource, **kwargs)
        return wrapper
    return decorator

# Эндпоинты
@app.get("/resources/{resource_id}")
@require_abac("read")
async def get_resource(resource_id: int, user: User, resource: Resource):
    return {"resource": resource, "accessed_by": user.username}

@app.put("/resources/{resource_id}")
@require_abac("update")
async def update_resource(resource_id: int, user: User, resource: Resource):
    return {"message": "Updated", "resource_id": resource_id}

@app.delete("/resources/{resource_id}")
@require_abac("delete")
async def delete_resource(resource_id: int, user: User, resource: Resource):
    return {"message": "Deleted", "resource_id": resource_id}

@app.post("/auth/login")
async def login(username: str):
    user = USERS_DB.get(username)
    if not user:
        raise HTTPException(status_code=401, detail="User not found")

    token = jwt.encode({"sub": username}, SECRET_KEY, algorithm="HS256")
    return {"access_token": token}

# Эндпоинт для проверки доступа
@app.post("/check-access")
async def check_access(
    resource_id: int,
    action: str,
    user: User = Depends(get_current_user)
):
    resource = RESOURCES_DB.get(resource_id)
    if not resource:
        return {"allowed": False, "reason": "Resource not found"}

    has_access = abac.check_access(
        subject=user.model_dump(),
        resource=resource.model_dump(),
        action=action,
        environment=get_environment_context()
    )

    return {
        "allowed": has_access,
        "user": user.username,
        "action": action,
        "resource_id": resource_id
    }
```

### Python — ABAC с DSL (Domain Specific Language)

```python
from typing import Any, Dict
import operator
import re

class PolicyDSL:
    """Простой DSL для определения ABAC политик"""

    OPERATORS = {
        "==": operator.eq,
        "!=": operator.ne,
        ">": operator.gt,
        ">=": operator.ge,
        "<": operator.lt,
        "<=": operator.le,
        "in": lambda a, b: a in b,
        "contains": lambda a, b: b in a,
    }

    @classmethod
    def parse_condition(cls, condition: str) -> callable:
        """Парсит строковое условие в функцию"""
        # Формат: "subject.department == 'finance'"

        def evaluate(context: Dict[str, Any]) -> bool:
            # Разбиваем на части
            parts = condition.split()

            if len(parts) < 3:
                raise ValueError(f"Invalid condition: {condition}")

            left_path = parts[0]
            op = parts[1]
            right_value = " ".join(parts[2:])

            # Получаем значение левой части
            left_value = cls._get_value(context, left_path)

            # Получаем значение правой части
            if right_value.startswith("'") and right_value.endswith("'"):
                right_value = right_value[1:-1]  # Строка
            elif right_value.startswith("[") and right_value.endswith("]"):
                right_value = eval(right_value)  # Список (осторожно с eval!)
            elif right_value.isdigit():
                right_value = int(right_value)
            elif right_value.replace(".", "").isdigit():
                right_value = float(right_value)
            else:
                right_value = cls._get_value(context, right_value)

            # Применяем оператор
            op_func = cls.OPERATORS.get(op)
            if not op_func:
                raise ValueError(f"Unknown operator: {op}")

            return op_func(left_value, right_value)

        return evaluate

    @classmethod
    def _get_value(cls, context: Dict[str, Any], path: str) -> Any:
        """Получает значение по пути (например, 'subject.department')"""
        parts = path.split(".")
        value = context

        for part in parts:
            if isinstance(value, dict):
                value = value.get(part)
            else:
                value = getattr(value, part, None)

            if value is None:
                return None

        return value

class PolicyBuilder:
    """Builder для создания политик"""

    def __init__(self, name: str):
        self.name = name
        self.target_conditions: list = []
        self.conditions: list = []
        self.effect = "permit"

    def for_action(self, *actions: str) -> 'PolicyBuilder':
        """Определяет целевые действия"""
        self.target_conditions.append(
            lambda ctx: ctx.get("action") in actions
        )
        return self

    def for_resource_type(self, *types: str) -> 'PolicyBuilder':
        """Определяет типы ресурсов"""
        self.target_conditions.append(
            lambda ctx: ctx.get("resource", {}).get("type") in types
        )
        return self

    def where(self, condition: str) -> 'PolicyBuilder':
        """Добавляет условие (DSL)"""
        self.conditions.append(PolicyDSL.parse_condition(condition))
        return self

    def permit(self) -> 'PolicyBuilder':
        self.effect = "permit"
        return self

    def deny(self) -> 'PolicyBuilder':
        self.effect = "deny"
        return self

    def build(self) -> Dict:
        return {
            "name": self.name,
            "target": lambda ctx: all(t(ctx) for t in self.target_conditions) if self.target_conditions else True,
            "condition": lambda ctx: all(c(ctx) for c in self.conditions) if self.conditions else True,
            "effect": self.effect
        }

# Использование DSL
policies = [
    PolicyBuilder("owner_access")
        .where("subject.id == resource.owner_id")
        .permit()
        .build(),

    PolicyBuilder("department_read")
        .for_action("read")
        .where("subject.department == resource.department")
        .permit()
        .build(),

    PolicyBuilder("clearance_check")
        .where("subject.clearance_level >= resource.classification_level")
        .permit()
        .build(),

    PolicyBuilder("deny_suspended")
        .where("subject.status == 'suspended'")
        .deny()
        .build(),

    PolicyBuilder("admin_access")
        .where("subject.role == 'admin'")
        .permit()
        .build(),
]

# Тестирование
context = {
    "subject": {"id": 1, "department": "finance", "clearance_level": 3, "status": "active", "role": "user"},
    "resource": {"id": 10, "owner_id": 2, "department": "finance", "classification_level": 2, "type": "document"},
    "action": "read",
    "environment": {}
}

for policy in policies:
    if policy["target"](context):
        result = policy["condition"](context)
        print(f"{policy['name']}: {result} -> {policy['effect']}")
```

## Плюсы ABAC

| Преимущество | Описание |
|--------------|----------|
| **Гибкость** | Любые комбинации атрибутов |
| **Динамичность** | Правила учитывают контекст |
| **Масштабируемость** | Не нужно создавать роли для каждого случая |
| **Fine-grained** | Очень детальный контроль |
| **Контекстность** | Учёт времени, места, устройства |

## Минусы ABAC

| Недостаток | Описание |
|------------|----------|
| **Сложность** | Труднее понять и отладить |
| **Производительность** | Вычисление политик при каждом запросе |
| **Аудит** | Сложнее отследить, кто имеет доступ |
| **Тестирование** | Много комбинаций для проверки |

## Когда использовать

### Подходит для:

- Динамических правил доступа
- Контекстно-зависимой авторизации
- Сложных enterprise-систем
- Когда RBAC недостаточно гибок

### НЕ подходит для:

- Простых приложений (overhead)
- Когда нужна простая аудируемость
- Систем с фиксированными ролями

## Best Practices

1. **Структурируй политики** — группируй по ресурсам или действиям
2. **Deny by default** — разрешай только явно указанное
3. **Кэшируй атрибуты** — для производительности
4. **Логируй решения** — для аудита и отладки
5. **Тестируй политики** — unit-тесты для каждой политики
6. **Используй DSL** — для читаемости политик

## ABAC vs RBAC

| Критерий | RBAC | ABAC |
|----------|------|------|
| Основа | Роли | Атрибуты |
| Гибкость | Низкая | Высокая |
| Сложность | Низкая | Высокая |
| Контекст | Нет | Да |
| Аудит | Простой | Сложный |
| Подходит | Корпоративные системы | Динамичные системы |

## Резюме

ABAC — это мощная модель авторизации для сложных сценариев, где RBAC недостаточно гибок. Она позволяет учитывать любые атрибуты при принятии решений. Однако ABAC требует большей работы по проектированию, реализации и тестированию. Часто используется в комбинации с RBAC.
