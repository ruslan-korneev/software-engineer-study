# Печать в Linux - lpr, lpstat и система CUPS

[prev: 03-groff](../08-formatting-output/03-groff.md) | [next: 01-what-is-compiling](../10-compiling/01-what-is-compiling.md)

---
## Введение в печать в Linux

В Linux печать управляется системой **CUPS** (Common Unix Printing System). CUPS предоставляет:

- Универсальный интерфейс для всех принтеров
- Поддержку сетевой печати
- Веб-интерфейс для настройки
- Совместимость с командами BSD (lpr, lpq, lprm)

## Установка и настройка CUPS

### Установка

```bash
# Debian/Ubuntu
sudo apt install cups

# RHEL/CentOS
sudo yum install cups

# Fedora
sudo dnf install cups

# Запуск службы
sudo systemctl start cups
sudo systemctl enable cups
```

### Веб-интерфейс

```bash
# Доступен по адресу:
http://localhost:631

# Для удалённого доступа
sudo cupsctl --remote-admin

# Настройка в /etc/cups/cupsd.conf
```

### Добавление принтера

```bash
# Через веб-интерфейс: http://localhost:631/admin

# Через командную строку
sudo lpadmin -p printer_name -E -v socket://192.168.1.100 -m everywhere

# Сетевой принтер по URI
sudo lpadmin -p HP_LaserJet -E -v ipp://printer.local/ipp/print -m everywhere

# Установить принтер по умолчанию
sudo lpadmin -d printer_name
```

## lpr - отправка на печать

### Базовое использование

```bash
# Напечатать файл
lpr document.pdf

# Напечатать несколько файлов
lpr file1.pdf file2.pdf file3.pdf

# Указать принтер
lpr -P HP_LaserJet document.pdf

# Из stdin
cat document.txt | lpr
echo "Test page" | lpr
```

### Опции печати

```bash
# Количество копий
lpr -# 3 document.pdf

# Двусторонняя печать
lpr -o sides=two-sided-long-edge document.pdf
lpr -o sides=two-sided-short-edge document.pdf

# Ориентация
lpr -o landscape document.pdf
lpr -o orientation-requested=4 document.pdf  # portrait

# Масштабирование
lpr -o fit-to-page document.pdf
lpr -o scaling=75 document.pdf    # 75%

# Страницы
lpr -o page-ranges=1-5 document.pdf
lpr -o page-ranges=1,3,5-7 document.pdf

# Несколько страниц на листе
lpr -o number-up=2 document.pdf   # 2 страницы на лист
lpr -o number-up=4 document.pdf   # 4 страницы на лист

# Качество
lpr -o print-quality=5 document.pdf  # 3=draft, 4=normal, 5=best

# Размер бумаги
lpr -o media=A4 document.pdf
lpr -o media=Letter document.pdf
```

### Приоритет задания

```bash
# Приоритет (1-100, по умолчанию 50)
lpr -q 75 document.pdf  # более высокий приоритет
```

## lpq - просмотр очереди печати

### Базовое использование

```bash
# Показать очередь по умолчанию
lpq

# Показать очередь конкретного принтера
lpq -P HP_LaserJet

# Все принтеры
lpq -a

# Пример вывода:
# HP_LaserJet is ready
# Rank    Owner   Job     File(s)         Total Size
# active  user    123     document.pdf    1024 bytes
# 1st     user    124     report.pdf      2048 bytes
```

### Подробная информация

```bash
# Подробный вывод
lpq -l

# Обновлять каждые N секунд
lpq +5    # обновлять каждые 5 секунд
```

## lprm - удаление заданий

### Удаление заданий

```bash
# Удалить текущее задание пользователя
lprm

# Удалить конкретное задание по номеру
lprm 123

# Удалить все задания пользователя
lprm -

# Удалить задание с принтера
lprm -P HP_LaserJet 123

# Удалить все задания (root)
sudo lprm -
```

## lpstat - статус печати

### Информация о принтерах

```bash
# Статус всех принтеров
lpstat -a

# Статус очереди
lpstat -o

# Статус конкретного принтера
lpstat -p HP_LaserJet

# Принтер по умолчанию
lpstat -d

# Все устройства
lpstat -v

# Длинный формат
lpstat -l

# Полная информация
lpstat -t
```

### Примеры вывода

```bash
$ lpstat -t
# scheduler is running
# system default destination: HP_LaserJet
# device for HP_LaserJet: socket://192.168.1.100
# HP_LaserJet accepting requests since ...
# printer HP_LaserJet is idle
```

