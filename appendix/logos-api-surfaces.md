# Appendix: Logos API Surfaces

This appendix is a reference listing of what the Logos blockchain APIs expose
today, layer by layer. It is descriptive: it records the surfaces as they are,
so that other documents can cite a stable inventory rather than re-deriving it.
It makes no recommendation about what should change.

Everything below was read from source at these commits:

| Repository                                     | Commit    | Role                               |
| ---------------------------------------------- | --------- | ---------------------------------- |
| `logos-blockchain/logos-blockchain`            | `ecb2cc6` | Logos L1 node                      |
| `logos-blockchain/logos-execution-zone`        | `47eba25` | LEZ sequencer, indexer, wallet     |
| `logos-blockchain/logos-execution-zone-module` | `b220144` | LEZ core module (`lez_core` 0.4.0) |
| `logos-blockchain/lez-indexer-module`          | `a1c10fe` | LEZ indexer module (1.1.1)         |
| `logos-blockchain/logos-blockchain-module`     | current   | L1 module (`blockchain_module`)    |

## How the layers stack

An application on Logos Core does not talk to a node directly. It calls a
module, the module calls an FFI, and the FFI calls a service. Two independent
stacks serve LEZ, and they do not meet:

```
  LEZ reads                          LEZ wallet and writes
  ---------                          ---------------------
  lez_indexer_module   (12 methods)  lez_core module     (46 methods)
    wraps 1:1                          wraps
  indexer_ffi          (8 queries)   wallet_ffi          (63 functions)
    wraps                              wraps
  indexer RPC          (12 methods)  WalletCore
    reads from                         talks to
  indexer store                      MultiSequencerClient
                                       calls
                                     sequencer RPC       (12 methods)
```

The consequence worth stating plainly: the `lez_core` module holds no indexer
client, and the indexer stack performs no writes. A capability reaches an
application only if it is exposed by the module at the top of one of these two
stacks.

Logos L1 has a single stack:

```
  blockchain_module  (29 methods, plus 3 block signals)
    wraps
  c-bindings         (46 exported functions)
    wraps
  Logos L1 node HTTP API  (42 routes)
```

## 1. LEZ Indexer FFI

`lez/indexer/ffi/src/api/`. Ten exported functions: eight queries plus two
lifecycle. This is the read surface an application reaches through
`lez_indexer_module`, which wraps all eight queries one to one.

| Function                        | Parameters                               | Returns                                    |
| ------------------------------- | ---------------------------------------- | ------------------------------------------ |
| `query_last_block`              | indexer                                  | `LastBlockIdResult`                        |
| `query_status`                  | indexer                                  | `*mut c_char` (JSON)                       |
| `query_block`                   | indexer, `block_id`                      | `PointerResult<FfiBlockOpt>`               |
| `query_block_by_hash`           | indexer, `hash`                          | `PointerResult<FfiBlockOpt>`               |
| `query_account`                 | indexer, `account_id`                    | `PointerResult<FfiAccount>`                |
| `query_transaction`             | indexer, `hash`                          | `PointerResult<FfiOption<FfiTransaction>>` |
| `query_block_vec`               | indexer, `before`, `limit`               | `PointerResult<FfiVec<FfiBlock>>`          |
| `query_transactions_by_account` | indexer, `account_id`, `offset`, `limit` | `PointerResult<FfiVec<FfiTransaction>>`    |
| `start_indexer`                 | config path                              | `OperationStatus`                          |
| `stop_indexer`                  | indexer                                  | `OperationStatus`                          |

The error type is four coarse variants: `Ok`, `NullPointer`,
`InitializationError`, `ClientError`. `ClientError` covers every failure mode,
so a caller cannot distinguish a missing record from an unreachable backend.

## 2. LEZ Indexer RPC

`lez/indexer/service/rpc/src/lib.rs`. Twelve methods, one of them a
subscription. The right-hand column records whether the indexer FFI exposes it.

| Method                       | Signature                                                | In FFI |
| ---------------------------- | -------------------------------------------------------- | ------ |
| `getLastFinalizedBlockId`    | `() -> Option<BlockId>`                                  | yes    |
| `getBlockById`               | `(BlockId) -> Option<Block>`                             | yes    |
| `getBlockByHash`             | `(HashType) -> Option<Block>`                            | yes    |
| `getBlocks`                  | `(Option<BlockId> before, u64 limit) -> Vec<Block>`      | yes    |
| `getAccount`                 | `(AccountId) -> Account`                                 | yes    |
| `getTransaction`             | `(HashType) -> Option<Transaction>`                      | yes    |
| `getTransactionsByAccount`   | `(AccountId, u64 offset, u64 limit) -> Vec<Transaction>` | yes    |
| `getStatus`                  | `() -> IndexerStatus`                                    | yes    |
| `getAccountAtBlock`          | `(AccountId, BlockId) -> Account`                        | **no** |
| `subscribeToFinalizedBlocks` | subscription, item `BlockId`                             | **no** |
| `getSchema`                  | `() -> serde_json::Value`                                | **no** |
| `checkHealth`                | `() -> ()`                                               | **no** |

