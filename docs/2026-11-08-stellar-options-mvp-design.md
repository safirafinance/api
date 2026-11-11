# Stellar Options — MVP Design Document

> **Version:** 3.0
> **Date:** November 8, 2026
> **Scope:** Full MVP without LP module
> **Status:** Approved for implementation
> **Reference:** Rysk Finance V12 (RFQ model, physical settlement patterns)

---

## 1. Overview

Stellar Options is a fully collateralized options trading and strategy infrastructure platform built on Stellar/Soroban. This design document covers the MVP: options trading (Covered Call, Cash-Secured Put, Long Put), Automated Wheel strategy with TradingView-driven Prediction Engine, RFQ-based liquidity from approved Market Makers and an internal Vault, and developer integration tools (Open API, SDK, UI Kit).

**Excluded from this MVP:** LP module (external liquidity providers, NAV, Performance Fee, withdrawal queue).

### 1.1 Option Lifecycle

1. The user or authorized Wheel strategy selects the exact token (sWBTC/sWETH), option type, size, strike, expiry, and applicable constraints.
2. The RFQ Router requests quotes from eligible approved MMs and the internal Vault.
3. Quotes are checked for executability (valid signature, instrument match, MM active) and compared on **best net price after Trading Fee and Opening Base Fee**.
4. The user approves the selected terms and signs the Soroban transaction.
5. In one atomic `open_option` transaction: seller's full collateral is locked, buyer's premium is paid, fees are collected, and the position is created on-chain.
6. The position remains open until expiry. **No early close or buyback.**
7. At expiry, the Settlement Worker fetches the verified oracle price and calls `determine_expiry`:
   - **OTM:** Collateral is released to seller automatically. Position closed. Seller keeps premium.
   - **ITM:** A 2-hour settlement window opens. Counterparty (buyer) is notified to deliver the counter-asset.
8. Within the settlement window:
   - **Delivery received:** Contract atomically exchanges assets at strike price via `deliver_settlement`. Position → `Settled`.
   - **No delivery:** After 2h, `finalize_default` releases seller collateral. Position → `Defaulted`. Seller keeps premium.
9. The resulting state, asset movements, and fees are reflected in the Web App, partner APIs, and position events audit trail.

### 1.2 Tech Stack Summary

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
| Oracle | Chainlink Data Feeds (off-chain API) + independent backup |

### 1.3 Repository Structure

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
| **Settlement Worker** | 3100 (health check only) | Monitors Soroban for expiry events, oracle price feeds, physical settlement, reconciliation |

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
│   ├── oracle/           # Chainlink API polling, backup oracle, freshness checks
│   ├── settlement/       # Expiry scanning, settlement window management, collateral release
│   └── reconciliation/   # On-chain vs off-chain state comparison, drift detection
├── shared/               # Shared with API Server: DB client, Stellar SDK, types
└── config/
```

Runs on a polling loop (configurable interval, default 15s). Exposes health and readiness endpoints on port 3100 (`GET /health`, `GET /ready`). Communicates with API Server through the shared PostgreSQL database and Redis pub/sub (for event notifications).

**Resilience:** Docker restart policy `unless-stopped`. Health check every 30s. If the worker misses 3 consecutive cycles, Docker restarts it. All settlement operations are **idempotent** — safe to retry after crash/restart. Worker logs last successful cycle timestamp to Redis; API Server can alert if stale.

**Concurrent write safety:** Settlement Worker owns all expiry/settlement state transitions on the `positions` table. API Server owns position creation and read queries. When updating position status, both processes use `SELECT ... FOR UPDATE` row-level locks to prevent race conditions. Clear ownership of state transitions prevents conflicts. **Event ordering:** Settlement Worker always commits DB transaction before publishing to Redis pub/sub — guarantees that API Server reads consistent state when processing events. If a Redis message is lost, the Reconciliation module (every 5 min) detects and corrects any missed state transitions.

### 2.4 Shared Infrastructure

- **PostgreSQL** — single database, shared by both processes. Migrations via Drizzle Kit.
- **Redis** — caching (quotes, oracle prices) + pub/sub (Settlement Worker → API Server event notifications)
- **Soroban** — `options_core` contract on Stellar testnet (dev/staging) and mainnet (production)

---

## 3. Soroban Contract: `options_core`

Single monolith contract handling all on-chain logic. Written in Rust using the Soroban SDK.

Fully collateralized positions with physical settlement. Seller locks full collateral at opening. At expiry, OTM options release collateral automatically. ITM options enter a settlement window where the counterparty delivers the required asset.

### 3.1 Data Structures

```rust
pub enum OptionType {
    Call,
    Put,
}

