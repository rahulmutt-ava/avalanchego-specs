# 26 — Versioning & Compatibility

> **Status:** Subsystem spec. Conforms to `00-overview-and-conventions.md` (binding:
> the drop-in compatibility surface §1, the protocol-constants centralization §5,
> the crate names §3, the `ava-version` home for version + upgrade data, the
> Go→Rust idiom mapping §6, the error model §7.1). Where this spec records a binding
> decision (the version string the Rust port reports, the min-compatible-peer rule,
> the embedded compatibility table), every other spec MUST conform.

This spec consolidates the **node version model** and the **handshake compatibility
matrix** — currently the only cross-cutting concern with no dedicated coverage. It
is a **direct drop-in-interop concern**: a Rust node must report a version string
Go peers and operators accept, and must accept/reject peers by exactly the same rule
the Go node uses, or it will be silently dropped from (or wrongly dropped by) the
network.

**Go source covered**

| Go path | Subject |
|---|---|
| `version/constants.go` | `Client` name, `RPCChainVMProtocol = 45`, `CurrentDatabase = "v1.4.5"`, `PrevDatabase = "v1.0.0"`, the `Current` / `MinimumCompatibleVersion` / `PrevMinimumCompatibleVersion` globals, `GetCompatibility` |
| `version/version.go` | the `Application{Name,Major,Minor,Patch}` struct, `String()`, `Semantic()`, `SemanticWithCommit()`, `Compare()` |
| `version/compatibility.go` | the `Compatibility` checker + `Compatible(peer)` rule |
| `version/compatibility.json` | RPCChainVM-protocol → supporting-avalanchego-versions map (embedded) |
| `version/string.go` | `Versions{Application,Database,RPCChainVM,Commit,Go}`, `GetVersions()`, `--version` / `--version-json` output, `GitCommit` |
| `network/peer/peer.go` | handshake build (`writeMessages`), the version compatibility enforcement (`shouldDisconnect`, `handleHandshake`) |
| `network/network.go` | `version.GetCompatibility(minCompatibleTime)` wiring into the peer config |
| `vms/rpcchainvm/vm.go`, `runtime/runtime.go`, `runtime/subprocess/initializer.go` | the plugin protocol-version handshake + `ErrProtocolVersionMismatch` |
| `node/node.go` | the on-disk database-version folder (`version.CurrentDatabase`) |
| `api/info/service.go` | `info.getNodeVersion` reply (`version`, `databaseVersion`, `rpcProtocolVersion`, `gitCommit`, `vmVersions`) |

**Rust crate produced / extended:** `ava-version` (the `Application` type, the
`Compatibility` checker, the embedded compatibility table, the version constants,
and the `Versions` aggregate). The network-side enforcement lives in `ava-network`
(`05`) but its rule is *defined here*. Cross-references: `05-networking-p2p.md`
(handshake), `03-core-primitives.md` §11 (upgrade/fork schedule, version primitives),
`07-vm-framework.md` (rpcchainvm protocol v45), `15-serialization-and-wire-formats.md`
(codec versions), `04-storage-and-databases.md` R2 (DB migration),
`13-config-flags-reference.md` (version flags), `14-api-rpc-reference.md`
(`info.getNodeVersion`).

---

## 1. The version taxonomy (what must match, what is negotiated)

A node carries **six distinct, independently-versioned surfaces.** Conflating them is
the most common interop bug. Each governs a different boundary, and each has a
different "must-match" rule.

