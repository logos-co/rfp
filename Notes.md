## Review

**What comes back.** Instead of a blob, provide a table with a row per keys with common function, description and chain.

[1] etc can we have anchors that work in Obsidian and/or GitHub markdown?


## RFP Notes


- 1.1. LEZ missing node version API
- 1.2 Logos Blockchain missing sync status? Seems incorrect
- 1.3 Network identification
	- LEZ returns channel id but not chain id of underlying Logos Blockchain
	- Chain id missing for Logos Blockchain
	- getblockchaininfo -> how is BTC diffferentiated from BCH?
- 1.4 -> Logos has a modern approach
- 1.7 - Logos Blockchain is UTXO based - how does it compare to Bitcoin?
	- Potential gap `getMulitpleAccounts` for LEZ?
- 1.10 makes sense: no contract on L1, and LEZ is similar to Solana in this aspect - the question is whether one can read PDA data? I assume so via `getAccount`. Gap seems to be `simulateTransaction`
- 1.12: do we need something similar to `listunspent` ? How does it play with LEZ private pre-states and post-states? aka, ensuring pre-states can be fetched.
- 1.13: seems fine... as long as it works for private tx (eg pre-states access)
- 1.15: `simulateTransaction` for LEZ needed, `testmempool` accept for Logos Blockchain?
- 1.16 `simulateTransaction` for LEZ should cover this. not sure if there's a difference between public and private. estimage gas probably needed for L1, for both execution and storage
- 1.17: dependends on LEZ gas model. So i'd say yes as aplace holder
- 1.18, 1.19, 1.20 seems non necessary but something like EIP-712 that can enable specific account ownership checks is probably needed for LEZ
- 1.22: might be useful especially for LEZ and private accounts (state changes mid proof generation, so cancel instead of generating longer)
- 1.23 doesn't seem critical, unless chaining private tx (with specific tx1.poststate = tx2.prestate )
- 1.24: probably need to ask more precise Logos Blockchain API
- 1.25 sounds fair to have a subscribe in place, depends on transport
- 1.26 clear gap
- 1.27 is needed to calculate post-state - are we sure it's not existing? how does the wallet calculate post-state? I guess local execution making it an unnecessary API? to be thought through
- 1.28 we did do a lamda prize re logs, so we need to review this in context. `getTransaction` might just be it
- 1.31 seems needed for logos l1, and pagination enhancement for lez
- 1.33 seems to be very architecture dependent, need to carefully review
- 1,34 seems needed