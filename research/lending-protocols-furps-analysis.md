# Lending/Borrowing Protocol FURPS+ Analysis

Comparative analysis of the 10 largest lending/borrowing protocols across EVM and Solana ecosystems.

---

## Table of Contents

### EVM Protocols
1. [Aave](#aave) — ~$40B+ TVL, 14+ chains
2. [Morpho](#morpho) — ~$6.3B TVL, modular/permissionless
3. [Spark](#spark) — ~$2.5B TVL, Sky/MakerDAO ecosystem
4. [Compound](#compound) — ~$2.1B TVL, OG lending protocol
5. [JustLend](#justlend) — ~$5.9B TVL, largest on Tron

### Solana Protocols
6. [Kamino Lend](#kamino-lend) — ~$2-3.5B TVL, dominant Solana lender
7. [Jupiter Lend](#jupiter-lend) — ~$1.65B TVL, DEX-integrated
8. [Marginfi](#marginfi) — significant TVL, advanced risk engine
9. [Save.Finance (Solend)](#savefinance-solend) — ~$400M+ TVL, oldest Solana lender
10. [Port Finance](#port-finance) — smaller TVL, fixed+variable rate

---

## EVM Protocols

---

<a id="aave"></a>
## 1. Aave

> ~$40B+ TVL | 14+ chains | 60%+ lending market share | $1T+ cumulative loans

### Functionality

1. **[CRITICAL]** Over-collateralized lending and borrowing — Users supply assets to liquidity pools and receive interest-bearing aTokens (e.g., aUSDC, aWETH) whose balance auto-increases over time. Borrowers post collateral exceeding the borrowed amount and receive variableDebtTokens that track accruing debt. All positions are always over-collateralized, with per-asset LTV ratios (e.g., 80% for major assets) governing maximum borrow power.

2. **[CRITICAL]** Algorithmic interest rate model — A dual-slope kinked curve adjusts rates based on pool utilization. Below the optimal utilization point (typically 80%), rates rise gently along slope1 (e.g., 4% APR); above it, rates rise steeply along slope2 (e.g., 75% APR) to incentivize repayment and liquidity return. Parameters (base rate, slopes, optimal point) are configurable per asset via governance.

3. **[CRITICAL]** Liquidation engine — When a borrower's Health Factor (Total Collateral x Weighted Liquidation Threshold / Total Debt) drops below 1, any network participant can permissionlessly liquidate the position. Liquidators repay debt and receive collateral at a discount (liquidation bonus). Up to 50% of debt is liquidatable when HF is between 0.95-1.0; 100% when HF falls below 0.95 or positions are under $2,000. V4 introduces health-targeted liquidations that sell only enough collateral to restore safe ratios.

4. **[CRITICAL]** Flash loans — Uncollateralized loans of any available amount that must be borrowed and repaid (plus a 0.05% fee) within a single block transaction. If repayment fails, the entire transaction reverts atomically. Used for arbitrage, liquidation bootstrapping, collateral swaps, and self-liquidation.

5. **[CRITICAL]** Efficiency Mode (eMode) — Correlated assets (e.g., USD stablecoins, ETH derivatives) are grouped into categories. When a user's collateral and borrowed assets belong to the same eMode category, they receive significantly higher LTV ratios, enabling capital-efficient strategies like leveraged yield farming on staked ETH.

6. **[CRITICAL]** GHO stablecoin — A decentralized, over-collateralized stablecoin native to the Aave Protocol. GHO is minted when users borrow against their collateral and burned on repayment (elastic supply). 100% of GHO borrow interest flows to the Aave DAO treasury. Features include: stkAAVE holders receive discount rates, a Peg Stability Module (GSM), and a Facilitator model where governance-approved contracts can mint/burn GHO up to defined caps.

7. **[CRITICAL]** Multi-chain deployment — Deployed across 14+ chains including Ethereum (81% of TVL), Polygon, Arbitrum, Optimism, Avalanche, Base, BNB Chain, Linea, Scroll, ZKsync, Gnosis, Sonic, Celo, and Aptos (first non-EVM deployment). V4 introduces a Cross-Chain Liquidity Layer (CCLL) to aggregate liquidity across networks.

8. **[CRITICAL]** Governance system — AAVE and stkAAVE holders vote on AIPs or delegate voting power. Three-stage process: TEMP CHECK, ARFC, and AIP. Short executor proposals have a 1-day timelock; long executor proposals have a 7-day timelock.

9. **[STANDARD]** Isolation Mode — Newly listed or volatile assets can be quarantined so they can only serve as collateral for borrowing a restricted set of assets up to a governance-defined debt ceiling.

10. **[STANDARD]** Siloed Mode — Assets with potentially manipulatable oracles are restricted so borrowers can only have a single borrow position when using that collateral.

11. **[STANDARD]** Credit delegation — Suppliers can delegate their unused borrowing power to other addresses, enabling uncollateralized lending to trusted counterparties.

12. **[STANDARD]** Supply and borrow caps — Per-asset caps set by governance that limit the maximum amount of each asset that can be supplied to or borrowed from the protocol.

13. **[STANDARD]** ERC-4626 tokenized vaults (V4) — Wraps positions in ERC-4626 standard vaults, enabling simplified accounting and composability.

14. **[STANDARD]** Reinvestment Module (V4) — Automatically deploys idle liquidity into external yield-generating strategies through automated allocation.

### Usability

1. **[CRITICAL]** aToken composability — aTokens are standard ERC-20 tokens with auto-rebasing balances. Users can hold, transfer, or use aTokens in other DeFi protocols while continuing to earn interest.

2. **[CRITICAL]** Single pool entry point (Pool.sol) — All user-facing operations are accessed through a single Pool contract per deployment, simplifying integration.

3. **[STANDARD]** L2 calldata optimization — Specialized L2Pool contracts compress calldata using 16-bit asset IDs instead of 160-bit addresses.

4. **[STANDARD]** Health Factor visibility — A single, intuitive metric summarizing position risk.

5. **[STANDARD]** Permit-based approvals — EIP-2612 support for single-transaction approve-and-interact.

6. **[STANDARD]** Governance delegation — Token holders can delegate voting power without transferring tokens.

### Reliability

1. **[CRITICAL]** Umbrella safety system — Users stake aTokens or GHO to backstop protocol solvency. Automated on-chain slashing triggers when bad debt exceeds a threshold — no governance vote required.

2. **[CRITICAL]** Oracle infrastructure — Chainlink decentralized price feeds as primary source. Specialized CAPO (Correlated Assets Price Oracle) for wrapped tokens and LSTs.

3. **[CRITICAL]** Multi-layered audit process — Zero high-severity findings across 345 days of security review for V4. Audits by ChainSecurity, Trail of Bits, Blackthorn, Certora (formal verification), and a 900-participant public contest. Bug bounty up to $1M via Immunefi.

4. **[CRITICAL]** Over-collateralization invariant — All borrow positions must maintain collateral value exceeding debt value.

5. **[STANDARD]** Legacy Safety Module (stkAAVE) — 20% maximum slashing risk, being phased out in favor of Umbrella.

6. **[STANDARD]** Atomic flash loan safety — Reverts entirely if not repaid within the same transaction.

7. **[STANDARD]** Governance timelocks — 1-day for standard changes, 7-day for core governance changes.

### Performance

1. **[CRITICAL]** Capital efficiency via eMode — Higher LTV ratios for correlated asset categories.

2. **[CRITICAL]** Utilization-responsive rate curves — Self-regulating interest rates maintain liquidity.

3. **[CRITICAL]** V4 Hub-and-Spoke architecture — Eliminates liquidity fragmentation by consolidating all assets into a unified Liquidity Hub per network.

4. **[STANDARD]** Gas optimization — V3.3 reduced gas by 59,732 units via storage pattern optimization.

5. **[STANDARD]** Cross-Chain Liquidity Layer (V4) — Aggregates liquidity from multiple networks.

6. **[STANDARD]** Reinvestment of idle capital (V4) — Deploys unused liquidity into external yield strategies.

### Supportability

1. **[CRITICAL]** Modular contract architecture — Features can be upgraded independently via specialized logic contracts.

2. **[CRITICAL]** Governance-controlled parameterization — All risk parameters adjustable via governance without contract upgrades. Granular role-based access (RISK_ADMIN, ASSET_LISTING_ADMIN, POOL_ADMIN).

3. **[CRITICAL]** Open-source codebase — All protocol contracts publicly available.

4. **[STANDARD]** Facilitator model for GHO — Extensible mint/burn authority via governance-approved Facilitators.

5. **[STANDARD]** Professional risk service providers — Gauntlet, Chaos Labs engaged as service providers.

6. **[STANDARD]** Comprehensive developer documentation — Smart contract references, integration guides, SDKs.

7. **[STANDARD]** Versioned upgrade path — Clear migration from V2 to V3 to V4 with coexistence during transitions.

---

<a id="morpho"></a>
## 2. Morpho

> ~$6.3B TVL | $1B+ active loans | Immutable base layer | ~650 lines of Solidity

### Functionality

1. **[CRITICAL]** Isolated, permissionless market creation — Anyone can deploy a lending market by specifying five immutable parameters: one loan asset, one collateral asset, one oracle, one LLTV ratio, and one IRM. Markets are fully isolated.

2. **[CRITICAL]** Two-layer architecture (Base Layer + Vault Layer) — Immutable accounting base layer (Morpho Blue) and mutable risk-management layer (Morpho Vaults). Base layer handles only lending/borrowing accounting; all strategy, curation, and UX abstraction live in the vault layer.

3. **[CRITICAL]** Curated vaults with role-based access control — Owner manages permissions, Curator sets risk parameters, Allocator manages portfolio allocation, Sentinel can reactively de-risk.

4. **[CRITICAL]** AdaptiveCurveIRM — Autonomous interest rate model that targets 90% utilization through a traditional curve mechanism and an adaptive mechanism that shifts the curve over time. Immutable per-market, requires no governance tuning.

5. **[CRITICAL]** Singleton contract design — All markets on a given chain live in a single smart contract. Reduces gas costs, enables free flash loans across all market liquidity.

6. **[CRITICAL]** External liquidation with incentive factor — When a position becomes unhealthy, external actors can liquidate up to 100% of the debt and receive collateral plus a liquidation incentive.

7. **[CRITICAL]** Oracle-agnostic price feeds — Market creators select the oracle at creation time. Supports Chainlink, Redstone, API3, Pyth, and Chronicle.

8. **[STANDARD]** Peer-to-peer matching (legacy Morpho Optimizers) — Original product that matched lenders/borrowers on Aave/Compound at mid-point rates. Replaced by Morpho Blue.

9. **[STANDARD]** Free flash loans — No fee, access to combined liquidity of all markets on the chain.

10. **[STANDARD]** Callback system — Supply, supplyCollateral, repay, and liquidate functions support callbacks before token transfer.

11. **[STANDARD]** Account management and authorization delegation — EIP-712 signed authorization delegation.

12. **[STANDARD]** Bundler3 — Atomic multi-step transaction execution using transient storage.

13. **[STANDARD]** Vault V2 cap system — Absolute and relative caps based on shared risk factors.

14. **[STANDARD]** Governance-controlled fee switch — Up to 25% fee on interest paid by borrowers.

15. **[STANDARD]** Vault gates — Access control for permissioned/institutional vaults.

### Usability

1. **[CRITICAL]** Vault abstraction for passive lenders — Users deposit an asset into a curated vault and earn yield without understanding individual markets.

2. **[CRITICAL]** Multichain availability — Deployed on Ethereum, Base (largest protocol by TVL), Polygon, Arbitrum, Optimism, Scroll, and 12+ EVM chains.

3. **[STANDARD]** Timelocked curator actions — Configurable 0-3 week timelocks give depositors exit windows.

4. **[STANDARD]** Sentinel emergency de-risking — Fast-response safety role for deallocating funds.

5. **[STANDARD]** EIP-712 signature-based authorization — Gasless authorization flows.

6. **[STANDARD]** Five-parameter market simplicity — Fully predictable and auditable from creation.

### Reliability

1. **[CRITICAL]** Immutable base layer contracts — Not upgradeable. Lindy effect strengthens over time.

2. **[CRITICAL]** Formal verification with Certora — Verified properties include market independence and reentrancy safety.

3. **[CRITICAL]** Minimal attack surface (~650 lines of Solidity) — Auditable by a single engineer.

4. **[CRITICAL]** Multiple independent security audits — Spearbit, OpenZeppelin, Trail of Bits, ChainSecurity.

5. **[CRITICAL]** Bug bounty program — Up to $2.5M via Immunefi.

6. **[CRITICAL]** Market isolation by design — Formally verified at the contract level.

7. **[STANDARD]** Invariant testing with Foundry — Fuzzing for edge cases.

8. **[STANDARD]** Governance minimization — Limited to treasury, fee switch, and LLTV/IRM whitelisting.

### Performance

1. **[CRITICAL]** Gas efficiency from singleton architecture — All markets share one contract, eliminating per-market deployment costs.

2. **[CRITICAL]** $6.8B TVL with $1B+ active loans — Validates large-scale operation.

3. **[STANDARD]** Layer 2 cost reduction — Base deployment has $320M+ TVL.

4. **[STANDARD]** Adaptive rate convergence — Speed proportional to distance from target utilization.

5. **[STANDARD]** Bundler3 transaction batching — Reduces gas for multi-step operations.

### Supportability

1. **[CRITICAL]** Modular, layered extensibility — Frozen base layer with permissionlessly extensible vault/application layer.

2. **[CRITICAL]** Permissionless integration surface — Create markets, build vaults, write liquidation bots without permission.

3. **[STANDARD]** Vault V2 adapter system — Protocol-agnostic vault allocation.

4. **[STANDARD]** Open-source codebase (MIT-compatible).

5. **[STANDARD]** Comprehensive documentation and tooling.

6. **[STANDARD]** Governance forum and DAO structure.

---

<a id="spark"></a>
## 3. Spark

> ~$2.5B TVL | Aave V3 fork | Sky/MakerDAO ecosystem | RWA-backed yields

### Functionality

1. **[CRITICAL]** SparkLend over-collateralized lending/borrowing — Core lending protocol (Aave v3 fork). Variable interest rates for most assets; USDS borrowing rates are governance-defined and sourced from Sky Protocol liquidity, acting as quasi-fixed rates.

2. **[CRITICAL]** Sky Savings Rate (SSR) / Dai Savings Rate (DSR) integration — Users deposit USDS, DAI, or USDC and receive non-rebasing receipt tokens (sUSDS, sDAI, sUSDC) that appreciate over time. Yield funded by Sky Protocol revenue.

3. **[CRITICAL]** Spark Liquidity Layer (SLL) — Capital routing system that mints, bridges, deploys, and manages stablecoin liquidity across networks. Enables governance-defined borrowing rates from Sky's $6.5B+ reserves.

4. **[CRITICAL]** Peg Stability Module (PSM) — Zero-slippage, zero-fee swaps between USDS, sUSDS, and USDC. Deployed cross-chain.

5. **[CRITICAL]** RWA-backed yield subsidy model — US Treasury bills and other RWAs generate off-chain revenue that supplements on-chain borrowing fees.

6. **[STANDARD]** Efficiency Mode (eMode) — Higher LTV for correlated asset pairs.

7. **[STANDARD]** Isolation Mode — Debt ceiling for higher-risk assets.

8. **[STANDARD]** Siloed Borrowing — Restricts borrowing for assets with manipulatable oracles.

9. **[STANDARD]** Flash Loans — 0.09% fee, single-reserve, atomic execution.

10. **[STANDARD]** Spark Prime (institutional margin lending) — Margin lending for institutions across CEXs, DeFi venues, and custodians.

11. **[STANDARD]** Spark Institutional Lending — Off-chain lending with on-chain liquidity pools (via Anchorage Digital).

12. **[STANDARD]** SPK governance token — 10B supply; 65% for user farming over 10 years.

### Usability

1. **[CRITICAL]** Unified savings on-ramp from any stablecoin — Deposit USDS, DAI, or USDC and earn SSR/DSR immediately. PSM handles conversions at 1:1.

2. **[CRITICAL]** Governance-defined transparent rates — USDS borrow rate is quasi-fixed, set by governance. More predictable than utilization-based rates.

3. **[STANDARD]** Cross-chain PSM availability — Same rates on Base, Arbitrum, Optimism without manual bridging.

4. **[STANDARD]** Health factor and liquidation transparency.

5. **[STANDARD]** No withdrawal fees.

6. **[STANDARD]** Receipt tokens as composable DeFi primitives — sUSDS, sDAI, sUSDC usable across DeFi.

### Reliability

1. **[CRITICAL]** Aggor oracle aggregation — Multi-source (Chronicle, Chainlink, Uniswap v3 TWAP fallback).

2. **[CRITICAL]** Kill Switch module — Disables borrowing when LSTs depeg beyond threshold.

3. **[CRITICAL]** Aave v3 battle-tested codebase — Billions secured over multiple years.

4. **[CRITICAL]** Supply and Borrow Caps — Governance-configurable, delegatable for rapid response.

5. **[STANDARD]** $5M bug bounty via Immunefi.

6. **[STANDARD]** Multi-auditor coverage — ChainSecurity plus Aave v3's audit history.

7. **[STANDARD]** Delegated risk parameter updates.

8. **[STANDARD]** Cross-chain bridging via canonical paths — SkyLink and Circle CCTP.

### Performance

1. **[CRITICAL]** Deep stablecoin liquidity from Sky reserves — $6.5B+ reserves prevent dramatic rate spikes on large operations.

2. **[CRITICAL]** Capital efficiency through eMode.

3. **[STANDARD]** L2 deployment for lower gas costs.

4. **[STANDARD]** Automated liquidity management via SLL.

5. **[STANDARD]** Flash loan single-transaction execution.

### Supportability

1. **[CRITICAL]** Sky governance alignment via Star/SubDAO structure — Backed by Sky's $6.5B+ managed assets.

2. **[STANDARD]** Open-source codebase.

3. **[STANDARD]** SPK token staking for protocol security.

4. **[STANDARD]** Modular architecture (SLL + SparkLend + Savings + PSM).

5. **[STANDARD]** Extensible risk management delegation.

6. **[STANDARD]** Multi-chain expansion roadmap.

---

<a id="compound"></a>
## 4. Compound

> ~$2.1B TVL | Ethereum + L2s | OG DeFi lending protocol (since 2018)

### Functionality

1. **[CRITICAL]** Lending (supply) — Users deposit ERC-20 tokens to earn interest. V2: each asset has its own cToken market. V3: only the base asset (e.g., USDC) earns supply interest; collateral assets do not.

2. **[CRITICAL]** Borrowing — V2 allows borrowing any listed asset; V3 restricts to a single base asset per market deployment, eliminating cross-asset contagion.

3. **[CRITICAL]** cToken exchange-rate model (V2) — Suppliers receive cTokens whose exchange rate increases monotonically. Composable ERC-20s usable across DeFi.

4. **[CRITICAL]** Jump-rate interest rate model — Piecewise-linear curve with a "kink" utilization threshold. V3 has independent supply and borrow rate curves.

5. **[CRITICAL]** Collateral factor / liquidation factor separation (V3) — Buffer zone between borrowing threshold and liquidation threshold.

6. **[CRITICAL]** Liquidation via absorb mechanism (V3) — All-or-nothing: seizes all collateral, protocol assumes debt, then sells collateral via `buyCollateral` at a discount.

7. **[CRITICAL]** Comptroller / risk engine — Validates every user action against risk parameters.

8. **[CRITICAL]** COMP governance token — On-chain governance. 25,000 COMP to propose, 400,000 votes quorum, 3-day voting, 2-day timelock.

9. **[CRITICAL]** Reserve system — Governance-set reserve factor diverts interest into protocol reserves as bad debt backstop.

10. **[STANDARD]** COMP distribution / incentives — Liquidity mining rewards.

11. **[STANDARD]** Operator / manager permissions (V3) — Delegated account management.

12. **[STANDARD]** Supply caps (V3) — Per-collateral limits.

13. **[STANDARD]** ERC-4626 vault wrapper (V3) — Composability with aggregators.

14. **[STANDARD]** Multi-chain deployment (V3) — Independent Comet instances per chain.

15. **[STANDARD]** Autonomous proposals — 100-COMP barrier for community-driven proposals.

### Usability

1. **[CRITICAL]** Single-asset borrowing simplicity (V3) — One borrowable base asset per market eliminates multi-asset pool risk assessment.

2. **[CRITICAL]** Transparent, real-time interest rates — Deterministic functions of utilization.

3. **[STANDARD]** No minimum deposit or borrow amounts.

4. **[STANDARD]** Instant withdrawal (subject to liquidity).

5. **[STANDARD]** Vote delegation UX.

6. **[STANDARD]** No simultaneous lend-and-borrow in the same market (V3).

### Reliability

1. **[CRITICAL]** Battle-tested smart contracts — Live since 2019, no protocol-level exploit of core lending logic.

2. **[CRITICAL]** Oracle architecture with fallback — UniswapAnchoredView cross-references Chainlink with Uniswap V2 TWAP. V3 uses Chainlink with staleness checks.

3. **[CRITICAL]** Risk isolation via single-borrowable-asset model (V3) — Eliminates pooled-risk contagion.

4. **[CRITICAL]** Reserve buffer against insolvency.

5. **[STANDARD]** Timelock on governance actions — 2-day mandatory delay.

6. **[STANDARD]** Conservative collateral factors — 0.65-0.85 depending on volatility.

7. **[STANDARD]** Liquidation incentive alignment.

### Performance

1. **[CRITICAL]** Immutable variable storage for gas optimization (V3) — ~3 gas vs ~2,100 for SLOAD.

2. **[CRITICAL]** Monolithic contract design (V3) — Eliminates inter-contract calls.

3. **[CRITICAL]** Per-second interest accrual (V3) — Consistent behavior on L2s with variable block intervals.

4. **[STANDARD]** Packed asset storage — Bit-packed immutable variables.

5. **[STANDARD]** Lazy interest accrual — Only on user interaction.

6. **[STANDARD]** L2 deployment for low-cost access.

### Supportability

1. **[CRITICAL]** Configurator + Factory upgrade pattern (V3) — New implementation deployed per parameter change, avoiding storage migration.

2. **[CRITICAL]** Transparent upgradeable proxy — User-facing address never changes.

3. **[CRITICAL]** Fully on-chain governance — All changes flow through governance proposals.

4. **[STANDARD]** Modular multi-chain deployment.

5. **[STANDARD]** Rigorous 71-step new-chain onboarding process.

6. **[STANDARD]** Open-source codebase (MIT).

7. **[STANDARD]** Autonomous proposal mechanism.

---

<a id="justlend"></a>
## 5. JustLend

> ~$5.9B TVL | Largest on Tron | Liquid staking (sTRX) + Energy rental

### Functionality

1. **[CRITICAL]** Pooled Lending and Borrowing (SBM) — Core money market on TRON. 19+ TRC-20 token markets. Algorithmic interest rates based on utilization curves.

2. **[CRITICAL]** jToken Receipt Mechanism — cToken-equivalent. Accrue interest via increasing exchange rate. Composable TRC-20 tokens.

3. **[CRITICAL]** Algorithmic Interest Rate Model (Jump Rate Model V2) — Dual-slope model with configurable kink threshold.

4. **[CRITICAL]** Overcollateralization and Liquidation Engine — Per-asset collateral factors. Up to 50% debt liquidatable per transaction. Permissionless with liquidation incentive.

5. **[CRITICAL]** Staked TRX (sTRX) — Liquid staking derivative. Auto-stakes TRX, votes for Super Representatives, claims rewards, rents Energy. ~9% APY. 14-day unbonding.

6. **[CRITICAL]** Energy Rental Market — Rent TRON Energy on-demand per transaction (8% tax). 100,000 Energy costs ~2.65 TRX vs staking ~9,928 TRX.

7. **[STANDARD]** GasFree Smart Wallet — Pay transaction fees in USDT instead of TRX. 750K+ transactions, $15.8B volume.

8. **[CRITICAL]** DAO Governance via JST Token — Lock JST, convert to WJST, vote on JIPs. 200M-vote proposal threshold, 600M-vote quorum, 2-day timelock.

9. **[STANDARD]** JST Buyback and Burn — 1.08B JST burned (~10.96% of supply). Deflationary model tied to protocol usage.

10. **[STANDARD]** Supply Mining / Liquidity Incentives.

11. **[STANDARD]** Multi-Asset Support — 19+ markets with independent risk parameters.

12. **[STANDARD]** sTRX Vault (USDD Minting) — Deposit sTRX to mint USDD stablecoin.

### Usability

1. **[CRITICAL]** One-Click Staking (sTRX) — Abstracts SR voting, reward claiming, and Energy rental.

2. **[CRITICAL]** Low Transaction Costs on TRON — ~3-second blocks, sub-cent fees.

3. **[STANDARD]** GasFree Stablecoin Payments.

4. **[STANDARD]** Unified Dashboard.

5. **[STANDARD]** Wallet Ecosystem Integration — TronLink and others.

6. **[STANDARD]** Instant Withdrawals.

### Reliability

1. **[CRITICAL]** Battle-Tested Smart Contracts — Operational since 2020, no major breach.

2. **[CRITICAL]** Oracle Migration to Chainlink — Replaced WINkLink in May 2025.

3. **[CRITICAL]** Permissionless Liquidation with Incentives — 50%-per-asset-per-transaction cap.

4. **[STANDARD]** Multiple Smart Contract Audits.

5. **[STANDARD]** Reserve Factor as Protocol Buffer.

6. **[STANDARD]** Conservative Collateral Factor Assignment — New assets can start at 0%.

### Performance

1. **[CRITICAL]** TRON Network Throughput — ~3-second blocks, 2,000 TPS. Per-block interest accrual.

2. **[CRITICAL]** $5.95B+ TVL — Deep liquidity pools.

3. **[STANDARD]** Algorithmic Rate Adjustment — Every block.

4. **[STANDARD]** Energy Rental Cost Efficiency — 3,700x more efficient than staking.

5. **[STANDARD]** Composable jToken Architecture.

### Supportability

1. **[CRITICAL]** DAO Governance with On-Chain Upgradeability — All parameters adjustable via JIP proposals.

2. **[STANDARD]** Open-Source Protocol Contracts.

3. **[STANDARD]** Comprehensive Documentation.

4. **[STANDARD]** Modular Interest Rate Models.

5. **[STANDARD]** Oracle Provider Abstraction.

6. **[STANDARD]** JUST Ecosystem Integration.

7. **[STANDARD]** JST Buyback Funded by Protocol Revenue.

---

## Solana Protocols

---

<a id="kamino-lend"></a>
## 6. Kamino Lend

> ~$2.8B+ TVL | $4.4B total deposits | Zero bad debt track record | 18 audits

### Functionality

1. **[CRITICAL]** Market Layer (Permissionless Lending Markets) — V2's core primitive. Anyone can create an isolated lending market with custom risk configurations. Modular architecture avoids cross-market risk contamination.

2. **[CRITICAL]** Vault Layer (Curated Earn Vaults) — Single-asset deposit vaults allocating across multiple V2 markets. Managed by professional curators (Gauntlet, Steakhouse Financial, Re7 Labs, Allez Labs, MEV Capital, Sentora). Conservative/Balanced/Aggressive profiles.

3. **[CRITICAL]** Elevation Mode (eMode) — Correlated asset groups with up to 90% LTV (up to 10x leverage). Per-pairing debt limits via E-Mode Caps.

4. **[CRITICAL]** Multiply (Leveraged Yield) — One-click leveraged positions using flash loans within a single transaction. Supports target-leverage mode with stop-loss/take-profit.

5. **[CRITICAL]** Auction-Based Liquidation (V2) — Dutch-auction mechanism. Starting penalty at 2%, capped at 10%. Competitive bidding drives down borrower penalties.

6. **[CRITICAL]** Multi-Oracle Price System — Pyth, Switchboard, and Chainlink simultaneously. TWAP and EWMA filters reject manipulation.

7. **[CRITICAL]** Automated Deleverage (Auto-Deleverage) — Protocol-level circuit breaker. Lowers caps and triggers automated position unwinding with 72-hour grace period.

8. **[CRITICAL]** Poly-Linear Interest Rate Curves — Up to 11 configurable points (vs standard 3-point kink model).

9. **[STANDARD]** Flash Loans — Available to external integrators.

10. **[STANDARD]** kToken Collateral Composability — Vault receipts usable as collateral in K-Lend markets.

11. **[STANDARD]** Borrow Factors — Risk-adjusted weighting based on Asset Risk Score.

12. **[STANDARD]** Supply and Borrow Caps — Per-asset, continuously monitored by Risk Council.

13. **[STANDARD]** KMNO Governance Token — 5% quorum, time-weighted staking multiplier (up to 300% boost).

14. **[STANDARD]** Concentrated Liquidity Management — Automated CLMM vault management on Orca, Raydium, Meteora.

### Usability

1. **[CRITICAL]** One-Click Multiply/Leverage — Single-action execution within one Solana transaction.

2. **[CRITICAL]** Curated Vault Abstraction — Labeled risk profiles for passive depositors.

3. **[CRITICAL]** Real-Time Risk Simulator — Shows impact of position changes before execution.

4. **[STANDARD]** Unified Dashboard — Lend, Borrow, Multiply, Liquidity, Earn in one app.

5. **[STANDARD]** Target Leverage with Stop-Loss/Take-Profit.

6. **[STANDARD]** Points and Seasonal Rewards Transparency.

7. **[STANDARD]** Low Transaction Costs — Solana sub-cent fees.

### Reliability

1. **[CRITICAL]** Zero Bad Debt Track Record — Through all market conditions.

2. **[CRITICAL]** 18 External Security Audits.

3. **[CRITICAL]** Formal Verification (Certora, OtterSec).

4. **[CRITICAL]** Isolated Market Risk Architecture (V2) — No cross-market propagation.

5. **[CRITICAL]** Oracle Redundancy with TWAP/EWMA Filtering.

6. **[STANDARD]** Automated Deleverage as Circuit Breaker.

7. **[STANDARD]** Open Source Codebase.

8. **[STANDARD]** Risk Council Governance — Monthly risk insight reports.

### Performance

1. **[CRITICAL]** Solana-Native Transaction Speed — ~400ms blocks, sub-cent costs.

2. **[CRITICAL]** Scale: $2.8B+ TVL, $4.4B Total Deposits.

3. **[CRITICAL]** Dynamic Vault Rebalancing — Automatic allocation based on on-chain conditions.

4. **[STANDARD]** V2 Launch Traction — $200M deposits and $80M loans within weeks.

5. **[STANDARD]** Efficient Liquidation Pipeline — Auction-based with Solana speed.

6. **[STANDARD]** Capital Efficiency via eMode — 90%+ LTV for correlated assets.

### Supportability

1. **[CRITICAL]** Modular V2 Architecture — Market Layer and Vault Layer with clean abstraction boundaries.

2. **[CRITICAL]** Professional Curator Ecosystem — Institutional-grade risk firms.

3. **[CRITICAL]** TypeScript SDK (klend-sdk, kliquidity-sdk).

4. **[STANDARD]** On-Chain Governance with Forum.

5. **[STANDARD]** Permissionless Market Creation.

6. **[STANDARD]** Configurable Per-Reserve Risk Parameters — 11-point rate curves, per-asset oracle config.

7. **[STANDARD]** Developer Documentation.

8. **[STANDARD]** Monthly Risk Reporting.

---

<a id="jupiter-lend"></a>
## 7. Jupiter Lend

> ~$1.65B TVL | Launched Aug 2025 | Tick-based liquidation | DEX-integrated

### Functionality

1. **[CRITICAL]** Isolated vault architecture — Each supply-borrow pair lives in its own vault with independent configuration. Vaults share a unified liquidity layer underneath (isolation is at configuration level, not capital level).

2. **[CRITICAL]** Rehypothecation of collateral — Collateral in one vault can be lent to borrowers in other vaults. Primary capital-efficiency mechanism but also principal risk vector.

3. **[CRITICAL]** Tick-based liquidation engine (Fluid) — Positions bucketed into discrete "ticks" by debt-to-collateral ratio. Price movement past a tick boundary batch-liquidates all positions atomically. Enables 0.1% liquidation penalties and 95% LTV.

4. **[CRITICAL]** High LTV ratios — 90-95%, roughly 10-25pp above industry average, made possible by tick-based liquidation efficiency.

5. **[CRITICAL]** Multiply (one-click leverage loops) — Automates deposit-borrow-redeposit loop using flash loans in a single transaction.

6. **[CRITICAL]** Jupiter DEX aggregator integration — ~95% of Solana DEX aggregator volume. Liquidated collateral settles as ordinary swaps on the DEX.

7. **[STANDARD]** Earn (deposit-and-yield) — Supply tokens to unified liquidity layer. Automated debt ceiling smooths withdrawals block-by-block.

8. **[STANDARD]** Broad collateral support — 40+ vaults: stablecoins, BTC wrappers, LSTs, JLP, JUP, community assets, tokenized stocks.

9. **[STANDARD]** Two-layer modular architecture — Liquidity Program (single-pool base) + higher-level programs. Third-party protocols can build via CPI.

10. **[STANDARD]** Automated debt ceiling on withdrawals — Block-by-block ramp prevents bank-run dynamics.

### Usability

1. **[CRITICAL]** Single-interface DeFi superapp — Swaps, perpetuals, lending, borrowing, and leverage from one UI at jup.ag.

2. **[CRITICAL]** One-click Multiply execution — Displays max multiplier, projected net APY, and liquidation price.

3. **[STANDARD]** Per-vault risk transparency.

4. **[STANDARD]** Transparency page — Every audit, risk parameter, and vault configuration.

5. **[STANDARD]** Developer SDK (TypeScript and Rust).

### Reliability

1. **[CRITICAL]** Atomic batch liquidations — All positions in a risk tier liquidated in one transaction. Deterministic outcomes.

2. **[CRITICAL]** Multi-oracle price feeds — Pyth, Chainlink, Solana-native pool prices. Chaos Labs 24/7 monitoring.

3. **[STANDARD]** Multiple independent audits — OtterSec, Offside Labs, Mixbytes, $107K Code4rena competitive audit.

4. **[STANDARD]** Chaos Labs risk monitoring platform.

5. **[STANDARD]** Per-vault supply caps.

6. **[STANDARD]** Withdrawal smoothing (automated debt ceiling).

### Performance

1. **[CRITICAL]** Solana-native execution speed — ~400ms blocks enable single-slot batch liquidations.

2. **[CRITICAL]** Rapid TVL scaling — $500M in 24 hours, $1B in 10 days, $1.65B by October 2025.

3. **[CRITICAL]** Capital efficiency via rehypothecation — Collateral earns additional yield by being lent out.

4. **[STANDARD]** Flash-loan-powered Multiply — Single-transaction leverage loops.

5. **[STANDARD]** Low fee structure — 1% protocol fees, 0.1% liquidation penalties.

### Supportability

1. **[CRITICAL]** Composable base layer for third-party protocols — CPI-extensible. TypeScript + Rust SDK.

2. **[STANDARD]** Modular two-layer architecture — Independent component upgrades.

3. **[STANDARD]** Open audit and risk data.

4. **[STANDARD]** Vault-level configurability.

5. **[STANDARD]** Fluid partnership as architecture provider.

---

<a id="marginfi"></a>
## 8. Marginfi

> Significant TVL | Global + Isolated + Native Stake Markets | Advanced risk engine

### Functionality

1. **[CRITICAL]** Global Market (mrgnlend) — Primary lending pool. All listed assets interconnected. Up to 16 assets per account.

2. **[CRITICAL]** Isolated Markets — Separate risk pools for higher-risk tokens. Zero collateral weight. One isolated asset per account.

3. **[CRITICAL]** Native Stake Market — Use Solana staking derivatives (jitoSOL, bSOL) as collateral while earning staking rewards.

4. **[CRITICAL]** Risk Engine — Deterministic, end-to-end. Evaluates every transaction before execution. Oracle confidence-interval-adjusted.

5. **[CRITICAL]** Dual-Weight System (Initial / Maintenance) — Separate weights gate new borrows vs. liquidations. Creates health buffer.

6. **[CRITICAL]** Liquidation Mechanism — Partial liquidations to restore health (not full wipeout). 5% penalty split 2.5% to liquidator, 2.5% to insurance fund. Permissionless.

7. **[CRITICAL]** Flash Loans — Deferred health checks via `ACCOUNT_IN_FLASHLOAN` flag. Zero fees, zero interest.

8. **[CRITICAL]** Oracle Integration (Pyth / Switchboard) — Confidence-interval-adjusted pricing (`P - Pc` for assets, `P + Pc` for liabilities). 60-second staleness cap. EMA pricing.

9. **[CRITICAL]** Utilization-Based Interest Rate Curves — Per-asset curves. 12.5% spread on USDC/USDT/SOL, 13.5% on others.

10. **[STANDARD]** e-mode (Efficiency Mode) — Automatic boost for correlated assets (80% to 90% for SOL/LST pairs).

11. **[STANDARD]** mrgnloop (Leveraged Looping) — Flash loan-powered leverage in one transaction.

12. **[STANDARD]** The Arena — Permissionless leveraged long/short on any Solana token including memecoins.

13. **[STANDARD]** Insurance Fund — Funded by liquidation penalties and spread fees.

14. **[STANDARD]** Emissions / Rewards — Per-bank token incentives, first-come-first-served claims.

15. **[STANDARD]** Points System — 1 point/day per $1 lent, 4 points/day per $1 borrowed.

16. **[STANDARD]** Groups and Banks Architecture — Modular per-asset configuration.

### Usability

1. **[CRITICAL]** Unified Account with Multi-Asset Positions — Single account, up to 16 assets.

2. **[CRITICAL]** One-Click Leveraged Positions — The Arena / mrgnloop abstract multi-step strategies.

3. **[CRITICAL]** Real-Time Health Factor Display — 0-100% with dual-weight buffer visualization.

4. **[STANDARD]** Permissionless Market Creation — Via The Arena, CLI, or TypeScript SDK.

5. **[STANDARD]** Automatic e-mode Activation.

6. **[STANDARD]** Multiple Client Interfaces — Web app, Rust CLI, TypeScript SDK.

7. **[STANDARD]** Clear APY/APR Distinction.

### Reliability

1. **[CRITICAL]** Partial Liquidation Design — Avoids cascading liquidation spirals.

2. **[CRITICAL]** Oracle Confidence-Interval Pricing — Built-in pessimistic safety margin.

3. **[CRITICAL]** Risk Isolation Between Market Types — Global, Isolated, and Native Stake ring-fenced.

4. **[CRITICAL]** Deterministic On-Chain Risk Checks — No off-chain component.

5. **[STANDARD]** Insurance Fund Backstop.

6. **[STANDARD]** Security Audits — OtterSec. Flash loan vulnerability responsibly disclosed and patched (Sep 2025).

7. **[STANDARD]** Governance Multisig for Group Administration.

8. **[STANDARD]** Exponential Moving Average Pricing — Reduces flash-crash susceptibility.

### Performance

1. **[CRITICAL]** Solana-Native Architecture — ~400ms blocks, sub-cent fees.

2. **[CRITICAL]** Atomic Flash Loan Execution — Single-transaction multi-step operations.

3. **[CRITICAL]** Zero-Fee Flash Loans — Reduces friction for arbitrageurs and liquidators.

4. **[STANDARD]** Utilization-Based Dynamic Rate Curves.

5. **[STANDARD]** Capital Efficiency via e-mode.

6. **[STANDARD]** Proven Scale — $1.7B in liquidations processed, $810M peak TVL.

### Supportability

1. **[CRITICAL]** Fully Open Source — `mrgnlabs/marginfi-v2` on GitHub.

2. **[CRITICAL]** CPI Composability — Any on-chain program can integrate.

3. **[STANDARD]** Multi-Language SDK Support — Rust CLI/SDK + TypeScript SDK.

4. **[STANDARD]** Anchor Framework — IDL generation, standardized instruction encoding.

5. **[STANDARD]** Comprehensive Documentation.

6. **[STANDARD]** Modular Bank/Group Architecture — Add assets without program upgrades.

7. **[STANDARD]** Delegate Admin System.

---

<a id="savefinance-solend"></a>
## 9. Save.Finance (Solend)

> ~$395M deposits | 129+ assets | Oldest Solana lending protocol (Aug 2021)

### Functionality

1. **[CRITICAL]** Pool-Based Lending and Borrowing — Deposit to earn, borrow against overcollateralized collateral. LTV typically 75%, up to 95% in isolated pools.

2. **[CRITICAL]** cTokens (Receipt Tokens) — SPL tokens whose redemption value increases as interest accrues. Freely transferable and composable.

3. **[CRITICAL]** Dynamic Interest Rate Model — Kinked utilization curve: gentle increase up to optimal, steep increase beyond.

4. **[CRITICAL]** Liquidation Engine — 20% partial liquidation per transaction. 5% bonus to liquidators. 30% of penalty to protocol.

5. **[CRITICAL]** Isolated (Permissionless) Pools — 200 SLND initialization fee. Creator configures all parameters. Creator earns 20% of origination fees. Risk is contained per pool.

6. **[CRITICAL]** Main Pools with Curated Risk Parameters — Flagship pools with strict risk filters and governance-approved parameters.

7. **[CRITICAL]** Oracle Integration (Pyth + Switchboard) — 60-second update interval or 0.5% deviation trigger. CEX + DEX price sources.

8. **[STANDARD]** Flash Loans — Two-instruction atomic pattern within a single transaction.

9. **[STANDARD]** sUSD Decentralized Stablecoin — CDP-style, 0% interest, SOL-backed.

10. **[STANDARD]** saveSOL Liquid Staking Token — Earn staking yield while maintaining liquidity.

11. **[STANDARD]** dumpy.fun (Memecoin Shorting) — Specialized platform for shorting memecoins.

12. **[STANDARD]** SLND Governance Token — 60% community allocation. Minimum 1-day voting period.

13. **[STANDARD]** Fee Structure — 0.1% origination fee (80/20 protocol/host split). 20% interest spread.

### Usability

1. **[CRITICAL]** One-Click Deposit and Borrow — Clean interface, single-transaction operations.

2. **[CRITICAL]** Transparent Risk Dashboard — Per-pool display of LTV, liquidation parameters, utilization, APY.

3. **[STANDARD]** Permissionless Pool Creation UI — Admin interface for configuration without CLI.

4. **[STANDARD]** Developer SDK and Open-Source Codebase.

5. **[STANDARD]** Multi-Pool Navigation — Browse and filter 100+ pools.

6. **[STANDARD]** Solana Wallet Ecosystem Integration.

### Reliability

1. **[CRITICAL]** Battle-Tested Smart Contracts — Live since August 2021 through multiple market cycles.

2. **[CRITICAL]** Multiple Independent Audits — Kudelski, Neodyme, OSEC.

3. **[CRITICAL]** Pool Isolation as Containment — Bad debt in isolated pools cannot propagate.

4. **[CRITICAL]** Dual-Oracle Redundancy — Pyth primary, Switchboard fallback.

5. **[STANDARD]** Bug Bounty Program — Up to $1M.

6. **[STANDARD]** Deposit and Borrow Limits — Per-account and global caps.

7. **[STANDARD]** Governance Safeguards — Minimum 1-day voting period after 2022 incident.

### Performance

1. **[CRITICAL]** Solana-Native Speed — ~400ms blocks, ~30x faster than Ethereum.

2. **[CRITICAL]** Sub-Cent Transaction Costs — Makes small positions viable.

3. **[CRITICAL]** Efficient Liquidation Execution — Fast blocks + low fees enable profitable small-position liquidation.

4. **[STANDARD]** High Throughput for Composability.

5. **[STANDARD]** Real-Time Interest Accrual.

6. **[STANDARD]** Oracle Update Frequency — 60-second intervals or 0.5% deviation.

### Supportability

1. **[CRITICAL]** Open-Source Program and SDK.

2. **[CRITICAL]** Permissionless Pool Extensibility — Scale to new assets without core team.

3. **[STANDARD]** Comprehensive Documentation.

4. **[STANDARD]** Modular Architecture — Independent pool parameters and rate models.

5. **[STANDARD]** Host Fee Model for Integrators — 80/20 split incentivizes third-party frontends.

6. **[STANDARD]** DAO-Governed Upgradability.

7. **[STANDARD]** Pyth and Switchboard Oracle Compatibility.

---

<a id="port-finance"></a>
## 10. Port Finance

> Smaller TVL | Variable + Fixed-rate lending | Flash loans | Cross-collateral

### Functionality

1. **[CRITICAL]** Variable-Rate Lending and Borrowing — Core money market with utilization-driven two-segment interest rate curves.

2. **[CRITICAL]** Fixed-Rate Lending via Sundial — Zero-coupon bond model. Positions tokenized into pTokens (principal, redeemable 1:1 at maturity) and yTokens (variable-rate interest). Buying pTokens at discount = lending at fixed rate.

3. **[CRITICAL]** Cross-Collateral Support — Up to 9 distinct assets simultaneously in a single obligation account.

4. **[CRITICAL]** Flash Loans — Single-transaction uncollateralized loans. Atomic revert on failure. Arbitrage, self-liquidation, collateral swaps.

5. **[STANDARD]** Interest Rate Swaps — Swap between fixed and variable rate exposure.

6. **[CRITICAL]** Reserve Pool and Liquidity Management — Per-asset isolated reserves with protocol-owned reserve factor as safety buffer.

7. **[CRITICAL]** Liquidation Engine — Permissionless. Up to 50% of outstanding debt per transaction. Liquidation bonus incentive.

8. **[STANDARD]** PORT Governance Token — Voting on parameters, fee revenue sharing.

9. **[STANDARD]** Liquidity Mining Rewards — PORT distribution to depositors and borrowers.

10. **[STANDARD]** Principal/Yield Token Marketplace — Secondary market for interest rate exposure via pTokens and yTokens.

### Usability

1. **[CRITICAL]** Single Obligation Account Model — One on-chain account per user across all collateral/borrowed assets. Single risk factor.

2. **[STANDARD]** Risk Factor Display — 0-100%+ percentage summarizing liquidation proximity.

3. **[STANDARD]** Web Dashboard Interface.

4. **[STANDARD]** Anchor CPI Adaptor — Typed Anchor CPI instructions for composability.

5. **[STANDARD]** Wallet Integration — Standard Solana wallets.

6. **[STANDARD]** Transparent Interest Rate Curves.

### Reliability

1. **[CRITICAL]** Atomic Flash Loan Safety — Solana transaction atomicity guarantees.

2. **[CRITICAL]** Third-Party Security Audits — Kudelski Security, SlowMist, Halborn bugfix review.

3. **[CRITICAL]** Bug Bounty Program via Immunefi — Critical vulnerability ($20-25M potential) was reported and patched. $180K + $450K PORT bounty paid.

4. **[CRITICAL]** Permissionless Liquidation — Open-source liquidator bot (`port-finance/liquidator`).

5. **[STANDARD]** Oracle Price Feeds — Pyth Network dependency.

6. **[STANDARD]** Reserve Factor as Safety Buffer.

7. **[STANDARD]** Collateral Ratio Minimums — ~115%.

### Performance

1. **[CRITICAL]** Solana Runtime Advantages — Sub-second blocks, sub-cent fees, high throughput.

2. **[CRITICAL]** Single-Transaction Flash Loans — Complete in ~400ms.

3. **[STANDARD]** On-Chain Interest Accrual via Refresh — Amortized across user interactions.

4. **[STANDARD]** Efficient Account Layout — Parallel transaction processing.

5. **[STANDARD]** Capital Efficiency via Cross-Collateral — Up to 9 assets per obligation.

### Supportability

1. **[CRITICAL]** Open-Source Codebase — `port-finance` GitHub: variable-rate-lending, sundial, liquidator, port-anchor-adaptor.

2. **[STANDARD]** Modular Program Architecture — Variable-rate and Sundial as separate programs.

3. **[STANDARD]** Anchor Framework Integration.

4. **[STANDARD]** Governance-Controlled Parameters.

5. **[STANDARD]** Documentation — docs.port.finance + litepaper.

6. **[STANDARD]** Published Rust Crates on crates.io.

---

## Cross-Protocol Summary: Critical Feature Patterns

The following features appear as **[CRITICAL]** across most or all protocols, representing the essential building blocks of a lending/borrowing platform:

| Feature | Protocols |
|---|---|
| **Over-collateralized lending/borrowing** | All 10 |
| **Algorithmic interest rate model** (utilization-based, kinked curve) | All 10 |
| **Permissionless liquidation engine** | All 10 |
| **Oracle integration** (multi-source preferred) | All 10 |
| **Governance system** | All 10 |
| **Flash loans** | 9/10 (all except JustLend) |
| **Efficiency/Elevation mode** (higher LTV for correlated assets) | Aave, Kamino, Marginfi, Spark, Compound (V3 single-asset model achieves similar) |
| **Isolated/permissionless markets** | Morpho, Kamino, Save.Finance, Marginfi, Port Finance, Jupiter Lend |
| **Curated vault abstraction** (for passive lenders) | Morpho, Kamino, Spark |
| **Composable receipt tokens** (aTokens, cTokens, jTokens, kTokens) | Aave, Compound, JustLend, Save.Finance, Port Finance, Kamino |
| **One-click leverage/multiply** | Kamino, Jupiter Lend, Marginfi |
| **Liquid staking integration** | JustLend (sTRX), Kamino, Marginfi |
| **Open-source codebase** | All 10 |
| **Multiple independent audits** | All 10 |
| **Battle-tested/immutable contracts** | Aave, Compound, Morpho, JustLend, Save.Finance |

### Differentiating Critical Features (unique or rare)

| Feature | Protocol | Why it matters |
|---|---|---|
| **Immutable base layer (~650 LoC)** | Morpho | Eliminates upgrade risk entirely |
| **GHO native stablecoin** | Aave | Revenue directly to DAO treasury |
| **RWA-backed yield subsidies** | Spark | Off-chain revenue funds competitive on-chain rates |
| **Tick-based batch liquidation** | Jupiter Lend | 0.1% penalty, 95% LTV |
| **Rehypothecation** | Jupiter Lend | Capital efficiency (and risk) multiplier |
| **Auction-based liquidation** | Kamino | Dutch auction drives down penalties competitively |
| **Auto-deleverage circuit breaker** | Kamino | Proactive risk management with 72h grace period |
| **Energy rental market** | JustLend | Tron-specific gas abstraction |
| **Fixed-rate lending via zero-coupon bonds** | Port Finance | Interest rate hedging primitive |
| **Umbrella safety system** (automated slashing) | Aave | No-governance-vote bad debt response |
| **Kill Switch module** | Spark | Auto-disables borrowing on LST depeg |
| **Confidence-interval oracle pricing** | Marginfi | Pessimistic safety margin built into every price |
