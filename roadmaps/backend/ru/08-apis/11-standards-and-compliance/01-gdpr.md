# GDPR (General Data Protection Regulation)

[prev: 02-server-sent-events](../10-real-time-apis/02-server-sent-events.md) | [next: 02-ccpa](./02-ccpa.md)

---

## Введение

**GDPR** (General Data Protection Regulation) — это регламент Европейского Союза о защите персональных данных, который вступил в силу 25 мая 2018 года. Это один из самых строгих и комплексных законов о конфиденциальности данных в мире, который устанавливает правила сбора, хранения и обработки персональных данных граждан ЕС.

GDPR применяется ко всем организациям, которые обрабатывают персональные данные граждан ЕС, независимо от того, где эти организации расположены. Это означает, что если ваш API обслуживает пользователей из Европы, вы обязаны соблюдать требования GDPR.

---

## История и предпосылки

### Хронология развития

| Год | Событие |
|-----|---------|
| 1995 | Принятие Data Protection Directive 95/46/EC |
| 2012 | Начало разработки GDPR |
| 2016 | Принятие регламента GDPR |
| 2018 | Вступление GDPR в силу (25 мая) |
| 2020+ | Активное правоприменение и крупные штрафы |

### Причины создания GDPR

1. **Устаревание предыдущего законодательства** — Директива 1995 года не учитывала современные технологии
2. **Глобализация данных** — данные стали пересекать границы мгновенно
3. **Рост киберугроз** — увеличение числа утечек данных
4. **Коммерциализация данных** — Big Data и таргетированная реклама

---

## Область применения

### Территориальное действие

GDPR применяется если:

- Организация находится в ЕС
- Организация предлагает товары/услуги гражданам ЕС
- Организация отслеживает поведение лиц в ЕС

```python
# Пример проверки применимости GDPR
def is_gdpr_applicable(user_location: str, company_location: str,
                        offers_to_eu: bool, tracks_eu_behavior: bool) -> bool:
    """
    Определяет, применяется ли GDPR к данной ситуации
    """
    eu_countries = {
        'AT', 'BE', 'BG', 'HR', 'CY', 'CZ', 'DK', 'EE', 'FI', 'FR',
        'DE', 'GR', 'HU', 'IE', 'IT', 'LV', 'LT', 'LU', 'MT', 'NL',
        'PL', 'PT', 'RO', 'SK', 'SI', 'ES', 'SE'
    }

    # Компания в ЕС
    if company_location in eu_countries:
        return True

    # Пользователь из ЕС
    if user_location in eu_countries:
        return True

    # Предложение услуг в ЕС или отслеживание поведения
    if offers_to_eu or tracks_eu_behavior:
        return True

    return False
```

### Ключевые понятия

| Термин | Определение |
|--------|-------------|
| **Data Subject** | Физическое лицо, чьи данные обрабатываются |
| **Data Controller** | Организация, определяющая цели обработки |
| **Data Processor** | Организация, обрабатывающая данные по поручению контроллера |
| **Personal Data** | Любая информация, относящаяся к идентифицированному лицу |
| **Processing** | Любая операция с персональными данными |

---

## Основные принципы GDPR

### 7 ключевых принципов (Статья 5)

1. **Законность, справедливость и прозрачность** (Lawfulness, fairness, transparency)
2. **Ограничение цели** (Purpose limitation)
3. **Минимизация данных** (Data minimization)
4. **Точность** (Accuracy)
5. **Ограничение хранения** (Storage limitation)
6. **Целостность и конфиденциальность** (Integrity and confidentiality)
7. **Подотчётность** (Accountability)

