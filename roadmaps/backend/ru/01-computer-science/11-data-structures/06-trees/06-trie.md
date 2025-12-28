# Префиксное дерево (Trie)

[prev: 05-heap](./05-heap.md) | [next: 01-graph-basics](../07-graphs/01-graph-basics.md)
---

## Определение

**Trie (префиксное дерево, бор)** — это древовидная структура данных для хранения строк, где каждый узел представляет один символ, а путь от корня до узла формирует префикс или полную строку.

```
Хранение слов: "cat", "car", "card", "care", "dog"

              root
             /    \
            c      d
           /        \
          a          o
         / \          \
        t   r          g*
            |\
            d* e*

* = конец слова
```

## Зачем нужно

### Практическое применение:
- **Автодополнение** — поиск слов по префиксу
- **Проверка орфографии** — словари
- **IP-маршрутизация** — longest prefix match
- **T9 словарь** — ввод текста на телефоне
- **Поисковые подсказки** — Google, YouTube
- **Решение задач на строки** — подсчёт уникальных префиксов

## Структура

```python
class TrieNode:
    def __init__(self):
        self.children = {}  # символ -> TrieNode
        self.is_end = False  # является ли концом слова

class Trie:
    def __init__(self):
        self.root = TrieNode()
```

### Визуализация структуры

```
Слова: ["apple", "app", "april"]

        root
          |
          a
          |
          p
         / \
        p   r
       /     \
      l       i
     /         \
    e*          l*

children = {
    'a': TrieNode {
        children = {
            'p': TrieNode {
                children = {
                    'p': TrieNode {
                        is_end = True,  # "app"
                        children = {
                            'l': TrieNode {
                                children = {
                                    'e': TrieNode {
                                        is_end = True  # "apple"
                                    }
                                }
                            }
                        }
                    },
                    'r': ...  # "april"
                }
            }
        }
    }
}
```

## Основные операции

### Вставка (Insert)

```python
def insert(self, word):
    """
    Вставка слова в trie.
    Время: O(m), где m — длина слова
    """
    node = self.root

    for char in word:
        if char not in node.children:
            node.children[char] = TrieNode()
        node = node.children[char]

    node.is_end = True
```

```
insert("car"):

root → 'c' (создать) → 'a' (создать) → 'r' (создать, is_end=True)
```

### Поиск (Search)

```python
def search(self, word):
    """
    Проверка наличия слова.
    Время: O(m)
    """
    node = self.root

    for char in word:
        if char not in node.children:
            return False
        node = node.children[char]

    return node.is_end  # важно проверить is_end!
```

```
search("car") → True
search("ca")  → False (is_end = False)
search("cat") → False (нет такого пути)
```

### Проверка префикса (Starts With)

```python
def starts_with(self, prefix):
    """
    Проверка наличия слов с данным префиксом.
    Время: O(m)
    """
    node = self.root

    for char in prefix:
        if char not in node.children:
            return False
        node = node.children[char]

    return True  # не проверяем is_end!
```

### Поиск слов по префиксу

```python
def find_words_with_prefix(self, prefix):
    """
    Находит все слова с данным префиксом.
    """
    node = self.root

    # Находим узел, соответствующий префиксу
    for char in prefix:
        if char not in node.children:
            return []
        node = node.children[char]

    # DFS для сбора всех слов
    words = []
    self._collect_words(node, prefix, words)
    return words

def _collect_words(self, node, current, words):
    if node.is_end:
        words.append(current)

    for char, child in node.children.items():
        self._collect_words(child, current + char, words)
```

### Удаление (Delete)

```python
def delete(self, word):
    """
    Удаление слова из trie.
    Время: O(m)
    """
    def _delete(node, word, depth):
        if depth == len(word):
            if not node.is_end:
                return False  # слово не найдено
            node.is_end = False
            return len(node.children) == 0  # можно ли удалить узел

        char = word[depth]
        if char not in node.children:
            return False

        should_delete = _delete(node.children[char], word, depth + 1)

        if should_delete:
            del node.children[char]
            return len(node.children) == 0 and not node.is_end

        return False

    _delete(self.root, word, 0)
```

## Полная реализация

