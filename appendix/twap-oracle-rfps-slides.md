---
title: "Oracles on LEZ — TWAP and Off-Chain Adaptors"
description: "Talk-track for RFP-019 + RFP-020"
tags: presentation, oracles, lez, rfp
slideOptions:
  theme: white
  transition: fade
  slideNumber: true
---

# Oracles on LEZ
### TWAP and Off-Chain Adaptors

RFP-019 + RFP-020

Note:
Open with: oracles are the missing infrastructure piece for DeFi on LEZ. Lending is specced (RFP-008) and the stablecoin is specced (RFP-013); neither works without a price feed. The DEX is specced (RFP-004) and does its own price discovery from pool state, so it doesn't consume an external feed for swaps, but the on-chain TWAP tier in RFP-019 reads its pool accumulators, so the DEX is upstream of part of the oracle work rather than a consumer of it. We're filling the price-feed gap with two RFPs.

---

# What is an oracle?

A program that brings off-chain data on-chain.

For DeFi, that almost always means a **price feed**.

Note:
Plain definition first. Smart contracts can't reach out to coingecko or a CEX themselves; they can only read state that's already on-chain. An oracle is the bridge: someone off-chain looks up the price, signs it, posts it on-chain, and the contract reads it.

Concrete example: Aave needs to know what 1 ETH is worth in USD to decide if a borrower's position is underwater. That number doesn't exist on-chain natively — it has to be fed in.

There are non-price oracles too (weather, sports scores, randomness) but ~99% of production oracle infrastructure today is price feeds.

---

# Why DeFi needs them

- **Lending** — collateral valuation, liquidation triggers
- **Stablecoins** — peg maintenance, redemption pricing
- **Derivatives** — settlement, mark-to-market
- **Liquidations** — when to act

Note:
Walk through each one. The constant: every protocol that has to know "what is this thing worth in dollars" needs an oracle. DEXes are the exception — AMMs do their own price discovery from pool state — but every protocol that *consumes* that price for solvency or settlement decisions needs an oracle.

For LEZ specifically: RFP-008 (lending) and RFP-013 (stablecoin) cannot ship without a price oracle. That's why this is the priority infra unlock.

Anchor stat: 36 documented flash-loan oracle attacks have caused $418M in cumulative DeFi losses. Bad oracles aren't just an inconvenience — they are the single biggest historical attack surface.

---

# Two kinds of oracle

|                | **On-chain**        | **Off-chain**                 |
| -------------- | ------------------- | ----------------------------- |
| Source         | DEX pool state      | external publishers           |
| Trust          | math + liquidity    | signer set                    |
| Day-one viable | needs deep pools    | needs only verifier program   |
| Coverage       | only on-chain pairs | any asset (USD, XMR, ZEC, FX) |

Note:
This is the core dichotomy. Not strict — there are hybrids — but useful for explaining tradeoffs.

On-chain: the canonical example is Uniswap V3's TWAP. The DEX itself stores a running price accumulator; any contract can read it and compute a time-weighted average. No external publisher, no signature, no bridge.

Off-chain: someone outside the chain (Chainlink, Pyth, RedStone) signs a price and gets it on-chain via either push or pull mechanics. We'll come back to push vs pull.

---

# On-chain TWAP — how it works

- Pool stores running accumulator: `price × elapsed_time`
- Reader takes accumulator at T1 and T2
- TWAP = (acc[T2] − acc[T1]) / (T2 − T1)
- Uniswap V3: geometric mean over log-tick accumulators
- Cardinality up to 65,535 observations (~9 days at Ethereum's 12s blocks)

Note:
Whiteboard moment if needed. Two values stored in the pool, both monotonically increasing. To get a TWAP, you read at two points in time and divide.

Geometric mean is what Uniswap V3 switched to (V2 was arithmetic). The reason: an attacker who pushes the price 10x up in one block and 10x back in the next leaves zero impact on the geometric mean, but pushes the arithmetic mean way up. Geometric mean is the right shape for multiplicative price processes.

Cardinality: how many past observations the pool stores. Default is 1; can be expanded up to 65,535 at a one-time storage cost. At 12-second blocks that's ~9 days of history.

---

# On-chain TWAP — the catch

- Security scales **linearly with pool depth**
- Two-block validator attack costs ≈ round-trip swap fees + price impact
- $1M pool offers far less protection than $100M pool
- **Doesn't work for off-chain assets** (no on-chain pool for USD, XMR, ZEC)

Note:
This is the load-bearing slide for "why we still need off-chain". TWAP's security comes from the cost of moving the pool price and holding it. On a $100M pool, that's expensive. On a $1M pool, it's cheap.

The PoS multi-block attack is the cleanest example: under PoS, validators know one epoch ahead whether they control consecutive blocks. A validator with two consecutive blocks can move the price in block N, accumulator records it, then move it back in block N+1. Cost ≈ round-trip swap fees + price impact + the foregone arbitrage. Cheaper than people think.

And the killer: TWAP only works for pairs that exist as pools on the chain. If you want USD/XMR pricing on LEZ, there's no LEZ pool that produces it (XMR isn't natively on LEZ). You need an off-chain feed.

