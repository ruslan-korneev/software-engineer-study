# SSH и SCP - безопасное удалённое подключение и передача файлов

## SSH - Secure Shell

### Что такое SSH?

**SSH** (Secure Shell) - криптографический сетевой протокол для безопасного удалённого доступа к системам. Обеспечивает:

- Шифрование всего трафика
- Аутентификацию по ключам или паролю
- Туннелирование других протоколов
- Передачу файлов (SCP, SFTP)

### Базовое подключение

```bash
# Подключиться к серверу
ssh user@hostname

# Указать порт (по умолчанию 22)
ssh -p 2222 user@hostname

# Подключиться как текущий пользователь
ssh hostname

# Выполнить команду и выйти
ssh user@hostname "ls -la"
ssh user@hostname "df -h && free -m"
```

### Аутентификация по ключам

#### Генерация ключей

```bash
# Создать пару ключей (RSA 4096 бит)
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

# Создать ключ Ed25519 (современный, рекомендуется)
ssh-keygen -t ed25519 -C "your_email@example.com"

# Указать имя файла
ssh-keygen -t ed25519 -f ~/.ssh/myserver_key

# Ключи сохраняются в:
# ~/.ssh/id_ed25519      - приватный ключ (никому не давать!)
# ~/.ssh/id_ed25519.pub  - публичный ключ (можно распространять)
```

#### Копирование ключа на сервер

```bash
# Автоматически (рекомендуется)
ssh-copy-id user@hostname

# С указанием ключа
ssh-copy-id -i ~/.ssh/mykey.pub user@hostname

# Вручную (если ssh-copy-id недоступен)
cat ~/.ssh/id_ed25519.pub | ssh user@hostname "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

#### Использование ключей

```bash
# Подключиться с конкретным ключом
ssh -i ~/.ssh/myserver_key user@hostname

# SSH-агент (хранит ключи в памяти)
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# Список добавленных ключей
ssh-add -l
```

### Конфигурационный файл SSH

Файл `~/.ssh/config` упрощает подключение:

```bash
# ~/.ssh/config
Host myserver
    HostName 192.168.1.100
    User admin
    Port 2222
    IdentityFile ~/.ssh/myserver_key

Host production
    HostName prod.example.com
    User deploy
    IdentityFile ~/.ssh/deploy_key
    ForwardAgent yes

Host github.com
    IdentityFile ~/.ssh/github_key

# Общие настройки для всех хостов
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    AddKeysToAgent yes
```

Теперь можно подключаться просто:
```bash
ssh myserver
ssh production
```

### Полезные опции SSH

```bash
# Подробный вывод (отладка)
ssh -v user@hostname      # verbose
ssh -vv user@hostname     # очень подробно
ssh -vvv user@hostname    # максимально подробно

# Сжатие (полезно на медленных соединениях)
ssh -C user@hostname

# Фоновый режим
ssh -f user@hostname "long-command"

# Без выполнения команды (только туннель)
ssh -N user@hostname

# Отключить проверку хоста (опасно!)
ssh -o StrictHostKeyChecking=no user@hostname

# X11 forwarding (запуск GUI приложений)
ssh -X user@hostname
firefox  # откроется локально
```

### Туннелирование (Port Forwarding)

#### Локальное перенаправление

Доступ к удалённому сервису через локальный порт:

```bash
# Синтаксис: -L локальный_порт:целевой_хост:целевой_порт
ssh -L 8080:localhost:80 user@server

# Теперь http://localhost:8080 -> server:80

# Доступ к базе данных через туннель
ssh -L 3306:localhost:3306 user@dbserver
mysql -h 127.0.0.1 -P 3306 -u root

# Доступ к внутреннему серверу
ssh -L 8080:internal-server:80 user@gateway
```

#### Удалённое перенаправление

Открыть доступ к локальному сервису извне:

```bash
# Синтаксис: -R удалённый_порт:локальный_хост:локальный_порт
ssh -R 8080:localhost:3000 user@server

# Теперь server:8080 -> localhost:3000

# Открыть локальный сайт для демонстрации
ssh -R 80:localhost:3000 user@server
```

#### Динамическое перенаправление (SOCKS прокси)

```bash
# Создать SOCKS5 прокси на локальном порту
ssh -D 1080 user@server

# Настроить браузер на прокси localhost:1080
# Весь трафик пойдёт через сервер
```

#### Туннель в фоне

```bash
# Создать туннель и уйти в фон
ssh -f -N -L 8080:localhost:80 user@server

# Проверить
ss -tlnp | grep 8080

# Завершить
kill $(pgrep -f "ssh -f -N -L")
```

### Jump Host (Bastion)

Подключение через промежуточный сервер:

```bash
# Через -J (ProxyJump)
ssh -J user@bastion user@internal-server

# Несколько хопов
ssh -J user@bastion1,user@bastion2 user@target

# В конфигурации
Host internal
    HostName 10.0.0.5
    User admin
    ProxyJump bastion

Host bastion
    HostName bastion.example.com
    User jump
```

## SCP - Secure Copy

### Что такое SCP?

**SCP** (Secure Copy Protocol) - копирование файлов через SSH. Безопасно, но не поддерживает продолжение передачи.

### Базовое использование

```bash
# Копировать файл на сервер
scp file.txt user@server:/path/to/destination/

