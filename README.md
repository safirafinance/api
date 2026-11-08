# Safira API

Backend for **Stellar Options** — fully collateralized options trading platform on Stellar/Soroban.

Users trade covered calls, cash-secured puts, and protective puts on tokenized BTC/ETH. The Automated Wheel strategy adapts trades dynamically via a TradingView-based Prediction Engine.

## Architecture

Two-process system sharing PostgreSQL and Redis:

- **API Server** (port 3000) — Fastify HTTP + WebSocket. Handles RFQ flow, market maker gateway, positions, strategies, risk checks, auth.
- **Settlement Worker** (port 3100) — background process polling oracle prices, settling expired options, reconciling on-chain state.

```
safira-api/
├── src/
│   ├── modules/
│   │   ├── rfq/            # Quote request lifecycle, best execution
│   │   ├── mm-gateway/     # WebSocket for market makers (JSON-RPC 2.0)
│   │   ├── positions/      # Position CRUD, on-chain mirroring
│   │   ├── strategy/       # Automated Wheel state machine
│   │   ├── prediction/     # TradingView webhook receiver
│   │   ├── risk/           # Position limits, exposure checks
│   │   ├── vault/          # Internal Vault pricing (Black-76)
│   │   ├── notifications/  # WebSocket push, webhooks
│   │   └── auth/           # Wallet signatures, API keys
│   ├── routes/
│   ├── shared/             # DB, Redis, Stellar SDK, types
│   └── config/
├── packages/
│   └── sdk/                # @safira/sdk — TypeScript SDK
└── docs/
```

## Tech Stack

| Layer | Tech |
|-------|------|
| Runtime | Node.js + TypeScript |
| Framework | Fastify |
| Database | PostgreSQL + Drizzle ORM |
| Cache / PubSub | Redis |
| Blockchain | Stellar / Soroban |
| Smart Contract | Rust (`options_core`) |
| Infrastructure | Docker Compose + Nginx |

## How It Works

1. User requests a quote (RFQ) for an option
2. Quotes collected from approved Market Makers + protocol Vault over 2-second window
3. Best price selected after fees
4. User confirms → unsigned Soroban transaction returned
5. User signs with wallet → API co-signs → submitted to Stellar
6. At expiry: Settlement Worker reads oracle price and settles automatically

All positions are fully collateralized on-chain. No margin, no liquidations.

## Products

| Product | What it does |
|---------|-------------|
| **Sell Higher** | Covered call — lock underlying, earn premium |
| **Buy Lower** | Cash-secured put — lock USDC, earn premium |
| **Protect** | Long put — buy downside protection |
| **Automated Wheel** | Alternates puts/calls, adapts via Prediction Engine signals |

## Infrastructure

```yaml
# Docker Compose services
api-server:        # port 3000 — API + WebSocket
settlement-worker: # port 3100 — oracle, settlement, reconciliation
postgres:          # port 5432
redis:             # port 6379
web:               # port 8080 — Next.js frontend (safira-web repo)
nginx:             # ports 80, 443 — reverse proxy + SSL
```

## Related Repos

| Repo | Purpose |
|------|---------|
| `safira-contracts` | Soroban smart contracts (Rust) |
| `safira-web` | Next.js frontend |
| `safira-ui-kit` | React component library (post-MVP) |

## Docs

- [MVP Design Document](docs/2026-11-08-stellar-options-mvp-design.md) — full architecture, API specs, settlement rules, fee model
