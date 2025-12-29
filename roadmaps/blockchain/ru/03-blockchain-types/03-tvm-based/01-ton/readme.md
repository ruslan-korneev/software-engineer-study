# TON (The Open Network)

## История создания

### От Telegram к TON

**TON** — один из самых амбициозных блокчейн-проектов, созданный командой Telegram под руководством **Павла Дурова** в 2018 году.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    История развития TON                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  2018 │ Telegram объявляет о TON, ICO собирает $1.7 млрд            │
│       │                                                              │
│  2019 │ Разработка TON Blockchain, тестнет                          │
│       │                                                              │
│  2020 │ SEC подаёт иск против Telegram                              │
│   май │ Telegram официально прекращает работу над TON               │
│       │                                                              │
│  2020 │ Сообщество запускает Free TON (позже Everscale)             │
│  2021 │ Newton (TON Foundation) продолжает развитие оригинального    │
│       │ TON под именем "The Open Network"                           │
│       │                                                              │
│  2022 │ Активное развитие экосистемы, Telegram интеграция           │
│  2023 │ TON Space, Fragment, Telegram Stars                          │
│  2024 │ TON входит в топ-10 криптовалют по капитализации            │
│       │                                                              │
└─────────────────────────────────────────────────────────────────────┘
```

### SEC и проблемы с регуляторами

В октябре 2019 года **SEC (Securities and Exchange Commission)** США подала иск против Telegram:

- Токен **Gram** был признан незарегистрированной ценной бумагой
- Telegram вернул инвесторам **$1.2 млрд**
- Заплатил штраф **$18.5 млн**
- Официально отказался от проекта

### Сообщество подхватывает проект

После отказа Telegram от TON:
1. **NewTON** (позже TON Foundation) — продолжил развитие оригинального кода
2. **Free TON** (позже Everscale) — запустил форк с изменённой экономикой

## Техническая архитектура

### Консенсус: Proof-of-Stake (PoS)

TON использует модифицированный **Byzantine Fault Tolerant (BFT)** PoS:

```
┌────────────────────────────────────────────────────────┐
│              Консенсус в TON                           │
├────────────────────────────────────────────────────────┤
│                                                        │
│  1. Валидаторы стейкают TON (минимум ~300,000 TON)    │
│                          ↓                             │
│  2. Сеть случайно выбирает подмножество валидаторов   │
│                          ↓                             │
│  3. Валидаторы голосуют за блоки (2/3 для подтверж.)  │
│                          ↓                             │
│  4. Блок добавляется в цепочку                        │
│                          ↓                             │
│  5. Награды распределяются пропорционально стейку     │
│                                                        │
│  Время блока: ~5 секунд                               │
│  Финальность: ~6 секунд                               │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Шардинг и бесконечная масштабируемость

TON — один из немногих блокчейнов с **динамическим шардингом**:

```
                    ┌─────────────────────┐
                    │    MASTERCHAIN      │
                    │   (workchain -1)    │
                    │   Хранит:           │
                    │   • Конфигурацию    │
                    │   • Список валид.   │
                    │   • Состояние шардов│
                    └──────────┬──────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
    ┌────┴────┐          ┌─────┴─────┐         ┌─────┴─────┐
    │Workchain│          │ Workchain │         │ Workchain │
    │    0    │          │     1     │         │    ...    │
    │(базовый)│          │(возможный)│         │ (до 2^32) │
    └────┬────┘          └───────────┘         └───────────┘
         │
    ┌────┼────┬────┐
    │    │    │    │
  Shard Shard Shard Shard    ← Динамическое деление
   00    01    10    11        при нагрузке
```

**Ключевые особенности:**

| Компонент | Описание |
|-----------|----------|
| **Masterchain** | Координирует всю сеть, хранит конфигурацию |
| **Workchain** | Независимый блокчейн со своими правилами |
| **Shardchain** | Подмножество workchain, обрабатывает часть аккаунтов |
| **Dynamic split/merge** | Автоматическое деление шардов при нагрузке |

### Структура адресов

