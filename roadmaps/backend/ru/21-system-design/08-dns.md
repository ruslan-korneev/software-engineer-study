# DNS (Domain Name System)

[prev: 07-availability-patterns](./07-availability-patterns.md) | [next: 09-cdn](./09-cdn.md)

---

## Что такое DNS и зачем он нужен

**DNS (Domain Name System)** — это распределённая иерархическая система, которая преобразует доменные имена (например, `google.com`) в IP-адреса (например, `142.250.74.46`). DNS часто называют "телефонной книгой интернета".

### Зачем нужен DNS

```
Без DNS:                          С DNS:
http://142.250.74.46     →     http://google.com
http://151.101.1.140     →     http://reddit.com
http://31.13.80.36       →     http://facebook.com
```

**Основные функции:**
- **Человекочитаемые адреса** — проще запомнить `amazon.com`, чем `54.239.28.85`
- **Абстракция от инфраструктуры** — IP может меняться, а домен остаётся
- **Балансировка нагрузки** — один домен может указывать на несколько серверов
- **Географическая маршрутизация** — направление пользователей к ближайшему серверу
- **Отказоустойчивость** — переключение на резервные серверы при сбоях

---

## Иерархия DNS

DNS имеет древовидную иерархическую структуру:

```
                         ┌─────────────┐
                         │  Root (.)   │
                         │  13 серверов │
                         └──────┬──────┘
                                │
            ┌───────────────────┼───────────────────┐
            │                   │                   │
      ┌─────▼─────┐       ┌─────▼─────┐       ┌─────▼─────┐
      │   .com    │       │   .org    │       │   .ru     │
      │   (TLD)   │       │   (TLD)   │       │   (TLD)   │
      └─────┬─────┘       └─────┬─────┘       └─────┬─────┘
            │                   │                   │
    ┌───────┼───────┐           │                   │
    │       │       │           │                   │
┌───▼──┐ ┌──▼──┐ ┌──▼──┐   ┌────▼────┐        ┌────▼────┐
│google│ │amazon│ │meta│   │wikipedia│        │yandex   │
│.com  │ │.com  │ │.com│   │.org     │        │.ru      │
└──────┘ └─────┘ └─────┘   └─────────┘        └─────────┘
```

### 1. Root DNS Servers (Корневые серверы)

- **13 логических серверов** (от A до M), но физически их сотни по всему миру (anycast)
- Управляются организациями: ICANN, Verisign, NASA, US Army и др.
- Знают адреса всех TLD-серверов
- Получают ~100 миллиардов запросов в день

```
Примеры корневых серверов:
a.root-servers.net  (Verisign)
b.root-servers.net  (USC-ISI)
...
m.root-servers.net  (WIDE Project, Япония)
```

### 2. TLD Servers (Top-Level Domain)

**Типы TLD:**

| Тип | Описание | Примеры |
|-----|----------|---------|
| **gTLD** | Generic (общие) | .com, .org, .net, .info |
| **ccTLD** | Country Code (страновые) | .ru, .uk, .de, .jp |
| **sTLD** | Sponsored (спонсируемые) | .gov, .edu, .mil |
| **New gTLD** | Новые общие | .app, .dev, .io, .ai |

TLD-серверы знают адреса авторитативных серверов для каждого домена второго уровня.

### 3. Authoritative DNS Servers (Авторитативные серверы)

- **Содержат актуальные DNS-записи** для конкретного домена
- Являются "источником правды" для домена
- Управляются владельцами доменов или хостинг-провайдерами

```
Пример: авторитативные серверы для google.com
ns1.google.com
ns2.google.com
ns3.google.com
ns4.google.com
```

### 4. Recursive Resolver (Рекурсивный резолвер)

- **Посредник** между клиентом и DNS-иерархией
- Выполняет всю работу по поиску IP-адреса
- Кэширует результаты для ускорения повторных запросов
- Обычно предоставляется ISP или публичными сервисами (Google 8.8.8.8, Cloudflare 1.1.1.1)

---

## Типы DNS записей

### Основные типы записей

#### A (Address) Record
Связывает домен с IPv4-адресом.

```dns
example.com.     IN    A    93.184.216.34
www.example.com. IN    A    93.184.216.34
```

#### AAAA Record
Связывает домен с IPv6-адресом.

