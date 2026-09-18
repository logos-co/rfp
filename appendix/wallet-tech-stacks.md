# Appendix: Wallet Application Technology Stacks

This appendix surveys what major multichain crypto wallets are actually built
with on desktop and mobile: the UI framework, the language, and where a native
or C++ core sits underneath. It is a separate concern from wallet key
management, which is surveyed in
[Appendix: Wallet Libraries and Key Management](./wallet-libraries-ecosystem.md).
That appendix answers how a client comes to hold keys and produce signed bytes;
this one answers what the application around those keys is written in.

The distinction matters for a team choosing an application framework. Key
management is portable across stacks, because the cores are C++ or Rust with
bindings. The UI layer is not: it determines hiring, the toolchain, and whether
the same code reaches a browser extension, a desktop binary, and two app stores.

Sections 1 to 4 hold to the sourcing standard used across these appendices:
every claim carries a first-party source that was fetched and confirmed to
resolve, and where something could not be confirmed the entry reads
`[NOT FOUND]`, which records the limit of the sourcing rather than an assertion
of absence.

Section 5 does not. It surveys the infrastructure the integrator categories run,
is drafted from secondary material, and is pending fact-check. It is marked as
such at its head and carries no references.

## Method, and why there is no ranking

Verifiability is uneven across this market, and it splits on one line: whether
the application code is public. For open-source wallets the stack is read
directly from the manifest that builds it, a `package.json` or `pubspec.yaml`,
or from the repository language breakdown. For closed-source wallets there is
usually no first-party statement of the stack at all, and the entry is
`[NOT FOUND]`.

No ordering is imposed on the wallets below. A ranking would need a published,
dated basis such as download counts or install figures, and no such first-party
ranking of multichain wallets was located. Wallets are therefore listed
alphabetically within each section, and no share-of-market or
percentage-of-wallets figure is asserted anywhere in this appendix.

## 1. Desktop and browser extension

| Wallet               | Stack                          | Source public | Evidence      |
| -------------------- | ------------------------------ | ------------- | ------------- |
| Exodus               | [NOT FOUND] for the app itself | no            | [1] [2]       |
| Ledger Live Desktop  | Electron, React, TypeScript    | yes           | [3] [4]       |
| MetaMask (extension) | TypeScript                     | yes           | [5]           |
| MetaMask Desktop     | Electron, TypeScript; archived | yes           | [6] [7]       |
| OKX                  | [NOT FOUND] for the app itself | no            | [8]           |
| Rabby (extension)    | TypeScript                     | yes           | [9]           |
| Rabby Desktop        | Electron, React, TypeScript    | yes           | [10] [11]     |
| Zerion (extension)   | TypeScript                     | yes           | [12]          |
| Bitget, Coinbase,    |                                |               |               |
| Keplr, Phantom       | [NOT FOUND] for desktop        | see §3        | [13] [14] [8] |

Two entries in the starting hypothesis for this survey did not survive checking,
and both concern desktop apps that no longer ship.

### 1.1 MetaMask Desktop is archived

The `MetaMask/metamask-desktop` repository is archived and read-only, with its
last push on 2 October 2024 [6]. It was never a general desktop wallet. The
README states in full capitals that "METAMASK DESKTOP IS AN EXPERIMENTAL
FEATURE. IT CAN ONLY BE USED WITH FLASK, THE CANARY DISTRIBUTION OF THE METAMASK
EXTENSION, INTENDED FOR DEVELOPERS" [7], and the repository description calls it
"a companion app that improves the overall performance of the MetaMask Extension
Flask build" [6]. It was an Electron companion process for a canary build, not a
product, and it is discontinued. MetaMask's shipping desktop presence is the
browser extension [5].

### 1.2 Rabby Desktop still has an open repository

The picture here is more equivocal, and worth stating precisely rather than
resolving. `RabbyHub/RabbyDesktop` is **not** archived, and its README describes
the stack directly: "Rabby desktop uses Electron, React, React Router, Webpack
and React Fast Refresh" [10]. The most recent release listed is v0.41.3 [11].

