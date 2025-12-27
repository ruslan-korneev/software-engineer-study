# Реализация заводских требований и обработка ошибок (Error Handling)

## Описание

Обработка ошибок в Kafka Consumer — критически важный аспект построения надёжных систем обработки сообщений. Ошибки могут возникать на разных уровнях: при подключении к брокерам, десериализации сообщений, обработке бизнес-логики и коммите offset'ов. Правильная стратегия обработки ошибок определяет, будет ли система терять данные, дублировать обработку или останавливаться при сбоях.

## Ключевые концепции

### Типы ошибок в Consumer

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Типы ошибок Consumer                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐ │
│  │   Retriable     │  │  Non-Retriable  │  │   Fatal Errors      │ │
│  │   Errors        │  │  Errors         │  │                     │ │
│  ├─────────────────┤  ├─────────────────┤  ├─────────────────────┤ │
│  │ • NetworkError  │  │ • Serialization │  │ • AuthError         │ │
│  │ • Timeout       │  │ • InvalidRecord │  │ • TopicNotExists    │ │
│  │ • NotLeader     │  │ • Schema Error  │  │ • GroupAuth Failed  │ │
│  │ • Coordinator   │  │ • Poison Pill   │  │ • ConfigError       │ │
│  │   Not Available │  │                 │  │                     │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────┘ │
│         ↓                    ↓                      ↓              │
│    Retry with           Dead Letter             Shutdown           │
│    backoff              Queue (DLQ)             + Alert            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Стратегии обработки ошибок

| Стратегия | Описание | Использование |
|-----------|----------|---------------|
| **Retry** | Повторная попытка с backoff | Временные сбои |
| **Skip** | Пропуск сообщения | Некритичные данные |
| **Dead Letter Queue** | Отправка в специальный топик | Важные данные |
| **Stop** | Остановка consumer | Критические ошибки |
| **Pause/Resume** | Приостановка партиции | Backpressure |

### Гарантии доставки

| Гарантия | Описание | Как достичь |
|----------|----------|-------------|
| **At-most-once** | Сообщение обработано 0-1 раз | Commit перед обработкой |
| **At-least-once** | Сообщение обработано 1+ раз | Commit после обработки |
| **Exactly-once** | Сообщение обработано ровно 1 раз | Транзакции + идемпотентность |

## Примеры кода

### Базовая обработка ошибок

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.errors.*;

import java.time.Duration;
import java.util.*;

public class BasicErrorHandling {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "error-handling-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
            consumer.subscribe(Collections.singletonList("my-topic"));

