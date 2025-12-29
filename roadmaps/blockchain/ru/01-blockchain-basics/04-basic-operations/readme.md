# Базовые операции блокчейна

Базовые операции блокчейна включают в себя создание транзакций, управление ключами, криптографическую подпись данных, валидацию и достижение консенсуса между узлами сети.

---

## 1. Транзакции

**Транзакция** — это атомарная единица данных в блокчейне, представляющая передачу ценности или выполнение операции.

### Структура транзакции

Типичная транзакция содержит:

| Поле | Описание |
|------|----------|
| `txid` | Уникальный идентификатор (хеш транзакции) |
| `inputs` | Ссылки на предыдущие транзакции (откуда берутся средства) |
| `outputs` | Адреса получателей и суммы |
| `timestamp` | Время создания |
| `signature` | Цифровая подпись отправителя |
| `nonce` | Счетчик для предотвращения повторов (в Ethereum) |

### Пример структуры транзакции в Python

```python
from dataclasses import dataclass
from typing import List
import hashlib
import json
import time


@dataclass
class TxInput:
    """Вход транзакции — ссылка на предыдущий выход."""
    prev_txid: str      # ID предыдущей транзакции
    output_index: int   # Индекс выхода в предыдущей транзакции
    signature: str = "" # Подпись владельца


@dataclass
class TxOutput:
    """Выход транзакции — получатель и сумма."""
    address: str        # Адрес получателя
    amount: float       # Сумма


@dataclass
class Transaction:
    """Транзакция блокчейна."""
    inputs: List[TxInput]
    outputs: List[TxOutput]
    timestamp: float = None

    def __post_init__(self):
        if self.timestamp is None:
            self.timestamp = time.time()

    def to_dict(self) -> dict:
        """Сериализация транзакции в словарь."""
        return {
            "inputs": [
                {"prev_txid": inp.prev_txid, "output_index": inp.output_index}
                for inp in self.inputs
            ],
            "outputs": [
                {"address": out.address, "amount": out.amount}
                for out in self.outputs
            ],
            "timestamp": self.timestamp
        }

    def calculate_hash(self) -> str:
        """Вычисление хеша транзакции."""
        tx_string = json.dumps(self.to_dict(), sort_keys=True)
        return hashlib.sha256(tx_string.encode()).hexdigest()


# Пример создания транзакции
tx = Transaction(
    inputs=[TxInput(prev_txid="abc123...", output_index=0)],
    outputs=[
        TxOutput(address="1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa", amount=0.5),
        TxOutput(address="1BvBMSEYstWetqTFn5Au4m4GFg7xJaNVN2", amount=0.3)  # Сдача
    ]
)

print(f"ID транзакции: {tx.calculate_hash()}")
```

---

## 2. Публичные и приватные ключи

Криптография с открытым ключом (асимметричная криптография) — основа безопасности блокчейна.

### Принцип работы

```
                    ┌─────────────────┐
                    │ Приватный ключ  │
                    │ (256 бит)       │
                    └────────┬────────┘
                             │
                    Эллиптическая кривая (ECDSA)
                             │
                             ▼
                    ┌─────────────────┐
                    │ Публичный ключ  │
                    │ (512 бит)       │
                    └────────┬────────┘
                             │
                    Hash160 (SHA256 + RIPEMD160)
                             │
                             ▼
                    ┌─────────────────┐
                    │ Адрес кошелька  │
                    └─────────────────┘
```

### Свойства ключей

| Характеристика | Приватный ключ | Публичный ключ |
|----------------|----------------|----------------|
| Секретность | Строго конфиденциален | Можно публиковать |
| Назначение | Подпись транзакций | Верификация подписей |
| Восстановление | Невозможно | Вычисляется из приватного |
| Формат (Bitcoin) | WIF (Wallet Import Format) | Сжатый/несжатый |

### Генерация ключей с помощью Python