Reports that the Rabby Desktop backend API was switched off, after which the app
ceases to function, trace to a single post on X that could not be fetched, and
to third-party news coverage of it. Neither meets the sourcing standard used
here, so the shutdown is `[NOT FOUND]`. What is confirmed first-party is
narrower: the repository remains unarchived and open, and Rabby's extension
continues under active development [9]. A team should not read the open
repository as evidence that the desktop product is live.

### 1.3 Electron is the desktop pattern where desktop code is visible

Among desktop wallets whose source is public, both build on Electron with React:
Ledger Live Desktop declares `electron` in devDependencies with `react` and
`react-dom` as dependencies [4], and Rabby Desktop states the same combination
[10]. This is a real pattern, but the sample it rests on is small, and it should
not be extended to the closed-source desktop wallets, for which no stack is
confirmed.

One element of the hypothesis behind this survey is contradicted outright. The
`ledger-live` monorepo contains no C++ whatsoever: its language breakdown is
TypeScript at 60,995,989 bytes and JavaScript at 12,282,944, with Kotlin and
Swift present only in the low tens of kilobytes [3]. A "C++ core" is not part of
this repository.

## 2. Mobile

| Wallet             | Stack                                  | Source public | Evidence |
| ------------------ | -------------------------------------- | ------------- | -------- |
| Bitget             | [NOT FOUND]                            | no            | [13]     |
| Coinbase Wallet    | [NOT FOUND], see §2.3                  | no            | [14]     |
| Exodus             | [NOT FOUND] for the app itself         | no            | [1] [2]  |
| Keplr              | [NOT FOUND], see §2.4                  | partly        | [15]     |
| Ledger Live Mobile | React Native, Expo, TypeScript         | yes           | [16]     |
| MetaMask Mobile    | React Native 0.83.6, Expo 55, React 19 | yes           | [17]     |
| OKX                | [NOT FOUND] for the app itself         | no            | [8]      |
| Phantom            | React Native, Expo, TypeScript         | no            | [18]     |
| Rainbow            | React Native 0.81.6, Expo 54           | yes           | [19]     |
| SafePal            | Flutter, Dart                          | yes           | [20]     |
| Trust Wallet       | C++ core, Swift and Kotlin bindings    | core only     | [21]     |
| Zerion             | Swift core on iOS, see §2.5            | partly        | [22]     |

### 2.1 React Native with Expo, read from manifests

Four mobile wallets are confirmed as React Native from the `package.json` that
builds them, and in every case Expo is present rather than bare React Native:

| Wallet             | react-native | expo    | Source |
| ------------------ | ------------ | ------- | ------ |
| MetaMask Mobile    | 0.83.6       | ^55.0.0 | [17]   |
| Rainbow            | 0.81.6       | 54.0.33 | [19]   |
| Ledger Live Mobile | catalog      | catalog | [16]   |
| Phantom            | yes          | yes     | [18]   |

Expo's presence in MetaMask Mobile is worth flagging, since React Native
adoption is often described as bare. MetaMask declares `expo` alongside some
eighteen `expo-*` modules including `expo-local-authentication`,
`expo-secure-store` equivalents, and `expo-updates` [17].

Phantom is closed source, but its stack is documented in a first-party
engineering case study carrying named Phantom engineers. Software engineer Jakub
Adamczyk states that "we don't have a lot of engineers with background in React
Native, and yet nearly every engineer at Phantom is able to contribute to the
mobile app", and the study lists the stack as Expo SDK, React Native, React
Query, TypeScript, FlashList and Expo Modules in a monorepo shared with the
browser extension [18]. Phantom writes native code where it must, for WebView
patches and Solana Mobile Wallet Adapter support [18].

### 2.2 Trust Wallet: a C++ core, and apps that are no longer open

Trust Wallet is the genuine outlier, but not quite in the way it is usually
described. What is open is the core, not the apps.

