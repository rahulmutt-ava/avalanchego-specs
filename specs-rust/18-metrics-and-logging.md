# 18 — Metrics & Logging (Observability Parity)

> **Status:** Conforms to `00-overview-and-conventions.md` (§4.6 dependencies,
> §7.3 logging & metrics). This is a **verbatim parity catalog**: operators'
> Grafana/Prometheus dashboards scrape these metric names and label keys, so they
> are a compatibility surface on par with the CLI flags (`13`) and the APIs (`14`).
> Any rename, retyping, or label change is a **compatibility break**.

**Go source covered:** `api/metrics/` (`multi_gatherer.go`, `prefix_gatherer.go`,
`label_gatherer.go`), `utils/metric/` (`namespace.go`, `averager.go`),
`utils/logging/` (`level.go`, `format.go`, `factory.go`, `log.go`),
`trace/` (`tracer.go`, `exporter.go`, `exporter_type.go`), `node/node.go`
(namespace wiring), `chains/manager.go` (per-chain gatherers), and the
per-subsystem metric registrations in `network/`, `network/peer/`,
`network/throttling/`, `snow/networking/{handler,router,sender,timeout,benchlist}/`,
`snow/consensus/snowman/`, `database/meterdb/`, `vms/metervm/`,
`vms/rpcchainvm/`, `vms/platformvm/metrics/`, `vms/avm/metrics/`,
`utils/resource/`, `utils/timer/adaptive_timeout_manager.go`, `api/server/`,
`api/health/`.

**Rust crates:** `prometheus` (registry + `TextEncoder`), `tracing` +
`tracing-subscriber` (logging), `opentelemetry` + `opentelemetry-otlp` +
`tracing-opentelemetry` (tracing). Metric wiring lives in `ava-api` (the
`/ext/metrics` handler + `MultiGatherer`), each subsystem crate owning its own
metric structs. Logging factory lives in `ava-node` / a small `ava-logging`
module.

**Cross-refs:** `/ext/metrics` endpoint — `12 §3.6`. Logging/metrics/tracing
flags — `13 §15`/`§Tracing`. gRPC-proxied plugin VM metric merge — `07`
(rpcchainvm) and `15` (wire formats). OTel tracing of API/consensus — `12`.

---

## 1. Registry model

### 1.1 Go model

avalanchego does **not** use a single flat Prometheus registry. It builds a tree
of `prometheus.Gatherer`s rooted at a node-level `MultiGatherer` and exposes the
merged result at `/ext/metrics`.

- **Root multi-gatherer** (`api/metrics/multi_gatherer.go`): the node creates one
  `MetricsGatherer` (a `prefixGatherer`). Every subsystem calls
  `metrics.MakeAndRegister(gatherer, name)`, which mints a fresh
  `prometheus.Registry`, registers it under `name`, and hands the registry back to
  the subsystem to populate.
- **Namespace prefixing** (`prefix_gatherer.go` + `utils/metric/namespace.go`):
  on `Gather()`, the prefix gatherer rewrites each metric family name to
  `AppendNamespace(prefix, originalName)` = `prefix + "_" + originalName` (the
  separator is the single byte `'_'`, `NamespaceSeparator`). The prefix is
  rejected if it would create overlapping namespaces (`eitherIsPrefix`).
- **Node-level namespaces** are all `avalanche_<subsystem>` (`PlatformName =
  "avalanche"`). From `node/node.go`:
  `avalanche_api`, `avalanche_benchlist`, `avalanche_db`, `avalanche_health`,
  `avalanche_meterdb`, `avalanche_network`, `avalanche_process`,
  `avalanche_requests`, `avalanche_resource_tracker`, `avalanche_responses`,
  `avalanche_rpcchainvm`, `avalanche_system_resources`, `avalanche_upgrade`.
- **Per-chain metrics use a LABEL, not a chain-ID in the name**
  (`chains/manager.go`). The chain manager registers, for each per-chain
  subsystem, a **`labelGatherer`** (`label_gatherer.go`) keyed by the label
  `chain`. On `Gather()` it injects the label `chain="<primaryAlias>"` (e.g.
  `chain="P"`, `chain="X"`, `chain="C"`) into every family. The label-gatherer
  itself is registered into the prefix gatherer under the namespace, so the final
  exposed name is `avalanche_<namespace>_<metric>{chain="<alias>", …}`.
  Per-chain namespaces: `avalanche_avalanche` (DAG consensus),
  `avalanche_handler`, `avalanche_meterchainvm`, `avalanche_meterdagvm`,
  `avalanche_proposervm`, `avalanche_p2p`, `avalanche_snowman`, `avalanche_stake`,
  plus one **per VM name** `avalanche_<vmName>` (the VM's own metrics, e.g.
  `avalanche_platformvm_*`, `avalanche_avm_*`, `avalanche_evm_*`).

