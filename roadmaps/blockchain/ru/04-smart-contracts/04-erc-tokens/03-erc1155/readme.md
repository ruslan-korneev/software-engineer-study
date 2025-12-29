# ERC-1155

## Введение

**ERC-1155** — это мульти-токен стандарт, разработанный командой Enjin в 2018 году. Он объединяет возможности ERC-20 (взаимозаменяемые токены) и ERC-721 (NFT) в одном контракте, обеспечивая высокую эффективность и гибкость.

## Ключевые особенности

### Преимущества ERC-1155

1. **Один контракт для всего** — взаимозаменяемые и уникальные токены в одном месте
2. **Batch операции** — перевод множества токенов за одну транзакцию
3. **Экономия газа** — до 90% экономии при массовых операциях
4. **Атомарные свопы** — обмен несколькими токенами одновременно
5. **Полу-взаимозаменяемость** — токены с ограниченным тиражом

### Сравнение с другими стандартами

| Операция | ERC-20 | ERC-721 | ERC-1155 |
|----------|--------|---------|----------|
| Перевод 1 токена | ~50k gas | ~80k gas | ~50k gas |
| Перевод 10 токенов | ~500k gas | ~800k gas | ~70k gas |
| Один контракт на проект | Нет | Нет | Да |
| Batch transfer | Нет | Нет | Да |

## Интерфейс ERC-1155

### Основной интерфейс

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IERC1155 {
    // События
    event TransferSingle(
        address indexed operator,
        address indexed from,
        address indexed to,
        uint256 id,
        uint256 value
    );

    event TransferBatch(
        address indexed operator,
        address indexed from,
        address indexed to,
        uint256[] ids,
        uint256[] values
    );

    event ApprovalForAll(
        address indexed account,
        address indexed operator,
        bool approved
    );

    event URI(string value, uint256 indexed id);

    // Баланс конкретного токена у владельца
    function balanceOf(address account, uint256 id) external view returns (uint256);

    // Балансы нескольких токенов у нескольких владельцев
    function balanceOfBatch(
        address[] calldata accounts,
        uint256[] calldata ids
    ) external view returns (uint256[] memory);

    // Одобрение оператора на все токены
    function setApprovalForAll(address operator, bool approved) external;

    // Проверка одобрения
    function isApprovedForAll(address account, address operator) external view returns (bool);

    // Безопасный перевод одного токена
    function safeTransferFrom(
        address from,
        address to,
        uint256 id,
        uint256 amount,
        bytes calldata data
    ) external;

    // Безопасный batch перевод
    function safeBatchTransferFrom(
        address from,
        address to,
        uint256[] calldata ids,
        uint256[] calldata amounts,
        bytes calldata data
    ) external;
}
```

### Интерфейс метаданных

```solidity
interface IERC1155MetadataURI is IERC1155 {
    // Возвращает URI для метаданных токена
    function uri(uint256 id) external view returns (string memory);
}
```

### Интерфейс получателя

```solidity
interface IERC1155Receiver {
    function onERC1155Received(
        address operator,
        address from,
        uint256 id,
        uint256 value,
        bytes calldata data
    ) external returns (bytes4);

