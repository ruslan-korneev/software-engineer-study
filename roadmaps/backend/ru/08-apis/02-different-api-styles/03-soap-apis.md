# SOAP APIs

## Что такое SOAP?

**SOAP (Simple Object Access Protocol)** — это протокол обмена структурированными сообщениями в распределённых вычислительных системах. Несмотря на слово "Simple" в названии, SOAP является довольно сложным и формальным протоколом.

SOAP был разработан в 1998 году и стал стандартом W3C. Он использует XML для форматирования сообщений и обычно работает поверх HTTP, хотя может использовать и другие протоколы (SMTP, TCP, JMS).

## Ключевые характеристики SOAP

- **Протокол, а не архитектурный стиль** — в отличие от REST
- **Строгая типизация** — все данные описаны в WSDL и XSD
- **Независимость от транспорта** — HTTP, SMTP, TCP и другие
- **Встроенная безопасность** — WS-Security стандарты
- **Надёжная доставка** — WS-ReliableMessaging
- **Транзакции** — WS-AtomicTransaction

## Структура SOAP-сообщения

SOAP-сообщение — это XML-документ со строгой структурой:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope
    xmlns:soap="http://www.w3.org/2003/05/soap-envelope"
    xmlns:ns="http://example.com/users">

    <!-- Заголовок (опционально) -->
    <soap:Header>
        <ns:Authentication>
            <ns:Token>abc123xyz</ns:Token>
        </ns:Authentication>
    </soap:Header>

    <!-- Тело (обязательно) -->
    <soap:Body>
        <ns:GetUserRequest>
            <ns:UserId>123</ns:UserId>
        </ns:GetUserRequest>
    </soap:Body>

</soap:Envelope>
```

### Компоненты сообщения

#### 1. Envelope (Конверт)
Корневой элемент, который определяет XML-документ как SOAP-сообщение.

```xml
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
    ...
</soap:Envelope>
```

#### 2. Header (Заголовок)
Опциональный элемент для метаданных: аутентификация, маршрутизация, транзакции.

```xml
<soap:Header>
    <wsse:Security xmlns:wsse="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-secext-1.0.xsd">
        <wsse:UsernameToken>
            <wsse:Username>user</wsse:Username>
            <wsse:Password>password</wsse:Password>
        </wsse:UsernameToken>
    </wsse:Security>
</soap:Header>
```

#### 3. Body (Тело)
Обязательный элемент, содержащий фактические данные запроса или ответа.

```xml
<soap:Body>
    <ns:CreateUserRequest>
        <ns:Name>John Doe</ns:Name>
        <ns:Email>john@example.com</ns:Email>
    </ns:CreateUserRequest>
</soap:Body>
```

#### 4. Fault (Ошибка)
Элемент для сообщений об ошибках внутри Body.

```xml
<soap:Body>
    <soap:Fault>
        <soap:Code>
            <soap:Value>soap:Sender</soap:Value>
        </soap:Code>
        <soap:Reason>
            <soap:Text xml:lang="en">User not found</soap:Text>
        </soap:Reason>
        <soap:Detail>
            <ns:ErrorCode>USER_404</ns:ErrorCode>
            <ns:UserId>999</ns:UserId>
        </soap:Detail>
    </soap:Fault>
</soap:Body>
```

## WSDL (Web Services Description Language)

**WSDL** — это XML-документ, описывающий веб-сервис: какие операции доступны, какие параметры принимают, какие данные возвращают.

### Структура WSDL

```xml
<?xml version="1.0" encoding="UTF-8"?>
<definitions name="UserService"
    targetNamespace="http://example.com/users"
    xmlns="http://schemas.xmlsoap.org/wsdl/"
    xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/"
    xmlns:tns="http://example.com/users"
    xmlns:xsd="http://www.w3.org/2001/XMLSchema">

    <!-- Типы данных -->
    <types>
        <xsd:schema targetNamespace="http://example.com/users">
            <xsd:element name="GetUserRequest">
                <xsd:complexType>
                    <xsd:sequence>
                        <xsd:element name="UserId" type="xsd:int"/>
                    </xsd:sequence>
                </xsd:complexType>
            </xsd:element>
            <xsd:element name="GetUserResponse">
                <xsd:complexType>
                    <xsd:sequence>
                        <xsd:element name="User" type="tns:UserType"/>
                    </xsd:sequence>
                </xsd:complexType>
            </xsd:element>
            <xsd:complexType name="UserType">
                <xsd:sequence>
                    <xsd:element name="Id" type="xsd:int"/>
                    <xsd:element name="Name" type="xsd:string"/>
                    <xsd:element name="Email" type="xsd:string"/>
                </xsd:sequence>
            </xsd:complexType>
        </xsd:schema>
    </types>

    <!-- Сообщения -->
    <message name="GetUserInput">
        <part name="parameters" element="tns:GetUserRequest"/>
    </message>
    <message name="GetUserOutput">
        <part name="parameters" element="tns:GetUserResponse"/>
    </message>

    <!-- Порт (интерфейс) -->
    <portType name="UserPortType">
        <operation name="GetUser">
            <input message="tns:GetUserInput"/>
            <output message="tns:GetUserOutput"/>
        </operation>
    </portType>

    <!-- Привязка к SOAP -->
    <binding name="UserBinding" type="tns:UserPortType">
        <soap:binding style="document"
            transport="http://schemas.xmlsoap.org/soap/http"/>
        <operation name="GetUser">
            <soap:operation soapAction="http://example.com/GetUser"/>
            <input>
                <soap:body use="literal"/>
            </input>
            <output>
                <soap:body use="literal"/>
            </output>
        </operation>
    </binding>

    <!-- Сервис -->
    <service name="UserService">
        <port name="UserPort" binding="tns:UserBinding">
            <soap:address location="http://example.com/soap/users"/>
        </port>
    </service>

