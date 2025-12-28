# Storage (HDD, SSD)

[prev: 02-memory-ram-rom](./02-memory-ram-rom.md) | [next: 04-motherboard-buses](./04-motherboard-buses.md)

---

## HDD (Hard Disk Drive)

Магнитные вращающиеся диски с головкой чтения/записи.

| Параметр | Значение |
|----------|----------|
| Объём | 1-20 TB |
| Скорость | 80-200 MB/s |
| Время доступа | 5-15 ms |
| IOPS | 100-200 |
| Цена | Дёшево |

**Плюсы:** дёшево, большие объёмы
**Минусы:** медленно, механика ломается

## SSD (Solid State Drive)

Flash-память без движущихся частей.

### Интерфейсы

| Тип | Скорость |
|-----|----------|
| SATA III | до 550 MB/s |
| NVMe PCIe 3.0 | до 3500 MB/s |
| NVMe PCIe 4.0 | до 7000 MB/s |

**Плюсы:** очень быстро, надёжно, тихо
**Минусы:** дороже, ограниченный ресурс записи (TBW)

## HDD vs SSD

| | HDD | SSD NVMe |
|-|-----|----------|
| Чтение | 150 MB/s | 5000 MB/s |
| IOPS | 200 | 500,000 |
| Цена/TB | $25 | $100 |

## Для Backend-разработчика

- **БД (OLTP)** → NVMe SSD (IOPS критичен)
- **Архивы, бэкапы** → HDD
- **RAID 1/10** → отказоустойчивость
- Мониторинг: `iostat -x 1`

---

[prev: 02-memory-ram-rom](./02-memory-ram-rom.md) | [next: 04-motherboard-buses](./04-motherboard-buses.md)
