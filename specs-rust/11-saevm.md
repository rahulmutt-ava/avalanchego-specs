# 11 — SAE VM (Streaming Asynchronous Execution, ACP-194)

> **Status:** Frontier subsystem. Mirrors the Go note in `vms/saevm/README.md`
> ("`strevm` … under active development … no guarantees about the stability of
> its … APIs"). Per `00` §7.7 this crate family is held to the **stricter lint
> bar**: `clippy::pedantic`, `#![deny(clippy::arithmetic_side_effects)]` on the
> gas-time crates, `overflow-checks = true` in *all* profiles (release too),
> analogous to the Go `lint-saevm` / gosec G115 pass. This spec conforms to
> `00-overview-and-conventions.md` and cross-references `04` (storage/Firewood),
> `06` (consensus / Snowman `block.ChainVM`), `07` (VM framework), and `10`
> (C-Chain on reth). Where it deviates it says so and justifies it.

## Go source covered

| Go path | Rust crate |
|---|---|
| `vms/saevm/sae/` (+ `sae/rpc/`) | `ava-saevm-core` |
| `vms/saevm/blocks/` (+ `blockstest`) | `ava-saevm-blocks` |
| `vms/saevm/saexec/` | `ava-saevm-exec` |
| `vms/saevm/saedb/` | `ava-saevm-db` |
| `vms/saevm/gastime/` | `ava-saevm-gastime` |
| `vms/saevm/gasprice/` | `ava-saevm-gasprice` |
| `vms/saevm/proxytime/` | `ava-saevm-proxytime` |
| `vms/saevm/hook/` | `ava-saevm-hook` |
| `vms/saevm/adaptor/` | `ava-saevm-adaptor` |
| `vms/saevm/cchain/` (+ `state`, `tx`, `txpool`) | `ava-saevm-cchain` |
| `vms/saevm/txgossip/` | `ava-saevm-txgossip` |
| `vms/saevm/worstcase/` | `ava-saevm-worstcase` |
| `vms/saevm/{intmath,cmputils,types,params}` | `ava-saevm-{intmath,cmputils,types,params}` |
| `vms/saevm/saetest/` | `ava-saevm-testutil` (dev-dependency) |
| `vms/saevm/docs/invariants.md` | §10 of this spec (test-enforced) |

> **`libevm` mapping.** The Go code is built on `github.com/ava-labs/libevm`
> (`core.ApplyTransaction`, `core/rawdb`, `core/state`, `triedb`, `core/txpool`).
> Per `00` §4.5 and `10`, our EVM execution layer is **reth/revm + Firewood**, so
> we do **not** port `libevm`; we re-derive the same *behaviour* on reth. The SAE
> machinery (gas-time, settlement, the streaming pipeline, recovery) is
> EVM-engine-agnostic and is what this spec specifies precisely. See §8 for the
> reth/`ava-evm` reuse decision and the exact mapping of `libevm` calls.

---

## 1. The SAE model

Classic Avalanche VMs are **synchronous**: a block is *verified* (executed) before
it can be voted on, so consensus latency includes execution latency and the chain
throughput is bounded by single-threaded execution that must finish inside the
consensus round. ACP-194 **decouples ordering from execution**:

1. **Consensus orders blocks first.** A block carries transactions plus a small
   amount of *lagged* execution metadata (see below). Verification is cheap — it
   does **not** execute the transactions. Snowman votes purely on ordering.
2. **Execution streams asynchronously behind the accepted frontier.** On
   `Accept`, the block is pushed onto a FIFO queue. A single executor task drains
   the queue, executes each block deterministically against the post-state of its
   parent, commits to the state DB, and advances an **execution frontier** that
   *lags* the **consensus (accepted) frontier**.
3. **Results are referenced with a delay (settlement).** A new block does not
   embed its *own* post-execution state root (it isn't known yet). It embeds the
   post-execution state root of the **last settled block** — an ancestor that
   finished executing at least `Tau` (gas-)seconds ago. It also carries the
   execution artefacts (receipts root, etc.) of the ancestors it newly settles.

### 1.1 Three frontiers (not two)

The Go code (`blocks/access.go::Frontier`) tracks **three** monotonic pointers,
all derivable from disk after restart (invariants §10):

```
height ─────────────────────────────────────────────────▶
   … b_{s}        …        b_{e}        …        b_{a}
     ▲                       ▲                     ▲
 LastSettled            LastExecuted          LastAccepted
 (S frontier)            (E frontier)          (A frontier)

  Guarantees (temporal "happens-before", invariants doc):
    b ∈ S  ⟹  b ∈ E  ⟹  b ∈ A          (settle after exec after accept)
    S frontier ≤ E frontier ≤ A frontier  (heights, always)
```

- **Accepted (A):** consensus has committed the ordering. `rawdb` *canonical*.
- **Executed (E):** the executor has produced and committed the post-state. This
  is the EVM "head"/"latest". Lags A by the queue depth.
- **Settled (S):** an executed block whose results have been *referenced by a
  later accepted block* — the point at which results are demonstrably agreed and
  treated as "safe"/"finalized" (SAE has no reorgs, so settlement is the disk-
  corruption-safety analogue, not a finality delay). `rawdb` *finalized*.

The RPC label mapping (`blocks/access.go::ResolveRPCNumber`, invariants §"Height
mapping"):

| RPC label | SAE frontier |
|---|---|
| `pending` | LastAccepted (A) |
| `latest` | LastExecuted (E) |
| `safe`, `finalized` | LastSettled (S) |

### 1.2 Settlement rule (the heart of ACP-194)

A block `b` *settles* the contiguous half-open range of ancestors
`(parent.LastSettled, b.LastSettled]` (`blocks/settlement.go::Settles`,
`Range`). `b.LastSettled` is chosen at build time as **the last ancestor that
finished executing no later than `BlockTime(b) − Tau`**, measured on the
**gas-time clock** (§2), via `blocks::last_to_settle_at`
(`LastToSettleAt`). The "can we settle yet?" predicate returns `(block, known)`:
settlement is only permitted when the executor has progressed far enough that the
*child* of the candidate is provably **not** finished by `settleAt` — otherwise
`known = false` and the builder reports `ErrExecutionLagging` and tries later.

> **Why gas-time, not wall-time.** "Finished executing `Tau` ago" is measured in
> the *gas clock*: a block's execution-completion instant is the gas-time after
> its last tx ticks the clock (§2). This makes the settlement delay a function of
> *work done*, deterministic and replay-stable, independent of how fast/slow real
> hardware executed. This is the protocol-defining novelty.

### 1.3 What a block carries about execution

A SAE block is an Ethereum block (RLP, byte-identical hashing) where the header's
**`Root` field is repurposed**: it is the **post-execution state root of the
block's last settled ancestor** (`blocks/export.go::SettledStateRoot` returns
`types.Block.Root()`), **not** this block's own post-state (which is unknown at
build time). The newly-settled ancestors' receipts are carried so peers can
reconstruct settled receipt roots. A small `hook.Settled` struct
(`{Height, GasUnix, GasNumerator, Excess}`) records the settled block's
post-execution **gas-time and excess** so a freshly synced/recovered node can
reconstruct the gas clock without re-deriving it (`hook.go::Settled`,
`SettledGasTime`). The block builder populates `GasLimit`, `BaseFee`, `GasUsed`
from the **worst-case** prediction (§7), since true values aren't known until
execution.

### 1.4 Recovery after restart (`sae/recovery.go`)

On startup the VM rebuilds all three frontiers from disk with **no trust in
in-memory state**:
1. `last_committed_block` = highest height whose **post-execution state root was
   committed** to the trie DB (`saedb::last_height_with_execution_root_committed`,
   driven by the commit interval / archival flag).
2. Re-enqueue every *accepted-but-not-executed* canonical block
   (`execute_all_accepted`) and `wait_until_executed` the tip → rebuilds E.
3. Walk back from the executed tip reconstructing the *consensus-critical* set
   (accepted blocks from LastExecuted back through LastSettled), calling
   `mark_settled` on those whose gas-time ≤ `BlockTime − Tau`
   (`consensus_critical_blocks`) → rebuilds S.

Determinism guarantee: re-execution from the last committed root reproduces the
exact same post-state roots, because execution is a pure function of the ordered
blocks + parent state (no wall-clock, no map-iteration nondeterminism — §6.1).

### 1.5 Data-flow diagram

```
        ┌── network: tx gossip ──┐
        ▼                        │
   txgossip::Set (mempool)       │
        │ TransactionsByPriority │
        ▼                        │
  ┌─────────────┐  BuildBlock    │      Snowman engine (06)
  │ blockBuilder │◀──────────────┼──────  Verify / Accept / SetPreference
  └─────┬───────┘                │            │
        │ worstcase::State       │            │ AcceptBlock
        │ (predict GasLimit/Fee) │            ▼
        ▼                        │     ┌──────────────────┐
   blocks::Block (eth block) ────┘     │  mark settled Σ   │ (D→M→I→X order)
        │                              │  enqueue(block)    │
        │                              └─────────┬─────────┘
        │                                        │ mpsc (FIFO, bounded)
        ▼                                        ▼
   consensusCritical map               ┌───────────────────────┐
   (hash → Block, A..S)                │ saexec::Executor task  │
                                       │  execute(b) on reth    │
                                       │  Firewood propose/commit│ (spawn_blocking, 04)
                                       │  mark_executed → E++    │
                                       └───────────┬───────────┘
                                                   ▼
                                       events: ChainHead / receipts / WaitUntilExecuted
```

---

## 2. Gas-as-time (`proxytime` + `gastime` + `gasprice`)

This is the protocol-defining accounting model. Three layers:

### 2.1 `proxytime` — time measured by a proxy unit

`proxytime::Time<D>` (Go `proxytime.Time[D Duration]`) represents an instant whose
*passage* is measured in some unit `D: ~u64` (for SAE, `D = Gas`). It stores
`seconds: u64`, `fraction: D` (with invariant `fraction < hertz`), and `hertz: D`
— the number of proxy units equivalent to **one wall second** (the "rate", `R`).
`Tick(d)` advances by `d` proxy units (carrying fraction → seconds);
`FastForwardTo` jumps forward to a `(unix, frac)` if it is in the future and
returns how far it advanced; `SetRate` rescales the fraction (rounding **up** for
monotonicity). It is `canoto`-serialized.

```rust
// ava-saevm-proxytime — newtype-parameterised, no floats, checked math.
pub trait ProxyUnit: Copy + Ord + Into<u128> { /* ~u64 */ }

#[derive(Clone, Debug)]
pub struct Time<D: ProxyUnit> {
    seconds: u64,
    fraction: D,   // invariant: fraction < hertz
    hertz: D,      // proxy units per wall-second (the rate R)
}

#[derive(Clone, Copy)]
pub struct FractionalSecond<D> { pub numerator: D, pub denominator: D }

impl<D: ProxyUnit> Time<D> {
    pub fn tick(&mut self, d: D);                        // advance by d proxy units
    pub fn rate(&self) -> D { self.hertz }
    pub fn fraction(&self) -> FractionalSecond<D>;
    pub fn fast_forward_to(&mut self, to: u64, to_frac: D) -> (u64, FractionalSecond<D>);
    pub fn set_rate(&mut self, hertz: D);                // rescales fraction, rounds UP
    pub fn compare(&self, other: &Self) -> Ordering;     // rates MAY differ
    pub fn as_time(&self) -> SystemTime;                 // for metrics only
}
```

> **Cross-multiplication compare** (`FractionalSecond::compare`) uses 128-bit
> widening (`u64::widening_mul` / `u128`) exactly like Go's `bits.Mul64` so two
> instants at different rates compare identically to Go.

### 2.2 `gastime` — the SAE gas clock (Tau-discipline newtype)

`gastime::Time` (Go `gastime.Time`) wraps a `proxytime::Time<Gas>` and adds
**ACP-176/194 dynamic-fee state**: `target: Gas` (the `T` parameter), `excess:
Gas` (the `x` variable), and a `GasPriceConfig`. The rate is pinned to
`rate = target * TargetToRate` where `TargetToRate = 2`, i.e. consuming
`target * 2` gas == one wall-second of gas-time. Constants:
`MinTarget = 1`, `MaxTarget = u64::MAX / 2`.

Two block-boundary operations (cited from `gastime/acp176.go`,
`gastime/gastime.go`):

- **`before_block(t)`** (`BeforeBlock`/`FastForwardToTime`): fast-forward the gas
  clock to be **no earlier than the block's wall timestamp** `t` (converting the
  sub-second ns to gas units at the current rate, rounding up). This is what makes
  an *idle* chain still advance its clock so price can decay.
