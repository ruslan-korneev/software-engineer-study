# Разработка NFT

## Обзор разработки NFT

Разработка NFT включает создание смарт-контрактов, систем метаданных, интеграцию с маркетплейсами и построение пользовательских интерфейсов для минтинга и управления токенами.

## Стандарты NFT

### ERC-721: Базовый стандарт NFT

ERC-721 — это стандарт для создания уникальных, неделимых токенов на Ethereum.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721Enumerable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Strings.sol";

contract MyNFTCollection is ERC721, ERC721URIStorage, ERC721Enumerable, Ownable {
    using Strings for uint256;

    uint256 private _nextTokenId;
    uint256 public maxSupply = 10000;
    uint256 public mintPrice = 0.08 ether;
    uint256 public maxPerWallet = 5;

    string private _baseTokenURI;
    bool public mintingEnabled = false;

    mapping(address => uint256) public mintedPerWallet;

    constructor(
        string memory name,
        string memory symbol,
        string memory baseURI
    ) ERC721(name, symbol) Ownable(msg.sender) {
        _baseTokenURI = baseURI;
    }

    // Публичный минт
    function mint(uint256 quantity) external payable {
        require(mintingEnabled, "Minting is not enabled");
        require(quantity > 0, "Quantity must be greater than 0");
        require(_nextTokenId + quantity <= maxSupply, "Exceeds max supply");
        require(
            mintedPerWallet[msg.sender] + quantity <= maxPerWallet,
            "Exceeds max per wallet"
        );
        require(msg.value >= mintPrice * quantity, "Insufficient payment");

        mintedPerWallet[msg.sender] += quantity;

        for (uint256 i = 0; i < quantity; i++) {
            uint256 tokenId = _nextTokenId++;
            _safeMint(msg.sender, tokenId);
        }
    }

    // Административный минт
    function adminMint(address to, uint256 quantity) external onlyOwner {
        require(_nextTokenId + quantity <= maxSupply, "Exceeds max supply");

        for (uint256 i = 0; i < quantity; i++) {
            uint256 tokenId = _nextTokenId++;
            _safeMint(to, tokenId);
        }
    }

    // Управление минтингом
    function setMintingEnabled(bool enabled) external onlyOwner {
        mintingEnabled = enabled;
    }

    function setMintPrice(uint256 price) external onlyOwner {
        mintPrice = price;
    }

    function setBaseURI(string memory baseURI) external onlyOwner {
        _baseTokenURI = baseURI;
    }

    // Вывод средств
    function withdraw() external onlyOwner {
        uint256 balance = address(this).balance;
        require(balance > 0, "No funds to withdraw");

        (bool success, ) = payable(owner()).call{value: balance}("");
        require(success, "Withdrawal failed");
    }

    // Переопределения для совместимости
    function _baseURI() internal view override returns (string memory) {
        return _baseTokenURI;
    }

    function tokenURI(uint256 tokenId) public view override(ERC721, ERC721URIStorage) returns (string memory) {
        _requireOwned(tokenId);

        string memory baseURI = _baseURI();
        return bytes(baseURI).length > 0
            ? string(abi.encodePacked(baseURI, tokenId.toString(), ".json"))
            : "";
    }

    function _update(address to, uint256 tokenId, address auth)
        internal
        override(ERC721, ERC721Enumerable)
        returns (address)
    {
        return super._update(to, tokenId, auth);
    }

    function _increaseBalance(address account, uint128 value)
        internal
        override(ERC721, ERC721Enumerable)
    {
        super._increaseBalance(account, value);
    }

    function supportsInterface(bytes4 interfaceId)
        public
        view
        override(ERC721, ERC721URIStorage, ERC721Enumerable)
        returns (bool)
    {
        return super.supportsInterface(interfaceId);
    }
}
```

### ERC-1155: Мульти-токен стандарт

ERC-1155 позволяет создавать как fungible, так и non-fungible токены в одном контракте.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";
import "@openzeppelin/contracts/token/ERC1155/extensions/ERC1155Supply.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Strings.sol";

contract GameAssets is ERC1155, ERC1155Supply, Ownable {
    using Strings for uint256;

    string public name = "Game Assets";
    string public symbol = "GAME";

    // Типы токенов
    uint256 public constant GOLD = 0;           // Fungible
    uint256 public constant SILVER = 1;         // Fungible
    uint256 public constant LEGENDARY_SWORD = 2; // NFT (max supply = 1)
    uint256 public constant RARE_ARMOR = 3;     // Semi-fungible (limited supply)

    // Максимальные запасы для разных типов
    mapping(uint256 => uint256) public maxSupply;
    mapping(uint256 => uint256) public mintPrice;

    string private _baseURI;

    constructor(string memory baseURI) ERC1155(baseURI) Ownable(msg.sender) {
        _baseURI = baseURI;

        // Настройка лимитов
        maxSupply[LEGENDARY_SWORD] = 1;    // Уникальный предмет
        maxSupply[RARE_ARMOR] = 100;       // Ограниченная серия
        // GOLD и SILVER без лимита (maxSupply = 0 означает бесконечно)

        // Настройка цен
        mintPrice[GOLD] = 0.001 ether;
        mintPrice[SILVER] = 0.0005 ether;
        mintPrice[LEGENDARY_SWORD] = 1 ether;
        mintPrice[RARE_ARMOR] = 0.1 ether;
    }

    // Минт токена
    function mint(uint256 id, uint256 amount) external payable {
        require(msg.value >= mintPrice[id] * amount, "Insufficient payment");

        if (maxSupply[id] > 0) {
            require(
                totalSupply(id) + amount <= maxSupply[id],
                "Exceeds max supply"
            );
        }

        _mint(msg.sender, id, amount, "");
    }

    // Batch минт
    function mintBatch(
        uint256[] memory ids,
        uint256[] memory amounts
    ) external payable {
        uint256 totalCost = 0;

        for (uint256 i = 0; i < ids.length; i++) {
            totalCost += mintPrice[ids[i]] * amounts[i];

            if (maxSupply[ids[i]] > 0) {
                require(
                    totalSupply(ids[i]) + amounts[i] <= maxSupply[ids[i]],
                    "Exceeds max supply"
                );
            }
        }

        require(msg.value >= totalCost, "Insufficient payment");

        _mintBatch(msg.sender, ids, amounts, "");
    }

    // Административные функции
    function adminMint(
        address to,
        uint256 id,
        uint256 amount
    ) external onlyOwner {
        if (maxSupply[id] > 0) {
            require(
                totalSupply(id) + amount <= maxSupply[id],
                "Exceeds max supply"
            );
        }
        _mint(to, id, amount, "");
    }

    function setMaxSupply(uint256 id, uint256 supply) external onlyOwner {
        require(totalSupply(id) <= supply, "Supply already exceeds new max");
        maxSupply[id] = supply;
    }

    function setMintPrice(uint256 id, uint256 price) external onlyOwner {
        mintPrice[id] = price;
    }

    function setBaseURI(string memory baseURI) external onlyOwner {
        _baseURI = baseURI;
    }

    // URI с подстановкой ID
    function uri(uint256 id) public view override returns (string memory) {
        return string(abi.encodePacked(_baseURI, id.toString(), ".json"));
    }

    function withdraw() external onlyOwner {
        uint256 balance = address(this).balance;
        require(balance > 0, "No funds to withdraw");

        (bool success, ) = payable(owner()).call{value: balance}("");
        require(success, "Withdrawal failed");
    }

    // Переопределения
    function _update(
        address from,
        address to,
        uint256[] memory ids,
        uint256[] memory values
    ) internal override(ERC1155, ERC1155Supply) {
        super._update(from, to, ids, values);
    }
}
```

