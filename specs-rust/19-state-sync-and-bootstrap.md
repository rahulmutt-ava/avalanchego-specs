# 19 — State Sync & Bootstrap

> **Status:** Conforms to `00-overview-and-conventions.md`. Consolidates the
> bootstrap state machine and per-VM state sync — machinery that is **diffuse
> across many Go files and is historically one of the buggiest areas** of the
> node. This spec is the single place the Rust port pins down the three-phase
> sync lifecycle, the bootstrap/state-sync actors, the merkledb `sync` protocol
> work-heap, and the per-VM matrix. Cross-references `06-consensus.md` (the
> Engine/Handler/Sender boundary, benchlist/timeout, `EngineState`),
> `04-storage-and-databases.md` (`SyncDb`, range/change proofs, the work-heap),
> `07-vm-framework.md` (`StateSyncableVm`/`StateSummary`/`StateSyncMode`),
> `08-platformvm-pchain.md` & `09-avm-xchain.md` (linear-bootstrap VMs),
> `10-cchain-evm-reth.md` (EVM snap+atomic sync), `11-saevm.md` (SAE),
> `15-serialization-and-wire-formats.md` (the `sync` proto + message ops),
> `05-networking-p2p.md` (the p2p SDK `Client`), `02-testing-strategy.md`
> (differential harness), `17-*` (actor conventions).

---

## 0. Go source coverage

| Go path | What it is | Rust crate |
|---|---|---|
| `snow/engine/snowman/bootstrap/{bootstrapper,acceptor,storage,metrics,config}.go` | Bootstrap state machine | `ava-engine::bootstrap` |
| `snow/engine/snowman/bootstrap/interval/{tree,blocks,state,interval}.go` | Fetched-block interval tree (resume after restart) | `ava-engine::bootstrap::interval` |
| `snow/consensus/snowman/bootstrapper/{poll,sampler,minority,majority}.go` | Frontier minority/majority polls + beacon sampling | `ava-engine::bootstrap::poll` |
| `snow/engine/snowman/syncer/{state_syncer,config}.go` | State-sync frontier sampling + weighted summary selection | `ava-engine::syncer` |
| `snow/engine/snowman/getter/getter.go` | Server side: answers `GetAncestors`/`Get`/frontier from local VM | `ava-engine::getter` |
| `snow/engine/common/{bootstrapable,state_syncer,vm}.go`, `no_ops_handlers.go` | `BootstrapableEngine`/`StateSyncer` traits, no-op mixins | `ava-engine` |
| `snow/engine/snowman/block/{state_syncable_vm,state_summary,state_sync_mode}.go` | `StateSyncableVM`/`StateSummary`/`StateSyncMode` (cross-ref 07) | `ava-vm::block` |
| `database/merkle/sync/{syncer,workheap,db,network_server,metrics}.go` | merkledb range/change-proof sync client+server+work-heap | `ava-merkledb::sync` |
| `database/merkle/firewood/syncer/` | Firewood-backed `sync.DB` (proves protocol reuse for EVM) | `ava-merkledb::sync` (Firewood impl) |
| `proto/sync/sync.proto` | Proof request/response wire (cross-ref 15) | `ava-merkledb::sync::proto` |
| `graft/evm/sync/{engine,client,evmstate,leaf,block,code,handlers,types}` | EVM snap-style state sync (shared coreth/subnet-evm) | `ava-evm::sync` (cross-ref 10) |
| `graft/coreth/plugin/evm/atomic/sync/` | Atomic-trie sync (coreth only) via `Extender` | `ava-evm::atomic::sync` (cross-ref 10) |
| `genesis/checkpoints.go` (`GetCheckpoints`) | Pinned block IDs injected into the bootstrap frontier | `ava-genesis` |

> The merkledb sync code moved from `x/sync` to `database/merkle/sync`; the proto
> package is still `proto/pb/sync`. The Go `Syncer[R, C]` is generic over the
> range/change proof types so the *same* client drives both the SHA-256 merkledb
> and the Firewood-ethhash EVM backends — see §4 and the `SyncDb` trait in `04`.

---

## 1. The three-phase lifecycle

A chain joins the network through up to three phases, in order, each handing off
to the next. The phase is the chain's `EngineState` (06 §3; the wire enum
`State{STATE_SYNCING, BOOTSTRAPPING, NORMAL_OP}`):

```
  ┌──────────────┐  state-sync skipped / not supported / done
  │ StateSyncing │ ─────────────────────────────────────────────┐
  │  (optional)  │                                               │
  └──────┬───────┘                                               │
         │ Static: VM syncs, engine waits for StateSyncDone      │
         │ Dynamic: engine advances; VM syncs in background      │
         ▼                                                       ▼
  ┌──────────────┐  caught up to within `bootstrap` of tip   ┌────────────┐
  │ Bootstrapping │ ─────────────────────────────────────────►│  NormalOp  │
  │ (fetch+exec)  │      onFinished(lastReqID)                 │ (Snowman)  │
  └──────────────┘                                            └────────────┘
```

