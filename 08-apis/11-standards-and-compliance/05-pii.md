# PII (Personally Identifiable Information)

## Введение

**PII** (Personally Identifiable Information) — это любая информация, которая может быть использована для идентификации, контакта или определения местоположения конкретного человека, либо самостоятельно, либо в сочетании с другими источниками данных.

Понимание и правильная обработка PII является фундаментом для соответствия большинству регуляторных требований (GDPR, CCPA, HIPAA и др.) и критически важно для защиты конфиденциальности пользователей.

---

## Определение и классификация

### Что такое PII

```python
from enum import Enum
from dataclasses import dataclass
from typing import List, Optional

class PIISensitivity(Enum):
    """Уровни чувствительности PII"""
    DIRECT = "direct"  # Позволяет напрямую идентифицировать
    QUASI = "quasi"    # Может идентифицировать в комбинации
    SENSITIVE = "sensitive"  # Особо чувствительные данные


@dataclass
class PIIElement:
    """Элемент PII"""
    name: str
    sensitivity: PIISensitivity
    description: str
    examples: List[str]


class PIIClassification:
    """
    Классификация типов PII
    """

    # Прямые идентификаторы — сами по себе идентифицируют человека
    DIRECT_IDENTIFIERS = [
        PIIElement(
            name="Full Name",
            sensitivity=PIISensitivity.DIRECT,
            description="Полное имя человека",
            examples=["John Smith", "Иван Иванов"]
        ),
        PIIElement(
            name="Social Security Number",
            sensitivity=PIISensitivity.SENSITIVE,
            description="Номер социального страхования (США)",
            examples=["123-45-6789"]
        ),
        PIIElement(
            name="Passport Number",
            sensitivity=PIISensitivity.SENSITIVE,
            description="Номер паспорта",
            examples=["AB1234567"]
        ),
        PIIElement(
            name="Driver's License",
            sensitivity=PIISensitivity.SENSITIVE,
            description="Номер водительского удостоверения",
            examples=["D123-456-78-901"]
        ),
        PIIElement(
            name="Email Address",
            sensitivity=PIISensitivity.DIRECT,
            description="Адрес электронной почты",
            examples=["john.smith@email.com"]
        ),
        PIIElement(
            name="Phone Number",
            sensitivity=PIISensitivity.DIRECT,
            description="Номер телефона",
            examples=["+1-555-123-4567"]
        ),
        PIIElement(
            name="Physical Address",
            sensitivity=PIISensitivity.DIRECT,
            description="Физический адрес",
            examples=["123 Main St, City, State 12345"]
        ),
        PIIElement(
            name="IP Address",
            sensitivity=PIISensitivity.DIRECT,
            description="IP-адрес устройства",
            examples=["192.168.1.1", "2001:db8::1"]
        ),
        PIIElement(
            name="Biometric Data",
            sensitivity=PIISensitivity.SENSITIVE,
            description="Биометрические данные",
            examples=["Отпечатки пальцев", "Скан лица", "Сетчатка"]
        ),
        PIIElement(
            name="National ID",
            sensitivity=PIISensitivity.SENSITIVE,
            description="Национальный идентификатор",
            examples=["ИНН", "СНИЛС"]
        )
    ]

    # Квази-идентификаторы — могут идентифицировать в комбинации
    QUASI_IDENTIFIERS = [
        PIIElement(
            name="Date of Birth",
            sensitivity=PIISensitivity.QUASI,
            description="Дата рождения",
            examples=["1990-01-15"]
        ),
        PIIElement(
            name="Gender",
            sensitivity=PIISensitivity.QUASI,
            description="Пол",
            examples=["Male", "Female", "Other"]
        ),
        PIIElement(
            name="ZIP Code",
            sensitivity=PIISensitivity.QUASI,
            description="Почтовый индекс",
            examples=["10001", "SW1A 1AA"]
        ),
        PIIElement(
            name="Race/Ethnicity",
            sensitivity=PIISensitivity.SENSITIVE,
            description="Раса или этническая принадлежность",
            examples=["Caucasian", "Asian", "African American"]
        ),
        PIIElement(
            name="Job Title",
            sensitivity=PIISensitivity.QUASI,
            description="Должность",
            examples=["Senior Developer", "CEO"]
        ),
        PIIElement(
            name="Employer",
            sensitivity=PIISensitivity.QUASI,
            description="Работодатель",
            examples=["Google", "Microsoft"]
        ),
        PIIElement(
            name="Education",
            sensitivity=PIISensitivity.QUASI,
            description="Образование",
            examples=["MIT", "Stanford"]
        )
    ]

    # Особо чувствительные данные
    SENSITIVE_PII = [
        PIIElement(
            name="Health Information",
            sensitivity=PIISensitivity.SENSITIVE,
            description="Медицинская информация",
            examples=["Диагнозы", "Медикаменты", "Результаты анализов"]
        ),
        PIIElement(
            name="Financial Data",
            sensitivity=PIISensitivity.SENSITIVE,
            description="Финансовые данные",
            examples=["Номер банковского счёта", "Кредитная история"]
        ),
        PIIElement(
            name="Credentials",
            sensitivity=PIISensitivity.SENSITIVE,
            description="Учётные данные",
            examples=["Пароли", "PIN-коды", "Секретные вопросы"]
        ),
        PIIElement(
            name="Religious Beliefs",
            sensitivity=PIISensitivity.SENSITIVE,
            description="Религиозные убеждения",
            examples=["Christianity", "Islam", "Buddhism"]
        ),
        PIIElement(
            name="Political Opinions",
            sensitivity=PIISensitivity.SENSITIVE,
            description="Политические взгляды",
            examples=["Членство в партии"]
        ),
        PIIElement(
            name="Sexual Orientation",
            sensitivity=PIISensitivity.SENSITIVE,
            description="Сексуальная ориентация",
            examples=["Heterosexual", "LGBTQ+"]
        ),
        PIIElement(
            name="Genetic Data",
            sensitivity=PIISensitivity.SENSITIVE,
            description="Генетические данные",
            examples=["ДНК-профиль"]
        ),
        PIIElement(
            name="Criminal Records",
            sensitivity=PIISensitivity.SENSITIVE,
            description="Судимость",
            examples=["Криминальная история"]
        )
    ]
```

