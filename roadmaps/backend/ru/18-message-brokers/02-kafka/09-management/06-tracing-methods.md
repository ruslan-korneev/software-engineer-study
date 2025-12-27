# Способы трассировки в Kafka

## Описание

Трассировка (tracing) в Apache Kafka позволяет отслеживать путь сообщений через распределённую систему, измерять латентность на каждом этапе обработки и диагностировать проблемы производительности. Это критически важно для микросервисных архитектур, где одно сообщение может проходить через множество сервисов.

Существует несколько подходов к трассировке: добавление заголовков в сообщения, интеграция с системами распределённой трассировки (OpenTelemetry, Jaeger, Zipkin) и использование специализированных инструментов.

## Ключевые концепции

### Типы трассировки

| Тип | Описание | Применение |
|-----|----------|------------|
| Message tracing | Отслеживание пути сообщения | End-to-end мониторинг |
| Request tracing | Трассировка запросов к брокеру | Диагностика производительности |
| Distributed tracing | Трассировка через микросервисы | Комплексные системы |
| Event correlation | Связывание событий | Анализ потоков данных |

### Ключевые понятия

| Термин | Описание |
|--------|----------|
| Trace | Полный путь запроса через систему |
| Span | Отдельная операция в trace |
| Trace ID | Уникальный идентификатор трассировки |
| Span ID | Идентификатор отдельной операции |
| Parent Span ID | Ссылка на родительскую операцию |
| Baggage | Контекстные данные, передаваемые между сервисами |

## Примеры

### Добавление заголовков трассировки

```java
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.header.Headers;
import java.util.UUID;

public class TracingProducer {

    private final KafkaProducer<String, String> producer;

    public TracingProducer(Properties props) {
        this.producer = new KafkaProducer<>(props);
    }

    public void sendWithTracing(String topic, String key, String value,
                                String traceId, String parentSpanId) {
        String spanId = UUID.randomUUID().toString().substring(0, 16);

        ProducerRecord<String, String> record = new ProducerRecord<>(topic, key, value);

        // Добавление заголовков трассировки
        Headers headers = record.headers();
        headers.add("trace-id", traceId.getBytes());
        headers.add("span-id", spanId.getBytes());
        headers.add("parent-span-id", parentSpanId.getBytes());
        headers.add("timestamp", String.valueOf(System.currentTimeMillis()).getBytes());
        headers.add("producer-id", "order-service".getBytes());

        producer.send(record, (metadata, exception) -> {
            if (exception == null) {
                System.out.printf("Sent message with trace-id=%s to partition=%d offset=%d%n",
                    traceId, metadata.partition(), metadata.offset());
            } else {
                System.err.println("Error sending message: " + exception.getMessage());
            }
        });
    }
}
```

### Извлечение заголовков в консьюмере

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.header.Header;
import java.nio.charset.StandardCharsets;
import java.util.*;

public class TracingConsumer {

    private final KafkaConsumer<String, String> consumer;

    public TracingConsumer(Properties props) {
        this.consumer = new KafkaConsumer<>(props);
    }

    public void consumeWithTracing(String topic) {
        consumer.subscribe(Collections.singletonList(topic));

        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

            for (ConsumerRecord<String, String> record : records) {
                // Извлечение заголовков трассировки
                String traceId = getHeader(record, "trace-id");
                String spanId = getHeader(record, "span-id");
                String parentSpanId = getHeader(record, "parent-span-id");
                String producerTimestamp = getHeader(record, "timestamp");

                // Вычисление латентности
                long latency = System.currentTimeMillis() - Long.parseLong(producerTimestamp);

                System.out.printf(
                    "Received message: trace-id=%s, span-id=%s, latency=%dms%n",
                    traceId, spanId, latency
                );

                // Создание нового span для обработки
                String newSpanId = UUID.randomUUID().toString().substring(0, 16);
                processMessage(record, traceId, newSpanId, spanId);
            }
        }
    }

    private String getHeader(ConsumerRecord<String, String> record, String key) {
        Header header = record.headers().lastHeader(key);
        return header != null ? new String(header.value(), StandardCharsets.UTF_8) : "";
    }

    private void processMessage(ConsumerRecord<String, String> record,
                                String traceId, String spanId, String parentSpanId) {
        // Обработка сообщения
    }
}
```

### Интеграция с OpenTelemetry

```java
import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.trace.*;
import io.opentelemetry.context.Context;
import io.opentelemetry.context.propagation.TextMapPropagator;
import io.opentelemetry.context.propagation.TextMapSetter;
import io.opentelemetry.context.propagation.TextMapGetter;
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.header.Headers;

