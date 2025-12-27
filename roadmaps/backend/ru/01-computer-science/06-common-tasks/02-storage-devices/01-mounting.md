# Монтирование файловых систем

## Что такое монтирование?

**Монтирование** - это процесс подключения файловой системы устройства к определённой точке в дереве каталогов Linux. В отличие от Windows, где каждый диск имеет свою букву (C:, D:), в Linux все устройства объединены в единое дерево с корнем `/`.

## Основные концепции

### Файлы устройств

Все устройства хранения в Linux представлены как файлы в `/dev/`:

```bash
# Жёсткие диски и SSD (SATA/NVMe)
/dev/sda        # первый SATA/SCSI диск
/dev/sdb        # второй диск
/dev/nvme0n1    # первый NVMe SSD

# Разделы
/dev/sda1       # первый раздел первого диска
/dev/sda2       # второй раздел
/dev/nvme0n1p1  # первый раздел NVMe

# USB-накопители
/dev/sdc        # обычно подключаются как sdX

# CD/DVD
/dev/sr0        # первый оптический привод
```

### Точка монтирования

**Точка монтирования** - это пустая директория, к которой будет подключена файловая система:

```bash
# Стандартные точки монтирования
/mnt            # временное монтирование
/media          # автомонтирование съёмных носителей
/home           # часто отдельный раздел
/boot           # загрузочный раздел
```

## Команда mount

### Базовое использование

```bash
# Синтаксис
mount [опции] устройство точка_монтирования

# Примеры
sudo mount /dev/sdb1 /mnt
sudo mount /dev/sdb1 /mnt/usb

# Монтирование с указанием типа ФС
sudo mount -t ext4 /dev/sdb1 /mnt
sudo mount -t ntfs /dev/sdc1 /mnt/windows
```

### Просмотр смонтированных ФС

```bash
# Все смонтированные файловые системы
mount

# Более читаемый формат
mount | column -t

# Фильтр по типу
mount -t ext4

# Через /proc
cat /proc/mounts

# Использование findmnt (рекомендуется)
findmnt
findmnt -t ext4
findmnt /dev/sda1
findmnt /home
```

### Опции монтирования

```bash
# Только для чтения
sudo mount -o ro /dev/sdb1 /mnt

# Чтение и запись (по умолчанию)
sudo mount -o rw /dev/sdb1 /mnt

# Несколько опций через запятую
sudo mount -o rw,noexec,nosuid /dev/sdb1 /mnt
```

#### Основные опции

| Опция | Описание |
|-------|----------|
| `ro` | Только чтение |
| `rw` | Чтение и запись |
| `noexec` | Запретить выполнение программ |
| `nosuid` | Игнорировать SUID/SGID биты |
| `nodev` | Игнорировать файлы устройств |
| `noatime` | Не обновлять время доступа (улучшает производительность) |
| `sync` | Синхронная запись (медленнее, безопаснее) |
| `async` | Асинхронная запись (быстрее) |
| `user` | Разрешить монтирование обычным пользователям |
| `auto` | Монтировать автоматически при загрузке |
| `noauto` | Не монтировать автоматически |
| `defaults` | rw, suid, dev, exec, auto, nouser, async |

### Монтирование по UUID или LABEL

```bash
# Узнать UUID раздела
sudo blkid
# или
lsblk -f

# Монтировать по UUID
sudo mount UUID="a1b2c3d4-e5f6-7890-abcd-ef1234567890" /mnt

# Монтировать по метке
sudo mount LABEL="MyData" /mnt
```

### Перемонтирование

```bash
# Перемонтировать с другими опциями (без размонтирования)
sudo mount -o remount,ro /mnt

# Перемонтировать корень только для чтения
sudo mount -o remount,ro /
```

## Команда umount

### Базовое использование

```bash
# Размонтировать по точке монтирования
sudo umount /mnt

# Размонтировать по устройству
sudo umount /dev/sdb1

# Размонтировать все ФС определённого типа
sudo umount -a -t tmpfs
```

### Решение проблемы "device is busy"

Если устройство занято, нужно найти процессы, использующие его:

```bash
# Показать процессы, использующие точку монтирования
lsof /mnt
# или
fuser -v /mnt

# Убить все процессы (осторожно!)
fuser -km /mnt

# Принудительное размонтирование (опасно!)
sudo umount -f /mnt

# Отложенное размонтирование (безопасно)
sudo umount -l /mnt    # lazy - размонтирует когда освободится
```

## Файл /etc/fstab

`/etc/fstab` определяет, какие ФС монтировать при загрузке системы.

### Формат файла

```bash
# устройство        точка_монтирования  тип    опции              dump  pass
UUID=xxx-xxx-xxx   /                    ext4   defaults           0     1
UUID=yyy-yyy-yyy   /home                ext4   defaults           0     2
UUID=zzz-zzz-zzz   swap                 swap   sw                 0     0
/dev/sdb1          /mnt/data            ext4   defaults,noatime   0     2
```

### Поля fstab

1. **Устройство** - /dev/xxx, UUID=, LABEL=
2. **Точка монтирования** - путь к директории
3. **Тип ФС** - ext4, xfs, ntfs, vfat, swap
4. **Опции** - параметры монтирования
5. **dump** - 0 или 1 (использовать ли утилиту dump для бекапа)
6. **pass** - порядок проверки fsck (0-не проверять, 1-корень, 2-остальные)

