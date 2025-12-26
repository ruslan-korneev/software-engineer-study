# CCPA (California Consumer Privacy Act)

## Введение

**CCPA** (California Consumer Privacy Act) — это закон штата Калифорния о защите персональных данных потребителей, вступивший в силу 1 января 2020 года. Это один из самых строгих законов о конфиденциальности в США, который предоставляет жителям Калифорнии значительные права в отношении их персональных данных.

В 2023 году CCPA был расширен законом **CPRA** (California Privacy Rights Act), который добавил новые права и ужесточил требования. Вместе они формируют комплексную систему защиты данных потребителей в Калифорнии.

---

## История и развитие

### Хронология

| Год | Событие |
|-----|---------|
| 2018 | Подписание CCPA губернатором Калифорнии (28 июня) |
| 2020 | Вступление CCPA в силу (1 января) |
| 2020 | Начало правоприменения (1 июля) |
| 2020 | Принятие CPRA избирателями (ноябрь) |
| 2023 | Вступление CPRA в силу (1 января) |
| 2023 | Создание California Privacy Protection Agency (CPPA) |

### Причины создания

1. **Отсутствие федерального закона о конфиденциальности** в США
2. **Скандалы с утечками данных** (Cambridge Analytica и др.)
3. **Растущее беспокойство потребителей** о приватности
4. **Успех GDPR** в Европе как модели регулирования

---

## Область применения

### На кого распространяется CCPA

CCPA применяется к коммерческим организациям, которые:

1. **Ведут бизнес в Калифорнии** И
2. Соответствуют **хотя бы одному** из критериев:
   - Годовой валовой доход более **$25 миллионов**
   - Покупают, получают, продают или делятся персональными данными **50,000+ потребителей/устройств/домохозяйств** в год
   - Получают **50%+ дохода от продажи персональных данных**

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class BusinessInfo:
    """Информация о бизнесе для проверки применимости CCPA"""
    annual_revenue: float  # в долларах
    consumers_data_processed: int  # количество потребителей
    revenue_from_data_sales_percent: float  # процент дохода от продажи данных
    operates_in_california: bool


def is_ccpa_applicable(business: BusinessInfo) -> dict:
    """
    Проверяет, применяется ли CCPA к данному бизнесу
    """
    if not business.operates_in_california:
        return {
            "applicable": False,
            "reason": "Бизнес не ведёт деятельность в Калифорнии"
        }

    # Проверка критериев CCPA
    criteria_met = []

    # Критерий 1: Доход > $25 млн
    if business.annual_revenue > 25_000_000:
        criteria_met.append("annual_revenue_over_25m")

    # Критерий 2: Данные 50,000+ потребителей
    if business.consumers_data_processed >= 50_000:
        criteria_met.append("processes_50k_consumers")

    # Критерий 3: 50%+ дохода от продажи данных
    if business.revenue_from_data_sales_percent >= 50:
        criteria_met.append("50_percent_revenue_from_data")

    if criteria_met:
        return {
            "applicable": True,
            "criteria_met": criteria_met,
            "recommendation": "Необходимо обеспечить соответствие CCPA"
        }

    return {
        "applicable": False,
        "reason": "Не соответствует критериям применимости CCPA"
    }


# Пример использования
business = BusinessInfo(
    annual_revenue=30_000_000,
    consumers_data_processed=100_000,
    revenue_from_data_sales_percent=10,
    operates_in_california=True
)

result = is_ccpa_applicable(business)
# {"applicable": True, "criteria_met": ["annual_revenue_over_25m", "processes_50k_consumers"]}
```

### Ключевые определения

| Термин | Определение |
|--------|-------------|
| **Consumer** | Резидент Калифорнии |
| **Business** | Коммерческая организация, соответствующая критериям |
| **Service Provider** | Обрабатывает данные по поручению бизнеса |
| **Third Party** | Получает данные от бизнеса (не сервис-провайдер) |
| **Personal Information** | Информация, идентифицирующая потребителя или домохозяйство |
| **Sale** | Передача данных за денежное или иное вознаграждение |
| **Share** (CPRA) | Передача данных для кросс-контекстной рекламы |

---

## Категории персональных данных

CCPA определяет широкий спектр персональной информации:

```python
from enum import Enum
from typing import List, Set

