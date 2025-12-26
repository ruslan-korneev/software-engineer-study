# MAC (Mandatory Access Control)

## Что такое MAC?

**MAC (Mandatory Access Control)** — это модель управления доступом, в которой доступ к ресурсам определяется централизованно на основе классификации субъектов и объектов. В отличие от DAC, где владелец ресурса сам определяет права доступа, в MAC политики безопасности устанавливаются системным администратором и не могут быть изменены обычными пользователями.

## Ключевые концепции

### Основные принципы MAC

| Принцип | Описание |
|---------|----------|
| **Централизованный контроль** | Политики задаются администратором |
| **Обязательность** | Пользователи не могут изменять права |
| **Классификация** | Субъекты и объекты имеют метки безопасности |
| **Принудительность** | Система принуждает к соблюдению политик |

### Уровни секретности (Security Labels)

```
┌─────────────────────────────────────────────────┐
│                TOP SECRET                        │
│  (Совершенно секретно)                          │
├─────────────────────────────────────────────────┤
│                   SECRET                         │
│  (Секретно)                                     │
├─────────────────────────────────────────────────┤
│                CONFIDENTIAL                      │
│  (Конфиденциально)                              │
├─────────────────────────────────────────────────┤
│                UNCLASSIFIED                      │
│  (Несекретно)                                   │
└─────────────────────────────────────────────────┘
```

### Модель Bell-LaPadula (Конфиденциальность)

Основные правила:
- **No Read Up (Simple Security)**: Субъект не может читать объекты с более высоким уровнем секретности
- **No Write Down (*-Property)**: Субъект не может записывать в объекты с более низким уровнем секретности

```
              Субъект: SECRET
                    │
    ┌───────────────┼───────────────┐
    │               │               │
    ▼               ▼               ▼
TOP SECRET      SECRET          CONFIDENTIAL
  ❌ Read       ✅ Read/Write      ❌ Write
  ❌ Write                        ✅ Read
```

### Модель Biba (Целостность)

Обратная Bell-LaPadula:
- **No Read Down**: Субъект не может читать менее целостные объекты
- **No Write Up**: Субъект не может записывать в более целостные объекты

## Практические примеры кода

### Python — базовая реализация MAC

