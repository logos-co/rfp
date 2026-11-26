---
id: RFP-029
title: Logos Blockchain Wallet API
tier: M
status: internal
category: Developer Tooling & Infrastructure
dependencies: []
---

# RFP-029 — Logos Blockchain Wallet API

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

This RFP specifies the Logos Blockchain wallet API: the API of the wallet module
for the Logos Blockchain, the L1 on which LEZ and other zones inscribe their
blocks. The wallet module holds keys and does everything that needs them: it
derives keys and addresses, tracks the notes its keys own, builds transactions,
generates the zero-knowledge proofs that value transfers carry, signs the
operations that are signed, and hands the result to a Logos Blockchain node for
the network.

It is the wallet-side counterpart to the
[Blockchain Node API](https://github.com/logos-co/ecosystem/issues/219), and the
Logos Blockchain counterpart to the
[LEZ Wallet API (RFP-028)](./RFP-028-lez-wallet-api.md). The split follows the
target architecture in
[Integrate Logos into your application](https://docs.logos.co/build-an-app/integrate-logos):
each Logos chain is served by a wallet module, which runs where the keys are,
and a node module, which follows the chain and holds no keys. The wallet reaches
the chain **only through the Blockchain Node API**, so the node can run in the
same application, on another device the user owns, or in an integrator's
infrastructure without any change to the wallet.

Proving is in scope because a Logos Blockchain value transfer carries a
zero-knowledge proof of note ownership in place of a signature, and that proof
is generated on the client. The wallet therefore runs where the keys are, which
is normally the user's application, or the operator's machine for a node
operator or a zone sequencer.

Today the Logos Blockchain node and wallet are combined in a single
`blockchain_module`, over C bindings that start a full node and expose key
management, balances, transfers, channel deposits, staking declarations, and
reward claims next to chain reads. This RFP separates the wallet half into its
own module, whose only route to the chain is the Blockchain Node API.

Callers reach the wallet module through Logos Core: other modules in Basecamp or
`logosctl` call it through the runtime, and an application outside Basecamp
reaches the same module through the
[Logos Runtime DK](https://github.com/logos-co/ecosystem/issues/238). There is
no separate per-language wrapper of the wallet.

The wallet API is delivered against the Logos Blockchain network that LEZ
testnet 0.4 runs on.

This RFP is not an open call for external proposals. The work is carried out by
the Logos Blockchain maintainers and contributed to the Logos-maintained
repositories of the Logos Blockchain wallet and its Logos Core module. Once the
API is implemented, its reference documentation supersedes this document for
integrators.

## 🔥 Why This Matters

Every participant in the Logos stack touches the L1 wallet. Users hold and send
the native token; exchanges and custodians need deposit addresses, sweeps, and
offline signing; node operators declare and withdraw stake and claim rewards;
and every zone sequencer, LEZ included, needs an L1 wallet to inscribe its
blocks and pay for doing so. Today all of them reach the wallet through a module
that also runs a full node, so a phone wallet, an exchange's signer, and a
sequencer each have to embed a node to hold a key.

Separating the wallet makes each of those deployments possible on its own terms:
keys on the user's device with a node elsewhere, an air-gapped signer for a
custodian, and a sequencer that signs inscriptions unattended under a policy its
operator set. It also gives integrators the shape they already know from other
chains: a wallet library that derives, signs, and proves, and a node that
accepts the result.

## 🏗 Design Decisions

- **The wallet holds the keys; the node holds none.** Key generation,
  derivation, proving, and signing happen in the wallet module. The Blockchain
  Node API follows the chain, serves state, and relays signed transactions.
- **The wallet depends on the node API contract, not on a node module.** A
  wallet configured with a node in the same runtime, on a paired device, or in
  an integrator's infrastructure behaves identically.
- **Proven value, signed operations.** Spending a note is proven; other Mantle
  operations (channel inscriptions, staking declarations, and the like) are
  signed with the key that controls them. The wallet hides that difference
  behind one build, approve, authorise, and submit flow, and reports which keys
  a transaction needs.
- **Non-human callers are first class.** A LEZ node in sequencer mode uses this
  wallet to inscribe blocks to its channel; a node operator's tooling uses it to
  declare stake and claim rewards. Both run unattended, so the wallet supports
  signing under a standing policy as well as signing approved by a person.
- **One seed, deterministic keys.** Every key the wallet creates is derived from
  one BIP-39 seed along documented paths, so a mnemonic alone restores it.
- **Watch-only is a first-class wallet kind.** A wallet holding only public
  material tracks notes and balances and refuses everything that spends.
- **Detached signing is supported.** A transaction can be built in one wallet,
  proven or signed in another, and submitted from a third, as PSBT allows on
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

Method names in the tables are indicative; the Logos Blockchain maintainers
finalise names, parameters, and encodings in the module contract. Every method
is a distinct method of the wallet module's Logos Core contract, so a deployer
can grant it on its own (see
[Callers, approval, and policy](#callers-approval-and-policy)).

##### Wallet lifecycle and storage

| Method                            | Purpose                                              |
| --------------------------------- | ---------------------------------------------------- |
| `create_wallet`                   | Create a wallet from a new seed; return the mnemonic |
| `restore_wallet`                  | Restore from a mnemonic, with an optional birthday   |
| `create_watch_only_wallet`        | Create a wallet holding public material only         |
| `open_wallet` / `close_wallet`    | Open or close a wallet's storage                     |
| `lock` / `unlock`                 | Drop or load decrypted key material                  |
| `change_password`                 | Re-encrypt the wallet under a new password           |
| `export_backup` / `import_backup` | Encrypted backup of wallet state beyond the seed     |
| `query_wallet_info`               | Kind (full or watch-only), lock state, sync position |
| `delete_wallet`                   | Remove a wallet's storage                            |

1. <a id="f-1"></a>The wallet MUST generate its seed from a cryptographically
   secure source and express it as a BIP-39 mnemonic of at least 24 words,
   returned once at creation and never again except through an administrative
   export ([#43](#f-43)).
2. <a id="f-2"></a>Restoring from a mnemonic MUST reproduce every key the
   original wallet derived, through discovery ([#9](#f-9)), with no other input
   than the mnemonic and an optional passphrase and birthday block.
3. <a id="f-3"></a>Wallet storage MUST be encrypted at rest under a key derived
   from the user's password with a memory-hard KDF (Argon2id or scrypt) at
   documented parameters. A wallet MUST NOT be creatable without encryption
   except under an explicit, documented development setting.
4. <a id="f-4"></a>A locked wallet MUST hold no decrypted secret in memory, and
   MUST still answer reads that need no secret.
5. <a id="f-5"></a>The wallet MUST store a birthday and start scanning there.
6. <a id="f-6"></a>A backup MUST carry what the seed cannot reproduce: imported
   keys, watch-only entries, labels, and standing policies, encrypted as wallet
   storage is.
7. <a id="f-7"></a>One module instance MUST be able to hold more than one
   wallet, each with its own seed, storage, and lock state, addressed by a
   wallet identifier on every call.

##### Keys, addresses, and derivation

| Method                      | Purpose                                                          |
| --------------------------- | ---------------------------------------------------------------- |
| `derive_address`            | Next receive address (a note-ownership key) on a documented path |
| `derive_signing_key`        | Next signing key for a given role (channel, staking, other)      |
| `import_key` / `remove_key` | Import or remove a key not derived from the seed                 |
| `list_keys`                 | Every key and address the wallet holds, with role and label      |
| `query_derivation_paths`    | The paths and indices in use                                     |

08. <a id="f-8"></a>Every key the wallet derives MUST be derived from the seed
    along a documented path, separated by role: note-ownership keys that receive
    and spend value, and signing keys for channels, staking declarations, and
    other signed operations. The documentation MUST publish the scheme with test
    vectors so that other wallets and hardware signers can implement it.
09. <a id="f-9"></a>On restore, the wallet MUST discover used keys of each role
    by deriving ahead and checking for activity through the node API, up to a
    documented gap limit that the caller MAY raise.
10. <a id="f-10"></a>Addresses MUST have one documented text encoding with a
    checksum and a network prefix, so an address for one network cannot be
    mistaken for another, and the wallet MUST reject a malformed or
    wrong-network address with an error of its own.
11. <a id="f-11"></a>The wallet MUST support importing a key not derived from
    the seed (for example, a key generated by today's `blockchain_module` or by
    an operator), and MUST flag such keys as outside the mnemonic backup.
12. <a id="f-12"></a>Each key MUST carry its role, so that the wallet refuses to
    use, for example, a channel key to sign a staking declaration.

##### Watch-only and deposit addresses

| Method                   | Purpose                                                  |
| ------------------------ | -------------------------------------------------------- |
| `export_watch_material`  | Public material for one key, or the whole wallet         |
| `import_watch_material`  | Add keys the wallet can watch but not spend from         |
| `derive_deposit_address` | A fresh deposit address, from watch material if possible |

13. <a id="f-13"></a>A wallet MUST be able to watch an address from its public
    material alone: detect the notes it owns, report its balance and history,
    and refuse every operation that spends or signs, with an error of its own.
14. <a id="f-14"></a>The documentation MUST state whether new deposit addresses
    can be derived from public material alone (as BIP-32 public derivation
    allows on secp256k1), or only with the secret key (as with hardened ed25519
    derivation). If they cannot, the documentation MUST describe the deposit
    model an exchange uses instead, and the wallet MUST support it: for example,
    a pool of addresses pre-derived by the full wallet and exported as watch
    material ([#15](#f-15)).
15. <a id="f-15"></a>A full wallet MUST be able to export watch material for a
    range of future addresses, so a watch-only deployment can hand out deposit
    addresses while the full wallet stays offline.

##### Notes, balances, and sync

| Method              | Purpose                                            |
| ------------------- | -------------------------------------------------- |
| `sync`              | Bring every key up to the node's head              |
| `rescan`            | Rescan from a block for some or all keys           |
| `query_sync_status` | Last synced block and hash, head, progress, errors |
| `query_balances`    | Balance per address and in total                   |
| `query_notes`       | The unspent notes the wallet owns                  |
| `query_history`     | Transactions affecting the wallet, paginated       |

16. <a id="f-16"></a>The wallet MUST follow its keys through the Blockchain Node
    API alone, track every note its keys own, and mark notes spent when a
    transaction spending them is included.
17. <a id="f-17"></a>Sync MUST be resumable from the last block and hash
    processed, and MUST detect that the node's chain no longer contains that
    block, roll the wallet's state back to the common ancestor, and resync.
18. <a id="f-18"></a>Balances MUST be reported per address and in total, as
    exact integers, split into final, not yet final, and outgoing (submitted by
    this wallet and not yet included), using the node's own notion of finality.
19. <a id="f-19"></a>History MUST include every transaction that creates or
    spends a note the wallet owns, or that a key of the wallet signed, with
    direction, amount, fee, status, and block, paginated by a stable cursor.
20. <a id="f-20"></a>The wallet MUST report the fee of each transaction it sent
    separately from the amounts transferred.

##### Transaction construction

| Method                         | Purpose                                                      |
| ------------------------------ | ------------------------------------------------------------ |
| `build_transfer`               | Send to one or more addresses                                |
| `build_sweep`                  | Move every note from some keys to one address                |
| `build_channel_deposit`        | Deposit value into a channel                                 |
| `build_channel_inscription`    | Inscribe data to a channel the wallet's key controls         |
| `build_staking_declaration`    | Declare, update activity, or withdraw a service declaration  |
| `build_reward_claim`           | Claim leader or proof-of-work rewards                        |
| `build_blend_join`             | Join Blend as a core node                                    |
| `query_required_authorisation` | The proofs and signatures a draft needs, and from which keys |

21. <a id="f-21"></a>The wallet MUST build every user and operator transaction
    the Logos Blockchain accepts and the current `blockchain_module` exposes:
    transfers, channel deposits, channel inscriptions, service declarations and
    their activity and withdrawal, leader and proof-of-work reward claims, and
    joining Blend.
22. <a id="f-22"></a>Building MUST produce a draft that is not yet proven or
    signed, which the caller can inspect, validate ([#25](#f-25)), export
    ([#30](#f-30)), prove, sign, or discard.
23. <a id="f-23"></a>For a transfer, the wallet MUST select the notes to spend
    and create change to an address of the wallet, and MUST document its
    selection policy. The caller MAY choose the notes and the change address.
24. <a id="f-24"></a>The wallet MUST set each draft's fee from the gas prices
    the node serves, and MUST let the caller set a fee cap the draft cannot
    exceed.

##### Validation and fee estimation

| Method         | Purpose                                                         |
| -------------- | --------------------------------------------------------------- |
| `validate`     | Check a draft against the node's current state, without sending |
| `estimate_fee` | Estimated fee for a draft, and the range it may move in         |

25. <a id="f-25"></a>The wallet MUST validate a draft locally before it is
    proven or signed: that the notes it spends are unspent at the node's head,
    that the keys it needs are present, that its fee meets the node's gas
    prices, and that it is well formed. A draft that fails MUST NOT be proven or
    signed unless the caller passes an explicit override.
26. <a id="f-26"></a>`estimate_fee` MUST return the estimated fee and its parts,
    priced against the node's current gas prices, with the range the fee could
    move to before inclusion.
27. <a id="f-27"></a>Validation MUST NOT send the draft to the node; only the
    state reads it needs reach the node.

##### Proving and signing

| Method                            | Purpose                                           |
| --------------------------------- | ------------------------------------------------- |
| `prove`                           | Generate the note-ownership proofs a draft needs  |
| `cancel_proving`                  | Cancel a proof in progress                        |
| `sign_transaction`                | Sign the operations a draft needs signed          |
| `sign_message` / `verify_message` | Domain-separated off-chain message signatures     |
| `export_draft` / `import_draft`   | Move a draft between wallets for detached signing |

28. <a id="f-28"></a>The wallet MUST generate the proofs a draft needs locally,
    report progress, and allow proving to be cancelled.
29. <a id="f-29"></a>The wallet MUST sign the operations a draft needs signed,
    each with the key whose role covers it ([#12](#f-12)).
30. <a id="f-30"></a>The wallet MUST support detached authorisation: export a
    draft, with everything another wallet needs to prove or sign it, as a
    documented, versioned, self-describing encoding; import such a draft; add
    proofs or signatures to it; and submit it from any wallet. A partially
    authorised draft MUST carry which proofs and signatures remain. This MUST
    work with the authorising wallet offline.
31. <a id="f-31"></a>`sign_message` MUST sign with a domain separation tag that
    a transaction signature can never share, and include the network identifier,
    so a signed message cannot be replayed as a transaction or on another
    network.
32. <a id="f-32"></a>The wallet MUST expose an external signer interface through
    which a signature can be produced outside the module (a hardware signer, an
    HSM, or another module), so that adding such a signer requires no change to
    the API.

##### Submission and tracking

| Method                     | Purpose                                                 |
| -------------------------- | ------------------------------------------------------- |
| `submit`                   | Submit an authorised transaction through the node       |
| `send`                     | Build, validate, prove and sign, and submit in one call |
| `query_transaction_status` | Status of a transaction this wallet submitted           |
| `rebroadcast`              | Resubmit a transaction the node no longer holds         |

33. <a id="f-33"></a>The wallet MUST submit only through the Blockchain Node
    API, and MUST track each transaction it submitted through the node's
    statuses until it is final, reflecting them in balances ([#18](#f-18)) and
    history.
34. <a id="f-34"></a>Submission MUST be idempotent: submitting the same
    transaction twice MUST NOT spend twice or report a failure for the second
    copy.
35. <a id="f-35"></a>A transaction dropped from the node's mempool or undone by
    a reorganisation MUST be rebroadcast while it remains valid, and one that
    can no longer be included MUST be reported as dropped, its notes returned to
    the spendable balance.
36. <a id="f-36"></a>`send` MUST be equivalent to the separate steps, applying
    the same validation, approval ([#40](#f-40)), and fee rules.

##### Notifications

| Method                       | Purpose                                                |
| ---------------------------- | ------------------------------------------------------ |
| `subscribe_to_wallet_events` | Incoming notes, balance changes, status, sync progress |
| `unsubscribe`                | Stop a subscription                                    |

37. <a id="f-37"></a>The wallet MUST notify a subscriber of incoming notes,
    balance changes, transaction status changes, sync progress, and
    reorganisations, each with the key and block it concerns.
38. <a id="f-38"></a>A subscription MUST deliver only events for the wallets and
    keys the subscribing caller has been granted.

##### Callers, approval, and policy

| Method                           | Purpose                                |
| -------------------------------- | -------------------------------------- |
| `list_policies`                  | Standing signing policies in force     |
| `grant_policy` / `revoke_policy` | Manage standing signing policies       |
| `export_seed`                    | Export the mnemonic (administrative)   |
| `export_key`                     | Export one secret key (administrative) |

39. <a id="f-39"></a>Every function MUST belong to exactly one category:
    - **Read**: wallet info, keys and addresses, balances, notes, history, sync
      status, validation, fee estimation, notifications;
    - **Spend**: building with intent to send, proving, signing, submitting,
      importing a draft to authorise;
    - **Administrative**: create, restore, delete, change password, import and
      remove keys, export the seed or a key, export backup, manage policies.
40. <a id="f-40"></a>A deployer MUST be able to grant a caller each category, or
    any single function, through Logos Core grants. Spend and administrative
    functions MUST be closed to a caller unless granted. A granted spend call
    MUST additionally require either the user's approval of that call, presented
    with what the transaction does (recipients, amounts, fee, operation and its
    target), or a standing policy that covers it.
41. <a id="f-41"></a>A standing policy MUST be scoped by caller, key, operation
    kind, target (such as a channel identifier), amount, fee cap, rate, and
    expiry, and MUST be revocable. It MUST be possible to express, at minimum,
    "this module may inscribe to channel X, paying at most F per inscription and
    at most T per day", which is what a sequencer needs.
42. <a id="f-42"></a>Every use of a standing policy MUST be recorded in a local
    audit log the administrative category can read, with the caller, the policy,
    the transaction hash, and the fee paid.
43. <a id="f-43"></a>Exporting the seed or a secret key MUST require user
    approval every time, with no standing policy able to cover it.
44. <a id="f-44"></a>Calls from different callers MUST be served concurrently,
    and drafts, approvals, and subscriptions belonging to one caller MUST NOT be
    visible to another.

##### Surface-wide obligations

45. <a id="f-45"></a>Errors MUST be machine-readable and stable, and MUST
    distinguish at minimum: wallet locked, wrong password, watch-only, wrong key
    role, not authorised, approval denied, approval timed out, policy exceeded,
    insufficient funds, validation failed, note already spent, wrong network,
    node unreachable, node behind, and invalid argument.
46. <a id="f-46"></a>Amounts MUST be carried exactly, never as floating-point
    numbers.
47. <a id="f-47"></a>The wallet MUST report which node it is using and that
    node's network, and MUST refuse to operate against a node on a different
    network from the wallet's.

#### Usability

1. The delivery MUST include the Logos Blockchain wallet module and its Logos
   Core contract, covering every function in Functionality, usable to build
   Logos modules that hold, receive, and send on the Logos Blockchain.
2. Every function MUST be callable through `logosctl`, with a working,
   documented invocation and example response, and every documented invocation
   MUST run as a doctest in CI.
3. The delivery MUST include a worked transfer example, as a script driving
   `logosctl` or a reference module: create a wallet, receive, send with change,
   and report each step's status and fee.
4. The delivery MUST include a worked exchange example: a watch-only deployment
   handing out deposit addresses and detecting deposits, with withdrawals built
   there, authorised by an offline wallet, and submitted from the watch-only
   one.
5. The delivery MUST include a worked sequencer example: a policy granting a
   module unattended inscriptions to one channel, the module inscribing under
   it, and an inscription past the policy's limits being refused.
6. The documentation MUST describe the approval flow a user sees, and how an
   application presents an approval request in Basecamp and when hosting Logos
   Core itself.
7. Error messages MUST be clear and actionable, carrying the machine-readable
   value of Functionality [#45](#f-45) alongside the text.
8. The documentation MUST publish the derivation scheme with test vectors, the
   address encoding, the detached draft encoding with examples, and the message
   signing format.
9. The delivery MUST include a migration guide from the wallet functions of
   `blockchain_module` and the C bindings, mapping each to its replacement or
   stating that it is dropped.

#### Reliability

1. Wallet storage MUST be crash-safe: a crash at any point leaves either the
   state before or the state after an operation, never a partial one, and never
   loses the record of a submitted transaction.
2. A wallet restored from its mnemonic on a fresh device MUST converge to the
   same keys, notes, balances, and history as the original.
3. A reorganisation MUST leave notes, balances, and history consistent with the
   node's new chain once sync completes.
4. A node becoming unreachable, slow, or behind MUST be reported, and MUST NOT
   cause the wallet to report stale balances as current.
5. Concurrent drafts from different callers MUST NOT spend the same note.
6. Unattended signing under a policy MUST keep working across a wallet restart
   without a person unlocking it, where the deployer has chosen an unlock method
   that allows it (such as a key held by the operating system's keystore), and
   the documentation MUST state the trade-off.

#### Performance

1. The documentation MUST state the measured proving time and peak memory for a
   transfer spending a stated number of notes, on stated desktop and
   mobile-class hardware.
2. The documentation MUST state sync throughput, and the time to restore a
   wallet with a stated history from its birthday.
3. The documentation MUST state the time from a sequencer's request to a
   submitted inscription under a standing policy, so a zone can budget its block
   cadence.
4. Benchmarks MUST be reproducible from the test suite against a local network
   or the testnet.

#### Supportability

01. The module MUST be built and tested against a local Logos Blockchain network
    and against the testnet LEZ testnet 0.4 runs on.
02. End-to-end integration tests MUST run in CI against a local Logos Blockchain
    network, with the wallet module and the Blockchain node module loaded in a
    Logos Core runtime, and the wallet calling the node through its contract.
03. CI MUST build and test the module and MUST be green on the default branch of
    every repository the work lands in.
04. Every hard requirement in Functionality, Usability, Reliability, and
    Performance MUST have at least one corresponding test.
05. Tests MUST cover, at minimum: restore from mnemonic with discovery across a
    gap; every transaction kind built, validated, authorised, submitted, and
    final; a draft that fails validation; a watch-only wallet refusing to spend;
    detached authorisation across two wallets; a reorganisation during sync and
    after submission; a node that is behind; two callers spending from one
    wallet; an approval denied; a policy that covers a call, one that does not,
    and one whose rate limit is reached.
06. A README MUST document end-to-end usage through `logosctl`, from installing
    and loading the module to sending a transfer.
07. The delivery MUST include integrator documentation in repo covering the
    journeys of a wallet application, an exchange, a node operator, and a zone
    sequencer.
08. The delivery MUST include a get started guide that takes a developer from an
    empty repository to a Logos Core module that calls the wallet module through
    its generated client.
09. The delivery MUST include a full API reference, generated from the module
    contract and published per version.
10. The delivery MUST include a design decision log in repo recording each major
    design decision and choice: the context, the options considered, the option
    chosen, and the rationale and trade-offs. The log MUST be updated as
    decisions are made and MUST be current at each milestone delivery.

#### + Privacy and Security

1. Secrets (seed and secret keys) MUST NOT leave the module except through the
   administrative exports of Functionality [#43](#f-43), and MUST NOT appear in
   logs, errors, or notifications.
2. Decrypted secrets MUST be zeroised when the wallet locks and when they are no
   longer needed.
3. The wallet MUST use a fresh address for change and, by default, for each
   receive request, and the documentation MUST state what address reuse reveals.
4. The delivery MUST include a privacy properties document covering: what a node
   operator learns from the wallet's calls, including when the node is in
   another party's infrastructure; what the chain reveals about a transfer, its
   notes, and its addresses to any observer; how note selection can link a
   user's addresses; and how a user discloses less.
5. The wallet MUST NOT send labels or other local metadata to the node.
6. The delivery MUST include a threat model for the module covering a malicious
   caller module, a caller abusing a standing policy, a malicious node, a stolen
   device with a locked wallet, and a compromised host process.

### Soft Requirements

1. The wallet SHOULD support Keycard as the first external signer through the
   interface of Functionality [#32](#f-32).
2. The wallet SHOULD support restoring the Logos Blockchain wallet and the LEZ
   wallet ([RFP-028](./RFP-028-lez-wallet-api.md)) from one mnemonic, with
   chain-separated derivation paths agreed between the two.
3. The wallet SHOULD support batch transfers to many recipients in one
   transaction, for exchange withdrawals.
4. The wallet SHOULD support multi-party authorisation of one draft through the
   detached encoding, for custodians that require more than one approver.
5. The wallet SHOULD let a user consolidate small notes, with the cost shown
   before approval.

### Out of Scope

- **Node functions**: following the chain, serving state, and relaying belong to
  the [Blockchain Node API](https://github.com/logos-co/ecosystem/issues/219).
- **Running a node, validating, mining, or Blend participation itself.** The
  wallet builds and authorises the transactions an operator sends (declarations,
  claims, joining Blend); doing the work they relate to is the node's.
- **A separate keystore module.** Keys stay in the wallet module. A keystore
  module that holds keys apart from the wallet is planned but not part of this
  RFP.
- **A GUI wallet application.** This RFP delivers the module and its contract.
- **The LEZ wallet**, specified in [RFP-028](./RFP-028-lez-wallet-api.md).
- **What Logos Core provides**: hosting, pairing, remote transport, and caller
  authentication.

## ⚠ Platform Dependencies

- **Blockchain Node API**
  ([ecosystem#219](https://github.com/logos-co/ecosystem/issues/219)): the
  wallet reaches the chain only through it, for notes, gas prices, finality, and
  submission. Until it is available as a separate module, the wallet's tests run
  against the node functions of `blockchain_module`, and the gap is recorded in
  the design decision log.
- **Logos Core module contracts and per-method grants**: needed for the
  categories, grants, and policies of Functionality [#39](#f-39) to
  [#44](#f-44).
- **An approval surface**: Logos Core or Basecamp needs a way for a module to
  raise an approval request to the user and receive the answer.
- **Peering**: for a wallet using a node on another device.
- **Logos Runtime DK**
  ([ecosystem#238](https://github.com/logos-co/ecosystem/issues/238)): for
  applications outside Basecamp.
- **LEZ Node API (RFP-027)**: in sequencer mode a LEZ node depends on this
  wallet to inscribe blocks. The policy of Functionality [#41](#f-41) is
  designed for that caller.

## Open Questions

1. The derivation scheme: curves, purpose, coin type, and path layout per key
   role.
2. Whether deposit addresses can be derived from public material for the Logos
   Blockchain's key types, and if not, the recommended deposit model.
3. The policy format for unattended signing, and how a deployment unlocks a
   wallet that signs unattended.
4. Whether staking declarations, reward claims, and Blend belong in this wallet
   or in an operator module that uses it.
5. Whether one mnemonic backs both the Logos Blockchain and the LEZ wallets, and
   whether the two share a detached draft encoding.

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [Blockchain Node API](https://github.com/logos-co/ecosystem/issues/219)
- [LEZ Wallet API (RFP-028)](./RFP-028-lez-wallet-api.md)
- [LEZ Node API (RFP-027)](./RFP-027-lez-node-api.md)
- [Integrate Logos into your application](https://docs.logos.co/build-an-app/integrate-logos)
- [Logos glossary](https://docs.logos.co/get-started/glossary)
- [logos-blockchain](https://github.com/logos-blockchain/logos-blockchain): the
  Logos Blockchain node, its wallet, and its C bindings
- [BIP-39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki),
  [BIP-32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki),
  [SLIP-0010](https://github.com/satoshilabs/slips/blob/master/slip-0010.md),
  [BIP-174 (PSBT)](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki)

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