```python
from ecdsa import SigningKey, SECP256k1
import hashlib
import base58


def generate_keypair():
    """Генерация пары ключей для блокчейна."""
    # Генерация приватного ключа (256 бит случайных данных)
    private_key = SigningKey.generate(curve=SECP256k1)

    # Получение публичного ключа
    public_key = private_key.get_verifying_key()

    return private_key, public_key


def private_key_to_wif(private_key: SigningKey) -> str:
    """Конвертация приватного ключа в WIF формат (Bitcoin mainnet)."""
    # Добавляем префикс 0x80 для mainnet
    extended_key = b'\x80' + private_key.to_string()

    # Двойной SHA256 для контрольной суммы
    checksum = hashlib.sha256(hashlib.sha256(extended_key).digest()).digest()[:4]

    # Base58Check кодирование
    return base58.b58encode(extended_key + checksum).decode()


def public_key_to_address(public_key) -> str:
    """Конвертация публичного ключа в Bitcoin-адрес."""
    # SHA256 хеш публичного ключа
    sha256_hash = hashlib.sha256(public_key.to_string()).digest()

    # RIPEMD160 хеш
    ripemd160 = hashlib.new('ripemd160')
    ripemd160.update(sha256_hash)
    public_key_hash = ripemd160.digest()

    # Добавляем версию (0x00 для mainnet)
    versioned_hash = b'\x00' + public_key_hash

    # Контрольная сумма
    checksum = hashlib.sha256(hashlib.sha256(versioned_hash).digest()).digest()[:4]

    # Base58Check кодирование
    return base58.b58encode(versioned_hash + checksum).decode()


# Демонстрация
private_key, public_key = generate_keypair()
print(f"Приватный ключ (hex): {private_key.to_string().hex()}")
print(f"Приватный ключ (WIF): {private_key_to_wif(private_key)}")
print(f"Публичный ключ (hex): {public_key.to_string().hex()}")
print(f"Адрес кошелька: {public_key_to_address(public_key)}")
```

---

## 3. Цифровая подпись (ECDSA)

**ECDSA** (Elliptic Curve Digital Signature Algorithm) — алгоритм цифровой подписи на эллиптических кривых.

### Процесс подписи

```
┌─────────────┐     ┌─────────────────┐     ┌─────────────┐
│   Данные    │ ──▶ │    SHA256       │ ──▶ │    Хеш      │
└─────────────┘     └─────────────────┘     └──────┬──────┘
                                                    │
                    ┌─────────────────┐             │
                    │ Приватный ключ  │             │
                    └────────┬────────┘             │
                             │                      │
                             ▼                      ▼
                    ┌───────────────────────────────┐
                    │        ECDSA подпись          │
                    │         (r, s)                │
                    └───────────────────────────────┘
```

### Реализация подписи и верификации

```python
from ecdsa import SigningKey, VerifyingKey, SECP256k1, BadSignatureError
import hashlib


class DigitalSignature:
    """Класс для работы с цифровыми подписями."""

    @staticmethod
    def sign_message(private_key: SigningKey, message: str) -> bytes:
        """
        Подписать сообщение приватным ключом.

        Args:
            private_key: Приватный ключ
            message: Сообщение для подписи

        Returns:
            Цифровая подпись в байтах
        """
        # Хешируем сообщение
        message_hash = hashlib.sha256(message.encode()).digest()

        # Создаем подпись
        signature = private_key.sign(message_hash)

        return signature

    @staticmethod
    def verify_signature(public_key: VerifyingKey, message: str, signature: bytes) -> bool:
        """
        Проверить подпись с помощью публичного ключа.

        Args:
            public_key: Публичный ключ
            message: Оригинальное сообщение
            signature: Подпись для проверки

        Returns:
            True если подпись валидна, False в противном случае
        """
        try:
            message_hash = hashlib.sha256(message.encode()).digest()
            public_key.verify(signature, message_hash)
            return True
        except BadSignatureError:
            return False


# Демонстрация
private_key = SigningKey.generate(curve=SECP256k1)
public_key = private_key.get_verifying_key()

# Подписываем сообщение
message = "Отправить 1 BTC на адрес 1A1zP1..."
signature = DigitalSignature.sign_message(private_key, message)

print(f"Сообщение: {message}")
print(f"Подпись: {signature.hex()}")

# Верифицируем подпись
is_valid = DigitalSignature.verify_signature(public_key, message, signature)
print(f"Подпись валидна: {is_valid}")

# Попытка подделки — изменяем сообщение
tampered_message = "Отправить 100 BTC на адрес 1A1zP1..."
is_valid_tampered = DigitalSignature.verify_signature(public_key, tampered_message, signature)
print(f"Подделанная подпись валидна: {is_valid_tampered}")
```