- **`tick(used)`** during execution: advances gas-time by `used` and grows excess
  by `used·(R−T)/R` (only the over-target portion accrues excess).
- **`after_block(used, target, cfg)`** (`AfterBlock`): final `tick(used)`, then
  **rescale excess** to the new `(target, scaling)` and **re-pin the rate** to the
  new target. `excess' = excess · (newT·newScale) / (oldT·oldScale)` rounded up,
  capped at `u64::MAX` (256-bit intermediate via `U256` — `scaleExcess`).

```rust
// ava-saevm-gastime
#[derive(Clone)]
pub struct GasTime {
    inner: proxytime::Time<Gas>,
    target: Gas,
    excess: Gas,
    config: GasPriceConfig,
}

impl GasTime {
    /// `target * TargetToRate` gas == 1 wall-second.
    pub const TARGET_TO_RATE: Gas = Gas(2);
    pub const MIN_TARGET: Gas = Gas(1);
    pub const MAX_TARGET: Gas = Gas(u64::MAX / 2);

    pub fn new(at: SystemTime, target: Gas, starting_excess: Gas, c: GasPriceConfig)
        -> Result<Self, Error>;

    pub fn target(&self) -> Gas { self.target }
    pub fn excess(&self) -> Gas { self.excess }
    pub fn rate(&self)  -> Gas { self.inner.rate() }

    /// "base fee" = price of one gas unit (ACP-176 exponential).
    pub fn price(&self) -> GasPrice {
        self.config.min_price.max(calculate_price(self.excess, self.excess_scaling_factor()))
    }

    pub fn before_block(&mut self, t: SystemTime);               // fast-forward to >= t
    pub fn tick(&mut self, used: Gas);                            // advance + accrue excess
    pub fn after_block(&mut self, used: Gas, target: Gas, c: GasPriceConfig) -> Result<(), Error>;
    pub fn compare(&self, other: &Self) -> Ordering;
}
```