`trustwallet/wallet-core` is active and C++-dominant: C++ at 5,120,432 bytes,
Rust at 4,172,855, C at 1,720,628, then Swift at 830,624 and Kotlin at 642,044
[23]. The README describes it as "an open-source, cross-platform, mobile-focused
library implementing low-level cryptographic wallet functionality for a high
number of blockchains", exposing "idiomatic interfaces for supported languages:
Swift for iOS and Java (Kotlin) for Android", covering "more than 130"
blockchains [21].

The applications, however, were withdrawn from public source years ago.
`trust-wallet-ios` is archived, Swift, last pushed 8 May 2019 [24], and
`trust-wallet-android-source` is archived, Java, and titled "Trust - Ethereum
Wallet for Android (Inactive Repository)" [25]. So "native Swift and Kotlin" is
confirmed for the archived 2019 apps and for the binding layer the current apps
consume, while the stack of the shipping apps is `[NOT FOUND]`.

The pattern the core demonstrates is nonetheless the one relevant to a
cross-language decision: a single C++ implementation reaching Swift and Kotlin
targets, consumed by wallets beyond Trust itself, with the README naming
Crypto.com, Frontier and Tokenary among its users [21].

### 2.3 Coinbase Wallet could not be sourced

Coinbase has published engineering posts on its React Native transition, and the
mobile app source is not public: no repository in the `coinbase` GitHub
organisation is the Wallet mobile app or extension [14]. Every Coinbase blog URL
attempted, on both `coinbase.com/blog` and the Medium mirror, returned HTTP 403
to fetching, so no Coinbase statement could be confirmed to resolve. Under the
standard used here the stack is therefore `[NOT FOUND]`, recording a sourcing
limit rather than absence: the posts exist, but they could not be fetched and
quoted.

### 2.4 Keplr is partly open, and its app repository moved

Keplr is a partial case. The `chainapsis/keplr-wallet` repository, which
previously held both extension and mobile packages, now returns 404 and does not
appear in the organisation's public repository listing; what remains public are
registries and examples such as `keplr-chain-registry` and `keplr-example` [15].
The stack of the shipping Keplr apps is `[NOT FOUND]`.

### 2.5 Zerion is native on iOS, not React Native

Zerion contradicts the assumption that every recent mobile wallet is React
Native. Its published core for Apple platforms, `zeriontech/wallet-core-ios`, is
Swift and is described as "This repository contains Wallet Core used by Zerion
iOS and macOS apps" [22], with an Android counterpart maintained separately
[26]. The browser extension is open source and TypeScript [12]. The full
application stack is not public, so the UI layer is `[NOT FOUND]`, but a Swift
core used by the iOS and macOS apps is confirmed.

### 2.6 SafePal is Flutter

SafePal's app repository is Dart under GPL-3.0 and has the structure of a
Flutter project, containing `pubspec.yaml`, `lib/`, `ios/` and `android/` [20].
Its last push was 15 January 2024, so it evidences the stack at that date rather
than necessarily the current build.

## 3. Open versus closed, and what that costs the survey

Whether a stack is knowable at all tracks almost exactly with whether the app is
open source.

| Wallet                | App source                              | Stack confirmed  |
| --------------------- | --------------------------------------- | ---------------- |
| Ledger Live           | open, MIT [3]                           | yes              |
| MetaMask              | open, extension and mobile [5] [17]     | yes              |
| Rabby                 | open, extension and desktop [9] [10]    | yes              |
| Rainbow               | open, GPL-3.0 [19]                      | yes              |
| SafePal               | open, GPL-3.0 [20]                      | yes              |
| Zerion                | extension open; apps closed [12] [22]   | partly           |
| Trust Wallet          | core open; apps archived 2019 [21] [24] | partly           |
| Keplr                 | app repo withdrawn [15]                 | no               |
| Phantom               | closed, but documented [18]             | yes, first-party |
| Exodus                | libraries only [1] [2]                  | no               |
| Bitget, Coinbase, OKX | closed [13] [14] [8]                    | no               |

