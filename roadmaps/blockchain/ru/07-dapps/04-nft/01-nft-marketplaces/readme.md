# NFT Маркетплейсы

## Обзор NFT маркетплейсов

NFT маркетплейсы — это платформы для покупки, продажи и обмена невзаимозаменяемых токенов. Они предоставляют инфраструктуру для листинга NFT, проведения аукционов и обработки транзакций.

## Основные NFT маркетплейсы

### OpenSea

**OpenSea** — крупнейший NFT маркетплейс, поддерживающий множество блокчейнов.

| Характеристика | Описание |
|----------------|----------|
| **Блокчейны** | Ethereum, Polygon, Arbitrum, Optimism, Base, Avalanche, Klaytn, BNB |
| **Комиссия** | 2.5% |
| **Протокол** | Seaport |
| **Особенности** | Lazy minting, коллекции, аукционы |

**Возможности OpenSea:**
- Создание коллекций без написания кода
- Lazy minting (минтинг при покупке)
- Аукционы и фиксированные цены
- Поддержка роялти
- Верификация коллекций

### Blur

**Blur** — профессиональный маркетплейс для трейдеров, ориентированный на скорость и низкие комиссии.

| Характеристика | Описание |
|----------------|----------|
| **Блокчейны** | Ethereum, Blast |
| **Комиссия** | 0% (до февраля 2024 было 0%, сейчас 0.5%) |
| **Особенности** | Aggregator, bid pools, analytics |

**Ключевые особенности Blur:**
- Агрегация листингов с других маркетплейсов
- Floor sweep (массовая покупка NFT по floor price)
- Bid pools для автоматических покупок
- Продвинутая аналитика
- Токен BLUR для стимулирования

### Rarible

**Rarible** — децентрализованный маркетплейс с токеном управления RARI.

| Характеристика | Описание |
|----------------|----------|
| **Блокчейны** | Ethereum, Polygon, Tezos, Flow, Immutable X |
| **Комиссия** | 2.5% |
| **Протокол** | Rarible Protocol |
| **Особенности** | DAO, multi-chain, protocol для разработчиков |

### Другие важные маркетплейсы

| Маркетплейс | Специализация | Блокчейн |
|-------------|---------------|----------|
| **Magic Eden** | Solana NFT, gaming | Solana, Ethereum, Polygon, Bitcoin |
| **LooksRare** | Ethereum NFT | Ethereum |
| **X2Y2** | Ethereum NFT | Ethereum |
| **Foundation** | Цифровое искусство | Ethereum |
| **SuperRare** | Премиум искусство | Ethereum |
| **Nifty Gateway** | Кураторские дропы | Ethereum |

## Seaport Protocol

**Seaport** — открытый протокол от OpenSea для децентрализованной торговли NFT.

### Архитектура Seaport

```
┌─────────────────────────────────────────────────────────────┐
│                      Seaport Protocol                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │   Orders    │    │   Fulfill   │    │   Cancel    │     │
│  │  (Listings) │    │   Orders    │    │   Orders    │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Conduit System                          │   │
│  │  (Управление разрешениями на передачу токенов)      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Zone System                             │   │
│  │  (Дополнительная валидация и ограничения)           │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Структура Order в Seaport

```solidity
struct OrderParameters {
    address offerer;                    // Создатель ордера
    address zone;                       // Адрес zone для валидации
    OfferItem[] offer;                  // Что предлагается
    ConsiderationItem[] consideration;  // Что требуется взамен
    OrderType orderType;                // Тип ордера
    uint256 startTime;                  // Время начала
    uint256 endTime;                    // Время окончания
    bytes32 zoneHash;                   // Данные для zone
    uint256 salt;                       // Уникальный salt
    bytes32 conduitKey;                 // Ключ conduit
    uint256 totalOriginalConsiderationItems;
}

struct OfferItem {
    ItemType itemType;      // 0: ETH, 1: ERC20, 2: ERC721, 3: ERC1155
    address token;          // Адрес токена
    uint256 identifierOrCriteria;
    uint256 startAmount;
    uint256 endAmount;
}

struct ConsiderationItem {
    ItemType itemType;
    address token;
    uint256 identifierOrCriteria;
    uint256 startAmount;
    uint256 endAmount;
    address payable recipient;  // Получатель
}
```

### Интеграция с Seaport

```typescript
import { Seaport } from "@opensea/seaport-js";
import { ethers } from "ethers";

// Инициализация Seaport
const provider = new ethers.providers.Web3Provider(window.ethereum);
const seaport = new Seaport(provider);

