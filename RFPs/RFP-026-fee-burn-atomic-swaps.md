---
id: RFP-026
title: Fee-Burn Atomic Swaps (Refund-Side Free-Option Mitigation)
tier: M
funding: $TBD
status: draft
category: Applications & Integrations
---

# RFP-026 — Fee-Burn Atomic Swaps (Refund-Side Free-Option Mitigation)

> **Note:** This RFP is a *decision-stage draft*. It exists to help the Logos
> team and the community compare cross-chain DEX designs across RFP-021 through
> RFP-026. Hard requirements, team profile, timeline, and contracting details
> are deliberately omitted; they will be filled in if the design is selected for
> funding.

## 🧭 Overview

Extend RFP-003 (Atomic Swaps with LEZ, open) with a **non-refundable fee on the
refund branch** that prices out the free option without requiring an on-LEZ
bond. The fee is burnt (sent to an unspendable output), not paid to the
counterparty, so it does not skew incentives. The mechanism is the proposal in
[eigenwallet PR #675](https://github.com/eigenwallet/core/pull/675) (open as of
2026-05-22), generalised to the LEZ-paired case.

The protocol intuition: today, in the deployed BTC-XMR adaptor-signature swap, a
taker who has locked BTC can refund at the cost of one Bitcoin transaction fee
(a few thousand sats). This makes the refund branch effectively free, which is
exactly what creates the free-option problem documented in the
[atomic-swaps primer](../appendix/atomic-swaps-primer.md#the-free-option-problem).
The fee-burn fix splits the refund branch so that taking it costs the taker a
configurable fraction of the locked amount. The fraction is set by the maker per
quote; the taker sees it before initiating and decides whether to trade.

The mechanism is a **substitute** for the bond-based approach in RFP-022, not a
complement. The two instruments price the same free option using different
escrow locations; layering both is double-counting (charging the option holder
twice for the same option).

- **RFP-022 LEZ bond**: the option premium is a separate amount escrowed on LEZ
  at Commit; slashes to the counterparty on LEZ-observable default;
  capital-efficient because it refunds on honest completion.
- **RFP-026 fee-burn**: the option premium is a haircut on the refunded
  principal taken on the external chain in the refund branch; burns rather than
  slashes (to avoid skewing the counterparty's incentives); does not refund on
  honest completion (only the *refund branch* is fee-burdened, so honest
  completions pay nothing).

Where each dominates:

- **Tier 1 (LEZ↔BTC, LEZ↔ETH)**: the LEZ bond strictly dominates. The bond
  returns on honest completion; the fee-burn destroys principal whenever the
  refund branch is taken. With both sides' locks LEZ-observable, the bond gives
  the same option-closure at lower expected capital cost.
- **Tier 2 (LEZ↔XMR)**: the fee-burn dominates the LEZ bond. RFP-022's bond
  cannot price the residual Phase-2 free option (the trigger event, the XMR-side
  lock, is off-LEZ; see RFP-022 §Tier 2). The fee-burn on the
  script-bearing-chain refund branch *can* price it because the trigger (the
  refund-branch transaction itself) is on a chain both parties observe directly.
  The fee-burn closes the gap RFP-022's bond cannot.

Applicants should not propose stacking both mechanisms on the same option
boundary. They should propose the fee-burn as the Tier-2-preferred primitive and
explicitly cede Tier 1 to RFP-022's LEZ bond, or argue why one mechanism alone
covers both tiers.

## Background: the eigenwallet PR #675 design

The eigenwallet PR (open, not yet merged as of 2026-05-22) modifies the
Bitcoin-side refund branch of the BTC-XMR adaptor-signature swap. Roles in the
eigenwallet convention: **Bob is the taker (BTC-side, locks BTC first); Alice is
the maker (XMR-side, locks XMR second).** This matches the deployed COMIT
direction.

Current state without fee-burn: after `TxCancel` is published, Bob has two paths
to recover his BTC after a 24h timelock — `TxFullRefund` returns all of `[B]`
(the BTC amount) to Bob; or Alice publishes `TxPunish` if Bob has stayed
offline. Bob's refund costs roughly the transaction fee of `TxFullRefund` plus 2
prior transactions.

Proposed state with fee-burn: the `TxFullRefund` path is **replaced** by
`TxPartialRefund`, which splits the locked output into two pieces: `[B]`
(refunded to Bob immediately) and a `Deposit [A+B]` (held in an intermediate
output). The deposit goes through a sub-protocol:

1. After a further short timelock (30 minutes in PR #675), Bob can publish
   `TxReclaim` to retrieve the deposit himself.
2. Before that timelock, Alice can publish `TxWithhold` to take the deposit into
   her control (because she observed Bob refunded against the spirit of the
   protocol).
3. After Alice has withheld, she can either publish `TxMercy` to release the
   deposit back to Bob (forgiving the refund), or burn it by not signing any
   further transaction (the deposit becomes permanently unspendable as soon as
   `TxWithhold`'s output spend conditions cannot be satisfied by either party
   alone).

The deposit fraction is set by the maker per quote (Alice configures her
`ask_spread` plus a refund-fee parameter); the taker sees the deposit fraction
before locking BTC and chooses whether to trade. The deposit is **burnt** (not
paid to Alice) so that Alice has no incentive to provoke a refund just to
collect the deposit; Alice's mercy path exists so honest refunds (e.g. Alice's
own connectivity loss) can be forgiven without permanent loss to Bob.

Source: [eigenwallet/core PR #675](https://github.com/eigenwallet/core/pull/675)
(accessed 2026-05-22; archived in research vault).

eigenwallet shipped a related-but-narrower mechanism in v4.0.0 (2026-03-16): an
**anti-spam deposit** where Alice can withhold part of a refund for up to 30
minutes then release with "mercy". PR #675 generalises this into a
protocol-level fee-burn with maker-set fraction. Source:
[eigenwallet/core release 4.0.0](https://github.com/eigenwallet/core/releases/tag/4.0.0)
(accessed 2026-05-22).

## Direction dependence: where the fee-burn works and where it does not

This is the central design question and the RFP applicants must address it
explicitly.

The fee-burn mechanism prices the option held by **the party that locks first
and would walk via the refund branch**. In the deployed BTC-XMR direction (BTC
locks first, eigenwallet's PR #675 case):

- Bob (BTC taker, locks first) is the party who can grief via refund. The
  fee-burn forces him to forfeit `[A+B]` deposit on every refund. The free
  option Bob holds at the Phase 1-to-Phase 2 boundary (between his BTC lock and
  Alice's XMR lock) is now priced.
- Alice (XMR maker, locks second) holds a different option at the Phase
  2-to-Phase 3 boundary (between her XMR lock and Bob's reveal of the secret).
  The fee-burn on Bob's refund branch does *nothing* to price this. Alice's
  option is priced separately, either by her own conduct (she chose to lock
  against Bob; her downside is lock-window opportunity cost on XMR) or by an
  LEZ-side bond on her side (which RFP-022 Tier 2 cannot fully provide because
  her XMR lock is not LEZ-observable).

Trade-direction analysis:

| Trade direction                                                | Who locks first                                                                                                                            | Who locks second            | Fee-burn prices whose option?                                                                                                                       |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| **LEZ→BTC** (user wants BTC, maker holds BTC)                  | LEZ-side party locks first (if the LEZ↔BTC pair follows the eigenwallet convention)                                                        | BTC-side party locks second | Option of the LEZ-side party who walks via LEZ-side refund branch. **Workable** because LEZ has script.                                             |
| **BTC→LEZ** (user wants LEZ-side asset, maker holds LEZ asset) | BTC-side party locks first                                                                                                                 | LEZ-side party locks second | Option of the BTC-side party who walks via BTC-side refund branch. **Workable** because Bitcoin has script (PR #675 demonstrates the construction). |
| **LEZ→XMR** (user wants XMR, maker holds XMR)                  | LEZ-side party locks first (deployed eigenwallet convention has BTC first; for LEZ↔XMR, LEZ takes the BTC role as the script-bearing side) | XMR-side party locks second | Option of the LEZ-side party who walks via LEZ-side refund branch. **Workable** because LEZ has script. The fee-burn lives on LEZ.                  |
| **XMR→LEZ** (user wants LEZ-side asset, maker holds LEZ asset) | LEZ-side party still locks first (same as above by primitive role)                                                                         | XMR-side party locks second | Same as above: fee-burn lives on the LEZ side. **Workable**.                                                                                        |

**Where the fee-burn does not work**: the **reverse XMR-first variant**
described in Hoenisch and del Pino 2021 §4 (a CLSAG-adaptor-signature
construction where the XMR-side party locks first). In that variant the
locked-first leg is on Monero, which has no script; a Monero-side fee-burn on
the refund branch is not constructible because Monero has no refund-branch
script at all (Gugger 2020 §3.1: "Monero does not require any particular
on-chain primitives (hashlocks, timelocks)"). For this direction, the fee-burn
approach is structurally infeasible until non-disclosing Monero proof primitives
(FCMP++) ship and allow lock state to be verified on a script-bearing chain.

**Where the fee-burn also does not work**: it does not price the
**second-locker's option** (the party who waits, sees the first lock, and
chooses whether to advance). That option lives on the no-fee-burn side of the
swap. For the BTC-XMR deployed direction, this is Alice's pre-XMR-lock option
(which she does not face as a real risk: walking away pre-lock costs her nothing
beyond the cancelled quote). For the LEZ↔XMR cases, Alice's pre-XMR-lock walk is
the residual non-bond-priceable option from RFP-022 Tier 2; the fee-burn does
not address it either.

## Desired properties

- **No on-LEZ bond required for the core mechanic.** The fee-burn lives in the
  external-chain script; LEZ is uninvolved in the option-pricing instrument
  itself. This is the structural difference from RFP-022.
- **Maker-set fraction.** Each maker chooses the deposit fraction per quote (a
  percentage of the locked amount, configurable in the maker daemon analogous to
  `ask_spread`). Takers see the fraction before initiating; choose not to trade
  if the fraction is too high.
- **Burn destination is not the counterparty.** The deposit is sent to an
  unspendable output (or to a 2-of-2 spend conditional on a transaction that
  neither party will ever sign in equilibrium). This prevents the maker from
  having an incentive to provoke refunds to collect the deposit.
- **Mercy path.** The maker can release the deposit back to the taker after
  withholding it, so honest refunds (e.g. connectivity loss on either side) can
  be forgiven. This is the analogue of bond-refund-on-honest-completion in
  RFP-022.
- **Dominates the LEZ bond on Tier 2 (LEZ↔XMR).** Because the fee-burn lives on
  the script-bearing chain where the refund-branch transaction itself is the
  trigger, it prices the off-LEZ residual option that RFP-022's bond cannot.
  Tier 2 swaps gain a real option-pricing mechanism (rather than only reputation
  \+ market competition).
- **Layerable with RFP-023.** Maker reputation gates which fraction takers will
  accept from which makers; takers building reputation can be offered lower
  fractions over time. Reputation reduces the friction; fee-burn protects the
  maker against new-identity griefing.

## High-level functionality and flow

```mermaid
sequenceDiagram
    autonumber
    participant A as Alice (maker)
    participant LEZ as LEZ swap contract
    participant Ext as External chain
    participant B as Bob (taker)

    Note over A,B: Phase 0 - Quote (with fee-burn parameter)
    A->>B: Quote (price, expiry, swap_id, refund_fee_fraction)

    Note over A,Ext: Phase 1 - Lock-Ext (taker locks first; deployed direction)
    B->>Ext: Lock to script with three exit paths:<br/>(a) claim by Alice with secret<br/>(b) partial-refund by Bob splits<br/>    [trade_amount] + [deposit = fee_fraction × trade_amount]<br/>(c) Alice's punish path if Bob disappears
    Ext-->>LEZ: Inclusion proof (verified by LEZ light client)

    Note over A,LEZ: Phase 2 - Lock-LEZ
    A->>LEZ: Lock trade_amount conditioned on s

    Note over B,LEZ: Phase 3 - Reveal
    B->>LEZ: Publish adaptor signature (reveals s)
    A->>Ext: Use s to claim external-chain output

    Note over A,B: Or: Phase 2' - Cancel (refund branch)
    B->>Ext: Publish TxPartialRefund:<br/>[trade_amount] back to Bob immediately<br/>[deposit] into intermediate output
    Note over A,B: After short timelock (e.g. 30 min)
    alt Maker forgives
        A->>Ext: TxMercy: deposit released to Bob
    else Maker withholds (deposit burns)
        A->>Ext: TxWithhold (deposit becomes unspendable)
    else Maker no-shows
        B->>Ext: TxReclaim: deposit returned to Bob
    end
```

The crucial detail: the deposit path is the *only* difference from a vanilla
refund. Honest completion is unchanged. Only the refund branch carries the
option premium.

## Pros

- **No LEZ capital lockup.** Takers and makers do not need LEZ-denominated
  assets to participate. The mechanism works on any pair where the
  script-bearing chain has refund-branch scripting.
- **Composable with the eigenwallet ecosystem.** Adopting the same construction
  as eigenwallet PR #675 means the LEZ atomic-swap protocol can interoperate
  with eigenwallet makers on the BTC-XMR pair. This is materially valuable for
  the BTC-XMR direction specifically.
- **Maker-set, not protocol-set.** Each maker chooses their fee fraction; the
  protocol does not impose a single rate. Markets discover the right fraction
  over time.
- **Burn destination avoids incentive skew.** Unlike a "refund fee paid to
  counterparty" design, the burn cannot motivate either party to grief the
  other. The deposit going to an unspendable output (after mercy timeout) is
  bounded loss for both sides.
- **Layerable with RFP-022 bonds.** A taker can post an LEZ bond *and* face a
  fee-burn on the external chain. The compound cost of the abort branch is the
  sum of the two.
- **Survives Monero-non-observability**: the fee-burn lives on the
  script-bearing chain. LEZ does not need to witness anything on the Monero
  side, so the LEZ↔XMR Tier 2 problem (from RFP-022) does not apply to the
  fee-burn mechanism itself.

## Cons

- **Does not price the second-locker's option.** The fee-burn lives in the
  first-locker's refund branch only. The option held by the second-locker (Alice
  in the BTC-XMR direction, between her XMR lock and Bob's reveal) is not priced
  by this mechanism. For full bilateral mitigation, layer with RFP-022 Tier 1
  (where both sides are LEZ-observable) or accept the asymmetry.
- **Does not work for XMR-first direction.** The reverse XMR-first variant
  (Hoenisch and del Pino 2021 §4) requires Monero-side refund-branch scripting,
  which does not exist. Pairs where the script-bearing side is unable to play
  the locks-first role at all (currently none in this bundle) would also be
  excluded.
- **Maker reputation is load-bearing.** The maker's mercy path is a
  discretionary choice; in the limit, a malicious maker could withhold every
  deposit and force burns. The deposit fraction must be sized small enough that
  honest takers tolerate the worst-case loss, which constrains how much option
  premium the mechanism can collect. Reputation (RFP-023) on the maker side is
  the soft pressure that keeps mercy honest.
- **External-chain transaction-fee overhead.** The fee-burn protocol requires
  multiple transactions on the external chain (cancel, partial-refund, withhold
  or reclaim or mercy). At high external-chain fee regimes, the protocol's gas
  cost competes with the deposit value.
- **Withhold/mercy/reclaim adds protocol surface.** Auditing and testing the
  additional sub-protocol is a real engineering cost. eigenwallet PR #675 is the
  reference, but adapting it to LEZ-paired swaps requires the LEZ-side analogue
  (when LEZ takes the script-bearing role).
- **Not a bond.** The deposit is not slashed to the counterparty (by design);
  the protocol cannot use the deposit to *compensate* the wronged party.
  Compensation requires RFP-022's on-LEZ bond mechanism layered on top.

## Risks

- **Withhold-mercy race conditions.** The
  Alice-withholds-then-mercifully-releases path must be atomic from a UX
  perspective; partial-protocol-completion (Alice withholds and disappears
  without mercy or final burn) leaves the deposit in limbo. Mitigation: design
  the deposit script so that Bob's reclaim path is always available after a hard
  deadline regardless of Alice's actions. PR #675 already has this property (the
  30-min timelock to `TxReclaim`).
- **Fraction-too-high griefing of takers.** A malicious maker could post an
  unreasonably high deposit fraction in their quote, then trade-and-withhold to
  permanently burn taker funds. Mitigation: discovery-layer fraction caps;
  reputation-weighted fraction acceptance; protocol-level upper bound on the
  deposit fraction (e.g. ≤ 20%).
- **Fraction-too-low under-pricing the option.** If the fraction is set below
  the option's true value (`σ × √T × notional`), the abort branch is still
  EV-positive for adversarial takers. Mitigation: applicants validate the
  maker-set fraction against actual observed volatility; document a sizing
  guidance comparable to the σ×√T×notional rule from the primer.
- **Coordinated fraction-collusion.** If most makers settle on the same fraction
  (a Schelling point), takers cannot route around abusive levels. Mitigation:
  discovery-layer transparency on per-maker fraction history; reputation rewards
  for makers who use lower fractions and honour mercy.
- **External-chain fee-spike defeats burn economics.** During Bitcoin mempool
  congestion, the cost of the refund-branch transactions can exceed the deposit
  value. Mitigation: dynamic deposit-sizing relative to recent external-chain
  fee rates; minimum-deposit floor.
- **Adoption fragmentation with eigenwallet.** If the LEZ-adapted fee-burn
  diverges from eigenwallet's PR #675 in non-backwards-compatible ways, LEZ
  takers cannot route to eigenwallet makers and vice versa. Mitigation:
  implement the same on-the-wire construction PR #675 specifies, with LEZ-side
  script primitives substituted only where LEZ takes the BTC role.

## Relationship to other RFPs in this bundle

- **RFP-003 (Atomic Swaps with LEZ, open)** is the foundation. RFP-026 modifies
  the refund branch of the per-pair atomic-swap construction; the joint-key
  setup, lock, and reveal flow are inherited from RFP-003.
- **RFP-022 (bonded atomic swaps)** is the *substitute* option-pricing
  mechanism. Both RFPs price the same free option but with different escrow
  locations and different capital efficiency. RFP-022's LEZ bond is more
  capital-efficient on Tier 1 (LEZ↔BTC, LEZ↔ETH) because it refunds on honest
  completion. RFP-026's fee-burn is the only mechanism that can price the
  residual off-LEZ option in Tier 2 (LEZ↔XMR). Applicants should propose the
  fee-burn for Tier 2 only and let RFP-022 cover Tier 1, or justify why one
  mechanism alone should cover both.
- **RFP-023 (reputation-based atomic swaps)** is the soft-pressure layer for the
  maker's mercy path. A maker who systematically withholds (rather than
  mercifully releases) earns reputation cost. RFP-023's slashable-event matrix
  should be extended to record `mercy_path_invoked` and `withhold_to_burn`
  separately.
- **RFP-021 (cross-chain privacy DEX)** is orthogonal: federated-custody middle
  chains have no atomic-swap refund branch to attach a fee-burn to. The two
  designs solve different problems.
- **RFP-024 (sXMR pure)** and **RFP-025 (sXMR with SLA)** could optionally adopt
  the fee-burn on the redemption-leg atomic swap. RFP-024's LP economy would
  benefit from refund-branch fee-burns reducing taker-side free-option exposure;
  RFP-025 option 2a could layer fee-burn on top of the LP bond.

## A note on the locks-first direction

This RFP's fee-burn mechanism is constructively dependent on which side locks
first. The bundle (see RFP-022 overview and the
[atomic-swaps primer](../appendix/atomic-swaps-primer.md#locking-order)) treats
locking order as driven by the draining-attack economic analysis, not by the
cryptographic primitive. For the deployed BTC-XMR direction (Bob = BTC taker
locks first), the fee-burn lives on Bitcoin and works as PR #675 specifies. For
LEZ-paired swaps following the same convention (LEZ-side locks first), the
fee-burn lives on LEZ and is implemented via the LEZ-side escrow script.

Applicants must:

1. **State the locks-first direction explicitly** for each pair the RFP targets.
   Default to the deployed convention (BTC-side / LEZ-side locks first); justify
   any deviation.
2. **Show that the fee-burn lives on the locks-first chain.** The fee-burn
   cannot be added to the second-locker's refund branch in a way that prices the
   first-locker's option; the geometry of the protocol forbids it.
3. **Mark which trade directions the chosen pair handles.** A fee-burn on the
   LEZ-side refund branch prices both LEZ→external and external→LEZ trade
   directions (since LEZ locks first either way under the deployed convention).
   A fee-burn on the BTC-side refund branch in a BTC-XMR pair only handles the
   BTC-first direction; the reverse XMR-first direction is structurally
   unaddressed by this mechanism.

## References

- [eigenwallet/core PR #675: fee-burn on refunds](https://github.com/eigenwallet/core/pull/675)
  (accessed 2026-05-22; body archived in research vault)
- [eigenwallet/core release 4.0.0: anti-spam deposit (cancel timelock reduction + 30-minute withhold/mercy)](https://github.com/eigenwallet/core/releases/tag/4.0.0)
  (accessed 2026-05-22)
- [RFP-003: Atomic Swaps with LEZ](./RFP-003-atomic-swaps.md)
- [RFP-022: Bonded Atomic Swaps](./RFP-022-bonded-atomic-swaps.md)
- [RFP-023: Reputation-Based Atomic Swaps](./RFP-023-reputation-atomic-swaps.md)
- [appendix/atomic-swaps-primer.md](../appendix/atomic-swaps-primer.md) —
  free-option framing and locking-order analysis
- [Han et al., On the optionality and fairness of Atomic Swaps, IACR 2019/896](https://eprint.iacr.org/2019/896)
  (accessed 2026-05-22)
- [Gugger, Bitcoin-Monero Cross-chain Atomic Swap, IACR 2020/1126](https://eprint.iacr.org/2020/1126.pdf)
  (accessed 2026-05-22)
- [Hoenisch and del Pino, Atomic Swaps between Bitcoin and Monero, arXiv:2101.12332](https://arxiv.org/abs/2101.12332)
  (accessed 2026-05-22)
