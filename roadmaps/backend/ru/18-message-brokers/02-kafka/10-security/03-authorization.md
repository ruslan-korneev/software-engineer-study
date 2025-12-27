# Авторизация в Apache Kafka

## Описание

Авторизация в Kafka — это механизм контроля доступа, определяющий, какие операции могут выполнять аутентифицированные пользователи над ресурсами кластера. После успешной аутентификации Kafka проверяет, имеет ли пользователь право на запрашиваемую операцию.

Kafka использует систему Access Control Lists (ACLs) — списков контроля доступа, которые связывают принципалов (пользователей) с разрешениями на определённые ресурсы. Начиная с Kafka 2.4, также поддерживается pluggable авторизация, позволяющая интегрировать внешние системы управления доступом.

## Ключевые концепции

### Компоненты авторизации

```
┌─────────────────────────────────────────────────────────────────────┐
│                    МОДЕЛЬ АВТОРИЗАЦИИ KAFKA                         │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│  PRINCIPAL  │ ──── │  OPERATION  │ ──── │  RESOURCE   │
│ (Кто)       │      │ (Что делает)│      │ (С чем)     │
└─────────────┘      └─────────────┘      └─────────────┘
      │                    │                    │
      ▼                    ▼                    ▼
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│ User:admin  │      │ Read        │      │ Topic:orders│
│ User:app    │      │ Write       │      │ Group:*     │
│ Group:dev   │      │ Create      │      │ Cluster     │
└─────────────┘      │ Delete      │      │ TransactionalId│
                     │ Alter       │      └─────────────┘
                     │ Describe    │
                     │ All         │
                     └─────────────┘

                  ┌─────────────────┐
                  │   PERMISSION    │
                  │ (Allow / Deny)  │
                  └─────────────────┘
```

### Типы ресурсов (Resource Types)

| Ресурс | Описание | Примеры операций |
|--------|----------|------------------|
| **Topic** | Топик Kafka | Read, Write, Create, Delete, Describe, Alter |
| **Group** | Consumer Group | Read, Describe, Delete |
| **Cluster** | Кластер Kafka | Create, Alter, Describe, ClusterAction |
| **TransactionalId** | ID транзакции | Write, Describe |
| **DelegationToken** | Делегированный токен | Describe |

### Операции (Operations)

```
┌──────────────────────────────────────────────────────────────────┐
│                      ОПЕРАЦИИ И РЕСУРСЫ                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  READ           WRITE          CREATE         DELETE             │
│  ────           ─────          ──────         ──────             │
│  • Topic        • Topic        • Topic        • Topic            │
│  • Group        • TransactionalId • Group     • Group            │
│                 • Cluster      • Cluster      • Cluster          │
│                                • DelegationToken                 │
│                                                                  │
│  DESCRIBE       ALTER          CLUSTER_ACTION    ALL             │
│  ────────       ─────          ──────────────    ───             │
│  • Topic        • Topic        • Cluster         • Все ресурсы   │
│  • Group        • Cluster                                        │
│  • Cluster      • Group                                          │
│  • TransactionalId                                               │
│  • DelegationToken                                               │
│                                                                  │
│  IDEMPOTENT_WRITE                                                │
│  ────────────────                                                │
│  • Cluster (для идемпотентных producers)                         │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Паттерны имён ресурсов (Resource Patterns)

| Паттерн | Описание | Пример |
|---------|----------|--------|
| **LITERAL** | Точное совпадение | `--topic orders` — только топик "orders" |
| **PREFIXED** | Префиксное совпадение | `--topic orders --resource-pattern-type prefixed` — orders, orders-v2, orders-dlq |
| **ANY** | Любой ресурс (только для describe) | Используется в административных запросах |

## Примеры конфигурации

### Включение авторизации на брокере

```properties
# server.properties

# Включение авторизатора
authorizer.class.name=kafka.security.authorizer.AclAuthorizer

