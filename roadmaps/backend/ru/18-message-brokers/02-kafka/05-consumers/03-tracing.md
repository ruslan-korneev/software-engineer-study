# Трассировка и Poll Loop

## Описание

Poll Loop — это центральный механизм работы Kafka Consumer, представляющий собой бесконечный цикл вызовов метода `poll()`. Этот цикл отвечает за получение сообщений, отправку heartbeat координатору группы и участие в процессе ребалансировки. Трассировка (tracing) позволяет отслеживать путь сообщений через систему и диагностировать проблемы производительности.

## Ключевые концепции

### Анатомия Poll Loop

```
┌─────────────────────────────────────────────────┐
│                  Poll Loop                      │
│                                                 │
│  ┌─────────┐   ┌─────────────┐   ┌──────────┐  │
│  │ poll()  │ → │  Обработка  │ → │  Коммит  │  │
│  └────┬────┘   └─────────────┘   └──────────┘  │
│       │                                        │
│       ▼                                        │
│  ┌─────────────────────────────────────────┐   │
│  │ Внутренние операции poll():             │   │
│  │ 1. Отправка heartbeat                   │   │
│  │ 2. Участие в ребалансировке             │   │
│  │ 3. Fetch запросы к брокерам             │   │
│  │ 4. Десериализация сообщений             │   │
│  └─────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

### Ключевые таймауты

| Параметр | Значение | Описание |
|----------|----------|----------|
| `max.poll.interval.ms` | 300000 (5 мин) | Макс. время между вызовами poll() |
| `session.timeout.ms` | 45000 | Таймаут сессии с координатором |
| `heartbeat.interval.ms` | 3000 | Интервал отправки heartbeat |
| `request.timeout.ms` | 30000 | Таймаут сетевых запросов |

### Состояния Consumer

```
UNSUBSCRIBED → SUBSCRIBING → STABLE ↔ REBALANCING → STABLE
                                ↓
                            DEAD (при таймауте)
```

## Примеры кода

### Базовый Poll Loop

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.errors.WakeupException;

import java.time.Duration;
import java.util.Collections;
import java.util.Properties;
import java.util.concurrent.atomic.AtomicBoolean;

public class BasicPollLoop {

    private final AtomicBoolean running = new AtomicBoolean(true);
    private KafkaConsumer<String, String> consumer;

    public void run() {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "poll-loop-example");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");

        // Настройки poll loop
        props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000);
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);
        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 45000);
        props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 3000);

        consumer = new KafkaConsumer<>(props);

        // Graceful shutdown
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            running.set(false);
            consumer.wakeup();
        }));

        try {
            consumer.subscribe(Collections.singletonList("events"));

            // Основной poll loop
            while (running.get()) {
                // poll() — точка входа для всех операций consumer
                ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofMillis(1000));

                // Обработка полученных записей
                for (ConsumerRecord<String, String> record : records) {
                    processRecord(record);
                }

                // Коммит после обработки
                if (!records.isEmpty()) {
                    consumer.commitSync();
                }
            }
        } catch (WakeupException e) {
            // Ожидаемое исключение при shutdown
            if (running.get()) {
                throw e;
            }
        } finally {
            consumer.close();
        }
    }

    private void processRecord(ConsumerRecord<String, String> record) {
        System.out.printf("Processing: partition=%d, offset=%d, value=%s%n",
                record.partition(), record.offset(), record.value());
    }
}
```

### Poll Loop с метриками и трассировкой

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.Metric;
import org.apache.kafka.common.MetricName;

import java.time.Duration;
import java.time.Instant;
import java.util.*;

public class TracingPollLoop {

    private KafkaConsumer<String, String> consumer;
    private long totalRecordsProcessed = 0;
    private long totalPollCalls = 0;
    private long totalProcessingTimeMs = 0;

    public void run() {
        Properties props = createConsumerProps();
        consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Collections.singletonList("events"));

