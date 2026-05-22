# Cross-Chain Trust Model Contrast

Survey appendix on the two architectural camps that deployed cross-chain swap protocols fall into, the trust assumptions each camp asks of users, and the adoption record of each. Pure survey of deployed and pre-mainnet protocols; no design choices made here.

For the underlying cryptographic mechanics of atomic swaps see the [atomic-swaps primer](./atomic-swaps-primer.md).

## Two architectural camps

Every deployed cross-chain swap design today collapses to one of two trust models: a federated-signer middle chain that custodies external assets, or a peer-to-peer atomic swap that custodies nothing.

### Federated-signer middle chain

A purpose-built chain whose validator set custodies external assets via a threshold signature scheme and runs swap matching natively.

Representative protocols:

- **Thorchain.** Cosmos SDK / CometBFT L1 launched 2021. Validators run an off-chain observer-signer daemon (Bifrost) and co-sign outbound transactions from per-asset TSS vaults using a GG20 ECDSA scheme (fork of Binance `tss-lib` upgraded from GG18 to GG20). Native swap matching via slip-based continuous liquidity pools (CLPs) with RUNE as universal pairing asset. ~103 active nodes (cap 120), minimum bond 400,020 RUNE per node. Sources: [DefiLlama Thorchain DEX dashboard](https://defillama.com/protocol/thorchain-dex) (accessed 2026-05-19); [State of the Network Feb 2026](https://blog.thorchain.org/state-of-the-network-february-2026/) (accessed 2026-05-19); [Thorchain Bifrost TSS docs](https://dev.thorchain.org/bifrost/tss.html) (accessed 2026-05-19).
- **Serai.** Substrate-based L1, pre-mainnet as of 2026-05 (Substrate blockchain audit complete 2026-04-15, internal testnet pending). Validator set forms per-coin threshold multisigs using FROST (per-curve, including FROSTLASS over CLSAG for Monero). Native AMM with constant-product pools. Validator cap 600 at launch, staking per-network. Sources: [serai.exchange](https://serai.exchange/) (accessed 2026-05-19); [Audit of Serai's Substrate Blockchain (2026-04-15)](https://serai.exchange/2026/04/15/serai-blockchain-audited.html) (accessed 2026-05-19); [Announcing monero-oxide (2025-09-09)](https://serai.exchange/2025/09/09/monero-serai-oxide.html) (accessed 2026-05-19).
- **Maya, Chainflip.** Cousin protocols of Thorchain, same architectural pattern.
- **Wormhole.** Distinct sub-pattern: not a swap chain but a guardian-attestation messaging protocol. 19 guardian operators sign 13-of-19 multisignature attestations (Verifiable Action Approvals) to relay messages across ~40 connected chains. Token movement uses lock-and-mint or burn-and-mint; cross-asset swaps happen on destination-chain DEXes or solver networks rather than inside Wormhole. Same trust shape (committee of known signers) but no protocol-owned liquidity, no native middle chain. Source: [Wormhole Guardians docs](https://wormhole.com/docs/protocol/infrastructure/guardians/) (accessed 2026-05-19).

What the user trusts:

1. A threshold of the validator set will not collude to spend from the vault. The economic backstop in Thorchain is a 2:1 invariant (bonded RUNE should exceed twice the pooled liquidity); see [Thorchain economic model](https://docs.thorchain.org/technical-documentation/technical-deep-dive/economic-model.md).
2. The cryptographic primitive used to combine signer contributions is sound. Not a free assumption. On 2026-05-15 a GG20 weakness (TSSHOCK-class: malformed proofs allowing key reconstruction by a colluding signer over many rounds) drained approximately $10.8M from a Thorchain Asgard vault. Sources: [Crypto Times, 2026-05-17](https://www.cryptotimes.io/2026/05/17/10-8-million-drained-inside-the-thorchain-exploit-that-froze-cross-chain-defi-for-13-hours/) (accessed 2026-05-19); [AMBCrypto](https://ambcrypto.com/thorchain-exploit-raises-fresh-concerns-over-mpc-wallet-security/) (accessed 2026-05-19).
3. The signer-to-chain integration is correct. The 2021 Thorchain ETH router exploits (two incidents, combined ~$16M) were not TSS failures: the Bifrost interface trusted smart-contract emitted events from a wrapping attacker contract, so the TSS layer functioned as designed while signing transactions that drained the vault. Source: [Thorchain post-mortem](https://medium.com/thorchain/post-mortem-eth-router-exploits-1-2-and-premature-return-to-trading-incident-2908928c5fb) (accessed 2026-05-19).
4. Implementations on every external chain are correct. The 2022-02-02 Wormhole Solana-side `load_instruction_at` bug let an attacker forge a VAA and mint 120k wETH unbacked ($326M). Source: [Halborn: Wormhole Hack February 2022](https://www.halborn.com/blog/post/explained-the-wormhole-hack-february-2022) (accessed 2026-05-19).

For Monero specifically: any threshold-signer custody of XMR is necessarily view-key-shared. Serai's FROSTLASS over CLSAG is the most advanced production-grade design but the validator set still observes which Monero outputs are committed to the swap. The privacy property is "honest-but-curious validators learn the protocol-side deposit history", not "validators learn nothing".

Pros of the federated-signer pattern:

- AMM-style liquidity. A single ordered state machine maintains pool invariants and serves all-comers without per-trade matching.
- One-step user experience: deposit with memo, await outbound. No counterparty interactivity, no refund flows, no online-availability requirement past broadcast.
- Sub-block-time settlement on the middle chain; only destination-chain finality and the TSS keysign delay the outbound.
- Arbitrary asset pairs at protocol-set pricing.
- Cryptoeconomic recourse: misbehaviour is slashable. Thorchain operates at a documented bond-to-pooled ratio; Serai caps custody at 33% of allocated validator stake per network. Sources: [Thorchain RUNE docs](https://docs.thorchain.org/understanding-thorchain/rune) (accessed 2026-05-19); [Serai Validator Sets spec](https://github.com/serai-dex/serai/blob/develop/spec/protocol/Validator%20Sets.md) (accessed 2026-05-19).

Cons of the federated-signer pattern:

- Custody risk is real and realised (see incidents above).
- The signer federation is a chokepoint for censorship and out-of-protocol pressure on individually identifiable validators.
- Pre-economic-security bootstrap. Bonded-security guarantees do not bind until the validator stake pool catches up with custody. This is the structural reason Serai is pre-launch despite a complete audit.
- Public middle-chain state links source and destination identities on the comparator chains.

### Atomic swap

A two-party exchange where atomicity comes from cryptography (HTLC or adaptor signatures) rather than a third party. See the [atomic-swaps primer](./atomic-swaps-primer.md) for mechanics.

Representative protocols (XMR-BTC corridor specifically):

- **COMIT Network** (`comit-network/xmr-btc-swap`). Original Rust reference implementation of BTC-XMR adaptor-signature swaps. Reference implementation `comit-rs` archived 2021-03; `xmr-btc-swap` unmaintained per repo notice, last release v1.0.0-rc.1 on 2024-11-15. Maintenance has migrated to community-led `eigenwallet/core`. Source: [github.com/comit-network/xmr-btc-swap](https://github.com/comit-network/xmr-btc-swap) (accessed 2026-05-19).
- **Farcaster Project** (`farcaster-project/farcaster-node`). Independent BTC-XMR implementation. Still listed as actively maintained as of 2026, with Lightning BTC support added to reduce BTC-side confirmation time. Community-scale rather than volume-scale operation. Sources: [xgram.io: Best Monero atomic swap platforms 2026](https://xgram.io/blog/best-xmr-atomic-swaps-and-community-services-2026) (accessed 2026-05-19); [github.com/farcaster-project](https://github.com/farcaster-project) (accessed 2026-05-19).
- **AtomicDEX / Komodo Wallet.** Rebranded to "Komodo Wallet" in 2025. Public trackers report no recent volume: BitDegree's listing notes "no data available for AtomicDEX because of exchange inactivity"; Nomics' last published 24-hour volume is approximately USD 5,737 from November 2021. Not wound down, but has not produced competitive volumes. Sources: [Komodo Platform Roadmap](https://roadmap.komodoplatform.com/) (accessed 2026-05-19); [BitDegree: AtomicDEX trading data](https://www.bitdegree.org/top-crypto-exchanges/atomicdex) (accessed 2026-05-19); [Nomics: AtomicDEX](https://nomics.com/exchanges/atomicdex) (accessed 2026-05-19).
- **Liquality.** Consumer atomic-swap wallet extension discontinued effective 2024-06-15. Company pivoted to in-app wallet and SDK products. Sources: [Liquality on X, 2024-05-20](https://x.com/Liquality_io/status/1792678368694985162) (accessed 2026-05-19); [Rootstock Helpdesk: Liquality](https://helpdesk.rootstock.io/solutions/liquality.html) (accessed 2026-05-19).

What the user trusts: nothing beyond the soundness of the cryptographic construction and the liveness of the two parties for the duration of the swap.

Pros:

- No custody risk. Funds never leave control of one of the two participants. No validator set to slash, no vault to drain.
- No signer federation: no n-of-m threshold to compromise, no per-chain observer to censor.
- No pre-economic-security window. Cryptographic security is full from day one because there is no bond-to-custody ratio to bootstrap.

Cons (intrinsic to atomicity, not implementation defects):

1. **Free option on both sides.** Once one party has locked, the other can wait and observe price movement before completing or walking away. See the [free-option section in the primer](./atomic-swaps-primer.md#the-free-option-problem) and [Han et al., IACR 2019/896](https://eprint.iacr.org/2019/896) (accessed 2026-05-19) for the literature reference.
2. **Settlement time dominated by the slowest chain.** Confirmations stack across both chains. Third-party documentation of atomic-swap practice notes that "a single swap can take 30 minutes to several hours to finalise". Source: [xgram.io: Best Monero atomic swap platforms 2026](https://xgram.io/blog/best-xmr-atomic-swaps-and-community-services-2026) (accessed 2026-05-19).
3. **Mandatory interactivity for both parties.** Both must be online for lock, reveal, and (in adversarial paths) refund.
4. **Per-trade matching.** No protocol-owned liquidity. Each swap requires a willing counterparty for the exact pair and size. Third-party summary: "you cannot easily swap large amounts because you need to find a specific peer willing to take the other side of that exact trade" ([xgram.io, 2026](https://xgram.io/blog/best-xmr-atomic-swaps-and-community-services-2026), accessed 2026-05-19).
5. **Pair coverage.** HTLC requires compatible scripting; adaptor signatures generalise this but each new pair requires cross-curve cryptographic work. BTC-XMR specifically took years of work between first published research and production implementation; see the primer for sourcing.

## Adoption record

The two camps have produced very different volume outcomes.

Federated-signer middle chains:

- **Thorchain cumulative swap volume** $112.201B as of 2026-05-19 (DefiLlama). 30-day volume $1.632B; annualised fees $29.76M; current TVL $70.24M. Source: [DefiLlama Thorchain DEX dashboard](https://defillama.com/protocol/thorchain-dex) (accessed 2026-05-19).
- **Wormhole cumulative transfer volume** $58.95B all-time (claimed); Portal Bridge DefiLlama figure $58.19B; 30-day volume $1.169B. Sources: [Connecting the Internet Economy (Wormhole blog, 2025-04-03)](https://wormhole.com/blog/connecting-the-internet-economy-wormhole-and-the-w-tokens-past-present-and) (accessed 2026-05-19); [Portal TVL on DefiLlama](https://defillama.com/protocol/portal) (accessed 2026-05-19).

Atomic-swap protocols:

- **Liquality** reported "$35M in cross-chain atomic swaps facilitated through its wallet and interface" lifetime. Source: [defiprime.com: Liquality](https://defiprime.com/liquality) (accessed 2026-05-19).
- COMIT, Farcaster, AtomicDEX: no comparable cumulative-volume figure published. Recent activity in all three is community-scale rather than volume-scale.

The gap is four orders of magnitude on cumulative volume in the published evidence.

### Stated rationale from the projects themselves

The Thorchain account published a 2019-07-02 Medium post arguing that "bridges are superior to atomic swaps", citing liquidity gravity (peer-to-peer matching requires a counterparty per trade), user experience (multi-hour timelocks and CLI tooling), and pair coverage. Source: [Thorchain Medium, "Why Cross-Chain bridges are superior to Atomic Swaps" (2019-07-02)](https://medium.com/thorchain/why-cross-chain-bridges-are-superior-to-atomic-swaps-aebde263103c) (accessed 2026-05-19).

Luke Parker (Serai lead developer) said directly in a MoneroTalk interview: "while I do love atomic swaps [..] I don't feel the community actually wants atomic swaps, which is a brutal truth" (timestamp 35:50). Parker explicitly groups Serai with Thorchain, Maya, and Chainflip rather than with Farcaster or COMIT. Source: [Monero Observer: MoneroTalk interview with kayabaNerve on Serai DEX](https://monero.observer/monerotalk-kayabanerve-interview-serai-dex/) (accessed 2026-05-19).

The COMIT design corpus (RFC series, 25+ ADR-style spike documents, public blog) makes no mention of staking, reputation, or counterparty accountability mechanisms; the project relied entirely on timelock refund paths and adaptor-signature secrecy. Source: [comit-network/spikes/0017-negotiation-and-execution-protocol.adoc](https://github.com/comit-network/spikes/blob/master/0017-negotiation-and-execution-protocol.adoc) (accessed 2026-05-20).

## References

- [Thorchain DEX dashboard, DefiLlama](https://defillama.com/protocol/thorchain-dex) (accessed 2026-05-19)
- [Thorchain State of the Network February 2026](https://blog.thorchain.org/state-of-the-network-february-2026/) (accessed 2026-05-19)
- [Thorchain Bifrost TSS docs](https://dev.thorchain.org/bifrost/tss.html) (accessed 2026-05-19)
- [Thorchain RUNE bond-to-pooled docs](https://docs.thorchain.org/understanding-thorchain/rune) (accessed 2026-05-19)
- [Thorchain post-mortem: 2021 ETH router exploits](https://medium.com/thorchain/post-mortem-eth-router-exploits-1-2-and-premature-return-to-trading-incident-2908928c5fb) (accessed 2026-05-19)
- [Thorchain Medium: "Why Cross-Chain bridges are superior to Atomic Swaps" (2019-07-02)](https://medium.com/thorchain/why-cross-chain-bridges-are-superior-to-atomic-swaps-aebde263103c) (accessed 2026-05-19)
- [Crypto Times: $10.8M Thorchain GG20/TSSHOCK exploit (2026-05-17)](https://www.cryptotimes.io/2026/05/17/10-8-million-drained-inside-the-thorchain-exploit-that-froze-cross-chain-defi-for-13-hours/) (accessed 2026-05-19)
- [AMBCrypto: Thorchain MPC wallet concerns](https://ambcrypto.com/thorchain-exploit-raises-fresh-concerns-over-mpc-wallet-security/) (accessed 2026-05-19)
- [serai.exchange](https://serai.exchange/) (accessed 2026-05-19)
- [Audit of Serai's Substrate Blockchain (2026-04-15)](https://serai.exchange/2026/04/15/serai-blockchain-audited.html) (accessed 2026-05-19)
- [Announcing monero-oxide / FROSTLASS CLSAGs (2025-09-09)](https://serai.exchange/2025/09/09/monero-serai-oxide.html) (accessed 2026-05-19)
- [Serai Validator Sets spec](https://github.com/serai-dex/serai/blob/develop/spec/protocol/Validator%20Sets.md) (accessed 2026-05-19)
- [Monero Observer: MoneroTalk interview with kayabaNerve on Serai DEX](https://monero.observer/monerotalk-kayabanerve-interview-serai-dex/) (accessed 2026-05-19)
- [Wormhole Guardians docs](https://wormhole.com/docs/protocol/infrastructure/guardians/) (accessed 2026-05-19)
- [Portal TVL on DefiLlama](https://defillama.com/protocol/portal) (accessed 2026-05-19)
- [Connecting the Internet Economy (Wormhole, 2025-04-03)](https://wormhole.com/blog/connecting-the-internet-economy-wormhole-and-the-w-tokens-past-present-and) (accessed 2026-05-19)
- [Halborn: Wormhole Hack February 2022](https://www.halborn.com/blog/post/explained-the-wormhole-hack-february-2022) (accessed 2026-05-19)
- [Han et al., On the optionality and fairness of Atomic Swaps, IACR 2019/896](https://eprint.iacr.org/2019/896) (accessed 2026-05-19)
- [comit-network/spikes/0017-negotiation-and-execution-protocol.adoc](https://github.com/comit-network/spikes/blob/master/0017-negotiation-and-execution-protocol.adoc) (accessed 2026-05-20)
- [github.com/comit-network/xmr-btc-swap](https://github.com/comit-network/xmr-btc-swap) (accessed 2026-05-19)
- [github.com/farcaster-project](https://github.com/farcaster-project) (accessed 2026-05-19)
- [xgram.io: Best Monero atomic swap platforms 2026](https://xgram.io/blog/best-xmr-atomic-swaps-and-community-services-2026) (accessed 2026-05-19)
- [Komodo Platform Roadmap](https://roadmap.komodoplatform.com/) (accessed 2026-05-19)
- [BitDegree: AtomicDEX trading data](https://www.bitdegree.org/top-crypto-exchanges/atomicdex) (accessed 2026-05-19)
- [Nomics: AtomicDEX](https://nomics.com/exchanges/atomicdex) (accessed 2026-05-19)
- [Liquality on X: discontinuation (2024-05-20)](https://x.com/Liquality_io/status/1792678368694985162) (accessed 2026-05-19)
- [Rootstock Helpdesk: Liquality](https://helpdesk.rootstock.io/solutions/liquality.html) (accessed 2026-05-19)
- [defiprime.com: Liquality cross-chain atomic swaps](https://defiprime.com/liquality) (accessed 2026-05-19)
