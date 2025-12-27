# Docker Networking

## Введение в сетевую модель Docker

Docker использует **Container Network Model (CNM)** - абстракцию, которая определяет, как контейнеры подключаются к сети и взаимодействуют друг с другом.

### Ключевые концепции CNM

1. **Sandbox** - изолированное сетевое окружение контейнера (network namespace)
2. **Endpoint** - точка подключения sandbox к network (виртуальный сетевой интерфейс)
3. **Network** - группа endpoints, которые могут взаимодействовать друг с другом

```
┌─────────────────────────────────────────────────────────────┐
│                        Docker Host                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │  Container  │  │  Container  │  │  Container  │          │
│  │  ┌───────┐  │  │  ┌───────┐  │  │  ┌───────┐  │          │
│  │  │Sandbox│  │  │  │Sandbox│  │  │  │Sandbox│  │          │
│  │  │ eth0  │  │  │  │ eth0  │  │  │  │ eth0  │  │          │
│  │  └───┬───┘  │  │  └───┬───┘  │  │  └───┬───┘  │          │
│  └──────┼──────┘  └──────┼──────┘  └──────┼──────┘          │
│         │                │                │                  │
│         ▼                ▼                ▼                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   Docker Network                     │    │
│  │                   (bridge0)                          │    │
│  └─────────────────────────────────────────────────────┘    │
│                            │                                 │
│                            ▼                                 │
│                     ┌──────────┐                            │
│                     │   eth0   │ ← Host interface           │
│                     └──────────┘                            │
└─────────────────────────────────────────────────────────────┘
```

### Просмотр доступных сетей

```bash
# Список всех сетей
docker network ls

# Вывод:
# NETWORK ID     NAME      DRIVER    SCOPE
# 7fca4eb8c647   bridge    bridge    local
# 9f904ee27bf5   host      host      local
# cf03ee007fb4   none      null      local
```

По умолчанию Docker создает три сети: **bridge**, **host** и **none**.

---

## Сетевые драйверы Docker

Docker поддерживает несколько сетевых драйверов, каждый для своего use case.

### 1. Bridge (default)

**Bridge** - сетевой драйвер по умолчанию. Создает виртуальный мост на хосте, к которому подключаются контейнеры.

```bash
# Запуск контейнера в default bridge network
docker run -d --name web nginx

# Проверка сети контейнера
docker inspect web --format '{{.NetworkSettings.Networks}}'
```

**Особенности default bridge:**
- Контейнеры получают IP из подсети 172.17.0.0/16
- Связь между контейнерами по IP-адресам
- **Нет встроенного DNS** (только через --link, deprecated)
- NAT для доступа во внешнюю сеть

```bash
# Просмотр информации о bridge сети
docker network inspect bridge

# Вывод (сокращенно):
# {
#     "Name": "bridge",
#     "Driver": "bridge",
#     "IPAM": {
#         "Config": [
#             {
#                 "Subnet": "172.17.0.0/16",
#                 "Gateway": "172.17.0.1"
#             }
#         ]
#     }
# }
```

### 2. Host

Контейнер использует сетевой стек хоста напрямую, без изоляции.

```bash
# Запуск с host networking
docker run -d --network host --name web-host nginx

# Nginx слушает напрямую на порту 80 хоста
curl localhost:80
```

**Преимущества:**
- Максимальная производительность (нет NAT overhead)
- Контейнер видит все сетевые интерфейсы хоста

**Недостатки:**
- Нет сетевой изоляции
- Возможны конфликты портов
- Не работает на Docker Desktop (Mac/Windows)

```bash
# Проверка - контейнер использует IP хоста
docker run --rm --network host alpine ip addr
```

### 3. None

Полностью отключает сетевой стек контейнера.

```bash
# Контейнер без сети
docker run -d --network none --name isolated alpine sleep 3600

# Проверка - только loopback
docker exec isolated ip addr
# 1: lo: <LOOPBACK,UP,LOWER_UP> ...
#     inet 127.0.0.1/8 scope host lo
```