```dns
example.com.     IN    AAAA    2606:2800:220:1:248:1893:25c8:1946
```

#### CNAME (Canonical Name) Record
Создаёт алиас (псевдоним) для другого домена.

```dns
www.example.com.    IN    CNAME    example.com.
blog.example.com.   IN    CNAME    example.wordpress.com.
cdn.example.com.    IN    CNAME    d111111abcdef8.cloudfront.net.
```

**Ограничения CNAME:**
- Нельзя использовать для корневого домена (apex domain)
- Нельзя совмещать с другими записями для того же имени

#### MX (Mail Exchange) Record
Указывает почтовые серверы для домена.

```dns
example.com.    IN    MX    10    mail1.example.com.
example.com.    IN    MX    20    mail2.example.com.
example.com.    IN    MX    30    backup-mail.example.com.
```
*Число (10, 20, 30) — приоритет. Меньшее значение = выше приоритет.*

#### NS (Name Server) Record
Указывает авторитативные DNS-серверы для домена.

```dns
example.com.    IN    NS    ns1.example.com.
example.com.    IN    NS    ns2.example.com.
```

#### TXT Record
Содержит произвольную текстовую информацию.

```dns
# SPF запись для email
example.com.    IN    TXT    "v=spf1 include:_spf.google.com ~all"

# Верификация домена
example.com.    IN    TXT    "google-site-verification=abc123..."

# DKIM подпись
selector._domainkey.example.com. IN TXT "v=DKIM1; k=rsa; p=MIGf..."
```

#### PTR (Pointer) Record
Обратный DNS — преобразует IP в домен (reverse DNS).

```dns
# Для IP 192.0.2.1
1.2.0.192.in-addr.arpa.    IN    PTR    host.example.com.
```

Используется для:
- Проверки email-серверов (anti-spam)
- Логирования и отладки
- Верификации серверов

#### SRV (Service) Record
Указывает хост и порт для определённого сервиса.

```dns
_service._protocol.name.    TTL    IN    SRV    priority weight port target.

# Пример: SIP сервис
_sip._tcp.example.com.   IN    SRV    10 60 5060 sipserver.example.com.
_sip._tcp.example.com.   IN    SRV    10 40 5060 sipbackup.example.com.

# Пример: XMPP/Jabber
_xmpp-server._tcp.example.com. IN SRV 5 0 5269 xmpp.example.com.
```

### Таблица сравнения записей

| Тип | Назначение | Пример значения |
|-----|-----------|-----------------|
| A | IPv4 адрес | 192.168.1.1 |
| AAAA | IPv6 адрес | 2001:db8::1 |
| CNAME | Алиас домена | other.example.com |
| MX | Почтовый сервер | mail.example.com (priority 10) |
| NS | DNS сервер | ns1.example.com |
| TXT | Текстовые данные | "v=spf1 ..." |
| PTR | Обратный DNS | host.example.com |
| SRV | Сервис + порт | server.example.com:5060 |

---

## Процесс DNS Resolution

### Рекурсивный запрос (Recursive Query)

Клиент делает один запрос к рекурсивному резолверу, который выполняет всю работу.

```
┌────────┐                 ┌──────────────┐
│ Клиент │ ─── запрос ───► │  Recursive   │
│        │                 │  Resolver    │
│        │ ◄── ответ ────  │  (8.8.8.8)   │
└────────┘                 └──────┬───────┘
                                  │
        Резолвер сам обходит всю иерархию DNS
                                  │
                    ┌─────────────┼─────────────┐
                    ▼             ▼             ▼
              ┌─────────┐  ┌───────────┐  ┌───────────┐
              │  Root   │  │    TLD    │  │Authoritative│
              │ Server  │  │  Server   │  │   Server   │
              └─────────┘  └───────────┘  └───────────┘
```

### Итеративный запрос (Iterative Query)

Сервер возвращает лучший известный ему ответ (часто — адрес следующего сервера).

```
Клиент: "Какой IP у www.example.com?"

┌────────┐         ┌─────────┐
│ Client │ ──1──►  │  Root   │
│        │ ◄──2──  │ Server  │  "Не знаю, спроси .com TLD"
└────────┘         └─────────┘

┌────────┐         ┌─────────┐
│ Client │ ──3──►  │  .com   │
│        │ ◄──4──  │  TLD    │  "Не знаю, спроси ns1.example.com"
└────────┘         └─────────┘

┌────────┐         ┌───────────────┐
│ Client │ ──5──►  │ Authoritative │
│        │ ◄──6──  │   (example)   │  "IP: 93.184.216.34"
└────────┘         └───────────────┘
```

