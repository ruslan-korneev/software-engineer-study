# Мониторинг API Security

## Зачем нужен мониторинг безопасности?

Мониторинг — критически важный компонент защиты API. Он позволяет:
- Обнаруживать атаки в реальном времени
- Анализировать подозрительную активность
- Реагировать на инциденты до нанесения ущерба
- Собирать доказательства для расследований

---

## 1. Централизованное логирование

### Принцип
Все сервисы и компоненты должны отправлять логи в единое хранилище.

### Популярные решения
- **ELK Stack** (Elasticsearch, Logstash, Kibana)
- **Loki + Grafana**
- **Splunk**
- **AWS CloudWatch Logs**
- **Datadog**

### Пример структуры лога

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "service": "api-gateway",
  "level": "warn",
  "request_id": "abc-123-def",
  "user_id": "user_456",
  "ip": "192.168.1.100",
  "method": "POST",
  "path": "/api/v1/payments",
  "status": 401,
  "response_time_ms": 45,
  "error": "Invalid authentication token"
}
```

### Что логировать
- Все запросы к API (метод, путь, статус, время ответа)
- Попытки аутентификации (успешные и неудачные)
- Изменения критических данных
- Ошибки и исключения
- Действия администраторов

---

## 2. Мониторинг запросов, ответов и ошибок

### Метрики для отслеживания

| Метрика | Описание | Порог тревоги |
|---------|----------|---------------|
| Request Rate | Количество запросов в секунду | Внезапный рост >200% |
| Error Rate | Процент ошибок 4xx/5xx | >5% от общего трафика |
| Latency | Время ответа | p95 > 500ms |
| Failed Auth | Неудачные попытки входа | >10 с одного IP за минуту |

### Пример настройки в Prometheus

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'api-server'
    static_configs:
      - targets: ['api:8080']
    metrics_path: '/metrics'
```

### Пример middleware для сбора метрик (Python/FastAPI)

```python
from prometheus_client import Counter, Histogram
import time

REQUEST_COUNT = Counter(
    'api_requests_total',
    'Total API requests',
    ['method', 'endpoint', 'status']
)

REQUEST_LATENCY = Histogram(
    'api_request_latency_seconds',
    'Request latency in seconds',
    ['method', 'endpoint']
)

@app.middleware("http")
async def metrics_middleware(request, call_next):
    start_time = time.time()
    response = await call_next(request)

    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=request.url.path,
        status=response.status_code
    ).inc()

    REQUEST_LATENCY.labels(
        method=request.method,
        endpoint=request.url.path
    ).observe(time.time() - start_time)

    return response
```

---

## 3. Настройка алертов

### Каналы оповещения
- **Email** — для некритичных уведомлений
- **Slack/Teams** — для командной работы
- **SMS/Phone** — для критических инцидентов
- **PagerDuty/OpsGenie** — для дежурств

### Примеры правил алертов

```yaml
# alertmanager rules
groups:
  - name: api_security_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(api_requests_total{status=~"5.."}[5m]) > 0.1
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"

      - alert: BruteForceAttempt
        expr: rate(auth_failures_total[1m]) > 10
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Possible brute force attack"

      - alert: UnusualTrafficSpike
        expr: rate(api_requests_total[5m]) > 2 * avg_over_time(rate(api_requests_total[5m])[1h:5m])
        for: 5m
        labels:
          severity: warning
```

---

## 4. Не логировать sensitive data!

### Что НЕЛЬЗЯ записывать в логи
- Пароли и секреты
- Токены доступа (JWT, API keys)
- Номера кредитных карт
- Персональные данные (паспорт, SSN)
- Медицинские данные

### Как защитить данные в логах

```python
import re

def sanitize_log(data: dict) -> dict:
    """Маскирует чувствительные данные перед логированием"""
    sensitive_keys = ['password', 'token', 'api_key', 'credit_card', 'ssn']
    sanitized = data.copy()

    for key in sanitized:
        if any(s in key.lower() for s in sensitive_keys):
            sanitized[key] = '***REDACTED***'
        elif isinstance(sanitized[key], str):
            # Маскируем номера карт
            sanitized[key] = re.sub(
                r'\b\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}\b',
                '****-****-****-****',
                sanitized[key]
            )

    return sanitized
```

### Пример middleware для очистки логов

```python
@app.middleware("http")
async def sanitize_logging_middleware(request, call_next):
    # Не логируем тело запросов к чувствительным эндпоинтам
    sensitive_paths = ['/auth/login', '/auth/register', '/payments']

    if request.url.path in sensitive_paths:
        logger.info(f"Request to {request.url.path} - body redacted")
    else:
        body = await request.body()
        logger.info(f"Request: {sanitize_log(json.loads(body))}")

    return await call_next(request)
```

---

## 5. IDS/IPS системы

### Intrusion Detection System (IDS)
Обнаруживает подозрительную активность и уведомляет администраторов.

### Intrusion Prevention System (IPS)
Автоматически блокирует обнаруженные угрозы.

### Популярные решения

| Система | Тип | Описание |
|---------|-----|----------|
| **ModSecurity** | WAF | Web Application Firewall с OWASP Core Rule Set |
| **Snort** | NIDS | Network-based IDS с открытым исходным кодом |
| **Suricata** | NIDS/NIPS | Высокопроизводительный анализатор трафика |
| **OSSEC** | HIDS | Host-based IDS с анализом логов |
| **Fail2ban** | IPS | Блокировка IP после неудачных попыток |
| **AWS WAF** | Cloud WAF | Управляемый WAF от Amazon |
| **Cloudflare WAF** | Cloud WAF | WAF с защитой от DDoS |

### Пример конфигурации Fail2ban

```ini
# /etc/fail2ban/jail.local
[api-auth]
enabled = true
port = http,https
filter = api-auth
logpath = /var/log/api/auth.log
maxretry = 5
findtime = 60
bantime = 3600

# /etc/fail2ban/filter.d/api-auth.conf
[Definition]
failregex = ^.*"status": 401.*"ip": "<HOST>".*$
ignoreregex =
```

### Пример правила ModSecurity

```apache
# Блокировка SQL injection
SecRule ARGS "@detectSQLi" \
    "id:1001,\
    phase:2,\
    block,\
    msg:'SQL Injection Attack Detected',\
    logdata:'Matched Data: %{MATCHED_VAR}',\
    severity:'CRITICAL'"

# Блокировка XSS
SecRule ARGS "@detectXSS" \
    "id:1002,\
    phase:2,\
    block,\
    msg:'XSS Attack Detected'"
```

---

## Best Practices

1. **Логируйте всё, кроме секретов** — больше данных = лучше анализ
2. **Используйте correlation ID** — для отслеживания запроса через все сервисы
3. **Настройте retention policy** — храните логи 30-90 дней
4. **Регулярно проверяйте алерты** — избегайте alert fatigue
5. **Автоматизируйте реагирование** — блокировка IP, уведомление команды
6. **Тестируйте мониторинг** — проводите учебные атаки

---

## Чек-лист внедрения

- [ ] Настроено централизованное логирование
- [ ] Все сервисы отправляют логи в единое хранилище
- [ ] Настроены метрики (request rate, error rate, latency)
- [ ] Настроены алерты для критических событий
- [ ] Sensitive data не попадает в логи
- [ ] Установлен WAF/IDS
- [ ] Настроена автоматическая блокировка подозрительных IP
- [ ] Команда знает, как реагировать на алерты
