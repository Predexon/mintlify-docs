# Predexon Trade SDK Documentation

## Overview

The Predexon Trade SDK is a local-first, noncustodial TypeScript SDK for trading on prediction markets. It supports two venues:

| Venue | Blockchain | Order Types | Status |
|-------|------------|-------------|--------|
| Polymarket | Polygon (EVM) | Market, Limit | Full support |
| Kalshi | Solana (DFlow) | Market only | Full support |

**Key Features:**
- Local order construction and signing (keys never leave your device)
- Cross-venue unified API
- Built-in wallet adapters for common providers
- Automatic idempotency key generation
- Comprehensive input validation
- Security-first design with server response validation

---

## Installation

```bash
npm install @predexon/trade-sdk
```

**Requirements:**
- Node.js 18+ (Node.js 20+ recommended for cleanest install)
- TypeScript 5.0+ (if using TypeScript)

**Peer Dependencies:**
The SDK bundles its dependencies internally. No peer dependencies required.

---

## Quick Start

### Polymarket (EVM)

```typescript
import { PredexonSDK, createEvmAdapter } from '@predexon/trade-sdk';

// Create SDK instance
const sdk = new PredexonSDK({
  apiKey: 'your-api-key',
  wallet: createEvmAdapter(yourWalletClient),
});

// Enable trading (one-time setup)
await sdk.enableTrading({
  core: { venue: 'polymarket', wallet: walletAddress },
  approvals: 'auto',
});

// Place a market order
const result = await sdk.placeOrder({
  core: {
    venue: 'polymarket',
    wallet: walletAddress,
    side: 'buy',
    amount: '10', // $10 USDC
    type: 'market',
  },
  flags: { tokenId: 'your-token-id' },
});
```

### Kalshi (Solana)

```typescript
import { PredexonSDK, createEvmAdapter } from '@predexon/trade-sdk';

// Create SDK with Solana signing capability
const sdk = new PredexonSDK({
  apiKey: 'your-api-key',
  wallet: {
    ...createEvmAdapter(evmWallet),
    signSolanaTransaction: async (base64Tx) => { /* sign and return base64 */ },
    signSolanaMessage: async (message) => { /* sign and return signature */ },
  },
});

// Enable trading (wallet registration)
await sdk.enableTrading({
  core: { venue: 'kalshi', wallet: solanaWalletAddress },
});

// Place a market order
const result = await sdk.placeOrder({
  core: {
    venue: 'kalshi',
    wallet: solanaWalletAddress,
    side: 'buy',
    amount: '10', // $10 USDC
    type: 'market',
  },
  flags: { ticker: 'TICKER-SYMBOL', outcome: 'Yes' },
});
```

---

## SDK Configuration

### Constructor Options

```typescript
interface PredexonSdkConfig {
  apiKey: string;                    // Required: Your Predexon API key
  wallet?: EvmSigner & SolanaSigner; // Optional: Wallet for signing
  timeouts?: {
    requestMs?: number;              // Request timeout (default: 30000ms)
  };
}
```

### Creating the SDK

```typescript
const sdk = new PredexonSDK({
  apiKey: 'pk_live_xxxxx',
  wallet: createEvmAdapter(walletClient),
  timeouts: {
    requestMs: 60000, // 60 second timeout
  },
});
```

---

## Wallet Adapters

The SDK provides adapters to normalize different wallet implementations.

### createEvmAdapter

Adapts standard EVM wallets (ethers.js, viem, wagmi) to the SDK interface.

```typescript
import { createEvmAdapter } from '@predexon/trade-sdk';

// With viem WalletClient
const wallet = createEvmAdapter(walletClient);

// With ethers.js Signer
const wallet = createEvmAdapter(ethersSigner);

// With wagmi
const wallet = createEvmAdapter(wagmiWalletClient);
```

**Supported wallet shapes:**
- `getAddress()` or `address` property
- `signTypedData(typedData)` (viem-style) or `_signTypedData(domain, types, value)` (ethers-style)
- `signTransaction(tx)`

### createPrivyAdapter

Adapts Privy embedded wallets.

```typescript
import { createPrivyAdapter } from '@predexon/trade-sdk';

const wallet = createPrivyAdapter(privyEmbeddedWallet);
```

