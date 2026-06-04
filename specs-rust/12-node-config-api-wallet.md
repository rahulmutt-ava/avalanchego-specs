# 12 — Node Assembly, Config/Flags, APIs, Indexer, Wallet, Genesis

> **Status:** Conforms to `00-overview-and-conventions.md` (binding). Cross-refs:
> `03` (ids/crypto), `04` (database backends), `05` (network config), `06`
> (consensus params), `07` (chains manager / rpcchainvm), `08`/`09`/`10`/`11`
> (P/X/C/SAE tx types reused by the wallet), `02` (testing harness).

## Go source paths covered

| Go area | Crate |
|---|---|
| `config/` (`config.go`, `flags.go`, `keys.go`, `viper.go`, `config.md`, `config/node/`) | `ava-config` |
| `node/`, `app/`, `main/` | `ava-node` + `avalanchego` (bin) |
| `api/` (`server/`, `info/`, `admin/`, `health/`, `metrics/`, `connectclient/`, `traced_handler.go`) | `ava-api` |
| `indexer/` | `ava-indexer` |
| `wallet/` (`chain/{p,x,c}`, `subnet/primary`) | `ava-wallet` |
| `genesis/` (`genesis_*.{go,json}`, `bootstrappers.json`, `params.go`, `checkpoints.json`) | `ava-genesis` |
| `nat/`, `trace/`, `subnets/` | `ava-node` submodules (`nat`, `trace`), `ava-config::subnets` |

This is the **drop-in-compat surface**: `./avalanchego` and `./avalanchego
--network-id=fuji` must behave identically to Go, accept every flag with the same
name/default/precedence, expose the same JSON-RPC/Connect APIs, and produce
identical genesis block IDs.

---

## 1. `ava-config` — flags & configuration

### 1.1 Goals & invariants

1. **Every flag** in `config/keys.go` (≈210 keys) is reproduced with the exact
   string name, type, and default. A golden parity test (§1.8) diffs the generated
   flag set against a snapshot extracted from the Go tree.
2. **Precedence** (highest wins), matching `spf13/viper` exactly (00 §7.5):
   `CLI flag > env var (AVAGO_*) > config file > built-in default`.
3. **Env var mapping:** `EnvVarName("avago", key)` uppercases and replaces `-`→`_`,
   e.g. `network-id` → `AVAGO_NETWORK_ID` (mirror `config/viper.go` `EnvPrefix =
   "avago"`, `DashesToUnderscores`).
4. **Config file** supplied via `--config-file` (path) or `--config-file-content`
   (base64, with `--config-file-content-type` ∈ {`json`,`yaml`,`toml`}, default
   `json`). Content flag overrides the path flag.
5. **`$AVALANCHEGO_DATA_DIR` / `$HOME` expansion** in path values, exactly as Go's
   `getExpandedArg` does (the `--data-dir` value is the expansion of
   `$AVALANCHEGO_DATA_DIR`).
6. **Deprecated keys**: emit the same `Config <key> has been deprecated, <msg>`
   warning to stdout when set in the file, and `--flag is deprecated` on the CLI.

### 1.2 Crate layout

```
ava-config/
├── keys.rs        // const KEY_* = "..."  (mirror keys.go, 1:1)
├── flags.rs       // build_command() -> clap::Command with every flag + default + help
├── defaults.rs    // default values pulled from ava-genesis::LocalParams, ava-network, etc.
├── precedence.rs  // the viper-equivalent Layered resolver (clap + env + file)
├── node.rs        // pub struct Config  (mirror config/node/config.go) — the resolved config
├── parse.rs       // get_node_config(): Layered -> Config (mirror config/config.go GetNodeConfig)
├── subnets.rs     // subnets::Config (mirror subnets/config.go) + subnet-config-dir loader
└── chain_config.rs// per-chain config-dir + chain-config-content loader
```

**Why not `figment`/`config`?** Evaluated both. `figment` has the cleanest
layered-provider model but does not replicate viper's exact key normalization
(case-insensitive, dash/underscore folding) or its `IsSet`/`InConfig` distinction
(needed for "ignored if X is specified" semantics). The `config` crate is similar.
**Decision:** use `clap` (derive-free `Command` built programmatically so flag
names are data, see §1.4) for CLI + help, and a **hand-written `Layered`
resolver** (`precedence.rs`, §1.5) for the env+file+default merge. This is ~300
lines and gives us bit-exact viper parity, which a third-party crate cannot
guarantee. `serde_json`/`serde_yaml`/`toml` parse the config file into a
`serde_json::Value` tree that the resolver queries.

### 1.3 Flag taxonomy (groups → representative flags)

The full set is ≈210 keys; below is the grouping with the load-bearing flags
(exact name, Rust type, default). Defaults that are `genesis.LocalParams.*` or
`constants.Default*` are sourced from `ava-genesis`/`ava-network`/`ava-consensus`
so they cannot drift.

**Process** (printed-and-quit): `version` (bool, false), `version-json` (bool, false).

**Node / paths:**
| flag | type | default |
|---|---|---|
| `data-dir` | String | `$HOME/.avalanchego` |
| `plugin-dir` | String | `<data>/plugins` |
| `chain-data-dir` | String | `<data>/chainData` |
| `db-dir` | String | `<data>/db` |
| `log-dir` | String | `<data>/logs` |
| `profile-dir` | String | `<data>/profiles` |
| `process-context-file` | String | `<data>/process.json` |
| `config-file` / `config-file-content` / `config-file-content-type` | String | `""`/`""`/`json` |
| `genesis-file` / `genesis-file-content` | String | `""` |
| `upgrade-file` / `upgrade-file-content` | String | `""` |
| `fd-limit` | u64 | `ulimit::DEFAULT_FD_LIMIT` |

**Network ID & ACP:** `network-id` (String, `mainnet`), `acp-support`
(Vec<i32>, nil), `acp-object` (Vec<i32>, nil).

**Database:** `db-type` (String, `leveldb` → maps to RocksDB backend, see 04;
accepts `leveldb`/`memdb`/`pebbledb` names for parity), `db-read-only` (bool,
false), `db-config-file`/`db-config-file-content` (String, "").

**Staking:** `staking-host` (String, ""), `staking-port` (u16, `9651`),
`sybil-protection-enabled` (bool, **true**), `sybil-protection-disabled-weight`
(u64, 100), `staking-tls-key-file` (`<data>/staking/staker.key`),
`staking-tls-cert-file` (`<data>/staking/staker.crt`), `*-content` (b64),
`staking-ephemeral-cert-enabled` (bool, false), `staking-signer-key-file`
(`<data>/staking/signer.key`), `staking-ephemeral-signer-enabled` (bool, false),
`staking-rpc-signer-endpoint` (String, ""), `partial-sync-primary-network`
(bool, false).