- **Phase 1 — State sync (optional).** Only entered if the VM implements
  `StateSyncableVm` and `state_sync_enabled()` returns `true` (controlled by the
  `state-sync-enabled` flag). The syncer finds a recent **state summary** that a
  weight-threshold of beacons agree on, hands it to the VM, and the VM downloads
  state out-of-band (merkledb proofs / EVM leaf+code requests). On
  `StateSyncSkipped`/no acceptable summary, this phase is a no-op pass-through to
  bootstrap. Reduces a multi-day genesis replay to a few hours and bounds
  bandwidth by *state size*, not chain length.
- **Phase 2 — Bootstrap.** Fetch the block ancestry from the local last-accepted
  (which after state sync is the syncable block) up to the network tip, then
  execute/accept forward. After state sync the gap is small; without it, the gap
  is the whole chain.
- **Phase 3 — Normal consensus.** The Snowman engine (06 §4.2) takes over.

**Handoffs are by callback, not by direct call:** the state syncer is constructed
with `on_done_state_syncing(ctx, last_req_id)`; the bootstrapper with
`on_finished(ctx, last_req_id)`. The `Handler`'s `EngineManager` (06 §5.2) owns
the three engines per `EngineType` and the callback chains them. `last_req_id`
threads through so request IDs never collide across phases.

> **Bug-class note.** The Go code is buggy here mostly because of (a) out-of-sync
> `requestID` handling, (b) restart/resume edge cases, (c) the
> Static/Dynamic/Skipped fork, and (d) target-root advancing mid-sync. Each is
> called out explicitly below and pinned by a proptest/differential test (§8).

---

## 2. Bootstrap state machine

### 2.1 Precise state diagram

`snow/engine/snowman/bootstrap/bootstrapper.go`. The protocol is run **possibly
multiple times** ("restart") until the number of blocks accepted per round stops
roughly halving — this is what lets a node catch up to a *moving* tip.

```
            Start(startReqID)
                  │  SetState(Bootstrapping); load lastAccepted;
                  │  rebuild interval Tree + missingBlockIDs from DB (resume)
                  ▼
        ┌───────────────────────┐  ShouldStart()==false (not enough stake)
        │ tryStartBootstrapping  │◄───── Connected/Disconnected re-checks
        └───────────┬───────────┘
                    │ ShouldStart()==true (once)
                    ▼
        ┌───────────────────────┐
        │   FRONTIER DISCOVERY   │  sample SampleK beacons (weighted);
        │  minority Poll         │  SendGetAcceptedFrontier → AcceptedFrontier
        └───────────┬───────────┘  (record opinion / GetAcceptedFrontierFailed)
                    │ minority finalized → union of frontier IDs
                    ▼
        ┌───────────────────────┐
        │   FRONTIER AGREEMENT   │  SendGetAccepted(union) to ALL beacons;
        │  majority Poll         │  keep IDs ≥ weight threshold (Accepted)
        └───────────┬───────────┘
                    │ majority finalized
        accepted==∅ │ ──────────► restart FRONTIER DISCOVERY
                    │ accepted≠∅
                    ▼
        ┌───────────────────────┐  add genesis checkpoints + accepted IDs to
        │   FETCH ANCESTRY       │  missingBlockIDs; for each: SendGetAncestors;
        │  (backward walk)       │  Ancestors → BatchedParseBlock → process();
        └───────────┬───────────┘  newly-discovered missing parent → fetch()
                    │ missingBlockIDs == ∅
                    ▼
        ┌───────────────────────┐  execute(tree) in height order:
        │   EXECUTE FORWARD      │  parse → accept (no verify) → write batch;
        │  (job queue)           │  set ConsensusContext.executing
        └───────────┬───────────┘
                    │ numToExecute < prevExecuted/2  →  restartBootstrapping
                    │ subnet not fully bootstrapped → await bootstrappingDelay (10s) → restart
                    │ done
                    ▼
              onFinished(lastReqID)  →  hand to Snowman engine
```

Key invariants ported verbatim:
- **`requestID` gating.** Every response handler (`AcceptedFrontier`, `Accepted`,
  `Ancestors`, all `*Failed`) drops messages whose `request_id != self.request_id`.
  In Rust this is a guard at the top of each handler; the `request_id` is
  monotonic `u32` (wrapping is fine — Go uses `++`).
- **Restart termination.** `tryStartExecuting` enforces
  `numToExecute < previouslyExecuted / 2` before restarting, guaranteeing
  termination even as new blocks are issued (Go: `executedStateTransitions`).
- **Ancestors are best-effort & validated.** The first block in an `Ancestors`
  reply must equal the requested ID, else the peer is penalized and refetched.
  `len > AncestorsMaxContainersReceived` is truncated. Parse failures →
  `PeerTracker.register_failure` + refetch.
