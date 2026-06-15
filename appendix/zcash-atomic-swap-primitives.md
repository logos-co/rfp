# Zcash Atomic Swap Primitives

What's needed for a non-custodial atomic swap between LEZ and Zcash, what Zcash
exposes today, and why the answer differs sharply between Zcash's transparent
and shielded pools.

## Two pools, two answers

Zcash has three on-chain pools. Their swap-relevant capabilities differ:

| Pool        | Address prefix    | Scripting                    | Spend authorization             | Curve     |
| ----------- | ----------------- | ---------------------------- | ------------------------------- | --------- |
| Transparent | `t1...`           | Bitcoin script (full subset) | secp256k1 ECDSA                 | secp256k1 |
| Sapling     | `zs1...`          | None                         | RedJubjub (Schnorr-like RedDSA) | Jubjub    |
| Orchard     | `u1...` / `o1...` | None                         | RedPallas (Schnorr-like RedDSA) | Pallas    |

The transparent pool is a near-clone of Bitcoin and supports HTLC-based atomic
swaps today via
[BIP-199](https://github.com/bitcoin/bips/blob/master/bip-0199.mediawiki). The
shielded pools (Sapling, Orchard) have no on-chain scripting at all, and as of
May 2026 there is no production-ready atomic swap to or from either of them.

## Transparent pool: implementable today

Zcash transparent inputs, outputs, and scripts behave as in Bitcoin. The opcodes
relevant to swaps (`OP_IF / OP_ELSE / OP_ENDIF`, `OP_SHA256`, `OP_HASH160`,
`OP_CHECKLOCKTIMEVERIFY`, `OP_CHECKSEQUENCEVERIFY`, `OP_CHECKSIG`, P2SH) all
work. The canonical HTLC layout from BIP-199 applies verbatim:

```
OP_IF
    [HASHOP] <digest> OP_EQUALVERIFY OP_DUP OP_HASH160 <seller pubkey hash>
OP_ELSE
    <num> [TIMEOUTOP] OP_DROP OP_DUP OP_HASH160 <buyer pubkey hash>
OP_ENDIF
OP_EQUALVERIFY
OP_CHECKSIG
```

### No Taproot on Zcash transparent

The transparent pool tracks Bitcoin's script system as of a point that predates
SegWit, and Zcash has not adopted SegWit: the
[ZIP-48 multisig specification](https://zips.z.cash/zip-0048) states that the
SegWit `script_type` values (Native SegWit / P2WSH and Nested SegWit /
P2SH-P2WSH) "MUST NOT be used in a Zcash context." Because Taproot
([BIP-341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki)) is
defined as SegWit version 1, it is **not available on Zcash transparent**
either. The transparent layer supports only P2PKH and P2SH outputs, signed with
**ECDSA** (not BIP-340 Schnorr).

Two consequences for this RFP:

- The key-path-indistinguishability approach used for the Bitcoin pair (a
  Taproot 2-of-2 whose cooperative spend reveals no script) is not expressible
  on Zcash transparent. The swap branch must live in an explicit P2SH script,
  which is why the HTLC route reveals a hash preimage on-chain and is publicly
  linkable.
- Unlinkability on the Zcash side therefore comes not from Taproot but from an
  **ECDSA adaptor signature** (the Fournier construction below), which moves the
  swap secret off-chain while leaving an ordinary P2SH spend on-chain. This is
  the transparent-pool analogue of what Taproot key-path spends provide for
  Bitcoin.

Both legs of an HTLC swap reveal the same hash preimage on-chain. This makes the
two legs publicly linkable across chains: a privacy limitation that
adaptor-signature constructions explicitly fix (see
[Appendix: Bitcoin and Monero Adaptor-Signature Swap Primitives](btc-xmr-adaptor-swap-primitives.md)).
Zcash transparent supports both:

- **HTLC**: standard, well-trodden, mirrors the existing ETH-LEZ reference
  implementation.
- **ECDSA adaptor signature**: Lloyd Fournier's construction with a DLEQ proof.
  The on-chain footprint becomes an ordinary 2-of-2 spend with no hash preimage
  broadcast.
- **ECDH multiplicative key aggregation**: a third route used by
  [Zwap](https://forum.zcashcommunity.com/t/zwap-unlinkable-cross-chain-atomic-swaps/55104),
  the most active 2025 Zcash atomic-swap project. Trades adaptor-sig math for
  ECDH + a hash-preimage half-HTLC. Less private than full adaptor-sig (still
  embeds a hash on both chains) but easier to implement on chains that only
  verify standard ECDSA.

For LEZ as the counterparty side, the LEZ-side primitives required (SHA-256,
BIP-340 Schnorr verification at the witness layer, block-height and timestamp
validity windows, AND-multisig witness sets) are all present today. See the
inline LEZ primitive status section in RFP-003 for file citations.

## Shielded pools: research-grade

Sapling and Orchard expose no scripting. The only cryptographic hooks a swap
protocol can touch are:

1. The **spendAuthSig**: a Schnorr-like signature over each spend. Sapling uses
   RedJubjub on the Jubjub curve; Orchard uses RedPallas on the Pallas curve.
   Both are instantiations of RedDSA.
2. The **transaction-level `nExpiryHeight`** field
   ([ZIP-203](https://zips.z.cash/zip-0203)). Consensus rule: "the transaction
   must be included in block N or earlier. Block N+1 will be too late, and the
   transaction will be removed from the mempool." The maximum is 499,999,999
   (~1185 years).

The critical limitation: **`nExpiryHeight` invalidates the *broadcast attempt*,
not the *funds*.** There is no shielded equivalent of `OP_CHECKLOCKTIMEVERIFY`
that creates a chain-enforced refund branch on a deposited note. Refund must
therefore be embedded in the swap-protocol's signed messages (e.g. a pre-signed
refund transaction exchanged before funding), the same approach taken by
adaptor-signature swaps on Bitcoin Taproot.

### The re-randomization wrinkle

RedDSA's per-spend re-randomization makes shielded adaptor sigs harder than the
BIP-340 case. Every Sapling / Orchard spend signs with a freshly randomized
validating key:

```
rk = ak + [α]·G          (α sampled by the spender, per spend)
spendAuthSig = RedDSA.Sign(rsk, sighash)    where rsk = ask + α
```

A standard Schnorr adaptor pre-signature `(R', s')` works only if both
pre-signer and completer agree on `α` ahead of time, since `rk` enters the
sighash and must match between pre-sig and completion. The completer also needs
`α` to verify the standard Schnorr equation against the on-chain `rk`. This is
not a fundamental obstacle (the FROST/RedDSA production stack already exposes a
`RandomizePrivate(α, ask)` API for working with randomized keys), but no
published paper or code specifically constructs an adaptor pre-signature against
Sapling or Orchard spend authorization.

### Cross-curve binding

A swap between LEZ (secp256k1) and Zcash shielded (Jubjub or Pallas) needs the
adaptor witness scalar to be valid on both curves. Group orders differ. The
standard mitigation, used by COMIT for XMR-BTC swaps, is to attach a cross-curve
DLEQ proof binding the same scalar across both groups. For Pallas the
pairing-friendly properties differ from Ed25519, so the COMIT cross-curve DLEQ
implementation is not a drop-in port; additional cryptographic work would be
required.

### Best generic framework available

The closest published framework is
[Universal Atomic Swaps](https://eprint.iacr.org/2021/1612) (Thyagarajan,
Malavolta, Moreno-Sánchez, IEEE S&P 2022), which proves a generic protocol over
"any chain that can verify signatures on transactions" using adaptor signatures
plus time-lock puzzles, with no custom scripting required. The paper does not
provide a Zcash-specific instantiation. Adapting it to RedJubjub / RedPallas
with re-randomization is the open work.

## What Maya Protocol is, and isn't

[Maya Protocol](https://docs.mayaprotocol.com/mayachain-dev-docs/protocol-development/chain-clients/zcash)
went live with native ZEC support in May 2025 and is the most visible production
"Zcash swap" path. It is worth being precise about the trust model: Maya is
**not** an atomic swap. It is a vault network with threshold-signed outbound
transactions. Funds are custodied by the TSS quorum of Maya node operators
between the inbound and outbound legs. Zcash side: transparent addresses
(`t1...`) only, ECDSA, OP_RETURN memos.

Both atomic swaps and TSS vaults are valid product categories: vaults give
better UX and broader asset coverage; atomic swaps give stronger trust
guarantees. They should not be conflated when scoping a non-custodial swap RFP.

## Feasibility matrix

| Pool              | Construction                       | Required Zcash primitive                                                                                   | Trust model          | Production status (2026-05)                                         |
| ----------------- | ---------------------------------- | ---------------------------------------------------------------------------------------------------------- | -------------------- | ------------------------------------------------------------------- |
| Transparent       | BIP-199 HTLC                       | OP_IF/ELSE, OP_SHA256/HASH160, OP_CLTV, OP_CHECKSIG, P2SH                                                  | Non-custodial atomic | Reference (ZBXCAT 2017); active in Zwap 2025                        |
| Transparent       | ECDSA adaptor signature (Fournier) | secp256k1 ECDSA + non-malleable `s`, refund via OP_CLTV                                                    | Non-custodial atomic | None Zcash-specific; well-tested for Bitcoin                        |
| Transparent       | ECDH key aggregation (Zwap)        | ECDSA + hash-preimage spend + OP_CLTV refund                                                               | Non-custodial atomic | Zwap alpha (Zcash + Ethereum)                                       |
| Transparent       | Vault / TSS routing                | OP_RETURN memo + ECDSA                                                                                     | Custodial-via-TSS    | Maya Protocol live since 2025-05                                    |
| Sapling           | RedJubjub adaptor signature        | Pre-sig on randomized `rk` over Jubjub; α-aware adaptor protocol; nExpiryHeight as broadcast-deadline only | Non-custodial atomic | None                                                                |
| Orchard           | RedPallas adaptor signature        | Same as Sapling on Pallas curve                                                                            | Non-custodial atomic | None                                                                |
| Sapling / Orchard | HTLC                               | n/a                                                                                                        | n/a                  | Not implementable: no on-chain scripting                            |
| Sapling / Orchard | Vault / TSS                        | RedDSA-FROST (production-grade)                                                                            | Custodial-via-TSS    | Possible in principle; no production deployment to a shielded vault |

## Recommendation for LEZ ↔ ZEC scope

- **Phase 1 (deliverable):** LEZ ↔ Zcash transparent via the same pattern used
  by the ETH-LEZ HTLC reference, with adaptor-signature variant as a stretch
  goal for unlinkability.
- **Phase 2 (research, separate work):** LEZ ↔ Zcash shielded using a
  Universal-Atomic-Swaps-style construction adapted to RedJubjub / RedPallas.
  Treat this as a research deliverable, not a delivery deliverable. Funding and
  timeline are open questions; the upstream cryptographic state of the art does
  not yet provide a drop-in design.

## References

### Zcash protocol

- [BIP-199: Hashed Time-Locked Contract transactions](https://github.com/bitcoin/bips/blob/master/bip-0199.mediawiki)
- [ZIP-48: Transparent Multisig Wallets](https://zips.z.cash/zip-0048): confirms
  Zcash does not use SegWit `script_type` values, hence no Taproot on the
  transparent pool
- [ZIP-202: Version 3 Transaction Format for Overwinter](https://zips.z.cash/zip-0202)
- [ZIP-203: Transaction Expiry](https://zips.z.cash/zip-0203)
- [ZIP-224: Orchard Shielded Protocol](https://zips.z.cash/zip-0224)
- [ZIP-304: Sapling Address Signatures](https://zips.z.cash/zip-0304)
- [Zcash Protocol Specification (NU6.1)](https://zips.z.cash/protocol/protocol.pdf)

### Cryptographic libraries

- [ZcashFoundation/redjubjub](https://github.com/ZcashFoundation/redjubjub):
  RedJubjub (Sapling)
- [ZcashFoundation/reddsa](https://github.com/ZcashFoundation/reddsa): generic
  RedDSA (Sapling + Orchard)
- [ZcashFoundation/frost-tools](https://github.com/ZcashFoundation/frost-tools):
  FROST threshold signing for Zcash, including rerandomized variant
- [zcash/orchard](https://github.com/zcash/orchard): Orchard reference
  implementation

### Atomic swap research and prior art

- [Universal Atomic Swaps (Thyagarajan, Malavolta, Moreno-Sánchez, 2022)](https://eprint.iacr.org/2021/1612):
  generic scriptless atomic swap framework
- [Multi-Party, Multi-Blockchain Atomic Swap Protocol with Universal Adaptor Secret (2024)](https://arxiv.org/pdf/2406.16822)
- [Zswap: zk-SNARK Based Non-Interactive Multi-Asset Swaps](https://eprint.iacr.org/2022/1002.pdf)
- [Zwap forum thread](https://forum.zcashcommunity.com/t/zwap-unlinkable-cross-chain-atomic-swaps/55104):
  Zcash transparent + Ethereum, ECDH-based
- [Maya Protocol Zcash docs](https://docs.mayaprotocol.com/mayachain-dev-docs/protocol-development/chain-clients/zcash):
  vault/TSS model (for contrast)
- [comit-network/xmr-btc-swap](https://github.com/comit-network/xmr-btc-swap):
  closest production analogue for adaptor swaps to a non-scriptable chain