### Различие между PII и Non-PII

```python
class PIIDetector:
    """
    Определение, является ли информация PII
    """

    # Очевидно PII
    ALWAYS_PII = [
        "full_name", "email", "phone", "ssn", "passport",
        "drivers_license", "address", "ip_address"
    ]

    # Может быть PII в контексте
    CONTEXT_DEPENDENT = [
        "age", "gender", "zip_code", "job_title",
        "company", "purchase_history"
    ]

    # Обычно не PII
    NOT_PII = [
        "aggregated_statistics",
        "anonymous_survey_responses",
        "de_identified_data",
        "truly_random_identifiers"
    ]

    def classify_data(self, data: dict) -> dict:
        """
        Классификация данных на PII и non-PII
        """
        classification = {
            "pii": [],
            "potential_pii": [],
            "non_pii": []
        }

        for field, value in data.items():
            field_lower = field.lower()

            if self._is_direct_pii(field_lower):
                classification["pii"].append({
                    "field": field,
                    "type": "direct_identifier",
                    "action_required": "protect_or_anonymize"
                })
            elif self._is_quasi_identifier(field_lower):
                classification["potential_pii"].append({
                    "field": field,
                    "type": "quasi_identifier",
                    "note": "May identify when combined with other data"
                })
            else:
                classification["non_pii"].append(field)

        return classification

    def _is_direct_pii(self, field: str) -> bool:
        """Проверка на прямой идентификатор"""
        pii_patterns = [
            "name", "email", "phone", "address", "ssn",
            "passport", "license", "ip", "device_id"
        ]
        return any(pattern in field for pattern in pii_patterns)

    def _is_quasi_identifier(self, field: str) -> bool:
        """Проверка на квази-идентификатор"""
        quasi_patterns = [
            "age", "birth", "gender", "zip", "postal",
            "job", "title", "employer", "education"
        ]
        return any(pattern in field for pattern in quasi_patterns)


# Пример использования
detector = PIIDetector()
user_data = {
    "user_id": "abc123",
    "email": "john@example.com",
    "age": 30,
    "zip_code": "10001",
    "purchase_count": 5
}

result = detector.classify_data(user_data)
# {
#   "pii": [{"field": "email", "type": "direct_identifier", ...}],
#   "potential_pii": [{"field": "age", ...}, {"field": "zip_code", ...}],
#   "non_pii": ["user_id", "purchase_count"]
# }
```

---

## PII в различных регуляторных контекстах

### Сравнение определений PII