public class OpenTelemetryKafkaTracing {

    private final OpenTelemetry openTelemetry;
    private final Tracer tracer;
    private final TextMapPropagator propagator;

    public OpenTelemetryKafkaTracing(OpenTelemetry openTelemetry) {
        this.openTelemetry = openTelemetry;
        this.tracer = openTelemetry.getTracer("kafka-producer");
        this.propagator = openTelemetry.getPropagators().getTextMapPropagator();
    }

    // TextMapSetter для заголовков Kafka
    private static final TextMapSetter<Headers> SETTER = (headers, key, value) -> {
        headers.remove(key);
        headers.add(key, value.getBytes());
    };

    // TextMapGetter для заголовков Kafka
    private static final TextMapGetter<Headers> GETTER = new TextMapGetter<>() {
        @Override
        public Iterable<String> keys(Headers headers) {
            List<String> keys = new ArrayList<>();
            headers.forEach(h -> keys.add(h.key()));
            return keys;
        }

        @Override
        public String get(Headers headers, String key) {
            var header = headers.lastHeader(key);
            return header != null ? new String(header.value()) : null;
        }
    };

    public void sendWithTracing(KafkaProducer<String, String> producer,
                                String topic, String key, String value) {
        // Создание span
        Span span = tracer.spanBuilder("kafka.produce")
            .setSpanKind(SpanKind.PRODUCER)
            .setAttribute("messaging.system", "kafka")
            .setAttribute("messaging.destination", topic)
            .setAttribute("messaging.destination_kind", "topic")
            .startSpan();

        try (var scope = span.makeCurrent()) {
            ProducerRecord<String, String> record = new ProducerRecord<>(topic, key, value);

            // Инъекция контекста трассировки в заголовки
            propagator.inject(Context.current(), record.headers(), SETTER);

            producer.send(record, (metadata, exception) -> {
                if (exception != null) {
                    span.recordException(exception);
                    span.setStatus(StatusCode.ERROR);
                } else {
                    span.setAttribute("messaging.kafka.partition", metadata.partition());
                    span.setAttribute("messaging.kafka.offset", metadata.offset());
                }
                span.end();
            });
        }
    }

    public void consumeWithTracing(ConsumerRecord<String, String> record) {
        // Извлечение контекста трассировки из заголовков
        Context extractedContext = propagator.extract(
            Context.current(),
            record.headers(),
            GETTER
        );

        // Создание span с родительским контекстом
        Span span = tracer.spanBuilder("kafka.consume")
            .setParent(extractedContext)
            .setSpanKind(SpanKind.CONSUMER)
            .setAttribute("messaging.system", "kafka")
            .setAttribute("messaging.destination", record.topic())
            .setAttribute("messaging.kafka.partition", record.partition())
            .setAttribute("messaging.kafka.offset", record.offset())
            .startSpan();

        try (var scope = span.makeCurrent()) {
            // Обработка сообщения
            processMessage(record);
            span.setStatus(StatusCode.OK);
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR);
            throw e;
        } finally {
            span.end();
        }
    }

    private void processMessage(ConsumerRecord<String, String> record) {
        // Бизнес-логика
    }
}
```

### Конфигурация OpenTelemetry

```java
import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.exporter.jaeger.JaegerGrpcSpanExporter;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;
import io.opentelemetry.sdk.resources.Resource;
import io.opentelemetry.semconv.resource.attributes.ResourceAttributes;

public class OpenTelemetryConfig {

    public static OpenTelemetry configure() {
        // Экспортер в Jaeger
        JaegerGrpcSpanExporter jaegerExporter = JaegerGrpcSpanExporter.builder()
            .setEndpoint("http://jaeger:14250")
            .build();

        // Ресурс с метаданными сервиса
        Resource resource = Resource.getDefault()
            .merge(Resource.create(
                io.opentelemetry.api.common.Attributes.of(
                    ResourceAttributes.SERVICE_NAME, "kafka-service",
                    ResourceAttributes.SERVICE_VERSION, "1.0.0"
                )
            ));

        // Провайдер трассировки
        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
            .addSpanProcessor(BatchSpanProcessor.builder(jaegerExporter).build())
            .setResource(resource)
            .build();

        return OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .buildAndRegisterGlobal();
    }
}
```

### Интеграция с Zipkin через Brave

```java
import brave.Tracing;
import brave.kafka.clients.KafkaTracing;
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.clients.consumer.*;
import zipkin2.reporter.brave.AsyncZipkinSpanHandler;
import zipkin2.reporter.okhttp3.OkHttpSender;