class CCPADataCategory(Enum):
    """Категории персональных данных по CCPA"""

    # Идентификаторы
    IDENTIFIERS = "identifiers"
    # Имя, псевдоним, адрес, email, номер телефона,
    # SSN, номер водительских прав, паспорт, IP-адрес

    # Коммерческая информация
    COMMERCIAL_INFO = "commercial_information"
    # История покупок, предпочтения

    # Биометрические данные
    BIOMETRIC = "biometric"
    # Отпечатки пальцев, сканы лица, голос

    # Интернет-активность
    INTERNET_ACTIVITY = "internet_activity"
    # История браузера, поисковые запросы, взаимодействие с сайтом

    # Геолокация
    GEOLOCATION = "geolocation"
    # Точные координаты устройства

    # Аудио/Видео
    AUDIO_VIDEO = "audio_visual"
    # Записи звонков, видео

    # Профессиональная информация
    PROFESSIONAL_INFO = "professional_info"
    # История работы, образование

    # Образование
    EDUCATION_INFO = "education"
    # Образовательные записи

    # Выводы и профили
    INFERENCES = "inferences"
    # Созданные профили на основе данных

    # Чувствительные данные (CPRA)
    SENSITIVE = "sensitive"
    # SSN, финансы, здоровье, сексуальная ориентация,
    # религия, этническая принадлежность, точная геолокация


class CCPADataInventory:
    """Инвентаризация данных для соответствия CCPA"""

    def __init__(self):
        self.data_categories: dict = {}

    def register_data_category(self,
                                category: CCPADataCategory,
                                data_elements: List[str],
                                purposes: List[str],
                                retention_period: str,
                                sold: bool = False,
                                shared: bool = False):
        """Регистрация категории данных"""
        self.data_categories[category.value] = {
            "elements": data_elements,
            "purposes": purposes,
            "retention": retention_period,
            "sold_to_third_parties": sold,
            "shared_for_advertising": shared
        }

    def get_disclosure_info(self) -> dict:
        """Информация для раскрытия потребителям"""
        return {
            "categories_collected": list(self.data_categories.keys()),
            "categories_sold": [
                cat for cat, info in self.data_categories.items()
                if info["sold_to_third_parties"]
            ],
            "categories_shared": [
                cat for cat, info in self.data_categories.items()
                if info["shared_for_advertising"]
            ],
            "purposes": self._aggregate_purposes()
        }

    def _aggregate_purposes(self) -> Set[str]:
        purposes = set()
        for info in self.data_categories.values():
            purposes.update(info["purposes"])
        return purposes


# Пример использования
inventory = CCPADataInventory()

inventory.register_data_category(
    category=CCPADataCategory.IDENTIFIERS,
    data_elements=["name", "email", "phone", "address"],
    purposes=["Account management", "Order fulfillment"],
    retention_period="3 years after last activity",
    sold=False,
    shared=False
)

inventory.register_data_category(
    category=CCPADataCategory.INTERNET_ACTIVITY,
    data_elements=["browsing_history", "product_views", "click_data"],
    purposes=["Analytics", "Personalization"],
    retention_period="12 months",
    sold=False,
    shared=True  # Передаётся для рекламы
)
```

---

## Права потребителей

### 1. Право знать (Right to Know)

```python
from fastapi import FastAPI, HTTPException, Request
from pydantic import BaseModel, EmailStr
from typing import List, Optional, Dict, Any
from datetime import datetime, timedelta
from enum import Enum

app = FastAPI()


class CCPADisclosureRequest(BaseModel):
    """Запрос на раскрытие информации"""
    consumer_email: EmailStr
    verification_method: str  # "email", "sms", "id_document"


class CCPADisclosureResponse(BaseModel):
    """Ответ на запрос о раскрытии"""
    request_date: str
    response_deadline: str  # 45 дней
    categories_collected: List[str]
    specific_pieces: Optional[Dict[str, Any]]  # Конкретные данные
    sources: List[str]
    purposes: List[str]
    third_parties: List[str]
    categories_sold_or_shared: List[str]


class CCPARightToKnowService:
    """Сервис для реализации права на информацию"""

    RESPONSE_DEADLINE_DAYS = 45
    LOOKBACK_PERIOD_MONTHS = 12  # Данные за последние 12 месяцев

    async def process_disclosure_request(
        self,
        consumer_id: str,
        include_specific_pieces: bool = False
    ) -> CCPADisclosureResponse:
        """
        Обработка запроса на раскрытие информации
        CCPA требует ответ в течение 45 дней
        """
        request_date = datetime.now()

        # Собираем информацию за последние 12 месяцев
        lookback_start = request_date - timedelta(days=365)

        response = CCPADisclosureResponse(
            request_date=request_date.isoformat(),
            response_deadline=(
                request_date + timedelta(days=self.RESPONSE_DEADLINE_DAYS)
            ).isoformat(),
            categories_collected=await self._get_collected_categories(consumer_id),
            specific_pieces=None,
            sources=await self._get_data_sources(consumer_id),
            purposes=await self._get_purposes(consumer_id),
            third_parties=await self._get_third_parties(consumer_id),
            categories_sold_or_shared=await self._get_sold_shared_categories(consumer_id)
        )

        # Конкретные данные предоставляются только по запросу
        if include_specific_pieces:
            response.specific_pieces = await self._get_specific_data(
                consumer_id,
                lookback_start
            )

        return response

    async def _get_collected_categories(self, consumer_id: str) -> List[str]:
        """Категории собранных данных"""
        return [
            "Identifiers (name, email, phone)",
            "Commercial information (purchase history)",
            "Internet activity (browsing data)",
            "Geolocation (approximate location)"
        ]

    async def _get_data_sources(self, consumer_id: str) -> List[str]:
        """Источники получения данных"""
        return [
            "Directly from consumer",
            "Cookies and tracking technologies",
            "Third-party data providers"
        ]


