# PCI DSS (Payment Card Industry Data Security Standard)

[prev: 02-ccpa](./02-ccpa.md) | [next: 04-hipaa](./04-hipaa.md)

---

## Введение

**PCI DSS** (Payment Card Industry Data Security Standard) — это глобальный стандарт безопасности данных платёжных карт, разработанный ведущими платёжными системами мира. Стандарт устанавливает требования к защите данных держателей карт при их хранении, обработке и передаче.

PCI DSS обязателен для всех организаций, которые принимают, обрабатывают, хранят или передают данные платёжных карт, независимо от их размера или количества транзакций.

---

## История и развитие

### Хронология

| Год | Версия | Ключевые изменения |
|-----|--------|-------------------|
| 2004 | 1.0 | Первая версия стандарта |
| 2006 | 1.1 | Создание PCI SSC (Security Standards Council) |
| 2010 | 2.0 | Усиление требований к тестированию |
| 2013 | 3.0 | Фокус на гибкости и безопасности |
| 2016 | 3.2 | Многофакторная аутентификация, SSL/TLS |
| 2018 | 3.2.1 | Уточнения и обновления |
| 2022 | 4.0 | Современный подход, кастомизированные контроли |
| 2024 | 4.0.1 | Актуальная версия |

### PCI Security Standards Council (PCI SSC)

Основан в 2006 году пятью крупнейшими платёжными системами:

- **Visa**
- **Mastercard**
- **American Express**
- **Discover**
- **JCB International**

---

## Область применения

### Кто должен соответствовать PCI DSS

PCI DSS применяется ко всем организациям, участвующим в обработке платежей:

```python
from enum import Enum
from dataclasses import dataclass
from typing import List, Optional

class EntityType(Enum):
    """Типы организаций в экосистеме платежей"""
    MERCHANT = "merchant"  # Торговец (принимает карты)
    SERVICE_PROVIDER = "service_provider"  # Сервис-провайдер
    ISSUER = "issuer"  # Эмитент карт (банк)
    ACQUIRER = "acquirer"  # Эквайер (банк торговца)
    PROCESSOR = "processor"  # Процессинговый центр


class MerchantLevel(Enum):
    """
    Уровни торговцев по количеству транзакций в год
    (могут различаться между платёжными системами)
    """
    LEVEL_1 = "level_1"  # > 6 млн транзакций
    LEVEL_2 = "level_2"  # 1-6 млн транзакций
    LEVEL_3 = "level_3"  # 20,000 - 1 млн e-commerce транзакций
    LEVEL_4 = "level_4"  # < 20,000 e-commerce или < 1 млн других


@dataclass
class PCIDSSApplicability:
    """Определение применимости PCI DSS"""
    entity_type: EntityType
    annual_transactions: int
    stores_card_data: bool
    processes_card_data: bool
    transmits_card_data: bool


def determine_merchant_level(transactions: int, is_ecommerce: bool) -> MerchantLevel:
    """
    Определение уровня торговца по Visa
    """
    if transactions > 6_000_000:
        return MerchantLevel.LEVEL_1
    elif transactions > 1_000_000:
        return MerchantLevel.LEVEL_2
    elif is_ecommerce and transactions > 20_000:
        return MerchantLevel.LEVEL_3
    else:
        return MerchantLevel.LEVEL_4


def get_validation_requirements(level: MerchantLevel) -> dict:
    """
    Требования к валидации соответствия по уровню торговца
    """
    requirements = {
        MerchantLevel.LEVEL_1: {
            "annual_onsite_assessment": True,  # QSA аудит
            "quarterly_network_scan": True,  # ASV сканирование
            "penetration_test": "annual",
            "aoc_required": True  # Attestation of Compliance
        },
        MerchantLevel.LEVEL_2: {
            "annual_saq": True,  # Self-Assessment Questionnaire
            "quarterly_network_scan": True,
            "penetration_test": "annual",
            "aoc_required": True
        },
        MerchantLevel.LEVEL_3: {
            "annual_saq": True,
            "quarterly_network_scan": True,
            "penetration_test": "recommended",
            "aoc_required": True
        },
        MerchantLevel.LEVEL_4: {
            "annual_saq": True,
            "quarterly_network_scan": "if_applicable",
            "penetration_test": "recommended",
            "aoc_required": "recommended"
        }
    }
    return requirements.get(level, {})
```

### Данные держателей карт (CHD)

```python
from dataclasses import dataclass
from enum import Enum

class DataSensitivity(Enum):
    """Уровни чувствительности данных"""
    CARDHOLDER_DATA = "cardholder_data"  # Данные держателя карты
    SENSITIVE_AUTH_DATA = "sensitive_auth_data"  # Чувствительные данные аутентификации


@dataclass
class CardholderData:
    """
    Данные держателя карты (CHD)
    Могут храниться при условии защиты
    """
    primary_account_number: str  # PAN - номер карты (ОБЯЗАТЕЛЬНО защищать)
    cardholder_name: Optional[str] = None  # Имя держателя
    expiration_date: Optional[str] = None  # Срок действия
    service_code: Optional[str] = None  # Сервисный код


@dataclass
class SensitiveAuthenticationData:
    """
    Чувствительные данные аутентификации (SAD)
    ЗАПРЕЩЕНО хранить после авторизации!
    """
    full_track_data: Optional[str] = None  # Данные магнитной полосы
    cvv_cvc: Optional[str] = None  # CAV2/CVC2/CVV2/CID
    pin_block: Optional[str] = None  # PIN / PIN block


# Правила хранения данных
STORAGE_RULES = {
    "pan": {
        "can_store": True,
        "must_protect": True,
        "methods": ["encryption", "truncation", "masking", "hashing"]
    },
    "cardholder_name": {
        "can_store": True,
        "must_protect": True,
        "methods": ["encryption"]
    },
    "expiration_date": {
        "can_store": True,
        "must_protect": True,
        "methods": ["encryption"]
    },
    "service_code": {
        "can_store": True,
        "must_protect": True,
        "methods": ["encryption"]
    },
    "full_track_data": {
        "can_store": False,  # ЗАПРЕЩЕНО после авторизации
        "reason": "Never store after authorization"
    },
    "cvv_cvc": {
        "can_store": False,  # ЗАПРЕЩЕНО после авторизации
        "reason": "Never store after authorization"
    },
    "pin_block": {
        "can_store": False,  # ЗАПРЕЩЕНО после авторизации
        "reason": "Never store after authorization"
    }
}
```

