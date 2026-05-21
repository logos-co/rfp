# sXMR Design Space: Synthetic XMR on LEZ

Background appendix for RFP-024 (sXMR pure non-custodial) and RFP-025 (sXMR with redemption SLA). Establishes the three design goals these RFPs partition between and the trade-offs that distinguish them.

## What sXMR is

A synthetic Monero token deployed as a program on the canonical Logos Execution Zone (LEZ). Oracle-priced for trading and composability inside LEZ; redeemed to real XMR via peer-to-peer atomic swap with no central custodian, no bridge, and no protocol-held reserves.

The wedge: the only synthetic in the published landscape that terminates in a privacy-preserving asset on a privacy chain. sBTC (Stacks) redeems to public BTC, leaving the destination on a transparent ledger. Every other commodity synthetic (sBTC variants, sETH, sLINK) redeems to traceable transparent assets. sXMR is the first design where the redemption path itself preserves privacy.

Honest framing: this is a synthetic with a soft, market-clearing peg, not a hard-redeemable synthetic. The oracle is the *quoted* price; the *achievable* price is whatever an XMR LP will swap for, when one is willing. Structurally closer to a DEX trading pair than to sBTC (Stacks).

The RFP set picks one of two product directions before specifying further. They are incompatible at the architectural level:

- **Non-custodial, mostly-works** (RFP-024). Pure atomic-swap design ships. Soft peg, market-dependent exit. The interesting product for privacy maximalists.
- **Always real XMR at oracle price** (RFP-025). Requires bonded LPs (slashable stable collateral on LEZ) or a protocol XMR reserve. Custody or solvency risk returns. The marketable product for SLA-needing audiences.

## Goal 1: non-custodial, mostly-works

The premise of RFP-024.

### Properties

- **Non-custodial.** No vault holds XMR. No signer set. No bridge.
- **Soft peg.** Oracle is a reference; achievable redemption price is whatever an LP quotes. Spread widens under stress without bound.
- **No redemption guarantee.** Counterparty may not exist when you want to exit.
- **Composable on LEZ.** sXMR is a vanilla LEZ token, callable by any other LEZ program (lending, DEX, governance).
- **Private exit.** Successful redemption deposits real XMR on Monero L1, severing the public trail.
- **Open LP set.** Anyone with XMR can quote; no permission required.
- **Regulatory minimalism.** Protocol does not handle XMR; arguably just a price feed plus a matching board.

### High-level flow

```
   sXMR LEZ program         oracle (XMR/USD)        Oracle program on LEZ
   (token + stable vault)  <-----------------       
                                                    
       mint  |  burn                                
                                                    
   Intent gossip                  match              Open XMR LPs
   (off-chain, via Logos                             (anonymous, free
   Delivery)                                         to enter/exit)
   sXMR <--> XMR quotes                              
                                                    
            atomic swap (adaptor-sig)                
            LEZ <----------------------> Monero L1   
                                                    
   sXMR holder gets XMR                  XMR LP gets sXMR,
   on Monero L1                          burns for stable
```

### Failure modes

1. **LP exodus.** All XMR holders stop quoting. sXMR trades at an indefinite discount to oracle.
2. **Adverse selection.** LPs only show up when oracle is below true XMR price (free money for them) and vanish when oracle is above (would-be loss). Redemption is asymmetric across regimes.
3. **Demand asymmetry.** Easy to mint sXMR (privacy-curious DeFi users want it); harder to source LPs (XMR maximalists may not want a public LP role at all).
4. **User experience cost.** Monero atomic swap windows are 30 to 60 minutes; both parties must be online unless an intent layer is built on top.

### When this fits

- Privacy-maximalist user base willing to accept variable redemption.
- Use cases that need sXMR for trading exposure rather than guaranteed redemption (DeFi composability, hedging, speculation).
- Builders willing to ship the cryptographically pure version and let the market clear.

## Cross-cutting design challenge

Two questions that affect both Goal 1 and Goal 2.