@app.post("/api/ccpa/know")
async def request_disclosure(
    request_data: CCPADisclosureRequest,
    request: Request
):
    """
    Эндпоинт для запроса на раскрытие информации (Right to Know)
    """
    # Верификация личности потребителя
    consumer_id = await verify_consumer_identity(
        request_data.consumer_email,
        request_data.verification_method
    )

    if not consumer_id:
        raise HTTPException(
            status_code=403,
            detail="Identity verification failed"
        )

    service = CCPARightToKnowService()
    return await service.process_disclosure_request(consumer_id)
```

### 2. Право на удаление (Right to Delete)

```python
class CCPADeletionException(Enum):
    """Исключения, когда удаление не требуется"""
    COMPLETE_TRANSACTION = "complete_transaction"
    SECURITY = "security_purposes"
    ERROR_REPAIR = "identify_and_repair_errors"
    FREE_SPEECH = "free_speech"
    CALECPA = "california_electronic_communications"
    RESEARCH = "scientific_research"
    INTERNAL_USE = "internal_business_use"
    LEGAL_OBLIGATION = "legal_obligation"
    CONSUMER_CONTRACT = "consumer_contract"


class CCPADeletionService:
    """Сервис для удаления данных потребителя"""

    async def process_deletion_request(self, consumer_id: str) -> dict:
        """
        Обработка запроса на удаление
        """
        # Проверка исключений
        exceptions = await self._check_deletion_exceptions(consumer_id)

        # Уведомление сервис-провайдеров об удалении
        await self._notify_service_providers(consumer_id)

        if exceptions:
            # Частичное удаление
            deleted_categories = await self._partial_delete(consumer_id, exceptions)
            return {
                "status": "partially_deleted",
                "deleted_categories": deleted_categories,
                "retained_categories": await self._get_retained_categories(
                    consumer_id, exceptions
                ),
                "retention_reasons": [e.value for e in exceptions]
            }

        # Полное удаление
        await self._full_delete(consumer_id)
        return {
            "status": "deleted",
            "timestamp": datetime.now().isoformat(),
            "confirmation": await self._generate_deletion_confirmation(consumer_id)
        }

    async def _check_deletion_exceptions(
        self,
        consumer_id: str
    ) -> List[CCPADeletionException]:
        """Проверка исключений для удаления"""
        exceptions = []

        # Проверка незавершённых транзакций
        if await self._has_pending_transactions(consumer_id):
            exceptions.append(CCPADeletionException.COMPLETE_TRANSACTION)

        # Проверка юридических обязательств
        if await self._has_legal_hold(consumer_id):
            exceptions.append(CCPADeletionException.LEGAL_OBLIGATION)

        return exceptions

    async def _notify_service_providers(self, consumer_id: str):
        """
        Уведомление сервис-провайдеров об удалении
        CCPA требует передачу запроса на удаление всем провайдерам
        """
        providers = await self._get_service_providers(consumer_id)
        for provider in providers:
            await self._send_deletion_instruction(provider, consumer_id)


@app.delete("/api/ccpa/delete/{consumer_id}")
async def delete_consumer_data(consumer_id: str, request: Request):
    """
    Эндпоинт для удаления данных (Right to Delete)
    """
    # Верификация личности
    if not await verify_consumer_for_deletion(request, consumer_id):
        raise HTTPException(status_code=403, detail="Verification required")

    service = CCPADeletionService()
    return await service.process_deletion_request(consumer_id)
```

### 3. Право на отказ от продажи/передачи (Right to Opt-Out)

```python
class CCPAOptOutService:
    """Сервис для управления отказами от продажи/передачи данных"""

    async def opt_out_of_sale(self, consumer_id: str) -> dict:
        """
        Обработка отказа от продажи данных
        Должен быть выполнен в течение 15 дней
        """
        await self._record_opt_out(consumer_id, "sale")
        await self._stop_selling_data(consumer_id)
        await self._notify_third_parties(consumer_id, "sale")

        return {
            "status": "opted_out",
            "type": "sale",
            "effective_date": datetime.now().isoformat(),
            "expires": None  # Бессрочно, пока потребитель не отменит
        }

    async def opt_out_of_sharing(self, consumer_id: str) -> dict:
        """
        Обработка отказа от передачи данных для рекламы (CPRA)
        """
        await self._record_opt_out(consumer_id, "sharing")
        await self._stop_sharing_data(consumer_id)
        await self._notify_advertising_partners(consumer_id)

        return {
            "status": "opted_out",
            "type": "sharing",
            "effective_date": datetime.now().isoformat()
        }

    async def process_gpc_signal(self, consumer_id: str, gpc_enabled: bool) -> dict:
        """
        Обработка Global Privacy Control (GPC) сигнала
        CPRA требует соблюдение GPC сигналов
        """
        if gpc_enabled:
            # Автоматический opt-out при получении GPC сигнала
            await self.opt_out_of_sale(consumer_id)
            await self.opt_out_of_sharing(consumer_id)

            return {
                "gpc_honored": True,
                "opted_out_of_sale": True,
                "opted_out_of_sharing": True
            }

        return {"gpc_honored": True, "gpc_enabled": False}