```python
from enum import IntEnum
from dataclasses import dataclass, field
from typing import List, Optional, Set
from functools import wraps

class SecurityLevel(IntEnum):
    """Уровни секретности (Bell-LaPadula)"""
    UNCLASSIFIED = 0
    CONFIDENTIAL = 1
    SECRET = 2
    TOP_SECRET = 3

class IntegrityLevel(IntEnum):
    """Уровни целостности (Biba)"""
    LOW = 0
    MEDIUM = 1
    HIGH = 2
    CRITICAL = 3

@dataclass
class SecurityLabel:
    """Метка безопасности"""
    security_level: SecurityLevel
    integrity_level: IntegrityLevel
    compartments: Set[str] = field(default_factory=set)  # Категории

    def dominates(self, other: 'SecurityLabel') -> bool:
        """Проверяет, доминирует ли эта метка над другой"""
        return (
            self.security_level >= other.security_level and
            self.compartments >= other.compartments
        )

    def __str__(self):
        compartments_str = ",".join(sorted(self.compartments)) if self.compartments else "NONE"
        return f"{self.security_level.name}:{self.integrity_level.name}:{{{compartments_str}}}"

@dataclass
class Subject:
    """Субъект (пользователь/процесс)"""
    id: str
    name: str
    clearance: SecurityLabel  # Уровень допуска

@dataclass
class Object:
    """Объект (ресурс)"""
    id: str
    name: str
    classification: SecurityLabel  # Уровень классификации
    owner_id: str

class MACPolicy:
    """Mandatory Access Control Policy Engine"""

    def __init__(self, model: str = "bell_lapadula"):
        self.model = model
        self.subjects: dict[str, Subject] = {}
        self.objects: dict[str, Object] = {}

    def register_subject(self, subject: Subject):
        self.subjects[subject.id] = subject

    def register_object(self, obj: Object):
        self.objects[obj.id] = obj

    def can_read(self, subject: Subject, obj: Object) -> bool:
        """Проверяет право на чтение"""
        if self.model == "bell_lapadula":
            # No Read Up: субъект может читать только объекты
            # с уровнем секретности <= своему уровню допуска
            return (
                subject.clearance.security_level >= obj.classification.security_level and
                subject.clearance.compartments >= obj.classification.compartments
            )
        elif self.model == "biba":
            # No Read Down: субъект может читать только объекты
            # с уровнем целостности >= своему уровню
            return subject.clearance.integrity_level <= obj.classification.integrity_level
        return False

    def can_write(self, subject: Subject, obj: Object) -> bool:
        """Проверяет право на запись"""
        if self.model == "bell_lapadula":
            # No Write Down: субъект может писать только в объекты
            # с уровнем секретности >= своему уровню допуска
            return (
                subject.clearance.security_level <= obj.classification.security_level and
                obj.classification.compartments >= subject.clearance.compartments
            )
        elif self.model == "biba":
            # No Write Up: субъект может писать только в объекты
            # с уровнем целостности <= своему уровню
            return subject.clearance.integrity_level >= obj.classification.integrity_level
        return False

    def can_execute(self, subject: Subject, obj: Object) -> bool:
        """Проверяет право на выполнение"""
        # Для выполнения обычно требуются оба условия
        return self.can_read(subject, obj)

    def check_access(self, subject_id: str, object_id: str, action: str) -> tuple[bool, str]:
        """Проверяет доступ и возвращает результат с объяснением"""
        subject = self.subjects.get(subject_id)
        obj = self.objects.get(object_id)

        if not subject:
            return False, f"Subject {subject_id} not found"
        if not obj:
            return False, f"Object {object_id} not found"

        if action == "read":
            allowed = self.can_read(subject, obj)
            if allowed:
                return True, f"Read allowed: {subject.clearance} dominates {obj.classification}"
            return False, f"Read denied: {subject.clearance} does not dominate {obj.classification}"

        elif action == "write":
            allowed = self.can_write(subject, obj)
            if allowed:
                return True, f"Write allowed"
            return False, f"Write denied: would violate {self.model} model"

        elif action == "execute":
            allowed = self.can_execute(subject, obj)
            if allowed:
                return True, "Execute allowed"
            return False, "Execute denied"

        return False, f"Unknown action: {action}"

# Использование
def demo_mac():
    # Создаём политику Bell-LaPadula
    mac = MACPolicy(model="bell_lapadula")

    # Создаём субъектов с разными уровнями допуска
    alice = Subject(
        id="alice",
        name="Alice (Top Secret clearance)",
        clearance=SecurityLabel(
            SecurityLevel.TOP_SECRET,
            IntegrityLevel.HIGH,
            {"NATO", "CRYPTO"}
        )
    )

    bob = Subject(
        id="bob",
        name="Bob (Secret clearance)",
        clearance=SecurityLabel(
            SecurityLevel.SECRET,
            IntegrityLevel.MEDIUM,
            {"NATO"}
        )
    )

    charlie = Subject(
        id="charlie",
        name="Charlie (Confidential clearance)",
        clearance=SecurityLabel(
            SecurityLevel.CONFIDENTIAL,
            IntegrityLevel.LOW,
            set()
        )
    )

    mac.register_subject(alice)
    mac.register_subject(bob)
    mac.register_subject(charlie)

    # Создаём объекты с разными уровнями классификации
    top_secret_doc = Object(
        id="doc1",
        name="Nuclear Codes",
        classification=SecurityLabel(
            SecurityLevel.TOP_SECRET,
            IntegrityLevel.CRITICAL,
            {"NATO", "CRYPTO"}
        ),
        owner_id="system"
    )

    secret_doc = Object(
        id="doc2",
        name="Military Plans",
        classification=SecurityLabel(
            SecurityLevel.SECRET,
            IntegrityLevel.HIGH,
            {"NATO"}
        ),
        owner_id="system"
    )

    public_doc = Object(
        id="doc3",
        name="Public Report",
        classification=SecurityLabel(
            SecurityLevel.UNCLASSIFIED,
            IntegrityLevel.LOW,
            set()
        ),
        owner_id="system"
    )

    mac.register_object(top_secret_doc)
    mac.register_object(secret_doc)
    mac.register_object(public_doc)

    # Тестируем доступ
    tests = [
        ("alice", "doc1", "read"),   # Alice -> Top Secret (should pass)
        ("alice", "doc3", "write"),  # Alice -> Public (should fail - no write down)
        ("bob", "doc1", "read"),     # Bob -> Top Secret (should fail - no read up)
        ("bob", "doc2", "read"),     # Bob -> Secret (should pass)
        ("charlie", "doc2", "read"), # Charlie -> Secret (should fail)
        ("charlie", "doc3", "read"), # Charlie -> Public (should pass)
    ]

    print("Bell-LaPadula Model Access Control Tests:")
    print("-" * 60)
    for subject_id, object_id, action in tests:
        allowed, reason = mac.check_access(subject_id, object_id, action)
        status = "✅ ALLOWED" if allowed else "❌ DENIED"
        print(f"{subject_id} {action} {object_id}: {status}")
        print(f"   Reason: {reason}")
        print()

demo_mac()
```

