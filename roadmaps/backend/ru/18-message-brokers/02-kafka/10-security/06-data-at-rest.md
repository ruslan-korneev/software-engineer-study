# Шифрование данных в состоянии покоя (Data at Rest)

[prev: 05-quotas](./05-quotas.md) | [next: 01-schema-registry-basics](../11-schema-registry/01-schema-registry-basics.md)

---

## Описание

Шифрование данных в состоянии покоя (Encryption at Rest) обеспечивает защиту данных, хранящихся на дисках Kafka брокеров. В отличие от шифрования "в движении" (in transit), которое защищает данные при передаче по сети, encryption at rest защищает данные от несанкционированного доступа при физическом доступе к дискам или их компрометации.

Apache Kafka не имеет встроенного механизма шифрования данных на диске, поэтому реализация encryption at rest требует использования внешних инструментов: шифрования на уровне файловой системы, блочных устройств, или шифрования на уровне приложения (client-side encryption).

## Ключевые концепции

### Уровни шифрования

```
┌─────────────────────────────────────────────────────────────────────┐
│                    УРОВНИ ШИФРОВАНИЯ DATA AT REST                   │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  УРОВЕНЬ 1: ШИФРОВАНИЕ НА УРОВНЕ ДИСКА (Full Disk Encryption)      │
│                                                                     │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │
│  │   Операции   │ ── │  Шифрование  │ ── │  Физический  │          │
│  │   файловой   │    │  (LUKS/dm-   │    │     диск     │          │
│  │   системы    │    │   crypt)     │    │              │          │
│  └──────────────┘    └──────────────┘    └──────────────┘          │
│                                                                     │
│  ✓ Прозрачно для Kafka                                              │
│  ✓ Защита от физического доступа                                    │
│  ✗ Не защищает от root-доступа к работающей системе                 │
├─────────────────────────────────────────────────────────────────────┤
│  УРОВЕНЬ 2: ШИФРОВАНИЕ НА УРОВНЕ ФАЙЛОВОЙ СИСТЕМЫ                  │
│                                                                     │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │
│  │    Kafka     │ ── │   eCryptfs/  │ ── │   Ext4/XFS   │          │
│  │    Logs      │    │   fscrypt    │    │              │          │
│  └──────────────┘    └──────────────┘    └──────────────┘          │
│                                                                     │
│  ✓ Шифрование конкретных директорий                                 │
│  ✓ Разные ключи для разных данных                                   │
│  ✗ Некоторая overhead производительности                             │
├─────────────────────────────────────────────────────────────────────┤
│  УРОВЕНЬ 3: CLIENT-SIDE ENCRYPTION (Шифрование на клиенте)         │
│                                                                     │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │
│  │   Producer   │ ── │  Шифрование  │ ── │    Kafka     │          │
│  │  Application │    │   (AES-256)  │    │    Broker    │          │
│  └──────────────┘    └──────────────┘    └──────────────┘          │
│                                                                     │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │
│  │   Consumer   │ ◄─ │ Расшифровка  │ ◄─ │    Kafka     │          │
│  │  Application │    │              │    │    Broker    │          │
│  └──────────────┘    └──────────────┘    └──────────────┘          │
│                                                                     │
│  ✓ End-to-end защита                                                │
│  ✓ Брокер не видит открытые данные                                  │
│  ✗ Усложнение логики приложений                                     │
│  ✗ Невозможность использовать Kafka Streams/KSQL для обработки     │
└─────────────────────────────────────────────────────────────────────┘
```

### Что защищает encryption at rest

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ДАННЫЕ KAFKA НА ДИСКЕ                            │
└─────────────────────────────────────────────────────────────────────┘

/var/kafka-logs/
├── topic-partition/
│   ├── 00000000000000000000.log      ← Данные сообщений (ТРЕБУЕТ ЗАЩИТЫ)
│   ├── 00000000000000000000.index    ← Индексы
│   ├── 00000000000000000000.timeindex
│   └── leader-epoch-checkpoint
├── __consumer_offsets-*/             ← Consumer offsets
├── recovery-point-offset-checkpoint
├── replication-offset-checkpoint
└── log-start-offset-checkpoint

УГРОЗЫ БЕЗ ШИФРОВАНИЯ:
• Физический доступ к серверу/диску
• Кража/потеря диска
• Неправильная утилизация оборудования
• Snapshot виртуальной машины
• Бэкапы без шифрования
• Доступ через другие ОС (при dual boot)
```

## Примеры конфигурации

### Шифрование на уровне диска (LUKS)

```bash
# ═══════════════════════════════════════════════════════════════════
#                    LUKS ENCRYPTION (Linux)
# ═══════════════════════════════════════════════════════════════════

