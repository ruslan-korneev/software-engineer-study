# ip и netstat - настройка сети и мониторинг соединений

## Команда ip - современный инструмент управления сетью

### Что такое ip?

**ip** - универсальная утилита из пакета iproute2 для управления сетевыми интерфейсами, адресами, маршрутами и туннелями. Заменяет устаревшие ifconfig, route, arp.

### Структура команды

```bash
ip [опции] объект команда
```

Объекты:
- `link` - сетевые интерфейсы
- `addr` (address) - IP-адреса
- `route` - таблица маршрутизации
- `neigh` (neighbour) - ARP-кэш
- `tunnel` - туннели
- `rule` - правила маршрутизации

### ip link - управление интерфейсами

```bash
# Показать все интерфейсы
ip link
ip link show
ip l                    # сокращённо

# Показать конкретный интерфейс
ip link show eth0

# Включить интерфейс
sudo ip link set eth0 up

# Выключить интерфейс
sudo ip link set eth0 down

# Изменить MTU
sudo ip link set eth0 mtu 9000

# Изменить MAC-адрес
sudo ip link set eth0 down
sudo ip link set eth0 address 00:11:22:33:44:55
sudo ip link set eth0 up

# Переименовать интерфейс
sudo ip link set eth0 name lan0

# Показать статистику
ip -s link show eth0
```

### ip addr - управление IP-адресами

```bash
# Показать все адреса
ip addr
ip addr show
ip a                    # сокращённо

# Показать адреса интерфейса
ip addr show eth0

# Показать только IPv4
ip -4 addr

# Показать только IPv6
ip -6 addr

# Добавить IP-адрес
sudo ip addr add 192.168.1.100/24 dev eth0

# Добавить дополнительный адрес (алиас)
sudo ip addr add 192.168.1.101/24 dev eth0

# Удалить IP-адрес
sudo ip addr del 192.168.1.100/24 dev eth0

# Удалить все адреса интерфейса
sudo ip addr flush dev eth0
```

### ip route - управление маршрутами

```bash
# Показать таблицу маршрутизации
ip route
ip route show
ip r                    # сокращённо

# Показать маршрут до хоста
ip route get 8.8.8.8

# Добавить маршрут по умолчанию (шлюз)
sudo ip route add default via 192.168.1.1

# Добавить маршрут к сети
sudo ip route add 10.0.0.0/8 via 192.168.1.254

# Добавить маршрут через интерфейс
sudo ip route add 10.0.0.0/8 dev eth0

# Удалить маршрут
sudo ip route del 10.0.0.0/8

# Удалить шлюз по умолчанию
sudo ip route del default

# Изменить маршрут
sudo ip route replace default via 192.168.1.1
```

### ip neigh - ARP-таблица

```bash
# Показать ARP-кэш
ip neigh
ip neigh show
ip n                    # сокращённо

# Добавить статическую запись
sudo ip neigh add 192.168.1.50 lladdr 00:11:22:33:44:55 dev eth0

# Удалить запись
sudo ip neigh del 192.168.1.50 dev eth0

# Очистить весь кэш
sudo ip neigh flush all
```

### Полезные опции ip

```bash
# Цветной вывод
ip -c addr

# JSON формат (для скриптов)
ip -j addr

# Краткий вывод
ip -br addr
ip -br link

# Только изменения (мониторинг)
ip monitor
```

## ifconfig - классический инструмент (устаревший)

### Базовое использование

```bash
# Показать все интерфейсы
ifconfig

# Показать конкретный интерфейс
ifconfig eth0

# Показать только активные
ifconfig -a

# Включить интерфейс
sudo ifconfig eth0 up

# Выключить интерфейс
sudo ifconfig eth0 down

# Назначить IP-адрес
sudo ifconfig eth0 192.168.1.100 netmask 255.255.255.0

# Изменить MTU
sudo ifconfig eth0 mtu 9000
```

### Сравнение ip и ifconfig

| Задача | ifconfig | ip |
|--------|----------|-----|
| Показать интерфейсы | `ifconfig` | `ip link` |
| Показать адреса | `ifconfig` | `ip addr` |
| Добавить IP | `ifconfig eth0 IP` | `ip addr add IP dev eth0` |
| Включить | `ifconfig eth0 up` | `ip link set eth0 up` |
| Маршруты | `route` | `ip route` |

## netstat - статистика сети (устаревший)

### Что такое netstat?

**netstat** показывает сетевые соединения, таблицы маршрутизации, статистику интерфейсов и другую сетевую информацию.

**Примечание:** netstat считается устаревшим. Рекомендуется использовать **ss** (socket statistics).

### Основные опции netstat

```bash
# Все соединения
netstat -a

# Только TCP
netstat -t

# Только UDP
netstat -u

# Показать номера портов (не резолвить в имена)
netstat -n

# Показать PID и имя программы
netstat -p

# Показать слушающие сокеты
netstat -l

# Комбинация: все TCP, числа, с программами
netstat -tnp

# Слушающие TCP порты с программами
sudo netstat -tlnp

# Статистика интерфейсов
netstat -i

# Таблица маршрутизации
netstat -r
netstat -rn     # без резолвинга

# Статистика протоколов
netstat -s
```

### Типичный вывод netstat