`IndexerStatus` carries `state`, `last_error`, `indexed_block_id`, and
`stall_reason`, so it reports how far ingestion has reached and why it stopped.
`checkHealth` returns unit and is the weaker of the two.

## 3. LEZ Sequencer RPC

`lez/sequencer/service/rpc/src/lib.rs`. Twelve methods. This is the write path
and the live-state path. No FFI wraps it directly; `wallet_ffi` reaches it
through `WalletCore` and `MultiSequencerClient`.

| Method              | Signature                                                                  | Also on indexer              |
| ------------------- | -------------------------------------------------------------------------- | ---------------------------- |
| `sendTransaction`   | `(LeeTransaction) -> HashType`                                             | no                           |
| `getBlock`          | `(BlockId) -> Option<Block>`                                               | as `getBlockById`            |
| `getBlockRange`     | `(BlockId start, BlockId end) -> Vec<Block>`                               | as `getBlocks`               |
| `getLastBlockId`    | `() -> BlockId`                                                            | as `getLastFinalizedBlockId` |
| `getAccount`        | `(AccountId) -> Account`                                                   | yes                          |
| `getAccountBalance` | `(AccountId) -> u128`                                                      | no                           |
| `getAccountsNonces` | `(Vec<AccountId>) -> Vec<Nonce>`                                           | no                           |
| `getTransaction`    | `(HashType) -> Option<(LeeTransaction, BlockId)>`                          | yes                          |
| `getProofsAndRoot`  | `(Vec<Commitment>) -> (Vec<Option<MembershipProof>>, CommitmentSetDigest)` | no                           |
| `getProgramIds`     | `() -> BTreeMap<String, ProgramId>`                                        | no                           |
| `getChannelId`      | `() -> ChannelId`                                                          | no                           |
| `checkHealth`       | `() -> ()`                                                                 | yes                          |

Note the sequencer's `getTransaction` returns the transaction paired with its
block id, while the indexer's returns the transaction alone.

## 4. LEZ Wallet FFI and the `lez_core` module

`lez/wallet-ffi/src/`, 63 exported functions across twelve areas. The `lez_core`
module calls 56 of them; the seven it does not are PDA derivation (two), key
import (two), a private-accounts key constructor, an instruction-word free
function, and a serialisation helper. None is a chain read. The module exposes
46 methods, excluding constructor and destructor.

| Area                 | Functions | Module surface                                                       |
| -------------------- | --------: | -------------------------------------------------------------------- |
| Wallet lifecycle     |         7 | `create_new`, `open`, `save`, `restore_storage`, `wallet_dir`        |
| Accounts             |        11 | create and list accounts, get account, register account              |
| Keys                 |         8 | public and private account keys                                      |
| Transfers            |         9 | public, shielded, deshielded, private, and owned variants            |
| Generic transactions |         5 | `send_generic_public_transaction`, private variant                   |
| Program deployment   |         6 | `send_program_deployment_transaction`, ELF accessors                 |
| Labels               |         5 | `check_label_available`, `add_label`, `resolve_label`                |
| Vault                |         3 | `get_vault_balance`, `vault_claim`, `vault_claim_private`            |
| Sync                 |         3 | `sync_to_block`, `get_last_synced_block`, `get_current_block_height` |
| PDA                  |         2 | not exposed by the module                                            |
| Bridge               |         1 | `bridge_withdraw`                                                    |
| Pinata               |         3 | `claim_pinata` and two private variants                              |

Read-side methods on the module are `list_accounts`, `get_balance`,
`get_account_public`, `get_account_private`, the two key getters,
`get_vault_balance`, `get_sequencer_addr`, the three sync methods,
`poll_transaction_status`, and the label lookups. `poll_transaction_status`
returns a boolean, carrying no state, position, or finality.

## 5. Logos L1 C bindings

`c-bindings/src/`, 46 exported functions.