# 1. Установка cryptsetup
apt-get install cryptsetup

# 2. Создание зашифрованного раздела
cryptsetup luksFormat /dev/sdb1

# 3. Открытие зашифрованного раздела
cryptsetup luksOpen /dev/sdb1 kafka-data

# 4. Создание файловой системы
mkfs.ext4 /dev/mapper/kafka-data

# 5. Монтирование
mkdir -p /var/kafka-logs
mount /dev/mapper/kafka-data /var/kafka-logs

# 6. Автоматическое монтирование при загрузке
# /etc/crypttab
kafka-data /dev/sdb1 /etc/kafka-keyfile luks

# /etc/fstab
/dev/mapper/kafka-data /var/kafka-logs ext4 defaults 0 2

# 7. Создание keyfile для автоматического открытия
dd if=/dev/urandom of=/etc/kafka-keyfile bs=512 count=4
chmod 400 /etc/kafka-keyfile
cryptsetup luksAddKey /dev/sdb1 /etc/kafka-keyfile
```

### Шифрование на уровне файловой системы (fscrypt)

```bash
# ═══════════════════════════════════════════════════════════════════
#                    FSCRYPT (Linux 4.1+)
# ═══════════════════════════════════════════════════════════════════

# 1. Установка fscrypt
apt-get install fscrypt libpam-fscrypt

# 2. Включение шифрования на файловой системе
tune2fs -O encrypt /dev/sdb1
fscrypt setup
fscrypt setup /var/kafka-logs

# 3. Создание политики шифрования
fscrypt encrypt /var/kafka-logs/data \
    --source=pam_passphrase \
    --name=kafka-encryption

# 4. Проверка статуса
fscrypt status /var/kafka-logs/data

# 5. Автоматическая разблокировка при входе (PAM integration)
# Настраивается через /etc/pam.d/
```

### AWS EBS Encryption

```yaml
# ═══════════════════════════════════════════════════════════════════
#                    AWS EBS ENCRYPTION
# ═══════════════════════════════════════════════════════════════════

# CloudFormation / Terraform пример
Resources:
  KafkaDataVolume:
    Type: AWS::EC2::Volume
    Properties:
      AvailabilityZone: !Ref AvailabilityZone
      Size: 1000  # GB
      VolumeType: gp3
      Iops: 16000
      Throughput: 1000
      Encrypted: true
      KmsKeyId: !Ref KafkaDataKey

  KafkaDataKey:
    Type: AWS::KMS::Key
    Properties:
      Description: KMS key for Kafka data encryption
      EnableKeyRotation: true
      KeyPolicy:
        Version: '2012-10-17'
        Statement:
          - Sid: Enable IAM policies
            Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:root'
            Action: 'kms:*'
            Resource: '*'
          - Sid: Allow Kafka instances
            Effect: Allow
            Principal:
              AWS: !GetAtt KafkaInstanceRole.Arn
            Action:
              - kms:Encrypt
              - kms:Decrypt
              - kms:GenerateDataKey*
            Resource: '*'
```

```hcl
# Terraform
resource "aws_ebs_volume" "kafka_data" {
  availability_zone = "us-east-1a"
  size              = 1000
  type              = "gp3"
  iops              = 16000
  throughput        = 1000
  encrypted         = true
  kms_key_id        = aws_kms_key.kafka_data.arn

  tags = {
    Name = "kafka-data"
  }
}

resource "aws_kms_key" "kafka_data" {
  description             = "KMS key for Kafka data encryption"
  enable_key_rotation     = true
  deletion_window_in_days = 30
}
```

### Client-Side Encryption

**Архитектура:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CLIENT-SIDE ENCRYPTION FLOW                      │
└─────────────────────────────────────────────────────────────────────┘

PRODUCER SIDE:
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Plaintext  │ ──► │  Serialize  │ ──► │   Encrypt   │ ──►
│    Data     │     │    (JSON)   │     │  (AES-256)  │
└─────────────┘     └─────────────┘     └─────────────┘
                                              │
    ┌─────────────┐                           │
    │  KMS / Vault│ ◄── Получить Data Key ────┤
    │             │                           │
    └─────────────┘                           │
                                              ▼
                                    ┌─────────────────┐
                                    │  Kafka Broker   │
                                    │  (Encrypted     │
                                    │   Data)         │
                                    └─────────────────┘
                                              │
CONSUMER SIDE:                                │
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Plaintext  │ ◄── │ Deserialize │ ◄── │   Decrypt   │ ◄──
│    Data     │     │    (JSON)   │     │  (AES-256)  │
└─────────────┘     └─────────────┘     └─────────────┘
                                              │
    ┌─────────────┐                           │
    │  KMS / Vault│ ◄── Получить Data Key ────┘
    │             │
    └─────────────┘
```