### Python (FastAPI) — MAC в веб-приложении

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import HTTPBearer
from pydantic import BaseModel
from typing import Set, Optional
from enum import IntEnum
import jwt

app = FastAPI()
security = HTTPBearer()

SECRET_KEY = "your-secret-key"

class SecurityLevel(IntEnum):
    PUBLIC = 0
    INTERNAL = 1
    CONFIDENTIAL = 2
    SECRET = 3
    TOP_SECRET = 4

class SecurityLabel(BaseModel):
    level: SecurityLevel
    compartments: Set[str] = set()

    def dominates(self, other: 'SecurityLabel') -> bool:
        return (
            self.level >= other.level and
            self.compartments >= other.compartments
        )

class User(BaseModel):
    id: str
    username: str
    clearance: SecurityLabel

class Resource(BaseModel):
    id: str
    name: str
    classification: SecurityLabel
    content: str

# Имитация БД
USERS_DB = {
    "admin": User(
        id="1",
        username="admin",
        clearance=SecurityLabel(level=SecurityLevel.TOP_SECRET, compartments={"HR", "FINANCE", "TECH"})
    ),
    "manager": User(
        id="2",
        username="manager",
        clearance=SecurityLabel(level=SecurityLevel.CONFIDENTIAL, compartments={"HR", "FINANCE"})
    ),
    "employee": User(
        id="3",
        username="employee",
        clearance=SecurityLabel(level=SecurityLevel.INTERNAL, compartments=set())
    ),
}

RESOURCES_DB = {
    "public-policy": Resource(
        id="1",
        name="Company Policy",
        classification=SecurityLabel(level=SecurityLevel.PUBLIC, compartments=set()),
        content="Public company policy..."
    ),
    "salary-data": Resource(
        id="2",
        name="Salary Data",
        classification=SecurityLabel(level=SecurityLevel.CONFIDENTIAL, compartments={"HR", "FINANCE"}),
        content="Employee salary information..."
    ),
    "trade-secrets": Resource(
        id="3",
        name="Trade Secrets",
        classification=SecurityLabel(level=SecurityLevel.SECRET, compartments={"TECH"}),
        content="Proprietary algorithms..."
    ),
    "merger-plans": Resource(
        id="4",
        name="Merger Plans",
        classification=SecurityLabel(level=SecurityLevel.TOP_SECRET, compartments={"FINANCE"}),
        content="Secret M&A plans..."
    ),
}

class MACEnforcer:
    """Mandatory Access Control Enforcer"""

    @staticmethod
    def check_read(user_clearance: SecurityLabel, resource_classification: SecurityLabel) -> bool:
        """Bell-LaPadula: No Read Up"""
        return user_clearance.dominates(resource_classification)

    @staticmethod
    def check_write(user_clearance: SecurityLabel, resource_classification: SecurityLabel) -> bool:
        """Bell-LaPadula: No Write Down"""
        return resource_classification.dominates(user_clearance)

async def get_current_user(credentials = Depends(security)) -> User:
    try:
        payload = jwt.decode(credentials.credentials, SECRET_KEY, algorithms=["HS256"])
        username = payload.get("sub")
        user = USERS_DB.get(username)
        if not user:
            raise HTTPException(status_code=401, detail="User not found")
        return user
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

