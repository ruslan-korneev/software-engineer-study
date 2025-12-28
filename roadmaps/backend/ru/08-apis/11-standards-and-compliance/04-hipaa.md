# HIPAA (Health Insurance Portability and Accountability Act)

[prev: 03-pci-dss](./03-pci-dss.md) | [next: 05-pii](./05-pii.md)

---

## Введение

**HIPAA** (Health Insurance Portability and Accountability Act) — это федеральный закон США, принятый в 1996 году, который устанавливает национальные стандарты для защиты конфиденциальной медицинской информации пациентов. HIPAA регулирует, как организации здравоохранения и их партнёры должны обрабатывать, хранить и передавать защищённую медицинскую информацию (PHI).

Для разработчиков API, работающих в сфере здравоохранения (HealthTech, MedTech, Digital Health), соблюдение HIPAA является обязательным требованием при работе с данными пациентов.

---

## История и развитие

### Хронология

| Год | Событие |
|-----|---------|
| 1996 | Принятие HIPAA Конгрессом США |
| 2000 | Публикация Privacy Rule |
| 2003 | Вступление Privacy Rule в силу |
| 2005 | Вступление Security Rule в силу |
| 2009 | Принятие HITECH Act (усиление требований) |
| 2013 | Публикация Omnibus Rule (расширение на Business Associates) |

### Основные компоненты HIPAA

1. **Privacy Rule** — Правила конфиденциальности
2. **Security Rule** — Правила безопасности
3. **Breach Notification Rule** — Правила уведомления об утечках
4. **Enforcement Rule** — Правила правоприменения
5. **HITECH Act** — Усиление ответственности и уведомлений

---

## Область применения

### На кого распространяется HIPAA

```python
from enum import Enum
from dataclasses import dataclass
from typing import List, Optional

class CoveredEntityType(Enum):
    """Типы организаций, подпадающих под HIPAA"""
    HEALTH_PLAN = "health_plan"  # Страховые компании
    HEALTH_CARE_PROVIDER = "health_care_provider"  # Поставщики медуслуг
    HEALTH_CARE_CLEARINGHOUSE = "health_care_clearinghouse"  # Клиринговые центры


class BusinessAssociateType(Enum):
    """Типы бизнес-партнёров"""
    CLOUD_PROVIDER = "cloud_provider"  # AWS, Azure, GCP
    SOFTWARE_VENDOR = "software_vendor"  # Разработчики ПО
    DATA_ANALYTICS = "data_analytics"  # Аналитические компании
    BILLING_SERVICE = "billing_service"  # Биллинговые сервисы
    LEGAL_CONSULTANT = "legal_consultant"  # Юридические консультанты
    IT_SUPPORT = "it_support"  # IT-поддержка


@dataclass
class HIPAAApplicabilityCheck:
    """Проверка применимости HIPAA"""
    organization_type: str
    handles_phi: bool
    electronic_phi: bool
    receives_phi_from_covered_entity: bool


def is_hipaa_applicable(check: HIPAAApplicabilityCheck) -> dict:
    """
    Определяет, применяется ли HIPAA к организации
    """
    # Проверка Covered Entity
    covered_entity_types = [
        "hospital", "clinic", "doctor_practice", "pharmacy",
        "health_insurance", "hmo", "medicare", "medicaid",
        "clearinghouse"
    ]

    if check.organization_type.lower() in covered_entity_types:
        return {
            "applicable": True,
            "role": "covered_entity",
            "requirements": [
                "Privacy Rule",
                "Security Rule",
                "Breach Notification Rule"
            ]
        }

    # Проверка Business Associate
    if check.handles_phi and check.receives_phi_from_covered_entity:
        return {
            "applicable": True,
            "role": "business_associate",
            "requirements": [
                "Security Rule",
                "Breach Notification Rule",
                "Business Associate Agreement (BAA) required"
            ]
        }

    # Не подпадает под HIPAA
    return {
        "applicable": False,
        "note": "Organization does not handle PHI from covered entities"
    }


# Пример проверки
tech_company = HIPAAApplicabilityCheck(
    organization_type="software_vendor",
    handles_phi=True,
    electronic_phi=True,
    receives_phi_from_covered_entity=True
)

result = is_hipaa_applicable(tech_company)
# {"applicable": True, "role": "business_associate", ...}
```

### Protected Health Information (PHI)