        while (true) {
            // Замер времени poll
            Instant pollStart = Instant.now();
            ConsumerRecords<String, String> records =
                    consumer.poll(Duration.ofMillis(1000));
            long pollDuration = Duration.between(pollStart, Instant.now()).toMillis();

            totalPollCalls++;

            // Трассировка poll
            System.out.printf("[TRACE] Poll #%d: получено %d записей за %d мс%n",
                    totalPollCalls, records.count(), pollDuration);

            if (!records.isEmpty()) {
                // Замер времени обработки
                Instant processingStart = Instant.now();

                for (ConsumerRecord<String, String> record : records) {
                    traceRecord(record);
                    processRecord(record);
                    totalRecordsProcessed++;
                }

                long processingDuration =
                        Duration.between(processingStart, Instant.now()).toMillis();
                totalProcessingTimeMs += processingDuration;

                System.out.printf("[TRACE] Обработка: %d записей за %d мс%n",
                        records.count(), processingDuration);

                consumer.commitSync();
            }

            // Периодический вывод метрик
            if (totalPollCalls % 100 == 0) {
                printMetrics();
            }
        }
    }

    private void traceRecord(ConsumerRecord<String, String> record) {
        // Трассировка отдельной записи
        System.out.printf("[TRACE] Record: topic=%s, partition=%d, offset=%d, " +
                        "timestamp=%d, timestampType=%s, key=%s%n",
                record.topic(),
                record.partition(),
                record.offset(),
                record.timestamp(),
                record.timestampType(),
                record.key());

        // Извлечение trace headers (если используется distributed tracing)
        record.headers().forEach(header -> {
            if (header.key().startsWith("trace")) {
                System.out.printf("[TRACE] Header: %s=%s%n",
                        header.key(), new String(header.value()));
            }
        });
    }

    private void printMetrics() {
        System.out.println("\n=== Consumer Metrics ===");
        System.out.printf("Total poll calls: %d%n", totalPollCalls);
        System.out.printf("Total records processed: %d%n", totalRecordsProcessed);
        System.out.printf("Average records per poll: %.2f%n",
                (double) totalRecordsProcessed / totalPollCalls);
        System.out.printf("Total processing time: %d ms%n", totalProcessingTimeMs);

        // Kafka native метрики
        Map<MetricName, ? extends Metric> metrics = consumer.metrics();
        for (Map.Entry<MetricName, ? extends Metric> entry : metrics.entrySet()) {
            MetricName name = entry.getKey();
            if (name.name().contains("records-lag") ||
                name.name().contains("fetch-rate") ||
                name.name().contains("records-consumed-rate")) {
                System.out.printf("%s: %.2f%n", name.name(), entry.getValue().metricValue());
            }
        }
        System.out.println("========================\n");
    }

    private void processRecord(ConsumerRecord<String, String> record) {
        // Бизнес-логика
    }

    private Properties createConsumerProps() {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "tracing-consumer");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        return props;
    }
}
```

### Интеграция с OpenTelemetry

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.SpanKind;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Context;
import io.opentelemetry.context.Scope;
import io.opentelemetry.context.propagation.TextMapGetter;
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.header.Header;

import java.nio.charset.StandardCharsets;
import java.time.Duration;
import java.util.*;

public class OpenTelemetryConsumer {

    private static final Tracer tracer =
            GlobalOpenTelemetry.getTracer("kafka-consumer");

    private static final TextMapGetter<ConsumerRecord<String, String>> getter =
            new TextMapGetter<>() {
                @Override
                public Iterable<String> keys(ConsumerRecord<String, String> carrier) {
                    List<String> keys = new ArrayList<>();
                    carrier.headers().forEach(h -> keys.add(h.key()));
                    return keys;
                }

                @Override
                public String get(ConsumerRecord<String, String> carrier, String key) {
                    Header header = carrier.headers().lastHeader(key);
                    return header != null ?
                            new String(header.value(), StandardCharsets.UTF_8) : null;
                }
            };

    public void run() {
        Properties props = createConsumerProps();

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
            consumer.subscribe(Collections.singletonList("events"));

            while (true) {
                ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    // Извлечение контекста трассировки из headers
                    Context extractedContext = GlobalOpenTelemetry.getPropagators()
                            .getTextMapPropagator()
                            .extract(Context.current(), record, getter);

                    // Создание span для обработки сообщения
                    Span span = tracer.spanBuilder("process " + record.topic())
                            .setParent(extractedContext)
                            .setSpanKind(SpanKind.CONSUMER)
                            .setAttribute("messaging.system", "kafka")
                            .setAttribute("messaging.destination", record.topic())
                            .setAttribute("messaging.kafka.partition", record.partition())
                            .setAttribute("messaging.kafka.offset", record.offset())
                            .startSpan();

                    try (Scope scope = span.makeCurrent()) {
                        processRecord(record);
                    } catch (Exception e) {
                        span.recordException(e);
                        throw e;
                    } finally {
                        span.end();
                    }
                }

                consumer.commitSync();
            }
        }
    }

    private void processRecord(ConsumerRecord<String, String> record) {
        // Бизнес-логика с автоматической трассировкой
        Span currentSpan = Span.current();
        currentSpan.addEvent("Processing started");

        // ... обработка ...

        currentSpan.addEvent("Processing completed");
    }

    private Properties createConsumerProps() {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "otel-consumer");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        return props;
    }
}
```

