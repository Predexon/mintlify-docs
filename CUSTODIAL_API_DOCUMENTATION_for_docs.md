# Predexon Custodial Trading API Documentation

## Overview

The Predexon Custodial Trading API provides a unified interface for trading on prediction markets across multiple venues (Polymarket and Kalshi). Users don't manage their own wallets - the API handles wallet creation, key management, and transaction signing on their behalf.

### Base URLs

| Environment | URL |
|-------------|-----|
| Production (Public) | `https://predexon-trade-api-8721254678.europe-north2.run.app` |
| Internal | `https://predexon-trade-api-8970529761.europe-north2.run.app` |

### Authentication

All requests require an API key passed via AWS API Gateway. The API key is validated by the gateway, and the key ID is forwarded to the backend via the `x-api-key-id` header.

**Required Headers:**
| Header | Description |
|--------|-------------|
| `x-api-key` | Your API key (validated by API Gateway) |
| `x-api-key-id` | API key identifier (auto-forwarded by Gateway) |

**Ownership Model:**

Each API key can create and manage multiple users. Users are isolated between API keys.

| API Key | Users Owned |
|---------|-------------|
| API Key A | User 1, User 2, User 3 |
| API Key B | User 4, User 5 |

- API Key A can only access Users 1, 2, 3
- API Key B can only access Users 4, 5
- Cross-access returns `403 Forbidden`

### Supported Venues

| Venue | Blockchain | Description |
|-------|------------|-------------|
| `polymarket` | Polygon | Polymarket CLOB with Gnosis Safe wallets |
| `kalshi` | Solana | Kalshi via DFlow atomic swaps |

### Rate Limits

Rate limiting is handled by AWS API Gateway. Contact support for limit details.

### Health Check

- **Method:** `GET`
- **Path:** `/health`
- **Auth Required:** No

**Response:**
```json
{
  "status": "ok",
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

### Geo-Restrictions

Kalshi trading is blocked in certain countries. Affected endpoints return:

```json
{
  "error": "Kalshi trading is not available in your region",
  "code": "GEO_RESTRICTED"
}
```

**Blocked Countries (51 total):**
Afghanistan, Algeria, Angola, Australia, Belarus, Belgium, Bolivia, Bulgaria, Burkina Faso, Cameroon, Canada, Central African Republic, China, Côte d'Ivoire, Cuba, Democratic Republic of the Congo, Ethiopia, France, Haiti, Iran, Iraq, Italy, Kenya, Laos, Lebanon, Libya, Mali, Monaco, Mozambique, Myanmar, Namibia, Nicaragua, Niger, North Korea, Poland, Russia, Singapore, Somalia, South Sudan, Sudan, Switzerland, Syria, Taiwan, Thailand, Ukraine, United Arab Emirates, United Kingdom, United States, Venezuela, Yemen, Zimbabwe

**Affected Endpoints:**
- `POST /api/users/create` (creates both Polymarket and Kalshi wallets)
- Any endpoint with `venue=kalshi`

---

## Endpoints

### Quick Reference

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/health` | Health check |
| `POST` | `/api/users/create` | Create user with wallets |
| `GET` | `/api/users/:userId` | Get user info |
| `DELETE` | `/api/users/:userId` | Delete user |
| `GET` | `/api/users/:userId/balance` | Get USDC balance |
| `POST` | `/api/users/:userId/orders` | Place order |
| `GET` | `/api/users/:userId/orders` | Get open orders |
| `DELETE` | `/api/users/:userId/orders/:orderId` | Cancel order |
| `DELETE` | `/api/users/:userId/orders` | Cancel all orders |
| `GET` | `/api/users/:userId/positions` | Get positions |
| `POST` | `/api/users/:userId/redeem` | Redeem resolved position |
| `POST` | `/api/users/:userId/withdraw` | Withdraw USDC |
| `POST` | `/api/users/:userId/withdraw/withdraw-sol` | Withdraw SOL from Kalshi wallet |
| `GET` | `/api/bridge/deposit` | Get cross-chain deposit addresses |

---

### User Management

#### Create User

Creates a new user with wallets for both Polymarket (Polygon) and Kalshi (Solana).

- **Method:** `POST`
- **Path:** `/api/users/create`
- **Auth Required:** Yes
- **Geo-Restriction:** Blocked in Kalshi-restricted countries

