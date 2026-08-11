# Appendix: Bridges and Wrapped Tokens

This appendix surveys the ecosystem context behind
[RFP-021](../RFPs/RFP-021-wrapped-erc20.md), a privacy-preserving wrapped ERC-20
bridge for LEZ, in two parts. The first and larger part surveys major
cross-chain bridge exploits, classifies their root causes, and sources the loss
figures relevant to RFP-021's claim that bridges are the most-attacked category
of DeFi infrastructure, and to its design choice to eliminate signer/validator
trust while still treating verification-logic correctness and upgrade-key
custody as first-order risks. The second, shorter part sources RFP-021's
stablecoin market-size claim, which motivates the specific tokens (USDC, USDT,
DAI, WETH) the RFP proposes wrapping first.

## Summary Table

Ordered by date. USD figures are as reported at or near the time of the hack;
where sources disagree, a range is given (see per-hack sections for detail).

| Hack                           | Date             | Amount (USD, as reported)                                                | Root cause category                                                                                        | Source    |
| ------------------------------ | ---------------- | ------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------- | --------- |
| Poly Network                   | 10 Aug 2021      | ~$611M (range $600M-$613M)                                               | (b) Verification/access-control bug (self-granted authority)                                               | [7][8][9] |
| THORChain (Bifrost/ETH Router) | 15 & 22 Jul 2021 | ~$8M + ~$8M (two incidents)                                              | (b) Smart contract / verification logic bug                                                                | [27][28]  |
| Qubit Finance (QBridge)        | 27-28 Jan 2022   | $80M                                                                     | (b) Smart contract / verification logic bug                                                                | [23][24]  |
| Wormhole                       | 2 Feb 2022       | ~$325M (range $320M-$326M)                                               | (b) Smart contract / verification logic bug                                                                | [4][5]    |
| Meter.io (Meter Passport)      | 5 Feb 2022       | ~$4.3M-$4.4M                                                             | (b) Smart contract / verification logic bug                                                                | [25][26]  |
| Ronin Network / Axie Infinity  | 23 Mar 2022      | ~$625M (range $540M-$625M)                                               | (a) Validator/multisig key custody compromise                                                              | [1][2]    |
| Harmony Horizon Bridge         | 23 Jun 2022      | ~$100M                                                                   | (a) Validator/multisig key custody compromise                                                              | [20][21]  |
| Nomad                          | 1 Aug 2022       | ~$190M (range $190M-$200M)                                               | (b) Verification logic bug, introduced via a legitimate upgrade                                            | [6][10]   |
| BNB Bridge / BSC Token Hub     | 7 Oct 2022       | ~$570M minted (range $566M-$586M); ~$100M-$110M actually moved off-chain | (b) Smart contract / verification logic bug                                                                | [17][18]  |
| Multichain                     | 6-7 Jul 2023     | ~$125M-$126M                                                             | (c) Admin/custody key compromise (MPC custody, single controlling party)                                   | [14][15]  |
| Orbit Chain                    | 31 Dec 2023      | ~$81M-$82M (range $81M-$86M)                                             | (a) Validator/multisig key custody compromise (disputed; operator cites insider/infrastructure compromise) | [29][30]  |

## Per-Hack Detail

### Poly Network (10 August 2021)

Poly Network's `EthCrossChainManager` contract accepted attacker-supplied
cross-chain transaction data and could invoke arbitrary functions via
`_executeCrossChainTx`, including privileged functions on the
`EthCrossChainData` contract that `EthCrossChainManager` owned. The attacker
crafted a call that invoked `EthCrossChainData.putCurEpochConPubKeyBytes()`, a
function intended only for legitimate keeper-rotation consensus, through this
unintended call chain, reassigning the "keeper" (the address authorized to
approve cross-chain fund releases) to an address they controlled. They then used
that self-granted authority to authorize withdrawals across Ethereum, BNB Smart
Chain, and Polygon. SlowMist's and BlockSec's independent analyses both state
explicitly that this was not a leaked private key [8][9]. Reported losses range
from $600.3M to $613M depending on the token-price snapshot used across the
three affected chains; $611M is the most frequently cited figure [7]. Nearly all
funds were returned by the attacker within about two weeks.