The **excess scaling factor** `K = TargetToExcessScaling · T` (capped to `u64`),
and `price = max(min_price, e^(x/K))` computed by the **integer** exponential
`gas::calculate_price` ported from `vms/components/gas` (`gasprice/...`). Defaults:
`TargetToExcessScaling = 87`, `MinPrice = 1`. `StaticPricing = true` pins `excess
= 0` (constant min price). `enforce_min_excess` binary-searches `excessForPrice`
to keep `excess` consistent with `min_price`.

### 2.3 The Tau discipline — making the lint impossible by construction

The Go repo forbids `time.Add(...TauSeconds)` (the `tausecondslint` CI grep): gas
is measured **in time**, and you must add a `time.Duration` (`params.Tau`), never
a raw second count. We reproduce this **structurally** so the mistake cannot
compile:

```rust
// ava-saevm-params
use std::time::Duration;

/// Tau: minimum (gas-)time between a block finishing execution and being
/// settled by a later block. C-Chain value: 5 s.
pub const TAU: Duration = Duration::from_secs(TAU_SECONDS);
pub const TAU_SECONDS: u64 = 5;

/// Lambda: minimum-gas-per-tx denominator (min charge = ceil(gas_limit/λ)).
pub const LAMBDA: u64 = 2;

/// A wall instant on the SAE timeline. Constructed only from `SystemTime`; the
/// ONLY way to shift it is by a `Duration`. There is deliberately **no**
/// `impl Add<u64>` and **no** way to add a bare second count — the Go
/// `tausecondslint` rule is enforced by the type system here.
#[derive(Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub struct BlockInstant(SystemTime);

impl BlockInstant {
    pub fn from_unix(secs: u64) -> Self { /* ... */ }
    /// Subtract a Duration (e.g. `instant.minus(params::TAU)`), saturating.
    pub fn minus(self, d: Duration) -> Self { /* saturating_sub */ }
    pub fn plus(self, d: Duration) -> Self { /* saturating_add */ }
}

// ❌ Will not compile — no Add<u64>/Sub<u64> exists, so `instant - TAU_SECONDS`
//    is a type error, exactly mirroring the Go forbidden pattern.
```

`last_to_settle` (`block_builder.go::lastToSettle`) thus reads
`block_time.minus(params::TAU)` — a `Duration` op only. The
`MaxQueueWallTime = MaxFullBlocksInClosedQueue · Tau · Lambda` constant
(`params.go`) is likewise a `Duration · u64` multiply, never a second add.

### 2.4 Derived block/queue limits (ACP-194 §)

From `worstcase/state.go` + `params`:
- **Max block gas** `Ω_B = R · Tau · Lambda` (`safeMaxBlockSize`, capped so a full
  *closed* queue fits in `u64`). With `Tau=5, Lambda=2`: `Ω_B = 10·R` gas.
- **Open-queue threshold** `MaxFullBlocksInOpenQueue = 2` (`Ω_Q = 2·Ω_B`).
- **Closed-queue cap** `MaxFullBlocksInClosedQueue = 3`.
- **Min gas charged per tx** `max(gas_used, ceil(gas_limit/Lambda))`
  (`hook::MinimumGasConsumption = ceil(txLimit / Lambda)`) — prevents
  high-limit/low-usage queue-stuffing attacks.

---

## 3. Crate layout (`ava-saevm` sub-workspace)

`ava-saevm` (per `00` §3) is an internal sub-workspace; dependency direction is
strictly downward:

