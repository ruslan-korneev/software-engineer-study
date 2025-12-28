# cut, paste, join - работа с колонками

[prev: 01-cat-sort-uniq](./01-cat-sort-uniq.md) | [next: 03-diff-patch](./03-diff-patch.md)

---
## cut - извлечение колонок

### Что такое cut?

**cut** извлекает части строк из файла или stdin. Работает с символами, байтами или полями (колонками).

### Извлечение по символам (-c)

```bash
# Первые 5 символов каждой строки
cut -c1-5 file.txt

# Символы с 5 по 10
cut -c5-10 file.txt

# С 5-го символа до конца
cut -c5- file.txt

# До 5-го символа
cut -c-5 file.txt

# Конкретные позиции
cut -c1,3,5 file.txt

# Несколько диапазонов
cut -c1-3,5-7,10- file.txt
```

### Извлечение по байтам (-b)

```bash
# Аналогично -c, но работает с байтами
# Важно для многобайтовых кодировок (UTF-8)
cut -b1-5 file.txt

# В UTF-8 кириллический символ = 2 байта!
echo "Привет" | cut -b1-2   # один символ П
echo "Привет" | cut -c1     # тоже П
```

### Извлечение полей (-f)

```bash
# Разделитель по умолчанию - TAB

# Первое поле
cut -f1 data.txt

# Поля 1 и 3
cut -f1,3 data.txt

# Поля 2-4
cut -f2-4 data.txt

# Все кроме поля 2
cut -f2 --complement data.txt
```

### Указание разделителя (-d)

```bash
# CSV файл (разделитель - запятая)
cut -d',' -f1,3 data.csv

# Файл с разделителем :
cut -d':' -f1,3 /etc/passwd

# Пробел как разделитель
cut -d' ' -f2 file.txt

# Разделитель |
cut -d'|' -f1-3 file.txt
```

### Изменение разделителя вывода (--output-delimiter)

```bash
# Заменить TAB на запятую в выводе
cut -f1,3 --output-delimiter=',' data.txt

# CSV в TSV
cut -d',' -f1-3 --output-delimiter=$'\t' data.csv
```

### Практические примеры

```bash
# Имена пользователей из /etc/passwd
cut -d':' -f1 /etc/passwd

# Домашние директории
cut -d':' -f1,6 /etc/passwd

# Извлечь IP из лога
cut -d' ' -f1 access.log

# Первая колонка CSV
cut -d',' -f1 data.csv

# Все колонки кроме первой
cut -d',' -f2- data.csv
```

## paste - объединение файлов

### Что такое paste?

**paste** объединяет файлы построчно, соединяя соответствующие строки разделителем.

### Базовое использование

```bash
# Объединить два файла
paste file1.txt file2.txt
# line1_file1    line1_file2
# line2_file1    line2_file2

# Объединить три файла
paste file1.txt file2.txt file3.txt
```

### Разделитель (-d)

```bash
# Запятая вместо TAB
paste -d',' file1.txt file2.txt

# Разные разделители для разных файлов
paste -d',;' file1.txt file2.txt file3.txt
# file1,file2;file3

# Без разделителя
paste -d'' file1.txt file2.txt
```

### Последовательное объединение (-s)

```bash
# Все строки файла в одну
paste -s file.txt
# line1    line2    line3    line4

# С разделителем
paste -s -d',' file.txt
# line1,line2,line3,line4

# Группировка по 2 строки
paste - - < file.txt
# line1    line2
# line3    line4

# По 3 строки
paste - - - < file.txt
```

### Практические примеры

```bash
# Создать CSV из нескольких файлов
paste -d',' names.txt ages.txt emails.txt > data.csv

# Нумерация строк
nl -ba file.txt | cut -f1 | paste - file.txt

# Транспонировать строку в столбец
echo "a b c d" | tr ' ' '\n' | paste -s
# a    b    c    d

# Объединить все строки через пробел
paste -s -d' ' file.txt
```

## join - объединение по ключу

### Что такое join?

**join** объединяет строки двух файлов по общему полю (как SQL JOIN). Файлы должны быть отсортированы по полю соединения!

### Базовое использование

```bash
# file1.txt:        file2.txt:
# 1 Alice           1 Engineer
# 2 Bob             2 Designer
# 3 Charlie         3 Manager

join file1.txt file2.txt
# 1 Alice Engineer
# 2 Bob Designer
# 3 Charlie Manager
```

### Указание полей соединения

