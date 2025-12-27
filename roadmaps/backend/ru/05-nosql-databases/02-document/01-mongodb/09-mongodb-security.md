# Безопасность MongoDB

Безопасность базы данных — критически важный аспект любого приложения. MongoDB предоставляет многоуровневую систему защиты, включающую аутентификацию, авторизацию, шифрование и аудит. В этом разделе рассмотрим все аспекты безопасности MongoDB.

## Аутентификация

Аутентификация — процесс проверки личности пользователя или приложения. MongoDB поддерживает несколько механизмов аутентификации.

### SCRAM (Salted Challenge Response Authentication Mechanism)

SCRAM — механизм аутентификации по умолчанию в MongoDB. Поддерживаются две версии:
- **SCRAM-SHA-1** — для совместимости со старыми версиями
- **SCRAM-SHA-256** — более безопасный, рекомендуется использовать

```javascript
// Создание пользователя с SCRAM-SHA-256
db.createUser({
  user: "appUser",
  pwd: "securePassword123!",
  roles: [{ role: "readWrite", db: "myapp" }],
  mechanisms: ["SCRAM-SHA-256"]  // Явно указываем механизм
})

// Подключение с аутентификацией
mongosh "mongodb://appUser:securePassword123!@localhost:27017/myapp?authMechanism=SCRAM-SHA-256"
```

Конфигурация в `mongod.conf`:
```yaml
security:
  authorization: enabled

setParameter:
  authenticationMechanisms: SCRAM-SHA-256
```

### x.509 Certificate Authentication

Аутентификация на основе клиентских сертификатов. Обеспечивает высокий уровень безопасности для межсервисной коммуникации.

```yaml
# mongod.conf
net:
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/mongodb/server.pem
    CAFile: /etc/mongodb/ca.pem
    allowConnectionsWithoutCertificates: false

security:
  clusterAuthMode: x509
```

```javascript
// Создание пользователя с x.509 аутентификацией
db.getSiblingDB("$external").runCommand({
  createUser: "CN=myClient,OU=Engineering,O=MyCompany,L=Moscow,ST=Moscow,C=RU",
  roles: [
    { role: "readWrite", db: "myapp" },
    { role: "clusterMonitor", db: "admin" }
  ]
})
```

Подключение с сертификатом:
```bash
mongosh --tls \
  --tlsCertificateKeyFile /path/to/client.pem \
  --tlsCAFile /path/to/ca.pem \
  --authenticationMechanism MONGODB-X509 \
  --authenticationDatabase '$external' \
  mongodb://localhost:27017
```

### LDAP Authentication (Enterprise)

Интеграция с корпоративными директориями (Active Directory, OpenLDAP).

```yaml
# mongod.conf
security:
  ldap:
    servers: "ldap.example.com"
    bind:
      method: simple
      queryUser: "cn=mongodb,ou=services,dc=example,dc=com"
      queryPassword: "ldap-password"
    userToDNMapping:
      '[
        {
          match: "(.+)",
          substitution: "uid={0},ou=users,dc=example,dc=com"
        }
      ]'
    authz:
      queryTemplate: "{USER}?memberOf?base"

setParameter:
  authenticationMechanisms: PLAIN
```

### Kerberos Authentication (Enterprise)

Интеграция с Kerberos для единого входа (SSO).

```yaml
# mongod.conf
security:
  authorization: enabled

setParameter:
  authenticationMechanisms: GSSAPI
```

```bash
# Подключение с Kerberos
kinit mongouser@EXAMPLE.COM
mongosh --authenticationMechanism GSSAPI \
  --authenticationDatabase '$external' \
  mongodb://localhost:27017
```

## Авторизация и роли

После аутентификации MongoDB использует систему ролей для определения прав доступа.

### Built-in Roles (Встроенные роли)

#### Database User Roles
```javascript
// read — чтение данных
db.grantRolesToUser("analyst", [{ role: "read", db: "analytics" }])

// readWrite — чтение и запись
db.grantRolesToUser("appUser", [{ role: "readWrite", db: "myapp" }])
```

