# Stellar Options — MVP Design Document

> **Version:** 1.0
> **Date:** November 8, 2026
> **Scope:** Full MVP without LP module
> **Status:** Approved for implementation

---

## 1. Overview

Stellar Options is a fully collateralized options trading and strategy infrastructure platform built on Stellar/Soroban. This design document covers the MVP: options trading (Covered Call, Cash-Secured Put, Long Put), Automated Wheel strategy with TradingView-driven Prediction Engine, RFQ-based liquidity from approved Market Makers and an internal Vault, and developer integration tools (Open API, SDK, UI Kit).

**Excluded from this MVP:** LP module (external liquidity providers, NAV, Performance Fee, withdrawal queue).

### 1.1 Tech Stack Summary

| Layer | Technology |
|---|---|
| Smart contracts | Rust / Soroban (monolith contract) |
| Backend services | TypeScript / Node.js (Fastify) |
| Database | PostgreSQL + Redis |
| Frontend | Next.js + React |
| SDK | TypeScript (npm package) |
| Infrastructure | Docker Compose + VPS (Nginx reverse proxy) |
| Blockchain | Stellar (Soroban) |
| Supported assets | sWBTC, sWETH (LayerZero/OFT bridge), USDC |
| Oracle | Chainlink (primary) + independent backup |

### 1.2 Repository Structure

Separate repositories:

| Repo | Contents |
|---|---|
| `safira-contracts` | Soroban smart contracts (Rust) |
| `safira-api` | All backend microservices (TS monorepo with workspaces) |
| `safira-web` | Next.js frontend |
| `safira-sdk` | TypeScript SDK (`@safira/sdk`) |
| `safira-ui-kit` | React component library (`@safira/ui-kit`) |

---

## 2. Service Architecture

Microservices architecture. All backend services live in the `safira-api` repo as a pnpm/turborepo workspace. Each service is a separate Docker container.

### 2.1 Service Decomposition

| Service | Responsibility | Port | Communication |
|---|---|---|---|
| **API Gateway** | Public REST API for users, partners, SDK. Auth, rate limiting, request routing | 3000 | REST in, Redis pub/sub internal |
| **RFQ Service** | Quote requests to MMs and Vault, validation, best execution selection | 3001 | Redis pub/sub |
| **MM Gateway** | WebSocket server for market makers. Auth, quote ingestion, RFQ distribution | 3002 | WebSocket in (from MMs), Redis pub/sub |
| **Strategy Service** | Automated Wheel execution, strategy state management, user configs | 3003 | Redis pub/sub |
| **Prediction Service** | TradingView webhook receiver, signal interpretation, advisory to Strategy | 3004 | HTTP webhooks in, Redis pub/sub |
| **Risk Service** | Position limits, exposure checks, Vault capital limits, hard/dynamic constraints | 3005 | Redis pub/sub, internal HTTP |
| **Settlement Worker** | Monitors Soroban for expiry, triggers settlement, oracle price checks | — | Stellar SDK, polling |
| **Oracle Service** | Chainlink price feeds, backup oracle, staleness checks, expiry price determination | 3006 | External APIs, Redis cache |
| **Notification Service** | Position status updates, strategy events, webhooks to partners | 3007/3008 | Redis pub/sub, WebSocket out |

### 2.2 Shared Infrastructure

- **PostgreSQL** — single database, schema-per-service isolation
- **Redis** — pub/sub messaging between services + caching (quotes, prices, sessions)
- **Soroban** — single `options_core` contract on Stellar testnet (dev/staging) and mainnet (production)

---

## 3. Soroban Contract: `options_core`

Single monolith contract handling all on-chain logic. Written in Rust using the Soroban SDK.

### 3.1 Data Structures

```rust
pub enum OptionType {
    Call,
    Put,
}

pub enum OptionStatus {
    Open,
    Expired,
    Exercised,
    Settled,
    Released,
}

pub struct Option {
    pub id: u64,
    pub option_type: OptionType,
    pub seller: Address,
    pub buyer: Address,
    pub underlying_token: Address,    // sWBTC or sWETH contract address
    pub collateral_token: Address,    // underlying for Call, USDC for Put
    pub collateral_amount: i128,
    pub strike_price: i128,           // scaled integer (6 decimals)
    pub premium: i128,
    pub expiry: u64,                  // ledger timestamp
    pub settlement_deadline: u64,     // expiry + settlement window
    pub status: OptionStatus,
    pub oracle_expiry_price: i128,    // set at expiry by oracle
    pub trading_fee: i128,
    pub opening_base_fee: i128,
}
```

