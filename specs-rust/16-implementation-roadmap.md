# 16 — Implementation Roadmap (Dependency-Ordered, Milestone-Based)

> Conforms to `00-overview-and-conventions.md`. This spec sequences the work
> described by `03`–`15` into a strict, executable build order. It is the
> top-to-bottom backlog an agent (or team) follows. Every milestone ends in a
> **concrete, automatable EXIT CRITERION** expressed as differential/golden
> tests per `02-testing-strategy.md`, and every milestone is built **red→green**
> (`02` §2): the named exit tests are written *first* and must fail for the right
> reason before implementation begins.

This document is intentionally prescriptive. It does not re-specify any
subsystem — it references the owning spec (`03`–`15`) for *what* to build and
fixes the *order* in which subsystems become implementable, the *gate* that
proves each step done, and the *risk* (`00` §11.2) each step retires.

---

## 0. Guiding principles

1. **Bring-up order mirrors a real node.** Primitives → codec/crypto → storage →
   networking/handshake → consensus skeleton → VM framework → chains
   (P → X → C) → SAE → node/APIs → interop/hardening. Read-only network join
   (state-sync/bootstrap) is enabled early (M4) so the node can *observe* a real
   network long before it can *produce* on every chain.
2. **The dependency direction is downward and acyclic** (`00` §3):
   `primitives → storage/codec/crypto → consensus/network → vms → node`. A crate
   is "implementable" only once every crate below it on the DAG (§1) has passed
   its own exit gate.
3. **Every exit criterion is a named, automatable test.** Golden vectors
   (`02` §6), proptest invariants (`02` §4), the `dbtest`/`codectest`/consensus
   batteries (`02` §7), and ultimately the differential harness (`02` §11). A
   milestone is *not* done until its exit tests are green in CI.
4. **TDD is mandatory** (`02` §2, §13.8): the milestone's exit tests land in the
   first commit, failing. No speculative generality.
5. **R1 (deterministic RNG parity) is the highest risk and gates the most
   code** (`00` §11.2). It is retired in M0 so nothing consensus-affecting builds
   on an unverified RNG.

---

## 1. Build-order DAG (crate tiers)

Tiers are strict: a tier may only depend on tiers above it. Within a tier,
crates are independent and can be built in parallel. The "becomes implementable
after" column is the gate that unblocks the tier.

| Tier | Crates (`00` §3) | Driving spec(s) | Becomes implementable after | Retires |
|---|---|---|---|---|
| **T0 — Primitives** | `ava-types`, `ava-codec` (+`-derive`), `ava-crypto`, `ava-utils`, `ava-version` | `03`, `15` | nothing (root) | R1 (sampler RNG) |
| **T1 — Storage** | `ava-database`, `ava-merkledb`, `ava-blockdb`, `ava-archivedb` (+`firewood` wiring) | `04`, `15` | T0 (codec for node/proof encodings, crypto for hashing) | R2 (Go-dir migration), part of R3 (Firewood) |
| **T2a — Wire** | `ava-message`, `ava-network` | `05`, `15` | T0 (codec, crypto/TLS) | R4 (zstd), R5 (Connect enum) |
| **T2b — Consensus core** | `ava-snow`, `ava-engine`, `ava-validators`, `ava-proposervm`, `ava-simplex` | `06` | T0 (sampler/RNG), T1 (height/block stores), T2a (sender/router glue) | R1 (windower), part of R5 |
| **T3 — VM framework** | `ava-vm`, `ava-vm-rpc`, `ava-secp256k1fx`, `ava-chains` | `07`, `15` | T2b (Block, ChainVm ↔ engine), T2a (`rpcdb`/appsender proxies) | (interop wiring for R-final) |
| **T4 — VMs** | `ava-platformvm`, `ava-avm`, `ava-evm`, `ava-saevm` | `08`, `09`, `10`, `11` | T3 (fx, components, mempool, chain manager) | R3 (reth library churn) |
| **T5 — Node/APIs** | `ava-api`, `ava-indexer`, `ava-wallet`, `ava-genesis`, `ava-config`, `ava-node`, `avalanchego` (bin) | `12`, `13`, `14` | T4 (all chains), T2/T3 (handlers) | (flag/API/genesis parity) |
| **X — Cross-cutting** | `ava-differential` harness, metrics-parity, error taxonomy, observability, Bazel/Nix CI, `tools/extract-vectors` | `02`, `01` | runs **in parallel** from M0; the harness deepens each milestone | — |

