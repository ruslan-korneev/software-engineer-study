# Kafka в реальном мире

## Описание

Apache Kafka используется крупнейшими технологическими компаниями мира для обработки триллионов сообщений ежедневно. В этом разделе мы рассмотрим реальные примеры использования Kafka, архитектурные решения и уроки, извлечённые из production-систем. Эти кейсы помогут понять, как применять Kafka в собственных проектах.

## Ключевые концепции

### Типичные масштабы использования

| Компания | Масштаб | Использование |
|----------|---------|---------------|
| LinkedIn | 7+ триллионов сообщений/день | Activity tracking, metrics |
| Netflix | 700+ миллиардов событий/день | Real-time analytics |
| Uber | 1+ триллион сообщений/день | Event sourcing, analytics |
| Spotify | 6+ миллиардов событий/день | User activity, recommendations |
| Twitter | 500+ миллиардов событий/день | Timeline, analytics |

## Примеры

### Кейс 1: LinkedIn — Родина Kafka

LinkedIn создал Kafka для решения своих внутренних проблем и продолжает быть крупнейшим пользователем.

**Проблема:**
```
До Kafka:
┌────────────┐     ┌────────────┐     ┌────────────┐
│ Web Server │────▶│  Database  │────▶│ Analytics  │
└────────────┘     └────────────┘     └────────────┘
      │                  │
      ▼                  ▼
┌────────────┐     ┌────────────┐
│   Logs     │     │   Search   │
└────────────┘     └────────────┘

Проблемы:
- N×M интеграций между системами
- Каждая интеграция — отдельный код
- Нет единой точки для аудита
- Сложно добавлять новые системы
```

**Решение с Kafka:**
```
После Kafka:
┌────────────┐     ┌────────────┐     ┌────────────┐
│ Web Server │     │   Mobile   │     │   API      │
└─────┬──────┘     └─────┬──────┘     └─────┬──────┘
      │                  │                  │
      └──────────────────┼──────────────────┘
                         ▼
              ┌─────────────────────┐
              │    Kafka Cluster    │
              │  (Central Hub)      │
              └──────────┬──────────┘
                         │
      ┌──────────────────┼──────────────────┐
      ▼                  ▼                  ▼
┌────────────┐     ┌────────────┐     ┌────────────┐
│  Database  │     │  Search    │     │ Analytics  │
└────────────┘     └────────────┘     └────────────┘
```

**Архитектура LinkedIn:**
```python
# Пример: отслеживание просмотров профиля
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['kafka-cluster:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

def track_profile_view(viewer_id, profile_id, timestamp):
    event = {
        "eventType": "PROFILE_VIEW",
        "viewerId": viewer_id,
        "profileId": profile_id,
        "timestamp": timestamp,
        "source": "web",
        "metadata": {
            "sessionId": "sess_123",
            "referrer": "search"
        }
    }

    # Ключ = profile_id для группировки событий профиля
    producer.send(
        'member-activity',
        key=profile_id.encode(),
        value=event
    )
```

**Результаты:**
- 7+ триллионов сообщений в день
- 100+ различных Kafka-кластеров
- Единая платформа для всех данных
- Время добавления нового consumer: минуты вместо недель

---

### Кейс 2: Netflix — Real-time Персонализация

Netflix использует Kafka для персонализации контента в реальном времени.

**Архитектура:**
```
┌─────────────────────────────────────────────────────────────────┐
│                     Netflix Data Pipeline                        │
│                                                                  │
│  ┌──────────┐                                                   │
│  │ Playback │──┐                                                │
│  │ Events   │  │                                                │
│  └──────────┘  │     ┌─────────────┐     ┌─────────────────┐   │
│                ├────▶│   Kafka     │────▶│ Flink/Spark     │   │
│  ┌──────────┐  │     │  (Keystone) │     │ (Real-time ML)  │   │
│  │   UI     │──┤     └─────────────┘     └────────┬────────┘   │
│  │ Clicks   │  │            │                     │             │
│  └──────────┘  │            ▼                     ▼             │
│                │     ┌─────────────┐     ┌─────────────────┐   │
│  ┌──────────┐  │     │    S3       │     │  Recommendation │   │
│  │ Search   │──┘     │  (Archive)  │     │     Service     │   │
│  │ Queries  │        └─────────────┘     └─────────────────┘   │
│  └──────────┘                                                   │
└─────────────────────────────────────────────────────────────────┘
```