**Use cases:**
- Изоляция для обработки sensitive данных
- Batch jobs без сетевого доступа
- Безопасность

### 4. Overlay

Создает распределенную сеть между несколькими Docker хостами. Используется в Docker Swarm и Kubernetes.

```bash
# Создание overlay сети (требует Swarm mode)
docker network create --driver overlay my-overlay

# Запуск сервиса в overlay сети
docker service create --name web --network my-overlay nginx
```

**Особенности:**
- Работает поверх существующей сетевой инфраструктуры (VXLAN)
- Автоматическое шифрование между хостами
- Встроенный DNS и load balancing

### 5. Macvlan

Назначает контейнеру реальный MAC-адрес, делая его видимым как физическое устройство в сети.

```bash
# Создание macvlan сети
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  my-macvlan

# Запуск контейнера с фиксированным IP
docker run -d --network my-macvlan \
  --ip 192.168.1.100 \
  --name web nginx
```

**Use cases:**
- Legacy приложения, требующие прямой доступ к физической сети
- Приложения, слушающие multicast трафик
- Интеграция контейнеров в существующую VLAN инфраструктуру

**Ограничения:**
- Требует promiscuous mode на хост-интерфейсе
- Не работает на многих облачных провайдерах

### 6. IPvlan

Похож на macvlan, но все контейнеры используют один MAC-адрес хоста.

```bash
# IPvlan L2 mode
docker network create -d ipvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  my-ipvlan-l2

# IPvlan L3 mode (routing)
docker network create -d ipvlan \
  --subnet=192.168.100.0/24 \
  -o parent=eth0 \
  -o ipvlan_mode=l3 \
  my-ipvlan-l3
```

**Отличия от macvlan:**
- Обходит ограничение на количество MAC-адресов (некоторые свитчи)
- L3 mode для routing без использования bridge

### Сравнение драйверов

| Драйвер  | Изоляция | Performance | Multi-host | Use Case |
|----------|----------|-------------|------------|----------|
| bridge   | Да       | Хорошая     | Нет        | Development, single host |
| host     | Нет      | Лучшая      | N/A        | High-performance apps |
| none     | Полная   | N/A         | Нет        | Security isolation |
| overlay  | Да       | Средняя     | Да         | Swarm, distributed apps |
| macvlan  | Да       | Хорошая     | Нет        | Legacy, direct network |
| ipvlan   | Да       | Хорошая     | Нет        | MAC address limits |

---

## Работа с bridge-сетями

### Создание пользовательских сетей

User-defined bridge networks имеют преимущества перед default bridge:

```bash
# Создание custom bridge network
docker network create my-network

# С указанием подсети и gateway
docker network create \
  --driver bridge \
  --subnet 172.20.0.0/16 \
  --ip-range 172.20.240.0/20 \
  --gateway 172.20.0.1 \
  my-custom-network

# Дополнительные опции
docker network create \
  --driver bridge \
  --subnet 10.10.0.0/24 \
  --opt com.docker.network.bridge.name=br-custom \
  --opt com.docker.network.bridge.enable_icc=true \
  --opt com.docker.network.bridge.enable_ip_masquerade=true \
  custom-bridge
```

### DNS resolution между контейнерами

В user-defined сетях работает **встроенный DNS** - контейнеры могут обращаться друг к другу по имени.

```bash
# Создаем сеть
docker network create app-network

# Запускаем backend
docker run -d --name backend --network app-network nginx

# Запускаем frontend - может обращаться к backend по имени
docker run -d --name frontend --network app-network alpine sleep 3600

# Проверяем DNS resolution
docker exec frontend ping backend -c 3
# PING backend (172.20.0.2): 56 data bytes
# 64 bytes from 172.20.0.2: seq=0 ttl=64 time=0.123 ms
```

**Важно:** В default bridge DNS по имени **не работает**!

