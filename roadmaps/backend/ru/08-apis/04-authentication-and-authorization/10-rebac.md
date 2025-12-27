# ReBAC (Relationship-Based Access Control)

## Что такое ReBAC?

**ReBAC (Relationship-Based Access Control)** — это модель управления доступом, в которой права доступа определяются на основе отношений между субъектами и объектами. Вместо прямого назначения разрешений, ReBAC моделирует реальные связи: "автор документа", "член команды", "друг пользователя".

## Ключевые концепции

### Основные элементы ReBAC

| Элемент | Описание | Пример |
|---------|----------|--------|
| **Entity** | Объект в системе | User, Document, Team |
| **Relation** | Тип связи | owner, member, viewer |
| **Tuple** | Конкретное отношение | (User:alice, owner, Document:doc1) |
| **Permission** | Право, выводимое из отношений | can_edit = owner OR writer |

### Граф отношений

```
                    ┌─────────────┐
                    │  Team:eng   │
                    └─────────────┘
                          │
                    member │
                          ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ User:alice  │────│ User:bob    │────│ User:charlie│
└─────────────┘    └─────────────┘    └─────────────┘
       │                 │
  owner│            editor│
       ▼                 ▼
┌─────────────┐    ┌─────────────┐
│ Doc:design  │    │ Doc:specs   │
└─────────────┘    └─────────────┘
       │
  parent│
       ▼
┌─────────────┐
│Folder:proj  │
└─────────────┘
```

### Zanzibar (Google) модель

Google Zanzibar — референсная реализация ReBAC:

```
# Формат tuple:
# <namespace>:<object_id>#<relation>@<user_type>:<user_id>

document:doc123#owner@user:alice
document:doc123#viewer@team:engineering#member
folder:project#parent@document:doc123
```

## Практические примеры кода

### Python — базовая реализация ReBAC