Two cases show that the two properties are not identical. Phantom is closed
source yet its stack is well evidenced, because the company published a case
study with named engineers [18]. Exodus is the reverse: it publishes 150-plus
repositories, and its open-source documentation page describes Hydra as "our
modular, cross-platform framework for building wallets" [2], but that page makes
no statement about which stack the products ship on, and the app code is
proprietary [1]. Exodus does maintain libraries whose names imply both runtimes,
`electron-ipc-broadcast` ("Broadcast IPC messages to all Electron processes")
and `shakl` ("A utility to create styled components in React Native") [27]. That
is suggestive, not a first-party statement about the shipping apps, so Exodus
remains `[NOT FOUND]`.

OKX illustrates a common confusion. OKX open-sourced wallet *SDKs*,
`js-wallet-sdk` and `go-wallet-sdk`, which are signing libraries under MIT [8].
That is not the wallet application, and it gives no evidence of the app's UI
stack.

## 4. Qt and QML in this space

The question this section records is narrow: does anyone ship a production
cryptocurrency wallet on Qt/QML? The answer is yes, in three confirmed cases,
none of them among the multichain wallets above.

| Project      | QML bytes  | Other            | Platforms          | Source    |
| ------------ | ---------- | ---------------- | ------------------ | --------- |
| Status       | 11,290,290 | Nim 4,792,264    | desktop and mobile | [28] [29] |
| Monero GUI   | 1,112,330  | C 4,244,094      | desktop            | [30] [31] |
| Electrum     | 628,431    | Python 7,007,013 | desktop, Android   | [32] [33] |
| Bitcoin Core | none       | C++ 14,912,711   | desktop            | [34]      |

**Status** is the most directly relevant data point, because it is the only
confirmed case of a QML wallet reaching both mobile app stores. Its README
describes the product as "a privacy-first, decentralised messenger built with
Nim and Qt/QML", lists Google Play and the Apple App Store among its downloads,
and states support for "Android 9 to 16" and "iOS 17 and higher" alongside
Windows, Linux and macOS builds [29]. The `status-desktop` repository, despite
its name, is the codebase carrying both: QML is its largest language at
11,290,290 bytes, ahead of Nim at 4,792,264 [28]. Its predecessor
`status-mobile` is a different stack entirely, Clojure at 6,038,185 bytes with
no QML present at all [35], so the QML app is the current direction rather than
a legacy artefact.

**Monero GUI** is QML on the desktop, with QML its second language at 1,112,330
bytes behind vendored C [30]. Its README requires Qt 5.12 as a minimum and lists
the QML modules the build needs, including `qml-module-qtquick-controls2`,
`qml-module-qt-labs-settings` and `qml-module-qtquick-templates2` [31].

**Electrum** ships QML on Android specifically. The repository carries 628,431
bytes of QML [32], `electrum/gui/` contains both a `qml` and a `qt` directory
[33], and the Android build is driven by a spec file named `buildozer_qml.spec`
[36]. The desktop interface is the separate Qt path, whose README instructs
installing `python3-pyqt6` [37]; notably no Kivy remains, which had been the
earlier Android GUI.

**Bitcoin Core** is the counter-example and confirms the hypothesis that it is
not QML. Its language breakdown contains no QML at all [34], its GUI being Qt
Widgets.

Two observations follow, offered as findings rather than as a recommendation.
The QML wallets are all single-asset or messenger-first projects, and none is a
multichain wallet of the kind surveyed in sections 1 and 2, so QML has no
confirmed presence in that specific market. At the same time, the claim that QML
cannot reach mobile app stores is contradicted by Status, which ships a Nim+QML
codebase to both [29]. The distribution question and the market-convention
question have different answers, and a decision that conflates them will misread
the evidence.

## 5. Integrator infrastructure, unverified

> **Sourcing note.** Unlike every section above, this one is **not** sourced to
> first-party evidence and carries no references. It is a working draft pending
> fact-check. Most of the organisations below publish no application code, so
> their stacks are not verifiable from a repository the way a wallet's is, and
> the claims here are likely to include errors of the kind the wallet sections
> corrected. Treat the whole section as a hypothesis to check rather than as
> established fact.