---

# Off-chain oracles — push vs pull, and how verification works

**Push:** oracle nodes write prices on-chain on heartbeat / threshold (Chainlink classic, Chronicle). Needs dedicated node operators.
**Pull:** consumer submits signed price data at txn time (Pyth; RedStone in pull mode). No dedicated oracle infrastructure on chain.

RedStone runs in both modes natively (50+ push deployments, 120+ pull deployments per the appendix survey).

Verification path is the same shape in both: signers sign data packages, an on-chain verifier recovers M-of-N signatures, verified price is published.

|               | **Pyth**                                                                      | **RedStone**                          |
| ------------- | ----------------------------------------------------------------------------- | ------------------------------------- |
| Mode          | Pull (Wormhole)                                                               | Push + Pull                           |
| Quorum        | 13-of-19                                                                      | typically 3-of-N                      |
| Scheme        | secp256k1 ECDSA + double-keccak256 (single keccak256 on Solana), Wormhole VAA | secp256k1 ECDSA + keccak256, calldata |
| Bridge needed | Wormhole                                                                      | none                                  |

Note:
Push is the chicken-and-egg model for new chains: push oracles need node operators, operators need TVL, TVL needs DeFi, DeFi needs oracles. Pull flips it — the consumer pays per update, no permanent infrastructure layer required, works on day one.

LEZ-specific wrinkle covered next slide: classical pull mode (verify-inside-the-consumer-tx) doesn't transfer cleanly because of the zkVM's in-circuit cost profile. We end up with a push-mode aggregator that consumers (including private accounts) read, regardless of whether the upstream is delivered as push or pull on its native chain.

Both Pyth and RedStone sign with secp256k1 ECDSA over keccak256 — same primitive, different wrapping. Pyth wraps in Wormhole VAAs (13 ECDSA recoveries + Merkle proof per update + guardian-set tracking on-chain). RedStone is calldata-only: the signed package is just bytes, the verifier recovers signers and checks against a registered allowlist. No bridge.

The next slide covers what LEZ has wired into its runtime, what guest programs can actually call, and what that means for the in-program cost of verifying these schemes.

---

# LEZ verification primitives — what's there

- LEZ is a RISC-V zkVM (built on RISC0)
- One signature primitive wired into the runtime: **single-key BIP-340 Schnorr over SHA-256**
- That primitive validates **transaction witnesses only** — it is **not exposed to guest programs**
- No threshold / aggregate primitives, no ECDSA, no ed25519 callable from program code

