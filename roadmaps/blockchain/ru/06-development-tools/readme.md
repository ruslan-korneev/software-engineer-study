# Инструменты разработки

Этот раздел охватывает инструменты и среды разработки для создания блокчейн-приложений.

## Содержание

### IDE для блокчейн-разработки
- [x] [IDE для блокчейн-разработки](./01-ides/readme.md) - обзор и сравнение
  - [x] [Remix IDE](./01-ides/01-remix/readme.md) - браузерная IDE от Ethereum Foundation
  - [x] [VS Code для Solidity](./01-ides/02-vscode/readme.md) - настройка VS Code для разработки

### Фреймворки разработки
- [x] [Фреймворки разработки](./02-frameworks/readme.md) - обзор и сравнение
  - [x] [Hardhat](./02-frameworks/01-hardhat/readme.md) - JavaScript/TypeScript фреймворк
  - [x] [Foundry](./02-frameworks/02-foundry/readme.md) - Rust-based фреймворк с Solidity тестами
  - [x] [Truffle](./02-frameworks/03-truffle/readme.md) - Legacy фреймворк (deprecated)
  - [x] [Brownie](./02-frameworks/04-brownie/readme.md) - Python фреймворк

### Контроль версий
- [x] [Контроль версий](./03-version-control/readme.md) - Git, CI/CD, управление секретами

## Обзор инструментов

### IDE

| Инструмент | Тип | Использование |
|------------|-----|---------------|
| **Remix** | Браузерная IDE | Обучение, прототипирование, отладка |
| **VS Code** | Локальный редактор | Профессиональная разработка |

### Фреймворки

| Фреймворк | Язык тестов | Статус | Рекомендация |
|-----------|-------------|--------|--------------|
| **Hardhat** | JavaScript/TS | Активный | Рекомендуется |
| **Foundry** | Solidity | Активный | Рекомендуется |
| **Truffle** | JavaScript | Deprecated | Только для legacy |
| **Brownie** | Python | Maintenance | Рассмотреть Ape |

## Рекомендуемый стек

### Для начинающих
1. **Remix** - для изучения Solidity
2. **Hardhat** - для первых проектов

### Для профессионалов
1. **VS Code + Solidity расширения**
2. **Foundry** - для тестирования
3. **Hardhat** - для деплоя и скриптов

### Безопасность
- **Git** с правильным `.gitignore`
- **GitHub Secrets** для CI/CD
- **Hardware wallets** для production деплоя
