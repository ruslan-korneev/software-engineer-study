# Кодирование Хаффмана (Huffman Coding)

[prev: 02-activity-selection](./02-activity-selection.md) | [next: 01-dp-concept](../08-dynamic-programming/01-dp-concept.md)
---

## Определение

**Кодирование Хаффмана** — жадный алгоритм для сжатия данных без потерь. Создаёт оптимальный префиксный код, где часто встречающиеся символы кодируются короткими последовательностями битов, а редкие — длинными.

**Префиксный код** — код, в котором ни одно кодовое слово не является префиксом другого, что позволяет однозначно декодировать сообщение.

## Зачем нужен

### Области применения:
- **Сжатие файлов** — ZIP, GZIP, BZIP2
- **Сжатие изображений** — JPEG, PNG
- **Сжатие видео** — MPEG
- **Передача данных** — минимизация трафика
- **Архиваторы** — часть алгоритмов сжатия

### Преимущества:
- Оптимальный среди префиксных кодов
- Без потерь (lossless)
- Простота реализации

## Как работает

### Идея алгоритма

```
1. Подсчитать частоту каждого символа
2. Создать листовые узлы для каждого символа
3. Повторять, пока не останется один узел:
   a) Извлечь два узла с минимальной частотой
   b) Создать новый узел с суммой частот
   c) Сделать извлечённые узлы детьми нового
4. Присвоить 0 левым рёбрам, 1 правым
```

### Визуализация построения

```
Символы и частоты: a=5, b=9, c=12, d=13, e=16, f=45

Шаг 1: Берём два минимальных (a=5, b=9)
        [14]
       /    \
     a:5    b:9

Шаг 2: Берём [14] и c=12
        [26]
       /    \
    c:12   [14]
          /    \
        a:5    b:9

Шаг 3: Берём d=13 и e=16
        [29]
       /    \
    d:13   e:16

Шаг 4: Берём [26] и [29]
           [55]
          /    \
       [26]    [29]
       /  \    /   \
    c:12 [14] d:13 e:16
         /  \
       a:5  b:9

Шаг 5: Берём f=45 и [55]
              [100]
             /     \
          f:45    [55]
                 /    \
              [26]    [29]
              /  \    /   \
           c:12 [14] d:13 e:16
                /  \
              a:5  b:9

Коды:
f = 0          (1 бит)
c = 100        (3 бита)
d = 101        (3 бита)
a = 1100       (4 бита)
b = 1101       (4 бита)
e = 111        (3 бита)
```

### Сравнение с фиксированным кодом

```
Символы: a, b, c, d, e, f (6 символов)
Фиксированный код: 3 бита на символ (log₂6 ≈ 2.58, округляем)

Сообщение: "aaabbbcccccddddddeeeeeeefffffffffffffffffffff"
(a=3, b=3, c=5, d=6, e=7, f=21 = 45 символов)

Фиксированный: 45 × 3 = 135 бит
Хаффман: 3×4 + 3×4 + 5×3 + 6×3 + 7×3 + 21×1 = 12 + 12 + 15 + 18 + 21 + 21 = 99 бит

Экономия: (135 - 99) / 135 = 26.7%
```

## Псевдокод

```
function buildHuffmanTree(frequencies):
    # Создаём min-heap с листовыми узлами
    heap = min_heap()
    for each (symbol, freq) in frequencies:
        heap.push(Node(symbol, freq))

    # Строим дерево
    while heap.size() > 1:
        left = heap.pop()
        right = heap.pop()
        merged = Node(null, left.freq + right.freq)
        merged.left = left
        merged.right = right
        heap.push(merged)

    return heap.pop()  # Корень дерева

function generateCodes(root):
    codes = {}

    function traverse(node, code):
        if node is leaf:
            codes[node.symbol] = code
            return

        traverse(node.left, code + "0")
        traverse(node.right, code + "1")

    traverse(root, "")
    return codes
```

### Реализация на Python

```python
import heapq
from collections import Counter

class HuffmanNode:
    def __init__(self, char, freq):
        self.char = char
        self.freq = freq
        self.left = None
        self.right = None

    def __lt__(self, other):
        return self.freq < other.freq


def build_huffman_tree(text):
    """Строит дерево Хаффмана для текста."""
    # Подсчёт частот
    frequency = Counter(text)

    # Создаём min-heap
    heap = [HuffmanNode(char, freq) for char, freq in frequency.items()]
    heapq.heapify(heap)

    # Строим дерево
    while len(heap) > 1:
        left = heapq.heappop(heap)
        right = heapq.heappop(heap)

        merged = HuffmanNode(None, left.freq + right.freq)
        merged.left = left
        merged.right = right

        heapq.heappush(heap, merged)

    return heap[0] if heap else None


def generate_codes(root):
    """Генерирует коды Хаффмана."""
    codes = {}

    def traverse(node, code):
        if node is None:
            return

        if node.char is not None:
            codes[node.char] = code if code else '0'  # Для одного символа
            return

        traverse(node.left, code + '0')
        traverse(node.right, code + '1')

    traverse(root, '')
    return codes


def huffman_encode(text):
    """Кодирует текст алгоритмом Хаффмана."""
    if not text:
        return '', {}, None

    root = build_huffman_tree(text)
    codes = generate_codes(root)

    encoded = ''.join(codes[char] for char in text)

    return encoded, codes, root


def huffman_decode(encoded, root):
    """Декодирует текст."""
    if not encoded or root is None:
        return ''

    decoded = []
    node = root

    for bit in encoded:
        if bit == '0':
            node = node.left
        else:
            node = node.right

        if node.char is not None:
            decoded.append(node.char)
            node = root

    return ''.join(decoded)


# Пример использования
text = "abracadabra"
encoded, codes, root = huffman_encode(text)

print(f"Исходный текст: {text}")
print(f"Коды: {codes}")
print(f"Закодировано: {encoded}")
print(f"Исходный размер: {len(text) * 8} бит")
print(f"Сжатый размер: {len(encoded)} бит")
print(f"Коэффициент сжатия: {len(encoded) / (len(text) * 8):.2%}")

decoded = huffman_decode(encoded, root)
print(f"Декодировано: {decoded}")
print(f"Совпадает: {text == decoded}")
```

