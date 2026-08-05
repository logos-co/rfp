# Appendix: Private Voting Ecosystem

This appendix surveys how governance voting works today across DAOs and
privacy-focused chains, and what existing private voting systems actually
deliver. It serves as a reference for
[RFP-006](../RFPs/RFP-006-private-voting.md), providing context on the observed
behaviour of public voting, the cryptographic approaches that have been tried,
their trust assumptions, and their deployment status.

This document describes what exists. It does not propose a design, make
recommendations, or draw conclusions about what should be built on Logos.

Research current as of August 2026. Where a figure is self-reported by a vendor,
comes from an aggregator rather than a primary source, or could not be verified,
this is stated inline.

## Systems surveyed

Systems are ordered by deployment maturity (most widely deployed first) and this
order is maintained throughout the document.

| System                  | Operator                | Layer                       | Privacy achieved                       | Trust assumption                | Status (Aug 2026)                  |
| ----------------------- | ----------------------- | --------------------------- | -------------------------------------- | ------------------------------- | ---------------------------------- |
| Snapshot (baseline)     | Snapshot Labs           | Off-chain signed messages   | None (fully public)                    | None beyond signature validity  | Dominant off-chain venue [1]       |
| Shutter shielded voting | Shutter + Snapshot Labs | Off-chain, threshold crypto | Delayed reveal only                    | Sub-threshold keyper collusion  | Live since Oct 2022 [2][3]         |
| Vocdoni (Vochain)       | Vocdoni                 | Purpose-built voting chain  | Anonymous census, encrypted ballots    | Census/CSP operator, validators | Superseded by DAVINCI [4][5]       |
| Vocdoni DAVINCI         | Vocdoni                 | zkRollup on Ethereum        | Ballots never individually decrypted   | Sequencer threshold majority    | Testnet; mainnet targeted [5]      |
| Secret Network (SEFI)   | SecretSwap              | Application on Secret       | Encrypted ballots, aggregate published | Intel SGX TEE                   | Launched 2021; current use n/v [6] |
| Penumbra                | Penumbra Labs           | Base-layer consensus        | Delegator choice hidden; weight leaks  | Cryptographic (zk)              | Live, mainnet [7][8]               |
| Semaphore               | PSE (Ethereum Fdn)      | Primitive / library         | Signaller anonymity; message public    | Groth16 trusted setup           | Maintenance mode [9][10]           |
| MACI                    | PSE (Ethereum Fdn)      | EVM contracts + circuits    | Vote hidden from all but coordinator   | Honest-but-curious coordinator  | v2 stable, v3 in progress [11][12] |
| MACI Aragon plugin      | Aragon + PSE            | Aragon OSx plugin           | As MACI                                | As MACI                         | Bespoke deployments [13]           |
| Oasis Sapphire / OPL    | Oasis                   | Confidential EVM            | Encrypted ballots, result returned     | Intel SGX TEE                   | Available; scale n/v [14]          |
| Nouns sprint prototypes | 3 funded teams          | Research                    | Varies (see section 6)                 | Varies                          | Not deployed [15][16]              |
| Helios / Belenios       | Academic                | Off-chain web voting        | Encrypted ballots, verifiable tally    | Threshold trustees (+registrar) | Deployed outside crypto [17][18]   |

`n/v` denotes not verified during this survey.

## 1. Baseline: how DAO governance voting works today

### 1.1 Votes are public and permanently attributable

The default across every major DAO governance venue is that a vote is a signed
message or transaction that permanently links a voting address to a choice.
Snapshot votes are off-chain signed messages with voting power computed from
token balances at a snapshot block; on-chain governors such as Compound Governor
Bravo record the vote as a transaction. In both cases the linkage between
address and choice is public and permanent.

A peer-reviewed dataset of Snapshot covering October 2020 to November 2023
records approximately 19,200 spaces, 127,000 proposals, and 51 million votes
[1]. This is the largest citable measurement of the space; platform marketing
figures circulating for 2025 and 2026 could not be traced to a primary source
and are not cited here.

### 1.2 Turnout is low, and the measured rate depends heavily on denominator

Reported DAO voter turnout ranges across more than an order of magnitude
depending on what is used as the denominator:

| Measurement                                 | Turnout                     | Source |
| ------------------------------------------- | --------------------------- | ------ |
| Study of 50 DAOs, against all token holders | 1.77%                       | [19]   |
| Eligible-versus-active ratio, recent votes  | 1--2%                       | [19]   |
| Cross-DAO average                           | 6.3%                        | [19]   |
| Compound / Uniswap / ENS / Gitcoin          | 34% / 31.4% / 39.2% / 28.6% | [20]   |
| Internet Computer SNS DAOs                  | ~64%                        | [20]   |

Turnout measured against all token holders yields low single digits; measured
against delegated or otherwise eligible voting power it reaches 30% and above.
For comparison, [19] cites 70--80% participation in traditional corporate
shareholder voting.

The consistent structural pattern across sources is low turnout combined with
overwhelming majorities: most proposals are decided by a small group with little
recorded opposition.

### 1.3 Voting power is concentrated

A review across DAOs found the single largest actor holding 35% of voting power
and the top three holding 63% collectively [19]. In CompoundDAO, 32 whale
addresses out of 2,482 unique voting addresses, each holding between 100,000 and
10 million COMP, are sufficient to materially sway outcomes [21].

### 1.4 Delegation concentrates influence further, and institutionalises public voting

A study of delegation networks across five protocols with full history to 19
June 2025 [22] reports:

| DAO      | Addresses in delegation network | Voting-power Gini | Nakamoto coefficient (50%) | Top delegate in-degree |
| -------- | ------------------------------- | ----------------- | -------------------------- | ---------------------- |
| ENS      | 115,600                         | 0.99              | 1                          | 6,614                  |
| Uniswap  | 49,926                          | 0.99              | 16                         | 454                    |
| Compound | 15,230                          | 0.99              | 13                         | 344                    |
| Aave     | 6,397                           | 0.94              | not reported               | 357                    |

The study's headline finding is that delegation consolidates influence in the
hands of a few highly visible actors rather than dispersing it. A Nakamoto
coefficient of 1 for ENS means a single delegate holds a majority of
participating voting power.

This is corroborated inside ENS governance itself. A July 2026 draft proposal to
delegate 5M ENS states that one delegate has enough quorum to execute any
proposal and to outvote the next 50 delegates combined, and notes participation
decline: proposals that once drew 3M votes have more recently struggled to meet
quorum [23].

Delegated governance also formalises the expectation that votes are public.
Uniswap's DAO Principles, executed as Proposal 78 on 30 January 2025 with
41,610,289 votes for and fewer than 1 against, states that rationales for votes
should be published in a manner understandable to the broader community, that
delegates should remain open to scrutiny, and that conflicts of interest should
be disclosed on delegate pages [24]. The Uniswap Delegate Reward Initiative
attaches payment to this expectation, setting a seven-day deadline from the end
of each vote for delegates to publish their rationale [25]. Optimism and
Arbitrum publish comparable delegate expectations [26][27].

Optimism has approximately 288 active delegates, with a further 346 inactive and
366 classified as "ghost" among the top 1,000 by voting power, and the top 10
delegates controlling approximately 48% of voting power [28]. This figure comes
from a third-party dashboard without a stated date.

### 1.5 Vote markets depend on votes being publicly verifiable

Several protocols operate open marketplaces where voting power is rented. These
are relevant to this survey for a mechanical reason: payment is conditional on a
vote the buyer can independently verify, which is only possible because votes
are public.

Votium's briber documentation instructs buyers to enter the number of tokens
rewarded to voters and permits setting a maximum number of tokens paid per
vlCVX, warning that rewards capped below market rate may receive fewer votes
[29]. This is an explicit price per unit of voting power. Distribution goes to
addresses that delegated to Votium or voted for incentivised pools on Snapshot
[30], conditioning payout on the recorded public vote.