// Создание листинга (продажа NFT)
async function createListing(
    nftAddress: string,
    tokenId: string,
    priceInEth: string
) {
    const { executeAllActions } = await seaport.createOrder({
        offer: [
            {
                itemType: 2, // ERC721
                token: nftAddress,
                identifier: tokenId,
            },
        ],
        consideration: [
            {
                amount: ethers.utils.parseEther(priceInEth).toString(),
                recipient: await provider.getSigner().getAddress(),
            },
        ],
    });

    const order = await executeAllActions();
    console.log("Order created:", order);
    return order;
}

// Выполнение покупки
async function fulfillOrder(order: any) {
    const { executeAllActions } = await seaport.fulfillOrder({
        order,
        accountAddress: await provider.getSigner().getAddress(),
    });

    const transaction = await executeAllActions();
    console.log("Purchase completed:", transaction);
    return transaction;
}

// Создание оффера (предложение на покупку)
async function makeOffer(
    nftAddress: string,
    tokenId: string,
    offerAmountInWeth: string,
    wethAddress: string
) {
    const { executeAllActions } = await seaport.createOrder({
        offer: [
            {
                itemType: 1, // ERC20 (WETH)
                token: wethAddress,
                amount: ethers.utils.parseEther(offerAmountInWeth).toString(),
            },
        ],
        consideration: [
            {
                itemType: 2, // ERC721
                token: nftAddress,
                identifier: tokenId,
                recipient: await provider.getSigner().getAddress(),
            },
        ],
    });

    const order = await executeAllActions();
    return order;
}

// Отмена ордера
async function cancelOrder(orderParameters: any) {
    const tx = await seaport.cancelOrders([orderParameters]);
    await tx.wait();
    console.log("Order cancelled");
}
```

### Batch операции в Seaport

```typescript
// Покупка нескольких NFT за одну транзакцию
async function fulfillMultipleOrders(orders: any[]) {
    const { executeAllActions } = await seaport.fulfillOrders({
        fulfillOrderDetails: orders.map(order => ({
            order,
        })),
        accountAddress: await provider.getSigner().getAddress(),
    });

    const transaction = await executeAllActions();
    return transaction;
}

// Создание collection offer
async function createCollectionOffer(
    collectionAddress: string,
    offerAmountInWeth: string,
    wethAddress: string
) {
    const { executeAllActions } = await seaport.createOrder({
        offer: [
            {
                itemType: 1,
                token: wethAddress,
                amount: ethers.utils.parseEther(offerAmountInWeth).toString(),
            },
        ],
        consideration: [
            {
                itemType: 4, // ERC721_WITH_CRITERIA
                token: collectionAddress,
                identifier: "0", // Любой токен из коллекции
                recipient: await provider.getSigner().getAddress(),
            },
        ],
    });

    return await executeAllActions();
}
```

## OpenSea API

### Получение NFT коллекции

```typescript
import axios from "axios";

const OPENSEA_API_KEY = process.env.OPENSEA_API_KEY;
const BASE_URL = "https://api.opensea.io/api/v2";

// Получение информации о коллекции
async function getCollection(collectionSlug: string) {
    const response = await axios.get(
        `${BASE_URL}/collections/${collectionSlug}`,
        {
            headers: {
                "X-API-KEY": OPENSEA_API_KEY,
            },
        }
    );
    return response.data;
}

// Получение NFT из коллекции
async function getNFTsFromCollection(collectionSlug: string, limit: number = 50) {
    const response = await axios.get(
        `${BASE_URL}/collection/${collectionSlug}/nfts`,
        {
            params: { limit },
            headers: {
                "X-API-KEY": OPENSEA_API_KEY,
            },
        }
    );
    return response.data.nfts;
}

// Получение конкретного NFT
async function getNFT(chain: string, contractAddress: string, tokenId: string) {
    const response = await axios.get(
        `${BASE_URL}/chain/${chain}/contract/${contractAddress}/nfts/${tokenId}`,
        {
            headers: {
                "X-API-KEY": OPENSEA_API_KEY,
            },
        }
    );
    return response.data.nft;
}

// Получение листингов
async function getListings(collectionSlug: string) {
    const response = await axios.get(
        `${BASE_URL}/listings/collection/${collectionSlug}/all`,
        {
            headers: {
                "X-API-KEY": OPENSEA_API_KEY,
            },
        }
    );
    return response.data.listings;
}

// Получение офферов
async function getOffers(collectionSlug: string) {
    const response = await axios.get(
        `${BASE_URL}/offers/collection/${collectionSlug}/all`,
        {
            headers: {
                "X-API-KEY": OPENSEA_API_KEY,
            },
        }
    );
    return response.data.offers;
}
```

## Blur API и интеграция

```typescript
// Blur использует собственный API для агрегации
// Пример получения floor price

