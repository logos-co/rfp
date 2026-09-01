# Appendix: Blockchain API and SDK Ecosystem

This appendix surveys what blockchain node APIs and client SDKs expose across
eight established ecosystems: which functions they offer, which transports they
serve them over, and which SDK languages they publish. It provides context for
the blockchain API and SDK RFPs, covering the API surface itself, the JSON-RPC
proxy and TypeScript SDK that consume it, and the further interface modules and
external SDKs built on top. Logos L1 and LEZ appear as the final rows of each
table, recording what they expose today.

Every claim carries a first-party source that was fetched and confirmed to
resolve. Logos rows cite files and line numbers in `logos-blockchain` at commit
`ecb2cc6` and `logos-execution-zone` at commit `47eba25`.

### On `[NOT FOUND]`

A survey of this kind fails in two directions: it can assert something that is
not there, or it can quietly omit something it did not look hard enough for.
Both markers below exist to keep the second failure visible rather than silent.

`[NOT FOUND]` in a table cell means no method, field, or route for that function
was found in the chain's official documentation. It is a statement about the
search, not about the chain. Three things could produce it: the capability
genuinely does not exist; it exists under a name the survey did not recognise;
or it exists somewhere the survey did not read, such as a consensus-layer API or
a non-canonical extension. Where the distinction was determinable the cell says
so, for example "no equivalent" where a chain's architecture excludes the
function outright, as with contract calls on Bitcoin.

"No specific or relevant context has been found" under **Why it exists** means
no changelog, standards-document motivation section, architecture decision
record, or maintainer discussion explaining the capability's origin was located.
This is the common case: twenty-three of the thirty-four functions carry it.
Most API surfaces are simply not documented as to motivation, and absence of a
recorded reason is not evidence that no reason existed.

Neither marker should be read as a gap in the chain. Read them as the boundary
of what was verified.

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

Documented rationale was found for eleven of the thirty-four functions, drawn
from standards documents with explicit motivation sections (BIP-174, BIP-331,
ERC-191, EIP-234, EIP-695, EIP-1559, EIP-1767, EIP-1898, EIP-4444, NEP-413),
architecture decision records, and maintainer discussions. The other
twenty-three record that no specific or relevant context has been found. API
surfaces are, on the evidence, largely undocumented as to motivation.

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

**Why it exists.** No specific or relevant context has been found.

**What comes back.** The request is empty, so the response shape is the whole
design. Fields are grouped below by what they tell the caller, since chains name
the same concepts differently.

| What the caller learns  | Where it appears                                                                                                           |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| Software version        | Cosmos `version`, `git_commit`, `build_tags`, `go_version` [10]; Logos L1 a bare string of package version and commit [19] |
| Framework version       | Cosmos `cosmos_sdk_version` [10]                                                                                           |
| Per-dependency versions | Cosmos `build_deps`, a repeated module path, version, and checksum [10]                                                    |
| Service identity        | Cosmos `name`, `app_name` [10]                                                                                             |

The spread is the finding. Cosmos returns a dependency manifest, so a client can
detect capability per module rather than per node build [10]. Logos L1 returns a
bare JSON string, which leaves no field structure to carry a network identifier,
feature flags, or retention bounds even if it wanted to [19].

Cosmos also annotates the schema itself with the release each field arrived in,
for example `(cosmos_proto.field_added_in) = "cosmos-sdk 0.43"` on
`cosmos_sdk_version` [10]. That makes version-conditional capability negotiation
a property of the contract rather than of documentation.

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

**Why it exists.** No specific or relevant context has been found.

**What comes back.** This is where retention bounds live on most chains, not on
the version call. Grouped by what the caller learns:

| What the caller learns  | Where it appears                                                                        |
| ----------------------- | --------------------------------------------------------------------------------------- |
| Is the node caught up   | Cosmos `syncing` [10]; Bitcoin `initialblockdownload` [105]; LEZ `state` [115]          |
| How far caught up       | Bitcoin `verificationprogress`, normalised 0 to 1 [105]                                 |
| Current tip             | Cosmos `latest_block_height` [10]; Bitcoin `blocks` [105]; LEZ `indexed_block_id` [115] |
| Headers ahead of blocks | Bitcoin `headers` [105]                                                                 |
| Retention floor         | Cosmos `earliest_block_height` [10]; Bitcoin `pruneheight` [105]                        |
| Retention as a range    | XRPL `complete_ledgers`, possibly disjoint [101]                                        |
| Retention policy        | Bitcoin `pruned`, `automatic_pruning`, `prune_target_size` [105]                        |
| Finality reached        | XRPL `validated_ledger` present, else `closed_ledger` [101]                             |
| Cannot safely serve     | XRPL `amendment_blocked` [101]                                                          |
| Why it stalled          | LEZ `stall_reason`, `last_error` [115]                                                  |

Three designs stand out. Bitcoin separates the validated chain from known
headers, so a caller can tell a syncing node from a stalled one, and splits
retention into four fields distinguishing whether pruning is on, where the floor
is, and whether it is automatic [105]. XRPL expresses retention as a range
expression rather than a floor, documented as possibly "a disjoint sequence such
as `24900901-24900984,24901116-24901158`", and signals finality by which field
is present rather than by a status enum [101].

LEZ discloses indexer staleness, which most surveyed chains do not: its status
is "the ingestion state plus the indexed L2 tip and, when stalled, the stall
diagnostics" [115], so a client can tell how far behind an indexer-backed read
is, and why.

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
| LEZ      | `getChannelId` [21]                |

**Why it exists.** EIP-695 records the problem that a network identifier alone
did not solve: "Currently although we can use `net_version` RPC call to get the
current network ID, there's no RPC for querying the chain ID. This makes it
impossible to determine the current actual blockchain using the RPC" [94]. The
concrete failure is silent misconnection across a fork: "An ETH/ETC client can
accidentally connect to an ETC/ETH RPC endpoint without knowing it unless it
tries to sign a transaction or it fetches a transaction that is known to have
signed with a chain ID" [94]. The chain identifier and the replay protection are
the same value; EIP-155 introduced it into signing so that a transaction valid
on one chain is invalid on its fork [86].