public class ZipkinKafkaTracing {

    private final KafkaTracing kafkaTracing;

    public ZipkinKafkaTracing() {
        // Настройка отправителя в Zipkin
        var sender = OkHttpSender.create("http://zipkin:9411/api/v2/spans");
        var spanHandler = AsyncZipkinSpanHandler.create(sender);

        // Настройка Brave
        Tracing tracing = Tracing.newBuilder()
            .localServiceName("kafka-service")
            .addSpanHandler(spanHandler)
            .build();

        // Создание KafkaTracing
        this.kafkaTracing = KafkaTracing.create(tracing);
    }

    public Producer<String, String> createTracingProducer(Properties props) {
        KafkaProducer<String, String> producer = new KafkaProducer<>(props);
        return kafkaTracing.producer(producer);
    }

    public Consumer<String, String> createTracingConsumer(Properties props) {
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        return kafkaTracing.consumer(consumer);
    }
}
```

### Spring Boot интеграция

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.*;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import io.micrometer.tracing.Tracer;
import io.micrometer.tracing.propagation.Propagator;

@Configuration
public class KafkaTracingConfig {

    @Bean
    public ProducerFactory<String, String> producerFactory(Tracer tracer) {
        Map<String, Object> configs = new HashMap<>();
        configs.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        configs.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configs.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

        // Добавление интерцептора для трассировки
        configs.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG,
            TracingProducerInterceptor.class.getName());

        return new DefaultKafkaProducerFactory<>(configs);
    }

    @Bean
    public ConsumerFactory<String, String> consumerFactory(Tracer tracer) {
        Map<String, Object> configs = new HashMap<>();
        configs.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        configs.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        configs.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        configs.put(ConsumerConfig.GROUP_ID_CONFIG, "tracing-consumer-group");

        // Добавление интерцептора для трассировки
        configs.put(ConsumerConfig.INTERCEPTOR_CLASSES_CONFIG,
            TracingConsumerInterceptor.class.getName());

        return new DefaultKafkaConsumerFactory<>(configs);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory(
            ConsumerFactory<String, String> consumerFactory) {
        ConcurrentKafkaListenerContainerFactory<String, String> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        return factory;
    }
}
```

### Кастомные интерцепторы для трассировки

```java
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.clients.consumer.*;
import java.util.*;

public class TracingProducerInterceptor<K, V> implements ProducerInterceptor<K, V> {

    @Override
    public ProducerRecord<K, V> onSend(ProducerRecord<K, V> record) {
        // Добавление traceId если его нет
        if (record.headers().lastHeader("trace-id") == null) {
            String traceId = UUID.randomUUID().toString();
            record.headers().add("trace-id", traceId.getBytes());
        }

        // Добавление timestamp
        record.headers().add("send-timestamp",
            String.valueOf(System.currentTimeMillis()).getBytes());

        return record;
    }

    @Override
    public void onAcknowledgement(RecordMetadata metadata, Exception exception) {
        if (exception != null) {
            System.err.println("Error sending record: " + exception.getMessage());
        }
    }

    @Override
    public void close() {}

    @Override
    public void configure(Map<String, ?> configs) {}
}

public class TracingConsumerInterceptor<K, V> implements ConsumerInterceptor<K, V> {

    @Override
    public ConsumerRecords<K, V> onConsume(ConsumerRecords<K, V> records) {
        for (ConsumerRecord<K, V> record : records) {
            var sendTimestampHeader = record.headers().lastHeader("send-timestamp");
            if (sendTimestampHeader != null) {
                long sendTimestamp = Long.parseLong(new String(sendTimestampHeader.value()));
                long latency = System.currentTimeMillis() - sendTimestamp;

                System.out.printf("Message latency: %dms, topic=%s, partition=%d, offset=%d%n",
                    latency, record.topic(), record.partition(), record.offset());
            }
        }
        return records;
    }

    @Override
    public void onCommit(Map<TopicPartition, OffsetAndMetadata> offsets) {}

    @Override
    public void close() {}

    @Override
    public void configure(Map<String, ?> configs) {}
}
```

