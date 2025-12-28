# cat, sort, uniq - базовая обработка текста

[prev: 05-quantifiers](../06-regular-expressions/05-quantifiers.md) | [next: 02-cut-paste-join](./02-cut-paste-join.md)

---
## cat - объединение и вывод файлов

### Что такое cat?

**cat** (concatenate) - утилита для вывода содержимого файлов и их объединения. Одна из самых простых и часто используемых команд.

### Базовое использование

```bash
# Вывести содержимое файла
cat file.txt

# Вывести несколько файлов
cat file1.txt file2.txt

# Объединить файлы в один
cat file1.txt file2.txt > combined.txt

# Добавить в конец файла
cat new_content.txt >> existing.txt
```

### Полезные опции

```bash
# Нумерация строк (-n)
cat -n file.txt
#      1  First line
#      2  Second line
#      3  Third line

# Нумерация только непустых строк (-b)
cat -b file.txt

# Показать табуляции как ^I (-T)
cat -T file.txt

# Показать концы строк как $ (-E)
cat -E file.txt
# Line one$
# Line two$

# Сжать пустые строки (-s)
cat -s file.txt  # множественные пустые -> одна пустая

# Показать непечатные символы (-v)
cat -v file.txt

# Комбинация: показать всё (-A = -vET)
cat -A file.txt
```

### Создание файлов

```bash
# Создать файл из stdin
cat > newfile.txt
Type content here
Press Ctrl+D to save

# Heredoc
cat > config.txt << EOF
line 1
line 2
line 3
EOF

# Добавить в файл
cat >> file.txt << EOF
additional content
EOF
```

### tac - cat наоборот

```bash
# Вывести файл в обратном порядке строк
tac file.txt
# Last line
# Middle line
# First line

# Полезно для логов
tac /var/log/syslog | head -20
```

## sort - сортировка строк

### Базовое использование

```bash
# Сортировка по алфавиту
sort file.txt

# Сохранить результат
sort file.txt > sorted.txt

# Сортировать несколько файлов
sort file1.txt file2.txt > sorted.txt

# Сортировать "на месте"
sort -o file.txt file.txt
```

### Порядок сортировки

```bash
# Обратный порядок (-r)
sort -r file.txt

# Числовая сортировка (-n)
sort -n numbers.txt
# Без -n: 1, 10, 2, 20, 3
# С -n:   1, 2, 3, 10, 20

# Человекочитаемые числа (-h)
sort -h sizes.txt
# 1K, 10K, 1M, 10M, 1G

# Игнорировать регистр (-f)
sort -f names.txt

# Сортировать по месяцам (-M)
sort -M months.txt
# Jan, Feb, Mar, Apr...
```

### Сортировка по полям

```bash
# Данные разделённые пробелами
# name score grade
# John 85 B
# Alice 92 A
# Bob 78 C

# Сортировать по 2-му полю (числовой)
sort -k2 -n data.txt

# По 2-му полю в обратном порядке
sort -k2 -nr data.txt

# Указать разделитель (-t)
# CSV файл: name,score,grade
sort -t',' -k2 -n data.csv

# Диапазон полей
sort -k2,2 -n data.txt    # только 2-е поле
sort -k1,2 data.txt       # поля 1-2
```

### Уникальность

```bash
# Удалить дубликаты (-u)
sort -u file.txt

# Проверить отсортирован ли файл (-c)
sort -c file.txt
# sort: file.txt:3: disorder: line

# Стабильная сортировка (-s)
sort -s -k1 data.txt  # сохранить порядок равных элементов
```

### Случайный порядок

```bash
# Перемешать строки (-R)
sort -R file.txt

# Альтернатива
shuf file.txt
```

### Практические примеры

```bash
# Топ 10 самых больших файлов
du -sh * | sort -rh | head -10

# Сортировка IP адресов
sort -t. -k1,1n -k2,2n -k3,3n -k4,4n ips.txt

# Сортировка по дате (ISO формат)
sort -k1 dates.txt

# Уникальные значения в колонке
cut -d',' -f2 data.csv | sort -u
```

