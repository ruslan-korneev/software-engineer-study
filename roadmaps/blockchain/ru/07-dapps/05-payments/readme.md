# Платежи в блокчейне

## Введение

Интеграция криптовалютных платежей в веб-приложения открывает новые возможности для бизнеса: глобальные транзакции без посредников, низкие комиссии, прозрачность и безопасность. В этом разделе рассмотрим различные подходы к приему криптовалютных платежей.

## Способы приема криптоплатежей

### 1. Платежные шлюзы

Платежные шлюзы упрощают интеграцию криптоплатежей, предоставляя готовую инфраструктуру.

#### BitPay

BitPay — один из старейших и надежных платежных процессоров для криптовалют.

**Преимущества:**
- Поддержка Bitcoin, Ethereum и других криптовалют
- Автоматическая конвертация в фиат
- Защита от волатильности
- REST API для интеграции

**Пример интеграции с BitPay (TypeScript):**

```typescript
import axios from 'axios';

interface BitPayInvoice {
  id: string;
  url: string;
  status: string;
  price: number;
  currency: string;
  expirationTime: number;
}

class BitPayService {
  private apiUrl: string;
  private apiToken: string;

  constructor(apiToken: string, testnet: boolean = false) {
    this.apiUrl = testnet
      ? 'https://test.bitpay.com/api/v2'
      : 'https://bitpay.com/api/v2';
    this.apiToken = apiToken;
  }

  async createInvoice(
    price: number,
    currency: string,
    orderId: string,
    buyerEmail?: string
  ): Promise<BitPayInvoice> {
    const response = await axios.post(
      `${this.apiUrl}/invoices`,
      {
        price,
        currency,
        orderId,
        buyer: buyerEmail ? { email: buyerEmail } : undefined,
        notificationURL: 'https://your-site.com/api/bitpay/webhook',
        redirectURL: 'https://your-site.com/payment/success',
      },
      {
        headers: {
          'Content-Type': 'application/json',
          'X-Accept-Version': '2.0.0',
          Authorization: `Bearer ${this.apiToken}`,
        },
      }
    );

    return response.data.data;
  }

  async getInvoice(invoiceId: string): Promise<BitPayInvoice> {
    const response = await axios.get(
      `${this.apiUrl}/invoices/${invoiceId}`,
      {
        headers: {
          Authorization: `Bearer ${this.apiToken}`,
        },
      }
    );

    return response.data.data;
  }
}

// Использование
const bitpay = new BitPayService('your-api-token', true);

async function processPayment() {
  const invoice = await bitpay.createInvoice(
    100,
    'USD',
    'ORDER-123',
    'customer@email.com'
  );

  console.log(`Платежная ссылка: ${invoice.url}`);
  console.log(`ID инвойса: ${invoice.id}`);
}
```

#### Coinbase Commerce

Coinbase Commerce — платежное решение от крупнейшей американской криптобиржи.

**Пример интеграции (TypeScript):**

```typescript
import crypto from 'crypto';

interface CoinbaseCharge {
  id: string;
  code: string;
  hosted_url: string;
  pricing: {
    local: { amount: string; currency: string };
    ethereum: { amount: string; currency: string };
    bitcoin: { amount: string; currency: string };
  };
  addresses: {
    ethereum: string;
    bitcoin: string;
  };
}

class CoinbaseCommerceService {
  private apiKey: string;
  private webhookSecret: string;
  private baseUrl = 'https://api.commerce.coinbase.com';

  constructor(apiKey: string, webhookSecret: string) {
    this.apiKey = apiKey;
    this.webhookSecret = webhookSecret;
  }

  async createCharge(
    name: string,
    description: string,
    amount: number,
    currency: string,
    metadata?: Record<string, string>
  ): Promise<CoinbaseCharge> {
    const response = await fetch(`${this.baseUrl}/charges`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-CC-Api-Key': this.apiKey,
        'X-CC-Version': '2018-03-22',
      },
      body: JSON.stringify({
        name,
        description,
        pricing_type: 'fixed_price',
        local_price: {
          amount: amount.toString(),
          currency,
        },
        metadata,
        redirect_url: 'https://your-site.com/payment/success',
        cancel_url: 'https://your-site.com/payment/cancel',
      }),
    });

    const data = await response.json();
    return data.data;
  }

  // Верификация webhook от Coinbase
  verifyWebhook(payload: string, signature: string): boolean {
    const computedSignature = crypto
      .createHmac('sha256', this.webhookSecret)
      .update(payload)
      .digest('hex');

    return crypto.timingSafeEqual(
      Buffer.from(signature),
      Buffer.from(computedSignature)
    );
  }

  // Обработчик webhook
  handleWebhook(payload: string, signature: string): void {
    if (!this.verifyWebhook(payload, signature)) {
      throw new Error('Invalid webhook signature');
    }

    const event = JSON.parse(payload);

    switch (event.type) {
      case 'charge:confirmed':
        console.log('Платеж подтвержден:', event.data.code);
        // Обновить статус заказа в БД
        break;
      case 'charge:failed':
        console.log('Платеж не прошел:', event.data.code);
        break;
      case 'charge:pending':
        console.log('Платеж в ожидании:', event.data.code);
        break;
    }
  }
}
```