> **Parity note (correcting a common misconception):** modern avalanchego does
> **not** embed the chain ID inside the metric name (older nodes did). The chain
> identity is carried by the `chain` label whose value is the chain's **primary
> alias**. Dashboards filter on `chain="C"` etc. The Rust port MUST reproduce the
> label-based scheme exactly, including using the primary alias (not the raw
> 32-byte chain ID) as the value.

- **gRPC-proxied plugin VMs** (`vms/rpcchainvm/vm_client.go`): a plugin VM hosted
  out-of-process exposes its metrics over gRPC. The host's `VMClient` itself
  **implements `prometheus.Gatherer`** and is registered under the VM namespace
  with the chain label (`ctx.Metrics.Register("", vm)`). Its `Gather()` merges (a)
  the VM's own metrics fetched via the `vm.proto` `Gather` RPC and (b) a local
  `grpc_prometheus.ServerMetrics` set covering the proxied callback servers
  (rpcdb/appsender/sharedmemory/etc.). See `07`/`15` for the `vm.Gather` wire
  contract.

### 1.2 Rust design

```rust
// crate: ava-api (metrics module) — mirrors api/metrics/*.go
use prometheus::{Registry, Gatherer, proto::MetricFamily};
use std::sync::{Arc, RwLock};

/// Separator byte between a namespace prefix and a metric name. MUST be '_'
/// to match utils/metric/namespace.go (NamespaceSeparatorByte).
pub const NAMESPACE_SEP: &str = "_";
/// Application namespace root — utils/constants PlatformName.
pub const PLATFORM_NAME: &str = "avalanche";
/// Per-chain label key — chains/manager.go ChainLabel.
pub const CHAIN_LABEL: &str = "chain";

/// A gatherer that rewrites every family name to `<prefix>_<name>`.
/// Mirrors api/metrics/prefix_gatherer.go.
pub struct PrefixGatherer {
    inner: RwLock<Vec<(String, Arc<dyn Gatherer + Send + Sync>)>>,
}

impl PrefixGatherer {
    /// Register `gatherer` under `prefix`; rejects overlapping namespaces
    /// (eitherIsPrefix). Returns Err on collision.
    pub fn register(&self, prefix: &str, g: Arc<dyn Gatherer + Send + Sync>)
        -> Result<(), MetricsError> { /* … */ Ok(()) }
}

impl Gatherer for PrefixGatherer {
    fn gather(&self) -> Vec<MetricFamily> {
        let mut out = Vec::new();
        for (prefix, g) in self.inner.read().unwrap().iter() {
            for mut fam in g.gather() {
                fam.set_name(format!("{prefix}{NAMESPACE_SEP}{}", fam.get_name()));
                out.push(fam);
            }
        }
        out
    }
}

/// A gatherer that injects a constant label into every metric.
/// Mirrors api/metrics/label_gatherer.go — used for the per-chain `chain` label.
pub struct LabelGatherer {
    label_name: String,
    inner: RwLock<Vec<(String /*label value*/, Arc<dyn Gatherer + Send + Sync>)>>,
}

/// Mints a fresh sub-registry, registers it under `name`, returns it for the
/// subsystem to populate. Mirrors api/metrics/MakeAndRegister.
pub fn make_and_register(parent: &PrefixGatherer, name: &str)
    -> Result<Registry, MetricsError>
{
    let reg = Registry::new();
    parent.register(name, Arc::new(reg.clone()) /* Registry: Gatherer */)?;
    Ok(reg)
}
```

The `/ext/metrics` handler (`12 §3.6`) calls `root.gather()` and serializes with
`prometheus::TextEncoder` (`text/plain; version=0.0.4`). Per-chain wiring in
`ava-chains` mirrors `chains/manager.go`: one `LabelGatherer("chain")` per
per-chain namespace, registered into the root prefix gatherer; each chain
registers its `Registry` under its primary alias. A plugin VM's host adapter
(`ava-vm-rpc`) implements `Gatherer` and merges the VM's `Gather` RPC output with
its local `grpc`-server metrics — same as `VMClient`.