**Пример: Обновление рекомендаций в реальном времени**
```java
// Kafka Streams приложение для real-time рекомендаций
public class ViewingHistoryProcessor {

    public static void main(String[] args) {
        StreamsBuilder builder = new StreamsBuilder();

        // Поток просмотров
        KStream<String, ViewEvent> views = builder.stream("viewing-events");

        // Агрегация просмотров по пользователю за последний час
        KTable<Windowed<String>, ViewingStats> recentViews = views
            .groupBy((key, event) -> event.getUserId())
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofHours(1)))
            .aggregate(
                ViewingStats::new,
                (userId, event, stats) -> stats.addView(event),
                Materialized.with(Serdes.String(), new ViewingStatsSerde())
            );

        // Обновление профиля пользователя для рекомендаций
        recentViews.toStream()
            .map((windowedKey, stats) -> {
                String userId = windowedKey.key();
                UserProfile profile = updateUserProfile(userId, stats);
                return new KeyValue<>(userId, profile);
            })
            .to("user-profiles-updated");

        KafkaStreams streams = new KafkaStreams(builder.build(), config);
        streams.start();
    }
}
```

**Ключевые метрики Netflix:**
- Latency: < 10ms от события до обновления рекомендаций
- Throughput: 700+ миллиардов событий в день
- Кластеры: Keystone (основной) + специализированные

---

### Кейс 3: Uber — Event Sourcing для поездок

Uber использует Kafka для отслеживания всех событий поездки.

**Архитектура поездки:**
```
┌─────────────────────────────────────────────────────────────────┐
│                    Uber Trip Event Flow                          │
│                                                                  │
│  Driver App        Rider App        Backend Services             │
│     │                 │                   │                      │
│     ▼                 ▼                   ▼                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    Kafka                                 │    │
│  │                                                          │    │
│  │  trip.requested → trip.matched → trip.started →         │    │
│  │  trip.location.updated → trip.completed → trip.rated    │    │
│  │                                                          │    │
│  └───────────────────────────┬─────────────────────────────┘    │
│                              │                                   │
│         ┌────────────────────┼────────────────────┐             │
│         ▼                    ▼                    ▼             │
│  ┌────────────┐      ┌────────────┐      ┌────────────┐        │
│  │  Pricing   │      │    ETA     │      │  Fraud     │        │
│  │  Service   │      │  Service   │      │ Detection  │        │
│  └────────────┘      └────────────┘      └────────────┘        │
└─────────────────────────────────────────────────────────────────┘
```

**Пример: Event Sourcing для поездки**
```python
from dataclasses import dataclass
from typing import List, Optional
from kafka import KafkaProducer, KafkaConsumer
import json
from datetime import datetime

@dataclass
class TripEvent:
    trip_id: str
    event_type: str
    timestamp: str
    data: dict

# Producer: Создание событий поездки
class TripEventPublisher:
    def __init__(self):
        self.producer = KafkaProducer(
            bootstrap_servers=['kafka:9092'],
            value_serializer=lambda v: json.dumps(v.__dict__, default=str).encode()
        )

    def trip_requested(self, trip_id: str, rider_id: str,
                       pickup: dict, dropoff: dict):
        event = TripEvent(
            trip_id=trip_id,
            event_type="TRIP_REQUESTED",
            timestamp=datetime.utcnow().isoformat(),
            data={
                "riderId": rider_id,
                "pickup": pickup,
                "dropoff": dropoff
            }
        )
        self.producer.send('trip-events', key=trip_id.encode(), value=event)

    def trip_matched(self, trip_id: str, driver_id: str, eta_seconds: int):
        event = TripEvent(
            trip_id=trip_id,
            event_type="TRIP_MATCHED",
            timestamp=datetime.utcnow().isoformat(),
            data={
                "driverId": driver_id,
                "etaSeconds": eta_seconds
            }
        )
        self.producer.send('trip-events', key=trip_id.encode(), value=event)

    def location_updated(self, trip_id: str, lat: float, lng: float):
        event = TripEvent(
            trip_id=trip_id,
            event_type="LOCATION_UPDATED",
            timestamp=datetime.utcnow().isoformat(),
            data={"lat": lat, "lng": lng}
        )
        self.producer.send('trip-events', key=trip_id.encode(), value=event)

# Consumer: Восстановление состояния поездки
class TripStateBuilder:
    def rebuild_trip_state(self, trip_id: str) -> dict:
        consumer = KafkaConsumer(
            'trip-events',
            bootstrap_servers=['kafka:9092'],
            auto_offset_reset='earliest',
            value_deserializer=lambda v: json.loads(v.decode())
        )

        state = {
            "tripId": trip_id,
            "status": "UNKNOWN",
            "riderId": None,
            "driverId": None,
            "locations": []
        }

        for message in consumer:
            event = message.value
            if event['trip_id'] != trip_id:
                continue

            if event['event_type'] == 'TRIP_REQUESTED':
                state['status'] = 'REQUESTED'
                state['riderId'] = event['data']['riderId']
            elif event['event_type'] == 'TRIP_MATCHED':
                state['status'] = 'MATCHED'
                state['driverId'] = event['data']['driverId']
            elif event['event_type'] == 'LOCATION_UPDATED':
                state['locations'].append(event['data'])
            elif event['event_type'] == 'TRIP_COMPLETED':
                state['status'] = 'COMPLETED'
                break

        return state
```