### 2. Прямые платежи на кошелек

Для полного контроля можно принимать платежи напрямую без посредников.

**Пример прямого приема платежей (TypeScript + ethers.js):**

```typescript
import { ethers } from 'ethers';

interface PaymentRequest {
  id: string;
  amount: bigint;
  recipientAddress: string;
  createdAt: Date;
  expiresAt: Date;
  status: 'pending' | 'confirmed' | 'expired';
}

class DirectPaymentService {
  private provider: ethers.Provider;
  private merchantWallet: string;
  private payments: Map<string, PaymentRequest> = new Map();

  constructor(rpcUrl: string, merchantWallet: string) {
    this.provider = new ethers.JsonRpcProvider(rpcUrl);
    this.merchantWallet = merchantWallet;
  }

  // Создание запроса на оплату
  createPaymentRequest(
    amountInEth: string,
    expirationMinutes: number = 30
  ): PaymentRequest {
    const paymentId = crypto.randomUUID();
    const now = new Date();

    const payment: PaymentRequest = {
      id: paymentId,
      amount: ethers.parseEther(amountInEth),
      recipientAddress: this.merchantWallet,
      createdAt: now,
      expiresAt: new Date(now.getTime() + expirationMinutes * 60000),
      status: 'pending',
    };

    this.payments.set(paymentId, payment);
    return payment;
  }

  // Генерация данных для QR-кода (EIP-681 формат)
  generatePaymentUri(paymentId: string): string {
    const payment = this.payments.get(paymentId);
    if (!payment) throw new Error('Payment not found');

    // EIP-681: ethereum:<address>@<chainId>?value=<amount>
    const amountInWei = payment.amount.toString();
    return `ethereum:${payment.recipientAddress}@1?value=${amountInWei}`;
  }

  // Мониторинг транзакций для подтверждения платежа
  async watchForPayment(
    paymentId: string,
    onConfirmed: (txHash: string) => void
  ): Promise<void> {
    const payment = this.payments.get(paymentId);
    if (!payment) throw new Error('Payment not found');

    const filter = {
      address: null,
      topics: [],
    };

    // Слушаем новые блоки
    this.provider.on('block', async (blockNumber) => {
      const block = await this.provider.getBlock(blockNumber, true);
      if (!block || !block.prefetchedTransactions) return;

      for (const tx of block.prefetchedTransactions) {
        // Проверяем, является ли транзакция платежом на наш кошелек
        if (
          tx.to?.toLowerCase() === payment.recipientAddress.toLowerCase() &&
          tx.value >= payment.amount
        ) {
          payment.status = 'confirmed';
          onConfirmed(tx.hash);
          this.provider.removeAllListeners('block');
          break;
        }
      }
    });
  }

  // Проверка транзакции по хешу
  async verifyTransaction(
    txHash: string,
    expectedAmount: bigint,
    requiredConfirmations: number = 12
  ): Promise<boolean> {
    const tx = await this.provider.getTransaction(txHash);
    if (!tx) return false;

    // Проверяем получателя
    if (tx.to?.toLowerCase() !== this.merchantWallet.toLowerCase()) {
      return false;
    }

    // Проверяем сумму
    if (tx.value < expectedAmount) {
      return false;
    }

    // Проверяем количество подтверждений
    const receipt = await tx.wait(requiredConfirmations);
    return receipt !== null && receipt.status === 1;
  }
}
```