---

## 12 требований PCI DSS

PCI DSS организован вокруг 6 целей и 12 требований:

### Цель 1: Построение и поддержание защищённой сети

#### Требование 1: Установка и поддержание конфигурации файрвола

```python
from dataclasses import dataclass
from typing import List, Dict
from enum import Enum

class FirewallRuleAction(Enum):
    ALLOW = "allow"
    DENY = "deny"
    LOG = "log"


@dataclass
class FirewallRule:
    """Правило файрвола для защиты CDE"""
    name: str
    source: str
    destination: str
    port: str
    protocol: str
    action: FirewallRuleAction
    justification: str  # Обоснование правила (требование PCI DSS)


class PCIDSSFirewallConfiguration:
    """
    Конфигурация файрвола согласно PCI DSS
    """

    def __init__(self):
        self.rules: List[FirewallRule] = []
        self.cde_network = "10.0.1.0/24"  # Cardholder Data Environment

    def get_required_rules(self) -> List[FirewallRule]:
        """
        Обязательные правила для CDE
        """
        return [
            FirewallRule(
                name="Deny all inbound by default",
                source="any",
                destination=self.cde_network,
                port="any",
                protocol="any",
                action=FirewallRuleAction.DENY,
                justification="Default deny all traffic to CDE"
            ),
            FirewallRule(
                name="Allow HTTPS to payment gateway",
                source="web_servers",
                destination="payment_processor",
                port="443",
                protocol="tcp",
                action=FirewallRuleAction.ALLOW,
                justification="Required for payment processing"
            ),
            FirewallRule(
                name="Block direct internet to CDE",
                source="internet",
                destination=self.cde_network,
                port="any",
                protocol="any",
                action=FirewallRuleAction.DENY,
                justification="CDE must not be directly accessible from internet"
            )
        ]

    def validate_configuration(self) -> Dict[str, bool]:
        """Проверка соответствия конфигурации PCI DSS"""
        return {
            "default_deny_configured": self._has_default_deny(),
            "cde_segmented": self._is_cde_segmented(),
            "no_direct_internet_access": self._no_direct_internet(),
            "rules_documented": self._all_rules_documented(),
            "review_schedule_set": self._has_review_schedule()
        }


# Пример конфигурации сегментации сети
NETWORK_SEGMENTATION = """
# PCI DSS Network Segmentation Example

## Сегменты сети:

1. **CDE (Cardholder Data Environment)** - 10.0.1.0/24
   - Серверы обработки платежей
   - Базы данных с данными карт
   - Изолирован от остальной сети

2. **DMZ** - 10.0.2.0/24
   - Web-серверы
   - API Gateway
   - WAF (Web Application Firewall)

3. **Corporate Network** - 10.0.3.0/24
   - Рабочие станции
   - Офисные системы
   - НЕ имеет доступа к CDE

4. **Management Network** - 10.0.4.0/24
   - Системы администрирования
   - Logging & Monitoring
   - Доступ к CDE только через bastion host
"""
```

#### Требование 2: Безопасная конфигурация систем

```python
class SystemHardeningChecklist:
    """
    Чеклист hardening систем для PCI DSS
    """

    VENDOR_DEFAULTS_TO_CHANGE = [
        "default_passwords",
        "default_accounts",
        "unnecessary_services",
        "unnecessary_protocols",
        "unnecessary_ports",
        "sample_applications"
    ]

    def get_hardening_requirements(self, system_type: str) -> Dict[str, List[str]]:
        """Требования к hardening по типу системы"""

        base_requirements = [
            "Изменить все пароли по умолчанию",
            "Отключить неиспользуемые сервисы",
            "Удалить учётные записи по умолчанию",
            "Настроить безопасные протоколы (TLS 1.2+)",
            "Включить логирование безопасности",
            "Настроить NTP синхронизацию"
        ]

        specific = {
            "linux_server": [
                *base_requirements,
                "Настроить SELinux/AppArmor",
                "Отключить root SSH",
                "Настроить файрвол (iptables/nftables)",
                "Настроить umask 027"
            ],
            "windows_server": [
                *base_requirements,
                "Включить Windows Firewall",
                "Настроить Group Policy",
                "Отключить SMBv1",
                "Включить BitLocker"
            ],
            "database": [
                *base_requirements,
                "Отключить удалённый root доступ",
                "Включить шифрование соединений",
                "Настроить минимальные привилегии",
                "Включить аудит запросов"
            ],
            "web_server": [
                *base_requirements,
                "Удалить дефолтные страницы",
                "Отключить directory listing",
                "Настроить Security Headers",
                "Включить HTTPS only"
            ]
        }

        return {"requirements": specific.get(system_type, base_requirements)}
```

