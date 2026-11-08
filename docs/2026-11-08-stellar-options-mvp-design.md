# Stellar Options — MVP Design Document

> **Version:** 2.0
> **Date:** November 8, 2026
> **Scope:** Full MVP without LP module
> **Status:** Approved for implementation
> **Reference:** Rysk Finance V12 (RFQ model, Opyn/Gamma settlement patterns)

---

## 1. Overview

Stellar Options is a fully collateralized options trading and strategy infrastructure platform built on Stellar/Soroban. This design document covers the MVP: options trading (Covered Call, Cash-Secured Put, Long Put), Automated Wheel strategy with TradingView-driven Prediction Engine, RFQ-based liquidity from approved Market Makers and an internal Vault, and developer integration tools (Open API, SDK, UI Kit).

**Excluded from this MVP:** LP module (external liquidity providers, NAV, Performance Fee, withdrawal queue).

### 1.1 Tech Stack Summary

| Layer | Technology |
|---|---|
| Smart contracts | Rust / Soroban (monolith contract) |
| Backend services | TypeScript / Node.js (Fastify) |
| Database | PostgreSQL (Drizzle ORM + migrations) + Redis |
| Frontend | Next.js + React |
| SDK | TypeScript (npm package, lives in `safira-api` repo until API stabilizes) |
| Infrastructure | Docker Compose + VPS (Nginx reverse proxy) |
| Blockchain | Stellar (Soroban) |
| Supported assets | sWBTC, sWETH (see Section 7 on bridge), USDC |
| Oracle | Chainlink (primary) + independent backup |

### 1.2 Repository Structure

| Repo | Contents |
|---|---|
| `safira-contracts` | Soroban smart contracts (Rust): `options_core`, sWBTC/sWETH token contracts |
| `safira-api` | Backend monolith: API Server + Settlement Worker + SDK package (`packages/sdk`) |
| `safira-web` | Next.js frontend |
| `safira-ui-kit` | React component library (`@safira/ui-kit`) — created after API stabilizes |

SDK lives inside `safira-api` as a workspace package during MVP. Types and interfaces are shared directly. Extracted to a separate repo when the API is stable.

---

## 2. Service Architecture

Hybrid architecture: **modular monolith API Server** + **separate Settlement Worker**. All business logic modules run in a single Node.js process with direct function calls between them. Settlement and oracle monitoring run as a separate process to avoid coupling with the API lifecycle.

### 2.1 Two Processes

| Process | Port | Responsibility |
|---|---|---|
| **API Server** | 3000 (HTTP + WS upgrade for MMs and users) | All request handling, RFQ, trading, strategy, prediction webhooks, notifications |
| **Settlement Worker** | 3100 (health check only) | Monitors Soroban for expiry events, oracle price feeds, automatic settlement, reconciliation |

Both processes share the same codebase and `shared/` modules.

### 2.2 API Server Internal Modules

All modules run in a single Fastify process and communicate via direct function calls (no network overhead, no serialization).

```
src/
├── modules/
│   ├── rfq/              # RFQ lifecycle, best execution, quote validation
│   ├── mm-gateway/       # WebSocket server for market makers, quote ingestion
│   ├── positions/        # Position CRUD, status tracking, on-chain mirroring
│   ├── strategy/         # Automated Wheel state machine, strategy configs
│   ├── prediction/       # TradingView webhook receiver, signal parsing
│   ├── risk/             # Position limits, exposure checks, capital limits
│   ├── vault/            # Internal Vault pricing (Black-76), capital tracking
│   ├── notifications/    # WebSocket push to users, partner webhook delivery
│   └── auth/             # Wallet signature verification, API key management
├── routes/               # Fastify route definitions (thin layer, delegates to modules)
├── shared/               # DB client (Drizzle), Redis client, Stellar SDK helpers, types
└── config/               # Environment config, feature flags
```

Each module exposes a typed interface (e.g. `RfqModule.requestQuotes(...)`) and receives dependencies via constructor injection. Modules do not import each other directly — they depend on interfaces, making future extraction to separate services straightforward.

### 2.3 Settlement Worker Internal Structure

```
src/
├── workers/
│   ├── oracle/           # Chainlink price polling, backup oracle, freshness checks
│   ├── settlement/       # Expiry scanning, automatic settlement, collateral release
│   └── reconciliation/   # On-chain vs off-chain state comparison, drift detection
├── shared/               # Shared with API Server: DB client, Stellar SDK, types
└── config/
```

Runs on a polling loop (configurable interval, default 15s). Exposes health check endpoint on port 3100 (`GET /health`). Communicates with API Server through the shared PostgreSQL database and Redis pub/sub (for event notifications).