**Staking economics** (defaults from `genesis::LocalParams`):
`uptime-requirement` (f64), `min-validator-stake`/`max-validator-stake`/
`min-delegator-stake` (u64), `min-delegation-fee` (u64, range [0,1e6]),
`min-stake-duration`/`max-stake-duration` (Duration), `stake-max-consumption-rate`/
`stake-min-consumption-rate` (u64), `stake-minting-period` (Duration),
`stake-supply-cap` (u64), `tx-fee`/`create-asset-tx-fee` (u64).

**Dynamic/validator fees** (from `LocalParams.DynamicFeeConfig` /
`ValidatorFeeConfig`): `dynamic-fees-{bandwidth-weight,db-read-weight,
db-write-weight,compute-weight,max-gas-capacity,max-gas-per-second,
target-gas-per-second,min-gas-price,excess-conversion-constant}` (all u64),
`validator-fees-{capacity,target,min-price,excess-conversion-constant}` (u64).

**HTTP API:**
| flag | type | default |
|---|---|---|
| `http-host` | String | `127.0.0.1` |
| `http-port` | u16 | `9650` |
| `http-tls-enabled` | bool | false |
| `http-tls-key-file` / `-cert-file` (+ `-content`) | String | "" |
| `http-allowed-origins` | String | `*` |
| `http-allowed-hosts` | Vec<String> | `["localhost"]` |
| `http-shutdown-wait` | Duration | `0` |
| `http-shutdown-timeout` | Duration | `10s` |
| `http-read-timeout` | Duration | `30s` |
| `http-read-header-timeout` | Duration | `30s` |
| `http-write-timeout` | Duration | `30s` |
| `http-idle-timeout` | Duration | `120s` |

**Enable/disable APIs:** `api-admin-enabled` (bool, **false**),
`api-info-enabled` (bool, **true**), `api-metrics-enabled` (bool, **true**),
`api-health-enabled` (bool, **true**).

**Snow consensus** (defaults `snowball::DefaultParameters`, see 06):
`snow-sample-size` (i32, K=20), `snow-quorum-size` (AlphaConfidence=15),
`snow-preference-quorum-size` (AlphaPreference), `snow-confidence-quorum-size`
(AlphaConfidence), `snow-commit-threshold` (Beta), `snow-concurrent-repolls`,
`snow-optimal-processing`, `snow-max-processing`, `snow-max-time-processing`
(Duration). **Simplex:** `simplex-max-network-delay`,
`simplex-max-rebroadcast-wait` (Duration). **ProposerVM:**
`proposervm-use-current-height` (bool, false), `proposervm-min-block-delay`
(Duration).

**Networking** (≈40 keys, defaults `constants.Default*` from 05): timeouts
(`network-{initial,minimum,maximum,maximum-inbound}-timeout`,
`-timeout-halflife`, `-timeout-coefficient` f64, `-read-handshake-timeout`,
`-ping-timeout`, `-ping-frequency`), peer-list gossip
(`network-peer-list-num-validator-ips` u, `-pull-gossip-frequency`,
`-bloom-reset-frequency`), `network-compression-type` (String, `zstd`),
`network-max-clock-difference`, `network-allow-private-ips` (bool — **default is
network-dependent**: false for mainnet/fuji, true otherwise; resolved in §1.6),
`network-require-validator-to-connect`, peer buffers
(`network-peer-{read,write}-buffer-size`), `network-tcp-proxy-enabled`/
`-read-timeout`, `network-tls-key-log-file-unsafe`, reconnect delays,
`network-no-ingress-connections-grace-period`. Connection throttling:
`network-inbound-connection-throttling-{cooldown,max-conns-per-sec f64}`,
`network-outbound-connection-throttling-rps`, `network-outbound-connection-timeout`.
Network health: `network-health-min-conn-peers`,
`-max-time-since-msg-{received,sent}`, `-max-portion-send-queue-full` f64,
`-max-send-fail-rate` f64, `-max-outstanding-request-duration`.

**Throttlers** (inbound/outbound, all from `constants.Default*`):
`throttler-inbound-{at-large-alloc-size,validator-alloc-size,
node-max-at-large-bytes,node-max-processing-msgs,bandwidth-refill-rate,
bandwidth-max-burst-size,cpu-max-recheck-delay,disk-max-recheck-delay,
cpu-validator-alloc,cpu-max-non-validator-usage,cpu-max-non-validator-node-usage,
disk-validator-alloc,disk-max-non-validator-usage,disk-max-non-validator-node-usage}`,
`throttler-outbound-{at-large-alloc-size,validator-alloc-size,
node-max-at-large-bytes}`.

**System / resource trackers:** `system-tracker-frequency` (500ms),
`-processing-halflife`/`-cpu-halflife` (15s), `-disk-halflife` (1m),
`-disk-required-available-space` (u64, 0, **deprecated**),
`-disk-required-available-space-percentage` (3),
`-disk-warning-threshold-available-space` (deprecated),
`-disk-warning-available-space-percentage` (10). CPU/disk alloc:
`throttler-inbound-cpu-*` map to `NumCPU`-derived defaults (`runtime::NumCPU` →
`num_cpus::get()`), disk allocs default `1000 * GiB`.

**Benchlist:** `benchlist-halflife` (Duration), `-unbench-probability` (f64),
`-bench-probability` (f64), `-duration` (Duration) — defaults from
`benchlist::Default*`.

**Bootstrap / state sync:** `bootstrap-ips`/`bootstrap-ids` (comma-sep String),
`state-sync-ips`/`state-sync-ids`, `bootstrap-beacon-connection-timeout` (1m),
`bootstrap-max-time-get-ancestors` (50ms),
`bootstrap-ancestors-max-containers-{sent,received}` (2000). When empty for
standard networks, defaults are filled from `ava-genesis` bootstrapper list (§1.6).

**Router / consensus engine:** `consensus-app-concurrency`,
`consensus-shutdown-timeout`, `consensus-frontier-poll-frequency`,
`router-health-max-drop-rate` (f64, 1), `router-health-max-outstanding-requests`
(u, 1024), `network-health-max-outstanding-request-duration` (5m),
`meter-vms-enabled` (bool, true), `uptime-metric-freq` (30s).

**Subnets / tracking:** `track-subnets` (comma-sep String, ""),
`subnet-config-dir` (`<data>/configs/subnets`) / `subnet-config-content` (b64),
`chain-config-dir` (`<data>/configs/chains`) / `chain-config-content` (b64),
`vm-aliases-file`/`-content`, `chain-aliases-file`/`-content`.