LEZ reports the zone's own channel identity, which distinguishes one zone from
another. Logos L1 reports no chain identifier, so a client cannot verify from
the API which network an L1 endpoint serves.

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
| Sui      | [NOT FOUND] as a method; OpenRPC document published [17]                    |
| Logos L1 | `GET /api-docs/openapi.json`, Swagger UI at `/swagger-ui` [20]              |
| LEZ      | `getSchema` on the indexer [22]                                             |

**Why it exists.** XRPL states the purpose of serving its codec schema from the
running node: `server_definitions` "returns an SDK-compatible
`definitions.json`, generated from the `xrpld` instance currently running. You
can use this to query a node in a network, quickly receiving the definitions
necessary to serialize/deserialize its binary data" [102]. The schema is not
static: the response was itself extended in xrpld 3.2.0 to add transaction and
ledger-entry format sections [102], which is the drift an SDK shipping a copied
schema has to track.

L1 serves a full OpenAPI interface description at runtime, which Bitcoin and
Solana do not. The LEZ `getSchema` method returns a JSON Schema for the block
type rather than an interface description, so a client cannot be generated from
it [22].

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

**Why it exists.** No specific or relevant context has been found.

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

**Why it exists.** No specific or relevant context has been found.

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

**Why it exists.** No specific or relevant context has been found.

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

**Why it exists.** No specific or relevant context has been found.

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

**Why it exists.** No specific or relevant context has been found.

**A response field name as a security property.** XRPL renamed a field in this
response because the old name implied a guarantee it did not provide. Its
documentation states the consequence directly: "If a financial institution's
integration with the XRP Ledger assumes that the `Amount` field of a Payment is
always the full amount delivered, malicious actors may be able to exploit that
assumption to steal money from the institution" [103]. Under partial payments
the delivered amount can be less than `Amount`, and the correct field to read is
`delivered_amount` in the transaction metadata [103]. The field was renamed to
`DeliverMax` "to make the field name more specific to its behavior and help
prevent the misunderstandings and exploit described below" [103]. The change
shipped behind an API version rather than in place: API v1 continues to render
`Amount`, API v2 renders `DeliverMax` [103].

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

**Why it exists.** No specific or relevant context has been found.

**On fixed response shapes.** EIP-1767 records the cost of a server that cannot
know which fields a caller wants. Fetching every receipt in a block leads node
implementations to "end up fetching and deserializing the same data repeatedly,
leading to `O(n^2)` effort to fetch all transaction receipts from a block
instead of `O(n)`" [98]. The same document gives a concrete overfetch example:
`totalDifficulty` is stored separately from the block header, "and many callers
do not require this field. However, every call to `eth_getBlock` still retrieves
this field, requiring a separate disk read, because the RPC server has no way of
knowing if the user requires this field or not" [98].

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

**Why it exists.** No specific or relevant context has been found.

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

**Why it exists.** The parameter is a replay protector, and fetching it is a
bottleneck for anyone sending concurrently. Cosmos ADR-070 names the affected
users: sequence values "prevent replay attacks and ensure transactions from the
same sender are included in blocks and executed in sequential order.
Unfortunately, this makes it difficult to reliably send many concurrent
transactions from the same sender. Victims of such limitations include IBC
relayers and crypto exchanges" [91].

Solana documents the opposite failure, where the parameter expires before the
transaction can be signed. Blockhashes are valid for roughly 150 blocks, and "a
side-effect of using recent blockhashes is the forced mortality of a transaction
even before its submission" [90]. Durable nonces exist to replace the expiring
parameter with a stored one, which the same guide ties to multi-signature
collection rounds where "one party signs a transaction, and others may confirm
at a later time" [90].

**What comes back.** Solana's `getLatestBlockhash` returns the blockhash paired
with `lastValidBlockHeight`, so the caller learns the expiry deadline together
with the parameter rather than having to compute it [25].

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

The reason construction stays client-side is a trust boundary. Aptos is the
instructive case because it ships a server-side encoder anyway, as an escape
hatch for languages without a canonical serialisation library, and warns against
it in its own node source: callers "may take advantage of
/transactions/encode_submission. When using this endpoint, make sure you trust
the node you're talking to, as it is possible they could manipulate your
request" [108]. The same comment tells SDK users with native serialisation
support that they "do not need to use this endpoint" [108]. The endpoint is
enabled by default, so the protection is documentation rather than configuration
[108].

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

**Why it exists.** No specific or relevant context has been found.

### 1.15 Simulate transaction execution

Runs a full transaction against current state without committing it, returning
the outcome it would have had.

| Chain    | Method                                                      |
| -------- | ----------------------------------------------------------- |
| Ethereum | `eth_simulateV1`, `eth_call` [35]                           |
| Bitcoin  | [NOT FOUND]; `testmempoolaccept` tests acceptance only [24] |
| Solana   | `simulateTransaction` [25]                                  |
| XRPL     | `simulate` [38]                                             |
| Cosmos   | `Simulate` [29]                                             |
| Stellar  | `simulateTransaction` [39]                                  |
| NEAR     | [NOT FOUND] [13]                                            |
| Sui      | `sui_dryRunTransactionBlock` [17]                           |
| Logos L1 | [NOT FOUND] [19]                                            |
| LEZ      | [NOT FOUND] [21]                                            |

**Why it exists.** Stellar documents the motivation directly: the endpoint
"calculates the effective transaction data, required authorizations, and minimal
resource fee", and "provides a way to test and analyze the potential outcomes of
a transaction without actually submitting it to the network" [39]. Returning the
transaction data the caller then submits makes simulation a construction step
rather than an optional check.

**What comes back.** Grouped by what the caller learns, using Stellar's response
as the fullest example:

| What the caller learns     | Where it appears                                                                                     |
| -------------------------- | ---------------------------------------------------------------------------------------------------- |
| Did it succeed             | Stellar `results` [39]; Sui execution effects [17]                                                   |
| What it would cost         | Stellar `cost`, `minResourceFee` [39]; Solana compute units consumed [25]; Sui gas cost summary [17] |
| What state would change    | Stellar `stateChanges`, `events` [39]                                                                |
| Data to submit with        | Stellar `transactionData` [39]                                                                       |
| Prerequisite step required | Stellar `restorePreamble` [39]                                                                       |
| Basis of the answer        | Stellar `latestLedger` [39]                                                                          |