```python
from dataclasses import dataclass
from typing import List

@dataclass
class ProtectedHealthInformation:
    """
    PHI — Защищённая медицинская информация

    Это любая информация, которая:
    1. Создаётся или получается covered entity
    2. Относится к физическому или психическому здоровью
    3. Может идентифицировать человека
    """
    pass


class PHIIdentifiers:
    """
    18 идентификаторов, которые делают информацию PHI
    При их удалении данные считаются de-identified
    """

    IDENTIFIERS = [
        "names",  # Имена
        "geographic_data",  # Все географические данные меньше штата
        "dates",  # Все даты (кроме года) связанные с пациентом
        "phone_numbers",  # Номера телефонов
        "fax_numbers",  # Номера факсов
        "email_addresses",  # Email адреса
        "social_security_numbers",  # SSN
        "medical_record_numbers",  # Номера медицинских записей
        "health_plan_beneficiary_numbers",  # Номера бенефициаров
        "account_numbers",  # Номера счетов
        "certificate_license_numbers",  # Номера сертификатов/лицензий
        "vehicle_identifiers",  # Идентификаторы транспортных средств
        "device_identifiers",  # Идентификаторы устройств
        "web_urls",  # Web URL
        "ip_addresses",  # IP адреса
        "biometric_identifiers",  # Биометрические данные
        "full_face_photos",  # Фотографии лица
        "any_other_unique_identifier"  # Любой другой уникальный идентификатор
    ]

    @classmethod
    def de_identify_data(cls, data: dict) -> dict:
        """
        Удаление идентификаторов для de-identification
        Safe Harbor Method
        """
        de_identified = data.copy()

        # Удаляем все 18 идентификаторов
        for identifier in cls.IDENTIFIERS:
            if identifier in de_identified:
                del de_identified[identifier]

        # Добавляем флаг
        de_identified["_de_identified"] = True
        de_identified["_method"] = "safe_harbor"

        return de_identified


# Что является PHI
PHI_EXAMPLES = {
    "is_phi": [
        "Имя пациента + диагноз",
        "Email пациента + результаты анализов",
        "Медицинская запись с любым идентификатором",
        "Счёт за медицинские услуги с именем",
        "Рецепт с адресом пациента"
    ],
    "not_phi": [
        "Агрегированная статистика заболеваний",
        "De-identified данные (удалены все 18 идентификаторов)",
        "Информация об умерших (более 50 лет)",
        "Данные сотрудников (не пациентов)"
    ]
}
```

---

## Privacy Rule (Правило конфиденциальности)

### Основные требования

```python
from enum import Enum
from typing import List, Optional
from datetime import datetime

class UseDisclosurePurpose(Enum):
    """Разрешённые цели использования и раскрытия PHI"""
    TREATMENT = "treatment"  # Лечение
    PAYMENT = "payment"  # Оплата
    HEALTHCARE_OPERATIONS = "healthcare_operations"  # Операционная деятельность
    PUBLIC_HEALTH = "public_health"  # Общественное здоровье
    LEGAL_REQUIREMENT = "legal_requirement"  # Юридическое требование
    RESEARCH = "research"  # Исследования (с согласия или IRB)


class PrivacyRuleRequirements:
    """
    Требования Privacy Rule
    """

    def get_permitted_uses(self) -> dict:
        """
        Использование PHI БЕЗ согласия пациента
        (разрешено для TPO - Treatment, Payment, Operations)
        """
        return {
            "treatment": {
                "description": "Предоставление медицинской помощи",
                "examples": [
                    "Передача данных между врачами для консультации",
                    "Отправка рецепта в аптеку",
                    "Направление к специалисту"
                ]
            },
            "payment": {
                "description": "Биллинг и оплата услуг",
                "examples": [
                    "Отправка счёта страховой компании",
                    "Проверка страхового покрытия",
                    "Взыскание долгов"
                ]
            },
            "healthcare_operations": {
                "description": "Административная деятельность",
                "examples": [
                    "Контроль качества",
                    "Обучение персонала",
                    "Аудит"
                ]
            }
        }

    def get_required_authorization(self) -> List[str]:
        """
        Случаи, требующие ПИСЬМЕННОГО согласия пациента
        """
        return [
            "Маркетинг (за редким исключением)",
            "Продажа PHI",
            "Психотерапевтические заметки",
            "Использование в исследованиях (большинство случаев)",
            "Раскрытие работодателю"
        ]


class MinimumNecessaryStandard:
    """
    Стандарт минимальной необходимости
    Использовать/раскрывать только минимум PHI для цели
    """

    def apply_minimum_necessary(self,
                                  full_record: dict,
                                  purpose: UseDisclosurePurpose) -> dict:
        """
        Применяет принцип минимальной необходимости
        """
        purpose_fields = {
            UseDisclosurePurpose.TREATMENT: [
                "patient_id", "name", "dob", "diagnoses",
                "medications", "allergies", "vitals", "history"
            ],
            UseDisclosurePurpose.PAYMENT: [
                "patient_id", "name", "dob", "insurance_id",
                "service_codes", "dates_of_service", "charges"
            ],
            UseDisclosurePurpose.HEALTHCARE_OPERATIONS: [
                "patient_id", "service_codes", "outcomes",
                "quality_metrics"  # Без имени если возможно
            ]
        }

        allowed_fields = purpose_fields.get(purpose, [])

        return {k: v for k, v in full_record.items() if k in allowed_fields}

    def get_exceptions(self) -> List[str]:
        """
        Исключения из стандарта минимальной необходимости
        """
        return [
            "Передача для лечения (врач-врач)",
            "Запрос самого пациента",
            "Раскрытие по авторизации пациента",
            "Раскрытие в HHS для расследования",
            "Раскрытие по закону"
        ]
```

### Права пациентов

