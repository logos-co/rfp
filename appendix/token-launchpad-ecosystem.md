# Appendix: Token Launchpad Ecosystem

This appendix surveys existing token launchpad protocols and related
infrastructure across the Ethereum and Solana ecosystems. It serves
as a reference for
[RFP-015](../RFPs/RFP-015-bonding-curve-launchpad.md) and
[RFP-016](../RFPs/RFP-016-lbp-launchpad.md), providing context on
current market participants, their mechanisms, and design trade-offs.

## Protocols Considered

Protocols are ordered by cumulative scale (largest first) and this
order is maintained throughout the document.

| Protocol         | Ecosystem              | Mechanism                                                  | Website                                           |
| ---------------- | ---------------------- | ---------------------------------------------------------- | ------------------------------------------------- |
| Pump.fun         | Solana                 | Constant-product bonding curve                             | [pump.fun](https://pump.fun/)                     |
| Fjord Foundry    | Ethereum (multi-chain) | LBP (Balancer-based)                                       | [fjordfoundry.com](https://www.fjordfoundry.com/) |
| Metaplex Genesis | Solana                 | Launch Pool (proportional), presale, uniform price auction | [metaplex.com](https://www.metaplex.com/)         |
| DAO Maker        | Ethereum               | SHO (Strong Holder Offering), DYCO                         | [daomaker.com](https://daomaker.com/)             |
| Flaunch          | Base (Ethereum L2)     | Bonding curve via Uniswap V4 hooks                         | [flaunch.gg](https://flaunch.gg/)                 |
| Polkastarter     | Ethereum, BNB Chain    | Fixed-price IDO with lottery allocation                    | [polkastarter.com](https://polkastarter.com/)     |
| DAOs.fun         | Solana                 | Fixed-price fair launch for investment DAOs                | [daos.fun](https://daos.fun/)                     |
## Scale and Traction

Launchpads do not hold persistent TVL the way lending or staking
protocols do: capital flows through them during a sale and then moves
on. The tables below use the best available scale metric for each
protocol (cumulative protocol revenue where available, otherwise
cumulative volume raised or AUM). TVL is shown where it applies.

DAOs.fun is omitted from this section due to insufficient
quantitative data.

### Cumulative scale

Protocols ranked by cumulative protocol revenue (or estimated
revenue where only total raised and fee rate are available).

| Protocol | Cumulative Revenue | TVL / AUM | Tokens Launched | Data as of |
|---|---|---|---|---|
| Pump.fun | $818M+ (protocol fees) [1][2] | $140M (Pumpswap DEX) [3] | 11.7M (H1 2025) [4] | Early 2026 |
| Fjord Foundry | ~$55M (est.) [a] | N/A | 717 LBPs [6] | Sep 2025 |
| Metaplex (all products) | $36M+ [7] | N/A | 20M+ fungible tokens [7] | Sep 2025 |
| DAO Maker | ~$4.5M (est.) [b] | N/A | N/A | Undated |
| Flaunch | $3.1M [9] | $2.03M [9] | N/A | Q1 2026 |
| Polkastarter | ~$460K (est.) [c] | $3.31M (POLS liquidity) [10] | 105+ projects [10] | Q1 2022 |

\[a] Estimated as 5% fee [11] applied to $1.1B cumulative raised [5].
\[b] Estimated as 5% fee [12] applied to $90M+ cumulative raised [8].
\[c] Estimated as 1% fee [13] applied to $46M+ cumulative raised [10].
Figure is from Q1 2022; no updated aggregate has been published.

### Yearly breakdown

Where per-year data exists. Blank cells indicate no data available
for that period.

| Protocol         | Metric                          | 2024                                                  | 2025                       | 2026 (YTD / projected) |
| ---------------- | ------------------------------- | ----------------------------------------------------- | -------------------------- | ---------------------- |
| Pump.fun         | Protocol revenue                | ~$358M [14]                                           | ~$460M+ [14]               |                        |
| Pump.fun         | Cumulative trading volume       |                                                       | >$150B (Dec 2025) [1]      |                        |
| Metaplex         | Protocol revenue (all products) | ~$12-15M (est.) [7]                                   | ~$27-30M (est.) [7]        |                        |
| Metaplex Genesis | Launchpad revenue               |                                                       | $422K/month (Aug 2025) [7] |                        |
| Flaunch          | Protocol revenue                |                                                       | ~$3.1M [9]                 | $100.6K (Q1) [9]       |
| Fjord Foundry    | Cumulative raised (all sales)   | >$1.1B (cumulative since 2021; no per-year split) [5] |                            |                        |
| DAO Maker        | Total raised                    | $90M+ (cumulative; no per-year split) [8]             |                            |                        |
| Polkastarter     | Total raised                    | $46M+ (cumulative as of Q1 2022; stale) [10]          |                            |                        |

**Caveats:**

- Metric types differ across protocols (protocol revenue vs. AUM vs.
  cumulative volume raised). Cross-category comparisons should account
  for this.
- Fjord Foundry, DAO Maker, and Polkastarter do not publish per-year
  breakdowns.
- Polkastarter's aggregate figure dates from Q1 2022; current
  cumulative total is likely higher but unpublished.
- Metaplex Genesis launchpad revenue is a small fraction of total
  Metaplex protocol revenue; most Metaplex revenue comes from Token
  Metadata fees charged to platforms like Pump.fun [7].

## Refund Mechanisms

Most crypto-native launchpads treat purchases as final: once a buyer
executes a swap or deposit, there is no platform-level refund. Two
notable exceptions exist: DAO Maker's DYCO product offers a 16-month
post-purchase refund window [15], and DAOs.fun allows redemption
during the fundraising phase with a 10% penalty [21]. Regulatory frameworks are moving in
the opposite direction: EU MiCA mandates a 14-day cooling-off period
for white paper offerings [19], and US Reg CF requires a 48-hour
cancellation window plus a mandatory minimum raise threshold [20].

### Summary

| Protocol | Post-purchase refund | Minimum raise (soft cap) | Cancellation during sale |
|---|---|---|---|
| Pump.fun | No | No | No |
| Fjord Foundry | No [11] | No [11] | Sell back at market price (buy+sell mode only) [11] |
| Metaplex Genesis | No [d] | Yes (Launch Pool only) [16] | Yes (Launch Pool: withdraw before claim window; 2% fee) [16] |
| DAO Maker (DYCO) | Yes: 16 months, USDC-backed, burn on refund [15] | No | Yes: unconditional within window [15] |
| DAO Maker (SHO) | No [12] | No | No |
| Flaunch | No [17] | No | No |
| Polkastarter | No [18] | No | No |
| DAOs.fun | No (AMM exit + mandatory expiry distribution) [21] | Yes (hard cap; refund if unmet) [21] | Yes: redeem during fundraising (10% penalty) [21] |

\[d] Metaplex Genesis Launch Pool allows withdrawal during the deposit
window (before tokens are distributed). If the funding goal is not
met, depositors receive a full refund [16]. Once the claim window
opens and tokens are distributed, purchases are final.

### Per-protocol detail

**Pump.fun.** No refund mechanism. Bonding curve purchases are
irreversible AMM swaps. Once a token graduates to Pumpswap, it trades
on the open market with no platform-level buyer protection.

**Fjord Foundry.** No refund mechanism [11]. LBP purchases are AMM
swaps and are final once executed. In buy+sell mode, participants can
sell tokens back into the pool during the sale at the current market
price (not at their purchase price). After the LBP ends, tokens are
distributed and there is no refund path. No minimum raise threshold
exists; the sale runs regardless of demand.

**Metaplex Genesis.** The Launch Pool mechanism provides partial buyer
protection: depositors can withdraw during the deposit window (subject
to a 2% fee), and if the funding goal is not met, all deposits are
refunded [16]. However, once the claim window opens and tokens are
distributed, there is no refund. Presale and Uniform Price Auction
modes have no refund mechanism.

**DAO Maker (DYCO).** The Dynamic Coin Offering is the only
crypto-native mechanism with a post-purchase performance refund [15].
All tokens sold are backed 1:1 by USDC in a smart contract for 16
months after the Token Generation Event. Participants can return
tokens at any time within this window to receive the original USDC
price; returned tokens are burned, reducing circulating supply. If the
token price falls below the refund price on the secondary market,
arbitrageurs can buy cheaply and refund at the higher price, creating
a reliable price floor. DYCO is a specialized product; most DAO Maker
sales use the SHO model, which has no refund mechanism.

**Flaunch.** No refund mechanism [17]. The 30-minute fixed-price fair
launch phase is one-directional (buyers cannot sell during this
window). After graduation to standard AMM pricing, tokens trade freely
with no platform-level refund.

**Polkastarter.** No refund mechanism [18]. Fixed-price IDO purchases
are atomic swaps and are final once executed. Participants who are
allowlisted but have not yet purchased can choose not to participate,
but there is no post-purchase cancellation.

**DAOs.fun.** Structured refund mechanism with three phases [21].
During the 7-day fundraising window, contributors can redeem their SOL
at any time, subject to a 10% early redemption penalty. If the
fundraising hard cap is not reached, contributors receive a full
refund. After a successful raise, there is no direct refund: token
holders can sell on the virtual AMM at market price, or wait for the
mandatory fund expiration (3 months to 1 year, set by the creator at
launch), at which point remaining assets are distributed
proportionally and token holders can burn tokens to redeem underlying
SOL.

### Regulatory context

Two regulatory frameworks mandate buyer refund rights for token sales:

- **EU MiCA** (Markets in Crypto-Assets Regulation): requires a 14-day
  right of withdrawal for crypto-asset purchases made under a white
  paper offering [19]. This applies to any platform operating in the
  EU.
- **US Regulation CF** (crowdfunding): requires a 48-hour cancellation
  window before the offering deadline, a mandatory minimum raise
  threshold (all funds returned if the target is not met), and
  automatic cancellation if material changes are made to the offering
  terms (investors must reconfirm within 5 business days) [20].

Neither framework applies directly to permissionless, non-custodial
on-chain launchpads. However, they indicate a regulatory direction
toward mandatory cooling-off periods for token sales.

## References

1. CoinDesk, "Most Influential: Pump.fun," Dec 2025.
   https://www.coindesk.com/coindesk-news/2025/12/10/most-influential-pump-fun
2. Tokenomics.com, "Pump.fun Tokenomics: How Pump Distributes $45M
   Monthly to Holders," Feb 2026.
   https://tokenomics.com/articles/pumpfun-tokenomics-how-pump-distributes-45m-monthly-to-holders
3. CoinLaunch, "Pump.fun Overview," Jul 2025.
   https://coinlaunch.space/projects/pumpfun/
4. Metaplex Foundation, "H1 2025 Recap."
   https://www.metaplex.foundation/blog/articles/metaplex-1h-25-recap
5. Fjord Foundry homepage (cumulative raised figure), accessed Apr
   2026. https://www.fjordfoundry.com/
6. Bitbond, "Fjord Foundry Review," Sep 2025.
7. Messari, "State of Metaplex" quarterly reports; Metaplex Foundation
   blog monthly roundups, 2023-2025.
   https://www.metaplex.foundation/blog/
8. CoinMarketCap, "What is DAO Maker."
   https://coinmarketcap.com/alexandria/article/what-is-dao-maker
9. DefiLlama, "Flaunch Protocol," accessed Apr 2026.
   https://defillama.com/protocol/flaunch
10. Polkastarter blog, "How to Participate in a Polkastarter IDO" and
    "All You Need to Know About POLS Power."
    https://blog.polkastarter.com/how-to-participate-in-a-polkastarter-ido/
11. Fjord Foundry, "LBP FAQ."
    https://help.fjordfoundry.com/fjord-foundry-docs/for-sale-creators/faqs-creators/lbp-faq
12. DAO Maker, GitBook documentation.
    https://dao-maker-1.gitbook.io/dao-maker
13. Polkastarter fee details partially inferred from CryptoRank and
    platform documentation; exact percentage not prominently published.
14. Yahoo Finance, "Pump.fun Quarterly Revenue," Dec 2025; AInvest,
    Jan 2026.
15. DAO Maker, "DYCO: Dynamic Coin Offering."
    https://learn.daomaker.com/dyco
16. Metaplex, "Genesis: Launch Pool" developer documentation.
    https://developers.metaplex.com/smart-contracts/genesis
17. Flaunch documentation and Bankless, "How to Launch Better
    Memecoins on Flaunch."
    https://docs.flaunch.gg/ ;
    https://www.bankless.com/read/how-to-launch-better-memecoins-on-flaunch
18. Polkastarter, "How to Participate in a Polkastarter IDO."
    https://blog.polkastarter.com/how-to-participate-in-a-polkastarter-ido/
19. EU Markets in Crypto-Assets Regulation (MiCA), right of withdrawal
    provisions; Shoosmiths, "Consumer Protection for Crypto Assets."
    https://www.shoosmiths.com/insights/articles/heating-up-consumer-protection-for-crypto-assets
20. US Regulation CF, 17 CFR §227.
    https://www.ecfr.gov/current/title-17/chapter-II/part-227
21. ChainCatcher, "daos.fun: The next liquidity black hole?" Oct 2024;
    Bitget News, "daos.fun: The next liquidity black hole?" Oct 2024;
    Blocmates, "Daos.fun is Shaking Up the Crypto Memecoin Space,"
    Jan 2025.
    https://www.chaincatcher.com/en/article/2149146
