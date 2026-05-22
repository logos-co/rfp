---
id: RFP-025
title: Synthetic XMR (sXMR) with Redemption SLA
tier: XL
funding: $TBD
status: draft
category: Applications & Integrations
---

# RFP-025 — Synthetic XMR (sXMR) with Redemption SLA

> **Note:** This RFP is a *decision-stage draft*. It exists to help the Logos
> team and the community compare cross-chain DEX designs across RFP-021 through
> RFP-025. Hard requirements, team profile, timeline, and contracting details
> are deliberately omitted; they will be filled in if the design is selected for
> funding.

## 🧭 Overview

Build a synthetic XMR token (sXMR) on LEZ with a redemption SLA: users can
redeem sXMR for real XMR at oracle price, on demand, up to the protocol's
committed capacity. The atomic swap is still the settlement primitive, but
counterparty availability is no longer left to the open market.

The RFP requires applicants to commit to one of two sub-designs:

- **Option 2a: Bonded LP set.** LPs join a registered set on LEZ, post stable
  collateral as bond, and are obligated to honour redemption requests within a
  window. Default triggers bond slashing paid to the redeemer. Non-custodial in
  the strict sense (LPs hold their own XMR) but bonded for performance.
- **Option 2b: Protocol XMR reserve.** The protocol accumulates an XMR reserve
  (from mint fees, a yield programme, or a one-time treasury seed) held in a
  threshold-signer multisig on Monero. Redemption draws directly from the
  reserve. Custodial: trust lives in the signer set.

The RFP exists so applicants can argue for one of the two and the Logos team can
pick. The two options have different threat models, different LP economics, and
different regulatory exposures. They cannot be combined cleanly: option 2b's
reserve and option 2a's open-but-bonded LP set are different protocols sharing
only the sXMR token name.

This RFP is the marketable companion to RFP-024 (sXMR pure non-custodial). A
reader should pick at least one of the two, and may pick both, depending on
which user segment they want to serve.

## Desired properties (both options)

- **Redemption SLA.** A user redeeming sXMR for XMR receives real XMR on Monero
  L1 within a documented window (e.g. 60 minutes for option 2a; instant
  settlement subject to atomic-swap timelock for option 2b).
- **Oracle-priced redemption.** Redemption price is within a documented
  tolerance of the oracle reference; not subject to LP-quote dispersion.
- **Composable sXMR token on LEZ.** Vanilla LEZ token (the LEZ token program
  standard from RFP-003 hard requirement 7), callable by lending, DEXes,
  governance, structured products.
- **Private exit.** Successful redemption ends with real XMR on Monero L1. The
  XMR side never leaks the redeemer's LEZ-side identity beyond the redemption
  flow itself.
- **Bondless taker entry path.** First-time redeemers (the taker side, from the
  atomic-swap perspective) are not required to post a LEZ-denominated bond for
  the first redemption up to a capped notional (worked example: US$100
  equivalent). After the first redemption, the taker has LEZ-denominated assets
  they can post against subsequent redemptions. The bond on the *LP side*
  (option 2a) or the *reserve* (option 2b) is unaffected; those are maker-side
  or protocol-side constructs.

## Option 2a: bonded LP set

LPs join a registered set on LEZ. Each LP posts stable collateral on LEZ equal
to (or some multiple of) their XMR commitment. When a redemption request is
routed to an LP, they must complete the atomic swap within a window. If they
default, their bond is slashed and paid to the redeemer. LPs may leave the set,
but only after a notice period that exceeds the redemption SLA.

**Inspired by existing prior art.** Option 2a's bonded-LP design is closest in
shape to **Thorchain's bonded-validator model**: a set of operators each post
slashable stake (Thorchain RUNE bonded ≥ ~2× pooled liquidity) and the protocol
enforces performance on the stake. See
[appendix/cross-chain-trust-model-contrast.md](../appendix/cross-chain-trust-model-contrast.md)
§Federated-signer middle chain for the Thorchain trust-model survey including
the May 2026 GG20/TSSHOCK exploit. The Logos adaptation differs in two respects:
(1) the bonded operators are *XMR sellers committing to redemption SLA*, not
protocol validators co-signing custody; (2) the LP bond is locked for the LP's
tenure plus the notice period, *separate* from any per-swap collateralisation.
The bond is a persistent performance bond against the LP's SLA across many
swaps, not a per-swap collateral. Option 2a inherits Thorchain's structural
problem — slashing requires attribution, and the protocol cannot adjudicate the
attribution from atomic-swap state alone (see the Enforceability caveat below).

