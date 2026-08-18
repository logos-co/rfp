# Appendix: Ethereum Light Client Ecosystem

This appendix surveys the production zk light-client implementations relevant to
[RFP-021](../RFPs/RFP-021-ethereum-state-attestation.md), a trustless Ethereum
state attestation primitive for LEZ. It exists to source one specific claim in
that RFP: that a **permissionless operator advancing shared on-chain
light-client state** is an established, audited, production pattern rather than
a novel design, and that the trust properties of that pattern are well
understood.

The survey covers what each implementation does, who is permitted to advance the
on-chain state, what the resulting trust model is, and what a LEZ implementation
can reuse. It deliberately includes one implementation that **restricts** update
submission, because the contrast is the clearest evidence that
permissionlessness is a deployment choice rather than an automatic consequence
of using zero-knowledge proofs.

## The pattern

Every implementation surveyed here decomposes the problem the same way.

Verifying that an Ethereum fact is true requires two things: proving that a
block header is a finalised header of the canonical chain, and proving that some
piece of state is included under that header. The first is expensive, involving
BLS12-381 aggregate signature verification over sync-committee signatures and
the verification of committee handoffs across sync-committee periods. The second
is comparatively cheap, being Merkle-Patricia path verification.

The two are also asymmetric in another way. Header finality is a fact about
Ethereum that every consumer needs and no consumer needs privately. Inclusion is
specific to the consumer's own transaction.

That asymmetry drives the architecture:

1. An **operator** runs off-chain. It follows the beacon chain, generates a
   zero-knowledge proof that a light-client finality update is validly signed by
   the sync committee, and submits that proof to a contract on the destination
   chain. Committee handoffs are proven the same way.
2. A **contract** on the destination chain verifies the proof and stores the
   resulting verified state: the current sync committee, and finalised header
   roots. The stored state accumulates and is readable by any consumer.
3. A **consumer** submits only a Merkle inclusion proof against a header already
   verified in that contract. It performs no signature verification of its own.

The operator is untrusted. It cannot insert a header the sync committee did not
sign, because the contract verifies the proof on submission; forging a header
would require forging BLS signatures from two thirds of the committee. The only
thing an operator can do is stop, which is a liveness failure rather than a
safety failure, and which is repaired by anyone else running the same operator
software.

This is the property that makes the operator role viable in a trust-minimised
design, and it is the property RFP-021 requires in Functionality #11.

## Implementations Surveyed

| Implementation                              | zkVM / proof system     | Update submission                 | Audit            | Notes                                                                                                 |
| ------------------------------------------- | ----------------------- | --------------------------------- | ---------------- | ----------------------------------------------------------------------------------------------------- |
| Telepathy (Succinct)                        | Custom zkSNARK circuits | Permissionless, no access control | **Veridise**     | Clearest published statement of the trust model; `step` and `rotate` are `external` with no modifiers |
| SP1 Helios (Succinct)                       | SP1 zkVM                | Permissionless in upstream        | **Zellic**       | Helios light client compiled into SP1; the modern reference implementation                            |
| r0vm-helios (Boundless, formerly RISC Zero) | **RISC Zero zkVM**      | Follows upstream                  | **zkSecurity**   | Fork of SP1 Helios; the closest existing artefact to a LEZ port target                                |
| SP1 Helios (Across fork)                    | SP1 zkVM                | **Permissioned**, `UPDATER_ROLE`  | **OpenZeppelin** | Deliberate divergence; the counter-example                                                            |

The audits named above are published: Veridise for Telepathy [10], Zellic for
upstream SP1 Helios [11], zkSecurity for r0vm-helios [4], and OpenZeppelin for
the Across fork [5]. The upstream and fork audits are separate engagements
covering different code.

