# События датчиков

## Описание

События датчиков (Sensor Events) представляют собой типичный use-case для Apache Kafka в IoT и промышленных системах. Датчики генерируют непрерывный поток данных, который необходимо собирать, обрабатывать и анализировать в реальном времени. Kafka идеально подходит для этой задачи благодаря высокой пропускной способности, надежности и возможности масштабирования.

## Архитектура системы обработки событий датчиков

```
┌─────────────┐     ┌─────────────┐     ┌─────────────────┐
│  Датчик 1   │────▶│             │     │                 │
├─────────────┤     │             │     │   Consumer 1    │
│  Датчик 2   │────▶│    Kafka    │────▶│   (Аналитика)   │
├─────────────┤     │   Cluster   │     │                 │
│  Датчик N   │────▶│             │     ├─────────────────┤
└─────────────┘     │             │────▶│   Consumer 2    │
                    └─────────────┘     │   (Оповещения)  │
                                        └─────────────────┘
```

## Модели данных

### Базовая модель события датчика

```python
# src/models/events.py
from dataclasses import dataclass, field, asdict
from datetime import datetime
from typing import Optional, Dict, Any
from enum import Enum
import json
import uuid


class SensorType(Enum):
    """Типы датчиков."""
    TEMPERATURE = "temperature"
    HUMIDITY = "humidity"
    PRESSURE = "pressure"
    MOTION = "motion"
    LIGHT = "light"
    CO2 = "co2"
    NOISE = "noise"
    VIBRATION = "vibration"


class EventPriority(Enum):
    """Приоритет события."""
    LOW = "low"
    NORMAL = "normal"
    HIGH = "high"
    CRITICAL = "critical"


@dataclass
class SensorEvent:
    """Событие датчика."""

    sensor_id: str
    sensor_type: SensorType
    value: float
    unit: str
    timestamp: datetime = field(default_factory=datetime.utcnow)
    event_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    location: Optional[str] = None
    metadata: Dict[str, Any] = field(default_factory=dict)

    def to_dict(self) -> dict:
        """Преобразование в словарь."""
        return {
            "event_id": self.event_id,
            "sensor_id": self.sensor_id,
            "sensor_type": self.sensor_type.value,
            "value": self.value,
            "unit": self.unit,
            "timestamp": self.timestamp.isoformat(),
            "location": self.location,
            "metadata": self.metadata,
        }

    def to_json(self) -> str:
        """Сериализация в JSON."""
        return json.dumps(self.to_dict())

    @classmethod
    def from_dict(cls, data: dict) -> "SensorEvent":
        """Создание из словаря."""
        return cls(
            event_id=data["event_id"],
            sensor_id=data["sensor_id"],
            sensor_type=SensorType(data["sensor_type"]),
            value=data["value"],
            unit=data["unit"],
            timestamp=datetime.fromisoformat(data["timestamp"]),
            location=data.get("location"),
            metadata=data.get("metadata", {}),
        )

    @classmethod
    def from_json(cls, json_str: str) -> "SensorEvent":
        """Десериализация из JSON."""
        return cls.from_dict(json.loads(json_str))


@dataclass
class AggregatedSensorEvent:
    """Агрегированные данные датчика."""

    sensor_id: str
    sensor_type: SensorType
    avg_value: float
    min_value: float
    max_value: float
    count: int
    window_start: datetime
    window_end: datetime
    unit: str

    def to_dict(self) -> dict:
        return {
            "sensor_id": self.sensor_id,
            "sensor_type": self.sensor_type.value,
            "avg_value": self.avg_value,
            "min_value": self.min_value,
            "max_value": self.max_value,
            "count": self.count,
            "window_start": self.window_start.isoformat(),
            "window_end": self.window_end.isoformat(),
            "unit": self.unit,
        }


@dataclass
class SensorAlert:
    """Оповещение от датчика."""

    alert_id: str
    sensor_id: str
    sensor_type: SensorType
    priority: EventPriority
    message: str
    value: float
    threshold: float
    timestamp: datetime = field(default_factory=datetime.utcnow)
    acknowledged: bool = False

    def to_dict(self) -> dict:
        return {
            "alert_id": self.alert_id,
            "sensor_id": self.sensor_id,
            "sensor_type": self.sensor_type.value,
            "priority": self.priority.value,
            "message": self.message,
            "value": self.value,
            "threshold": self.threshold,
            "timestamp": self.timestamp.isoformat(),
            "acknowledged": self.acknowledged,
        }
```

## Producer для событий датчиков

