# mkfs и fsck - создание и проверка файловых систем

## mkfs - создание файловых систем

### Что такое mkfs?

**mkfs** (make filesystem) - команда для создания (форматирования) файловой системы на разделе диска или блочном устройстве.

### Типы файловых систем Linux

| ФС | Описание | Использование |
|----|----------|---------------|
| **ext4** | Четвёртая расширенная ФС | Стандарт для Linux |
| **xfs** | Высокопроизводительная ФС | RHEL, большие файлы |
| **btrfs** | B-tree ФС | Снимки, RAID, сжатие |
| **f2fs** | Flash-friendly FS | SSD, SD карты |
| **vfat/fat32** | FAT32 | USB, совместимость |
| **ntfs** | Windows NTFS | Windows разделы |
| **exfat** | Extended FAT | Большие USB |

### Базовое использование mkfs

```bash
# Общий синтаксис
mkfs -t тип устройство
# или
mkfs.тип устройство

# Примеры:
sudo mkfs -t ext4 /dev/sdb1
sudo mkfs.ext4 /dev/sdb1       # эквивалентно

sudo mkfs.xfs /dev/sdc1
sudo mkfs.btrfs /dev/sdd1
sudo mkfs.vfat /dev/sde1       # FAT32
```

### mkfs.ext4 - подробно

```bash
# Базовое форматирование
sudo mkfs.ext4 /dev/sdb1

# С меткой тома
sudo mkfs.ext4 -L "MyData" /dev/sdb1

# С указанием UUID
sudo mkfs.ext4 -U random /dev/sdb1
sudo mkfs.ext4 -U 12345678-1234-1234-1234-123456789abc /dev/sdb1

# Размер блока (1024, 2048, 4096)
sudo mkfs.ext4 -b 4096 /dev/sdb1

# Процент зарезервированного места для root (по умолчанию 5%)
sudo mkfs.ext4 -m 1 /dev/sdb1    # 1% вместо 5%
sudo mkfs.ext4 -m 0 /dev/sdb1    # 0% (для не-системных разделов)

# Принудительно (без подтверждения)
sudo mkfs.ext4 -F /dev/sdb1

# Комбинация опций
sudo mkfs.ext4 -L "Data" -m 1 -b 4096 /dev/sdb1
```

### mkfs.xfs

```bash
# Базовое форматирование
sudo mkfs.xfs /dev/sdb1

# С меткой
sudo mkfs.xfs -L "XFSData" /dev/sdb1

# Принудительно (перезаписать существующую ФС)
sudo mkfs.xfs -f /dev/sdb1

# Размер блока
sudo mkfs.xfs -b size=4096 /dev/sdb1
```

### mkfs.btrfs

```bash
# Базовое форматирование
sudo mkfs.btrfs /dev/sdb1

# С меткой
sudo mkfs.btrfs -L "BtrfsData" /dev/sdb1

# RAID0 на нескольких устройствах
sudo mkfs.btrfs -d raid0 -m raid1 /dev/sdb /dev/sdc

# Принудительно
sudo mkfs.btrfs -f /dev/sdb1
```

### mkfs.vfat (FAT32)

```bash
# FAT32 для USB
sudo mkfs.vfat -F 32 /dev/sdb1

# С меткой
sudo mkfs.vfat -F 32 -n "USB_DRIVE" /dev/sdb1

# Проверка на сбойные сектора
sudo mkfs.vfat -c /dev/sdb1
```

### mkswap - создание swap

```bash
# Создать swap раздел
sudo mkswap /dev/sdb2

# С меткой
sudo mkswap -L "swap" /dev/sdb2

# Включить swap
sudo swapon /dev/sdb2

# Отключить
sudo swapoff /dev/sdb2

# Создать swap файл
sudo dd if=/dev/zero of=/swapfile bs=1M count=4096
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Добавить в fstab
# /swapfile none swap sw 0 0
```

## tune2fs - настройка ext ФС

`tune2fs` позволяет изменять параметры ext2/ext3/ext4 без переформатирования.