### 3. QR-коды для платежей

QR-коды упрощают процесс оплаты для пользователей мобильных кошельков.

**Генерация QR-кода (TypeScript + qrcode):**

```typescript
import QRCode from 'qrcode';

interface QRPaymentData {
  address: string;
  amount?: string;
  chainId?: number;
  tokenAddress?: string; // Для ERC-20 токенов
  message?: string;
}

class PaymentQRGenerator {
  // Генерация URI для ETH платежа (EIP-681)
  generateEthUri(data: QRPaymentData): string {
    let uri = `ethereum:${data.address}`;

    if (data.chainId) {
      uri += `@${data.chainId}`;
    }

    const params: string[] = [];

    if (data.amount) {
      const amountWei = ethers.parseEther(data.amount).toString();
      params.push(`value=${amountWei}`);
    }

    if (params.length > 0) {
      uri += '?' + params.join('&');
    }

    return uri;
  }

  // Генерация URI для ERC-20 токена
  generateTokenUri(data: QRPaymentData): string {
    if (!data.tokenAddress) throw new Error('Token address required');

    // EIP-681 для токенов: ethereum:<token_address>/transfer?address=<recipient>&uint256=<amount>
    let uri = `ethereum:${data.tokenAddress}`;

    if (data.chainId) {
      uri += `@${data.chainId}`;
    }

    uri += `/transfer?address=${data.address}`;

    if (data.amount) {
      // Для USDT/USDC обычно 6 decimals, для DAI - 18
      const decimals = 6; // Настроить по токену
      const amountSmallestUnit = BigInt(parseFloat(data.amount) * 10 ** decimals);
      uri += `&uint256=${amountSmallestUnit}`;
    }

    return uri;
  }

  // Генерация QR-кода как Data URL
  async generateQRDataUrl(uri: string): Promise<string> {
    return await QRCode.toDataURL(uri, {
      width: 300,
      margin: 2,
      color: {
        dark: '#000000',
        light: '#ffffff',
      },
    });
  }

  // Генерация QR как SVG строки
  async generateQRSvg(uri: string): Promise<string> {
    return await QRCode.toString(uri, {
      type: 'svg',
      width: 300,
    });
  }
}

// Пример использования
const qrGenerator = new PaymentQRGenerator();

// ETH платеж
const ethUri = qrGenerator.generateEthUri({
  address: '0x742d35Cc6634C0532925a3b844Bc9e7595f9c5dB',
  amount: '0.1',
  chainId: 1,
});

// USDC платеж
const usdcUri = qrGenerator.generateTokenUri({
  address: '0x742d35Cc6634C0532925a3b844Bc9e7595f9c5dB',
  tokenAddress: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48', // USDC на Ethereum
  amount: '100',
  chainId: 1,
});
```

## Платежи стейблкоинами

Стейблкоины (USDT, USDC, DAI) решают проблему волатильности криптовалют.

### Сравнение популярных стейблкоинов

| Стейблкоин | Тип | Decimals | Сети |
|------------|-----|----------|------|
| USDT | Централизованный | 6 | Ethereum, Tron, BSC, Polygon |
| USDC | Централизованный | 6 | Ethereum, Solana, Polygon, Arbitrum |
| DAI | Децентрализованный | 18 | Ethereum, Polygon, Arbitrum |