# Для KRaft (без ZooKeeper)
# authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer

# Суперпользователи (обходят ACL проверки)
super.users=User:admin;User:kafka-broker

# Разрешить операции если ACL не найден (ОПАСНО в production!)
# По умолчанию false - запрещает всё, что явно не разрешено
allow.everyone.if.no.acl.found=false

# Логирование авторизации
# log4j.logger.kafka.authorizer.logger=DEBUG
```

### Управление ACLs через kafka-acls.sh

```bash
# Формат команды
kafka-acls.sh --bootstrap-server <host:port> \
    --command-config <admin.properties> \
    --<action> \
    --<principal> \
    --<operation> \
    --<resource>

# Файл admin.properties для аутентификации
# admin.properties
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
    username="admin" \
    password="admin-secret";
ssl.truststore.location=/path/to/truststore.jks
ssl.truststore.password=truststore-password
```

### Примеры создания ACLs

```bash
# ═══════════════════════════════════════════════════════════════════
#                        PRODUCER ACLs
# ═══════════════════════════════════════════════════════════════════

# Разрешить producer запись в конкретный топик
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --add \
    --allow-principal User:order-service \
    --operation Write \
    --topic orders

# Разрешить producer запись во все топики с префиксом
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --add \
    --allow-principal User:order-service \
    --operation Write \
    --topic orders \
    --resource-pattern-type prefixed

# Разрешить Describe для получения метаданных топика
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --add \
    --allow-principal User:order-service \
    --operation Describe \
    --topic orders

# ═══════════════════════════════════════════════════════════════════
#                        CONSUMER ACLs
# ═══════════════════════════════════════════════════════════════════

# Разрешить consumer читать из топика
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --add \
    --allow-principal User:analytics-service \
    --operation Read \
    --topic orders

# Разрешить consumer использовать consumer group
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --add \
    --allow-principal User:analytics-service \
    --operation Read \
    --group analytics-group

# Полный набор ACLs для consumer
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --add \
    --allow-principal User:analytics-service \
    --operation Read \
    --operation Describe \
    --topic orders

kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --add \
    --allow-principal User:analytics-service \
    --operation Read \
    --operation Describe \
    --group analytics-group

# ═══════════════════════════════════════════════════════════════════
#                    TRANSACTIONAL PRODUCER ACLs
# ═══════════════════════════════════════════════════════════════════

# Для транзакционного producer
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --add \
    --allow-principal User:payment-service \
    --operation Write \
    --operation Describe \
    --transactional-id payment-tx

# Разрешить IdempotentWrite на уровне кластера
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --add \
    --allow-principal User:payment-service \
    --operation IdempotentWrite \
    --cluster

# ═══════════════════════════════════════════════════════════════════
#                    ADMIN / CLUSTER ACLs
# ═══════════════════════════════════════════════════════════════════

# Разрешить создание топиков
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --add \
    --allow-principal User:admin \
    --operation Create \
    --cluster

# Разрешить изменение конфигурации топиков
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --add \
    --allow-principal User:admin \
    --operation Alter \
    --operation AlterConfigs \
    --topic orders

# Разрешить удаление топиков
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --add \
    --allow-principal User:admin \
    --operation Delete \
    --topic orders
```

### Просмотр и удаление ACLs

```bash
# Просмотр всех ACLs
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --list

# Просмотр ACLs для конкретного топика
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --list \
    --topic orders

# Просмотр ACLs для конкретного пользователя
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --list \
    --principal User:order-service

# Удаление ACL
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --remove \
    --allow-principal User:order-service \
    --operation Write \
    --topic orders

# Удаление всех ACLs для топика
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --remove \
    --topic orders
```

### Deny ACLs (Запрещающие правила)

```bash
# Запретить конкретному пользователю доступ (имеет приоритет над Allow)
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --add \
    --deny-principal User:blocked-user \
    --operation All \
    --topic sensitive-data

