# Logos Stack Trust Assumptions for RFPs

**Snapshot date:** 2026-04-28
**Source:** [roadmap.logos.co](https://roadmap.logos.co), tracked at
[github.com/logos-co/roadmap](https://github.com/logos-co/roadmap)

This appendix captures what the Logos technology stack commits to in
its FURPS documents as of the snapshot date, so that RFPs can scope
their privacy and security requirements correctly. RFPs should not
claim to defend against things the stack already guarantees, and must
not claim to inherit guarantees the stack does not commit to.

The snapshot is intentionally narrow: each entry quotes the FURPS
language verbatim and links to the exact commit-dated source. When the
stack FURPS evolve, this appendix should be re-snapshotted before the
next batch of RFPs is updated.

## Sources reviewed

| Component | File | Last updated | Source |
|-----------|------|--------------|--------|
| Blockchain / LEZ | `blockchain/furps/index.md` | 2026-03-23 | [link](https://github.com/logos-co/roadmap/blob/v4/content/blockchain/furps/index.md) |
| AnonComms (overview) | `anoncomms/furps/index.md` | 2025-12-01 | [link](https://github.com/logos-co/roadmap/blob/v4/content/anoncomms/furps/index.md) |
| AnonComms Mix | `anoncomms/furps/mix.md` | 2026-04-08 | [link](https://github.com/logos-co/roadmap/blob/v4/content/anoncomms/furps/mix.md) |
| AnonComms RLN | `anoncomms/furps/rln.md` | 2026-03-24 | [link](https://github.com/logos-co/roadmap/blob/v4/content/anoncomms/furps/rln.md) |
| Logos Core | `logoscore/furps/logos-core.md` | 2025-12-15 | [link](https://github.com/logos-co/roadmap/blob/v4/content/logoscore/furps/logos-core.md) |
| Messaging overview | `messaging/furps/index.md` | 2025-11-13 (dated) | [link](https://github.com/logos-co/roadmap/blob/v4/content/messaging/furps/index.md) |
| Messaging Mix | `messaging/furps/core/mix.md` | 2025-11-13 (dated) | [link](https://github.com/logos-co/roadmap/blob/v4/content/messaging/furps/core/mix.md) |
| Messaging RLN Relay | `messaging/furps/core/rln_relay.md` | 2025-11-13 (dated) | [link](https://github.com/logos-co/roadmap/blob/v4/content/messaging/furps/core/rln_relay.md) |
| Storage privacy filesharing | `storage/furps/privacy-preserving-filesharing-furps.md` | 2026-04-08 | [link](https://github.com/logos-co/roadmap/blob/v4/content/storage/furps/privacy-preserving-filesharing-furps.md) |
| Storage anonymous downloads | `storage/furps/anonymous-downloads-over-mix.md` | 2026-04-08 | [link](https://github.com/logos-co/roadmap/blob/v4/content/storage/furps/anonymous-downloads-over-mix.md) |

## What each component commits to

### Blockchain and LEZ

The Blockchain FURPS commits to the following items relevant to RFP
threat modelling. Quotations are verbatim.

**Blockchain consensus and propagation:**

- "Leaders can propose blocks privately"
  (`blockchain:ppos.2`)
- "Leaders can claim block rewards without revealing their block
  proposal" (`blockchain:ppos.3`)
- "Censorship resistance against malicious broadcasters"
  (`blockchain:ppos.6`)
- "Blend edge node privacy" (`blockchain:ppos.7`)
- "Distributed block building. Enabling tagging attack resistance and
  removing the leader as SPOF" (`blockchain:block-building.21`)

**LEZ programmable privacy:**

- "LEZ supports Programmable Privacy by allowing LEZ Programs to be
  agnostic as to whether they are interacting with private or public
  accounts" (`lez:programmable-privacy.22`)
- "The same LEZ Programs can be used in both private and public
  execution contexts" (Usability `lez.7`)

**LEZ sequencer:**

- "LEZ Sequencer accepts transactions from users, orders them and
  posts them to Logos Blockchain" (`lez:sequencer.23`)
- "Sequencer manages pending vs. safe vs. confirmed transactions"
  (`lez:sequencer.24`)
- "Sequencer maintains funds to pay for blockchain transactions"
  (`lez:sequencer.25`)
- "LEZ Sequencer supports decentralized sequencing through Blockchain
  enforced sequencer coordination, ensuring crash tolerance"
  (Reliability `lez.4`)

**LEZ indexer and RPC:**

- "Indexer follows LEZ channel in blockchain" (`lez:indexer.26`)
- "Indexer validates messages in the channel, skips invalid
  messages" (`lez:indexer.27`)
- "Indexer maintains state history" (`lez:indexer.29`)
- "Indexer provides RPC endpoints for querying LEZ state"
  (`lez:indexer.30`)

**LEZ program model:**

- "Programs have defined interface exposing input/output accounts and
  contextual information (block number, random oracle, etc.)"
  (`lez:program-interface.31`)
- "Programs can call other programs deployed on LEZ"
  (`lez:cross-program-calls.32`)

**LEZ bridging:**

- "Channel Balance management" (`lez:bridging.33`)
- "Sequencer signing on withdrawal" (`lez:bridging.34`)
- "User deposits from Blockchain to LEZ" (`lez:bridging.35`)

**Blockchain liveness and finality:**

- "Blockchain prioritizes liveness over safety ensuring we are
  resilient to large network failures" (Reliability `blockchain.1`)
- "Blockchain provides 18hrs for failures to resolve before the chain
  may split requiring manual intervention" (Reliability
  `blockchain.2`)
- "Blockchain finalizes transactions in 18hrs" (Performance
  `blockchain.1`)
- "Practical finality can be achieved much sooner" (Performance
  `blockchain.2`)
- "Blocks are produced on average every 30s" (Performance
  `blockchain.3`)

What the Blockchain / LEZ FURPS does **not** commit to:

- No statement that LEZ user transactions submitted to the sequencer
  are routed over an anonymising mixnet.
- No statement about mempool or pre-confirmation observability for
  third parties.
- No statement that Indexer RPC queries are anonymised or that the
  caller of an RPC query is unlinkable from the address being queried.
- No statement about availability or privacy of off-chain storage
  used by LEZ programs.

### AnonComms

The AnonComms component delivers a mix protocol and an RLN service.
The AnonComms FURPS lists integration targets for the mix protocol;
LEZ transaction submission is not one of them.

**Mix integration targets** (`anoncomms/furps/mix.md`):

- "The libp2p mix protocol with DoS and Sybil protection is
  integrated in nim-libp2p" (Usability `mix.2`)
- "The libp2p mix protocol with DoS and Sybil protection is
  integrated into Waku Lightpush protocol as reference integration"
  (Usability `mix.3`) [verbatim quote from source; current
  stack-level term: Messaging Light Push]
- "A libp2p module with mix capability is integrated into Logos
  Core" (Usability `mix.4`)
- "The libp2p mix protocol is integrated into the Logos Chat module"
  (Usability `mix.12`)

**Mix functional properties:**

- "The libp2p mixnet is protected against trivial DoS attacks"
  (`mix.3`)
- "The libp2p mixnet is protected against a 50% + 1 Sybil attack"
  (`mix.4`)
- "Nodes can generate cover traffic to increase K-anonymity in the
  mixnet" (`mix.10`)
- "Providers can anonymously register as a hidden service"
  (`mix.11`)
- "Clients can discover and anonymously access hidden services"
  (`mix.12`)

**RLN:**

- "An RLN membership allocation service can register ID commitments
  on behalf of third parties" (`rln.1`)
- "Logos modules can use the service as client to obtain adequate
  registered RLN identities without interacting with the contract"
  (`rln.3`)
- "The RLN contract is implemented for Logos Execution Zone"
  (Usability `rln.3`)

What AnonComms does **not** commit to:

- LEZ transaction submission (user → sequencer) is not listed as a
  Mix integration target. An RFP cannot assume that transactions
  submitted to LEZ via the Logos Core wallet are anonymised at the
  network layer.

### Logos Core

Logos Core (`liblogos`) is a minimal Qt-based module loader.

- "The library shall enable loading and unloading of modules"
  (Functionality `logos-core.1`)
- "The library shall provide a central QObject Registry"
  (Functionality `logos-core.2`)
- "The library shall minimally facilitate the Qt event loop without
  providing additional functionalities (such as transports, UI,
  networking) directly" (Functionality `logos-core.5`)

What Logos Core does **not** commit to:

- It is explicitly not a transport or networking layer. Anonymity,
  privacy, and security properties for an RFP do not come from Logos
  Core itself; they come from the modules that an RFP composes.

### Messaging

Messaging FURPS (dated 2025-11-13) commits to a stack with its own
Mix and RLN. Relevant items for RFPs that use Messaging:

- Mix at the messaging layer: "Relay nodes can mount mixnet protocol,
  acting as sender, intermediary or exit nodes" (`messaging:mix.1`)
- RLN Relay for spam protection at the messaging layer:
  "Relay node can attach RLN proof for outbound messages"
  (`rln_relay.2`); "Relay node can verify RLN proof for inbound
  messages" (`rln_relay.3`)
- Store: "Provides historical message retrieval from the relay
  network" (`store.1`)
- Store explicit limitation: "(limitation) No guarantees in terms of
  message presence or retention duration" (Reliability `store.2`)

Messaging is a separate sub-system from LEZ. RFPs that use Messaging
for out-of-band communication (e.g., a chat-style interaction layer)
inherit these properties; RFPs that interact only with LEZ programs do
not.

### Storage

Storage FURPS (dated 2026-04-08) define privacy properties for
filesharing.

- "Neither the identity of publishers nor that of downloaders should
  be revealed to other participants; i.e., we want full publisher and
  downloader unlinkability. This includes queries"
  (`privacy-preserving-filesharing-furps`, Security 1)
- "Cache nodes should be able to plausibly deny knowledge of the
  contents they are caching" (`privacy-preserving-filesharing-furps`,
  Security 2)
- "The node downloading the file should not be linkable to the
  download. Note that the node that provides the file is not
  anonymized here" (`anonymous-downloads-over-mix`, Security 1)

Storage is a separate sub-system from LEZ. RFPs that use Storage for
file or asset distribution inherit these properties for the file
distribution path. They do not extend to LEZ on-chain state.

## Stack-wide trust assumptions an RFP can rely on

These are the consolidated assumptions an RFP for an LEZ program (and
its SDK / mini-app) can rely on without needing to re-prove them.

1. **Programmable privacy works as specified.** LEZ private accounts
   provide the cryptographic unlinkability properties advertised by
   the underlying primitives. The same program logic runs in both
   private and public contexts.
2. **The LEZ sequencer behaves as specified.** It accepts
   transactions, orders them, posts them to Logos Blockchain, and
   manages the pending / safe / confirmed transaction lifecycle. It
   tolerates crashes through state persistence. Decentralised
   sequencing is enforced by the blockchain.
3. **The LEZ indexer is correct.** It validates messages, skips
   invalid ones, applies blocks to local state, maintains history,
   and exposes RPC endpoints with content that reflects committed
   chain state.
4. **Blockchain liveness and finality.** Transactions finalise within
   18 hours; practical finality is achieved sooner; blocks every 30
   seconds on average.
5. **Block proposer privacy.** Logos Blockchain leaders propose
   blocks privately and claim rewards without revealing their
   proposal. The blockchain is censorship-resistant against malicious
   broadcasters at the consensus layer.
6. **LEZ bridging primitives.** Deposits from Blockchain to LEZ and
   sequencer-signed withdrawals from LEZ are provided by the
   platform.
7. **Program model.** Programs can read input/output accounts and
   contextual data (block number, random oracle), and can call other
   programs.

## Stack-wide non-guarantees an RFP must not assume

These are properties that, as of the snapshot date, the stack FURPS
do **not** commit to. RFPs that need them must either treat them as
out-of-scope explicitly or design around them at the application
layer.

1. **No anonymous LEZ transaction submission at the network layer.**
   The AnonComms Mix integration list does not include LEZ transaction
   submission. An LEZ transaction sent to the sequencer is observable
   on the network path and the sequencer learns the submitter's
   network identity.
2. **No mempool / pre-confirmation privacy.** The sequencer is
   documented to manage pending / safe / confirmed transactions.
   There is no commitment that the pending state is hidden from
   third-party observers (sequencer operators, indexer operators,
   sophisticated network observers).
3. **No anonymous indexer RPC queries.** The indexer exposes RPC
   endpoints for querying LEZ state. There is no commitment that the
   originator of a query is unlinkable from the address being
   queried, or that an RPC operator cannot correlate a sequence of
   queries to a single user.
4. **No off-chain storage privacy by default for LEZ state.** Storage
   FURPS apply to the filesharing stack, not to LEZ on-chain state
   nor to private-account witness data unless an RFP explicitly
   composes Storage for that purpose.
5. **Censorship resistance is at the consensus layer, not the
   sequencer.** "Censorship resistance against malicious
   broadcasters" applies to block propagation. There is no equivalent
   commitment that the LEZ sequencer cannot censor or reorder
   user transactions.
6. **No commitment about side channels.** Transaction timing, gas
   patterns, account size, and similar side channels visible on
   public chain state are not addressed by any FURPS.

## How RFPs should use this appendix

RFPs that build on LEZ should:

- Assume guarantees only from the "Stack-wide trust assumptions"
  list above.
- Treat anything in the "Non-guarantees" list as either explicitly
  out of scope or as a problem the application layer must address.
- Cite the specific FURPS item by component and number when relying
  on a guarantee, so reviewers can verify against this snapshot.

When the underlying FURPS change, this appendix must be re-validated
before its trust assumptions are referenced in new RFPs.