```python
from dataclasses import dataclass
from datetime import datetime, timedelta
from typing import Optional, List
from enum import Enum

class LegalBasis(Enum):
    CONSENT = "consent"
    CONTRACT = "contract"
    LEGAL_OBLIGATION = "legal_obligation"
    VITAL_INTERESTS = "vital_interests"
    PUBLIC_TASK = "public_task"
    LEGITIMATE_INTERESTS = "legitimate_interests"


@dataclass
class GDPRCompliantDataRecord:
    """
    Структура данных, соответствующая принципам GDPR
    """
    # Данные
    data: dict

    # Принцип 1: Законность — правовое основание
    legal_basis: LegalBasis
    consent_timestamp: Optional[datetime] = None

    # Принцип 2: Ограничение цели
    processing_purposes: List[str] = None

    # Принцип 3: Минимизация — только необходимые поля
    required_fields: List[str] = None

    # Принцип 4: Точность
    last_verified: datetime = None

    # Принцип 5: Ограничение хранения
    retention_period: timedelta = None
    deletion_date: datetime = None

    # Принцип 6: Целостность
    encrypted: bool = True
    access_log: List[dict] = None

    def is_retention_expired(self) -> bool:
        """Проверка истечения срока хранения"""
        if self.deletion_date:
            return datetime.now() > self.deletion_date
        return False

    def minimize_data(self) -> dict:
        """Возвращает только необходимые поля"""
        if not self.required_fields:
            return self.data
        return {k: v for k, v in self.data.items() if k in self.required_fields}
```

---

## Права субъектов данных

GDPR предоставляет гражданам ЕС 8 фундаментальных прав:

### 1. Право на информацию (Right to be informed)

```python
from fastapi import FastAPI, Request
from pydantic import BaseModel
from typing import List

app = FastAPI()

class PrivacyNotice(BaseModel):
    """Уведомление о конфиденциальности согласно GDPR"""
    controller_identity: str
    contact_details: str
    dpo_contact: str  # Data Protection Officer
    purposes: List[str]
    legal_basis: str
    recipients: List[str]
    retention_period: str
    data_subject_rights: List[str]
    right_to_withdraw_consent: bool
    right_to_lodge_complaint: bool
    automated_decision_making: bool


@app.get("/api/privacy-notice")
async def get_privacy_notice() -> PrivacyNotice:
    """
    Эндпоинт для получения информации о обработке данных
    Статья 13 GDPR
    """
    return PrivacyNotice(
        controller_identity="Example Company Ltd",
        contact_details="privacy@example.com",
        dpo_contact="dpo@example.com",
        purposes=[
            "Предоставление услуг",
            "Обработка платежей",
            "Коммуникация с клиентами"
        ],
        legal_basis="Исполнение договора (Статья 6(1)(b))",
        recipients=["Платёжные провайдеры", "Хостинг-провайдер"],
        retention_period="3 года после последней активности",
        data_subject_rights=[
            "Право на доступ",
            "Право на исправление",
            "Право на удаление",
            "Право на ограничение обработки",
            "Право на переносимость",
            "Право на возражение"
        ],
        right_to_withdraw_consent=True,
        right_to_lodge_complaint=True,
        automated_decision_making=False
    )
```

### 2. Право на доступ (Right of access)

```python
from fastapi import HTTPException
from typing import Dict, Any
import json

class GDPRDataAccessService:
    """Сервис для реализации права на доступ к данным"""

    async def get_user_data(self, user_id: str) -> Dict[str, Any]:
        """
        Возвращает все данные пользователя
        Статья 15 GDPR — срок ответа: 1 месяц
        """
        user_data = await self._collect_all_user_data(user_id)

        return {
            "subject_access_request": {
                "user_id": user_id,
                "request_date": datetime.now().isoformat(),
                "response_deadline": (datetime.now() + timedelta(days=30)).isoformat(),
                "data_categories": list(user_data.keys()),
                "data": user_data,
                "processing_purposes": await self._get_processing_purposes(user_id),
                "recipients": await self._get_data_recipients(user_id),
                "source": await self._get_data_source(user_id),
                "retention_period": await self._get_retention_info(user_id)
            }
        }

    async def _collect_all_user_data(self, user_id: str) -> Dict[str, Any]:
        """Собирает данные из всех источников"""
        return {
            "profile": await self._get_profile_data(user_id),
            "orders": await self._get_order_history(user_id),
            "communications": await self._get_communication_history(user_id),
            "preferences": await self._get_preferences(user_id),
            "activity_logs": await self._get_activity_logs(user_id)
        }


@app.get("/api/users/{user_id}/data-export")
async def export_user_data(user_id: str, request: Request):
    """
    Subject Access Request (SAR) endpoint
    """
    # Проверка аутентификации и авторизации
    if not await verify_user_identity(request, user_id):
        raise HTTPException(status_code=403, detail="Identity verification required")

    service = GDPRDataAccessService()
    return await service.get_user_data(user_id)
```