### The orderbook probably should not be on-chain

An on-chain orderbook (whether on the canonical LEZ or anywhere else) is expensive, leaks intent publicly (which undermines the privacy story), and provides no security benefit: the atomic swap is what makes settlement trustless, not the matching layer. A better split:

- **Off-chain matching via Logos Delivery.** LPs broadcast quotes; redeemers broadcast intents; parties pair up bilaterally. Quotes and intents never hit chain state.
- **On-chain settlement only.** A minimal LEZ program that verifies the atomic-swap primitive (lock, reveal, refund) on the LEZ side. It knows nothing about prices, identities, or who matched with whom.

This also aligns better with the privacy proposition: an on-chain orderbook would be a public registry of "everyone trying to acquire real XMR right now."

### Atomic-swap execution is hard to enforce on-chain

Atomic swaps are deliberately symmetric: either party can refuse the next message at any stage, and both sides refund at timeout. On-chain evidence cannot distinguish:

- An LP maliciously refusing to proceed.
- An LP whose node went down or lost connectivity.
- A redeemer never locking their side, making the LP's non-lock correct behaviour.
- A redeemer locking and then refusing to reveal, blaming the LP.

This is the whole point of an atomic swap: nobody can be forced to complete, and nobody loses funds if they walk away.

Critically, this means an atomic swap behaves like a free option for whichever party acts next. Stage by stage:

1. **Alice (redeemer) locks sXMR on LEZ.** Bob (LP) now holds a free option: if the XMR price has moved in his favour since the quote, he locks his XMR and the swap proceeds; if it has moved against him, he simply does not lock. Alice's sXMR refunds at timeout. Bob has paid only the cost of waiting and gained the optionality of letting an adverse swap expire.
2. **Bob (LP) locks XMR on Monero.** Alice now holds the same free option in reverse. If the price has moved in her favour, she completes the swap by revealing the secret; if it has moved against her, she does not reveal. Both sides refund. Alice has paid only fees.
3. **Secret reveal.** Once the secret is revealed by either side, the swap completes; this is the only stage where the protocol is no longer optional.

So at every stage before completion, the next party to act has a no-cost-beyond-fees option to abandon the swap based on price movement during the lock window. The "swap" is in effect a short-dated American option that either party can let expire.

This is well-known in the atomic-swap literature (the "free option problem"). It is not a bug in any particular implementation; it is the cost of atomicity itself. The cross-cutting [trust-model contrast appendix](./cross-chain-trust-model-contrast.md) covers the mitigations in depth.

Consequently, **a clean "LP defaulted, slash the bond" rule is not enforceable from on-chain state alone.** Refusing to proceed is *valid behaviour* under the protocol, not a default. Any slashing or "punishment" mechanism needs an off-chain attestation of who refused, which means one of:

- A trusted attestor or committee deciding default, which reintroduces centralised trust.
- A reputation system, which is only useful at scale and cannot enforce against first-time defection.
- A multi-attestor oracle quorum watching both chains, which adds its own trust assumption and liveness requirement.

Without one of those, the on-chain program can only do one thing: deny future LP slots, queue priority, or fee tiers. It cannot slash collateral with cryptographic certainty.

### Implication for Goal 2

- **Goal 2a (bonded LPs)** is structurally weaker than its description suggests. The bond cannot be slashed on a simple "LP refused" condition; any real slashing requires a reputation or attestor system layered on top. Read the design as best-effort with a reputation layer (consuming the primitive from RFP-023), not as cryptographically guaranteed redemption.
- **Goal 2b (protocol reserve)** is unaffected by the symmetry problem; trust simply lives in the signer set instead.

## Goal 2: always real XMR at oracle price

The premise of RFP-025. sXMR must be redeemable for real XMR at (or very near) the oracle price, on demand. The atomic swap is still the settlement primitive, but counterparty availability is no longer left to the open market. The protocol either commits LPs to honour redemption or holds an XMR reserve itself.