### Базовый Producer

```python
# src/producers/base_producer.py
from abc import ABC, abstractmethod
from typing import Optional, Callable
from confluent_kafka import Producer
import logging

logger = logging.getLogger(__name__)


class BaseProducer(ABC):
    """Базовый класс для Kafka Producer."""

    def __init__(self, config: dict):
        self.config = config
        self._producer: Optional[Producer] = None

    @property
    def producer(self) -> Producer:
        """Ленивая инициализация producer."""
        if self._producer is None:
            self._producer = Producer(self.config)
        return self._producer

    def delivery_callback(self, err, msg) -> None:
        """Callback для подтверждения доставки."""
        if err:
            logger.error(f"Ошибка доставки сообщения: {err}")
        else:
            logger.debug(
                f"Сообщение доставлено в {msg.topic()} "
                f"[{msg.partition()}] @ offset {msg.offset()}"
            )

    @abstractmethod
    def send(self, topic: str, key: str, value: bytes) -> None:
        """Отправка сообщения."""
        pass

    def flush(self, timeout: float = 10.0) -> int:
        """Ожидание отправки всех сообщений."""
        return self.producer.flush(timeout)

    def close(self) -> None:
        """Закрытие producer."""
        if self._producer:
            self._producer.flush()
            self._producer = None
```

### Producer для датчиков

```python
# src/producers/sensor_producer.py
import json
import random
import time
from datetime import datetime
from typing import List, Optional
from confluent_kafka import Producer
import logging

from src.models.events import SensorEvent, SensorType
from src.producers.base_producer import BaseProducer
from src.config.kafka_config import get_config

logger = logging.getLogger(__name__)


class SensorProducer(BaseProducer):
    """Producer для отправки событий датчиков."""

    def __init__(self, config: dict, topic: str = "sensor-events"):
        super().__init__(config)
        self.topic = topic

    def send(self, topic: str, key: str, value: bytes) -> None:
        """Отправка сообщения в Kafka."""
        self.producer.produce(
            topic=topic,
            key=key.encode("utf-8"),
            value=value,
            callback=self.delivery_callback,
        )
        # Poll для обработки callbacks
        self.producer.poll(0)

    def send_event(self, event: SensorEvent) -> None:
        """Отправка события датчика."""
        key = f"{event.sensor_type.value}:{event.sensor_id}"
        value = event.to_json().encode("utf-8")

        self.send(self.topic, key, value)
        logger.info(f"Отправлено событие: {event.sensor_id} = {event.value}")

    def send_batch(self, events: List[SensorEvent]) -> None:
        """Отправка пакета событий."""
        for event in events:
            self.send_event(event)

        # Ожидаем доставки всех сообщений
        remaining = self.producer.flush(timeout=10.0)
        if remaining > 0:
            logger.warning(f"{remaining} сообщений не доставлено")


class SensorSimulator:
    """Симулятор датчиков для тестирования."""

    def __init__(self, producer: SensorProducer):
        self.producer = producer
        self.sensors = self._init_sensors()

    def _init_sensors(self) -> List[dict]:
        """Инициализация виртуальных датчиков."""
        return [
            {"id": "temp-001", "type": SensorType.TEMPERATURE, "unit": "°C", "base": 22, "variance": 5},
            {"id": "temp-002", "type": SensorType.TEMPERATURE, "unit": "°C", "base": 20, "variance": 3},
            {"id": "hum-001", "type": SensorType.HUMIDITY, "unit": "%", "base": 50, "variance": 15},
            {"id": "press-001", "type": SensorType.PRESSURE, "unit": "hPa", "base": 1013, "variance": 20},
            {"id": "co2-001", "type": SensorType.CO2, "unit": "ppm", "base": 400, "variance": 100},
            {"id": "light-001", "type": SensorType.LIGHT, "unit": "lux", "base": 500, "variance": 200},
        ]

    def generate_event(self, sensor: dict) -> SensorEvent:
        """Генерация события для датчика."""
        value = sensor["base"] + random.uniform(-sensor["variance"], sensor["variance"])

        return SensorEvent(
            sensor_id=sensor["id"],
            sensor_type=sensor["type"],
            value=round(value, 2),
            unit=sensor["unit"],
            location="Building A, Floor 1",
            metadata={
                "firmware_version": "1.2.3",
                "battery_level": random.randint(60, 100),
            }
        )

    def run(self, interval: float = 1.0, duration: Optional[float] = None) -> None:
        """Запуск симуляции."""
        logger.info("Запуск симуляции датчиков...")
        start_time = time.time()

        try:
            while True:
                if duration and (time.time() - start_time) > duration:
                    break

                for sensor in self.sensors:
                    event = self.generate_event(sensor)
                    self.producer.send_event(event)

                time.sleep(interval)

        except KeyboardInterrupt:
            logger.info("Симуляция остановлена пользователем")
        finally:
            self.producer.close()


def main():
    """Точка входа для Producer."""
    import logging
    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s - %(name)s - %(levelname)s - %(message)s"
    )

    config = get_config()
    producer = SensorProducer(config.to_producer_config())
    simulator = SensorSimulator(producer)

    # Запуск симуляции на 60 секунд
    simulator.run(interval=0.5, duration=60)


if __name__ == "__main__":
    main()
```

