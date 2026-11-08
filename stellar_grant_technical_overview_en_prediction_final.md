# Stellar Options — Technical Overview for the Stellar Community Fund

**Stellar / Soroban · Product & Technical Overview · November 2026**

---

## 1. Executive Summary

**Stellar Options** is building shared infrastructure for fully collateralized options and automated strategies on Stellar. BTC and ETH token holders will be able to earn option premiums by choosing a price at which they are willing to buy or sell, or buy put options to protect against a price decline. The same capabilities are designed for third-party wallets and DeFi applications through an **Open API, SDK, and a ready-to-use UI Kit**.

A core product differentiator is the **Prediction Engine embedded in the Automated Wheel strategy**. Rather than simply repeating covered calls and cash-secured puts, the strategy uses forward-looking market scenarios to adapt strike selection, expiration, and the timing of its next trade, with the aim of achieving more consistent option-premium income across changing market conditions. The Prediction Engine guides strategy decisions directly; it is not merely an analytics tool for the protocol Vault.

The broader proposition is **one options infrastructure layer for multiple applications**. Our web app, external wallets, and DeFi applications share the same options engine, liquidity sources, and settlement contracts. Vetted external market makers provide the primary liquidity, complemented by quotes from a protocol-operated Vault. The RFQ router checks which quotes can be filled and selects the best price after fees.

Soroban smart contracts handle position creation, collateral, trading fees, and physical settlement. When an option is exercised and settled, the parties exchange the selected BTC or ETH token for USDC at the strike price, rather than settling only the difference in cash.

## 2. The Problem We Address

Options give BTC and ETH holders more ways to manage their assets. Users can earn premiums by selling options, set target prices for potential purchases or sales, and use options to protect against market downturns. Traditional options venues require users to understand contract types, strike prices, expirations, pricing, and settlement mechanics. For wallets and DeFi products, adding options typically means sourcing market makers, building settlement infrastructure, and managing pricing and risk.

Stellar Options brings these capabilities together:

- **For users:** straightforward flows that show the premium, collateral, fees, expiration, and possible outcomes before a trade is confirmed.
- **For developers:** reusable APIs, SDKs, RFQ liquidity, and Soroban contracts so partner apps can offer options without building the underlying infrastructure.

Users selling covered calls or cash-secured puts choose a strike and expiration and receive the premium when the trade opens. The Protect flow lets users buy a put option, giving them the right to sell the chosen asset at the strike price upon exercise.

## 3. Why Stellar: Value for the Ecosystem

Stellar provides the **settlement layer** for USDC, tokenized assets, collateral, and options positions. Soroban smart contracts lock collateral, track positions, collect fees, and exchange assets atomically upon successful exercise. Quote discovery, market data, risk checks, and strategy automation run off-chain, leaving settlement and asset transfers to Soroban.

Potential contributions to the ecosystem include:

1. **A new DeFi building block:** fully collateralized options on supported BTC and ETH tokens, settled on Stellar.
2. **More ways to use USDC:** collateral for cash-secured puts and settlement for physically delivered options.
3. **Shared infrastructure for builders:** wallets and DeFi apps can integrate options without deploying a separate options engine or sourcing liquidity for each application.
4. **Broader utility for tokenized assets:** supported BTC and ETH tokens can be used as collateral and settlement assets.
5. **More on-chain activity:** opening and settling options create transactions and asset flows on Stellar.

We will measure this impact through active wallets, completed trades, collateral volume, and usage from partner integrations.

## 4. Products and User Experiences

| Experience | User objective | Underlying mechanism |
|---|---|---|
| **Sell Higher / Earn** | Choose a price at which to sell, lock the asset, and receive a premium. If the option is exercised, sell at the chosen price | Fully collateralized Covered Call |
| **Buy Lower / Earn** | Choose a purchase price, lock the required USDC, and receive a premium. If exercised, buy at the chosen price | Fully collateralized Cash-Secured Put |
| **Protect** | Buy a put option that provides the right to sell the asset at the chosen strike price at expiry | Long Put |
| **Automated Wheel** | Use market forecasts to adapt strike selection, expiration and when to open the next position, while automating cash-secured puts and covered calls | Strategy Vault and Prediction Engine |