**Преимущества Event Sourcing в Uber:**
- Полная история каждой поездки
- Возможность replay для отладки проблем
- Разные сервисы читают одни события
- Аудит и compliance

---

### Кейс 4: Spotify — Персонализация музыки

Spotify использует Kafka для real-time обработки пользовательской активности.

**Архитектура:**
```
┌─────────────────────────────────────────────────────────────────┐
│                 Spotify Event Pipeline                           │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                     User Actions                          │   │
│  │  Play, Skip, Save, Like, Search, Playlist Add            │   │
│  └───────────────────────────┬──────────────────────────────┘   │
│                              ▼                                   │
│                    ┌─────────────────┐                          │
│                    │     Kafka       │                          │
│                    │  (Event Hub)    │                          │
│                    └────────┬────────┘                          │
│                             │                                    │
│     ┌───────────────────────┼───────────────────────┐           │
│     ▼                       ▼                       ▼           │
│  ┌──────────┐        ┌──────────────┐        ┌──────────┐      │
│  │  Taste   │        │   Session    │        │ Discover │      │
│  │ Profile  │        │   Analysis   │        │  Weekly  │      │
│  │ Builder  │        │              │        │ Pipeline │      │
│  └──────────┘        └──────────────┘        └──────────┘      │
│       │                     │                      │            │
│       └─────────────────────┼──────────────────────┘            │
│                             ▼                                    │
│                    ┌─────────────────┐                          │
│                    │ Recommendation  │                          │
│                    │     Engine      │                          │
│                    └─────────────────┘                          │
└─────────────────────────────────────────────────────────────────┘
```

**Пример: Обновление taste profile**
```python
from kafka import KafkaConsumer
from collections import defaultdict
import json

class TasteProfileUpdater:
    """
    Обновляет музыкальный профиль пользователя на основе его действий
    """

    def __init__(self):
        self.consumer = KafkaConsumer(
            'user-listening-events',
            bootstrap_servers=['kafka:9092'],
            group_id='taste-profile-service',
            value_deserializer=lambda v: json.loads(v.decode())
        )

        # В реальности - Redis или Cassandra
        self.user_profiles = defaultdict(lambda: {
            'genres': defaultdict(float),
            'artists': defaultdict(float),
            'audio_features': {
                'energy': 0.5,
                'danceability': 0.5,
                'valence': 0.5
            }
        })

    def process_events(self):
        for message in self.consumer:
            event = message.value
            user_id = event['userId']

            if event['eventType'] == 'TRACK_PLAYED':
                self._update_from_play(user_id, event)
            elif event['eventType'] == 'TRACK_SKIPPED':
                self._update_from_skip(user_id, event)
            elif event['eventType'] == 'TRACK_SAVED':
                self._update_from_save(user_id, event)

    def _update_from_play(self, user_id: str, event: dict):
        profile = self.user_profiles[user_id]
        track = event['track']

        # Увеличиваем вес жанров прослушанного трека
        for genre in track.get('genres', []):
            profile['genres'][genre] += 1.0

        # Увеличиваем вес артиста
        profile['artists'][track['artistId']] += 1.0

        # Обновляем audio features (скользящее среднее)
        alpha = 0.1  # Коэффициент обучения
        for feature in ['energy', 'danceability', 'valence']:
            if feature in track.get('audioFeatures', {}):
                profile['audio_features'][feature] = (
                    (1 - alpha) * profile['audio_features'][feature] +
                    alpha * track['audioFeatures'][feature]
                )

        # Публикуем обновлённый профиль
        self._publish_profile_update(user_id, profile)

    def _update_from_skip(self, user_id: str, event: dict):
        profile = self.user_profiles[user_id]
        track = event['track']

        # Уменьшаем вес жанров пропущенного трека
        for genre in track.get('genres', []):
            profile['genres'][genre] -= 0.5

    def _update_from_save(self, user_id: str, event: dict):
        profile = self.user_profiles[user_id]
        track = event['track']

        # Сохранение = сильный сигнал
        for genre in track.get('genres', []):
            profile['genres'][genre] += 5.0

        profile['artists'][track['artistId']] += 5.0
```

