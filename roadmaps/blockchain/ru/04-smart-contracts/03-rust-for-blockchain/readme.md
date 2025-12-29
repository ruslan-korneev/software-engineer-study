# Rust для блокчейна

## Почему Rust выбирают для блокчейн-разработки

Rust стал одним из самых популярных языков для разработки блокчейн-систем благодаря уникальному сочетанию характеристик:

### Ключевые преимущества

1. **Безопасность памяти** — система владения (ownership) и заимствования (borrowing) исключает целые классы ошибок: утечки памяти, гонки данных, use-after-free
2. **Производительность** — компиляция в нативный код без сборщика мусора, сравнимая с C/C++
3. **Предсказуемость** — отсутствие GC означает отсутствие непредсказуемых пауз
4. **Строгая типизация** — ошибки обнаруживаются на этапе компиляции
5. **Современные инструменты** — Cargo, rustfmt, clippy, отличная документация

### Блокчейны на Rust

- **Solana** — высокопроизводительный блокчейн (65,000+ TPS)
- **Polkadot/Substrate** — экосистема парачейнов
- **Near Protocol** — шардированный блокчейн
- **Cosmos (CosmWasm)** — смарт-контракты для Cosmos SDK
- **Aptos и Sui** — новые L1 блокчейны

---

## Solana программы (Anchor Framework)

Solana использует уникальную модель, где смарт-контракты называются **программами**. Anchor — это фреймворк, упрощающий разработку.

### Базовая структура Anchor программы

```rust
use anchor_lang::prelude::*;

// ID программы (генерируется при деплое)
declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkgLwJNEsQRzP");

#[program]
pub mod counter {
    use super::*;

    // Инициализация счётчика
    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        let counter = &mut ctx.accounts.counter;
        counter.count = 0;
        counter.authority = ctx.accounts.authority.key();
        Ok(())
    }

    // Увеличение счётчика
    pub fn increment(ctx: Context<Increment>) -> Result<()> {
        let counter = &mut ctx.accounts.counter;
        counter.count += 1;
        Ok(())
    }
}

// Структура аккаунта для хранения данных
#[account]
pub struct Counter {
    pub count: u64,
    pub authority: Pubkey,
}

// Контекст для инициализации
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,
        payer = authority,
        space = 8 + 8 + 32  // discriminator + count + pubkey
    )]
    pub counter: Account<'info, Counter>,
    #[account(mut)]
    pub authority: Signer<'info>,
    pub system_program: Program<'info, System>,
}

// Контекст для инкремента
#[derive(Accounts)]
pub struct Increment<'info> {
    #[account(
        mut,
        has_one = authority
    )]
    pub counter: Account<'info, Counter>,
    pub authority: Signer<'info>,
}
```

### Особенности Solana/Anchor

- **Account Model** — данные хранятся в отдельных аккаунтах
- **Rent** — аккаунты платят за хранение или должны быть "rent-exempt"
- **PDA (Program Derived Addresses)** — детерминированные адреса, контролируемые программой
- **CPI (Cross-Program Invocation)** — вызовы между программами

---

## Substrate/Polkadot разработка

Substrate — фреймворк для создания блокчейнов, используемый в экосистеме Polkadot.

### Пример паллета (модуля) Substrate

