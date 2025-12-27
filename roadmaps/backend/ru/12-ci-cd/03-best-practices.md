# Лучшие практики CI/CD

## Введение

Правильно настроенный CI/CD пайплайн — это основа надёжной и быстрой доставки программного обеспечения. В этом документе собраны проверенные практики, которые помогут избежать типичных ошибок и построить эффективный процесс разработки.

## Общие принципы

### 1. Быстрая обратная связь

Чем быстрее разработчик узнает о проблеме, тем дешевле её исправить.

```yaml
# Структура пайплайна: от быстрых проверок к медленным
jobs:
  # 1. Быстрые проверки (< 1 мин)
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run lint          # ~30 сек

  # 2. Unit-тесты (1-5 мин)
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run test:unit     # ~2 мин

  # 3. Integration-тесты (5-15 мин)
  integration-tests:
    needs: [lint, unit-tests]      # Запускаем только если первые прошли
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:integration

  # 4. E2E-тесты (15-30 мин)
  e2e-tests:
    needs: [integration-tests]
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:e2e
```

### 2. Fail Fast

Прерывайте пайплайн сразу при первой ошибке, не тратя ресурсы.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: true    # Остановить все jobs при первой ошибке
      matrix:
        node: [18, 20, 22]
    steps:
      - run: npm test

  # Критические проверки идут первыми
  security:
    runs-on: ubuntu-latest
    steps:
      - run: npm audit --audit-level=critical
        # Если есть критические уязвимости - сразу fail
```

### 3. Идемпотентность

Пайплайн должен давать одинаковый результат при каждом запуске.

```yaml
# Плохо: зависимость от внешнего состояния
steps:
  - run: npm install  # Может установить разные версии

# Хорошо: фиксированные версии
steps:
  - run: npm ci       # Строго по package-lock.json

# Плохо: использование latest тегов
steps:
  - uses: actions/checkout@latest

# Хорошо: фиксированные версии actions
steps:
  - uses: actions/checkout@v4
```

### 4. Воспроизводимость

Любой может воспроизвести билд локально.

```yaml
# Пример: использование Docker для воспроизводимости
jobs:
  build:
    runs-on: ubuntu-latest
    container:
      image: node:20-alpine
      # Тот же образ можно использовать локально:
      # docker run -v $(pwd):/app node:20-alpine npm run build
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
```

## Управление секретами

### Никогда не храните секреты в коде

```yaml
# КРИТИЧЕСКАЯ ОШИБКА!
env:
  API_KEY: "sk-1234567890"
  DATABASE_URL: "postgres://user:password@host/db"

# ПРАВИЛЬНО: используйте secrets
env:
  API_KEY: ${{ secrets.API_KEY }}
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

### Ротация секретов

```yaml
# Пример: проверка возраста секретов
steps:
  - name: Check secret rotation
    run: |
      # Проверяем дату последнего обновления секрета
      # Если больше 90 дней - предупреждение
      echo "::warning::Consider rotating secrets older than 90 days"
```

### Минимальные привилегии

```yaml
# Ограничьте права токена
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read      # Только чтение репозитория
      packages: write     # Запись в packages
      # Не давайте лишних прав!

    steps:
      - uses: actions/checkout@v4
      - run: npm publish
```

### Маскирование в логах

```yaml
steps:
  - name: Use secret safely
    run: |
      # Маскируем значение в логах
      echo "::add-mask::${{ secrets.API_KEY }}"

      # Теперь секрет не будет виден в логах
      curl -H "Authorization: Bearer ${{ secrets.API_KEY }}" \
           https://api.example.com/data
```

## Оптимизация производительности

### Параллельное выполнение

```yaml
# Плохо: последовательное выполнение
jobs:
  lint:
    runs-on: ubuntu-latest
    steps: [...]
  test:
    needs: lint        # Ждём lint
    runs-on: ubuntu-latest
    steps: [...]
  build:
    needs: test        # Ждём test
    runs-on: ubuntu-latest
    steps: [...]
# Общее время: lint + test + build

# Хорошо: параллельное выполнение независимых задач
jobs:
  lint:
    runs-on: ubuntu-latest
    steps: [...]
  test:
    runs-on: ubuntu-latest
    steps: [...]
  build:
    runs-on: ubuntu-latest
    steps: [...]
  deploy:
    needs: [lint, test, build]  # Ждём все три
    runs-on: ubuntu-latest
    steps: [...]
# Общее время: max(lint, test, build) + deploy
```