### Цель 2: Защита данных держателей карт

#### Требование 3: Защита хранимых данных

```python
import hashlib
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend
import os

class PANProtection:
    """
    Защита номера карты (PAN) согласно PCI DSS
    """

    def __init__(self, encryption_key: bytes):
        self.key = encryption_key

    def mask_pan(self, pan: str) -> str:
        """
        Маскирование PAN
        Можно показывать только первые 6 и последние 4 цифры
        """
        if len(pan) < 13:
            raise ValueError("Invalid PAN length")

        first_six = pan[:6]
        last_four = pan[-4:]
        masked = first_six + "*" * (len(pan) - 10) + last_four

        return masked  # 411111******1111

    def truncate_pan(self, pan: str) -> str:
        """
        Усечение PAN
        Хранить можно максимум первые 6 И последние 4
        (но не оба одновременно если позволяет восстановить PAN)
        """
        # Для большинства случаев - только последние 4
        return pan[-4:]

    def hash_pan(self, pan: str, salt: str) -> str:
        """
        Хэширование PAN (односторонняя криптография)
        Требует strong cryptography + salt
        """
        salted = f"{salt}{pan}".encode()
        return hashlib.sha256(salted).hexdigest()

    def encrypt_pan(self, pan: str) -> bytes:
        """
        Шифрование PAN (AES-256)
        Ключи должны храниться отдельно от данных
        """
        iv = os.urandom(16)
        cipher = Cipher(
            algorithms.AES(self.key),
            modes.GCM(iv),
            backend=default_backend()
        )
        encryptor = cipher.encryptor()
        encrypted = encryptor.update(pan.encode()) + encryptor.finalize()

        return iv + encryptor.tag + encrypted

    def decrypt_pan(self, encrypted_data: bytes) -> str:
        """Расшифровка PAN"""
        iv = encrypted_data[:16]
        tag = encrypted_data[16:32]
        ciphertext = encrypted_data[32:]

        cipher = Cipher(
            algorithms.AES(self.key),
            modes.GCM(iv, tag),
            backend=default_backend()
        )
        decryptor = cipher.decryptor()
        return (decryptor.update(ciphertext) + decryptor.finalize()).decode()


class KeyManagement:
    """
    Управление криптографическими ключами
    Требование 3.5-3.7 PCI DSS
    """

    def __init__(self):
        self.key_encryption_key = None  # KEK
        self.data_encryption_keys = {}  # DEKs

    def generate_key(self, purpose: str) -> bytes:
        """Генерация криптографического ключа"""
        key = os.urandom(32)  # 256 бит

        # Ключ должен быть защищён
        protected_key = self._protect_key(key)
        self.data_encryption_keys[purpose] = protected_key

        return key

    def _protect_key(self, key: bytes) -> bytes:
        """
        Защита ключа шифрования данных (DEK)
        с помощью ключа шифрования ключей (KEK)
        """
        if not self.key_encryption_key:
            raise ValueError("KEK not initialized")

        f = Fernet(self.key_encryption_key)
        return f.encrypt(key)

    def rotate_key(self, purpose: str) -> bytes:
        """
        Ротация ключа
        Требуется минимум ежегодно или при компрометации
        """
        # 1. Генерируем новый ключ
        new_key = self.generate_key(f"{purpose}_new")

        # 2. Перешифровываем данные новым ключом
        # (должно быть реализовано в зависимости от системы)

        # 3. Удаляем старый ключ
        old_key = self.data_encryption_keys.pop(purpose, None)

        # 4. Переименовываем новый ключ
        self.data_encryption_keys[purpose] = self.data_encryption_keys.pop(f"{purpose}_new")

        return new_key

    def get_key_policy(self) -> dict:
        """Политика управления ключами"""
        return {
            "key_generation": {
                "algorithm": "AES-256",
                "source": "cryptographically_secure_rng"
            },
            "key_storage": {
                "location": "HSM или защищённое хранилище",
                "access": "split_knowledge_dual_control",
                "backup": "encrypted_in_separate_location"
            },
            "key_rotation": {
                "frequency": "annual_minimum",
                "on_compromise": "immediate",
                "on_personnel_change": "when_key_custodian_leaves"
            },
            "key_destruction": {
                "method": "cryptographic_erasure",
                "documentation": "required"
            }
        }
```

#### Требование 4: Шифрование при передаче