**Реализация на Python:**

```python
import json
import base64
from typing import Optional
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
from kafka import KafkaProducer, KafkaConsumer
import boto3
import os

class EncryptionService:
    """Сервис шифрования с использованием AWS KMS или локальных ключей"""

    def __init__(self, kms_key_id: Optional[str] = None):
        self.kms_key_id = kms_key_id
        if kms_key_id:
            self.kms_client = boto3.client('kms')
        else:
            # Локальный ключ для разработки
            self.local_key = os.environ.get('ENCRYPTION_KEY', Fernet.generate_key())
            self.fernet = Fernet(self.local_key)

    def get_data_key(self) -> tuple[bytes, bytes]:
        """Получить data key от KMS"""
        if self.kms_key_id:
            response = self.kms_client.generate_data_key(
                KeyId=self.kms_key_id,
                KeySpec='AES_256'
            )
            return (
                response['Plaintext'],      # Для шифрования
                response['CiphertextBlob']  # Для хранения
            )
        else:
            # Локальный режим
            key = AESGCM.generate_key(bit_length=256)
            return key, key

    def decrypt_data_key(self, encrypted_key: bytes) -> bytes:
        """Расшифровать data key через KMS"""
        if self.kms_key_id:
            response = self.kms_client.decrypt(
                CiphertextBlob=encrypted_key
            )
            return response['Plaintext']
        else:
            return encrypted_key

    def encrypt(self, plaintext: bytes) -> dict:
        """
        Envelope Encryption:
        1. Получить data key от KMS
        2. Зашифровать данные data key-ом
        3. Вернуть зашифрованные данные + зашифрованный data key
        """
        data_key, encrypted_data_key = self.get_data_key()

        # Шифрование с AES-GCM
        nonce = os.urandom(12)
        aesgcm = AESGCM(data_key)
        ciphertext = aesgcm.encrypt(nonce, plaintext, None)

        return {
            'encrypted_data': base64.b64encode(nonce + ciphertext).decode(),
            'encrypted_key': base64.b64encode(encrypted_data_key).decode(),
            'algorithm': 'AES-256-GCM'
        }

    def decrypt(self, encrypted_message: dict) -> bytes:
        """Расшифровка envelope encrypted сообщения"""
        encrypted_data = base64.b64decode(encrypted_message['encrypted_data'])
        encrypted_key = base64.b64decode(encrypted_message['encrypted_key'])

        # Расшифровать data key
        data_key = self.decrypt_data_key(encrypted_key)

        # Расшифровать данные
        nonce = encrypted_data[:12]
        ciphertext = encrypted_data[12:]
        aesgcm = AESGCM(data_key)

        return aesgcm.decrypt(nonce, ciphertext, None)


class EncryptingSerializer:
    """Serializer с шифрованием для Kafka"""

    def __init__(self, encryption_service: EncryptionService):
        self.encryption = encryption_service

    def serialize(self, data) -> bytes:
        """Сериализация и шифрование"""
        json_data = json.dumps(data).encode('utf-8')
        encrypted = self.encryption.encrypt(json_data)
        return json.dumps(encrypted).encode('utf-8')

    def deserialize(self, data: bytes):
        """Расшифровка и десериализация"""
        encrypted_message = json.loads(data.decode('utf-8'))
        decrypted = self.encryption.decrypt(encrypted_message)
        return json.loads(decrypted.decode('utf-8'))


class EncryptedKafkaProducer:
    """Producer с автоматическим шифрованием"""

    def __init__(self, kafka_config: dict, kms_key_id: str = None):
        self.encryption = EncryptionService(kms_key_id)
        self.serializer = EncryptingSerializer(self.encryption)

        self.producer = KafkaProducer(
            **kafka_config,
            value_serializer=self.serializer.serialize
        )

    def send(self, topic: str, value: dict, key: bytes = None):
        """Отправка зашифрованного сообщения"""
        return self.producer.send(topic, value=value, key=key)

    def flush(self):
        self.producer.flush()


class EncryptedKafkaConsumer:
    """Consumer с автоматической расшифровкой"""

    def __init__(self, kafka_config: dict, topics: list, kms_key_id: str = None):
        self.encryption = EncryptionService(kms_key_id)
        self.serializer = EncryptingSerializer(self.encryption)

        self.consumer = KafkaConsumer(
            *topics,
            **kafka_config,
            value_deserializer=self.serializer.deserialize
        )

    def __iter__(self):
        return iter(self.consumer)

    def close(self):
        self.consumer.close()


# Пример использования
if __name__ == '__main__':
    kafka_config = {
        'bootstrap_servers': ['kafka:9092'],
        'security_protocol': 'SASL_SSL',
        'sasl_mechanism': 'SCRAM-SHA-512',
        'sasl_plain_username': 'producer',
        'sasl_plain_password': 'secret',
    }

    # Producer
    producer = EncryptedKafkaProducer(
        kafka_config,
        kms_key_id='alias/kafka-data-key'
    )

    sensitive_data = {
        'user_id': 12345,
        'ssn': '123-45-6789',
        'credit_card': '4111111111111111'
    }

    producer.send('sensitive-topic', sensitive_data)
    producer.flush()

    # Consumer
    consumer = EncryptedKafkaConsumer(
        {**kafka_config, 'group_id': 'secure-consumer'},
        ['sensitive-topic'],
        kms_key_id='alias/kafka-data-key'
    )

    for message in consumer:
        print(f"Decrypted message: {message.value}")
```

