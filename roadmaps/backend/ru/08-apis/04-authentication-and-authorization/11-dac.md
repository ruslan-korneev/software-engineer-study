# DAC (Discretionary Access Control)

## Что такое DAC?

**DAC (Discretionary Access Control)** — это модель управления доступом, в которой владелец ресурса сам определяет, кто имеет к нему доступ. В отличие от MAC, где политики задаются централизованно, в DAC каждый пользователь может делегировать права на свои ресурсы другим пользователям по своему усмотрению.

## Ключевые концепции

### Основные принципы DAC

| Принцип | Описание |
|---------|----------|
| **Владелец решает** | Создатель ресурса определяет права доступа |
| **Делегирование** | Владелец может передавать права другим |
| **ACL (Access Control List)** | Список разрешений для каждого ресурса |
| **Дискреционность** | Решения на усмотрение владельца |

### Access Control List (ACL)

```
┌─────────────────────────────────────────────────────────────┐
│  File: /documents/report.pdf                                 │
│  Owner: alice                                                │
├─────────────────────────────────────────────────────────────┤
│  Access Control List:                                        │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ Principal        │ Read │ Write │ Execute │ Delete      │ │
│  ├─────────────────────────────────────────────────────────┤ │
│  │ alice (owner)    │  ✓   │   ✓   │    ✓    │    ✓       │ │
│  │ bob              │  ✓   │   ✓   │    ✗    │    ✗       │ │
│  │ charlie          │  ✓   │   ✗   │    ✗    │    ✗       │ │
│  │ group:marketing  │  ✓   │   ✗   │    ✗    │    ✗       │ │
│  │ public           │  ✗   │   ✗   │    ✗    │    ✗       │ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Unix/POSIX права доступа

```bash
-rwxr-x---  1 alice  staff  4096 Jan 15 10:30 report.pdf
 │││ ││││
 │││ │││└── Others: ---  (no access)
 │││ ││└─── Others: ---
 │││ │└──── Others: ---
 │││ └───── Group: r-x  (read, execute)
 ││└─────── Group: r-x
 │└──────── Group: r-x
 └───────── Owner: rwx  (read, write, execute)

# Числовое представление
# rwx = 4+2+1 = 7
# r-x = 4+0+1 = 5
# --- = 0+0+0 = 0
# 750 = rwxr-x---
```

## Практические примеры кода

### Python — базовая реализация DAC

```python
from dataclasses import dataclass, field
from typing import Dict, Set, Optional, List
from enum import Flag, auto
from datetime import datetime

class Permission(Flag):
    """Флаги разрешений"""
    NONE = 0
    READ = auto()
    WRITE = auto()
    EXECUTE = auto()
    DELETE = auto()
    SHARE = auto()
    ADMIN = READ | WRITE | EXECUTE | DELETE | SHARE

@dataclass
class ACLEntry:
    """Запись в ACL"""
    principal_id: str
    principal_type: str  # "user" или "group"
    permissions: Permission
    granted_by: str
    granted_at: datetime = field(default_factory=datetime.utcnow)
    expires_at: Optional[datetime] = None

    def is_expired(self) -> bool:
        if self.expires_at is None:
            return False
        return datetime.utcnow() > self.expires_at

@dataclass
class Resource:
    """Ресурс с ACL"""
    id: str
    name: str
    owner_id: str
    acl: List[ACLEntry] = field(default_factory=list)
    parent_id: Optional[str] = None  # Для наследования
    created_at: datetime = field(default_factory=datetime.utcnow)