```python
from dataclasses import dataclass
from typing import List

@dataclass
class TLSConfiguration:
    """
    Конфигурация TLS для PCI DSS
    """
    min_version: str = "TLSv1.2"
    preferred_version: str = "TLSv1.3"

    # Разрешённые cipher suites (PCI DSS 4.0)
    allowed_ciphers: List[str] = None

    def __post_init__(self):
        if self.allowed_ciphers is None:
            self.allowed_ciphers = [
                # TLS 1.3
                "TLS_AES_256_GCM_SHA384",
                "TLS_CHACHA20_POLY1305_SHA256",
                "TLS_AES_128_GCM_SHA256",
                # TLS 1.2
                "ECDHE-ECDSA-AES256-GCM-SHA384",
                "ECDHE-RSA-AES256-GCM-SHA384",
                "ECDHE-ECDSA-AES128-GCM-SHA256",
                "ECDHE-RSA-AES128-GCM-SHA256"
            ]

    def get_nginx_config(self) -> str:
        """Пример конфигурации Nginx для PCI DSS"""
        return f"""
# PCI DSS Compliant TLS Configuration

ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers on;
ssl_ciphers '{':'.join(self.allowed_ciphers)}';

# HSTS (минимум 1 год)
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

# Отключаем слабые алгоритмы
ssl_ecdh_curve secp384r1;
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:50m;
ssl_session_tickets off;

# OCSP Stapling
ssl_stapling on;
ssl_stapling_verify on;
"""


class TransmissionSecurity:
    """Безопасность передачи данных"""

    PROHIBITED_CHANNELS = [
        "unencrypted_email",
        "ftp",
        "http",
        "telnet",
        "instant_messaging_unencrypted"
    ]

    ALLOWED_CHANNELS = [
        "https",
        "sftp",
        "encrypted_email_pgp_smime",
        "vpn_with_strong_encryption",
        "tls_1_2_or_higher"
    ]

    def validate_transmission(self, channel: str, destination: str) -> dict:
        """Валидация канала передачи"""
        if channel in self.PROHIBITED_CHANNELS:
            return {
                "allowed": False,
                "reason": f"Channel {channel} is not allowed for CHD transmission",
                "recommendation": "Use HTTPS, SFTP, or VPN"
            }

        return {
            "allowed": True,
            "requirements": [
                "Verify TLS version >= 1.2",
                "Verify certificate validity",
                "Log transmission for audit"
            ]
        }
```

### Цель 3: Программа управления уязвимостями

#### Требование 5: Антивирусная защита

```python
class AntiMalwareRequirements:
    """Требования к защите от вредоносного ПО"""

    def get_requirements(self) -> dict:
        return {
            "deployment": {
                "all_systems_protected": True,
                "commonly_affected_systems": [
                    "windows_workstations",
                    "windows_servers",
                    "mac_workstations"
                ],
                "risk_evaluation_for_others": True  # Linux, Unix
            },
            "updates": {
                "automatic_updates": True,
                "signature_update_frequency": "at_least_daily",
                "engine_updates": "as_released"
            },
            "scanning": {
                "real_time_scanning": True,
                "periodic_scans": "at_least_daily",
                "on_access_scanning": True
            },
            "logging": {
                "audit_logs_enabled": True,
                "retention": "per_requirement_10",  # 1 год
                "centralized_logging": True
            },
            "user_controls": {
                "users_cannot_disable": True,
                "exceptions_require_approval": True,
                "exceptions_time_limited": True
            }
        }
```

#### Требование 6: Безопасная разработка

```python
from enum import Enum
from typing import List, Dict

class VulnerabilitySeverity(Enum):
    CRITICAL = "critical"
    HIGH = "high"
    MEDIUM = "medium"
    LOW = "low"


class SecureDevelopmentLifecycle:
    """
    Безопасный жизненный цикл разработки (SDLC)
    Требование 6 PCI DSS
    """

    # Сроки установки патчей
    PATCHING_TIMEFRAMES = {
        VulnerabilitySeverity.CRITICAL: 30,  # дней
        VulnerabilitySeverity.HIGH: 30,
        VulnerabilitySeverity.MEDIUM: 90,
        VulnerabilitySeverity.LOW: 180
    }

    def get_secure_coding_requirements(self) -> List[str]:
        """Требования к безопасному кодированию"""
        return [
            # OWASP Top 10 покрытие
            "Injection (SQL, OS, LDAP)",
            "Broken Authentication",
            "Sensitive Data Exposure",
            "XML External Entities (XXE)",
            "Broken Access Control",
            "Security Misconfiguration",
            "Cross-Site Scripting (XSS)",
            "Insecure Deserialization",
            "Using Components with Known Vulnerabilities",
            "Insufficient Logging & Monitoring"
        ]

    def get_code_review_requirements(self) -> dict:
        """Требования к review кода"""
        return {
            "manual_review": {
                "required_for": "all_custom_code",
                "reviewer": "qualified_individual",
                "focus_areas": [
                    "authentication_logic",
                    "authorization_checks",
                    "input_validation",
                    "error_handling",
                    "cryptographic_implementations"
                ]
            },
            "automated_tools": {
                "sast": True,  # Static Application Security Testing
                "dast": True,  # Dynamic Application Security Testing
                "sca": True,   # Software Composition Analysis
                "frequency": "before_each_release"
            },
            "separation_of_duties": {
                "developer_cannot_approve_own_code": True,
                "production_deployment_separate": True
            }
        }


class ChangeManagement:
    """
    Управление изменениями для CDE
    """

    def get_change_control_process(self) -> Dict[str, List[str]]:
        return {
            "documentation": [
                "Описание изменения",
                "Оценка влияния на безопасность",
                "Одобрение уполномоченных лиц",
                "Тестирование функциональности",
                "Процедуры отката"
            ],
            "testing": [
                "Функциональное тестирование",
                "Тестирование безопасности",
                "Регрессионное тестирование",
                "Тестирование в staging среде"
            ],
            "separation": [
                "Разделение dev/test/prod сред",
                "Отдельные учётные записи для prod",
                "Разные права доступа по средам"
            ],
            "rollback": [
                "Документированные процедуры отката",
                "Резервные копии перед изменениями",
                "Тестирование процедур отката"
            ]
        }
```

### Цель 4: Строгий контроль доступа

