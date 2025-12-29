# React для dApps

## Введение

React — самый популярный фреймворк для создания пользовательских интерфейсов децентрализованных приложений. В сочетании с библиотеками wagmi и viem, React предоставляет мощный инструментарий для Web3-разработки.

## Настройка проекта

### Создание проекта

```bash
# Создание нового React-приложения с Vite
npm create vite@latest my-dapp -- --template react-ts

cd my-dapp

# Установка Web3 зависимостей
npm install wagmi viem @tanstack/react-query
npm install @rainbow-me/rainbowkit
```

### Базовая конфигурация

```typescript
// src/config/wagmi.ts
import { http, createConfig } from 'wagmi';
import { mainnet, sepolia, polygon } from 'wagmi/chains';
import { injected, walletConnect } from 'wagmi/connectors';

const projectId = 'YOUR_WALLETCONNECT_PROJECT_ID';

export const config = createConfig({
  chains: [mainnet, sepolia, polygon],
  connectors: [
    injected(),
    walletConnect({ projectId }),
  ],
  transports: {
    [mainnet.id]: http(),
    [sepolia.id]: http(),
    [polygon.id]: http(),
  },
});
```

### Настройка провайдеров

```typescript
// src/main.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { WagmiProvider } from 'wagmi';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { RainbowKitProvider } from '@rainbow-me/rainbowkit';
import { config } from './config/wagmi';
import App from './App';

import '@rainbow-me/rainbowkit/styles.css';

const queryClient = new QueryClient();

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <WagmiProvider config={config}>
      <QueryClientProvider client={queryClient}>
        <RainbowKitProvider>
          <App />
        </RainbowKitProvider>
      </QueryClientProvider>
    </WagmiProvider>
  </React.StrictMode>
);
```

## Подключение кошелька

### С использованием RainbowKit

```typescript
// src/components/ConnectWallet.tsx
import { ConnectButton } from '@rainbow-me/rainbowkit';

export function ConnectWallet() {
  return (
    <ConnectButton
      accountStatus="avatar"
      chainStatus="icon"
      showBalance={true}
    />
  );
}
```

### Кастомная кнопка подключения

```typescript
// src/components/CustomConnectButton.tsx
import { ConnectButton } from '@rainbow-me/rainbowkit';

export function CustomConnectButton() {
  return (
    <ConnectButton.Custom>
      {({
        account,
        chain,
        openAccountModal,
        openChainModal,
        openConnectModal,
        mounted,
      }) => {
        const connected = mounted && account && chain;

        return (
          <div>
            {!connected ? (
              <button onClick={openConnectModal}>
                Подключить кошелек
              </button>
            ) : chain.unsupported ? (
              <button onClick={openChainModal}>
                Неподдерживаемая сеть
              </button>
            ) : (
              <div style={{ display: 'flex', gap: 12 }}>
                <button onClick={openChainModal}>
                  {chain.name}
                </button>
                <button onClick={openAccountModal}>
                  {account.displayName}
                  {account.displayBalance && ` (${account.displayBalance})`}
                </button>
              </div>
            )}
          </div>
        );
      }}
    </ConnectButton.Custom>
  );
}
```

### Подключение через wagmi hooks

```typescript
// src/components/WagmiConnect.tsx
import { useConnect, useAccount, useDisconnect } from 'wagmi';
import { injected } from 'wagmi/connectors';

export function WagmiConnect() {
  const { address, isConnected } = useAccount();
  const { connect, isPending } = useConnect();
  const { disconnect } = useDisconnect();

  if (isConnected) {
    return (
      <div>
        <p>Подключен: {address}</p>
        <button onClick={() => disconnect()}>
          Отключить
        </button>
      </div>
    );
  }

  return (
    <button
      onClick={() => connect({ connector: injected() })}
      disabled={isPending}
    >
      {isPending ? 'Подключение...' : 'Подключить MetaMask'}
    </button>
  );
}
```

## Хуки wagmi

### useAccount - информация об аккаунте

```typescript
import { useAccount } from 'wagmi';

function AccountInfo() {
  const {
    address,           // Адрес кошелька
    isConnected,       // Подключен ли кошелек
    isConnecting,      // В процессе подключения
    isDisconnected,    // Отключен
    connector,         // Текущий коннектор
    chain,             // Текущая сеть
  } = useAccount();

  return (
    <div>
      {isConnected && (
        <>
          <p>Адрес: {address}</p>
          <p>Сеть: {chain?.name}</p>
          <p>Коннектор: {connector?.name}</p>
        </>
      )}
    </div>
  );
}
```