```bash
# Default bridge - DNS не работает
docker run -d --name test1 alpine sleep 3600
docker run -d --name test2 alpine sleep 3600

docker exec test2 ping test1
# ping: bad address 'test1'  ← ОШИБКА

# Работает только по IP
docker exec test2 ping 172.17.0.2
```

### Network aliases

Контейнеру можно назначить дополнительные DNS-имена:

```bash
# Запуск с alias
docker run -d \
  --name postgres-main \
  --network app-network \
  --network-alias db \
  --network-alias database \
  postgres:15

# Теперь контейнер доступен по именам:
# - postgres-main
# - db
# - database
```

### Подключение контейнера к нескольким сетям

```bash
# Создаем две сети
docker network create frontend-net
docker network create backend-net

# Запускаем контейнер в одной сети
docker run -d --name api --network frontend-net nginx

# Подключаем ко второй сети
docker network connect backend-net api

# Проверяем
docker inspect api --format '{{json .NetworkSettings.Networks}}' | jq
# {
#   "backend-net": { ... },
#   "frontend-net": { ... }
# }

# Отключение от сети
docker network disconnect frontend-net api
```

### Изоляция контейнеров

User-defined networks обеспечивают изоляцию:

```bash
# Сеть A
docker network create network-a
docker run -d --name app-a --network network-a alpine sleep 3600

# Сеть B
docker network create network-b
docker run -d --name app-b --network network-b alpine sleep 3600

# app-a не видит app-b
docker exec app-a ping app-b -c 1
# ping: bad address 'app-b'  ← Изолированы
```

**Архитектурный паттерн - изоляция микросервисов:**

```
┌─────────────────────────────────────────────────────────┐
│                     frontend-net                         │
│  ┌─────────┐         ┌─────────┐                        │
│  │  nginx  │ ──────▶ │   api   │                        │
│  └─────────┘         └────┬────┘                        │
└────────────────────────────┼────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────┐
│                     backend-net                          │
│                       ┌────┴────┐                       │
│                       │   api   │ (connected to both)   │
│                       └────┬────┘                       │
│                            │                            │
│           ┌────────────────┼────────────────┐           │
│           ▼                ▼                ▼           │
│     ┌──────────┐    ┌──────────┐    ┌──────────┐       │
│     │ postgres │    │  redis   │    │ rabbitmq │       │
│     └──────────┘    └──────────┘    └──────────┘       │
└─────────────────────────────────────────────────────────┘
```

---

## Публикация портов

### Флаги -p и -P

**-p (publish)** - явная публикация порта:

```bash
# Базовый синтаксис: -p [host_ip:]host_port:container_port[/protocol]

# Публикация на все интерфейсы
docker run -d -p 8080:80 nginx
# Доступ: curl localhost:8080

# Публикация нескольких портов
docker run -d -p 8080:80 -p 8443:443 nginx

# Указание протокола (по умолчанию tcp)
docker run -d -p 53:53/udp -p 53:53/tcp bind9

# Публикация диапазона портов
docker run -d -p 8000-8010:8000-8010 myapp
```

**-P (publish-all)** - публикация всех EXPOSE портов на случайные порты хоста:

```bash
# Dockerfile с EXPOSE
# EXPOSE 80 443

docker run -d -P nginx

# Проверка назначенных портов
docker port <container_id>
# 80/tcp -> 0.0.0.0:32768
# 443/tcp -> 0.0.0.0:32769
```

### Binding к конкретным интерфейсам

```bash
# Только localhost (не доступен извне)
docker run -d -p 127.0.0.1:8080:80 nginx

# Конкретный IP хоста
docker run -d -p 192.168.1.100:8080:80 nginx

# Только IPv6
docker run -d -p [::1]:8080:80 nginx

# Случайный порт на конкретном IP
docker run -d -p 127.0.0.1::80 nginx
# Проверка: docker port <container_id>
```

### Проверка опубликованных портов

```bash
# Все порты контейнера
docker port my-container

# Конкретный порт
docker port my-container 80/tcp

# Через inspect
docker inspect --format='{{range $p, $conf := .NetworkSettings.Ports}}{{$p}} -> {{(index $conf 0).HostPort}}{{"\n"}}{{end}}' my-container
```

