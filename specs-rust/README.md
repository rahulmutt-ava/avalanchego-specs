# specs-rust — Rust port of avalanchego

This directory is a **complete, standalone specification** for a from-scratch Rust
implementation of an Avalanche node that is a **drop-in replacement** for
`avalanchego`: byte-exact wire/codec/block compatibility, identical CLI flags and
APIs, the EVM layer rebuilt on [**reth**](https://github.com/paradigmxyz/reth), and
the Merkle state DB on [**Firewood**](https://github.com/ava-labs/firewood) as a
direct Rust dependency. A coding agent should be able to derive the implementation
from these documents.

## Read this first

**[`00-overview-and-conventions.md`](00-overview-and-conventions.md)** is the
canonical reference: goals, the compatibility surface, the Cargo workspace / crate
layout, the binding external-dependency table, the cross-cutting engineering
conventions, the Go→Rust idiom mapping, and (§11) the **ratified cross-spec
decisions and open risks**. Every other file conforms to it.

## Reading order

| # | File | Subsystem |
|---|------|-----------|
| 00 | [overview-and-conventions](00-overview-and-conventions.md) | goals, layout, deps, conventions, ratified decisions & risks |
| 01 | [development-environment](01-development-environment.md) | Nix flakes, Bazel (bzlmod + rules_rust + gazelle), task/test runners, AGENTS.md/CLAUDE.md |
| 02 | [testing-strategy](02-testing-strategy.md) | unit + proptest, TDD, golden vectors, fuzzing, the differential Go-vs-Rust harness |
| 03 | [core-primitives](03-core-primitives.md) | ids, the linear codec, crypto (secp256k1/BLS/staking certs), utils, version/upgrade |
| 04 | [storage-and-databases](04-storage-and-databases.md) | `Database` family, RocksDB/mem backends, merkledb, blockdb, archivedb, Firewood |
| 05 | [networking-p2p](05-networking-p2p.md) | wire protocol, messages, peers/handshake (TLS), router, throttling, NAT |
| 06 | [consensus](06-consensus.md) | Snowball/Snowman, engines, validators, proposervm, simplex |
| 07 | [vm-framework](07-vm-framework.md) | VM traits, rpcchainvm plugins, avax UTXO components, fx/secp256k1fx, chains manager |
| 08 | [platformvm-pchain](08-platformvm-pchain.md) | P-Chain: staking, subnets, L1s (ACP-77), validator state, warp |
| 09 | [avm-xchain](09-avm-xchain.md) | X-Chain: assets, UTXOs, nftfx/propertyfx, atomic import/export |
| 10 | [cchain-evm-reth](10-cchain-evm-reth.md) | C-Chain & EVM subnets on reth + Firewood-ethhash; atomic txs; warp/precompiles |
| 11 | [saevm](11-saevm.md) | SAE / ACP-194 streaming asynchronous execution; gas-as-time; the minimal C-Chain |
| 12 | [node-config-api-wallet](12-node-config-api-wallet.md) | node assembly, config/flags (drop-in parity), APIs, indexer, wallet, genesis |
| 13 | [config-flags-reference](13-config-flags-reference.md) | exhaustive verbatim catalog of every CLI/config flag — name, type, default, env var, Rust/clap type |
| 14 | [api-rpc-reference](14-api-rpc-reference.md) | exhaustive catalog of every exposed API/RPC endpoint with params/returns |
| 15 | [serialization-and-wire-formats](15-serialization-and-wire-formats.md) | authoritative catalog of all protobuf/gRPC packages, the linear codec, p2p framing/zstd, RLP, address/string encodings |
| 16 | [implementation-roadmap](16-implementation-roadmap.md) | dependency-ordered milestones M0–M9, each ending in an automatable differential/golden exit criterion; risk burndown |
| 17 | [runtime-architecture](17-runtime-architecture.md) | the tokio task/channel topology, backpressure, the cancellation tree, and exact shutdown ordering |
| 18 | [metrics-and-logging](18-metrics-and-logging.md) | verbatim Prometheus metric-name catalog (a parity surface) + the logging/tracing/OTel model |
| 19 | [state-sync-and-bootstrap](19-state-sync-and-bootstrap.md) | state-sync → bootstrap → consensus, the per-VM sync matrix, merkledb/EVM sync |
| 20 | [warp-icm](20-warp-icm.md) | Avalanche Warp / Interchain Messaging end-to-end: formats, signing, aggregation, verification, precompile |
| 21 | [fee-economics-math](21-fee-economics-math.md) | every fee/economics formula with worked integer vectors (consensus-critical) |
| 22 | [test-vectors-and-oracle](22-test-vectors-and-oracle.md) | the golden-vector corpus, its manifest, and the Go extraction harness that makes TDD executable |
| 23 | [genesis-construction](23-genesis-construction.md) | exact per-chain genesis byte/ID assembly; the expected genesis IDs per network |
| 24 | [determinism-and-clock](24-determinism-and-clock.md) | the determinism audit checklist (PR gate) + the injectable clock / virtual-time abstraction |
| 25 | [key-management-and-signing](25-key-management-and-signing.md) | staking-TLS & BLS key lifecycle; the local/remote signer abstraction |
| 26 | [versioning-and-compatibility](26-versioning-and-compatibility.md) | the version taxonomy + handshake compatibility matrix; the wire version string to report |
| 27 | [crash-consistency-and-recovery](27-crash-consistency-and-recovery.md) | atomic-commit invariant, crash-point→recovery matrix, per-VM recovery, crash-injection tests |

> Files 13–15 are reference catalogs (config, APIs, serialization). Files 16–17 are
> cross-cutting wiring (build order, runtime topology). Files 18–21 consolidate
> diffuse surfaces (metrics, sync, warp, fee math). Files 22–24 make the build
> executable & correct (test oracle, genesis, determinism). Files 25–27 close
> operational completeness (keys/signing, versioning, crash recovery).

## Conventions used by every spec

Each file: Go source paths covered → public Rust API (traits/types) → invariants &
protocol constants → Go→Rust mapping → external crates (consistent with `00` §4) →
test plan (per `02`) → "Performance notes / improvements over Go". Cross-references
use the filename.

## Highest-priority risks before implementation

See `00` §11.2. In short: **R1** — reproduce gonum's MT19937/MT19937-64 stream
bit-for-bit for the consensus sampler and proposervm windower (golden-gated);
**R3** — reth's library API is unstable (pin a vendored revision, wrap every
touch-point); **R2** — migrating from a Go Pebble/LevelDB data dir needs an import
tool.
