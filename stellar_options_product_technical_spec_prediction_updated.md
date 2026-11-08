# Stellar Options — Product and Technical Specification

> **Version:** 2.7  
> **Date:** November 3, 2026  
> **Ecosystem:** Stellar / Soroban  
> **Purpose:** Product, business model, architecture, risk principles, and integration capabilities.

---

## 1. Executive Summary

Stellar Options is an options trading and strategy infrastructure platform built on Stellar and Soroban. It enables holders of tokenized BTC and ETH to earn premiums by selling fully collateralized options, purchase downside protection, and access rule-based strategies through a streamlined interface.

The platform combines quotes from **approved external market makers (MMs)** with a **protocol-operated Vault as an additional, non-primary source of liquidity**. A request-for-quote (RFQ) router checks whether quotes are executable and selects the best net price for the user. On-chain contracts handle collateral, premiums, mandatory fees, and physical settlement.

End-user experiences include **Earn, Sell Higher, Buy Lower, and Protect**. Automated Wheel strategies operate exclusively through a Strategy Vault. The same underlying infrastructure is available to ecosystem partners through an **Open API, SDK, and optional UI Kit**. The first-party Web App is a client of those common platform capabilities rather than a separate trading implementation.

Returns are variable and never guaranteed. Options, market making, wrapped assets, bridging, and automated strategies involve financial, operational, and counterparty risk.

## 2. Product Opportunity and Value

### End users

Holding BTC or ETH presents familiar decisions: at what price to sell, at what price to buy, and how to manage downside risk. Conventional options interfaces impose substantial complexity through strikes, expiries, margin concepts, implied volatility, and multi-step execution.

Stellar Options expresses common objectives as understandable actions and displays the premium, potential outcomes, locked collateral, fees, and execution terms before a transaction is signed. Multiple quote sources improve price discovery. Strategy automation reduces manual repetition without promising fixed yield.

### Ecosystem partners

Wallets, DeFi applications, exchanges, and other Web3 products can embed option transactions and strategies without creating their own pricing, market-maker integration, settlement contracts, or liquidity network. A single trading and settlement infrastructure supports all distribution channels.

## 3. Target Users and Use Cases

| Audience | Objective | Product |
|---|---|---|
| BTC/ETH holder | Sell at a target price while collecting an option premium | Sell Higher / Covered Call |
| USDC holder | Buy BTC/ETH at a target price while collecting a premium | Buy Lower / Cash-Secured Put |
| Strategy user | Repeatedly use option premiums within predefined limits | Automated Wheel |
| BTC/ETH holder seeking downside protection | Establish a minimum sale price for a defined period | Protect / Long Put |
| Approved professional market maker | Access RFQ order flow and execute options | MM Integration |
| Liquidity provider | Allocate capital to disclosed options risks | LP Vault |
| Wallet or DeFi partner | Offer options within an existing product | Open API, SDK, UI Kit |

## 4. User-Facing Products

### 4.1 Sell Higher — Covered Call

A BTC/ETH holder specifies an amount, target selling price (strike), and expiry, and sells a fully collateralized Call. The underlying token is locked as collateral and the option premium is received at opening.

For example, if BTC is priced at $100,000, a user may collateralize one BTC token, sell a seven-day Call with a $105,000 strike, and receive an illustrative $900 premium. If the option finishes out of the money (OTM), the token is released and the premium is retained. If it finishes in the money (ITM) **and the buyer supplies the settlement asset**, the token is exchanged for USDC at the strike. The seller forgoes appreciation above the strike but still bears underlying price risk on an unexercised position. A buyer default may prevent the expected exchange.

### 4.2 Buy Lower — Cash-Secured Put

A user locks enough USDC to buy the selected BTC/ETH token at a stated strike and receives a premium for selling a Put. For example, one BTC token with a $95,000 strike requires $95,000 USDC of collateral, subject to the precise contract terms. If OTM at expiry, USDC is released. If ITM and the Put buyer delivers the underlying token, the seller purchases it at the strike. The token may then have a market value below the purchase cost; physical settlement is not guaranteed if the buyer fails to deliver.

### 4.3 Automated Wheel

The Wheel alternates Cash-Secured Puts and Covered Calls in response to the assets actually received through successful settlement. Starting with USDC, the strategy sells a Put; after successful assignment it may sell a Call against the received BTC/ETH token; after successful Call settlement it may resume Put writing.