Two of those fields make simulation a construction step rather than a check.
`transactionData` is "The recommended Soroban Transaction Data to use when
submitting the simulated transaction" and `minResourceFee` is the "Recommended
minimum resource fee to add when submitting the transaction" \[39\]: the caller
copies both back into the transaction before submitting. `restorePreamble`
signals that archived ledger entries must be restored first, so simulation can
return a prerequisite rather than a simple pass or fail.

Neither Logos target exposes simulation. Six of the eight surveyed chains
execute the transaction and return its outcome; Bitcoin checks acceptance
without executing, and NEAR exposes no equivalent.

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

**Why it exists.** No specific or relevant context has been found.

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
| LEZ      | [NOT FOUND] as a fee-estimation method [21]                       |

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

Two projects have documented moving away from it. Go Ethereum removed the
`personal` namespace after deprecating it for roughly twenty months behind an
opt-in flag, and the removal note gives a mechanical consequence that is easy to
overlook: "With the removal of `personal`, as far as I know we have no more API
methods which contain credentials, and if we want to implement
logging-capabilities of RPC ingress payload, it would be possible after this"
[109]. Node-held keys mean a node cannot safely log its own request payloads.
Bitcoin Core has kept its wallet RPCs but is separating the wallet into its own
process, describing the monolithic structure as carrying "increased security
risks due to the tight integration of components" [110].

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

**Why it exists.** Message signing is separated from transaction signing because
an unconstrained signed blob can itself be a valid transaction. ERC-191
introduced a prefix byte for exactly this: "The initial `0x19` byte is intended
to ensure that the `signed_data` is not valid RLP", with the consequence that
"any EIP-191 `signed_data` can never be an Ethereum transaction" [87]. NEAR
reached the same conclusion independently in NEP-413, which states the hazard
directly: "An attacker could make the user inadvertently sign a valid
transaction which, once signed, could be submitted into the network to execute
it" [92]. Both solve it with a prefix tag that makes the payload invalid under
the chain's transaction encoding, and NEP-413 adds a nonce because "including a
nonce helps to mitigate replay attacks, in which an attacker can delay or
re-send a signed message" [92].

EIP-712 records the second problem, blind signing: "Currently signed messages
are an opaque hex string displayed to the user with little context about the
items that make up the message" [88].

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

**Why it exists.** No specific or relevant context has been found.

### 1.21 Broadcast a signed transaction

Hands a signed transaction to the node for gossip and inclusion, returning an
identifier to track it by.

| Chain    | Method                                                              |
| -------- | ------------------------------------------------------------------- |
| Ethereum | `eth_sendRawTransaction` [36]                                       |
| Bitcoin  | `sendrawtransaction` [24]                                           |
| Solana   | `sendTransaction` [42]                                              |
| XRPL     | `submit`, `submit_multisigned` [26]                                 |
| Cosmos   | `BroadcastTx` [29], `/broadcast_tx_sync`, `/broadcast_tx_async` [9] |
| Stellar  | `sendTransaction` [27]                                              |
| NEAR     | `send_tx`, `broadcast_tx_async` [34]                                |
| Sui      | `sui_executeTransactionBlock` [17]                                  |
| Logos L1 | `POST /mempool/add/tx` [19]                                         |
| LEZ      | `sendTransaction` [21]                                              |

**Why it exists.** No specific or relevant context has been found.

Solana's documentation warns that "a successful response doesn't guarantee the
transaction will be processed or confirmed" [42].

**What comes back.** Responses range from a bare identifier to a full result:

| What the caller learns | Where it appears                                                                 |
| ---------------------- | -------------------------------------------------------------------------------- |
| Transaction identifier | Cosmos `txhash` [93]; Solana the first signature [42]                            |
| Accepted or rejected   | Cosmos `code`, `codespace` [93]; XRPL `engine_result`, `engine_result_code` [26] |
| Human-readable reason  | Cosmos `raw_log`, `info` [93]; XRPL `engine_result_message` [26]                 |
| Where it landed        | Cosmos `height`, `timestamp` [93]                                                |
| Resource usage         | Cosmos `gas_wanted`, `gas_used` [93]                                             |
| Effects                | Cosmos `events`, `logs` [93]                                                     |
| Echo of the submission | Cosmos `tx` [93]; XRPL `tx_blob`, `tx_json` [26]                                 |

Two cautions are documented on the responses themselves. The Cosmos proto
comments that `raw_log` is "The output of the application's logger (raw string).
May be non-deterministic" [93], so it is not safe to parse. XRPL's engine result
is a provisional outcome rather than a ledger result [26], and Solana warns that
"a successful response doesn't guarantee the transaction will be processed or
confirmed" [42].

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

**Why it exists.** No specific or relevant context has been found.

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

**Why it exists.** No specific or relevant context has been found.

**Why it exists** (where it does). Bitcoin's package relay records the failure
that per-transaction evaluation creates: "Only individually considering
transactions for submission to the mempool creates a limitation in the node's
ability to determine which transactions to include in the mempool, since it
cannot take into account descendants until all the transactions are in the
mempool", and "This limitation harms users' ability to fee-bump their
transactions" [89]. A low-fee parent is rejected before the child that would pay
for it can be seen.

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

**Why it exists.** No specific or relevant context has been found.

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

**Why it exists.** No specific or relevant context has been found. as an
explicit rationale document. NEAR states the operational trade-off, noting the
stricter milestones "can take several seconds", which is why the level is a
caller parameter [34].

This is one of the sharpest divergences in the survey. NEAR parameterises the
milestone on one method, offering `NONE`, `INCLUDED`, `EXECUTED_OPTIMISTIC`,
`INCLUDED_FINAL`, `EXECUTED`, and `FINAL` [34]. Cosmos splits it across three
broadcast endpoints. Solana turns it into a subscription. Ethereum, Bitcoin,
XRPL, and Stellar leave it to client-side polling.

NEAR's six levels replaced a binary choice. The originating issue states the
problem as "Currently there is only option to either broadcast tx or broadcast
and wait for finality. There should be a way to configure what user wants to
wait when broadcasting tx" [112]. The docs now describe the older pair as kept
"for backward compatibility but offer less control over when the call returns"
[34]. Note also that NEAR distinguishes refund receipts from non-refund receipts
across its levels, because refunds settle later and most callers do not need to
wait for them [34].