### Custom Wallet Implementation

You can implement the wallet interface directly:

```typescript
const customWallet = {
  async getAddress(): Promise<string> {
    return '0x...';
  },
  async signTypedData(typedData: Eip712TypedData): Promise<string> {
    // Sign EIP-712 typed data
    return signature;
  },
  async signTransaction(tx: EvmTransactionRequest): Promise<string> {
    // Sign and serialize transaction
    return signedTxHex;
  },
  // For Kalshi support:
  async signSolanaTransaction(base64Tx: string): Promise<string> {
    // Sign Solana transaction, return base64
    return signedBase64;
  },
  async signSolanaMessage(message: string): Promise<string> {
    // Sign arbitrary message
    return signature;
  },
};
```

---

## Wallet Interface Types

### EvmSigner (Polymarket)

```typescript
interface EvmSigner {
  getAddress(): Promise<string>;
  signTypedData(typedData: Eip712TypedData): Promise<string>;
  signTransaction(tx: EvmTransactionRequest): Promise<string>;
}

interface Eip712TypedData {
  domain: Record<string, unknown>;
  types: Record<string, Array<{ name: string; type: string }>>;
  primaryType: string;
  message: Record<string, unknown>;
}

interface EvmTransactionRequest {
  to: string;
  data: string;
  value: string;
  chainId?: number;
  nonce?: number;
}
```

### SolanaSigner (Kalshi)

```typescript
interface SolanaSigner {
  signSolanaTransaction(transactionBase64: string): Promise<string>;
  signSolanaMessage?(message: string): Promise<string>; // Required for enableTrading
}
```

---

## Core Methods

### enableTrading

Enable trading for a venue. Must be called once before placing orders.

```typescript
async enableTrading(request: EnableTradingRequest): Promise<EnableTradingResponse>
```

#### Polymarket

Handles CLOB credential registration and contract approvals.

```typescript
await sdk.enableTrading({
  core: { venue: 'polymarket', wallet: '0x...' },
  approvals: 'auto', // 'auto' | 'manual' | 'skip'
});
```

**Approval modes:**
- `'auto'`: SDK signs and broadcasts approval transactions automatically
- `'manual'`: Returns unsigned approval transactions for you to broadcast
- `'skip'`: Skip approval check (use if already approved)

**Response:**
```typescript
interface EnableTradingResponse {
  venue: 'polymarket';
  tradingEnabled: boolean;
  approvalsComplete: boolean;
  clobRegistered: boolean;
  approvalTransactions?: ApprovalTransaction[]; // Only for 'manual' mode
}
```

#### Kalshi

Registers your Solana wallet with Predexon.

```typescript
await sdk.enableTrading({
  core: { venue: 'kalshi', wallet: 'SolanaPublicKey...' },
});
```

**Response:**
```typescript
interface EnableTradingResponse {
  venue: 'kalshi';
  tradingEnabled: boolean;
  ownershipVerified: boolean;
}
```

---

### placeOrder

Build, sign, and submit an order in one call. This is the primary API for trading.

```typescript
async placeOrder(request: PlaceOrderRequest): Promise<PlaceOrderResponse>
```

#### Polymarket Orders

**Market BUY (by USDC amount):**
```typescript
const result = await sdk.placeOrder({
  core: {
    venue: 'polymarket',
    wallet: '0x...',
    side: 'buy',
    amount: '100', // Spend $100 USDC
    type: 'market',
  },
  flags: { tokenId: '12345...' },
});
```

**Market BUY (by share size):**
```typescript
const result = await sdk.placeOrder({
  core: {
    venue: 'polymarket',
    wallet: '0x...',
    side: 'buy',
    size: '50', // Buy 50 shares
    type: 'market',
  },
  flags: { tokenId: '12345...' },
});
```

**Market SELL:**
```typescript
const result = await sdk.placeOrder({
  core: {
    venue: 'polymarket',
    wallet: '0x...',
    side: 'sell',
    size: '50', // Sell 50 shares
    type: 'market',
  },
  flags: { tokenId: '12345...' },
});
```

