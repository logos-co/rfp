# Appendix: Liquidation & Auction Ecosystem

This appendix surveys existing liquidation mechanisms and auction systems across the DeFi ecosystem. It serves as a reference for [RFP-014: Liquidation & Auction Engine](../RFPs/RFP-014-liquidation-auction-engine.md), providing context on design evolution, trade-offs between mechanisms, and why an increasing-discount auction model was selected as a reusable engine pattern.

While liquidation systems are typically tightly coupled to their host CDP protocols, RFP-014 proposes a **reusable liquidation program** that any CDP product can deploy as its own instance. This appendix examines the evolution of liquidation mechanisms (particularly MakerDAO's progression and Reflexer's discount-based approach), alternative mechanisms excluded from scope, and the architectural rationale for decoupling liquidation from CDP logic.

## Liquidation Mechanisms Surveyed

| Protocol | Auction Type | Speed | Incentive Model | Reusability |
|----------|--------------|-------|-----------------|-------------|
| Reflexer (RAI) | Increasing discount | Fast (seconds-minutes) | Escalating keeper discount | Coupled to RAI |
| Sky/MakerDAO V1 | English auction | Slow (hours) | Bid competition | Coupled to Maker |
| Sky/MakerDAO V2+ (Liq 2.0) | Dutch auction | Medium (minutes) | Descending price, flash-loan composable | Coupled to Maker |
| Aave/Compound | Direct liquidation | Instant (atomic) | Bonus % of collateral | Coupled per protocol |
| Liquity V1 | Stability Pool absorption | Instant | SP depositors receive discounted collateral | Unique to Liquity |
| Liquity V2 | Stability Pool absorption | Instant | SP earns 75% of interest + liquidation gains | Unique to Liquity |
| crvUSD (LLAMMA) | Soft liquidation via AMM | Continuous | Gradual conversion, reversible | Coupled to Curve |

## Liquidation Mechanism Evolution

### Sky/MakerDAO: The Evolution from English to Dutch

MakerDAO's (now Sky) liquidation mechanism has evolved across versions, demonstrating the trade-offs in auction design:

**Liquidation 1.2: English Auctions (Ascending Price)**
- Collateral seized when position undercollateralized via `bite` transaction
- Two-phase English auction: first phase (`tend`) raises DAI bids for fixed collateral lots; second phase (`dent`) returns excess collateral while maintaining the DAI bid
- **Problem:** Slow. Auctions could take hours, with bidder capital locked until outbid or auction settlement
- Exposed to network congestion risk: during MakerDAO's ["Black Thursday" event (March 2020)](https://blog.makerdao.com/the-market-collapse-of-march-12-2020-how-it-impacted-makerdao/), gas prices spiked and liquidations cleared at near-zero bids, creating ~$6M in bad debt