# Обязательная ссылка "Do Not Sell or Share My Personal Information"
@app.get("/api/ccpa/opt-out")
async def get_opt_out_page():
    """
    Страница для отказа от продажи/передачи данных
    Должна быть доступна с каждой страницы сайта
    """
    return {
        "title": "Do Not Sell or Share My Personal Information",
        "description": "Используйте эту страницу для отказа от продажи "
                       "или передачи ваших персональных данных",
        "actions": [
            {"id": "opt_out_sale", "label": "Отказаться от продажи данных"},
            {"id": "opt_out_share", "label": "Отказаться от передачи для рекламы"}
        ]
    }


@app.post("/api/ccpa/opt-out")
async def process_opt_out(
    opt_out_type: str,  # "sale", "sharing", "both"
    request: Request
):
    """Обработка запроса на opt-out"""
    consumer_id = await get_consumer_from_request(request)
    service = CCPAOptOutService()

    # Проверка GPC заголовка
    gpc_header = request.headers.get("Sec-GPC")
    if gpc_header == "1":
        return await service.process_gpc_signal(consumer_id, True)

    if opt_out_type == "sale":
        return await service.opt_out_of_sale(consumer_id)
    elif opt_out_type == "sharing":
        return await service.opt_out_of_sharing(consumer_id)
    elif opt_out_type == "both":
        sale_result = await service.opt_out_of_sale(consumer_id)
        share_result = await service.opt_out_of_sharing(consumer_id)
        return {"sale": sale_result, "sharing": share_result}
```

### 4. Право на недискриминацию (Right to Non-Discrimination)

```python
class NonDiscriminationPolicy:
    """
    Политика недискриминации
    CCPA запрещает дискриминацию потребителей, использующих свои права
    """

    PROHIBITED_ACTIONS = [
        "Отказ в товарах или услугах",
        "Взимание разных цен",
        "Предоставление услуг более низкого качества",
        "Угрозы вышеуказанным"
    ]

    ALLOWED_ACTIONS = [
        "Финансовые стимулы за предоставление данных",
        "Программы лояльности (с раскрытием условий)",
        "Разные цены при различии в предоставляемых данных"
    ]

    def validate_pricing_practice(self, practice: dict) -> dict:
        """
        Проверка ценовой практики на соответствие CCPA
        """
        if practice.get("discriminates_on_privacy_rights"):
            return {
                "valid": False,
                "violation": "Дискриминация на основе использования прав CCPA"
            }

        if practice.get("financial_incentive"):
            # Финансовые стимулы разрешены если:
            # 1. Потребитель информирован
            # 2. Есть возможность отказаться
            # 3. Стимул не принуждает
            if not all([
                practice.get("consumer_informed"),
                practice.get("opt_out_available"),
                not practice.get("coercive")
            ]):
                return {
                    "valid": False,
                    "violation": "Финансовый стимул не соответствует требованиям"
                }

        return {"valid": True}


class FinancialIncentiveProgram:
    """Программа финансовых стимулов (разрешена CCPA)"""

    def __init__(self, program_name: str):
        self.program_name = program_name
        self.terms: dict = {}

    def set_terms(self,
                   value_of_data: float,
                   incentive_offered: float,
                   calculation_method: str):
        """
        Установка условий программы
        Должна быть разумная связь между стоимостью данных и стимулом
        """
        self.terms = {
            "data_value": value_of_data,
            "incentive": incentive_offered,
            "calculation": calculation_method
        }

    def get_notice(self) -> dict:
        """Уведомление о финансовых стимулах для потребителя"""
        return {
            "program": self.program_name,
            "incentive": self.terms.get("incentive"),
            "data_collected": self._list_data_collected(),
            "value_calculation": self.terms.get("calculation"),
            "opt_in_required": True,
            "opt_out_available": True,
            "how_to_opt_out": self._get_opt_out_instructions()
        }