- **Resume after restart.** On `Start`, the interval `Tree` and `missingBlockIDs`
  are reconstructed from the DB (`getMissingBlockIDs`), so a node that crashed
  mid-bootstrap resumes from already-fetched blocks. The tree records contiguous
  fetched height ranges (`bootstrap/interval/`).
- **Self-request as a retry timer.** If `PeerTracker::select_peer()` returns none,
  `fetch` sends the `GetAncestors` to *itself* (guaranteed to fail), using the
  message timeout as a retry mechanism until a peer reconnects (Go comment, line
  ~444). Port this exactly — it is load-bearing for isolated nodes.
- **Completion criterion.** Done when `missingBlockIDs == ∅`, not awaiting a
  timeout, not already `NormalOp`, the per-round accepted count is not still
  halving, AND the subnet (`BootstrapTracker`) reports all its chains
  bootstrapped. Otherwise wait `bootstrappingDelay` (10 s) and restart to track
  the tip.

### 2.2 Rust design — the bootstrap actor

Bootstrap is an engine implementing the 06 `Handler` trait (with no-op mixins for
the ops it ignores: `StateSummary*`, `Put`, `Query`, `Chits`, `Simplex`). It runs
inside the per-chain `ChainHandler` task (06 §5.2) — it is **not** its own task;
all state is owned by the single handler task, so no locks are held across
`.await` (overview §7.2). Outbound calls go through the `Sender` (06 §5.3); the
`TimeoutManager` synthesizes `*Failed` ops.

```rust
/// snow/engine/snowman/bootstrap/bootstrapper.go
pub struct Bootstrapper {
    cfg: BootstrapConfig,
    metrics: BootstrapMetrics,
    request_id: u32,

    started: bool,
    restarted: bool,

    minority: FrontierPoll,            // bootstrapper.Minority — sampled beacons
    majority: FrontierPoll,            // bootstrapper.Majority — all beacons, weighted

    starting_height: u64,              // lastAccepted height at phase start
    tip_height: u64,                   // max height seen
    executed_state_transitions: u64,   // for the halving termination rule
    awaiting_timeout: bool,

    // (node, req_id) <-> requested block ID
    outstanding: BiMap<Request, Id>,
    outstanding_times: HashMap<Request, Instant>,

    tree: interval::Tree,              // fetched-height intervals (resume)
    missing_block_ids: HashSet<Id>,

    peer_tracker: Arc<PeerTracker>,    // bandwidth-weighted peer selection (06)
    eta: EtaTracker,                   // progress logging only (non-consensus)

    on_finished: OnFinished,           // FnOnce(ctx, last_req_id) -> Result<()>
}

#[async_trait::async_trait]
impl AncestorsHandler for Bootstrapper {
    async fn ancestors(&mut self, node: NodeId, req: u32, blks: Vec<Bytes>) -> Result<()> {
        let Some(wanted) = self.outstanding.delete_key(&Request { node, req }) else {
            return Ok(()); // not ours — drop (Go: debug log)
        };
        let started = self.outstanding_times.remove(&Request { node, req });
        if blks.is_empty() { self.peer_tracker.register_failure(node); return self.fetch(wanted).await; }
        let blks = &blks[..blks.len().min(self.cfg.ancestors_max_received)];
        let parsed = self.cfg.vm.batched_parse_block(blks).await
            .map_err(|_| ())  // on err: penalize + refetch (below)
            .unwrap_or_default();
        if parsed.first().map(|b| b.id()) != Some(wanted) {
            self.peer_tracker.register_failure(node);
            return self.fetch(wanted).await;
        }
        // bandwidth = bytes / latency → feeds PeerTracker selection weights
        self.peer_tracker.register_response(node, bandwidth(&parsed, started));
        let ancestors = parsed[1..].iter().map(|b| (b.id(), b.clone())).collect();
        self.process(parsed[0].clone(), Some(ancestors)).await?;
        self.try_start_executing().await
    }

    async fn get_ancestors_failed(&mut self, node: NodeId, req: u32) -> Result<()> {
        if let Some(blk) = self.outstanding.delete_key(&Request { node, req }) {
            self.peer_tracker.register_failure(node);
            self.fetch(blk).await   // immediate retry to a (possibly) different peer
        } else { Ok(()) }
    }
}
```

`process` writes fetched blocks into the interval `Tree` (a DB batch) and, if it
discovered a new missing parent, schedules the next `fetch`. `try_start_executing`
runs the forward job queue via `execute(tree)`, which streams blocks in height
order through a `parse_acceptor` that calls `Block::accept` **without re-verifying**
(the `non_verifying_parser` — bootstrap trusts the accepted frontier; verification
during catch-up is a §9 perf option, gated). Map ops:

| Go | Rust |
|---|---|
| `bimap.BiMap[Request, ids.ID]` | `BiMap<Request, Id>` (`ava-utils`) |
| `interval.Tree` (btree over fetched height ranges) | `interval::Tree` over `BTreeMap<u64, Range>` |
| `parseAcceptor` / `execute` | `ParseAcceptor` + `execute(&mut tree)` streaming iterator |
| `EtaTracker` | `EtaTracker` (logging only, **not** consensus — needs no determinism) |
| `bootstrapper.Sample` (weighted beacon sample) | `validators::Sampler` (06; **R1 RNG parity**) |
| `TimeoutRegistrar.RegisterTimeout` | `TimeoutManager::register` → `Bootstrapper::timeout` op |

### 2.3 The getter (server side)

`snow/engine/snowman/getter/getter.go` answers other nodes' `GetAcceptedFrontier`
(→ local `LastAccepted`), `GetAccepted` (filter the requested IDs to those locally
accepted), `Get` (→ a single block), and `GetAncestors` (walk parents from the
requested block, bounded by `constants.MaxContainersLen`, a byte budget, and a
time budget — copy all three limits verbatim). It lives in `ava-engine::getter`
and is shared by all three phases (a bootstrapping node still serves these).

---

## 3. State-sync state machine

### 3.1 Diagram

`snow/engine/snowman/syncer/state_syncer.go`. Runs only if the VM is a
`StateSyncableVm` and enabled.

```
   Start(startReqID): SetState(StateSyncing); requestID = startReqID
        │  (waits for ShouldStart via StartupTracker, like bootstrap)
        ▼
   startup():
     - sample SampleK beacons → frontierSeeders (each weight 1)
     - targetVoters = ALL state-sync beacons
     - GetOngoingSyncStateSummary(): if present, seed it as a candidate
       (locallyAvailableSummary) so an interrupted sync can resume
     - if no seeders → onDoneStateSyncing (skip to bootstrap)
        │ SendGetStateSummaryFrontier (≤ maxOutstanding=50 at a time)
        ▼
   FRONTIER: StateSummaryFrontier / GetStateSummaryFrontierFailed
     - ParseStateSummary(bytes) → weightedSummaries[id]; track unique heights
     - when all pending seeders answered:
         frontierAlpha = frontiersTotalWeight*Alpha / beaconsTotalWeight
         if respondedWeight < frontierAlpha → restart startup()  (network flaky)
         else requestID++; SendGetAcceptedStateSummary(uniqueHeights) to ALL voters
        ▼
   VOTE: AcceptedStateSummary / GetAcceptedStateSummaryFailed
     - for each summaryID a voter accepts: weightedSummaries[id].weight += nodeWeight
     - when all voters answered:
         drop summaries with weight < Alpha
         if none left:
            votingStake < Alpha → restart startup()   (timeouts: retry)
            else → onDoneStateSyncing               (split votes: give up, bootstrap)
         else: preferred = selectSyncableStateSummary()
               mode = preferred.Accept(ctx)   ←── VM decides
        ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │ Skipped → onDoneStateSyncing (bootstrap)                          │
   │ Static  → StateSyncing=true; WAIT for VM Notify(StateSyncDone),   │
   │           then onDoneStateSyncing                                 │
   │ Dynamic → StateSyncing=true; onDoneStateSyncing NOW; VM syncs in  │
   │           background while engine bootstraps+votes                 │
   └──────────────────────────────────────────────────────────────────┘
```

### 3.2 Summary selection

`selectSyncableStateSummary`: **prefer the locally-available (ongoing) summary if
it is still among the validated set** (lets the VM resume); otherwise pick the
**highest height**. This is deliberately *not* highest-weight — among
sufficiently-supported summaries, newest wins. Port exactly.

The acceptance threshold is `Alpha` (the consensus alpha-confidence weight, from
`Parameters`), the **same** quorum weight Snowman uses — a summary must be
accepted by ≥ `Alpha` total beacon weight. The frontier phase uses a scaled
`frontierAlpha` so a partial seeder set can still proceed.

### 3.3 Rust design — the state-syncer actor

```rust
/// snow/engine/snowman/syncer/state_syncer.go
pub struct StateSyncer {
    cfg: SyncerConfig,
    request_id: u32,
    started: bool,
    state_sync_vm: Arc<dyn StateSyncableVm>,        // from 07
    on_done: OnDone,                                // FnOnce(ctx, last_req_id)

    locally_available_summary: Option<Arc<dyn StateSummary>>,
    frontier_seeders: validators::Manager,          // sampled, weight 1 each
    target_seeders: HashSet<NodeId>, pending_seeders: HashSet<NodeId>, failed_seeders: HashSet<NodeId>,
    target_voters:  HashSet<NodeId>, pending_voters:  HashSet<NodeId>, failed_voters:  HashSet<NodeId>,

    weighted_summaries: HashMap<Id, WeightedSummary>,  // {summary, weight}
    summary_heights: HashSet<u64>, unique_heights: Vec<u64>,
}

#[derive(Clone, Copy, PartialEq, Eq)]
pub enum StateSyncMode { Skipped = 1, Static = 2, Dynamic = 3 } // wire-exact (07/15)

impl StateSyncer {
    async fn on_voting_complete(&mut self, ctx: &Ctx) -> Result<()> {
        self.weighted_summaries.retain(|_, ws| ws.weight >= self.cfg.alpha);
        if self.weighted_summaries.is_empty() {
            let voting_stake = self.cfg.beacons.total_weight()? 
                - self.cfg.beacons.subset_weight(&self.failed_voters)?;
            if voting_stake < self.cfg.alpha { return self.startup(ctx).await; } // retry
            return (self.on_done)(ctx, self.request_id).await;                   // split → bootstrap
        }
        let preferred = self.select_syncable_state_summary();
        match preferred.accept(ctx).await? {
            StateSyncMode::Skipped => (self.on_done)(ctx, self.request_id).await,
            StateSyncMode::Static  => { ctx.state_syncing.set(true); Ok(()) }     // wait for Notify
            StateSyncMode::Dynamic => { ctx.state_syncing.set(true);
                                        (self.on_done)(ctx, self.request_id).await }
        }
    }
}
```