    function onERC1155BatchReceived(
        address operator,
        address from,
        uint256[] calldata ids,
        uint256[] calldata values,
        bytes calldata data
    ) external returns (bytes4);
}
```

## Полная реализация ERC-1155

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract MyMultiToken {
    // Маппинг: id токена => (владелец => баланс)
    mapping(uint256 => mapping(address => uint256)) private _balances;

    // Маппинг одобрений операторов
    mapping(address => mapping(address => bool)) private _operatorApprovals;

    // Базовый URI для метаданных
    string private _uri;

    // События
    event TransferSingle(
        address indexed operator,
        address indexed from,
        address indexed to,
        uint256 id,
        uint256 value
    );

    event TransferBatch(
        address indexed operator,
        address indexed from,
        address indexed to,
        uint256[] ids,
        uint256[] values
    );

    event ApprovalForAll(address indexed account, address indexed operator, bool approved);
    event URI(string value, uint256 indexed id);

    constructor(string memory uri_) {
        _uri = uri_;
    }

    function uri(uint256) public view returns (string memory) {
        return _uri;
    }

    function balanceOf(address account, uint256 id) public view returns (uint256) {
        require(account != address(0), "ERC1155: zero address");
        return _balances[id][account];
    }

    function balanceOfBatch(
        address[] memory accounts,
        uint256[] memory ids
    ) public view returns (uint256[] memory) {
        require(accounts.length == ids.length, "ERC1155: length mismatch");

        uint256[] memory batchBalances = new uint256[](accounts.length);
        for (uint256 i = 0; i < accounts.length; ++i) {
            batchBalances[i] = balanceOf(accounts[i], ids[i]);
        }
        return batchBalances;
    }

    function setApprovalForAll(address operator, bool approved) public {
        require(msg.sender != operator, "ERC1155: self approval");
        _operatorApprovals[msg.sender][operator] = approved;
        emit ApprovalForAll(msg.sender, operator, approved);
    }

    function isApprovedForAll(address account, address operator) public view returns (bool) {
        return _operatorApprovals[account][operator];
    }

    function safeTransferFrom(
        address from,
        address to,
        uint256 id,
        uint256 amount,
        bytes memory data
    ) public {
        require(
            from == msg.sender || isApprovedForAll(from, msg.sender),
            "ERC1155: not authorized"
        );
        _safeTransferFrom(from, to, id, amount, data);
    }

    function safeBatchTransferFrom(
        address from,
        address to,
        uint256[] memory ids,
        uint256[] memory amounts,
        bytes memory data
    ) public {
        require(
            from == msg.sender || isApprovedForAll(from, msg.sender),
            "ERC1155: not authorized"
        );
        _safeBatchTransferFrom(from, to, ids, amounts, data);
    }

    // Внутренние функции
    function _safeTransferFrom(
        address from,
        address to,
        uint256 id,
        uint256 amount,
        bytes memory data
    ) internal {
        require(to != address(0), "ERC1155: zero address");

        uint256 fromBalance = _balances[id][from];
        require(fromBalance >= amount, "ERC1155: insufficient balance");

        unchecked {
            _balances[id][from] = fromBalance - amount;
        }
        _balances[id][to] += amount;

        emit TransferSingle(msg.sender, from, to, id, amount);

        _doSafeTransferAcceptanceCheck(msg.sender, from, to, id, amount, data);
    }

    function _safeBatchTransferFrom(
        address from,
        address to,
        uint256[] memory ids,
        uint256[] memory amounts,
        bytes memory data
    ) internal {
        require(ids.length == amounts.length, "ERC1155: length mismatch");
        require(to != address(0), "ERC1155: zero address");

        for (uint256 i = 0; i < ids.length; ++i) {
            uint256 id = ids[i];
            uint256 amount = amounts[i];

            uint256 fromBalance = _balances[id][from];
            require(fromBalance >= amount, "ERC1155: insufficient balance");

            unchecked {
                _balances[id][from] = fromBalance - amount;
            }
            _balances[id][to] += amount;
        }

        emit TransferBatch(msg.sender, from, to, ids, amounts);

        _doSafeBatchTransferAcceptanceCheck(msg.sender, from, to, ids, amounts, data);
    }

    function _mint(
        address to,
        uint256 id,
        uint256 amount,
        bytes memory data
    ) internal {
        require(to != address(0), "ERC1155: zero address");

        _balances[id][to] += amount;
        emit TransferSingle(msg.sender, address(0), to, id, amount);

        _doSafeTransferAcceptanceCheck(msg.sender, address(0), to, id, amount, data);
    }

    function _mintBatch(
        address to,
        uint256[] memory ids,
        uint256[] memory amounts,
        bytes memory data
    ) internal {
        require(to != address(0), "ERC1155: zero address");
        require(ids.length == amounts.length, "ERC1155: length mismatch");

        for (uint256 i = 0; i < ids.length; ++i) {
            _balances[ids[i]][to] += amounts[i];
        }

        emit TransferBatch(msg.sender, address(0), to, ids, amounts);

        _doSafeBatchTransferAcceptanceCheck(msg.sender, address(0), to, ids, amounts, data);
    }

    function _burn(address from, uint256 id, uint256 amount) internal {
        require(from != address(0), "ERC1155: zero address");

        uint256 fromBalance = _balances[id][from];
        require(fromBalance >= amount, "ERC1155: insufficient balance");

        unchecked {
            _balances[id][from] = fromBalance - amount;
        }

        emit TransferSingle(msg.sender, from, address(0), id, amount);
    }

    function _doSafeTransferAcceptanceCheck(
        address operator,
        address from,
        address to,
        uint256 id,
        uint256 amount,
        bytes memory data
    ) private {
        if (to.code.length > 0) {
            try IERC1155Receiver(to).onERC1155Received(operator, from, id, amount, data) returns (bytes4 response) {
                if (response != IERC1155Receiver.onERC1155Received.selector) {
                    revert("ERC1155: rejected tokens");
                }
            } catch {
                revert("ERC1155: non-receiver");
            }
        }
    }

    function _doSafeBatchTransferAcceptanceCheck(
        address operator,
        address from,
        address to,
        uint256[] memory ids,
        uint256[] memory amounts,
        bytes memory data
    ) private {
        if (to.code.length > 0) {
            try IERC1155Receiver(to).onERC1155BatchReceived(operator, from, ids, amounts, data) returns (bytes4 response) {
                if (response != IERC1155Receiver.onERC1155BatchReceived.selector) {
                    revert("ERC1155: rejected tokens");
                }
            } catch {
                revert("ERC1155: non-receiver");
            }
        }
    }
}

interface IERC1155Receiver {
    function onERC1155Received(
        address operator,
        address from,
        uint256 id,
        uint256 value,
        bytes calldata data
    ) external returns (bytes4);

    function onERC1155BatchReceived(
        address operator,
        address from,
        uint256[] calldata ids,
        uint256[] calldata values,
        bytes calldata data
    ) external returns (bytes4);
}
```