**Limit BUY:**
```typescript
const result = await sdk.placeOrder({
  core: {
    venue: 'polymarket',
    wallet: '0x...',
    side: 'buy',
    size: '100', // Buy 100 shares
    price: '0.65', // At $0.65 per share
    type: 'limit',
  },
  flags: { tokenId: '12345...' },
});
```

**Limit SELL:**
```typescript
const result = await sdk.placeOrder({
  core: {
    venue: 'polymarket',
    wallet: '0x...',
    side: 'sell',
    size: '100',
    price: '0.70',
    type: 'limit',
  },
  flags: { tokenId: '12345...' },
});
```

#### Kalshi Orders

**Market BUY:**
```typescript
const result = await sdk.placeOrder({
  core: {
    venue: 'kalshi',
    wallet: 'SolanaPublicKey...',
    side: 'buy',
    amount: '50', // Spend $50 USDC
    type: 'market',
  },
  flags: {
    ticker: 'PRES-2024-DEM',
    outcome: 'Yes', // or 'No'
  },
});
```

**Market SELL:**
```typescript
const result = await sdk.placeOrder({
  core: {
    venue: 'kalshi',
    wallet: 'SolanaPublicKey...',
    side: 'sell',
    size: '25', // Sell 25 tokens
    type: 'market',
  },
  flags: {
    ticker: 'PRES-2024-DEM',
    outcome: 'Yes',
  },
});
```

#### Response

```typescript
interface PlaceOrderResponse {
  orderId: string;
  venue: 'polymarket' | 'kalshi';
  status: 'filled' | 'pending' | 'canceled' | 'failed' | 'reverted';
  price: string | null;
  amount?: string;      // USDC amount
  size?: string;        // Share/token size
  filledAmount?: string;
  createdAt: string;
  message?: string;
}
```

---

### prepareOrder

Build and sign an order without submitting. For advanced use cases.

```typescript
async prepareOrder(request: PrepareOrderRequest): Promise<UnifiedPrepareOrderResponse>
```

#### Polymarket Response

```typescript
interface PrepareOrderResponse {
  order: PolymarketOrder;
  signature: string;
}
```

#### Kalshi Response

```typescript
interface PrepareKalshiOrderResponse {
  transaction: string;  // Base64 Solana transaction
  orderParams: KalshiOrderParams;
  executionMode?: 'sync' | 'async';
  sponsorAddress?: string;
}
```

---

### submitOrder

Submit a previously prepared order. For advanced use cases.

```typescript
async submitOrder(request: UnifiedSubmitOrderRequest): Promise<PlaceOrderResponse>
```

#### Polymarket Submit

```typescript
await sdk.submitOrder({
  order: preparedOrder,
  signature: signature,
  orderType: 'market', // or 'limit'
  idempotencyKey: 'optional-key',
});
```

#### Kalshi Submit

```typescript
await sdk.submitOrder({
  walletAddress: 'SolanaPublicKey...',
  signedTransaction: signedBase64,
  orderParams: { ticker, outcome, side, amount },
});
```

---

### getOpenOrders

List open orders for a wallet.

```typescript
async getOpenOrders(params): Promise<OpenOrder[]>
```

```typescript
const orders = await sdk.getOpenOrders({
  core: { venue: 'polymarket', wallet: '0x...' },
});
```

**Response:**
```typescript
interface OpenOrder {
  orderId: string;
  venue: 'polymarket' | 'kalshi';
  tokenId?: string;   // Polymarket
  ticker?: string;    // Kalshi
  side: string;
  outcome?: string;
  amount: string;
  amountMatched?: string;
  price: string;
  status: string;
  createdAt: string;
}
```

---

### cancelOrder

Cancel a specific open order.

```typescript
async cancelOrder(params): Promise<void>
```

```typescript
await sdk.cancelOrder({
  core: { venue: 'polymarket', wallet: '0x...' },
  orderId: 'order-id-123',
});
```

---

### cancelAllOrders

Cancel all open orders for a wallet.

```typescript
async cancelAllOrders(params): Promise<{ cancelled: number }>
```

```typescript
const result = await sdk.cancelAllOrders({
  core: { venue: 'polymarket', wallet: '0x...' },
});
console.log(`Cancelled ${result.cancelled} orders`);
```

---

### getPositions

Get current positions for a wallet.

```typescript
async getPositions(params): Promise<Position[]>
```