```python
from dataclasses import dataclass, field
from typing import Dict, Set, List, Optional, Tuple
from collections import defaultdict

@dataclass(frozen=True)
class Entity:
    """Сущность в системе"""
    type: str
    id: str

    def __str__(self):
        return f"{self.type}:{self.id}"

@dataclass(frozen=True)
class RelationTuple:
    """Кортеж отношения: (object, relation, subject)"""
    object: Entity
    relation: str
    subject: Entity

    def __str__(self):
        return f"{self.object}#{self.relation}@{self.subject}"

class ReBAC:
    """Relationship-Based Access Control Engine"""

    def __init__(self):
        # Хранилище отношений: object -> relation -> set of subjects
        self.tuples: Dict[Entity, Dict[str, Set[Entity]]] = defaultdict(lambda: defaultdict(set))

        # Определения разрешений: namespace -> permission -> set of relations
        self.permission_definitions: Dict[str, Dict[str, Set[str]]] = defaultdict(dict)

        # Иерархия отношений (relation inheritance)
        self.relation_hierarchy: Dict[str, List[str]] = {}

    def define_permission(self, namespace: str, permission: str, relations: Set[str]):
        """Определяет разрешение на основе отношений"""
        self.permission_definitions[namespace][permission] = relations

    def define_relation_hierarchy(self, relation: str, implies: List[str]):
        """Определяет иерархию отношений (owner implies editor, viewer)"""
        self.relation_hierarchy[relation] = implies

    def add_relation(self, tuple: RelationTuple):
        """Добавляет отношение"""
        self.tuples[tuple.object][tuple.relation].add(tuple.subject)

    def remove_relation(self, tuple: RelationTuple):
        """Удаляет отношение"""
        self.tuples[tuple.object][tuple.relation].discard(tuple.subject)

    def get_subjects(self, object: Entity, relation: str) -> Set[Entity]:
        """Получает всех субъектов с данным отношением к объекту"""
        subjects = set(self.tuples[object][relation])

        # Добавляем субъектов из иерархии отношений
        for parent_relation, implied_relations in self.relation_hierarchy.items():
            if relation in implied_relations:
                subjects.update(self.tuples[object][parent_relation])

        return subjects

    def has_relation(self, object: Entity, relation: str, subject: Entity) -> bool:
        """Проверяет наличие прямого или унаследованного отношения"""
        # Прямое отношение
        if subject in self.tuples[object][relation]:
            return True

        # Отношение через иерархию
        for parent_relation, implied_relations in self.relation_hierarchy.items():
            if relation in implied_relations:
                if subject in self.tuples[object][parent_relation]:
                    return True

        return False

    def check_permission(
        self,
        object: Entity,
        permission: str,
        subject: Entity,
        visited: Optional[Set[Tuple[Entity, str]]] = None
    ) -> bool:
        """Проверяет разрешение с учётом отношений и их транзитивности"""
        if visited is None:
            visited = set()

        # Защита от циклов
        key = (object, permission)
        if key in visited:
            return False
        visited.add(key)

        namespace = object.type
        permission_def = self.permission_definitions.get(namespace, {}).get(permission)

        if not permission_def:
            return False

        for relation in permission_def:
            # Проверяем прямое отношение
            if self.has_relation(object, relation, subject):
                return True

            # Проверяем через групповые отношения
            # Например: document#viewer@team:eng#member означает
            # что члены команды eng имеют право viewer на документ
            for related_entity in self.tuples[object][relation]:
                if related_entity.type != "user":
                    # Это группа/команда — проверяем членство
                    if self.has_relation(related_entity, "member", subject):
                        return True

        # Проверяем родительские объекты (иерархия)
        for parent in self.tuples[object].get("parent", set()):
            if self.check_permission(parent, permission, subject, visited):
                return True

        return False

    def expand_permission(self, object: Entity, permission: str) -> Set[Entity]:
        """Возвращает всех субъектов с данным разрешением"""
        subjects = set()
        namespace = object.type
        permission_def = self.permission_definitions.get(namespace, {}).get(permission, set())

        for relation in permission_def:
            subjects.update(self.get_subjects(object, relation))

        return subjects

# Использование
def demo_rebac():
    rebac = ReBAC()

    # Определяем иерархию отношений
    rebac.define_relation_hierarchy("owner", ["editor", "viewer"])
    rebac.define_relation_hierarchy("editor", ["viewer"])

    # Определяем разрешения на основе отношений
    rebac.define_permission("document", "can_view", {"viewer", "editor", "owner"})
    rebac.define_permission("document", "can_edit", {"editor", "owner"})
    rebac.define_permission("document", "can_delete", {"owner"})
    rebac.define_permission("document", "can_share", {"owner"})

    rebac.define_permission("folder", "can_view", {"viewer", "owner"})
    rebac.define_permission("folder", "can_create_docs", {"owner"})

    rebac.define_permission("team", "can_manage", {"admin"})

    # Создаём сущности
    alice = Entity("user", "alice")
    bob = Entity("user", "bob")
    charlie = Entity("user", "charlie")

    team_eng = Entity("team", "engineering")
    doc1 = Entity("document", "design-doc")
    doc2 = Entity("document", "specs")
    folder = Entity("folder", "project")

    # Устанавливаем отношения
    rebac.add_relation(RelationTuple(team_eng, "admin", alice))
    rebac.add_relation(RelationTuple(team_eng, "member", alice))
    rebac.add_relation(RelationTuple(team_eng, "member", bob))
    rebac.add_relation(RelationTuple(team_eng, "member", charlie))

    rebac.add_relation(RelationTuple(doc1, "owner", alice))
    rebac.add_relation(RelationTuple(doc1, "editor", bob))
    rebac.add_relation(RelationTuple(doc1, "viewer", team_eng))  # Вся команда может смотреть

    rebac.add_relation(RelationTuple(doc2, "owner", bob))

    rebac.add_relation(RelationTuple(folder, "owner", alice))
    rebac.add_relation(RelationTuple(doc1, "parent", folder))  # doc1 в folder

    # Тестируем разрешения
    tests = [
        (doc1, "can_view", alice, "Alice is owner"),
        (doc1, "can_view", bob, "Bob is editor"),
        (doc1, "can_view", charlie, "Charlie is team member"),
        (doc1, "can_edit", alice, "Alice is owner"),
        (doc1, "can_edit", bob, "Bob is editor"),
        (doc1, "can_edit", charlie, "Charlie is only viewer"),
        (doc1, "can_delete", alice, "Alice is owner"),
        (doc1, "can_delete", bob, "Bob is only editor"),
        (doc2, "can_edit", alice, "Alice has no relation to doc2"),
        (doc2, "can_edit", bob, "Bob is owner"),
    ]

    print("ReBAC Permission Checks:")
    print("-" * 60)
    for object, permission, subject, note in tests:
        result = rebac.check_permission(object, permission, subject)
        status = "✅" if result else "❌"
        print(f"{status} {subject.id} {permission} {object.id} ({note})")

demo_rebac()
```

