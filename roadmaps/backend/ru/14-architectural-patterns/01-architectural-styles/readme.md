# Архитектурные стили (Architectural Styles)

## Обзор

Архитектурные стили — это фундаментальные паттерны организации программных систем. Каждый стиль определяет структуру, компоненты, связи и принципы взаимодействия в системе.

## Темы раздела

- [x] [Многослойная архитектура](./01-layered.md) — Layered / N-tier Architecture
- [x] [Клиент-серверная архитектура](./02-client-server.md) — Client-Server Architecture
- [x] [Монолитная архитектура](./03-monolithic.md) — Monolithic Architecture
- [x] [Сервис-ориентированная архитектура](./04-soa.md) — Service-Oriented Architecture (SOA)
- [x] [Микросервисная архитектура](./05-microservices.md) — Microservices Architecture
- [x] [Бессерверная архитектура](./06-serverless.md) — Serverless Architecture

## Сравнение архитектурных стилей

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    EVOLUTION OF ARCHITECTURAL STYLES                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1960s-1980s         1990s            2000s           2010s+                │
│  ─────────────       ──────           ─────           ──────                │
│                                                                              │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐    ┌──────────┐            │
│  │Monolithic│────▶│ Client-  │────▶│   SOA    │───▶│Microser- │            │
│  │          │     │ Server   │     │          │    │  vices   │            │
│  └──────────┘     └──────────┘     └──────────┘    └──────────┘            │
│       │                                                  │                  │
│       │                                                  ▼                  │
│       │               ┌──────────────────────────────────────┐              │
│       └──────────────▶│    Serverless / FaaS (2014+)         │              │
│                       └──────────────────────────────────────┘              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Критерии выбора

| Критерий | Monolithic | SOA | Microservices | Serverless |
|----------|------------|-----|---------------|------------|
| **Размер команды** | Small | Medium-Large | Large | Any |
| **Сложность домена** | Simple | Complex | Complex | Simple-Medium |
| **Масштабируемость** | Vertical | Horizontal | Horizontal | Auto |
| **Deployment** | Simple | Medium | Complex | Simple |
| **DevOps зрелость** | Low | Medium | High | Medium |
| **Time to market** | Fast | Slow | Medium | Fast |
| **Стоимость старта** | Low | High | Medium | Low |

## Когда использовать

### Многослойная (Layered)
- Традиционные бизнес-приложения
- Проекты с чёткой доменной логикой
- Команды с разным уровнем опыта

### Клиент-серверная (Client-Server)
- Web-приложения
- Мобильные приложения с backend
- Централизованные системы

### Монолитная (Monolithic)
- Стартапы и MVP
- Небольшие команды
- Простой домен
- Начало нового проекта (Monolith First)

### SOA
- Enterprise интеграция
- B2B взаимодействие
- Legacy modernization
- Регулируемые отрасли

### Микросервисная (Microservices)
- Большие команды (50+)
- Разные требования к масштабированию
- Continuous deployment
- Высокая resilience

### Serverless
- Event-driven processing
- Нерегулярная нагрузка
- MVP и прототипы
- Scheduled tasks

## Путь эволюции архитектуры

```
┌─────────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE EVOLUTION PATH                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Start here                                                     │
│       │                                                          │
│       ▼                                                          │
│   ┌────────────────┐                                            │
│   │   Monolith     │   Быстрая разработка, понимание домена     │
│   │   (simple)     │                                            │
│   └───────┬────────┘                                            │
│           │                                                      │
│           │   Рост команды, определение boundaries               │
│           ▼                                                      │
│   ┌────────────────┐                                            │
│   │   Modular      │   Подготовка к разделению                  │
│   │   Monolith     │                                            │
│   └───────┬────────┘                                            │
│           │                                                      │
│           │   Независимые команды, разные SLA                   │
│           ▼                                                      │
│   ┌────────────────┐                                            │
│   │ Microservices  │   Полная независимость                     │
│   └───────┬────────┘                                            │
│           │                                                      │
│           │   Event-driven, переменная нагрузка                 │
│           ▼                                                      │
│   ┌────────────────┐                                            │
│   │  Serverless    │   Zero ops, auto-scaling                   │
│   │  + Micro       │                                            │
│   └────────────────┘                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Дополнительные ресурсы

### Книги
- **"Software Architecture: The Hard Parts"** — Neal Ford, Mark Richards
- **"Fundamentals of Software Architecture"** — Mark Richards, Neal Ford
- **"Building Microservices"** — Sam Newman
- **"Clean Architecture"** — Robert C. Martin

### Онлайн-ресурсы
- [Martin Fowler's Blog](https://martinfowler.com/)
- [microservices.io](https://microservices.io/)
- [Azure Architecture Center](https://docs.microsoft.com/azure/architecture/)

---

> **Примечание**: Выбор архитектурного стиля — это trade-off. Нет "лучшего" стиля — есть стиль, подходящий для конкретных требований, команды и контекста.