| # | Version number | Go source / current value | Governs | Match rule | Negotiated vs fixed |
|---|---|---|---|---|---|
| 1 | **Node application version** | `version.Current = avalanchego 1.14.2` | The human/operator-facing identity; carried in the P2P Handshake `Client{name,major,minor,patch}` and reported by `info.getNodeVersion`. | Reported as a string; consumed by peers (rule #2) and operators. | **Fixed** per build. |
| 2 | **P2P / network compatibility** | `MinimumCompatibleVersion = 1.14.0`, `PrevMinimumCompatibleVersion = 1.13.0`; checker `version.Compatibility` | Whether two peers will stay connected and exchange consensus messages. | **The min-compatible-peer rule (§4):** reject a peer on an *older major*, or below the min-compatible version (which one applies depends on the upgrade time). | **Negotiated at handshake** (each side applies the rule to the other's reported version). |
| 3 | **rpcchainvm plugin protocol** | `version.RPCChainVMProtocol = 45` | Whether a (possibly out-of-process, possibly Rust) VM plugin can be hosted by a given node binary. | **Exact equality** — `host == plugin`, else `ErrProtocolVersionMismatch`. | **Checked at plugin handshake** (`Runtime.Initialize`), but each side's value is fixed. Cross-ref `07` §11.1(1). |
| 4 | **Per-VM codec versions** | per-VM `codec` version tags (e.g. P-Chain `txs` codec, X-Chain, SAE) — see `15` | How serialized blocks/txs are decoded; gates which type registrations are active. | **Exact** per type (the codec version byte prefixes serialized objects); a decoder must understand the version it reads. | **Fixed** in the codec; not negotiated — divergence is a hard decode failure. |
| 5 | **Database / on-disk schema** | `version.CurrentDatabase = "v1.4.5"`, `PrevDatabase = "v1.0.0"` | The on-disk data-directory layout / schema; the DB folder name. | The node opens the dir for `CurrentDatabase`; a `PrevDatabase` dir triggers a migration. | **Fixed** per build; migration on upgrade. Cross-ref `04` R2. |
| 6 | **Network-upgrade (fork) schedule** | `upgrade/` activation **times** — see `03` §11 | Which protocol rules (fees, tx types, ACPs) are active. | **Time-based**, *not* version-based: a fork activates at a wall-clock timestamp identical on every node of a network. | **Fixed** (baked into the binary per network); the *activation* is negotiated only implicitly via the clock. |

**The crucial relationships:**

- #2 (network compat) is *derived from but distinct from* #1 (app version): a node on
  `1.14.2` advertises `1.14.2`, but **accepts** any peer `≥ 1.14.0` (post-upgrade) —
  the min-compatible floor lags the current version so rolling upgrades work.
- #6 (forks) is **orthogonal to #1/#2**: avalanchego activates forks by *time*, not by
  counting peer versions. A node does not refuse to run because peers are old; it
  refuses to *connect* to peers below the floor (#2). Two nodes on different binary
  versions but past the same fork time agree on protocol rules. This is why the
  min-compatible floor (#2) is bumped *in lockstep with* a fork that changes the wire
  in an incompatible way — the floor enforces "everyone who can still connect
  understands the active rules."
- #3 (plugin protocol) is a *local* host↔plugin contract; it never appears on the
  network and is unrelated to #2. A node can speak network-version `1.14.2` to peers
  while refusing to load a plugin built against rpcchainvm `44`.
- #4 (codec) and #6 (forks) interact: a fork may activate new tx types that use a new
  codec version; nodes past the fork time must have the binary (#1) that registers
  that codec version. The min-compatible floor (#2) is the mechanism that guarantees
  this across the connected set.

---

## 2. The application version

### 2.1 The Rust type

Mirror `version.Application` exactly. Go memoizes `String()` via `sync.Once`; in Rust
the string is cheap to format, so we either format on demand or cache in a
`OnceLock`. `Compare` is the derived lexicographic `(major, minor, patch)` ordering.

```rust
// crates/ava-version/src/application.rs
// Copyright (C) 2019, Ava Labs, Inc. All rights reserved.
// See the file LICENSE for licensing terms.

/// Mirrors `version.Application`. The wire-relevant fields are exactly
/// (name, major, minor, patch); they are carried in the P2P Handshake's
/// `Client` message (see `05` §1.4) and reported by `info.getNodeVersion`.
#[derive(Clone, Debug, Eq, PartialEq, serde::Serialize, serde::Deserialize)]
pub struct Application {
    pub name: String,
    pub major: u32,
    pub minor: u32,
    pub patch: u32,
}

impl Application {
    /// `avalanchego/MAJOR.MINOR.PATCH` — MUST be byte-identical to Go's
    /// `Application.String()` (`fmt.Sprintf("%s/%d.%d.%d", ...)`).
    /// This is the Handshake string and the `info.getNodeVersion.version` field.
    pub fn display(&self) -> String {
        format!("{}/{}.{}.{}", self.name, self.major, self.minor, self.patch)
    }

    /// `vMAJOR.MINOR.PATCH` — mirrors `Semantic()`.
    pub fn semantic(&self) -> String {
        format!("v{}.{}.{}", self.major, self.minor, self.patch)
    }

    /// `vMAJOR.MINOR.PATCH@<commit>` if commit present — mirrors `SemanticWithCommit`.
    pub fn semantic_with_commit(&self, git_commit: &str) -> String {
        if git_commit.is_empty() {
            self.semantic()
        } else {
            format!("{}@{}", self.semantic(), git_commit)
        }
    }
}

impl std::fmt::Display for Application {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}/{}.{}.{}", self.name, self.major, self.minor, self.patch)
    }
}

// Ordering = (major, minor, patch). Derive Ord/PartialOrd on a tuple view so
// `Compare` matches Go's `cmp.Compare` chain exactly (name is NOT part of ordering).
impl Ord for Application {
    fn cmp(&self, o: &Self) -> std::cmp::Ordering {
        (self.major, self.minor, self.patch).cmp(&(o.major, o.minor, o.patch))
    }
}
impl PartialOrd for Application {
    fn partial_cmp(&self, o: &Self) -> Option<std::cmp::Ordering> { Some(self.cmp(o)) }
}
```

> **Note — `name` is excluded from ordering.** Go's `Compare` only compares the three
> integers; it never compares `Name`. This matters for §4: the compatibility check
> does **not** require the peer's client name to be `"avalanchego"` — it compares
> only major/minor/patch. A peer reporting a *different* client name with a compatible
> version number is still accepted. (This is what lets alternate clients interoperate.)

### 2.2 The version string the Rust port reports (binding decision)

The Handshake's `Client.name` and `info.getNodeVersion.version` MUST be a string Go
peers and operators treat as a compatible avalanchego. The wire identity is the
*number triple* (§2.1 note: name is not compared), but operators, dashboards, and
the explorer parse the `name` and the `avalanchego/x.y.z` string.

**Decision.** The Rust port reports:

- **`name = "avalanchego"`** and a **`major.minor.patch` that this build is wire- and
  codec-compatible with** (i.e. it tracks the Go `version.Current` it was validated
  against — at time of writing `1.14.2`). The Handshake `Client` and the
  `info.getNodeVersion.version` field are therefore **byte-identical** to a Go node's
  (`avalanchego/1.14.2`). This is mandatory for interop: peers apply the min-compatible
  rule to our triple (§4), and any operator tooling that string-matches `avalanchego/`
  keeps working.
- A **distinguishing build suffix is carried out-of-band, never in the compared
  fields.** Specifically:
  - the **`gitCommit`** field of `info.getNodeVersion` and the `--version` line may
    carry a marker (e.g. the Rust build's commit, optionally a `+rust` annotation in
    the *commit*/build-metadata position only);
  - we do **not** alter `name`, `major`, `minor`, or `patch`, because those are the
    fields fed into the compatibility comparison and into peer-version metrics.

> **Honesty vs interop — the tradeoff, recorded.** A maximally honest port would
> advertise `name = "avalanche-rs"`. Because §2.1 shows `name` is not part of the
> compatibility comparison, that *would still interoperate at the protocol level*.
> **But** operator tooling, peer-version telemetry, and some validator dashboards
> aggregate by the `name` string, and a node that reports an unknown client name can
> be deprioritized or look anomalous in fleet monitoring. The conservative,
> drop-in-faithful choice is to **masquerade as `avalanchego`** on the wire and surface
> the Rust identity only through `gitCommit`/build metadata, which no peer uses for
> the connect/reject decision. If the project later wants on-wire honesty, flipping
> `name` is a one-line change with no protocol consequence — it is gated behind a
> config flag (`--version-client-name`, default `avalanchego`) so the decision is
> reversible without code changes. This flag is the only knob; the numeric triple is
> never operator-configurable (it is a build constant tied to the validated Go
> version, per `00` §5).

### 2.3 The `Versions` aggregate (`--version` / `--version-json`)

Mirror `version.Versions` and its `String()` so `--version` output is format-stable
and `--version-json` unmarshals identically.

```rust
// crates/ava-version/src/versions.rs
#[derive(Clone, Debug, serde::Serialize, serde::Deserialize)]
pub struct Versions {
    pub application: String, // Current.display() => "avalanchego/1.14.2"
    pub database: String,    // CURRENT_DATABASE => "v1.4.5"
    pub rpcchainvm: u64,     // RPC_CHAIN_VM_PROTOCOL => 45
    #[serde(skip_serializing_if = "String::is_empty")]
    pub commit: String,      // GIT_COMMIT (may be empty)
    pub go: String,          // Go reports the Go toolchain; the Rust port reports
                             // the rustc version here so the field is non-empty and
                             // the JSON shape is unchanged. (Field name kept "go".)
}

impl std::fmt::Display for Versions {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        // Mirrors version/string.go: "<app> [database=<db>, rpcchainvm=<n>, [commit=<c>, ]go=<g>]"
        write!(f, "{} [database={}, rpcchainvm={}, ", self.application, self.database, self.rpcchainvm)?;
        if !self.commit.is_empty() {
            write!(f, "commit={}, ", self.commit)?;
        }
        write!(f, "go={}]", self.go)
    }
}
```

> **`go` field.** Go fills it with `runtime.Version()` minus the `go` prefix. The
> field name is part of the `--version-json` schema, so we keep the key `go` but
> populate it with the rustc version (e.g. `1.xx.y`). A consumer that only reads the
> field as an opaque toolchain string is unaffected; a consumer that *requires* it to
> parse as a Go version is out of scope (none in avalanchego do). This is documented in
> `13`/`14`.

`info.getNodeVersion` reply (`14`): `version = Current.display()`,
`databaseVersion = CURRENT_DATABASE`, `rpcProtocolVersion = RPC_CHAIN_VM_PROTOCOL`,
`gitCommit = GIT_COMMIT`, `vmVersions = vm_manager.versions()`.

### 2.4 The version constants

```rust
// crates/ava-version/src/constants.rs — copied verbatim from version/constants.go (00 §5).
pub const CLIENT: &str = "avalanchego";
/// Bumped only when a plugin VM must be rebuilt against the latest node to stay
/// compatible. Cross-ref `07`. EXACT-equality checked at the plugin handshake.
pub const RPC_CHAIN_VM_PROTOCOL: u64 = 45;
pub const CURRENT_DATABASE: &str = "v1.4.5";
pub const PREV_DATABASE: &str = "v1.0.0";

use std::sync::LazyLock;
pub static CURRENT: LazyLock<Application> = LazyLock::new(|| Application {
    name: CLIENT.to_string(), major: 1, minor: 14, patch: 2,
});
pub static MINIMUM_COMPATIBLE_VERSION: LazyLock<Application> = LazyLock::new(|| Application {
    name: CLIENT.to_string(), major: 1, minor: 14, patch: 0,
});
pub static PREV_MINIMUM_COMPATIBLE_VERSION: LazyLock<Application> = LazyLock::new(|| Application {
    name: CLIENT.to_string(), major: 1, minor: 13, patch: 0,
});
/// Set at build time (build.rs / env), mirrors `version.GitCommit`. May be empty.
pub static GIT_COMMIT: &str = env!("AVA_GIT_COMMIT", "");
```

These are protocol constants and so must carry a doc-comment citing
`version/constants.go` and be covered by a golden test (`00` §5).

---

## 3. The `Compatibility` checker (the min-compatible-peer rule)

Mirror `version.Compatibility`. The ordering invariant is
`Current ≥ MinCompatibleAfterUpgrade ≥ MinCompatible`. The check has two clauses:

1. **Major-version gate:** if *our* major is **less than** the peer's major → reject.
   (We refuse to talk to a node from a newer major line we cannot understand. Note the
   asymmetry: we never reject a peer for being on an *older* major directly here — an
   older major fails clause 2 because its `(major,...)` triple is below the floor.)
2. **Min-compatible floor:** the floor is **`MinCompatibleAfterUpgrade`** if the clock
   is at/after the configured `UpgradeTime`, else **`MinCompatible`**. The peer must
   compare `≥` the floor.

```rust
// crates/ava-version/src/compatibility.rs
use std::time::SystemTime;

/// Mirrors `version.Compatibility`. Invariant: current >= min_after_upgrade >= min.
pub struct Compatibility {
    pub current: Application,
    pub min_compatible_after_upgrade: Application,
    pub min_compatible: Application,
    /// The fork time that switches the floor from `min` to `min_after_upgrade`.
    pub upgrade_time: SystemTime,
    /// Mockable clock (mirrors `mockable.Clock`) — injected for tests.
    pub clock: Clock,
}

impl Compatibility {
    /// Mirrors `Compatibility.Compatible`. True ⇒ the peer is connectable AND we can
    /// exchange consensus messages with it. Used by `ava-network` at handshake and on
    /// every Ping (see `05` §3.2 `should_disconnect`, peer.go `shouldDisconnect`).
    pub fn compatible(&self, peer: &Application) -> bool {
        // Clause 1: older major than the peer ⇒ incompatible.
        if self.current.major < peer.major {
            return false;
        }
        // Clause 2: pick the floor by clock vs upgrade_time, peer must be >= floor.
        let floor = if self.clock.now() < self.upgrade_time {
            &self.min_compatible
        } else {
            &self.min_compatible_after_upgrade
        };
        peer >= floor
    }
}

/// Constructed like `version.GetCompatibility(upgrade_time)`. The `upgrade_time`
/// is the network's relevant fork activation time, threaded in by `ava-network`
/// (Go: `network.go` passes `minCompatibleTime` → `version.GetCompatibility`).
pub fn get_compatibility(upgrade_time: SystemTime) -> Compatibility {
    Compatibility {
        current: CURRENT.clone(),
        min_compatible_after_upgrade: MINIMUM_COMPATIBLE_VERSION.clone(),
        min_compatible: PREV_MINIMUM_COMPATIBLE_VERSION.clone(),
        upgrade_time,
        clock: Clock::real(),
    }
}
```

### 3.1 Where the rule is enforced (cross-ref `05`)

`ava-network` (`05` §3.2) holds an `Arc<Compatibility>` in the shared `PeerConfig`
(Go: `peer.Config.VersionCompatibility = version.GetCompatibility(minCompatibleTime)`).
The enforcement points, both copied from `network/peer/peer.go`:

- **`handle_handshake`** — on receipt of the peer's `Handshake`, parse its
  `Client{name,major,minor,patch}` into an `Application`, then run
  `should_disconnect`; if `!compatible(peer)` → **close the connection** before
  marking the peer connected (this is one of the §1.4 / §1.4-list disconnect reasons
  in `05`). Separately, if **our** `current < peer` version, log an *informational*
  "peer attempting to connect with newer version; you may want to update" — this is
  **not** a disconnect by itself (it only disconnects if the peer is below *our* floor
  by clause 2). Mirrors peer.go lines around the `myVersion.Compare(p.version) < 0`
  log.
- **`should_disconnect`** (called at handshake *and* on every Ping tick, every 22.5 s
  — `05` §1.5/§3.2) — re-runs `compatible(peer)`; returns true (disconnect) if the
  peer is no longer compatible. Re-checking on Ping accounts for the clock crossing
  `upgrade_time` mid-connection: a peer that was compatible under the pre-upgrade floor
  can become incompatible the instant the upgrade time passes, and is then dropped on
  the next tick. This is the safety mechanism that *clears old peers off the network at
  a fork boundary.*

> **Binding rule (drop-in):** `ava-network` MUST call `Compatibility::compatible`
> with exactly these inputs (peer's reported triple; our `current`; the floor selected
> by `clock vs upgrade_time`) at both points, and disconnect on `false`. No additional
> Rust-side version policy may be layered on (e.g. we must NOT reject a peer for a
> differing client `name`).

---

## 4. The `compatibility.json` model (the embedded compatibility table)

`version/compatibility.json` maps each **RPCChainVM protocol version** (string key) to
the **set of avalanchego versions** that shipped that protocol version. Structure
(abridged from the real file):

```json
{
  "45": ["v1.14.2"],
  "44": ["v1.14.0", "v1.14.1"],
  "43": ["v1.13.4", "v1.13.5"],
  "42": ["v1.13.3"],
  "...": ["..."],
  "16": ["v1.8.0", "v1.8.1", "v1.8.2", "v1.8.3", "v1.8.4", "v1.8.5", "v1.8.6"]
}
```

**What it is and is NOT.** It is `version.RPCChainVMProtocolCompatibility`, loaded via
`//go:embed compatibility.json`. The Go comment is explicit: *"This is not used by
avalanchego, but is useful for downstream libraries."* It is a **lookup table for VM
authors / tooling** ("which node releases can host my plugin built against protocol
44?") — it is **not** consulted in the connect/reject path (that is §3's numeric rule)
and **not** the network compatibility matrix. We reproduce it because:

- `info`/admin tooling and the plugin-management story (`07`) surface it;
- a golden test pins it byte-for-byte against the Go file (`00` §5).

### 4.1 Rust representation

Embed the JSON verbatim and parse it once. Keys are decimal strings → `u64`; values
are semantic version strings.

```rust
// crates/ava-version/src/compat_table.rs
use std::collections::BTreeMap;
use std::sync::LazyLock;

/// Mirrors `version.RPCChainVMProtocolCompatibility`. Map: rpcchainvm protocol
/// version -> the avalanchego releases that implemented it. Lookup/tooling only;
/// NOT used for peer accept/reject (that is `Compatibility::compatible`, §3).
/// Source: `version/compatibility.json` (embedded; golden-tested for byte parity).
static COMPATIBILITY_JSON: &str = include_str!("../compatibility.json");

pub static RPC_CHAIN_VM_PROTOCOL_COMPATIBILITY: LazyLock<BTreeMap<u64, Vec<String>>> =
    LazyLock::new(|| {
        // Go parses string keys; serde_json gives String keys we re-key to u64.
        let raw: BTreeMap<String, Vec<String>> =
            serde_json::from_str(COMPATIBILITY_JSON).expect("embedded compatibility.json is valid");
        raw.into_iter()
            .map(|(k, v)| (k.parse::<u64>().expect("protocol version key is decimal"), v))
            .collect()
    });
```

> **`compatibility.json` is a committed input** copied verbatim from the Go tree under
> `crates/ava-version/compatibility.json` (it is data, not a generated artifact, so it
> *is* checked in — distinct from `00` §11.1(8) generated artifacts). Bumping the node
> version updates both `constants.rs` and this file in the same change, exactly as Go
> bumps `constants.go` + `compatibility.json` together.

---

## 5. rpcchainvm protocol v45 (plugin compatibility)

Cross-ref `07` §11.1(1) for the reverse-dial handshake mechanics. The version concern:
host and plugin must agree on **`RPC_CHAIN_VM_PROTOCOL = 45` by exact equality.**

- The host (node) serves a `Runtime` gRPC server; the plugin dials back and calls
  `Runtime.Initialize(protocol_version, vm_addr)` (Go: `vms/rpcchainvm/vm.go` passes
  `version.RPCChainVMProtocol`; the runtime's `Initialize`, `runtime/subprocess/
  initializer.go`, compares it).
- **On mismatch** the host returns `ErrProtocolVersionMismatch` with a message naming
  both versions and the plugin path, and the plugin fails to load. Reproduce the
  sentinel and the message shape:

```rust
// crates/ava-vm-rpc/src/runtime.rs  (uses ava-version::RPC_CHAIN_VM_PROTOCOL)
#[derive(thiserror::Error, Debug)]
pub enum RuntimeError {
    #[error("RPCChainVM protocol version mismatch between AvalancheGo and Virtual Machine plugin")]
    ProtocolVersionMismatch,
    // ...
}

/// Mirrors subprocess/initializer.go `Initialize`. Called once (guard with a `Once`).
pub fn check_protocol(plugin_protocol: u64, plugin_path: &str) -> Result<(), RuntimeError> {
    if ava_version::RPC_CHAIN_VM_PROTOCOL != plugin_protocol {
        // Message mirrors Go: names node version, host protocol, plugin path, plugin protocol.
        tracing::error!(
            host_version = %ava_version::CURRENT.display(),
            host_protocol = ava_version::RPC_CHAIN_VM_PROTOCOL,
            %plugin_path,
            plugin_protocol,
            "RPCChainVM protocol version mismatch; update the VM or the node so the protocol versions match exactly"
        );
        return Err(RuntimeError::ProtocolVersionMismatch);
    }
    Ok(())
}
```

**Interop both directions** (the `00` §1 gRPC-plugin requirement): a Rust plugin
hosted by a Go node, or a Go plugin hosted by a Rust node, must each report/expect `45`.
Because the check is exact equality and the constant is shared from `ava-version`, a
Rust plugin built against this workspace is hostable by Go `1.14.2` (which also
implements `45`) and vice-versa. A Rust *node* hosting a Go plugin built against `44`
rejects it identically to how a Go node would.

---

## 6. Database / schema versioning (cross-ref `04` R2)

- The on-disk version is recorded **as the database subdirectory name**. Go
  (`node/node.go`): for leveldb the dir is `<db-path>/<networkID>/v1.4.5`
  (`version.CurrentDatabase`); pebble uses `pebble`; other backends use `db`. The Rust
  port (RocksDB default per `00` §4.4) opens a dir named for `CURRENT_DATABASE`.
- There is no in-file schema-version byte separate from this folder convention; the
  folder name **is** the schema version. `PrevDatabase = "v1.0.0"` records the prior
  schema for migration logic.
- **Migration trigger:** when the node finds a data directory written by a *prior*
  schema version (or, per `04` R2, a Go-written Pebble/LevelDB dir), it must run a
  migration/import rather than open it in place. `04` R2 already records that RocksDB
  replaces both Go KV backends, so bootstrapping from a Go data dir is an **import
  path, not in-place open**; the schema-version folder name is how that condition is
  detected. Until the import tool exists, the node documents non-support and refuses to
  start against a foreign/older dir (rather than silently corrupting it).
- `databaseVersion` is reported in `info.getNodeVersion` as `CURRENT_DATABASE`
  (`14`), byte-identical to Go (`v1.4.5`).

---

## 7. Upgrade / downgrade story

### 7.1 Rolling upgrades & the moving floor

The min-compatible floor (#2, §3) is what makes a **rolling upgrade** safe across a
heterogeneous fleet:

- A network running `1.13.x` upgrades node-by-node to `1.14.x`. During the roll, a
  `1.14.2` node has `current = 1.14.2`, post-upgrade floor `1.14.0`, pre-upgrade floor
  `1.13.0`. **Before** the configured `upgrade_time`, the floor is `1.13.0`, so the new
  node still accepts the not-yet-upgraded `1.13.x` peers. **After** `upgrade_time`, the
  floor rises to `1.14.0` and stragglers below it are dropped on the next Ping tick
  (§3.1). This is the controlled cut-over.
- Therefore the operator-visible upgrade procedure (upgrade all nodes *before* the fork
  time) is enforced by the same machinery in the Rust port with no behavioral change.

### 7.2 Interaction with the time-based fork schedule (`03` §11)

Forks (#6) activate by **wall-clock time**, identical on every node of a network
(Mainnet/Fuji each have their own baked-in `upgrade/` times — `03` §11). The
relationship:

- The `upgrade_time` threaded into `Compatibility` is the **fork time that gates the
  floor switch** (Go: `network.go`'s `minCompatibleTime`). So the network-version floor
  and the protocol-rule activation move together at the same instant.
- **"Don't activate a fork the binary doesn't understand" safety:** a node only knows
  the fork times baked into *its* binary. It cannot be tricked by peers into activating
  a rule it lacks code for, because fork activation reads the *local* clock against
  *local* baked-in times — it is not negotiated. Conversely, a node whose binary lacks
  an upcoming fork's rules is below the min-compatible floor for the post-fork network
  and is **dropped at the fork time** (§3.1), so it cannot continue producing/validating
  blocks under stale rules on the upgraded network. The floor (#2) and the fork clock
  (#6) jointly guarantee that the connected set at any time agrees on the active rules.
- The Rust port copies the fork times verbatim into `ava-version` (`00` §5; `03` §11)
  and must **never** activate a fork its code does not implement — the same invariant as
  Go. Differential tests (`02`) pin the activation timestamps.

### 7.3 Downgrade

A downgrade is the reverse roll and is only safe **before** the relevant fork time and
only to a version `≥` the floor the rest of the fleet enforces. After a fork activates,
downgrading below `MinimumCompatibleVersion` disconnects the node (§3.1). The Rust port
inherits this exactly; there is no Rust-specific downgrade path.

---

## 8. Go → Rust mapping (subsystem-specific)

| Go | Rust |
|---|---|
| `version.Application` (+ `sync.Once` memoized `String()`) | `Application` struct; `Display`/`display()` (cache via `OnceLock` if profiled hot) |
| `Application.Compare` (`cmp.Compare` chain) | `Ord` on `(major, minor, patch)` tuple (name excluded) |
| `version.Compatibility` + `Compatible(peer)` | `Compatibility` + `compatible(peer)` (§3) |
| `version.GetCompatibility(upgradeTime)` | `get_compatibility(upgrade_time)` |
| `mockable.Clock` | `Clock` (real/mock) injected into `Compatibility` (mirrors `03` clock) |
| `//go:embed compatibility.json` | `include_str!` + `serde_json` → `LazyLock<BTreeMap<u64, Vec<String>>>` |
| `version.Versions` + `GetVersions()` + `String()` | `Versions` + `Display` (§2.3) |
| `version.GitCommit` (build-time `-ldflags`) | `env!("AVA_GIT_COMMIT")` via `build.rs` |
| `runtime.Version()` → `go` field | rustc version string in the `go`-keyed field (schema kept) |
| `RPCChainVMProtocol` exact-equality check | `check_protocol` + `RuntimeError::ProtocolVersionMismatch` (§5) |
| `version.CurrentDatabase` folder name | `CURRENT_DATABASE` RocksDB dir name (§6) |

**Error model** (`00` §7.1). `ava-version` is mostly data; its only failure modes are
the embedded-JSON parse (an invariant — `expect`, since a bad embed is a build bug) and
the rpcchainvm mismatch, which lives in `ava-vm-rpc` as
`RuntimeError::ProtocolVersionMismatch` (a typed sentinel matched via `matches!`,
mirroring `errors.Is(ErrProtocolVersionMismatch)`). The network-side disconnect-on-
incompatible is not an `Error` value — it is a `bool` from `compatible()` consumed by
`05`'s `should_disconnect` (matching Go).

---

## 9. Test plan (cross-ref `02`)

1. **Version-string golden vs Go.** `Application{avalanchego,1,14,2}.display()` ==
   `"avalanchego/1.14.2"`; `semantic()` == `"v1.14.2"`;
   `semantic_with_commit("abc")` == `"v1.14.2@abc"`; the empty-commit case ==
   `"v1.14.2"`. Pin against bytes captured from the Go `Application.String()` /
   `Semantic()` / `SemanticWithCommit()`. Also pin `Versions::Display` output shape
   (`"avalanchego/1.14.2 [database=v1.4.5, rpcchainvm=45, go=...]"`, with and without
   `commit=`).
2. **`info.getNodeVersion` golden** (`14`): the JSON reply field-for-field matches a
   Go node's (`version`, `databaseVersion=v1.4.5`, `rpcProtocolVersion=45`,
   `gitCommit`, `vmVersions`), modulo the build-specific `gitCommit`/`go` values.
3. **Handshake-compatibility matrix.** A table test over `(our_current, floor_pre,
   floor_post, upgrade_time, clock, peer_version) → accept/reject`, asserting
   `compatible()` matches the Go `Compatible` for every cell. Mandatory cells:
   - peer on a **newer major** than ours → reject (clause 1).
   - peer **below the pre-upgrade floor** with `clock < upgrade_time` → reject.
   - same peer with `clock < upgrade_time` but `≥ pre-floor` → accept.
   - peer `≥ pre-floor` but `< post-floor`, with `clock ≥ upgrade_time` → **reject**
     (the fork-boundary cut-over).
   - peer == our `current` → accept; peer **newer** (same major) → accept (we only log).
   - peer with a **different `name`** but compatible triple → accept (name not compared).
   - the *mid-connection* transition: a peer accepted before `upgrade_time` is rejected
     by `should_disconnect` after the clock crosses it (mock clock).
4. **Cross-port differential** (`02` harness): a Rust node and a Go node on the same
   local network complete the P2P handshake (`05` §9.9) — the Go node logs the Rust
   peer's version as `avalanchego/1.14.2` and reports it connected; a Rust node with its
   numeric triple lowered below the Go node's floor is dropped by the Go node, and
   vice-versa.
5. **Plugin-protocol-mismatch test.** `check_protocol(45, path)` → `Ok`;
   `check_protocol(44, path)` → `Err(ProtocolVersionMismatch)` with the message naming
   both versions. A Go plugin built against `44` is rejected by the Rust host
   identically to the Go host (cross-ref `07` test plan).
6. **`compatibility.json` byte-parity golden** (`00` §5): the embedded file parses to a
   map equal to the Go `RPCChainVMProtocolCompatibility`, and the checked-in file is
   byte-identical to `version/compatibility.json`.
7. **Database-version test** (`04`): the opened DB dir name == `v1.4.5`; a dir written
   by a `PrevDatabase`/foreign backend triggers the migration/refusal path, not an
   in-place open.

---

## 10. Performance notes / improvements over Go

Versioning is **not** on any hot path; performance is trivial. The only micro-notes,
each "safe because" it changes no observable behavior (`00` §6.1/§9):

- `Application::display()` is a tiny `format!`; cache in a `OnceLock` only if a profile
  ever shows it hot (it is logged once per peer at connect, far from the message loop).
  *Safe:* the string is deterministic.
- `compatible()` is a handful of integer comparisons; called per-handshake and per-Ping
  (22.5 s) — negligible. No optimization warranted.
- The embedded `compatibility.json` is parsed once into a `LazyLock<BTreeMap>`; lookups
  are tooling-only and not latency-sensitive. *Safe:* read-only after init.