### Смарт-контракт для приема платежей стейблкоинами

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract StablecoinPayments is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // Поддерживаемые токены
    mapping(address => bool) public supportedTokens;

    // Информация о платеже
    struct Payment {
        address payer;
        address token;
        uint256 amount;
        bytes32 orderId;
        uint256 timestamp;
        bool refunded;
    }

    // Все платежи
    mapping(bytes32 => Payment) public payments;

    // События
    event PaymentReceived(
        bytes32 indexed paymentId,
        address indexed payer,
        address token,
        uint256 amount,
        bytes32 orderId
    );

    event PaymentRefunded(
        bytes32 indexed paymentId,
        address indexed recipient,
        uint256 amount
    );

    event TokenAdded(address token);
    event TokenRemoved(address token);

    constructor(address[] memory _tokens) Ownable(msg.sender) {
        for (uint i = 0; i < _tokens.length; i++) {
            supportedTokens[_tokens[i]] = true;
            emit TokenAdded(_tokens[i]);
        }
    }

    // Добавить поддерживаемый токен
    function addToken(address _token) external onlyOwner {
        require(!supportedTokens[_token], "Token already supported");
        supportedTokens[_token] = true;
        emit TokenAdded(_token);
    }

    // Удалить токен из поддерживаемых
    function removeToken(address _token) external onlyOwner {
        require(supportedTokens[_token], "Token not supported");
        supportedTokens[_token] = false;
        emit TokenRemoved(_token);
    }

    // Оплата стейблкоином
    function pay(
        address _token,
        uint256 _amount,
        bytes32 _orderId
    ) external nonReentrant returns (bytes32) {
        require(supportedTokens[_token], "Token not supported");
        require(_amount > 0, "Amount must be greater than 0");

        // Генерируем уникальный ID платежа
        bytes32 paymentId = keccak256(
            abi.encodePacked(msg.sender, _token, _amount, _orderId, block.timestamp)
        );

        require(payments[paymentId].timestamp == 0, "Payment already exists");

        // Переводим токены на контракт
        IERC20(_token).safeTransferFrom(msg.sender, address(this), _amount);

        // Сохраняем информацию о платеже
        payments[paymentId] = Payment({
            payer: msg.sender,
            token: _token,
            amount: _amount,
            orderId: _orderId,
            timestamp: block.timestamp,
            refunded: false
        });

        emit PaymentReceived(paymentId, msg.sender, _token, _amount, _orderId);

        return paymentId;
    }

    // Возврат платежа (только владелец)
    function refund(bytes32 _paymentId) external onlyOwner nonReentrant {
        Payment storage payment = payments[_paymentId];

        require(payment.timestamp != 0, "Payment not found");
        require(!payment.refunded, "Already refunded");

        payment.refunded = true;

        IERC20(payment.token).safeTransfer(payment.payer, payment.amount);

        emit PaymentRefunded(_paymentId, payment.payer, payment.amount);
    }

    // Вывод средств владельцем
    function withdraw(address _token, uint256 _amount) external onlyOwner {
        IERC20(_token).safeTransfer(owner(), _amount);
    }

    // Вывод всех средств конкретного токена
    function withdrawAll(address _token) external onlyOwner {
        uint256 balance = IERC20(_token).balanceOf(address(this));
        require(balance > 0, "No balance to withdraw");
        IERC20(_token).safeTransfer(owner(), balance);
    }

    // Получить информацию о платеже
    function getPayment(bytes32 _paymentId) external view returns (Payment memory) {
        return payments[_paymentId];
    }
}
```

### Клиентская интеграция для работы с контрактом

```typescript
import { ethers } from 'ethers';

const PAYMENT_CONTRACT_ABI = [
  'function pay(address _token, uint256 _amount, bytes32 _orderId) external returns (bytes32)',
  'function getPayment(bytes32 _paymentId) external view returns (tuple(address payer, address token, uint256 amount, bytes32 orderId, uint256 timestamp, bool refunded))',
  'event PaymentReceived(bytes32 indexed paymentId, address indexed payer, address token, uint256 amount, bytes32 orderId)',
];

const ERC20_ABI = [
  'function approve(address spender, uint256 amount) external returns (bool)',
  'function allowance(address owner, address spender) external view returns (uint256)',
  'function balanceOf(address account) external view returns (uint256)',
];

interface TokenConfig {
  address: string;
  decimals: number;
  symbol: string;
}

class StablecoinPaymentClient {
  private provider: ethers.Provider;
  private paymentContract: ethers.Contract;
  private contractAddress: string;

  // Адреса стейблкоинов в Ethereum Mainnet
  static readonly TOKENS: Record<string, TokenConfig> = {
    USDT: {
      address: '0xdAC17F958D2ee523a2206206994597C13D831ec7',
      decimals: 6,
      symbol: 'USDT',
    },
    USDC: {
      address: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48',
      decimals: 6,
      symbol: 'USDC',
    },
    DAI: {
      address: '0x6B175474E89094C44Da98b954EescdeCB5DB3A77F',
      decimals: 18,
      symbol: 'DAI',
    },
  };

  constructor(
    provider: ethers.Provider,
    contractAddress: string
  ) {
    this.provider = provider;
    this.contractAddress = contractAddress;
    this.paymentContract = new ethers.Contract(
      contractAddress,
      PAYMENT_CONTRACT_ABI,
      provider
    );
  }