**The Prediction Engine is a core decision-making component of Automated Wheel, not merely a Vault inventory analytics module.** It evaluates probabilistic price and volatility scenarios to help select strikes, expirations, and when to open or defer the next option. The objective is a more adaptive strategy and more consistent premium generation across different market conditions, without promising fixed returns. The Strategy Engine executes authorized decisions through the Strategy Vault, and the Risk Engine enforces hard constraints.

**Automated Wheel is available only through the Strategy Vault.** Strategy permissions constrain eligible assets, position size, expiry, strikes, minimum net premiums, and overall capital usage. If quotes are unavailable or unsuitable, the strategy skips an opening. Failure to complete physical settlement does not trigger a fictitious asset transition.

The Wheel is not a savings product and offers no fixed APY. Users remain exposed to adverse underlying price movements, capped upside on Calls, premiums that change over time, and counterparty non-delivery.

### 4.4 Protect — Long Put

A user holding the selected BTC/ETH token purchases a Put by paying an upfront premium. An ITM Put provides the contractual right to exchange that token for USDC at the strike, provided settlement terms are satisfied. The user is the **buyer** of protection, not a seller earning a premium.

Options are not closed early in the platform's defined operating model. They run through expiry and the applicable settlement window.

## 5. User Experience

The first-party Web App presents goal-oriented flows: Earn, Sell Higher, Buy Lower, and Protect. A user selects a supported asset, size, strategy, strike or desired target, and expiry, then views executable RFQ pricing and the net effect of fees.

Position screens distinguish active, expired, settled, failed-to-settle, and collateral-released states. The interface explains physical delivery, what happens when counterparties provide settlement assets, how fees are charged, and why projected yield or premium is not a guaranteed investment return.

**Asset selection is straightforward.** Where multiple BTC or ETH representations are supported, users choose the specific available token (for example, the protocol's sWBTC/sWETH or another provider's wrapped token). Each is a separate selectable asset. Options settle in the exact selected token; there is no implied automatic exchange or conversion between wrappers.

## 6. Hybrid Liquidity

### 6.1 Approved external market makers

The platform operates a **permissioned MM model**. Only approved market makers can submit executable RFQ quotes. Access is governed by eligibility, quote quality, risk limits, and operational requirements. Open permissionless market-making access is not part of the product model.

MMs pay premiums when purchasing options and must supply the required counter-asset within the defined expiry settlement window if exercise is applicable. They do **not** escrow the full prospective settlement amount for an option's entire life. Market makers independently choose how to hedge BTC/ETH exposure on any exchanges, venues, or chains, including spot, derivatives, and options markets. Local Stellar derivatives liquidity is not a prerequisite. They remain responsible for having the exact settlement token or USDC available on Stellar when needed.

### 6.2 Internal Vault as supplementary liquidity

A protocol-operated Vault participates as **an additional liquidity source**, not the mandatory or dominant MM. It can quote alongside approved external MMs, may decline requests, and is never entitled to preferential execution. It is initially funded with team and/or investor capital, with external LP participation available after operational validation under the published Vault rules.

### 6.3 Best Execution RFQ Router

The router first tests quote validity and executable conditions: instrument, selected underlying token, amount, strike, expiry, available capacity, authorization, validity period, and applicable limits. It then selects the **best net price after mandatory fees** among executable quotes. Own-Vault quotes have no advantage over external MM quotes.

A quote for a different tokenized representation of BTC/ETH is a different instrument and is not silently substituted. Quotes may be rejected or no trade opened if execution requirements are not met. Splitting a single trade across multiple counterparties is not required by the defined routing model.

## 7. Protocol Revenue and Fees

### 7.1 Trading Fee

The platform charges a **Trading Fee initially set to 1% of the actual option premium** on every successfully opened option, regardless of access channel or liquidity source. The rate is configurable and changes apply prospectively only, with appropriate disclosure.

### 7.2 Opening Base Fee

Every successfully opened option also incurs a **fixed Opening Base Fee denominated in USDC**, in addition to the Trading Fee. The amount is configurable; no particular numeric rate is specified by this document. There is no base fee for merely requesting or rejecting a quote, and the fee is not charged again at expiry.

### 7.3 Partner revenue share

Partners integrating through the Open API, SDK, or UI Kit may earn an agreed **share of the Trading Fee already charged to the user**. The protocol always retains its own portion of the Trading Fee. The Opening Base Fee is retained entirely by the protocol. A partner share does not add an extra mandatory surcharge to the user. Specific sharing percentages remain subject to partner agreements.

### 7.4 Vault Performance Fee