### Канонический код Хаффмана

```python
def canonical_huffman(text):
    """
    Канонический код Хаффмана.
    Преимущество: нужно хранить только длины кодов.
    """
    root = build_huffman_tree(text)
    codes = generate_codes(root)

    # Получаем длины кодов
    code_lengths = {char: len(code) for char, code in codes.items()}

    # Сортируем по длине, затем по символу
    sorted_chars = sorted(code_lengths.items(), key=lambda x: (x[1], x[0]))

    # Генерируем канонические коды
    canonical = {}
    code = 0
    prev_length = 0

    for char, length in sorted_chars:
        code <<= (length - prev_length)
        canonical[char] = format(code, f'0{length}b')
        code += 1
        prev_length = length

    return canonical


text = "abracadabra"
codes = canonical_huffman(text)
print("Канонические коды:", codes)
```

### Адаптивное кодирование Хаффмана

```python
class AdaptiveHuffman:
    """
    Адаптивный Хаффман: обновляет дерево по мере чтения.
    Не требует предварительного знания частот.
    """

    def __init__(self):
        self.tree = None
        self.codes = {}
        self.frequencies = Counter()

    def encode_symbol(self, symbol):
        """Кодирует один символ и обновляет дерево."""
        self.frequencies[symbol] += 1

        if symbol in self.codes:
            result = self.codes[symbol]
        else:
            # Новый символ — передаём как есть
            result = format(ord(symbol), '08b')

        # Перестраиваем дерево
        self.tree = build_huffman_tree_from_freq(self.frequencies)
        self.codes = generate_codes(self.tree)

        return result
```

## Анализ сложности

| Операция | Сложность |
|----------|-----------|
| Подсчёт частот | O(n) |
| Построение дерева | O(k log k), k = уникальных символов |
| Генерация кодов | O(k) |
| Кодирование | O(n) |
| Декодирование | O(n) |
| **Общая** | **O(n + k log k)** |

Пространство: O(k) для дерева и кодов

## Типичные ошибки и Edge Cases

### 1. Один уникальный символ

```python
def huffman_encode_safe(text):
    if len(set(text)) == 1:
        # Один символ — код '0' или фиксированный
        return '0' * len(text), {text[0]: '0'}, None
    # ...
```

### 2. Пустой текст

```python
def huffman_encode_safe(text):
    if not text:
        return '', {}, None
    # ...
```

### 3. Неправильное сравнение узлов

```python
# ОШИБКА: без __lt__ heapq не работает с объектами
class WrongNode:
    def __init__(self, char, freq):
        self.char = char
        self.freq = freq

# ПРАВИЛЬНО
class CorrectNode:
    def __init__(self, char, freq):
        self.char = char
        self.freq = freq

    def __lt__(self, other):
        return self.freq < other.freq
```

### 4. Сериализация дерева

```python
def serialize_tree(root):
    """Сериализует дерево для хранения."""
    if root is None:
        return ''

    if root.char is not None:
        return '1' + root.char

    return '0' + serialize_tree(root.left) + serialize_tree(root.right)


def deserialize_tree(data, index=[0]):
    """Восстанавливает дерево."""
    if index[0] >= len(data):
        return None

    if data[index[0]] == '1':
        index[0] += 1
        node = HuffmanNode(data[index[0]], 0)
        index[0] += 1
        return node

    index[0] += 1
    node = HuffmanNode(None, 0)
    node.left = deserialize_tree(data, index)
    node.right = deserialize_tree(data, index)
    return node
```

### 5. Сжатие в байты

```python
def bits_to_bytes(bits):
    """Конвертирует строку битов в байты."""
    # Добавляем padding
    padding = 8 - len(bits) % 8
    bits += '0' * padding

    result = bytearray()
    result.append(padding)  # Первый байт — размер padding

    for i in range(0, len(bits), 8):
        byte = int(bits[i:i+8], 2)
        result.append(byte)

    return bytes(result)


def bytes_to_bits(data):
    """Конвертирует байты обратно в биты."""
    padding = data[0]
    bits = ''.join(format(byte, '08b') for byte in data[1:])
    return bits[:-padding] if padding else bits
```

### Рекомендации:
1. Хаффман оптимален для известного распределения частот
2. Для неизвестного распределения используйте адаптивный Хаффман
3. Канонический код проще хранить и передавать
4. Для практического сжатия комбинируйте с LZ77/LZ78 (DEFLATE)
5. Проверяйте edge cases: один символ, пустой текст, Unicode

---
[prev: 02-activity-selection](./02-activity-selection.md) | [next: 01-dp-concept](../08-dynamic-programming/01-dp-concept.md)