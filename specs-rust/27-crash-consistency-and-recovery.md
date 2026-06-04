# 27 — Crash-Consistency & Recovery

> **Status:** Conforms to `00-overview-and-conventions.md`. This spec consolidates
> the crash-consistency / atomicity guarantees and per-VM restart recovery that are
> diffuse across the Go tree (and across specs `04`, `07`, `08`, `09`, `10`, `11`,
> `17`, `19`). It introduces **no new on-disk format** and **no new protocol
> surface** — it pins down the *ordering and recovery rules* the storage layer
> (`04`) and the VMs (`08`–`11`) must obey so that a crash at any point leaves a
> state a restart can reconcile to a Go-identical, consistent result. Where this
> spec states a binding ordering rule, every VM spec MUST conform.

This is fundamentally about **one invariant** — *a block Accept is all-or-nothing* —
and the recovery procedures that hold it across an unclean stop.

**Go source covered**

| Go path | Subject |
|---|---|
| `node/node.go` (`initDatabase`, `Shutdown`; `ungracefulShutdown` key) | crash marker write-on-start / clear-on-clean-stop |
| `vms/platformvm/block/executor/acceptor.go` (`apricotAtomicBlock`, `optionBlock`, `commonAccept`) | P-Chain atomic accept boundary |
| `vms/platformvm/state/state.go` (`Commit`, `CommitBatch`, `Abort`, `write*`) | P-Chain commit batch |
| `vms/avm/block/executor/block.go` (`Accept`), `vms/avm/tx.go` | X-Chain atomic accept boundary |
| `chains/atomic/shared_memory.go` (`Apply`), `chains/atomic/writer.go` (`WriteAll`) | **cross-chain shared-memory atomic commit** |
| `graft/coreth/plugin/evm/atomic/state/atomic_state.go` (`Accept`), `atomic_backend.go`, `atomic_trie.go` | C-Chain atomic-trie + shared-memory commit |
| `graft/coreth/plugin/evm/vm.go` (`ReadLastAccepted`, `initChainState`, `initializeStateSync`) | EVM accepted-block reconciliation |
| `vms/proposervm/vm.go` (`repairAcceptedChainByHeight`) | proposervm height-index repair |
| `vms/saevm/sae/recovery.go`, `saedb`, `blocks/execution.go` | SAE re-execution behind the frontier (cross-ref `11`) |
| `database/corruptabledb/db.go` | poison-on-error |
| `database/versiondb`, `x/blockdb` recovery scan | atomic commit + torn-write recovery (cross-ref `04`) |

**Rust crates touched (no new crate):** `ava-database` (the WriteBatch / versiondb
commit + corruptabledb), `ava-platformvm`/`ava-avm`/`ava-evm`/`ava-saevm` (the
per-VM accept boundary & recovery), `ava-chains` (shared-memory `Apply`/`WriteAll`),
`ava-proposervm` (height-index repair), `ava-node` (the `ungracefulShutdown`
marker), `ava-engine` (bootstrap resume — cross-ref `19`).

---

## 1. What "committed" means at each layer

Recovery reasoning depends on a precise definition of *committed* at each storage
layer. The hierarchy (cross-ref `04`):

| Layer | "Committed" means | Atomic unit | Crash-durable when |
|---|---|---|---|
| **RocksDB WriteBatch** (`04` §2.1) | `WriteBatch::write()` returned `Ok` | the whole batch | the WAL fsync (or WAL+memtable) for that batch completed; RocksDB guarantees all-or-nothing per batch. |
| **versiondb** (`04` §2.4) | `Commit()` flushed the in-mem overlay into **one** base batch and wrote it | the overlay snapshot | the underlying RocksDB batch is durable; `Abort()` discards with zero disk effect. |
| **Firewood** (`04` §4.2) | `proposal.commit()` advanced the tip and durably wrote nodes | one proposal (state-root transition) | Firewood's commit fsync completed. `propose` alone is **in-memory** — root is *computable* but not durable. |
| **reth-db (EVM chain data)** (`10`) | the block is *canonicalized* (canonical-hash + head pointers written) | the canonicalize batch | reth-db write durable. |
| **Shared memory** (`chains/atomic`) | a peer chain can `Get` the exported UTXO | merged into the *same* RocksDB batch as the producing chain's state | that single batch is durable. |

> **Binding (CC-LAYERS).** A block is "accepted-and-durable" **iff** its VM's state
> diff, its last-accepted pointer, and any shared-memory mutations it produces are
> all reachable from one durable atomic unit. The whole of §2 exists to make that
> one unit a **single batch write**.