```python
from dataclasses import dataclass
from datetime import datetime, timedelta
from typing import Dict, Any, Optional

@dataclass
class PatientRightsRequest:
    """Запрос пациента на реализацию прав"""
    patient_id: str
    right_type: str
    request_date: datetime
    details: Dict[str, Any]


class PatientRightsService:
    """
    Реализация прав пациентов по HIPAA Privacy Rule
    """

    # Сроки ответа на запросы
    RESPONSE_DEADLINES = {
        "access": 30,  # дней (можно продлить ещё на 30)
        "amendment": 60,  # дней
        "accounting": 60,  # дней
        "restriction": 0,  # нет срока, но должен быть разумным
    }

    async def process_access_request(self, patient_id: str) -> dict:
        """
        Право на доступ к PHI
        Пациент может получить копию своих медицинских записей
        """
        records = await self._get_patient_records(patient_id)

        return {
            "request_type": "access",
            "patient_id": patient_id,
            "response_deadline": (
                datetime.now() + timedelta(days=self.RESPONSE_DEADLINES["access"])
            ).isoformat(),
            "records": records,
            "format_options": ["paper", "electronic"],
            "fee": self._calculate_fee(records)  # Только разумные расходы
        }

    async def process_amendment_request(self,
                                         patient_id: str,
                                         amendment: dict) -> dict:
        """
        Право на внесение поправок
        Пациент может запросить исправление неточной информации
        """
        # Оцениваем запрос
        assessment = await self._assess_amendment(patient_id, amendment)

        if assessment["approved"]:
            await self._apply_amendment(patient_id, amendment)
            # Уведомляем тех, кому данные были раскрыты
            await self._notify_recipients(patient_id, amendment)

            return {
                "status": "approved",
                "amendment_applied": True,
                "notification_sent": True
            }
        else:
            # Отказ должен быть обоснован
            return {
                "status": "denied",
                "reason": assessment["denial_reason"],
                "patient_can_submit_statement": True,
                "statement_will_be_attached": True
            }

    async def process_accounting_request(self, patient_id: str) -> dict:
        """
        Право на учёт раскрытий
        Пациент может узнать, кому раскрывалась его PHI
        """
        # Последние 6 лет (можно запросить меньший период)
        disclosures = await self._get_disclosure_history(
            patient_id,
            years=6
        )

        # Исключения (не включаются в accounting)
        excluded = [
            "TPO (treatment, payment, operations)",
            "Раскрытие самому пациенту",
            "По авторизации пациента",
            "В directory (справочник)"
        ]

        return {
            "request_type": "accounting_of_disclosures",
            "period": "last_6_years",
            "disclosures": [d for d in disclosures if not d.get("excluded")],
            "excluded_categories": excluded,
            "each_disclosure_includes": [
                "date",
                "recipient_name",
                "recipient_address",
                "description_of_phi",
                "purpose"
            ]
        }

    async def process_restriction_request(self,
                                           patient_id: str,
                                           restriction: dict) -> dict:
        """
        Право на ограничение использования
        Пациент может запросить ограничения на использование PHI
        """
        # Организация НЕ ОБЯЗАНА соглашаться, КРОМЕ:
        # Если пациент оплатил услугу полностью сам и просит
        # не раскрывать страховщику

        if restriction.get("self_pay_restriction"):
            # Обязательно согласиться
            await self._apply_restriction(patient_id, restriction)
            return {
                "status": "accepted",
                "mandatory": True,
                "reason": "Self-pay restriction is mandatory under HIPAA"
            }

        # Для других случаев — на усмотрение организации
        decision = await self._evaluate_restriction(patient_id, restriction)
        return decision

    async def provide_notice_of_privacy_practices(self) -> dict:
        """
        Notice of Privacy Practices (NPP)
        Обязательное уведомление о практиках конфиденциальности
        """
        return {
            "title": "Notice of Privacy Practices",
            "required_content": [
                "Как PHI может использоваться и раскрываться",
                "Права пациента в отношении PHI",
                "Обязанности организации по защите PHI",
                "Как подать жалобу",
                "Контактная информация Privacy Officer"
            ],
            "must_be_provided": "При первом обращении",
            "acknowledgment_required": True,
            "posted_prominently": True
        }
```

---

## Security Rule (Правило безопасности)

### Три категории защитных мер

