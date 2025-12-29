# Криптография и хеширование в блокчейне

Криптография является фундаментом безопасности блокчейн-технологий. Она обеспечивает целостность данных, аутентификацию участников и невозможность подделки транзакций.

---

## Основы криптографии

Криптография — это наука о методах обеспечения конфиденциальности, целостности и подлинности информации.

### Симметричное шифрование (AES)

**Симметричное шифрование** использует один и тот же ключ для шифрования и расшифрования данных.

**AES (Advanced Encryption Standard)** — современный стандарт симметричного шифрования:
- Размеры ключа: 128, 192 или 256 бит
- Блочный шифр с размером блока 128 бит
- Высокая скорость работы

```python
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend
import os

def aes_encrypt(plaintext: bytes, key: bytes) -> tuple[bytes, bytes]:
    """Шифрование AES в режиме CBC"""
    iv = os.urandom(16)  # Вектор инициализации
    cipher = Cipher(algorithms.AES(key), modes.CBC(iv), backend=default_backend())
    encryptor = cipher.encryptor()

    # Добавляем padding до кратности 16 байтам
    padding_length = 16 - (len(plaintext) % 16)
    padded_plaintext = plaintext + bytes([padding_length] * padding_length)

    ciphertext = encryptor.update(padded_plaintext) + encryptor.finalize()
    return ciphertext, iv

def aes_decrypt(ciphertext: bytes, key: bytes, iv: bytes) -> bytes:
    """Расшифрование AES в режиме CBC"""
    cipher = Cipher(algorithms.AES(key), modes.CBC(iv), backend=default_backend())
    decryptor = cipher.decryptor()

    padded_plaintext = decryptor.update(ciphertext) + decryptor.finalize()
    padding_length = padded_plaintext[-1]

    return padded_plaintext[:-padding_length]

# Пример использования
key = os.urandom(32)  # 256-битный ключ
message = b"Hello, Blockchain!"

encrypted, iv = aes_encrypt(message, key)
decrypted = aes_decrypt(encrypted, key, iv)
print(f"Исходное сообщение: {message}")
print(f"Расшифрованное: {decrypted}")
```

### Асимметричное шифрование (RSA, ECC)

**Асимметричное шифрование** использует пару ключей: публичный (для шифрования) и приватный (для расшифрования).

#### RSA (Rivest-Shamir-Adleman)

- Основан на сложности факторизации больших чисел
- Типичные размеры ключей: 2048, 4096 бит
- Медленнее симметричного шифрования

```python
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.primitives import hashes

# Генерация ключевой пары RSA
private_key = rsa.generate_private_key(
    public_exponent=65537,
    key_size=2048
)
public_key = private_key.public_key()

# Шифрование публичным ключом
message = b"Secret message for blockchain"
ciphertext = public_key.encrypt(
    message,
    padding.OAEP(
        mgf=padding.MGF1(algorithm=hashes.SHA256()),
        algorithm=hashes.SHA256(),
        label=None
    )
)

# Расшифрование приватным ключом
plaintext = private_key.decrypt(
    ciphertext,
    padding.OAEP(
        mgf=padding.MGF1(algorithm=hashes.SHA256()),
        algorithm=hashes.SHA256(),
        label=None
    )
)
print(f"Расшифрованное сообщение: {plaintext.decode()}")
```

#### ECC (Elliptic Curve Cryptography)

ECC — криптография на эллиптических кривых. Используется в Bitcoin и Ethereum.

**Преимущества ECC:**
- Меньший размер ключей при той же безопасности (256 бит ECC ≈ 3072 бит RSA)
- Быстрее операции подписания и верификации
- Меньше данных для хранения и передачи

```python
from cryptography.hazmat.primitives.asymmetric import ec
from cryptography.hazmat.primitives import hashes

# Генерация ключей на кривой secp256k1 (используется в Bitcoin)
private_key = ec.generate_private_key(ec.SECP256K1())
public_key = private_key.public_key()

print(f"Размер приватного ключа: {private_key.key_size} бит")
```

### Сравнение симметричного и асимметричного шифрования

| Характеристика | Симметричное | Асимметричное |
|----------------|--------------|---------------|
| Количество ключей | 1 | 2 (пара) |
| Скорость | Высокая | Низкая |
| Применение | Шифрование данных | Обмен ключами, подписи |
| Примеры | AES, ChaCha20 | RSA, ECC |

---

## Хеш-функции

### Что такое хеш-функция

**Хеш-функция** — это математическая функция, которая преобразует входные данные произвольной длины в выходную строку фиксированной длины (хеш, дайджест).