---

## 2. Metric catalog — the parity-critical families

> **Scope.** These are the **parity-critical families** that power the standard
> dashboards. They are not the complete set: the metric-name golden test (§3)
> snapshots the full Go `/ext/metrics` output and diffs it mechanically; any
> family the test discovers but this catalog omits is still binding. Types:
> C = Counter, G = Gauge, H = Histogram, Avg = an "averager" pair (see note).
> The exposed name is `<namespace>_<Name>`; the `Namespace` column gives the
> `avalanche_*` prefix. `[chain]` in the Labels column marks families that carry
> the per-chain `chain="<alias>"` label.

> **Averager note.** `utils/metric/NewAveragerWithErrs(name, …)` registers **two**
> metrics: `<name>_count` (Counter) and `<name>_sum` (Counter). In the tables
> below an `Avg` row named `foo` expands to `foo_count` + `foo_sum`. The Rust port
> uses a small `Averager` helper producing the identical pair.

### 2.1 Network — `avalanche_network_*` (`network/metrics.go`)

| Name | Type | Labels | Meaning |
|---|---|---|---|
| `peers` | G | — | Number of network peers |
| `tracked` | G | — | Currently tracked IPs being connected to |
| `peers_subnet` | G | `subnetID` | Peers validating a particular subnet |
| `time_since_last_msg_received` | G | — | ns since last msg received |
| `time_since_last_msg_sent` | G | — | ns since last msg sent |
| `send_fail_rate` | G | — | Portion of recently-failed sends |
| `times_connected` | C | — | Completed handshakes with a peer |
| `times_disconnected` | C | — | Disconnects after a handshake |
| `accept_failed` | C | — | Listener failed to accept inbound |
| `inbound_conn_throttler_allowed` | C | — | Allowed inbound connections |
| `tls_conn_rejected` | C | — | Rejected: unsupported TLS cert |
| `num_useless_peerlist_bytes` | C | — | Useless bytes in PeerList msgs |
| `inbound_conn_throttler_rate_limited` | C | — | Inbound rejected by rate-limit |
| `node_uptime_weighted_average` | G | — | Uptime weighted by observer stake |
| `node_uptime_rewarding_stake` | G | — | % stake deeming node reward-eligible |
| `peer_connected_duration_average` | G | — | Avg peer connection duration (ns) |

### 2.2 Per-peer message I/O — `avalanche_network_*` (`network/peer/metrics.go`)

Registered into the same network registerer (shares `avalanche_network`).

| Name | Type | Labels | Meaning |
|---|---|---|---|
| `round_trip_count` / `round_trip_sum` | Avg | — | Ping/pong round-trip time |
| `clock_skew_count` / `clock_skew_sum` | Avg | — | Observed peer clock skew |
| `msgs_failed_to_parse` | C | — | Received msgs that failed to parse |
| `msgs_failed_to_send` | C | `op` | Msgs that failed to send, by op |
| `msgs` | C | `io`,`op`,`compressed` | Msgs sent/received per op (`io`∈{`sent`,`received`}, `compressed`∈{`true`,`false`}) |
| `msgs_bytes` | C | `io`,`op`,`compressed` | Bytes on the wire per op |
| `msgs_bytes_saved` | C | `io`,`op` | Bytes saved by compression per op |

### 2.3 Throttling — `avalanche_network_*` (`network/throttling/*.go`)

| Name | Type | Labels | Meaning |
|---|---|---|---|
| `bandwidth_throttler_inbound_awaiting_acquire` | G | — | Inbound conns awaiting bandwidth |
| `buffer_throttler_inbound_awaiting_acquire` | G | — | Inbound awaiting a buffer slot |
| `byte_throttler_inbound_remaining_at_large_bytes` | G | — | At-large byte budget remaining |
| `byte_throttler_inbound_remaining_validator_bytes` | G | — | Validator byte budget remaining |
| `byte_throttler_inbound_awaiting_acquire` | G | — | Inbound awaiting byte acquire |
| `byte_throttler_inbound_awaiting_release` | G | — | Inbound awaiting byte release |
| `throttler_total_waits` | C | — | Inbound resource-throttler waits |
| `throttler_total_no_waits` | C | — | Inbound acquired without waiting |
| `throttler_awaiting_acquire` | G | — | Inbound awaiting CPU-resource acquire |
| `throttler_outbound_acquire_successes` | C | — | Outbound acquire successes |
| `throttler_outbound_acquire_failures` | C | — | Outbound acquire failures |
| `throttler_outbound_remaining_at_large_bytes` | G | — | Outbound at-large remaining |
| `throttler_outbound_remaining_validator_bytes` | G | — | Outbound validator remaining |
| `throttler_outbound_awaiting_release` | G | — | Outbound awaiting release |