## Consumer для событий датчиков

### Базовый Consumer

```python
# src/consumers/base_consumer.py
from abc import ABC, abstractmethod
from typing import Optional, Callable, List
from confluent_kafka import Consumer, KafkaError, KafkaException
import logging
import signal

logger = logging.getLogger(__name__)


class BaseConsumer(ABC):
    """Базовый класс для Kafka Consumer."""

    def __init__(self, config: dict, topics: List[str]):
        self.config = config
        self.topics = topics
        self._consumer: Optional[Consumer] = None
        self._running = False

    @property
    def consumer(self) -> Consumer:
        """Ленивая инициализация consumer."""
        if self._consumer is None:
            self._consumer = Consumer(self.config)
        return self._consumer

    @abstractmethod
    def process_message(self, key: str, value: bytes, headers: dict) -> None:
        """Обработка сообщения - реализуется в подклассах."""
        pass

    def start(self) -> None:
        """Запуск consumer."""
        self.consumer.subscribe(self.topics)
        self._running = True

        # Обработка сигналов для graceful shutdown
        signal.signal(signal.SIGINT, self._signal_handler)
        signal.signal(signal.SIGTERM, self._signal_handler)

        logger.info(f"Consumer подписан на топики: {self.topics}")
        self._consume_loop()

    def _signal_handler(self, signum, frame) -> None:
        """Обработчик сигналов."""
        logger.info("Получен сигнал остановки...")
        self._running = False

    def _consume_loop(self) -> None:
        """Основной цикл потребления сообщений."""
        try:
            while self._running:
                msg = self.consumer.poll(timeout=1.0)

                if msg is None:
                    continue

                if msg.error():
                    if msg.error().code() == KafkaError._PARTITION_EOF:
                        logger.debug(
                            f"Достигнут конец партиции {msg.topic()}"
                            f"[{msg.partition()}]"
                        )
                    else:
                        raise KafkaException(msg.error())
                else:
                    # Обработка сообщения
                    key = msg.key().decode("utf-8") if msg.key() else None
                    value = msg.value()
                    headers = dict(msg.headers()) if msg.headers() else {}

                    try:
                        self.process_message(key, value, headers)
                    except Exception as e:
                        logger.error(f"Ошибка обработки сообщения: {e}")
                        # Здесь можно добавить логику Dead Letter Queue

        except KafkaException as e:
            logger.error(f"Ошибка Kafka: {e}")
        finally:
            self.close()

    def close(self) -> None:
        """Закрытие consumer."""
        if self._consumer:
            self._consumer.close()
            self._consumer = None
            logger.info("Consumer закрыт")
```

### Consumer для обработки событий датчиков

