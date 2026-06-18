---
id: RFP-025
title: Synthetic XMR (sXMR) — Real XMR in Threshold-Signer Multisig
tier: XL
funding: $TBD
status: dropped
category: Applications & Integrations
---

# RFP-025 — Synthetic XMR Backed by Real XMR in Trusted Multisig [DROPPED]

> **This RFP has been dropped.** It is retained as a tombstone for the
> historical record; the full original design is in git history.

## Why it was dropped

RFP-025 proposed an sXMR token backed 1:1 by real XMR held in a threshold-signer
multisig on Monero. That is the same federated-custody trust class as the
dropped RFP-021: a bonded signer set custodies real assets, carries custody and
liveness risk, is a censorship and coercion chokepoint, and leaks Monero deposit
history to signers via view-key sharing. It reintroduces precisely the trust
assumptions the bundle's preferred CDP path deliberately avoids.

The bundle's direction is the most trustless path available. For synthetic XMR
(and any other asset) the preferred design is non-custodial:

- **RFP-024** — CDP-backed `sASSET`, a debt instrument against stable
  collateral. The protocol never touches the tracked asset.
- **RFP-026** — optional atomic-swap redemption to the real asset on its home
  chain, again with no protocol-side custody.

The deployed prior art for the custody-backed approach (sBTC, Secret Monero
Bridge) and the federated-signer trust analysis are preserved in
[appendix/synthetics-design-space.md](../appendix/synthetics-design-space.md)
and
[appendix/cross-chain-trust-model-contrast.md](../appendix/cross-chain-trust-model-contrast.md).
The full original RFP-025 text is available in this repository's git history.
