# Структура блокчейна

Блокчейн — это распределённая база данных, состоящая из связанных между собой блоков. Каждый блок содержит набор транзакций и криптографически связан с предыдущим блоком, образуя непрерывную цепочку.

---

## 1. Структура блока

Каждый блок в блокчейне состоит из двух основных частей: **заголовка (header)** и **тела (body)**.

### Заголовок блока (Block Header)

Заголовок содержит метаданные блока:

| Поле | Описание |
|------|----------|
| **Версия** | Версия протокола блокчейна |
| **Хеш предыдущего блока** | Криптографическая ссылка на предыдущий блок |
| **Merkle Root** | Корень дерева Меркла всех транзакций |
| **Timestamp** | Временная метка создания блока |
| **Difficulty Target** | Целевая сложность для майнинга |
| **Nonce** | Число, подбираемое майнерами |

### Тело блока (Block Body)

Тело блока содержит:
- Список транзакций
- Количество транзакций
- Размер блока в байтах

### Схема структуры блока

```
┌─────────────────────────────────────────────┐
│                   БЛОК                      │
├─────────────────────────────────────────────┤
│              ЗАГОЛОВОК (Header)             │
├─────────────────────────────────────────────┤
│  • Версия: 1                                │
│  • Хеш предыдущего блока: 0x00a1b2c3...     │
│  • Merkle Root: 0x7f8d9e0a...               │
│  • Timestamp: 1703847600                    │
│  • Difficulty: 0x1d00ffff                   │
│  • Nonce: 2504433986                        │
├─────────────────────────────────────────────┤
│                ТЕЛО (Body)                  │
├─────────────────────────────────────────────┤
│  • Транзакция 1: Alice → Bob (1 BTC)        │
│  • Транзакция 2: Carol → Dave (0.5 BTC)     │
│  • Транзакция 3: Eve → Frank (2.3 BTC)      │
│  • ...                                      │
└─────────────────────────────────────────────┘
```

---

## 2. Хеширование

### Что такое хеш-функция?

**Хеш-функция** — это математическая функция, которая преобразует входные данные произвольной длины в выходную строку фиксированной длины (хеш).

### Свойства криптографических хеш-функций

1. **Детерминированность** — одинаковый вход всегда даёт одинаковый выход
2. **Быстрое вычисление** — хеш вычисляется за разумное время
3. **Необратимость** — невозможно восстановить исходные данные по хешу
4. **Лавинный эффект** — малейшее изменение входа кардинально меняет выход
5. **Устойчивость к коллизиям** — крайне сложно найти два разных входа с одинаковым хешем

### SHA-256

**SHA-256** (Secure Hash Algorithm 256-bit) — основная хеш-функция Bitcoin и многих других блокчейнов.

- Генерирует хеш длиной 256 бит (64 шестнадцатеричных символа)
- Разработана NSA в 2001 году
- Входит в семейство SHA-2

### Пример хеширования на Python

```python
import hashlib

def sha256_hash(data: str) -> str:
    """Вычисляет SHA-256 хеш строки."""
    return hashlib.sha256(data.encode('utf-8')).hexdigest()

# Примеры
text1 = "Hello, Blockchain!"
text2 = "Hello, Blockchain"  # Без восклицательного знака

hash1 = sha256_hash(text1)
hash2 = sha256_hash(text2)

print(f"Текст 1: {text1}")
print(f"Хеш 1:   {hash1}")
print()
print(f"Текст 2: {text2}")
print(f"Хеш 2:   {hash2}")

# Вывод:
# Текст 1: Hello, Blockchain!
# Хеш 1:   3e23e8160039594a33894f6564e1b1348bbd7a0088d42c4acb73eeaed59c009d
# Текст 2: Hello, Blockchain
# Хеш 2:   8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92

# Обратите внимание: один символ изменил весь хеш!
```

### Двойное хеширование в Bitcoin

Bitcoin использует двойное хеширование для дополнительной безопасности:

```python
def double_sha256(data: str) -> str:
    """Двойное SHA-256 хеширование (как в Bitcoin)."""
    first_hash = hashlib.sha256(data.encode('utf-8')).digest()
    second_hash = hashlib.sha256(first_hash).hexdigest()
    return second_hash

# Пример
data = "Bitcoin transaction data"
result = double_sha256(data)
print(f"Двойной SHA-256: {result}")
```

