# NFT (Non-Fungible Tokens)

## Что такое NFT?

**NFT (Non-Fungible Token)** — это невзаимозаменяемый токен, представляющий собой уникальный цифровой актив на блокчейне. В отличие от криптовалют (BTC, ETH), каждый NFT уникален и не может быть заменен другим токеном.

### Ключевые характеристики NFT

| Характеристика | Описание |
|----------------|----------|
| **Уникальность** | Каждый токен имеет уникальный идентификатор |
| **Неделимость** | NFT нельзя разделить на части (в отличие от ERC-20) |
| **Проверяемость** | История владения прозрачна и верифицируема |
| **Переносимость** | Токены можно передавать между кошельками |
| **Программируемость** | Поддержка роялти и других механик |

## Стандарты NFT

### ERC-721 — Базовый стандарт

ERC-721 — первый и наиболее распространённый стандарт для NFT в сети Ethereum.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MyNFT is ERC721, Ownable {
    uint256 private _nextTokenId;

    constructor() ERC721("MyNFT", "MNFT") Ownable(msg.sender) {}

    function mint(address to) public onlyOwner returns (uint256) {
        uint256 tokenId = _nextTokenId++;
        _safeMint(to, tokenId);
        return tokenId;
    }
}
```

**Основные функции ERC-721:**
- `balanceOf(owner)` — количество NFT у владельца
- `ownerOf(tokenId)` — владелец конкретного токена
- `transferFrom(from, to, tokenId)` — передача токена
- `approve(to, tokenId)` — разрешение на передачу
- `setApprovalForAll(operator, approved)` — разрешение оператору

### ERC-1155 — Мульти-токен стандарт

ERC-1155 позволяет управлять множеством типов токенов (как fungible, так и non-fungible) в одном контракте.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract GameItems is ERC1155, Ownable {
    uint256 public constant GOLD = 0;      // Fungible
    uint256 public constant SWORD = 1;     // Non-fungible (amount = 1)
    uint256 public constant SHIELD = 2;    // Semi-fungible

    constructor() ERC1155("https://game.example/api/item/{id}.json") Ownable(msg.sender) {}

    function mint(address to, uint256 id, uint256 amount, bytes memory data) public onlyOwner {
        _mint(to, id, amount, data);
    }

    function mintBatch(
        address to,
        uint256[] memory ids,
        uint256[] memory amounts,
        bytes memory data
    ) public onlyOwner {
        _mintBatch(to, ids, amounts, data);
    }
}
```

### Сравнение ERC-721 и ERC-1155

| Критерий | ERC-721 | ERC-1155 |
|----------|---------|----------|
| **Тип токенов** | Только NFT | NFT + FT + Semi-FT |
| **Gas-эффективность** | Ниже | Выше (batch операции) |
| **Сложность** | Простой | Сложнее |
| **Применение** | Искусство, PFP | Игры, коллекции |
| **Метаданные** | tokenURI() | uri() с {id} |

## Метаданные NFT

### Структура метаданных (ERC-721 Metadata JSON Schema)

```json
{
    "name": "Awesome NFT #1",
    "description": "This is an awesome NFT from my collection",
    "image": "ipfs://QmXxx.../image.png",
    "external_url": "https://myproject.com/nft/1",
    "attributes": [
        {
            "trait_type": "Background",
            "value": "Blue"
        },
        {
            "trait_type": "Rarity",
            "value": "Legendary"
        },
        {
            "display_type": "number",
            "trait_type": "Power",
            "value": 100
        },
        {
            "display_type": "boost_percentage",
            "trait_type": "Speed Boost",
            "value": 25
        }
    ],
    "animation_url": "ipfs://QmYyy.../animation.mp4"
}
```

### Хранение метаданных

| Способ | Плюсы | Минусы |
|--------|-------|--------|
| **On-chain** | Постоянство, децентрализация | Дорого, ограничения |
| **IPFS** | Децентрализация, бесплатно | Требует пиннинга |
| **Arweave** | Постоянное хранение | Платное |
| **Централизованный сервер** | Просто, гибко | Единая точка отказа |

