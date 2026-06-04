# 15 — Serialization & Wire Formats (the authoritative catalog)

> **Status:** Canonical, byte-exact serialization catalog for the Rust port. This
> file is the single index of **every** serialization format and wire protocol in
> `avalanchego`: all protobuf/gRPC packages, the avalanche linear codec, the p2p
> framing + compression, EVM RLP, and the address/string encodings. It conforms to
> `00-overview-and-conventions.md` (§1 compatibility surface, §4.2 serialization
> deps). It **references** the detailed treatments in other specs rather than
> duplicating them — `03` (linear codec / Packer, CB58/bech32), `05` (p2p framing
> & zstd), `07` (rpcchainvm proxied services), `10` (EVM RLP), `12` (genesis,
> JSON-RPC, Connect routing), `14` (JSON-RPC).
>
> Field numbers, type IDs, and codec version bytes recorded here are **protocol
> constants**: they MUST match Go for interop. Treat any divergence as a bug.

Go source: `proto/**/*.proto`, `connectproto/**/*.proto`, `proto/buf.yaml`,
`proto/buf.gen.yaml`, `connectproto/buf.{yaml,gen.yaml}`, plus the non-proto
formats covered in the cross-referenced specs.

---

## 1. Master format table

| Format | Where used | Library (Rust) | Byte-exact required? | Spec ref |
|---|---|---|---|---|
| **Avalanche linear codec** (BE, length-prefixed, 2-byte version, `u32` typeID interface registry) | All P/X tx & block bytes; genesis bytes; warp **UnsignedMessage** payloads & addressed-call/justification bodies; C-Chain **atomic** tx bytes; state-sync proof leaf serialization; the signed-IP message; `signer`/`bls` PoP message bodies | hand-written `ava-codec` (+ `ava-codec-derive`) — **never** bincode/borsh | **YES** | `03` §2 |
| **p2p framing** (4-byte BE length prefix + proto3 `p2p.Message`; 2 MiB cap) | Inter-node TLS stream | `ava-network` framing helpers + `prost` | **YES** for framing & length rules | `05` §1.1 |
| **zstd recursive packing** (`compressed_zstd` = self-nested `Message`) | Compressed p2p messages | `zstd` crate (default level, 2 MiB window) | **DECODE-compatible only** (R4) | `05` §1.3 |
| **protobuf / gRPC** (proto3) | All `proto/` + `connectproto/` services (§3) | `prost` + `tonic` (+ `tonic-build`) | **canonical-proto-equivalent** (see §6) | `07`, `12`, this file §3 |
| **EVM RLP** (blocks, txs, receipts, headers) | C-Chain & EVM subnets | `reth`/`alloy` (`alloy-rlp`) | **YES** (hashes must match) | `10` |
| **CB58** (base58 + 4-byte sha256-tail checksum) | `ids.ID`/`ShortID`/`NodeID` string form, CB58 tx/key encodings | hand-rolled on `bs58` | **YES** | `03` §3.2 |
| **bech32** chain-prefixed addresses (`<chain>-<bech32(hrp,addr)>`) | P/X addresses; hrp per network | `bech32` crate | **YES** | `03` §3.3 |
| **hex `0x`** (+ optional 4-byte checksum) | EVM addresses/hashes; `formatting.Encode` Hex/HexC/HexNC | `hex` / alloy | **YES** | `03` §3.2, `10` |
| **`NodeID-<CB58>`** | NodeID display/parse | `ava-types` | **YES** | `03` §1 |
| **JSON-RPC 2.0** | HTTP APIs (info, platform, avm, C-Chain RPC) | `serde_json` + axum JSON-RPC shim | structurally exact | `14`, `12` |
| **Connect protocol** (gRPC/gRPC-Web/Connect over h2c) | `connectproto/` services (proposervm, xsvm) | `tonic` + Connect shim | proto-equivalent | `12` (R5), this file §3.15–3.16 |
| **Genesis JSON** → produced genesis bytes (linear codec) | Network genesis | `serde_json` (JSON) + `ava-codec` (bytes) | **YES** (genesis block IDs) | `12` |

The single byte-exactness carve-out is zstd (R4): the differential harness asserts
a Go node *accepts* our frames and that we decode theirs — not that compressed
bytes are identical. Decode is fully deterministic.

---

## 2. Buf / codegen toolchain (the proto build)

Two buf roots, identical lint/breaking config, different codegen:

| Root | `buf.yaml` | `buf.gen.yaml` plugins | Notes |
|---|---|---|---|
| `proto/` | name `buf.build/ava-labs/avalanche`; dep `buf.build/prometheus/client-model`; **excludes `io/prometheus`** from generate (Go pulls prometheus as a buf dep; the `.proto` is kept for non-Go langs like **Rust**) | `go` (→ `pb`, `paths=source_relative`), `go-grpc` | breaking: `FILE`; lint: `STANDARD` minus `SERVICE_SUFFIX`, `RPC_REQUEST_STANDARD_NAME`, `RPC_RESPONSE_STANDARD_NAME`, `PACKAGE_VERSION_SUFFIX`; ignores `aliasreader/aliasreader.proto` & `net/conn/conn.proto`; allows empty req/resp and same req==resp |
| `connectproto/` | same name/deps/excludes | `go`, **`connect-go`** | same lint/breaking; no per-file ignores |

Lint flags that matter for the Rust port: `rpc_allow_google_protobuf_empty_requests`/
`_responses` and `rpc_allow_same_request_response` are **true** — many RPCs use
`google.protobuf.Empty` and several reuse a single message as both request and
response (e.g. `aliasreader`, `net/conn`).

**Rust codegen (per `00` §4.2 / decision 8):** `prost` + `tonic` via `build.rs`
(`tonic-build`), **not committed**; Bazel uses `rust_prost_library`. The two roots
become two generated module trees: `proto/` → core/plugin gRPC, `connectproto/` →
Connect services (served via tonic + a Connect-protocol compat layer, `12`/R5).
The excluded `io/prometheus/client/metrics.proto` (referenced by `vm.proto`'s
`Gather`) **must** be vendored/compiled on the Rust side since we don't have Go's
buf-dep shortcut — pull `prometheus-client-model`-equivalent protos into the
build.

---

## 3. Protobuf packages — services & messages (verbatim)

**Inventory: 20 `.proto` files, 18 distinct packages, 17 gRPC/Connect services.**
Packages: `p2p`, `vm`, `vm.runtime`, `rpcdb`, `appsender`, `sharedmemory`,
`validatorstate`, `warp`, `signer`, `sync`, `http`, `http.responsewriter`,
`io.reader`, `io.writer`, `net.conn`, `aliasreader`, `platformvm`, `sdk` (no
service), and Connect: `proposervm`, `xsvm`.