**Request Body:** None required

**Success Response (201 Created):**
```json
{
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "createdAt": "2024-01-15T10:30:00.000Z",
  "polymarketWalletAddress": "0x1234567890abcdef1234567890abcdef12345678",
  "kalshiWalletAddress": "7BfknMUrXjZ96swfM4uLzDXi9QgtcdjQKiWcG4QxW2So"
}
```

**Response Fields:**
| Field | Type | Description |
|-------|------|-------------|
| `userId` | string | UUID v4 identifier for the user |
| `createdAt` | string | ISO 8601 timestamp |
| `polymarketWalletAddress` | string | Gnosis Safe address for Polymarket deposits |
| `kalshiWalletAddress` | string | Solana wallet address for Kalshi deposits |

**Errors:**
| Status | Error | Cause |
|--------|-------|-------|
| 401 | "API key required" | Missing or invalid API key |
| 403 | "Access denied from this region" | Kalshi geo-restriction |
| 500 | "Failed to create user. Please try again." | Internal error |

---

#### Get User

Retrieves user information including wallet addresses.

- **Method:** `GET`
- **Path:** `/api/users/:userId`
- **Auth Required:** Yes (must own user)

**Path Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| `userId` | string (UUID) | Yes | The user's unique identifier |

**Success Response (200 OK):**
```json
{
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "createdAt": "2024-01-15T10:30:00.000Z",
  "polymarketWalletAddress": "0x1234567890abcdef1234567890abcdef12345678",
  "kalshiWalletAddress": "7BfknMUrXjZ96swfM4uLzDXi9QgtcdjQKiWcG4QxW2So"
}
```

**Errors:**
| Status | Error | Cause |
|--------|-------|-------|
| 401 | "API key required" | Missing API key |
| 403 | "Access denied" | User not owned by this API key |
| 404 | "User not found" | Invalid userId or user doesn't exist |
| 500 | "Failed to get user. Please try again." | Internal error |

---

#### Delete User

Deletes a user and all associated data (wallets, credentials).

- **Method:** `DELETE`
- **Path:** `/api/users/:userId`
- **Auth Required:** Yes (must own user)

**Path Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| `userId` | string (UUID) | Yes | The user's unique identifier |

**Success Response:** `204 No Content`

**Errors:**
| Status | Error | Cause |
|--------|-------|-------|
| 401 | "API key required" | Missing API key |
| 403 | "Access denied" | User not owned by this API key |
| 404 | "User not found" | User doesn't exist |
| 500 | "Failed to delete user. Please try again." | Internal error |

---

### Trading

#### Get Balance

Returns the user's USDC balance across venues.

- **Method:** `GET`
- **Path:** `/api/users/:userId/balance`
- **Auth Required:** Yes (must own user)

**Path Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| `userId` | string (UUID) | Yes | The user's unique identifier |

**Query Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `venue` | string | No | all | Filter by venue: `polymarket` or `kalshi` |

**Success Response (200 OK):**
```json
{
  "balances": [
    {
      "venue": "polymarket",
      "available": "150.50",
      "locked": "25.00",
      "total": "175.50"
    },
    {
      "venue": "kalshi",
      "available": "100.00",
      "locked": "0",
      "total": "100.00"
    }
  ]
}
```

**Balance Fields:**
| Field | Type | Description |
|-------|------|-------------|
| `venue` | string | The venue name |
| `available` | string | USDC available for trading |
| `locked` | string | USDC locked in open orders |
| `total` | string | Total USDC (available + locked) |

**Errors:**
| Status | Error | Cause |
|--------|-------|-------|
| 400 | "Invalid venue: 'xxx'. Must be 'polymarket' or 'kalshi'" | Invalid venue parameter |
| 403 | "Kalshi trading is not available in your region" | Geo-blocked (if venue=kalshi) |
| 404 | "User not found" | User doesn't exist |
| 500 | "Failed to get balance. Please try again." | Internal error |

---

#### Place Order

Places a buy or sell order on a prediction market.

- **Method:** `POST`
- **Path:** `/api/users/:userId/orders`
- **Auth Required:** Yes (must own user)

**Path Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| `userId` | string (UUID) | Yes | The user's unique identifier |

**Request Body:**

*For Polymarket:*
```json
{
  "venue": "polymarket",
  "tokenId": "71321045679252212594626385...",
  "side": "buy",
  "amount": "50.00",
  "price": "0.65",
  "type": "limit"
}
```