### Сравнение ERC-721 и ERC-1155

| Критерий | ERC-721 | ERC-1155 |
|----------|---------|----------|
| **Типы токенов** | Только NFT | NFT + FT + Semi-FT |
| **Уникальность** | Каждый токен уникален | Токены с одинаковым ID идентичны |
| **Gas (1 минт)** | ~50,000 | ~50,000 |
| **Gas (batch минт)** | N * ~50,000 | ~50,000 + N * ~5,000 |
| **Применение** | Искусство, PFP | Игры, коллекции |
| **Простота** | Простой | Сложнее |

## Метаданные NFT

### On-chain vs Off-chain метаданные

```
┌─────────────────────────────────────────────────────────────┐
│                    Метаданные NFT                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  On-chain                      Off-chain                    │
│  ┌──────────────────┐         ┌──────────────────────────┐ │
│  │ Хранение в       │         │ IPFS / Arweave /         │ │
│  │ смарт-контракте  │         │ Centralized Server       │ │
│  └──────────────────┘         └──────────────────────────┘ │
│                                                              │
│  Плюсы:                       Плюсы:                        │
│  - Постоянство                - Дешевле                     │
│  - Децентрализация            - Гибкость                    │
│  - Нет внешних зависимостей   - Большие файлы              │
│                                                              │
│  Минусы:                      Минусы:                       │
│  - Дорого                     - Зависимость от хранилища   │
│  - Ограничения размера        - Может исчезнуть            │
│  - Сложнее обновлять          - Требует пиннинга (IPFS)    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### On-chain SVG NFT

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/utils/Base64.sol";
import "@openzeppelin/contracts/utils/Strings.sol";

contract OnChainSVGNFT is ERC721 {
    using Strings for uint256;

    uint256 private _nextTokenId;

    // Данные каждого NFT
    struct TokenData {
        string name;
        string color;
        uint256 size;
    }

    mapping(uint256 => TokenData) public tokenData;

    constructor() ERC721("OnChain SVG", "OSVG") {}

    function mint(string memory name, string memory color, uint256 size) external {
        uint256 tokenId = _nextTokenId++;
        tokenData[tokenId] = TokenData(name, color, size);
        _safeMint(msg.sender, tokenId);
    }

    function generateSVG(uint256 tokenId) internal view returns (string memory) {
        TokenData memory data = tokenData[tokenId];

        return string(abi.encodePacked(
            '<svg xmlns="http://www.w3.org/2000/svg" width="350" height="350">',
            '<rect width="100%" height="100%" fill="#1a1a2e"/>',
            '<circle cx="175" cy="175" r="', data.size.toString(), '" fill="', data.color, '"/>',
            '<text x="175" y="320" text-anchor="middle" fill="white" font-size="24">', data.name, '</text>',
            '</svg>'
        ));
    }

    function tokenURI(uint256 tokenId) public view override returns (string memory) {
        _requireOwned(tokenId);

        TokenData memory data = tokenData[tokenId];
        string memory svg = generateSVG(tokenId);

        string memory json = Base64.encode(bytes(string(abi.encodePacked(
            '{"name":"', data.name, ' #', tokenId.toString(), '",',
            '"description":"Fully on-chain SVG NFT",',
            '"image":"data:image/svg+xml;base64,', Base64.encode(bytes(svg)), '",',
            '"attributes":[',
                '{"trait_type":"Color","value":"', data.color, '"},',
                '{"trait_type":"Size","value":', data.size.toString(), '}',
            ']}'
        ))));

        return string(abi.encodePacked("data:application/json;base64,", json));
    }
}
```