Before opening a trade, users see the asset, amount, strike, expiration, premium, collateral, fees, and possible outcomes. Options remain open until expiration; early close-out is not supported. The Wheel strategy moves between assets only after physical settlement has actually completed.

### Prediction Engine: an adaptive Wheel strategy

A conventional Wheel strategy alternates between cash-secured puts and covered calls. Simply repeating the cycle does not account for changing trends, volatility, or whether the available premium adequately compensates for the risk taken.

**Our Prediction Engine is part of the Wheel’s decision-making process.** It evaluates probabilistic price and volatility scenarios to help the Strategy Engine choose strikes and expirations, assess available premiums, and decide whether to open another position or wait. After a successful physical settlement, it also helps assess when and how to continue the strategy with the asset actually received.

This approach is designed to make the Wheel more responsive to market conditions and support more consistent premium generation, rather than forcing a new trade on every cycle. The strategy operates within user-defined settings and the Risk Engine’s hard limits. Predictions inform decisions but do not override risk controls or promise any particular return.

Partner applications can access the strategy through the same shared APIs and integration tools used by the first-party app.

## 5. High-Level Architecture

| Layer | Components | Responsibility |
|---|---|---|
| **User-facing apps** | Web App, partner applications, UI Kit | Strategy selection, trade terms, transaction confirmation |
| **Integration** | Open API, SDK, events and notifications | Access to quotes, positions, and trade status |
| **Liquidity** | Vetted MMs, protocol-operated Vault | Available quotes from multiple liquidity sources |
| **Strategy and risk** | RFQ Router, Strategy Engine, Prediction Engine, Risk Engine | Quote selection, adaptive Wheel decisions, strategy execution, and risk controls |
| **Market data** | Pricing references and oracles | Market data and verified expiry prices |
| **On-chain** | Soroban Options Contracts | Collateral, positions, fees, and physical settlement |
| **Assets** | USDC and supported BTC/ETH tokens | Assets used as collateral and for settlement |

**Separation of responsibilities:** off-chain services discover quotes and prepare transactions. Soroban contracts enforce position state, collateral custody, fees, and asset transfers. Partner API access does not grant control over trading contracts.

### Representative trade lifecycle

1. The user selects a supported BTC or ETH token, strategy, trade size, strike, and expiration.
2. The RFQ router requests quotes from vetted market makers and, when available, the Vault.
3. The router checks whether quotes are current and executable, then selects the best price **after all applicable fees**.
4. The user confirms the trade. A single Soroban transaction opens the position, locks the seller's collateral, transfers the premium, and collects fees.
5. At expiration, an oracle price determines whether the option is eligible for exercise. If the exercise conditions are met and the buyer supplies the required asset, the contract exchanges both assets atomically. Otherwise, the position ends under predefined rules and the seller’s collateral is released.

## 6. Liquidity and Quote Selection

**The platform works with vetted market makers** who provide quotes for supported options. Each market maker is responsible for its quotes and for delivering the required settlement asset on Stellar. Market makers can hedge BTC and ETH exposure on external exchanges or other networks using spot, futures, or options. They do not need a local derivatives market on Stellar to hedge those positions.

The **protocol-operated Vault** adds liquidity alongside external market makers and receives no preferential routing. It may quote, limit order size, or decline a request based on available capital and risk limits. The Vault will initially be funded by the team and investors, with external liquidity providers (LPs) able to participate later.

**Best Execution** checks whether a quote can be filled before selecting the best price after fees. If there is no eligible quote, no trade is opened. The same routing rules apply to our own app and partner integrations.

## 7. Open API, SDK, and UI Kit

**Developer tools are a core part of the platform.** Our web app and partner applications use the same underlying trading capabilities.

The **Open API** provides supported assets and options, RFQ quotes, positions, fees, settlement data, and strategy status. The **SDK** simplifies wallet integration, user approvals, Soroban transaction preparation, and status tracking. The **UI Kit** offers ready-to-use components for choosing assets, configuring covered calls and puts, and viewing quotes and positions. Partners can adopt selected components and style them to match their applications.

All integrations share the same RFQ and Soroban infrastructure; partners do not need to build separate options engines or source dedicated liquidity. The **partner API is separate from the private interface used by vetted market makers to submit quotes**.