---

## 2. The atomicity model — block Accept is all-or-nothing

### 2.1 The invariant (BINDING — CC-ATOMIC)

> **CC-ATOMIC.** On `Accept(block)`, a VM MUST persist, in **one atomic batch
> write**, the union of: (1) the block's **state diff**, (2) the **last-accepted
> pointer** (and height→ID index), and (3) any **cross-chain shared-memory
> requests** the block's txs produce. A crash either leaves *all* of these durable
> or *none* — never a partial subset. There is no intermediate fsync inside the
> sequence that another reader (or shared memory) can observe.

This is exactly what the Go acceptor does. `vms/platformvm/block/executor/
acceptor.go` for an atomic block:

1. `blkState.onAcceptState.Apply(a.state)` — fold the block's diff into the
   `versiondb`-backed state (in memory only).
2. `batch, _ := a.state.CommitBatch()` — materialize the state diff (state diff
   **+ the last-accepted pointer**, since `state.write` sets `"last accepted"`)
   into a base **batch that is not yet written**.
3. `a.ctx.SharedMemory.Apply(blkState.atomicRequests, batch)` — the shared-memory
   mutations are computed into *their own* versiondb batch and then **merged with
   `batch` and written in a single `WriteBatch::write()`** (`WriteAll`, §2.3).
4. `defer a.state.Abort()` — if anything before the write failed, the in-mem
   overlay is dropped and nothing hit disk.

The X-Chain (`vms/avm/block/executor/block.go::Accept`) and the C-Chain atomic
layer (`atomic_state.go::Accept`) follow the identical shape; the C-Chain merges
**three** batches (the snowman/metadata `commitBatch`, the atomic-trie/repo
`atomicChangesBatch`, and the shared-memory ops) into one write.

### 2.2 The Rust pattern — one WriteBatch