### IPFS метаданные

```typescript
// Загрузка метаданных на IPFS через Pinata
import axios from "axios";
import FormData from "form-data";
import fs from "fs";

const PINATA_API_KEY = process.env.PINATA_API_KEY;
const PINATA_SECRET_KEY = process.env.PINATA_SECRET_KEY;

// Загрузка изображения
async function uploadImageToPinata(imagePath: string): Promise<string> {
    const formData = new FormData();
    formData.append("file", fs.createReadStream(imagePath));

    const response = await axios.post(
        "https://api.pinata.cloud/pinning/pinFileToIPFS",
        formData,
        {
            headers: {
                "Content-Type": `multipart/form-data; boundary=${formData.getBoundary()}`,
                "pinata_api_key": PINATA_API_KEY,
                "pinata_secret_api_key": PINATA_SECRET_KEY,
            },
        }
    );

    return `ipfs://${response.data.IpfsHash}`;
}

// Загрузка JSON метаданных
async function uploadMetadataToPinata(metadata: object): Promise<string> {
    const response = await axios.post(
        "https://api.pinata.cloud/pinning/pinJSONToIPFS",
        metadata,
        {
            headers: {
                "Content-Type": "application/json",
                "pinata_api_key": PINATA_API_KEY,
                "pinata_secret_api_key": PINATA_SECRET_KEY,
            },
        }
    );

    return `ipfs://${response.data.IpfsHash}`;
}

