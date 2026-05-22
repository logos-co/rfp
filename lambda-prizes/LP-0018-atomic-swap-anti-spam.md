<!-- Draft staged in logos-co/rfp; will move to logos-co/lambda-prize once accepted. -->

<!-- Don't forget to add/update this prize in the table in README.md when moved. -->

# LP-0018: Anti-Spam Mechanism for Atomic Swaps [Draft]

**`Status`**:

- Draft: Not yet ready
- Open: Ready for application
- Completed: Submission accepted, prize completed

**`Logos Circle: N/A`**

## Overview

Atomic swaps are deliberately symmetric: either party can refuse the next
message at any stage and both sides refund at timeout. This is the design, not a
bug. But it lets a malicious taker spam makers: lock funds against a maker's
quote, wait until the price moves, complete or refund accordingly. The refund
branch costs the taker only a few external-chain transaction fees, while the
maker's inventory is wedged for the lock window and the maker absorbs the loss
when the taker walks. Han et al. (IACR 2019/896) prove this is formally
equivalent to a premium-free American Call Option and quantify the implicit
premium at approximately 2% of asset value for crypto pairs.

This prize is for an **innovative mechanism that prices out the taker's free
option** in the LEZ atomic-swap protocol (RFP-003), without specifying the
mechanism. The bar is a working implementation that demonstrably deters spam and
free-option exploitation while preserving the non-custodial
cryptographic-trust-only properties of the underlying atomic swap. Reference
prior art exists
([eigenwallet PR #675](https://github.com/eigenwallet/core/pull/675)); the prize
does not prescribe that approach. Solvers are free to design bonds, fee-burns,
reputation, deposits, slashing schemes, or any combination, as long as the
chosen mechanism survives the evaluation against the success criteria below.

## Motivation

The atomic-swap branch of the cross-chain DEX design tree has known structural
weakness on the free-option problem. The Logos cross-chain DEX bundle (RFPs 021,
024, 025) keeps the vanilla RFP-003 atomic swap as the privacy-non-custodial
primitive, but vanilla atomic swaps remain economically unattractive for makers
at any scale because of the free-option exposure. Without a credible anti-spam
mechanism, the maker side of the LEZ atomic-swap market collapses to a hobbyist
scale (see [eigenwallet](https://github.com/eigenwallet/core/): community-scale,
single-digit active makers, BTC→XMR direction only).

A competitive prize is the right mechanism because the design space is large and
the right answer is not obvious:

- **On-LEZ bonds** price the option premium via slashable collateral; they
  require LEZ-denominated capital from at least one side and they only work at
  boundaries that LEZ can observe (Tier 1 pairs like LEZ↔BTC, LEZ↔ETH).
- **External-chain fee-burns on the refund branch**
  ([eigenwallet PR #675](https://github.com/eigenwallet/core/pull/675)) price
  the option premium by destroying a fraction of locked principal on the
  script-bearing chain. They reach boundaries that LEZ cannot observe but they
  cost real principal on every refund (no refund on honest completion) and they
  require careful incentive design so the maker cannot weaponise the deposit.
- **Reputation-based deterrence** (the subject of the companion prize
  [LP-0019](./LP-0019-atomic-swap-maker-reputation.md)) provides soft pressure
  on repeat makers; first-time and anonymous takers escape this entirely.
- **Hybrid designs** combining the above in different proportions.

Each carries trade-offs in capital efficiency, capital denomination, anti-DDoS
coverage, and how it interacts with the LEZ↔XMR asymmetric case (where the
XMR-side lock is not LEZ-observable). The Logos team does not want to pre-judge
the answer; this prize is open to any approach.

## Success Criteria

### Functionality

- [ ] Demonstrably deters a taker who would otherwise lock an external-chain
  asset against a maker's quote and walk via the refund branch at near-zero
  cost. The submission must include a clearly-documented adversarial scenario
  and show how the mechanism makes the abort branch EV-negative for the taker.
- [ ] Works in at least one trade direction for the LEZ↔BTC pair (the corridor
  with the most existing prior art) and clearly states which other pairs
  (LEZ↔ETH, LEZ↔XMR in both directions) the mechanism covers, with justification
  for any exclusions.
- [ ] Compatible with the RFP-003 LEZ atomic-swap SDK as the underlying
  primitive; does not require changes to the cryptographic core (joint-key
  setup, adaptor signature, lock, reveal).
- [ ] Preserves non-custody: no third party (signer set, validator, oracle,
  attestor) holds user funds at any stage. The mechanism cannot reintroduce the
  federated-custody trust assumption.
- [ ] Survives the maker-locks-first case (in BTC→XMR direction, the Bitcoin
  side locks first by deployed convention). The mechanism must price the option
  held by whichever side locks first.
- [ ] Burnt or escrowed principal in adverse paths is **not** payable to the
  counterparty in a way that creates new griefing incentives (e.g. if the maker
  can profit by provoking a refund, that defeats the purpose). The submission
  must include an incentive-compatibility argument.
- [ ] Handles connectivity-loss / honest-refund cases gracefully: an honest
  party who refunds due to connectivity issues should not lose
  disproportionately.
- [ ] Discoverable parameters: the cost the taker faces under the mechanism
  (deposit fraction, bond size, etc.) is known to the taker before they initiate
  the swap; quote-level discovery is acceptable.

### Usability

- [ ] Provide a module/SDK that can be used to build Logos modules for
  interacting with the program.
- [ ] Provide a Logos Basecamp app GUI with local build instructions,
  downloadable assets, and loadable in Logos app (Basecamp).
- [ ] Provide an IDL for the LEZ program, using the
  [SPEL framework](https://github.com/logos-co/spel).

### Reliability

- [ ] The mechanism does not introduce new failure modes that lock user funds
  permanently in protocol-construction-error states. Every escrowed amount must
  have a finite-time recovery path under all adversarial choices the
  counterparty can make.
- [ ] Race conditions between counterparty actions (e.g. simultaneous refund and
  claim attempts) are documented and resolved deterministically.

### Performance

- [ ] Document the compute unit (CU) cost of each on-chain operation introduced
  by the mechanism, on both LEZ devnet/testnet and the external chain (Bitcoin,
  Ethereum, etc.) if the mechanism touches those.
- [ ] Quantify the additional transaction count vs vanilla RFP-003 atomic swap
  (e.g. how many additional Bitcoin transactions does a refund involve).
- [ ] State the expected external-chain transaction-fee cost ranges under normal
  and high-fee regimes (Bitcoin mempool congestion in particular).

### Supportability

- [ ] The program is deployed and tested on LEZ devnet/testnet.
- [ ] End-to-end integration tests run against a LEZ sequencer (standalone mode)
  and are included in CI.
- [ ] CI must be green on the default branch.
- [ ] A README documents end-to-end usage: deployment steps, program addresses,
  and step-by-step instructions for interacting with the program via CLI and
  Basecamp app.
- [ ] A reproducible end-to-end demo script is provided and works against a real
  local sequencer with `RISC0_DEV_MODE=0`.
- [ ] A recorded video demo of the end-to-end flow is included in the
  submission; the recording must show terminal output (including proof
  generation) to confirm `RISC0_DEV_MODE=0` was active.
- [ ] The demo includes at least one "adversarial taker spams the maker"
  scenario where the mechanism deters the attack, plus the honest-completion
  scenario where the mechanism imposes no cost on honest users.

## Scope

### In Scope

- The anti-spam mechanism itself: design, on-chain components (LEZ program,
  external-chain script changes if any), client-side SDK, and integration with
  RFP-003's per-pair atomic-swap modules.
- Incentive analysis: a written argument for why the mechanism is
  incentive-compatible and what attacker strategies it deters or admits.
- One concrete pair (LEZ↔BTC is the recommended starting point; LEZ↔ETH or
  LEZ↔XMR is acceptable if justified).
- A reference integration: working demo of a swap that uses the mechanism,
  including at least one adversarial path that exercises the deterrent.

### Out of Scope

- Modifying the underlying RFP-003 atomic-swap cryptography (joint-key setup,
  adaptor signature, lock/reveal). Solvers may add escrow logic around the swap
  but must not alter the swap primitive itself.
- Reintroducing federated trust (TSS custody, signer sets, oracle attestors).
  RFP-021 covers the federated-custody design space; this prize is for
  non-custodial mechanisms.
- A polished consumer UI beyond what's needed for the demo.
- Ongoing maintenance, security audit, or mainnet deployment beyond the testnet
  integration.
- Solving the LEZ↔XMR direction-symmetry problem in full (the residual off-LEZ
  option that vanilla atomic swaps inherently leave open; see also the companion
  prize [LP-0019](./LP-0019-atomic-swap-maker-reputation.md) on the off-chain
  reputation side of the same problem). A submission that addresses the LEZ↔BTC
  case cleanly is sufficient; addressing LEZ↔XMR is a bonus.

## Prize Structure

- **Total Prize:** $X
- **Effort:** L

> Leave prize pool blank — this will be determined by the Logos team. Single
> winner by default.

## Eligibility

Open to any individual or team. Submissions must be original work. Teams must
hold the rights to all submitted code and agree to license it under MIT or
Apache-2.0.

## Submission Requirements

A submission must include:

- A public repository containing the LEZ program(s), client SDK, integration
  tests, demo script, and any external-chain script changes.
- A written design document (in the repo) covering the mechanism, the
  adversarial model, the incentive-compatibility argument, and any honest-refund
  / connectivity-loss handling.
- A narrated video walkthrough demo showing (a) honest completion, (b) at least
  one adversarial-taker scenario where the mechanism deters the attack, and (c)
  any external-chain transactions involved. The demo must show terminal output
  including proof generation with `RISC0_DEV_MODE=0`.
- A FURPS self-assessment (see
  [solution template](https://github.com/logos-co/lambda-prize/blob/main/solutions/LP-0000.md)).
- A short comparison section against the reference prior art (at minimum:
  eigenwallet PR #675), stating what the submission borrows, where it diverges,
  and why.

## Evaluation Process

By default, submissions are evaluated first-come-first-served against the
success criteria. The first submission that meets all criteria wins.

Evaluators will independently clone the repository and run the demo script from
a clean environment; the script must succeed without modification. Evaluators
will also exercise at least one adversarial-taker scenario themselves to verify
the deterrent.

Because the design space is large and multiple valid approaches exist,
evaluators may rank tied submissions on:

1. Coverage across pairs (a mechanism that handles LEZ↔BTC, LEZ↔ETH, and at
   least one direction of LEZ↔XMR ranks above one that handles only LEZ↔BTC).
2. Capital efficiency (mechanisms that impose less cost on honest users rank
   above mechanisms that always impose cost).
3. Incentive-compatibility argument quality.
4. Integration cleanliness with RFP-003's existing per-pair modules.

The following policies apply to all prizes (see
[evaluation policies](https://github.com/logos-co/lambda-prize/blob/main/README.md#evaluation-policies)):

- **Submissions:** each builder (or team) is allowed a maximum of **3
  submissions** per prize, with at most **one submission/review per week**.
- **Feedback:** initial evaluation feedback is limited to a pass/fail indication
  against the success criteria.

## Resources

- [RFP-003: Atomic Swaps with LEZ](https://github.com/logos-co/rfp/blob/master/RFPs/RFP-003-atomic-swaps.md)
  — the vanilla atomic-swap protocol this mechanism builds on.
- [eigenwallet PR #675: fee-burn on refunds](https://github.com/eigenwallet/core/pull/675)
  — reference prior art; an open proposal that introduces a non-refundable burn
  on the refund branch via maker-set deposit fraction, withhold path, and mercy
  release. Solvers are free to adopt, adapt, or reject this approach.
- [eigenwallet/core release 4.0.0 anti-spam deposit](https://github.com/eigenwallet/core/releases/tag/4.0.0)
  — shipped narrower mechanism (cancel-timelock reduction plus 30-minute
  withhold/mercy) that PR #675 generalises.
- [appendix/atomic-swaps-primer.md](https://github.com/logos-co/rfp/blob/master/appendix/atomic-swaps-primer.md)
  — atomic-swap mechanics, free-option framing, `σ × √T × notional` notation for
  sizing.
- [appendix/cross-chain-trust-model-contrast.md](https://github.com/logos-co/rfp/blob/master/appendix/cross-chain-trust-model-contrast.md)
  — the federated-signers-vs-atomic-swaps trust contrast that motivates keeping
  atomic swaps non-custodial.
- [Han, Lin, Yu, On the optionality and fairness of Atomic Swaps, IACR 2019/896](https://eprint.iacr.org/2019/896)
  — the canonical free-option-problem paper; proves formal equivalence to a
  premium-free American Call Option and estimates the implicit premium at ~2% of
  asset value for crypto pairs.
- [Gugger, Bitcoin-Monero Cross-chain Atomic Swap, IACR 2020/1126](https://eprint.iacr.org/2020/1126.pdf)
  — protocol fundamentals; relevant for direction-dependence analysis.
- [Hoenisch and del Pino, Atomic Swaps between Bitcoin and Monero, arXiv:2101.12332](https://arxiv.org/abs/2101.12332)
  — §4 covers the draining-attack analysis that determines which side locks
  first, useful for solvers reasoning about XMR-first variants.
- [LP-0019: Off-Chain Verifiable Reputation for Atomic-Swap Makers](./LP-0019-atomic-swap-maker-reputation.md)
  — companion prize that addresses the reputation layer. A submission may
  consume LP-0019's reputation primitive as part of its design.

## Potential for Subsequent λPrizes

If this prize is awarded for a LEZ↔BTC-only mechanism, a follow-up prize may be
opened for LEZ↔XMR coverage once non-disclosing Monero proof primitives (FCMP++
or equivalent) reach production-ready status, since LEZ↔XMR has structural
challenges (the off-LEZ lock is not observable from LEZ) that this prize does
not require solvers to address.
