# Appendix: Reflexive Stablecoin Ecosystem

This appendix surveys existing stablecoin protocols across the DeFi ecosystem. It serves as a reference for [RFP-013: Reflexive Stablecoin Protocol](../RFPs/RFP-013-reflexive-stablecoin-protocol.md), providing context on design approaches, their trade-offs, and why a governance-minimized reflexive model was selected.

The stablecoin design space spans centralized custodial models (USDC), governance-heavy over-collateralized systems (Sky/MakerDAO), governance-minimized reflexive systems (RAI), and non-governance alternatives (LUSD). This appendix focuses on the subset relevant to RFP-013's design choice: why RAI-style reflexive control was selected over governance-heavy or centralized alternatives.

## Stablecoin Protocols Considered

Protocols are ordered by relevance to RFP-013's design philosophy (closest match first).

| Protocol | Type | Governance | Collateral | Primary Mechanism | Launch | TVL/Status |
|----------|------|------------|------------|-------------------|--------|------------|
| Reflexer (RAI) | Reflexive | Governance-minimized | ETH only | PID controller, floating redemption price | 2021 | ~$5M TVL, ~$1.8M mcap; low activity but operational |
| HAI | Reflexive | Governance-minimized | Multi | RAI fork with multi-collateral | 2024 | Active on Optimism |
| Liquity V1 (LUSD) | Non-governance | No governance | ETH only | Redemption-based, pure ETH backing | 2021 | ~$153M TVL; LUSD supply ~$37M (declining due to V2 migration) |
| Liquity V2 (BOLD) | Non-governance | No governance | ETH, wstETH, rETH | User-set interest rates, immutable | 2025 | ~$95M TVL; actively growing |
| Sky/MakerDAO (DAI/USDS) | Governance-heavy | Extensive | Multi | DSChief governance, MKR/SKY voting | 2017 | ~$5.3B TVL (Sky Lending); rebranded to Sky in 2024 |
| USDC | Centralized | Corporate | Fiat reserves | Off-chain custody, mint/burn | 2018 | ~$60B+ supply |

## Stablecoin Design Spectrum

### Centralized: USDC

USDC is issued by Circle with fiat reserves held at regulated banks. It offers the tightest peg stability and deepest liquidity but requires trust in Circle's solvency, reserve management, and regulatory compliance. Circle maintains blacklist capabilities and can freeze accounts.

**Why not this approach for LEZ:** Centralization contradicts Logos' ethos of credibly neutral infrastructure. Blacklist capability and issuer dependencies create systemic risks. RFP-013 explicitly targets decentralized alternatives.

### Governance-Heavy: Sky/MakerDAO (DAI/USDS)

MakerDAO (rebranded to Sky in 2024) operates via MKR/SKY token governance. Protocol parameters (stability fees, collateral types, debt ceilings) require voting. The system uses multi-collateral backing (ETH, WBTC, real-world assets) and has evolved toward broad collateral inclusion and governance-active management. The rebrand introduced USDS alongside DAI, with DAI remaining operational.

**Key mechanism:** Users lock collateral in Vaults, mint DAI/USDS against it. If collateral ratio falls below liquidation ratio, position is liquidated via auction.

**Why not this approach for LEZ:** Extensive governance creates capture risks and reduces predictability. The reflexive stablecoin approach trades governance intervention for mathematical self-correction.

### Governance-Minimized: RAI / Reflexer

RAI pioneered the "governance minimized, reflexive" stablecoin model. It uses only ETH collateral and removes most governance parameters in favor of a feedback controller.

**Core mechanism:**
- Redemption price: A floating target price that drifts over time
- Redemption rate: The rate at which redemption price changes, computed by a PID controller based on market price deviation
- Market price vs redemption price: If market price trades at premium to redemption price, redemption rate goes negative, causing redemption price to fall and eventually reducing demand
- Self-stabilization: No governance votes to adjust rates; mathematics enforces equilibrium

**Position structure (SAFEs):**
- Locked ETH collateral
- Normalized debt (debt units, not nominal RAI)
- Accumulated rates (stability fees applied continuously)
- Nominal debt = normalized debt × accumulated rate

**Current status:** RAI remains operational but with very low activity (~397 active Safes, ~$5M TVL, daily trading volume under $15K). The protocol achieved its governance-minimization goals but struggled with adoption, partly due to the difficulty of bootstrapping liquidity for a non-pegged asset and negative redemption rates discouraging LP participation.