```
ava-saevm-params      (Tau, Lambda, BlockInstant, queue limits — leaf)
ava-saevm-intmath     (mul_div, mul_div_ceil, ceil_div, bounded_{add,sub,mul} — leaf, no_std-able)
ava-saevm-cmputils    (test/cmp helpers; dev only)
ava-saevm-proxytime   → intmath
ava-saevm-gastime     → proxytime, intmath, ava-types(Gas/GasPrice), gas (price fn)
ava-saevm-gasprice    → gastime, blocks  (eth_gasPrice/feeHistory estimator)
ava-saevm-types       → ava-database (HeightIndex), ava-evm (reth block/header types)
ava-saevm-hook        → gastime, proxytime, intmath, params, types, ava-evm
ava-saevm-worstcase   → blocks, gastime, hook, db, params, ava-evm
ava-saevm-blocks      → gastime, proxytime, params, types, hook, adaptor, ava-evm
ava-saevm-adaptor     → ava-snow, ava-engine (block.ChainVM bridge; no SAE deps)
ava-saevm-db          → hook, ava-evm/firewood (04), ava-types
ava-saevm-exec        → blocks, gastime, hook, db, types, ava-evm/reth
ava-saevm-txgossip    → ava-network(gossip), blocks, exec, ava-evm
ava-saevm-core        → all of the above (the sae.VM) + sae-rpc
ava-saevm-cchain      → core, ava-evm, ava-secp256k1fx-equivalent (avax import/export)
ava-saevm-testutil    (dev): saetest, blockstest, txtest, escrow
```

Public surface highlights:

- **`ava-saevm-core::Vm`** — the SAE VM (everything except `Initialize`,
  provided by a *harness*; §5). Implements the generic `adaptor::ChainVm`.
- **`ava-saevm-blocks::Block`** — eth block + SAE lifecycle (§4).
- **`ava-saevm-exec::Executor`** — the async streaming engine (§6).
- **`ava-saevm-db::Tracker`** — state-DB / Firewood-revision tracker (§7).
- **`ava-saevm-hook::{Points, BlockBuilder, Op, Settled}`** — lifecycle hooks
  (§8) — the seam the C-Chain and future subnets plug into.
- **`ava-saevm-gastime::GasTime`**, **`ava-saevm-params::{TAU, LAMBDA}`** (§2).
- **`ava-saevm-cchain::Vm`** — the minimal EVM C-Chain (§8).

---

## 4. Blocks (`ava-saevm-blocks`) — format, codec, lifecycle

### 4.1 Byte-exact format

A SAE block **is** an Ethereum block. Wire encoding is **RLP of the eth block**
(`blocks/snow.go::Bytes` = `rlp.EncodeToBytes`), hashing is Keccak of the header —
**byte-for-byte identical to a geth/`libevm` block** so peers and tooling
interoperate. Field semantics that differ under SAE (all *interpretation*, not
layout):

| Header field | Standard eth | SAE meaning |
|---|---|---|
| `Root` | this block's post-state | **settled ancestor's** post-exec state root (`SettledStateRoot`) |
| `ReceiptHash`/`Bloom`/`GasUsed` | this block's exec results | results of the **newly settled ancestors** carried for them |
| `BaseFee`,`GasLimit` | this block's | **worst-case prediction** by the builder; real values emerge at execution |
| `Time` | block time | inclusion time of txs; **execution** time is the separate gas-time |

A side `hook.Settled {Height, GasUnix, GasNumerator, Excess}` is embedded via the
hook's block-extras mechanism (libevm "extras" → reth header extension in `10`),
recording the settled block's gas-clock so recovery/state-sync can rebuild it.
**`canoto`** is used for the *persisted* execution-results blob
(`blocks/execution.canoto.go`, §7), **not** for the block on the wire.

Parsing (`sae/blocks.go::ParseBlock`) RLP-decodes, checks height fits `u64`,
rejects blocks > `now + 10s` (`maxFutureBlockSeconds`), and verifies tx/uncle/
withdrawals hashes match the header. Ancestry is **not** populated at parse — only
on successful `VerifyBlock`.

### 4.2 Lifecycle state machine

`blocks::Block` (Go `blocks.Block`) tracks stage with channels + atomics:

```rust
// ava-saevm-blocks — stricter-lint crate.
pub struct Block {
    eth: reth_primitives::SealedBlock,         // the wire block (10)
    // Invariant: Some(ancestry) iff NOT yet settled (severed for GC after settle).
    ancestry: ArcSwapOption<Ancestry>,         // {parent, last_settled}
    synchronous: bool,                          // genesis / last pre-SAE block
    bounds: OnceLock<WorstCaseBounds>,          // set pre-execution
    execution: ArcSwapOption<ExecutionResults>, // Some iff executed
    interim_execution_time: ArcSwapOption<proxytime::Time<Gas>>, // monotonic, during exec
    executed: Notify,                            // fired after `execution` set
    settled:  Notify,                            // fired after `ancestry` cleared
}

#[derive(PartialEq, Eq, PartialOrd, Ord)]
pub enum LifeCycleStage { NotExecuted /*=Accepted*/, Executed, Settled }
```

Key methods (cite `blocks/execution.go`, `settlement.go`):

- `mark_executed(...)`: writes receipts + canoto execution blob to disk **first**,
  sets `execution`, advances the E pointer, fires `executed` — strict **D→M→I→X**
  ordering (§10). Once only.
- `mark_settled(last_settled_ptr)`: CAS `ancestry` to `None` (severs parent links
  → GC), updates the S pointer, fires `settled`. Once only.
- `mark_synchronous(...)`: combined exec+settle for the genesis / last pre-SAE
  block (self-settling — impossible under normal SAE rules).
- `settles()` → `Range(parent.last_settled, self.last_settled)` — disjoint per
  block, contiguous overall.
- `executed_by_gas_time()`, `post_execution_state_root()`, `receipts()`, etc.
  **block** on the `executed` notify (with a `MaxQueueWallTime` warn) then read.
- `restore_execution_artefacts(...)` / `restore_settled_block(...)`: rebuild from
  disk (recovery / `GetBlock` of a settled block).

The Go `runtime.AddCleanup` GC-leak counter (`InMemoryBlockCount`) maps to a
`Drop` impl decrementing an `AtomicI64` (test observability — §10).

---

## 5. Adaptor — exposing SAE as a Snowman `ChainVM` (cross-ref 06/07)