### 3. Право на исправление (Right to rectification)

```python
from pydantic import BaseModel, EmailStr
from typing import Optional

class UserDataUpdate(BaseModel):
    """Модель для обновления данных пользователя"""
    email: Optional[EmailStr] = None
    name: Optional[str] = None
    address: Optional[str] = None
    phone: Optional[str] = None


class RectificationService:
    """Сервис для исправления персональных данных"""

    async def rectify_data(self, user_id: str, updates: UserDataUpdate) -> dict:
        """
        Исправление неточных данных
        Статья 16 GDPR
        """
        # Логируем изменения для аудита
        old_data = await self._get_current_data(user_id)

        # Применяем изменения
        await self._apply_updates(user_id, updates)

        # Уведомляем третьи стороны, которым данные были переданы
        await self._notify_recipients(user_id, updates)

        return {
            "status": "rectified",
            "timestamp": datetime.now().isoformat(),
            "updated_fields": [k for k, v in updates.dict().items() if v is not None]
        }

    async def _notify_recipients(self, user_id: str, updates: UserDataUpdate):
        """Уведомление получателей данных об изменениях (Статья 19)"""
        recipients = await self._get_data_recipients(user_id)
        for recipient in recipients:
            await self._send_rectification_notice(recipient, user_id, updates)


@app.patch("/api/users/{user_id}/rectify")
async def rectify_user_data(user_id: str, updates: UserDataUpdate, request: Request):
    """Исправление персональных данных"""
    service = RectificationService()
    return await service.rectify_data(user_id, updates)
```

### 4. Право на удаление / Право быть забытым (Right to erasure)

```python
from enum import Enum
from typing import List

class ErasureException(Enum):
    """Исключения, когда удаление не требуется"""
    FREEDOM_OF_EXPRESSION = "freedom_of_expression"
    LEGAL_OBLIGATION = "legal_obligation"
    PUBLIC_HEALTH = "public_health"
    ARCHIVING = "archiving_public_interest"
    LEGAL_CLAIMS = "legal_claims"


class ErasureService:
    """Сервис для удаления персональных данных"""

    async def process_erasure_request(self, user_id: str) -> dict:
        """
        Обработка запроса на удаление
        Статья 17 GDPR
        """
        # Проверяем исключения
        exceptions = await self._check_erasure_exceptions(user_id)
        if exceptions:
            return {
                "status": "partially_denied",
                "reason": "Legal exceptions apply",
                "exceptions": [e.value for e in exceptions],
                "data_retained": await self._get_retained_data_categories(user_id)
            }

        # Выполняем удаление
        await self._erase_user_data(user_id)

        # Уведомляем третьи стороны
        await self._notify_recipients_of_erasure(user_id)

        return {
            "status": "erased",
            "timestamp": datetime.now().isoformat(),
            "confirmation_id": await self._generate_erasure_certificate(user_id)
        }

    async def _erase_user_data(self, user_id: str):
        """Полное удаление данных из всех систем"""
        erasure_tasks = [
            self._erase_from_primary_db(user_id),
            self._erase_from_backups(user_id),
            self._erase_from_logs(user_id),
            self._erase_from_analytics(user_id),
            self._erase_from_third_parties(user_id)
        ]
        await asyncio.gather(*erasure_tasks)

    async def _check_erasure_exceptions(self, user_id: str) -> List[ErasureException]:
        """Проверка исключений для удаления"""
        exceptions = []

        # Проверка юридических обязательств
        if await self._has_pending_legal_matters(user_id):
            exceptions.append(ErasureException.LEGAL_CLAIMS)

        # Проверка требований по хранению (например, налоговые документы)
        if await self._has_legal_retention_requirement(user_id):
            exceptions.append(ErasureException.LEGAL_OBLIGATION)

        return exceptions


@app.delete("/api/users/{user_id}")
async def delete_user_data(user_id: str, request: Request):
    """
    Right to be forgotten endpoint
    """
    service = ErasureService()
    return await service.process_erasure_request(user_id)
```

