---
id: RFP-007
title: Privacy-Preserving Token Vesting
tier: L
funding: $XXXXX
status: draft
category: Applications & Integrations
---


# RFP-007 — Privacy-Preserving Token Vesting

## 🧭 Overview

Build a token vesting program on LEZ that locks project, team, or investor
tokens against configurable time-based schedules — cliff, linear, or
milestone-based. Recipients claim vested tokens via a
deshield→claim→re-shield pattern: each claim is executed from a fresh,
single-use public account funded by deshield, and the output is immediately
re-shielded to the recipient's private account. Neither the link between
the claim and the recipient's private account identity, nor the subsequent
movements of claimed tokens, are traceable on-chain.

Token vesting is a prerequisite for credible token launches. Every protocol
that issues a token must lock some allocation for founders, early
contributors, or ecosystem reserves. The team building this should have
experience with SVM program development and DeFi token mechanics. Familiarity
with the deshield→action→re-shield privacy pattern (see
[RFP-004](./RFP-004-privacy-preserving-dex.md)) is strongly recommended.

## 🔥 Why This Matters

On transparent chains, vesting contracts (Streamflow, Sablier, Team.Finance)
are surveillance tools: every cliff unlock, every linear release, and every
claim is publicly linkable to a beneficiary address, exposing token unlock
schedules to the market and enabling adversarial trading against known unlock
events. A Solana founder's vesting cliff is trivially queryable on-chain;
traders index these events and position against them before the beneficiary
can act.

On LEZ, the deshield→claim→re-shield pattern makes claims unlinkable to
the recipient's private identity. Observers see that tokens were claimed from
a vesting schedule; they cannot determine which private account holds those
tokens or trace subsequent movements. This makes LEZ the only environment
where teams can vest tokens without creating a public unlock calendar for
adversarial front-running.

Vesting is also a composition primitive. A launchpad (see
[RFP-006](./RFP-006-token-launchpad.md)) or investor round on LEZ needs
vesting to be in place before tokens can be distributed to contributors and
early backers. This RFP is a direct prerequisite for a credible token
economy on LEZ.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. Implement a vesting program on LEZ that locks tokens in escrow against
   a configured schedule. The program must support:
   - **Cliff + linear vesting**: tokens become claimable in a lump sum at
     the cliff date, then linearly over the remaining duration (e.g.,
     1-year cliff, 3-year linear release).
   - **Fully linear vesting**: no cliff; tokens accrue continuously from
     the start date.
   - **Milestone-based vesting**: the schedule creator explicitly
     signals completion of discrete tranches; each signal unlocks a
     fixed token amount for the beneficiary to claim.
2. The schedule creator specifies the beneficiary as a public account
   address at schedule creation. Recipients claim vested tokens by
   executing a claim from a fresh, single-use public account funded
   via deshield (native token for gas only — no input token is required
   since the vested tokens come from the program escrow), and re-shield
   the claimed tokens to their private account immediately. The schedule creator does not learn the
   recipient's private account address from this interaction.
3. The schedule creator can cancel an active schedule at any time,
   returning the unclaimed, unvested token balance to the creator's
   account. Tokens already vested but not yet claimed remain claimable
   by the beneficiary after cancellation.
4. The schedule creator can configure whether the vesting position
   (beneficiary rights) is transferable to a new public account address.
   Transferability must be set at schedule creation and cannot be changed
   afterwards.
5. Multi-recipient batch creation: a creator can set up vesting schedules
   for multiple recipients in a single operation, up to the LEZ transaction
   size limit. Document the maximum number of recipients achievable in one
   transaction.
6. The program emits a log event (using the LEZ event/log mechanism — see
   [LP-0012](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md))
   for each state transition: schedule creation, claim, cancellation,
   milestone signal, and beneficiary transfer. Event schema must be
   documented.

#### Usability

1. Provide an SDK that can be used to build Logos modules interacting with
   the vesting program. The SDK must expose the full lifecycle: create
   schedule, query claimable amount, claim, cancel, signal milestone, and
   transfer beneficiary. The SDK must support both direct public account
   interaction and the deshield→claim→re-shield pattern for private
   account interaction. When the private account path is used, the SDK
   must handle the gas-only deshield as a single indivisible action.
2. Provide a Logos mini-app GUI with local build instructions, downloadable
   assets, and loadable in Logos app (Basecamp) via git repo. The mini-app
   must cover at minimum:
   - **Recipient view**: current vesting position, total locked, amount
     claimable now, next unlock event, claim action.
   - **Creator view**: create a new schedule (all parameters), list active
     schedules, cancel a schedule, signal a milestone.
3. The mini-app must display a pre-claim confirmation summary: claimable
   amount and estimated transaction fee. When using the private account
   path, it must also confirm that the recipient's shielded balance covers
   the gas-only deshield; a clear, actionable error must be shown if the
   balance is insufficient.
4. When using the private account path, the mini-app must display a
   privacy disclosure before each claim, identifying what will be visible
   on-chain (claim amount, ephemeral public account address, vesting
   schedule program address) and what will not be traceable (the
   recipient's private account and the subsequent destination of claimed
   tokens).