```
Входные данные (любой размер) → Хеш-функция → Хеш (фиксированный размер)
```

### Основные свойства криптографических хеш-функций

1. **Детерминированность** — одинаковые входные данные всегда дают одинаковый хеш
2. **Односторонность** — по хешу невозможно восстановить исходные данные
3. **Устойчивость к коллизиям** — практически невозможно найти два разных входа с одинаковым хешем
4. **Лавинный эффект** — малейшее изменение входа кардинально меняет хеш
5. **Быстрота вычисления** — хеш должен вычисляться эффективно

### SHA-256 (Bitcoin)

**SHA-256** (Secure Hash Algorithm 256-bit) — хеш-функция, используемая в Bitcoin.

- Выход: 256 бит (64 шестнадцатеричных символа)
- Используется для: хеширования блоков, Proof of Work, адресов

```python
import hashlib

def sha256(data: str) -> str:
    """Вычисление SHA-256 хеша"""
    return hashlib.sha256(data.encode()).hexdigest()

# Примеры
print(f"SHA-256('Hello'): {sha256('Hello')}")
print(f"SHA-256('hello'): {sha256('hello')}")  # Совершенно другой хеш!
print(f"SHA-256(''): {sha256('')}")

# Демонстрация лавинного эффекта
text1 = "Blockchain"
text2 = "blockchain"  # Только одна буква отличается
print(f"\nЛавинный эффект:")
print(f"'{text1}': {sha256(text1)}")
print(f"'{text2}': {sha256(text2)}")
```

**Двойное хеширование в Bitcoin:**

```python
def double_sha256(data: bytes) -> bytes:
    """Двойной SHA-256, используется в Bitcoin"""
    return hashlib.sha256(hashlib.sha256(data).digest()).digest()

# Bitcoin использует двойное хеширование для защиты от length extension атак
block_header = b"sample block header data"
block_hash = double_sha256(block_header)
print(f"Хеш блока: {block_hash.hex()}")
```

### Keccak-256 (Ethereum)

**Keccak-256** — хеш-функция, используемая в Ethereum (не путать со стандартизованным SHA-3).

```python
from Crypto.Hash import keccak

def keccak256(data: bytes) -> bytes:
    """Вычисление Keccak-256 хеша (как в Ethereum)"""
    k = keccak.new(digest_bits=256)
    k.update(data)
    return k.digest()

# Пример: вычисление адреса Ethereum из публичного ключа
def eth_address_from_pubkey(public_key_bytes: bytes) -> str:
    """
    Получение Ethereum адреса из публичного ключа.
    Адрес = последние 20 байт Keccak-256 хеша публичного ключа.
    """
    hash_bytes = keccak256(public_key_bytes)
    address = hash_bytes[-20:]  # Последние 20 байт
    return "0x" + address.hex()

# Демонстрация
sample_pubkey = bytes.fromhex("04" + "a" * 128)  # Условный публичный ключ
print(f"Ethereum адрес: {eth_address_from_pubkey(sample_pubkey)}")
```

### Сравнение SHA-256 и Keccak-256

| Характеристика | SHA-256 | Keccak-256 |
|----------------|---------|------------|
| Используется в | Bitcoin | Ethereum |
| Размер выхода | 256 бит | 256 бит |
| Архитектура | Merkle-Damgård | Sponge |
| Уязвимость к length extension | Да | Нет |

---

## Деревья Меркла (Merkle Trees)

### Структура дерева Меркла

**Дерево Меркла** — это бинарное дерево хешей, где:
- Листья содержат хеши данных (например, транзакций)
- Каждый родительский узел содержит хеш конкатенации своих потомков
- Корень дерева (Merkle Root) представляет все данные

```
                    Merkle Root
                   /            \
              Hash(H1+H2)    Hash(H3+H4)
              /      \        /      \
            H1       H2     H3       H4
            |        |      |        |
           Tx1      Tx2    Tx3      Tx4
```

### Реализация дерева Меркла