```
Полный адрес TON:
┌────────────────────────────────────────────────────────────────┐
│  0:1234567890abcdef1234567890abcdef1234567890abcdef1234567890ab │
│  │ │                                                            │
│  │ └─ 256-bit hash состояния контракта (hex)                   │
│  │                                                              │
│  └─── workchain_id (0 = базовый, -1 = masterchain)             │
└────────────────────────────────────────────────────────────────┘

User-friendly форматы:
• Raw:         0:1234...90ab
• Bounceable:  EQA...xyz (для контрактов)
• Non-bounce:  UQA...xyz (для кошельков)
```

## Смарт-контракты в TON

### Языки программирования

#### FunC — основной язык

```func
#include "stdlib.fc";

;; Хранилище данных контракта
global int ctx_id;
global int ctx_counter;

() load_data() impure inline {
    var ds = get_data().begin_parse();
    ctx_id = ds~load_uint(32);
    ctx_counter = ds~load_uint(64);
}

() save_data() impure inline {
    set_data(begin_cell()
        .store_uint(ctx_id, 32)
        .store_uint(ctx_counter, 64)
    .end_cell());
}

;; Основной обработчик входящих сообщений
() recv_internal(int my_balance, int msg_value, cell in_msg_full, slice in_msg_body) impure {
    if (in_msg_body.slice_empty?()) {
        return ();  ;; Пустое сообщение (просто пополнение)
    }

    load_data();

    int op = in_msg_body~load_uint(32);  ;; Код операции
    int query_id = in_msg_body~load_uint(64);  ;; ID запроса

    if (op == 1) {  ;; Increment counter
        ctx_counter += 1;
        save_data();
        return ();
    }

    if (op == 2) {  ;; Get counter (отправляем ответ)
        slice sender = in_msg_full.begin_parse();
        sender~skip_bits(4);  ;; пропускаем флаги
        slice sender_addr = sender~load_msg_addr();

        send_raw_message(
            begin_cell()
                .store_uint(0x10, 6)  ;; флаги
                .store_slice(sender_addr)
                .store_coins(0)
                .store_uint(0, 107)
                .store_uint(3, 32)  ;; op = response
                .store_uint(query_id, 64)
                .store_uint(ctx_counter, 64)
            .end_cell(),
            64  ;; mode: carry remaining gas
        );
        return ();
    }

    throw(0xffff);  ;; Неизвестная операция
}

;; Get-метод (только для чтения)
int get_counter() method_id {
    load_data();
    return ctx_counter;
}
```

#### Tact — современный высокоуровневый язык

```tact
import "@stdlib/deploy";

// Определение сообщений
message Increment {
    amount: Int as uint32;
}

message Reset {
    newValue: Int as uint64;
}

// Контракт счётчика
contract Counter with Deployable {
    id: Int as uint32;
    counter: Int as uint64;
    owner: Address;

    // Конструктор
    init(id: Int) {
        self.id = id;
        self.counter = 0;
        self.owner = sender();
    }

    // Обработчик сообщения Increment
    receive(msg: Increment) {
        self.counter += msg.amount;
        self.reply("Incremented!".asComment());
    }

    // Обработчик сообщения Reset (только владелец)
    receive(msg: Reset) {
        require(sender() == self.owner, "Only owner can reset");
        self.counter = msg.newValue;
    }

    // Обработчик пустого сообщения (пополнение баланса)
    receive() {
        // Просто принимаем TON
    }

    // Get-методы
    get fun counter(): Int {
        return self.counter;
    }

    get fun id(): Int {
        return self.id;
    }

    get fun owner(): Address {
        return self.owner;
    }
}
```

### Сравнение FunC и Tact

| Аспект | FunC | Tact |
|--------|------|------|
| **Синтаксис** | C-подобный | TypeScript-подобный |
| **Типизация** | Слабая | Строгая |
| **Абстракции** | Низкоуровневые | Высокоуровневые |
| **Контроль** | Полный над TVM | Ограниченный |
| **Безопасность** | Требует опыта | Встроенные проверки |
| **Применение** | Критичные контракты | Быстрая разработка |

## Стандарты токенов

### Jettons (TEP-74)