### 2.4 Message handler — `avalanche_handler_*` (`snow/networking/handler/*.go`)

| Name | Type | Labels | Meaning |
|---|---|---|---|
| `expired` | C | `[chain]`,`op` | Messages expired before handling |
| `messages` | C | `[chain]`,`op` | Messages handled, by op |
| `message_handling_time` | G | `[chain]`,`op` | Time spent handling, by op (ns) |
| `locking_time` | G | `[chain]` | Time waiting on the handler lock |
| `count` | G | `[chain]` | Messages currently in the queue |
| `nodes` | G | `[chain]` | Nodes with ≥1 ready message |
| `excessive_cpu` | C | `[chain]` | Messages deferred for excessive CPU |

### 2.5 Chain router — `avalanche_requests_*` (`snow/networking/router/chain_router_metrics.go`)

Router and timeout metrics share the `requests`/`responses` node namespaces.

| Name | Type | Labels | Meaning |
|---|---|---|---|
| `outstanding` | G | — | Outstanding requests (all types) |
| `longest_running` | G | — | ns the longest outstanding request took |
| `dropped` | C | — | Dropped requests (all types) |

### 2.6 Timeout / requests — `avalanche_requests_*` & `avalanche_responses_*` (`snow/networking/timeout/metrics.go`, `utils/timer/adaptive_timeout_manager.go`)

| Name | Type | Labels | Namespace | Meaning |
|---|---|---|---|---|
| `messages` | C | `chain`,`op` | `responses` | Responses received per chain/op |
| `message_latencies` | G | `chain`,`op` | `responses` | Response latency per chain/op (ns) |
| `current_timeout` | G | — | `requests` | Current adaptive network timeout (ns) |
| `average_latency` | G | — | `requests` | Average network latency (ns) |
| `timeouts` | C | — | `requests` | Timed-out requests |
| `pending_timeouts` | G | — | `requests` | Pending (not-yet-fired) timeouts |

> The `responses` namespace's `chain` label here is part of the **metric's own
> labels** (the timeout manager registers a `…Vec` directly), distinct from the
> label-gatherer injection — but the exposed label key is identically `chain`.

### 2.7 Benchlist — `avalanche_benchlist_*` (`snow/networking/benchlist/benchlist.go`)

| Name | Type | Labels | Meaning |
|---|---|---|---|
| `benched_num` | G | — | Currently benched validators |
| `benched_weight` | G | — | Weight of benched validators |

### 2.8 Snowman consensus — `avalanche_snowman_*` (`snow/consensus/snowman/metrics.go`)

All carry the `[chain]` label.

| Name | Type | Labels | Meaning |
|---|---|---|---|
| `max_verified_height` | G | `[chain]` | Highest verified height |
| `last_accepted_height` | G | `[chain]` | Last accepted height |
| `last_accepted_timestamp` | G | `[chain]` | Unix-secs of last accepted block |
| `blks_processing` | G | `[chain]` | Currently processing blocks |
| `blks_accepted_container_size_sum` | G | `[chain]` | Cumulative accepted block bytes |
| `blks_rejected_container_size_sum` | G | `[chain]` | Cumulative rejected block bytes |
| `blks_polls_accepted_count` / `_sum` | Avg | `[chain]` | Polls from issuance→acceptance |
| `blks_accepted_count` / `_sum` | Avg | `[chain]` | Time issuance→acceptance (ns) |
| `blks_polls_rejected_count` / `_sum` | Avg | `[chain]` | Polls from issuance→rejection |
| `blks_rejected_count` / `_sum` | Avg | `[chain]` | Time issuance→rejection (ns) |
| `consensus_latencies` | H | `[chain]` | Issuance→acceptance, bucketed (8×1s linear) |
| `blks_build_accept_latency` | G | `[chain]` | Block-timestamp→accept time (ns) |
| `polls_successful` | C | `[chain]` | Successful polls |
| `polls_failed` | C | `[chain]` | Failed polls |

### 2.9 metervm (generic VM-method timing) — `avalanche_meterchainvm_*` / `avalanche_meterdagvm_*` (`vms/metervm/*.go`)