The Go "build a batch, then merge sibling batches, then one `Write()`" maps onto a
single `ava-database` batch whose ops are replayed together. (Recall from `04`:
`versiondb::CommitBatch()` returns the batch *without writing*; `Batch::replay`
re-applies one batch's ops onto another; `Batch::write` is the atomic boundary.)

```rust
// ava-platformvm / ava-avm accept boundary (mirrors acceptor.go + WriteAll).
// Runs on the chain's single-threaded accept path (no .await holds a DB lock — 00 §7.2).
fn accept_atomic(&self, blk: &Block) -> Result<(), Error> {
    // 1. Fold the block's diff into the versiondb-backed state (in memory).
    blk.on_accept_state.apply(&self.state)?;

    // On ANY early return below, the in-memory overlay is dropped and NOTHING
    // reached disk — the Go `defer state.Abort()`.
    let _abort = self.state.abort_guard();

    // 2. Materialize state diff + last-accepted pointer into an unwritten batch.
    //    `commit_batch()` writes "last accepted", height->ID, the UTXO/staker
    //    diffs, etc. into `state_batch` but does NOT flush it.
    let state_batch = self.state.commit_batch()?;

    // 3. Merge the shared-memory requests into the SAME batch and write once.
    //    `apply` builds the shared-memory versiondb batch, replays it AND
    //    `state_batch` onto one base WriteBatch, and calls write() exactly once.
    self.shared_memory.apply(&blk.atomic_requests, &[state_batch])?;
    //                                                ^ single atomic write happens inside

    _abort.disarm(); // success: keep the now-durable state
    Ok(())
}
```

For **Firewood-backed state** (EVM `10`, SAE `11`) the "state diff" half is a
Firewood `propose → commit` rather than KV ops, but CC-ATOMIC still holds **per
state layer** with an explicit ordering rule between layers (§2.4), because
Firewood and reth-db / the KV consensus store are *separate* durable engines and
cannot share one batch.

### 2.3 `WriteAll` / `SharedMemory::apply` — the merge primitive

`chains/atomic/writer.go::WriteAll` is the load-bearing primitive: it `Inner()`s
the base batch, `Replay`s every sibling batch onto it, then calls `Write()` **once**.
`SharedMemory::apply` (`shared_memory.go`) builds a fresh `versiondb` over the
shared-memory DB, performs the remove-from-inbound / put-to-outbound mutations
(sorted by `sharedID` to impose a deterministic lock order and avoid deadlock),
takes its `CommitBatch()`, then calls `WriteAll(smBatch, vmBatches...)`.

```rust
// ava-chains::atomic
pub fn write_all(base: &mut dyn Batch, others: &[Box<dyn Batch>]) -> Result<(), Error> {
    let base = base.inner();
    for b in others {
        b.inner().replay(base)?; // re-apply ops; no disk yet
    }
    base.write() // the ONE atomic boundary
}

impl SharedMemory {
    pub fn apply(&self, reqs: &RequestsByChain, vm_batches: &[Box<dyn Batch>]) -> Result<(), Error> {
        let vdb = VersionDb::new(self.mem.db.clone());
        let mut shared_ids: Vec<_> = reqs.keys().map(|peer| shared_id(self.this_chain, *peer)).collect();
        shared_ids.sort();                       // deterministic lock order (Go utils.Sort)
        for sid in &shared_ids { /* RemoveValue on inbound, SetValue on outbound */ }
        let mut sm_batch = vdb.commit_batch()?;  // unwritten
        write_all(sm_batch.as_mut(), vm_batches) // merge VM batch(es) + write once
    }
}
```

> **Binding (ATOMIC-1 / cross-ref `07`).** All of P/X/C use this exact merge, so a
> UTXO exported by one chain becomes visible to the importing chain **iff** the
> exporting block became durable — there is no window where the export is half-done.
> This is the same `avax.UTXO` byte contract pinned by the X↔P / X↔C differential
> tests (`07`/`08`/`09`/`10`).

### 2.4 Multi-engine ordering for Firewood-backed VMs (BINDING — CC-ORDER)

When state lives in Firewood while consensus pointers + shared memory live in the
KV store (EVM `10`, SAE `11`), CC-ATOMIC cannot be one physical batch. We fix a
**commit order** so that a crash mid-sequence is always recoverable by replay:

> **CC-ORDER.** Commit **execution/state durably first**, then write the
> consensus/last-accepted pointer (+ shared-memory) batch. Equivalently: *never*
> advance the last-accepted pointer past a state that is not yet durably committed
> (or trivially re-derivable). The recovery procedure tolerates "state committed
> but pointer not advanced" (re-do the cheap pointer write / re-execute idempotently)
> but **forbids** "pointer advanced but state missing".

- **SAE** already obeys this structurally: execution state lags the accepted
  frontier (`11` §1), and `mark_executed` writes receipts + the canoto execution
  blob (D) **before** advancing in-memory pointers (M/I) and firing notifies (X) —
  the **D→M→I→X** ordering (`11` §10, invariants 3–4). Settled (finalized) hash is
  persisted **before** the canonical (accepted) hash.
- **EVM (`10`)**: Firewood `commit` (state durable) precedes the reth-db
  canonicalize + the `snowman_accepted` last-accepted-key write; the atomic-trie
  commit + shared-memory `Apply` is its own merged batch (`atomic_state.go`).

---

## 3. Crash points & recovery matrix

The accept/execute sequence has a small set of distinguishable crash points. For
each, the table gives the on-restart recovery action and the invariant it restores.
("SM" = shared memory; "LA" = last-accepted pointer.)

| # | Crash point | On-disk state at crash | Restart recovery action | Restored invariant |
|---|---|---|---|---|
| C0 | **Before** the merged `write()` (during `Apply`/`Replay`, before any fsync) | Nothing from this block. LA still points at parent. | None needed. Engine re-offers the block (it was never accepted); VM re-runs Accept. | CC-ATOMIC: *none* of the block landed. |
| C1 | **Mid-batch** (inside RocksDB `write()` / WAL append) | RocksDB guarantees the WriteBatch is all-or-nothing; partial WAL is discarded on open. | RocksDB WAL replay on open yields either full batch or none. Treat as C0 (none) or C5 (all). | CC-ATOMIC: batch atomicity (RocksDB property). |
| C2 | **After state-commit, before SM** — *only possible for the multi-engine VMs* (EVM/SAE), where state is a separate Firewood commit (C1 is the single-batch case) | Firewood state durable; KV LA + SM **not** written. | Re-derive: LA from the KV store still = parent; VM re-executes/redoes the (idempotent) pointer + SM write for this height. State already present is reused (root match), not recomputed. | CC-ORDER: state-ahead-of-pointer is repaired by redo; never the reverse. |
| C3 | **After SM mutation visible to peer, before producer durable** | **Impossible by construction** — SM is in the *same* batch as the producer's LA (§2.3), so a peer can only read the export after the producer's write durably committed. | n/a. | Cross-chain two-sided consistency (see §3.1). |
| C4 | **After commit, before ack to engine** (write durable, process dies before returning from `Accept`) | Block fully durable; engine's consensus DB may not have recorded the decision. | proposervm height-index repair (§5.5) + engine re-offer: VM reports the *new* LA; engine reconciles. Re-accept is a no-op (idempotent: same LA already set). | Idempotent accept; consensus vs VM LA reconciled. |
| C5 | **During bootstrap / state-sync** | Partial fetched-block set / partial trie. | Resume from the interval `Tree` + `missingBlockIDs` (bootstrap) or re-offer the persisted summary (state-sync) — cross-ref `19` §7. | Bootstrap resumability (no restart from genesis). |
| C6 | **During SAE async execution** (block accepted, executor crashed mid-block, before its commit) | Accepted frontier ahead of executed frontier; execution state at last committed interval. | SAE recovery (`11` §1.4): re-enqueue every accepted-but-unexecuted canonical block from `last_committed_block`, re-execute (pure ⇒ identical roots), rebuild E then S. | Recovery equivalence (`11` invariant 7). |
| C7 | **Unclean stop, marker left set** | Any of the above + the `ungracefulShutdown` key present. | Node logs the warning and runs the extra integrity checks of §4. | Detect-and-verify rather than assume-clean. |

### 3.1 The trickiest case — shared-memory cross-chain two-sided consistency

The dangerous scenario in any cross-chain atomic op is: **chain A exports a UTXO,
chain B imports it; a crash leaves A's export half-applied or leaves the UTXO
double-spendable.** Go (and this port) make this *impossible* by never splitting the
producer's commit from the shared-memory mutation:

- A's **export** Accept merges A's state diff + A's LA + the **outbound SM put**
  into one batch (§2.3). Either A's block is durable *and* the UTXO is in shared
  memory, or neither. There is no state where A "moved on" but the UTXO is missing,
  nor where the UTXO exists but A never accepted the export.
- B's **import** Accept merges B's state diff + B's LA + the **inbound SM remove**
  (consume the UTXO) into one batch. Either B durably imported *and* the UTXO is
  consumed from shared memory, or neither — so the UTXO can never be imported twice
  across a crash.

Because both sides are single-batch atomic, the only reachable states are the four
consistent corners (neither, A-only-with-UTXO-pending, A-and-B, …); the
inconsistent corners (UTXO consumed but B not durable; A advanced but UTXO absent)
are unreachable. **This is why C3 is "impossible by construction" and is the single
most important reason CC-ATOMIC mandates *one* batch.**

> **Test focus (§7):** the crash-injection suite kills the process specifically
> between the SM `Replay` and the `write()` (the only window) and asserts the peer
> chain observes the all-or-nothing result, matching a Go node driven through the
> same sequence.

---

## 4. The `ungracefulShutdown` marker

`node/node.go` writes a sentinel **on start** (after the genesis-hash sanity check)
and **deletes it on clean shutdown** (last thing before `DB.Close()`). On the next
start, if the key is still present, the prior run died uncleanly.

```rust
// ava-node — mirrors node.go initDatabase() + Shutdown().
// Key bytes are byte-exact with Go (node/node.go:115): b"ungracefulShutdown".
const UNGRACEFUL_SHUTDOWN: &[u8] = b"ungracefulShutdown";

impl Node {
    /// Called during init, right after the genesis-hash check (04 §10.2 node-level key).
    fn mark_running(&self) -> Result<bool, Error> {
        let unclean = self.db.has(UNGRACEFUL_SHUTDOWN)?;
        if unclean {
            warn!("detected previous ungraceful shutdown"); // span/field parity (00 §7.3)
        }
        self.db.put(UNGRACEFUL_SHUTDOWN, &[])?; // nil value, exactly as Go
        Ok(unclean)
    }

    /// Step 13 of the shutdown order (17 §4.3): persistence LAST, before db.close().
    fn clear_running(&self) {
        if let Err(e) = self.db.delete(UNGRACEFUL_SHUTDOWN) {
            error!(error = %e, "failed to delete ungraceful shutdown key");
        }
    }
}
```

Ordering is load-bearing and pinned by `17` §4.3: the marker is written **after**
the DB opens and **deleted as the last step before `db.close()`**, so a crash at any
point in between leaves it set.

### 4.1 What an unclean shutdown triggers (extra integrity checks)

Go itself only logs the warning, but the marker is the signal that gates the *extra*
checks the storage/VM layers already perform. The Rust port wires the
`unclean: bool` result through to the per-VM init so that on an unclean restart we
run the stricter path (on a clean restart these are cheap/skippable):

- **merkledb `cleanShutdown` flag** (`04` §10.8): merkledb writes its own
  `metadataPrefix→"cleanShutdown"` byte; if missing/false on open, it **rebuilds
  intermediate nodes from value nodes** before serving. The node-level marker is the
  coarse signal; the merkledb flag is the fine one. Unclean ⇒ honor the rebuild.
- **blockdb torn-write scan** (`04` §5.1): if `data_file_size > indexed_size`, scan
  from `next_write_offset`, validate per-block header+checksum, rebuild the index.
  Always safe to run; an unclean stop is when it actually does work.
- **Firewood revision re-validation** (`04` §4.4 / `11` §7): on unclean restart, the
  Tracker confirms the head root matches the last committed proposal and flattens to
  the last durable revision (SAE has no reorgs).
- **proposervm height-index repair** (§5.5) always runs on init; it is the primary
  consumer of "pointer advanced but inner state behind" (C4).
- **SAE re-execution** (§5.4 / `11` §1.4) always runs; it is the C6 handler.

> The marker is intentionally a *hint that selects the conservative branch*, not a
> correctness dependency: every recovery procedure below is **idempotent and safe to
> run even after a clean shutdown**, so a lost/forged marker degrades to "extra work,
> identical result," never corruption.

---

## 5. Per-VM recovery procedures

All procedures share the shape: **read the persisted last-accepted, reconcile it
with the durable state and with the consensus engine, redo/replay any cheap
idempotent tail, then resume the sync state machine (`19`).**

### 5.1 P-Chain (`ava-platformvm`, cross-ref `08`)

Flat prefixed KV over a `versiondb` (no merkle state — `04` §10.3). State lives in
one base DB; `Accept` is the single-batch case (C1), so recovery is trivial:

1. Read `"singleton"→"last accepted"` (`04` §10.3). The state diff and LA are in the
   same committed batch, so they are always consistent (C0 or C5 only — no C2).
2. Re-open the `versiondb` overlay empty; the base DB *is* the truth.
3. Report LA to the engine; the engine reconciles (C4) and resumes bootstrap (`19`).
   No replay or repair is needed beyond the proposervm wrapper (§5.5).

### 5.2 X-Chain (`ava-avm`, cross-ref `09`)

Identical to P-Chain (flat KV, single-batch accept, `04` §10.4). Read
`"singleton"→0x02 lastAccepted`; the UTXO set / index and the LA pointer are in one
batch. The linearization (DAG→linear) bootstrap state resumes via `19`.

### 5.3 C-Chain (`ava-evm`, Firewood + reth-db reconciliation, cross-ref `10`)

Two durable engines (Firewood state, reth-db chain data) plus the avalanche-side KV
(`04` §10.5), so CC-ORDER (§2.4) governs. On boot (`graft/coreth/plugin/evm/vm.go`):

1. **`ReadLastAccepted`** — read the avalanche `snowman_accepted →
   last_accepted_key` (the EVM block hash + height).
2. **`initChainState`** — set reth-db's accepted/canonical head to that block. If
   reth-db's head is **behind** the recorded LA (crash between Firewood commit and
   reth canonicalize — C2), re-canonicalize forward to LA (state is already durable
   in Firewood; this is the idempotent redo). If reth-db is **ahead** (should not
   happen under CC-ORDER), it is a fatal inconsistency.
