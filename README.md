# StackCast - Decentralized Prediction Market on Stacks

A production-ready prediction market platform built on Stacks blockchain, featuring a Polymarket-style CLOB (Central Limit Order Book) architecture with optimistic oracle resolution and ECDSA signature verification. Users bet with **real sBTC** (Bitcoin-backed tokens) as collateral.

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      USER WALLETS                               │
│  • Hold sBTC (Bitcoin-backed collateral)                        │
│  • Hold YES/NO position tokens (ERC-1155 style)                 │
│  • Sign orders with ECDSA secp256k1 (same as Bitcoin)           │
└────────────────────┬────────────────────────────────────────────┘
                     │ HTTP (signed orders)
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                 CLOB API (Off-chain Matching)                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Smart Order Router                                     │   │
│  │  • MARKET: Multi-level execution with slippage check    │   │
│  │  • LIMIT: Single-price placement                        │   │
│  │  • Execution preview before placement                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Matching Engine (100ms intervals)                      │   │
│  │  • Price-time priority matching                         │   │
│  │  • Continuous order matching loop                       │   │
│  │  • Automatic trade creation                             │   │
│  └─────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Redis Storage (Upstash/Local)                          │   │
│  │  • Order book persistence                               │   │
│  │  • Market data & stats                                  │   │
│  │  • Trade history                                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Signature Verification                                 │   │
│  │  • ECDSA secp256k1 (Bitcoin's crypto)                   │   │
│  │  • Public key recovery from signature                   │   │
│  │  • Prevents order forgery                               │   │
│  └─────────────────────────────────────────────────────────┘   │
└────────────────────┬────────────────────────────────────────────┘
                     │ (matched orders + signatures)
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│            CTFExchange (On-chain Settlement)                    │
│  • Verifies ECDSA signatures on-chain (secp256k1-recover?)      │
│  • Executes atomic swaps (YES ↔ NO tokens)                     │
│  • 0.5% protocol trading fee                                    │
│  • No liquidity pool - pure P2P matching                        │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│         ConditionalTokens (Token Registry & Escrow)             │
│  • ERC-1155 style position token balances                       │
│  • Split: sBTC → YES+NO tokens (deposit collateral)            │
│  • Merge: YES+NO → sBTC (withdraw collateral)                  │
│  • Redeem: Winning tokens → sBTC (after resolution)            │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│            Oracle Adapter + Optimistic Oracle                   │
│  • Market initialization with condition IDs                     │
│  • UMA-style optimistic resolution (24hr dispute window)        │
│  • Dispute resolution with token-weighted voting                │
│  • Final answer reporting to CTF                                │
└─────────────────────────────────────────────────────────────────┘
```

## 🔐 Security: ECDSA secp256k1 Signature Verification

StackCast uses **ECDSA (Elliptic Curve Digital Signature Algorithm)** with the **secp256k1** curve - the same cryptographic protocol that secures Bitcoin!

### **How It Works**

#### **1. Order Signing (Frontend)**
```typescript
// User's wallet signs the order hash
const orderHash = SHA256(
  maker + taker +
  makerPositionId + takerPositionId +
  makerAmount + takerAmount +
  salt + expiration
)
const signature = wallet.sign(orderHash) // → 65-byte RSV signature
```

**Signature Format (65 bytes):**
- **R** (32 bytes): x-coordinate of ephemeral public key
- **S** (32 bytes): Signature proof value
- **V** (1 byte): Recovery ID (0-3) enables public key recovery

#### **2. Backend Verification**
```typescript
// Recover public key from signature
const recoveredPubKey = secp256k1.recover(orderHash, signature)
const isValid = (recoveredPubKey === expectedPublicKey)
```

#### **3. On-Chain Verification (Clarity Smart Contract)**
```clarity
;; Contract verifies signature again during settlement
(secp256k1-recover? order-hash signature)
  → recovered-pubkey
    → (principal-of? recovered-pubkey)
      → Check if matches expected signer
```

### **Why ECDSA secp256k1?**

✅ **Bitcoin-Compatible**: Same crypto as Bitcoin transactions
✅ **Public Key Recovery**: No need to store public keys on-chain
✅ **Compact**: Only 65 bytes vs 128+ for other schemes
✅ **Proven Security**: 15+ years securing $1+ trillion in Bitcoin
✅ **Native Clarity Support**: Built-in `secp256k1-recover?` function

### **Security Properties**

- **Non-Repudiation**: Only private key holder can create valid signature
- **Tamper-Proof**: Changing 1 bit in order → signature becomes invalid
- **Replay Protection**: Salt + expiration prevent signature reuse
- **Double Verification**: Backend AND contract both verify (defense in depth)

## 💰 How Token Flows Work

### Collateral: sBTC (Bitcoin-Backed Tokens)

StackCast uses **sBTC** as the collateral token - a 1:1 Bitcoin-backed fungible token on Stacks. This means users bet with real Bitcoin value that can be unwrapped to actual BTC.

**Why sBTC?**

- 🔗 Backed 1:1 by Bitcoin
- 💰 Real value, not test tokens
- 🏦 Can trade on Stacks DEXes
- 🔄 Unwrap to native Bitcoin
- ✅ Already deployed on mainnet

### The Conditional Token Framework (CTF)

Users interact with sBTC through three core operations:

#### 1. **Split Position** (Deposit sBTC → Get YES/NO tokens)

```clarity
;; User deposits real sBTC into the contract
(contract-call? SBTC-TOKEN transfer
  collateral-amount
  tx-sender
  (as-contract tx-sender)
  none
)
;; Contract mints YES + NO position tokens (equal amounts)
```

**Example:**

```
User deposits:  100 sBTC (≈ $7,000 at $70k/BTC)
Contract locks: 100 sBTC in escrow
User receives:  100 YES + 100 NO tokens
```

#### 2. **Merge Positions** (Burn YES+NO → Get sBTC back)

```clarity
;; User burns equal YES + NO tokens
;; Contract returns real sBTC from escrow
(as-contract (contract-call? SBTC-TOKEN transfer
  collateral-amount
  tx-sender
  recipient
  none
))
```

**Example:**

```
User burns:       50 YES + 50 NO tokens
Contract returns: 50 sBTC (unlocks from escrow)
```

#### 3. **Redeem** (After resolution, winning tokens → sBTC)

```clarity
;; After oracle resolves market (e.g., YES wins)
;; Burn YES tokens, get sBTC payout
(as-contract (contract-call? SBTC-TOKEN transfer
  payout-amount
  tx-sender
  recipient
  none
))
```

**Example:**

```
Market resolves: YES wins
User holds:      100 YES tokens
User redeems:    100 sBTC (original collateral)
User with NO:    Gets nothing (tokens burned, value lost)
```

### sBTC Across Networks

| Network              | sBTC Address                                           | How to Get              |
| -------------------- | ------------------------------------------------------ | ----------------------- |
| **Simnet** (testing) | `ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.sbtc-token` | Auto-funded by Clarinet |
| **Devnet** (local)   | `ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.sbtc-token` | Auto-funded by Clarinet |
| **Testnet**          | `SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token` | Faucet                  |
| **Mainnet**          | `SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token` | Bridge from BTC         |

**Clarinet Magic:** Reference the simnet address in your code, and Clarinet automatically remaps it during testnet/mainnet deployment.

## 📦 Project Structure

```
stackcast/
├── stackcast-contracts/     # Clarity smart contracts (Stacks)
│   ├── contracts/
│   │   ├── sip-010-trait.clar          # SIP-010 fungible token standard
│   │   ├── conditional-tokens.clar      # Core CTF - splits sBTC into YES/NO
│   │   ├── ctf-exchange.clar            # Settlement layer with ECDSA verification
│   │   ├── optimistic-oracle.clar       # UMA-style optimistic oracle
│   │   └── oracle-adapter.clar          # Connects oracle to CTF
│   ├── tests/                           # Clarinet tests (vitest)
│   └── Clarinet.toml                    # Clarinet config (includes sBTC requirement)
│
├── server/                  # TypeScript CLOB API (Bun)
│   ├── src/
│   │   ├── index.ts                     # Express server with CORS
│   │   ├── types/
│   │   │   ├── order.ts                 # Order, Trade, Market, OrderType enums
│   │   │   └── express.d.ts             # Express augmentation
│   │   ├── services/
│   │   │   ├── redisClient.ts           # Redis connection (local/Upstash)
│   │   │   ├── orderManagerRedis.ts     # Redis-based order storage & indexing
│   │   │   ├── matchingEngine.ts        # Price-time priority matching (100ms)
│   │   │   ├── smartRouter.ts           # Multi-level execution planner
│   │   │   ├── stacksMonitor.ts         # Block height monitoring & auto-expiration
│   │   │   └── stacksSettlement.ts      # On-chain trade settlement
│   │   ├── routes/
│   │   │   ├── markets.ts               # Market CRUD & stats
│   │   │   ├── smartOrders.ts           # LIMIT/MARKET order placement with sig verification
│   │   │   ├── orderbook.ts             # Orderbook, trades, price feeds
│   │   │   └── oracle.ts                # Oracle resolution endpoints
│   │   └── utils/
│   │       └── signatureVerification.ts # ECDSA secp256k1 verification
│   └── package.json
│
└── web/                     # Frontend (React + Vite + TypeScript)
    ├── src/
    │   ├── App.tsx                      # Main app router
    │   ├── contexts/
    │   │   └── WalletContext.tsx        # Stacks wallet integration (@stacks/connect)
    │   ├── api/
    │   │   ├── client.ts                # Base API client
    │   │   └── queries/                 # React Query hooks
    │   ├── pages/
    │   │   ├── Markets.tsx              # Market list view
    │   │   ├── MarketDetail.tsx         # Trading interface with auto split/merge
    │   │   ├── Portfolio.tsx            # User positions & orders
    │   │   ├── Oracle.tsx               # Oracle proposal/voting
    │   │   └── Redeem.tsx               # Redeem winning positions
    │   ├── components/
    │   │   └── ui/                      # Radix UI components (shadcn)
    │   ├── lib/
    │   │   ├── config.ts                # Network & contract configs
    │   │   └── utils.ts                 # Utility functions
    │   └── utils/
    │       ├── orderSigning.ts          # ECDSA order hash computation & signing
    │       └── stacksHelpers.ts         # Contract interaction utilities
    └── package.json
```

## 🚀 Quick Start

### Prerequisites

- [Clarinet](https://github.com/hirosystems/clarinet) (for Stacks contracts)
- [Bun](https://bun.sh/) (for backend)
- [Redis](https://redis.io/) (for order storage) OR [Upstash Redis](https://upstash.com/)
- [Node.js](https://nodejs.org/) 18+ (for frontend)
- Stacks wallet ([Leather](https://leather.io/) or [Xverse](https://www.xverse.app/))

### 1. Smart Contracts

```bash
cd stackcast-contracts

# Check all contracts compile
clarinet check

# Run tests (auto-funds wallets with sBTC)
clarinet test

# Deploy to devnet (local)
clarinet devnet start

# Deploy to testnet
clarinet deploy --testnet
```

**Note:** `Clarinet.toml` includes sBTC as a requirement:

```toml
[project]
requirements = [
  { contract_id = "ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.sbtc-token" }
]
```

This means Clarinet automatically:

- Downloads the sBTC contract
- Funds test wallets with sBTC in simnet/devnet
- Remaps addresses for testnet/mainnet deployment

### 2. Backend API

```bash
cd server

# Install dependencies
bun install

# Start Redis (Option 1: Local Docker)
docker run -d -p 6379:6379 redis:latest

# OR use Upstash Redis (Option 2: Cloud)
# Set REDIS_URL in .env to your Upstash Redis URL

# Set up environment
cp .env.example .env
# Edit .env with your contract addresses after deployment

# Run development server (with hot reload)
bun run dev

# Production
bun start
```

Server runs on `http://localhost:3000`

The backend includes:

- **REST API**: Markets, smart orders, orderbook endpoints
- **Matching Engine**: Runs every 100ms, price-time priority matching
- **Smart Router**: Multi-level execution planning for MARKET orders
- **Stacks Monitor**: Tracks block height, expires old orders automatically
- **Redis Storage**: Persistent order book and market data
- **Signature Verification**: ECDSA secp256k1 validation

### 3. Frontend Web App

```bash
cd web

# Install dependencies
npm install

# Set up environment
cp .env.example .env
# Configure VITE_API_BASE_URL and VITE_STACKS_NETWORK

# Run development server
npm run dev

# Build for production
npm run build
```

App runs on `http://localhost:5173`

Features:

- **WalletContext**: React context for Stacks wallet integration
- **Order Signing**: ECDSA secp256k1 message signing for order authentication
- **Smart Order Placement**: LIMIT and MARKET order types
- **Auto Split/Merge**: Automatically deposits/withdraws sBTC as needed
- **Real-time Execution Preview**: See how order would execute before placing
- **Position Balance Checking**: Verifies sufficient balance before trading
- **Network Support**: Devnet, testnet, mainnet configurations

### 4. Test the API

```bash
# Check health
curl http://localhost:3000/health

# Get markets
curl http://localhost:3000/api/markets

# Preview MARKET order execution (no placement)
curl -X POST http://localhost:3000/api/smart-orders/preview \
  -H "Content-Type: application/json" \
  -d '{
    "marketId": "market1",
    "outcome": "yes",
    "side": "BUY",
    "orderType": "MARKET",
    "size": 100
  }'
```

## 📡 API Documentation

### Markets

#### Create Market

```bash
POST /api/markets
Content-Type: application/json

{
  "question": "Will BTC hit $100k by Dec 31, 2025?",
  "creator": "ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM",
  "conditionId": "0x..."
}
```

#### Get All Markets

```bash
GET /api/markets
```

#### Get Market Details

```bash
GET /api/markets/:marketId
```

#### Get Market Stats

```bash
GET /api/markets/:marketId/stats
```

Returns:
```json
{
  "success": true,
  "stats": {
    "totalVolume": 150000,
    "totalTrades": 45,
    "openOrders": 12,
    "lastPrice": 66,
    "priceChange24h": 2.5
  }
}
```

### Smart Orders (LIMIT & MARKET)

#### Preview Order Execution (No Placement)

```bash
POST /api/smart-orders/preview
Content-Type: application/json

{
  "marketId": "market1",
  "outcome": "yes",        # "yes" or "no"
  "side": "BUY",           # "BUY" or "SELL"
  "orderType": "MARKET",   # "MARKET" or "LIMIT"
  "size": 500,
  "maxSlippage": 5         # For MARKET: max acceptable slippage %
}
```

Response:
```json
{
  "success": true,
  "plan": {
    "feasible": true,
    "orderType": "MARKET",
    "totalSize": 500,
    "levels": [
      { "price": 65, "size": 200, "cost": 130000, "cumulativeSize": 200 },
      { "price": 66, "size": 300, "cost": 198000, "cumulativeSize": 500 }
    ],
    "averagePrice": 65.6,
    "totalCost": 328000,
    "slippage": 1.54,
    "bestPrice": 65,
    "worstPrice": 66
  }
}
```

#### Place Order (LIMIT or MARKET)

```bash
POST /api/smart-orders
Content-Type: application/json

# LIMIT Order Example
{
  "maker": "ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM",
  "marketId": "market1",
  "outcome": "yes",
  "side": "BUY",
  "orderType": "LIMIT",
  "size": 100,
  "price": 510000,         # In micro-sats (0.51 sBTC per token)
  "salt": "1234567890",
  "expiration": 999999999,
  "signature": "0x...",    # 65-byte ECDSA signature
  "publicKey": "0x..."     # 33-byte compressed public key
}

# MARKET Order Example
{
  "maker": "ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM",
  "marketId": "market1",
  "outcome": "yes",
  "side": "BUY",
  "orderType": "MARKET",
  "size": 500,
  "maxSlippage": 5,        # Max 5% slippage
  "salt": "1234567890",
  "expiration": 999999999,
  "signature": "0x...",
  "publicKey": "0x..."
}
```

Response (MARKET):
```json
{
  "success": true,
  "orderType": "MARKET",
  "orders": [
    { "orderId": "ord_abc", "price": 65, "size": 200 },
    { "orderId": "ord_def", "price": 66, "size": 300 }
  ],
  "executionPlan": {
    "averagePrice": 65.6,
    "totalCost": 328000,
    "slippage": 1.54
  }
}
```

Response (LIMIT):
```json
{
  "success": true,
  "orderType": "LIMIT",
  "order": {
    "orderId": "ord_xyz",
    "price": 66,
    "size": 100,
    "status": "OPEN"
  }
}
```

### Orderbook

#### Get Orderbook

```bash
GET /api/orderbook/:marketId?outcome=yes
```

Returns:
```json
{
  "success": true,
  "orderbook": {
    "outcome": "yes",
    "bids": [
      { "price": 66, "size": 5000, "orderCount": 3 },
      { "price": 64, "size": 2000, "orderCount": 1 }
    ],
    "asks": [
      { "price": 68, "size": 3000, "orderCount": 2 },
      { "price": 70, "size": 1000, "orderCount": 1 }
    ]
  }
}
```

#### Get Recent Trades

```bash
GET /api/orderbook/:marketId/trades?limit=50
```

#### Get Market Price

```bash
GET /api/orderbook/:marketId/price?outcome=yes
```

Returns:
```json
{
  "success": true,
  "price": {
    "bid": 66,
    "ask": 68,
    "mid": 67,
    "last": 67
  }
}
```

### Health Check

```bash
GET /health
```

## 🎯 Key Features

### 1. Conditional Tokens Framework (CTF)

- **Split**: `100 sBTC → 100 YES + 100 NO` (deposits collateral)
- **Merge**: `100 YES + 100 NO → 100 sBTC` (withdraws collateral)
- **Transfer**: P2P token transfers via ERC-1155 style balances
- **Redeem**: After resolution, `100 YES (if YES won) → 100 sBTC`

### 2. Smart Order Router

**MARKET Orders:**
- Executes across multiple price levels
- Calculates average execution price and slippage
- Checks liquidity before placement
- Splits into multiple LIMIT orders at different prices

**LIMIT Orders:**
- Single price placement
- Joins order book at specified price
- Waits for matching counterparty

**Execution Preview:**
- Shows exactly how order would execute
- Displays price levels, sizes, and costs
- Calculates slippage and feasibility
- Updates in real-time as user types

### 3. CLOB Matching Engine

- **Price-Time Priority**: Best price first, then FIFO (First In First Out)
- **Matching Frequency**: Every 100ms continuous loop
- **Atomic Swaps**: Matched orders execute atomically on-chain
- **Fee**: 0.5% trading fee collected by protocol

### 4. Optimistic Oracle

- **Propose**: Anyone can propose an answer (requires bond: 100 tokens)
- **Dispute**: 24-hour challenge window (144 blocks)
- **Vote**: If disputed, token holders vote (48-hour voting period)
- **Resolve**: Final answer reported to CTF for redemption

### 5. Oracle Adapter

- **Market Creation**: Initializes both CTF condition and oracle question
- **Resolution**: Fetches oracle result and reports to CTF
- **Lifecycle Management**: Coordinates between oracle and token system

## 🔐 Security Features

### Cryptographic Security
- ✅ **ECDSA secp256k1 signatures** (Bitcoin's proven crypto)
- ✅ **Public key recovery** verification (backend + on-chain)
- ✅ **SHA-256 order hashing** (tamper-proof)
- ✅ **RSV signature format** (65-byte compact signatures)

### Order Security
- ✅ **Salt-based replay protection** (prevent signature reuse)
- ✅ **Order expiration by block height** (auto-expire old orders)
- ✅ **Maker-only order cancellation** (only order creator can cancel)
- ✅ **Signature verification** (frontend signs, backend + contract verify)

### Protocol Security
- ✅ **Emergency pause functionality** (owner can pause trading)
- ✅ **Approval system for token transfers** (ERC-1155 style approvals)
- ✅ **Bond-based oracle security** (proposers stake tokens)
- ✅ **Dispute resolution** (token-weighted voting)

## 📊 Example Flow: Creating and Trading a Market

### 1. Create Market

```bash
curl -X POST http://localhost:3000/api/markets \
  -H "Content-Type: application/json" \
  -d '{
    "question": "Will ETH reach $10k in 2025?",
    "creator": "ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM"
  }'
```

### 2. Market Maker Provides Liquidity

**First, split sBTC into YES+NO tokens on-chain:**

```clarity
;; User deposits 10,000 sBTC
;; Contract mints 10,000 YES + 10,000 NO
(contract-call? .conditional-tokens split-position
  u10000000000  ;; 10,000 sBTC in micro-sats
  condition-id)
```

**Then, place LIMIT orders on CLOB:**

```bash
# Sell YES at 55¢
curl -X POST http://localhost:3000/api/smart-orders \
  -H "Content-Type: application/json" \
  -d '{
    "maker": "ST2CY5V39NHDPWSXMW9QDT3HC3GD6Q6XX4CFRK9AG",
    "marketId": "market_abc123",
    "outcome": "yes",
    "side": "SELL",
    "orderType": "LIMIT",
    "price": 550000,
    "size": 10000,
    "salt": "123456",
    "expiration": 999999,
    "signature": "0x...",
    "publicKey": "0x..."
  }'

# Sell NO at 45¢ (YES + NO = 100¢)
curl -X POST http://localhost:3000/api/smart-orders \
  -H "Content-Type: application/json" \
  -d '{
    "maker": "ST2CY5V39NHDPWSXMW9QDT3HC3GD6Q6XX4CFRK9AG",
    "marketId": "market_abc123",
    "outcome": "no",
    "side": "SELL",
    "orderType": "LIMIT",
    "price": 450000,
    "size": 10000,
    "salt": "123457",
    "expiration": 999999,
    "signature": "0x...",
    "publicKey": "0x..."
  }'
```

### 3. Trader Buys YES with MARKET Order

```bash
curl -X POST http://localhost:3000/api/smart-orders \
  -H "Content-Type: application/json" \
  -d '{
    "maker": "ST2JHG361ZXG51QTKY2NQCVBPPRRE2KZB1HR05NNC",
    "marketId": "market_abc123",
    "outcome": "yes",
    "side": "BUY",
    "orderType": "MARKET",
    "size": 1000,
    "maxSlippage": 2,
    "salt": "789012",
    "expiration": 999999,
    "signature": "0x...",
    "publicKey": "0x..."
  }'
```

### 4. Matching Engine Automatically Matches

- Within 100ms, matching engine runs
- Finds SELL order at 55¢
- Creates trade: 1000 YES @ 55¢
- Fee collected: 0.5% = 2.75 sBTC
- Settlement service broadcasts to Stacks blockchain
- Contract verifies signatures and executes atomic swap

### 5. Market Resolves: ETH hits $10k (YES wins!)

**Oracle resolution:**

```clarity
(contract-call? .optimistic-oracle propose-answer question-id u1) ;; 1 = YES
;; Wait 24 hours (144 blocks)...
(contract-call? .optimistic-oracle resolve question-id)
(contract-call? .oracle-adapter resolve-market market-id)
```

**Winners redeem:**

```clarity
;; Trader redeems 1000 YES tokens
(contract-call? .conditional-tokens redeem-positions
  condition-id
  u0)  ;; outcome index 0 = YES
;; Receives: 1000 sBTC (can unwrap to real BTC!)
```

**Market maker:**

```clarity
;; Still holds 9000 YES tokens (sold 1000)
;; Redeems for: 9000 sBTC
;; Lost: The 1000 sBTC from sold YES tokens
;; Net: Made profit from selling NO tokens
```

## 🧪 Testing

### Contract Tests

```bash
cd stackcast-contracts
clarinet test
```

Tests verify:

- ✅ Real sBTC transfers (deposit, withdrawal, redemption)
- ✅ Split/merge logic with proper collateral escrow
- ✅ ECDSA signature verification (secp256k1-recover?)
- ✅ Oracle resolution and dispute flow
- ✅ Exchange settlement with fee collection

### Backend API Tests

```bash
cd server

# Test matching engine
bun test

# Test signature verification
bun run scripts/test-signatures.ts

# Full integration test
bun run scripts/test-api.ts
```

## 🚢 Deployment

### Testnet Deployment

1. **Update Clarinet.toml** with your deployer address
2. **Deploy contracts**:

```bash
cd stackcast-contracts
clarinet deploy --testnet
```

3. **Set up Redis** (choose one):

```bash
# Option 1: Local Redis with Docker
docker run -d -p 6379:6379 redis:latest

# Option 2: Upstash Redis (cloud)
# Get Redis URL from https://upstash.com/
export REDIS_URL="redis://...@upstash.io:6379"
```

4. **Update server .env** with deployed contract addresses:

```env
STACKS_NETWORK=testnet
CTF_EXCHANGE_ADDRESS=ST1234...ctf-exchange
REDIS_URL=redis://localhost:6379  # or Upstash URL
```

5. **Start backend**:

```bash
cd server
bun start
```

6. **Deploy frontend** (Vercel/Netlify):

```bash
cd web
npm run build
# Deploy dist/ folder to your hosting provider
```

### Mainnet Deployment

Same as testnet, but use `--mainnet` flag and update network config to `mainnet`.

## 🛠️ Tech Stack

### Blockchain
- **Layer 2**: Stacks (Bitcoin L2)
- **Smart Contracts**: Clarity language
- **Collateral**: sBTC (1:1 Bitcoin-backed)
- **Token Standard**: SIP-010 (fungible tokens)

### Backend
- **Runtime**: Bun (fast TypeScript runtime)
- **Framework**: Express.js
- **Storage**: Redis (Upstash or local)
- **Crypto**: @stacks/encryption (ECDSA secp256k1)
- **Network**: @stacks/network, @stacks/transactions

### Frontend
- **Framework**: React 19
- **Build Tool**: Vite
- **State**: React Query (@tanstack/react-query)
- **Routing**: React Router
- **Wallet**: @stacks/connect (Leather/Xverse)
- **UI**: Radix UI + Tailwind CSS
- **Icons**: Lucide React

### Testing
- **Contracts**: Clarinet + Vitest
- **Backend**: Bun test
- **E2E**: Manual testing with real wallets

## 📝 Roadmap

### Smart Contracts ✅

- ✅ Conditional Tokens (sBTC collateral)
- ✅ CTF Exchange (atomic settlement + ECDSA verification)
- ✅ Optimistic Oracle (dispute resolution)
- ✅ Oracle Adapter (market lifecycle)
- ✅ Full test coverage (integration tests)

### Backend ✅

- ✅ CLOB matching engine (price-time priority, 100ms intervals)
- ✅ Smart Router (multi-level MARKET order execution)
- ✅ Order management (Redis-backed, persistent)
- ✅ REST API (markets, smart orders, orderbook)
- ✅ Stacks block monitor (auto-expires orders)
- ✅ ECDSA secp256k1 signature verification

### Frontend ✅

- ✅ React + Vite + TypeScript + Tailwind
- ✅ Wallet integration (@stacks/connect)
- ✅ ECDSA order signing (wallet message signing)
- ✅ Smart order placement (LIMIT + MARKET)
- ✅ Auto split/merge (insufficient balance handling)
- ✅ Real-time execution preview
- ✅ Network configuration (devnet/testnet/mainnet)
- ✅ Contract interaction utilities
- ✅ UI components (Radix UI)

### To-Do

- [ ] WebSocket for real-time orderbook updates
- [ ] PostgreSQL persistence (replace Redis for production)
- [ ] Market maker bot examples
- [ ] Advanced order types (FOK, IOC, Post-Only)
- [ ] Historical data API & charting
- [ ] TradingView/Lightweight Charts integration
- [ ] Mobile responsive design improvements
- [ ] Monitoring & alerting (DataDog/Sentry)

## 🎯 Production Readiness

| Component                      | Status         | Notes                                              |
| ------------------------------ | -------------- | -------------------------------------------------- |
| **Smart Contracts**            | ✅ Production  | ECDSA verified, real sBTC collateral               |
| **CLOB API**                   | ✅ Production  | Redis-backed, 100ms matching, signature verified   |
| **Smart Router**               | ✅ Production  | MARKET/LIMIT orders, slippage protection           |
| **Frontend**                   | ✅ Production  | Wallet integration, auto split/merge, UI complete  |
| **Signature Verification**     | ✅ Production  | ECDSA secp256k1 (backend + on-chain)               |
| **Tests**                      | ✅ Production  | Contract & integration tests passing               |
| **Deployment**                 | ✅ Ready       | Can deploy to testnet/mainnet                      |
| **Monitoring**                 | ⚠️ Needed      | Add DataDog/Sentry for production                  |
| **Market Maker Bots**          | ⚠️ Needed      | Need example bots for liquidity                    |

### For Production Launch:

- ✅ Smart contracts with ECDSA verification
- ✅ Backend API with Redis & 100ms matching
- ✅ Stacks wallet integration (Leather/Xverse)
- ✅ ECDSA order signing & verification (3 layers)
- ✅ Frontend UI with smart order placement
- ✅ Auto split/merge for insufficient balance
- ⚠️ Need monitoring/alerting
- ⚠️ Need market maker bots for liquidity

## 📄 License

MIT

## 🤝 Contributing

This project is open source. Feel free to fork and improve!

## 🔗 Links

- [Stacks Docs](https://docs.stacks.co)
- [Clarity Language](https://book.clarity-lang.org)
- [Clarinet](https://github.com/hirosystems/clarinet)
- [sBTC Documentation](https://docs.hiro.so/en/tools/clarinet/sbtc-integration)
- [Bun Runtime](https://bun.sh)
- [ECDSA secp256k1](https://en.bitcoin.it/wiki/Secp256k1)
- [Upstash Redis](https://upstash.com/)