Every entry is an averager (`<name>_count` + `<name>_sum`), all `[chain]`-labelled.
ChainVM methods (`meterchainvm`): `build_block`, `build_block_err`, `parse_block`,
`parse_block_err`, `get_block`, `get_block_err`, `set_preference`, `last_accepted`,
`verify`, `verify_err`, `accept`, `reject`, `should_verify_with_context`,
`verify_with_context`, `verify_with_context_err`, `get_block_id_at_height`, and
(when the VM implements the extensions) `build_block_with_context[_err]`,
`set_preference_with_context`, `get_ancestors`, `batched_parse_block`,
`state_sync_enabled`, `get_ongoing_state_sync_summary`, `get_last_state_summary`,
`parse_state_summary[_err]`, `get_state_summary[_err]`. DAGVM methods
(`meterdagvm`): `parse_tx`, `parse_tx_err`, `verify_tx`, `verify_tx_err`,
`accept`, `reject`.

### 2.10 meterdb — `avalanche_meterdb_*` (`database/meterdb/db.go`)

| Name | Type | Labels | Meaning |
|---|---|---|---|
| `calls` | C | `method` | Calls to the DB, by method |
| `duration` | G | `method` | Time in DB calls (ns), by method |
| `size` | C | `method` | Bytes passed in calls, by method |

`method` ∈ {`has`,`get`,`put`,`delete`,`new_batch`,`new_iterator`,`compact`,
`close`,`health_check`,`batch_put`,`batch_delete`,`batch_size`,`batch_write`,
`batch_reset`,`batch_replay`,`batch_inner`,`iterator_next`,`iterator_error`,
`iterator_key`,`iterator_value`,`iterator_release`}.

### 2.11 VM-specific — `avalanche_platformvm_*`, `avalanche_avm_*`, `avalanche_evm_*`

Registered into the VM's `ctx.Registerer` ⇒ `avalanche_<vmName>_*` with `[chain]`.

PlatformVM (`vms/platformvm/metrics/*.go`):

| Name | Type | Labels | Meaning |
|---|---|---|---|
| `blks_accepted` | C/Avg | `[chain]`,`type` | Accepted P-Chain blocks by block type |
| `txs_accepted` | C/Avg | `[chain]`,`type` | Accepted P-Chain txs by tx type |
| `time_until_unstake` | G | `[chain]` | Time until this node unstakes |
| `time_until_unstake_subnet` | G | `[chain]`,`subnetID` | Time until subnet unstake |
| `local_staked` | G | `[chain]` | Stake controlled by this node |
| `total_staked` | G | `[chain]` | Total primary-network stake |
| `gas_consumed` | G | `[chain]` | Cumulative gas consumed |
| `gas_capacity` | G | `[chain]` | Current gas capacity |
| `active_l1_validators` | G | `[chain]` | Active L1 validators |
| `excess` | G | `[chain]` | Dynamic-fee excess |
| `price` | G | `[chain]` | Dynamic-fee price |
| `accrued_validator_fees` | G | `[chain]` | Accrued validator fees |
| `validator_sets_cached` | G | `[chain]` | Cached validator sets |
| `validator_sets_created` | C | `[chain]` | Validator sets created |
| `validator_sets_height_diff_sum` | G | `[chain]` | Σ height diffs computing sets |
| `validator_sets_duration_sum` | G | `[chain]` | Σ time computing sets (ns) |

AVM (`vms/avm/metrics/*.go`): `txs_accepted` (by `type`), `tx_refreshes`,
`tx_refresh_hits`, `tx_refresh_misses`.

EVM (C-Chain) metrics are produced by the reth/coreth driver (`10`); they appear
under `avalanche_evm_*` with `[chain]` and are diffed by the same golden test.

### 2.12 API server — `avalanche_api_*` (`api/server/metrics.go`)

| Name | Type | Labels | Meaning |
|---|---|---|---|
| `calls_processing` | G | `base` | In-flight API calls per base route |
| `calls` | C | `base` | Processed API calls per base route |
| `calls_duration` | G | `base` | Σ time handling calls (ns) per base |

### 2.13 Health — `avalanche_health_*` (`api/health/`)

| Name | Type | Labels | Meaning |
|---|---|---|---|
| `checks_failing` | G | `check`,`tag` | Currently-failing health checks |