### Poll Loop с паузой и возобновлением

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.TopicPartition;

import java.time.Duration;
import java.util.*;

public class PausableConsumer {

    private KafkaConsumer<String, String> consumer;
    private final Map<TopicPartition, Long> partitionBacklog = new HashMap<>();
    private static final int BACKLOG_THRESHOLD = 1000;

    public void run() {
        Properties props = createConsumerProps();
        consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Collections.singletonList("events"));

        while (true) {
            ConsumerRecords<String, String> records =
                    consumer.poll(Duration.ofMillis(1000));

            // Обработка по партициям
            for (TopicPartition partition : records.partitions()) {
                List<ConsumerRecord<String, String>> partitionRecords =
                        records.records(partition);

                for (ConsumerRecord<String, String> record : partitionRecords) {
                    boolean processed = processWithBackpressure(record, partition);

                    if (!processed) {
                        // Пауза партиции при перегрузке
                        System.out.printf("[TRACE] Приостановка партиции %s%n", partition);
                        consumer.pause(Collections.singleton(partition));

                        // Откат позиции к необработанной записи
                        consumer.seek(partition, record.offset());
                        break;
                    }
                }
            }

            // Проверка возможности возобновления
            checkAndResume();

            consumer.commitSync();
        }
    }

    private boolean processWithBackpressure(ConsumerRecord<String, String> record,
                                            TopicPartition partition) {
        long currentBacklog = partitionBacklog.getOrDefault(partition, 0L);

        if (currentBacklog >= BACKLOG_THRESHOLD) {
            System.out.printf("[TRACE] Backlog превышен для %s: %d%n",
                    partition, currentBacklog);
            return false;
        }

        // Увеличиваем backlog (в реальности это было бы асинхронная очередь)
        partitionBacklog.put(partition, currentBacklog + 1);

        // Имитация асинхронной обработки
        processAsync(record, partition);

        return true;
    }

    private void processAsync(ConsumerRecord<String, String> record,
                              TopicPartition partition) {
        // Асинхронная обработка с уменьшением backlog по завершении
        new Thread(() -> {
            try {
                // Обработка
                Thread.sleep(10);

                // Уменьшаем backlog
                partitionBacklog.compute(partition, (k, v) -> v != null ? v - 1 : 0);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }).start();
    }

    private void checkAndResume() {
        Set<TopicPartition> paused = consumer.paused();

        for (TopicPartition partition : paused) {
            long backlog = partitionBacklog.getOrDefault(partition, 0L);

            if (backlog < BACKLOG_THRESHOLD / 2) {
                System.out.printf("[TRACE] Возобновление партиции %s, backlog=%d%n",
                        partition, backlog);
                consumer.resume(Collections.singleton(partition));
            }
        }
    }

    private Properties createConsumerProps() {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "pausable-consumer");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        return props;
    }
}
```

### Poll Loop на Python с трассировкой

```python
from confluent_kafka import Consumer, KafkaError
import time
import logging
from dataclasses import dataclass
from typing import Optional
import json

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@dataclass
class PollMetrics:
    total_polls: int = 0
    total_records: int = 0
    total_errors: int = 0
    total_processing_time_ms: float = 0
    last_poll_duration_ms: float = 0
    last_batch_size: int = 0