**Resilience:** Docker restart policy `unless-stopped`. Health check every 30s. If the worker misses 3 consecutive cycles, Docker restarts it. All settlement operations are **idempotent** — safe to retry after crash/restart. Worker logs last successful cycle timestamp to Redis; API Server can alert if stale.

### 2.4 Shared Infrastructure

- **PostgreSQL** — single database, shared by both processes. Migrations via Drizzle Kit.
- **Redis** — caching (quotes, oracle prices) + pub/sub (Settlement Worker → API Server event notifications)
- **Soroban** — `options_core` contract on Stellar testnet (dev/staging) and mainnet (production)

---

## 3. Soroban Contract: `options_core`

Single monolith contract handling all on-chain logic. Written in Rust using the Soroban SDK.

Inspired by Rysk/Opyn Gamma patterns: fully collateralized positions, automatic settlement at expiry based on oracle price, no buyer-initiated exercise.

### 3.1 Data Structures

```rust
pub enum OptionType {
    Call,
    Put,
}

pub enum OptionStatus {
    Open,           // Position created, collateral locked
    Settled,        // Expiry price set, collateral distributed (ITM)
    Expired,        // Expiry price set, OTM — collateral returned to seller
    Defaulted,      // Settlement failed (edge case) — collateral returned to seller
}

pub struct OptionPosition {
    pub id: u64,
    pub option_type: OptionType,
    pub seller: Address,
    pub buyer: Address,
    pub underlying_token: Address,    // sWBTC or sWETH contract address
    pub settlement_token: Address,    // USDC contract address
    pub quantity: i128,               // amount of underlying (e.g. 1.0 BTC = 10_000_000 at 7 decimals)
    pub strike_price: i128,           // USDC per unit of underlying (6 decimals)
    pub premium: i128,                // USDC amount paid by buyer to seller
    pub collateral_amount: i128,      // locked: quantity of underlying (Call) or strike*quantity USDC (Put)
    pub collateral_token: Address,    // underlying_token for Call, settlement_token for Put
    pub expiry: u64,                  // ledger timestamp
    pub status: OptionStatus,
    pub oracle_expiry_price: i128,    // set at settlement, 0 until then
    pub trading_fee: i128,            // USDC
    pub opening_base_fee: i128,       // USDC
    pub created_at: u64,              // ledger timestamp
}
```

**Key difference from v1.1:** Added `quantity` field, removed `settlement_deadline` (settlement is automatic, not buyer-initiated), simplified status enum, renamed to `OptionPosition` to avoid Rust keyword conflict.

### 3.2 Entry Points

| Function | Auth | Description |
|---|---|---|
| `open_option(params: OpenOptionParams)` | Both seller + buyer sign | Atomic: lock seller collateral + transfer premium from buyer to seller + collect fees + create position. Requires multi-auth (see 3.6). |
| `settle(option_id, expiry_price)` | Oracle admin key | Called by Settlement Worker after expiry. Sets oracle price, computes ITM/OTM, distributes collateral automatically. Single atomic operation. |
| `get_option(option_id)` | Public | Read position data |
| `get_options_by_address(address)` | Public | List positions for a given seller or buyer |
| `configure_fees(trading_fee_bps, base_fee)` | Contract admin | Update fee parameters (prospective only) |
| `add_supported_token(token_address)` | Contract admin | Whitelist token for trading |
| `pause() / unpause()` | Contract admin | Emergency: prevent new openings without affecting existing positions |

**Removed from v1.1:** `exercise()`, `release_collateral()` — settlement is now fully automatic (like Rysk).

### 3.3 Fee Enforcement

- **Trading Fee:** 1% of premium, collected at `open_option`
- **Opening Base Fee:** Fixed USDC amount (configurable), collected at `open_option`
- Both fees are enforced in the same atomic Soroban transaction that opens the position
- Direct contract calls cannot bypass fees — logic is in the contract itself
- **Fee formula cap** (inspired by Rysk): `trading_fee = MIN(0.01 * underlying_price * quantity, 0.01 * premium)`. This prevents disproportionate fees on deep OTM options with tiny premiums.

### 3.4 Collateral Rules

| Option Type | Seller Collateral | Buyer Pays |
|---|---|---|
| Covered Call (Sell Higher) | `quantity` of underlying token (sWBTC/sWETH) | Premium (USDC) |
| Cash-Secured Put (Buy Lower) | `strike_price * quantity` in USDC | Premium (USDC) |
| Long Put (Protect) | MM/Vault is the seller — they post `strike_price * quantity` USDC | Premium (USDC) |

