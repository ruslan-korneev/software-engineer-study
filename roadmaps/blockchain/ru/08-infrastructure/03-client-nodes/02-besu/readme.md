# Hyperledger Besu

## Обзор

**Hyperledger Besu** — это Ethereum-клиент корпоративного уровня, написанный на Java. Разработанный под эгидой Hyperledger (Linux Foundation), Besu совместим с Ethereum mainnet и специально спроектирован для:

- Корпоративных блокчейн-решений
- Публичных и приватных сетей
- Консорциумных блокчейнов
- Решений с высокими требованиями к compliance

Besu полностью совместим с Ethereum mainnet и поддерживает все стандартные EVM операции.

## Основные возможности

### Преимущества для enterprise

| Возможность | Описание |
|-------------|----------|
| **Permissioning** | Контроль доступа к сети на уровне нод и аккаунтов |
| **Privacy** | Приватные транзакции между участниками |
| **Multiple Consensus** | Поддержка IBFT 2.0, QBFT, Clique, PoW, PoS |
| **Monitoring** | Встроенная поддержка Prometheus и Grafana |
| **GraphQL** | GraphQL API наряду с JSON-RPC |
| **EVM Compatible** | Полная совместимость с Ethereum |

### Поддерживаемые сети

- Ethereum Mainnet
- Sepolia, Holesky (тестнеты)
- Приватные сети
- Консорциумные сети

## Установка

### Требования

- Java 17+ (LTS версия рекомендуется)
- 8 GB RAM минимум (16 GB рекомендуется)
- SSD с достаточным пространством

### Скачивание бинарников

```bash
# Скачать последнюю версию
wget https://github.com/hyperledger/besu/releases/download/24.1.0/besu-24.1.0.tar.gz

# Распаковать
tar -xzf besu-24.1.0.tar.gz
cd besu-24.1.0

# Проверить установку
./bin/besu --version
```

### Установка через Homebrew (macOS)

```bash
brew tap hyperledger/besu
brew install besu

besu --version
```

### Docker

```bash
# Запуск через Docker
docker pull hyperledger/besu:latest

docker run -it -p 8545:8545 -p 8546:8546 -p 30303:30303 \
  -v /path/to/data:/opt/besu/data \
  hyperledger/besu:latest \
  --data-path=/opt/besu/data \
  --rpc-http-enabled \
  --rpc-ws-enabled
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'
services:
  besu:
    image: hyperledger/besu:latest
    container_name: besu-node
    ports:
      - "8545:8545"
      - "8546:8546"
      - "30303:30303"
    volumes:
      - besu-data:/opt/besu/data
    command:
      - --data-path=/opt/besu/data
      - --rpc-http-enabled
      - --rpc-http-api=ETH,NET,WEB3
      - --rpc-ws-enabled
      - --host-allowlist=*

volumes:
  besu-data:
```

## Режимы синхронизации

### Snap Sync (рекомендуется)

```bash
besu --sync-mode=SNAP
```

- Быстрая начальная синхронизация
- Загрузка снапшота состояния
- Время: несколько часов

### Fast Sync

```bash
besu --sync-mode=FAST
```

- Загрузка блоков без исполнения транзакций
- Получение состояния от пиров
- Время: несколько часов

### Full Sync

```bash
besu --sync-mode=FULL
```

- Полная валидация всех транзакций
- Время: несколько дней

### Checkpoint Sync

```bash
besu --sync-mode=CHECKPOINT \
  --checkpoint-post-merge-enabled
```

- Синхронизация от checkpoint
- Самый быстрый способ для начала

## Конфигурация

### Базовый запуск

```bash
besu \
  --data-path=/var/lib/besu \
  --network=mainnet \
  --sync-mode=SNAP \
  --rpc-http-enabled \
  --rpc-http-host=0.0.0.0 \
  --rpc-http-port=8545 \
  --rpc-http-api=ETH,NET,WEB3 \
  --rpc-ws-enabled \
  --rpc-ws-host=0.0.0.0 \
  --rpc-ws-port=8546
```

### Файл конфигурации (TOML)

```toml
# config.toml
data-path="/var/lib/besu"
network="mainnet"
sync-mode="SNAP"

# RPC настройки
rpc-http-enabled=true
rpc-http-host="0.0.0.0"
rpc-http-port=8545
rpc-http-api=["ETH", "NET", "WEB3", "TXPOOL"]
rpc-http-cors-origins=["*"]

# WebSocket
rpc-ws-enabled=true
rpc-ws-host="0.0.0.0"
rpc-ws-port=8546
rpc-ws-api=["ETH", "NET", "WEB3"]

# GraphQL
graphql-http-enabled=true
graphql-http-host="0.0.0.0"
graphql-http-port=8547

# Метрики
metrics-enabled=true
metrics-host="0.0.0.0"
metrics-port=9545

# P2P
p2p-host="0.0.0.0"
p2p-port=30303
max-peers=25

# Логирование
logging="INFO"
```

