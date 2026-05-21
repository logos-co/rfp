---
id: RFP-022
title: Bonded Atomic Swaps (Two Tiers)
tier: XL
funding: $TBD
status: draft
category: Applications & Integrations
---

# RFP-022 — Bonded Atomic Swaps (Two Tiers)

> **Note:** This RFP is a *decision-stage draft*. It exists to help the Logos team and the community compare cross-chain DEX designs across RFP-021 through RFP-025. Hard requirements, team profile, timeline, and contracting details are deliberately omitted; they will be filled in if the design is selected for funding.

## 🧭 Overview

Extend RFP-003 (Atomic Swaps with LEZ, open) with a maker/taker bond layer on LEZ that constrains the free-option problem inherent to atomic swaps. Bonds are posted on LEZ in stables or LEZ-native assets; slashing is conditioned on LEZ-observable failures to advance through the swap state machine.

The design splits into two tiers that reflect a structural asymmetry in the underlying cryptography:

- **Tier 1 (LEZ to BTC, LEZ to ETH).** Both sides' locks are verifiable on LEZ via a chain-watching light-client module. Both Alice's and Bob's bonds are slashable on default; full bilateral free-option mitigation.
- **Tier 2 (LEZ to XMR).** Alice's XMR lock cannot be proven on LEZ without view-key disclosure to public state, because Monero has no SPV-style proof primitive that does not reveal the per-tx private key and blinding factor. Bob's lock (on LEZ) remains observable, so Bob's bond is slashable; Alice's bond is slashable only on her LEZ-observable abandonment (failure to reveal after Bob has locked Logos). Alice keeps a residual pre-XMR-lock free option that only reputation (RFP-023) can constrain.

The Bond layer is a strict superset of RFP-003. Builders should consume the per-pair SDKs from RFP-003 unchanged; this RFP adds the bond escrow contract, the bond accounting, and the LEZ-side proof verification primitives.

## Desired properties

- **Non-custodial.** No vault holds external assets; no signer set. Bonds live in LEZ-native assets on LEZ.
- **Free-option mitigation (Tier 1).** Symmetric bonding makes both sides' optionality strictly EV-negative when bonds are sized above the option value (`σ × √T × notional`, around 2 to 5% of trade notional for 1-hour windows).
- **Free-option mitigation (Tier 2).** Bob's optionality is closed by his slashable bond. Alice's post-Bob-lock optionality is closed by her bond. Alice's pre-XMR-lock optionality is *not* closed; this is the structural limit of the asymmetry.
- **Unauthenticated proof submitter.** Either party can broadcast the other's signed lock transaction (broadcasting is permissionless on every supported chain). The LEZ inclusion-proof submitter is also unauthenticated. This eliminates "attest or be slashed" grief vectors: a malformed lock simply never lands, the state machine times out, no slashing dispute occurs.
- **Bondless taker entry path.** First-time takers can complete a capped first swap (worked example: US$100 equivalent notional) without posting a taker bond. After the first swap, the taker has LEZ-denominated assets they can post as bond against larger swap sizes. This is enforceable by the LEZ escrow program directly; no reputation registry needed.
- **Upgrade clause for Tier 2.** When a non-disclosing Monero proof primitive becomes production-ready (FCMP++ or equivalent), Tier 2 collapses into Tier 1: Alice's XMR lock becomes verifiable on LEZ without view-key disclosure, and the residual free option closes.
- **Composes with RFP-023 reputation.** Maker reputation (and zk-membership-proof taker reputation if available) compounds the cost of defection. In Tier 2 specifically, taker reputation is load-bearing because it is the only restraint on Alice's pre-lock free option.

## High-level functionality and flow

### Tier 1: LEZ to BTC (example)

```
Phase                  Alice              LEZ contract           Bob
                                                                       
0. Quote               <----------- Logos Delivery ---------->
                       Joint-key setup for 2-of-2 Taproot output
                       
1. Commit              post B_alice
                       (LEZ-side bond)    -->  receives B_alice
                       
2. Lock-BTC            sign BTC lock tx --> over Waku -->         broadcast to BTC
                                                                  
                       <-- inclusion proof from anyone --> verifies PoW, merkle, scriptpubkey, amount
                       
                       slash window opens                         if Bob does not advance:
                                                                  B_bob_slice goes to Alice
                       
3. Lock-Logos          (waits)            <-- receives           Bob locks trade_amount
                                          trade_amount +         + B_bob_slice
                                          B_bob_slice            conditioned on secret s
                       
4. Reveal              publishes adaptor sig --> reveals s
                                                                  if Alice does not reveal:
                                                                  B_alice goes to Bob
                       
5. Settle              <-- Alice's bond                          Bob claims BTC using s
                       refunds
                       Bob's bond slice
                       releases
```