</definitions>
```

### Компоненты WSDL

| Элемент | Описание |
|---------|----------|
| `types` | XSD-схемы для типов данных |
| `message` | Определение сообщений (входных и выходных) |
| `portType` | Набор операций (интерфейс сервиса) |
| `binding` | Связь операций с протоколом (SOAP) |
| `service` | Адрес endpoint'а сервиса |

## Практический пример: SOAP-сервис на Python

### Сервер (с использованием Spyne)

```python
from spyne import Application, Service, rpc
from spyne import Unicode, Integer, ComplexModel, Array
from spyne.protocol.soap import Soap11
from spyne.server.wsgi import WsgiApplication

# Модели данных
class User(ComplexModel):
    id = Integer
    name = Unicode
    email = Unicode

class CreateUserRequest(ComplexModel):
    name = Unicode
    email = Unicode

class UserListResponse(ComplexModel):
    users = Array(User)
    total = Integer

# Сервис
class UserService(Service):

    @rpc(Integer, _returns=User)
    def get_user(ctx, user_id):
        """Получить пользователя по ID"""
        # Имитация базы данных
        users_db = {
            1: User(id=1, name="John Doe", email="john@example.com"),
            2: User(id=2, name="Jane Smith", email="jane@example.com")
        }

        user = users_db.get(user_id)
        if not user:
            raise ValueError(f"User with id {user_id} not found")
        return user

    @rpc(_returns=UserListResponse)
    def get_all_users(ctx):
        """Получить всех пользователей"""
        users = [
            User(id=1, name="John Doe", email="john@example.com"),
            User(id=2, name="Jane Smith", email="jane@example.com")
        ]
        return UserListResponse(users=users, total=len(users))

    @rpc(CreateUserRequest, _returns=User)
    def create_user(ctx, request):
        """Создать нового пользователя"""
        # Генерация ID и создание пользователя
        new_user = User(
            id=3,
            name=request.name,
            email=request.email
        )
        return new_user

    @rpc(Integer, Unicode, _returns=User)
    def update_user_email(ctx, user_id, new_email):
        """Обновить email пользователя"""
        user = User(id=user_id, name="John Doe", email=new_email)
        return user

    @rpc(Integer, _returns=Unicode)
    def delete_user(ctx, user_id):
        """Удалить пользователя"""
        return f"User {user_id} deleted successfully"


# Создание приложения
application = Application(
    [UserService],
    tns='http://example.com/users',
    in_protocol=Soap11(validator='lxml'),
    out_protocol=Soap11()
)

wsgi_application = WsgiApplication(application)

if __name__ == '__main__':
    from wsgiref.simple_server import make_server
    server = make_server('0.0.0.0', 8000, wsgi_application)
    print("SOAP server running on http://localhost:8000")
    print("WSDL available at http://localhost:8000/?wsdl")
    server.serve_forever()
```

### Клиент (с использованием Zeep)

```python
from zeep import Client
from zeep.exceptions import Fault

# Подключение к сервису
client = Client('http://localhost:8000/?wsdl')

# Просмотр доступных операций
print("Available operations:")
for service in client.wsdl.services.values():
    for port in service.ports.values():
        for operation in port.binding._operations.values():
            print(f"  - {operation.name}")

