# Blackboard Pattern (Паттерн доски)

## Что такое Blackboard Pattern?

Blackboard Pattern (паттерн доски) — это архитектурный паттерн, в котором несколько специализированных подсистем (knowledge sources) совместно работают над решением сложной проблемы, обмениваясь информацией через общую структуру данных (blackboard). Паттерн был разработан для систем искусственного интеллекта и распознавания речи.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Blackboard Pattern                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│   │ Knowledge   │  │ Knowledge   │  │ Knowledge   │            │
│   │  Source 1   │  │  Source 2   │  │  Source 3   │            │
│   │ (Эксперт 1) │  │ (Эксперт 2) │  │ (Эксперт 3) │            │
│   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘            │
│          │                │                │                    │
│          │    читает/     │                │                    │
│          │    пишет       │                │                    │
│          ▼                ▼                ▼                    │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                    BLACKBOARD                            │  │
│   │            (Общее пространство решений)                  │  │
│   │                                                          │  │
│   │   ┌─────────────────────────────────────────────────┐   │  │
│   │   │  Уровень 3: Финальное решение                   │   │  │
│   │   ├─────────────────────────────────────────────────┤   │  │
│   │   │  Уровень 2: Промежуточные гипотезы              │   │  │
│   │   ├─────────────────────────────────────────────────┤   │  │
│   │   │  Уровень 1: Исходные данные                     │   │  │
│   │   └─────────────────────────────────────────────────┘   │  │
│   └─────────────────────────────────────────────────────────┘  │
│                          ▲                                      │
│                          │ управляет                            │
│                   ┌──────┴──────┐                              │
│                   │  Controller │                              │
│                   │ (Планировщик)                              │
│                   └─────────────┘                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Компоненты паттерна

### 1. Blackboard (Доска)

Общая структура данных, где хранятся все гипотезы и решения.

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Dict, List, Any, Optional
from enum import Enum
import threading


class ConfidenceLevel(Enum):
    """Уровни уверенности в гипотезе"""
    LOW = 0.3
    MEDIUM = 0.6
    HIGH = 0.9
    CERTAIN = 1.0


@dataclass
class Hypothesis:
    """Гипотеза на доске"""
    id: str
    level: int  # Уровень абстракции
    content: Any
    confidence: float
    source: str  # Какой knowledge source создал
    created_at: datetime = field(default_factory=datetime.now)
    updated_at: datetime = field(default_factory=datetime.now)
    dependencies: List[str] = field(default_factory=list)


class Blackboard:
    """
    Доска — общее пространство для обмена гипотезами.
    """

    def __init__(self, levels: int = 3):
        self.levels = levels
        self._hypotheses: Dict[int, Dict[str, Hypothesis]] = {
            level: {} for level in range(levels)
        }
        self._lock = threading.RLock()
        self._observers: List[callable] = []

    def add_hypothesis(self, hypothesis: Hypothesis) -> None:
        """Добавить гипотезу на доску"""
        with self._lock:
            level = hypothesis.level
            if level not in self._hypotheses:
                raise ValueError(f"Invalid level: {level}")

            self._hypotheses[level][hypothesis.id] = hypothesis
            self._notify_observers("added", hypothesis)

    def update_hypothesis(self, hypothesis_id: str, level: int,
                          updates: dict) -> Optional[Hypothesis]:
        """Обновить существующую гипотезу"""
        with self._lock:
            if hypothesis_id not in self._hypotheses.get(level, {}):
                return None

            hypothesis = self._hypotheses[level][hypothesis_id]

            for key, value in updates.items():
                if hasattr(hypothesis, key):
                    setattr(hypothesis, key, value)

            hypothesis.updated_at = datetime.now()
            self._notify_observers("updated", hypothesis)
            return hypothesis

    def get_hypotheses(self, level: int = None,
                       min_confidence: float = 0.0) -> List[Hypothesis]:
        """Получить гипотезы с фильтрацией"""
        with self._lock:
            result = []

            levels_to_check = [level] if level is not None else range(self.levels)

            for lvl in levels_to_check:
                for hypothesis in self._hypotheses.get(lvl, {}).values():
                    if hypothesis.confidence >= min_confidence:
                        result.append(hypothesis)

            return result

    def get_hypothesis(self, hypothesis_id: str, level: int) -> Optional[Hypothesis]:
        """Получить конкретную гипотезу"""
        with self._lock:
            return self._hypotheses.get(level, {}).get(hypothesis_id)

    def remove_hypothesis(self, hypothesis_id: str, level: int) -> bool:
        """Удалить гипотезу"""
        with self._lock:
            if hypothesis_id in self._hypotheses.get(level, {}):
                hypothesis = self._hypotheses[level].pop(hypothesis_id)
                self._notify_observers("removed", hypothesis)
                return True
            return False

    def subscribe(self, observer: callable) -> None:
        """Подписаться на изменения"""
        self._observers.append(observer)

    def _notify_observers(self, action: str, hypothesis: Hypothesis) -> None:
        """Уведомить наблюдателей"""
        for observer in self._observers:
            observer(action, hypothesis)

    def get_final_solution(self) -> Optional[Hypothesis]:
        """Получить финальное решение (высший уровень с максимальной уверенностью)"""
        top_level = self.levels - 1
        hypotheses = self.get_hypotheses(level=top_level)

        if not hypotheses:
            return None

        return max(hypotheses, key=lambda h: h.confidence)
