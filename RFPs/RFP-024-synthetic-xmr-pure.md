---
id: RFP-024
title: Synthetic XMR (sXMR), Pure Non-Custodial Design
tier: XL
funding: $TBD
status: draft
category: Applications & Integrations
---

# RFP-024 — Synthetic XMR (sXMR), Pure Non-Custodial Design

> **Note:** This RFP is a *decision-stage draft*. It exists to help the Logos
> team and the community compare cross-chain DEX designs across RFP-021 through
> RFP-025. Hard requirements, team profile, timeline, and contracting details
> are deliberately omitted; they will be filled in if the design is selected for
> funding.

## 🧭 Overview

Build a synthetic XMR token (sXMR) on LEZ that tracks the XMR price via oracle,
is composable across LEZ DeFi, and redeems to real XMR via peer-to-peer atomic
swap. The protocol holds no XMR, runs no signer set, and offers no redemption
SLA.

The wedge: no published, live synthetic redeems to real XMR on Monero L1
*non-custodially, via peer-to-peer atomic swap*. Two distinct prior-art families
exist, neither of which fills this corner:

- **Bridge-custodied real XMR.** Secret Network's Secret Monero Bridge (live
  since August 2021) ran sXMR as a SNIP-20 token bridged via a multi-signature
  Monero wallet operated by consensus-node operators; the trust shape is a
  signer set, not peer-to-peer atomicity. Source:
  [github.com/maxkoda-cpu/Secret-Monero-Bridge](https://github.com/maxkoda-cpu/Secret-Monero-Bridge)
  (accessed 2026-05-22).
- **CDP-collateralised oracle-priced synth, no real XMR.** Synthetix listed sXMR
  on Ethereum L1 (Hadar release, 2020-03-30) as an SNX-collateralised
  oracle-priced ERC-20; the contract held no Monero, only SNX collateral was at
  risk, and the synth was not redeemable to XMR. Source:
  [Synthetix blog: Hadar release](https://blog.synthetix.io/new-synths-update-for-the-upcoming-hadar-release/)
  (accessed 2026-05-22). The Synthetix sXMR and the Secret Network sXMR share a
  ticker but are unrelated products; see
  [appendix/synthetics-design-space.md](../appendix/synthetics-design-space.md)
  §Two unrelated sXMR products.

Other commodity-tracking synthetics (sBTC on Stacks, sETH-style synths) redeem
to transparent assets, leaving the destination on a public ledger. The Haven
Protocol xAsset family (xUSD and other privacy-preserving xAssets) ran on a
Monero-forked L1 from 2018 until project closure on 2024-12-12 (a range-proof
validation vulnerability allowed unbounded illicit minting; >94% of known XHV
supply was controlled by attackers at closure). Haven never offered an xXMR
product; its design was over-collateralised synthetics minted against XHV, not
peer-to-peer atomic-swap redemption. Source:
[Haven Protocol: Project Closure Announcement (2024-12-12)](https://havenprotocol.org/2024/12/12/project-closure-announcement/)
(accessed 2026-05-22). Haven's shutdown is a structural failure mode worth
noting: the same ring-signature properties that protect users prevent
post-incident wallet identification and freezing.

**Trade-off accepted up front.** This RFP deliberately leaves the free option of
the redemption-leg atomic swap unpriced. RFP-022's LEZ bond and RFP-026's
external-chain fee-burn both price that option but at the cost of LP capital
efficiency or refund-branch principal loss. Goal 1's premise is that a
privacy-maximalist user base will tolerate variable redemption availability (LPs
may be slow to show up, spreads may widen under stress) in exchange for the
cleanest non-custody story. The unpriced free option is the explicit cost the
protocol pays for that positioning; LPs bear it. If you want the option priced,
choose RFP-022, RFP-026, or RFP-025 instead.

This RFP positions itself as the first design where the redemption path itself
is both privacy-preserving (deposits real XMR on Monero L1) and non-custodial
(atomic-swap rather than signer-set bridge). See
[appendix/synthetics-design-space.md](../appendix/synthetics-design-space.md)
for the deployed-synthetics survey including the Secret Monero Bridge negative
case, and
[appendix/cross-chain-trust-model-contrast.md](../appendix/cross-chain-trust-model-contrast.md)
for the federated-signer-vs-atomic-swap trust contrast.

The honest framing: this is a synthetic with a soft, market-clearing peg, not a
hard-redeemable synthetic. The oracle is the *quoted* price; the *achievable*
price is whatever an XMR LP will swap for, when one is willing. Structurally
closer to a DEX trading pair than to sBTC (Stacks). The product fits a
privacy-maximalist audience that wants XMR-shaped exposure inside LEZ DeFi
without depending on protocol-held custody.

## Desired properties

- **Non-custodial.** No vault holds XMR. No signer set. No bridge.
- **Soft peg.** The oracle is a reference price; the achievable redemption price
  is whatever an LP quotes. Spread widens under stress without bound.
- **No redemption guarantee.** A counterparty may not exist when the user wants
  to exit. The protocol does not commit to availability.
- **Composable on LEZ.** sXMR is a vanilla LEZ token (the LEZ token program
  standard from RFP-003 hard requirement 7), callable by any other LEZ program:
  lending markets, DEXes, governance, structured products.
- **Private exit.** Successful redemption deposits real XMR on Monero L1,
  severing the public trail. The XMR side never touches LEZ.
- **Open LP set.** Anyone with XMR can quote. No permission required; no LP
  registry; no bond.
- **Regulatory minimalism.** The protocol does not handle XMR; it is, in
  defensible terms, a price feed plus a matching board for users to find each
  other.
- **Off-chain orderbook.** Quotes and intents flow over Logos Delivery (the same
  coordination primitive RFP-003 uses for maker advertisement). The on-chain
  artefact is the atomic-swap settlement program; the matching itself is
  bilateral and off-chain.

## High-level functionality and flow

```
   sXMR LEZ program           oracle (XMR/USD)        Oracle program on LEZ
   (token + stable vault)  <----------------------    
                                                      
       mint  |  burn                                  
                                                      
   Intent gossip                  match               Open XMR LPs
   (off-chain via Logos                               (anonymous; free
   Delivery)                                          to enter/exit)
   sXMR <--> XMR quotes                               
                                                      
            atomic swap (adaptor-sig protocol from RFP-003)
            LEZ <------------------------------> Monero L1
                                                      
   sXMR holder gets XMR                    XMR LP gets sXMR,
   on Monero L1                            burns for stable
```

### Mint

A user deposits a stable (or other LEZ-accepted asset) into the sXMR collateral
vault on LEZ; the protocol mints sXMR to the user at the current oracle price
(with a configurable mint fee). The vault holds the collateral; sXMR circulates
freely.

### Redemption (the privacy-preserving exit)

1. Alice (sXMR holder) signals intent to redeem over Logos Delivery.
2. Bob (open-set XMR LP) sees the intent, computes his quote (oracle price plus
   his spread), and replies bilaterally.
3. Alice accepts; Alice and Bob execute an atomic swap using the RFP-003 LEZ-XMR
   SDK. Alice's sXMR is burned on LEZ; Bob's XMR arrives on Monero at an address
   Alice controls. The matching, the quote, and the bilateral acknowledgement
   happen off-chain over Logos Delivery and Logos Chat.
4. Bob's swapped sXMR is burned by Bob; the released stable collateral becomes
   Bob's payout (less any protocol fee).

### Failure modes (no protocol enforcement)

- If Bob walks away mid-swap, the atomic-swap timelock returns Alice's sXMR; she
  retains her position.
- If no Bob exists, Alice's sXMR trades at a discount to oracle until a Bob
  shows up or Alice gives up. There is no protocol-side compensation for
  redemption delay.

## Pros

- **Cleanest privacy story in the bundle.** Successful redemption ends with real
  XMR on Monero L1. No protocol-side custody, no signer-set deposit-history leak
  (the RFP-021 and RFP-025 option 2b weakness), no view-key disclosure (the
  RFP-022 Tier 2 constraint).
- **Cryptographic non-custody is full.** No vault, no bond, no SLA. The trust
  assumption is the oracle (for pricing) and the soundness of the RFP-003
  atomic-swap protocol. The protocol cannot lose user funds in a custody breach
  because it does not have custody.
- **Composable from day one.** sXMR is a vanilla LEZ token; lending markets and
  DEXes can integrate without coordinating with the sXMR team. The product
  surface scales with the rest of LEZ DeFi.
- **Regulatory defensibility is highest.** The protocol does not handle XMR.
  Operators of price feeds and matching boards have meaningfully different
  exposure from operators of XMR-custodying bridges.
- **Lowest engineering cost in the bundle's synthetics line.** No SLA
  enforcement; no LP registry; no slashing logic; no signer-set custody. The
  core deliverable is a token, a collateral vault, an oracle integration, and an
  intent layer over Logos Delivery.
- **Open LP set is censorship-resistant.** No permissioned set; no KYC; no
  operator that can be coerced into refusing service.

## Cons

- **Soft peg widens under stress without bound.** The achievable redemption
  price is the marginal LP quote. If LPs vanish, sXMR can trade at any discount
  to oracle; the protocol cannot intervene.
- **No redemption SLA.** Users who want guaranteed redemption (institutions,
  market makers, structured products) cannot use sXMR directly under Goal 1;
  they need RFP-025.
- **Predicted demand asymmetry.** Mint demand is expected to be easy
  (privacy-curious DeFi users want XMR exposure inside LEZ); LP supply is
  expected to be harder to source (XMR maximalists may not want a public LP role
  at all, even pseudonymous). The LP side is the predicted structural
  bottleneck; applicants should validate against early LP onboarding before
  scaling.
- **Predicted adverse selection of LPs.** LPs are expected to show up when
  oracle is below true market XMR price (free money for them on the redemption
  leg) and to vanish when oracle is above (would-be loss). Redemption
  availability is expected to be asymmetric across regimes; analogous to
  uncollateralised peer-to-peer market-making in other venues but unverified for
  this product.
- **Atomic-swap UX inherited.** Settlement time is dominated by Monero block
  confirmations, typically under an hour but with variance from network
  conditions; both parties online for the duration. The intent layer over Logos
  Delivery softens this but cannot remove it.
- **No protocol-side enforcement of LP behaviour.** Refusing to proceed is
  *valid behaviour* under the atomic-swap protocol; the protocol cannot
  distinguish malicious refusal from connectivity loss. Reputation (RFP-023) is
  the only available restraint, and even then it operates as soft pressure, not
  enforcement.
- **Oracle dependency.** The oracle is the only price signal; oracle attack or
  stale price degrades minting accuracy. Mitigation lives outside this RFP
  (existing oracle RFPs in the bundle, or oracle aggregation patterns).

## Risks

- **LP exodus.** All XMR holders stop quoting. sXMR trades at an indefinite
  discount to oracle until LPs return. Mitigation: design the protocol to
  survive long discount regimes without auto-liquidation; let the discount be
  the market signal that restores LP supply.
- **Oracle manipulation.** A manipulated XMR/USD oracle lets an attacker mint
  sXMR cheaply or under-collateralise existing positions. Mitigation: use a
  redundant oracle stack with median-of-N pricing; impose configurable
  price-deviation guards on minting.
- **Collateral solvency.** If the collateral asset (stables or otherwise) is
  itself compromised (depeg, exploit, regulatory freeze), sXMR backing degrades.
  Mitigation: accept only collateral with documented threat models; cap
  protocol-wide collateral concentration by asset.
- **Privacy-claim overreach.** sXMR itself is *not* a private asset on LEZ (the
  token program is public state). The privacy property applies *only* to the
  redemption-to-XMR path. If users mint sXMR from a public LEZ account and never
  redeem, no privacy is conferred. The communication challenge is non-trivial;
  documentation must be honest.
- **Regulatory inflection on Monero.** If a jurisdiction outlaws Monero
  entirely, the redemption path becomes legally fraught for LPs in that
  jurisdiction. The protocol itself is insulated (it does not handle XMR) but
  the LP economy may concentrate in friendlier jurisdictions. Strategic, not
  technical.
- **Atomic-swap maker burnout.** The free-option problem RFP-022 addresses
  applies here too: LPs can be free-optioned by takers. Without bonds (Goal 1's
  premise), this can drive LPs away. Mitigation: optional layered consumption of
  RFP-022 Tier 2 (bonded XMR atomic swaps) or RFP-023 (reputation) for the
  redemption leg, as separate products on top of the same sXMR token.
- **No protocol-side fee revenue capture.** Open LP set means LPs capture the
  spread; the protocol's only revenue is mint and burn fees on sXMR itself.
  Sustainability depends on mint/burn volume, which depends on LEZ DeFi
  composability adoption.

## Relationship to other RFPs in this bundle

- **RFP-003 (Atomic Swaps with LEZ, open)** is the foundation: the LEZ-XMR
  atomic-swap SDK is the redemption settlement layer. RFP-024 does not modify
  RFP-003; it builds the sXMR product on top of it.
- **RFP-025 (sXMR with SLA)** is the complementary design. RFP-024 targets the
  privacy-maximalist audience accepting market-clearing redemption; RFP-025
  targets the SLA-needing audience accepting custody risk. The two together
  cover the synthetics design space. A reader should pick one or both based on
  which user segment they want to serve.
- **RFP-022 (bonded atomic swaps)** could optionally be consumed by RFP-024 as
  the redemption-leg primitive: instead of bare atomic swaps, redemption uses
  bonded atomic swaps. This compounds LP commitment but adds bond friction on
  the LP side. The pure design in this RFP does not require this layering.
- **RFP-023 (reputation-based atomic swaps)** could optionally be consumed for
  LP-discovery UX (takers see LP reputation before initiating). Not strictly
  required for the core product.
- **RFP-021 (cross-chain privacy DEX)** is orthogonal. RFP-021 offers real-XMR
  cross-chain swaps with federated custody; RFP-024 offers synthetic-XMR
  exposure with atomic-swap redemption. Different products; same broad audience
  can use both.
- **RFP-004 (Privacy-Preserving DEX, open)** is the natural single-chain trading
  venue for sXMR once minted. Trade sXMR against stables on RFP-004's AMM pools
  under the deshield-swap-reshield pattern; redeem to real XMR via RFP-024.

See
[appendix/synthetics-design-space.md](../appendix/synthetics-design-space.md)
for the deployed-synthetics survey (Haven, Synthetix, sBTC, Secret Monero
Bridge) and the privacy-coin specific constraints. See
[appendix/atomic-swaps-primer.md](../appendix/atomic-swaps-primer.md) for
atomic-swap mechanics and the free-option framing relevant to the redemption-leg
LP economics.

## References

- [RFP-003: Atomic Swaps with LEZ](./RFP-003-atomic-swaps.md)
- [appendix/synthetics-design-space.md](../appendix/synthetics-design-space.md)
- [appendix/atomic-swaps-primer.md](../appendix/atomic-swaps-primer.md)
- [appendix/cross-chain-trust-model-contrast.md](../appendix/cross-chain-trust-model-contrast.md)
- [Bitcoin to Monero atomic swaps (getmonero.org, 2021-08-20)](https://www.getmonero.org/2021/08/20/atomic-swaps.html)
  (accessed 2026-05-21)
- [eigenwallet/core (active fork of comit-network/xmr-btc-swap; v4.6.1, 2026-05-15)](https://github.com/eigenwallet/core)
  (accessed 2026-05-21)
- [Secret Network Monero Bridge (custodial XMR-to-Secret Network bridge, deployed prior art)](https://github.com/maxkoda-cpu/Secret-Monero-Bridge)
  (accessed 2026-05-22)
- [Haven Protocol documentation](https://docs.havenprotocol.org) (accessed
  2026-05-22)
- [docs.stacks.co/concepts/sbtc](https://docs.stacks.co/concepts/sbtc) (accessed
  2026-05-22)
- [Hiro: Who are the sBTC signers, breaking down SIP-028](https://www.hiro.so/blog/who-are-the-sbtc-signers-breaking-down-sip-028)
  (accessed 2026-05-22)
- [Synthetix blog: New Synths update for the upcoming Hadar release](https://blog.synthetix.io/new-synths-update-for-the-upcoming-hadar-release/)
  (accessed 2026-05-22)
- [Haven Protocol: Project Closure Announcement (2024-12-12)](https://havenprotocol.org/2024/12/12/project-closure-announcement/)
  (accessed 2026-05-22)