This covers the parties that integrate a chain rather than the wallets that hold
keys: the integrator categories RFP-027 writes its requirements against.

### 5.1 Centralised exchanges

| Exchange | Architecture       | Languages and technologies                                                                                     |
| -------- | ------------------ | -------------------------------------------------------------------------------------------------------------- |
| Coinbase | Microservices      | Go (primary), Ruby on Rails (legacy), React/TypeScript front end, PostgreSQL, MongoDB, AWS, Docker, Kubernetes |
| Binance  | Java microservices | Java/Spring Boot, Go, Python, cloud-native, distributed matching engine                                        |
| Kraken   | Rust-first         | Rust (default), C++ (trading engine), Python, PHP (legacy), Tokio async, AWS                                   |

Reported patterns: C++ or Rust for latency-critical matching engines; Go, Java
or Node.js for REST and WebSocket API layers; PostgreSQL as the primary store
with Redis for caching and MongoDB for logs; AWS with Kubernetes and Docker.

### 5.2 Custodians

| Custodian         | Model                           | Languages                                    | Key technologies                                                                       |
| ----------------- | ------------------------------- | -------------------------------------------- | -------------------------------------------------------------------------------------- |
| Fireblocks        | MPC-CMP, multi-cloud            | Go, Python, TypeScript                       | MPC-CMP, AWS Nitro, GCP Confidential Spaces, Intel SGX, HSMs (Thales, Securosys)       |
| Anchorage Digital | OCC-regulated qualified custody | Go, C, C++                                   | HSMs, post-quantum cryptography, SOC 1/2 Type II, Porto API, Terraform, Vault, GraphQL |
| Copper            | MPC plus ClearLoop              | Ruby on Rails, Ember.js                      | MPC 2-of-3 quorum signing, PostgreSQL, OpenSearch, Redis, Resque, secure enclaves      |
| BitGo             | Multi-sig plus MPC              | Go, TypeScript/Node.js, Java, Python, Kotlin | 2-of-3 multi-signature, MPC, HSM-managed keys, regulated trust entities                |

Reported patterns: Go dominates custody back ends (Fireblocks, Anchorage,
BitGo); Python for scripting, MPC and chain integrations; C and C++ for
cryptographic hot paths and HSM integration; TypeScript for SDKs and API layers.
Copper's Ruby on Rails is the outlier.

### 5.3 Payment gateways

| Gateway           | Stack                     | Technologies                                                                                            |
| ----------------- | ------------------------- | ------------------------------------------------------------------------------------------------------- |
| BitPay            | Node.js full stack        | Node.js, Bitcore, SDKs in PHP, Java, Python and C#, REST API, chain indexing                            |
| Coinbase Commerce | Unified payments          | React, Node.js, Coinbase Wallet API, gasless USDC checkout                                              |
| Stripe (Bridge)   | Stablecoin infrastructure | Bridge API (acquired 2024), orchestration layer, compliance abstraction, multi-chain stablecoin routing |

### 5.4 Data and price aggregators

| Aggregator    | Stack                      | Technologies                                                                                                       |
| ------------- | -------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| CoinGecko     | Rails and PostgreSQL       | Ruby on Rails, PostgreSQL, Redis, Cloudflare, AWS EC2/RDS, BigQuery, Tailwind, TypeScript and Python SDKs          |
| CoinMarketCap | API platform               | REST API, WebSocket feeds, AWS, caching layers, proprietary normalisation                                          |
| DeFiLlama     | Open-source TVL aggregator | JavaScript/Node.js, on-chain indexing, custom price service over CoinGecko/CMC and DEX pools, multi-chain adapters |

### 5.5 Node and RPC providers