---

## 4. Валидация транзакций

Каждый узел сети проверяет входящие транзакции перед их ретрансляцией и включением в блок.

### Этапы валидации

1. **Синтаксическая проверка**
   - Корректность формата данных
   - Наличие всех обязательных полей
   - Допустимые размеры полей

2. **Проверка подписи**
   - Подпись создана владельцем входов
   - Подпись соответствует данным транзакции

3. **Проверка двойной траты**
   - Входы не были потрачены ранее
   - Входы существуют в UTXO-наборе

4. **Проверка баланса**
   - Сумма входов >= сумма выходов
   - Разница идет на комиссию майнеру

5. **Проверка правил консенсуса**
   - Соответствие текущим правилам сети
   - Корректный формат скриптов

### Пример валидатора

```python
from typing import Set, Optional


class TransactionValidator:
    """Валидатор транзакций блокчейна."""

    def __init__(self, utxo_set: Set[str]):
        """
        Args:
            utxo_set: Множество непотраченных выходов (UTXO)
        """
        self.utxo_set = utxo_set  # Формат: "txid:output_index"

    def validate(self, tx: Transaction, signatures_valid: bool = True) -> tuple[bool, str]:
        """
        Полная валидация транзакции.

        Returns:
            (is_valid, error_message)
        """
        # 1. Проверка структуры
        if not tx.inputs:
            return False, "Транзакция не имеет входов"

        if not tx.outputs:
            return False, "Транзакция не имеет выходов"

        # 2. Проверка сумм (не должны быть отрицательными)
        for output in tx.outputs:
            if output.amount <= 0:
                return False, f"Недопустимая сумма выхода: {output.amount}"

        # 3. Проверка двойной траты (входы должны быть в UTXO)
        input_total = 0.0
        for inp in tx.inputs:
            utxo_key = f"{inp.prev_txid}:{inp.output_index}"
            if utxo_key not in self.utxo_set:
                return False, f"Вход не найден в UTXO: {utxo_key}"
            # В реальности здесь бы получали сумму из UTXO
            input_total += 1.0  # Упрощение

        # 4. Проверка подписей (упрощено)
        if not signatures_valid:
            return False, "Недействительная подпись"

        # 5. Проверка баланса
        output_total = sum(out.amount for out in tx.outputs)
        if input_total < output_total:
            return False, f"Недостаточно средств: {input_total} < {output_total}"

        return True, "Транзакция валидна"


# Пример использования
utxo = {"abc123:0", "def456:1", "ghi789:0"}
validator = TransactionValidator(utxo)

# Валидная транзакция
valid_tx = Transaction(
    inputs=[TxInput(prev_txid="abc123", output_index=0)],
    outputs=[TxOutput(address="addr1", amount=0.5)]
)
is_valid, msg = validator.validate(valid_tx)
print(f"Транзакция 1: {msg}")

# Попытка двойной траты
double_spend_tx = Transaction(
    inputs=[TxInput(prev_txid="nonexistent", output_index=0)],
    outputs=[TxOutput(address="addr1", amount=0.5)]
)
is_valid, msg = validator.validate(double_spend_tx)
print(f"Транзакция 2: {msg}")
```

---

## 5. Механизмы консенсуса

**Консенсус** — это процесс достижения согласия между узлами распределенной сети о состоянии блокчейна.

### Сравнение основных механизмов

| Механизм | Энергопотребление | Скорость | Децентрализация | Безопасность |
|----------|-------------------|----------|-----------------|--------------|
| **PoW** | Очень высокое | Низкая | Высокая | Высокая |
| **PoS** | Низкое | Средняя | Средняя | Высокая |
| **DPoS** | Низкое | Высокая | Низкая | Средняя |
| **PBFT** | Низкое | Высокая | Низкая | Высокая |

### Proof of Work (PoW)

**Принцип**: Майнеры соревнуются в решении криптографической задачи (нахождение хеша с определенным количеством ведущих нулей).