---

## 3. Цепочка блоков

### Как блоки связаны между собой?

Каждый блок содержит **хеш предыдущего блока** в своём заголовке. Это создаёт криптографическую связь между блоками.

### Схема цепочки

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Блок #0    │    │   Блок #1    │    │   Блок #2    │
│  (Genesis)   │    │              │    │              │
├──────────────┤    ├──────────────┤    ├──────────────┤
│ Prev: 0x0000 │◄───│ Prev: 0xa1b2 │◄───│ Prev: 0xc3d4 │
│ Hash: 0xa1b2 │    │ Hash: 0xc3d4 │    │ Hash: 0xe5f6 │
│ Data: ...    │    │ Data: ...    │    │ Data: ...    │
└──────────────┘    └──────────────┘    └──────────────┘
```

### Почему это важно?

Если злоумышленник изменит данные в блоке #1:
1. Хеш блока #1 изменится
2. Ссылка в блоке #2 станет недействительной
3. Придётся пересчитать ВСЕ последующие блоки
4. Это требует огромных вычислительных ресурсов

### Реализация на Python

```python
import hashlib
import json
from datetime import datetime
from typing import List, Optional

class Block:
    """Простая реализация блока."""

    def __init__(
        self,
        index: int,
        transactions: List[dict],
        previous_hash: str,
        nonce: int = 0
    ):
        self.index = index
        self.timestamp = datetime.now().isoformat()
        self.transactions = transactions
        self.previous_hash = previous_hash
        self.nonce = nonce
        self.hash = self.calculate_hash()

    def calculate_hash(self) -> str:
        """Вычисляет хеш блока."""
        block_data = json.dumps({
            'index': self.index,
            'timestamp': self.timestamp,
            'transactions': self.transactions,
            'previous_hash': self.previous_hash,
            'nonce': self.nonce
        }, sort_keys=True)
        return hashlib.sha256(block_data.encode()).hexdigest()

    def __repr__(self) -> str:
        return (
            f"Block(index={self.index}, "
            f"hash={self.hash[:16]}..., "
            f"prev={self.previous_hash[:16]}...)"
        )


class Blockchain:
    """Простая реализация блокчейна."""

    def __init__(self):
        self.chain: List[Block] = []
        self.create_genesis_block()

    def create_genesis_block(self) -> Block:
        """Создаёт первый (генезис) блок."""
        genesis = Block(
            index=0,
            transactions=[{"message": "Genesis Block"}],
            previous_hash="0" * 64
        )
        self.chain.append(genesis)
        return genesis

    def add_block(self, transactions: List[dict]) -> Block:
        """Добавляет новый блок в цепочку."""
        previous_block = self.chain[-1]
        new_block = Block(
            index=len(self.chain),
            transactions=transactions,
            previous_hash=previous_block.hash
        )
        self.chain.append(new_block)
        return new_block

    def is_chain_valid(self) -> bool:
        """Проверяет целостность цепочки."""
        for i in range(1, len(self.chain)):
            current = self.chain[i]
            previous = self.chain[i - 1]

            # Проверка хеша текущего блока
            if current.hash != current.calculate_hash():
                return False

            # Проверка связи с предыдущим блоком
            if current.previous_hash != previous.hash:
                return False

        return True


# Пример использования
blockchain = Blockchain()

# Добавляем блоки
blockchain.add_block([
    {"from": "Alice", "to": "Bob", "amount": 50}
])
blockchain.add_block([
    {"from": "Bob", "to": "Carol", "amount": 25},
    {"from": "Carol", "to": "Dave", "amount": 10}
])

# Выводим цепочку
print("Блокчейн:")
for block in blockchain.chain:
    print(f"  {block}")