3. **Atomic-trie reconciliation** — `atomic_backend` re-derives the last applied
   shared-memory cursor (`atomicTrieLastAppliedToSharedMemory`, `04` §10.5) and
   re-applies any atomic ops for accepted-but-not-yet-applied heights (the
   atomic-trie commit cadence lags, like SAE execution).
4. **`initializeStateSync`** — if below the syncable threshold, resume/skip state
   sync (`19` §5/§6); else hand off to bootstrap.

> The Firewood `propose→commit` pipelining (`04` §4.2) means the *recovery start
> point* is the last committed Firewood revision, not necessarily the LA height; the
> gap (committed-root → LA) is replayed by re-executing blocks deterministically
> against the recovered state, exactly like the synchronous coreth path.

### 5.4 SAE (`ava-saevm`, re-execute from the execution frontier, cross-ref `11`)

SAE's design *is* a recovery design: execution lags acceptance, so a crash always
leaves "accepted ahead of executed." `sae/recovery.go` (`11` §1.4) on boot:

1. `last_committed_block` = highest height whose **post-execution state root was
   committed** to Firewood (`saedb::last_height_with_execution_root_committed`,
   driven by the commit interval / archival flag).
2. **Re-enqueue** every accepted-but-not-executed canonical block from there
   (`execute_all_accepted`) and `wait_until_executed` the tip → rebuilds the
   **E** frontier. Because execution is a pure function of (ordered blocks + parent
   state), the replayed roots are byte-identical (no wall-clock / map-order
   nondeterminism — `00` §6.1), so `maybe_commit` lands on the same heights.
