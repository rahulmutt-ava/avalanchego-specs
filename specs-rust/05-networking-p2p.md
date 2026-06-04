# 05 — Networking & P2P (`ava-message`, `ava-network`)

> **Status:** Subsystem spec. Conforms to `00-overview-and-conventions.md` (binding:
> crate names, library table §4, compatibility-surface §1, conventions §6–§9).
> This document is **standalone** but cross-references `03-core-primitives.md`
> (ids/codec/crypto/TLS certs), `06-consensus.md` (the router → engine handoff),
> `07-vm-framework.md` (the AppRequest/AppGossip SDK consumers), `12-node-config-api-wallet.md`
> (config flags), and `02-testing-strategy.md` (differential harness).

**The P2P wire protocol MUST be byte-exact for interoperability with Go nodes.**
A Rust node must join Mainnet/Fuji and be indistinguishable from a Go peer on the
wire: identical TLS handshake, identical length-framing, identical protobuf
encoding, identical op semantics, identical peer-list gossip and ping/pong/uptime.
Everything in §1 (Wire Protocol) is a hard compatibility surface; §2–§6 are
implementation freedom in Rust as long as the bytes on the socket are unchanged.

---

## 0. Go source map

| Go area | What it is | Rust home |
|---|---|---|
| `message/ops.go` | `Op` opcodes, `Unwrap`/`ToOp`, `UnrequestedOps`, `FailedToResponseOps` | `ava-message::ops` |
| `message/messages.go` | `msgBuilder` (proto marshal + zstd compression), `InboundMessage`/`OutboundMessage` | `ava-message::codec` |
| `message/outbound_msg_builder.go`, `inbound_msg_builder.go`, `creator.go`, `internal_msg_builder.go`, `fields.go` | builder API, `Creator` | `ava-message::builder` |
| `proto/p2p/p2p.proto` | **the wire schema** (`Message` oneof + all sub-messages) | `ava-message` (prost-generated `p2p` module) |
| `proto/sdk` (`network/p2p/...`) | app-level request/response + gossip framing | `ava-network::p2p` |
| `network/peer/{peer,config,network,message_queue,msg_length,upgrader,tls_config,ip,ip_signer,info,set}.go` | per-peer state machine, framing, TLS upgrade, IP signing | `ava-network::peer` |
| `network/network.go`, `config.go`, `ip_tracker.go`, `tracked_ip.go` | the `Network` service, peer-list gossip, reconnection | `ava-network::network` |
| `network/dialer/` | outbound dialer + dial throttle | `ava-network::dialer` |
| `network/throttling/` | inbound/outbound msg byte throttlers, bandwidth, conn-upgrade, dial | `ava-network::throttling` |
| `network/p2p/` (`router`, `client`, `handler`, `throttler`, `node_sampler`, `gossip/`, `acp118/`) | app SDK on top of AppRequest/AppResponse/AppGossip | `ava-network::p2p` |
| `snow/networking/router/inbound_handler.go` (`InboundHandler`, `ExternalHandler`) | the consensus-facing routing trait | trait defined here, **consumed by `06`** |
| `nat/` | UPnP + NAT-PMP port mapping | `ava-network::nat` |
| `ids/node_id.go` (`NodeIDFromCert`), `staking/`, `utils/ips`, `utils/bloom` | NodeID derivation, certs, IP types, bloom filter | `ava-crypto` / `ava-types` (`03`) |
| `utils/constants/networking.go` | all network constants | `ava-types::constants` |

---

## 1. Wire protocol (HARD compatibility surface — byte-exact)

### 1.1 Transport & framing

- Transport: **TCP** (`constants.NetworkType = "tcp"`), upgraded to **TLS 1.3** (§1.6).
- Optionally fronted by the **PROXY protocol v1/v2** when `network-tcp-proxy-enabled`
  (`pires/go-proxyproto`); off by default (`DefaultNetworkTCPProxyEnabled=false`).
- After the TLS handshake, the channel carries a stream of **length-prefixed
  protobuf messages**. Framing (`network/peer/msg_length.go`, `peer.go`):
  - **4-byte big-endian `uint32` length prefix** (`wrappers.IntLen = 4`), then exactly
    that many bytes of message payload.
  - Max payload = **`DefaultMaxMessageSize = 2 MiB`** (`2 * units.MiB`). A length
    `> 2 MiB` is a protocol error → drop the connection. (Go `readMsgLen`/`writeMsgLen`.)
  - There is **no** explicit compression flag in the frame; compression is encoded
    *inside* the protobuf (§1.3).
- Write path uses **vectored I/O**: Go writes `net.Buffers{lenBytes, msgBytes}` in one
  `writev`. Rust mirrors this (§4.3) — do not coalesce into a fresh allocation.
- Read path: read 4 bytes → parse len → acquire from inbound byte throttler (§5) →
  read `len` bytes → parse. Read/write deadlines are reset to `now + PongTimeout`
  (`DefaultPingPongTimeout = 30s`) on every framing step.

> **Byte-exactness note:** the only bytes on the socket are `len_be_u32 || proto_bytes`.
> `proto_bytes` is the canonical proto3 encoding of `p2p.Message`. Because the
> Go side uses `google.golang.org/protobuf` and we use `prost`, **field ordering and
> default-elision must match.** proto3 wire encoding is deterministic per field
> number; both libraries emit fields in ascending field-number order and omit
> zero/empty scalar fields, so encodings coincide. The golden-vector test (§9)
> guards this.

### 1.2 Op-code table — SOURCE OF TRUTH

`message/ops.go` defines `Op byte` with `iota` ordering. These integer values are an
**internal routing/metrics tag, not on the wire** (the wire identity is the protobuf
`oneof` field number, §1.3). We still reproduce the exact `Op` enum because it is the
source of truth other crates (`06`, `07`) match on, and metrics labels use the
`String()` names.