---

### Кейс 5: Банковская система — Fraud Detection

Использование Kafka для обнаружения мошенничества в реальном времени.

**Архитектура:**
```
┌─────────────────────────────────────────────────────────────────┐
│              Real-time Fraud Detection System                    │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    Transaction Sources                      │ │
│  │   ATM   │   POS   │   Online   │   Mobile   │   Wire      │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                   │
│                              ▼                                   │
│                    ┌─────────────────┐                          │
│                    │     Kafka       │                          │
│                    │  transactions   │                          │
│                    └────────┬────────┘                          │
│                             │                                    │
│  ┌──────────────────────────┼──────────────────────────────┐   │
│  │                    Fraud Detection                       │   │
│  │                                                          │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │   │
│  │  │   Rules     │  │     ML      │  │   Behavioral    │  │   │
│  │  │   Engine    │  │   Models    │  │    Analysis     │  │   │
│  │  └──────┬──────┘  └──────┬──────┘  └────────┬────────┘  │   │
│  │         │                │                   │           │   │
│  │         └────────────────┼───────────────────┘           │   │
│  │                          ▼                               │   │
│  │                  ┌─────────────┐                         │   │
│  │                  │   Decision  │                         │   │
│  │                  │   Combiner  │                         │   │
│  │                  └──────┬──────┘                         │   │
│  └─────────────────────────┼───────────────────────────────┘   │
│                            ▼                                    │
│           ┌────────────────┴────────────────┐                  │
│           ▼                                 ▼                  │
│    ┌────────────┐                    ┌────────────┐           │
│    │   ALLOW    │                    │   BLOCK    │           │
│    │            │                    │  + Alert   │           │
│    └────────────┘                    └────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

**Пример: Real-time Fraud Detection**
```python
from kafka import KafkaConsumer, KafkaProducer
from dataclasses import dataclass
from typing import Optional
import json
from datetime import datetime, timedelta

@dataclass
class Transaction:
    transaction_id: str
    card_id: str
    amount: float
    currency: str
    merchant_id: str
    merchant_category: str
    location: dict  # lat, lng, country
    timestamp: str

@dataclass
class FraudDecision:
    transaction_id: str
    decision: str  # ALLOW, BLOCK, REVIEW
    risk_score: float
    reasons: list