| Area           | Functions                                                                                                 |
| -------------- | --------------------------------------------------------------------------------------------------------- |
| Node lifecycle | `start_lb_node`, `shutdown_node`, `participate`                                                           |
| Configuration  | `generate_user_config`, `update_user_config`, `migrate_user_config`, `migrate_user_config_0_1_2`          |
| Keys           | `generate_key`, `add_key`, `remove_key`                                                                   |
| Chain reads    | `get_block`, `get_blocks`, `get_block_events`, `get_transaction`, `get_cryptarchia_info`, `get_time_info` |
| Wallet         | `get_balance`, `get_known_addresses`, `get_wallet_notes`, `transfer_funds`, `wallet_fund_tx`              |
| Submission     | `submit_signed_transaction`                                                                               |
| Subscriptions  | `subscribe_to_new_blocks`, `subscribe_to_processed_blocks`, `subscribe_to_lib_blocks`                     |
| Channels       | `channel_deposit`, `channel_deposit_with_notes`, `get_channel_state`                                      |
| Proof of work  | `pow_start_mining`, `pow_stop_mining`, `pow_claim`, `pow_claimable_rewards`                               |
| Leadership     | `leader_claim`, `get_claimable_vouchers`                                                                  |
| Blend          | `blend_info`, `blend_join_as_core_node`                                                                   |
| Networking     | `get_peer_id`                                                                                             |
| Status helpers | `is_ok`, `is_error`                                                                                       |
| Memory         | the remaining `free_*` functions                                                                          |

The committed header `c-bindings/logos_blockchain.h` declares eleven functions
and is stale against these 46; it is regenerated at build time.

## 6. Logos L1 node HTTP API

`nodes/api-common/src/paths.rs` defines 42 route constants, registered in
`nodes/node/binary/src/api/backend.rs`. Grouped by area:

| Area          | Routes                                                                                                                                                                                                                                                       |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Node          | `/version`, `/time/info`, `/network/info`, `/network/dial_peer`                                                                                                                                                                                              |
| Cryptarchia   | `/cryptarchia/info`, `/cryptarchia/headers`, `/cryptarchia/blocks`, `/cryptarchia/blocks/:id`, `/cryptarchia/blocks/:id/events`, `/cryptarchia/blocks_range`, `/cryptarchia/events/blocks/stream`, `/cryptarchia/lib-stream`, `/cryptarchia/transaction/:id` |
| Mantle        | `/mantle/metrics`, `/mantle/status`, `/mantle/gas-prices`, `/mantle/sdp/declarations`, `/mantle/sdp/snapshot`                                                                                                                                                |
| Mempool       | `/mempool/add/tx`, `/mempool/view`                                                                                                                                                                                                                           |
| Wallet        | `/wallet/:public_key/balance`, `/wallet/transactions/transfer-funds`, `/wallet/sign/ed25519`, `/wallet/sign/zk`, `/wallet/fund`                                                                                                                              |
| Channels      | `/channel/:id`, `/channel/deposit`                                                                                                                                                                                                                           |
| SDP           | `/sdp/declaration`, `/sdp/activity`, `/sdp/withdrawal`, `/sdp/set-declaration-id`                                                                                                                                                                            |
| Blend         | `/blend/info`, `/blend/join`, `/blend/transactions/disperse`, `/blend/transactions/pending`                                                                                                                                                                  |
| Proof of work | `/pow/mining/start`, `/pow/mining/stop`, `/pow/claim`, `/pow/rewards/claimable`                                                                                                                                                                              |
| Leadership    | `/leader/claim`, `/leader/claim/vouchers`                                                                                                                                                                                                                    |
| Admin         | `/admin/tracing/filter`                                                                                                                                                                                                                                      |
| Discovery     | `/api-docs/openapi.json`, `/swagger-ui`                                                                                                                                                                                                                      |

An OpenAPI document is served at `/api-docs/openapi.json`. It omits five wired
routes, including both signing endpoints, declares no response body for most
successful responses, and registers three component schemas.

## 7. Logos L1 module

`logos-blockchain-module`, 29 public methods over the C bindings, plus three Qt
signals driven by C callbacks: `newBlock`, `processedBlock`, and `libBlock`. It
exposes the chain reads the LEZ core module does not: `get_block`, `get_blocks`,
`get_transaction`, `get_cryptarchia_info`, `get_block_events`, and
`get_time_info`.

## 8. Coverage summary

Counting distinct capabilities rather than method names:

| Layer              | Surface                            | Count |
| ------------------ | ---------------------------------- | ----: |
| LEZ indexer RPC    | read methods plus one subscription |    12 |
| LEZ indexer FFI    | queries plus lifecycle             |    10 |
| LEZ indexer module | wrapped methods                    |    12 |
| LEZ sequencer RPC  | read and write methods             |    12 |
| LEZ wallet FFI     | exported functions                 |    63 |
| LEZ core module    | public methods                     |    46 |
| L1 node HTTP       | route constants                    |    42 |
| L1 C bindings      | exported functions                 |    46 |
| L1 module          | public methods, plus 3 signals     |    29 |

Two observations follow from the inventory. Four indexer RPC methods stop at the
FFI boundary rather than at the module boundary, so the FFI is where the LEZ
read surface is bounded. And nine sequencer methods have no indexer counterpart
at all, of which `getAccountBalance` and `getAccountsNonces` are derivable from
data the indexer already stores, since its `Account` record carries both
`balance` and `nonce`.