```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end = False
        self.count = 0  # опционально: количество слов с этим префиксом

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word):
        node = self.root
        for char in word:
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]
            node.count += 1
        node.is_end = True

    def search(self, word):
        node = self._find_node(word)
        return node is not None and node.is_end

    def starts_with(self, prefix):
        return self._find_node(prefix) is not None

    def count_prefix(self, prefix):
        node = self._find_node(prefix)
        return node.count if node else 0

    def _find_node(self, prefix):
        node = self.root
        for char in prefix:
            if char not in node.children:
                return None
            node = node.children[char]
        return node

    def get_all_words(self):
        words = []
        self._collect_words(self.root, "", words)
        return words

    def _collect_words(self, node, current, words):
        if node.is_end:
            words.append(current)
        for char, child in node.children.items():
            self._collect_words(child, current + char, words)

    def autocomplete(self, prefix, limit=10):
        """Автодополнение с ограничением количества"""
        node = self._find_node(prefix)
        if node is None:
            return []

        words = []
        self._collect_words_limited(node, prefix, words, limit)
        return words

    def _collect_words_limited(self, node, current, words, limit):
        if len(words) >= limit:
            return

        if node.is_end:
            words.append(current)

        for char in sorted(node.children.keys()):  # лексикографический порядок
            if len(words) >= limit:
                break
            self._collect_words_limited(
                node.children[char],
                current + char,
                words,
                limit
            )
```

## Анализ сложности

| Операция | Время | Память |
|----------|-------|--------|
| Insert | O(m) | O(m) |
| Search | O(m) | O(1) |
| Starts With | O(m) | O(1) |
| Delete | O(m) | O(1) |
| Autocomplete | O(m + k) | O(k) |

Где m — длина слова, k — количество результатов

### Память

```
Худший случай: O(ALPHABET × m × n)
- ALPHABET = размер алфавита (26 для английского)
- m = средняя длина слова
- n = количество слов

На практике меньше благодаря общим префиксам.
```

## Примеры с разбором

### Пример 1: Автодополнение

```python
class Autocomplete:
    def __init__(self, words):
        self.trie = Trie()
        for word in words:
            self.trie.insert(word.lower())

    def suggest(self, prefix, max_results=5):
        prefix = prefix.lower()
        return self.trie.autocomplete(prefix, max_results)

# Использование:
ac = Autocomplete([
    "apple", "application", "apply", "apt",
    "banana", "band", "bandana"
])

ac.suggest("app")  # ["apple", "application", "apply"]
ac.suggest("ban")  # ["banana", "band", "bandana"]
ac.suggest("xyz")  # []
```

### Пример 2: Словарь с проверкой орфографии

```python
class SpellChecker:
    def __init__(self, dictionary):
        self.trie = Trie()
        for word in dictionary:
            self.trie.insert(word.lower())

    def is_correct(self, word):
        return self.trie.search(word.lower())

    def suggest_corrections(self, word, max_distance=1):
        """Находит похожие слова (расстояние Левенштейна ≤ max_distance)"""
        suggestions = []
        self._find_similar(self.trie.root, "", word.lower(), max_distance, suggestions)
        return suggestions

    def _find_similar(self, node, current, target, remaining, suggestions):
        if remaining < 0:
            return

        if node.is_end:
            distance = self._levenshtein(current, target)
            if distance <= remaining:
                suggestions.append((current, distance))

        for char, child in node.children.items():
            self._find_similar(child, current + char, target, remaining, suggestions)

    @staticmethod
    def _levenshtein(s1, s2):
        # Упрощённая реализация
        if len(s1) < len(s2):
            return SpellChecker._levenshtein(s2, s1)
        if len(s2) == 0:
            return len(s1)

        prev_row = range(len(s2) + 1)
        for i, c1 in enumerate(s1):
            curr_row = [i + 1]
            for j, c2 in enumerate(s2):
                insertions = prev_row[j + 1] + 1
                deletions = curr_row[j] + 1
                substitutions = prev_row[j] + (c1 != c2)
                curr_row.append(min(insertions, deletions, substitutions))
            prev_row = curr_row

        return prev_row[-1]
```

### Пример 3: Подсчёт уникальных префиксов

```python
def count_unique_prefixes(words):
    """
    Находит минимальный уникальный префикс для каждого слова.

    ["zebra", "dog", "duck", "dove"] →
    {"zebra": "z", "dog": "dog", "duck": "du", "dove": "dov"}
    """
    trie = Trie()
    for word in words:
        trie.insert(word)

    result = {}
    for word in words:
        node = trie.root
        prefix = ""

        for char in word:
            prefix += char
            node = node.children[char]

            # Если только одно слово проходит через этот узел
            if node.count == 1:
                break

        result[word] = prefix

    return result
```

### Пример 4: Поиск слов в матрице (Word Search II)

