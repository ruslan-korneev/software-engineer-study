# ping и traceroute - диагностика сети

[prev: 04-dd](../02-storage-devices/04-dd.md) | [next: 02-ip-netstat](./02-ip-netstat.md)

---
## ping - проверка доступности хоста

### Что такое ping?

**ping** - утилита для проверки доступности сетевого узла и измерения времени отклика. Использует протокол ICMP (Internet Control Message Protocol).

### Принцип работы

1. ping отправляет ICMP Echo Request на целевой хост
2. Хост отвечает ICMP Echo Reply
3. ping измеряет время между запросом и ответом (RTT - Round Trip Time)

### Базовое использование

```bash
# Пинговать хост (бесконечно, Ctrl+C для остановки)
ping google.com

# Пинговать IP-адрес
ping 8.8.8.8

# Указать количество пакетов
ping -c 4 google.com

# Пример вывода:
# PING google.com (142.250.74.206) 56(84) bytes of data.
# 64 bytes from fra16s51-in-f14.1e100.net: icmp_seq=1 ttl=117 time=12.3 ms
# 64 bytes from fra16s51-in-f14.1e100.net: icmp_seq=2 ttl=117 time=11.8 ms
# 64 bytes from fra16s51-in-f14.1e100.net: icmp_seq=3 ttl=117 time=12.1 ms
# 64 bytes from fra16s51-in-f14.1e100.net: icmp_seq=4 ttl=117 time=11.9 ms
#
# --- google.com ping statistics ---
# 4 packets transmitted, 4 received, 0% packet loss, time 3004ms
# rtt min/avg/max/mdev = 11.816/12.025/12.312/0.180 ms
```

### Расшифровка вывода

- **icmp_seq** - порядковый номер пакета
- **ttl** - Time To Live (сколько маршрутизаторов может пройти пакет)
- **time** - время отклика в миллисекундах
- **packet loss** - процент потерянных пакетов
- **rtt min/avg/max/mdev** - минимальное/среднее/максимальное время и отклонение

### Основные опции

```bash
# Количество пакетов
ping -c 10 google.com

# Интервал между пакетами (секунды, по умолчанию 1)
ping -i 0.5 google.com      # каждые 0.5 секунды
ping -i 2 google.com        # каждые 2 секунды

# Flood ping (требует root, очень быстро)
sudo ping -f google.com

# Размер пакета (по умолчанию 56 байт + 8 ICMP заголовок = 64)
ping -s 1000 google.com     # 1000 байт данных

# Таймаут ожидания ответа (секунды)
ping -W 2 google.com        # ждать 2 секунды

# Deadline - общее время работы (секунды)
ping -w 10 google.com       # работать 10 секунд

# Тихий режим (только статистика в конце)
ping -q -c 10 google.com

# Не резолвить имена (быстрее)
ping -n 8.8.8.8

# Указать источник
ping -I eth0 google.com     # интерфейс
ping -I 192.168.1.100 google.com  # IP
```

### IPv4 vs IPv6

```bash
# Принудительно IPv4
ping -4 google.com
ping4 google.com

# Принудительно IPv6
ping -6 google.com
ping6 google.com
```

### Типичные проблемы

```bash
# Хост недоступен
# ping: connect: Network is unreachable
# Причина: нет маршрута к сети

# Нет ответа
# Request timeout for icmp_seq 0
# Причины: хост выключен, файрвол блокирует ICMP, хост не существует

# 100% packet loss
# Причины: сетевые проблемы, блокировка ICMP
```

## traceroute - трассировка маршрута

### Что такое traceroute?

**traceroute** показывает путь (маршрут), который проходят пакеты до целевого хоста, включая все промежуточные маршрутизаторы.

### Принцип работы

1. Отправляет пакеты с TTL=1 (первый роутер ответит ICMP Time Exceeded)
2. Увеличивает TTL на 1 и повторяет
3. Продолжает до достижения цели или максимального TTL

### Базовое использование

```bash
# Linux
traceroute google.com

# macOS
traceroute google.com

# Windows (другое название)
tracert google.com

# Пример вывода:
# traceroute to google.com (142.250.74.206), 30 hops max, 60 byte packets
#  1  _gateway (192.168.1.1)  0.531 ms  0.488 ms  0.462 ms
#  2  10.0.0.1 (10.0.0.1)  8.234 ms  8.205 ms  8.178 ms
#  3  core-router.isp.net (85.115.1.1)  12.456 ms  12.423 ms  12.398 ms
#  4  edge-router.isp.net (85.115.2.1)  15.678 ms  15.645 ms  15.619 ms
#  5  google-peer.isp.net (85.115.3.1)  18.901 ms  18.867 ms  18.840 ms
#  6  142.250.74.206 (142.250.74.206)  20.123 ms  20.089 ms  20.062 ms
```

### Расшифровка вывода

- **Hop number** - номер перехода (маршрутизатора)
- **Hostname/IP** - имя и адрес маршрутизатора
- **3 значения времени** - время трёх проб в мс
- **\*** - нет ответа (таймаут или ICMP заблокирован)

### Основные опции

```bash
# Максимальное количество хопов (по умолчанию 30)
traceroute -m 15 google.com

# Количество проб на каждый хоп (по умолчанию 3)
traceroute -q 1 google.com

# Не резолвить имена (быстрее)
traceroute -n google.com

# Таймаут ожидания (секунды)
traceroute -w 2 google.com

# Использовать ICMP вместо UDP (как ping)
traceroute -I google.com

# Использовать TCP SYN (полезно если UDP/ICMP блокируется)
sudo traceroute -T google.com

# Указать порт
traceroute -p 80 google.com

# Указать интерфейс
traceroute -i eth0 google.com
```