#### Database Administration Roles
```javascript
// dbAdmin — администрирование конкретной БД
db.grantRolesToUser("dbManager", [{ role: "dbAdmin", db: "myapp" }])

// dbOwner — полный контроль над БД
db.grantRolesToUser("owner", [{ role: "dbOwner", db: "myapp" }])

// userAdmin — управление пользователями
db.grantRolesToUser("userManager", [{ role: "userAdmin", db: "myapp" }])
```

#### Cluster Administration Roles
```javascript
// clusterAdmin — полное администрирование кластера
// clusterManager — управление и мониторинг кластера
// clusterMonitor — только мониторинг
// hostManager — управление серверами

db.getSiblingDB("admin").grantRolesToUser("clusterOps", [
  { role: "clusterMonitor", db: "admin" }
])
```

#### Backup and Restoration Roles
```javascript
// backup — права для резервного копирования
// restore — права для восстановления

db.getSiblingDB("admin").createUser({
  user: "backupOperator",
  pwd: "backupPass123!",
  roles: [
    { role: "backup", db: "admin" },
    { role: "restore", db: "admin" }
  ]
})
```

#### All-Database Roles
```javascript
// readAnyDatabase — чтение любой БД
// readWriteAnyDatabase — чтение/запись любой БД
// dbAdminAnyDatabase — администрирование любой БД
// userAdminAnyDatabase — управление пользователями любой БД

db.getSiblingDB("admin").createUser({
  user: "superAdmin",
  pwd: "superSecure!",
  roles: [{ role: "root", db: "admin" }]  // Максимальные права
})
```

### Custom Roles (Пользовательские роли)

Создание ролей с точечными правами:

```javascript
// Роль для аналитика — только чтение определённых коллекций
db.createRole({
  role: "analyticsReader",
  privileges: [
    {
      resource: { db: "myapp", collection: "orders" },
      actions: ["find", "aggregate"]
    },
    {
      resource: { db: "myapp", collection: "products" },
      actions: ["find"]
    }
  ],
  roles: []  // Не наследует другие роли
})

// Роль для API с ограниченными правами записи
db.createRole({
  role: "apiWriter",
  privileges: [
    {
      resource: { db: "myapp", collection: "events" },
      actions: ["insert"]  // Только вставка, без обновления/удаления
    },
    {
      resource: { db: "myapp", collection: "users" },
      actions: ["find", "update"]  // Чтение и обновление, без удаления
    }
  ],
  roles: []
})

// Роль с наследованием
db.createRole({
  role: "seniorDeveloper",
  privileges: [
    {
      resource: { db: "myapp", collection: "" },  // Все коллекции
      actions: ["createIndex", "dropIndex"]
    }
  ],
  roles: [
    { role: "readWrite", db: "myapp" },
    { role: "read", db: "logs" }
  ]
})
```

Управление ролями:
```javascript
// Обновление роли
db.updateRole("analyticsReader", {
  privileges: [
    {
      resource: { db: "myapp", collection: "orders" },
      actions: ["find", "aggregate", "listIndexes"]
    }
  ]
})

// Добавление прав к роли
db.grantPrivilegesToRole("apiWriter", [
  {
    resource: { db: "myapp", collection: "sessions" },
    actions: ["insert", "remove"]
  }
])

// Просмотр роли
db.getRole("analyticsReader", { showPrivileges: true })

// Удаление роли
db.dropRole("oldRole")
```

## Role-Based Access Control (RBAC)

RBAC в MongoDB позволяет реализовать принцип наименьших привилегий.

### Принципы RBAC

```javascript
// 1. Принцип наименьших привилегий — давать минимально необходимые права
db.createRole({
  role: "orderProcessor",
  privileges: [
    {
      resource: { db: "shop", collection: "orders" },
      actions: ["find", "update"]
    }
  ],
  roles: []
})

// 2. Разделение обязанностей — разные роли для разных задач
db.createRole({
  role: "orderCreator",
  privileges: [
    { resource: { db: "shop", collection: "orders" }, actions: ["insert"] }
  ],
  roles: []
})

db.createRole({
  role: "orderApprover",
  privileges: [
    { resource: { db: "shop", collection: "orders" }, actions: ["find", "update"] }
  ],
  roles: []
})

// 3. Иерархия ролей
db.createRole({
  role: "orderManager",
  privileges: [],
  roles: [
    { role: "orderCreator", db: "shop" },
    { role: "orderApprover", db: "shop" },
    { role: "orderProcessor", db: "shop" }
  ]
})
```