This sits at the boundary between categories (b) and (c): no credentials were
compromised, so it is fundamentally an access-control/verification bug, but its
*effect* (self-appointing as the sole trusted signer) is functionally equivalent
to what a stolen keeper or governance key would achieve.

### THORChain Bifrost/ETH Router (15 and 22 July 2021)

Two separate incidents in the ETH Router bridging component of THORChain, a
cross-chain liquidity network rather than a pure lock-and-mint bridge. In both
cases, attacker-deployed token contracts fed fake metadata to the router,
tricking it into refunding real assets for fake or manipulated deposits.
THORChain's own post-mortem confirms both as router logic bugs, not key
compromises [27][28]. Losses were approximately $8M in each incident. Included
here as a borderline case: it is a cross-chain bridging component being
exploited via a logic bug, but THORChain's overall architecture is an
AMM/liquidity network, not a lock-and-mint bridge in the RFP-021 sense.

### Qubit Finance / QBridge (27-28 January 2022)

QBridge exposed a legacy `deposit()` function alongside a newer `depositETH()`
function. The attacker called the legacy function with the ETH resource ID but
supplied zero actual ETH and a malformed token address; the contract's
transfer-verification logic did not check that a real transfer had occurred, so
it emitted the same deposit event the BSC-side relayer used to mint qXETH. This
minted 77,162 qXETH against no real collateral, which the attacker then used as
loan collateral to drain approximately $80M in ETH, BTC-B, stablecoins, and
other tokens [23][24]. A pure smart contract logic bug; no key or upgrade
mechanism was involved.

### Wormhole (2 February 2022)

The Solana-side Wormhole program used a deprecated helper,
`load_instruction_at`, to verify that a `secp256k1_program`
signature-verification instruction preceded a mint instruction. The deprecated
function did not check that the "instructions sysvar" account passed to it was
Solana's genuine sysvar account. The attacker supplied a spoofed sysvar account
pre-populated with fabricated data mimicking a valid signature-verification
instruction, and the program accepted this forged input as proof that the
required guardian signatures had validated a Verified Action Approval (VAA).
This let the attacker mint 120,000 wETH on Solana with no real ETH backing on
Ethereum [4][5]. USD figures range from $320M to $326M depending on the ETH
price used at time of reporting; the token amount (120,000 wETH) is the more
stable figure. The bug was patched by switching to
`load_instruction_at_checked`. This is a verification-logic bug in
already-deployed program code, unrelated to any contract upgrade or key
compromise.

### Meter.io / Meter Passport (5 February 2022)

Meter Passport's auto-wrap/unwrap convenience feature for native gas tokens
(ETH, BNB) did not properly restrict direct interaction with the wrapped ERC-20
contracts, nor verify that a matching real value transfer had occurred, letting
the attacker mint WETH/WBNB without depositing real collateral [25][26]. The
direct loss to Meter was approximately $4.3M-$4.4M (1,391 ETH and 2.74 BTC),
with an additional ~$3.3M cascading loss at the dependent lending protocol
Hundred Finance. Same general failure family as Qubit: an event-driven relayer
trusting an event that did not correspond to a real transferred value.

### Ronin Network / Axie Infinity Bridge (23 March 2022)

