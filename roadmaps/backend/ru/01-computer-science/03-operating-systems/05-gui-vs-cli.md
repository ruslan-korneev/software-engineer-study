# Графический интерфейс vs Командная строка (GUI vs CLI)

[prev: 04-file-systems](./04-file-systems.md) | [next: 01-terminal-emulators](../04-shell-basics/01-what-is-shell/01-terminal-emulators.md)

---
## Определения

**GUI (Graphical User Interface)** — графический пользовательский интерфейс, позволяющий взаимодействовать с компьютером через визуальные элементы: окна, кнопки, иконки, меню.

**CLI (Command Line Interface)** — интерфейс командной строки, где пользователь вводит текстовые команды для взаимодействия с системой.

## Сравнение GUI и CLI

| Критерий | GUI | CLI |
|----------|-----|-----|
| **Кривая обучения** | Низкая, интуитивно | Высокая, требует изучения команд |
| **Скорость работы** | Медленнее (клики) | Быстрее (для опытных) |
| **Автоматизация** | Ограничена | Полная (скрипты) |
| **Ресурсоёмкость** | Высокая | Низкая |
| **Удалённый доступ** | Требует VNC/RDP | SSH достаточно |
| **Повторяемость** | Сложно повторить действия | Легко (история, скрипты) |
| **Документирование** | Скриншоты | Текстовые команды |
| **Серверы** | Редко используется | Стандарт |

## Почему CLI важен для backend-разработчика

### 1. Работа с серверами

Серверы обычно работают без GUI для экономии ресурсов.

```bash
# Подключение к серверу
ssh user@server.example.com

# Типичные серверные задачи
sudo systemctl status nginx
tail -f /var/log/nginx/access.log
docker ps
```

### 2. Автоматизация

```bash
# Скрипт деплоя
#!/bin/bash
git pull origin main
pip install -r requirements.txt
python manage.py migrate
sudo systemctl restart myapp
```

### 3. DevOps и CI/CD

```yaml
# GitHub Actions (CI/CD)
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: |
          ssh server "cd /app && git pull && docker-compose up -d"
```

### 4. Эффективность

```bash
# GUI: найти все большие файлы вручную
# CLI: одна команда
find /var/log -size +100M -exec ls -lh {} \;

# GUI: переименовать 1000 файлов вручную
# CLI: одна команда
for f in *.jpg; do mv "$f" "prefix_$f"; done
```

## Оболочки (Shells)

### Популярные оболочки

| Shell | Описание | Особенности |
|-------|----------|-------------|
| **bash** | Bourne Again Shell | Стандарт в Linux, мощный скриптинг |
| **zsh** | Z Shell | Расширенный bash, плагины (Oh My Zsh) |
| **fish** | Friendly Interactive Shell | Автодополнение, подсветка |
| **sh** | Bourne Shell | POSIX-совместимый, базовый |
| **PowerShell** | Windows | Объектная модель, .NET интеграция |

### Bash — основы

```bash
# Текущая оболочка
echo $SHELL
# /bin/bash

# Переменные
NAME="World"
echo "Hello, $NAME"

# Условия
if [ -f "file.txt" ]; then
    echo "File exists"
fi

# Циклы
for i in 1 2 3; do
    echo "Number: $i"
done

# Функции
greet() {
    echo "Hello, $1!"
}
greet "User"
```

## Основные команды CLI

### Навигация

```bash
# Текущий каталог
pwd
# /home/user

# Содержимое каталога
ls -la

# Переход
cd /path/to/directory
cd ~        # Домашний каталог
cd -        # Предыдущий каталог
cd ..       # Родительский каталог
```

### Работа с файлами

```bash
# Создание
touch file.txt
mkdir directory

# Просмотр
cat file.txt
less file.txt
head -n 10 file.txt
tail -n 10 file.txt

# Копирование и перемещение
cp source dest
mv source dest

# Удаление
rm file.txt
rm -r directory/
```

### Поиск

```bash
# Поиск файлов
find /path -name "*.txt"
locate filename

# Поиск в содержимом
grep "pattern" file.txt
grep -r "pattern" directory/
grep -i "pattern" file.txt  # Без учёта регистра
```

### Обработка текста

```bash
# Сортировка
sort file.txt
sort -n file.txt      # Числовая сортировка
sort -r file.txt      # Обратная сортировка

# Уникальные строки
uniq file.txt
sort file.txt | uniq -c  # С подсчётом

# Подсчёт
wc -l file.txt        # Строки
wc -w file.txt        # Слова
wc -c file.txt        # Байты

# Замена
sed 's/old/new/g' file.txt

# Извлечение колонок
cut -d',' -f1,3 file.csv
awk '{print $1, $3}' file.txt
```

### Конвейеры (Pipes)

Мощная концепция: вывод одной команды → ввод другой.

```bash
# Найти 10 самых больших файлов
du -ah /home | sort -rh | head -10

# Подсчитать уникальные IP в логах
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head

# Найти процессы Python
ps aux | grep python

# Сложный конвейер
cat /var/log/syslog | grep "error" | cut -d' ' -f1-3 | sort | uniq -c | sort -rn
```

## Терминальные мультиплексоры

### tmux

Позволяет работать с несколькими терминалами в одном окне, сохранять сессии.

```bash
# Создание сессии
tmux new -s mysession

# Отключение от сессии (без закрытия)
Ctrl+b, d

# Список сессий
tmux ls

# Подключение к сессии
tmux attach -t mysession

# Разделение окна
Ctrl+b, %    # Вертикально
Ctrl+b, "    # Горизонтально

# Переключение между панелями
Ctrl+b, стрелки
```

