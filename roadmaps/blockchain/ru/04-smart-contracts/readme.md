# Смарт-контракты

## Обзор

Смарт-контракты — это самоисполняющиеся программы, развёрнутые на блокчейне. Они автоматически выполняют заранее определённые условия без посредников, обеспечивая прозрачность и неизменяемость бизнес-логики.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Жизненный цикл смарт-контракта                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Разработка      Тестирование      Деплой        Эксплуатация  │
│   ──────────      ────────────      ──────        ────────────  │
│   │ Solidity │ → │   Unit    │ → │ Testnet │ → │ Мониторинг │  │
│   │ Vyper    │   │ Integrat. │   │ Mainnet │   │ Обновления │  │
│   │ Rust     │   │ Coverage  │   │ Verify  │   │ Алерты     │  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Статус контента

> `[ ]` - контент не заполнен, `[x]` - контент заполнен (>10 строк)

### Языки программирования

| Статус | Тема | Описание |
|--------|------|----------|
| [x] | [Solidity](./01-solidity/readme.md) | Основной язык для EVM |
| [x] | [Основы Solidity](./01-solidity/01-basics/readme.md) | Типы данных, функции, модификаторы |
| [x] | [Продвинутый Solidity](./01-solidity/02-advanced/readme.md) | Наследование, события, assembly |
| [x] | [Паттерны проектирования](./01-solidity/03-patterns/readme.md) | Factory, Proxy, Access Control |
| [x] | [Vyper](./02-vyper/readme.md) | Безопасная альтернатива Solidity |
| [x] | [Rust для блокчейна](./03-rust-for-blockchain/readme.md) | Solana, Substrate, Near |

### Стандарты токенов

| Статус | Тема | Описание |
|--------|------|----------|
| [x] | [ERC токены (обзор)](./04-erc-tokens/readme.md) | Обзор стандартов Ethereum |
| [x] | [ERC-20](./04-erc-tokens/01-erc20/readme.md) | Взаимозаменяемые токены |
| [x] | [ERC-721](./04-erc-tokens/02-erc721/readme.md) | NFT стандарт |
| [x] | [ERC-1155](./04-erc-tokens/03-erc1155/readme.md) | Мульти-токен стандарт |

### Тестирование

| Статус | Тема | Описание |
|--------|------|----------|
| [x] | [Тестирование (обзор)](./05-testing/readme.md) | Подходы к тестированию |
| [x] | [Unit-тестирование](./05-testing/01-unit-testing/readme.md) | Hardhat, Foundry |
| [x] | [Интеграционное тестирование](./05-testing/02-integration-testing/readme.md) | Fork-тесты, DEX |
| [x] | [Покрытие кода](./05-testing/03-code-coverage/readme.md) | Coverage инструменты |

### Деплой и операции

| Статус | Тема | Описание |
|--------|------|----------|
| [x] | [Деплой](./06-deployment/readme.md) | Тестнеты, мейннет, верификация |
| [x] | [Мониторинг](./07-monitoring/readme.md) | The Graph, Tenderly, алерты |
| [x] | [Обновления контрактов](./08-upgrades/readme.md) | Proxy паттерны, governance |

## Ключевые концепции

### Особенности смарт-контрактов

1. **Неизменяемость** — код нельзя изменить после деплоя
2. **Детерминизм** — одинаковый ввод всегда даёт одинаковый результат
3. **Публичность** — весь код и данные видны всем
4. **Атомарность** — транзакции либо выполняются полностью, либо откатываются

### Стоимость операций (Gas)

```solidity
// Операции и их примерная стоимость в gas
// SSTORE (новое значение)  ~20,000 gas
// SSTORE (изменение)       ~5,000 gas
// SLOAD                    ~2,100 gas
// Вызов функции            ~21,000 gas (base)
// Трансфер ETH             ~21,000 gas
```

## Рекомендуемый порядок изучения

```
1. Solidity Basics ──────────┐
                             │
2. Solidity Advanced ────────┼──► 4. ERC Tokens ──► 5. Testing
                             │
3. Design Patterns ──────────┘
                                          │
                                          ▼
                              6. Deployment ──► 7. Monitoring
                                          │
                                          ▼
                              8. Upgrades (опционально)
```

## Инструменты

| Категория | Инструменты |
|-----------|-------------|
| Фреймворки | Hardhat, Foundry, Truffle |
| IDE | Remix, VS Code + Solidity |
| Тестирование | Mocha, Forge, Waffle |
| Безопасность | Slither, Mythril, Echidna |
| Мониторинг | The Graph, Tenderly, Defender |

## Дополнительные ресурсы

- [Solidity Documentation](https://docs.soliditylang.org/)
- [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/)
- [Ethereum Development Tutorials](https://ethereum.org/developers/tutorials/)
- [Smart Contract Security Best Practices](https://consensys.github.io/smart-contract-best-practices/)
