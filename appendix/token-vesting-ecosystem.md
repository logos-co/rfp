# Appendix: Token Vesting Ecosystem

This appendix surveys existing token vesting protocols and related
infrastructure across the Ethereum and Solana ecosystems. It serves
as a reference for
[RFP-017](../RFPs/RFP-017-token-vesting.md), providing context on
current market participants, their mechanisms, fee structures, and
design trade-offs.

## Protocols Surveyed

Protocols are ordered by adoption scale (largest first) and this
order is maintained throughout the document.

| Protocol | Ecosystem | Type | Fee | TVL / Locked Value | Launched |
|---|---|---|---|---|---|
| Magna | EVM + Solana + Aptos | Enterprise vesting | Undisclosed (enterprise) | $60B peak (2025) [1] | 2021 |
| Sablier | EVM (27 chains) + Solana | Streaming / vesting | Zero (optional broker 0--10%) [2] | ~$500K on-chain [3] | 2019 |
| Streamflow | Solana (primary) | Vesting / distribution | 0.16 SOL per contract [4] | N/A | 2021 |
| Hedgey Finance | EVM + Solana | Vesting / lockups | Zero [5] | N/A | 2022 |
| Jupiter Lock | Solana | Vesting / locks | Zero [6] | N/A | 2024 |
| Team Finance | EVM | Vesting / LP locks | ~$150 flat per chain [7] | N/A | 2020 |
| Superfluid | EVM | Streaming / vesting | Gas only [8] | N/A | 2021 |
| LlamaPay | EVM | Streaming / payroll | Zero [9] | N/A | 2022 |
| Bonfida | Solana | Reference implementation | Zero [10] | N/A | 2020 |
| AbdelStark ERC-20 | EVM | Reference library | Zero (self-deployed) [11] | N/A | 2021 |
| Tokenomist | Data / analytics | Aggregator | Free (basic tier) [12] | N/A (data only) | 2022 |

## Scale and Traction

Token vesting is a mature, high-value infrastructure category.
Tokenomist documented $97.43B in token vesting unlocks across
major crypto projects during 2025 alone [12], spanning
infrastructure and L1 tokens, DeFi, AI, and L2 sectors.

The on-chain vesting protocol market has consolidated around a
small number of dominant protocols. EVM and Solana are effectively
saturated for standard vesting functionality; competition has
driven fees to zero for most protocols. The frontier is privacy,
novel VMs, and cross-chain composability.