`ava-saevm-adaptor` (Go `adaptor/`) is a **generic bridge**: it converts a
`ChainVm<BP>` (a VM that returns *plain block-property* objects and takes the
state-changing methods on the VM itself) into the `block::ChainVm` +
`BuildBlockWithContext` + `SetPreferenceWithContext` traits Snowman expects
(`07` §"ChainVm/Block traits"). The key design choice (kept from Go): **the
block does not know about the VM** — `Verify/Accept/Reject` live on the VM
(`VerifyBlock(ctx, block_ctx, b)`, `AcceptBlock(ctx, b)`, …) and the adaptor's
wrapper `Block` forwards `snowman::Block` calls back to its owning VM.

```rust
// ava-saevm-adaptor
#[async_trait]
pub trait ChainVm<BP: BlockProperties>: ava_vm::CommonVm {
    async fn get_block(&self, id: Id) -> Result<BP, VmError>;
    async fn parse_block(&self, bytes: &[u8]) -> Result<BP, VmError>;
    async fn build_block(&self, ctx: Option<&block::Context>) -> Result<BP, VmError>;

    async fn verify_block(&self, ctx: Option<&block::Context>, b: &BP) -> Result<(), VmError>;
    async fn accept_block(&self, b: &BP) -> Result<(), VmError>;
    async fn reject_block(&self, b: &BP) -> Result<(), VmError>;

    async fn set_preference(&self, id: Id, ctx: Option<&block::Context>) -> Result<(), VmError>;
    async fn last_accepted(&self) -> Result<Id, VmError>;
    async fn get_block_id_at_height(&self, h: u64) -> Result<Id, VmError>;
}

pub trait BlockProperties: Clone {
    fn id(&self) -> Id;
    fn parent(&self) -> Id;
    fn bytes(&self) -> &[u8];
    fn height(&self) -> u64;
    fn timestamp(&self) -> SystemTime;
}

/// Wraps the generic VM into the concrete Snowman traits (07).
pub fn convert<BP, V: ChainVm<BP>>(vm: Arc<V>) -> impl ava_engine::block::ChainVm { /* Adaptor */ }
```

`ava-saevm-blocks::Block` implements `BlockProperties` (Go `blocks/snow.go`).
`ava-saevm-core::Vm` implements `ChainVm<Block>` — its `verify_block` *rebuilds*
the block from its parent + the hook builder and compares hashes (cheap, no
execution; `sae/blocks.go::VerifyBlock`); during **bootstrapping** it skips the
rebuild (peers verify by hash) and blocks on `wait_until_executed` so the engine's
accept-in-a-loop cannot outrun the executor and FATAL (`AcceptBlock` +
`verifyWhenBootstrapping`).

> **`block.Context`** (proposervm P-Chain height) threads through unchanged (06).
> SAE blocks return `ShouldVerifyWithContext = true`.

---

## 6. `saexec` — the streaming execution engine (tokio pipeline)

The Go `saexec.Executor` is a goroutine draining a buffered channel. Rust port:

```rust
// ava-saevm-exec
pub struct Executor {
    tracker: Arc<saedb::Tracker>,          // state DB + Firewood revisions (§7)
    queue_tx: mpsc::Sender<Arc<Block>>,    // bounded FIFO, cap = 2 * commit_interval
    last_executed: arc_swap::ArcSwap<Block>,
    head_events: broadcast::Sender<ChainHeadEvent>,
    receipts: DashMap<TxHash, eventual::Value<Receipt>>,
    chain_ctx: ChainContext,               // recent-headers LRU for BLOCKHASH
    config: Arc<reth_chainspec::ChainSpec>,
    hooks: Arc<dyn hook::Points>,
    shutdown: CancellationToken,
    task: JoinHandle<()>,
}

impl Executor {
    pub fn new(last_executed: Arc<Block>, /* sources, chainspec, db, hooks */) -> Result<Arc<Self>, Error>;
    pub async fn enqueue(&self, b: Arc<Block>) -> Result<(), Error>; // backpressure here
    pub fn last_executed(&self) -> Arc<Block>;
    pub fn subscribe_chain_head(&self) -> broadcast::Receiver<ChainHeadEvent>;
}
```

### 6.1 The execute step (`saexec/execution.go::Execute`)

Per dequeued block, on a **dedicated execution thread** (not the async reactor —
`00` §7.2; this thread owns the Firewood handle so calls it synchronously per
`04` §4.2):

1. Sanity: `last_executed.hash() == b.parent_hash()` (else error → fatal log).
2. Clone the parent's `executed_by_gas_time` → `gas_clock`; `before_block(block_time)`.
3. Open `state_db` at `parent.post_execution_state_root()` (Firewood `view`/
   reth `StateProvider`, §7/§8).
4. `hooks.before_executing_block(rules, state, eth)`.
5. `base_fee = gas_clock.price()`; **check it against the worst-case bound**
   (`CheckBaseFeeBound`); set `header.base_fee`.
6. For each tx: check worst-case sender balance, run `reth`/`revm`
   `ApplyTransaction` (`10`), `per_tx_clock.tick(receipt.gas_used)`,
   `set_interim_execution_time` (lets `LastToSettleAt` settle mid-block), fix up
   receipt `BlockHash`/`EffectiveGasPrice`, publish to the `eventual::Value`
   receipt buffer.
7. `hooks.end_of_block_ops` → apply mint/burn `Op`s, ticking the clock.
8. `hooks.after_executing_block`.
9. `gas_clock.after_block(block_gas_consumed, target, cfg)` (§2.2) → the final
   execution gas-time.
10. **Commit:** `state.commit()` → new root; `tracker.maybe_commit(settled_root,
    exec_root, height)` (§7); `tracker.track(root)`; then `mark_executed(...)` in
    strict **D→M→I→X** order; emit chain-head/receipt events.

Errors are classified: `Fatal` (consensus-critical, e.g. a tx *errored* rather
than reverted — points at the emergency playbook and **stops the executor**, which
is correct: continuing would corrupt the head) vs recoverable. A reverted tx is
**normal** (it still consumes gas). Determinism: the entire function is a pure
function of `(ordered block, parent state, chain config, hooks)` — no wall-clock
in any consensus output (`FinishBy.Wall` is metrics-only), no map iteration over
unsorted collections.