### Integration economics

Each successful trade opening incurs a **Trading Fee of 1% of the premium** plus a **fixed Opening Base Fee in USDC**. The Opening Base Fee is configurable; its amount has not yet been set. Both fees are collected in the Soroban transaction that opens the option, whether initiated through our app, a partner integration, or a direct contract call.

Partners may receive an agreed **share of the Trading Fee** without adding to the user’s fee. The protocol always retains part of the Trading Fee and receives the full Opening Base Fee. Partner revenue shares are negotiated separately.

## 8. Supported Assets and Cross-Chain Transfers

Users can choose among **supported BTC and ETH tokens**, including sWBTC/sWETH and tokens issued by other providers or bridges. Users select the specific token when opening an option. Depending on the option type, collateral is posted in that token or in USDC. Physical settlement uses the specific token selected for the trade. Different wrapped versions of BTC or ETH are not automatically swapped or substituted.

For protocol-issued sWBTC/sWETH, the design uses **Lock → Mint / Burn → Unlock** through LayerZero/OFT. Tokens can be minted only in response to a verified bridge message confirming that the underlying asset has been locked, with protection against replay. Administrators cannot mint unbacked tokens or arbitrarily seize user assets.

Before listing a token, the team will assess its bridge or issuer, redemption process, and availability for settlement. Third-party wrapped tokens do not have to use our LayerZero bridge.

## 9. Physical Settlement and Security

Option sellers post **full collateral**. The buyer pays the premium upfront without locking the full amount needed for settlement throughout the option’s term. If the option is exercisable at expiry, the buyer provides the required token or USDC within the settlement window. Soroban then exchanges the assets atomically at the strike price.

If the buyer does not deliver the settlement asset within that window, the option expires under predefined contract rules and the seller’s collateral is released.

Key security measures include:

- Authenticated, time-limited RFQ quotes that cannot be reused.
- Trading and Opening Base Fees collected in the same on-chain transaction that opens the option, including direct contract calls.
- Chainlink as the primary BTC/USD and ETH/USD oracle, with an independent backup and checks for stale or inconsistent prices.
- Predefined procedures for oracle outages, preventing operators from choosing a more favorable expiry price.
- Fixed and dynamic Vault risk limits, with locked collateral kept separate from available capital.
- Limited administrative permissions and emergency controls that cannot be used to seize customer assets.

## 10. Revenue Model and Launch Requirements

The protocol earns revenue through Trading Fees and Opening Base Fees. Once outside liquidity providers (LPs) join the Vault, a Performance Fee will also apply. Partner integrations extend distribution without fragmenting liquidity.

Launching the platform requires active market makers for supported assets, reliable oracle data, sufficient Vault capital, and tested physical settlement.

## 11. Measuring Results and Ecosystem Impact

The project’s progress can be measured through:

| Area | Evidence of progress |
|---|---|
| On-chain capability | Fully collateralized option trades and physical settlement on Soroban |
| Safety | Tests of settlement failures, oracle outages, rejected quotes, and collateral release |
| Liquidity | Active market makers, RFQs with executable quotes, and prices after fees |
| Network usage | Active wallets, trades, locked collateral, and network transactions |
| Developer adoption | Integrated apps, API/SDK trade volume, documentation, and sample integrations |
| User clarity | Successful settlements, clear trade terms, and transparent fees |

Specific targets, timelines, and budget will be included in a separate grant proposal.

## 12. What Is Needed to Launch

- Build and test Soroban contracts and integrate supported BTC/ETH tokens.
- Onboard market makers who can quote options and provide settlement assets.
- Review LayerZero settings and the security of supported third-party tokens.
- Ensure oracle prices are reliable at option expiry.
- Complete smart contract audits, operational testing, and legal review of each participant’s role.

## Conclusion

**Stellar Options brings shared options infrastructure to the Stellar ecosystem.** Soroban handles collateral and settlement, the RFQ router combines market-maker quotes with additional Vault liquidity, and the Open API, SDK, and UI Kit let wallets and DeFi apps offer options within their existing products. Rather than building another isolated trading venue, we aim to give Stellar developers a common foundation for options and automated strategies.

---