pub enum OptionStatus {
    Open,              // Position created, collateral locked, awaiting expiry
    AwaitingSettlement, // Expired ITM — waiting for counterparty delivery within settlement window
    Settled,           // Physical settlement completed — assets exchanged
    Expired,           // Expired OTM — collateral returned to seller
    Defaulted,         // Settlement window elapsed without delivery — collateral returned to seller
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
    pub oracle_expiry_price: i128,    // set at expiry determination, 0 until then
    pub spot_price_at_opening: i128,  // market price when position opened (for display/P&L)
    pub settlement_deadline: u64,     // expiry + 2h for ITM options, 0 for OTM
    pub trading_fee: i128,            // USDC
    pub opening_base_fee: i128,       // USDC
    pub created_at: u64,              // ledger timestamp
}
```

### 3.2 Entry Points

| Function | Auth | Description |
|---|---|---|
| `open_option(params: OpenOptionParams)` | Buyer signs | Atomic: lock seller collateral + transfer premium from buyer to seller + collect fees + create position. MM/Vault provides server-side signature via signed quote commitment. |
| `determine_expiry(option_id, expiry_price)` | Oracle admin key | Called by Settlement Worker after expiry. Sets oracle price, determines ITM/OTM. OTM → releases collateral immediately (`Expired`). ITM → sets `settlement_deadline` = now + 2h (`AwaitingSettlement`). |
| `deliver_settlement(option_id)` | Counterparty | Called within settlement window. Counterparty delivers required asset, contract exchanges atomically. Status → `Settled`. |
| `finalize_default(option_id)` | Oracle admin key | Called after settlement window expires with no delivery. Returns collateral to seller. Status → `Defaulted`. |
| `get_option(option_id)` | Public | Read position data |
| `configure_fees(trading_fee_bps, base_fee)` | Contract admin | Update fee parameters (prospective only) |
| `add_supported_token(token_address)` | Contract admin | Whitelist token for trading |
| `pause() / unpause()` | Contract admin | Emergency: prevent new position openings. Existing positions continue to expiry and settlement normally. Does not affect `determine_expiry`, `deliver_settlement`, or `finalize_default`. |

**Removed `get_options_by_address`:** Querying positions by address is done off-chain via PostgreSQL indexes. On-chain contract only supports lookup by `option_id`, avoiding expensive iteration over storage.

### 3.3 Fee Enforcement

- **Trading Fee:** 1% of premium, collected at `open_option`
- **Opening Base Fee:** Fixed USDC amount (configurable), collected at `open_option`
- Both fees are enforced in the same atomic Soroban transaction that opens the position
- Direct contract calls cannot bypass fees — logic is in the contract itself
- **Formula:** `trading_fee = 0.01 * premium`. Simple and matching the original spec. No cap needed — premium is already a small fraction of notional value.

### 3.4 Collateral Rules

| Option Type | Seller Collateral | Buyer Pays at Opening | Buyer Delivers at Settlement (ITM) |
|---|---|---|---|
| Covered Call (Sell Higher) | `quantity` of underlying token (sWBTC/sWETH) | Premium (USDC) | `strike_price * quantity` USDC |
| Cash-Secured Put (Buy Lower) | `strike_price * quantity` in USDC | Premium (USDC) | `quantity` of underlying token |
| Long Put (Protect) | MM/Vault is the seller — posts `strike_price * quantity` USDC | Premium (USDC) | `quantity` of underlying token |

All seller collateral locked in the contract until settlement or expiry. No rehypothecation. No liquidations.

**Buyer does NOT lock settlement funds at opening** — only pays the premium. At settlement (ITM), the buyer must deliver the counter-asset within the 2-hour settlement window. If the buyer fails to deliver, the option defaults and seller collateral is returned.

### 3.5 Physical Settlement Flow

Physical settlement — parties exchange actual assets at the strike price. Same model as Rysk Finance.

**At expiry:**
1. Option reaches `expiry` timestamp
2. Settlement Worker polls for expired options with `status = Open`
3. Oracle module fetches verified expiry price from Chainlink API (+ backup source)
4. Settlement Worker calls `determine_expiry(option_id, expiry_price)` on Soroban

**OTM (no settlement needed):**
- Covered Call: `expiry_price < strike_price` → contract returns underlying to seller. Status → `Expired`
- Cash-Secured Put: `expiry_price > strike_price` → contract returns USDC to seller. Status → `Expired`
- Seller keeps premium in all cases

**ITM (settlement window opens):**
- Contract sets `settlement_deadline = now + 7200s` (2 hours). Status → `AwaitingSettlement`
- Settlement Worker notifies counterparty (buyer for calls, buyer for puts) via API Server → WebSocket/webhook
- Counterparty has 2 hours to call `deliver_settlement(option_id)` with the required asset
- On delivery: contract atomically exchanges assets at strike price. Status → `Settled`
  - **Covered Call ITM:** Buyer delivers `strike * qty` USDC → receives `qty` underlying from contract. Seller receives USDC.
  - **Cash-Secured Put ITM:** Buyer delivers `qty` underlying → receives `strike * qty` USDC from contract. Seller receives underlying.

**Default (no delivery within 2h):**
- Settlement Worker calls `finalize_default(option_id)` after deadline
- Seller collateral returned. Seller keeps premium. Status → `Defaulted`
- Buyer loses only the premium paid at opening

**Seller keeps premium in ALL outcomes** (transferred at opening).

If both oracles are unavailable at expiry: settlement postponed. Worker retries on next cycle. Max postponement 24h, after which option defaults and collateral returns to seller.

### 3.6 Transaction Authorization

Opening an option requires buyer authorization. The MM/Vault (seller) commits via their signed quote.

**Flow:**
1. MM provides a signed quote (Ed25519) during RFQ — this is their binding commitment
2. API Server builds the `open_option` Soroban transaction with all parameters from the accepted quote
3. Returns unsigned transaction XDR to the buyer
4. Buyer signs with their Stellar wallet (Freighter/WalletConnect)
5. API Server adds the MM/Vault operational signature server-side (authorized by the quote commitment)
6. Fully signed transaction submitted to Stellar network

**For Wheel strategy trades:** Vault's operational key signs the seller side. User's pre-authorized strategy delegation key signs the buyer side (configured during strategy setup).

**Soroban auth:** Both `seller` and `buyer` addresses must call `require_auth()` in the contract. The transaction includes auth entries for both parties. The MM's signed quote serves as off-chain proof of intent; their on-chain auth is provided by the API Server using the MM's deposited operational key.

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
| Server → MM | `settlement.pending` | ITM position entering settlement window — MM must deliver counter-asset |
| Server → MM | `settlement.result` | Settlement outcome for MM's positions |
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

### 4.4 MM Settlement Model

MMs do NOT pre-deposit or lock funds. They commit to settlement by providing signed quotes. At expiry, if the option is ITM, the MM has a **2-hour settlement window** to deliver the counter-asset on Stellar. This matches Rysk Finance's approach.

**MM obligations at settlement (when MM is seller):**
- **ITM Covered Call:** MM locked underlying at opening. Buyer delivers USDC. Contract exchanges atomically. No additional MM action needed — collateral already locked.
- **ITM Cash-Secured Put:** MM locked USDC at opening. Buyer delivers underlying. Contract exchanges atomically. No additional MM action needed.

**MM obligations at settlement (when MM is buyer):**
- **ITM Covered Call:** MM must deliver `strike * qty` USDC within 2h. Receives underlying.
- **ITM Cash-Secured Put:** MM must deliver `qty` underlying within 2h. Receives USDC.

If MM fails to deliver within the window → option defaults, seller collateral returned, seller keeps premium. MM loses nothing except reputation and the premium they paid.

**MM risk management:** MMs are responsible for managing their own hedges on external venues (CEXs, other DEXs). They do not need a local derivatives market on Stellar to hedge.

### 4.5 Best Execution Logic

1. Collect all quotes within 2-second aggregation window
2. Filter: valid Ed25519 signature, not expired, correct instrument, MM is active
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

**Important:** Signals only trigger the **first** position opening in a Wheel cycle. After that, the Wheel auto-spins (see Section 5.2).

### 5.2 Strategy Module (Automated Wheel)

Part of the API Server process. Executes Automated Wheel strategy based on Prediction Engine signals.

**Core Wheel logic:**

The TradingView signal triggers ONLY the first position in a cycle. After that, the Wheel automatically cycles to return the user to their original deposit asset (USDC) plus all collected premiums.

**Cycle flow:**

```
Signal (sell_put or sell_call) → Open FIRST position → Auto-settle at expiry
  → If OTM: collateral returned, premium kept → IDLE (wait for next signal)
  → If ITM: physical settlement, user receives counter-asset
    → Auto-open REVERSE position at SAME STRIKE, 7-day expiry
      → If OTM: keep asset + premium → auto-open another reverse at same strike, 7d
      → If ITM: physical settlement, returned to original asset → IDLE + all premiums collected
