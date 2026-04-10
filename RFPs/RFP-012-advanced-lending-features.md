---
id: RFP-012
title: Curated Lending Vaults
tier: L
funding: $XXXXX
status: open
dependencies: RFP-008
category: Applications & Integrations
---


# RFP-012 — Curated Lending Vaults

## 🧭 Overview

Build a curated vault layer on top of the isolated-market lending
protocol delivered by [RFP-008](./RFP-008-lending-borrowing-protocol.md).
Each vault accepts deposits of a single loan token, issues transferable
vault share tokens (SPL tokens), and allocates deposits across multiple
Morpho-style lending markets according to a curator-defined strategy.
This is the MetaMorpho equivalent for LEZ: it transforms raw isolated
markets into a managed lending product with passive yield, risk
curation, and composable receipt tokens.

Morpho Blue's isolated-market core is powerful but requires lenders to
choose individual markets manually (evaluating collateral quality,
oracle reliability, LLTV risk). Curated vaults abstract this complexity:
depositors supply a single token, the curator allocates across vetted
markets, and depositors earn blended yield. On Ethereum, MetaMorpho
vaults reached [DATA NEEDED] in deposits, demonstrating that the
vault layer is where most passive capital concentrates. The applying
team should ideally be the same team that delivered RFP-008, or have
deep familiarity with the deployed protocol's internals.

## 🔥 Why This Matters

The isolated-market model in RFP-008 optimises for safety and
permissionlessness, but it fragments liquidity: each market is
independent, and lenders must actively manage exposure across markets.
Most passive lenders (the largest source of TVL in any lending
ecosystem) will not do this. Without curated vaults, the protocol's
TVL ceiling is limited to active, sophisticated lenders.

Curated vaults solve this by aggregating passive capital and directing
it where it earns the best risk-adjusted yield. On Ethereum, the
majority of Morpho deposits flow through MetaMorpho vaults rather than
directly into isolated markets. For the Logos ecosystem, vaults
transform the lending protocol from a tool for power users into a
passive yield product accessible to all participants, dramatically
expanding the protocol's addressable market and TVL.

Vaults also create a natural role for risk specialists (curators) who
compete on yield and safety. This decentralises risk management:
instead of a single governance body deciding which assets are safe,
multiple curators offer different risk/yield profiles, and depositors
choose the vault that matches their preference.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. **Vault creation**: anyone can create a vault by specifying the
   loan token, a vault name, and a vault symbol. The creator becomes
   the vault's initial curator. Vault creation is permissionless.
2. **Vault share tokens**: depositors receive SPL tokens representing
   their proportional share of the vault's total assets. Vault shares
   are freely transferable and composable with other LEZ programs.
   The share-to-asset exchange rate increases monotonically as interest
   accrues (absent bad debt).
3. **Deposit and withdraw**: users can deposit the vault's loan token
   and receive vault shares. Users can redeem vault shares for the
   underlying loan token, subject to available liquidity across the
   vault's allocated markets.
4. **Market allocation**: the curator (or an authorised allocator)
   allocates vault deposits across RFP-008 lending markets. The
   curator sets a supply cap per market, limiting the maximum amount
   the vault can supply to each market.
5. **Supply queue and withdraw queue**: the curator defines an ordered
   list of markets for deposits (supply queue) and withdrawals
   (withdraw queue). Deposits flow into markets in supply queue order
   until each market's cap is reached. Withdrawals draw from markets
   in withdraw queue order.
6. **Curator role**: the curator can update supply caps, reorder
   queues, add or remove markets from the vault's allocation set, and
   set a performance fee. The curator can transfer the curator role to
   another account.
7. **Allocator role**: the curator can authorise allocator accounts
   that may reallocate funds between the vault's approved markets
   within the curator's supply caps. Allocators cannot change caps,
   queues, or fees.
8. **Performance fee**: a configurable percentage of yield accrued by
   the vault is directed to the curator as a performance fee. The fee
   is denominated in vault shares and minted to the curator's account
   on each interaction that triggers interest accrual.
9. **Timelock for parameter changes**: changes to supply caps, market
   additions, curator transfer, and fee adjustments are subject to a
   configurable timelock. The timelock duration is set at vault
   creation and is immutable.
10. **Guardian role**: the curator can appoint a guardian that can
    revoke pending (timelocked) parameter changes before they take
    effect. The guardian cannot initiate changes, only cancel them.
    This is a safety mechanism against compromised curator keys.

#### Usability

1. All vault features are exposed in the SDK, CLI, and mini-app built
   in RFP-008; no separate front-end is required, only extension of
   the existing one.
2. The mini-app displays per-vault: current APY, total deposits,
   allocated markets with individual utilisation, supply caps and
   remaining capacity, and the curator's performance fee.
3. The mini-app displays per-user: vault share balance, current value
   in underlying token, and accrued yield since deposit.
4. When interacting via a private account, the SDK must handle the
   atomic deshield (deposit token + native gas) as a single
   indivisible user action, consistent with the RFP-008 privacy
   pattern.