```python
from enum import Enum
from dataclasses import dataclass
from typing import List, Dict

class SafeguardCategory(Enum):
    ADMINISTRATIVE = "administrative"
    PHYSICAL = "physical"
    TECHNICAL = "technical"


class AdministrativeSafeguards:
    """
    Административные защитные меры
    """

    def get_requirements(self) -> Dict[str, dict]:
        return {
            "security_management_process": {
                "status": "required",
                "components": [
                    "Risk analysis",
                    "Risk management",
                    "Sanction policy",
                    "Information system activity review"
                ]
            },
            "assigned_security_responsibility": {
                "status": "required",
                "description": "Назначение Security Officer"
            },
            "workforce_security": {
                "status": "required",
                "components": [
                    "Authorization/supervision",
                    "Workforce clearance procedure",
                    "Termination procedures"
                ]
            },
            "information_access_management": {
                "status": "required",
                "components": [
                    "Access authorization",
                    "Access establishment and modification"
                ]
            },
            "security_awareness_training": {
                "status": "required",
                "components": [
                    "Security reminders",
                    "Protection from malicious software",
                    "Log-in monitoring",
                    "Password management"
                ]
            },
            "security_incident_procedures": {
                "status": "required",
                "components": [
                    "Response and reporting"
                ]
            },
            "contingency_plan": {
                "status": "required",
                "components": [
                    "Data backup plan",
                    "Disaster recovery plan",
                    "Emergency mode operation plan",
                    "Testing and revision procedures",
                    "Applications and data criticality analysis"
                ]
            },
            "evaluation": {
                "status": "required",
                "description": "Периодическая оценка соответствия"
            },
            "business_associate_contracts": {
                "status": "required",
                "description": "Контракты с Business Associates"
            }
        }


class PhysicalSafeguards:
    """
    Физические защитные меры
    """

    def get_requirements(self) -> Dict[str, dict]:
        return {
            "facility_access_controls": {
                "status": "required",
                "components": [
                    "Contingency operations",
                    "Facility security plan",
                    "Access control and validation",
                    "Maintenance records"
                ]
            },
            "workstation_use": {
                "status": "required",
                "description": "Политики использования рабочих станций"
            },
            "workstation_security": {
                "status": "required",
                "description": "Физическая защита рабочих станций"
            },
            "device_and_media_controls": {
                "status": "required",
                "components": [
                    "Disposal",
                    "Media re-use",
                    "Accountability",
                    "Data backup and storage"
                ]
            }
        }


class TechnicalSafeguards:
    """
    Технические защитные меры
    """

    def get_requirements(self) -> Dict[str, dict]:
        return {
            "access_control": {
                "status": "required",
                "components": [
                    {
                        "name": "Unique user identification",
                        "status": "required"
                    },
                    {
                        "name": "Emergency access procedure",
                        "status": "required"
                    },
                    {
                        "name": "Automatic logoff",
                        "status": "addressable"
                    },
                    {
                        "name": "Encryption and decryption",
                        "status": "addressable"
                    }
                ]
            },
            "audit_controls": {
                "status": "required",
                "description": "Механизмы для записи и анализа доступа к ePHI"
            },
            "integrity": {
                "status": "required",
                "components": [
                    {
                        "name": "Mechanism to authenticate ePHI",
                        "status": "addressable"
                    }
                ]
            },
            "person_or_entity_authentication": {
                "status": "required",
                "description": "Проверка подлинности лица/системы"
            },
            "transmission_security": {
                "status": "required",
                "components": [
                    {
                        "name": "Integrity controls",
                        "status": "addressable"
                    },
                    {
                        "name": "Encryption",
                        "status": "addressable"
                    }
                ]
            }
        }


# Примечание: "addressable" не означает "опциональный"
# Это означает, что организация должна:
# 1. Оценить, разумно ли внедрить меру
# 2. Если да — внедрить
# 3. Если нет — документировать почему и внедрить альтернативу
```

### Практическая реализация Security Rule

```python
from datetime import datetime
from typing import Optional
import hashlib
import secrets

class HIPAASecurityImplementation:
    """
    Практическая реализация требований HIPAA Security Rule
    """

    # Access Control
    class AccessControl:
        """Контроль доступа к ePHI"""

        def unique_user_identification(self, user_id: str) -> dict:
            """Требование: Уникальная идентификация пользователя"""
            return {
                "user_id": user_id,
                "created_at": datetime.now().isoformat(),
                "status": "active",
                "authentication_method": "password_and_mfa",
                "last_password_change": None,
                "password_expires_in_days": 90
            }

        def emergency_access_procedure(self,
                                         patient_id: str,
                                         emergency_type: str,
                                         requester_id: str) -> dict:
            """Процедура экстренного доступа"""
            emergency_record = {
                "emergency_id": secrets.token_hex(8),
                "patient_id": patient_id,
                "requester_id": requester_id,
                "emergency_type": emergency_type,
                "timestamp": datetime.now().isoformat(),
                "requires_post_access_review": True
            }

            # Логируем для аудита
            self._log_emergency_access(emergency_record)

            return {
                "access_granted": True,
                "emergency_id": emergency_record["emergency_id"],
                "note": "Post-access review required within 24 hours"
            }

        def automatic_logoff(self, session_id: str,
                              idle_minutes: int = 15) -> dict:
            """Автоматическое завершение сессии"""
            return {
                "session_id": session_id,
                "idle_timeout_minutes": idle_minutes,
                "warning_before_logoff_seconds": 60,
                "save_unsaved_work": True
            }

    # Audit Controls
    class AuditControls:
        """Аудит-контроли"""

        def log_phi_access(self,
                           user_id: str,
                           patient_id: str,
                           action: str,
                           phi_accessed: List[str]) -> dict:
            """Логирование доступа к ePHI"""
            log_entry = {
                "log_id": secrets.token_hex(16),
                "timestamp": datetime.now().isoformat(),
                "user_id": user_id,
                "patient_id": patient_id,
                "action": action,  # view, create, update, delete, print, export
                "phi_fields_accessed": phi_accessed,
                "source_ip": self._get_source_ip(),
                "user_agent": self._get_user_agent(),
                "success": True
            }

            # Записываем в неизменяемый лог
            self._write_audit_log(log_entry)

            return log_entry

        def generate_audit_report(self,
                                   patient_id: str,
                                   start_date: datetime,
                                   end_date: datetime) -> dict:
            """Генерация отчёта аудита для пациента"""
            return {
                "patient_id": patient_id,
                "period": {
                    "start": start_date.isoformat(),
                    "end": end_date.isoformat()
                },
                "access_events": self._get_access_events(
                    patient_id, start_date, end_date
                ),
                "generated_at": datetime.now().isoformat(),
                "generated_by": self._get_current_user()
            }

    # Transmission Security
    class TransmissionSecurity:
        """Безопасность передачи данных"""

        def get_encryption_requirements(self) -> dict:
            """Требования к шифрованию"""
            return {
                "in_transit": {
                    "protocol": "TLS 1.2 or higher",
                    "ciphers": [
                        "TLS_AES_256_GCM_SHA384",
                        "TLS_CHACHA20_POLY1305_SHA256"
                    ],
                    "certificate_validation": True
                },
                "at_rest": {
                    "algorithm": "AES-256",
                    "key_management": "HSM or equivalent",
                    "key_rotation": "annual"
                },
                "email": {
                    "requirement": "Encrypt if contains ePHI",
                    "methods": ["S/MIME", "PGP", "Portal-based"]
                }
            }

        def validate_transmission(self, destination: str,
                                    contains_phi: bool) -> dict:
            """Валидация передачи данных"""
            if not contains_phi:
                return {"validation_required": False}

            return {
                "validation_required": True,
                "checks": [
                    {"name": "TLS version", "minimum": "1.2"},
                    {"name": "Certificate validity", "required": True},
                    {"name": "Recipient authorization", "required": True}
                ]
            }
```