### Аудит прав пользователей

```javascript
// Просмотр всех пользователей
db.getUsers()

// Детальная информация о пользователе
db.getUser("appUser", { showPrivileges: true, showCredentials: false })

// Все роли пользователя
db.runCommand({
  usersInfo: { user: "appUser", db: "myapp" },
  showPrivileges: true
})

// Проверка, имеет ли пользователь определённые права
db.runCommand({
  connectionStatus: 1,
  showPrivileges: true
})
```

## Шифрование

### Encryption in Transit (TLS/SSL)

Шифрование данных при передаче по сети:

```yaml
# mongod.conf — полная конфигурация TLS
net:
  port: 27017
  bindIp: 0.0.0.0
  tls:
    mode: requireTLS                           # Требовать TLS
    certificateKeyFile: /etc/mongodb/server.pem  # Сертификат сервера
    CAFile: /etc/mongodb/ca.pem                # CA сертификат
    allowConnectionsWithoutCertificates: false  # Требовать клиентский сертификат
    allowInvalidCertificates: false            # Не принимать невалидные сертификаты
    allowInvalidHostnames: false               # Проверять hostname
    disabledProtocols: TLS1_0,TLS1_1           # Отключить старые протоколы
```

Генерация сертификатов для тестирования:
```bash
# Создание CA
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 365 -key ca.key -out ca.pem \
  -subj "/CN=MongoDB-CA/O=MyCompany"

# Создание серверного сертификата
openssl genrsa -out server.key 4096
openssl req -new -key server.key -out server.csr \
  -subj "/CN=mongodb.example.com/O=MyCompany"
openssl x509 -req -days 365 -in server.csr -CA ca.pem -CAkey ca.key \
  -CAcreateserial -out server.crt

# Объединение ключа и сертификата
cat server.key server.crt > server.pem
chmod 600 server.pem
```

Подключение с TLS:
```bash
# mongosh с TLS
mongosh --tls \
  --tlsCAFile /path/to/ca.pem \
  --tlsCertificateKeyFile /path/to/client.pem \
  mongodb://localhost:27017

# Connection string
mongodb://user:password@host:27017/db?tls=true&tlsCAFile=/path/to/ca.pem
```

### Encryption at Rest (Enterprise)

Шифрование данных на диске:

```yaml
# mongod.conf — шифрование хранилища
security:
  enableEncryption: true
  encryptionCipherMode: AES256-CBC
  encryptionKeyFile: /etc/mongodb/encryption-key

# Альтернативно — использование KMIP (Key Management Interoperability Protocol)
security:
  enableEncryption: true
  kmip:
    serverName: kmip.example.com
    port: 5696
    clientCertificateFile: /etc/mongodb/kmip-client.pem
    serverCAFile: /etc/mongodb/kmip-ca.pem
```

Генерация ключа шифрования:
```bash
# Создание 256-битного ключа
openssl rand -base64 32 > /etc/mongodb/encryption-key
chmod 600 /etc/mongodb/encryption-key
chown mongodb:mongodb /etc/mongodb/encryption-key
```

### Client-Side Field Level Encryption (CSFLE)

Шифрование отдельных полей на стороне клиента:

```javascript
// Настройка CSFLE с автоматическим шифрованием
const { MongoClient, ClientEncryption } = require('mongodb');

// Ключ шифрования (в production использовать KMS)
const localMasterKey = crypto.randomBytes(96);

const kmsProviders = {
  local: {
    key: localMasterKey
  }
  // Или использовать AWS KMS, Azure Key Vault, GCP KMS
};

const keyVaultNamespace = "encryption.__keyVault";

const client = new MongoClient(uri, {
  autoEncryption: {
    keyVaultNamespace,
    kmsProviders,
    schemaMap: {
      "myapp.users": {
        bsonType: "object",
        properties: {
          ssn: {
            encrypt: {
              bsonType: "string",
              algorithm: "AEAD_AES_256_CBC_HMAC_SHA_512-Deterministic"
            }
          },
          creditCard: {
            encrypt: {
              bsonType: "string",
              algorithm: "AEAD_AES_256_CBC_HMAC_SHA_512-Random"
            }
          }
        }
      }
    }
  }
});

// Данные автоматически шифруются при записи и расшифровываются при чтении
await db.collection('users').insertOne({
  name: "John Doe",
  ssn: "123-45-6789",        // Будет зашифровано
  creditCard: "4111111111111111"  // Будет зашифровано
});
```