```python
class PIIDefinitions:
    """
    Определения PII в различных регуляторных рамках
    """

    GDPR = {
        "term": "Personal Data",
        "definition": """
            Любая информация, относящаяся к идентифицированному или
            идентифицируемому физическому лицу ('субъекту данных')
        """,
        "includes": [
            "Имя",
            "Идентификационный номер",
            "Данные о местоположении",
            "Онлайн-идентификатор",
            "Физические, физиологические, генетические, психические,
             экономические, культурные или социальные признаки"
        ],
        "special_categories": [
            "Расовое или этническое происхождение",
            "Политические взгляды",
            "Религиозные убеждения",
            "Членство в профсоюзах",
            "Генетические данные",
            "Биометрические данные",
            "Данные о здоровье",
            "Сексуальная жизнь/ориентация"
        ]
    }

    CCPA = {
        "term": "Personal Information",
        "definition": """
            Информация, которая идентифицирует, относится к, описывает,
            может быть связана с, или разумно может быть связана с
            конкретным потребителем или домохозяйством
        """,
        "categories": [
            "Идентификаторы",
            "Коммерческая информация",
            "Биометрическая информация",
            "Интернет-активность",
            "Геолокационные данные",
            "Аудио/визуальная информация",
            "Профессиональная информация",
            "Образовательная информация",
            "Выводы из данных"
        ],
        "excludes": [
            "Публично доступная информация",
            "De-identified или агрегированная информация"
        ]
    }

    HIPAA = {
        "term": "Protected Health Information (PHI)",
        "definition": """
            Индивидуально идентифицируемая медицинская информация,
            передаваемая или хранимая в любой форме
        """,
        "identifiers_18": [
            "Имена",
            "Географические данные (меньше штата)",
            "Даты (кроме года)",
            "Номера телефонов",
            "Номера факсов",
            "Email-адреса",
            "SSN",
            "Номера медицинских записей",
            "Номера страховых полисов",
            "Номера счетов",
            "Номера сертификатов/лицензий",
            "Идентификаторы транспортных средств",
            "Идентификаторы устройств",
            "Web URL",
            "IP-адреса",
            "Биометрические данные",
            "Фотографии лица",
            "Любой уникальный идентификатор"
        ]
    }

    NIST = {
        "term": "Personally Identifiable Information",
        "definition": """
            Любая информация об индивидууме, хранящаяся агентством,
            включая информацию, которая может быть использована
            для определения или отслеживания личности
        """,
        "types": {
            "linked": "Непосредственно связана с индивидуумом (имя, SSN)",
            "linkable": "Может быть связана с индивидуумом (дата рождения, место)"
        }
    }
```

---

## Практические методы защиты PII

### 1. Минимизация данных

```python
from dataclasses import dataclass
from typing import List, Dict, Optional
from enum import Enum

class DataPurpose(Enum):
    """Цели сбора данных"""
    ACCOUNT_CREATION = "account_creation"
    PAYMENT_PROCESSING = "payment_processing"
    MARKETING = "marketing"
    ANALYTICS = "analytics"
    LEGAL_COMPLIANCE = "legal_compliance"


class PIIMinimization:
    """
    Принцип минимизации данных:
    Собирать только те данные, которые необходимы для цели
    """

    # Минимально необходимые поля для каждой цели
    REQUIRED_FIELDS = {
        DataPurpose.ACCOUNT_CREATION: ["email", "password"],
        DataPurpose.PAYMENT_PROCESSING: [
            "name", "billing_address", "payment_method"
        ],
        DataPurpose.MARKETING: ["email"],  # С согласия
        DataPurpose.ANALYTICS: [],  # Не требует PII
        DataPurpose.LEGAL_COMPLIANCE: [
            "name", "address", "tax_id"
        ]
    }

    def validate_data_collection(self,
                                  collected_fields: List[str],
                                  purpose: DataPurpose) -> dict:
        """
        Проверка соответствия принципу минимизации
        """
        required = set(self.REQUIRED_FIELDS[purpose])
        collected = set(collected_fields)

        # Лишние поля
        excessive = collected - required
        # Недостающие поля
        missing = required - collected

        return {
            "compliant": len(excessive) == 0,
            "required_fields": list(required),
            "collected_fields": list(collected),
            "excessive_fields": list(excessive),
            "missing_fields": list(missing),
            "recommendation": self._get_recommendation(excessive)
        }

    def _get_recommendation(self, excessive: set) -> str:
        if not excessive:
            return "Data collection follows minimization principle"
        return f"Consider removing unnecessary fields: {', '.join(excessive)}"


# Пример модели с минимизацией
from pydantic import BaseModel, Field

class MinimalUserRegistration(BaseModel):
    """Минимальная модель для регистрации"""
    email: str = Field(..., description="Required for account")
    password: str = Field(..., description="Required for authentication")
    # НЕ собираем: name, phone, address, dob
    # Эти данные запрашиваем только когда действительно нужны


class ExtendedProfileOptional(BaseModel):
    """Дополнительные данные — только по необходимости"""
    name: Optional[str] = None  # Для персонализации, если хотят
    phone: Optional[str] = None  # Только для 2FA, если выбрали
    address: Optional[str] = None  # Только при заказе физических товаров
```

### 2. Псевдонимизация

