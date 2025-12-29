# ERC-721

## Введение

**ERC-721** — это стандарт для невзаимозаменяемых токенов (NFT) в сети Ethereum. Предложен в январе 2018 года Уильямом Энтрикеном, Дитером Ширли и другими. Стал основой для цифрового искусства, коллекционных предметов и игровых активов.

## Что такое невзаимозаменяемые токены?

В отличие от ERC-20, каждый ERC-721 токен уникален:
- Каждый токен имеет уникальный `tokenId`
- Токены неделимы (нельзя отправить 0.5 NFT)
- Токены могут иметь метаданные (изображение, описание)

## Интерфейс ERC-721

### Основной интерфейс

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IERC721 {
    // События
    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
    event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);
    event ApprovalForAll(address indexed owner, address indexed operator, bool approved);

    // Возвращает количество NFT у владельца
    function balanceOf(address owner) external view returns (uint256 balance);

    // Возвращает владельца токена
    function ownerOf(uint256 tokenId) external view returns (address owner);

    // Безопасный перевод с проверкой получателя
    function safeTransferFrom(
        address from,
        address to,
        uint256 tokenId,
        bytes calldata data
    ) external;

    function safeTransferFrom(
        address from,
        address to,
        uint256 tokenId
    ) external;

    // Перевод без проверки (может привести к потере NFT)
    function transferFrom(
        address from,
        address to,
        uint256 tokenId
    ) external;

    // Одобрение на передачу конкретного токена
    function approve(address to, uint256 tokenId) external;

    // Одобрение оператора на все токены владельца
    function setApprovalForAll(address operator, bool approved) external;

    // Получить одобренный адрес для токена
    function getApproved(uint256 tokenId) external view returns (address operator);

    // Проверить, является ли адрес оператором
    function isApprovedForAll(address owner, address operator) external view returns (bool);
}
```

### Интерфейс метаданных

```solidity
interface IERC721Metadata is IERC721 {
    // Название коллекции
    function name() external view returns (string memory);

    // Символ коллекции
    function symbol() external view returns (string memory);

