# Appendix: Blockchain API and SDK Ecosystem

This appendix surveys what blockchain node APIs and client SDKs expose across
eight established ecosystems: which functions they offer, which transports they
serve them over, and which SDK languages they publish. It provides context for
the blockchain API and SDK RFPs, covering the API surface itself, the JSON-RPC
proxy and TypeScript SDK that consume it, and the further interface modules and
external SDKs built on top. Logos L1 and LEZ appear as the final rows of each
table, recording what they expose today.

Every claim carries a first-party source that was fetched and confirmed to
resolve. Where a capability could not be confirmed on official documentation,
the entry reads `[NOT FOUND]`: that records the limit of the sourcing, not an
assertion that the capability is absent. Logos rows cite files and line numbers
in `logos-blockchain` at commit `ecb2cc6` and `logos-execution-zone` at commit
`47eba25`.

## Chains Surveyed

Chains are ordered by integration ubiquity, meaning how commonly an exchange,
custodian, or payment provider must integrate them, and this order is maintained
throughout the document. The set spans account and UTXO models and every major
RPC style.

| Chain    | State model | Primary node transport                  | Machine-readable contract         |
| -------- | ----------- | --------------------------------------- | --------------------------------- |
| Ethereum | Account     | JSON-RPC over HTTP and WebSocket [1]    | OpenRPC [2]                       |
| Bitcoin  | UTXO        | JSON-RPC over HTTP, plus REST [3][4]    | none found                        |
| Solana   | Account     | JSON-RPC over HTTP and WebSocket [5]    | none found                        |
| XRPL     | Account     | JSON-RPC over HTTP and WebSocket [6]    | protobuf, internal only [7]       |
| Cosmos   | Account     | gRPC, REST, and JSON-RPC [8][9]         | protobuf and OpenAPI [10][9]      |
| Stellar  | Account     | JSON-RPC over HTTP [11]                 | OpenRPC [12]                      |
| NEAR     | Account     | JSON-RPC over HTTP [13]                 | OpenAPI [14]                      |
| Sui      | Object      | gRPC, with JSON-RPC deprecated [15][16] | protobuf and GraphQL SDL [17][18] |
| Logos L1 | Note based  | REST over HTTP [19]                     | OpenAPI [20]                      |
| LEZ      | Account     | JSON-RPC over HTTP and WebSocket [21]   | runtime `getSchema` only [22]     |

All eight external chains expose the same broad capability families, so the
interesting differences are in how each is shaped rather than whether it exists.

## 1. API Functions

Thirty-four functions, ordered by family: discovery and node identity, ledger
and state reads, transaction construction, simulation and fee estimation,
signing, submission, status and confirmation, effects, history and streaming,
and errors. Each entry states what the operation does, which chains expose it
and under what name, and the documented reason it exists where one could be
found.