Two sub-designs, each restoring some trust that Goal 1 deliberately avoided. RFP-025 puts the choice between them to applicants.

### Sub-design 2a: bonded LP set

LPs join a registered set. Each LP posts stable collateral on LEZ equal to (or some multiple of) their XMR commitment. When a redemption request is routed to an LP, they must complete the atomic swap within a window. If they default, their bond is slashed and paid to the redeemer. LPs may leave the set, but only after a notice period that exceeds the redemption SLA.

Enforceability caveat (see the cross-cutting section above): "LP defaulted" is not a verdict an on-chain program can render from atomic-swap state alone. Any slashing rule needs an off-chain attestor or a reputation system to attribute fault. Without one, the bond can be used to gate participation (priority, fee tiers, future-slot access) but not slashed with cryptographic certainty on a single failed swap. The realistic implementation consumes the RFP-023 reputation primitive as the attribution layer.

```
                    sXMR LEZ program
                    + LP registry
                    + slashing logic
                    
       redemption request                LP bond
                                         
     sXMR holder                         Bonded LP
     burns sXMR,    <----- atomic swap   posts stable bond,
     receives XMR          (adaptor-sig, delivers XMR or
                          SLA)           forfeits bond
                                         
     if LP defaults: bond is paid out as compensation,
     reputation attests non-delivery
```

### Sub-design 2b: protocol XMR reserve

The protocol accumulates an XMR reserve from mint fees, a yield programme, or a one-time treasury seed. Reserve is held in a threshold-signer multisig or analogous custody arrangement on Monero. Redemption draws from the reserve directly, with the atomic swap acting as the settlement rail between the reserve custodian and the redeemer.

```
                    sXMR LEZ program
                    + reserve accounting
                    
       burn sXMR                trigger swap
                                         
                Reserve module
                (LEZ program)
                                         
            atomic swap
                                         
        Threshold-signer reserve on Monero
        (n-of-m, bonded signers)
```

At this point the design has reinvented sBTC (Stacks) with an oracle bolted on. The atomic swap is just the wire format; trust lives in the signer set. The same view-key-shared TSS custody constraint applies as in RFP-021 (Serai-like federated middle chain): honest-but-curious signers learn the protocol-side deposit history. This is the structural trade-off Goal 2b accepts in exchange for the redemption SLA.

### Properties comparison

| Property | 2a Bonded LPs | 2b Protocol reserve |
|----------|---------------|---------------------|
| Custodian | None (LPs custody their own XMR) | Yes (signer set) |
| Redemption guarantee | Up to total bonded capacity | Up to reserve size |
| Slashing surface | Yes (bond slashed on default, attested via reputation) | No (reserve is the slashing) |
| Oracle role | Pricing plus default attestation | Pricing only |
| LP economics | Yield from spreads plus protocol incentives, less bond opportunity cost | Not applicable (no third-party LPs) |
| Decentralisation | High (anyone can be a bonded LP if they post the bond) | Low (signer set is gated) |
| Censorship resistance | Medium (LP set is registered) | Low (signers are known) |
| Best-case redemption speed | Atomic swap (30 to 60 min) | Atomic swap (30 to 60 min) |
| Failure mode | Bond runs out under coordinated default | Signer collusion or custody breach |

### When this fits

- Audiences that need a redemption SLA (institutions, market makers, structured products).
- Use cases where sXMR is collateral inside other protocols and a stable peg matters more than purist non-custody.
- Regulatory contexts where "guaranteed redemption" is a feature, not a liability.

### Failure modes

- **Bonded LPs (2a):** coordinated default exceeds bonded capacity; the slashing oracle for default attestation is itself a trusted component; bond opportunity cost limits LP supply.
- **Protocol reserve (2b):** signer set is a target; if reserve is undercollateralised, peg breaks; effectively recreates sBTC (Stacks) custody risk. The TSS custody also leaks deposit history to signers (the Monero privacy break called out in the [trust-model contrast appendix](./cross-chain-trust-model-contrast.md)).