---

## Breach Notification Rule

### Определение и процедуры

```python
from datetime import datetime, timedelta
from enum import Enum
from typing import List, Optional
from dataclasses import dataclass

class BreachSeverity(Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"


@dataclass
class HIPAABreach:
    """Определение утечки по HIPAA"""
    breach_id: str
    discovery_date: datetime
    description: str
    phi_involved: List[str]
    individuals_affected: int
    unsecured_phi: bool  # Была ли PHI незащищённой (незашифрованной)


class BreachNotificationService:
    """
    Сервис уведомлений об утечках
    Breach Notification Rule требует уведомления в течение 60 дней
    """

    NOTIFICATION_DEADLINE_DAYS = 60
    HHS_IMMEDIATE_THRESHOLD = 500  # Утечки 500+ лиц — немедленное уведомление HHS

    def assess_breach(self, incident: dict) -> dict:
        """
        Оценка инцидента: является ли он утечкой по HIPAA

        4 фактора для оценки риска:
        1. Тип PHI
        2. Кто получил несанкционированный доступ
        3. Была ли PHI действительно получена/просмотрена
        4. Степень снижения риска
        """
        risk_assessment = {
            "phi_nature_and_extent": self._assess_phi_sensitivity(incident),
            "unauthorized_recipient": self._assess_recipient(incident),
            "phi_actually_acquired": self._assess_acquisition(incident),
            "risk_mitigation": self._assess_mitigation(incident)
        }

        # Если низкая вероятность компрометации — не утечка
        if self._low_probability_compromise(risk_assessment):
            return {
                "is_breach": False,
                "reason": "Low probability that PHI was compromised",
                "documentation_required": True
            }

        return {
            "is_breach": True,
            "risk_assessment": risk_assessment,
            "notification_required": True
        }

    async def process_breach(self, breach: HIPAABreach) -> dict:
        """
        Обработка подтверждённой утечки
        """
        notification_deadline = (
            breach.discovery_date + timedelta(days=self.NOTIFICATION_DEADLINE_DAYS)
        )

        result = {
            "breach_id": breach.breach_id,
            "notification_deadline": notification_deadline.isoformat(),
            "notifications": []
        }

        # 1. Уведомление пострадавших лиц
        individual_notification = await self._notify_individuals(breach)
        result["notifications"].append(individual_notification)

        # 2. Уведомление HHS
        if breach.individuals_affected >= self.HHS_IMMEDIATE_THRESHOLD:
            # Немедленное уведомление для крупных утечек
            hhs_notification = await self._notify_hhs_immediately(breach)
        else:
            # Ежегодный отчёт для мелких утечек
            hhs_notification = await self._log_for_annual_report(breach)
        result["notifications"].append(hhs_notification)

        # 3. Уведомление СМИ (если 500+ лиц в одном штате)
        if breach.individuals_affected >= 500:
            media_notification = await self._notify_media(breach)
            result["notifications"].append(media_notification)

        return result

    async def _notify_individuals(self, breach: HIPAABreach) -> dict:
        """
        Уведомление пострадавших лиц
        """
        notification_content = {
            "description": breach.description,
            "date_of_breach": breach.discovery_date.isoformat(),
            "phi_involved": self._describe_phi(breach.phi_involved),
            "steps_taken": await self._get_remediation_steps(breach),
            "steps_individuals_should_take": self._get_individual_steps(breach),
            "contact_information": await self._get_contact_info()
        }

        return {
            "type": "individual_notification",
            "method": "first_class_mail",  # Требуется письмо
            "alternative_methods": [
                "email_if_prior_consent",
                "phone_for_urgent",
                "substitute_notice_if_outdated_contact"
            ],
            "content": notification_content,
            "deadline": (
                breach.discovery_date + timedelta(days=self.NOTIFICATION_DEADLINE_DAYS)
            ).isoformat()
        }

    def _get_individual_steps(self, breach: HIPAABreach) -> List[str]:
        """Рекомендуемые действия для пострадавших"""
        steps = [
            "Мониторинг своих медицинских записей на предмет ошибок",
            "Проверка explanation of benefits (EOB) от страховой"
        ]

        if "ssn" in breach.phi_involved:
            steps.append("Размещение fraud alert на кредитных отчётах")
            steps.append("Рассмотрение credit freeze")

        if "financial" in breach.phi_involved:
            steps.append("Мониторинг финансовых счетов")

        return steps
```