### 5. Право на переносимость данных (Right to data portability)

```python
import json
from io import BytesIO

class DataPortabilityService:
    """Сервис для переноса данных"""

    async def export_portable_data(self, user_id: str, format: str = "json") -> bytes:
        """
        Экспорт данных в машиночитаемом формате
        Статья 20 GDPR
        """
        # Собираем данные, предоставленные пользователем
        user_provided_data = await self._get_user_provided_data(user_id)

        if format == "json":
            return json.dumps(user_provided_data, indent=2, ensure_ascii=False).encode()
        elif format == "csv":
            return self._convert_to_csv(user_provided_data)
        elif format == "xml":
            return self._convert_to_xml(user_provided_data)

        raise ValueError(f"Unsupported format: {format}")

    async def _get_user_provided_data(self, user_id: str) -> dict:
        """
        Получает только данные, предоставленные самим пользователем
        (не включает аналитику или выводы системы)
        """
        return {
            "personal_info": await self._get_personal_info(user_id),
            "uploaded_content": await self._get_uploaded_content(user_id),
            "preferences": await self._get_user_preferences(user_id),
            "submitted_forms": await self._get_form_submissions(user_id)
        }


@app.get("/api/users/{user_id}/portable-data")
async def get_portable_data(user_id: str, format: str = "json"):
    """Экспорт данных для переноса"""
    service = DataPortabilityService()
    data = await service.export_portable_data(user_id, format)

    return Response(
        content=data,
        media_type=f"application/{format}",
        headers={
            "Content-Disposition": f"attachment; filename=user_data.{format}"
        }
    )
```

### 6-8. Дополнительные права

```python
class GDPRRightsService:
    """Полный набор прав субъекта данных"""

    async def restrict_processing(self, user_id: str, reason: str) -> dict:
        """
        Право на ограничение обработки (Статья 18)
        """
        await self._mark_data_as_restricted(user_id)
        return {"status": "processing_restricted", "reason": reason}

    async def object_to_processing(self, user_id: str,
                                    processing_type: str,
                                    reason: str) -> dict:
        """
        Право на возражение (Статья 21)
        """
        if processing_type == "direct_marketing":
            # Возражение против маркетинга — безусловное
            await self._stop_marketing(user_id)
            return {"status": "marketing_stopped"}

        # Для других типов требуется оценка
        assessment = await self._assess_objection(user_id, processing_type, reason)
        return assessment

    async def contest_automated_decision(self, user_id: str,
                                          decision_id: str) -> dict:
        """
        Право не подвергаться автоматизированному решению (Статья 22)
        """
        # Запрос человеческого пересмотра
        review_request = await self._request_human_review(user_id, decision_id)
        return {
            "status": "review_requested",
            "review_id": review_request.id,
            "expected_response_time": "5 business days"
        }
```

---

## Правовые основания для обработки

GDPR определяет 6 законных оснований для обработки персональных данных (Статья 6):