## uniq - удаление дубликатов

### Важно!

**uniq работает только с отсортированными данными!** Дубликаты должны идти подряд.

```bash
# НЕПРАВИЛЬНО - данные не отсортированы
echo -e "a\nb\na" | uniq
# a
# b
# a  <- дубликат не удалён!

# ПРАВИЛЬНО - сначала sort
echo -e "a\nb\na" | sort | uniq
# a
# b
```

### Базовое использование

```bash
# Удалить повторяющиеся строки
sort file.txt | uniq

# То же самое что sort -u
sort -u file.txt
```

### Подсчёт и фильтрация

```bash
# Подсчитать количество повторений (-c)
sort file.txt | uniq -c
#       3 apple
#       1 banana
#       2 cherry

# Отсортировать по частоте
sort file.txt | uniq -c | sort -rn
#       3 apple
#       2 cherry
#       1 banana

# Только дубликаты (-d)
sort file.txt | uniq -d
# apple
# cherry

# Только уникальные (-u)
sort file.txt | uniq -u
# banana

# Все строки кроме первого вхождения (-D)
sort file.txt | uniq -D
```

### Игнорирование при сравнении

```bash
# Игнорировать регистр (-i)
sort -f file.txt | uniq -i

# Пропустить первые N полей (-f)
uniq -f 1 data.txt    # игнорировать первое поле

# Пропустить первые N символов (-s)
uniq -s 5 data.txt    # игнорировать первые 5 символов

# Сравнивать только N символов (-w)
uniq -w 10 data.txt   # сравнивать только первые 10
```

### Практические примеры

```bash
# Подсчёт уникальных IP в логе
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# Найти дублирующиеся строки
sort file.txt | uniq -d

# Уникальные ошибки
grep ERROR app.log | sort | uniq -c | sort -rn

# Статистика по кодам ответа
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# Найти файлы-дубликаты по размеру
find . -type f -exec du -b {} \; | sort | uniq -d -w 10
```

## Комбинирование команд

### Типичные пайплайны

```bash
# Частота слов в файле
cat file.txt | tr ' ' '\n' | sort | uniq -c | sort -rn | head -20

# Уникальные email адреса
grep -Eo '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+' file.txt | sort -u

# Статистика логов
cat access.log | cut -d' ' -f1 | sort | uniq -c | sort -rn

# Общие строки в двух файлах
sort file1.txt file2.txt | uniq -d

# Строки только в file1
sort file1.txt file2.txt file2.txt | uniq -u

# Строки только в file2
sort file1.txt file1.txt file2.txt | uniq -u
```

### head и tail

```bash
# Первые 10 строк
head file.txt
head -n 10 file.txt
head -10 file.txt

# Последние 10 строк
tail file.txt
tail -n 10 file.txt

# Все кроме первых 10
tail -n +11 file.txt

# Все кроме последних 10
head -n -10 file.txt

# Следить за файлом
tail -f log.txt

# Строки 20-30
head -30 file.txt | tail -11
sed -n '20,30p' file.txt
```

## Резюме команд

| Команда | Описание |
|---------|----------|
| `cat file` | Вывести файл |
| `cat -n file` | С номерами строк |
| `cat -A file` | Показать спецсимволы |
| `tac file` | В обратном порядке |
| `sort file` | Сортировать |
| `sort -n file` | Числовая сортировка |
| `sort -r file` | Обратный порядок |
| `sort -k2 file` | По 2-му полю |
| `sort -u file` | Уникальные |
| `uniq` | Удалить дубликаты |
| `uniq -c` | Подсчитать дубликаты |
| `uniq -d` | Только дубликаты |
| `uniq -u` | Только уникальные |
| `head -n 10` | Первые 10 строк |
| `tail -n 10` | Последние 10 строк |
| `tail -f` | Следить за файлом |

---

[prev: 05-quantifiers](../06-regular-expressions/05-quantifiers.md) | [next: 02-cut-paste-join](./02-cut-paste-join.md)