# Запретить доступ с определённого хоста
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --add \
    --deny-principal User:* \
    --deny-host 192.168.1.100 \
    --operation All \
    --topic orders
```

### Ограничение по хостам

```bash
# Разрешить доступ только с определённых хостов
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --add \
    --allow-principal User:order-service \
    --allow-host 10.0.0.* \
    --operation Write \
    --topic orders

# Разрешить доступ с подсети
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --add \
    --allow-principal User:analytics-service \
    --allow-host 192.168.1.0/24 \
    --operation Read \
    --topic orders
```

## Программное управление ACLs

### Java Admin Client

```java
import org.apache.kafka.clients.admin.*;
import org.apache.kafka.common.acl.*;
import org.apache.kafka.common.resource.*;

import java.util.*;
import java.util.concurrent.ExecutionException;

public class KafkaAclManager {

    private final AdminClient adminClient;

    public KafkaAclManager(Properties props) {
        this.adminClient = AdminClient.create(props);
    }

    /**
     * Создание ACL для producer
     */
    public void createProducerAcl(String principal, String topicName)
            throws ExecutionException, InterruptedException {

        // Ресурс - топик
        ResourcePattern resource = new ResourcePattern(
            ResourceType.TOPIC,
            topicName,
            PatternType.LITERAL
        );

        // ACL Entry - разрешение на Write
        AccessControlEntry writeEntry = new AccessControlEntry(
            "User:" + principal,
            "*",  // любой хост
            AclOperation.WRITE,
            AclPermissionType.ALLOW
        );

        // ACL Entry - разрешение на Describe
        AccessControlEntry describeEntry = new AccessControlEntry(
            "User:" + principal,
            "*",
            AclOperation.DESCRIBE,
            AclPermissionType.ALLOW
        );

        // Создание ACL bindings
        Collection<AclBinding> aclBindings = Arrays.asList(
            new AclBinding(resource, writeEntry),
            new AclBinding(resource, describeEntry)
        );

        // Применение ACLs
        CreateAclsResult result = adminClient.createAcls(aclBindings);
        result.all().get();

        System.out.println("Producer ACLs created for " + principal);
    }

    /**
     * Создание ACL для consumer
     */
    public void createConsumerAcl(String principal, String topicName, String groupId)
            throws ExecutionException, InterruptedException {

        List<AclBinding> bindings = new ArrayList<>();

        // ACL для топика
        ResourcePattern topicResource = new ResourcePattern(
            ResourceType.TOPIC, topicName, PatternType.LITERAL
        );

        bindings.add(new AclBinding(topicResource, new AccessControlEntry(
            "User:" + principal, "*", AclOperation.READ, AclPermissionType.ALLOW
        )));

        bindings.add(new AclBinding(topicResource, new AccessControlEntry(
            "User:" + principal, "*", AclOperation.DESCRIBE, AclPermissionType.ALLOW
        )));

        // ACL для consumer group
        ResourcePattern groupResource = new ResourcePattern(
            ResourceType.GROUP, groupId, PatternType.LITERAL
        );

        bindings.add(new AclBinding(groupResource, new AccessControlEntry(
            "User:" + principal, "*", AclOperation.READ, AclPermissionType.ALLOW
        )));

        bindings.add(new AclBinding(groupResource, new AccessControlEntry(
            "User:" + principal, "*", AclOperation.DESCRIBE, AclPermissionType.ALLOW
        )));

        adminClient.createAcls(bindings).all().get();
        System.out.println("Consumer ACLs created for " + principal);
    }

    /**
     * Создание ACL с префиксным паттерном
     */
    public void createPrefixedAcl(String principal, String topicPrefix)
            throws ExecutionException, InterruptedException {

        ResourcePattern resource = new ResourcePattern(
            ResourceType.TOPIC,
            topicPrefix,
            PatternType.PREFIXED  // Префиксный паттерн
        );

        Collection<AclBinding> bindings = Arrays.asList(
            new AclBinding(resource, new AccessControlEntry(
                "User:" + principal, "*", AclOperation.READ, AclPermissionType.ALLOW
            )),
            new AclBinding(resource, new AccessControlEntry(
                "User:" + principal, "*", AclOperation.WRITE, AclPermissionType.ALLOW
            ))
        );

        adminClient.createAcls(bindings).all().get();
    }