// Генерация метаданных для коллекции
async function generateCollectionMetadata(
    collectionSize: number,
    baseImageFolder: string
) {
    const metadataArray = [];

    for (let i = 0; i < collectionSize; i++) {
        // Загружаем изображение
        const imageUri = await uploadImageToPinata(`${baseImageFolder}/${i}.png`);

        // Создаём метаданные
        const metadata = {
            name: `My NFT #${i}`,
            description: "An awesome NFT from my collection",
            image: imageUri,
            attributes: [
                {
                    trait_type: "Background",
                    value: getRandomBackground(),
                },
                {
                    trait_type: "Rarity",
                    value: calculateRarity(i),
                },
            ],
        };

        // Загружаем метаданные
        const metadataUri = await uploadMetadataToPinata(metadata);
        metadataArray.push(metadataUri);

        console.log(`Uploaded metadata for token ${i}: ${metadataUri}`);
    }

    return metadataArray;
}

function getRandomBackground(): string {
    const backgrounds = ["Blue", "Red", "Green", "Purple", "Gold"];
    return backgrounds[Math.floor(Math.random() * backgrounds.length)];
}

function calculateRarity(tokenId: number): string {
    if (tokenId < 10) return "Legendary";
    if (tokenId < 100) return "Epic";
    if (tokenId < 500) return "Rare";
    return "Common";
}
```

## ERC-2981: Роялти

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/common/ERC2981.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract NFTWithRoyalties is ERC721, ERC2981, Ownable {
    uint256 private _nextTokenId;
    string private _baseTokenURI;

    // Роялти по умолчанию: 5% (500 basis points)
    uint96 public constant DEFAULT_ROYALTY = 500;

    constructor(
        string memory name,
        string memory symbol,
        string memory baseURI,
        address royaltyReceiver
    ) ERC721(name, symbol) Ownable(msg.sender) {
        _baseTokenURI = baseURI;
        // Устанавливаем роялти по умолчанию для всей коллекции
        _setDefaultRoyalty(royaltyReceiver, DEFAULT_ROYALTY);
    }

    function mint(address to) external onlyOwner returns (uint256) {
        uint256 tokenId = _nextTokenId++;
        _safeMint(to, tokenId);
        return tokenId;
    }

    // Минт с кастомными роялти
    function mintWithRoyalty(
        address to,
        address royaltyReceiver,
        uint96 royaltyPercentage // В basis points (100 = 1%)
    ) external onlyOwner returns (uint256) {
        require(royaltyPercentage <= 10000, "Royalty too high"); // Max 100%

        uint256 tokenId = _nextTokenId++;
        _safeMint(to, tokenId);
        _setTokenRoyalty(tokenId, royaltyReceiver, royaltyPercentage);

        return tokenId;
    }

    // Изменить роялти для конкретного токена
    function setTokenRoyalty(
        uint256 tokenId,
        address receiver,
        uint96 feeNumerator
    ) external onlyOwner {
        _setTokenRoyalty(tokenId, receiver, feeNumerator);
    }

    // Изменить роялти по умолчанию
    function setDefaultRoyalty(address receiver, uint96 feeNumerator) external onlyOwner {
        _setDefaultRoyalty(receiver, feeNumerator);
    }

    // Удалить роялти по умолчанию
    function deleteDefaultRoyalty() external onlyOwner {
        _deleteDefaultRoyalty();
    }

    // Удалить роялти для конкретного токена
    function resetTokenRoyalty(uint256 tokenId) external onlyOwner {
        _resetTokenRoyalty(tokenId);
    }

    function _baseURI() internal view override returns (string memory) {
        return _baseTokenURI;
    }

    function supportsInterface(bytes4 interfaceId)
        public
        view
        override(ERC721, ERC2981)
        returns (bool)
    {
        return super.supportsInterface(interfaceId);
    }
}
```

