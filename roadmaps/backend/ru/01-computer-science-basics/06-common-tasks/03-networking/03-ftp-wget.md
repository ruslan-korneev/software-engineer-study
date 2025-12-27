# FTP, wget и curl - передача файлов

## FTP - File Transfer Protocol

### Что такое FTP?

**FTP** (File Transfer Protocol) - протокол для передачи файлов между компьютерами по сети. Работает по модели клиент-сервер на портах 20 (данные) и 21 (управление).

**Важно:** FTP передаёт данные и пароли в открытом виде. Для безопасной передачи используйте SFTP или SCP.

### Командный клиент ftp

```bash
# Подключиться к FTP-серверу
ftp ftp.example.com

# Подключиться с указанием пользователя
ftp user@ftp.example.com

# Пример сессии:
# Connected to ftp.example.com.
# 220 Welcome to FTP server.
# Name: username
# 331 Please specify the password.
# Password: ****
# 230 Login successful.
# ftp>
```

### Команды FTP-клиента

```bash
# Навигация
pwd                 # показать текущую директорию на сервере
lpwd                # показать локальную директорию
cd /path            # перейти в директорию на сервере
lcd /path           # перейти в локальную директорию
ls                  # список файлов на сервере
!ls                 # список локальных файлов

# Передача файлов
get file.txt        # скачать файл
mget *.txt          # скачать несколько файлов
put file.txt        # загрузить файл
mput *.txt          # загрузить несколько файлов

# Режим передачи
ascii               # текстовый режим
binary              # бинарный режим (для всех файлов кроме текста)

# Управление
delete file.txt     # удалить файл
mkdir dirname       # создать директорию
rmdir dirname       # удалить директорию
rename old new      # переименовать

# Другое
hash                # показывать прогресс
prompt              # вкл/выкл подтверждения для mget/mput
bye / quit          # выйти
```

### lftp - продвинутый FTP-клиент

```bash
# Установка
sudo apt install lftp

# Подключение
lftp ftp://user:password@ftp.example.com

# Скачать директорию рекурсивно
lftp -e "mirror /remote/dir /local/dir; quit" ftp://user:pass@host

# Загрузить директорию
lftp -e "mirror -R /local/dir /remote/dir; quit" ftp://user:pass@host

# Продолжить прерванную загрузку
lftp -e "get -c largefile.zip; quit" ftp://host
```

## wget - загрузка файлов из командной строки

### Что такое wget?

**wget** - утилита для загрузки файлов по HTTP, HTTPS и FTP. Поддерживает докачку, рекурсивную загрузку и работу в фоновом режиме.

### Базовое использование

```bash
# Скачать файл
wget https://example.com/file.zip

# Скачать с другим именем
wget -O newname.zip https://example.com/file.zip

# Скачать в директорию
wget -P /tmp https://example.com/file.zip

# Продолжить прерванную загрузку
wget -c https://example.com/largefile.zip

# Скачать тихо (без вывода)
wget -q https://example.com/file.zip

# Показывать прогресс-бар
wget --progress=bar https://example.com/file.zip
```

### Загрузка нескольких файлов

```bash
# Из списка URL в файле
wget -i urls.txt

# Несколько URL в командной строке
wget https://example.com/file1.zip https://example.com/file2.zip
```

### Рекурсивная загрузка (зеркалирование)

```bash
# Скачать весь сайт
wget -r https://example.com/

# С ограничением глубины
wget -r -l 2 https://example.com/

# Зеркало сайта для офлайн просмотра
wget --mirror --convert-links --adjust-extension --page-requisites \
     --no-parent https://example.com/

# Сокращённая версия
wget -mkEpnp https://example.com/
```

### Опции загрузки

```bash
# Ограничение скорости
wget --limit-rate=1M https://example.com/file.zip

# Таймаут
wget --timeout=30 https://example.com/file.zip

# Количество попыток
wget --tries=10 https://example.com/file.zip

# Повторять бесконечно
wget --tries=0 https://example.com/file.zip

# Игнорировать ошибки SSL
wget --no-check-certificate https://example.com/file.zip

# Использовать прокси
wget -e use_proxy=yes -e http_proxy=proxy:8080 https://example.com/

# Загрузка в фоне
wget -b https://example.com/largefile.zip
# Лог сохраняется в wget-log
tail -f wget-log
```

### Аутентификация

```bash
# HTTP Basic Auth
wget --user=username --password=password https://example.com/file.zip

# FTP
wget ftp://user:pass@ftp.example.com/file.zip

# Сохранить пароль в .wgetrc (для безопасности)
echo "user=username" >> ~/.wgetrc
echo "password=password" >> ~/.wgetrc
```

### Работа с заголовками

```bash
# Пользовательский User-Agent
wget --user-agent="Mozilla/5.0" https://example.com/

# Дополнительные заголовки
wget --header="Accept: application/json" https://api.example.com/

# Отправить cookies
wget --load-cookies cookies.txt https://example.com/

# Сохранить cookies
wget --save-cookies cookies.txt https://example.com/
```

## curl - универсальный инструмент передачи данных

### Что такое curl?

**curl** (Client URL) - инструмент для передачи данных с использованием различных протоколов: HTTP, HTTPS, FTP, SFTP, SCP и многих других.

### Базовое использование

```bash
# Получить содержимое URL (вывод в stdout)
curl https://example.com/

# Сохранить в файл с оригинальным именем
curl -O https://example.com/file.zip

# Сохранить с заданным именем
curl -o myfile.zip https://example.com/file.zip

# Продолжить загрузку
curl -C - -O https://example.com/largefile.zip

# Показать прогресс
curl -# -O https://example.com/file.zip

# Тихий режим
curl -s https://example.com/
```