```
                    ┌─────────────────────────┐
                    │      Данные блока       │
                    │  + предыдущий хеш       │
                    │  + nonce                │
                    └───────────┬─────────────┘
                                │
                                ▼
                    ┌─────────────────────────┐
                    │        SHA256           │
                    └───────────┬─────────────┘
                                │
                                ▼
            ┌───────────────────┴───────────────────┐
            │                                       │
            ▼                                       ▼
    Хеш < целевого                          Хеш >= целевого
    значения?                               значения
            │                                       │
            ▼                                       ▼
    Блок найден!                            Увеличить nonce,
    Награда майнеру                         повторить
```

### Proof of Stake (PoS)

**Принцип**: Валидаторы блокируют (stake) свои монеты как залог. Право создания блока выбирается пропорционально размеру стейка.

**Преимущества:**
- Энергоэффективность
- Нет необходимости в специальном оборудовании
- Экономическая мотивация честного поведения

**Примеры:** Ethereum 2.0, Cardano, Polkadot

### Delegated Proof of Stake (DPoS)

**Принцип**: Держатели токенов голосуют за делегатов, которые будут создавать блоки.

**Особенности:**
- Высокая скорость транзакций (до 10,000+ TPS)
- Ограниченное число валидаторов (21-101)
- Демократическое управление

**Примеры:** EOS, TRON, BitShares

### Practical Byzantine Fault Tolerance (PBFT)

**Принцип**: Узлы обмениваются сообщениями для достижения консенсуса. Система устойчива к (n-1)/3 злонамеренным узлам.

**Фазы PBFT:**
1. Pre-prepare — лидер предлагает блок
2. Prepare — узлы подтверждают получение
3. Commit — узлы голосуют за принятие

**Примеры:** Hyperledger Fabric, некоторые приватные блокчейны

---

## 6. Майнинг

**Майнинг** — процесс создания новых блоков путем решения криптографической задачи (в PoW).

### Процесс майнинга

```python
import hashlib
import time


class Block:
    """Блок в блокчейне."""

    def __init__(self, index: int, transactions: list, previous_hash: str):
        self.index = index
        self.timestamp = time.time()
        self.transactions = transactions
        self.previous_hash = previous_hash
        self.nonce = 0
        self.hash = None

    def calculate_hash(self) -> str:
        """Вычисление хеша блока."""
        block_string = f"{self.index}{self.timestamp}{self.transactions}{self.previous_hash}{self.nonce}"
        return hashlib.sha256(block_string.encode()).hexdigest()


class Miner:
    """Майнер блокчейна."""

    def __init__(self, difficulty: int = 4):
        """
        Args:
            difficulty: Количество ведущих нулей в хеше
        """
        self.difficulty = difficulty
        self.target = "0" * difficulty

    def mine_block(self, block: Block) -> tuple[str, int, float]:
        """
        Майнинг блока — поиск nonce, дающего хеш с нужным количеством нулей.

        Returns:
            (hash, nonce, time_elapsed)
        """
        start_time = time.time()

        while True:
            block_hash = block.calculate_hash()

            if block_hash.startswith(self.target):
                elapsed = time.time() - start_time
                block.hash = block_hash
                return block_hash, block.nonce, elapsed

            block.nonce += 1

            # Вывод прогресса каждые 100000 попыток
            if block.nonce % 100000 == 0:
                print(f"Nonce: {block.nonce}, текущий хеш: {block_hash[:16]}...")


# Демонстрация майнинга
print("Запуск майнинга с difficulty=4 (4 ведущих нуля)")
print("-" * 50)

block = Block(
    index=1,
    transactions=["tx1: Alice -> Bob: 1 BTC", "tx2: Bob -> Charlie: 0.5 BTC"],
    previous_hash="0000000000000000000000000000000000000000000000000000000000000000"
)

miner = Miner(difficulty=4)
found_hash, nonce, elapsed = miner.mine_block(block)

print(f"\nБлок найден!")
print(f"Хеш: {found_hash}")
print(f"Nonce: {nonce}")
print(f"Время: {elapsed:.2f} секунд")
```

### Награда за майнинг

- **Block Reward** — новые монеты, создаваемые с каждым блоком
- **Transaction Fees** — комиссии за транзакции в блоке

В Bitcoin награда уменьшается вдвое каждые 210,000 блоков (примерно 4 года):
- 2009: 50 BTC
- 2012: 25 BTC
- 2016: 12.5 BTC
- 2020: 6.25 BTC
- 2024: 3.125 BTC