### Эффективное кэширование

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Кэш зависимостей
      - name: Cache node_modules
        uses: actions/cache@v4
        id: cache
        with:
          path: node_modules
          key: node-${{ runner.os }}-${{ hashFiles('package-lock.json') }}

      # Устанавливаем только если кэш не найден
      - name: Install dependencies
        if: steps.cache.outputs.cache-hit != 'true'
        run: npm ci

      # Кэш сборки (для incremental builds)
      - name: Cache build
        uses: actions/cache@v4
        with:
          path: |
            .next/cache
            dist/.cache
          key: build-${{ runner.os }}-${{ hashFiles('src/**') }}
          restore-keys: |
            build-${{ runner.os }}-
```

### Оптимизация Docker

```yaml
jobs:
  build-image:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build with cache
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: myapp:latest
          # Используем GitHub Actions cache
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Многоэтапные Docker-образы

```dockerfile
# Dockerfile с multi-stage build
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

## Тестирование в CI/CD

### Пирамида тестирования

```
          /\
         /  \         E2E Tests (мало, медленные)
        /    \
       /──────\
      /        \      Integration Tests
     /──────────\
    /            \    Unit Tests (много, быстрые)
   /──────────────\
```

```yaml
jobs:
  # Unit tests - запускаем всегда
  unit:
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:unit -- --coverage

  # Integration - запускаем на PR и main
  integration:
    if: github.event_name == 'pull_request' || github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:integration

  # E2E - только перед деплоем
  e2e:
    if: github.ref == 'refs/heads/main'
    needs: [unit, integration]
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:e2e
```

### Покрытие кода

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci

      - name: Run tests with coverage
        run: npm run test:coverage

      # Загрузка в Codecov
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage/lcov.info
          fail_ci_if_error: true

      # Проверка минимального покрытия
      - name: Check coverage threshold
        run: |
          COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage is below 80%: $COVERAGE%"
            exit 1
          fi
```

### Тестирование на разных платформах

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [18, 20, 22]
        exclude:
          # Пропускаем комбинацию node 18 + windows
          - os: windows-latest
            node: 18
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm test
```

## Стратегии развёртывания

### Blue-Green Deployment

```yaml
jobs:
  deploy-blue-green:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Green
        run: |
          # Деплоим новую версию в Green окружение
          kubectl apply -f k8s/green-deployment.yaml

      - name: Health check Green
        run: |
          # Проверяем здоровье Green
          for i in {1..30}; do
            if curl -s http://green.myapp.com/health | grep -q "ok"; then
              echo "Green is healthy"
              exit 0
            fi
            sleep 10
          done
          echo "Green health check failed"
          exit 1

      - name: Switch traffic to Green
        run: |
          # Переключаем трафик на Green
          kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'

      - name: Keep Blue as rollback
        run: |
          # Blue остаётся для быстрого отката
          echo "Blue is now standby"
```

### Canary Deployment

```yaml
jobs:
  deploy-canary:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy Canary (10% traffic)
        run: |
          kubectl apply -f k8s/canary-deployment.yaml
          # Настраиваем 10% трафика на canary
          kubectl apply -f k8s/canary-virtual-service.yaml

      - name: Monitor Canary (5 min)
        run: |
          # Мониторим метрики 5 минут
          sleep 300
          ERROR_RATE=$(curl -s prometheus/query?query=error_rate | jq '.data.result[0].value[1]')
          if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
            echo "Error rate too high: $ERROR_RATE"
            exit 1
          fi

      - name: Increase to 50%
        run: |
          kubectl apply -f k8s/canary-50-percent.yaml
          sleep 300
          # Повторяем проверки...

      - name: Full rollout
        run: |
          kubectl apply -f k8s/full-deployment.yaml
```

### Feature Flags

```yaml
jobs:
  deploy-with-flags:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy with feature disabled
        env:
          FEATURE_NEW_UI: "false"
        run: |
          # Деплоим с выключенной фичей
          envsubst < k8s/deployment.yaml | kubectl apply -f -

      - name: Enable for beta users
        run: |
          # Включаем для 10% пользователей через LaunchDarkly/Unleash
          curl -X PATCH https://app.launchdarkly.com/api/v2/flags/new-ui \
            -H "Authorization: ${{ secrets.LD_API_KEY }}" \
            -d '{"percentageRollout": 10}'