```
                    sXMR LEZ program
                    + LP registry
                    + slashing logic (depends on an off-chain
                      attribution layer; see LP-0019)
                    
       redemption                          LP bond
       request                             
                                           
       sXMR holder                         Bonded LP
       burns sXMR,         atomic swap     posts stable bond
       receives XMR    <---------------->  delivers XMR or
                          (adaptor-sig)    forfeits bond
                                           
       if LP defaults: bond paid out as compensation,
       attribution layered via LP-0019 reputation system
```

**Enforceability caveat.** "LP defaulted" is not a verdict an on-chain program
can render from atomic-swap state alone (per the cross-cutting analysis in
[appendix/cross-chain-trust-model-contrast.md](../appendix/cross-chain-trust-model-contrast.md)).
Refusing to proceed is *valid behaviour* under the atomic-swap protocol; the LEZ
contract cannot distinguish malicious refusal from connectivity loss. The
realistic implementation depends on the off-chain reputation system that lambda
prize [LP-0019](../lambda-prizes/LP-0019-atomic-swap-maker-reputation.md) is
expected to deliver: an LP who repeatedly fails to honour redemptions loses
reputation; the bond is slashed against the *attested* default condition, not
against atomic-swap state directly. Without LP-0019 (or an equivalent off-chain
attestation mechanism), the bond can be used only to gate participation
(priority, fee tiers, future-slot access), not slashed with cryptographic
certainty on a single failed swap.

## Option 2b: protocol XMR reserve

The protocol holds an XMR reserve in a threshold-signer multisig on Monero.
Redemption draws from the reserve directly, with the atomic swap acting as the
settlement rail between the reserve custodian and the redeemer.

```
                    sXMR LEZ program
                    + reserve accounting
                    
       burn sXMR                    trigger swap
                                       
                              Reserve module
                              (LEZ program)
                                       
                              atomic swap
                              (adaptor-sig)
                                       
                  Threshold-signer reserve on Monero
                  (n-of-m, bonded signers, view-key-shared)
```

**Inspired by existing prior art.** Option 2b directly adopts **sBTC's (Stacks)
threshold-signer custody model**: a federation of bonded signers (15-signer, 70%
threshold in SIP-028; 14 currently operating with 10-of-14) holds the underlying
asset on its native chain and produces redemptions on-demand. See
[appendix/synthetics-design-space.md](../appendix/synthetics-design-space.md)
§Redeem-to-underlying with custody for the sBTC trust-shape survey. Option 2b
adds an **oracle-priced peg layer** on top of the sBTC-style custody, replacing
sBTC's 1:1 redemption with oracle-tracked redemption. The structural overlap
with sBTC is the custody side; the peg semantics differ. The same
view-key-shared TSS custody constraint applies as in RFP-021: honest-but-curious
signers learn the protocol-side deposit history. This is the structural
trade-off option 2b accepts in exchange for the redemption SLA. The signer set
must be bonded and slashable to make the trust assumption explicit.

The TSS custody design itself follows **Serai's FROST-over-CLSAG** approach for
the Monero case (rather than Thorchain's GG20 ECDSA, which suffered the May 2026
TSSHOCK exploit; see
[appendix/cross-chain-trust-model-contrast.md](../appendix/cross-chain-trust-model-contrast.md)
§Federated-signer middle chain). Serai is pre-mainnet as of 2026-05; option 2b
applicants should track Serai's monero-oxide work as the production-ready
FROSTLASS instantiation.

## High-level functionality and flow (common)

### Mint

Identical to RFP-024. User deposits accepted LEZ collateral; protocol mints sXMR
at oracle price; collateral sits in vault.

### Redemption (option 2a)

1. Alice (sXMR holder) submits a redemption request to the LEZ sXMR program.
2. Program routes the request to a bonded LP in the registry (round-robin,
   reputation-weighted, or auction-priced).