5. When interacting via a private account, before each vault
   operation, the mini-app must show the estimated transaction fee and
   confirm that the user's shielded balance covers both the deposit
   amount and fees within the single deshield action.
6. Vault share tokens must be usable as collateral in RFP-008 lending
   markets (subject to an oracle being available for the vault share
   price).

#### Reliability

1. Vault share value (assets per share) is monotonically
   non-decreasing absent bad debt in underlying markets.
2. Supply caps are enforced atomically; concurrent deposits cannot
   jointly exceed a market's cap.
3. Withdrawal always succeeds if the total available liquidity across
   the vault's allocated markets covers the requested amount, even if
   individual markets have insufficient liquidity (the vault draws
   from multiple markets in queue order).
4. If a market in the vault's allocation incurs bad debt, the loss is
   socialised across all vault depositors proportionally. The vault
   must track and report realised bad debt per market.

#### Performance

1. Document the transaction size of each vault operation (deposit,
   withdraw, reallocate) on LEZ. LEZ's block size is limited and this
   budget may change during testnet.
2. A vault with up to 10 allocated markets must not exceed LEZ compute
   limits on any single deposit or withdrawal operation.

#### Supportability

1. The updated program is deployed and tested on LEZ devnet/testnet.
2. End-to-end integration tests run against a LEZ sequencer
   (standalone mode) and are included in CI; CI must be green on
   the default branch.
3. Every hard requirement in Functionality, Usability, Reliability,
   and Performance has at least one corresponding test.
4. A README addendum documents the vault features: vault creation,
   curator setup, market allocation, deposit/withdrawal flow, and
   guardian configuration.

#### + Privacy

1. The mini-app and SDK must support both direct public account
   interaction and the deshield→interact→reshield pattern for private
   account interaction with vaults. When a user chooses the private
   account path, the SDK must enforce the complete
   deshield→interact→reshield pattern; the reshield step must not be
   skippable.
2. When using the private account path, the mini-app must display a
   pre-confirmation summary for each vault operation that clearly
   identifies what will be visible on-chain (deposit/withdrawal
   amount, vault address, vault share mint/burn, ephemeral
   intermediary account) and what will remain private (the originating
   private account, the destination of re-shielded vault shares or
   withdrawn tokens, and any link between separate interactions by the
   same user).
3. The ephemeral public account (account A) created during the
   deshield step must never be reused across vault operations. Each
   interaction from a private account must use a freshly generated
   account with no prior on-chain history.

### Soft Requirements

If possible.

#### Functionality

1. **Vault performance history**: an on-chain or indexed record of
   historical vault APY, enabling depositors to evaluate curator
   track records before depositing.

#### Reliability

1. **Formal verification of vault invariants**: (a) vault share value
   cannot decrease except through realised bad debt, (b) supply caps
   are never exceeded, (c) timelock duration cannot be shortened after
   vault creation.

### Out of Scope

- Native stablecoin issuance (CDP-style minting against collateral)
- Governance token design and distribution
- Cross-chain liquidity or bridging
- Leveraged looping / one-click multiply products
- Any feature already delivered by RFP-008

### Privacy Architecture

See [Appendix: Lending Platform Context](../appendix/lending-platform.md#privacy-architecture).

## ⚠ Platform Dependencies

This RFP is open for proposal submission. However, development is blocked until the following is satisfied:

1. **RFP-008 is live on LEZ devnet/testnet**: this RFP extends the
   deployed protocol; it cannot proceed without the base layer.

All other platform primitives required by this RFP (including LP-0015
general cross-program calls, oracle provider, and on-chain clock) are
hard blockers for RFP-008 and will therefore be resolved before this
RFP opens. See
[Appendix: Lending Platform Context](../appendix/lending-platform.md#platform-dependencies)
for details.

## 👤 Recommended Team Profile

Team experienced with:

- Morpho Blue and MetaMorpho vault architecture (or equivalent
  vault/aggregator protocols such as Yearn, ERC-4626 vaults)
- Solana or SVM program development (Anchor or native)
- DeFi risk curation and yield optimisation strategies
- Token standard design (vault share token mechanics)
- Front-end development for DeFi applications

Ideally the same team that delivered RFP-008.

## ⏱ Timeline Expectations

Estimated duration: **12 weeks** (after platform dependencies are
resolved and RFP-008 is live).


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

- [RFP-008 — Lending & Borrowing Protocol](./RFP-008-lending-borrowing-protocol.md)
- [RFP-001 — Admin Authority Library](./RFP-001-admin-authority-lib.md) — reference pattern for admin-gated operations (F6, F10)
- [RFP-002 — Freeze Authority Library](./RFP-002-freeze-authority-lib.md)
- [Appendix: Lending Platform Context](../appendix/lending-platform.md) — shared privacy architecture and platform dependencies
- TODO: Oracle integration guide for LEZ
- TODO: SPEL framework documentation


## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