### Корреляция событий

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class EventCorrelator {

    private final Map<String, List<TraceEvent>> traceEvents = new ConcurrentHashMap<>();

    public void recordEvent(String traceId, TraceEvent event) {
        traceEvents.computeIfAbsent(traceId, k -> new ArrayList<>()).add(event);
    }

    public List<TraceEvent> getTrace(String traceId) {
        return traceEvents.getOrDefault(traceId, Collections.emptyList());
    }

    public TraceAnalysis analyzeTrace(String traceId) {
        List<TraceEvent> events = getTrace(traceId);
        if (events.isEmpty()) {
            return null;
        }

        events.sort(Comparator.comparing(TraceEvent::getTimestamp));

        long startTime = events.get(0).getTimestamp();
        long endTime = events.get(events.size() - 1).getTimestamp();
        long totalDuration = endTime - startTime;

        Map<String, Long> serviceDurations = new HashMap<>();
        for (int i = 0; i < events.size() - 1; i++) {
            TraceEvent current = events.get(i);
            TraceEvent next = events.get(i + 1);
            long duration = next.getTimestamp() - current.getTimestamp();
            serviceDurations.merge(current.getService(), duration, Long::sum);
        }

        return new TraceAnalysis(traceId, totalDuration, serviceDurations, events.size());
    }

    public static class TraceEvent {
        private final String spanId;
        private final String service;
        private final String operation;
        private final long timestamp;

        public TraceEvent(String spanId, String service, String operation) {
            this.spanId = spanId;
            this.service = service;
            this.operation = operation;
            this.timestamp = System.currentTimeMillis();
        }

        // Getters
        public String getSpanId() { return spanId; }
        public String getService() { return service; }
        public String getOperation() { return operation; }
        public long getTimestamp() { return timestamp; }
    }

    public static class TraceAnalysis {
        private final String traceId;
        private final long totalDuration;
        private final Map<String, Long> serviceDurations;
        private final int eventCount;

        public TraceAnalysis(String traceId, long totalDuration,
                           Map<String, Long> serviceDurations, int eventCount) {
            this.traceId = traceId;
            this.totalDuration = totalDuration;
            this.serviceDurations = serviceDurations;
            this.eventCount = eventCount;
        }

        // Getters
        public String getTraceId() { return traceId; }
        public long getTotalDuration() { return totalDuration; }
        public Map<String, Long> getServiceDurations() { return serviceDurations; }
        public int getEventCount() { return eventCount; }
    }
}
```

### Docker Compose для инфраструктуры трассировки

```yaml
# docker-compose-tracing.yml

version: '3.8'
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "6831:6831/udp"  # Thrift compact protocol
      - "6832:6832/udp"  # Thrift binary protocol
      - "5778:5778"      # Config
      - "16686:16686"    # UI
      - "14250:14250"    # gRPC
      - "14268:14268"    # HTTP collector
    environment:
      - COLLECTOR_OTLP_ENABLED=true

  zipkin:
    image: openzipkin/zipkin:latest
    ports:
      - "9411:9411"

  otel-collector:
    image: otel/opentelemetry-collector:latest
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8888:8888"   # Prometheus metrics
    depends_on:
      - jaeger
```

```yaml
# otel-collector-config.yaml

receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024

exporters:
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true

  logging:
    loglevel: debug

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [jaeger, logging]
```

## Best Practices

### Рекомендации по трассировке

1. **Используйте стандартные заголовки** (W3C Trace Context) для совместимости
2. **Минимизируйте оверхед** — не добавляйте избыточные данные в заголовки
3. **Семплирование** — используйте семплирование для высоконагруженных систем
4. **Храните trace данные** отдельно от бизнес-данных

### Семплирование

```java
// Семплирование на основе rate
public class RateSampler {
    private final double samplingRate;
    private final Random random = new Random();

    public RateSampler(double samplingRate) {
        this.samplingRate = samplingRate;
    }

    public boolean shouldSample() {
        return random.nextDouble() < samplingRate;
    }
}
```

### Стандартные заголовки W3C

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
tracestate: vendor1=value1,vendor2=value2
```

### Мониторинг производительности трассировки

```java
// Метрики трассировки
- trace_count_total — количество трасс
- span_duration_seconds — длительность spans
- trace_sampling_decisions — решения семплирования
- tracing_overhead_seconds — накладные расходы на трассировку
```