```

**Key rules:**
1. **Signal triggers first opening only** — either `sell_put` (start with USDC, aim to buy underlying) or `sell_call` (start with underlying, aim to sell for USDC)
2. **After ITM settlement:** Wheel automatically opens the reverse position to return to the original deposit asset
3. **Same strike always:** The reverse position uses the SAME strike price as the initial position. Goal: recover the exact original deposit amount
4. **7-day expiry:** All auto-reverse positions use 7-day expiry
5. **OTM on reverse = repeat:** If the reverse position expires OTM, the user still holds the "wrong" asset. Wheel auto-opens another reverse at the same strike, 7d. Repeats until ITM settlement returns user to the original asset
6. **Goal:** On completion of a full cycle, user has their original deposit back + ALL premiums collected along the way

**Example cycle (starting with USDC via sell_put):**
1. Signal: `sell_put` BTC at $95,000, 14d → user locks $95,000 USDC
2. Expiry: BTC at $93,000 (ITM) → user receives 1 BTC, pays $95,000 → now holds 1 BTC
3. Auto-open: `sell_call` BTC at $95,000, 7d → user locks 1 BTC
4. Expiry: BTC at $94,000 (OTM) → user keeps 1 BTC + call premium
5. Auto-open: `sell_call` BTC at $95,000, 7d → user locks 1 BTC again
6. Expiry: BTC at $96,000 (ITM) → user receives $95,000 USDC, gives 1 BTC
7. Cycle complete: user back to $95,000 USDC + put premium + 2x call premiums → IDLE

**On signal received (first opening only):**
1. Validate signal against user's strategy config (allowed assets, expiries, capital)
2. Check `RiskModule.validatePosition()` for position limits
3. Determine strike (from signal hint or auto-select based on config delta range)
4. Request RFQ → get best quote from MM/Vault as counterparty
5. If quote meets minimum premium threshold → build Soroban tx → open position
6. If not → skip, log reason, notify user

**On auto-reverse (after ITM settlement):**
1. Use the SAME strike price as the settled position
2. Set expiry = 7 days from now
3. Request RFQ → get best quote
4. If quote received → open reverse position automatically
5. If no quote → retry on next Settlement Worker cycle (every 15s) for up to 1 hour. If still no quote after 1 hour → stop strategy (`active = false`), alert user. User keeps the asset and can restart the strategy manually or withdraw.

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
  allowed_expiries: string[];         // e.g. ["7d", "14d", "30d"] (for signal-triggered first position)
  active: boolean;
  current_state: WheelState;          // persisted state machine position
  current_position_id: string | null; // active position if any
  underlying_balance: string;         // underlying received from settled puts
  original_deposit_asset: "USDC" | "sWBTC" | "sWETH";  // what user started with
  initial_strike: string | null;      // strike from the first position (used for all auto-reverses)
  cycle_premiums_collected: string;   // total premiums collected in current cycle
}

type WheelState = "idle" | "put_open" | "holding_underlying" | "call_open" | "holding_usdc";
```