All collateral locked in the contract until settlement. No rehypothecation. No liquidations.

### 3.5 Automatic Settlement Flow (Rysk-inspired)

Settlement is fully automatic. No buyer action required at expiry. No settlement window.

1. Option reaches `expiry` timestamp
2. Settlement Worker polls for expired options with `status = Open`
3. Oracle module fetches verified expiry price from Chainlink (+ backup)
4. Settlement Worker calls `settle(option_id, expiry_price)` on Soroban
5. Contract logic inside `settle()`:

**For Covered Call (seller locked underlying):**
- If `expiry_price >= strike_price` (ITM): contract transfers `strike_price * quantity` USDC from buyer's pre-approved balance to seller, transfers `quantity` underlying from contract to buyer. Status → `Settled`.
- If `expiry_price < strike_price` (OTM): contract returns `quantity` underlying to seller. Status → `Expired`.

**For Cash-Secured Put (seller locked USDC):**
- If `expiry_price <= strike_price` (ITM): contract transfers `quantity` underlying from buyer's pre-approved balance to seller, transfers `strike_price * quantity` USDC from contract to buyer. Status → `Settled`.
- If `expiry_price > strike_price` (OTM): contract returns USDC to seller. Status → `Expired`.

**Important: physical settlement requires buyer to have pre-approved sufficient balance.** If buyer has insufficient balance at settlement time, the option defaults: seller collateral is returned, seller keeps premium. Status → `Defaulted`. This matches the spec requirement that the protocol does not guarantee settlement.

6. Seller keeps premium in all cases (transferred at opening)
7. If both oracles unavailable: settlement postponed. Worker retries on next cycle. Max postponement configurable (e.g. 24h), after which option defaults and collateral returns to seller.

### 3.6 Transaction Authorization (Multi-Auth)

Opening an option requires both parties to authorize:

**Flow:**
1. API Server builds the `open_option` Soroban transaction with all parameters
2. Returns unsigned transaction XDR to the user (buyer for Protect, seller for Sell Higher / Buy Lower)
3. User signs with their Stellar wallet (Freighter/WalletConnect)
4. API Server adds the counterparty signature:
   - For user-initiated trades: MM/Vault signs server-side (MM's deposited key or Vault's operational key)
   - For strategy-initiated trades: Strategy Vault operational key signs both sides
5. Fully signed transaction submitted to Stellar network

**Soroban multi-auth:** Both `seller` and `buyer` addresses must call `require_auth()` in the contract. The transaction includes auth entries for both parties.

### 3.7 On-chain Oracle Price Bounds

The `settle()` function includes a sanity check: the provided `expiry_price` must be within a configurable deviation from the last known on-chain price reference (e.g. ±50% from the price at option opening). This prevents a compromised oracle key from setting an absurd price. The bound is wide enough to accommodate legitimate market moves but catches obvious manipulation.

```rust
// Inside settle()
let max_deviation_bps: i128 = 5000; // 50%
let opening_ref_price = option.strike_price; // approximate reference
let deviation = abs(expiry_price - opening_ref_price) * 10000 / opening_ref_price;
if deviation > max_deviation_bps {
    panic!("expiry price exceeds safety bounds");
}
```

---

## 4. RFQ Flow & Market Maker Integration

### 4.1 RFQ Lifecycle

All steps below happen within the API Server process via direct module calls. Inspired by Rysk's 2-second aggregation window.

```
User Request → routes/rfq → RfqModule.requestQuotes(rfq_params)
                               ├→ MmGatewayModule.fanOutRfq()  (WS to MMs)
                               └→ VaultModule.generateQuote()   (internal quote)

   ── 2-second aggregation window ──

MM quotes arrive via WS → MmGatewayModule → RfqModule.collectQuotes()
                                               ↓
                                         RiskModule.validateQuotes()
                                               ↓
                                         RfqModule.bestExecution()  (best net price after fees)
                                               ↓
                                         Return best quote to user (real-time via WS)
                                               ↓
                                         User confirms → PositionsModule.buildTx() → open_option()
```

### 4.2 RFQ Request Parameters

```typescript
interface RfqRequest {
  underlying_token: string;       // sWBTC or sWETH address
  option_type: "call" | "put";
  quantity: string;               // e.g. "1.0" (decimal string)
  strike_price: string;           // USDC per unit
  expiry: string;                 // ISO 8601 timestamp
  direction: "sell" | "buy";      // user is selling or buying the option
}
```

### 4.3 MM Gateway (WebSocket Server)

