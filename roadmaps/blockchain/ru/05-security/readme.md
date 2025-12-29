# Безопасность смарт-контрактов

Безопасность — критически важный аспект разработки смарт-контрактов. В отличие от традиционного ПО, смарт-контракты после деплоя становятся неизменяемыми, а ошибки могут привести к потере миллионов долларов. Этот раздел охватывает все аспекты безопасности: от типичных уязвимостей до процесса профессионального аудита.

## Почему безопасность так важна

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 ОСОБЕННОСТИ СМАРТ-КОНТРАКТОВ                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐       │
│  │ НЕИЗМЕНЯЕМОСТЬ  │   │   ПУБЛИЧНОСТЬ   │   │   ФИНАНСОВЫЙ    │       │
│  │                 │   │                 │   │      РИСК       │       │
│  │  После деплоя   │   │   Код доступен  │   │   Контракты     │       │
│  │  код нельзя     │   │   для анализа   │   │   управляют     │       │
│  │  изменить       │   │   всем          │   │   миллиардами   │       │
│  └─────────────────┘   └─────────────────┘   └─────────────────┘       │
│                                                                         │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐       │
│  │ НЕОБРАТИМОСТЬ   │   │  COMPOSABILITY  │   │   24/7 ДОСТУП   │       │
│  │   ТРАНЗАКЦИЙ    │   │                 │   │                 │       │
│  │  Украденное     │   │   Взлом одного  │   │   Атаки могут   │       │
│  │  невозможно     │   │   протокола     │   │   происходить   │       │
│  │  вернуть        │   │   влияет на все │   │   круглосуточно │       │
│  └─────────────────┘   └─────────────────┘   └─────────────────┘       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Статистика потерь

| Год | Потери от взломов | Крупнейший инцидент |
|-----|-------------------|---------------------|
| 2016 | ~$60M | The DAO (reentrancy) |
| 2020 | ~$120M | bZx flash loan attacks |
| 2021 | ~$1.3B | Poly Network ($611M) |
| 2022 | ~$3.8B | Ronin Bridge ($625M) |
| 2023 | ~$1.7B | Mixin Network ($200M) |
| 2024 | ~$1.4B | WazirX ($235M) |

## Содержание раздела

### 1. Распространённые уязвимости
Изучение основных типов атак на смарт-контракты:

- [x] [**Reentrancy атаки**](./01-common-vulnerabilities/01-reentrancy/readme.md) — механизм, защита, паттерн CEI
- [x] [**Overflow/Underflow**](./01-common-vulnerabilities/02-overflow-underflow/readme.md) — арифметические уязвимости, SafeMath
- [x] [**Access Control**](./01-common-vulnerabilities/03-access-control/readme.md) — контроль доступа, Ownable, RBAC

[Обзор раздела →](./01-common-vulnerabilities/readme.md)

### 2. Практики безопасности
Методологии и подходы к обеспечению безопасности:

- [x] [**Fuzz Testing**](./02-security-practices/01-fuzz-testing/readme.md) — property-based testing, инварианты
- [x] [**Static Analysis**](./02-security-practices/02-static-analysis/readme.md) — автоматический анализ кода

[Обзор раздела →](./02-security-practices/readme.md)

### 3. Инструменты безопасности
Обзор основных инструментов для аудита и защиты:

- [x] [**Slither**](./03-security-tools/01-slither/readme.md) — статический анализатор от Trail of Bits
- [x] [**Echidna**](./03-security-tools/02-echidna/readme.md) — property-based фаззер
- [x] [**OpenZeppelin**](./03-security-tools/03-openzeppelin/readme.md) — библиотека безопасных контрактов

[Обзор раздела →](./03-security-tools/readme.md)

### 4. Процесс аудита
Всё о профессиональном аудите смарт-контрактов:

- [x] [**Процесс аудита**](./04-audit-process/readme.md) — этапы, методология, чеклисты, стоимость

## Пирамида безопасности

```
                           ▲
                          /│\     Формальная верификация
                         / │ \    (Certora, Halmos)
                        /  │  \
                       /   │   \   Внешний аудит
                      /    │    \  (Trail of Bits, OpenZeppelin)
                     /     │     \
                    /      │      \  Bug Bounty
                   /       │       \ (Immunefi, Code4rena)
                  /        │        \
                 /         │         \ Fuzz Testing
                /          │          \ (Echidna, Foundry)
               /           │           \
              /            │            \ Статический анализ
             /             │             \ (Slither, Mythril)
            /              │              \
           /               │               \ Unit тесты
          /                │                \ (Foundry, Hardhat)
         /                 │                 \
        ───────────────────────────────────────
                    Code Review & Best Practices
```

## Рекомендуемый порядок изучения

### Уровень 1: Основы безопасности
1. Изучите [распространённые уязвимости](./01-common-vulnerabilities/readme.md)
2. Поймите механизмы атак (Reentrancy, Overflow, Access Control)
3. Изучите паттерны защиты (CEI, ReentrancyGuard, Ownable)

### Уровень 2: Инструменты
1. Настройте [Slither](./03-security-tools/01-slither/readme.md) для вашего проекта
2. Научитесь писать property-based тесты с [Echidna](./03-security-tools/02-echidna/readme.md)
3. Используйте [OpenZeppelin](./03-security-tools/03-openzeppelin/readme.md) контракты

