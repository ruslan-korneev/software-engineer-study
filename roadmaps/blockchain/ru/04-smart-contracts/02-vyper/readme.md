# Vyper

## Что такое Vyper и зачем он нужен

**Vyper** — это питоноподобный язык программирования для написания смарт-контрактов на Ethereum Virtual Machine (EVM). Он был создан как альтернатива Solidity с акцентом на безопасность, простоту и аудируемость кода.

### Основные цели Vyper

1. **Безопасность** — язык намеренно ограничен, чтобы минимизировать поверхность атаки
2. **Простота** — код должен быть максимально читаемым и понятным
3. **Аудируемость** — легко проверять, что делает контракт

### Почему Vyper?

- Синтаксис, похожий на Python, делает код интуитивно понятным
- Отсутствие сложных конструкций снижает вероятность ошибок
- Предсказуемое потребление газа
- Идеален для контрактов, где безопасность критична (DeFi, хранилища)

---

## Сравнение с Solidity

| Характеристика | Vyper | Solidity |
|----------------|-------|----------|
| Синтаксис | Python-подобный | JavaScript/C-подобный |
| Наследование | Нет | Да (множественное) |
| Модификаторы функций | Нет | Да |
| Рекурсия | Запрещена | Разрешена |
| Inline Assembly | Нет | Да |
| Циклы | Только ограниченные | Любые |
| Перегрузка функций | Нет | Да |
| Сложность | Низкая | Высокая |

### Пример: один и тот же контракт

**Solidity:**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SimpleStorage {
    uint256 private storedValue;

    function set(uint256 _value) public {
        storedValue = _value;
    }

    function get() public view returns (uint256) {
        return storedValue;
    }
}
```

**Vyper:**
```vyper
# @version ^0.3.7

stored_value: uint256

@external
def set(_value: uint256):
    self.stored_value = _value

@external
@view
def get() -> uint256:
    return self.stored_value
```

---

## Основы синтаксиса Vyper

### Структура контракта

```vyper
# @version ^0.3.7

# Интерфейсы (импорты)
from vyper.interfaces import ERC20

# Константы
MAX_SUPPLY: constant(uint256) = 1000000

# Неизменяемые переменные (устанавливаются при деплое)
owner: immutable(address)

# Переменные состояния
total_supply: public(uint256)
balances: HashMap[address, uint256]

# События
event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    amount: uint256

# Конструктор
@external
def __init__():
    owner = msg.sender
    self.total_supply = 0

# Функции
@external
def transfer(_to: address, _amount: uint256):
    # логика функции
    pass
```

### Отступы и форматирование

Vyper, как и Python, использует отступы для определения блоков кода:

```vyper
@external
def example(_condition: bool) -> uint256:
    if _condition:
        return 100
    else:
        return 0
```

---

## Типы данных и переменные

### Базовые типы

```vyper
# Целые числа
my_int: int256 = -100           # Знаковое целое (от -2^255 до 2^255-1)
my_uint: uint256 = 100          # Беззнаковое целое (от 0 до 2^256-1)
small_int: uint8 = 255          # 8-битное беззнаковое

# Логический тип
is_active: bool = True

# Адрес
user_address: address = 0x1234567890123456789012345678901234567890

# Байты
data: bytes32 = 0x0000000000000000000000000000000000000000000000000000000000000001
dynamic_bytes: Bytes[100] = b"Hello"  # Динамические байты (макс. 100)

# Строки
name: String[64] = "My Token"   # Строка с максимальной длиной 64
```

### Составные типы

```vyper
# Статический массив
numbers: uint256[5]             # Массив из 5 элементов

# HashMap (аналог mapping в Solidity)
balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

# Структуры
struct Person:
    name: String[100]
    age: uint256
    wallet: address

people: HashMap[uint256, Person]
```

### Специальные типы переменных

```vyper
# Константа — известна на этапе компиляции
MAX_VALUE: constant(uint256) = 1000

# Immutable — устанавливается один раз в конструкторе
deployer: immutable(address)

@external
def __init__():
    deployer = msg.sender

# Public — автоматически создает getter
total_count: public(uint256)
```

---

## Функции и декораторы

### Декораторы видимости

```vyper
@external    # Можно вызвать извне контракта
def external_function():
    pass

@internal    # Только внутри контракта
def _internal_function():
    pass
```

### Декораторы состояния

```vyper
@view        # Только чтение, не изменяет состояние
@external
def get_balance(_addr: address) -> uint256:
    return self.balances[_addr]

@pure        # Не читает и не изменяет состояние
@external
def add(_a: uint256, _b: uint256) -> uint256:
    return _a + _b
```

### Специальные декораторы

```vyper
@payable     # Функция может принимать ETH
@external
def deposit():
    self.balances[msg.sender] += msg.value

@nonreentrant("lock")  # Защита от reentrancy атак
@external
def withdraw(_amount: uint256):
    assert self.balances[msg.sender] >= _amount
    self.balances[msg.sender] -= _amount
    send(msg.sender, _amount)
```

### Возвращаемые значения

```vyper
@external
@view
def get_info() -> (uint256, address, bool):
    return (self.total_supply, self.owner, self.is_active)
```

---

## Примеры смарт-контрактов на Vyper

### Простой токен ERC-20

```vyper
# @version ^0.3.7

from vyper.interfaces import ERC20

implements: ERC20

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    value: uint256

name: public(String[32])
symbol: public(String[32])
decimals: public(uint8)
totalSupply: public(uint256)
balanceOf: public(HashMap[address, uint256])
allowance: public(HashMap[address, HashMap[address, uint256]])