5. Provide an IDL for the vesting program using the
   [SPEL framework](https://github.com/logos-co/spel).
6. Failed claims must return clear, actionable error messages — including
   the case where nothing is yet claimable (with the next unlock
   time displayed).

#### Reliability

1. A claim is atomic: a failed or rejected claim does not consume vested
   tokens and leaves the claimant able to retry without loss.
2. A schedule cancellation is atomic: partial failure leaves the schedule
   in its prior state, not an undefined one.
3. Concurrent claims against independent schedules must not corrupt shared
   program state.
4. Milestone signalling is idempotent per milestone index: signalling the
   same milestone twice must be rejected with a deterministic error and
   must not double-unlock tokens.

#### Performance

1. A single claim completes within one LEZ transaction.
2. Document the compute unit (CU) cost of each operation: create schedule,
   claim, cancel, signal milestone, transfer beneficiary. Note the LEZ
   testnet version against which measurements were taken, as the
   per-transaction compute budget may change.

#### Supportability

1. The vesting program is deployed and tested on LEZ devnet/testnet.
2. End-to-end integration tests run against a LEZ sequencer (standalone
   mode) and are included in CI. CI must be green on the default branch.
3. Every hard requirement in Functionality, Usability, Reliability, and
   Performance has at least one corresponding test.
4. A README documents end-to-end usage: deployment steps, program
   addresses, and step-by-step instructions for creating a schedule,
   claiming, and cancelling via CLI and mini-app.

#### + Privacy

1. The mini-app and SDK must support both direct public account
   interaction and the deshield→claim→re-shield pattern for private
   account interaction. When a user chooses the private account path,
   the SDK must enforce the complete deshield→claim→re-shield pattern —
   the re-shield step must not be skippable.
2. When using the private account path, the mini-app must display the
   pre-confirmation privacy disclosure described in the Usability section
   before each claim, identifying what is visible on-chain (claim amount,
   ephemeral intermediary account) and what remains private (the
   originating private account, the destination of re-shielded tokens,
   and any link between separate claims by the same recipient).
3. When using the private account path, the SDK must validate that the
   re-shield target is a private (shielded) account before submitting
   the transaction, and reject with an explicit error if it is not.
4. The ephemeral public account created during the deshield step must
   never be reused across operations. Each claim from a private account
   must use a freshly generated account with no prior on-chain history.

### Privacy Architecture

Vesting schedule parameters (token, total amount, schedule type, cliff
date, end date, beneficiary public address) are public on-chain state.
This is a deliberate architectural choice: public state enables
permissionless observability of schedule terms and composability with
other programs.

User privacy is optionally enforced at the UX layer. The mini-app and
SDK support both direct public account interaction and private account
interaction via the deshield→claim→re-shield pattern. When users opt to
interact from a private account, the SDK must enforce the complete
pattern as described below.

#### Why public accounts are required

The vesting program is public on-chain state. Like all LEZ programs, it
can only release tokens to public token accounts — it cannot write directly
to a shielded balance. Re-shielding is a separate operation performed by
the account owner after receiving the tokens. This is the same fundamental
constraint as the DEX and lending protocol.

#### Interaction flow

For every claim from a private account:

1. The recipient initiates the action from their private account. The SDK
   deshields a small amount of native token for gas to a **fresh,
   single-use** public account (account A) with no prior on-chain history.
   Unlike the DEX, the recipient does not deshield any project token — the
   vested tokens come *from* the program, not from the recipient.
2. Account A submits the claim to the vesting program. The program releases
   the vested amount to account A's token account.
3. Account A re-shields the claimed tokens to the recipient's private
   account. Account A is never reused.

> **Gas:** Gas must come exclusively from the deshield in step 1. Funding
> account A from any external source — such as a CEX withdrawal or a
> known wallet — creates an on-chain link to an existing identity and
> breaks the privacy guarantee. The SDK must make this impossible.

#### What is public (observable on-chain)

- All vesting schedule parameters: token, total amount, schedule type,
  cliff date, end date, beneficiary public address.
- All claims: amount claimed, the ephemeral intermediary account address,
  and timestamp.
- Cancellations and milestone signals.

#### What is private (when using the private account path)

- Which private account the claimed tokens are re-shielded into.
- Any link between multiple claims by the same recipient (no on-chain
  linkability across ephemeral accounts).
- Subsequent movements of claimed tokens after re-shielding.

### Soft Requirements

- Support for streaming vesting: tokens accrue per block or per second and
  are claimable at any point (Sablier-style continuous streaming), rather
  than discrete unlock events.
- The creator can nominate a separate cancellation authority that is
  distinct from the creator wallet (useful when vesting is managed by a
  multisig or DAO).

## 👤 Recommended Team Profile

Team experienced with:

- Solana or SVM program development (Anchor or native)
- Token program and account model (ATAs, PDAs, token accounts)
- DeFi smart contract security and testing
- Front-end development for wallet/DeFi applications
- Familiarity with privacy interaction patterns (shielded accounts,
  deshield/re-shield — see RFP-004)

## ⏱ Timeline Expectations

Estimated duration: **10–12 weeks**


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

- [Logos Documentation](https://github.com/logos-co/logos-docs)
- [RFP-004 — Privacy-Preserving DEX](./RFP-004-privacy-preserving-dex.md)
  (canonical reference for the deshield→action→re-shield pattern)
- [LP-0012 — Event/Log mechanism for LEZ](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md)
- [LP-0013 — Token program improvements: mint authorities](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0013.md)
- [Sablier v2 — streaming vesting reference](https://github.com/sablier-labs/v2-core)
- [Streamflow Finance — vesting reference](https://streamflow.finance/)

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