```python
import hashlib
import secrets
from typing import Dict, Optional
from datetime import datetime

class Pseudonymization:
    """
    Псевдонимизация — замена идентификаторов на псевдонимы
    Данные можно восстановить при наличии ключа
    """

    def __init__(self, secret_key: bytes):
        self.secret_key = secret_key
        self.mapping_table: Dict[str, str] = {}  # В реальности — защищённое хранилище

    def pseudonymize(self, identifier: str) -> str:
        """
        Создание псевдонима для идентификатора
        """
        # Генерируем случайный псевдоним
        pseudonym = f"PSE_{secrets.token_hex(8)}"

        # Сохраняем маппинг (в защищённом хранилище)
        self.mapping_table[pseudonym] = self._encrypt(identifier)

        return pseudonym

    def re_identify(self, pseudonym: str) -> Optional[str]:
        """
        Восстановление оригинального идентификатора
        Только с правильным ключом
        """
        encrypted = self.mapping_table.get(pseudonym)
        if not encrypted:
            return None
        return self._decrypt(encrypted)

    def _encrypt(self, data: str) -> str:
        """Шифрование данных маппинга"""
        # Реализация зависит от требований
        return data  # Placeholder

    def _decrypt(self, data: str) -> str:
        """Расшифровка данных маппинга"""
        return data  # Placeholder


class TokenizedPII:
    """
    Токенизация PII — замена на случайный токен
    Без криптографической связи с оригиналом
    """

    def __init__(self):
        self.token_vault: Dict[str, str] = {}  # Защищённое хранилище

    def tokenize(self, pii: str, pii_type: str) -> str:
        """
        Создание токена для PII
        """
        token = f"TOK_{pii_type.upper()}_{secrets.token_hex(12)}"

        # Храним связь в vault
        self.token_vault[token] = pii

        return token

    def detokenize(self, token: str) -> Optional[str]:
        """
        Получение оригинального PII по токену
        """
        return self.token_vault.get(token)


# Пример использования
pseudonymizer = Pseudonymization(b"secret_key")
tokenizer = TokenizedPII()

# Оригинальные данные
user_email = "john.doe@example.com"
user_ssn = "123-45-6789"

# Псевдонимизация email
pseudo_email = pseudonymizer.pseudonymize(user_email)
# "PSE_a1b2c3d4e5f6g7h8"

# Токенизация SSN
token_ssn = tokenizer.tokenize(user_ssn, "ssn")
# "TOK_SSN_1a2b3c4d5e6f7g8h9i0j"
```

### 3. Анонимизация

```python
from typing import Dict, List
import random

class Anonymization:
    """
    Анонимизация — необратимое удаление идентифицирующей информации
    После анонимизации данные НЕ являются PII
    """

    def k_anonymity(self, dataset: List[Dict],
                     quasi_identifiers: List[str],
                     k: int) -> List[Dict]:
        """
        K-анонимность: каждая комбинация квази-идентификаторов
        должна встречаться минимум k раз

        Например, k=5 означает, что каждая комбинация
        (возраст, пол, zip) есть минимум у 5 людей
        """
        # Группируем по квази-идентификаторам
        groups = self._group_by_quasi_ids(dataset, quasi_identifiers)

        # Обобщаем группы с < k записями
        anonymized = []
        for group in groups.values():
            if len(group) >= k:
                anonymized.extend(group)
            else:
                # Обобщаем до достижения k
                generalized = self._generalize_group(group, quasi_identifiers)
                anonymized.extend(generalized)

        return anonymized

    def _generalize_group(self, group: List[Dict],
                           quasi_ids: List[str]) -> List[Dict]:
        """Обобщение данных для достижения k-анонимности"""
        generalized = []
        for record in group:
            gen_record = record.copy()
            for qi in quasi_ids:
                gen_record[qi] = self._generalize_value(qi, record[qi])
            generalized.append(gen_record)
        return generalized

    def _generalize_value(self, field: str, value) -> str:
        """Обобщение конкретного значения"""
        generalizations = {
            "age": lambda v: f"{(int(v) // 10) * 10}-{(int(v) // 10) * 10 + 9}",
            "zip_code": lambda v: v[:3] + "**",
            "city": lambda _: "***",
        }

        if field in generalizations:
            return generalizations[field](value)
        return "*"


class DataMasking:
    """
    Маскирование данных для отображения
    """

    @staticmethod
    def mask_email(email: str) -> str:
        """Маскирование email: j***@example.com"""
        if "@" not in email:
            return "***@***"
        local, domain = email.split("@")
        if len(local) <= 1:
            return f"*@{domain}"
        return f"{local[0]}***@{domain}"

    @staticmethod
    def mask_phone(phone: str) -> str:
        """Маскирование телефона: ***-***-1234"""
        digits = ''.join(filter(str.isdigit, phone))
        if len(digits) < 4:
            return "***"
        return f"***-***-{digits[-4:]}"

    @staticmethod
    def mask_ssn(ssn: str) -> str:
        """Маскирование SSN: ***-**-1234"""
        digits = ''.join(filter(str.isdigit, ssn))
        if len(digits) < 4:
            return "***-**-****"
        return f"***-**-{digits[-4:]}"

    @staticmethod
    def mask_credit_card(card: str) -> str:
        """Маскирование карты: ****-****-****-1234"""
        digits = ''.join(filter(str.isdigit, card))
        if len(digits) < 4:
            return "****-****-****-****"
        return f"****-****-****-{digits[-4:]}"

    @staticmethod
    def mask_name(name: str) -> str:
        """Маскирование имени: J*** D***"""
        parts = name.split()
        masked = []
        for part in parts:
            if len(part) > 0:
                masked.append(f"{part[0]}***")
        return " ".join(masked)


# Пример использования
masker = DataMasking()

print(masker.mask_email("john.doe@example.com"))  # j***@example.com
print(masker.mask_phone("+1-555-123-4567"))       # ***-***-4567
print(masker.mask_ssn("123-45-6789"))             # ***-**-6789
print(masker.mask_credit_card("4111111111111111"))  # ****-****-****-1111
```