Where outside LP capital is admitted, the Vault charges a **15% Performance Fee** on new net profit above the applicable High-Water Mark. NAV is valued on a per-share basis so subscriptions and withdrawals are not mistaken for investment performance. Fee assessment occurs **weekly**. Open option positions and liabilities must be reflected in NAV, while realized-versus-unrealized profit and valuation policy are governed by the Vault accounting rules.

There is no additional dedicated MM fee, protocol settlement fee, or ongoing management fee under the currently defined fee schedule. Network and third-party costs are distinct.

### 7.5 Atomic collection

Opening Base Fee and Trading Fee are enforced and transferred **as part of the same successful Soroban transaction** that opens the position, transfers its premium, and locks collateral. The contract logic itself enforces the fee rules; a direct invocation of the contract cannot circumvent them by bypassing an API or SDK. Failed or reverted openings do not retain trading fees, although network charges may still apply.

## 8. Platform Architecture

| Component | Responsibility |
|---|---|
| Web App | First-party user experience built on the common integration interfaces |
| Supported Asset Catalog | Available BTC/ETH representations and the settlement token selected by the user |
| Public Open API | Market discovery, RFQ requests, transactions, positions, settlement status, and strategy access |
| SDK | Wallet integration, transaction construction and user authorization, shared client logic |
| UI Kit | Optional embeddable interfaces for asset selection, option entry, quotes, risk, fees, and positions |
| Approved MM Connectivity | Permissioned quote submission, eligibility, operational status, and controls |
| RFQ Router | Quote validation, executability checks, and best net execution |
| Pricing Engine | Risk-aware pricing of quotes originated by the internal Vault |
| Risk Engine | Hard and dynamic limits, inventory, position exposure, and capital eligibility |
| Prediction Engine | Market forecasts driving adaptive strike, expiry and entry decisions for Wheel, as well as Vault inventory decisions |
| Strategy Engine and Strategy Vault | Execution of user-authorized Wheel decisions informed by the Prediction Engine and bounded by risk controls |
| Oracle Layer | Underlying reference prices and expiry-price validation |
| Soroban Options Contracts | Position creation, collateralization, expiry, physical settlement, and fees |
| Monitoring and Operations | Reconciliation, alerts, settlement tracking, and incident handling |

Off-chain services handle requests, market data, decision support, routing, automation, and monitoring. **Soroban contracts are authoritative for collateral ownership, position state, enforceable fees, and asset transfers.**

### 8.1 Open API, SDK, and UI Kit are core architecture

These are not optional add-ons or separate future product categories. The first-party Web App and independent partner applications use the same underlying options, quotes, transaction, position, and settlement capabilities.

The API/SDK exposes supported assets, expiry and strike options, quote requests, Best Execution results, wallet-authorized transaction flows, portfolio history, exercise/expiry status, settlement results, and supported Strategy Vault operations. Events and status notifications can be provided for partner applications. The optional UI Kit accelerates integration without requiring partners to use identical branding or layouts.

The **partner-facing Open API is distinct from the restricted MM quote-submission interface**. Integrator access never grants permission to become an MM, override the Risk Engine, bypass fees, or change contract-defined settlement obligations. A common network of available MM/Vault liquidity serves different integration channels; no partner-specific options engine or dedicated liquidity pool is required.

## 9. Option Lifecycle

1. The user or authorized strategy selects the exact token, option type, size, strike, expiry, and applicable constraints.
2. The RFQ Router requests quotes from eligible approved MMs and the additional Vault.
3. Quotes are checked for executability and compared on net financial outcome, including the Trading Fee and Opening Base Fee.
4. The user approves the selected terms and signs the Soroban transaction.
5. In one atomic opening, the seller's full collateral is locked, the buyer's premium is paid, fees are collected, and the position is created.
6. The position remains open until expiry; early buyback/closing is not supported.
7. The expiry price is established from the designated oracle process. If ITM, the buyer is expected to deliver the counter-asset within a finite settlement window.
8. The contract atomically exchanges the assets upon valid delivery or, upon a buyer default and expiry of the permitted window, terminates the position and releases the seller's collateral according to predetermined rules.
9. The resulting state, asset movements, and fees are reflected in the Web App and partner APIs.

## 10. Physical Settlement and Counterparty Boundaries

All covered options use **physical settlement**, rather than a mere USDC cash transfer equal to the difference between market price and strike. Exercise is assessed at expiry (European-style exercise). For a Call, the relevant underlying token and USDC are exchanged at the strike. For a Put, USDC and the underlying token are exchanged at the strike.

