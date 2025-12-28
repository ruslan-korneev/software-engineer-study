# OWASP Top 10 — Критические уязвимости веб-приложений

[prev: 01-https](./01-https.md) | [next: 03-cors](./03-cors.md)

---

## Что такое OWASP?

**OWASP** (Open Web Application Security Project) — это некоммерческая организация, которая занимается улучшением безопасности программного обеспечения. OWASP Top 10 — это регулярно обновляемый список наиболее критичных угроз безопасности веб-приложений.

## OWASP Top 10 (2021)

### 1. A01:2021 — Broken Access Control (Нарушение контроля доступа)

**Описание:** Пользователи могут получить доступ к данным или функциям, к которым у них не должно быть доступа.

**Примеры уязвимости:**
```python
# УЯЗВИМЫЙ КОД: Прямой доступ к объектам по ID
@app.route('/api/users/<user_id>/profile')
def get_profile(user_id):
    # Любой может посмотреть профиль любого пользователя!
    return User.query.get(user_id).to_dict()

# БЕЗОПАСНЫЙ КОД: Проверка прав доступа
@app.route('/api/users/<user_id>/profile')
@login_required
def get_profile(user_id):
    # Проверяем, что пользователь запрашивает свой профиль
    if current_user.id != int(user_id) and not current_user.is_admin:
        abort(403)
    return User.query.get(user_id).to_dict()
```

**Защита:**
- Реализуйте проверку авторизации на сервере
- Используйте RBAC (Role-Based Access Control)
- Отключите просмотр директорий
- Логируйте ошибки доступа

---

### 2. A02:2021 — Cryptographic Failures (Криптографические сбои)

**Описание:** Неправильное использование или отсутствие криптографии для защиты конфиденциальных данных.

**Примеры уязвимости:**
```python
# УЯЗВИМЫЙ КОД: Хранение пароля в открытом виде
def create_user(username, password):
    user = User(username=username, password=password)  # ОПАСНО!
    db.session.add(user)

# УЯЗВИМЫЙ КОД: Использование устаревшего MD5
import hashlib
password_hash = hashlib.md5(password.encode()).hexdigest()  # ОПАСНО!

# БЕЗОПАСНЫЙ КОД: Использование bcrypt
from bcrypt import hashpw, gensalt, checkpw

def create_user(username, password):
    password_hash = hashpw(password.encode(), gensalt())
    user = User(username=username, password_hash=password_hash)
    db.session.add(user)

def verify_password(password, password_hash):
    return checkpw(password.encode(), password_hash)
```

**Защита:**
- Используйте современные алгоритмы (AES-256, bcrypt, Argon2)
- Шифруйте данные при передаче (TLS 1.2+)
- Не храните чувствительные данные без необходимости
- Используйте безопасные генераторы случайных чисел

---

### 3. A03:2021 — Injection (Инъекции)

**Описание:** Вредоносные данные отправляются в интерпретатор как часть команды или запроса.

**SQL Injection:**
```python
# УЯЗВИМЫЙ КОД
query = f"SELECT * FROM users WHERE username = '{username}'"
# Атака: username = "admin'--" даст доступ к admin

# БЕЗОПАСНЫЙ КОД: Параметризованные запросы
cursor.execute("SELECT * FROM users WHERE username = %s", (username,))

# SQLAlchemy ORM (безопасно по умолчанию)
user = User.query.filter_by(username=username).first()
```

**Command Injection:**
```python
# УЯЗВИМЫЙ КОД
import os
os.system(f"ping {user_input}")  # ОПАСНО!
# Атака: user_input = "google.com; rm -rf /"

# БЕЗОПАСНЫЙ КОД
import subprocess
subprocess.run(["ping", "-c", "4", user_input], capture_output=True)
```

**NoSQL Injection (MongoDB):**
```javascript
// УЯЗВИМЫЙ КОД
db.users.find({ username: req.body.username, password: req.body.password });
// Атака: { "username": "admin", "password": { "$ne": "" } }

// БЕЗОПАСНЫЙ КОД: Валидация типов
const username = String(req.body.username);
const password = String(req.body.password);
db.users.find({ username, password });
```