### Полный процесс разрешения DNS

```
Пользователь вводит: www.example.com

1. Браузер проверяет свой кэш
   └─► Если есть — использует закэшированный IP

2. ОС проверяет свой кэш и файл /etc/hosts
   └─► Если есть — возвращает IP

3. Запрос к рекурсивному резолверу (обычно от ISP)
   └─► Резолвер проверяет свой кэш
       └─► Если есть — возвращает IP

4. Резолвер начинает итеративный обход:

   4.1 Запрос к Root Server (.)
       └─► Ответ: "Спроси .com TLD" + адреса TLD серверов

   4.2 Запрос к .com TLD Server
       └─► Ответ: "Спроси example.com" + адреса NS серверов

   4.3 Запрос к Authoritative Server (ns1.example.com)
       └─► Ответ: "93.184.216.34" + TTL

5. Резолвер кэширует ответ и возвращает клиенту

6. Браузер подключается к 93.184.216.34
```

### Время разрешения DNS

```
Холодный запрос (без кэша):
Root → TLD → Authoritative = 100-300 мс

Запрос из кэша резолвера:
< 10 мс

Запрос из кэша браузера:
< 1 мс
```

---

## DNS Кэширование и TTL

### Уровни кэширования

```
┌─────────────────────────────────────────────────────────┐
│                    Кэширование DNS                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌───────────┐                                          │
│  │ Браузер   │  ← Кэш DNS записей (Chrome: 1 мин)      │
│  └─────┬─────┘                                          │
│        │                                                │
│  ┌─────▼─────┐                                          │
│  │    ОС     │  ← Системный DNS кэш                    │
│  └─────┬─────┘    Windows: ipconfig /displaydns        │
│        │          Linux: systemd-resolve --statistics   │
│        │          Mac: dscacheutil -statistics          │
│  ┌─────▼─────┐                                          │
│  │  Router   │  ← Локальный DNS кэш роутера            │
│  └─────┬─────┘                                          │
│        │                                                │
│  ┌─────▼─────┐                                          │
│  │ ISP DNS   │  ← Кэш провайдера                       │
│  └─────┬─────┘                                          │
│        │                                                │
│  ┌─────▼─────┐                                          │
│  │  Public   │  ← Google (8.8.8.8), Cloudflare (1.1.1.1)│
│  │  Resolver │                                          │
│  └───────────┘                                          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### TTL (Time To Live)

TTL определяет, как долго DNS-запись может храниться в кэше.

```dns
# Примеры TTL
example.com.    300     IN    A    93.184.216.34    # 5 минут
example.com.    3600    IN    A    93.184.216.34    # 1 час
example.com.    86400   IN    A    93.184.216.34    # 1 день
```

### Выбор TTL

| TTL | Когда использовать |
|-----|-------------------|
| **30-300 сек** | Динамические сервисы, failover, A/B тесты |
| **300-3600 сек** | Стандартные веб-приложения |
| **3600-86400 сек** | Стабильные сервисы, CDN |
| **86400+ сек** | Редко меняющиеся записи (MX, NS) |

**Компромисс:**
- **Низкий TTL**: быстрое распространение изменений, но больше нагрузка на DNS
- **Высокий TTL**: меньше DNS-запросов, но медленное распространение изменений

### Стратегия при миграции

```
До миграции (за 24-48 часов):
1. Снизить TTL с 86400 до 300 секунд
2. Дождаться, пока старый TTL истечёт везде

Во время миграции:
3. Изменить IP-адрес в DNS
4. Изменения распространятся за 5 минут

