# CI/CD Security (Безопасность конвейера развёртывания)

## Что такое CI/CD Security?

**CI/CD Security** — это набор практик защиты конвейера непрерывной интеграции и развёртывания. Компрометация CI/CD может привести к:

- Внедрению вредоносного кода в production
- Утечке секретов и credentials
- Несанкционированному доступу к инфраструктуре
- Supply chain атакам

---

## Угрозы CI/CD

| Угроза | Описание | Последствия |
|--------|----------|-------------|
| Компрометация репозитория | Злоумышленник получает доступ к коду | Внедрение backdoor |
| Утечка секретов | Credentials попадают в логи/код | Доступ к production |
| Dependency confusion | Подмена зависимостей | Выполнение вредоносного кода |
| Insecure pipeline | Отсутствие проверок | Деплой уязвимого кода |
| Privilege escalation | Избыточные права CI | Атака на инфраструктуру |

---

## 1. Безопасное хранение секретов

### Никогда не храните секреты в коде!

```python
# ПЛОХО: Секреты в коде
DATABASE_URL = "postgresql://user:password123@localhost/db"
API_KEY = "sk_live_abc123"

# ХОРОШО: Секреты из окружения
import os
DATABASE_URL = os.environ["DATABASE_URL"]
API_KEY = os.environ["API_KEY"]
```

### GitHub Secrets

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          API_KEY: ${{ secrets.API_KEY }}
        run: |
          # Секреты доступны как переменные окружения
          ./deploy.sh

      # ВАЖНО: Не выводите секреты в логи!
      - name: Debug (ПЛОХО!)
        run: echo ${{ secrets.API_KEY }}  # ❌ Секрет попадёт в логи!
```

### Vault для управления секретами

```python
import hvac

class SecretManager:
    def __init__(self):
        self.client = hvac.Client(
            url=os.environ["VAULT_ADDR"],
            token=os.environ["VAULT_TOKEN"]
        )

    def get_secret(self, path: str) -> dict:
        """Получение секрета из Vault"""
        secret = self.client.secrets.kv.v2.read_secret_version(
            path=path,
            mount_point="secret"
        )
        return secret["data"]["data"]

# Использование
secrets = SecretManager()
db_config = secrets.get_secret("database/production")
# {"host": "...", "password": "..."}
```

### Ротация секретов

```yaml
# GitHub Action для ротации секретов
name: Rotate Secrets

on:
  schedule:
    - cron: '0 0 1 * *'  # Каждый месяц

jobs:
  rotate:
    runs-on: ubuntu-latest
    steps:
      - name: Generate new API key
        run: |
          NEW_KEY=$(openssl rand -hex 32)
          # Обновляем в Vault
          vault kv put secret/api-key value=$NEW_KEY

      - name: Update application
        run: |
          # Перезапускаем приложение с новым ключом
          kubectl rollout restart deployment/api
```

---

## 2. Сканирование зависимостей

### Dependabot (GitHub)

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "daily"
    open-pull-requests-limit: 10

    # Группировка обновлений
    groups:
      security:
        applies-to: security-updates

    # Игнорирование major версий для стабильности
    ignore:
      - dependency-name: "django"
        update-types: ["version-update:semver-major"]
```

### Safety (Python)

```yaml
# .github/workflows/security.yml
name: Security Scan

on: [push, pull_request]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install safety

      - name: Check for vulnerabilities
        run: safety check -r requirements.txt --full-report
```

### Snyk

```yaml
- name: Run Snyk
  uses: snyk/actions/python@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    args: --severity-threshold=high
```

### Trivy (контейнеры)

```yaml
- name: Scan Docker image
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'myapp:${{ github.sha }}'
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'

- name: Upload Trivy results
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: 'trivy-results.sarif'
```

---

## 3. Статический анализ кода (SAST)

### Bandit (Python)

```yaml
- name: Run Bandit
  run: |
    pip install bandit
    bandit -r src/ -f json -o bandit-report.json || true

- name: Check Bandit results
  run: |
    if grep -q '"severity": "HIGH"' bandit-report.json; then
      echo "High severity issues found!"
      cat bandit-report.json
      exit 1
    fi
```

