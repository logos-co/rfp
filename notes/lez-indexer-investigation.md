# LEZ indexer: answers to the open questions

Source investigation against `logos-execution-zone` @ `47eba25`, verified
against the code. Spot-checks by me marked VERIFIED.

## The pivotal finding

**The indexer already executes every transaction on ingest and computes exactly
the state diff an effects API would return, then throws it away.**

`lez/chain_state/src/apply.rs:159` (VERIFIED):

```rust
transaction
    .clone()
    .execute_on_state(state, block.header.block_id, block.header.timestamp)
```

`execute_on_state` is `compute_state_diff` plus apply. The diff type is
`StateDiff { signer_account_ids, public_diff, new_commitments, new_nullifiers,
program }`. So an effects API and a simulate API are the same piece of work:
persist and expose something already being computed.

The comment in `lez/common/src/transaction.rs:141-156` says the indexer
"replays blocks the sequencer already validated and inscribed on Bedrock, so it
trusts those inscriptions and re-derives state without re-validating them".

## Two premises of mine that were wrong

**1. `public_actions` is inverted from what I assumed.** It is present on
PRIVACY-PRESERVING transactions and ABSENT on public ones. Not an oversight:
private state exists on-chain only as commitments, so plaintext must travel
inside the transaction, while public state is in the replicated state machine
and can simply be queried.

`lez/indexer/service/protocol/src/lib.rs:220-253`. `PublicMessage` has
`{program_id, account_ids, nonces, instruction_data}` and no post-state.
`PrivacyPreservingMessage` has `public_actions: Vec<PublicActionWithID>`.

**2. `bedrock_status` is a constant, not a status.** VERIFIED at
`lez/indexer/core/src/block_store.rs:242-243`:

```rust
let mut stored = block.clone();
stored.bedrock_status = BedrockStatus::Finalized;
```

The indexer unconditionally overwrites it before persisting. Every block it
returns reads `Finalized`; `Safe` is unreachable in indexer data. It looks like
a working status field and is not one. Any RFP text relying on it is wrong.

## Answers

| Q | Finding |
| --- | --- |
| Q1 effects | Cannot determine a public transaction's balance effect from indexer data. Only route is `getAccountAtBlock(N)` vs `(N-1)`, and that is not even in the FFI. |
| Q2 private | Third parties can read neither pre- nor post-states of the private leg; post-state plaintext is inside `encrypted_post_state`, needing the viewing key. The public leg of a shielded transaction IS readable. |
| Q3 PDA | `getAccount` returns program data as an opaque base64 blob, max 100 KiB. Token balance lives in `Account.data`, separate from `Account.balance`. Requires client-side PDA derivation plus Borsh decode against an unpublished schema. |
| Q4 batch | No batch account read at RPC or FFI. N accounts is N round trips. But `multi_get_cf` is already used for blocks and transactions in the same file, so it is a small change. |
| Q5 simulate | Architecturally straightforward: executor, state and diff type are all already on the indexer. Caveat: simulates against finalised state, not a live tip. |
| Q6 mempool | No path at all. The mempool is an in-process `tokio::sync::mpsc` channel inside the sequencer, with no query surface. Net-new work. |
| Q7 status | See the `bedrock_status` finding above. A real per-transaction status API is needed. `IndexerStatus` is where the actual signal lives. |
| Q8 events | Programs emit nothing. `ProgramOutput` has no log or event field and never reaches the indexer. Detecting a token transfer today means decoding `instruction_data` against a private Rust enum plus diffing Borsh-decoded balances. |
| Q9 pagination | Both cursors are STABLE. `before` is a block id on immutable data; `offset` indexes a per-account append-only sequence. The real gaps: no account transaction count, no newest-first ordering, undocumented exclusive-`before`. |
| Q10 retention | Never prunes. Full `V03State` snapshot every 100 blocks, kept for ever. No retention config. Real operational cost for an exchange. |

## The gap that matters most

`getAccountAtBlock` exists on the indexer RPC but NOT in the FFI. Since the RFP
targets the FFI, historical account state, the only current route to a balance
delta, is unreachable by an integrator running an indexer module.

## Corroborating the framing

`lez/sequencer/service/rpc/src/lib.rs:46-47` (VERIFIED) carries its own note:

```
// TODO: These functions should be removed after wallet starts using indexer
// for this type of queries.
```

The sequencer is knowingly serving indexer-shaped queries as an interim
measure. Treating it as a disappearing black box matches the code's own intent.

## Caveats the investigation flagged

- RocksDB `multi_get_cf` snapshot semantics under concurrent ingestion are not
  settled by source alone; a torn paging read is unlikely but not excluded.
- `next_messages` in the zone SDK is pinned to a rev not present in the local
  checkout, so the Q6 conclusion rests on the `ZoneMessage` doc comment and the
  indexer's call pattern rather than on reading that function.
- `getAccountAtBlock` worst case replays 99 blocks from the nearest breakpoint,
  each running the RISC0 executor. Latency not measurable from source.
- One sequencer capability has no obvious indexer home: `getProofsAndRoot`,
  Merkle membership proofs against the live commitment set, needed to BUILD
  privacy-preserving transactions. That is wallet-side work, so out of scope
  here, but it should not be lost.
