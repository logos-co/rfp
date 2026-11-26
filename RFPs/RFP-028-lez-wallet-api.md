---
id: RFP-028
title: LEZ Wallet API
tier: L
status: internal
category: Developer Tooling & Infrastructure
dependencies: []
---

# RFP-028 — LEZ Wallet API

> **Note.** This specification describes an outcome that may benefit the Logos
> ecosystem. It is a proposal rather than an instruction. Its requirements
> reflect the technical compatibility with the Logos technology stack and are
> the criteria against which proposals and milestones are evaluated. Logos makes
> no representation as to the legal or regulatory treatment of this
> specification or any implementation of it in any jurisdiction.
>
> This specification may also refer to other RFPs, LPs or their specifications.
> Such references are made for the purpose of identifying technical
> requirements, capabilities, dependencies or criteria applicable to a
> Submission under this RFP, including components that may need to be developed
> or incorporated as part of that Submission.
>
> Such references do not, in themselves, require or direct an Applicant to use,
> extend, modify or otherwise rely on any previous code, implementation or
> Submission associated with the referenced RFP, LP or specification. Where such
> references are made, it is for the Applicant to determine how best to develop
> or incorporate the relevant components to satisfy the requirements of the
> current RFP, whether to develop them independently or otherwise.
>
> Teams implementing it are solely responsible for (i) assessing the risks and
> implications of what they build; (ii) obtaining their own professional advice;
> and (iii) for complying with any legal and regulatory requirements that apply
> to them. Software developed under the Program is published and maintained by
> its developers, not by Logos.
>
> Anyone who chooses to deploy, host, operate or use software developed under
> the Program, whether or not they were awarded a grant under the Program, does
> so at their own risk and is solely responsible for complying with any legal or
> regulatory requirements that apply to them. See the
> [Terms & Conditions](../TERMS_AND_CONDITIONS.md).
>
> Deploying the software described in this RFP, operating any service based on
> it, or carrying on business through it may amount to regulated activity in
> some jurisdictions, including where it involves holding or managing users'
> assets or providing services to others. Whoever conducts any such activity
> does so as principal, in their own name, and is solely responsible for
> assessing its regulatory treatment, including any licensing, registration,
> sanctions or anti-money laundering obligations that may apply to them. Logos
> does not make any representation, provide any advice or assume any
> responsibility in respect of any such determination or compliance.

## 🧭 Overview

This RFP specifies the LEZ wallet API: the API of the LEZ wallet module. The
wallet module holds keys and does everything that needs them: it derives
accounts, tracks the state of the accounts it controls or can view, builds and
simulates transactions, generates the proofs a privacy-preserving transaction
carries, signs, and hands the result to a LEZ node for the network.