**Jettons** — стандарт взаимозаменяемых токенов в TON (аналог ERC-20):

```
┌─────────────────────────────────────────────────────────────────┐
│                    Архитектура Jettons                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│         ┌──────────────────────────┐                            │
│         │    Jetton Master         │                            │
│         │  (хранит метаданные)     │                            │
│         │  • total_supply          │                            │
│         │  • name, symbol          │                            │
│         │  • admin_address         │                            │
│         └────────────┬─────────────┘                            │
│                      │                                           │
│      ┌───────────────┼───────────────┐                          │
│      │               │               │                          │
│  ┌───┴───┐      ┌────┴────┐     ┌────┴────┐                     │
│  │Wallet │      │ Wallet  │     │ Wallet  │                     │
│  │User A │      │ User B  │     │ User C  │                     │
│  │bal:100│      │ bal:50  │     │ bal:200 │                     │
│  └───────┘      └─────────┘     └─────────┘                     │
│                                                                  │
│  * Каждый пользователь имеет отдельный контракт-кошелёк        │
│  * Масштабируется через шардинг                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### Пример Jetton Wallet на Tact

```tact
import "@stdlib/deploy";
import "@stdlib/ownable";

message(0x0f8a7ea5) JettonTransfer {
    queryId: Int as uint64;
    amount: Int as coins;
    destination: Address;
    responseDestination: Address;
    customPayload: Cell?;
    forwardTonAmount: Int as coins;
    forwardPayload: Slice as remaining;
}

message(0x7362d09c) JettonTransferNotification {
    queryId: Int as uint64;
    amount: Int as coins;
    sender: Address;
    forwardPayload: Slice as remaining;
}

contract JettonWallet with Ownable {
    balance: Int as coins;
    owner: Address;
    master: Address;

    init(owner: Address, master: Address) {
        self.balance = 0;
        self.owner = owner;
        self.master = master;
    }

    receive(msg: JettonTransfer) {
        require(sender() == self.owner, "Only owner can transfer");
        require(self.balance >= msg.amount, "Insufficient balance");

        self.balance -= msg.amount;

        // Отправляем токены на новый кошелёк
        send(SendParameters{
            to: self.getWalletAddress(msg.destination),
            value: 0,
            mode: SendRemainingValue,
            body: JettonTransferNotification{
                queryId: msg.queryId,
                amount: msg.amount,
                sender: self.owner,
                forwardPayload: msg.forwardPayload
            }.toCell()
        });
    }

    get fun balance(): Int {
        return self.balance;
    }

    // Вычисление адреса кошелька для получателя
    get fun getWalletAddress(owner: Address): Address {
        // ... логика вычисления адреса
        return owner; // упрощённо
    }
}
```

### NFT (TEP-62)

```
┌────────────────────────────────────────────────────────────┐
│                   NFT в TON                                │
├────────────────────────────────────────────────────────────┤
│                                                             │
│     ┌────────────────┐                                     │
│     │ NFT Collection │ ← Коллекция (метаданные, роялти)   │
│     └───────┬────────┘                                     │
│             │                                               │
│    ┌────────┼────────┐                                     │
│    │        │        │                                      │
│ ┌──┴──┐  ┌──┴──┐  ┌──┴──┐                                  │
│ │NFT 1│  │NFT 2│  │NFT 3│ ← Отдельные контракты           │
│ │owner│  │owner│  │owner│                                  │
│ └─────┘  └─────┘  └─────┘                                  │
│                                                             │
│ Особенности:                                               │
│ • Каждый NFT — отдельный смарт-контракт                   │
│ • Поддержка on-chain метаданных                           │
│ • Встроенные роялти для создателей                        │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

## Экосистема TON

### Интеграция с Telegram

