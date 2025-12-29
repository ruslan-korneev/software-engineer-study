# DAO (Децентрализованные автономные организации)

## Что такое DAO?

**DAO (Decentralized Autonomous Organization)** — это организация, управляемая смарт-контрактами и её участниками через механизмы голосования. В отличие от традиционных компаний с иерархической структурой, DAO функционирует на основе прозрачных правил, записанных в коде.

### Ключевые характеристики DAO

- **Децентрализация** — нет единого центра управления
- **Автономность** — правила исполняются автоматически смарт-контрактами
- **Прозрачность** — все решения и транзакции видны в блокчейне
- **Токенизированное управление** — участники голосуют токенами

### Как работает DAO

```
┌─────────────────────────────────────────────────────────────┐
│                        DAO Lifecycle                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Создание предложения (Proposal)                         │
│         ↓                                                   │
│  2. Период обсуждения                                       │
│         ↓                                                   │
│  3. Голосование (Voting)                                    │
│         ↓                                                   │
│  4. Подсчёт голосов                                         │
│         ↓                                                   │
│  5. Timelock (период ожидания)                              │
│         ↓                                                   │
│  6. Исполнение (Execution)                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Governance Tokens (Токены управления)

Governance токены предоставляют держателям право голоса в DAO. Количество токенов обычно определяет вес голоса.

### Типы governance токенов

| Тип | Описание | Примеры |
|-----|----------|---------|
| **Native governance** | Токен создан специально для управления | UNI, COMP, AAVE |
| **Wrapped tokens** | Обёрнутая версия другого токена | veTokens (veCRV) |
| **NFT governance** | NFT как единица голоса | Nouns DAO |
| **Soulbound tokens** | Непередаваемые токены | некоторые DAO для репутации |

### Пример ERC20Votes токена

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Votes.sol";

contract GovernanceToken is ERC20, ERC20Permit, ERC20Votes {
    constructor()
        ERC20("MyGovernanceToken", "MGT")
        ERC20Permit("MyGovernanceToken")
    {
        _mint(msg.sender, 1_000_000 * 10**decimals());
    }

    // Переопределение функций для совместимости ERC20 и ERC20Votes
    function _update(
        address from,
        address to,
        uint256 value
    ) internal override(ERC20, ERC20Votes) {
        super._update(from, to, value);
    }

    function nonces(address owner)
        public
        view
        override(ERC20Permit, Nonces)
        returns (uint256)
    {
        return super.nonces(owner);
    }
}
```

### Делегирование голосов

Важная особенность ERC20Votes — делегирование. Пользователь должен делегировать свои голоса (себе или другому адресу), чтобы они учитывались:

```solidity
// Делегирование себе
governanceToken.delegate(msg.sender);

// Делегирование другому адресу
governanceToken.delegate(delegateeAddress);

// Делегирование через подпись (gasless)
governanceToken.delegateBySig(delegatee, nonce, expiry, v, r, s);
```

---

## Механизмы голосования

### On-chain голосование

Голосование происходит непосредственно в блокчейне через смарт-контракты.

**Преимущества:**
- Полная прозрачность
- Автоматическое исполнение
- Нельзя подделать результаты

**Недостатки:**
- Высокая стоимость газа
- Низкая активность голосования
- Медленное исполнение

### Off-chain голосование (Snapshot)

**Snapshot** — платформа для off-chain голосования, использующая подписи вместо транзакций.