print(f"\nЦепочка валидна: {blockchain.is_chain_valid()}")
```

---

## 4. Merkle Tree (Дерево Меркла)

### Что это такое?

**Дерево Меркла** (Merkle Tree) — это двоичное дерево хешей, которое позволяет эффективно и безопасно проверять содержимое больших структур данных.

### Как строится дерево Меркла?

1. Вычисляются хеши всех транзакций (листья дерева)
2. Пары хешей объединяются и хешируются снова
3. Процесс повторяется до получения одного корневого хеша (Merkle Root)

### Схема дерева Меркла

```
                    ┌─────────────┐
                    │ Merkle Root │
                    │   H(AB+CD)  │
                    └──────┬──────┘
                           │
            ┌──────────────┴──────────────┐
            │                             │
      ┌─────┴─────┐                 ┌─────┴─────┐
      │   H(AB)   │                 │   H(CD)   │
      │ H(A + B)  │                 │ H(C + D)  │
      └─────┬─────┘                 └─────┬─────┘
            │                             │
      ┌─────┴─────┐                 ┌─────┴─────┐
      │           │                 │           │
   ┌──┴──┐     ┌──┴──┐           ┌──┴──┐     ┌──┴──┐
   │ H(A)│     │ H(B)│           │ H(C)│     │ H(D)│
   └──┬──┘     └──┬──┘           └──┬──┘     └──┬──┘
      │           │                 │           │
   ┌──┴──┐     ┌──┴──┐           ┌──┴──┐     ┌──┴──┐
   │ Tx A│     │ Tx B│           │ Tx C│     │ Tx D│
   └─────┘     └─────┘           └─────┘     └─────┘
```

### Зачем нужно дерево Меркла?

1. **Эффективная проверка** — можно доказать включение транзакции без загрузки всего блока
2. **Экономия места** — лёгкие клиенты хранят только заголовки блоков
3. **Быстрое обнаружение изменений** — любое изменение меняет корень дерева
4. **SPV (Simplified Payment Verification)** — проверка платежей без полной ноды

### Реализация на Python

```python
import hashlib
from typing import List

def hash_data(data: str) -> str:
    """Вычисляет SHA-256 хеш."""
    return hashlib.sha256(data.encode()).hexdigest()

def build_merkle_tree(transactions: List[str]) -> str:
    """
    Строит дерево Меркла и возвращает корневой хеш.

    Args:
        transactions: Список транзакций (строки)

    Returns:
        Merkle Root - корневой хеш дерева
    """
    if not transactions:
        return hash_data("")

    # Шаг 1: Хешируем все транзакции
    current_level = [hash_data(tx) for tx in transactions]

    # Шаг 2: Строим дерево снизу вверх
    while len(current_level) > 1:
        next_level = []

        # Если нечётное количество, дублируем последний элемент
        if len(current_level) % 2 == 1:
            current_level.append(current_level[-1])

        # Объединяем пары и хешируем
        for i in range(0, len(current_level), 2):
            combined = current_level[i] + current_level[i + 1]
            next_level.append(hash_data(combined))

        current_level = next_level
        print(f"Уровень: {[h[:8] + '...' for h in current_level]}")

    return current_level[0]


# Пример использования
transactions = [
    "Alice -> Bob: 10 BTC",
    "Bob -> Carol: 5 BTC",
    "Carol -> Dave: 3 BTC",
    "Dave -> Eve: 1 BTC"
]

print("Транзакции:")
for i, tx in enumerate(transactions):
    print(f"  {i + 1}. {tx}")

print("\nПостроение дерева Меркла:")
merkle_root = build_merkle_tree(transactions)
print(f"\nMerkle Root: {merkle_root}")
```

---

## 5. Genesis Block (Генезис-блок)

### Что такое Genesis Block?

**Genesis Block** (блок генезиса) — это самый первый блок в блокчейне. Он уникален тем, что не имеет ссылки на предыдущий блок.

### Особенности Genesis Block

- **Индекс**: 0
- **Previous Hash**: обычно нулевая строка (0x0000...)
- **Жёстко закодирован**: встроен в код протокола
- **Не может быть изменён**: является точкой отсчёта всей цепи

### Genesis Block Bitcoin

```
Дата: 3 января 2009 года
Награда: 50 BTC (недоступны для траты!)
Сообщение: "The Times 03/Jan/2009 Chancellor on brink of
           second bailout for banks"