### 6.2 Backpressure, ordering, recovery

- **FIFO + bounded queue:** `mpsc` with capacity `2 * commit_interval`; `enqueue`
  `await`s when full → natural backpressure onto `AcceptBlock`. The *builder* also
  refuses to build when the worst-case queue exceeds `MaxFullBlocksInOpenQueue ·
  Ω_B` (`worstcase::ErrQueueFull`) — so consensus paces itself to execution.
- **Single executor task** guarantees total order == accept order; no parallel
  execution across blocks (state dependencies are serial). *Within* a block,
  txs are serial (EVM semantics).
- **Shutdown:** `CancellationToken` cancels; the task finishes the in-flight block
  then `tracker.close(last_root)` flushes the Firewood/snapshot layer.
- **Recovery** re-drives the queue from disk (§1.4); because execution is pure,
  the replayed roots match and `maybe_commit` lands on the same heights.

---

## 7. `saedb` — storage model (consensus-state vs execution-state, Firewood)

SAE keeps **two logically distinct kinds of state** (invariants doc):

- **Consensus state** (ordering): canonical hashes, head/finalized pointers,
  block bodies — the `rawdb`-style KV (our `ava-evm` rawdb-equivalent over the KV
  `Database`, `04`). Written on `AcceptBlock`.
- **Execution state** (the EVM trie): account/storage state, keyed by **state
  root**, in **Firewood** (`04` §4.2/§4.3). Written on execution-commit, **behind**
  the consensus state.
- Plus a **height-indexed `ExecutionResults`** DB (`types::ExecutionResults`
  wrapping `database::HeightIndex`) holding the per-block canoto blob
  `{gas_time, base_fee, receipt_root, post_state_root}` so executed artefacts
  survive restart independently of the trie commit cadence.

### 7.1 Tracker + Firewood revision contract

```rust
// ava-saevm-db
pub struct Tracker {
    state: Arc<ava_evm::FirewoodStateProvider>,  // Firewood Db (04)
    config: Config,                               // commit_interval, archival
    // reth-side caches/snapshot equivalent
}

impl Tracker {
    /// Increment the retain-count for a root (consensus needs it).
    pub fn track(&self, root: B256);
    /// Decrement; when zero and not on disk, the in-memory revision may drop.
    pub fn untrack(&self, root: B256);
    /// Commit policy:
    ///  - archival      → commit `execution_root` (every block)
    ///  - height%N == 0  → commit `settled_root`
    ///  - else           → nothing (pipelined; root still readable in-memory)
    pub fn maybe_commit(&self, settled_root: B256, execution_root: B256, height: u64) -> Result<(), Error>;
    pub fn state_db(&self, root: B256) -> Result<StateDb, Error>; // open at any retained revision
}
```

This is the **Firewood-pipelining contract with `04`**: `propose` yields the EVM
state root *before* `commit`, so the executor knows `post_execution_state_root`
(which a *future* block will embed once settled) without durably committing every
block. `commit` is deferred to the **commit interval** (default 4096) — or every
block when archival — and runs on the execution thread (`spawn_blocking`-free per
`04` §4.2). Retained revisions are bounded by the consensus-critical window
(LastExecuted back to LastSettled) via `track`/`untrack` ref-counting, mapping Go's
`triedb.Reference`/`Dereference` onto Firewood's `RevisionManager`. SAE has **no
reorgs**, so on close we flatten the snapshot to the last root unconditionally.

`last_height_with_execution_root_committed` (used by recovery, §1.4) reads the
head header and rounds down to the last committed interval (or returns the
settled-by height for that block), or the head if archival.

> **Deviation from `04` default (RocksDB) noted:** the *consensus* KV uses the
> standard `ava-database` backend; the *execution* trie is Firewood directly,
> exactly as `04` §4.2/§4.3 prescribe for the EVM. No new storage engine.

---

## 8. `cchain` — the minimal EVM C-Chain on `sae.VM` (reuse decision vs 10)

`ava-saevm-cchain` (Go `cchain/`, the "sae: Implement minimal C-Chain VM" commit)
is a thin VM that **composes** `sae::Vm` with the C-Chain-specific pieces:

- **`hooks` (`cchain/hooks.go`)** — the `hook::PointsG<Tx>` implementation:
  header building, gas config (`GasConfigAfter`), end-of-block mint/burn ops for
  **atomic Import/Export** of AVAX, block rebuild for verification.