Ronin's bridge required 5-of-9 validator signatures to authorize withdrawals.
The attacker compromised five validator keys: four Sky Mavis-run validators via
a single senior engineer, who was compromised through a fake LinkedIn job offer
whose "offer" PDF carried spyware, giving the attackers a foothold into Sky
Mavis's infrastructure and its four validator keys; and a fifth signature from
the third-party Axie DAO validator, obtained not by directly breaching Axie DAO
but because Axie DAO had granted Sky Mavis a temporary gas-fee allowlist
delegation in November 2021 (to handle demand load) that was never revoked. The
attackers' already compromised systems used this still-active delegated
permission to sign on Axie DAO's behalf [1][2]. Reported losses are 173,600 ETH
plus 25.5M USDC, most commonly valued at $625M at the time; figures as low as
$540M appear in some contemporaneous reporting depending on the ETH price used.
The exploit occurred on 23 March 2022 and was discovered on 29 March 2022, when
a user's 5,000 ETH withdrawal failed. This is a direct compromise of validator
signing keys that custody bridge funds via multisig consensus; no contract bug
or upgrade mechanism was involved. Ronin suffered a second, unrelated incident
in August 2024 via an authorized-upgrade bug (see "Upgrade-key compromise: what
the evidence shows," below).

### Harmony Horizon Bridge (23 June 2022)

The Ethereum-side Horizon bridge used a multisig requiring only two signatures
(commonly cited as 2-of-5) to authorize withdrawals. Per Harmony's own incident
summary, attackers ran a multi-stage intrusion: phishing and social engineering
against developers to plant malware, which gave visibility into internal
communications and non-public bridge infrastructure code, culminating in
backdoor access to privileged servers where keys were generated or decrypted on
the fly. This gave the attackers enough plaintext private keys to meet the
two-signature threshold and authorize eleven fraudulent withdrawal transactions
[20][21]. Losses were approximately $100M across ETH, WBTC, USDC, USDT, BUSD,
and other tokens. The attack was later attributed by Elliptic and the FBI to the
Lazarus Group, with funds laundered via Tornado Cash. A direct compromise of
fund-movement signing keys; no smart contract logic was exploited.

### Nomad (1 August 2022)

During a routine, authorized contract upgrade on 21 April 2022, Nomad's team
initialized the `Replica` contract's trusted-root storage by calling
`initialize()` with `0x00` set as an acceptable root. In Solidity, unset mapping
entries also default to `0x00`. Because the `process()` function treated any
message whose root matched an entry in the acceptable-root mapping as valid, and
`0x00` was both the deliberately "trusted" value and the default value for an
unproven message, every unproven message hash was automatically treated as
proved. This let the first attacker submit an arbitrary spoofed message to drain
funds; because the exploit calldata was trivially copy-pasteable (swap the
recipient address), it triggered a chaotic free-for-all involving roughly 960
transactions from hundreds of opportunistic addresses on 1 August 2022 [6][10].
Losses are commonly cited at $190M, with some sources rounding to $200M
depending on how traced/recovered funds are accounted for.

No upgrade key or admin key was stolen or misused by an external attacker. The
vulnerability was introduced by Nomad's own team through a legitimate,
authorized upgrade with a catastrophic initialization error — a distinct failure
mode from a compromised or misused upgrade key. See "Upgrade-key compromise:
what the evidence shows," below, for why this distinction matters.

### BNB Bridge / BSC Token Hub (7 October 2022)

The BSC Token Hub validated cross-chain messages using an IAVL Merkle proof
scheme. The verification code failed to enforce that a proof node had no forged
"right" child: it computed the root hash using only the "left" field and never
checked for an injected "right" attribute. The attacker took a legitimate
historical proof, injected a crafted "right" node containing a forged payload
claiming a deposit that never happened, and the verifier's root-hash comparison
passed because the malicious right-child data was never included in the hash it
validated against. This let the attacker forge two withdrawal proofs and mint
1,000,000 BNB each (2,000,000 BNB total) with no real backing deposit [17][18].
This is a pure verification-logic bug (the BNB Chain team named it
"Dragonfruit"); no keys were stolen or misused. The minted amount is commonly
valued at $566M-$586M at the time (~$570M is the most frequently cited
midpoint); of that, only about $100M-$110M was actually moved off-chain before
validators halted the network roughly 90 minutes after the exploit began, with
about $7M frozen immediately and the remainder effectively immobilized once the
attacker's address was blacklisted.

### Multichain (6-7 July 2023)

Multichain used an MPC (multi-party computation) custody scheme for bridge
funds. Multichain's CEO, Zhaojun He, was reportedly detained by Chinese police
in Kunming on 21 May 2023, roughly six weeks before large, unauthorized outflows
began. Outflows occurred in waves starting 6-7 July 2023: approximately $58M in
USDC, 1,023.8 WBTC, and 7,214 wETH left the Fantom bridge contract within 30
minutes, followed by smaller outflows from Dogechain and Moonriver bridge
contracts. Multichain's own team stated they had lost access to the MPC node
servers and could not reach Zhaojun after his arrest [14]. No smart contract
vulnerability has been identified; security-firm analysis states the pattern
does not obviously relate to any previously audited vulnerability. Total losses
are commonly cited at $125M-$126M. A Singapore High Court ruling, in a suit
brought by Fantom against Multichain, found the breach was possible because
Zhaojun held ultimate privileges and control over the custodied assets, though
the court declined to find that he or Multichain actually diverted the funds
[15]; causation remains legally unresolved even though centralized operational
control over the MPC custody scheme is established. This is best classified as
an admin/custody key compromise: the MPC key shares functioned as the
operational custody mechanism for bridge funds, and a single controlling party
(or someone with access to his systems) retained effective unilateral control,
undermining the "multi-party" security assumption the scheme depended on.

### Orbit Chain (31 December 2023)

Independent security researchers and Rekt.news attributed this exploit to
compromise of 7-of-10 multisig signer keys on Orbit's Ethereum vault, likely via
social engineering [29]. Orbit's developer, Ozys, published an official
statement on 25 January 2024 explicitly denying both a smart contract
vulnerability and the theft of a validator key, instead attributing the root
cause to a former CISO who "arbitrarily changed firewall policies" shortly
before departing the company (resigned 6 December, exploit occurred 31 December)
[30] — implying insider sabotage or infrastructure compromise rather than direct
key theft. The two accounts are not fully mutually exclusive: a compromised
firewall could be the vector through which signer keys were exposed. This is a
genuine, unresolved disagreement between independent researchers and the
project's own statement, noted here rather than resolved. Losses are reported in
the $81M-$86M range, with $81.5M the most commonly cited figure. No mention in
any source of an upgradeable proxy or upgrade key; the funds were held directly
in a multisig-controlled vault.

## Upgrade-key compromise: what the evidence shows

RFP-021 treats an upgradeable verifier contract's *upgrade key* as a key-custody
attack surface of the same kind as a validator or multisig key, one layer
removed: whoever holds the upgrade key can substitute the verification logic
itself, rather than directly signing a fraudulent withdrawal. This is a distinct
category from every hack tabulated above, none of which involved an upgrade key
being stolen or misused by an external attacker. Two related but distinct
patterns emerge from the historical record, and they should not be conflated:

**Pattern A: a legitimate, authorized upgrade introduces a catastrophic bug.**
No key is stolen; the team that legitimately controls the upgrade key exercises
it correctly, but the new logic is broken. Nomad (above) is the clearest bridge
example: a routine upgrade set a trusted root to a value that collided with the
default "unproven" value. A second, later example from the same project family
is instructive: Ronin suffered a second, unrelated incident in August 2024,
distinct from the March 2022 validator-key compromise. During a `BridgeManager`
contract upgrade, the team skipped calling `initializeV3()`, leaving an internal
`_totalOperatorWeight` value at zero, which disabled the minimum-vote-weight
check and let a single signature pass verification where several should have
been required. An MEV bot found the bug and withdrew $12M, later returned as a
whitehat disclosure for a $500K bounty [19]. Both cases show that the
bug-introduction risk from a mutable, upgradeable verifier is real and has
caused (Nomad) or nearly caused (Ronin 2024) major losses, even absent any
attacker gaining control of the upgrade key itself.

**Pattern B: an attacker manufactures upgrade or governance authority through a
separate bug**, rather than stealing the legitimate key outright. Audius (July
2022, a DeFi governance protocol, not a bridge) is the clearest example: a
storage-slot collision between the proxy's admin slot and OpenZeppelin's
`Initializable` flag let an attacker call `initialize()` again on an
already-initialized proxy and seize governance control, which they then used to
push a malicious governance action draining approximately $6M in AUDIO tokens
[11][12]. Wormhole had a near-miss of the same kind: an uninitialized UUPS proxy
would have let an attacker call `initialize()`, set themselves as guardian, and
push a malicious upgrade; this was caught and responsibly disclosed through
Wormhole's bug bounty program before any exploitation, with a $10M bounty paid
and no funds lost [13]. Poly Network (above) is a bridge-specific variant of
this same pattern: no key was stolen, but a verification bug let the attacker
self-grant an authority (the "keeper") that functions as the bridge's trusted
signer set, achieving an effect equivalent to a stolen governance key without
stealing one.

**No documented case of a completed bridge hack via a genuinely stolen or
phished upgrade key was found.** The clearest examples of an upgrade key or
admin key being directly compromised and used to push malicious logic are
outside the bridge category: PAID Network (March 2021), where an attacker
compromised the single private key controlling proxy upgrade rights, upgraded to
malicious logic, and minted approximately 59.5M tokens for a loss of roughly $3M
[16]; and Wasabi Protocol (April 2026), a perpetuals protocol where an attacker
compromised the deployer EOA holding sole admin rights (no multisig or
timelock), granted itself an admin role, and executed unauthorized upgrades on
vault proxies, draining approximately $4.5M-$5.5M across three chains. Both are
DeFi protocols, not cross-chain bridges. Security-firm commentary (Trail of
Bits' 2020 review of Aave's upgradeability, and general framing from Immunefi
and OpenZeppelin) treats post-deployment upgrade mechanisms as a
disproportionate source of protocol losses as a general class, but without a
bridge-specific incident list to cite.

The practical reading for RFP-021's argument: the *category* of risk (a mutable
verifier controlled by a key that can swap in malicious or broken logic) is
real, attested by security-firm framing, and has caused actual bridge losses
through Pattern A (a legitimate upgrade shipping a bug, Nomad and Ronin 2024)
and a bridge-specific near-miss of Pattern B (Wormhole's uninitialized proxy,
caught before exploitation). It has not, in the documented record, caused a
bridge loss through the narrower scenario of an attacker stealing an upgrade key
outright, though that exact pattern has occurred in non-bridge DeFi (PAID
Network, Wasabi Protocol). An immutable program with an explicit migration path
removes the upgrade key from the attack surface entirely, closing off both
patterns rather than only the one with a clean historical precedent.