| `Op` (value) | `String()` | proto field # in `Message.message` | Category |
|---|---|---|---|
| `Ping` (0) | `ping` | 11 | Handshake |
| `Pong` (1) | `pong` | 12 | Handshake |
| `Handshake` (2) | `handshake` | 13 | Handshake |
| `GetPeerList` (3) | `get_peerlist` | 35 | Handshake |
| `PeerList` (4) | `peerlist` | 14 | Handshake |
| `GetStateSummaryFrontier` (5) | `get_state_summary_frontier` | 15 | State sync |
| `GetStateSummaryFrontierFailed` (6) | `…_failed` | — (internal) | State sync |
| `StateSummaryFrontier` (7) | `state_summary_frontier` | 16 | State sync |
| `GetAcceptedStateSummary` (8) | `get_accepted_state_summary` | 17 | State sync |
| `GetAcceptedStateSummaryFailed` (9) | `…_failed` | — (internal) | State sync |
| `AcceptedStateSummary` (10) | `accepted_state_summary` | 18 | State sync |
| `GetAcceptedFrontier` (11) | `get_accepted_frontier` | 19 | Bootstrap |
| `GetAcceptedFrontierFailed` (12) | `…_failed` | — (internal) | Bootstrap |
| `AcceptedFrontier` (13) | `accepted_frontier` | 20 | Bootstrap |
| `GetAccepted` (14) | `get_accepted` | 21 | Bootstrap |
| `GetAcceptedFailed` (15) | `…_failed` | — (internal) | Bootstrap |
| `Accepted` (16) | `accepted` | 22 | Bootstrap |
| `GetAncestors` (17) | `get_ancestors` | 23 | Bootstrap |
| `GetAncestorsFailed` (18) | `…_failed` | — (internal) | Bootstrap |
| `Ancestors` (19) | `ancestors` | 24 | Bootstrap |
| `Get` (20) | `get` | 25 | Consensus |
| `GetFailed` (21) | `get_failed` | — (internal) | Consensus |
| `Put` (22) | `put` | 26 | Consensus |
| `PushQuery` (23) | `push_query` | 27 | Consensus |
| `PullQuery` (24) | `pull_query` | 28 | Consensus |
| `QueryFailed` (25) | `query_failed` | — (internal) | Consensus |
| `Chits` (26) | `chits` | 29 | Consensus |
| `AppRequest` (27) | `app_request` | 30 | Application |
| `AppError` (28) | `app_error` | 34 | Application |
| `AppResponse` (29) | `app_response` | 31 | Application |
| `AppGossip` (30) | `app_gossip` | 32 | Application |
| `Connected` (31) | `connected` | — (internal) | Internal |
| `Disconnected` (32) | `disconnected` | — (internal) | Internal |
| `Notify` (33) | `notify` | — (internal) | Internal |
| `GossipRequest` (34) | `gossip_request` | — (internal) | Internal |
| `Simplex` (35) | `simplex` | 36 | Simplex |

Plus the reserved compression field `compressed_zstd = 2` (§1.3). Field `1` and `37`
are reserved (`reserved 1; reserved 37;`), as are several intra-message fields (the
`reserved 4/5` markers in `GetAcceptedFrontier`/`Get`/etc — these correspond to a
removed `engine_type` field and **must stay unset** so we don't accidentally re-use
the number).

Two classification sets must be reproduced verbatim (used by the timeout/response
bookkeeping in `06`):
- **`UNREQUESTED_OPS`** = {`GetAcceptedFrontier`, `GetAccepted`, `GetAncestors`, `Get`,
  `PushQuery`, `PullQuery`, `AppRequest`, `AppGossip`, `GetStateSummaryFrontier`,
  `GetAcceptedStateSummary`, `Simplex`}.
- **`FAILED_TO_RESPONSE_OPS`** maps each `*Failed` internal op to its successful
  counterpart (e.g. `GetFailed→Put`, `QueryFailed→Chits`, `AppError→AppResponse`).

### 1.3 Compression negotiation (recursive packing — exact)

avalanchego avoids a compression flag by **nesting a `Message` inside a `Message`**
(`message/messages.go::marshal`):

1. Marshal the real message `M` → `uncompressed_bytes`.
2. If compression enabled (`DefaultNetworkCompressionType = zstd`): `c = zstd(uncompressed_bytes)`,
   then build `Message{ compressed_zstd: c }` (field 2) and marshal *that*.
3. The receiver unmarshals the outer `Message`; if `compressed_zstd` is non-empty it
   `zstd_decompress` it and unmarshals the inner `Message` (`unmarshal`).

Rules for byte-exactness:
- Only **zstd** is supported (`compression.TypeZstd`). gzip/snappy are *not* used on
  the modern wire — fields `<10` are reserved for "other compression algorithms" but
  none is active. `ava-message` still vendors a `flate2` gzip path **only** for legacy
  decode tolerance; it is never produced. (Citation: `messages.go` only branches on
  `TypeNone`/`TypeZstd`; `errUnknownCompressionType` for anything else.)
- Which ops are compressed is decided by the **outbound builder** per message
  (`outbound_msg_builder.go` passes `compressionType` per op). Handshake-class
  messages and ping/pong are generally sent uncompressed; bulk messages
  (Put/Ancestors/PushQuery/App*) are zstd. We copy the per-op decision exactly.
- The zstd encoder must be configured so output is byte-compatible. Go uses
  `klauspost/compress/zstd` with a fixed window (max = 2 MiB). zstd output is not
  uniquely defined by the spec, **but compatibility does not require identical
  compressed bytes** — only that each side can *decode* the other. The decode path
  is fully deterministic. So we standardize the encoder (`zstd` crate, default level)
  and rely on cross-decodability; the differential test confirms a Go node accepts our
  frames and vice-versa. (This is the one place where exact byte-equality is **not**
  required, only round-trip interop.)

### 1.4 Handshake sequence (exact ordering)

Per `network/peer/peer.go`. After the TLS upgrade completes and the NodeID is derived
from the peer cert (§1.6), three tasks spin up (`readMessages`, `writeMessages`,
`sendNetworkMessages`). The application-level handshake:

```
        local                                   remote
  (writeMessages first action)
  ── Handshake (op 13) ───────────────────────────►   handleHandshake:
                                                        - networkID match
                                                        - |myTime - peerTime| ≤ MaxClockDifference (60s)
                                                        - parse Client version; check compat
                                                        - ≤ maxNumTrackedSubnets (16) subnets
                                                        - verify SignedIP (TLS sig over ip||port||ts)
                                                        - verify BLS PoP sig (if validator w/ BLS key)
                                                        - gotHandshake = true
  handleHandshake (same checks)  ◄──── Handshake ──┘
                                                       sends ► PeerList (op 14, bypassThrottling)
  handlePeerList:                ◄──── PeerList ─────┘
   - if !finishedHandshake && gotHandshake:
        Network.Connected(id); finishedHandshake=true; close(onFinishHandshake)
   - Track(discoveredIPs)
  sends ► PeerList in response to remote Handshake
```

Key invariants (must reproduce):
- **The first message each side writes is `Handshake`** (forced in `writeMessages`
  before the queue loop). It carries: `network_id`, `my_time` (unix s), `ip_addr`
  (16-byte form via `As16()`), `ip_port`, `upgrade_time`, `ip_signing_time`,
  `ip_node_id_sig` (TLS sig), `tracked_subnets`, `Client{name,major,minor,patch}`,
  `supported_acps`, `objected_acps`, `known_peers` (bloom filter+salt), `ip_bls_sig`
  (BLS PoP), `all_subnets`.
- A peer **must** respond to an inbound `Handshake` with a `PeerList`; receiving the
  `PeerList` completes the handshake. `GetPeerList` is **not** answered until the
  handshake is finished.