All files are `proto3`. Field numbers below are load-bearing.

### 3.1 `p2p` — `proto/p2p/p2p.proto` (THE inter-node wire schema)

No service (carried over the raw framed TLS stream, `05`). The root is one big
`oneof`. `reserved 1` (until E upgrade), `reserved 37` (next unused).

**`Message.message` oneof** — tag → type:

| Field | Tag | Type | Class |
|---|---|---|---|
| `compressed_zstd` | 2 | `bytes` | zstd-wrapped inner `Message` (§1, `05` §1.3); tags <10 reserved for other algos |
| `ping` | 11 | `Ping` | Network |
| `pong` | 12 | `Pong` | Network |
| `handshake` | 13 | `Handshake` | Network |
| `get_peer_list` | 35 | `GetPeerList` | Network |
| `peer_list` | 14 | `PeerList` | Network |
| `get_state_summary_frontier` | 15 | `GetStateSummaryFrontier` | State-sync |
| `state_summary_frontier` | 16 | `StateSummaryFrontier` | State-sync |
| `get_accepted_state_summary` | 17 | `GetAcceptedStateSummary` | State-sync |
| `accepted_state_summary` | 18 | `AcceptedStateSummary` | State-sync |
| `get_accepted_frontier` | 19 | `GetAcceptedFrontier` | Bootstrap |
| `accepted_frontier` | 20 | `AcceptedFrontier` | Bootstrap |
| `get_accepted` | 21 | `GetAccepted` | Bootstrap |
| `accepted` | 22 | `Accepted` | Bootstrap |
| `get_ancestors` | 23 | `GetAncestors` | Bootstrap |
| `ancestors` | 24 | `Ancestors` | Bootstrap |
| `get` | 25 | `Get` | Consensus |
| `put` | 26 | `Put` | Consensus |
| `push_query` | 27 | `PushQuery` | Consensus |
| `pull_query` | 28 | `PullQuery` | Consensus |
| `chits` | 29 | `Chits` | Consensus |
| `app_request` | 30 | `AppRequest` | App |
| `app_response` | 31 | `AppResponse` | App |
| `app_gossip` | 32 | `AppGossip` | App |
| `app_error` | 34 | `AppError` | App |
| `simplex` | 36 | `Simplex` | Simplex |

**Messages (field name : tag : type):**

- `Ping`: `uptime`:1:`uint32` (primary-network %, [0,100]); `reserved 2` (until Etna).
- `Pong`: `reserved 1,2` (until E upgrade) — empty on the wire now.
- `Handshake`: `network_id`:1:`uint32`; `my_time`:2:`uint64`; `ip_addr`:3:`bytes`; `ip_port`:4:`uint32`; `upgrade_time`:5:`uint64`; `ip_signing_time`:6:`uint64`; `ip_node_id_sig`:7:`bytes`; `tracked_subnets`:8:`repeated bytes`; `client`:9:`Client`; `supported_acps`:10:`repeated uint32`; `objected_acps`:11:`repeated uint32`; `known_peers`:12:`BloomFilter`; `ip_bls_sig`:13:`bytes`; `all_subnets`:14:`bool`.
- `Client`: `name`:1:`string`; `major`:2:`uint32`; `minor`:3:`uint32`; `patch`:4:`uint32`.
- `BloomFilter`: `filter`:1:`bytes`; `salt`:2:`bytes`.
- `ClaimedIpPort`: `x509_certificate`:1:`bytes`; `ip_addr`:2:`bytes`; `ip_port`:3:`uint32`; `timestamp`:4:`uint64`; `signature`:5:`bytes`; `tx_id`:6:`bytes`.
- `GetPeerList`: `known_peers`:1:`BloomFilter`; `all_subnets`:2:`bool`.
- `PeerList`: `claimed_ip_ports`:1:`repeated ClaimedIpPort`.
- `GetStateSummaryFrontier`: `chain_id`:1:`bytes`; `request_id`:2:`uint32`; `deadline`:3:`uint64` (ns).
- `StateSummaryFrontier`: `chain_id`:1; `request_id`:2; `summary`:3:`bytes`.
- `GetAcceptedStateSummary`: `chain_id`:1; `request_id`:2; `deadline`:3; `heights`:4:`repeated uint64`.
- `AcceptedStateSummary`: `chain_id`:1; `request_id`:2; `summary_ids`:3:`repeated bytes`.
- `GetAcceptedFrontier`: `reserved 4`; `chain_id`:1; `request_id`:2; `deadline`:3.
- `AcceptedFrontier`: `chain_id`:1; `request_id`:2; `container_id`:3:`bytes`.
- `GetAccepted`: `reserved 5`; `chain_id`:1; `request_id`:2; `deadline`:3; `container_ids`:4:`repeated bytes`.
- `Accepted`: `chain_id`:1; `request_id`:2; `container_ids`:3:`repeated bytes`.
- `GetAncestors`: `chain_id`:1; `request_id`:2; `deadline`:3; `container_id`:4:`bytes`; `engine_type`:5:`EngineType`.
- `Ancestors`: `chain_id`:1; `request_id`:2; `containers`:3:`repeated bytes`.
- `Get`: `reserved 5`; `chain_id`:1; `request_id`:2; `deadline`:3; `container_id`:4:`bytes`.
- `Put`: `chain_id`:1; `request_id`:2; `container`:3:`bytes`.
- `PushQuery`: `reserved 5`; `chain_id`:1; `request_id`:2; `deadline`:3; `container`:4:`bytes`; `requested_height`:6:`uint64`.
- `PullQuery`: `reserved 5`; `chain_id`:1; `request_id`:2; `deadline`:3; `container_id`:4:`bytes`; `requested_height`:6:`uint64`.
- `Chits`: `chain_id`:1; `request_id`:2; `preferred_id`:3:`bytes`; `accepted_id`:4:`bytes`; `preferred_id_at_height`:5:`bytes`; `accepted_height`:6:`uint64`.
- `AppRequest`: `chain_id`:1; `request_id`:2; `deadline`:3; `app_bytes`:4:`bytes`.
- `AppResponse`: `chain_id`:1; `request_id`:2; `app_bytes`:3:`bytes`.
- `AppError`: `chain_id`:1; `request_id`:2; `error_code`:3:`sint32` (VMs may define >0); `error_message`:4:`string`.
- `AppGossip`: `chain_id`:1; `app_bytes`:2:`bytes`.
- `Simplex`: `chain_id`:1:`bytes`; oneof `message`: `block_proposal`:2:`BlockProposal`; `vote`:3:`Vote`; `empty_vote`:4:`EmptyVote`; `finalize_vote`:5:`Vote`; `notarization`:6:`QuorumCertificate`; `empty_notarization`:7:`EmptyNotarization`; `finalization`:8:`QuorumCertificate`; `replication_request`:9:`ReplicationRequest`; `replication_response`:10:`ReplicationResponse`.
- `BlockProposal`: `block`:1:`bytes`; `vote`:2:`Vote`.
- `ProtocolMetadata`: `version`:1:`uint32`; `epoch`:2:`uint64`; `round`:3:`uint64`; `seq`:4:`uint64`; `prev`:5:`bytes`.
- `EmptyVoteMetadata`: `epoch`:1:`uint64`; `round`:2:`uint64`.
- `BlockHeader`: `metadata`:1:`ProtocolMetadata`; `digest`:2:`bytes`.
- `Signature`: `signer`:1:`bytes`; `value`:2:`bytes`.
- `Vote`: `block_header`:1:`BlockHeader`; `signature`:2:`Signature`.
- `EmptyVote`: `metadata`:1:`EmptyVoteMetadata`; `signature`:2:`Signature`.
- `QuorumCertificate`: `block_header`:1:`BlockHeader`; `quorum_certificate`:2:`bytes`.
- `EmptyNotarization`: `metadata`:1:`EmptyVoteMetadata`; `quorum_certificate`:2:`bytes`.
- `ReplicationRequest`: `seqs`:1:`repeated uint64`; `latest_round`:2:`uint64`.
- `ReplicationResponse`: `data`:1:`repeated QuorumRound`; `latest_round`:2:`QuorumRound`.
- `QuorumRound`: `block`:1:`bytes`; `notarization`:2:`QuorumCertificate`; `empty_notarization`:3:`EmptyNotarization`; `finalization`:4:`QuorumCertificate`.