### 4. Шифрование

```python
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
import os
import base64

class PIIEncryption:
    """
    Шифрование PII для безопасного хранения
    """

    def __init__(self):
        self.key = Fernet.generate_key()
        self.fernet = Fernet(self.key)

    def encrypt_pii(self, pii: str) -> bytes:
        """Шифрование PII"""
        return self.fernet.encrypt(pii.encode())

    def decrypt_pii(self, encrypted: bytes) -> str:
        """Расшифровка PII"""
        return self.fernet.decrypt(encrypted).decode()


class FieldLevelEncryption:
    """
    Шифрование на уровне полей
    Разные поля — разные ключи
    """

    def __init__(self, master_key: bytes):
        self.master_key = master_key
        self.field_keys: Dict[str, bytes] = {}

    def _derive_field_key(self, field_name: str) -> bytes:
        """Деривация ключа для конкретного поля"""
        if field_name not in self.field_keys:
            salt = field_name.encode()
            kdf = PBKDF2HMAC(
                algorithm=hashes.SHA256(),
                length=32,
                salt=salt,
                iterations=100000,
            )
            self.field_keys[field_name] = kdf.derive(self.master_key)
        return self.field_keys[field_name]

    def encrypt_field(self, field_name: str, value: str) -> bytes:
        """Шифрование конкретного поля"""
        key = self._derive_field_key(field_name)
        nonce = os.urandom(12)
        aesgcm = AESGCM(key)
        encrypted = aesgcm.encrypt(nonce, value.encode(), None)
        return nonce + encrypted

    def decrypt_field(self, field_name: str, encrypted_value: bytes) -> str:
        """Расшифровка поля"""
        key = self._derive_field_key(field_name)
        nonce = encrypted_value[:12]
        ciphertext = encrypted_value[12:]
        aesgcm = AESGCM(key)
        return aesgcm.decrypt(nonce, ciphertext, None).decode()


class EncryptedPIIStorage:
    """
    Хранилище с шифрованием PII полей
    """

    # Поля, которые нужно шифровать
    PII_FIELDS = ["name", "email", "phone", "ssn", "address", "dob"]

    def __init__(self, encryption_key: bytes):
        self.encryption = FieldLevelEncryption(encryption_key)

    def encrypt_record(self, record: dict) -> dict:
        """Шифрование PII полей в записи"""
        encrypted = {}
        for key, value in record.items():
            if key in self.PII_FIELDS and value:
                encrypted[key] = self.encryption.encrypt_field(key, str(value))
                encrypted[f"{key}_encrypted"] = True
            else:
                encrypted[key] = value
        return encrypted

    def decrypt_record(self, encrypted_record: dict) -> dict:
        """Расшифровка PII полей"""
        decrypted = {}
        for key, value in encrypted_record.items():
            if key.endswith("_encrypted"):
                continue
            if encrypted_record.get(f"{key}_encrypted"):
                decrypted[key] = self.encryption.decrypt_field(key, value)
            else:
                decrypted[key] = value
        return decrypted
```

---

## PII в API

### 1. Безопасная обработка PII в API