**State machine:**

```
Signal(sell_put) → IDLE → PUT_OPEN
Signal(sell_call) → IDLE → CALL_OPEN

PUT_OPEN + OTM expiry → IDLE (USDC returned, premium kept)
PUT_OPEN + ITM settle → HOLDING_UNDERLYING (received underlying)
  → auto-open sell_call at same strike, 7d → CALL_OPEN

CALL_OPEN + ITM settle → IDLE (received USDC = original deposit, cycle complete)
CALL_OPEN + OTM expiry → HOLDING_UNDERLYING (still holds underlying)
  → auto-open sell_call at same strike, 7d → CALL_OPEN (repeat)

CALL_OPEN(started from sell_call signal) + OTM expiry → HOLDING_USDC
  → auto-open sell_put at same strike, 7d → PUT_OPEN (repeat)
CALL_OPEN(started from sell_call signal) + ITM settle → IDLE (received underlying = original deposit)
```

**Settlement assumption:** Wheel state machine transitions only on `Settled` or `Expired` statuses. `Defaulted` is a contract-level safety net (prevents collateral from being locked forever), not a Wheel business flow. Approved MMs are economically incentivized to deliver within the settlement window (ITM = profitable settlement for buyer). If a `Defaulted` status occurs — strategy is paused, admin is alerted, manual resolution required.

