# Основные концепции CI/CD

## Введение

CI/CD (Continuous Integration / Continuous Delivery / Continuous Deployment) — это набор практик и методологий, которые позволяют командам разработчиков часто и надёжно доставлять изменения кода в production. Эти практики являются фундаментом современной разработки программного обеспечения и DevOps-культуры.

## Continuous Integration (CI) — Непрерывная интеграция

### Что это такое?

**Continuous Integration** — это практика разработки, при которой разработчики регулярно (несколько раз в день) интегрируют свой код в общий репозиторий. Каждая интеграция автоматически проверяется путём сборки проекта и запуска тестов.

### Ключевые принципы CI

1. **Единый репозиторий кода** — весь код проекта хранится в одном месте (Git)
2. **Автоматизированная сборка** — код собирается автоматически при каждом изменении
3. **Автоматизированное тестирование** — тесты запускаются автоматически
4. **Быстрая обратная связь** — разработчики узнают о проблемах сразу
5. **Частые коммиты** — небольшие изменения интегрируются часто

### Пример базового CI-процесса

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Commit    │───►│    Build    │───►│    Test     │───►│   Report    │
│   (Push)    │    │   Project   │    │    Suite    │    │   Results   │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

### Преимущества CI

- **Раннее обнаружение ошибок** — баги находятся сразу, пока контекст ещё свежий
- **Уменьшение интеграционных проблем** — мелкие изменения легче интегрировать
- **Повышение качества кода** — автоматические проверки гарантируют стандарты
- **Ускорение разработки** — меньше времени на ручную проверку и отладку

### Что включает CI-пайплайн?

```yaml
# Типичные этапы CI
stages:
  - checkout      # Получение кода из репозитория
  - install       # Установка зависимостей
  - lint          # Проверка стиля кода (ESLint, Pylint, etc.)
  - build         # Сборка проекта
  - test          # Запуск unit-тестов
  - coverage      # Проверка покрытия кода тестами
```

## Continuous Delivery (CD) — Непрерывная доставка

### Что это такое?

**Continuous Delivery** — это расширение CI, которое гарантирует, что код всегда находится в состоянии, готовом к развёртыванию в production. Развёртывание происходит по нажатию кнопки (вручную).

### Ключевые принципы Continuous Delivery

1. **Код всегда готов к релизу** — main/master ветка всегда в deployable состоянии
2. **Автоматизированный процесс доставки** — от коммита до production всё автоматизировано
3. **Ручное одобрение релиза** — финальное решение о релизе принимает человек
4. **Развёртывание как рутина** — релизы должны быть простыми и безопасными

### Пример процесса Continuous Delivery

```
┌────────┐   ┌────────┐   ┌─────────┐   ┌─────────────┐   ┌────────────┐
│   CI   │──►│ Deploy │──►│ Staging │──►│   Manual    │──►│ Production │
│ Passed │   │  Stage │   │  Tests  │   │  Approval   │   │   Deploy   │
└────────┘   └────────┘   └─────────┘   └─────────────┘   └────────────┘
```

### Артефакты и версионирование

```yaml
# Пример создания артефакта
build:
  stage: build
  script:
    - npm run build
    - docker build -t myapp:${CI_COMMIT_SHA} .
    - docker push myapp:${CI_COMMIT_SHA}
  artifacts:
    paths:
      - dist/
    expire_in: 1 week
```

## Continuous Deployment — Непрерывное развёртывание

### Что это такое?

**Continuous Deployment** — это следующий шаг после Continuous Delivery. Каждое изменение, которое проходит все этапы пайплайна, автоматически развёртывается в production без ручного вмешательства.

### Отличие от Continuous Delivery

| Аспект | Continuous Delivery | Continuous Deployment |
|--------|--------------------|-----------------------|
| Автоматизация | До staging | До production |
| Ручное одобрение | Требуется | Не требуется |
| Частота релизов | По решению команды | При каждом коммите |
| Риски | Ниже | Выше (требует зрелой команды) |

### Пример процесса Continuous Deployment

```
┌────────┐   ┌────────┐   ┌─────────┐   ┌────────────┐
│   CI   │──►│ Deploy │──►│ Staging │──►│ Production │
│ Passed │   │  Stage │   │  Tests  │   │  (Auto)    │
└────────┘   └────────┘   └─────────┘   └────────────┘
                                              │
                                              ▼
                                       ┌────────────┐
                                       │ Monitoring │
                                       │ & Rollback │
                                       └────────────┘
```

### Требования для Continuous Deployment

1. **Высокое покрытие тестами** — 80%+ покрытия автоматическими тестами
2. **Feature flags** — возможность включать/выключать функции без деплоя
3. **Мониторинг** — real-time отслеживание состояния приложения
4. **Автоматический rollback** — откат при обнаружении проблем
5. **Canary/Blue-Green deployments** — стратегии безопасного деплоя

## Сравнение CI, CD (Delivery) и CD (Deployment)

```
                    Continuous Integration
                    ├── Code Commit
                    ├── Build
                    ├── Unit Tests
                    └── Integration Tests
                              │
                              ▼
                    Continuous Delivery
                    ├── Deploy to Staging
                    ├── Acceptance Tests
                    ├── Performance Tests
                    └── Manual Approval ◄─── Ручное решение
                              │
                              ▼
                    Continuous Deployment
                    ├── Auto Deploy to Production
                    ├── Smoke Tests
                    └── Monitoring
```

## Инструменты CI/CD

### Популярные платформы