---

### 4. A04:2021 — Insecure Design (Небезопасный дизайн)

**Описание:** Отсутствие мер безопасности на этапе проектирования.

**Пример:** Система восстановления пароля через "секретные вопросы":
```python
# НЕБЕЗОПАСНЫЙ ДИЗАЙН: Секретные вопросы легко угадать
def reset_password(username, mother_maiden_name):
    user = User.query.filter_by(username=username).first()
    if user.secret_answer == mother_maiden_name:
        return generate_new_password()

# БЕЗОПАСНЫЙ ДИЗАЙН: Токен на email
def request_password_reset(email):
    user = User.query.filter_by(email=email).first()
    if user:
        token = generate_secure_token()
        cache.set(f"reset:{token}", user.id, timeout=3600)
        send_email(email, f"Reset link: /reset?token={token}")

def reset_password(token, new_password):
    user_id = cache.get(f"reset:{token}")
    if not user_id:
        abort(400, "Invalid or expired token")
    # Сбрасываем пароль...
    cache.delete(f"reset:{token}")
```

**Защита:**
- Моделирование угроз на этапе проектирования
- Принцип минимальных привилегий
- Безопасные паттерны проектирования

---

### 5. A05:2021 — Security Misconfiguration (Неправильная конфигурация)

**Описание:** Использование небезопасных настроек по умолчанию или ошибки конфигурации.

**Примеры проблем:**
```python
# ОПАСНО: Debug-режим в production
app = Flask(__name__)
app.config['DEBUG'] = True  # Показывает stack trace пользователям!

# БЕЗОПАСНО
app.config['DEBUG'] = os.environ.get('FLASK_ENV') == 'development'
```

```yaml
# docker-compose.yml - ОПАСНО
services:
  db:
    image: postgres
    environment:
      POSTGRES_PASSWORD: postgres  # Пароль по умолчанию!
    ports:
      - "5432:5432"  # Открыт в интернет!
```

**Чек-лист безопасной конфигурации:**
- [ ] Удалены стандартные учётные записи
- [ ] Отключены ненужные сервисы и порты
- [ ] Настроены заголовки безопасности
- [ ] Обновлены все зависимости
- [ ] Отключены directory listing и debug-режим

---

### 6. A06:2021 — Vulnerable and Outdated Components (Уязвимые компоненты)

**Описание:** Использование библиотек с известными уязвимостями.

**Проверка уязвимостей:**
```bash
# Python
pip install safety
safety check

# Node.js
npm audit
npm audit fix

# Автоматическое обновление (GitHub Dependabot)
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
```

**Защита:**
- Регулярно обновляйте зависимости
- Используйте только поддерживаемые версии
- Удаляйте неиспользуемые зависимости
- Подпишитесь на уведомления о CVE

---

### 7. A07:2021 — Identification and Authentication Failures

**Описание:** Ошибки в системе аутентификации.

```python
# УЯЗВИМЫЙ КОД: Нет защиты от брутфорса
@app.route('/login', methods=['POST'])
def login():
    user = User.query.filter_by(username=request.form['username']).first()
    if user and user.check_password(request.form['password']):
        login_user(user)
        return redirect('/dashboard')
    return 'Invalid credentials'

# БЕЗОПАСНЫЙ КОД: Rate limiting + защита от перебора
from flask_limiter import Limiter

limiter = Limiter(app, key_func=get_remote_address)

@app.route('/login', methods=['POST'])
@limiter.limit("5 per minute")  # Максимум 5 попыток в минуту
def login():
    user = User.query.filter_by(username=request.form['username']).first()

    # Проверяем блокировку аккаунта
    if user and user.is_locked():
        return 'Account temporarily locked', 429

    if user and user.check_password(request.form['password']):
        user.reset_failed_attempts()
        login_user(user)
        return redirect('/dashboard')

    if user:
        user.increment_failed_attempts()

    # Одинаковое сообщение (не раскрываем существование пользователя)
    return 'Invalid credentials', 401
```