Ethereum's block tags were argued in public rather than designed in one pass. A
proposed `unsafe` tag was dropped after Danny Ryan objected that if a safe-head
algorithm often returns the head, then ""safe" and "unsafe" would often return
the same thing. This is semantically unsound and confusion" [111]. The surviving
`safe` tag is defined in the schema as "The most recent block that is safe from
re-orgs under honest majority and certain synchronicity assumptions" [11].

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

**Why it exists.** No specific or relevant context has been found.

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

**Why it exists.** No specific or relevant context has been found.

Stellar is the only surveyed chain with a dedicated effects resource rather than
effects inferred from logs or receipts.

**What comes back, and how it grew.** Ethereum's receipt is a record of several
upgrades accumulating in one response type, with fields conditionally present by
era:

| What the caller learns | Field                     | Present when                          |
| ---------------------- | ------------------------- | ------------------------------------- |
| Did it succeed         | `root` [100]              | before the Byzantium upgrade          |
| Did it succeed         | `status` [100]            | after the Byzantium upgrade           |
| What it cost           | `effectiveGasPrice` [100] | defined against the EIP-1559 base fee |
| What it cost           | `blobGasUsed` [100]       | blob transactions only, per EIP-4844  |
| Cumulative block cost  | `cumulativeGasUsed` [100] | always                                |

Both success indicators are retained, because an effects response must describe
transactions that predate its own current format. `root` is "The
post-transaction state root. Only specified for transactions included before the
Byzantium upgrade" while `status` is "Either 1 (success) or 0 (failure). Only
specified for transactions included after the Byzantium upgrade" [100].

Cosmos records the cost of untyped effects, describing events implemented "as
`map[string]string`" which "makes these events difficult to consume as it
requires a great deal of raw string matching and parsing" [106].

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

**Why it exists.** EIP-234 documents why a log query needs to name its own
position. A subscription-based consumer cannot reliably track removals: "A
client (dApp) who needs reliable notification of both log additions (on new
blocks) and log removals (on chain reorgs) cannot achieve this while relying
solely on subscriptions and filters. This is because a combination of a network
or remote node failure during a reorg can result in the client getting out of
sync with reality" [96]. The sharper point concerns empty results: if the
response is an empty array, "the client is in a situation where they don't know
what block the results are for", and "there is no decision the client can make
that allows them a guarantee of recovery" [96]. A response must carry enough
context to identify the position it describes.

Unbounded log queries are also where per-provider divergence appears. A Geth
maintainer issue proposing default block-range limits states the reason plainly:
"Unlike Erigon Geth can not comply with get log requests with broad block ranges
which would only result on timeouts and CPU abuse" [46]. The same query can
therefore succeed against one client and fail against another.

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

**Why it exists.** EIP-1898 gives the clearest statement, and it is about read
coherence rather than archival curiosity. Without a way to pin the block, "a
wallet which just executed a transfer may want to display the balances of both
the sender and recipient. If there is a re-org in between when the balance of
the sender is queried via `eth_getBalance` and when the balance of the recipient
is queried, the balances may not reconcile" [95]. The EIP explicitly rejects
solving this with a stateful subscription, on the grounds that it "requires a
persistent connection between the client and node" and "increases coupling
between the client and the node" [95].

The counterpart question, why old state stops being readable, is answered by
EIP-4444: historical data "is not necessary for validating new blocks", and the
proposal deliberately forces the issue rather than leaving it to chance, "to
force clients to seek historical data from other sources, instead of relying on
the optional behavior of some clients which would result in quality degradation"
[97]. The design lesson for an API is that retention should be declared, not
discovered through a failed request.

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

**Why it exists.** No specific or relevant context has been found.

**A cautionary case.** Solana's `getProgramAccounts` used to degrade silently
when given a bad filter. A changelog entry records the fix and the prior
behaviour: the endpoint "now returns JSON-RPC errors when malformed filters are
provided (previously these malformed filters would be silently ignored and the
RPC call would execute an unfiltered query)" [104]. The caller received a wrong
answer and the node did the most expensive possible work. An invalid query
should be rejected loudly rather than falling back to a permissive default.

Three idioms are visible: an opaque marker whose "value is stable even if there
is a change in the server's range of available ledgers" (XRPL) [47], a reusable
protobuf pagination message shared by every module query (Cosmos) [49], and HAL
hypermedia links (Stellar Horizon) [50]. LEZ uses both a cursor and an offset
idiom in one interface.

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

**Why it exists.** No specific or relevant context has been found.

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

**Why it exists.** No specific or relevant context has been found.

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
| Logos L1 | `slot_from` on `/cryptarchia/blocks_range` [84]                   |
| LEZ      | [NOT FOUND]; `subscribeToFinalizedBlocks` takes no arguments [22] |

**Why it exists.** No specific or relevant context has been found.

This is the weakest-supported capability in the survey, and that is itself the
finding. Among the surveyed chains only Stellar documents cursor-based
resumption on a live stream: "Horizon will start at the earliest known effect
unless a cursor is set, in which case it will start from that cursor" [48].
Bitcoin offers loss detection without replay. The other six document no resume
mechanism.

There is a mechanical reason the majority lack it. Server-sent events carry
resumption in the transport: the `Last-Event-ID` header "reports an EventSource
object's last event ID string to the server when the user agent is to
reestablish the connection" [66]. WebSocket has no equivalent standard
mechanism, so every chain that streams over WebSocket must invent its own
cursor, and most have not. Stellar's resumable stream is served over SSE from
Horizon [48][59].

Logos L1 is an exception worth noting: `/cryptarchia/blocks_range` accepts
`slot_from` and `slot_to` bounds with a server batch size, so a consumer can
restart from a chosen slot [84]. Its unbounded live stream
(`/cryptarchia/events/blocks/stream`) accepts no position, so resumption means
switching to the range route rather than resuming the stream itself.

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

**Why it exists.** EIP-1474 frames the error taxonomy as a response to client
divergence rather than to error semantics: nodes "expose RPC endpoints with
differing method signatures; this forces applications to work around method
inconsistencies to maintain compatibility with various Ethereum RPC
implementations" [99]. EIP-1898 supplies the design rule that the taxonomy
itself does not state: two different failure causes must not share a code,
because the caller's recovery differs, so a block-not-found error "should be
different than the error code for the block not found case so that the caller
can distinguish the cases" [95].