### Использование роялти в маркетплейсе

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/interfaces/IERC2981.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract NFTMarketplace is ReentrancyGuard {
    struct Listing {
        address seller;
        uint256 price;
        bool active;
    }

    // NFT контракт => Token ID => Listing
    mapping(address => mapping(uint256 => Listing)) public listings;

    uint256 public marketplaceFee = 250; // 2.5%
    address public feeRecipient;

    event Listed(address indexed nft, uint256 indexed tokenId, address seller, uint256 price);
    event Sold(address indexed nft, uint256 indexed tokenId, address buyer, uint256 price);

    constructor(address _feeRecipient) {
        feeRecipient = _feeRecipient;
    }

    function listNFT(address nft, uint256 tokenId, uint256 price) external {
        IERC721(nft).transferFrom(msg.sender, address(this), tokenId);

        listings[nft][tokenId] = Listing({
            seller: msg.sender,
            price: price,
            active: true
        });

        emit Listed(nft, tokenId, msg.sender, price);
    }

    function buyNFT(address nft, uint256 tokenId) external payable nonReentrant {
        Listing storage listing = listings[nft][tokenId];
        require(listing.active, "Listing not active");
        require(msg.value >= listing.price, "Insufficient payment");

        listing.active = false;

        uint256 salePrice = listing.price;

        // Рассчитываем комиссию маркетплейса
        uint256 marketplaceCut = (salePrice * marketplaceFee) / 10000;

        // Проверяем и рассчитываем роялти
        uint256 royaltyAmount = 0;
        address royaltyReceiver;

        if (IERC165(nft).supportsInterface(type(IERC2981).interfaceId)) {
            (royaltyReceiver, royaltyAmount) = IERC2981(nft).royaltyInfo(tokenId, salePrice);
        }

        // Рассчитываем сумму для продавца
        uint256 sellerProceeds = salePrice - marketplaceCut - royaltyAmount;

        // Переводим NFT покупателю
        IERC721(nft).transferFrom(address(this), msg.sender, tokenId);

        // Распределяем платежи
        payable(feeRecipient).transfer(marketplaceCut);

        if (royaltyAmount > 0 && royaltyReceiver != address(0)) {
            payable(royaltyReceiver).transfer(royaltyAmount);
        }

        payable(listing.seller).transfer(sellerProceeds);

        // Возвращаем излишек
        if (msg.value > salePrice) {
            payable(msg.sender).transfer(msg.value - salePrice);
        }

        emit Sold(nft, tokenId, msg.sender, salePrice);
    }
}
```

## Gas-оптимизация: ERC-721A

ERC-721A от Azuki оптимизирован для batch minting.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "erc721a/contracts/ERC721A.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract OptimizedNFT is ERC721A, Ownable {
    uint256 public maxSupply = 10000;
    uint256 public mintPrice = 0.08 ether;
    uint256 public maxPerTx = 10;

    string private _baseTokenURI;
    bool public mintingEnabled = false;

    constructor(string memory baseURI) ERC721A("OptimizedNFT", "ONFT") Ownable(msg.sender) {
        _baseTokenURI = baseURI;
    }

    // Оптимизированный batch mint
    function mint(uint256 quantity) external payable {
        require(mintingEnabled, "Minting not enabled");
        require(quantity > 0 && quantity <= maxPerTx, "Invalid quantity");
        require(_totalMinted() + quantity <= maxSupply, "Exceeds supply");
        require(msg.value >= mintPrice * quantity, "Insufficient payment");

        _mint(msg.sender, quantity);
    }

    // Получение токенов владельца (оптимизировано в ERC721A)
    function tokensOfOwner(address owner) external view returns (uint256[] memory) {
        uint256 tokenCount = balanceOf(owner);
        uint256[] memory tokenIds = new uint256[](tokenCount);

        uint256 index = 0;
        for (uint256 i = _startTokenId(); i < _nextTokenId(); i++) {
            if (_exists(i) && ownerOf(i) == owner) {
                tokenIds[index++] = i;
            }
        }

        return tokenIds;
    }

    function setMintingEnabled(bool enabled) external onlyOwner {
        mintingEnabled = enabled;
    }

    function _baseURI() internal view override returns (string memory) {
        return _baseTokenURI;
    }

    function _startTokenId() internal pure override returns (uint256) {
        return 1;
    }
}
```

