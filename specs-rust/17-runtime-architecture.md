# 17 — Runtime Architecture (Concurrency, Tasks, Channels, Lifecycle)

> **Status:** Conforms to `00-overview-and-conventions.md` (binding), esp. §7.2
> (async/threading), §7.4 (cancellation & lifecycles), §6 (goroutine→task
> mapping). This is the **authoritative** concurrency/task/channel topology and
> lifecycle document for the whole workspace. Where a subsystem spec describes
> *its* tasks, this file is the cross-cutting source of truth for how they are
> owned, sized, cancelled, and supervised. Cross-refs: `05` (per-peer actor +
> message queue), `06` (per-chain Handler/Router/TimeoutManager/engine loop),
> `07` (chains manager, VM middleware), `11` (SAE streaming pipeline),
> `12` (`ava-node` init/shutdown ordering), `16` (milestone build order),
> `18` (metrics/logging task instrumentation).

## Go source paths covered

| Go area | What this spec extracts |
|---|---|
| `node/node.go` (`New`, `Dispatch`, `Shutdown`) | long-lived node goroutines + boot/shutdown ordering |
| `network/network.go` (`Dispatch`, `runTimers`, `dial`) | accept loop, dialer, periodic timers |
| `network/peer/peer.go` (`Start`, `readMessages`, `writeMessages`, `sendNetworkMessages`) + `message_queue.go` | the 3-goroutine per-peer actor + outbound queue |
| `snow/networking/handler/handler.go` (`dispatchSync`/`dispatchAsync`/`dispatchChans`, `asyncMessagePool`) + `message_queue.go` | per-chain handler: 3 dispatchers + async worker pool + 2 queues |
| `snow/networking/router/chain_router.go` | routing + timeout registration |
| `snow/networking/timeout/manager.go` + `utils/timer/adaptive_timeout_manager.go` | the timeout dispatch loop |
| `snow/networking/sender/{sender,external_sender}.go` | outbound (synchronous, no own task) |
| `snow/networking/tracker/resource_tracker.go` | CPU/disk usage sampling loop |
| `api/health/{worker,health}.go` | health check loop |
| `chains/manager.go` | one engine+handler per chain |
| `network/p2p/gossip/gossip.go` (`Every`, push/pull gossipers) | gossip tickers |
| `vms/saevm/saexec` (executor goroutine) | the SAE streaming-execution pipeline |
| `x/...` / Firewood | commit threads (`spawn_blocking`) |

---

## 1. Runtime ownership

### 1.1 One runtime, owned by the binary

There is **exactly one** `tokio` multi-thread runtime in the process. It is built
by the `avalanchego` binary (`12` §9) and passed into `Node::new` as a
`tokio::runtime::Handle`. Every library crate (`ava-network`, `ava-engine`,
`ava-chains`, `ava-vm`, `ava-saevm`, `ava-api`, …) **spawns onto the ambient
runtime** via `tokio::spawn`/the passed `Handle`; none constructs its own
`Runtime`. This mirrors Go, where the entire node shares one goroutine scheduler.

```rust
// avalanchego/src/main.rs (sketch)
fn main() -> std::process::ExitCode {
    let rt = tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .thread_name("ava-worker")
        // worker_threads defaults to num_cpus; Go has no equivalent knob — leave default.
        .build()
        .expect("build tokio runtime");
    let handle = rt.handle().clone();
    rt.block_on(async move {
        let node = Node::new(config, log_factory, log, handle).await?;
        install_signal_handlers(node.clone()); // SIGINT/SIGTERM -> node.shutdown(0)
        node.dispatch().await?;                // blocks until the P2P stack stops
        Ok::<_, anyhow::Error>(())
    });
    std::process::ExitCode::from(node.exit_code() as u8)
}
```

> **Rule (00 §7.2): no sub-runtimes.** `block_on`, `Runtime::new`, or
> `Handle::current().block_on` inside library code is forbidden (nested runtimes
> panic and break cancellation). A `clippy`/grep CI gate forbids
> `Runtime::new`/`Builder::*::build` outside the `avalanchego` bin and tests.

### 1.2 Dedicated blocking pools

Async worker threads must never block. CPU-bound or FFI-blocking work is moved
off the reactor:

| Work | Mechanism | Why |
|---|---|---|
| RocksDB reads/writes/batches (`ava-database`) | `spawn_blocking` **at the call site** (00 §11.3: the `Database` trait stays synchronous) | RocksDB FFI blocks the calling thread. |
| Firewood propose/commit (`ava-merkledb`/`04`, SAE `11`) | `spawn_blocking` | trie hashing + disk commit is CPU+IO heavy; SAE pipelines it behind consensus (00 §9). |
| EVM/revm execution (`ava-evm`/`10`, SAE `cchain`/`11`) | `spawn_blocking` (single block) or a dedicated `rayon` pool for independent batch verification | revm is synchronous and CPU-bound. |
| Signature batch verification (secp256k1, BLS aggregate; `ava-crypto`) | `rayon` data-parallel pool | independent, parallelizable (00 §9) — **only** where it cannot reorder observable effects. |
| pprof CPU profiling (`admin` API) | `spawn_blocking` | sampling profiler holds a thread. |