It is the wallet-side counterpart to the
[LEZ Node API (RFP-027)](./RFP-027-lez-node-api.md). The split follows the
target architecture in
[Integrate Logos into your application](https://docs.logos.co/build-an-app/integrate-logos):
each Logos chain is served by a wallet module, which runs where the keys are,
and a node module, which follows the chain and holds no keys. The wallet reaches
the chain **only through the LEZ Node API**, so the node can run in the same
application, on another device the user owns, or in an integrator's
infrastructure without any change to the wallet.

Today the `lez_core` Logos Core module provides the LEZ wallet over the
`wallet_ffi` crate and reaches a sequencer directly over HTTP. This RFP replaces
that with a wallet module whose only route to the chain is the node API, and
whose contract covers the full set of wallet operations, including several the
current module does not reach (view-only accounts, simulation, detached signing,
rescanning, and pending private messages).

Callers reach the wallet module through Logos Core: other modules in Basecamp or
`logosctl` call it through the runtime, and an application outside Basecamp
reaches the same module through the
[Logos Runtime DK](https://github.com/logos-co/ecosystem/issues/238). There is
no separate per-language wrapper of the wallet.

The wallet API is delivered for **LEZ testnet 0.4**.

This RFP is not an open call for external proposals. The work is carried out by
the LEZ maintainers and contributed to the Logos-maintained repositories of the
LEZ wallet and its Logos Core module. Once the API is implemented, its reference
documentation supersedes this document for integrators.

## 🔥 Why This Matters

An integrator adding a chain needs a wallet before it needs a single read: it
must derive accounts from a backed-up seed, generate deposit addresses, sign
offline, and know what a transaction will cost before sending it. Every
established chain answers those needs with a wallet library separate from the
node (BDK on Bitcoin, viem and ethers on Ethereum, `@solana/kit`, CosmJS,
`@mysten/sui`), and the node accepts signed bytes.

LEZ makes the wallet heavier than on those chains. A privacy-preserving
transaction is executed and proven on the client, over notes only the wallet can
decrypt, and receiving into a private account means scanning the chain with a
viewing key. Without a stable wallet API, every exchange, custodian, auditor,
and application would have to re-implement proving, note scanning, and key
handling, or embed the current wallet internals directly. A clean wallet API is
what makes LEZ private accounts usable by anyone other than the reference
wallet, and what lets users keep their keys on their own device while using
someone else's node.

## 🏗 Design Decisions

- **The wallet holds the keys; the node holds none.** Derivation, signing,
  proving, decryption, and scanning happen in the wallet module. The node serves
  chain data and accepts signed transactions (RFP-027).
- **The wallet depends on the node API contract, not on a node module.** It
  never calls a sequencer or a node internal. A wallet configured with a node in
  the same runtime, on a paired device, or in an integrator's infrastructure
  behaves identically.
- **Simulation and fee estimation belong to the wallet.** The node cannot
  execute the private part of a privacy-preserving transaction, and executing
  the public part there would show a shared node every transaction a caller
  considered. The wallet already holds the executor; it simulates both parts
  locally against state it reads from the node at the head (RFP-027, "Simulation
  belongs to the wallet, not to the node").
- **The node serves; the wallet scans.** The wallet follows its private accounts
  by reading the node's position-only walk of encrypted notes (RFP-027 viewing
  key reads) and trial-decrypting locally, as a Zcash light client does with
  lightwalletd. The wallet never hands a viewing key to a node.
- **One seed, deterministic accounts.** Every account the wallet creates is
  derived from one BIP-39 seed along documented, hardened paths, so a mnemonic
  alone restores the wallet.
- **View-only is a first-class wallet kind.** A wallet holding only viewing keys
  (an auditor, an accountant, a custodian's monitoring, a member of a shared
  private account given sight but not control) runs the same sync and the same
  reads, and refuses everything that needs a spending key.
- **Signing is gated per caller.** Other modules call the wallet through Logos
  Core. Reads are grantable to a caller; anything that spends, signs, or exports
  secrets requires either explicit user approval or a standing policy the user
  granted, in the manner of an EIP-1193 provider and of Clef.
- **Detached signing is supported.** A transaction can be built in one wallet,
  signed or proven in another, and submitted from a third, as PSBT allows on
  Bitcoin, so a key can live offline.

## ✅ Scope of Work

The key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
RECOMMENDED, NOT RECOMMENDED, MAY, and OPTIONAL in this document are to be
interpreted as described in BCP 14
([RFC 2119](https://www.rfc-editor.org/rfc/rfc2119),
[RFC 8174](https://www.rfc-editor.org/rfc/rfc8174)) when, and only when, they
appear in all capitals, as shown here.

### Hard Requirements

#### Functionality

Method names in the tables are indicative; the LEZ maintainers finalise names,
parameters, and encodings in the module contract. Every method is a distinct
method of the wallet module's Logos Core contract, so a deployer can grant it on
its own (see [Callers, approval, and policy](#callers-approval-and-policy)).

##### Wallet lifecycle and storage

| Method                            | Purpose                                              |
| --------------------------------- | ---------------------------------------------------- |
| `create_wallet`                   | Create a wallet from a new seed; return the mnemonic |
| `restore_wallet`                  | Restore from a mnemonic, with an optional birthday   |
| `create_view_only_wallet`         | Create a wallet holding viewing keys and public ids  |
| `open_wallet` / `close_wallet`    | Open or close a wallet's storage                     |
| `lock` / `unlock`                 | Drop or load decrypted key material                  |
| `change_password`                 | Re-encrypt the wallet under a new password           |
| `export_backup` / `import_backup` | Encrypted backup of wallet state beyond the seed     |
| `query_wallet_info`               | Kind (full or view-only), lock state, sync position  |
| `delete_wallet`                   | Remove a wallet's storage                            |

1. <a id="f-1"></a>The wallet MUST generate its seed from a cryptographically
   secure source and express it as a BIP-39 mnemonic of at least 24 words,
   returned once at creation and never again except through an administrative
   export ([#57](#f-57)).
2. <a id="f-2"></a>Restoring from a mnemonic MUST reproduce every account the
   original wallet derived, through account discovery ([#9](#f-9)), with no
   other input than the mnemonic and an optional passphrase and birthday block.
3. <a id="f-3"></a>Wallet storage MUST be encrypted at rest under a key derived
   from the user's password with a memory-hard KDF (Argon2id or scrypt) at
   documented parameters. A wallet MUST NOT be creatable without encryption
   except under an explicit, documented development setting.
4. <a id="f-4"></a>A locked wallet MUST hold no decrypted secret in memory, and
   MUST still answer reads that need no secret (balances and history already
   synced, account identifiers).
5. <a id="f-5"></a>The wallet MUST store a birthday (the first block its seed
   could have been used at) and start scanning there, so restoring a recent
   wallet does not walk the chain from genesis.
6. <a id="f-6"></a>A backup MUST carry what the seed cannot reproduce: labels,
   imported viewing keys and watch-only accounts, shared private account
   memberships, and transaction notes the user added, encrypted as wallet
   storage is.
7. <a id="f-7"></a>One module instance MUST be able to hold more than one
   wallet, each with its own seed, storage, and lock state, addressed by a
   wallet identifier on every call.

##### Keys and derivation

| Method                   | Purpose                                                   |
| ------------------------ | --------------------------------------------------------- |
| `derive_public_account`  | Next public account on a documented path                  |
| `derive_private_account` | Next private account and its key set                      |
| `derive_public_pda`      | Public program-derived account id for a program and seed  |
| `derive_private_pda`     | Private program-derived account id for a program and seed |
| `query_derivation_paths` | The paths and indices in use                              |

08. <a id="f-8"></a>Every key the wallet holds MUST be derived from the seed
    along a documented path. Public account signing keys and the key set of a
    private account (its spending or nullifier key and its viewing key) MUST use
    hardened derivation (SLIP-0010 style) with a LEZ-specific purpose and coin
    type, and the documentation MUST publish the scheme with test vectors so
    other wallets and hardware signers can implement it.
09. <a id="f-9"></a>On restore, the wallet MUST discover used accounts of each
    kind by deriving ahead and checking for activity through the node API, up to
    a documented gap limit that the caller MAY raise.
10. <a id="f-10"></a>The wallet MUST derive public and private PDA identifiers
    exactly as the LEZ state machine does, so a caller can compute an account
    identifier before the account exists.
11. <a id="f-11"></a>The viewing key of a private account MUST be separable from
    its spending material, so that it can be exported ([#18](#f-18)) without
    granting the ability to spend.

##### Accounts

| Method                        | Purpose                                                      |
| ----------------------------- | ------------------------------------------------------------ |
| `list_accounts`               | Every account the wallet holds or views, with kind and label |
| `query_account`               | One account: kind, identifiers, label, sync state            |
| `import_public_key`           | Import a public account key (full)                           |
| `import_watch_account`        | Watch a public account by identifier, without its key        |
| `set_label` / `resolve_label` | Local labels for accounts                                    |
| `create_shared_account`       | Create a shared private account and its member material      |
| `join_shared_account`         | Join a shared private account from member material           |

12. <a id="f-12"></a>The wallet MUST hold public accounts, private accounts,
    private PDAs, and shared private accounts, and every account record MUST
    state its kind and whether the wallet can spend from it, only view it, or
    only watch its public state.
13. <a id="f-13"></a>Public accounts MUST be reported with their CAIP-10
    identifier, using the LEZ chain identifier the node serves (RFP-027, network
    identity), alongside the base58 form LEZ tools print.
14. <a id="f-14"></a>The wallet MUST support shared private accounts: creating
    one, distributing to each member the material it is entitled to (spending or
    view-only), joining one, and syncing it. The documentation MUST state who
    can spend, who can view, and how membership material is derived and
    transferred.
15. <a id="f-15"></a>Labels are local to the wallet and MUST NOT be sent to the
    node.

##### View-only and watch-only access

| Method                    | Purpose                                                    |
| ------------------------- | ---------------------------------------------------------- |
| `export_viewing_key`      | Export one private or shared account's viewing key         |
| `import_viewing_key`      | Add a private or shared account the wallet can view        |
| `export_view_only_bundle` | Every viewing key and public id of a wallet, with birthday |

16. <a id="f-16"></a>A wallet MUST be able to view a private account, a private
    PDA, and a shared private account from its viewing key alone: sync it,
    report its balances and history, and decrypt its incoming notes and
    messages.
17. <a id="f-17"></a>A view-only account, and every account in a view-only
    wallet, MUST refuse every operation that spends, signs, proves, or derives
    spending material, with an error of its own rather than a generic failure.
18. <a id="f-18"></a>Exporting a viewing key MUST be an administrative operation
    requiring user approval ([#57](#f-57)), and the export MUST carry the
    account's birthday so the importer scans from there.
19. <a id="f-19"></a>The documentation MUST state, for audit use, what a viewing
    key holder can and cannot establish about an account: incoming and outgoing
    transfers, balances over time, counterparties as far as the chain reveals
    them, and whether it can detect spends the account made. It MUST state the
    same for a view-only member of a shared private account.
20. <a id="f-20"></a>A public account MUST be watchable by identifier alone,
    with balances and history served from the node's public reads.

##### Sync

| Method              | Purpose                                                      |
| ------------------- | ------------------------------------------------------------ |
| `sync`              | Bring every account up to the node's head                    |
| `rescan`            | Rescan from a block for some or all accounts                 |
| `query_sync_status` | Last synced block and hash, head, progress, errors           |
| `set_sync_mode`     | Continuous (following the node's subscriptions) or on demand |

21. <a id="f-21"></a>The wallet MUST follow its accounts through the node API
    alone: public accounts through account reads and account history, private
    accounts through the viewing key walk of encrypted notes, publications, and
    recovery bindings (RFP-027 private message delivery and viewing key reads).
22. <a id="f-22"></a>For each private account it holds or views, the wallet MUST
    trial-decrypt every note in the range it syncs, verify each decrypted note
    against the commitment that carries it, and maintain the account's current
    state, its unspent commitments, and their membership witnesses.
23. <a id="f-23"></a>Sync MUST be resumable from the last block and hash
    processed, and MUST detect that the node's head no longer contains that
    block, roll the wallet's state back to the common ancestor, and resync.
24. <a id="f-24"></a>The wallet MUST distinguish state derived from blocks at
    `pending` from state at `finalized`, and report both ([#28](#f-28)).
25. <a id="f-25"></a>Sync of private accounts MUST NOT name to the node any
    account identifier, commitment, or private address the wallet holds, except
    where the documentation identifies a call that must ([#36](#f-36), Privacy
    [#3](#priv-3)).
26. <a id="f-26"></a>The wallet MUST receive the private messages and recovery
    notes addressed to its private accounts, both clear and sealed, and apply
    them, so a private account can be paid while the wallet is offline.

##### Balances and history

| Method              | Purpose                                            |
| ------------------- | -------------------------------------------------- |
| `query_balances`    | Native and token balances per account and in total |
| `query_history`     | An account's transactions, paginated               |
| `query_transaction` | One transaction as the wallet sees it              |
| `query_notes`       | A private account's unspent notes                  |

27. <a id="f-27"></a>Balances MUST be reported per account and per token (native
    and builtin token programs), for public and private accounts alike, with the
    amount as an exact integer.
28. <a id="f-28"></a>Each balance MUST be split into finalised, pending
    (inscribed but not final), and outgoing (submitted by this wallet and not
    yet inscribed), so an application never shows unconfirmed value as spent or
    received without saying so.
29. <a id="f-29"></a>History MUST include private transactions the wallet can
    decrypt, with direction, counterparty as far as known, amount, fee, status,
    and block, and MUST be paginated by a stable cursor.
30. <a id="f-30"></a>The wallet MUST attribute the fee of each transaction it
    sent separately from the amounts transferred, matching the node's fee report
    for public legs.

##### Transaction construction

| Method                     | Purpose                                               |
| -------------------------- | ----------------------------------------------------- |
| `build_transfer`           | Public, shield, deshield, or private transfer         |
| `build_public_call`        | Call a program from public accounts                   |
| `build_private_call`       | Call a program with private inputs                    |
| `build_program_deployment` | Deploy or update a program through the program loader |
| `query_fee_options`        | Fee payment options for a draft transaction           |

31. <a id="f-31"></a>The wallet MUST build every transaction kind LEZ testnet
    0.4 accepts from a user: public transfers, shielding (public to private),
    deshielding (private to public), private transfers, generic public program
    calls, generic privacy-preserving program calls, and program deployment and
    update.
32. <a id="f-32"></a>Building MUST produce a draft transaction that is not yet
    signed or proven, which the caller can inspect, simulate ([#38](#f-38)),
    export ([#44](#f-44)), sign, prove, or discard.
33. <a id="f-33"></a>The wallet MUST manage nonces for the public accounts it
    signs for, including transactions it has submitted that are not yet
    inscribed, and MUST NOT reuse a nonce across concurrent drafts.
34. <a id="f-34"></a>The wallet MUST set the fee terms of each transaction: for
    a public transaction the payer, gas limit, tip, and fee cap; for a
    privacy-preserving transaction either a fee reserve committed in its proof,
    with tip and refund destination, or a fee declaration signed by a public
    signer. The caller MAY override each; otherwise the wallet chooses them from
    simulation ([#38](#f-38)) and the node's fee state.
35. <a id="f-35"></a>The wallet MUST select the private notes a
    privacy-preserving transaction spends and the outputs it creates, and MUST
    document its selection policy.
36. <a id="f-36"></a>Before proving, the wallet MUST obtain the membership
    proofs its spends need, from its own witnesses where it holds them and from
    the node otherwise, and MUST check the node's mempool for a transaction
    spending the same nullifiers or nonces.
37. <a id="f-37"></a>The wallet MUST set validity windows on the transactions it
    builds, with a documented default the caller MAY override.

##### Simulation and fee estimation

| Method         | Purpose                                                         |
| -------------- | --------------------------------------------------------------- |
| `simulate`     | Execute a draft locally; return outcome, gas, and state changes |
| `estimate_fee` | Estimated total fee and its parts for a draft                   |

38. <a id="f-38"></a>The wallet MUST simulate a draft transaction locally,
    executing its public part and its private part against account and program
    state read from the node at the head, and return whether it applies, the gas
    it consumes in each dimension (execution and storage), the state changes of
    its public part, and the private state changes of the accounts the wallet
    holds.
39. <a id="f-39"></a>`estimate_fee` MUST return the estimated fee broken into
    base fee per dimension and tip, priced against the node's current fee state,
    with the bounds the fee could move to before inclusion.
40. <a id="f-40"></a>The wallet MUST refuse to sign or prove a transaction whose
    simulation fails, unless the caller passes an explicit override, and MUST
    return the failure reason from simulation.
41. <a id="f-41"></a>Simulation MUST NOT send the draft or any part of it to the
    node; only the state reads it needs reach the node.

##### Proving and signing

| Method                            | Purpose                                             |
| --------------------------------- | --------------------------------------------------- |
| `prove`                           | Generate the proof of a privacy-preserving draft    |
| `cancel_proving`                  | Cancel a proof in progress                          |
| `sign_transaction`                | Sign a draft with the keys of the accounts it needs |
| `sign_message` / `verify_message` | Domain-separated off-chain message signatures       |
| `export_draft` / `import_draft`   | Move a draft between wallets for detached signing   |

42. <a id="f-42"></a>The wallet MUST generate the proof of a privacy-preserving
    transaction locally, report progress, and allow it to be cancelled.
43. <a id="f-43"></a>Before proving, the wallet MUST compare the circuit
    identifier it proves with against the one the node verifies (RFP-027 node
    version) and refuse on a mismatch.
44. <a id="f-44"></a>The wallet MUST support detached signing: export a draft,
    with everything another wallet needs to sign or prove it, as a documented,
    versioned, self-describing encoding; import such a draft; add signatures or
    a proof to it; and submit it from any wallet. A partially signed draft MUST
    carry which signatures remain. This MUST work with the signing wallet
    offline.
45. <a id="f-45"></a>`sign_message` MUST sign with a domain separation tag that
    a transaction signature can never share, and include the LEZ chain
    identifier, so a signed message cannot be replayed as a transaction or on
    another chain.
46. <a id="f-46"></a>The wallet MUST expose an external signer interface through
    which a signature for a public account key can be produced outside the
    module (a hardware signer, an HSM, or another module), so that adding such a
    signer requires no change to the API.

##### Submission and tracking

| Method                     | Purpose                                                |
| -------------------------- | ------------------------------------------------------ |
| `submit`                   | Submit a signed or proven transaction through the node |
| `send`                     | Build, simulate, prove or sign, and submit in one call |
| `query_transaction_status` | Status of a transaction this wallet submitted          |
| `rebroadcast`              | Resubmit a transaction the node no longer knows        |

47. <a id="f-47"></a>The wallet MUST submit only through the LEZ Node API, and
    MUST track each transaction it submitted through the node's statuses
    (received, pending, finalised, not known), reflecting them in balances
    ([#28](#f-28)) and history.
48. <a id="f-48"></a>Submission MUST be idempotent: submitting the same signed
    transaction twice MUST NOT spend twice or report a failure for the second
    copy.
49. <a id="f-49"></a>A transaction that returns to received or becomes not known
    after a head replacement MUST be rebroadcast while it remains valid, and a
    transaction that can no longer be included MUST be reported as dropped, its
    effects removed from pending balances.
50. <a id="f-50"></a>`send` MUST be equivalent to the separate steps, applying
    the same simulation check, approval ([#55](#f-55)), and fee rules.

##### Notifications

| Method                       | Purpose                                                |
| ---------------------------- | ------------------------------------------------------ |
| `subscribe_to_wallet_events` | Balance changes, incoming funds, status, sync progress |
| `unsubscribe`                | Stop a subscription                                    |

51. <a id="f-51"></a>The wallet MUST notify a subscriber of incoming funds,
    balance changes, transaction status changes, received private messages, sync
    progress, and head replacement, each with the account and block it concerns.
52. <a id="f-52"></a>A subscription MUST deliver only events for the wallets and
    accounts the subscribing caller has been granted.

##### Callers, approval, and policy

| Method                           | Purpose                                        |
| -------------------------------- | ---------------------------------------------- |
| `list_policies`                  | Standing signing policies the user has granted |
| `grant_policy` / `revoke_policy` | Manage standing signing policies               |
| `export_seed`                    | Export the mnemonic (administrative)           |

53. <a id="f-53"></a>Every function MUST belong to exactly one category:
    - **Read**: wallet info, accounts, balances, history, sync status,
      simulation, fee estimation, notifications;
    - **Spend**: building with intent to send, signing, proving, submitting,
      importing a draft to sign;
    - **Administrative**: create, restore, delete, change password, export seed,
      export viewing keys, export backup, manage policies.
54. <a id="f-54"></a>A deployer MUST be able to grant a caller each category, or
    any single function, through Logos Core grants. Spend and administrative
    functions MUST be closed to a caller unless granted.
55. <a id="f-55"></a>A granted spend call MUST additionally require either the
    user's approval of that call, presented with what the transaction does
    (recipient, amounts, fee, accounts touched, program called), or a standing
    policy that covers it.
56. <a id="f-56"></a>A standing policy MUST be scoped by caller, account,
    program, recipient, amount, fee cap, and expiry, so that a module can, for
    example, pay up to a limit to one program without prompting, and MUST be
    revocable.
57. <a id="f-57"></a>`export_seed` and viewing key export MUST require user
    approval every time, with no standing policy able to cover them.
58. <a id="f-58"></a>Calls from different callers MUST be served concurrently,
    and drafts, approvals, and subscriptions belonging to one caller MUST NOT be
    visible to another.

##### Surface-wide obligations

59. <a id="f-59"></a>Errors MUST be machine-readable and stable, and MUST
    distinguish at minimum: wallet locked, wrong password, view-only, not
    authorised, approval denied, approval timed out, insufficient funds,
    simulation failed, nonce conflict, mempool conflict, circuit mismatch, node
    unreachable, node behind, and invalid argument.
60. <a id="f-60"></a>Amounts and nonces (128-bit) MUST be carried exactly, never
    as floating-point numbers.
61. <a id="f-61"></a>The wallet MUST report which node it is using (in the same
    runtime, a paired runtime, or remote) and that node's chain identifier, and
    MUST refuse to operate against a node whose chain identifier differs from
    the wallet's network.

#### Usability

1. The delivery MUST include the LEZ wallet module and its Logos Core contract,
   covering every function in Functionality, usable to build Logos modules that
   hold, receive, and send LEZ.
2. Every function MUST be callable through `logosctl`, with a working,
   documented invocation and example response, and every documented invocation
   MUST run as a doctest in CI.
3. The delivery MUST include a worked example, as a script driving `logosctl` or
   a reference module, that creates a wallet, receives to a public and a private
   account, shields, sends privately, deshields to a public account, and reports
   each step's status and fee.
4. The delivery MUST include a worked view-only example: export a viewing key
   from one wallet, import it into a view-only wallet, and show the same
   balances and history there, including for a shared private account.
5. The delivery MUST include a worked detached signing example with the signing
   wallet offline.
6. The documentation MUST describe the approval flow a user sees, and how an
   application presents an approval request in Basecamp and when hosting Logos
   Core itself.
7. Error messages MUST be clear and actionable, carrying the machine-readable
   value of Functionality [#59](#f-59) alongside the text.
8. The documentation MUST publish the derivation scheme with test vectors, the
   detached draft encoding with examples, and the message signing format.

#### Reliability

1. Wallet storage MUST be crash-safe: a crash at any point leaves either the
   state before or the state after an operation, never a partial one, and never
   loses the record of a submitted transaction.
2. A wallet restored from its mnemonic on a fresh device MUST converge to the
   same accounts, balances, and history as the original, for every account kind
   derived from the seed.
3. A head replacement MUST leave balances, history, and witnesses consistent
   with the node's new head once sync completes.
4. A node becoming unreachable, slow, or behind MUST be reported, and MUST NOT
   cause the wallet to report stale balances as current.
5. The wallet MUST work against a node in the same runtime, in a paired runtime,
   and in another party's infrastructure, with no change other than which node
   it is pointed at.
6. Concurrent drafts from different callers MUST NOT produce conflicting
   transactions (shared nonces or shared notes).

#### Performance

1. The documentation MUST state the measured proving time and peak memory of
   each privacy-preserving transaction kind, on stated desktop and mobile-class
   hardware.
2. The documentation MUST state sync throughput: blocks and encrypted notes per
   second for a wallet with a stated number of private accounts, and the time to
   restore a wallet with a stated history from its birthday.
3. The documentation MUST state the latency of simulation and fee estimation for
   each transaction kind.
4. The documentation MUST state the wallet's storage growth per account and per
   transaction.
5. Benchmarks MUST be reproducible from the test suite against a LEZ devnet or
   testnet node.

#### Supportability

01. The module MUST be delivered for LEZ testnet 0.4: built and tested against a
    LEZ devnet and LEZ testnet 0.4.
02. End-to-end integration tests MUST run in CI against a local stack of a Logos
    Blockchain node and LEZ nodes, with the wallet module and the LEZ node
    module loaded in a Logos Core runtime, and the wallet calling the node
    through its contract.
03. CI MUST build and test the module and MUST be green on the default branch of
    every repository the work lands in.
04. Every hard requirement in Functionality, Usability, Reliability, and
    Performance MUST have at least one corresponding test.
05. Tests MUST cover, at minimum: restore from mnemonic with discovery across a
    gap; every transaction kind built, simulated, proven or signed, submitted,
    and finalised; a transaction whose simulation fails; a view-only wallet
    refusing to sign; viewing a shared private account as a view-only member;
    detached signing across two wallets; a head replacement during sync and
    after submission; a node that is behind; concurrent callers sending from one
    account; an approval denied and a policy that covers and one that does not
    cover a call; and a circuit mismatch.
06. A README MUST document end-to-end usage through `logosctl`, from installing
    and loading the module to sending a private transfer.
07. The delivery MUST include integrator documentation in repo covering the
    journeys of a wallet application, an exchange tracking deposits and sending
    withdrawals, and an auditor with viewing keys.
08. The delivery MUST include a get started guide that takes a developer from an
    empty repository to a Logos Core module that calls the wallet module through
    its generated client.
09. The delivery MUST include a full API reference, generated from the module
    contract and published per version.
10. The delivery MUST include a design decision log in repo recording each major
    design decision and choice: the context, the options considered, the option
    chosen, and the rationale and trade-offs. The log MUST be updated as
    decisions are made and MUST be current at each milestone delivery.
11. The delivery MUST include a migration guide from the `lez_core` module and
    `wallet_ffi`, mapping each existing function to its replacement or stating
    that it is dropped.

#### + Privacy and Security

1. <a id="priv-1"></a>Secrets (seed, spending and nullifier keys, viewing keys,
   decrypted notes) MUST NOT leave the module except through the administrative
   exports of Functionality [#57](#f-57) and [#18](#f-18), and MUST NOT appear
   in logs, errors, or notifications.
2. <a id="priv-2"></a>Decrypted secrets MUST be zeroised when the wallet locks
   and when they are no longer needed.
3. <a id="priv-3"></a>The documentation MUST identify every wallet operation
   that causes a call to the node naming an account identifier, commitment,
   nullifier, or private address, and MUST prefer position-only reads wherever
   the node API offers them.
4. <a id="priv-4"></a>The delivery MUST include a privacy properties document
   covering: what a node operator learns from the wallet's calls, including when
   the node is in another party's infrastructure; what timing and pattern of
   sync and submission reveal; what a viewing key holder learns; what a member
   of a shared private account learns about the other members; and how a user
   discloses less (own node, position-only reads, batching).
5. <a id="priv-5"></a>The wallet MUST NOT send labels, notes the user attaches,
   or any other local metadata to the node.
6. <a id="priv-6"></a>The delivery MUST include a threat model for the module
   covering a malicious caller module, a malicious node, a stolen device with a
   locked wallet, and a compromised host process.

### Soft Requirements

1. The wallet SHOULD support Keycard as the first external signer through the
   interface of Functionality [#46](#f-46).
2. The wallet SHOULD support restoring the LEZ wallet and the Logos Blockchain
   wallet ([RFP-029](./RFP-029-blockchain-wallet-api.md)) from one mnemonic,
   with chain-separated derivation paths agreed between the two.
3. The wallet SHOULD keep its membership witnesses current from the node's
   commitment-set frontier rather than requesting proofs that name its
   commitments (RFP-027 soft requirement 5).
4. The wallet SHOULD offer batching: several transfers in one transaction where
   the protocol allows it.
5. The wallet SHOULD offer a `wallet_` namespace of request methods modelled on
   EIP-1193 and CAIP-25, so web-style applications can integrate through a
   familiar provider shape.

### Out of Scope

- **Node functions**: following the chain, serving state, and relaying belong to
  the LEZ Node API (RFP-027).
- **A separate keystore module.** Keys stay in the wallet module. A keystore
  module that holds keys apart from the wallet, as the Logos EVM wallet already
  has, is planned but not part of this RFP.
- **A GUI wallet application.** This RFP delivers the module and its contract.
- **Multisig coordination**, which belongs to
  [RFP-005](./RFP-005-multisig-and-coordination.md).
- **Bridging** to other chains.
- **The Logos Blockchain wallet**, specified in
  [RFP-029](./RFP-029-blockchain-wallet-api.md).
- **What Logos Core provides**: hosting, pairing, remote transport, and caller
  authentication.

## ⚠ Platform Dependencies

- **LEZ Node API (RFP-027)**: the wallet reaches the chain only through it. The
  viewing key walk, membership proofs, publications, recovery bindings, the
  head, transaction statuses, and the fee state it serves are all needed at run
  time. Where a node function is not yet available, the wallet's tests run
  against what is, and the gap is recorded in the design decision log.
- **Logos Core module contracts and per-method grants**: needed for the
  categories and grants of Functionality [#53](#f-53) to [#58](#f-58).
- **An approval surface**: Logos Core or Basecamp needs a way for a module to
  raise an approval request to the user and receive the answer, including when
  the caller is headless.
- **Peering**: for a wallet using a node on another device.
- **Logos Runtime DK**
  ([ecosystem#238](https://github.com/logos-co/ecosystem/issues/238)): for
  applications outside Basecamp.

## Open Questions

1. The LEZ derivation scheme: purpose, coin type, and path layout for public
   accounts, private accounts, and shared private accounts.
2. Whether, and how, one mnemonic backs both the LEZ and the Logos Blockchain
   wallets.
3. How shared private account membership material is derived and transferred,
   and what a view-only member sees.
4. How an approval request reaches a user when the calling module is headless or
   the wallet runs on another device from the application.
5. Whether the detached draft encoding should be shared with the Logos
   Blockchain wallet.

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [LEZ Node API (RFP-027)](./RFP-027-lez-node-api.md)
- [Logos Blockchain Wallet API (RFP-029)](./RFP-029-blockchain-wallet-api.md)
- [Integrate Logos into your application](https://docs.logos.co/build-an-app/integrate-logos)
- [LEZ documentation](https://docs.logos.co/lez)
- [Logos glossary](https://docs.logos.co/get-started/glossary)
- [logos-execution-zone](https://github.com/logos-blockchain/logos-execution-zone):
  the LEZ node, sequencer, and wallet
- [logos-execution-zone-module](https://github.com/logos-blockchain/logos-execution-zone-module):
  the `lez_core` Logos Core module
- [BIP-39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki),
  [SLIP-0010](https://github.com/satoshilabs/slips/blob/master/slip-0010.md),
  [BIP-174 (PSBT)](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki),
  [EIP-1193](https://eips.ethereum.org/EIPS/eip-1193)

The Resources listed above, and any third-party material referred to elsewhere
in this specification, are for informational purposes only and are intended
solely as guidance regarding tools or other artefacts that may be relevant to
preparing a Submission under the Program. They may include, or may refer to,
earlier published specifications, some of which may have received Submissions.
Logos does not endorse or approve, and makes no representation regarding, any
Submission or referenced material. It is ultimately the Applicant’s
responsibility to determine whether, and how, if at all, to use any Resource.

## 💬 Feedback

This RFP is not open for proposals. Integrators with questions about the API it
describes, or with needs it does not meet, are invited to raise them in the
repository's **Discussions**.