WebSocket endpoint: `ws://api-server:3000/ws/mm` (upgraded from HTTP).

**Protocol:** JSON-RPC 2.0 (like Rysk V12).

**Authentication:** On connect, MM sends `auth` message with API key + Ed25519 signature of a server-provided nonce. This provides non-repudiation (MM cannot deny they submitted a quote).

**Message types:**

| Direction | Method | Description |
|---|---|---|
| Server → MM | `rfq.new` | New RFQ request with parameters |
| MM → Server | `rfq.quote` | Quote response: premium, valid_until, signature |
| Server → MM | `rfq.result` | Trade result: `won` / `lost` / `no_fill` |
| Server → MM | `rfq.trade` | Confirmation when MM's quote is accepted with tx details |
| Server → MM | `settlement.notify` | Settlement result for MM's positions |
| MM → Server | `heartbeat` | Keep-alive |

**Quote aggregation:** 2-second window from RFQ broadcast. Best quote selected after window closes. MM can update their quote within the window (latest quote wins).

**Quote fields:**
```typescript
interface MmQuote {
  rfq_id: string;
  premium: string;            // USDC amount (decimal string)
  valid_until: number;        // Unix timestamp
  nonce: string;              // Unique, prevents replay
  signature: string;          // Ed25519 signature over (rfq_id + premium + valid_until + nonce)
}
```

### 4.4 MM Deposit Requirement (Rysk-inspired)

Before quoting, MMs must deposit USDC collateral into their on-chain account managed by `options_core`. The contract verifies sufficient deposited balance before executing a trade. This ensures MMs can fulfill settlement obligations.

| Function | Description |
|---|---|
| `mm_deposit(amount)` | MM deposits USDC into their contract-managed balance |
| `mm_withdraw(amount)` | MM withdraws free (non-locked) USDC |
| `mm_balance(address)` | Read available and locked balances |

When a trade opens with an MM as counterparty, the required collateral is deducted from their deposited balance and locked in the position. This replaces the v1.1 model where MMs would need to sign every individual transaction with their Stellar key.

### 4.5 Best Execution Logic

1. Collect all quotes within 2-second aggregation window
2. Filter: valid Ed25519 signature, not expired, correct instrument, MM has sufficient deposited balance
3. For each quote: `net_price = premium - trading_fee - opening_base_fee`
4. Select quote with best `net_price` for the user
5. No preferential treatment for Vault quotes vs MM quotes
6. If no eligible quote: no trade opened, user notified

### 4.6 Vault Internal Pricing

- Vault acts as an additional quote source (when available capital allows)
- Black-76 baseline + inventory adjustment + risk-based spread
- Can decline to quote if capital insufficient or risk limits reached
- Quotes go through same Best Execution as external MM quotes

---

## 5. Prediction Engine & Strategy Service

### 5.1 Prediction Module

Part of the API Server process. Receives TradingView alert webhooks and interprets signals for the Strategy module. No ML models — signal-based decision making.

**Endpoint:** `POST /webhooks/tradingview`

**Webhook authentication:** TradingView alerts include a configurable secret token in a custom header (`X-Safira-Secret`). The Prediction module validates this token matches the configured environment variable before processing.

**Signal processing:**
1. Receive TradingView alert payload
2. Validate `X-Safira-Secret` header
3. Parse and validate: action, asset, expiry duration, optional strike hint
4. Store signal in PostgreSQL
5. Call `StrategyModule.onSignal()` directly (same process)

**Signal Schema:**

```typescript
interface PredictionSignal {
  id: string;
  asset: "BTC" | "ETH";
  action: "sell_put" | "sell_call";   // sell = user sells option, receives premium
  expiry_duration: string;             // "7d", "14d", "30d"
  strike_hint: number | null;          // optional strike suggestion (% from current price)
  raw_payload: Record<string, unknown>;
  received_at: Date;
}
```

**Terminology clarification:** In the Wheel strategy, the user is always the option **seller** (earning premium). `sell_put` = user sells a cash-secured put (commits USDC to buy underlying at strike). `sell_call` = user sells a covered call (commits underlying to sell at strike). The signal tells the strategy WHEN to sell and with what parameters — not to buy options.

There are no close/exit signals. Positions run to automatic expiry settlement.

### 5.2 Strategy Module (Automated Wheel)

Part of the API Server process. Executes Automated Wheel strategy based on Prediction Engine signals.

**Wheel flow:**