3. LP receives the request; LP and Alice execute an atomic swap via the RFP-003
   LEZ-XMR SDK within the SLA window.
4. On success, Alice's sXMR is burned, LP claims the released collateral as
   their payout.
5. On failure (LP times out): bond slashing is triggered. LP-0019 off-chain
   reputation attestation establishes that the failure was the LP's fault (not
   the redeemer's); slashed bond is paid to the redeemer as compensation; LP's
   reputation is decremented.

### Redemption (option 2b)

1. Alice submits a redemption request.
2. Program signals the reserve module; threshold signers co-sign a Monero spend
   from the reserve to a destination Alice supplies.
3. Atomic-swap protocol enforces the conditional: Alice's sXMR is burned only if
   the Monero spend lands on chain.
4. Reserve accounting updates; if the reserve is depleted below a safety
   threshold, mints are paused until the reserve is replenished from mint fees
   or treasury.

## Pros (both options)

- **Redemption SLA is a real product feature.** Institutions, market makers, and
  structured-products integrators can underwrite sXMR positions because they
  have a documented exit window.
- **Hard peg within capacity.** Redemption price tracks oracle within a
  documented tolerance, not LP-quote dispersion. The sXMR token becomes usable
  as collateral inside other LEZ programs with predictable mark-to-market.
- **Composable from day one.** Vanilla LEZ token; the SLA is what makes
  downstream integrations viable.
- **Private exit preserved.** Successful redemption ends with real XMR on Monero
  L1, regardless of which option is chosen. Privacy on the destination chain is
  intact.
- **Bondless taker entry path.** Privacy-seeking redeemers without LEZ assets
  can complete a first capped redemption without bond friction.

## Pros (option 2a-specific)

- **Non-custodial in the strict sense.** LPs custody their own XMR; the protocol
  does not hold the underlying asset. No vault to drain.
- **Decentralised LP set.** Anyone meeting the bond requirement can join the
  registered set.
- **Slashing bounds the loss.** A defaulting LP's bond becomes the redeemer's
  compensation; protocol-wide solvency is bounded by total bonded capacity.
- **No signer-set custody risk for sXMR holders.** Trust assumption is the
  bonded-LP set's collective behaviour, which is bounded by the bond size, not
  by signer-set integrity.

## Pros (option 2b-specific)

- **Strongest SLA guarantee.** Redemption draws from a protocol-managed reserve;
  no LP discovery, no per-redemption matching. Settlement is deterministic up to
  the atomic-swap protocol's timelock.
- **Predictable redemption capacity.** Bounded by the reserve size; capacity
  planning is governance-driven rather than LP-supply-driven.
- **Lower operational complexity for redeemers.** Single counterparty (the
  reserve module), not a matched LP.

## Cons (both options)

- **Some form of trust returns.** RFP-024 (pure) trusts only the oracle and the
  atomic-swap protocol; RFP-025 trusts additionally an LP set (option 2a) or a
  signer set (option 2b). The cypherpunk story is weaker.
- **Atomic-swap UX is still inherited.** Settlement time is dominated by Monero
  block confirmations, typically under an hour but with variance from network
  conditions; both parties online for the duration. The SLA constrains the
  *availability* of the counterparty but not the cryptographic settlement time.
- **Oracle dependency is sharper.** With an SLA on oracle-priced redemption, an
  oracle failure has SLA-breaking consequences, not just mint-side accuracy
  consequences.
- **Regulatory exposure is higher.** A protocol that commits to redeeming a
  privacy coin on demand draws more scrutiny than RFP-024's
  price-feed-plus-matching-board posture.

## Cons (option 2a-specific)

- **Slashing requires off-chain attribution.** The LEZ contract cannot
  adjudicate "LP defaulted" from atomic-swap state. LP-0019 off-chain reputation
  is the realistic attribution mechanism; without it, the bond gates
  participation but does not slash on single defaults.
- **Bond opportunity cost limits LP supply.** Locking stable collateral against
  XMR commitment is expensive; LP yield must clear the opportunity cost.
  Realistic capacity is constrained by the LP economy's appetite for the
  bond/yield trade-off.