### 2.14 Resource usage / tracking — `avalanche_system_resources_*` & `avalanche_resource_tracker_*`

System resources (`utils/resource/metrics.go`, `avalanche_system_resources`):

| Name | Type | Labels | Meaning |
|---|---|---|---|
| `num_cpu_cycles` | G | `processID` | Total CPU cycles |
| `num_disk_reads` | G | `processID` | Total disk reads |
| `num_disk_read_bytes` | G | `processID` | Total disk read bytes |
| `num_disk_writes` | G | `processID` | Total disk writes |
| `num_disk_write_bytes` | G | `processID` | Total disk write bytes |

The `resource_tracker` namespace exposes per-process/per-chain CPU & disk usage
gauges aggregated from the above (diffed by the golden test).

### 2.15 rpcchainvm / gRPC — `avalanche_rpcchainvm_*` and per-VM gRPC server metrics

The plugin host registers `grpc_prometheus.ServerMetrics` for the proxied
callback servers (rpcdb/appsender/sharedmemory/validatorstate/warp/aliasreader)
under `avalanche_rpcchainvm_*`. These follow grpc-prometheus naming
(`grpc_server_handled_total`, `grpc_server_handling_seconds`,
`grpc_server_msg_received_total`, etc.) and are merged via the `VMClient`
gatherer. The Rust host (`ava-vm-rpc`) must emit the **same** grpc-prometheus
family names; use a tonic interceptor mirroring grpc-prometheus, registered into
the same sub-registry.

---

## 3. Naming-parity rule & golden test

**Policy.** The set of (metric family name, type, label-key set) exposed at
`/ext/metrics` is a frozen compatibility surface. Adding a metric is allowed;
renaming, removing, retyping, or changing a label key of an existing one is a
**compatibility break** and must be flagged in review the same way an API or flag
change is.

**Golden test** (`02` differential harness):

1. Boot a Go `avalanchego` node (Fuji or local, all default chains P/X/C up) and
   `GET /ext/metrics`. Parse with a Prometheus text parser into the set
   `{(name, type, sorted(label_keys))}`. Drop per-scrape volatile *values*; keep
   only the schema. Snapshot it as the golden file.
2. Boot the Rust node identically and diff its `/ext/metrics` schema against the
   golden set. The test asserts the Rust set is a **superset** of every Go family
   (no Go family missing; label keys identical) and reports Rust-only families for
   manual triage.
3. Run it per network and per chain (so chain-labelled families are exercised with
   `chain="P"|"X"|"C"`).

The test must be allowed to ignore the runtime/process collectors' inherent gaps
documented in §4 (those are explicitly waived, not silently dropped).

---

## 4. Go runtime & process collectors (the honest gap)

`node/node.go` registers two stdlib collectors under `avalanche_process`:

- `collectors.NewProcessCollector(...)` — emits standard `process_*` families:
  `process_cpu_seconds_total`, `process_resident_memory_bytes`,
  `process_virtual_memory_bytes`, `process_open_fds`, `process_max_fds`,
  `process_start_time_seconds`, etc. (note: these are **not** prefixed with
  `avalanche_process_`; the process collector emits the canonical `process_*`
  names directly, and the prefix gatherer only renames families that don't already
  carry their own namespace — verify against the golden snapshot).
- `collectors.NewGoCollector()` — emits Go-runtime-specific `go_*` families:
  `go_goroutines`, `go_threads`, `go_gc_duration_seconds`, `go_memstats_*`,
  `go_sched_*`, etc. These describe the **Go GC and runtime** and have **no Rust
  equivalent**.

**Rust replacement (honest):**

| Go collector | Rust replacement | Parity |
|---|---|---|
| `process_*` | `prometheus::process_collector::ProcessCollector` (Linux) | **Full** — same `process_*` family names. On macOS the crate is a no-op; document that gap. |
| `go_*` (GC/runtime) | **No equivalent.** Optionally emit a small set of `rust_*` / `tokio_*` runtime gauges (e.g. via `tokio-metrics`): worker count, task counts, alloc stats. | **Gap.** Do **not** fake `go_*` names. |

The golden test's `process_*` parity is enforced on Linux (the production target);
the `go_*` families are listed in a **documented waiver** in the test so the diff
stays meaningful. We do not synthesize `go_gc_duration_seconds` etc. — emitting a
metric named after Go's GC from a Rust node would be dishonest and would mislead
GC-tuning dashboards. Dashboards that key off `go_*` must be ported to the
`rust_*`/`tokio_*` equivalents or to OTel runtime metrics.