После миграции:
5. Убедиться, что всё работает
6. Вернуть TTL обратно к 86400
```

---

## DNS в контексте System Design

### 1. Round-Robin DNS для балансировки нагрузки

Простейший способ распределить нагрузку между серверами.

```dns
# Несколько A-записей для одного домена
api.example.com.    300    IN    A    192.168.1.1
api.example.com.    300    IN    A    192.168.1.2
api.example.com.    300    IN    A    192.168.1.3
```

**Как работает:**

```
Запрос 1 → 192.168.1.1, 192.168.1.2, 192.168.1.3
Запрос 2 → 192.168.1.2, 192.168.1.3, 192.168.1.1
Запрос 3 → 192.168.1.3, 192.168.1.1, 192.168.1.2
...
```

**Преимущества:**
- Простота реализации
- Не требует дополнительной инфраструктуры
- Бесплатно

**Недостатки:**
- Нет health checks — трафик идёт даже на упавшие серверы
- Неравномерное распределение из-за кэширования
- Нет учёта нагрузки серверов
- Нет session affinity

### 2. GeoDNS (Географическая маршрутизация)

Направляет пользователей к ближайшему серверу на основе их геолокации.

```
                         ┌──────────────┐
                         │   GeoDNS     │
                         │   Provider   │
                         └──────┬───────┘
                                │
         ┌──────────────────────┼──────────────────────┐
         │                      │                      │
         ▼                      ▼                      ▼
    ┌─────────┐           ┌─────────┐           ┌─────────┐
    │  US DC  │           │  EU DC  │           │ Asia DC │
    │10.0.1.1 │           │10.0.2.1 │           │10.0.3.1 │
    └─────────┘           └─────────┘           └─────────┘
         ▲                      ▲                      ▲
         │                      │                      │
    Пользователи            Пользователи          Пользователи
    из США                  из Европы             из Азии
```

**Пример конфигурации (AWS Route 53):**

```json
{
  "Name": "api.example.com",
  "Type": "A",
  "SetIdentifier": "us-east",
  "GeoLocation": {
    "ContinentCode": "NA"
  },
  "TTL": 300,
  "ResourceRecords": [{"Value": "10.0.1.1"}]
}

{
  "Name": "api.example.com",
  "Type": "A",
  "SetIdentifier": "eu-west",
  "GeoLocation": {
    "ContinentCode": "EU"
  },
  "TTL": 300,
  "ResourceRecords": [{"Value": "10.0.2.1"}]
}
```

**Применение:**
- CDN (Cloudflare, Akamai, CloudFront)
- Глобально распределённые приложения
- Соответствие требованиям по локализации данных (GDPR)

### 3. DNS Failover

Автоматическое переключение на резервный сервер при сбое основного.

```
Нормальная работа:
┌────────┐     ┌─────────────┐     ┌─────────────┐
│ Client │────►│ DNS + Health│────►│ Primary     │ ✓ Healthy
└────────┘     │   Checks    │     │ Server      │
               └─────────────┘     └─────────────┘

При сбое Primary:
┌────────┐     ┌─────────────┐     ┌─────────────┐
│ Client │────►│ DNS + Health│  ✗  │ Primary     │ ✗ Unhealthy
└────────┘     │   Checks    │     │ Server      │
               └──────┬──────┘     └─────────────┘
                      │
                      ▼
               ┌─────────────┐
               │ Secondary   │ ✓ Healthy
               │ Server      │
               └─────────────┘
```

**Типы Health Checks:**
- **HTTP/HTTPS** — проверка endpoint'а
- **TCP** — проверка порта
- **DNS** — проверка DNS-ответа

**Пример конфигурации (AWS Route 53):**

```json
{
  "Name": "api.example.com",
  "Type": "A",
  "SetIdentifier": "primary",
  "Failover": "PRIMARY",
  "HealthCheckId": "health-check-id-123",
  "TTL": 60,
  "ResourceRecords": [{"Value": "10.0.1.1"}]
}

{
  "Name": "api.example.com",
  "Type": "A",
  "SetIdentifier": "secondary",
  "Failover": "SECONDARY",
  "TTL": 60,
  "ResourceRecords": [{"Value": "10.0.2.1"}]
}
```

### 4. Weighted Routing

Распределение трафика по весам (для canary deployments, A/B testing).

```dns
# 90% трафика на стабильную версию
api.example.com.  IN  A  10.0.1.1  ; weight: 90

# 10% трафика на canary
api.example.com.  IN  A  10.0.1.2  ; weight: 10
```

### 5. Latency-Based Routing

Направляет на сервер с наименьшей задержкой.

```
Пользователь из Москвы:
├─► US Server: 150ms
├─► EU Server: 40ms   ← Выбран
└─► Asia Server: 180ms
```

---

## Проблемы и ограничения DNS

### 1. Проблемы безопасности

#### DNS Spoofing / Cache Poisoning
Атакующий подменяет DNS-ответы, перенаправляя на вредоносные серверы.

```
Нормальный процесс:
User → Resolver → "bank.com = 10.0.1.1" (правильный)

