# Next.js для dApps

## Введение

Next.js — мощный React-фреймворк, который предоставляет серверный рендеринг (SSR), статическую генерацию (SSG), маршрутизацию и множество оптимизаций из коробки. Для Web3-приложений Next.js особенно полезен благодаря SEO-оптимизации и гибкой архитектуре.

## Особенности Next.js для Web3

### Преимущества
- **SSR/SSG** — улучшенное SEO для публичных страниц dApp
- **API Routes** — серверные эндпоинты для off-chain операций
- **Middleware** — проверка авторизации и редиректы
- **Image Optimization** — оптимизация NFT изображений
- **App Router** — современная архитектура с Server Components

### Вызовы
- **Клиентские библиотеки** — wagmi и ethers работают только в браузере
- **window объект** — недоступен на сервере
- **Гидратация** — состояние кошелька должно синхронизироваться

## Настройка проекта

### Создание проекта

```bash
# Создание Next.js приложения с TypeScript
npx create-next-app@latest my-dapp --typescript --tailwind --app

cd my-dapp

# Установка Web3 зависимостей
npm install wagmi viem @tanstack/react-query
npm install @rainbow-me/rainbowkit
npm install @rainbow-me/rainbowkit-siwe-next-auth next-auth
```

### Структура проекта

```
my-dapp/
├── app/
│   ├── layout.tsx          # Корневой layout с провайдерами
│   ├── page.tsx            # Главная страница
│   ├── api/                # API Routes
│   │   └── [...]/route.ts
│   └── (routes)/           # Группы маршрутов
│       ├── dashboard/
│       └── mint/
├── components/
│   ├── providers/          # Web3 провайдеры
│   ├── wallet/             # Компоненты кошелька
│   └── ui/                 # UI компоненты
├── config/
│   └── wagmi.ts            # Конфигурация wagmi
├── hooks/                  # Кастомные хуки
├── lib/                    # Утилиты
└── contracts/              # ABI контрактов
```

## Конфигурация

### Wagmi конфигурация

```typescript
// config/wagmi.ts
import { http, createConfig, cookieStorage, createStorage } from 'wagmi';
import { mainnet, sepolia, polygon, arbitrum } from 'wagmi/chains';
import { injected, walletConnect, coinbaseWallet } from 'wagmi/connectors';

const projectId = process.env.NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID!;

export const config = createConfig({
  chains: [mainnet, sepolia, polygon, arbitrum],
  connectors: [
    injected(),
    walletConnect({ projectId }),
    coinbaseWallet({ appName: 'My dApp' }),
  ],
  transports: {
    [mainnet.id]: http(process.env.NEXT_PUBLIC_MAINNET_RPC),
    [sepolia.id]: http(process.env.NEXT_PUBLIC_SEPOLIA_RPC),
    [polygon.id]: http(process.env.NEXT_PUBLIC_POLYGON_RPC),
    [arbitrum.id]: http(process.env.NEXT_PUBLIC_ARBITRUM_RPC),
  },
  // Важно для SSR - сохранение состояния в cookies
  storage: createStorage({
    storage: cookieStorage,
  }),
  ssr: true,
});

// Типы для TypeScript
declare module 'wagmi' {
  interface Register {
    config: typeof config;
  }
}
```

### Переменные окружения

```bash
# .env.local
NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID=your_project_id

# RPC URLs (можно использовать Infura, Alchemy, или публичные)
NEXT_PUBLIC_MAINNET_RPC=https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY
NEXT_PUBLIC_SEPOLIA_RPC=https://eth-sepolia.g.alchemy.com/v2/YOUR_KEY
NEXT_PUBLIC_POLYGON_RPC=https://polygon-mainnet.g.alchemy.com/v2/YOUR_KEY

# Серверные переменные (не публичные)
NEXTAUTH_SECRET=your_nextauth_secret
NEXTAUTH_URL=http://localhost:3000
```

## App Router интеграция

### Провайдеры