### Python (FastAPI) — ReBAC API

```python
from fastapi import FastAPI, Depends, HTTPException, status
from pydantic import BaseModel
from typing import List, Set, Optional
from collections import defaultdict
import jwt

app = FastAPI()

SECRET_KEY = "your-secret-key"

# Модели
class Entity(BaseModel):
    type: str
    id: str

    def __hash__(self):
        return hash((self.type, self.id))

    def __eq__(self, other):
        return self.type == other.type and self.id == other.id

class RelationTupleCreate(BaseModel):
    object_type: str
    object_id: str
    relation: str
    subject_type: str
    subject_id: str

class CheckRequest(BaseModel):
    object_type: str
    object_id: str
    permission: str
    subject_type: str
    subject_id: str

# ReBAC Engine (упрощённая версия)
class ReBAC:
    def __init__(self):
        self.tuples = defaultdict(lambda: defaultdict(set))
        self.permissions = {
            "document": {
                "can_view": {"viewer", "editor", "owner"},
                "can_edit": {"editor", "owner"},
                "can_delete": {"owner"},
                "can_share": {"owner"},
            },
            "folder": {
                "can_view": {"viewer", "owner"},
                "can_create": {"owner"},
            },
            "team": {
                "can_manage": {"admin"},
                "can_invite": {"admin", "owner"},
            }
        }

    def add_tuple(self, obj: Entity, relation: str, subject: Entity):
        self.tuples[(obj.type, obj.id)][relation].add((subject.type, subject.id))

    def remove_tuple(self, obj: Entity, relation: str, subject: Entity):
        self.tuples[(obj.type, obj.id)][relation].discard((subject.type, subject.id))

    def check(self, obj: Entity, permission: str, subject: Entity) -> bool:
        allowed_relations = self.permissions.get(obj.type, {}).get(permission, set())

        for relation in allowed_relations:
            # Прямое отношение
            if (subject.type, subject.id) in self.tuples[(obj.type, obj.id)][relation]:
                return True

            # Через группу
            for entity_key in self.tuples[(obj.type, obj.id)][relation]:
                entity_type, entity_id = entity_key
                if entity_type != "user":
                    # Проверяем членство
                    if (subject.type, subject.id) in self.tuples[(entity_type, entity_id)]["member"]:
                        return True

        return False

    def expand(self, obj: Entity, permission: str) -> List[dict]:
        """Возвращает всех субъектов с разрешением"""
        allowed_relations = self.permissions.get(obj.type, {}).get(permission, set())
        subjects = []

        for relation in allowed_relations:
            for entity_key in self.tuples[(obj.type, obj.id)][relation]:
                entity_type, entity_id = entity_key
                subjects.append({
                    "type": entity_type,
                    "id": entity_id,
                    "via_relation": relation
                })

        return subjects

rebac = ReBAC()

# Инициализация тестовых данных
def init_data():
    # Команда
    team = Entity(type="team", id="engineering")
    rebac.add_tuple(team, "admin", Entity(type="user", id="alice"))
    rebac.add_tuple(team, "member", Entity(type="user", id="alice"))
    rebac.add_tuple(team, "member", Entity(type="user", id="bob"))
    rebac.add_tuple(team, "member", Entity(type="user", id="charlie"))

    # Документ
    doc = Entity(type="document", id="design-doc")
    rebac.add_tuple(doc, "owner", Entity(type="user", id="alice"))
    rebac.add_tuple(doc, "editor", Entity(type="user", id="bob"))
    rebac.add_tuple(doc, "viewer", Entity(type="team", id="engineering"))

init_data()

# API endpoints
@app.post("/tuples")
async def create_tuple(data: RelationTupleCreate):
    """Создаёт новое отношение"""
    obj = Entity(type=data.object_type, id=data.object_id)
    subject = Entity(type=data.subject_type, id=data.subject_id)
    rebac.add_tuple(obj, data.relation, subject)
    return {"message": "Tuple created"}

@app.delete("/tuples")
async def delete_tuple(data: RelationTupleCreate):
    """Удаляет отношение"""
    obj = Entity(type=data.object_type, id=data.object_id)
    subject = Entity(type=data.subject_type, id=data.subject_id)
    rebac.remove_tuple(obj, data.relation, subject)
    return {"message": "Tuple deleted"}

@app.post("/check")
async def check_permission(data: CheckRequest):
    """Проверяет разрешение"""
    obj = Entity(type=data.object_type, id=data.object_id)
    subject = Entity(type=data.subject_type, id=data.subject_id)

    allowed = rebac.check(obj, data.permission, subject)

    return {
        "allowed": allowed,
        "object": f"{data.object_type}:{data.object_id}",
        "permission": data.permission,
        "subject": f"{data.subject_type}:{data.subject_id}"
    }

@app.get("/expand/{object_type}/{object_id}/{permission}")
async def expand_permission(object_type: str, object_id: str, permission: str):
    """Возвращает всех субъектов с разрешением"""
    obj = Entity(type=object_type, id=object_id)
    subjects = rebac.expand(obj, permission)
    return {"subjects": subjects}

# Защищённые эндпоинты с ReBAC
async def get_current_user(authorization: str = None) -> Entity:
    # Упрощённо — в реальности из JWT
    return Entity(type="user", id="alice")

@app.get("/documents/{doc_id}")
async def get_document(doc_id: str, current_user: Entity = Depends(get_current_user)):
    """Получение документа с проверкой ReBAC"""
    doc = Entity(type="document", id=doc_id)

    if not rebac.check(doc, "can_view", current_user):
        raise HTTPException(
            status_code=403,
            detail="You don't have permission to view this document"
        )

    return {"document_id": doc_id, "content": "Document content..."}

@app.put("/documents/{doc_id}")
async def update_document(doc_id: str, current_user: Entity = Depends(get_current_user)):
    """Обновление документа с проверкой ReBAC"""
    doc = Entity(type="document", id=doc_id)

    if not rebac.check(doc, "can_edit", current_user):
        raise HTTPException(
            status_code=403,
            detail="You don't have permission to edit this document"
        )

    return {"message": "Document updated"}

@app.delete("/documents/{doc_id}")
async def delete_document(doc_id: str, current_user: Entity = Depends(get_current_user)):
    """Удаление документа с проверкой ReBAC"""
    doc = Entity(type="document", id=doc_id)

    if not rebac.check(doc, "can_delete", current_user):
        raise HTTPException(
            status_code=403,
            detail="You don't have permission to delete this document"
        )

    return {"message": "Document deleted"}
```

