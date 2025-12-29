# Тестирование смарт-контрактов

## Введение

Тестирование смарт-контрактов — критически важный этап разработки блокчейн-приложений. В отличие от традиционного программного обеспечения, смарт-контракты после деплоя становятся **неизменяемыми** (immutable), а ошибки могут привести к потере реальных денег. История блокчейна знает множество дорогостоящих взломов из-за уязвимостей в коде.

## Почему тестирование критически важно

### Финансовые риски

- **The DAO Hack (2016)** — потеря $60 млн из-за уязвимости reentrancy
- **Parity Wallet (2017)** — заморожено $150 млн навсегда
- **Ronin Bridge (2022)** — украдено $625 млн
- **Wormhole (2022)** — взлом на $320 млн

### Особенности смарт-контрактов

```
┌─────────────────────────────────────────────────────────────┐
│              Традиционный софт vs Смарт-контракты           │
├─────────────────────────────┬───────────────────────────────┤
│     Традиционный софт       │       Смарт-контракты         │
├─────────────────────────────┼───────────────────────────────┤
│ Можно исправить баги        │ Код неизменяем после деплоя   │
│ Откат изменений возможен    │ Транзакции необратимы         │
│ Приватные данные скрыты     │ Всё публично на блокчейне     │
│ Ограниченный финансовый риск│ Прямой доступ к средствам     │
│ Централизованный контроль   │ Автономное исполнение         │
└─────────────────────────────┴───────────────────────────────┘
```

## Виды тестирования

### 1. Unit тесты (Модульные тесты)

Тестирование отдельных функций контракта в изоляции.

```javascript
// Пример unit-теста на Hardhat
describe("Token", function() {
  it("should transfer tokens correctly", async function() {
    const [owner, recipient] = await ethers.getSigners();
    const token = await Token.deploy(1000);

    await token.transfer(recipient.address, 100);

    expect(await token.balanceOf(recipient.address)).to.equal(100);
    expect(await token.balanceOf(owner.address)).to.equal(900);
  });
});
```

### 2. Интеграционные тесты

Проверка взаимодействия между несколькими контрактами.

```javascript
describe("DeFi Integration", function() {
  it("should swap tokens through DEX", async function() {
    // Деплой токенов и DEX
    const tokenA = await TokenA.deploy();
    const tokenB = await TokenB.deploy();
    const dex = await DEX.deploy(tokenA.address, tokenB.address);

    // Тестирование полного флоу обмена
    await tokenA.approve(dex.address, 1000);
    await dex.swap(tokenA.address, tokenB.address, 100);

    // Проверка результатов
    expect(await tokenB.balanceOf(user.address)).to.be.gt(0);
  });
});
```

### 3. Fuzz тестирование

Автоматическая генерация случайных входных данных для поиска edge cases.

```solidity
// Foundry fuzz test
function testFuzz_Transfer(address to, uint256 amount) public {
    vm.assume(to != address(0));
    vm.assume(amount <= token.balanceOf(address(this)));

    uint256 balanceBefore = token.balanceOf(to);
    token.transfer(to, amount);

    assertEq(token.balanceOf(to), balanceBefore + amount);
}
```

### 4. Invariant тесты

Проверка инвариантов — условий, которые всегда должны быть истинными.

```solidity
// Инвариант: общий supply токенов не должен меняться
function invariant_totalSupplyConstant() public {
    assertEq(token.totalSupply(), INITIAL_SUPPLY);
}
```

### 5. Fork тесты

Тестирование на форке реальной сети (mainnet fork).

```javascript
// Hardhat fork конфигурация
networks: {
  hardhat: {
    forking: {
      url: process.env.MAINNET_RPC_URL,
      blockNumber: 18000000
    }
  }
}
```

## Инструменты тестирования

### Hardhat

Самый популярный фреймворк для разработки на JavaScript/TypeScript.

```bash
# Установка
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox

# Запуск тестов
npx hardhat test
npx hardhat test --grep "transfer"  # Фильтрация тестов
```

**Преимущества:**
- Большое сообщество и экосистема плагинов
- Интеграция с ethers.js
- Console.log в Solidity для дебага
- Встроенная локальная сеть

### Foundry

Современный быстрый фреймворк на Rust с тестами на Solidity.