---

## 5. Logging model

### 5.1 Levels (`utils/logging/level.go`)

Go defines 8 levels (`Verbo` = `iota - 9`, ascending): **Verbo, Debug, Trace,
Info, Warn, Error, Fatal, Off**. Note Go's unusual ordering: **Trace sits *above*
Debug** (Verbo < Debug < Trace < Info < …). Parsing is case-insensitive
(`verbo`/`debug`/`trace`/`info`/`warn`/`error`/`fatal`/`off`).

Map to `tracing` (which has only TRACE/DEBUG/INFO/WARN/ERROR):

| avalanchego | `tracing::Level` | Notes |
|---|---|---|
| Verbo | `TRACE` + a `verbo` target/marker | Most verbose; below Debug |
| Debug | `DEBUG` | |
| Trace | `DEBUG` (or a custom `trace` marker) | Go's Trace is between Debug & Info; emulate with a level filter, not native TRACE |
| Info | `INFO` | |
| Warn | `WARN` | |
| Error | `ERROR` | |
| Fatal | `ERROR` + process exit | tracing has no FATAL; log at ERROR then exit |
| Off | filter = off | `LevelFilter::OFF` |

Because `tracing` has 5 levels but avalanchego has 8, the port keeps a small
`AvaLevel` enum reproducing the 8 names + ordering, and renders the **exact level
string** (`verbo`/`trace`/`fatal`/…) into the log output. The `tracing-subscriber`
`EnvFilter`/per-layer filter is derived from `AvaLevel` so a config of
`--log-level=verbo` shows everything and `=trace` hides Debug/Verbo — matching Go.

### 5.2 Formats (`utils/logging/format.go`)

`--log-format` ∈ {`auto`, `plain`, `colors`, `json`}. `auto` ⇒ `colors` if stdout
is a TTY, else `plain` (`ToFormat(h, fd)`).

- **plain / colors:** console encoder, time layout `[01-02|15:04:05.000]`, level
  rendered via the level string (colorized in `colors`).
- **json:** zap JSON encoder with `EncoderConfig`:
  - `TimeKey = "timestamp"` (ISO8601), `LevelKey = "level"` (lowercase level
    string via `jsonLevelEncoder`), `NameKey = "logger"`, `CallerKey = "caller"`,
    `MessageKey = "msg"`, duration encoded as **nanoseconds (integer)**.

Reproduce the JSON line shape exactly:

```json
{"level":"info","timestamp":"2026-06-04T12:00:00.000Z","logger":"C","caller":"chain/foo.go:42","msg":"accepted block","height":1234}
```

Rust: a custom `tracing_subscriber::fmt` JSON layer (or hand-rolled
`FormatEvent`) that emits exactly these keys in this order, lowercased level
string, ISO8601 timestamp, and integer-nanosecond durations. Structured fields
(`tracing` event fields) become top-level JSON keys, mirroring zap's
`SugaredLogger` key/value pairs (00 §7.3: span/field names mirror Go so greps
keep working).

### 5.3 Per-chain log files & rotation (`utils/logging/factory.go`)

- The `LogFactory` opens one rotating writer per logger. The **main** logger
  writes `main.log`; `MakeChain(primaryAlias)` sets `LoggerName = primaryAlias`
  and `MsgPrefix = "<alias> Chain"`, producing **`<log-dir>/<alias>.log`**
  (e.g. `C.log`, `P.log`, `X.log`).
- Rotation uses `lumberjack`: `Filename = <log-dir>/<LoggerName>.log`,
  `MaxSize = log-rotater-max-size` (MB, default 8), `MaxBackups =
  log-rotater-max-files` (default 7), `MaxAge = log-rotater-max-age` (days,
  default 0 = keep all), `Compress = log-rotater-compress-enabled` (gzip).
- **Display vs file split:** two cores per logger — a **file** core at
  `--log-level` and a **stdout/display** core at `--log-display-level` (blank ⇒
  inherits `--log-level`). `--log-disable-display-plugin-logs` suppresses plugin
  output on stdout.

### 5.4 tracing-subscriber design (Rust)