---

## 7. Подтверждение транзакций (Confirmations)

**Подтверждение** — это включение транзакции в блок и последующее добавление новых блоков поверх.

### Количество подтверждений

```
                    Текущий блок
                         │
    ┌────────┬───────────┼───────────┬────────┐
    ▼        ▼           ▼           ▼        ▼
┌──────┐ ┌──────┐   ┌──────┐   ┌──────┐ ┌──────┐
│Блок  │ │Блок  │   │Блок  │   │Блок  │ │Блок  │
│ N-4  │ │ N-3  │   │ N-2  │   │ N-1  │ │  N   │
└──────┘ └──────┘   └──────┘   └──────┘ └──────┘
                        │
                   ┌────┴────┐
                   │ TX в    │
                   │ блоке   │
                   │ N-2     │
                   └─────────┘
                        │
                   3 подтверждения
                   (блоки N-1, N, ...)
```

### Рекомендуемое количество подтверждений

| Сеть | Малые суммы | Средние суммы | Крупные суммы |
|------|-------------|---------------|---------------|
| Bitcoin | 1-2 | 3-4 | 6+ |
| Ethereum | 12 | 20 | 32+ |
| Litecoin | 3 | 6 | 12+ |

### Вероятность реорганизации

С каждым новым подтверждением вероятность отмены транзакции экспоненциально уменьшается:

- 1 подтверждение: ~50% риска (при атаке 50% хешрейта)
- 2 подтверждения: ~25% риска
- 6 подтверждений: ~0.024% риска

```python
def calculate_confirmation_security(confirmations: int, attacker_hashrate: float = 0.3) -> float:
    """
    Расчет вероятности успешной атаки двойной траты.

    Args:
        confirmations: Количество подтверждений
        attacker_hashrate: Доля хешрейта атакующего (0-1)

    Returns:
        Вероятность успеха атаки
    """
    if attacker_hashrate >= 0.5:
        return 1.0  # 51% атака всегда успешна в долгосрочной перспективе

    # Упрощенная формула: q^n, где q = attacker_hashrate / (1 - attacker_hashrate)
    q = attacker_hashrate / (1 - attacker_hashrate)
    probability = q ** confirmations

    return probability


# Пример расчета безопасности
print("Вероятность успешной атаки (30% хешрейта атакующего):")
for conf in [1, 2, 3, 4, 5, 6]:
    prob = calculate_confirmation_security(conf, 0.3)
    print(f"  {conf} подтверждение(й): {prob*100:.4f}%")
```

---

## Практические советы

### Безопасность ключей

1. **Никогда не храните приватные ключи в открытом виде**
2. **Используйте аппаратные кошельки для крупных сумм**
3. **Создавайте резервные копии seed-фразы**
4. **Не генерируйте ключи на онлайн-сервисах**

### Отправка транзакций

1. **Проверяйте адрес получателя несколько раз**
2. **Используйте адекватную комиссию для скорости подтверждения**
3. **Дождитесь нужного количества подтверждений перед считыванием транзакции завершенной**

### Выбор консенсуса для проекта

- **PoW** — максимальная безопасность, публичные сети
- **PoS** — энергоэффективность, средняя децентрализация
- **DPoS** — высокая скорость, меньшая децентрализация
- **PBFT** — приватные/консорциумные блокчейны

---

## Дополнительные ресурсы

- [Bitcoin Developer Documentation](https://developer.bitcoin.org/)
- [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf)
- [Mastering Bitcoin by Andreas Antonopoulos](https://github.com/bitcoinbook/bitcoinbook)
- [ECDSA Specification (SEC 2)](https://www.secg.org/sec2-v2.pdf)

---

## Итоги

В этом разделе мы изучили:

1. **Транзакции** — структура, входы/выходы, хеширование
2. **Криптография** — генерация ключей, адреса кошельков
3. **Цифровые подписи** — ECDSA, подпись и верификация
4. **Валидация** — проверки, которые выполняют ноды
5. **Консенсус** — PoW, PoS, DPoS, PBFT и их сравнение
6. **Майнинг** — процесс создания блоков
7. **Подтверждения** — безопасность и вероятность реорганизации

Эти концепции составляют фундамент работы любого блокчейна и необходимы для понимания более сложных тем.