The unauthenticated proof submitter property: Bob can broadcast Alice's signed BTC lock himself; the LEZ inclusion-proof submitter is also unauthenticated (Bob, Alice, or a watchtower service can post the proof). If Alice signs a malformed lock (wrong amount, wrong scriptpubkey), Bob does not broadcast; the tx never lands on Bitcoin; the inclusion proof never materialises; the LEZ state machine quietly times out. No slashing dispute, no fraud-proof window. The swap fails closed because the precondition for state advancement (a real BTC lock) never holds.

### Tier 2: LEZ to XMR

Same phase structure, with the following differences:

- Phase 2 (Lock-XMR): Alice's XMR lock is *not* verifiable on LEZ. The state machine cannot transition from Commit to Lock-Logos based on observation of Alice's Monero lock. Bob detects Alice's lock by running a Monero wallet himself (the bilateral `check_tx_proof` she sends him privately, which does *not* go on LEZ). Bob's decision to advance to Lock-Logos is off-chain.
- If Alice never locks XMR after Commit, the state machine times out; both bonds refund (no LEZ-observable default). Alice has paid only gas; she keeps her pre-XMR-lock free option.
- Phase 3 onwards: Bob's lock on LEZ is observable. From this point forward, Tier 2 matches Tier 1: Alice's bond is slashable on failure to reveal; Bob's bond is slashable on failure to complete after the reveal.

### Bondless taker entry path

A taker without LEZ-denominated assets initiates a "first-swap" mode flagged in the LEZ escrow program:

- Trade notional capped at a small value (worked example: US$100 equivalent), sized against expected free-option value at the protocol's typical lock window.
- No taker bond required.
- After completion, the taker has LEZ-denominated assets in their account from the swap proceeds. They can post these as bond against subsequent larger swaps.
- The cap is enforced by the LEZ escrow program; no reputation registry is required to make the cap binding. This decouples the bondless entry path from the (more complex) reputation infrastructure in RFP-023.

## Pros

- **Closes the free-option problem cryptoeconomically for BTC and ETH (Tier 1).** No bilateral counterparty trust, no third-party attestation, no validator federation. The slash is enforced by the LEZ smart contract directly off the on-chain state of both chains.
- **Preserves the non-custodial property of atomic swaps.** No vault to drain, no TSS to break, no validator set to compromise. Funds never leave Alice's or Bob's control during the swap.
- **Builds cleanly on RFP-003.** Per-pair SDKs and the LEZ-side Risc0 escrow programs from RFP-003 are reused; this RFP layers on the bond escrow and the proof verification primitives. The dependency chain is clean.
- **Material improvement for the LEZ to XMR pair (Tier 2) on the maker side.** Bob's free option is closed even though Alice's lock is not verifiable on LEZ. This unblocks a category of makers who today refuse to post against atomic-swap takers because they can be free-optioned.
- **Bondless taker entry path solves the onboarding chicken-and-egg.** A privacy-seeking taker arriving from XMR or BTC does not need to acquire LEZ assets before their first swap. They complete a small first swap, accumulate LEZ assets, and bond from there. No KYC-tolerant on-ramp required.
- **Upgrade-clean for FCMP++.** When the non-disclosing Monero proof primitive ships, Tier 2 collapses into Tier 1 with no protocol-level rewrite. The RFP carries an explicit upgrade clause.

## Cons

- **Does not fix settlement time.** Settlement is still bounded by source-chain finality plus LEZ finality plus the timelock window. Hours, not minutes. The bond does not accelerate cryptographic settlement.
- **Does not fix interactivity.** Both parties must be online to lock, reveal, and (if the other side defaults) submit the slash claim. The bond removes the incentive to grief but not the requirement to participate.
- **Per-trade matching, no AMM.** No protocol-owned liquidity, no AMM pricing. Each swap requires a willing counterparty for the exact pair and exact size. RFP-021 wins decisively on liquidity gravity.
- **Bond opportunity cost.** Makers must lock LEZ-denominated bond capital, which yields nothing during the lock window. This raises maker spreads relative to the unbonded (free-option) atomic swap of RFP-003.
- **Bond denomination friction.** First-time takers need LEZ-denominated bond assets. The bondless-taker capped-entry path mitigates this but only for the first swap.
- **Tier 2 is structurally weaker.** Alice retains a pre-XMR-lock free option on the LEZ to XMR pair. Reputation (RFP-023) is the only available restraint on this option under current Monero cryptography. Users must understand the asymmetry.
- **More complex than RFP-003.** Bond accounting, slash conditions, light-client modules, dispute windows. The protocol surface and audit surface both grow.