Named incentive buyers on Votium include Curve, Frax, and PayPal. A cumulative
figure of $301.07 million across 69 rounds, averaging $4.36M per round, is
quoted by a third party citing Votium [31]; it could not be confirmed at source.
Hidden Hand, operated by Redacted Cartel, has been reported at over $35 million
in incentives with a 4% protocol fee [32][33]; this figure is partial and
undated. Paladin operates vote-renting and delegation-optimisation across Quest,
Votium, and Hidden Hand [34]. Bribe yields on veCRV reportedly exceeded 50% APY
at peak, though current volumes are a fraction of 2021 levels and Curve
emissions decline roughly 16% each August [35]; these are analyst commentary
rather than audited data.

## 2. Documented governance attacks involving vote acquisition

The following incidents are multi-source corroborated. They are included because
they document, with dates and figures, how voting power has been acquired and
used.

**Beanstalk, 17 April 2022, $182M.** The attacker flash-loaned over $1B in DAI,
USDC, USDT, BEAN, and LUSD from Aave, Uniswap V2, and SushiSwap, converted it to
Stalk to exceed 67% voting power, and invoked `emergencyCommit` to execute
malicious proposals BIP-18 and BIP-19 within a single transaction. Net profit
was approximately $80M; BEAN depegged from roughly $1 to $0.11 [36][37][38].
Voting power was acquired atomically within one block, with no capital at risk
over time.

**Build Finance DAO, 9--10 February 2022, approximately $470k.** A first
malicious proposal was spotted by a moderator and voted down. The attacker moved
tokens to a fresh wallet and resubmitted; the second proposal was not picked up
by the Discord notification bot and passed unnoticed. The attacker minted BUILD
tokens, drained Balancer and Uniswap liquidity along with 130,000 METRIC from
the treasury, and laundered approximately 160 ETH [39][40][41]. This was a
monitoring failure rather than a cryptographic one.

**Tornado Cash, 20 May 2023, approximately 483,000 TORN.** The attacker deployed
a proposal claiming to be identical to a previously approved one. It contained a
`selfdestruct` and redeploy path granting 1.2 million votes across attacker
addresses against roughly 70,000 legitimate votes, achieving complete governance
capture. On 26 May the attacker's own proposal 21 restoring governance passed
517,000 votes to zero [42][43][44][45]. Voters approved malicious code because
they trusted a claimed-identical proposal.

**Mango Markets, 11 October 2022, approximately $116M.** Avraham Eisenberg
manipulated MNGO perpetuals, holding approximately 488M of roughly 500M
circulating tokens, to inflate collateral and drain the protocol. He then
submitted a governance proposal offering to return funds in exchange for a
bounty and voted for his own proposal using tokens acquired in the exploit,
casting over 33 million votes. That proposal failed quorum [46][47][48]. This is
the most heavily documented case of an attacker voting with illegitimately
acquired tokens, carrying SEC and CFTC filings as primary sources.

**Compound Proposal 289, passed 28 July 2024, 499,000 COMP (approximately
$24M).** A bloc known as the "Golden Boys," led by the actor known as Humpy,
accumulated COMP on the open market and passed a proposal moving approximately
5% of the treasury into their goldCOMP product. The tally was 682,191 for and
633,636 against. A Compound security advisor linked several accumulating
accounts; the same actor had previously used multiple wallets to control over
50% of vote share in a 2022 Balancer dispute. The proposal was later rescinded
in exchange for a staking product [49][50][51]. Voting power here was acquired
legitimately on the open market, which distinguishes this case from the
flash-loan attacks above.

**Arbitrum AIP-1, March--April 2023.** The Arbitrum Foundation sought
ratification for 750M ARB (approximately $1B). Approximately 50M ARB had already
been moved on-chain to an "Administrative Budget Wallet" weeks before the vote,
with 40M loaned to a market maker and 10M converted to fiat. The Foundation
subsequently characterised AIP-1 as a ratification rather than a proposal.
Follow-up AIP-1.05, which would have returned 700M ARB, failed with 118M ARB
against, representing 84.01% of votes [52][53][54][55]. This is a process and
legitimacy dispute rather than an exploit.

## 3. Documented behavioural effects of public voting

### 3.1 The receipt-freeness argument

The foundational analysis is "On-Chain Vote Buying and the Rise of Dark DAOs" by
Daian, Kell, Miers, and Juels (IC3, 2 July 2018) [56]. Its core claim is that
transparency inverts into vulnerability: because votes are cryptographically
verifiable and permanently recorded, a voter can prove how they voted to a
buyer. This defeats the protection a secret ballot provides, where a voter can
take payment and vote otherwise. The paper frames two classical e-voting
requirements that on-chain voting breaks: **receipt-freeness** (a voter cannot
prove how they voted) and **coercion resistance** (a voter can credibly lie
about how they voted). The authors argue all permissionless voting systems where
users generate their own keys are vulnerable to this class of attack. They
define a "Dark DAO" as a bribery cartel using TEEs such that nobody, including
the DAO's creator, can determine the number of participants, the amount pledged,
or the precise logic of the attack.

A 2023 follow-up, "DAO Decentralization: Voting-Bloc Entropy, Bribery, and Dark
DAOs" [57], published at USENIX Security 2025 [58], introduces the Voting-Bloc
Entropy (VBE) metric and reports the first working Dark DAO prototype, built
using TEEs on Oasis Sapphire to attack Ethereum DAOs. One of its results is that
the risk of systemic bribery increases with increasing DAO decentralization.

Note that this literature cuts in both directions: privacy is the defence
against verifiable vote buying, and privacy technology (specifically TEEs) is
also what makes covert vote-buying cartels feasible.

### 3.2 Measured voting biases at scale

"Voting Biases in Decentralized Autonomous Organization (DAO) Governance"
(Balietti, Saggese, Strohmaier, 10 July 2026) [21] analyses the Snapshot dataset
described in section 1.1. Measured association with voting-power share:

| Bias                     | Effect   |
| ------------------------ | -------- |
| Author-selected choice   | +58.8 pp |
| Approval-oriented choice | +27.1 pp |
| First-listed choice      | +7.7 pp  |

On visibility, the authors note that Snapshot votes and intermediate results are
often publicly visible during the voting period, making DAO voting a natural
environment for information cascades, and observe that late voters are more
likely to support leading alternatives. Their stated conclusion is that
ordering, author signals, and vote visibility should be treated as institutional
design choices rather than neutral implementation details.

The authors state a limitation directly relevant here: their regression does not
isolate visibility effects from other mechanisms, and their data lacks the
timing granularity to separate information cascades from alternatives. The
position, author, and approval effects are measured; the herding-from-live-tally
claim is motivated but not cleanly identified.

### 3.3 Evidence gaps

No rigorous quantitative study was found that isolates herding caused by live
tallies, or that measures last-minute whale vote timing. The observation that
whales vote late to avoid revealing position early is widely repeated in
commentary [59] but is not, as far as this survey found, quantified in any
study. Vendor commentary from privacy providers naming bandwagoning, apathy, and
last-minute advantage as mechanisms [60] cites no data and is not treated as
evidence here.

## 4. Delayed-reveal systems: Shutter on Snapshot

Shutter Network and Snapshot Labs launched shielded voting on 13 October 2022
[2][61].

**Mechanism.** Keypers, a permissioned node set, run distributed key generation
to produce an eon keypair; the public key is broadcast and the secret exists
only as shares. Per-proposal epoch keys derive from the eon key using the
proposal identifier, so voters compute the epoch public key locally with no
keyper interaction required to vote. After voting closes, keypers publish
decryption keys and votes are revealed.

