# VS Code для Solidity

## Почему VS Code для блокчейн-разработки

**Visual Studio Code** — самый популярный редактор кода для разработки смарт-контрактов. В отличие от браузерного Remix, VS Code предоставляет полноценную IDE для профессиональной разработки.

### Преимущества VS Code

- **Локальная разработка** — полный контроль над файлами и проектом
- **Интеграция с Git** — версионирование из коробки
- **Богатая экосистема расширений** — тысячи плагинов
- **Интеграция с фреймворками** — Hardhat, Foundry, Truffle
- **Кроссплатформенность** — Windows, macOS, Linux
- **Кастомизация** — настройка под любые нужды
- **Интегрированный терминал** — выполнение команд без переключения окон

### Установка

Скачайте VS Code с официального сайта: [https://code.visualstudio.com](https://code.visualstudio.com)

```bash
# macOS через Homebrew
brew install --cask visual-studio-code

# Ubuntu/Debian
sudo apt install code

# Windows через Chocolatey
choco install vscode
```

---

## Основные расширения для Solidity

### 1. Solidity (Juan Blanco)

**Самое популярное расширение** для разработки на Solidity.

**Установка:**
```
ID: JuanBlanco.solidity
```

Или через командную строку:
```bash
code --install-extension JuanBlanco.solidity
```

**Возможности:**
- Подсветка синтаксиса Solidity
- Автодополнение кода (IntelliSense)
- Компиляция контрактов (Ctrl/Cmd + F5)
- Переход к определению (Go to Definition)
- Поиск всех ссылок (Find All References)
- Code snippets
- Поддержка Foundry и Hardhat
- Линтинг через solhint

**Настройки (settings.json):**
```json
{
    "solidity.compileUsingRemoteVersion": "v0.8.20",
    "solidity.defaultCompiler": "remote",
    "solidity.enableLocalNodeCompiler": false,
    "solidity.packageDefaultDependenciesContractsDirectory": "src",
    "solidity.packageDefaultDependenciesDirectory": "lib"
}
```

### 2. Hardhat for Visual Studio Code (Nomic Foundation)

Официальное расширение от создателей Hardhat.

**Установка:**
```
ID: NomicFoundation.hardhat-solidity
```

**Возможности:**
- Продвинутое автодополнение
- Навигация по коду (Go to Definition, Find References)
- Интеграция с Hardhat проектами
- Подсказки для импортов
- Быстрые исправления (Quick Fixes)
- Inline validation

**Важно:** Расширения Juan Blanco и Nomic Foundation могут конфликтовать. Рекомендуется использовать только одно:
- **Juan Blanco** — для универсальной разработки
- **Nomic Foundation** — для Hardhat-проектов

### 3. Solidity Visual Developer (tintinweb)

Расширенные визуальные инструменты:

```
ID: tintinweb.solidity-visual-auditor
```

**Возможности:**
- Визуализация контрактов (UML-диаграммы)
- Анализ безопасности
- Подсветка уязвимостей
- Graph view для зависимостей
- Code lens с метриками

---

## Настройка форматирования и линтинга

### Prettier для Solidity

Установите расширение Prettier:
```
ID: esbenp.prettier-vscode
```

И плагин для Solidity:
```bash
npm install --save-dev prettier prettier-plugin-solidity
```

**Конфигурация (.prettierrc):**
```json
{
    "plugins": ["prettier-plugin-solidity"],
    "overrides": [
        {
            "files": "*.sol",
            "options": {
                "parser": "solidity-parse",
                "printWidth": 100,
                "tabWidth": 4,
                "useTabs": false,
                "singleQuote": false,
                "bracketSpacing": false,
                "explicitTypes": "always"
            }
        }
    ]
}
```

**Настройка VS Code для автоформатирования:**
```json
{
    "editor.formatOnSave": true,
    "[solidity]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode"
    }
}
```

### Solhint — линтер для Solidity

```bash
npm install --save-dev solhint
```

**Конфигурация (.solhint.json):**
```json
{
    "extends": "solhint:recommended",
    "plugins": [],
    "rules": {
        "compiler-version": ["error", "^0.8.0"],
        "func-visibility": ["warn", {"ignoreConstructors": true}],
        "max-line-length": ["warn", 120],
        "not-rely-on-time": "warn",
        "reason-string": ["warn", {"maxLength": 64}],
        "no-inline-assembly": "off",
        "no-empty-blocks": "off"
    }
}
```

**Запуск линтера:**
```bash
npx solhint 'contracts/**/*.sol'
```

### Slither — статический анализатор

```bash
pip install slither-analyzer

# Анализ проекта
slither . --config-file slither.config.json
```

**Конфигурация (slither.config.json):**
```json
{
    "detectors_to_exclude": "naming-convention,solc-version",
    "filter_paths": "node_modules,lib"
}
```

---

## Snippets для Solidity

### Встроенные сниппеты (Juan Blanco)

Расширение Juan Blanco включает множество готовых сниппетов:

| Триггер | Генерируемый код |
|---------|------------------|
| `contract` | Базовый контракт |
| `function` | Функция |
| `event` | Событие |
| `modifier` | Модификатор |
| `mapping` | Маппинг |
| `struct` | Структура |
| `enum` | Перечисление |
| `require` | require() проверка |
| `constructor` | Конструктор |

### Пользовательские сниппеты

Создайте файл `.vscode/solidity.code-snippets`:

```json
{
    "SPDX License MIT": {
        "prefix": "spdx",
        "body": [
            "// SPDX-License-Identifier: MIT"
        ],
        "description": "SPDX License Identifier"
    },
    "Solidity Contract Template": {
        "prefix": "sol-contract",
        "body": [
            "// SPDX-License-Identifier: MIT",
            "pragma solidity ^0.8.20;",
            "",
            "/**",
            " * @title ${1:ContractName}",
            " * @notice ${2:Description}",
            " */",
            "contract ${1:ContractName} {",
            "    // State variables",
            "    address public owner;",
            "",
            "    // Events",
            "    event ${3:EventName}(${4:params});",
            "",
            "    // Errors",
            "    error Unauthorized();",
            "",
            "    // Modifiers",
            "    modifier onlyOwner() {",
            "        if (msg.sender != owner) revert Unauthorized();",
            "        _;",
            "    }",
            "",
            "    constructor() {",
            "        owner = msg.sender;",
            "    }",
            "",
            "    $0",
            "}"
        ],
        "description": "Solidity Contract Template"
    },
    "ERC20 Import": {
        "prefix": "erc20",
        "body": [
            "import \"@openzeppelin/contracts/token/ERC20/ERC20.sol\";"
        ],
        "description": "Import OpenZeppelin ERC20"
    },
    "Custom Error": {
        "prefix": "error",
        "body": [
            "error ${1:ErrorName}(${2:params});"
        ],
        "description": "Custom Solidity error"
    },
    "NatSpec Function": {
        "prefix": "natspec-func",
        "body": [
            "/**",
            " * @notice ${1:Description}",
            " * @param ${2:paramName} ${3:paramDescription}",
            " * @return ${4:returnDescription}",
            " */",
            "function ${5:functionName}(${6:params}) public ${7:view} returns (${8:returnType}) {",
            "    $0",
            "}"
        ],
        "description": "Function with NatSpec documentation"
    }
}
```

---

## Интеграция с Hardhat

### Структура Hardhat проекта

```
my-project/
├── contracts/
│   └── MyContract.sol
├── scripts/
│   └── deploy.js
├── test/
│   └── MyContract.test.js
├── hardhat.config.js
├── package.json
└── .vscode/
    └── settings.json
```

### Настройка VS Code для Hardhat

**.vscode/settings.json:**
```json
{
    "solidity.packageDefaultDependenciesDirectory": "node_modules",
    "solidity.packageDefaultDependenciesContractsDirectory": "",
    "solidity.compileUsingRemoteVersion": "v0.8.20",
    "editor.formatOnSave": true,
    "[solidity]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode"
    },
    "[javascript]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode"
    },
    "[typescript]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode"
    }
}
```

### VS Code Tasks для Hardhat

**.vscode/tasks.json:**
```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Compile",
            "type": "shell",
            "command": "npx hardhat compile",
            "group": "build",
            "problemMatcher": []
        },
        {
            "label": "Test",
            "type": "shell",
            "command": "npx hardhat test",
            "group": "test",
            "problemMatcher": []
        },
        {
            "label": "Deploy Local",
            "type": "shell",
            "command": "npx hardhat run scripts/deploy.js --network localhost",
            "problemMatcher": []
        },
        {
            "label": "Start Node",
            "type": "shell",
            "command": "npx hardhat node",
            "isBackground": true,
            "problemMatcher": []
        }
    ]
}
```

Запуск: **Ctrl/Cmd + Shift + P** → "Tasks: Run Task"

### Debug Configuration

**.vscode/launch.json:**
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "node",
            "request": "launch",
            "name": "Hardhat Test",
            "program": "${workspaceFolder}/node_modules/hardhat/internal/cli/cli.js",
            "args": ["test"],
            "console": "integratedTerminal"
        },
        {
            "type": "node",
            "request": "launch",
            "name": "Deploy Script",
            "program": "${workspaceFolder}/node_modules/hardhat/internal/cli/cli.js",
            "args": ["run", "scripts/deploy.js"],
            "console": "integratedTerminal"
        }
    ]
}
```

---

## Интеграция с Foundry

### Структура Foundry проекта

```
my-foundry-project/
├── src/
│   └── MyContract.sol
├── test/
│   └── MyContract.t.sol
├── script/
│   └── Deploy.s.sol
├── lib/
│   └── forge-std/
├── foundry.toml
└── .vscode/
    └── settings.json