```
            T0 primitives ── R1 retired here
           /      |       \
      T1 storage  T2a wire  (crypto/codec feed all)
           \      |       /
            T2b consensus core ── R1 (windower) confirmed
                  |
            T3 VM framework
                  |
   ┌──────────────┼───────────────┐
 T4 P-Chain   T4 X-Chain   T4 C-Chain ── T4 SAE
   └──────────────┼───────────────┘
                  |
            T5 node/config/api/wallet/genesis
                  |
            interop + hardening
   (X: ava-differential + CI run alongside every tier)
```

**Ordering decision — why P before C, with C deferrable.** The dependency DAG
allows P, X, C to be built in parallel once T3 lands. We sequence **P first**
(M4) because: (a) P-Chain serves `ValidatorState` (`08` §7), which the consensus
engine and proposervm windower (`06` §6.2, §7.3) consume to weight sampling for
*every* chain — without it no chain can be validated against the real network;
(b) P-Chain read-only sync (M4) is the cheapest way to *join* a real network and
exercise networking + consensus + bootstrap end-to-end against a Go node; (c) it
retires the R1 windower confirmation on live validator data. X precedes C (M5
before M6) only because X reuses the same `secp256k1fx`/UTXO/codec machinery as P
(near-zero new infra), making it a fast differential win, whereas C drags in the
entire reth/Firewood-ethhash stack (R3). **C-Chain may be pulled earlier
(swapped with M5)** if EVM/dApp compatibility is the business priority — it has
*no* dependency on X-Chain, only on T3 + Firewood-ethhash (T1) + atomic-UTXO
(`07` ATOMIC-1). The roadmap notes this as a sanctioned reorder, not a rewrite.

---

## 2. Milestones M0–M9

Each milestone: **goal**, **crates/specs**, **scope (what works at the end)**,
**EXIT CRITERIA (named tests)**, **new golden-vector sets**, **risks retired**.
Test names use the `02` conventions: `differential::*` (harness, `02` §11),
`golden::*` (vectors, `02` §6), `prop::*` (proptest, `02` §4), `conformance::*`
(battery, `02` §7).

