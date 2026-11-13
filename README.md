![Banner](/public/logos-rfps.png)

# Logos RFP Program

The Logos RFP Program supports independent developers and teams through grants
to build tools, infrastructure, applications and integrations that expand the
Logos ecosystem.

Through Requests for Proposals (RFPs), Logos invites builders to submit projects
that demonstrate the capabilities of the Logos technology stack and help
accelerate ecosystem adoption.

Projects supported under the Logos RFP Program may range from experimental
prototypes and proofs-of-concept to production-ready or near-production-ready
applications and integrations.

> **Note** By participating in the Logos RFP Program, including by submitting a
> proposal through this repository, you agree to the
> [Terms & Conditions](./TERMS_AND_CONDITIONS.md). Please read them carefully
> before submitting a proposal.

______________________________________________________________________

## About Logos

Logos is a social movement and decentralised technology stack.

The Logos stack comprises three core layers:

- **Blockchain** – A privacy-preserving Layer 1 for execution, settlement, and
  coordination. The Logos Execution Zone (LEZ) is the programmable environment
  where decentralised applications run, featuring a unique separation of public
  and private state with shielded balances and private state as first-class
  primitives.
- **Storage** – Durable, censorship-resistant data availability powering fully
  decentralised apps and file sharing.
- **Messaging** – Private peer-to-peer communication that resists surveillance
  and censorship.

Together these form a unified, modular ecosystem, accessible through **Logos
Core**, a plugin-based runtime that lets developers compose all three layers
into privacy-preserving applications.

______________________________________________________________________

## Submitting a Proposal

How to submit a proposal:

1. Browse the open RFPs below
2. Read the RFP you want to apply for in the `rfps/` folder
3. Submit a proposal by [creating a new issue](../../issues/new)
4. We review proposals weekly and typically respond within 14 days

Use the GitHub issue pertaining to your proposal for any questions or
clarifications. Updates regarding the decision of your proposal will be made on
your issue / contact information provided in the proposal.