class DACSystem:
    """Discretionary Access Control System"""

    def __init__(self):
        self.resources: Dict[str, Resource] = {}
        self.users: Dict[str, Set[str]] = {}  # user_id -> set of group_ids

    def register_user(self, user_id: str, groups: Set[str] = None):
        """Регистрирует пользователя"""
        self.users[user_id] = groups or set()

    def add_user_to_group(self, user_id: str, group_id: str):
        """Добавляет пользователя в группу"""
        if user_id in self.users:
            self.users[user_id].add(group_id)

    def create_resource(
        self,
        resource_id: str,
        name: str,
        owner_id: str,
        parent_id: Optional[str] = None
    ) -> Resource:
        """Создаёт ресурс с владельцем"""
        resource = Resource(
            id=resource_id,
            name=name,
            owner_id=owner_id,
            parent_id=parent_id
        )

        # Владелец автоматически получает полные права
        resource.acl.append(ACLEntry(
            principal_id=owner_id,
            principal_type="user",
            permissions=Permission.ADMIN,
            granted_by="system"
        ))

        self.resources[resource_id] = resource
        return resource

    def grant_permission(
        self,
        grantor_id: str,
        resource_id: str,
        principal_id: str,
        principal_type: str,
        permissions: Permission,
        expires_at: Optional[datetime] = None
    ) -> bool:
        """Выдаёт разрешения (только если grantor имеет право SHARE)"""
        resource = self.resources.get(resource_id)
        if not resource:
            return False

        # Проверяем, может ли grantor выдавать права
        if not self._check_permission(resource, grantor_id, Permission.SHARE):
            return False

        # Проверяем, что grantor не выдаёт больше прав, чем имеет сам
        grantor_perms = self._get_effective_permissions(resource, grantor_id)
        if not (permissions & grantor_perms) == permissions:
            return False

        # Добавляем или обновляем ACL entry
        for entry in resource.acl:
            if entry.principal_id == principal_id and entry.principal_type == principal_type:
                entry.permissions |= permissions
                entry.granted_by = grantor_id
                entry.granted_at = datetime.utcnow()
                entry.expires_at = expires_at
                return True

        resource.acl.append(ACLEntry(
            principal_id=principal_id,
            principal_type=principal_type,
            permissions=permissions,
            granted_by=grantor_id,
            expires_at=expires_at
        ))
        return True

    def revoke_permission(
        self,
        revoker_id: str,
        resource_id: str,
        principal_id: str,
        principal_type: str,
        permissions: Permission
    ) -> bool:
        """Отзывает разрешения"""
        resource = self.resources.get(resource_id)
        if not resource:
            return False

        # Владелец или пользователь с ADMIN могут отзывать права
        if revoker_id != resource.owner_id:
            if not self._check_permission(resource, revoker_id, Permission.ADMIN):
                return False

        for entry in resource.acl:
            if entry.principal_id == principal_id and entry.principal_type == principal_type:
                entry.permissions &= ~permissions
                if entry.permissions == Permission.NONE:
                    resource.acl.remove(entry)
                return True

        return False

    def _get_effective_permissions(
        self,
        resource: Resource,
        user_id: str
    ) -> Permission:
        """Вычисляет эффективные разрешения пользователя"""
        effective = Permission.NONE

        # Проверяем прямые разрешения
        for entry in resource.acl:
            if entry.is_expired():
                continue

            if entry.principal_type == "user" and entry.principal_id == user_id:
                effective |= entry.permissions
            elif entry.principal_type == "group":
                user_groups = self.users.get(user_id, set())
                if entry.principal_id in user_groups:
                    effective |= entry.permissions

        # Наследование от родительского ресурса
        if resource.parent_id:
            parent = self.resources.get(resource.parent_id)
            if parent:
                parent_perms = self._get_effective_permissions(parent, user_id)
                # Наследуем только READ (типичное поведение)
                if Permission.READ in parent_perms:
                    effective |= Permission.READ

        return effective

    def _check_permission(
        self,
        resource: Resource,
        user_id: str,
        required: Permission
    ) -> bool:
        """Проверяет наличие разрешения"""
        effective = self._get_effective_permissions(resource, user_id)
        return (effective & required) == required

    def check_access(
        self,
        user_id: str,
        resource_id: str,
        required_permission: Permission
    ) -> tuple[bool, str]:
        """Проверяет доступ пользователя к ресурсу"""
        resource = self.resources.get(resource_id)
        if not resource:
            return False, "Resource not found"

        if self._check_permission(resource, user_id, required_permission):
            return True, f"Access granted via ACL"

        return False, "Access denied: insufficient permissions"

    def get_acl(self, resource_id: str, requester_id: str) -> Optional[List[dict]]:
        """Возвращает ACL ресурса (только для владельца или ADMIN)"""
        resource = self.resources.get(resource_id)
        if not resource:
            return None

        if requester_id != resource.owner_id:
            if not self._check_permission(resource, requester_id, Permission.ADMIN):
                return None

        return [
            {
                "principal_id": e.principal_id,
                "principal_type": e.principal_type,
                "permissions": str(e.permissions),
                "granted_by": e.granted_by,
                "granted_at": e.granted_at.isoformat(),
                "expires_at": e.expires_at.isoformat() if e.expires_at else None
            }
            for e in resource.acl
        ]

