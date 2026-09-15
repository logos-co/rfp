# Indexer FFI as the target surface (investigation, no file changes)

All clones under `~/src/logos-blockchain/`:
lez-indexer-module, logos-blockchain-module, logos-execution-zone,
logos-execution-zone-module.

## The layer stack for LEZ reads

    lez_indexer_module (C++, 12 methods)   <- Logos Core module
      wraps the 8 queries
    indexer_ffi (Rust, 8 query fns + 2 lifecycle)   <- THE TARGET
      wraps
    indexer RPC (12 methods incl. 1 subscription)
      reads from
    indexer store (populated by internal block-stream ingestion)

Separately, and NOT connected to the above:

    lez_core module (C++, 41 methods)
      wraps
    wallet_ffi (Rust, 63 fns)
      wraps
    WalletCore -> MultiSequencerClient   <- sequencer only, NO indexer client

## KEY: the FFI is the bottleneck, not the module

CORRECTION (from a later source review): the module has 11 public methods, not
8. It calls all 8 FFI queries plus both lifecycle functions, and adds
`reset_storage`, which wraps no FFI function (it calls the module's own stop
then deletes the store directory in C++). It also flattens the FFI's error
signalling, returning an empty string for both not-found and failure. Nothing
is dropped at the module layer. So a missing capability is missing because the
FFI does not export it.

FFI exports (`lez/indexer/ffi/src/api/`):
- lifecycle.rs: start_indexer, stop_indexer
- query.rs: query_last_block, query_status, query_block, query_block_by_hash,
  query_account, query_transaction, query_block_vec,
  query_transactions_by_account
- memory.rs: 1 free fn; logging.rs: 1

## Indexer RPC methods NOT exposed through the FFI (4 of 12)

| RPC method | FFI | Consequence |
| --- | --- | --- |
| `getAccountAtBlock` | ABSENT | no historical/pinned state read at all |
| `subscribeToFinalizedBlocks` | ABSENT | no push; consumers must poll |
| `getSchema` | ABSENT | no machine-readable description reachable |
| `checkHealth` | ABSENT | (minor; getStatus is richer and IS exposed) |

Verified: grep for `at_block|atblock|historical` in the FFI returns nothing;
grep for `subscri|callback` returns only internal ingestion references
(`lez/indexer/ffi/src/api/lifecycle.rs:122` writes blocks into the store as a
side effect), no exposed callback.

## Mapping to the 34 catalogue functions, via the indexer FFI

Present through the FFI:
- 1.2 health/sync    -> query_status (state, tip, stall diagnostics)
- 1.5 get block      -> query_block, query_block_by_hash
- 1.6 chain tip      -> query_last_block
- 1.7 get account    -> query_account
- 1.9 read tx        -> query_transaction
- 1.30 paginate      -> query_block_vec (cursor), query_transactions_by_account
                        (offset). Two idioms, as at the RPC layer.
- 1.31 address hist  -> query_transactions_by_account

Absent through the FFI (gaps under the module framing):
- 1.29 historical state at a version -> getAccountAtBlock not exported
- 1.32 subscribe to blocks/events    -> subscription not exported
- 1.4 machine-readable description   -> getSchema not exported
- 1.3 network identity               -> getChannelId is a SEQUENCER method;
      neither FFI exposes it
- 1.8 balance                        -> no balance query in the indexer FFI
      (getAccountBalance is sequencer-side); query_account returns the account
      record, so a balance is derivable but not a first-class read
- 1.15/1.16/1.17 simulate, cost, fee -> nothing; sequencer has none either
- 1.21 submit                        -> indexer is read-only by design; submit
      lives in wallet_ffi/lez_core
- 1.24 status                        -> no per-transaction status; lez_core has
      only poll_transaction_status returning a BOOL
- 1.25 wait for confirmation         -> nothing
- 1.26 mempool                       -> nothing anywhere
- 1.27/1.28 effects and events       -> nothing; programs emit no events at all
- 1.34 errors                        -> need to check the FFI error taxonomy

## L1 module for contrast (logos-blockchain-module, ~32 methods)
Notably MORE complete on reads than the zone module: get_block, get_blocks,
get_transaction, get_cryptarchia_info, get_block_events, get_time_info, plus
three block signals (newBlock, processedBlock, libBlock) driven by C callbacks.
So L1 already exposes block subscription through its module; LEZ does not
through either module.

## Open question for the user (asked)
Whether LEZ rows should be assessed against the indexer FFI alone, or indexer
FFI + wallet FFI together (since submission and balance only exist in the
latter). The user's latest message says the indexer module FFI is the target,
which suggests reads are judged against indexer_ffi and write/wallet paths
remain wallet_ffi. Need to confirm before editing.