def require_clearance(min_level: SecurityLevel, required_compartments: Set[str] = None):
    """Decorator factory для проверки уровня допуска"""
    async def clearance_checker(user: User = Depends(get_current_user)):
        if user.clearance.level < min_level:
            raise HTTPException(
                status_code=403,
                detail=f"Insufficient clearance level. Required: {min_level.name}"
            )

        if required_compartments:
            if not required_compartments.issubset(user.clearance.compartments):
                missing = required_compartments - user.clearance.compartments
                raise HTTPException(
                    status_code=403,
                    detail=f"Missing compartment access: {missing}"
                )

        return user
    return clearance_checker

@app.post("/auth/login")
async def login(username: str):
    user = USERS_DB.get(username)
    if not user:
        raise HTTPException(status_code=401, detail="User not found")

    token = jwt.encode(
        {
            "sub": username,
            "clearance_level": user.clearance.level,
            "compartments": list(user.clearance.compartments)
        },
        SECRET_KEY,
        algorithm="HS256"
    )
    return {"access_token": token}

@app.get("/resources")
async def list_accessible_resources(user: User = Depends(get_current_user)):
    """Возвращает только ресурсы, доступные пользователю"""
    accessible = []

    for resource_id, resource in RESOURCES_DB.items():
        if MACEnforcer.check_read(user.clearance, resource.classification):
            accessible.append({
                "id": resource.id,
                "name": resource.name,
                "classification": resource.classification.level.name,
                "compartments": list(resource.classification.compartments)
            })

    return {
        "user": user.username,
        "clearance": user.clearance.level.name,
        "accessible_resources": accessible
    }

@app.get("/resources/{resource_id}")
async def get_resource(resource_id: str, user: User = Depends(get_current_user)):
    """Получение ресурса с проверкой MAC"""
    resource = RESOURCES_DB.get(resource_id)

    if not resource:
        raise HTTPException(status_code=404, detail="Resource not found")

    # MAC check: No Read Up
    if not MACEnforcer.check_read(user.clearance, resource.classification):
        raise HTTPException(
            status_code=403,
            detail=f"Access denied. Your clearance ({user.clearance.level.name}) "
                   f"is insufficient for this resource ({resource.classification.level.name})"
        )

    return {
        "resource": resource,
        "accessed_by": user.username,
        "user_clearance": user.clearance.level.name
    }

@app.put("/resources/{resource_id}")
async def update_resource(
    resource_id: str,
    content: str,
    user: User = Depends(get_current_user)
):
    """Обновление ресурса с проверкой MAC"""
    resource = RESOURCES_DB.get(resource_id)

    if not resource:
        raise HTTPException(status_code=404, detail="Resource not found")

    # MAC check: No Write Down
    if not MACEnforcer.check_write(user.clearance, resource.classification):
        raise HTTPException(
            status_code=403,
            detail=f"Write denied. Writing from {user.clearance.level.name} "
                   f"to {resource.classification.level.name} would violate MAC policy"
        )

    resource.content = content
    return {"message": "Resource updated", "resource_id": resource_id}

# Эндпоинты с жёсткими требованиями к clearance
@app.get("/admin/classified")
async def classified_admin_panel(
    user: User = Depends(require_clearance(SecurityLevel.SECRET, {"TECH"}))
):
    """Доступ только для пользователей с SECRET clearance и TECH compartment"""
    return {"message": "Welcome to classified admin panel", "user": user.username}

@app.get("/hr/salaries")
async def hr_salaries(
    user: User = Depends(require_clearance(SecurityLevel.CONFIDENTIAL, {"HR", "FINANCE"}))
):
    """Доступ только для HR и Finance"""
    return {"salaries": "Salary data here..."}
```

### SELinux пример (Linux MAC)

```bash
# SELinux — реализация MAC в Linux

# Проверка статуса SELinux
sestatus

# Типичные команды
getenforce          # Получить текущий режим
setenforce 0        # Permissive mode
setenforce 1        # Enforcing mode

# Просмотр контекста безопасности
ls -Z /etc/passwd
# system_u:object_r:passwd_file_t:s0 /etc/passwd