`spawn_blocking` uses tokio's blocking-thread pool (default max 512 threads; bound
it lower via `max_blocking_threads` if RocksDB/Firewood concurrency must be
capped to match Go's behavior). `rayon` pools are **separate** from tokio's and
sized to physical cores; they are created once and shared via `Arc`.

> **Rule: no lock across `.await`** (00 §7.2). Hold a `parking_lot` lock only for
> the synchronous critical section; drop it before awaiting. Long-lived shared
> state that must be touched across awaits uses the **actor pattern** (§7) or
> `tokio::sync` primitives, not a `std`/`parking_lot` guard carried over a yield.

---

## 2. The complete task graph

### 2.1 ASCII topology

```
                              ┌──────────────────────────────────────────┐
                              │            avalanchego (bin)               │
                              │  owns Runtime + root CancellationToken     │
                              │  signal handler -> node.shutdown(0)        │
                              └───────────────────┬────────────────────────┘
                                                  │ spawns
                    ┌─────────────────────────────┼───────────────────────────────┐
                    │                             ava-node                          │
                    │   (orchestrates start/shutdown ordering; holds child tokens)  │
                    └──┬───────────┬───────────┬───────────┬───────────┬────────────┘
                       │           │           │           │           │
        ┌──────────────▼──┐  ┌─────▼─────┐  ┌──▼───────┐ ┌─▼────────┐ ┌▼──────────────┐
        │   ava-network    │  │ TimeoutMgr│  │ HealthLoop│ │ResourceTr│ │  API/HTTP     │
        │                  │  │ (1 task)  │  │ (1 task)  │ │(1 task)  │ │  server task  │
        │  accept loop ────┼──┐                                          │  + WS fanout  │
        │  dialer (N tasks)│  │ inbound conn                             └───────────────┘
        │  runTimers (1)   │  │ throttler (1)
        │  inbound throttle│  │
        └────────┬─────────┘  │
                 │ per accepted/dialed peer (P peers):
       ┌─────────▼───────────────────────────────────┐
       │              Peer actor (×P)                  │
       │   read task  ─►  inbound parse                │
       │   write task ◄─  outbound queue (deque)       │
       │   sender task (periodic ping/pingpong)        │
       └───────────────────┬──────────────────────────┘
                            │ InboundMessage
                   ┌────────▼─────────┐
                   │   ChainRouter     │  (no own task; sync fan-out under lock;
                   │  HandleInbound    │   spawns one detached task per message:
                   └────────┬──────────┘   `go cr.handleMessage`)
                            │ Push(msg)  routed by chainID
        ┌───────────────────▼───────────────────────────────────────┐
        │                Per-chain Handler (×C chains)                │  06
        │  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
        │  │dispatchSync│  │dispatchAsync│  │dispatchChans│           │
        │  │  (1 task)  │  │  (1 task)  │  │  (1 task)  │            │
        │  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘            │
        │   sync queue       async queue    msgFromVM / VM-notify     │
        │        │                │ spawns onto                       │
        │        │                ▼ asyncMessagePool (semaphore=K)    │
        │   ENGINE (single-task owner of consensus state) ───────────┼─► VM (07)
        └────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────▼──────────────┐   ┌──────────────────────────┐
        │  Gossip tickers (per protocol)    │   │  SAE executor pipeline    │ 11
        │  push/pull (Every loop)           │   │  1 executor task draining │
        └───────────────────────────────────┘   │  FIFO mpsc; commits via   │
                                                 │  spawn_blocking (Firewood)│
        ┌───────────────────────────────────┐   └──────────────────────────┘
        │  Indexer acceptor (broadcast sub)  │
        │  NAT ip-updater (1), profiler (1)  │
        └───────────────────────────────────┘
```

### 2.2 Long-lived task table

Counts: **per-node** = 1 per process; **per-peer** = ×P (live peers); **per-chain**
= ×C (tracked chains). "Cancels" lists the cancellation source (§4).