```typescript
const positions = await sdk.getPositions({
  core: { venue: 'polymarket', wallet: '0x...' },
});
```

**Response:**
```typescript
interface Position {
  venue: 'polymarket' | 'kalshi';
  tokenId?: string;        // Polymarket
  ticker?: string;         // Kalshi
  title?: string | null;
  outcome: string;
  size: string;
  status: 'active' | 'resolved' | 'redeemable';
  result?: 'won' | 'lost' | null;
  averagePrice?: string | null;
  currentPrice?: string | null;
  currentValue?: string | null;
  pnl?: string | null;
}
```

---

### getBalance

Get wallet balance.

```typescript
async getBalance(params): Promise<Balance[]>
```

```typescript
const balances = await sdk.getBalance({
  core: { venue: 'polymarket', wallet: '0x...' },
});
```

**Response:**
```typescript
interface Balance {
  venue: 'polymarket' | 'kalshi';
  available: string;  // Available for trading
  locked: string;     // Locked in open orders
  total: string;      // Total balance
}
```

---

### redeem

Redeem resolved positions.

```typescript
async redeem(request: RedeemRequest): Promise<RedeemResponse>
```

#### Polymarket

```typescript
const result = await sdk.redeem({
  core: { venue: 'polymarket', wallet: '0x...' },
  flags: { tokenId: 'resolved-token-id' },
});
```

#### Kalshi

```typescript
const result = await sdk.redeem({
  core: { venue: 'kalshi', wallet: 'SolanaPublicKey...' },
  flags: { ticker: 'TICKER', outcome: 'Yes' },
});
```

**Response:**
```typescript
interface RedeemResponse {
  transactionHash: string;
  venue: 'polymarket' | 'kalshi';
  status: 'completed' | 'reverted' | 'pending';
  amountRedeemed?: string;
  sizeRedeemed?: string;
  message?: string;
}
```

---

## Order Semantics Reference

### Polymarket

| Order Type | Side | Required | Optional |
|------------|------|----------|----------|
| Market | BUY | `amount` OR `size` | - |
| Market | SELL | `size` | - |
| Limit | BUY | `size` + `price` | - |
| Limit | SELL | `size` + `price` | - |

- `amount` = USDC value (how much to spend)
- `size` = number of shares
- `price` = price per share (0 < price <= 1)

### Kalshi

| Order Type | Side | Required |
|------------|------|----------|
| Market | BUY | `amount` (USDC) |
| Market | SELL | `size` (tokens) |

**Note:** Kalshi only supports market orders. Limit orders are not supported.

---

## Flags Reference

### PolymarketFlags

```typescript
interface PolymarketFlags {
  tokenId: string;       // Required: The outcome token ID
  feeRateBps?: string;   // Optional: Fee rate in basis points (default: '0')
  expiration?: string;   // Optional: Order expiration timestamp (default: '0' = no expiration)
}
```

### KalshiFlags

```typescript
interface KalshiFlags {
  ticker: string;                    // Required: Market ticker symbol
  outcome: 'Yes' | 'No';             // Required: Which outcome to trade
  executionMode?: 'sync' | 'async';  // Optional: Execution mode (default: 'sync')
}
```

---

## Idempotency

The SDK automatically generates cryptographically secure idempotency keys when not provided. You can also provide your own:

```typescript
await sdk.placeOrder({
  core: { ... },
  flags: { ... },
  idempotencyKey: 'my-custom-key-12345',
});
```

**Key format:** `sdk-order:{timestamp}:{random}`

To generate keys manually:
```typescript
import { generateIdempotencyKey } from '@predexon/trade-sdk';

const key = generateIdempotencyKey();
// Example: "sdk-order:lq2abc:a1b2c3d4e5f6..."

const customKey = generateIdempotencyKey('my-prefix');
// Example: "my-prefix:lq2abc:a1b2c3d4e5f6..."
```

---

## Error Handling

The SDK throws typed errors for different failure scenarios:

