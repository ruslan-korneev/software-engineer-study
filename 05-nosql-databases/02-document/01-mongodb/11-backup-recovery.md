# Backup and Recovery

## Введение

Резервное копирование (backup) и восстановление (recovery) являются критически важными аспектами администрирования MongoDB. Потеря данных может произойти по множеству причин: аппаратные сбои, человеческие ошибки, программные баги, кибератаки или стихийные бедствия. Грамотно спланированная стратегия бэкапов позволяет:

- **Минимизировать потерю данных** (RPO - Recovery Point Objective)
- **Сократить время восстановления** (RTO - Recovery Time Objective)
- **Обеспечить соответствие требованиям** регуляторов и бизнеса
- **Защитить от ransomware** и других угроз
- **Поддержать миграции** и тестирование на продакшн-данных

MongoDB предоставляет несколько методов резервного копирования, каждый из которых имеет свои преимущества и ограничения.

---

## mongodump и mongorestore

### Обзор

`mongodump` и `mongorestore` — это стандартные утилиты командной строки для создания логических бэкапов MongoDB. Они экспортируют данные в формате BSON и позволяют гибко управлять процессом резервного копирования.

### Базовый синтаксис

```bash
# Бэкап всей базы данных
mongodump --uri="mongodb://localhost:27017" --out=/backup/dump

# Восстановление из бэкапа
mongorestore --uri="mongodb://localhost:27017" /backup/dump
```

### Основные опции mongodump

```bash
# Бэкап с аутентификацией
mongodump \
  --host=localhost \
  --port=27017 \
  --username=admin \
  --password=secret \
  --authenticationDatabase=admin \
  --out=/backup/dump

# Бэкап конкретной базы данных
mongodump --db=myapp --out=/backup/dump

# Бэкап конкретной коллекции
mongodump --db=myapp --collection=users --out=/backup/dump

# Бэкап с фильтрацией документов
mongodump --db=myapp --collection=orders \
  --query='{"status": "completed", "createdAt": {"$gte": {"$date": "2024-01-01T00:00:00Z"}}}'
```

### Сжатие и архивирование

```bash
# Сжатие с gzip (каждый файл отдельно)
mongodump --gzip --out=/backup/dump

# Создание единого архива
mongodump --archive=/backup/myapp.archive

# Сжатый архив
mongodump --archive=/backup/myapp.archive.gz --gzip

# Бэкап напрямую в stdout (для pipe в другие команды)
mongodump --archive --gzip | aws s3 cp - s3://my-bucket/backup.archive.gz
```

### Опция --oplog для консистентных бэкапов

Для replica set критически важна опция `--oplog`, которая захватывает oplog-записи во время дампа, обеспечивая point-in-time консистентность:

```bash
# Консистентный бэкап replica set
mongodump --oplog --out=/backup/dump

# Восстановление с применением oplog
mongorestore --oplogReplay /backup/dump
```

**Важно**: `--oplog` работает только при подключении к replica set и создаёт файл `oplog.bson` в корне дампа.

### Основные опции mongorestore

```bash
# Восстановление в другую базу данных
mongorestore --nsFrom="myapp.*" --nsTo="myapp_restored.*" /backup/dump

# Удаление существующих данных перед восстановлением
mongorestore --drop /backup/dump

# Восстановление из архива
mongorestore --archive=/backup/myapp.archive.gz --gzip

# Восстановление конкретной коллекции
mongorestore --db=myapp --collection=users /backup/dump/myapp/users.bson

# Параллельное восстановление (ускорение)
mongorestore --numParallelCollections=4 /backup/dump

# Пропуск индексов (создать вручную позже)
mongorestore --noIndexRestore /backup/dump
```

### Преимущества и недостатки

**Преимущества:**
- Простота использования
- Гибкость (бэкап отдельных БД/коллекций)
- Портативность между версиями MongoDB
- Возможность фильтрации данных

**Недостатки:**
- Медленнее, чем снапшоты для больших БД
- Влияет на производительность кластера
- Не подходит для очень больших баз (терабайты)

---

## Снапшоты файловой системы