3. Walk back from the executed tip marking the consensus-critical set settled
   (`mark_settled` where gas-time ≤ `BlockTime − Tau`) → rebuilds **S**.

This is the C6 handler. Invariant 7 (`11` §10) — *restart reconstructs identical
A/E/S frontiers and roots* — is the differential assertion.

### 5.5 proposervm height-index repair (`ava-proposervm`, cross-ref `06`)

The proposervm wraps the inner VM; the two persist their last-accepted pointers
**independently** (proposervm in its own DB via `vm.db` versiondb; inner in the inner
VM's DB). A crash can commit the proposervm pointer but not the inner one (C4).
`repairAcceptedChainByHeight` runs on every init **and** on `SetState`:

```rust
// ava-proposervm — mirrors vm.go::repairAcceptedChainByHeight.
fn repair_accepted_chain_by_height(&self) -> Result<(), Error> {
    let inner_la = self.inner.get_block(self.inner.last_accepted()?)?;
    let pro_la_id = match self.state.get_last_accepted() {
        Err(Error::NotFound) => return Ok(()), // pre-fork: nothing to repair
        other => other?,
    };
    let pro_h = self.get_post_fork_block(pro_la_id)?.height();
    let inner_h = inner_la.height();
    if pro_h < inner_h {
        return Err(Error::fatal("proposervm height below inner height")); // must never happen
    }
    if pro_h == inner_h { return Ok(()); }              // consistent

    // proposervm is AHEAD of the inner VM (C4): roll proposervm BACK to inner height.
    let fork_h = self.state.get_fork_height()?;
    if fork_h > inner_h {
        self.state.delete_last_accepted()?;             // rolled back past the fork
        return self.db.commit();
    }
    let new_id = self.state.get_block_id_at_height(inner_h)?; // fatal if pruned (NumHistoricalBlocks too low)
    self.state.set_last_accepted(new_id)?;
    self.db.commit()
}
```

The repair is **one-directional**: proposervm may only be ahead of the inner VM
(because the inner accept commits before the proposervm pointer in the accept path),
so recovery rolls the proposervm pointer **back** to the inner height. The reverse
(`pro_h < inner_h`) is a hard fatal.

---

## 6. Corruption handling (`corruptabledb`, fatal-vs-recoverable)

### 6.1 `corruptabledb` poison semantics

The node wraps its base on-disk DB in `corruptabledb` (`04` §2.6). On any error
**other than** `Closed`/`NotFound`, it latches the error and every subsequent op
returns it — "closing the database to avoid possible corruption." This converts a
silent IO/corruption fault into a hard, fail-fast stop rather than continued writes
over corrupt state.

```rust
// ava-database::corruptabledb — already specified in 04 §2.6; recovery semantics here.
struct CorruptableDb<D> { inner: D, poison: parking_lot::RwLock<Option<Error>> }

impl<D> CorruptableDb<D> {
    /// Latch on Error::Other only; Closed/NotFound are normal control flow.
    fn handle_error(&self, e: &Error) {
        if matches!(e, Error::Other(_)) {
            let mut p = self.poison.write();
            if p.is_none() { *p = Some(Error::Other(anyhow::anyhow!("database corrupted"))); }
        }
    }
    fn check(&self) -> Result<(), Error> {
        if let Some(e) = self.poison.read().as_ref() { return Err(e.clone_like()); }
        Ok(())
    }
}
```

### 6.2 Fatal vs recoverable classification (cross-ref `17` §5 supervision)

| Class | Examples | Handling |
|---|---|---|
| **Recoverable** | `NotFound` (missing key — normal), `Closed` (during shutdown), a tx *revert* (SAE/EVM — consumes gas, normal) | Control flow; not an error condition. |
| **Bootstrap-recoverable** | Partial fetched set (C5), behind-frontier execution (C6), pointer/state skew (C2/C4) | The §5 recovery procedures + `19` resume; idempotent. |
| **Fatal (stop the subsystem)** | corruptabledb poison; a SAE tx that *errored* (not reverted) — corrupts the head if continued (`11` §6.1); `pro_h < inner_h`; reth-db ahead of LA; checksum mismatch in blockdb that the scan can't reconcile | Stop the executor / fail node init; surface to `ava-node`. Per `17` §5, a fatal DB/VM error cancels the chain token and **does not auto-restart** — it requires operator intervention (the SAE "emergency playbook" pointer). |

> The boundary mirrors Go: recoverable errors flow as `Result`/sentinels; fatal ones
> stop the affected chain's tasks (cancel its `CancellationToken`, `17` §4) and are
> logged at fatal level. A poisoned base DB fails *all* chains (it is the shared base).

---

## 7. Recovery / bootstrap interaction (cross-ref `19`)

Restart recovery (§5) and the three-phase sync lifecycle (`19`) compose: §5 restores
the VM to a consistent *durable* last-accepted; `19` then resumes the **sync** state
machine from there.

- **Interrupted bootstrap (C5).** On `Start`, the bootstrap actor rebuilds the
  interval `Tree` and `missingBlockIDs` from the DB and resumes fetching from the
  already-fetched intervals — never from genesis (`19` §2.1/§7).
- **Interrupted state-sync (C5).** `GetOngoingSyncStateSummary` re-offers the
  persisted summary so the syncer resumes against the same target root; EVM resumes
  per-trie leaf progress, merkledb re-derives from local roots (`19` §3.2/§6/§7).
- **Ordering with the marker (§4).** The `ungracefulShutdown` warning is logged
  before any chain inits; each chain's §5 recovery then runs, then `19` resumes. A
  node that crashed mid-sync and mid-execute (both C5 and C6) handles them
  independently per chain — there is no global recovery barrier.

---

## 8. Go → Rust mapping

| Go | Rust |
|---|---|
| `acceptor.go` `Apply` → `CommitBatch` → `SharedMemory.Apply(reqs, batch)` → `defer Abort` | `on_accept_state.apply` → `commit_batch` → `shared_memory.apply(reqs, &[batch])` → `abort_guard` (§2.2) |
| `chains/atomic/writer.go::WriteAll` | `ava_chains::atomic::write_all` (replay siblings onto base `inner()`, one `write()`) |
| `chains/atomic/shared_memory.go::Apply` (sorted `sharedID`, versiondb, `WriteAll`) | `SharedMemory::apply` (sorted ids, `VersionDb`, `write_all`) |
| `versiondb` `Commit`/`CommitBatch`/`Abort` | `VersionDb` (`04` §2.4) `commit`/`commit_batch`/`abort` |
| `coreth atomic_state.go::Accept` (3-batch merge; bonus-block branch) | `ava-evm` atomic accept (Firewood commit then merged KV+atomic+SM batch; bonus-block branch) |
| `node.go` `ungracefulShutdown` key | `ava-node` `UNGRACEFUL_SHUTDOWN` (byte-exact key; `mark_running`/`clear_running`, §4) |
| `proposervm vm.go::repairAcceptedChainByHeight` | `ava-proposervm::repair_accepted_chain_by_height` (§5.5) |
| `sae/recovery.go` (rebuild A/E/S) | `ava-saevm-core` recovery (`11` §1.4, §5.4) |
| `evm vm.go` `ReadLastAccepted`/`initChainState`/`initializeStateSync` | `ava-evm` boot reconciliation (§5.3) |
| `corruptabledb.handleError`/`corrupted` | `CorruptableDb::handle_error`/`check` (§6.1) |
| `merkledb cleanShutdown` flag / blockdb recovery scan | `04` §10.8 / §5.1 (unclean-triggered, §4.1) |
| `errors.Is(ErrNotFound)` control flow | `matches!(e, Error::NotFound)` (do not treat as fatal) |

### Error model

Per `00` §7.1: each crate's `thiserror` enum + `pub type Result<T>`. The recovery
sentinels are matched, not stringly-compared: `Error::NotFound` (pre-fork / missing
LA → "nothing to repair"), `Error::Closed` (shutdown race), and a dedicated
`Error::Fatal(_)` (or per-VM `Fatal` variant, `11` §11) for the §6.2 fatal class
that stops the chain's tasks. `anyhow` only in the binary/tests.

---

## 9. Test plan — crash-injection suite (cross-ref `02`)

The differential harness (`02`) already drives identical block/tx sequences through
a Rust node and a Go oracle. This spec adds a **crash-injection** layer on top.

1. **Crash-point proptest (the core suite).** Parameterize the accept/execute path
   with a `CrashPoint` enum covering every row of §3 (C0–C7) plus sub-points (before
   SM replay, between replay and `write`, between Firewood commit and KV write, mid
   blockdb append, between commit interval and head for SAE). A test seam injects a
   panic/`process::abort`-equivalent at the chosen point (in-process: a
   `FailpointDb` wrapper that errors/aborts on the N-th `write()`; out-of-process: a
   harness that `kill -9`s a child node at a logged checkpoint). On restart, run the
   §5 recovery and assert:
   - the reconciled last-accepted, state root, and (for SAE) A/E/S frontiers equal
     the **Go node's** state after the same crash + restart (recovery equivalence,
     `11` invariant 7);
   - **CC-ATOMIC**: every accepted block is *fully* present or *fully* absent — never
     a partial state diff / dangling LA / orphan SM entry.