```typescript
// components/providers/Web3Provider.tsx
'use client';

import { WagmiProvider } from 'wagmi';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { RainbowKitProvider, darkTheme } from '@rainbow-me/rainbowkit';
import { config } from '@/config/wagmi';
import { useState, type ReactNode } from 'react';

import '@rainbow-me/rainbowkit/styles.css';

interface Props {
  children: ReactNode;
  cookies: string | null;
}

export function Web3Provider({ children, cookies }: Props) {
  const [queryClient] = useState(
    () => new QueryClient({
      defaultOptions: {
        queries: {
          staleTime: 60 * 1000, // 1 минута
          gcTime: 5 * 60 * 1000, // 5 минут
        },
      },
    })
  );

  // Восстановление состояния из cookies для SSR
  const initialState = cookies
    ? (JSON.parse(cookies) as State)
    : undefined;

  return (
    <WagmiProvider config={config} initialState={initialState}>
      <QueryClientProvider client={queryClient}>
        <RainbowKitProvider
          theme={darkTheme({
            accentColor: '#7b3fe4',
            borderRadius: 'medium',
          })}
          locale="ru"
        >
          {children}
        </RainbowKitProvider>
      </QueryClientProvider>
    </WagmiProvider>
  );
}
```

### Корневой Layout

```typescript
// app/layout.tsx
import type { Metadata } from 'next';
import { Inter } from 'next/font/google';
import { headers, cookies } from 'next/headers';
import { Web3Provider } from '@/components/providers/Web3Provider';
import { Header } from '@/components/layout/Header';
import './globals.css';

const inter = Inter({ subsets: ['latin', 'cyrillic'] });

export const metadata: Metadata = {
  title: 'My dApp',
  description: 'Децентрализованное приложение на Next.js',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  // Получаем cookies для гидратации состояния wagmi
  const cookieStore = cookies();
  const wagmiCookie = cookieStore.get('wagmi.store')?.value ?? null;

  return (
    <html lang="ru">
      <body className={inter.className}>
        <Web3Provider cookies={wagmiCookie}>
          <Header />
          <main className="container mx-auto px-4 py-8">
            {children}
          </main>
        </Web3Provider>
      </body>
    </html>
  );
}
```

### Клиентские компоненты

```typescript
// components/wallet/ConnectWallet.tsx
'use client';

import { ConnectButton } from '@rainbow-me/rainbowkit';

export function ConnectWallet() {
  return (
    <ConnectButton
      accountStatus={{
        smallScreen: 'avatar',
        largeScreen: 'full',
      }}
      chainStatus="icon"
      showBalance={{
        smallScreen: false,
        largeScreen: true,
      }}
    />
  );
}
```

```typescript
// components/layout/Header.tsx
'use client';

import Link from 'next/link';
import { ConnectWallet } from '@/components/wallet/ConnectWallet';

export function Header() {
  return (
    <header className="border-b bg-white/50 backdrop-blur-sm sticky top-0 z-50">
      <div className="container mx-auto px-4 py-4 flex justify-between items-center">
        <nav className="flex gap-6">
          <Link href="/" className="font-bold text-xl">
            My dApp
          </Link>
          <Link href="/dashboard" className="hover:text-primary">
            Dashboard
          </Link>
          <Link href="/mint" className="hover:text-primary">
            Mint
          </Link>
        </nav>
        <ConnectWallet />
      </div>
    </header>
  );
}
```

## Server Components и Client Components

### Разделение ответственности

```typescript
// app/dashboard/page.tsx (Server Component)
import { Suspense } from 'react';
import { DashboardStats } from './DashboardStats';
import { WalletBalance } from './WalletBalance';
import { Loading } from '@/components/ui/Loading';

// Серверный компонент - может использовать async/await
export default async function DashboardPage() {
  // Можно получать данные на сервере
  const stats = await fetchPublicStats();

  return (
    <div className="space-y-8">
      <h1 className="text-3xl font-bold">Dashboard</h1>

      {/* Серверные данные */}
      <section>
        <h2>Статистика протокола</h2>
        <DashboardStats stats={stats} />
      </section>

      {/* Клиентский компонент с Web3 */}
      <Suspense fallback={<Loading />}>
        <WalletBalance />
      </Suspense>
    </div>
  );
}

async function fetchPublicStats() {
  const res = await fetch('https://api.example.com/stats', {
    next: { revalidate: 60 }, // ISR - обновление каждые 60 секунд
  });
  return res.json();
}
```