```rust
// crate: ava-logging (used by ava-node) — mirrors utils/logging/factory.go
use tracing_subscriber::{layer::SubscriberExt, fmt, registry::Registry, Layer};
use tracing_appender::rolling; // size/age rotation handled by a lumberjack-like writer

/// Build the subscriber: a display layer (stdout, at display level) + one
/// rolling file layer per logger (file at file level). Format chosen by `fmt`.
pub fn init_logging(cfg: &LogConfig) -> Result<LogHandles, LogError> {
    let display = fmt::layer()
        .with_writer(std::io::stdout)
        .with_ansi(matches!(cfg.format, Format::Colors))      // auto→colors|plain
        .event_format(ava_format(cfg.format))                  // plain/colors/json
        .with_filter(level_filter(cfg.display_level));         // --log-display-level

    // Per-chain file appender; LoggerName == primary alias ⇒ "<alias>.log".
    let (main_writer, _g) = rolling_lumberjack(&cfg.dir, "main", &cfg.rotation);
    let main_file = fmt::layer()
        .with_writer(main_writer)
        .event_format(ava_format(Format::Json /*or per cfg*/))
        .with_filter(level_filter(cfg.file_level));            // --log-level

    let subscriber = Registry::default().with(display).with(main_file);
    tracing::subscriber::set_global_default(subscriber)?;
    Ok(/* reload handles for admin setLoggerLevel — 12 §3.5 */)
}

/// Per-chain logger added on chain creation (chains/manager MakeChain).
/// Spans tagged `chain=<alias>` route to this layer; file = "<alias>.log".
pub fn make_chain_logger(alias: &str, cfg: &LogConfig) -> Layer { /* … */ }
```

- `rolling_lumberjack` wraps `tracing-appender` (or a small custom writer)
  honoring `MaxSize`/`MaxBackups`/`MaxAge`/`Compress` to match lumberjack's
  filename-timestamp + gzip behavior.
- Per-chain routing: events carry a `chain` field (the alias); a layer filter (or
  per-chain layer keyed on the field) directs them to `<alias>.log`, matching
  Go's per-chain logger. The admin `setLoggerLevel` endpoint (`12 §3.5`) flips the
  per-logger filter via `tracing_subscriber::reload` handles.
- Levels rendered as the **exact avalanchego strings** (§5.1) so existing log
  greps and JSON parsers keep working.

---

## 6. Tracing / OpenTelemetry (`trace/`)

Cross-ref `12` (API/consensus spans) and `13 §Tracing` (flags). avalanchego wires
an OTLP exporter behind a `TracerProvider` sampled by ratio.

| Flag (`13`) | Effect |
|---|---|
| `--tracing-exporter-type` ∈ {`disabled`,`grpc`,`http`} | Selects OTLP transport (`trace/exporter_type.go`). `disabled` ⇒ no-op tracer. |
| `--tracing-endpoint` | OTLP collector endpoint (default per transport). |
| `--tracing-insecure` (default true) | No TLS to the collector. |
| `--tracing-sample-rate` (default 0.1) | `TraceIDRatioBased` sampler; ≥1 always, ≤0 never. |
| `--tracing-headers k=v` | Extra OTLP headers. |

**Rust:** `opentelemetry-otlp` exporter (gRPC via tonic / HTTP), wrapped by
`tracing-opentelemetry`'s `OpenTelemetryLayer` added to the subscriber from §5.4
when tracing is enabled. Sampler = `Sampler::TraceIdRatioBased(rate)`. Resource
attributes (service name `avalanchego`, version, network/node ID) mirror Go's so
spans group identically in the trace backend. The layer is disabled (not added)
when `--tracing-exporter-type=disabled`, matching Go's no-op tracer.

---

## 7. Performance notes / improvements over Go

- The Go `multiGatherer.Gather()` takes a global `RWMutex` for the whole scrape;
  the Rust `PrefixGatherer` can gather sub-registries concurrently (each
  `Registry::gather()` is independent and lock-light) and merge — a scrape-latency
  win on nodes with many chains, with **no name/label change** (the merged family
  set is identical, so §3 still passes). Sort the final family list to keep output
  byte-stable for snapshot diffing.
- Per-chain label injection (`LabelGatherer`) can pre-intern the `chain` label
  pair per chain instead of allocating per family per scrape.
- `tracing`'s static-level filtering compiles out below-threshold call sites
  cheaper than Go's per-call level check; combined with `valuable`/structured
  fields it avoids the per-log allocation Go's sugared logger incurs — observable
  only as lower overhead, never as a format/content change (§5 shape is frozen).
- Both improvements are gated by the golden tests (§3 for metrics, a JSON-line
  shape test for logs) per 00 §9.