### Примеры записей fstab

```bash
# Корневая ФС
UUID=a1b2c3d4-... / ext4 defaults 0 1

# Домашний раздел
UUID=e5f6g7h8-... /home ext4 defaults,noatime 0 2

# Windows раздел (NTFS)
UUID=ABCD1234... /mnt/windows ntfs-3g defaults,uid=1000,gid=1000 0 0

# USB диск (не монтировать автоматически)
UUID=ijkl-mnop /mnt/usb vfat noauto,user,rw 0 0

# Сетевой ресурс NFS
server:/share /mnt/nfs nfs defaults 0 0

# tmpfs (RAM диск)
tmpfs /tmp tmpfs defaults,noatime,mode=1777 0 0
```

### Проверка fstab

```bash
# Смонтировать все из fstab
sudo mount -a

# Проверить синтаксис (перед перезагрузкой!)
sudo findmnt --verify

# Монтировать конкретную запись из fstab
sudo mount /mnt/data
```

## Информация о дисках и разделах

### lsblk - список блочных устройств

```bash
# Базовый вывод
lsblk

# С файловыми системами и UUID
lsblk -f

# С размерами в байтах
lsblk -b

# Все поля
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,UUID

# Пример вывода:
# NAME   SIZE TYPE FSTYPE MOUNTPOINT       UUID
# sda    500G disk
# ├─sda1 512M part vfat   /boot/efi        ABCD-1234
# ├─sda2   1G part ext4   /boot            xxxx-xxxx
# └─sda3 498G part ext4   /                yyyy-yyyy
```

### blkid - информация о блочных устройствах

```bash
# Все устройства
sudo blkid

# Конкретное устройство
sudo blkid /dev/sda1

# Только UUID
sudo blkid -s UUID /dev/sda1

# Вывод в формате для fstab
sudo blkid -o export /dev/sda1
```

### df - использование дискового пространства

```bash
# Базовый вывод
df

# В человекочитаемом формате
df -h

# С типом ФС
df -T

# Только локальные ФС
df -l

# Конкретная ФС
df -h /home

# Пример:
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda3       458G  125G  310G  29% /
# /dev/sda1       511M   62M  450M  12% /boot/efi
```

### du - использование места директориями

```bash
# Размер директории
du -sh /var/log

# Размер с поддиректориями
du -h /var/log

# Глубина вывода
du -h --max-depth=1 /var

# Отсортировать по размеру
du -h --max-depth=1 /var | sort -h

# Самые большие файлы/папки
du -ah /var | sort -rh | head -10
```

## Специальные типы монтирования

### Монтирование ISO образа

```bash
# Создать точку монтирования
sudo mkdir /mnt/iso

# Смонтировать ISO
sudo mount -o loop image.iso /mnt/iso

# Размонтировать
sudo umount /mnt/iso
```

### Монтирование образа диска

```bash
# С указанием смещения (для образов с разделами)
sudo mount -o loop,offset=$((2048*512)) disk.img /mnt
```

### bind mount - повторное монтирование директории

```bash
# Смонтировать директорию в другое место
sudo mount --bind /home/user/data /var/www/html/data

# В fstab:
/home/user/data /var/www/html/data none bind 0 0
```

### tmpfs - файловая система в RAM

```bash
# Создать tmpfs
sudo mount -t tmpfs -o size=512m tmpfs /mnt/ramdisk

# В fstab для /tmp:
tmpfs /tmp tmpfs defaults,noatime,mode=1777,size=2g 0 0
```

## Автоматическое монтирование

### systemd mount units

Вместо fstab можно использовать systemd unit файлы:

```ini
# /etc/systemd/system/mnt-data.mount
[Unit]
Description=Mount Data Partition

[Mount]
What=/dev/sdb1
Where=/mnt/data
Type=ext4
Options=defaults

[Install]
WantedBy=multi-user.target
```

```bash
# Активировать
sudo systemctl enable mnt-data.mount
sudo systemctl start mnt-data.mount
```

### autofs - монтирование по требованию

```bash
# Установка
sudo apt install autofs

# Конфигурация /etc/auto.master
/mnt/auto   /etc/auto.misc

# /etc/auto.misc
usb    -fstype=auto    :/dev/sdb1

# Перезапуск
sudo systemctl restart autofs

# Теперь /mnt/auto/usb автоматически монтируется при обращении
```

## Полезные советы

### 1. Безопасное извлечение USB

```bash
# Синхронизировать буферы
sync

# Размонтировать
sudo umount /media/user/USB

# Или использовать eject
sudo eject /dev/sdb
```

### 2. Проверка перед монтированием

```bash
# Проверить файловую систему
sudo fsck /dev/sdb1

# Автоматическое исправление
sudo fsck -a /dev/sdb1
```

### 3. Монтирование NTFS с полным доступом

```bash
# Установить драйвер
sudo apt install ntfs-3g

# Монтировать
sudo mount -t ntfs-3g /dev/sdc1 /mnt/windows
```

## Резюме команд

| Команда | Описание |
|---------|----------|
| `mount` | Показать/выполнить монтирование |
| `umount` | Размонтировать |
| `findmnt` | Показать дерево монтирования |
| `lsblk` | Список блочных устройств |
| `blkid` | UUID и типы ФС |
| `df` | Использование дискового пространства |
| `du` | Размер директорий |
| `fuser` | Процессы, использующие ФС |
| `lsof` | Открытые файлы |