---

## Сети в Docker Compose

### Автоматическое создание сетей

Docker Compose автоматически создает сеть `<project_name>_default`:

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    image: nginx
    ports:
      - "8080:80"

  api:
    image: node:18
    # web и api в одной сети, видят друг друга по имени
```

```bash
docker-compose up -d
# Creating network "myproject_default" with the default driver
# Creating myproject_web_1
# Creating myproject_api_1
```

### Определение custom сетей

```yaml
version: '3.8'

services:
  nginx:
    image: nginx
    networks:
      - frontend

  api:
    image: node:18
    networks:
      - frontend
      - backend

  postgres:
    image: postgres:15
    networks:
      - backend
    environment:
      POSTGRES_PASSWORD: secret

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: br-backend
```

### Подключение к внешним сетям

```yaml
version: '3.8'

services:
  app:
    image: myapp
    networks:
      - existing-network
      - internal

networks:
  existing-network:
    external: true
    name: my-existing-network  # имя существующей сети

  internal:
    driver: bridge
```

```bash
# Сначала создаем внешнюю сеть
docker network create my-existing-network

# Затем запускаем compose
docker-compose up -d
```

### Network aliases в Compose

```yaml
version: '3.8'

services:
  postgres-primary:
    image: postgres:15
    networks:
      backend:
        aliases:
          - db
          - database
          - postgres

  app:
    image: myapp
    networks:
      - backend
    environment:
      # Можно использовать любой alias
      DATABASE_URL: postgres://user:pass@db:5432/mydb

networks:
  backend:
```

### Расширенная конфигурация сетей

```yaml
version: '3.8'

services:
  web:
    image: nginx
    networks:
      frontend:
        ipv4_address: 172.28.0.10
        aliases:
          - web.local

  api:
    image: node:18
    networks:
      frontend:
        ipv4_address: 172.28.0.11
      backend:
        ipv4_address: 172.29.0.11

networks:
  frontend:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.28.0.0/16
          gateway: 172.28.0.1

  backend:
    driver: bridge
    internal: true  # Нет доступа во внешнюю сеть
    ipam:
      config:
        - subnet: 172.29.0.0/16
```

### Полный пример: Микросервисная архитектура

```yaml
version: '3.8'

services:
  # Frontend tier
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    networks:
      - frontend
    depends_on:
      - api

  # Application tier
  api:
    build: ./api
    networks:
      - frontend
      - backend
    environment:
      - DATABASE_URL=postgres://app:secret@db:5432/appdb
      - REDIS_URL=redis://cache:6379
    depends_on:
      - db
      - cache

  worker:
    build: ./worker
    networks:
      - backend
    environment:
      - DATABASE_URL=postgres://app:secret@db:5432/appdb
      - RABBITMQ_URL=amqp://mq:5672
    depends_on:
      - db
      - mq

  # Data tier
  db:
    image: postgres:15
    networks:
      - backend
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=appdb
      - POSTGRES_USER=app
      - POSTGRES_PASSWORD=secret

  cache:
    image: redis:7-alpine
    networks:
      - backend

  mq:
    image: rabbitmq:3-management
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # Изоляция от внешней сети

volumes:
  postgres_data:
```

---

## Overlay сети для Docker Swarm

Overlay networks позволяют контейнерам на разных хостах общаться так, будто они в одной локальной сети.

### Инициализация Swarm

```bash
# На manager node
docker swarm init --advertise-addr <MANAGER-IP>

# На worker nodes
docker swarm join --token <TOKEN> <MANAGER-IP>:2377
```

### Создание overlay сети

```bash
# Создание overlay сети
docker network create --driver overlay --attachable my-overlay

# --attachable позволяет подключать standalone контейнеры
```

### Развертывание сервиса

```bash
# Создание сервиса в overlay сети
docker service create \
  --name web \
  --network my-overlay \
  --replicas 3 \
  --publish published=80,target=80 \
  nginx