```

### 5. Дополнительные права CPRA

```python
class CPRARightsService:
    """Дополнительные права, добавленные CPRA (2023)"""

    async def limit_sensitive_data_use(self, consumer_id: str) -> dict:
        """
        Право на ограничение использования чувствительных данных
        """
        sensitive_categories = [
            "ssn", "financial_accounts", "precise_geolocation",
            "racial_ethnic_origin", "religious_beliefs",
            "health_data", "sexual_orientation", "genetic_data"
        ]

        for category in sensitive_categories:
            await self._limit_use(consumer_id, category)

        return {
            "status": "limited",
            "categories_affected": sensitive_categories,
            "timestamp": datetime.now().isoformat()
        }

    async def correct_inaccurate_data(
        self,
        consumer_id: str,
        corrections: Dict[str, Any]
    ) -> dict:
        """
        Право на исправление неточных данных (CPRA)
        """
        # Применяем исправления
        for field, new_value in corrections.items():
            await self._update_field(consumer_id, field, new_value)

        # Уведомляем получателей данных
        await self._notify_data_recipients(consumer_id, corrections)

        return {
            "status": "corrected",
            "fields_updated": list(corrections.keys()),
            "timestamp": datetime.now().isoformat()
        }

    async def access_automated_decision_info(
        self,
        consumer_id: str
    ) -> dict:
        """
        Право на информацию об автоматизированном принятии решений (CPRA)
        """
        return {
            "automated_decisions": await self._get_automated_decisions(consumer_id),
            "logic_used": await self._get_decision_logic(),
            "expected_outcomes": await self._get_possible_outcomes(),
            "right_to_opt_out": True
        }

    async def opt_out_automated_decisions(self, consumer_id: str) -> dict:
        """
        Право на отказ от автоматизированного принятия решений (CPRA)
        """
        await self._mark_manual_review_required(consumer_id)

        return {
            "status": "opted_out",
            "effect": "All significant decisions will require human review",
            "timestamp": datetime.now().isoformat()
        }
```

---

## Практические рекомендации для API

### 1. Обязательные ссылки и уведомления

```python
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()


class CCPAPrivacyNotice:
    """Уведомление о конфиденциальности для CCPA"""

    @staticmethod
    def generate_notice() -> dict:
        return {
            "effective_date": "2024-01-01",
            "last_updated": "2024-01-01",

            # Какие данные собираются
            "categories_collected": [
                {
                    "category": "Identifiers",
                    "examples": ["name", "email", "phone", "IP address"],
                    "purpose": "Account management, order processing",
                    "retention": "3 years after last activity"
                },
                {
                    "category": "Commercial Information",
                    "examples": ["purchase history", "browsing data"],
                    "purpose": "Order fulfillment, recommendations",
                    "retention": "3 years after last activity"
                }
            ],

            # Источники данных
            "sources": [
                "Directly from you",
                "Automatically through your use of our services",
                "Third-party partners"
            ],

            # Цели использования
            "purposes": [
                "Provide our services",
                "Process payments",
                "Personalize experience",
                "Marketing (with consent)",
                "Security and fraud prevention"
            ],

            # Продажа/передача данных
            "sale_and_sharing": {
                "sells_personal_info": False,
                "shares_for_advertising": True,
                "categories_sold": [],
                "categories_shared": ["Internet activity for targeted advertising"],
                "opt_out_link": "/ccpa/opt-out"
            },

            # Права потребителей
            "consumer_rights": {
                "right_to_know": True,
                "right_to_delete": True,
                "right_to_correct": True,
                "right_to_opt_out": True,
                "right_to_limit_sensitive": True,
                "right_to_non_discrimination": True,
                "request_methods": [
                    "Online form at /ccpa/request",
                    "Email: privacy@example.com",
                    "Phone: 1-800-XXX-XXXX"
                ],
                "verification_required": True,
                "authorized_agent_allowed": True
            },

            # Контактная информация
            "contact": {
                "email": "privacy@example.com",
                "phone": "1-800-XXX-XXXX",
                "address": "123 Privacy St, San Francisco, CA 94105"
            }
        }


@app.get("/privacy-policy")
async def get_privacy_policy():
    """Политика конфиденциальности"""
    return CCPAPrivacyNotice.generate_notice()


# Обязательные ссылки в футере
REQUIRED_FOOTER_LINKS = [
    {"text": "Privacy Policy", "href": "/privacy-policy"},
    {"text": "Do Not Sell or Share My Personal Information", "href": "/ccpa/opt-out"},
    {"text": "Limit the Use of My Sensitive Personal Information", "href": "/ccpa/limit-sensitive"}
]
```

### 2. Верификация личности потребителя

```python
from enum import Enum
from typing import Tuple

class VerificationLevel(Enum):
    """Уровни верификации для разных типов запросов"""
    LOW = "low"  # Для opt-out
    MEDIUM = "medium"  # Для right to know (categories)
    HIGH = "high"  # Для right to know (specific pieces), deletion