# Демонстрация
def demo_dac():
    dac = DACSystem()

    # Регистрируем пользователей
    dac.register_user("alice", {"marketing"})
    dac.register_user("bob", {"engineering", "marketing"})
    dac.register_user("charlie", {"engineering"})
    dac.register_user("dave", set())

    # Alice создаёт документ
    doc = dac.create_resource("doc1", "Marketing Plan", "alice")
    print(f"Created: {doc.name} (owner: {doc.owner_id})")

    # Alice выдаёт права
    dac.grant_permission("alice", "doc1", "bob", "user", Permission.READ | Permission.WRITE)
    dac.grant_permission("alice", "doc1", "marketing", "group", Permission.READ)

    # Проверяем доступ
    tests = [
        ("alice", "doc1", Permission.DELETE, "Owner delete"),
        ("bob", "doc1", Permission.READ, "Bob read (granted)"),
        ("bob", "doc1", Permission.WRITE, "Bob write (granted)"),
        ("bob", "doc1", Permission.DELETE, "Bob delete (not granted)"),
        ("charlie", "doc1", Permission.READ, "Charlie read (not in marketing)"),
        ("dave", "doc1", Permission.READ, "Dave read (no access)"),
    ]

    print("\nAccess Control Tests:")
    print("-" * 50)
    for user_id, resource_id, permission, description in tests:
        allowed, reason = dac.check_access(user_id, resource_id, permission)
        status = "✅" if allowed else "❌"
        print(f"{status} {description}: {reason}")

    # Bob пытается выдать права (у него нет SHARE)
    result = dac.grant_permission("bob", "doc1", "dave", "user", Permission.READ)
    print(f"\nBob tries to share: {'Success' if result else 'Denied'}")

    # Alice выдаёт Bob право SHARE
    dac.grant_permission("alice", "doc1", "bob", "user", Permission.SHARE)

    # Теперь Bob может выдавать права
    result = dac.grant_permission("bob", "doc1", "dave", "user", Permission.READ)
    print(f"Bob tries to share after getting SHARE permission: {'Success' if result else 'Denied'}")

    # Проверяем, что Dave теперь имеет доступ
    allowed, _ = dac.check_access("dave", "doc1", Permission.READ)
    print(f"Dave can now read: {allowed}")

demo_dac()
```

### Python (FastAPI) — DAC API

```python
from fastapi import FastAPI, Depends, HTTPException, status
from pydantic import BaseModel
from typing import List, Optional, Set
from enum import Flag, auto
from datetime import datetime
import jwt

app = FastAPI()

SECRET_KEY = "your-secret-key"

class Permission(Flag):
    NONE = 0
    READ = auto()
    WRITE = auto()
    DELETE = auto()
    SHARE = auto()
    ADMIN = READ | WRITE | DELETE | SHARE

class ACLEntryCreate(BaseModel):
    principal_id: str
    principal_type: str  # "user" or "group"
    permissions: List[str]  # ["READ", "WRITE"]

class ACLEntryResponse(BaseModel):
    principal_id: str
    principal_type: str
    permissions: List[str]
    granted_by: str
    granted_at: datetime

class ResourceCreate(BaseModel):
    name: str
    parent_id: Optional[str] = None

class ResourceResponse(BaseModel):
    id: str
    name: str
    owner_id: str
    created_at: datetime

# Хранилище (в production — БД)
class Storage:
    resources = {}
    users = {"alice": {"marketing"}, "bob": {"engineering"}, "charlie": set()}

storage = Storage()

def parse_permissions(perm_list: List[str]) -> Permission:
    result = Permission.NONE
    for p in perm_list:
        result |= Permission[p]
    return result

def permission_to_list(perm: Permission) -> List[str]:
    result = []
    for p in Permission:
        if p != Permission.NONE and p != Permission.ADMIN and (perm & p) == p:
            result.append(p.name)
    return result

class User(BaseModel):
    id: str
    groups: Set[str] = set()

async def get_current_user() -> User:
    # В реальности — из JWT
    return User(id="alice", groups={"marketing"})