| # | Task | Crate | Count | Owns | Inbound channel(s) (type · cap · overflow) | Outbound | Cancelled by |
|---|------|-------|-------|------|--------------------------------------------|----------|--------------|
| 1 | Network **accept loop** (`network.Dispatch`) | `ava-network` | per-node | the TCP listener | OS `accept()` (not a channel) | spawns Peer actors via upgrade | root token (listener close → loop exits) |
| 2 | **Dialer** (`network.dial`) | `ava-network` | per-dial (transient, ≤ outbound-throttle RPS) | one outbound connection attempt | trigger from `Track`/reconnect | spawns Peer actor on success | network close ctx; self-exits on connect/fail+backoff |
| 3 | Inbound-conn **upgrade throttler** (`inboundConnUpgradeThrottler.Dispatch`) | `ava-network` | per-node | per-IP cooldown state | internal accept notifications | gates accept loop | root token |
| 4 | **runTimers** | `ava-network` | per-node | peerlist-pull / bloom-reset / uptime tickers | 3 `tokio::time::interval` (no channel) | calls into peers/ip-tracker | root token (`select!` on token) |
| 5 | Peer **read** task (`readMessages`) | `ava-network` | per-peer | the read half of the TLS conn | socket bytes | parsed `InboundMessage` → Router | peer `StartClose` / read error / root token |
| 6 | Peer **write** task (`writeMessages`) | `ava-network` | per-peer | the write half of the TLS conn | **outbound message queue** (see §3) | socket bytes | peer close (queue closed) |
| 7 | Peer **sender** task (`sendNetworkMessages`) | `ava-network` | per-peer | ping/pong + periodic gossip-trigger timers | ping ticker + `getPeerListChan` (cap **1**) | enqueues onto outbound queue | peer close |
| 8 | **ChainRouter** message handler (`handleMessage`) | `ava-engine` (networking) | per-message (transient, spawned by `HandleInbound`) | request→reply correlation (under lock) | `InboundMessage` from peers | `Handler::push` + `TimeoutManager` register/remove | parent op completes (short-lived) |
| 9 | **TimeoutManager** dispatch loop (`adaptiveTimeoutManager.Dispatch`) | `ava-engine` | per-node | the timeout min-heap + adaptive EWMA | `Put`/`Remove` (method calls under lock) + one fire timer | invokes timeout handlers (route a failure msg) | `timeout_manager.stop()` (shutdown step 4) |
| 10 | Handler **dispatchSync** | `ava-engine` | per-chain | drains the **sync** queue into the engine | sync `MessageQueue` (see §3) | engine method calls (single-threaded) | handler `Stop`/`closingChan` |
| 11 | Handler **dispatchAsync** | `ava-engine` | per-chain | drains the **async** queue | async `MessageQueue` (see §3) | spawns onto async worker pool | handler `Stop` |
| 12 | Handler **dispatchChans** | `ava-engine` | per-chain | VM→engine notifications | `msgFromVMChan` (unbuffered `mpsc(1)`) + change-notifier | engine `Notify`/gossip | handler `Stop` |
| 13 | **Async worker pool** (`asyncMessagePool`, errgroup w/ `SetLimit`) | `ava-engine` | per-chain (×K workers, **K = `consensus-app-concurrency`, default 2**) | App/cross-chain message handling | tasks submitted by #11 | `AppResponse`/`AppError` via Sender | handler `Stop` (pool drained/joined) |
| 14 | **Engine** | `ava-engine` | per-chain (logically; runs inside #10 as the single owner) | **all consensus state** (Snowman/Avalanche instance, bootstrapper) | sync queue (via #10) | Sender → Router → peers; VM calls | handler `Stop` |
| 15 | **Gossip** push/pull tickers (`gossip.Every`) | `ava-network` p2p | per-gossip-protocol (per chain that gossips) | a `tokio::time::interval` | ticker | `AppGossip` outbound via Sender | chain/root token |
| 16 | **SAE executor** pipeline | `ava-saevm` (`saexec`) | per-SAE-chain | the execution frontier + in-flight block | FIFO `mpsc<Arc<Block>>`, **cap = `2 * commit_interval`**, push **blocks** | Firewood commit via `spawn_blocking` | chain token (finishes in-flight, then exits) |
| 17 | **Firewood commit** worker | `ava-merkledb`/`ava-saevm` | per-commit (transient) | one trie propose+commit | invoked by #16 / sync VM accept | new state root | n/a (blocking task runs to completion) |
| 18 | **API/HTTP server** | `ava-api` | per-node | the HTTP listener (axum/hyper) | TCP accept (hyper) | per-request handler futures | `api_server.shutdown()` (step 9) / on error → `node.shutdown(1)` |
| 19 | **WebSocket pub-sub fanout** | `ava-api` | per-subscribed-feed | a `broadcast` sender | `broadcast` from acceptor | WS frames to clients | client disconnect / server shutdown |
| 20 | **Indexer** acceptor | `ava-indexer` | per-indexed-chain | the append-only index batch | `AcceptorGroup` `broadcast` sub | RocksDB batch via `spawn_blocking` | `indexer.close()` (step 11) |
| 21 | **Health** check loop | `ava-api`/health | per-node | the registered-checker set + EWMA | `tokio::time::interval(freq)` | health result map | root token |
| 22 | **ResourceTracker** sampler | `ava-engine`/tracker | per-node | per-peer CPU/disk EWMA | `tokio::time::interval(system-tracker-frequency, 500ms)` | usage metrics + throttler input | `resource_manager.shutdown()` (step 3) |
| 23 | **NAT ip-updater** | `ava-node` (nat) | per-node | the public-IP + port mappings | `tokio::time::interval(public-ip-resolution-frequency)` | updates advertised IP (network) | `ip_updater.stop()` (step 10) |
| 24 | **Continuous profiler** | `ava-node` | per-node (opt-in) | rotating pprof files | `tokio::time::interval(profile-continuous-freq)` | files on disk | `profiler.shutdown()` (step 7) |
| 25 | **Plugin runtime host** (rpcchainvm) | `ava-vm-rpc` | per-plugin VM | a child process + gRPC channel | tonic server (Runtime svc) + client | proxied rpcdb/appsender/etc. | `runtime_manager.stop()` (step 12) |

**Task-type count: 25 long-lived task *types*.** Steady-state instance count
scales as `O(1) (node-wide) + 3·P (peers) + (3 + K)·C (chains)` plus transient
per-message/per-dial/per-commit tasks.

> **Determinism boundary (cross-ref `06`):** task **#14 (Engine)** is the *single
> task that owns consensus state* — it is driven only by **#10 (dispatchSync)**,
> so all consensus-affecting handling is serialized in accept/queue order. Async
> work (**#11/#13**, App messages, cross-chain) and gossip (**#15**) run **off the
> consensus path** and must never mutate consensus state. SAE's executor (**#16**)
> is likewise a *single* task so execution order == accept order (00 §6.1).

---

## 3. Channel sizing & backpressure

Go uses a mix of buffered channels, an unbounded throttled deque, and a
condvar-guarded bounded queue. The Rust mapping preserves the **backpressure
semantics** (block vs drop) per channel, because that distinction is
load-bearing: consensus messages **block/throttle** (never silently lost), gossip
**drops** under pressure.

| Go construct (path) | Go capacity / kind | Rust primitive | Capacity | Backpressure / overflow |
|---|---|---|---|---|
| Peer outbound queue (`peer.NewThrottledMessageQueue`, `message_queue.go`) | **unbounded deque** (`buffer.NewUnboundedDeque`, init 64) gated by a **byte-based outbound throttler** (`OutboundThrottlerNodeMaxAtLargeBytes = MaxMessageSize`) | `mpsc::unbounded` **gated by a byte `Semaphore`** (acquire bytes on push, release on pop) | unbounded items, bounded **bytes** | `Push` returns `false` (drop) if throttler can't grant bytes — matches Go: a peer that floods is dropped, not allowed to OOM us. |
| Peer outbound queue, *blocking* variant (`peer.NewBlockingMessageQueue`) | `make(chan, bufferSize)` | `mpsc::channel(buffer_size)` | from config | **blocks** on full (used where we must not drop). |
| `getPeerListChan` (`peer.go:163`) | `make(chan struct{}, 1)` | `mpsc::channel(1)` or `Notify` | **1** | coalescing: a pending request need not be duplicated; extra signals dropped. |
| `msgFromVMChan` (`handler.go:154`) | `make(chan, 0)` (unbuffered) | `mpsc::channel(1)` | **1** | unbuffered rendezvous → `mpsc(1)`; VM blocks until the chans dispatcher takes it (00 §6 mapping). |
| `closingChan`/`closed` (`handler.go:159-160`) | `make(chan struct{})` (signal) | `CancellationToken` + `oneshot`/`Notify` | — | one-shot close signal (no payload). |
| Handler **sync** queue (`handler.NewMessageQueue("sync")`) | condvar-guarded, bounded by **`throttler-inbound-node-max-processing-msgs`** + at-large/validator byte allocs | `mpsc::channel(N)` fronted by the **inbound msg throttler** (semaphore by msg-count + bytes) | per-node processing-msg cap | **blocks/throttles** the router push; over the per-node cap, the inbound throttler holds the bytes (consensus msgs are never dropped, only back-pressured) — see `05` throttling. |
| Handler **async** queue (`handler.NewMessageQueue("async")`) | same throttled queue, "async" tag | same as sync | per-node cap | same throttle; drained onto the worker pool. |
| `asyncMessagePool` (`errgroup.SetLimit(threadPoolSize)`) | worker pool limit = **`consensus-app-concurrency` (default 2)** | `tokio::sync::Semaphore(K)` + `tokio::spawn` | **K = 2** | `acquire().await` blocks dispatchAsync until a permit frees — bounded App concurrency. |
| ChainRouter `HandleInbound` (`go cr.handleMessage`) | one goroutine per inbound message | `tokio::spawn` per message (or a small bounded pool) | unbounded spawns (rate-limited upstream by peer/throttler) | bounded indirectly by the inbound throttler; no own buffer. |
| `AcceptorGroup` dispatch (`snow.AcceptorGroup`, indexer/pubsub) | fan-out callback | `tokio::sync::broadcast(cap)` | tuned (e.g. 1024) | **lagging** receivers get `RecvError::Lagged` (oldest dropped) — acceptors must keep up; the indexer offloads writes to `spawn_blocking`. |
| SAE accept→execute queue (`saexec`, `11` §6) | buffered channel | `mpsc::channel(2 * commit_interval)` | **2·commit_interval** | **blocks** the accept path; if the executor can't keep up the node FATALs rather than unbounded-buffer (`11` §5) — *consensus cannot outrun execution*. |
| Watch/config values (e.g. current public IP, current frontier height) | shared via locks/atomics | `tokio::sync::watch` / `arc_swap::ArcSwap` | 1 (latest) | lossy-by-design (only latest matters). |

**Rules.**
1. **Consensus/inbound messages: never silently drop** — they are throttled
   (byte+count semaphores) so a slow chain back-pressures the source, matching
   Go's processing-message throttler. SAE goes further: outrunning execution is a
   **fatal**, not a drop.
2. **Gossip & peer-flood protection: drop** — outbound peer queue drops when the
   byte throttler is exhausted; `broadcast` lagging drops oldest. These are not
   consensus-load-bearing.
3. **Unbuffered Go channels → `mpsc(1)` or `oneshot`** (rendezvous semantics),
   never `unbounded` (which would change ordering/backpressure).
4. **Capacities are cited from Go and centralized** (00 §5): the semaphore byte
   limits live in `ava-network` throttling config; `K` and `commit_interval` are
   config-driven (`12` §1.3). A golden test asserts defaults
   (`consensus-app-concurrency=2`, throttler at-large = `MaxMessageSize`) match Go.

---

## 4. Cancellation & graceful shutdown

### 4.1 The CancellationToken tree

`context.Context` cancellation becomes a **tree of `CancellationToken`s** (00
§7.4). The root lives in `Node`; every subsystem gets a `child_token()`, and every
chain a grandchild. Cancelling a parent cancels the whole subtree; cancelling a
child does not affect siblings or the parent.

```rust
use tokio_util::sync::CancellationToken;
use tokio_util::task::TaskTracker;

pub struct Node {
    shutdown: CancellationToken,        // root
    tasks: TaskTracker,                 // node-level long-lived tasks
    // …
}

// network gets a child; each peer a grandchild of the network token.
let net_token   = node.shutdown.child_token();
let peer_token  = net_token.child_token();          // cancelled when net closes OR node shuts down

// chain manager gets a child; each chain a grandchild; each chain owns a TaskTracker.
let chains_token = node.shutdown.child_token();
let chain_token  = chains_token.child_token();       // per-chain (subnet→chain layering, §4.2)
```

Layering (`root → subnet → chain → task`):

```
root (Node.shutdown)
├── network          (net_token)
│   └── peer×P       (peer_token = child of net_token)
├── subnet×S         (subnet_token)
│   └── chain×C      (chain_token = child of subnet_token)
│       ├── handler dispatchers (sync/async/chans)
│       ├── async worker pool
│       ├── gossip tickers
│       └── SAE executor (if SAE)
├── timeout manager / resource tracker / health
└── api / indexer / nat / profiler
```

Stopping a single subnet (e.g. untracking it) cancels `subnet_token`, tearing
down only that subnet's chains — without touching the network or other subnets.

### 4.2 Task pattern (cancel-aware skeleton)

Every long-lived task `select!`s its work against its token and is registered on a
`TaskTracker` so shutdown can drain it (cross-ref `02`'s "drain your
TaskTracker" rule):

```rust
fn spawn_dispatch_sync(
    tracker: &TaskTracker,
    token: CancellationToken,
    mut queue: mpsc::Receiver<HandlerMsg>,
    engine: Arc<Mutex<Engine>>,   // single-owner; lock NOT held across .await — see §7
) {
    tracker.spawn(async move {
        loop {
            tokio::select! {
                biased;                              // check cancellation first
                _ = token.cancelled() => break,      // graceful: stop pulling new work
                maybe = queue.recv() => match maybe {
                    Some(msg) => engine_handle(&engine, msg).await,  // drives consensus
                    None => break,                   // queue closed
                }
            }
        }
        // task-local cleanup here; the tracker.close()+wait() joins us.
    });
}
```

`biased` + cancellation-first ensures we stop accepting new work promptly. The
in-flight message is allowed to finish (drain), then the loop exits.

### 4.3 Shutdown order (mirror `node.Shutdown`, exact)

`Node::shutdown` runs **once** (`OnceCell`) and reproduces `node/node.go::Shutdown`
step-for-step (`12` §2.4). The order is load-bearing — stop *producers* before
*consumers*, persist last, kill children last:

1. Register the `shuttingDown` health check (flip node unhealthy), sleep
   `http-shutdown-wait` (lets LBs drain).
2. `staking_signer.shutdown()`.
3. `resource_manager.shutdown()` (cancels #22).
4. `timeout_manager.stop()` (cancels #9 — stop firing request timeouts).
5. `chain_manager.shutdown()` — for each chain: cancel `chain_token`, **drain**
   the handler (finish in-flight sync msg, join the async worker pool via its
   `TaskTracker`), stop the SAE executor (finish in-flight block, commit), stop
   gossip tickers, then drop the engine/VM.
6. `benchlist.shutdown()`.
7. `profiler.shutdown()` (cancels #24).
8. `net.start_close()` — close the listener (stops #1/#3/#4), cancel `net_token`
   so every peer actor (#5/#6/#7) unwinds.
9. `api_server.shutdown()` — graceful with `http-shutdown-timeout` (10s default);
   stops #18/#19.
10. `nat.unmap_all_ports()`, `ip_updater.stop()` (cancels #23).
11. `indexer.close()` (cancels #20, flush final batch).
12. `runtime_manager.stop()` — kill plugin subprocesses (#25).
13. `db.delete(UNGRACEFUL_SHUTDOWN)` then `db.close()` — **persistence last**.
14. `tracer.close()` (flush spans).

### 4.4 Draining vs aborting; join & timeout

- **Drain (preferred):** cancel the token, let `select!` loops finish the in-flight
  unit, then `tracker.close(); tracker.wait().await` to join. Used for handlers,
  the SAE executor, the indexer — anything whose in-flight unit must complete for
  correctness (a half-applied accept must not be left).
- **Abort (last resort):** wrap the join in `tokio::time::timeout(deadline, …)`;
  if a task does not drain within the bound (`consensus-shutdown-timeout`), log
  and `JoinHandle::abort()` it. Aborting is acceptable only for tasks whose
  in-flight work is idempotent or discardable (gossip, periodic timers, the HTTP
  server beyond its grace window).
- **Order:** always cancel → drain-with-timeout → abort-stragglers → drop owned
  resources. Never `abort()` a task that holds a DB write batch mid-commit; route
  those through the drain path so the batch either fully applies or is discarded
  atomically (`ava-database` versioned batch, `04`).

---

## 5. Supervision & restart

There is **no generic actor-restart supervisor.** Avalanche's Go model treats most
failures as fatal-to-the-node, and we mirror that:

| Task class | On panic / fatal error | Restartable? |
|---|---|---|
| Engine / handler dispatchers (#10–#14) | **fatal** → `node.shutdown(1)`. A corrupt consensus state must not be silently restarted (would risk divergence). Mirrors Go `RecoverAndExit`/`onFatal`. | No. |
| SAE executor (#16) | **fatal** → stop the executor, then `node.shutdown(1)` (`11` §5: an execution error is unrecoverable; points at the recovery playbook on restart). | No (recovery is a *cold restart* that re-derives frontiers from disk, `11` §1.4). |
| Network accept loop / dialer (#1/#2) | **resilient**: log, back off, continue accepting/redialing (Go sleeps on accept error). A single bad connection never kills the node. | Yes (self-healing loop). |
| Per-peer actor (#5–#7) | **isolated**: a read/write error closes *that* peer only (`StartClose`); the node redials if it's a tracked validator. | The connection, not the task, is "restarted" (a fresh dial spawns fresh tasks). |
| API/HTTP server (#18) | **fatal if unexpected**: on dispatch error while not shutting down, log fatal then `node.shutdown(1)` (mirror `node.go::Dispatch`). | No. |
| Periodic timers, gossip, health, resource tracker (#4/#15/#21/#22) | **resilient**: a single tick error is logged; the loop continues. | Yes (loop continues). |
| Plugin subprocess (#25) | plugin crash surfaces as a gRPC transport error → the owning chain fails → `node.shutdown(1)` (matches go-plugin behavior). | No (operator restarts the node). |

**Panic propagation.** Library code is panic-free in steady state (00 §8: no
`unwrap`/`expect` outside proven invariants). A panic in a spawned task is caught
at the spawn boundary (`JoinHandle` returns `Err(JoinError::is_panic)`); for fatal
classes we log the backtrace (mirror `log.StopOnPanic` / `GetStacktrace`) and call
`node.shutdown(1)`. SIGABRT dumps **all** task/thread backtraces to stderr
(`12` §2.5). The process exit code is the one recorded by the first fatal
`shutdown(code)`.

---

## 6. Determinism note (cross-ref `06`)

Consensus must be bit-for-bit reproducible (00 §6.1), and concurrency is the
biggest threat to that. The invariants:

1. **One task owns consensus state.** The engine (#14) is driven *only* by the
   single sync dispatcher (#10). No other task may read-modify-write the
   Snowman/Avalanche instance. This makes message-handling order == sync-queue
   order, exactly as Go's single handler goroutine guarantees.
2. **Async/App/cross-chain work is off the consensus path.** The async pool (#13),
   gossip (#15), and App handlers must not mutate consensus state; they only
   produce outbound messages or feed the VM mempool. Their nondeterministic
   scheduling therefore cannot change which block finalizes.
3. **SAE: single executor, FIFO.** The executor (#16) drains a FIFO `mpsc` so
   execution order == accept order regardless of `spawn_blocking` scheduling
   (`11` §6). Pipelined Firewood commits are decoupled for throughput but
   `maybe_commit` lands on the *same* heights every run (00 §9, validated by
   `prop::sae_execution_determinism`, `16` M7).
4. **No map-iteration / RNG nondeterminism in tasks.** Anything serialized or
   sampled inside a task uses sorted iteration / the gonum-parity MT19937 sampler
   (00 §6.1, R1) — never `HashMap` iteration order or `tokio` task-completion
   order as an input to consensus.
5. **Parallelism only where order-independent.** `rayon` batch signature
   verification (§1.2) is allowed because the *result* (valid/invalid set) is
   order-independent; the *application* of results stays in input order.

---

## 7. Send/Sync & ownership patterns

### 7.1 Shared handles: `Arc<dyn Trait>`

Cross-subsystem dependencies are object-safe traits behind `Arc` (00 §6, §11.3):
`Arc<dyn Database>`, `Arc<Network>`, `Arc<dyn Router>`, `Arc<dyn ChainManager>`,
`Arc<dyn validators::Manager>`. These are cheap to clone into each task. Traits
that cross the async boundary are `Send + Sync`; async methods use `#[async_trait]`
(00 §4.1) until native AFIT suffices.

### 7.2 The actor pattern (owned state + command enum)

State that is mutated across `.await` points is **owned by a single task** and
mutated only via messages — this is how we satisfy "no lock across `.await`"
without sprinkling `tokio::Mutex`. Handles are cheap `mpsc::Sender` clones:

```rust
/// Commands the owning task understands (one variant per operation).
enum EngineCmd {
    Inbound(InboundMessage),
    Notify(common::Message),
    Health { reply: oneshot::Sender<HealthResult> }, // request/response via oneshot
    Shutdown,
}

/// Cheap, clonable handle — what every other task holds.
#[derive(Clone)]
pub struct EngineHandle(mpsc::Sender<EngineCmd>);

impl EngineHandle {
    pub async fn health(&self) -> Result<HealthResult> {
        let (tx, rx) = oneshot::channel();
        self.0.send(EngineCmd::Health { reply: tx }).await.map_err(|_| Error::Closed)?;
        rx.await.map_err(|_| Error::Closed)
    }
}

/// The single owning task: holds `state` by value, no lock needed.
fn run_engine(mut state: Engine, mut rx: mpsc::Receiver<EngineCmd>, token: CancellationToken) {
    tokio::spawn(async move {
        loop {
            tokio::select! {
                biased;
                _ = token.cancelled() => break,
                Some(cmd) = rx.recv() => match cmd {
                    EngineCmd::Inbound(m)        => state.handle_inbound(m).await,
                    EngineCmd::Notify(m)         => state.notify(m).await,
                    EngineCmd::Health { reply }  => { let _ = reply.send(state.health()); }
                    EngineCmd::Shutdown          => break,
                },
                else => break,
            }
        }
    });
}
```

### 7.3 Channels over locks

- Prefer an **mpsc command channel** to a shared `tokio::Mutex<State>` for any
  state touched across awaits (avoids contention + accidental held-lock-across-
  await).
- Use `parking_lot::Mutex`/`RwLock` only for **short, synchronous** critical
  sections (metric counters, lookup maps read without awaiting) — and drop the
  guard before any `.await`.
- Use `arc_swap::ArcSwap` / `tokio::sync::watch` for read-mostly "latest value"
  state (validator-set snapshots `06`/`00` §9, current public IP, current
  frontier heights) so readers never block.
- `oneshot` for request/response; `broadcast` for fan-out (acceptors, pub-sub);
  `Semaphore` for bounded concurrency (the async pool, byte throttlers).

---

## 8. Anti-patterns

**Go pitfalls not to reproduce:**
- **Goroutine leaks from forgotten `select` on a done channel.** Every task
  `select!`s its `CancellationToken`; orphan tasks are forbidden (TaskTracker
  drains them).
- **Unbounded goroutine spawn per inbound message without a throttle.** The Rust
  port keeps the inbound byte/count throttler in front of any per-message spawn so
  a flood can't exhaust the runtime.
- **`time.Add(...TauSeconds)` style untyped duration math** (the Go CI grep gate):
  use typed `Duration`/`Instant` everywhere (00 §6.1, SAE Tau rules).
- **Relying on unbuffered-channel rendezvous semantics implicitly** — make the
  `mpsc(1)`/`oneshot` choice explicit; never widen to `unbounded` "to be safe"
  (that changes backpressure and can mask a stuck consumer).

**Rust-specific pitfalls:**
- **Holding a `parking_lot`/`std` lock across `.await`** — deadlocks or starves
  the reactor (00 §7.2). Use the actor pattern or `tokio::sync` instead.
- **Spawning a second `tokio::Runtime`** (or `block_on` inside async) in a library
  crate — nested-runtime panic, breaks the single-runtime rule (§1.1).
- **Blocking the reactor with RocksDB/Firewood/revm calls** instead of
  `spawn_blocking`/`rayon` (§1.2) — stalls every other task on that worker.
- **`broadcast` without handling `Lagged`** — a slow acceptor silently drops
  messages and may diverge an index; either keep up (offload writes) or treat
  `Lagged` as fatal where ordering matters.
- **`abort()`ing a task mid-DB-write** — leaves partial state; route shutdown
  through the drain path so batches apply atomically (§4.4).
- **`tokio::Mutex` everywhere by reflex** — most shared state is read-mostly
  (`ArcSwap`/`watch`) or single-owner (actor); reach for a `tokio::Mutex` only
  when state genuinely must be mutated across awaits by multiple tasks.

---

## 9. Test plan (ref `02`)

- **Shutdown-ordering test:** record the actual `Node::shutdown` step order and
  assert it equals the Go sequence (§4.3); a SIGTERM integration test asserts
  graceful drain within `consensus-shutdown-timeout` (mirrors `12` §12.7).
- **Cancellation-propagation test:** cancel a `subnet_token`; assert only that
  subnet's chains' tasks join and the network/other subnets are untouched.
- **Backpressure tests:** flood a peer's outbound queue past the byte throttler →
  assert `Push` returns drop (not block, not OOM); flood a chain's sync queue →
  assert the router back-pressures (no message loss); saturate the SAE executor →
  assert FATAL, not unbounded growth (`11` §5).
- **No-lock-across-await lint:** a `clippy`/custom gate (`02`) flags
  `parking_lot` guards live across an `.await`.
- **No-sub-runtime gate:** grep CI forbids `Runtime::new`/`block_on` outside the
  bin/tests (§1.1).
- **Determinism (loom/proptest):** `prop::sae_execution_determinism` (two forced
  pipeline schedules → identical settled roots, `16` M7); a `loom` model of the
  engine command channel asserts single-owner serialization (`00` §11.9).
- **Channel-default golden test:** assert `consensus-app-concurrency=2`,
  outbound at-large = `MaxMessageSize`, `system-tracker-frequency=500ms`, etc.
  match the Go constants (00 §5).

---

## 10. Performance notes / improvements over Go

- **Pipelined Firewood commits (SAE, #16/#17).** Execution and trie-commit run on
  separate `spawn_blocking` work so consensus accept never blocks on disk
  (00 §9); ordering preserved by the FIFO + deterministic `maybe_commit` heights.
  *Safe because* `prop::sae_execution_determinism` proves identical settled state.
- **`rayon` batch signature verification.** secp256k1/BLS-aggregate batches verify
  data-parallel instead of Go's serial loop (00 §9). *Safe because* the
  valid/invalid result set is order-independent and applied in input order (§6.5).
- **`ArcSwap`/`watch` validator-set & frontier snapshots.** Lock-free read-mostly
  access removes the `RWMutex` read contention Go incurs on the hot path (00 §9).
- **`bytes::Bytes` zero-copy on the peer read/write tasks (#5/#6).** Avoids the
  per-message copy `[]byte` round-trips incur (00 §9); buffers are reused on the
  read loop.
- **Bounded `spawn_blocking` pool sized to match Go's DB concurrency** keeps tail
  latency predictable rather than tokio's default 512 blocking threads.
- All such changes are gated by the differential harness (`02`, `16`): identical
  external behavior vs. the Go node, never a behavioral change for throughput.