```

### 2. Knowledge Source (Источник знаний)

Специализированный модуль, который анализирует данные и создаёт гипотезы.

```python
from abc import ABC, abstractmethod


class KnowledgeSource(ABC):
    """
    Базовый класс для источников знаний.
    Каждый источник специализируется на определённом аспекте проблемы.
    """

    def __init__(self, name: str, blackboard: Blackboard):
        self.name = name
        self.blackboard = blackboard

    @abstractmethod
    def can_contribute(self) -> bool:
        """Проверить, может ли источник внести вклад"""
        pass

    @abstractmethod
    def contribute(self) -> None:
        """Внести вклад на доску"""
        pass

    def create_hypothesis(self, level: int, content: Any,
                          confidence: float,
                          dependencies: List[str] = None) -> Hypothesis:
        """Создать гипотезу"""
        hypothesis = Hypothesis(
            id=f"{self.name}-{uuid.uuid4().hex[:8]}",
            level=level,
            content=content,
            confidence=confidence,
            source=self.name,
            dependencies=dependencies or []
        )
        self.blackboard.add_hypothesis(hypothesis)
        return hypothesis


# Пример: Распознавание изображений
class EdgeDetectorKS(KnowledgeSource):
    """Источник знаний: обнаружение границ"""

    def can_contribute(self) -> bool:
        # Проверяем, есть ли исходное изображение
        raw_data = self.blackboard.get_hypotheses(level=0)
        return any(h.content.get("type") == "raw_image" for h in raw_data)

    def contribute(self) -> None:
        raw_images = [
            h for h in self.blackboard.get_hypotheses(level=0)
            if h.content.get("type") == "raw_image"
        ]

        for raw in raw_images:
            # Обнаружение границ
            edges = self._detect_edges(raw.content["data"])

            self.create_hypothesis(
                level=1,
                content={
                    "type": "edges",
                    "data": edges,
                    "source_image": raw.id
                },
                confidence=0.8,
                dependencies=[raw.id]
            )

    def _detect_edges(self, image_data) -> list:
        # Алгоритм обнаружения границ
        return []


class ShapeRecognizerKS(KnowledgeSource):
    """Источник знаний: распознавание форм"""

    def can_contribute(self) -> bool:
        # Нужны данные о границах
        edges = self.blackboard.get_hypotheses(level=1, min_confidence=0.5)
        return any(h.content.get("type") == "edges" for h in edges)

    def contribute(self) -> None:
        edge_data = [
            h for h in self.blackboard.get_hypotheses(level=1, min_confidence=0.5)
            if h.content.get("type") == "edges"
        ]

        for edge in edge_data:
            shapes = self._recognize_shapes(edge.content["data"])

            for shape in shapes:
                self.create_hypothesis(
                    level=1,
                    content={
                        "type": "shape",
                        "shape_type": shape["type"],
                        "coordinates": shape["coords"]
                    },
                    confidence=shape["confidence"],
                    dependencies=[edge.id]
                )

    def _recognize_shapes(self, edges) -> list:
        # Алгоритм распознавания форм
        return []


