# Appendix: Multisig and Coordination Ecosystem Behaviour

This appendix surveys how existing multisig implementations across the Bitcoin,
Ethereum, and Solana ecosystems behave, with a focus on a single product
question: **what data does a multisig involve, and which parts of it are visible
to the outside world?** It is written from a product angle (what an operator, a
counterparty, or a passive observer can and cannot see), not a
contract-internals angle.

All data is sourced from the
[research-multisig-sovereign](https://github.com/marclawclaw/research-multisig-sovereign)
vault; individual subject and pattern notes contain full citations. Access date
for all figures is 2026-07-06 unless otherwise noted.

This appendix contains facts and observations only. It does not state
requirements, and it does not decide which data *should* be private for a Logos
multisig: that is the subject of
[RFP-005](../RFPs/RFP-005-multisig-and-coordination.md).

## Protocols considered

Nine implementations were studied, spanning the on-chain program layer and the
coordinator/UI layer across three ecosystems. Adoption is measured by GitHub
stars: multisig software holds assets in custody accounts rather than a DeFi
liquidity pool, so total-value-locked (TVL) is not an applicable metric, and no
authoritative TVL figure is published for the custody layer.

| Implementation                                                  | Ecosystem / Layer              | Stars    | Last commit | Licence                     |
| --------------------------------------------------------------- | ------------------------------ | -------- | ----------- | --------------------------- |
| Bitcoin native script multisig (P2SH / P2WSH), via rust-bitcoin | Bitcoin / on-chain library     | 2,641    | 2026-07-05  | CC0-1.0                     |
| FROST threshold signatures, via Frostsnap                       | Bitcoin / threshold-sig        | 147      | 2026-07-06  | MIT                         |
| Sparrow Wallet                                                  | Bitcoin / coordinator UI       | 2,029    | 2026-07-04  | Apache 2.0                  |
| Specter Desktop                                                 | Bitcoin / coordinator server   | 838      | 2026-06-30  | MIT                         |
| Safe (Gnosis Safe) smart account                                | Ethereum / on-chain program    | 2,165    | 2026-06-05  | LGPL-3.0-only               |
| Safe self-hosted stack (UI + relay + CLI)                       | Ethereum / coordinator stack   | 576 (UI) | 2026-07-03  | GPL-3.0 / MIT / FSL-1.1-MIT |
| ERC-4337 smart account, via ZeroDev Kernel                      | Ethereum / account abstraction | 247      | 2026-06-30  | MIT                         |
| Squads smart-account-program (v4)                               | Solana / on-chain program      | 42       | 2026-05-25  | AGPL-3.0                    |
| Squads v4 public UI                                             | Solana / coordinator UI        | 30       | 2025-03-06  | MIT                         |

## 1. What data a multisig involves, and whether it can be private

A multisig is not a single secret; it is a collection of distinct data and
metadata items, each of which can be independently visible or hidden. The table
below enumerates those items. For each, it records whether that item **could**
be kept private on the Logos Execution Zone (LEZ), and what existing multisig
implementations do with it today.

"Could be private on LEZ" refers to the native execution model of the Logos
Execution Environment (LEE), described in section 2: the same program can run
over public accounts (visible on-chain) or private accounts (only a commitment
to the post-state is published). It states a capability of the platform, not a
recommendation.

Where identity is concerned, the precise property is **unlinkability between a
user's account and its role in a multisig**, not "hiding a user's identity." A
signer always has an account; what can be made private is the *link* between
that account and the fact that it holds a signer or admin role in a specific
multisig.

| #   | Data / metadata item                                                                                 | Could be private on LEZ?  | What existing multisigs do                                                                                                                                                                                   |
| --- | ---------------------------------------------------------------------------------------------------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | Existence: that an account acts as a multisig at all                                                 | Yes                       | Safe, Squads, ERC-4337: always public (the account is visibly a multisig contract/PDA). FROST and MuSig2 n-of-n: hidden (indistinguishable from a single-key account).                                       |
| 2   | Member set: which accounts are signers (account↔signer-role link)                                    | Yes                       | Safe, Squads, ERC-4337: always public (signer accounts are on-chain state). Bitcoin script: public keys revealed at spend. FROST / MuSig2: hidden.                                                           |
| 3   | Threshold configuration: the M and N of M-of-N                                                       | Yes                       | Safe, Squads, ERC-4337: always public. Bitcoin script: revealed at spend. FROST / MuSig2: hidden.                                                                                                            |
| 4   | Approval attribution: which signer approved a given action (account↔approval link)                   | Yes                       | Squads: always public (each approval is separate on-chain state). Safe: revealed in the executing transaction, or earlier via on-chain `approveHash`. FROST / MuSig2: hidden (no per-signer trace survives). |
| 5   | Pending proposal: that an action is proposed and awaiting quorum, plus its metadata                  | Yes                       | Squads: always public (a pending proposal is a live on-chain account). Safe: typically off-chain until execution; the relay operator running the transaction service sees it.                                |
| 6   | Action payload: the target, calldata, and amounts being executed                                     | Yes                       | Safe, Squads, ERC-4337: public at execution. Bitcoin: outputs and amounts public on-chain.                                                                                                                   |
| 7   | Holdings: the balance held by the multisig's vault                                                   | Yes                       | Safe, Squads, ERC-4337: always public (balances are on-chain state). Bitcoin: public per-UTXO, pseudonymous.                                                                                                 |
| 8   | Execution linkage: that a given on-chain effect originated from this multisig (multisig↔effect link) | Yes                       | Safe, Squads, ERC-4337: public (the effect is a transaction from the multisig account). Bitcoin: linkable via the spending input.                                                                            |
| 9   | Coordination content: the discussion and signing messages exchanged between signers                  | Yes (off-chain by nature) | No researched on-chain multisig addresses this. Safe's transaction service leaks proposal metadata to whoever runs the relay. Bitcoin coordinators use an external channel (email, SD card, QR).             |
| 10  | Co-signing social graph: that this set of accounts jointly administers a multisig                    | Yes                       | Not addressed by any researched multisig. On account-model chains it is directly readable from the on-chain signer set (item 2); on Bitcoin it is inferable at spend.                                        |

The prose below explains each item's cross-ecosystem behaviour precisely.

### 1.1 Existence, members, threshold (items 1-3)

On the account-model chains, these three items are inseparable from how the
quorum is enforced. Safe stores its owner set as a linked list and its threshold
as a storage slot; Squads stores its members and threshold in a settings
account; ZeroDev Kernel stores guardian addresses and weights in contract
storage. In all three, the chain must read this state to enforce the rule, so
the state is public by construction. See \[[patterns/squads-permission-model]\]
and the Safe contract architecture note \[[tools/safe-smart-account]\].

Bitcoin native script (P2SH / P2WSH) reveals the full script (all public keys
and the threshold) at spend time. Taproot changes this: a key-path spend using
MuSig2 reveals nothing (see section 3), while a script-path spend reveals only
the executed leaf and hides sibling leaves via the Merkle tree
(\[[patterns/taproot-key-path-musig2]\]).

FROST is the outlier: the threshold is enforced cryptographically off-chain, and
the chain sees a single Schnorr signature. Signer count, threshold, and the fact
that the account is a multisig at all are invisible
(\[[patterns/privacy-landscape]\]).

### 1.2 Approval attribution and pending proposals (items 4-5)

Squads persists each approval as separate on-chain state before execution, which
produces a live, queryable pending-proposal view: any observer can see who has
approved and who has not, in real time
(\[[patterns/squads-transaction-lifecycle]\]). Safe typically collects
signatures off-chain and presents them together in the final `execTransaction`
call; owners may optionally record consent early with the on-chain `approveHash`
function.

It is worth stating precisely what this difference is and is not. Both models
yield an on-chain, verifiable record of who authorised an action *after it
executes*: Safe carries the signatures in the executing transaction, and Squads
carries them in the proposal account. Neither model has a public authorisation
audit trail that the other lacks. The genuine difference is only *staging*:
Squads makes approvals public before execution as a live pending view; Safe does
not. This is a coordination-visibility difference, not an audit-trail advantage
of one over the other.

A separate and distinct property, noted in the Bybit analysis
(\[[patterns/security-incidents-safe]\]), is *intent verifiability at signing
time*: with off-chain signature collection there is no on-chain record that a
signer intended the transaction they actually signed, which is the gap both the
Bybit (February 2025, ~US$1.5B) and WazirX (July 2024, ~US$235M) incidents
exploited at the UI layer. This is about what a signer can verify at the moment
of signing, not about what survives on-chain afterwards.

### 1.3 Action, holdings, execution linkage (items 6-8)

On every researched account-model multisig, the executed action and its effects
are public: the transaction, its target, its calldata, and the resulting balance
changes are all on-chain. The multisig's vault balance is likewise public
contract or PDA state. On Bitcoin, amounts and outputs are public per-UTXO and
pseudonymous; the spending input links the effect back to the multisig UTXO.

No researched implementation hides the link between a multisig and the on-chain
effects it produces. Where privacy exists (FROST, Taproot key-path), it hides
the *multisig structure*, not the *transaction effect*: the payment itself is
still a visible Bitcoin transaction, just one that looks single-signer.

### 1.4 Coordination content and social graph (items 9-10)

Coordination content is off-chain in every ecosystem, because signing protocols
define message formats but not transports (see section 4). What differs is what
the transport leaks. Bitcoin coordinators (Sparrow, Specter) exchange partially
signed transactions over a channel the user chooses (SD card, QR, USB, file
share) and add no transport of their own
(\[[patterns/inband-signing-coordination]\]). Safe's self-hosted transaction
service is a purpose-built relay; whoever operates it sees pending transaction
hashes, collected signatures, and which owners have approved
(\[[patterns/off-chain-signature-relay]\],
\[[patterns/safe-service-architecture]\]).

The co-signing social graph (that a particular set of accounts jointly
administers one multisig) is not treated as a distinct privacy concern by any
researched implementation. On account-model chains it falls directly out of the
public signer set; on Bitcoin it is inferable at spend. No surveyed tool offers
a metadata-resistant coordination channel.

## 2. The LEE execution model (public vs private accounts)

This section records how the Logos Execution Environment handles state, because
it is what makes every "yes" in the section 1 table possible. It is a
description of the platform, not a proposed design.

Source: Logos documentation, "Introduction to the Logos Execution Zone"
(https://docs.logos.co/lez), accessed 2026-07-07.

The LEE separates persistent state into two kinds of account, and runs the
**same program bytecode** over either:

- **Public accounts** are stored on-chain as a map from account ID to state. The
  account ID is publicly visible. Executions that modify public accounts are
  validated by LEZ validators through transparent re-execution, like a standard
  RISC-V call.

- **Private accounts** are stored locally on the account holder's own node. When
  their state changes, only a **commitment** to the new post-state is published
  on-chain, together with a **nullifier** that retires the previous commitment
  so it cannot be reused. Correctness is proven with a Risc0 zero-knowledge
  proof that validators verify without seeing the underlying data. Each private
  account carries a nullifier keypair (the private key authorises executions;
  the public key serves as the account ID) and a viewing keypair (used to
  produce and verify proofs without revealing the owner).

The product consequence for multisig is direct: running a multisig program over
private accounts means the chain records only a post-state commitment, a
nullifier, and a validity proof. It does not record the members, the threshold,
the approvals, the pending proposal, or the action. Privacy on LEZ is therefore
not a feature that must be added to a multisig; it is the native
private-execution mode of the platform applied to the multisig's accounts. This
is what distinguishes LEZ from every account-model chain in this survey, where
the chain must read the quorum state in the clear in order to enforce it.

## 3. Quorum privacy across ecosystems

The degree to which a multisig's structure is visible follows directly from
where the quorum is enforced. The pattern is consistent across every
implementation studied (\[[patterns/privacy-landscape]\]):

| Posture                                        | How achieved                                                                         | What remains visible                              |
| ---------------------------------------------- | ------------------------------------------------------------------------------------ | ------------------------------------------------- |
| Full structural privacy (FROST; MuSig2 n-of-n) | Quorum enforced off-chain in a cryptographic session; one Schnorr signature on-chain | Nothing about the multisig; looks single-key      |
| Partial (Taproot script-path, via MAST)        | Unused script leaves hidden in a Merkle tree                                         | The executed quorum leaf (its keys and threshold) |
| None (P2WSH; Safe; Squads; ERC-4337)           | Quorum enforced by reading on-chain state                                            | Members, threshold, and every approval            |

Two observations hold across the whole survey:

1. Every implementation that achieves on-chain structural privacy does so by
   moving multisig logic *off-chain* into a cryptographic session. Every
   implementation that keeps multisig logic *on-chain* is transparent by
   construction, because the chain must read the quorum to enforce it.

2. MuSig2 achieves this only for n-of-n (all signers must participate). Genuine
   k-of-n structural privacy on Bitcoin requires FROST, which as of 2026-07-06
   is still at BIP-draft stage and supported only by purpose-built hardware
   (Frostsnap's ESP32-C3 device); no mainstream hardware wallet supports it
   (\[[patterns/frost-taproot-privacy]\], \[[tools/bitcoin-frost-threshold]\]).

The centralised industry "solution" to account-model privacy (a trusted relayer
or co-processor that manages signer identity off-chain and presents a single
abstract signer on-chain) trades sovereignty for privacy. Community proposals
for private Safe ownership (zkSafe, Semaphore-based anonymous signer modules)
exist but none reached production sovereign tooling; they required either a
trusted setup or a centralised coordinator (\[[patterns/privacy-landscape]\]).

## 4. Coordination: how signers reach quorum

Signing coordination is either **in-band** (the signing-round messages flow
through the same protocol or network as the final transaction) or
**out-of-band** (signers exchange signing material through a medium external to
the protocol). The split follows the execution model
(\[[patterns/inband-signing-coordination]\]):

| Implementation               | Coordination mode | Channel                                                                                           |
| ---------------------------- | ----------------- | ------------------------------------------------------------------------------------------------- |
| Squads v4                    | In-band           | Solana chain: proposals and per-member approvals are native instructions                          |
| Safe via `approveHash`       | In-band           | EVM chain: each owner records approval on-chain                                                   |
| Safe via transaction service | Relay-mediated    | A purpose-built HTTP relay, part of the Safe stack but not the chain                              |
| ERC-4337 (ZeroDev Kernel)    | Out-of-band       | Guardian signatures aggregated off-chain before submission to a bundler                           |
| FROST (Frostsnap)            | In-band (session) | The coordinator app's own wire protocol carries DKG and signing rounds; no external file exchange |
| Sparrow, Specter             | Out-of-band       | Partially-signed transactions exchanged via SD card, QR, USB, or file share                       |

Three facts about coordination bear on a product decision:

- **Bitcoin's PSBT (BIP-174/370) is a container format, not a transport.** It
  defines how to package a partially-signed transaction but specifies no way to
  deliver it between signers. Sparrow and Specter document several transfer
  methods precisely because none is built in
  (\[[patterns/psbt-coordinator-pattern]\]).

- **Account-model chains can coordinate in-band by storing coordination state
  on-chain,** as Squads does with proposal and approval accounts. Safe's
  on-chain `approveHash` does the same but is rarely used because paying gas per
  signer per proposal is a deterrent; off-chain collection via the relay is the
  common path.

- **Threshold-signature schemes carry their own session protocol.** FROST's DKG
  and signing rounds are a structured message exchange that runs within the
  protocol; a coordinator app (or a peer-to-peer layer) implements it without
  any external messaging service (\[[patterns/frost-usb-coordination]\],
  \[[patterns/frost-dkg-ceremony]\]). Frostsnap deliberately runs this over USB
  serial rather than airgapped QR, because a k-of-n session over QR codes needs
  many scans per transaction and the usability cost drives users to skip
  verification.

The product gap this exposes: no surveyed sovereign multisig provides an
encrypted, metadata-resistant coordination channel. Coordination is either
public on-chain state (Squads), a relay that sees the metadata (Safe), or a
user-supplied external channel with no privacy guarantee (Bitcoin). An
end-to-end-encrypted messaging channel dedicated to a multisig's signers, such
as the Logos chat module, is not matched by any implementation in this survey.

## 5. Roles, policies, and extensibility

Beyond the basic M-of-N check, the surveyed implementations offer graduated
controls that shape what an operational multisig can express.

- **Role separation.** Squads assigns each signer a three-bit permission mask
  (Initiate / Vote / Execute), so a proposing key need not be a voting key, and
  execution can be a separate role. Program invariants require at least one
  signer with each permission and forbid an impossible threshold
  (\[[patterns/squads-permission-model]\]). ZeroDev Kernel expresses roles as
  weighted guardians, where the threshold is a minimum cumulative weight
  (\[[patterns/weighted-multisig-onchain-vote]\]).

- **Time locks and cancellation.** Squads supports a per-multisig time lock (0
  to about three months) enforced on-chain between approval and execution, and
  allows an approved proposal to be cancelled by a threshold of cancellation
  votes before it executes (\[[patterns/squads-transaction-lifecycle]\]). Safe's
  core contract has no time lock; it must be added as a guard or module
  (\[[tools/safe-smart-account]\]).

- **Spending limits and policies.** Squads supports policy accounts with their
  own signer set and threshold for specific action types, and spending-limit
  authorities (\[[patterns/squads-policy-framework]\]). Safe offers an Allowance
  Module for spending limits.

- **Extensibility, and its cost.** Safe's guard and module system lets arbitrary
  contracts hook execution (guards) or execute without the M-of-N check
  (modules) (\[[patterns/guard-module-system]\]). This is powerful and
  dangerous: a module has unlimited authority, and the SquidRouter incident (May
  2026, ~US$3.2M) drained 86 Safes through a vulnerable third-party module that
  owners had enabled (\[[patterns/security-incidents-safe]\]). ERC-4337 uses a
  modular validator-plugin model with a defined install/uninstall lifecycle
  (\[[patterns/erc7579-module-lifecycle]\],
  \[[patterns/modular-validator-plugin]\]).

## 6. Composition: how a multisig triggers actions

A multisig is only useful if it can act on other programs. The surveyed
implementations delegate execution rather than embedding target logic:

- **Safe** executes an arbitrary `to` / `value` / `data` call (or delegatecall)
  through `execTransaction`, and batches multiple sub-calls through the
  MultiSend library (\[[tools/safe-smart-account]\]).

- **Squads** executes a stored instruction against a target program through a
  vault cross-program invocation, and supports batches executed sequentially
  after a single approval (\[[patterns/squads-transaction-lifecycle]\]).

- **The Logos public multisig sample app** (`logos-co/lez-multisig`) follows the
  Squads model on LEZ: a proposal stores a serialised instruction and target
  program ID, and on execution the multisig emits a LEZ `ChainedCall` to the
  target, so the multisig never modifies external state directly and composes
  with any LEZ program. Source: `logos-co/lez-multisig` README and SPEC,
  accessed 2026-07-07. Its accounts (multisig state, proposal, vault) are public
  LEZ accounts; per the section 1 table, the same design run over private
  accounts would publish only commitments.

The composition mechanism is orthogonal to privacy: whether the multisig's own
state is public or private, it still delegates to a target program, and the
target execution's own visibility follows that target's account choices.

## 7. Hardware signing and sovereignty (operational notes)

Two operational facts recur across the survey and bear on any real deployment:

- **Hardware-wallet support is uneven.** Bitcoin has the broadest coverage: PSBT
  and the hardware wallet interface (HWI) support Coldcard, Trezor, Ledger,
  Jade, BitBox02, and many airgapped signers
  (\[[patterns/airgapped-signing-pattern]\]). EVM signing uses EIP-712 typed
  data on Ledger, Trezor, and Keystone. Solana hardware signing requires
  blind-signing in practice. FROST supports only Frostsnap's purpose-built
  device, no mainstream hardware wallet.

- **Sovereignty has different costs per stack.** Bitcoin coordinators need a
  Bitcoin Core or Electrum backend. Safe can run as a full Docker stack (many
  services) or CLI-only with no infrastructure. Squads needs only an RPC node.
  ERC-4337 requires a bundler, which must be separately self-hosted or is
  defaulted to a hosted provider (\[[patterns/bundler-liveness-dependency]\],
  \[[patterns/server-first-coordinator-pattern]\]).

## 8. Summary of observations

- A multisig involves at least ten distinct data and metadata items (section 1);
  on every account-model chain surveyed, most of them are public by construction
  because the chain must read the quorum to enforce it.

- The only implementations that keep multisig structure private (FROST, MuSig2
  n-of-n) do so by moving the quorum off-chain into a cryptographic session, at
  the cost of hardware and tooling immaturity and, for MuSig2, an n-of-n-only
  limitation.

- The LEE private-account model is the one execution environment in this survey
  where a fully programmable M-of-N multisig can run with its state (members,
  threshold, approvals, action) published only as commitments, without a trusted
  coordinator and without abandoning k-of-n.

- "On-chain versus off-chain approval" is a coordination-visibility choice, not
  an audit-trail trade-off: both models yield an on-chain authorisation record
  at execution.

- No surveyed sovereign multisig offers an encrypted, metadata-resistant
  coordination channel for its signers.

## References

| Source                                   | URL                                                                                  | Access date |
| ---------------------------------------- | ------------------------------------------------------------------------------------ | ----------- |
| research-multisig-sovereign vault        | https://github.com/marclawclaw/research-multisig-sovereign                           | 2026-07-06  |
| Introduction to the Logos Execution Zone | https://docs.logos.co/lez                                                            | 2026-07-07  |
| Squads smart-account-program             | https://github.com/Squads-Protocol/smart-account-program                             | 2026-07-06  |
| Safe smart account                       | https://github.com/safe-global/safe-smart-account                                    | 2026-07-06  |
| Frostsnap                                | https://github.com/frostsnap/frostsnap                                               | 2026-07-06  |
| ZeroDev Kernel                           | https://github.com/zerodevapp/kernel                                                 | 2026-07-06  |
| Sparrow Wallet                           | https://github.com/sparrowwallet/sparrow                                             | 2026-07-06  |
| Specter Desktop                          | https://github.com/cryptoadvance/specter-desktop                                     | 2026-07-06  |
| rust-bitcoin                             | https://github.com/rust-bitcoin/rust-bitcoin                                         | 2026-07-06  |
| BIP-327 (MuSig2)                         | https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki                       | 2026-07-06  |
| BIP-341 (Taproot)                        | https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki                       | 2026-07-06  |
| Bybit incident analysis (Check Point)    | https://research.checkpoint.com/2025/the-bybit-incident-when-research-meets-reality/ | 2026-07-06  |
| WazirX hack analysis (QuillAudits)       | https://www.quillaudits.com/blog/hack-analysis/wazirx-235m-hack                      | 2026-07-06  |
| logos-co/lez-multisig                    | https://github.com/logos-co/lez-multisig                                             | 2026-07-07  |
