# Appendix: DEX Ecosystem Behaviour

This appendix surveys how existing decentralised exchanges implement
behaviours required by
[RFP-004](../RFPs/RFP-004-privacy-preserving-dex.md). Each section
maps an RFP requirement to the equivalent mechanism in production
protocols, showing that the requirement reflects established industry
practice. All data is sourced from the
[research-dex](https://github.com/marclawclaw/research-dex) vault;
individual project notes contain full citations.

## Protocols considered

| Protocol | Ecosystem | Type | TVL | Cumulative volume |
|----------|-----------|------|-----|-------------------|
| Uniswap V2 | Ethereum | Constant-product AMM | ~$945M | $604B+ |
| Uniswap V4 | Ethereum (17 chains) | Singleton AMM with hooks | >$1B | >$190B (2025) |
| Balancer V3 | Ethereum (9 chains) | Vault-based AMM with hooks | $104M to $151M | N/A (fee data only) |
| Curve StableSwapNG | Ethereum | StableSwap AMM | ~$2.3B | ~$126B (2025) |
| CoW Protocol | Ethereum (10 chains) | Intent-based batch auction | N/A (no custody) | $87B (2025) |
| Raydium | Solana | AMM + CLMM | $1.4B | $695.7B all-time |
| Orca Whirlpools | Solana (+ Eclipse) | CLMM | $246.8M | >$300B |

## 1. Constant-product AMM with public pool state

**RFP-004 requirement:** Implement an AMM program with public liquidity
pools (requirement F.1).

**Ecosystem practice:** The constant-product invariant (`x * y = k`) is
the most widely deployed AMM mechanism. Uniswap V2 is the reference
implementation with over 100 forks (PancakeSwap, SushiSwap, Aerodrome,
and others). Pool state (reserves, price, cumulative volume) is fully
public on-chain in every surveyed protocol. No production AMM stores
pool state privately.

On Solana, both Raydium and Orca implement the same constant-product
invariant for their base pools, adapted for the SVM account model:
token balances reside in SPL Token Accounts whose authority is a
program-derived address (PDA), and pool state is stored in a separate
data account.

RFP-004's specification of a public-state AMM with the
deshield/swap/re-shield privacy layer applied at the UX level is
consistent with the universal industry pattern: pool state is public;
privacy, where it exists, is enforced outside the pool contract.

## 2. Immutable fee tiers with multiple pools per pair

**RFP-004 requirement:** The pool creator selects a fee tier at
creation (e.g. 0.01%, 0.05%, 0.3%, 1%); the tier is immutable.
Multiple pools for the same pair with different tiers can coexist
(requirement F.6).

**Ecosystem practice:**

| Protocol | Fee tier model | Immutable | Multiple pools per pair |
|----------|---------------|-----------|------------------------|
| Uniswap V2 | Fixed 0.3% (all pools) | Yes | No (one pool per pair) |
| Uniswap V4 | Configurable per pool; hooks can add dynamic fees | Yes (base tier) | Yes |
| Balancer V3 | Configurable per pool; StableSurge hook adds dynamic fee | Yes (base tier) | Yes |
| Curve StableSwapNG | Configurable per pool; `offpeg_fee_multiplier` adds dynamic scaling | Yes (base tier) | Yes |
| Raydium CLMM | 8 fee tiers (0.01% to 2%) | Yes | Yes |
| Orca Whirlpools | 6 tick spacings mapping to fee tiers (0.01% to 2%) | Yes (adaptive pools add a variable component) | Yes |

Immutable base fee tiers are the norm. Every protocol launched after
Uniswap V2 supports multiple pools per pair with different fee tiers.
The RFP-004 fee model (0.01%, 0.05%, 0.3%, 1%) mirrors the standard
Uniswap V3/V4 tier set, which is also the set used by Raydium and Orca
(with additional tiers at 0.02% and 2%).

## 3. Trading fees paid by trader, distributed to LPs

**RFP-004 requirement:** Trading fees are paid by the trader and
distributed to LPs (requirement F.6).

**Ecosystem practice:**

| Protocol | Fee payer | LP share | Protocol share | Other |
|----------|-----------|----------|----------------|-------|
| Uniswap V2 | Trader (input token) | 100% (protocol fee optional: 0.05%) | 0% (or 1/6 of 0.3% when active) | Protocol fee activated Dec 2025 |
| Uniswap V4 | Trader (input token) | Majority | Configurable per pool | Hook can adjust |
| Balancer V3 | Trader (input token) | ~50% (varies) | ~50% (split with pool creator) | Yield fee: 10% on boosted pools |
| Curve StableSwapNG | Trader (output token) | Majority | Admin fee (fraction of swap fee) | Dynamic fee on imbalanced pools |
| Raydium CLMM | Trader | 84% | 12% RAY buyback + 4% treasury | |
| Orca Whirlpools | Trader | 87% | 12% DAO treasury + 1% Climate Fund | |

In every surveyed protocol, the trader pays fees as part of the swap.
Fees accrue to LPs implicitly by growing the pool invariant (Uniswap
V2/V4), or via fee accumulator tracking (CLMM protocols, Curve).

The RFP-004 model (trader pays, LPs receive) is universal.

## 4. Slippage protection with minimum output guarantees

**RFP-004 requirement:** Implement slippage protection with
user-configurable tolerance and minimum output guarantees (requirement
F.7).

**Ecosystem practice:** Every surveyed protocol enforces slippage
protection via a `minimum_amount_out` (or equivalent) parameter that
reverts the transaction if the output falls below the user's threshold:

| Protocol | Parameter | Enforcement |
|----------|-----------|-------------|
| Uniswap V2 | `amountOutMin` on Router | Router reverts if output < minimum |
| Uniswap V4 | `amountSpecified` + `sqrtPriceLimitX96` | PoolManager reverts on limit breach |
| Balancer V3 | `limit` in swap params | Vault reverts if output < limit |
| Curve StableSwapNG | `_min_dy` on `exchange()` | Pool reverts if output < minimum |
| Raydium | `minimum_amount_out` | Program reverts if output < minimum |
| Orca Whirlpools | `other_amount_threshold` | Program reverts if output < threshold |

This is a universal safety mechanism. No production AMM omits it.

## 5. Pool creation for arbitrary token pairs

**RFP-004 requirement:** Support creation of liquidity pools for
arbitrary token pairs (requirement F.2).

**Ecosystem practice:** Permissionless pool creation is the default:

| Protocol | Permissionless | Factory pattern |
|----------|---------------|-----------------|
| Uniswap V2 | Yes | `createPair()` on Factory contract |
| Uniswap V4 | Yes | `initialize()` on PoolManager |
| Balancer V3 | Yes | Pool registration on Vault |
| Curve StableSwapNG | Yes | `CurveStableSwapFactoryNG.vy` using `create_from_blueprint()` |
| Raydium | Yes | `initialize()` instruction per pool type |
| Orca Whirlpools | Yes | `initialize_pool()` instruction |

Every protocol uses a factory or registry pattern where any user can
create a pool for any token pair. Curve restricts some parameters
(asset type classification) but pool creation itself is open.

## 6. Single-transaction operations

**RFP-004 requirement:** A swap, pool creation, and liquidity
operations each complete within a single LEZ transaction (requirements
P.1, P.3).

**Ecosystem practice:** Every surveyed protocol executes swaps, pool
creation, and liquidity add/remove as atomic single-transaction
operations. On Ethereum, this is a single EVM transaction. On Solana,
this is a single Solana transaction (which may contain multiple
instructions).

Uniswap V4 and Balancer V3 go further: multi-hop swaps across multiple
pools settle in a single transaction with only two token transfers
(input and output) regardless of the number of intermediate pools,
using flash accounting. On Solana, Raydium and Orca achieve multi-hop
routing through Jupiter aggregation within a single transaction.

The RFP-004 single-transaction requirement is consistent with
every production protocol.

## 7. Pool analytics: aggregate volume, TVL, and fee revenue

**RFP-004 requirement:** Provide a pool analytics view showing
aggregate volume, TVL, and fee revenue without revealing individual
positions (requirement U.3).

**Ecosystem practice:** All surveyed protocols expose pool-level
aggregate metrics on-chain or via indexers:

| Metric | On-chain | Typical source |
|--------|----------|----------------|
| TVL | Derived from pool token balances (public) | DeFiLlama, protocol subgraphs |
| Volume | Cumulative swap counters stored on-chain (Uniswap V2: `kLast`, TWAP accumulators; Raydium CLMM: `swap_in_amount` / `swap_out_amount` fields) | Subgraph indexing of Swap events |
| Fee revenue | Derived from volume x fee rate, or accumulated in on-chain fee counters | Protocol dashboards |

Individual LP positions are public on-chain in every protocol (LP token
balances in Uniswap V2, NFT positions in CLMMs, `admin_balances` in
Curve). RFP-004's privacy model does not change this: LP positions
remain public, but the private account that originated the funds is not
traceable when the deshield pattern is used.

## 8. Associated Token Accounts for token custody

**RFP-004 requirement:** Use Associated Token Accounts (ATAs) for all
token interactions, derived per `(owner, mint)` pair (requirement F.8).

**Ecosystem practice on Solana:** Both Raydium and Orca use
deterministic PDA-derived token accounts for pool vaults:

- **Raydium CLMM:** Vault seed `["pool_vault", pool_state, token_mint]`
- **Orca Whirlpools:** Similar PDA derivation per pool per token mint

User-side, the SPL Associated Token Account program derives one token
account per `(wallet, mint)` pair, and Solana DEX frontends universally
create ATAs on-demand.

On LEZ (which uses SVM), the ATA requirement aligns with standard
Solana practice.

## 9. Vault accounting and balance reconciliation

**RFP-004 context:** The DEX program must maintain correct pool
balances under concurrent swaps (requirement R.1).

Three distinct vault accounting models exist in production:

### Cached reserves with sync/skim (Uniswap V2)

The pool caches token reserves in storage (`reserve0`, `reserve1`) and
updates them after each operation. Actual ERC-20 balances can drift
from cached reserves through direct transfers, rebasing tokens, or
fee-on-transfer mechanics. Two functions handle reconciliation:

- **`sync()`** updates cached reserves to match live balances (absorbs
  a deficit from a negative rebase; LPs bear the loss).
- **`skim(to)`** transfers the surplus (balance minus reserve) to a
  caller (extracts orphaned tokens from a positive rebase; anyone can
  call it).

This model is simple and self-healing but leaks positive rebase value
to MEV bots (the `skim()` surplus is publicly extractable).

### Live balance reads with fee isolation (Curve StableSwapNG)

The pool reads live token balances via `_balance()` on every operation
and stores admin fees in a separate `admin_balances[]` array. There is
no cached reserve to drift. Positive rebases accrue to LPs (not
skimmable by bots); negative rebases reduce LP value automatically.
Admin fees are isolated from LP balances, so neither rebases nor direct
transfers corrupt fee accounting.

### Flash accounting / transient accounting (Uniswap V4, Balancer V3)

A singleton contract holds all pool tokens. Operations accumulate
credits and debits in EIP-1153 transient storage. Only the net token
amounts are transferred at the end of the session. There is no
cached reserve separate from the live balance. This model eliminates
the sync/skim pattern entirely and achieves a 98% gas reduction for
state tracking versus persistent storage.

### Solana/SVM account model (Raydium, Orca)

Token balances reside in SPL Token Accounts whose authority is a pool
PDA. The pool program can only move tokens *out* of the vault by
signing with the PDA via `invoke_signed`. However, any external account
can transfer tokens *into* a vault account: the SPL Token `transfer`
instruction only requires the sender's signature, not the recipient's.
This means surplus can accumulate in vault accounts through unsolicited
transfers, just as on Ethereum.

Neither Raydium nor Orca implements a `sync()` or `skim()` equivalent.
Both protocols update pool state atomically within each swap or
liquidity instruction and do not read live vault balances for reserve
reconciliation. In practice, unsolicited transfers to pool vaults are
rare on Solana (there are no rebasing tokens, no fee-on-transfer
mechanics, and no interest accrual), so the surplus problem has not
been operationally significant. The surplus simply sits in the vault,
unacknowledged by the pool's reserve accounting.

### Summary

| Model | Surplus possible | Recovery mechanism | Rebase handling |
|-------|-----------------|-------------------|-----------------|
| Cached reserves (V2) | Yes (direct transfer, rebase) | `sync()` / `skim()` | Partial |
| Live balance reads (Curve) | No (reads live balance) | Not needed | Native |
| Flash accounting (V4, Balancer V3) | No (singleton, transient deltas) | Not needed | Via hooks |
| SVM accounts (Raydium, Orca) | Yes (unsolicited transfer) | None implemented | Not applicable (no rebasing tokens on SVM) |

## 10. Recommendation: `sync()` and `recoverSurplus()` functions

**Context:** The LEZ DEX implementation uses cached reserves (the
Uniswap V2 pattern). Because anyone can transfer SPL tokens into a
pool's vault account without invoking the pool program, the actual
vault balance can exceed the pool's recorded reserves. Two functions
handle this discrepancy:

- **`sync()`** updates the cached reserves to match the live vault
  balance, absorbing the surplus into the pool. The surplus becomes
  part of the pool's reserves, benefiting all current LPs
  proportionally.
- **`recoverSurplus(to)`** (Uniswap V2's `skim()`) transfers the
  difference between the vault balance and the cached reserve to a
  specified address. The reserves remain unchanged; only the excess
  leaves.

### How surplus arises on LEZ

On SVM, surplus can only arise through one channel: an external account
explicitly transfers tokens into the pool's vault account outside the
normal swap or liquidity flow. Unlike Ethereum, there are no rebasing
tokens, no fee-on-transfer mechanics, and no interest-bearing token
accrual on SVM. The surplus problem on LEZ is therefore narrower than
on Ethereum: it is limited to unsolicited direct transfers, whether
accidental or intentional.

### Why sync() is needed

Even though surplus arises less frequently on SVM than on Ethereum,
`sync()` serves an important role: it reconciles the pool's internal
accounting with the actual vault balance. Without it, surplus tokens
sit in the vault permanently, unacknowledged by the pool's reserve
state. `sync()` lets anyone absorb the surplus into the pool, where it
benefits LPs.

On Ethereum, `sync()` also handles the opposite case: if a negative
rebase reduces the vault balance below the cached reserve, `sync()`
writes down the reserve so that swaps do not revert. On SVM, this
scenario cannot occur (token balances only change through explicit
instructions), so `sync()` on LEZ is strictly a surplus-absorption
function.

### The case for recoverSurplus()

The implementor identifies one scenario where `recoverSurplus()` is
genuinely useful: all LPs remove their liquidity, but surplus tokens
remain in the vault. In this state:

- Calling `sync()` would set the reserves to the vault balance, but
  with zero LP tokens outstanding, no one holds a claim on those
  reserves.
- The next user to `sync()` and then add liquidity would effectively
  absorb the surplus into their position.
- `recoverSurplus()` provides a direct path to extract the tokens
  without requiring someone to enter the pool first.

### The case against recoverSurplus()

In Uniswap V2, `skim()` is permissionless: any caller can extract the
surplus at any time. This creates a race condition:

- On Ethereum, positive rebases generate surplus continuously, and MEV
  bots monitor all pools for skimmable amounts. The economic value of
  rebases is captured by bots, not LPs.
- A caller can front-run a `recoverSurplus()` call with `sync()`,
  absorbing the surplus into the pool and leaving nothing to recover.

On LEZ, the MEV concern is reduced because surplus only arises from
unsolicited transfers (no rebasing tokens), making extractable surplus
rare and unpredictable. However, the front-running risk remains: any
pending `recoverSurplus()` transaction can be neutralised by a `sync()`
call.

### Ecosystem precedent

| Protocol | sync() | skim() / recoverSurplus() | Notes |
|----------|--------|--------------------------|-------|
| Uniswap V2 | Yes (permissionless) | Yes (permissionless) | Both needed for rebasing tokens and overflow protection |
| Uniswap V4 | Different function (settlement, not reconciliation) | No | Flash accounting eliminates cached reserve drift |
| Balancer V3 | No | No | Transient accounting; no cached reserves |
| Curve StableSwapNG | No (reads live balances) | No | Admin fees isolated in separate array |
| Raydium | No | No | Surplus silently accumulates; unrecoverable |
| Orca Whirlpools | No | No | Same as Raydium |

Uniswap V2 is the only protocol that implements both functions.
Later protocols eliminated the need entirely by moving to live balance
reads or transient accounting. Solana protocols do not implement either
function, and surplus tokens in Raydium and Orca vaults are
effectively permanently stuck.

### Recommendation

**Implement `sync()`. Implement `recoverSurplus()`, restricted to
the empty-pool case.**

1. **`sync()`** should be permissionless, matching Uniswap V2. It
   absorbs surplus into the pool, benefiting LPs. This is the expected
   behaviour: if tokens arrive in the vault, they should become part of
   the pool.

2. **`recoverSurplus(to)`** should be callable only when the pool has
   zero liquidity (total LP supply is zero). This addresses the
   implementor's scenario (surplus in an empty pool) without creating a
   permissionless extraction function that competes with `sync()` during
   normal operation.

   When LP supply is zero, `sync()` cannot benefit anyone because there
   are no LP holders. In this state, `recoverSurplus()` is the only
   way to move the tokens out, and there is no front-running concern
   because `sync()` in an empty pool would just set reserves that no
   one can claim.

   If a broader permissionless `recoverSurplus()` is preferred (matching
   Uniswap V2's `skim()` exactly), the trade-off is that callers will
   always race against `sync()` callers. On LEZ this is low-stakes
   (surplus is rare), but the restricted version is cleaner.

3. **Do not implement live balance reads as an alternative.** Curve
   StableSwapNG's `_balance()` approach avoids the problem entirely
   but adds an external call per token per operation. Since the LEZ DEX
   already uses cached reserves and `sync()`, switching to live reads
   would be a larger architectural change for marginal benefit on a
   chain with no rebasing tokens.

## References

Full research notes with per-claim source URLs are available in
the [research-dex](https://github.com/marclawclaw/research-dex)
Obsidian vault. Key sources by section:

1. Uniswap V2 whitepaper: https://app.uniswap.org/whitepaper.pdf
2. Uniswap V4 whitepaper: https://app.uniswap.org/whitepaper-v4.pdf
3. Balancer V3 launch: https://medium.com/balancer-protocol/balancer-v3-is-live-2e5e8462aa4c
4. Curve StableSwapNG docs: https://docs.curve.finance/stableswap-exchange/stableswap-ng/pools/overview/
5. Raydium CLMM source: https://github.com/raydium-io/raydium-clmm
6. Orca Whirlpools source: https://github.com/orca-so/whirlpools
7. Orca developer docs: https://dev.orca.so
8. DeFiLlama: https://defillama.com