**Enum** `EngineType`: `ENGINE_TYPE_UNSPECIFIED`=0; `ENGINE_TYPE_DAG`=1 (X-Chain only); `ENGINE_TYPE_CHAIN`=2.

> Note: the `bytes` fields named `summary`, `container`, `app_bytes`, `summary_ids`,
> etc. carry **linear-codec or VM-defined** payloads — opaque to proto, byte-exact
> per their owning subsystem.

### 3.2 `vm` — `proto/vm/vm.proto` (rpcchainvm plugin protocol, v45)

Protocol version **`RPCChainVMProtocol = 45`** (`version/constants.go`). `service VM`
(all unary). The host (node) is the gRPC **client**; the plugin is the **server**.
Imports `google/protobuf/{duration,empty,timestamp}.proto` and
`io/prometheus/client/metrics.proto`.

**`service VM` RPCs (request → response):**

| RPC | Request | Response | Group |
|---|---|---|---|
| `Initialize` | `InitializeRequest` | `InitializeResponse` | ChainVM |
| `SetState` | `SetStateRequest` | `SetStateResponse` | |
| `Shutdown` | `Empty` | `Empty` | |
| `CreateHandlers` | `Empty` | `CreateHandlersResponse` | |
| `NewHTTPHandler` | `Empty` | `NewHTTPHandlerResponse` | |
| `WaitForEvent` | `Empty` | `WaitForEventResponse` | |
| `Connected` | `ConnectedRequest` | `Empty` | |
| `Disconnected` | `DisconnectedRequest` | `Empty` | |
| `BuildBlock` | `BuildBlockRequest` | `BuildBlockResponse` | |
| `ParseBlock` | `ParseBlockRequest` | `ParseBlockResponse` | |
| `GetBlock` | `GetBlockRequest` | `GetBlockResponse` | |
| `SetPreference` | `SetPreferenceRequest` | `Empty` | |
| `Health` | `Empty` | `HealthResponse` | |
| `Version` | `Empty` | `VersionResponse` | |
| `AppRequest` | `AppRequestMsg` | `Empty` | |
| `AppRequestFailed` | `AppRequestFailedMsg` | `Empty` | |
| `AppResponse` | `AppResponseMsg` | `Empty` | |
| `AppGossip` | `AppGossipMsg` | `Empty` | |
| `Gather` | `Empty` | `GatherResponse` | |
| `GetAncestors` | `GetAncestorsRequest` | `GetAncestorsResponse` | BatchedChainVM |
| `BatchedParseBlock` | `BatchedParseBlockRequest` | `BatchedParseBlockResponse` | |
| `GetBlockIDAtHeight` | `GetBlockIDAtHeightRequest` | `GetBlockIDAtHeightResponse` | HeightIndexed |
| `StateSyncEnabled` | `Empty` | `StateSyncEnabledResponse` | StateSyncableVM |
| `GetOngoingSyncStateSummary` | `Empty` | `GetOngoingSyncStateSummaryResponse` | |
| `GetLastStateSummary` | `Empty` | `GetLastStateSummaryResponse` | |
| `ParseStateSummary` | `ParseStateSummaryRequest` | `ParseStateSummaryResponse` | |
| `GetStateSummary` | `GetStateSummaryRequest` | `GetStateSummaryResponse` | |
| `BlockVerify` | `BlockVerifyRequest` | `BlockVerifyResponse` | Block |
| `BlockAccept` | `BlockAcceptRequest` | `Empty` | |
| `BlockReject` | `BlockRejectRequest` | `Empty` | |
| `StateSummaryAccept` | `StateSummaryAcceptRequest` | `StateSummaryAcceptResponse` | StateSummary |

**Enums:** `State`{`STATE_UNSPECIFIED`=0,`STATE_STATE_SYNCING`=1,`STATE_BOOTSTRAPPING`=2,`STATE_NORMAL_OP`=3}; `Error`{`ERROR_UNSPECIFIED`=0,`ERROR_CLOSED`=1,`ERROR_NOT_FOUND`=2,`ERROR_STATE_SYNC_NOT_IMPLEMENTED`=3}; `Message`{`MESSAGE_UNSPECIFIED`=0,`MESSAGE_BUILD_BLOCK`=1,`MESSAGE_STATE_SYNC_FINISHED`=2}; nested `StateSummaryAcceptResponse.Mode`{`MODE_UNSPECIFIED`=0,`MODE_SKIPPED`=1,`MODE_STATIC`=2,`MODE_DYNAMIC`=3}.