### useBalance - баланс кошелька

```typescript
import { useBalance } from 'wagmi';

function Balance() {
  const { address } = useAccount();

  // Баланс нативного токена (ETH)
  const { data: ethBalance, isLoading } = useBalance({
    address,
  });

  // Баланс ERC20 токена
  const { data: tokenBalance } = useBalance({
    address,
    token: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48', // USDC
  });

  if (isLoading) return <p>Загрузка...</p>;

  return (
    <div>
      <p>ETH: {ethBalance?.formatted} {ethBalance?.symbol}</p>
      <p>USDC: {tokenBalance?.formatted} {tokenBalance?.symbol}</p>
    </div>
  );
}
```

### useChainId и useSwitchChain - работа с сетями

```typescript
import { useChainId, useSwitchChain } from 'wagmi';
import { mainnet, polygon, sepolia } from 'wagmi/chains';

function NetworkSwitcher() {
  const chainId = useChainId();
  const { switchChain, isPending } = useSwitchChain();

  const chains = [mainnet, polygon, sepolia];

  return (
    <div>
      <p>Текущая сеть: {chainId}</p>
      <div>
        {chains.map((chain) => (
          <button
            key={chain.id}
            onClick={() => switchChain({ chainId: chain.id })}
            disabled={isPending || chainId === chain.id}
          >
            {chain.name}
          </button>
        ))}
      </div>
    </div>
  );
}
```

## Чтение данных из смарт-контрактов

### useReadContract - одиночный вызов

```typescript
import { useReadContract } from 'wagmi';

// ABI контракта (можно импортировать из JSON)
const erc20Abi = [
  {
    name: 'balanceOf',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'account', type: 'address' }],
    outputs: [{ type: 'uint256' }],
  },
  {
    name: 'totalSupply',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ type: 'uint256' }],
  },
] as const;

function TokenBalance() {
  const { address } = useAccount();

  const { data, isLoading, error, refetch } = useReadContract({
    address: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48',
    abi: erc20Abi,
    functionName: 'balanceOf',
    args: [address!],
    query: {
      enabled: !!address, // Запрос только если есть адрес
    },
  });

  if (isLoading) return <p>Загрузка...</p>;
  if (error) return <p>Ошибка: {error.message}</p>;

  return (
    <div>
      <p>Баланс: {data?.toString()}</p>
      <button onClick={() => refetch()}>Обновить</button>
    </div>
  );
}
```

### useReadContracts - множественные вызовы (Multicall)

```typescript
import { useReadContracts } from 'wagmi';

function MultipleReads() {
  const tokenAddress = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48';

  const { data, isLoading } = useReadContracts({
    contracts: [
      {
        address: tokenAddress,
        abi: erc20Abi,
        functionName: 'name',
      },
      {
        address: tokenAddress,
        abi: erc20Abi,
        functionName: 'symbol',
      },
      {
        address: tokenAddress,
        abi: erc20Abi,
        functionName: 'totalSupply',
      },
    ],
  });

  if (isLoading) return <p>Загрузка...</p>;

  const [name, symbol, totalSupply] = data || [];

  return (
    <div>
      <p>Название: {name?.result as string}</p>
      <p>Символ: {symbol?.result as string}</p>
      <p>Общий запас: {totalSupply?.result?.toString()}</p>
    </div>
  );
}
```

## Запись в смарт-контракты

### useWriteContract - отправка транзакций

```typescript
import { useWriteContract, useWaitForTransactionReceipt } from 'wagmi';
import { parseEther, parseUnits } from 'viem';

const erc20WriteAbi = [
  {
    name: 'transfer',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'to', type: 'address' },
      { name: 'amount', type: 'uint256' },
    ],
    outputs: [{ type: 'bool' }],
  },
  {
    name: 'approve',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'spender', type: 'address' },
      { name: 'amount', type: 'uint256' },
    ],
    outputs: [{ type: 'bool' }],
  },
] as const;

function TransferToken() {
  const [recipient, setRecipient] = useState('');
  const [amount, setAmount] = useState('');

  const {
    data: hash,
    writeContract,
    isPending,
    error
  } = useWriteContract();

  // Ожидание подтверждения транзакции
  const { isLoading: isConfirming, isSuccess } = useWaitForTransactionReceipt({
    hash,
  });

  const handleTransfer = () => {
    writeContract({
      address: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48',
      abi: erc20WriteAbi,
      functionName: 'transfer',
      args: [recipient as `0x${string}`, parseUnits(amount, 6)],
    });
  };

  return (
    <div>
      <input
        placeholder="Адрес получателя"
        value={recipient}
        onChange={(e) => setRecipient(e.target.value)}
      />
      <input
        placeholder="Сумма"
        value={amount}
        onChange={(e) => setAmount(e.target.value)}
      />
      <button
        onClick={handleTransfer}
        disabled={isPending || isConfirming}
      >
        {isPending ? 'Подтвердите в кошельке...' :
         isConfirming ? 'Ожидание подтверждения...' :
         'Отправить'}
      </button>

      {isSuccess && (
        <p>
          Транзакция успешна!{' '}
          <a href={`https://etherscan.io/tx/${hash}`} target="_blank">
            Посмотреть на Etherscan
          </a>
        </p>
      )}

      {error && <p>Ошибка: {error.message}</p>}
    </div>
  );
}
```

### useSendTransaction - отправка ETH

```typescript
import { useSendTransaction, useWaitForTransactionReceipt } from 'wagmi';
import { parseEther } from 'viem';