### 3.2 Entry Points

| Function | Description |
|---|---|
| `open_option(seller, buyer, option_type, underlying_token, collateral_amount, strike_price, premium, expiry)` | Atomic: lock seller collateral + transfer premium to seller + collect fees + create position |
| `set_expiry_price(option_id, price)` | Oracle service sets verified expiry price. Admin-authorized. |
| `exercise(option_id)` | Buyer delivers counter-asset within settlement window. Atomic exchange at strike. |
| `release_collateral(option_id)` | After settlement deadline if buyer didn't exercise. Returns collateral to seller. |
| `get_option(option_id)` | Read position data |
| `configure_fees(trading_fee_bps, base_fee)` | Admin: update fee parameters |
| `add_supported_token(token_address)` | Admin: whitelist token for trading |

### 3.3 Fee Enforcement

- **Trading Fee:** 1% of premium, collected at `open_option`
- **Opening Base Fee:** Fixed USDC amount (configurable), collected at `open_option`
- Both fees are enforced in the same atomic Soroban transaction that opens the position
- Direct contract calls cannot bypass fees — logic is in the contract itself

### 3.4 Collateral Rules

| Option Type | Collateral | Posted By |
|---|---|---|
| Covered Call (Sell Higher) | Exact amount of underlying (sWBTC/sWETH) | Seller |
| Cash-Secured Put (Buy Lower) | `strike_price * amount` in USDC | Seller |
| Long Put (Protect) | Premium only (no collateral lock) | Buyer pays premium |

### 3.5 Settlement Flow

1. At expiry, Oracle Service calls `set_expiry_price` with verified price
2. If ITM: buyer has `settlement_window` (configurable, e.g. 24h) to call `exercise` with counter-asset
3. `exercise` atomically swaps seller's collateral for buyer's counter-asset at strike price
4. If buyer doesn't exercise within window: anyone can call `release_collateral`
5. Seller keeps premium in all cases (already transferred at opening)
6. If both oracles unavailable at expiry: settlement postponed under predefined recovery procedure

---

## 4. RFQ Flow & Market Maker Integration

### 4.1 RFQ Lifecycle

```
User Request → API Gateway → RFQ Service → MM Gateway (fan-out to MMs via WS)
                                         → Vault Pricing (internal quote)

MMs respond quotes → MM Gateway → RFQ Service
                                    ↓
                              Risk Service validates
                                    ↓
                              Best Execution: pick best net price after fees
                                    ↓
                              Return selected quote to user
                                    ↓
                              User confirms → Soroban tx built & signed → open_option()
```

### 4.2 MM Gateway (WebSocket Server)

- MMs connect via authenticated WebSocket (API key + HMAC signature)
- **Inbound messages:** `quote_response`, `heartbeat`, `status_update`
- **Outbound messages:** `rfq_request`, `trade_confirmation`, `settlement_notification`
- Quote timeout: configurable (default 10s) — unresponsive MMs excluded from RFQ round
- Quote fields: `mm_id`, `option_type`, `underlying`, `strike`, `expiry`, `premium`, `size`, `valid_until`, `signature`

### 4.3 Best Execution Logic

1. Collect all quotes within timeout window
2. Filter: valid signature, not expired, correct instrument, within risk limits
3. For each quote: `net_price = premium - trading_fee - opening_base_fee`
4. Select quote with best `net_price` for the user
5. No preferential treatment for Vault quotes vs MM quotes
6. If no eligible quote: no trade opened

### 4.4 Vault Internal Pricing

- Vault acts as an additional quote source (when available capital allows)
- Black-76 baseline + inventory adjustment + risk-based spread
- Can decline to quote if capital insufficient or risk limits reached
- Quotes go through same Best Execution as external MM quotes