```typescript
import {
  SdkError,
  NetworkError,
  AuthError,
  ValidationError,
  SigningError,
  OrderRejectedError,
  NotSupportedError,
} from '@predexon/trade-sdk';

try {
  await sdk.placeOrder({ ... });
} catch (error) {
  if (error instanceof ValidationError) {
    console.log('Invalid input:', error.message);
    console.log('Error code:', error.code); // 'VALIDATION_ERROR'
  } else if (error instanceof AuthError) {
    console.log('Authentication failed:', error.message);
  } else if (error instanceof NetworkError) {
    console.log('Network issue:', error.message);
  } else if (error instanceof SigningError) {
    console.log('Wallet signing failed:', error.message);
  } else if (error instanceof OrderRejectedError) {
    console.log('Order rejected:', error.message);
  } else if (error instanceof NotSupportedError) {
    console.log('Not supported:', error.message);
  }
}
```

### Error Types

| Error Class | Code | Description |
|-------------|------|-------------|
| `SdkError` | `SDK_ERROR` | Base error class |
| `NetworkError` | `NETWORK_ERROR` | HTTP/network failures |
| `AuthError` | `AUTH_ERROR` | API key or wallet auth issues |
| `ValidationError` | `VALIDATION_ERROR` | Invalid input parameters |
| `SigningError` | `SIGNING_ERROR` | Wallet signing failures |
| `OrderRejectedError` | `ORDER_REJECTED` | Order rejected by venue |
| `NotSupportedError` | `NOT_SUPPORTED` | Unsupported operation |

---

## Security

### Architecture

The SDK is designed with security as a priority:

1. **Local-first signing**: Private keys never leave your device. All order signing happens locally.

2. **Hard-locked API endpoint**: The SDK only communicates with `https://trade.predexon.com`. This cannot be overridden to prevent MITM attacks.

3. **No redirects**: HTTP redirects are rejected to prevent API key leakage.

4. **HTTPS only**: All connections use HTTPS.

### Validation

The SDK validates server responses before signing:

**Polymarket:**
- CLOB auth typed data domain validation (name, version, chainId)
- Redemption transaction contract allowlist validation
- Local redemption transaction construction (server only provides conditionId)

**Kalshi:**
- Order prepare response echo validation (ticker, outcome, side, amount)
- Solana transaction signer validation:
  - Expected wallet must be a required signer
  - No unexpected signers allowed
  - Fee payer must be wallet or known sponsor

### Contract Addresses (Polymarket)

The SDK uses hardcoded, verified contract addresses:

| Contract | Address |
|----------|---------|
| USDC | `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174` |
| CTF | `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045` |
| CTF Exchange | `0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E` |
| NegRisk CTF Exchange | `0xC5d563A36AE78145C45a50134d48A1215220f80a` |
| NegRisk Adapter | `0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296` |

---

## TypeScript Support

The SDK is written in TypeScript and exports all types:

```typescript
import type {
  // Core types
  Venue,
  OrderSide,

  // Request types
  PlaceOrderRequest,
  PrepareOrderRequest,
  EnableTradingRequest,
  RedeemRequest,

  // Response types
  PlaceOrderResponse,
  EnableTradingResponse,
  RedeemResponse,
  OpenOrder,
  Position,
  Balance,

  // Flags
  PolymarketFlags,
  KalshiFlags,

  // Wallet types
  EvmSigner,
  SolanaSigner,
  PrivySigner,
  Eip712TypedData,
  EvmTransactionRequest,

  // Config
  PredexonSdkConfig,

  // Order types
  PolymarketOrder,
  KalshiOrderParams,

  // Approval types
  ApprovalStatus,
  ApprovalTransaction,
} from '@predexon/trade-sdk';
```

---

## Module Formats

The SDK ships as dual ESM/CJS:

```typescript
// ESM (recommended)
import { PredexonSDK } from '@predexon/trade-sdk';

// CommonJS
const { PredexonSDK } = require('@predexon/trade-sdk');
```

---

## Complete Examples

### Full Polymarket Trading Flow