## Аудит (Enterprise)

Аудит позволяет отслеживать все операции в базе данных.

### Настройка аудита

```yaml
# mongod.conf
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.json
  filter: '{ atype: { $in: ["authenticate", "createUser", "dropUser", "createRole", "dropRole", "createDatabase", "dropDatabase", "createCollection", "dropCollection"] } }'

# Или вывод в syslog
auditLog:
  destination: syslog
  format: JSON
```

### Фильтрация аудит-событий

```yaml
# Аудит только операций аутентификации
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.json
  filter: '{ atype: "authenticate" }'

# Аудит операций конкретного пользователя
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.json
  filter: '{ "param.user": "admin" }'

# Аудит операций записи на конкретной коллекции
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.json
  filter: '{
    atype: { $in: ["insert", "update", "delete"] },
    "param.ns": "myapp.sensitive_data"
  }'
```

### Формат аудит-логов

```json
{
  "atype": "authenticate",
  "ts": { "$date": "2024-01-15T10:30:00.000Z" },
  "local": { "ip": "127.0.0.1", "port": 27017 },
  "remote": { "ip": "192.168.1.100", "port": 45678 },
  "users": [{ "user": "appUser", "db": "myapp" }],
  "roles": [{ "role": "readWrite", "db": "myapp" }],
  "param": {
    "user": "appUser",
    "db": "myapp",
    "mechanism": "SCRAM-SHA-256"
  },
  "result": 0
}
```

### Runtime аудит

```javascript
// Изменение фильтра аудита в runtime
db.adminCommand({
  setAuditConfig: 1,
  filter: { atype: { $in: ["authenticate", "createUser"] } },
  auditAuthorizationSuccess: true
})

// Получение текущей конфигурации
db.adminCommand({ getAuditConfig: 1 })
```

## Network Security

### IP Whitelist (Binding)

```yaml
# mongod.conf — ограничение по IP
net:
  port: 27017
  bindIp: 127.0.0.1,192.168.1.10  # Только локальный и конкретный IP
  bindIpAll: false                 # НЕ слушать на всех интерфейсах
```

### Firewall Rules

```bash
# iptables — разрешить только с определённых IP
iptables -A INPUT -p tcp --dport 27017 -s 192.168.1.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 27017 -j DROP

# ufw
ufw allow from 192.168.1.0/24 to any port 27017
ufw deny 27017
```

### VPC и Private Networks

При развёртывании в облаке:

```yaml
# MongoDB Atlas — настройка IP Access List через API
# Или использование VPC Peering / Private Link

# AWS VPC Peering
# 1. Создать VPC peering connection
# 2. Добавить маршруты в route tables
# 3. Настроить security groups

# Пример security group для MongoDB
# Inbound rules:
#   - Type: Custom TCP, Port: 27017, Source: 10.0.0.0/16 (VPC CIDR)
# Outbound rules:
#   - Type: All traffic, Destination: 0.0.0.0/0
```

### Network Compression

```yaml
# mongod.conf — сжатие сетевого трафика
net:
  compression:
    compressors: snappy,zstd,zlib
```

## Безопасная конфигурация

### Production-ready mongod.conf

```yaml
# Полный пример безопасной конфигурации
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  logRotate: reopen

storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 2

net:
  port: 27017
  bindIp: 127.0.0.1,10.0.1.5
  maxIncomingConnections: 1000
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/mongodb/server.pem
    CAFile: /etc/mongodb/ca.pem
    allowConnectionsWithoutCertificates: false
    disabledProtocols: TLS1_0,TLS1_1

security:
  authorization: enabled
  javascriptEnabled: false  # Отключить JS если не используется

setParameter:
  authenticationMechanisms: SCRAM-SHA-256
  enableLocalhostAuthBypass: false  # Отключить bypass для localhost

processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/mongod.pid

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
```