| Инструмент | Тип | Особенности |
|------------|-----|-------------|
| **GitHub Actions** | Cloud/Self-hosted | Интеграция с GitHub, YAML-конфигурация |
| **GitLab CI/CD** | Cloud/Self-hosted | Встроен в GitLab, мощный и гибкий |
| **Jenkins** | Self-hosted | Open-source, огромное количество плагинов |
| **CircleCI** | Cloud | Быстрый, хорошая интеграция с GitHub |
| **Travis CI** | Cloud | Простой, популярен в open-source |
| **Azure DevOps** | Cloud | Microsoft экосистема, enterprise-ready |

### Выбор инструмента

```
Для начинающих:
├── GitHub Actions — если код на GitHub
└── GitLab CI/CD — если код на GitLab

Для enterprise:
├── Jenkins — максимальный контроль
├── Azure DevOps — если используете Microsoft stack
└── TeamCity — если используете JetBrains экосистему
```

## Пайплайн как код (Pipeline as Code)

### Принцип

Вся конфигурация CI/CD хранится в виде кода в репозитории проекта. Это обеспечивает:

- **Версионирование** — история изменений пайплайна
- **Code Review** — изменения проходят review
- **Воспроизводимость** — одинаковый пайплайн в любой среде
- **Документация** — код сам является документацией

### Пример: GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run tests
        run: npm test

      - name: Build
        run: npm run build
```

### Пример: GitLab CI/CD

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

variables:
  NODE_VERSION: "20"

build:
  stage: build
  image: node:${NODE_VERSION}
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/

test:
  stage: test
  image: node:${NODE_VERSION}
  script:
    - npm ci
    - npm run test:coverage
  coverage: '/Lines\s*:\s*(\d+\.?\d*)%/'

deploy_staging:
  stage: deploy
  script:
    - echo "Deploying to staging..."
  environment:
    name: staging
  only:
    - develop

deploy_production:
  stage: deploy
  script:
    - echo "Deploying to production..."
  environment:
    name: production
  when: manual
  only:
    - main
```

## Стратегии развёртывания

### Blue-Green Deployment

```
                    ┌─────────────────┐
                    │  Load Balancer  │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
       ┌──────▼──────┐              ┌───────▼─────┐
       │    Blue     │              │    Green    │
       │  (Active)   │              │  (Standby)  │
       │   v1.0.0    │              │   v1.1.0    │
       └─────────────┘              └─────────────┘
```

**Процесс:**
1. Новая версия деплоится в Green окружение
2. Тестирование в Green
3. Переключение трафика с Blue на Green
4. Blue становится standby (для быстрого rollback)

### Canary Deployment

```
                    ┌─────────────────┐
                    │  Load Balancer  │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │ 90%                    10%  │
       ┌──────▼──────┐              ┌───────▼─────┐
       │   Stable    │              │   Canary    │
       │   v1.0.0    │              │   v1.1.0    │
       └─────────────┘              └─────────────┘
```

**Процесс:**
1. Новая версия получает небольшой % трафика (1-10%)
2. Мониторинг метрик и ошибок
3. Постепенное увеличение трафика
4. Полный переход или откат

### Rolling Deployment

```
Время T1:  [v1] [v1] [v1] [v1] [v1]
Время T2:  [v2] [v1] [v1] [v1] [v1]
Время T3:  [v2] [v2] [v1] [v1] [v1]
Время T4:  [v2] [v2] [v2] [v1] [v1]
Время T5:  [v2] [v2] [v2] [v2] [v2]
```

**Процесс:**
1. Инстансы обновляются по одному
2. Старые инстансы заменяются новыми
3. Минимальный downtime

## Типичные ошибки и антипаттерны

### 1. Длинные пайплайны

```yaml
# Плохо: последовательное выполнение
stages:
  - lint
  - unit-tests
  - integration-tests
  - e2e-tests
  - build
# Время: 30 минут

# Хорошо: параллельное выполнение
jobs:
  lint:
    ...
  unit-tests:
    ...
  integration-tests:
    ...
# Время: 10 минут
```

### 2. Отсутствие кэширования

```yaml
# Плохо: каждый раз скачиваем зависимости
- run: npm install

# Хорошо: используем кэш
- uses: actions/cache@v3
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('package-lock.json') }}
- run: npm ci
```

### 3. Секреты в коде

```yaml
# НИКОГДА так не делайте!
env:
  API_KEY: "sk-12345"

# Правильно: используйте secrets
env:
  API_KEY: ${{ secrets.API_KEY }}
```

## Метрики CI/CD

### Ключевые показатели

| Метрика | Описание | Цель |
|---------|----------|------|
| **Lead Time** | Время от коммита до production | < 1 день |
| **Deployment Frequency** | Частота деплоев | Несколько раз в день |
| **MTTR** | Время восстановления после сбоя | < 1 часа |
| **Change Failure Rate** | % неудачных изменений | < 15% |

## Заключение

CI/CD — это не просто инструменты, а культура и практики, которые позволяют:

1. **Быстрее доставлять ценность** пользователям
2. **Снижать риски** за счёт частых небольших изменений
3. **Повышать качество** через автоматизацию
4. **Улучшать коммуникацию** в команде

Начните с простого CI-пайплайна и постепенно добавляйте этапы по мере роста проекта и команды.

## Дополнительные ресурсы

- [The DevOps Handbook](https://itrevolution.com/the-devops-handbook/)
- [Continuous Delivery by Jez Humble](https://continuousdelivery.com/)
- [DORA Metrics](https://www.devops-research.com/research.html)