**Logging:** `log-dir`, `log-level` (String, `info`), `log-display-level` (""),
`log-format` (`auto`), `log-rotater-max-size` (u, 8 MB),
`log-rotater-max-files` (u, 7), `log-rotater-max-age` (u, 0),
`log-rotater-compress-enabled` (bool, false),
`log-disable-display-plugin-logs` (bool, false).

**Health:** `health-check-frequency` (30s), `health-check-averager-halflife`.

**Indexer:** `index-enabled` (bool, **false**), `index-allow-incomplete`
(bool, false).

**Profiling:** `profile-dir`, `profile-continuous-enabled` (bool, false),
`profile-continuous-freq` (15m), `profile-continuous-max-files` (i32, 5).

**Public IP / NAT:** `public-ip` (String, ""), `public-ip-resolution-frequency`
(5m), `public-ip-resolution-service` (String, "" — `opendns`/`ifconfigco`/
`ifconfigme`).

**Tracing (OTel):** `tracing-exporter-type` (String, `disabled` —
`disabled`/`grpc`/`http`), `tracing-endpoint` (""), `tracing-insecure`
(bool, **true**), `tracing-sample-rate` (f64, 0.1), `tracing-headers`
(map String→String, `--tracing-headers k=v`).

### 1.4 clap construction (names as data)

Flags are declared in a table so the parity test can enumerate them and so help
text matches Go's. We build `clap::Command` programmatically:

```rust
pub struct FlagSpec {
    pub key: &'static str,                  // e.g. "network-id"
    pub kind: FlagKind,                     // Bool|String|U64|I64|F64|Duration|StringSlice|IntSlice|StringMap
    pub default: DefaultVal,                // resolved lazily (some pull from ava-genesis)
    pub help: &'static str,
    pub deprecated: Option<&'static str>,
}

pub fn build_command(specs: &[FlagSpec]) -> clap::Command {
    let mut cmd = clap::Command::new("avalanchego")
        .version(ava_version::CURRENT.to_string())
        .disable_help_flag(false)            // Go pflag prints help on --help (ErrHelp -> exit 0)
        .arg_required_else_help(false);      // no args => run mainnet node
    for s in specs {
        let mut arg = clap::Arg::new(s.key).long(s.key).help(s.help);
        arg = match s.kind {
            FlagKind::Bool => arg.num_args(0..=1)        // pflag bools accept `--x` and `--x=true`
                                  .default_missing_value("true")
                                  .value_parser(clap::value_parser!(bool)),
            FlagKind::Duration => arg.value_parser(parse_go_duration), // "30s","5m","120ms"
            FlagKind::StringSlice => arg.value_delimiter(','),
            _ => arg,
        };
        if let Some(msg) = s.deprecated { arg = arg.help(format!("DEPRECATED: {msg}")); }
        cmd = cmd.arg(arg);
    }
    cmd
}
```

> **Duration parity:** Go uses `time.ParseDuration` (`ns,us,ms,s,m,h`). We provide
> `parse_go_duration` (not `humantime`, which rejects `300ms` written `0.3s` etc.)
> to accept the exact Go grammar.

### 1.5 The precedence resolver (viper shim)

```rust
/// Layered config resolver. Mirrors spf13/viper precedence:
/// explicit-set CLI flag > env (AVAGO_*) > config file > default.
pub struct Layered {
    cli: clap::ArgMatches,                 // parsed flags; ArgMatches::value_source tells us if set
    env: HashMap<String, String>,          // AVAGO_* snapshot, keys normalized to flag form
    file: serde_json::Value,               // config file parsed into a JSON tree (json|yaml|toml)
    specs: HashMap<&'static str, FlagSpec>,
}

impl Layered {
    pub fn build(cmd: clap::Command, args: Vec<String>, specs: &[FlagSpec]) -> Result<Self> {
        let cli = cmd.try_get_matches_from(args)?;       // ErrHelp -> exit 0 handled by caller
        // 1. env snapshot: AVAGO_NETWORK_ID -> "network-id"
        let env = std::env::vars()
            .filter_map(|(k, v)| k.strip_prefix("AVAGO_").map(|r|
                (r.to_ascii_lowercase().replace('_', "-"), v)))
            .collect();
        // 2. config file: --config-file-content (b64) takes priority over --config-file path
        let file = load_config_file(&cli)?;              // empty Value::Null if none
        Ok(Self { cli, env, file, specs: index(specs) })
    }

    /// True if the key was provided anywhere except its default (viper IsSet).
    pub fn is_set(&self, key: &str) -> bool {
        matches!(self.cli.value_source(key), Some(clap::parser::ValueSource::CommandLine))
            || self.env.contains_key(key)
            || self.file_lookup(key).is_some()
    }

    pub fn get_string(&self, key: &str) -> String {
        if matches!(self.cli.value_source(key), Some(clap::parser::ValueSource::CommandLine)) {
            return self.cli.get_one::<String>(key).cloned().unwrap_or_default();
        }
        if let Some(v) = self.env.get(key)            { return v.clone(); }
        if let Some(v) = self.file_lookup(key)        { return v.as_str().unwrap_or_default().into(); }
        self.default_string(key)                       // FlagSpec default (may pull ava-genesis)
    }
    // get_bool/get_u64/get_f64/get_duration/get_string_slice analogous.
}
```

Path expansion (`getExpandedArg`) is applied on read for path-typed keys:
`$AVALANCHEGO_DATA_DIR` expands to the resolved `data-dir`, other `$VARS` via
`std::env::var`.

### 1.6 Network-dependent defaults & file loaders (`parse.rs`)

`get_node_config(&Layered) -> Config` mirrors `config/config.go::GetNodeConfig`,
including the order-sensitive derived values:

- **`network-id`** parsed via `ava-types::network_id_from_string` (accepts
  `mainnet`/`fuji`/`local`/numeric). It gates many later defaults.
- **`network-allow-private-ips`**: if the flag is *set*, use it; else default
  `false` for Mainnet/Fuji, `true` otherwise (Go comment in `flags.go:178`).
- **Bootstrappers**: if `bootstrap-ips`/`-ids` empty and network is standard, fill
  from `ava-genesis::bootstrappers(network_id)`.
- **Genesis bytes**: from `--genesis-file(-content)` for custom networks, else
  the embedded config (`ava-genesis`, §6). Produces `GenesisBytes` + `AvaxAssetID`.
- **Subnet configs**: read `subnet-config-content` (b64 map) or every
  `<subnet-config-dir>/<subnetID>.json` → `HashMap<SubnetID, subnets::Config>`.
- **Chain configs**: `chain-config-content` (b64) or
  `<chain-config-dir>/<chainAlias>/{config.json,upgrade.json}` →
  `HashMap<String, chains::ChainConfig{ config: Vec<u8>, upgrade: Vec<u8> }>`.