class ObjectClassifierKS(KnowledgeSource):
    """Источник знаний: классификация объектов"""

    def can_contribute(self) -> bool:
        shapes = self.blackboard.get_hypotheses(level=1, min_confidence=0.6)
        return any(h.content.get("type") == "shape" for h in shapes)

    def contribute(self) -> None:
        shapes = [
            h for h in self.blackboard.get_hypotheses(level=1, min_confidence=0.6)
            if h.content.get("type") == "shape"
        ]

        # Группируем формы для классификации объектов
        objects = self._classify_objects(shapes)

        for obj in objects:
            self.create_hypothesis(
                level=2,
                content={
                    "type": "object",
                    "class": obj["class"],
                    "bounding_box": obj["bbox"]
                },
                confidence=obj["confidence"],
                dependencies=[s.id for s in obj["shapes"]]
            )

    def _classify_objects(self, shapes) -> list:
        return []
```

### 3. Controller (Контроллер)

Управляет процессом решения, выбирая какой источник знаний активировать.

```python
from typing import List
import random


class Controller:
    """
    Контроллер управляет процессом решения.
    Выбирает, какие источники знаний активировать.
    """

    def __init__(self, blackboard: Blackboard):
        self.blackboard = blackboard
        self.knowledge_sources: List[KnowledgeSource] = []
        self.max_iterations = 100

    def register(self, knowledge_source: KnowledgeSource) -> None:
        """Зарегистрировать источник знаний"""
        self.knowledge_sources.append(knowledge_source)

    def run(self) -> Optional[Hypothesis]:
        """Запустить процесс решения"""
        iteration = 0

        while iteration < self.max_iterations:
            # Находим источники, готовые внести вклад
            ready_sources = [
                ks for ks in self.knowledge_sources
                if ks.can_contribute()
            ]

            if not ready_sources:
                break

            # Выбираем источник для активации
            selected = self._select_source(ready_sources)

            # Источник вносит вклад
            selected.contribute()

            # Проверяем, достигнуто ли решение
            solution = self.blackboard.get_final_solution()
            if solution and solution.confidence >= 0.9:
                return solution

            iteration += 1

        return self.blackboard.get_final_solution()

    def _select_source(self, ready_sources: List[KnowledgeSource]) -> KnowledgeSource:
        """
        Стратегия выбора источника знаний.
        Можно использовать разные стратегии:
        - Приоритетная
        - Round-robin
        - На основе потенциального вклада
        """
        # Простая стратегия: случайный выбор
        return random.choice(ready_sources)


class PriorityController(Controller):
    """Контроллер с приоритетами"""

    def __init__(self, blackboard: Blackboard):
        super().__init__(blackboard)
        self.priorities: Dict[str, int] = {}

    def set_priority(self, source_name: str, priority: int) -> None:
        """Установить приоритет источника"""
        self.priorities[source_name] = priority

    def _select_source(self, ready_sources: List[KnowledgeSource]) -> KnowledgeSource:
        """Выбор по приоритету"""
        return max(
            ready_sources,
            key=lambda ks: self.priorities.get(ks.name, 0)
        )
```

## Полный пример: Система диагностики

```python
# Пример: Медицинская диагностика

from dataclasses import dataclass
from typing import List, Dict


@dataclass
class Symptom:
    """Симптом пациента"""
    name: str
    severity: float  # 0-1
    duration_days: int


@dataclass
class Diagnosis:
    """Диагноз"""
    disease: str
    confidence: float
    supporting_symptoms: List[str]
    recommended_tests: List[str]