**Защита:**
- Многофакторная аутентификация (MFA)
- Сильная политика паролей
- Rate limiting для логина
- Безопасное хранение сессий

---

### 8. A08:2021 — Software and Data Integrity Failures

**Описание:** Нарушение целостности ПО или данных.

```python
# УЯЗВИМЫЙ КОД: Десериализация непроверенных данных
import pickle

@app.route('/api/load')
def load_data():
    data = request.get_data()
    return pickle.loads(data)  # ОПАСНО! RCE возможен

# БЕЗОПАСНЫЙ КОД: Использование безопасных форматов
import json

@app.route('/api/load')
def load_data():
    return json.loads(request.get_data())
```

**Защита:**
- Проверяйте цифровые подписи пакетов
- Используйте CI/CD с проверкой целостности
- Избегайте небезопасной десериализации

---

### 9. A09:2021 — Security Logging and Monitoring Failures

**Описание:** Отсутствие логирования и мониторинга безопасности.

```python
import logging
from datetime import datetime

# Настройка логирования безопасности
security_logger = logging.getLogger('security')
security_logger.setLevel(logging.INFO)

handler = logging.FileHandler('/var/log/app/security.log')
handler.setFormatter(logging.Formatter(
    '%(asctime)s - %(levelname)s - %(message)s'
))
security_logger.addHandler(handler)

@app.route('/login', methods=['POST'])
def login():
    username = request.form['username']
    ip = request.remote_addr

    user = authenticate(username, request.form['password'])

    if user:
        security_logger.info(f"LOGIN_SUCCESS user={username} ip={ip}")
        return redirect('/dashboard')
    else:
        security_logger.warning(f"LOGIN_FAILED user={username} ip={ip}")
        return 'Invalid credentials', 401

# Логирование критических действий
def delete_user(user_id):
    security_logger.warning(
        f"USER_DELETED user_id={user_id} by={current_user.id} ip={request.remote_addr}"
    )
```

**Что логировать:**
- Успешные и неуспешные попытки входа
- Ошибки доступа (403)
- Изменения критических данных
- Административные действия

---

### 10. A10:2021 — Server-Side Request Forgery (SSRF)

**Описание:** Сервер делает запросы к произвольным URL по указанию пользователя.

```python
# УЯЗВИМЫЙ КОД
import requests

@app.route('/fetch')
def fetch_url():
    url = request.args.get('url')
    response = requests.get(url)  # SSRF!
    # Атака: url=http://169.254.169.254/latest/meta-data/ (AWS metadata)
    return response.text

# БЕЗОПАСНЫЙ КОД: Белый список доменов
ALLOWED_DOMAINS = ['api.example.com', 'cdn.example.com']

@app.route('/fetch')
def fetch_url():
    url = request.args.get('url')
    parsed = urlparse(url)

    # Проверка домена
    if parsed.netloc not in ALLOWED_DOMAINS:
        abort(400, 'Domain not allowed')

    # Блокируем внутренние IP
    try:
        ip = socket.gethostbyname(parsed.hostname)
        if ipaddress.ip_address(ip).is_private:
            abort(400, 'Internal IPs not allowed')
    except:
        abort(400, 'Invalid hostname')

    return requests.get(url).text
```

## Инструменты для тестирования

```bash
# OWASP ZAP — автоматический сканер
docker run -t owasp/zap2docker-stable zap-baseline.py -t https://example.com

# SQLMap — поиск SQL-инъекций
sqlmap -u "http://example.com/page?id=1" --dbs

# Burp Suite — ручное тестирование
# https://portswigger.net/burp
```

## Заключение

OWASP Top 10 — это фундамент для понимания веб-безопасности. Ключевые принципы защиты:

1. **Никогда не доверяй входным данным** — всегда валидируй
2. **Принцип наименьших привилегий** — давай минимум необходимых прав
3. **Защита в глубину** — несколько уровней защиты
4. **Безопасность по умолчанию** — всё закрыто, пока явно не открыто
5. **Регулярное обновление** — следи за уязвимостями в зависимостях

---

[prev: 01-https](./01-https.md) | [next: 03-cors](./03-cors.md)