#### Требование 7: Ограничение доступа

```python
from dataclasses import dataclass
from typing import Set
from enum import Enum

class AccessLevel(Enum):
    """Уровни доступа к данным карт"""
    NO_ACCESS = 0
    READ_MASKED = 1  # Только маскированные данные
    READ_FULL = 2    # Полные данные
    WRITE = 3        # Запись
    ADMIN = 4        # Администрирование


@dataclass
class Role:
    """Роль в системе"""
    name: str
    access_level: AccessLevel
    allowed_actions: Set[str]
    requires_mfa: bool


class AccessControlSystem:
    """
    Система контроля доступа
    Need-to-know и Least Privilege
    """

    def __init__(self):
        self.roles: Dict[str, Role] = {}
        self._define_default_roles()

    def _define_default_roles(self):
        """Определение стандартных ролей"""
        self.roles = {
            "customer_service": Role(
                name="Customer Service",
                access_level=AccessLevel.READ_MASKED,
                allowed_actions={"view_masked_pan", "view_transaction_history"},
                requires_mfa=True
            ),
            "payment_processor": Role(
                name="Payment Processor",
                access_level=AccessLevel.READ_FULL,
                allowed_actions={"process_payment", "view_full_pan", "refund"},
                requires_mfa=True
            ),
            "security_admin": Role(
                name="Security Administrator",
                access_level=AccessLevel.ADMIN,
                allowed_actions={"manage_users", "view_logs", "configure_security"},
                requires_mfa=True
            ),
            "developer": Role(
                name="Developer",
                access_level=AccessLevel.NO_ACCESS,
                allowed_actions={"access_dev_environment"},
                requires_mfa=True
            )
        }

    def check_access(self, user_role: str, requested_action: str) -> dict:
        """Проверка права доступа"""
        role = self.roles.get(user_role)

        if not role:
            return {"allowed": False, "reason": "Unknown role"}

        if requested_action in role.allowed_actions:
            return {
                "allowed": True,
                "requires_mfa": role.requires_mfa,
                "access_level": role.access_level.name
            }

        return {
            "allowed": False,
            "reason": f"Action {requested_action} not allowed for {user_role}"
        }


def get_access_control_policy() -> str:
    return """
# Политика контроля доступа к данным держателей карт

## Принцип need-to-know
Доступ предоставляется только тем сотрудникам, которым он необходим
для выполнения их должностных обязанностей.

## Принцип least privilege
Каждый пользователь получает минимальные права, достаточные
для выполнения его задач.

## Требования:
1. Доступ к полному PAN разрешён только:
   - Сотрудникам процессинга платежей
   - Сотрудникам fraud-мониторинга
   - По специальному одобрению для расследований

2. Все остальные роли видят только маскированный PAN

3. Доступ к CDE требует:
   - Уникальной учётной записи
   - Многофакторной аутентификации
   - Одобрения руководителя

4. Доступ пересматривается:
   - Ежеквартально
   - При смене должности
   - При увольнении
"""
```

#### Требование 8: Идентификация и аутентификация

```python
from datetime import datetime, timedelta
import secrets
import hashlib

class AuthenticationRequirements:
    """
    Требования к аутентификации PCI DSS
    """

    # Требования к паролям
    PASSWORD_POLICY = {
        "min_length": 12,  # PCI DSS 4.0: минимум 12 символов
        "require_complexity": True,  # Буквы + цифры + спецсимволы
        "max_age_days": 90,  # Смена каждые 90 дней
        "history": 4,  # Нельзя повторять последние 4 пароля
        "lockout_threshold": 10,  # Блокировка после 10 неудачных попыток
        "lockout_duration_minutes": 30
    }

    # MFA требуется для:
    MFA_REQUIRED_FOR = [
        "all_remote_access",
        "all_administrative_access",
        "all_access_to_cde"
    ]


class MFAImplementation:
    """Реализация многофакторной аутентификации"""

    APPROVED_METHODS = [
        "hardware_token",
        "software_token_totp",
        "push_notification",
        "sms_otp",  # Менее безопасный, но допустим
        "biometric"
    ]

    def verify_mfa(self, user_id: str, mfa_token: str, method: str) -> dict:
        """Проверка MFA"""
        if method not in self.APPROVED_METHODS:
            return {"valid": False, "reason": "Unapproved MFA method"}

        # Проверка токена (зависит от метода)
        is_valid = self._verify_token(user_id, mfa_token, method)

        return {
            "valid": is_valid,
            "timestamp": datetime.now().isoformat(),
            "method": method
        }

    def _verify_token(self, user_id: str, token: str, method: str) -> bool:
        """Верификация токена (placeholder)"""
        # Реальная реализация зависит от MFA провайдера
        return True


class UserAccountManagement:
    """Управление учётными записями"""

    def get_account_requirements(self) -> dict:
        return {
            "unique_ids": {
                "requirement": "Каждый пользователь должен иметь уникальный ID",
                "no_shared_accounts": True,
                "no_generic_accounts": True
            },
            "service_accounts": {
                "documented": True,
                "password_rotation": "at_least_annually",
                "restricted_permissions": True
            },
            "vendor_accounts": {
                "enabled_only_when_needed": True,
                "monitored_when_active": True,
                "strong_authentication": True
            },
            "account_lifecycle": {
                "onboarding": "documented_approval_process",
                "modification": "approved_change_request",
                "offboarding": "immediate_disable_or_within_24h"
            }
        }


class SessionManagement:
    """Управление сессиями"""

    SESSION_TIMEOUT_MINUTES = 15  # Для неактивных сессий

    def configure_session(self) -> dict:
        return {
            "idle_timeout": f"{self.SESSION_TIMEOUT_MINUTES} minutes",
            "absolute_timeout": "8 hours",
            "re_authentication": "For sensitive operations",
            "session_token": {
                "cryptographically_random": True,
                "regenerate_after_login": True,
                "invalidate_on_logout": True
            }
        }
```