- Disconnect (close conn) on: wrong `network_id`; clock skew > 60s; incompatible
  version; > 16 tracked subnets; `supported_acps ∩ objected_acps ≠ ∅`; invalid IP /
  zero port; invalid TLS-IP signature; invalid BLS sig for a registered validator;
  duplicate Handshake; bloom salt > `maxBloomSaltLen = 32`.
- ACPs are filtered to `constants.CurrentACPs` on receipt.

### 1.5 Ping / Pong & uptime tracking

- `sendNetworkMessages` ticks every **`PingFrequency = 3/4 * 30s = 22.5s`**
  (`DefaultPingFrequency`). Each tick: (a) re-check `AllowConnection` and (after
  handshake) `shouldDisconnect`; (b) send `Ping{ uptime }` where `uptime ∈ [0,100]`
  is *our* primary-network uptime as computed by the local uptime calculator
  (`UptimeCalculator.CalculateUptimePercent` × 100).
- On `Ping`: store the peer's claimed `uptime` (reject & close if `> 100`), reply `Pong`.
- On `Pong`: compute RTT = `now - lastPingSent` (ms), record metrics, clear
  `lastPingSent`. An unsolicited `Pong` (no ping outstanding) closes the connection.
- A peer that doesn't respond within `PongTimeout` (the read deadline) is dropped by
  the read loop's deadline expiring.
- `Ping`/`Pong` are the liveness + observed-uptime mechanism: `ObservedUptime()` is what
  *peers think our uptime is*; surfaced to the P-Chain uptime API (`06`/`12`).

### 1.6 TLS & identity (must match `crypto/tls` so Go↔Rust handshake)

`network/peer/tls_config.go`, `upgrader.go`, `ids/node_id.go`, `staking/tls.go`.

- **TLS config (both directions):**
  - `MinVersion = TLS 1.3` (TLS13-only). This is the single most important interop
    constant: both peers negotiate from the TLS 1.3 suite set, so cipher suites
    overlap automatically (`AES_128_GCM_SHA256`, `AES_256_GCM_SHA384`,
    `CHACHA20_POLY1305_SHA256`) — all three are offered by both Go's `crypto/tls` and
    rustls. We do **not** restrict suites below the TLS1.3 default set.
  - Server: `ClientAuth = RequireAnyClientCert` → **mutual TLS, any (self-signed) cert
    accepted**. `InsecureSkipVerify = true` + a `VerifyConnection` callback
    (`ValidateCertificate`) that checks **only the leaf public key**, not a CA chain.
  - **No SNI / hostname verification.** Connections are authenticated purely by the
    peer's public key → derived NodeID. (Audited by Quantstamp per the Go comment.)
- **`ValidateCertificate`** (the leaf-key policy we must reproduce exactly):
  - ≥1 peer cert present; leaf non-nil.
  - `ECDSA` ⇒ curve **must be P-256** (else reject).
  - `RSA` ⇒ `staking.ValidateRSAPublicKeyIsWellFormed` (modulus length rules, see `03`;
    `allowedRSASmallModulusLen = 2048`).
  - Any other key type ⇒ reject.
- **NodeID derivation (`ids.NodeIDFromCert`)** — cross-ref `03`:
  `NodeID = RIPEMD160( SHA256( cert.Raw ) )` → a **20-byte** ID. `cert.Raw` is the DER
  of the leaf cert. This must be bit-identical; it is the network identity.
- **Staking cert generation (`staking/tls.go`)** — for `init`/new nodes:
  ECDSA P-256 self-signed, `SerialNumber=0`, `NotBefore=2000-01-01`(ish, `time.Date(2000,Jan,0,...)`),
  `NotAfter=+100y`, `KeyUsage=DigitalSignature`, `BasicConstraintsValid=true`, PKCS#8
  PEM key + PEM cert. We reproduce this with `rcgen` so a Rust-generated identity is
  indistinguishable from a Go-generated one (and existing Go `staker.crt`/`staker.key`
  files load unchanged — see `12`).
- **IP signing (`ip.go`, `ip_signer.go`):** the signed bytes are
  `ip.As16() (16) || port (u16 BE) || timestamp (u64 BE)` (`UnsignedIP::bytes` via
  `wrappers.Packer`). The **TLS signature** is over `SHA256(ipBytes)` using the staking
  key (`crypto.SHA256`); the **BLS signature** is a *proof-of-possession* signature over
  the raw `ipBytes`. Verification: `SignedIP::Verify` requires `timestamp ≤ maxTimestamp`
  (= now + 60s) and a valid TLS sig over `ipBytes` from the peer cert. Registered
  validators with a BLS key must additionally sign with BLS (checked in
  `shouldDisconnect`). Signed-IP caching: re-sign only when the dynamic IP changes.

> Why a custom verifier is mandatory in Rust: rustls' default verifiers require a CA
> chain. avalanche uses self-signed leaf certs as raw identity, so we implement a
> `danger::ServerCertVerifier` (client side) and `danger::ClientCertVerifier` (server
> side) that run `ValidateCertificate` then return `…Verified::assertion()`. See §4.5.

---

## 2. `ava-message` design

Crate `ava-message`. Responsibilities: the generated proto types, the `Op` enum +
classification sets, the inbound/outbound codec (proto marshal + zstd recursive
packing + 4-byte framing helpers), and the builder API. Zero-copy on the read path
with `bytes::Bytes`.

### 2.1 Generated proto

`build.rs` runs `prost-build`/`tonic-build` over `proto/p2p/p2p.proto` →
`pub mod p2p` with `Message`, the `message::Message` `oneof` enum, and all sub-message
structs (`Ping`, `Handshake`, `PeerList`, `ClaimedIpPort`, `Get`, `Put`, `Chits`,
`AppRequest`, `Simplex`, …). prost generates an `Option<message::Message>` for the
`oneof`. `prost` and Go's protobuf produce identical proto3 wire bytes (§1.1 note).

### 2.2 The `Op` enum & `Message` wrapper