```
Signal (sell_put) → Validate → RFQ (user=seller, MM=buyer) → Open Put → Auto-settle at expiry
  → If settled (ITM): user receives underlying token (bought at strike)
    → Signal (sell_call) → Open Call with received asset → Auto-settle at expiry
      → If settled (ITM): user receives USDC (sold at strike) → cycle continues
  → If expired (OTM): user keeps premium + collateral released → wait for next signal
```

**On signal received:**
1. Validate signal against user's strategy config (allowed assets, expiries, capital)
2. Check `RiskModule.validatePosition()` for position limits
3. Determine strike (from signal hint or auto-select based on config delta range)
4. Request RFQ → get best quote from MM/Vault as counterparty
5. If quote meets minimum premium threshold → build Soroban tx → open position
6. If not → skip, log reason, notify user

**Fan-out handling:** One TradingView signal triggers strategy evaluation for ALL active configs for that asset. To avoid overwhelming MMs with simultaneous RFQs:
- Process strategy configs sequentially with 500ms delay between RFQs
- Each RFQ is independent — different users may get different quotes/MMs
- If MM capacity is exhausted, remaining users get `no_fill` and wait for next signal

**Strategy Configuration (per user):**

```typescript
interface StrategyConfig {
  id: string;
  user_address: string;
  strategy_type: "wheel";
  asset: "sWBTC" | "sWETH";
  allocated_capital: string;          // USDC amount (decimal string)
  max_position_size: string;          // max underlying per position
  min_net_premium_bps: number;        // minimum acceptable premium in basis points
  allowed_expiries: string[];         // e.g. ["7d", "14d", "30d"]
  active: boolean;
  current_state: WheelState;          // persisted state machine position
  current_position_id: string | null; // active position if any
  underlying_balance: string;         // underlying received from settled puts
}

type WheelState = "idle" | "put_open" | "holding_underlying" | "call_open";
```

**State machine:** `IDLE → PUT_OPEN → HOLDING_UNDERLYING → CALL_OPEN → IDLE`

Transitions:
- `IDLE` → `PUT_OPEN`: on `sell_put` signal, if capital available
- `PUT_OPEN` → `HOLDING_UNDERLYING`: on settlement (ITM put, user received underlying)
- `PUT_OPEN` → `IDLE`: on expiry (OTM, USDC returned)
- `HOLDING_UNDERLYING` → `CALL_OPEN`: on `sell_call` signal
- `CALL_OPEN` → `IDLE`: on settlement (ITM call, user received USDC) or expiry (OTM, underlying returned → stays `HOLDING_UNDERLYING`)

Settlement events arrive via Redis pub/sub from Settlement Worker → Strategy module updates state.

---

## 6. Oracle & Settlement Worker

The Settlement Worker is a **separate Node.js process** (not part of the API Server). It handles all time-sensitive background work: oracle price feeds, automatic settlement, and state reconciliation.

### 6.1 Oracle Module (inside Settlement Worker)

- Polls Chainlink BTC/USD and ETH/USD price feeds every 15 seconds
- Independent backup oracle (configurable provider)
- **Freshness check:** reject prices older than configurable threshold (default 5 min)
- **Deviation check:** if primary and backup differ > 2% → flag as suspicious, use primary but log alert
- **Hard deviation:** if differ > 10% → refuse to settle, alert, wait for human review
- Caches latest prices in Redis key `oracle:price:{asset}` with TTL (shared with API Server for display/pricing)
- At expiry: determines definitive expiry price with strict validation rules
- If both oracles unavailable → settlement postponed, retries on next cycle. Max postponement 24h, then option defaults.

### 6.2 Oracle Key Security

The Settlement Worker signs `settle()` transactions with a dedicated oracle admin key.

- **Storage:** Environment variable injected via Docker secret (not in `.env` file on disk)
- **Scope:** This key can ONLY call `settle()` and `pause()`/`unpause()`. Cannot transfer funds, change fees, or modify token whitelist. Separate admin key for contract configuration.
- **On-chain bounds:** `settle()` rejects prices outside ±50% of strike price (see Section 3.7). Even a compromised key cannot set an arbitrary price.
- **Rotation:** Key can be rotated via contract admin function without redeploying.

### 6.3 Settlement Module (inside Settlement Worker)

Runs on a polling loop (configurable interval, default 15s).

**Flow:**
1. Scan `positions` table for `expiry <= now AND status = 'open'`
2. For each expired position:
   a. Fetch verified expiry price from oracle module
   b. Build and sign `settle(option_id, expiry_price)` Soroban transaction
   c. Submit to Stellar network
   d. On success: update off-chain DB position status + oracle_expiry_price
   e. On failure: log error, retry on next cycle (idempotent — contract checks status)
