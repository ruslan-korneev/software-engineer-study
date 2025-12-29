# Solana

## История создания

**Solana** была основана **Anatoly Yakovenko** в 2017 году. До этого Яковенко работал в Qualcomm над операционными системами. Идея Solana родилась из понимания того, что блокчейн-сети неэффективно используют время.

**Ключевые даты:**
- **2017** - Публикация whitepaper Proof of History
- **2018** - Основание Solana Labs
- **Март 2020** - Запуск mainnet beta
- **2021** - Взрывной рост экосистемы, токен SOL достиг $260

## Технические инновации

### Proof of History (PoH)

Proof of History - это **криптографические часы**, позволяющие доказать, что событие произошло в определенный момент времени.

```
PoH последовательность:
hash₀ = SHA256(seed)
hash₁ = SHA256(hash₀)
hash₂ = SHA256(hash₁)
...
hashₙ = SHA256(hashₙ₋₁)

Каждый хеш зависит от предыдущего → временная метка без синхронизации
```

**Преимущества:**
- Не нужна синхронизация между нодами для определения порядка
- Валидаторы могут работать параллельно
- Исторический порядок транзакций доказуем

### Tower BFT

Tower BFT - это адаптация PBFT консенсуса, оптимизированная под PoH:

```
┌─────────────────────────────────────────────────────────────┐
│                    Tower BFT Voting                          │
├─────────────────────────────────────────────────────────────┤
│  Slot 1: Vote → lockout 2 slots                             │
│  Slot 2: Vote → lockout 4 slots  (2¹ × 2)                   │
│  Slot 3: Vote → lockout 8 slots  (2² × 2)                   │
│  ...                                                         │
│  Slot n: Vote → lockout 2ⁿ slots                            │
├─────────────────────────────────────────────────────────────┤
│  Чем больше голосов подряд, тем сложнее откатить            │
└─────────────────────────────────────────────────────────────┘
```

### Sealevel - Параллельное выполнение

Solana может обрабатывать тысячи транзакций параллельно благодаря:

```rust
// Транзакция указывает все аккаунты, которые будет использовать
Transaction {
    // Если аккаунты не пересекаются - параллельное выполнение
    accounts: [
        AccountMeta { pubkey: user_a, is_signer: true, is_writable: true },
        AccountMeta { pubkey: token_account_a, is_signer: false, is_writable: true },
    ],
    instructions: [...],
}
```

## Архитектура сети

### Slots и Epochs

```
┌─────────────────────────────────────────────────────────────┐
│                        Epoch (432,000 slots ≈ 2-3 дня)      │
├─────────────────────────────────────────────────────────────┤
│ Slot 1 │ Slot 2 │ Slot 3 │ ... │ Slot 432,000             │
│ 400ms  │ 400ms  │ 400ms  │     │                          │
├─────────────────────────────────────────────────────────────┤
│ Leader │ Leader │ Leader │     │ Leader                    │
│  (Val1)│ (Val2) │ (Val1) │     │ (ValN)                   │
└─────────────────────────────────────────────────────────────┘
```

- **Slot** - временной интервал (~400ms), в течение которого лидер создает блок
- **Epoch** - 432,000 слотов (~2-3 дня), после чего пересчитываются ставки и лидеры
- **Leader Schedule** - детерминированно определяется в начале эпохи

### Типы нод

| Тип | Описание | Требования |
|-----|----------|------------|
| **Validator** | Участвует в консенсусе, голосует | 128GB RAM, 12+ cores |
| **RPC Node** | Обслуживает запросы клиентов | 256GB RAM, высокая пропускная способность |
| **Archive Node** | Хранит всю историю | Терабайты хранилища |

## Модель аккаунтов

Solana использует уникальную модель, где **программы и данные разделены**:

```
┌────────────────────┐     ┌────────────────────┐
│   Program Account  │     │    Data Account    │
├────────────────────┤     ├────────────────────┤
│ pubkey: Abc123...  │     │ pubkey: Xyz789...  │
│ owner: BPF Loader  │     │ owner: Abc123...   │
│ executable: true   │     │ executable: false  │
│ data: [bytecode]   │     │ data: [user_data]  │
│ lamports: 1000000  │     │ lamports: 2039280  │
└────────────────────┘     └────────────────────┘
        │                          │
        │    Программа владеет     │
        └──────────────────────────┘
```

### Rent (Аренда)

Аккаунты должны поддерживать минимальный баланс для хранения данных:

```rust
// Минимальный баланс для аккаунта
let rent = Rent::get()?;
let min_balance = rent.minimum_balance(data_size);
// Примерно 0.00203928 SOL за 1KB данных
```

## Программирование на Solana

### Структура программы

```rust
use anchor_lang::prelude::*;

declare_id!("YourProgramIdHere11111111111111111111111111");

#[program]
pub mod hello_solana {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>, data: u64) -> Result<()> {
        let my_account = &mut ctx.accounts.my_account;
        my_account.data = data;
        msg!("Data stored: {}", data);
        Ok(())
    }

    pub fn update(ctx: Context<Update>, new_data: u64) -> Result<()> {
        let my_account = &mut ctx.accounts.my_account;
        my_account.data = new_data;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = user, space = 8 + 8)]
    pub my_account: Account<'info, MyAccount>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Update<'info> {
    #[account(mut)]
    pub my_account: Account<'info, MyAccount>,
}

#[account]
pub struct MyAccount {
    pub data: u64,
}
```

### Program Derived Addresses (PDA)

PDA позволяют программам детерминированно создавать адреса:

```rust
// Создание PDA
let (pda, bump) = Pubkey::find_program_address(
    &[
        b"user-stats",
        user.key().as_ref(),
    ],
    program_id,
);

// Использование в Anchor
#[derive(Accounts)]
pub struct CreateUserStats<'info> {
    #[account(
        init,
        payer = user,
        space = 8 + UserStats::INIT_SPACE,
        seeds = [b"user-stats", user.key().as_ref()],
        bump
    )]
    pub user_stats: Account<'info, UserStats>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

### Cross-Program Invocation (CPI)

```rust
use anchor_spl::token::{self, Transfer, Token};

pub fn transfer_tokens(ctx: Context<TransferTokens>, amount: u64) -> Result<()> {
    let cpi_accounts = Transfer {
        from: ctx.accounts.from_token_account.to_account_info(),
        to: ctx.accounts.to_token_account.to_account_info(),
        authority: ctx.accounts.authority.to_account_info(),
    };

    let cpi_program = ctx.accounts.token_program.to_account_info();
    let cpi_ctx = CpiContext::new(cpi_program, cpi_accounts);

    token::transfer(cpi_ctx, amount)?;
    Ok(())
}
```

## Инструменты разработки

### Anchor Framework

```bash
# Установка
cargo install --git https://github.com/coral-xyz/anchor anchor-cli

# Создание проекта
anchor init my_project

# Структура проекта
my_project/
├── Anchor.toml          # Конфигурация
├── programs/            # Rust программы
│   └── my_project/
│       └── src/
│           └── lib.rs
├── tests/               # Тесты
└── app/                 # Frontend
```

### Solana CLI

```bash
# Установка
sh -c "$(curl -sSfL https://release.solana.com/stable/install)"

# Создание кошелька
solana-keygen new

# Проверка баланса
solana balance

# Airdrop на devnet
solana airdrop 2 --url devnet

# Деплой программы
solana program deploy target/deploy/my_program.so
```

### Тестирование

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { HelloSolana } from "../target/types/hello_solana";

describe("hello-solana", () => {
  anchor.setProvider(anchor.AnchorProvider.env());
  const program = anchor.workspace.HelloSolana as Program<HelloSolana>;

  it("Initializes account", async () => {
    const myAccount = anchor.web3.Keypair.generate();

    await program.methods
      .initialize(new anchor.BN(42))
      .accounts({
        myAccount: myAccount.publicKey,
        user: anchor.getProvider().wallet.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
      })
      .signers([myAccount])
      .rpc();

    const account = await program.account.myAccount.fetch(myAccount.publicKey);
    expect(account.data.toNumber()).to.equal(42);
  });
});
```

## Экосистема

### DeFi
- **Jupiter** - агрегатор DEX (ведущий на Solana)
- **Raydium** - AMM и DEX
- **Marinade** - ликвидный стейкинг
- **Drift** - перпетуал DEX
- **Kamino** - yield оптимизация

### NFT
- **Magic Eden** - крупнейший NFT маркетплейс
- **Tensor** - NFT маркетплейс для трейдеров
- **Metaplex** - стандарт NFT на Solana

### Инфраструктура
- **Phantom** - ведущий кошелек
- **Solflare** - альтернативный кошелек
- **Helius** - RPC провайдер и API
- **QuickNode** - RPC инфраструктура

## Сравнение моделей

### Account vs UTXO vs State

| Модель | Пример | Описание |
|--------|--------|----------|
| **UTXO** | Bitcoin | Неизрасходованные выходы, как "монеты" |
| **Account (EVM)** | Ethereum | Глобальное состояние, балансы |
| **Account (Solana)** | Solana | Разделение кода и данных |

```
Bitcoin UTXO:
[UTXO_1: 0.5 BTC] ──┐
                    ├──> [TX] ──> [UTXO_3: 0.8 BTC] (recipient)
[UTXO_2: 0.4 BTC] ──┘           [UTXO_4: 0.1 BTC] (change)

Ethereum Account:
Address A: { balance: 1.0 ETH, nonce: 5 }
Address B: { balance: 0.5 ETH, nonce: 3 }
TX: A → B (0.3 ETH)
Address A: { balance: 0.7 ETH, nonce: 6 }
Address B: { balance: 0.8 ETH, nonce: 3 }

Solana Account:
[Program] ←owns─ [Data Account A]
                 [Data Account B]
TX: Program.transfer(A, B, amount)
```

## Преимущества и недостатки

### Преимущества

- **Скорость** - ~400ms финализация, 65,000 TPS теоретически
- **Низкие комиссии** - ~$0.00025 за транзакцию
- **Параллельная обработка** - тысячи транзакций одновременно
- **Растущая экосистема** - DeFi, NFT, GameFi
- **Качественные инструменты** - Anchor, Solana CLI

### Недостатки

- **История простоев:**
  - Сентябрь 2021 - 17 часов (перегрузка)
  - Май 2022 - 7 часов (NFT минт бот)
  - Февраль 2023 - 20 часов (баг в runtime)
  - Февраль 2024 - 5 часов (баг в консенсусе)

- **Централизация** - высокие требования к валидаторам
- **Сложность разработки** - модель аккаунтов непривычна
- **Меньше разработчиков** - по сравнению с EVM экосистемой

## Производительность в реальности

```
Теоретический TPS:     65,000
Реальный TPS:          2,000-4,000 (обычно)
Пиковый TPS:           10,000+ (во время ажиотажа)

Комиссия за транзакцию: ~5,000 lamports (0.000005 SOL ≈ $0.0005)
Время блока:            ~400ms
Время финализации:      ~400ms (optimistic), ~12s (finalized)
```

## Ссылки

- [Solana Documentation](https://docs.solana.com/)
- [Anchor Framework](https://www.anchor-lang.com/)
- [Solana Cookbook](https://solanacookbook.com/)
- [Solana Stack Exchange](https://solana.stackexchange.com/)
- [Solana FM - Explorer](https://solana.fm/)