```
┌─────────────────────────────────────────────────────────┐
│                  Snapshot Voting Flow                    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. Создатель публикует предложение                     │
│         ↓                                               │
│  2. Snapshot фиксирует баланс токенов на блоке          │
│         ↓                                               │
│  3. Участники подписывают голос (EIP-712)               │
│         ↓                                               │
│  4. Подписи хранятся в IPFS                             │
│         ↓                                               │
│  5. Результаты верифицируемы off-chain                  │
│         ↓                                               │
│  6. Исполнение через multisig или on-chain              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Преимущества Snapshot:**
- Бесплатное голосование (gasless)
- Высокая активность участников
- Гибкие стратегии подсчёта голосов

**Недостатки:**
- Требует доверия к исполнителям
- Не автоматизировано

### Сравнение подходов

| Характеристика | On-chain | Off-chain (Snapshot) |
|----------------|----------|---------------------|
| Стоимость | Высокая (gas) | Бесплатно |
| Безопасность | Максимальная | Требует доверия |
| Автоматизация | Полная | Ручное исполнение |
| Скорость | Медленно | Быстро |
| Участие | Низкое | Высокое |

---

## Timelock контракты

**Timelock** — контракт, который задерживает исполнение транзакций на определённый период. Это даёт участникам время отреагировать на потенциально вредоносные предложения.

### Зачем нужен Timelock

1. **Защита от атак** — время для обнаружения и реакции
2. **Прозрачность** — все видят предстоящие изменения
3. **Возможность выхода** — пользователи могут вывести средства до исполнения

### Пример использования TimelockController

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/governance/TimelockController.sol";

contract MyTimelock is TimelockController {
    // minDelay — минимальная задержка в секундах
    // proposers — адреса, которые могут создавать предложения
    // executors — адреса, которые могут исполнять предложения
    // admin — начальный администратор (обычно address(0) для отказа от прав)

    constructor(
        uint256 minDelay,
        address[] memory proposers,
        address[] memory executors,
        address admin
    ) TimelockController(minDelay, proposers, executors, admin) {}
}
```

### Workflow с Timelock

```solidity
// 1. Планирование операции
timelock.schedule(
    target,      // адрес контракта
    value,       // количество ETH
    data,        // calldata
    predecessor, // зависимость от другой операции
    salt,        // уникальный идентификатор
    delay        // задержка (не меньше minDelay)
);

// 2. После истечения задержки — исполнение
timelock.execute(
    target,
    value,
    data,
    predecessor,
    salt
);

// 3. Отмена (если необходимо)
timelock.cancel(operationId);
```

---

## OpenZeppelin Governor

OpenZeppelin предоставляет модульную систему для создания DAO на основе контракта **Governor**.

### Архитектура Governor

```
┌─────────────────────────────────────────────────────────────┐
│                    Governor Architecture                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐    ┌─────────────────────────────┐    │
│  │  Governor Core  │◄───│  GovernorSettings           │    │
│  └────────┬────────┘    │  (voting delay, period,     │    │
│           │             │   proposal threshold)        │    │
│           │             └─────────────────────────────┘    │
│           │                                                 │
│           │             ┌─────────────────────────────┐    │
│           ├─────────────│  GovernorVotes              │    │
│           │             │  (ERC20Votes integration)    │    │
│           │             └─────────────────────────────┘    │
│           │                                                 │
│           │             ┌─────────────────────────────┐    │
│           ├─────────────│  GovernorVotesQuorumFraction│    │
│           │             │  (quorum calculation)        │    │
│           │             └─────────────────────────────┘    │
│           │                                                 │
│           │             ┌─────────────────────────────┐    │
│           └─────────────│  GovernorTimelockControl    │    │
│                         │  (timelock integration)      │    │
│                         └─────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Полный пример Governor контракта

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/governance/Governor.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorSettings.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorCountingSimple.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorVotes.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorVotesQuorumFraction.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorTimelockControl.sol";

contract MyGovernor is
    Governor,
    GovernorSettings,
    GovernorCountingSimple,
    GovernorVotes,
    GovernorVotesQuorumFraction,
    GovernorTimelockControl
{
    constructor(
        IVotes _token,
        TimelockController _timelock
    )
        Governor("MyGovernor")
        GovernorSettings(
            7200,   // voting delay: 1 день (в блоках, ~12 сек/блок)
            50400,  // voting period: 1 неделя
            100e18  // proposal threshold: 100 токенов
        )
        GovernorVotes(_token)
        GovernorVotesQuorumFraction(4) // 4% кворум
        GovernorTimelockControl(_timelock)
    {}

    // Обязательные переопределения для разрешения конфликтов

    function votingDelay()
        public
        view
        override(Governor, GovernorSettings)
        returns (uint256)
    {
        return super.votingDelay();
    }

    function votingPeriod()
        public
        view
        override(Governor, GovernorSettings)
        returns (uint256)
    {
        return super.votingPeriod();
    }

    function quorum(uint256 blockNumber)
        public
        view
        override(Governor, GovernorVotesQuorumFraction)
        returns (uint256)
    {
        return super.quorum(blockNumber);
    }

    function state(uint256 proposalId)
        public
        view
        override(Governor, GovernorTimelockControl)
        returns (ProposalState)
    {
        return super.state(proposalId);
    }

    function proposalNeedsQueuing(uint256 proposalId)
        public
        view
        override(Governor, GovernorTimelockControl)
        returns (bool)
    {
        return super.proposalNeedsQueuing(proposalId);
    }

    function proposalThreshold()
        public
        view
        override(Governor, GovernorSettings)
        returns (uint256)
    {
        return super.proposalThreshold();
    }

    function _queueOperations(
        uint256 proposalId,
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        bytes32 descriptionHash
    ) internal override(Governor, GovernorTimelockControl) returns (uint48) {
        return super._queueOperations(proposalId, targets, values, calldatas, descriptionHash);
    }

    function _executeOperations(
        uint256 proposalId,
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        bytes32 descriptionHash
    ) internal override(Governor, GovernorTimelockControl) {
        super._executeOperations(proposalId, targets, values, calldatas, descriptionHash);
    }

    function _cancel(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        bytes32 descriptionHash
    ) internal override(Governor, GovernorTimelockControl) returns (uint256) {
        return super._cancel(targets, values, calldatas, descriptionHash);
    }

    function _executor()
        internal
        view
        override(Governor, GovernorTimelockControl)
        returns (address)
    {
        return super._executor();
    }
}
```

