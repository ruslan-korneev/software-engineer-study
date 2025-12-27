# Here Documents (Here-документы)

## Что такое Here Documents

**Here Document** (heredoc) - это способ передачи многострочного текста команде. Это удобный инструмент для:
- Создания многострочных строк
- Передачи входных данных программам
- Генерации файлов конфигурации
- Встраивания кода других языков в скрипт

## Базовый синтаксис

```bash
command << DELIMITER
текст
может быть
многострочным
DELIMITER
```

Где `DELIMITER` - это любое слово (маркер), обозначающее начало и конец текста. Часто используют: `EOF`, `END`, `HEREDOC`.

### Простой пример

```bash
cat << EOF
Привет, мир!
Это многострочный текст.
Третья строка.
EOF
```

Вывод:
```
Привет, мир!
Это многострочный текст.
Третья строка.
```

## Варианты синтаксиса

### 1. Стандартный heredoc (с раскрытием переменных)

```bash
name="Иван"
cat << EOF
Привет, $name!
Сегодня $(date +%Y-%m-%d)
Ваша домашняя директория: $HOME
EOF
```

Вывод:
```
Привет, Иван!
Сегодня 2024-01-15
Ваша домашняя директория: /home/user
```

### 2. Quoted heredoc (без раскрытия)

Если маркер в кавычках, переменные НЕ раскрываются:

```bash
name="Иван"
cat << 'EOF'
Привет, $name!
Сегодня $(date +%Y-%m-%d)
Переменная записывается как $VAR
EOF
```

Вывод:
```
Привет, $name!
Сегодня $(date +%Y-%m-%d)
Переменная записывается как $VAR
```

### 3. Heredoc с отступами (<<-)

Оператор `<<-` игнорирует начальные табуляции:

```bash
if true; then
    cat <<- EOF
		Этот текст будет выведен
		без начальных табуляций
		EOF
fi
```

**Важно:** работает только с табуляциями (tab), не с пробелами!

## Запись в файл

### Создание файла

```bash
cat << EOF > config.txt
# Конфигурационный файл
server=localhost
port=8080
debug=true
EOF
```

### Добавление в файл

```bash
cat << EOF >> config.txt
# Дополнительные настройки
timeout=30
EOF
```

## Практические примеры

### Генерация HTML

```bash
#!/bin/bash

title="Моя страница"
content="Привет, мир!"

cat << EOF > page.html
<!DOCTYPE html>
<html>
<head>
    <title>$title</title>
    <meta charset="utf-8">
</head>
<body>
    <h1>$title</h1>
    <p>$content</p>
    <p>Сгенерировано: $(date)</p>
</body>
</html>
EOF
```

### Генерация SQL

```bash
#!/bin/bash

db_name="myapp"
table_name="users"

mysql << EOF
USE $db_name;
CREATE TABLE IF NOT EXISTS $table_name (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
EOF
```

### SSH с командами

```bash
#!/bin/bash

remote_host="server.example.com"

ssh "$remote_host" << EOF
cd /var/log
ls -la
tail -n 20 syslog
df -h
EOF
```

### Создание скрипта из скрипта

```bash
#!/bin/bash

cat << 'EOF' > /tmp/backup.sh
#!/bin/bash
# Автоматически сгенерированный скрипт бэкапа
SOURCE_DIR="/var/data"
BACKUP_DIR="/backup"
DATE=$(date +%Y%m%d)

tar -czf "${BACKUP_DIR}/backup_${DATE}.tar.gz" "$SOURCE_DIR"
echo "Backup completed: backup_${DATE}.tar.gz"
EOF

chmod +x /tmp/backup.sh
```

### Генерация конфигов

```bash
#!/bin/bash

app_name="myapp"
port="${PORT:-8080}"
workers="${WORKERS:-4}"

cat << EOF > /etc/nginx/sites-available/$app_name
server {
    listen 80;
    server_name ${app_name}.example.com;

    location / {
        proxy_pass http://127.0.0.1:${port};
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
    }
}
EOF
```

**Обратите внимание:** Nginx-переменные (`$host`, `$remote_addr`) экранированы обратным слешем, чтобы bash не пытался их раскрыть.

## Here Strings

**Here String** (`<<<`) - упрощённая версия для передачи одной строки:

```bash
# Вместо echo "Hello" | command
# Используйте:
command <<< "Hello"

# Примеры
grep "pattern" <<< "search in this string"

read -r first second <<< "one two"
echo $first   # one
echo $second  # two

bc <<< "2 + 2"  # 4
```

### Разница с pipe

```bash
# Pipe создаёт подпроцесс
echo "data" | while read line; do
    var=$line
done
echo $var  # Пусто! (переменная в подпроцессе)

# Here string не создаёт подпроцесс
while read line; do
    var=$line
done <<< "data"
echo $var  # data
```

## Heredoc и функции

### Функция с heredoc

```bash
print_help() {
    cat << EOF
Использование: $(basename "$0") [опции] файл

Опции:
    -h, --help     Показать эту справку
    -v, --verbose  Подробный вывод
    -o FILE        Выходной файл

Примеры:
    $(basename "$0") input.txt
    $(basename "$0") -v -o output.txt input.txt
EOF
}

print_help
```

### Возврат многострочного текста

```bash
generate_config() {
    local host=$1
    local port=$2

    cat << EOF
[server]
host = $host
port = $port
timeout = 30
EOF
}

# Сохранить в переменную
config=$(generate_config "localhost" 8080)
echo "$config"

# Сохранить в файл
generate_config "localhost" 8080 > server.conf
```

## Специальные случаи

### Heredoc в переменную

```bash
# Способ 1: через подстановку команды
message=$(cat << EOF
Строка 1
Строка 2
Строка 3
EOF
)

# Способ 2: read с -d (разделитель)
read -r -d '' message << EOF
Строка 1
Строка 2
Строка 3
EOF
```

### Heredoc в цикле

```bash
for user in alice bob charlie; do
    cat << EOF >> /etc/users.conf
[user.$user]
name = $user
home = /home/$user
shell = /bin/bash

EOF
done
```

### Heredoc с sudo

```bash
# Неправильно - перенаправление в пользовательском контексте
sudo cat << EOF > /etc/protected.conf  # Ошибка прав
content
EOF

# Правильно
sudo tee /etc/protected.conf << EOF > /dev/null
content
EOF

# Или через bash -c
sudo bash -c 'cat << EOF > /etc/protected.conf
content
EOF'
```

## Best Practices

1. **Используйте понятные маркеры**: `EOF`, `SQL`, `HTML`, `CONFIG`

2. **Кавычьте маркер для буквального текста**: `<< 'EOF'`

3. **Избегайте сложного экранирования** - если много спецсимволов, используйте quoted heredoc

4. **Проверяйте генерируемый контент**:
   ```bash
   # Сначала вывести на экран
   cat << EOF
   ...
   EOF

   # Потом записать в файл
   cat << EOF > file.conf
   ...
   EOF
   ```