Where an error is reported matters as much as its code. Solana moved signature
verification failure out of the transport error channel and into the result
body, so that it "will now be attached to the simulation result's `err` property
as `TransactionError::SignatureFailure` instead of being thrown as a JSON RPC
API error (-32003)" [104]. Transport failures and execution outcomes are
different things, and moving one to the other is a breaking change.

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

Sui's own justification for the split is published in a blog post rather than
the documentation: "as applications on Sui have evolved to serve a wide variety
of use cases, a one-size-fits-all query interface creates real bottlenecks.
Different use cases need different things" [107]. The same post gives the
retention argument for a separate archival service: "Full nodes prune data to
stay performant. That's fine for most operations, but it makes deep historical
queries difficult or impossible without running a dedicated full node yourself"
[107]. The migration guide concedes the cost of replacing one interface with
two: "Most exchanges and indexers end up using both: gRPC for the live data path
and transaction submission, GraphQL for ad hoc queries" [16].

Two retirement mechanics are worth recording. Algorand distinguishes a removed
endpoint from a nonexistent one, routing previously valid v1 paths to "410 Gone"
with a body pointing at v2 while leaving invalid v1 paths at 404 [113]. Solana
removed fifteen named endpoints in one major version, with the stated reasoning
that the server "is carrying around a lot of obsolete and deprecated endpoints.
v2.0 means it is time to remove them" [114]. No project in the survey has
published a retrospective on what such a removal cost its integrators.

No transport migration was found for Ethereum, Cosmos, NEAR, Solana, or XRPL.

## 3. SDK Languages

### 3.1 Which languages each ecosystem publishes

First party means the repository sits under the project's own GitHub
organisation, or the official documentation labels it official. Where the two
tests disagree, the row says so.

| Chain    | TS/JS            | Rust                     | Go               | Python           | Java/Kotlin      | Swift          | C/C++            | .NET           |
| -------- | ---------------- | ------------------------ | ---------------- | ---------------- | ---------------- | -------------- | ---------------- | -------------- |
| Ethereum | community [67]   | community [68]           | first party [69] | first party [70] | community [67]   | [NOT FOUND]    | [NOT FOUND]      | community [67] |
| Bitcoin  | community [71]   | community [71]           | community [71]   | community [71]   | community [71]   | community [72] | community [71]   | community [71] |
| Solana   | first party [73] | first party [74]         | community [73]   | community [73]   | community [73]   | [NOT FOUND]    | [NOT FOUND]      | [NOT FOUND]    |
| XRPL     | first party [75] | first party [85]         | community [75]   | first party [75] | first party [75] | [NOT FOUND]    | first party [75] | [NOT FOUND]    |
| Cosmos   | first party [76] | [NOT FOUND]              | first party [77] | [NOT FOUND]      | [NOT FOUND]      | [NOT FOUND]    | [NOT FOUND]      | [NOT FOUND]    |
| Stellar  | first party [78] | first party [78]         | first party [78] | community [78]   | community [78]   | community [78] | [NOT FOUND]      | community [78] |
| NEAR     | first party [79] | first party [79]         | [NOT FOUND]      | community [79]   | [NOT FOUND]      | [NOT FOUND]    | [NOT FOUND]      | [NOT FOUND]    |
| Sui      | first party [80] | first party [80]         | community [80]   | community [80]   | community [80]   | community [80] | [NOT FOUND]      | [NOT FOUND]    |
| Logos L1 | [NOT FOUND]      | in-repo Rust client [19] | [NOT FOUND]      | [NOT FOUND]      | [NOT FOUND]      | [NOT FOUND]    | C ABI [81]       | [NOT FOUND]    |
| LEZ      | [NOT FOUND]      | in-repo Rust client [21] | [NOT FOUND]      | [NOT FOUND]      | [NOT FOUND]      | [NOT FOUND]    | wallet FFI [21]  | [NOT FOUND]    |

TypeScript and Rust appear in every external ecosystem surveyed except Cosmos,
where no Rust client SDK sits under the `cosmos` organisation. Go and Python
appear in seven of eight. At the thin end, C and C++ appear as a first-party SDK
only for XRPL, .NET is community maintained everywhere it appears, and no
surveyed chain publishes a first-party Swift SDK.

Bitcoin is the sharpest contrast: it has no foundation-published SDK at all. The
`bitcoin` GitHub organisation contains the node and the BIPs, and every client
library is community maintained [71].

Sui and Stellar state maintainership explicitly, splitting their SDK pages into
official and community sections. Ethereum, Bitcoin, Cosmos, and XRPL publish no
such labels, so those rows rest on the organisation test alone. XRPL's Go entry
is the clearest case: its client-libraries page lists `Peersyst/xrpl-go` without
comment, and the repository sits outside the XRPLF organisation [75].

### 3.2 Generating many SDKs from one core

BDK is the clearest worked example of the FFI pattern. The `bdk-ffi` repository
"creates a library ready for export to other languages using uniffi-rs for the
Rust-based bdk_wallet library" [72]. It builds Kotlin, Android, and Swift
targets in-repo, and Kotlin JVM, Python, Dart, and React Native TypeScript are
maintained as separate downstream repositories consuming the same binding layer
[72]. One Rust core therefore reaches six language targets, although only the
first group is produced by the binding repository itself. Notably it comes from
a community project rather than a foundation, and the BDK core repository does
not mention the bindings at all.

Cosmos's preference is recorded as an encoding decision that cascades into
clients. ADR-019 lists "Codegen-based over reflection-based" among its criteria
and states the problem with the predecessor: Amino "has proven to be a big
pain-point in regards to supporting object serialization across clients written
in various languages" [116]. Mesh makes generation a property of the
specification, noting that requests and responses "can be crafted with
auto-generated code using Swagger Codegen or OpenAPI Generator" [117].

Schema-driven generation is the other route, and the survey shows both ends of
the spectrum. Cosmos generates from protobuf: each module "exposes a Protobuf
Query service that defines state queries", gRPC is chosen because it "has decent
client support in several languages", and a Swagger specification "is exposed
under the /swagger route on the API server" [8]. This is why Cosmos publishes so
few named per-language SDKs: the protobuf contract plus each language's own gRPC
toolchain substitutes for them. Sui similarly publishes protobuf interface
definitions usable to generate client libraries [60].