  // Оплата стейблкоином
  async pay(
    signer: ethers.Signer,
    tokenSymbol: 'USDT' | 'USDC' | 'DAI',
    amount: string,
    orderId: string
  ): Promise<{ txHash: string; paymentId: string }> {
    const token = StablecoinPaymentClient.TOKENS[tokenSymbol];
    if (!token) throw new Error('Unsupported token');

    const tokenContract = new ethers.Contract(
      token.address,
      ERC20_ABI,
      signer
    );

    const amountInSmallestUnit = ethers.parseUnits(amount, token.decimals);
    const orderIdBytes = ethers.id(orderId); // keccak256 хеш

    // Проверяем баланс
    const signerAddress = await signer.getAddress();
    const balance = await tokenContract.balanceOf(signerAddress);
    if (balance < amountInSmallestUnit) {
      throw new Error(`Insufficient ${tokenSymbol} balance`);
    }

    // Проверяем и устанавливаем approve
    const allowance = await tokenContract.allowance(
      signerAddress,
      this.contractAddress
    );

    if (allowance < amountInSmallestUnit) {
      console.log('Approving tokens...');
      const approveTx = await tokenContract.approve(
        this.contractAddress,
        amountInSmallestUnit
      );
      await approveTx.wait();
      console.log('Tokens approved');
    }

    // Выполняем платеж
    const paymentContractWithSigner = this.paymentContract.connect(signer);
    const tx = await paymentContractWithSigner.pay(
      token.address,
      amountInSmallestUnit,
      orderIdBytes
    );

    const receipt = await tx.wait();

    // Извлекаем paymentId из события
    const event = receipt.logs.find(
      (log: any) => log.topics[0] === ethers.id('PaymentReceived(bytes32,address,address,uint256,bytes32)')
    );

    const paymentId = event?.topics[1] || '';

    return {
      txHash: receipt.hash,
      paymentId,
    };
  }

  // Подписка на события платежей
  onPaymentReceived(
    callback: (paymentId: string, payer: string, amount: bigint) => void
  ): void {
    this.paymentContract.on(
      'PaymentReceived',
      (paymentId, payer, token, amount, orderId) => {
        callback(paymentId, payer, amount);
      }
    );
  }

  // Проверка статуса платежа
  async getPaymentStatus(paymentId: string): Promise<{
    confirmed: boolean;
    refunded: boolean;
    amount: string;
    payer: string;
  }> {
    const payment = await this.paymentContract.getPayment(paymentId);

    return {
      confirmed: payment.timestamp > 0,
      refunded: payment.refunded,
      amount: ethers.formatUnits(payment.amount, 6), // Предполагаем 6 decimals
      payer: payment.payer,
    };
  }
}
```

## Верификация платежей

### Прослушивание событий блокчейна

```typescript
import { ethers } from 'ethers';
import { WebSocket } from 'ws';

interface PaymentWatcher {
  paymentId: string;
  expectedAmount: bigint;
  recipientAddress: string;
  tokenAddress?: string;
  callback: (confirmed: boolean, txHash?: string) => void;
}

class BlockchainPaymentVerifier {
  private provider: ethers.WebSocketProvider;
  private watchers: Map<string, PaymentWatcher> = new Map();

  constructor(wsRpcUrl: string) {
    this.provider = new ethers.WebSocketProvider(wsRpcUrl);
  }

  // Отслеживание ETH платежа
  watchEthPayment(
    paymentId: string,
    recipientAddress: string,
    expectedAmount: bigint,
    timeoutMs: number = 1800000, // 30 минут
    callback: (confirmed: boolean, txHash?: string) => void
  ): void {
    const watcher: PaymentWatcher = {
      paymentId,
      expectedAmount,
      recipientAddress,
      callback,
    };

    this.watchers.set(paymentId, watcher);

    // Слушаем pending транзакции для быстрого отклика
    this.provider.on('pending', async (txHash) => {
      try {
        const tx = await this.provider.getTransaction(txHash);
        if (!tx) return;

        if (
          tx.to?.toLowerCase() === recipientAddress.toLowerCase() &&
          tx.value >= expectedAmount
        ) {
          console.log(`Pending payment detected: ${txHash}`);

          // Ждем подтверждения
          const receipt = await tx.wait(1);
          if (receipt && receipt.status === 1) {
            watcher.callback(true, txHash);
            this.watchers.delete(paymentId);
          }
        }
      } catch (error) {
        // Транзакция могла быть заменена или отменена
      }
    });

    // Таймаут
    setTimeout(() => {
      if (this.watchers.has(paymentId)) {
        watcher.callback(false);
        this.watchers.delete(paymentId);
      }
    }, timeoutMs);
  }