**Fan-out:** MVP supports one active Wheel strategy per user per asset. No fan-out scaling needed. One TradingView signal triggers evaluation for all active configs for that asset, processed sequentially.

Settlement events arrive via Redis pub/sub from Settlement Worker → Strategy module updates state and triggers auto-reverse if needed.

---

## 6. Oracle & Settlement Worker

The Settlement Worker is a **separate Node.js process** (not part of the API Server). It handles all time-sensitive background work: oracle price feeds, settlement lifecycle, and state reconciliation.

### 6.1 Oracle Module (inside Settlement Worker)

**Chainlink on Stellar:** Chainlink does not run natively on Stellar/Soroban. The oracle module fetches prices from Chainlink Data Feeds via their off-chain API (HTTP). The Settlement Worker is the authorized oracle operator that submits verified prices to the Soroban contract.

- Polls Chainlink BTC/USD and ETH/USD price feeds every 15 seconds via HTTP API
- Independent backup oracle (configurable provider, e.g. Pyth, Redstone, or aggregated CEX prices)
- **Freshness check:** reject prices older than configurable threshold (default 5 min)
- **Cross-source validation:** if primary (Chainlink) and backup differ > 2% → flag as suspicious, use primary but log alert
- **Hard deviation:** if differ > 10% → refuse to settle, alert, wait for human review
- Caches latest prices in Redis key `oracle:price:{asset}` with TTL (shared with API Server for display/pricing)
- At expiry: determines definitive expiry price using both sources with strict validation
- If both oracles unavailable → settlement postponed, retries on next cycle. Max postponement 24h, then option defaults.