| Provider           | Architecture            | Technologies                                                                                             |
| ------------------ | ----------------------- | -------------------------------------------------------------------------------------------------------- |
| Alchemy            | Supernode               | Node.js, Rust, Go, multi-region AWS/GCP, custom node infrastructure, JSON-RPC and WebSocket, 100+ chains |
| Infura (Consensys) | Ethereum infrastructure | Geth and OpenEthereum clients, AWS, JSON-RPC over HTTPS and WSS, IPFS gateway                            |
| QuickNode          | Global RPC network      | Multi-client, 83+ chains and 140+ networks, Streams and Webhooks, gRPC for Solana, 17+ regions           |

Reported patterns: client diversity (Geth, Nethermind, Besu, Erigon); edge nodes
across many regions for latency; JSON-RPC, REST, WebSocket, gRPC and GraphQL
side by side; webhooks and custom indexing as add-ons.

### 5.6 Fiat on and off ramps

| Provider | Integration             | Technologies                                                                                 |
| -------- | ----------------------- | -------------------------------------------------------------------------------------------- |
| MoonPay  | Embedded widget and API | JavaScript SDK, headless widget, card processing via Stripe/Checkout.com, bank transfer APIs |
| Transak  | Whitelabel API          | React widget, whitelabel API, Visa Direct, virtual accounts, webhooks                        |
| Onramper | Aggregator router       | Aggregator API over 30+ onramps, smart routing, single integration                           |

### 5.7 Bridges

| Bridge    | Model               | Technologies                                                                                           |
| --------- | ------------------- | ------------------------------------------------------------------------------------------------------ |
| LayerZero | Omnichain messaging | Decentralised Verifier Network, Ultra Light Node, oracle plus relayer, OFT/ONFT standards, 130+ chains |
| Wormhole  | Guardian network    | 19-validator guardian set, Verified Action Approval, Native Token Transfer, 45+ chains                 |
| Across    | Intent-based        | UMA Optimistic Oracle, competitive relayer network, request for quote, optimistic settlement           |

### 5.8 Tax and accounting providers

| Provider    | Model                  | Technologies                                                                                  |
| ----------- | ---------------------- | --------------------------------------------------------------------------------------------- |
| Koinly      | Automated tax platform | 850+ API integrations, CSV import, DeFi parsers, cost-basis calculation, read-only API access |
| CoinTracker | Portfolio and tax      | 500+ integrations, TurboTax and H&R Block export, cost-basis optimisation                     |
| TaxBit      | Enterprise compliance  | API-driven tax engine, ERP integration, Form 1099-DA, DAC8/CARF, 70+ jurisdictions            |

Reported patterns: ingestion over read-only exchange APIs, block explorers and
CSV; cost-basis calculation across FIFO, LIFO and HIFO; reporting into IRS forms
and consumer tax software; ERP connectors for enterprise.

### 5.9 Cross-cutting patterns

| Layer          | Reported choices                                         |
| -------------- | -------------------------------------------------------- |
| Front end      | React, React Native, TypeScript, Tailwind                |
| Back end       | Node.js, Go, Java, Python, Rust                          |
| Databases      | PostgreSQL, MongoDB, Redis, BigQuery                     |
| Chain access   | web3.js, ethers.js, viem, Solana web3.js                 |
| Infrastructure | AWS, GCP, Azure, Kubernetes, Docker                      |
| Key security   | HSMs, MPC, secure enclaves (SGX, Nitro), multi-signature |
| API surfaces   | REST, GraphQL, WebSocket, gRPC, JSON-RPC                 |

## References

01. Exodus, "Is Exodus open source?" Knowledge Base article.
    https://www.exodus.com/support/en/articles/8598678-is-exodus-open-source
02. Exodus, "Open Source at Exodus" documentation hub.
    https://docs.exodus.com/open-source
03. Ledger, "ledger-live" monorepo, GitHub API repository and language
    endpoints. https://api.github.com/repos/LedgerHQ/ledger-live/languages
04. Ledger, "ledger-live-desktop" package manifest.
    https://raw.githubusercontent.com/LedgerHQ/ledger-live/develop/apps/ledger-live-desktop/package.json
05. MetaMask, "metamask-extension" repository metadata.
    https://api.github.com/repos/MetaMask/metamask-extension
