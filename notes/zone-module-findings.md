# Zone module investigation (no file changes yet)

## The correct framing (user's)
The **zone module** is the API surface to enhance. Whether it satisfies a call
from a local indexer, a remote sequencer, or a mempool is an implementation
detail. Therefore: **a sequencer or indexer capability NOT exposed by the zone
module is still a gap.**

This invalidates how the appendix currently credits LEZ. It credits indexer
methods that the module cannot reach.

## Repos (newly cloned)
- `logos-blockchain/logos-execution-zone-module` @ `b220144` (2026-08-27)
- `logos-blockchain/lez-indexer-module` (separate module)
- `logos-blockchain/logos-blockchain-module` (L1 equivalent)

## The zone module
`lez_core` v0.4.0, C++/CMake Logos Core module.
Single interface header: `src/lez_core_module.h` (110 lines).
Implementation: `src/lez_core_module.cpp` (1345 lines).
metadata.json declares one external library: `wallet_ffi`.

### KEY FINDING 1: the module never touches the indexer
`grep -niE "indexer|getBlock|getTransaction|subscribe"` over the .cpp returns
NOTHING. All 58 external calls are `wallet_ffi_*`.

### KEY FINDING 2: the chain path is sequencer-only
wallet-ffi -> lez/wallet WalletCore -> `MultiSequencerClient`
(`lez/wallet/src/lib.rs:105,177`). There is NO indexer client anywhere in the
wallet. So indexer-only capabilities are unreachable by construction.

### KEY FINDING 3: the FFI is not the bottleneck
wallet-ffi exports 63 `#[no_mangle] extern "C"` functions; the module uses 58.
The 7 unused are PDA derivation, key import, and a serialization helper. NONE
are blockchain reads. So the missing reads are absent from the FFI itself, not
merely unwrapped by the module.

## What the zone module exposes (grouped)
Wallet lifecycle: create_new, open, save, restore_storage, wallet_dir
Accounts: create_account_public/private, list_accounts, get_account_public/
  private, get_public_account_key, get_private_account_keys,
  register_public/private_account
Encoding: account_id_to_base58, account_id_from_base58
Balance: get_balance, get_vault_balance
Sync: sync_to_block, get_last_synced_block, get_current_block_height
Transfers: transfer_public/shielded/deshielded/private/shielded_owned/
  private_owned
Generic tx: send_generic_public_transaction, send_generic_private_transaction,
  send_program_deployment_transaction
Programs (ELF bytes): authenticated_transfer_elf, token_elf, amm_elf, ata_elf
Status: poll_transaction_status (returns bool)
Bridge: bridge_withdraw
Vault: vault_claim, vault_claim_private
Naming: check_label_available, add_label, resolve_label,
  get_all_labels_for_account
Config: get_sequencer_addr, name, version
Pinata (demo/game): claim_pinata + 2 private variants

## Sequencer methods the wallet consumes (so could be surfaced)
get_last_block_id(4), get_transaction(3), get_account(3),
get_proofs_and_root(2), get_program_ids(2), get_block(2),
send_transaction(1), get_block_range(1), get_accounts_nonces(1),
get_account_balance(1). Not used: check_health, get_channel_id.

## Consequences for the appendix (34 functions)
Under the module framing, LEZ rows currently crediting indexer methods must be
re-marked as gaps. Affected:
- 1.2 health/sync: indexer getStatus unreachable. Module has
  get_last_synced_block / get_current_block_height, which is a sync position
  but no health or stall diagnostics.
- 1.5 get block: sequencer getBlock exists and wallet calls it, but the MODULE
  exposes no block read. GAP.
- 1.9 read transaction: same. Wallet calls get_transaction; module exposes none.
  GAP.
- 1.24 status: module has poll_transaction_status returning BOOL only. No
  state, no finality, no position. Much weaker than the appendix implies.
- 1.29 historical state: indexer getAccountAtBlock unreachable. GAP.
- 1.31 address history: indexer getTransactionsByAccount unreachable. GAP.
- 1.32 subscribe: indexer subscribeToFinalizedBlocks unreachable. GAP.
- 1.3 network id: getChannelId exists on sequencer but the wallet does not call
  it and the module does not expose it. GAP (module has get_sequencer_addr,
  which is a config value, not a chain identity).
- 1.6 chain tip: module has get_current_block_height. PARTIAL (height only,
  no id/hash/timestamp).
- 1.7/1.8 account/balance: module HAS get_account_public/private and
  get_balance. Comparable.
- 1.30 pagination: nothing paginated is exposed at all.
- 1.34 errors: need to check WalletFfiError taxonomy.

## Still to check
- lez-indexer-module: is it a SEPARATE module that does expose indexer reads?
  If so, the framing question is whether that counts. User said "zone module"
  specifically, but this needs confirming before I write.
- logos-blockchain-module: the L1 equivalent, same treatment needed for L1 rows.
- WalletFfiError variants for 1.34.