```

## Безопасность

### Сканирование зависимостей

```yaml
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      # Проверка npm пакетов
      - name: npm audit
        run: npm audit --audit-level=high
        continue-on-error: true

      # Snyk для глубокого анализа
      - name: Snyk Security Scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

      # Trivy для Docker образов
      - name: Scan Docker image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
```

### Статический анализ кода (SAST)

```yaml
jobs:
  sast:
    runs-on: ubuntu-latest
    steps:
      # CodeQL для GitHub
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v2
        with:
          languages: javascript, python

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v2

      # Semgrep для дополнительных проверок
      - name: Semgrep Scan
        uses: semgrep/semgrep-action@v1
        with:
          config: p/security-audit
```

### Подпись артефактов

```yaml
jobs:
  build-and-sign:
    runs-on: ubuntu-latest
    steps:
      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .

      # Подпись с помощью cosign
      - name: Sign image
        uses: sigstore/cosign-installer@main

      - name: Sign the image
        run: |
          cosign sign --key env://COSIGN_PRIVATE_KEY \
            myapp:${{ github.sha }}
        env:
          COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
```

## Мониторинг и откат

### Health Checks после деплоя

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: kubectl apply -f k8s/

      - name: Wait for rollout
        run: kubectl rollout status deployment/myapp --timeout=5m

      - name: Health check
        run: |
          for i in {1..10}; do
            STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://myapp.com/health)
            if [ "$STATUS" = "200" ]; then
              echo "Health check passed"
              exit 0
            fi
            echo "Attempt $i failed with status $STATUS"
            sleep 30
          done
          echo "Health check failed after 10 attempts"
          exit 1

      - name: Rollback on failure
        if: failure()
        run: kubectl rollout undo deployment/myapp
```

### Автоматический откат

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        id: deploy
        run: |
          kubectl apply -f k8s/
          kubectl rollout status deployment/myapp --timeout=5m

      - name: Smoke tests
        id: smoke
        run: npm run test:smoke
        continue-on-error: true

      - name: Check metrics
        id: metrics
        run: |
          # Проверяем метрики через Prometheus
          ERROR_RATE=$(curl -s "prometheus/api/v1/query?query=rate(http_errors[5m])" | jq '.data.result[0].value[1]')
          if (( $(echo "$ERROR_RATE > 0.05" | bc -l) )); then
            echo "Error rate too high: $ERROR_RATE"
            exit 1
          fi
        continue-on-error: true

      - name: Rollback if needed
        if: steps.smoke.outcome == 'failure' || steps.metrics.outcome == 'failure'
        run: |
          echo "Rolling back due to failed checks"
          kubectl rollout undo deployment/myapp

          # Уведомление в Slack
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -d '{"text": "Deployment rolled back due to failed health checks"}'
```

## Документирование и аудит

### Changelog автоматизация

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      # Генерация changelog из conventional commits
      - name: Generate Changelog
        uses: TriPSs/conventional-changelog-action@v4
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          output-file: "CHANGELOG.md"

      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          body_path: CHANGELOG.md
          tag_name: ${{ github.ref_name }}
```

### Аудит пайплайна

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Log pipeline start
        run: |
          echo "Pipeline started at $(date -u +"%Y-%m-%dT%H:%M:%SZ")"
          echo "Triggered by: ${{ github.actor }}"
          echo "Commit: ${{ github.sha }}"
          echo "Branch: ${{ github.ref_name }}"

      # ... остальные шаги ...

      - name: Log pipeline end
        if: always()
        run: |
          echo "Pipeline finished at $(date -u +"%Y-%m-%dT%H:%M:%SZ")"
          echo "Status: ${{ job.status }}"
```

## Организация workflow-файлов

### Структура для больших проектов

```
.github/
├── workflows/
│   ├── ci.yml              # Основной CI пайплайн
│   ├── cd-staging.yml      # Деплой в staging
│   ├── cd-production.yml   # Деплой в production
│   ├── release.yml         # Релизы
│   ├── security.yml        # Проверки безопасности
│   └── scheduled/
│       ├── dependency-update.yml
│       └── security-scan.yml
├── actions/
│   └── custom-action/      # Кастомные actions
│       ├── action.yml
│       └── index.js
└── CODEOWNERS              # Кто может апрувить изменения
```

### Переиспользуемые workflows

```yaml
# .github/workflows/reusable-test.yml
name: Reusable Test Workflow

