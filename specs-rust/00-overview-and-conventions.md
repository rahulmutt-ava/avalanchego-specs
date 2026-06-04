# 00 — Overview & Cross-Cutting Conventions

> **Status:** Canonical reference for the Rust port of `avalanchego`. Every other
> spec under `specs-rust/` MUST conform to the decisions recorded here. When a
> later spec needs to deviate, it must say so explicitly and justify it.

This document defines the goals, the workspace/crate layout, the canonical set of
external Rust dependencies, and the cross-cutting engineering conventions (error
handling, async, logging, metrics, configuration, serialization) shared by all
subsystems. Read it before reading or writing any other spec file.

---

## 1. Goal & non-goals

**Goal.** A from-scratch Rust implementation of an Avalanche node that is a
**drop-in replacement** for `avalanchego`: byte-for-byte wire/codec compatibility,
identical persisted on-disk formats where they are part of the network protocol,
the same CLI flags and subcommands, the same JSON-RPC / Connect APIs, and the same
consensus behavior. A Rust node must be able to join Mainnet, Fuji, and local
networks and interoperate with Go nodes in the same network, indistinguishably.

The EVM execution layer (C-Chain and EVM-based subnets) is built on
[**reth**](https://github.com/paradigmxyz/reth) instead of `libevm`/`coreth`.
The Merkle-trie state database is [**Firewood**](https://github.com/ava-labs/firewood)
used as a direct Rust dependency (no CGO/FFI shim — we link the Rust crate).

**Non-goals.**
- Not a re-architecture: we preserve protocol semantics. Performance improvements
  are in-scope only when they do not change observable network/consensus behavior
  (see §9).
- Not a 1:1 transliteration of Go idioms: we use idiomatic Rust (ownership, traits,
  `async`/`await`, `Result`) — see §7.
- The grafted EVM forks (`graft/coreth`, `graft/evm`, `graft/subnet-evm`) are
  **reference inputs**, not transliteration targets. We re-derive their *behavior*
  on top of reth.

### Compatibility surface (the contract we must not break)

| Surface | Requirement |
|---|---|
| P2P wire protocol | Exact: message IDs, framing, TLS handshake, peer-list gossip, ping/pong. See `05-networking-p2p.md`. |
| Codec | Exact: `avalanchego` linear codec (big-endian, length-prefixed, version-tagged). See `03-core-primitives.md`. |
| Block/Tx formats | Exact: P-Chain, X-Chain, C-Chain (incl. Atomic), SAE blocks. Hashes must match. |
| Merkle roots | Exact: `merkledb` root hashes; Firewood ethhash for EVM state. |
| JSON-RPC / Connect APIs | Exact: method names, request/response shapes, error codes. |
| CLI flags & config | Exact: every flag in `config/`, same defaults, same precedence (flags > env > file). |
| Genesis | Exact: produce identical genesis block IDs for Mainnet/Fuji. |
| gRPC plugin protocol (`rpcchainvm`) | Exact: a Rust VM plugin must speak the same `vm.proto`/`runtime.proto` so it can be hosted by a Go node and vice-versa. |

---

## 2. Source-of-truth mapping (Go tree → spec)

| Go area | Spec file |
|---|---|
| `ids/`, `codec/`, `utils/`, `version/`, `upgrade/`, `staking/` (crypto), `vms/components/verify` | `03-core-primitives.md` |
| `database/`, `x/merkledb`, `x/blockdb`, `x/archivedb`, Firewood | `04-storage-and-databases.md` |
| `network/`, `message/`, `proto/p2p`, `proto/sdk` | `05-networking-p2p.md` |
| `snow/` (consensus, engine, networking, validators, uptime), `vms/proposervm`, `simplex/` | `06-consensus.md` |
| `vms/` framework (`manager.go`, `rpcchainvm`, `components`, `fx`, `secp256k1fx`, `txs`, `metervm`, `tracedvm`, `registry`), `chains/`, `proto/vm` | `07-vm-framework.md` |
| `vms/platformvm` | `08-platformvm-pchain.md` |
| `vms/avm`, `vms/nftfx`, `vms/propertyfx` | `09-avm-xchain.md` |
| `vms/evm`, `graft/coreth`, `graft/evm`, `graft/subnet-evm`, reth integration | `10-cchain-evm-reth.md` |
| `vms/saevm` (SAE / ACP-194) | `11-saevm.md` |
| `node/`, `app/`, `main/`, `config/`, `api/`, `indexer/`, `wallet/`, `genesis/`, `nat/`, `trace/`, `subnets/` | `12-node-config-api-wallet.md` |
| Nix, Bazel, layout, AGENTS.md/CLAUDE.md | `01-development-environment.md` |
| Testing (unit, property, differential, TDD) | `02-testing-strategy.md` |

---

## 3. Workspace & crate layout

A single **Cargo workspace** (also driven by Bazel — see `01`). Crate names are
prefixed `ava-`. The dependency direction is strictly downward (no cycles):
`primitives` → `storage`/`codec`/`crypto` → `consensus`/`network` → `vms` → `node`.

```
avalanche-rs/                      # repo root (workspace)
├── Cargo.toml                     # [workspace] members + shared [workspace.dependencies]
├── rust-toolchain.toml            # pinned stable toolchain
├── flake.nix / flake.lock         # dev environment (01)
├── MODULE.bazel / BUILD.bazel     # Bazel bzlmod + rules_rust (01)
├── AGENTS.md / CLAUDE.md
├── crates/
│   ├── ava-types/                 # ids, fixed-size byte arrays, errors, primitive newtypes
│   ├── ava-codec/                 # linear codec (de/ser), packer, codec manager
│   ├── ava-crypto/                # secp256k1, bls, hashing, staking certs/TLS
│   ├── ava-utils/                 # set, bag, sampler, linked, timer, math, wrappers, window
│   ├── ava-version/               # version + upgrade schedule (network upgrade times)
│   ├── ava-database/              # KV trait + backends (rocksdb, mem, prefix, version, meter, corruptable, rpc)
│   ├── ava-merkledb/              # path-based merkle trie (+ Firewood binding)
│   ├── ava-blockdb/               # x/blockdb
│   ├── ava-archivedb/             # x/archivedb
│   ├── ava-message/               # wire messages (p2p.proto + app messages)
│   ├── ava-network/               # peers, handshake, router, throttling, gossip, NAT
│   ├── ava-snow/                  # consensus context, decidable, consensus instances
│   ├── ava-engine/                # snowman + avalanche engines, bootstrap, getter, sender
│   ├── ava-validators/            # validator set, manager, state
│   ├── ava-proposervm/            # proposervm wrapper
│   ├── ava-simplex/               # simplex integration (if enabled)
│   ├── ava-vm/                    # ChainVM/BlockVM traits, common VM components, fx, metervm, tracedvm
│   ├── ava-vm-rpc/                # rpcchainvm: gRPC plugin host + guest (tonic)
│   ├── ava-secp256k1fx/           # the secp256k1 fee/transfer feature extension
│   ├── ava-platformvm/            # P-Chain
│   ├── ava-avm/                   # X-Chain (+ nftfx, propertyfx)
│   ├── ava-evm/                   # C-Chain / EVM subnet on reth (10)
│   ├── ava-saevm/                 # SAE VM (11) — workspace-within: sub-crates per subpackage
│   ├── ava-chains/                # chain manager (wires VMs into consensus)
│   ├── ava-api/                   # HTTP/Connect API servers + admin/info/health
│   ├── ava-indexer/               # tx/block indexer
│   ├── ava-wallet/                # wallet SDK (build/sign/issue)
│   ├── ava-genesis/               # genesis configs + genesis block generation
│   ├── ava-config/                # CLI flags + config loading (drop-in flag parity)
│   ├── ava-node/                  # node assembly (the library)
│   └── avalanchego/               # the binary (main) — name kept for drop-in invocation
├── proto/                         # .proto files (shared with Go); build.rs via tonic-build / prost
└── tests/                         # cross-crate integration, differential harness (02)
```

`ava-saevm` is large enough to be an internal sub-workspace mirroring
`vms/saevm/{sae,cchain,blocks,saexec,saedb,gastime,gasprice,proxytime,hook,adaptor,
txgossip,intmath,cmputils,types,params}` — see `11-saevm.md`.

---

## 4. Canonical external dependencies

These choices are **binding** across the workspace. Pin exact versions in
`Cargo.toml [workspace.dependencies]`; do not introduce a second crate for the same
job without amending this table.

### 4.1 Runtime & async

| Concern | Crate | Notes |
|---|---|---|
| Async runtime | `tokio` (full) | Multi-threaded scheduler. The Go node is heavily goroutine-based; map goroutines → tokio tasks. |
| Async utilities | `futures`, `tokio-util`, `async-trait` | `async-trait` for object-safe async traits until native async-fn-in-trait is sufficient. |
| Concurrency primitives | `parking_lot` (sync Mutex/RwLock), `tokio::sync` (async), `arc-swap`, `dashmap` | Prefer `parking_lot` for short non-async critical sections. |
| Cancellation / structured shutdown | `tokio_util::sync::CancellationToken` | Replaces Go `context.Context` cancellation. See §7.4. |

### 4.2 Serialization & RPC

| Concern | Crate | Notes |
|---|---|---|
| Avalanche linear codec | **hand-written** in `ava-codec` | Must be byte-exact; do NOT use `bincode`/`borsh`. serde-style derive macro provided by `ava-codec-derive`. |
| JSON (API) | `serde`, `serde_json` | API request/response types. |
| Protobuf | `prost`, `prost-types` | Generated from `proto/`. |
| gRPC | `tonic` (+ `tonic-build`) | `rpcchainvm` plugin protocol, `sync`, `sharedmemory`, etc. |
| Connect RPC | `tonic` + `tonic-web` / custom Connect shim | avalanchego exposes Connect endpoints; serve gRPC + gRPC-Web; add Connect-protocol compat layer where required. See `12`. |
| HTTP server/client | `axum` + `hyper` + `tower`, `reqwest` (client) | `gorilla/mux` + `gorilla/rpc` (JSON-RPC 2.0) → `axum` with a JSON-RPC 2.0 router shim in `ava-api`. |
| WebSocket | `tokio-tungstenite` | pub-sub API streams. |

### 4.3 Cryptography

| Concern | Crate | Notes |
|---|---|---|
| secp256k1 | `secp256k1` (Bitcoin Core libsecp256k1 bindings) | Matches `dcrd/secp256k1` semantics; need recoverable signatures + the exact pubkey/address derivation. Verify malleability rules match Go (`secp256k1fx`). |
| BLS12-381 | `blst` | Same library `avalanchego` uses (`supranational/blst`); guarantees signature/aggregate/PoP compatibility. Wrap in `ava-crypto::bls`. |
| SHA-256 / SHA-512 | `sha2` | |
| RIPEMD-160 | `ripemd` | address derivation (`hashing.PubkeyBytesToAddress`). |
| Keccak-256 | `sha3` / reth's `alloy-primitives` keccak | EVM + some IDs. |
| TLS / X.509 staking certs | `rustls` + `rcgen` (cert gen) + `x509-parser` | Staking certs and node-ID derivation MUST match Go exactly (see `05`). Pin `rustls` crypto provider to match cipher suites Go offers. |
| Random | `rand`, `rand_chacha` | Deterministic sampling where required by consensus. |

### 4.4 Storage

| Concern | Crate | Notes |
|---|---|---|
| Merkle trie state | **`firewood`** (direct dep) | EVM state + (where applicable) avalanche merkledb-equivalent. See `04`. |
| Embedded KV (leveldb-equivalent) | `rust-rocksdb` | `goleveldb`/`pebble` are Go-specific; RocksDB is the closest production-grade embedded KV with the same semantics (ordered, prefix iteration, batch, snapshots). Provide a `Database` trait so backends are swappable. |
| In-memory KV | custom `BTreeMap`-backed (`ava-database::memdb`) | mirrors `memdb`. |
| EVM ancient/static files | reth `reth-db` / `reth-nippy-jar` | for EVM block/receipt storage. |

> **Decision — Pebble:** Go uses both `goleveldb` and CockroachDB `pebble`. There is
> no mature pure-Rust Pebble. We standardize on **RocksDB** as the on-disk default
> and keep the `Database` trait so a node started against a Go-written Pebble dir
> must be handled by an **import/migration** path, not in-place read (call this out
> in `04`; on-disk KV layout of the *generic* DB is not part of the network
> protocol, but bootstrapping from an existing Go data dir IS a migration concern).

### 4.5 EVM execution

| Concern | Crate | Notes |
|---|---|---|
| EVM | **reth** crates (`reth-evm`, `revm`, `reth-primitives`, `reth-provider`, `reth-rpc`, `alloy-*`) | C-Chain & EVM subnets. Customizations via reth's `ConfigureEvm`/`EvmConfig`, inspector hooks, and custom `PayloadBuilder`. See `10`. |

### 4.6 Observability & ops

| Concern | Crate | Notes |
|---|---|---|
| Logging | `tracing`, `tracing-subscriber` | Structured logging; replicate avalanchego log levels & per-chain log files. |
| Metrics | `prometheus` (the `prometheus` crate) | Must expose the same metric names/labels as Go where scraped by existing dashboards. |
| Tracing/OTel | `opentelemetry`, `opentelemetry-otlp`, `tracing-opentelemetry` | matches `trace/` (OTLP gRPC exporter). |
| Profiling | `pprof` (optional), `tokio-console` (dev) | matches the admin profile endpoints. |

### 4.7 Errors, config, misc

| Concern | Crate | Notes |
|---|---|---|
| Error types (libraries) | `thiserror` | typed errors per crate. |
| Error context (binary/top) | `anyhow` | only in `avalanchego` bin & tests. |
| CLI / flags | `clap` (derive) + custom layer | must reproduce avalanchego's flag names exactly; see `12`. Config precedence shim mirrors `spf13/viper`. |
| Config files | `serde_json`, `serde_yaml`, `toml` | avalanchego accepts JSON config; keep JSON authoritative. |
| Time | `std::time`, `chrono` (only for formatting) | Consensus time math must be integer/`Duration`-based (see SAE `Tau` rules). |
| Collections | `indexmap`, `smallvec`, `bytes` | `bytes::Bytes` for zero-copy buffers on the network path. |
| Property testing | `proptest` | primary PBT framework (see `02`). |
| Mocking | `mockall` | narrow, local mocks preferred (mirror Go's direction). |
| Parameterized/table tests | `rstest` | dev-only; expresses Go-style table tests (see `02`). |
| Test runner | `cargo-nextest` | canonical runner (see `01`/`02`); doctests via `cargo test --doc`. |
| Test assertions | `assert_matches`, `pretty_assertions` (dev) | |

---

## 5. Repository-protocol constants & magic values

All network-defining constants (codec versions, message type IDs, max sizes,
upgrade activation times, network IDs, fee/gas parameters, genesis bytes) MUST be
copied verbatim from the Go tree and centralized:

- Protocol/network constants → `ava-types` (e.g. `NetworkID`, `MaxMessageSize`).
- Codec versions → `ava-codec`.
- Upgrade times (`upgrade/`) → `ava-version`.
- Genesis → `ava-genesis` (embed the same genesis JSON/bytes the Go node embeds).

Each such constant must carry a doc-comment citing the Go source path it mirrors,
and be covered by a golden test (see `02`).

---

## 6. Go → Rust idiom mapping (project-wide)

| Go pattern | Rust port |
|---|---|
| `interface` with methods | `trait` (object-safe → `dyn Trait` behind `Arc`; otherwise generics). Async methods via `#[async_trait]` or native AFIT. |
| `struct` with method set | `struct` + `impl`. |
| `error` value + `if err != nil` | `Result<T, E>` with `?`; typed `E` via `thiserror`. Sentinel errors (`var ErrFoo = errors.New`) → enum variants; `errors.Is` → pattern match / `matches!`. |
| `context.Context` | explicit args: `&CancellationToken` for cancellation, `Span`/tracing for tracing, and pass deadlines as `Instant`. Never smuggle values through a context bag. |
| goroutine + channel | `tokio::spawn` + `tokio::sync::mpsc`/`broadcast`/`watch`. Unbuffered chan → `mpsc(1)` or `oneshot`. `select{}` → `tokio::select!`. |
| `sync.Mutex`/`RWMutex` | `parking_lot::Mutex`/`RwLock` (sync) or `tokio::sync::*` (held across `.await`). |
| `sync.Pool` | `object-pool` or a custom freelist; often unnecessary (ownership). |
| nil-able pointer | `Option<T>` / `Option<Arc<T>>`. |
| `[]byte` | `Vec<u8>`, `&[u8]`, or `bytes::Bytes` (network). Fixed arrays (`[32]byte`) → `[u8; 32]` newtypes. |
| `big.Int` | `ruint`/`alloy_primitives::U256` (EVM) or `num-bigint::BigUint` (general). Prefer fixed-width `u64`/`u128` where the Go code is bounded. |
| generics via `~`/codegen | Rust generics + trait bounds; avoid over-abstracting. |
| `go:generate` mocks | `mockall` (`#[cfg_attr(test, automock)]`). |
| struct tags (`serialize:"true"`) | `#[codec(...)]` derive attributes in `ava-codec-derive`. |

### 6.1 Determinism rules

Consensus-affecting code must be deterministic and match Go bit-for-bit:
- **Map iteration:** Go maps are unordered and avalanchego sorts before
  serialization. In Rust, never serialize a `HashMap` directly — sort keys (use
  `BTreeMap` or explicit `sort`) exactly where Go does.
- **Float:** forbidden in consensus/codec paths (none in Go either).
- **Integer overflow:** Go uses `utils/math` safe ops; Rust uses checked
  arithmetic (`checked_add` → typed error), never silent wrap. EVM uses `U256`.
- **Sampling/RNG:** the validator/consensus sampler must reproduce Go's sampling
  given the same seed (see `06`).

---

## 7. Cross-cutting engineering conventions

### 7.1 Error handling
- Each crate defines its own `Error` enum with `thiserror`; a crate-level
  `pub type Result<T> = std::result::Result<T, Error>;`.
- Preserve Go sentinel errors as variants and assert on them in tests where Go uses
  `errors.Is` (mirrors the repo's `ErrorIs` lint rule).
- `anyhow` only in `avalanchego` (bin) and test code.

### 7.2 Async & threading
- One `tokio` multi-thread runtime owned by `ava-node`. Library crates accept a
  handle / spawn onto the ambient runtime; they do not create their own runtimes.
- Blocking work (RocksDB, Firewood, crypto-heavy ops, EVM execution) runs on
  `spawn_blocking` or dedicated rayon pools — never block the async reactor.
- Hold no `std`/`parking_lot` lock across an `.await`.

### 7.3 Logging & metrics
- `tracing` everywhere; span names and fields mirror Go log messages so operators'
  greps keep working. Per-chain log routing as in Go.
- Every subsystem registers Prometheus metrics under the same namespace/subsystem
  names Go uses. A metrics-name golden test guards parity.

### 7.4 Cancellation & lifecycles
- `CancellationToken` threads through long-running tasks; graceful shutdown cancels
  the root token and awaits task joins (mirrors `node.Shutdown`).
- Components expose `async fn start`/`async fn shutdown`; the node orchestrates
  ordering identically to Go.

### 7.5 Configuration
- `ava-config` reproduces every flag and default. Precedence: CLI flag > env var
  (`AVAGO_…`) > config file > default — matching viper. A parity test diffs the
  generated flag list against a snapshot extracted from the Go `config/`.

### 7.6 Unsafe & FFI
- `#![forbid(unsafe_code)]` by default in every crate; opt out only where a binding
  (blst, firewood, rocksdb, secp256k1) requires it, isolated behind a safe wrapper
  module with a `// SAFETY:` rationale and property tests.

### 7.7 API stability of SAE
- `ava-saevm` mirrors the Go note: **no API-stability guarantees**, under active
  development. Hold it to the stricter lint bar (clippy pedantic + overflow checks),
  analogous to the Go `lint-saevm` / gosec G115 pass.

---

## 8. Code style & lints (enforced)

- Edition 2021 (or latest stable edition), `rustfmt` with a checked-in
  `rustfmt.toml`; `clippy` with `-D warnings`. SAE crates additionally enable
  `clippy::pedantic` + `overflow-checks = true` in all profiles.
- License header on every `.rs` file (mirror Go header.yml):
  ```rust
  // Copyright (C) 2019, Ava Labs, Inc. All rights reserved.
  // See the file LICENSE for licensing terms.
  ```
- Module layout: prefer `foo.rs` + `foo/` over `mod.rs` where it improves clarity.
- Public items documented (`#![warn(missing_docs)]` on library crates).
- No `unwrap()`/`expect()` in non-test library code except with a proven invariant
  documented inline.

---

## 9. Performance posture & noted improvements

High performance is a first-class requirement. General rules:
- Zero-copy on the network and codec hot paths (`bytes::Bytes`, borrow don't clone).
- Avoid per-message allocation; reuse buffers/pools on the P2P read/write loop.
- Parallelize verification where Go is serial but the work is independent
  (signature batches, tx verification) — **only** when it cannot change ordering of
  observable effects.

Each subsystem spec contains a **"Performance notes / improvements over Go"**
section. Candidate improvements identified during analysis are recorded there with
an explicit "safe because…" justification tied to the determinism rules (§6.1).
Examples to be validated per-subsystem:
- BLS aggregate-signature batch verification (`ava-crypto`).
- Lock-free / sharded validator-set reads (`ava-validators`).
- `spawn_blocking` + pipelining for Firewood commits decoupled from consensus
  (`ava-saevm`, per ACP-194's streaming async execution model).

Any such change must be backed by a differential test (`02`) proving identical
external behavior vs. the Go node.

---

## 10. Spec index

| File | Subsystem |
|---|---|
| `00-overview-and-conventions.md` | this file |
| `01-development-environment.md` | Nix flakes, Bazel (bzlmod + rules_rust + gazelle), layout, AGENTS/CLAUDE |
| `02-testing-strategy.md` | unit, property (proptest), TDD, differential testing |
| `03-core-primitives.md` | ids, codec, crypto, utils, version/upgrade |
| `04-storage-and-databases.md` | database backends, merkledb, blockdb, archivedb, Firewood |
| `05-networking-p2p.md` | network, message, peer/handshake, router, NAT |
| `06-consensus.md` | snow consensus & engines, validators, proposervm, simplex |
| `07-vm-framework.md` | VM traits, rpcchainvm, chains manager, fx, metervm/tracedvm |
| `08-platformvm-pchain.md` | P-Chain |
| `09-avm-xchain.md` | X-Chain (+ nftfx/propertyfx) |
| `10-cchain-evm-reth.md` | C-Chain / EVM subnets on reth |
| `11-saevm.md` | SAE / ACP-194 streaming async execution |
| `12-node-config-api-wallet.md` | node assembly, config/flags, APIs, indexer, wallet, genesis |
| `13-config-flags-reference.md` | exhaustive verbatim catalog of every CLI/config flag (name, type, default, env var, group) |
| `14-api-rpc-reference.md` | exhaustive catalog of every exposed API/RPC endpoint (info/admin/health/index/P/X/C-Chain/proposervm/Connect) |
| `15-serialization-and-wire-formats.md` | authoritative catalog: all protobuf/gRPC packages, linear codec, p2p framing/zstd, RLP, address/string encodings |
| `16-implementation-roadmap.md` | dependency-ordered milestones (M0–M9) with differential-harness exit criteria; risk burndown |
| `17-runtime-architecture.md` | tokio task/channel topology, backpressure, cancellation tree, shutdown ordering |
| `18-metrics-and-logging.md` | verbatim Prometheus metric catalog (parity surface) + the logging/tracing model |
| `19-state-sync-and-bootstrap.md` | the three-phase sync model: state-sync → bootstrap → consensus; per-VM matrix |
| `20-warp-icm.md` | Avalanche Warp / Interchain Messaging end-to-end (formats, signing, aggregation, verify, precompile) |
| `21-fee-economics-math.md` | all fee/economics formulas (ACP-103/176, AP3/AP4, staking reward, SAE gas-time) with worked vectors |
| `22-test-vectors-and-oracle.md` | the golden-vector corpus + manifest + Go extraction harness; static-vs-live oracle rule |
| `23-genesis-construction.md` | exact per-chain genesis byte/ID construction algorithm; expected genesis IDs per network |
| `24-determinism-and-clock.md` | the determinism audit checklist (PR gate) + the injectable clock / virtual-time abstraction |
| `25-key-management-and-signing.md` | staking-TLS & BLS key lifecycle; the local/remote signer abstraction (proto/signer) |
| `26-versioning-and-compatibility.md` | the version taxonomy + handshake compatibility matrix; the wire version string to report |
| `27-crash-consistency-and-recovery.md` | atomic-commit invariant, crash-point→recovery matrix, per-VM recovery, crash-injection tests |

---

## 11. Ratified cross-spec decisions & open risks

These were surfaced while authoring the subsystem specs and are **binding** across
the workspace. They refine, but do not override, §1–§10.

### 11.1 Ratified decisions

1. **rpcchainvm is NOT HashiCorp go-plugin.** Modern `avalanchego` (RPCChainVM
   protocol **v45**) uses a **reverse-dial handshake** over `proto/vm/runtime`: the
   host serves a `Runtime` gRPC server and passes its address via env
   `AVALANCHE_VM_RUNTIME_ENGINE_ADDR`; the plugin binds an ephemeral TCP listener
   and calls `Runtime.Initialize(protocol_version, addr)` back; the host then dials
   the VM. Plain insecure loopback gRPC — no magic cookie, stdout handshake, or
   mTLS. The plugin always *dials* the proxied callback services (rpcdb, appsender,
   sharedmemory, validatorstate, warp, aliasreader); the node always *serves* them.
   This symmetry is what makes Rust↔Go plugin interop work in both directions. See
   `07`.

2. **VM wrapping order (exact):** inner VM → `tracedvm` → `proposervm` → `metervm`
   → `tracedvm` → change-notifier, then `initialize`. See `07`/`06`.

3. **`Database` trait is synchronous**; the object-safe `Arc<dyn DynDatabase>` is the
   currency every subsystem passes around. Blocking calls are wrapped in
   `spawn_blocking` *at call sites*, never inside the trait. Only two sentinel
   errors: `Closed`, `NotFound`. See `04`.

4. **merkledb vs Firewood hashing:** Firewood default hashing = SHA-256, *byte-
   compatible* with `x/merkledb`; Firewood `ethhash` feature = Keccak-256 /
   Ethereum-MPT = EVM state root. Therefore: P/X-Chain on-wire roots use
   `ava-merkledb` (faithful reimpl, authoritative for the `merkle/sync` protocol);
   C-Chain / EVM-subnet / SAE state use `firewood` with `features=["ethhash"]`. See
   `04`/`10`/`11`.

5. **One EVM engine, two drivers.** `ava-evm` (reth/revm + Firewood-ethhash, spec
   `10`) is the *synchronous* C-Chain driver. SAE's `cchain` (spec `11`) is the
   *asynchronous* (ACP-194) driver. They **share** the revm executor and the
   Firewood state layout. **Requirement on `10`:** expose its revm
   `BlockExecutor`/executor factory and its Firewood `StateProvider` as reusable
   public APIs (not buried inside a sync-only `ChainVm`), so `ava-saevm-exec` can
   call them. Recorded in `10` §"Reuse contract".

6. **reth integration mode = reth-as-a-library executor**, not the Engine API.
   Snowman owns fork choice; we need the pre-commit state root to vote on, and
   Accept/Reject map to canonicalize/discard with no reorgs. See `10` (incl. the
   eight flagged gaps G0–G8 and their wrapper designs).

7. **Atomic-UTXO byte contract (ATOMIC-1).** `07` defines the canonical
   `avax.UTXO` + `secp256k1fx` output encodings *once*; P/X/C each register those
   fx type IDs so a UTXO exported by one chain decodes on the others via shared
   memory. Pinned by X↔P and X↔C differential tests. See `07`/`08`/`09`/`10`.

8. **Test infrastructure (from `01`/`02`):** task runner = **go-task** via
   `./scripts/run_task.sh`; test runner = **cargo-nextest** (`--profile ci`); proto
   codegen = **tonic/prost via `build.rs`, not committed** (Bazel uses
   `rust_prost_library`); mocks = **mockall, macro-generated, not committed**;
   dependency policy = **cargo-deny**; `Cargo.lock` is the single version source
   consumed by Bazel `crate_universe`. No generated artifacts are committed;
   dirty-tree CI gates guard `Cargo.lock`, `MODULE.bazel.lock`, and gazelle
   `BUILD.bazel` files. Bazel = bzlmod + `rules_rust` + `crate_universe` +
   `gazelle_rust`.

9. **Go `-race` has no direct Rust analogue.** CI substitutes overflow-checks +
   debug-assertions (`dev-checks`/`ci` profiles) as the "catch latent bugs" gate,
   with ThreadSanitizer/`loom` reserved for targeted concurrency tests. See `01`/`02`.

### 11.2 Open risks (must be resolved/gated before the dependent code merges)

- **R1 — Deterministic RNG parity (HIGHEST RISK).** The consensus validator
  sampler (`utils/sampler`, spec `03`/`06`) and the **proposervm windower** use
  gonum's **MT19937 / MT19937-64**, whose seeding schedule differs from both the Go
  stdlib and common Rust MT crates (`rand_mt`, `mersenne_twister`). The Rust port
  MUST reproduce gonum's stream **bit-for-bit** — likely a vendored/hand-ported
  MT19937 — and is gated on a golden-vector match against Go before any
  consensus-affecting code that depends on it can merge. Owner: `ava-utils`
  (sampler) + `ava-proposervm` (windower).

- **R2 — Pebble/LevelDB on-disk migration.** RocksDB replaces both; a node pointed
  at an existing Go data dir requires an *import* path, not in-place open
  (Pebble especially). The generic KV layout is not part of the network protocol,
  but bootstrapping from a Go data dir is. Specify/implement the migration tool;
  until then, document non-support. See `04` (R2 detail).

- **R3 — reth library API instability (gap G0).** reth's library crates have no
  stable public API and no first-class external-consensus entrypoint. Pin a
  vendored reth revision, wrap every touch-point behind `ava-evm` adapters, and
  budget for upstream churn. See `10`.

- **R4 — zstd compression is not byte-identical across libraries.** The one carve-
  out from byte-exactness: the differential harness asserts a Go node *accepts* our
  frames (and we decode theirs), not that compressed payloads are byte-identical.
  Decode is fully deterministic, so this is sound. See `05`.

- **R5 — Connect-protocol endpoint enumeration.** `connectproto/` services are
  routed generically through the shared h2c port in `12`; `05`/`07` must enumerate
  the specific Connect services they expose. Tracking item.

> **Convention for all specs:** each file begins with the Go source paths it covers,
> states the public Rust API (traits, key types, crate), enumerates invariants &
> protocol constants, gives the Go→Rust mapping for non-obvious pieces, lists the
> external crates it relies on (consistent with §4), defines its test plan
> (referencing `02`), and ends with a "Performance notes / improvements over Go"
> section. Cross-reference other specs by filename.