No surveyed chain publishes a written argument for hand-writing client SDKs
instead of generating them. Stellar's build-your-own-SDK guide is sometimes read
that way, but it scopes itself to contract SDKs: "This is for building an SDK
for writing smart contracts" [82]. Its client SDKs are simply listed with their
maintainers, several of them community-run [78], which is consistent with
hand-writing without arguing for it.

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
| Estimate execution cost                        | 5 of 8         |
| Pre-submission acceptance check                | 6 of 8         |
| Wait for a chosen confirmation level           | 4 of 8         |
| Read-only contract call                        | 6 of 8         |
| Verify a signature                             | 3 of 8         |
| Build an unsigned transaction as a node method | 3 of 8         |
| Encode and decode transaction bytes            | 2 of 8         |
| Submit a batch or package                      | 1 of 8         |

Simulation is the widest gap: six of the eight surveyed chains execute a
transaction and return its outcome without committing it, and neither Logos
target does. Pre-submission checking and read-only calls follow at six of eight.

Network identification is absent on Logos L1 alone: all eight surveyed chains
expose it, and LEZ exposes `getChannelId` [21].

### 4.2 Present but narrower than the surveyed norm

| Function                      | Logos state                                                                                                   |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------- |
| Get a balance                 | L1 scoped to node-custodial keys, not arbitrary subjects [19]                                                 |
| Get transaction status        | L1 reports mempool status only, no lifecycle or finality [19]                                                 |
| Get execution effects         | L1 block scoped, no per-transaction effects view [19]; LEZ absent [21]                                        |
| Paginate a result set         | LEZ mixes a cursor idiom and an offset idiom in one interface [22]                                            |
| Subscribe to new blocks       | Live streams accept no start position on either target; L1 offers a separate bounded range route [19][84][22] |
| Structured errors             | LEZ uses stock JSON-RPC codes with free text, no taxonomy [55]                                                |
| Query historical transactions | LEZ indexer only, offset paginated [22]                                                                       |
| Estimate the fee or fee rate  | L1 serves gas-price inputs, not an estimate for given bytes [19]                                              |

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

001. Go Ethereum, "JSON-RPC Server" documentation.
     https://geth.ethereum.org/docs/interacting-with-geth/rpc
002. Ethereum, "execution-apis" specification repository.
     https://github.com/ethereum/execution-apis
003. Bitcoin Core, "JSON-RPC Interface" documentation.
     https://raw.githubusercontent.com/bitcoin/bitcoin/master/doc/JSON-RPC-interface.md
004. Bitcoin Core, "Unauthenticated REST Interface" documentation.
     https://github.com/bitcoin/bitcoin/blob/master/doc/REST-interface.md
005. Solana, "RPC API" documentation. https://solana.com/docs/rpc
006. XRPL, "HTTP and WebSocket APIs" reference.
     https://xrpl.org/docs/references/http-websocket-apis
007. XRPL, "xrp_ledger.proto" gRPC service definition (rippled).
     https://raw.githubusercontent.com/XRPLF/rippled/develop/include/xrpl/proto/org/xrpl/rpc/v1/xrp_ledger.proto
008. Cosmos SDK, "gRPC, REST, and CometBFT Endpoints" documentation, v0.50.
     https://docs.cosmos.network/sdk/v0.50/learn/advanced/grpc_rest
009. CometBFT, "RPC OpenAPI specification".
     https://raw.githubusercontent.com/cometbft/cometbft/main/rpc/openapi/openapi.yaml
010. Cosmos SDK, "cosmos/base/tendermint/v1beta1/query.proto".
     https://raw.githubusercontent.com/cosmos/cosmos-sdk/main/proto/cosmos/base/tendermint/v1beta1/query.proto
011. Stellar, "Stellar RPC" documentation.
     https://developers.stellar.org/docs/data/apis/rpc
012. Stellar, "stellar-rpc.openrpc.json" specification document.
     https://raw.githubusercontent.com/stellar/stellar-docs/main/static/stellar-rpc.openrpc.json
013. NEAR, "RPC API" documentation. https://docs.near.org/api/rpc/introduction
014. NEAR, "nearcore JSON-RPC OpenAPI specification".
     https://raw.githubusercontent.com/near/nearcore/master/chain/jsonrpc/openapi/openapi.json
015. Sui, "Full Node Protocol" reference.
     https://docs.sui.io/references/fullnode-protocol
016. Sui, "JSON-RPC Migration" documentation.
     https://docs.sui.io/develop/accessing-data/json-rpc-migration
017. Sui, "Sui JSON-RPC OpenRPC specification", version 1.80.0.
     https://raw.githubusercontent.com/MystenLabs/sui/main/crates/sui-open-rpc/spec/openrpc.json
018. Sui, "GraphQL RPC" documentation. https://docs.sui.io/concepts/graphql-rpc
019. logos-blockchain, `nodes/api-common/src/paths.rs` and
     `nodes/node/binary/src/api/backend.rs`, commit `ecb2cc6`.
     https://github.com/logos-blockchain/logos-blockchain
020. logos-blockchain, `nodes/node/binary/src/api/openapi.rs` and
     `nodes/node/binary/src/api/backend.rs:215`, commit `ecb2cc6`.
     https://github.com/logos-blockchain/logos-blockchain
021. logos-execution-zone, `lez/sequencer/service/rpc/src/lib.rs`, commit
     `47eba25`. https://github.com/logos-blockchain/logos-execution-zone
022. logos-execution-zone, `lez/indexer/service/rpc/src/lib.rs`, commit
     `47eba25`. https://github.com/logos-blockchain/logos-execution-zone
023. Ethereum, "JSON-RPC API" documentation.
     https://ethereum.org/en/developers/docs/apis/json-rpc/
024. Bitcoin, "Original Bitcoin client RPC API reference".
     https://developer.bitcoin.org/reference/rpc/
025. Solana, "RPC HTTP Methods" documentation. https://solana.com/docs/rpc/http
026. XRPL, "Public API Methods" reference.
     https://xrpl.org/public-api-methods.html
027. Stellar, "Stellar RPC Methods" reference.
     https://developers.stellar.org/docs/data/apis/rpc/api-reference/methods
028. Ethereum, "execution-apis: src/eth/client.yaml".
     https://raw.githubusercontent.com/ethereum/execution-apis/main/src/eth/client.yaml
029. Cosmos SDK, "cosmos/tx/v1beta1/service.proto".
     https://raw.githubusercontent.com/cosmos/cosmos-sdk/main/proto/cosmos/tx/v1beta1/service.proto