    // URI с метаданными токена
    function tokenURI(uint256 tokenId) external view returns (string memory);
}
```

### Интерфейс получателя

Контракты, получающие NFT через `safeTransferFrom`, должны реализовать:

```solidity
interface IERC721Receiver {
    function onERC721Received(
        address operator,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external returns (bytes4);
}
```

## Полная реализация ERC-721

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/utils/introspection/ERC165.sol";

contract MyNFT is ERC165 {
    // Метаданные коллекции
    string public name;
    string public symbol;

    // Маппинги
    mapping(uint256 => address) private _owners;
    mapping(address => uint256) private _balances;
    mapping(uint256 => address) private _tokenApprovals;
    mapping(address => mapping(address => bool)) private _operatorApprovals;

    // Счётчик токенов
    uint256 private _tokenIdCounter;

    // Базовый URI для метаданных
    string private _baseTokenURI;

    // События
    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
    event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);
    event ApprovalForAll(address indexed owner, address indexed operator, bool approved);

    constructor(
        string memory _name,
        string memory _symbol,
        string memory baseURI
    ) {
        name = _name;
        symbol = _symbol;
        _baseTokenURI = baseURI;
    }

    // ERC165: поддержка интерфейсов
    function supportsInterface(bytes4 interfaceId) public view virtual override returns (bool) {
        return
            interfaceId == 0x80ac58cd || // ERC721
            interfaceId == 0x5b5e139f || // ERC721Metadata
            super.supportsInterface(interfaceId);
    }

    function balanceOf(address owner) public view returns (uint256) {
        require(owner != address(0), "ERC721: zero address");
        return _balances[owner];
    }

    function ownerOf(uint256 tokenId) public view returns (address) {
        address owner = _owners[tokenId];
        require(owner != address(0), "ERC721: invalid token ID");
        return owner;
    }

    function tokenURI(uint256 tokenId) public view returns (string memory) {
        require(_exists(tokenId), "ERC721: invalid token ID");
        return string(abi.encodePacked(_baseTokenURI, _toString(tokenId), ".json"));
    }

    function approve(address to, uint256 tokenId) public {
        address owner = ownerOf(tokenId);
        require(to != owner, "ERC721: approval to owner");
        require(
            msg.sender == owner || isApprovedForAll(owner, msg.sender),
            "ERC721: not authorized"
        );

        _tokenApprovals[tokenId] = to;
        emit Approval(owner, to, tokenId);
    }

    function getApproved(uint256 tokenId) public view returns (address) {
        require(_exists(tokenId), "ERC721: invalid token ID");
        return _tokenApprovals[tokenId];
    }

    function setApprovalForAll(address operator, bool approved) public {
        require(operator != msg.sender, "ERC721: approve to caller");
        _operatorApprovals[msg.sender][operator] = approved;
        emit ApprovalForAll(msg.sender, operator, approved);
    }

    function isApprovedForAll(address owner, address operator) public view returns (bool) {
        return _operatorApprovals[owner][operator];
    }

    function transferFrom(address from, address to, uint256 tokenId) public {
        require(_isApprovedOrOwner(msg.sender, tokenId), "ERC721: not authorized");
        _transfer(from, to, tokenId);
    }

    function safeTransferFrom(address from, address to, uint256 tokenId) public {
        safeTransferFrom(from, to, tokenId, "");
    }

    function safeTransferFrom(
        address from,
        address to,
        uint256 tokenId,
        bytes memory data
    ) public {
        require(_isApprovedOrOwner(msg.sender, tokenId), "ERC721: not authorized");
        _safeTransfer(from, to, tokenId, data);
    }

    // Публичная функция минтинга
    function mint(address to) public returns (uint256) {
        uint256 tokenId = _tokenIdCounter;
        _tokenIdCounter++;
        _safeMint(to, tokenId);
        return tokenId;
    }

    // Внутренние функции
    function _exists(uint256 tokenId) internal view returns (bool) {
        return _owners[tokenId] != address(0);
    }

    function _isApprovedOrOwner(address spender, uint256 tokenId) internal view returns (bool) {
        address owner = ownerOf(tokenId);
        return (spender == owner ||
                isApprovedForAll(owner, spender) ||
                getApproved(tokenId) == spender);
    }

    function _transfer(address from, address to, uint256 tokenId) internal {
        require(ownerOf(tokenId) == from, "ERC721: incorrect owner");
        require(to != address(0), "ERC721: zero address");

        // Очищаем approval
        delete _tokenApprovals[tokenId];

        _balances[from] -= 1;
        _balances[to] += 1;
        _owners[tokenId] = to;

        emit Transfer(from, to, tokenId);
    }

    function _safeTransfer(
        address from,
        address to,
        uint256 tokenId,
        bytes memory data
    ) internal {
        _transfer(from, to, tokenId);
        require(
            _checkOnERC721Received(from, to, tokenId, data),
            "ERC721: transfer to non ERC721Receiver"
        );
    }

    function _safeMint(address to, uint256 tokenId) internal {
        _mint(to, tokenId);
        require(
            _checkOnERC721Received(address(0), to, tokenId, ""),
            "ERC721: transfer to non ERC721Receiver"
        );
    }

    function _mint(address to, uint256 tokenId) internal {
        require(to != address(0), "ERC721: zero address");
        require(!_exists(tokenId), "ERC721: token exists");

        _balances[to] += 1;
        _owners[tokenId] = to;

        emit Transfer(address(0), to, tokenId);
    }

    function _burn(uint256 tokenId) internal {
        address owner = ownerOf(tokenId);

        delete _tokenApprovals[tokenId];
        _balances[owner] -= 1;
        delete _owners[tokenId];

        emit Transfer(owner, address(0), tokenId);
    }

    function _checkOnERC721Received(
        address from,
        address to,
        uint256 tokenId,
        bytes memory data
    ) private returns (bool) {
        if (to.code.length > 0) {
            try IERC721Receiver(to).onERC721Received(msg.sender, from, tokenId, data) returns (bytes4 retval) {
                return retval == IERC721Receiver.onERC721Received.selector;
            } catch {
                return false;
            }
        }
        return true;
    }

    function _toString(uint256 value) internal pure returns (string memory) {
        if (value == 0) return "0";
        uint256 temp = value;
        uint256 digits;
        while (temp != 0) {
            digits++;
            temp /= 10;
        }
        bytes memory buffer = new bytes(digits);
        while (value != 0) {
            digits -= 1;
            buffer[digits] = bytes1(uint8(48 + uint256(value % 10)));
            value /= 10;
        }
        return string(buffer);
    }
}

interface IERC721Receiver {
    function onERC721Received(
        address operator,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external returns (bytes4);
}
```

## Метаданные NFT

### Структура метаданных (JSON)

```json
{
    "name": "My NFT #1",
    "description": "Уникальный цифровой артефакт",
    "image": "ipfs://QmXxx.../1.png",
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
            "trait_type": "Power",
            "value": 95,
            "display_type": "number"
        }
    ]
}
```

### Хранение метаданных

1. **IPFS** — децентрализованное хранилище (рекомендуется)
2. **Arweave** — постоянное хранилище
3. **Централизованный сервер** — проще, но рискованно

## Использование OpenZeppelin

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721Enumerable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MyNFTCollection is ERC721, ERC721URIStorage, ERC721Enumerable, Ownable {
    uint256 private _tokenIdCounter;
    uint256 public maxSupply = 10000;
    uint256 public mintPrice = 0.05 ether;

    constructor(address initialOwner)
        ERC721("MyCollection", "MYC")
        Ownable(initialOwner)
    {}

    function mint(string memory uri) public payable returns (uint256) {
        require(_tokenIdCounter < maxSupply, "Max supply reached");
        require(msg.value >= mintPrice, "Insufficient payment");

        uint256 tokenId = _tokenIdCounter;
        _tokenIdCounter++;

        _safeMint(msg.sender, tokenId);
        _setTokenURI(tokenId, uri);

        return tokenId;
    }

    function withdraw() public onlyOwner {
        payable(owner()).transfer(address(this).balance);
    }

    // Обязательные переопределения
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

    function tokenURI(uint256 tokenId)
        public view
        override(ERC721, ERC721URIStorage)
        returns (string memory)
    {
        return super.tokenURI(tokenId);
    }

    function supportsInterface(bytes4 interfaceId)
        public view
        override(ERC721, ERC721URIStorage, ERC721Enumerable)
        returns (bool)
    {
        return super.supportsInterface(interfaceId);
    }
}
```

## Расширения ERC-721

### ERC721Enumerable

Позволяет перечислять все токены владельца:

```solidity
// Получить ID токена по индексу у владельца
function tokenOfOwnerByIndex(address owner, uint256 index) returns (uint256);

// Получить ID токена по глобальному индексу
function tokenByIndex(uint256 index) returns (uint256);
```

### ERC721Royalty (ERC-2981)

Добавляет информацию о роялти:

```solidity
import "@openzeppelin/contracts/token/common/ERC2981.sol";

contract NFTWithRoyalty is ERC721, ERC2981 {
    constructor() ERC721("NFT", "NFT") {
        // 5% роялти создателю
        _setDefaultRoyalty(msg.sender, 500);
    }
}
```

## Популярные NFT коллекции

| Коллекция | Токенов | Особенности |
|-----------|---------|-------------|
| CryptoPunks | 10,000 | Первые NFT, on-chain |
| Bored Ape Yacht Club | 10,000 | Утилити, комьюнити |
| Azuki | 10,000 | Anime стиль |
| Art Blocks | Varies | Генеративное искусство |

## Безопасность NFT

### Защита от реентерабельности

```solidity
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract SecureNFT is ERC721, ReentrancyGuard {
    function mint() public payable nonReentrant {
        // Безопасный минтинг
    }
}
```

### Проверка владельца

```solidity
modifier onlyTokenOwner(uint256 tokenId) {
    require(ownerOf(tokenId) == msg.sender, "Not token owner");
    _;
}
```

## Дополнительные ресурсы

- [EIP-721 Specification](https://eips.ethereum.org/EIPS/eip-721)
- [OpenZeppelin ERC721](https://docs.openzeppelin.com/contracts/5.x/erc721)
- [NFT Metadata Standards](https://docs.opensea.io/docs/metadata-standards)