**Реализация на Java:**

```java
import javax.crypto.Cipher;
import javax.crypto.SecretKey;
import javax.crypto.spec.GCMParameterSpec;
import javax.crypto.spec.SecretKeySpec;
import java.security.SecureRandom;
import java.util.Base64;

import com.amazonaws.services.kms.AWSKMS;
import com.amazonaws.services.kms.AWSKMSClientBuilder;
import com.amazonaws.services.kms.model.*;

import org.apache.kafka.common.serialization.Serializer;
import org.apache.kafka.common.serialization.Deserializer;

public class EncryptingSerializer implements Serializer<String> {

    private static final String ALGORITHM = "AES/GCM/NoPadding";
    private static final int GCM_IV_LENGTH = 12;
    private static final int GCM_TAG_LENGTH = 128;

    private final AWSKMS kmsClient;
    private final String kmsKeyId;

    public EncryptingSerializer(String kmsKeyId) {
        this.kmsClient = AWSKMSClientBuilder.defaultClient();
        this.kmsKeyId = kmsKeyId;
    }

    @Override
    public byte[] serialize(String topic, String data) {
        try {
            // 1. Получить data key от KMS
            GenerateDataKeyResult dataKeyResult = kmsClient.generateDataKey(
                new GenerateDataKeyRequest()
                    .withKeyId(kmsKeyId)
                    .withKeySpec(DataKeySpec.AES_256)
            );

            byte[] plaintextKey = dataKeyResult.getPlaintext().array();
            byte[] encryptedKey = dataKeyResult.getCiphertextBlob().array();

            // 2. Зашифровать данные
            SecretKey secretKey = new SecretKeySpec(plaintextKey, "AES");
            Cipher cipher = Cipher.getInstance(ALGORITHM);

            byte[] iv = new byte[GCM_IV_LENGTH];
            new SecureRandom().nextBytes(iv);

            GCMParameterSpec parameterSpec = new GCMParameterSpec(GCM_TAG_LENGTH, iv);
            cipher.init(Cipher.ENCRYPT_MODE, secretKey, parameterSpec);

            byte[] ciphertext = cipher.doFinal(data.getBytes());

            // 3. Собрать envelope
            // Format: encryptedKeyLength(4) + encryptedKey + iv + ciphertext
            byte[] result = new byte[4 + encryptedKey.length + iv.length + ciphertext.length];
            System.arraycopy(intToBytes(encryptedKey.length), 0, result, 0, 4);
            System.arraycopy(encryptedKey, 0, result, 4, encryptedKey.length);
            System.arraycopy(iv, 0, result, 4 + encryptedKey.length, iv.length);
            System.arraycopy(ciphertext, 0, result, 4 + encryptedKey.length + iv.length, ciphertext.length);

            return result;

        } catch (Exception e) {
            throw new RuntimeException("Encryption failed", e);
        }
    }

    private byte[] intToBytes(int value) {
        return new byte[] {
            (byte)(value >> 24),
            (byte)(value >> 16),
            (byte)(value >> 8),
            (byte)value
        };
    }

    @Override
    public void close() {
        kmsClient.shutdown();
    }
}

public class DecryptingDeserializer implements Deserializer<String> {

    private final AWSKMS kmsClient;

    public DecryptingDeserializer() {
        this.kmsClient = AWSKMSClientBuilder.defaultClient();
    }

    @Override
    public String deserialize(String topic, byte[] data) {
        try {
            // 1. Разобрать envelope
            int encryptedKeyLength = bytesToInt(data, 0);
            byte[] encryptedKey = new byte[encryptedKeyLength];
            System.arraycopy(data, 4, encryptedKey, 0, encryptedKeyLength);

            byte[] iv = new byte[12];
            System.arraycopy(data, 4 + encryptedKeyLength, iv, 0, 12);

            byte[] ciphertext = new byte[data.length - 4 - encryptedKeyLength - 12];
            System.arraycopy(data, 4 + encryptedKeyLength + 12, ciphertext, 0, ciphertext.length);

            // 2. Расшифровать data key через KMS
            DecryptResult decryptResult = kmsClient.decrypt(
                new DecryptRequest().withCiphertextBlob(java.nio.ByteBuffer.wrap(encryptedKey))
            );
            byte[] plaintextKey = decryptResult.getPlaintext().array();

            // 3. Расшифровать данные
            SecretKey secretKey = new SecretKeySpec(plaintextKey, "AES");
            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
            GCMParameterSpec parameterSpec = new GCMParameterSpec(128, iv);
            cipher.init(Cipher.DECRYPT_MODE, secretKey, parameterSpec);

            byte[] plaintext = cipher.doFinal(ciphertext);
            return new String(plaintext);

        } catch (Exception e) {
            throw new RuntimeException("Decryption failed", e);
        }
    }

    private int bytesToInt(byte[] bytes, int offset) {
        return ((bytes[offset] & 0xFF) << 24) |
               ((bytes[offset + 1] & 0xFF) << 16) |
               ((bytes[offset + 2] & 0xFF) << 8) |
               (bytes[offset + 3] & 0xFF);
    }

    @Override
    public void close() {
        kmsClient.shutdown();
    }
}
```