3. Publish settlement events to Redis pub/sub channel `settlement:events`
4. API Server's notification module picks up events → pushes to users via WebSocket

**Idempotency:** Calling `settle()` on an already-settled position is a no-op (contract checks `status == Open`). Safe to retry after crashes.

### 6.4 Reconciliation Module

Runs every 5 minutes:
1. Fetch all `Open` positions from DB
2. For each, read on-chain state via `get_option()`
3. If on-chain status differs from DB → update DB to match on-chain (on-chain is source of truth)
4. Log discrepancies for investigation

---

## 7. sWBTC / sWETH Bridge (LayerZero/OFT)

### 7.1 MVP Approach

Full LayerZero/OFT bridge is complex infrastructure. For MVP:

- **Origin chain:** Ethereum mainnet (WBTC, WETH are most liquid there)
- **Soroban side:** Deploy sWBTC and sWETH as standard Soroban token contracts (SEP-41 compatible)
- **Bridge contracts:** OFT adapter on Ethereum (Solidity), OFT endpoint on Stellar (Soroban Rust)
- **DVN config:** Use LayerZero default DVN set for initial security; evaluate custom DVN post-launch

**For development/testing:** Deploy sWBTC/sWETH as simple mintable test tokens on Stellar testnet with an admin mint function. Bridge integration tested separately.

### 7.2 Token Flow

- **Lock → Mint:** User locks WBTC/WETH in OFT adapter on Ethereum → LayerZero message → mint sWBTC/sWETH on Stellar
- **Burn → Unlock:** Burn sWBTC/sWETH on Stellar → LayerZero message → unlock WBTC/WETH on Ethereum

### 7.3 Security

- No admin mint capability in production — only bridge-verified messages can mint
- Replay protection via nonces in LayerZero messages
- Rate limit on minting: max X tokens per hour (configurable, prevents bridge exploit drain)
- Peer/endpoint authentication between chains
- Bridge monitoring in Settlement Worker: track pending transfers, alert on stuck messages

### 7.4 Fallback

If LayerZero is unavailable:
- Existing sWBTC/sWETH on Stellar continue to work for trading
- New minting/burning is paused until bridge recovers
- Positions and settlements are unaffected (they use tokens already on Stellar)

---

## 8. API Layer & Developer Tools

### 8.1 API Layer (routes in API Server)

- **Framework:** Fastify (TypeScript)
- **Auth:** Wallet-based (Stellar Ed25519 keypair signature of a nonce) + API keys for partners
- **Rate limiting:** Per API key / wallet address. Default: 10 RFQ/min per user, 100 req/min general.
- Thin route handlers that delegate to internal modules via direct function calls
- **Health/readiness:** `GET /health` (liveness), `GET /ready` (DB + Redis + Stellar connectivity)

### 8.2 Open API Endpoints

```
GET  /health                         — Liveness check
GET  /ready                          — Readiness check (dependencies)
GET  /assets                         — Supported tokens and their details
GET  /options/available              — Available strikes, expiries for a given asset
GET  /fees                           — Current fee schedule

POST /rfq/request                    — Request quotes for an option
  Body: { underlying_token, option_type, quantity, strike_price, expiry, direction }
  Returns: { rfq_id } — subscribe to WS for real-time quote updates

POST /options/open                   — Confirm best quote, returns unsigned Soroban tx XDR
  Body: { rfq_id, quote_id }
  Returns: { tx_xdr, signing_instructions }

POST /options/submit                 — Submit signed transaction
  Body: { tx_xdr_signed }
  Returns: { tx_hash, position_id }

GET  /positions                      — User's positions list (query: status, asset)
GET  /positions/:id                  — Position detail + settlement status

POST /strategies/wheel/configure     — Create/update Automated Wheel strategy config
GET  /strategies/wheel/status        — Current strategy state, position, history
POST /strategies/wheel/stop          — Stop strategy (prevents new openings)

WS   /ws/user                        — Real-time: RFQ quotes, position updates, strategy events
WS   /ws/mm                          — Market maker RFQ interface (JSON-RPC 2.0)
```

### 8.3 SDK (`@safira/sdk`)

Lives in `safira-api` repo as `packages/sdk` during MVP. TypeScript package.

- Typed wrappers for all Open API endpoints
- Stellar wallet integration helpers (Freighter, WalletConnect)
- Soroban transaction building, signing, and submission
- WebSocket event subscription with auto-reconnect
- Error handling and retry logic

### 8.4 UI Kit (`@safira/ui-kit`)