function SendEth() {
  const [to, setTo] = useState('');
  const [amount, setAmount] = useState('');

  const { data: hash, sendTransaction, isPending } = useSendTransaction();

  const { isLoading: isConfirming, isSuccess } = useWaitForTransactionReceipt({
    hash,
  });

  const handleSend = () => {
    sendTransaction({
      to: to as `0x${string}`,
      value: parseEther(amount),
    });
  };

  return (
    <div>
      <input
        placeholder="Адрес получателя"
        value={to}
        onChange={(e) => setTo(e.target.value)}
      />
      <input
        placeholder="Сумма ETH"
        value={amount}
        onChange={(e) => setAmount(e.target.value)}
      />
      <button onClick={handleSend} disabled={isPending || isConfirming}>
        {isPending ? 'Подтвердите...' :
         isConfirming ? 'Отправка...' :
         'Отправить ETH'}
      </button>
      {isSuccess && <p>Транзакция: {hash}</p>}
    </div>
  );
}
```

## Подписание сообщений

### useSignMessage

```typescript
import { useSignMessage, useVerifyMessage } from 'wagmi';

function SignMessage() {
  const [message, setMessage] = useState('');

  const {
    data: signature,
    signMessage,
    isPending,
    error
  } = useSignMessage();

  const { data: isValid } = useVerifyMessage({
    address: useAccount().address,
    message,
    signature,
  });

  return (
    <div>
      <textarea
        placeholder="Введите сообщение для подписи"
        value={message}
        onChange={(e) => setMessage(e.target.value)}
      />
      <button
        onClick={() => signMessage({ message })}
        disabled={isPending || !message}
      >
        {isPending ? 'Подписание...' : 'Подписать'}
      </button>

      {signature && (
        <div>
          <p>Подпись: {signature}</p>
          <p>Валидна: {isValid ? 'Да' : 'Нет'}</p>
        </div>
      )}

      {error && <p>Ошибка: {error.message}</p>}
    </div>
  );
}
```

## Отслеживание событий

### useWatchContractEvent

```typescript
import { useWatchContractEvent } from 'wagmi';

const transferEventAbi = [
  {
    name: 'Transfer',
    type: 'event',
    inputs: [
      { indexed: true, name: 'from', type: 'address' },
      { indexed: true, name: 'to', type: 'address' },
      { indexed: false, name: 'value', type: 'uint256' },
    ],
  },
] as const;