  // Отслеживание ERC-20 платежа через события Transfer
  watchTokenPayment(
    paymentId: string,
    tokenAddress: string,
    recipientAddress: string,
    expectedAmount: bigint,
    timeoutMs: number = 1800000,
    callback: (confirmed: boolean, txHash?: string) => void
  ): void {
    const tokenContract = new ethers.Contract(
      tokenAddress,
      ['event Transfer(address indexed from, address indexed to, uint256 value)'],
      this.provider
    );

    const filter = tokenContract.filters.Transfer(null, recipientAddress);

    const listener = async (from: string, to: string, value: bigint, event: any) => {
      if (value >= expectedAmount) {
        const tx = await event.getTransaction();
        const receipt = await tx.wait(1);

        if (receipt && receipt.status === 1) {
          callback(true, tx.hash);
          tokenContract.off(filter, listener);
          this.watchers.delete(paymentId);
        }
      }
    };

    tokenContract.on(filter, listener);

    this.watchers.set(paymentId, {
      paymentId,
      expectedAmount,
      recipientAddress,
      tokenAddress,
      callback,
    });

    // Таймаут
    setTimeout(() => {
      if (this.watchers.has(paymentId)) {
        tokenContract.off(filter, listener);
        callback(false);
        this.watchers.delete(paymentId);
      }
    }, timeoutMs);
  }

  // Верификация уже совершенной транзакции
  async verifyTransaction(
    txHash: string,
    expectedRecipient: string,
    expectedAmount: bigint,
    tokenAddress?: string,
    requiredConfirmations: number = 12
  ): Promise<{ valid: boolean; confirmations: number }> {
    const receipt = await this.provider.getTransactionReceipt(txHash);
    if (!receipt || receipt.status !== 1) {
      return { valid: false, confirmations: 0 };
    }

    const currentBlock = await this.provider.getBlockNumber();
    const confirmations = currentBlock - receipt.blockNumber;

    if (tokenAddress) {
      // Верификация ERC-20 транзакции
      const transferTopic = ethers.id('Transfer(address,address,uint256)');
      const transferLog = receipt.logs.find(
        (log) =>
          log.address.toLowerCase() === tokenAddress.toLowerCase() &&
          log.topics[0] === transferTopic
      );

      if (!transferLog) {
        return { valid: false, confirmations };
      }

      const decodedData = ethers.AbiCoder.defaultAbiCoder().decode(
        ['uint256'],
        transferLog.data
      );
      const transferredAmount = decodedData[0];
      const toAddress = '0x' + transferLog.topics[2].slice(26);

      const valid =
        toAddress.toLowerCase() === expectedRecipient.toLowerCase() &&
        transferredAmount >= expectedAmount &&
        confirmations >= requiredConfirmations;

      return { valid, confirmations };
    } else {
      // Верификация ETH транзакции
      const tx = await this.provider.getTransaction(txHash);
      if (!tx) {
        return { valid: false, confirmations };
      }

      const valid =
        tx.to?.toLowerCase() === expectedRecipient.toLowerCase() &&
        tx.value >= expectedAmount &&
        confirmations >= requiredConfirmations;

      return { valid, confirmations };
    }
  }

  // Остановить все наблюдения
  stopAll(): void {
    this.watchers.clear();
    this.provider.removeAllListeners();
  }
}
```

## Безопасность

### Основные угрозы и защита

1. **Фронтраннинг**
   - Используйте commit-reveal схему для крупных платежей
   - Устанавливайте адекватный gas price

2. **Replay-атаки**
   - Включайте chainId в подписи
   - Используйте nonce для каждого платежа

3. **Неверная верификация**
   - Всегда проверяйте статус транзакции (receipt.status)
   - Ждите достаточное количество подтверждений (12+ для крупных сумм)
   - Проверяйте точный адрес контракта токена

4. **Ошибки округления**
   - Работайте с целыми числами в наименьших единицах
   - Учитывайте разные decimals у токенов

### Чеклист безопасности

```typescript
class PaymentSecurityChecker {
  // Минимальное количество подтверждений в зависимости от суммы
  static getRequiredConfirmations(amountUsd: number): number {
    if (amountUsd < 100) return 1;
    if (amountUsd < 1000) return 6;
    if (amountUsd < 10000) return 12;
    return 30; // Крупные суммы требуют больше подтверждений
  }

