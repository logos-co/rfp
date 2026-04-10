# Appendix: Token Launchpad Ecosystem

This appendix surveys existing token launchpad protocols and related
infrastructure across the Ethereum and Solana ecosystems. It serves
as a reference for
[RFP-015](../RFPs/RFP-015-bonding-curve-launchpad.md) and
[RFP-016](../RFPs/RFP-016-lbp-launchpad.md), providing context on
current market participants, their mechanisms, and design trade-offs.

## Protocols Considered

Protocols are ordered by cumulative scale (largest first) and this
order is maintained throughout the document.

| Protocol         | Ecosystem              | Mechanism                                                  | Website                                           |
| ---------------- | ---------------------- | ---------------------------------------------------------- | ------------------------------------------------- |
| Pump.fun         | Solana                 | Constant-product bonding curve                             | [pump.fun](https://pump.fun/)                     |
| Fjord Foundry    | Ethereum (multi-chain) | LBP (Balancer-based)                                       | [fjordfoundry.com](https://www.fjordfoundry.com/) |
| Metaplex Genesis | Solana                 | Launch Pool (proportional), presale, uniform price auction | [metaplex.com](https://www.metaplex.com/)         |
| DAO Maker        | Ethereum               | SHO (Strong Holder Offering), DYCO                         | [daomaker.com](https://daomaker.com/)             |
| Flaunch          | Base (Ethereum L2)     | Bonding curve via Uniswap V4 hooks                         | [flaunch.gg](https://flaunch.gg/)                 |
| Polkastarter     | Ethereum, BNB Chain    | Fixed-price IDO with lottery allocation                    | [polkastarter.com](https://polkastarter.com/)     |
| DAOs.fun         | Solana                 | Fixed-price fair launch for investment DAOs                | [daos.fun](https://daos.fun/)                     |

Balancer is the protocol layer that provides the weighted pool
primitive underlying the LBP mechanism. Fjord Foundry (originally
Copper, rebranded after merger) is the primary launchpad platform
built on Balancer LBPs. LBPs can also be deployed directly on
Balancer via smart contract interaction; Fjord provides the no-code
interface used by most projects. The LBP mechanism has been in
continuous production use since 2021, with 717 LBPs launched,
106,762 cumulative participants, and over $1.1B in funds raised
across the Fjord Foundry platform [5][6].

## Scale and Traction

The tables below use cumulative volume (trading volume or total
funds raised, depending on availability) and protocol revenue as
scale metrics.

DAOs.fun is omitted from this section due to insufficient
quantitative data.

### Cumulative scale

Protocols ranked by cumulative volume (trading volume or total
funds raised, depending on availability).

| Protocol | Cumulative Volume | Cumulative Revenue | Tokens Launched | Data as of |
|---|---|---|---|---|
| Pump.fun | >$150B (trading volume) [1] | $818M+ (protocol fees) [1][2] | 11.7M (H1 2025) [4] | Early 2026 |
| Fjord Foundry | ~$1.5B (swap volume) [22] | ~$55M (est.) [a] | 717 LBPs; 106,762 participants [6] | Sep 2025 |
| Metaplex Genesis | ~$55-70M (est. funds raised) [d] | $36M+ (all Metaplex products) [7] | 20M+ fungible tokens [7] | Nov 2025 |
| DAO Maker | $90M+ (total raised) [8] | ~$4.5M (est.) [b] | N/A | Undated |
| Flaunch | N/A | $3.1M [9] | N/A | Q1 2026 |
| Polkastarter | $46M+ (total raised) [10] | ~$460K (est.) [c] | 105+ projects [10] | Q1 2022 |

\[a] Estimated as 5% fee [11] applied to $1.1B cumulative raised [5].
\[b] Estimated as 5% fee [12] applied to $90M+ cumulative raised [8].
\[c] Estimated as 1% fee [13] applied to $46M+ cumulative raised [10].
Figure is from Q1 2022; no updated aggregate has been published.
\[d] Estimated from Genesis protocol revenue ($1.1M, Jul to Nov 2025)
divided by 2% deposit fee [16]. Genesis launched in July 2025 (alpha);
no 2024 data exists.

### Yearly breakdown

Where per-year data exists. Blank cells indicate no data available
for that period.

| Protocol | Metric | 2024 | 2025 | 2026 (YTD) |
|---|---|---|---|---|
| Pump.fun | Trading volume (est.) | ~$32B [e] | ~$58B [e] | ~$8B (Q1) [e] |
| Pump.fun | Protocol revenue | ~$321M [14] | ~$583M [14] | ~$82M (Q1) [14] |
| Metaplex Genesis | Funds raised (est.) | N/A (launched Jul 2025) | ~$55-70M [d] | |
| Metaplex | Protocol revenue (all products) | ~$12-15M (est.) [7] | ~$27-30M (est.) [7] | |
| Flaunch | Protocol revenue | | ~$3.1M [9] | $100.6K (Q1) [9] |
| Fjord Foundry | Cumulative raised (all sales) | >$1.1B (since 2021; no per-year split) [5] | | |
| DAO Maker | Total raised | $90M+ (cumulative; no per-year split) [8] | | |
| Polkastarter | Total raised | $46M+ (cumulative as of Q1 2022; stale) [10] | | |

\[e] Implied from quarterly protocol fees divided by 1% fee rate [14].
Actual trading volume may differ due to fee-exempt transactions or
variable fee tiers.

**Caveats:**

- Metric types differ across protocols (trading volume vs. funds
  raised vs. protocol revenue). Cross-category comparisons should
  account for this. Pump.fun's trading volume includes secondary DEX
  trading on Pumpswap, not only bonding curve sales.
- Fjord Foundry, DAO Maker, and Polkastarter do not publish per-year
  breakdowns.
- Polkastarter's aggregate figure dates from Q1 2022; current
  cumulative total is likely higher but unpublished.
- Metaplex Genesis launchpad revenue is a small fraction of total
  Metaplex protocol revenue; most Metaplex revenue comes from Token
  Metadata fees charged to platforms like Pump.fun [7].

## Refund Mechanisms

Most crypto-native launchpads treat purchases as final: once a buyer
executes a swap or deposit, there is no platform-level refund. Notable
exceptions exist: Fjord Foundry's tiered sales (Seed, Private, Public
rounds) support a minimum cap with automatic refunds if the target is
not met [33]; DAO Maker's DYCO product offers a 16-month post-purchase
refund window [15]; and DAOs.fun allows redemption during the
fundraising phase with a 10% penalty [21]. Regulatory frameworks are moving in
the opposite direction: EU MiCA mandates a 14-day cooling-off period
for white paper offerings [19], and US Reg CF requires a 48-hour
cancellation window plus a mandatory minimum raise threshold [20].

### Summary

| Protocol | Post-purchase refund | Minimum raise (soft cap) | Cancellation during sale |
|---|---|---|---|
| Pump.fun | No | No | No |
| Fjord Foundry (LBP) | No [11] | No [11] | Sell back at market price (buy+sell mode only) [11] |
| Fjord Foundry (tiered sales) | No | Yes (minimum cap; auto-refund if unmet) [33] | No |
| Metaplex Genesis | No [d] | Yes (Launch Pool only) [16] | Yes (Launch Pool: withdraw before claim window; 2% fee) [16] |
| DAO Maker (DYCO) | Yes: 16 months, USDC-backed, burn on refund [15] | No | Yes: unconditional within window [15] |
| DAO Maker (SHO) | No [12] | No | No |
| Flaunch | No [17] | No | No |
| Polkastarter | No [18] | No | No |
| DAOs.fun | No (AMM exit + mandatory expiry distribution) [21] | Yes (hard cap; refund if unmet) [21] | Yes: redeem during fundraising (10% penalty) [21] |

\[d] Metaplex Genesis Launch Pool allows withdrawal during the deposit
window (before tokens are distributed). If the funding goal is not
met, depositors receive a full refund [16]. Once the claim window
opens and tokens are distributed, purchases are final.

### Per-protocol detail

**Pump.fun.** No refund mechanism. Bonding curve purchases are
irreversible AMM swaps. Once a token graduates to Pumpswap, it trades
on the open market with no platform-level buyer protection.

**Fjord Foundry.** Refund properties differ by sale type. For LBPs:
no refund mechanism [11]. LBP purchases are AMM swaps and are final
once executed. In buy+sell mode, participants can sell tokens back
into the pool during the sale at the current market price (not at
their purchase price). After the LBP ends, tokens are distributed
and there is no refund path. No minimum raise threshold exists; the
sale runs regardless of demand. For tiered sales (Seed, Private,
Public rounds): Fjord supports a minimum cap setting; if the target
is not met, contributions are automatically refunded [33]. This
refund feature does not apply to LBPs.

**Metaplex Genesis.** The Launch Pool mechanism provides partial buyer
protection: depositors can withdraw during the deposit window (subject
to a 2% fee), and if the funding goal is not met, all deposits are
refunded [16]. However, once the claim window opens and tokens are
distributed, there is no refund. Presale and Uniform Price Auction
modes have no refund mechanism.

**DAO Maker (DYCO).** The Dynamic Coin Offering is the only
crypto-native mechanism with a post-purchase performance refund [15].
All tokens sold are backed 1:1 by USDC in a smart contract for 16
months after the Token Generation Event. Participants can return
tokens at any time within this window to receive the original USDC
price; returned tokens are burned, reducing circulating supply. If the
token price falls below the refund price on the secondary market,
arbitrageurs can buy cheaply and refund at the higher price, creating
a reliable price floor. DYCO is a specialized product; most DAO Maker
sales use the SHO model, which has no refund mechanism.

**Flaunch.** No refund mechanism [17]. The 30-minute fixed-price fair
launch phase is one-directional (buyers cannot sell during this
window). After graduation to standard AMM pricing, tokens trade freely
with no platform-level refund.

**Polkastarter.** No refund mechanism [18]. Fixed-price IDO purchases
are atomic swaps and are final once executed. Participants who are
allowlisted but have not yet purchased can choose not to participate,
but there is no post-purchase cancellation.

**DAOs.fun.** Structured refund mechanism with three phases [21].
During the 7-day fundraising window, contributors can redeem their SOL
at any time, subject to a 10% early redemption penalty. If the
fundraising hard cap is not reached, contributors receive a full
refund. After a successful raise, there is no direct refund: token
holders can sell on the virtual AMM at market price, or wait for the
mandatory fund expiration (3 months to 1 year, set by the creator at
launch), at which point remaining assets are distributed
proportionally and token holders can burn tokens to redeem underlying
SOL.

### Regulatory context

Two regulatory frameworks mandate buyer refund rights for token sales:

- **EU MiCA** (Markets in Crypto-Assets Regulation): requires a 14-day
  right of withdrawal for crypto-asset purchases made under a white
  paper offering [19]. This applies to any platform operating in the
  EU.
- **US Regulation CF** (crowdfunding): requires a 48-hour cancellation
  window before the offering deadline, a mandatory minimum raise
  threshold (all funds returned if the target is not met), and
  automatic cancellation if material changes are made to the offering
  terms (investors must reconfirm within 5 business days) [20].

Neither framework applies directly to permissionless, non-custodial
on-chain launchpads. However, they indicate a regulatory direction
toward mandatory cooling-off periods for token sales.

## Allowlist & Access Control

Launchpad access models range from fully permissionless (any wallet
can participate) to fully gated (mandatory KYC and token staking).
Among the crypto-native launchpads surveyed, the majority default to
open access; gating is either absent or optional.

### Summary

| Protocol | Default access | Gating method | Enforcement |
|---|---|---|---|
| Pump.fun | Open (permissionless) | None | N/A |
| Fjord Foundry | Open; optional project whitelist | CSV whitelist upload by creator [11] | Off-chain (platform UI) |
| Metaplex Genesis | Configurable per-launch | Wallet allowlist, optional KYC, geo-blocking [16] | On-chain (smart contract) |
| DAO Maker (SHO) | $DAO stakers + KYC | Token staking (min 2,000 $DAO) + mandatory platform KYC [12] | Hybrid (on-chain staking, off-chain KYC) |
| Flaunch | Open (permissionless) | None | N/A |
| Polkastarter | $POLS stakers + per-project KYC | Token staking (min 1,000 POLS) + lottery + per-project KYC [10] | Hybrid (on-chain staking, off-chain KYC) |
| DAOs.fun | Creators: invitation-only; Buyers: open | Invite codes for creators; none for buyers [21] | Platform-level |

### Per-protocol detail

**Pump.fun.** Fully permissionless [1]. No allowlist, KYC, or access
restrictions. Any wallet can create or buy tokens.

**Fjord Foundry.** Permissionless by default for LBPs [11]. Project
creators can optionally upload a CSV whitelist to restrict
participation, enforced at the platform UI level (not on-chain).
The $FJO token is not required for participation. LBPs also provide
structural anti-bot protection via high starting price.

**Metaplex Genesis.** Most configurable access control among the
platforms studied [16]. Creators can choose open access, wallet
allowlist, or KYC-gated launches, all enforced on-chain at the
smart contract level. Supports per-wallet deposit limits and
geo-blocking. The Launch Pool's proportional distribution mechanism
provides some natural Sybil resistance: allocation is proportional
to deposit size, so splitting across wallets yields no advantage.

**DAO Maker.** Mandatory $DAO staking (minimum 2,000 $DAO) plus
platform-level KYC [12]. A seven-tier system based on staked amount
(250 to 100,000 $DAO) assigns non-linear bonus multipliers (DAO
Power). Higher tiers receive disproportionate allocation advantages.
SHO priority rounds serve high-tier holders first, then remaining
allocation opens as FCFS.

**Flaunch.** Fully permissionless [17]. Built on Uniswap V4 hooks
with no access restrictions. No allowlist, KYC, or token-gating.

**Polkastarter.** $POLS staking (minimum 1,000 POLS Power) with a
7-day minimum hold requirement [10]. A lottery system weighted by
POLS Power allocates spots. KYC is delegated to individual projects
(not centralized by the platform). Guaranteed allocation is
available only at the highest tier (50,000+ POLS).

**DAOs.fun.** Creator access is invitation-only (invite codes from
the team) [21]. Buyer participation is open once a DAO is live,
with no KYC or staking requirement. The 10% early redemption penalty
serves as an economic Sybil deterrent during the fundraising window.

### Per-buyer allocation limits

Per-wallet or per-buyer allocation limits are available on platforms
that use fixed-price or proportional distribution mechanisms:
Metaplex Genesis supports per-wallet deposit limits enforced on-chain
[16]; Fjord Foundry tiered sales (Seed, Private, Public rounds)
support per-user min/max allocation per tier [33]; DAO Maker and
Polkastarter enforce allocation limits through their staking tier
systems [12][10].

No bonding curve platform (Pump.fun, Flaunch, Meteora DBC) implements
per-buyer limits. Bonding curves are open AMM pools where any wallet
can submit unlimited buy transactions; the mechanism does not track
cumulative spend per address.

## Fee Structures

Fee models vary widely across launchpad protocols, from zero-fee
permissionless deployment to enterprise pricing tiers exceeding
$50,000. The table below summarises fee structures for the seven
crypto-native launchpads surveyed in this appendix.

| Protocol | Issuer Fee | Buyer Fee | Setup Cost | Staking Required |
|---|---|---|---|---|
| Pump.fun | 0% | 1% (protocol fee on trades) [1][2] | Free | None |
| Fjord Foundry | 5% of collateral raised [11] | 2% swap fee [11][34] | Free | None |
| Metaplex Genesis | 0% (fee on deposits) [16] | 2% on deposits [16] | Free | None |
| DAO Maker | 5% of tokens [12] | 5% (DAO SHO) / 30% (Public SHO) [12] | Undisclosed | 2,000–100,000 $DAO ($28–$1,400+) [12] |
| Flaunch | Undisclosed | Swap fee (Uniswap V4) [17] | Free (gas only) | None |
| Polkastarter | ~1% of raised [13] | Network gas only [10] | Undisclosed | 1,000–50,000 POLS ($300–$25,000) [10] |
| DAOs.fun | 10% early redemption penalty [21] | None during fundraise; AMM fee post-launch [21] | Free | None |

### Per-protocol detail

**Pump.fun.** Zero issuer fee for token creation; the protocol
charges a 1% fee on all trades executed through its bonding curve and
Pumpswap DEX [1][2]. This buyer-side-only model minimises friction
for token creators and has been a primary driver of Pump.fun's scale
(11.7M tokens launched in H1 2025). The fee is collected per
transaction at swap time.

**Fjord Foundry.** Charges issuers 5% of collateral raised at sale
close [11]. Buyers pay a 2% swap fee on each transaction [34]. No
upfront setup cost and no native token requirement ($FJO is not
needed for participation). The 5% issuer fee is competitive for a
managed platform that provides a no-code UI and marketing
distribution.

**Metaplex Genesis.** Zero issuer fee; the protocol charges a 2%
fee on buyer deposits [16]. Setup is free (gas costs only on
Solana). No native token requirement. The depositor-pays model is
unusual: it makes launches maximally cheap for issuers while
shifting costs to participants.

**DAO Maker.** Charges issuers 5% of tokens allocated to the sale
[12]. Buyer fees depend on tier: DAO SHO participants pay 5% of
allocated tokens; Public SHO participants pay 30% unless they hold
at least 2,000 $DAO [12]. Setup costs are undisclosed and negotiated
per project. The staking requirement (2,000 to 100,000 $DAO) is the
primary access gate and functions as an additional implicit cost.

**Flaunch.** Built on Uniswap V4 hooks with no disclosed
platform-level fee beyond Uniswap swap fees [17]. Token creation
costs only gas. No staking requirement. Revenue is generated through
swap fee accumulation in the hook mechanism.

**Polkastarter.** Charges issuers approximately 1% of funds raised
[13]. Buyers pay only network gas. However, participation requires
staking 1,000 to 50,000 POLS tokens ($300 to $25,000), which
functions as a significant implicit cost [10]. Guaranteed allocation
requires 50,000+ POLS.

**DAOs.fun.** No traditional issuer or buyer fee during the
fundraise window [21]. The 10% early redemption penalty during the
7-day fundraise serves as both a Sybil deterrent and a revenue
mechanism. After a successful raise, tokens trade on a virtual AMM
with standard swap fees.

### Design implications

Across the seven protocols, the sustainable fee range clusters
around 1–2% per transaction or 5% at close. Pump.fun (1% buyer
fee) and Metaplex Genesis (2% deposit fee) represent the low end;
Fjord Foundry (5% issuer fee) represents the upper bound for
crypto-native platforms. Fees above 5% are found only in
enterprise-regulated platforms (Republic, Securitize) serving
institutional markets [23]. Staking requirements (DAO Maker,
Polkastarter) function as hidden fees: they impose a capital lockup
cost of $300 to $25,000+ and create plutocratic access barriers
where participation rights scale with token holdings rather than
genuine interest in the project.

### Onchain fee enforcement

Among the protocols surveyed, fee enforcement falls into three
categories: per-swap onchain fees (deducted atomically in the swap
transaction), at-close onchain fees (deducted when the creator
withdraws proceeds), and platform-level fees (collected off-chain
or via UI restrictions).

#### Per-swap onchain fees

Pump.fun, Balancer (underlying Fjord Foundry LBPs), Flaunch
(Uniswap V4 hooks), and Metaplex Genesis all enforce fees onchain
at swap time.

In Balancer v2 weighted pools (the contract underlying all Fjord
LBPs), the swap fee is deducted from the **input amount before**
the weighted math formula runs [29][34]. The adjusted input is:

```
A_in = A_sent × (1 - swap_fee_percentage)
```

This `A_in` is then substituted into the standard weighted pool
out-given-in formula. The fee portion stays in the pool (increasing
LP value), minus the Balancer protocol fee share. In
`LiquidityBootstrappingPool.sol`, the `_onSwapMinimal` function
handles this explicitly [34]:

```solidity
// For GIVEN_IN (exact input):
request.amount = request.amount.mulDown(
    getSwapFeePercentage().complement()
);
```

Both directions round against the trader (in favour of the pool).
The LBP contract inherits standard weighted pool fee handling with
no LBP-specific fee logic; the only LBP-specific behaviour is
time-dependent weight interpolation [34].

Pump.fun uses an equivalent model: a 1% fee deducted per trade,
collected atomically in the same transaction [1][2].

#### Balancer protocol fee

Balancer v2 imposes a protocol-level fee that is a **percentage of
the swap fee**, not of the swap amount [35]. The current rate is
50% of collected swap fees (set by Balancer governance via BIP-371,
August 2023) [35]. For example, if a pool charges a 2% swap fee,
1% accrues to LPs (stays in pool) and 1% goes to the Balancer
`ProtocolFeesCollector` contract. From the swapper's perspective,
the total cost is the same regardless of the protocol/LP split.

Balancer v3 retains the 50% swap fee share but reduced the yield
fee share from 50% to 10% to encourage adoption of boosted
pools [36]. v3 also introduced a Pool Creator Fee, allowing pool
deployers to earn a share of fees from their pools [37].

#### At-close onchain fees

Fjord Foundry collects a 5% platform fee on collateral raised at
sale close [11]. Fjord's wrapper contract (historically the Copper
Launch proxy) acts as the Balancer pool controller; only Fjord's
contract can call `exitPool` on the underlying Balancer pool,
allowing it to retain 5% of collateral before forwarding proceeds
to the creator [11]. The creator interacts with Fjord's contract,
not Balancer directly, and cannot bypass the fee.

This means Fjord LBPs have two distinct onchain fee layers: a 2%
per-swap fee (enforced by Balancer's pool contract, split 50/50
between LPs and protocol) and a 5% at-close fee (enforced by
Fjord's wrapper contract) [11][34].

#### Fee enforcement comparison

| Protocol | Fee Type | Enforcement | When Collected | Deduction Point |
|---|---|---|---|---|
| Pump.fun | 1% per trade | Onchain (Solana program) | Per-swap | From swap amount |
| Fjord/Balancer | 2% swap fee | Onchain (Balancer pool contract) | Per-swap | From input before formula |
| Fjord | 5% platform fee | Onchain (Fjord wrapper contract) | At-close | From collateral proceeds |
| Metaplex Genesis | 2% deposit fee | Onchain (Solana program) | Per-deposit | From deposit amount |
| Flaunch | Swap fee | Onchain (Uniswap V4 hook) | Per-swap | Via hook mechanism |
| Polkastarter | ~1% of raised | Off-chain / undisclosed | At-close | From proceeds |

## Sale Lifecycle and Close Mechanics

Sale lifecycle controls span a wide spectrum, from fully
centralised platform management to immutable on-chain parameters
with no admin override. The table below summarises close triggers,
pause capabilities, and post-close behaviour for the surveyed
protocols.

| Protocol | Auto-Close Triggers | Emergency Pause | Post-Close Behaviour | Enforcement |
|---|---|---|---|---|
| Pump.fun | Supply target (bonding curve graduation) [1] | No | Auto-migration to Pumpswap DEX [1] | On-chain (Solana program) |
| Fjord Foundry | Time expiry; optional hardcap [11] | Yes (creator pauses swaps) [11] | Manual claim; collateral to creator minus 5% fee [11] | Hybrid (on-chain pause + platform UI) |
| Metaplex Genesis | Deposit window close (time-based) [16] | Not documented | Auto-distribute proportional (Launch Pool); FCFS (Presale); refund if goal not met [16] | On-chain (Solana program) |
| DAO Maker | Time window expiry [12] | Platform-level only [12] | Manual claim; vesting per project [12] | Hybrid (off-chain allocation + on-chain claim) |
| Flaunch | 30-minute fixed-price window expiry [17] | Not documented | Auto-graduation to Uniswap V4 AMM [17] | On-chain (Uniswap V4 hook) |
| Polkastarter | Hardcap reached; time expiry [10] | Not documented | Atomic swap (immediate token receipt) [10] | On-chain (swap) + off-chain (allowlist) |
| DAOs.fun | Fundraise hard cap; 7-day window expiry [21] | Not documented | Fund manager deploys capital; mandatory expiry distribution [21] | On-chain (Solana program) |

### Centralisation spectrum

The protocols studied fall along a spectrum from fully
decentralised to fully centralised lifecycle control [24]:

- **Permissionless immutable** (Pump.fun, Metaplex Genesis):
  parameters locked at creation; no admin override possible. The
  sale runs to completion or fails according to on-chain rules.
- **Creator-controlled** (Fjord Foundry, Flaunch): smart contract
  gives the sale creator pause and resume capability; the platform
  adds a UI layer but does not override on-chain state.
- **Platform-controlled** (DAO Maker, Polkastarter): the platform
  manages the full lifecycle with off-chain allocation and
  on-chain claim. Issuers have limited on-chain autonomy.

DAOs.fun occupies a middle position: fundraise parameters are set
at creation and enforced on-chain, but post-raise fund deployment
is delegated to a human fund manager with discretionary authority.

### Per-protocol detail

**Pump.fun.** The bonding curve closes automatically when the supply
target is reached (graduation threshold), triggering migration of
liquidity to Pumpswap [1]. There is no pause, no manual close, no
time-based expiry, and no admin override. Post-graduation, the token
trades on the open DEX market. Curves that never reach the graduation
threshold remain as dormant on-chain accounts indefinitely. The
graduation rate is very low: approximately 1.4% historically [30],
dropping to 0.7% to 0.8% in mid-2025 [31]. This means over 98% of
launched tokens become zombie sales with no close mechanism and no
way for the creator to reclaim deposited tokens. No bonding curve
platform in the ecosystem (Pump.fun, Meteora DBC, Flaunch, Raydium
LaunchLab) implements an optional time-based close to address this
[32].

**Fjord Foundry.** The sale creator can pause swaps during an LBP
(reversible) or cancel a Fixed Price Sale before it starts [11].
LBPs close when the time window expires or when an optional hardcap
is reached. After close, participants manually claim tokens through
the platform; the creator receives collateral minus the 5% platform
fee.

**Metaplex Genesis.** Deposit windows close on a time basis [16].
The Launch Pool distributes tokens proportionally to all depositors;
if the funding goal is not met, depositors receive a full refund.
No documented emergency pause or admin override exists at the
protocol level.

**DAO Maker.** Lifecycle is fully platform-controlled [12]. Sales
open and close on a schedule managed by the DAO Maker team. Buyers
claim tokens after the sale; vesting schedules are project-specific.
The DYCO variant adds a 16-month refund window backed by USDC
reserves [15].

**DAOs.fun.** The 7-day fundraise window closes automatically on
time expiry or when the hard cap is reached [21]. If the cap is
not met, contributors receive a full refund. After a successful
raise, the fund manager deploys capital; at the mandatory expiry
date (3 months to 1 year), remaining assets are distributed
proportionally to token holders.

## LBP Pricing Mechanism

The Liquidity Bootstrapping Pool pricing mechanism originates from
Balancer's weighted pool primitive. Fjord Foundry uses the same
underlying formulas. The core components are weight interpolation,
a spot price function, and a swap output function.

### Weight interpolation

At any point in time `t`, the token weight is linearly interpolated
between its start and end values:

```
w_token(t) = w_start + (w_end - w_start) × (t - t_start) / (t_end - t_start)
w_collateral(t) = 1 - w_token(t)
```

Where `w_start` is the initial token weight (e.g., 0.99), `w_end`
is the final token weight (e.g., 0.01), `t_start` and `t_end` are
the sale start and end timestamps, and `t` is the current block
timestamp. The function is linear and continuous over the sale
duration [29].

### Spot price

At any reserve state, the spot price of the project token in terms
of collateral is:

```
price = (reserve_collateral × w_token) / (reserve_token × w_collateral)
```

As weights shift (`w_token` decreases, `w_collateral` increases),
the price naturally falls even if reserves are unchanged. This is
the mechanism behind the LBP's declining price curve [29].

### Swap output (buy formula)

Given a collateral input `C_in`, the token output is computed using
the Balancer weighted pool swap formula:

```
tokens_out = reserve_token × (1 - (reserve_collateral / (reserve_collateral + C_in)) ^ (w_collateral / w_token))
```

After each buy, `reserve_token` decreases by `tokens_out` and
`reserve_collateral` increases by `C_in` [29].

### Lazy weight computation

Modern Balancer (v2/v3) and Fjord Foundry use lazy computation:
every swap triggers weight recalculation based on the block
timestamp and the stored weight schedule. No external `pokeWeights`
call is required for correct pricing; the on-chain price always
reflects the current weight at transaction time [29]. Older Balancer
v1 implementations required explicit poke transactions to advance
the weight schedule, which created stale-weight arbitrage
opportunities when pokes were delayed.

### Pause and weight progression

When a sale creator pauses swaps (available on Fjord Foundry and
Balancer pools created with the `canPauseSwapping` permission),
the weight schedule continues to advance because weights are
time-based, not trade-based [11][29]. This prevents a creator from
pausing at a weight that is artificially favourable and resuming
after detecting large buy orders.

## Allowlist Privacy in Shielded Execution Environments

On a transparent chain, an allowlist directly degrades participant
privacy: if the allowlist contains 50 addresses and a purchase is
observed on-chain, the observer knows the buyer is one of 50.
The anonymity set equals the allowlist size, potentially reducing
k-anonymity from millions of active wallets to a small, enumerable
group [25][26].

In a shielded execution environment such as LEZ, this framing is
misleading. LEZ private account activity is shielded by
construction: an observer sees that some private account performed
an action, but cannot determine which account or what action. The
relevant anonymity set is therefore all private accounts in the
zone, not the allowlist [25]. The allowlist determines eligibility,
not visibility.

### Privacy-permissioning tradeoff

The privacy properties of an allowlist depend on both the allowlist
design and the execution environment.

| Allowlist Design | Observer can enumerate allowlist? | Observer can link buyer to entry? | Anonymity set |
|---|---|---|---|
| Public address list on transparent chain | Yes | Yes (via transaction origin) | Size of allowlist |
| Public address list on ZK private zone (LEZ) | Yes | No (shielded execution) | All private accounts in zone |
| Private allowlist with ZK set membership | No | No | All accounts that could prove membership |
| No allowlist on transparent chain | N/A | Yes (public transaction) | All active wallets |
| No allowlist on ZK private zone (LEZ) | N/A | No | All private accounts in zone |

In LEZ, both a public address allowlist and a private ZK
allowlist achieve the same practical privacy level as no allowlist
at all, because the execution environment's shielding prevents
linking buyers to allowlist entries. The allowlist restricts
eligibility without degrading privacy [25].

### ZK set membership proofs

The recommended approach for allowlist enforcement in a privacy-
preserving context is a ZK set membership proof based on a Merkle
tree commitment [25][27][28]:

1. The sale creator builds a Merkle tree over hashed allowlist
   entries and publishes only the Merkle root on-chain.
2. A participant proves knowledge of a leaf and a Merkle path such
   that `MerkleVerify(root, leaf, path) = true`, without revealing
   which leaf they hold.
3. A nullifier (derived from the leaf) prevents the same allowlist
   entry from participating twice, providing Sybil resistance
   without revealing identity.

This construction is a standard cryptographic primitive, used in
production by Tornado Cash (withdrawal proofs), Semaphore (anonymous
group signalling), and proposed for privacy-preserving KYC in
zk-creds [27]. For implementations on LEZ, this approach avoids
publishing the allowlist on-chain and preserves the zone-wide
anonymity set regardless of allowlist size.

## References

1. CoinDesk, "Most Influential: Pump.fun," Dec 2025.
   https://www.coindesk.com/coindesk-news/2025/12/10/most-influential-pump-fun
2. Tokenomics.com, "Pump.fun Tokenomics: How Pump Distributes $45M
   Monthly to Holders," Feb 2026.
   https://tokenomics.com/articles/pumpfun-tokenomics-how-pump-distributes-45m-monthly-to-holders
3. CoinLaunch, "Pump.fun Overview," Jul 2025.
   https://coinlaunch.space/projects/pumpfun/
4. Metaplex Foundation, "H1 2025 Recap."
   https://www.metaplex.foundation/blog/articles/metaplex-1h-25-recap
5. Fjord Foundry homepage (cumulative raised figure), accessed Apr
   2026. https://www.fjordfoundry.com/
6. Bitbond, "Fjord Foundry Review," Sep 2025.
7. Messari, "State of Metaplex" quarterly reports; Metaplex Foundation
   blog monthly roundups, 2023-2025.
   https://www.metaplex.foundation/blog/
8. CoinMarketCap, "What is DAO Maker."
   https://coinmarketcap.com/alexandria/article/what-is-dao-maker
9. DefiLlama, "Flaunch Protocol," accessed Apr 2026.
   https://defillama.com/protocol/flaunch
10. Polkastarter blog, "How to Participate in a Polkastarter IDO" and
    "All You Need to Know About POLS Power."
    https://blog.polkastarter.com/how-to-participate-in-a-polkastarter-ido/
11. Fjord Foundry, "LBP FAQ."
    https://help.fjordfoundry.com/fjord-foundry-docs/for-sale-creators/faqs-creators/lbp-faq
12. DAO Maker, GitBook documentation.
    https://dao-maker-1.gitbook.io/dao-maker
13. Polkastarter fee details partially inferred from CryptoRank and
    platform documentation; exact percentage not prominently published.
14. Yahoo Finance, "Pump.fun Quarterly Revenue," Dec 2025; AInvest,
    Jan 2026.
15. DAO Maker, "DYCO: Dynamic Coin Offering."
    https://learn.daomaker.com/dyco
16. Metaplex, "Genesis: Launch Pool" developer documentation.
    https://developers.metaplex.com/smart-contracts/genesis
17. Flaunch documentation and Bankless, "How to Launch Better
    Memecoins on Flaunch."
    https://docs.flaunch.gg/ ;
    https://www.bankless.com/read/how-to-launch-better-memecoins-on-flaunch
18. Polkastarter, "How to Participate in a Polkastarter IDO."
    https://blog.polkastarter.com/how-to-participate-in-a-polkastarter-ido/
19. EU Markets in Crypto-Assets Regulation (MiCA), right of withdrawal
    provisions; Shoosmiths, "Consumer Protection for Crypto Assets."
    https://www.shoosmiths.com/insights/articles/heating-up-consumer-protection-for-crypto-assets
20. US Regulation CF, 17 CFR §227.
    https://www.ecfr.gov/current/title-17/chapter-II/part-227
21. ChainCatcher, "daos.fun: The next liquidity black hole?" Oct 2024;
    Bitget News, "daos.fun: The next liquidity black hole?" Oct 2024;
    Blocmates, "Daos.fun is Shaking Up the Crypto Memecoin Space,"
    Jan 2025.
    https://www.chaincatcher.com/en/article/2149146
22. RootData, "Fjord Foundry" project page (swap volume figure),
    accessed Apr 2026.
23. Fee Structure Comparison research note (internal), compiled from
    official documentation for each platform, Apr 2026. Sources:
    DAO Maker GitBook, Fjord Foundry LBP FAQ, Metaplex Genesis docs,
    Balancer protocol fee docs, Republic issuer pricing, Securitize
    fee schedule.
24. Sale Lifecycle & Close Mechanics research note (internal),
    compiled from official documentation and smart contract
    repositories, Apr 2026. Sources: Balancer CRP source code,
    Fjord Foundry LBP FAQ, Metaplex Genesis docs, Securitize DS
    Protocol, Republic early close help, TokenSoft token contracts.
25. Allowlist Privacy and K-Anonymity Analysis research note
    (internal), Apr 2026.
26. zk-X509: Practical Anonymous Credentials for X.509 Certificates,
    arXiv:2603.25190, Mar 2026 (Merkle anonymity set analysis).
    https://arxiv.org/html/2603.25190v1
27. Choudhuri et al., "zk-creds: Flexible Anonymous Credentials
    from zkSNARKs," IEEE S&P 2023.
    https://obj.umiacs.umd.edu/ieeesp23/zk-creds.pdf
28. Privacy-Preserving Smart Contracts using zkSNARKs,
    arXiv:2501.03391, Jan 2025.
    https://arxiv.org/pdf/2501.03391
29. Balancer, "Weighted Math" and "Liquidity Bootstrapping FAQ,"
    Balancer v2 documentation.
    https://docs.balancer.fi/concepts/explore-available-balancer-pools/weighted-pool/weighted-math.html ;
    https://balancer.gitbook.io/balancer/smart-contracts/smart-pools/liquidity-bootstrapping-faq
30. Bitget, "Pump.fun statistics," citing Dune Analytics, Aug 2024.
    https://www.bitget.com/news/detail/12560604442705
31. The Defiant, "Pump.fun graduation rate drops," mid-2025.
    https://www.thedefiant.io/news/defi/pump-fun-graduation-rate-drops
32. Bonding Curve Time-Based Close Mechanisms research note
    (internal), compiled from Pump.fun, Meteora DBC, Flaunch,
    Raydium LaunchLab documentation, Apr 2026.
33. Fjord Foundry, "Seed, Private and Public Rounds: Can I refund
    participants if my sale doesn't hit the minimum cap?"
    https://help.fjordfoundry.com/fjord-foundry-docs/for-sale-creators/seed-private-and-public-rounds#can-i-refund-participants-if-my-sale-doesnt-hit-the-minimum-cap
34. Balancer v2, `LiquidityBootstrappingPool.sol` source code
    (swap fee deduction in `_onSwapMinimal`); Bitbond Fjord Foundry
    review (2% swap fee); Fjord Foundry LBP FAQ (fee structure).
    https://github.com/balancer/balancer-v2-monorepo/blob/master/pkg/pool-weighted/contracts/lbp/LiquidityBootstrappingPool.sol ;
    https://www.bitbond.com/resources/fjord-foundry-launchpad-review/ ;
    https://help.fjordfoundry.com/fjord-foundry-docs/for-sale-creators/faqs-creators/lbp-faq
35. Balancer v2, "Protocol Fees" and "Governable Protocol Fees"
    documentation (protocol fee is percentage of swap fee, 50% rate
    per BIP-371).
    https://docs.balancer.fi/concepts/governance/protocol-fees.html ;
    https://balancer.gitbook.io/balancer-v2/ecosystem/governance/governable-protocol-fees
36. Balancer v3, "Protocol Fee Model" documentation (50% swap fee
    share, 10% yield fee share).
    https://docs.balancer.fi/concepts/protocol-fee-model/protocol-fee-model.html
37. Balancer v3, "Pool Creator Fee" documentation.
    https://docs.balancer.fi/concepts/core-concepts/pool-creator-fee.html