### Semgrep

```yaml
- name: Semgrep Scan
  uses: returntocorp/semgrep-action@v1
  with:
    config: >-
      p/python
      p/security-audit
      p/owasp-top-ten
```

### SonarQube

```yaml
- name: SonarQube Scan
  uses: SonarSource/sonarqube-scan-action@master
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}

- name: Quality Gate
  uses: SonarSource/sonarqube-quality-gate-action@master
  timeout-minutes: 5
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

---

## 4. Проверка Docker образов

### Безопасный Dockerfile

```dockerfile
# Используем официальный образ с конкретной версией
FROM python:3.11-slim-bookworm

# Не запускаем от root
RUN useradd --create-home --shell /bin/bash app
USER app
WORKDIR /home/app

# Копируем только необходимое
COPY --chown=app:app requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

COPY --chown=app:app src/ ./src/

# Не храним секреты в образе
# ENV API_KEY=secret  # ❌ НИКОГДА!

# Используем non-root порт
EXPOSE 8000

CMD ["python", "-m", "uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Hadolint (линтер Dockerfile)

```yaml
- name: Lint Dockerfile
  uses: hadolint/hadolint-action@v3.1.0
  with:
    dockerfile: Dockerfile
    failure-threshold: warning
```

### Dockle (аудит образов)

```yaml
- name: Build image
  run: docker build -t myapp:test .

- name: Run Dockle
  uses: erzz/dockle-action@v1
  with:
    image: myapp:test
    failure-threshold: high
```

---

## 5. Защита пайплайна

### Branch Protection Rules

```yaml
# Через GitHub API или UI
# Settings → Branches → Add rule

# Требования для main:
# - Require pull request before merging
# - Require approvals: 2
# - Require status checks: tests, security-scan
# - Require signed commits
# - Do not allow force pushes
# - Do not allow deletions
```

### Ограничение прав GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

# Минимальные права по умолчанию
permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pytest

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest

    # Расширенные права только для деплоя
    permissions:
      contents: read
      id-token: write  # Для OIDC аутентификации

    steps:
      - uses: actions/checkout@v4
      - name: Deploy
        run: ./deploy.sh
```

### OIDC вместо долгоживущих токенов

```yaml
# Аутентификация в AWS без статических credentials
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/github-actions
    aws-region: eu-west-1
    # Нет access keys!

- name: Deploy to ECS
  run: |
    aws ecs update-service --cluster prod --service api --force-new-deployment
```

---

## 6. Подпись артефактов

### Подпись Docker образов (cosign)

```yaml
- name: Install cosign
  uses: sigstore/cosign-installer@v3

- name: Build and push
  run: |
    docker build -t ghcr.io/myorg/myapp:${{ github.sha }} .
    docker push ghcr.io/myorg/myapp:${{ github.sha }}

- name: Sign image
  env:
    COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
  run: |
    cosign sign --key env://COSIGN_PRIVATE_KEY ghcr.io/myorg/myapp:${{ github.sha }}

# Проверка при деплое
- name: Verify signature
  run: |
    cosign verify --key cosign.pub ghcr.io/myorg/myapp:${{ github.sha }}
```

### SBOM (Software Bill of Materials)

```yaml
- name: Generate SBOM
  uses: anchore/sbom-action@v0
  with:
    image: myapp:${{ github.sha }}
    format: spdx-json
    output-file: sbom.spdx.json

- name: Upload SBOM
  uses: actions/upload-artifact@v4
  with:
    name: sbom
    path: sbom.spdx.json
```

---

## 7. Безопасность окружений

### Environment protection rules

```yaml
# .github/workflows/deploy.yml
jobs:
  deploy-staging:
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh staging

  deploy-production:
    needs: deploy-staging
    environment:
      name: production
      url: https://api.example.com

    # Требует ручного одобрения
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh production
```

### Изоляция environments

```yaml
# Разные секреты для разных окружений
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }}
    env:
      # Секреты автоматически берутся из соответствующего environment
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

