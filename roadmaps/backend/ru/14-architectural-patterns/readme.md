# Architectural Patterns

Архитектурные паттерны — это проверенные решения для организации структуры программных систем. Они определяют высокоуровневую организацию компонентов и их взаимодействие.

## Содержание

### 01. Архитектурные стили

Основные подходы к организации архитектуры приложений.

- [x] [Layered Architecture](./01-architectural-styles/01-layered.md) — Многослойная архитектура
- [x] [Client-Server](./01-architectural-styles/02-client-server.md) — Клиент-серверная архитектура
- [x] [Monolithic](./01-architectural-styles/03-monolithic.md) — Монолитные приложения
- [x] [SOA](./01-architectural-styles/04-soa.md) — Service-Oriented Architecture
- [x] [Microservices](./01-architectural-styles/05-microservices.md) — Микросервисная архитектура
- [x] [Serverless](./01-architectural-styles/06-serverless.md) — Бессерверная архитектура

### 02. Паттерны данных и состояния

Паттерны для управления данными, состоянием и транзакциями в распределённых системах.

- [x] [CQRS](./02-data-patterns/01-cqrs.md) — Command Query Responsibility Segregation
- [x] [Event Sourcing](./02-data-patterns/02-event-sourcing.md) — Хранение событий как источник истины
- [x] [Sharding](./02-data-patterns/03-sharding.md) — Горизонтальное партиционирование данных
- [x] [Saga](./02-data-patterns/04-saga.md) — Распределённые транзакции

### 03. Паттерны коммуникации

Паттерны организации взаимодействия между компонентами системы.

- [x] [MVC](./03-communication-patterns/01-mvc.md) — Model-View-Controller
- [x] [Pub-Sub](./03-communication-patterns/02-pub-sub.md) — Publish-Subscribe
- [x] [Controller-Responder](./03-communication-patterns/03-controller-responder.md) — Master-Slave

### 04. Паттерны устойчивости

Паттерны для обеспечения отказоустойчивости и стабильности системы.

- [x] [Circuit Breaker](./04-resilience-patterns/01-circuit-breaker.md) — Предохранитель
- [x] [Throttling](./04-resilience-patterns/02-throttling.md) — Rate Limiting
- [x] [Service Mesh](./04-resilience-patterns/03-service-mesh.md) — Сервисная сетка

### 05. Паттерны развёртывания и миграции

Паттерны для развёртывания, миграции и эксплуатации приложений.

- [x] [Strangler](./05-deployment-patterns/01-strangler.md) — Постепенная миграция
- [x] [Static Content Hosting](./05-deployment-patterns/02-static-content-hosting.md) — Хостинг статики
- [x] [Twelve-Factor App](./05-deployment-patterns/03-twelve-factor-apps.md) — 12 факторов

---

## Как изучать

1. **Начните с архитектурных стилей** — это фундамент для понимания остальных паттернов
2. **Изучите паттерны данных** — критически важны для распределённых систем
3. **Освойте паттерны коммуникации** — определяют взаимодействие компонентов
4. **Добавьте паттерны устойчивости** — обеспечивают надёжность production-систем
5. **Завершите паттернами развёртывания** — подготовят к DevOps практикам

## Источники

- [Red Hat: 14 Software Architecture Patterns](https://www.redhat.com/en/blog/14-software-architecture-patterns)
- [Microsoft: Cloud Design Patterns](https://docs.microsoft.com/en-us/azure/architecture/patterns/)
- [Martin Fowler's Patterns](https://martinfowler.com/articles/patterns.html)