```python
from fastapi import FastAPI, Request, HTTPException, Depends
from pydantic import BaseModel, EmailStr, Field, validator
from typing import Optional, List
import re

app = FastAPI()


class PIIRequest(BaseModel):
    """Модель запроса с PII"""
    email: EmailStr
    phone: Optional[str] = None
    ssn: Optional[str] = None

    @validator('phone')
    def validate_phone(cls, v):
        if v and not re.match(r'^\+?[\d\s-]{10,}$', v):
            raise ValueError('Invalid phone format')
        return v

    @validator('ssn')
    def validate_ssn(cls, v):
        if v and not re.match(r'^\d{3}-\d{2}-\d{4}$', v):
            raise ValueError('Invalid SSN format')
        return v


class PIISafeLogger:
    """
    Логгер, который никогда не записывает PII
    """
    PII_PATTERNS = {
        "email": r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}',
        "phone": r'\+?\d[\d\s-]{9,}',
        "ssn": r'\d{3}-\d{2}-\d{4}',
        "credit_card": r'\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}',
        "ip": r'\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}'
    }

    def sanitize(self, message: str) -> str:
        """Удаление PII из сообщения"""
        sanitized = message
        for pii_type, pattern in self.PII_PATTERNS.items():
            sanitized = re.sub(pattern, f'[{pii_type.upper()}_REDACTED]', sanitized)
        return sanitized

    def log(self, level: str, message: str, **kwargs):
        """Безопасное логирование"""
        safe_message = self.sanitize(message)
        safe_kwargs = {k: self.sanitize(str(v)) for k, v in kwargs.items()}
        print(f"[{level}] {safe_message} {safe_kwargs}")


# Middleware для защиты PII
class PIIProtectionMiddleware:
    """Middleware для защиты PII в запросах/ответах"""

    async def __call__(self, request: Request, call_next):
        # Логируем запрос без PII
        safe_logger = PIISafeLogger()
        safe_logger.log("INFO", f"Request to {request.url.path}")

        # Обрабатываем запрос
        response = await call_next(request)

        # Проверяем, что PII не утекает в заголовках
        self._check_headers(response.headers)

        return response

    def _check_headers(self, headers):
        """Проверка заголовков на наличие PII"""
        pii_in_headers = ["email", "phone", "ssn", "name"]
        for header in headers:
            header_lower = header.lower()
            if any(pii in header_lower for pii in pii_in_headers):
                # Alert: PII в заголовках!
                pass


# API endpoints
@app.get("/api/users/{user_id}")
async def get_user(user_id: str, include_pii: bool = False):
    """
    Получение пользователя
    PII возвращается только если явно запрошено
    """
    user = await fetch_user(user_id)

    if not include_pii:
        # Удаляем/маскируем PII
        return {
            "id": user["id"],
            "email": DataMasking.mask_email(user["email"]),
            "created_at": user["created_at"]
        }

    # Полные данные — требует дополнительной авторизации
    return user


@app.post("/api/users")
async def create_user(user_data: PIIRequest):
    """
    Создание пользователя
    PII валидируется и шифруется перед сохранением
    """
    # Шифруем PII перед сохранением
    encrypted_data = encrypt_pii_fields(user_data.dict())

    # Сохраняем
    user_id = await save_user(encrypted_data)

    # Возвращаем без PII
    return {"id": user_id, "status": "created"}
```

### 2. PII в URL и параметрах запроса

```python
class PIIURLGuidelines:
    """
    Рекомендации по PII в URL
    """

    # НИКОГДА не включать в URL:
    NEVER_IN_URL = [
        "email",
        "ssn",
        "credit_card",
        "password",
        "phone",
        "address",
        "dob"
    ]

    # Альтернативы:
    ALTERNATIVES = {
        "email_in_url": "Использовать user_id или токен",
        "phone_in_url": "Использовать phone_id или хэш",
        "sensitive_filters": "Передавать в теле POST запроса"
    }


# Плохой пример (НЕ ДЕЛАТЬ!)
# GET /api/users?email=john@example.com
# GET /api/users/john@example.com/orders

# Хороший пример
# GET /api/users/USR_abc123/orders
# POST /api/users/search  с email в теле запроса

@app.post("/api/users/search")
async def search_users(search: dict):
    """
    Поиск пользователей — PII в теле запроса, не в URL
    """
    email = search.get("email")
    # ...


@app.get("/api/users/{user_token}")
async def get_user_by_token(user_token: str):
    """
    Получение пользователя по токену, не по email
    """
    # user_token — это не PII
    user = await find_by_token(user_token)
    return user
```

### 3. PII в ответах API

```python
from pydantic import BaseModel
from typing import Optional, List

class UserResponse(BaseModel):
    """Модель ответа с контролируемым PII"""
    id: str
    email_masked: str  # j***@example.com
    phone_masked: Optional[str] = None  # ***-***-1234
    name_masked: Optional[str] = None  # J*** D***

    class Config:
        # Запрещаем extra поля, чтобы PII случайно не попало
        extra = "forbid"


class PIIResponseFilter:
    """
    Фильтр PII в ответах API
    """

    def __init__(self, pii_fields: List[str]):
        self.pii_fields = pii_fields
        self.masker = DataMasking()

    def filter_response(self, data: dict, mask: bool = True) -> dict:
        """
        Фильтрация PII из ответа
        """
        filtered = {}
        for key, value in data.items():
            if key in self.pii_fields:
                if mask:
                    filtered[f"{key}_masked"] = self._mask_value(key, value)
                # Пропускаем оригинальное значение
            else:
                filtered[key] = value
        return filtered

    def _mask_value(self, field: str, value) -> str:
        """Маскирование значения по типу поля"""
        masking_methods = {
            "email": self.masker.mask_email,
            "phone": self.masker.mask_phone,
            "ssn": self.masker.mask_ssn,
            "name": self.masker.mask_name,
            "credit_card": self.masker.mask_credit_card
        }

        method = masking_methods.get(field, lambda x: "***")
        return method(str(value))


# Middleware для автоматической фильтрации
class PIIResponseMiddleware:
    """Автоматическая фильтрация PII из всех ответов"""

    PII_FIELDS = ["email", "phone", "ssn", "name", "address", "dob"]

    async def __call__(self, request: Request, call_next):
        response = await call_next(request)

        # Проверяем, нужно ли фильтровать
        if self._should_filter(request):
            # Фильтруем response body
            pass

        return response

    def _should_filter(self, request: Request) -> bool:
        """Определяем, нужна ли фильтрация"""
        # Не фильтруем если:
        # - Пользователь запрашивает свои данные
        # - Есть специальный permission
        # - Внутренний API
        return True
```

