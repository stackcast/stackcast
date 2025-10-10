# Stackcast - Decentralized Prediction Market on Stacks

A production-ready prediction market platform built on Stacks blockchain, featuring a Polymarket-style CLOB (Central Limit Order Book) architecture with optimistic oracle resolution. Users bet with **real sBTC** (Bitcoin-backed tokens) as collateral.

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    USER WALLETS                         │
│  • Hold sBTC (Bitcoin-backed collateral)                │
│  • Hold YES/NO position tokens                          │
│  • Traders place orders via CLOB API                    │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│              CLOB API (Off-chain)                       │
│  • Order book management (Bun TypeScript)               │
│  • Matching engine (100ms intervals)                    │
│  • Order validation                                     │
└────────────────┬────────────────────────────────────────┘
                 │ (matched orders)
                 ▼
┌─────────────────────────────────────────────────────────┐
│         CTFExchange (On-chain Settlement)               │
│  • Executes atomic swaps                                │
│  • 0.5% trading fee                                     │
│  • No liquidity pool - pure P2P                         │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│      ConditionalTokens (Token Registry)                 │
│  • ERC-1155 style balances                              │
│  • Split/Merge logic (sBTC ↔ YES+NO)                   │
│  • Redemption after resolution                          │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│         Oracle Adapter + Optimistic Oracle              │
│  • Market initialization                                │
│  • Resolution via optimistic oracle                     │
│  • Dispute resolution with bonding                      │
└─────────────────────────────────────────────────────────┘
```

## 💰 How Token Flows Work

### Collateral: sBTC (Bitcoin-Backed Tokens)

Stackcast uses **sBTC** as the collateral token - a 1:1 Bitcoin-backed fungible token on Stacks. This means users bet with real Bitcoin value that can be unwrapped to actual BTC.

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
User deposits:  100 sBTC (≈ $7)
Contract locks: 100 sBTC
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
User burns:     50 YES + 50 NO tokens
Contract returns: 50 sBTC
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
User holds: 100 YES tokens
User redeems: 100 sBTC (original collateral)
User with NO: Gets nothing (burned)
```

### sBTC Across Networks

| Network | sBTC Address | How to Get |
|---------|-------------|------------|
| **Simnet** (testing) | `SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token` | Auto-funded by Clarinet |
| **Devnet** (local) | `SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token` | Auto-funded by Clarinet |
| **Testnet** | `ST1F7QA2MDF17S807EPA36TSS8AMEFY4KA9TVGWXT.sbtc-token` | Faucet |
| **Mainnet** | `SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token` | Bridge from BTC |

**Clarinet Magic:** Reference the simnet address in your code, and Clarinet automatically remaps it during testnet/mainnet deployment.

## 📦 Project Structure

```
stackcast/
├── stackcast-contracts/     # Clarity smart contracts (Stacks)
│   ├── contracts/
│   │   ├── sip-010-trait.clar          # SIP-010 fungible token standard
│   │   ├── conditional-tokens.clar      # Core CTF - splits sBTC into YES/NO
│   │   ├── ctf-exchange.clar            # Settlement layer for trades
│   │   ├── optimistic-oracle.clar       # UMA-style optimistic oracle
│   │   └── oracle-adapter.clar          # Connects oracle to CTF
│   ├── tests/                           # Clarinet tests
│   └── Clarinet.toml                    # Clarinet config (includes sBTC requirement)
│
├── backend/                 # TypeScript CLOB API (Bun)
│   ├── src/
│   │   ├── index.ts                     # Express server
│   │   ├── types/order.ts               # Type definitions
│   │   ├── services/
│   │   │   ├── orderManager.ts          # Order storage & indexing
│   │   │   └── matchingEngine.ts        # Price-time priority matching
│   │   └── routes/
│   │       ├── markets.ts               # Market CRUD
│   │       ├── orders.ts                # Order placement/cancellation
│   │       └── orderbook.ts             # Orderbook & trades
│   └── package.json
│
└── web/                     # Frontend (React - not implemented yet)
```

## 🚀 Quick Start