Rationale is sourced in only four cases across the whole catalogue: transaction
construction and offline signing (BIP-174), fee estimation (EIP-1559),
simulation (Stellar's own documentation), and the wallet error taxonomy
(EIP-1193). Every other function reads `[NOT FOUND]` under **Why it exists**.
API surfaces are, on the evidence, largely undocumented as to motivation.

### 1.1 Get node version and software identity

Returns the node's software version and build identity so a client can adapt to
what the node actually runs.

| Chain    | Method                                  |
| -------- | --------------------------------------- |
| Ethereum | `web3_clientVersion` [23]               |
| Bitcoin  | `getnetworkinfo` [24]                   |
| Solana   | `getVersion` [25]                       |
| XRPL     | `server_info` [26]                      |
| Cosmos   | `GetNodeInfo` [10]                      |
| Stellar  | `getVersionInfo` [27]                   |
| NEAR     | `status` [13]                           |
| Sui      | `LedgerService.GetServiceInfo` [15]     |
| Logos L1 | `GET /version` [19]                     |
| LEZ      | `checkHealth`, no version reported [21] |

**Why it exists.** [NOT FOUND]

### 1.2 Health and sync-status check

Reports whether the node is healthy and caught up with the chain tip, so clients
can avoid reading stale state.

| Chain    | Method                                          |
| -------- | ----------------------------------------------- |
| Ethereum | `eth_syncing` [28]                              |
| Bitcoin  | `getblockchaininfo` [24]                        |
| Solana   | `getHealth` [25]                                |
| XRPL     | `server_state` [26]                             |
| Cosmos   | `/health`, `/status`, `GetSyncing` [9][10]      |
| Stellar  | `getHealth` [27]                                |
| NEAR     | `health` [13]                                   |
| Sui      | [NOT FOUND] [15]                                |
| Logos L1 | [NOT FOUND] [19]                                |
| LEZ      | `checkHealth`, and indexer `getStatus` [21][22] |

**Why it exists.** [NOT FOUND]

### 1.3 Identify the network or chain

Returns a stable identifier for the chain the node serves, so a client cannot
accidentally sign for or read from the wrong network.

| Chain    | Method                             |
| -------- | ---------------------------------- |
| Ethereum | `eth_chainId` [28]                 |
| Bitcoin  | `getblockchaininfo` [24]           |
| Solana   | `getGenesisHash` [25]              |
| XRPL     | `server_info` [26]                 |
| Cosmos   | `/genesis`, `/genesis_chunked` [9] |
| Stellar  | `getNetwork` [27]                  |
| NEAR     | `genesis_config` [13]              |
| Sui      | `sui_getChainIdentifier` [17]      |
| Logos L1 | [NOT FOUND] [19]                   |
| LEZ      | [NOT FOUND] [21]                   |

**Why it exists.** [NOT FOUND]

Neither Logos target reports a network identifier, so a client cannot verify
from the API which network an endpoint serves.

### 1.4 Retrieve a machine-readable API description

Serves the node's own interface definition so clients can be generated or
validated against the live node.

| Chain    | Method                                                                      |
| -------- | --------------------------------------------------------------------------- |
| Ethereum | [NOT FOUND] as a runtime method; spec is a committed OpenRPC corpus [2][28] |
| Bitcoin  | [NOT FOUND] [24]                                                            |
| Solana   | [NOT FOUND] [25]                                                            |
| XRPL     | `server_definitions` [26]                                                   |
| Cosmos   | [NOT FOUND] as an RPC; contract is committed `.proto` files [29]            |
| Stellar  | [NOT FOUND] as a method; OpenRPC document published [12]                    |
| NEAR     | [NOT FOUND] as a method; OpenAPI document published [14]                    |
| Sui      | OpenRPC document published [17]                                             |
| Logos L1 | `GET /api-docs/openapi.json`, Swagger UI at `/swagger-ui` [20]              |
| LEZ      | `getSchema` on the indexer [22]                                             |

**Why it exists.** [NOT FOUND]

Both Logos targets serve a machine-readable description at runtime, which
Bitcoin and Solana do not.

### 1.5 Get a block by height or hash

Fetches one block, ledger, or checkpoint and its contents by identifier.

| Chain    | Method                                                           |
| -------- | ---------------------------------------------------------------- |
| Ethereum | `eth_getBlockByNumber`, `eth_getBlockByHash` [23]                |
| Bitcoin  | `getblock`, `getblockhash`, `getblockheader` [24]                |
| Solana   | `getBlock` [25]                                                  |
| XRPL     | `ledger` [26]                                                    |
| Cosmos   | `GetBlockByHeight`, `/block`, `/block_by_hash` [10][9]           |
| Stellar  | `getLedgers` [27]                                                |
| NEAR     | `block`, `chunk` [13]                                            |
| Sui      | `sui_getCheckpoint` [17]                                         |
| Logos L1 | `GET /cryptarchia/blocks/:id` [19]                               |
| LEZ      | `getBlock`, indexer `getBlockById` and `getBlockByHash` [21][22] |

**Why it exists.** [NOT FOUND]

### 1.6 Get the current chain tip

Returns the latest block or ledger the node considers current, the anchor for
most subsequent reads.

| Chain    | Method                                                       |
| -------- | ------------------------------------------------------------ |
| Ethereum | `eth_blockNumber` [28]                                       |
| Bitcoin  | `getbestblockhash`, `getblockcount` [24]                     |
| Solana   | `getSlot`, `getBlockHeight` [25]                             |
| XRPL     | `ledger_current`, `ledger_closed` [26]                       |
| Cosmos   | `GetLatestBlock` [10]                                        |
| Stellar  | `getLatestLedger` [27]                                       |
| NEAR     | `block` for the final block [13]                             |
| Sui      | `sui_getLatestCheckpointSequenceNumber` [17]                 |
| Logos L1 | `GET /cryptarchia/info` [19]                                 |
| LEZ      | `getLastBlockId`, indexer `getLastFinalizedBlockId` [21][22] |

**Why it exists.** [NOT FOUND]

### 1.7 Get an account or object

Reads the ledger record for one address, account, or object.

| Chain    | Method                                                              |
| -------- | ------------------------------------------------------------------- |
| Ethereum | `eth_getCode`, `eth_getStorageAt`; no single account object [30]    |
| Bitcoin  | `gettxout`; UTXO model has no account record [24]                   |
| Solana   | `getAccountInfo`, `getMultipleAccounts` [25]                        |
| XRPL     | `account_info`, `account_objects` [26]                              |
| Cosmos   | `auth Query.Account` [31]                                           |
| Stellar  | `getLedgerEntries` [27]                                             |
| NEAR     | `query` with an account request type [13]                           |
| Sui      | `sui_getObject`, `sui_multiGetObjects` [17]                         |
| Logos L1 | [NOT FOUND]; nearest is the custodial wallet balance route [19]     |
| LEZ      | `getAccount`, indexer `getAccount` and `getAccountAtBlock` [21][22] |

**Why it exists.** [NOT FOUND]

### 1.8 Get a balance

Returns the balance held by an address, for the native asset or a named token.

| Chain    | Method                                                               |
| -------- | -------------------------------------------------------------------- |
| Ethereum | `eth_getBalance` [30]                                                |
| Bitcoin  | `getbalances`, `listunspent`, wallet scoped [24]                     |
| Solana   | `getBalance`, `getTokenAccountBalance` [25]                          |
| XRPL     | `account_info`, `gateway_balances`, `account_lines` [26]             |
| Cosmos   | `bank Query.Balance`, `Query.AllBalances` [32]                       |
| Stellar  | [NOT FOUND] on Stellar RPC; served by ledger entries or Horizon [27] |
| NEAR     | `query` with `view_account` [13]                                     |
| Sui      | `suix_getBalance`, `suix_getAllBalances` [17]                        |
| Logos L1 | `GET /wallet/:public_key/balance`, custodial keys only [19]          |
| LEZ      | `getAccountBalance` [21]                                             |

**Why it exists.** [NOT FOUND]

The L1 balance read is scoped to keys the node's wallet controls rather than
being a general subject query.

### 1.9 Read a transaction by hash

Retrieves a single transaction, and normally its result, by identifier.

| Chain    | Method                                              |
| -------- | --------------------------------------------------- |
| Ethereum | `eth_getTransactionByHash` [33]                     |
| Bitcoin  | `getrawtransaction` [24]                            |
| Solana   | `getTransaction` [25]                               |
| XRPL     | `tx`, `transaction_entry` [26]                      |
| Cosmos   | `GetTx`, `/tx` [29][9]                              |
| Stellar  | `getTransaction` [27]                               |
| NEAR     | `tx` [34]                                           |
| Sui      | `sui_getTransactionBlock` [17]                      |
| Logos L1 | `GET /cryptarchia/transaction/:id` [19]             |
| LEZ      | `getTransaction`, indexer `getTransaction` [21][22] |

**Why it exists.** [NOT FOUND]

### 1.10 Read-only contract call

Executes contract or program code against current state and returns the result
without producing a transaction.

| Chain    | Method                                              |
| -------- | --------------------------------------------------- |
| Ethereum | `eth_call` [35]                                     |
| Bitcoin  | no equivalent; no contract layer [24]               |
| Solana   | `simulateTransaction`; no separate call method [25] |
| XRPL     | no equivalent among public methods [26]             |
| Cosmos   | `ABCIQuery` [10]                                    |
| Stellar  | `simulateTransaction` [27]                          |
| NEAR     | `query` with `call_function` [13]                   |
| Sui      | `sui_devInspectTransactionBlock` [17]               |
| Logos L1 | [NOT FOUND] [19]                                    |
| LEZ      | [NOT FOUND] [21]                                    |

**Why it exists.** [NOT FOUND]

### 1.11 Get the validator or committee set

Returns who is currently validating, needed for stake, governance, and
light-client verification.

| Chain    | Method                                                              |
| -------- | ------------------------------------------------------------------- |
| Ethereum | [NOT FOUND] on the execution API; consensus layer not surveyed [28] |
| Bitcoin  | no equivalent; proof of work [24]                                   |
| Solana   | `getVoteAccounts`, `getClusterNodes` [25]                           |
| XRPL     | `manifest`, `feature` [26]                                          |
| Cosmos   | `GetLatestValidatorSet`, `GetValidatorSetByHeight` [10]             |
| Stellar  | [NOT FOUND] [27]                                                    |
| NEAR     | `validators` [13]                                                   |
| Sui      | `suix_getCommitteeInfo`, `suix_getLatestSuiSystemState` [17]        |
| Logos L1 | `GET /mantle/sdp/declarations`, `GET /mantle/sdp/snapshot` [19]     |
| LEZ      | [NOT FOUND] [21]                                                    |

**Why it exists.** [NOT FOUND]

### 1.12 Fetch construction parameters

Retrieves the chain-specific freshness or ordering value a transaction must
carry to be valid.

| Chain    | Method                                                            |
| -------- | ----------------------------------------------------------------- |
| Ethereum | `eth_getTransactionCount` [30]                                    |
| Bitcoin  | `listunspent`; UTXO selection, no nonce [24]                      |
| Solana   | `getLatestBlockhash`, `isBlockhashValid` [25]                     |
| XRPL     | `account_info` for the account sequence [26]                      |
| Cosmos   | `auth Query.Account` for account number and sequence [31]         |
| Stellar  | [NOT FOUND] on Stellar RPC; sequence comes from account data [27] |
| NEAR     | `query` with `view_access_key` [13]                               |
| Sui      | `suix_getReferenceGasPrice`, `suix_getCoins` [17]                 |
| Logos L1 | `GET /mantle/gas-prices` only [19]                                |
| LEZ      | `getAccountsNonces` [21]                                          |

**Why it exists.** [NOT FOUND]

### 1.13 Build an unsigned transaction

Assembles a well-formed but unsigned transaction from an intent, ready to hand
to a signer.

| Chain    | Method                                                              |
| -------- | ------------------------------------------------------------------- |
| Ethereum | [NOT FOUND] as a node method; construction is client side [36]      |
| Bitcoin  | `createrawtransaction`, `createpsbt`, `walletcreatefundedpsbt` [24] |
| Solana   | [NOT FOUND] as a node method [25]                                   |
| XRPL     | [NOT FOUND] as a build-only method [26]                             |
| Cosmos   | `TxEncode`, `TxEncodeAmino` [29]                                    |
| Stellar  | [NOT FOUND] as a node method [27]                                   |
| NEAR     | [NOT FOUND] as a node method [13]                                   |
| Sui      | `unsafe_moveCall`, `unsafe_transferObject`, `unsafe_publish` [17]   |
| Logos L1 | [NOT FOUND]; construction is client side [19]                       |
| LEZ      | [NOT FOUND]; construction is client side [21]                       |

**Why it exists.** BIP-174 states the problem PSBT was created to solve:
"Creating unsigned or partially signed transactions to be passed around to
multiple signers is currently implementation dependent, making it hard for
people who use different wallet software from being able to easily do so." The
goal was "a standard and extensible format that can be used between clients to
allow people to pass around the same transaction to sign and combine their
signatures" [37].

### 1.14 Encode and decode transaction bytes

Converts between a structured transaction and its wire encoding, in both
directions.

| Chain    | Method                                                                 |
| -------- | ---------------------------------------------------------------------- |
| Ethereum | [NOT FOUND] as a node method [23]                                      |
| Bitcoin  | `decoderawtransaction`, `decodepsbt`, `converttopsbt` [24]             |
| Solana   | [NOT FOUND] as a node method [25]                                      |
| XRPL     | [NOT FOUND]; `server_definitions` serves the codec schema instead [26] |
| Cosmos   | `TxEncode`, `TxDecode`, `TxEncodeAmino`, `TxDecodeAmino` [29]          |
| Stellar  | [NOT FOUND] as a node method [27]                                      |
| NEAR     | [NOT FOUND] as a node method [13]                                      |
| Sui      | [NOT FOUND] as a node method [17]                                      |
| Logos L1 | [NOT FOUND] as a node method [19]                                      |
| LEZ      | [NOT FOUND] as a node method [21]                                      |

**Why it exists.** [NOT FOUND]

### 1.15 Simulate transaction execution

Runs a full transaction against current state without committing it, returning
the outcome it would have had.

| Chain    | Method                                                     |
| -------- | ---------------------------------------------------------- |
| Ethereum | `eth_simulateV1`, `eth_call` [35]                          |
| Bitcoin  | `testmempoolaccept`; acceptance rather than execution [24] |
| Solana   | `simulateTransaction` [25]                                 |
| XRPL     | `simulate` [38]                                            |
| Cosmos   | `Simulate` [29]                                            |
| Stellar  | `simulateTransaction` [39]                                 |
| NEAR     | [NOT FOUND] [13]                                           |
| Sui      | `sui_dryRunTransactionBlock` [17]                          |
| Logos L1 | [NOT FOUND] [19]                                           |
| LEZ      | [NOT FOUND] [21]                                           |

**Why it exists.** Stellar documents the motivation directly. Simulation lets a
client "Calculate resource requirements", "Validate authorization", and "Test
and analyze the potential outcomes of a transaction without actually submitting
it to the network". It also returns `transactionData`, "The recommended Soroban
Transaction Data to use when submitting the simulated transaction", which makes
simulation a construction step rather than an optional check [39].

Neither Logos target exposes simulation. Six of the eight surveyed chains do.

### 1.16 Estimate execution cost

Returns how much execution resource a transaction would consume, so the sender
can set a sufficient limit.

| Chain    | Method                                                         |
| -------- | -------------------------------------------------------------- |
| Ethereum | `eth_estimateGas` [35]                                         |
| Bitcoin  | no equivalent; cost is a function of size [24]                 |
| Solana   | `simulateTransaction` returns compute units consumed [25]      |
| XRPL     | no equivalent; fixed and scaling fee model [26]                |
| Cosmos   | `Simulate`, "for estimating gas usage" [29]                    |
| Stellar  | `simulateTransaction` returns `cost` and `minResourceFee` [39] |
| NEAR     | [NOT FOUND]; `gas_price` is a price, not a usage estimate [13] |
| Sui      | `sui_dryRunTransactionBlock` returns a gas cost summary [17]   |
| Logos L1 | [NOT FOUND] [19]                                               |
| LEZ      | [NOT FOUND] [21]                                               |

**Why it exists.** [NOT FOUND]

### 1.17 Estimate the fee or fee rate

Returns the current market price of inclusion so a client can choose a fee that
confirms without overpaying.

| Chain    | Method                                                            |
| -------- | ----------------------------------------------------------------- |
| Ethereum | `eth_feeHistory`, `eth_gasPrice`, `eth_maxPriorityFeePerGas` [40] |
| Bitcoin  | `estimatesmartfee` [24]                                           |
| Solana   | `getFeeForMessage`, `getRecentPrioritizationFees` [25]            |
| XRPL     | `fee` [26]                                                        |
| Cosmos   | [NOT FOUND] as a dedicated method; gas via `Simulate` [29]        |
| Stellar  | `getFeeStats` [27]                                                |
| NEAR     | `gas_price` [13]                                                  |
| Sui      | `suix_getReferenceGasPrice` [17]                                  |
| Logos L1 | `GET /mantle/gas-prices` [19]                                     |
| LEZ      | [NOT FOUND]; fees unimplemented [21]                              |

**Why it exists.** EIP-1559 states the problem it set out to solve: first-price
auctions require "complex fee estimation algorithms" that "often end up not
working very well, leading to frequent fee overpayment". The protocol-adjusted
base fee was introduced so that wallets could "auto-set gas fees reliably" [41].

### 1.18 Sign a transaction

Produces the signature that authorises a constructed transaction.

| Chain    | Method                                                                |
| -------- | --------------------------------------------------------------------- |
| Ethereum | `eth_signTransaction` [23]                                            |
| Bitcoin  | `signrawtransactionwithkey`, `walletprocesspsbt`, `finalizepsbt` [24] |
| Solana   | [NOT FOUND] as a node method; signing is client side [25]             |
| XRPL     | `sign`, `sign_for` [26]                                               |
| Cosmos   | [NOT FOUND] as a node method; signing is client side [29]             |
| Stellar  | [NOT FOUND] as a node method [27]                                     |
| NEAR     | [NOT FOUND] as a node method [13]                                     |
| Sui      | [NOT FOUND] as a node method [17]                                     |
| Logos L1 | `POST /wallet/sign/ed25519`, `POST /wallet/sign/zk` [19]              |
| LEZ      | [NOT FOUND] as a node method; signing is client side [21]             |

**Why it exists.** For the offline and hardware case, BIP-174 states PSBT "will
allow offline signers such as air-gapped wallets and hardware wallets to be able
to sign transactions without needing direct access to the UTXO set and without
risk of being defrauded" [37].

Node-hosted signing is the historical pattern; five of the eight surveyed chains
have no node signing method at all. Logos L1 holds key material and signs on the
node.

### 1.19 Sign an arbitrary message

Signs a non-transaction payload, used for authentication and off-chain proof of
key control.

| Chain    | Method                                               |
| -------- | ---------------------------------------------------- |
| Ethereum | `eth_sign` [23]                                      |
| Bitcoin  | `signmessage`, `signmessagewithprivkey` [24]         |
| Solana   | [NOT FOUND] as a node method [25]                    |
| XRPL     | [NOT FOUND] as a message-signing method [26]         |
| Cosmos   | [NOT FOUND] as a node method [29]                    |
| Stellar  | [NOT FOUND] as a node method [27]                    |
| NEAR     | [NOT FOUND] as a node method [13]                    |
| Sui      | [NOT FOUND] as a signing method [17]                 |
| Logos L1 | [NOT FOUND] as a distinct message-signing route [19] |
| LEZ      | [NOT FOUND] [21]                                     |

**Why it exists.** [NOT FOUND]

### 1.20 Verify a signature

Checks a signature against a message and key without touching chain state.

| Chain    | Method                                              |
| -------- | --------------------------------------------------- |
| Ethereum | [NOT FOUND] as a node method [23]                   |
| Bitcoin  | `verifymessage` [24]                                |
| Solana   | [NOT FOUND] as a node method [25]                   |
| XRPL     | `channel_verify`, scoped to channel claims [26]     |
| Cosmos   | [NOT FOUND] as a node method [29]                   |
| Stellar  | [NOT FOUND] as a node method [27]                   |
| NEAR     | [NOT FOUND] as a node method [13]                   |
| Sui      | `SignatureVerificationService.VerifySignature` [15] |
| Logos L1 | [NOT FOUND] as a client-facing verify method [19]   |
| LEZ      | [NOT FOUND] as a client-facing verify method [21]   |

**Why it exists.** [NOT FOUND]

### 1.21 Broadcast a signed transaction

Hands a signed transaction to the node for gossip and inclusion, returning an
identifier to track it by.

| Chain    | Method                                                          |
| -------- | --------------------------------------------------------------- |
| Ethereum | `eth_sendRawTransaction` [36]                                   |
| Bitcoin  | `sendrawtransaction` [24]                                       |
| Solana   | `sendTransaction` [42]                                          |
| XRPL     | `submit`, `submit_multisigned` [26]                             |
| Cosmos   | `BroadcastTx`, `/broadcast_tx_sync`, `/broadcast_tx_async` [29] |
| Stellar  | `sendTransaction` [27]                                          |
| NEAR     | `send_tx`, `broadcast_tx_async` [34]                            |
| Sui      | `sui_executeTransactionBlock` [17]                              |
| Logos L1 | `POST /mempool/add/tx` [19]                                     |
| LEZ      | `sendTransaction` [21]                                          |

**Why it exists.** [NOT FOUND]

Solana's documentation warns that "a successful response doesn't guarantee the
transaction will be processed or confirmed" [42].

### 1.22 Pre-submission acceptance check

Validates a signed transaction against mempool or node policy before
broadcasting, so a doomed transaction fails locally and cheaply.

| Chain    | Method                                                              |
| -------- | ------------------------------------------------------------------- |
| Ethereum | [NOT FOUND] as a distinct method [36]                               |
| Bitcoin  | `testmempoolaccept` [24]                                            |
| Solana   | preflight inside `sendTransaction`, `skipPreflight` to disable [42] |
| XRPL     | `simulate` [38]                                                     |
| Cosmos   | `/check_tx` [9]                                                     |
| Stellar  | `simulateTransaction` [39]                                          |
| NEAR     | [NOT FOUND] [34]                                                    |
| Sui      | `sui_dryRunTransactionBlock` [17]                                   |
| Logos L1 | [NOT FOUND]; validation happens on submission [19]                  |
| LEZ      | [NOT FOUND] as a standalone method [21]                             |

**Why it exists.** [NOT FOUND]

### 1.23 Submit a batch or package of transactions

Submits several interdependent transactions together so they are evaluated as a
unit.

| Chain    | Method                                                                    |
| -------- | ------------------------------------------------------------------------- |
| Ethereum | JSON-RPC batch only, no atomic chain semantics [43]                       |
| Bitcoin  | [NOT FOUND] on the fetched RPC reference [24]                             |
| Solana   | [NOT FOUND]; batching is within one transaction [25]                      |
| XRPL     | [NOT FOUND] [26]                                                          |
| Cosmos   | [NOT FOUND]; batching is within a transaction's messages [29]             |
| Stellar  | [NOT FOUND]; batching is within a transaction's operations [27]           |
| NEAR     | [NOT FOUND]; batching is within a transaction's actions [13]              |
| Sui      | `unsafe_batchTransaction` builds a batch rather than submitting many [17] |
| Logos L1 | [NOT FOUND] [19]                                                          |
| LEZ      | [NOT FOUND] [21]                                                          |

**Why it exists.** [NOT FOUND]

The dominant pattern is that batching is expressed inside one transaction, as
Cosmos messages, Stellar operations, NEAR actions, Solana instructions, and Sui
programmable transaction blocks, rather than as a multi-transaction submission
API.

### 1.24 Get transaction status

Reports where a submitted transaction sits: unknown, pending, included,
executed, or final, and whether it succeeded.

| Chain    | Method                                          |
| -------- | ----------------------------------------------- |
| Ethereum | `eth_getTransactionReceipt` [33]                |
| Bitcoin  | `gettransaction`, wallet scoped [24]            |
| Solana   | `getSignatureStatuses` [44]                     |
| XRPL     | `tx` [26]                                       |
| Cosmos   | `GetTx`, `/tx` [29][9]                          |
| Stellar  | `getTransaction` [27]                           |
| NEAR     | `tx`, `EXPERIMENTAL_tx_status` [34]             |
| Sui      | `sui_getTransactionBlock` [17]                  |
| Logos L1 | `POST /mantle/status`, mempool status only [19] |
| LEZ      | status projected from `getTransaction` [21]     |

**Why it exists.** [NOT FOUND]

Solana's `getSignatureStatuses` returns a `confirmationStatus` of processed,
confirmed, or finalized [44].

### 1.25 Wait for a chosen confirmation level

Blocks until a transaction reaches a caller-specified degree of certainty,
rather than making the caller poll.

| Chain    | Method                                                   |
| -------- | -------------------------------------------------------- |
| Ethereum | [NOT FOUND]; SDKs poll receipts [36]                     |
| Bitcoin  | [NOT FOUND] as a wait method [24]                        |
| Solana   | `signatureSubscribe`, push rather than blocking [45]     |
| XRPL     | [NOT FOUND]; poll `tx` or use `subscribe` [26]           |
| Cosmos   | `/broadcast_tx_commit` [9]                               |
| Stellar  | [NOT FOUND]; polling `getTransaction` is documented [27] |
| NEAR     | `wait_until` on `send_tx` and `tx` [34]                  |
| Sui      | request type on `sui_executeTransactionBlock` [17]       |
| Logos L1 | [NOT FOUND] [19]                                         |
| LEZ      | [NOT FOUND]; the module polls in a loop [21]             |

**Why it exists.** [NOT FOUND] as an explicit rationale document. NEAR states
the operational trade-off, noting the stricter milestones "can take several
seconds", which is why the level is a caller parameter [34].

This is one of the sharpest divergences in the survey. NEAR parameterises the
milestone on one method, offering `NONE`, `INCLUDED`, `EXECUTED_OPTIMISTIC`,
`INCLUDED_FINAL`, `EXECUTED`, and `FINAL` [34]. Cosmos splits it across three
broadcast endpoints. Solana turns it into a subscription. Ethereum, Bitcoin,
XRPL, and Stellar leave it to client-side polling.

### 1.26 Inspect the mempool or pending set

Lists transactions the node holds but has not yet included in a block.

| Chain    | Method                                                        |
| -------- | ------------------------------------------------------------- |
| Ethereum | `eth_newPendingTransactionFilter`; no direct dump [46]        |
| Bitcoin  | `getrawmempool`, `getmempoolentry`, `getmempoolinfo` [24]     |
| Solana   | [NOT FOUND]; no mempool, transactions forward to leaders [25] |
| XRPL     | [NOT FOUND] [26]                                              |
| Cosmos   | `/unconfirmed_txs`, `/num_unconfirmed_txs` [9]                |
| Stellar  | [NOT FOUND] [27]                                              |
| NEAR     | [NOT FOUND] [13]                                              |
| Sui      | [NOT FOUND] [17]                                              |
| Logos L1 | `GET /mempool/view` [19]                                      |
| LEZ      | [NOT FOUND] [21]                                              |

**Why it exists.** [NOT FOUND]

### 1.27 Get execution effects and state changes

Returns the concrete state deltas a transaction produced, rather than just a
success flag.

| Chain    | Method                                                         |
| -------- | -------------------------------------------------------------- |
| Ethereum | `eth_getTransactionReceipt`; effects inferred from logs [33]   |
| Bitcoin  | [NOT FOUND]; effects implicit in the UTXO set delta [24]       |
| Solana   | `getTransaction`, with pre and post balances [25]              |
| XRPL     | `tx` metadata describing how the ledger changed [47]           |
| Cosmos   | `GetBlockResults`, `GetLatestBlockResults` [10]                |
| Stellar  | `GET /effects` on Horizon, a first-class effects resource [48] |
| NEAR     | `changes`, `block_effects` [13]                                |
| Sui      | `sui_getTransactionBlock` effects [17]                         |
| Logos L1 | `GET /cryptarchia/blocks/:id/events`, block scoped [19]        |
| LEZ      | [NOT FOUND] as a queryable per-transaction effects view [21]   |

**Why it exists.** [NOT FOUND]

Stellar is the only surveyed chain with a dedicated effects resource rather than
effects inferred from logs or receipts.

### 1.28 Query contract events or logs

Retrieves structured events emitted by execution, filtered by address, topic, or
type.

| Chain    | Method                                                          |
| -------- | --------------------------------------------------------------- |
| Ethereum | `eth_getLogs` [46]                                              |
| Bitcoin  | no equivalent; no contract layer [24]                           |
| Solana   | `logsSubscribe` for push; `getTransaction` per transaction [45] |
| XRPL     | no equivalent; transaction metadata serves the purpose [26]     |
| Cosmos   | `GetTxsEvent`, `/tx_search` [29]                                |
| Stellar  | `getEvents` [27]                                                |
| NEAR     | [NOT FOUND] as an events method [13]                            |
| Sui      | `suix_queryEvents`, `sui_getEvents` [17]                        |
| Logos L1 | `GET /cryptarchia/blocks/:id/events` [19]                       |
| LEZ      | [NOT FOUND] [21]                                                |

**Why it exists.** [NOT FOUND]

### 1.29 Read historical state at a past version

Reads a ledger record as it stood at an earlier block, ledger, or version.

| Chain    | Method                                                    |
| -------- | --------------------------------------------------------- |
| Ethereum | block-tag parameter on state methods [30]                 |
| Bitcoin  | `gettxoutproof`, `verifytxoutproof` [24]                  |
| Solana   | `getBlock`, `getTransaction` bounded by retention [25]    |
| XRPL     | `ledger_entry`, `ledger_data` with `ledger_index` [26]    |
| Cosmos   | height-parameterised queries [10]                         |
| Stellar  | `getLedgerEntries` [27]                                   |
| NEAR     | `query` and `changes` with `block_id` [13]                |
| Sui      | `sui_tryGetPastObject`, `sui_tryMultiGetPastObjects` [17] |
| Logos L1 | `?tip=` parameter on the wallet balance route [19]        |
| LEZ      | indexer `getAccountAtBlock` [22]                          |

**Why it exists.** [NOT FOUND]

Sui is explicit about its own limits here: "there is no software-level
guarantee/SLA that objects with past versions can be retrieved by this API"
[17].

### 1.30 Paginate a result set

Walks a long result set in bounded pages.

| Chain    | Method                                                                    |
| -------- | ------------------------------------------------------------------------- |
| Ethereum | [NOT FOUND] as a cursor scheme; `eth_getLogs` bounds by block range [46]  |
| Bitcoin  | `listtransactions` with count and skip, offset style [24]                 |
| Solana   | `getSignaturesForAddress` with before, until, and limit [25]              |
| XRPL     | `account_tx` with a stable `marker` [47]                                  |
| Cosmos   | shared `PageRequest` and `PageResponse` messages [49]                     |
| Stellar  | `cursor`, `limit`, `order`, and `_links.next` [50]                        |
| NEAR     | [NOT FOUND] as a general cursor [13]                                      |
| Sui      | cursor paging on `suix_queryTransactionBlocks` and siblings [17]          |
| Logos L1 | [NOT FOUND] as a cursor scheme [19]                                       |
| LEZ      | `getBlocks` uses a cursor; `getTransactionsByAccount` uses an offset [22] |

**Why it exists.** [NOT FOUND]

Three idioms are visible: an opaque stable marker that "remains stable across
requests even if the server's available ledger range changes" (XRPL) [47], a
reusable protobuf pagination message shared by every module query (Cosmos) [49],
and HAL hypermedia links (Stellar Horizon) [50]. LEZ uses both a cursor and an
offset idiom in one interface.

### 1.31 Query historical transactions for an address

Returns the transaction history touching a given account.

| Chain    | Method                                                   |
| -------- | -------------------------------------------------------- |
| Ethereum | [NOT FOUND] on the standard node API [46]                |
| Bitcoin  | `listtransactions`, `listsinceblock`, wallet scoped [24] |
| Solana   | `getSignaturesForAddress` [25]                           |
| XRPL     | `account_tx` [47]                                        |
| Cosmos   | `GetTxsEvent`, `/tx_search` [29]                         |
| Stellar  | `getTransactions`, plus Horizon collections [27]         |
| NEAR     | [NOT FOUND] on the RPC [13]                              |
| Sui      | `suix_queryTransactionBlocks` [17]                       |
| Logos L1 | [NOT FOUND] [19]                                         |
| LEZ      | indexer `getTransactionsByAccount` [22]                  |

**Why it exists.** [NOT FOUND]

Ethereum has no address-history method on the standard node API, which is why
third-party indexers occupy that role.

### 1.32 Subscribe to new blocks and to events

Pushes new blocks, or matching events and account changes, to the client as they
occur.

| Chain    | Blocks                                          | Events or accounts                                           |
| -------- | ----------------------------------------------- | ------------------------------------------------------------ |
| Ethereum | `eth_subscribe("newHeads")` [51]                | `eth_subscribe("logs")` [51]                                 |
| Bitcoin  | `-zmqpubhashblock`, `-zmqpubrawblock` [52]      | `-zmqpubrawtx`, `-zmqpubsequence` [52]                       |
| Solana   | `slotSubscribe`, `blockSubscribe` [45]          | `accountSubscribe`, `programSubscribe`, `logsSubscribe` [45] |
| XRPL     | `subscribe` ledger stream [26]                  | `subscribe` transaction and account streams [26]             |
| Cosmos   | `/subscribe` [9]                                | `/subscribe` with an event query [9]                         |
| Stellar  | Horizon streaming mode [48]                     | Horizon streaming mode [48]                                  |
| NEAR     | [NOT FOUND] [13]                                | [NOT FOUND] [13]                                             |
| Sui      | `SubscriptionService.SubscribeCheckpoints` [15] | `SubscribeEvents`, `SubscribeTransactions` [15]              |
| Logos L1 | `GET /cryptarchia/events/blocks/stream` [19]    | [NOT FOUND] as a filtered event stream [19]                  |
| LEZ      | indexer `subscribeToFinalizedBlocks` [22]       | [NOT FOUND] [22]                                             |

**Why it exists.** [NOT FOUND]

### 1.33 Resume a stream from a known position

Restarts an interrupted stream from the last position the client processed,
without gaps.

| Chain    | Mechanism                                                         |
| -------- | ----------------------------------------------------------------- |
| Ethereum | [NOT FOUND] [51]                                                  |
| Bitcoin  | `-zmqpubsequence` allows loss detection, not replay [52]          |
| Solana   | [NOT FOUND] [45]                                                  |
| XRPL     | [NOT FOUND] for streams; `marker` covers historical paging [26]   |
| Cosmos   | [NOT FOUND] [9]                                                   |
| Stellar  | `cursor` on streaming mode [48]                                   |
| NEAR     | [NOT FOUND]; no streaming surface [13]                            |
| Sui      | [NOT FOUND] on the documented subscription methods [15]           |
| Logos L1 | [NOT FOUND]; the block stream accepts no start position [19]      |
| LEZ      | [NOT FOUND]; `subscribeToFinalizedBlocks` takes no arguments [22] |

**Why it exists.** [NOT FOUND]

This is the weakest-supported capability in the survey, and that is itself the
finding. Only Stellar documents true cursor-based resumption: "Horizon will
start at the earliest known effect unless a cursor is set, in which case it will
start from that cursor" [48]. Bitcoin offers loss detection without replay.
Every other surveyed chain, and both Logos targets, document no resume
mechanism.

### 1.34 Structured errors and a code taxonomy

Returns failures in a machine-readable shape, ideally with a defined code set so
clients can branch on failure type rather than parsing strings.

| Chain    | Envelope                            | Code taxonomy                                  |
| -------- | ----------------------------------- | ---------------------------------------------- |
| Ethereum | JSON-RPC 2.0 error object [43]      | JSON-RPC codes [43]                            |
| Bitcoin  | JSON-RPC error object [3]           | JSON-RPC codes [43]                            |
| Solana   | JSON-RPC 2.0 error object [43]      | JSON-RPC codes [43]                            |
| XRPL     | dedicated error format [53]         | named string codes [53]                        |
| Cosmos   | gRPC status [54]                    | gRPC canonical codes [54]                      |
| Stellar  | JSON-RPC 2.0 error object [43]      | JSON-RPC codes [43]                            |
| NEAR     | JSON-RPC 2.0 error object [43]      | JSON-RPC codes [43]                            |
| Sui      | JSON-RPC and gRPC status [43][54]   | JSON-RPC and gRPC codes [43][54]               |
| Logos L1 | HTTP status with an error body [19] | [NOT FOUND] as a defined taxonomy              |
| LEZ      | JSON-RPC error object [21]          | standard codes only, no application codes [55] |

**Why it exists.** EIP-1193 shows the wallet-layer extension of the idea,
defining provider codes on top of JSON-RPC: 4001 User Rejected Request, 4100
Unauthorized, 4200 Unsupported Method, 4900 Disconnected, and 4901 Chain
Disconnected [56].

Except for XRPL, which publishes a dedicated error-format page, the envelope
rows follow from each chain's transport choice rather than from a per-chain
documented error contract. XRPL is the one surveyed chain using named string
codes such as `amendmentBlocked`, `tooBusy`, and `unknownCmd` rather than
numeric JSON-RPC codes, despite serving JSON-RPC [53].

LEZ constructs errors with the standard JSON-RPC `InternalError` code and a
free-text message, with no application code space, category, or retryability
signal [55].

## 2. RPC and Transport Types

### 2.1 What each chain serves

| Chain    | JSON-RPC HTTP   | JSON-RPC WS     | REST               | gRPC              | GraphQL     | Push                   |
| -------- | --------------- | --------------- | ------------------ | ----------------- | ----------- | ---------------------- |
| Ethereum | yes [1]         | yes [1]         | no [1]             | [NOT FOUND]       | yes [57]    | WebSocket and IPC [51] |
| Bitcoin  | yes [3]         | no [3]          | yes, read only [4] | [NOT FOUND]       | [NOT FOUND] | ZeroMQ [52]            |
| Solana   | yes [5]         | yes [5]         | [NOT FOUND]        | [NOT FOUND]       | [NOT FOUND] | WebSocket [45]         |
| XRPL     | yes [6]         | yes [6]         | no [58]            | internal only [7] | [NOT FOUND] | WebSocket only [26]    |
| Cosmos   | yes [9]         | yes [9]         | yes [8]            | yes [8]           | [NOT FOUND] | WebSocket [9]          |
| Stellar  | yes [11]        | [NOT FOUND]     | Horizon [48]       | [NOT FOUND]       | [NOT FOUND] | SSE on Horizon [59]    |
| NEAR     | yes [13]        | [NOT FOUND]     | [NOT FOUND]        | [NOT FOUND]       | [NOT FOUND] | [NOT FOUND]            |
| Sui      | deprecated [16] | deprecated [16] | [NOT FOUND]        | yes [60]          | yes [18]    | gRPC streaming [15]    |
| Logos L1 | no [19]         | no [19]         | yes [19]           | no [19]           | no [19]     | chunked HTTP [19]      |
| LEZ      | yes [21]        | yes [21]        | no [21]            | no [21]           | no [21]     | WebSocket [22]         |

REST on Bitcoin is a separate unauthenticated read-only interface enabled with
`-rest`, carrying an explicit warning about browser access on the same host [4].
XRPL's gRPC surface exists for Clio to extract data from the node rather than as
a public client API [7].

Note that the Ethereum rows describe the execution layer only. The consensus
layer has its own REST API, which was not surveyed.

### 2.2 Characteristics

**JSON-RPC** is "a stateless, light-weight remote procedure call (RPC)
protocol", explicitly transport agnostic [43]. It supports batching and
notifications. It defines no schema and says nothing about streaming, so every
subscription mechanism in this survey is a per-chain extension layered on a
full-duplex transport rather than a JSON-RPC feature. OpenRPC fills the schema
gap, built on JSON Schema with discovery via `rpc.discover` [61].

**REST** is browser and cache friendly. Cosmos states its REST layer exists so
web applications can reach the same functionality "without requiring HTTP2" [8].
It has no native streaming, so Horizon bolts on SSE. It need not be hand
written: Cosmos generates it from protobuf via gRPC-gateway, and "for each gRPC
endpoint defined in a Protobuf Query service, the Cosmos SDK offers a REST
equivalent" [8].

**gRPC** uses protocol buffers "as both its Interface Definition Language (IDL)
and as its underlying message interchange format", with code generation across
Java, Go, Python, Ruby, C++, C#, Dart, Objective-C, PHP, and Rust [62].
Streaming is first class: server streaming means "the client sends a request to
the server and gets a stream to read a sequence of messages back" [63]. It is
the only transport here where push is part of the core protocol. Its cost is
browser reach, which is precisely why Cosmos maintains a REST facade.

**GraphQL** covers "a type system, query language and execution semantics,
static validation, and type introspection" [64]. Geth's own justification for
serving it is that it avoids "the extra load on the client for filling in fields
which are not needed" and allows "combining several traditional JSON-RPC
requests into one query" [57]. The specification defines `subscription` as an
operation type but binds it to no transport [64][65], and neither Geth's nor
Sui's GraphQL surface documents subscriptions. In both Sui and Stellar, the
rich-query surface sits on the indexer rather than the node.

**WebSocket** is full duplex, and Geth's constraint is the general rule:
"Subscriptions require a full duplex connection" [51]. It is the de facto
subscription transport across Ethereum, Solana, XRPL, and Cosmos.

**SSE** is unidirectional server to client, using `text/event-stream` [66]. Its
decisive advantage for ledger tailing is built-in resumption: the
`Last-Event-ID` header "reports an EventSource object's last event ID string to
the server when the user agent is to reestablish the connection" [66]. WebSocket
has no equivalent standard mechanism, so each chain must invent its own cursor.
This is the mechanical reason stream resumption is so poorly supported across
the survey.

### 2.3 Migrations

Sui is retiring an entire API generation on a published timetable. JSON-RPC is
"deprecated. Migrate to either gRPC or GraphQL RPC before the week of July 27,
2026, when JSON-RPC is disabled on Sui Foundation mainnet full nodes", with full
decommission in mid-October 2026 [16][18]. The split rationale is "gRPC for
real-time full node access, archival service access and transaction execution,
and GraphQL RPC for rich, structured queries" [16].

The sub-migration is the more instructive detail: WebSocket subscriptions "have
been deprecated since mid-2024", replaced by gRPC server streaming, where
"SubscribeEvents with an EventFilter replaces sui_subscribeEvent" [16]. A chain
moving push off WebSocket and onto gRPC streaming, with server-side filtering as
the stated gain.

Bitcoin Core shows the opposite, compatibility-preserving pattern. It serves
JSON-RPC 1.1 and 2.0 concurrently, selected per request: "A 2.0 request is
identified by the presence of "jsonrpc": "2.0" in the request body. If that key
\+ value is not present ... the legacy JSON-RPC v1.1 protocol is followed
instead" [3].

Stellar RPC and Horizon are commonly mistaken for a migration. They are not. The
RPC documentation lists as an explicit non-goal: "A drop-in replacement for
Horizon. Horizon provides several indexing features not commonly supported by
RPC nodes" [11]. It is a deliberate node and indexer split.

No transport migration was found for Ethereum, Cosmos, NEAR, Solana, or XRPL.

## 3. SDK Languages

### 3.1 Which languages each ecosystem publishes

First party means the repository sits under the project's own GitHub
organisation, or the official documentation labels it official. Where the two
tests disagree, the row says so.

| Chain    | TS/JS            | Rust                     | Go                         | Python           | Java/Kotlin      | Swift          | C/C++            | .NET           |
| -------- | ---------------- | ------------------------ | -------------------------- | ---------------- | ---------------- | -------------- | ---------------- | -------------- |
| Ethereum | community [67]   | community [68]           | first party [69]           | first party [70] | community [67]   | [NOT FOUND]    | [NOT FOUND]      | community [67] |
| Bitcoin  | community [71]   | community [71]           | community [71]             | community [71]   | community [71]   | community [72] | community [71]   | community [71] |
| Solana   | first party [73] | first party [74]         | community [73]             | community [73]   | community [73]   | [NOT FOUND]    | [NOT FOUND]      | [NOT FOUND]    |
| XRPL     | first party [75] | [NOT FOUND]              | official, outside org [75] | first party [75] | first party [75] | [NOT FOUND]    | first party [75] | [NOT FOUND]    |
| Cosmos   | first party [76] | [NOT FOUND]              | first party [77]           | [NOT FOUND]      | [NOT FOUND]      | [NOT FOUND]    | [NOT FOUND]      | [NOT FOUND]    |
| Stellar  | first party [78] | first party [78]         | first party [78]           | community [78]   | community [78]   | community [78] | [NOT FOUND]      | community [78] |
| NEAR     | first party [79] | first party [79]         | [NOT FOUND]                | community [79]   | [NOT FOUND]      | [NOT FOUND]    | [NOT FOUND]      | [NOT FOUND]    |
| Sui      | first party [80] | first party [80]         | community [80]             | community [80]   | community [80]   | community [80] | [NOT FOUND]      | [NOT FOUND]    |
| Logos L1 | [NOT FOUND]      | in-repo Rust client [19] | [NOT FOUND]                | [NOT FOUND]      | [NOT FOUND]      | [NOT FOUND]    | C ABI [81]       | [NOT FOUND]    |
| LEZ      | [NOT FOUND]      | in-repo Rust client [21] | [NOT FOUND]                | [NOT FOUND]      | [NOT FOUND]      | [NOT FOUND]    | wallet FFI [21]  | [NOT FOUND]    |

TypeScript and Rust appear in every external ecosystem surveyed. Go and Python
appear in seven of eight. At the thin end, C and C++ appear as a first-party SDK
only for XRPL, .NET is community maintained everywhere it appears, and no
surveyed chain publishes a first-party Swift SDK.

Bitcoin is the sharpest contrast: it has no foundation-published SDK at all. The
`bitcoin` GitHub organisation contains the node and the BIPs, and every client
library is community maintained [71].

Ethereum, Bitcoin, and Cosmos publish no documentation page labelling SDKs
official or community, so those rows rest on the organisation test alone. XRPL,
Sui, and Stellar state maintainership explicitly. XRPL's Go row is a genuine
conflict: the official documentation labels `Peersyst/xrpl-go` official, yet it
sits outside the XRPLF organisation [75].

### 3.2 Generating many SDKs from one core

BDK is the clearest worked example of the FFI pattern. The `bdk-ffi` repository
"creates a library ready for export to other languages using uniffi-rs for the
Rust-based bdk_wallet library", generating Kotlin and Android, Swift, Kotlin
JVM, Python, Dart, and React Native TypeScript packages from a single Rust core
[72]. Notably it comes from a community project rather than a foundation, and
the BDK core repository does not mention the bindings at all.

Schema-driven generation is the other route, and the survey shows both ends of
the spectrum. Cosmos generates from protobuf: each module "exposes a Protobuf
Query service that defines state queries", gRPC is chosen because it "has decent
client support in several languages", and a Swagger specification "is exposed
under the /swagger route on the API server" [8]. This is why Cosmos publishes so
few named per-language SDKs: the protobuf contract plus each language's own gRPC
toolchain substitutes for them. Sui similarly publishes protobuf interface
definitions usable to generate client libraries [60].

Stellar is explicitly the opposite. Its build-your-own-SDK guide describes
manual implementation, listing value conversions, host functions, SDK types, and
metadata as things the implementer writes [82].

### 3.3 Deprecations

Sui's JSON-RPC retirement cascades into its SDKs, which is why it ships a new
Rust SDK that "does not support JSON RPC" alongside a legacy one that does [80].

Ethereum's web3.js "was archived on March 4, 2025", with the documentation
advising users to "consider using alternative libraries like ethers.js or viem
for new projects" [67].

Solana renamed rather than deprecated: the 2.x line of `@solana/web3.js` became
`@solana/kit` [73], and the earlier v1 line, wallet adapter, and framework kit
packages "are superseded. New apps build directly on `@solana/kit`" [83].

## 4. Gap Summary

What Logos L1 and LEZ expose today against the functions catalogued above,
collected from the tables in section 1. This records observed state; it makes no
recommendation about what should be built.

### 4.1 Absent on both targets

| Function                                       | Chains with it |
| ---------------------------------------------- | -------------- |
| Simulate transaction execution                 | 6 of 8         |
| Estimate execution cost                        | 6 of 8         |
| Pre-submission acceptance check                | 6 of 8         |
| Wait for a chosen confirmation level           | 4 of 8         |
| Read-only contract call                        | 6 of 8         |
| Identify the network or chain                  | 8 of 8         |
| Verify a signature                             | 3 of 8         |
| Resume a stream from a known position          | 1 of 8         |
| Build an unsigned transaction as a node method | 3 of 8         |
| Encode and decode transaction bytes            | 2 of 8         |
| Submit a batch or package                      | 1 of 8         |

Network identification is the widest gap: all eight surveyed chains expose it
and neither Logos target does. Stream resumption is absent on both, but it is
also absent on seven of the eight surveyed chains.

### 4.2 Present but narrower than the surveyed norm

| Function                      | Logos state                                                               |
| ----------------------------- | ------------------------------------------------------------------------- |
| Get a balance                 | L1 scoped to node-custodial keys, not arbitrary subjects [19]             |
| Get transaction status        | L1 reports mempool status only, no lifecycle or finality [19]             |
| Get execution effects         | L1 block scoped, no per-transaction effects view [19]; LEZ absent [21]    |
| Paginate a result set         | LEZ mixes a cursor idiom and an offset idiom in one interface [22]        |
| Subscribe to new blocks       | Neither accepts a start position, so a dropped stream loses data [19][22] |
| Structured errors             | LEZ uses stock JSON-RPC codes with free text, no taxonomy [55]            |
| Query historical transactions | LEZ indexer only, offset paginated [22]                                   |
| Estimate the fee or fee rate  | L1 serves gas-price inputs, not an estimate for given bytes [19]          |

### 4.3 Present and comparable

L1 and LEZ both serve block reads, chain tip, transaction reads, and submission.
Both publish a machine-readable interface description at runtime, which Bitcoin
and Solana do not: L1 serves OpenAPI at `/api-docs/openapi.json` [20] and LEZ
serves `getSchema` [22]. L1 exposes a mempool view [19], which five of the eight
surveyed chains do not.

### 4.4 Existing FFI surface

L1 ships a C ABI generated by cbindgen [81]. The header exports ten symbols, of
which four are API operations: `get_cryptarchia_info`, `get_balance`,
`subscribe_to_new_blocks`, and `transfer_funds`. The remainder are node
lifecycle (`start_lb_node`, `stop_node`), memory management, and a status
helper. Against roughly forty HTTP routes on the node [19], the binding is
oriented around embedding and running a node rather than consuming the node API
as a client. `subscribe_to_new_blocks` is callback based and carries no resume
anchor [81].

## References

01. Go Ethereum, "JSON-RPC Server" documentation.
    https://geth.ethereum.org/docs/interacting-with-geth/rpc
02. Ethereum, "execution-apis" specification repository.
    https://github.com/ethereum/execution-apis
03. Bitcoin Core, "JSON-RPC Interface" documentation.
    https://raw.githubusercontent.com/bitcoin/bitcoin/master/doc/JSON-RPC-interface.md
04. Bitcoin Core, "Unauthenticated REST Interface" documentation.
    https://github.com/bitcoin/bitcoin/blob/master/doc/REST-interface.md
05. Solana, "RPC API" documentation. https://solana.com/docs/rpc
06. XRPL, "HTTP and WebSocket APIs" reference.
    https://xrpl.org/docs/references/http-websocket-apis
07. XRPL, "xrp_ledger.proto" gRPC service definition (rippled).
    https://raw.githubusercontent.com/XRPLF/rippled/develop/include/xrpl/proto/org/xrpl/rpc/v1/xrp_ledger.proto
08. Cosmos SDK, "gRPC, REST, and CometBFT Endpoints" documentation, v0.50.
    https://docs.cosmos.network/sdk/v0.50/learn/advanced/grpc_rest
09. CometBFT, "RPC OpenAPI specification".
    https://raw.githubusercontent.com/cometbft/cometbft/main/rpc/openapi/openapi.yaml
10. Cosmos SDK, "cosmos/base/tendermint/v1beta1/query.proto".
    https://raw.githubusercontent.com/cosmos/cosmos-sdk/main/proto/cosmos/base/tendermint/v1beta1/query.proto
11. Stellar, "Stellar RPC" documentation.
    https://developers.stellar.org/docs/data/apis/rpc
12. Stellar, "stellar-rpc.openrpc.json" specification document.
    https://raw.githubusercontent.com/stellar/stellar-docs/main/static/stellar-rpc.openrpc.json
13. NEAR, "RPC API" documentation. https://docs.near.org/api/rpc/introduction
14. NEAR, "nearcore JSON-RPC OpenAPI specification".
    https://raw.githubusercontent.com/near/nearcore/master/chain/jsonrpc/openapi/openapi.json
15. Sui, "Full Node Protocol" reference.
    https://docs.sui.io/references/fullnode-protocol
16. Sui, "JSON-RPC Migration" documentation.
    https://docs.sui.io/develop/accessing-data/json-rpc-migration
17. Sui, "Sui JSON-RPC OpenRPC specification", version 1.80.0.
    https://raw.githubusercontent.com/MystenLabs/sui/main/crates/sui-open-rpc/spec/openrpc.json
18. Sui, "GraphQL RPC" documentation. https://docs.sui.io/concepts/graphql-rpc
19. logos-blockchain, `nodes/api-common/src/paths.rs` and
    `nodes/node/binary/src/api/backend.rs`, commit `ecb2cc6`.
    https://github.com/logos-blockchain/logos-blockchain
20. logos-blockchain, `nodes/node/binary/src/api/openapi.rs` and
    `nodes/node/binary/src/api/backend.rs:215`, commit `ecb2cc6`.
    https://github.com/logos-blockchain/logos-blockchain
21. logos-execution-zone, `lez/sequencer/service/rpc/src/lib.rs`, commit
    `47eba25`. https://github.com/logos-blockchain/logos-execution-zone
22. logos-execution-zone, `lez/indexer/service/rpc/src/lib.rs`, commit
    `47eba25`. https://github.com/logos-blockchain/logos-execution-zone
23. Ethereum, "JSON-RPC API" documentation.
    https://ethereum.org/en/developers/docs/apis/json-rpc/
24. Bitcoin, "Original Bitcoin client RPC API reference".
    https://developer.bitcoin.org/reference/rpc/
25. Solana, "RPC HTTP Methods" documentation. https://solana.com/docs/rpc/http
26. XRPL, "Public API Methods" reference.
    https://xrpl.org/public-api-methods.html
27. Stellar, "Stellar RPC Methods" reference.
    https://developers.stellar.org/docs/data/apis/rpc/api-reference/methods
28. Ethereum, "execution-apis: src/eth/client.yaml".
    https://raw.githubusercontent.com/ethereum/execution-apis/main/src/eth/client.yaml
29. Cosmos SDK, "cosmos/tx/v1beta1/service.proto".
    https://raw.githubusercontent.com/cosmos/cosmos-sdk/main/proto/cosmos/tx/v1beta1/service.proto
30. Ethereum, "execution-apis: src/eth/state.yaml".
    https://raw.githubusercontent.com/ethereum/execution-apis/main/src/eth/state.yaml
31. Cosmos SDK, "cosmos/auth/v1beta1/query.proto".
    https://raw.githubusercontent.com/cosmos/cosmos-sdk/main/proto/cosmos/auth/v1beta1/query.proto
32. Cosmos SDK, "cosmos/bank/v1beta1/query.proto".
    https://raw.githubusercontent.com/cosmos/cosmos-sdk/main/proto/cosmos/bank/v1beta1/query.proto
33. Ethereum, "execution-apis: src/eth/transaction.yaml".
    https://raw.githubusercontent.com/ethereum/execution-apis/main/src/eth/transaction.yaml
34. NEAR, "RPC: Transactions" documentation.
    https://docs.near.org/api/rpc/transactions
35. Ethereum, "execution-apis: src/eth/execute.yaml".
    https://raw.githubusercontent.com/ethereum/execution-apis/main/src/eth/execute.yaml
36. Ethereum, "execution-apis: src/eth/submit.yaml".
    https://raw.githubusercontent.com/ethereum/execution-apis/main/src/eth/submit.yaml
37. Bitcoin, "BIP-174: Partially Signed Bitcoin Transaction Format".
    https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki
38. XRPL, "simulate" transaction method.
    https://xrpl.org/docs/references/http-websocket-apis/public-api-methods/transaction-methods/simulate
39. Stellar, "simulateTransaction" method reference.
    https://developers.stellar.org/docs/data/apis/rpc/api-reference/methods/simulateTransaction
40. Ethereum, "execution-apis: src/eth/fee_market.yaml".
    https://raw.githubusercontent.com/ethereum/execution-apis/main/src/eth/fee_market.yaml
41. Ethereum, "EIP-1559: Fee market change for ETH 1.0 chain".
    https://eips.ethereum.org/EIPS/eip-1559
42. Solana, "sendTransaction" RPC method.
    https://solana.com/docs/rpc/http/sendtransaction
43. JSON-RPC Working Group, "JSON-RPC 2.0 Specification".
    https://www.jsonrpc.org/specification
44. Solana, "getSignatureStatuses" RPC method.
    https://solana.com/docs/rpc/http/getsignaturestatuses
45. Solana, "RPC WebSocket Methods" documentation.
    https://solana.com/docs/rpc/websocket
46. Ethereum, "execution-apis: src/eth/filter.yaml".
    https://raw.githubusercontent.com/ethereum/execution-apis/main/src/eth/filter.yaml
47. XRPL, "account_tx" account method.
    https://xrpl.org/docs/references/http-websocket-apis/public-api-methods/account-methods/account_tx
48. Stellar, "List All Effects" Horizon reference.
    https://developers.stellar.org/docs/data/apis/horizon/api-reference/list-all-effects
49. Cosmos SDK, "cosmos/base/query/v1beta1/pagination.proto".
    https://raw.githubusercontent.com/cosmos/cosmos-sdk/main/proto/cosmos/base/query/v1beta1/pagination.proto
50. Stellar, "Horizon Pagination" documentation.
    https://developers.stellar.org/docs/data/apis/horizon/api-reference/structure/pagination
51. Go Ethereum, "Publish and Subscribe" documentation.
    https://geth.ethereum.org/docs/interacting-with-geth/rpc/pubsub
52. Bitcoin Core, "ZeroMQ" documentation.
    https://github.com/bitcoin/bitcoin/blob/master/doc/zmq.md
53. XRPL, "Error Formatting" API convention.
    https://xrpl.org/docs/references/http-websocket-apis/api-conventions/error-formatting
54. gRPC, "Status codes and their use in gRPC".
    https://grpc.io/docs/guides/status-codes/
55. logos-execution-zone, `lez/indexer/service/src/service.rs:343-349`, commit
    `47eba25`. https://github.com/logos-blockchain/logos-execution-zone
56. Ethereum, "EIP-1193: Ethereum Provider JavaScript API".
    https://eips.ethereum.org/EIPS/eip-1193
57. Go Ethereum, "GraphQL Server" documentation.
    https://geth.ethereum.org/docs/interacting-with-geth/rpc/graphql
58. XRPL, "Request Formatting" API convention.
    https://xrpl.org/docs/references/http-websocket-apis/api-conventions/request-formatting
59. Stellar, "Horizon Streaming" documentation.
    https://developers.stellar.org/docs/data/apis/horizon/api-reference/structure/streaming
60. Sui, "sui-apis" protobuf interface definitions.
    https://github.com/MystenLabs/sui-apis
61. OpenRPC, project homepage. https://open-rpc.org/
62. gRPC, "Introduction to gRPC".
    https://grpc.io/docs/what-is-grpc/introduction/
63. gRPC, "Core concepts, architecture and lifecycle".
    https://grpc.io/docs/what-is-grpc/core-concepts/
64. GraphQL Foundation, "GraphQL specification" repository.
    https://raw.githubusercontent.com/graphql/graphql-spec/main/README.md
65. GraphQL Foundation, "GraphQL over HTTP" specification repository.
    https://github.com/graphql/graphql-over-http
66. WHATWG, "Server-sent events", HTML Living Standard.
    https://html.spec.whatwg.org/multipage/server-sent-events.html
67. Ethereum, "JavaScript API libraries" documentation.
    https://ethereum.org/en/developers/docs/apis/javascript/
68. Ethereum, "Rust developer resources".
    https://ethereum.org/en/developers/docs/programming-languages/rust/
69. Ethereum, "go-ethereum" repository. https://github.com/ethereum/go-ethereum
70. Ethereum, "web3.py" repository. https://github.com/ethereum/web3.py
71. Bitcoin, "Bitcoin Development" resources page.
    https://bitcoin.org/en/development
72. Bitcoin Dev Kit, "bdk-ffi" repository.
    https://github.com/bitcoindevkit/bdk-ffi
73. Solana, "Solana Clients" documentation. https://solana.com/docs/clients
74. Solana, "Rust Client" documentation. https://solana.com/docs/clients/rust
75. XRPL, "Client Libraries" reference.
    https://xrpl.org/docs/references/client-libraries
76. Cosmos, "CosmJS" repository. https://github.com/cosmos/cosmjs
77. Cosmos, "cosmos-sdk" repository. https://github.com/cosmos/cosmos-sdk
78. Stellar, "Client SDKs" documentation.
    https://developers.stellar.org/docs/tools/sdks/client-sdks
79. NEAR, "NEAR API" documentation. https://docs.near.org/tools/near-api
80. Sui, "Sui SDKs" reference. https://docs.sui.io/references/sui-sdks
81. logos-blockchain, `c-bindings/logos_blockchain.h` and
    `c-bindings/cbindgen.toml`, commit `ecb2cc6`.
    https://github.com/logos-blockchain/logos-blockchain
82. Stellar, "Build Your Own SDK" documentation.
    https://developers.stellar.org/docs/tools/sdks/build-your-own
83. Solana, "web3.js Compatibility" documentation.
    https://solana.com/docs/frontend/web3-compat