```typescript
import { PredexonSDK, createEvmAdapter } from '@predexon/trade-sdk';
import { createWalletClient, http } from 'viem';
import { polygon } from 'viem/chains';
import { privateKeyToAccount } from 'viem/accounts';

// 1. Create wallet client
const account = privateKeyToAccount('0x...');
const walletClient = createWalletClient({
  account,
  chain: polygon,
  transport: http(),
});

// 2. Create SDK
const sdk = new PredexonSDK({
  apiKey: process.env.PREDEXON_API_KEY!,
  wallet: createEvmAdapter(walletClient),
});

// 3. Enable trading (first time only)
const enableResult = await sdk.enableTrading({
  core: { venue: 'polymarket', wallet: account.address },
  approvals: 'auto',
});
console.log('Trading enabled:', enableResult.tradingEnabled);

// 4. Check balance
const balances = await sdk.getBalance({
  core: { venue: 'polymarket', wallet: account.address },
});
console.log('Available USDC:', balances[0].available);

// 5. Place a market buy order
const order = await sdk.placeOrder({
  core: {
    venue: 'polymarket',
    wallet: account.address,
    side: 'buy',
    amount: '10',
    type: 'market',
  },
  flags: { tokenId: 'your-token-id' },
});
console.log('Order placed:', order.orderId, order.status);

// 6. Check positions
const positions = await sdk.getPositions({
  core: { venue: 'polymarket', wallet: account.address },
});
console.log('Positions:', positions);

// 7. Place a limit sell order
const limitOrder = await sdk.placeOrder({
  core: {
    venue: 'polymarket',
    wallet: account.address,
    side: 'sell',
    size: '5',
    price: '0.75',
    type: 'limit',
  },
  flags: { tokenId: 'your-token-id' },
});
console.log('Limit order:', limitOrder.orderId);

// 8. View open orders
const openOrders = await sdk.getOpenOrders({
  core: { venue: 'polymarket', wallet: account.address },
});
console.log('Open orders:', openOrders.length);

// 9. Cancel all orders
const cancelled = await sdk.cancelAllOrders({
  core: { venue: 'polymarket', wallet: account.address },
});
console.log('Cancelled:', cancelled.cancelled);
```

### Full Kalshi Trading Flow

```typescript
import { PredexonSDK, createEvmAdapter } from '@predexon/trade-sdk';
import { Keypair } from '@solana/web3.js';
import nacl from 'tweetnacl';

// 1. Create Solana keypair
const solanaKeypair = Keypair.generate();

// 2. Create SDK with Solana signer
const sdk = new PredexonSDK({
  apiKey: process.env.PREDEXON_API_KEY!,
  wallet: {
    // EVM methods (can be stubs if only using Kalshi)
    async getAddress() { return '0x0000000000000000000000000000000000000000'; },
    async signTypedData() { throw new Error('Not needed for Kalshi'); },
    async signTransaction() { throw new Error('Not needed for Kalshi'); },

    // Solana methods
    async signSolanaTransaction(base64Tx: string): Promise<string> {
      const txBytes = Buffer.from(base64Tx, 'base64');
      const { VersionedTransaction } = await import('@solana/web3.js');
      const tx = VersionedTransaction.deserialize(txBytes);
      tx.sign([solanaKeypair]);
      return Buffer.from(tx.serialize()).toString('base64');
    },
    async signSolanaMessage(message: string): Promise<string> {
      const messageBytes = new TextEncoder().encode(message);
      const signature = nacl.sign.detached(messageBytes, solanaKeypair.secretKey);
      return Buffer.from(signature).toString('base64');
    },
  },
});

// 3. Enable trading
const walletAddress = solanaKeypair.publicKey.toBase58();
await sdk.enableTrading({
  core: { venue: 'kalshi', wallet: walletAddress },
});

// 4. Place market buy
const order = await sdk.placeOrder({
  core: {
    venue: 'kalshi',
    wallet: walletAddress,
    side: 'buy',
    amount: '25',
    type: 'market',
  },
  flags: {
    ticker: 'PRES-2024-DEM',
    outcome: 'Yes',
  },
});
console.log('Kalshi order:', order.orderId);

// 5. Check positions
const positions = await sdk.getPositions({
  core: { venue: 'kalshi', wallet: walletAddress },
});
console.log('Positions:', positions);
```

### Error Handling Example