### Обзор

Снапшоты файловой системы — это быстрый способ создания бэкапа путём копирования файлов данных MongoDB на уровне блочного устройства. Это особенно эффективно для больших баз данных.

### Требования к консистентности

Для создания консистентного снапшота необходимо:

1. **Остановить запись** или использовать journaling
2. **Убедиться, что журнал и данные на одном томе** (или делать atomic snapshot обоих)
3. **Использовать fsync + lock** для гарантии консистентности

```javascript
// Блокировка записи перед снапшотом
db.fsyncLock()

// После создания снапшота - разблокировка
db.fsyncUnlock()
```

### LVM Snapshots (Linux)

LVM (Logical Volume Manager) позволяет создавать мгновенные снапшоты:

```bash
# 1. Блокировка MongoDB
mongo --eval "db.fsyncLock()"

# 2. Создание LVM снапшота
lvcreate --size 10G --snapshot --name mongodb-snap /dev/vg0/mongodb-data

# 3. Разблокировка MongoDB
mongo --eval "db.fsyncUnlock()"

# 4. Монтирование снапшота для копирования
mkdir /mnt/snapshot
mount /dev/vg0/mongodb-snap /mnt/snapshot

# 5. Копирование данных
cp -a /mnt/snapshot/* /backup/mongodb-$(date +%Y%m%d)/

# 6. Размонтирование и удаление снапшота
umount /mnt/snapshot
lvremove -f /dev/vg0/mongodb-snap
```

### AWS EBS Snapshots

```bash
# Скрипт для создания EBS снапшота

#!/bin/bash
VOLUME_ID="vol-0123456789abcdef0"
INSTANCE_ID="i-0123456789abcdef0"

# Блокировка MongoDB
mongo --eval "db.fsyncLock()"

# Создание снапшота
SNAPSHOT_ID=$(aws ec2 create-snapshot \
  --volume-id $VOLUME_ID \
  --description "MongoDB backup $(date +%Y-%m-%d)" \
  --query 'SnapshotId' \
  --output text)

echo "Created snapshot: $SNAPSHOT_ID"

# Разблокировка MongoDB
mongo --eval "db.fsyncUnlock()"

# Ожидание завершения снапшота
aws ec2 wait snapshot-completed --snapshot-ids $SNAPSHOT_ID
echo "Snapshot completed"
```

### Azure Managed Disk Snapshots

```bash
# Azure CLI команды
RESOURCE_GROUP="myResourceGroup"
DISK_NAME="mongodb-data-disk"
SNAPSHOT_NAME="mongodb-snapshot-$(date +%Y%m%d)"

# Блокировка MongoDB
mongo --eval "db.fsyncLock()"

# Создание снапшота
az snapshot create \
  --resource-group $RESOURCE_GROUP \
  --source $DISK_NAME \
  --name $SNAPSHOT_NAME

# Разблокировка MongoDB
mongo --eval "db.fsyncUnlock()"
```

### Google Cloud Persistent Disk Snapshots

```bash
# GCP команды
ZONE="us-central1-a"
DISK_NAME="mongodb-data"
SNAPSHOT_NAME="mongodb-snapshot-$(date +%Y%m%d)"

# Создание снапшота
gcloud compute disks snapshot $DISK_NAME \
  --zone=$ZONE \
  --snapshot-names=$SNAPSHOT_NAME \
  --storage-location=us
```

### Преимущества и недостатки

**Преимущества:**
- Очень быстрое создание (секунды)
- Минимальное влияние на производительность
- Подходит для очень больших БД
- Инкрементальные снапшоты в облаке

**Недостатки:**
- Требует блокировки или остановки записи
- Менее портативны между платформами
- Восстановление требует той же инфраструктуры
- Сложнее восстановить отдельные коллекции

---

## MongoDB Atlas Backup

MongoDB Atlas предоставляет полностью управляемые решения для резервного копирования.

### Cloud Backup (Рекомендуемый)

Cloud Backup использует нативные снапшоты облачных провайдеров:

**Возможности:**
- Автоматические снапшоты по расписанию
- Point-in-time recovery до последних 7 дней
- Queryable Snapshots (запросы к бэкапам)
- Восстановление в другой кластер

**Настройка через Atlas UI:**
1. Перейти в Cluster > Backup
2. Включить Cloud Backup
3. Настроить политику хранения

**Политики хранения (пример):**
```
Hourly snapshots:  хранить 24 часа
Daily snapshots:   хранить 7 дней
Weekly snapshots:  хранить 4 недели
Monthly snapshots: хранить 12 месяцев
```

### Legacy Backup (Устаревший)

Legacy Backup использует агенты для копирования данных:

- Доступен только для M10+ кластеров
- Постепенно заменяется Cloud Backup
- Поддерживает Continuous Backup

### Serverless Backup

Для Serverless instances:

- Автоматические ежедневные снапшоты
- Хранение 2 дня
- Восстановление через UI или API

### Восстановление в Atlas

```bash
# Через Atlas CLI
atlas backups restores start \
  --clusterName myCluster \
  --snapshotId 6123456789abcdef01234567 \
  --targetClusterName myCluster-restored

# Скачивание снапшота
atlas backups snapshots download \
  --clusterName myCluster \
  --snapshotId 6123456789abcdef01234567 \
  --out /backup/snapshot.tar.gz
```

---

## Point-in-Time Recovery (PITR)

### Как работает PITR

Point-in-Time Recovery позволяет восстановить данные на любой момент времени. Это работает благодаря **oplog** — операционному логу, который хранит все операции записи.

**Принцип работы:**
1. Создаётся полный бэкап (baseline)
2. Непрерывно сохраняется oplog
3. При восстановлении: восстанавливается бэкап + применяется oplog до нужного момента

### Настройка PITR для self-hosted MongoDB

```bash
# 1. Создание базового бэкапа с oplog
mongodump --oplog --out=/backup/baseline

# 2. Непрерывное копирование oplog (в отдельном процессе)
#!/bin/bash
while true; do
  mongodump \
    --db=local \
    --collection=oplog.rs \
    --out=/backup/oplog/$(date +%Y%m%d_%H%M%S) \
    --query="{\"ts\": {\"\$gt\": Timestamp($(date +%s), 0)}}"
  sleep 60  # Каждую минуту
done
```

### Восстановление на определённый момент

```bash
# 1. Восстановление базового бэкапа
mongorestore --drop /backup/baseline

# 2. Применение oplog до определённого времени
mongorestore \
  --oplogReplay \
  --oplogLimit="Timestamp(1704067200, 1)" \  # Unix timestamp
  /backup/oplog/
```

### Восстановление с использованием oplogReplay

```bash
# Указание точного времени (формат BSON timestamp)
mongorestore \
  --oplogReplay \
  --oplogLimit="1704067200:1" \
  /backup/dump

# Восстановление до определённой операции
# Формат: seconds:increment
# Можно найти нужный timestamp в oplog:
mongo --eval 'db.oplog.rs.find().sort({ts:-1}).limit(5).pretty()'
```

### PITR в MongoDB Atlas

В Atlas PITR доступен из коробки для Cloud Backup:

1. **Backup > Restores**
2. Выбрать **Point in Time**
3. Указать дату и время
4. Выбрать целевой кластер

**Ограничения:**
- Доступно за последние 7 дней (по умолчанию)
- Можно увеличить до 35 дней (M10+)
- Точность: до секунды

---

## Стратегии бэкапов

### Полные vs Инкрементальные бэкапы

**Полные бэкапы:**
- Содержат все данные
- Независимы друг от друга
- Занимают много места
- Долго создаются

**Инкрементальные бэкапы:**
- Только изменения с последнего бэкапа
- Быстрее создаются
- Меньше места
- Зависят от предыдущих бэкапов

### Стратегия 3-2-1

Классическая стратегия резервного копирования:

- **3** копии данных (1 оригинал + 2 бэкапа)
- **2** разных типа носителей
- **1** копия off-site (в другом дата-центре/регионе)

### Расписание бэкапов (пример)