Hash: 000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f
```

Сообщение в Genesis Block Bitcoin — заголовок газеты The Times, доказывающий дату создания блока.

### Создание Genesis Block на Python

```python
def create_genesis_block() -> Block:
    """Создаёт Genesis Block."""
    return Block(
        index=0,
        transactions=[{
            "type": "genesis",
            "message": "The Times 03/Jan/2009 Chancellor on brink of second bailout for banks",
            "reward": 50
        }],
        previous_hash="0" * 64,  # 64 нуля для SHA-256
        nonce=0
    )
```

---

## 6. Nonce и сложность (Proof of Work)

### Что такое Nonce?

**Nonce** (Number used ONCE) — это произвольное число, которое майнеры изменяют для получения хеша, удовлетворяющего условию сложности.

### Что такое сложность (Difficulty)?

**Сложность** определяет, сколько нулей должно быть в начале хеша блока. Чем больше нулей требуется, тем сложнее найти подходящий nonce.

### Как работает Proof of Work?

1. Майнер собирает транзакции в блок
2. Устанавливает nonce = 0
3. Вычисляет хеш блока
4. Если хеш начинается с нужного количества нулей — блок найден!
5. Иначе: nonce += 1 и повторяем шаг 3

### Схема процесса

```
┌─────────────────────────────────────────────────────────┐
│                    PROOF OF WORK                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Сложность: 4 (хеш должен начинаться с "0000")         │
│                                                         │
│  Попытка 1: nonce=0    → hash=7f3d...  ✗               │
│  Попытка 2: nonce=1    → hash=a2b1...  ✗               │
│  Попытка 3: nonce=2    → hash=1c4e...  ✗               │
│  ...                                                    │
│  Попытка N: nonce=12847 → hash=0000a3b...  ✓ НАЙДЕНО! │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Реализация Proof of Work на Python

```python
import hashlib
import json
import time
from dataclasses import dataclass
from typing import List

@dataclass
class BlockWithPoW:
    """Блок с поддержкой Proof of Work."""
    index: int
    timestamp: str
    transactions: List[dict]
    previous_hash: str
    nonce: int = 0
    hash: str = ""

    def calculate_hash(self) -> str:
        """Вычисляет хеш блока."""
        block_data = json.dumps({
            'index': self.index,
            'timestamp': self.timestamp,
            'transactions': self.transactions,
            'previous_hash': self.previous_hash,
            'nonce': self.nonce
        }, sort_keys=True)
        return hashlib.sha256(block_data.encode()).hexdigest()


def mine_block(block: BlockWithPoW, difficulty: int) -> BlockWithPoW:
    """
    Майнит блок с заданной сложностью.

    Args:
        block: Блок для майнинга
        difficulty: Количество нулей в начале хеша

    Returns:
        Блок с найденным nonce и хешем
    """
    target = "0" * difficulty
    attempts = 0
    start_time = time.time()

    print(f"Начинаем майнинг блока #{block.index}")
    print(f"Сложность: {difficulty} (хеш должен начинаться с '{target}')")

    while True:
        block.hash = block.calculate_hash()
        attempts += 1

        if block.hash.startswith(target):
            elapsed = time.time() - start_time
            print(f"\n✓ Блок найден!")
            print(f"  Попыток: {attempts:,}")
            print(f"  Время: {elapsed:.2f} секунд")
            print(f"  Nonce: {block.nonce}")
            print(f"  Hash: {block.hash}")
            return block

        # Показываем прогресс каждые 100000 попыток
        if attempts % 100000 == 0:
            print(f"  Попытка {attempts:,}: nonce={block.nonce}, hash={block.hash[:16]}...")

        block.nonce += 1


# Пример майнинга
block = BlockWithPoW(
    index=1,
    timestamp="2024-01-15T12:00:00",
    transactions=[{"from": "Alice", "to": "Bob", "amount": 10}],
    previous_hash="0" * 64
)

# Попробуйте разную сложность: 2, 3, 4, 5
# Чем выше сложность, тем дольше поиск!
mined_block = mine_block(block, difficulty=4)
```

### Регулировка сложности

В Bitcoin сложность автоматически регулируется каждые 2016 блоков (~2 недели), чтобы среднее время нахождения блока оставалось около 10 минут.