**Property provided.** Fairness and information symmetry during the voting
window. There are no early results, so no visible tally to follow, and no
last-minute swing once the direction of the tally is apparent.

**Property not provided.** Ballot secrecy after the fact. Once the proposal
closes, votes are decrypted and individual voter-to-choice linkage becomes
public and permanent. This is a delayed-reveal ballot, not a secret ballot, and
it does not defend against retaliation, vote buying, or coercion after the vote
concludes. Shutter's own later material states that public voting undermines
governance through retaliation, pressure, vote-buying, and strategic signalling
over honest conviction [3].

**Trust assumptions.** Privacy holds only if fewer than the threshold of keypers
collude; a colluding threshold could decrypt early. The keyper set is not fully
decentralized, and the original release was labelled a beta. A liveness risk
exists in the other direction: votes could become permanently undecryptable if
too few keypers participate. Keypers cannot censor votes or influence an ongoing
proposal.

**Adoption.** Shutter reports 881 or more DAOs protected, 372,914 or more votes
encrypted, three years live with no privacy breaches, and 87% of DAOs still
using it after one year [62]. These figures are vendor self-reported. Named
adopters include the Arbitrum Foundation, ShapeShift DAO, Layer2DAO, MoonDAO,
SporkDAO, Doodles DAO, Bankless DAO, Alpaca Finance, ParaSwap, Token Engineering
Academy, and Giveth. ShapeShift adopted in December 2022 and has run over 70
proposals with it. Governance discussions on adoption also occurred at Euler
[63], Stake DAO [64], and ENS [65]; the outcome of each was not verified.

**Successor design.** On 21 October 2025 Shutter announced Permanent Shielded
Voting, which addresses the after-the-fact linkage gap. It uses linear
threshold-homomorphic ElGamal with zero-knowledge proofs: voters encrypt choices
and prove in ZK that the ciphertext encodes a valid option, encrypted votes are
homomorphically aggregated into an encrypted tally, and keypers
threshold-decrypt only the aggregate, so individual votes remain private
permanently [3]. At announcement the status was proof-of-concept complete in a
forked Snapshot UI, with testnet integration pending. Whether this reached
mainnet by August 2026 was not verified.

## 5. Never-decrypt systems

### 5.1 Vocdoni and DAVINCI

Vocdoni is a voting infrastructure provider historically backed by the Aragon
Association. For several years it ran voting on Vochain, a purpose-specific
voting L1. The current architecture is DAVINCI, a voting-specific zkRollup
settling on Ethereum [5].

**Design.** A zkSNARK-based state machine run by a decentralized network of
off-chain sequencers, settling on Ethereum and other EVM chains, using EIP-4844
data blobs for data availability. Elections are defined on Ethereum, votes are
cast and aggregated off-chain by sequencers, and only verified proofs are
committed on-chain [5].

**Cryptography.** Threshold homomorphic ElGamal for ballot encryption, so that
individual votes are never decrypted and only aggregate results are computed.
zkSNARKs operate at three levels: vote proofs establishing ballot validity
without revealing the choice, state transition proofs establishing correct
aggregation, and results proofs establishing correct final tally. The
implementation uses the gnark framework with EdDSA over BN254 for CSP census
proofs, and optional GPU acceleration via Icicle marked experimental [66].

**Census models.** Two distinct mechanisms exist. A ZK census proof uses a
Merkle proof showing the voter's zkCensusKey is a leaf of the census tree,
proven in zero knowledge so membership is established without revealing the
secret key [67][68]. Alternatively, a Credential Service Provider evaluates
eligibility per user during the process and issues a Chaumian blind signature
over the voter address and process ID, so the CSP cannot link the credential to
the cast ballot [69][70]. The CSP is trusted for eligibility gating even though
blinding prevents it from deanonymising the vote.

**Results during voting.** Ballots are encrypted throughout and decryption
requires collaboration among a majority of sequencers, so no single party can
access results prematurely [5].

**Deployments.** Òmnium Cultural, a large Catalan cultural organisation, runs
governance for over 180,000 members; one reported election drew 6,723 votes with
international observers, using SMS two-factor authentication, blind signatures,
and a voting-receipt code [71]. Vocdoni has partnered with Decidim, the
participatory-democracy platform [72]. FC Barcelona fan-club elections have been
reported as using the technology [73], though scale figures were not
independently verified. Turnout-increase claims on Vocdoni's testimonials page
are self-reported marketing and are not cited as evidence here.

**Status.** Active and in transition. Public testnet launched June 2025;
davinci-node currently runs on Sepolia with active development, and public
mainnet was targeted for Q3 2026 [5][66].

### 5.2 Penumbra

Penumbra is a Cosmos-ecosystem, IBC-connected shielded chain whose governance is
modelled on the Cosmos Hub with yes, no, and abstain options, but with delegator
ballot privacy built into the base-layer protocol [7][8].

**Asymmetric privacy model.** Validator votes are transparent actions, signed by
and attributable to the specific validator, and act as default votes for their
delegators. Delegator votes are anonymous but not fully unlinkable: per the
specification, a delegator vote reveals only the voting power used in the vote
and the validator the voting delegator had delegated to. The vote choice is
unlinked from identity, but the voting weight and the delegated validator both
leak, which reduces the anonymity set when a validator has few delegators [7].

**Snapshot.** Voting power is fixed at proposal start: a delegator's power
equals the unbonded staking token value of the delegation tokens they had staked
to an active validator when the proposal started voting. A delegator vote is a
spend proof for a delegation note plus an inclusion proof that the note's
creation height was strictly before voting started, which blocks buying voting
power after seeing a proposal [7].

**Double-vote prevention.** A delegator vote reveals the nullifier for the note
used to justify the vote. The node keeps per-proposal nullifier sets for voting,
entirely distinct from the main nullifier set, so voting does not spend or
encumber the note [7].

**Proposal types.** Four: signaling with no mechanised effect, emergency which
concludes at one-third yes and can halt the chain, parameter change which
applies only if old parameters exactly match current state, and community pool
spend. Community pool spend has been disabled via chain parameter pending
community consensus. Mainnet quorum is 40% [8].

**Tallies are public.** The `pcli query governance proposal [ID] tally` command
returns totals across all validators along with a per-validator breakdown [74].
Privacy protects the individual delegator's choice, not the aggregate. Neither
validators nor delegators can change a vote once cast.

Penumbra ratified its mainnet via a parameter-change proposal, and Liquidity
Tournament funding proposals of approximately 150,000 UM from the community pool
have been proposed. A community governance frontend exists at vote.penumbra.zone
[75]; the host did not respond during this survey, so the full proposal history
and 2026 tallies were not verified.

### 5.3 Zcash coinholder polling

Zcash runs two parallel governance tracks with materially different privacy
properties.

**ZCAP (Zcash Community Advisory Panel), via Helios.** The NU7 Priorities poll
of March 2025 ran on a hosted Helios instance covering seven questions on ZSAs,
the Network Sustainability Mechanism, Memo Bundles, and Sprout pool removal
[76]. This is one-person-one-vote among panel members rather than coin-weighted,
and the voter roll is public. Helios provides ballot encryption and
verifiability, not anonymity of participation.

**Coinholder polls, coin-weighted.** Two mechanisms appear in the sources and
this survey could not determine which was used for each specific poll. The
distinction matters because their privacy properties are close to opposite.

The transparent-address mechanism documented in `zec-coin-polling` [77] is
substantially public despite using shielded memos: voters hold ZEC in a
*transparent* voting address, a snapshot at a cut-off height sets weight, and
the ballot is sent as a shielded memo to a vote-reception z-address whose
viewing key is deliberately published so anyone can verify tallies. The README
states that voting preferences associated with each transparent address are
revealed to anyone with the vote reception viewing key, that the amount posted
for weight is publicly visible on-chain, and that the user must accept the risk
of associating their voting preference with their posted ZEC amount. This design
achieves public verifiability, not voter privacy.