2. **Shared-memory two-sided consistency (the trickiest case, §3.1).** Drive an
   X→P export/import (and X→C) and crash specifically in the [SM-replay, write)
   window on both the export and the import side. Assert the peer chain observes
   all-or-nothing and the UTXO is never double-spendable nor lost — against the Go
   oracle. This is the highest-value test in the suite.
3. **Partial-write fuzz.** Fuzz torn writes at the storage layer: blockdb data-file
   ahead of index (assert the recovery scan rebuilds the index identically to Go —
   `04` §6.8), and truncated RocksDB WAL (assert all-or-nothing batch replay).
   merkledb `cleanShutdown=false` ⇒ assert intermediate-node rebuild reproduces the
   same root.
4. **proposervm repair property test.** Construct skewed (proposervm-ahead) states
   at random heights incl. across the fork height; assert `repair_accepted_chain_by_
   height` rolls back to the inner height (or deletes past the fork) and that
   `pro_h < inner_h` is rejected as fatal.
5. **Marker semantics.** Assert the `ungracefulShutdown` key is byte-exact, present
   after start, absent after clean shutdown, and present after an injected crash; and
   that a forged/missing marker degrades to "extra work, identical result" (§4.1).
6. **Corruptabledb poison.** Inject an `Error::Other` mid-run; assert all subsequent
   ops latch the error, the chain's tasks stop (cancel token, `17` §5), and the node
   does not silently continue writing.