    /**
     * Получение списка ACLs
     */
    public void listAcls() throws ExecutionException, InterruptedException {
        DescribeAclsResult result = adminClient.describeAcls(AclBindingFilter.ANY);
        Collection<AclBinding> acls = result.values().get();

        for (AclBinding acl : acls) {
            System.out.printf("Resource: %s, Pattern: %s, Principal: %s, Operation: %s, Permission: %s%n",
                acl.pattern().name(),
                acl.pattern().patternType(),
                acl.entry().principal(),
                acl.entry().operation(),
                acl.entry().permissionType()
            );
        }
    }

    /**
     * Удаление ACLs для пользователя
     */
    public void deleteUserAcls(String principal)
            throws ExecutionException, InterruptedException {

        AclBindingFilter filter = new AclBindingFilter(
            ResourcePatternFilter.ANY,
            new AccessControlEntryFilter(
                "User:" + principal,
                null,
                AclOperation.ANY,
                AclPermissionType.ANY
            )
        );

        DeleteAclsResult result = adminClient.deleteAcls(Collections.singleton(filter));
        result.all().get();

        System.out.println("ACLs deleted for " + principal);
    }

    public void close() {
        adminClient.close();
    }
}
```

### Python пример

```python
from confluent_kafka.admin import (
    AdminClient,
    AclBinding,
    AclBindingFilter,
    ResourceType,
    ResourcePatternType,
    AclOperation,
    AclPermissionType
)

class KafkaAclManager:
    def __init__(self, config):
        self.admin = AdminClient(config)

    def create_producer_acl(self, principal: str, topic: str):
        """Создание ACL для producer"""
        acls = [
            AclBinding(
                restype=ResourceType.TOPIC,
                name=topic,
                resource_pattern_type=ResourcePatternType.LITERAL,
                principal=f"User:{principal}",
                host="*",
                operation=AclOperation.WRITE,
                permission_type=AclPermissionType.ALLOW
            ),
            AclBinding(
                restype=ResourceType.TOPIC,
                name=topic,
                resource_pattern_type=ResourcePatternType.LITERAL,
                principal=f"User:{principal}",
                host="*",
                operation=AclOperation.DESCRIBE,
                permission_type=AclPermissionType.ALLOW
            )
        ]

        futures = self.admin.create_acls(acls)
        for acl, future in futures.items():
            try:
                future.result()
                print(f"Created ACL: {acl}")
            except Exception as e:
                print(f"Failed to create ACL {acl}: {e}")

    def create_consumer_acl(self, principal: str, topic: str, group: str):
        """Создание ACL для consumer"""
        acls = [
            # Topic ACLs
            AclBinding(
                restype=ResourceType.TOPIC,
                name=topic,
                resource_pattern_type=ResourcePatternType.LITERAL,
                principal=f"User:{principal}",
                host="*",
                operation=AclOperation.READ,
                permission_type=AclPermissionType.ALLOW
            ),
            # Group ACLs
            AclBinding(
                restype=ResourceType.GROUP,
                name=group,
                resource_pattern_type=ResourcePatternType.LITERAL,
                principal=f"User:{principal}",
                host="*",
                operation=AclOperation.READ,
                permission_type=AclPermissionType.ALLOW
            )
        ]

        futures = self.admin.create_acls(acls)
        for acl, future in futures.items():
            try:
                future.result()
                print(f"Created ACL: {acl}")
            except Exception as e:
                print(f"Failed to create ACL {acl}: {e}")

    def list_acls(self, principal: str = None):
        """Получение списка ACLs"""
        filter = AclBindingFilter(
            restype=ResourceType.ANY,
            name=None,
            resource_pattern_type=ResourcePatternType.ANY,
            principal=f"User:{principal}" if principal else None,
            host=None,
            operation=AclOperation.ANY,
            permission_type=AclPermissionType.ANY
        )

        futures = self.admin.describe_acls(filter)
        acls = futures.result()

        for acl in acls:
            print(f"Resource: {acl.restype.name}:{acl.name}, "
                  f"Principal: {acl.principal}, "
                  f"Operation: {acl.operation.name}, "
                  f"Permission: {acl.permission_type.name}")

        return acls

    def delete_acls(self, principal: str):
        """Удаление всех ACLs пользователя"""
        filter = AclBindingFilter(
            restype=ResourceType.ANY,
            name=None,
            resource_pattern_type=ResourcePatternType.ANY,
            principal=f"User:{principal}",
            host=None,
            operation=AclOperation.ANY,
            permission_type=AclPermissionType.ANY
        )

        futures = self.admin.delete_acls([filter])
        for filter, future in futures.items():
            try:
                result = future.result()
                print(f"Deleted {len(result)} ACLs for {principal}")
            except Exception as e:
                print(f"Failed to delete ACLs: {e}")