*For Kalshi:*
```json
{
  "venue": "kalshi",
  "ticker": "PRES-2024-DJT",
  "outcome": "Yes",
  "side": "buy",
  "amount": "50.00"
}
```

**Request Body Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `venue` | string | Yes | `polymarket` or `kalshi` |
| `tokenId` | string | Polymarket only | The outcome token ID (required for Polymarket) |
| `ticker` | string | Kalshi only | Market ticker (required for Kalshi, e.g., "PRES-2024-DJT") |
| `outcome` | string | Kalshi only | `Yes` or `No` (required for Kalshi) |
| `side` | string | Yes | `buy` or `sell` |
| `amount` | string | Yes | USDC amount to spend (buy) or tokens to sell (sell) |
| `price` | string | Polymarket limit only | Price per share (0-1). Required for Polymarket limit orders. Ignored for market orders and Kalshi (uses DFlow's best available price). |
| `type` | string | No | `limit` (default) or `market`. Kalshi orders are always atomic swaps regardless of this field. |

**Amount Semantics:**
- **BUY:** Amount of USDC to spend
- **SELL:** Number of tokens/contracts to sell

**Success Response (201 Created):**
```json
{
  "orderId": "0x1234567890abcdef...",
  "venue": "polymarket",
  "tokenId": "71321045679252212594626385...",
  "side": "buy",
  "outcome": "Yes",
  "amount": "50.00",
  "price": "0.65",
  "status": "live",
  "createdAt": "2024-01-15T10:30:00.000Z"
}
```

**Order Status Values:**
| Status | Description |
|--------|-------------|
| `live` | Order is active on the orderbook (Polymarket) |
| `filled` | Order fully executed (Kalshi - atomic swaps) |
| `matched` | Order partially or fully matched (Polymarket) |

**Errors:**
| Status | Error | Cause |
|--------|-------|-------|
| 400 | "Missing required fields: venue, side, amount" | Missing required parameters |
| 400 | "Missing required fields: tokenId" | Polymarket missing tokenId |
| 400 | "Missing required fields: price" | Polymarket limit order missing price |
| 400 | "Missing required fields: ticker, outcome" | Kalshi missing ticker/outcome |
| 400 | "Polymarket orders require tokenId" | tokenId validation in service |
| 400 | "Kalshi orders require ticker and outcome (Yes/No)" | ticker/outcome validation in service |
| 400 | "Price must be between 0 and 1" | Invalid price for Polymarket limit order |
| 400 | "not enough balance / allowance" | Insufficient USDC balance (Polymarket) |
| 400 | "Insufficient USDC balance to buy X contracts" | Insufficient balance (Kalshi) |
| 400 | "Insufficient token balance to sell X contracts" | Trying to sell more than owned |
| 403 | "Kalshi trading is not available in your region" | Geo-blocked (if venue=kalshi) |

---

#### Get Orders

Returns open orders for the user.

- **Method:** `GET`
- **Path:** `/api/users/:userId/orders`
- **Auth Required:** Yes (must own user)

**Query Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `venue` | string | No | all | Filter by venue |

**Success Response (200 OK):**
```json
{
  "orders": [
    {
      "orderId": "0x1234567890abcdef...",
      "venue": "polymarket",
      "tokenId": "71321045679252...",
      "side": "buy",
      "outcome": "Yes",
      "amount": "50.00",
      "price": "0.65",
      "status": "live",
      "createdAt": "2024-01-15T10:30:00.000Z"
    }
  ]
}
```

**Note:** Kalshi orders via DFlow are atomic swaps and don't persist as open orders. This endpoint primarily returns Polymarket orders.

**Errors:**
| Status | Error | Cause |
|--------|-------|-------|
| 400 | "Invalid venue: 'xxx'. Must be 'polymarket' or 'kalshi'" | Invalid venue parameter |
| 403 | "Kalshi trading is not available in your region" | Geo-blocked (if venue=kalshi) |
| 404 | "User not found" | User doesn't exist |
| 500 | "Failed to get orders. Please try again." | Internal error |

---

#### Cancel Order

Cancels a specific open order.

- **Method:** `DELETE`
- **Path:** `/api/users/:userId/orders/:orderId`
- **Auth Required:** Yes (must own user)

**Path Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| `userId` | string (UUID) | Yes | The user's unique identifier |
| `orderId` | string | Yes | The order ID to cancel |

**Query Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| `venue` | string | No | Specify venue if known |

**Success Response:** `204 No Content`

**Errors:**
| Status | Error | Cause |
|--------|-------|-------|
| 400 | "Invalid venue: 'xxx'. Must be 'polymarket' or 'kalshi'" | Invalid venue parameter |
| 400 | "Kalshi orders via DFlow cannot be cancelled - they are atomic swaps" | Kalshi orders can't be cancelled |
| 403 | "Kalshi trading is not available in your region" | Geo-blocked (if venue=kalshi) |
| 500 | "Failed to cancel order. Please try again." | Order not found or already filled |

---

#### Cancel All Orders

Cancels all open orders for the user.

- **Method:** `DELETE`
- **Path:** `/api/users/:userId/orders`
- **Auth Required:** Yes (must own user)

**Query Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| `venue` | string | No | Filter by venue |

**Success Response (200 OK):**
```json
{
  "cancelled": 3
}
```

**Errors:**
| Status | Error | Cause |
|--------|-------|-------|
| 400 | "Invalid venue: 'xxx'. Must be 'polymarket' or 'kalshi'" | Invalid venue parameter |
| 403 | "Kalshi trading is not available in your region" | Geo-blocked (if venue=kalshi) |
| 500 | "Failed to cancel orders. Please try again." | Internal error |

---

#### Get Positions

Returns the user's current positions across venues.

- **Method:** `GET`
- **Path:** `/api/users/:userId/positions`
- **Auth Required:** Yes (must own user)

**Query Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| `venue` | string | No | Filter by venue |

**Success Response (200 OK):**
```json
{
  "positions": [
    {
      "venue": "polymarket",
      "tokenId": "71321045679252...",
      "title": "Will Bitcoin reach $100k by 2024?",
      "outcome": "Yes",
      "size": "100.50",
      "status": "active",
      "result": null,
      "averagePrice": "0.45",
      "currentPrice": "0.62",
      "currentValue": "62.31",
      "pnl": "17.08"
    },
    {
      "venue": "kalshi",
      "ticker": "PRES-2024-DJT",
      "title": "Trump wins 2024 election",
      "outcome": "Yes",
      "size": "50.00",
      "status": "redeemable",
      "result": "won",
      "averagePrice": "0.55",
      "currentPrice": "1.0000",
      "currentValue": "50.00",
      "pnl": "22.50"
    }
  ]
}
```

**Position Fields:**
| Field | Type | Description |
|-------|------|-------------|
| `venue` | string | Trading venue |
| `tokenId` | string | Polymarket token ID (if Polymarket) |
| `ticker` | string | Kalshi market ticker (if Kalshi) |
| `title` | string | Market question/title |
| `outcome` | string | Position side (e.g., "Yes", "No", team name) |
| `size` | string | Number of shares/contracts held |
| `status` | string | `active`, `resolved`, or `redeemable` |
| `result` | string | `won`, `lost`, or `null` (if active) |
| `averagePrice` | string | Average entry price |
| `currentPrice` | string | Current market price (or 1.0/0.0 if resolved) |
| `currentValue` | string | Current position value in USDC |
| `pnl` | string | Unrealized profit/loss |

**Errors:**
| Status | Error | Cause |
|--------|-------|-------|
| 400 | "Invalid venue: 'xxx'. Must be 'polymarket' or 'kalshi'" | Invalid venue parameter |
| 403 | "Kalshi trading is not available in your region" | Geo-blocked (if venue=kalshi) |
| 404 | "User not found" | User doesn't exist |
| 500 | "Failed to get positions. Please try again." | Internal error |

---

### Redemptions

#### Redeem Position

Redeems a resolved position, converting winning tokens back to USDC.

- **Method:** `POST`
- **Path:** `/api/users/:userId/redeem`
- **Auth Required:** Yes (must own user)

**Request Body:**

*For Polymarket:*
```json
{
  "venue": "polymarket",
  "tokenId": "71321045679252212594626385..."
}
```

*For Kalshi:*
```json
{
  "venue": "kalshi",
  "ticker": "PRES-2024-DJT",
  "outcome": "Yes"
}
```

**Request Body Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `venue` | string | Yes | `polymarket` or `kalshi` |
| `tokenId` | string | Polymarket only | Token ID to redeem |
| `ticker` | string | Kalshi only | Market ticker |
| `outcome` | string | Kalshi only | `Yes` or `No` |

**Success Response (200 OK):**
```json
{
  "transactionHash": "0x1234567890abcdef...",
  "venue": "polymarket",
  "tokenId": "71321045679252...",
  "title": "Will Bitcoin reach $100k?",
  "outcome": "Yes",
  "sizeRedeemed": "100.50",
  "result": "won",
  "amountRedeemed": "100.50",
  "status": "completed"
}
```

**Response Fields:**
| Field | Type | Description |
|-------|------|-------------|
| `transactionHash` | string | Blockchain transaction hash |
| `venue` | string | Trading venue |
| `tokenId` / `ticker` | string | Market identifier |
| `title` | string | Market question |
| `outcome` | string | Position side that was redeemed |
| `sizeRedeemed` | string | Number of tokens burned |
| `result` | string | `won` or `lost` |
| `amountRedeemed` | string | USDC received |
| `status` | string | Always `completed` |

**Errors:**
| Status | Error | Cause |
|--------|-------|-------|
| 400 | "Missing required fields: tokenId" | Missing Polymarket tokenId |
| 400 | "Missing required fields: ticker, outcome" | Missing Kalshi params |
| 400 | "Market is not resolved yet (status: active)" | Market still trading |
| 400 | "No tokens to redeem" | User has no position |
| 400 | "Redemption not available (status: closed)" | Redemption window closed |
| 403 | "Kalshi trading is not available in your region" | Geo-blocked (if venue=kalshi) |
| 500 | "Redemption failed. Please try again." | Internal error |

---

### Withdrawals

#### Withdraw USDC

Withdraws USDC from the user's wallet to an external address.

- **Method:** `POST`
- **Path:** `/api/users/:userId/withdraw`
- **Auth Required:** Yes (must own user)

**Request Body:**
```json
{
  "venue": "polymarket",
  "amount": "100.00",
  "destinationAddress": "0x742d35Cc6634C0532925a3b844Bc454e4438f44e",
  "chain": "polygon"
}
```

**Request Body Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `venue` | string | Yes | `polymarket` or `kalshi` |
| `amount` | string | Yes | USDC amount to withdraw |
| `destinationAddress` | string | Yes | Recipient wallet address |
| `chain` | string | Yes | `polygon` or `solana` |

**Success Response (200 OK):**
```json
{
  "transactionHash": "0x1234567890abcdef...",
  "amount": "100.00",
  "destinationAddress": "0x742d35Cc6634C0532925a3b844Bc454e4438f44e",
  "chain": "polygon",
  "status": "completed"
}
```

**Errors:**
| Status | Error | Cause |
|--------|-------|-------|
| 400 | "Missing required fields: venue, amount, destinationAddress, chain" | Missing params |
| 400 | "Polymarket withdrawals must use polygon chain" | Venue/chain mismatch |
| 400 | "Kalshi withdrawals must use solana chain" | Venue/chain mismatch |
| 400 | "Invalid Polygon destination address (must be 0x...)" | Bad Polygon address |
| 400 | "Invalid Solana destination address" | Bad Solana address |
| 403 | "Kalshi trading is not available in your region" | Geo-blocked (if venue=kalshi) |
| 500 | "Withdrawal failed. Please try again." | Insufficient balance or network error |

---

#### Withdraw SOL

Withdraws native SOL from the user's Kalshi (Solana) wallet to an external address. User pays their own transaction fees.

- **Method:** `POST`
- **Path:** `/api/users/:userId/withdraw/withdraw-sol`
- **Auth Required:** Yes (must own user)

**Request Body:**
```json
{
  "destinationAddress": "7BfknMUrXjZ96swfM4uLzDXi9QgtcdjQKiWcG4QxW2So",
  "amount": "0.05"
}
```

**Request Body Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `destinationAddress` | string | Yes | Recipient Solana wallet address |
| `amount` | string | Yes | SOL amount to withdraw (e.g., "0.05") |

**Success Response (200 OK):**
```json
{
  "success": true,
  "transactionHash": "5KtP9LMv...",
  "amount": "0.05",
  "destinationAddress": "7BfknMUrXjZ96swfM4uLzDXi9QgtcdjQKiWcG4QxW2So"
}
```

**Errors:**
| Status | Error | Cause |
|--------|-------|-------|
| 400 | "Missing required fields: destinationAddress, amount" | Missing params |
| 500 | "User has no Kalshi wallet" | User doesn't have a Solana wallet |
| 500 | "Invalid Solana destination address" | Invalid address format |
| 500 | "Insufficient SOL balance..." | Not enough SOL (including fee) |

---

### Bridge (Cross-Chain Deposits)

#### Get Deposit Info

Returns deposit addresses for funding a Polymarket wallet from other chains (Ethereum, Solana, Bitcoin, etc.).

- **Method:** `GET`
- **Path:** `/api/bridge/deposit`
- **Auth Required:** Yes

**Query Parameters:**
| Name | Type | Required | Description |
|------|------|----------|-------------|
| `wallet` | string | Yes | Polymarket proxy wallet address (0x...) |

**Success Response (200 OK):**
```json
{
  "wallet": "0x1234567890abcdef1234567890abcdef12345678",
  "depositAddresses": {
    "evm": "0xabcd...1234",
    "solana": "7BfknMUrXjZ96swfM4uLzDXi9QgtcdjQKiWcG4QxW2So",
    "bitcoin": "bc1q..."
  },
  "supportedAssets": [
    {
      "chain": "Ethereum",
      "chainId": "1",
      "token": "USDC",
      "tokenAddress": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
      "decimals": 6,
      "minDepositUsd": 5
    },
    {
      "chain": "Solana",
      "chainId": "solana",
      "token": "USDC",
      "tokenAddress": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "decimals": 6,
      "minDepositUsd": 5
    }
  ]
}
```

**Response Fields:**
| Field | Type | Description |
|-------|------|-------------|
| `wallet` | string | The queried wallet address |
| `depositAddresses.evm` | string | Address for EVM chain deposits (ETH, Arbitrum, etc.) |
| `depositAddresses.solana` | string | Address for Solana deposits |
| `depositAddresses.bitcoin` | string | Address for Bitcoin deposits |
| `supportedAssets` | array | List of supported tokens and chains |
| `supportedAssets[].chain` | string | Chain name |
| `supportedAssets[].chainId` | string | Chain identifier |
| `supportedAssets[].token` | string | Token symbol |
| `supportedAssets[].tokenAddress` | string | Token contract address |
| `supportedAssets[].decimals` | number | Token decimal places |
| `supportedAssets[].minDepositUsd` | number | Minimum deposit in USD |

**Errors:**
| Status | Error | Cause |
|--------|-------|-------|
| 400 | "Missing required parameter: wallet" | No wallet provided |
| 400 | "Invalid wallet address format. Must be a valid Ethereum address (0x...)" | Wallet is not a valid Ethereum address |
| 500 | "Failed to get deposit info. [details]" | Bridge API error |

**Usage Notes:**
- Use the user's `polymarketWalletAddress` from the Create User response
- Deposits are automatically bridged to USDC on Polygon
- Minimum deposits vary by chain/token (check `minDepositUsd`)
- Deposit addresses are cached for 15 minutes

---

## Data Models

### User

```typescript
interface UserInfo {
  userId: string;              // UUID v4
  createdAt: string;           // ISO 8601 timestamp
  polymarketWalletAddress: string | null;  // Gnosis Safe address
  kalshiWalletAddress: string | null;      // Solana wallet address
}
```

### Order

```typescript
interface OrderResponse {
  orderId: string;
  venue: string;
  tokenId?: string;       // Polymarket only
  ticker?: string;        // Kalshi only
  side: string;           // "buy" | "sell"
  outcome: string;
  amount: string;
  price: string;
  status: string;
  createdAt: string;
}
```

### Position

```typescript
interface PositionInfo {
  venue: string;
  tokenId?: string;       // Polymarket only
  ticker?: string;        // Kalshi only
  title: string | null;
  outcome: string;
  size: string;
  status: "active" | "resolved" | "redeemable";
  result: "won" | "lost" | null;
  averagePrice: string | null;
  currentPrice: string | null;
  currentValue: string | null;
  pnl: string | null;
}
```

### Balance

```typescript
interface BalanceInfo {
  venue: string;
  available: string;
  locked: string;
  total: string;
}
```

---

## Concepts

### Custody Model

The custodial API manages wallets on behalf of users:

- **Polymarket (Polygon):** Each user gets a Turnkey-managed signing wallet + Gnosis Safe proxy. The Safe address is the deposit address.
- **Kalshi (Solana):** Each user gets a Turnkey-managed Solana wallet. Trading uses DFlow atomic swaps.

### Order Types

**Polymarket:**
- **Limit Orders:** Placed on the CLOB orderbook at a specific price
- **Market Orders:** Execute immediately at best available price

**Kalshi (via DFlow):**
- All orders are atomic swaps - they execute immediately or fail
- No persistent orderbook presence

### Position Lifecycle

```
┌────────────┐      Market       ┌────────────┐     Redemption    ┌─────────────┐
│   ACTIVE   │ ───  Settles  ─── │  RESOLVED  │ ───   Opens   ─── │ REDEEMABLE  │
└────────────┘                   └────────────┘                   └─────────────┘
     │                                 │                                │
 Can trade                        Outcome known                   Can redeem
 (buy/sell)                       (won/lost)                      for USDC
```

| Status | Description | Actions Available |
|--------|-------------|-------------------|
| `active` | Market is trading | Buy, Sell |
| `resolved` | Market settled, outcome determined | Wait for redemption |
| `redeemable` | Redemption window open | Redeem for USDC |

### Fee Structure

| Venue | Trading Fees | Gas Fees |
|-------|--------------|----------|
| **Polymarket** | Per CLOB fee schedule | None (gasless relay - no POL needed) |
| **Kalshi** | Included in DFlow swap | User pays SOL |

**Polymarket Gasless Relay:** Transactions are submitted through Polymarket's relay infrastructure. Users don't need POL (Polygon's native token) in their wallets.

**Kalshi Gas:** Users need SOL in their Solana wallet for transaction fees.

---

## Code Examples

### Create User and Place Order (Python)

```python
import requests

BASE_URL = "https://predexon-trade-api-8721254678.europe-north2.run.app"
HEADERS = {"x-api-key": "your-api-key"}

# Create user
response = requests.post(f"{BASE_URL}/api/users/create", headers=HEADERS)
user = response.json()
user_id = user["userId"]

print(f"Created user: {user_id}")
print(f"Deposit USDC to: {user['polymarketWalletAddress']}")

# Place order (after depositing)
order = {
    "venue": "polymarket",
    "tokenId": "71321045679252212594626385...",
    "side": "buy",
    "amount": "50.00",
    "price": "0.65"
}
response = requests.post(
    f"{BASE_URL}/api/users/{user_id}/orders",
    headers=HEADERS,
    json=order
)
print(f"Order placed: {response.json()}")
```

### Get Positions (cURL)

```bash
curl -X GET \
  "https://predexon-trade-api-8721254678.europe-north2.run.app/api/users/USER_ID/positions" \
  -H "x-api-key: your-api-key"
```

### Place Kalshi Order (TypeScript)

```typescript
const BASE_URL = "https://predexon-trade-api-8721254678.europe-north2.run.app";

async function placeKalshiOrder(userId: string, apiKey: string) {
  const response = await fetch(`${BASE_URL}/api/users/${userId}/orders`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "x-api-key": apiKey,
    },
    body: JSON.stringify({
      venue: "kalshi",
      ticker: "PRES-2024-DJT",
      outcome: "Yes",
      side: "buy",
      amount: "25.00",
    }),
  });

  return response.json();
}
```

---

## Important Notes

### Polymarket vs Kalshi Differences

| Feature | Polymarket | Kalshi |
|---------|------------|--------|
| Blockchain | Polygon | Solana |
| Order Type | Limit/Market on CLOB | Atomic swaps via DFlow |
| Market Identifier | `tokenId` | `ticker` + `outcome` |
| Gas Fees | Gasless relay | User pays SOL |
| Cancel Orders | Yes | No (atomic execution) |

### Error Handling

- All endpoints return JSON error responses: `{"error": "message"}`
- 400 errors include helpful details about missing/invalid fields
- 500 errors are generic - check server logs for details

### Best Practices

1. **Check balance before ordering** - Prevents failed transactions
2. **Poll positions for updates** - Markets resolve asynchronously
3. **Handle rate limits** - Implement exponential backoff
4. **Store user IDs** - Required for all subsequent operations