030. Ethereum, "execution-apis: src/eth/state.yaml".
     https://raw.githubusercontent.com/ethereum/execution-apis/main/src/eth/state.yaml
031. Cosmos SDK, "cosmos/auth/v1beta1/query.proto".
     https://raw.githubusercontent.com/cosmos/cosmos-sdk/main/proto/cosmos/auth/v1beta1/query.proto
032. Cosmos SDK, "cosmos/bank/v1beta1/query.proto".
     https://raw.githubusercontent.com/cosmos/cosmos-sdk/main/proto/cosmos/bank/v1beta1/query.proto
033. Ethereum, "execution-apis: src/eth/transaction.yaml".
     https://raw.githubusercontent.com/ethereum/execution-apis/main/src/eth/transaction.yaml
034. NEAR, "RPC: Transactions" documentation.
     https://docs.near.org/api/rpc/transactions
035. Ethereum, "execution-apis: src/eth/execute.yaml".
     https://raw.githubusercontent.com/ethereum/execution-apis/main/src/eth/execute.yaml
036. Ethereum, "execution-apis: src/eth/submit.yaml".
     https://raw.githubusercontent.com/ethereum/execution-apis/main/src/eth/submit.yaml
037. Bitcoin, "BIP-174: Partially Signed Bitcoin Transaction Format".
     https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki
038. XRPL, "simulate" transaction method.
     https://xrpl.org/docs/references/http-websocket-apis/public-api-methods/transaction-methods/simulate
039. Stellar, "simulateTransaction" method reference.
     https://developers.stellar.org/docs/data/apis/rpc/api-reference/methods/simulateTransaction
040. Ethereum, "execution-apis: src/eth/fee_market.yaml".
     https://raw.githubusercontent.com/ethereum/execution-apis/main/src/eth/fee_market.yaml
041. Ethereum, "EIP-1559: Fee market change for ETH 1.0 chain".
     https://eips.ethereum.org/EIPS/eip-1559
042. Solana, "sendTransaction" RPC method.
     https://solana.com/docs/rpc/http/sendtransaction
043. JSON-RPC Working Group, "JSON-RPC 2.0 Specification".
     https://www.jsonrpc.org/specification
044. Solana, "getSignatureStatuses" RPC method.
     https://solana.com/docs/rpc/http/getsignaturestatuses
045. Solana, "RPC WebSocket Methods" documentation.
     https://solana.com/docs/rpc/websocket
046. Ethereum, "execution-apis: src/eth/filter.yaml".
     https://raw.githubusercontent.com/ethereum/execution-apis/main/src/eth/filter.yaml
047. XRPL, "account_tx" account method.
     https://xrpl.org/docs/references/http-websocket-apis/public-api-methods/account-methods/account_tx
048. Stellar, "List All Effects" Horizon reference.
     https://developers.stellar.org/docs/data/apis/horizon/api-reference/list-all-effects
049. Cosmos SDK, "cosmos/base/query/v1beta1/pagination.proto".
     https://raw.githubusercontent.com/cosmos/cosmos-sdk/main/proto/cosmos/base/query/v1beta1/pagination.proto
050. Stellar, "Horizon Pagination" documentation.
     https://developers.stellar.org/docs/data/apis/horizon/api-reference/structure/pagination
051. Go Ethereum, "Publish and Subscribe" documentation.
     https://geth.ethereum.org/docs/interacting-with-geth/rpc/pubsub
052. Bitcoin Core, "ZeroMQ" documentation.
     https://github.com/bitcoin/bitcoin/blob/master/doc/zmq.md
053. XRPL, "Error Formatting" API convention.
     https://xrpl.org/docs/references/http-websocket-apis/api-conventions/error-formatting
054. gRPC, "Status codes and their use in gRPC".
     https://grpc.io/docs/guides/status-codes/
055. logos-execution-zone, `lez/indexer/service/src/service.rs:343-349`, commit
     `47eba25`. https://github.com/logos-blockchain/logos-execution-zone
056. Ethereum, "EIP-1193: Ethereum Provider JavaScript API".
     https://eips.ethereum.org/EIPS/eip-1193
057. Go Ethereum, "GraphQL Server" documentation.
     https://geth.ethereum.org/docs/interacting-with-geth/rpc/graphql
058. XRPL, "Request Formatting" API convention.
     https://xrpl.org/docs/references/http-websocket-apis/api-conventions/request-formatting
059. Stellar, "Horizon Streaming" documentation.
     https://developers.stellar.org/docs/data/apis/horizon/api-reference/structure/streaming
060. Sui, "sui-apis" protobuf interface definitions.
     https://github.com/MystenLabs/sui-apis
061. OpenRPC, project homepage. https://open-rpc.org/
062. gRPC, "Introduction to gRPC".
     https://grpc.io/docs/what-is-grpc/introduction/
063. gRPC, "Core concepts, architecture and lifecycle".
     https://grpc.io/docs/what-is-grpc/core-concepts/
064. GraphQL Foundation, "GraphQL specification" repository.
     https://raw.githubusercontent.com/graphql/graphql-spec/main/README.md
065. GraphQL Foundation, "GraphQL over HTTP" specification repository.
     https://github.com/graphql/graphql-over-http
066. WHATWG, "Server-sent events", HTML Living Standard.
     https://html.spec.whatwg.org/multipage/server-sent-events.html
067. Ethereum, "JavaScript API libraries" documentation.
     https://ethereum.org/en/developers/docs/apis/javascript/
068. Ethereum, "Rust developer resources".
     https://ethereum.org/en/developers/docs/programming-languages/rust/
069. Ethereum, "go-ethereum" repository. https://github.com/ethereum/go-ethereum
070. Ethereum, "web3.py" repository. https://github.com/ethereum/web3.py
071. Bitcoin, "Bitcoin Development" resources page.
     https://bitcoin.org/en/development
072. Bitcoin Dev Kit, "bdk-ffi" repository.
     https://github.com/bitcoindevkit/bdk-ffi
073. Solana, "Solana Clients" documentation. https://solana.com/docs/clients
074. Solana, "Rust Client" documentation. https://solana.com/docs/clients/rust
075. XRPL, "Client Libraries" reference.
     https://xrpl.org/docs/references/client-libraries