## lpadmin - администрирование принтеров

### Управление принтерами

```bash
# Добавить принтер
sudo lpadmin -p printer_name -E -v device_uri -m driver.ppd

# Удалить принтер
sudo lpadmin -x printer_name

# Установить по умолчанию
sudo lpadmin -d printer_name

# Включить/выключить принтер
cupsenable printer_name
cupsdisable printer_name

# Принять/отклонить задания
cupsaccept printer_name
cupsreject printer_name
```

### Настройки принтера

```bash
# Установить опции по умолчанию
sudo lpadmin -p printer_name -o media=A4
sudo lpadmin -p printer_name -o sides=two-sided-long-edge

# Показать опции принтера
lpoptions -p printer_name -l

# Сохранить опции для пользователя
lpoptions -p printer_name -o media=A4
```

## lp - альтернатива lpr (System V)

```bash
# Печать файла
lp document.pdf

# Указать принтер
lp -d HP_LaserJet document.pdf

# Количество копий
lp -n 3 document.pdf

# Опции (аналогично lpr -o)
lp -o sides=two-sided-long-edge document.pdf

# Приоритет (1-100)
lp -q 75 document.pdf

# Заголовок задания
lp -t "My Document" document.pdf
```

## a2ps - форматирование для печати

### Что такое a2ps?

**a2ps** (any to PostScript) конвертирует текстовые файлы в PostScript для печати с красивым форматированием.

### Установка

```bash
sudo apt install a2ps
```

### Использование

```bash
# Базовая печать
a2ps file.txt

# Сохранить в файл
a2ps file.txt -o output.ps

# Печать кода с подсветкой
a2ps --highlight=python script.py

# Две колонки
a2ps -2 file.txt

# Одна колонка
a2ps -1 file.txt

# Без заголовка
a2ps --no-header file.txt

# Альбомная ориентация
a2ps -r file.txt

# Нумерация строк
a2ps -n file.txt

# Границы страниц
a2ps --borders=yes file.txt
```

### Практические примеры

```bash
# Печать исходного кода
a2ps -2 --highlight=python -n script.py | lpr

# Печать нескольких файлов
a2ps -2 *.c -o code.ps

# Книжный формат (для чтения)
a2ps -B -1 --medium=A4 document.txt
```

## enscript - альтернатива a2ps

```bash
# Установка
sudo apt install enscript

# Базовое использование
enscript file.txt

# Печать с подсветкой
enscript --highlight=python script.py

# Две колонки
enscript -2r file.txt

# PDF вывод
enscript -p output.ps file.txt
ps2pdf output.ps output.pdf
```

## Практические сценарии

### Печать документов

```bash
# Печать PDF
lpr document.pdf

# Печать определённых страниц
lpr -o page-ranges=1-5 document.pdf

# Двусторонняя печать буклета
lpr -o number-up=2 -o sides=two-sided-short-edge document.pdf
```

### Печать из приложений

```bash
# Печать веб-страницы
wkhtmltopdf https://example.com page.pdf && lpr page.pdf

# Печать man-страницы
man command | a2ps -1 | lpr

# Печать diff
diff file1 file2 | a2ps | lpr
```

### Автоматизация печати

```bash
#!/bin/bash
# Автопечать новых файлов

WATCH_DIR="/home/user/print_queue"
inotifywait -m "$WATCH_DIR" -e create |
while read path action file; do
    lpr "${path}${file}"
    rm "${path}${file}"
done
```

## Устранение проблем

```bash
# Перезапуск CUPS
sudo systemctl restart cups

# Просмотр логов
sudo tail -f /var/log/cups/error_log

# Очистить очередь
cancel -a

# Проверить статус
lpstat -t

# Тестовая страница
lpr /usr/share/cups/data/testprint
```

## Резюме команд

| Команда | Описание |
|---------|----------|
| `lpr file` | Напечатать файл |
| `lpr -P printer file` | На конкретный принтер |
| `lpr -# N` | N копий |
| `lpr -o option` | Опции печати |
| `lpq` | Показать очередь |
| `lprm job_id` | Удалить задание |
| `lpstat -t` | Полный статус |
| `lpstat -d` | Принтер по умолчанию |
| `lpadmin -p name` | Настроить принтер |
| `lpoptions -l` | Опции принтера |
| `cancel -a` | Отменить все задания |

---

[prev: 03-groff](../08-formatting-output/03-groff.md) | [next: 01-what-is-compiling](../10-compiling/01-what-is-compiling.md)
