# Bitcoin and Monero Adaptor-Signature Swap Primitives

What an adaptor-signature atomic swap is, why RFP-003 mandates adaptor
signatures rather than HTLCs for the Bitcoin pair, why Monero permits no other
construction, and the prior art each claim rests on.

## Adaptor signatures in one page

An adaptor signature (also called a pre-signature or verifiably encrypted
signature) is a signature that is deliberately incomplete: it verifies against a
message and public key only once a secret scalar `t` (the adaptor witness, with
public point `T = t·G`) is folded in.

The swap-relevant properties, formalised by
[Aumayr et al.](https://eprint.iacr.org/2020/476) as pre-signature adaptability,
witness extractability, and aEUF-CMA security:

1. **Adaptability**: anyone holding the pre-signature `(R, s')` and the witness
   `t` can complete it to a valid signature `s = s' + t`.
2. **Extractability**: anyone holding the pre-signature who later sees the
   completed signature on-chain can recover the witness as `t = s − s'`.

Property 2 is what makes the swap atomic: claiming funds on one chain requires
publishing `s`, and publishing `s` unavoidably reveals `t` to the counterparty,
who uses it to claim (or co-sign) on the other chain. The hash preimage of an
HTLC is replaced by an off-chain scalar reveal, so no swap-identifying data
appears in any script. The construction originates in Andrew Poelstra's
[Scriptless Scripts](https://github.com/apoelstra/scriptless-scripts) work; a
production-grade ECDSA adaptor specification with test vectors exists in the
[DLC specs](https://github.com/discreetlogcontracts/dlcspecs/blob/master/ECDSA-adaptor.md).
For chains that only verify ECDSA, Lloyd Fournier's
[one-time verifiably encrypted signatures](https://github.com/LLFourn/one-time-VES)
construction provides the ECDSA variant, carrying a DLEQ proof that the
encryption is well-formed.

## Bitcoin: why not HTLCs

Two distinct claims in RFP-003 need support here.

**HTLCs link the two swap legs.** A BIP-199 style HTLC gates spending on
revealing a preimage whose hash is embedded in the script. Both chains embed the
same hash, and the claim transactions on both chains reveal the same preimage,
so any observer can pair the two legs across chains and identify the
transactions as a swap. This linkability is the stated motivation of the
scriptless-scripts programme and of the
[h4sh3d Bitcoin-Monero swap paper](https://eprint.iacr.org/2020/1126.pdf), which
both replace the preimage with an adaptor witness precisely to remove the
cross-chain identifier.

**Taproot key-path spends are indistinguishable from ordinary payments.** Under
[BIP-341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki), a
key-path spend publishes only a single BIP-340 Schnorr signature; no script is
revealed at all. A 2-of-2 swap escrow becomes an aggregated key (for example via
[MuSig2, BIP-327](https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki))
whose key-path spend looks identical to any single-signer Taproot payment. This
indistinguishability applies to the **cooperative (claim) path**: both parties
co-sign the key-path spend and nothing about the swap is revealed on-chain.

### Two ways to build the refund branch

[Taproot (BIP-341)](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki)
gives a single output two independent spending conditions: a key-path spend
against the output key, and a script-path spend that reveals one leaf of a
committed script tree (BIP-341, "Script validation rules"). The refund branch
can be built with either. The cooperative claim path is the key-path spend in
both cases; the choice is only about how a party reclaims funds when the
counterparty stops responding.

**Option A: pre-signed timelocked refund (key-path).** At setup, the parties
co-sign a refund transaction carrying an absolute
([`nLockTime` / BIP-65 `OP_CHECKLOCKTIMEVERIFY`](https://github.com/bitcoin/bips/blob/master/bip-0065.mediawiki))
or relative
([`nSequence` / BIP-68](https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki))
timelock and hold it off-chain, broadcasting it only if the swap stalls. The
refund spend is itself an ordinary key-path spend, so it is indistinguishable
from any other payment. This is the technique used by Lightning, whose
commitment and HTLC outputs are spent by transactions pre-signed during channel
operation rather than by revealing on-chain script branches
([BOLT 3, "Commitment Transaction"](https://github.com/lightning/bolts/blob/master/03-transactions.md)),
and by the swap protocols cited below. The cost is moved off-chain: the refund
is enforced not by consensus but by each party correctly constructing and
*retaining* the pre-signed transaction, and by the cross-chain timelock ordering
(see "Timelock ordering" below) holding.

**Option B: script-path refund (tapleaf).** A timelocked refund clause (for
example `<delay> OP_CHECKSEQUENCEVERIFY OP_DROP <refunder_key> OP_CHECKSIG`,
using the relative timelock opcode from
[BIP-112](https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki)) is
committed as a leaf in the Taproot tree. The refunder spends that leaf after the
delay, with no pre-agreed transaction to construct or store. The refund is
enforced directly by Bitcoin consensus. The cost is on-chain: a script-path
spend must reveal the leaf script and a control block proving its inclusion in
the tree
([BIP-341, "Script validation rules"](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki)),
so the refund spend is publicly identifiable as the refund branch of a swap
escrow. The cooperative path is unaffected and stays indistinguishable.

The trade-off is **privacy-on-refund vs. fragility / state burden**:

|                           | Pre-signed (A)                                                                                           | Script-path (B)                                |
| ------------------------- | -------------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| Refund spend on-chain     | Indistinguishable key-path spend                                                                         | Reveals tapleaf; identifiable as a swap refund |
| Refund enforcement        | Off-chain: holds only if the pre-signed tx was built correctly and retained, and timelock ordering holds | On-chain: enforced by consensus                |
| Failure modes added       | Lost/corrupted refund tx, malformed timelock, fee-rate set at setup time may be unbroadcastable later    | None beyond standard CSV semantics             |
| Implementation complexity | Higher: refund-tx lifecycle, persistence, fee management                                                 | Lower: one extra tapleaf, standard CSV         |

A privacy leak on the refund branch is occasional by nature (a refund only
happens when a swap fails), whereas the fragility of Option A is borne on every
swap. Which side that balance favours is an engineering question worth measuring
rather than assuming; RFP-003 asks applicants to do so.

## Monero: no scripting, key-share transfer

Monero has no script system: outputs are spendable solely by producing a ring
signature with the output's spend key. There is no opcode for hashes, timelocks,
or branches, so neither HTLCs nor on-chain multisig escrows in the Bitcoin sense
are expressible. The [h4sh3d protocol](https://eprint.iacr.org/2020/1126.pdf)
(independently developed and shipped by COMIT) achieves atomicity anyway:

1. The swap funds are sent to a Monero address whose spend key is the sum of two
   shares, one held by each party.
2. The Bitcoin side is locked under an adaptor signature whose witness *is* the
   counterparty's Monero key share.
3. Claiming the Bitcoin leg publishes the completed signature, which leaks the
   witness (extractability, above), handing the counterparty the full Monero
   spend key.

Because the two key shares live on different curves (secp256k1 on the Bitcoin
side, Ed25519 on the Monero side), the protocol requires a **cross-curve DLEQ
proof** that the same scalar underlies both public points. COMIT published a
dedicated library for this:
[cross-curve-dleq](https://github.com/comit-network/cross-curve-dleq). Refunds
on the Monero side are likewise handled by pre-agreed transactions and timelock
asymmetry, since Monero cannot express a refund branch on-chain.

## Timelock ordering across chains

All of the cited protocols share the same refund-safety argument: the refund
deadline on the counterparty chain must strictly succeed the refund deadline on
the home chain, with a margin covering worst-case confirmation latency, so that
one party can never claim on one chain after the other party has already
refunded on the other. RFP-003 carries this requirement for every pair (see the
Zcash functional requirement and LEZ open question 2). The argument and
parameter discussion appear in section 4 of the
[h4sh3d paper](https://eprint.iacr.org/2020/1126.pdf).

## Production status

| Project                                                                     | Pairs                           | Construction                                                                                | Status                                                                                        |
| --------------------------------------------------------------------------- | ------------------------------- | ------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| [comit-network/xmr-btc-swap](https://github.com/comit-network/xmr-btc-swap) | BTC ↔ XMR                       | Schnorr adaptor + cross-curve DLEQ, key-share transfer                                      | Production; mainnet swaps since 2021, GUI via [UnstoppableSwap](https://unstoppableswap.net/) |
| [Farcaster](https://github.com/farcaster-project)                           | BTC ↔ XMR                       | Same family, Monero CCS-funded reimplementation                                             | Alpha / development                                                                           |
| [BasicSwap (Particl)](https://github.com/basicswap/basicswap)               | BTC, XMR, and other UTXO chains | Adaptor-signature protocol for the XMR pairs                                                | Live DEX                                                                                      |
| Lightning Network                                                           | BTC                             | Pre-signed timelocked refund transactions (the off-chain refund technique referenced above) | Production since 2018                                                                         |

## Relevance to RFP-003

RFP-003 requires the Bitcoin pair to use BIP-340 adaptor signatures with Taproot
key-path spends (no custom scripts), and the Monero pair to use the h4sh3d/COMIT
key-share construction with cross-curve DLEQ proofs. Both constructions are
published, formally analysed, and have shipped in production software listed
above; the engineering risk for those pairs is integration work, not novel
cryptography. The Zcash transparent pair is analysed separately in
[Appendix: Zcash Atomic Swap Primitives](zcash-atomic-swap-primitives.md).

## References

- [Aumayr, Ersoy, Erwig, Faust, Hostáková, Maffei, Moreno-Sánchez, Riahi, *Generalized Channels from Limited Blockchain Scripts and Adaptor Signatures*](https://eprint.iacr.org/2020/476):
  formal adaptor-signature security definitions
- [h4sh3d, *Bitcoin-Monero Cross-chain Atomic Swap*](https://eprint.iacr.org/2020/1126.pdf):
  the BTC ↔ XMR protocol mandated for the Monero pair
- [Andrew Poelstra, Scriptless Scripts](https://github.com/apoelstra/scriptless-scripts):
  origin of the adaptor-signature swap idea
- [DLC specs, ECDSA Adaptor Signatures](https://github.com/discreetlogcontracts/dlcspecs/blob/master/ECDSA-adaptor.md):
  ECDSA adaptor spec with test vectors
- [Lloyd Fournier, one-time VES](https://github.com/LLFourn/one-time-VES): ECDSA
  adaptor construction with DLEQ proof
- [BIP-340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki),
  [BIP-341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki),
  [BIP-327 (MuSig2)](https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki)
- [BIP-199: Hashed Time-Locked Contract transactions](https://github.com/bitcoin/bips/blob/master/bip-0199.mediawiki):
  the HTLC baseline being avoided on the Bitcoin pair
- [BIP-65 (`OP_CHECKLOCKTIMEVERIFY`)](https://github.com/bitcoin/bips/blob/master/bip-0065.mediawiki),
  [BIP-68 (relative lock-time via `nSequence`)](https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki),
  [BIP-112 (`OP_CHECKSEQUENCEVERIFY`)](https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki):
  the timelock primitives behind both refund constructions
- [BOLT 3: Bitcoin Transaction and Script Formats](https://github.com/lightning/bolts/blob/master/03-transactions.md):
  Lightning's pre-signed commitment/HTLC transactions, prior art for the
  pre-signed refund construction
- [comit-network/xmr-btc-swap](https://github.com/comit-network/xmr-btc-swap)
  and
  [comit-network/cross-curve-dleq](https://github.com/comit-network/cross-curve-dleq)
