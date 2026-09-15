# Wallet FFI as a fourth RFP: what I found

## The architecture the user described
    SDK (TypeScript, #220) consumes BOTH:
      indexer FFI  -> reads            (#219, being written now)
      wallet FFI   -> keys, signing    (NEEDS ITS OWN RFP - does not exist yet)

So #220 is ONE deliverable hitting TWO FFIs. The split is at the FFI layer, not
in the SDK. This resolves the one-SDK-or-two question and implies a 4th RFP.

## LEZ curve: secp256k1, NOT ed25519. This matters a lot.
`lee/state_machine/src/signature/mod.rs:55,68,72` uses `k256::schnorr`
(SigningKey, VerifyingKey, Signature) = BIP-340 Schnorr over secp256k1.
Confirmed k256 is the RustCrypto secp256k1 crate (`Cargo.toml:202`).
Line 40 comment references "pre-2022 BIP-340/Keycard".
`lee/state_machine/src/signature/bip340_test_vectors` exists.

=> WATCH-ONLY DERIVATION IS CRYPTOGRAPHICALLY POSSIBLE ON LEZ.
Unlike Solana/Stellar/NEAR/Sui (ed25519, hardened-only), LEZ is in the same
family as Bitcoin/Ethereum/Cosmos where public derivation works.
This is a genuine advantage for CEX integration and should be stated.

## Private accounts have a VIEWING KEY model
`lez/wallet/src/account_manager.rs:310-315`: private accounts carry
nsk / npk / vpk and `AccountId::from((&npk, &vpk, identifier))`.
vpk = viewing public key. So there are TWO watch-only-ish mechanisms:
  - public accounts: secp256k1 public derivation (BIP-32 style)
  - private accounts: viewing keys (Zcash/Monero style)
Both need explicit FFI exposure for a CEX to track deposits.

## What wallet_ffi exposes today re: keys (19 fns in keys.rs + account.rs)
create_account_public / create_account_private
create_private_accounts_key
import_public_account / import_private_account
get_public_account_key / get_private_account_keys
get_account_public / get_account_private
resolve_public_account / resolve_private_account
account_id_to_base58 / account_id_from_base58
list_accounts, get_balance
+ free_* helpers

## GAPS in wallet_ffi for a CEX (candidates for the 4th RFP)
- NO explicit derivation-path function. No BIP-32/44 path parameter anywhere.
  grep for bip32|bip39|bip44|slip.?0010|derivation_path in wallet/ + wallet-ffi/
  found NOTHING except mnemonic handling.
- `restore_data(handle, mnemonic, password, depth)` takes a DEPTH and the doc
  warns "depth parameter induces exponential growth in execution time"
  (`lez/wallet-ffi/src/wallet.rs:279`), which reads as SCANNING, not
  deterministic derivation. A CEX needs deterministic address derivation,
  not a scan.
- NO watch-only / xpub export, despite the curve supporting it.
- NO viewing-key export for private accounts as a distinct capability.
- NO signature-only export: every exported transfer builds, signs, AND submits
  in one call. A CEX with an HSM cannot sign without also submitting.
- Keycard hardware support exists in the CLI (`lez/wallet/src/cli/keycard.rs`)
  but is NOT exposed through the FFI.

## Cross-check needed
Whether the 4th RFP is a NEW issue or an expansion of #220's scope is the
user's call. My read: it is a distinct FFI surface, so it parallels #219 and
deserves its own issue.