The **seller fully collateralizes the obligation at opening**. The buyer pays the premium at opening but does not have to lock the entire eventual settlement asset during the option term. When an option is ITM at expiry, the buyer must provide the required counter-asset and authorization in a finite settlement window.

**The protocol does not maintain a settlement guarantee reserve and neither the protocol treasury nor the Vault covers an MM's missing settlement assets.** Consequently, successful physical exchange at strike is not guaranteed if an option buyer defaults or is operationally unable to deliver. Seller collateral must not be transferred without the contracted counter-delivery; after the deadline the remaining collateral is safely released in accordance with contract rules. The seller keeps the original premium, subject to the agreed fees already charged.

Settlement mechanics must provide a terminal state and must not leave funds locked indefinitely if the buyer does not participate. Users and partners must be shown these boundaries clearly. A paused service or disconnected MM does not erase existing positions.

## 11. Market Data and Pricing

RFQ prices are provided by eligible liquidity sources. The Vault uses **market-referenced, dynamically risk-adjusted pricing**: market inputs and volatility references, baseline option valuation (including Black-76 where appropriate), inventory, Delta/Vega sensitivities, exposures, and available risk capital. The Vault can widen prices, reduce size, or decline to quote.

Alongside its primary role in Automated Wheel, the **Prediction Engine** can inform the Vault's decisions about holding or selling free inventory and directional risk scenarios. This engine is advisory: it cannot override the hard limits or safety rules of the Risk Engine. There is no promised yield derived from predictive output.

Quotes are time-limited and bound to the exact instrument and transaction conditions to reduce stale-price and replay risk.

## 12. Vault Capital and Risk Management

### 12.1 Collateral and inventory

The Vault must track free assets separately from collateral already committed to open options, prospective expiry settlement obligations, LP withdrawal requests, and other liabilities. Encumbered collateral cannot be reused to fund withdrawals or an unrelated position.

### 12.2 Settlement-driven inventory

The Vault does not blindly rebalance to a fixed BTC/USDC or ETH/USDC split after settlement. It evaluates the market value, liquidity, risk limits, and expected cost of selling assets received through exercise. It may retain inventory and temporarily stop or reduce quotes instead of taking an unfavorable swap. Hard risk constraints may nevertheless require reducing inventory even at a loss.

### 12.3 Hard and dynamic risk limits

Vault risk control combines **non-negotiable absolute limits** with **dynamic limits** adapting to market volatility, portfolio Greeks, available capital, asset concentration, and aggregate obligations. The Prediction Engine cannot exempt a position from these rules.

### 12.4 Funding and LP participation

The Vault initially uses team and/or investor capital. The system is designed to admit external LP capital after operational validation, under defined share accounting, NAV valuation, risk disclosure, and withdrawal rules. Vault capital is not an implicit insurance fund for defaulting external MMs.

### 12.5 NAV and liquidity provider withdrawals

LP participation is represented by shares valued using NAV per share. NAV incorporates assets, appropriately valued open option positions, liabilities, and fees. **Performance Fee is assessed weekly at 15% above the applicable High-Water Mark**, without treating net capital contributions as profit.

Withdrawals are hybrid: a request can be fulfilled immediately from genuinely free, unencumbered capital, while any remainder joins a transparent queue and is processed as obligations settle and liquidity becomes available. No guaranteed instant withdrawal is implied for capital supporting open positions.

## 13. Oracle and Expiry Integrity

**Chainlink is the primary intended price source** for BTC/USD and ETH/USD, with an independent backup and settlement safeguards. The oracle layer validates data freshness, source consistency, and acceptable deviation before determining the expiry reference price.

If the primary data is unavailable or invalid, backup data may be used under objective published rules. If neither source supports a reliable expiry determination, settlement is postponed under a predefined recovery procedure. A delay cannot become an opportunity for an operator to choose a convenient later price. Once committed as the valid expiry price on Soroban, the result cannot be arbitrarily changed by an administrator.

The specific timestamp-alignment, historical reference, tolerance, and fallback parameters must be established as enforceable technical terms before operating real positions.

## 14. Automation and User Authorization

Strategy Engine actions are limited to permissions explicitly given by the user or specified through Strategy Vault participation. Strategy parameters include supported tokens, allocated capital, maximum position size, permitted strikes and expiries, and minimum acceptable net premium.

Automated operation is not a blank authorization to trade. An unavailable executable quote or breached risk limit causes the strategy to wait rather than open an unfavorable position. Stopping a strategy prevents new openings without undoing current contractual obligations.