- **Coordinated default can exceed bonded capacity.** If many LPs default
  simultaneously (e.g. during a Monero price shock), aggregate slashing may not
  cover aggregate redemption demand. The peg breaks.
- **LP notice period limits flexibility.** LPs must wait out a notice period
  exceeding the SLA before exiting the set; this raises the cost of becoming an
  LP.

## Cons (option 2b-specific)

- **Custodial.** A signer set holds real XMR on Monero. Custody risk is real:
  signer collusion, key compromise, or signing-software bug can drain the
  reserve. This is the failure mode that hit Thorchain on 2026-05-15 (TSS
  implementation weakness, $10.8M). The Wormhole February 2022 incident ($326M)
  is a related-but-distinct category: a per-chain bridge-contract bug
  (`load_instruction_at` on Solana) bypassed the signer set entirely rather than
  compromising it; the lesson is that per-chain contract surface adds attack
  vectors independent of the TSS itself.
- **View-key-shared custody leaks Monero deposit history to signers.** The same
  compromise RFP-021 makes. Honest-but-curious signers learn the protocol-side
  Monero deposit history; this is the structural cost of TSS custody of XMR with
  current cryptography.
- **Adopts sBTC (Stacks)'s threshold-signer custody model, with an oracle-priced
  peg replacing 1:1 redemption.** The custody novelty is small relative to
  existing custodial XMR wraps (the differentiator is the LEZ privacy execution
  on the sXMR token side, plus the explicit slashable-signer-bond posture).
- **Reserve undercollateralisation breaks the peg.** If the reserve is drawn
  down faster than it can be replenished from fees, redemption capacity goes to
  zero; sXMR loses its peg.
- **Signer-set membership is gated.** Lower decentralisation than option 2a;
  signer set is a censorship and coercion target.

## Risks (both options)

- **Oracle manipulation.** A manipulated XMR/USD oracle lets an attacker mint
  sXMR cheaply or extract reserve XMR (option 2b) at unfavourable rates.
  Mitigation: redundant oracle stack with median-of-N pricing; configurable
  price-deviation guards; SLA-aware oracle staleness checks.
- **Regulatory action.** Either option may attract jurisdiction-specific bans on
  the protocol, LP participation, or reserve custody. Mitigation: jurisdictional
  diversity in LP set or signer set; documented compliance posture; willingness
  to operate as a permissionless smart contract regardless of any single
  jurisdiction's stance.
- **First-swap cap evasion.** A redeemer could split a large redemption into
  many capped first-redemptions under fresh pseudonyms. Mitigation: rate limits
  enforced at the LEZ escrow program; combine with off-chain reputation gating
  (LP-0019) for higher tiers.

## Risks (option 2a-specific)

- **Off-chain reputation dependency (LP-0019).** Without an off-chain
  attribution mechanism, the bond is not actually slashable on default. If
  lambda prize
  [LP-0019](../lambda-prizes/LP-0019-atomic-swap-maker-reputation.md) is not yet
  awarded when option 2a ships, the protocol launches with a weaker enforcement
  story than its marketing implies. Mitigation: sequence LP-0019 first, or
  include a stub-attestor mechanism in RFP-025 itself (with the limitations of
  stub-attestor centralisation documented honestly).
- **Reputation gaming attacks on the LP side.** An LP can build reputation
  cheaply by completing many small redemptions then default on a large one.
  Mitigation: notional-weighted reputation; cap per-LP redemption size
  proportional to accumulated reputation and bond.
- **Bond denomination volatility.** If the bond asset (stables on LEZ) depegs,
  LPs become under-collateralised against their XMR commitments. Mitigation: cap
  protocol-wide LP collateral concentration by asset; require LPs to top up on
  bond-asset depeg events.

## Risks (option 2b-specific)

- **Signer-set compromise.** A captured signer set can drain the entire reserve.
  Mitigation: large signer set (Serai uses up to 600; Thorchain ~100);
  permissionless entry; high bond-to-custodied ratio; emergency halt mechanism;
  geographic diversity.
- **Signer-set offline event.** If the signer set cannot reach threshold to sign
  (network partition, mass node failure), redemption SLA breaks even with no
  malice. Mitigation: redundant signer geographic placement; documented signer
  SLOs; emergency-halt mechanism that pauses redemption gracefully rather than
  failing under load.