```typescript
// app/dashboard/WalletBalance.tsx
'use client';

import { useAccount, useBalance } from 'wagmi';
import { formatEther } from 'viem';

export function WalletBalance() {
  const { address, isConnected } = useAccount();
  const { data: balance, isLoading } = useBalance({ address });

  if (!isConnected) {
    return (
      <div className="p-6 bg-gray-100 rounded-lg">
        <p>Подключите кошелек для просмотра баланса</p>
      </div>
    );
  }

  if (isLoading) {
    return <div className="animate-pulse h-20 bg-gray-200 rounded-lg" />;
  }

  return (
    <div className="p-6 bg-gradient-to-r from-purple-500 to-blue-500 rounded-lg text-white">
      <h3 className="text-lg opacity-80">Ваш баланс</h3>
      <p className="text-3xl font-bold">
        {balance?.formatted} {balance?.symbol}
      </p>
      <p className="text-sm opacity-60 mt-2">
        {address}
      </p>
    </div>
  );
}
```

## API Routes

### Верификация подписи

```typescript
// app/api/verify/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { verifyMessage } from 'viem';

export async function POST(request: NextRequest) {
  try {
    const { address, message, signature } = await request.json();

    // Проверка подписи
    const isValid = await verifyMessage({
      address,
      message,
      signature,
    });

    if (!isValid) {
      return NextResponse.json(
        { error: 'Invalid signature' },
        { status: 401 }
      );
    }

    // Подпись валидна - можно выполнить действие
    // Например, создать сессию, записать в БД и т.д.

    return NextResponse.json({
      success: true,
      address
    });
  } catch (error) {
    return NextResponse.json(
      { error: 'Verification failed' },
      { status: 500 }
    );
  }
}
```

### Получение данных из блокчейна

```typescript
// app/api/token/[address]/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { createPublicClient, http, formatUnits } from 'viem';
import { mainnet } from 'viem/chains';

const client = createPublicClient({
  chain: mainnet,
  transport: http(process.env.MAINNET_RPC),
});

const erc20Abi = [
  {
    name: 'name',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ type: 'string' }],
  },
  {
    name: 'symbol',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ type: 'string' }],
  },
  {
    name: 'decimals',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ type: 'uint8' }],
  },
  {
    name: 'totalSupply',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ type: 'uint256' }],
  },
] as const;

export async function GET(
  request: NextRequest,
  { params }: { params: { address: string } }
) {
  try {
    const tokenAddress = params.address as `0x${string}`;

    const [name, symbol, decimals, totalSupply] = await Promise.all([
      client.readContract({
        address: tokenAddress,
        abi: erc20Abi,
        functionName: 'name',
      }),
      client.readContract({
        address: tokenAddress,
        abi: erc20Abi,
        functionName: 'symbol',
      }),
      client.readContract({
        address: tokenAddress,
        abi: erc20Abi,
        functionName: 'decimals',
      }),
      client.readContract({
        address: tokenAddress,
        abi: erc20Abi,
        functionName: 'totalSupply',
      }),
    ]);

    return NextResponse.json({
      address: tokenAddress,
      name,
      symbol,
      decimals,
      totalSupply: formatUnits(totalSupply, decimals),
    }, {
      headers: {
        'Cache-Control': 'public, s-maxage=60, stale-while-revalidate=300',
      },
    });
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to fetch token data' },
      { status: 500 }
    );
  }
}
```

## Аутентификация с SIWE

### Sign-In with Ethereum