class FraudDetector:
    def __init__(self):
        self.consumer = KafkaConsumer(
            'transactions',
            bootstrap_servers=['kafka:9092'],
            group_id='fraud-detector',
            value_deserializer=lambda v: json.loads(v.decode())
        )

        self.producer = KafkaProducer(
            bootstrap_servers=['kafka:9092'],
            value_serializer=lambda v: json.dumps(v.__dict__).encode()
        )

        # In-memory state (в реальности - Redis/Flink state)
        self.recent_transactions = {}  # card_id -> list of transactions
        self.card_locations = {}  # card_id -> last location

    def process(self):
        for message in self.consumer:
            txn = Transaction(**message.value)
            decision = self._analyze_transaction(txn)

            # Публикуем решение
            self.producer.send(
                'fraud-decisions',
                key=txn.transaction_id.encode(),
                value=decision
            )

            # Обновляем состояние
            self._update_state(txn)

    def _analyze_transaction(self, txn: Transaction) -> FraudDecision:
        risk_score = 0.0
        reasons = []

        # Правило 1: Проверка на velocity (много транзакций за короткое время)
        velocity_risk = self._check_velocity(txn)
        if velocity_risk > 0:
            risk_score += velocity_risk
            reasons.append(f"High velocity: {velocity_risk:.2f}")

        # Правило 2: Проверка на impossible travel
        travel_risk = self._check_impossible_travel(txn)
        if travel_risk > 0:
            risk_score += travel_risk
            reasons.append(f"Impossible travel detected: {travel_risk:.2f}")

        # Правило 3: Необычная сумма
        amount_risk = self._check_unusual_amount(txn)
        if amount_risk > 0:
            risk_score += amount_risk
            reasons.append(f"Unusual amount: {amount_risk:.2f}")

        # Правило 4: Подозрительная категория
        category_risk = self._check_high_risk_category(txn)
        if category_risk > 0:
            risk_score += category_risk
            reasons.append(f"High risk category: {category_risk:.2f}")

        # Принятие решения
        if risk_score >= 0.8:
            decision = "BLOCK"
        elif risk_score >= 0.5:
            decision = "REVIEW"
        else:
            decision = "ALLOW"

        return FraudDecision(
            transaction_id=txn.transaction_id,
            decision=decision,
            risk_score=risk_score,
            reasons=reasons
        )

    def _check_velocity(self, txn: Transaction) -> float:
        recent = self.recent_transactions.get(txn.card_id, [])

        # Транзакции за последние 5 минут
        cutoff = datetime.utcnow() - timedelta(minutes=5)
        recent_count = sum(
            1 for t in recent
            if datetime.fromisoformat(t['timestamp']) > cutoff
        )

        if recent_count > 5:
            return 0.8
        elif recent_count > 3:
            return 0.4
        return 0.0

    def _check_impossible_travel(self, txn: Transaction) -> float:
        last_location = self.card_locations.get(txn.card_id)
        if not last_location:
            return 0.0

        # Расстояние между локациями
        distance_km = self._haversine_distance(
            last_location['lat'], last_location['lng'],
            txn.location['lat'], txn.location['lng']
        )

        # Время между транзакциями
        time_diff_hours = (
            datetime.fromisoformat(txn.timestamp) -
            datetime.fromisoformat(last_location['timestamp'])
        ).total_seconds() / 3600

        # Максимально возможная скорость перемещения (km/h)
        max_speed = 900  # Самолёт

        if time_diff_hours > 0:
            required_speed = distance_km / time_diff_hours
            if required_speed > max_speed:
                return 1.0  # Impossible travel

        return 0.0

    def _check_unusual_amount(self, txn: Transaction) -> float:
        if txn.amount > 10000:
            return 0.6
        elif txn.amount > 5000:
            return 0.3
        return 0.0

    def _check_high_risk_category(self, txn: Transaction) -> float:
        high_risk_categories = [
            'gambling',
            'crypto_exchange',
            'wire_transfer',
            'adult_entertainment'
        ]
        if txn.merchant_category in high_risk_categories:
            return 0.4
        return 0.0
```

---

### Кейс 6: E-commerce — Order Processing Pipeline

**Архитектура обработки заказов:**
```
┌─────────────────────────────────────────────────────────────────┐
│                 E-commerce Order Pipeline                        │
│                                                                  │
│  ┌──────────┐                                                   │
│  │ Checkout │                                                   │
│  │ Service  │                                                   │
│  └────┬─────┘                                                   │
│       │                                                          │
│       ▼                                                          │
│  orders.created ──────────────────────────────────────────────▶ │
│       │                                                          │
│       ├──────────────────┬──────────────────┬─────────────────┐ │
│       ▼                  ▼                  ▼                 ▼ │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐    ┌───────┐ │
│  │ Payment  │      │Inventory │      │ Shipping │    │ Email │ │
│  │ Service  │      │ Service  │      │ Service  │    │Service│ │
│  └────┬─────┘      └────┬─────┘      └────┬─────┘    └───┬───┘ │
│       │                 │                 │              │      │
│       ▼                 ▼                 ▼              │      │
│  orders.paid      inventory.reserved  shipping.created  │      │
│       │                 │                 │              │      │
│       └─────────────────┴─────────────────┴──────────────┘      │
│                                │                                 │
│                                ▼                                 │
│                    ┌─────────────────────┐                      │
│                    │   Order Saga        │                      │
│                    │   Orchestrator      │                      │
│                    └─────────────────────┘                      │
└─────────────────────────────────────────────────────────────────┘
```

**Пример: Saga Pattern для обработки заказа**
```python
from kafka import KafkaConsumer, KafkaProducer
from enum import Enum
import json

class OrderState(Enum):
    CREATED = "CREATED"
    PAYMENT_PENDING = "PAYMENT_PENDING"
    PAYMENT_COMPLETED = "PAYMENT_COMPLETED"
    INVENTORY_RESERVED = "INVENTORY_RESERVED"
    SHIPPING_CREATED = "SHIPPING_CREATED"
    COMPLETED = "COMPLETED"
    FAILED = "FAILED"

