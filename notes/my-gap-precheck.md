# My independent pre-check of the LEZ gap bullets (parallel to the reviewer)

7 bullets cite capabilities that NO FFI exposes, so under the module framing
they wrongly credit LEZ:

| Line | Function | Cites | Reachable? |
| --- | --- | --- | --- |
| 248  | 1.3 network id      | getChannelId (sequencer)   | NO FFI |
| 292  | 1.4 API description | getSchema (indexer)        | NO FFI |
| 404  | 1.8 balance         | getAccountBalance (seq)    | NO FFI (but see below) |
| 558  | 1.12 constr. params | getAccountsNonces (seq)    | NO FFI (but see below) |
| 1249 | 1.29 historical     | getAccountAtBlock (idx)    | NO FFI |
| 1350 | 1.32 subscribe      | subscribeToFinalizedBlocks | NO FFI |
| 1401 | 1.33 resume         | subscribeToFinalizedBlocks | NO FFI |

Verified directly: grep of lez/indexer/ffi/src/api/query.rs for
account_at_block / at_block / subscribe returns ABSENT for all three.
lez-indexer-module header has 0 matches for "subscribe".

## Nuance the reviewer should catch
- 1.8 balance and 1.12 nonces: the sequencer methods are unreachable, BUT the
  indexer Account record carries `balance` and `nonce`
  (lez/indexer/service/protocol/src/lib.rs:133-138), and query_account IS
  exposed. So an application CAN obtain both, just not via a dedicated
  accessor. These are "present but awkward", not absent. Easy bridge.
- 1.29 and 1.33 are the worst: they analyse the QUALITY of a capability
  (pinned read, resume semantics) that applications cannot reach at all.
- 1.32: the lez_core module also has no subscription; the only push anywhere
  in LEZ stops at the indexer RPC. Contrast with L1, whose module DOES expose
  three block signals.

## Also to check
- 1.24 status: bullet says status is derived from block bedrock_status. At the
  module boundary the only status is poll_transaction_status returning a BOOL
  (and collapsing errors to false). Likely UNDERSTATED.
- 1.5 / 1.9: bullets say "comparable". True via indexer FFI query_block /
  query_transaction, so probably still correct, but the module flattens
  not-found and error into an empty string, which is worth noting.
