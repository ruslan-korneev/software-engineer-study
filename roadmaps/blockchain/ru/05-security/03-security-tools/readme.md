# Инструменты безопасности

Безопасность смарт-контрактов критически важна, поскольку ошибки в коде могут привести к потере миллионов долларов. Экосистема блокчейна предоставляет множество инструментов для обнаружения уязвимостей и защиты контрактов.

## Обзор инструментов

### Категории инструментов безопасности

```
┌─────────────────────────────────────────────────────────────────┐
│                  ИНСТРУМЕНТЫ БЕЗОПАСНОСТИ                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │ СТАТИЧЕСКИЙ     │  │ ДИНАМИЧЕСКИЙ    │  │ БИБЛИОТЕКИ      │ │
│  │ АНАЛИЗ          │  │ АНАЛИЗ          │  │ БЕЗОПАСНОСТИ    │ │
│  │                 │  │                 │  │                 │ │
│  │ • Slither       │  │ • Echidna       │  │ • OpenZeppelin  │ │
│  │ • Mythril       │  │ • Foundry Fuzz  │  │ • Solmate       │ │
│  │ • Securify      │  │ • Manticore     │  │ • PRBMath       │ │
│  │ • Solhint       │  │                 │  │                 │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │ ФОРМАЛЬНАЯ      │  │ МОНИТОРИНГ      │  │ АУДИТ           │ │
│  │ ВЕРИФИКАЦИЯ     │  │                 │  │                 │ │
│  │                 │  │                 │  │                 │ │
│  │ • Certora       │  │ • Forta         │  │ • Manual Review │ │
│  │ • Halmos        │  │ • OpenZeppelin  │  │ • Bug Bounty    │ │
│  │ • HEVM          │  │   Defender      │  │                 │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Сравнение подходов

| Подход | Инструменты | Преимущества | Недостатки |
|--------|-------------|--------------|------------|
| **Статический анализ** | Slither, Mythril | Быстрый, находит паттерны | Ложные срабатывания |
| **Фаззинг** | Echidna, Foundry | Находит edge cases | Требует написания свойств |
| **Формальная верификация** | Certora, Halmos | Математические гарантии | Сложность написания спеков |
| **Библиотеки** | OpenZeppelin | Проверенный код | Зависимость от версий |

## Пайплайн безопасности

```
┌──────────────────────────────────────────────────────────────────┐
│                     SECURITY PIPELINE                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Разработка  →  Локальное   →  CI/CD      →  Pre-deploy  →     │
│                  тестирование   проверки      аудит             │
│                                                                  │
│   ┌─────────┐   ┌─────────────┐ ┌─────────┐   ┌──────────┐      │
│   │ Solhint │   │ Unit tests  │ │ Slither │   │ External │      │
│   │ Prettier│   │ Foundry     │ │ Mythril │   │ Audit    │      │
│   └─────────┘   │ Echidna     │ │ Securify│   │ Bug      │      │
│                 │ Coverage    │ └─────────┘   │ Bounty   │      │
│                 └─────────────┘               └──────────┘      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

## Рекомендуемый стек инструментов

### Минимальный набор

1. **Slither** - статический анализ для быстрого обнаружения типичных уязвимостей
2. **Foundry Fuzz** - встроенный фаззинг для тестирования свойств
3. **OpenZeppelin** - безопасные базовые контракты

### Расширенный набор

1. **Slither** + **Mythril** - комбинация статического и символического анализа
2. **Echidna** - продвинутый фаззинг со сложными свойствами
3. **Certora** - формальная верификация для критических протоколов
4. **OpenZeppelin Defender** - мониторинг и автоматизация

## Интеграция в рабочий процесс

```yaml
# .github/workflows/security.yml
name: Security Checks

on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1

      - name: Run Slither
        uses: crytic/slither-action@v0.3.0
        with:
          sarif: results.sarif

      - name: Run Foundry Tests
        run: |
          forge build
          forge test -vvv

      - name: Check Coverage
        run: forge coverage --report summary
```

## Стоимость и доступность

| Инструмент | Лицензия | Стоимость | Уровень сложности |
|------------|----------|-----------|-------------------|
| Slither | AGPL-3.0 | Бесплатно | Низкий |
| Echidna | AGPL-3.0 | Бесплатно | Средний |
| OpenZeppelin | MIT | Бесплатно | Низкий |
| Mythril | MIT | Бесплатно | Средний |
| Certora | Коммерческая | $$$ | Высокий |
| Forta | - | Бесплатно | Средний |

## Что изучить в этом разделе

- [x] [Slither](./01-slither/readme.md) - статический анализатор от Trail of Bits
- [x] [Echidna](./02-echidna/readme.md) - property-based фаззер
- [x] [OpenZeppelin](./03-openzeppelin/readme.md) - библиотека безопасных контрактов

## Best Practices

### Многоуровневая защита

```
┌─────────────────────────────────────────────────────────────┐
│                   DEFENSE IN DEPTH                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Level 1: Code Quality                                     │
│   ├── Linters (Solhint)                                    │
│   ├── Code formatting                                       │
│   └── Naming conventions                                    │
│                                                             │
│   Level 2: Automated Analysis                               │
│   ├── Slither (pattern detection)                          │
│   ├── Mythril (symbolic execution)                         │
│   └── Gas optimization checks                               │
│                                                             │
│   Level 3: Property Testing                                 │
│   ├── Unit tests (Foundry/Hardhat)                         │
│   ├── Fuzz testing (Echidna/Foundry)                       │
│   └── Invariant testing                                     │
│                                                             │
│   Level 4: Formal Verification (optional)                   │
│   ├── Certora                                              │
│   └── Halmos                                                │
│                                                             │
│   Level 5: External Review                                  │
│   ├── Professional audit                                    │
│   └── Bug bounty program                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Чеклист перед деплоем

- [ ] Все тесты проходят
- [ ] Slither не показывает high/medium уязвимостей
- [ ] Покрытие тестами > 90%
- [ ] Фаззинг-тесты не находят нарушений
- [ ] Код прошел peer review
- [ ] Проведен аудит (для продакшена с TVL)
- [ ] Настроен мониторинг
- [ ] Подготовлен план incident response

## Ресурсы

- [Trail of Bits Guidelines](https://github.com/crytic/building-secure-contracts)
- [OpenZeppelin Security Best Practices](https://docs.openzeppelin.com/contracts/5.x/)
- [Secureum Bootcamp](https://github.com/x676f64/secureum-mind_map)
- [Smart Contract Security Field Guide](https://scsfg.io/)