Separate repo. React component library. Created after API/SDK stabilizes.

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
| Positions | Active, expired, settled, defaulted positions with details |
| Wheel Strategy | Configure and monitor Automated Wheel |
| Settings | Wallet connection, notifications, API key management |

### 9.2 Integration

- Uses `@safira/sdk` for all API communication
- Wallet integration: Freighter browser extension, WalletConnect
- Real-time updates via WebSocket `/ws/user` endpoint
- Server-side rendering for landing/marketing pages, client-side for trading UI

---

## 10. Database Schema

Single PostgreSQL instance, shared by API Server and Settlement Worker. Migrations managed by Drizzle Kit.

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
    quantity NUMERIC NOT NULL,
    strike_price NUMERIC NOT NULL,
    premium NUMERIC NOT NULL,
    collateral_amount NUMERIC NOT NULL,
    collateral_token TEXT NOT NULL,
    expiry TIMESTAMPTZ NOT NULL,
    status TEXT NOT NULL DEFAULT 'open'
        CHECK (status IN ('open', 'settled', 'expired', 'defaulted')),
    oracle_expiry_price NUMERIC,
    trading_fee NUMERIC NOT NULL,
    opening_base_fee NUMERIC NOT NULL,
    open_tx_hash TEXT,
    settle_tx_hash TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    settled_at TIMESTAMPTZ
);

CREATE INDEX idx_positions_status_expiry ON positions (status, expiry);
CREATE INDEX idx_positions_seller ON positions (seller_address);
CREATE INDEX idx_positions_buyer ON positions (buyer_address);

-- RFQ requests
CREATE TABLE rfq_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_address TEXT NOT NULL,
    underlying_token TEXT NOT NULL,
    option_type TEXT NOT NULL,
    quantity NUMERIC NOT NULL,
    strike_price NUMERIC NOT NULL,
    expiry TIMESTAMPTZ NOT NULL,
    direction TEXT NOT NULL CHECK (direction IN ('sell', 'buy')),
    status TEXT NOT NULL DEFAULT 'open'
        CHECK (status IN ('open', 'filled', 'no_fill', 'expired')),
    best_quote_id UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    closed_at TIMESTAMPTZ
);

-- RFQ quotes
CREATE TABLE quotes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rfq_id UUID NOT NULL REFERENCES rfq_requests(id),
    mm_id UUID REFERENCES market_makers(id),
    is_vault_quote BOOLEAN NOT NULL DEFAULT false,
    premium NUMERIC NOT NULL,
    valid_until TIMESTAMPTZ NOT NULL,
    nonce TEXT NOT NULL UNIQUE,
    signature TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'won', 'lost', 'expired')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_quotes_rfq ON quotes (rfq_id);

-- Market makers
CREATE TABLE market_makers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    api_key_hash TEXT NOT NULL UNIQUE,
    ed25519_pubkey TEXT NOT NULL,         -- for quote signature verification
    status TEXT NOT NULL DEFAULT 'active'
        CHECK (status IN ('active', 'suspended')),
    stellar_address TEXT NOT NULL,         -- for on-chain deposit/settlement
    config JSONB NOT NULL DEFAULT '{}',   -- supported assets, size limits
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- MM on-chain deposit tracking (mirrors contract state)
CREATE TABLE mm_balances (
    mm_id UUID PRIMARY KEY REFERENCES market_makers(id),
    available_usdc NUMERIC NOT NULL DEFAULT 0,
    locked_usdc NUMERIC NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Strategy configs (includes persistent state)
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
    current_state TEXT NOT NULL DEFAULT 'idle'
        CHECK (current_state IN ('idle', 'put_open', 'holding_underlying', 'call_open')),
    current_position_id UUID REFERENCES positions(id),
    underlying_balance NUMERIC NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_strategy_active ON strategy_configs (active, asset) WHERE active = true;

-- Prediction signals
CREATE TABLE prediction_signals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset TEXT NOT NULL,
    action TEXT NOT NULL CHECK (action IN ('sell_put', 'sell_call')),
    expiry_duration TEXT NOT NULL,
    strike_hint NUMERIC,
    raw_payload JSONB NOT NULL,
    received_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    processed BOOLEAN NOT NULL DEFAULT false
);