def check_permission(resource_id: str, user: User, required: Permission) -> bool:
    resource = storage.resources.get(resource_id)
    if not resource:
        return False

    # Владелец имеет все права
    if resource["owner_id"] == user.id:
        return True

    effective = Permission.NONE
    for entry in resource.get("acl", []):
        if entry["principal_type"] == "user" and entry["principal_id"] == user.id:
            effective |= entry["permissions"]
        elif entry["principal_type"] == "group" and entry["principal_id"] in user.groups:
            effective |= entry["permissions"]

    return (effective & required) == required

@app.post("/resources", response_model=ResourceResponse)
async def create_resource(data: ResourceCreate, user: User = Depends(get_current_user)):
    """Создаёт ресурс"""
    import uuid
    resource_id = str(uuid.uuid4())

    resource = {
        "id": resource_id,
        "name": data.name,
        "owner_id": user.id,
        "parent_id": data.parent_id,
        "acl": [],
        "created_at": datetime.utcnow()
    }

    storage.resources[resource_id] = resource

    return ResourceResponse(
        id=resource_id,
        name=data.name,
        owner_id=user.id,
        created_at=resource["created_at"]
    )

@app.get("/resources/{resource_id}")
async def get_resource(resource_id: str, user: User = Depends(get_current_user)):
    """Получает ресурс (требует READ)"""
    if not check_permission(resource_id, user, Permission.READ):
        raise HTTPException(status_code=403, detail="Access denied")

    resource = storage.resources.get(resource_id)
    return {
        "id": resource["id"],
        "name": resource["name"],
        "owner_id": resource["owner_id"]
    }

@app.put("/resources/{resource_id}")
async def update_resource(
    resource_id: str,
    data: ResourceCreate,
    user: User = Depends(get_current_user)
):
    """Обновляет ресурс (требует WRITE)"""
    if not check_permission(resource_id, user, Permission.WRITE):
        raise HTTPException(status_code=403, detail="Access denied")

    resource = storage.resources.get(resource_id)
    resource["name"] = data.name
    return {"message": "Updated"}

@app.delete("/resources/{resource_id}")
async def delete_resource(resource_id: str, user: User = Depends(get_current_user)):
    """Удаляет ресурс (требует DELETE или owner)"""
    resource = storage.resources.get(resource_id)
    if not resource:
        raise HTTPException(status_code=404, detail="Not found")

    if resource["owner_id"] != user.id:
        if not check_permission(resource_id, user, Permission.DELETE):
            raise HTTPException(status_code=403, detail="Access denied")

    del storage.resources[resource_id]
    return {"message": "Deleted"}

@app.get("/resources/{resource_id}/acl")
async def get_acl(resource_id: str, user: User = Depends(get_current_user)):
    """Получает ACL ресурса (только owner или ADMIN)"""
    resource = storage.resources.get(resource_id)
    if not resource:
        raise HTTPException(status_code=404, detail="Not found")

    if resource["owner_id"] != user.id:
        if not check_permission(resource_id, user, Permission.ADMIN):
            raise HTTPException(status_code=403, detail="Access denied")

    return {
        "owner": resource["owner_id"],
        "acl": [
            {
                "principal_id": e["principal_id"],
                "principal_type": e["principal_type"],
                "permissions": permission_to_list(e["permissions"]),
                "granted_by": e["granted_by"]
            }
            for e in resource.get("acl", [])
        ]
    }

@app.post("/resources/{resource_id}/acl")
async def add_acl_entry(
    resource_id: str,
    entry: ACLEntryCreate,
    user: User = Depends(get_current_user)
):
    """Добавляет запись в ACL (требует SHARE)"""
    resource = storage.resources.get(resource_id)
    if not resource:
        raise HTTPException(status_code=404, detail="Not found")

    # Проверяем право SHARE
    if resource["owner_id"] != user.id:
        if not check_permission(resource_id, user, Permission.SHARE):
            raise HTTPException(status_code=403, detail="Cannot share")

    permissions = parse_permissions(entry.permissions)

    # Добавляем запись
    resource["acl"].append({
        "principal_id": entry.principal_id,
        "principal_type": entry.principal_type,
        "permissions": permissions,
        "granted_by": user.id,
        "granted_at": datetime.utcnow()
    })

    return {"message": "ACL entry added"}