## Использование OpenZeppelin

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";
import "@openzeppelin/contracts/token/ERC1155/extensions/ERC1155Burnable.sol";
import "@openzeppelin/contracts/token/ERC1155/extensions/ERC1155Supply.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract GameItems is ERC1155, ERC1155Burnable, ERC1155Supply, Ownable {
    // ID токенов
    uint256 public constant GOLD = 0;
    uint256 public constant SILVER = 1;
    uint256 public constant SWORD = 2;
    uint256 public constant SHIELD = 3;
    uint256 public constant LEGENDARY_SWORD = 4;

    // Максимальное количество для каждого типа
    mapping(uint256 => uint256) public maxSupply;

    constructor(address initialOwner)
        ERC1155("https://game.example/api/item/{id}.json")
        Ownable(initialOwner)
    {
        // Взаимозаменяемые токены (валюта)
        maxSupply[GOLD] = type(uint256).max;
        maxSupply[SILVER] = type(uint256).max;

        // Обычные предметы (ограниченный тираж)
        maxSupply[SWORD] = 10000;
        maxSupply[SHIELD] = 10000;

        // Уникальный предмет (NFT)
        maxSupply[LEGENDARY_SWORD] = 1;
    }

    function setURI(string memory newuri) public onlyOwner {
        _setURI(newuri);
    }

    function mint(
        address account,
        uint256 id,
        uint256 amount,
        bytes memory data
    ) public onlyOwner {
        require(
            totalSupply(id) + amount <= maxSupply[id],
            "Max supply exceeded"
        );
        _mint(account, id, amount, data);
    }

    function mintBatch(
        address to,
        uint256[] memory ids,
        uint256[] memory amounts,
        bytes memory data
    ) public onlyOwner {
        for (uint256 i = 0; i < ids.length; i++) {
            require(
                totalSupply(ids[i]) + amounts[i] <= maxSupply[ids[i]],
                "Max supply exceeded"
            );
        }
        _mintBatch(to, ids, amounts, data);
    }

    // Переопределение для ERC1155Supply
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

## Gaming Use Cases

### Инвентарь игрока