---

## Business Associate Agreement (BAA)

### Требования к контракту

```python
from dataclasses import dataclass
from typing import List, Dict
from datetime import datetime

@dataclass
class BusinessAssociateAgreement:
    """
    Business Associate Agreement (BAA)
    Обязательный контракт между Covered Entity и Business Associate
    """
    covered_entity: str
    business_associate: str
    effective_date: datetime
    phi_permitted_uses: List[str]
    phi_permitted_disclosures: List[str]


class BAARequirements:
    """
    Обязательные элементы BAA
    """

    def get_required_provisions(self) -> Dict[str, str]:
        """
        Обязательные положения BAA
        """
        return {
            "permitted_uses": """
                Описание разрешённых способов использования PHI
                Business Associate может использовать PHI только:
                - Как указано в контракте
                - По требованию закона
                - Для собственного управления/администрирования
            """,

            "safeguards": """
                BA обязуется использовать соответствующие защитные меры
                для предотвращения несанкционированного использования/раскрытия
            """,

            "reporting": """
                BA обязуется сообщать о:
                - Любом использовании/раскрытии не предусмотренном контрактом
                - Утечках unsecured PHI
                - Инцидентах безопасности
            """,

            "subcontractors": """
                BA обязуется заключать BAA со своими субподрядчиками,
                которые получают доступ к PHI
            """,

            "patient_rights": """
                BA обязуется обеспечивать реализацию прав пациентов:
                - Доступ к PHI
                - Внесение поправок
                - Учёт раскрытий
            """,

            "hhs_access": """
                BA обязуется предоставлять доступ к документации
                по запросу HHS для проверки соответствия
            """,

            "return_or_destroy": """
                По окончании контракта BA обязуется:
                - Вернуть или уничтожить всю PHI
                - Или сохранить с теми же обязательствами защиты
            """,

            "termination": """
                CE может расторгнуть контракт при нарушении BA
            """
        }

    def get_liability_provisions(self) -> Dict[str, str]:
        """
        Положения об ответственности
        """
        return {
            "breach_liability": """
                BA несёт ответственность за утечки, вызванные его действиями
            """,

            "subcontractor_liability": """
                BA несёт ответственность за своих субподрядчиков
            """,

            "indemnification": """
                Типичные положения о возмещении убытков
            """,

            "insurance": """
                Требования к страхованию кибер-рисков
            """
        }


class BAASampleClauses:
    """
    Примеры формулировок для BAA
    """

    @staticmethod
    def get_security_clause() -> str:
        return """
SECURITY SAFEGUARDS

Business Associate shall:

1. Implement administrative, physical, and technical safeguards that
   reasonably and appropriately protect the confidentiality, integrity,
   and availability of the electronic protected health information
   that it creates, receives, maintains, or transmits on behalf of
   Covered Entity.

2. Ensure that any agent, including a subcontractor, to whom it
   provides such information agrees to implement reasonable and
   appropriate safeguards.

3. Report to Covered Entity any Security Incident of which it
   becomes aware, including breaches of unsecured protected health
   information, as required by 45 CFR § 164.410.
"""

    @staticmethod
    def get_breach_notification_clause() -> str:
        return """
BREACH NOTIFICATION

1. Discovery of Breach. Business Associate shall notify Covered Entity
   of any Breach of Unsecured PHI without unreasonable delay and in
   no event later than thirty (30) calendar days after discovery.

2. Contents of Notice. Such notice shall include:
   - Identification of each individual affected
   - Description of the PHI involved
   - Date of the Breach and date of discovery
   - Description of what Business Associate is doing to mitigate harm
"""
```

---

## Практические рекомендации для API

### 1. Архитектура HIPAA-совместимого API