`Notify(StateSyncDone)` (VM → engine, 06 `InternalHandler::notify`) clears
`ctx.state_syncing` and (Static mode) fires `on_done`. The bootstrapper also
handles `StateSyncDone` (Dynamic mode finishes while it is already bootstrapping)
by just clearing the flag.

> **Invariant (Dynamic).** When `Accept` returns `Dynamic`, the VM guarantees
> `LastAccepted`/`GetBlock`/`ParseBlock`/`Verify` behave as if fully synced for
> the syncable block — the engine starts voting immediately. Hold the EVM impl to
> this (10 §10).

---

## 4. merkledb sync (`database/merkle/sync`)

This is the VM-internal "out-of-band" mechanism a `StateSyncableVm` uses to pull a
trie at a target root. It is a separate protocol from the engine ops above,
carried over the **p2p SDK `Client`/handler** (`network/p2p`, 05), not the
consensus message set. Payloads are `proto/sync` (15 §3.10); proof verification is
byte-exact against `04` §3.6. Both `ava-merkledb` (SHA-256, P-Chain merkle state)
and the Firewood path implement the `SyncDb` trait (04 §3.7).

### 4.1 Work-splitting algorithm

The keyspace `[Nothing, Nothing]` (whole trie) is split into ranges; each range is
a `WorkItem`. A range with `local_root == Empty` has never been downloaded → fetch
a **range proof**; a range already downloaded but whose root changed → fetch a
**change proof** from `local_root` to the target. Completing a work item:
1. If the returned `largest_handled_key` < `work.end`, the unfetched tail
   `[largest_handled_key, work.end]` is re-enqueued.
2. If the target root advanced while the item was in flight (`stale`), the
   *handled* sub-range is re-enqueued at high priority against the new root
   (forces a change-proof pass); else it goes into `processed_work`.
3. `enqueue_work` **splits a range into two halves** when there is spare capacity
   (`processing + unprocessed ≤ 2 * SimultaneousWorkLimit`) to fan out parallelism;
   otherwise it inserts whole.

`MergeInsert` coalesces adjacent ranges that share a boundary **and** the same
`local_root` (so the `processed_work` heap stays compact and `KeyspacePercent`
progress is meaningful).

**Target root may advance.** `UpdateSyncTarget(new_root)` (called by the VM when a
newer syncable block lands) moves all `processed_work` back into `unprocessed_work`
at high priority — they must be re-validated as change proofs against the new root.
This "syncing toward a moving target" is exactly the part the EVM and P-Chain both
rely on and a frequent source of Go bugs; pin it with §8 tests.

### 4.2 Rust work-heap & parallel sync driver

The Go `Syncer` uses a `sync.Cond` + bounded goroutines. In Rust the work loop is
a `tokio` task that dispatches up to `simultaneous_work_limit` concurrent
fetch+verify tasks; **proof verification (CPU-bound, independent per range) runs on
a `rayon` pool** — a safe parallelism win (verification only checks a recomputed
root against an expected root; it has no ordering effect — overview §6.1, §9).