```rust
#![cfg_attr(not(feature = "std"), no_std)]

pub use pallet::*;

#[frame_support::pallet]
pub mod pallet {
    use frame_support::pallet_prelude::*;
    use frame_system::pallet_prelude::*;

    #[pallet::pallet]
    pub struct Pallet<T>(_);

    // Конфигурация паллета
    #[pallet::config]
    pub trait Config: frame_system::Config {
        type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
        #[pallet::constant]
        type MaxLength: Get<u32>;
    }

    // Хранилище данных
    #[pallet::storage]
    #[pallet::getter(fn something)]
    pub type Something<T: Config> = StorageValue<_, u32>;

    #[pallet::storage]
    pub type Balances<T: Config> = StorageMap<
        _,
        Blake2_128Concat,
        T::AccountId,
        u128,
        ValueQuery
    >;

    // События
    #[pallet::event]
    #[pallet::generate_deposit(pub(super) fn deposit_event)]
    pub enum Event<T: Config> {
        SomethingStored { value: u32, who: T::AccountId },
        Transferred { from: T::AccountId, to: T::AccountId, amount: u128 },
    }

    // Ошибки
    #[pallet::error]
    pub enum Error<T> {
        NoneValue,
        StorageOverflow,
        InsufficientBalance,
    }

    // Экструзики (транзакции)
    #[pallet::call]
    impl<T: Config> Pallet<T> {
        #[pallet::call_index(0)]
        #[pallet::weight(10_000)]
        pub fn do_something(origin: OriginFor<T>, something: u32) -> DispatchResult {
            let who = ensure_signed(origin)?;

            <Something<T>>::put(something);

            Self::deposit_event(Event::SomethingStored { value: something, who });
            Ok(())
        }

        #[pallet::call_index(1)]
        #[pallet::weight(10_000)]
        pub fn transfer(
            origin: OriginFor<T>,
            to: T::AccountId,
            amount: u128
        ) -> DispatchResult {
            let from = ensure_signed(origin)?;

            let from_balance = Balances::<T>::get(&from);
            ensure!(from_balance >= amount, Error::<T>::InsufficientBalance);

            Balances::<T>::mutate(&from, |b| *b -= amount);
            Balances::<T>::mutate(&to, |b| *b += amount);

            Self::deposit_event(Event::Transferred { from, to, amount });
            Ok(())
        }
    }
}
```

---

## Near Protocol смарт-контракты

Near использует WebAssembly (Wasm) и предоставляет удобный SDK для Rust.

### Пример контракта Near

```rust
use near_sdk::borsh::{self, BorshDeserialize, BorshSerialize};
use near_sdk::collections::LookupMap;
use near_sdk::{env, near_bindgen, AccountId, Balance, Promise};

#[near_bindgen]
#[derive(BorshDeserialize, BorshSerialize)]
pub struct TokenContract {
    balances: LookupMap<AccountId, Balance>,
    total_supply: Balance,
    owner: AccountId,
}

impl Default for TokenContract {
    fn default() -> Self {
        panic!("Contract should be initialized before usage")
    }
}

#[near_bindgen]
impl TokenContract {
    #[init]
    pub fn new(total_supply: Balance) -> Self {
        assert!(!env::state_exists(), "Already initialized");
        let owner = env::predecessor_account_id();

        let mut balances = LookupMap::new(b"b");
        balances.insert(&owner, &total_supply);

        Self {
            balances,
            total_supply,
            owner,
        }
    }

    pub fn get_balance(&self, account_id: AccountId) -> Balance {
        self.balances.get(&account_id).unwrap_or(0)
    }

    pub fn transfer(&mut self, to: AccountId, amount: Balance) {
        let sender = env::predecessor_account_id();
        let sender_balance = self.get_balance(sender.clone());

        assert!(sender_balance >= amount, "Insufficient balance");

        self.balances.insert(&sender, &(sender_balance - amount));

        let receiver_balance = self.get_balance(to.clone());
        self.balances.insert(&to, &(receiver_balance + amount));

        env::log_str(&format!("Transfer {} from {} to {}", amount, sender, to));
    }

    // Payable функция — принимает NEAR
    #[payable]
    pub fn deposit(&mut self) {
        let account = env::predecessor_account_id();
        let amount = env::attached_deposit();

        let balance = self.get_balance(account.clone());
        self.balances.insert(&account, &(balance + amount));
    }

    // Вывод NEAR
    pub fn withdraw(&mut self, amount: Balance) -> Promise {
        let account = env::predecessor_account_id();
        let balance = self.get_balance(account.clone());

        assert!(balance >= amount, "Insufficient balance");

        self.balances.insert(&account, &(balance - amount));

        Promise::new(account).transfer(amount)
    }
}
```

---

## Сравнение Rust и Solidity