```python
from fastapi import FastAPI, Depends, HTTPException, Request
from fastapi.security import OAuth2PasswordBearer
from pydantic import BaseModel
from typing import Optional, List
import logging

# Настройка безопасного логирования (без PHI)
class HIPAALogger:
    """Логгер, не записывающий PHI"""

    REDACT_FIELDS = [
        "patient_name", "ssn", "dob", "address", "phone",
        "email", "mrn", "diagnosis", "medications"
    ]

    def log_access(self, user_id: str, resource: str, action: str):
        """Логирование доступа без PHI"""
        logging.info(f"ACCESS: user={user_id} resource={resource} action={action}")

    def sanitize_for_logging(self, data: dict) -> dict:
        """Удаление PHI перед логированием"""
        sanitized = {}
        for key, value in data.items():
            if key.lower() in self.REDACT_FIELDS:
                sanitized[key] = "[REDACTED]"
            else:
                sanitized[key] = value
        return sanitized


app = FastAPI(title="HIPAA Compliant Healthcare API")


# Модели данных
class PatientData(BaseModel):
    """Модель данных пациента"""
    patient_id: str
    # PHI fields - требуют шифрования при хранении
    name: Optional[str] = None
    dob: Optional[str] = None
    ssn: Optional[str] = None
    medical_record_number: Optional[str] = None


class AuditLogEntry(BaseModel):
    """Запись аудит-лога"""
    user_id: str
    patient_id: str
    action: str
    timestamp: str
    phi_accessed: List[str]


# Middleware для аудита
class HIPAAAuditMiddleware:
    """Middleware для аудита всех запросов к PHI"""

    async def __call__(self, request: Request, call_next):
        # Логируем входящий запрос
        user_id = await self._get_user_id(request)
        resource = request.url.path

        # Выполняем запрос
        response = await call_next(request)

        # Логируем результат (без PHI)
        await self._log_access(user_id, resource, request.method, response.status_code)

        return response


# Сервис авторизации
class HIPAAAuthorizationService:
    """Сервис авторизации доступа к PHI"""

    async def check_access(self,
                            user_id: str,
                            patient_id: str,
                            purpose: str) -> bool:
        """
        Проверка права доступа к PHI пациента
        """
        # Проверяем:
        # 1. Есть ли у пользователя роль для доступа
        # 2. Есть ли связь с пациентом (treating physician, etc.)
        # 3. Есть ли согласие пациента (если требуется)
        # 4. Применяется ли break-the-glass для экстренных случаев

        user_roles = await self._get_user_roles(user_id)
        patient_relationships = await self._get_patient_relationships(
            user_id, patient_id
        )

        # Проверка Treatment relationship
        if purpose == "treatment":
            return any(r in ["treating_physician", "nurse", "care_team"]
                      for r in patient_relationships)

        # Проверка Payment
        if purpose == "payment":
            return "billing_staff" in user_roles

        # Проверка Operations
        if purpose == "operations":
            return "administrator" in user_roles

        return False


# Эндпоинты
@app.get("/api/patients/{patient_id}")
async def get_patient(
    patient_id: str,
    purpose: str,
    current_user: str = Depends(get_current_user),
    auth_service: HIPAAAuthorizationService = Depends()
):
    """
    Получение данных пациента

    - Требует аутентификации
    - Проверяет авторизацию
    - Применяет minimum necessary
    - Логирует доступ
    """
    # Проверка авторизации
    if not await auth_service.check_access(current_user, patient_id, purpose):
        raise HTTPException(
            status_code=403,
            detail="Access denied - no treatment relationship"
        )

    # Получаем данные
    patient_data = await get_patient_data(patient_id)

    # Применяем minimum necessary
    filtered_data = apply_minimum_necessary(patient_data, purpose)

    # Логируем доступ
    await log_phi_access(current_user, patient_id, "read", list(filtered_data.keys()))

    return filtered_data


@app.post("/api/patients/{patient_id}/emergency-access")
async def emergency_access(
    patient_id: str,
    emergency_type: str,
    current_user: str = Depends(get_current_user)
):
    """
    Экстренный доступ (Break-the-glass)

    - Предоставляет доступ без обычной авторизации
    - Требует обоснования
    - Логируется отдельно
    - Требует последующего review
    """
    emergency_record = {
        "user_id": current_user,
        "patient_id": patient_id,
        "emergency_type": emergency_type,
        "timestamp": datetime.now().isoformat(),
        "reviewed": False
    }

    await log_emergency_access(emergency_record)

    # Получаем данные
    patient_data = await get_patient_data(patient_id)

    # Уведомляем Privacy Officer
    await notify_privacy_officer(emergency_record)

    return {
        "data": patient_data,
        "emergency_id": emergency_record["emergency_id"],
        "warning": "This access will be reviewed within 24 hours"
    }
```

### 2. Шифрование ePHI

```python
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
from cryptography.hazmat.backends import default_backend
import base64
import os

class ePHIEncryption:
    """
    Шифрование ePHI согласно HIPAA
    """

    def __init__(self, master_key: bytes):
        self.master_key = master_key

    def encrypt_phi(self, phi_data: str) -> bytes:
        """
        Шифрование PHI
        """
        # Генерируем соль для каждого шифрования
        salt = os.urandom(16)

        # Деривация ключа
        kdf = PBKDF2HMAC(
            algorithm=hashes.SHA256(),
            length=32,
            salt=salt,
            iterations=100000,
            backend=default_backend()
        )
        key = base64.urlsafe_b64encode(kdf.derive(self.master_key))

        # Шифрование
        f = Fernet(key)
        encrypted = f.encrypt(phi_data.encode())

        # Возвращаем соль + зашифрованные данные
        return salt + encrypted

    def decrypt_phi(self, encrypted_data: bytes) -> str:
        """
        Расшифровка PHI
        """
        # Извлекаем соль
        salt = encrypted_data[:16]
        ciphertext = encrypted_data[16:]

        # Деривация ключа
        kdf = PBKDF2HMAC(
            algorithm=hashes.SHA256(),
            length=32,
            salt=salt,
            iterations=100000,
            backend=default_backend()
        )
        key = base64.urlsafe_b64encode(kdf.derive(self.master_key))

        # Расшифровка
        f = Fernet(key)
        return f.decrypt(ciphertext).decode()


class EncryptedPHIStorage:
    """Хранилище зашифрованной PHI"""

    def __init__(self, encryption_service: ePHIEncryption):
        self.encryption = encryption_service

    async def store_patient_record(self,
                                     patient_id: str,
                                     record: dict) -> str:
        """Сохранение зашифрованной записи"""
        import json

        # Шифруем PHI поля
        phi_fields = ["name", "ssn", "dob", "address", "phone", "diagnosis"]

        encrypted_record = {}
        for key, value in record.items():
            if key in phi_fields and value:
                encrypted_record[key] = self.encryption.encrypt_phi(str(value))
            else:
                encrypted_record[key] = value

        # Сохраняем в БД
        record_id = await self._save_to_db(patient_id, encrypted_record)
        return record_id

    async def retrieve_patient_record(self, patient_id: str) -> dict:
        """Получение и расшифровка записи"""
        encrypted_record = await self._get_from_db(patient_id)

        phi_fields = ["name", "ssn", "dob", "address", "phone", "diagnosis"]

        decrypted_record = {}
        for key, value in encrypted_record.items():
            if key in phi_fields and value:
                decrypted_record[key] = self.encryption.decrypt_phi(value)
            else:
                decrypted_record[key] = value

        return decrypted_record
```