  // Проверка адреса на валидность
  static isValidAddress(address: string): boolean {
    return ethers.isAddress(address);
  }

  // Проверка контракта токена
  static async verifyTokenContract(
    provider: ethers.Provider,
    tokenAddress: string,
    expectedSymbol: string
  ): Promise<boolean> {
    const tokenContract = new ethers.Contract(
      tokenAddress,
      ['function symbol() view returns (string)'],
      provider
    );

    try {
      const symbol = await tokenContract.symbol();
      return symbol === expectedSymbol;
    } catch {
      return false;
    }
  }

  // Проверка на подозрительную активность
  static async checkSuspiciousActivity(
    provider: ethers.Provider,
    address: string
  ): Promise<{ suspicious: boolean; reason?: string }> {
    const txCount = await provider.getTransactionCount(address);
    const balance = await provider.getBalance(address);

    // Новый адрес с большим балансом может быть подозрительным
    if (txCount === 0 && balance > ethers.parseEther('100')) {
      return {
        suspicious: true,
        reason: 'New address with large balance',
      };
    }

    return { suspicious: false };
  }
}
```

## Best Practices

### 1. Архитектура платежной системы

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│   Frontend  │────▶│  Backend API │────▶│  Blockchain │
│   (React)   │     │   (Node.js)  │     │   Network   │
└─────────────┘     └──────────────┘     └─────────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │   Database   │
                    │  (PostgreSQL)│
                    └──────────────┘
```

### 2. Рекомендации по реализации

- **Храните приватные ключи безопасно** — используйте HSM или сервисы вроде AWS KMS
- **Логируйте все операции** — для аудита и отладки
- **Используйте идемпотентность** — повторные запросы не должны создавать дублирующие платежи
- **Реализуйте механизм повторных попыток** — сеть может быть временно недоступна
- **Отделяйте горячие и холодные кошельки** — минимизируйте риски
- **Мониторьте gas prices** — откладывайте транзакции при высоких комиссиях

### 3. Обработка ошибок

```typescript
enum PaymentError {
  INSUFFICIENT_FUNDS = 'INSUFFICIENT_FUNDS',
  NETWORK_ERROR = 'NETWORK_ERROR',
  TIMEOUT = 'TIMEOUT',
  INVALID_ADDRESS = 'INVALID_ADDRESS',
  UNSUPPORTED_TOKEN = 'UNSUPPORTED_TOKEN',
  TRANSACTION_FAILED = 'TRANSACTION_FAILED',
}

class PaymentException extends Error {
  constructor(
    public code: PaymentError,
    message: string,
    public details?: any
  ) {
    super(message);
    this.name = 'PaymentException';
  }
}

// Обработчик с повторными попытками
async function withRetry<T>(
  operation: () => Promise<T>,
  maxRetries: number = 3,
  delayMs: number = 1000
): Promise<T> {
  let lastError: Error | undefined;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      lastError = error as Error;
      console.log(`Attempt ${attempt} failed:`, error);

      if (attempt < maxRetries) {
        await new Promise((resolve) => setTimeout(resolve, delayMs * attempt));
      }
    }
  }

  throw lastError;
}
```

## Полезные ресурсы

- [EIP-681: URL Format for Transaction Requests](https://eips.ethereum.org/EIPS/eip-681)
- [BitPay API Documentation](https://bitpay.com/api/)
- [Coinbase Commerce API](https://docs.cdp.coinbase.com/commerce-onchain/docs/welcome)
- [ethers.js Documentation](https://docs.ethers.org/)
- [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/)

## Заключение

Интеграция криптовалютных платежей требует тщательного подхода к безопасности и надежности. Выбор между платежными шлюзами и прямыми платежами зависит от требований бизнеса: шлюзы проще в интеграции, но прямые платежи дают полный контроль. Стейблкоины решают проблему волатильности и являются оптимальным выбором для большинства e-commerce сценариев.