Запуск с конфигурационным файлом:

```bash
besu --config-file=config.toml
```

### Подключение к сетям

```bash
# Mainnet
besu --network=mainnet

# Sepolia тестнет
besu --network=sepolia

# Holesky тестнет
besu --network=holesky

# Приватная сеть
besu --genesis-file=genesis.json --network-id=12345
```

## Приватные сети

### Genesis файл для IBFT 2.0

```json
{
  "config": {
    "chainId": 1337,
    "berlinBlock": 0,
    "ibft2": {
      "blockperiodseconds": 2,
      "epochlength": 30000,
      "requesttimeoutseconds": 4
    }
  },
  "nonce": "0x0",
  "timestamp": "0x58ee40ba",
  "extraData": "0x...",
  "gasLimit": "0x1fffffffffffff",
  "difficulty": "0x1",
  "mixHash": "0x...",
  "coinbase": "0x0000000000000000000000000000000000000000",
  "alloc": {
    "fe3b557e8fb62b89f4916b721be55ceb828dbd73": {
      "balance": "0xad78ebc5ac6200000"
    }
  }
}
```

### Genesis файл для QBFT

```json
{
  "config": {
    "chainId": 1337,
    "berlinBlock": 0,
    "qbft": {
      "blockperiodseconds": 2,
      "epochlength": 30000,
      "requesttimeoutseconds": 4
    }
  },
  "nonce": "0x0",
  "timestamp": "0x58ee40ba",
  "extraData": "0x...",
  "gasLimit": "0x1fffffffffffff",
  "difficulty": "0x1",
  "alloc": {}
}
```

### Запуск приватной сети

```bash
# Инициализация данных для валидатора
besu --data-path=node1 \
  --genesis-file=genesis.json \
  --rpc-http-enabled \
  --rpc-http-api=ETH,NET,IBFT \
  --host-allowlist="*" \
  --rpc-http-cors-origins="all"
```

## Permissioning (Контроль доступа)

### Локальное разрешение нод

```toml
# permissions_config.toml
nodes-allowlist=["enode://...@192.168.1.1:30303","enode://...@192.168.1.2:30303"]
```

```bash
besu --permissions-nodes-config-file=permissions_config.toml \
  --permissions-nodes-config-file-enabled
```

### Локальное разрешение аккаунтов

```toml
# permissions_config.toml
accounts-allowlist=["0xfe3b557e8fb62b89f4916b721be55ceb828dbd73"]
```

```bash
besu --permissions-accounts-config-file=permissions_config.toml \
  --permissions-accounts-config-file-enabled
```

### Разрешения на основе смарт-контрактов

```bash
besu --permissions-nodes-contract-enabled \
  --permissions-nodes-contract-address=0x... \
  --permissions-accounts-contract-enabled \
  --permissions-accounts-contract-address=0x...
```

## Privacy (Приватность)

Besu поддерживает приватные транзакции через протокол Tessera.

### Архитектура

```
+--------+     +----------+     +---------+
|  Besu  | <-> | Tessera  | <-> | Другие  |
|  Node  |     | (Privacy |     | Tessera |
|        |     |  Manager)|     |  Nodes  |
+--------+     +----------+     +---------+
```

### Конфигурация с Tessera

```bash
besu \
  --privacy-enabled \
  --privacy-url=http://127.0.0.1:9101 \
  --privacy-public-key-file=tessera/key.pub
```

### Приватная транзакция

```javascript
const privateTransaction = {
  from: "0x...",
  to: "0x...",
  data: "0x...",
  privateFrom: "BULeR8JyUWhiuuCMU/HLA0Q5pzkYT+cHII3ZKBey3Bo=",
  privateFor: ["QfeDAys9MPDs2XHExtc84jKGHxZg/aj52DTh0vtA3Xc="],
  restriction: "restricted"
};
```

## API

### JSON-RPC

```bash
# Включить HTTP RPC
besu --rpc-http-enabled --rpc-http-api=ETH,NET,WEB3,TXPOOL,DEBUG

# Пример запроса
curl -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  http://localhost:8545
```

### GraphQL

```bash
# Включить GraphQL
besu --graphql-http-enabled --graphql-http-port=8547
```