```typescript
// lib/siwe.ts
import { SiweMessage } from 'siwe';

export function createSiweMessage(
  address: string,
  chainId: number,
  nonce: string
) {
  const message = new SiweMessage({
    domain: window.location.host,
    address,
    statement: 'Войти в My dApp',
    uri: window.location.origin,
    version: '1',
    chainId,
    nonce,
  });

  return message.prepareMessage();
}
```

```typescript
// app/api/auth/[...nextauth]/route.ts
import NextAuth from 'next-auth';
import CredentialsProvider from 'next-auth/providers/credentials';
import { SiweMessage } from 'siwe';

const handler = NextAuth({
  providers: [
    CredentialsProvider({
      name: 'Ethereum',
      credentials: {
        message: { type: 'text' },
        signature: { type: 'text' },
      },
      async authorize(credentials) {
        try {
          const siwe = new SiweMessage(
            JSON.parse(credentials?.message || '{}')
          );

          const result = await siwe.verify({
            signature: credentials?.signature || '',
          });

          if (result.success) {
            return {
              id: siwe.address,
              address: siwe.address,
            };
          }
          return null;
        } catch (e) {
          return null;
        }
      },
    }),
  ],
  session: {
    strategy: 'jwt',
  },
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.address = user.address;
      }
      return token;
    },
    async session({ session, token }) {
      session.address = token.address as string;
      return session;
    },
  },
});

export { handler as GET, handler as POST };
```

```typescript
// components/auth/SignInButton.tsx
'use client';

import { useAccount, useSignMessage } from 'wagmi';
import { signIn, signOut, useSession } from 'next-auth/react';
import { SiweMessage } from 'siwe';

export function SignInButton() {
  const { address, isConnected } = useAccount();
  const { signMessageAsync } = useSignMessage();
  const { data: session, status } = useSession();

  const handleSignIn = async () => {
    if (!address) return;

    try {
      // Получаем nonce с сервера
      const nonceRes = await fetch('/api/auth/nonce');
      const nonce = await nonceRes.text();

      // Создаем SIWE сообщение
      const message = new SiweMessage({
        domain: window.location.host,
        address,
        statement: 'Войти в My dApp',
        uri: window.location.origin,
        version: '1',
        chainId: 1,
        nonce,
      });

      const preparedMessage = message.prepareMessage();

      // Подписываем сообщение
      const signature = await signMessageAsync({
        message: preparedMessage,
      });

      // Авторизуемся через NextAuth
      await signIn('credentials', {
        message: JSON.stringify(message),
        signature,
        redirect: false,
      });
    } catch (error) {
      console.error('Sign in error:', error);
    }
  };

  if (status === 'loading') {
    return <button disabled>Загрузка...</button>;
  }

  if (session) {
    return (
      <div className="flex items-center gap-4">
        <span>Вошел как {session.address?.slice(0, 6)}...</span>
        <button onClick={() => signOut()}>Выйти</button>
      </div>
    );
  }

  if (!isConnected) {
    return <p>Сначала подключите кошелек</p>;
  }

  return (
    <button onClick={handleSignIn}>
      Войти с Ethereum
    </button>
  );
}
```

## Middleware

### Защита маршрутов

```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { getToken } from 'next-auth/jwt';

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // Защищенные маршруты
  const protectedPaths = ['/dashboard', '/profile', '/settings'];

  const isProtected = protectedPaths.some(path =>
    pathname.startsWith(path)
  );

  if (isProtected) {
    const token = await getToken({
      req: request,
      secret: process.env.NEXTAUTH_SECRET,
    });

    if (!token) {
      const loginUrl = new URL('/login', request.url);
      loginUrl.searchParams.set('callbackUrl', pathname);
      return NextResponse.redirect(loginUrl);
    }
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/dashboard/:path*', '/profile/:path*', '/settings/:path*'],
};
```

## SSR с блокчейн данными

### Получение данных на сервере

```typescript
// lib/viem-server.ts
import { createPublicClient, http } from 'viem';
import { mainnet } from 'viem/chains';

// Серверный клиент viem (без кошелька)
export const publicClient = createPublicClient({
  chain: mainnet,
  transport: http(process.env.MAINNET_RPC),
});
```