class TracingConsumer:
    def __init__(self, config: dict, topics: list):
        self.consumer = Consumer(config)
        self.consumer.subscribe(topics)
        self.metrics = PollMetrics()
        self.running = True

    def run(self):
        """Основной poll loop с трассировкой"""
        logger.info("Starting consumer poll loop")

        try:
            while self.running:
                # Замер времени poll
                poll_start = time.time()
                msg = self.consumer.poll(timeout=1.0)
                poll_duration = (time.time() - poll_start) * 1000

                self.metrics.total_polls += 1
                self.metrics.last_poll_duration_ms = poll_duration

                if msg is None:
                    self._trace_empty_poll()
                    continue

                if msg.error():
                    self._handle_error(msg.error())
                    continue

                # Трассировка и обработка сообщения
                self._trace_message(msg)

                processing_start = time.time()
                self._process_message(msg)
                processing_duration = (time.time() - processing_start) * 1000

                self.metrics.total_records += 1
                self.metrics.total_processing_time_ms += processing_duration

                # Коммит после обработки
                self.consumer.commit(asynchronous=False)

                # Периодический вывод метрик
                if self.metrics.total_polls % 100 == 0:
                    self._log_metrics()

        except KeyboardInterrupt:
            logger.info("Shutdown requested")
        finally:
            self.consumer.close()
            logger.info("Consumer closed")

    def _trace_message(self, msg):
        """Трассировка отдельного сообщения"""
        trace_info = {
            'topic': msg.topic(),
            'partition': msg.partition(),
            'offset': msg.offset(),
            'timestamp': msg.timestamp(),
            'key': msg.key().decode('utf-8') if msg.key() else None,
            'headers': dict(msg.headers()) if msg.headers() else {}
        }

        logger.debug(f"[TRACE] Message received: {json.dumps(trace_info)}")

        # Извлечение trace context из headers
        if msg.headers():
            for key, value in msg.headers():
                if key.startswith('trace'):
                    logger.debug(f"[TRACE] Header {key}={value.decode('utf-8')}")

    def _trace_empty_poll(self):
        """Трассировка пустого poll"""
        logger.debug(f"[TRACE] Empty poll, duration={self.metrics.last_poll_duration_ms:.2f}ms")

    def _handle_error(self, error):
        """Обработка ошибок"""
        self.metrics.total_errors += 1

        if error.code() == KafkaError._PARTITION_EOF:
            logger.debug(f"[TRACE] Reached end of partition")
        else:
            logger.error(f"[ERROR] Kafka error: {error}")

    def _process_message(self, msg):
        """Обработка сообщения"""
        value = msg.value().decode('utf-8')
        logger.info(f"Processing: {value[:100]}...")  # Первые 100 символов

    def _log_metrics(self):
        """Вывод метрик"""
        avg_records = self.metrics.total_records / max(self.metrics.total_polls, 1)
        avg_processing = self.metrics.total_processing_time_ms / max(self.metrics.total_records, 1)

        logger.info(f"""
=== Poll Loop Metrics ===
Total polls: {self.metrics.total_polls}
Total records: {self.metrics.total_records}
Total errors: {self.metrics.total_errors}
Avg records per poll: {avg_records:.2f}
Avg processing time: {avg_processing:.2f}ms
Last poll duration: {self.metrics.last_poll_duration_ms:.2f}ms
=========================
        """)

if __name__ == '__main__':
    config = {
        'bootstrap.servers': 'localhost:9092',
        'group.id': 'tracing-python-consumer',
        'auto.offset.reset': 'earliest',
        'enable.auto.commit': False,
    }

    consumer = TracingConsumer(config, ['events'])
    consumer.run()
```

## Best Practices

### 1. Правильная настройка таймаутов

```java
// Формула: session.timeout > 3 * heartbeat.interval
props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 45000);     // 45 сек
props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 15000);  // 15 сек

// max.poll.interval должен покрывать максимальное время обработки batch
props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000);  // 5 мин

// Уменьшите max.poll.records если обработка медленная
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 100);
```

### 2. Мониторинг здоровья poll loop

```java
public class HealthMonitor {
    private volatile long lastPollTime = System.currentTimeMillis();
    private static final long POLL_TIMEOUT_WARNING = 60000; // 1 минута

    public void recordPoll() {
        lastPollTime = System.currentTimeMillis();
    }

    public boolean isHealthy() {
        long timeSinceLastPoll = System.currentTimeMillis() - lastPollTime;
        if (timeSinceLastPoll > POLL_TIMEOUT_WARNING) {
            System.err.printf("[WARNING] No poll for %d ms!%n", timeSinceLastPoll);
            return false;
        }
        return true;
    }
}
```

### 3. Обработка долгих операций

```java
// Вариант 1: Уменьшить batch size
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 10);