```graphql
# Пример запроса
{
  block(number: 1) {
    hash
    transactions {
      hash
      from {
        address
      }
      to {
        address
      }
      value
    }
  }
}
```

### WebSocket

```bash
besu --rpc-ws-enabled --rpc-ws-api=ETH,NET,WEB3

# Подписка на новые блоки
wscat -c ws://localhost:8546
> {"jsonrpc":"2.0","method":"eth_subscribe","params":["newHeads"],"id":1}
```

## Systemd сервис

```ini
# /etc/systemd/system/besu.service
[Unit]
Description=Hyperledger Besu Ethereum Client
After=network.target

[Service]
Type=simple
User=besu
Group=besu
ExecStart=/opt/besu/bin/besu \
  --config-file=/etc/besu/config.toml

Restart=on-failure
RestartSec=5
LimitNOFILE=65535

Environment="JAVA_OPTS=-Xmx4g"

[Install]
WantedBy=multi-user.target
```

```bash
# Создание пользователя
sudo useradd -r -s /bin/false besu

# Создание директорий
sudo mkdir -p /var/lib/besu /etc/besu
sudo chown besu:besu /var/lib/besu

# Управление сервисом
sudo systemctl daemon-reload
sudo systemctl enable besu
sudo systemctl start besu
sudo systemctl status besu

# Логи
sudo journalctl -u besu -f
```

## Мониторинг

### Prometheus метрики

```bash
besu --metrics-enabled --metrics-host=0.0.0.0 --metrics-port=9545
```

### Важные метрики

| Метрика | Описание |
|---------|----------|
| `besu_blockchain_height` | Высота блокчейна |
| `besu_peers_connected` | Количество пиров |
| `besu_synchronizer_*` | Статус синхронизации |
| `besu_transaction_pool_*` | Транзакции в пуле |
| `jvm_memory_*` | Использование памяти JVM |

### Prometheus конфигурация

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'besu'
    static_configs:
      - targets: ['localhost:9545']
    metrics_path: /metrics
```

### Health checks

```bash
# Liveness check
curl http://localhost:8545/liveness

# Readiness check
curl http://localhost:8545/readiness
```

## Best Practices

### Безопасность

1. **Ограничьте доступ к API**
   ```bash
   besu --host-allowlist="localhost,127.0.0.1" \
     --rpc-http-cors-origins="https://yourdomain.com"
   ```

2. **Используйте TLS для RPC**
   ```bash
   besu --rpc-http-tls-enabled \
     --rpc-http-tls-keystore-file=keystore.p12 \
     --rpc-http-tls-keystore-password-file=password.txt
   ```

3. **Аутентификация для Engine API**
   ```bash
   besu --engine-jwt-secret=/path/to/jwt.hex
   ```

### Производительность

1. **Настройка JVM**
   ```bash
   export JAVA_OPTS="-Xmx8g -XX:+UseG1GC"
   ```

2. **Оптимизация хранилища**
   ```bash
   besu --data-storage-format=BONSAI  # Компактное хранение
   ```

3. **Кэширование**
   ```bash
   besu --Xplugin-rocksdb-cache-capacity=2048
   ```

### Высокая доступность

1. **Несколько RPC endpoints**
2. **Load balancer перед нодами**
3. **Регулярный мониторинг и алертинг**

### Обновление

```bash
# 1. Проверить release notes
# 2. Сделать бэкап данных
# 3. Остановить ноду
sudo systemctl stop besu

# 4. Обновить бинарники
wget https://github.com/hyperledger/besu/releases/download/NEW_VERSION/besu-NEW_VERSION.tar.gz
tar -xzf besu-NEW_VERSION.tar.gz
sudo cp -r besu-NEW_VERSION/* /opt/besu/

# 5. Запустить ноду
sudo systemctl start besu

# 6. Проверить логи
sudo journalctl -u besu -f
```

## Сравнение с Geth

| Аспект | Besu | Geth |
|--------|------|------|
| Язык | Java | Go |
| Enterprise features | Да | Ограниченно |
| Permissioning | Встроенный | Через плагины |
| Privacy | Tessera | Нет |
| GraphQL | Да | Да |
| Consensus | IBFT, QBFT, PoS | PoS |
| Поддержка | Hyperledger | Ethereum Foundation |

## Полезные ресурсы

- [Официальная документация](https://besu.hyperledger.org/)
- [GitHub репозиторий](https://github.com/hyperledger/besu)
- [Hyperledger Discord](https://discord.gg/hyperledger)
- [Wiki](https://wiki.hyperledger.org/display/BESU)