```bash
# Crontab для mongodump

# Ежечасные инкрементальные (oplog)
0 * * * * /scripts/backup-oplog.sh

# Ежедневные полные в 2:00
0 2 * * * /scripts/backup-full.sh daily

# Еженедельные полные в воскресенье 3:00
0 3 * * 0 /scripts/backup-full.sh weekly

# Ежемесячные полные 1-го числа в 4:00
0 4 1 * * /scripts/backup-full.sh monthly
```

### Скрипт бэкапа с ротацией

```bash
#!/bin/bash
# /scripts/backup-full.sh

BACKUP_TYPE=${1:-daily}
BACKUP_DIR="/backup/mongodb"
DATE=$(date +%Y%m%d_%H%M%S)
MONGODB_URI="mongodb://backup_user:password@localhost:27017/?authSource=admin"

# Создание бэкапа
mongodump \
  --uri="$MONGODB_URI" \
  --oplog \
  --gzip \
  --archive="$BACKUP_DIR/$BACKUP_TYPE/mongodb-$DATE.archive.gz"

# Проверка успешности
if [ $? -eq 0 ]; then
  echo "Backup successful: mongodb-$DATE.archive.gz"

  # Ротация по типу
  case $BACKUP_TYPE in
    hourly)
      find $BACKUP_DIR/hourly -mtime +1 -delete  # Хранить 24 часа
      ;;
    daily)
      find $BACKUP_DIR/daily -mtime +7 -delete   # Хранить 7 дней
      ;;
    weekly)
      find $BACKUP_DIR/weekly -mtime +28 -delete # Хранить 4 недели
      ;;
    monthly)
      find $BACKUP_DIR/monthly -mtime +365 -delete # Хранить 1 год
      ;;
  esac
else
  echo "Backup FAILED!"
  # Отправить алерт
  curl -X POST "https://hooks.slack.com/..." -d '{"text":"MongoDB backup failed!"}'
  exit 1
fi
```

### Retention Policies

| Тип бэкапа | Частота | Хранение | Использование |
|------------|---------|----------|---------------|
| Hourly | Каждый час | 24 часа | Быстрое восстановление |
| Daily | Ежедневно | 7-30 дней | Стандартное восстановление |
| Weekly | Еженедельно | 4-12 недель | Долгосрочное хранение |
| Monthly | Ежемесячно | 12-24 месяца | Архив, compliance |
| Yearly | Ежегодно | 7+ лет | Регуляторные требования |

---

## Best Practices

### 1. Тестирование восстановления

**Регулярно проверяйте бэкапы!** Бэкап бесполезен, если его нельзя восстановить.

```bash
#!/bin/bash
# Скрипт тестирования восстановления

TEST_DB="backup_test_$(date +%s)"
BACKUP_FILE="/backup/mongodb/daily/latest.archive.gz"

# Восстановление в тестовую БД
mongorestore \
  --archive="$BACKUP_FILE" \
  --gzip \
  --nsFrom='myapp.*' \
  --nsTo="$TEST_DB.*" \
  --drop

# Проверка количества документов
ORIGINAL_COUNT=$(mongo myapp --eval "db.users.count()" --quiet)
RESTORED_COUNT=$(mongo $TEST_DB --eval "db.users.count()" --quiet)

if [ "$ORIGINAL_COUNT" -eq "$RESTORED_COUNT" ]; then
  echo "Backup verification PASSED: $RESTORED_COUNT documents"
else
  echo "Backup verification FAILED: expected $ORIGINAL_COUNT, got $RESTORED_COUNT"
  exit 1
fi

# Очистка
mongo $TEST_DB --eval "db.dropDatabase()"
```

### 2. Мониторинг бэкапов

**Метрики для отслеживания:**
- Размер бэкапов
- Время создания бэкапа
- Успешность/неуспешность
- Время с последнего успешного бэкапа

**Prometheus метрики (пример):**

```yaml
# prometheus-mongodb-backup-exporter
mongodb_backup_last_success_timestamp
mongodb_backup_duration_seconds
mongodb_backup_size_bytes
mongodb_backup_documents_count
```

**Alerting правила:**