@app.delete("/resources/{resource_id}/acl/{principal_type}/{principal_id}")
async def remove_acl_entry(
    resource_id: str,
    principal_type: str,
    principal_id: str,
    user: User = Depends(get_current_user)
):
    """Удаляет запись из ACL"""
    resource = storage.resources.get(resource_id)
    if not resource:
        raise HTTPException(status_code=404, detail="Not found")

    if resource["owner_id"] != user.id:
        if not check_permission(resource_id, user, Permission.ADMIN):
            raise HTTPException(status_code=403, detail="Access denied")

    resource["acl"] = [
        e for e in resource["acl"]
        if not (e["principal_id"] == principal_id and e["principal_type"] == principal_type)
    ]

    return {"message": "ACL entry removed"}
```

## Unix-style ACL пример

```bash
# Просмотр ACL
getfacl /path/to/file

# Установка ACL
setfacl -m u:bob:rw /path/to/file      # Дать bob read/write
setfacl -m g:developers:rx /path/to/dir # Дать группе read/execute
setfacl -x u:bob /path/to/file         # Удалить ACL для bob

# Default ACL (для новых файлов в директории)
setfacl -d -m g:developers:rx /path/to/dir

# Пример вывода getfacl
# file: example.txt
# owner: alice
# group: staff
# user::rwx
# user:bob:rw-
# group::r--
# group:developers:r-x
# mask::rwx
# other::---
```

## Плюсы DAC

| Преимущество | Описание |
|--------------|----------|
| **Гибкость** | Владелец сам решает, кому давать доступ |
| **Простота** | Легко понять и реализовать |
| **Делегирование** | Можно передавать права другим |
| **Широкое использование** | Unix, Windows, файловые системы |

## Минусы DAC

| Недостаток | Описание |
|------------|----------|
| **Уязвимость к Trojan Horse** | Вредоносный код может копировать данные |
| **Нет централизованного контроля** | Каждый владелец решает сам |
| **Сложность аудита** | Трудно отследить все права |
| **Нет защиты от утечек** | Пользователь может скопировать данные |

## Когда использовать

### Подходит для:

- Файловых систем
- Документов и контента пользователей
- Систем совместной работы
- Когда нужна гибкость для пользователей

### НЕ подходит для:

- Систем с высокими требованиями к безопасности
- Когда нужен централизованный контроль
- Военных/правительственных систем
- Защиты от инсайдерских угроз

## Best Practices

1. **Принцип наименьших привилегий** — давай минимально необходимые права
2. **Регулярный аудит** — проверяй ACL на предмет избыточных прав
3. **Expire dates** — устанавливай срок действия разрешений
4. **Группы вместо пользователей** — управляй доступом через группы
5. **Логирование** — записывай изменения ACL

## Типичные ошибки

### Ошибка 1: Избыточные права

```python
# ПЛОХО: всем полный доступ
acl.grant(user, Permission.ADMIN)

# ХОРОШО: только необходимые права
acl.grant(user, Permission.READ)
```

### Ошибка 2: Отсутствие отзыва прав

```python
# ПЛОХО: права остаются после ухода сотрудника

# ХОРОШО: процедура offboarding
def offboard_user(user_id):
    for resource in get_user_resources(user_id):
        revoke_all_permissions(resource, user_id)
```

### Ошибка 3: Нет срока действия

```python
# ПЛОХО: вечные права
acl.grant(contractor, Permission.READ)

# ХОРОШО: временный доступ
acl.grant(contractor, Permission.READ, expires_at=datetime(2024, 12, 31))
```

## DAC vs MAC vs RBAC

| Характеристика | DAC | MAC | RBAC |
|----------------|-----|-----|------|
| Кто управляет | Владелец | Система | Admin |
| Гибкость | Высокая | Низкая | Средняя |
| Безопасность | Средняя | Высокая | Средняя |
| Аудит | Сложный | Простой | Средний |
| Использование | Файлы, docs | Военные | Бизнес-приложения |

## Резюме

DAC — это простая и гибкая модель управления доступом, где владелец ресурса сам определяет права доступа. Она широко используется в файловых системах и системах совместной работы. Главное преимущество — гибкость, главный недостаток — сложность централизованного контроля и аудита. Для систем с высокими требованиями к безопасности рассмотри MAC или комбинацию с RBAC.