### Prerequisites
- [Clarinet](https://github.com/hirosystems/clarinet) (for Stacks contracts)
- [Bun](https://bun.sh/) (for backend)
- Stacks wallet (for testnet deployment)

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
  { contract_id = "SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token" }
]
```

This means Clarinet automatically:
- Downloads the sBTC contract
- Funds test wallets with sBTC in simnet/devnet
- Remaps addresses for testnet/mainnet deployment

### 2. Backend API

```bash
cd backend

# Install dependencies
bun install

# Set up environment
cp .env.example .env
# Edit .env with your contract addresses after deployment

# Run development server (with hot reload)
bun run dev

# Production
bun start
```

Server runs on `http://localhost:3000`

### 3. Test the API

```bash
# Check health
curl http://localhost:3000/health

# Run full demo
cd backend
bun scripts/test-api.ts
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

### Orders

#### Place Order
```bash
POST /api/orders
Content-Type: application/json

{
  "maker": "ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM",
  "marketId": "market_123",
  "positionId": "market_123_yes",
  "side": "BUY",           // or "SELL"
  "price": 66,              // 0-100 (66 = $0.66)
  "size": 1000,             // token amount
  "salt": "1234567890",
  "expiration": 999999999   // block height
}
```

#### Get Order
```bash
GET /api/orders/:orderId
```

#### Get User Orders
```bash
GET /api/orders/user/:address?status=OPEN&marketId=market_123
```

#### Cancel Order
```bash
DELETE /api/orders/:orderId
Content-Type: application/json

{
  "maker": "ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM"
}
```

### Orderbook

#### Get Orderbook
```bash
GET /api/orderbook/:marketId?positionId=market_123_yes
```

Returns:
```json
{
  "success": true,
  "orderbook": {
    "positionId": "market_123_yes",
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
GET /api/orderbook/:marketId/price
```

### Health Check
```bash
GET /health
```

## 🎯 Key Features

### 1. Conditional Tokens Framework (CTF)
- **Split**: `100 sBTC → 100 YES + 100 NO`
- **Merge**: `100 YES + 100 NO → 100 sBTC`
- **Transfer**: P2P token transfers
- **Redeem**: After resolution, `100 YES (if YES won) → 100 sBTC`

### 2. CLOB Matching Engine
- **Price-Time Priority**: Best price first, then FIFO
- **Matching Frequency**: Every 100ms
- **Atomic Swaps**: Matched orders execute atomically on-chain
- **Fee**: 0.5% trading fee

### 3. Optimistic Oracle
- **Propose**: Anyone can propose an answer (requires bond: 100 tokens)
- **Dispute**: 24-hour challenge window
- **Vote**: If disputed, token holders vote (48-hour period)
- **Resolve**: Final answer reported to CTF for redemption

### 4. Oracle Adapter
- **Market Creation**: Initializes both CTF condition and oracle question
- **Resolution**: Fetches oracle result and reports to CTF

## 🔐 Security Features

- ✅ Nonce-based replay protection
- ✅ Order expiration by block height
- ✅ Emergency pause functionality
- ✅ Maker-only order cancellation
- ✅ Bond-based oracle security
- ✅ Approval system for token transfers

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
(contract-call? .conditional-tokens split-position condition-id u10000)
```

**Then, place orders on CLOB:**
```bash
# Sell YES at 55 cents
curl -X POST http://localhost:3000/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "maker": "ST2CY5V39NHDPWSXMW9QDT3HC3GD6Q6XX4CFRK9AG",
    "marketId": "market_abc123",
    "positionId": "market_abc123_yes",
    "side": "SELL",
    "price": 55,
    "size": 10000
  }'

# Sell NO at 45 cents (YES + NO = 100)
curl -X POST http://localhost:3000/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "maker": "ST2CY5V39NHDPWSXMW9QDT3HC3GD6Q6XX4CFRK9AG",
    "marketId": "market_abc123",
    "positionId": "market_abc123_no",
    "side": "SELL",
    "price": 45,
    "size": 10000
  }'
```

### 3. Trader Buys YES
```bash
curl -X POST http://localhost:3000/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "maker": "ST2JHG361ZXG51QTKY2NQCVBPPRRE2KZB1HR05NNC",
    "marketId": "market_abc123",
    "positionId": "market_abc123_yes",
    "side": "BUY",
    "price": 55,
    "size": 1000
  }'
```

### 4. Matching Engine Automatically Matches
- Within 100ms, orders match
- Trade executed: 1000 YES @ 55 cents
- Fee collected: 0.5% = 2.75 sBTC

### 5. Market Resolves: ETH hits $10k (YES wins!)

**Oracle resolution:**
```clarity
(contract-call? .optimistic-oracle propose-answer question-id u1) ;; 1 = YES
;; Wait 24 hours...
(contract-call? .oracle-adapter finalize-market condition-id)
```

**Winners redeem:**
```clarity
;; Trader redeems 1000 YES tokens
(contract-call? .conditional-tokens redeem-position condition-id u1)
;; Receives: 1000 sBTC (can unwrap to real BTC!)
```

**Market maker:**
```clarity
;; Still holds 9000 YES tokens (sold 1000)
;; Redeems for: 9000 sBTC
;; Lost: The 1000 sBTC from sold YES tokens
```

## 🧪 Testing

### Contract Tests
```bash
cd stackcast-contracts
clarinet test
```

Tests verify:
- ✅ Real sBTC transfers (deposit, withdrawal, redemption)
- ✅ Split/merge logic
- ✅ Oracle resolution
- ✅ Exchange settlement

### API Tests
```bash
cd backend
bun scripts/test-api.ts
```

## 🚢 Deployment

### Testnet Deployment

1. **Update Clarinet.toml** with your deployer address
2. **Deploy contracts**:
```bash
cd stackcast-contracts
clarinet deploy --testnet
```

3. **Update backend .env** with deployed contract addresses
4. **Start backend**:
```bash
cd backend
bun start
```

### Mainnet Deployment
Same as testnet, but use `--mainnet` flag and update network config.

## 🛠️ Tech Stack

- **Blockchain**: Stacks (Bitcoin L2)
- **Smart Contracts**: Clarity
- **Collateral Token**: sBTC (1:1 Bitcoin-backed)
- **Backend**: Bun + TypeScript + Express
- **Testing**: Clarinet + Vitest
- **Token Standard**: SIP-010 (fungible tokens)

## 📝 Roadmap

### Smart Contracts ✅
- ✅ Conditional Tokens (sBTC collateral)
- ✅ CTF Exchange (atomic settlement)
- ✅ Optimistic Oracle (dispute resolution)
- ✅ Oracle Adapter (market lifecycle)

### Backend ✅
- ✅ CLOB matching engine
- ✅ Order management
- ✅ REST API

### To-Do
- [ ] Frontend (React + Stacks.js)
- [ ] WebSocket for real-time orderbook updates
- [ ] Signature verification (Stacks signatures)
- [ ] Database persistence (SQLite/Postgres)
- [ ] Market maker bot examples
- [ ] Advanced order types (FOK, IOC, Post-Only)
- [ ] Historical data API
- [ ] Chart integration
- [ ] Mobile app

## 🎯 Production Readiness

| Component | Status | Notes |
|-----------|--------|-------|
| **Smart Contracts** | ✅ Production | Uses real sBTC collateral |
| **CLOB API** | ⚠️ Demo | In-memory storage (needs Redis/Postgres for prod) |
| **Tests** | ✅ Production | Verifies real sBTC movements |
| **Deployment** | ✅ Ready | Can deploy to testnet/mainnet |

### For Production Launch:
- ✅ Smart contracts ready
- ❌ Backend needs Redis/Postgres
- ❌ Need monitoring/alerting
- ❌ Need market maker bots
- ❌ Need frontend integration

## 📄 License

MIT

## 🤝 Contributing

This is a hackathon project. Feel free to fork and improve!

## 🔗 Links

- [Stacks Docs](https://docs.stacks.co)
- [Clarity Language](https://book.clarity-lang.org)
- [Clarinet](https://github.com/hirosystems/clarinet)
- [sBTC Documentation](https://docs.hiro.so/en/tools/clarinet/sbtc-integration)
- [Bun](https://bun.sh)