---

## 8. Мониторинг и аудит

### Аудит GitHub Actions

```yaml
# Webhook для отправки событий в SIEM
- name: Audit log
  if: always()
  run: |
    curl -X POST ${{ secrets.AUDIT_WEBHOOK }} \
      -H "Content-Type: application/json" \
      -d '{
        "workflow": "${{ github.workflow }}",
        "run_id": "${{ github.run_id }}",
        "actor": "${{ github.actor }}",
        "event": "${{ github.event_name }}",
        "ref": "${{ github.ref }}",
        "status": "${{ job.status }}",
        "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"
      }'
```

### Alerts на подозрительную активность

```yaml
- name: Check for suspicious patterns
  run: |
    # Проверка на добавление новых секретов
    git diff HEAD~1 --name-only | grep -E '\.(env|key|pem)$' && \
      echo "::warning::Suspicious file added"

    # Проверка на изменение CI конфигурации
    git diff HEAD~1 --name-only | grep -E '\.github/workflows/' && \
      echo "::warning::CI configuration changed"
```

---

## Полный пример безопасного CI/CD

```yaml
# .github/workflows/secure-pipeline.yml
name: Secure CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

permissions:
  contents: read
  security-events: write

jobs:
  # 1. Статический анализ
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Bandit
        run: |
          pip install bandit
          bandit -r src/ -f sarif -o bandit.sarif || true

      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/python

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: bandit.sarif

  # 2. Проверка зависимостей
  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Safety check
        run: |
          pip install safety
          safety check -r requirements.txt

      - name: License check
        run: |
          pip install pip-licenses
          pip-licenses --fail-on="GPL;AGPL"

  # 3. Тесты
  test:
    needs: [security-scan, dependency-scan]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tests
        run: |
          pip install -r requirements.txt
          pytest --cov=src tests/

  # 4. Сборка образа
  build:
    needs: test
    runs-on: ubuntu-latest
    outputs:
      image: ${{ steps.build.outputs.image }}
    steps:
      - uses: actions/checkout@v4

      - name: Lint Dockerfile
        uses: hadolint/hadolint-action@v3.1.0

      - name: Build image
        id: build
        run: |
          IMAGE=ghcr.io/${{ github.repository }}:${{ github.sha }}
          docker build -t $IMAGE .
          echo "image=$IMAGE" >> $GITHUB_OUTPUT

      - name: Scan image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}:${{ github.sha }}
          severity: 'CRITICAL,HIGH'
          exit-code: 1

  # 5. Деплой (только main)
  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://api.example.com

    permissions:
      contents: read
      id-token: write

    steps:
      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: eu-west-1

      - name: Deploy
        run: |
          aws ecs update-service \
            --cluster production \
            --service api \
            --force-new-deployment
```

---

## Best Practices

1. **Принцип наименьших привилегий** — CI имеет только необходимые права
2. **Секреты в Vault/Secrets Manager** — не в коде и не в переменных
3. **Сканирование на каждый PR** — не допускать уязвимости в main
4. **Подпись артефактов** — гарантия целостности
5. **Ручное одобрение production** — человек в цепочке
6. **Аудит всех действий** — логирование для расследований
7. **Регулярная ротация секретов** — минимизация ущерба при утечке
8. **Изоляция окружений** — staging не имеет доступа к production

---

## Чек-лист CI/CD Security

- [ ] Секреты хранятся в Secrets Manager, не в коде
- [ ] Включено сканирование зависимостей (Dependabot, Safety)
- [ ] Настроен SAST (Bandit, Semgrep, SonarQube)
- [ ] Docker образы сканируются на уязвимости (Trivy)
- [ ] Branch protection включён для main
- [ ] Минимальные permissions в workflows
- [ ] OIDC вместо статических credentials
- [ ] Environment protection для production
- [ ] Ведётся аудит CI/CD активности
- [ ] Артефакты подписываются (cosign)