## Whitelist с Merkle Tree

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract WhitelistNFT is ERC721, Ownable {
    uint256 private _nextTokenId;
    bytes32 public merkleRoot;

    uint256 public whitelistPrice = 0.05 ether;
    uint256 public publicPrice = 0.08 ether;

    mapping(address => bool) public whitelistClaimed;

    enum SalePhase { Inactive, Whitelist, Public }
    SalePhase public salePhase = SalePhase.Inactive;

    constructor() ERC721("WhitelistNFT", "WLNFT") Ownable(msg.sender) {}

    // Whitelist mint с Merkle proof
    function whitelistMint(bytes32[] calldata proof) external payable {
        require(salePhase == SalePhase.Whitelist, "Whitelist sale not active");
        require(!whitelistClaimed[msg.sender], "Already claimed");
        require(msg.value >= whitelistPrice, "Insufficient payment");

        // Верификация Merkle proof
        bytes32 leaf = keccak256(abi.encodePacked(msg.sender));
        require(MerkleProof.verify(proof, merkleRoot, leaf), "Invalid proof");

        whitelistClaimed[msg.sender] = true;
        _safeMint(msg.sender, _nextTokenId++);
    }

    // Публичный минт
    function publicMint() external payable {
        require(salePhase == SalePhase.Public, "Public sale not active");
        require(msg.value >= publicPrice, "Insufficient payment");

        _safeMint(msg.sender, _nextTokenId++);
    }

    // Административные функции
    function setMerkleRoot(bytes32 _merkleRoot) external onlyOwner {
        merkleRoot = _merkleRoot;
    }

    function setSalePhase(SalePhase phase) external onlyOwner {
        salePhase = phase;
    }

    // Проверка адреса в whitelist
    function isWhitelisted(address account, bytes32[] calldata proof) external view returns (bool) {
        bytes32 leaf = keccak256(abi.encodePacked(account));
        return MerkleProof.verify(proof, merkleRoot, leaf);
    }
}
```

### Генерация Merkle Tree (JavaScript)

```typescript
import { MerkleTree } from "merkletreejs";
import { keccak256 } from "ethers";

// Список адресов для whitelist
const whitelist = [
    "0x1234567890123456789012345678901234567890",
    "0xabcdefabcdefabcdefabcdefabcdefabcdefabcd",
    "0x9876543210987654321098765432109876543210",
];

// Создаём листья дерева
const leaves = whitelist.map(addr => keccak256(addr));

// Создаём Merkle Tree
const merkleTree = new MerkleTree(leaves, keccak256, { sortPairs: true });

// Получаем root
const merkleRoot = merkleTree.getHexRoot();
console.log("Merkle Root:", merkleRoot);

// Генерация proof для адреса
function getProof(address: string): string[] {
    const leaf = keccak256(address);
    return merkleTree.getHexProof(leaf);
}

// Пример использования
const userAddress = "0x1234567890123456789012345678901234567890";
const proof = getProof(userAddress);
console.log("Proof for", userAddress, ":", proof);
```

## Reveal механика

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Strings.sol";

contract RevealableNFT is ERC721, Ownable {
    using Strings for uint256;

    uint256 private _nextTokenId;

    string public hiddenMetadataURI;
    string public revealedBaseURI;
    bool public revealed = false;

    constructor(
        string memory _hiddenMetadataURI
    ) ERC721("RevealableNFT", "RNFT") Ownable(msg.sender) {
        hiddenMetadataURI = _hiddenMetadataURI;
    }

    function mint() external payable {
        _safeMint(msg.sender, _nextTokenId++);
    }

    function tokenURI(uint256 tokenId) public view override returns (string memory) {
        _requireOwned(tokenId);

        if (!revealed) {
            return hiddenMetadataURI;
        }

        return string(abi.encodePacked(revealedBaseURI, tokenId.toString(), ".json"));
    }

    // Reveal коллекции
    function reveal(string memory _revealedBaseURI) external onlyOwner {
        revealed = true;
        revealedBaseURI = _revealedBaseURI;
    }

    // Установить placeholder метаданные
    function setHiddenMetadataURI(string memory _hiddenMetadataURI) external onlyOwner {
        hiddenMetadataURI = _hiddenMetadataURI;
    }
}
```

## Тестирование NFT контрактов