# Контекст процесса
ps -eZ | grep httpd
# system_u:system_r:httpd_t:s0 ... /usr/sbin/httpd

# Изменение контекста файла
chcon -t httpd_sys_content_t /var/www/html/index.html

# Политики SELinux
semanage boolean -l | grep httpd
setsebool -P httpd_enable_cgi on
```

### AppArmor пример (Linux MAC)

```bash
# AppArmor — альтернатива SELinux

# Статус профилей
aa-status

# Пример профиля /etc/apparmor.d/usr.bin.myapp
profile myapp /usr/bin/myapp {
  # Разрешить чтение
  /etc/myapp.conf r,

  # Разрешить чтение/запись
  /var/log/myapp.log rw,

  # Разрешить выполнение
  /usr/lib/myapp/** ix,

  # Запретить всё остальное (implicit deny)
}

# Применить профиль
apparmor_parser -r /etc/apparmor.d/usr.bin.myapp
```

## Сравнение MAC, DAC и RBAC

| Характеристика | MAC | DAC | RBAC |
|----------------|-----|-----|------|
| Кто устанавливает права | Система/Admin | Владелец ресурса | Admin |
| Основа решений | Security labels | ACL | Роли |
| Гибкость | Низкая | Высокая | Средняя |
| Безопасность | Высокая | Низкая | Средняя |
| Применение | Военные, правительство | Файловые системы | Бизнес-приложения |

## Плюсы MAC

| Преимущество | Описание |
|--------------|----------|
| **Строгий контроль** | Пользователи не могут обойти политики |
| **Защита от утечек** | No Write Down предотвращает утечку данных |
| **Централизация** | Единые политики для всей системы |
| **Аудируемость** | Легко отследить все решения о доступе |
| **Соответствие требованиям** | Подходит для compliance (военные, здравоохранение) |

## Минусы MAC

| Недостаток | Описание |
|------------|----------|
| **Сложность** | Трудно настроить и поддерживать |
| **Негибкость** | Сложно делать исключения |
| **Overhead** | Производительность может страдать |
| **Пользовательский опыт** | Может мешать работе |
| **Административная нагрузка** | Требует постоянного управления |

## Когда использовать

### Подходит для:

- Военных и правительственных систем
- Здравоохранения (HIPAA compliance)
- Финансовых систем (SOX compliance)
- Критической инфраструктуры
- Систем с высокими требованиями к конфиденциальности

### НЕ подходит для:

- Типичных веб-приложений
- Систем, где нужна гибкость
- Стартапов и небольших проектов
- Систем с часто меняющимися требованиями

## Best Practices

1. **Начни с анализа** — определи уровни классификации данных
2. **Минимальные привилегии** — давай минимально необходимый допуск
3. **Документируй политики** — чёткое описание каждого уровня
4. **Тестируй тщательно** — MAC-ошибки сложно отлаживать
5. **Комбинируй с DAC/RBAC** — для гибкости в нижних уровнях
6. **Аудит** — логируй все решения о доступе

## Типичные ошибки

### Ошибка 1: Слишком высокая классификация

```python
# ПЛОХО: всё помечено как TOP_SECRET
all_docs = SecurityLevel.TOP_SECRET

# ХОРОШО: адекватная классификация
public_docs = SecurityLevel.PUBLIC
internal_docs = SecurityLevel.INTERNAL
sensitive_docs = SecurityLevel.CONFIDENTIAL
```

### Ошибка 2: Игнорирование compartments

```python
# ПЛОХО: только уровни
label = SecurityLabel(level=SecurityLevel.SECRET, compartments=set())

# ХОРОШО: уровни + категории
label = SecurityLabel(level=SecurityLevel.SECRET, compartments={"HR", "FINANCE"})
```

### Ошибка 3: Нет процедуры понижения классификации

```python
# Должен быть процесс для:
# - Пересмотра классификации документов
# - Понижения уровня при необходимости
# - Аудита изменений классификации
```

## Резюме

MAC — это строгая модель управления доступом для систем с высокими требованиями безопасности. Она обеспечивает централизованный контроль и предотвращает утечку информации. Однако MAC сложен в реализации и негибок, поэтому используется в основном в государственных и военных системах. Для большинства коммерческих приложений лучше подходит комбинация RBAC/ABAC.
