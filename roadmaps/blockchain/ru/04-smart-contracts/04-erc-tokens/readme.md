# ERC токены

## Что такое ERC?

**ERC (Ethereum Request for Comments)** — это стандарты, описывающие правила реализации токенов и смарт-контрактов в сети Ethereum. Название происходит от системы предложений улучшений, где разработчики публикуют свои идеи для обсуждения сообществом.

Каждый ERC проходит через процесс EIP (Ethereum Improvement Proposal):
1. **Draft** — черновик предложения
2. **Review** — обсуждение сообществом
3. **Final** — принятый стандарт

## Зачем нужны стандарты токенов?

### Проблема без стандартов

До появления стандартов каждый разработчик создавал токены по-своему:
- Разные названия функций (`send`, `transfer`, `move`)
- Разные параметры и возвращаемые значения
- Несовместимость между кошельками и биржами

### Преимущества стандартизации

1. **Совместимость** — любой кошелёк может работать с любым токеном стандарта
2. **Безопасность** — проверенные паттерны снижают риск уязвимостей
3. **Интеграция** — биржи и DeFi протоколы легко добавляют новые токены
4. **Предсказуемость** — разработчики знают, чего ожидать от контракта

## Основные стандарты токенов

### ERC-20: Взаимозаменяемые токены

Самый популярный стандарт для создания криптовалют и utility токенов.

**Характеристики:**
- Все токены идентичны и взаимозаменяемы
- 1 USDT = 1 USDT (любой токен равен другому)
- Делимые (можно отправить 0.5 токена)

**Примеры:** USDT, USDC, LINK, UNI, SHIB

### ERC-721: Невзаимозаменяемые токены (NFT)

Стандарт для уникальных цифровых активов.

**Характеристики:**
- Каждый токен уникален (имеет свой ID)
- Неделимые (нельзя отправить 0.5 NFT)
- Привязаны к метаданным (изображение, описание)

**Примеры:** CryptoPunks, Bored Ape Yacht Club, цифровое искусство

### ERC-1155: Мульти-токен стандарт

Гибридный стандарт, объединяющий ERC-20 и ERC-721.

**Характеристики:**
- Один контракт для разных типов токенов
- Поддержка batch операций
- Эффективнее по газу

**Примеры:** Игровые предметы, коллекции NFT

## Сравнительная таблица

| Характеристика | ERC-20 | ERC-721 | ERC-1155 |
|---------------|--------|---------|----------|
| Взаимозаменяемость | Да | Нет | Оба варианта |
| Уникальность | Нет | Да | Опционально |
| Делимость | Да | Нет | Опционально |
| Batch операции | Нет | Нет | Да |
| Газ на трансфер | Низкий | Средний | Оптимизирован |
| Метаданные | Нет | Да | Да |

## Другие важные стандарты

### ERC-777: Улучшенный ERC-20

Расширение ERC-20 с дополнительными возможностями:
- Хуки при отправке/получении токенов
- Операторы (делегирование прав)
- Обратная совместимость с ERC-20

```solidity
interface IERC777 {
    function send(address recipient, uint256 amount, bytes calldata data) external;
    function operatorSend(
        address sender,
        address recipient,
        uint256 amount,
        bytes calldata data,
        bytes calldata operatorData
    ) external;
}
```

### ERC-4626: Токенизированные хранилища

Стандарт для yield-bearing токенов (токенов с доходностью):

```solidity
interface IERC4626 {
    function deposit(uint256 assets, address receiver) external returns (uint256 shares);
    function withdraw(uint256 assets, address receiver, address owner) external returns (uint256 shares);
    function totalAssets() external view returns (uint256);
}
```

**Примеры:** Aave aTokens, Compound cTokens

### ERC-2981: Роялти для NFT

Стандарт для выплаты роялти создателям NFT при перепродаже:

```solidity
interface IERC2981 {
    function royaltyInfo(uint256 tokenId, uint256 salePrice)
        external view returns (address receiver, uint256 royaltyAmount);
}
```

## Экосистема и инструменты

### OpenZeppelin Contracts

Библиотека проверенных реализаций стандартов:

```bash
npm install @openzeppelin/contracts
```

```solidity
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";
```

### Площадки и маркетплейсы

- **ERC-20:** Uniswap, SushiSwap, биржи
- **ERC-721:** OpenSea, Rarible, Foundation
- **ERC-1155:** OpenSea, игровые маркетплейсы

## Безопасность токенов

### Общие уязвимости

1. **Reentrancy** — повторный вход в функцию до завершения
2. **Integer Overflow** — переполнение чисел (решено в Solidity 0.8+)
3. **Approve Race Condition** — состояние гонки при изменении allowance

### Лучшие практики

```solidity
// Используйте проверенные библиотеки
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

// Добавляйте модификаторы доступа
import "@openzeppelin/contracts/access/Ownable.sol";

// Используйте Pausable для экстренных случаев
import "@openzeppelin/contracts/security/Pausable.sol";
```

## Процесс создания токена

1. **Выбор стандарта** — определите тип токена
2. **Написание контракта** — используйте OpenZeppelin
3. **Тестирование** — unit-тесты и тестнет
4. **Аудит** — проверка безопасности (для серьёзных проектов)
5. **Деплой** — размещение в mainnet
6. **Верификация** — публикация исходного кода на Etherscan

## Выбор стандарта для проекта

| Сценарий | Рекомендуемый стандарт |
|----------|----------------------|
| Криптовалюта, utility токен | ERC-20 |
| Цифровое искусство, уникальные предметы | ERC-721 |
| Игровые предметы (разные типы) | ERC-1155 |
| Токены с доходностью | ERC-4626 |
| NFT с роялти | ERC-721 + ERC-2981 |

## Дополнительные ресурсы

- [EIP официальный репозиторий](https://eips.ethereum.org/)
- [OpenZeppelin Docs](https://docs.openzeppelin.com/)
- [Ethereum.org Token Standards](https://ethereum.org/en/developers/docs/standards/tokens/)