## Root-cause breakdown

Excluding THORChain (a borderline case, not a lock-and-mint bridge in the
RFP-021 sense) and treating Poly Network and Multichain per their categorization
above, the ten core hacks in the summary table break down as follows by dollar
value (approximate, using the midpoint of any reported range):

- **(a) Validator/multisig key custody compromise:** Ronin (~$625M), Harmony
  Horizon (~$100M), Orbit Chain (~$81.5M, disputed). Subtotal: **~$806.5M**.
- **(b) Smart contract / verification logic bug:** Wormhole (~$323M), Nomad
  (~$190M), BNB Bridge (~$576M minted, ~$105M actually exfiltrated), Qubit
  Finance (~$80M), Meter.io (~$4.35M), Poly Network (~$611M, boundary case).
  Subtotal using minted/at-risk BNB Bridge figure: **~$1.784B**; using the
  amount actually moved off-chain for BNB Bridge instead: **~$1.313B**.
- **(c) Admin/governance key compromise:** Multichain (~$125.5M, boundary case
  with (a) given MPC custody). Subtotal: **~$125.5M**.

This does not cleanly reproduce a "half key custody, half verification bugs"
split in dollar terms across this specific set: by these figures,
verification-logic bugs account for a larger share of gross value at risk than
key-custody compromises, driven heavily by the BNB Bridge minted amount and Poly
Network's near-$611M figure, both of which are boundary cases (BNB Bridge
because most of the minted value was never actually moved off-chain; Poly
Network because the bug's effect mimicked a stolen governance key without one
being stolen). If Poly Network is instead counted toward key custody (its effect
being equivalent to a stolen keeper key) and BNB Bridge's actually-exfiltrated
figure is used rather than its minted figure, category (a) plus Poly Network's
$611M would dominate instead. The underlying point RFP-021 makes — that key
custody is a large but not exclusive source of bridge losses, and
verification-logic bugs are a comparably large source — holds directionally
across every reasonable way of drawing the boundary cases; a precise 50/50 split
should not be asserted as an exact figure, because it depends heavily on how BNB
Bridge's minted vs. exfiltrated amount and Poly Network's boundary case are each
counted.