**Messages (key ones; field : tag : type):**
- `InitializeRequest`: `network_id`:1:`uint32`; `subnet_id`:2; `chain_id`:3; `node_id`:4; `public_key`:5 (BLS); `x_chain_id`:6; `c_chain_id`:7; `avax_asset_id`:8; `chain_data_dir`:9:`string`; `genesis_bytes`:10; `upgrade_bytes`:11; `config_bytes`:12; `db_server_addr`:13:`string`; `server_addr`:14:`string` (callback bundle); `network_upgrades`:15:`NetworkUpgrades`. (bytes unless noted.)
- `NetworkUpgrades`: `apricot_phase_1_time`:1 … (all `google.protobuf.Timestamp` except) `apricot_phase_4_min_p_chain_height`:5:`uint64`; `cortina_x_chain_stop_vertex_id`:12:`bytes`; `granite_epoch_duration`:17:`google.protobuf.Duration`; through `helicon_time`:18. Full list: apricot_phase_1..post_6 (1–9), banff(10), cortina(11)+stopvertex(12), durango(13), etna(14), fortuna(15), granite(16)+epoch_duration(17), helicon(18).
- `InitializeResponse`/`SetStateResponse`: `last_accepted_id`:1; `last_accepted_parent_id`:2; `height`:3:`uint64`; `bytes`:4; `timestamp`:5:`Timestamp`.
- `SetStateRequest`: `state`:1:`State`.
- `CreateHandlersResponse`: `handlers`:1:`repeated Handler`. `Handler`: `prefix`:1:`string`; `server_addr`:2:`string`.
- `NewHTTPHandlerResponse`: `server_addr`:1:`string`.
- `WaitForEventResponse`: `message`:1:`Message`.
- `BuildBlockRequest`: `p_chain_height`:1:`optional uint64`.
- `BuildBlockResponse`: `id`:1; `parent_id`:2; `bytes`:3; `height`:4:`uint64`; `timestamp`:5:`Timestamp`; `verify_with_context`:6:`bool`.
- `ParseBlockRequest`: `bytes`:1. `ParseBlockResponse`: `id`:1; `parent_id`:2; `height`:4; `timestamp`:5; `verify_with_context`:6 (note: **no field 3**).
- `GetBlockRequest`: `id`:1. `GetBlockResponse`: `parent_id`:1; `bytes`:2; `height`:4; `timestamp`:5; `err`:6:`Error`; `verify_with_context`:7.
- `SetPreferenceRequest`: `id`:1.
- `BlockVerifyRequest`: `bytes`:1; `p_chain_height`:2:`optional uint64`. `BlockVerifyResponse`: `timestamp`:1.
- `BlockAcceptRequest`/`BlockRejectRequest`: `id`:1.
- `HealthResponse`: `details`:1:`bytes`. `VersionResponse`: `version`:1:`string`.
- `AppRequestMsg`: `node_id`:1; `request_id`:2:`uint32`; `deadline`:3:`Timestamp`; `request`:4:`bytes`.
- `AppRequestFailedMsg`: `node_id`:1; `request_id`:2; `error_code`:3:`sint32`; `error_message`:4:`string`.
- `AppResponseMsg`: `node_id`:1; `request_id`:2; `response`:3:`bytes`.
- `AppGossipMsg`: `node_id`:1; `msg`:2:`bytes`.
- `ConnectedRequest`: `node_id`:1; `name`:2:`string`; `major`:3; `minor`:4; `patch`:5 (`uint32`). `DisconnectedRequest`: `node_id`:1.
- `GetAncestorsRequest`: `blk_id`:1; `max_blocks_num`:2:`int32`; `max_blocks_size`:3:`int32`; `max_blocks_retrival_time`:4:`int64`. `GetAncestorsResponse`: `blks_bytes`:1:`repeated bytes`.
- `BatchedParseBlockRequest`: `request`:1:`repeated bytes`. `BatchedParseBlockResponse`: `response`:1:`repeated ParseBlockResponse`.
- `GetBlockIDAtHeightRequest`: `height`:1:`uint64`. `GetBlockIDAtHeightResponse`: `blk_id`:1; `err`:2.
- `GatherResponse`: `metric_families`:1:`repeated io.prometheus.client.MetricFamily`.
- `StateSyncEnabledResponse`: `enabled`:1:`bool`; `err`:2. `GetOngoingSyncStateSummaryResponse`/`GetLastStateSummaryResponse`: `id`:1; `height`:2:`uint64`; `bytes`:3; `err`:4. `ParseStateSummaryRequest`: `bytes`:1. `ParseStateSummaryResponse`: `id`:1; `height`:2; `err`:3. `GetStateSummaryRequest`: `height`:1. `GetStateSummaryResponse`: `id`:1; `bytes`:2; `err`:3. `StateSummaryAcceptRequest`: `bytes`:1. `StateSummaryAcceptResponse`: `mode`:1:`Mode`; `err`:2.

### 3.3 `vm.runtime` — `proto/vm/runtime/runtime.proto` (reverse-dial handshake)