076. Cosmos, "CosmJS" repository. https://github.com/cosmos/cosmjs
077. Cosmos, "cosmos-sdk" repository. https://github.com/cosmos/cosmos-sdk
078. Stellar, "Client SDKs" documentation.
     https://developers.stellar.org/docs/tools/sdks/client-sdks
079. NEAR, "NEAR API" documentation. https://docs.near.org/tools/near-api
080. Sui, "Sui SDKs" reference. https://docs.sui.io/references/sui-sdks
081. logos-blockchain, `c-bindings/logos_blockchain.h` and
     `c-bindings/cbindgen.toml`, commit `ecb2cc6`.
     https://github.com/logos-blockchain/logos-blockchain
082. Stellar, "Build Your Own SDK" documentation.
     https://developers.stellar.org/docs/tools/sdks/build-your-own
083. Solana, "web3.js Compatibility" documentation.
     https://solana.com/docs/frontend/web3-compat
084. logos-blockchain, `nodes/api-common/src/paths.rs:33` and
     `nodes/node/binary/src/api/queries.rs`, commit `ecb2cc6`.
     https://github.com/logos-blockchain/logos-blockchain
085. XRPL Foundation, "xrpl-rust" repository. https://github.com/XRPLF/xrpl-rust
086. Ethereum, "EIP-155: Simple replay attack protection".
     https://eips.ethereum.org/EIPS/eip-155
087. Ethereum, "ERC-191: Signed Data Standard".
     https://eips.ethereum.org/EIPS/eip-191
088. Ethereum, "EIP-712: Typed structured data hashing and signing".
     https://eips.ethereum.org/EIPS/eip-712
089. Bitcoin, "BIP-331: Ancestor Package Relay".
     https://github.com/bitcoin/bips/blob/master/bip-0331.mediawiki
090. Solana, "Durable and Offline Transaction Signing using Nonces".
     https://solana.com/developers/guides/advanced/introduction-to-durable-nonces
091. Cosmos SDK, "ADR-070: Unordered Transactions".
     https://raw.githubusercontent.com/cosmos/cosmos-sdk/main/docs/architecture/adr-070-unordered-account.md
092. NEAR, "NEP-413: Wallet API for signing messages".
     https://github.com/near/NEPs/blob/master/neps/nep-0413.md
093. Cosmos SDK, "cosmos/base/abci/v1beta1/abci.proto".
     https://raw.githubusercontent.com/cosmos/cosmos-sdk/main/proto/cosmos/base/abci/v1beta1/abci.proto
094. Ethereum, "EIP-695: Create eth_chainId method for JSON-RPC".
     https://eips.ethereum.org/EIPS/eip-695
095. Ethereum, "EIP-1898: Add blockHash to defaultBlock methods".
     https://eips.ethereum.org/EIPS/eip-1898
096. Ethereum, "EIP-234: Add blockHash to JSON-RPC filter options".
     https://eips.ethereum.org/EIPS/eip-234
097. Ethereum, "EIP-4444: Bound Historical Data in Execution Clients".
     https://eips.ethereum.org/EIPS/eip-4444
098. Ethereum, "EIP-1767: GraphQL interface to Ethereum node data".
     https://eips.ethereum.org/EIPS/eip-1767
099. Ethereum, "EIP-1474: Remote procedure call specification".
     https://eips.ethereum.org/EIPS/eip-1474
100. Ethereum, "execution-apis: src/schemas/receipt.yaml".
     https://raw.githubusercontent.com/ethereum/execution-apis/main/src/schemas/receipt.yaml
101. XRPL, "server_info" method reference.
     https://raw.githubusercontent.com/XRPLF/xrpl-dev-portal/master/docs/references/http-websocket-apis/public-api-methods/server-info-methods/server_info.md
102. XRPL, "server_definitions" method reference.
     https://raw.githubusercontent.com/XRPLF/xrpl-dev-portal/master/docs/references/http-websocket-apis/public-api-methods/server-info-methods/server_definitions.md
103. XRPL, "Partial Payments" concept documentation.
     https://raw.githubusercontent.com/XRPLF/xrpl-dev-portal/master/docs/concepts/payment-types/partial-payments.md
104. Anza, "Agave CHANGELOG".
     https://raw.githubusercontent.com/anza-xyz/agave/master/CHANGELOG.md
105. Bitcoin Core, "src/rpc/blockchain.cpp".
     https://raw.githubusercontent.com/bitcoin/bitcoin/master/src/rpc/blockchain.cpp
106. Cosmos SDK, "ADR-032: Typed Events" (status: proposed).
     https://raw.githubusercontent.com/cosmos/cosmos-sdk/main/docs/architecture/adr-032-typed-events.md
107. Sui, "GraphQL and Archival Store Complete the Sui Data Stack".
     https://www.sui.io/blog/graphql-archival-store-sui-data-stack
108. Aptos, "aptos-core: api/src/transactions.rs".
     https://raw.githubusercontent.com/aptos-labs/aptos-core/main/api/src/transactions.rs
109. Go Ethereum, "all: remove personal RPC namespace", PR 30704.
     https://github.com/ethereum/go-ethereum/pull/30704
110. Bitcoin Core, "doc/design/multiprocess.md".
     https://raw.githubusercontent.com/bitcoin/bitcoin/master/doc/design/multiprocess.md
111. Ethereum, "JSON-RPC: Add finalized and safe blocks", execution-apis PR 200.
     https://github.com/ethereum/execution-apis/pull/200
112. NEAR, "RPC end point to configure broadcast transaction await", nearcore
     issue 6837. https://github.com/near/nearcore/issues/6837
113. Algorand, "algod: Sunset v1 handlers", go-algorand PR 4847.
     https://github.com/algorand/go-algorand/pull/4847
114. Anza, "Remove support for deprecated rpc endpoints", agave PR 1809.
     https://github.com/anza-xyz/agave/pull/1809
115. logos-execution-zone, `lez/indexer/service/protocol/src/lib.rs:444`, commit
     `47eba25`. https://github.com/logos-blockchain/logos-execution-zone
116. Cosmos SDK, "ADR-019: Protocol Buffer State Encoding".
     https://raw.githubusercontent.com/cosmos/cosmos-sdk/main/docs/architecture/adr-019-protobuf-state-encoding.md
117. Coinbase, "mesh-specifications" repository.
     https://raw.githubusercontent.com/coinbase/mesh-specifications/master/README.md
