# Geth (Go Ethereum)

## Обзор

**Geth** (Go Ethereum) — это официальная реализация протокола Ethereum на языке Go. Это самый популярный и широко используемый клиент Ethereum, который позволяет:

- Запускать полную ноду Ethereum
- Майнить/валидировать блоки
- Отправлять транзакции
- Развертывать и взаимодействовать со смарт-контрактами
- Исследовать историю блокчейна

Geth поддерживается Ethereum Foundation и является эталонной реализацией протокола.

## Основные возможности

### Типы нод

| Тип | Описание | Использование |
|-----|----------|---------------|
| **Full Node** | Хранит полное состояние блокчейна | Валидация, разработка |
| **Light Node** | Загружает только заголовки блоков | Легкие клиенты, мобильные |
| **Archive Node** | Полная история всех состояний | Аналитика, блок-эксплореры |

### Ключевые функции

- **JSON-RPC API** — взаимодействие с нодой через HTTP/WebSocket
- **JavaScript Console** — интерактивная консоль для работы с Ethereum
- **Account Management** — создание и управление аккаунтами
- **Mining/Staking** — участие в консенсусе сети
- **Private Networks** — создание приватных сетей для разработки

## Установка

### macOS (Homebrew)

```bash
# Установка через Homebrew
brew tap ethereum/ethereum
brew install ethereum

# Проверка версии
geth version
```

### Ubuntu/Debian

```bash
# Добавление репозитория
sudo add-apt-repository -y ppa:ethereum/ethereum
sudo apt-get update

# Установка
sudo apt-get install ethereum

# Проверка
geth version
```

### Сборка из исходников

```bash
# Клонирование репозитория
git clone https://github.com/ethereum/go-ethereum.git
cd go-ethereum

# Сборка
make geth

# Бинарник будет в build/bin/geth
./build/bin/geth version
```

### Docker

```bash
# Запуск через Docker
docker pull ethereum/client-go:latest

docker run -it -p 8545:8545 -p 30303:30303 \
  -v /path/to/data:/root/.ethereum \
  ethereum/client-go:latest
```

## Режимы синхронизации

### Snap Sync (рекомендуется)

Современный метод синхронизации, оптимизированный для скорости:

```bash
geth --syncmode snap
```

- Загружает последний снапшот состояния
- Время синхронизации: несколько часов
- Требует ~500 GB дискового пространства

### Full Sync

Полная проверка всех транзакций с genesis блока:

```bash
geth --syncmode full
```

- Полная валидация всей истории
- Время синхронизации: несколько дней
- Требует ~800 GB дискового пространства

### Light Sync

Легкий режим для ограниченных ресурсов:

```bash
geth --syncmode light
```

- Загружает только заголовки блоков
- Минимальные требования к ресурсам
- Зависит от full nodes для данных

### Archive Node

Полная история всех состояний:

```bash
geth --syncmode full --gcmode archive
```

- Хранит все исторические состояния
- Требует >12 TB дискового пространства
- Необходим для блок-эксплореров

## Конфигурация

### Базовые параметры

```bash
geth \
  --datadir /path/to/data \      # Директория данных
  --networkid 1 \                 # ID сети (1 = mainnet)
  --syncmode snap \               # Режим синхронизации
  --http \                        # Включить HTTP-RPC
  --http.addr 0.0.0.0 \          # Адрес для RPC
  --http.port 8545 \             # Порт RPC
  --http.api eth,net,web3 \      # Доступные API
  --ws \                          # Включить WebSocket
  --ws.port 8546                  # Порт WebSocket
```

### Файл конфигурации (TOML)

```toml
# config.toml
[Eth]
NetworkId = 1
SyncMode = "snap"

[Node]
DataDir = "/var/lib/geth"
HTTPHost = "0.0.0.0"
HTTPPort = 8545
HTTPModules = ["eth", "net", "web3"]
WSHost = "0.0.0.0"
WSPort = 8546

[Node.P2P]
MaxPeers = 50
ListenAddr = ":30303"
```

Запуск с конфигурационным файлом:

```bash
geth --config config.toml
```

### Подключение к разным сетям

```bash
# Mainnet (по умолчанию)
geth

# Sepolia (тестнет)
geth --sepolia

# Holesky (тестнет)
geth --holesky

# Приватная сеть
geth --datadir ./private --networkid 12345 init genesis.json
geth --datadir ./private --networkid 12345
```