async function getBlurFloorPrice(collectionAddress: string) {
    // Blur API требует аутентификации
    const response = await fetch(
        `https://api.blur.io/v1/collections/${collectionAddress}`,
        {
            headers: {
                "Authorization": `Bearer ${BLUR_API_KEY}`,
            },
        }
    );
    const data = await response.json();
    return data.collection.floorPrice;
}

// Получение bids
async function getBlurBids(collectionAddress: string) {
    const response = await fetch(
        `https://api.blur.io/v1/collections/${collectionAddress}/executable-bids`,
        {
            headers: {
                "Authorization": `Bearer ${BLUR_API_KEY}`,
            },
        }
    );
    return response.json();
}
```

## Rarible Protocol

### Интеграция с Rarible SDK

```typescript
import { createRaribleSdk } from "@rarible/sdk";
import { toCollectionId, toItemId } from "@rarible/types";

// Инициализация SDK
const sdk = createRaribleSdk(wallet, "prod");

// Создание листинга
async function createRaribleListing(
    contractAddress: string,
    tokenId: string,
    priceInEth: string
) {
    const itemId = toItemId(`ETHEREUM:${contractAddress}:${tokenId}`);

    const sellOrder = await sdk.order.sell({
        itemId,
        amount: 1,
        price: priceInEth,
        currency: {
            "@type": "ETH",
        },
    });

    return sellOrder;
}

// Покупка NFT
async function buyOnRarible(orderId: string) {
    const result = await sdk.order.buy({
        orderId,
        amount: 1,
    });

    return result;
}

// Создание оффера
async function makeRaribleOffer(
    contractAddress: string,
    tokenId: string,
    offerPrice: string
) {
    const itemId = toItemId(`ETHEREUM:${contractAddress}:${tokenId}`);

    const bid = await sdk.order.bid({
        itemId,
        amount: 1,
        price: offerPrice,
        currency: {
            "@type": "WETH",
        },
    });

    return bid;
}
```

## Агрегаторы NFT

### Reservoir

Reservoir — инфраструктура для NFT агрегации и API.

```typescript
import { createClient, getClient } from "@reservoir0x/reservoir-sdk";

// Инициализация клиента
createClient({
    chains: [{
        id: 1,
        baseApiUrl: "https://api.reservoir.tools",
        default: true,
        apiKey: RESERVOIR_API_KEY,
    }],
});

// Покупка NFT с лучшей ценой
async function buyToken(
    contractAddress: string,
    tokenId: string,
    signer: any
) {
    await getClient()?.actions.buyToken({
        items: [{ token: `${contractAddress}:${tokenId}` }],
        signer,
        onProgress: (steps) => {
            console.log("Progress:", steps);
        },
    });
}

// Получение floor ask
async function getFloorAsk(collectionAddress: string) {
    const response = await fetch(
        `https://api.reservoir.tools/collections/v6?id=${collectionAddress}`,
        {
            headers: {
                "x-api-key": RESERVOIR_API_KEY,
            },
        }
    );
    const data = await response.json();
    return data.collections[0].floorAsk;
}
```

## Сравнение маркетплейсов

| Критерий | OpenSea | Blur | Rarible |
|----------|---------|------|---------|
| **Комиссия** | 2.5% | 0.5% | 2.5% |
| **Роялти** | По желанию | Опционально | Обязательно |
| **Скорость** | Средняя | Высокая | Средняя |
| **Ликвидность** | Высокая | Высокая | Средняя |
| **Мульти-чейн** | Да | Нет | Да |
| **Аналитика** | Базовая | Продвинутая | Базовая |
| **Целевая аудитория** | Все | Трейдеры | Создатели |

## Best Practices для работы с маркетплейсами

### Для разработчиков

1. **Выбор маркетплейса**
   - Оцените целевую аудиторию проекта
   - Учитывайте комиссии и роялти
   - Проверьте поддержку нужного блокчейна

2. **Интеграция**
   - Используйте официальные SDK
   - Кэшируйте данные для производительности
   - Обрабатывайте rate limits

3. **Листинг коллекции**
   - Верифицируйте коллекцию
   - Добавьте качественные метаданные
   - Настройте роялти

### Для трейдеров

1. **Анализ**
   - Следите за floor price
   - Анализируйте объёмы торгов
   - Используйте агрегаторы

2. **Безопасность**
   - Проверяйте контракты
   - Остерегайтесь фишинга
   - Используйте hardware кошельки

## Полезные ресурсы

- [OpenSea Developer Docs](https://docs.opensea.io/)
- [Seaport Protocol](https://github.com/ProjectOpenSea/seaport)
- [Blur API](https://docs.blur.foundation/)
- [Rarible Protocol](https://docs.rarible.org/)
- [Reservoir API](https://docs.reservoir.tools/)
- [NFT Price Floor](https://nftpricefloor.com/) — аналитика