### Использование OpenFGA (Open Source Zanzibar)

```python
# Установка: pip install openfga-sdk

from openfga_sdk import OpenFgaClient, ClientConfiguration
from openfga_sdk.models import *

# Конфигурация
config = ClientConfiguration(
    api_url="http://localhost:8080",
    store_id="your-store-id"
)

async def setup_openfga():
    async with OpenFgaClient(config) as client:
        # Определяем модель авторизации
        authorization_model = WriteAuthorizationModelRequest(
            type_definitions=[
                TypeDefinition(
                    type="user",
                ),
                TypeDefinition(
                    type="team",
                    relations={
                        "member": Userset(this={}),
                        "admin": Userset(this={}),
                    }
                ),
                TypeDefinition(
                    type="document",
                    relations={
                        "owner": Userset(this={}),
                        "editor": Userset(this={}),
                        "viewer": Userset(
                            union=UsersetUnion(
                                child=[
                                    Userset(this={}),
                                    Userset(computed_userset=ObjectRelation(relation="editor")),
                                    Userset(computed_userset=ObjectRelation(relation="owner")),
                                ]
                            )
                        ),
                    },
                    metadata=Metadata(
                        relations={
                            "owner": RelationMetadata(
                                directly_related_user_types=[
                                    RelationReference(type="user")
                                ]
                            ),
                            "editor": RelationMetadata(
                                directly_related_user_types=[
                                    RelationReference(type="user"),
                                    RelationReference(type="team", relation="member")
                                ]
                            ),
                        }
                    )
                ),
            ]
        )

        response = await client.write_authorization_model(authorization_model)
        print(f"Model created: {response.authorization_model_id}")

async def create_tuples():
    async with OpenFgaClient(config) as client:
        # Добавляем отношения
        await client.write(
            WriteRequest(
                writes=TupleKeys(
                    tuple_keys=[
                        TupleKey(
                            user="user:alice",
                            relation="owner",
                            object="document:design-doc"
                        ),
                        TupleKey(
                            user="user:bob",
                            relation="editor",
                            object="document:design-doc"
                        ),
                        TupleKey(
                            user="team:engineering#member",
                            relation="viewer",
                            object="document:design-doc"
                        ),
                    ]
                )
            )
        )

async def check_permission():
    async with OpenFgaClient(config) as client:
        # Проверяем разрешение
        response = await client.check(
            CheckRequest(
                tuple_key=TupleKey(
                    user="user:alice",
                    relation="viewer",
                    object="document:design-doc"
                )
            )
        )
        print(f"Alice can view: {response.allowed}")
```