```bash
# -1 N - поле из первого файла
# -2 N - поле из второго файла

# Соединить по 2-му полю первого и 1-му второго
join -1 2 -2 1 file1.txt file2.txt
```

### Разделитель полей (-t)

```bash
# CSV файлы
join -t',' file1.csv file2.csv

# С указанием полей
join -t',' -1 1 -2 2 file1.csv file2.csv
```

### Вывод полей (-o)

```bash
# Указать какие поля выводить
# Формат: FILENUM.FIELDNUM

join -o 1.1,1.2,2.2 file1.txt file2.txt
# 1 Alice Engineer
# 2 Bob Designer

# Все поля первого файла + 2-е поле второго
join -o 1.1,1.2,2.2 file1.txt file2.txt
```

### Несовпадающие строки

```bash
# Показать несопоставленные из файла 1 (-a 1)
join -a 1 file1.txt file2.txt

# Показать несопоставленные из обоих файлов (-a 1 -a 2)
join -a 1 -a 2 file1.txt file2.txt

# Только несопоставленные из файла 1 (-v 1)
join -v 1 file1.txt file2.txt

# Заполнитель для пропущенных (-e)
join -a 1 -a 2 -e 'NULL' -o 1.1,1.2,2.2 file1.txt file2.txt
```

### Практические примеры

```bash
# Подготовка файлов (должны быть отсортированы!)
sort -k1 file1.txt > file1_sorted.txt
sort -k1 file2.txt > file2_sorted.txt

# Объединение
join file1_sorted.txt file2_sorted.txt

# Объединение CSV по 2-му полю
sort -t',' -k2 data1.csv > sorted1.csv
sort -t',' -k2 data2.csv > sorted2.csv
join -t',' -1 2 -2 2 sorted1.csv sorted2.csv

# LEFT JOIN (все из первого + совпадения из второго)
join -a 1 -e 'NULL' -o 1.1,1.2,2.2 file1.txt file2.txt

# INNER JOIN (только совпадения)
join file1.txt file2.txt

# Найти строки только в первом файле
join -v 1 file1.txt file2.txt

# Найти строки только во втором файле
join -v 2 file1.txt file2.txt
```

## Комбинирование команд

### Пайплайны

```bash
# Извлечь поля, отсортировать, посчитать уникальные
cut -d',' -f2 data.csv | sort | uniq -c | sort -rn

# Объединить данные из нескольких источников
paste <(cut -d':' -f1 /etc/passwd) <(cut -d':' -f3 /etc/passwd)

# Транспонировать CSV (строки в колонки)
# Простой случай: несколько строк -> одна строка
cat file.txt | paste -s -d','

# Сравнение двух файлов
sort file1.txt > s1.txt
sort file2.txt > s2.txt
join -v 1 s1.txt s2.txt  # только в file1
join -v 2 s1.txt s2.txt  # только в file2
join s1.txt s2.txt       # общие
```

### Работа с CSV

```bash
# Извлечь колонки
cut -d',' -f1,3 data.csv

# Переставить колонки
cut -d',' -f3,1,2 data.csv

# Добавить новую колонку
paste -d',' data.csv <(echo "new_column"; seq $(wc -l < data.csv))

# Удалить колонку (все кроме 2-й)
cut -d',' --complement -f2 data.csv
```

## Альтернативы: awk

Для сложной обработки колонок лучше использовать awk:

```bash
# cut эквивалент
awk -F',' '{print $1,$3}' data.csv

# paste эквивалент
awk 'FNR==NR {a[NR]=$0; next} {print a[FNR],$0}' file1.txt file2.txt

# join эквивалент
awk 'FNR==NR {a[$1]=$0; next} $1 in a {print a[$1],$2}' file1.txt file2.txt
```

## Резюме команд

| Команда | Описание |
|---------|----------|
| `cut -c1-5` | Символы 1-5 |
| `cut -f1,3` | Поля 1 и 3 |
| `cut -d',' -f1` | Поле 1, разделитель запятая |
| `cut --complement -f2` | Всё кроме поля 2 |
| `paste f1 f2` | Объединить построчно |
| `paste -d','` | С разделителем |
| `paste -s` | Все строки в одну |
| `join f1 f2` | Объединить по ключу |
| `join -a 1` | LEFT JOIN |
| `join -v 1` | Только из f1 |
| `join -t','` | Разделитель полей |

---

[prev: 01-cat-sort-uniq](./01-cat-sort-uniq.md) | [next: 03-diff-patch](./03-diff-patch.md)