# Проверка
docker service ps web
# ID         NAME    IMAGE   NODE      DESIRED STATE   CURRENT STATE
# abc123     web.1   nginx   node-1    Running         Running
# def456     web.2   nginx   node-2    Running         Running
# ghi789     web.3   nginx   node-3    Running         Running
```

### Шифрование overlay трафика

```bash
# Создание зашифрованной сети
docker network create \
  --driver overlay \
  --opt encrypted \
  secure-overlay

# Все данные между хостами шифруются IPsec
```

### Ingress network

Docker Swarm автоматически создает **ingress** сеть для routing mesh:

```bash
docker network ls
# NETWORK ID     NAME      DRIVER    SCOPE
# abc123         ingress   overlay   swarm
```

Routing mesh обеспечивает доступ к сервису через любой node кластера.

---

## Container Networking Internals

### Linux Network Namespaces

Каждый контейнер работает в своем network namespace:

```bash
# Просмотр namespace контейнера
docker inspect --format '{{.NetworkSettings.SandboxKey}}' my-container
# /var/run/docker/netns/abc123

# Просмотр всех namespaces
sudo ls -la /var/run/docker/netns/

# Выполнение команды в namespace контейнера
sudo nsenter --net=/var/run/docker/netns/abc123 ip addr
```

### Virtual Ethernet (veth) pairs

Docker создает пару виртуальных интерфейсов для каждого контейнера:

```bash
# На хосте - один конец veth пары
ip link show type veth
# 5: veth123@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> ...

# В контейнере - другой конец (eth0)
docker exec my-container ip link
# 4: eth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
```

**Архитектура:**

```
┌────────────────────────────────────────────────────────────┐
│                        Docker Host                          │
│                                                              │
│   ┌──────────────────┐      ┌──────────────────┐            │
│   │   Container A    │      │   Container B    │            │
│   │  ┌────────────┐  │      │  ┌────────────┐  │            │
│   │  │   eth0     │  │      │  │   eth0     │  │            │
│   │  │172.17.0.2  │  │      │  │172.17.0.3  │  │            │
│   │  └─────┬──────┘  │      │  └─────┬──────┘  │            │
│   └────────┼─────────┘      └────────┼─────────┘            │
│            │ veth pair              │ veth pair             │
│            ▼                        ▼                       │
│   ┌────────────────────────────────────────────────┐       │
│   │              docker0 (bridge)                   │       │
│   │              172.17.0.1                         │       │
│   └────────────────────┬───────────────────────────┘       │
│                        │                                    │
│                        │ iptables NAT                       │
│                        ▼                                    │
│                  ┌──────────┐                               │
│                  │   eth0   │  Host interface               │
│                  │192.168.1.100                             │
│                  └──────────┘                               │
└────────────────────────────────────────────────────────────┘
```

### iptables правила

Docker использует iptables для NAT и фильтрации:

```bash
# Просмотр правил Docker
sudo iptables -t nat -L -n -v

# Цепочка DOCKER для port forwarding
sudo iptables -t nat -L DOCKER -n -v
# Chain DOCKER (2 references)
# target  prot  dpt   destination
# DNAT    tcp   8080  172.17.0.2:80

# Цепочка DOCKER-ISOLATION для изоляции сетей
sudo iptables -L DOCKER-ISOLATION-STAGE-1 -n -v
```

**Основные цепочки:**
- `DOCKER` - DNAT для port forwarding
- `DOCKER-ISOLATION-STAGE-1/2` - изоляция между сетями
- `DOCKER-USER` - пользовательские правила (не перезаписываются Docker)

### Добавление правил в DOCKER-USER

```bash
# Блокировка доступа к контейнерам из определенной подсети
sudo iptables -I DOCKER-USER -s 10.0.0.0/8 -j DROP