# Вызов операций
try:
    # Получить пользователя
    user = client.service.get_user(user_id=1)
    print(f"User: {user.name} ({user.email})")

    # Получить всех пользователей
    response = client.service.get_all_users()
    print(f"Total users: {response.total}")
    for u in response.users:
        print(f"  - {u.id}: {u.name}")

    # Создать пользователя
    create_request = client.get_type('ns0:CreateUserRequest')(
        name="New User",
        email="new@example.com"
    )
    new_user = client.service.create_user(create_request)
    print(f"Created: {new_user.name}")

except Fault as e:
    print(f"SOAP Fault: {e.message}")
```

### Raw HTTP запрос к SOAP-сервису

```python
import requests

# SOAP-запрос вручную
soap_request = """<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/"
               xmlns:ns="http://example.com/users">
    <soap:Body>
        <ns:get_user>
            <ns:user_id>1</ns:user_id>
        </ns:get_user>
    </soap:Body>
</soap:Envelope>
"""

headers = {
    'Content-Type': 'text/xml; charset=utf-8',
    'SOAPAction': 'get_user'
}

response = requests.post(
    'http://localhost:8000',
    data=soap_request,
    headers=headers
)

print(response.text)
```

## WS-* Стандарты

SOAP поддерживает множество дополнительных стандартов (WS-*):

### WS-Security

Стандарт для обеспечения безопасности сообщений.

```xml
<soap:Header>
    <wsse:Security xmlns:wsse="http://docs.oasis-open.org/wss/2004/01/...">
        <!-- Username Token -->
        <wsse:UsernameToken>
            <wsse:Username>user</wsse:Username>
            <wsse:Password Type="PasswordDigest">hash...</wsse:Password>
            <wsse:Nonce>nonce...</wsse:Nonce>
            <wsu:Created>2024-01-01T00:00:00Z</wsu:Created>
        </wsse:UsernameToken>

        <!-- Или X.509 Certificate -->
        <wsse:BinarySecurityToken>certificate...</wsse:BinarySecurityToken>

        <!-- Цифровая подпись -->
        <ds:Signature>...</ds:Signature>

        <!-- Шифрование -->
        <xenc:EncryptedData>...</xenc:EncryptedData>
    </wsse:Security>
</soap:Header>
```

### WS-ReliableMessaging

Гарантированная доставка сообщений с подтверждением.

### WS-AtomicTransaction

Распределённые транзакции между несколькими сервисами.

### WS-Addressing

Стандартизированная адресация для маршрутизации сообщений.

```xml
<soap:Header>
    <wsa:To>http://example.com/service</wsa:To>
    <wsa:Action>http://example.com/GetUser</wsa:Action>
    <wsa:MessageID>uuid:12345-67890</wsa:MessageID>
    <wsa:ReplyTo>
        <wsa:Address>http://client.example.com/callback</wsa:Address>
    </wsa:ReplyTo>
</soap:Header>
```

## Сравнение с другими стилями API

### SOAP vs REST

| Критерий | SOAP | REST |
|----------|------|------|
| Формат данных | Только XML | JSON, XML, и др. |
| Протокол | SOAP (поверх HTTP, SMTP, TCP) | HTTP |
| Контракт | WSDL (строгий) | OpenAPI (опциональный) |
| Типизация | Строгая (XSD) | Опциональная |
| Производительность | Ниже (XML verbose) | Выше (JSON compact) |
| Безопасность | WS-Security | HTTPS, OAuth |
| Состояние | Может быть stateful | Stateless |
| Сложность | Высокая | Низкая |
| Tooling | Генерация клиентов из WSDL | Ручная разработка |

### SOAP vs gRPC

| Критерий | SOAP | gRPC |
|----------|------|------|
| Формат данных | XML | Protocol Buffers |
| Протокол | HTTP/1.1, SMTP | HTTP/2 |
| Streaming | Нет | Да (bidirectional) |
| Производительность | Низкая | Высокая |
| Контракт | WSDL | .proto файлы |
| Browser support | Да | Ограниченная |
| Enterprise features | Полные WS-* | Частичные |

## Когда использовать SOAP

**Подходит для:**
- **Enterprise-интеграции** — банки, страховые, государственные системы
- **Требования к безопасности** — WS-Security, подписи, шифрование
- **Распределённые транзакции** — WS-AtomicTransaction
- **Legacy-системы** — интеграция с существующими SOAP-сервисами
- **Строгие контракты** — когда важна формальная спецификация

**Не подходит для:**
- Мобильных приложений (большой overhead XML)
- Браузерных приложений (сложность работы с XML)
- Микросервисов (избыточная сложность)
- Высоконагруженных систем (низкая производительность)

## Best Practices

### 1. Документирование через WSDL

```python
# Spyne автоматически генерирует WSDL
# Добавляй документацию к методам
@rpc(Integer, _returns=User)
def get_user(ctx, user_id):
    """
    Получает пользователя по идентификатору.

    Args:
        user_id: Уникальный идентификатор пользователя

    Returns:
        User: Объект пользователя с полями id, name, email

    Raises:
        Fault: Если пользователь не найден
    """
    pass