```typescript
// app/nft/[id]/page.tsx
import { publicClient } from '@/lib/viem-server';
import { NFTDetails } from './NFTDetails';
import type { Metadata } from 'next';

const nftAbi = [
  {
    name: 'tokenURI',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'tokenId', type: 'uint256' }],
    outputs: [{ type: 'string' }],
  },
  {
    name: 'ownerOf',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'tokenId', type: 'uint256' }],
    outputs: [{ type: 'address' }],
  },
] as const;

interface Props {
  params: { id: string };
}

// Генерация метаданных для SEO
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const nftData = await getNFTData(params.id);

  return {
    title: nftData?.name || `NFT #${params.id}`,
    description: nftData?.description,
    openGraph: {
      images: [nftData?.image || ''],
    },
  };
}

async function getNFTData(tokenId: string) {
  try {
    const contractAddress = process.env.NFT_CONTRACT_ADDRESS as `0x${string}`;

    const [tokenURI, owner] = await Promise.all([
      publicClient.readContract({
        address: contractAddress,
        abi: nftAbi,
        functionName: 'tokenURI',
        args: [BigInt(tokenId)],
      }),
      publicClient.readContract({
        address: contractAddress,
        abi: nftAbi,
        functionName: 'ownerOf',
        args: [BigInt(tokenId)],
      }),
    ]);

    // Загружаем метаданные
    const metadataRes = await fetch(tokenURI);
    const metadata = await metadataRes.json();

    return {
      ...metadata,
      owner,
      tokenId,
    };
  } catch (error) {
    console.error('Failed to fetch NFT:', error);
    return null;
  }
}

export default async function NFTPage({ params }: Props) {
  const nftData = await getNFTData(params.id);

  if (!nftData) {
    return <div>NFT не найден</div>;
  }

  return (
    <div className="max-w-4xl mx-auto">
      <div className="grid md:grid-cols-2 gap-8">
        <div>
          <img
            src={nftData.image}
            alt={nftData.name}
            className="rounded-lg w-full"
          />
        </div>
        <div>
          <h1 className="text-3xl font-bold">{nftData.name}</h1>
          <p className="text-gray-600 mt-2">{nftData.description}</p>
          <div className="mt-4">
            <p className="text-sm text-gray-500">Владелец</p>
            <p className="font-mono">{nftData.owner}</p>
          </div>
          {/* Клиентский компонент для интерактивных действий */}
          <NFTDetails tokenId={params.id} />
        </div>
      </div>
    </div>
  );
}
```

## Оптимизация

### Кэширование данных

```typescript
// lib/cache.ts
import { unstable_cache } from 'next/cache';
import { publicClient } from './viem-server';

export const getTokenPrice = unstable_cache(
  async (tokenAddress: string) => {
    // Получение цены из oracle или API
    const price = await fetchPrice(tokenAddress);
    return price;
  },
  ['token-price'],
  {
    revalidate: 60, // Обновление каждые 60 секунд
    tags: ['prices'],
  }
);

// Инвалидация кэша
import { revalidateTag } from 'next/cache';

export async function updatePrices() {
  revalidateTag('prices');
}
```

### Streaming и Suspense

```typescript
// app/portfolio/page.tsx
import { Suspense } from 'react';
import { TokenList } from './TokenList';
import { NFTGallery } from './NFTGallery';
import { ActivityFeed } from './ActivityFeed';

export default function PortfolioPage() {
  return (
    <div className="space-y-8">
      <h1>Портфолио</h1>

      {/* Каждый компонент загружается независимо */}
      <Suspense fallback={<TokenListSkeleton />}>
        <TokenList />
      </Suspense>

      <Suspense fallback={<NFTGallerySkeleton />}>
        <NFTGallery />
      </Suspense>

      <Suspense fallback={<ActivitySkeleton />}>
        <ActivityFeed />
      </Suspense>
    </div>
  );
}
```

## Деплой

### Vercel конфигурация

```json
// vercel.json
{
  "buildCommand": "next build",
  "devCommand": "next dev",
  "installCommand": "npm install",
  "framework": "nextjs",
  "regions": ["fra1"],
  "env": {
    "NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID": "@walletconnect-project-id",
    "MAINNET_RPC": "@mainnet-rpc",
    "NEXTAUTH_SECRET": "@nextauth-secret"
  }
}
```

### Переменные окружения на Vercel

```bash
# Production переменные
vercel env add NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID production
vercel env add MAINNET_RPC production
vercel env add NEXTAUTH_SECRET production
vercel env add NEXTAUTH_URL production