# Разрешение только определенных портов
sudo iptables -I DOCKER-USER -p tcp --dport 80 -j ACCEPT
sudo iptables -I DOCKER-USER -p tcp --dport 443 -j ACCEPT
sudo iptables -A DOCKER-USER -j DROP
```

### Bridge interface

```bash
# Информация о docker0 bridge
ip addr show docker0
# docker0: <BROADCAST,MULTICAST,UP,LOWER_UP>
#     inet 172.17.0.1/16 brd 172.17.255.255

# Подключенные интерфейсы
bridge link show docker0

# Или через brctl
sudo brctl show docker0
# bridge name  bridge id          STP enabled  interfaces
# docker0      8000.024216a12345  no           veth123
#                                              veth456
```

---

## Troubleshooting сетевых проблем

### docker network inspect

```bash
# Полная информация о сети
docker network inspect my-network

# Конкретные поля
docker network inspect my-network --format '{{.IPAM.Config}}'

# Список контейнеров в сети
docker network inspect my-network --format '{{json .Containers}}' | jq

# IP-адреса контейнеров
docker network inspect bridge --format '{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{"\n"}}{{end}}'
```

### Проверка connectivity

```bash
# 1. Проверка сетевых настроек контейнера
docker exec my-container ip addr
docker exec my-container ip route

# 2. DNS resolution
docker exec my-container nslookup other-container
docker exec my-container cat /etc/resolv.conf

# 3. Ping между контейнерами
docker exec container-a ping container-b -c 3

# 4. Проверка портов
docker exec my-container netstat -tlnp
docker exec my-container ss -tlnp

# 5. Curl для HTTP
docker exec my-container curl -v http://other-container:8080

# 6. Tcpdump для анализа трафика
docker exec my-container tcpdump -i eth0 -n port 80
```

### Отладка с использованием nicolaka/netshoot

```bash
# Запуск отладочного контейнера в сети
docker run -it --rm --network my-network nicolaka/netshoot

# В контейнере доступны: curl, ping, dig, nslookup, tcpdump, netstat, iptables и др.

# Или присоединение к network namespace существующего контейнера
docker run -it --rm --network container:my-container nicolaka/netshoot
```

### Типичные проблемы и решения

#### 1. Контейнеры не видят друг друга по имени

**Проблема:** DNS не работает в default bridge network.

```bash
# Плохо
docker run -d --name db postgres
docker run -d --name app myapp
# app не может резолвить 'db'

# Хорошо - используйте custom network
docker network create mynet
docker run -d --name db --network mynet postgres
docker run -d --name app --network mynet myapp
```

#### 2. Порт недоступен снаружи

```bash
# Проверки:
# 1. Контейнер запущен?
docker ps

# 2. Порт опубликован?
docker port my-container

# 3. Приложение слушает на 0.0.0.0, а не 127.0.0.1?
docker exec my-container netstat -tlnp

# 4. Firewall хоста?
sudo iptables -L -n | grep <port>
sudo ufw status
```

#### 3. Контейнер не может достучаться до внешней сети

```bash
# 1. Проверка DNS
docker exec my-container cat /etc/resolv.conf
docker exec my-container ping 8.8.8.8

# 2. Проверка маршрутизации
docker exec my-container ip route

# 3. Проверка IP forwarding на хосте
cat /proc/sys/net/ipv4/ip_forward  # Должно быть 1

# 4. Проверка iptables NAT
sudo iptables -t nat -L POSTROUTING -n -v
```

#### 4. Конфликт IP-адресов

```bash
# Проверка подсетей
docker network inspect bridge --format '{{(index .IPAM.Config 0).Subnet}}'

# Создание сети с другой подсетью
docker network create --subnet 10.10.0.0/16 my-network
```

#### 5. "bind: address already in use"

```bash
# Найти процесс, занимающий порт
sudo lsof -i :8080
sudo netstat -tlnp | grep 8080

# Остановить или использовать другой порт
docker run -p 8081:80 nginx
```

### Полезные команды для диагностики

```bash
# Все сети с деталями
docker network ls --no-trunc