Proposals must follow the submission template. RFPs that deliver a Logos
Basecamp module also follow
[Security and Release Integrity](#security-and-release-integrity) for their
security tier.

______________________________________________________________________

## What We Support

Proposals should align with one of the below focus categories:

- Developer Tooling and Infrastructure
- Applications and Integrations
- Ecosystem and Community Enablement

## Open RFPs

Click an RFP to view details. Use the Submit Proposal button to apply.

<!-- RFP_TABLE_START -->

| ID      | Title                                                                                       | Tier | Status | Submission Deadline | Category                           | Submit Proposal                                                                     |
| ------- | ------------------------------------------------------------------------------------------- | ---- | ------ | ------------------- | ---------------------------------- | ----------------------------------------------------------------------------------- |
| RFP-001 | [Admin Authority Library](RFPs/RFP-001-admin-authority-lib.md)                              | XS   | closed | -                   | Developer Tooling & Infrastructure | Closed                                                                              |
| RFP-002 | [Freeze Authority Library](RFPs/RFP-002-freeze-authority-lib.md)                            | XS   | closed | -                   | Developer Tooling & Infrastructure | Closed                                                                              |
| RFP-003 | [Atomic Swaps with LEZ](RFPs/RFP-003-atomic-swaps.md)                                       | XL   | closed | -                   | Applications & Integrations        | Closed                                                                              |
| RFP-004 | [Privacy-Preserving DEX](RFPs/RFP-004-privacy-preserving-dex.md)                            | XL   | closed | -                   | Applications & Integrations        | Closed                                                                              |
| RFP-005 | [Multisig and Coordination](RFPs/RFP-005-multisig-and-coordination.md)                      | L    | closed | -                   | Applications & Integrations        | Closed                                                                              |
| RFP-008 | [Lending & Borrowing Protocol](RFPs/RFP-008-lending-borrowing-protocol.md)                  | XL   | closed | -                   | Applications & Integrations        | Closed                                                                              |
| RFP-012 | [Curated Lending Vaults](RFPs/RFP-012-curated-lending-vaults.md)                            | L    | closed | -                   | Applications & Integrations        | Reopening soon                                                                      |
| RFP-013 | [Reflexive Stablecoin Protocol](RFPs/RFP-013-reflexive-stablecoin-protocol.md)              | XL   | closed | -                   | Applications & Integrations        | Closed                                                                              |
| RFP-014 | [Liquidation & Auction Engine](RFPs/RFP-014-liquidation-auction-engine.md)                  | L    | closed | -                   | Applications & Integrations        | Reopening soon                                                                      |
| RFP-015 | [Token Launchpad: Bonding Curve](RFPs/RFP-015-bonding-curve-launchpad.md)                   | L    | closed | -                   | Applications & Integrations        | Closed                                                                              |
| RFP-016 | [Token Launchpad: LBP](RFPs/RFP-016-lbp-launchpad.md)                                       | XL   | closed | -                   | Applications & Integrations        | Closed                                                                              |
| RFP-017 | [Privacy-Preserving Token Vesting](RFPs/RFP-017-token-vesting.md)                           | L    | closed | -                   | Applications & Integrations        | Closed                                                                              |
| RFP-019 | [On-Chain TWAP Oracle](RFPs/RFP-019-twap-oracle.md)                                         | L    | closed | -                   | Developer Tooling & Infrastructure | Closed                                                                              |
| RFP-020 | [RedStone Off-Chain Oracle Adaptor for LEZ](RFPs/RFP-020-redstone-oracle-adaptor.md)        | M    | closed | -                   | Developer Tooling & Infrastructure | Closed                                                                              |
| RFP-021 | [Trustless Ethereum State Attestation for LEZ](RFPs/RFP-021-ethereum-state-attestation.md)  | M    | open   | 18/11/26            | Developer Tooling & Infrastructure | [Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml) |
| RFP-022 | [Privacy-Preserving Wrapped ERC-20 and Ether Bridge for LEZ](RFPs/RFP-022-wrapped-erc20.md) | L    | open   | 18/11/26            | Developer Tooling & Infrastructure | [Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml) |

<!-- RFP_TABLE_END -->

## Security and Release Integrity

Installing a Logos Basecamp module runs its author's native code with the user's
full account privileges. The requirements below make sure the build Logos
evaluates at each milestone is the build users install later, and that it cannot
be swapped or impersonated afterwards. They apply to every RFP whose
deliverables include a Basecamp module (`.lgx`), unless the RFP states
otherwise. Each RFP states a `security_tier` and references these rules instead
of restating them.

### Security tiers

| Tier | What the module handles                                                    | Requirements                                                                                                       |
| ---- | -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| 0    | No secrets and no user data                                                | Baseline                                                                                                           |
| 1    | User data (messages, files, contacts, identities)                          | Baseline, plus a data-handling note in the README: what the module stores or sends, where, and how it is protected |
| 2    | Keys or funds, including wallets and AI agents that hold or can move funds | Baseline, plus [Tier 2 requirements](#tier-2-requirements)                                                         |

### Baseline requirements (all tiers)

01. **Signed packages.** Every `.lgx` release in the team's catalog is signed
    with `lgx sign`, using `signing_mode: inline` or `signing_mode: external` in
    the release workflow. The signer DID is listed under `trustedSigners` in the
    catalog's `logos-repo.json`. See
    [Signing in `logos-modules-release-base`](https://github.com/logos-co/logos-modules-release-base#signing),
    [Signing in `logos-modules-release-action`](https://github.com/logos-co/logos-modules-release-action#signing)
    and the
    [`lgx` signing guide](https://github.com/logos-co/logos-package/blob/master/doctests/outputs/lgx-signing.md).
    Unsigned releases are not accepted as milestone deliverables.
02. **Signer declared up front.** The `did:jwk:` signer DID is declared in the
    proposal or, at the latest, in the first milestone delivery, and in the
    project README. If the key must be rotated, announce the new DID in the
    proposal issue before publishing a release signed with it.
03. **Persistent (immutable) GitHub releases.** Enable release immutability on
    the catalog repository, and on the source repository if it publishes
    releases, before publishing any release delivered for a milestone. Follow
    GitHub's official instructions:
    [Preventing changes to your releases](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/establish-provenance-and-integrity/preventing-changes-to-your-releases).
    See also
    [About immutable releases](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/immutable-releases)
    and
    [Verifying the integrity of a release](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/verify-release-integrity).
    Immutability applies only to releases published after it is enabled, so the
    catalog's rolling `index` release (which `rebuild-index` updates in place)
    must already exist when you turn it on.
04. **Freeze rule.** A milestone is approved for the evaluated build, identified
    by its `rootHash`. Never replace, delete or re-tag a published release or
    its assets. Fixes ship as a new version and are delivered again.
05. **Release integrity manifest.** Each milestone delivery records exactly what
    is delivered: source commit SHA, catalog `logos-repo.json` URL, package name
    and version, release URL, `rootHash`, SHA-256 of the `.lgx`, built platform
    variants and signer DID. Take the values from the release sidecar JSON or
    `lgx manifest`. Reviewers recompute them.
06. **Canonical install instructions.** The README, demo and any promotion give
    the exact catalog URL, signer DID and `rootHash`. Never tell users to
    "search for" the module, add a catalog from a chat message, or ignore or
    disable signature warnings.
07. **An address nobody else can claim.** Host the catalog on a GitHub
    organisation account, not a personal one, and do not rename or delete the
    account or repository during the grant or the post-delivery period. If your
    release workflow uploads the `.lgx` to Logos Storage, record its CID in the
    manifest.
08. **Account and repository security.**
    - Two-factor authentication on every account with write access to the source
      or catalog repository
      ([GitHub 2FA](https://docs.github.com/en/authentication/securing-your-account-with-two-factor-authentication-2fa/configuring-two-factor-authentication)).
    - Protected default branch and protected release tags, for example with
      [rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets).
    - Every third-party GitHub Action and reusable workflow pinned to a full
      commit SHA, including `logos-modules-release-action`
      ([Secure use of GitHub Actions](https://docs.github.com/en/actions/reference/security/secure-use)).
      The `@v1` and `@v4` references in a fresh fork of
      `logos-modules-release-base` must be replaced with SHAs.
    - The signing key is stored only as an Actions secret, in an HSM or on a
      hardware token. It is never committed, never printed in logs, and never
      exposed to workflows triggered by pull requests.
09. **Secure development basics.**
    - No plaintext private keys, seed phrases or tokens in the repository, logs,
      screenshots or demo videos.
    - Dependencies are pinned with committed lockfiles (`Cargo.lock`,
      `flake.lock`, and so on).
    - Every dependency is checked to exist and to be the intended package,
      especially names suggested by AI assistants.
10. **Safe for reviewers to run.** Reviewers run deliverables on isolated
    machines with testnet funds only. The build and demo scripts must work
    without `--accept-flake-config`, without extra Nix substituters or trusted
    keys, without piping unpinned remote scripts into a shell, and without
    access to any of the reviewer's credentials. Content in a deliverable that
    tries to instruct AI tools used in review is grounds for rejecting the
    milestone.

### Tier 2 requirements

In addition to the baseline:

1. **Threat model** in the repository, delivered in the first milestone: assets,
   actors, trust boundaries, and what happens when a co-installed module, a peer
   or untrusted input (chat messages, other agents) is adversarial.
2. **Approval enforced in the core module.** The module that holds keys requires
   user approval for every transfer, signature and key export, and checks which
   module is calling. Approval that only lives in the calling UI does not count.
3. **No plaintext keys.** Keys are encrypted at rest with a user secret or held
   in the OS keystore, and opening the wallet requires unlocking.
4. **Spend caps outside the agent.** For autonomous agents, spending limits are
   enforced by the wallet module or the on-chain program, where the agent's own
   logic and prompts cannot raise them.
5. **Independent review.** The proposal includes an audit or independent
   security review milestone covering the delivered build.

### Post-delivery obligations

For the period set in the Grant Agreement, the team:

- publishes only signed releases from the declared key;
- never replaces or deletes published release assets;
- reports any known vulnerability, or any compromise of a signing key or of an
  account with write access, within 72 hours, by commenting on the proposal
  issue (without exploit details) or through a private channel provided by
  Logos.

### How review results are worded

A milestone review reads as "evaluated `rootHash` X on date Y against the
criteria of RFP-XXX". It is not an approval, verification, audit or statement
that a module is safe.

> **Important** Logos never sends install links or catalog URLs by direct
> message. Report impersonation in the proposal issue or on Discord.

______________________________________________________________________

## Terms & Conditions

All participants are bound by the
[Terms & Conditions](./TERMS_AND_CONDITIONS.md). Key points (summary only):

- Logos acts solely as grantor. It does not custody users' assets and does not
  build, deploy, host, operate or control software developed under the Program.
- RFP specifications are proposals, not instructions, and are not legal,
  regulatory, financial or technical advice. Publishing an RFP is not a
  commitment nor an intention by Logos to build, deploy, operate or maintain
  anything.
- Teams act independently and are responsible for what they build. Logos makes
  no representation as to their legal or regulatory treatment, and each team
  must obtain its own advice before implementing one. Teams retain full
  discretion over the approach to build any software under the Program.
- Anyone who deploys, hosts, operates or uses software developed under the
  Program does so at their own risk and on their own legal and regulatory
  responsibility.
- Submissions must be open sourced under MIT and Apache 2.0 unless otherwise
  agreed, and are public and non-confidential.
- Logos retains sole discretion over evaluation and Grant awards.
- No Grant is awarded and no obligation to provide any Grant arises unless a
  Grant Agreement is executed, and acceptance of a submission is not an
  endorsement, audit, or security review or warranty.

See the full [Terms & Conditions](./TERMS_AND_CONDITIONS.md) for eligibility,
IP, liability, and other provisions.

______________________________________________________________________

## License

Licensed under either of:

- [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0)
- [MIT License](http://opensource.org/licenses/MIT)

at your option.

## Formatting

Markdown files in this repository are formatted with
[mdformat](https://mdformat.readthedocs.io/). The configuration lives in
`.mdformat.toml` at the repo root.

Install mdformat along with the GitHub-Flavored Markdown and frontmatter
plugins:

```bash
pip install mdformat mdformat-gfm mdformat-frontmatter
```

Format all markdown files before committing:

```bash
mdformat .
```

Please run `mdformat` on any markdown you change before opening a pull request,
so reviews focus on content rather than formatting noise.

## How to add a new RFP

To create a new Request For Proposal, follow the
[RFP creation guide](/NEW_RFP.md)