```
┌─────────────────────────────────────────────┐
│ $ vim app.py       │ $ python app.py       │
│                    │ Running on port 8000  │
│                    │ ...                   │
├────────────────────┴────────────────────────┤
│ $ tail -f /var/log/app.log                  │
│ [INFO] Request received                     │
└─────────────────────────────────────────────┘
```

### screen

Альтернатива tmux, предустановлена на многих системах.

```bash
# Создание сессии
screen -S mysession

# Отключение
Ctrl+a, d

# Подключение
screen -r mysession
```

## Эффективная работа в CLI

### История команд

```bash
# Показать историю
history

# Поиск в истории
Ctrl+r  # Затем начните вводить команду

# Повтор последней команды
!!

# Повтор команды по номеру
!123

# Последний аргумент предыдущей команды
!$

# Все аргументы предыдущей команды
!*
```

### Горячие клавиши Bash

```
Ctrl+a    — в начало строки
Ctrl+e    — в конец строки
Ctrl+u    — удалить до начала строки
Ctrl+k    — удалить до конца строки
Ctrl+w    — удалить слово назад
Ctrl+l    — очистить экран
Ctrl+c    — прервать команду
Ctrl+z    — приостановить (fg для продолжения)
Ctrl+d    — выход (EOF)
Tab       — автодополнение
Tab Tab   — показать варианты
```

### Алиасы

```bash
# Добавить в ~/.bashrc или ~/.zshrc

# Сокращения
alias ll='ls -la'
alias la='ls -A'
alias l='ls -CF'

# Безопасность
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'

# Git
alias gs='git status'
alias gc='git commit'
alias gp='git push'

# Docker
alias dc='docker-compose'
alias dps='docker ps'

# Применить изменения
source ~/.bashrc
```

## CLI-инструменты для разработчика

### Работа с сетью

```bash
# Проверка доступности
ping google.com

# HTTP-запросы
curl https://api.example.com/data
curl -X POST -d '{"key":"value"}' -H "Content-Type: application/json" https://api.example.com

# Скачивание файлов
wget https://example.com/file.zip

# Просмотр портов
netstat -tlnp
ss -tlnp

# DNS
nslookup example.com
dig example.com
```

### Мониторинг системы

```bash
# Процессы
top
htop          # Более удобный

# Использование диска
df -h
du -sh *

# Память
free -h

# Сетевая активность
iftop
nethogs
```

### Работа с Docker

```bash
# Список контейнеров
docker ps -a

# Логи
docker logs -f container_name

# Вход в контейнер
docker exec -it container_name bash

# Docker Compose
docker-compose up -d
docker-compose logs -f
docker-compose down
```

### Git

```bash
# Статус
git status

# Коммит
git add .
git commit -m "message"

# Ветки
git branch
git checkout -b new-branch

# История
git log --oneline --graph

# Разрешение конфликтов
git merge branch
# Редактируем конфликты
git add .
git commit
```

## GUI в разработке

### Когда GUI полезен

1. **IDE** (VS Code, PyCharm) — редактирование кода, отладка
2. **Database GUI** (DBeaver, pgAdmin) — просмотр данных
3. **API-тестирование** (Postman, Insomnia) — визуальное тестирование
4. **Git GUI** (GitKraken, Sourcetree) — визуализация истории
5. **Docker Desktop** — управление контейнерами

### Комбинированный подход

Опытные разработчики используют оба интерфейса:

```
Разработка:
├── IDE (VS Code) — написание кода
│   └── Встроенный терминал — CLI команды
├── Browser — тестирование веб-приложений
└── CLI — Git, Docker, деплой

Администрирование серверов:
└── CLI (SSH) — единственный вариант
```

## Best Practices

### Для CLI

1. **Изучайте базовые команды** — они одинаковы на всех Unix-системах
2. **Используйте алиасы** — сокращайте часто используемые команды
3. **Пишите скрипты** — автоматизируйте повторяющиеся задачи
4. **Используйте tmux/screen** — для долгих сессий
5. **Читайте man-страницы** — `man command` для документации

```bash
# Полезные привычки
man ls              # Документация
ls --help           # Краткая справка
type command        # Что такое команда
which command       # Где находится
```

### Для разработки

1. **Освойте CLI вашего стека**: npm, pip, docker, kubectl
2. **Настройте терминал**: zsh + Oh My Zsh, красивый prompt
3. **Используйте dotfiles**: храните конфиги в Git

```bash
# Пример .zshrc
export PATH="$HOME/bin:$PATH"
source ~/.aliases

# Красивый prompt с git-информацией
PROMPT='%F{cyan}%~%f $(git_prompt_info) %# '
```

## Заключение

Для backend-разработчика CLI — это:
- **Необходимость** для работы с серверами
- **Эффективность** для автоматизации задач
- **Стандарт** для DevOps и CI/CD
- **Универсальность** между различными системами

Рекомендуется:
1. Освоить базовые команды Linux
2. Изучить bash-скриптинг
3. Настроить удобное окружение (zsh, tmux)
4. Практиковаться ежедневно

GUI и CLI дополняют друг друга — используйте каждый там, где он наиболее эффективен.

---

[prev: 04-file-systems](./04-file-systems.md) | [next: 01-terminal-emulators](../04-shell-basics/01-what-is-shell/01-terminal-emulators.md)