## Chainalysis figures: what the primary sources actually say

RFP-021 cites Chainalysis for the claim that over $2B was stolen from
cross-chain bridges in 2022, representing 64% of that year's DeFi hacking
losses. Chainalysis in fact published two separate figures, from two separate
reports, that are often conflated:

- **"Cross-Chain Bridge Hacks Emerge as Top Security Risk"** (2 August 2022)
  \[3\]: "Chainalysis estimates that $2 billion in cryptocurrency has been
  stolen across 13 separate cross-chain bridge hacks... Attacks on bridges
  account for 69% of total funds stolen in 2022 so far." This is a
  **partial-year** figure (through early August 2022), and the 69% is a share of
  **all** cryptocurrency theft in that period, not specifically of DeFi losses.
- **"2022 Biggest Year Ever for Crypto Hacking"** (1 February 2023) \[3\]: "DeFi
  protocols as victims accounted for 82.1% of all cryptocurrency stolen by
  hackers — a total of $3.1 billion... And of that $3.1 billion, 64% came from
  cross-chain bridge protocols specifically." This is the **full-year** figure,
  and the 64% is a share of **DeFi-specific** losses ($3.1B), not of all crypto
  theft. 64% of $3.1B is approximately $1.98B, close to but not identical to the
  earlier $2B figure, which came from a different, partial-year,
  differently-denominated report.

