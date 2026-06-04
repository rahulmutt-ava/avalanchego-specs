# 02 — Testing Strategy (TDD, Property, Differential)

> **Status:** Binding for every crate in the Rust port. Conforms to
> `00-overview-and-conventions.md`. Where this spec sets a mandate (e.g. "every
> crate MUST ship proptest strategies + golden vectors"), later subsystem specs
> inherit it and may only *add* tests, never waive these requirements.

This document defines how the Rust port of `avalanchego` is tested. It mirrors
and improves on how `avalanchego` tests itself, translating each Go practice into
its idiomatic Rust equivalent and adding a **high-level differential harness** that
runs the Go and Rust binaries side-by-side under randomized input and asserts
identical observable behavior. The differential harness is the central deliverable
that proves the drop-in-replacement contract from `00 §1`.

---

## 0. Go source paths this spec covers (source of truth)

The Rust strategy is derived from these Go testing artifacts; each is the
reference for the corresponding Rust mechanism.

| Go artifact | What it establishes | Rust mirror |
|---|---|---|
| `scripts/build_test.sh` | canonical run: `-tags test -shuffle=on -race -timeout=120s -coverprofile` | `cargo nextest` profile + `cargo-llvm-cov` (§11) |
| `Taskfile.yml` (`test-*`, `test-fuzz`, `test-e2e`, `test-load`, `test-upgrade`, `*reexecut*`) | task surface for all suites | `cargo xtask test-*` aliases (§1, §9, §10) |
| `database/dbtest/{dbtest.go,benchmark.go}` | a backend-agnostic conformance + benchmark battery run against every KV backend | generic `run_database_suite<D: Database>()` (§7) |
| `codec/codectest/codectest.go` | a `GeneralCodec` conformance suite (`RunAll`) + `FuzzStructUnmarshal` | generic `run_codec_suite()` + proptest round-trip + fuzz target (§4, §6, §8) |
| `snow/snowtest/{context,decidable,status}.go` | shared consensus test fixtures | `ava-snow::testutil` fixtures (§5) |
| `vms/saevm/saetest/{saetest.go,wallet.go,heightdb.go,goleak.go,escrow}` | SAE test harness + leak checks | `ava-saevm::saetest` + leak assertions via tokio-console/`#[tokio::test]` drop checks (§5) |
| `tests/e2e/**` (ginkgo, `tests/fixture/e2e`, `tests/fixture/tmpnet`) | network-level e2e against a real binary via tmpnet | reuse tmpnet, drive from a Rust harness (§9, §10) |
| `tests/load/**` | sustained tx load + metrics assertions | Rust load generator on tmpnet (§10) |
| `tests/upgrade/**` | start old binary → upgrade to new → assert continuity | Rust upgrade harness (§10) |
| `tests/reexecute/{vm.go,c,blockexport,db.go}` | replay a recorded block range through a VM, assert state | Rust block re-execution harness — also a differential oracle (§10, §9) |
| Go `func FuzzX(f *testing.F)` (e.g. `codec/*`, `database/*`, `utils/cb58`, `utils/sampler`) | native Go fuzzing | `cargo-fuzz`/`libfuzzer` + `arbitrary` (§8) |
| testify `require`, table tests, `go.uber.org/mock` | unit-test idioms | `assert!`/`assert_eq!`/`assert_matches!`, array-of-struct loops or `rstest`, `mockall` (§2, §3) |

---

## 1. Principles & test taxonomy

Five layers, fastest→slowest, each gating the next in CI:

1. **Unit** (`#[cfg(test)]` in-module) — pure logic, no I/O. (§3)
2. **Property** (`proptest`) — invariants over generated inputs; **mandatory per
   crate**. (§4)
3. **Conformance / golden** (`insta` + committed vectors) — byte-exactness vs
   vectors extracted from the Go tree. **Mandatory for every protocol surface.** (§6)
4. **Integration** (per-crate `tests/` dir) — multi-component, real backends
   (RocksDB temp dir, in-process network). (§3.4, §7)
5. **Differential / e2e / load / upgrade** (`tests/` workspace, `tests/differential/`)
   — boot real binaries, compare against the Go oracle. (§9, §10)

**Runner.** `cargo nextest run` is the canonical runner (parallel, per-test
process isolation ≈ Go's `-shuffle`, stable JUnit output for CI). `proptest`/
`cargo-fuzz` targets that nextest cannot host run via `cargo test --doc` fallbacks
and dedicated `cargo xtask` tasks. The Go invariants we replicate:

| Go knob | Rust equivalent |
|---|---|
| `-tags test` (test-only code) | `#[cfg(test)]` + a `testutil` Cargo feature for cross-crate fixtures |
| `-shuffle=on` | nextest runs tests in randomized order by default; pin a seed in CI for repro |
| `-race` | run the suite under **ThreadSanitizer** (`RUSTFLAGS=-Zsanitizer=thread`, nightly) in a dedicated CI job, plus `loom` for lock-free code (§5) |
| `-timeout=120s` | nextest `slow-timeout`/`leak-timeout` in `.config/nextest.toml` |
| `-coverprofile`/`-covermode=atomic` | `cargo llvm-cov nextest` (§11) |

`cargo xtask` mirrors Taskfile task names so muscle memory carries over:
`test-unit`, `test-unit-fast` (no TSan), `test-fuzz`, `test-fuzz-long`,
`test-e2e`, `test-load`, `test-upgrade`, `test-reexecute`, `test-differential`.

---

## 2. Red/Green TDD methodology (mandated workflow)

Every functional PR follows **Red → Green → Refactor**, enforced by review and CI:

1. **Red.** Write the failing test first. For a new feature the test encodes the
   spec's invariant or a golden vector extracted from Go. Commit it (it must fail
   for the *right* reason — assert the failure message, not a compile error).
2. **Green.** Write the minimum implementation to pass. No speculative generality.
3. **Refactor.** Clean up under green tests; performance work (`00 §9`) happens
   here, always re-running the property + differential suites that guard behavior.

**Spec → test-first mapping.** Each subsystem spec ends with a *test plan*
referencing this file. That plan is the backlog: every invariant and every
protocol constant (`00 §5`) becomes a named test *before* the implementation
task that satisfies it. A subsystem is "done" only when its coverage matrix
(§10.x / §10-port) is green.

**Per-PR checklist (CI-enforced where possible):**
- [ ] New/changed behavior has a test that failed before the change (reviewer
      verifies via the PR's first commit).
- [ ] Touched crate's proptest strategies still hold; new invariants added.
- [ ] If a protocol surface changed, golden vectors regenerated *and reviewed*
      against Go (a vector diff is a red flag — it means we broke compatibility).
- [ ] Coverage did not regress below the crate's floor (§11).

---

## 3. Unit tests

### 3.1 Assertions — the `require` mapping

Go uses `require := require.New(t)` everywhere (the repo bans `assert`). Rust has
no analog object; the mapping is mechanical and is the project standard:

| Go `require` | Rust |
|---|---|
| `require.NoError(err)` | `let x = result.unwrap();` is **banned** in non-test, but in tests use `result.expect("ctx")` or `?` (tests return `anyhow::Result<()>`) |
| `require.Equal(want, got)` | `assert_eq!(got, want)` (order: actual, expected; use `pretty_assertions::assert_eq!` for big diffs) |
| `require.ErrorIs(err, ErrFoo)` | `assert_matches!(result, Err(Error::Foo { .. }))` |
| `require.True(b)` / `False` | `assert!(b)` / `assert!(!b)` |
| `require.Len(s, n)` | `assert_eq!(s.len(), n)` |
| `require.ElementsMatch(a, b)` | sort then `assert_eq!`, or `assert_unordered_eq!` helper |

Mirror the Go `ErrorIs` lint rule: when Go uses `errors.Is`, the Rust test uses
`assert_matches!` against the typed `Error` variant — never a string compare.

### 3.2 Table tests

The Go idiom is an array/map of structs iterated with `t.Run`. Two sanctioned Rust
forms; prefer the first for parity, the second when cases are heavy:

```rust
#[test]
fn add_overflow() {
    struct Case { name: &'static str, a: u64, b: u64, want: Result<u64, Error> }
    let cases = [
        Case { name: "ok",       a: 1,        b: 2, want: Ok(3) },
        Case { name: "overflow", a: u64::MAX, b: 1, want: Err(Error::Overflow) },
    ];
    for c in cases {
        let got = safemath::add(c.a, c.b);
        assert_eq!(got, c.want, "case {}", c.name);
    }
}
```

```rust
// Heavyweight / setup-per-case: rstest parameterization (mirrors map-of-struct).
#[rstest]
#[case::ok(1, 2, Ok(3))]
#[case::overflow(u64::MAX, 1, Err(Error::Overflow))]
fn add(#[case] a: u64, #[case] b: u64, #[case] want: Result<u64, Error>) {
    assert_eq!(safemath::add(a, b), want);
}
```

`rstest` is added to `[workspace.dependencies]` (dev) as the canonical
parameterized-test crate. `assert_matches` is already in `00 §4.7`.

### 3.3 Layout

- In-module unit tests: `mod tests { use super::*; ... }` under `#[cfg(test)]`.
- Cross-crate shared fixtures live behind a `testutil` feature (so they ship to
  dependents' tests without polluting release builds) — the Rust analog of Go's
  `*test` helper packages (`snowtest`, `saetest`, `codectest`, `dbtest`). These
  are *libraries of test helpers*, not test binaries.
- Integration tests live in each crate's `tests/` dir (one binary per file).

### 3.4 Async & determinism

- Async tests use `#[tokio::test(flavor = "multi_thread")]`; deterministic
  scheduling tests use `flavor = "current_thread"` + `tokio::time::pause()` for
  controllable virtual time (replaces Go's mock clocks in `utils/timer`).
- No wall-clock sleeps in tests; advance virtual time. Consensus-time tests use
  injected `Clock` traits (see `06`).

---

## 4. Property-based tests (proptest) — MANDATED per crate

**Every crate MUST ship a `proptest` suite.** Property tests encode the invariants
that golden vectors cannot exhaustively cover. Use `proptest` (`00 §4.7`),
not `quickcheck` (proptest has integrated shrinking + a persisted regression
corpus, which we require).

### 4.1 Shrinking & the committed regression corpus

proptest defaults to
`FileFailurePersistence::SourceParallel("proptest-regressions")`: when a property
fails, the minimal failing seed is written to `proptest-regressions/<path>.txt`
next to the source. **These files MUST be committed to git** so every later run
(and every other developer/CI machine) replays the known failures first. This is
the proptest analog of a fuzz corpus and is non-negotiable — a deleted regression
file is a test regression.

Standard config in a workspace `proptest.toml` (or per-`ProptestConfig`): bump
`cases` for protocol-critical crates (e.g. codec: `cases = 4096`), set
`max_shrink_iters`, and **never disable** `failure_persistence`.

### 4.2 Properties to test per subsystem (concrete, non-exhaustive)

| Crate / subsystem | Property |
|---|---|
| `ava-codec` | **round-trip**: `decode(encode(x)) == x` for all typed `x`; **byte-exactness**: `encode(x)` equals the Go-extracted golden bytes; decoding never panics on arbitrary `&[u8]`; length-prefix bounds enforced |
| `ava-types` (ids, cb58) | `decode(encode(id)) == id`; cb58 checksum round-trip (mirrors Go `FuzzEncodeDecode`); address derivation is a pure function of pubkey |
| `ava-crypto` | sign→verify always accepts; tampered message/sig always rejects; BLS aggregate(verify) == individual verifies; secp256k1 malleability rules match Go (low-S) |
| `ava-merkledb` / Firewood | inserting a set of K/V in **any order** yields the **same root**; `root(after delete-all) == empty root`; proof verifies iff key present; range-proof completeness |
| `ava-database` | a sequence of put/delete then iterate equals an in-memory `BTreeMap` oracle; batch == sequential apply; iterator with prefix == filtered oracle (the proptest form of the dbtest battery, §7) |
| `ava-utils::sampler` | weighted sampler reproduces Go's distribution for a fixed seed; without replacement never repeats; sample count == requested |
| `ava-utils::math` (safemath) | `checked_add`/`mul` overflow exactly where Go's `safemath` errors; never silently wraps |
| `ava-snow` (consensus) | **safety**: two conflicting decisions can never both finalize across any randomized vote sequence; **liveness**: with a sufficiently coherent majority, a value finalizes within bounded rounds; preference is monotone |
| mempool / `txpool` (avm/saevm) | ordering invariant: pop order is a total order by (gas price, nonce, arrival) identical to Go; add/remove idempotence; no tx lost across reorg |
| `ava-network` | message frame round-trips; handshake state machine reaches `Connected` for any valid ordering; never reads past declared length |
| `ava-genesis` | genesis bytes are a pure function of config; same config → same genesis block ID |

### 4.3 Sketch — codec round-trip + byte-exactness

```rust
use proptest::prelude::*;

proptest! {
    #![proptest_config(ProptestConfig { cases: 4096, ..ProptestConfig::default() })]

    // Invariant 1: round-trip identity for any value of a serializable type.
    #[test]
    fn utxo_roundtrip(u in arb_utxo()) {
        let bytes = codec::encode(&u, CODEC_VERSION).unwrap();
        let back: Utxo = codec::decode(&bytes).unwrap();
        prop_assert_eq!(back, u);
    }

    // Invariant 2: decode must never panic on arbitrary input (fuzz-lite).
    #[test]
    fn decode_never_panics(raw in proptest::collection::vec(any::<u8>(), 0..4096)) {
        let _ = codec::decode::<Utxo>(&raw); // Ok or Err, never panic/UB
    }
}

// Custom strategy producing a structurally valid Utxo.
fn arb_utxo() -> impl Strategy<Value = Utxo> {
    (any::<[u8; 32]>(), 0u32..16, any::<u64>())
        .prop_map(|(tx_id, out_idx, amount)| Utxo::new(tx_id.into(), out_idx, amount))
}
```

Byte-exactness lives in the golden layer (§6): the strategy generates the value,
and for a curated subset we additionally assert `encode(x)` equals a committed
vector — proptest covers *structure*, goldens pin the *bytes*.

---

## 5. Shared test fixtures & concurrency testing

- **Fixtures** mirror Go's helper packages as `testutil`-feature modules:
  `ava-snow::testutil` (context, decidable, status — from `snowtest`),
  `ava-codec::codectest`, `ava-database::dbtest` (§7), `ava-saevm::saetest`
  (wallet builder, height-db, escrow scenarios).
- **Goroutine-leak detection.** Go's `saetest/goleak.go` uses `uber/goleak`. The
  Rust analog: each async test that spawns tasks holds `JoinHandle`s and asserts
  they complete on a `CancellationToken` within `leak-timeout`; a
  `tokio_util::task::TaskTracker` is the structured-shutdown oracle. A test fails
  if the tracker isn't drained at shutdown.
- **`loom`.** Lock-free / atomics code (sharded validator set `00 §9`,
  `arc-swap` reads, mpsc shutdown) gets `loom` model-checking tests under
  `#[cfg(loom)]` — exhaustive interleaving exploration that complements the
  ThreadSanitizer (`-race`) job.

---

## 6. Conformance / golden tests (byte-for-byte vs Go)

These guard the compatibility surface in `00 §1`. **Every protocol surface MUST
ship golden vectors**: codec encodings, genesis block IDs (Mainnet/Fuji),
message framing, address derivation, merkle roots, BLS & secp256k1 signatures.

### 6.1 Mechanism

- Vectors stored under `tests/vectors/<surface>/` at the workspace root, checked
  into git, each with a `.json` manifest (input + expected hex output + a
  provenance note citing the Go source path and commit).
- Tests load the vector and assert byte-equality. For human-reviewable structured
  output use `insta` snapshots (`assert_snapshot!`/`assert_json_snapshot!`); for
  raw bytes assert `hex::encode(got) == vector.expected`. `insta`'s
  `cargo insta review` workflow makes intentional changes explicit — but for
  protocol bytes an unexpected snapshot change is a **compatibility break**, not a
  routine update, and the manifest's provenance note must be updated to match a
  Go change.

### 6.2 Procedure to extract golden vectors from the Go tree

A one-time-per-surface, repeatable extraction (scripted under
`tools/extract-vectors/`, runnable in CI to detect drift):

1. Add a tiny Go program / `_test.go` in a scratch dir importing the relevant Go
   package (codec, genesis, hashing, bls, message). It constructs canonical inputs
   and writes `{input, hexOutput}` JSON to `tests/vectors/<surface>/<case>.json`.
   Reuse existing Go test inputs verbatim where they exist (`codectest`,
   `genesis` test fixtures, `staking` cert fixtures).
2. For genesis: run the Go node's genesis generation for Mainnet & Fuji and record
   the resulting genesis block IDs + bytes.
3. For signatures: extract Go-produced secp256k1/BLS signatures over fixed
   messages with fixed keys; the Rust test re-verifies them and re-signs to assert
   determinism where the scheme is deterministic.
4. Commit vectors + provenance. A CI "vectors-drift" job re-runs extraction
   against the pinned Go submodule/commit and fails if `tests/vectors/` changes —
   ensuring vectors track the Go source of truth.

### 6.3 Examples of mandated golden surfaces

`codec` (every registered type), `ids` cb58, `genesis` (block IDs), `message`
framing (each message type's wire bytes), `crypto` (node-ID-from-cert,
addr-from-pubkey, BLS PoP), `merkledb` roots for fixed key sets, P/X/C/SAE block
hashes for fixed blocks.

---

## 7. Mocking & the Database conformance battery

### 7.1 Mocking (`mockall`)

Narrow, local mocks preferred (the Go repo is reducing mock usage). Use
`#[cfg_attr(test, mockall::automock)]` on traits, or `mock!{}` for foreign traits.

```rust
#[cfg_attr(test, mockall::automock)]
pub trait VmClient: Send + Sync {
    async fn get_block(&self, id: Id) -> Result<Block>;
}

#[tokio::test]
async fn engine_requests_missing_block() {
    let mut vm = MockVmClient::new();
    vm.expect_get_block()
        .withf(|id| *id == KNOWN_ID)
        .times(1)
        .returning(|_| Ok(Block::genesis()));
    let engine = Engine::new(Arc::new(vm));
    assert_matches!(engine.advance().await, Ok(_));
}
```

Prefer hand-written fakes (e.g. an in-memory VM) for stateful collaborators;
reserve `mockall` for interaction/expectation testing.

### 7.2 The reusable `Database` conformance suite (mirrors `dbtest`)

Go's `dbtest.Tests`/`TestsBasic`/`Benchmarks` are run against every backend
(`memdb`, `leveldb`, `pebbledb`, `prefixdb`, `versiondb`, `meterdb`,
`corruptabledb`, `rpcdb`). Rust replicates this as a single generic function
exported from `ava-database` under the `testutil` feature, invoked by each
backend's `tests/conformance.rs`:

```rust
// ava-database/src/dbtest.rs  (feature = "testutil")
pub fn run_database_suite<D, F>(new: F)
where
    D: Database,
    F: Fn() -> D,
{
    simple_key_value(new());
    overwrite(new());
    batch_atomicity(new());
    iterator_prefix(new());
    iterator_snapshot_isolation(new());
    delete_then_get_none(new());
    closed_db_errors(new());
    // ... full parity with dbtest.Tests + TestsBasic
}

// Property battery: any op sequence behaves like a BTreeMap oracle.
pub fn run_database_proptests<D, F>(new: F)
where D: Database, F: Fn() -> D + Clone {
    use proptest::prelude::*;
    proptest!(|(ops in proptest::collection::vec(arb_db_op(), 0..256))| {
        let db = new();
        let mut oracle = std::collections::BTreeMap::<Vec<u8>, Vec<u8>>::new();
        for op in &ops { apply(&db, &mut oracle, op); }
        prop_assert_eq!(dump(&db), oracle); // full-scan equality
    });
}
```

```rust
// ava-database backends: crates/ava-database/tests/conformance_rocksdb.rs
#[test]
fn rocksdb_conformance() {
    let dir = tempfile::tempdir().unwrap();
    ava_database::dbtest::run_database_suite(|| RocksDb::open(dir.path()).unwrap());
}
#[test]
fn rocksdb_proptests() {
    ava_database::dbtest::run_database_proptests(|| RocksDb::open_temp().unwrap());
}
```

`ava-codec` exposes the analogous `run_codec_suite()` (from `codectest`) and
`ava-snow` the consensus-instance battery. Backend-specific benchmarks reuse
`dbtest`'s benchmark inputs via criterion (§9).

---

## 8. Fuzzing (`cargo-fuzz` + `arbitrary`)

Mirror Go's `func FuzzX` targets and the `test-fuzz`/`test-fuzz-long` tasks. Each
crate with a parser/decoder ships fuzz targets in a `fuzz/` sub-crate using
`libfuzzer-sys` + the `arbitrary` crate (`00`-aligned dev deps).

**Mandated targets** (parity with Go fuzz tests, expanded):
- `ava-codec`: decode every registered type from arbitrary bytes (mirrors
  `FuzzStructUnmarshal`); plus a **round-trip differential** fuzz target:
  `decode→encode→decode` stability.
- `ava-message`: parse arbitrary wire frames (length-prefixed) — must never panic
  or over-read; mirrors network frame parsing.
- `ava-types` cb58: `FuzzEncodeDecode` analog.
- block parsers: P/X/C/SAE block `decode` from arbitrary bytes.
- `ava-merkledb`: arbitrary op stream against the oracle (structure-aware).

```rust
// crates/ava-codec/fuzz/fuzz_targets/decode_utxo.rs
#![no_main]
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    if let Ok(u) = ava_codec::decode::<ava_types::Utxo>(data) {
        // Round-trip must be byte-stable for anything that decoded.
        let re = ava_codec::encode(&u, ava_codec::VERSION).unwrap();
        let back: ava_types::Utxo = ava_codec::decode(&re).unwrap();
        assert_eq!(u, back);
    }
});
```

Structure-aware fuzzing uses `#[derive(arbitrary::Arbitrary)]` on input wrapper
types (e.g. a `Vec<DbOp>` for the merkledb target). Corpus lives in
`fuzz/corpus/<target>/` (committed seeds), crashes in `fuzz/artifacts/`.
`cargo xtask test-fuzz` runs each target briefly (smoke, like Go's `test-fuzz`);
`test-fuzz-long FUZZTIME=180` runs the long pass. Crash artifacts are committed
as regression seeds. (Requires nightly + LLVM sanitizers; runs in a dedicated CI
job on x86-64/aarch64 Linux.)

---

## 9. Benchmarks (`criterion`) & perf gating

- Every Go `func BenchmarkX` gets a `criterion` benchmark in the crate's
  `benches/` dir. Reuse the same input generators as the dbtest/codec batteries.
- `criterion` produces stable statistical comparisons; a CI "bench-guard" job runs
  the critical-path benches (codec encode/decode, merkledb commit, signature
  verify, mempool push/pop, message framing) and **fails on a >X% regression**
  vs the committed baseline (`criterion`'s `--save-baseline`/`cargo-criterion`
  comparison; threshold per-bench, default 10%).
- This is where `00 §9` performance claims (BLS batch verify, sharded validator
  reads, pipelined Firewood commits) are *measured*; any such optimization must
  show a bench win **and** pass the differential suite (§9 differential) proving
  no behavior change.

---

## 10. Porting Go tests + the suites (e2e / load / upgrade / reexecute)

### 10.1 Mandate: replicate ALL relevant Go tests

Every Go `_test.go` that asserts protocol/consensus/state behavior MUST have a
Rust counterpart. Pure-Go-idiom tests (testing Go-specific plumbing that doesn't
exist in the Rust design) are exempt and recorded as such.

**Tracking — the coverage matrix.** Each crate ships `tests/PORTING.md` (a table)
listing every Go test in the corresponding Go package, its Rust counterpart (or
"N/A — reason"), and status (`ported` / `wip` / `na`). A workspace
`cargo xtask porting-report` aggregates these into a single matrix and a percent-
ported number per subsystem. A subsystem's tests are "fully ported" when its
matrix has no `wip` rows and every non-`na` Go test maps to a passing Rust test.
CI publishes the matrix; specs reference their crate's row as the definition of
done.

A bootstrap script enumerates Go tests (`go test -list '.*' ./...`) to seed each
`PORTING.md` so nothing is silently missed.

### 10.2 e2e (reuse tmpnet)

Go e2e uses ginkgo + `tests/fixture/tmpnet` to spin a local multi-node network and
exercise it via the public API. **We reuse tmpnet as-is** (it manages networks of
*any* avalanchego-compatible binary via `AVALANCHEGO_PATH`): point it at the Rust
`avalanchego` binary. The Rust e2e harness lives in `tests/e2e/` and:
- shells out to / embeds tmpnet to start a network of N Rust nodes (and, for
  differential runs, mixed Go+Rust nodes — proving interop, `00 §1`),
- drives scenarios with the Rust `ava-wallet` + an API client,
- asserts via `require`-style assertions and golden API-response snapshots.
ginkgo's spec organization maps to nextest test functions grouped by module
(`banff`, `c`, `p`, `x`, `faultinjection`).

### 10.3 load

Mirror `tests/load`: a Rust load generator builds/issues a sustained tx stream
(C-Chain transfers, X/P-Chain txs) against a tmpnet network, scrapes Prometheus
(parity metric names, `00 §7.3`), and asserts throughput/latency SLOs + zero
errors. `cargo xtask test-load -- --load-timeout=30s`.

### 10.4 upgrade

Mirror `tests/upgrade`: start a network on the previous released **Go** binary,
then replace nodes with the **Rust** binary across an activation height, asserting
chain continuity and no fork. This doubles as a migration test for the
Go-data-dir → RocksDB import path (`00 §4.4`).

### 10.5 reexecute (also a differential oracle)

Mirror `tests/reexecute`: replay a recorded range of mainnet C-Chain (and P/X)
blocks through the Rust VM from a fixed starting state and assert the resulting
state/merkle roots match the recorded expected roots. Because the expected roots
come from the Go node, **this is a differential test on recorded data** — cheaper
than a live two-binary run and ideal for CI. Block export/import format reuses the
Go `blockexport` artifacts.

---

## 11. High-level differential testing — the key deliverable

A property-based external harness that boots **both** the Go `avalanchego` and the
Rust `avalanchego`, feeds identical randomized inputs, and asserts identical
observable outputs. This is the executable proof of the `00 §1` contract.

### 11.1 Location & shape

- Crate: **`tests/differential/`** (workspace member `ava-differential`, not
  published).
- Two run modes:
  - **Two-binary live** — start a Go network and a Rust network (via tmpnet) with
    identical genesis/config/seed; replay the same input program against both.
  - **Recorded-oracle** — replay against the Rust binary only and compare to
    Go-recorded outputs (the reexecute/golden path, §10.5). Cheaper, deterministic,
    runs on every PR; the live mode runs nightly + pre-release.

### 11.2 Input generation (proptest-driven)

A single `Strategy` produces an **input program**: a `Vec<Action>` plus a seed.

```rust
#[derive(Debug, Clone, arbitrary::Arbitrary)]
enum Action {
    IssueTx(TxSpec),          // P/X/C/SAE tx built deterministically from a seed
    ApiCall(ApiRequest),      // a JSON-RPC/Connect request from the supported set
    AdvanceTime(u64),         // bump both nodes' (mocked) clocks identically
    RestartNode(NodeIdx),     // crash + recovery sequence
    Partition(Vec<NodeIdx>),  // network partition then heal
    AwaitFinalization,        // quiesce: drive consensus to a stable height
}

fn arb_program() -> impl Strategy<Value = (u64 /*seed*/, Vec<Action>)> {
    (any::<u64>(), proptest::collection::vec(arb_action(), 1..200))
}
```

Tx/key material is derived deterministically from the program seed so both
networks receive *identical bytes*. The same seed is fed to the validator sampler
RNG on both sides (`00 §6.1`).

### 11.3 Oracle / comparison

After each `AwaitFinalization`, compare a **canonical observation** from both
networks; any mismatch fails and the seed is reported for replay:

| Observation | Source | Compared as |
|---|---|---|
| Last accepted block ID + height per chain | `info`/`platform`/C-Chain RPC | exact equality |
| State / merkle root per chain | RPC + reexecute roots | exact bytes |
| API responses for each `ApiCall` | the two HTTP endpoints | structural JSON equality after normalization (§11.4) |
| Validator set / uptime view | `platform.getCurrentValidators` | exact after sorting |
| Mempool acceptance/rejection of each tx | issue-tx response + later inclusion | identical accept/reject + identical inclusion order |
| Peer/handshake behavior (live mode) | network metrics + a Go↔Rust mixed net | both reach the same height; no fork |

### 11.4 Handling nondeterminism

The harness *removes* expected non-determinism before comparison so only protocol
divergence remains:
- **Timestamps / wall-clock:** both nodes run with an injected/controlled clock;
  `AdvanceTime` steps both identically. Timestamp fields in API responses are
  normalized out (or asserted only where they're consensus-relevant).
- **Node IDs / TLS certs:** generated from the seed and assigned identically to
  the i-th Go and i-th Rust node, so peer-dependent fields match. Where a field is
  inherently per-instance and not protocol-relevant, it's masked in the
  normalizer.
- **Map/iteration ordering:** normalize collections by sorting before compare
  (Go already sorts on the wire; this catches accidental Rust `HashMap` leakage,
  `00 §6.1`).
- **Async timing/log lines:** never compared; only quiesced, finalized state is
  compared (hence `AwaitFinalization` barriers).

### 11.5 Seed reproducibility

proptest persists the minimal failing `(seed, program)` to
`tests/differential/proptest-regressions/` (committed, §4.1). A failure prints a
`DIFFERENTIAL_SEED=<n>` line; `cargo xtask test-differential --seed <n>` replays a
single program deterministically (single-binary recorded mode for fast triage,
then live mode to confirm). The mismatching observation is dumped as a diff
(`pretty_assertions`).

### 11.6 Harness sketch

```rust
proptest! {
    #![proptest_config(ProptestConfig { cases: 64, ..ProptestConfig::default() })]
    #[test]
    fn go_and_rust_agree((seed, program) in arb_program()) {
        let cfg = NetworkConfig::deterministic(seed, /*nodes*/ 5);
        // Boot both networks with identical genesis/config (tmpnet-managed).
        let go   = Network::start(Binary::Go,   &cfg)?;
        let rust = Network::start(Binary::Rust, &cfg)?;
        let mut driver = LockstepDriver::new(seed); // derives all tx/key bytes

        for action in &program {
            driver.apply(&go,   action)?;
            driver.apply(&rust, action)?;
            if matches!(action, Action::AwaitFinalization) {
                let a = Observation::collect(&go)?.normalized();
                let b = Observation::collect(&rust)?.normalized();
                prop_assert_eq!(a, b, "divergence; replay DIFFERENTIAL_SEED={seed}");
            }
        }
        go.shutdown()?; rust.shutdown()?;
    }
}
```

`Binary::Rust` vs `Binary::Go` is just a different `AVALANCHEGO_PATH` handed to
tmpnet, so the same harness drives both. `Network`/`LockstepDriver`/`Observation`
live in `ava-differential` and are reused by the live and recorded modes.

### 11.7 CI integration

- **Per-PR:** recorded-oracle differential (fast, deterministic) on a small `cases`
  budget + the reexecute suite.
- **Nightly / pre-release:** live two-binary mode with larger `cases`, the mixed
  Go↔Rust interop network, and the upgrade suite.
- Any failure auto-files the seed + observation diff; the seed is committed to the
  regression corpus so it replays forever.

---

## 12. Coverage

- Tooling: **`cargo llvm-cov nextest`** (source-based coverage, the Rust analog of
  Go's `-coverprofile -covermode=atomic`); emit `lcov` + HTML; upload to the CI
  coverage service.
- **Targets (floors, CI-gated):** protocol-critical crates (`ava-codec`,
  `ava-types`, `ava-crypto`, `ava-merkledb`, `ava-database`, `ava-message`,
  consensus core) ≥ **90%** line coverage; VMs and node assembly ≥ **80%**;
  glue/CLI ≥ **70%**. A PR may not lower a crate below its floor.
- Coverage is necessary-not-sufficient: the property + golden + differential
  layers are the real correctness gate; coverage just catches untested code paths.

---

## 13. Cross-spec contract (others MUST honor)

1. **Every crate ships a proptest suite** with its committed
   `proptest-regressions/` corpus; deleting a regression file is a test regression.
2. **Every protocol surface ships golden vectors** under `tests/vectors/<surface>/`
   with provenance citing the Go source; a vectors-drift CI job tracks the Go
   source of truth.
3. **`ava-database` exposes the generic `run_database_suite`/`run_database_proptests`;
   every backend MUST pass them** (the dbtest contract). Same for
   `ava-codec::run_codec_suite` and `ava-snow` consensus batteries.
4. **Each crate maintains `tests/PORTING.md`**; "done" = matrix has no `wip` rows.
   `cargo xtask porting-report` aggregates.
5. **Parsers/decoders ship a `cargo-fuzz` target** (codec, message, blocks,
   merkledb, cb58); crash artifacts committed as regression seeds.
6. **The differential harness contract:** every subsystem exposes its state in a
   form the `ava-differential` `Observation` can collect and compare (block IDs,
   merkle/state roots, canonical API responses, validator views, mempool order).
   Any `00 §9` performance optimization MUST pass the differential suite.
7. **`#[tokio::test]` async tests must drain their `TaskTracker`** (no leaked
   tasks) and use virtual time, not wall-clock sleeps.
8. **TDD is mandatory:** the failing test lands in the PR's first commit; protocol
   changes that alter a golden vector are treated as compatibility breaks and
   reviewed as such.
