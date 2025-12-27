# PBAC (Policy-Based Access Control)

## Что такое PBAC?

**PBAC (Policy-Based Access Control)** — это модель управления доступом, в которой решения об авторизации принимаются на основе политик (policies), написанных на специальном языке. PBAC позволяет определять сложные, гибкие правила доступа, которые могут учитывать любые факторы: атрибуты, роли, контекст, время и т.д.

## Ключевые концепции

### Основные элементы PBAC

| Элемент | Описание | Пример |
|---------|----------|--------|
| **Policy** | Правило авторизации | "Менеджеры могут одобрять расходы до $10000" |
| **Rule** | Условие в политике | amount <= 10000 AND user.role == "manager" |
| **Effect** | Результат политики | allow, deny |
| **Target** | Когда политика применяется | resource.type == "expense" |
| **Condition** | Дополнительные условия | context.time BETWEEN "09:00" AND "18:00" |

### Структура политики

```yaml
policy:
  name: "expense-approval-policy"
  description: "Managers can approve expenses up to $10,000"

  target:
    resource: "expense"
    action: "approve"

  rules:
    - effect: allow
      condition:
        all:
          - subject.role == "manager"
          - resource.amount <= 10000

    - effect: allow
      condition:
        all:
          - subject.role == "director"
          - resource.amount <= 50000

    - effect: allow
      condition:
        subject.role == "cfo"

  default: deny
```

### Архитектура PBAC

```
┌─────────────────────────────────────────────────────────────┐
│                    Policy Decision Point (PDP)               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                    Policy Engine                         │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │ │
│  │  │  Policies   │  │   Rules     │  │ Conditions  │     │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘     │ │
│  └─────────────────────────────────────────────────────────┘ │
│                           │                                   │
│          ┌────────────────┼────────────────┐                 │
│          ▼                ▼                ▼                 │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐         │
│  │ Subject Info │ │ Resource Info│ │ Context Info │         │
│  │ (PIP)        │ │ (PIP)        │ │ (PIP)        │         │
│  └──────────────┘ └──────────────┘ └──────────────┘         │
└─────────────────────────────────────────────────────────────┘
                            │
                    Decision│ (allow/deny)
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                Policy Enforcement Point (PEP)                │
│                     (Application)                            │
└─────────────────────────────────────────────────────────────┘
```

## Практические примеры кода

### Python — базовый Policy Engine