```

### Настройка VS Code для Foundry

**.vscode/settings.json:**
```json
{
    "solidity.packageDefaultDependenciesDirectory": "lib",
    "solidity.packageDefaultDependenciesContractsDirectory": "src",
    "solidity.compileUsingRemoteVersion": "v0.8.20",
    "solidity.formatter": "forge",
    "editor.formatOnSave": true
}
```

### Remappings для Foundry

**remappings.txt:**
```
forge-std/=lib/forge-std/src/
@openzeppelin/=lib/openzeppelin-contracts/
solmate/=lib/solmate/src/
```

VS Code автоматически использует этот файл для разрешения импортов.

### Tasks для Foundry

**.vscode/tasks.json:**
```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Forge Build",
            "type": "shell",
            "command": "forge build",
            "group": "build"
        },
        {
            "label": "Forge Test",
            "type": "shell",
            "command": "forge test -vvv",
            "group": "test"
        },
        {
            "label": "Forge Test Specific",
            "type": "shell",
            "command": "forge test --match-test ${input:testName} -vvv",
            "problemMatcher": []
        },
        {
            "label": "Forge Coverage",
            "type": "shell",
            "command": "forge coverage",
            "problemMatcher": []
        }
    ],
    "inputs": [
        {
            "id": "testName",
            "description": "Test function name",
            "type": "promptString"
        }
    ]
}
```

---

## Полезные расширения

### GitLens

```
ID: eamodio.gitlens
```

Расширенная интеграция с Git:
- Blame annotations в каждой строке
- История файла
- Сравнение веток
- Визуализация commit graph

### Error Lens

```
ID: usernamehw.errorlens
```

Показывает ошибки и предупреждения прямо в строке кода:
- Мгновенная видимость проблем
- Цветовая кодировка серьезности
- Встроенные подсказки

### Todo Tree

```
ID: Gruntfuggly.todo-tree
```

Отслеживание TODO/FIXME комментариев:
```solidity
// TODO: Add access control
// FIXME: Potential reentrancy
// HACK: Temporary workaround
```

### Better Comments

```
ID: aaron-bond.better-comments
```

Цветовая подсветка комментариев:
```solidity
// ! Важное предупреждение (красный)
// ? Вопрос (синий)
// TODO: Задача (оранжевый)
// * Информация (зеленый)
```

### Markdown Preview Enhanced

```
ID: shd101wyy.markdown-preview-enhanced
```

Просмотр README и документации:
- Поддержка диаграмм (Mermaid)
- Математические формулы
- Экспорт в PDF

### Thunder Client

```
ID: rangav.vscode-thunder-client
```

REST-клиент для тестирования API:
- Взаимодействие с JSON-RPC
- Тестирование backend endpoints
- Коллекции запросов

### Docker

```
ID: ms-azuretools.vscode-docker
```

Управление Docker-контейнерами:
- Запуск локальных нод
- Управление Ganache/Anvil
- Просмотр логов

---

## Keyboard Shortcuts

### Навигация

| Комбинация | Действие |
|------------|----------|
| F12 | Go to Definition |
| Shift + F12 | Find All References |
| Ctrl/Cmd + Click | Go to Definition |
| Ctrl/Cmd + P | Quick Open файла |
| Ctrl/Cmd + Shift + O | Go to Symbol |
| Ctrl/Cmd + G | Go to Line |

### Редактирование

| Комбинация | Действие |
|------------|----------|
| Ctrl/Cmd + D | Select next occurrence |
| Ctrl/Cmd + Shift + L | Select all occurrences |
| Alt + Up/Down | Move line |
| Ctrl/Cmd + / | Toggle comment |
| Ctrl/Cmd + Shift + K | Delete line |
| F2 | Rename symbol |

### Терминал и задачи

| Комбинация | Действие |
|------------|----------|
| Ctrl/Cmd + ` | Toggle terminal |
| Ctrl/Cmd + Shift + B | Run build task |
| Ctrl/Cmd + Shift + P | Command Palette |