```rust
/// Opcode — internal routing/metrics tag (NOT on the wire). Source of truth for `06`/`07`.
/// Values mirror `message/ops.go` `iota` exactly; do not reorder.
#[derive(Clone, Copy, PartialEq, Eq, Hash, Debug)]
#[repr(u8)]
pub enum Op {
    Ping = 0, Pong, Handshake, GetPeerList, PeerList,
    GetStateSummaryFrontier, GetStateSummaryFrontierFailed, StateSummaryFrontier,
    GetAcceptedStateSummary, GetAcceptedStateSummaryFailed, AcceptedStateSummary,
    GetAcceptedFrontier, GetAcceptedFrontierFailed, AcceptedFrontier,
    GetAccepted, GetAcceptedFailed, Accepted,
    GetAncestors, GetAncestorsFailed, Ancestors,
    Get, GetFailed, Put, PushQuery, PullQuery, QueryFailed, Chits,
    AppRequest, AppError, AppResponse, AppGossip,
    Connected, Disconnected, Notify, GossipRequest, Simplex,
}

impl Op {
    pub fn as_str(self) -> &'static str { /* exact strings from ops.go */ }
    /// `Message_*` oneof variant → Op (mirrors `ToOp`).
    pub fn of(m: &p2p::message::Message) -> Result<Op, Error>;
}

/// A decoded inbound message handed to the router. Owns its proto payload.
pub struct InboundMessage {
    pub node_id: NodeId,
    pub op: Op,
    pub message: p2p::message::Message,   // the unwrapped oneof variant
    pub expiration: Option<Instant>,      // min(deadline, max_message_timeout)
    pub bytes_saved_compression: i64,
    pub(crate) on_finished: OnFinished,   // Drop-guard → throttler release (§5)
}

/// Ready-to-send framed bytes (length prefix is added at write time).
pub struct OutboundMessage {
    pub bypass_throttling: bool,
    pub op: Op,
    pub bytes: Bytes,                     // proto bytes (possibly zstd-wrapped)
    pub bytes_saved_compression: i64,
}
```