### Отключение опасных функций

```javascript
// Отключение server-side JavaScript
db.adminCommand({ setParameter: 1, javascriptEnabled: false })

// Проверка параметров безопасности
db.adminCommand({ getParameter: "*" })
```

### Безопасные права на файлы

```bash
# Права доступа к файлам MongoDB
chown -R mongodb:mongodb /var/lib/mongodb
chown -R mongodb:mongodb /var/log/mongodb
chmod 700 /var/lib/mongodb
chmod 755 /var/log/mongodb
chmod 600 /etc/mongodb/mongod.conf
chmod 600 /etc/mongodb/*.pem
```

## Защита от инъекций

### NoSQL Injection

MongoDB подвержена NoSQL-инъекциям, если не валидировать входные данные.

```javascript
// УЯЗВИМЫЙ код — прямая подстановка пользовательского ввода
app.post('/login', async (req, res) => {
  const { username, password } = req.body;

  // Опасно! Атакующий может отправить { "$gt": "" } как password
  const user = await db.collection('users').findOne({
    username: username,
    password: password  // { "$gt": "" } вернёт любого пользователя
  });
});

// БЕЗОПАСНЫЙ код — валидация типов
app.post('/login', async (req, res) => {
  const { username, password } = req.body;

  // Проверяем, что это строки
  if (typeof username !== 'string' || typeof password !== 'string') {
    return res.status(400).json({ error: 'Invalid input' });
  }

  // Хешируем пароль (никогда не храним в открытом виде!)
  const hashedPassword = await bcrypt.hash(password, 10);

  const user = await db.collection('users').findOne({
    username: username,
    password: hashedPassword
  });
});
```

### Использование Schema Validation

```javascript
// Валидация на уровне коллекции
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["username", "email", "password"],
      properties: {
        username: {
          bsonType: "string",
          minLength: 3,
          maxLength: 50,
          pattern: "^[a-zA-Z0-9_]+$"
        },
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
        },
        password: {
          bsonType: "string",
          minLength: 60,  // bcrypt hash length
          maxLength: 60
        },
        role: {
          enum: ["user", "admin", "moderator"]
        }
      },
      additionalProperties: false  // Запретить дополнительные поля
    }
  },
  validationLevel: "strict",
  validationAction: "error"
});
```

### Параметризованные запросы

```javascript
// Используйте параметризованные запросы вместо конкатенации строк
const mongoose = require('mongoose');

// Модель с валидацией
const UserSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    match: /^[a-zA-Z0-9_]+$/,
    minlength: 3,
    maxlength: 50
  },
  email: {
    type: String,
    required: true,
    match: /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/
  }
});

// Безопасный поиск
async function findUser(username) {
  // Mongoose автоматически экранирует спецсимволы
  return User.findOne({ username: username });
}
```

### Санитизация входных данных

```javascript
const mongoSanitize = require('express-mongo-sanitize');

// Middleware для удаления операторов MongoDB из входных данных
app.use(mongoSanitize());

// Или ручная санитизация
function sanitize(obj) {
  if (obj instanceof Object) {
    for (const key in obj) {
      if (/^\$/.test(key)) {
        delete obj[key];
      } else {
        sanitize(obj[key]);
      }
    }
  }
  return obj;
}

app.post('/search', (req, res) => {
  const query = sanitize(req.body.query);
  // ...
});
```

## Security Checklist и Best Practices

### Pre-deployment Checklist