class SymptomAnalyzerKS(KnowledgeSource):
    """Анализ симптомов"""

    def can_contribute(self) -> bool:
        raw_data = self.blackboard.get_hypotheses(level=0)
        return any(h.content.get("type") == "symptoms" for h in raw_data)

    def contribute(self) -> None:
        symptoms_data = [
            h for h in self.blackboard.get_hypotheses(level=0)
            if h.content.get("type") == "symptoms"
        ]

        for data in symptoms_data:
            symptoms = data.content["symptoms"]

            # Группируем симптомы по системам организма
            grouped = self._group_by_system(symptoms)

            for system, system_symptoms in grouped.items():
                severity = self._calculate_severity(system_symptoms)

                self.create_hypothesis(
                    level=1,
                    content={
                        "type": "symptom_group",
                        "system": system,
                        "symptoms": system_symptoms,
                        "severity": severity
                    },
                    confidence=0.7 + (severity * 0.2),
                    dependencies=[data.id]
                )

    def _group_by_system(self, symptoms: List[Symptom]) -> Dict[str, List[Symptom]]:
        systems = {
            "respiratory": ["cough", "shortness_of_breath", "wheezing"],
            "cardiovascular": ["chest_pain", "palpitations", "fatigue"],
            "digestive": ["nausea", "abdominal_pain", "diarrhea"],
            "neurological": ["headache", "dizziness", "numbness"]
        }

        grouped = {}
        for symptom in symptoms:
            for system, system_symptoms in systems.items():
                if symptom.name.lower() in system_symptoms:
                    if system not in grouped:
                        grouped[system] = []
                    grouped[system].append(symptom)

        return grouped

    def _calculate_severity(self, symptoms: List[Symptom]) -> float:
        if not symptoms:
            return 0.0
        return sum(s.severity for s in symptoms) / len(symptoms)


class DiseaseMatcherKS(KnowledgeSource):
    """Сопоставление с известными болезнями"""

    DISEASE_PATTERNS = {
        "flu": {
            "systems": ["respiratory"],
            "symptoms": ["fever", "cough", "fatigue"],
            "min_symptoms": 2
        },
        "pneumonia": {
            "systems": ["respiratory"],
            "symptoms": ["fever", "cough", "shortness_of_breath", "chest_pain"],
            "min_symptoms": 3
        },
        "gastritis": {
            "systems": ["digestive"],
            "symptoms": ["abdominal_pain", "nausea", "bloating"],
            "min_symptoms": 2
        }
    }

    def can_contribute(self) -> bool:
        groups = self.blackboard.get_hypotheses(level=1, min_confidence=0.5)
        return any(h.content.get("type") == "symptom_group" for h in groups)

    def contribute(self) -> None:
        symptom_groups = [
            h for h in self.blackboard.get_hypotheses(level=1, min_confidence=0.5)
            if h.content.get("type") == "symptom_group"
        ]

        for disease, pattern in self.DISEASE_PATTERNS.items():
            matching_groups = [
                g for g in symptom_groups
                if g.content["system"] in pattern["systems"]
            ]

            if not matching_groups:
                continue

            # Подсчитываем совпадающие симптомы
            all_symptoms = []
            for group in matching_groups:
                all_symptoms.extend([
                    s.name.lower() for s in group.content["symptoms"]
                ])

            matching_symptoms = [
                s for s in pattern["symptoms"]
                if s in all_symptoms
            ]

            if len(matching_symptoms) >= pattern["min_symptoms"]:
                confidence = len(matching_symptoms) / len(pattern["symptoms"])

                self.create_hypothesis(
                    level=2,
                    content={
                        "type": "disease_candidate",
                        "disease": disease,
                        "matching_symptoms": matching_symptoms,
                        "missing_symptoms": [
                            s for s in pattern["symptoms"]
                            if s not in matching_symptoms
                        ]
                    },
                    confidence=confidence,
                    dependencies=[g.id for g in matching_groups]
                )