- **`state` (`cchain/state`)** — cross-chain shared-memory / atomic-tx state
  (codec'd; an avalanchego `Database`).
- **`tx` (`cchain/tx`)** — Import/Export tx types, fx, codec (avalanchego linear
  codec, `03`).
- **`txpool` (`cchain/txpool`)** — pool for the *atomic* (cross-chain) txs,
  separate from the EVM `txgossip` mempool; `WaitForEvent` selects across both.
- **`api` (`cchain/api.go`)** — the `avax` JSON-RPC service (import/export),
  mounted at `/avax` alongside the SAE EVM RPC.

`Initialize` (`cchain/vm.go`) sets up genesis, builds the hooks, constructs
`sae::NewVM`, then the atomic txpool. It is the *harness* that supplies the
`Initialize` method `sae::Vm` deliberately omits (§5).

### Reuse decision (binding cross-ref to `10`)

**Decision: SAE's `cchain` reuses `ava-evm` (reth/revm + Firewood), not an
independent EVM.** Rationale and mapping:

- The Go `cchain` runs EVM execution through `libevm` `core.ApplyTransaction` /
  `core/state` / `triedb`. In Rust those become **reth/revm** + the Firewood
  `StateProvider` defined in `10`/`04`. There is exactly one EVM engine in the
  workspace (`00` §4.5). `saexec::Execute` calls `ava-evm`'s revm executor; it
  does **not** re-implement opcodes.
- The **C-Chain block-building/fee/atomic-tx semantics** that Go's `cchain` adds
  on top of `libevm` are re-expressed as: (a) SAE `hook::Points` for the SAE
  lifecycle, and (b) reth `ConfigureEvm` / custom `PayloadBuilder` inspector hooks
  where the customization is *EVM-internal* (precompiles, fee recipient) — per
  `10`'s extension model. The two coexist: SAE hooks own the *streaming/settlement*
  concerns; reth config owns the *EVM-execution* concerns.
- What is **distinct** from `10`'s C-Chain: the *consensus/ordering* layer. The
  reth-based C-Chain in `10` is the *synchronous* (current) C-Chain; SAE's
  `cchain` is the *asynchronous* (ACP-194) C-Chain. They **share the EVM engine and
  Firewood state layout** but differ entirely in the block lifecycle (synchronous
  verify-then-vote vs. order-then-stream-execute). `10` and `11` therefore share
  `ava-evm` as a dependency; `11` adds the SAE machinery around it.

> **Net:** one `revm`, one Firewood state format, two block-lifecycle drivers.
> `10` MUST expose its revm executor + Firewood `StateProvider` as reusable APIs
> (not bury them in a synchronous-only VM) so `ava-saevm-exec` can call them.
> This is recorded as a cross-spec requirement on `10`.

---

## 9. Hooks, txgossip, worst-case analysis

### 9.1 Hooks (`ava-saevm-hook`)

`hook::Points` / `PointsG<Tx>` are the **only** seam between the SAE core and a
concrete chain — they make the core EVM-policy-agnostic. The trait set (cited
`hook/hook.go`): `execution_results_db`, `gas_config_after`, `block_time`,
`settled_by`, `end_of_block_ops`, `can_execute_transaction`,
`before/after_executing_block`, and the generic `BlockBuilder<Tx>`
(`build_header`, `potential_end_of_block_ops`, `build_block`) +
`block_rebuilder_from`. `Op` (mint/burn with nonce-authorized debits + min-balance
guard) and `Settled` are the data types. Ported as object-safe `#[async_trait]`
traits behind `Arc<dyn …>` where dynamic, generic over `Tx: Transaction` for the
builder.

### 9.2 Tx gossip (`ava-saevm-txgossip`)

An EVM mempool (`txpool` over the reth pool, `10`) wrapped in avalanchego's
push/pull gossip (`05`): `Transaction` is `gossip::Gossipable` (RLP), `Set`
couples a `gossip::BloomSet` with the pool, `TransactionsByPriority` feeds the
builder, and push (100 ms)/pull (1 s) gossipers run as tokio tasks
(`sae/vm.go::NewVM` P2P section). `priority.go` orders by effective tip.

### 9.3 Worst-case analysis (`ava-saevm-worstcase`)

Because a block is built **before** its txs execute and its `BaseFee`/`GasLimit`
are *predictions*, the builder must guarantee every included tx will *still be
valid and affordable* whenever it eventually executes — under the **worst case**
that the entire queue ahead of it consumed maximum gas (pushing base fee up). Go
`worstcase.State` replays settled→parent history then the new block on a state DB,
tracking the worst-case gas clock, base fee, per-op min-balances, and gas limit
`Ω_B = R·Tau·Lambda`. A tx is includable iff it passes intrinsic validation,
EOA/nonce checks, the `can_execute_transaction` hook, and worst-case affordability
(`mulAdd(gas, fee_cap, value)` with `U256`). The resulting `WorstCaseBounds`
(`MaxBaseFee`, `LatestEndTime`, `MinOpBurnerBalances`) are attached to the block;
the executor **asserts** actual ≤/≥ bound (`CheckBaseFeeBound`,
`CheckSenderBalanceBound`) and logs (test-fatal) on violation — an early-warning
that the prediction model is wrong. Ported with `U256` (`ruint`) and checked math.

---

## 10. Invariants (from `docs/invariants.md`) — test-enforced

Each is a property/integration test (`02`). `D`=disk, `M`=memory, `I`=internal
pointer, `X`=external signal; `→` = "happens-before" (the prerequisite occurs
*before* the guarantor).

1. **Frontier ordering:** at all times `height(S) ≤ height(E) ≤ height(A)`.
2. **Stage causality:** `b∈S → b∈E → b∈A` (a block is settled only if executed,
   executed only if accepted).
3. **Persistence ordering on execute:** `mark_executed` writes receipts + canoto
   blob + head pointers to disk (D), *then* sets the `execution` cell (M), *then*
   advances `last_executed` (I), *then* fires `executed` (X). Test asserts a
   reader observing X can always read D.
4. **Persistence ordering on accept:** finalized-hash (settled) persisted **before**
   canonical-hash (accepted): `D(σ∈S) → D(b∈A)`.
5. **Settle-in-order:** `mark_settled` called on `Σ_n` in increasing height;
   `M(b_n∈S) → M(b_{n-1}∈S)`.
6. **Atomics-before-broadcast:** the internal pointer is updated before the
   `WaitUntil{Executed,Settled}` notify fires (poll sees ≥ what broadcast saw).
7. **Recovery equivalence:** a node restarted from disk reconstructs identical A/E/S
   frontiers and identical post-state roots (differential vs. pre-restart).
8. **GC of settled ancestry:** after `mark_settled`, `parent()`/`last_settled()`
   return `None`; `InMemoryBlockCount` returns to baseline (no leak).
9. **No reorg:** acceptance is final; the snapshot layer may be flattened freely.
10. **Receipt-root match:** stored `receipt_root == derive_sha(receipts)`
    (`CheckInvariants`).
11. **Determinism:** execution output independent of wall-clock and map order
    (§6.1); base-fee/gas-time are pure functions of the ordered chain.

---

## 11. Go → Rust mapping (non-obvious)

| Go | Rust |
|---|---|
| `proxytime.Time[D Duration]` (`~uint64`) | `proxytime::Time<D: ProxyUnit>` |
| `gas.Gas`, `gas.Price` | `ava_types::{Gas, GasPrice}` (`u64` newtypes) |
| `time.Add(-params.Tau)` (forbidden raw `TauSeconds`) | `BlockInstant::minus(params::TAU)` — no `Add<u64>` exists (§2.3) |
| `holiman/uint256`, `big.Int` | `ruint::U256` / `alloy_primitives::U256` |
| `atomic.Pointer[Block]` | `arc_swap::ArcSwap<Block>` / `ArcSwapOption` |
| `chan struct{}` close as broadcast | `tokio::sync::Notify` / `oneshot` |
| `event.FeedOf[T]` | `tokio::sync::broadcast` |
| `eventual.Value[*Receipt]` | a `OnceCell`/`watch`-backed `Eventual<T>` |
| `syncMap[K,V]` with on-store/on-delete | `DashMap` + explicit track/untrack calls |
| `io.Closer` `toClose` reverse order | `Vec<Box<dyn FnOnce>>` drained in reverse on shutdown (or `Drop`) |
| `runtime.AddCleanup` leak counter | `Drop` + `AtomicI64` (test only) |
| `canoto` codec | `ava-codec`-derived canoto-compatible blob (persisted only) |
| `rlp.EncodeToBytes(ethBlock)` | reth `alloy_rlp` encode (byte-identical) |
| `core.ApplyTransaction` (libevm) | `ava-evm` revm executor (`10`) |
| `triedb`/`snapshot.Tree` | Firewood `Db` revisions + reth state caches (`04`) |
| goroutine executor + buffered chan | tokio task + bounded `mpsc` (dedicated exec thread for Firewood) |
| `context.Context` | `&CancellationToken` + deadlines (`00` §6) |

### Error model

Per crate, `thiserror` enum + `pub type Result<T>`. Sentinels preserved as
variants and matched (mirroring Go `errors.Is`): `ErrExecutionLagging`,
`ErrQueueFull`, `ErrBlockTime{UnderMinimum,BeforeParent,AfterMaximum}`,
`ErrParentHashMismatch`, `ErrHashMismatch`, `ErrBlockTooFarInFuture`,
`ErrSettled{Root,Height}Mismatch`, `ErrNotFound`/`ErrFutureBlockNotResolved`/
`ErrNonCanonicalBlock`. The executor distinguishes a **`Fatal`** variant
(stops the stream, points at the playbook) from recoverable errors. `anyhow` only
in the binary/tests (`00` §7.1).

---

## 12. Test plan (cross-ref `02`)

- **Gas-time accounting (proptest):** `proxytime`/`gastime` are pure integer math
  — property tests for monotonicity (`tick`/`set_rate`/`fast_forward_to` never go
  backwards), `compare` consistency across differing rates, `before_block`/
  `after_block` excess scaling round-trips, `price` ≥ `min_price`, and
  `excess_for_price ∘ calculate_price` inverse bounds. Differential against the Go
  `gastime`/`proxytime` (run Go as oracle, `02`).
- **Tau-discipline compile-fail test:** a `trybuild` case asserting
  `block_instant - TAU_SECONDS` (a `u64`) does **not** compile — the structural
  analogue of the Go `tausecondslint`.
- **Settlement logic:** table tests for `Range`/`Settles`/`last_to_settle_at`
  including the `known=false` (execution-lagging) path and the synchronous-block
  edge cases.
- **Execution determinism:** execute the same ordered block set on N runs / N
  thread counts → identical roots, receipts, gas-times. Map-order fuzzing of any
  intermediate collections must not change output (§6.1).
- **Recovery / restart:** build→accept→execute→settle a chain, snapshot disk, drop
  the VM, reconstruct → assert identical A/E/S frontiers and roots (invariant 7);
  fuzz the crash point (mid-execute, between commit interval and head).
- **Worst-case as property tests:** for random tx sets, assert the executor's
  actual base fee ≤ `MaxBaseFee` and sender balances ≥ `MinOpBurnerBalances`
  (the bounds must *never* be violated) and that `ErrQueueFull` paces the builder.
- **Backpressure:** flood `accept` faster than execution; assert the queue stays
  bounded and the builder refuses via `ErrQueueFull` rather than unbounded growth.
- **Adaptor/Snowman conformance (`06`/`07`):** the converted VM satisfies the
  `block::ChainVm` contract; bootstrap accept-in-a-loop blocks on execution.
- **Differential vs Go `saevm` (`02`):** drive identical block/tx sequences
  through both; assert byte-identical block hashes, state roots, receipt roots,
  base fees, settlement choices, and frontier heights at every step.
- **Invariants §10:** each numbered invariant has a dedicated assertion harness.

---

## 13. Performance notes / improvements over Go

SAE is precisely where Rust + the Firewood-pipelining contract (`04`) shine; all
gains are **observably-neutral** (same hashes/roots/ordering — §6.1, validated by
the differential suite §12):

1. **Execution fully off the consensus latency path.** This is ACP-194's whole
   premise; we keep it. Consensus votes on cheap (no-execution) verification, so
   round latency is independent of EVM throughput. Quantify in load tests as
   *blocks accepted/s decoupled from gas executed/s* up to the `Ω_Q` queue cap.
2. **Firewood `propose`/`commit` pipelining (`04` §4.2, `00` §9).** The state root
   is available at `propose` time (pre-commit), so the executor advances E and
   reports roots while `commit` lands durably on the **commit interval** in the
   background — Go already batches via `triedb` commit-interval, but Firewood's
   stacked proposals let us *overlap* hashing of block N+1 with commit of block N
   on the dedicated exec thread. Safe because commit cadence does not affect any
   consensus output, only durability/recovery start point.
3. **Parallel signature recovery / worst-case validation at build time.** Sender
   recovery for the candidate tx set and worst-case affordability checks are
   independent across txs → `rayon` batch (Go does `SenderCacher.Recover`
   async already; we widen it). Ordering of *inclusion* is unchanged.
4. **Zero-copy block bytes / receipt buffers** on the gossip + RPC paths
   (`bytes::Bytes`, `00` §9) — RLP encode once, share.
5. **Lock-free frontiers.** `arc_swap` reads for A/E/S pointers and the
   consensus-critical map (`DashMap`) replace Go's `RWMutex`-guarded `syncMap`,
   removing reader contention on hot RPC paths (`latest`/`safe` label resolution).
6. **`Notify`/`broadcast` instead of channel-close fan-out** for
   `WaitUntil{Executed,Settled}` and chain-head events — fewer allocations,
   bounded broadcast lag with explicit handling.

Each is gated behind a differential test proving identical external behaviour vs.
the Go `saevm` reference before it is enabled.
