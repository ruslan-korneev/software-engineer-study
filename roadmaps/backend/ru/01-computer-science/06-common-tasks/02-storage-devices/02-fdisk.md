# fdisk - управление разделами дисков

[prev: 01-mounting](./01-mounting.md) | [next: 03-mkfs-fsck](./03-mkfs-fsck.md)

---
## Что такое fdisk?

**fdisk** - это классическая утилита для создания и управления таблицей разделов на дисках. Она работает с таблицами разделов MBR (Master Boot Record) и GPT (GUID Partition Table).

## Таблицы разделов: MBR vs GPT

### MBR (Master Boot Record)

- Старый стандарт (1983 год)
- Максимум 4 первичных раздела (или 3 первичных + 1 расширенный)
- Максимальный размер раздела: 2 ТБ
- Загрузочный код в первых 446 байтах

### GPT (GUID Partition Table)

- Современный стандарт (часть UEFI)
- До 128 разделов (по умолчанию)
- Максимальный размер раздела: 9.4 ЗБ (зеттабайт)
- Резервная копия таблицы в конце диска
- Защита CRC32

## Просмотр информации о дисках

### Список разделов

```bash
# Показать все диски и разделы
sudo fdisk -l

# Конкретный диск
sudo fdisk -l /dev/sda

# Пример вывода:
# Disk /dev/sda: 500 GiB, 536870912000 bytes, 1048576000 sectors
# Disk model: Samsung SSD 870
# Units: sectors of 1 * 512 = 512 bytes
# Sector size (logical/physical): 512 bytes / 512 bytes
# Disklabel type: gpt
#
# Device       Start        End    Sectors   Size Type
# /dev/sda1     2048    1050623    1048576   512M EFI System
# /dev/sda2  1050624    3147775    2097152     1G Linux filesystem
# /dev/sda3  3147776 1048573951 1045426176 498.5G Linux filesystem
```

### Альтернативные инструменты

```bash
# lsblk - удобный вывод
lsblk

# parted - показать детали
sudo parted -l

# sfdisk - скриптовый вывод
sudo sfdisk -l /dev/sda
```

## Интерактивный режим fdisk

### Запуск

```bash
# Открыть диск для редактирования
sudo fdisk /dev/sdb

# ВНИМАНИЕ: Изменения применяются только после команды 'w'
```

### Основные команды fdisk

| Команда | Описание |
|---------|----------|
| `m` | Показать справку |
| `p` | Печать таблицы разделов |
| `n` | Создать новый раздел |
| `d` | Удалить раздел |
| `t` | Изменить тип раздела |
| `l` | Список кодов типов |
| `w` | Записать изменения и выйти |
| `q` | Выйти без сохранения |
| `g` | Создать новую GPT таблицу |
| `o` | Создать новую MBR таблицу |
| `v` | Проверить таблицу |
| `F` | Показать свободное место |

### Создание раздела

```bash
sudo fdisk /dev/sdb

# Пример сессии:
Command (m for help): n         # новый раздел
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p           # первичный
Partition number (1-4, default 1): 1
First sector (2048-xxx, default 2048):    # Enter - использовать по умолчанию
Last sector, +/-sectors or +/-size{K,M,G,T,P}: +100G  # размер 100GB

Command (m for help): w         # записать изменения
```

### Удаление раздела

```bash
sudo fdisk /dev/sdb

Command (m for help): d
Partition number (1-4): 2       # номер раздела для удаления

Command (m for help): w
```

### Изменение типа раздела

```bash
sudo fdisk /dev/sdb

Command (m for help): t
Partition number (1-4): 1
Hex code (type L to list all codes): 8e    # Linux LVM
# или
Hex code: 82                               # Linux swap
Hex code: 83                               # Linux filesystem
Hex code: ef                               # EFI System

Command (m for help): w
```

### Распространённые коды типов

| Код | Тип |
|-----|-----|
| `83` | Linux |
| `82` | Linux swap |
| `8e` | Linux LVM |
| `fd` | Linux RAID |
| `ef` | EFI System |
| `7` | HPFS/NTFS |
| `c` | FAT32 (LBA) |

## Создание GPT таблицы

```bash
sudo fdisk /dev/sdb

Command (m for help): g         # создать GPT
Created a new GPT disklabel

Command (m for help): n         # создать раздел
Partition number (1-128, default 1): 1
First sector: Enter
Last sector: +500G

Command (m for help): w
```

## gdisk - альтернатива для GPT

`gdisk` - специализированный инструмент для GPT дисков.

```bash
# Установка
sudo apt install gdisk

# Запуск
sudo gdisk /dev/sdb

# Команды аналогичны fdisk
# p - печать
# n - новый раздел
# d - удалить
# w - записать
# q - выйти
```

## parted - универсальный инструмент

`parted` работает как с MBR, так и с GPT, и поддерживает диски больше 2 ТБ.

### Интерактивный режим