### Уровень 3: Практика
1. Решайте задачи на [Ethernaut](https://ethernaut.openzeppelin.com/)
2. Проходите [Damn Vulnerable DeFi](https://www.damnvulnerabledefi.xyz/)
3. Участвуйте в соревновательных аудитах на Code4rena

### Уровень 4: Профессиональный аудит
1. Изучите [процесс аудита](./04-audit-process/readme.md)
2. Читайте публичные аудиторские отчёты
3. Практикуйтесь в написании отчётов

## Чеклист безопасности перед деплоем

```markdown
## Pre-deployment Security Checklist

### Код
- [ ] Все функции имеют корректные модификаторы доступа
- [ ] Используется паттерн Checks-Effects-Interactions
- [ ] ReentrancyGuard на критических функциях
- [ ] Проверки нулевых адресов и значений
- [ ] События для всех значимых изменений состояния

### Тестирование
- [ ] Unit тесты с покрытием >90%
- [ ] Integration тесты
- [ ] Fuzz тесты (Echidna или Foundry)
- [ ] Fork тесты на mainnet данных

### Автоматический анализ
- [ ] Slither без high/medium findings
- [ ] Mythril анализ пройден
- [ ] Solhint без критичных замечаний

### Внешняя проверка
- [ ] Code review командой
- [ ] Внешний аудит (для продакшена)
- [ ] Bug bounty программа настроена

### Операционная безопасность
- [ ] Multisig для admin функций
- [ ] Timelock для критических изменений
- [ ] Emergency pause механизм
- [ ] Мониторинг настроен (Forta, Defender)
- [ ] Incident response план готов
```

## Ключевые концепции

### Defense in Depth
Многоуровневая защита — не полагайтесь на один механизм:

```
┌─────────────────────────────────────────────────────────┐
│ Layer 1: Code Quality                                   │
│   └── Clean code, comments, naming conventions          │
├─────────────────────────────────────────────────────────┤
│ Layer 2: Testing                                        │
│   └── Unit, integration, fuzz, formal verification      │
├─────────────────────────────────────────────────────────┤
│ Layer 3: Static Analysis                                │
│   └── Slither, Mythril, Aderyn                         │
├─────────────────────────────────────────────────────────┤
│ Layer 4: External Review                                │
│   └── Audit, bug bounty                                │
├─────────────────────────────────────────────────────────┤
│ Layer 5: Runtime Protection                             │
│   └── Monitoring, circuit breakers, incident response   │
└─────────────────────────────────────────────────────────┘
```

### Принцип минимальных привилегий
Давайте только необходимые права:

```solidity
// Плохо: одна роль admin с полными правами
function adminDoEverything() external onlyAdmin { ... }

// Хорошо: разделение обязанностей
function mint(address to, uint256 amount) external onlyRole(MINTER_ROLE) { ... }
function pause() external onlyRole(PAUSER_ROLE) { ... }
function upgrade(address newImpl) external onlyRole(UPGRADER_ROLE) { ... }
```

### Fail-Safe Defaults
По умолчанию — максимальная безопасность:

```solidity
// Плохо: по умолчанию разрешено
mapping(address => bool) public blacklisted;
function transfer(address to) external {
    require(!blacklisted[msg.sender], "Blacklisted");
}

// Хорошо: по умолчанию запрещено (для критичных операций)
mapping(address => bool) public whitelisted;
function criticalOperation() external {
    require(whitelisted[msg.sender], "Not whitelisted");
}
```

## Полезные ресурсы

### Обучение
- [Secureum Bootcamp](https://github.com/x676f64/secureum-mind_map) — структурированное обучение
- [Smart Contract Security Field Guide](https://scsfg.io/) — справочник по безопасности
- [Consensys Best Practices](https://consensys.github.io/smart-contract-best-practices/) — рекомендации

### Практика
- [Ethernaut](https://ethernaut.openzeppelin.com/) — CTF от OpenZeppelin
- [Damn Vulnerable DeFi](https://www.damnvulnerabledefi.xyz/) — DeFi уязвимости
- [Capture the Ether](https://capturetheether.com/) — классические задачи

### Инструменты
- [Slither](https://github.com/crytic/slither) — статический анализ
- [Echidna](https://github.com/crytic/echidna) — фаззинг
- [Foundry](https://github.com/foundry-rs/foundry) — тестирование и фаззинг
- [OpenZeppelin Contracts](https://github.com/OpenZeppelin/openzeppelin-contracts) — безопасные контракты

### Отчёты и исследования
- [Rekt News](https://rekt.news/) — разборы взломов
- [Trail of Bits Publications](https://github.com/trailofbits/publications) — публичные отчёты
- [SWC Registry](https://swcregistry.io/) — каталог уязвимостей

## Заключение

Безопасность смарт-контрактов — это не разовая задача, а непрерывный процесс. Ключевые принципы:

1. **Пишите простой код** — сложность порождает уязвимости
2. **Используйте проверенные библиотеки** — не изобретайте велосипед
3. **Тестируйте тщательно** — покрытие кода и edge cases
4. **Автоматизируйте проверки** — интегрируйте инструменты в CI/CD
5. **Проводите аудит** — внешний взгляд критически важен
6. **Мониторьте после деплоя** — реагируйте на инциденты быстро

> "В мире смарт-контрактов ошибка в одной строке кода может стоить миллионы долларов."