// Вариант 2: Увеличить max.poll.interval
props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 600000);

// Вариант 3: Асинхронная обработка с паузой
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

    if (!records.isEmpty()) {
        consumer.pause(consumer.assignment());

        CompletableFuture.runAsync(() -> processRecords(records))
            .thenRun(() -> {
                consumer.resume(consumer.assignment());
                consumer.commitSync();
            });
    }
}
```

### 4. Graceful shutdown

```java
private final AtomicBoolean running = new AtomicBoolean(true);

public void shutdown() {
    running.set(false);
    consumer.wakeup(); // Немедленно прерывает poll()
}

// В poll loop
try {
    while (running.get()) {
        ConsumerRecords<String, String> records =
                consumer.poll(Duration.ofMillis(1000));
        // обработка
    }
} catch (WakeupException e) {
    if (running.get()) throw e; // Не ожидалось
} finally {
    consumer.close(Duration.ofSeconds(30)); // Таймаут закрытия
}
```

## Распространённые ошибки

### 1. Слишком долгая обработка в poll loop

```java
// НЕПРАВИЛЬНО - превышение max.poll.interval.ms
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    for (ConsumerRecord<String, String> record : records) {
        Thread.sleep(60000); // 60 секунд на запись!
        // Consumer будет исключён из группы
    }
}

// ПРАВИЛЬНО - контроль времени обработки
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    long batchStart = System.currentTimeMillis();

    for (ConsumerRecord<String, String> record : records) {
        processRecord(record);

        // Проверка времени
        if (System.currentTimeMillis() - batchStart > 240000) { // 4 мин
            // Коммит текущей позиции и продолжение
            consumer.commitSync();
            break;
        }
    }
}
```

### 2. Игнорирование poll() при отсутствии данных

```java
// НЕПРАВИЛЬНО - пропуск poll() при пустом результате
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    if (records.isEmpty()) {
        Thread.sleep(5000); // НЕ вызывайте poll() редко!
        continue;
    }
    // обработка
}

// ПРАВИЛЬНО - poll() должен вызываться регулярно
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    if (!records.isEmpty()) {
        // обработка
    }
    // poll() вызывается постоянно для heartbeat
}
```

### 3. Блокировка в poll loop

```java
// НЕПРАВИЛЬНО - блокирующие операции
while (true) {
    records = consumer.poll(Duration.ofMillis(1000));
    for (ConsumerRecord<String, String> record : records) {
        // Блокирующий HTTP вызов
        httpClient.post(record.value()).get(); // Может зависнуть!
    }
}

// ПРАВИЛЬНО - асинхронная обработка или таймауты
while (true) {
    records = consumer.poll(Duration.ofMillis(1000));
    for (ConsumerRecord<String, String> record : records) {
        httpClient.post(record.value())
                  .orTimeout(5, TimeUnit.SECONDS)
                  .exceptionally(e -> handleTimeout(e));
    }
}
```

### 4. Отсутствие обработки WakeupException

```java
// НЕПРАВИЛЬНО - WakeupException не обработан
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    // При вызове wakeup() будет необработанное исключение
}

// ПРАВИЛЬНО
try {
    while (running.get()) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
        // обработка
    }
} catch (WakeupException e) {
    // Игнорируем если это запланированный shutdown
    if (running.get()) {
        throw e;
    }
} finally {
    consumer.close();
}
```

## Метрики для мониторинга

| Метрика | Описание | Пороговое значение |
|---------|----------|-------------------|
| `records-lag-max` | Максимальный лаг | < 10000 |
| `records-consumed-rate` | Скорость потребления | Зависит от нагрузки |
| `fetch-rate` | Частота fetch запросов | > 0 |
| `poll-idle-ratio-avg` | Доля времени ожидания | 0.5 - 0.8 |
| `time-between-poll-avg` | Среднее время между poll | < max.poll.interval.ms / 2 |

## Дополнительные ресурсы

- [Kafka Consumer Poll Behavior](https://kafka.apache.org/documentation/#consumerconfigs)
- [OpenTelemetry Kafka Instrumentation](https://opentelemetry.io/docs/instrumentation/java/automatic/spring-boot/)
- [Confluent Consumer Configuration](https://docs.confluent.io/platform/current/installation/configuration/consumer-configs.html)
