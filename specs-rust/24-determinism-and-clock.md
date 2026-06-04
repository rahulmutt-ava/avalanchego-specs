# 24 — Determinism Audit & the Injectable Clock

> **Status:** Conforms to `00-overview-and-conventions.md`. This spec turns the
> §6.1 determinism rules into (A) an enforceable, PR-gate **audit checklist** and
> (B) a concrete **injectable clock / virtual-time abstraction**. It is a
> cross-cutting correctness artifact: every consensus/codec-affecting crate is in
> scope. It introduces no new wire formats — it constrains *how* the other specs
> compute, hash, and time things so a Rust node stays bit-for-bit with Go.

**Go source covered.** `utils/timer/mockable/clock.go` (the `Clock` injected
everywhere), `utils/timer/adaptive_timeout_manager.go`, the sort-before-serialize
sites (`codec/reflectcodec/type_codec.go`, `utils/sorting.go`,
`vms/components/avax/transferables.go`), the network clock-skew path
(`network/peer/peer.go`, `network/network.go`, `network/config.go`,
`utils/constants/networking.go`), the proposervm timing (`vms/proposervm/block.go`,
`vms/proposervm/pre_fork_block.go`), and the SAE gas-time discipline
(`vms/saevm/{gastime,proxytime,params}`, `cchain/tx/fx.go`).

**Cross-refs.** `00` §6.1 (the rules), §11.2 R1 (RNG parity); `01` §7.3/§7.4
(clippy + xtask lints); `02` §virtual time (tokio pause); `03` §10 (gonum
MT19937); `05` (handshake time check); `06` (timeout manager, windower); `11`
(Tau discipline); `17` (single-task-owns-consensus-state); `21` (fee/gas math).

---

## PART A — Determinism audit checklist

A node is deterministic iff: given identical inputs (messages, blocks, txs),
identical seed, and identical injected clock, two runs produce identical bytes,
hashes, votes, and persisted state — across machines and across N repeats. The
hazards below are the only known ways to break that. Every PR touching a
consensus/codec/VM crate is reviewed against this table; the **Auto** column says
whether CI can catch a violation mechanically.

> Legend for the *Enforcement* column: **clippy** = a clippy lint at `deny`;
> **xtask** = a custom `cargo xtask lint-determinism` grep/AST check (see §A.2);
> **golden** = a differential/golden test (`02`); **review** = human PR review
> (no mechanical check exists).