`go_package = .../proto/pb/vm/manager`. `service Runtime`: `Initialize(InitializeRequest) → google.protobuf.Empty`. The **host serves** `Runtime`; the plugin dials it back (env `AVALANCHE_VM_RUNTIME_ENGINE_ADDR`, `07` §). `InitializeRequest`: `protocol_version`:1:`uint32` (must equal 45); `addr`:2:`string` (plugin's ephemeral gRPC listener, e.g. `127.0.0.1:50001`). This is the v45 reverse-dial entry point — **not** HashiCorp go-plugin (`07`/decision 1).

### 3.4 `rpcdb` — `proto/rpcdb/rpcdb.proto` (Database over gRPC, `04`)

`service Database` (proxied to the plugin; node serves it):
`Has(HasRequest)→HasResponse`, `Get(GetRequest)→GetResponse`, `Put(PutRequest)→PutResponse`, `Delete(DeleteRequest)→DeleteResponse`, `Compact(CompactRequest)→CompactResponse`, `Close(CloseRequest)→CloseResponse`, `HealthCheck(Empty)→HealthCheckResponse`, `WriteBatch(WriteBatchRequest)→WriteBatchResponse`, `NewIteratorWithStartAndPrefix(...Request)→(...Response)`, `IteratorNext(IteratorNextRequest)→IteratorNextResponse`, `IteratorError(IteratorErrorRequest)→IteratorErrorResponse`, `IteratorRelease(IteratorReleaseRequest)→IteratorReleaseResponse`.
Enum `Error`{`UNSPECIFIED`=0,`CLOSED`=1,`NOT_FOUND`=2} carries `database.ErrClosed`/`ErrNotFound` over the wire (the only two sentinels, `00`/decision 3).
Messages: `HasRequest{key:1}`/`HasResponse{has:1:bool,err:2}`; `GetRequest{key:1}`/`GetResponse{value:1,err:2}`; `PutRequest{key:1,value:2}`/`PutResponse{err:1}`; `DeleteRequest{key:1}`/`DeleteResponse{err:1}`; `CompactRequest{start:1,limit:2}`/`CompactResponse{err:1}`; `CloseRequest{}`/`CloseResponse{err:1}`; `WriteBatchRequest{puts:1:repeated PutRequest,deletes:2:repeated DeleteRequest}`/`WriteBatchResponse{err:1}`; `NewIteratorRequest{}` (unused); `NewIteratorWithStartAndPrefixRequest{start:1,prefix:2}`/`...Response{id:1:uint64}`; `IteratorNextRequest{id:1}`/`IteratorNextResponse{data:1:repeated PutRequest}`; `IteratorErrorRequest{id:1}`/`IteratorErrorResponse{err:1}`; `IteratorReleaseRequest{id:1}`/`IteratorReleaseResponse{err:1}`; `HealthCheckResponse{details:1:bytes}`.

### 3.5 `appsender` — `proto/appsender/appsender.proto`

`service AppSender` (node serves; plugin dials): `SendAppRequest(SendAppRequestMsg)→Empty`, `SendAppResponse(SendAppResponseMsg)→Empty`, `SendAppError(SendAppErrorMsg)→Empty`, `SendAppGossip(SendAppGossipMsg)→Empty`.
`SendAppRequestMsg{node_ids:1:repeated bytes, request_id:2:uint32, request:3:bytes}`; `SendAppResponseMsg{node_id:1, request_id:2, response:3}`; `SendAppErrorMsg{node_id:1, request_id:2, error_code:3:sint32, error_message:4:string}`; `SendAppGossipMsg{node_ids:1:repeated bytes, validators:2:uint64, non_validators:3:uint64, peers:4:uint64, msg:5:bytes}`.

### 3.6 `sharedmemory` — `proto/sharedmemory/sharedmemory.proto`

`service SharedMemory` (node serves): `Get(GetRequest)→GetResponse`, `Indexed(IndexedRequest)→IndexedResponse`, `Apply(ApplyRequest)→ApplyResponse`.
`BatchPut{key:1,value:2}`; `BatchDelete{key:1}`; `Batch{puts:1:repeated BatchPut, deletes:2:repeated BatchDelete}`; `AtomicRequest{remove_requests:1:repeated bytes, put_requests:2:repeated Element, peer_chain_id:3:bytes}`; `Element{key:1, value:2, traits:3:repeated bytes}`; `GetRequest{peer_chain_id:1, keys:2:repeated bytes}`/`GetResponse{values:1:repeated bytes}`; `IndexedRequest{peer_chain_id:1, traits:2:repeated bytes, start_trait:3, start_key:4, limit:5:int32}`/`IndexedResponse{values:1:repeated bytes, last_trait:2, last_key:3}`; `ApplyRequest{requests:1:repeated AtomicRequest, batches:2:repeated Batch}`/`ApplyResponse{}`.
> The `Element.value`/`key`/`traits` are the **atomic UTXO** bytes (linear codec, ATOMIC-1 in `00`/decision 7) — opaque to proto.

### 3.7 `validatorstate` — `proto/validatorstate/validator_state.proto`

`service ValidatorState` (node serves): `GetMinimumHeight(Empty)→GetMinimumHeightResponse`, `GetCurrentHeight(Empty)→GetCurrentHeightResponse`, `GetSubnetID(GetSubnetIDRequest)→GetSubnetIDResponse`, `GetWarpValidatorSets(GetWarpValidatorSetsRequest)→GetWarpValidatorSetsResponse`, `GetValidatorSet(GetValidatorSetRequest)→GetValidatorSetResponse`, `GetCurrentValidatorSet(GetCurrentValidatorSetRequest)→GetCurrentValidatorSetResponse`.
`GetMinimumHeightResponse{height:1}`/`GetCurrentHeightResponse{height:1}` (uint64); `GetSubnetIDRequest{chain_id:1}`/`GetSubnetIDResponse{subnet_id:1}`; `GetWarpValidatorSetsRequest{height:1}`; `GetValidatorSetRequest{height:1, subnet_id:2}`; `GetCurrentValidatorSetRequest{subnet_id:1}`; `Validator{node_id:1, weight:2:uint64, public_key:3:bytes(uncompressed,maybe empty), start_time:4:uint64, min_nonce:5:uint64, is_active:6:bool, validation_id:7:bytes, is_l1_validator:8:bool}`; `GetWarpValidatorSetsResponse{validator_sets:1:repeated WarpValidatorSet}`; `WarpValidatorSet{subnet_id:1, total_weight:2:uint64, validators:3:repeated WarpValidator}`; `WarpValidator{public_key:1:bytes(uncompressed,nonempty), weight:2:uint64, node_ids:3:repeated bytes(nonempty)}`; `GetValidatorSetResponse{validators:1:repeated Validator}`; `GetCurrentValidatorSetResponse{validators:1:repeated Validator, current_height:2:uint64}`.

### 3.8 `warp` — `proto/warp/message.proto`

> **Naming caveat:** despite the path `warp/message.proto`, this file defines a
> gRPC **`warp.Signer`** service, **not** the warp message wire format. The warp
> message format (`UnsignedMessage`, `BitSetSignature`, `AddressedCall`, etc.) is
> **not protobuf** — it is the **avalanche linear codec** (`vms/platformvm/warp`,
> see `03` §2 / `08`). Treat the warp message format as a linear-codec artifact
> (§4), byte-exact, with `bytes payload` here carrying that codec output.
`service Signer`: `Sign(SignRequest)→SignResponse`. `SignRequest{network_id:1:uint32, source_chain_id:2:bytes, payload:3:bytes}`; `SignResponse{signature:1:bytes}` (the BLS signature over the constructed `UnsignedMessage`).

### 3.9 `signer` — `proto/signer/signer.proto`

`service Signer` (BLS local-signer proxy): `PublicKey(PublicKeyRequest)→PublicKeyResponse`, `Sign(SignRequest)→SignResponse`, `SignProofOfPossession(SignProofOfPossessionRequest)→SignProofOfPossessionResponse`.
`PublicKeyRequest{}`/`PublicKeyResponse{public_key:1:bytes}`; `SignRequest{message:1:bytes}`/`SignResponse{signature:1:bytes}`; `SignProofOfPossessionRequest{message:1:bytes}`/`SignProofOfPossessionResponse{signature:1:bytes}`. (Distinct package from `warp.Signer`.)

### 3.10 `sync` — `proto/sync/sync.proto` (merkledb state-sync proofs)

No service (carried as `AppRequest`/`AppResponse` app bytes, `04`/`05`). Messages:
- `ProofRequest`: oneof `request`: `change_proof`:1:`ChangeProofRequest`; `range_proof`:2:`RangeProofRequest`.
- `ChangeProofRequest{start_root_hash:1, end_root_hash:2, start_key:3:MaybeBytes, end_key:4:MaybeBytes, key_limit:5:uint32, bytes_limit:6:uint32}`.
- `RangeProofRequest{root_hash:1, start_key:2:MaybeBytes, end_key:3:MaybeBytes, key_limit:4:uint32, bytes_limit:5:uint32}`.
- `ProofResponse`: oneof `response`: `change_proof`:1:`bytes`; `range_proof`:2:`bytes`.
- `ChangeProof{start_proof:1:repeated ProofNode, end_proof:2:repeated ProofNode, key_changes:3:repeated KeyChange}`.
- `RangeProof{start_proof:1:repeated ProofNode, end_proof:2:repeated ProofNode, key_values:3:repeated KeyValue}`.
- `ProofNode{key:1:Key, value_or_hash:2:MaybeBytes, children:3:map<uint32,bytes>}`.  ← **only proto map on the wire in the whole tree** (see §6 caveat).
- `KeyChange{key:1, value:2:MaybeBytes}`; `Key{length:1:uint64, value:2:bytes}`; `MaybeBytes{value:1:bytes}` (presence = "something"); `KeyValue{key:1, value:2}`.
> The leaf node serialization that feeds merkledb root hashing is the **merkledb
> codec** (`04`), byte-exact; proto here is only the transport envelope.

### 3.11 `http` — `proto/http/http.proto` (HTTP-over-gRPC for plugin API handlers)

`service HTTP`: `Handle(HTTPRequest)→HTTPResponse` (full http1-over-http2 with inline conn/responsewriter servers, supports websockets); `HandleSimple(HandleSimpleHTTPRequest)→HandleSimpleHTTPResponse` (headers+body only, cheaper).
Messages mirror Go `net/http`/`net/url`/`crypto/tls`: `URL{scheme:1, opaque:2, user:3:Userinfo, host:4, path:5, raw_path:6, force_query:7:bool, raw_query:8, fragment:9}`; `Userinfo{username:1, password:2, password_set:3:bool}`; `Element{key:1, values:2:repeated string}`; `Certificates{cert:1:repeated bytes}`; `ConnectionState{version:1:uint32, handshake_complete:2:bool, did_resume:3:bool, cipher_suite:4:uint32, negotiated_protocol:5, server_name:6, peer_certificates:7:Certificates, verified_chains:8:repeated Certificates, signed_certificate_timestamps:9:repeated bytes, ocsp_response:10:bytes}`; `Request{method:1, url:2:URL, proto:3, proto_major:4:int32, proto_minor:5:int32, header:6:repeated Element, content_length:8:int64, transfer_encoding:9:repeated string, host:10, form:11:repeated Element, post_form:12:repeated Element, trailer_keys:13:repeated string, remote_addr:14, request_uri:15, tls:16:ConnectionState}` (**no field 7**); `ResponseWriter{header:1:repeated Element, server_addr:2}`; `HTTPRequest{response_writer:1:ResponseWriter, request:2:Request}`; `HTTPResponse{header:1:repeated Element}`; `HandleSimpleHTTPRequest{method:1, url:2:string, request_headers:3:repeated Element, body:4:bytes, response_headers:5:repeated Element}`; `HandleSimpleHTTPResponse{code:1:int32, headers:2:repeated Element, body:3:bytes}`.

### 3.12 `http.responsewriter` — `proto/http/responsewriter/responsewriter.proto`

`service Writer` (plugin dials the node's ResponseWriter): `Write(WriteRequest)→WriteResponse`, `WriteHeader(WriteHeaderRequest)→Empty`, `Flush(Empty)→Empty` (no-op), `Hijack(Empty)→HijackResponse`.
`Header{key:1, values:2:repeated string}`; `WriteRequest{headers:1:repeated Header, payload:2:bytes}`/`WriteResponse{written:1:int32}`; `WriteHeaderRequest{headers:1:repeated Header, status_code:2:int32}`; `HijackResponse{local_network:1, local_string:2, remote_network:3, remote_string:4, server_addr:5}`.

### 3.13 `io.reader` / `io.writer` / `net.conn` (hijack plumbing)

- `io.reader` `service Reader`: `Read(ReadRequest{length:1:int32})→ReadResponse{read:1:bytes, error:2:Error}`. `Error{error_code:1:ErrorCode, message:2}`; enum `ErrorCode`{`UNSPECIFIED`=0,`EOF`=1}.
- `io.writer` `service Writer`: `Write(WriteRequest{payload:1:bytes})→WriteResponse{written:1:int32, error:2:optional string}`.
- `net.conn` `service Conn`: `Read(ReadRequest)→ReadResponse`, `Write(WriteRequest)→WriteResponse`, `Close(Empty)→Empty`, `SetDeadline/SetReadDeadline/SetWriteDeadline(SetDeadlineRequest)→Empty`. `ReadRequest{length:1:int32}`/`ReadResponse{read:1:bytes, error:2:Error}`; `WriteRequest{payload:1:bytes}`/`WriteResponse{length:1:int32, error:2:optional string}`; `SetDeadlineRequest{time:1:bytes}` (Go-`time.Time` GobEncode bytes); `Error{error_code:1, message:2}`; enum `ErrorCode`{`UNSPECIFIED`=0,`EOF`=1,`OS_ERR_DEADLINE_EXCEEDED`=2}.

### 3.14 `aliasreader` — `proto/aliasreader/aliasreader.proto`

`service AliasReader` (node serves): `Lookup(Alias)→ID`, `PrimaryAlias(ID)→Alias`, `Aliases(ID)→AliasList`. `ID{id:1:bytes}`; `Alias{alias:1:string}`; `AliasList{aliases:1:repeated string}`. (Reuses messages as req & resp — allowed by buf config; lint-ignored.)

### 3.15 `platformvm` — `proto/platformvm/platformvm.proto`

No service. Helper types for L1/ACP-77 warp justifications: `L1ValidatorRegistrationJustification` oneof `preimage`: `convert_subnet_to_l1_tx_data`:1:`SubnetIDIndex`; `register_l1_validator_message`:2:`bytes`. `SubnetIDIndex{subnet_id:1:bytes, index:2:uint32}`. These get linear-codec-wrapped into warp justification payloads (`08`).

### 3.16 `sdk` — `proto/sdk/sdk.proto` (p2p app-level SDK)

No service — these are **AppRequest/AppResponse/AppGossip bodies** (`network/p2p`, `05`/`07`). `PullGossipRequest{salt:2:bytes, filter:3:bytes}` (**note: no field 1**); `PullGossipResponse{gossip:1:repeated bytes}`; `PushGossip{gossip:1:repeated bytes}`; `SignatureRequest{message:1:bytes, justification:2:bytes}` (ACP-118 warp BLS sig request); `SignatureResponse{signature:1:bytes}`.

### 3.17 Connect: `proposervm` — `connectproto/proposervm/service.proto`

`go_package = .../connectproto/pb/proposervm`. `service ProposerVM` (Connect protocol, `12`): `GetProposedHeight(GetProposedHeightRequest{})→GetProposedHeightReply{height:1:uint64}`; `GetCurrentEpoch(GetCurrentEpochRequest{})→GetCurrentEpochReply{number:1:uint64, p_chain_height:2:uint64, start_time:3:int64}`.

### 3.18 Connect: `xsvm` — `connectproto/xsvm/service.proto`

`go_package = .../connectproto/pb/xsvm`. `service Ping` (the **only streaming** service in the tree): `Ping(PingRequest)→PingReply` (unary); `StreamPing(stream StreamPingRequest)→stream StreamPingReply` (**bidi streaming**). `PingRequest/PingReply{message:1:string}`; `StreamPingRequest/StreamPingReply{message:1:string}`.

---

## 4. Non-protobuf serialization formats (index)

### 4.1 Avalanche linear codec (BYTE-EXACT) — see `03` §2

Wire rules (summary; authoritative detail in `03`): big-endian integers; `u32`
length+count prefix for `Vec`/maps; `u16` length for `String`; fixed `[u8;N]` with
no prefix; a **2-byte codec version** prepended by `codec.Manager.Marshal`; a
`u32` **typeID** prefix for each interface field, assigned in **registration
order** from 0 (with `SkipRegistrations` gaps); trailing-byte check (`ErrExtraSpace`)
on unmarshal; `len > i32::MAX` rejected (`ErrMaxSliceLenExceeded`).

**Artifacts encoded with the linear codec (all byte-exact):**
- All **P-Chain** and **X-Chain** transaction & block bytes (the `bytes` in p2p
  `Put`/`Ancestors`, `vm.proto` block bytes) — `08`/`09`.
- **C-Chain atomic** import/export tx bytes (NOT RLP) — `09`/`10`, ATOMIC-1.
- The **genesis bytes** produced from genesis JSON — `12`.
- **Warp** `UnsignedMessage` / `AddressedCall` / `BitSetSignature` / `Payload` and
  the L1 justification payloads (§3.8, §3.15) — `08`.
- **State-sync** merkledb proof leaf/node serialization feeding root hashing
  (proto `sync` is only the envelope) — `04`.
- The **signed-IP** message (`ip_signing_time` + ip/port body) hashed & signed in
  `Handshake`/`ClaimedIpPort` — `05`.

### 4.2 p2p framing & compression — see `05` §1.1, §1.3

4-byte BE `uint32` length prefix + proto3 `p2p.Message`; max payload
`DefaultMaxMessageSize = 2 MiB`; length `>2 MiB` → drop. Compression = **zstd**
only, via the self-nested `compressed_zstd` (field 2) `Message`-in-`Message`
trick; per-op decision copied from `outbound_msg_builder.go`. zstd is
**decode-compatible only** (R4), not byte-identical.

### 4.3 EVM RLP — see `10`

C-Chain/EVM-subnet blocks, headers, txs, receipts use Ethereum **RLP** via
`reth`/`alloy` (`alloy-rlp`); hashes (block, tx, receipt, state root via
Firewood-ethhash) must match. **Exception:** the C-Chain **atomic** tx codec is the
avalanche **linear codec**, not RLP (§4.1).

### 4.4 Address / ID string encodings — see `03` §1, §3.2, §3.3

- **CB58** = `base58(payload ++ last4(sha256(payload)))` — `ids.ID`/`ShortID`/raw
  byte encodings. Single sha256 tail (cf. Bitcoin's double sha256).
- **`NodeID-<CB58>`** for `NodeID` display; parse requires the prefix.
- **bech32** chain-prefixed: `"<chainAlias>-<bech32(hrp, addr20)>"` (e.g. `X-avax1…`,
  `P-fuji1…`). HRP per network: `avax`(mainnet), `fuji`, `local`, `custom`(fallback).
- **hex `0x`**: EVM addresses/hashes; `formatting.Encode` Hex/HexC = `0x+hex(payload++ck4)`,
  HexNC = `0x+hex(payload)` (no checksum).

### 4.5 JSON-RPC 2.0 / Connect / genesis JSON — see `14`, `12`

JSON-RPC 2.0 over HTTP (`gorilla/rpc` → axum shim) for info/platform/avm/C-Chain;
Connect-protocol endpoints (§3.17–3.18) over the shared h2c port (R5); genesis is
JSON input → linear-codec genesis bytes (`12`).

---

## 5. prost / tonic mapping

**Codegen.** `proto/` and `connectproto/` each get a `build.rs` invocation of
`tonic-build`/`prost-build` (Bazel: `rust_prost_library` + `rust_tonic_library`).
Generated code is **not committed** (`00`/decision 8). Package `p2p` → module
`p2p`; nested package `http.responsewriter` → `http::responsewriter`; `io.reader`
→ `io::reader`; `net.conn` → `net::conn`; `vm.runtime` → `vm::runtime`. The
`go_package` option is Go-only; Rust module paths derive from the proto `package`.

**Service traits.** Each `service` yields, via tonic:
- a server trait `#[async_trait] trait <Svc>` (e.g. `vm_server::Vm`,
  `database_server::Database`) implemented by our handlers, plus a
  `<Svc>Server<T>` tower service;
- a client `<Svc>Client<Channel>` with one async method per RPC.

Direction matters (from `07`): the **plugin** implements `vm_server::Vm` and
`runtime_client`; the **node** implements `database_server`, `appsender_server`,
`sharedmemory_server`, `validatorstate_server`, `aliasreader_server`,
`warp/signer_server`, `http*` servers, and `runtime_server`. tonic clients/servers
are generated for both sides so Rust↔Go interop works in either role.

**Proto → Rust type mapping (prost):**

| Proto | Rust (prost) | Care |
|---|---|---|
| `oneof` | a generated `enum` in a sub-module; field is `Option<Enum>` | `Message.message`, `Simplex.message`, `ProofRequest/Response.{request,response}`, `L1ValidatorRegistrationJustification.preimage` |
| `optional <scalar>` | `Option<T>` | `BuildBlockRequest.p_chain_height`, `BlockVerifyRequest.p_chain_height`, `io.writer`/`net.conn` `error` |
| `bytes` | `Vec<u8>` by default — **configure `bytes(".")` so they map to `bytes::Bytes`** for zero-copy on hot paths (`00` §9) | all the opaque codec/RLP payloads |
| `repeated bytes` | `Vec<Bytes>` (with the above) | |
| `map<uint32,bytes>` | `HashMap<u32, Bytes>` | **only** `ProofNode.children` — see §6 caveat |
| `enum` | Rust `enum` with `#[repr(i32)]`; unknown → keep raw `i32` via prost open-enum | `State`, `Error`, `EngineType`, `Message`, `ErrorCode`, `Mode` |
| `google.protobuf.Empty` | `()` (tonic maps to `Empty`) | many RPCs |
| `google.protobuf.Timestamp` | `prost_types::Timestamp` → convert to/from `SystemTime` | block timestamps, deadlines |
| `google.protobuf.Duration` | `prost_types::Duration` | `granite_epoch_duration` |
| `io.prometheus.client.MetricFamily` | vendored prometheus-client-model proto compiled into our build (Go excludes it; **we must include it**) | `GatherResponse` |
| `sint32` | `i32` (zigzag) | `error_code` fields |

Well-known types used: `Empty`, `Timestamp`, `Duration` (`prost-types`), and the
external `io/prometheus/client/metrics.proto`. The Connect services (`proposervm`,
`xsvm`) are compiled the same way; serving them over the Connect protocol needs the
Connect compat shim (`12`/R5). `xsvm.Ping.StreamPing` is the lone bidi-streaming RPC
→ tonic `Streaming<T>` in/out.

---

## 6. Byte-exactness matrix

| Format | Rust output byte-identical to Go? | Rationale / mechanism |
|---|---|---|
| Linear codec (all P/X tx/block, genesis, warp, signed-IP, atomic tx) | **YES** | hand-written `ava-codec`; typeID order + 2-byte version pinned; golden tests |
| p2p framing (4-byte BE len + proto) | **YES** | identical length prefix, 2 MiB cap |
| zstd compressed payload | **NO — decode-compatible only** | R4; libraries differ; harness asserts mutual decode |
| protobuf messages | **canonical-proto-equivalent** | proto3 has no canonical-bytes guarantee, but avalanchego’s usage is deterministic: **no `map` on any consensus/codec path** (the single `map` — `ProofNode.children` — rides inside an `AppResponse` that is itself never re-hashed, so map nondeterminism is harmless). prost emits fields in tag order; matches Go for the shapes used. Treat as semantically equal, not byte-equal. |
| EVM RLP (blocks/txs/receipts/header) | **YES** | RLP is canonical; reth/alloy hashes must equal Go |
| EVM state root | **YES** | Firewood-ethhash = Keccak/Eth-MPT (`04`/`10`) |
| merkledb root / proof leaves | **YES** | `ava-merkledb` faithful reimpl (`04`) |
| CB58 / bech32 / hex / NodeID string | **YES** | deterministic encoders; golden-vector each |
| Genesis bytes & block ID | **YES** | JSON → linear codec → identical ID |
| JSON-RPC payloads | structurally exact | field names/casing/types match; whitespace not significant |

**Proto map nondeterminism (the one caveat):** proto3 maps have no defined wire
order, so they are forbidden anywhere bytes are hashed/signed. avalanchego avoids
maps on the wire entirely except `ProofNode.children` (state-sync transport, never
re-serialized for hashing). Rust: never serialize a `HashMap` on a byte-exact path;
the merkledb node hashing uses the merkledb codec, not proto (`04`). See `00` §6.1.

**Codec version bytes:** the 2-byte BE version prefix is part of every
linear-codec artifact and part of consensus validation — never omit it; the
unknown-version and extra-space checks are mandatory (`03` §2.2).

---

## 7. Test-vector plan (ref `02`)

Golden vectors per format, extracted from the Go tree and asserted bit-for-bit
(differential where applicable):

| Format | Golden source (Go) | Test |
|---|---|---|
| p2p `Message` per oneof variant | `message/messages_test.go`, `proto/pb/p2p` round-trips | encode our `p2p.Message` for each tag, diff bytes vs Go-marshaled fixture; assert field numbers |
| zstd recursive packing | `message/messages_test.go` compression cases | assert Go-compressed frame decodes to our inner `Message` and vice-versa (decode-only, R4) |
| Linear codec primitives & typeIDs | `codec/`, `vms/.../codec.go` registration order; dump typeID table | golden typeID table per interface (`03` §7); round-trip P/X tx/block fixtures |
| Genesis bytes/ID | `genesis/genesis_test.go` (mainnet/fuji) | recompute genesis bytes → assert identical block ID |
| Warp UnsignedMessage | `vms/platformvm/warp/*_test.go` | encode/sign → byte+sig match |
| Atomic UTXO (ATOMIC-1) | shared-memory + `secp256k1fx` fixtures | X↔P / X↔C decode cross-tests |
| State-sync proofs | `x/merkledb/.../sync` proof fixtures | proto envelope round-trip + merkledb root equality |
| rpcchainvm | `vms/rpcchainvm/*_test.go` | host a Rust VM under a Go node and vice-versa; assert v45 handshake + each RPC |
| rpcdb/appsender/sharedmemory/validatorstate/aliasreader | each package’s Go tests | tonic client↔Go server and Go client↔tonic server round-trips |
| CB58/bech32/hex/NodeID | `utils/cb58`, `utils/formatting/address` tests | golden string round-trips per network HRP |
| EVM RLP | `graft/coreth` / reth fixtures | block/tx/receipt hash equality (`10`) |
| Connect proposervm/xsvm | `connectproto` Go tests | Connect-protocol request/response parity |

---

## 8. Performance notes / improvements over Go

- Map `bytes`/`repeated bytes` proto fields to `bytes::Bytes` (prost `bytes(".")`)
  so the p2p read path and rpcdb iterator pages avoid per-message copies (`00` §9).
- Reuse a single `zstd` encoder/decoder context per peer connection (window = 2 MiB)
  rather than per-message init.
- Linear-codec `marshal` pre-sizes the `Packer` buffer via the `size()` pass (as Go
  does) to avoid reallocation; safe — output is unchanged.
- All such changes are byte-output-preserving (or decode-equivalent for zstd) and
  are gated by the §7 golden/differential vectors.
