# Cross-Chain Trust Model Contrast

Background appendix for the cross-chain RFP bundle (RFP-021, RFP-022, RFP-023, RFP-024, RFP-025). Establishes the design space the bundle navigates and the mitigations that distinguish the proposed RFPs from each other.

## Two architectural camps

Every deployed cross-chain swap design today collapses to one of two trust models.

### Federated-signer middle chain

Examples: Thorchain (live since 2021), Serai (pre-mainnet as of 2026-05), Maya, Chainflip. A purpose-built chain whose validator set custodies external assets via a threshold signature scheme (GG20 ECDSA for Thorchain; FROST per-curve for Serai, including FROSTLASS over CLSAG for Monero) and runs swap matching natively.

What the user trusts:

1. A threshold of the validator set will not collude to spend from the vault.
2. The cryptographic primitive used to combine signer contributions is sound. Not a free assumption: Thorchain's GG20 TSS failed in production in May 2026, draining $10.8M from an Asgard vault via a TSSHOCK-class weakness. Source: [Crypto Times, 2026-05-17](https://www.cryptotimes.io/2026/05/17/10-8-million-drained-inside-the-thorchain-exploit-that-froze-cross-chain-defi-for-13-hours/).
3. The implementations on every external chain are correct. Not a free assumption either: a Solana-side `load_instruction_at` bug let an attacker forge a Wormhole VAA in February 2022 and mint 120k wETH unbacked ($326M). Source: [Halborn](https://www.halborn.com/blog/post/explained-the-wormhole-hack-february-2022).

Pros:

- AMM-style liquidity. A single ordered state machine maintains pool invariants and serves all-comers without per-trade matching.
- One-step user experience: deposit-with-memo, await outbound. No counterparty interactivity, no refund flows, no online-availability requirement past broadcast.
- Sub-block-time settlement on the middle chain; only destination-chain finality and the TSS keysign delay the outbound.
- Arbitrary asset pairs at protocol-set pricing.
- Cryptoeconomic recourse: misbehaviour is slashable. Thorchain runs 2:1 bonded plus 1:1 pooled. Serai caps custody at 33% of allocated validator stake. Source: [Serai Validator Sets spec](https://github.com/serai-dex/serai/blob/develop/spec/protocol/Validator%20Sets.md).

Cons:

- Custody risk is real and realised (see incidents cited above).
- The signer federation is a chokepoint for censorship and out-of-protocol pressure on individually identifiable validators.
- Pre-economic-security bootstrap. Serai's mint-on-bootstrap design illustrates that the slashable-bond argument does not bind until the validator-stake pool catches up with custody.
- Public middle-chain state links source and destination identities on the comparator chains. Every comparator publishes the source-to-destination link on at least one public ledger.
- For Monero specifically: any TSS custody of XMR is necessarily view-key-shared. Serai's FROSTLASS over CLSAG is the most advanced production-grade design, but the validator set still observes which Monero outputs are committed to the swap; the privacy property is "honest-but-curious validators learn the LEZ-to-Monero deposit history" not "validators learn nothing".

### Atomic swap

Examples: COMIT Network (xmr-btc-swap, archived as of 2024-11), Farcaster, AtomicDEX (rebranded to Komodo Wallet, no recent volume), Liquality (discontinued 2024-06-15). All but Farcaster have wound down. The cryptographic primitive: HTLC for script-compatible pairs; adaptor signatures with cross-curve DLEQ proofs for Monero pairs.

What the user trusts: nothing beyond the soundness of the cryptographic construction and the liveness of the two parties for the duration of the swap.

Pros:

- No custody risk. Funds never leave control of one of the two participants. There is no validator set to slash and no vault to drain.
- No signer federation: no 13-of-19, no 67% threshold, no per-chain observer to be censored or compromised.
- No pre-economic-security window. The cryptographic security is full from day one because there is no bond-to-custody ratio to bootstrap.

Cons:

1. **Free option on both sides.** Once one party has locked, the other can wait and observe price movement before completing or walking away. The locker's downside is opportunity cost on inventory locked for the entire timelock window; the waiter's downside is gas plus time. Empirically this is the failure mode that kills BTC-XMR adoption: makers refuse to lock against takers who can free-option them for hours. Recognised in the atomic-swap literature as "the free option problem"; not a bug in any particular implementation, but the cost of atomicity itself.
2. **Settlement time dominated by the slowest chain.** A single swap can take 30 minutes to several hours to finalise because confirmations stack across both chains. Refund timelocks are measured in hours, not minutes.
3. **Mandatory interactivity for both parties.** Both parties must be online for lock, reveal, and (in adversarial paths) refund. If Alice goes offline mid-swap, Bob waits out the refund window, and vice versa.
4. **Per-trade matching.** No protocol-owned liquidity, no AMM pricing; each swap requires a willing counterparty for the exact pair and exact size.
5. **Pair coverage.** HTLC requires compatible scripting on both chains. BTC-XMR specifically required about five years of cryptographic work (2017 proposal to 2021 working implementation via adaptor signatures over Ed25519 and secp256k1). Source: [getmonero.org: Bitcoin to Monero atomic swaps are now live, 2021-08-20](https://www.getmonero.org/2021/08/20/atomic-swaps.html); [Hoenisch and del Pino, IACR 2020/1126](https://eprint.iacr.org/2020/1126.pdf).

Cons (1) through (3) are the ones the design space can plausibly attack without abandoning the atomic-swap model entirely. (4) is structural: "fixing" it by adding protocol-owned liquidity reinvents the middle-chain DEX. (5) is a per-pair engineering cost paid once.

### Why the federated middle chain has won the volume race

Thorchain has processed $112B cumulative volume since 2021. The four most-cited atomic-swap projects have collectively produced roughly $35M in volume over the same period (Liquality lifetime figure, [defiprime.com: Liquality](https://defiprime.com/liquality)). The structural reasons:

- Liquidity gravity. Peer-to-peer matching requires a counterparty per trade; AMM pools concentrate liquidity into a single state machine.
- User experience. Multi-hour timelocks, online-availability requirements, refund flows, and command-line tooling have prevented retail adoption.
- Pair coverage. HTLC was constrained to chain pairs with compatible scripting for years before adaptor-signature techniques generalised it.

Source: [Thorchain Medium, "Why Cross-Chain bridges are superior to Atomic Swaps" (2019-07-02)](https://medium.com/thorchain/why-cross-chain-bridges-are-superior-to-atomic-swaps-aebde263103c).

## Mitigations for the addressable atomic-swap cons

Three independent levers. None is sufficient on its own; their combination defines specific niches that the proposed RFPs target.

### Mitigation 1: same-asset-to-same-asset (bridge, not trade)

If the two legs of the swap are denominations of the *same* underlying asset (e.g. native BTC on Bitcoin and an LEZ-side wrapped-BTC token redeemable 1:1 to native BTC via a light-client inclusion proof), the *trade* component disappears: there is no relative-price volatility between the two legs, so the free-option value collapses toward zero. The mid-swap optionality that makes the maker refuse to lock against an arbitrary taker is a function of `σ × √T × notional`; if `σ → 0` for the pair, the option is worthless and the timelock window stops being an exploitable asset.

Important scoping:

- This argument applies only to a 1:1 wrapped or SPV-backed token, where redemption is at a contractually fixed ratio and the only `σ` is residual peg slack (premium/discount, queue depth, fees).
- It does *not* apply to an oracle-priced synthetic token (e.g. an sXMR that tracks the XMR price via oracle, collateralised by stables or other assets). The peg slack of an oracle-priced synthetic *is* the volatility the free option pays out on. Oracle-priced synthetic exposure is a distinct product (see RFP-024 and RFP-025); it does not solve the free-option problem.

Feasibility per chain:

- For chains with public outputs (BTC, ETH), a 1:1 wrap via light-client proofs is principled. The LEZ-side mint primitive is a Risc0 program that verifies an inclusion proof against a header chain (forkable from ZeroSync or Citrea's Clementine LCP for BTC; from existing Ethereum light-client work for ETH).
- For Monero, no principled wrap is feasible today. Monero has no SPV-style proof primitive that can demonstrate "address Y received amount X" without view-key sharing: ring signatures, RingCT, and one-time stealth addresses defeat external observation by design. Monero's bilateral `check_tx_proof` works in a private wallet-to-wallet context but cannot be lifted to a public LEZ contract without disclosing the per-tx private key and blinding factor to world-readable state, which is mathematically equivalent to view-key disclosure for the swap output. The FCMP++ (full-chain membership and metadata-private proofs) research direction may unlock a non-disclosing variant; it is pre-production. Source: [Monero stealth address documentation](https://www.getmonero.org/library/Zero-to-Monero-2-0-0.pdf); [Monero FCMP++ overview](https://www.getmonero.org/resources/moneropedia/fcmp.html).

Where Mitigation 1 fits in the bundle:

- BTC and ETH: complementary to RFP-022 Tier 1 (bonded atomic swaps), but worth treating as a distinct design (the user does not have to overpay a bond when the trade itself has near-zero volatility).
- XMR: not currently feasible. Move to Mitigation 2 or 3 for free-option control.

### Mitigation 2: bonded atomic swap (forced completion via slashing)

The maker/taker bond design RFP-022 specifies in detail. Bonds posted on LEZ; slashing conditioned on LEZ-observable failures to advance through the swap state machine.

#### The free-option problem in concrete form

Standard adaptor-signature swap. Alice locks first (XMR or BTC) into a 2-of-2 output. Bob then locks Logos in an LEZ contract conditioned on secret `s`. To claim Logos, Alice publishes a signature that reveals `s` to Bob; Bob uses `s` to sweep Alice's lock.

- Between Lock-Logos and Reveal, Alice holds a free option on the price.
- Between Alice's first lock and Bob's Logos lock, Bob can refuse to lock if the price moved.

#### Why slashing only works on the LEZ side

Monero has no scripting. Bitcoin has scripting too limited to hold a bond conditioned on protocol behaviour. The only place a slash can be enforced is on a smart-contract chain. LEZ is the right host for both bonds, but the LEZ contract needs to verify the preconditions (the locks) before applying any slash. This verification primitive splits cleanly into two tiers.

#### Tier 1: symmetric bonding for LEZ to BTC and LEZ to ETH

Both sides' locks are verifiable on LEZ via a chain-watching light-client module: BTC via a Risc0 header-chain light client (forkable from ZeroSync or Citrea Clementine LCP); ETH via existing Ethereum light-client work (e.g. Nimbus-derived). The locker's outputs on these chains are publicly identifiable to a known scriptpubkey or address, so an inclusion proof on LEZ leaks nothing the locker relied on as private. Both Alice's and Bob's bonds are slashable on default: full bilateral free-option mitigation.

Phases and slash conditions (LEZ to BTC example):

| Phase | What happens | Slash condition |
|-------|--------------|-----------------|
| 0. Quote | Bob signs quote (price, expiry, swap_id, refund_pubkeys); Alice and Bob run joint-key setup for the BTC 2-of-2 Taproot output | none |
| 1. Commit | Alice posts `B_alice` on LEZ referencing swap_id | none |
| 2. Lock-BTC | Alice constructs and signs the BTC lock tx, sends raw bytes to Bob over Waku; Bob verifies and broadcasts to Bitcoin (if Bob stalls, Alice broadcasts herself). Once confirmed, anyone submits `{btc_block_headers, merkle_proof, raw_tx}` to the LEZ swap contract, which verifies PoW, inclusion, scriptpubkey, and amount | If lock is confirmed on BTC but Bob does not advance to Lock-Logos within window: `B_bob_slice` goes to Alice |
| 3. Lock-Logos | Bob locks `trade_amount` plus `B_bob_slice` in LEZ contract conditioned on `s` | none |
| 4. Reveal | Alice publishes adaptor signature, revealing `s` to Bob | If Alice does not reveal within window: `B_alice` goes to Bob |
| 5. Settle | Bob claims BTC using `s`; Alice's bond and Bob's bond slice released | If Bob does not claim before deadline: no slash (capital loss is on Bob); Alice's bond auto-refunds |

The unauthenticated proof-submitter property: Bob can broadcast Alice's signed lock tx himself (broadcasting is permissionless on every chain); the LEZ inclusion-proof submitter is also unauthenticated. This eliminates a class of grief vectors: if Alice signs a malformed lock tx (wrong amount, wrong scriptpubkey), Bob simply does not broadcast it, the tx never lands on Bitcoin, the inclusion proof never materialises, and the LEZ state machine quietly times out. There is no "attest or be slashed" dispute to adjudicate, because the precondition for state advancement (a real BTC lock) never holds.

Same reasoning applies symmetrically to ETH.

#### Tier 2: asymmetric bonding for LEZ to XMR

Alice's XMR lock cannot be proven on LEZ without view-key disclosure to public state, per the impossibility result above (Monero stealth addresses plus RingCT defeat external observation; `check_tx_proof` lifted to a public contract is equivalent to view-key disclosure for the swap output). Consequence:

- **Bob's bond is slashable on default.** Bob's lock is on LEZ and fully observable. Bond mechanic works exactly as in Tier 1 for Bob's side.
- **Alice's bond is slashable only on LEZ-observable abandonment.** It cannot be slashed for "failing to lock XMR" because LEZ cannot verify whether she did. It *can* be slashed for failing to reveal the secret on LEZ after Bob has locked Logos, because both the reveal and Bob's lock are LEZ-observable.
- **Residual free option Alice keeps.** Alice can refuse to lock XMR after Commit without on-chain consequence (her bond is gated by the reveal-after-Bob-lock event, which never occurs in this branch). Bob detects this off-chain by not seeing the XMR lock arrive and walks away without locking Logos; no slash on either side. Alice keeps a pre-XMR-lock free option but loses the post-Bob-lock free option. The residual option is smaller (it is the option to walk away from a quote before any meaningful commitment) but it is real.

Tier 2 user experience: "Bob is bonded; Alice is reputation-gated only" under the current cryptography. When FCMP++ ships, Alice's lock becomes verifiable on LEZ without view-key disclosure and Tier 2 collapses into Tier 1; the RFP should carry an explicit upgrade clause.

#### What both tiers fix

- Con (1), the free option, partially or fully depending on tier. In Tier 1 the slash makes both parties' optionality strictly EV-negative when bonds are sized above option value (`σ × √T × notional`, so 2-5% of trade notional for 1-hour windows). In Tier 2 only Bob's optionality is closed.

#### What neither tier fixes

- Con (2): settlement time. Still bounded by source-chain finality plus LEZ finality plus the timelock window. The bond does not accelerate cryptographic settlement.
- Con (3): interactivity. Both parties must be online to lock, reveal, and (if the other side defaults) submit the slash claim. The bond removes the incentive to grief but not the requirement to participate.
- Cross-chain bond correlation: if Bob is matched against N concurrent swaps and LEZ re-orgs or his observer crashes, all N swaps slash him. Per-maker concurrency caps or bond scaling with active-swap count are needed.

### Mitigation 3: maker and taker reputation

A maker is a repeat participant; a long history of completed swaps is itself a slashable asset because losing the reputation forfeits all future fee revenue. This is the same argument that secures Wormhole's Guardians without bonded stake (reputation as economic gravity) but applied to a maker registry rather than a signer set. The proposed primitive is the subject of RFP-023.

#### Maker reputation: trivially linkable

Bob already wants a persistent identity on LEZ to receive quote requests, accumulate fee revenue, amortise bond posting, and signal trustworthiness. Layering reputation (count of completed swaps, slash history, time-in-protocol, total fee revenue) on top of (or instead of) a bond compounds the cost of defection. A maker with 10,000 completed swaps walking away from a single griefable trade is an irrational actor; the reputation acts as a long-tailed bond that the protocol cannot directly slash but the market does.

#### Taker reputation: the privacy tension

Takers are the population a privacy-positioned cross-chain DEX specifically wants to keep anonymous. A persistent on-chain taker identity that accumulates reputation is at direct odds with the privacy positioning, because reputation requires linkability across swaps by definition.

Two design paths the proposed RFPs require applicants to consider:

- **Capped anonymous takers.** First-swap takers size-capped (US$100 notional) without reputation; the cap relaxes after N successful completions under the same persistent pseudonym. Linkability is opt-in: a taker who wants larger size accepts linkability as the cost.
- **Zero-knowledge reputation.** Takers prove "I have completed at least N swaps with zero slashes" without revealing *which* swaps, via a Sparse Merkle Tree of swap outcomes maintained by the LEZ escrow program plus a zk membership proof. Preserves unlinkability across swaps while letting the taker borrow against accumulated reputation. Engineering-expensive; reuses the LEZ zkVM (Risc0) that RFP-003 already establishes.

#### Threat model

- Sybil attacks: cheap to mint new pseudonyms. Mitigation: reputation accrual is rate-limited by completed-swap throughput and notional caps. Cost of building reputation equals the cost of N honest swaps plus the bond-equivalent capital tied up over the accrual window.
- Maker griefing under multiple identities: a maker who slashes one identity can spin up another. Same mitigation: reputation accrual takes time and capital.
- Sidechannel deanonymisation: even zk reputation has timing, notional, and maker-side-view sidechannels. The privacy claim is "unlinkable across the public ledger" not "unlinkable to a maker who actively profiles its counterparties". This distinction must be documented for users.
- Bootstrap problem: the protocol has zero reputation at launch, so the first cohort of swaps has the strongest free-option exposure. Mitigation: start with caps plus RFP-022 bonds for early adopters; transition to reputation-only as reputation accrues.

## The bondless-taker capped-entry mechanic

A cross-cutting onboarding constraint that every bonding RFP in this bundle adopts (RFP-022, RFP-025).

The problem: a privacy-seeking taker arriving from XMR or BTC has no LEZ-denominated assets yet. Requiring them to acquire LEZ assets before their first swap, then lock those assets as a bond, is exactly the chicken-and-egg the cross-chain DEX is supposed to solve.

The mechanic:

- First swap is capped at a small notional (worked example: US$100 equivalent), with no taker bond required.
- The cap is enforceable by the LEZ escrow program directly; no reputation registry needed.
- After completing the first swap, the taker has LEZ-denominated assets in their account. They can post these as a bond against larger swap sizes (or, if RFP-023 reputation is available, accrue reputation in lieu of bond).
- The cap value (US$100) is illustrative; applicants size it against expected free-option value at the protocol's typical lock window.

Why this matters: it is the only way the bonding designs (RFP-022, RFP-025) provide a viable entry path for first-time takers without making them custody-tolerating or KYC-tolerating to acquire LEZ assets out of band.

## Synthesis

The three mitigations combine, with each RFP in the bundle picking a different combination:

| RFP | Mitigation 1 | Mitigation 2 | Mitigation 3 | Bondless taker |
|-----|--------------|--------------|--------------|----------------|
| RFP-021 (federated middle chain) | not applicable | not applicable | not applicable | not applicable |
| RFP-022 (bonded atomic swaps) | complementary (BTC, ETH) | core mechanic (two tiers) | maker side | yes |
| RFP-023 (reputation-based) | not applicable | not applicable | core mechanic | inherits via cap |
| RFP-024 (sXMR pure) | not applicable | not applicable | not applicable | not applicable |
| RFP-025 (sXMR with SLA) | not applicable | LP-side bond (2a) or reserve (2b) | LP side | yes |

The federated middle chain (RFP-021) sidesteps the free-option problem entirely by removing atomicity in favour of AMM-style execution. The bonded and reputation designs attack the free-option problem directly. The sXMR designs use atomic swaps as the redemption settlement layer rather than the DEX trading primitive, so the free-option problem appears in a different form (LP unavailability rather than counterparty defection).

A reader choosing between the RFPs should ask:

- Is custody acceptable in exchange for AMM liquidity and one-step UX? RFP-021.
- Is non-custody worth multi-hour settlement and interactivity, for BTC and ETH at scale? RFP-022 Tier 1.
- Is non-custody worth multi-hour settlement and interactivity for XMR specifically, accepting that Alice retains a residual pre-lock free option? RFP-022 Tier 2 plus RFP-023.
- Is reputation as economic gravity preferable to bonded collateral for the trust gradient? RFP-023.
- Is the requirement "synthetic XMR exposure inside LEZ DeFi composability" rather than "real XMR settlement"? RFP-024 (no SLA) or RFP-025 (with SLA).

## References

- [Bitcoin to Monero atomic swaps are now live (getmonero.org, 2021-08-20)](https://www.getmonero.org/2021/08/20/atomic-swaps.html)
- [Hoenisch and del Pino, Atomic Swaps between Bitcoin and Monero, IACR 2020/1126](https://eprint.iacr.org/2020/1126.pdf)
- [Decred blog: On-Chain Atomic Swaps (2017)](https://blog.decred.org/2017/09/20/On-Chain-Atomic-Swaps/)
- [comit-network/xmr-btc-swap (archived 2024-11)](https://github.com/comit-network/xmr-btc-swap)
- [Thorchain Medium: Why Cross-Chain bridges are superior to Atomic Swaps (2019-07-02)](https://medium.com/thorchain/why-cross-chain-bridges-are-superior-to-atomic-swaps-aebde263103c)
- [Thorchain docs: Continuous Liquidity Pools](https://docs.thorchain.org/technical-documentation/thorchain-finance/continuous-liquidity-pools)
- [Serai Validator Sets spec](https://github.com/serai-dex/serai/blob/develop/spec/protocol/Validator%20Sets.md)
- [Serai AMM docs](https://docs.serai.exchange/amm/)
- [Halborn: Wormhole Hack February 2022](https://www.halborn.com/blog/post/explained-the-wormhole-hack-february-2022)
- [Crypto Times: $10.8M drained from Thorchain (2026-05-17)](https://www.cryptotimes.io/2026/05/17/10-8-million-drained-inside-the-thorchain-exploit-that-froze-cross-chain-defi-for-13-hours/)
- [Monero whitepaper: Zero to Monero 2.0](https://www.getmonero.org/library/Zero-to-Monero-2-0-0.pdf)
- [Monero FCMP overview](https://www.getmonero.org/resources/moneropedia/fcmp.html)
- [defiprime.com: Liquality](https://defiprime.com/liquality)
- [Liquality discontinuation announcement (2024-05-20)](https://x.com/Liquality_io/status/1792678368694985162)
