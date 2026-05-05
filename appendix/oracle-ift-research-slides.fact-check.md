# Fact-Check Review: twap-oracle-rfps-slides.md

**Verdict:** APPROVED with WARNINGs

**Scope:** All factual claims in `/home/fryorcraken/src/logos-co/twap-oracle-rfps-slides.md` (talk-track for RFP-019 + RFP-020).
**Sources checked:** 1 live URL fetched; cross-referenced against `appendix/oracle-ecosystem.md`, `RFPs/RFP-019-twap-oracle.md`, `RFPs/RFP-020-redstone-oracle-adaptor.md` in the rfp-twap-oracle repo.
**Standard applied:** Slides are a synthesis layer over the appendix (which carries the citations). Tolerance: numeric claims must match the appendix exactly or within rounding; categorical claims must not contradict the appendix's nuance. Freshness threshold: 6 months for ecosystem stats, tighter where the slide claims current state.

The slides faithfully summarise the appendix on the load-bearing technical claims (signature schemes, quorum sizes, cardinality, manipulation cost shape, production architectures). Findings below are mostly nuance loss in the synthesis, not fabrication or staleness.

## CRITICAL (0)

None.

## WARNING (2)

- **L57-61 (Two kinds of oracle / Off-chain oracles tables) — RedStone classified as Pull only.**
  - Issue: The push vs pull table at L107-110 says "Pull (Pyth, RedStone)" and the table at L112-117 lists RedStone alongside Pyth without flagging that RedStone runs in both modes. The slide later contradicts itself at L210: "RedStone supports both push and pull natively (50+ push deployments, 120+ pull deployments)". An audience reading only the early table will infer RedStone is pull-only, which the appendix (line 28: "50+ push / 120+ pull") and slide L210 explicitly refute.
  - Source: appendix `oracle-ecosystem.md` line 28; slide internal contradiction.
  - Fix: in the L107-110 table, change RedStone's classification to "Push + Pull" or add a footnote that RedStone supports both, with the LEZ design adopting push.

- **L352 (Why RedStone speaker notes) — Pyth "120+ publishers" vs appendix internal inconsistency.**
  - Issue: The slide cites "120+ publishers, confidence intervals" for Pyth in the speaker notes. The appendix is internally inconsistent: line 27 and line 68 say "120+ institutional publishers (April 2026)", while line 291 (the comparison table) says "70+ first-party publishers". The slide picks the higher figure without flagging the discrepancy. Either the appendix table is stale or the headline figure is overstated; the slide inherits that ambiguity.
  - Source: appendix lines 27, 68, 291.
  - Fix: reconcile in the appendix first (pick a single figure with a dated source, update the comparison table); then the slide can cite the resolved number with confidence.

## NOTE (4)

- **L143 / L417 — DLC oracle roster framed as if all four are live.**
  - The slide lists "Pythia, Sibyls, Suredbits, Ernest Oracle" as BIP-340 BTC/USD publishers. Per the appendix live catalogue (lines 622-654): Pythia is the only live mainnet operation; Sibyls operator is dead and the GitHub repo deleted; Ernest is on hiatus since 2025-06-02. Calling them a four-member live ecosystem overstates current breadth. The argument (none publish XMR/ZEC, all are single-operator BTC discrete-event publishers) is unaffected, but the audience may infer more activity than exists.
  - Suggested phrasing: "the candidates that did or do (Pythia is live; Sibyls, Suredbits, Ernest Oracle are dormant or on hiatus)…"

- **L78-79 / L240-241 — "12-second blocks" assumption for cardinality math.**
  - The "~9 days" figure derives from Uniswap V3 on Ethereum (12-second blocks × 65,535). LEZ block time may differ. The slide is in a Uniswap V3 reference context so it is not strictly wrong, but the arithmetic does not transfer one-to-one to LEZ. Worth a single-sentence callout if the audience may conflate the two.

- **L116 (Off-chain oracles table) — "secp256k1 ECDSA + keccak256 (Wormhole VAA)" elides "double-keccak256 off Solana".**
  - Appendix line 342-343: "Each guardian signs the double-keccak256 hash of the VAA body (single keccak256 on Solana, because the Solana secp256k1 program hashes the message itself)". Acceptable simplification for slide prose; flagged here so the speaker is aware if questioned.

- **Speaker-note TODOs flagged but not fact-check issues.**
  - L17, L247, L394, L397, L437 contain explicit TODOs (audience profile, RFP funding source, calendar dates, application window, additional Q&A). These are unfinished content, not factual errors. Resolve before presenting.

## Verified (sampled)

- **L50** — "36 documented flash-loan oracle attacks have caused $418M in cumulative DeFi losses." Matches appendix line 213-214 (Rekt Database via Ormer paper).
- **L78** — "Cardinality up to 65,535 observations (~9 days)". Matches appendix line 240-241 (218 hours ≈ 9 days at 12s blocks).
- **L114** — Pyth "13-of-19" quorum. Matches appendix line 339-341.
- **L114** — RedStone "typically 3-of-N". Matches appendix line 397-399 ("at least three unique signers as a balance").
- **L132-134** — "LEZ has one signature primitive wired into the runtime: single-key BIP-340 Schnorr over SHA-256… not exposed to guest programs." Matches appendix lines 302-311 verbatim in substance.
- **L136 / L200 / L320 / L415** — `https://github.com/fryorcraken/lez-signature-bench` URL live (HTTP 200), and content contains the cited terms (secp256k1, ECDSA, RISC0, "minutes", "consumer hardware"). Claim that proof generation is "in the order of minutes on consumer hardware" is consistent with the linked repo.
- **L210** — RedStone "50+ push deployments, 120+ pull deployments per the appendix survey." Matches appendix line 28.
- **L231-235** — Aave V3 / Compound V2 / MakerDAO / Liquity V2 architecture summaries. Match appendix lines 1007-1078 in detail (Coinbase + Uni V2 with 20% divergence; OSM 1-hour delay; Liquity V2 single fallback; Aave V3 governance-set fallback).
- **L169 / L419** — "FrostOracle (Chen et al., IEEE iThings 2023)" and iBTC/DLC.Link federation as closest production precedent. Matches appendix lines 530-553.
- **L184-189** — PoS multi-block validator attack cost shape (round-trip swap fees + price impact, no back-run competition). Matches appendix lines 184-192.
- **L243-244 / L262-263** — RFP-019 5% divergence threshold. Matches RFP-019 lines 135-136, 247-248.
- **L120 (speaker notes)** — Pyth Pythnet "every 400ms" cadence. Matches appendix line 71.
- **L143** — "Switchboard signs with ed25519." Matches appendix line 326.

## Sources fetched

- `https://github.com/fryorcraken/lez-signature-bench` — live (HTTP 200), content verified.
- Appendix `appendix/oracle-ecosystem.md` (1347 lines) — read in full for cross-check.
- `RFPs/RFP-019-twap-oracle.md` (460 lines) — relevant sections cross-checked.
- `RFPs/RFP-020-redstone-oracle-adaptor.md` (685 lines) — relevant sections cross-checked.

The slides do not introduce new external URLs beyond the lez-signature-bench prototype; they rely on the appendix as the upstream citation layer. No external claim was found that is contradicted by primary sources.