on:
  workflow_call:
    inputs:
      node-version:
        type: string
        default: '20'
    secrets:
      npm-token:
        required: false

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
      - run: npm ci
      - run: npm test
```

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test-node-18:
    uses: ./.github/workflows/reusable-test.yml
    with:
      node-version: '18'

  test-node-20:
    uses: ./.github/workflows/reusable-test.yml
    with:
      node-version: '20'
```

## Антипаттерны и как их избежать

### 1. Слишком длинные пайплайны

```yaml
# Плохо: один огромный job
jobs:
  everything:
    runs-on: ubuntu-latest
    steps:
      - run: npm install      # 2 мин
      - run: npm run lint     # 1 мин
      - run: npm run build    # 3 мин
      - run: npm run test     # 5 мин
      - run: npm run e2e      # 10 мин
      - run: docker build     # 5 мин
      - run: docker push      # 3 мин
      # Общее время: 29 минут, всё последовательно!

# Хорошо: параллельные независимые jobs
jobs:
  lint: ...      # 1 мин
  test: ...      # 5 мин (параллельно с lint)
  build: ...     # 3 мин (параллельно с lint и test)
  e2e:
    needs: [build]  # После сборки
  docker:
    needs: [lint, test, build]  # После всех проверок
  # Общее время: ~15 минут
```

### 2. Отсутствие кэширования

```yaml
# Плохо: каждый раз скачиваем всё заново
- run: npm install  # 2 минуты каждый раз

# Хорошо: кэшируем зависимости
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('package-lock.json') }}
- run: npm ci  # 10 секунд с кэшем
```

### 3. Хардкод версий везде

```yaml
# Плохо: версии разбросаны по всему файлу
jobs:
  build:
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
  test:
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: '20'  # Дублирование!

# Хорошо: используем переменные
env:
  NODE_VERSION: '20'

jobs:
  build:
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
  test:
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
```

### 4. Игнорирование ошибок

```yaml
# Плохо: тихое игнорирование ошибок
- run: npm test || true

# Хорошо: обработка с уведомлением
- name: Run tests
  id: tests
  run: npm test
  continue-on-error: true

- name: Notify on test failure
  if: steps.tests.outcome == 'failure'
  run: |
    curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
      -d '{"text": "Tests failed! Check: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"}'
    exit 1  # Всё равно фейлим пайплайн
```

## Чек-лист для ревью CI/CD

### Безопасность
- [ ] Секреты не захардкожены
- [ ] Используются минимальные права (permissions)
- [ ] Есть сканирование зависимостей
- [ ] Есть SAST проверки

### Производительность
- [ ] Независимые jobs запускаются параллельно
- [ ] Кэширование настроено для зависимостей
- [ ] Быстрые проверки идут первыми

### Надёжность
- [ ] Версии actions зафиксированы
- [ ] Есть таймауты для jobs и steps
- [ ] Есть health checks после деплоя
- [ ] Настроен автоматический откат

### Maintainability
- [ ] Workflow-файлы хорошо структурированы
- [ ] Используются переменные вместо хардкода
- [ ] Есть комментарии для сложных частей
- [ ] Переиспользуемые части вынесены

## Заключение

Эффективный CI/CD — это баланс между:

1. **Скоростью** — быстрая обратная связь
2. **Надёжностью** — стабильные и предсказуемые пайплайны
3. **Безопасностью** — защита секретов и кода
4. **Maintainability** — простота поддержки

Начните с простого пайплайна и постепенно добавляйте улучшения по мере роста проекта.

## Дополнительные ресурсы

- [GitHub Actions Best Practices](https://docs.github.com/en/actions/learn-github-actions/security-hardening-for-github-actions)
- [DORA Metrics](https://www.devops-research.com/research.html)
- [The DevOps Handbook](https://itrevolution.com/the-devops-handbook/)
- [Accelerate: Building High Performing Technology Organizations](https://itrevolution.com/accelerate-book/)