# Использование
config = {
    'bootstrap.servers': 'kafka:9094',
    'security.protocol': 'SASL_SSL',
    'sasl.mechanism': 'SCRAM-SHA-512',
    'sasl.username': 'admin',
    'sasl.password': 'admin-secret',
    'ssl.ca.location': '/path/to/ca-cert.pem'
}

manager = KafkaAclManager(config)
manager.create_producer_acl('order-service', 'orders')
manager.create_consumer_acl('analytics-service', 'orders', 'analytics-group')
manager.list_acls()
```

## Best Practices

### 1. Принцип минимальных привилегий

```bash
# ПЛОХО: Давать все права
kafka-acls.sh --add --allow-principal User:app --operation All --topic '*'

# ХОРОШО: Только необходимые права
kafka-acls.sh --add --allow-principal User:order-producer \
    --operation Write --operation Describe \
    --topic orders

kafka-acls.sh --add --allow-principal User:order-consumer \
    --operation Read --operation Describe \
    --topic orders \
    --group order-processors
```

### 2. Использование префиксных паттернов

```bash
# Структурированные имена топиков
# team.service.entity.version
# orders.payment.transactions.v1
# orders.payment.refunds.v1

# ACL для всех топиков команды
kafka-acls.sh --add --allow-principal User:payment-team \
    --operation All \
    --topic orders.payment \
    --resource-pattern-type prefixed
```

### 3. Разделение по средам

```bash
# Development
kafka-acls.sh --add --allow-principal User:dev-team \
    --operation All \
    --topic dev- \
    --resource-pattern-type prefixed

# Staging
kafka-acls.sh --add --allow-principal User:qa-team \
    --operation Read --operation Describe \
    --topic staging- \
    --resource-pattern-type prefixed

# Production - строгие ACLs
kafka-acls.sh --add --allow-principal User:prod-order-service \
    --operation Write \
    --topic prod-orders
```

### 4. Аудит ACLs

```bash
#!/bin/bash
# audit-acls.sh - скрипт аудита ACLs

echo "=== Kafka ACL Audit Report ==="
echo "Date: $(date)"
echo

# Все ACLs
echo "=== All ACLs ==="
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --list

# Проверка wildcard ACLs (потенциально опасные)
echo
echo "=== Wildcard ACLs (Review Required) ==="
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --list | grep -E "(topic=\*|group=\*|operation=All)"