```python
from enum import Enum
from dataclasses import dataclass
from typing import Optional
from datetime import datetime

class LegalBasis(Enum):
    """Правовые основания для обработки данных"""
    CONSENT = "consent"  # Согласие
    CONTRACT = "contract"  # Исполнение договора
    LEGAL_OBLIGATION = "legal_obligation"  # Юридическое обязательство
    VITAL_INTERESTS = "vital_interests"  # Жизненно важные интересы
    PUBLIC_TASK = "public_task"  # Публичная задача
    LEGITIMATE_INTERESTS = "legitimate_interests"  # Законные интересы


@dataclass
class ProcessingRecord:
    """Запись о правовом основании обработки"""
    data_category: str
    legal_basis: LegalBasis
    purpose: str
    documented_at: datetime

    # Специфичные для согласия
    consent_text: Optional[str] = None
    consent_timestamp: Optional[datetime] = None
    consent_method: Optional[str] = None  # e.g., "checkbox", "signature"

    # Специфичные для законных интересов
    lia_assessment: Optional[str] = None  # Legitimate Interest Assessment


class ConsentManager:
    """Управление согласиями пользователей"""

    async def record_consent(self, user_id: str,
                              purpose: str,
                              consent_text: str,
                              method: str) -> str:
        """
        Запись согласия с полной документацией
        """
        consent_record = {
            "user_id": user_id,
            "purpose": purpose,
            "consent_text": consent_text,
            "method": method,
            "timestamp": datetime.now().isoformat(),
            "ip_address": await self._get_request_ip(),
            "user_agent": await self._get_user_agent(),
            "version": "1.0"  # Версия текста согласия
        }

        consent_id = await self._store_consent(consent_record)
        return consent_id

    async def withdraw_consent(self, user_id: str, purpose: str) -> dict:
        """
        Отзыв согласия — должен быть так же прост, как и дача согласия
        """
        await self._mark_consent_withdrawn(user_id, purpose)
        await self._stop_processing(user_id, purpose)

        return {
            "status": "consent_withdrawn",
            "purpose": purpose,
            "timestamp": datetime.now().isoformat(),
            "effect": "Processing stopped immediately"
        }

    async def check_consent(self, user_id: str, purpose: str) -> bool:
        """Проверка наличия действующего согласия"""
        consent = await self._get_active_consent(user_id, purpose)
        return consent is not None and not consent.get("withdrawn")
```

---

## Практические рекомендации для API

### 1. Privacy by Design & by Default

```python
from pydantic import BaseModel, Field
from typing import Optional

class UserRegistrationRequest(BaseModel):
    """
    Модель регистрации с учётом Privacy by Design
    """
    # Обязательные поля — минимум необходимого
    email: str
    password: str

    # Опциональные поля с пустыми значениями по умолчанию
    # (Privacy by Default — минимальный сбор данных)
    name: Optional[str] = None
    phone: Optional[str] = None

    # Явное согласие на каждую цель обработки
    consent_marketing: bool = Field(
        default=False,
        description="Согласие на маркетинговые коммуникации"
    )
    consent_analytics: bool = Field(
        default=False,
        description="Согласие на аналитику поведения"
    )
    consent_third_party: bool = Field(
        default=False,
        description="Согласие на передачу данных партнёрам"
    )


class GDPRCompliantAPI:
    """Базовый класс для GDPR-совместимого API"""

    def __init__(self):
        self.default_retention_days = 365 * 3  # 3 года
        self.log_access = True
        self.encrypt_at_rest = True
        self.encrypt_in_transit = True

    async def process_data(self, user_id: str, data: dict, purpose: str):
        """Обработка данных с соблюдением GDPR"""

        # 1. Проверка правового основания
        if not await self._has_legal_basis(user_id, purpose):
            raise PermissionError("No legal basis for processing")

        # 2. Минимизация данных
        minimized_data = self._minimize_data(data, purpose)

        # 3. Логирование доступа
        await self._log_access(user_id, purpose)

        # 4. Обработка
        result = await self._process(minimized_data)

        # 5. Установка срока хранения
        await self._set_retention(user_id, purpose)

        return result

    def _minimize_data(self, data: dict, purpose: str) -> dict:
        """Оставляет только необходимые для цели данные"""
        required_fields = self._get_required_fields(purpose)
        return {k: v for k, v in data.items() if k in required_fields}
```