---

## Штрафы и последствия несоблюдения

### Структура штрафов (HITECH Act)

| Уровень нарушения | Штраф за нарушение | Максимум в год |
|-------------------|-------------------|----------------|
| **Tier 1**: Не знал и не мог знать | $100 - $50,000 | $25,000 |
| **Tier 2**: Разумные основания знать | $1,000 - $50,000 | $100,000 |
| **Tier 3**: Умышленное пренебрежение (исправлено) | $10,000 - $50,000 | $250,000 |
| **Tier 4**: Умышленное пренебрежение (не исправлено) | $50,000+ | $1,500,000 |

### Уголовная ответственность

| Нарушение | Наказание |
|-----------|-----------|
| Незаконное получение/раскрытие PHI | До $50,000 + 1 год тюрьмы |
| Получение под ложным предлогом | До $100,000 + 5 лет тюрьмы |
| С намерением продать/использовать | До $250,000 + 10 лет тюрьмы |

### Крупнейшие штрафы HIPAA

| Год | Организация | Штраф | Причина |
|-----|-------------|-------|---------|
| 2023 | Banner Health | $1.25 млн | Утечка 2.81 млн записей |
| 2022 | Advocate Aurora | $5.55 млн | Отсутствие BAA, утечка |
| 2021 | Excellus Health | $5.1 млн | Утечка 9.3 млн записей |
| 2018 | Anthem | $16 млн | Крупнейшая утечка (78.8 млн) |

---

## Чеклист соответствия HIPAA

### Административные требования

- [ ] Назначен Privacy Officer
- [ ] Назначен Security Officer
- [ ] Проведён Risk Analysis
- [ ] Создан Risk Management Plan
- [ ] Разработаны политики и процедуры
- [ ] Проведено обучение персонала
- [ ] Установлены санкции за нарушения
- [ ] Создан план реагирования на инциденты
- [ ] Заключены BAA со всеми Business Associates

### Технические требования

- [ ] Уникальная идентификация пользователей
- [ ] Контроль доступа на основе ролей
- [ ] Шифрование ePHI при хранении
- [ ] Шифрование ePHI при передаче (TLS)
- [ ] Audit logging всех доступов к PHI
- [ ] Автоматическое завершение сессий
- [ ] Процедура экстренного доступа
- [ ] Резервное копирование данных
- [ ] Disaster recovery plan

### Чеклист для API

```python
class HIPAAAPIComplianceChecker:
    """Проверка соответствия API требованиям HIPAA"""

    def check_compliance(self, api_spec: dict) -> dict:
        return {
            "access_control": {
                "authentication": self._has_authentication(api_spec),
                "authorization": self._has_role_based_access(api_spec),
                "unique_user_ids": self._has_unique_ids(api_spec),
                "session_management": self._has_session_timeout(api_spec)
            },
            "audit_controls": {
                "access_logging": self._logs_all_access(api_spec),
                "audit_trail": self._has_audit_trail(api_spec),
                "tamper_proof_logs": self._has_immutable_logs(api_spec)
            },
            "transmission_security": {
                "encryption_in_transit": self._has_tls(api_spec),
                "tls_version": self._tls_version(api_spec)
            },
            "data_integrity": {
                "input_validation": self._has_input_validation(api_spec),
                "integrity_verification": self._has_integrity_checks(api_spec)
            },
            "phi_handling": {
                "minimum_necessary": self._enforces_minimum_necessary(api_spec),
                "phi_not_in_urls": self._no_phi_in_urls(api_spec),
                "phi_not_in_logs": self._no_phi_in_logs(api_spec)
            }
        }
```

---

## Полезные ресурсы

- [HHS HIPAA Homepage](https://www.hhs.gov/hipaa/)
- [HIPAA Security Rule Guidance](https://www.hhs.gov/hipaa/for-professionals/security/guidance/)
- [OCR Breach Portal](https://ocrportal.hhs.gov/ocr/breach/breach_report.jsf)
- [NIST HIPAA Security Rule Toolkit](https://csrc.nist.gov/projects/security-content-automation-protocol/hipaa)
- [HHS Sample BAA Provisions](https://www.hhs.gov/hipaa/for-professionals/covered-entities/sample-business-associate-agreement-provisions/)

---

[prev: 03-pci-dss](./03-pci-dss.md) | [next: 05-pii](./05-pii.md)
