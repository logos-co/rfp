---
id: RFP-021
title: Cross-Chain Privacy DEX (Federated Middle Layer)
tier: XL
funding: $TBD
status: dropped
category: Applications & Integrations
---

# RFP-021 — Cross-Chain Privacy DEX (Federated Middle Layer) [DROPPED]

> **This RFP has been dropped.** It is retained as a tombstone for the
> historical record; the full original design is in git history.

## Why it was dropped

RFP-021 proposed a federated-signer middle chain that custodies external assets
(BTC, ETH, XMR) in a threshold-signature vault held by the LEZ validator set,
following the Thorchain/Serai pattern. That model reintroduces exactly the trust
assumption the cross-chain bundle is designed to avoid: a federated signer set
that holds user funds, can be coerced or captured, carries a non-zero historical
loss rate, and (for XMR) leaks deposit history to signers via view-key sharing.

The bundle's direction is the most trustless path available. The preferred stack
is non-custodial:

- **RFP-024** — CDP-backed synthetics (no custody of the tracked asset at all).
- **RFP-026** — atomic-swap redemption of synthetics to the real asset, built on
  RFP-003, with the protocol holding none of the real asset.

The neutral survey of the federated-middle-chain model and its adoption and
failure record is preserved in
[appendix/cross-chain-trust-model-contrast.md](../appendix/cross-chain-trust-model-contrast.md).
The full original RFP-021 text (overview, flow, pros/cons, risks, references) is
available in this repository's git history.