### 2. Data Protection Impact Assessment (DPIA)

```python
@dataclass
class DPIA:
    """
    Data Protection Impact Assessment
    Требуется для обработки высокого риска (Статья 35)
    """
    project_name: str
    assessment_date: datetime
    assessor: str

    # Описание обработки
    processing_description: str
    purposes: List[str]
    data_categories: List[str]
    data_subjects: List[str]  # Категории субъектов

    # Оценка необходимости
    necessity_assessment: str
    proportionality_assessment: str

    # Оценка рисков
    risks: List[dict]  # {"description": str, "likelihood": str, "impact": str}

    # Меры по снижению рисков
    mitigations: List[dict]  # {"risk": str, "measure": str, "status": str}

    # Заключение
    residual_risk_level: str  # "low", "medium", "high"
    approval_status: str
    dpo_opinion: Optional[str] = None


def requires_dpia(processing_operation: dict) -> bool:
    """
    Определяет, требуется ли DPIA для данной операции
    """
    high_risk_indicators = [
        processing_operation.get("automated_decision_making"),
        processing_operation.get("large_scale_processing"),
        processing_operation.get("sensitive_data"),
        processing_operation.get("systematic_monitoring"),
        processing_operation.get("vulnerable_subjects"),  # дети, пациенты
        processing_operation.get("innovative_technology"),
        processing_operation.get("cross_border_transfer")
    ]

    # DPIA требуется если 2+ индикатора высокого риска
    return sum(bool(i) for i in high_risk_indicators) >= 2
```

### 3. Уведомление об утечках данных

```python
from datetime import datetime, timedelta
from enum import Enum

class BreachSeverity(Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    CRITICAL = "critical"


class DataBreachHandler:
    """
    Обработчик утечек данных
    Статья 33-34 GDPR: уведомление в течение 72 часов
    """

    NOTIFICATION_DEADLINE = timedelta(hours=72)

    async def report_breach(self, breach_details: dict) -> dict:
        """Регистрация и обработка утечки данных"""

        breach_record = {
            "id": self._generate_breach_id(),
            "detected_at": datetime.now().isoformat(),
            "notification_deadline": (
                datetime.now() + self.NOTIFICATION_DEADLINE
            ).isoformat(),
            "details": breach_details,
            "severity": self._assess_severity(breach_details),
            "status": "investigating"
        }

        # Сохраняем запись
        await self._store_breach_record(breach_record)

        # Уведомляем DPO
        await self._notify_dpo(breach_record)

        # Оцениваем необходимость уведомления надзорного органа
        if self._requires_authority_notification(breach_record):
            await self._prepare_authority_notification(breach_record)

        # Оцениваем необходимость уведомления субъектов
        if self._requires_subject_notification(breach_record):
            await self._prepare_subject_notification(breach_record)

        return breach_record

    def _requires_authority_notification(self, breach: dict) -> bool:
        """
        Уведомление надзорного органа требуется,
        если есть риск для прав и свобод субъектов
        """
        return breach["severity"] in [BreachSeverity.MEDIUM,
                                       BreachSeverity.HIGH,
                                       BreachSeverity.CRITICAL]

    def _requires_subject_notification(self, breach: dict) -> bool:
        """
        Уведомление субъектов требуется,
        если есть высокий риск для их прав и свобод
        """
        return breach["severity"] in [BreachSeverity.HIGH,
                                       BreachSeverity.CRITICAL]

    async def _prepare_authority_notification(self, breach: dict) -> dict:
        """
        Подготовка уведомления для надзорного органа
        """
        return {
            "nature_of_breach": breach["details"].get("description"),
            "categories_of_data": breach["details"].get("data_types"),
            "approximate_number_of_subjects": breach["details"].get("affected_count"),
            "dpo_contact": await self._get_dpo_contact(),
            "likely_consequences": self._assess_consequences(breach),
            "measures_taken": breach["details"].get("mitigation_measures", [])
        }
```