The Orchard-note mechanism, integrated into Zodl 3.5.0 and announced 28 May
2026, allows users to keep funds shielded at all times with no need to move
funds before or after voting, using Orchard notes to prove balance [78][79].
Zodl states that votes are private and funds never leave the wallet, with
Keystone hardware signing and support for delegation of voting power. The
protocol's design and implementation sections were not retrievable, so nullifier
usage and exactly what is publicly revealed were not verified.

**Results.** The June 2025 NU7 coinholder poll saw 7.25% participation of
circulating ZEC, with ballot counts from 210 on Tachyon down to 55 on Consensus
Accounts. The Zcash Foundation cautioned that a near-unanimous result from 55
ballots is a different signal than one from 210 ballots [80]. STARK Proof
Verification via TZEs split the two tracks sharply: 56.8% ZCAP support against
83.6% coinholder opposition. The November 2025 Coinholder-Directed Retroactive
Grants Program approved 5 of 9 proposals [81]. ZIP 1016 ended the Direct Funding
model at NU6 in November 2024, preserving 8% for Zcash Community Grants and
routing 12% to a protocol-controlled lockbox that coinholders direct, through
the third halving in late 2028 [82].

## 6. Coordinator-based systems: MACI and its integrations

### 6.1 MACI

MACI (Minimal Anti-Collusion Infrastructure) is maintained by Privacy & Scaling
Explorations with Ethereum Foundation support, originating from a 2019 proposal
by Vitalik Buterin [83]. It is the most widely referenced private voting
architecture in the EVM ecosystem.

**Flow.** A voter registers a public key in the MACI contract, with a gatekeeper
contract checking eligibility and the voter receiving voice credits. A
coordinator creates a poll with start and end times. The voter encrypts their
vote client-side using ECDH between their own private key and the coordinator's
public key, and posts the encrypted message on-chain. After voting, the
coordinator merges messages, decrypts them off-chain, generates zkSNARK proofs
of message-processing validity and correct tallying, and submits the proofs
on-chain, where anyone can verify them. The final tally is revealed; individual
votes remain encrypted [11].

**Cryptography.** ECDH with the Poseidon cipher from zk-kit for vote encryption;
Groth16 zkSNARK circuits written in Circom and compiled via circomkit, with
`processMessages` and `tallyVotes` as the key circuits; Poseidon hashing
throughout; and LazyIMT incremental Merkle trees for signup and message storage,
replacing the earlier AccQueue structure in v2 [11].

**Trusted setup.** Groth16 requires a phase-2 trusted setup ceremony. MACI v1.2
held a ceremony at ceremony.pse.dev; v2.0 required a new ceremony due to circuit
changes. Tooling is p0tion. The coordinator must download ceremony zkey
artifacts before generating proofs [11].

**Security properties as claimed by the project** \[84\]:

| Property             | Mechanism                                                     |
| -------------------- | ------------------------------------------------------------- |
| Collusion resistance | zkSNARK proof of valid vote without revealing it              |
| Receipt-freeness     | Encrypted messages; coordinator private key needed to decrypt |
| Privacy              | ECDH encryption with the coordinator's public key             |
| Uncensorability      | Circuit enforces that all valid messages are processed        |
| Unforgeability       | EdDSA signature on messages, verified in-circuit              |
| Non-repudiation      | Append-only message chain; a later vote supersedes an earlier |
| Correct execution    | zkSNARK proof verified on-chain                               |

**Coordinator trust.** The coordinator can decrypt all votes, cannot censor
valid votes because the circuit enforces full processing, and cannot produce a
false tally because the proof is verified on-chain. Privacy and collusion
resistance therefore hold only against an honest-but-curious coordinator. A
coordinator colluding with a briber breaks vote privacy. Decentralizing the
coordinator is the project's stated central open research problem, with MPC,
homomorphic encryption, and TEE approaches all listed as research rather than
shipped [11][85].

**Version history** \[85\]:

| Version | Date      | Key changes                                                                                           |
| ------- | --------- | ----------------------------------------------------------------------------------------------------- |
| v1.0    | 2021      | Original release; ECDH plus zkSNARK; clr.fund integration                                             |
| v1.1.1  | ~2022     | Trusted setup ceremony, circuit improvements                                                          |
| v1.2.0  | Feb 2024  | circomkit and zk-kit adoption, non-QV voting, EAS gatekeeper, critical censorship bug fix             |
| v2.0.0  | Mid 2024  | Removed Topup and Subsidy, LazyIMT trees, concurrent polls, new gatekeepers, Hardhat tasks, PSE audit |
| v2.5.0  | Late 2024 | Tally results struct, Anon Aadhaar gatekeeper, separate proof generation and submission tasks         |
| v3.0    | Planned   | Per-poll voice credits and gatekeeping, vote hash-chains, off-chain relayer voting                    |

A censorship bug allowing the coordinator to selectively censor votes was found
in v1.x and fixed in v1.2 [86]. The v2.0 release passed PSE's internal audit
with no significant issues [87].

**Gatekeepers.** Eligibility is pluggable: FreeForAll, EAS, Hats Protocol,
Gitcoin Passport, Zupass, Semaphore, and Anon Aadhaar gatekeepers all ship in
`maci-contracts` [88].

**Components.** MACI is a pnpm monorepo spanning circuits, contracts, core state
machine, crypto primitives, domain objects, CLI, SDK, subgraph, a NestJS
coordinator service exposing REST endpoints for deployment, merging, proof
generation and submission, and a relayer service that accepts encrypted votes
off-chain and posts them on behalf of voters, hiding the voter's sending address
[88][89].

**Scaling.** The project reports 10,000 concurrent voters achieved in 2023, with
100,000 in progress for 2026 and 1M targeted for 2028. Techniques under
exploration include SNARK folding schemes and group-wise matching for quadratic
funding [85].

**Integrations.** clr.fund for quadratic funding, the Gitcoin Allo stack,
Aragon, PriVote, and PSE's own maci-platform [85].

### 6.2 MACI Aragon OSx plugin

Aragon and the MACI team built an official MACI voting plugin for Aragon OSx,
distinct from the earlier Nouns PoC [13][90].

A DAO member calls `createProposal` on the plugin, which checks the caller holds
sufficient governance tokens and creates a MACI poll, capturing the current
block number minus one as the eligibility snapshot. Members register once via
the Aragon frontend, with the MACI public key generated on-device from their
wallet. Members then vote yes, no, or abstain through the Aragon interface with
votes end-to-end encrypted, and can change their vote at any point during the
voting period. After the proposal ends the coordinator service processes ballots
off-chain, generates a zkSNARK proof of correct tally, and submits it for
on-chain verification; execution proceeds via OSx permissions if quorum and
support are met.

The plugin is EVM-native, with supported networks including Ethereum mainnet,
Arbitrum, Optimism, Scroll, and Linea. MACI voting is more expensive than plain
token voting; costs are described as negligible on L2 and higher on mainnet
[91].

Status is bespoke builds for DAOs that need private voting now, with general
availability as a standard OSx plugin stated as future work rather than shipped
[13].

## 7. Research prototypes: Nouns DAO Private Voting Research Sprint

Nouns DAO issued a call for proposals in February 2023 via Prop House. Twenty
proposals were submitted; the top three by Nouner vote each received a 70,000
USDC grant for a roughly three-month sprint. Proposals needed to place top three
and clear 300 votes. Funded by Nouns proposal 216 [15][16].