`OnFinished` is a small RAII guard replacing Go's `onFinishedHandling func()`. Go's
contract — "call `OnFinishedHandling` exactly once or the throttler leaks" — becomes
*"the throttler permit is released on `Drop`"*, which is leak-proof by construction
(an improvement over Go's manual discipline; see §10).

### 2.3 Codec (`MsgBuilder`)

```rust
pub struct MsgBuilder {
    zstd: ZstdCodec,                  // window/limit = DefaultMaxMessageSize (2 MiB)
    max_message_timeout: Duration,
    metrics: CompressionMetrics,      // codec_compressed_count / _duration (same names)
}

impl MsgBuilder {
    /// Marshal + (optional) recursive-zstd-pack. Returns (bytes, bytes_saved, op).
    fn marshal(&self, m: &p2p::Message, c: Compression) -> Result<(Bytes, i64, Op)>;
    /// Unmarshal outer Message; if compressed_zstd set, decompress + unmarshal inner.
    fn unmarshal(&self, b: &[u8]) -> Result<(p2p::Message, i64, Op)>;
    pub fn create_outbound(&self, m: p2p::Message, c: Compression, bypass: bool)
        -> Result<OutboundMessage>;
    pub fn parse_inbound(&self, b: Bytes, node: NodeId, on_finished: OnFinished)
        -> Result<InboundMessage>;
}
```

Framing helpers (`ava-message::frame`), byte-exact with `msg_length.go`:

```rust
pub const MAX_MESSAGE_SIZE: u32 = 2 * 1024 * 1024;        // DefaultMaxMessageSize
pub fn write_msg_len(buf: &mut BytesMut, len: u32) -> Result<()>; // BE u32, err if > max
pub fn read_msg_len(b: [u8; 4], max: u32) -> Result<u32>;          // BE u32, err if > max
```

### 2.4 Builder API (`Creator = OutboundMsgBuilder + InboundMsgBuilder`)

Mirror `message::Creator`. One method per outbound op; each constructs the proto and
calls `create_outbound` with the per-op compression choice copied from Go:

```rust
pub trait OutboundMsgBuilder {
    fn handshake(&self, network_id: u32, my_time: u64, ip: SocketAddr,
                 client_name: &str, major: u32, minor: u32, patch: u32,
                 upgrade_time: u64, ip_signing_time: u64,
                 tls_sig: &[u8], bls_sig: &[u8], tracked_subnets: &[Id],
                 supported_acps: &[u32], objected_acps: &[u32],
                 known_peers_filter: &[u8], known_peers_salt: &[u8],
                 all_subnets: bool) -> Result<OutboundMessage>;
    fn ping(&self, uptime: u32) -> Result<OutboundMessage>;
    fn pong(&self) -> Result<OutboundMessage>;
    fn get_peer_list(&self, filter: &[u8], salt: &[u8], all_subnets: bool) -> Result<OutboundMessage>;
    fn peer_list(&self, peers: &[ClaimedIpPort], bypass: bool) -> Result<OutboundMessage>;
    fn get(&self, chain: Id, request_id: u32, deadline: Duration, container: Id) -> Result<OutboundMessage>;
    fn put(&self, chain: Id, request_id: u32, container: Bytes) -> Result<OutboundMessage>;
    fn push_query(&self, chain: Id, request_id: u32, deadline: Duration, container: Bytes, height: u64) -> Result<OutboundMessage>;
    fn chits(&self, chain: Id, request_id: u32, preferred: Id, accepted: Id, preferred_at_height: Id, accepted_height: u64) -> Result<OutboundMessage>;
    fn app_request(&self, chain: Id, request_id: u32, deadline: Duration, app_bytes: Bytes) -> Result<OutboundMessage>;
    // … get_accepted_frontier, accepted, get_ancestors, ancestors, app_response,
    //    app_error, app_gossip, state-sync ops, simplex …
}
```

`deadline` is encoded as **nanoseconds (`u64`)** in the proto (`deadline (ns)`),
matching Go. Inbound deadline → `expiration = now + min(deadline, max_message_timeout)`.

---

## 3. `ava-network` design

Crate `ava-network`. The runtime: a TCP listener, a dialer, per-peer actors, the
peer-list gossip / IP-tracker, throttlers, and the routing handoff to consensus.

### 3.1 The `Network` trait & service

```rust
#[async_trait]
pub trait Network: Send + Sync {
    /// Run until closed or fatal error (mirrors Dispatch()).
    async fn dispatch(self: Arc<Self>) -> Result<()>;
    fn start_close(&self);                                  // idempotent

    fn manually_track(&self, node_id: NodeId, ip: SocketAddr);
    fn peer_info(&self, node_ids: &[NodeId]) -> Vec<PeerInfo>;
    fn node_uptime(&self) -> Result<UptimeResult>;

    // sender::ExternalSender — outbound consensus/app messages (consumed by `06`):
    fn send(&self, msg: OutboundMessage, cfg: SendConfig, subnet: Id, allower: &dyn Allower) -> HashSet<NodeId>;
    fn gossip(&self, msg: OutboundMessage, subnet: Id, cfg: GossipConfig, allower: &dyn Allower) -> HashSet<NodeId>;

    // health::Checker, peer::Network (Connected/Disconnected/Track/KnownPeers/Peers) …
}
```

The concrete `NetworkImpl` holds (mirrors the Go `network` struct):
- `peer_config: Arc<PeerConfig>` — shared per-peer config (message creator, throttlers,
  router handle, validators, version-compat, IP signer, clock, metrics).
- `listener`, `dialer`, `server_upgrader`, `client_upgrader` (TLS, §4.5).
- `ip_tracker` (peer-list gossip state), `connecting_peers`, `connected_peers`
  (`PeerSet`s), `tracked_ips: DashMap<NodeId, TrackedIp>`.
- `outbound_msg_throttler`, `inbound_conn_upgrade_throttler` (§5).
- A root `CancellationToken` for shutdown (replaces `onCloseCtx`).

**Lock discipline** (from the Go comment): acquire `peers` lock before
`manually_tracked_ids` lock. In Rust we prefer `DashMap` / `arc-swap` for the peer
sets to avoid a global `RwLock` (lock-free reads; see §10), keeping the documented
order where a single lock still spans both.

### 3.2 The `Peer` actor (task-per-peer)

Go runs **three goroutines per peer** (`readMessages`, `writeMessages`,
`sendNetworkMessages`) sharing a `Peer` struct + a `MessageQueue`. Rust maps this to
**three tokio tasks per peer** plus a command channel:

```rust
pub struct PeerHandle {
    id: NodeId,
    cmd_tx: mpsc::Sender<PeerCommand>,   // Send/StartSendGetPeerList/StartClose
    on_finish_handshake: Notify,
    on_closed: Notify,
}

enum PeerCommand { Send(OutboundMessage), GetPeerList, Close }

struct Peer {
    cfg: Arc<PeerConfig>,
    conn_read: ReadHalf,  conn_write: WriteHalf,     // split TLS stream
    id: NodeId, cert: Certificate, is_ingress: bool,
    queue: MessageQueue,                              // outbound queue (§3.3)
    // handshake state — written only by the read task before handshake completes:
    ip: Option<SignedIp>, version: Option<AppVersion>,
    tracked_subnets: HashSet<Id>, supported_acps: HashSet<u32>, objected_acps: HashSet<u32>,
    got_handshake: AtomicBool, finished_handshake: AtomicBool,
    observed_uptime: AtomicU32,
    last_sent: AtomicI64, last_received: AtomicI64, last_ping_sent: AtomicI64,
    txid_of_verified_bls_key: Id,
    close: CancellationToken,
}
```

Three tasks (mirroring Go), all sharing `Arc<Peer>`; the last to exit calls
`Network::disconnected` (Go's `numExecuting` countdown → a `tokio::sync` barrier or an
`Arc` strong-count drop guard):

1. **read task** — `loop { read 4-byte len; read_msg_len; throttler.acquire(len).await;
   read len bytes into Bytes; parse_inbound; dispatch }`. Resets read deadline to
   `now + PongTimeout` around each read (tokio: wrap reads in `tokio::time::timeout`).
   Network-level ops (`Ping/Pong/Handshake/GetPeerList/PeerList`) handled inline;
   everything else, **only after `finished_handshake`**, goes to
   `router.handle_inbound(&token, msg)` (§3.6).
2. **write task** — first writes the `Handshake` message (forced), then loops:
   `try_pop` (non-blocking) → write; when empty, flush, then blocking `pop().await` →
   write. Writes `len || bytes` with vectored I/O. Resets write deadline each message.
3. **net-messages task** — `tokio::select!` over: the `GetPeerList` trigger (debounced
   `mpsc(1)`), a `ping_interval` ticker (`PingFrequency = 22.5s`), and `close` token.
   On the ticker: re-check `AllowConnection` + `should_disconnect`; send `Ping{uptime}`.

`should_disconnect` reproduces the version-compat + BLS-PoP check, caching
`txid_of_verified_bls_key` to avoid re-verifying each tick.

### 3.3 Outbound message queue

Mirror `network/peer/message_queue.go`. Two implementations behind a `MessageQueue`
trait: `Push(msg) -> bool`, `Pop().await -> Option<msg>`, `PopNow() -> Option<msg>`,
`Close()`.

- **`ThrottledMessageQueue`** (default per-peer): an unbounded `VecDeque` guarded by a
  mutex + `Notify` (replacing Go's `sync.Cond`). On `Push`: first `outbound_throttler.acquire`;
  on drop/close/pop: `outbound_throttler.release`. Drops + invokes `on_failed` if the
  throttler refuses or the queue is closed. `bypass_throttling` messages skip acquire.
- **`BlockingMessageQueue`**: a bounded `mpsc` (used in tests / special senders).

> **Note on priorities:** the production queue is FIFO, not multi-priority. The only
> "priority" mechanism is `OutboundMessage::bypass_throttling` (handshake/PeerList
> replies skip the throttler) — reproduce that bit exactly.

### 3.4 Dialer

`ava-network::dialer`. `Dial(ip).await -> TcpStream` with a connection timeout
(`DefaultOutboundConnectionTimeout = 30s`) and a **dial throttler** (token bucket,
`DefaultOutboundConnectionThrottlingRps = 50`). Maps to `tokio::net::TcpStream::connect`
wrapped in `tokio::time::timeout`, gated by a `governor`/`tokio` rate limiter.

### 3.5 Peer-list gossip, IP tracking, IP signing

Reproduce `network/ip_tracker.go` + the gossip cadence from `constants`:
- We advertise validator IPs via **`PeerList`** messages, pulled by **`GetPeerList`**
  carrying a **bloom filter** of already-known peers + a random salt (so we only send
  unknown IPs). `Network::peers(node, tracked, all_subnets, filter, salt)` returns the
  not-yet-known IPs the requester should learn.
- Cadence (all from `networking.go`): push gossip every
  `DefaultNetworkPeerListGossipFreq = 1m`; pull gossip (`GetPeerList`) every
  `DefaultNetworkPeerListPullGossipFreq = 2s`; bloom reset every
  `DefaultNetworkPeerListBloomResetFreq = 1m`; gossip up to
  `DefaultNetworkPeerListNumValidatorIPs = 15` validator IPs;
  validator-gossip-size = 20, peers-gossip-size = 10, non-validator = 0.
- `ManuallyTrack` / `Track` add IPs to `tracked_ips` and keep reconnecting with
  exponential backoff (`DefaultNetworkInitialReconnectDelay = 1s` →
  `DefaultNetworkMaxReconnectDelay = 1m`).
- A `ClaimedIpPort` in a `PeerList` carries the peer's **X.509 cert** + signed IP; on
  receipt we `staking::parse_certificate`, validate, and only connect if the signed IP
  verifies (delegated to the IP-tracker / dial logic).
- **IP signer** (`IpSigner`): caches the `SignedIp` for the current dynamic IP; re-signs
  on IP change (TLS sig over `SHA256(ip_bytes)` + BLS PoP over `ip_bytes`). Uses
  `arc-swap` for the cached signed IP (lock-free read on the hot connect path).

### 3.6 Routing inbound → consensus (the handoff trait)

This is the **cross-spec contract** between `ava-network` and `ava-engine`/`ava-snow`
(`06`). Mirror `snow/networking/router/inbound_handler.go`:

```rust
/// Handles a fully-parsed inbound consensus/app message. Implemented by `06`'s
/// ChainRouter; held by every Peer. THE source of truth for the network→consensus
/// boundary — `06` MUST implement exactly this.
#[async_trait]
pub trait InboundHandler: Send + Sync {
    async fn handle_inbound(&self, ctx: &CancellationToken, msg: InboundMessage);
}

/// Adds peer lifecycle. The Network calls these; `06`'s ChainRouter implements it.
#[async_trait]
pub trait ExternalHandler: InboundHandler {
    fn connected(&self, node_id: NodeId, version: &AppVersion, subnet_id: Id);
    fn disconnected(&self, node_id: NodeId);
}
```

- A `Peer` holds `Arc<dyn InboundHandler>` and calls `handle_inbound` for every
  non-handshake op once `finished_handshake`. The Go `context.Background()` becomes the
  peer/network `CancellationToken`.
- The `Network` calls `ExternalHandler::connected/disconnected` from the peer set
  bookkeeping (Go: `chain_router.go` `Connected`/`Disconnected`). `connected` is invoked
  per **tracked subnet** the peer shares with us (Go iterates subnets).
- The message's `OnFinished` drop-guard releases the inbound throttler permit once the
  router is done with it (the router takes ownership of `InboundMessage`).

### 3.7 Tracked subnets

- A peer advertises `tracked_subnets` (≤16) + `all_subnets` in its Handshake. The
  primary network ID is always implicitly tracked.
- We only `connected`-notify the router for the **intersection** of our tracked subnets
  and the peer's. Primary-network validators (and nodes with `ConnectToAllValidators` /
  `network-require-validator-to-connect` semantics) request all-subnet IPs
  (`requestAllSubnetIPs`).

### 3.8 The app-level SDK (`ava-network::p2p`) — NOT gRPC

`network/p2p/` is a request/response + gossip framework built **on top of**
`AppRequest`/`AppResponse`/`AppError`/`AppGossip` — i.e. it rides the same custom TLS
wire protocol, **not** gRPC. (gRPC/`tonic` is only for the local `rpcchainvm` plugin
protocol — see `07`. The inter-node p2p never uses gRPC.) Design:

- **Handler-prefixed framing** (`router.go::ParseMessage`): an app message body is
  `uvarint(handler_id) || protocol_bytes`. The SDK `Router` parses the uvarint prefix
  and dispatches to the registered `Handler` for that `handler_id`. Reproduce
  `binary.Uvarint` exactly (LEB128 unsigned varint).
- **`Router`** (implements the VM-facing `AppHandler`): `AppRequest`/`AppGossip` →
  look up handler by id → call it; `AppResponse`/`AppError` → resolve the pending
  request callback by `request_id`. **Invariant: the SDK uses odd-numbered request IDs**
  (`requestID = 1` then `+2`) so they don't collide with engine-issued request IDs —
  reproduce this.
- **`Client`** (per `handler_id`): `app_request(node, bytes, callback)`,
  `app_gossip(bytes)`. Sends via the engine's `AppSender` (→ `Network::send`).
- **`p2p/gossip`**: pull/push gossip with a bloom-filter set-reconciliation
  (`Gossiper`, `Set`, `BloomFilter`), and **`acp118`** (warp-signature aggregation).
  These are byte-compatible application protocols; reproduce their framing. Consumed by
  `08`/`09`/`10`/`11` (mempool gossip, warp).

```rust
#[async_trait]
pub trait Handler: Send + Sync {
    async fn app_request(&self, node: NodeId, deadline: Instant, req: &[u8])
        -> Result<Vec<u8>, AppError>;
    async fn app_gossip(&self, node: NodeId, msg: &[u8]);
}
```

---

## 4. TLS layer & custom verification (Rust)

### 4.1 Crate choices (consistent with §4 of `00`)

`rustls` (+ `tokio-rustls`) for TLS, `rcgen` for staking-cert generation, `x509-parser`
for parsing peer leaf certs / extracting the SPKI, `ring` (or `aws-lc-rs`) as the
rustls crypto provider. **Decision against libp2p:** avalanche uses a *custom*
TLS-then-length-prefixed-protobuf protocol with self-signed-cert identity and its own
peer discovery; libp2p's transport/identity/muxing model does not match the wire and
would break interop. libp2p was considered and rejected. `bytes::Bytes` for zero-copy
framing; `zstd` crate for compression (+ `flate2` only for legacy decode tolerance).

### 4.2 TLS config construction

- **Both** the server and client `rustls` configs:
  - Protocol versions: TLS 1.3 only (`&[&rustls::version::TLS13]`).
  - Crypto provider: default (`ring`/`aws-lc-rs`) provider, **unmodified suite list** —
    its TLS1.3 suites (`AES_128_GCM`, `AES_256_GCM`, `CHACHA20_POLY1305`) match Go.
  - Our own staking cert + key as the certificate (server *and* client present a cert →
    mutual TLS).
  - **No** ALPN (Go sets none), **no** SNI verification.
- Server config: `.with_client_cert_verifier(Arc::new(AvaClientCertVerifier))` — requires
  a client cert (`RequireAnyClientCert` equivalent) and runs our leaf-key policy.
- Client config: `.dangerous().with_custom_certificate_verifier(Arc::new(AvaServerCertVerifier))`.

### 4.3 Upgrader

`Upgrader::upgrade(tcp) -> (NodeId, TlsStream, Certificate)`:

```rust
pub async fn upgrade(&self, tcp: TcpStream) -> Result<(NodeId, TlsStream<TcpStream>, Certificate)> {
    let tls = match self.side { Server => self.acceptor.accept(tcp).await?,
                                Client => self.connector.connect(no_verify_name, tcp).await? };
    let leaf = tls.get_ref().1.peer_certificates().and_then(|c| c.first()).ok_or(NoCert)?;
    let cert = staking::parse_certificate(leaf.as_ref())?;   // 03
    let node_id = NodeId::from_cert(&cert);                  // RIPEMD160(SHA256(DER))
    Ok((node_id, tls, cert))
}
```

After upgrade, split the stream into read/write halves (`tokio::io::split`) for the
read/write tasks; the write task uses `write_vectored` for `len || payload`.

### 4.4 Server-side client-cert verifier

```rust
#[derive(Debug)]
struct AvaClientCertVerifier { supported: WebPkiSupportedAlgorithms }

impl rustls::server::danger::ClientCertVerifier for AvaClientCertVerifier {
    fn root_hint_subjects(&self) -> &[DistinguishedName] { &[] }
    fn offer_client_auth(&self) -> bool { true }
    fn client_auth_mandatory(&self) -> bool { true } // == RequireAnyClientCert

    fn verify_client_cert(&self, end_entity: &CertificateDer, _inter: &[CertificateDer],
                          _now: UnixTime) -> Result<ClientCertVerified, rustls::Error> {
        validate_leaf_public_key(end_entity)?;        // == ValidateCertificate
        Ok(ClientCertVerified::assertion())            // no CA chain check (by design)
    }
    fn verify_tls13_signature(&self, m: &[u8], c: &CertificateDer, dss: &DigitallySignedStruct)
        -> Result<HandshakeSignatureValid, rustls::Error> {
        rustls::crypto::verify_tls13_signature(m, c, dss, &self.supported) // real sig check
    }
    fn supported_verify_schemes(&self) -> Vec<SignatureScheme> { self.supported.all.to_vec() }
}
```

`verify_client_cert` skips the chain (no CA) but **`verify_tls13_signature` still runs
the real handshake-signature check** via the crypto provider — proving the peer holds
the private key for the presented leaf. That, plus `validate_leaf_public_key`, is
exactly what Go's `InsecureSkipVerify + VerifyConnection` achieves.

### 4.5 Client-side server-cert verifier

The mirror: `rustls::client::danger::ServerCertVerifier` with `verify_server_cert`
running `validate_leaf_public_key` then `ServerCertVerified::assertion()`, and
`verify_tls13_signature` delegating to the provider.

```rust
fn validate_leaf_public_key(der: &CertificateDer) -> Result<(), rustls::Error> {
    let (_, cert) = x509_parser::parse_x509_certificate(der.as_ref()).map_err(other)?;
    match cert.public_key().parsed().map_err(other)? {
        PublicKey::EC(ec)  => require_p256(&ec),                  // ErrCurveMismatch otherwise
        PublicKey::RSA(rsa) => staking::validate_rsa_well_formed(&rsa), // 03; min modulus 2048
        _ => Err(ErrUnsupportedKeyType.into()),
    }
}
```

---

## 5. Throttling (Rust designs)

All in `ava-network::throttling`, reproducing `network/throttling/*`. Constants from
`utils/constants/networking.go`.

| Throttler | Go | Rust design |
|---|---|---|
| **Dial throttler** | `golang.org/x/time/rate` limiter, `OutboundConnectionThrottlingRps=50` | `governor` (GCRA) or a tokio token bucket; `acquire().await` before dial |
| **Inbound conn-upgrade throttler** | per-IP cooldown `10s`, `MaxConnsPerSec=256` | `DashMap<IpAddr, Instant>` + a global rate limiter; reject (drop TCP) if too soon |
| **Inbound msg byte throttler** | per-node + at-large + validator byte pools; `Acquire(ctx,size,node).await -> ReleaseFunc` | fairness-preserving: a `Semaphore`-like custom allocator with three pools (vdr `32 MiB`, at-large `6 MiB`, node-max `2 MiB`); returns a `ReleasePermit` (RAII, replaces `ReleaseFunc`) |
| **Bandwidth throttler** | per-node token bucket, refill `512 KiB/s`, burst `2 MiB` | `governor` per-node keyed limiter; `acquire(size,node).await` |
| **Inbound msg buffer throttler** | max processing msgs/node `1024` | a per-node `Semaphore(1024)` |
| **Inbound resource (CPU/disk) throttler** | recheck-delay loop on CPU/disk usage | a tracker polled with `5s` recheck delay; pauses acquisition when over budget |
| **Outbound msg byte throttler** | per-node + at-large + vdr pools (`32 MiB`/`32 MiB`/`2 MiB`) | same allocator design as inbound; `acquire(msg,node) -> bool` (non-blocking, drops on refusal) |

Key Rust principle: **every `acquire` returns an RAII permit whose `Drop` releases the
allocation.** This eliminates the Go "must call `ReleaseFunc`/`onFinishedHandling`
exactly once or leak" footgun (§10). The byte throttler must preserve Go's *fairness*
(progress guarantee + per-node single-outstanding-acquire invariant) so a slow node
can't starve others — reproduced with the same per-node `waitingToAcquire` queueing,
keyed by node, woken when bytes free up.

### Inbound byte-throttler acquire (shape)

```rust
/// Returns a permit once `size` bytes are available for `node`. Permit's Drop releases.
async fn acquire(&self, size: u64, node: NodeId, cancel: &CancellationToken) -> ReleasePermit;
```

Pool selection mirrors Go: charge the node's validator allocation first (weighted by
stake), then the at-large pool, capped by `nodeMaxAtLargeBytes`; block if neither has
room, waking on release.