### 4. Международная передача данных

```python
class InternationalTransferHandler:
    """
    Обработка международной передачи данных
    Глава V GDPR (Статьи 44-49)
    """

    # Страны с адекватным уровнем защиты (решение Комиссии ЕС)
    ADEQUATE_COUNTRIES = {
        'AD', 'AR', 'CA', 'FO', 'GG', 'IL', 'IM', 'JP', 'JE',
        'NZ', 'KR', 'CH', 'GB', 'UY'
    }

    async def validate_transfer(self, destination_country: str,
                                  data_category: str) -> dict:
        """Проверка законности передачи данных"""

        # Проверка адекватности
        if destination_country in self.ADEQUATE_COUNTRIES:
            return {
                "allowed": True,
                "basis": "adequacy_decision",
                "country": destination_country
            }

        # Проверка стандартных договорных условий (SCC)
        if await self._has_scc(destination_country):
            return {
                "allowed": True,
                "basis": "standard_contractual_clauses",
                "additional_measures_required":
                    await self._check_additional_measures(destination_country)
            }

        # Проверка обязательных корпоративных правил (BCR)
        if await self._has_bcr(destination_country):
            return {
                "allowed": True,
                "basis": "binding_corporate_rules"
            }

        # Проверка исключений (Статья 49)
        derogation = await self._check_derogations(destination_country, data_category)
        if derogation:
            return {
                "allowed": True,
                "basis": "derogation",
                "type": derogation
            }

        return {
            "allowed": False,
            "reason": "No valid transfer mechanism"
        }
```

---

## Штрафы и последствия несоблюдения

### Уровни штрафов

| Уровень | Максимальный штраф | Нарушения |
|---------|-------------------|-----------|
| **Нижний** | До 10 млн EUR или 2% годового оборота | Нарушения требований к документации, безопасности, уведомлениям |
| **Верхний** | До 20 млн EUR или 4% годового оборота | Нарушения принципов обработки, прав субъектов, международных передач |

### Крупнейшие штрафы GDPR

| Год | Компания | Штраф | Причина |
|-----|----------|-------|---------|
| 2023 | Meta (Ireland) | 1.2 млрд EUR | Незаконная передача данных в США |
| 2022 | Meta (Ireland) | 405 млн EUR | Нарушения в Instagram (данные детей) |
| 2021 | Amazon (Luxembourg) | 746 млн EUR | Нарушения в таргетированной рекламе |
| 2022 | Google (France) | 150 млн EUR | Сложности с отказом от cookies |

```python
def calculate_potential_fine(company_info: dict,
                              violation_type: str,
                              violation_severity: str) -> dict:
    """
    Оценка потенциального штрафа
    """
    annual_turnover = company_info.get("annual_turnover_eur", 0)

    if violation_type == "lower_tier":
        max_percentage = 0.02  # 2%
        max_fixed = 10_000_000  # 10 млн EUR
    else:  # upper_tier
        max_percentage = 0.04  # 4%
        max_fixed = 20_000_000  # 20 млн EUR

    percentage_fine = annual_turnover * max_percentage
    max_fine = max(percentage_fine, max_fixed)

    # Корректировка на основе severity
    severity_multipliers = {
        "minor": 0.1,
        "moderate": 0.3,
        "serious": 0.6,
        "very_serious": 1.0
    }

    estimated_fine = max_fine * severity_multipliers.get(violation_severity, 0.5)

    return {
        "max_possible_fine": max_fine,
        "estimated_fine": estimated_fine,
        "factors_considered": [
            "nature_of_violation",
            "intentional_or_negligent",
            "mitigation_actions",
            "previous_violations",
            "cooperation_with_authority"
        ]
    }
```