06. MetaMask, "metamask-desktop" repository metadata, archived.
    https://api.github.com/repos/MetaMask/metamask-desktop
07. MetaMask, "metamask-desktop" repository README.
    https://github.com/MetaMask/metamask-desktop
08. OKX, "js-wallet-sdk" and "go-wallet-sdk" repositories.
    https://github.com/okx/js-wallet-sdk
09. Rabby, "Rabby" extension repository metadata.
    https://api.github.com/repos/RabbyHub/Rabby
10. Rabby, "RabbyDesktop" repository and README.
    https://github.com/RabbyHub/RabbyDesktop
11. Rabby, "RabbyDesktop" releases listing.
    https://github.com/RabbyHub/RabbyDesktop/releases
12. Zerion, "zerion-wallet-extension" repository metadata.
    https://api.github.com/repos/zeriontech/zerion-wallet-extension
13. Bitget Wallet, GitHub organisation. https://github.com/bitgetwallet
14. Coinbase, GitHub organisation repository listing.
    https://api.github.com/orgs/coinbase/repos
15. Chainapsis, GitHub organisation repository listing.
    https://api.github.com/orgs/chainapsis/repos
16. Ledger, "ledger-live-mobile" package manifest.
    https://raw.githubusercontent.com/LedgerHQ/ledger-live/develop/apps/ledger-live-mobile/package.json
17. MetaMask, "metamask-mobile" package manifest.
    https://raw.githubusercontent.com/MetaMask/metamask-mobile/main/package.json
18. Expo, "How Phantom ships a secure, high-performance crypto wallet with
    Expo", with quotes from named Phantom engineers.
    https://expo.dev/blog/how-phantom-ships-a-secure-high-performance-crypto-wallet-with-expo
19. Rainbow, "rainbow" repository metadata and package manifest.
    https://raw.githubusercontent.com/rainbow-me/rainbow/develop/package.json
20. SafePal, "safepal-app" repository metadata and top-level contents.
    https://api.github.com/repos/SafePalWallet/safepal-app/contents/
21. Trust Wallet, "wallet-core" repository README.
    https://github.com/trustwallet/wallet-core
22. Zerion, "wallet-core-ios" repository metadata.
    https://api.github.com/repos/zeriontech/wallet-core-ios
23. Trust Wallet, "wallet-core" language breakdown.
    https://api.github.com/repos/trustwallet/wallet-core/languages
24. Trust Wallet, "trust-wallet-ios" repository metadata, archived.
    https://api.github.com/repos/trustwallet/trust-wallet-ios
25. Trust Wallet, "trust-wallet-android-source" repository metadata, archived.
    https://api.github.com/repos/trustwallet/trust-wallet-android-source
26. Zerion, "wallet-core-android" repository.
    https://github.com/zeriontech/wallet-core-android
27. Exodus, "ExodusOSS" organisation repository listing.
    https://api.github.com/orgs/ExodusOSS/repos
28. Status, "status-desktop" language breakdown.
    https://api.github.com/repos/status-im/status-desktop/languages
29. Status, "status-desktop" repository README.
    https://github.com/status-im/status-desktop
30. Monero, "monero-gui" language breakdown.
    https://api.github.com/repos/monero-project/monero-gui/languages
31. Monero, "monero-gui" repository README.
    https://github.com/monero-project/monero-gui
32. Electrum, "electrum" language breakdown.
    https://api.github.com/repos/spesmilo/electrum/languages
33. Electrum, "electrum/gui" directory listing.
    https://api.github.com/repos/spesmilo/electrum/contents/electrum/gui
34. Bitcoin Core, "bitcoin" language breakdown.
    https://api.github.com/repos/bitcoin/bitcoin/languages
35. Status, "status-mobile" language breakdown.
    https://api.github.com/repos/status-im/status-mobile/languages
36. Electrum, "contrib/android" directory listing, containing
    `buildozer_qml.spec`.
    https://api.github.com/repos/spesmilo/electrum/contents/contrib/android
37. Electrum, "electrum" repository README. https://github.com/spesmilo/electrum