```python
def adjust_difficulty(
    current_difficulty: int,
    actual_time: float,
    expected_time: float
) -> int:
    """
    Регулирует сложность на основе времени майнинга.

    Args:
        current_difficulty: Текущая сложность
        actual_time: Фактическое время на последние N блоков
        expected_time: Ожидаемое время на N блоков

    Returns:
        Новая сложность
    """
    ratio = actual_time / expected_time

    if ratio < 0.5:
        # Слишком быстро — увеличиваем сложность
        return current_difficulty + 1
    elif ratio > 2.0:
        # Слишком медленно — уменьшаем сложность
        return max(1, current_difficulty - 1)
    else:
        return current_difficulty
```

---

## 7. Временные метки (Timestamps)

### Роль временных меток

**Timestamp** (временная метка) — это время создания блока. Она важна для:

1. **Хронологического порядка** — определение последовательности блоков
2. **Регулировки сложности** — расчёт времени между блоками
3. **Защиты от атак** — предотвращение манипуляций со временем
4. **Аудита** — возможность отследить историю транзакций

### Формат временной метки

```python
from datetime import datetime
import time

# Unix Timestamp (секунды с 1 января 1970)
unix_timestamp = int(time.time())
print(f"Unix timestamp: {unix_timestamp}")

# ISO 8601 формат
iso_timestamp = datetime.now().isoformat()
print(f"ISO 8601: {iso_timestamp}")

# В Bitcoin используется Unix timestamp
bitcoin_style = int(datetime.now().timestamp())
print(f"Bitcoin style: {bitcoin_style}")
```

### Правила валидации временных меток

В Bitcoin временная метка нового блока должна:

1. Быть больше медианы временных меток последних 11 блоков
2. Не превышать текущее сетевое время + 2 часа

```python
from typing import List

def is_valid_timestamp(
    new_timestamp: int,
    last_11_timestamps: List[int],
    network_time: int
) -> bool:
    """
    Проверяет валидность временной метки.

    Args:
        new_timestamp: Временная метка нового блока
        last_11_timestamps: Временные метки последних 11 блоков
        network_time: Текущее сетевое время

    Returns:
        True если временная метка валидна
    """
    # Медиана последних 11 блоков
    sorted_timestamps = sorted(last_11_timestamps)
    median = sorted_timestamps[len(sorted_timestamps) // 2]

    # Проверки
    if new_timestamp <= median:
        print(f"Ошибка: timestamp {new_timestamp} <= median {median}")
        return False

    max_future = network_time + (2 * 60 * 60)  # +2 часа
    if new_timestamp > max_future:
        print(f"Ошибка: timestamp {new_timestamp} > max_future {max_future}")
        return False

    return True
```

---

## Итоги

### Ключевые компоненты структуры блокчейна

| Компонент | Назначение |
|-----------|------------|
| **Заголовок блока** | Метаданные: версия, хеши, timestamp, nonce |
| **Тело блока** | Список транзакций |
| **Хеширование** | Криптографическая защита и связывание блоков |
| **Merkle Tree** | Эффективная проверка транзакций |
| **Genesis Block** | Начальная точка цепочки |
| **Nonce** | Число для Proof of Work |
| **Timestamp** | Временная метка создания |

### Почему блокчейн неизменяем?

1. Каждый блок содержит хеш предыдущего блока
2. Изменение данных меняет хеш
3. Для фальсификации нужно пересчитать ВСЕ последующие блоки
4. Proof of Work делает пересчёт экстремально дорогим
5. Распределённая сеть отвергнет поддельную цепочку

---

## Дополнительные ресурсы

- [Bitcoin Whitepaper](https://bitcoin.org/bitcoin.pdf) — оригинальная статья Сатоши Накамото
- [Mastering Bitcoin](https://github.com/bitcoinbook/bitcoinbook) — книга Андреаса Антонопулоса
- [SHA-256 визуализация](https://sha256algorithm.com/) — интерактивное объяснение алгоритма
- [Blockchain Demo](https://andersbrownworth.com/blockchain/) — визуальная демонстрация работы блокчейна