---

## Инвентаризация и классификация PII

### Data Discovery и классификация

```python
from dataclasses import dataclass
from typing import List, Dict, Set
from datetime import datetime
from enum import Enum

class DataSource(Enum):
    DATABASE = "database"
    FILE_STORAGE = "file_storage"
    API_LOGS = "api_logs"
    THIRD_PARTY = "third_party"


@dataclass
class PIIInventoryItem:
    """Элемент инвентаризации PII"""
    field_name: str
    pii_type: str
    sensitivity: PIISensitivity
    data_source: DataSource
    purpose: str
    retention_period: str
    encryption_status: bool
    access_controls: List[str]
    legal_basis: str  # Для GDPR


class PIIInventory:
    """
    Инвентаризация PII в системе
    """

    def __init__(self):
        self.inventory: List[PIIInventoryItem] = []

    def add_item(self, item: PIIInventoryItem):
        """Добавление элемента в инвентарь"""
        self.inventory.append(item)

    def get_by_sensitivity(self, sensitivity: PIISensitivity) -> List[PIIInventoryItem]:
        """Получение элементов по уровню чувствительности"""
        return [i for i in self.inventory if i.sensitivity == sensitivity]

    def get_unencrypted(self) -> List[PIIInventoryItem]:
        """Получение незашифрованных PII"""
        return [i for i in self.inventory if not i.encryption_status]

    def generate_report(self) -> dict:
        """Генерация отчёта по инвентаризации"""
        return {
            "total_pii_fields": len(self.inventory),
            "by_sensitivity": {
                "sensitive": len(self.get_by_sensitivity(PIISensitivity.SENSITIVE)),
                "direct": len(self.get_by_sensitivity(PIISensitivity.DIRECT)),
                "quasi": len(self.get_by_sensitivity(PIISensitivity.QUASI))
            },
            "encryption_coverage": {
                "encrypted": len([i for i in self.inventory if i.encryption_status]),
                "unencrypted": len(self.get_unencrypted())
            },
            "by_source": self._count_by_source(),
            "generated_at": datetime.now().isoformat()
        }

    def _count_by_source(self) -> Dict[str, int]:
        counts = {}
        for item in self.inventory:
            source = item.data_source.value
            counts[source] = counts.get(source, 0) + 1
        return counts


# Пример использования
inventory = PIIInventory()

inventory.add_item(PIIInventoryItem(
    field_name="users.email",
    pii_type="email",
    sensitivity=PIISensitivity.DIRECT,
    data_source=DataSource.DATABASE,
    purpose="Account authentication",
    retention_period="Account lifetime + 30 days",
    encryption_status=True,
    access_controls=["auth_service", "admin_panel"],
    legal_basis="Contract (GDPR Art. 6(1)(b))"
))

inventory.add_item(PIIInventoryItem(
    field_name="orders.billing_address",
    pii_type="address",
    sensitivity=PIISensitivity.DIRECT,
    data_source=DataSource.DATABASE,
    purpose="Order fulfillment",
    retention_period="7 years (tax requirement)",
    encryption_status=True,
    access_controls=["order_service", "finance_team"],
    legal_basis="Legal obligation (GDPR Art. 6(1)(c))"
))

report = inventory.generate_report()
```

### Автоматическое обнаружение PII

