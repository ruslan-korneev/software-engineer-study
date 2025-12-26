# Управление PostgreSQL с помощью systemd

## Введение

**systemd** — это система инициализации и менеджер служб для Linux, который стал стандартом де-факто в большинстве современных дистрибутивов (Ubuntu, Debian, CentOS, RHEL, Fedora). systemd обеспечивает управление жизненным циклом служб, включая PostgreSQL, и предоставляет мощные инструменты для мониторинга, логирования и автоматического перезапуска.

## Основные концепции systemd

### Unit-файлы

systemd использует **unit-файлы** для описания служб. Для PostgreSQL unit-файл обычно находится в:
- `/lib/systemd/system/postgresql.service` — базовый файл
- `/etc/systemd/system/postgresql.service` — переопределения администратора

### Структура unit-файла PostgreSQL

```ini
[Unit]
Description=PostgreSQL database server
Documentation=man:postgres(1)
After=network.target

[Service]
Type=notify
User=postgres
ExecStart=/usr/lib/postgresql/15/bin/postgres -D /var/lib/postgresql/15/main
ExecReload=/bin/kill -HUP $MAINPID
KillMode=mixed
KillSignal=SIGINT
TimeoutSec=infinity

[Install]
WantedBy=multi-user.target
```

## Основные команды управления

### Запуск и остановка

```bash
# Запуск PostgreSQL
sudo systemctl start postgresql

# Остановка PostgreSQL
sudo systemctl stop postgresql

# Перезапуск PostgreSQL
sudo systemctl restart postgresql

# Мягкий перезапуск (перечитывание конфигурации без разрыва соединений)
sudo systemctl reload postgresql
```

### Управление автозапуском

```bash
# Включить автозапуск при загрузке системы
sudo systemctl enable postgresql

# Отключить автозапуск
sudo systemctl disable postgresql

# Проверить статус автозапуска
systemctl is-enabled postgresql
```

### Проверка статуса

```bash
# Подробный статус службы
sudo systemctl status postgresql

# Проверка, запущена ли служба
systemctl is-active postgresql

# Проверка на наличие ошибок
systemctl is-failed postgresql
```

### Пример вывода статуса

```
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled)
     Active: active (exited) since Mon 2024-01-15 10:30:00 UTC
   Main PID: 1234 (postgres)
      Tasks: 7 (limit: 4915)
     Memory: 32.0M
        CPU: 1.234s
     CGroup: /system.slice/postgresql.service
             ├─1234 /usr/lib/postgresql/15/bin/postgres -D /var/lib/postgresql/15/main
             ├─1236 postgres: checkpointer
             ├─1237 postgres: background writer
             └─1238 postgres: walwriter
```

## Работа с несколькими экземплярами

В Debian/Ubuntu PostgreSQL часто устанавливается с поддержкой нескольких кластеров (инстансов):

```bash
# Управление конкретным кластером
sudo systemctl start postgresql@15-main
sudo systemctl stop postgresql@15-main
sudo systemctl status postgresql@15-main

# Управление всеми кластерами
sudo systemctl start postgresql
sudo systemctl stop postgresql
```

### Создание нового экземпляра

```bash
# Создание нового кластера
sudo pg_createcluster 15 secondary

# Запуск нового кластера
sudo systemctl start postgresql@15-secondary

# Включение автозапуска для нового кластера
sudo systemctl enable postgresql@15-secondary
```

## Просмотр логов

systemd интегрирован с **journald** — системой логирования:

```bash
# Просмотр логов PostgreSQL
sudo journalctl -u postgresql

# Логи в реальном времени
sudo journalctl -u postgresql -f

# Логи за последний час
sudo journalctl -u postgresql --since "1 hour ago"

# Логи за определенную дату
sudo journalctl -u postgresql --since "2024-01-15" --until "2024-01-16"

# Только ошибки
sudo journalctl -u postgresql -p err

# Последние 100 строк
sudo journalctl -u postgresql -n 100
```

## Настройка служебного файла

### Создание override-файла

Не редактируйте оригинальный unit-файл — используйте override:

```bash
# Создание директории и файла override
sudo systemctl edit postgresql
```

Это откроет редактор и создаст файл `/etc/systemd/system/postgresql.service.d/override.conf`.

### Примеры настроек