- **Validation** mirrors `config.go`: e.g. `sybil-protection-disabled` is rejected
  on Mainnet/Fuji; ephemeral certs rejected on standard networks; staking key/cert
  generated if absent and not ephemeral.

`Config` (the resolved struct, `config/node/config.go`) is the **shared contract**
read by `ava-node` and every subsystem. It is `#[derive(Clone)]`, not
serde-serialized on the hot path; it embeds sub-configs owned by other crates:
`NetworkConfig` (05), `BenchlistConfig`/`ConsensusParams` (06),
`DatabaseConfig` (04), `LoggingConfig`, `HTTPConfig` (§3), `TraceConfig` (§7),
`SubnetConfigs`, `ChainConfigs`, `IPCConfig`, staking certs/signer.

### 1.7 subnet & chain config dirs (`subnets.rs`)

`subnets::Config` mirrors `subnets/config.go`:
```rust
pub struct Config {
    pub validator_only: bool,
    pub allowed_nodes: BTreeSet<NodeID>,         // only valid when validator_only
    pub consensus_parameters: snowball::Parameters, // per-subnet snow overrides (06)
    pub proposer_min_block_delay: Duration,
    pub gossip_config: GossipConfig,
}
```
Validation error `allowedNodes can only be set when ValidatorOnly is true` is
preserved as an enum variant. The Primary Network subnet config comes from the
top-level consensus flags; tracked subnets merge file overrides.

### 1.8 Tests

- **Flag-parity golden test** (`02`): a `gen-flags` xtask runs Go
  `config.BuildFlagSet()` and dumps `name,type,default,deprecated` to
  `tests/golden/flags.json`. The Rust test builds `build_command(&FLAG_SPECS)`,
  serializes the same shape, and asserts set-equality (names), per-flag type
  equality, and default-string equality. CI fails on drift either way.
- **Precedence unit tests**: matrix of {flag, env, file, default} proving the
  ordering and `is_set` semantics; base64 content vs path; data-dir expansion.
- **Duration grammar** proptest against Go's `time.ParseDuration` vectors.

---

## 2. `ava-node` — node assembly & lifecycle

### 2.1 Runtime ownership (00 §7.2)

`avalanchego` (bin) builds the single `tokio` multi-thread runtime and passes a
`Handle` into `Node::new`. Library crates spawn onto it; none create their own
runtime. A root `CancellationToken` is created in `Node` and child-cloned per
subsystem; `Shutdown` cancels it and awaits joins.

### 2.2 Initialization order (mirror `node/node.go::New`, exact)

The Go order is load-bearing (comments call out dependencies). The Rust `Node::new`
reproduces it step-for-step:

1. Parse staking TLS cert → `NodeID` (`ids::NodeIDFromCert`, see 03/05).
2. Build BLS staking signer (`newStakingSigner`) + proof-of-possession.
3. Log "initializing node" with version/commit/nodeID/POP/config.
4. `VMAliaser` + `VMManager` (07).
5. `init_bootstrappers` (beacon list from config).
6. `trace::new(TraceConfig)` (§7).
7. `init_metrics` (Prometheus multi-gatherer).
8. `init_nat` (§8).
9. `init_api_server` (bind HTTP listener; §3).
10. `init_metrics_api`.
11. `init_database` (04; RocksDB/mem; writes `ungracefulShutdown` marker key).
12. `init_shared_memory` (atomic memory for X↔C/P atomic ops).
13. Build `message::Creator` (05) — **must be after metrics, before networking/
    chain-manager/engine** (shares the network namespace registerer).
14. `validators::Manager`; wrap in `overriddenManager` if sybil protection off.
15. `init_resource_manager`, `init_cpu_targeter`, `init_disk_targeter`.
16. `init_networking(network_registerer)` (05).
17. `init_event_dispatchers` (Block/Tx/Vertex `AcceptorGroup`).
18. `init_health_api` — **before chain manager** (§3.4).
19. `add_default_vm_aliases`.
20. `init_chain_manager(avax_asset_id)` (07).
21. `init_vms` (VM registry; rpcchainvm runtime manager).
22. `init_admin_api`, `init_info_api`.
23. `init_chain_aliases`, `init_api_aliases` (from genesis bytes).
24. `init_indexer` (§5).
25. `health.start(freq)`, `init_profiler`.
26. `init_chains(genesis_bytes)` — creates the P-Chain, which bootstraps X & C.

```rust
pub struct Node {
    pub log: Logger,
    pub id: NodeID,
    pub config: Arc<Config>,
    staking_tls_signer: Arc<dyn Signer>,
    staking_signer: bls::Signer,
    db: Arc<dyn Database>,
    net: Arc<Network>,                       // 05
    chain_router: Arc<dyn Router>,           // 06
    chain_manager: Arc<dyn ChainManager>,    // 07
    vm_manager: Arc<vms::Manager>,
    vm_registry: Arc<dyn VMRegistry>,
    runtime_manager: Arc<dyn rpc_runtime::Manager>, // plugin subprocess host
    validators: Arc<dyn validators::Manager>,
    bootstrappers: Arc<dyn validators::Manager>,
    api_server: Arc<dyn ApiServer>,          // 3
    indexer: Arc<dyn Indexer>,               // 5
    health: Arc<dyn Health>,
    benchlist: Arc<dyn benchlist::Manager>,
    timeout_manager: Arc<timeout::Manager>,
    resource_manager: Arc<dyn resource::Manager>,
    nat: Nat,                                // 8 (port mapper + ip updater)
    tracer: Arc<dyn Tracer>,                 // 7
    metrics: MultiGatherer,
    shutdown: CancellationToken,             // root token
    exit_code: AtomicI32,
    shutting_down: AtomicBool,
    shutdown_once: tokio::sync::OnceCell<()>,
}

impl Node {
    pub async fn new(config: Arc<Config>, log_factory: Arc<LogFactory>, log: Logger,
                     rt: tokio::runtime::Handle) -> Result<Arc<Self>> {
        // … steps 1–26 above, returning early with typed errors …
    }
}
```

### 2.3 Dispatch (mirror `node.go::Dispatch`)

```rust
pub async fn dispatch(self: &Arc<Self>) -> Result<()> {
    self.write_process_context()?;          // process.json: {pid, uri, stakingAddress}
    // API server task: on unexpected exit, Shutdown(1)
    let api = self.clone();
    tokio::spawn(async move {
        if let Err(e) = api.api_server.serve().await {
            if !api.shutting_down.load(Relaxed) { api.log.fatal("API server dispatch failed", e); }
            api.shutdown(1).await;
        }
    });
    // bootstrap-beacon-connection-timeout warn task (select on onSufficientlyConnected)
    // manually track state-sync + bootstrap peers
    for ip in &self.config.state_sync_ips { self.net.manually_track(...); }
    for b in &self.config.bootstrappers    { self.net.manually_track(b.id, b.ip); }
    let ret = self.net.dispatch().await;     // blocks until P2P stack stops
    self.shutdown(1).await;                  // if net stops, shut down
    ret
}
```