```python
import hashlib
from typing import List, Optional

def sha256_hash(data: bytes) -> bytes:
    """Вычисление SHA-256 хеша"""
    return hashlib.sha256(data).digest()

class MerkleTree:
    def __init__(self, transactions: List[bytes]):
        self.transactions = transactions
        self.tree = []
        self.root = self._build_tree()

    def _build_tree(self) -> bytes:
        """Построение дерева Меркла"""
        # Хешируем все транзакции (листья)
        current_level = [sha256_hash(tx) for tx in self.transactions]
        self.tree.append(current_level)

        # Строим дерево снизу вверх
        while len(current_level) > 1:
            next_level = []

            # Если нечетное количество, дублируем последний элемент
            if len(current_level) % 2 == 1:
                current_level.append(current_level[-1])

            # Объединяем пары и хешируем
            for i in range(0, len(current_level), 2):
                combined = current_level[i] + current_level[i + 1]
                parent_hash = sha256_hash(combined)
                next_level.append(parent_hash)

            self.tree.append(next_level)
            current_level = next_level

        return current_level[0] if current_level else b''

    def get_root(self) -> str:
        """Получение корня дерева в hex формате"""
        return self.root.hex()

    def get_proof(self, index: int) -> List[tuple]:
        """
        Получение доказательства Меркла для транзакции по индексу.
        Возвращает список пар (хеш, позиция), где позиция 'L' или 'R'.
        """
        proof = []
        current_index = index

        for level in self.tree[:-1]:  # Все уровни кроме корня
            # Определяем соседа
            if current_index % 2 == 0:
                sibling_index = current_index + 1
                position = 'R'  # Сосед справа
            else:
                sibling_index = current_index - 1
                position = 'L'  # Сосед слева

            if sibling_index < len(level):
                proof.append((level[sibling_index].hex(), position))

            current_index //= 2

        return proof

# Пример использования
transactions = [
    b"Alice -> Bob: 10 BTC",
    b"Bob -> Charlie: 5 BTC",
    b"Charlie -> Dave: 2 BTC",
    b"Dave -> Alice: 1 BTC"
]

tree = MerkleTree(transactions)
print(f"Merkle Root: {tree.get_root()}")

# Получаем доказательство для второй транзакции
proof = tree.get_proof(1)
print(f"\nДоказательство для транзакции 1:")
for hash_val, pos in proof:
    print(f"  {pos}: {hash_val[:16]}...")
```

### Преимущества деревьев Меркла

1. **Эффективная верификация** — проверка транзакции требует O(log n) хешей вместо O(n)
2. **Экономия места** — легкие клиенты хранят только корень и запрашивают доказательства
3. **Быстрое обнаружение изменений** — изменение одной транзакции меняет корень
4. **SPV (Simplified Payment Verification)** — легкие кошельки могут проверять транзакции без скачивания всего блокчейна

### Верификация с помощью доказательства Меркла

```python
def verify_merkle_proof(tx_hash: bytes, proof: List[tuple], root: bytes) -> bool:
    """
    Верификация транзакции с помощью доказательства Меркла.
    """
    current_hash = tx_hash

    for sibling_hash_hex, position in proof:
        sibling_hash = bytes.fromhex(sibling_hash_hex)

        if position == 'L':
            combined = sibling_hash + current_hash
        else:
            combined = current_hash + sibling_hash

        current_hash = sha256_hash(combined)

    return current_hash == root

# Верификация
tx_to_verify = transactions[1]
tx_hash = sha256_hash(tx_to_verify)
is_valid = verify_merkle_proof(tx_hash, proof, tree.root)
print(f"\nТранзакция валидна: {is_valid}")
```

---

## Цифровые подписи

### ECDSA (Elliptic Curve Digital Signature Algorithm)

**ECDSA** — алгоритм цифровой подписи на эллиптических кривых, используемый в Bitcoin и Ethereum.

**Компоненты:**
- **Приватный ключ** — случайное число (256 бит)
- **Публичный ключ** — точка на эллиптической кривой
- **Подпись** — пара чисел (r, s)

### Процесс подписания

1. Вычислить хеш сообщения: `z = hash(message)`
2. Сгенерировать случайное число `k`
3. Вычислить точку `R = k * G` (где G — генератор кривой)
4. `r = R.x mod n`
5. `s = k⁻¹ * (z + r * private_key) mod n`
6. Подпись: `(r, s)`

### Процесс верификации

1. Вычислить хеш сообщения: `z = hash(message)`
2. Вычислить `u1 = z * s⁻¹ mod n`
3. Вычислить `u2 = r * s⁻¹ mod n`
4. Вычислить точку `P = u1 * G + u2 * public_key`
5. Подпись валидна если `P.x == r`

### Реализация ECDSA

