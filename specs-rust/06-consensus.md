# 06 — Consensus, Engines, Validators & ProposerVM

> **Status:** Conforms to `00-overview-and-conventions.md`. Covers the Snowball/
> Snowman consensus algorithms, the consensus context threaded into VMs, the
> consensus engines and their networking glue (handler/router/timeout/benchlist/
> tracker), the validator subsystem (set/manager/state + deterministic sampling),
> `proposervm` (Snowman++), and the Simplex BFT integration. Cross-references
> `03-core-primitives.md` (ids, codec, crypto, **sampler**), `05-networking-p2p.md`
> (the wire `Sender`/router boundary), and `07-vm-framework.md` (the `ChainVM`
> traits this drives). Specs 07–11 depend on the `ConsensusContext` and the
> `Engine ↔ VM ↔ Router` trait boundaries fixed here.

---

## 0. Go source coverage

| Go path | What it is | Rust crate |
|---|---|---|
| `snow/context.go` | `Context` / `ConsensusContext` carried into VMs | `ava-snow` |
| `snow/decidable.go`, `snow/choices/` | `Decidable`, `Status` | `ava-snow` |
| `snow/acceptor.go`, `snow/state.go` | `Acceptor` callback, `EngineState`/`State` enum | `ava-snow` |
| `snow/consensus/snowball/` | Slush/Snowflake/Snowball primitives, `Tree`, `Parameters` | `ava-snow` |
| `snow/consensus/snowman/` | `Topological` linear-chain consensus | `ava-snow` |
| `snow/consensus/avalanche/`, `snow/consensus/snowstorm/` | DAG consensus (vertices) — legacy/decommissioned | `ava-snow` (documented; not built) |
| `snow/consensus/simplex/parameters.go` | Simplex tunables | `ava-simplex` |
| `snow/engine/common/` | `Engine`/`Handler` traits, message ops, `Sender`, bootstrap, `Halter`, `AppError` | `ava-engine` |
| `snow/engine/snowman/` | Snowman engine: issue/poll/vote, voter, getter, bootstrap, state-sync | `ava-engine` |
| `snow/engine/avalanche/` | Avalanche DAG engine — legacy | `ava-engine` (documented; not built) |
| `snow/networking/{handler,router,sender,timeout,benchlist,tracker}` | Per-chain message handler, chain router, outbound sender, adaptive timeouts, benchlist, resource tracking | `ava-engine` (networking sub-module) |
| `snow/validators/` | `Set`/`Manager`/`State`, sampling, BLS pubkeys, connector, uptime hooks | `ava-validators` |
| `snow/uptime/` | uptime tracking/calculator | `ava-validators` (uptime module) |
| `vms/proposervm/` | Snowman++ proposer wrapper: blocks, windower, height index, options | `ava-proposervm` |
| `simplex/` | Simplex BFT consensus integration | `ava-simplex` |

> Avalanche DAG consensus (`avalanche`/`snowstorm`/vertices) is **decommissioned**
> on all live networks (X-Chain is linearized to Snowman). We document its shape
> for completeness but do **not** build it in the initial port; the wire ops for
> vertices (`PushQuery`/`Chits` on vertices) remain in the `Sender`/`Handler`
> surface only insofar as they overlap the Snowman ops.

---

## 1. Overview & the data-flow picture

A running chain is a pipeline of cooperating actors, one set per chain:

```
                 inbound p2p msg
network (05) ─────────────────────►  ChainRouter ──► per-chain Handler (mpsc queue + task)
                                          │                    │ pops msg, routes by EngineType+state
 outbound p2p msg                         │                    ▼
network (05) ◄──── OutboundSender ◄── Sender ◄──────  Engine (Bootstrapper | Snowman | StateSyncer)
                       ▲                                        │ Add/RecordPoll
              TimeoutManager (adaptive)                         ▼
              Benchlist                                Consensus  (Topological)
              ResourceTracker                                    │ Accept/Reject
                                                                 ▼
                                                       VM (07)  (ParseBlock/BuildBlock/SetPreference)
```

Per chain there is exactly one `Handler` task draining a bounded async queue.
The handler hands each message to the engine selected by `(EngineType, EngineState)`
(state-sync → bootstrapping → consensus). The Snowman `Engine` issues blocks into
the `Consensus` instance (`Topological`), samples `k` validators for polls via the
`Sender`, registers request timeouts with the `TimeoutManager`, and on finalization
calls `Block::accept`/`reject` (which drive the VM). This mirrors Go's
goroutine-per-chain `handler` + engine design exactly; we replace goroutines+chans
with tokio tasks + `mpsc`.

---

## 2. Snowball / Snowman consensus algorithm

This is the determinism-critical core. The math is integer-only; there are **no
floats** on the consensus path (the one float, `MinPercentConnectedHealthy`, is a
health-reporting heuristic, not a consensus decision).

### 2.1 Parameters (copy exact defaults)

`snow/consensus/snowball/parameters.go`. Defaults and the validity predicate must
match Go bit-for-bit (they are network-tuning values agreed across the network).

```rust
/// Mirrors `snowball.Parameters`. Integer thresholds only.
#[derive(Clone, Copy, Debug, serde::Serialize, serde::Deserialize)]
pub struct Parameters {
    /// Nodes sampled per poll.
    pub k: u32,
    /// Vote threshold to change preference (slush).
    pub alpha_preference: u32,
    /// Vote threshold to increment a confidence counter (snowflake).
    pub alpha_confidence: u32,
    /// Consecutive successful polls required to finalize.
    pub beta: u32,
    /// Target number of outstanding polls while work is processing.
    pub concurrent_repolls: u32,
    /// Soft cap used to throttle block building.
    pub optimal_processing: u32,
    /// Health: unhealthy if more than this many items are outstanding.
    pub max_outstanding_items: u32,
    /// Health: unhealthy if an item processes longer than this.
    pub max_item_processing_time: std::time::Duration,
}

pub const DEFAULT_PARAMETERS: Parameters = Parameters {
    k: 20,
    alpha_preference: 15,
    alpha_confidence: 15,
    beta: 20,
    concurrent_repolls: 4,
    optimal_processing: 10,
    max_outstanding_items: 256,
    max_item_processing_time: std::time::Duration::from_secs(30),
};

/// Safety buffer for `min_percent_connected_healthy` (health only). Range [0,1].
pub const MIN_PERCENT_CONNECTED_BUFFER: f64 = 0.2;
```

`Parameters::verify()` enforces (port the exact branch order, same `Error::ParametersInvalid`
variant for each — tests assert via `matches!`):