**Team 1: Poseidon, DeFROST** [92]. FROST (Flexible Round-Optimized Schnorr
Threshold Signatures) for committee distributed key generation without a trusted
dealer, combined with threshold homomorphic encryption over the Baby JubJub
curve. Two phases: a two-round DKG, then encrypted voting with t-of-n committee
tally. Ballots are encrypted but voter addresses remain public, a deliberate
choice to preserve existing Nouns UX and support delegation and multisigs,
motivated by roughly 55% of Nouns holders delegating and roughly 15% of voting
power sitting in multisig contracts. Trust assumption is an honest-but-curious
committee with privacy holding if fewer than t members collude. Prototype
circuits and contracts were published; a production UI was not delivered.

**Team 2: Aragon ZK Research with Aztec Labs** [93][94][95][96]. Aztec
implemented Ethereum storage proofs in Noir; AZKR designed the voting scheme on
top. The approach proves ownership of *a* Noun without revealing which Noun.
Components were a zkRegistry mapping Ethereum addresses to BabyJubJub public key
commitments derived deterministically from a wallet signature, a Voting Smart
Contract, Noir circuits for vote and tally proofs, and Timelock.zone for
time-locked ballot encryption.

The vote circuit proves signature of NFT ownership, signature of vote choice,
nullifier uniqueness as the hash of the ownership signature, encryption
correctness under Diffie-Hellman with the timelock key, vote validity in
{0,1,2}, NFT ownership via a storage path from the token storage root,
zkRegistry registration via a second path, and delegation status via a third
[95].

Measured costs from the technical report: registration approximately 45k gas,
process creation approximately 700k gas, vote submission approximately 690k gas,
tally proof submission approximately 522k gas. Vote generation took
approximately 12 minutes per ballot on an i7 U-series laptop with 32GB RAM.
Tallying took approximately 5 minutes for up to 16 NFTs at 106k constraints, and
approximately 2 hours for 256 NFTs at 1.5M constraints, on a laptop [95].

Stated limitations: CLI-only, no weighted voting (one vote per NFT), no multisig
support, no delay-relayer service, manual storage-root matching, and no audit
[95][96].

Timelock.zone implements TLCS (Time Lock Cryptographic Service), which derives
future key pairs from Drand beacon rounds published by the League of Entropy.
Public keys for future periods are available immediately while private keys
become computable only after the corresponding Drand rounds pass. Security
requires at least one honest participant in key generation, and participants
need not be online at reveal. Two variants exist: zk-TLCS requiring ZK proofs of
correct parameter generation, and standard TLCS without them, the latter used in
production. A Cosmos chain, `tlcs-chain`, was built to decentralize the service
[97][98].

This is a materially different fairness model from MACI: the timelock key is
mathematically unavailable before the release round, whereas a MACI coordinator
holds the decryption key from the start and is trusted not to use it early.

**Team 3: Vortex, by team 水 (Mizu)** [99]. ZK proofs with Merkle tree updates,
structured as a main scheme with a fallback scheme, a registration mechanism,
optimistic validation, and an optimistic pool design for vote aggregation.
Delivered as a design exposition rather than a working system.

**Outcome.** None of the three was deployed to Nouns DAO production governance.
All produced research reports and proofs-of-concept. Nethermind Research
subsequently proposed an independent evaluation of all three to advise Nouns on
next steps [16]; the outcome of that evaluation was not verified. Mach34
submitted a Noir-based proposal but was not among the three winners. Cicada
(a16z crypto), MACI, and Vocdoni were not sprint grantees, though a16z's Cicada
research later referenced the Vortex team's optimistic proof checker.

## 8. Primitives and adjacent systems

### 8.1 Semaphore

Semaphore is a general anonymous-signalling primitive maintained by PSE, created
around 2019--2020, with v4 current [9][100]. It provides three functions: create
private identities, add identities to groups, and send anonymous signals.

An identity commitment is derived by hashing the identity secret and stored as a
leaf in an incremental Merkle tree (LeanIMT with dynamic depth in v4). A
nullifier derives from the user's secret combined with a scope, so each proof
yields a unique nullifier and double-signalling within a scope is detectable
while anonymity is preserved; in voting the scope would be the ballot
identifier. A Groth16 zkSNARK proves that the user is a member of the group and
that the same user created both the message and the proof [101][102].

**Property.** Anonymity of the signaller within the anonymity set of the group,
plus one-signal-per-scope enforcement. The signal itself is public, so applied
directly to voting the vote choice is visible while the voter is hidden, which
is the mirror image of Shutter's property. Semaphore has no native notion of
weighted voting; one identity produces one signal.

**Trust and limits.** Groth16 trusted setup, with the circuit compiled for a
MAX_DEPTH range of 1--32 during its ceremony. Anonymity is bounded by group
size. Tree depth 1--32 in v4 implies a theoretical ceiling of 2^32 members. v4's
dynamic depth means groups grow without a pre-declared size. Transferring an
off-chain 1,000-member group on-chain cost approximately $8,000 under v3 versus
under $1,000 under v4 [103]. A documented per-proof on-chain verification gas
figure was not found; the benchmarks page renders numbers as images.

**Status.** Maintained but not actively developed; PSE describes it as monitored
for bug fixes rather than under feature development, with repository activity
through December 2025 [10][100].

### 8.2 TEE-based systems

**Secret Network.** Chain governance on Secret uses the standard Cosmos SDK gov
module and is public: 1 staked SCRT equals 1 vote, seven-day voting period,
33.4% quorum, 50% threshold, 33.4% veto, four options, with delegators
inheriting validator votes unless they vote themselves [104]. Private voting on
Secret exists at the application layer via secret contracts. SEFI Governance
(SecretSwap) launched on mainnet 27 July 2021, billed as the first private
voting application on a public blockchain, where only staked SEFI could vote,
individual votes stayed private, and aggregate distribution was published after
the period [6]. The trust assumption is Intel SGX rather than pure cryptography.
Whether SEFI governance remains active in 2026 was not verified.

**Oasis Sapphire and the Oasis Privacy Layer.** Sapphire is a confidential EVM
runtime whose confidentiality rests on TEEs. The Oasis Privacy Layer lets a DAO
on another chain keep its core contract in place and delegate balloting: a
secret-ballot contract on Sapphire receives a proposal over a message-passing
bridge, runs the vote confidentially, and returns only the final result to the
home chain [14]. Sources for this are Oasis's own blog posts and a demo; no
independent confirmation of a major production DAO using it at scale was found.

Note the dual use documented in section 3.1: Oasis Sapphire is also the platform
on which the working Dark DAO bribery prototype was built [57][58].

### 8.3 Chains with private execution but public governance

Several privacy-focused chains run transparent, address-attributable governance
despite privacy-capable stacks.

**Aztec.** Contracts are written in Noir using the Aztec.nr framework with dual
private and public state, and privacy logic executes client-side with proofs
generated locally [105]. Aztec's own governance is explicitly public: a
three-stage AZIP to AZUP to on-chain voting pipeline where upgrades are proposed
in public and debated in the open, sequencers signal quorum first, then
tokenholders vote, and sequencer staked voting power defaults to "yea" requiring
active opposition [106]. Private voting appears as a demonstrated use case only
at prototype grade: `quadratic_voting_noir` is a demo circuit in which ballots
are hidden from all but a trusted ballot manager [107], and the NounsDAO work
with Aragon described in section 7 was a research sprint [108]. Alpha Mainnet
launched following a governance vote; TGE occurred 12 February 2026.

**Mina.** MIP voting is fully public and address-attributable. Votes are cast by
sending oneself a transaction with a keyword in the memo field, and all vote
transactions can be viewed on Mina block explorers, weighted by stake with a
staking-ledger snapshot two epochs before voting [109]. The Mesa Upgrade (MIPs
6--9) passed via on-chain vote 8--15 December 2025 [110].