### Цель 5: Мониторинг и тестирование

#### Требование 10: Логирование и мониторинг

```python
from datetime import datetime
from dataclasses import dataclass
from typing import List
import json

@dataclass
class AuditLogEntry:
    """
    Запись аудит-лога согласно PCI DSS
    """
    timestamp: datetime
    user_id: str
    event_type: str
    component: str
    success: bool
    source_ip: str
    details: dict


class AuditLoggingSystem:
    """
    Система аудит-логирования
    Требование 10 PCI DSS
    """

    # События, которые ОБЯЗАТЕЛЬНО логировать
    REQUIRED_EVENTS = [
        "user_login_success",
        "user_login_failure",
        "user_logout",
        "access_to_cardholder_data",
        "actions_by_admin",
        "access_to_audit_logs",
        "invalid_access_attempts",
        "use_of_identification_mechanisms",
        "creation_of_system_objects",
        "audit_log_initialization",
        "audit_log_stopping",
        "security_event"
    ]

    # Данные, которые ОБЯЗАТЕЛЬНО включать в лог
    REQUIRED_LOG_FIELDS = [
        "user_identification",
        "event_type",
        "date_and_time",
        "success_or_failure",
        "origination_of_event",
        "affected_data_component_resource"
    ]

    def log_event(self, event: AuditLogEntry) -> str:
        """Создание записи аудит-лога"""
        log_record = {
            "timestamp": event.timestamp.isoformat(),
            "user_id": event.user_id,
            "event_type": event.event_type,
            "component": event.component,
            "success": event.success,
            "source_ip": event.source_ip,
            "details": event.details
        }

        # Логи должны быть защищены от изменения
        return self._write_immutable_log(log_record)

    def _write_immutable_log(self, record: dict) -> str:
        """Запись в неизменяемое хранилище"""
        # Реализация зависит от системы (SIEM, централизованный лог-сервер)
        log_id = self._generate_log_id()
        # Записать в защищённое хранилище
        return log_id

    def get_retention_policy(self) -> dict:
        """Политика хранения логов"""
        return {
            "online_retention": "90 days minimum",
            "offline_retention": "1 year minimum",
            "protection": [
                "Access restricted to authorized personnel",
                "Logs cannot be modified or deleted",
                "Integrity monitoring enabled"
            ]
        }


class LogMonitoring:
    """Мониторинг логов"""

    ALERTS_REQUIRED_FOR = [
        "multiple_failed_logins",
        "unauthorized_access_attempts",
        "changes_to_security_controls",
        "audit_log_tampering",
        "malware_detection",
        "critical_system_errors"
    ]

    def get_monitoring_requirements(self) -> dict:
        return {
            "daily_review": {
                "required": True,
                "logs_to_review": [
                    "security_events",
                    "authentication_logs",
                    "access_to_cde_logs",
                    "admin_activity_logs"
                ]
            },
            "automated_monitoring": {
                "required": True,
                "alerts": self.ALERTS_REQUIRED_FOR,
                "response_time": "immediate_for_critical"
            },
            "time_synchronization": {
                "ntp_required": True,
                "all_systems_synchronized": True
            }
        }
```

#### Требование 11: Тестирование безопасности

```python
class SecurityTestingRequirements:
    """
    Требования к тестированию безопасности
    """

    def get_testing_schedule(self) -> dict:
        return {
            "vulnerability_scans": {
                "internal": {
                    "frequency": "quarterly",
                    "tool": "approved_scanning_tool",
                    "rescan_after_remediation": True
                },
                "external": {
                    "frequency": "quarterly",
                    "performed_by": "approved_scanning_vendor_asv",
                    "passing_scan_required": True
                }
            },
            "penetration_testing": {
                "frequency": "at_least_annually",
                "after_significant_changes": True,
                "scope": [
                    "network_layer",
                    "application_layer",
                    "segmentation_controls"
                ],
                "methodology": "industry_accepted"
            },
            "wireless_analysis": {
                "frequency": "quarterly",
                "scope": "all_locations_with_cde"
            },
            "segmentation_testing": {
                "frequency": "at_least_annually",
                "after_segmentation_changes": True
            }
        }


class ASVScanRequirements:
    """Требования к ASV-сканированию"""

    def get_requirements(self) -> dict:
        return {
            "vendor": {
                "must_be": "PCI SSC Approved Scanning Vendor",
                "list_available_at": "pcisecuritystandards.org"
            },
            "scope": [
                "All external IP addresses",
                "All external-facing domains",
                "All internet-accessible systems"
            ],
            "passing_criteria": {
                "no_vulnerabilities": ["cvss >= 4.0"],
                "no_high_severity": True,
                "false_positives_documented": True
            },
            "remediation": {
                "rescan_required_after_fix": True,
                "90_day_window_for_passing": True
            }
        }
```

### Цель 6: Политика информационной безопасности

#### Требование 12: Политики и процедуры