| M | Goal | Crates / specs | Scope at end | EXIT CRITERIA (named tests) | New golden vectors | Risks retired |
|---|---|---|---|---|---|---|
| **M0** | Foundations | `ava-types`, `ava-codec`(+derive), `ava-crypto`, `ava-utils`, `ava-version` · `03`, `15` §4 | Byte-exact linear codec; cb58/bech32 addresses; secp256k1 (recoverable, low-S) + BLS (sign/agg/PoP); SHA/RIPEMD/Keccak; NodeID-from-cert; MT19937/-64 sampler **bit-for-bit** with gonum; upgrade schedule constants | `golden::codec_all_types` (every registered type's bytes); `golden::cb58_addr_bech32`; `golden::bls_sign_pop` + `golden::secp_recover`; `golden::nodeid_from_cert`; **`golden::sampler_mt19937_stream`** (Rust stream == gonum stream for fixed seeds — the R1 gate); `prop::codec_roundtrip` (cases=4096), `conformance::run_codec_suite` | `codec/`, `ids/` (cb58), `crypto/` (bls-pop, secp-recover, nodeid), `sampler/` (MT streams), `upgrade/` (activation times) | **R1** (RNG parity — *highest*) |
| **M1** | Storage | `ava-database`(+backends), `ava-merkledb`, `firewood` wiring, `ava-blockdb`, `ava-archivedb` · `04`, `15` | RocksDB/mem/prefix/version/meter/corruptable/rpc backends; path-based merkle trie with Go-exact root + proofs; merkle/sync proof codec; Firewood linked (SHA + ethhash features); block/archive stores; Go-data-dir import path documented | `conformance::run_database_suite` + `prop::db_oracle_btreemap` (every backend); **`golden::merkledb_root`** (fixed K/V sets → Go root); `golden::merkledb_proof` + `golden::range_proof` (wire-critical); `prop::merkle_order_independent_root`; `golden::firewood_ethhash_root` (vs Go EVM root); `prop::blockdb_roundtrip` | `merkledb/` (roots, proofs, range-proofs), `sync/` (proof wire), `firewood/` (ethhash roots) | **R2** (migration path scoped), part of **R3** (Firewood link) |
| **M2** | Networking handshake | `ava-message`, `ava-network` · `05`, `15` §3.1 | TLS upgrader matching Go cipher suites; message framing + op-code table + compression negotiation; Peer actor; Handshake→PeerList exchange; ping/pong/uptime; IP signing/gossip. **A Rust node TLS-handshakes a real Go node on Fuji, exchanges Handshake+PeerList, and stays connected.** | `golden::message_frames` (each op's wire bytes); `prop::frame_roundtrip` + fuzz `decode_never_overreads`; `prop::handshake_reaches_connected`; **`differential::interop_handshake`** (Rust node ↔ live Go node on Fuji: completes handshake, receives PeerList, holds connection ≥N s, no disconnect) | `message/` (per-op frames), `tls/` (node-id-from-cert handshake transcript) | **R4** (zstd accept-not-byte-equal), **R5** (network Connect svc enumerated) |
| **M3** | Consensus skeleton + VM framework | `ava-snow`, `ava-engine`, `ava-validators`, `ava-proposervm`, `ava-simplex`; `ava-vm`, `ava-vm-rpc`, `ava-secp256k1fx`, `ava-chains` · `06`, `07` | Snowball/Snowman instances; engine op state machine + bootstrap/getter/sender; validator set/manager/state; proposervm wrapper + windower; ChainVm/Block traits; fx + secp256k1fx; chain manager; rpcchainvm host+guest. A **no-op/test VM reaches finalization in a simulated multi-node cluster.** | `prop::consensus_safety` (no two conflicting decisions finalize, any vote seq); `prop::consensus_liveness` (coherent majority finalizes in bounded rounds); `prop::preference_monotone`; **`golden::proposervm_block`** (byte-exact block + signature); **`golden::windower_schedule`** (proposer ordering for fixed validator set — confirms R1 on the windower); `conformance::snow_battery`; `differential::testvm_finalizes` (simulated cluster reaches stable height) | `proposervm/` (blocks, windower schedules), `secp256k1fx/` (OutputOwners multisig) | confirms **R1** (windower); wires **R-final** interop scaffolding |
| **M4** | P-Chain read-only sync | `ava-platformvm` (+ engine bootstrap/state-sync) · `08`, `06` §4.3–4.4 | Parse/verify/accept all P-Chain tx & block types; flat-KV state + staker model + validator metadata (codec v2); reward/fee math; serve `ValidatorState`; **bootstrap from the network (state-sync or linear) and follow Fuji P-Chain to tip read-only.** | `golden::pchain_block_hash` + `golden::pchain_tx_codec` (all types); `prop::pchain_tx_roundtrip`; **`differential::pchain_sync_to_tip`** (sync Fuji P-Chain; at matching heights, last-accepted block ID + state hash + `getCurrentValidators` (sorted) == Go); `differential::validatorstate_parity` (windower-relevant view matches Go) | `platformvm/` (block hashes, tx codecs, reward vectors, validator-diff windows) | exercises full stack vs live net; locks `ValidatorState` contract for all chains |
| **M5** | X-Chain full issue/accept | `ava-avm` (+`nftfx`,`propertyfx`) · `09`, `07` ATOMIC-1 | Full tx model, fx dispatch, UTXO state, syntactic/semantic verify + executor, linearized Snowman blocks, tx gossip, X↔P atomic import/export; **issue and accept X-Chain txs.** | `golden::xchain_block_hash` + `golden::xchain_tx_codec`; **`differential::xchain_issue_tx`** (10k proptest-generated X-Chain tx programs → identical block IDs + UTXO sets vs a Go node); `differential::atomic_xp` (X↔P import/export UTXO bytes decode cross-chain) | `avm/` (block hashes, tx/op codecs), `atomic/` (cross-chain UTXO bytes) | proves ATOMIC-1 (X↔P); fastest differential win (reuses M0/M3 infra) |
| **M6** | C-Chain on reth | `ava-evm` (reth/revm + Firewood-ethhash + atomic txs) · `10`, `04` §4 | Snowman ChainVm adapter over reth executor; on-demand block building; Firewood-ethhash state backend; atomic tx import/export + atomic trie; dynamic fees per fork; warp + stateful precompiles; `eth_*`/`avax.*` RPC | `golden::cchain_block_wire` + `golden::cchain_genesis_root`; **`differential::cchain_state_root`** (reexecute recorded mainnet C-Chain block range → state roots match Go, `02` §10.5); `differential::atomic_xc` (X↔C atomic import/export parity); `prop::evm_fee_schedule_per_fork` | `cchain/` (block wire, state roots), reexecute fixtures (`blockexport`) | **R3** (reth library wrapped + pinned, gaps G0–G8 closed) |
| **M7** | SAE VM | `ava-saevm` (sub-workspace) · `11` | Streaming async execution pipeline (3 frontiers, Tau-discipline gas clock); SAE blocks + lifecycle; saedb (consensus vs execution state on Firewood revisions); adaptor → Snowman; recovery after restart; cchain-on-SAE; hooks/txgossip | `golden::sae_block_hash`; **`prop::sae_execution_determinism`** (same block stream → same settled state regardless of pipeline scheduling); `differential::sae_recovery` (crash+restart resettles to identical state, `11` §1.4); `differential::sae_streaming` (vs Go SAE node); invariants from `11` §10 as tests | `saevm/` (block hashes, settlement vectors, recovery transcripts) | retires SAE async-pipeline determinism risk; validates `00` §9 pipelined-commit optimization via differential |
| **M8** | Node / config / API / wallet / genesis | `ava-config`, `ava-genesis`, `ava-api`, `ava-indexer`, `ava-wallet`, `ava-node`, `avalanchego` bin · `12`, `13`, `14` | Full node assembly + lifecycle/shutdown ordering; every CLI flag with exact names/defaults/precedence; JSON-RPC 2.0 + Connect + WS APIs across all chains; indexer; wallet SDK; genesis generation. **A single `avalanchego` binary drop-in starts/stops like Go.** | **`golden::flag_parity`** (generated flag list == Go `config/` snapshot, `13` §25); **`differential::api_parity`** (every endpoint in `14`: structural-JSON-equal responses vs Go after normalization); **`golden::genesis_block_id`** (Mainnet + Fuji genesis block IDs == Go, `02` §6.2); `differential::indexer_parity`; `prop::config_precedence` (flag>env>file>default) | `config/` (flag snapshot), `genesis/` (block IDs + bytes), `api/` (per-endpoint response snapshots) | retires flag/API/genesis-parity risk; enables full e2e |
| **M9** | Plugin interop + hardening | `ava-vm-rpc`, all crates · `07` §5, `02` §10.3–10.5 | rpcchainvm Rust↔Go **both directions** (Rust VM hosted by Go node; Go VM hosted by Rust node); load/upgrade/reexecute suites; perf gating; mixed Go+Rust network | **`differential::plugin_rust_in_go`** + **`differential::plugin_go_in_rust`** (reverse-dial handshake v45, proxied services work both ways, `00` §11.1); **`differential::mixed_network`** (live Go+Rust nodes, all chains, no fork, same tip); `test-upgrade` (Go→Rust across activation height, incl. Go-dir→RocksDB import); `bench-guard` perf gates (`02` §9) | mixed-net interop transcripts, upgrade-continuity fixtures | **R-final** (drop-in acceptance, §4) |

---

## 3. Per-milestone TDD entry points (first failing tests to write)

The exact first red test for each milestone — written before any implementation:

| M | First failing test (red) | Why it's the right entry point |
|---|---|---|
| M0 | `golden::sampler_mt19937_stream` — assert the Rust MT19937 stream equals the committed gonum stream for seed `0` | R1 is the highest-risk gate; if the RNG is wrong, every later consensus test is invalid. Pin it first. |
| M1 | `golden::merkledb_root` for the empty trie, then a single-key trie | Root parity is the storage compatibility contract; smallest cases first, expand to the proptest order-independence property. |
| M2 | `golden::message_frames` for the `Handshake` op, then `differential::interop_handshake` | Frame bytes must be exact before a live Go node will accept the handshake. |
| M3 | `prop::consensus_safety` against an in-memory cluster of the test VM | Safety is the non-negotiable consensus invariant; write it before the engine exists. |
| M4 | `golden::pchain_tx_codec` for `AddPermissionlessValidatorTx`, then `differential::pchain_sync_to_tip` at height 0 (genesis) | Codec-exactness gates accept/verify; genesis-height sync proves the bootstrap loop before chasing the tip. |
| M5 | `differential::xchain_issue_tx` seeded with a single `BaseTx` (1 program), then scale to 10k | Start the differential program generator tiny, grow `cases`. |
| M6 | `differential::cchain_state_root` over a 1-block recorded range from genesis | Reexecute is the cheapest differential oracle (`02` §10.5); 1 block proves the executor+Firewood-ethhash wiring. |
| M7 | `prop::sae_execution_determinism` on a 2-tx block under two forced pipeline schedules | Determinism under async scheduling is SAE's core risk; force-schedule before optimizing. |
| M8 | `golden::flag_parity` (snapshot diff), then `golden::genesis_block_id` (Fuji) | Flag/genesis goldens are pure-function checks — fast, deterministic, catch drift immediately. |
| M9 | `differential::plugin_rust_in_go` — minimal Rust test-VM hosted by a Go node completes `Runtime.Initialize` reverse-dial | The handshake (`00` §11.1) is the interop linchpin; prove it before driving traffic. |

---

## 4. Cross-cutting workstreams (run in parallel to the critical path)

These are not milestones; they are **continuous** workstreams (tier **X**) that
start at M0 and deepen every milestone. They are staffed in parallel and gate
merges across the critical path.

| Workstream | Spec | Cadence | Gate it enforces |
|---|---|---|---|
| **Differential harness** (`ava-differential`) | `02` §11 | Built at M0 (recorded-oracle mode first, two-binary mode by M2); every subsystem adds an `Observation` collector as it lands (`02` §13.6) | Per-PR recorded-oracle diff + reexecute; nightly live two-binary + mixed net |
| **Golden-vector extraction** (`tools/extract-vectors`) | `02` §6.2 | One extractor per surface, written with each milestone's goldens | `vectors-drift` CI job: re-extract vs pinned Go commit, fail on diff |
| **Metrics-name parity** | `00` §7.3 | Each subsystem registers metrics; a golden `metrics_names` snapshot grows per milestone | A metrics-name golden test guards dashboard parity |
| **Error taxonomy** | `00` §7.1 | Per-crate `thiserror` enums; Go sentinels → variants as each crate lands | `assert_matches!` tests mirror Go `errors.Is` (`02` §3.1) |
| **Observability** (tracing/OTel) | `00` §7.3, `12` §7 | Span names mirror Go log messages; OTLP exporter wired in M8 | Operator greps keep working; trace endpoints parity |
| **Bazel/Nix CI** | `01`, `00` §11.8 | From M0: nextest `--profile ci`, `cargo-deny`, gazelle, dirty-tree gates on `Cargo.lock`/`MODULE.bazel.lock`/`BUILD.bazel` | Reproducible builds; TSan/`loom` jobs substitute for Go `-race` (`00` §11.9) |
| **Fuzzing corpora** | `02` §8 | A `cargo-fuzz` target per parser (codec M0, message M2, blocks M4–M7, merkledb M1) | `test-fuzz` smoke per PR; crash artifacts committed as seeds |
| **PORTING.md matrices** | `02` §10.1 | Seeded per crate from `go test -list`; updated each milestone | A subsystem is "done" only when its matrix has no `wip` rows |

---

## 5. Definition of "drop-in replacement" acceptance (final end-to-end)

The port is accepted as a drop-in replacement (`00` §1) only when **all** of the
following are simultaneously green — this is the M9 exit and the project's
definition of done:

1. **Joins Mainnet & Fuji.** A Rust `avalanchego` boots against Mainnet and Fuji
   with stock config, bootstraps all chains to tip, and tracks the tip
   indefinitely without forking from the Go network.
2. **Interoperates indistinguishably.** In a mixed Go+Rust network
   (`differential::mixed_network`), Go peers cannot distinguish a Rust node:
   identical handshake/PeerList behavior, identical accepted block IDs and
   state/merkle roots at every height across P/X/C/SAE, no fork.
3. **Passes the full differential suite.** Every `differential::*` test (`02`
   §11) — recorded-oracle per-PR and live two-binary nightly — passes at the
   target `cases` budget, including reexecute over recorded mainnet ranges
   (`02` §10.5).
4. **Flag parity.** `golden::flag_parity` shows zero diff vs the Go `config/`
   flag snapshot (names, types, defaults, env vars, precedence) (`13`).
5. **API parity.** `differential::api_parity` shows structural-JSON equality for
   every endpoint cataloged in `14` (info/admin/health/index/P/X/C/proposervm/
   Connect/WS) after nondeterminism normalization (`02` §11.4).
6. **Genesis parity.** `golden::genesis_block_id` reproduces the exact
   Mainnet and Fuji genesis block IDs and bytes (`02` §6.2).
7. **Plugin interop both directions.** `differential::plugin_rust_in_go` and
   `differential::plugin_go_in_rust` pass under RPCChainVM v45 (`00` §11.1).
8. **Upgrade continuity.** `test-upgrade` shows a Go→Rust swap across an
   activation height with no fork, including the Go-data-dir → RocksDB import
   path (`02` §10.4, `00` §4.4).
9. **Perf gates hold.** `bench-guard` (`02` §9) shows no regression beyond
   threshold on the critical-path benches; every `00` §9 optimization that ships
   is backed by a passing differential test.

---

## 6. Risk-burndown table (`00` §11.2 → retiring milestone)

| Risk | Description | Retired by | How it is proven retired |
|---|---|---|---|
| **R1** | Deterministic RNG parity — gonum MT19937/-64 sampler + proposervm windower must reproduce Go bit-for-bit (HIGHEST) | **M0** (sampler stream), confirmed **M3** (windower) | `golden::sampler_mt19937_stream` passes in M0 before any consensus code; `golden::windower_schedule` in M3 confirms the windower; `differential::validatorstate_parity` in M4 validates on live data |
| **R2** | Pebble/LevelDB → RocksDB on-disk migration; bootstrapping from a Go data dir needs an import path | **M1** (path scoped + documented), fully exercised **M9** | M1 documents non-support of in-place open and specs the import tool; `test-upgrade` in M9 exercises the Go-dir → RocksDB import across an upgrade |
| **R3** | reth library API instability (gap G0); no stable external-consensus entrypoint | **M1** (Firewood-ethhash link), fully **M6** | M1 links Firewood with `ethhash`; M6 wraps every reth touch-point behind `ava-evm` adapters, pins a vendored reth revision, and closes gaps G0–G8 — guarded by `differential::cchain_state_root` |
| **R4** | zstd compression not byte-identical across libraries | **M2** | `differential::interop_handshake` (and per-op frame tests) assert a Go node *accepts* our frames and we decode theirs — not byte-identical payloads (sound: decode is deterministic) |
| **R5** | Connect-protocol endpoint enumeration | **M2** (network Connect svcs), completed **M8** (API catalog) | `05`/`07` Connect services enumerated and routed by M2; M8's `differential::api_parity` covers the full Connect endpoint set from `14` |

---

## 7. Summary

Ten milestones (M0–M9) over five crate tiers (T0–T5) plus a continuous
cross-cutting tier (X). The critical path is **primitives → storage/wire →
consensus+VM-framework → P-Chain → X-Chain → C-Chain → SAE → node/APIs →
interop**. Three ordering decisions are load-bearing: **R1 (RNG parity) is
retired first in M0** so nothing consensus-affecting builds on an unverified
sampler; **P-Chain precedes X and C (M4)** because it serves the
`ValidatorState` every chain's consensus needs and is the cheapest read-only
network join; and **X precedes C (M5 before M6)** only to bank a fast
differential win on shared infra — C-Chain has no X dependency and may be pulled
ahead of X if EVM compatibility is prioritized. Every milestone exits on named
differential/golden tests and is built red→green per `02`.