**Aleo.** Governance for ARCs runs through the Aleo Foundation Governance
Platform, launched May 2024, where ALEO holders vote on protocol changes [111].
Whether platform voting is private was not verified; the specific proposal page
checked returned HTTP 404. Private voting dApps in Leo exist as developer
examples [112][113], with at least one documented as deployed to testnet rather
than mainnet.

### 8.4 Academic and off-chain systems

**Helios** is an open-source academic web voting system, used for the Zcash ZCAP
polls described in section 5.3. It provides ballot encryption and end-to-end
verifiability with a public bulletin board. Formal analysis found that deployed
Helios versions 2.0 and 3.1.4 do not satisfy ballot secrecy or universal
verifiability, though a proposed Helios'16 variant does, and that Helios is
vulnerable to ballot stuffing [17]. Trust rests on trustees holding threshold
decryption key shares.

**Belenios** is a Helios derivative whose principal advantage is eligibility
verifiability and ballot-stuffing resistance, achieved by adding a registrar
issuing voter signing credentials. It satisfies both ballot secrecy and
universal verifiability. BeleniosRF extends it to non-interactive
receipt-freeness [18]. Trust splits across registrar and decryption trustees;
collusion between them breaks privacy.

**Property mechanics from the literature** [114][115]. Everlasting privacy is
built from perfectly hiding commitments plus NIZKs, so privacy survives future
cryptanalysis of the encryption scheme. Receipt-freeness is realised via mixnets
or homomorphic tallying. The two standard tally methods are additively
homomorphic, which is simple vote addition and best suited to yes/no ballots,
and randomizable or mixnet-based, which is needed for ranked or write-in
ballots. Coercion resistance and participation privacy remain open research
problems for both systems.

## 9. Cross-cutting observations

These are patterns visible across the systems surveyed above. They are
descriptive, not prescriptive.