function TransferWatcher() {
  const [transfers, setTransfers] = useState<any[]>([]);

  useWatchContractEvent({
    address: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48',
    abi: transferEventAbi,
    eventName: 'Transfer',
    onLogs: (logs) => {
      setTransfers((prev) => [...prev, ...logs]);
    },
  });

  return (
    <div>
      <h3>Последние переводы:</h3>
      <ul>
        {transfers.slice(-10).map((log, i) => (
          <li key={i}>
            {log.args.from} → {log.args.to}: {log.args.value?.toString()}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

## Работа с ethers.js в React

### Использование ethers.js напрямую

```typescript
import { useEffect, useState } from 'react';
import { ethers, BrowserProvider, Contract } from 'ethers';

function useEthers() {
  const [provider, setProvider] = useState<BrowserProvider | null>(null);
  const [signer, setSigner] = useState<ethers.Signer | null>(null);
  const [address, setAddress] = useState<string>('');

  useEffect(() => {
    const init = async () => {
      if (window.ethereum) {
        const browserProvider = new BrowserProvider(window.ethereum);
        setProvider(browserProvider);

        const accounts = await window.ethereum.request({
          method: 'eth_accounts'
        });

        if (accounts.length > 0) {
          const signer = await browserProvider.getSigner();
          setSigner(signer);
          setAddress(await signer.getAddress());
        }
      }
    };
    init();
  }, []);

  const connect = async () => {
    if (provider) {
      await window.ethereum.request({ method: 'eth_requestAccounts' });
      const signer = await provider.getSigner();
      setSigner(signer);
      setAddress(await signer.getAddress());
    }
  };

  return { provider, signer, address, connect };
}

// Использование
function EthersComponent() {
  const { provider, signer, address, connect } = useEthers();
  const [balance, setBalance] = useState<string>('');

  useEffect(() => {
    const fetchBalance = async () => {
      if (provider && address) {
        const balance = await provider.getBalance(address);
        setBalance(ethers.formatEther(balance));
      }
    };
    fetchBalance();
  }, [provider, address]);

  return (
    <div>
      {address ? (
        <>
          <p>Адрес: {address}</p>
          <p>Баланс: {balance} ETH</p>
        </>
      ) : (
        <button onClick={connect}>Подключить</button>
      )}
    </div>
  );
}
```

## Кастомные хуки

### useContract - переиспользуемый хук для контрактов

```typescript
// src/hooks/useContract.ts
import { useMemo } from 'react';
import { usePublicClient, useWalletClient } from 'wagmi';
import { getContract } from 'viem';

export function useContract<TAbi extends readonly unknown[]>(
  address: `0x${string}`,
  abi: TAbi
) {
  const publicClient = usePublicClient();
  const { data: walletClient } = useWalletClient();

  const contract = useMemo(() => {
    if (!publicClient) return null;

    return getContract({
      address,
      abi,
      client: {
        public: publicClient,
        wallet: walletClient ?? undefined,
      },
    });
  }, [address, abi, publicClient, walletClient]);

  return contract;
}

// Использование
function MyComponent() {
  const contract = useContract(tokenAddress, erc20Abi);

  const fetchData = async () => {
    if (contract) {
      const balance = await contract.read.balanceOf([userAddress]);
      console.log(balance);
    }
  };
}
```

### useTokenData - получение информации о токене

```typescript
// src/hooks/useTokenData.ts
import { useReadContracts } from 'wagmi';
import { formatUnits } from 'viem';

const tokenAbi = [
  { name: 'name', type: 'function', stateMutability: 'view', inputs: [], outputs: [{ type: 'string' }] },
  { name: 'symbol', type: 'function', stateMutability: 'view', inputs: [], outputs: [{ type: 'string' }] },
  { name: 'decimals', type: 'function', stateMutability: 'view', inputs: [], outputs: [{ type: 'uint8' }] },
  { name: 'totalSupply', type: 'function', stateMutability: 'view', inputs: [], outputs: [{ type: 'uint256' }] },
] as const;

export function useTokenData(address: `0x${string}`) {
  const { data, isLoading, error } = useReadContracts({
    contracts: [
      { address, abi: tokenAbi, functionName: 'name' },
      { address, abi: tokenAbi, functionName: 'symbol' },
      { address, abi: tokenAbi, functionName: 'decimals' },
      { address, abi: tokenAbi, functionName: 'totalSupply' },
    ],
  });

  const [name, symbol, decimals, totalSupply] = data || [];

  return {
    name: name?.result as string | undefined,
    symbol: symbol?.result as string | undefined,
    decimals: decimals?.result as number | undefined,
    totalSupply: totalSupply?.result && decimals?.result
      ? formatUnits(totalSupply.result as bigint, decimals.result as number)
      : undefined,
    isLoading,
    error,
  };
}
```

## Лучшие практики

### 1. Типизация контрактов

```typescript
// Генерация типов из ABI с помощью wagmi CLI
// wagmi.config.ts
import { defineConfig } from '@wagmi/cli';
import { react } from '@wagmi/cli/plugins';

export default defineConfig({
  out: 'src/generated.ts',
  contracts: [
    {
      name: 'ERC20',
      abi: erc20Abi,
    },
  ],
  plugins: [react()],
});
```

### 2. Обработка ошибок

```typescript
function TransactionForm() {
  const { writeContract, error } = useWriteContract();

  const getErrorMessage = (error: Error | null) => {
    if (!error) return null;

    const message = error.message.toLowerCase();

    if (message.includes('user rejected')) {
      return 'Транзакция отклонена пользователем';
    }
    if (message.includes('insufficient funds')) {
      return 'Недостаточно средств для оплаты gas';
    }
    if (message.includes('nonce too low')) {
      return 'Ошибка nonce. Попробуйте снова';
    }

    return 'Произошла ошибка. Попробуйте позже';
  };

  return (
    <div>
      {error && (
        <div className="error">
          {getErrorMessage(error)}
        </div>
      )}
    </div>
  );
}
```

### 3. Оптимизация запросов

```typescript
import { useReadContract } from 'wagmi';

function OptimizedComponent() {
  const { address } = useAccount();

  const { data } = useReadContract({
    address: contractAddress,
    abi: contractAbi,
    functionName: 'balanceOf',
    args: [address!],
    query: {
      enabled: !!address,
      staleTime: 30_000, // Данные считаются свежими 30 секунд
      gcTime: 5 * 60_000, // Кэш хранится 5 минут
      refetchInterval: 60_000, // Автообновление каждую минуту
    },
  });
}
```

### 4. Состояние загрузки

```typescript
function LoadingStates() {
  const { isConnecting, isReconnecting } = useAccount();
  const { data, isLoading, isFetching } = useReadContract({...});
  const { isPending, isConfirming } = useWriteContract();

  if (isConnecting || isReconnecting) {
    return <Spinner text="Подключение к кошельку..." />;
  }

  if (isLoading) {
    return <Skeleton />;
  }

  if (isPending) {
    return <Modal title="Подтвердите транзакцию в кошельке" />;
  }

  if (isConfirming) {
    return <Modal title="Ожидание подтверждения в сети..." />;
  }

  return <div>{/* Основной контент */}</div>;
}
```

## Полный пример dApp

```typescript
// src/App.tsx
import { useState } from 'react';
import { useAccount, useBalance, useReadContract, useWriteContract, useWaitForTransactionReceipt } from 'wagmi';
import { ConnectButton } from '@rainbow-me/rainbowkit';
import { parseUnits, formatUnits } from 'viem';

const tokenAddress = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48';
const tokenAbi = [...] as const;

export default function App() {
  const { address, isConnected } = useAccount();
  const [recipient, setRecipient] = useState('');
  const [amount, setAmount] = useState('');

  // Баланс ETH
  const { data: ethBalance } = useBalance({ address });

  // Баланс токена
  const { data: tokenBalance, refetch } = useReadContract({
    address: tokenAddress,
    abi: tokenAbi,
    functionName: 'balanceOf',
    args: [address!],
    query: { enabled: !!address },
  });

  // Отправка токенов
  const { data: hash, writeContract, isPending, error } = useWriteContract();
  const { isLoading: isConfirming, isSuccess } = useWaitForTransactionReceipt({ hash });

  const handleTransfer = () => {
    writeContract({
      address: tokenAddress,
      abi: tokenAbi,
      functionName: 'transfer',
      args: [recipient as `0x${string}`, parseUnits(amount, 6)],
    });
  };

  // Обновление баланса после успешной транзакции
  useEffect(() => {
    if (isSuccess) refetch();
  }, [isSuccess, refetch]);

  return (
    <div className="app">
      <header>
        <h1>My dApp</h1>
        <ConnectButton />
      </header>

      {isConnected && (
        <main>
          <section className="balances">
            <h2>Балансы</h2>
            <p>ETH: {ethBalance?.formatted}</p>
            <p>USDC: {tokenBalance ? formatUnits(tokenBalance as bigint, 6) : '0'}</p>
          </section>

          <section className="transfer">
            <h2>Перевод USDC</h2>
            <input
              placeholder="Адрес получателя"
              value={recipient}
              onChange={(e) => setRecipient(e.target.value)}
            />
            <input
              placeholder="Сумма"
              value={amount}
              onChange={(e) => setAmount(e.target.value)}
            />
            <button
              onClick={handleTransfer}
              disabled={isPending || isConfirming || !recipient || !amount}
            >
              {isPending ? 'Подтвердите...' :
               isConfirming ? 'Отправка...' :
               'Отправить'}
            </button>

            {isSuccess && (
              <p className="success">
                Успешно! <a href={`https://etherscan.io/tx/${hash}`}>Tx</a>
              </p>
            )}
            {error && <p className="error">{error.message}</p>}
          </section>
        </main>
      )}
    </div>
  );
}
```

## Дополнительные ресурсы

- [wagmi Documentation](https://wagmi.sh/)
- [viem Documentation](https://viem.sh/)
- [RainbowKit Documentation](https://www.rainbowkit.com/)
- [React Query Documentation](https://tanstack.com/query)
- [ethers.js Documentation](https://docs.ethers.org/)