**Magna** is the largest vesting platform by TVL, reaching $60B
at peak in 2025 [1]. Magna serves 160+ enterprise clients with
both on-chain escrow contracts and off-chain schedule management
(tokens remain in treasury, distributed at unlock time). The
platform supports EVM, Solana, and Aptos, making it the broadest
multi-VM vesting solution. In February 2026, Magna was acquired by
Payward (Kraken's parent company) [13], validating the strategic
value of vesting infrastructure at institutional scale.

**Sablier** is the longest-running vesting protocol (since 2019),
deployed across 27 EVM chains and Solana [2]. It offers four
product packages (Lockup, Flow, Airdrops, Bob) and is the only
protocol with custom vesting curves (LockupDynamic). All stream
positions are ERC-721 NFTs, enabling secondary market trading and
DeFi composability. Backed by a16z crypto. On-chain TVL appears
modest (~$500K on DeFiLlama [3]), though aggregate stream volume
across 27 chains is substantially larger; short-lived streams and
multi-chain fragmentation understate the snapshot metric.

**Streamflow** is the most complete Solana-native vesting
platform, covering vesting, airdrops, staking, token locks, and
on-chain payroll [4]. It charges flat SOL fees per contract
(0.16 SOL, approximately $20 at current prices) and offers an
auto-claim keeper that pushes tokens to recipients at each unlock
event for an additional 0.25 SOL. Cancellation permissions are
configurable (sender, recipient, or either party). Streamflow
does not use NFT-based positions; vesting positions are
non-transferable.

**Hedgey Finance** is free at all levels and provides the cleanest
conceptual model in the ecosystem: Vesting Plans (revocable, for
employees and contributors) and Lockup Plans (non-revocable,
transferable, for investors) [5]. Both types are represented as
ERC-721 NFTs, with bound (soulbound) variants available.
Hedgey uniquely supports governance voting with locked tokens.
In April 2024, $44.5M was drained from a deprecated
ClaimCampaigns.sol contract via a flash loan exploit [14]; core
Vesting and Lockup Plan contracts were not directly affected.

**Jupiter Lock** launched in August 2024 as a free, open-source,
audited vesting tool built by Jupiter Exchange, the leading Solana
DEX aggregator [6]. Its distribution advantage is inherent:
teams already using Jupiter for trading naturally discover Jupiter
Lock. All locks are publicly viewable on the UI, positioning
transparency as a credibility signal. Audited by Sec3 and
OtterSec.

**Team Finance** bundles vesting contracts with LP token locks and
a branded embeddable widget that projects can place on their own
website [7]. The branded vesting widget is unique in the
ecosystem. Fixed USD-equivalent fees (~$150 per chain in native
token) apply to each action, with Pro and Enterprise subscription
tiers available.

**Superfluid** implements gas-free continuous streaming via the
Super Token standard (an ERC-20 extension) [8]. Balances update
mathematically at query time without per-tick gas costs. A
VestingScheduler module adds cliff and end-date support on top of
the core streaming primitives. Superfluid does not escrow tokens
(sender can cancel at any time), making it a weaker commitment
device than escrow-based protocols.

**LlamaPay** is a gas-efficient EVM streaming protocol (3.2 to
3.7x cheaper than Sablier) using a shared contract model rather
than per-stream deployments [9]. It is focused on payroll, not
structured vesting: it supports no cliff, no milestone, and no
discrete unlock schedules.

**Bonfida** produced the first audited Solana vesting contract
(2020) [10]. It is a reference implementation, not a hosted
product: a minimal on-chain program with a permissionless unlock
crank (any wallet can trigger token release at the specified slot
height). Bonfida influenced the broader Solana vesting ecosystem
but is largely superseded by Streamflow and Jupiter Lock.

**AbdelStark ERC-20** is an Apache-2.0 Solidity library providing
cliff + linear vesting with a configurable slice period parameter
that unifies continuous and discrete unlock models [11]. Audited
by Hacken. Published as the `erc20-token-vesting` npm package.
It is a developer library, not a hosted protocol.

**Tokenomist** (formerly TokenUnlocks) is the canonical token
unlock data aggregator [12]. It tracks on-chain-verified vesting
schedules across major crypto projects and published the
$97.43B aggregate unlock figure for 2025. Tokenomist does not
provide vesting infrastructure; it provides the data layer that
institutional investors use to anticipate unlock events.

## Schedule Types

Protocols implement between one and six schedule types. RFP-017
requires three: cliff + linear, fully linear, and milestone-based.

| Protocol | Cliff + Linear | Fully Linear | Milestone | Periodic Tranched | Custom Curves | Per-second Streaming |
|---|---|---|---|---|---|---|
| Magna | Yes | Yes | Yes | Yes | No | No |
| Sablier | Yes | Yes | No | Yes (LockupTranched) | Yes (LockupDynamic) | Yes |
| Streamflow | Yes | Yes | No | Yes | No | No |
| Hedgey Finance | Yes | Yes | Yes (admin-triggered) | Yes | No | No |
| Jupiter Lock | Yes | Yes | No | Yes | No | No |
| Team Finance | Yes | Yes | No | Yes | No | No |
| Superfluid | Yes (scheduler) | Yes | No | No | No | Yes |
| LlamaPay | No | Yes | No | No | No | Yes |
| Bonfida | Yes | Yes | No | Yes (slot-based) | No | No |
| AbdelStark ERC-20 | Yes | Yes | No | Yes (via slicePeriodSeconds) | No | No |

**Cliff + linear** is the de facto industry standard for team and
investor allocations. Every major protocol supports it. The
typical configuration is a 1-year cliff followed by 3 years of
linear release (4 years total).

**Fully linear** (cliff = 0) is supported by every protocol that
supports cliff + linear. It is appropriate for ongoing
contributors, advisors, and grant programmes where disbursement
should begin immediately.

**Milestone-based** is the rarest schedule type. Only Hedgey and
Magna support admin-triggered milestones. No protocol in the
studied set implements trustless off-chain milestone verification;
all milestone implementations require a trusted admin or
governance body to signal completion. RFP-017's decision to keep
milestone authority with the creator (not an on-chain oracle)
aligns with existing practice.

**Custom curves** are exclusive to Sablier (LockupDynamic), which
allows arbitrary vesting shapes defined as segment arrays with
different rates. This enables exponential, front-loaded, step
function, and other non-linear schedules. No other protocol offers
this capability.

**Per-second streaming** is offered by Sablier, LlamaPay, and
Superfluid. It distributes tokens continuously rather than in
discrete events, eliminating concentrated unlock-day sell
pressure. RFP-017 does not require streaming; the three required
schedule types cover the full range of real-world vesting
relationships.

## Fee Structures

Fee competition in the vesting market has been largely won by the
zero-fee camp: 7 of 10 protocols studied have zero protocol fees.

| Protocol | Protocol Fee | Per-Schedule Cost | Claim Fee | Notes |
|---|---|---|---|---|
| Magna | Undisclosed [1] | Undisclosed | Undisclosed | Enterprise pricing; negotiated per client |
| Sablier | Zero [2] | Gas only | Gas only | Optional broker fee (0--10%) for third-party integrators; official UI is free |
| Streamflow | 0.16 SOL (~$20) per contract [4] | 0.16 SOL | 0.009--0.017 SOL | Auto-claim add-on: 0.25 SOL; airdrop clawback: 1.70% of tokens returned |
| Hedgey Finance | Zero [5] | Gas only | Gas only | 100% free at all levels |
| Jupiter Lock | Zero [6] | Solana gas only | Solana gas only | Confirmed by Jupiter's official announcement |
| Team Finance | ~$150 per chain [7] | Flat per action | Gas only | Pro and Enterprise subscription tiers available |
| Superfluid | Gas (stream start/stop) [8] | Gas only | Zero (continuous netting) | SUP governance token launched Feb 2025 |
| LlamaPay | Zero [9] | Gas only | Gas only | Shared contract model minimises per-stream gas |
| Bonfida | Zero [10] | Solana tx (~$0.0005) | Solana tx (~$0.0005) | Unlock crank fee paid by whoever calls it |
| AbdelStark ERC-20 | Zero [11] | Gas (EVM) | Gas (EVM) | Self-deployed library |

**Streamflow's 1.70% airdrop clawback fee** is the only
percentage-based fee in the ecosystem. It applies specifically to
airdrop cancellations (tokens returned via clawback), not to
standard vesting operations. Standard Streamflow vesting
cancellations incur no percentage fee.

**Team Finance's flat USD fee** (~$150 per chain in native token)
is predictable regardless of token value. It is efficiently priced
for multi-recipient distributions: the per-recipient cost falls
to near-zero for large batches. Pro and Enterprise subscription
tiers are available for high-volume projects.

**Sablier's broker fee model** is unique: third-party front-end
integrators can charge up to 10% of the streamed token amount
when users create streams through their interface [2]. The
official Sablier UI charges zero broker fee. This creates an
incentive for third-party developers to build on Sablier without
the core protocol needing to charge fees.

### Design implications

Zero fees are the dominant model and the most adoption-friendly
design. Any RFP-017 implementation on LEZ should default to zero
protocol fees, with a governance-activatable fee switch for
future monetisation. Sablier V2 included such a switch
(admin-controlled, hard-capped at 10%) but never activated it
and removed it in V2.2; current Sablier versions charge modest
interface fees instead [2]. A broker fee for third-party
integrators and launchpad partners is worth considering. LEZ
transaction and proof costs will be the primary user-facing
expense.

### Cost comparison for 100-recipient vesting

For a standardised scenario (100 recipients, 1,000 tokens each,
4-year schedule with 1-year cliff):

| Protocol | Approach | Approximate Total Cost (100 recipients) |
|---|---|---|
| Sablier (Arbitrum/Base) | Bulk multi-call | ~$10 creation gas + recipient claim gas |
| Sablier (Merkle airdrop) | Merkle root + pull | ~$10 one-time setup; recipients pay own claim gas |
| Sablier (Ethereum mainnet) | Bulk multi-call | $500--$2,000 creation gas |
| Streamflow (Solana) | CSV bulk | 16 SOL (~$2,000) |
| Hedgey (Arbitrum) | Multi-call | ~$10 creation gas + recipient claim gas |
| Jupiter Lock (Solana) | Individual | Negligible (Solana gas only) |
| Team Finance (Ethereum) | Flat fee | ~$150 flat + EVM gas |
| Bonfida (Solana) | Individual | Negligible (Solana gas only) |

The Merkle airdrop model (Sablier) is the most gas-efficient
approach for 1,000+ recipients: one on-chain Merkle root
commitment covers all recipients, with each recipient paying their
own claim gas. Creator cost does not scale with recipient count.

## Cancellation and Revocation

Every protocol that supports cancellation follows a universal
invariant: **vested tokens belong to the recipient and cannot be
reclaimed; unvested tokens return to the creator.**

| Protocol | Cancellation Model | Vested Handling | Unvested Handling | Notes |
|---|---|---|---|---|
| Magna | Standard split (on-chain); creator-controlled (off-chain) [1] | Recipient keeps | Creator reclaims | Off-chain mode gives creator full distribution control |
| Sablier | Cancelable or non-cancelable (set at creation) [15] | Recipient keeps | Creator reclaims | One-way conversion: cancelable to non-cancelable (irreversible) |
| Streamflow | Configurable: sender, recipient, or either [4] | Recipient keeps | Creator reclaims | No pause/resume |
| Hedgey Finance | Vesting Plans: revocable by admin; Lockup Plans: non-revocable [5] | Recipient keeps | Admin reclaims | Two-product separation cleanly distinguishes revocable vs irrevocable |
| Jupiter Lock | Configurable: creator, recipient, both, or neither [6] | Recipient keeps | Creator reclaims | "Neither" creates an irrevocable commitment |
| Team Finance | Revocable schedules: creator cancels [7] | Recipient keeps | Creator reclaims | |
| Superfluid | Sender cancels any time [8] | Recipient keeps (accrued) | Stops streaming | No escrow: sender can simply stop the stream |
| LlamaPay | Payer cancels any time [9] | Recipient keeps (accrued) | Stops streaming | No escrow |
| Bonfida | Not documented [10] | N/A | N/A | Likely no cancellation in base design |
| AbdelStark ERC-20 | Revocable flag set at creation [11] | Recipient keeps | Owner reclaims | Non-revocable schedules cannot be cancelled |

All protocols guarantee that vested-but-unclaimed tokens remain
claimable by the recipient after cancellation, with no documented
time expiry.

**Sablier's unique features.** Sablier Lockup offers a one-way
conversion from cancelable to non-cancelable [15]: once a stream
is made non-cancelable, this change is irreversible. This serves
as a credible commitment mechanism (projects can issue vesting to
investors and later make it irrevocable as a trust signal).
Sablier Flow (open-ended streaming) uniquely supports pause and
resume rather than outright cancellation [15]; pausing preserves
accrued debt through cycles. No other protocol offers
pause/resume.

**Hedgey's two-product separation** is the cleanest conceptual
model for the revocable/irrevocable distinction [5]. Vesting Plans
(for employees, contributors) are revocable by the Vesting Admin.
Lockup Plans (for investors) are non-revocable by design and
transferable via NFT, giving investors an irrevocable right to
locked tokens with secondary market exit.

**Configurable cancel permissions** (Streamflow, Jupiter Lock)
allow the same contract interface to serve both revocable team
vesting and irrevocable investor lockups by setting the
cancellation authority at creation time [4][6].

RFP-017 follows the universal invariant: on cancellation, vested
tokens remain claimable by the beneficiary and unvested tokens
return to the creator.

## Transferable Positions

Position transferability determines whether the right to receive
future vested tokens can itself be transferred or sold.

| Protocol | Transferable | Mechanism | Secondary Market |
|---|---|---|---|
| Magna | Not publicly documented [1] | Enterprise platform; not open source | N/A |
| Sablier | Yes | ERC-721 NFT per stream [2] | Any NFT marketplace (OpenSea, Blur, etc.) |
| Hedgey Finance | Yes (configurable) | ERC-721 NFT per plan; bound (soulbound) variant available [5] | NFT marketplace; Lockup Plan NFTs are non-revocable + transferable |
| Streamflow | No [4] | Fixed recipient address | None |
| Jupiter Lock | Configurable | Recipient-change permission at creation [6] | No secondary market |
| Team Finance | No [7] | Fixed recipient address | None |
| Superfluid | No (Super Tokens are transferable, but stream positions are not) [8] | N/A | N/A |
| LlamaPay | No [9] | Fixed recipient address | None |
| Bonfida | No [10] | Fixed recipient address | None |
| AbdelStark ERC-20 | No [11] | Fixed recipient address | None |

**Sablier's NFT model** is the most composable approach. Every
stream is an ERC-721 NFT held by the recipient [2]. The NFT can be
sold on any marketplace, used as collateral in lending protocols
that accept ERC-721, or transferred instantly without creator
involvement. A recipient with a 4-year linear schedule who is 1
year in can sell the NFT at a discount to the present value of
the remaining stream, giving the buyer the right to receive the
remaining vested tokens.

**Hedgey's NFT model** adds a critical investor protection layer
[5]. Lockup Plan NFTs are both non-revocable and transferable:
the buyer of a Lockup Plan NFT has a guaranteed, irrevocable right
to the locked tokens. Vesting Plan NFTs are transferable but
revocable (the Vesting Admin can still revoke), so buyers accept
revocation risk. Hedgey also offers "bound" (soulbound) variants
for identity-linked vesting where secondary market trading is
undesirable.

**Jupiter Lock and Streamflow** offer a simpler model: the
recipient address can be changed if the creator granted that
permission at creation [4][6]. This is less flexible than
NFT-based ownership because it typically requires explicit
permission and does not enable frictionless secondary market
trading.

RFP-017 specifies configurable transferability at creation (frozen
afterwards). The NFT-based model (Sablier, Hedgey) represents
current best practice for composability and secondary market
access, though the implementation choice is left to the proposer.

## Privacy

**No existing token vesting protocol provides any privacy features
whatsoever.** This is not a partial gap; it is an absolute absence
across every protocol studied.

| Protocol | Schedule Amounts Public | Recipient Addresses Public | Claim Events Public | Any Privacy Feature |
|---|---|---|---|---|
| Magna | Yes | Yes | Yes | No |
| Sablier | Yes | Yes | Yes | No |
| Streamflow | Yes | Yes | Yes | No |
| Hedgey Finance | Yes | Yes | Yes | No |
| Jupiter Lock | Yes (by design) | Yes | Yes | No |
| Team Finance | Yes | Yes | Yes | No |
| Superfluid | Yes | Yes | Yes | No |
| LlamaPay | Yes | Yes | Yes | No |
| Bonfida | Yes | Yes | Yes | No |
| AbdelStark ERC-20 | Yes | Yes | Yes | No |

All protocols publish all vesting parameters on-chain: token,
total amount, schedule type, cliff date, end date, and
beneficiary address. Every claim event (amount, recipient,
timestamp) is publicly visible. Cancellations, milestone signals,
and position transfers are all observable.

**Jupiter Lock explicitly celebrates transparency as a product
feature** [6]. All locks are displayed on a public explorer,
positioning on-chain visibility as a credibility signal for
project teams and investors.

The entire sector frames on-chain vesting transparency as a
benefit (investors can verify schedules), not a limitation. No
protocol has ZK-based vesting on its roadmap as of April 2026.

### The problem with full transparency

This universal transparency creates real problems:

- **Front-running unlock events.** Unlock schedules are public;
  traders position ahead of large unlocks anticipating sell
  pressure. The April 2025 MANTRA ($OM) collapse (90% in hours,
  from over $6 to under $0.50, wiping over $5B in market cap
  [18]), while not solely caused by front-running, illustrates
  the fragility of transparent unlock mechanics at scale.
- **Investor allocation leakage.** A project's fundraising round
  terms are effectively public once vesting contracts are
  deployed. Competitors, token analytics firms, and short sellers
  all have access.
- **Recipient targeting.** Large token grants to known addresses
  are publicly trackable. Recipients of significant allocations
  become targets for phishing and social engineering.

### What RFP-017's claim-and-re-shield would change

RFP-017 introduces privacy at the post-claim layer:

1. Schedule parameters remain public (identical to all existing
   protocols).
2. Claim events are public (amount, beneficiary, timestamp).
3. Claimed tokens are deposited into LEZ's shielded pool via the
   claim-and-re-shield pattern.
4. Subsequent movements of claimed tokens are private.

This means recipients can accumulate and manage vested tokens
without on-chain surveillance. Investors cannot be front-run when
their large unlock events trigger. Contributor grants are not
publicly visible to competitors after claiming. Post-vesting token
movements (swap, stake, transfer) are private.

RFP-017 would be the first privacy-preserving token vesting
protocol in production, even in this limited claim-and-re-shield
form.

## References

[1] The Block, "Kraken buys token vesting platform Magna, which
hit peak TVL of $60 billion in 2025," Feb 2026.
https://www.theblock.co/post/390358/kraken-buys-token-vesting-platform-magna-hit-peak-tvl-60-billion-in-2025

[2] Sablier documentation: Fees.
https://docs.sablier.com/concepts/protocol/fees

[3] DeFiLlama: Sablier protocol page.
https://defillama.com/protocol/sablier

[4] Streamflow documentation: Costs of using Streamflow.
https://docs.streamflow.finance/costs-of-using-streamflow

[5] Hedgey Finance website and GitHub: Locked_VestingTokenPlans.
https://hedgey.finance/ ;
https://github.com/hedgey-finance/Locked_VestingTokenPlans

[6] Jupiter Exchange, Jupiter Lock announcement and support
article.
https://x.com/JupiterExchange/status/1826959175089545604 ;
https://support.jup.ag/hc/en-us/articles/22620899015452-What-is-Jupiter-Lock

[7] Team Finance documentation: pricing and fee blog.
https://docs.team.finance/team-finance-pricing ;
https://blog.team.finance/token-management-service-fees/

[8] Superfluid documentation: vesting use case.
https://docs.superfluid.org/docs/use-cases/vesting

[9] LlamaPay website and GitHub.
https://llamapay.io/ ;
https://github.com/LlamaPay/llamapay

[10] Bonfida token-vesting GitHub repository.
https://github.com/Bonfida/token-vesting

[11] AbdelStark/token-vesting-contracts GitHub and Hacken audit.
https://github.com/AbdelStark/token-vesting-contracts ;
https://github.com/abdelhamidbakhta/token-vesting-contracts/blob/main/audits/hacken_audit_report.pdf

[12] Tokenomist (formerly TokenUnlocks), "2025 Token Unlocks
Review."
https://insights.unlocks.app/2025-token-unlocks-review-a-complete-breakdown-of-emissions-insider-vesting-and-market-impact/

[13] CoinDesk, "Kraken continues acquisition streak by buying
token management firm Magna ahead of IPO push," Feb 2026.
https://www.coindesk.com/business/2026/02/18/kraken-continues-acquisition-streak-by-buying-token-management-firm-magna-ahead-of-ipo-push

[14] Halborn, "Explained: the Hedgey Finance hack (April 2024)."
https://www.halborn.com/blog/post/explained-the-hedgey-finance-hack-april-2024

[15] Sablier documentation: cancelability.
https://docs.sablier.com/concepts/cancelability

[16] Kraken blog, "Payward acquires Magna."
https://blog.kraken.com/news/payward-acquires-magna

[17] arXiv 2501.03391, "Privacy-Preserving Smart Contracts for
Permissioned Blockchains: A zk-SNARK-Based Recipe," Jan 2025.
https://arxiv.org/html/2501.03391v1

[18] CoinDesk, "Mantra's OM Crashes 90% in Bizarre Sell-Off as
Team Alleges 'Forced Liquidations'," Apr 2025.
https://www.coindesk.com/markets/2025/04/14/mantra-s-om-crashes-90-in-bizarre-selloff-as-team-alleges-forced-liquidations