```

### 2. Обработка ошибок

```python
from spyne import Fault

@rpc(Integer, _returns=User)
def get_user(ctx, user_id):
    user = users_db.get(user_id)
    if not user:
        raise Fault(
            faultcode='Client.NotFound',
            faultstring=f'User {user_id} not found',
            detail={'userId': user_id}
        )
    return user
```

### 3. Версионирование

```python
# Используй разные namespace для версий
application_v1 = Application(
    [UserServiceV1],
    tns='http://example.com/users/v1',
    ...
)

application_v2 = Application(
    [UserServiceV2],
    tns='http://example.com/users/v2',
    ...
)
```

### 4. Логирование SOAP-сообщений

```python
from zeep import Client
from zeep.plugins import HistoryPlugin

history = HistoryPlugin()
client = Client('http://localhost:8000/?wsdl', plugins=[history])

# Вызов
client.service.get_user(user_id=1)

# Логирование
print("Request:")
print(etree.tostring(history.last_sent['envelope'], pretty_print=True))
print("Response:")
print(etree.tostring(history.last_received['envelope'], pretty_print=True))
```

## Типичные ошибки

### 1. Игнорирование WSDL-валидации

```python
# Плохо: отключение валидации
client = Client('http://service/?wsdl', strict=False)

# Хорошо: использование валидации
from spyne.protocol.soap import Soap11
application = Application(
    [UserService],
    in_protocol=Soap11(validator='lxml'),  # Включена валидация
    ...
)
```

### 2. Неправильная обработка namespace

```xml
<!-- Плохо: отсутствует namespace -->
<GetUserRequest>
    <UserId>1</UserId>
</GetUserRequest>

<!-- Хорошо: правильный namespace -->
<ns:GetUserRequest xmlns:ns="http://example.com/users">
    <ns:UserId>1</ns:UserId>
</ns:GetUserRequest>
```

### 3. Жёсткая привязка к структуре XML

```python
# Плохо: ручной парсинг XML
import xml.etree.ElementTree as ET
root = ET.fromstring(response.text)
user_id = root.find('.//UserId').text

# Хорошо: использование библиотеки
from zeep import Client
client = Client(wsdl_url)
user = client.service.get_user(1)
print(user.id)  # Типизированный объект
```

### 4. Синхронные вызовы для всех операций

```python
# Для длительных операций используй асинхронность
import asyncio
from zeep import AsyncClient
from zeep.transports import AsyncTransport
from httpx import AsyncClient as HttpxClient

async def main():
    async with HttpxClient() as httpx_client:
        transport = AsyncTransport(client=httpx_client)
        client = AsyncClient('http://service/?wsdl', transport=transport)
        result = await client.service.long_operation()
```

## Инструменты для работы с SOAP

### Python
- **Zeep** — современный SOAP-клиент
- **Spyne** — фреймворк для создания SOAP-сервисов
- **suds-community** — альтернативный клиент

### Тестирование
- **SoapUI** — GUI для тестирования SOAP-сервисов
- **Postman** — поддержка SOAP-запросов
- **curl** — ручные запросы

### Мониторинг
- **Wireshark** — анализ SOAP-трафика
- **tcpdump** — захват пакетов

## Пример SOAP-запроса через curl

```bash
curl -X POST http://localhost:8000 \
  -H "Content-Type: text/xml; charset=utf-8" \
  -H "SOAPAction: get_user" \
  -d '<?xml version="1.0"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/"
               xmlns:ns="http://example.com/users">
  <soap:Body>
    <ns:get_user>
      <ns:user_id>1</ns:user_id>
    </ns:get_user>
  </soap:Body>
</soap:Envelope>'
```

## Дополнительные ресурсы

- [W3C SOAP Specification](https://www.w3.org/TR/soap/)
- [WSDL Specification](https://www.w3.org/TR/wsdl20/)
- [WS-Security](https://www.oasis-open.org/committees/wss/)
- [Zeep Documentation](https://docs.python-zeep.org/)
- [Spyne Documentation](http://spyne.io/)