```yaml
# Prometheus alerting rules
groups:
  - name: mongodb-backup
    rules:
      - alert: MongoDBBackupFailed
        expr: mongodb_backup_last_success_timestamp < (time() - 86400)
        for: 1h
        labels:
          severity: critical
        annotations:
          summary: "MongoDB backup failed"
          description: "No successful backup in the last 24 hours"

      - alert: MongoDBBackupSlow
        expr: mongodb_backup_duration_seconds > 3600
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MongoDB backup taking too long"
```

### 3. Безопасность бэкапов

**Шифрование:**

```bash
# Шифрование бэкапа с помощью GPG
mongodump --archive --gzip | \
  gpg --symmetric --cipher-algo AES256 \
  --output /backup/mongodb-encrypted.archive.gz.gpg

# Расшифровка
gpg --decrypt /backup/mongodb-encrypted.archive.gz.gpg | \
  mongorestore --archive --gzip
```

**Шифрование с помощью OpenSSL:**

```bash
# Шифрование
mongodump --archive --gzip | \
  openssl enc -aes-256-cbc -salt -pbkdf2 \
  -out /backup/mongodb-encrypted.archive.gz.enc

# Расшифровка
openssl enc -aes-256-cbc -d -pbkdf2 \
  -in /backup/mongodb-encrypted.archive.gz.enc | \
  mongorestore --archive --gzip
```

**Контроль доступа:**

```bash
# Права доступа к директории бэкапов
chmod 700 /backup/mongodb
chown mongodb:mongodb /backup/mongodb

# Отдельный пользователь для бэкапов
use admin
db.createUser({
  user: "backup_user",
  pwd: "secure_password",
  roles: [
    { role: "backup", db: "admin" },
    { role: "restore", db: "admin" }
  ]
})
```

### 4. Документация и процедуры

**Документируйте:**
- Расположение бэкапов
- Расписание бэкапов
- Процедуру восстановления (с примерами команд)
- Контакты ответственных лиц
- RTO и RPO требования

**Runbook восстановления (пример):**

```markdown
## Процедура восстановления MongoDB

### Полное восстановление
1. Остановить приложения, подключённые к MongoDB
2. Найти последний бэкап: `ls -la /backup/mongodb/daily/`
3. Выполнить восстановление:
   ```
   mongorestore --drop --archive=/backup/mongodb/daily/latest.archive.gz --gzip
   ```
4. Проверить данные: `mongo --eval "db.stats()"`
5. Запустить приложения

### Point-in-Time Recovery
1. Определить целевой timestamp
2. Восстановить базовый бэкап
3. Применить oplog до нужного момента
```

### 5. Оптимизация производительности

```bash
# Параллельное создание бэкапа
mongodump --numParallelCollections=4 --out=/backup/dump

# Использование read preference secondary
mongodump \
  --uri="mongodb://rs0/myapp?readPreference=secondary" \
  --out=/backup/dump

# Сжатие на лету для экономии I/O
mongodump --archive --gzip | \
  aws s3 cp - s3://my-bucket/backup-$(date +%Y%m%d).archive.gz
```

---

## Сравнение методов бэкапа

| Критерий | mongodump | Снапшоты FS | Atlas Backup |
|----------|-----------|-------------|--------------|
| Скорость создания | Медленно | Быстро | Быстро |
| Влияние на production | Среднее | Низкое | Минимальное |
| Гранулярность | Коллекция | Весь volume | Кластер |
| PITR | С oplog | Нет | Да |
| Портативность | Высокая | Низкая | Средняя |
| Сложность настройки | Низкая | Средняя | Очень низкая |
| Стоимость | Бесплатно | Зависит | Включено в Atlas |

---

## Полезные ресурсы

- [MongoDB Backup Methods](https://docs.mongodb.com/manual/core/backups/)
- [mongodump Documentation](https://docs.mongodb.com/database-tools/mongodump/)
- [MongoDB Atlas Backup](https://docs.atlas.mongodb.com/backup/)
- [Point-in-Time Recovery](https://docs.mongodb.com/manual/tutorial/restore-replica-set-from-backup/)