**No on-chain price bounds check.** The contract trusts the authorized oracle operator to submit correct prices. Price validation happens entirely off-chain via multi-source verification. This avoids blocking legitimate settlements during extreme market moves (e.g. a real 60% crash would be blocked by a ±50% bound).

### 6.2 Oracle Key Security

The Settlement Worker signs `determine_expiry()` and `finalize_default()` transactions with a dedicated oracle admin key.

- **Storage:** Environment variable injected via Docker secret (not in `.env` file on disk)
- **Scope:** This key can ONLY call `determine_expiry()`, `finalize_default()`, and `pause()`/`unpause()`. Cannot transfer funds, change fees, or modify token whitelist. Separate admin key for contract configuration.
- **Rotation:** Key can be rotated via contract admin function without redeploying.

### 6.3 Settlement Module (inside Settlement Worker)

Runs on a polling loop (configurable interval, default 15s).

**Phase 1 — Expiry determination:**
1. Scan `positions` table for `expiry <= now AND status = 'open'`
2. For each expired position:
   a. Fetch verified expiry price from oracle module
   b. Build and sign `determine_expiry(option_id, expiry_price)` Soroban transaction
   c. Submit to Stellar network
   d. OTM: collateral released automatically on-chain. Update DB status → `expired`
   e. ITM: on-chain status → `awaiting_settlement`, deadline = now + 2h. Update DB.
   f. Notify counterparty via Redis pub/sub → API Server → WebSocket
3. On tx failure: log error, retry on next cycle (idempotent — contract checks status)

**Phase 2 — Settlement window monitoring:**
1. Scan `positions` table for `status = 'awaiting_settlement'`
2. Check if `deliver_settlement` has been called on-chain (reconciliation)
3. If settlement_deadline passed without delivery:
   a. Call `finalize_default(option_id)` → collateral returned to seller
   b. Update DB status → `defaulted`
4. If delivery completed on-chain → update DB status → `settled`

**Phase 3 — Strategy auto-reverse:**
1. After ITM settlement, check if position belongs to a Wheel strategy
2. If yes → trigger auto-reverse position opening via Strategy module (Redis pub/sub → API Server)
3. Strategy module handles RFQ and position opening with same strike, 7d expiry

**Idempotency:** All contract calls check current on-chain status before acting. Calling `determine_expiry()` on an already-determined position is a no-op. Safe to retry after crashes.

Publish all settlement events to Redis pub/sub channel `settlement:events`. API Server's notification module picks up events → pushes to users via WebSocket.

### 6.4 Reconciliation Module

Runs every 5 minutes:
1. Fetch all `Open` and `AwaitingSettlement` positions from DB
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
| Positions | Active, awaiting settlement, settled, expired, defaulted positions with details |
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
        CHECK (status IN ('open', 'awaiting_settlement', 'settled', 'expired', 'defaulted')),
    oracle_expiry_price NUMERIC,
    spot_price_at_opening NUMERIC,       -- market price at time of opening (for display/P&L)
    settlement_deadline TIMESTAMPTZ,     -- expiry + 2h for ITM options, NULL for OTM
    trading_fee NUMERIC NOT NULL,
    opening_base_fee NUMERIC NOT NULL,
    open_tx_hash TEXT,
    settle_tx_hash TEXT,
    deliver_tx_hash TEXT,                -- counterparty delivery tx (for physical settlement)
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    settled_at TIMESTAMPTZ
);