### 4.5 Quote Authentication

- Each quote signed by MM with their private key (HMAC)
- Time-limited via `valid_until` timestamp
- One-time use: quote ID invalidated after trade or expiry
- Stored in Redis with TTL matching `valid_until`

---

## 5. Prediction Engine & Strategy Service

### 5.1 Prediction Service

Receives TradingView alert webhooks and interprets signals for the Strategy Service. No ML models — signal-based decision making.

**Endpoint:** `POST /webhooks/tradingview`

**Signal processing:**
1. Receive TradingView alert payload
2. Parse and validate: action, asset, expiry duration, optional strike hint
3. Store signal in PostgreSQL
4. Publish to Redis channel `prediction:signals`

**Signal Schema:**

```typescript
interface PredictionSignal {
  id: string;
  asset: "BTC" | "ETH";
  action: "buy_put" | "buy_call";     // only buy signals
  expiry_duration: string;             // "7d", "14d", "30d"
  strike_hint: number | null;          // optional strike suggestion
  raw_payload: Record<string, unknown>;
  received_at: Date;
}
```

There are no sell signals. Positions run to expiry automatically.

### 5.2 Strategy Service (Automated Wheel)

Executes Automated Wheel strategy based on Prediction Engine signals.

**Wheel flow:**

```
TradingView signal (buy_put) → Validate → RFQ → Open Put → Wait for expiry
  → If exercised (ITM): receive BTC/ETH token
    → Wait for TradingView signal (buy_call) → Open Call with received asset → Wait for expiry
      → If exercised: receive USDC → cycle continues
  → If not exercised (OTM): keep premium, collateral released → wait for next signal
```

**On signal received:**
1. Validate signal against user's strategy config (allowed assets, expiries, capital)
2. Check Risk Service for position limits
3. Determine strike (from signal hint or auto-select based on config)
4. Request RFQ → get best quote
5. If quote meets minimum premium threshold → build Soroban tx → open position
6. If not → skip, log reason

**Strategy Configuration (per user):**

```typescript
interface StrategyConfig {
  user_address: string;
  strategy_type: "wheel";
  asset: "sWBTC" | "sWETH";
  allocated_capital: bigint;          // USDC amount
  max_position_size: bigint;
  min_net_premium_bps: number;        // minimum acceptable premium in basis points
  allowed_expiries: string[];         // e.g. ["7d", "14d", "30d"]
  active: boolean;
}
```

**State machine:** `IDLE → PUT_OPEN → PUT_SETTLED → CALL_OPEN → CALL_SETTLED → IDLE`

Transitions are driven by Prediction Engine signals + settlement events. If no signal arrives, the strategy stays idle.

---

## 6. Oracle & Settlement Worker

### 6.1 Oracle Service

- Polls Chainlink BTC/USD and ETH/USD price feeds at regular intervals
- Independent backup oracle (configurable provider)
- **Freshness check:** reject prices older than configurable threshold (e.g. 5 min)
- **Deviation check:** if primary and backup differ beyond threshold → flag, don't use
- Publishes prices to Redis `oracle:prices` channel + caches latest
- At expiry: determines definitive expiry price with strict validation rules
- Calls Soroban `set_expiry_price()` with verified price
- If both oracles unavailable → settlement postponed, no operator price selection allowed

### 6.2 Settlement Worker

Separate process (no HTTP port), runs on polling schedule.

**Flow:**
1. Scan options with `expiry <= now` and `status = Open`
2. Trigger Oracle Service to set expiry price if not yet set
3. Monitor settlement window for buyer `exercise` calls
4. After `settlement_deadline`: call `release_collateral` for unexercised options
5. Update off-chain DB position status to match on-chain state
6. Emit events to Notification Service
7. Periodic reconciliation: compare on-chain state with off-chain DB, fix discrepancies

---

## 7. sWBTC / sWETH Bridge (LayerZero/OFT)

### 7.1 Token Flow

- **Lock → Mint:** User locks WBTC/WETH on origin chain → LayerZero OFT message → mint sWBTC/sWETH on Stellar
- **Burn → Unlock:** Burn sWBTC/sWETH on Stellar → LayerZero OFT message → unlock WBTC/WETH on origin chain