| Аспект | Rust | Solidity |
|--------|------|----------|
| **Безопасность памяти** | Гарантируется компилятором | Управляется EVM |
| **Кривая обучения** | Крутая | Умеренная |
| **Производительность** | Очень высокая | Ограничена EVM |
| **Экосистема** | Solana, Near, Polkadot | Ethereum, EVM-сети |
| **Инструменты** | Cargo, clippy | Hardhat, Foundry |
| **Типизация** | Строгая, выразительная | Строгая, базовая |
| **Reentrancy** | Предотвращается дизайном | Требует модификаторов |
| **Газ/вычисления** | Compute units | Gas |

### Когда выбирать Rust

- Высокопроизводительные приложения
- Сложная бизнес-логика
- Потребность в безопасности на уровне языка
- Работа с Solana, Near, Polkadot

### Когда выбирать Solidity

- EVM-совместимые сети
- Большое количество готовых библиотек (OpenZeppelin)
- Быстрый старт для новичков
- DeFi протоколы на Ethereum

---

## Инструменты и экосистема

### Solana

```bash
# Установка Solana CLI
sh -c "$(curl -sSfL https://release.solana.com/stable/install)"

# Установка Anchor
cargo install --git https://github.com/coral-xyz/anchor anchor-cli

# Создание проекта
anchor init my_program
cd my_program
anchor build
anchor test
```

### Substrate

```bash
# Установка Substrate
curl https://getsubstrate.io -sSf | bash

# Создание ноды
substrate-node-new my-node
cd my-node
cargo build --release
```

### Near

```bash
# Установка near-cli
npm install -g near-cli

# Создание проекта
cargo new --lib my_contract
cd my_contract
# Добавить near-sdk в Cargo.toml

# Сборка для Wasm
cargo build --target wasm32-unknown-unknown --release
```

### Полезные библиотеки

| Библиотека | Назначение |
|------------|------------|
| `anchor-lang` | Фреймворк для Solana |
| `near-sdk` | SDK для Near Protocol |
| `ink!` | Смарт-контракты для Substrate |
| `cosmwasm-std` | SDK для CosmWasm |
| `borsh` | Эффективная сериализация |
| `serde` | JSON сериализация |

---

## Лучшие практики

### Безопасность

```rust
// 1. Всегда проверяйте права доступа
pub fn admin_only(ctx: Context<AdminAction>) -> Result<()> {
    require!(
        ctx.accounts.authority.key() == ctx.accounts.config.admin,
        ErrorCode::Unauthorized
    );
    // ...
}

// 2. Избегайте переполнения
let result = amount.checked_add(fee).ok_or(ErrorCode::Overflow)?;

// 3. Используйте константы для магических чисел
const MAX_SUPPLY: u64 = 1_000_000_000;
const FEE_BASIS_POINTS: u64 = 100; // 1%
```

### Оптимизация

```rust
// 1. Минимизируйте размер хранимых данных
#[account]
pub struct CompactData {
    pub value: u32,      // 4 байта вместо u64
    pub flags: u8,       // битовые флаги
}

// 2. Используйте zero-copy для больших структур (Anchor)
#[account(zero_copy)]
pub struct LargeData {
    pub items: [u64; 1000],
}

// 3. Batch операции где возможно
pub fn batch_transfer(ctx: Context<BatchTransfer>, transfers: Vec<Transfer>) -> Result<()> {
    for t in transfers {
        // Обработка в одной транзакции
    }
    Ok(())
}
```

---

## Полезные ресурсы

- [The Rust Book](https://doc.rust-lang.org/book/) — официальное руководство по Rust
- [Anchor Book](https://www.anchor-lang.com/) — документация Anchor
- [Solana Cookbook](https://solanacookbook.com/) — рецепты для Solana
- [Near Docs](https://docs.near.org/) — документация Near Protocol
- [Substrate Docs](https://docs.substrate.io/) — документация Substrate
- [Rustlings](https://github.com/rust-lang/rustlings) — интерактивные упражнения

---

## Заключение

Rust становится стандартом для высокопроизводительных блокчейнов благодаря:

1. **Безопасности** — предотвращение ошибок на этапе компиляции
2. **Производительности** — нативная скорость выполнения
3. **Надёжности** — строгая система типов
4. **Экосистеме** — активное сообщество и качественные инструменты

Изучение Rust для блокчейн-разработки требует времени, но даёт доступ к передовым платформам и позволяет писать более безопасный и эффективный код.