Both figures are directionally consistent and both support the underlying claim
that bridges were the single largest category of DeFi hacking losses in 2022.
But the $2B dollar figure and the 64% percentage, as commonly paired in
secondary reporting (including news aggregation that RFP-021's figure likely
traces to), actually come from two different Chainalysis reports measuring
different populations over different time windows, not from a single, internally
consistent statistic. The more defensible citation is the February 2023 report's
64%-of-2022-DeFi-losses figure on its own, or the August 2022 report's
$2B/69%-of-total-crypto-theft figure on its own, rather than pairing the two
numbers as if from one source.

### Trend since 2022

Chainalysis does not appear to publish a recurring, directly comparable
bridge-hack-specific percentage in its subsequent annual reports; coverage
shifted toward broader categories such as private-key compromises and
centralized-exchange hacks, and toward bridges as a money-laundering conduit
(chain-hopping) rather than as the primary attack vector. Total crypto hacking
losses (all categories, not bridge-specific) reported by Chainalysis in
subsequent years:

- **2023:** approximately $1.7B, down roughly 54% year-over-year [31].
- **2024:** approximately $2.2B, up roughly 21% year-over-year [32]; the largest
  bridge-specific incident that year was Orbit Chain (~$81.8M), a small fraction
  of the total.
- **2025 (mid-year):** more than $2.17B stolen by mid-July 2025, already
  exceeding all of 2024, though dominated by the Bybit exchange hack (~$1.5B, a
  private-key compromise, not a bridge exploit) rather than by bridge-specific
  losses [33].

The overall pattern is a marked decline in bridge hacks specifically as a share
of total DeFi/crypto losses since the 2022 peak, even as total crypto theft has
fluctuated and, by mid-2025, exceeded 2022 levels again driven by other
categories (notably exchange and private-key compromises).

## Stablecoin Market Context

RFP-021 cites USDT and USDC together holding "over 80%" of a stablecoin market
"above $300B in mid-2026" as the rationale for prioritising those two assets
(alongside DAI and WETH) as the bridge's first wrapped tokens. DefiLlama tracks
total stablecoin market capitalisation at approximately $310B-$313B through
mid-2026, having peaked above $321B earlier in the year before shedding roughly
$10B by late July [34][35]. The "above $300B" framing is accurate as a floor,
though it understates the market by omitting the more precise $310B+ figure
available from the same tracker.

Tether (USDT) is the largest stablecoin by a wide margin, with a market cap of
approximately $183.4B and roughly 59-64% of total stablecoin supply depending on
the exact date sampled [36][37]. Circle's USDC is second, at approximately
$72B-$75B, roughly 23-24% of supply [37][38]. Combined, USDT and USDC hold
approximately 89% of total stablecoin market capitalisation as of early August
2026 [36][37] — comfortably clearing RFP-021's "over 80%" claim rather than
merely meeting it.