```rust
/// database/merkle/sync/{workheap,syncer}.go
#[derive(Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub enum Priority { Low = 1, Med = 2, High = 3, Retry = 4 } // Retry highest

pub struct WorkItem {
    start: Option<Vec<u8>>,    // None = unbounded below (Maybe.Nothing)
    end:   Option<Vec<u8>>,    // None = unbounded above
    priority: Priority,
    local_root: Id,            // Empty ⇒ range proof; else change proof
    attempt: u32,
    queued_at: Instant,
}

/// Max-heap by priority + a BTree ordered by `start` for merge/split.
pub struct WorkHeap {
    inner: BinaryHeap<Reverse<HeapKey>>,        // pop highest priority
    sorted: BTreeMap<RangeStart, WorkItem>,     // None sorts smallest
    closed: bool,
}
impl WorkHeap {
    pub fn merge_insert(&mut self, item: WorkItem) { /* coalesce adjacent same-root */ }
    pub fn get_work(&mut self) -> Option<WorkItem> { /* pop highest priority */ }
    pub fn keyspace_percent(&self, root: Id) -> f64 { /* truncate keys to 8 bytes */ }
}

pub struct Syncer<D: SyncDb> {
    db: Arc<D>,
    target_root: ArcSwap<Id>,                  // UpdateSyncTarget swaps this
    empty_root: Id,
    unprocessed: Mutex<WorkHeap>,
    processed: Mutex<WorkHeap>,
    processing: AtomicUsize,
    limit: usize,                              // SimultaneousWorkLimit
    work_available: Notify,                    // replaces sync.Cond
    client: Arc<p2p::Client>,                  // 05 SDK client
    state_sync_nodes: Vec<NodeId>,            // optional pinned servers (round-robin)
    verify_pool: rayon::ThreadPool,
}

impl<D: SyncDb> Syncer<D> {
    pub async fn sync(&self, token: &CancellationToken) -> Result<()> {
        self.unprocessed.lock().insert(WorkItem::whole_keyspace(Priority::Low));
        self.work_loop(token).await?;                 // dispatch ≤ limit tasks
        let root = self.db.merkle_root()?;
        if root != *self.target_root.load() {
            return Err(Error::FinishedWithUnexpectedRoot { expected: *self.target_root.load(), got: root });
        }
        Ok(())
    }

    async fn do_work(&self, work: WorkItem, token: &CancellationToken) {
        // exponential backoff on retries: initial 10ms ×1.5 → cap 1s
        let wait = backoff(work.attempt).saturating_sub(work.queued_at.elapsed());
        tokio::select! { _ = token.cancelled() => return, _ = sleep(wait) => {} }
        let target = *self.target_root.load();
        let req = if work.local_root == Id::EMPTY {
            ProofRequest::range(target, &work.start, &work.end, KEY_LIMIT, BYTE_LIMIT)
        } else {
            ProofRequest::change(work.local_root, target, &work.start, &work.end, KEY_LIMIT, BYTE_LIMIT)
        };
        match self.send_request(req, token).await {                    // AppRequest (05)
            Ok(bytes) => {
                // verify off the reactor; commit on the (blocking) DB thread
                let next = self.verify_pool.install(|| self.db.verify_and_commit(&work, &bytes, target));
                match next {
                    Ok(largest_handled) => self.complete_work_item(work, largest_handled, target),
                    Err(_) => self.retry_work(work),                  // bump to Retry priority
                }
            }
            Err(_) => self.retry_work(work),
        }
        self.finish_work_item();
    }
}
```

Notes:
- `ArcSwap<Id>` for `target_root` replaces the Go `RWMutex` over `TargetRoot`
  (lock-free reads on the hot path; overview §9).
- `Notify` replaces `lock.Cond`; the work loop awaits `work_available` when at the
  concurrency cap or temporarily out of work but with in-flight items.
- `state_sync_nodes` round-robin (`AtomicUsize` index) when the VM pins servers;
  otherwise `client.app_request_any` (05) lets the SDK pick.
- **Server side** (`network_server.go`): answers `RangeProofRequest`/
  `ChangeProofRequest` from a historical DB revision, capped by `key_limit` and
  `bytes_limit`; the response is `≤ bytes_limit` or the client rejects it
  (`errTooManyBytes`). Both directions are byte-exact for Go interop.

Error model: `ErrFinishedWithUnexpectedRoot`, `ErrInsufficientHistory`,
`ErrNoEndRoot`, `errInvalidRangeProof`, `errInvalidChangeProof`, `errTooManyBytes`
→ `ava-merkledb::sync::Error` variants (thiserror), matched where Go uses
`errors.Is`.

---

## 5. Per-VM matrix

| VM | `StateSyncEnabled`? | Sync mechanism | What gets synced |
|---|---|---|---|
| **P-Chain** (`platformvm`, 08) | **No** — does not implement `StateSyncableVm` | Linear bootstrap only (§2) | Full block ancestry from last-accepted → tip; state rebuilt by executing every block |
| **X-Chain** (`avm`, 09) | **No** | Linear bootstrap only (§2) | Same as P-Chain (linearized Snowman chain) |
| **C-Chain** (`evm`/coreth, 10) | **Yes** (flag `state-sync-enabled`, default on; skips if < `defaultStateSyncMinBlocks`=300k) | EVM **snap-style** leaf sync **+** atomic-trie sync, then linear bootstrap of the gap | Account trie + storage tries + contract code (at the syncable root), the **atomic trie** (cross-chain shared-memory ops), 256 parent blocks (BLOCKHASH), block/metadata pointers |
| **EVM subnets** (`subnet-evm`, 10) | **Yes** | EVM snap-style leaf sync, then linear bootstrap | Account/storage tries + code + 256 parents (no atomic trie) |
| **SAEVM** (`saevm`, 11) | per-VM (under active development) | merkledb/Firewood sync (§4) where applicable, else linear bootstrap | SAE state at a syncable root (Firewood); cross-ref 11 |
| **P-Chain merkle state tooling** | — (used inside P-Chain, not via engine state-sync) | **merkledb `sync`** (§4) over `proto/sync` | merkledb trie ranges via range/change proofs |
| Wrappers: `proposervm`, `metervm`, `tracedvm`, `rpcchainvm` | Delegate | Pass-through | Forward to inner VM (06 §wrapping order; 07) |