## Property matrix across goals

| Property | Goal 1 (pure) | Goal 2a (bonded LPs) | Goal 2b (reserve) | sBTC (reference) |
|----------|---------------|----------------------|-------------------|------------------|
| Custodian | None | None (LPs self-custody) | Yes | Yes |
| Reserve | None | None | Yes | Yes |
| Redemption guarantee | None | Bounded by bonds | Bounded by reserve | Bounded by reserve |
| Peg type | Soft | Hard within capacity | Hard within capacity | Hard 1:1 |
| Oracle dependency | Pricing | Pricing plus default attestation | Pricing | None for peg |
| LP role | Optional, open | Registered, bonded | None | None |
| Privacy on redemption | Yes (XMR L1) | Yes (XMR L1) | Yes (XMR L1) | No (BTC L1) |
| Worst-case failure | Indefinite discount, no exit | Bond depleted, partial exit | Signer compromise, full loss | Signer compromise, full loss |
| Closest existing analogue | DEX trading pair | Bonded relay (no direct analogue) | sBTC (Stacks) | sBTC (Stacks) |

## Decision tree

```
                Does the design need a redemption SLA?
                              
                no                          yes
                                              
                                              
         Goal 1 (pure)              Does the protocol accept custody risk?
         RFP-024                                
                                no                              yes
                                                                  
                                                                  
                              Goal 2a (bonded LPs)        Goal 2b (reserve)
                              RFP-025 option (a)         RFP-025 option (b)
                                                         (oracle-priced sBTC)
```

## Pre-spec validation

Before committing to either goal, applicants should validate:

1. **Atomic swap UX with Monero today.** Live protocols (eigenwallet/unstoppableswap fork of comit-network/xmr-btc-swap, Farcaster) take 30 to 60 minutes and require both parties online. Confirm async or intent-based variants are production-grade before designing UX around them.
2. **LP supply.** Will XMR holders actually LP? They self-select for privacy maximalism and may not want a public on-chain LP role. The LP side is the bottleneck; validate before designing the rest.
3. **Bond economics (Goal 2a only).** Required bond size as a multiple of XMR notional; opportunity cost of locked stable collateral on LEZ; expected APY needed to attract bonded LPs.
4. **Signer set sourcing (Goal 2b only).** Same problem space as sBTC (Stacks); revisit that project's trust assumptions before reinventing them. Also revisit RFP-021's federated-middle-chain trust analysis, which covers the same TSS custody design space.
5. **Regulatory.** A synthetic that terminates in a privacy coin will draw scrutiny under any of the three designs. Goal 1 has the cleanest defence (protocol is a price feed and a matching board); Goal 2b has the weakest (protocol custodies XMR).

## Bottom line

- **Goal 1 (RFP-024)** is the most interesting design and the one with the strongest privacy story. It will not satisfy users who expect a redemption SLA.
- **Goal 2a (RFP-025 option a)** is the most novel of the SLA-bearing designs: non-custodial, atomic-swap-settled, but with bonded LPs underwriting redemption capacity. No direct analogue exists in the published landscape. Depends on RFP-023 reputation for default attribution.
- **Goal 2b (RFP-025 option b)** is a real product but, structurally, an oracle-priced sBTC (Stacks) for Monero. The atomic swap is cosmetic; the trust assumption is the signer set.

Pick the goal before writing the spec. The three designs have different threat models, different LP economics, and different regulatory exposures.

## References

- [sBTC (Stacks) Bitcoin layer documentation](https://docs.stacks.co/concepts/sbtc)
- [Synthetix V3 documentation](https://docs.synthetix.io/v/v3/)
- [eigenwallet/core (XMR atomic swap fork of comit-network/xmr-btc-swap)](https://github.com/eigenwallet/core)
- [Farcaster project](https://github.com/farcaster-project)
- [Bitcoin to Monero atomic swaps (getmonero.org)](https://www.getmonero.org/2021/08/20/atomic-swaps.html)