---

## Чеклист соответствия GDPR

### Организационные меры

- [ ] Назначен Data Protection Officer (DPO) если требуется
- [ ] Проведена инвентаризация данных (data mapping)
- [ ] Созданы Records of Processing Activities (ROPA)
- [ ] Разработана политика конфиденциальности
- [ ] Внедрены процедуры реагирования на запросы субъектов
- [ ] Установлены процедуры уведомления об утечках
- [ ] Проведено обучение персонала

### Технические меры

- [ ] Реализовано шифрование данных (at rest и in transit)
- [ ] Внедрена псевдонимизация где возможно
- [ ] Настроен контроль доступа (принцип наименьших привилегий)
- [ ] Реализовано логирование доступа к персональным данным
- [ ] Внедрены механизмы автоматического удаления устаревших данных
- [ ] Реализованы endpoint'ы для прав субъектов данных

### API-специфичные требования

```python
class GDPRComplianceChecker:
    """Проверка соответствия API требованиям GDPR"""

    def check_api_compliance(self, api_spec: dict) -> dict:
        """Проверка API на соответствие GDPR"""

        checks = {
            "endpoints": {
                "privacy_notice": self._has_privacy_notice_endpoint(api_spec),
                "data_export": self._has_data_export_endpoint(api_spec),
                "data_deletion": self._has_deletion_endpoint(api_spec),
                "consent_management": self._has_consent_endpoints(api_spec),
                "data_rectification": self._has_rectification_endpoint(api_spec)
            },
            "security": {
                "https_only": self._enforces_https(api_spec),
                "authentication": self._has_authentication(api_spec),
                "rate_limiting": self._has_rate_limiting(api_spec),
                "input_validation": self._has_input_validation(api_spec)
            },
            "documentation": {
                "data_categories_documented": self._documents_data_categories(api_spec),
                "retention_periods_defined": self._defines_retention(api_spec),
                "legal_basis_specified": self._specifies_legal_basis(api_spec)
            }
        }

        compliance_score = self._calculate_score(checks)

        return {
            "checks": checks,
            "compliance_score": compliance_score,
            "status": "compliant" if compliance_score >= 0.9 else "needs_work",
            "recommendations": self._generate_recommendations(checks)
        }
```

### Чеклист для разработчиков

- [ ] Все персональные данные собираются с правовым основанием
- [ ] Реализована минимизация данных (собираем только необходимое)
- [ ] Пользователь может получить свои данные в машиночитаемом формате
- [ ] Пользователь может запросить удаление своих данных
- [ ] Пользователь может исправить свои данные
- [ ] Пользователь может отозвать согласие так же легко, как и дать его
- [ ] Логируются все операции с персональными данными
- [ ] Данные защищены при передаче (TLS) и хранении (шифрование)
- [ ] Сроки хранения определены и автоматически соблюдаются
- [ ] При передаче данных за пределы ЕС используются легальные механизмы

---

## Полезные ресурсы

- [Официальный текст GDPR](https://eur-lex.europa.eu/eli/reg/2016/679/oj)
- [Руководства EDPB (European Data Protection Board)](https://edpb.europa.eu/our-work-tools/general-guidance_en)
- [ICO (UK) Guidance](https://ico.org.uk/for-organisations/guide-to-data-protection/guide-to-the-general-data-protection-regulation-gdpr/)
- [CNIL (France) Guidelines](https://www.cnil.fr/en/gdpr-developers-guide)

---

[prev: 02-server-sent-events](../10-real-time-apis/02-server-sent-events.md) | [next: 02-ccpa](./02-ccpa.md)