            while (true) {
                try {
                    ConsumerRecords<String, String> records =
                            consumer.poll(Duration.ofMillis(1000));

                    for (ConsumerRecord<String, String> record : records) {
                        try {
                            processRecord(record);
                        } catch (ProcessingException e) {
                            // Ошибка обработки конкретного сообщения
                            handleProcessingError(record, e);
                        }
                    }

                    consumer.commitSync();

                } catch (WakeupException e) {
                    // Ожидаемое исключение при shutdown
                    throw e;

                } catch (RetriableException e) {
                    // Можно повторить (сетевые ошибки, временная недоступность)
                    System.err.println("Retriable error, will retry: " + e.getMessage());
                    sleep(1000);

                } catch (CommitFailedException e) {
                    // Ребалансировка произошла — партиции потеряны
                    System.err.println("Commit failed due to rebalance: " + e.getMessage());
                    // Продолжаем — данные будут переобработаны

                } catch (SerializationException e) {
                    // Ошибка десериализации — пропуск или DLQ
                    System.err.println("Serialization error: " + e.getMessage());
                    // Нужна специальная обработка — см. ниже

                } catch (AuthorizationException | AuthenticationException e) {
                    // Фатальная ошибка — нет доступа
                    System.err.println("Auth error, shutting down: " + e.getMessage());
                    throw e;
                }
            }
        }
    }

    private static void processRecord(ConsumerRecord<String, String> record)
            throws ProcessingException {
        System.out.printf("Processing: %s%n", record.value());
        // Бизнес-логика, которая может выбросить исключение
    }

    private static void handleProcessingError(ConsumerRecord<String, String> record,
                                              Exception e) {
        System.err.printf("Failed to process record: partition=%d, offset=%d, error=%s%n",
                record.partition(), record.offset(), e.getMessage());
        // Логирование, отправка в DLQ и т.д.
    }

    private static void sleep(long ms) {
        try {
            Thread.sleep(ms);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}

class ProcessingException extends Exception {
    public ProcessingException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

### Dead Letter Queue (DLQ)

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.header.Headers;
import org.apache.kafka.common.header.internals.RecordHeaders;

import java.time.Duration;
import java.util.*;

public class DeadLetterQueueExample {

    private static final String DLQ_TOPIC = "my-topic-dlq";
    private static final int MAX_RETRIES = 3;

    private KafkaConsumer<String, String> consumer;
    private KafkaProducer<String, String> dlqProducer;

    public void run() {
        Properties consumerProps = createConsumerProps();
        Properties producerProps = createProducerProps();

        consumer = new KafkaConsumer<>(consumerProps);
        dlqProducer = new KafkaProducer<>(producerProps);

        consumer.subscribe(Collections.singletonList("my-topic"));

        try {
            while (true) {
                ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    boolean processed = processWithRetry(record, MAX_RETRIES);

                    if (!processed) {
                        // Отправка в DLQ после неудачных попыток
                        sendToDeadLetterQueue(record);
                    }
                }

                consumer.commitSync();
            }
        } finally {
            consumer.close();
            dlqProducer.close();
        }
    }

    private boolean processWithRetry(ConsumerRecord<String, String> record, int maxRetries) {
        int attempt = 0;
        long backoff = 100; // начальная задержка в мс

        while (attempt < maxRetries) {
            try {
                processRecord(record);
                return true;
            } catch (RetriableProcessingException e) {
                attempt++;
                System.err.printf("Attempt %d/%d failed: %s%n",
                        attempt, maxRetries, e.getMessage());

                if (attempt < maxRetries) {
                    sleep(backoff);
                    backoff *= 2; // exponential backoff
                }
            } catch (NonRetriableProcessingException e) {
                // Не повторяем — сразу в DLQ
                System.err.println("Non-retriable error: " + e.getMessage());
                return false;
            }
        }

        return false;
    }

    private void sendToDeadLetterQueue(ConsumerRecord<String, String> record) {
        // Добавляем метаданные об ошибке в headers
        Headers headers = new RecordHeaders();
        headers.add("original-topic", record.topic().getBytes());
        headers.add("original-partition", String.valueOf(record.partition()).getBytes());
        headers.add("original-offset", String.valueOf(record.offset()).getBytes());
        headers.add("original-timestamp", String.valueOf(record.timestamp()).getBytes());
        headers.add("failure-reason", "Max retries exceeded".getBytes());
        headers.add("failure-time", String.valueOf(System.currentTimeMillis()).getBytes());

        // Копируем оригинальные headers
        record.headers().forEach(h -> headers.add("orig-" + h.key(), h.value()));

        ProducerRecord<String, String> dlqRecord = new ProducerRecord<>(
                DLQ_TOPIC,
                null,           // partition
                record.key(),
                record.value(),
                headers
        );

        try {
            dlqProducer.send(dlqRecord, (metadata, exception) -> {
                if (exception != null) {
                    System.err.println("Failed to send to DLQ: " + exception.getMessage());
                } else {
                    System.out.printf("Sent to DLQ: topic=%s, partition=%d, offset=%d%n",
                            metadata.topic(), metadata.partition(), metadata.offset());
                }
            });
        } catch (Exception e) {
            System.err.println("Error sending to DLQ: " + e.getMessage());
            // Критическая ошибка — возможно, стоит остановить consumer
        }
    }

    private void processRecord(ConsumerRecord<String, String> record)
            throws RetriableProcessingException, NonRetriableProcessingException {
        // Бизнес-логика
        String value = record.value();

        if (value == null || value.isEmpty()) {
            throw new NonRetriableProcessingException("Empty message");
        }

        // Имитация временной ошибки
        if (Math.random() < 0.3) {
            throw new RetriableProcessingException("Temporary failure");
        }

        System.out.println("Processed: " + value);
    }

    private Properties createConsumerProps() {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "dlq-example-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        return props;
    }

    private Properties createProducerProps() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        return props;
    }

    private void sleep(long ms) {
        try {
            Thread.sleep(ms);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}

class RetriableProcessingException extends Exception {
    public RetriableProcessingException(String message) {
        super(message);
    }
}

class NonRetriableProcessingException extends Exception {
    public NonRetriableProcessingException(String message) {
        super(message);
    }
}
```

### Обработка ошибок десериализации

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.serialization.Deserializer;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.time.Duration;
import java.util.*;

public class DeserializationErrorHandling {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "deser-error-group");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        // Использование кастомного десериализатора с обработкой ошибок
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  ErrorHandlingDeserializer.class.getName());

        // Конфигурация для ErrorHandlingDeserializer
        props.put("value.deserializer.delegate",
                  "com.example.JsonDeserializer");
        props.put("value.deserializer.error.handler",
                  "com.example.DeserializationErrorHandler");

        try (KafkaConsumer<String, DeserializationResult<MyObject>> consumer =
                     new KafkaConsumer<>(props)) {

            consumer.subscribe(Collections.singletonList("my-topic"));

            while (true) {
                ConsumerRecords<String, DeserializationResult<MyObject>> records =
                        consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, DeserializationResult<MyObject>> record : records) {
                    DeserializationResult<MyObject> result = record.value();

                    if (result.isSuccess()) {
                        processRecord(result.getValue());
                    } else {
                        handleDeserializationError(record, result.getError());
                    }
                }

                consumer.commitSync();
            }
        }
    }

    private static void processRecord(MyObject obj) {
        System.out.println("Processing: " + obj);
    }

    private static void handleDeserializationError(
            ConsumerRecord<String, DeserializationResult<MyObject>> record,
            Exception error) {

        System.err.printf("Deserialization failed: partition=%d, offset=%d, error=%s%n",
                record.partition(), record.offset(), error.getMessage());

        // Опции:
        // 1. Пропустить сообщение
        // 2. Отправить в DLQ
        // 3. Сохранить raw bytes для анализа
    }
}

// Результат десериализации
class DeserializationResult<T> {
    private final T value;
    private final Exception error;
    private final byte[] rawBytes;

    public static <T> DeserializationResult<T> success(T value) {
        return new DeserializationResult<>(value, null, null);
    }

    public static <T> DeserializationResult<T> failure(Exception error, byte[] rawBytes) {
        return new DeserializationResult<>(null, error, rawBytes);
    }

    private DeserializationResult(T value, Exception error, byte[] rawBytes) {
        this.value = value;
        this.error = error;
        this.rawBytes = rawBytes;
    }

    public boolean isSuccess() {
        return error == null;
    }

    public T getValue() {
        return value;
    }

    public Exception getError() {
        return error;
    }

    public byte[] getRawBytes() {
        return rawBytes;
    }
}

// Кастомный десериализатор с обработкой ошибок
class ErrorHandlingDeserializer<T> implements Deserializer<DeserializationResult<T>> {

    private Deserializer<T> delegate;

    @Override
    public void configure(Map<String, ?> configs, boolean isKey) {
        String delegateClass = (String) configs.get("value.deserializer.delegate");
        try {
            delegate = (Deserializer<T>) Class.forName(delegateClass)
                    .getDeclaredConstructor().newInstance();
            delegate.configure(configs, isKey);
        } catch (Exception e) {
            throw new RuntimeException("Failed to create delegate deserializer", e);
        }
    }

    @Override
    public DeserializationResult<T> deserialize(String topic, byte[] data) {
        if (data == null) {
            return DeserializationResult.success(null);
        }

        try {
            T value = delegate.deserialize(topic, data);
            return DeserializationResult.success(value);
        } catch (Exception e) {
            return DeserializationResult.failure(e, data);
        }
    }

    @Override
    public void close() {
        if (delegate != null) {
            delegate.close();
        }
    }
}

class MyObject {
    // Ваш объект данных
}
```

### Circuit Breaker Pattern

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.TopicPartition;

import java.time.Duration;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicReference;

public class CircuitBreakerConsumer {

    private KafkaConsumer<String, String> consumer;
    private CircuitBreaker circuitBreaker;

    public void run() {
        Properties props = createConsumerProps();
        consumer = new KafkaConsumer<>(props);
        circuitBreaker = new CircuitBreaker(5, Duration.ofSeconds(30));

        consumer.subscribe(Collections.singletonList("my-topic"));

        while (true) {
            ConsumerRecords<String, String> records =
                    consumer.poll(Duration.ofMillis(1000));

            if (circuitBreaker.isOpen()) {
                // Circuit breaker открыт — пауза партиций
                System.out.println("Circuit breaker OPEN, pausing consumption");
                consumer.pause(consumer.assignment());

                if (circuitBreaker.shouldAttemptReset()) {
                    System.out.println("Attempting circuit breaker reset...");
                    circuitBreaker.attemptReset();
                    consumer.resume(consumer.assignment());
                } else {
                    sleep(1000);
                    continue;
                }
            }

            for (ConsumerRecord<String, String> record : records) {
                try {
                    processRecord(record);
                    circuitBreaker.recordSuccess();
                } catch (Exception e) {
                    circuitBreaker.recordFailure();

                    if (circuitBreaker.isOpen()) {
                        // Откат позиции — сообщение будет переобработано
                        consumer.seek(
                                new TopicPartition(record.topic(), record.partition()),
                                record.offset()
                        );
                        break;
                    }
                }
            }

            if (!circuitBreaker.isOpen()) {
                consumer.commitSync();
            }
        }
    }

    private void processRecord(ConsumerRecord<String, String> record) throws Exception {
        // Бизнес-логика
        System.out.println("Processing: " + record.value());
    }

    private Properties createConsumerProps() {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "circuit-breaker-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        return props;
    }

    private void sleep(long ms) {
        try {
            Thread.sleep(ms);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}

class CircuitBreaker {

    enum State { CLOSED, OPEN, HALF_OPEN }

    private final int failureThreshold;
    private final Duration resetTimeout;
    private final AtomicInteger failureCount = new AtomicInteger(0);
    private final AtomicInteger successCount = new AtomicInteger(0);
    private final AtomicReference<State> state = new AtomicReference<>(State.CLOSED);
    private volatile Instant lastFailureTime;

    public CircuitBreaker(int failureThreshold, Duration resetTimeout) {
        this.failureThreshold = failureThreshold;
        this.resetTimeout = resetTimeout;
    }

    public void recordSuccess() {
        if (state.get() == State.HALF_OPEN) {
            int successes = successCount.incrementAndGet();
            if (successes >= 3) { // 3 успешных операции для закрытия
                reset();
            }
        } else {
            failureCount.set(0);
        }
    }

    public void recordFailure() {
        lastFailureTime = Instant.now();
        int failures = failureCount.incrementAndGet();

        if (failures >= failureThreshold) {
            state.set(State.OPEN);
            System.out.println("Circuit breaker tripped OPEN after " + failures + " failures");
        }
    }

    public boolean isOpen() {
        return state.get() == State.OPEN;
    }

    public boolean shouldAttemptReset() {
        if (state.get() != State.OPEN) {
            return false;
        }

        return lastFailureTime != null &&
               Instant.now().isAfter(lastFailureTime.plus(resetTimeout));
    }

    public void attemptReset() {
        state.set(State.HALF_OPEN);
        successCount.set(0);
        System.out.println("Circuit breaker transitioning to HALF_OPEN");
    }

    private void reset() {
        state.set(State.CLOSED);
        failureCount.set(0);
        successCount.set(0);
        System.out.println("Circuit breaker CLOSED");
    }
}
```

### Обработка ошибок на Python

```python
from confluent_kafka import Consumer, Producer, KafkaError, KafkaException
import json
import time
import logging
from dataclasses import dataclass
from typing import Optional, Any
from enum import Enum

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class ErrorType(Enum):
    RETRIABLE = "retriable"
    NON_RETRIABLE = "non_retriable"
    FATAL = "fatal"

@dataclass
class ProcessingError:
    error_type: ErrorType
    message: str
    exception: Optional[Exception] = None

class RetryConfig:
    def __init__(self, max_retries: int = 3, initial_backoff: float = 0.1):
        self.max_retries = max_retries
        self.initial_backoff = initial_backoff

class ErrorHandlingConsumer:
    def __init__(self, consumer_config: dict, dlq_topic: str):
        self.consumer = Consumer(consumer_config)
        self.dlq_producer = Producer({
            'bootstrap.servers': consumer_config['bootstrap.servers']
        })
        self.dlq_topic = dlq_topic
        self.retry_config = RetryConfig()
        self.running = True

    def run(self, topics: list):
        self.consumer.subscribe(topics)

        try:
            while self.running:
                try:
                    msg = self.consumer.poll(timeout=1.0)

                    if msg is None:
                        continue

                    if msg.error():
                        self._handle_kafka_error(msg.error())
                        continue

                    # Обработка с retry
                    success = self._process_with_retry(msg)

                    if not success:
                        self._send_to_dlq(msg)

                    # Коммит после обработки
                    self.consumer.commit(asynchronous=False)

                except KafkaException as e:
                    logger.error(f"Kafka exception: {e}")
                    self._handle_kafka_exception(e)

        except KeyboardInterrupt:
            logger.info("Shutdown requested")
        finally:
            self._cleanup()

    def _process_with_retry(self, msg) -> bool:
        """Обработка с повторными попытками"""
        attempt = 0
        backoff = self.retry_config.initial_backoff

        while attempt < self.retry_config.max_retries:
            try:
                self._process_message(msg)
                return True

            except RetriableError as e:
                attempt += 1
                logger.warning(f"Attempt {attempt}/{self.retry_config.max_retries} "
                             f"failed: {e}")

                if attempt < self.retry_config.max_retries:
                    time.sleep(backoff)
                    backoff *= 2  # exponential backoff

            except NonRetriableError as e:
                logger.error(f"Non-retriable error: {e}")
                return False

        logger.error(f"Max retries exceeded for message at offset {msg.offset()}")
        return False

    def _process_message(self, msg):
        """Обработка сообщения"""
        try:
            value = msg.value().decode('utf-8')
            data = json.loads(value)

            # Бизнес-логика
            logger.info(f"Processing: {data}")

            # Имитация случайной ошибки
            if 'error' in data:
                if data['error'] == 'retriable':
                    raise RetriableError("Temporary failure")
                elif data['error'] == 'fatal':
                    raise NonRetriableError("Fatal error")

        except json.JSONDecodeError as e:
            raise NonRetriableError(f"Invalid JSON: {e}")

    def _send_to_dlq(self, msg):
        """Отправка в Dead Letter Queue"""
        headers = [
            ('original-topic', msg.topic().encode()),
            ('original-partition', str(msg.partition()).encode()),
            ('original-offset', str(msg.offset()).encode()),
            ('failure-time', str(int(time.time() * 1000)).encode()),
        ]

        # Добавляем оригинальные headers
        if msg.headers():
            for key, value in msg.headers():
                headers.append((f'orig-{key}', value))

        try:
            self.dlq_producer.produce(
                self.dlq_topic,
                key=msg.key(),
                value=msg.value(),
                headers=headers,
                callback=self._dlq_delivery_callback
            )
            self.dlq_producer.flush()

        except Exception as e:
            logger.error(f"Failed to send to DLQ: {e}")

    def _dlq_delivery_callback(self, err, msg):
        if err:
            logger.error(f"DLQ delivery failed: {err}")
        else:
            logger.info(f"Sent to DLQ: topic={msg.topic()}, "
                       f"partition={msg.partition()}, offset={msg.offset()}")

    def _handle_kafka_error(self, error):
        """Обработка ошибок Kafka"""
        if error.code() == KafkaError._PARTITION_EOF:
            logger.debug("Reached end of partition")

        elif error.code() in [
            KafkaError._ALL_BROKERS_DOWN,
            KafkaError._TRANSPORT
        ]:
            logger.error(f"Broker connection error: {error}")
            time.sleep(5)  # Backoff перед retry

        elif error.code() == KafkaError._AUTHENTICATION:
            logger.error(f"Authentication error: {error}")
            self.running = False  # Fatal error

        else:
            logger.error(f"Kafka error: {error}")

    def _handle_kafka_exception(self, exception):
        """Обработка исключений Kafka"""
        logger.error(f"Kafka exception: {exception}")
        time.sleep(1)

    def _cleanup(self):
        """Очистка ресурсов"""
        try:
            self.consumer.commit(asynchronous=False)
        except Exception:
            pass
        finally:
            self.consumer.close()
            self.dlq_producer.flush()

class RetriableError(Exception):
    pass

class NonRetriableError(Exception):
    pass

if __name__ == '__main__':
    config = {
        'bootstrap.servers': 'localhost:9092',
        'group.id': 'error-handling-python-group',
        'auto.offset.reset': 'earliest',
        'enable.auto.commit': False,
    }

    consumer = ErrorHandlingConsumer(config, 'my-topic-dlq')
    consumer.run(['my-topic'])
```

### Graceful Shutdown

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.errors.WakeupException;

import java.time.Duration;
import java.util.*;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.atomic.AtomicBoolean;

public class GracefulShutdownExample {

    private final AtomicBoolean running = new AtomicBoolean(true);
    private final CountDownLatch shutdownLatch = new CountDownLatch(1);
    private KafkaConsumer<String, String> consumer;

    public static void main(String[] args) throws InterruptedException {
        GracefulShutdownExample app = new GracefulShutdownExample();

        // Регистрация shutdown hook
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            System.out.println("Shutdown hook triggered");
            app.shutdown();
            try {
                app.awaitShutdown();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }));

        app.run();
    }

    public void run() {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "graceful-shutdown-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        consumer = new KafkaConsumer<>(props);

        try {
            consumer.subscribe(Collections.singletonList("my-topic"));

            while (running.get()) {
                ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    if (!running.get()) {
                        // Прерывание обработки при shutdown
                        break;
                    }
                    processRecord(record);
                }

                if (!records.isEmpty() && running.get()) {
                    consumer.commitSync();
                }
            }

        } catch (WakeupException e) {
            // Ожидаемое исключение при shutdown
            if (running.get()) {
                throw e;
            }
            System.out.println("Wakeup exception caught, shutting down");

        } finally {
            try {
                // Финальный коммит
                consumer.commitSync();
                System.out.println("Final commit successful");
            } catch (Exception e) {
                System.err.println("Final commit failed: " + e.getMessage());
            } finally {
                consumer.close(Duration.ofSeconds(30));
                System.out.println("Consumer closed");
                shutdownLatch.countDown();
            }
        }
    }

    public void shutdown() {
        System.out.println("Initiating shutdown...");
        running.set(false);
        consumer.wakeup(); // Прерывает poll()
    }

    public void awaitShutdown() throws InterruptedException {
        shutdownLatch.await();
        System.out.println("Shutdown complete");
    }

    private void processRecord(ConsumerRecord<String, String> record) {
        System.out.printf("Processing: partition=%d, offset=%d%n",
                record.partition(), record.offset());
    }
}
```

## Best Practices

### 1. Классификация ошибок

```java
public class ErrorClassifier {

    public static ErrorType classify(Exception e) {
        if (e instanceof RetriableException ||
            e instanceof TimeoutException ||
            e instanceof NetworkException) {
            return ErrorType.RETRIABLE;
        }

        if (e instanceof SerializationException ||
            e instanceof InvalidRecordException) {
            return ErrorType.NON_RETRIABLE;
        }

        if (e instanceof AuthorizationException ||
            e instanceof AuthenticationException) {
            return ErrorType.FATAL;
        }

        // По умолчанию — retriable
        return ErrorType.RETRIABLE;
    }
}

enum ErrorType {
    RETRIABLE,
    NON_RETRIABLE,
    FATAL
}
```

### 2. Exponential Backoff

```java
public class ExponentialBackoff {
    private final long initialInterval;
    private final double multiplier;
    private final long maxInterval;

    public ExponentialBackoff(long initialInterval, double multiplier, long maxInterval) {
        this.initialInterval = initialInterval;
        this.multiplier = multiplier;
        this.maxInterval = maxInterval;
    }

    public long getWaitTime(int attempt) {
        double wait = initialInterval * Math.pow(multiplier, attempt - 1);
        return Math.min((long) wait, maxInterval);
    }

    // Использование:
    // ExponentialBackoff backoff = new ExponentialBackoff(100, 2.0, 30000);
    // Thread.sleep(backoff.getWaitTime(attempt));
}
```

### 3. Мониторинг и алертинг

```java
// Метрики для отслеживания
public class ConsumerMetrics {
    private final MeterRegistry registry;

    private final Counter processedMessages;
    private final Counter failedMessages;
    private final Counter dlqMessages;
    private final Timer processingTime;

    public ConsumerMetrics(MeterRegistry registry) {
        this.registry = registry;
        this.processedMessages = registry.counter("kafka.consumer.messages.processed");
        this.failedMessages = registry.counter("kafka.consumer.messages.failed");
        this.dlqMessages = registry.counter("kafka.consumer.messages.dlq");
        this.processingTime = registry.timer("kafka.consumer.processing.time");
    }

    public void recordSuccess(long durationMs) {
        processedMessages.increment();
        processingTime.record(Duration.ofMillis(durationMs));
    }

    public void recordFailure() {
        failedMessages.increment();
    }

    public void recordDlq() {
        dlqMessages.increment();
    }
}
```

### 4. Идемпотентная обработка

```java
public class IdempotentProcessor {
    private final Set<String> processedIds = new ConcurrentHashMap<>().newKeySet();
    // В production используйте Redis или БД

    public boolean processIfNotDuplicate(ConsumerRecord<String, String> record) {
        String messageId = extractMessageId(record);

        if (processedIds.contains(messageId)) {
            System.out.println("Duplicate message, skipping: " + messageId);
            return true; // Считаем успехом
        }

        try {
            processMessage(record);
            processedIds.add(messageId);
            return true;
        } catch (Exception e) {
            return false;
        }
    }

    private String extractMessageId(ConsumerRecord<String, String> record) {
        // Из header или генерируем из topic+partition+offset
        return record.topic() + "-" + record.partition() + "-" + record.offset();
    }

    private void processMessage(ConsumerRecord<String, String> record) {
        // Бизнес-логика
    }
}
```

## Распространённые ошибки

### 1. Игнорирование ошибок коммита

```java
// НЕПРАВИЛЬНО
consumer.commitSync(); // Может выбросить CommitFailedException!

// ПРАВИЛЬНО
try {
    consumer.commitSync();
} catch (CommitFailedException e) {
    // Партиции были переназначены
    log.warn("Commit failed, records may be reprocessed");
}
```

### 2. Бесконечный retry

```java
// НЕПРАВИЛЬНО
while (true) {
    try {
        processRecord(record);
        break;
    } catch (Exception e) {
        // Бесконечный цикл!
    }
}

// ПРАВИЛЬНО
int maxRetries = 3;
for (int i = 0; i < maxRetries; i++) {
    try {
        processRecord(record);
        break;
    } catch (RetriableException e) {
        if (i == maxRetries - 1) {
            sendToDlq(record);
        }
    }
}
```

### 3. Потеря контекста ошибки при DLQ

```java
// НЕПРАВИЛЬНО — теряем информацию
producer.send(new ProducerRecord<>(dlqTopic, record.value()));

// ПРАВИЛЬНО — сохраняем контекст
Headers headers = new RecordHeaders();
headers.add("original-topic", record.topic().getBytes());
headers.add("original-partition", String.valueOf(record.partition()).getBytes());
headers.add("original-offset", String.valueOf(record.offset()).getBytes());
headers.add("error-message", exception.getMessage().getBytes());
headers.add("error-timestamp", String.valueOf(System.currentTimeMillis()).getBytes());

producer.send(new ProducerRecord<>(dlqTopic, null, record.key(), record.value(), headers));
```

### 4. Неправильный shutdown

```java
// НЕПРАВИЛЬНО — потеря данных
System.exit(0);

// ПРАВИЛЬНО — graceful shutdown
running.set(false);
consumer.wakeup();
// Ждём завершения poll loop
shutdownLatch.await(30, TimeUnit.SECONDS);
```

## Дополнительные ресурсы

- [Kafka Error Handling Best Practices](https://kafka.apache.org/documentation/#consumerconfigs)
- [Confluent Dead Letter Queue Pattern](https://www.confluent.io/blog/kafka-connect-dead-letter-queue/)
- [Exactly-once Semantics in Kafka](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/)
- [Spring Kafka Error Handling](https://docs.spring.io/spring-kafka/docs/current/reference/html/#error-handling)