```python
# src/consumers/sensor_consumer.py
import json
from typing import Dict, List, Callable, Optional
from datetime import datetime, timedelta
from collections import defaultdict
from dataclasses import dataclass
import logging

from src.models.events import SensorEvent, SensorAlert, EventPriority, SensorType
from src.consumers.base_consumer import BaseConsumer
from src.config.kafka_config import get_config

logger = logging.getLogger(__name__)


@dataclass
class ThresholdRule:
    """Правило для порогового значения."""
    sensor_type: SensorType
    min_value: Optional[float] = None
    max_value: Optional[float] = None
    priority: EventPriority = EventPriority.HIGH


class SensorConsumer(BaseConsumer):
    """Consumer для обработки событий датчиков."""

    def __init__(
        self,
        config: dict,
        topics: List[str],
        alert_callback: Optional[Callable[[SensorAlert], None]] = None
    ):
        super().__init__(config, topics)
        self.alert_callback = alert_callback
        self.thresholds = self._init_thresholds()
        self.event_buffer: Dict[str, List[SensorEvent]] = defaultdict(list)
        self.stats = defaultdict(lambda: {"count": 0, "sum": 0, "min": float("inf"), "max": float("-inf")})

    def _init_thresholds(self) -> List[ThresholdRule]:
        """Инициализация правил для оповещений."""
        return [
            ThresholdRule(SensorType.TEMPERATURE, min_value=10, max_value=35, priority=EventPriority.HIGH),
            ThresholdRule(SensorType.HUMIDITY, min_value=20, max_value=80, priority=EventPriority.NORMAL),
            ThresholdRule(SensorType.CO2, max_value=1000, priority=EventPriority.CRITICAL),
            ThresholdRule(SensorType.PRESSURE, min_value=980, max_value=1050, priority=EventPriority.NORMAL),
        ]

    def process_message(self, key: str, value: bytes, headers: dict) -> None:
        """Обработка события датчика."""
        try:
            event = SensorEvent.from_json(value.decode("utf-8"))

            # Логирование события
            logger.debug(
                f"Получено событие: {event.sensor_id} = {event.value} {event.unit}"
            )

            # Обновление статистики
            self._update_stats(event)

            # Проверка пороговых значений
            self._check_thresholds(event)

            # Добавление в буфер для агрегации
            self.event_buffer[event.sensor_id].append(event)

            # Периодическая агрегация
            if len(self.event_buffer[event.sensor_id]) >= 100:
                self._aggregate_events(event.sensor_id)

        except json.JSONDecodeError as e:
            logger.error(f"Ошибка парсинга JSON: {e}")
        except Exception as e:
            logger.error(f"Ошибка обработки события: {e}")

    def _update_stats(self, event: SensorEvent) -> None:
        """Обновление статистики по датчику."""
        stats = self.stats[event.sensor_id]
        stats["count"] += 1
        stats["sum"] += event.value
        stats["min"] = min(stats["min"], event.value)
        stats["max"] = max(stats["max"], event.value)
        stats["last_value"] = event.value
        stats["last_update"] = event.timestamp

    def _check_thresholds(self, event: SensorEvent) -> None:
        """Проверка пороговых значений и генерация оповещений."""
        for rule in self.thresholds:
            if rule.sensor_type != event.sensor_type:
                continue

            alert = None

            if rule.min_value is not None and event.value < rule.min_value:
                alert = SensorAlert(
                    alert_id=f"alert-{event.event_id}",
                    sensor_id=event.sensor_id,
                    sensor_type=event.sensor_type,
                    priority=rule.priority,
                    message=f"Значение {event.value} ниже минимального порога {rule.min_value}",
                    value=event.value,
                    threshold=rule.min_value,
                )

            elif rule.max_value is not None and event.value > rule.max_value:
                alert = SensorAlert(
                    alert_id=f"alert-{event.event_id}",
                    sensor_id=event.sensor_id,
                    sensor_type=event.sensor_type,
                    priority=rule.priority,
                    message=f"Значение {event.value} превышает максимальный порог {rule.max_value}",
                    value=event.value,
                    threshold=rule.max_value,
                )

            if alert:
                logger.warning(f"ALERT: {alert.message}")
                if self.alert_callback:
                    self.alert_callback(alert)

    def _aggregate_events(self, sensor_id: str) -> None:
        """Агрегация событий для датчика."""
        events = self.event_buffer[sensor_id]
        if not events:
            return

        values = [e.value for e in events]

        aggregated = {
            "sensor_id": sensor_id,
            "count": len(values),
            "avg": sum(values) / len(values),
            "min": min(values),
            "max": max(values),
            "window_start": events[0].timestamp.isoformat(),
            "window_end": events[-1].timestamp.isoformat(),
        }

        logger.info(f"Агрегированные данные для {sensor_id}: {aggregated}")

        # Очистка буфера
        self.event_buffer[sensor_id] = []

    def get_stats(self) -> Dict:
        """Получение статистики по всем датчикам."""
        result = {}
        for sensor_id, stats in self.stats.items():
            result[sensor_id] = {
                "count": stats["count"],
                "avg": stats["sum"] / stats["count"] if stats["count"] > 0 else 0,
                "min": stats["min"],
                "max": stats["max"],
                "last_value": stats.get("last_value"),
                "last_update": stats.get("last_update"),
            }
        return result


class AlertConsumer(BaseConsumer):
    """Consumer для обработки оповещений."""

    def __init__(self, config: dict):
        super().__init__(config, topics=["alerts"])
        self.alert_handlers: Dict[EventPriority, Callable] = {}

    def register_handler(
        self,
        priority: EventPriority,
        handler: Callable[[SensorAlert], None]
    ) -> None:
        """Регистрация обработчика для определенного приоритета."""
        self.alert_handlers[priority] = handler

    def process_message(self, key: str, value: bytes, headers: dict) -> None:
        """Обработка оповещения."""
        try:
            data = json.loads(value.decode("utf-8"))
            alert = SensorAlert(
                alert_id=data["alert_id"],
                sensor_id=data["sensor_id"],
                sensor_type=SensorType(data["sensor_type"]),
                priority=EventPriority(data["priority"]),
                message=data["message"],
                value=data["value"],
                threshold=data["threshold"],
                timestamp=datetime.fromisoformat(data["timestamp"]),
            )

            # Вызов соответствующего обработчика
            handler = self.alert_handlers.get(alert.priority)
            if handler:
                handler(alert)
            else:
                logger.info(f"Получено оповещение: {alert.message}")

        except Exception as e:
            logger.error(f"Ошибка обработки оповещения: {e}")


def main():
    """Точка входа для Consumer."""
    import logging
    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s - %(name)s - %(levelname)s - %(message)s"
    )

    config = get_config()
    consumer_config = config.to_consumer_config()
    consumer_config["group.id"] = "sensor-processors"

    def alert_handler(alert: SensorAlert):
        """Обработчик оповещений."""
        print(f"[{alert.priority.value.upper()}] {alert.message}")

    consumer = SensorConsumer(
        config=consumer_config,
        topics=["sensor-events"],
        alert_callback=alert_handler
    )

    try:
        consumer.start()
    except KeyboardInterrupt:
        pass
    finally:
        stats = consumer.get_stats()
        print("\n=== Статистика ===")
        for sensor_id, data in stats.items():
            print(f"{sensor_id}: count={data['count']}, avg={data['avg']:.2f}")


if __name__ == "__main__":
    main()
```