```python
import re
from typing import List, Tuple

class PIIScanner:
    """
    Автоматический сканер PII в данных
    """

    PATTERNS = {
        "email": r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}',
        "phone_us": r'\+?1?[-.\s]?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}',
        "phone_intl": r'\+\d{1,3}[-.\s]?\d{1,4}[-.\s]?\d{1,4}[-.\s]?\d{1,9}',
        "ssn": r'\d{3}-\d{2}-\d{4}',
        "credit_card": r'\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}',
        "ip_v4": r'\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}',
        "date_of_birth": r'\d{4}-\d{2}-\d{2}|\d{2}/\d{2}/\d{4}',
        "passport_us": r'[A-Z]\d{8}',
        "drivers_license": r'[A-Z]{1,2}\d{6,8}'
    }

    def scan_text(self, text: str) -> List[Tuple[str, str, int, int]]:
        """
        Сканирование текста на PII

        Returns: List of (pii_type, value, start_pos, end_pos)
        """
        findings = []

        for pii_type, pattern in self.PATTERNS.items():
            for match in re.finditer(pattern, text):
                findings.append((
                    pii_type,
                    match.group(),
                    match.start(),
                    match.end()
                ))

        return findings

    def scan_dict(self, data: dict, path: str = "") -> List[dict]:
        """
        Рекурсивное сканирование словаря на PII
        """
        findings = []

        for key, value in data.items():
            current_path = f"{path}.{key}" if path else key

            if isinstance(value, str):
                text_findings = self.scan_text(value)
                for pii_type, pii_value, _, _ in text_findings:
                    findings.append({
                        "path": current_path,
                        "pii_type": pii_type,
                        "value_masked": self._mask_finding(pii_value, pii_type)
                    })

            elif isinstance(value, dict):
                findings.extend(self.scan_dict(value, current_path))

            elif isinstance(value, list):
                for i, item in enumerate(value):
                    if isinstance(item, dict):
                        findings.extend(self.scan_dict(item, f"{current_path}[{i}]"))
                    elif isinstance(item, str):
                        text_findings = self.scan_text(item)
                        for pii_type, pii_value, _, _ in text_findings:
                            findings.append({
                                "path": f"{current_path}[{i}]",
                                "pii_type": pii_type,
                                "value_masked": self._mask_finding(pii_value, pii_type)
                            })

        return findings

    def _mask_finding(self, value: str, pii_type: str) -> str:
        """Маскирование найденного PII для отчёта"""
        if len(value) <= 4:
            return "***"
        return f"{value[:2]}***{value[-2:]}"


# Пример использования
scanner = PIIScanner()

test_data = {
    "user": {
        "contact": "Email: john.doe@example.com, Phone: +1-555-123-4567",
        "ssn": "123-45-6789"
    },
    "notes": ["Customer SSN is 987-65-4321", "Regular note"]
}

findings = scanner.scan_dict(test_data)
# [
#   {"path": "user.contact", "pii_type": "email", "value_masked": "jo***om"},
#   {"path": "user.contact", "pii_type": "phone_us", "value_masked": "+1***67"},
#   {"path": "user.ssn", "pii_type": "ssn", "value_masked": "12***89"},
#   {"path": "notes[0]", "pii_type": "ssn", "value_masked": "98***21"}
# ]
```

---

## Чеклист защиты PII

### Организационные меры

- [ ] Проведена инвентаризация всех PII в системе
- [ ] Определены владельцы данных для каждой категории PII
- [ ] Разработана политика обработки PII
- [ ] Проведено обучение персонала
- [ ] Установлены процедуры реагирования на утечки

### Технические меры

- [ ] Шифрование PII при хранении
- [ ] Шифрование PII при передаче (TLS)
- [ ] Контроль доступа к PII (RBAC)
- [ ] Логирование доступа к PII
- [ ] Автоматическое удаление устаревших PII
- [ ] PII не присутствует в логах
- [ ] PII не присутствует в URL

### Для API разработчиков

```python
class PIIComplianceChecker:
    """Проверка API на соответствие требованиям PII"""

    def check_compliance(self, api_spec: dict) -> dict:
        return {
            "data_collection": {
                "minimization": self._checks_minimization(api_spec),
                "purpose_limitation": self._checks_purpose(api_spec),
                "consent_mechanism": self._has_consent(api_spec)
            },
            "data_storage": {
                "encryption_at_rest": self._checks_encryption(api_spec),
                "retention_policy": self._has_retention(api_spec),
                "access_controls": self._checks_access(api_spec)
            },
            "data_transmission": {
                "encryption_in_transit": self._checks_tls(api_spec),
                "no_pii_in_urls": self._no_pii_urls(api_spec),
                "response_filtering": self._checks_response_filter(api_spec)
            },
            "logging": {
                "pii_excluded": self._no_pii_logs(api_spec),
                "access_audited": self._has_audit(api_spec)
            },
            "user_rights": {
                "access_endpoint": self._has_access_endpoint(api_spec),
                "deletion_endpoint": self._has_deletion_endpoint(api_spec),
                "export_endpoint": self._has_export_endpoint(api_spec)
            }
        }
```

---

## Полезные ресурсы

- [NIST Privacy Framework](https://www.nist.gov/privacy-framework)
- [NIST SP 800-122: Guide to Protecting PII](https://csrc.nist.gov/publications/detail/sp/800-122/final)
- [ISO 27701: Privacy Information Management](https://www.iso.org/standard/71670.html)
- [OWASP Data Protection Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/User_Privacy_Protection_Cheat_Sheet.html)
- [ICO Guide to PII](https://ico.org.uk/for-organisations/guide-to-data-protection/)
