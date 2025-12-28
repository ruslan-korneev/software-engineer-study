# Меню в bash-скриптах

[prev: 03-validating-input](./03-validating-input.md) | [next: 01-while-loop](../06-flow-control-loops/01-while-loop.md)

---
## Базовое меню с select

Команда `select` создаёт простое меню выбора:

```bash
#!/bin/bash

PS3="Выберите опцию: "

select option in "Опция 1" "Опция 2" "Опция 3" "Выход"; do
    case $option in
        "Опция 1")
            echo "Вы выбрали опцию 1"
            ;;
        "Опция 2")
            echo "Вы выбрали опцию 2"
            ;;
        "Опция 3")
            echo "Вы выбрали опцию 3"
            ;;
        "Выход")
            echo "До свидания!"
            break
            ;;
        *)
            echo "Неверный выбор"
            ;;
    esac
done
```

Вывод:
```
1) Опция 1
2) Опция 2
3) Опция 3
4) Выход
Выберите опцию:
```

## Переменная PS3

`PS3` - это приглашение для команды `select`:

```bash
PS3="Ваш выбор: "        # Стандартный вариант
PS3=">>> "                # Минималистичный
PS3=$'\nВведите номер: '  # С переводом строки
```

## Меню из массива

```bash
#!/bin/bash

options=("Создать файл" "Удалить файл" "Показать файлы" "Выход")

PS3="Выберите действие: "

select opt in "${options[@]}"; do
    case $REPLY in  # $REPLY содержит введённый номер
        1) touch "newfile.txt"; echo "Файл создан" ;;
        2) rm -i "newfile.txt"; echo "Файл удалён" ;;
        3) ls -la ;;
        4) break ;;
        *) echo "Неверный выбор: $REPLY" ;;
    esac
done
```

## Динамическое меню

### Меню из файлов директории

```bash
#!/bin/bash

echo "Выберите файл для редактирования:"

PS3="Номер файла: "

select file in *.txt "Отмена"; do
    if [[ "$file" == "Отмена" ]]; then
        echo "Отменено"
        break
    elif [[ -n "$file" ]]; then
        echo "Редактирование: $file"
        ${EDITOR:-nano} "$file"
        break
    else
        echo "Неверный выбор"
    fi
done
```

### Меню из результата команды

```bash
#!/bin/bash

# Меню из пользователей системы
PS3="Выберите пользователя: "

select user in $(cut -d: -f1 /etc/passwd | head -10) "Отмена"; do
    if [[ "$user" == "Отмена" ]]; then
        break
    elif [[ -n "$user" ]]; then
        echo "Информация о $user:"
        id "$user"
        break
    fi
done
```

## Меню без select

### Простое меню с case

```bash
#!/bin/bash

show_menu() {
    echo "================================"
    echo "       ГЛАВНОЕ МЕНЮ"
    echo "================================"
    echo "1. Показать дату"
    echo "2. Показать время"
    echo "3. Показать пользователя"
    echo "4. Выход"
    echo "================================"
}

while true; do
    show_menu
    read -rp "Выберите опцию [1-4]: " choice

    case $choice in
        1)
            date +%Y-%m-%d
            ;;
        2)
            date +%H:%M:%S
            ;;
        3)
            whoami
            ;;
        4)
            echo "До свидания!"
            exit 0
            ;;
        *)
            echo "Неверный выбор!"
            ;;
    esac

    echo
    read -rp "Нажмите Enter для продолжения..."
done
```

### Меню с очисткой экрана

```bash
#!/bin/bash

show_menu() {
    clear
    echo "╔════════════════════════════════╗"
    echo "║      СИСТЕМНЫЙ МОНИТОР         ║"
    echo "╠════════════════════════════════╣"
    echo "║ 1. Использование диска         ║"
    echo "║ 2. Использование памяти        ║"
    echo "║ 3. Запущенные процессы         ║"
    echo "║ 4. Сетевые соединения          ║"
    echo "║ q. Выход                       ║"
    echo "╚════════════════════════════════╝"
}

while true; do
    show_menu
    read -rsn1 -p "Ваш выбор: " choice
    echo

    case $choice in
        1)
            df -h
            ;;
        2)
            free -h
            ;;
        3)
            ps aux --sort=-%mem | head -10
            ;;
        4)
            netstat -tuln 2>/dev/null || ss -tuln
            ;;
        q|Q)
            echo "Выход..."
            exit 0
            ;;
        *)
            echo "Неверный выбор!"
            ;;
    esac

    echo
    read -rp "Нажмите Enter..."
done
```

## Многоуровневое меню