**Why this approach for LEZ:** Governance minimization aligns with LEZ's goal of credibly neutral infrastructure. The system is more predictable (no surprise parameter changes) and harder to capture. RFP-013 follows this design with LEZ-specific adaptations. RAI's adoption challenges are noted; LEZ's integrated ecosystem may provide a more favorable bootstrapping environment.

### Non-Governance: Liquity (LUSD / BOLD)

Liquity V1 eliminates governance entirely. It uses pure ETH backing, a fixed 110% collateralization ratio, and a redemption mechanism (users can redeem LUSD for ETH at face value, effectively creating a price floor). The protocol is fully immutable with zero exploits to date.

Liquity V2 launched in January 2025 with its new stablecoin BOLD, expanding collateral to include ETH, wstETH, and rETH, and introducing user-set interest rates. V2 maintains full immutability while addressing V1 limitations around forced deleveraging from aggressive redemptions. V2 directs 75% of interest revenue to Stability Pool depositors and 25% to DEX liquidity providers.

**Key difference from RAI:** No floating redemption price. LUSD/BOLD target $1.00 via redemption arbitrage (if LUSD < $1, buy and redeem for $1 of ETH). The protocol has no parameter governance - everything is immutable.

**Why not this approach for LEZ:** While attractive for its immutability, Liquity's redemption mechanism creates different dynamics. RAI uses a **floating redemption price** as its target - the "peg" exists but drifts based on market conditions and PID controller feedback. This is distinct from both rigid $1.00 pegs (LUSD, DAI, USDC) and completely unpegged free-float tokens. The floating target allows the system to self-correct without governance intervention while still providing a measurable redemption value.

### Multi-Collateral Reflexive: HAI

HAI is a direct fork of RAI on Optimism that adds multi-collateral support (stETH, rETH, etc.) while maintaining the reflexive redemption rate mechanism. It demonstrates that the RAI model can evolve beyond pure ETH backing.

**Relevance to RFP-013:** Shows evolutionary path if multi-collateral is desired later. RFP-013 starts with single-collateral for simplicity but could theoretically extend following HAI's pattern.

## Design Consideration: Reflexive + Redemption Hybrid

A natural question is whether combining RAI's PID controller with Liquity's direct redemption mechanism would address RAI's known adoption challenges while preserving the non-pegged, governance-minimized design. No production system has attempted this combination, but the design space is worth examining to explain why RFP-013 starts with pure reflexive control.

### What the hybrid would look like

In a hybrid model, anyone could redeem LEZ for `redemption_price` worth of ETH (from the riskiest CDPs, Liquity-style), creating an immediate arbitrage floor: if `LEZ_market < redemption_price`, buy on market, redeem for ETH, pocket the difference. The PID controller would continue managing the ceiling (market above redemption price triggers negative redemption rate, gradually reducing the target). The result is belt-and-suspenders stabilization: redemptions enforce the floor instantly, the PID manages the ceiling gradually, and the target still floats rather than pegging to $1.

### Why this is attractive

RAI's bootstrapping problem stems from having no fast correction mechanism in either direction. The PID slowly grinds price deviations down over hours or days, requiring sophisticated arbitrageurs and deep secondary market liquidity that RAI never achieved. Direct redemptions would create day-one utility and arbitrage opportunity without requiring deep liquidity. The average absolute deviation from target would be smaller, which means negative redemption rate episodes (the main LP deterrent in RAI) would be shorter and shallower, though not eliminated - the PID still needs negative rates when market trades above target.

### Why this approach was rejected

The hybrid introduces a dual-feedback-loop control problem that, upon formal analysis, turns out to be asymmetric in a way that undermines the design rationale.

The redemption floor only binds when the market trades *below* the redemption price — it creates arbitrage that pushes the price back up to floor. However, the dominant failure mode that drove RAI's adoption collapse is an *upside* demand event: persistent buying pressure keeps the market *above* target, the PID drives the redemption rate negative for extended periods, LP yield turns into carry bleed, LPs exit, the order book thins, deviations widen, the rate goes more negative, and the loop closes. The hybrid's redemption floor never fires in this regime.

Simulation analysis confirms this asymmetry. In prolonged-negative-rate scenarios (the RAI death spiral), hybrid and pure PID collapse to identical outcomes — the same LP floor, the same ~6.3% mean peg error, the same supply blow-out. The hybrid provides **zero protection** against the single failure mode that matters most. The negative-rate regime persisted ~96% of the scenario duration in simulation; the redemption floor was inert throughout.