### 7.2 Security

- No admin mint capability — only bridge-verified messages can mint
- Replay protection via nonces in LayerZero messages
- DVN (Decentralized Verifier Network) configuration for message validation
- Peer/endpoint authentication between chains

### 7.3 Token Contracts

- Separate Soroban token contracts for sWBTC and sWETH
- Standard Stellar token interface (SEP-41 compatible)
- Bridge monitoring: track pending transfers, confirm completions

---

## 8. API Gateway & Developer Tools

### 8.1 API Gateway

- **Framework:** Fastify (TypeScript)
- **Auth:** Wallet-based (Stellar keypair signature) + API keys for partners
- **Rate limiting:** Per API key / wallet address
- Routes requests to internal services via Redis pub/sub or internal HTTP

### 8.2 Open API Endpoints

```
GET  /assets                         — Supported tokens and their details
GET  /options/available              — Available strikes, expiries for a given asset
POST /rfq/request                    — Request quotes for an option
GET  /rfq/:id                        — Get quote result for a pending RFQ
POST /options/open                   — Confirm and open option (returns unsigned Soroban tx)
GET  /positions                      — User's positions list
GET  /positions/:id                  — Position detail + settlement status
POST /strategies/wheel/configure     — Set up Automated Wheel strategy
GET  /strategies/wheel/status        — Current strategy state and history
POST /strategies/wheel/stop          — Stop strategy (prevents new openings)
GET  /fees                           — Current fee schedule
WS   /events                         — Real-time position updates, strategy events
```

### 8.3 SDK (`@safira/sdk`)

Separate repo `safira-sdk`. TypeScript package published to npm.

- Typed wrappers for all Open API endpoints
- Stellar wallet integration helpers (Freighter, WalletConnect)
- Soroban transaction building and signing
- WebSocket event subscription
- Error handling and retry logic

### 8.4 UI Kit (`@safira/ui-kit`)

Separate repo `safira-ui-kit`. React component library published to npm.

- Asset selector, strike/expiry picker, quote display
- Position card, position list, strategy dashboard
- Themeable via CSS variables / design tokens
- Depends on `@safira/sdk`
- Embeddable in partner applications

---

## 9. Web App

Separate repo `safira-web`. Next.js + React application.

### 9.1 Pages

| Page | Description |
|---|---|
| Landing | Product overview, value proposition |
| Trade | Sell Higher (Covered Call), Buy Lower (Cash-Secured Put), Protect (Long Put) |
| Positions | Active, expired, settled, released positions with details |
| Wheel Strategy | Configure and monitor Automated Wheel |
| Settings | Wallet connection, notifications, API key management |

### 9.2 Integration

- Uses `@safira/sdk` for all API communication
- Wallet integration: Freighter browser extension, WalletConnect
- Real-time updates via WebSocket `/events` endpoint
- Server-side rendering for landing/marketing pages, client-side for trading UI

---

## 10. Database Schema

Single PostgreSQL instance, schema-per-service isolation.

### 10.1 Core Tables