### Работа с IPv6

```bash
# IPv6 трассировка
traceroute -6 google.com
# или
traceroute6 google.com
```

## mtr - объединение ping и traceroute

**mtr** (My TraceRoute) - комбинирует функции ping и traceroute в реальном времени.

### Установка

```bash
# Debian/Ubuntu
sudo apt install mtr

# RHEL/CentOS
sudo yum install mtr
```

### Использование

```bash
# Интерактивный режим (по умолчанию)
mtr google.com

# Текстовый режим (отчёт)
mtr -r google.com

# Отчёт с N циклами
mtr -r -c 100 google.com

# Без DNS
mtr -n google.com

# Широкий формат (показать IP и имена)
mtr -w google.com

# Пример вывода:
#                              My traceroute  [v0.95]
# hostname                      Loss%   Snt   Last   Avg  Best  Wrst StDev
# 1. _gateway                    0.0%    10    0.5   0.6   0.4   0.8   0.1
# 2. 10.0.0.1                    0.0%    10    8.2   8.5   7.9   9.1   0.4
# 3. core-router.isp.net         0.0%    10   12.1  12.4  11.8  13.2   0.5
# 4. google-peer.isp.net         0.0%    10   18.5  18.9  18.1  19.8   0.6
# 5. 142.250.74.206              0.0%    10   20.2  20.5  19.8  21.3   0.5
```

### Расшифровка колонок mtr

| Колонка | Описание |
|---------|----------|
| Loss% | Процент потерь |
| Snt | Отправлено пакетов |
| Last | Последнее время |
| Avg | Среднее время |
| Best | Лучшее время |
| Wrst | Худшее время |
| StDev | Стандартное отклонение |

## Практические сценарии

### Диагностика проблем с сетью

```bash
# 1. Проверить локальный хост
ping -c 3 127.0.0.1

# 2. Проверить шлюз по умолчанию
ip route | grep default
ping -c 3 192.168.1.1

# 3. Проверить DNS
ping -c 3 8.8.8.8

# 4. Проверить резолвинг DNS
ping -c 3 google.com

# 5. Трассировка для понимания где проблема
traceroute google.com
```

### Мониторинг качества связи

```bash
# Непрерывный мониторинг с mtr
mtr google.com

# Или с ping и анализом потерь
ping -c 100 google.com | grep -E "time=|loss"
```

### Проверка перед развёртыванием

```bash
# Проверить доступность сервера
ping -c 5 -W 2 server.example.com

# Проверить путь
traceroute -n server.example.com

# Проверить конкретный порт (с nmap или nc)
nc -zv server.example.com 22
```

### Скрипт проверки списка хостов

```bash
#!/bin/bash
hosts="google.com github.com example.com"

for host in $hosts; do
    if ping -c 1 -W 2 $host > /dev/null 2>&1; then
        echo "OK: $host"
    else
        echo "FAIL: $host"
    fi
done
```

## Дополнительные инструменты

### hping3 - расширенный ping

```bash
# Установка
sudo apt install hping3

# TCP ping (полезно когда ICMP заблокирован)
sudo hping3 -S -p 80 -c 3 google.com

# SYN флуд (для тестирования, осторожно!)
sudo hping3 -S --flood -p 80 target.com
```

### fping - быстрый ping многих хостов

```bash
# Установка
sudo apt install fping

# Пинговать несколько хостов
fping google.com github.com example.com

# Из файла
fping -f hosts.txt

# Показать только живые хосты
fping -a -f hosts.txt
```

### arping - ping на уровне ARP

```bash
# Полезен для локальной сети
sudo arping -c 3 192.168.1.1

# Найти MAC-адрес
sudo arping -c 1 192.168.1.1
```

## Firewall и ICMP

### Почему ping может не работать

1. **Файрвол на хосте** блокирует входящие ICMP
2. **Файрвол провайдера** фильтрует ICMP
3. **Маршрутизатор** не отвечает на ICMP
4. **Rate limiting** на ICMP ответы

### Разрешить ping в iptables

```bash
# Разрешить входящие ICMP echo-request
sudo iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# Разрешить исходящие ICMP echo-reply
sudo iptables -A OUTPUT -p icmp --icmp-type echo-reply -j ACCEPT
```

### Проверить доступность через TCP

```bash
# Если ICMP заблокирован, используйте:
nc -zv google.com 80
# или
curl -I --connect-timeout 5 http://google.com
# или
traceroute -T -p 80 google.com
```

## Резюме команд

| Команда | Описание |
|---------|----------|
| `ping host` | Проверить доступность |
| `ping -c 4 host` | Отправить 4 пакета |
| `ping -i 0.2 host` | Интервал 0.2 сек |
| `ping -s 1000 host` | Размер пакета 1000 байт |
| `traceroute host` | Показать маршрут |
| `traceroute -n host` | Без DNS резолвинга |
| `traceroute -I host` | Использовать ICMP |
| `mtr host` | Интерактивный мониторинг |
| `mtr -r host` | Отчёт |

---

[prev: 04-dd](../02-storage-devices/04-dd.md) | [next: 02-ip-netstat](./02-ip-netstat.md)