```bash
#!/bin/bash

main_menu() {
    echo "ГЛАВНОЕ МЕНЮ"
    echo "1. Файлы"
    echo "2. Система"
    echo "3. Выход"
}

files_menu() {
    echo "МЕНЮ ФАЙЛОВ"
    echo "1. Список файлов"
    echo "2. Создать файл"
    echo "3. Удалить файл"
    echo "4. Назад"
}

system_menu() {
    echo "СИСТЕМНОЕ МЕНЮ"
    echo "1. Информация о системе"
    echo "2. Дисковое пространство"
    echo "3. Назад"
}

handle_files_menu() {
    while true; do
        clear
        files_menu
        read -rp "Выбор: " choice
        case $choice in
            1) ls -la; read -rp "Enter..." ;;
            2) read -rp "Имя файла: " f; touch "$f" ;;
            3) read -rp "Имя файла: " f; rm -i "$f" ;;
            4) return ;;
            *) echo "Неверный выбор" ;;
        esac
    done
}

handle_system_menu() {
    while true; do
        clear
        system_menu
        read -rp "Выбор: " choice
        case $choice in
            1) uname -a; read -rp "Enter..." ;;
            2) df -h; read -rp "Enter..." ;;
            3) return ;;
            *) echo "Неверный выбор" ;;
        esac
    done
}

# Главный цикл
while true; do
    clear
    main_menu
    read -rp "Выбор: " choice
    case $choice in
        1) handle_files_menu ;;
        2) handle_system_menu ;;
        3) echo "До свидания!"; exit 0 ;;
        *) echo "Неверный выбор" ;;
    esac
done
```

## Интерактивное меню с клавишами

```bash
#!/bin/bash

# Меню с навигацией стрелками
arrow_menu() {
    local options=("$@")
    local selected=0
    local key

    while true; do
        clear
        echo "Используйте стрелки для выбора, Enter для подтверждения"
        echo

        for i in "${!options[@]}"; do
            if [[ $i -eq $selected ]]; then
                echo " > ${options[$i]}"
            else
                echo "   ${options[$i]}"
            fi
        done

        # Чтение клавиши
        read -rsn1 key

        case $key in
            A)  # Стрелка вверх (ESC[A)
                ((selected--))
                [[ $selected -lt 0 ]] && selected=$((${#options[@]} - 1))
                ;;
            B)  # Стрелка вниз (ESC[B)
                ((selected++))
                [[ $selected -ge ${#options[@]} ]] && selected=0
                ;;
            "")  # Enter
                echo "${options[$selected]}"
                return $selected
                ;;
        esac
    done
}

# Использование
options=("Создать" "Редактировать" "Удалить" "Выход")
choice=$(arrow_menu "${options[@]}")
echo "Вы выбрали: $choice"
```

## Меню с цветами

```bash
#!/bin/bash

# Определение цветов
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color
BOLD='\033[1m'

colored_menu() {
    clear
    echo -e "${BOLD}${BLUE}╔══════════════════════════════╗${NC}"
    echo -e "${BOLD}${BLUE}║${NC}      ${BOLD}ГЛАВНОЕ МЕНЮ${NC}          ${BOLD}${BLUE}║${NC}"
    echo -e "${BOLD}${BLUE}╠══════════════════════════════╣${NC}"
    echo -e "${BOLD}${BLUE}║${NC} ${GREEN}1.${NC} Показать информацию      ${BOLD}${BLUE}║${NC}"
    echo -e "${BOLD}${BLUE}║${NC} ${GREEN}2.${NC} Проверить статус         ${BOLD}${BLUE}║${NC}"
    echo -e "${BOLD}${BLUE}║${NC} ${GREEN}3.${NC} Выполнить действие       ${BOLD}${BLUE}║${NC}"
    echo -e "${BOLD}${BLUE}║${NC} ${RED}q.${NC} Выход                     ${BOLD}${BLUE}║${NC}"
    echo -e "${BOLD}${BLUE}╚══════════════════════════════╝${NC}"
}

while true; do
    colored_menu
    read -rsn1 -p "$(echo -e ${YELLOW}Ваш выбор: ${NC})" choice

    case $choice in
        1) echo -e "\n${GREEN}Информация системы:${NC}"; uname -a ;;
        2) echo -e "\n${GREEN}Статус:${NC} OK" ;;
        3) echo -e "\n${YELLOW}Выполнение...${NC}" ;;
        q|Q) echo -e "\n${GREEN}До свидания!${NC}"; exit 0 ;;
        *) echo -e "\n${RED}Неверный выбор!${NC}" ;;
    esac

    read -rp "Нажмите Enter..."
done
```

## Обработка сигналов в меню

```bash
#!/bin/bash

# Перехват Ctrl+C
trap 'echo -e "\nИспользуйте опцию Выход"' INT

# Очистка при выходе
trap 'echo "Очистка..."; rm -f /tmp/menu_$$' EXIT

show_menu() {
    echo "1. Опция 1"
    echo "2. Опция 2"
    echo "3. Выход"
}

while true; do
    show_menu
    read -rp "Выбор: " choice
    case $choice in
        1) echo "Опция 1" ;;
        2) echo "Опция 2" ;;
        3) exit 0 ;;
    esac
done
```

## Лучшие практики для меню

1. **Всегда предоставляйте опцию выхода**
2. **Обрабатывайте неверный ввод**
3. **Давайте обратную связь** после каждого действия
4. **Очищайте экран** для лучшей читаемости
5. **Используйте паузу** после вывода результатов
6. **Перехватывайте сигналы** (Ctrl+C)

---

[prev: 03-validating-input](./03-validating-input.md) | [next: 01-while-loop](../06-flow-control-loops/01-while-loop.md)