@external
def __init__(_name: String[32], _symbol: String[32], _supply: uint256):
    self.name = _name
    self.symbol = _symbol
    self.decimals = 18
    self.totalSupply = _supply * 10 ** 18
    self.balanceOf[msg.sender] = self.totalSupply
    log Transfer(empty(address), msg.sender, self.totalSupply)

@external
def transfer(_to: address, _value: uint256) -> bool:
    self.balanceOf[msg.sender] -= _value
    self.balanceOf[_to] += _value
    log Transfer(msg.sender, _to, _value)
    return True

@external
def approve(_spender: address, _value: uint256) -> bool:
    self.allowance[msg.sender][_spender] = _value
    log Approval(msg.sender, _spender, _value)
    return True

@external
def transferFrom(_from: address, _to: address, _value: uint256) -> bool:
    self.allowance[_from][msg.sender] -= _value
    self.balanceOf[_from] -= _value
    self.balanceOf[_to] += _value
    log Transfer(_from, _to, _value)
    return True
```

### Аукцион

```vyper
# @version ^0.3.7

event Bid:
    bidder: indexed(address)
    amount: uint256

owner: immutable(address)
end_time: public(uint256)
highest_bidder: public(address)
highest_bid: public(uint256)
ended: public(bool)

pending_returns: HashMap[address, uint256]

@external
def __init__(_duration: uint256):
    owner = msg.sender
    self.end_time = block.timestamp + _duration

@external
@payable
def bid():
    assert block.timestamp < self.end_time, "Auction ended"
    assert msg.value > self.highest_bid, "Bid too low"

    if self.highest_bidder != empty(address):
        self.pending_returns[self.highest_bidder] += self.highest_bid

    self.highest_bidder = msg.sender
    self.highest_bid = msg.value
    log Bid(msg.sender, msg.value)

@external
def withdraw():
    amount: uint256 = self.pending_returns[msg.sender]
    assert amount > 0, "Nothing to withdraw"
    self.pending_returns[msg.sender] = 0
    send(msg.sender, amount)

@external
def end_auction():
    assert block.timestamp >= self.end_time, "Not yet ended"
    assert not self.ended, "Already ended"
    self.ended = True
    send(owner, self.highest_bid)
```

---

## Ограничения языка (намеренные для безопасности)

### Что запрещено в Vyper

1. **Наследование классов** — избегает проблем с порядком разрешения методов
2. **Модификаторы функций** — логика должна быть явной в теле функции
3. **Inline Assembly** — предотвращает низкоуровневые манипуляции
4. **Рекурсия** — исключает stack overflow и сложности с газом
5. **Бесконечные циклы** — все циклы должны иметь известный верхний предел
6. **Перегрузка операторов** — упрощает понимание кода
7. **Бинарные операции с фиксированной точкой** — предотвращает ошибки округления

### Ограниченные циклы

```vyper
# Неправильно (не скомпилируется):
# for i in range(n):  # n — переменная

# Правильно:
MAX_ITERATIONS: constant(uint256) = 100

@external
def process_items(_items: DynArray[uint256, 100]):
    for i in range(MAX_ITERATIONS):
        if i >= len(_items):
            break
        # обработка _items[i]
```

### Проверки переполнения

Vyper автоматически проверяет арифметические операции:

```vyper
@external
def safe_add(_a: uint256, _b: uint256) -> uint256:
    # Автоматически откатится при переполнении
    return _a + _b
```

---

## Инструменты разработки

### Компилятор Vyper

```bash
# Установка
pip install vyper

# Компиляция
vyper contract.vy                    # Байткод
vyper -f abi contract.vy             # ABI
vyper -f bytecode_runtime contract.vy # Runtime bytecode
```

### Brownie (фреймворк)

```bash
# Установка
pip install eth-brownie

# Создание проекта
brownie init

# Компиляция Vyper контрактов
brownie compile

# Тестирование
brownie test
```

### Ape Framework

```bash
# Установка
pip install eth-ape

# Инициализация проекта
ape init

# Установка плагина Vyper
ape plugins install vyper

# Компиляция
ape compile
```

### Тестирование с pytest

```python
# tests/test_token.py
import pytest
from brownie import Token, accounts

@pytest.fixture
def token():
    return Token.deploy("MyToken", "MTK", 1000, {"from": accounts[0]})

def test_initial_supply(token):
    assert token.totalSupply() == 1000 * 10**18

def test_transfer(token):
    token.transfer(accounts[1], 100, {"from": accounts[0]})
    assert token.balanceOf(accounts[1]) == 100
```

### Foundry (поддержка Vyper)

```bash
# Установка vyper плагина для forge
forge install vyperlang/vyper

# В foundry.toml
[profile.default]
vyper = "0.3.7"
```

---

## Лучшие практики

1. **Используйте `@nonreentrant`** для функций, работающих с ETH
2. **Проверяйте входные данные** с помощью `assert`
3. **Делайте переменные `public`** только когда нужен автоматический getter
4. **Используйте `immutable`** для значений, известных при деплое
5. **Документируйте код** с помощью NatSpec комментариев

```vyper
@external
@view
def get_user_balance(_user: address) -> uint256:
    """
    @notice Возвращает баланс пользователя
    @param _user Адрес пользователя
    @return Баланс в wei
    """
    return self.balances[_user]
```

---

## Ресурсы для изучения

- [Официальная документация Vyper](https://docs.vyperlang.org/)
- [Vyper by Example](https://vyper-by-example.org/)
- [GitHub репозиторий Vyper](https://github.com/vyperlang/vyper)
- [Curve Finance контракты (Vyper)](https://github.com/curvefi/curve-contract)