```typescript
import {
  PredexonSDK,
  createEvmAdapter,
  ValidationError,
  AuthError,
  NetworkError,
  OrderRejectedError,
} from '@predexon/trade-sdk';

async function placeOrderSafely(sdk: PredexonSDK, request: PlaceOrderRequest) {
  try {
    const result = await sdk.placeOrder(request);
    return { success: true, order: result };
  } catch (error) {
    if (error instanceof ValidationError) {
      // Invalid input - fix the request
      return { success: false, error: 'validation', message: error.message };
    }
    if (error instanceof AuthError) {
      // Auth issue - check API key or wallet
      return { success: false, error: 'auth', message: error.message };
    }
    if (error instanceof NetworkError) {
      // Network issue - retry later
      return { success: false, error: 'network', message: error.message };
    }
    if (error instanceof OrderRejectedError) {
      // Order rejected - check liquidity/balance
      return { success: false, error: 'rejected', message: error.message };
    }
    throw error; // Unknown error
  }
}
```

### Manual Approval Flow

```typescript
// For users who want to broadcast approvals themselves
const result = await sdk.enableTrading({
  core: { venue: 'polymarket', wallet: address },
  approvals: 'manual',
});

if (result.approvalTransactions) {
  for (const tx of result.approvalTransactions) {
    // Broadcast each transaction using your preferred method
    console.log('Approval needed:', {
      to: tx.to,
      data: tx.data,
      chainId: tx.chainId,
    });

    // Example with viem:
    // await walletClient.sendTransaction(tx);
  }
}
```

---

## Validation Rules

### Polymarket Validation

| Rule | Description |
|------|-------------|
| tokenId required | Must be a non-empty string |
| Price range | Must be > 0 and <= 1 |
| Wallet match | Signer address must match wallet parameter |
| Market BUY | Requires `amount` OR `size` (not both) |
| Market SELL | Requires `size` only |
| Limit orders | Requires `size` and `price` |
| Min order size | Validated against orderbook minimum |

### Kalshi Validation

| Rule | Description |
|------|-------------|
| ticker required | Must be non-empty |
| outcome required | Must be 'Yes' or 'No' |
| Market orders only | Limit orders not supported |
| Market BUY | Requires `amount` (USDC) |
| Market SELL | Requires `size` (tokens) |
| Positive amounts | All amounts must be > 0 |

---

## FAQ

### Why can't I set a custom API base URL?

The SDK hard-locks to `https://trade.predexon.com` to prevent man-in-the-middle attacks. A compromised or malicious base URL could steal your API key or serve malicious transactions for signing.

### Why does Kalshi only support market orders?

Kalshi's DFlow infrastructure currently only supports market (Fill-And-Kill) orders. Limit order support may be added in the future.

### How do I handle wallet connection in a browser?

Use the appropriate adapter for your wallet provider:

```typescript
// For wagmi/viem
const walletClient = await getWalletClient();
const wallet = createEvmAdapter(walletClient);

// For Privy
const wallet = createPrivyAdapter(privyEmbeddedWallet);
```

### Can I use the SDK without a wallet?

Some read operations don't require a wallet:

```typescript
const sdk = new PredexonSDK({ apiKey: 'your-key' });

// These work without a wallet
const positions = await sdk.getPositions({ core: { venue: 'polymarket', wallet: address } });
const balance = await sdk.getBalance({ core: { venue: 'polymarket', wallet: address } });
const orders = await sdk.getOpenOrders({ core: { venue: 'polymarket', wallet: address } });
```

But placing orders, enabling trading, and redemption require a wallet.

### How do idempotency keys work?

Idempotency keys ensure that retrying a failed request doesn't create duplicate orders. The SDK auto-generates cryptographically secure keys, but you can provide your own:

```typescript
// Auto-generated (recommended)
await sdk.placeOrder({ core: {...}, flags: {...} });

// Custom key
await sdk.placeOrder({
  core: {...},
  flags: {...},
  idempotencyKey: `user-${userId}-order-${Date.now()}`,
});
```

---

## Changelog

### v0.0.2
- Local Polymarket redemption transaction construction
- Kalshi Solana transaction signer validation
- Echo field validation for Kalshi prepare responses
- Added `@solana/web3.js` dependency

### v0.0.1
- Initial release
- Polymarket support (market + limit orders)
- Kalshi support (market orders)
- Local order construction using `@polymarket/clob-client`
- Wallet adapters for EVM and Privy
- Comprehensive input validation
- Typed error classes