С атакой:
User → Resolver → "bank.com = 6.6.6.6" (сервер атакующего!)
```

**Защита: DNSSEC** — криптографическая подпись DNS-записей.

#### DNS Amplification Attack (DDoS)
Использование DNS для усиления DDoS-атак.

```
Атакующий отправляет маленький запрос (60 байт)
с поддельным source IP (жертвы)
        │
        ▼
DNS Server отвечает большим ответом (3000 байт)
на IP жертвы
        │
        ▼
Жертва получает огромный трафик
```

### 2. Задержка распространения (Propagation Delay)

```
Изменил A-запись в 10:00
        │
        ├─► Некоторые пользователи видят новый IP через 5 минут
        ├─► Другие — через час
        └─► Некоторые — через 24+ часов (старый кэш)
```

**Причина:** разные уровни кэширования с разными TTL.

### 3. Single Point of Failure

Если DNS недоступен, пользователи не могут подключиться даже к работающим серверам.

```
DNS сервер недоступен:
User: "Не могу открыть example.com"
Server: "Я работаю! Но никто не знает мой IP..."
```

**Решение:** множественные NS-серверы, anycast.

### 4. Ограничения кэширования

- Нельзя гарантировать, что все клиенты увидят обновление одновременно
- Некоторые резолверы игнорируют низкий TTL
- Браузеры имеют свои политики кэширования

### 5. Не подходит для real-time балансировки

DNS не знает о:
- Текущей нагрузке серверов
- Количестве активных соединений
- Производительности серверов

**Для real-time балансировки используйте Load Balancer (L4/L7).**

---

## Сравнение с другими подходами к балансировке

| Аспект | DNS | Hardware LB | Software LB |
|--------|-----|-------------|-------------|
| **Уровень OSI** | L7 (Application) | L4/L7 | L4/L7 |
| **Health Checks** | Ограниченные | Полные | Полные |
| **Скорость реакции** | Минуты (TTL) | Мгновенно | Мгновенно |
| **Session Affinity** | Нет | Да | Да |
| **Стоимость** | Низкая | Высокая | Средняя |
| **Масштабируемость** | Очень высокая | Ограничена | Высокая |

---

## Примеры использования

### Пример 1: Проверка DNS с помощью dig

```bash
# Полный запрос с трассировкой
$ dig +trace www.google.com

; <<>> DiG 9.18.1 <<>> +trace www.google.com
;; global options: +cmd
.                       518400  IN      NS      a.root-servers.net.
.                       518400  IN      NS      b.root-servers.net.
...
com.                    172800  IN      NS      a.gtld-servers.net.
...
google.com.             172800  IN      NS      ns1.google.com.
...
www.google.com.         300     IN      A       142.250.74.36

# Простой запрос
$ dig www.google.com +short
142.250.74.36

# Запрос конкретного типа записи
$ dig google.com MX +short
10 smtp.google.com.

# Запрос к конкретному DNS серверу
$ dig @8.8.8.8 example.com
```

### Пример 2: Настройка DNS для веб-приложения

```dns
; Зона example.com
$TTL 3600

; SOA запись
@   IN  SOA     ns1.example.com. admin.example.com. (
                2024010101  ; Serial
                3600        ; Refresh
                600         ; Retry
                604800      ; Expire
                86400       ; Minimum TTL
)

; Name servers
@           IN      NS      ns1.example.com.
@           IN      NS      ns2.example.com.

; Name server A records
ns1         IN      A       10.0.1.1
ns2         IN      A       10.0.2.1

; Main website
@           IN      A       93.184.216.34
www         IN      CNAME   @

; API servers (round-robin)
api         IN      A       10.0.1.10
api         IN      A       10.0.1.11
api         IN      A       10.0.1.12

; Mail
@           IN      MX      10      mail1.example.com.
@           IN      MX      20      mail2.example.com.
mail1       IN      A       10.0.2.1
mail2       IN      A       10.0.2.2

; Email authentication
@           IN      TXT     "v=spf1 mx include:_spf.google.com ~all"