- `k/2 < alpha_preference`  (integer division, matching Go's `p.K/2`)
- `alpha_preference <= alpha_confidence`
- `alpha_confidence <= k`
- `0 < concurrent_repolls <= beta`
- `0 < optimal_processing`
- `0 < max_outstanding_items`
- `max_item_processing_time > 0`

> Keep the Go easter-egg branch (`alpha_confidence == 3 && alpha_preference == 28`)
> as an explicit `verify` failure for behavioral parity (it's an unreachable-ordering
> guard); a doc-comment cites the Go source. The ASCII-art message body is not load-bearing.

JSON parsing accepts a legacy `alpha` field; when present, `alpha_preference = alpha`
and `alpha_confidence = alpha`. Reproduce in `ava-config` (12).

### 2.2 Primitives: Slush → Snowflake → Snowball, unary/binary/n-nary

Go builds consensus from layered primitives. We port them as small structs in
`ava-snow::snowball`. The key insight: a *snowflake* tracks **confidence as
consecutive successful polls**, reset to zero on any unsuccessful poll; a
*snowball* (the n-nary tree) layers preference-by-popularity on top.

**Termination conditions.** Go generalizes "beta consecutive polls at
alpha_confidence" into an ascending list of `(alpha_confidence, beta)` pairs; the
default builds a single condition `[(alpha_confidence, beta)]`. Each condition has
its own confidence counter; finalization occurs when **any** counter reaches its
beta. Port verbatim:

```rust
#[derive(Clone, Copy)]
struct TerminationCondition { alpha_confidence: u32, beta: u32 }

/// Unary snowflake: deciding on one value (used for the no-conflict case).
/// Invariants: conditions ascending by alpha_confidence; beta descending;
/// confidence[i] >= confidence[i+1] (except after early finalization).
struct UnarySnowflake {
    alpha_preference: u32,
    conditions: Vec<TerminationCondition>,
    confidence: Vec<u32>, // len == conditions.len()
    finalized: bool,
}

impl UnarySnowflake {
    fn record_poll(&mut self, count: u32) {
        for (i, c) in self.conditions.iter().enumerate() {
            if count < c.alpha_confidence {
                // Did not reach this threshold ⇒ clears this and all higher counters.
                self.confidence[i..].fill(0);
                return;
            }
            self.confidence[i] += 1;
            if self.confidence[i] >= c.beta {
                self.finalized = true;
                return;
            }
        }
    }
    fn record_unsuccessful_poll(&mut self) { self.confidence.fill(0); }
    fn finalized(&self) -> bool { self.finalized }
}
```

The **n-nary snowflake** (`NnarySnowflake`) adds a `preference` (the slush layer)
and an `alpha_preference` gate:

```rust
impl NnarySnowflake {
    fn record_poll(&mut self, count: u32, choice: Id) {
        if self.finalized { return; }
        if count < self.alpha_preference {
            self.confidence.fill(0); // unsuccessful poll
            return;
        }
        if choice != self.preference {
            self.confidence.fill(0); // preference change resets confidence...
        }
        self.preference = choice; // ...slush always adopts the >=alpha_preference choice
        for (i, c) in self.conditions.iter().enumerate() {
            if count < c.alpha_confidence { self.confidence[i..].fill(0); return; }
            self.confidence[i] += 1;
            if self.confidence[i] >= c.beta { self.finalized = true; return; }
        }
    }
}
```

The **binary** variants (`BinarySnowflake`/`BinarySnowball`) decide between two
`int` choices and exist because the `Tree` (§2.3) splits on bit prefixes. Port
`binary_slush.go`, `binary_snowflake.go`, `binary_snowball.go`, `unary_snowball.go`,
`nnary_snowball.go` as direct transliterations; they are pure integer state machines
with exhaustive Go unit tests we replicate as golden tests.

### 2.3 The Snowball `Tree` (multi-choice instance)

`snowball/tree.go` is a modified Patricia (radix) tree over choice IDs (256-bit),
splitting on the first differing bit (`commonPrefix` in bits). It is the
`Consensus` instance attached to a *parent* block to decide among that block's
*children*. The `Factory` produces a `Unary` instance for the first child and
`Add` grows the tree, extending unary nodes to binary nodes at the divergence bit.
This amortizes a single network poll across the whole preferred branch.

`Consensus` trait (`snowball/consensus.go`):

```rust
pub trait Consensus: std::fmt::Display {
    fn add(&mut self, new_choice: Id);
    fn preference(&self) -> Id;
    /// Returns whether this poll was successful (changed/confirmed state).
    fn record_poll(&mut self, votes: &Bag<Id>) -> bool;
    fn record_unsuccessful_poll(&mut self);
    fn finalized(&self) -> bool;
}

pub trait Factory {
    fn new_nnary(&self, params: &Parameters, choice: Id) -> Box<dyn NnaryInstance>;
    fn new_unary(&self, params: &Parameters) -> Box<dyn UnaryInstance>;
}
```

`Bag<Id>` is the multiset of votes (`ava-utils::bag`, 03). `record_poll(bag)` on the
tree finds the bit-prefix path matching the bag's plurality choice and pushes the
vote count down to the relevant snowflake/snowball nodes.

### 2.4 Snowman = Topological (the linear-chain consensus)

`snowman/topological.go` — **the single most important file in this spec**. It
maintains a tree of processing blocks rooted at the last-accepted block, each
non-leaf parent owning a snowball `Consensus` instance to choose among its children.
A single `record_poll` over a vote bag is propagated transitively toward genesis via
a Kahn topological sort, then applied bottom-up.

Block trait (`snowman/block.go` + `snow/decidable.go`):

```rust
#[async_trait::async_trait]
pub trait Block: Send + Sync {
    fn id(&self) -> Id;
    fn parent(&self) -> Id;
    fn height(&self) -> u64;
    fn timestamp(&self) -> std::time::SystemTime;
    fn bytes(&self) -> &[u8];
    async fn verify(&self, token: &CancellationToken) -> Result<()>;
    async fn accept(&self, token: &CancellationToken) -> Result<()>;
    async fn reject(&self, token: &CancellationToken) -> Result<()>;
}
```

Consensus trait (`snowman/consensus.go`):

```rust
#[async_trait::async_trait]
pub trait SnowmanConsensus: Send {
    fn initialize(&mut self, ctx: Arc<ConsensusContext>, params: Parameters,
                  last_accepted_id: Id, last_accepted_height: u64,
                  last_accepted_time: std::time::SystemTime) -> Result<()>;
    fn num_processing(&self) -> usize;
    fn add(&mut self, blk: Arc<dyn Block>) -> Result<()>;
    fn processing(&self, id: Id) -> bool;
    fn is_preferred(&self, id: Id) -> bool;
    fn last_accepted(&self) -> (Id, u64);
    fn preference(&self) -> Id;
    fn preference_at_height(&self, height: u64) -> Option<Id>;
    async fn record_poll(&mut self, token: &CancellationToken, votes: &Bag<Id>) -> Result<()>;
    fn get_parent(&self, id: Id) -> Option<Id>;
    fn health_check(&self) -> Result<serde_json::Value>;
}
```

Internal state of `Topological` (port the field set exactly — it is the basis of
the determinism contract):

```rust
struct Topological {
    factory: Box<dyn Factory>,
    poll_number: u64,
    ctx: Arc<ConsensusContext>,
    params: Parameters,
    last_accepted_id: Id,
    last_accepted_height: u64,
    /// blockID -> node. Contains the last-accepted block (as the genesis-of-tree)
    /// plus every processing block.
    blocks: HashMap<Id, SnowmanBlock>,
    preferred_ids: HashSet<Id>,
    preferred_heights: HashMap<u64, Id>, // height -> preferred blockID
    preference: Id,                       // highest-height preferred block (the tail)
    // scratch reused across calls (kahn sort): leaves set + kahn node map
}

struct SnowmanBlock {
    blk: Option<Arc<dyn Block>>, // None for the last-accepted root
    should_falter: bool,
    sb: Option<Box<dyn Consensus>>, // snowball instance; None until a child is added
    children: HashMap<Id, Arc<dyn Block>>,
}
```

#### `add`
`snowman_block.go` + `topological.go::Add`:
1. Reject duplicate (`Error::DuplicateAdd`).
2. Parent must exist in `blocks` else `Error::UnknownParentBlock`.
3. `parent.add_child(blk)`: if the parent has no snowball instance, create
   `Tree::new(factory, params, child_id)` and the children map; else `sb.add(child_id)`.
4. Insert the new `SnowmanBlock`.
5. If `parent_id == self.preference`, extend preference: `preference = blk_id`,
   add to `preferred_ids` and `preferred_heights[height]`.

#### `record_poll` (the heart)
`topological.go::RecordPoll` — runtime `4·|live| + |votes|`, space `2·|live| + |votes|`:

1. `poll_number += 1`.
2. If `votes.len() >= alpha_preference`:
   - **`calculate_in_degree(votes)`** — Kahn sort setup. For each voted block in the
     tree (drop votes for unknown or already-decided blocks): add the vote count to
     its parent's kahn-node bag; walk ancestors toward the last-accepted block
     incrementing `in_degree`; track `leaves` (kahn nodes with no inbound edges).
   - **`push_votes()`** — pop leaves; for each leaf whose accumulated votes
     `>= alpha_preference`, push a `Votes{ parent_id, votes }` record onto a stack;
     propagate the leaf's total vote count up to its parent's kahn node, decrement
     parent in-degree, enqueue parent when it becomes a leaf.
3. **`vote(vote_stack)`** — pop the stack (bottom-up toward genesis):
   - If the stack is empty: mark the last-accepted block `should_falter = true`
     (the whole tree falters next poll); record a failed poll; return current preference.
   - For each `Votes` record, skip if its block was already rejected (`break`).
   - Honor a pending `should_falter`: call `sb.record_unsuccessful_poll()` first, clear the flag.
   - `successful |= sb.record_poll(votes)`.
   - **Accept rule:** if `sb.finalized()` **and** the block is the last-accepted block,
     accept its preferred child via `accept_preferred_child` and delete the old
     last-accepted from `blocks`. (Acceptance only ever happens for a child of the
     current last-accepted block — i.e. linear, one block at a time.)
   - Track the preferred branch: `new_preferred` follows `sb.preference()` while
     `on_preferred_branch` holds (the next stack entry equals this block's preference).
   - **Falter siblings:** every child of the voted block that is *not* the one about
     to be polled next gets `should_falter = true` (its confidence must reset before
     it can win again).
4. If `new_preferred` already ∈ `preferred_ids`, preference unchanged → return.
   Otherwise clear `preferred_ids`/`preferred_heights`, set `preference = new_preferred`,
   walk from preferred block up to last-accepted (filling the preferred set), then
   walk down following each snowball's `preference()` to the tail.

#### `accept_preferred_child`
`topological.go::acceptPreferredChild`:
- `pref = sb.preference()`; `child = children[pref]`.
- **Ordering invariant (critical):** call `ConsensusContext.block_acceptor.accept(ctx, pref, bytes)`
  **before** `child.accept()`. (The acceptor notifies indexers/atomic-memory; the VM
  block accept commits state. Order must match Go.)
- Update `last_accepted_id/height`; remove from preferred maps.
- Reject all sibling children, then `reject_transitively` their descendants (stack-based DFS).
- Note: `blocks` still contains the *newly* accepted block (it becomes the new root);
  the old root was deleted by the caller.

`HealthCheck` reports `processingBlocks`, `longestProcessingBlock`,
`avgAcceptanceTime`, `lastAcceptedID/Height`; unhealthy when processing count
> `max_outstanding_items`, oldest processing time > `max_item_processing_time`, or
average acceptance time > `15s` (`maxAcceptanceTime`). Port the same JSON shape and
the joined error.

#### Safety & liveness (the invariants the proptests guard)
- **Safety:** a block is only accepted when its parent's snowball instance is
  finalized *and* the parent is the last-accepted block. Acceptance is strictly
  linear; siblings are rejected. Two conflicting blocks (same height, different ID)
  can never both be accepted.
- **Liveness:** while connected to ≥ `alpha` honest stake voting for a branch, that
  branch's confidence climbs to `beta` and finalizes; faltering only resets
  branches that fail to reach `alpha`.

---

## 3. Consensus Context (threaded into VMs)

`snow/context.go` defines `Context` and `ConsensusContext`. In Rust we replace the
single `*snow.Context` **value bag** with an `Arc<ChainContext>` of immutable
identity/handles plus explicitly-passed dynamic state — honoring §6/§7.4 ("never
smuggle values through a context bag", "no `Lock` field"). The Go `Context.Lock`
(deprecated) is **dropped**; concurrency is structured per-actor.

```rust
/// Immutable per-chain identity + shared handles. Cheaply cloneable (`Arc`).
/// Replaces `snow.Context`. Threaded into the VM at `initialize` (07).
pub struct ChainContext {
    pub network_id: u32,
    pub subnet_id: Id,
    pub chain_id: Id,
    pub node_id: NodeId,
    /// This node's BLS public key (warp/uptime). `None` if not a staker.
    pub public_key: Option<bls::PublicKey>,
    pub network_upgrades: upgrade::Config, // ava-version (03)

    pub x_chain_id: Id,
    pub c_chain_id: Id,
    pub avax_asset_id: Id,

    /// Per-chain structured logger (tracing span; per-chain file routing).
    pub log: Logger,
    pub metrics: Arc<MultiGatherer>,         // ava metrics
    pub shared_memory: Arc<dyn SharedMemory>, // chains/atomic (07/08)
    pub bc_lookup: Arc<dyn AliaserReader>,    // chain alias resolution
    pub warp_signer: Arc<dyn warp::Signer>,   // ava-crypto / platformvm warp
    /// P-Chain validator state (Snowman++ / warp / uptime).
    pub validator_state: Arc<dyn ValidatorState>,
    /// Chain-specific scratch directory.
    pub chain_data_dir: std::path::PathBuf,
}

/// Adds consensus-runtime handles & dynamic state. Owned by the engine/handler.
pub struct ConsensusContext {
    pub chain: Arc<ChainContext>,
    pub primary_alias: String,
    pub registerer: Arc<prometheus::Registry>, // consensus metrics
    /// Acceptor callbacks fired on accept (block/tx/vertex). See §3.1.
    pub block_acceptor: Arc<dyn Acceptor>,
    pub tx_acceptor: Arc<dyn Acceptor>,
    /// Dynamic state — atomics, read concurrently.
    pub state: ArcSwap<EngineState>,    // current engine phase
    pub executing: AtomicBool,          // replaying txs during bootstrap
    pub state_syncing: AtomicBool,
}
```

> **Cross-spec decision (binding for 07–11):** VMs receive `Arc<ChainContext>` at
> `initialize`, **not** `ConsensusContext` (the VM has no business touching engine
> phase). The acceptor callbacks and dynamic phase live in `ConsensusContext`, owned
> by `ava-engine`. Anything a VM previously read off `*snow.Context` that is
> *dynamic* (e.g. "am I bootstrapping?") is passed as an explicit method argument or
> a dedicated handle, never read off a mutable shared bag.

### 3.1 Acceptor / Decidable / Status / EngineState

```rust
/// snow/acceptor.go
#[async_trait::async_trait]
pub trait Acceptor: Send + Sync {
    async fn accept(&self, ctx: &ConsensusContext, container_id: Id, bytes: &[u8]) -> Result<()>;
}

/// snow/choices/status.go — wire/persisted enum, values must match Go.
#[derive(Clone, Copy, PartialEq, Eq)]
pub enum Status { Unknown = 0, Processing = 1, Rejected = 2, Accepted = 3 }

/// snow/state.go — engine phase.
#[derive(Clone, Copy, PartialEq, Eq)]
pub enum EngineState { Initializing, StateSyncing, Bootstrapping, NormalOp }

/// EngineType selects which engine handles a message (Avalanche vs Snowman).
#[derive(Clone, Copy, PartialEq, Eq)]
pub enum EngineType { Avalanche, Snowman }
```

`Decidable` from Go collapses into the `Block` trait's `id/accept/reject`; `Status`
is queried from the VM where needed rather than carried on the block object.

---

## 4. Consensus engines

### 4.1 The message-op state machine

`snow/engine/common/engine.go` defines `Engine = Handler + Start + HealthCheck`,
where `Handler` is the union of every inbound op. We model it as one trait per op
group (object-safe, `#[async_trait]`), composed into `Handler`. Every method takes
`(node_id, request_id, ...)`; all node IDs are pre-authenticated by the network (05).

```rust
#[async_trait::async_trait]
pub trait Handler: AllGetsServer + StateSyncHandler + FrontierHandler
    + AcceptedHandler + AncestorsHandler + PutHandler + QueryHandler
    + ChitsHandler + AppHandler + InternalHandler + Send {}

#[async_trait::async_trait]
pub trait QueryHandler: Send {
    async fn pull_query(&mut self, node: NodeId, req: u32, container: Id, requested_height: u64) -> Result<()>;
    async fn push_query(&mut self, node: NodeId, req: u32, container: Bytes, requested_height: u64) -> Result<()>;
}
#[async_trait::async_trait]
pub trait ChitsHandler: Send {
    async fn chits(&mut self, node: NodeId, req: u32,
                   preferred: Id, preferred_at_height: Id, accepted: Id, accepted_height: u64) -> Result<()>;
    async fn query_failed(&mut self, node: NodeId, req: u32) -> Result<()>;
}
#[async_trait::async_trait]
pub trait PutHandler: Send {
    async fn put(&mut self, node: NodeId, req: u32, container: Bytes) -> Result<()>;
    async fn get_failed(&mut self, node: NodeId, req: u32) -> Result<()>;
}
// ... GetHandler/AncestorsHandler/AcceptedHandler/FrontierHandler/StateSync mirror Go 1:1.

#[async_trait::async_trait]
pub trait InternalHandler: Connector + Send {
    async fn gossip(&mut self) -> Result<()>;
    async fn shutdown(&mut self) -> Result<()>;
    async fn notify(&mut self, msg: VmToEngineMessage) -> Result<()>; // VM woke the engine
}

#[async_trait::async_trait]
pub trait Engine: Handler {
    async fn start(&mut self, start_req_id: u32) -> Result<()>;
    fn health_check(&self) -> Result<serde_json::Value>;
}
```

The full op set (must all exist for wire/handler parity, even when a given engine
no-ops them): `GetStateSummaryFrontier`/`StateSummaryFrontier`,
`GetAcceptedStateSummary`/`AcceptedStateSummary`, `GetAcceptedFrontier`/
`AcceptedFrontier`, `GetAccepted`/`Accepted`, `GetAncestors`/`Ancestors`,
`Get`/`Put`, `PushQuery`/`PullQuery`/`Chits`, `AppRequest`/`AppResponse`/
`AppGossip`/`AppError`, plus `*Failed` variants for every request, plus
`Connected`/`Disconnected`, `Gossip`, `Shutdown`, `Notify`, `Simplex`. Each request
op has a matching `*_failed` callback fired by the `TimeoutManager` when no response
arrives. No-op defaults come from a `NoOpHandler` mixin (mirrors
`common/no_ops_handlers.go`) so an engine that doesn't handle state-sync just logs+drops.

`AppError` (`common/error.go`) is a typed application error with an `i32` code +
message, carried on `AppRequestFailed`/`SendAppError`; predefined codes (e.g.
`ErrTimeout`, `ErrUndefined`) must match Go integer values.

### 4.2 Snowman engine (issue / poll / vote loop)

`snow/engine/snowman/engine.go` is the normal-operation engine. Its config carries
`Arc<ConsensusContext>`, the `Parameters`, the `SnowmanConsensus`, the
`block::ChainVM` (07), the `Sender`, the `validators::Manager`, and a
`ConnectedValidators` tracker. State (port the field set):

```rust
struct SnowmanEngine {
    cfg: Config,
    metrics: Metrics,
    request_id: u32,
    polls: PollSet,                              // outstanding polls (early-term)
    blk_reqs: BiMap<Request, Id>,                // outstanding Get requests <-> blkID
    pending: HashMap<Id, Arc<dyn Block>>,        // queued, waiting on ancestry
    unverified_id_to_ancestor: AncestorTree,
    unverified_block_cache: SizedLruCache<Id, Arc<dyn Block>>, // 64 MiB
    accepted_frontiers: AcceptedTracker,         // peers' accepted frontiers
    blocked: JobScheduler<Id>,                   // ops blocked on a block issuing
    pending_build_blocks: u32,
}
```

Key flows (all converge on `execute_deferred_work` which drains the `blocked`
scheduler and pumps queries):

- **Issue path** (`Put`, `PushQuery`, or VM `Notify`→build): parse the block; if it
  matches an outstanding `Get`, clear that request; `issue_from` walks the ancestry —
  blocks whose parents are unknown trigger `Sender::send_get` to the providing node;
  blocks whose ancestry is satisfied are `verify`d then `Consensus::add`ed. A
  `voter`/issuer job is parked in `blocked` until dependencies resolve.
- **Poll path** (`repoll`/`Gossip`): sample `k` validators (weighted, via the
  `Sender`'s validator sampling — §6), `request_id += 1`, register the poll in
  `polls`, register a timeout, and `Sender::send_pull_query` (or push). For block
  gossip a single **uniformly** sampled connected validator is queried for the
  next-height preference (Go uses uniform sampling here to cap bandwidth for
  high-stake nodes — preserve that).
- **Vote path** (`Chits` → `voter` job, `voter.go`): when a chit arrives, the
  `voter` is parked until the voted blocks are issued; on execution it bubbles the
  vote to the nearest processing ancestor (`get_processing_ancestor`), records the
  vote in `polls`; once a poll is complete the resulting `Bag<Id>` is fed to
  `Consensus::record_poll`, then `VM::set_preference(consensus.preference())`. If
  `num_processing() > 0` the engine repolls; else it can quiesce.
- **Respond path** (`sendChits`): always answer a query with the current
  `preferred`, `preferred_at_height(requested_height)`, `last_accepted` IDs+height,
  *before* attempting to issue the queried container.

**Early-termination polls** (`snowman/poll/`): a poll completes as soon as the
outstanding responses can no longer change the alpha-preference/alpha-confidence
outcome (the `EarlyTermFactory` is parameterized by `alpha_preference`/
`alpha_confidence`). Port the early-term predicate exactly; it affects how quickly
`record_poll` is invoked but not the final decision (safety is independent of it).

`concurrent_repolls` bounds how many polls are outstanding while work is processing;
`optimal_processing` throttles local block building. Both copied from `Parameters`.

### 4.3 Bootstrapping engine

`snow/engine/snowman/bootstrap/` + `common/bootstrapable.go`. State machine:

1. **Frontier discovery:** sample a beacon/validator set; `SendGetAcceptedFrontier`;
   collect `AcceptedFrontier` (their last-accepted block IDs).
2. **Frontier agreement:** `SendGetAccepted` with the union of frontiers; keep IDs
   that ≥ a weight threshold of peers report accepted.
3. **Fetch ancestry:** for each accepted tip, `SendGetAncestors`; peers reply
   `Ancestors` (a best-effort chain of block bytes, oldest-last). Parse, store in the
   `interval` tree (`bootstrap/interval/`), and continue requesting until the chain
   connects back to the local last-accepted block (or genesis).
4. **Execute:** once the full range is fetched, replay/verify+accept blocks in
   height order (the `acceptor.go` writes them), setting `ConsensusContext.executing`.
5. **Handoff:** when caught up and within `bootstrap` of the network tip, transition
   `EngineState::Bootstrapping → NormalOp` and hand control to the Snowman engine
   (`onFinished`). The bootstrapper is `Halter`-aware (a `CancellationToken`) so it
   can abort promptly on shutdown.

The `getter` (`snowman/getter/`) is the server side: it answers other nodes'
`GetAncestors`/`Get`/`GetAcceptedFrontier`/`GetAccepted` from the local VM, bounded
by `MaxContainersGetAncestors` / a byte/time budget (copy the limits).

### 4.4 State sync

`snow/engine/snowman/syncer/` + the `StateSyncableVM` (07). Before bootstrapping a
node may fetch a recent state summary: `GetStateSummaryFrontier` →
`GetAcceptedStateSummary` (which summaries does a weight-threshold of peers accept?)
→ hand the chosen summary to the VM's state syncer, which fetches state out-of-band
(e.g. merkle range proofs / Firewood). On completion the engine bootstraps the small
remaining gap then goes to normal op. Engines that don't support it use the no-op
state-summary handlers.

---

## 5. Networking glue (handler / router / sender / timeout / benchlist / tracker)

This is the bridge between `ava-network` (05) and the engines. Lives in
`ava-engine::networking`.

### 5.1 ChainRouter

`snow/networking/router/chain_router.go`. One process-wide router. Owns the map
`chain_id -> Handler`, matches inbound responses to outstanding requests via the
`TimeoutManager`, and on timeout synthesizes the matching `*Failed` message into the
handler. Responsibilities:
- Route every decoded inbound p2p message to the right chain `Handler` (drop if the
  chain is unknown or the sender isn't allowed via `Handler::should_handle`).
- Register each outgoing request (`(node, chain, request_id, op)`); on response or
  timeout, clear it and (on timeout) enqueue the corresponding failure op.
- Health: report unhealthy if a chain handler's queue is backed up beyond limits or
  too many requests are dropping (router `health.go`).

```rust
#[async_trait::async_trait]
pub trait Router: Send + Sync {
    fn add_chain(&self, handler: Arc<Handler>);
    async fn handle_inbound(&self, msg: InboundMessage); // from network (05)
    fn register_request(&self, node: NodeId, chain: Id, req: u32, op: Op, deadline: Instant);
    async fn connected(&self, node: NodeId, version: Arc<NodeVersion>, subnet: Id);
    async fn disconnected(&self, node: NodeId);
    async fn shutdown(&self, token: &CancellationToken);
}
```

### 5.2 Per-chain Handler = the actor

`snow/networking/handler/handler.go` + `message_queue.go`. **The canonical
goroutine→task mapping.** Each chain has:
- A bounded **async message queue** (`mpsc` channel) with two logical classes —
  *sync* messages (consensus ops processed one at a time, in order, holding the chain
  state) and *async* messages (`AppRequest`/`AppGossip`/cross-chain — processed
  concurrently on a worker pool). Go uses a single queue with a sync/async split and
  a `sync.RWMutex` over the chain; we model this as **one handler task** owning the
  consensus state, plus a `tokio` worker pool (or `spawn_blocking`/`JoinSet`) for
  async messages that don't touch consensus state.
- Message prioritization & throttling by the `ResourceTracker` (CPU/disk per peer);
  expired-deadline messages are dropped before processing (`Expiration` on the queue).
- Engine selection: by `(EngineState, EngineType)`. A message tagged for an engine
  not currently active is dropped or deferred.

```rust
pub struct ChainHandler {
    ctx: Arc<ConsensusContext>,
    validators: Arc<dyn ValidatorManager>,
    engines: EngineManager,           // {state_syncer, bootstrapper, consensus} per EngineType
    queue: mpsc::Receiver<HandlerMessage>,
    resource_tracker: Arc<dyn ResourceTracker>,
    msg_from_vm: mpsc::Receiver<VmToEngineMessage>,
    gossip_frequency: Duration,
    halt: CancellationToken,
}

impl ChainHandler {
    /// Spawned as a tokio task: drains queue + VM channel + gossip ticker
    /// via tokio::select!, dispatching to the active engine.
    pub fn start(self, recover_panic: bool) -> tokio::task::JoinHandle<()>;
}
```

`Handler::push(msg)` is the `mpsc::Sender` side. Shutdown cancels `halt`, drains,
and signals `on_stopped`. A consensus message taking longer than
`syncProcessingTimeWarnLimit` (30 s) logs a warning (port the constant).

### 5.3 OutboundSender

`snow/networking/sender/sender.go`. Implements the engine `Sender` trait by
translating each `Send*` call into wire messages (05), choosing recipients
(validator sampling / `SendConfig`), registering the request+deadline with the
`TimeoutManager`, and handing bytes to the network. The `Sender` trait the **engine**
sees (binding boundary 07 does *not* touch — only engines do):

```rust
#[async_trait::async_trait]
pub trait Sender: Send + Sync {
    // Frontier / accepted (bootstrap)
    fn send_get_accepted_frontier(&self, nodes: &HashSet<NodeId>, req: u32);
    fn send_accepted_frontier(&self, node: NodeId, req: u32, container: Id);
    fn send_get_accepted(&self, nodes: &HashSet<NodeId>, req: u32, ids: &[Id]);
    fn send_accepted(&self, node: NodeId, req: u32, ids: &[Id]);
    // Fetch
    fn send_get(&self, node: NodeId, req: u32, container: Id);
    fn send_get_ancestors(&self, node: NodeId, req: u32, container: Id);
    fn send_put(&self, node: NodeId, req: u32, container: Bytes);
    fn send_ancestors(&self, node: NodeId, req: u32, containers: Vec<Bytes>);
    // Query / vote
    fn send_push_query(&self, nodes: &HashSet<NodeId>, req: u32, container: Bytes, requested_height: u64);
    fn send_pull_query(&self, nodes: &HashSet<NodeId>, req: u32, container: Id, requested_height: u64);
    fn send_chits(&self, node: NodeId, req: u32,
                  preferred: Id, preferred_at_height: Id, accepted: Id, accepted_height: u64);
    // State sync (omitted for brevity — mirrors Go StateSummarySender)
    // App
    async fn send_app_request(&self, nodes: &HashSet<NodeId>, req: u32, bytes: Bytes) -> Result<()>;
    async fn send_app_response(&self, node: NodeId, req: u32, bytes: Bytes) -> Result<()>;
    async fn send_app_error(&self, node: NodeId, req: u32, code: i32, msg: &str) -> Result<()>;
    async fn send_app_gossip(&self, cfg: SendConfig, bytes: Bytes) -> Result<()>;
}

pub struct SendConfig {
    pub node_ids: HashSet<NodeId>,
    pub validators: usize,     // >= connected validators ⇒ all validators
    pub non_validators: usize,
    pub peers: usize,
}
```

### 5.4 Adaptive TimeoutManager

`snow/networking/timeout/manager.go` + `utils/timer/adaptive_timeout_manager.go`.
Tracks an exponentially-decaying average of observed network latency and sets the
per-request timeout to `timeout_coefficient × average`, clamped to
`[minimum_timeout, maximum_timeout]`. Config (`AdaptiveTimeoutConfig`) — port the
struct and its validity checks (`initial∈[min,max]`, `coefficient≥1`, `halflife>0`):

```rust
pub struct AdaptiveTimeoutConfig {
    pub initial_timeout: Duration,
    pub minimum_timeout: Duration,
    pub maximum_timeout: Duration,
    pub timeout_coefficient: f64, // >= 1
    pub timeout_halflife: Duration,
}
```

The averager uses a half-life decay (`utils/math::Averager`); it is **not** on the
consensus determinism path (timeouts only affect liveness/latency, not which block
is accepted), so float math is acceptable here. On register it starts a timer
(tokio `sleep` / a `DelayQueue`); on fire it calls the router's failure path and
**lengthens** the average (latency penalty); on response it records the observed
latency and **shortens** it.

### 5.5 Benchlist

`snow/networking/benchlist/`. Per chain, tracks consecutive request failures per
peer; once a peer exceeds a failure threshold within a window it is *benched* —
the `Sender` immediately fails requests to it (and skips it in sampling) for a
randomized cooldown, preventing the engine from stalling on an unresponsive
high-stake validator. Port the thresholds and the randomized duration.

### 5.6 ResourceTracker / Targeter

`snow/networking/tracker/`. Measures per-peer CPU and disk usage attributed to
processing their messages; the `Targeter` computes each peer's fair-share budget
(a base allocation + a stake-weighted bonus). The handler uses this to prioritize
and throttle the async message pool. Float math, off the consensus path.

---

## 6. Validators (`ava-validators`)

### 6.1 Set / Manager / State

`snow/validators/{set,manager,state}.go`. `Validator` carries
`{node_id, public_key: Option<bls::PublicKey>, tx_id, weight}`. `GetValidatorOutput`
is the public projection `{node_id, public_key, weight}`.

```rust
pub struct Validator { pub node_id: NodeId, pub public_key: Option<bls::PublicKey>,
                       pub tx_id: Id, pub weight: u64 }

pub struct GetValidatorOutput { pub node_id: NodeId,
                                pub public_key: Option<bls::PublicKey>, pub weight: u64 }

pub trait ValidatorManager: Send + Sync {
    fn add_staker(&self, subnet: Id, node: NodeId, pk: Option<bls::PublicKey>, tx: Id, weight: u64) -> Result<()>;
    fn add_weight(&self, subnet: Id, node: NodeId, weight: u64) -> Result<()>;
    fn remove_weight(&self, subnet: Id, node: NodeId, weight: u64) -> Result<()>;
    fn get_weight(&self, subnet: Id, node: NodeId) -> u64;
    fn get_validator(&self, subnet: Id, node: NodeId) -> Option<Validator>;
    fn get_validator_ids(&self, subnet: Id) -> Vec<NodeId>;
    fn subset_weight(&self, subnet: Id, ids: &HashSet<NodeId>) -> Result<u64>;
    fn total_weight(&self, subnet: Id) -> Result<u64>; // err on u64 overflow
    fn num_validators(&self, subnet: Id) -> usize;
    fn num_subnets(&self) -> usize;
    /// Deterministic weighted sampling WITHOUT replacement (§6.2).
    fn sample(&self, subnet: Id, size: usize) -> Result<Vec<NodeId>>;
    fn register_callback_listener(&self, subnet: Id, l: Arc<dyn ManagerCallbackListener>);
}

#[async_trait::async_trait]
pub trait ValidatorState: Send + Sync {        // snow/validators/state.go
    async fn get_minimum_height(&self) -> Result<u64>;
    async fn get_current_height(&self) -> Result<u64>;
    async fn get_subnet_id(&self, chain: Id) -> Result<Id>;
    async fn get_validator_set(&self, height: u64, subnet: Id)
        -> Result<BTreeMap<NodeId, GetValidatorOutput>>;
    async fn get_current_validator_set(&self, subnet: Id)
        -> Result<(BTreeMap<Id, GetCurrentValidatorOutput>, u64)>;
    async fn get_warp_validator_sets(&self, height: u64) -> Result<HashMap<Id, WarpSet>>;
}
```

> **Determinism (binding):** `get_validator_set` returns a `BTreeMap` (sorted by
> `NodeId`). Any place a validator set is sampled or serialized **must** iterate it
> in `NodeId` order, exactly where Go calls `utils.Sort` — never a `HashMap`
> iteration (§6.1 of `00`). The proposervm windower depends on this canonical order.

The `manager` wraps per-subnet `Set`s under a lock. Improvement candidate (§8):
`ArcSwap<Set>` for lock-free reads. `lockedState`/`cachedState` wrappers from Go
become composable `ValidatorState` adapters (an LRU cache adapter + a lock adapter).

### 6.2 Deterministic weighted sampling (must reuse 03 sampler)

`set.go` samples via `sampler.NewWeightedWithoutReplacement()` over the per-validator
weights, with validators ordered by `NodeId`. The Rust port **reuses
`ava-utils::sampler`** (03): `WeightedWithoutReplacement` for poll sampling and the
**deterministic, source-seeded** variant for the windower. The determinism contract:

- The same `(weights[], seed)` MUST produce the same index sequence as Go's
  `utils/sampler`. `03` owns this; `06` only consumes it.
- Weights array is built from the `NodeId`-sorted validator slice.
- Poll sampling (`engine` queries) uses the non-deterministic OS RNG (sampling who
  to *ask* doesn't affect the *decision*), **except** the windower path which uses a
  seeded MT (§7.3).

`Connector` (peer up/down) updates a `ConnectedValidators` tracker used for the
"sample one connected validator" gossip path and for the network-connectivity health
check (`min_percent_connected`).

### 6.3 Uptime (`snow/uptime/`)

`Manager`/`Calculator` track per-validator connected duration per subnet, persisted
via an uptime `State`. The P-Chain (08) reads uptimes to compute staking rewards and
ACP-77 continuous-fee liveness. Port `manager.go` (start/stop tracking on
connect/disconnect, `CalculateUptime`) + the `LockedCalculator` wrapper as a
`tokio::sync::Mutex`-guarded adapter. No consensus determinism concern (rewards are
computed by the P-Chain VM from these values, which are themselves agreed via
P-Chain blocks).

---

## 7. ProposerVM (Snowman++) — `ava-proposervm`

`vms/proposervm/`. A VM **middleware**: it wraps an inner `block::ChainVM` (07) and
presents itself as a `block::ChainVM` to the engine, adding soft-leader proposer
windows to throttle block production and reduce conflicts. **Block formats are on
the wire and persisted — byte-exact required.**

### 7.1 Block formats (byte-exact)

Encoded with `proposervm/block/codec.go`: `linearcodec` default, `CodecVersion = 0`,
max size = p2p message limit. Three registered types **in this exact registration
order** (the codec type-ID is the registration index — must match):

| idx | Go type | Fields (`serialize:"true"`, in order) |
|---|---|---|
| 0 | `statelessBlock` | `StatelessBlock: statelessUnsignedBlock`, `Signature: []byte` |
| 1 | `option` | `PrntID: ids.ID`, `InnerBytes: []byte` |
| 2 | `statelessGraniteBlock` | `StatelessGraniteBlock: statelessUnsignedGraniteBlock`, `Signature: []byte` |

```rust
// statelessUnsignedBlock — the signed body of a post-fork block.
struct StatelessUnsignedBlock {
    parent_id: Id,        // serialize
    timestamp: i64,       // serialize (unix seconds)
    p_chain_height: u64,  // serialize
    certificate: Vec<u8>, // serialize (X.509 staking cert DER; empty ⇒ unsigned)
    block: Vec<u8>,       // serialize (inner VM block bytes)
}
struct StatelessUnsignedGraniteBlock {       // post-Granite (ACP-181 epochs)
    stateless_block: StatelessUnsignedBlock, // serialize
    epoch: Epoch,                            // serialize
}
struct Epoch { p_chain_height: u64, number: u64, start_time: i64 } // all serialize
```

Block kinds:
- **Pre-fork block** (`pre_fork_block.go`): a thin wrapper that *is* the inner block
  bytes with no proposer header — used before the Apricot Phase 4 / proposervm fork
  height. On the wire it is just the inner block.
- **Post-fork signed block** (`statelessBlock`): `parent_id` is the proposervm
  parent; `timestamp` is the proposer's slot time; `p_chain_height` pins the
  validator set used for the windower; `certificate`+`signature` authenticate the
  proposer (or both empty ⇒ "unsigned"/anyone-can-propose block built after the max
  delay). **ID = SHA-256 of the unsigned bytes** (the serialized form minus the
  length-prefixed signature suffix). The signature is over a `Header{chain, parent,
  body}` (`header.go`, also linearcodec), **not** over the block bytes directly.
- **Post-Granite block** (`statelessGraniteBlock`): adds the ACP-181 `Epoch`. Same
  signing/ID scheme; `verify` rejects a zero `Epoch`.
- **Option block** (`option`): for oracle/option inner blocks (e.g. P-Chain
  proposal/commit/abort). `id = SHA-256(bytes)`; no signature.

Porting note: the ID computation strips the signature by length — `id =
sha256(bytes[.. len(bytes) - 4 - len(sig)])`. Reproduce exactly (the `wrappers.IntLen`
= 4-byte length prefix). Signature verification calls `staking::check_signature`
over `Header::build(chain_id, parent_id, body_id).bytes()`.

### 7.2 Fork schedule & oracle/option blocks

Three regimes selected by the inner block timestamp / height against `upgrade::Config`
(03): **pre-fork** (no header), **post-fork pre-Durango** (proposer windows from a
*start time* onward), **post-Durango** (discrete per-slot proposers), and
**post-Granite** (adds epochs). Oracle blocks (inner VM blocks that expose
`options()`) are wrapped as `option` blocks so the engine can poll between them.

### 7.3 The windower (byte-exact, determinism-critical)

`vms/proposervm/proposer/windower.go`. Decides which validator may propose at a
given height/slot. **This is consensus-affecting and MUST match Go exactly**,
including the PRNG.

Constants (copy verbatim):

```rust
pub const WINDOW_DURATION: Duration = Duration::from_secs(5);
pub const MAX_VERIFY_WINDOWS: u64 = 6;   // 30s
pub const MAX_BUILD_WINDOWS: u64 = 60;   // 5min
pub const MAX_LOOK_AHEAD_SLOTS: u64 = 720; // 1h / 5s
```

`chain_source = big-endian u64 of the first 8 bytes of chain_id` (Go:
`wrappers.Packer{Bytes: chainID[:]}.UnpackLong()`).

**Sampler setup (`make_sampler`)**: fetch the validator set at `p_chain_height`,
**drop the empty NodeID** (inactive ACP-77 validators), build `validatorData{id,
weight}` slice **sorted by NodeId** (`utils.Sort`), feed weights into a
`DeterministicWeightedWithoutReplacement` seeded from the provided MT source.

**Pre-Durango (`Proposers`/`Delay`)** uses a **32-bit MT19937**; seed
`chain_source ^ block_height`; sample `min(max_windows, total_weight)` indices →
ordered proposer list; a validator's delay is `index × WINDOW_DURATION`.

**Post-Durango (`ExpectedProposer`/`MinDelayForProposer`)** uses a **64-bit
MT19937_64**; per-slot seed = `chain_source ^ block_height ^ bit_reverse64(slot)`
(slot bit-reversed to decorrelate the height/slot state spaces), sample exactly one
index:

```rust
fn expected_proposer(vals: &[ValidatorData], source: &mut Mt19937_64,
                     sampler: &mut DeterministicWeightedWithoutReplacement,
                     block_height: u64, slot: u64) -> Result<NodeId> {
    source.seed(self.chain_source ^ block_height ^ (slot.reverse_bits()));
    let idx = sampler.sample(1).ok_or(Error::UnexpectedSamplerFailure)?[0];
    Ok(vals[idx].id)
}

pub fn time_to_slot(start: SystemTime, now: SystemTime) -> u64 {
    if now < start { 0 } else { (now.duration_since(start).unwrap().as_nanos()
        / WINDOW_DURATION.as_nanos()) as u64 }
}
```

> **CRITICAL determinism dependency.** Go uses **gonum's `mathext/prng.MT19937`
> and `MT19937_64`**, not stdlib MT. Their seeding (`Seed(uint64)`) and output
> stream must be reproduced bit-for-bit. The Rust `mersenne_twister` crate (or a
> vendored port) implements the standard MT, but **seeding conventions differ
> between MT implementations** — we MUST port gonum's exact `Seed`/`Uint64`/`Uint32`
> routines (or vendor them) and pin a golden-vector test (`02`) comparing the
> windower's `ExpectedProposer` output against captured Go outputs for a fixed
> validator set across many (height, slot) pairs. This belongs in `03` alongside the
> sampler. Do **not** assume `mersenne_twister`'s seed matches gonum's without the
> golden test passing.

### 7.4 Height index & inner-VM wiring

`height_indexed_vm.go` maintains a `height -> proposervm blockID` index so the engine
can serve `GetAncestors`/state-sync by height and so the VM can map inner heights.
The VM persists its blocks in its own DB (04), wraps the inner VM's
`BuildBlock`/`ParseBlock`/`SetPreference`/`LastAccepted`/`GetBlock`, and implements
the batched + state-syncable VM interfaces by delegating (`batched_vm.go`,
`state_syncable_vm.go`). When building, it computes the proposer slot
(`getPreDurangoSlotTime`/`getPostDurangoSlotTime`), waits for this node's slot, signs
with the staking cert, and emits the post-fork block.

---

## 8. Simplex integration (`ava-simplex`)

`simplex/` + `snow/consensus/simplex/parameters.go`. Simplex is an **alternative
single-decree-per-round BFT** consensus (a classic propose→vote→finalize protocol
with quorum certificates), offered as a pluggable consensus for subnets that want
BFT finality with a fixed validator set, instead of Snowman's metastable sampling.
High-level integration only (the initial port may stub the engine behind a feature
flag):

- **Messages** (`messages.go`, canoto-encoded `qc.canoto.go`, `block.canoto.go`):
  proposals, votes, finalization, and **quorum certificates (QCs)** aggregated via
  **BLS** (`bls.go` — reuse `ava-crypto::bls`, threshold = ⅔ stake). These flow as
  `p2p.Simplex` messages through the same router/handler, dispatched to the
  `SimplexHandler::simplex(node, msg)` op already in the `Handler` trait.
- **Block** (`block.go`): wraps an inner VM block + a Simplex header; finality is a
  QC over the block, not metastable confidence — so there is **no** Snowball
  `Parameters` (k/alpha/beta); instead the safety threshold is a BLS quorum.
- **Engine** (`engine.go`): a round-based state machine (epoch/round, leader =
  round-robin or VRF over the validator set, timeouts for view-change). It plugs in
  where the Snowman engine would, presenting the same `common::Engine`/`Handler`
  surface to the handler/router. The `comm.go` layer maps engine sends to the
  `Sender`.
- **Storage** (`storage.go`): persists finalized blocks + QCs.

Rust mapping: a tokio actor identical in shape to the Snowman engine; BLS aggregation
via `blst`. **No third-party BFT/quorum crate** — reimplement to match the Go wire
format (canoto) and BLS aggregation rules exactly. Mark it `cfg(feature = "simplex")`
in `ava-simplex`; not on by default.

---

## 9. Go → Rust mapping summary

| Go construct | Rust |
|---|---|
| goroutine-per-chain `handler` + `sync.RWMutex` over chain state | one tokio handler **task** owning consensus state; async msgs on a worker pool |
| unbuffered chans between router→handler→engine | `tokio::mpsc` (bounded); `oneshot` for request/response |
| `context.Context` (cancel) on every engine method | `&CancellationToken` arg + structured shutdown (§7.4 of 00) |
| `*snow.Context` value bag | `Arc<ChainContext>` (identity/handles) + explicit dynamic args; drop `Lock` |
| `snowball.Consensus`/`Nnary`/`Unary` interfaces | object-safe traits in `ava-snow::snowball` |
| `bag.Bag[ids.ID]` | `ava-utils::Bag<Id>` (03) |
| `set.Set[ids.NodeID]` | `HashSet<NodeId>` / `ava-utils::Set` |
| `bimap.BiMap` (blkReqs) | `ava-utils::BiMap` (03) |
| `utils.Sort` before sampling/serializing | `BTreeMap` / explicit `sort_unstable` on `NodeId` |
| gonum `prng.MT19937(_64)` | vendored/ported MT matching gonum seeding (golden-tested, in 03) |
| `AdaptiveTimeoutManager` + averager | tokio `DelayQueue` + half-life `Averager` (float ok, off consensus path) |
| `Halter` | `CancellationToken` |
| `errors.Join` (health) | collect `Vec<Error>` → `Error::Multiple` / `anyhow` in bin |

**Error model.** Each crate has a `thiserror` enum + `Result<T>` alias (§7.1 of 00).
Snowman sentinels become variants asserted via `matches!`: `DuplicateAdd`,
`UnknownParentBlock`, `TooManyProcessingBlocks`, `BlockProcessingTooLong`,
`AcceptanceTimeTooHigh`; snowball `ParametersInvalid`; windower `AnyoneCanPropose`,
`UnexpectedSamplerFailure`. A `record_poll`/`accept` returning `Err` is a **critical**
error: the engine logs and the chain halts (matching Go, which propagates the error
up and shuts the chain).

---

## 10. Test plan (see `02-testing-strategy.md`)

- **Snowball/Snowman unit golden tests:** port the entire Go test corpus
  (`*_snowflake_test.go`, `consensus_test.go`, `topological`'s `consensus_test.go`,
  `network_test.go`) as table tests; assert identical state transitions.
- **proptest — safety:** random sequences of `add`/`record_poll` over random vote
  bags; assert **no two conflicting blocks finalize** (never accept two blocks at the
  same height) and that accepted-set is always a chain. Use the Go `network_test`
  metastable model as a reference oracle.
- **proptest — liveness:** with ≥ alpha honest votes for one branch each round, the
  branch finalizes within a bounded number of polls; faltering branches don't
  livelock.
- **Sampler determinism (vs golden):** captured Go index sequences for
  `(weights, seed)` across many inputs; assert exact match (in `03`, consumed here).
- **Windower golden vectors:** captured Go `ExpectedProposer`/`Proposers`/`Delay`
  outputs for fixed validator sets across many `(height, slot, pChainHeight)`;
  assert exact `NodeId` match — this is the gonum-MT compatibility gate.
- **proposervm block round-trip vectors:** decode Go-produced block bytes (pre-fork,
  post-fork signed/unsigned, Granite, option), re-encode, assert byte-identical;
  assert IDs (sha256-of-unsigned) match Go; verify a Go-signed block's signature.
- **Engine integration (differential):** drive a Rust Snowman engine and a Go engine
  with the same scripted inbound message stream (Put/PushQuery/Chits/timeouts) over a
  mock `Sender`/`Router`; assert identical outbound messages and identical
  accept/reject sequences.
- **Network-partition / Byzantine simulations:** multi-node in-memory network
  (mirror `vms/proposervm/vm_byzantine_test.go` and router `chain_router_test.go`):
  conflicting blocks, equivocating proposers, dropped responses → safety holds,
  liveness resumes on heal.
- **Timeout/benchlist/tracker:** unit tests for adaptive timeout decay, bench/unbench
  thresholds, and per-peer resource budgeting (behavioral, not byte-exact).

---

## 11. Performance notes / improvements over Go

Each item is "safe because…" tied to §6.1/§9 of `00`.

- **Lock-free validator-set reads (`ava-validators`):** store each subnet `Set`
  behind `ArcSwap`; readers (sampling, weight lookups, windower) take a snapshot with
  no lock. *Safe because* sampling input is the `NodeId`-sorted snapshot — identical
  to Go's locked read — and writes are atomically swapped between polls.
- **Parallel block verification during bootstrap:** verify fetched ancestors whose
  parents are already verified in parallel (`JoinSet`), but **accept** strictly in
  height order. *Safe because* acceptance order (the only observable effect) is
  unchanged; verification is a pure predicate.
- **Zero-copy on the handler/sender path:** carry container bytes as `bytes::Bytes`
  from the network read through to `ParseBlock`/`SendPut`, avoiding the per-message
  copies Go makes. *Safe because* bytes are immutable.
- **`spawn_blocking` for VM accept/verify:** Firewood/RocksDB commits and EVM
  execution run off the reactor so the handler task stays responsive; the per-chain
  consensus state is still mutated by the single handler task (no added concurrency
  on the decision path). *Safe because* the ordering of `accept`/`reject` is
  preserved by the single-owner handler task.
- **Sharded async message pool:** the async (`AppRequest`/gossip) class is processed
  on a bounded `JoinSet` sized to CPU, while consensus messages stay single-threaded
  — matching Go's sync/async split but with cleaner backpressure via bounded `mpsc`.
- **Early-poll short-circuit:** the early-termination poll already exists in Go;
  ensure the Rust `PollSet` evaluates the predicate incrementally on each chit
  (O(1)) rather than rescanning. *Safe because* it only changes *when* `record_poll`
  fires, not the resulting decision.

All improvements gated behind differential tests (§10) proving identical external
behavior vs. the Go node.

Sources for crate research: [mersenne_twister (docs.rs)](https://docs.rs/mersenne_twister/),
[gonum mathext/prng](https://pkg.go.dev/gonum.org/v1/gonum/mathext/prng).