Supply share is not the same claim as settlement dominance, and the two should
not be conflated. On supply, USDT leads by a wide margin. On on-chain DEX
settlement volume specifically, the picture has shifted materially during 2026:
USDC accounted for roughly 70-80% of adjusted stablecoin transaction volume in
H1 2026, ahead of USDT's roughly 25%, a reversal from historical patterns where
USDT dominated volume [39][40]. Curve and Uniswap remain the primary venues for
USDC/USDT pairs, and stablecoin swaps make up the majority of DEX trading volume
on Ethereum [39]. On lending markets, Aave, the largest lending venue by
deposits, reportedly holds an 80%+ share of USDT and USDC deposits/borrows on
Ethereum, with stablecoin pools forming the protocol's largest liquidity base
[41]. Taken together, this supports RFP-021's framing that USDT and USDC are the
settlement asset most DeFi money markets and DEXes actually run on, though the
volume and collateral-share figures are drawn from secondary market trackers and
platform-level reporting rather than a single authoritative cross-protocol
study, and should be read as directionally strong rather than precisely
measured.

## References

01. [Halborn — Explained: The Ronin Hack (March 2022)](https://www.halborn.com/blog/post/explained-the-ronin-hack-march-2022)
02. [Ronin/Sky Mavis — Back to Building: Ronin Security Breach](https://roninchain.com/blog/posts/back-to-building-ronin-security-breach-6513cc78a5edc1001b03c364)
03. [Chainalysis — Cross-Chain Bridge Hacks Emerge as Top Security Risk](https://www.chainalysis.com/blog/cross-chain-bridge-hacks-2022/)
    (2 Aug 2022, partial-year, $2B/69%-of-total-theft figure); see also
    [Chainalysis — 2022 Biggest Year Ever for Crypto Hacking](https://www.chainalysis.com/blog/2022-biggest-year-ever-for-crypto-hacking/)
    (1 Feb 2023, full-year, 64%-of-DeFi-losses figure)
04. [Halborn — Explained: The Wormhole Hack (February 2022)](https://www.halborn.com/blog/post/explained-the-wormhole-hack-february-2022)
05. [CertiK — Wormhole Bridge Exploit Incident Analysis](https://www.certik.com/blog/wormhole-bridge-exploit-incident-analysis)
06. [Immunefi — Hack Analysis: Nomad Bridge, August 2022](https://medium.com/immunefi/hack-analysis-nomad-bridge-august-2022-5aa63d53814a)
07. [Halborn — Explained: The Poly Network Hack (August 2021)](https://www.halborn.com/blog/post/explained-the-poly-network-hack-august-2021)
08. [SlowMist — The Root Cause of Poly Network Being Hacked](https://slowmist.medium.com/the-root-cause-of-poly-network-being-hacked-ec2ee1b0c68f)
09. [BlockSec — The Initial Analysis of the PolyNetwork Hack](https://blocksecteam.medium.com/the-initial-analysis-of-the-polynetwork-hack-270ac6072e2a)
10. [Halborn — The Nomad Bridge Hack: A Deeper Dive](https://www.halborn.com/blog/post/the-nomad-bridge-hack-a-deeper-dive)
11. [Audius — Audius Governance Takeover Post-Mortem](https://blog.audius.co/article/audius-governance-takeover-post-mortem-7-23-22)
12. [ChainLight — Audius Variant Scanner: Scanning Storage Collisions Between Ethereum Contracts](https://blog.chainlight.io/en-audius-variant-scanner-scanning-storage-collisions-between-ethereum-contracts-4d2d64b77566)
13. [Immunefi — Wormhole Uninitialized Proxy Bugfix Review](https://immunefi.com/blog/bug-fix-reviews/wormhole-uninitialized-proxy-bugfix-review/)
14. [Halborn — Explained: The Multichain Hack (July 2023)](https://www.halborn.com/blog/post/explained-the-multichain-hack-july-2023)
15. [DL News — Singapore Court Ruling Fuels View Multichain Hack Was Inside Job](https://www.dlnews.com/articles/defi/singapore-court-fuels-view-multichain-hack-was-inside-job/)
16. [Halborn — Explained: The PAID Network Hack (March 2021)](https://www.halborn.com/blog/post/explained-the-paid-network-hack-march-2021)
17. [Halborn — Explained: The BNB Chain Hack (October 2022)](https://www.halborn.com/blog/post/explained-the-bnb-chain-hack-october-2022)
18. [Immunefi — Hack Analysis: Binance Bridge, October 2022](https://immunefi.com/blog/bug-fix-reviews/hack-analysis-binance-bridge-october-2022/)
19. [Halborn — Explained: The Ronin Network Hack (August 2024)](https://www.halborn.com/blog/post/explained-the-ronin-network-hack-august-2024)
20. [Halborn — Explained: The Harmony Horizon Bridge Hack](https://www.halborn.com/blog/post/explained-the-harmony-horizon-bridge-hack)
21. [Harmony Community Forum — Summary of the Horizon Bridge Incident](https://talk.harmony.one/t/summary-of-the-horizon-bridge-incident/20990)
22. [Elliptic — The Harmony Horizon Bridge Hack](https://www.elliptic.co/resources/harmony-horizon-bridge-hack)
23. [Halborn — Explained: The Qubit Hack (January 2022)](https://www.halborn.com/blog/post/explained-the-qubit-hack-january-2022)
24. [SlowMist — Our Analysis of the $80M Qubit Finance Exploit](https://slowmist.medium.com/our-analysis-of-the-80m-qubit-finance-exploit-b0f272cd8c25)
25. [Halborn — Explained: The Meter.io Hack (February 2022)](https://www.halborn.com/blog/post/explained-the-meter-io-hack-february-2022)
26. [Cointelegraph — Latest DeFi Bridge Exploit Results in $4.4M Losses for Meter](https://cointelegraph.com/news/latest-defi-bridge-exploit-results-in-4-4m-losses-for-meter)
27. [Halborn — Explained: The THORChain Hack (July 2021)](https://www.halborn.com/blog/post/explained-the-thorchain-hack-july-2021)
28. [THORChain — Post-Mortem: ETH Router Exploits 1 & 2, and Premature Return to Trading](https://medium.com/thorchain/post-mortem-eth-router-exploits-1-2-and-premature-return-to-trading-incident-2908928c5fb)
29. [Rekt.news — Orbit Bridge Rekt](https://rekt.news/orbit-bridge-rekt)
30. [Orbit Chain — Official Statement Regarding Orbit Bridge Exploit](https://medium.com/orbit-chain/official-statement-regarding-orbit-bridge-exploit-551928f3dc52)
31. [Chainalysis — Stolen Crypto Falls in 2023](https://www.chainalysis.com/blog/crypto-hacking-stolen-funds-2024/)
32. [Chainalysis — $2.2 Billion Stolen in Crypto in 2024](https://www.chainalysis.com/blog/crypto-hacking-stolen-funds-2025/)
33. [Chainalysis — 2025 Crypto Crime Mid-Year Update](https://www.chainalysis.com/blog/2025-crypto-crime-mid-year-update/)
34. [DefiLlama — Stablecoins](https://defillama.com/stablecoins) — total
    stablecoin market cap and per-issuer breakdown, tracked live
35. [Bitcoin Foundation — Stablecoin Market Cap Tops $321B, Extending 2026 Growth](https://bitcoinfoundation.org/news/stablecoin-news/stablecoin-market-cap-tops-321b/)
36. [CoinLaw — Tether Statistics 2026](https://coinlaw.io/tether-statistics/) —
    USDT market cap and supply-share figures, early August 2026
37. [The Motley Fool — Which Stablecoins Are the Largest and Most Popular](https://www.fool.com/research/largest-stablecoins/)
    — USDT/USDC combined market-share figure
38. [USDC.org — USDC Supply Stats](https://usdc.org/stats) — USDC circulating
    supply and market cap
39. [KuCoin — USDC Accounts for 70% of Adjusted Stablecoin Volume in H1 2026](https://www.kucoin.com/news/flash/usdc-accounts-for-70-of-adjusted-stablecoin-volume-in-h1-2026)
40. [CoinDesk — Circle's USDC Is Leaving Tether Behind in the Stablecoin Volume Race](https://www.coindesk.com/business/2026/07/06/circle-s-usdc-is-leaving-tether-behind-in-the-stablecoin-volume-race)
41. [Aave — Aave Is Infrastructure for Scaling Stablecoins](https://aave.com/blog/stablecoin-infrastructure)
    — Aave stablecoin deposit/borrow share commentary

______________________________________________________________________

*This appendix was prepared to support RFP-021. For clarification or additions,
please use the RFP repository Discussions.*