CREATE INDEX idx_positions_status_expiry ON positions (status, expiry);
CREATE INDEX idx_positions_seller ON positions (seller_address);
CREATE INDEX idx_positions_buyer ON positions (buyer_address);
CREATE INDEX idx_positions_settlement ON positions (status, settlement_deadline)
    WHERE status = 'awaiting_settlement';

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
    stellar_address TEXT NOT NULL,         -- for on-chain settlement
    status TEXT NOT NULL DEFAULT 'active'
        CHECK (status IN ('active', 'suspended')),
    config JSONB NOT NULL DEFAULT '{}',   -- supported assets, size limits
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
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
        CHECK (current_state IN ('idle', 'put_open', 'holding_underlying', 'call_open', 'holding_usdc')),
    current_position_id UUID REFERENCES positions(id),
    underlying_balance NUMERIC NOT NULL DEFAULT 0,
    original_deposit_asset TEXT NOT NULL DEFAULT 'USDC',
    initial_strike NUMERIC,              -- strike from first position, used for all auto-reverses
    cycle_premiums_collected NUMERIC NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (user_address, asset)         -- one active Wheel per user per asset
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
    asset TEXT PRIMARY KEY,
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

-- Position events (audit trail for all state transitions)
CREATE TABLE position_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    position_id UUID NOT NULL REFERENCES positions(id),
    event_type TEXT NOT NULL,            -- 'opened', 'expiry_determined', 'settlement_requested',
                                         -- 'settled', 'expired', 'defaulted', 'auto_reverse_opened'
    old_status TEXT,
    new_status TEXT,
    actor TEXT,                          -- 'user', 'mm:<id>', 'settlement_worker', 'strategy_engine'
    tx_hash TEXT,
    metadata JSONB DEFAULT '{}',         -- additional context (oracle price, quote details, etc.)
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_position_events_position ON position_events (position_id);
CREATE INDEX idx_position_events_type ON position_events (event_type, created_at);
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
- **Alerts:** Settlement Worker heartbeat stale > 2 min, oracle price stale > 10 min, failed settlement tx, MM disconnect, settlement window approaching deadline
- **Metrics:** Prometheus endpoint on both processes → Grafana dashboards
- **Health endpoints:** Both processes expose `GET /health` (liveness) and `GET /ready` (dependency connectivity check: DB, Redis, Stellar RPC)

---

## 12. Security

### 12.1 On-chain

- Full seller collateralization enforced by contract — no undercollateralized positions
- Atomic fee collection in `open_option` — cannot be bypassed via direct contract calls
- Admin functions split into two keys: **oracle key** (determine_expiry, finalize_default, pause) and **config key** (fees, token whitelist)
- No admin mint for sWBTC/sWETH in production — only bridge-verified messages
- `pause()` halts new position openings only — existing positions continue to expiry and settlement normally
- 2-hour settlement window for ITM physical delivery; automatic default after window expires

### 12.2 Off-chain

- MM quote signatures: **Ed25519** (asymmetric, non-repudiable — MM cannot deny submitting a quote)
- Quote nonce + one-time use prevent replay attacks
- Rate limiting: 10 RFQ/min per user, 100 req/min general
- Input validation on all routes (Fastify JSON Schema validation)
- API key + Stellar Ed25519 wallet signature authentication
- TradingView webhook validation: shared secret in `X-Safira-Secret` header + IP allowlist if supported
- Internal modules communicate via function calls (no network attack surface between modules)
- Sensitive keys stored as Docker secrets, not in `.env` files
- Oracle price validation: multi-source verification off-chain (Chainlink + backup), cross-source deviation checks, freshness enforcement

### 12.3 Operational

- Settlement Worker health check every 30s, auto-restart on failure, heartbeat table for monitoring
- All settlement operations are idempotent — safe to retry after crash
- Oracle staleness detection: reject prices > 5 min old, alert on backup oracle divergence
- Bridge monitoring: alert on stuck LayerZero messages > 30 min
- Reconciliation module every 5 min: on-chain state is source of truth
- Error retry policy: exponential backoff with max 3 retries for Soroban tx, then alert
- Position events audit trail: all state transitions logged with actor, tx hash, and metadata
- Row-level locking (`SELECT ... FOR UPDATE`) on position state transitions to prevent concurrent write conflicts

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
- **On-chain position queries by address** — use PostgreSQL indexes instead

These may be addressed in subsequent design documents.