### Взаимодействие с Governor

```solidity
// 1. Создание предложения
uint256 proposalId = governor.propose(
    targets,     // массив адресов контрактов
    values,      // массив значений ETH
    calldatas,   // массив calldata
    description  // описание предложения
);

// 2. Голосование
// support: 0 = Against, 1 = For, 2 = Abstain
governor.castVote(proposalId, 1); // голос "За"

// Голосование с причиной
governor.castVoteWithReason(proposalId, 1, "This proposal improves...");

// 3. Постановка в очередь (после успешного голосования)
governor.queue(targets, values, calldatas, descriptionHash);

// 4. Исполнение (после истечения timelock)
governor.execute(targets, values, calldatas, descriptionHash);
```

---

## Примеры реальных DAO

### Uniswap DAO

**UNI** — governance токен Uniswap с общей эмиссией 1 млрд токенов.

| Параметр | Значение |
|----------|----------|
| Proposal Threshold | 0.25% от supply (~2.5M UNI) |
| Quorum | 4% от supply (~40M UNI) |
| Voting Period | 7 дней |
| Timelock Delay | 2 дня |

**Управляет:**
- Параметры протокола
- Распределение treasury
- Гранты и инициативы

### Compound DAO

**COMP** — один из первых governance токенов в DeFi.

| Параметр | Значение |
|----------|----------|
| Proposal Threshold | 25,000 COMP |
| Quorum | 400,000 COMP |
| Voting Period | ~3 дня |
| Timelock Delay | 2 дня |

**Управляет:**
- Процентные ставки
- Поддерживаемые активы
- Параметры ликвидации

### Aave DAO

**AAVE** и **stkAAVE** — токены управления Aave.

| Параметр | Значение |
|----------|----------|
| Proposal Threshold | 80,000 AAVE |
| Quorum | 2% или 6.5% (зависит от типа) |
| Voting Period | 3 дня |
| Timelock Delay | 1 день |

**Особенности:**
- Два уровня голосования (Short/Long Timelock)
- Guardians для экстренной отмены
- Cross-chain governance

### Сравнение параметров DAO