### 2.4 Shutdown ordering (mirror `node.go::shutdown`, exact)

`shutdown` runs once (OnceCell). Order:
1. Register `shuttingDown` health check (forces unhealthy), sleep `http-shutdown-wait`.
2. `staking_signer.shutdown()`.
3. `resource_manager.shutdown()`.
4. `timeout_manager.stop()`.
5. `chain_manager.shutdown()`.
6. `benchlist.shutdown()`.
7. `profiler.shutdown()`.
8. `net.start_close()`.
9. `api_server.shutdown()` (graceful, `http-shutdown-timeout`).
10. `nat.unmap_all_ports()`, `ip_updater.stop()`.
11. `indexer.close()`.
12. `runtime_manager.stop()` (kill plugin subprocesses).
13. `db.delete(UNGRACEFUL_SHUTDOWN)` then `db.close()`.
14. `tracer.close()`.

The root `CancellationToken` is cancelled at step 8/9 so spawned tasks unwind; we
`join` their handles before returning (Go relies on `Net.Dispatch()` returning).

### 2.5 Signals & exit (mirror `app/` + `main/`)

The `avalanchego` bin installs a `tokio::signal` handler for SIGINT/SIGTERM →
`node.shutdown(0)`, and SIGABRT → dump all task/thread backtraces to stderr (Go's
`GetStacktrace(true)`). `exit_code()` blocks on the node's run future and returns
the recorded code. Panics in the dispatch task are caught, logged, and re-raised
analogous to `log.StopOnPanic`.

### 2.6 Plugin subprocess mode (`rpcchainvm`)

Two roles, both via `ava-vm-rpc` (tonic, 07):
- **Host:** `runtime_manager` launches plugin binaries from `plugin-dir`, performs
  the go-plugin handshake (env `AVALANCHEGO_PLUGIN_PROTOCOL_VERSION`, magic
  cookie, stdout handshake line `1|<proto>|tcp|<addr>|grpc`), dials the gRPC
  `vm.proto`/`runtime.proto` server, and registers the VM. A Rust node can host a
  Go plugin and vice-versa (00 compatibility surface).
- **Guest:** a Rust VM plugin binary prints the same handshake line and serves
  `vm.proto`; runnable under a Go node. `ava-node` itself is not a plugin; this is
  a separate small `main` per VM crate (see 07/10/11).

---

## 3. `ava-api` — HTTP server, JSON-RPC 2.0 shim, APIs

### 3.1 Server (mirror `api/server`)

`axum` + `hyper` + `tower` replace `gorilla/mux` + `net/http`. Routes are mounted
under base `/ext`. Requirements preserved:
- **h2c** (HTTP/2 cleartext) with `MaxConcurrentStreams=64` — needed so Connect/
  gRPC-Web clients work over the same port. Use `hyper` with `http2` + a `tower`
  service that negotiates h2c (no ALPN since no TLS by default).
- **CORS** via `tower-http::cors`: `allow_origins` from `http-allowed-origins`
  (default `*`), `allow_credentials=true` (mirrors `rs/cors` config).
- **Allowed-Hosts** middleware: reject requests whose `Host` header isn't in
  `http-allowed-hosts` (wildcard `*` accepts all; empty/IP host always accepted)
  with `403` (mirror `allowed_hosts.go::filterInvalidHosts`).
- **`node-id` response header** set on every response.
- **Timeouts** from `HTTPConfig` (read/read-header/write/idle) via a `tower` layer.
- **Per-chain reject middleware**: until a chain's consensus state is `NormalOp`,
  its routes return `503 "API call rejected because chain is not done
  bootstrapping"`.
- **Metrics + trace wrappers** per handler (mirror `wrapMiddleware`):
  per-chain/per-base request metrics, and OTel span when tracing enabled
  (`traced_handler.go`).

```rust
#[async_trait]
pub trait ApiServer: Send + Sync {
    fn add_route(&self, handler: BoxedHandler, base: &str, endpoint: &str) -> Result<()>;
    fn add_aliases(&self, endpoint: &str, aliases: &[String]) -> Result<()>;
    /// Mount a chain's VM HTTP handlers under /ext/bc/<chainID>/<endpoint>.
    async fn register_chain(&self, name: &str, ctx: &ConsensusContext, vm: Arc<dyn CommonVM>);
    async fn serve(&self) -> Result<()>;
    async fn shutdown(&self) -> Result<()>;
}
```

**Chain mounting contract** (the cross-crate API contract): `register_chain` calls
`vm.create_handlers()` → `HashMap<extension, Handler>` and mounts each under
`/ext/bc/<chainID>/<extension>` (extension `""` allowed; validated like Go's
`url.ParseRequestURI`). It also mounts the VM's header-route handler (used by EVM
for `/rpc`, `/ws`) via `add_header_route(chainID, handler)`. Chain aliases
(`X`,`C`,`P`) are registered as path aliases so `/ext/bc/X` works. This matches
`chains.Manager` → `server.RegisterChain` in 07.

### 3.2 JSON-RPC 2.0 shim (replaces `gorilla/rpc/v2` + `json` codec)

Go registers a service object with method-per-exported-method, dispatched as
`{"jsonrpc":"2.0","method":"<service>.<Method>","params":[{…}],"id":N}`. We
provide a small router over `axum`:

```rust
/// One registered JSON-RPC service (e.g. "info", "admin", "health").
pub struct JsonRpcService {
    name: String,
    methods: HashMap<String, BoxedRpcMethod>,   // "GetNodeID" -> handler
}
pub type BoxedRpcMethod =
    Box<dyn Fn(serde_json::Value) -> BoxFuture<'static, Result<serde_json::Value, RpcError>>
        + Send + Sync>;

#[derive(Deserialize)]
struct Req { jsonrpc: String, method: String, #[serde(default)] params: serde_json::Value, id: Value }
#[derive(Serialize)]
struct Resp { jsonrpc: &'static str, #[serde(skip_serializing_if="Option::is_none")] result: Option<Value>,
              #[serde(skip_serializing_if="Option::is_none")] error: Option<RpcErr>, id: Value }

/// axum handler: POST /ext/<path>  with Content-Type application/json[;charset=UTF-8]
async fn dispatch(State(reg): State<Arc<ServiceRegistry>>, body: Bytes) -> impl IntoResponse {
    let req: Req = match serde_json::from_slice(&body) { Ok(r) => r, Err(_) =>
        return json_error(PARSE_ERROR, "parse error", Value::Null) };
    // gorilla uses "<service>.<Method>"; params is a 1-element array per gorilla codec.
    let (svc, m) = req.method.split_once('.').unwrap_or(("", &req.method));
    let arg = first_param(req.params);             // gorilla: params[0]
    match reg.lookup(svc, m) {
        Some(f) => match f(arg).await {
            Ok(result) => Json(Resp { jsonrpc:"2.0", result:Some(result), error:None, id:req.id }),
            Err(e)     => Json(Resp { jsonrpc:"2.0", result:None,
                                      error:Some(e.into_jsonrpc()), id:req.id }),
        },
        None => json_error(METHOD_NOT_FOUND, "method not found", req.id),
    }
}
```