-- Vault state (internal capital only, no LP module)
CREATE TABLE vault_state (
    asset TEXT PRIMARY KEY,              -- simplified: asset IS the PK
    available_usdc NUMERIC NOT NULL DEFAULT 0,
    locked_usdc NUMERIC NOT NULL DEFAULT 0,
    available_underlying NUMERIC NOT NULL DEFAULT 0,
    locked_underlying NUMERIC NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Settlement Worker heartbeat (for monitoring)
CREATE TABLE worker_heartbeat (
    worker_name TEXT PRIMARY KEY,
    last_cycle_at TIMESTAMPTZ NOT NULL,
    cycle_count BIGINT NOT NULL DEFAULT 0,
    last_error TEXT
);
```

---

## 11. Infrastructure

### 11.1 Docker Compose Services

```yaml
services:
  api-server:
    build: .
    command: ["node", "dist/api-server.js"]
    ports: ["3000:3000"]
    restart: unless-stopped
    depends_on: [postgres, redis]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      retries: 3

  settlement-worker:
    build: .
    command: ["node", "dist/settlement-worker.js"]
    ports: ["3100:3100"]
    restart: unless-stopped
    depends_on: [postgres, redis]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3100/health"]
      interval: 30s
      retries: 3

  postgres:
    image: postgres:16
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  web:
    build: ../safira-web
    ports: ["8080:8080"]

  nginx:
    image: nginx:alpine
    ports: ["80:80", "443:443"]
    volumes: ["./nginx.conf:/etc/nginx/nginx.conf"]
```

6 containers total. Simple to deploy, monitor, and debug.

### 11.2 Environments

| Environment | Blockchain | sWBTC/sWETH | Purpose |
|---|---|---|---|
| Local | Stellar standalone / Soroban sandbox | Mintable test tokens (admin faucet) | Development |
| Staging | Stellar testnet | Mintable test tokens (admin faucet) | Integration testing |
| Production | Stellar mainnet | LayerZero bridge from Ethereum | Live trading |

### 11.3 Deployment

- VPS provider: Hetzner or DigitalOcean
- Docker Compose for orchestration
- Nginx reverse proxy with SSL (Let's Encrypt)
- Environment configs via Docker secrets for sensitive values (keys), `.env` for non-sensitive config

### 11.4 Logging & Monitoring

- **Structured logging:** JSON format via `pino` (Fastify's default logger)
- **Log aggregation:** Loki + Grafana on VPS (lightweight) or stdout → Docker logging driver
- **Alerts:** Settlement Worker heartbeat stale > 2 min, oracle price stale > 10 min, failed settlement tx, MM disconnect
- **Metrics:** Prometheus endpoint on both processes → Grafana dashboards

---

## 12. Security

### 12.1 On-chain

- Full seller collateralization enforced by contract — no undercollateralized positions
- Atomic fee collection in `open_option` — cannot be bypassed via direct contract calls
- Admin functions split into two keys: **oracle key** (settle only) and **config key** (fees, token whitelist)
- No admin mint for sWBTC/sWETH in production — only bridge-verified messages
- Oracle expiry price bounded by ±50% of strike (prevents manipulation by compromised key)
- `pause()` can halt new openings without affecting existing positions
- MM deposits locked in contract — no rehypothecation

### 12.2 Off-chain

- MM quote signatures: **Ed25519** (asymmetric, non-repudiable — MM cannot deny submitting a quote)
- Quote nonce + one-time use prevent replay attacks
- Rate limiting: 10 RFQ/min per user, 100 req/min general
- Input validation on all routes (Fastify JSON Schema validation)
- API key + Stellar Ed25519 wallet signature authentication
- TradingView webhook validation: shared secret in `X-Safira-Secret` header + IP allowlist if supported
- Internal modules communicate via function calls (no network attack surface between modules)
- Sensitive keys stored as Docker secrets, not in `.env` files

### 12.3 Operational

- Settlement Worker health check every 30s, auto-restart on failure, heartbeat table for monitoring
- All settlement operations are idempotent — safe to retry after crash
- Oracle staleness detection: reject prices > 5 min old, alert on backup oracle divergence
- Bridge monitoring: alert on stuck LayerZero messages > 30 min
- Reconciliation module every 5 min: on-chain state is source of truth
- Error retry policy: exponential backoff with max 3 retries for Soroban tx, then alert

---

## 13. Out of Scope

The following are explicitly excluded from this MVP design:

- **LP module** — external liquidity providers, NAV accounting, Performance Fee (15% HWM), withdrawal queue, share-based participation
- **Early close / buyback** of open options before expiry
- **Third-party wrapped tokens** — only protocol-issued sWBTC/sWETH
- **Permissionless market making** — MMs are approved and permissioned
- **Cross-token substitution** — no automatic swaps between different BTC/ETH wrappers
- **Mobile application**
- **Fiat on/off ramp**
- **Multi-leg strategies** (spreads, strangles, butterflies) — Wheel only for MVP

These may be addressed in subsequent design documents.