| # | Hazard | Rule (what you must do) | How to detect | Enforcement | Go reference behavior |
|---|--------|-------------------------|---------------|-------------|-----------------------|
| 1 | **Collection iteration → serialize/hash** | Never serialize, hash, or fold a `HashMap`/`HashSet` in iteration order. Use `BTreeMap`/`BTreeSet`, or collect + `sort` at the *exact* Go sort site. | grep for `HashMap`/`HashSet`/`dashmap`/`indexmap` field types in any `#[derive(Codec)]` type or any fn reachable from `encode`/`hash`. | xtask (ban `HashMap`/`HashSet` in codec-derive types) + clippy aux + review | `codec/reflectcodec/type_codec.go:463` sorts map keys before writing; `utils.Sort`/`SortByHash` (`utils/sorting.go`) and the `Sortable[T]` impls on `ids.ID`/`NodeID`/`ShortID`. |
| 2 | **Floating point on consensus/codec paths** | No `f32`/`f64` in any crate reachable from codec, hashing, block/tx build, voting, or sampling. (Allowed only in *metrics/log* values, never decisions.) | grep `f32`/`f64`/`as f64`/`.sqrt()` in consensus crates; deny `clippy::float_arithmetic` there. | clippy (`float_arithmetic = deny` in consensus crates) + xtask | Go has no floats on these paths. The one float in Go — `clockDifference` skew math (`peer.go:842`) and `TimeoutCoefficient` — is non-consensus (metrics/local timing), see §B.3. |
| 3 | **Integer arithmetic / overflow** | Use checked ops (`checked_add`/`checked_mul` → typed `Error`), never silent wrap. Use `U256` where Go uses `big.Int` (EVM/fee math, `21`); fixed `u64`/`u128` where Go is bounded. No `as` truncating casts on value-carrying ints. | clippy `arithmetic_side_effects`; SAE crates additionally `overflow-checks = true`. grep narrowing `as u32`/`as u64`. | clippy + xtask (G115 analogue, `01` §7.4) + review for `as` casts | `utils/math` safe ops everywhere; `gosec G115` gate on `vms/saevm`. |
| 4 | **RNG** | The *only* RNG on consensus paths is the vendored gonum **MT19937 / MT19937-64** (`03` §10). `rand::thread_rng`, `rand::random`, `StdRng`, `ChaCha` are forbidden on sampler/windower paths. Seeds are derived exactly as Go (`06` windower). | grep `thread_rng`/`rand::random`/`SmallRng`/`StdRng`/`OsRng` in consensus crates; require `ava_utils::mt19937`. | xtask (ban non-vendored RNG in `ava-utils` sampler / `ava-proposervm`) + golden | `utils/sampler/rand.go` uses `gonum prng.MT19937(_64)`; output stored in chain state (R1, `00` §11.2). |
| 5 | **Time** | Read time *only* through the injected `Arc<dyn Clock>` (Part B). `Instant::now()`, `SystemTime::now()`, `Utc::now()`, `tokio::time::Instant::now()` are forbidden outside `ava-utils::clock` and the binary's wiring. | grep `Instant::now`/`SystemTime::now`/`Utc::now`/`Local::now` outside the clock crate allowlist. | xtask (ban wall-clock outside `ava-utils::clock`) + review | `mockable.Clock` injected into network, timeout mgr, proposervm, every VM, SAE; tests call `clock.Set(...)`. |
| 6 | **Task / goroutine ordering** | Consensus state is owned by a *single* task (`17`, `06`); no observable output may depend on the order two concurrent tasks happen to be scheduled. No `FuturesUnordered` feeding an order-sensitive accumulator; no "first reply wins" into shared state without a deterministic tiebreak. | review of `tokio::spawn`/`select!`/`join_all`/`FuturesUnordered` near consensus state; `loom` tests for the few shared structures. | review + `loom`/golden (no mechanical grep) | Go funnels consensus through a single handler goroutine per chain (`snow/networking/handler`); the engine is not re-entrant. |
| 7 | **Set/map iteration in tx/block building** | When building a tx or block, inputs/outputs/credentials/validators are emitted in the Go-defined **sorted** order, not hash-map order. Sort with the same comparator Go uses (byte order / `Sortable`). | grep `.iter()`/`.values()`/`.into_iter()` over a map feeding a `push`/`extend` into a tx/block field; require an explicit `sort_*` first. | review + golden (tx-byte differential, `02`) | `transferables.go` sorts inputs/outputs; `utxo_fetching.go:74` sorts addrs "for pagination"; UTXO/input/output `Sortable` impls. |
| 8 | **SAE Tau / time discipline** | No raw-seconds arithmetic on SAE instants. Shift instants only by a `Duration` (`params::TAU`, `params::Tau·Lambda`). The `BlockInstant`/`GasTime` newtypes have **no** `Add<u64>` (`11` §2.3), so the mistake cannot compile. | type system (no `impl Add<u64>`); xtask grep for `+ TAU_SECONDS` / bare-second adds as defense-in-depth. | type-enforced + xtask (`tausecondslint` analogue) | Go `tausecondslint` CI grep bans `time.Add(...TauSeconds)`; must add `params.Tau` (a `Duration`). |
| 9 | **Serialization field order** | Field/struct encode order is fixed by the `#[derive(Codec)]` macro (declaration order), never by reflection over a map or by a runtime-computed order. Adding/reordering fields is a wire change → golden test. | derive macro is authoritative; golden codec vectors fail on any drift. | golden + review | Go linearcodec walks struct fields in declaration order; map keys sorted (#1). |

### A.1 Worked "don't do this" snippets

```rust
// HAZARD #1/#7 — nondeterministic: HashMap iteration order leaks into bytes.
let mut buf = Vec::new();
for (k, v) in &self.outputs_by_id {        // HashMap → order varies per run
    buf.extend(encode(k)); buf.extend(encode(v));
}

// CORRECT — sort at the exact Go site, then encode.
let mut ids: Vec<_> = self.outputs_by_id.keys().copied().collect();
ids.sort();                                 // ids::ID Ord == Go Sortable byte order
for id in ids {
    buf.extend(encode(&id));
    buf.extend(encode(&self.outputs_by_id[&id]));
}
```

```rust
// HAZARD #5 — wall-clock leak on a consensus path.
let now = std::time::SystemTime::now();     // ❌ xtask rejects outside ava-utils::clock

// CORRECT — read the injected clock.
let now = self.clock.unix();                // Arc<dyn Clock>, mockable in tests
```

### A.2 Enforcement strategy (what is mechanical vs review-only)

Three tiers, mirroring Go's split between `golangci-lint`, the `tausecondslint`
grep, and human review:

- **Clippy (`deny`), workspace-wide** — extend `00` §8 / `01` §7.3:
  `clippy::float_arithmetic` (deny in consensus crates, hazard #2),
  `clippy::arithmetic_side_effects` (already `warn`; raise to `deny` in
  consensus + all `ava-saevm*` crates, hazard #3), `clippy::cast_possible_truncation`
  (warn, hazard #3). These are pure clippy and need no custom tooling.

- **`cargo xtask lint-determinism`** — a custom workspace lint binary
  (the Rust analogue of Go's custom `scripts/lint.sh` greps + `tausecondslint`),
  run in CI's `lint-all` (`01`). It is a syntactic/AST pass with an allowlist file
  and covers what clippy cannot express:
  - **Ban wall-clock** (#5): no `Instant::now`/`SystemTime::now`/`Utc::now`/
    `Local::now`/`tokio::time::Instant::now` outside `ava-utils::clock` and the
    `avalanchego` bin's wiring module (allowlisted by path).
  - **Ban `HashMap`/`HashSet`/`IndexMap` fields in `#[derive(Codec)]` types** (#1):
    walk derive inputs; require `BTreeMap`/`BTreeSet` or a `#[codec(skip)]`/manual
    impl with a documented sort.
  - **Ban non-vendored RNG** (#4) in `ava-utils` (sampler) and `ava-proposervm`:
    no `rand::{thread_rng,random,rngs::*}` — must use `ava_utils::mt19937`.
  - **Tau / bare-second add** (#8): grep `+ *TAU_SECONDS` and `Duration::from_secs`
    fed into instant arithmetic, as defense-in-depth behind the type-level guard.
  The xtask emits file:line diagnostics and exits non-zero; allowlist entries
  require an inline `// determinism-allow: <reason>` comment, which the xtask
  verifies is present.

- **Review-only** — hazards #6 (task ordering) and the *semantic* half of #7/#9
  (is this the *same* sort comparator Go uses?). No grep can prove ordering
  independence; these rely on the single-task-owns-state architecture (`17`/`06`),
  on `loom` tests for the handful of shared structures, and on the differential
  harness (`02`) catching any byte drift.

> **Net:** of the 9 hazards, **#1, #2, #3, #4, #5, #8 are auto-enforceable**
> (clippy + xtask), **#9 is golden-test-enforced**, and **#6 plus the semantic
> correctness of #7 are review-only**. The checklist table is pasted into the PR
> template as a tick-box gate for any diff touching `ava-codec`, `ava-snow`,
> `ava-engine`, `ava-proposervm`, `ava-validators`, the `ava-*vm` crates, or
> `ava-utils` sampler/clock.

---

## PART B — Clock / time abstraction

### B.1 The rule

Library crates **never** read the wall clock ambiently. They take an
`Arc<dyn Clock>` handle, injected exactly where Go injects `mockable.Clock`.
The only place a real wall clock is constructed is the `avalanchego` binary's
wiring (and test harnesses). This makes hazard #5 enforceable and makes every
time-dependent behavior reproducible under a `MockClock`.

This is a direct port of `utils/timer/mockable/clock.go`, whose four methods
(`Time`, `UnixTime`, `Unix`, plus `Set`/`Sync` for testing) we reproduce as a
trait with a real impl and a mock impl.

```rust
// ava-utils::clock
// Copyright (C) 2019, Ava Labs, Inc. All rights reserved.
// See the file LICENSE for licensing terms.

use std::sync::Arc;
use std::time::{Duration, SystemTime, UNIX_EPOCH};

/// Mirrors Go `mockable.MaxTime` (`time.Unix(1<<63-62135596801, 0)`), used as a
/// "never" sentinel for deadlines. Nanoseconds dropped, matching Go.
pub const MAX_UNIX_SECS: u64 = (1u64 << 63) - 62_135_596_801;

/// A handle to "the current time", injected wherever Go injects `mockable.Clock`.
/// Object-safe so it can travel as `Arc<dyn Clock>`. All consensus/codec/VM time
/// reads go through this; never call `SystemTime::now()` directly (hazard #5).
pub trait Clock: Send + Sync {
    /// Wall-clock instant. Go `Clock.Time()`. May move backward (NTP step) —
    /// callers needing monotonicity use `monotonic()` (see §B.4).
    fn now(&self) -> SystemTime;

    /// Wall instant truncated to whole seconds. Go `Clock.UnixTime()`.
    fn unix_time(&self) -> SystemTime {
        let secs = self.unix();
        UNIX_EPOCH + Duration::from_secs(secs)
    }

    /// Unix timestamp in seconds, clamped to >= 0. Go `Clock.Unix()`
    /// (`max(t.Unix(), 0)` then `uint64`).
    fn unix(&self) -> u64 {
        self.now()
            .duration_since(UNIX_EPOCH)
            .map(|d| d.as_secs())
            .unwrap_or(0) // pre-epoch clamps to 0, matching Go's max(.,0)
    }

    /// Elapsed wall time since `earlier`, saturating at zero (Go often does
    /// `clock.Time().Sub(t0)`; e.g. proposervm `timeSinceBootstrapping`).
    fn since(&self, earlier: SystemTime) -> Duration {
        self.now().duration_since(earlier).unwrap_or(Duration::ZERO)
    }

    /// Monotonic reading for latency/timeout measurement (see §B.4). The real
    /// clock backs this with `Instant`; the mock derives it from advances.
    fn monotonic(&self) -> tokio::time::Instant;
}

/// Production clock: reads the OS wall clock and a process-monotonic instant.
/// This is the ONLY type allowed to touch `SystemTime::now`/`Instant::now`
/// (determinism-allow: this is the clock crate; xtask allowlists this module).
#[derive(Clone, Default)]
pub struct RealClock;

impl Clock for RealClock {
    fn now(&self) -> SystemTime {
        SystemTime::now() // determinism-allow: ava-utils::clock
    }
    fn monotonic(&self) -> tokio::time::Instant {
        tokio::time::Instant::now() // honors tokio pause() in tests (§B.2)
    }
}

/// Test clock: replaces Go `Clock.Set`/`Sync`. Faked time is shared so a test
/// can hold the `Arc<dyn Clock>` and still advance it.
#[derive(Clone, Default)]
pub struct MockClock {
    inner: Arc<parking_lot::Mutex<MockState>>,
}

#[derive(Default)]
struct MockState {
    /// `Some` == faked (Go `faked == true`); `None` == fall through to wall clock.
    faked: Option<SystemTime>,
    /// Monotonic base; advanced by `advance`, independent of the faked wall time.
    mono: Option<tokio::time::Instant>,
}

impl MockClock {
    /// Construct already faked at a fixed instant — the common test setup.
    pub fn at(t: SystemTime) -> Self {
        let s = Self::default();
        s.set(t);
        s
    }

    /// Go `Clock.Set(t)` — pin the clock to `t` and mark it faked.
    pub fn set(&self, t: SystemTime) {
        let mut g = self.inner.lock();
        g.faked = Some(t);
    }

    /// Go `Clock.Sync()` — stop faking, fall back to the wall clock.
    pub fn sync(&self) {
        self.inner.lock().faked = None;
    }

    /// Move faked time forward by `d` (no Go equivalent method; Go tests call
    /// `Set(old.Add(d))`). Also advances the monotonic reading by `d`.
    pub fn advance(&self, d: Duration) {
        let mut g = self.inner.lock();
        if let Some(t) = g.faked {
            g.faked = Some(t + d);
        }
        if let Some(m) = g.mono {
            g.mono = Some(m + d);
        }
    }
}

impl Clock for MockClock {
    fn now(&self) -> SystemTime {
        match self.inner.lock().faked {
            Some(t) => t,
            None => SystemTime::now(), // determinism-allow: ava-utils::clock
        }
    }
    fn monotonic(&self) -> tokio::time::Instant {
        let mut g = self.inner.lock();
        *g.mono.get_or_insert_with(tokio::time::Instant::now)
    }
}
```

`Arc<dyn Clock>` is cloned and stored by each subsystem that Go gives a
`mockable.Clock`. Constructors take `clock: Arc<dyn Clock>`; the node builds one
`RealClock` (wrapped `Arc`) and threads the *same handle* everywhere, so a single
NTP correction (or a single test `set`) is observed consistently across
subsystems — exactly the Go behavior where one `*mockable.Clock` is shared.

### B.2 Virtual time in async tests (composes with the injected clock)

Two independent time axes must both be controlled in tests, and they compose:

1. **The injected `Clock`** controls *content* time — block timestamps, the
   handshake `myTime`, SAE gas-time, validator start/end times. Tests use a
   `MockClock` and call `set`/`advance`.
2. **The tokio timer** controls *scheduling* time — `tokio::time::sleep`,
   `interval`, and the adaptive-timeout-manager's deadlines. Tests use
   `#[tokio::test(start_paused = true)]` (a.k.a. `flavor = "current_thread"` +
   `tokio::time::pause()`, per `02` §virtual time) and `tokio::time::advance(d)`.

`RealClock::monotonic()` / `MockClock::monotonic()` return a
`tokio::time::Instant`, so the timeout manager's latency measurement and its
deadline scheduling **both** honor `tokio::time::pause()`. The convention for a
deterministic timing test is to advance *both* axes together:

```rust
#[tokio::test(start_paused = true)]
async fn deadline_fires_after_timeout() {
    let clock = MockClock::at(unix(1_700_000_000));
    let mgr = AdaptiveTimeoutManager::new(cfg, clock.clone().into_arc());
    mgr.register(req_id, on_timeout);

    // advance virtual scheduling time AND content time in lock-step
    clock.advance(Duration::from_secs(2));
    tokio::time::advance(Duration::from_secs(2)).await;

    assert!(timed_out.load(Ordering::SeqCst));
}
```

> **Rule (from `02`):** tests never sleep on the wall clock and never read the
> real wall clock; they advance virtual time. The `MockClock` is what lets
> consensus-time and scheduling-time stay in sync.

### B.3 Clock-skew / NTP handling (faithful port)

Go does **not** run an NTP daemon; it tolerates skew and rejects peers that are
too far off. We reproduce this exactly (`05`).

- **Config:** `network-max-clock-difference`, default **1 minute**
  (`utils/constants/networking.go: DefaultNetworkMaxClockDifference = time.Minute`),
  validated `>= 0` (`config/config.go`). Port to `ava-config` /
  `ava-network::Config { max_clock_difference: Duration }`.

- **Handshake time check (`network/peer/peer.go`).** On receiving a `Handshake`:
  ```text
  local_unix      = clock.unix()
  clock_diff_secs = abs(msg.my_time as f64 - local_unix as f64)   // local metric only
  metrics.clock_skew_count.inc(); metrics.clock_skew_sum.add(clock_diff_secs)
  if clock_diff_secs > max_clock_difference.as_secs_f64() {
      // beacon peers logged at WARN, others DEBUG; then close the connection
      start_close(); return
  }
  ```
  Two notes for parity: (1) the skew comparison uses `f64` in Go but it is a
  **local connection-management decision, not consensus** — so it is *exempt*
  from hazard #2; keep it in `ava-network` (a non-consensus crate) and out of any
  codec/hash path. (2) The signed-IP timestamp is then verified against
  `max_timestamp = local_time + max_clock_difference`
  (`SignedIP.Verify`) — port verbatim.

- **`my_time` we send** is `clock.unix()` (the injected clock), so a `MockClock`
  drives the handshake deterministically in differential tests.

- **Health / SyncTime.** Go's `Clock.Sync()` reverts to wall time; there is no
  built-in NTP correction loop, and the network skew counter
  (`ClockSkewCount`/`ClockSkewSum`, `network/peer/metrics.go`) is the operator's
  signal. The Rust health check surfaces the same gauges (`18`) and, like Go, a
  large average skew is *reported*, not auto-corrected — operators fix NTP. There
  is no auto-`Sync`; `MockClock::sync()` exists only for tests.

- **Other `maxSkew` sites (proposervm, `06`).** `vms/proposervm/block.go` and
  `pre_fork_block.go` use a constant `maxSkew = 10 * time.Second`:
  `max_timestamp = vm.clock.now() + Duration::from_secs(10)` to bound a block's
  future timestamp, and `new_timestamp = vm.clock.now()` *truncated to seconds*
  when building. Port the constant verbatim and use `clock.unix_time()` for the
  truncation (it already truncates to whole seconds, matching Go
  `Truncate(time.Second)`). The xsvm example's `maxClockSkew = 10s` is the same
  pattern for VM block builders generally.

### B.4 Monotonic vs wall-clock guidance

| Subsystem | Needs | Why |
|---|---|---|
| Block/tx **timestamps**, handshake `my_time`, validator start/stop, SAE gas-time | **wall** (`clock.unix`/`unix_time`) | These are *content* — they go into hashed/persisted data and must agree network-wide; they may legitimately move backward on an NTP step. |
| **Adaptive timeout manager** latency + deadlines (`06` §5.4) | **monotonic** (`clock.monotonic` → `tokio::time::Instant`) | Latency must not go negative on a clock step; deadlines must fire on elapsed time. Go uses `time.Time` arithmetic but the values are local-only; we improve robustness with `Instant` while staying behavior-equivalent under test. |
| **Health "time since last accepted"**, rate limiters, gossip intervals | **monotonic** | Local liveness measures; never serialized. |
| **Recovery / catch-up "how long bootstrapping took"** (proposervm `timeSinceBootstrapping`) | **wall** to match Go's `clock.Time().Sub(t0)`, or monotonic if the value is purely local | Keep whichever the Go code uses for the value that influences a decision; differential-test it. |

Rule of thumb: **anything that gets hashed or sent on the wire uses wall time
via `clock.unix*`; anything that only measures local elapsed time for
liveness/timeouts uses `clock.monotonic`.**

### B.5 Go → Rust mapping

| Go | Rust |
|---|---|
| `mockable.Clock` (struct, value-injected) | `Arc<dyn Clock>` (object-safe handle) |
| `Clock.Time()` | `Clock::now() -> SystemTime` |
| `Clock.UnixTime()` (truncate to sec) | `Clock::unix_time() -> SystemTime` |
| `Clock.Unix()` (`max(.,0) as uint64`) | `Clock::unix() -> u64` |
| `t1.Sub(t0)` | `Clock::since(t0) -> Duration` (saturating) |
| `Clock.Set(t)` (test) | `MockClock::set(t)` / `MockClock::at(t)` |
| `Clock.Sync()` (test) | `MockClock::sync()` |
| (Go: `Set(old.Add(d))`) | `MockClock::advance(d)` |
| `mockable.MaxTime` | `MAX_UNIX_SECS` const |
| `time.Now()` ambient | **forbidden** outside `ava-utils::clock` (hazard #5) |
| `Truncate(time.Second)` | `unix_time()` / `Duration::from_secs(.unix())` |
| proposervm `maxSkew = 10*time.Second` | `const MAX_SKEW: Duration = Duration::from_secs(10)` |
| `DefaultNetworkMaxClockDifference = time.Minute` | `DEFAULT_MAX_CLOCK_DIFFERENCE: Duration = Duration::from_secs(60)` |

**Injection points (one `Arc<dyn Clock>` handle, same as Go's shared `*Clock`):**

- `ava-network`: `peer::Config`, `network::Network`, throttlers
  (`inbound_conn_upgrade_throttler`, `inbound_resource_throttler`), `ip_signer`,
  `p2p::throttler` — the handshake time check & IP-signing timestamps.
- `ava-engine` networking: `handler`, `message_queue`, `chain_router`, and the
  **adaptive timeout manager** (`utils/timer/adaptive_timeout_manager.go`).
- `ava-proposervm`: VM time for block build/verify (`maxSkew`, truncation).
- `ava-validators` / `ava-snow` uptime: `snow/uptime/manager.go`.
- `ava-utils::window` (the sliding window) and the indexer (`indexer/index.go`).
- VMs: `ava-platformvm` (`vm.go`, `txs/executor/backend`, `utxo/verifier`,
  `state/chain_time_helpers`, validators manager), `ava-avm` (`vm.go`, builder,
  spender), `ava-evm` uptime tracker, and **`ava-saevm`** gas-time
  (`cchain/tx/fx.go`'s `Clock()`), where the clock feeds the Tau/gas-time
  discipline (`11`, hazard #8).

### B.6 Test plan (references `02`)

- **Determinism proptest (the headline test).** For a generated workload
  (messages/txs/blocks) + a fixed RNG seed + a `MockClock` with a fixed start and
  a fixed advance schedule: run the subsystem **N times** (N≥16) and assert all
  runs produce *byte-identical* outputs (encoded blocks, vote bags, computed
  hashes, persisted state root). Property: `forall seed, workload, clock_script:
  run(seed, workload, clock) == run(seed, workload, clock)` across repeats and
  across at least two target triples (CI matrix). Lives in `tests/` per `02`.

- **No-wall-clock-leak lint test.** A CI step runs `cargo xtask lint-determinism`
  and asserts zero findings for hazards #1/#4/#5/#8; a unit test additionally
  greps the *compiled* crate's source for `SystemTime::now`/`Instant::now`
  outside the allowlist (belt-and-suspenders with the xtask).

- **Clock-skew tests (port Go `peer` tests).** Table test over `clock_diff` vs
  `max_clock_difference` boundary (`< max`, `== max`, `> max`) asserting
  connect/close and the skew metric increments; signed-IP timestamp verify
  against `local + max_clock_difference`.

- **Timeout-manager virtual-time test** (§B.2): `start_paused` + lock-step
  `clock.advance` / `tokio::time::advance`, asserting deadlines fire at the
  computed adaptive duration and latency feedback is monotonic.

- **Mock parity test.** `MockClock` reproduces every observable of Go's
  `clock_test.go`: `set` ⇒ faked, `sync` ⇒ wall, `unix` clamps pre-epoch to 0,
  `unix_time` truncates sub-second.

- **Differential (`02`).** The differential harness drives both nodes with the
  same `AdvanceTime(u64)` scenario step (`02` §) and asserts identical block
  timestamps and hashes — this is the integration-level guard for hazards #5/#8.

### B.7 Performance notes / improvements over Go

- `Arc<dyn Clock>` adds one vtable call per time read. On hot paths (per-message
  read loop) this is negligible vs syscall cost; where a tight inner loop reads
  time repeatedly, cache one `clock.now()` per loop iteration (matching Go, which
  also snapshots `clock.Time()` once, e.g. `localTime` in the handshake).
- `RealClock` is zero-sized; cloning the `Arc` is a refcount bump. No per-call
  allocation.
- Using `tokio::time::Instant` for `monotonic()` lets the **same** code path run
  under `start_paused` in tests with no branching, removing Go's need for separate
  mock-clock plumbing in timing tests and giving deterministic, instant
  (no real sleep) timeout tests.
- The determinism xtask is a syntactic pass over already-parsed source (reuse
  `syn`); it runs in the existing `lint-all` job with negligible added wall time,
  unlike Go's multiple `grep` passes.