```bash
# Показать параметры ФС
sudo tune2fs -l /dev/sdb1

# Изменить метку тома
sudo tune2fs -L "NewLabel" /dev/sdb1

# Изменить зарезервированное место (1%)
sudo tune2fs -m 1 /dev/sdb1

# Изменить интервал проверки (90 дней)
sudo tune2fs -i 90d /dev/sdb1

# Отключить автопроверку по времени
sudo tune2fs -i 0 /dev/sdb1

# Изменить количество монтирований до проверки
sudo tune2fs -c 50 /dev/sdb1

# Отключить автопроверку по количеству
sudo tune2fs -c 0 /dev/sdb1

# Добавить journal (преобразовать ext2 в ext3)
sudo tune2fs -j /dev/sdb1

# Включить/выключить опции ФС
sudo tune2fs -O ^has_journal /dev/sdb1  # отключить журнал
sudo tune2fs -O extent /dev/sdb1         # включить экстенты
```

## fsck - проверка файловых систем

### Что такое fsck?

**fsck** (file system check) - утилита для проверки и восстановления целостности файловых систем.

### Когда использовать fsck

- После некорректного выключения
- При ошибках чтения/записи
- По расписанию (автоматически)
- Перед клонированием раздела

### Важные правила

1. **Всегда размонтируйте раздел перед fsck!**
2. Для корневого раздела - загрузитесь с live USB
3. Сделайте бекап перед восстановлением

### Базовое использование

```bash
# Проверить раздел (должен быть размонтирован)
sudo fsck /dev/sdb1

# Автоматически исправлять ошибки
sudo fsck -a /dev/sdb1    # только безопасные исправления
sudo fsck -y /dev/sdb1    # отвечать "да" на все вопросы

# Проверить без исправления (только отчёт)
sudo fsck -n /dev/sdb1

# Принудительная проверка
sudo fsck -f /dev/sdb1

# Указать тип ФС
sudo fsck -t ext4 /dev/sdb1
sudo fsck.ext4 /dev/sdb1      # эквивалентно
```

### Коды возврата fsck

| Код | Значение |
|-----|----------|
| 0 | Нет ошибок |
| 1 | Ошибки исправлены |
| 2 | Требуется перезагрузка |
| 4 | Ошибки не исправлены |
| 8 | Операционная ошибка |
| 16 | Ошибка синтаксиса |
| 32 | fsck отменён пользователем |
| 128 | Ошибка shared library |

### e2fsck - для ext ФС

```bash
# Базовая проверка
sudo e2fsck /dev/sdb1

# Автоисправление
sudo e2fsck -p /dev/sdb1    # preen mode (безопасно)
sudo e2fsck -y /dev/sdb1    # отвечать "да"

# Принудительная проверка
sudo e2fsck -f /dev/sdb1

# Проверка сбойных блоков (медленно!)
sudo e2fsck -c /dev/sdb1

# Подробный вывод
sudo e2fsck -v /dev/sdb1

# Показать прогресс
sudo e2fsck -C 0 /dev/sdb1
```

### xfs_repair - для XFS

```bash
# Проверка XFS
sudo xfs_repair /dev/sdb1

# Только проверить (без исправления)
sudo xfs_repair -n /dev/sdb1

# Принудительная обработка
sudo xfs_repair -L /dev/sdb1    # очистить журнал (опасно!)

# Подробный вывод
sudo xfs_repair -v /dev/sdb1
```

### btrfsck / btrfs check - для Btrfs

```bash
# Проверка
sudo btrfs check /dev/sdb1

# Только чтение (рекомендуется сначала)
sudo btrfs check --readonly /dev/sdb1

# С исправлением (осторожно!)
sudo btrfs check --repair /dev/sdb1
```

## Проверка корневой ФС

Корневую ФС нельзя проверить на работающей системе. Варианты:

### Способ 1: Загрузка с Live USB

```bash
# Загрузиться с Ubuntu Live USB
# Найти корневой раздел
lsblk

# Проверить
sudo fsck -y /dev/sda2
```

### Способ 2: Форсировать проверку при загрузке

