# Интеграция с фронтендом

## Введение

Интеграция фронтенда с блокчейном — это ключевой аспект разработки децентрализованных приложений (dApps). Современные Web3-приложения позволяют пользователям взаимодействовать со смарт-контрактами через привычный веб-интерфейс.

## Основные компоненты интеграции

### 1. Web3-провайдеры

Web3-провайдер — это мост между браузером и блокчейном. Он позволяет:
- Подключаться к блокчейн-сети
- Отправлять транзакции
- Читать данные из смарт-контрактов
- Подписывать сообщения

```typescript
// Пример получения провайдера через MetaMask
const provider = window.ethereum;

// Запрос подключения кошелька
const accounts = await provider.request({
  method: 'eth_requestAccounts'
});
```

### 2. Библиотеки для работы с блокчейном

#### ethers.js
Самая популярная библиотека для взаимодействия с Ethereum:

```typescript
import { ethers } from 'ethers';

// Создание провайдера
const provider = new ethers.BrowserProvider(window.ethereum);

// Получение signer для подписи транзакций
const signer = await provider.getSigner();

// Получение адреса пользователя
const address = await signer.getAddress();
```

#### viem
Современная, типобезопасная альтернатива ethers.js:

```typescript
import { createPublicClient, http } from 'viem';
import { mainnet } from 'viem/chains';

const client = createPublicClient({
  chain: mainnet,
  transport: http(),
});

// Получение баланса
const balance = await client.getBalance({
  address: '0x...'
});
```

#### wagmi
React-хуки для работы с Ethereum:

```typescript
import { useAccount, useBalance, useConnect } from 'wagmi';

function WalletInfo() {
  const { address, isConnected } = useAccount();
  const { data: balance } = useBalance({ address });

  return (
    <div>
      {isConnected ? (
        <p>Баланс: {balance?.formatted} ETH</p>
      ) : (
        <p>Кошелек не подключен</p>
      )}
    </div>
  );
}
```

## Подключение кошельков

### MetaMask

MetaMask — самый популярный браузерный кошелек:

```typescript
async function connectMetaMask() {
  if (typeof window.ethereum !== 'undefined') {
    try {
      const accounts = await window.ethereum.request({
        method: 'eth_requestAccounts',
      });
      return accounts[0];
    } catch (error) {
      console.error('Пользователь отклонил подключение');
    }
  } else {
    console.error('MetaMask не установлен');
  }
}
```

### WalletConnect

WalletConnect позволяет подключаться к мобильным кошелькам:

```typescript
import { WalletConnectConnector } from '@wagmi/connectors/walletConnect';

const connector = new WalletConnectConnector({
  options: {
    projectId: 'your-project-id',
    showQrModal: true,
  },
});
```

### RainbowKit

RainbowKit предоставляет готовый UI для подключения кошельков:

```typescript
import { ConnectButton } from '@rainbow-me/rainbowkit';

function App() {
  return (
    <div>
      <ConnectButton />
    </div>
  );
}
```

## Взаимодействие со смарт-контрактами

### Чтение данных (Read)

```typescript
import { ethers } from 'ethers';

const contractABI = [
  'function balanceOf(address) view returns (uint256)',
  'function name() view returns (string)',
];

const contract = new ethers.Contract(
  contractAddress,
  contractABI,
  provider
);

// Чтение данных не требует gas
const balance = await contract.balanceOf(userAddress);
const name = await contract.name();
```

### Запись данных (Write)

```typescript
// Для записи нужен signer
const contractWithSigner = contract.connect(signer);

// Отправка транзакции
const tx = await contractWithSigner.transfer(
  recipientAddress,
  ethers.parseEther('1.0')
);

// Ожидание подтверждения
const receipt = await tx.wait();
console.log('Транзакция подтверждена:', receipt.hash);
```

## Обработка событий

```typescript
// Подписка на события контракта
contract.on('Transfer', (from, to, amount, event) => {
  console.log(`Transfer: ${from} -> ${to}: ${amount}`);
});

// Получение исторических событий
const filter = contract.filters.Transfer(userAddress);
const events = await contract.queryFilter(filter);
```

## Архитектура dApp

```
┌─────────────────────────────────────────────────────────────┐
│                      Frontend (React/Next.js)                │
├─────────────────────────────────────────────────────────────┤
│  UI Components  │  State Management  │  Web3 Hooks (wagmi)  │
├─────────────────────────────────────────────────────────────┤
│                   Web3 Provider (ethers/viem)                │
├─────────────────────────────────────────────────────────────┤
│              Wallet (MetaMask/WalletConnect)                 │
├─────────────────────────────────────────────────────────────┤
│                  RPC Node (Infura/Alchemy)                   │
├─────────────────────────────────────────────────────────────┤
│                    Blockchain (Ethereum)                     │
└─────────────────────────────────────────────────────────────┘
```

## Лучшие практики

### 1. Обработка ошибок

```typescript
try {
  const tx = await contract.someFunction();
  await tx.wait();
} catch (error: any) {
  if (error.code === 'ACTION_REJECTED') {
    console.log('Пользователь отклонил транзакцию');
  } else if (error.code === 'INSUFFICIENT_FUNDS') {
    console.log('Недостаточно средств для gas');
  } else {
    console.error('Неизвестная ошибка:', error);
  }
}
```

### 2. Оптимизация производительности

- Используйте кэширование для read-запросов
- Batch-запросы через Multicall
- Подписка на события вместо polling

### 3. Безопасность

- Всегда проверяйте сеть (chainId)
- Показывайте пользователю детали транзакции
- Не храните приватные ключи на клиенте
- Используйте проверенные библиотеки

### 4. UX

- Показывайте состояние транзакции
- Предоставляйте ссылки на block explorer
- Обрабатывайте отключение кошелька
- Поддерживайте несколько сетей

## Инструменты разработки

| Инструмент | Назначение |
|------------|------------|
| ethers.js | Базовая библиотека для Web3 |
| viem | Типобезопасная альтернатива |
| wagmi | React-хуки для Web3 |
| RainbowKit | UI для подключения кошельков |
| ConnectKit | Альтернативный UI |
| Hardhat | Локальная разработка |
| Foundry | Быстрое тестирование |

## Структура раздела

- [React для dApps](./01-react/readme.md) - интеграция с React
- [Next.js для dApps](./02-nextjs/readme.md) - интеграция с Next.js

## Дополнительные ресурсы

- [ethers.js Documentation](https://docs.ethers.org/)
- [viem Documentation](https://viem.sh/)
- [wagmi Documentation](https://wagmi.sh/)
- [RainbowKit Documentation](https://www.rainbowkit.com/)