```python
class InformationSecurityPolicy:
    """
    Политика информационной безопасности
    Требование 12 PCI DSS
    """

    def get_required_policies(self) -> List[str]:
        return [
            "Information Security Policy",
            "Acceptable Use Policy",
            "Access Control Policy",
            "Password Policy",
            "Incident Response Policy",
            "Data Classification Policy",
            "Vendor Management Policy",
            "Change Management Policy",
            "Physical Security Policy",
            "Network Security Policy",
            "Encryption Policy",
            "Logging and Monitoring Policy"
        ]

    def get_policy_requirements(self) -> dict:
        return {
            "review": {
                "frequency": "at_least_annually",
                "after_significant_changes": True,
                "approval": "executive_management"
            },
            "distribution": {
                "all_relevant_personnel": True,
                "acknowledgment_required": True
            },
            "awareness": {
                "initial_training": "upon_hire",
                "annual_training": True,
                "acknowledgment_tracking": True
            }
        }


class IncidentResponsePlan:
    """План реагирования на инциденты"""

    def get_plan_requirements(self) -> dict:
        return {
            "roles_and_responsibilities": {
                "incident_response_team": True,
                "contact_list": True,
                "escalation_procedures": True
            },
            "incident_types": [
                "Data breach",
                "Unauthorized access",
                "Malware infection",
                "DoS attack",
                "Physical security breach"
            ],
            "procedures": {
                "detection": "How to identify incidents",
                "containment": "How to limit damage",
                "eradication": "How to remove threat",
                "recovery": "How to restore normal operations",
                "lessons_learned": "Post-incident review"
            },
            "notification": {
                "payment_brands": "per_brand_requirements",
                "law_enforcement": "as_required",
                "affected_parties": "as_required_by_law"
            },
            "testing": {
                "frequency": "at_least_annually",
                "type": "tabletop_or_simulation"
            }
        }


class ServiceProviderManagement:
    """Управление сервис-провайдерами"""

    def get_requirements(self) -> dict:
        return {
            "due_diligence": {
                "before_engagement": True,
                "pci_dss_status_verification": True,
                "scope_of_services": True
            },
            "contractual_requirements": [
                "Acknowledgment of responsibility for CHD security",
                "PCI DSS compliance attestation",
                "Right to audit",
                "Breach notification requirements",
                "Clear delineation of responsibilities"
            ],
            "monitoring": {
                "annual_pci_dss_status_check": True,
                "aoc_collection": True,
                "responsibility_matrix": True
            }
        }
```

---

## Практические рекомендации для API

### 1. Tokenization для безопасной обработки платежей

```python
import secrets
from typing import Optional
from datetime import datetime, timedelta

class TokenizationService:
    """
    Токенизация данных карт
    Замена PAN на токен, не имеющий криптографической связи с оригиналом
    """

    def __init__(self):
        self.token_vault = {}  # В реальности - защищённое хранилище

    def tokenize(self, pan: str, expiry: str) -> dict:
        """
        Создание токена для PAN
        """
        # Генерируем случайный токен
        token = f"tok_{secrets.token_hex(16)}"

        # Сохраняем связь в защищённом vault
        self.token_vault[token] = {
            "pan_encrypted": self._encrypt(pan),
            "expiry": expiry,
            "created_at": datetime.now().isoformat(),
            "token_expiry": (datetime.now() + timedelta(days=365)).isoformat()
        }

        return {
            "token": token,
            "last_four": pan[-4:],
            "card_type": self._detect_card_type(pan)
        }

    def detokenize(self, token: str) -> Optional[str]:
        """
        Получение оригинального PAN по токену
        (только в PCI DSS scope!)
        """
        vault_entry = self.token_vault.get(token)
        if not vault_entry:
            return None

        return self._decrypt(vault_entry["pan_encrypted"])

    def _detect_card_type(self, pan: str) -> str:
        """Определение типа карты по BIN"""
        if pan.startswith("4"):
            return "Visa"
        elif pan.startswith(("51", "52", "53", "54", "55")):
            return "Mastercard"
        elif pan.startswith(("34", "37")):
            return "American Express"
        return "Unknown"

    def _encrypt(self, data: str) -> str:
        """Шифрование (placeholder)"""
        return data  # Реальное шифрование

    def _decrypt(self, data: str) -> str:
        """Расшифровка (placeholder)"""
        return data


class PaymentAPIDesign:
    """Дизайн платёжного API с учётом PCI DSS"""

    def create_payment_endpoint(self) -> str:
        """Пример endpoint'а для создания платежа"""
        return """
@app.post("/api/payments")
async def create_payment(request: PaymentRequest):
    '''
    Создание платежа

    ВАЖНО: Этот endpoint должен быть в PCI DSS scope
    если принимает данные карты напрямую.

    Рекомендуется: Использовать payment provider (Stripe, Braintree)
    для вывода из scope.
    '''
    # Вариант 1: Принимаем токен от payment provider
    # (мы вне PCI DSS scope для данных карт)
    payment_token = request.payment_token

    # Вариант 2: Если обрабатываем карты сами
    # (полный PCI DSS scope)
    # pan = request.card_number  # ТРЕБУЕТ PCI DSS!

    result = await payment_processor.charge(
        token=payment_token,
        amount=request.amount,
        currency=request.currency
    )

    return {"payment_id": result.id, "status": result.status}
"""
```

### 2. Минимизация PCI DSS scope