## Обработка ошибок и Dead Letter Queue

```python
# src/consumers/dlq_handler.py
from typing import Callable
from confluent_kafka import Producer
import json
import logging

logger = logging.getLogger(__name__)


class DeadLetterQueueHandler:
    """Обработчик Dead Letter Queue."""

    def __init__(self, producer_config: dict, dlq_topic: str = "dead-letter-queue"):
        self.producer = Producer(producer_config)
        self.dlq_topic = dlq_topic

    def send_to_dlq(
        self,
        original_topic: str,
        key: str,
        value: bytes,
        error: str,
        retry_count: int = 0
    ) -> None:
        """Отправка сообщения в DLQ."""
        dlq_message = {
            "original_topic": original_topic,
            "original_key": key,
            "original_value": value.decode("utf-8", errors="replace"),
            "error": str(error),
            "retry_count": retry_count,
        }

        self.producer.produce(
            topic=self.dlq_topic,
            key=key.encode("utf-8") if key else None,
            value=json.dumps(dlq_message).encode("utf-8"),
        )
        self.producer.flush()

        logger.warning(f"Сообщение отправлено в DLQ: {error}")
```

## Best Practices

### Проектирование событий

1. **Неизменяемость**: События должны быть неизменяемыми после создания
2. **Уникальные идентификаторы**: Каждое событие должно иметь уникальный ID
3. **Метки времени**: Всегда включайте timestamp создания события
4. **Версионирование**: Используйте версии схемы для обратной совместимости

### Обработка событий

1. **Идемпотентность**: Consumer должен корректно обрабатывать дублирующие события
2. **Порядок обработки**: Используйте ключи партиций для сохранения порядка
3. **Graceful Shutdown**: Реализуйте корректное завершение работы
4. **Мониторинг**: Отслеживайте lag и производительность обработки

### Масштабирование

1. **Партиционирование**: Используйте ключи для равномерного распределения
2. **Consumer Groups**: Масштабируйте обработку добавлением consumer-ов
3. **Буферизация**: Используйте буферы для пакетной обработки

### Обработка ошибок

1. **Retry Logic**: Реализуйте повторные попытки с экспоненциальной задержкой
2. **Dead Letter Queue**: Отправляйте необработанные сообщения в DLQ
3. **Circuit Breaker**: Используйте circuit breaker для внешних зависимостей

## Дополнительные ресурсы

- [Apache Kafka Best Practices](https://kafka.apache.org/documentation/#design)
- [Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html)
- [Designing Events](https://developer.confluent.io/patterns/event/event-envelope/)