The [Axelar light client](https://github.com/commonprefix/axelar-light-client)
[8] belongs to the same family but is omitted from the table: it applies the
sync-committee protocol directly rather than through a zkVM, and its repository
does not state an update-submission policy, so it contributes no evidence either
way on the question this survey is asking.

zkBridge [6] is adjacent but is deliberately not listed above. It follows the
same shared-state shape, an updater contract maintaining verified headers which
applications read with their own Merkle proofs, and its paper claims
permissionless header relay. It is not an Ethereum sync-committee light client,
however: the paper is generic over light-client protocols, its
Ethereum-as-source instantiation is pre-Merge longest-chain proof of work, and
Polyhedra's production deployment runs as a LayerZero DVN attesting per message
rather than as an open header store. It is prior art for the shape, not for the
anchor.

### Telepathy

Telepathy states the trust model explicitly, which is why it is the primary
source for the pattern rather than the most current implementation.

An Operator queries the sync committee for the latest finalised block header and
generates a zkSNARK attesting that enough of the committee signed it, submitted
via `step`. The same Operator handles committee rotation: the sync committee
rotates roughly every 27 hours, and the Operator generates a separate proof
attesting to the correct selection of the next committee, submitted via `rotate`
[1].

On permissionlessness, the documentation is unambiguous: "these actors are
completely permissionless. Succinct operates their own version of these actors
so that the user experience of integrating with Telepathy painless, but as all
`LightClient` and `TelepathyRouter` update functions are permissionless (with no
access control), anyone who wishes to can run their own versions of an operator
and relayer can do so" [1]. The `step(...)` and `rotate(...)` functions are
`external` with no access control modifiers.

The division of labour matters for RFP-021's cost analysis: the Operator bears
the proving cost of consensus verification, and a Relayer separately submits
Merkle proofs validating a message against the header root already stored in the
light client [1].

### SP1 Helios

SP1 Helios is an implementation of a zk Ethereum light client using the Altair
sync committee, combining the Helios light client with the SP1 zkVM [2]. It
verifies the consensus of a source chain inside the execution environment of a
destination chain.

The operator "keeps an on-chain SP1 Helios light client updated by proving
finalized source-chain consensus updates and submitting them to the
destination-chain SP1 Helios contract" [2].

Two commitment modes are relevant to RFP-021's Functionality #3, which requires
inclusion proofs for accounts, storage slots, and receipts or logs. The default
mode commits the finalised light-client state and the execution state root. An
execution-header mode additionally commits the finalised execution block hash
and the finalised execution receipts root [2]. A design that needs receipt and
log proofs requires the receipts root to be committed, which is a concrete
interface decision an applicant will face; the documentation puts the cost of
that mode at approximately 45k additional gas per successful update [2].

### r0vm-helios

r0vm-helios is the implementation most directly relevant to LEZ, because LEZ
runs on RISC0.

It is a fork of SP1 Helios that verifies source-chain consensus in the execution
environment of a destination chain, built on the RISC Zero zkVM, and it has
received an independent security audit from zkSecurity, published in the
`risc0/rz-security` repository [3][4].

Architecturally it matches the pattern: a consensus RPC endpoint (a beacon node
supporting the Altair light-client protocol methods), an operator that generates
proofs and maintains light-client state, and a destination-chain contract
storing verified consensus data [3].

Its existence is why RFP-021 directs applicants to assess reuse rather than
reimplement. An audited RISC Zero implementation of the hardest and most
security-critical component is a materially different starting point from a
blank sheet, and the RFP's Reliability #6 requirement to integrate mature
audited implementations points here. What it is **not** is a drop-in: it targets
EVM destination chains with Solidity contracts, and LEZ is neither. The port
surface, what changes and what carries over unmodified, is an assessment the
applicant must perform and justify.

### The Across fork: where the pattern is deliberately broken

Across Protocol maintains a fork of SP1 Helios that makes update submission
permissioned, and it is the most instructive entry in this survey.

The contract inherits `AccessControlEnumerable` and gates the `update` function
behind an `UPDATER_ROLE`. (The repository has since been restructured and its
`main` branch no longer carries the Solidity contracts; the contract described
here lives on the `audit/march-31-2024` branch, which is what the audit covers.)
Per the OpenZeppelin audit, "the updaters set is immutable, defined at
deployment and cannot be modified afterward, meaning that no new addresses can
be added or removed post-deployment", and "Risk Labs will control the updater
role, ensuring that only trusted entities can submit updates" [5]. No address
holds `DEFAULT_ADMIN_ROLE`, so the role cannot be granted or revoked after
deployment. The audit reached that end state via a finding rather than
confirming a clean intent: code comments said the role "should be admin-less"
while a redundant `_setRoleAdmin` call and contradictory test comments muddied
it, and the superfluous call was subsequently removed [5].

Two conclusions follow, and both are load-bearing for RFP-021.

**Permissionlessness is a choice, not a consequence.** A team can port an
audited zk light client faithfully and still ship a design in which one named
party is the only entity able to advance the state. Nothing about using zk
proofs prevents this. RFP-021 therefore cannot treat the property as implied by
the technology and must require it explicitly.

**Safety-trustlessness and censorship-resistance are separable.** Even in the
Across design, a rogue updater cannot forge a header; the proof verification
stops that, and safety survives. What is lost is liveness and
censorship-resistance: if Risk Labs stops submitting, or declines to submit,
nobody else can advance the light client, and every consumer stalls. Because the
updater set is immutable, there is no recovery path short of deploying a new
contract.

That distinction is exactly what RFP-021 asks proposals to state, and this fork
demonstrates that a design can satisfy "no trusted party can forge" while
failing "no party can block".

## Operator economics

The pattern raises an obvious question: if nobody is required to run an
operator, why would anyone?

The answer visible across these deployments is that the cost of advancing the
state is small and the parties who need it advanced are identifiable and
motivated. Submitting an update is a single transaction whose cost is amortised
across every consumer reading the resulting header, and any protocol whose users
depend on fresh headers has a direct incentive to keep them fresh. Succinct runs
operators for Telepathy for user-experience reasons while leaving the role open
to anyone [1]; Across runs its own because its bridge depends on it.

This is a weaker guarantee than a slashing-backed obligation, and it should be
stated as such: no operator is contractually bound to act. What the design
guarantees is that a stalled light client can be restarted by any party willing
to spend the gas, without permission from anyone, and without any pre-existing
relationship to the deployment. The failure mode is a delay, not a loss, and
recovery does not depend on the party that stopped.

## Bootstrap trust and deployment multiplicity

Sync-committee tracking cannot start from nothing. It begins from a
weak-subjectivity checkpoint, a known-good committee, and follows verified
handoffs forward from there. That checkpoint is a one-off trust input at
deployment: everything after it is verified, and nothing before it is.

Two properties of this bootstrap are worth stating plainly, because they shape
how a deployment should be treated.

**The checkpoint is independently verifiable.** Unlike a signer set, whose
honesty cannot be checked from outside, a configured checkpoint is a claim about
public Ethereum history that anyone can confirm against public sources before
relying on the deployment. A deployment whose checkpoint is wrong or
unverifiable is detectably so, which is why RFP-021 requires a documented
verification procedure rather than only a configured value.

**Deployment is not exclusive.** Nothing prevents a second contract being
deployed with a different checkpoint. If a deployment's bootstrap is
mis-configured, or its checkpoint falls outside the weak-subjectivity period,
the remedy is a fresh deployment rather than a governance action over the
existing one. This is a genuine safety valve and it is the reason the primitive
does not need a mutable checkpoint under privileged control.

It carries a cost that pulls in the opposite direction, and the tension should
be acknowledged rather than resolved by assertion. Fragmenting across
deployments splits the operator incentive, duplicates gas expenditure, and
forces every consumer to decide which contract it trusts. The value of the
shared-state architecture comes precisely from many consumers reading the same
verified headers. A canonical deployment that all consumers converge on is
therefore the intended outcome, with the ability to deploy another treated as a
recovery path rather than a routine expectation.

## What this implies for a LEZ implementation

Three consequences carry directly into RFP-021.

**The cost problem changes shape.** RFP-021's central open question is whether
in-zkVM BLS12-381 verification fits a LEZ transaction budget. Under the shared
state architecture, that cost sits in the operator's submission and is amortised
across every consumer, while a consumer's own transaction contains only Merkle
verification. This does not make the BLS cost disappear, and measuring it
remains a primary deliverable, but it decouples that cost from the per-consumer
budget.

**The interface is a stored header set, not a per-transaction proof.** A
consumer reads a verified header from shared state and proves inclusion against
it. RFP-021's requirement that verification be callable as a library, so that
verification and the action it authorises are atomic, applies to the inclusion
and predicate steps against already-verified consensus state.

**Reuse is an assessment, not an instruction.** r0vm-helios is audited, RISC
Zero based, and solves the hardest component. Whether its guest program, its
operator, or only its architecture transfers to LEZ depends on details an
applicant is better placed to evaluate than this specification is. The RFP
points at it as a candidate and requires the assessment to be made and justified
rather than assuming the answer.

## References

01. [Telepathy — Off-chain Actors](https://docs.telepathy.xyz/telepathy-protocol/actors);
    see also
    [Telepathy — Sync Committee Protocol](https://docs.telepathy.xyz/telepathy-protocol/sync-committees)
    and
    [Telepathy — Smart Contracts](https://docs.telepathy.xyz/telepathy-protocol/contracts)
02. [succinctlabs/sp1-helios](https://github.com/succinctlabs/sp1-helios); see
    also [SP1 Helios documentation](https://succinctlabs.github.io/sp1-helios/)
03. [boundless-xyz/r0vm-helios](https://github.com/boundless-xyz/r0vm-helios)
04. [risc0/rz-security — audits](https://github.com/risc0/rz-security/tree/main/audits)
    (zkSecurity audit of r0vm-helios)
05. [OpenZeppelin — SP1 Helios Audit](https://www.openzeppelin.com/news/sp1-helios-audit);
    see also
    [across-protocol/sp1-helios](https://github.com/across-protocol/sp1-helios)
06. [zkBridge: Trustless Cross-chain Bridges Made Practical](https://rdi.berkeley.edu/zkp/uploads/paper.pdf)
    (Berkeley RDI)
07. [a16z/helios](https://github.com/a16z/helios) (the underlying Helios light
    client)
08. [commonprefix/axelar-light-client](https://github.com/commonprefix/axelar-light-client)
09. [Ethereum Altair light-client specification](https://github.com/ethereum/consensus-specs/tree/master/specs/altair/light-client)
10. [Veridise — Succinct Labs Telepathy audit, 11 March 2023](https://veridise.com/audits-archive/company/succinct/succinct-labs-telepathy-2023-03-11/)
11. Zellic — SP1 Helios audit, 18 to 29 July 2025, published as
    `SP1 Helios - Zellic Audit Report.pdf` in
    [succinctlabs/sp1-helios](https://github.com/succinctlabs/sp1-helios)