```
┌────────────────────────────────────────────────────────────────┐
│              TON + Telegram Экосистема                         │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────────────┐   │
│  │ TON Space   │   │  Fragment   │   │ Telegram Stars      │   │
│  │(Кошелёк в TG)│   │(Юзернеймы  │   │(Внутренняя валюта) │   │
│  │             │   │ как NFT)   │   │                     │   │
│  └─────────────┘   └─────────────┘   └─────────────────────┘   │
│                                                                 │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────────────┐   │
│  │ TON Connect │   │ Mini Apps   │   │ Wallet Bot          │   │
│  │(Auth proto) │   │(TWA/TMA)   │   │ @wallet             │   │
│  └─────────────┘   └─────────────┘   └─────────────────────┘   │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### Ключевые компоненты экосистемы

| Компонент | Описание |
|-----------|----------|
| **TON Space** | Встроенный кошелёк в Telegram |
| **Fragment** | Маркетплейс юзернеймов и номеров как NFT |
| **TON Connect** | Протокол авторизации через кошелёк |
| **Telegram Mini Apps** | Web-приложения внутри Telegram |
| **Telegram Stars** | Внутренняя валюта, конвертируемая в TON |

## Инструменты разработчика

### Blueprint — фреймворк для разработки

```bash
# Создание нового проекта
npm create ton@latest

# Структура проекта Blueprint
my-project/
├── contracts/         # Контракты на FunC/Tact
├── scripts/          # Скрипты деплоя
├── tests/            # Тесты
├── wrappers/         # TypeScript обёртки
└── blueprint.config.ts
```

### Пример теста на TypeScript

```typescript
import { Blockchain, SandboxContract, TreasuryContract } from '@ton/sandbox';
import { Counter } from '../wrappers/Counter';
import { toNano } from '@ton/core';
import '@ton/test-utils';

describe('Counter', () => {
    let blockchain: Blockchain;
    let deployer: SandboxContract<TreasuryContract>;
    let counter: SandboxContract<Counter>;

    beforeEach(async () => {
        blockchain = await Blockchain.create();
        deployer = await blockchain.treasury('deployer');

        counter = blockchain.openContract(
            await Counter.fromInit(0n)
        );

        await counter.send(
            deployer.getSender(),
            { value: toNano('0.05') },
            { $$type: 'Deploy', queryId: 0n }
        );
    });

    it('should increment', async () => {
        const before = await counter.getCounter();

        await counter.send(
            deployer.getSender(),
            { value: toNano('0.05') },
            { $$type: 'Increment', amount: 5n }
        );

        const after = await counter.getCounter();
        expect(after).toEqual(before + 5n);
    });
});
```

### Основные библиотеки

| Библиотека | Назначение |
|------------|------------|
| **@ton/core** | Работа с ячейками, адресами, сообщениями |
| **@ton/crypto** | Криптографические функции |
| **@ton/ton** | Взаимодействие с TON API |
| **@ton/sandbox** | Локальный эмулятор для тестов |
| **@tact-lang/compiler** | Компилятор Tact |

### Инструменты CLI

```bash
# toncli — CLI для FunC
pip install toncli
toncli start wallet
toncli deploy

# Blueprint CLI
npx blueprint create Counter  # Создать контракт
npx blueprint build           # Скомпилировать
npx blueprint test            # Запустить тесты
npx blueprint run             # Запустить скрипт
```

## Преимущества TON

1. **Масштабируемость** — до миллионов TPS благодаря шардингу
2. **Быстрые транзакции** — финальность ~5-6 секунд
3. **Низкие комиссии** — ~$0.01-0.05 за транзакцию
4. **Telegram интеграция** — 900+ млн потенциальных пользователей
5. **Развитая экосистема** — DeFi, NFT, GameFi
6. **Удобство Tact** — простой вход для новых разработчиков

## Недостатки и риски

1. **Сложность TVM** — асинхронная модель требует иного подхода
2. **Молодая экосистема** — меньше аудированных библиотек
3. **Централизация** — зависимость от Telegram и TON Foundation
4. **Регуляторные риски** — история с SEC
5. **Меньше разработчиков** — по сравнению с Ethereum/Solana

## Ресурсы для изучения

- [TON Documentation](https://docs.ton.org/)
- [Tact by Example](https://tact-by-example.org/)
- [TON Dev Community](https://t.me/tondev)
- [TON Cookbook](https://docs.ton.org/develop/smart-contracts/cookbook)
- [Blueprint Template](https://github.com/ton-org/blueprint)