```typescript
import { expect } from "chai";
import { ethers } from "hardhat";
import { MyNFTCollection } from "../typechain-types";

describe("MyNFTCollection", function () {
    let nft: MyNFTCollection;
    let owner: any;
    let user1: any;
    let user2: any;

    beforeEach(async function () {
        [owner, user1, user2] = await ethers.getSigners();

        const NFT = await ethers.getContractFactory("MyNFTCollection");
        nft = await NFT.deploy("MyNFT", "MNFT", "ipfs://QmXxx.../");
        await nft.waitForDeployment();
    });

    describe("Minting", function () {
        it("Should allow owner to enable minting", async function () {
            await nft.setMintingEnabled(true);
            expect(await nft.mintingEnabled()).to.equal(true);
        });

        it("Should allow users to mint when enabled", async function () {
            await nft.setMintingEnabled(true);
            const mintPrice = await nft.mintPrice();

            await nft.connect(user1).mint(1, { value: mintPrice });

            expect(await nft.balanceOf(user1.address)).to.equal(1);
            expect(await nft.ownerOf(0)).to.equal(user1.address);
        });

        it("Should reject minting when disabled", async function () {
            const mintPrice = await nft.mintPrice();

            await expect(
                nft.connect(user1).mint(1, { value: mintPrice })
            ).to.be.revertedWith("Minting is not enabled");
        });

        it("Should reject insufficient payment", async function () {
            await nft.setMintingEnabled(true);

            await expect(
                nft.connect(user1).mint(1, { value: 0 })
            ).to.be.revertedWith("Insufficient payment");
        });

        it("Should enforce max per wallet limit", async function () {
            await nft.setMintingEnabled(true);
            const mintPrice = await nft.mintPrice();
            const maxPerWallet = await nft.maxPerWallet();

            // Mint max allowed
            await nft.connect(user1).mint(maxPerWallet, {
                value: mintPrice * maxPerWallet
            });

            // Try to mint one more
            await expect(
                nft.connect(user1).mint(1, { value: mintPrice })
            ).to.be.revertedWith("Exceeds max per wallet");
        });
    });

    describe("Token URI", function () {
        it("Should return correct token URI", async function () {
            await nft.setMintingEnabled(true);
            const mintPrice = await nft.mintPrice();

            await nft.connect(user1).mint(1, { value: mintPrice });

            expect(await nft.tokenURI(0)).to.equal("ipfs://QmXxx.../0.json");
        });
    });

    describe("Royalties", function () {
        it("Should return correct royalty info", async function () {
            // Если контракт поддерживает ERC2981
            const salePrice = ethers.parseEther("1");
            const [receiver, amount] = await nft.royaltyInfo(0, salePrice);

            // 5% роялти
            expect(amount).to.equal(ethers.parseEther("0.05"));
        });
    });
});
```

## Best Practices разработки NFT

### Безопасность

1. **Используйте проверенные библиотеки**
   - OpenZeppelin Contracts
   - ERC721A для оптимизации

2. **Access Control**
   - Ограничивайте административные функции
   - Используйте Ownable или AccessControl

3. **Reentrancy Protection**
   - Используйте ReentrancyGuard для функций с переводами

4. **Проводите аудит**
   - Автоматический анализ (Slither, Mythril)
   - Профессиональный аудит для крупных проектов

### Gas-оптимизация

1. **Batch operations**
   - ERC721A для эффективного batch minting
   - ERC1155 для мульти-токен операций

2. **Merkle Trees**
   - Для whitelist вместо mapping

3. **Off-chain метаданные**
   - IPFS вместо on-chain хранения

4. **Оптимизация storage**
   - Упаковка переменных
   - Использование events вместо storage

### Метаданные

1. **Децентрализованное хранение**
   - IPFS с пиннингом
   - Arweave для постоянства

2. **Content-addressable URI**
   - Используйте CID для неизменяемости

3. **Стандартная структура**
   - Следуйте ERC721 Metadata JSON Schema

## Полезные ресурсы

- [EIP-721](https://eips.ethereum.org/EIPS/eip-721)
- [EIP-1155](https://eips.ethereum.org/EIPS/eip-1155)
- [EIP-2981](https://eips.ethereum.org/EIPS/eip-2981)
- [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/)
- [ERC721A](https://www.erc721a.org/)
- [Seaport Protocol](https://docs.opensea.io/reference/seaport-overview)
- [Hardhat](https://hardhat.org/docs)
- [NFT School](https://nftschool.dev/)