```bash
# Установка
curl -L https://foundry.paradigm.xyz | bash
foundryup

# Запуск тестов
forge test
forge test -vvvv  # Максимальная детализация
```

**Преимущества:**
- Высокая скорость выполнения
- Тесты на Solidity (ближе к реальному коду)
- Встроенный fuzz и invariant тестинг
- Cheatcodes для манипуляции состоянием

### Сравнение фреймворков

```
┌──────────────────┬─────────────────┬─────────────────┐
│   Критерий       │    Hardhat      │    Foundry      │
├──────────────────┼─────────────────┼─────────────────┤
│ Язык тестов      │ JavaScript/TS   │ Solidity        │
│ Скорость         │ Средняя         │ Очень быстрая   │
│ Fuzz testing     │ Через плагины   │ Встроенный      │
│ Экосистема       │ Большая         │ Растущая        │
│ Кривая обучения  │ Низкая          │ Средняя         │
│ Дебаг            │ console.log     │ Traces, labels  │
└──────────────────┴─────────────────┴─────────────────┘
```

## Структура тестов

### Типичная организация

```
test/
├── unit/
│   ├── Token.test.js
│   ├── Vault.test.js
│   └── Governance.test.js
├── integration/
│   ├── DeFiFlow.test.js
│   └── FullProtocol.test.js
├── fuzz/
│   └── Token.fuzz.t.sol
├── invariant/
│   └── Protocol.invariant.t.sol
└── fork/
    └── MainnetIntegration.test.js
```

### Паттерн AAA (Arrange-Act-Assert)

```javascript
describe("Vault", function() {
  it("should allow deposits", async function() {
    // Arrange - подготовка
    const vault = await Vault.deploy();
    const depositAmount = ethers.parseEther("1.0");

    // Act - действие
    await vault.deposit({ value: depositAmount });

    // Assert - проверка
    expect(await vault.balanceOf(owner.address)).to.equal(depositAmount);
  });
});
```

## Метрики качества тестов

### Code Coverage (Покрытие кода)

```bash
# Hardhat
npx hardhat coverage

# Foundry
forge coverage
```

**Целевые показатели:**
- **Lines** — покрытие строк кода (>90%)
- **Branches** — покрытие условных переходов (>85%)
- **Functions** — покрытие функций (100%)
- **Statements** — покрытие инструкций (>90%)

### Другие метрики

- **Mutation testing** — устойчивость тестов к изменениям кода
- **Gas snapshots** — отслеживание потребления газа
- **Test execution time** — скорость выполнения тестов

## Best Practices

### 1. Тестируйте все edge cases

```javascript
describe("Transfer edge cases", function() {
  it("should revert on zero address", async function() {
    await expect(
      token.transfer(ethers.ZeroAddress, 100)
    ).to.be.revertedWith("Invalid address");
  });

  it("should revert on insufficient balance", async function() {
    await expect(
      token.transfer(recipient.address, MAX_UINT256)
    ).to.be.revertedWith("Insufficient balance");
  });
});
```

### 2. Используйте fixtures для производительности

```javascript
async function deployFixture() {
  const [owner, user1, user2] = await ethers.getSigners();
  const token = await Token.deploy(1000000);
  return { token, owner, user1, user2 };
}

describe("Token", function() {
  it("test 1", async function() {
    const { token, owner } = await loadFixture(deployFixture);
    // ...
  });
});
```

### 3. Тестируйте события

```javascript
it("should emit Transfer event", async function() {
  await expect(token.transfer(recipient.address, 100))
    .to.emit(token, "Transfer")
    .withArgs(owner.address, recipient.address, 100);
});
```

### 4. Документируйте тесты

```javascript
/**
 * @notice Тестирование функции withdraw
 * @dev Проверяет корректность вывода средств с учетом комиссии
 * Сценарий: пользователь вносит 1 ETH, затем выводит с комиссией 0.1%
 */
it("should withdraw with fee", async function() {
  // ...
});
```

## Заключение

Тестирование смарт-контрактов требует особого подхода из-за неизменяемости кода и финансовых рисков. Комбинация unit, integration, fuzz и invariant тестов обеспечивает максимальное покрытие и уверенность в безопасности контракта. Инвестиции в качественное тестирование всегда окупаются — стоимость исправления бага в проде несравнимо выше стоимости его обнаружения на этапе тестирования.