class DiagnosisRefinementKS(KnowledgeSource):
    """Уточнение диагноза"""

    def can_contribute(self) -> bool:
        candidates = self.blackboard.get_hypotheses(level=2, min_confidence=0.5)
        return any(h.content.get("type") == "disease_candidate" for h in candidates)

    def contribute(self) -> None:
        candidates = [
            h for h in self.blackboard.get_hypotheses(level=2, min_confidence=0.5)
            if h.content.get("type") == "disease_candidate"
        ]

        if not candidates:
            return

        # Выбираем наиболее вероятного кандидата
        best = max(candidates, key=lambda c: c.confidence)

        # Формируем финальный диагноз
        diagnosis = Diagnosis(
            disease=best.content["disease"],
            confidence=best.confidence,
            supporting_symptoms=best.content["matching_symptoms"],
            recommended_tests=self._get_recommended_tests(best.content["disease"])
        )

        self.create_hypothesis(
            level=2,  # Финальный уровень
            content={
                "type": "final_diagnosis",
                "diagnosis": diagnosis
            },
            confidence=best.confidence * 0.95,  # Немного снижаем уверенность
            dependencies=[best.id]
        )

    def _get_recommended_tests(self, disease: str) -> List[str]:
        tests = {
            "flu": ["Rapid influenza test", "Chest X-ray"],
            "pneumonia": ["Chest X-ray", "Blood test", "Sputum culture"],
            "gastritis": ["Endoscopy", "H. pylori test"]
        }
        return tests.get(disease, [])


# Использование системы диагностики
def run_diagnosis(symptoms: List[Symptom]) -> Optional[Diagnosis]:
    """Запуск процесса диагностики"""

    # Создаём доску с 3 уровнями
    blackboard = Blackboard(levels=3)

    # Добавляем исходные данные
    initial = Hypothesis(
        id="input-symptoms",
        level=0,
        content={
            "type": "symptoms",
            "symptoms": symptoms
        },
        confidence=1.0,
        source="input"
    )
    blackboard.add_hypothesis(initial)

    # Создаём источники знаний
    sources = [
        SymptomAnalyzerKS("symptom_analyzer", blackboard),
        DiseaseMatcherKS("disease_matcher", blackboard),
        DiagnosisRefinementKS("diagnosis_refinement", blackboard)
    ]

    # Создаём контроллер
    controller = PriorityController(blackboard)
    controller.set_priority("symptom_analyzer", 3)
    controller.set_priority("disease_matcher", 2)
    controller.set_priority("diagnosis_refinement", 1)

    for source in sources:
        controller.register(source)

    # Запускаем
    solution = controller.run()

    if solution and solution.content.get("type") == "final_diagnosis":
        return solution.content["diagnosis"]

    return None


# Пример использования
symptoms = [
    Symptom("fever", 0.8, 3),
    Symptom("cough", 0.6, 5),
    Symptom("fatigue", 0.7, 4)
]

diagnosis = run_diagnosis(symptoms)
if diagnosis:
    print(f"Diagnosis: {diagnosis.disease}")
    print(f"Confidence: {diagnosis.confidence:.0%}")
    print(f"Recommended tests: {', '.join(diagnosis.recommended_tests)}")
```

## Плюсы и минусы

### Плюсы

1. **Модульность** — источники знаний независимы
2. **Гибкость** — легко добавлять новые источники
3. **Инкрементальное решение** — решение строится постепенно
4. **Параллелизм** — источники могут работать параллельно
5. **Обработка неопределённости** — работа с вероятностями

### Минусы

1. **Сложность** — сложная координация между компонентами
2. **Отладка** — трудно отследить путь к решению
3. **Производительность** — overhead на синхронизацию
4. **Детерминизм** — результаты могут отличаться

## Когда использовать

### Используйте Blackboard Pattern когда:

- Сложная проблема без чёткого алгоритма решения
- Несколько экспертных систем должны работать вместе
- Решение строится инкрементально
- Нужна работа с неопределённостью
- Распознавание образов, речи, NLP

### Не используйте когда:

- Есть чёткий алгоритм решения
- Простые CRUD операции
- Требуется детерминированный результат
- Маленькая команда

## Заключение

Blackboard Pattern — мощный паттерн для решения сложных недетерминированных задач, где несколько экспертных модулей совместно строят решение. Он особенно полезен в системах искусственного интеллекта, распознавания образов и экспертных системах.
