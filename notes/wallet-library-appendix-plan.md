# Plan: a third appendix on wallet libraries

## Why separate (user's call, and I agree)
The existing survey is organised around NODE API surfaces: functions,
transports, and the SDKs that wrap them. Wallet libraries are a different
category. rust-bitcoin, BDK, ethers.js, bitcoinjs-lib do key derivation,
address generation, and local signing, much of which never touches a node.
Filing them under "RPC and transport types" would misrepresent both.

Confirmed the gap is real:
- ZERO mentions of BIP-32, BIP-39, BIP-44, mnemonic, derivation path, xpub, or
  HD wallet anywhere in the existing appendix.
- NONE of the 34 catalogued functions covers key or address derivation.
- ethers.js and viem appear exactly once each, and only as node-client SDKs in
  a deprecation note. rust-bitcoin and bitcoinjs-lib appear zero times in that
  framing.

## The user's key correction, which is the spine of the section
Removing geth's `personal` namespace did NOT remove the capability, it
RELOCATED it:

| Before (node holds key) | After (library holds key) |
| --- | --- |
| `personal_sendTransaction` | library signs locally, submits via `eth_sendRawTransaction` |
| `personal_sign` | library-side `signMessage` |

So the node API did not shrink in what it ENABLES. It shed the custody half and
kept the transport half. The wallet library consumes the surviving `eth`
namespace on the integrator's behalf.

VERIFIED myself: the four methods an Ethereum wallet library needs all exist in
the execution-apis spec:
- eth_getTransactionCount (state.yaml) - nonce
- eth_chainId (client.yaml) - replay protection
- eth_estimateGas (execute.yaml) - fee
- eth_sendRawTransaction (submit.yaml) - submit

VERIFIED: all eight surveyed chains have a signed-bytes submission endpoint
(from the appendix's own fact-checked 1.21 table), which is what makes the
sign-locally-submit-raw pattern universally possible.

## The asymmetry the user identified, verified
- Bitcoin Core STILL ships createwallet, getnewaddress,
  signrawtransactionwithwallet, listunspent (developer.bitcoin.org, HTTP 200).
- geth REMOVED the personal namespace, PR #30704 merged 2024-10-31 (verified
  via GitHub API).

So: same capability, two different homes. An integrator on Bitcoin can use the
node; on Ethereum they must bring a library.

## Structure to write
1. Why this is a separate concern from the node API
2. Where wallet functionality lives, per chain (node / library / both)
3. The relocation: what personal_* became, with the mapping table
4. What an exchange actually needs: HD derivation, watch-only deposit
   addresses, offline signing. Per-chain support.
5. Library inventory: name, repo, first-party or community, node client vs
   wallet library vs both
6. The BDK pattern: one Rust core, FFI bindings to six language targets.
   Directly relevant to issue #219.
7. Logos: what exists today. wallet_ffi is wallet-shaped already (63 fns,
   accounts, keys, transfers). L1 has node-side signing routes that the
   survey already flags as the pattern others retreated from.

## Open question for the user once drafted
Whether to also cover the custody/HSM angle (hardware signing, air-gapped) or
keep it to libraries only. The research brief asks for it, so I will include a
short subsection and can cut it.