```bash
sudo netstat -tlnp
# Proto Recv-Q Send-Q Local Address    Foreign Address  State   PID/Program
# tcp   0      0      0.0.0.0:22       0.0.0.0:*        LISTEN  1234/sshd
# tcp   0      0      127.0.0.1:3306   0.0.0.0:*        LISTEN  5678/mysqld
# tcp   0      0      0.0.0.0:80       0.0.0.0:*        LISTEN  9012/nginx
```

## ss - современная замена netstat

### Преимущества ss

- Быстрее netstat
- Больше информации
- Лучшая фильтрация
- Активно развивается

### Основные опции ss

```bash
# Все сокеты
ss -a

# Только TCP
ss -t

# Только UDP
ss -u

# Слушающие сокеты
ss -l

# Показать номера (не резолвить)
ss -n

# Показать процессы
ss -p

# Расширенная информация
ss -e

# Показать таймеры
ss -o

# Комбинация: слушающие TCP с процессами
sudo ss -tlnp

# Все TCP соединения
ss -ta

# Все установленные соединения
ss -t state established

# Показать использование памяти
ss -m
```

### Фильтрация в ss

```bash
# По порту
ss -tn 'sport = :22'          # исходящий порт 22
ss -tn 'dport = :443'         # порт назначения 443

# По состоянию
ss -t state established
ss -t state listening
ss -t state time-wait
ss -t state close-wait

# По адресу
ss -tn 'src 192.168.1.0/24'   # из подсети
ss -tn 'dst 8.8.8.8'          # к адресу

# Комбинация фильтров
ss -tn 'dport = :443 and state established'

# Исключить состояние
ss -t state all '( not established )'
```

### Примеры использования ss

```bash
# Кто слушает на порту 80
sudo ss -tlnp 'sport = :80'

# Сколько соединений к веб-серверу
ss -tn 'dport = :443' | wc -l

# Соединения в состоянии TIME-WAIT
ss -tn state time-wait | head

# Показать все сокеты процесса nginx
ss -tlnp | grep nginx

# Статистика по состояниям
ss -s
```

### Вывод ss -s (статистика)

```
Total: 523
TCP:   45 (estab 12, closed 8, orphaned 0, timewait 5)

Transport Total     IP        IPv6
RAW       0         0         0
UDP       12        8         4
TCP       37        25        12
INET      49        33        16
FRAG      0         0         0
```

## Практические сценарии

### Диагностика сетевых проблем

```bash
# 1. Проверить интерфейсы
ip link

# 2. Проверить IP-адреса
ip addr

# 3. Проверить шлюз
ip route

# 4. Проверить DNS
cat /etc/resolv.conf

# 5. Проверить соединения
ss -tn
```

### Найти что использует порт

```bash
# Кто слушает порт 8080
sudo ss -tlnp 'sport = :8080'
# или
sudo lsof -i :8080
# или
sudo fuser 8080/tcp
```

### Мониторинг соединений в реальном времени

```bash
# С помощью watch
watch -n 1 'ss -tn state established | wc -l'

# Или iftop (трафик в реальном времени)
sudo apt install iftop
sudo iftop -i eth0
```

### Настройка статического IP

```bash
# Временно (до перезагрузки)
sudo ip addr add 192.168.1.100/24 dev eth0
sudo ip link set eth0 up
sudo ip route add default via 192.168.1.1

# Постоянно через netplan (Ubuntu 18.04+)
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - 192.168.1.100/24
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]

# Применить
sudo netplan apply
```

### Скрипт проверки сетевых соединений

```bash
#!/bin/bash
echo "=== Network Interfaces ==="
ip -br link

echo -e "\n=== IP Addresses ==="
ip -br addr

echo -e "\n=== Default Route ==="
ip route | grep default

echo -e "\n=== Listening Ports ==="
ss -tlnp

echo -e "\n=== Active Connections ==="
ss -tn state established | head -10
```

## Дополнительные инструменты

### nmcli - NetworkManager CLI

```bash
# Показать соединения
nmcli connection show

# Показать устройства
nmcli device status

# Подключиться к WiFi
nmcli device wifi connect "SSID" password "password"

# Создать статическое соединение
nmcli connection add type ethernet con-name "static" \
    ifname eth0 ip4 192.168.1.100/24 gw4 192.168.1.1
```

### ethtool - настройки Ethernet

```bash
# Информация об интерфейсе
ethtool eth0

# Скорость и дуплекс
ethtool eth0 | grep -E "Speed|Duplex"

# Статистика драйвера
ethtool -S eth0

# Установить скорость
sudo ethtool -s eth0 speed 1000 duplex full
```

## Резюме команд

| Задача | Команда |
|--------|---------|
| Показать интерфейсы | `ip link` или `ip -br link` |
| Показать IP-адреса | `ip addr` или `ip -br addr` |
| Добавить IP | `sudo ip addr add IP/mask dev iface` |
| Включить интерфейс | `sudo ip link set iface up` |
| Показать маршруты | `ip route` |
| Добавить шлюз | `sudo ip route add default via IP` |
| Показать ARP | `ip neigh` |
| Слушающие порты | `sudo ss -tlnp` |
| Все TCP соединения | `ss -tn` |
| Фильтр по порту | `ss -tn 'dport = :80'` |
| Статистика сокетов | `ss -s` |