# Проверка Deny ACLs
echo
echo "=== Deny ACLs ==="
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --list | grep "DENY"
```

### 5. Автоматизация управления ACLs

```yaml
# acl-config.yaml - декларативная конфигурация
services:
  order-service:
    principal: User:order-service
    topics:
      - name: orders
        operations: [Write, Describe]
      - name: order-events
        operations: [Write, Describe]
    groups: []

  analytics-service:
    principal: User:analytics-service
    topics:
      - name: orders
        operations: [Read, Describe]
      - name: order-events
        operations: [Read, Describe]
    groups:
      - name: analytics-processors
        operations: [Read, Describe]

  payment-service:
    principal: User:payment-service
    topics:
      - name: payments
        operations: [Write, Describe]
        pattern_type: prefixed
    transactional_ids:
      - name: payment-tx
        operations: [Write, Describe]
    cluster:
      operations: [IdempotentWrite]
```

```python
# apply-acls.py - применение декларативной конфигурации
import yaml
from kafka_acl_manager import KafkaAclManager

def apply_acl_config(config_file: str, manager: KafkaAclManager):
    with open(config_file) as f:
        config = yaml.safe_load(f)

    for service_name, service_config in config['services'].items():
        principal = service_config['principal'].replace('User:', '')

        # Topic ACLs
        for topic in service_config.get('topics', []):
            for operation in topic['operations']:
                manager.create_topic_acl(
                    principal=principal,
                    topic=topic['name'],
                    operation=operation,
                    pattern_type=topic.get('pattern_type', 'literal')
                )

        # Group ACLs
        for group in service_config.get('groups', []):
            for operation in group['operations']:
                manager.create_group_acl(
                    principal=principal,
                    group=group['name'],
                    operation=operation
                )

        print(f"Applied ACLs for {service_name}")

if __name__ == '__main__':
    manager = KafkaAclManager(admin_config)
    apply_acl_config('acl-config.yaml', manager)
```

## Confluent RBAC (Role-Based Access Control)

Для enterprise сред Confluent предлагает RBAC поверх ACLs:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CONFLUENT RBAC МОДЕЛЬ                            │
└─────────────────────────────────────────────────────────────────────┘

  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
  │    USER     │ ──── │    ROLE     │ ──── │  RESOURCE   │
  │             │      │             │      │   SCOPE     │
  └─────────────┘      └─────────────┘      └─────────────┘
        │                    │                    │
        │              ┌─────┴─────┐              │
        │              │           │              │
        ▼              ▼           ▼              ▼
  ┌──────────┐  ┌───────────┐  ┌───────────┐  ┌─────────────┐
  │ alice    │  │ Developer │  │ Operator  │  │ Cluster: *  │
  │ bob      │  │ Admin     │  │ Viewer    │  │ Topic: ord* │
  │ service  │  │ Consumer  │  │ Producer  │  │ Group: *    │
  └──────────┘  └───────────┘  └───────────┘  └─────────────┘
```

```bash
# Назначение роли через confluent CLI
confluent iam rbac role-binding create \
    --principal User:alice \
    --role DeveloperRead \
    --resource Topic:orders \
    --kafka-cluster-id <cluster-id>

# Доступные роли
# - SystemAdmin
# - ClusterAdmin
# - Operator
# - DeveloperRead
# - DeveloperWrite
# - DeveloperManage
```

## Troubleshooting

```bash
# Включение отладки авторизации
# log4j.properties
log4j.logger.kafka.authorizer.logger=DEBUG

# Типичные ошибки
# TopicAuthorizationException: Not authorized to access topics
# GroupAuthorizationException: Not authorized to access group

# Проверка ACLs для пользователя
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --list \
    --principal User:problem-user

# Проверка ACLs для топика
kafka-acls.sh --bootstrap-server kafka:9094 \
    --command-config admin.properties \
    --list \
    --topic problem-topic
```

## Дополнительные ресурсы

- [Apache Kafka Authorization](https://kafka.apache.org/documentation/#security_authz)
- [KIP-11 - Authorization Interface](https://cwiki.apache.org/confluence/display/KAFKA/KIP-11+-+Authorization+Interface)
- [Confluent RBAC](https://docs.confluent.io/platform/current/security/rbac/index.html)