→ Any signature a program needs to verify (BIP-340 from a DLC publisher, ECDSA-keccak from RedStone or Pyth, ed25519 from Switchboard) runs **in-circuit**. Prototype work on secp256k1 ECDSA inside RISC0 ([`fryorcraken/lez-signature-bench`](https://github.com/fryorcraken/lez-signature-bench)) shows proof generation in the order of minutes on consumer hardware; private-execution pull is infeasible, public-mode write-side cost amortises across reads.

Note:
This is the slide that reframes the rest of the deck.

LEZ is a RISC-V zkVM built on RISC0. The runtime has one signature primitive wired in: single-key BIP-340 Schnorr over SHA-256, used to validate the witness on each transaction at admission time. Crucially, this primitive is not exposed to guest programs — a program running inside the RISC-V zkVM cannot call it as a host function. There is also no callable ECDSA, no callable ed25519, no keccak256 host function, no threshold/aggregate primitive.

Every general-purpose price oracle in production today (RedStone, Pyth via Wormhole, Chainlink Data Streams, Chronicle's per-signer leg) signs with secp256k1 ECDSA over keccak256. Switchboard signs with ed25519. The DLC oracle ecosystem signs with BIP-340 Schnorr over SHA-256 — same scheme as LEZ's transaction primitive but, again, not callable from a guest program. (Of the DLC publishers, only Pythia is currently live; Sibyls is dead, Ernest is on hiatus, Suredbits is dormant.)

The cost question is therefore the same shape across signature schemes: any in-program verification has to run as RISC-V code inside the RISC0 zkVM. Some primitives are more expensive to prove than others (ECDSA recovery and keccak256 in particular), but none get the "free precompile" treatment.

The next slide covers the four shapes available for closing this gap.

---

# Four adaptor shapes

| Shape                        | What it is                                                                                                | Trust set                    | Engineering                       | In-circuit cost question                  |
| ---------------------------- | --------------------------------------------------------------------------------------------------------- | ---------------------------- | --------------------------------- | ----------------------------------------- |
| **A** Re-signer relayer      | Relayer's signature authenticates the LEZ tx itself; program does an equality check on the caller         | 1 operator                   | small                             | avoided (runtime-handled, not in-program) |
| **B** FROST federation       | t-of-n threshold Schnorr emitting one BIP-340 sig                                                         | t honest signers             | green-field R&D                   | yes (in-circuit Schnorr)                  |
| **C** DLC-oracle extension   | BIP-340-native publisher (Pythia, etc.)                                                                   | 1 op (or M-of-K independent) | small to medium                   | yes (in-circuit Schnorr × N bits)         |
| **D** secp256k1 ECDSA on LEZ | RedStone/Pyth ECDSA-keccak verified in program code (day 1); precompile is the cost-conditional follow-on | upstream signers             | implementation + cost measurement | yes (in-circuit ECDSA)                    |

→ **RFP-020 picks D.** RFP-019 (on-chain TWAP) is structurally separate from this choice. Shape A is rejected (trust collapse). Shape B is conditional on LEZ later exposing Schnorr verification to guest programs at low cost. Shape C is documented; structurally a fit for prediction markets, not current oracle work.

Note:
Quick walk-through of the four:

A — A LEZ-side process pulls RedStone/Pyth payloads, verifies them off-chain, then submits a regular LEZ transaction calling the price-aggregator program with the resulting price. The relayer's BIP-340 signature authenticates the transaction itself; the runtime validates it at admission time as part of the standard transaction-admission flow, and the program does an equality check on the caller against a registered relayer pubkey. No in-program signature verification, so no in-circuit cost. Trust collapses to one re-signer. Rejected.

The structural test that distinguishes A from B/C/D: **whose signature authenticates the LEZ transaction**. A's transaction is signed by the re-signer (runtime handles verification). B/C/D's transactions carry an upstream publisher's signature inside calldata, distinct from the transaction sender — the guest program has to verify it in-circuit.

B — A t-of-n federation does DKG and produces one BIP-340-verifiable Schnorr signature per price update. Libraries exist (ZF FROST, Blockstream bip-frost-dkg, jesseposner FROST-BIP340) but no oracle is using FROST in production today; FrostOracle (Chen et al., IEEE iThings 2023) is the lone academic proposal, and the closest production precedent is iBTC/DLC.Link's federation for *contract-outcome* attestation, not price feeds. Crucially, BIP-340 verification is not exposed to guest programs on LEZ either (the runtime primitive is for transaction witness only), so a guest program consuming a FROST-aggregated signature has to verify it in-circuit — same in-circuit cost question shape D faces with ECDSA. **Shape B is therefore not the right call unless LEZ later exposes Schnorr verification to guest programs at acceptable cost**; without that, you're paying the same in-circuit cost as D plus the green-field R&D risk.

C — Bitcoin DLC oracles (Pythia, Sibyls, Suredbits, Ernest Oracle) publish BIP-340 attestations. Two disqualifiers, either sufficient on its own. First, same runtime dependency as B: each verification runs in-circuit at unmeasured cost, and the numeric DLC encoding multiplies that by N (bit-precision of the price), so shape C is not the right call unless LEZ later exposes Schnorr verification to guest programs at acceptable cost. Second, even with cheap Schnorr verification the structural fit is prediction markets and discrete-outcome contracts, not streaming price feeds. Better positioned for a future prediction-market RFP than for the current oracle work. The DLC info is in the appendix for reference.

D — Implement RedStone's secp256k1 ECDSA + keccak256 verification as RISC-V program code inside the RISC0 zkVM. Push-mode aggregator: write side does the verify once per update, private accounts read the resulting public price account. The cost is the open variable; measuring it is the first deliverable of RFP-020. If measured cost is acceptable, the adaptor ships on the runtime as it stands. If not, a follow-on RFP proposes adding a secp256k1 ECDSA + keccak256 precompile (public-mode only). The precompile is therefore an optimisation path, not a precondition.

RFP-019 reads on-chain AMM pool state and accumulates observations — no external publisher signature is involved, so the LEZ-native primitive is sufficient and the off-chain signature question doesn't apply.

---

# Push mode on LEZ — same end-state design under D1 and D2

- Public-mode aggregator does the verify **once** per update
- Stores price + timestamp in a public price account
- Private accounts **read** the slot — no signature work in the private path

```
Public-mode aggregator:
  in-program ECDSA + keccak verify (RISC-V on RISC0)
  writes price to public account

Private account:
  reads public price account
  → no signature work in the private path
```

The reasoning differs slightly between D1 (in-program verify) and D2 (precompile follow-on); the design is the same.

Note:
The push-mode aggregator pattern is the right design under both implementation paths. The reasoning differs.

**Under D1 (RISC-V in-program ECDSA, day-1 path):** in-program verification is reachable from private execution (the same RISC-V code can run inside a user's private proof), but prototype data ([`fryorcraken/lez-signature-bench`](https://github.com/fryorcraken/lez-signature-bench)) puts proof generation in the order of minutes on consumer hardware. That rules out private-execution pull mode in practice, absent a RISC0-specific signature-verification accelerator. Push mode amortises the public-mode write-side cost across all downstream reads. Pull mode from public execution is technically possible but pays full proving cost per consumer transaction with no amortisation, strictly worse than reading the public price account.

**Under D2 (precompile follow-on, only if D1's measurement forces it):** the asymmetry becomes structural rather than economic. A precompile is unreachable from private execution because anything in a private transaction has to be expressible inside the RISC-V zkVM circuit, and a host function lives outside it. Private execution would have to either (a) verify in the privacy circuit (forfeits batching, defeats the precompile's purpose) or (b) place the signature in the transaction journal and break privacy. Neither option preserves both efficiency and privacy. So under D2, push mode is not a preference — it's the only design that works.

In both cases, the aggregator program is the same. That's what makes D2 a localised swap-in for D1 if the cost measurement forces the upgrade.

A LEZ-specific freshness pattern: a user who needs a price fresher than the heartbeat can submit a public transaction that pushes a fresh signed payload to the aggregator, then submit a private transaction immediately after that reads the just-updated price. Verification is paid in the public path (cheaper); the private path does no signature work. This recovers pull mode's "fresh at transaction time" property for private consumers without paying in-circuit cost.

The same mechanism enables a **consumer-pays push variant** with no dedicated relayer at all: users push when they need it, idle periods incur zero update cost. Operationally pull, structurally push. Whether to run a heartbeat relayer in addition is a deployment-time choice. Both can coexist; the program logic does not distinguish between them.

RedStone supports both push and pull natively (50+ push deployments, 120+ pull deployments per the appendix survey). RFP-020's push-mode aggregator on LEZ can consume RedStone's own pusher, run its own, or rely on consumer-pays push, in any combination.

A note on demand: the framing here treats private-execution pull as a capability that current paths foreclose, but whether any consumer protocol on LEZ actually needs it is a separate, unsettled question. Some consuming protocols already have design reasons to keep specific actions in public transactions; the LSC stablecoin in RFP-013 is one example. A follow-on RFP that proposes a private-pull path should start by establishing that some downstream consumer genuinely needs it, not just by selecting a friendlier primitive.

---

# TWAP vs off-chain — head to head

|                           | **TWAP**             | **Off-chain pull**    |
| ------------------------- | -------------------- | --------------------- |
| Trust assumption          | DEX liquidity        | signer set honesty    |
| Day-one viable on LEZ     | no (DEX comes later) | yes                   |
| Privacy assets (XMR, ZEC) | no                   | yes                   |
| Cost per query            | very cheap           | calldata + sig verify |
| Manipulation defence      | depth-dependent      | M-of-N signers        |

→ **Best practice: use both.** Production protocols cross-check.

Note:
Walk through the row by row. They're complementary, not competing. The production norm in EVM DeFi is multi-tier:
- Aave V3: Chainlink primary + configurable fallback
- Compound V2: Coinbase reporter anchored against Uni V2 TWAP (20% divergence triggers a circuit breaker)
- MakerDAO: median feed + 1-hour OSM delay as a manipulation circuit breaker
- Liquity V2: Chainlink + simple secondary fallback

That's what the circuit-breaker interface in RFP-019 is for: when both sources are present, divergence above threshold flags the price as disputed.

---

# What RFPs are for at Logos

- A spec + funding ask, published openly
- External teams apply via GitHub issue
- Bootstrap the LEZ DeFi stack from outside
- We don't pick implementers; market does

Note:
TODO: confirm with user — RFP funding source (Logos treasury / IFT / specific budget?), application review cadence, whether there's a Lambda Prize tie-in for any of these RFPs, whether scope is locked or up for revision during review.

The frame for the audience: RFPs are how we get parts of the LEZ stack built without staffing every layer in-house. We write the spec carefully, publish the budget, and external builders apply. Past examples: lending (RFP-008, in flight), DEX (RFP-004, awarded), launchpads (RFP-015 / RFP-016, in flight).

Today: oracles. Two RFPs covering the two oracle tiers we just walked through.

---

# The two oracle RFPs

|               | **RFP-019**            | **RFP-020**                                                      |
| ------------- | ---------------------- | ---------------------------------------------------------------- |
| Subject       | On-chain TWAP          | RedStone off-chain adaptor (RISC-V in-program verify)            |
| Tier          | L                      | M                                                                |
| Duration      | 8–12 wks               | 6–10 wks (adaptor) + LEZ runtime work                            |
| Hard blockers | RFP-004 (DEX), LP-0015 | None at runtime level (precompile is cost-conditional follow-on) |

Both feed into RFP-008 (lending), RFP-013 (stablecoin), RFP-004 (DEX consumers).

Note:
Why two RFPs and not one: different dependency profiles, different timelines.

- RFP-019 reads on-chain AMM pool state, so it's gated on the DEX (RFP-004) landing. No external publisher signature is involved, so the LEZ-native single-sig primitive is sufficient.

- RFP-020 picks shape D: implement RedStone's secp256k1 ECDSA + keccak256 verification as RISC-V program code inside RISC0, push-mode aggregator pattern. Cost measurement is a primary deliverable. If the measured cost is acceptable, the adaptor ships on the runtime as it stands. If not, a follow-on RFP proposes adding a secp256k1 ECDSA + keccak256 precompile to LEZ (public-execution mode only). The precompile is therefore an optimisation path, not a precondition for RFP-020.

The push-mode design is structural, not stylistic — see the next slide.

Awarding them as one combined RFP would have forced a single team to wait on the DEX before delivering anything. Splitting lets the off-chain tier ship in parallel with DEX work.

---

# RFP-019 — On-Chain TWAP Oracle

**What it ships**
- Reads DEX pool accumulators; computes geometric-mean TWAP
- Defines **canonical oracle price account standard** (SVM IDL)
- **Circuit breaker** against external feeds via that standard

**Why it matters**
- Single-source feeds = single point of failure
- Layered defence = on-chain + off-chain cross-check (production norm)
- Unlocks **LSC composite oracle path** (RFP-013 Path B)

Tier L · ~8–12 weeks · gated on RFP-004 (DEX) + LP-0015

Note:
RFP-019 is structurally separate from the off-chain primitive question covered in earlier slides. It reads LEZ-native AMM pool state, accumulates price observations, and exposes them through a program account. No external publisher signature is involved; the LEZ-native single-sig primitive (used for transaction authentication, not data attestation) is sufficient.

The deliverable splits into two halves with different timelines:
1. The **standard + circuit-breaker interface** — pure design work, can be specced, IDLed and prototyped in parallel with RFP-004. SVM-style account-data structures are append-friendly so the standard can evolve later without breaking consumers.
2. The **TWAP program itself** — reads pool accumulators, so it's gated on the DEX landing. This is the longer pole.

The "why" pitch in three beats:
1. Every consuming protocol on LEZ inherits the oracle's risk profile. Ship only off-chain feeds, you're trusting one publisher set.
2. Cross-checking on-chain vs off-chain is the production norm. The circuit breaker (5% divergence threshold) is how every major lending protocol limits oracle damage.
3. On-chain TWAP unlocks designs that don't trust an external publisher for the pair — specifically the LGS/USD + LGS/LSC composite path that RFP-013 may pick.

Soft blocker: LP-0012 (event emission) for dashboard / monitoring; not critical.

---

# RFP-020 — RedStone Off-Chain Adaptor (shape D)

- **Day 1**: implement secp256k1 ECDSA + keccak256 verification as RISC-V program code inside RISC0
- **Push-mode aggregator** → public price account → private accounts read
- **Cost measurement is a primary deliverable** (compute units, RISC0 proof time + size)
- Precompile is a **cost-conditional follow-on**, not a hard blocker
- **Day-one delivery: XMR/USD and ZEC/USD**

Tier M · ~6–10 weeks · no Wormhole dependency · no runtime change required

Note:
LEZ is a RISC-V zkVM built on RISC0. The runtime's existing BIP-340 signature primitive validates transaction witnesses only; it is not exposed to guest programs, and there is no callable ECDSA / keccak host function either. So any signature a program needs to verify runs in-circuit. Prototype work on secp256k1 ECDSA inside RISC0 ([`fryorcraken/lez-signature-bench`](https://github.com/fryorcraken/lez-signature-bench)) shows proof generation in the order of minutes on consumer hardware: private-execution pull mode is therefore not on the table, and the adaptor design has to be push-mode aggregator with the verification cost amortised across reads.

RedStone's data nodes sign price packages with secp256k1 ECDSA over keccak256. RFP-020 implements that verification path in RISC-V program code using existing Rust crates (k256 / sha3 / equivalents) and proves it via RISC0 alongside the rest of the program. The structural choice is push-mode aggregator: the verifier runs once per update on the write side, prices land in a public price account, and private accounts compose by reading the slot. Pull mode for private accounts is not on the menu — verifying a signature inside the privacy circuit forfeits batching benefits, and putting it in the transaction journal breaks privacy.

Prototype data ([`fryorcraken/lez-signature-bench`](https://github.com/fryorcraken/lez-signature-bench)) already establishes that naive in-circuit ECDSA in RISC0 is slow enough to rule out private-execution pull mode (proof generation in the order of minutes on consumer hardware). Public-mode write-side cost is the open variable, since amortisation across all reads can make a per-update cost workable that would be unworkable per-private-transaction. Measurement of the public-mode cost is the first deliverable. Two outcomes:

1. **Cost is acceptable.** The adaptor ships on the runtime as it stands.
2. **Cost is unacceptable.** The measurement becomes the input to a follow-on RFP that proposes adding a secp256k1 ECDSA + keccak256 precompile to LEZ for public-execution mode. The applicant should design the verification path so that swapping in a precompile later is a localised change.

Separately, if a signature scheme exists that yields acceptable in-circuit cost for **private** execution (RISC0-friendly hash + curve choices, or a different signature primitive entirely), a future RFP could propose either building or modifying an existing oracle network to publish in that scheme; that would unlock pull-mode reads from inside private transactions, which the current ECDSA-on-RISC0 path forecloses. Whether such a follow-on is worth pursuing depends on consumer demand for private-execution pull, which is not yet confirmed: parts of RFP-013's LSC stablecoin design already constrain specific actions to public transactions, so the capability is only worth chasing if a downstream consumer actually needs it.

XMR and ZEC are mandatory deliverables. RedStone was chosen because both feeds are in its public token registry, it has no bridge dependency, and it natively supports both push and pull modes (RedStone runs its own pusher for existing push deployments, so the LEZ side can either consume that or operate its own relayer).

---

# RFP-020 — Why RedStone as upstream source?

Coverage matrix for XMR/USD + ZEC/USD upstream of LEZ:

| Provider     | Both feeds? | Self-serve?           | Bridge needed? |
| ------------ | ----------- | --------------------- | -------------- |
| Chainlink    | partial     | **no** (permissioned) | n/a            |
| Pyth         | yes         | yes                   | **Wormhole**   |
| DIA Lumina   | yes         | yes (bespoke)         | n/a            |
| **RedStone** | **yes**     | **yes**               | **none**       |

→ RedStone is the simplest upstream source. The LEZ-side verification path is the same regardless of which one we pick.

Note:
This is a process-of-elimination on the *upstream* side, holding the LEZ-side decision (shape D — RISC-V in-program ECDSA + keccak verification, push-mode aggregator) constant.

- Chainlink: needs business engagement; no self-serve onboarding for a new chain. Out for day one.
- Pyth: technically the strongest (120+ publishers, confidence intervals) but requires Wormhole on LEZ in addition to the ECDSA+keccak primitive. Two dependencies, not one. Future RFP.
- DIA Lumina: permissionless, but each new chain needs a bespoke deployment.
- RedStone: connector format is portable, no bridge, no team engagement, both feeds in the public registry.

The adaptor framework is upstream-agnostic. Swapping in another upstream is a matter of replacing the data-package decoder and the signer-set registration; the verification path stays the same.

Reviewer note: seugu (anon-comms) reviewed PR #37 and pushed back on the oracle technical claims. We verified the signature schemes against primary sources — the appendix has all the citations.

---

# How the two RFPs fit together

```
                   RFP-019 (TWAP)
                   defines + populates
                          │
      Canonical oracle price account standard (SVM IDL)
                          │
                   ┌──────┴──────┐
                   │             │
            RFP-020 RedStone   future Pyth RFP
            populates          populates

Consumers (lending, stablecoin, DEX) read **one** struct.
```

Note:
The architectural payoff. Consuming protocols don't care which oracle source provides the price — they read the canonical account, get price + timestamp + source ID + confidence + dispute flag, and act.

If we add Pyth later, the wrapped token RFP, or anything else, the consumers don't need to update. The standard is append-friendly so it can grow without breaking existing consumers.

The circuit-breaker logic in RFP-019 also lives at this layer: when both TWAP and at least one external source are registered for a pair, divergence triggers the dispute flag.

---

# Timeline + next steps

- RFP-019: open immediately; build deferred until RFP-004 lands
- RFP-020: builds on LEZ as it stands today (no runtime change required up front)
- Cost measurement is a primary deliverable; if measured cost is unacceptable, a follow-on RFP proposes the precompile
- Canonical price account standard (RFP-019) and adaptor (RFP-020) can be designed in parallel

TODO: calendar dates, application window, review cadence, funding numbers.

Note:
TODO: get from user — actual timelines, funding numbers, go-live dates. $XXXXX placeholders still in both RFPs. Also confirm whether these RFPs need approval from a specific committee.

The split-and-parallel story:

- RFP-019's standard + circuit-breaker interface is pure design work and can ship ahead of the DEX. The TWAP program itself waits on RFP-004.
- RFP-020's verification path is RISC-V in-program code from the start. No runtime work is on the critical path. If the cost measurement deliverable shows the in-program path is too expensive at production cadence, a follow-on RFP for a public-mode precompile becomes the optimisation path, with this adaptor as the immediate consumer.

Decision pending from user: keep RFP-020 as one combined RFP or split.

---

# Q & A

Note:
Anticipated questions and the short answer:

**Q: Why does the adaptor implement ECDSA verification in program code rather than relying on a precompile?** Because LEZ doesn't currently expose any signature primitive to guest programs (the runtime's BIP-340 primitive validates transaction witnesses, not program-callable). LEZ is RISC0-based, so any signature scheme can be implemented in program code; the open question is whether the resulting cost is acceptable. RFP-020's first deliverable is to measure that cost. If it's acceptable for the push-mode aggregator's update cadence, the adaptor ships. If it's not, a follow-on RFP proposes adding a secp256k1 ECDSA + keccak256 precompile for public-execution mode.

**Q: What about private execution? Can a private transaction verify a RedStone payload inline (pull mode)?** No. Prototype data ([`fryorcraken/lez-signature-bench`](https://github.com/fryorcraken/lez-signature-bench)) puts proof generation for in-circuit secp256k1 ECDSA in the order of minutes on consumer hardware, which rules pull mode out in practice. Putting the signature in the transaction journal would break privacy. Even if a precompile is added later, it would live outside the ZK proof boundary and remain callable only from public execution. Private accounts compose by reading the public price account that the push-mode aggregator writes to.

**Q: Why not just sign in BIP-340 Schnorr+SHA-256 at the oracle, since that matches LEZ's transaction-witness primitive?** Two reasons. First, the runtime's BIP-340 primitive is not exposed to guest programs either, so a program verifying a BIP-340 signature also pays in-circuit cost — same shape as the ECDSA case. Second, no production price oracle signs in BIP-340 today; the candidates that did or do (Pythia is the only live publisher; Sibyls operator is dead, Ernest on hiatus, Suredbits dormant) are single-operator BTC/USD publishers built around discrete-event attestation, and none publish XMR/USD or ZEC/USD. See appendix shape A through D for the full analysis.

**Q: What about FROST? Threshold Schnorr that emits one BIP-340 signature?** Same in-circuit cost question as ECDSA, plus no oracle is using FROST in production today (closest precedent is iBTC/DLC.Link's federation for *contract-outcome* attestation, not price feeds). Pursuing this is the right call only if LEZ later exposes Schnorr verification to guest programs at low cost. Without that, it carries the same in-circuit cost as the ECDSA path plus green-field R&D risk.

**Q: What about DLC oracles (shape C)?** Documented in the appendix for reference. Structurally a fit for prediction markets and discrete-outcome contracts, not streaming price feeds. Same in-circuit cost question multiplied by N bits of precision (numeric DLC encoding signs the outcome bit-by-bit). Better positioned for a future prediction-market RFP.

**Q: Why isn't the secp256k1 precompile on the LEZ roadmap?** The LEZ runtime team has flagged genuine open questions beyond the oracle use case: nullifier tracking for replay protection, privacy-circuit branching to support Ethereum-signed private accounts, and identifier-flow / wallet implications around the recent multiple-private-accounts-per-keyset work. None blocks the narrow oracle use of the precompile, but they explain why it needs its own argument. Making the precompile a cost-conditional follow-on rather than a precondition lets RFP-020 ship without that argument having to land first.

**Q: How does LSC (RFP-013) consume these feeds?** Two production-security paths, both viable on top of RFP-019 + RFP-020. Path A: LSC/USD direct via off-chain oracle — simple, but exposed to thin LSC CEX liquidity early. Path B: LGS/USD external + on-chain LGS/LSC TWAP — deeper LGS liquidity, but the TWAP becomes the manipulation surface. We deliberately leave this open; RFP-013's implementer ships production economics and picks A or B as a business decision. The canonical price account standard keeps the choice swappable later.

**Q: Why not Pyth first?** Same in-circuit ECDSA + keccak verification path as RedStone, plus a Wormhole dependency on the LEZ side (guardian-set tracking, 13-of-19 VAA verification, Merkle proof). RedStone has only the verification; no bridge. Pyth becomes a future RFP once Wormhole-on-LEZ is decided.

**Q: Why not Chainlink?** Permissioned onboarding. We can't self-deploy; we'd need a business engagement.

**Q: What about TWAP manipulation on shallow LEZ pools at launch?** That is exactly why we need the off-chain tier as a cross-check. The circuit breaker in RFP-019 catches divergence between TWAP and the external feed.

**Q: Why M-of-N for RedStone? What's the failure model?** Signer compromise. If 3-of-N signers collude, they can sign arbitrary prices. The mitigation is the circuit breaker (cross-check against on-chain TWAP) plus signer-set rotation by the program owner.

**Q: Does this preclude future hybrid designs (TEE-attested, ZK-verified)?** No. The canonical price account standard is append-friendly. Future adaptors populate the same struct. Switchboard (TEE) and DIA Lumina (ZK) could be added without re-doing the consumer side, once their respective verification primitives are available on LEZ.

TODO: any other questions the audience is likely to ask that I haven't anticipated.

---

# Appendix: citations and detail

- `appendix/oracle-ecosystem.md` — full ecosystem survey
- RFP-019 — `RFPs/RFP-019-twap-oracle.md`
- RFP-020 — `RFPs/RFP-020-redstone-oracle-adaptor.md`
- PR #37 — review thread

Note:
For deep dives, point at the appendix. The full coverage matrix, signature scheme analysis (Wormhole VAA + RedStone), TWAP manipulation cost analysis, and citations all live there. The two RFPs themselves are tight; the appendix is the long-form reference.