**Two distinct privacy boundaries are in production, and they are commonly
conflated.** Delayed-reveal systems (Shutter's original scheme) hide the tally
during voting and publish every ballot afterwards. Never-decrypt systems
(Vocdoni DAVINCI, Shutter Permanent, Poseidon's DeFROST) open only the
aggregate. Only the latter defends against retaliation, coercion, or post-hoc
vote buying. The ecosystem is visibly migrating toward the latter, and
independently converging on the same primitive: threshold-homomorphic ElGamal
with ZK ballot-validity proofs.

**Almost every production-grade shielded voting system rests on a threshold
committee.** Shutter's keypers, Vocdoni's sequencers, MACI's coordinator, and
Poseidon's DeFROST committee are all variants of the same assumption. The
designs that avoid it entirely (AZKR/Aztec's timelock construction, Vortex)
remain unshipped research. Penumbra and the Zcash Orchard-note mechanism are the
two surveyed cases that place ballot privacy in base-layer consensus rather than
in a committee.

**Privacy-focused chain does not imply private governance.** Secret Network,
Mina, and Aztec all run transparent, address-attributable governance despite
privacy-capable execution stacks.

**Trust assumptions bifurcate cleanly into cryptographic and hardware.**
Cryptographic approaches (Penumbra, Zcash, MACI, Vocdoni, Aztec) rest on zk
proofs with no trusted operator for correctness. Hardware approaches (Secret via
SGX, Oasis via Sapphire) rest on TEE integrity. The TEE systems shipped working
private voting years earlier; the zk systems carry weaker assumptions but
thinner production deployment.

**Public tallies are universal.** Every system surveyed publishes aggregate
results. The privacy boundary is always the individual ballot, never the
outcome. Penumbra additionally publishes a per-validator breakdown.

**No production system achieves receipt-freeness or coercion resistance in the
formal sense.** Penumbra's voter can prove how they voted. Zcash's
transparent-address mechanism actively publishes the linkage. MACI's
receipt-freeness holds only against a coordinator who does not collude with the
briber. Only the academic literature (BeleniosRF) addresses this directly, and
it is not deployed on-chain anywhere this survey found.

**Delegation and ballot privacy are in structural tension.** Ballot privacy
protects ordinary token holders from coercion, bribery, and herding, while
delegated governance depends on delegates being publicly accountable for how
they vote, an expectation that Uniswap, Optimism, and Arbitrum have all
formalised and that Uniswap financially incentivises. Penumbra is the surveyed
system that most explicitly separates the two, making validator votes
transparent and attributable while keeping delegator choices private. The Nouns
sprint's Poseidon team made the opposite trade for the same reason, keeping
voter addresses public specifically to preserve delegation and multisig support.

**The privacy literature cuts both ways.** Privacy is the documented defence
against verifiable vote buying, and TEE-based privacy is also what makes covert
vote-buying cartels feasible. Both claims come from the same research group, and
the Dark DAO prototype was built on a confidential-EVM platform marketed for
confidential DAO voting.

## References

[1] S. Balietti, P. Saggese, M. Strohmaier, "Voting Biases in Decentralized
Autonomous Organization (DAO) Governance," arXiv 2607.09435, 10 July 2026
(dataset description). https://arxiv.org/abs/2607.09435

[2] Shutter Network, "Shutter brings shielded voting to Snapshot," 13 October
2022\. https://blog.shutter.network/shutter-brings-shielded-voting-to-snapshot/

[3] Shutter Network, "Permanent Shielded Voting is coming to Snapshot," 21
October 2025.
https://blog.shutter.network/permanent-shielded-voting-is-coming-to-snapshot/

[4] Vocdoni blog, DAVINCI universal voting protocol announcement.
https://vocdoni.io/blog/davinci-universal-voting-protocol/ (page body was not
retrievable through redirects during this survey; details cross-checked against
[5] and [66])

[5] DAVINCI protocol site. https://davinci.vote/

[6] Secret Network, "SEFI Governance live on mainnet: private voting," 27 July
2021\. https://ghost.scrt.network/sefi-governance-live-mainnet-private-voting/

[7] Penumbra protocol specification, Governance.
https://protocol.penumbra.zone/main/governance.html

[8] Penumbra guide, Governance overview.
https://guide.penumbra.zone/overview/gov

[9] Semaphore v4.0.0 release.
https://github.com/semaphore-protocol/semaphore/releases/tag/v4.0.0

[10] PSE projects, Semaphore. https://pse.dev/projects/semaphore

[11] MACI documentation, Introduction. https://maci.pse.dev/docs/introduction

[12] MACI GitHub repository.
https://github.com/privacy-scaling-explorations/maci

[13] Aragon blog, "Private onchain voting on Aragon with MACI."
https://blog.aragon.org/private-onchain-voting-on-aragon-with-maci/

[14] Oasis, "Oasis 101: Confidential DAO Voting."
https://oasis.net/blog/oasis-101-confidential-dao-voting

[15] Nouns Private Voting Research Sprint specification.
https://hackmd.io/@el4d/BkDBV1Pso ;
https://prop.house/nouns/private-voting-research-sprint

[16] Nethermind Research, "Evaluation of the Private Voting Research Sprint
solutions and advising on next steps for Nouns," Nouns Camp candidate.
https://www.nouns.camp/candidates/evaluation-of-the-private-voting-research-sprint-solutions-and-advising-on-next-steps-for-nouns-2e0e2f18b64fdbf02a76255a6b9017c2aa9d82d8

[17] "Election Verifiability Revisited: Automated Security Proofs and Attacks on
Helios and Belenios."
https://www.researchgate.net/publication/353816050_Election_Verifiability_Revisited_Automated_Security_Proofs_and_Attacks_on_Helios_and_Belenios

[18] "BeleniosRF: A Non-interactive Receipt-Free Electronic Voting Scheme."
https://www.researchgate.net/publication/310823447_BeleniosRF_A_Non-interactive_Receipt-Free_Electronic_Voting_Scheme

[19] "DAO Governance: Voting Power, Participation and Controversy — A Review and
an Empirical Analysis."
https://www.researchgate.net/publication/397728414_DAO_Governance_Voting_Power_Participation_and_Controversy_-_A_Review_and_an_Empirical_Analysis

[20] "Democracy for DAOs," arXiv 2507.20234. https://arxiv.org/pdf/2507.20234

[21] S. Balietti, P. Saggese, M. Strohmaier, "Voting Biases in Decentralized
Autonomous Organization (DAO) Governance," arXiv 2607.09435.
https://arxiv.org/html/2607.09435v1

[22] "Fairness in Token Delegation: Mitigating Voting Power Concentration in
DAOs," arXiv 2510.05830. https://arxiv.org/html/2510.05830v1

[23] ENS governance forum, "Draft: Reform DAO governance by delegating 5M ENS
tokens," 6 July 2026.
https://discuss.ens.domains/t/draft-reform-dao-governance-by-delegating-5m-ens-tokens/22247

[24] Uniswap Proposal 78, DAO Principles, executed 30 January 2025.
https://vote.uniswapfoundation.org/proposals/78

[25] Uniswap Proposal 80, Delegate Reward Initiative.
https://vote.uniswapfoundation.org/proposals/80

[26] Optimism, Delegate expectations.
https://community.optimism.io/docs/governance/delegate/

[27] Arbitrum Foundation, Delegation concepts.
https://docs.arbitrum.foundation/concepts/delegate-delegation

[28] Curia Optimism delegate dashboard (undated).
https://optimism.curiahub.xyz/delegate

[29] Votium briber manual. https://docs.votium.app/explainers/briber-manual

[30] Votium vlCVX FAQ. https://docs.votium.app/faq/vlcvx-faq

[31] Asymmetry Finance, afCVX documentation, quoting Votium cumulative figures.
https://docs.asymmetry.finance/afcvx-asymmetry-finance-convex (third-party
attribution; not confirmed at source)

[32] ChainCatcher coverage of Hidden Hand incentive volumes.
https://www.chaincatcher.com/en/article/2180822

[33] Redacted Cartel, Hidden Hand launch post.
https://mirror.xyz/0xE90c74145245B498fef924fAdC7bb34253c7cF90/CZDYoNk97LWOSvnOXst5ugbM5B1WHlcW3MCu4-5LIFE

[34] Paladin, delegation and bribe-yield documentation.
https://keep.paladin.vote/blog/how-to-delegate-your-vl-cvx-or-vl-aura-to-paladin-to-farm-optimized-bribe-yield/

[35] Mitosis University, "veTokenomics: bribe markets, gauge voting incentives
and Curve wars mechanics."
https://university.mitosis.org/vetokenomics-bribe-markets-gauge-voting-incentives-and-curve-wars-mechanics/
(analyst commentary)

[36] Immunefi, "Hack Analysis: Beanstalk Governance Attack, April 2022."
https://medium.com/immunefi/hack-analysis-beanstalk-governance-attack-april-2022-f42788fc821e

[37] CoinDesk, "Attacker Drains $182M From Beanstalk Stablecoin Protocol," 17
April 2022.
https://www.coindesk.com/tech/2022/04/17/attacker-drains-182m-from-beanstalk-stablecoin-protocol

[38] BleepingComputer, "Beanstalk DeFi platform loses $182 million in flash loan
attack."
https://www.bleepingcomputer.com/news/security/beanstalk-defi-platform-loses-182-million-in-flash-loan-attack/

[39] The Block, "Build Finance DAO suffers hostile governance takeover, loses
$470,000."
https://www.theblock.co/post/134180/build-finance-dao-suffers-hostile-governance-takeover-loses-470000

[40] Decrypt, "Build Finance DAO falls to governance takeover."
https://decrypt.co/92970/build-finance-dao-falls-to-governance-takeover

[41] CryptoSlate, "Build Finance DAO hostile takeover, treasury drained."
https://cryptoslate.com/build-finance-dao-hostile-takeover-treasury-drained/

[42] CoinDesk, "Attacker Takes Over Tornado Cash DAO With Vote Fraud," 21 May
2023\.
https://www.coindesk.com/tech/2023/05/21/attacker-takes-over-tornado-cash-dao-with-vote-fraud-token-slumps-40

[43] The Block, "Attacker uses malicious proposal to take over Tornado Cash
governance."
https://www.theblock.co/post/231637/attacker-uses-malicious-proposal-to-take-over-tornado-cash-governance

[44] Halborn, "Explained: The Tornado Cash Hack (May 2023)."
https://www.halborn.com/blog/post/explained-the-tornado-cash-hack-may-2023

[45] Unchained, "Tornado Cash DAO passes attacker's proposal to restore
governance."
https://unchainedcrypto.com/tornado-cash-dao-passes-attackers-proposal-to-restore-governance/

[46] SEC press release 2023-13, Mango Markets.
https://www.sec.gov/newsroom/press-releases/2023-13 ; complaint:
https://www.sec.gov/files/litigation/complaints/2023/comp-pr2023-13.pdf

[47] CFTC press release 8647-23.
https://www.cftc.gov/PressRoom/PressReleases/8647-23

[48] CoinDesk, "Mango Markets exploiter thought a DAO protected him. Then US
courts showed up."
https://www.coindesk.com/business/2023/01/31/mango-markets-exploiter-thought-a-dao-protected-him-then-us-courts-showed-up

[49] The Block, "$24 million Compound Finance proposal passed by whale over DAO
objections."
https://www.theblock.co/post/307943/24-million-compound-finance-proposal-passed-by-whale-over-dao-objections

[50] Cointelegraph, "Golden Boys behind Compound governance attack agree to
rescind proposal."
https://cointelegraph.com/news/golden-boys-behind-compound-governance-attack-agree-to-rescind-proposal

[51] Unchained, "Compound governance attackers agree to cancel proposal in
exchange for staking product."
https://unchainedcrypto.com/compound-governance-attackers-agree-to-cancel-proposal-in-exchange-for-staking-product/

[52] CoinDesk, "Contentious Arbitrum vote over $1B in tokens: 'ratification, not
request,' says Foundation."
https://www.coindesk.com/business/2023/04/02/contentious-arbitrum-vote-over-1b-in-tokens-ratification-not-request-says-foundation

[53] The Block, "Arbitrum proposal to return funds failed."
https://www.theblock.co/post/226561/arbitrum-proposal-return-funds-failed

[54] Blockworks, "Arbitrum walks back proposal."
https://blockworks.co/news/arbitrum-walks-back-proposal

[55] DL News, "Arbitrum governance vote."
https://www.dlnews.com/articles/defi/arbitrum-governance-vote-arb-dao-airdrop-aip-1-proposal/

[56] P. Daian, T. Kell, I. Miers, A. Juels, "On-Chain Vote Buying and the Rise
of Dark DAOs," IC3, 2 July 2018.
https://initc3org.medium.com/on-chain-vote-buying-and-the-rise-of-dark-daos-b01f5bd77030

[57] J. Austgen, A. Fábrega, S. Allen, K. Babel, M. Kelkar, A. Juels, "DAO
Decentralization: Voting-Bloc Entropy, Bribery, and Dark DAOs," arXiv
2311.03530, 6 November 2023. https://arxiv.org/abs/2311.03530 ;
https://www.cs.cornell.edu/~babel/papers/dao-vbe-dd.pdf

[58] USENIX Security 2025 proceedings version.
https://www.usenix.org/system/files/usenixsecurity25-fabrega-entropy.pdf

[59] DAO Digest, "How whales swing DAO votes."
https://daodigest.com/how-whales-swing-dao-votes/ (commentary; no supporting
data)