class ConsumerVerificationService:
    """Сервис верификации личности потребителя"""

    VERIFICATION_REQUIREMENTS = {
        "opt_out": VerificationLevel.LOW,
        "know_categories": VerificationLevel.MEDIUM,
        "know_specific": VerificationLevel.HIGH,
        "delete": VerificationLevel.HIGH,
        "correct": VerificationLevel.HIGH
    }

    async def verify_consumer(
        self,
        request_type: str,
        consumer_data: dict
    ) -> Tuple[bool, str]:
        """
        Верификация личности потребителя
        """
        required_level = self.VERIFICATION_REQUIREMENTS.get(
            request_type,
            VerificationLevel.HIGH
        )

        if required_level == VerificationLevel.LOW:
            # Достаточно email или cookie
            return await self._low_verification(consumer_data)

        elif required_level == VerificationLevel.MEDIUM:
            # Требуется подтверждение email или 2 точки данных
            return await self._medium_verification(consumer_data)

        else:  # HIGH
            # Требуется подтверждение email + дополнительная верификация
            return await self._high_verification(consumer_data)

    async def _low_verification(self, consumer_data: dict) -> Tuple[bool, str]:
        """Низкий уровень верификации"""
        if consumer_data.get("email") or consumer_data.get("consumer_id"):
            return True, "Verified via identifier"
        return False, "No identifier provided"

    async def _medium_verification(self, consumer_data: dict) -> Tuple[bool, str]:
        """Средний уровень верификации"""
        # Вариант 1: Подтверждённый email
        if await self._verify_email(consumer_data.get("email")):
            return True, "Verified via confirmed email"

        # Вариант 2: Два совпадающих элемента данных
        matched_points = await self._match_data_points(consumer_data)
        if len(matched_points) >= 2:
            return True, f"Verified via {len(matched_points)} data points"

        return False, "Insufficient verification data"

    async def _high_verification(self, consumer_data: dict) -> Tuple[bool, str]:
        """Высокий уровень верификации"""
        # Требуется подтверждённый email
        if not await self._verify_email(consumer_data.get("email")):
            return False, "Email verification required"

        # Плюс дополнительная проверка
        additional_verified = await self._additional_verification(consumer_data)
        if additional_verified:
            return True, "Fully verified"

        return False, "Additional verification required"

    async def verify_authorized_agent(
        self,
        agent_data: dict,
        consumer_authorization: dict
    ) -> Tuple[bool, str]:
        """
        Верификация уполномоченного представителя
        CCPA позволяет потребителям использовать агентов
        """
        # Проверка регистрации агента
        if not await self._is_registered_agent(agent_data.get("registration_id")):
            return False, "Agent not registered with Secretary of State"

        # Проверка авторизации от потребителя
        if not await self._verify_authorization(consumer_authorization):
            return False, "Consumer authorization not verified"

        return True, "Authorized agent verified"


@app.post("/api/ccpa/verify")
async def verify_for_request(
    request_type: str,
    verification_data: dict
):
    """Эндпоинт для верификации перед выполнением запроса"""
    service = ConsumerVerificationService()
    verified, message = await service.verify_consumer(request_type, verification_data)

    if not verified:
        raise HTTPException(status_code=403, detail=message)

    return {"verified": True, "method": message}
```

### 3. Обработка GPC (Global Privacy Control)

```python
from fastapi import Request, Response
from fastapi.middleware.base import BaseHTTPMiddleware


class GPCMiddleware(BaseHTTPMiddleware):
    """
    Middleware для обработки GPC (Global Privacy Control) сигналов
    CPRA требует соблюдение GPC
    """

    async def dispatch(self, request: Request, call_next):
        # Проверяем GPC заголовок
        gpc_header = request.headers.get("Sec-GPC")

        if gpc_header == "1":
            # Пользователь включил GPC — обрабатываем как opt-out
            consumer_id = await self._get_consumer_id(request)

            if consumer_id:
                # Автоматический opt-out
                await self._apply_gpc_opt_out(consumer_id)

        response = await call_next(request)

        # Подтверждаем соблюдение GPC в ответе
        if gpc_header == "1":
            response.headers["Sec-GPC-Honored"] = "1"

        return response

    async def _apply_gpc_opt_out(self, consumer_id: str):
        """Применяет opt-out на основе GPC сигнала"""
        opt_out_service = CCPAOptOutService()
        await opt_out_service.process_gpc_signal(consumer_id, True)


# Регистрация middleware
app.add_middleware(GPCMiddleware)


# Проверка GPC в JavaScript
GPC_JS_CHECK = """
// Проверка GPC в браузере
if (navigator.globalPrivacyControl) {
    // Пользователь включил GPC
    // Отправляем информацию на сервер
    fetch('/api/ccpa/gpc', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({gpc: true})
    });
}
"""
```

### 4. Ведение записей (Recordkeeping)

```python
from datetime import datetime, timedelta
from typing import List, Dict