```solidity
contract GameInventory is ERC1155 {
    // Структура предмета
    struct ItemType {
        string name;
        uint256 maxSupply;
        bool isFungible;
        uint256 power;
    }

    mapping(uint256 => ItemType) public itemTypes;
    uint256 public nextItemId;

    // Создание нового типа предмета
    function createItemType(
        string memory name,
        uint256 maxSupply,
        bool isFungible,
        uint256 power
    ) external returns (uint256) {
        uint256 itemId = nextItemId++;
        itemTypes[itemId] = ItemType(name, maxSupply, isFungible, power);
        return itemId;
    }

    // Награда игроку за квест
    function rewardPlayer(
        address player,
        uint256[] memory itemIds,
        uint256[] memory amounts
    ) external {
        _mintBatch(player, itemIds, amounts, "");
    }

    // Крафтинг предмета
    function craft(
        uint256[] memory ingredientIds,
        uint256[] memory ingredientAmounts,
        uint256 resultId
    ) external {
        // Сжигаем ингредиенты
        _burnBatch(msg.sender, ingredientIds, ingredientAmounts);
        // Создаём результат
        _mint(msg.sender, resultId, 1, "");
    }
}
```

### Торговая площадка

```solidity
contract GameMarketplace {
    IERC1155 public gameItems;

    struct Listing {
        address seller;
        uint256 tokenId;
        uint256 amount;
        uint256 pricePerItem;
    }

    mapping(uint256 => Listing) public listings;
    uint256 public nextListingId;

    function listItems(
        uint256 tokenId,
        uint256 amount,
        uint256 pricePerItem
    ) external returns (uint256) {
        gameItems.safeTransferFrom(
            msg.sender,
            address(this),
            tokenId,
            amount,
            ""
        );

        uint256 listingId = nextListingId++;
        listings[listingId] = Listing(msg.sender, tokenId, amount, pricePerItem);
        return listingId;
    }

    function buyItems(uint256 listingId, uint256 amount) external payable {
        Listing storage listing = listings[listingId];
        require(listing.amount >= amount, "Not enough items");
        require(msg.value >= listing.pricePerItem * amount, "Insufficient payment");

        listing.amount -= amount;

        gameItems.safeTransferFrom(
            address(this),
            msg.sender,
            listing.tokenId,
            amount,
            ""
        );

        payable(listing.seller).transfer(msg.value);
    }
}
```

## Метаданные ERC-1155

### Структура JSON

```json
{
    "name": "Gold Coin",
    "description": "In-game currency",
    "image": "https://game.example/images/gold.png",
    "properties": {
        "type": "currency",
        "rarity": "common"
    }
}
```

### URI с подстановкой ID

Стандарт поддерживает `{id}` в URI, который заменяется на hex-представление ID:

```solidity
// URI: "https://api.example/token/{id}.json"
// Для токена 1: "https://api.example/token/0000000000000000000000000000000000000000000000000000000000000001.json"
```

## Сравнение стандартов: когда что использовать

| Сценарий | Лучший выбор |
|----------|--------------|
| Криптовалюта | ERC-20 |
| Уникальное искусство | ERC-721 |
| Игровая валюта + предметы | ERC-1155 |
| Коллекция с разными тиражами | ERC-1155 |
| Билеты на события | ERC-1155 |
| Членские карты | ERC-721 или ERC-1155 |

## Безопасность ERC-1155

### Проверка получателя

```solidity
// Контракт-получатель должен реализовать интерфейс
contract SafeReceiver is IERC1155Receiver {
    function onERC1155Received(
        address,
        address,
        uint256,
        uint256,
        bytes calldata
    ) external pure returns (bytes4) {
        return this.onERC1155Received.selector;
    }

    function onERC1155BatchReceived(
        address,
        address,
        uint256[] calldata,
        uint256[] calldata,
        bytes calldata
    ) external pure returns (bytes4) {
        return this.onERC1155BatchReceived.selector;
    }
}
```

## Дополнительные ресурсы

- [EIP-1155 Specification](https://eips.ethereum.org/EIPS/eip-1155)
- [OpenZeppelin ERC1155](https://docs.openzeppelin.com/contracts/5.x/erc1155)
- [Enjin Documentation](https://docs.enjin.io/)