## 15. Supported Assets and Bridging

The platform supports options on **BTC/USDC and ETH/USDC** using individually supported BTC/ETH representations on Stellar. Users choose the **specific available asset** when opening an option. This may be the protocol-issued sWBTC or sWETH or a wrapped BTC/ETH token provided by another issuer or bridge.

Different wrapped representations are presented as distinct assets. An option's collateral and physical settlement use the exact token selected at opening. **The platform does not automatically substitute or exchange different wrappers as part of option execution.** Approved MMs and the Vault can choose which of the supported tokens they quote. Token acceptance is subject to reasonable checks on availability, redeemability, custody/bridge characteristics, and delivery practicality.

### 15.1 Protocol-issued assets via LayerZero/OFT

For sWBTC and sWETH, the intended asset flow is **Lock → Mint / Burn → Unlock**. An eligible underlying wrapped token is locked on an originating chain; a validated LayerZero message authorizes the corresponding token representation on Stellar. A reverse transfer burns the Stellar representation and authorizes unlocking the original token.

There is **no discretionary administrator mint or asset-seizure power**. Minting is permitted only as the consequence of valid, authorized, single-use bridge messages matching locked underlying assets. Peer/endpoint authentication, message replay protection, DVN configuration, and custody risks remain essential even if the core contracts cannot be upgraded.

Other providers' tokens remain separate supported choices rather than being forced through the protocol's bridge.

### 15.2 Market-maker delivery

An MM may hedge through any external market or chain. Nevertheless, it must be able to hold or obtain the exact Stellar token or USDC needed for physical delivery within the settlement window. The platform should present the relevant token identity and counterparty/bridge risks transparently.

## 16. Security and Operational Resilience

Core invariants include complete seller collateralization, atomic transfer of payment and underlying assets on successful settlement, mandatory contract-level fees, verified quote authorization, stale quote and replay protection, no arbitrary administrator withdrawal of user collateral, and no unilateral substitution of settlement assets.

Critical configuration privileges should be explicitly scoped and governed. Operational monitoring covers expired positions, delivery windows, oracle status, bridge availability, liquidity inventory, pending withdrawals, and settlement incidents. Emergency controls can restrict new operations without granting arbitrary ownership over previously escrowed user funds.

Any claim of guaranteed strike settlement, guaranteed yield, bridge convertibility, or immediate LP withdrawal must be avoided unless backed by the actual enforceable economic model; the defined model does not offer these guarantees.

## 17. Product Advantages and Quality Criteria

- User-centric option scenarios instead of a professional-only trading terminal.
- Permissioned professional MM liquidity augmented by an internal Vault.
- Best Execution based on executable quotes and best net user price, without routing favoritism.
- Fully collateralized option sellers and transparent physical settlement conditions.
- Automated Wheel using a controlled Strategy Vault.
- SDK, Open API, and optional UI Kit for third-party distribution through common liquidity and settlement infrastructure.
- Support for multiple individually selected wrapped representations of BTC and ETH.
- Contract-enforced recurring protocol monetization from trading activity across all integration channels.

## 18. Product and Operational Metrics

Useful measures include executable quote coverage by token and expiry, net user pricing, quote rejection reasons, successful openings, premiums and fee revenue, effective MM participation, physical settlement completion, failed-to-deliver incidents, strategy retention, Vault returns and drawdowns, LP capital and withdrawal queue health, and integration-driven transaction volume. Risk outcomes must be reported alongside volumes or premium yields.

## 19. Conditions for Operational Readiness

The platform must have validated contracts, adequate funding for its internal Vault, approved MM participation, operational support for the selected BTC/ETH tokens, viable token delivery and redemption, reliable oracle feeds and failure handling, enforceable transaction fees, documented user risk disclosures, and tested settlement outcomes—including MM non-delivery.

Public partner interfaces should faithfully expose the same trade terms and settlement states as the first-party Web App. Regulatory and contractual roles for the platform, Vault operator, market makers, and integration partners require jurisdiction-appropriate review.

## Conclusion

Stellar Options is conceived as a shared options infrastructure layer for consumer products and external Web3 integrations. Its main design principles are **permissioned but competing liquidity sources, a supplementary internal Vault, executable-quote routing, transparent mandatory fees, physically settled options on the exact selected asset, and reusable SDK/API/UI capabilities**.

The platform does not promise fixed returns, automatic replacement of a defaulting MM, or interchangeability between different wrapped tokens. Its product and architecture remain extensible without defining a particular MVP feature cutoff in this specification.