class CCPARecordkeeping:
    """
    Ведение записей по требованиям CCPA
    Обязательно для бизнесов с данными 10+ млн потребителей
    """

    RETENTION_PERIOD = timedelta(days=24 * 30)  # 24 месяца

    def __init__(self):
        self.requests_log: List[dict] = []
        self.metrics: dict = {}

    async def log_request(self,
                           request_type: str,
                           consumer_id: str,
                           outcome: str,
                           processing_time_days: int):
        """Логирование запроса потребителя"""
        record = {
            "request_id": self._generate_id(),
            "type": request_type,  # know, delete, opt-out, correct
            "consumer_id_hash": self._hash_consumer_id(consumer_id),
            "received_date": datetime.now().isoformat(),
            "outcome": outcome,  # fulfilled, denied, partial
            "processing_time_days": processing_time_days,
            "compliance_met": processing_time_days <= 45
        }

        await self._store_record(record)

    async def generate_annual_metrics(self) -> dict:
        """
        Годовые метрики для раскрытия
        Должны быть опубликованы 1 июля каждого года
        """
        year = datetime.now().year - 1  # Предыдущий год

        requests_by_type = await self._count_by_type(year)
        response_times = await self._calculate_response_times(year)

        return {
            "reporting_year": year,
            "metrics": {
                "requests_to_know": {
                    "received": requests_by_type.get("know", 0),
                    "complied_full": requests_by_type.get("know_fulfilled", 0),
                    "complied_partial": requests_by_type.get("know_partial", 0),
                    "denied": requests_by_type.get("know_denied", 0),
                    "median_response_days": response_times.get("know", 0)
                },
                "requests_to_delete": {
                    "received": requests_by_type.get("delete", 0),
                    "complied_full": requests_by_type.get("delete_fulfilled", 0),
                    "complied_partial": requests_by_type.get("delete_partial", 0),
                    "denied": requests_by_type.get("delete_denied", 0),
                    "median_response_days": response_times.get("delete", 0)
                },
                "requests_to_opt_out": {
                    "received": requests_by_type.get("opt_out", 0),
                    "complied": requests_by_type.get("opt_out_fulfilled", 0),
                    "denied": requests_by_type.get("opt_out_denied", 0),
                    "median_response_days": response_times.get("opt_out", 0)
                }
            },
            "publication_date": f"{year + 1}-07-01"
        }


@app.get("/api/ccpa/metrics")
async def get_ccpa_metrics():
    """Публичные метрики по запросам CCPA"""
    recordkeeper = CCPARecordkeeping()
    return await recordkeeper.generate_annual_metrics()
```

---

## Штрафы и последствия несоблюдения

### Структура штрафов

| Тип нарушения | Штраф |
|---------------|-------|
| Непреднамеренное нарушение | До $2,500 за каждое нарушение |
| Преднамеренное нарушение | До $7,500 за каждое нарушение |
| Нарушение в отношении детей (<16 лет) | До $7,500 за каждое нарушение |

### Частное право на иск (Private Right of Action)

```python
class CCPADataBreachLiability:
    """
    Ответственность за утечку данных
    CCPA предоставляет частное право на иск при утечках
    """

    # Размер ущерба, который может требовать потребитель
    STATUTORY_DAMAGES = {
        "minimum": 100,   # $100 за потребителя
        "maximum": 750    # $750 за потребителя
    }

    def calculate_potential_liability(self,
                                       affected_consumers: int,
                                       actual_damages: float = 0) -> dict:
        """
        Расчёт потенциальной ответственности при утечке
        """
        # Минимальный statutory damages
        min_liability = affected_consumers * self.STATUTORY_DAMAGES["minimum"]
        max_liability = affected_consumers * self.STATUTORY_DAMAGES["maximum"]

        # Потребитель может выбрать actual damages если они выше
        if actual_damages > max_liability:
            return {
                "type": "actual_damages",
                "amount": actual_damages,
                "note": "Actual damages exceed statutory maximum"
            }

        return {
            "type": "statutory_damages",
            "range": {
                "minimum": min_liability,
                "maximum": max_liability
            },
            "affected_consumers": affected_consumers,
            "additional_risks": [
                "Injunctive relief",
                "Declaratory relief",
                "Attorney fees and costs"
            ]
        }

    def assess_breach_for_private_action(self, breach_info: dict) -> dict:
        """
        Оценка утечки на предмет частного иска
        """
        # Частный иск возможен только если:
        # 1. Произошла утечка определённых категорий данных
        covered_categories = [
            "social_security_number",
            "drivers_license_number",
            "financial_account_number",
            "medical_information",
            "health_insurance_information",
            "username_password_security_qa"
        ]

        breached_categories = breach_info.get("data_categories", [])
        covered_breached = set(breached_categories) & set(covered_categories)

        if not covered_breached:
            return {
                "private_action_risk": False,
                "reason": "Breached data not covered by private right of action"
            }

        # 2. Бизнес не реализовал разумные меры безопасности
        security_measures = breach_info.get("security_measures", {})

        return {
            "private_action_risk": True,
            "covered_categories": list(covered_breached),
            "potential_liability": self.calculate_potential_liability(
                breach_info.get("affected_count", 0)
            ),
            "mitigation": "Потребитель должен дать 30 дней на устранение до иска"
        }