```python
class PCIScopeReduction:
    """Стратегии уменьшения PCI DSS scope"""

    def get_strategies(self) -> dict:
        return {
            "tokenization": {
                "description": "Замена PAN на токены",
                "benefit": "Токены не являются данными карт",
                "implementation": "Использовать payment processor"
            },
            "p2pe": {
                "description": "Point-to-Point Encryption",
                "benefit": "Данные зашифрованы от терминала до процессора",
                "requirement": "PCI P2PE сертифицированное решение"
            },
            "iframe_redirect": {
                "description": "Ввод данных карты на странице провайдера",
                "benefit": "Данные карты не проходят через наши серверы",
                "implementation": "Stripe Elements, Braintree Drop-in"
            },
            "network_segmentation": {
                "description": "Изоляция CDE от остальной сети",
                "benefit": "Уменьшает количество систем в scope",
                "requirement": "Правильно настроенные файрволы"
            }
        }


# Пример с Stripe (вне PCI DSS scope)
STRIPE_INTEGRATION = """
# Frontend (JavaScript)
const stripe = Stripe('pk_test_...');
const elements = stripe.elements();
const cardElement = elements.create('card');
cardElement.mount('#card-element');

// Данные карты никогда не попадают на наш сервер!
const {paymentMethod, error} = await stripe.createPaymentMethod({
    type: 'card',
    card: cardElement
});

// Отправляем только токен на наш сервер
await fetch('/api/payments', {
    method: 'POST',
    body: JSON.stringify({
        payment_method_id: paymentMethod.id,  // Только ID, не данные карты!
        amount: 1000
    })
});
"""
```

---

## Штрафы и последствия несоблюдения

### Финансовые санкции

| Источник | Размер штрафа |
|----------|---------------|
| Visa/Mastercard | $5,000 - $100,000 в месяц |
| При утечке | До $500,000+ за инцидент |
| Компенсация карт | $3 - $10 за каждую перевыпущенную карту |
| Forensic investigation | $20,000 - $100,000+ |

### Дополнительные последствия

- Запрет на приём карт (смертельно для e-commerce)
- Повышенные комиссии за транзакции
- Репутационный ущерб
- Регуляторные санкции
- Коллективные иски

```python
def calculate_breach_cost(cards_compromised: int,
                           forensic_needed: bool,
                           reissuance_needed: bool) -> dict:
    """
    Оценка стоимости инцидента
    """
    costs = {
        "forensic_investigation": 50000 if forensic_needed else 0,
        "card_reissuance": cards_compromised * 5 if reissuance_needed else 0,
        "fines_estimated": min(cards_compromised * 50, 500000),
        "notification_costs": cards_compromised * 2,
        "credit_monitoring": cards_compromised * 10,
        "legal_fees": 100000,  # Оценка
        "reputation_damage": "Incalculable"
    }

    total = sum(v for v in costs.values() if isinstance(v, (int, float)))
    costs["total_estimated"] = total

    return costs
```

---

## Чеклист соответствия PCI DSS

### Быстрый чеклист по требованиям

- [ ] **Req 1**: Файрволы настроены и документированы
- [ ] **Req 2**: Системы hardened, дефолтные пароли изменены
- [ ] **Req 3**: Данные карт защищены (шифрование, маскирование)
- [ ] **Req 4**: TLS 1.2+ для передачи данных
- [ ] **Req 5**: Антивирус установлен и обновляется
- [ ] **Req 6**: Безопасная разработка, патчи установлены
- [ ] **Req 7**: Доступ ограничен по need-to-know
- [ ] **Req 8**: Уникальные ID, MFA, политика паролей
- [ ] **Req 9**: Физическая безопасность (если применимо)
- [ ] **Req 10**: Логирование и мониторинг настроены
- [ ] **Req 11**: Сканирование и пентесты проводятся
- [ ] **Req 12**: Политики документированы и обновляются

### Для разработчиков API

```python
class PCIDSSAPIChecklist:
    """Чеклист для API разработчиков"""

    def get_checklist(self) -> dict:
        return {
            "data_handling": [
                "Никогда не логировать полный PAN",
                "Никогда не хранить CVV/CVC",
                "Маскировать PAN в логах и UI",
                "Шифровать PAN при хранении (AES-256)"
            ],
            "transmission": [
                "TLS 1.2 или выше",
                "Валидные сертификаты",
                "HSTS включён",
                "Нет передачи CHD по email/chat"
            ],
            "authentication": [
                "MFA для доступа к CDE",
                "Уникальные учётные записи",
                "Сильные пароли (12+ символов)",
                "Таймаут сессии 15 минут"
            ],
            "logging": [
                "Логировать все доступы к CHD",
                "Логировать все admin действия",
                "Защитить логи от изменения",
                "Хранить логи минимум 1 год"
            ],
            "development": [
                "Не использовать production данные в dev/test",
                "Code review для всех изменений",
                "Vulnerability scanning",
                "Separation of duties"
            ]
        }
```

---

## Полезные ресурсы

- [PCI Security Standards Council](https://www.pcisecuritystandards.org/)
- [PCI DSS v4.0 Document Library](https://www.pcisecuritystandards.org/document_library/)
- [SAQ Templates](https://www.pcisecuritystandards.org/document_library/?category=saqs)
- [List of ASV Companies](https://www.pcisecuritystandards.org/assessors_and_solutions/approved_scanning_vendors)
- [PCI SSC Blog](https://blog.pcisecuritystandards.org/)

---

[prev: 02-ccpa](./02-ccpa.md) | [next: 04-hipaa](./04-hipaa.md)