## Примеры команд

### Управление аккаунтами

```bash
# Создание нового аккаунта
geth account new --datadir /path/to/data

# Список аккаунтов
geth account list --datadir /path/to/data

# Импорт приватного ключа
geth account import --datadir /path/to/data keyfile.txt
```

### JavaScript Console

```bash
# Запуск консоли
geth attach http://localhost:8545

# Или при запуске ноды
geth console
```

```javascript
// В консоли Geth
// Проверка баланса
eth.getBalance(eth.accounts[0])

// Отправка транзакции
eth.sendTransaction({
  from: eth.accounts[0],
  to: "0x...",
  value: web3.toWei(1, "ether")
})

// Информация о блоке
eth.getBlock("latest")

// Синхронизация
eth.syncing
```

### JSON-RPC запросы

```bash
# Получить номер последнего блока
curl -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  http://localhost:8545

# Получить баланс
curl -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_getBalance","params":["0x...", "latest"],"id":1}' \
  http://localhost:8545

# Отправить raw транзакцию
curl -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_sendRawTransaction","params":["0x..."],"id":1}' \
  http://localhost:8545
```

## Systemd сервис

```ini
# /etc/systemd/system/geth.service
[Unit]
Description=Geth Ethereum Node
After=network.target

[Service]
Type=simple
User=ethereum
ExecStart=/usr/bin/geth \
  --datadir /var/lib/geth \
  --syncmode snap \
  --http \
  --http.addr 0.0.0.0 \
  --http.port 8545 \
  --http.api eth,net,web3 \
  --ws \
  --ws.port 8546 \
  --metrics \
  --metrics.port 6060

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
# Управление сервисом
sudo systemctl daemon-reload
sudo systemctl enable geth
sudo systemctl start geth
sudo systemctl status geth

# Просмотр логов
sudo journalctl -u geth -f
```

## Мониторинг

### Встроенные метрики

```bash
# Включение метрик
geth --metrics --metrics.addr 0.0.0.0 --metrics.port 6060
```

### Важные метрики

- `chain/head/header` — номер последнего заголовка
- `chain/head/block` — номер последнего блока
- `p2p/peers` — количество подключенных пиров
- `txpool/pending` — ожидающие транзакции
- `txpool/queued` — транзакции в очереди

### Prometheus + Grafana

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'geth'
    static_configs:
      - targets: ['localhost:6060']
```

## Best Practices

### Безопасность

1. **Никогда не открывайте RPC на 0.0.0.0 без защиты**
   ```bash
   # Используйте firewall или reverse proxy
   geth --http.addr 127.0.0.1
   ```

2. **Ограничьте доступные API**
   ```bash
   # Только необходимые модули
   geth --http.api eth,net,web3
   # Избегайте: personal, admin, debug
   ```

3. **Используйте JWT для Engine API**
   ```bash
   geth --authrpc.jwtsecret /path/to/jwt.hex
   ```

### Производительность

1. **SSD обязателен** — HDD неприемлем для нод Ethereum

2. **Достаточно RAM**
   - Full node: минимум 16 GB
   - Archive node: минимум 32 GB

3. **Настройка лимитов**
   ```bash
   geth --cache 4096 --maxpeers 50
   ```

4. **Регулярная очистка**
   ```bash
   # Удаление устаревших данных
   geth snapshot prune-state
   ```

### Резервное копирование

```bash
# Остановить ноду перед бэкапом
sudo systemctl stop geth

# Создать бэкап
tar -czvf geth-backup.tar.gz /var/lib/geth/geth/chaindata

# Запустить ноду
sudo systemctl start geth
```

### Обновление

```bash
# 1. Остановить ноду
sudo systemctl stop geth

# 2. Обновить Geth
sudo apt-get update && sudo apt-get upgrade ethereum

# 3. Запустить с новой версией
sudo systemctl start geth

# 4. Проверить логи
sudo journalctl -u geth -f
```

## Полезные ресурсы

- [Официальная документация](https://geth.ethereum.org/docs)
- [GitHub репозиторий](https://github.com/ethereum/go-ethereum)
- [Ethereum Stack Exchange](https://ethereum.stackexchange.com/)
- [Go Ethereum Discord](https://discord.gg/nthXNEv)
