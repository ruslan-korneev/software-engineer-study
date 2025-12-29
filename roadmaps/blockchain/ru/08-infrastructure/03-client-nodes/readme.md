# Клиентские ноды

## Обзор

**Клиентские ноды** (или клиенты блокчейна) — это программное обеспечение, которое реализует протокол блокчейн-сети и позволяет участвовать в её работе. Клиент выполняет следующие функции:

- Подключение к P2P сети блокчейна
- Синхронизация и хранение данных блокчейна
- Валидация транзакций и блоков
- Предоставление API для взаимодействия с сетью
- Трансляция транзакций в сеть

## Типы нод

### По функциональности

| Тип | Описание | Применение |
|-----|----------|------------|
| **Full Node** | Хранит полное текущее состояние блокчейна | Валидация, разработка DApps |
| **Light Node** | Хранит только заголовки блоков | Мобильные приложения, легкие кошельки |
| **Archive Node** | Хранит всю историю состояний | Блок-эксплореры, аналитика |
| **Validator Node** | Участвует в создании блоков | Стейкинг, консенсус |

### По режиму синхронизации

- **Snap Sync** — быстрая синхронизация через снапшоты состояния
- **Fast Sync** — загрузка блоков без полного исполнения
- **Full Sync** — полная валидация всей истории
- **Light Sync** — минимальная синхронизация (только заголовки)

## Архитектура Ethereum после The Merge

После перехода Ethereum на Proof of Stake (The Merge), архитектура ноды изменилась:

```
┌─────────────────────────────────────────┐
│           Ethereum Node                  │
├─────────────────┬───────────────────────┤
│  Execution      │    Consensus          │
│  Client         │    Client             │
│  (Geth, Besu,   │    (Prysm, Lighthouse,│
│   Nethermind,   │     Teku, Nimbus,     │
│   Erigon)       │     Lodestar)         │
├─────────────────┴───────────────────────┤
│         Engine API (JWT Auth)           │
└─────────────────────────────────────────┘
```

### Execution Client

Отвечает за:
- Исполнение транзакций и смарт-контрактов
- Управление состоянием (state)
- Хранение данных блокчейна
- JSON-RPC API для DApps

### Consensus Client

Отвечает за:
- Реализацию Proof of Stake
- Валидацию блоков в Beacon Chain
- Управление стейкингом
- Синхронизацию с сетью валидаторов

## Популярные клиенты Ethereum

### Execution Clients

| Клиент | Язык | Особенности |
|--------|------|-------------|
| **Geth** | Go | Эталонная реализация, самый популярный |
| **Besu** | Java | Enterprise-фичи, permissioning |
| **Nethermind** | C# | Высокая производительность |
| **Erigon** | Go | Оптимизирован для archive nodes |

### Consensus Clients

| Клиент | Язык | Особенности |
|--------|------|-------------|
| **Prysm** | Go | Самый популярный |
| **Lighthouse** | Rust | Высокая производительность |
| **Teku** | Java | Enterprise, от ConsenSys |
| **Nimbus** | Nim | Легковесный |
| **Lodestar** | TypeScript | Для JavaScript разработчиков |

## Выбор клиента

### Для разработки DApps

- Geth или Besu для локальной разработки
- Testnet для тестирования
- Alchemy/Infura для продакшена (если не нужна своя нода)

### Для валидации

- Комбинация разных клиентов для децентрализации
- Geth + Lighthouse или Besu + Prysm
- Archive node не нужен для валидации

### Для enterprise

- Besu с permissioning и privacy
- IBFT 2.0 или QBFT консенсус
- Tessera для приватных транзакций

### Для аналитики и эксплореров

- Erigon для archive nodes (меньше места)
- Geth archive для полной совместимости
- GraphQL API для сложных запросов

## Требования к оборудованию

### Full Node (Mainnet)

| Компонент | Минимум | Рекомендуется |
|-----------|---------|---------------|
| CPU | 4 cores | 8+ cores |
| RAM | 16 GB | 32 GB |
| Storage | 1 TB SSD | 2 TB NVMe SSD |
| Network | 25 Mbps | 100+ Mbps |

### Archive Node

| Компонент | Минимум | Рекомендуется |
|-----------|---------|---------------|
| CPU | 8 cores | 16+ cores |
| RAM | 32 GB | 64 GB |
| Storage | 12 TB SSD | 16+ TB NVMe |
| Network | 100 Mbps | 1 Gbps |

## API и интерфейсы

### JSON-RPC

Стандартный API для взаимодействия:

```bash
curl -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  http://localhost:8545
```

### WebSocket

Для подписок и real-time данных:

```javascript
const ws = new WebSocket('ws://localhost:8546');
ws.send(JSON.stringify({
  jsonrpc: '2.0',
  method: 'eth_subscribe',
  params: ['newHeads'],
  id: 1
}));
```

### GraphQL

Гибкие запросы к данным блокчейна:

```graphql
{
  block(number: 12345678) {
    hash
    transactions {
      hash
      from { address }
      to { address }
      value
    }
  }
}
```

## Безопасность

### Основные принципы

1. **Не открывайте RPC на публичные адреса без защиты**
2. **Используйте JWT для Engine API**
3. **Ограничивайте доступные API модули**
4. **Применяйте firewall и reverse proxy**
5. **Регулярно обновляйте клиенты**

### Рекомендуемые API модули

```bash
# Для DApps (публичный доступ)
--http.api eth,net,web3

# Полный доступ (только локально)
--http.api eth,net,web3,txpool,debug,admin
```

## Мониторинг

### Ключевые метрики

- **Высота блока** — синхронизация с сетью
- **Количество пиров** — здоровье P2P соединений
- **Размер mempool** — ожидающие транзакции
- **Использование ресурсов** — CPU, RAM, диск

### Инструменты

- Prometheus + Grafana
- Встроенные метрики клиентов
- Health check endpoints

## Содержание раздела

- [x] [Geth (Go Ethereum)](./01-geth/readme.md) — официальный клиент Ethereum на Go
- [x] [Hyperledger Besu](./02-besu/readme.md) — enterprise Ethereum клиент на Java

## Полезные ресурсы

- [Ethereum Nodes Documentation](https://ethereum.org/en/developers/docs/nodes-and-clients/)
- [Client Diversity](https://clientdiversity.org/)
- [Run a Node](https://ethereum.org/en/run-a-node/)
- [Etherscan Node Tracker](https://etherscan.io/nodetracker)