> Today's live P/X chains **bootstrap linearly** — there is no merkledb-sync
> wired into their engine state-sync path (they don't implement `StateSyncableVm`).
> The merkledb `sync` protocol exists and is exercised by P-Chain's internal
> merkle-state tooling and is the reuse target for any future merkle-state VM. The
> only engine-level state sync on live networks is the **EVM** path.

---

## 6. EVM state sync (cross-ref 10 §10)

`graft/evm/sync` is the shared coreth/subnet-evm implementation; `ava-evm::sync`
re-derives its *behavior* on Firewood-ethhash. Steps the VM runs after
`Accept→Static/Dynamic`:

1. **Wipe snapshot data**, then persist the chosen summary to disk so a restart
   can `GetOngoingSyncStateSummary` and resume (§3.2 resume path).
2. **Sync 256 parent blocks** of the syncable block (`BlockRequest`) — needed for
   the `BLOCKHASH` opcode; backfilled into reth-db.
3. **Sync VM-specific state** — coreth's **atomic trie** via the `Extender`
   interface (key = block height + chain ID; value = codec `atomic.Requests`;
   committed every 4096 blocks). Synced over a **second Firewood-ethhash instance**
   (10 §6.4), then `ApplyToSharedMemory` from the synced cursor.
4. **Sync the EVM state** — account trie, code, storage tries:
   - `LeafRequest{NodeType, Root, Start, End}` returns sorted key/value leafs **+
     a merkle proof** for the range; the client validates the proof, retries up to
     `maxRetryAttempts`=32 from a different peer on failure, and pages with
     `Start = lastKey` until the trie is complete.
   - Account leafs are converted to `SlimRLP` and written to the snapshot; ones
     with code are queued (`code.Queue`); ones with a storage root spawn a storage
     trie task (`defaultNumThreads`=4 concurrent tries; the account worker blocks
     when 4 are in flight).
   - Leafs feed a **`StackTrie`**: because keys arrive in sorted order,
     intermediary nodes are hashed as soon as a path's children are known, so
     **non-leaf nodes are never transmitted** — the trie (and root) is reconstructed
     locally. `OnFinish` hashes the remainder → the root, which must equal the
     summary's account root.
5. **Update pointers** — verify the engine-supplied block matches the summary
   hash/height, checkpoint the chain indexer, reset in-memory + on-disk
   `BlockChain` pointers, set last-accepted, apply post-sync ops (coreth applies
   the atomic ops).
6. **Send `StateSyncDone`** on the to-engine channel → engine bootstraps the
   remaining gap (syncable block → tip).

**Mapping to reth honestly.** reth's own **staged sync / snap sync and the Engine
API are NOT used** (10 §1, §6, gap G0): Snowman owns fork choice and we need the
pre-commit state root to vote on. The Go leaf/range protocol over the p2p SDK is
reproduced **byte-exact**, served from a **Firewood historical revision**; the
client reconstructs a Firewood-ethhash trie and checks the root. So the *transport
+ proof* shape matches Go, while reth/revm supplies execution and Firewood
supplies the trie — we borrow reth's `StackTrie`-equivalent leaf-insert idea but
not its networking. The `evmstate` resume set (`state_syncer_progress.go`) maps to
a persisted in-progress-trie set restored from the snapshot on restart.

**Resume.** Persisted summary + per-trie progress (restored by iterating snapshot
keys back into a `StackTrie` and continuing from the next key). On completion the
ongoing summary is removed from disk.

---

## 7. Failure handling (cross-ref 06)

The bootstrap/state-sync actors lean on the **same** networking substrate as
normal consensus (06 §5): `TimeoutManager`, `Benchlist`, `ResourceTracker`,
`PeerTracker`.

| Failure | Bootstrap | State sync (engine) | merkledb / EVM sync |
|---|---|---|---|
| No response in time | `TimeoutManager` synthesizes `GetAncestorsFailed` → `register_failure` + immediate `fetch` retry (new peer) | `*Failed` op → mark seeder/voter failed; treat voter as "accepts nothing" | `onResponse(err)` → `retryWork` (priority→Retry), exponential backoff 10ms×1.5→1s |
| Malformed / wrong block / bad proof | penalize peer (`PeerTracker.register_failure`), refetch | `ParseStateSummary` err → drop that summary, continue | drop response, `retryWork`; up to 32 attempts (EVM) |
| Peer benched | `Benchlist` (06) excludes it from sampling; selection falls to others | same — sampling skips benched | SDK `Client` won't dial benched peers |
| Too few responses (frontier/votes) | restart `startBootstrapping` (re-sample) | `respondedWeight < frontierAlpha` or `votingStake < Alpha` → `startup()` restart | n/a (per-range retry) |
| Split votes (≥Alpha responded, none ≥Alpha) | n/a | give up state sync → `onDone` → **bootstrap** | n/a |
| Disconnected from all peers | `fetch` self-requests as a retry timer until reconnect | `StartupTracker` blocks `ShouldStart` | `app_request_any` has no target → retried |
| Target root advanced | n/a (bootstrap re-runs to track tip) | n/a | `UpdateSyncTarget` re-queues processed work at High priority |
| Crash mid-phase | resume from interval `Tree` + `missingBlockIDs` on `Start` | `GetOngoingSyncStateSummary` re-offers the summary | EVM: persisted summary + per-trie progress; merkledb: re-derive from local roots |

**Peer selection** is bandwidth-weighted (`PeerTracker`, 06): `register_response`
records `bytes/latency`, biasing future `select_peer` toward fast peers. Preserve
the weighting math (it affects which peer is asked, not correctness).

---

## 8. Test plan (cross-ref 02)

- **Bootstrap convergence (proptest).** Generate a random chain (random heights,
  random fork-then-linearize) and a set of mock peers serving randomized
  *accepted frontiers* and partial `Ancestors` (drop/duplicate/truncate). Assert
  the bootstrapper always converges to the **same** last-accepted tip and
  executes blocks in a strictly increasing height order, for any peer schedule.
- **Restart/resume (proptest).** Crash the bootstrapper at a random point; on
  restart, assert it resumes from the interval `Tree` and reaches the identical
  tip without re-fetching already-stored blocks.
- **State-sync selection (rstest table).** Cover Skipped/Static/Dynamic, the
  "ongoing summary preferred", "highest height otherwise", "split votes → skip to
  bootstrap", and "too many timeouts → restart" branches; assert the exact
  `on_done` vs wait-for-`StateSyncDone` behavior.
- **merkledb sync proof round-trip (proptest).** For a random trie, `range_proof`
  / `change_proof` produced by the server must verify against the client and, when
  committed, reproduce the **byte-exact** target root (04 golden vectors). Include
  a `UpdateSyncTarget` mid-sync that advances the root and assert the final root is
  the *new* target. Fuzz the work-heap `merge_insert`/split for range
  non-overlap + coverage invariants.
- **Differential — Rust syncs from Go and vice-versa (02 harness).**
  - merkledb: a Rust syncer pulls a trie from a Go `network_server` and the
    committed root matches; a Go syncer pulls from the Rust server. Byte-compare
    `proto/sync` request/response frames.
  - EVM: a Rust C-Chain node state-syncs from a Go coreth node (account/storage
    tries + code + atomic trie + 256 parents) and ends with identical state root,
    atomic root, and last-accepted; and the reverse.
  - End-to-end: a Rust node joins a Go local network, completes the three phases,
    and votes — indistinguishable from a Go joiner.
- **Wire parity (golden).** `proto/sync` field tags, the engine state-sync ops
  (`GetStateSummaryFrontier`…`AcceptedStateSummary`, ops 15–18 in 15), and the
  EVM `LeafsRequest`/`BlockRequest` codec are byte-frozen against Go snapshots.

---

## 9. Performance notes / improvements over Go

- **Parallel proof verification (rayon).** merkledb/Firewood proof verification is
  CPU-bound and independent per range; run it on a rayon pool off the tokio
  reactor. Safe because verification only compares a recomputed root to an expected
  root — no observable-ordering effect (overview §6.1, §9). This is the headline
  win for state sync throughput.
- **Pipelined commit via Firewood.** Decouple proof *commit* from proof *fetch*:
  fetched+verified ranges are committed on Firewood's dedicated blocking commit
  thread (`spawn_blocking`, 04 §1.2/§4.2) while the next ranges are in flight —
  mirrors the SAE streaming-commit posture (11; overview §11.1 decision 9).
- **Lock-free target root.** `ArcSwap<Id>` for the moving sync target replaces the
  Go `RWMutex`; `Notify` replaces `sync.Cond` (no spurious wakeups, structured
  cancellation via `CancellationToken`).
- **Bandwidth-weighted fan-out.** Keep Go's `PeerTracker` weighting but issue
  `GetAncestors`/leaf requests to several fast peers concurrently up to the
  outstanding-request cap (Go already pipelines; we keep the cap identical so the
  *observed* request pattern is comparable, validated by the differential harness).
- **Parallel ancestor verification (optional, gated).** Bootstrap currently
  accepts fetched blocks *without* re-verifying (trusted frontier). If a future
  hardened mode re-verifies, verification of independent fetched blocks can be
  parallelized — but only behind a flag and a differential test, since it must not
  change which blocks become accepted (overview §9).