### Confluent Schema Registry с шифрованием

```yaml
# Конфигурация для Field-Level Encryption с Confluent
# schema-registry-encryption.properties

# Включение шифрования полей
# Требует Confluent Platform Enterprise

# KMS провайдер
encryption.kms.provider=aws-kms
encryption.kms.key.id=alias/schema-registry-key

# Правила шифрования в схеме (Avro)
# В схеме указываются поля для шифрования
```

```json
// Avro schema с метаданными шифрования
{
  "type": "record",
  "name": "User",
  "fields": [
    {"name": "id", "type": "long"},
    {"name": "name", "type": "string"},
    {
      "name": "ssn",
      "type": "string",
      "confluent:tags": ["PII", "ENCRYPT"]
    },
    {
      "name": "credit_card",
      "type": "string",
      "confluent:tags": ["PCI", "ENCRYPT"]
    }
  ]
}
```

## Best Practices

### 1. Выбор стратегии шифрования

```markdown
## Рекомендации по выбору

### Disk Encryption (LUKS, BitLocker, AWS EBS)
✅ Рекомендуется для:
  - Защита от физического доступа
  - Compliance требования (HIPAA, PCI DSS)
  - Cloud deployments

✅ Преимущества:
  - Прозрачно для приложений
  - Минимальный performance impact
  - Простота управления

### Client-Side Encryption
✅ Рекомендуется для:
  - End-to-end защита конфиденциальных данных
  - Zero-trust архитектура
  - Multi-tenant environments

⚠️ Ограничения:
  - Невозможно использовать Kafka Streams / KSQL
  - Сложность key management
  - Увеличение размера сообщений (overhead)
```

### 2. Управление ключами