```python
from dataclasses import dataclass, field
from typing import Dict, Any, List, Callable, Optional
from enum import Enum
import operator
import re

class Effect(Enum):
    ALLOW = "allow"
    DENY = "deny"

@dataclass
class Condition:
    """Условие в правиле"""
    expression: str
    evaluator: Callable[[Dict[str, Any]], bool] = None

@dataclass
class Rule:
    """Правило в политике"""
    name: str
    effect: Effect
    condition: Condition
    priority: int = 0

@dataclass
class Policy:
    """Политика авторизации"""
    name: str
    description: str
    target: Dict[str, str]  # resource, action
    rules: List[Rule]
    default_effect: Effect = Effect.DENY

class ConditionEvaluator:
    """Оценщик условий"""

    OPERATORS = {
        "==": operator.eq,
        "!=": operator.ne,
        ">": operator.gt,
        ">=": operator.ge,
        "<": operator.lt,
        "<=": operator.le,
        "in": lambda a, b: a in b,
        "contains": lambda a, b: b in a,
        "matches": lambda a, b: bool(re.match(b, str(a))),
    }

    @classmethod
    def evaluate(cls, expression: str, context: Dict[str, Any]) -> bool:
        """Оценивает выражение в контексте"""
        # Поддерживаем AND/OR
        if " AND " in expression:
            parts = expression.split(" AND ")
            return all(cls.evaluate(p.strip(), context) for p in parts)

        if " OR " in expression:
            parts = expression.split(" OR ")
            return any(cls.evaluate(p.strip(), context) for p in parts)

        # Парсим простое условие
        for op_str, op_func in cls.OPERATORS.items():
            if f" {op_str} " in expression:
                left, right = expression.split(f" {op_str} ", 1)
                left_val = cls._resolve_value(left.strip(), context)
                right_val = cls._resolve_value(right.strip(), context)
                return op_func(left_val, right_val)

        # Boolean expression
        return cls._resolve_value(expression, context)

    @classmethod
    def _resolve_value(cls, path: str, context: Dict[str, Any]) -> Any:
        """Разрешает значение из контекста"""
        # Литералы
        if path.startswith('"') and path.endswith('"'):
            return path[1:-1]
        if path.startswith("'") and path.endswith("'"):
            return path[1:-1]
        if path.isdigit():
            return int(path)
        if path.replace(".", "").isdigit():
            return float(path)
        if path.lower() == "true":
            return True
        if path.lower() == "false":
            return False
        if path.startswith("[") and path.endswith("]"):
            return eval(path)  # Осторожно в production!

        # Путь в контексте (subject.role, resource.amount)
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

class PolicyEngine:
    """Движок политик"""

    def __init__(self):
        self.policies: List[Policy] = []

    def add_policy(self, policy: Policy):
        """Добавляет политику"""
        self.policies.append(policy)

    def evaluate(
        self,
        subject: Dict[str, Any],
        resource: Dict[str, Any],
        action: str,
        context: Optional[Dict[str, Any]] = None
    ) -> tuple[Effect, str]:
        """Оценивает политики и возвращает решение"""
        eval_context = {
            "subject": subject,
            "resource": resource,
            "action": action,
            "context": context or {}
        }

        applicable_policies = []

        # Находим применимые политики
        for policy in self.policies:
            if self._matches_target(policy.target, resource, action):
                applicable_policies.append(policy)

        if not applicable_policies:
            return Effect.DENY, "No applicable policy found"

        # Оцениваем правила
        deny_reasons = []
        allow_found = False
        allow_reason = ""

        for policy in applicable_policies:
            for rule in sorted(policy.rules, key=lambda r: -r.priority):
                try:
                    if ConditionEvaluator.evaluate(rule.condition.expression, eval_context):
                        if rule.effect == Effect.DENY:
                            return Effect.DENY, f"Denied by rule '{rule.name}' in policy '{policy.name}'"
                        elif rule.effect == Effect.ALLOW:
                            allow_found = True
                            allow_reason = f"Allowed by rule '{rule.name}' in policy '{policy.name}'"
                except Exception as e:
                    deny_reasons.append(f"Error evaluating rule '{rule.name}': {str(e)}")

        if allow_found:
            return Effect.ALLOW, allow_reason

        return Effect.DENY, f"Default deny. Reasons: {'; '.join(deny_reasons) if deny_reasons else 'No matching rule'}"

    def _matches_target(self, target: Dict[str, str], resource: Dict[str, Any], action: str) -> bool:
        """Проверяет, соответствует ли запрос цели политики"""
        if "resource" in target:
            if resource.get("type") != target["resource"]:
                return False
        if "action" in target:
            if action != target["action"]:
                return False
        return True

# Использование
def demo_pbac():
    engine = PolicyEngine()

    # Политика одобрения расходов
    expense_policy = Policy(
        name="expense-approval",
        description="Rules for expense approval",
        target={"resource": "expense", "action": "approve"},
        rules=[
            Rule(
                name="manager-limit",
                effect=Effect.ALLOW,
                condition=Condition("subject.role == 'manager' AND resource.amount <= 10000"),
                priority=10
            ),
            Rule(
                name="director-limit",
                effect=Effect.ALLOW,
                condition=Condition("subject.role == 'director' AND resource.amount <= 50000"),
                priority=20
            ),
            Rule(
                name="cfo-unlimited",
                effect=Effect.ALLOW,
                condition=Condition("subject.role == 'cfo'"),
                priority=30
            ),
            Rule(
                name="deny-suspended",
                effect=Effect.DENY,
                condition=Condition("subject.status == 'suspended'"),
                priority=100  # Высший приоритет
            ),
        ],
        default_effect=Effect.DENY
    )

    # Политика доступа к документам
    document_policy = Policy(
        name="document-access",
        description="Rules for document access",
        target={"resource": "document"},
        rules=[
            Rule(
                name="owner-access",
                effect=Effect.ALLOW,
                condition=Condition("subject.id == resource.owner_id"),
                priority=50
            ),
            Rule(
                name="department-read",
                effect=Effect.ALLOW,
                condition=Condition(
                    "action == 'read' AND subject.department == resource.department"
                ),
                priority=10
            ),
            Rule(
                name="confidential-clearance",
                effect=Effect.DENY,
                condition=Condition(
                    "resource.classification == 'confidential' AND subject.clearance_level < 2"
                ),
                priority=100
            ),
        ]
    )

    engine.add_policy(expense_policy)
    engine.add_policy(document_policy)

    # Тестируем
    print("Policy-Based Access Control Tests:")
    print("=" * 60)

    tests = [
        # Expense tests
        {
            "subject": {"id": "1", "role": "manager", "status": "active"},
            "resource": {"type": "expense", "amount": 5000},
            "action": "approve",
            "desc": "Manager approves $5000 expense"
        },
        {
            "subject": {"id": "2", "role": "manager", "status": "active"},
            "resource": {"type": "expense", "amount": 15000},
            "action": "approve",
            "desc": "Manager approves $15000 expense (over limit)"
        },
        {
            "subject": {"id": "3", "role": "director", "status": "active"},
            "resource": {"type": "expense", "amount": 30000},
            "action": "approve",
            "desc": "Director approves $30000 expense"
        },
        {
            "subject": {"id": "4", "role": "cfo", "status": "suspended"},
            "resource": {"type": "expense", "amount": 100000},
            "action": "approve",
            "desc": "Suspended CFO approves expense"
        },
        # Document tests
        {
            "subject": {"id": "5", "department": "engineering", "clearance_level": 1},
            "resource": {"type": "document", "owner_id": "5", "department": "engineering", "classification": "public"},
            "action": "read",
            "desc": "User reads own document"
        },
        {
            "subject": {"id": "6", "department": "engineering", "clearance_level": 1},
            "resource": {"type": "document", "owner_id": "7", "department": "engineering", "classification": "confidential"},
            "action": "read",
            "desc": "Low clearance user reads confidential doc"
        },
    ]

    for test in tests:
        effect, reason = engine.evaluate(
            subject=test["subject"],
            resource=test["resource"],
            action=test["action"]
        )
        status = "✅ ALLOW" if effect == Effect.ALLOW else "❌ DENY"
        print(f"\n{test['desc']}")
        print(f"  Result: {status}")
        print(f"  Reason: {reason}")

demo_pbac()
```