# Контейнеры без сети
docker ps --filter "network=none"

# Очистка неиспользуемых сетей
docker network prune

# Логи Docker daemon
sudo journalctl -u docker.service -f

# Проверка iptables правил Docker
sudo iptables-save | grep -i docker
```

---

## Best Practices для Docker Networking

### 1. Используйте user-defined networks

```bash
# Плохо
docker run -d --name app1 myapp
docker run -d --name app2 myapp

# Хорошо
docker network create app-network
docker run -d --name app1 --network app-network myapp
docker run -d --name app2 --network app-network myapp
```

**Преимущества:**
- Встроенный DNS
- Лучшая изоляция
- Возможность подключения к нескольким сетям

### 2. Изолируйте сервисы по сетям

```yaml
# docker-compose.yml
services:
  nginx:
    networks:
      - frontend  # Только public-facing

  api:
    networks:
      - frontend  # Получает запросы от nginx
      - backend   # Общается с БД

  db:
    networks:
      - backend   # Изолирован от внешнего мира

networks:
  frontend:
  backend:
    internal: true  # Без доступа во внешнюю сеть
```

### 3. Не публикуйте порты БД

```yaml
# Плохо - БД доступна извне
services:
  postgres:
    ports:
      - "5432:5432"

# Хорошо - БД только для внутренних сервисов
services:
  postgres:
    networks:
      - backend
    # ports не нужны - другие контейнеры достучатся по internal DNS
```

### 4. Используйте network aliases для абстракции

```yaml
services:
  postgres-primary:
    networks:
      backend:
        aliases:
          - db  # Абстрактное имя

  app:
    environment:
      DATABASE_HOST: db  # Не привязан к конкретному имени контейнера
```

### 5. Ограничивайте binding портов

```bash
# Плохо - доступно со всех интерфейсов
docker run -p 8080:80 myapp

# Хорошо - только localhost (для development)
docker run -p 127.0.0.1:8080:80 myapp

# Production - через reverse proxy
# Nginx/Traefik с TLS -> internal containers
```

### 6. Настройте DOCKER-USER для firewall

```bash
# Правила в DOCKER-USER не перезаписываются Docker
sudo iptables -I DOCKER-USER -i eth0 -p tcp --dport 3306 -j DROP
sudo iptables -I DOCKER-USER -i eth0 -s 10.0.0.0/8 -p tcp --dport 3306 -j ACCEPT
```

### 7. Используйте internal networks для изоляции

```yaml
networks:
  internal:
    internal: true  # Нет NAT gateway - полная изоляция
```

### 8. Мониторинг сетевых метрик

```bash
# Статистика по контейнерам
docker stats --format "table {{.Name}}\t{{.NetIO}}"

# Детальная информация
docker exec my-container cat /proc/net/dev
```

### 9. Документируйте сетевую архитектуру

```yaml
# docker-compose.yml
networks:
  # Public-facing tier - receives external traffic
  frontend:
    driver: bridge

  # Application tier - API servers
  middleware:
    driver: bridge

  # Data tier - databases, caches (isolated)
  backend:
    driver: bridge
    internal: true
```

### 10. Регулярная очистка

```bash
# Удаление неиспользуемых сетей
docker network prune

# Проверка "висящих" сетей
docker network ls --filter "dangling=true"
```

---

## Заключение

Docker networking - мощная система, которая позволяет:

- **Изолировать** сервисы друг от друга
- **Объединять** контейнеры в виртуальные сети
- **Масштабировать** приложения на несколько хостов
- **Контролировать** сетевой трафик

Ключевые моменты:

1. **Всегда используйте user-defined networks** вместо default bridge
2. **Изолируйте backend сервисы** с помощью internal networks
3. **Не публикуйте порты БД** напрямую - используйте internal networking
4. **Используйте overlay networks** для multi-host deployments
5. **Понимайте internals** (iptables, veth, namespaces) для troubleshooting

Правильная настройка сети - один из ключевых аспектов безопасного и эффективного использования Docker в production.