## Плюсы ReBAC

| Преимущество | Описание |
|--------------|----------|
| **Естественность** | Моделирует реальные отношения |
| **Гибкость** | Легко добавлять новые типы отношений |
| **Масштабируемость** | Эффективно работает с большими графами |
| **Fine-grained** | Очень детальный контроль |
| **Аудируемость** | Можно объяснить, почему доступ разрешён |

## Минусы ReBAC

| Недостаток | Описание |
|------------|----------|
| **Сложность** | Требует понимания графовой модели |
| **Performance** | Обход графа может быть медленным |
| **Consistency** | Нужна консистентность при обновлениях |
| **Tooling** | Меньше готовых решений чем для RBAC |

## Когда использовать

### Подходит для:

- Систем совместной работы (Google Docs, Notion)
- Социальных сетей
- Файловых хранилищ
- Систем с иерархией (папки, организации)
- Когда доступ зависит от отношений

### НЕ подходит для:

- Простых приложений без сложных отношений
- Систем с фиксированными ролями
- Когда достаточно RBAC

## Best Practices

1. **Моделируй реальные отношения** — не придумывай искусственные
2. **Используй иерархию** — owner -> editor -> viewer
3. **Кэшируй результаты** — проверки могут быть дорогими
4. **Денормализуй для производительности** — храни вычисленные разрешения
5. **Аудит изменений** — логируй все изменения отношений

## Популярные реализации

| Проект | Описание |
|--------|----------|
| **Google Zanzibar** | Оригинальная система Google |
| **OpenFGA** | Open source реализация |
| **SpiceDB** | Высокопроизводительная реализация |
| **Ory Keto** | Часть экосистемы Ory |
| **Warrant** | Managed ReBAC сервис |

## Резюме

ReBAC — это мощная модель авторизации для систем со сложными отношениями между сущностями. Она особенно полезна для приложений совместной работы и социальных платформ. Главное преимущество — естественное моделирование реальных отношений. Для реализации рекомендуется использовать готовые решения (OpenFGA, SpiceDB) вместо написания с нуля.