| DAO | Proposal Threshold | Quorum | Voting Period | Timelock |
|-----|-------------------|--------|---------------|----------|
| Uniswap | 2.5M UNI (0.25%) | 40M UNI (4%) | 7 дней | 2 дня |
| Compound | 25K COMP (~0.25%) | 400K COMP (~4%) | 3 дня | 2 дня |
| Aave | 80K AAVE (~0.5%) | 2-6.5% | 3 дня | 1 день |
| MakerDAO | N/A (executive) | N/A | Continuous | 48 часов |

---

## Паттерны и best practices

### 1. Многоуровневое управление

```solidity
// Разные timelock для разных типов решений
contract MultiTierGovernance {
    TimelockController public shortTimelock;  // 1 день — minor changes
    TimelockController public longTimelock;   // 7 дней — critical changes

    function getTimelockForAction(bytes4 selector) public view returns (TimelockController) {
        if (isCriticalAction(selector)) {
            return longTimelock;
        }
        return shortTimelock;
    }
}
```

### 2. Emergency механизмы

```solidity
contract GovernorWithEmergency is Governor {
    address public guardian;

    modifier onlyGuardian() {
        require(msg.sender == guardian, "Not guardian");
        _;
    }

    // Экстренная отмена вредоносного предложения
    function emergencyCancel(uint256 proposalId) external onlyGuardian {
        _cancel(proposalId);
        emit EmergencyCancelled(proposalId);
    }

    // Guardian может отказаться от прав
    function renounceGuardian() external onlyGuardian {
        guardian = address(0);
    }
}
```

### 3. Защита от flash loan атак

```solidity
// ERC20Votes уже защищён — использует checkpoints
// Голосующий должен иметь токены ДО создания предложения

function getPastVotes(address account, uint256 blockNumber)
    public view returns (uint256)
{
    // Возвращает баланс на момент snapshot block
    // Flash loan не поможет, т.к. токены нужны заранее
}
```

### 4. Делегирование с условиями

```solidity
contract ConditionalDelegation {
    mapping(address => DelegationRule) public delegationRules;

    struct DelegationRule {
        address delegate;
        uint256 maxVotingPower;
        bytes4[] allowedActions;
    }

    // Делегат может голосовать только в рамках условий
}
```

### 5. Оптимизация газа для голосования

```solidity
// Batch voting для нескольких предложений
function castVoteBatch(
    uint256[] calldata proposalIds,
    uint8[] calldata supports
) external {
    for (uint256 i = 0; i < proposalIds.length; i++) {
        _castVote(proposalIds[i], msg.sender, supports[i], "");
    }
}
```

---

## Безопасность DAO

### Распространённые уязвимости

| Уязвимость | Описание | Защита |
|------------|----------|--------|
| **Flash loan attack** | Займ токенов для голосования | Checkpoints (ERC20Votes) |
| **Proposal spam** | Создание множества предложений | Proposal threshold |
| **Low participation** | Недостаток голосов для кворума | Incentives, off-chain voting |
| **Timelock bypass** | Обход задержки | Правильная настройка ролей |
| **Proposal manipulation** | Изменение предложения | Хеширование description |

### Checklist безопасности

```
✓ Используйте ERC20Votes для защиты от flash loans
✓ Установите разумный proposal threshold
✓ Настройте достаточный timelock delay
✓ Добавьте guardian для экстренных ситуаций
✓ Проведите аудит перед деплоем
✓ Тестируйте все сценарии голосования
✓ Документируйте процесс governance
```

---

## Полезные ресурсы

- [OpenZeppelin Governor Documentation](https://docs.openzeppelin.com/contracts/5.x/governance)
- [Snapshot Documentation](https://docs.snapshot.org/)
- [Compound Governance](https://compound.finance/governance)
- [Tally — Governance Dashboard](https://www.tally.xyz/)
- [How to Build a DAO](https://ethereum.org/en/dao/)

---

## Практические задания

1. **Базовое DAO** — создайте governance токен и Governor контракт
2. **Интеграция Timelock** — добавьте задержку исполнения
3. **Off-chain voting** — настройте space в Snapshot
4. **Proposal lifecycle** — реализуйте полный цикл от предложения до исполнения
5. **Emergency system** — добавьте guardian и механизм отмены