```bash
# Создать файл-триггер
sudo touch /forcefsck

# Перезагрузиться
sudo reboot

# Альтернатива: параметр ядра
# В GRUB добавить: fsck.mode=force
```

### Способ 3: Через systemd

```bash
# Запланировать проверку
sudo systemctl enable systemd-fsck-root.service
```

## Автоматическая проверка

### Настройка в fstab

Последнее поле в fstab определяет порядок проверки:

```bash
# /etc/fstab
# устройство    точка    тип    опции    dump  pass
UUID=xxx-xxx   /        ext4   defaults  0     1    # проверяется первым
UUID=yyy-yyy   /home    ext4   defaults  0     2    # проверяется после корня
UUID=zzz-zzz   /data    ext4   defaults  0     0    # не проверяется
```

### Настройка параметров проверки

```bash
# Показать текущие настройки
sudo tune2fs -l /dev/sda1 | grep -E "mount count|Check interval"

# Установить проверку каждые 30 монтирований
sudo tune2fs -c 30 /dev/sda1

# Установить проверку каждые 180 дней
sudo tune2fs -i 180d /dev/sda1

# Отключить автопроверку (не рекомендуется)
sudo tune2fs -c 0 -i 0 /dev/sda1
```

## badblocks - проверка на сбойные блоки

```bash
# Только чтение (безопасно)
sudo badblocks -v /dev/sdb

# Чтение-запись (уничтожит данные!)
sudo badblocks -w /dev/sdb

# Неразрушающий тест чтения-записи
sudo badblocks -n /dev/sdb

# Передать список bad blocks в e2fsck
sudo badblocks -o bad-blocks.txt /dev/sdb1
sudo e2fsck -l bad-blocks.txt /dev/sdb1
```

## Практические сценарии

### Подготовка нового диска

```bash
# 1. Создать раздел
sudo fdisk /dev/sdb
# n, Enter, Enter, Enter, w

# 2. Создать ФС с меткой и 1% резерва
sudo mkfs.ext4 -L "Data" -m 1 /dev/sdb1

# 3. Получить UUID
sudo blkid /dev/sdb1

# 4. Смонтировать
sudo mkdir /mnt/data
sudo mount /dev/sdb1 /mnt/data

# 5. Добавить в fstab
echo "UUID=xxx-xxx /mnt/data ext4 defaults 0 2" | sudo tee -a /etc/fstab
```

### Восстановление после краха

```bash
# 1. Загрузиться с Live USB

# 2. Найти повреждённый раздел
lsblk

# 3. Проверить и исправить
sudo fsck -y /dev/sda2

# 4. Если много ошибок - сохранить отчёт
sudo e2fsck -y /dev/sda2 2>&1 | tee fsck_report.txt

# 5. Перезагрузиться
```

### Преобразование ext2 в ext4

```bash
# 1. Размонтировать
sudo umount /dev/sdb1

# 2. Проверить ФС
sudo e2fsck -f /dev/sdb1

# 3. Добавить журнал (ext2 -> ext3)
sudo tune2fs -j /dev/sdb1

# 4. Добавить функции ext4
sudo tune2fs -O extents,uninit_bg,dir_index /dev/sdb1

# 5. Проверить снова
sudo e2fsck -f /dev/sdb1
```

## Резюме команд

| Задача | Команда |
|--------|---------|
| Создать ext4 | `sudo mkfs.ext4 /dev/sdX1` |
| Создать XFS | `sudo mkfs.xfs /dev/sdX1` |
| Создать FAT32 | `sudo mkfs.vfat -F 32 /dev/sdX1` |
| Создать swap | `sudo mkswap /dev/sdX1` |
| Проверить ФС | `sudo fsck /dev/sdX1` |
| Авто-исправить | `sudo fsck -y /dev/sdX1` |
| Настроить ext | `sudo tune2fs -l /dev/sdX1` |
| Изменить метку | `sudo tune2fs -L "Label" /dev/sdX1` |
| Проверить блоки | `sudo badblocks -v /dev/sdX` |
