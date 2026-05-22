# Synthetics Design Space

Survey appendix on deployed synthetic-asset designs and what each protocol commits to (and trusts) at the redemption boundary. Pure survey of deployed and historical synthetics; no design choices made here.

A "synthetic" in this appendix means a token on chain A that tracks the price of an asset on chain B (or an off-chain reference) without being a direct wrapping of that asset. Pure wrapped tokens (one-to-one custody-backed representations) are out of scope.

## Taxonomy

Deployed synthetic designs differ on two axes:

1. **What backs the synthetic.** Stable collateral (over-collateralised CDP), the underlying asset itself (custody-backed), or no backing at all (pure pegged float).
2. **What redemption means.** Convertible to the underlying asset at oracle price, convertible to the collateral at oracle price, or not redeemable but freely tradable.

The deployed examples populate three corners of this space:

### Oracle-priced over-collateralised synthetics

Synthetics minted against stable collateral, valued by oracle, settled in collateral rather than the tracked asset.

- **Haven Protocol (XHV / xUSD / other xAssets).** Monero-forked L1 launched 2018. Users lock XHV to mint xAssets at oracle price; conversion between xAssets burns the source and mints the destination. Over-collateralised. xUSD has depegged multiple times (notably 2022-2023) due to low liquidity, oracle delays, and market stress. As of 2026-05-22, XHV market cap is approximately $5.5M and xUSD supply approximately $1.2M; daily transactions approximately 50-100. Sources: [Haven Protocol Documentation](https://docs.havenprotocol.org) (accessed 2026-05-22); [CoinGecko: Haven (XHV)](https://www.coingecko.com/en/coins/haven) (accessed 2026-05-22); [Haven Explorer](https://explorer.havenprotocol.org) (accessed 2026-05-22). Note: prior PR-57 appendix text stated Haven shut down in December 2024; this claim has been flagged for vault verification as it does not match the current explorer/market-cap data above.
- **Synthetix.** SNX stakers mint sUSD and other synthetic equivalents against SNX collateral, debt pooled across all stakers. Used as a reference for "oracle-priced over-collateralised synth" rather than for a specific Monero-relevant property here. Note: prior PR-57 appendix text cited SIP-302 as the canonical reference for V3 collateral and snxUSD minting; this citation has been flagged for vault verification.

What you trust in this design family:

- The oracle. A compromised oracle is an infinite-mint vulnerability.
- The collateralisation ratio holds under price-of-collateral stress (otherwise the protocol becomes insolvent against outstanding synthetics).
- The liquidation mechanism executes faster than collateral price collapse.

Privacy properties (Haven specifically): Monero-style ring signatures and stealth addresses protect transfers of xAssets, but Haven has no smart-contract layer, so xAssets cannot be used in DeFi outside Haven itself.

### Redeem-to-underlying with custody

Synthetics minted against the underlying asset held by a custodial bridge, valued by direct redemption.

- **sBTC (Stacks).** Synthetic BTC on Stacks, redeemable to native BTC. The custody arrangement (signer set, threshold scheme, redemption SLA) has been flagged for vault verification before specific claims are made here. Canonical docs page: [docs.stacks.co/learn/sbtc](https://docs.stacks.co/learn/sbtc). This claim and its specific trust-shape characterisation have been added to the pending-research request for PR #57.
- **Secret Monero Bridge** (Secret Network). Mainnet launched August 2021. Multi-signature Monero wallet operated by consensus-node signers (MSCNOs) communicating over I2P; users deposit XMR, receive sXMR (a SNIP-20 token on Secret Network) usable in Secret DeFi (e.g. SecretSwap). Source: [Bitcoin Insider: Secret Monero Bridge mainnet launch](https://www.bitcoininsider.org/article/123189/secret-network-announces-launch-secret-monero-bridge-mainnet) (accessed 2026-05-22); [github.com/maxkoda-cpu/Secret-Monero-Bridge](https://github.com/maxkoda-cpu/Secret-Monero-Bridge) (accessed 2026-05-22).

What you trust in this design family:

- The custodian (signer set, multisig threshold).
- The cryptographic primitive combining signer contributions (TSS, multisig, FROST).
- The off-chain coordination among signers does not produce a censorship chokepoint or collusion path.

This trust shape is structurally identical to a federated-signer middle chain (see the [trust-model contrast appendix](./cross-chain-trust-model-contrast.md)); the synthetic-token wrapper is cosmetic relative to the trust assumption.

### Redeem-to-underlying without custody

This design corner has no fully deployed example in the published landscape as of 2026-05. The Secret Monero Bridge above is custodial; the BTC-XMR atomic-swap projects (COMIT, Farcaster) execute peer-to-peer swaps but do not issue a synthetic token at all. A redeem-to-underlying synthetic where redemption itself is peer-to-peer-atomic and the protocol never custodies the underlying is a published design space but not a published implementation.

## Privacy-coin specific constraints

Synthetic designs that target Monero (or other privacy coins) inherit constraints from the underlying:

- **No SPV-style proof of state.** Monero has no proof primitive that can demonstrate "address Y received amount X" to a third party without view-key disclosure. Bilateral proofs (`check_tx_proof` family) exist but lifting them to public on-chain state is equivalent to view-key disclosure for the swap output, which deanonymises the swap. Sources: [Monero, Zero to Monero 2.0 §Payment Proofs](https://www.getmonero.org/library/Zero-to-Monero-2-0-0.pdf) (accessed 2026-05-21).
- **Custody-side privacy leak.** Threshold custody of XMR requires view-key sharing among signers. Honest-but-curious signers learn the deposit history of the synthetic-side mint and burn flow. This is a property of any TSS custody of XMR, including the most advanced production design (Serai's FROSTLASS over CLSAG); see the [trust-model contrast appendix](./cross-chain-trust-model-contrast.md).
- **No native smart contracts on Monero.** Designs that want programmability over a Monero-backed synthetic must run that programmability on a separate chain; the synthetic and its DeFi context are necessarily off Monero.
- **Future cryptographic primitives.** The FCMP++ (full-chain membership and metadata-private proofs) research direction may unlock new non-disclosing proof variants; it is pre-production. Source: [Monero, FCMP++ announcement (2024-04-27)](https://www.getmonero.org/2024/04/27/fcmps.html) (accessed 2026-05-21).

## Negative example: Secret Monero Bridge

The Secret Monero Bridge is the closest deployed prior art for bridging XMR into a programmable privacy ecosystem. The documented launch reception is instructive:

1. **Trusted operator model without economic security.** The bridge used a multi-signature Monero wallet controlled by consensus-node operators. There was no bonded collateral, no slashing, and no cryptographic proof of correctness. Security relied on social trust plus I2P anonymisation of the operator network. Source: [`maxkoda-cpu/Secret-Monero-Bridge`](https://github.com/maxkoda-cpu/Secret-Monero-Bridge) (accessed 2026-05-22).
2. **Privacy-hostile onboarding UX.** The mainnet release required users to provide an email address and use Discord for support tickets; the Monero community viewed this as antithetical to privacy principles and widely refused to use the bridge. Source: vault note `projects/secret-network.md` documents this controversy; the primary documentation is Bitcoin Insider's launch coverage and contemporary Monero-community forum discussions, both available via the cited launch announcement.
3. **Unclear current status.** As of 2025-2026 the GitHub repository has limited recent activity and community forum posts question whether the bridge is still maintained. Source: [`maxkoda-cpu/Secret-Monero-Bridge`](https://github.com/maxkoda-cpu/Secret-Monero-Bridge) (accessed 2026-05-22).

These are properties of the bridge's design and reception; whether any specific lesson applies to a future synthetic depends on the future synthetic's own choices and is left to the relevant RFPs.

## How minting works in deployed synthetic protocols

Common patterns across the deployed examples:

- **Lock collateral, mint synthetic at oracle price** (Haven xAssets, Synthetix sUSD). The collateral is the protocol's native token (or a basket); the synthetic is valued by oracle at mint. Over-collateralisation absorbs price moves.
- **Deposit underlying, mint synthetic 1:1 minus fees** (Secret Monero Bridge sXMR, sBTC on Stacks). The synthetic is a wrapped representation; mint is custodial, burn unlocks the underlying.
- **No deployed example exists** for a synthetic that mints against stable collateral *and* redeems to the underlying via peer-to-peer atomic swap. This is the design space targeted by the bundle's sXMR RFPs.

## References

- [Haven Protocol Documentation](https://docs.havenprotocol.org) (accessed 2026-05-22)
- [Haven Protocol Whitepaper](https://havenprotocol.org/whitepaper/) (accessed 2026-05-22)
- [CoinGecko: Haven (XHV)](https://www.coingecko.com/en/coins/haven) (accessed 2026-05-22)
- [Haven Explorer](https://explorer.havenprotocol.org) (accessed 2026-05-22)
- [Secret Network documentation](https://docs.scrt.network) (accessed 2026-05-22)
- [Bitcoin Insider: Secret Monero Bridge mainnet launch](https://www.bitcoininsider.org/article/123189/secret-network-announces-launch-secret-monero-bridge-mainnet) (accessed 2026-05-22)
- [`maxkoda-cpu/Secret-Monero-Bridge`](https://github.com/maxkoda-cpu/Secret-Monero-Bridge) (accessed 2026-05-22)
- [Secret Monero Bridge Devpost](https://devpost.com/software/secret-monero-bridge) (accessed 2026-05-22)
- [Monero, Zero to Monero 2.0 (whitepaper)](https://www.getmonero.org/library/Zero-to-Monero-2-0-0.pdf) (accessed 2026-05-21)
- [Monero, FCMP++ announcement (2024-04-27)](https://www.getmonero.org/2024/04/27/fcmps.html) (accessed 2026-05-21)
- [docs.stacks.co/learn/sbtc](https://docs.stacks.co/learn/sbtc) (referenced; specific trust-shape claims pending vault verification)