### Python (FastAPI) с Open Policy Agent (OPA)

```python
from fastapi import FastAPI, Depends, HTTPException, Request
from pydantic import BaseModel
from typing import Dict, Any, Optional
import httpx

app = FastAPI()

OPA_URL = "http://localhost:8181/v1/data"

class AuthorizationRequest(BaseModel):
    input: Dict[str, Any]

async def check_opa(policy_path: str, input_data: Dict[str, Any]) -> bool:
    """Проверяет политику в OPA"""
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{OPA_URL}/{policy_path}",
            json={"input": input_data}
        )

        if response.status_code != 200:
            raise HTTPException(status_code=500, detail="OPA error")

        result = response.json()
        return result.get("result", {}).get("allow", False)

class User(BaseModel):
    id: str
    role: str
    department: str

async def get_current_user(request: Request) -> User:
    # Из JWT/session
    return User(id="1", role="manager", department="engineering")

# Middleware для OPA
async def opa_authorize(
    request: Request,
    user: User = Depends(get_current_user)
):
    """Проверяет авторизацию через OPA"""
    input_data = {
        "user": user.model_dump(),
        "method": request.method,
        "path": request.url.path,
        "headers": dict(request.headers)
    }

    allowed = await check_opa("authz/allow", input_data)

    if not allowed:
        raise HTTPException(status_code=403, detail="Access denied by policy")

    return user

# Эндпоинты с OPA авторизацией
@app.get("/api/resources")
async def list_resources(user: User = Depends(opa_authorize)):
    return {"resources": ["resource1", "resource2"]}

@app.post("/api/expenses/{expense_id}/approve")
async def approve_expense(
    expense_id: str,
    user: User = Depends(get_current_user)
):
    """Одобрение расхода с проверкой через OPA"""
    # Получаем данные о расходе
    expense = {"id": expense_id, "amount": 5000, "status": "pending"}

    input_data = {
        "user": user.model_dump(),
        "action": "approve",
        "resource": {
            "type": "expense",
            **expense
        }
    }

    allowed = await check_opa("expenses/allow", input_data)

    if not allowed:
        raise HTTPException(
            status_code=403,
            detail="You are not authorized to approve this expense"
        )

    return {"message": "Expense approved", "expense_id": expense_id}
```

### Rego политики для OPA

```rego
# authz.rego - политики авторизации

package authz

import future.keywords.if
import future.keywords.in

default allow := false

# Администраторы имеют полный доступ
allow if {
    input.user.role == "admin"
}

# Менеджеры могут читать ресурсы своего отдела
allow if {
    input.method == "GET"
    input.user.role == "manager"
    startswith(input.path, "/api/resources")
}

# expenses.rego - политики для расходов

package expenses

import future.keywords.if

default allow := false

# Менеджеры могут одобрять расходы до $10000
allow if {
    input.action == "approve"
    input.user.role == "manager"
    input.resource.amount <= 10000
}

# Директора могут одобрять расходы до $50000
allow if {
    input.action == "approve"
    input.user.role == "director"
    input.resource.amount <= 50000
}

# CFO может одобрять любые расходы
allow if {
    input.action == "approve"
    input.user.role == "cfo"
}

# Заблокированные пользователи не могут ничего
allow := false if {
    input.user.status == "suspended"
}

# Возвращаем причину отказа
reason := "Amount exceeds your approval limit" if {
    input.action == "approve"
    input.user.role == "manager"
    input.resource.amount > 10000
}

reason := "User is suspended" if {
    input.user.status == "suspended"
}
```