# Preview переменные (для PR previews)
vercel env add NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID preview
```

## Лучшие практики

### 1. Разделение серверного и клиентского кода

```typescript
// Плохо - смешивание серверного и клиентского кода
'use client';
import { useAccount } from 'wagmi';
import { db } from '@/lib/database'; // Ошибка - серверный код

// Хорошо - четкое разделение
// components/UserProfile.tsx (Client)
'use client';
import { useAccount } from 'wagmi';

export function UserProfile({ userData }: { userData: UserData }) {
  const { address } = useAccount();
  // Используем только клиентский код
}

// app/profile/page.tsx (Server)
import { db } from '@/lib/database';
import { UserProfile } from '@/components/UserProfile';

export default async function ProfilePage() {
  const userData = await db.user.findFirst();
  return <UserProfile userData={userData} />;
}
```

### 2. Обработка гидратации

```typescript
// hooks/useHydration.ts
'use client';

import { useState, useEffect } from 'react';

export function useHydration() {
  const [hydrated, setHydrated] = useState(false);

  useEffect(() => {
    setHydrated(true);
  }, []);

  return hydrated;
}

// Использование
function WalletAddress() {
  const { address } = useAccount();
  const hydrated = useHydration();

  // Избегаем несоответствия при гидратации
  if (!hydrated) {
    return <span className="animate-pulse bg-gray-200 w-32 h-4" />;
  }

  return <span>{address || 'Не подключен'}</span>;
}
```

### 3. Error Boundaries

```typescript
// app/dashboard/error.tsx
'use client';

export default function DashboardError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div className="text-center py-12">
      <h2 className="text-xl font-bold text-red-600">
        Что-то пошло не так
      </h2>
      <p className="text-gray-600 mt-2">
        {error.message}
      </p>
      <button
        onClick={reset}
        className="mt-4 px-4 py-2 bg-blue-500 text-white rounded"
      >
        Попробовать снова
      </button>
    </div>
  );
}
```

### 4. Loading состояния

```typescript
// app/dashboard/loading.tsx
export default function DashboardLoading() {
  return (
    <div className="space-y-4">
      <div className="h-8 bg-gray-200 rounded animate-pulse w-1/4" />
      <div className="grid grid-cols-3 gap-4">
        {[1, 2, 3].map((i) => (
          <div
            key={i}
            className="h-32 bg-gray-200 rounded animate-pulse"
          />
        ))}
      </div>
    </div>
  );
}
```

## Полный пример приложения

```typescript
// app/mint/page.tsx
import { MintForm } from './MintForm';
import { MintStats } from './MintStats';
import { publicClient } from '@/lib/viem-server';

// Серверный компонент - получаем статистику минта
async function getMintStats() {
  const totalSupply = await publicClient.readContract({
    address: process.env.NFT_CONTRACT as `0x${string}`,
    abi: nftAbi,
    functionName: 'totalSupply',
  });

  const maxSupply = await publicClient.readContract({
    address: process.env.NFT_CONTRACT as `0x${string}`,
    abi: nftAbi,
    functionName: 'maxSupply',
  });

  return {
    minted: Number(totalSupply),
    total: Number(maxSupply),
  };
}

export default async function MintPage() {
  const stats = await getMintStats();

  return (
    <div className="max-w-xl mx-auto">
      <h1 className="text-3xl font-bold mb-8">Mint NFT</h1>

      <MintStats minted={stats.minted} total={stats.total} />

      {/* Клиентский компонент для минта */}
      <MintForm maxSupply={stats.total} currentSupply={stats.minted} />
    </div>
  );
}
```

```typescript
// app/mint/MintForm.tsx
'use client';