### HTTP методы

```bash
# GET (по умолчанию)
curl https://api.example.com/users

# POST с данными
curl -X POST -d "name=John&age=30" https://api.example.com/users

# POST с JSON
curl -X POST \
     -H "Content-Type: application/json" \
     -d '{"name":"John","age":30}' \
     https://api.example.com/users

# PUT
curl -X PUT -d "name=Jane" https://api.example.com/users/1

# DELETE
curl -X DELETE https://api.example.com/users/1

# PATCH
curl -X PATCH -d '{"age":31}' https://api.example.com/users/1
```

### Загрузка файлов

```bash
# Загрузить файл (multipart/form-data)
curl -F "file=@/path/to/file.pdf" https://example.com/upload

# Несколько файлов
curl -F "file1=@file1.pdf" -F "file2=@file2.pdf" https://example.com/upload

# С дополнительными полями
curl -F "file=@file.pdf" -F "description=My file" https://example.com/upload
```

### Заголовки

```bash
# Добавить заголовок
curl -H "Authorization: Bearer token123" https://api.example.com/

# Несколько заголовков
curl -H "Accept: application/json" \
     -H "Authorization: Bearer token123" \
     https://api.example.com/

# Показать заголовки ответа
curl -I https://example.com/
curl --head https://example.com/

# Показать всё (заголовки + тело)
curl -i https://example.com/

# Подробный вывод (включая запрос)
curl -v https://example.com/
```

### Аутентификация

```bash
# Basic Auth
curl -u username:password https://example.com/

# Bearer Token
curl -H "Authorization: Bearer token123" https://api.example.com/

# Digest Auth
curl --digest -u username:password https://example.com/
```

### Cookies

```bash
# Отправить cookie
curl -b "session=abc123" https://example.com/

# Загрузить cookies из файла
curl -b cookies.txt https://example.com/

# Сохранить cookies в файл
curl -c cookies.txt https://example.com/login

# Использовать и сохранять
curl -b cookies.txt -c cookies.txt https://example.com/
```

### SSL/TLS

```bash
# Игнорировать ошибки сертификата
curl -k https://self-signed.example.com/

# Использовать клиентский сертификат
curl --cert client.crt --key client.key https://example.com/

# Указать CA bundle
curl --cacert /path/to/ca-bundle.crt https://example.com/
```

### Прокси

```bash
# HTTP прокси
curl -x proxy.example.com:8080 https://example.com/

# SOCKS5 прокси
curl --socks5 localhost:1080 https://example.com/

# С аутентификацией
curl -x user:pass@proxy.example.com:8080 https://example.com/
```

### Полезные опции

```bash
# Следовать редиректам
curl -L https://example.com/redirect

# Ограничить скорость
curl --limit-rate 1M https://example.com/file.zip

# Таймауты
curl --connect-timeout 10 --max-time 300 https://example.com/

# Повторные попытки
curl --retry 5 --retry-delay 10 https://example.com/

# Показать только код ответа
curl -s -o /dev/null -w "%{http_code}" https://example.com/

# Показать время выполнения
curl -w "\nTime: %{time_total}s\n" https://example.com/
```

### JSON обработка

```bash
# С форматированием через jq
curl -s https://api.example.com/users | jq .

# Извлечь поле
curl -s https://api.example.com/users | jq '.[0].name'
```

## Сравнение wget и curl

| Функция | wget | curl |
|---------|------|------|
| Рекурсивная загрузка | Да | Нет |
| Продолжение загрузки | `-c` | `-C -` |
| REST API работа | Ограниченно | Отлично |
| Протоколы | HTTP, HTTPS, FTP | 25+ протоколов |
| Загрузка файлов | Нет | `-F` |
| Вывод в stdout | `-O -` | По умолчанию |
| Прогресс | По умолчанию | `-#` |

## Практические примеры

### Скрипт резервного копирования на FTP

```bash
#!/bin/bash
FTP_HOST="ftp.example.com"
FTP_USER="user"
FTP_PASS="pass"
LOCAL_DIR="/backup"
REMOTE_DIR="/backups"

lftp -u $FTP_USER,$FTP_PASS $FTP_HOST <<EOF
mirror -R $LOCAL_DIR $REMOTE_DIR
bye
EOF
```

### Загрузка с авторизацией и API

```bash
# Получить токен
TOKEN=$(curl -s -X POST \
    -d '{"username":"user","password":"pass"}' \
    -H "Content-Type: application/json" \
    https://api.example.com/login | jq -r '.token')

# Использовать токен
curl -H "Authorization: Bearer $TOKEN" \
    https://api.example.com/protected
```

### Проверка доступности URL

```bash
#!/bin/bash
URL="https://example.com"
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" $URL)

if [ "$HTTP_CODE" = "200" ]; then
    echo "Site is up"
else
    echo "Site returned $HTTP_CODE"
fi
```

## Резюме команд

| Задача | wget | curl |
|--------|------|------|
| Скачать файл | `wget URL` | `curl -O URL` |
| Скачать с именем | `wget -O name URL` | `curl -o name URL` |
| Продолжить | `wget -c URL` | `curl -C - -O URL` |
| POST запрос | - | `curl -X POST -d "data" URL` |
| Заголовки | `wget --header=` | `curl -H ""` |
| Аутентификация | `wget --user --password` | `curl -u user:pass` |
| Рекурсивно | `wget -r URL` | - |
| Только заголовки | `wget --spider` | `curl -I URL` |