```python
def find_words(board, words):
    """
    Находит все слова из списка в матрице букв.
    Слова могут идти в любом направлении (вверх, вниз, влево, вправо).
    """
    # Строим trie из слов
    trie = Trie()
    for word in words:
        trie.insert(word)

    result = set()
    rows, cols = len(board), len(board[0])

    def dfs(row, col, node, path):
        if node.is_end:
            result.add(path)

        if row < 0 or row >= rows or col < 0 or col >= cols:
            return

        char = board[row][col]
        if char not in node.children:
            return

        board[row][col] = '#'  # помечаем как посещённую

        for dr, dc in [(-1, 0), (1, 0), (0, -1), (0, 1)]:
            dfs(row + dr, col + dc, node.children[char], path + char)

        board[row][col] = char  # восстанавливаем

    for r in range(rows):
        for c in range(cols):
            dfs(r, c, trie.root, "")

    return list(result)
```

### Пример 5: T9 словарь

```python
class T9Dictionary:
    """
    T9 — система ввода текста на кнопочных телефонах.
    2 = abc, 3 = def, 4 = ghi, 5 = jkl, 6 = mno, 7 = pqrs, 8 = tuv, 9 = wxyz
    """
    KEYPAD = {
        '2': 'abc', '3': 'def', '4': 'ghi', '5': 'jkl',
        '6': 'mno', '7': 'pqrs', '8': 'tuv', '9': 'wxyz'
    }

    def __init__(self, words):
        self.trie = Trie()
        for word in words:
            self.trie.insert(word.lower())

    def predict(self, digits):
        """
        Возвращает все слова, соответствующие нажатым кнопкам.
        """
        if not digits:
            return []

        result = []
        self._search(self.trie.root, digits, 0, "", result)
        return result

    def _search(self, node, digits, index, current, result):
        if index == len(digits):
            if node.is_end:
                result.append(current)
            return

        digit = digits[index]
        if digit not in self.KEYPAD:
            return

        for char in self.KEYPAD[digit]:
            if char in node.children:
                self._search(
                    node.children[char],
                    digits,
                    index + 1,
                    current + char,
                    result
                )

# Использование:
t9 = T9Dictionary(["good", "home", "gone", "hood"])
t9.predict("4663")  # ["good", "gone", "home", "hood"]
```

## Оптимизации

### Сжатое Trie (Radix Tree)

```
Обычное Trie:              Radix Tree:
     root                      root
      |                         |
      r                       romane
     /                         / \
    o                      ulus    n
   /                              / \
  m                              e   us
 / \
a   u
|   |
n   l
|   |
e   u
    |
    s

Экономия памяти при общих длинных префиксах.
```

### Массив вместо хэш-таблицы

```python
class TrieNodeArray:
    def __init__(self):
        self.children = [None] * 26  # для английских букв
        self.is_end = False

    def _index(self, char):
        return ord(char) - ord('a')
```

## Типичные ошибки

### 1. Забытая проверка is_end

```python
# НЕПРАВИЛЬНО
def search(self, word):
    node = self._find_node(word)
    return node is not None  # "car" найдено, если есть "card"!

# ПРАВИЛЬНО
def search(self, word):
    node = self._find_node(word)
    return node is not None and node.is_end
```

### 2. Изменение слова во время итерации

```python
# НЕПРАВИЛЬНО
for word in words:
    word = word.lower()  # не изменяет оригинал в words
    trie.insert(word)

# ПРАВИЛЬНО
for word in words:
    trie.insert(word.lower())
```

### 3. Неправильное удаление

```python
# НЕПРАВИЛЬНО — просто убираем флаг
def delete_wrong(self, word):
    node = self._find_node(word)
    if node:
        node.is_end = False
    # Узлы остаются, даже если не нужны!

# ПРАВИЛЬНО — удаляем ненужные узлы рекурсивно
```

## Сравнение с другими структурами

| Операция | Trie | Hash Table | BST |
|----------|------|------------|-----|
| Insert | O(m) | O(m)* | O(m log n) |
| Search | O(m) | O(m) | O(m log n) |
| Prefix search | O(m + k) | O(n) | O(m log n + k) |
| Memory | O(Σ × m × n) | O(m × n) | O(m × n) |

*m — длина слова, n — количество слов, k — количество результатов, Σ — размер алфавита

## Когда использовать Trie

**Используй Trie, когда:**
- Нужен поиск по префиксу
- Автодополнение
- Много общих префиксов
- Словарь с проверкой орфографии

**НЕ используй Trie, когда:**
- Строки случайные, нет общих префиксов
- Важна экономия памяти
- Только точный поиск (хэш-таблица лучше)

---
[prev: 05-heap](./05-heap.md) | [next: 01-graph-basics](../07-graphs/01-graph-basics.md)