### Cedar (AWS)

```cedar
// Политика на языке Cedar (используется в AWS Verified Permissions)

// Менеджеры могут одобрять расходы до $10000
permit (
    principal in Role::"manager",
    action == Action::"approve",
    resource is Expense
) when {
    resource.amount <= 10000
};

// Директора могут одобрять расходы до $50000
permit (
    principal in Role::"director",
    action == Action::"approve",
    resource is Expense
) when {
    resource.amount <= 50000
};

// CFO может одобрять любые расходы
permit (
    principal in Role::"cfo",
    action == Action::"approve",
    resource is Expense
);

// Запрет для заблокированных пользователей
forbid (
    principal,
    action,
    resource
) when {
    principal.status == "suspended"
};
```

## Policy Languages

| Язык | Использование | Особенности |
|------|---------------|-------------|
| **Rego** | Open Policy Agent | Declarative, Datalog-based |
| **Cedar** | AWS Verified Permissions | Human-readable, typed |
| **XACML** | Enterprise standards | XML-based, verbose |
| **Casbin** | Multi-language | DSL, various adapters |
| **Polar** | Oso | Python-like syntax |

## Плюсы PBAC

| Преимущество | Описание |
|--------------|----------|
| **Гибкость** | Любые условия можно выразить в политике |
| **Централизация** | Все правила в одном месте |
| **Аудируемость** | Политики можно проверять и тестировать |
| **Разделение** | Логика авторизации отделена от кода |
| **Динамичность** | Политики можно менять без деплоя |

## Минусы PBAC

| Недостаток | Описание |
|------------|----------|
| **Сложность** | Нужно изучить policy language |
| **Производительность** | Оценка политик может быть медленной |
| **Debugging** | Сложно отлаживать сложные политики |
| **Инфраструктура** | Нужен policy engine (OPA, etc.) |

## Когда использовать

### Подходит для:

- Сложных правил авторизации
- Enterprise-систем с compliance требованиями
- Когда нужно часто менять правила
- Микросервисной архитектуры

### НЕ подходит для:

- Простых приложений
- Когда RBAC/ABAC достаточно
- Стартапов без выделенной команды безопасности

## Best Practices

1. **Тестируй политики** — unit-тесты для каждой политики
2. **Version control** — храни политики в Git
3. **Default deny** — запрещай по умолчанию
4. **Мониторинг** — логируй все решения
5. **Кэширование** — кэшируй результаты оценки
6. **Dry-run mode** — тестируй изменения перед применением

## Типичные ошибки

### Ошибка 1: Default allow

```rego
# ПЛОХО
default allow := true

# ХОРОШО
default allow := false
```

### Ошибка 2: Нет приоритетов

```python
# ПЛОХО: порядок правил не определён
rules = [allow_rule, deny_rule]

# ХОРОШО: явные приоритеты
rules = [
    Rule(..., priority=100),  # Deny rules first
    Rule(..., priority=10),   # Allow rules after
]
```

### Ошибка 3: Слишком сложные политики

```rego
# ПЛОХО: одно огромное правило
allow if {
    condition1
    condition2
    # ... 50 conditions
}

# ХОРОШО: разбить на модули
allow if { admin_check }
allow if { manager_check }
allow if { user_check }
```

## PBAC vs RBAC vs ABAC

| Аспект | RBAC | ABAC | PBAC |
|--------|------|------|------|
| Основа | Роли | Атрибуты | Политики |
| Гибкость | Низкая | Высокая | Очень высокая |
| Сложность | Низкая | Средняя | Высокая |
| Выразительность | Ограниченная | Хорошая | Полная |
| Аудит | Простой | Средний | Отличный |

## Резюме

PBAC — это мощный подход к авторизации для сложных систем. Он позволяет выражать любые правила доступа в декларативном виде и централизованно управлять ими. Главное преимущество — гибкость и аудируемость. Для реализации рекомендуется использовать готовые policy engines: OPA для общего назначения, Cedar для AWS, Casbin для multi-language проектов.