```markdown
## Аутентификация
- [ ] Включена аутентификация (security.authorization: enabled)
- [ ] Используется SCRAM-SHA-256 (не SCRAM-SHA-1)
- [ ] Отключен localhost auth bypass
- [ ] Все пользователи имеют сильные пароли
- [ ] Удалён/отключён пользователь по умолчанию

## Авторизация
- [ ] Применён принцип наименьших привилегий
- [ ] Созданы отдельные пользователи для разных приложений
- [ ] Роль root используется только для начальной настройки
- [ ] Регулярный аудит прав пользователей

## Сеть
- [ ] MongoDB не слушает на 0.0.0.0
- [ ] Настроен firewall/security groups
- [ ] Используется TLS для всех соединений
- [ ] Отключены старые версии TLS (1.0, 1.1)
- [ ] MongoDB работает в приватной сети/VPC

## Шифрование
- [ ] TLS для соединений (net.tls.mode: requireTLS)
- [ ] Шифрование at-rest (Enterprise) или LUKS/dm-crypt
- [ ] Ключи шифрования хранятся безопасно (KMS)
- [ ] CSFLE для чувствительных данных

## Операционная безопасность
- [ ] MongoDB запущен от непривилегированного пользователя
- [ ] Ограничены права на файлы конфигурации
- [ ] Настроен аудит (Enterprise)
- [ ] Логи отправляются в SIEM
- [ ] Настроены алерты безопасности

## Приложение
- [ ] Валидация всех входных данных
- [ ] Защита от NoSQL-инъекций
- [ ] Connection string не в коде (используйте env vars)
- [ ] Пароли не логируются
```

### Команды для проверки безопасности

```javascript
// Проверка статуса аутентификации
db.adminCommand({ getCmdLineOpts: 1 })

// Список всех пользователей
db.getSiblingDB("admin").getUsers()

// Проверка прав текущего пользователя
db.runCommand({ connectionStatus: 1, showPrivileges: true })

// Проверка параметров TLS
db.adminCommand({ getParameter: 1, tlsMode: 1 })

// Проверка аудита
db.adminCommand({ getAuditConfig: 1 })

// Поиск пользователей со слишком широкими правами
db.getSiblingDB("admin").getUsers().forEach(user => {
  user.roles.forEach(role => {
    if (role.role === "root" || role.role === "dbOwner") {
      print(`WARNING: User ${user.user} has privileged role: ${role.role}`);
    }
  });
});
```

### Автоматизация проверок безопасности

```bash
#!/bin/bash
# security-check.sh — скрипт проверки безопасности MongoDB

MONGO_URI="mongodb://admin:password@localhost:27017/admin"

echo "=== MongoDB Security Check ==="

# Проверка аутентификации
echo -n "Authentication enabled: "
mongosh $MONGO_URI --quiet --eval "db.adminCommand({getCmdLineOpts:1}).parsed.security.authorization" 2>/dev/null || echo "FAILED"

# Проверка TLS
echo -n "TLS mode: "
mongosh $MONGO_URI --quiet --eval "db.adminCommand({getParameter:1, tlsMode:1}).tlsMode" 2>/dev/null || echo "FAILED"

# Проверка binding
echo -n "Bind IP: "
mongosh $MONGO_URI --quiet --eval "db.adminCommand({getCmdLineOpts:1}).parsed.net.bindIp" 2>/dev/null || echo "FAILED"

# Проверка JavaScript
echo -n "JavaScript enabled: "
mongosh $MONGO_URI --quiet --eval "db.adminCommand({getParameter:1, javascriptEnabled:1}).javascriptEnabled" 2>/dev/null || echo "FAILED"

echo "=== Check Complete ==="
```

### Регулярные задачи безопасности

```javascript
// Ротация паролей
db.changeUserPassword("appUser", "newSecurePassword!");

// Отзыв неиспользуемых прав
db.revokeRolesFromUser("oldUser", [{ role: "readWrite", db: "oldDb" }]);

// Удаление неактивных пользователей
db.dropUser("inactiveUser");

// Обновление роли
db.updateRole("customRole", {
  privileges: [/* обновлённый список прав */],
  roles: []
});
```

## Заключение

Безопасность MongoDB требует комплексного подхода:

1. **Defense in Depth** — многоуровневая защита (сеть, аутентификация, авторизация, шифрование)
2. **Принцип наименьших привилегий** — давать только необходимые права
3. **Регулярный аудит** — проверять права и активность пользователей
4. **Мониторинг** — отслеживать подозрительную активность
5. **Обновления** — своевременно обновлять MongoDB для получения патчей безопасности

Ключевые моменты:
- Всегда включайте аутентификацию в production
- Используйте TLS для всех соединений
- Создавайте отдельных пользователей для каждого приложения
- Валидируйте входные данные для защиты от инъекций
- Регулярно проводите аудит безопасности