> **Codec parity detail:** `gorilla/rpc/v2/json` (v1, not v2 "json2") wraps params
> in a **single-element array** and uses method names `Service.Method`. Error
> objects are `{"code","message","data"}`. Our shim replicates that wire shape so
> existing clients/SDKs are unaffected. Method names use Go's exported PascalCase
> (`info.GetNodeID`, `health.Health`).

Service method registration is generated by a small derive/macro (`#[rpc_service("info")]`)
so adding a method can't drift from the trait; each method is `async fn(&self,
Args) -> Result<Reply, RpcError>` with `Args`/`Reply` = serde types mirroring Go.

### 3.3 `info` API (method-by-method parity, `api/info/service.go`)

Mounted at `/ext/info`. Methods (request→reply types mirror Go field names/json
tags exactly): `getNodeVersion` (version, databaseVersion, rpcProtocolVersion,
gitCommit, vmVersions), `getNodeID` (nodeID, nodePOP), `getNodeIP`, `getNetworkID`,
`getNetworkName`, `getBlockchainID(alias)`, `peers(nodeIDs)` →
{numPeers, peers:[{ip,publicIP,nodeID,version,lastSent,lastReceived,
observedUptime,trackedSubnets,benched}]}, `isBootstrapped(chain)`, `upgrades`
(returns `upgrade::Config`), `uptime` → {rewardingStakePercentage,
weightedAveragePercentage}, `acps` → supported/objected ACP sets, `getTxFee` →
{txFee, createAssetTxFee, …}, `getVMs` → {vms: map[vmID→aliases], fxs}. `Info` is
constructed from `Parameters{version,nodeID,nodePOP,networkID,vmManager,upgrades,
txFee,createAssetTxFee}` plus validators/chainManager/network/benchlist handles.

### 3.4 `health` API (`api/health`)

Mounted at `/ext/health`. **Dual handler** (mirror `handler.go`):
- **GET** `/ext/health` (and `?tag=...`) → `200` healthy / `503` unhealthy, body
  `{checks: map[name→Result], healthy: bool}`. Used by load balancers/k8s.
- **POST** JSON-RPC: `health.Health(tags)`, `health.Readiness(tags)`,
  `health.Liveness(tags)`. Reply `{checks, healthy}`.

`Health` (worker, `api/health/health.go`) runs registered checkers on
`health-check-frequency`, with EWMA averaging (`health-check-averager-halflife`),
tagged by subsystem (`ApplicationTag`, per-chain). Checkers register via
`register_health_check(name, checker, tags)`. The `shuttingDown` checker is
registered at shutdown to flip the node unhealthy. Initialized **before** the chain
manager so chains can register their bootstrap-health checks.

### 3.5 `admin` API (`api/admin`)

Mounted at `/ext/admin`, **disabled by default** (`api-admin-enabled`). Methods:
`startCPUProfiler`/`stopCPUProfiler`/`memoryProfile`/`lockProfile` (write pprof
files to `profile-dir` — Rust uses the `pprof` crate + tokio metrics for lock
profile), `alias(endpoint, alias)`, `aliasChain(chain, alias)`,
`getChainAliases(chain)`, `stacktrace` (dump goroutine→task backtraces),
`setLoggerLevel(loggerName, logLevel, displayLevel)` /
`getLoggerLevel(loggerName)` → per-logger {logLevel, displayLevel} (wired to
`tracing-subscriber` reload handles), `getConfig` → the resolved node config as
JSON (the `Config` from §1.6 serialized), `loadVMs` → rescan `plugin-dir` and
register new VMs, `dbGet(key)` → {value, errorCode} (read raw DB by hex key).

### 3.6 `metrics` API (`api/metrics`)

Mounted at `/ext/metrics`, enabled by default. Serves Prometheus text exposition
from a `MultiGatherer` that prefixes each subsystem's registry (mirror
`prefix_gatherer.go`/`label_gatherer.go`). The `prometheus` crate's
`TextEncoder` produces the output; metric names/labels must match Go (00 §7.3,
golden test in `02`).

### 3.7 Connect / gRPC-Web

Some endpoints are exposed via Connect (e.g. for the `connectclient`). We serve
gRPC over the same h2c port via `tonic`, add `tonic-web` for gRPC-Web, and a thin
Connect-protocol compat layer (Connect unary = POST with
`Content-Type: application/json|proto`, no gRPC framing) where Go exposes Connect.
Detailed per-service in 05/07; the API server just routes by content-type/path.

### 3.8 WebSocket pub-sub

EVM (`/ext/bc/C/ws`) and any VM exposing a pub-sub stream use
`tokio-tungstenite` upgraded from the same axum router (header-route handlers,
§3.1). The X-Chain/legacy `pubsub` filtered feed (bloom-filter address subscribe)
is implemented as a `tokio::sync::broadcast` fan-out behind a WS upgrade handler,
matching the Go `pubsub` message framing.

### 3.9 Auth token

Go's historical API-token auth is deprecated/removed in current avalanchego;
access control is via `http-allowed-hosts` + CORS + binding to `127.0.0.1` by
default. We reproduce exactly that (no bearer-token middleware) and keep a
feature-gated hook should a token layer be reintroduced.

---

## 4. (reserved — merged into §2/§3)

---

## 5. `ava-indexer` — block/tx indexer & index API

Mirror `indexer/`. The indexer subscribes to the node's `AcceptorGroup`s
(Block/Tx/Vertex) and, for **Primary-Network chains only**, persists an
append-only index keyed by container ID and by monotonic height.

- **`register_chain(name, ctx, vm)`** (mirror `indexer.go`): skips non-primary
  subnets and already-indexed chains. Creates per-chain indices with byte prefixes
  `tx=0x01`, `vtx=0x02`, `block=0x03`. Enforces the **incomplete-index** safety
  rule: if `index-enabled` toggles such that an index would have gaps and
  `index-allow-incomplete=false`, the node fatals (same as Go).
- **`Accept`** (mirror `index.go`): writes `containerID→bytes`,
  `height→containerID`, `containerID→height`, advances `nextAcceptedIndex`. Uses
  `ava-database` versioned batch for atomicity.
- **Index API** mounted per chain at `/ext/index/<chainAlias>/{block,tx,vtx}` as a
  JSON-RPC service. Methods (mirror `service.go`): `getLastAccepted`,
  `getContainerByIndex(index)`, `getContainerByID(id)`,
  `getContainerRange(startIndex, numToFetch)`, `getIndex(id)→{index}`,
  `isAccepted(id)→{isAccepted}`. Reply `FormattedContainer{id, bytes(hex/cb58),
  timestamp, encoding, index}`.
- **Encoding**: containers returned in the requested `encoding` (`hex`/`cb58`),
  matching the `formatting` package (03).

```rust
#[async_trait]
pub trait Indexer: Send + Sync {
    async fn register_chain(&self, name: &str, ctx: &ConsensusContext, vm: Arc<dyn CommonVM>);
    async fn close(&self) -> Result<()>;
}
```

Closed on node shutdown (step 11). A persisted `hasRun`/`incomplete` marker
matches Go so restart semantics are identical.

---

## 6. `ava-genesis` — genesis configs & block generation

Mirror `genesis/`. **Source of truth** for: embedded Mainnet/Fuji/Local configs,
bootstrapper lists, staking economics params, and the function that derives
genesis bytes + block IDs.

### 6.1 Embedded data

The three JSON files (`genesis_mainnet.json`, `genesis_fuji.json`,
`genesis_local.json`), `bootstrappers.json`, and `checkpoints.json` are embedded
verbatim with `include_str!`. `Params` (`params.go`): `MainnetParams`,
`FujiParams`, `LocalParams` (used as flag defaults in §1) embedded as Rust consts
with doc-comments citing the Go source (00 §5).

```rust
pub static MAINNET_GENESIS_CONFIG: &str = include_str!("../data/genesis_mainnet.json");
pub fn bootstrappers(network_id: u32) -> Vec<Bootstrapper>;       // {id: NodeID, ip: SocketAddr}
pub fn genesis_bytes(network_id: u32, custom: Option<&Config>) -> Result<(Vec<u8>, Id /*avaxAssetID*/)>;
```

### 6.2 `from_config` (mirror `genesis.go::FromConfig`)

Produces the byte representation of the P-Chain genesis (the network genesis
state) and the AVAX asset ID. **Determinism is critical** (00 §6.1):
- Build the AVM genesis: one asset `AVAX` (denom 9), `FixedCap` holders sorted via
  `utils.Sort` over allocations; `Memo` = concatenated ETH addresses.
- `avaxAssetID = hash of the AVM genesis tx` (mirror `AVAXAssetID`).
- Build P-Chain UTXOs (skipping initially-staked funds) and
  `PermissionlessValidator`s from `InitialStakers`, with the staking offset/
  duration math reproduced exactly (`splitAllocations`, `stakingOffset`).
- Bech32 address formatting via `ava-crypto::address::format_bech32` with the
  network HRP (03).
- Encode with the `ava-codec` linear codec (byte-exact) → genesis bytes.
- The **genesis block ID** = `ids::Id::from_bytes(hashing::compute_hash256(...))`
  over the platformvm genesis block, matching `08`.

### 6.3 Helpers

`vm_genesis(genesis_bytes, vm_id) -> p_chain::Tx` and
`avax_asset_id(avm_genesis_bytes) -> Id` mirror the Go funcs (used by `ava-node`
to extract per-VM genesis and the asset ID for the chain manager).

### 6.4 Tests

- **Genesis-block-ID golden test** (`02`): for Mainnet, Fuji, Local, assert
  `genesis_bytes` and the derived genesis block ID byte-match values dumped from
  Go (`genesis.FromConfig`). This is the strongest single compatibility check.
- Bootstrapper-list parity test (count + IDs + IPs vs `bootstrappers.json`).
- Param-default parity (the `LocalParams` consts feeding §1 flag defaults).

---

## 7. `trace` (OpenTelemetry)

Mirror `trace/`. `TraceConfig { exporter: TraceConfig, sample_rate, … }`.
`ExporterType` enum `Disabled|Grpc|Http` (string parse: `disabled`/`grpc`/`http`),
default `Disabled`. When enabled, build an `opentelemetry-otlp` exporter (gRPC via
tonic or HTTP), `tracing-insecure` controls TLS, `tracing-sample-rate` →
`Sampler::TraceIdRatioBased`, `tracing-headers` → exporter metadata. Wire into the
global `tracing` subscriber via `tracing-opentelemetry`. `tracer.close()` flushes
on shutdown (step 14). A no-op tracer (mirror `noop.go`) when disabled.

---

## 8. `nat` (port mapping)

Mirror `nat/`: `Router` trait with `upnp`, `pmp`, and `no_router` impls. The node's
`Mapper` maps the staking + (optionally) HTTP ports, renews on
`public-ip-resolution-frequency`, and unmaps on shutdown (step 10). Public-IP
resolution service (`opendns`/`ifconfigco`/`ifconfigme`) drives a `dynamicip`
updater feeding the network's advertised IP (05). Rust crates: `igd-next` (UPnP),
a small NAT-PMP client, or pure resolution if `public-ip` is static.

---

## 9. `avalanchego` binary

`main` mirrors `main/main.go`:
1. Register EVM extras (reth/coreth-equivalent init, 10).
2. `build_command(&FLAG_SPECS)`; parse args. `--help` → exit 0.
3. `--version` → print `version::get_versions().to_string()`; `--version-json` →
   pretty JSON; both set → error exit 1.
4. Build `Layered`, then `get_node_config` → `Config`.
5. If stdout is a TTY, print the ASCII `Header` banner (`app/app.go`).
6. `chmod_r` data/log dirs to `rwx` (mirror `app.New`), build `LogFactory`,
   raise fd limit (`fd-limit`).
7. Build tokio runtime, `Node::new`, run `dispatch`, install signal handlers,
   block on exit code, `std::process::exit(code)`.

Drop-in invocations validated: `./avalanchego`, `./avalanchego
--network-id=fuji`, `--config-file`, `--http-port=0`, `--version`,
`--track-subnets=...`, etc.

---

## 10. Go → Rust mapping (non-obvious)

| Go | Rust |
|---|---|
| `spf13/pflag` + `spf13/viper` | `clap::Command` (names as data) + hand-written `Layered` resolver |
| `viper.IsSet`/`InConfig` | `Layered::is_set` via `ArgMatches::value_source` + env/file probes |
| `gorilla/mux` + `gorilla/rpc/v2` + `json` codec | `axum`/`tower` router + JSON-RPC 2.0 shim (single-elem params, `Service.Method`) |
| `h2c.NewHandler` | `hyper` http2 + h2c service |
| `rs/cors` | `tower-http::cors` (allow-credentials, origins) |
| `net/http` GET-or-RPC health handler | axum handler branching on method (§3.4) |
| `snow.AcceptorGroup` subscription | `tokio::sync::broadcast` + `Acceptor` trait |
| `runtime.Manager` (go-plugin) | `ava-vm-rpc` tonic host + handshake-line protocol |
| `time.ParseDuration` | `parse_go_duration` |
| `os.Expand` data-dir | path expander in `Layered` |
| `version.GetVersions()` | `ava-version::get_versions()` (serde-serializable) |

## 11. Error model

Each crate exposes `thiserror` `Error` + `Result<T>` (00 §7.1). Sentinels
preserved as variants: `ConfigError::DeprecatedRejected`,
`ConfigError::AllowedNodesWithoutValidatorOnly`, `GenesisError::*`,
`ApiError::ChainNotBootstrapped` (→ 503), `RpcError{code,message,data}` (JSON-RPC
codes: parse −32700, invalid request −32600, method not found −32601, invalid
params −32602, internal −32603). The bin uses `anyhow` for top-level context.

## 12. Test plan (ref `02`)

1. **Flag-parity golden test** — names/types/defaults vs Go snapshot (§1.8). CI-blocking.
2. **Genesis-block-ID golden test** — Mainnet/Fuji/Local bytes + block IDs (§6.4).
3. **Config-precedence unit tests** — flag>env>file>default matrix + path expansion.
4. **API response-parity (differential harness, 02)** — drive identical JSON-RPC
   requests at a Go node and a Rust node (`info`/`admin`/`health`/`index`) and
   assert byte/shape-equal responses; metrics-name parity.
5. **Wallet tx-building vectors** — fixed UTXO sets/keys → assert signed-tx bytes
   match Go `wallet` outputs (see §13).
6. **Indexer semantics** — accept ordering, incomplete-index fatal behavior,
   range queries, restart markers.
7. **Lifecycle** — init/shutdown ordering test (record step order, compare to Go
   sequence); SIGTERM graceful shutdown integration test.

---

## 13. `ava-wallet` — SDK (build / sign / issue)

Mirror `wallet/`. Reuses tx types from `08` (P), `09` (X), `10` (C). Three layers
per chain (mirror `wallet/chain/{p,x,c}`): **Builder** (selects UTXOs, constructs
unsigned txs), **Signer** (produces credentials), **Backend** (tracks UTXOs/owners,
state for the builder), plus a **Wallet** facade combining build+sign+issue.

```rust
// wallet/chain/p/builder/builder.go -> ava-wallet::p::Builder
#[async_trait]
pub trait PBuilder {
    fn context(&self) -> &Context;
    async fn get_balance(&self, opts: &[Option_]) -> Result<HashMap<Id, u64>>;
    async fn new_base_tx(&self, outs: Vec<TransferableOutput>, opts: &[Option_]) -> Result<p::tx::BaseTx>;
    async fn new_add_permissionless_validator_tx(&self, v: &SubnetValidator, signer: PopSigner,
        asset: Id, validation_rewards: &Owner, delegation_rewards: &Owner,
        delegation_shares: u32, opts: &[Option_]) -> Result<p::tx::AddPermissionlessValidatorTx>;
    async fn new_create_subnet_tx(&self, owner: &Owner, opts: &[Option_]) -> Result<p::tx::CreateSubnetTx>;
    async fn new_create_chain_tx(&self, subnet: Id, genesis: Vec<u8>, vm: Id, fxs: Vec<Id>,
        name: String, opts: &[Option_]) -> Result<p::tx::CreateChainTx>;
    async fn new_import_tx(&self, src: Id, to: &Owner, opts: &[Option_]) -> Result<p::tx::ImportTx>;
    async fn new_export_tx(&self, dst: Id, outs: Vec<TransferableOutput>, opts: &[Option_]) -> Result<p::tx::ExportTx>;
    // … add/remove subnet validator, transfer ownership, transform subnet,
    //     L1 conversion / register / set-weight / increase-balance / disable (ACP-77) …
    async fn utxos(&self, src: Id) -> Result<Vec<Utxo>>;
    async fn get_owner(&self, owner_id: Id) -> Result<Owner>;
}
```

- **UTXO selection** mirrors Go (`common`): deterministic — sort UTXOs, prefer
  locked-then-unlocked to satisfy `amount + fee`, respect locktime/threshold; the
  algorithm must match so produced txs (and their IDs) are byte-identical.
- **Signer** (`chain/p/signer`): per-input secp256k1 credentials, BLS PoP for
  validator txs; keychain abstraction (`ava-crypto`).
- **Backend** keeps the UTXO set / owners; `with_options` overlays (memo, change
  owner, custom fee, min-issuance-time).
- **Wallet facade** (`p::Wallet`): `issue_*_tx` = build → sign → `issue_tx`
  (submit via the chain client) + record in backend.
- **Primary wallet** (`wallet/subnet/primary`): `make_wallet(uri, keychain,
  config)` fetches state (UTXOs, subnets, owners) over the API and wires
  P/X/C wallets; `Wallet{p, x, c}` facade with `NewWalletWithOptions`.
- **C-Chain wallet** (`chain/c`): atomic import/export between C and X/P (no EVM
  account txs — those go through reth's RPC, 10).

Builders/signers are pure (no I/O) given a `Backend` snapshot, enabling the
golden tx-vector tests (§12.5).

---

## 14. Performance notes / improvements over Go

- **Config parse** is one-shot at startup; no hot-path concern. Build the env
  snapshot once (avoid per-key `os.Getenv`).
- **API server:** `axum`/`hyper` with `bytes::Bytes` request bodies avoids the
  per-request allocation `net/http` incurs; JSON-RPC dispatch is a `HashMap`
  lookup (Go uses reflection per call). Reuse `serde_json` buffers via a pooled
  `Vec<u8>` writer. **Safe because** responses are byte-validated against Go in
  the differential harness (§12.4).
- **Indexer writes:** batch `Accept` writes (already a versioned batch) and offload
  the RocksDB commit to `spawn_blocking`, decoupled from the consensus acceptor
  callback — **safe because** index ordering is preserved by the monotonic height
  key and the acceptor delivers in-order.
- **Wallet:** parallelize signature generation across inputs (independent) —
  **safe because** credential order is fixed by input index, so tx bytes are
  unchanged.
- **Metrics:** the `prometheus` crate gathers without the Go gatherer's lock
  contention; prefix-gatherer merges are cheap clones of `Arc`-shared families.
- **Genesis** is computed once and cached; the golden test guards determinism.