# Копировать с сервера
scp user@server:/path/to/file.txt ./

# Копировать директорию рекурсивно
scp -r directory/ user@server:/path/

# Указать порт
scp -P 2222 file.txt user@server:/path/

# Копировать с указанием ключа
scp -i ~/.ssh/mykey file.txt user@server:/path/
```

### Опции SCP

```bash
# Сохранить атрибуты (время, права)
scp -p file.txt user@server:/path/

# Сжатие
scp -C largefile.tar user@server:/path/

# Ограничить скорость (Kbit/s)
scp -l 1000 file.txt user@server:/path/

# Тихий режим
scp -q file.txt user@server:/path/

# Подробный вывод
scp -v file.txt user@server:/path/
```

### Копирование между серверами

```bash
# Напрямую между серверами
scp user1@server1:/file user2@server2:/path/

# Через локальную машину
scp -3 user1@server1:/file user2@server2:/path/
```

## SFTP - SSH File Transfer Protocol

### Что такое SFTP?

**SFTP** - интерактивный протокол передачи файлов через SSH. Более функционален чем SCP.

### Интерактивный режим

```bash
# Подключиться
sftp user@server

# Команды в сессии
sftp> pwd           # директория на сервере
sftp> lpwd          # локальная директория
sftp> ls            # список файлов на сервере
sftp> lls           # локальный список
sftp> cd /path      # перейти на сервере
sftp> lcd /path     # перейти локально
sftp> get file      # скачать
sftp> put file      # загрузить
sftp> mget *.txt    # скачать несколько
sftp> mput *.txt    # загрузить несколько
sftp> mkdir dir     # создать директорию
sftp> rm file       # удалить файл
sftp> bye           # выйти
```

### Неинтерактивный режим

```bash
# Выполнить команды из файла
sftp -b commands.txt user@server

# Одна команда
sftp user@server <<< "get /path/to/file"

# Скрипт
sftp user@server << EOF
cd /var/www
get index.html
bye
EOF
```

## rsync через SSH

**rsync** - мощный инструмент для синхронизации, часто используется с SSH:

```bash
# Синхронизировать директорию на сервер
rsync -avz /local/dir/ user@server:/remote/dir/

# С сервера
rsync -avz user@server:/remote/dir/ /local/dir/

# Показать прогресс
rsync -avz --progress /local/ user@server:/remote/

# Удалять лишние файлы на приёмнике
rsync -avz --delete /local/ user@server:/remote/

# Через нестандартный порт
rsync -avz -e "ssh -p 2222" /local/ user@server:/remote/

# Сухой прогон (показать что будет сделано)
rsync -avzn /local/ user@server:/remote/
```

## Безопасность SSH

### Настройка сервера (/etc/ssh/sshd_config)

```bash
# Отключить вход по паролю
PasswordAuthentication no

# Отключить root логин
PermitRootLogin no

# Разрешить только определённых пользователей
AllowUsers admin deploy

# Изменить порт
Port 2222

# Использовать только протокол 2
Protocol 2

# Ограничить попытки аутентификации
MaxAuthTries 3

# После изменений
sudo systemctl restart sshd
```

### SSH hardening

```bash
# Использовать только сильные алгоритмы
# /etc/ssh/sshd_config
KexAlgorithms curve25519-sha256@libssh.org,diffie-hellman-group-exchange-sha256
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com

# Fail2Ban для защиты от брутфорса
sudo apt install fail2ban
```

### Известные хосты

```bash
# Файл ~/.ssh/known_hosts содержит отпечатки серверов

# Получить отпечаток сервера
ssh-keyscan server.example.com

# Добавить в known_hosts
ssh-keyscan server.example.com >> ~/.ssh/known_hosts

# Удалить устаревший ключ
ssh-keygen -R server.example.com
```

## Практические примеры

### Автоматизация резервного копирования

```bash
#!/bin/bash
# Бекап через rsync
rsync -avz --delete \
    --exclude='.cache' \
    --exclude='node_modules' \
    /home/user/ user@backup-server:/backups/$(hostname)/

# Логирование
echo "Backup completed at $(date)" >> /var/log/backup.log
```

### SSH с автоматическим восстановлением

```bash
# autossh поддерживает соединение
sudo apt install autossh

# Туннель с автовосстановлением
autossh -M 0 -f -N -L 8080:localhost:80 user@server
```

### Мультиплексирование соединений

```bash
# ~/.ssh/config
Host *
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 600

# Создать директорию
mkdir -p ~/.ssh/sockets

# Первое подключение создаёт сокет
# Последующие используют его (быстрее)
```

## Резюме команд

| Задача | Команда |
|--------|---------|
| Подключиться | `ssh user@host` |
| Другой порт | `ssh -p 2222 user@host` |
| С ключом | `ssh -i ~/.ssh/key user@host` |
| Выполнить команду | `ssh user@host "command"` |
| Локальный туннель | `ssh -L 8080:localhost:80 user@host` |
| SOCKS прокси | `ssh -D 1080 user@host` |
| Копировать на сервер | `scp file user@host:/path/` |
| Копировать с сервера | `scp user@host:/file ./` |
| Синхронизировать | `rsync -avz /local/ user@host:/remote/` |
| Генерация ключа | `ssh-keygen -t ed25519` |
| Копировать ключ | `ssh-copy-id user@host` |