import { useState } from 'react';
import { useAccount, useWriteContract, useWaitForTransactionReceipt } from 'wagmi';
import { parseEther } from 'viem';
import { ConnectButton } from '@rainbow-me/rainbowkit';

const nftAbi = [...] as const;
const contractAddress = process.env.NEXT_PUBLIC_NFT_CONTRACT as `0x${string}`;

interface Props {
  maxSupply: number;
  currentSupply: number;
}

export function MintForm({ maxSupply, currentSupply }: Props) {
  const [quantity, setQuantity] = useState(1);
  const { address, isConnected } = useAccount();

  const { data: hash, writeContract, isPending, error } = useWriteContract();
  const { isLoading: isConfirming, isSuccess } = useWaitForTransactionReceipt({ hash });

  const mintPrice = 0.05; // ETH
  const isSoldOut = currentSupply >= maxSupply;

  const handleMint = () => {
    writeContract({
      address: contractAddress,
      abi: nftAbi,
      functionName: 'mint',
      args: [BigInt(quantity)],
      value: parseEther((mintPrice * quantity).toString()),
    });
  };

  if (!isConnected) {
    return (
      <div className="text-center py-8">
        <p className="mb-4">Подключите кошелек для минта</p>
        <ConnectButton />
      </div>
    );
  }

  if (isSoldOut) {
    return (
      <div className="text-center py-8 bg-gray-100 rounded-lg">
        <p className="text-xl font-bold">Sold Out!</p>
      </div>
    );
  }

  return (
    <div className="space-y-6 bg-white p-6 rounded-lg shadow">
      <div>
        <label className="block text-sm font-medium mb-2">
          Количество
        </label>
        <div className="flex items-center gap-4">
          <button
            onClick={() => setQuantity(Math.max(1, quantity - 1))}
            className="px-4 py-2 bg-gray-200 rounded"
          >
            -
          </button>
          <span className="text-xl font-bold">{quantity}</span>
          <button
            onClick={() => setQuantity(Math.min(10, quantity + 1))}
            className="px-4 py-2 bg-gray-200 rounded"
          >
            +
          </button>
        </div>
      </div>

      <div className="border-t pt-4">
        <div className="flex justify-between">
          <span>Цена за 1 NFT</span>
          <span>{mintPrice} ETH</span>
        </div>
        <div className="flex justify-between font-bold text-lg mt-2">
          <span>Итого</span>
          <span>{(mintPrice * quantity).toFixed(3)} ETH</span>
        </div>
      </div>

      <button
        onClick={handleMint}
        disabled={isPending || isConfirming}
        className="w-full py-3 bg-gradient-to-r from-purple-500 to-blue-500
                   text-white font-bold rounded-lg disabled:opacity-50"
      >
        {isPending ? 'Подтвердите в кошельке...' :
         isConfirming ? 'Минтинг...' :
         `Mint ${quantity} NFT`}
      </button>

      {isSuccess && (
        <div className="p-4 bg-green-100 text-green-800 rounded-lg">
          <p className="font-bold">Успешно!</p>
          <a
            href={`https://etherscan.io/tx/${hash}`}
            target="_blank"
            className="underline"
          >
            Посмотреть транзакцию
          </a>
        </div>
      )}

      {error && (
        <div className="p-4 bg-red-100 text-red-800 rounded-lg">
          {error.message}
        </div>
      )}
    </div>
  );
}
```

## Дополнительные ресурсы

- [Next.js Documentation](https://nextjs.org/docs)
- [wagmi Documentation](https://wagmi.sh/)
- [RainbowKit with Next.js](https://www.rainbowkit.com/docs/installation)
- [NextAuth.js Documentation](https://next-auth.js.org/)
- [SIWE Documentation](https://docs.login.xyz/)
- [Vercel Deployment](https://vercel.com/docs)
