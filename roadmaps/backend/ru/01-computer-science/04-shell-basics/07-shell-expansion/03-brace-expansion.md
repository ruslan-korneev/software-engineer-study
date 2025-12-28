# Brace Expansion (раскрытие фигурных скобок)

[prev: 02-tilde-expansion](./02-tilde-expansion.md) | [next: 04-parameter-expansion](./04-parameter-expansion.md)

---
## Что такое brace expansion?

**Brace expansion** — механизм shell, который генерирует произвольные строки из шаблона с фигурными скобками. В отличие от pathname expansion, не зависит от существующих файлов.

```bash
$ echo {a,b,c}
a b c

$ echo file{1,2,3}.txt
file1.txt file2.txt file3.txt
```

## Два типа brace expansion

### 1. Списки (через запятую)

```bash
$ echo {apple,banana,cherry}
apple banana cherry

$ echo {red,green,blue}-color
red-color green-color blue-color

$ echo pre-{one,two,three}-post
pre-one-post pre-two-post pre-three-post
```

### 2. Последовательности (через ..)

```bash
$ echo {1..5}
1 2 3 4 5

$ echo {a..e}
a b c d e

$ echo {A..E}
A B C D E
```

## Последовательности в деталях

### Числовые последовательности
```bash
$ echo {1..10}
1 2 3 4 5 6 7 8 9 10

$ echo {01..10}           # с ведущими нулями
01 02 03 04 05 06 07 08 09 10

$ echo {5..1}             # обратный порядок
5 4 3 2 1

$ echo {0..100..10}       # с шагом
0 10 20 30 40 50 60 70 80 90 100
```

### Буквенные последовательности
```bash
$ echo {a..z}
a b c d e f g h i j k l m n o p q r s t u v w x y z

$ echo {z..a}             # обратный порядок
z y x w v u t s r q p o n m l k j i h g f e d c b a

$ echo {a..z..2}          # с шагом
a c e g i k m o q s u w y
```

## Практические примеры

### Создание структуры директорий
```bash
$ mkdir -p project/{src,tests,docs,config}
# Создаёт:
# project/src/
# project/tests/
# project/docs/
# project/config/

$ mkdir -p app/{frontend,backend}/{src,tests,docs}
# Вложенные директории
```

### Создание нумерованных файлов
```bash
$ touch file{1..10}.txt
# file1.txt file2.txt ... file10.txt

$ touch chapter{01..12}.md
# chapter01.md chapter02.md ... chapter12.md
```

### Множественные расширения
```bash
$ touch {main,utils,helpers}.{js,ts,test.js}
# main.js main.ts main.test.js
# utils.js utils.ts utils.test.js
# helpers.js helpers.ts helpers.test.js
```

### Бэкап файла
```bash
$ cp config.yml{,.bak}
# Эквивалент: cp config.yml config.yml.bak

$ cp important.txt{,.$(date +%Y%m%d)}
# Копия с датой
```

### Переименование
```bash
$ mv file.txt{,.old}
# mv file.txt file.txt.old
```

## Комбинирование с другими раскрытиями

### С переменными
```bash
$ NAME="report"
$ touch ${NAME}-{2023,2024,2025}.pdf
# report-2023.pdf report-2024.pdf report-2025.pdf
```

### С путями
```bash
$ ls /var/log/{syslog,auth.log,kern.log}
```

### Вложенные скобки
```bash
$ echo {a,b{1,2,3},c}
a b1 b2 b3 c

$ mkdir -p {bin,lib/{x86,arm}}
```

## Порядок раскрытия

Brace expansion выполняется **первым**, до всех остальных раскрытий:

```bash
$ echo ${HOME}/{doc,img}
/home/user/doc /home/user/img

# Порядок:
# 1. Brace: ${HOME}/{doc,img} → ${HOME}/doc ${HOME}/img
# 2. Parameter: ${HOME}/doc → /home/user/doc
```

## Когда НЕ работает

### В кавычках
```bash
$ echo "{a,b,c}"
{a,b,c}                  # НЕ раскрывается

$ echo '{1..5}'
{1..5}                   # НЕ раскрывается
```

### Без запятой или ..
```bash
$ echo {abc}
{abc}                    # нужна запятая или ..
```

### С пробелами
```bash
$ echo {a, b, c}
{a, b, c}                # пробелы ломают синтаксис
```

## Отличия от pathname expansion

| Аспект | Brace Expansion | Pathname Expansion |
|--------|-----------------|-------------------|
| Зависит от файлов | Нет | Да |
| Генерирует | Произвольные строки | Существующие файлы |
| Порядок выполнения | Первым | Последним |
| Символы | `{}`, `,`, `..` | `*`, `?`, `[]` |

```bash
$ echo {a,b,c}.txt       # всегда a.txt b.txt c.txt
$ echo *.txt             # только существующие .txt файлы
```

## Продвинутые примеры

### Генерация IP-адресов
```bash
$ echo 192.168.1.{1..255}
$ echo 10.0.{0..3}.{1..254}
```

### Временные интервалы
```bash
$ mkdir logs-{2023..2025}-{01..12}
# logs-2023-01, logs-2023-02, ... logs-2025-12
```

### Тестовые данные
```bash
$ for i in user{1..100}; do echo "$i"; done
```

### Шаблон для загрузки
```bash
$ curl -O http://example.com/file{01..10}.zip
```

## Советы

1. **Используйте для создания структур** — быстрее, чем множество mkdir
2. **Проверяйте с echo** перед выполнением:
   ```bash
   $ echo mkdir -p project/{src,test,doc}
   mkdir -p project/src project/test project/doc
   ```
3. **Помните о порядке** — brace expansion происходит первым
4. **Ведущие нули** автоматически выравнивают числа: `{01..10}`
5. **Пустой элемент** полезен для бэкапов: `file{,.bak}`

---

[prev: 02-tilde-expansion](./02-tilde-expansion.md) | [next: 04-parameter-expansion](./04-parameter-expansion.md)