Where the hybrid *does* help is symmetric bank-run and liquidity-crunch events, cutting max peg error from ~13% to ~1.2% and keeping LP P&L positive. This is a real but narrow benefit — and it is available through a much smaller surface area: an emergency redemption module that is off by default and activated only when a circuit breaker fires (e.g., >3% sustained downside deviation for >6h).

Beyond the asymmetry problem, the hybrid increases attack surface — a CDP vault, instant redemption, and a PID controller form three coupled control loops to harden — and adds parameter overhead (redemption fee, capacity, floor activation threshold). The additional complexity creates a false sense of security that may tempt weaker PID gains. Liquity-style redemptions also force-close the riskiest CDPs involuntarily (V1's biggest UX pain point), requiring an interest-rate priority mechanism (à la Liquity V2) that adds further protocol complexity.

A mechanism comparison across 11 families (29 candidates) found that LP-Lockup — structurally locking a fraction of pool liquidity against withdrawal shocks — is the only mechanism strictly dominant over pure RAI on every scenario × metric cell tested. The hybrid floor loses on LP P&L or depth in at least one stress scenario. The dominant risk to the system is LP retention under sustained negative carry, not a gap that can be closed at the redemption layer.

A separate sweep of insurance reserve + fee kicker combinations (49 candidates × 8 scenarios × 24 seeds = 9,600 simulations) reinforces this conclusion and adds an important nuance. An insurance reserve of 8–12% of TVL with adaptive deployment (scaling intervention with error severity) achieves 16–21% less peg error than pure RAI with LP returns slightly improved (ratio 1.007–1.010), scoring 34 wins / 21 ties / 1 loss. However, the fee kicker — routing negative-rate holding tax to LPs as a subsidy — is counterproductive at every setting tested. Kicker payments attract LP entry, the pool overcrowds, per-unit returns dilute by more than the kicker compensates, and net LP P&L drops to 97% of RAI at full kicker. The optimal kicker setting is zero. Direct peg intervention (insurance reserve buying/selling against deviations) beats indirect incentives (fee routing to LPs) because second-order effects (overcrowding, dilution) dominate the intended first-order benefit.

RFP-013 specifies pure reflexive control. If downside liquidity crunches prove problematic in production, the recommended path is an emergency redemption module (off by default, activated by circuit breaker), not a permanently active hybrid floor. The effective supplementary mechanisms are an insurance reserve with adaptive deployment, aggressive integrator anti-windup with hard clamping, and TWAP oracle hardening with staleness circuit breakers — these address the actual variance in the system.

## Why Public State for CDPs

### The Privacy Architecture Challenge

A natural question: why not private position state? If users value privacy, shouldn't CDP positions be shielded?

**The technical reality:** Private CDP state is an open research problem with no production implementation.

### Why CDP Positions Must Be Public

| Requirement | Why Public State Necessary | Private State Challenge |
|-------------|---------------------------|------------------------|
| **Liquidation monitoring** | Keepers must identify undercollateralized positions to trigger liquidation | With private positions, the question becomes *who* checks health and *when*. Oracle prices are public, and the ZK circuit itself is straightforward (prove `private_collateral * public_price < private_debt * liquidation_ratio`). The hard problem is authority and timing: if the CDP owner generates the proof, they control when liquidation-eligibility becomes visible (see research challenge 1 below). If a third party checks, they need access to the private position data, reintroducing trust. |
| **Permissionless position scanning** | Keepers, aggregators, and yield optimizers scan all positions to find liquidation opportunities, compute system-wide collateralization, and build on top of the protocol | Private state breaks *discoverability*: external protocols cannot enumerate or scan positions without owner cooperation. Owner-authorized interactions (e.g., passing a private record to a composing program) still work, but permissionless scanning by keepers, dashboards, and risk monitors does not. |
| **Systemic risk observability** | Total system collateralization, bad debt accumulation, and concentration risk are visible in real-time, allowing markets to price systemic risk | With private CDPs, liquidation failures become silent. Nobody knows how much bad debt is accumulating until it surfaces as a protocol insolvency event. For a reflexive stablecoin where the PID controller depends on market confidence, this observability is load-bearing: if the market cannot assess system health, the redemption price signal loses credibility, undermining the core stabilization mechanism. |

**The RFP-013 approach:**
- Position state: Public (collateral amounts, debt, ratios)
- User privacy: Enforced at UX layer via deshield-interact-reshield pattern

This matches the architecture in RFP-008 (Lending & Borrowing): public protocol state with optional private user interaction.

### Research Frontier: ZK CDPs

Private CDP systems are an active research area, but no production-ready system exists. Key projects exploring relevant primitives:

- **Aztec Network:** The original Aztec Connect (private DeFi bridge using UTXO model) was [sunsetted in March 2023](https://medium.com/aztec-protocol/sunsetting-aztec-connect-a786edce5cae) due to centralization and resource constraints. The team rebuilt from scratch and launched Aztec Ignition Chain in November 2025 as a general-purpose privacy L2 on Ethereum, entering Alpha in early 2026. Aztec now supports programmable private smart contracts via Noir, meaning CDPs *could theoretically* be built on it, but none exist yet. A critical proving-system vulnerability disclosed in March 2026 (fix planned for July 2026) underscores the immaturity of the execution environment.
- **Aleo:** ZK-native L1 with mainnet live and the most advanced privacy-native DeFi infrastructure among the protocols listed. Paxos launched USAD (private stablecoin) in February 2026 and Circle launched USDCx (confidential USDC) in January 2026. Confidential DeFi tools (dark pools, private lending) are being explored in the ecosystem, but no CDP/stablecoin-minting system exists in production. Aleo's Leo language and ZK execution model could support private CDPs, but the coordination problems described below remain unsolved.
- **Manta Network:** Privacy-focused Ethereum L2 (Manta Pacific), not a ZK-native L1. Focused primarily on private token swaps via its MantaSwap product, processing 12M+ ZK transactions in Q1 2026. The Polkadot-based Manta Atlantic parachain is sunsetting August 2026. No CDP or lending primitives exist on Manta.

**Note on "zk" terminology:** Some protocols use "zk" (e.g., StarkNet, zkSync) for validity proofs that provide scalability and data compression, not transaction privacy. Lending protocols on these chains (e.g., the now-defunct zkLend on StarkNet, which was hacked for $9.5M in February 2025 and [shut down in June 2025](https://www.bitget.com/news/detail/12560604834674)) had fully public position state despite operating on "zk-rollups". This distinction matters: zk-for-scalability ≠ zk-for-privacy.

**The research challenges:** Private state CDPs face fundamental coordination problems centered on *who checks position health* and *when*:

1. **Owner-generated health proofs (the timing problem):** The most natural ZK approach: the CDP owner periodically proves their position is healthy against a public oracle price, without revealing collateral or debt amounts. The ZK circuit is tractable (prove `collateral * price >= debt * ratio` over private inputs). The problem is that the owner controls proof cadence. If a price crash makes their position undercollateralized, they can delay submitting the proof while racing to add collateral or repay debt. This turns liquidation from a permissionless real-time check into a game of timing, undermining system solvency during exactly the conditions (rapid price drops) when liquidation matters most. A protocol-mandated proof interval (every N blocks) mitigates this but doesn't eliminate it: the owner still has N blocks of grace period, and gas costs per proof create ongoing UX burden.

2. **Mandatory periodic proofs with default liquidation:** A strengthened version of (1): require health proofs every N blocks, and treat failure to prove as liquidation-eligible. This creates conditional privacy: positions stay private as long as owners keep proving health. This is a legitimate design option with real tradeoffs - it adds per-epoch gas costs for every CDP holder, creates a UX burden (miss a proof window and your healthy position gets liquidated), requires careful N calibration (too short = expensive, too long = undercollateralized positions persist), and still allows N-block gaming windows. Importantly, when a position *is* liquidated via proof timeout, the liquidator still needs to learn the position details to execute, requiring either a reveal step or trusted execution.

3. **Trusted keepers / decryption committees:** A set of permissioned parties (a keeper committee, threshold decryption group, or TEE enclave) with access to encrypted position data, checking health and triggering liquidations. Practically simpler than ZK approaches but reintroduces centralization and trust: the committee must be trusted not to front-run, not to selectively liquidate, and to remain available. For Logos' credibly neutral infrastructure goals, this is a poor fit, though it may be acceptable for other protocol contexts.

4. **FHE (Fully Homomorphic Encryption):** Emerging FHE-based DeFi infrastructure (Fhenix, Zama/fhEVM, Inco) allows on-chain computation over encrypted data without decryption. In principle, a protocol could compute `encrypted_collateral * public_price < encrypted_debt * threshold` homomorphically and produce a boolean liquidation-eligibility result. This would solve the timing problem since the protocol itself checks health at every block. However, FHE operations are orders of magnitude more expensive than plaintext computation, current implementations cannot support real-time liquidation checks at the throughput required for a production CDP system, and the tooling is immature. This is the most promising long-term direction but not production-viable as of April 2026.

5. **Forced full decryption on timeout:** Protocol decrypts the entire position after X blocks of inactivity. Unlike approach (2), this reveals actual amounts rather than just health status, completely breaking privacy for inactive or distressed positions. Creates perverse incentives where owners must "ping" regularly to maintain privacy, and the threat of full reveal may deter privacy-conscious users from using the protocol at all.

None of these approaches are production-ready as of April 2026. Approach (2) is the closest to viable but carries significant UX and economic overhead. Approach (4) is the most architecturally promising but furthest from production. RFP-013 acknowledges this landscape and scopes privacy to the UX layer - public protocol state enables permissionless liquidation, systemic risk observability, and composability, while the deshield-interact-reshield pattern provides user-level privacy for their interactions with the protocol.

## Fee Structure Comparison

| Protocol | Stability Fee | Liquidation Penalty | Revenue Destination |
|----------|--------------|---------------------|---------------------|
| Sky/MakerDAO | Variable (governance vote) | 13% flat | MKR buy-and-burn, surplus buffer |
| RAI | Variable (PID controller) | 12% | Surplus auction (burn ETH) |
| Liquity V1 | 0.5% one-time (borrowing fee) | No explicit penalty; SP depositors capture excess collateral (~10% at 110% CR) | LQTY stakers, Stability Pool |
| Liquity V2 | User-set interest rate | No explicit penalty; SP depositors capture excess collateral | 75% Stability Pool, 25% DEX LPs |
| RFP-013 | Configurable (governance for rate, controller for mechanism) | Configurable | Protocol-defined |

RFP-013 leaves fee structure flexible but recommends RAI-style accumulation: stability fees accumulate in a surplus buffer, with surplus auctions when above threshold.

## Design Decisions Summary

| Decision | Selected Approach | Rejected Alternatives | Rationale |
|----------|-----------------|----------------------|-----------|
| **Price targeting** | Floating redemption price (reflexive) | Fixed $1.00 peg | Self-correcting without governance |
| **Rate control** | PID controller | Governance voting | Minimize capture risk |
| **Governance scope** | Limited (safety ratios, fees) | Extensive (rates, collateral types) | Predictable, harder to game |
| **Stabilization** | Pure reflexive (PID controller) | Reflexive + redemption hybrid | Simulation analysis shows the hybrid's redemption floor is asymmetric — addresses downside only, provides zero protection against the dominant upside death spiral (the RAI failure mode). Emergency redemption as a circuit-breaker-activated backstop remains an option for downside crunches. |
| **Collateral** | Single-asset initially (simple) | Multi-collateral at launch | Reduce complexity risk |
| **State privacy** | Public protocol state | Private state | Technical feasibility, permissionless liquidation, protocol integration |
| **User privacy** | UX-layer (deshield-interact-reshield) | None, or ZK state | Best available tradeoff |

## References

1. [RAI Whitepaper](https://github.com/reflexer-labs/whitepapers/blob/master/English/rai-english.pdf) - Reflexer Labs, 2021
2. [GEB Code Repository](https://github.com/reflexer-labs/geb) - Reflexer implementation
3. [Sky (formerly MakerDAO) Documentation](https://docs.sky.money/) - Sky protocol mechanics
4. [Liquity V1 Whitepaper](https://www.liquity.org/blog/whitepaper) - Non-governance stablecoin design
5. [Liquity V2](https://www.liquity.org/) - Immutable borrowing with user-set interest rates
6. [HAI Documentation](https://docs.letsgethai.com/) - Multi-collateral RAI fork
7. [Reflexer Blog: Why RAI](https://reflexer-labs.medium.com/why-rai-2f8502ba1fbd) - Governance minimization rationale
8. [DeFi Llama: Stablecoins](https://defillama.com/stablecoins) - Market data and TVL comparisons
9. [Aztec Network](https://aztec.network/) - Privacy L2 on Ethereum
10. [Aleo](https://aleo.org/) - ZK-native L1 blockchain
11. [Sunsetting Aztec Connect](https://medium.com/aztec-protocol/sunsetting-aztec-connect-a786edce5cae) - Aztec Labs, March 2023

---

*This appendix was prepared to support RFP-013. For the companion liquidation mechanisms appendix, see [Appendix: Liquidation & Auction Ecosystem](./appendix-liquidation-auction-ecosystem.md). For clarification or additions, please use the RFP repository Discussions.*