```ini
[Service]
# Увеличение лимитов файловых дескрипторов
LimitNOFILE=65536

# Увеличение лимита процессов
LimitNPROC=65536

# Изменение приоритета I/O
IOSchedulingClass=best-effort
IOSchedulingPriority=0

# Настройка OOM killer (защита от убийства при нехватке памяти)
OOMScoreAdjust=-900

# Переменные окружения
Environment="PGDATA=/var/lib/postgresql/15/main"

# Автоматический перезапуск при сбое
Restart=on-failure
RestartSec=5s
```

### Применение изменений

```bash
# Перезагрузка конфигурации systemd
sudo systemctl daemon-reload

# Перезапуск службы для применения изменений
sudo systemctl restart postgresql
```

## Зависимости и порядок запуска

### Настройка зависимостей

```ini
[Unit]
Description=PostgreSQL database server
After=network.target
After=network-online.target
Wants=network-online.target

# Запуск после монтирования файловых систем
After=local-fs.target

# Зависимость от другой службы
Requires=some-storage.service
After=some-storage.service
```

### Проверка зависимостей

```bash
# Показать зависимости службы
systemctl list-dependencies postgresql

# Показать обратные зависимости (кто зависит от PostgreSQL)
systemctl list-dependencies postgresql --reverse
```

## Best Practices

### 1. Используйте reload вместо restart когда возможно

```bash
# Для изменений в postgresql.conf (большинство параметров)
sudo systemctl reload postgresql

# restart нужен только для параметров, требующих перезапуска
# (shared_buffers, max_connections и др.)
sudo systemctl restart postgresql
```

### 2. Настройте мониторинг

```bash
# Создание таймера для проверки здоровья
# /etc/systemd/system/postgresql-health.timer

[Unit]
Description=PostgreSQL Health Check Timer

[Timer]
OnBootSec=5min
OnUnitActiveSec=1min

[Install]
WantedBy=timers.target
```

### 3. Настройте автоматический перезапуск

```ini
[Service]
Restart=on-failure
RestartSec=10s
StartLimitIntervalSec=60s
StartLimitBurst=3
```

### 4. Защита от OOM Killer

```ini
[Service]
OOMScoreAdjust=-1000  # Максимальная защита
```

## Типичные ошибки

### 1. Редактирование оригинального unit-файла

**Неправильно:**
```bash
sudo vim /lib/systemd/system/postgresql.service
```

**Правильно:**
```bash
sudo systemctl edit postgresql
```

### 2. Забыть daemon-reload после изменений

```bash
# После любых изменений в unit-файлах
sudo systemctl daemon-reload
```

### 3. Использование restart вместо reload

Restart прерывает все соединения. Используйте reload для:
- Изменения `postgresql.conf` (большинство параметров)
- Изменения `pg_hba.conf`

### 4. Игнорирование зависимостей

При использовании network storage (NFS, iSCSI) убедитесь, что PostgreSQL запускается после монтирования:

```ini
[Unit]
After=remote-fs.target
Requires=remote-fs.target
```

## Полезные команды для отладки

```bash
# Анализ времени запуска
systemd-analyze blame | grep postgresql

# Проверка синтаксиса unit-файла
systemd-analyze verify /etc/systemd/system/postgresql.service.d/override.conf

# Показать все настройки службы
systemctl show postgresql

# Показать изменения в override
systemctl cat postgresql
```

## Интеграция с другими инструментами

### Ansible

```yaml
- name: Ensure PostgreSQL is running and enabled
  ansible.builtin.systemd:
    name: postgresql
    state: started
    enabled: yes
    daemon_reload: yes
```

### Terraform (через provisioner)

```hcl
provisioner "remote-exec" {
  inline = [
    "sudo systemctl enable postgresql",
    "sudo systemctl start postgresql"
  ]
}
```

## Заключение

systemd предоставляет мощный и стандартизированный способ управления PostgreSQL в современных Linux-системах. Основные преимущества:
- Централизованное управление всеми службами
- Интеграция с journald для логирования
- Автоматический перезапуск при сбоях
- Управление зависимостями между службами
- Поддержка нескольких экземпляров PostgreSQL

Для production-систем рекомендуется настроить override-файлы с увеличенными лимитами ресурсов и защитой от OOM killer.