```python
from cryptography.hazmat.primitives.asymmetric import ec
from cryptography.hazmat.primitives import hashes
from cryptography.exceptions import InvalidSignature

class ECDSAWallet:
    def __init__(self):
        """Создание нового кошелька с ключевой парой"""
        self.private_key = ec.generate_private_key(ec.SECP256K1())
        self.public_key = self.private_key.public_key()

    def sign(self, message: bytes) -> bytes:
        """Подписание сообщения"""
        signature = self.private_key.sign(
            message,
            ec.ECDSA(hashes.SHA256())
        )
        return signature

    def verify(self, message: bytes, signature: bytes, public_key=None) -> bool:
        """Верификация подписи"""
        if public_key is None:
            public_key = self.public_key

        try:
            public_key.verify(
                signature,
                message,
                ec.ECDSA(hashes.SHA256())
            )
            return True
        except InvalidSignature:
            return False

    def get_public_key_bytes(self) -> bytes:
        """Получение публичного ключа в байтах"""
        from cryptography.hazmat.primitives.serialization import (
            Encoding, PublicFormat
        )
        return self.public_key.public_bytes(
            Encoding.X962,
            PublicFormat.UncompressedPoint
        )

# Пример использования
wallet = ECDSAWallet()

# Подписываем транзакцию
transaction = b"Send 1 BTC from Alice to Bob"
signature = wallet.sign(transaction)

print(f"Транзакция: {transaction.decode()}")
print(f"Подпись: {signature.hex()[:64]}...")
print(f"Длина подписи: {len(signature)} байт")

# Верифицируем подпись
is_valid = wallet.verify(transaction, signature)
print(f"Подпись валидна: {is_valid}")

# Попытка подделки
fake_transaction = b"Send 100 BTC from Alice to Bob"
is_fake_valid = wallet.verify(fake_transaction, signature)
print(f"Поддельная транзакция валидна: {is_fake_valid}")
```

### Применение в транзакциях блокчейна

```python
import json
import time

class Transaction:
    def __init__(self, sender_pubkey: bytes, recipient: str, amount: float):
        self.sender = sender_pubkey.hex()
        self.recipient = recipient
        self.amount = amount
        self.timestamp = time.time()
        self.signature = None

    def to_dict(self) -> dict:
        """Преобразование в словарь (без подписи)"""
        return {
            "sender": self.sender,
            "recipient": self.recipient,
            "amount": self.amount,
            "timestamp": self.timestamp
        }

    def get_hash(self) -> bytes:
        """Получение хеша транзакции для подписания"""
        tx_string = json.dumps(self.to_dict(), sort_keys=True)
        return hashlib.sha256(tx_string.encode()).digest()

    def sign(self, wallet: ECDSAWallet):
        """Подписание транзакции"""
        tx_hash = self.get_hash()
        self.signature = wallet.sign(tx_hash)

    def verify(self, public_key) -> bool:
        """Верификация подписи транзакции"""
        if self.signature is None:
            return False

        tx_hash = self.get_hash()
        wallet = ECDSAWallet()
        return wallet.verify(tx_hash, self.signature, public_key)

# Создаем кошелек и транзакцию
alice_wallet = ECDSAWallet()
bob_address = "0x742d35Cc6634C0532925a3b844Bc9e7595f8B2E0"

tx = Transaction(
    sender_pubkey=alice_wallet.get_public_key_bytes(),
    recipient=bob_address,
    amount=1.5
)

# Подписываем транзакцию
tx.sign(alice_wallet)

print(f"Транзакция создана:")
print(f"  От: {tx.sender[:32]}...")
print(f"  Кому: {tx.recipient}")
print(f"  Сумма: {tx.amount}")
print(f"  Подписана: {'Да' if tx.signature else 'Нет'}")

# Верификация
is_valid = tx.verify(alice_wallet.public_key)
print(f"  Верификация: {'Успешно' if is_valid else 'Ошибка'}")
```

---

## Итоги

### Ключевые концепции

| Концепция | Применение в блокчейне |
|-----------|------------------------|
| Симметричное шифрование | Шифрование приватных данных |
| Асимметричное шифрование | Генерация адресов, обмен ключами |
| Хеш-функции | Связывание блоков, Proof of Work, адреса |
| Деревья Меркла | Эффективная верификация транзакций |
| Цифровые подписи | Авторизация транзакций |

### Безопасность блокчейна

Криптография обеспечивает:
1. **Целостность** — изменение данных обнаруживается через хеши
2. **Аутентификацию** — цифровые подписи подтверждают владельца
3. **Неотказуемость** — подписанную транзакцию нельзя отрицать
4. **Конфиденциальность** — при необходимости данные могут быть зашифрованы

---

## Дополнительные ресурсы

- [Bitcoin Whitepaper](https://bitcoin.org/bitcoin.pdf)
- [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf)
- [Mastering Bitcoin - Chapter 4: Keys and Addresses](https://github.com/bitcoinbook/bitcoinbook)
- [Cryptography and Network Security by William Stallings](https://www.pearson.com/us/higher-education/program/Stallings-Cryptography-and-Network-Security-Principles-and-Practice-7th-Edition/PGM334788.html)
