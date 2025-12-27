# Выключение и перезагрузка системы

## Команда shutdown

**shutdown** — безопасное выключение или перезагрузка системы с уведомлением пользователей.

### Синтаксис
```bash
shutdown [опции] [время] [сообщение]
```

### Немедленное выключение
```bash
$ sudo shutdown now
$ sudo shutdown -h now      # halt (остановить)
$ sudo shutdown -P now      # power off (выключить питание)
```

### Перезагрузка
```bash
$ sudo shutdown -r now      # reboot (перезагрузка)
```

### Отложенное выключение

```bash
$ sudo shutdown -h +10      # через 10 минут
$ sudo shutdown -h 23:00    # в 23:00
$ sudo shutdown -r +5 "Система будет перезагружена через 5 минут"
```

### Отмена запланированного выключения
```bash
$ sudo shutdown -c
$ sudo shutdown -c "Выключение отменено"
```

## Команды halt, poweroff, reboot

### halt — остановить систему
```bash
$ sudo halt
```

### poweroff — выключить питание
```bash
$ sudo poweroff
```

### reboot — перезагрузить
```bash
$ sudo reboot
```

Эти команды — часто ссылки на systemctl:
```bash
$ ls -l /sbin/reboot
/sbin/reboot -> /bin/systemctl
```

## Команда systemctl (systemd)

На современных системах с systemd:

```bash
$ sudo systemctl poweroff   # выключить
$ sudo systemctl reboot     # перезагрузить
$ sudo systemctl halt       # остановить
$ sudo systemctl suspend    # в спящий режим
$ sudo systemctl hibernate  # в гибернацию
```

## Уровни выполнения (runlevels)

### SysV init (устаревший)

| Уровень | Описание |
|---------|----------|
| 0 | Выключение |
| 1 | Однопользовательский режим |
| 2-4 | Многопользовательский |
| 5 | Графический режим |
| 6 | Перезагрузка |

```bash
$ sudo init 0     # выключить
$ sudo init 6     # перезагрузить
$ sudo telinit 0  # альтернатива
```

### Systemd targets

| Target | Описание | Аналог runlevel |
|--------|----------|-----------------|
| poweroff.target | Выключение | 0 |
| rescue.target | Восстановление | 1 |
| multi-user.target | Многопользовательский | 3 |
| graphical.target | Графический | 5 |
| reboot.target | Перезагрузка | 6 |

```bash
$ sudo systemctl isolate poweroff.target
$ sudo systemctl isolate reboot.target
```

## Что происходит при выключении

1. **Уведомление пользователей** — wall сообщение
2. **Остановка сервисов** — systemd останавливает units
3. **Размонтирование файловых систем**
4. **Синхронизация дисков** — sync
5. **Остановка ядра**
6. **Выключение питания** (если poweroff)

## Команда sync

Записать все буферы на диск:

```bash
$ sync
```

Важно перед аварийным выключением.

## Практические сценарии

### Плановое выключение сервера

```bash
# Уведомить и выключить через час
$ sudo shutdown -h +60 "Плановое обслуживание. Система будет выключена через 60 минут."
```

### Быстрая перезагрузка

```bash
$ sudo reboot
# или
$ sudo shutdown -r now
```

### Выключение в конкретное время

```bash
$ sudo shutdown -h 02:00 "Ночное выключение"
```

### Отмена запланированного выключения

```bash
$ sudo shutdown -c "Выключение отменено администратором"
```

### Принудительная перезагрузка (крайний случай)

```bash
$ sudo reboot -f          # принудительно (не рекомендуется)
```

## Сообщения пользователям

### wall — сообщение всем

```bash
$ sudo wall "Система будет выключена через 10 минут"
```

### Автоматические сообщения shutdown

При использовании shutdown пользователи получают предупреждения:

```
Broadcast message from root@server (pts/0):
The system is going down for maintenance in 10 minutes!
```

## Логи выключения

```bash
$ last -x shutdown
$ last -x reboot
$ journalctl --list-boots   # список загрузок
```

## Аварийные ситуации

### Magic SysRq

Если система полностью зависла, можно использовать Magic SysRq:

```
Alt + SysRq + R  — вернуть клавиатуру
Alt + SysRq + E  — SIGTERM всем процессам
Alt + SysRq + I  — SIGKILL всем процессам
Alt + SysRq + S  — Sync дисков
Alt + SysRq + U  — Размонтировать readonly
Alt + SysRq + B  — Reboot

Мнемоника: REISUB (Busier наоборот)
```

Должен быть включён:
```bash
$ echo 1 | sudo tee /proc/sys/kernel/sysrq
```

### Принудительная перезагрузка

```bash
$ sudo reboot -f            # без остановки сервисов
$ sudo reboot -ff           # немедленно
$ echo b | sudo tee /proc/sysrq-trigger   # через sysrq
```

## Различия команд

| Команда | Действие |
|---------|----------|
| `shutdown -h now` | Безопасное выключение с уведомлениями |
| `shutdown -r now` | Безопасная перезагрузка |
| `poweroff` | Быстрое выключение |
| `reboot` | Быстрая перезагрузка |
| `halt` | Остановить CPU (без выключения питания) |
| `systemctl poweroff` | Через systemd |

## Советы

1. **На серверах** используйте shutdown с задержкой — дайте пользователям время
2. **sync** перед любым нештатным выключением
3. **Проверяйте журналы** после нештатных перезагрузок
4. **Не используйте -f** без крайней необходимости
5. **Magic SysRq** — последний шанс при полном зависании