## Risks

- **Cross-chain bond correlation.** If Bob is matched against N concurrent swaps and the LEZ chain re-orgs or his observer crashes, all N swaps may slash him. Mitigation: per-maker concurrency caps; bond scaling with active-swap count; explicit re-org tolerance windows.
- **Light-client implementation risk (Tier 1).** The BTC and ETH light-client modules are the load-bearing primitive. A bug that lets an attacker submit a false inclusion proof is a direct theft vector. Mitigation: fork from well-audited references (ZeroSync, Citrea Clementine LCP for BTC; Nimbus-derived for ETH); independent audit budget.
- **Bond sizing parameter risk.** Bond too small leaves residual optionality; bond too large prices honest makers out of the market. Volatility regime changes (e.g. XMR price moves of 20% in a session) widen the option value. Mitigation: protocol-adjustable bond parameters; optional volatility-indexed bond sizing.
- **Adversarial bond-bootstrap attack.** An attacker who controls the first set of makers can credibly claim "reputation-rich" status and capture taker flow. Mitigation: combine bond requirements with reputation accrual (RFP-023) so reputation cannot be purchased without time-and-capital cost.
- **FCMP++ upgrade slippage (Tier 2).** If the non-disclosing Monero proof primitive does not ship on the expected horizon, Tier 2 remains permanently asymmetric. Mitigation: design the protocol assuming Tier 2 is the steady state; treat FCMP++ as an optional improvement, not a dependency.
- **First-swap cap evasion.** A taker could split a large trade into many capped first swaps under fresh pseudonyms. Mitigation: cap by IP, device fingerprint (weak), or by Sybil-resistant identity proof (stronger); combine with rate limits enforced at the LEZ escrow program.

## Relationship to other RFPs in this bundle

- **RFP-003 (Atomic Swaps with LEZ, open)** is the foundation this RFP extends. The per-pair SDKs (LEZ-BTC, LEZ-XMR, LEZ-ETH), the Risc0 LEZ-side escrow, and the custom-LEZ-token support (RFP-003 hard requirement 7) are dependencies. RFP-022 layers bond escrow, slash conditions, and chain-watching light-client modules on top.
- **RFP-021 (cross-chain privacy DEX)** is the federated-custody alternative. RFP-021 sacrifices non-custody for AMM liquidity and one-step UX; RFP-022 sacrifices liquidity gravity for non-custody. The two coexist in a complete cross-chain DEX strategy.
- **RFP-023 (reputation-based atomic swaps)** is the bonding alternative. RFP-022 consumes the maker-reputation primitive from RFP-023; in Tier 2 specifically, the taker-reputation primitive is the only restraint on Alice's residual pre-XMR-lock free option. If RFP-023 ships later, RFP-022 specifies a stub interface and degrades to "count of completed swaps" reputation in the interim.
- **RFP-024 (sXMR pure)** and **RFP-025 (sXMR with SLA)** are orthogonal. They target synthetic XMR exposure inside LEZ DeFi; this RFP targets real-asset atomic swaps. They could be deployed alongside.

See [appendix/cross-chain-trust-model-contrast.md](../appendix/cross-chain-trust-model-contrast.md) for the full mitigation-2 design analysis and the impossibility argument for the LEZ to XMR Tier 2 asymmetry.

## References

- [RFP-003: Atomic Swaps with LEZ](./RFP-003-atomic-swaps.md)
- [eth-lez-atomic-swaps reference implementation](https://github.com/logos-co/eth-lez-atomic-swaps)
- [Bitcoin to Monero atomic swaps (getmonero.org, 2021-08-20)](https://www.getmonero.org/2021/08/20/atomic-swaps.html)
- [Hoenisch and del Pino, Atomic Swaps between Bitcoin and Monero, IACR 2020/1126](https://eprint.iacr.org/2020/1126.pdf)
- [Adaptor signatures, Lloyd Fournier](https://github.com/LLFourn/one-time-vrf/blob/master/main.pdf)
- [Scriptless Scripts, Andrew Poelstra](https://github.com/apoelstra/scriptless-scripts)
- [BIP-340: Schnorr signatures for secp256k1](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki)
- [BIP-341: Taproot](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki)
- [Citrea Clementine LCP](https://citrea.xyz/learn/clementine)
- [ZeroSync](https://zerosync.org/)
- [Monero whitepaper: Zero to Monero 2.0](https://www.getmonero.org/library/Zero-to-Monero-2-0-0.pdf)
- [Monero FCMP overview](https://www.getmonero.org/resources/moneropedia/fcmp.html)