```sql
-- Positions (off-chain mirror of on-chain state)
CREATE TABLE positions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    on_chain_id BIGINT UNIQUE,
    option_type TEXT NOT NULL CHECK (option_type IN ('call', 'put')),
    seller_address TEXT NOT NULL,
    buyer_address TEXT NOT NULL,
    underlying_token TEXT NOT NULL,
    collateral_amount NUMERIC NOT NULL,
    strike_price NUMERIC NOT NULL,
    premium NUMERIC NOT NULL,
    expiry TIMESTAMPTZ NOT NULL,
    settlement_deadline TIMESTAMPTZ NOT NULL,
    status TEXT NOT NULL DEFAULT 'open'
        CHECK (status IN ('open', 'expired', 'exercised', 'settled', 'released')),
    oracle_expiry_price NUMERIC,
    trading_fee NUMERIC NOT NULL,
    opening_base_fee NUMERIC NOT NULL,
    tx_hash TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- RFQ quotes
CREATE TABLE quotes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rfq_id UUID NOT NULL,
    mm_id UUID REFERENCES market_makers(id),
    option_type TEXT NOT NULL,
    underlying TEXT NOT NULL,
    strike NUMERIC NOT NULL,
    expiry TIMESTAMPTZ NOT NULL,
    premium NUMERIC NOT NULL,
    valid_until TIMESTAMPTZ NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'selected', 'rejected', 'expired')),
    signature TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Market makers
CREATE TABLE market_makers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    api_key_hash TEXT NOT NULL UNIQUE,
    status TEXT NOT NULL DEFAULT 'active'
        CHECK (status IN ('active', 'suspended')),
    config JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Strategy configs
CREATE TABLE strategy_configs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_address TEXT NOT NULL,
    strategy_type TEXT NOT NULL DEFAULT 'wheel',
    asset TEXT NOT NULL,
    allocated_capital NUMERIC NOT NULL,
    max_position_size NUMERIC NOT NULL,
    min_net_premium_bps INT NOT NULL,
    allowed_expiries TEXT[] NOT NULL,
    active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Prediction signals
CREATE TABLE prediction_signals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset TEXT NOT NULL,
    action TEXT NOT NULL CHECK (action IN ('buy_put', 'buy_call')),
    expiry_duration TEXT NOT NULL,
    strike_hint NUMERIC,
    raw_payload JSONB NOT NULL,
    received_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    processed BOOLEAN NOT NULL DEFAULT false
);

-- Vault state (internal capital only, no LP module)
CREATE TABLE vault_state (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset TEXT NOT NULL UNIQUE,
    available_capital NUMERIC NOT NULL DEFAULT 0,
    locked_collateral NUMERIC NOT NULL DEFAULT 0,
    total_inventory NUMERIC NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 11. Infrastructure

### 11.1 Docker Compose Services

```yaml
services:
  api-gateway:         # Fastify, port 3000
  rfq-service:         # port 3001
  mm-gateway:          # WebSocket, port 3002
  strategy-service:    # port 3003
  prediction-service:  # port 3004
  risk-service:        # port 3005
  settlement-worker:   # no port, polling
  oracle-service:      # port 3006
  notification-service: # port 3007, WS 3008
  postgres:            # port 5432
  redis:               # port 6379
  web:                 # Next.js, port 8080
  nginx:               # reverse proxy, ports 80/443
```

### 11.2 Environments

| Environment | Blockchain | Purpose |
|---|---|---|
| Local | Stellar standalone / Soroban sandbox | Development |
| Staging | Stellar testnet | Integration testing |
| Production | Stellar mainnet | Live trading |

### 11.3 Deployment

- VPS provider: Hetzner or DigitalOcean
- Docker Compose for orchestration
- Nginx reverse proxy with SSL (Let's Encrypt)
- Environment configs via `.env` files per service

---

## 12. Security

### 12.1 On-chain

- Full seller collateralization enforced by contract
- Atomic fee collection in `open_option` — cannot be bypassed
- Admin functions (fee config, token whitelist) require authorized address
- No admin mint for sWBTC/sWETH — only bridge-verified
- Oracle expiry price immutable once set

### 12.2 Off-chain

- MM quote signatures (HMAC) prevent forgery
- Quote TTL + one-time use prevent replay attacks
- Rate limiting on all public API endpoints
- Input validation on all services
- API key + wallet signature authentication
- TradingView webhook validation (secret token)
- Inter-service communication over internal Docker network

### 12.3 Operational

- Settlement Worker monitors for stalled positions
- Oracle staleness detection and fallback procedures
- Bridge monitoring for pending/failed transfers
- Reconciliation between on-chain and off-chain state

---

## 13. Out of Scope

The following are explicitly excluded from this MVP design:

- **LP module** — external liquidity providers, NAV accounting, Performance Fee (15% HWM), withdrawal queue, share-based participation
- **Early close / buyback** of open options
- **Third-party wrapped tokens** — only protocol-issued sWBTC/sWETH
- **Permissionless market making** — MMs are approved and permissioned
- **Cross-token substitution** — no automatic swaps between different BTC/ETH wrappers
- **Mobile application**
- **Fiat on/off ramp**

These may be addressed in subsequent design documents.