; CDN
static      IN      CNAME   d111111abcdef8.cloudfront.net.
cdn         IN      CNAME   example.b-cdn.net.

; Development
dev         IN      A       10.0.3.1
staging     IN      A       10.0.3.2
```

### Пример 3: DNS Failover с AWS Route 53 (Terraform)

```hcl
# Health check для primary сервера
resource "aws_route53_health_check" "primary" {
  fqdn              = "primary.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = "3"
  request_interval  = "30"
}

# Primary record
resource "aws_route53_record" "primary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"
  ttl     = 60

  failover_routing_policy {
    type = "PRIMARY"
  }

  set_identifier  = "primary"
  health_check_id = aws_route53_health_check.primary.id
  records         = ["10.0.1.1"]
}

# Secondary (failover) record
resource "aws_route53_record" "secondary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"
  ttl     = 60

  failover_routing_policy {
    type = "SECONDARY"
  }

  set_identifier = "secondary"
  records        = ["10.0.2.1"]
}
```

### Пример 4: Программная работа с DNS (Python)

```python
import dns.resolver
import dns.rdatatype

def get_dns_records(domain: str, record_type: str = 'A') -> list:
    """Получить DNS записи для домена."""
    try:
        answers = dns.resolver.resolve(domain, record_type)
        return [str(rdata) for rdata in answers]
    except dns.resolver.NXDOMAIN:
        return []
    except dns.resolver.NoAnswer:
        return []

# Примеры использования
print("A records:", get_dns_records("google.com", "A"))
print("MX records:", get_dns_records("google.com", "MX"))
print("NS records:", get_dns_records("google.com", "NS"))
print("TXT records:", get_dns_records("google.com", "TXT"))

# Проверка всех записей
def get_all_records(domain: str) -> dict:
    """Получить все типы DNS записей."""
    record_types = ['A', 'AAAA', 'CNAME', 'MX', 'NS', 'TXT', 'SOA']
    result = {}

    for rtype in record_types:
        records = get_dns_records(domain, rtype)
        if records:
            result[rtype] = records

    return result

all_records = get_all_records("google.com")
for rtype, records in all_records.items():
    print(f"{rtype}: {records}")
```

---

## Лучшие практики

### Для надёжности:
1. **Используйте минимум 2 NS-сервера** в разных сетях
2. **Настройте мониторинг DNS** (время ответа, доступность)
3. **Используйте авторитетных DNS-провайдеров** (Cloudflare, AWS Route 53, Google Cloud DNS)

### Для производительности:
1. **Оптимизируйте TTL** — баланс между скоростью обновления и нагрузкой
2. **Используйте anycast DNS** для низкой латентности
3. **Минимизируйте цепочки CNAME** (каждый CNAME = дополнительный запрос)

### Для безопасности:
1. **Включите DNSSEC** для защиты от spoofing
2. **Настройте SPF, DKIM, DMARC** для email
3. **Используйте DNS-over-HTTPS (DoH)** или DNS-over-TLS (DoT)

---

## Полезные инструменты

| Инструмент | Назначение |
|------------|-----------|
| `dig` | Подробные DNS-запросы |
| `nslookup` | Простые DNS-запросы |
| `host` | Быстрый DNS lookup |
| [dnschecker.org](https://dnschecker.org) | Проверка пропагации DNS |
| [mxtoolbox.com](https://mxtoolbox.com) | Диагностика email/DNS |
| [intodns.com](https://intodns.com) | Анализ DNS-конфигурации |

---

## Резюме

**DNS — критически важный компонент интернет-инфраструктуры:**

- Преобразует доменные имена в IP-адреса
- Имеет иерархическую структуру (Root → TLD → Authoritative)
- Поддерживает различные типы записей для разных целей
- Активно кэшируется на всех уровнях (TTL)

**В System Design DNS используется для:**

- Базовой балансировки нагрузки (Round-Robin)
- Географической маршрутизации (GeoDNS)
- Отказоустойчивости (Failover)
- Canary deployments (Weighted routing)

**Ограничения:**

- Не подходит для real-time балансировки
- Задержка пропагации из-за кэширования
- Ограниченные health checks

**Рекомендация:** используйте DNS в сочетании с Load Balancer для полноценной балансировки нагрузки и отказоустойчивости.