7. **Bootstrap/state-sync resume (cross-ref `19` §8).** Crash mid-phase; assert
   resume from the interval `Tree` / re-offered summary reaches the identical
   post-sync state — composed with §5 recovery for the already-accepted prefix.

Each crash-injection case is a `proptest` over (crash point, chain type, sequence)
and asserts byte-identical post-recovery state vs the Go node (`02`).

---

## 10. Performance notes / improvements over Go

All are behavior-preserving (`00` §6.1/§9) and gated by §9's differential +
crash-injection tests; **"safe because" the atomic boundary is preserved**.

- **Commit batching is a perf lever, not just correctness.** Merging the state
  diff + LA + SM into one RocksDB `WriteBatch` (§2.2) is *one* WAL fsync per Accept
  instead of several — the same number Go does, but the Rust `Batch::replay` path
  avoids re-serializing keys (zero-copy `Bytes`, `04` §9). Safe because the merged
  batch is byte-equivalent to Go's `WriteAll` output.
- **Firewood propose/commit pipelining (cross-ref `04` §4.2, `11` §13).** SAE/EVM
  compute the next state root via `propose` (in-memory, hot path) and `commit`
  durably on a background thread at the commit interval — overlapping hashing of
  block N+1 with commit of block N. Safe because the *recovery start point* (last
  committed Firewood revision) and the re-execution replay are deterministic, so a
  later commit cadence only changes how much is replayed on restart, never the
  resulting roots (CC-ORDER, §2.4).
- **Parallel recovery re-execution where independent.** SAE recovery re-executes a
  *serial* chain (state-dependent), so the execution itself stays serial, but sender
  recovery / worst-case re-validation of the replayed blocks fan out over `rayon`
  (`11` §13.3). Safe because inclusion ordering is fixed by the accepted chain.
- **Idempotent, lock-light reconciliation.** The §5 reconciliations are pure reads +
  a single small commit, run once at init under no contention; `arc_swap` frontier
  pointers (`11` §13.5) make the post-recovery hot path lock-free.
- **Fail-fast corruption.** corruptabledb's latch (§6.1) stops a faulting node in
  one extra `RwLock` read per op (negligible) rather than risking a slow corruption
  spread — strictly safer than Go, identical semantics.