---

## 6. NAT (`ava-network::nat`)

Mirror `nat/`. A `Router` trait + a `PortMapper`:

```rust
pub trait NatRouter: Send + Sync {
    fn supports_nat(&self) -> bool;
    fn map_port(&self, internal: u16, external: u16, desc: &str, duration: Duration) -> Result<()>;
    fn unmap_port(&self, internal: u16, external: u16) -> Result<()>;
    fn external_ip(&self) -> Result<IpAddr>;
}
pub fn get_router() -> Box<dyn NatRouter>; // try UPnP, then NAT-PMP, else NoRouter
```

- **UPnP** + **NAT-PMP/PCP** via the **`igd-next`** crate (UPnP IGD) and a PMP crate
  (`natpmp`/`crab-nat`). `get_router()` probes UPnP first, then PMP, else a no-op router
  (matching `nat.GetRouter`).
- `PortMapper`: a background tokio task that (re)maps the staking port every
  `mapTimeout/… ` (Go: `mapTimeout = 30m`, `maxRefreshRetries = 3`) and unmaps on
  shutdown.

---

## 7. Config knobs (cross-ref `12`)

All `network-*` flags map to a `NetworkConfig` in `ava-config`, defaults from
`utils/constants/networking.go` (each constant carries a doc-comment citing the Go
path per `00` §5). Notable groups:

- **Sizes/timeouts:** `network-max-message-size` (default `2 MiB`),
  `network-peer-read-buffer-size`/`-write-buffer-size` (`8 KiB`),
  `network-ping-timeout` (`30s`), `network-ping-frequency` (`22.5s`),
  `network-read-handshake-timeout` (`15s`), `network-max-clock-difference` (`60s`),
  `network-maximum-inbound-message-timeout` (`10s`).
- **Compression:** `network-compression-type` (default `zstd`).
- **Peer-list gossip:** `network-peer-list-num-validator-ips` (15),
  `…-validator-gossip-size` (20), `…-peers-gossip-size` (10),
  `…-gossip-frequency` (1m), `…-pull-gossip-frequency` (2s).
- **Throttling:** the inbound/outbound at-large/vdr/node byte allocs, bandwidth
  refill/burst, conn-upgrade cooldown + max-conns-per-sec, outbound dial RPS + timeout.
- **TLS/proxy:** staking cert/key paths (`12`), `network-tcp-proxy-enabled` (false),
  `network-tcp-proxy-read-timeout` (3s).
- **Validator gating:** `network-require-validator-to-connect`,
  `network-allow-private-ips`, tracked-subnets list.

---

## 8. Go → Rust mapping (subsystem-specific)

| Go | Rust |
|---|---|
| goroutine-per-peer (×3) + `Peer` struct | task-per-peer (×3 tokio tasks) + `Arc<Peer>` |
| `MessageQueue` (`sync.Cond` deque) | `MessageQueue` (`Mutex<VecDeque>` + `Notify`) / `mpsc` |
| `chan struct{}` triggers (`getPeerListChan`, `onFinishHandshake`, `onClosed`) | `Notify` / `CancellationToken` / `oneshot` |
| `context.Context` cancel (`onClosingCtx`) | `CancellationToken` |
| `numExecuting int64` countdown → `Disconnected` | last-task drop guard / `tokio::sync` join barrier |
| `onFinishedHandling func()` / `ReleaseFunc` | RAII `OnFinished` / `ReleasePermit` (Drop releases) |
| `net.Buffers` `writev` | `AsyncWrite::write_vectored` (or `poll_write_vectored`) |
| `bufio.Reader/Writer` | `BufReader`/`BufWriter` over the split TLS stream |
| `tls.Config{InsecureSkipVerify, VerifyConnection}` | rustls custom `ServerCertVerifier`/`ClientCertVerifier` (§4) |
| `ids.NodeIDFromCert` (RIPEMD160∘SHA256) | `NodeId::from_cert` in `ava-crypto` (`03`) |
| `golang.org/x/time/rate` | `governor` (GCRA) |
| `utils/bloom` (peer-list filter) | `ava-types::bloom` (byte-compatible, see `03`) |
| `proto/pb/p2p` (Go protobuf) | prost-generated `p2p` module (`ava-message`) |
| `InboundHandler`/`ExternalHandler` | `InboundHandler`/`ExternalHandler` traits (§3.6) |