```

### Примеры штрафов и дел

| Год | Компания | Результат | Причина |
|-----|----------|-----------|---------|
| 2022 | Sephora | $1.2 млн | Продажа данных без opt-out, игнорирование GPC |
| 2023 | Various | Уведомления о нарушениях | Неполные политики конфиденциальности |

---

## Сравнение CCPA и GDPR

| Аспект | CCPA/CPRA | GDPR |
|--------|-----------|------|
| **Юрисдикция** | Калифорния (USA) | Европейский Союз |
| **Субъекты защиты** | Потребители (резиденты CA) | Физические лица в ЕС |
| **Модель согласия** | Opt-out (отказ) | Opt-in (согласие) |
| **Правовое основание** | Не требуется явно | 6 оснований |
| **Право на удаление** | Да (с исключениями) | Да (с исключениями) |
| **Продажа данных** | Регулируется отдельно | Часть общей обработки |
| **Штрафы** | До $7,500 за нарушение | До 4% годового оборота |
| **Частные иски** | Да (при утечках) | Ограничены |

---

## Чеклист соответствия CCPA

### Организационные меры

- [ ] Определена применимость CCPA к бизнесу
- [ ] Проведена инвентаризация персональных данных
- [ ] Создана/обновлена политика конфиденциальности
- [ ] Обучен персонал по CCPA
- [ ] Назначены ответственные за обработку запросов
- [ ] Установлены процедуры верификации личности

### Технические требования

- [ ] Реализованы endpoint'ы для всех прав потребителей
- [ ] Добавлена ссылка "Do Not Sell or Share My Personal Information"
- [ ] Добавлена ссылка "Limit the Use of My Sensitive Personal Information"
- [ ] Реализована поддержка GPC (Global Privacy Control)
- [ ] Настроено ведение записей запросов (24 месяца)
- [ ] Реализована передача запросов сервис-провайдерам

### Контракты

- [ ] Обновлены договоры с сервис-провайдерами
- [ ] Добавлены требуемые положения в контракты
- [ ] Получены подтверждения от третьих сторон

### Чеклист для разработчиков API

```python
class CCPAComplianceChecker:
    """Проверка API на соответствие CCPA"""

    def check_compliance(self, api_spec: dict) -> dict:
        checks = {
            "endpoints": {
                "right_to_know": self._has_endpoint(api_spec, "know"),
                "right_to_delete": self._has_endpoint(api_spec, "delete"),
                "right_to_opt_out": self._has_endpoint(api_spec, "opt-out"),
                "right_to_correct": self._has_endpoint(api_spec, "correct"),
                "limit_sensitive": self._has_endpoint(api_spec, "limit-sensitive")
            },
            "privacy_notice": {
                "categories_disclosed": self._checks_categories(api_spec),
                "purposes_disclosed": self._checks_purposes(api_spec),
                "sale_sharing_disclosed": self._checks_sale_info(api_spec),
                "rights_explained": self._checks_rights_info(api_spec)
            },
            "gpc_support": {
                "header_recognized": self._checks_gpc_header(api_spec),
                "auto_opt_out": self._checks_gpc_action(api_spec)
            },
            "verification": {
                "identity_verification": self._has_verification(api_spec),
                "agent_support": self._has_agent_verification(api_spec)
            },
            "recordkeeping": {
                "request_logging": self._logs_requests(api_spec),
                "metrics_available": self._has_metrics(api_spec)
            }
        }

        score = self._calculate_score(checks)

        return {
            "checks": checks,
            "compliance_score": score,
            "status": "compliant" if score >= 0.95 else "needs_work"
        }
```

---

## Полезные ресурсы

- [Официальный текст CCPA](https://leginfo.legislature.ca.gov/faces/codes_displayText.xhtml?lawCode=CIV&division=3.&title=1.81.5.&part=4.&chapter=&article=)
- [California Privacy Protection Agency (CPPA)](https://cppa.ca.gov/)
- [CCPA Regulations](https://oag.ca.gov/privacy/ccpa/regulations)
- [CPRA Text](https://vig.cdn.sos.ca.gov/2020/general/pdf/topl-prop24.pdf)
- [Global Privacy Control](https://globalprivacycontrol.org/)