**Liquidation 2.0: Dutch Auctions (Descending Price) - [MIP 45](https://forum.makerdao.com/t/mip45-liquidations-2-0-liq-2-0-liquidation-system-redesign/6352)**
- Collateral offered via `Clipper` contracts at a high starting price that decreases deterministically over time according to a configurable price-vs-time curve (linear, stepwise exponential, or continuous exponential)
- First bidder to accept price wins via `take` function; partial bids permitted
- **Key improvement:** Single-block composability - bidders can use flash loans to purchase collateral atomically with zero capital requirement, leveraging all on-chain DAI liquidity
- Keeper incentive: flat DAI tip plus percentage-based chip proportional to vault debt size
- Per-collateral circuit breakers limit total DAI being raised simultaneously

**Why Liq 2.0 matters for RFP-014:** The move from English to Dutch eliminated capital lockup and enabled flash-loan participation. However, Dutch auctions still require time for price discovery (the price must descend to a level bidders accept). RFP-014's discount-based approach trades price discovery for speed: the discount is known immediately, removing the waiting period.

### Reflexer (RAI): Discount-Based Auctions

Reflexer's GEB framework implements [three collateral auction types](https://docs.reflexer.finance/system-contracts/auction-module), each representing a different point in the speed vs. price-discovery tradeoff:

**1. English Collateral Auction House** - Two-phase ascending auction similar to MakerDAO Liq 1.2. Included for compatibility but not used in RAI production due to speed limitations.

**2. Fixed Discount Collateral Auction House** - Collateral sold at a constant discount to the current oracle price throughout the auction's lifetime. Bidders call `buyCollateral`, submitting RAI and receiving collateral at the fixed discount. No price discovery - the discount is set at auction creation and never changes.

**3. [Increasing Discount Collateral Auction House](https://docs.reflexer.finance/system-contracts/auction-module/increasing-discount-collateral-auction-house)** - Starts with a small discount (`minDiscount`) that grows over time at `perSecondDiscountUpdateRate` up to `maxDiscount`. This is the variant RAI used in production. It combines speed (early bidders get collateral quickly at a small discount) with resilience (if no one bids early, the growing discount eventually attracts bidders).

**In all discount variants:**
- Bidders submit RAI, and the contract calculates collateral to return based on the current discount, the collateral's oracle price, and the system coin's redemption price
- Multiple bidders can purchase from the same auction (partial fills)
- Flash-loan compatible: bidders can atomically buy collateral and sell on DEX
- No capital lockup period
- Liquidation penalty of 12% provides keeper incentive margin

**Why increasing discount for RFP-014:** The increasing discount model gives the best balance for a new ecosystem: it creates immediate opportunity for early keepers (small discount) while guaranteeing auction completion even in thin markets (discount escalates). This is more forgiving than a fixed discount (which may be set too low for thin markets or too high, giving away value) and faster than Dutch auctions (no descending price curve to wait through).

## Alternative Mechanisms (Out of Scope)

### Direct Liquidation with Bonus (Aave, Compound)

**Mechanism:**
- Liquidator repays a portion of the borrower's debt (up to a close factor, e.g. 50%)
- Receives borrower's collateral at a bonus (e.g. 5-10% above debt value)
- Can be composed with flash loans: borrow → repay debt → seize collateral → sell collateral → repay flash loan, all atomically

**Advantages:**
- Instant, single-transaction settlement
- No auction state to manage
- With flash loans, zero capital requirement for liquidators

**Why not selected for RFP-014:**
- Requires deep on-chain liquidity for flash loans to be profitable
- Fixed bonus doesn't adapt to market conditions (thin markets may need higher incentive)
- No price discovery: bonus is governance-set, not market-determined
- Tightly coupled to lending protocol account structure
- LEZ liquidity landscape uncertain at this stage

### Stability Pool Liquidation (Liquity)

Liquity's liquidation has three distinct layers:

**1. Trigger (keepers):** External keepers monitor Trove collateralization ratios and call `liquidateTroves()` when positions fall below 110% CR. Keepers receive gas compensation (200 LUSD + 0.5% of the liquidated Trove's collateral) for triggering. This is a conventional keeper role identical in function to MakerDAO or Reflexer keepers - someone must detect undercollateralization and initiate the process.

**2. Resolution - primary (Stability Pool):** LUSD/BOLD depositors in the Stability Pool act as pre-committed liquidation counterparties. When a liquidation is triggered, the SP's LUSD is burned to cover the Trove's debt, and SP depositors receive the Trove's ETH collateral pro-rata. The spread between the Trove's debt value and its collateral value (at 110% CR, this is roughly a 10% gain) is the SP depositor's reward. Settlement is instant - no auction, no bidding, no waiting.

**3. Resolution - fallback (redistribution):** If the Stability Pool has insufficient LUSD to absorb the debt, the remaining debt and collateral are redistributed across all active Troves proportionally. This is a socialized backstop that doesn't require any additional infrastructure.

**Note:** This is distinct from Liquity's *redemption* mechanism (which lets anyone exchange LUSD for ETH at face value from the lowest-collateralized Troves). Redemptions are a peg stability mechanism, not a liquidation mechanism. See the [Reflexive Stablecoin appendix](./appendix-reflexive-stablecoin-ecosystem.md#design-consideration-reflexive--redemption-hybrid) for discussion of how redemptions interact with reflexive stablecoin design.

**Advantages:**
- SP depositors are pre-committed capital, eliminating auction timing risk
- Instant resolution - no price discovery delay
- Keeper role is lightweight (trigger only, no bidding strategy needed)
- MEV extraction reduced: SP depositors all receive the same pro-rata share, no competitive bidding

**Why not selected as the primary mechanism for RFP-014:**
- Requires a dedicated Stability Pool with its own incentive structure (interest share in V2, LQTY rewards in V1), creating a bootstrapping dependency separate from the CDP protocol
- SP must have sufficient deposits before liquidations can be processed reliably - on a new chain, this is a chicken-and-egg problem
- Deeply coupled to Liquity's token mechanics; not easily abstracted behind a generic interface
- Less flexible for different CDP designs that may have different collateral types or risk parameters

**However**, the trigger layer (keeper detects undercollateralization, calls liquidation function) is shared across all approaches. The difference is in resolution: Liquity routes to a Stability Pool, RFP-014 routes to an increasing-discount auction, and Aave-style direct liquidation resolves at the CDP level without an engine. See [Compatibility with RFP-013 Hybrid Evolution](#compatibility-with-rfp-013-hybrid-evolution) for how these paths relate to the hybrid design discussed in the stablecoin appendix.

### Soft Liquidation via AMM (crvUSD / LLAMMA)

**Mechanism:**
- Curve's [LLAMMA (Lending-Liquidating AMM Algorithm)](https://docs.curve.fi/crvUSD/amm/) places borrower collateral into a specialized AMM that automatically converts between collateral and crvUSD as the price moves
- As collateral price drops, the AMM gradually sells collateral for crvUSD (soft liquidation)
- As price recovers, the AMM gradually buys collateral back (de-liquidation)
- Full liquidation only occurs if the position is left in soft liquidation too long and losses accumulate

**Advantages:**
- Gradual, reversible liquidation reduces the impact of temporary price wicks
- No keeper infrastructure for the soft liquidation phase
- Borrowers can recover without being fully liquidated

**Why not selected for RFP-014:**
- Requires a custom AMM (LLAMMA) integrated with the CDP system - cannot be decoupled as a reusable engine
- Complex pricing math involving band-based liquidity distribution
- Borrowers suffer "soft liquidation losses" from the AMM's spread even if they recover
- Novel mechanism with limited production history outside Curve's ecosystem
- Not compatible with the reusable engine pattern (deeply coupled to AMM state)

### Insurance Fund / Socialized Loss (Various)

**Mechanism:**
- Protocol maintains insurance fund from fees
- Covers bad debt when collateral insufficient
- If fund exhausted, losses socialized across depositors or stakers

**Why explicitly excluded from RFP-014:**
- Creates governance capture vectors (who decides payouts?)
- Moral hazard (riskier position-taking if socialized backstop)
- Complicates keeper incentives
- RFP-014 uses debt auction house for shortfalls instead (market-based, not governance-based)

## The Reusable Engine Pattern

### The Coupling Problem

Traditionally, liquidation logic is tightly coupled to the CDP protocol:
- Maker's liquidation (`Dog` + `Clipper` contracts) is part of the Maker codebase
- Aave's liquidation is Aave-specific
- Liquity's Stability Pool is deeply integrated with Trove state
- Each new CDP product rebuilds similar logic

**Problems with coupling:**
- Security: Each new implementation needs fresh audit
- Time-to-market: Months rebuilding proven mechanics
- Fragmentation: Keeper ecosystem split across implementations
- Upgrade complexity: Improvements don't propagate

### The Reusable Engine Solution (RFP-014)

RFP-014 proposes a **reusable liquidation program** that any CDP product can deploy:

```
CDP Protocol A -----> Liquidation Engine Instance A
                      (configured for A's parameters)

CDP Protocol B -----> Liquidation Engine Instance B
                      (configured for B's parameters)

RFP-013 Stablecoin -> Liquidation Engine Instance S
                      (configured for RAI-style reflexive params)

Future Lending -----> Liquidation Engine Instance L
                      (configured for lending parameters)
```

**Each instance:**
- Inherits battle-tested auction logic
- Configured for host protocol's collateral types, ratios, discounts
- Isolated state per deployment (no cross-protocol contamination)
- Shared audit, shared keeper bot logic

### Integration Interface

The liquidation engine requires a minimal interface from host CDP protocols:

```
Host -> Engine:
  - notify_undercollateralized_position(position_id, collateral, debt)
  - get_collateral_price(collateral_type) -> price, confidence

Engine -> Host:
  - seize_collateral(position_id, amount) -> bool
  - cancel_debt(position_id, amount) -> bool
  - mint_surplus(amount) -> bool  // for debt auctions
```

This interface abstraction allows the same engine code to service different CDP designs:
- Reflexive stablecoins (RFP-013)
- Multi-collateral lending (future RFPs)
- Single-asset synthetic assets
- Custom CDP products

### Benefits of Reusable Design

| Benefit | Explanation |
|---------|-------------|
| **Security** | Single codebase receives focused audit attention; improvements benefit all instances |
| **Time-to-market** | New CDP products don't rebuild liquidation from scratch; configure and deploy |
| **Liquidator ecosystem** | Keepers can operate across all LEZ CDP products with same tooling; concentrated liquidity |
| **Upgrade path** | Engine improvements can be adopted by all deployed instances (or remain isolated per-instance) |
| **Composability** | Standard interface enables CDP protocol aggregation, analytics, cross-protocol features |

### Compatibility with RFP-013: Reflexive Stablecoin Specifics

The integration interface above is generic, but RFP-013's reflexive stablecoin introduces two specific requirements that implementations must account for:

**Redemption price in discount calculations.** In a fixed-peg stablecoin ($1 target), the auction discount math is straightforward: sell collateral at X% below oracle price, accept system coins at face value. In a reflexive stablecoin, the system coin's value *is* the floating redemption price. The GEB implementation handles this explicitly - the `FixedDiscountCollateralAuctionHouse` reads `lastReadRedemptionPrice` and uses it (alongside the collateral oracle price) to calculate collateral amounts returned to bidders. If the engine assumes a $1 unit of account instead of the current redemption price, the discount calculation is wrong and keepers either overpay or the protocol undersells collateral. The `get_collateral_price` interface should therefore also expose a `get_system_coin_redemption_price` call, or the host protocol must normalize prices before passing them to the engine.

**Debt auction backstop.** The debt auction mechanism ("mint protocol tokens, auction for stablecoin") assumes a mintable protocol token (FLX in Reflexer, MKR in MakerDAO). RFP-013 explicitly lists governance token design as out of scope. This creates a dependency gap: either RFP-014's debt auction requires a protocol token that doesn't yet exist, or the debt auction component needs an alternative backstop. Options include relying solely on the surplus buffer (no backstop beyond accumulated fees), accepting that unresolvable bad debt must be socialized or written off, or deferring debt auctions until a protocol token is defined. Implementations should treat the debt auction house as a configurable component that can be activated later once a protocol token exists.

### Compatibility with RFP-013 Hybrid Evolution

The [Reflexive Stablecoin appendix](./appendix-reflexive-stablecoin-ecosystem.md#design-consideration-reflexive--redemption-hybrid) discusses a potential hybrid model combining RAI's PID controller with Liquity-style direct redemptions. If this hybrid is pursued, the liquidation engine's scope expands in two ways:

**Redemption-triggered position closures.** In a hybrid, redemptions force-close the riskiest CDPs (Liquity-style). This is not a liquidation in the traditional sense (the position isn't undercollateralized), but it uses similar mechanics: collateral is seized from a position and transferred to the redeemer. The engine's integration interface would need to support a `redeem_against_position(position_id, amount)` call distinct from `seize_collateral`, since redemption has different authorization rules (anyone can redeem, not just keepers responding to undercollateralization).

**Direct liquidation as a CDP-level function.** The Liquity Stability Pool model requires pre-committed capital from depositors, creating a bootstrapping dependency unsuitable for a new ecosystem. A simpler complementary path is direct liquidation (Aave-style), where a keeper repays a position's debt and receives collateral at a configurable bonus in a single atomic transaction.

Critically, direct liquidation does not require the auction engine at all. It is a function on the CDP program itself (RFP-013), callable by any keeper:

```
Undercollateralized position detected
        |
        +---> Path C: Keeper calls RFP-013 directly
        |     keeper repays debt, receives collateral at bonus
        |     No auction engine involved. Flash-loan composable.
        |
        +---> Path A: Collateral routed to RFP-014 auction engine
              increasing-discount auction handles thin markets
              No keeper capital needed (discount attracts bidders)
```

Path C is fast and simple but requires keepers with capital (or flash-loan access). Path A is the fallback: if no keeper takes the direct liquidation within N blocks, or if the CDP protocol is configured for auction-only resolution, collateral flows to the RFP-014 auction engine where escalating discounts guarantee eventual clearing even in thin markets.

This means:
- **RFP-014** (this RFP) delivers the auction engine (Path A). This is the scope of the current deliverable.
- **RFP-013** currently scopes all liquidation to RFP-014. If the hybrid evolution is pursued, RFP-013 would add a direct `liquidate(position_id, debt_amount)` function (Path C) as a CDP-level feature, independent of the auction engine.
- The two paths are complementary, not competing. Path C handles the common case (liquid markets, capital-rich keepers). Path A handles the stress case (thin markets, volatile conditions, no immediate taker).

In this model, the liquidation engine's integration interface remains as specified: the CDP protocol notifies the engine when collateral needs to be auctioned. Direct liquidation bypasses this interface entirely since it's resolved at the CDP layer. A Stability Pool (Liquity-style Resolution B) remains a viable but more complex addition for later stages when the ecosystem has sufficient depositor liquidity.

## Auction Types Comparison

### English Auctions (Ascending)

- Two phases: raise DAI bids, then return excess collateral
- Highest bidder wins after timeout
- **Time:** Hours typical
- **Capital lockup:** Yes (bid locked until outbid)
- **Best for:** Maximizing seller revenue in liquid markets
- **Worst for:** Liquidations (time = risk; capital lockup excludes flash-loan participants)
- **Used by:** MakerDAO Liq 1.2 (deprecated), Reflexer (available but not used in production)

### Dutch Auctions (Descending)

- Start above market price, price decreases deterministically
- First acceptor wins; partial fills allowed
- **Time:** Minutes to hours depending on price curve
- **Capital lockup:** No (instant settlement)
- **Best for:** Price discovery with time pressure; flash-loan composable
- **Worst for:** Volatile collateral (clearing price may lag rapid price movements)
- **Used by:** Sky/MakerDAO Liq 2.0 (current production)

### Fixed / Increasing Discount Auctions

- Collateral offered at discount to oracle price; fixed or escalating
- Bidders purchase immediately at current discount; partial fills allowed
- **Time:** Seconds to minutes
- **Capital lockup:** No (instant settlement)
- **Best for:** Fast liquidation, predictable keeper incentives, thin markets (escalation ensures clearing)
- **Worst for:** May undervalue collateral if discount is too aggressive; relies on oracle accuracy
- **Used by:** Reflexer/RAI (increasing discount variant in production)

### Batch Auctions (Not in RFP-014)

- Aggregate multiple liquidations into periodic clearing events
- Clear at uniform price across all participants
- **Time:** Periodic batches (minutes to hours)
- **Best for:** Gas efficiency, MEV mitigation, fair price for all participants
- **Worst for:** Immediate resolution (positions wait for batch)
- **Used by:** CoW Protocol (for DEX), not widely used for liquidations

**RFP-014 selection:** Increasing discount with configurable escalation schedule. Fast, predictable, resilient in thin markets.

## Surplus and Debt Auctions

Beyond collateral liquidation, RFP-014 includes system balance mechanisms:

### Surplus Auctions

**Trigger:** Protocol accumulates excess stablecoin above threshold (from stability fees, liquidation penalties)

**Mechanism:** Auction surplus stablecoin for reference token (e.g., ETH). In Reflexer's implementation, this uses a dedicated Surplus Auction House where bidders offer increasing amounts of protocol token (FLX) for a fixed lot of surplus RAI.

**Outcome:** Protocol token burned, creating buyback pressure. Reference token or surplus distributed per protocol policy.

### Debt Auctions

**Trigger:** Collateral auction doesn't raise enough to cover debt + penalty (shortfall)

**Mechanism:** Mint protocol tokens (IOUs), auction for stablecoin to recapitalize. In Reflexer, the Debt Auction House mints FLX and auctions it for RAI to cover bad debt. In MakerDAO, MKR is minted and auctioned for DAI.

**Outcome:** Dilution of protocol token to cover system debt. Acts as lender-of-last-resort mechanism.

**RFP-014 scope:** Both surplus and debt auction houses included as configurable components.

## Why Public State for Auctions

Similar to RFP-013's public position state, RFP-014's auction state is public:

| Requirement | Public State Necessity |
|-------------|------------------------|
| **Bidder participation** | Bidders need to see current discount, remaining collateral, and bid status to make purchase decisions |
| **Keeper monitoring** | Keepers scan undercollateralized positions across protocols to trigger liquidations and participate in auctions |
| **Price verification** | Auction clearing prices serve as market signals; aggregators and oracles can reference liquidation data |
| **Composability** | Other protocols may build on auction outcomes (insurance products, liquidation analytics, automated strategies) |

**Private auctions add complexity without clear benefit:** Unlike CDP positions (where users have a legitimate privacy interest in their financial exposure), auction state has no natural privacy stakeholder. Bidders want transparency to assess opportunity. The protocol needs transparency for correct settlement. Sealed-bid auctions exist in traditional finance but introduce additional challenges (bid commitment schemes, reveal phases, griefing via non-reveal) that add protocol complexity without improving liquidation outcomes.

**Privacy approach:** Same as RFP-013 - public protocol state, private user interaction via deshield-interact-reshield. Bidders can participate from private accounts using ephemeral public accounts; auction state remains public.

## References

1. [RAI / GEB Auction Module](https://docs.reflexer.finance/system-contracts/auction-module) - Reflexer auction type documentation
2. [Fixed Discount Collateral Auction House](https://docs.reflexer.finance/system-contracts/auction-module/fixed-discount-collateral-auction-house) - GEB Docs
3. [Increasing Discount Collateral Auction House](https://docs.reflexer.finance/system-contracts/auction-module/increasing-discount-collateral-auction-house) - GEB Docs
4. [MIP 45: Liquidations 2.0](https://forum.makerdao.com/t/mip45-liquidations-2-0-liq-2-0-liquidation-system-redesign/6352) - MakerDAO Liquidation System Redesign
5. [Sky Liquidation 2.0 Developer Guide](https://github.com/sky-ecosystem/developerguides/blob/master/mcd/collateral-auction-integration-guide/collateral-auction-integration-guide.md) - Clipper contract integration
6. [Aave Liquidation Docs](https://docs.aave.com/faq/liquidations) - Direct liquidation mechanics
7. [Liquity V1 Whitepaper](https://www.liquity.org/blog/whitepaper) - Stability Pool liquidation design
8. [Liquity V2](https://www.liquity.org/) - Immutable borrowing with updated stability mechanics
9. [crvUSD LLAMMA Documentation](https://docs.curve.fi/crvUSD/amm/) - Soft liquidation via AMM
10. [DeFi Llama: Liquidations](https://defillama.com/liquidations) - Cross-protocol liquidation data
11. [ChainSecurity: MakerDAO Liquidations 2.0 Audit](https://www.chainsecurity.com/security-audit/makerdao-liquidations-2-0) - Security assessment of Dutch auction implementation

---

*This appendix was prepared to support RFP-014. For the companion stablecoin design appendix, see [Appendix: Reflexive Stablecoin Ecosystem](./appendix-reflexive-stablecoin-ecosystem.md). For clarification or additions, please use the RFP repository Discussions.*