```python
# Key rotation стратегия
class KeyRotationManager:
    """Управление ротацией ключей шифрования"""

    def __init__(self, kms_client, key_alias: str):
        self.kms = kms_client
        self.key_alias = key_alias

    def rotate_key(self):
        """Ротация master key в KMS"""
        # AWS KMS поддерживает автоматическую ротацию
        self.kms.enable_key_rotation(KeyId=self.key_alias)

    def re_encrypt_data(self, old_data: bytes) -> bytes:
        """
        Re-encryption с новым ключом.
        KMS автоматически использует актуальный ключ.
        """
        # Расшифровка старым ключом (KMS хранит историю)
        decrypted = self.kms.decrypt(CiphertextBlob=old_data)

        # Шифрование новым ключом
        encrypted = self.kms.encrypt(
            KeyId=self.key_alias,
            Plaintext=decrypted['Plaintext']
        )

        return encrypted['CiphertextBlob']
```

### 3. Мониторинг шифрования

```yaml
# Prometheus alerts
groups:
  - name: encryption-alerts
    rules:
      - alert: DiskEncryptionNotEnabled
        expr: node_disk_encryption_enabled == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk encryption not enabled on {{ $labels.instance }}"

      - alert: KMSKeyRotationDisabled
        expr: aws_kms_key_rotation_enabled == 0
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "KMS key rotation disabled for {{ $labels.key_id }}"

      - alert: EncryptionDecryptionErrors
        expr: rate(encryption_errors_total[5m]) > 0
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Encryption/decryption errors detected"
```

### 4. Compliance checklist

```markdown
## Checklist для Encryption at Rest

### Disk Encryption
- [ ] Все Kafka data volumes зашифрованы
- [ ] ZooKeeper data volumes зашифрованы
- [ ] Ключи шифрования хранятся в HSM/KMS
- [ ] Автоматическая ротация ключей включена
- [ ] Recovery процедуры документированы и протестированы

### Client-Side Encryption (если используется)
- [ ] Выбран надёжный алгоритм (AES-256-GCM)
- [ ] Envelope encryption реализован
- [ ] Key management интегрирован с KMS
- [ ] Ротация data keys настроена
- [ ] Backward compatibility для расшифровки старых сообщений

### Operational Security
- [ ] Доступ к ключам шифрования ограничен
- [ ] Аудит логирование доступа к ключам
- [ ] Separation of duties (разные люди управляют ключами и данными)
- [ ] Secure key backup процедуры

### Compliance
- [ ] Документация соответствия (PCI DSS, HIPAA, GDPR)
- [ ] Регулярные аудиты
- [ ] Penetration testing
```

### 5. Performance considerations

```python
# Benchmark шифрования
import time
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os

def benchmark_encryption(message_size_kb: int, iterations: int = 1000):
    """Измерение overhead шифрования"""
    key = AESGCM.generate_key(bit_length=256)
    aesgcm = AESGCM(key)
    data = os.urandom(message_size_kb * 1024)

    # Шифрование
    start = time.time()
    for _ in range(iterations):
        nonce = os.urandom(12)
        ciphertext = aesgcm.encrypt(nonce, data, None)
    encrypt_time = (time.time() - start) / iterations * 1000

    # Расшифровка
    start = time.time()
    for _ in range(iterations):
        plaintext = aesgcm.decrypt(nonce, ciphertext, None)
    decrypt_time = (time.time() - start) / iterations * 1000

    overhead = (len(ciphertext) - len(data)) / len(data) * 100

    print(f"Message size: {message_size_kb} KB")
    print(f"Encryption time: {encrypt_time:.3f} ms")
    print(f"Decryption time: {decrypt_time:.3f} ms")
    print(f"Size overhead: {overhead:.2f}%")

# Примерные результаты:
# Message size: 1 KB  -> Encrypt: 0.05ms, Decrypt: 0.05ms, Overhead: 1.6%
# Message size: 10 KB -> Encrypt: 0.15ms, Decrypt: 0.15ms, Overhead: 0.16%
# Message size: 100 KB -> Encrypt: 1.2ms, Decrypt: 1.2ms, Overhead: 0.016%
```

## Дополнительные ресурсы

- [AWS EBS Encryption](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSEncryption.html)
- [Azure Disk Encryption](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/disk-encryption-overview)
- [LUKS Documentation](https://gitlab.com/cryptsetup/cryptsetup/-/wikis/LUKS-standard)
- [Confluent Schema Registry Encryption](https://docs.confluent.io/platform/current/schema-registry/security/schema-encryption.html)
- [NIST Guidelines for Encryption](https://csrc.nist.gov/publications/detail/sp/800-175b/rev-1/final)

---

[prev: 05-quotas](./05-quotas.md) | [next: 01-schema-registry-basics](../11-schema-registry/01-schema-registry-basics.md)