[60] A. Caravello, Shutter Network, "DAO voting confidence is in decline: how to
restore it," 20 May 2025.
https://blog.shutter.network/dao-voting-confidence-is-in-decline-how-to-restore-it/
(vendor commentary; no cited data)

[61] Snapshot Labs announcement of shielded voting, 13 October 2022.
https://x.com/SnapshotLabs/status/1580674555710181378

[62] Shutter Network, shielded voting product page (vendor self-reported
figures). https://shutter.network/shielded-voting/

[63] Euler governance forum, shielded voting RFC.
https://forum.euler.finance/t/rfc-switch-snapshot-space-to-shielded-voting-secret-ballots-using-shutter-threshold-encryption/442

[64] Stake DAO governance, SDGP-33.
https://gov.stakedao.org/t/sdgp-33-switch-snapshot-space-to-shielded-voting-secret-ballots-using-shutter-threshold-encryption/914

[65] ENS governance forum, "Temp check: shielded voting for ENS Snapshot
proposals."
https://discuss.ens.domains/t/temp-check-shielded-voting-for-ens-snapshot-proposals/22142

[66] Vocdoni davinci-node repository. https://github.com/vocdoni/davinci-node

[67] Vocdoni documentation, ZK Census Proof.
https://docs.vocdoni.io/architecture/protocol/anonymous-voting/zk-census-proof

[68] Vocdoni blog, "Anonymous voting with zkSNARKs."
https://blog.vocdoni.io/anonymous-voting-zksnarks/

[69] Vocdoni documentation, Blind Signatures.
https://docs.vocdoni.io/architecture/protocol/anonymous-voting/blind-signatures

[70] Vocdoni documentation, Census Overview.
https://docs.vocdoni.io/architecture/census/census-overview

[71] Vocdoni testimonials (vendor-published case studies).
https://vocdoni.io/testimonials

[72] Democracy Technologies, "Blockchain voting: Decidim and Vocdoni."
https://democracy-technologies.org/voting/blockchain-voting-decidim-vocdoni/

[73] The Fintech Times, FC Barcelona blockchain voting coverage.
https://thefintechtimes.com/fc-barcelona-reaches-milestone-in-blockchain-tech-adoption-immortalising-johan-cruyff/

[74] Penumbra pcli governance guide.
https://guide.penumbra.zone/usage/pcli/governance

[75] Penumbra community governance frontend (host did not respond during this
survey). https://vote.penumbra.zone/

[76] Zcash NU7 Priorities ZCAP poll, March 2025, Helios instance.
https://vote.heliosvoting.org/helios/e/zcash-nu7-priorities-zcap-poll-mar-2025

[77] N. Wilcox, `zec-coin-polling` repository README.
https://github.com/nathan-at-least/zec-coin-polling/blob/main/README.md

[78] Coin Voting Book, proposal details.
https://hhanh00.github.io/coin-voting-book/proposal/details.html

[79] Zcash Community Forum, "Zodl 3.5.0: vote with your ZEC," 28 May 2026.
https://forum.zcashcommunity.com/t/zodl-3-5-0-vote-with-your-zec/55888

[80] Zcash Foundation, "NU7 polling results: what we heard and where we go from
here."
https://zfnd.org/nu7-polling-results-what-we-heard-and-where-we-go-from-here/

[81] Zcash Community Forum, "Coinholder-Directed Retroactive Grants Program
results, Nov 2025."
https://forum.zcashcommunity.com/t/coinholder-directed-retroactive-grants-program-results-nov-2025/53487/19

[82] ZIP 1016. https://zips.z.cash/zip-1016

[83] V. Buterin, "Minimal anti-collusion infrastructure," ethresear.ch.
https://ethresear.ch/t/minimal-anti-collusion-infrastructure/5413

[84] MACI documentation, security properties table.
https://maci.pse.dev/docs/introduction

[85] MACI roadmap. https://maci.pse.dev/roadmap ; MACI v2.0 release.
https://maci.pse.dev/blog/2024-v2

[86] MACI v1.2.0 release notes and censorship bug fix (PR #1170).
https://maci.pse.dev/blog/maci-v1-2-0-release ;
https://github.com/privacy-scaling-explorations/maci/pull/1170

[87] PSE audit report, 31 July 2024.
https://maci.pse.dev/assets/files/20240731_PSE_Audit_audit_report-a0a0f08a5c621ccd0389d5e345c119be.pdf

[88] MACI monorepo package and contract listing.
https://github.com/privacy-scaling-explorations/maci

[89] MACI coordinator service.
https://maci.pse.dev/blog/maci-coordinator-service

[90] PSE, "MACI Aragon plugin." https://maci.pse.dev/blog/maci-aragon-plugin ;
https://github.com/privacy-scaling-explorations/maci-voting-plugin-aragon

[91] MACI supported networks and costs.
https://maci.pse.dev/docs/supported-networks/

[92] Poseidon team, "DeFROST." https://hackmd.io/6ZFxxxnKT0iH-GJHUxKekw

[93] Aragon ZK Research, Nouns general report, 17 August 2023.
https://research.aragon.org/nouns.html

[94] Aztec, "NounsDAO private voting final update."
https://aztec.network/blog/nounsdao-private-voting-final-update

[95] Aragon ZK Research, Nouns technical report.
https://research.aragon.org/nouns-tech.html

[96] `nouns-anonymous-voting` repository.
https://github.com/aragonzkresearch/nouns-anonymous-voting

[97] Aragon ZK Research, Timelock report.
https://research.aragon.org/timelock.html ;
https://github.com/aragonzkresearch/blog/blob/main/pdf/azkr-timelock-zone.pdf

[98] Timelock.zone service. https://timelock.zone

[99] Team 水 (Mizu), "Vortex." https://mizu-dao.github.io/vortex/

[100] Semaphore protocol GitHub organisation.
https://github.com/semaphore-protocol

[101] Semaphore documentation. https://docs.semaphore.pse.dev/

[102] Semaphore proofs guide. https://docs.semaphore.pse.dev/guides/proofs

[103] Semaphore benchmarks and contracts reference.
https://docs.semaphore.pse.dev/benchmarks ;
https://docs.semaphore.pse.dev/technical-reference/contracts

[104] Secret Network governance documentation.
https://docs.scrt.network/guides/governance.html

[105] Aztec developer documentation. https://aztec.network/developers

[106] Aztec, "How Aztec governance works."
https://aztec.network/blog/how-aztec-governance-works

[107] `quadratic_voting_noir` demo repository.
https://github.com/joss-aztec/quadratic_voting_noir

[108] Aztec Labs, "The time NounsDAO got private voting," March 2023.
https://medium.com/aztec-protocol/the-time-nounsdao-got-private-voting-4336fe4a2c29

[109] Mina Protocol, "On-chain voting for Mina Improvement Proposals (MIPs),
part 2."
https://minaprotocol.com/blog/on-chain-voting-for-mina-improvement-proposals-mips-part-2

[110] Mina Protocol, "The Mesa Upgrade on-chain vote," December 2025.
https://minaprotocol.com/blog/the-mesa-upgrade-on-chain-vote

[111] Aleo, "Launching the Aleo Foundation Governance Platform," May 2024.
https://aleo.org/post/launching-the-aleo-foundation-governance-platform/

[112] `zvote` repository. https://github.com/zsociety-io/zvote

[113] `aleo-voting-app` repository.
https://github.com/youssef-cherrat/aleo-voting-app

[114] "Secrecy and Verifiability: An Introduction to Electronic Voting," arXiv
2602.12398. https://arxiv.org/pdf/2602.12398

[115] "Efficient Universally-Verifiable Electronic Voting with Everlasting
Privacy." https://link.springer.com/chapter/10.1007/978-3-031-71070-4_15