class OrderSagaOrchestrator:
    def __init__(self):
        self.consumer = KafkaConsumer(
            'orders.created',
            'payments.completed',
            'payments.failed',
            'inventory.reserved',
            'inventory.failed',
            'shipping.created',
            bootstrap_servers=['kafka:9092'],
            group_id='order-saga',
            value_deserializer=lambda v: json.loads(v.decode())
        )

        self.producer = KafkaProducer(
            bootstrap_servers=['kafka:9092'],
            value_serializer=lambda v: json.dumps(v).encode()
        )

        self.order_states = {}  # order_id -> state

    def run(self):
        for message in self.consumer:
            topic = message.topic
            event = message.value
            order_id = event['orderId']

            if topic == 'orders.created':
                self._handle_order_created(order_id, event)
            elif topic == 'payments.completed':
                self._handle_payment_completed(order_id, event)
            elif topic == 'payments.failed':
                self._handle_payment_failed(order_id, event)
            elif topic == 'inventory.reserved':
                self._handle_inventory_reserved(order_id, event)
            elif topic == 'inventory.failed':
                self._handle_inventory_failed(order_id, event)
            elif topic == 'shipping.created':
                self._handle_shipping_created(order_id, event)

    def _handle_order_created(self, order_id: str, event: dict):
        # Начинаем saga: отправляем команду на оплату
        self.order_states[order_id] = OrderState.PAYMENT_PENDING

        self.producer.send('payment.commands', value={
            'command': 'PROCESS_PAYMENT',
            'orderId': order_id,
            'amount': event['totalAmount'],
            'paymentMethod': event['paymentMethod']
        })

    def _handle_payment_completed(self, order_id: str, event: dict):
        self.order_states[order_id] = OrderState.PAYMENT_COMPLETED

        # Следующий шаг: резервирование товара
        self.producer.send('inventory.commands', value={
            'command': 'RESERVE_INVENTORY',
            'orderId': order_id,
            'items': event['items']
        })

    def _handle_payment_failed(self, order_id: str, event: dict):
        self.order_states[order_id] = OrderState.FAILED

        # Компенсация: уведомить клиента
        self.producer.send('notifications.commands', value={
            'command': 'NOTIFY_PAYMENT_FAILED',
            'orderId': order_id,
            'reason': event['reason']
        })

    def _handle_inventory_reserved(self, order_id: str, event: dict):
        self.order_states[order_id] = OrderState.INVENTORY_RESERVED

        # Следующий шаг: создание доставки
        self.producer.send('shipping.commands', value={
            'command': 'CREATE_SHIPMENT',
            'orderId': order_id,
            'address': event['shippingAddress']
        })

    def _handle_inventory_failed(self, order_id: str, event: dict):
        self.order_states[order_id] = OrderState.FAILED

        # Компенсация: вернуть деньги
        self.producer.send('payment.commands', value={
            'command': 'REFUND_PAYMENT',
            'orderId': order_id,
            'reason': 'Inventory not available'
        })

    def _handle_shipping_created(self, order_id: str, event: dict):
        self.order_states[order_id] = OrderState.COMPLETED

        # Уведомить клиента о успешном заказе
        self.producer.send('notifications.commands', value={
            'command': 'NOTIFY_ORDER_COMPLETED',
            'orderId': order_id,
            'trackingNumber': event['trackingNumber']
        })
```

## Best Practices

### Уроки из реальных проектов

| Урок | Описание |
|------|----------|
| **Начинайте просто** | LinkedIn начал с одного use case, потом масштабировал |
| **Инвестируйте в мониторинг** | Netflix: 90% времени на observability |
| **Планируйте retention** | Uber хранит события поездок годами для аудита |
| **Используйте Schema Registry** | Spotify: все события имеют versioned schema |
| **Автоматизируйте операции** | Большие компании используют K8s operators |

### Чек-лист перед production

```
□ Определена схема событий (Avro/Protobuf + Schema Registry)
□ Настроен мониторинг (lag, throughput, errors)
□ Есть план disaster recovery
□ Протестирована обработка ошибок
□ Настроены алерты на критические метрики
□ Документированы топики и их назначение
□ Есть runbook для типичных проблем
```

## Краткое резюме

Реальные примеры использования Kafka в крупных компаниях показывают её универсальность: от activity tracking в LinkedIn до fraud detection в банках, от персонализации в Netflix до event sourcing в Uber. Ключевые уроки: начинайте с простых use cases, инвестируйте в мониторинг и observability, планируйте схему данных заранее, и не бойтесь использовать паттерны вроде Event Sourcing и Saga для сложных бизнес-процессов.