**Error model.** `ava-message::Error` and `ava-network::Error` (`thiserror`), preserving
Go sentinels as variants and asserting via `matches!`/`ErrorIs`-style tests: e.g.
`MaxMessageLengthExceeded`, `InvalidMessageLength`, `UnknownCompressionType`,
`NoCertsSent`, `EmptyCert`, `CurveMismatch`, `UnsupportedKeyType`,
`TimestampTooFarInFuture`, `InvalidTlsSignature`, `ExistingAppProtocol`,
`UnrequestedResponse`, `UnregisteredHandler`. Per `00` §7.1 these are typed, not strings.

---

## 9. Test plan (cross-ref `02`)

1. **Proptest round-trips** — for every `Op`, `build → marshal → frame → read_msg_len →
   unmarshal → unwrap` is identity; arbitrary field values. Compression on/off both
   round-trip (`marshal(zstd) → unmarshal == original`).
2. **Golden framing vectors** — capture real frames (`len_be || proto_bytes`) emitted by
   a Go node for each op (Handshake, Ping, PeerList, Get, Put, Chits, AppRequest, …) and
   assert our encoder reproduces **byte-identical** `proto_bytes` (and that we decode the
   Go frame). Stored under `tests/golden/p2p/`. This guards the §1.1 byte-exactness note.
   (zstd frames: assert cross-**decodability**, not byte-equality — §1.3.)
3. **NodeID golden** — known staking cert DER → known NodeID string (cross-ref `03`),
   proving `RIPEMD160(SHA256(DER))` parity.
4. **Signed-IP golden** — `UnsignedIP::bytes()` layout (`16 || port_be || ts_be`) and
   TLS/BLS signature verification against Go-produced signatures.
5. **Mock-peer handshake test** — two in-process `Peer` actors over a duplex stream
   (or `tokio::io::duplex`) complete the Handshake↔PeerList↔finished sequence; assert
   `connected` fires, ping/pong RTT recorded, disconnect on networkID/version/clock-skew
   mismatch and on `supported ∩ objected` ACPs.
6. **TLS interop unit test** — a rustls server config + rustls client config handshake
   succeeds with self-signed P-256 staking certs; rejects non-P-256 ECDSA, small-modulus
   RSA, and unsupported key types (mirrors `tls_config_test.go`).
7. **Throttler tests** — port `inbound_msg_byte_throttler_test.go` /
   `outbound_msg_throttler_test.go` semantics: fairness, progress guarantee, per-node
   single outstanding acquire, release-on-drop, no leak.
8. **SDK router test** — uvarint handler-prefix parse, odd request-id invariant,
   pending-request resolution, `UnregisteredHandler` AppError path.
9. **Differential interop vs a real Go node** (`02` harness) — a Rust node dials a Go
   node on a local network: TLS handshake + p2p Handshake complete, PeerList gossip
   exchanged, ping/pong observed-uptime converges, and the Go node reports the Rust peer
   as connected & at expected uptime. The acceptance bar for this spec.

---

## 10. Performance notes / improvements over Go

All "safe because…" justifications tie to `00` §6.1 / §9 (no change to observable
wire/consensus behavior; only internal mechanics differ).

- **Zero-copy read path.** Read message bytes straight into a `BytesMut`, freeze to
  `Bytes`, and let prost decode borrowing from it; the `InboundMessage` owns the buffer.
  Avoids Go's `make([]byte, msgLen)` per message copy into the parser. *Safe:* decoded
  values are identical.
- **Vectored writes + batched drains.** The write task drains all currently-queued
  messages and issues a single `write_vectored` of interleaved `[len, payload, len,
  payload, …]` slices, then flushes — fewer syscalls than Go's per-message
  `io.CopyN`. *Safe:* the byte stream is identical (concatenation is associative);
  framing unchanged.
- **RAII permits eliminate throttler/handling leaks.** `OnFinished` and `ReleasePermit`
  release on `Drop`, removing Go's "call exactly once or leak" invariant entirely.
  *Safe:* purely an internal accounting fix; throttle limits unchanged.
- **Lock-free peer set.** `DashMap`/`arc-swap` for `connected_peers` and the cached
  `SignedIp` → contention-free reads on the send hot path and the connect path, vs Go's
  `peersLock sync.RWMutex`. *Safe:* observable peer membership/ordering is unchanged;
  send fan-out still iterates the same set.
- **Per-task buffer reuse.** Each peer's read/write task keeps a reusable `BytesMut`
  scratch (mirrors Go's `bufio` buffers + the spirit of `sync.Pool`) — avoids
  per-message allocation. *Safe:* no wire effect.
- **Batched/parallel signature verification (bounded).** BLS PoP and TLS-IP signature
  checks on `PeerList` ingestion can be verified on a `spawn_blocking`/rayon pool in
  parallel, since each `ClaimedIpPort` is independent and the *set* of accepted IPs is
  order-independent. *Safe:* the accepted IP set is identical regardless of verification
  order (no ordering-dependent side effects); cross-ref `03` BLS batch verify.

> Any improvement above must be backed by the differential interop test (§9.9) proving
> a Go node cannot distinguish a Rust peer.