```bash
sudo parted /dev/sdb

# Показать информацию
(parted) print

# Создать GPT таблицу
(parted) mklabel gpt

# Создать раздел
(parted) mkpart primary ext4 0% 50%
(parted) mkpart primary ext4 50% 100%

# Удалить раздел
(parted) rm 1

# Изменить размер (осторожно!)
(parted) resizepart 1 200GB

# Выход
(parted) quit
```

### Команды одной строкой

```bash
# Создать GPT таблицу
sudo parted /dev/sdb mklabel gpt

# Создать раздел
sudo parted /dev/sdb mkpart primary ext4 0% 100%

# Показать разделы
sudo parted /dev/sdb print
```

## sfdisk - скриптовое управление

`sfdisk` удобен для автоматизации и резервного копирования таблиц разделов.

### Резервное копирование таблицы разделов

```bash
# Сохранить таблицу разделов
sudo sfdisk -d /dev/sda > sda_partitions.txt

# Восстановить таблицу разделов
sudo sfdisk /dev/sdb < sda_partitions.txt
```

### Создание разделов из скрипта

```bash
# Формат: start, size, type, bootable
echo "start=2048, size=+100G, type=83" | sudo sfdisk /dev/sdb

# Несколько разделов
cat << EOF | sudo sfdisk /dev/sdb
label: gpt
start=2048, size=512M, type=ef00
start=, size=100G, type=8300
start=, size=, type=8300
EOF
```

## Практические сценарии

### Подготовка нового диска

```bash
# 1. Посмотреть текущее состояние
sudo fdisk -l /dev/sdb

# 2. Создать GPT таблицу и раздел
sudo fdisk /dev/sdb
# g (создать GPT)
# n (новый раздел, принять все по умолчанию)
# w (записать)

# 3. Создать файловую систему
sudo mkfs.ext4 /dev/sdb1

# 4. Смонтировать
sudo mkdir /mnt/data
sudo mount /dev/sdb1 /mnt/data

# 5. Добавить в fstab для автомонтирования
sudo blkid /dev/sdb1
# Записать UUID в /etc/fstab
```

### Клонирование структуры разделов

```bash
# Экспортировать с исходного диска
sudo sfdisk -d /dev/sda > partitions.txt

# Импортировать на новый диск
sudo sfdisk /dev/sdb < partitions.txt
```

### Изменение размера раздела

**ВНИМАНИЕ:** Изменение размера - опасная операция! Сделайте резервную копию!

```bash
# 1. Размонтировать раздел
sudo umount /dev/sdb1

# 2. Проверить файловую систему
sudo e2fsck -f /dev/sdb1

# 3. Изменить размер ФС (уменьшить до 50G)
sudo resize2fs /dev/sdb1 50G

# 4. Изменить размер раздела в fdisk
sudo fdisk /dev/sdb
# d - удалить раздел
# n - создать новый раздел с новым размером
# w - записать

# 5. Для увеличения - сначала раздел, потом ФС
sudo resize2fs /dev/sdb1
```

## cfdisk - визуальный интерфейс

`cfdisk` предоставляет псевдографический интерфейс:

```bash
sudo cfdisk /dev/sdb
```

Управление:
- Стрелки: навигация
- Enter: выбор
- `New`: создать раздел
- `Delete`: удалить
- `Type`: изменить тип
- `Write`: записать изменения
- `Quit`: выход

## Важные предупреждения

### Перед работой с разделами

1. **Сделайте резервную копию данных!**
2. Размонтируйте все разделы диска
3. Убедитесь, что выбрали правильный диск
4. Изменения применяются только после `w`

### Обновление ядра о изменениях

```bash
# Перечитать таблицу разделов
sudo partprobe /dev/sdb

# Альтернатива
sudo hdparm -z /dev/sdb

# Или перезагрузка системы
```

### Проверка выравнивания

Современные диски требуют выравнивания разделов по 1МБ (2048 секторов):

```bash
# Проверить выравнивание в parted
sudo parted /dev/sdb align-check optimal 1

# fdisk по умолчанию выравнивает правильно
```

## Сравнение инструментов

| Инструмент | MBR | GPT | Интерфейс | Особенности |
|------------|-----|-----|-----------|-------------|
| `fdisk` | Да | Да | Интерактивный | Классический, прост в использовании |
| `gdisk` | Нет | Да | Интерактивный | Специализирован для GPT |
| `parted` | Да | Да | Интерактивный/CLI | Изменение размера, скрипты |
| `sfdisk` | Да | Да | CLI | Скрипты, резервирование |
| `cfdisk` | Да | Да | TUI | Визуальный интерфейс |

## Резюме

```bash
# Просмотр
sudo fdisk -l              # все диски
lsblk                      # древовидная структура

# Редактирование
sudo fdisk /dev/sdX        # интерактивный режим

# Основные команды в fdisk
p - печать
n - новый раздел
d - удалить раздел
t - изменить тип
w - записать и выйти
q - выйти без сохранения
g - создать GPT
o - создать MBR

# После изменений
sudo partprobe /dev/sdX    # обновить ядро
sudo mkfs.ext4 /dev/sdX1   # создать ФС
```

---

[prev: 01-mounting](./01-mounting.md) | [next: 03-mkfs-fsck](./03-mkfs-fsck.md)