## Роялти (ERC-2981)

ERC-2981 — стандарт для роялти, позволяющий авторам получать процент от вторичных продаж.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/common/ERC2981.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract NFTWithRoyalty is ERC721, ERC2981, Ownable {
    uint256 private _nextTokenId;

    constructor() ERC721("RoyaltyNFT", "RNFT") Ownable(msg.sender) {
        // Устанавливаем роялти 5% для всех токенов
        _setDefaultRoyalty(msg.sender, 500); // 500 = 5%
    }

    function mint(address to) public onlyOwner returns (uint256) {
        uint256 tokenId = _nextTokenId++;
        _safeMint(to, tokenId);
        return tokenId;
    }

    // Установить роялти для конкретного токена
    function setTokenRoyalty(uint256 tokenId, address receiver, uint96 feeNumerator) public onlyOwner {
        _setTokenRoyalty(tokenId, receiver, feeNumerator);
    }

    // Поддержка интерфейсов
    function supportsInterface(bytes4 interfaceId) public view override(ERC721, ERC2981) returns (bool) {
        return super.supportsInterface(interfaceId);
    }
}
```

## Применение NFT

### Основные области использования

1. **Цифровое искусство** — уникальные произведения от художников
2. **Коллекционирование** — PFP-коллекции (Bored Apes, CryptoPunks)
3. **Игровые активы** — предметы, персонажи, земли в играх
4. **Музыка и видео** — права на контент
5. **Виртуальная недвижимость** — участки в метавселенных
6. **Доменные имена** — ENS, Unstoppable Domains
7. **Членство и доступ** — токены для эксклюзивного доступа
8. **Документы** — сертификаты, дипломы, билеты

## Архитектура NFT-проекта

```
┌─────────────────────────────────────────────────────────────┐
│                      Frontend (React/Next.js)                │
│  - Галерея NFT                                              │
│  - Mint страница                                            │
│  - Профиль пользователя                                     │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Web3 Integration                        │
│  - ethers.js / viem                                         │
│  - wagmi / RainbowKit                                       │
│  - Wallet Connection                                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Smart Contracts                           │
│  - ERC-721 / ERC-1155 токен                                 │
│  - Marketplace контракт                                     │
│  - Роялти (ERC-2981)                                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Metadata Storage                          │
│  - IPFS (Pinata, Infura, nft.storage)                       │
│  - Arweave                                                  │
│  - Centralized API                                          │
└─────────────────────────────────────────────────────────────┘
```

## Best Practices

### При разработке NFT

1. **Безопасность**
   - Используйте проверенные библиотеки (OpenZeppelin)
   - Проводите аудит смарт-контрактов
   - Реализуйте access control

2. **Gas-оптимизация**
   - Используйте ERC-721A для batch minting
   - Минимизируйте хранение on-chain
   - Используйте merkle trees для whitelist

3. **Метаданные**
   - Храните на IPFS или Arweave
   - Используйте Content-Addressable хэши
   - Реализуйте reveal механику правильно

4. **UX**
   - Понятный mint процесс
   - Отображение статуса транзакции
   - Интеграция с популярными кошельками

## Полезные инструменты

| Инструмент | Назначение |
|------------|------------|
| **OpenZeppelin** | Библиотека смарт-контрактов |
| **Hardhat** | Разработка и тестирование |
| **Pinata / nft.storage** | Пиннинг на IPFS |
| **Alchemy / Infura** | RPC провайдеры |
| **OpenSea SDK** | Интеграция с маркетплейсом |
| **thirdweb** | No-code/low-code NFT платформа |

## Дополнительные ресурсы

- [EIP-721: Non-Fungible Token Standard](https://eips.ethereum.org/EIPS/eip-721)
- [EIP-1155: Multi Token Standard](https://eips.ethereum.org/EIPS/eip-1155)
- [EIP-2981: NFT Royalty Standard](https://eips.ethereum.org/EIPS/eip-2981)
- [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/)
- [NFT School](https://nftschool.dev/)