---

## Рекомендуемая конфигурация

### Полный settings.json для Solidity

```json
{
    // Solidity
    "solidity.compileUsingRemoteVersion": "v0.8.20",
    "solidity.packageDefaultDependenciesDirectory": "node_modules",
    "solidity.packageDefaultDependenciesContractsDirectory": "",
    "solidity.linter": "solhint",

    // Editor
    "editor.formatOnSave": true,
    "editor.rulers": [100],
    "editor.tabSize": 4,
    "editor.detectIndentation": false,
    "editor.minimap.enabled": false,

    // Files
    "files.autoSave": "afterDelay",
    "files.autoSaveDelay": 1000,
    "files.associations": {
        "*.sol": "solidity"
    },

    // Formatters
    "[solidity]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "editor.tabSize": 4
    },
    "[javascript]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode"
    },
    "[typescript]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode"
    },
    "[json]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode"
    },

    // Git
    "git.autofetch": true,
    "git.confirmSync": false,

    // Terminal
    "terminal.integrated.defaultProfile.osx": "zsh",
    "terminal.integrated.fontSize": 13,

    // Explorer
    "explorer.confirmDelete": false,
    "explorer.confirmDragAndDrop": false
}
```

---

## Заключение

VS Code — идеальный выбор для профессиональной разработки смарт-контрактов:

1. **Установите базовые расширения:**
   - Juan Blanco Solidity или Nomic Foundation Hardhat
   - Prettier с плагином Solidity
   - GitLens, Error Lens

2. **Настройте проект:**
   - settings.json с правильными путями
   - tasks.json для автоматизации
   - Линтинг через solhint

3. **Используйте интеграцию:**
   - Hardhat или Foundry
   - Git для версионирования
   - Терминал для команд

VS Code в сочетании с Hardhat/Foundry обеспечивает профессиональный workflow для разработки блокчейн-проектов любой сложности.