- **TSS implementation bug.** Thorchain's GG20 TSS exploit on 2026-05-15
  ($10.8M) is the canonical example. Mitigation: choose FROST over GG20; budget
  for Cypher Stack-equivalent audit; isolate signer-software dependencies.

## Relationship to other RFPs in this bundle

- **RFP-024 (sXMR pure non-custodial)** is the complementary design. RFP-025
  trades non-custody for SLA. A reader should pick one or both based on the
  target audience: pure-cypherpunk users for RFP-024; SLA-needing users for
  RFP-025.
- **RFP-003 (Atomic Swaps with LEZ, open)** is the foundation: the LEZ-XMR
  atomic-swap SDK is the redemption settlement layer for both options.
- **Lambda prize
  [LP-0019 (off-chain maker reputation)](../lambda-prizes/LP-0019-atomic-swap-maker-reputation.md)**
  is a hard dependency for option 2a's slashing mechanism. The off-chain
  attribution it produces is what makes "LP defaulted" attributable; without it,
  the bond gates participation but cannot be slashed on a single failed swap
  with cryptographic certainty.
- **Lambda prize
  [LP-0018 (atomic-swap anti-spam mechanism)](../lambda-prizes/LP-0018-atomic-swap-anti-spam.md)**
  could optionally be layered on the redemption-leg atomic swap to deter
  taker-side griefing. Not strictly required for either option; the LP-side bond
  / reserve already addresses maker-side performance.
- **RFP-021 (cross-chain privacy DEX)** is orthogonal: it offers real-asset
  cross-chain swaps with federated custody; this RFP offers synthetic-XMR
  exposure with managed redemption. Option 2b shares the view-key-shared TSS
  custody trade-off with RFP-021's XMR support; the two RFPs could share
  signer-set infrastructure if both are funded.
- **RFP-004 (Privacy-Preserving DEX, open)** is the natural single-chain trading
  venue for sXMR.

See
[appendix/synthetics-design-space.md](../appendix/synthetics-design-space.md)
for the deployed-synthetics survey (Haven, Synthetix, sBTC, Secret Monero
Bridge) including the redeem-to-underlying-with-custody design family (option
2b's pattern). See
[appendix/cross-chain-trust-model-contrast.md](../appendix/cross-chain-trust-model-contrast.md)
for the federated-signer custody analysis that applies to option 2b.

## References

- [RFP-003: Atomic Swaps with LEZ](./RFP-003-atomic-swaps.md)
- [appendix/synthetics-design-space.md](../appendix/synthetics-design-space.md)
- [appendix/cross-chain-trust-model-contrast.md](../appendix/cross-chain-trust-model-contrast.md)
- [appendix/atomic-swaps-primer.md](../appendix/atomic-swaps-primer.md)
- [docs.stacks.co/concepts/sbtc](https://docs.stacks.co/concepts/sbtc) (accessed
  2026-05-22) — sBTC is a 1:1 BTC-backed asset on Stacks; custody is a 15-signer
  federation with a 70% threshold (current operating set 14 signers, 10-of-14 to
  sign peg-out); withdrawal latency ~6 Bitcoin blocks.
- [Hiro: Who are the sBTC signers, breaking down SIP-028](https://www.hiro.so/blog/who-are-the-sbtc-signers-breaking-down-sip-028)
  (accessed 2026-05-22)
- [Crypto Times: $10.8M drained from Thorchain on 2026-05-15](https://www.cryptotimes.io/2026/05/17/10-8-million-drained-inside-the-thorchain-exploit-that-froze-cross-chain-defi-for-13-hours/)
  (accessed 2026-05-19)
- [Halborn: Wormhole Hack on 2022-02-02 (technical analysis)](https://www.halborn.com/blog/post/explained-the-wormhole-hack-february-2022)
  (accessed 2026-05-19)
- [FROST: Flexible Round-Optimized Schnorr Threshold Signatures (Komlo and Goldberg, SAC 2020 / IACR 2020/852)](https://eprint.iacr.org/2020/852)
  (accessed 2026-05-21)
