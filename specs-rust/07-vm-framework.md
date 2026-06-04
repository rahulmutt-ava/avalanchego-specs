# 07 — VM Framework (`ava-vm`, `ava-vm-rpc`, `ava-chains`, `ava-secp256k1fx`)

> **Status:** Conforms to `00-overview-and-conventions.md`. This spec defines the
> VM contract that the consensus engine (`06-consensus.md`) drives, the shared VM
> building blocks (`avax`/`verify`/`chain`/`gas`), the feature-extension (`fx`)
> framework + `secp256k1fx`, the VM middleware (metervm/tracedvm), the
> out-of-process gRPC plugin host/guest (`rpcchainvm`), the VM registry/manager,
> and the `chains::Manager` chain-creation pipeline. The concrete VMs (`08`–`11`)
> implement the traits fixed here.
>
> **Binding cross-spec boundary (from `06`):** the engine calls `ChainVm`/`Block`;
> a VM receives an `Arc<ChainContext>` + a `Database` (`Arc<dyn DynDatabase>`, see
> `04`) + an `AppSender` handle at `initialize`. Those exact shapes are reproduced
> below and **must not drift** — `08`/`09`/`10`/`11` depend on them.

---

## 1. Go source → Rust mapping

| Go area | Rust |
|---|---|
| `snow/engine/common/vm.go` (`VM`), `snow/engine/common/engine.go` (`AppHandler`), `validators/connector.go`, `api/health` | `ava-vm::{Vm, AppHandler, HealthCheck, Connector}` |
| `snow/engine/snowman/block/{vm,batched_vm,block_context_vm,state_syncable_vm,state_summary,state_sync_mode}.go`, `snow/consensus/snowman/block.go` | `ava-vm::block::{ChainVm, Block, ...}` |
| `vms/manager.go`, `vms/registry/` | `ava-vm::{VmManager, VmRegistry, Factory}` |
| `vms/rpcchainvm/`, `vms/rpcchainvm/runtime/`, `proto/vm`, `proto/vm/runtime` | `ava-vm-rpc` |
| `proto/{rpcdb,appsender,sharedmemory,messenger,validatorstate,warp,http,net,io,aliasreader,keystore}` | `ava-vm-rpc::proxy::*` |
| `vms/components/avax/` | `ava-vm::components::avax` |
| `vms/components/verify/` | `ava-vm::components::verify` |
| `vms/components/chain/` | `ava-vm::components::chain` (state/cache decorator) |
| `vms/components/gas/` | `ava-vm::components::gas` |
| `vms/txs/mempool/` | `ava-vm::mempool` |
| `vms/fx/`, `vms/secp256k1fx/` | `ava-vm::fx`, `ava-secp256k1fx` |
| `vms/nftfx`, `vms/propertyfx` | `ava-avm` (re-uses `fx` trait; see `09`) |
| `vms/metervm/`, `vms/tracedvm/` | `ava-vm::middleware::{MeterVm, TracedVm}` |
| `chains/manager.go`, `chains/atomic/` | `ava-chains` |

---

## 2. The core VM trait family (`ava-vm::block`)

The Go contract is a tower of embedded interfaces:
`block.ChainVM` ⊇ `common.VM` ⊇ `AppHandler` + `health.Checker` + `validators.Connector`,
plus `Getter`/`Parser` and the optional extensions (`BatchedChainVM`,
`BuildBlockWithContextChainVM`, `SetPreferenceWithContextChainVM`,
`StateSyncableVM`, and per-block `WithVerifyContext`). In Rust we keep the same
decomposition (one trait per Go interface, object-safe via `#[async_trait]`), so
the engine can hold `Arc<dyn ChainVm>` and downcast/probe optional capabilities.

All async traits use `async_trait::async_trait` (per `00` §4.1). Cancellation is a
`&CancellationToken` argument (replacing `context.Context`); deadlines are passed
as `std::time::Instant` where Go passes a `time.Time` deadline. Errors are the
crate `Error` (`thiserror`); the Go sentinel `database.ErrNotFound` is preserved as
`Error::NotFound` and asserted via `matches!` (mirrors `errors.Is`).

### 2.1 `Vm` — the `common.VM` base

```rust
/// Mirrors `snow/engine/common.VM`. The base every VM implements.
#[async_trait::async_trait]
pub trait Vm: AppHandler + HealthCheck + Connector + Send + Sync {
    /// `Initialize`. The VM receives only immutable identity/handles
    /// (`Arc<ChainContext>`, from 06) — NOT the engine's `ConsensusContext`.
    /// `db` is the per-chain VM database (04). `app_sender` is the outbound
    /// app-message handle. `fxs` are the feature extensions attached to this VM.
    #[allow(clippy::too_many_arguments)]
    async fn initialize(
        &mut self,
        token: &CancellationToken,
        chain_ctx: Arc<ChainContext>,        // 06 §3
        db: Arc<dyn DynDatabase>,            // 04
        genesis_bytes: &[u8],
        upgrade_bytes: &[u8],
        config_bytes: &[u8],
        fxs: Vec<Fx>,                        // §6
        app_sender: Arc<dyn AppSender>,      // §2.6
    ) -> Result<()>;

    /// `SetState` — engine tells the VM its phase. Replaces the engine reading
    /// the VM's state off a shared bag (06 keeps `EngineState` in the engine).
    async fn set_state(&mut self, token: &CancellationToken, state: EngineState) -> Result<()>;

    async fn shutdown(&mut self, token: &CancellationToken) -> Result<()>;
    async fn version(&self, token: &CancellationToken) -> Result<String>;

    /// `CreateHandlers` — `[extension] -> HTTP handler` served under
    /// `/ext/bc/[chainID]/[extension]` (12). Returned as boxed `tower::Service`s.
    async fn create_handlers(&mut self, token: &CancellationToken)
        -> Result<HashMap<String, HttpHandler>>;

    /// `NewHTTPHandler` — single handler routed via the `server.HTTPHeaderRoute`
    /// header carrying this chain's id.
    async fn new_http_handler(&mut self, token: &CancellationToken)
        -> Result<Option<HttpHandler>>;

    /// `WaitForEvent` — blocks until the VM has something for the engine
    /// (e.g. `PendingTxs`) or the token is cancelled. The handler task awaits
    /// this and forwards via `InternalHandler::notify` (06 §4.1).
    async fn wait_for_event(&self, token: &CancellationToken) -> Result<VmEvent>;
}

/// `snow/engine/common.Message`. Wire/internal enum — values match Go.
#[derive(Clone, Copy, PartialEq, Eq)]
pub enum VmEvent { PendingTxs = 1, StateSyncDone = 2 }
```

`HttpHandler` is `Box<dyn tower::Service<http::Request<Body>, Response = http::Response<Body>>>`
plus a `lock_options` field mirroring Go's `common.HTTPHandler{LockOptions}`
(no-lock / read / write); in Rust the lock semantics collapse since the VM is its
own actor, but the enum is preserved for gRPC wire parity (see §5).

### 2.2 `AppHandler`, `HealthCheck`, `Connector`

```rust
/// `snow/engine/common.AppHandler` (the VM's inbound app-message side).
#[async_trait::async_trait]
pub trait AppHandler: Send + Sync {
    async fn app_request(&mut self, token: &CancellationToken, node: NodeId,
        request_id: u32, deadline: std::time::Instant, request: &[u8]) -> Result<()>;
    async fn app_request_failed(&mut self, token: &CancellationToken, node: NodeId,
        request_id: u32, err: AppError) -> Result<()>;
    async fn app_response(&mut self, token: &CancellationToken, node: NodeId,
        request_id: u32, response: &[u8]) -> Result<()>;
    async fn app_gossip(&mut self, token: &CancellationToken, node: NodeId,
        msg: &[u8]) -> Result<()>;
}

/// `api/health.Checker`. Healthy ⇒ `Ok(json)`; unhealthy ⇒ `Err`.
#[async_trait::async_trait]
pub trait HealthCheck: Send + Sync {
    async fn health_check(&self, token: &CancellationToken) -> Result<serde_json::Value>;
}

/// `snow/validators.Connector`.
#[async_trait::async_trait]
pub trait Connector: Send + Sync {
    async fn connected(&mut self, token: &CancellationToken, node: NodeId,
        version: AppVersion) -> Result<()>;   // ava-version (03)
    async fn disconnected(&mut self, token: &CancellationToken, node: NodeId) -> Result<()>;
}

/// `snow/engine/common.AppError` — typed app error, `i32` code + message.
/// `Is` compares by code (Go). Predefined codes match Go integer values.
#[derive(Clone, thiserror::Error, Debug)]
#[error("{code}: {message}")]
pub struct AppError { pub code: i32, pub message: String }
impl AppError {
    pub const UNDEFINED: i32 = 0;   // ErrUndefined
    pub const TIMEOUT: i32   = -1;  // ErrTimeout
}
```

### 2.3 `Block` — re-exported from `06`

The `Block` trait is **owned by `06`** (it is `snow/consensus/snowman.Block` +
`snow.Decidable`) and re-exported by `ava-vm`. Reproduced here for reference;
`08`–`11` implement it:

```rust
#[async_trait::async_trait]
pub trait Block: Send + Sync {
    fn id(&self) -> Id;                       // Decidable.ID
    fn parent(&self) -> Id;
    fn height(&self) -> u64;
    fn timestamp(&self) -> std::time::SystemTime;
    fn bytes(&self) -> &[u8];
    async fn verify(&self, token: &CancellationToken) -> Result<()>;
    async fn accept(&self, token: &CancellationToken) -> Result<()>;
    async fn reject(&self, token: &CancellationToken) -> Result<()>;
}
```

Optional per-block extension (`block.WithVerifyContext`), probed by the engine via
`Option<&dyn WithVerifyContext>` (downcast helper on the wrapper block):

```rust
/// `block.WithVerifyContext`. Implemented when a block's validity depends on the
/// P-Chain height (proposervm-driven). Only called when proposervm is activated.
#[async_trait::async_trait]
pub trait WithVerifyContext: Send + Sync {
    async fn should_verify_with_context(&self, token: &CancellationToken) -> Result<bool>;
    async fn verify_with_context(&self, token: &CancellationToken, ctx: &BlockContext) -> Result<()>;
}

/// `block.Context` (canoto type — field id must match for proto/canoto parity).
pub struct BlockContext { pub p_chain_height: u64 }
```

### 2.4 `ChainVm` — the Snowman VM

```rust
/// `block.ChainVM`. The engine (06) holds `Arc<Mutex<dyn ChainVm>>` per chain
/// (the per-chain handler is the only caller, so contention is nil — see 06 §5).
#[async_trait::async_trait]
pub trait ChainVm: Vm {
    /// `BuildBlock`. Err if the VM doesn't want to issue a block.
    async fn build_block(&mut self, token: &CancellationToken) -> Result<Arc<dyn Block>>;

    /// `Getter.GetBlock`. `Err(Error::NotFound)` if unknown.
    async fn get_block(&self, token: &CancellationToken, id: Id) -> Result<Arc<dyn Block>>;

    /// `Parser.ParseBlock`. Bytes must round-trip to the same block on any node.
    async fn parse_block(&self, token: &CancellationToken, bytes: &[u8]) -> Result<Arc<dyn Block>>;

    /// `SetPreference` — the engine's currently preferred (leaf) block.
    async fn set_preference(&mut self, token: &CancellationToken, id: Id) -> Result<()>;

    /// `LastAccepted` — defaults to genesis if nothing accepted yet.
    async fn last_accepted(&self, token: &CancellationToken) -> Result<Id>;

    /// `GetBlockIDAtHeight`. `Err(Error::NotFound)` if the height index is
    /// unknown (e.g. after state sync).
    async fn get_block_id_at_height(&self, token: &CancellationToken, height: u64) -> Result<Id>;

    // ---- optional capability probes (Go: type assertions). Default = unsupported.

    /// `BuildBlockWithContextChainVM`. Called iff proposervm is active.
    fn as_build_with_context(&self) -> Option<&dyn BuildBlockWithContext> { None }
    /// `SetPreferenceWithContextChainVM`.
    fn as_set_preference_with_context(&self) -> Option<&dyn SetPreferenceWithContext> { None }
    /// `BatchedChainVM`.
    fn as_batched(&self) -> Option<&dyn BatchedChainVm> { None }
    /// `StateSyncableVM`.
    fn as_state_syncable(&self) -> Option<&dyn StateSyncableVm> { None }
}

#[async_trait::async_trait]
pub trait BuildBlockWithContext: Send + Sync {
    async fn build_block_with_context(&self, token: &CancellationToken,
        ctx: &BlockContext) -> Result<Arc<dyn Block>>;
}
#[async_trait::async_trait]
pub trait SetPreferenceWithContext: Send + Sync {
    async fn set_preference_with_context(&self, token: &CancellationToken,
        id: Id, ctx: &BlockContext) -> Result<()>;
}
```

> **Mutability note.** Go's `ChainVM` mixes read-only (`GetBlock`, `LastAccepted`)
> and mutating (`BuildBlock`, `SetPreference`, `Initialize`) methods on one
> interface, relying on `ctx.Lock`. In Rust we drop the lock (per `06` §3) and let
> the owning per-chain handler hold `&mut`/`&` as appropriate. The trait above uses
> `&mut self` only for the genuinely mutating ops; read ops take `&self`. Object
> safety is preserved (no generic methods).

### 2.5 Batched + state-syncable VMs

```rust
/// `block.BatchedChainVM` — efficient bulk fetch/parse over the network.
#[async_trait::async_trait]
pub trait BatchedChainVm: Send + Sync {
    async fn get_ancestors(&self, token: &CancellationToken, blk_id: Id,
        max_blocks_num: usize, max_blocks_size: usize,
        max_retrieval_time: std::time::Duration) -> Result<Vec<Vec<u8>>>;
    async fn batched_parse_block(&self, token: &CancellationToken,
        blks: &[Vec<u8>]) -> Result<Vec<Arc<dyn Block>>>;
}

/// `block.StateSyncableVM`.
#[async_trait::async_trait]
pub trait StateSyncableVm: Send + Sync {
    async fn state_sync_enabled(&self, token: &CancellationToken) -> Result<bool>;
    async fn get_ongoing_sync_state_summary(&self, token: &CancellationToken)
        -> Result<Arc<dyn StateSummary>>;          // Err(NotFound) if none
    async fn get_last_state_summary(&self, token: &CancellationToken)
        -> Result<Arc<dyn StateSummary>>;
    async fn parse_state_summary(&self, token: &CancellationToken, bytes: &[u8])
        -> Result<Arc<dyn StateSummary>>;
    async fn get_state_summary(&self, token: &CancellationToken, height: u64)
        -> Result<Arc<dyn StateSummary>>;
}

/// `block.StateSummary`.
#[async_trait::async_trait]
pub trait StateSummary: Send + Sync {
    fn id(&self) -> Id;
    fn height(&self) -> u64;
    fn bytes(&self) -> &[u8];
    async fn accept(&self, token: &CancellationToken) -> Result<StateSyncMode>;
}

/// `block.StateSyncMode` — values match Go (1/2/3).
#[derive(Clone, Copy, PartialEq, Eq)]
pub enum StateSyncMode { Skipped = 1, Static = 2, Dynamic = 3 }
```

The default `get_ancestors`/`batched_parse_block` *fallbacks* (Go's free
functions `block.GetAncestors`/`block.BatchedParseBlock`, which walk parents one
at a time when the VM is not batched) live in `ava-vm` as helpers operating on a
`&dyn ChainVm`; they replicate the `maxBlocksNum`/`maxBlocksSize`
(`+ wrappers.IntLen` per element) / `maxBlocksRetrievalTime` accounting and the
`Err(NotFound) ⇒ empty/break` special-casing byte-for-byte.

### 2.6 `AppSender` — the outbound handle the VM gets at init

This is the VM-facing **subset** of `06`'s `Sender` (`common.AppSender`). The VM
never sees the consensus `Sender`; it only gets `AppSender`.

```rust
/// `snow/engine/common.AppSender`. Handed to the VM at `initialize`.
#[async_trait::async_trait]
pub trait AppSender: Send + Sync {
    async fn send_app_request(&self, token: &CancellationToken,
        nodes: &HashSet<NodeId>, request_id: u32, bytes: Vec<u8>) -> Result<()>;
    async fn send_app_response(&self, token: &CancellationToken,
        node: NodeId, request_id: u32, bytes: Vec<u8>) -> Result<()>;
    async fn send_app_error(&self, token: &CancellationToken,
        node: NodeId, request_id: u32, code: i32, message: &str) -> Result<()>;
    async fn send_app_gossip(&self, token: &CancellationToken,
        config: SendConfig, bytes: Vec<u8>) -> Result<()>;   // SendConfig from 06
}
```

---

## 3. Shared components (`ava-vm::components`)

### 3.1 `avax` — the UTXO model

Byte-exact codec types (per `03`; `serialize:"true"` fields → `#[codec]` in
registration order). The `FxID` fields are `serialize:"false"` — **not encoded**
(set at runtime from the registered fx). `verify::State`/`TransferableIn`/
`TransferableOut` are object-safe trait objects (`Arc<dyn ...>`) so a VM can mix
fx output types in one tx.

```rust
/// `avax.UTXOID`. `id` is derived lazily: `TxID.prefix(output_index as u64)`
/// (= sha256 of TxID ++ be64(index)); cached. `Symbol` is runtime-only.
pub struct UtxoId {
    pub tx_id: Id,            // serialize
    pub output_index: u32,    // serialize
    // runtime-only:
    symbol: bool,
    id: OnceCell<Id>,
}
impl UtxoId {
    pub fn input_id(&self) -> Id { *self.id.get_or_init(|| self.tx_id.prefix(&[self.output_index as u64])) }
}

/// `avax.Asset`.
pub struct Asset { pub id: Id }   // serialize; Verify: non-empty

/// `avax.UTXO`.
pub struct Utxo {
    pub utxo_id: UtxoId,                 // serialize (embedded)
    pub asset: Asset,                    // serialize (embedded)
    pub out: Arc<dyn verify::State>,     // serialize (fx output, typeID-tagged)
}

/// `avax.TransferableOutput`. Sorted by (assetID, then codec(out) bytes).
pub struct TransferableOutput {
    pub asset: Asset,                    // serialize
    pub fx_id: Id,                       // serialize:"false" — runtime only
    pub out: Arc<dyn TransferableOut>,   // serialize
}

/// `avax.TransferableInput`. Sorted+unique by UTXOID (TxID, then index).
pub struct TransferableInput {
    pub utxo_id: UtxoId,                 // serialize (embedded)
    pub asset: Asset,                    // serialize (embedded)
    pub fx_id: Id,                       // serialize:"false" — runtime only
    pub r#in: Arc<dyn TransferableIn>,   // serialize
}

/// `avax.TransferableIn`/`TransferableOut`.
pub trait TransferableOut: verify::State { fn amount(&self) -> u64; }
pub trait TransferableIn: verify::Verifiable {
    fn amount(&self) -> u64;
    fn cost(&self) -> Result<u64>;       // Coster
}
```

Sorting (`SortTransferableOutputs`/`SortTransferableInputsWithSigners`,
`IsSortedTransferableOutputs`) is **consensus-affecting** — reproduce Go's
comparator exactly: outputs sort by `assetID` then by the codec-marshalled
`Out` bytes (`bytes.Compare`); inputs sort by `UTXOID` (`TxID` then
`OutputIndex`). Per `00` §6.1 we never serialize a map; we `sort` exactly where Go
does. The signer-paired input sort permutes the signer slice in lockstep.

`avax.FlowChecker` (`flow_checker.go`) — the `Produce`/`Consume` per-asset balance
ledger used by `VerifyTx`: `VerifyTx(fee, fee_asset, ins, outs, codec)` burns the
fee, sums produced/consumed per asset, and requires `consumed >= produced` for
each asset plus all-sorted. Port the integer overflow checks via `checked_add`
(`safemath`, `03`).

`avax.Metadata` / `BaseTx` (`base_tx.go`): the common tx preamble
`{network_id: u32, blockchain_id: Id, outs: Vec<TransferableOutput>,
ins: Vec<TransferableInput>, memo: Vec<u8>}` — embedded by P-Chain/X-Chain txs
(`08`/`09`). Cached unsigned/signed `bytes` + `id` (the `Metadata` struct) →
`OnceCell<(Vec<u8>, Id)>` set at `initialize_metadata`.

**Atomic UTXOs / shared memory** (`atomic_utxos.go` + `chains/atomic`): cross-chain
transfers. The `SharedMemory` trait (proxied over gRPC in §5; owned by `ava-chains`)
exposes atomic key/value/traits batched commits:

```rust
/// `chains/atomic.SharedMemory`. Each chain gets its own view (peerChainID-keyed).
pub trait SharedMemory: Send + Sync {
    fn get(&self, peer_chain: Id, keys: &[Vec<u8>]) -> Result<Vec<Vec<u8>>>; // len == keys.len()
    fn indexed(&self, peer_chain: Id, traits: &[Vec<u8>], start_trait: &[u8],
        start_key: &[u8], limit: usize)
        -> Result<(Vec<Vec<u8>>, Vec<u8>, Vec<u8>)>;     // (values, last_trait, last_key)
    /// Atomically apply per-chain Put/Remove requests together with `batches`
    /// (must share the underlying DB — 04). Backs P/X/C atomic state writes.
    fn apply(&self, requests: HashMap<Id, Requests>, batches: &[Box<dyn Batch>]) -> Result<()>;
}
pub struct Requests { pub remove: Vec<Vec<u8>>, pub put: Vec<Element> }   // serialize
pub struct Element { pub key: Vec<u8>, pub value: Vec<u8>, pub traits: Vec<Vec<u8>> } // serialize
```

`addresses.go`: bech32 address formatting (`address.Format(chainAlias, hrp, bytes)`)
and parsing — shared by fx JSON marshalling and the API; lives in `ava-crypto`/
`ava-utils` (`03`) and is re-exported.

### 3.2 `verify`

```rust
/// `verify.Verifiable`.
pub trait Verifiable { fn verify(&self) -> Result<()>; }
/// `verify.State` = ContextInitializable + Verifiable + IsState marker.
/// `InitCtx` becomes an explicit `init_ctx(&ChainContext)` used only for JSON
/// address formatting; it is NOT on the codec path.
pub trait State: Verifiable + Send + Sync {
    fn init_ctx(&self, ctx: &ChainContext);
}
/// `verify.All(..)` — short-circuit on first error.
pub fn all(items: &[&dyn Verifiable]) -> Result<()> { for v in items { v.verify()?; } Ok(()) }
```

The Go `IsState`/`IsNotState` marker split (preventing an `OutputOwners`, which is
`IsNotState`, from being used where a `State` output is expected) is encoded at the
Rust type level by simply not implementing `State` for `OutputOwners`.

### 3.3 `chain` — the block state/cache decorator

`vms/components/chain.State` is a caching layer a VM wraps around its raw
block storage so it presents idempotent `GetBlock`/`ParseBlock` and dedups
in-flight verification. Port as `ava-vm::components::chain::State`, a generic
helper a concrete VM composes (P/X-Chain do; SAE has its own):

```rust
pub struct ChainStateConfig<B: Block> {
    pub decided_cache_size: usize, pub missing_cache_size: usize,
    pub unverified_cache_size: usize, pub bytes_to_id_cache_size: usize,
    pub last_accepted: Arc<B>,
    pub get_block:   Box<dyn Fn(&CancellationToken, Id) -> BoxFuture<Result<Arc<B>>> + Send + Sync>,
    pub unmarshal:   Box<dyn Fn(&CancellationToken, &[u8]) -> BoxFuture<Result<Arc<B>>> + Send + Sync>,
    pub build_block: Box<dyn Fn(&CancellationToken) -> BoxFuture<Result<Arc<B>>> + Send + Sync>,
    // optional: batched_unmarshal, build_block_with_context
}
```

Internally: `verified_blocks: HashMap<Id, BlockWrapper>` (in-consensus),
`decided`/`unverified`/`missing` LRU caches, `bytes_to_id` LRU. The `BlockWrapper`
intercepts `verify`/`accept`/`reject` to move the block between cache tiers and
keep `last_accepted` correct (Go `chain/block.go`). Caches use the `ava-utils`
sized LRU (`03`); sizes come from config.

### 3.4 `gas` — dynamic fee accounting

`vms/components/gas` is the ACP-103 dynamic-fee primitive (also used by SAE, `11`).
Pure integer math (no float, per `00` §6.1):

```rust
pub struct Gas(pub u64);
pub struct Price(pub u64);
pub struct GasState { pub capacity: Gas, pub excess: Gas }   // serialize
pub struct Dimensions(pub [u64; NUM_DIMENSIONS]);            // bandwidth/read/write/compute

impl Gas {
    /// `Gas.Cost(price)` = checked `gas * price`.
    pub fn cost(self, price: Price) -> Result<u64> { safemath::mul(self.0, price.0) }
}
/// `CalculatePrice(min_price, excess, excess_conversion_constant)` — the
/// exponential `min_price * e^(excess/K)` approximated by Go's integer
/// `fakeExponential`. Reproduce the exact fixed-point loop (no float).
pub fn calculate_price(min_price: Price, excess: Gas, k: Gas) -> Price { /* ... */ }
```

`GasState::advance`/`consume` mirror `state.go` capacity refill + excess decay.
This is consensus-affecting where SAE/EVM fee math depends on it — golden-vector
tested (`02`).

---

## 4. The `fx` framework + `secp256k1fx` (`ava-secp256k1fx`)

### 4.1 The `Fx` trait

Go's `fx.Fx` interface (and `secp256k1fx.VM` callback) generalized to Rust. A VM
holds `Vec<Fx>` (`common.Fx{ ID, Fx }`) and calls into them to verify spends. The
fx registers its codec types into the VM's codec registry at `initialize`.

```rust
/// `common.Fx` — an fx instance bound to its ID.
pub struct Fx { pub id: Id, pub fx: Arc<dyn FxInstance> }

/// `vms/fx.Fx` + the verification surface from `secp256k1fx.Fx`.
/// All `*Intf interface{}` params become typed trait objects (`&dyn ...`);
/// the runtime `ok` type-asserts in Go become `downcast_ref` returning
/// `Error::WrongType*` variants.
pub trait FxInstance: Send + Sync {
    /// `Initialize(vm)` — register codec types into the VM's registry,
    /// stash the clock + recover-cache. `vm` is the fx's view of its host VM.
    fn initialize(&mut self, vm: Arc<dyn FxVm>) -> Result<()>;
    fn bootstrapping(&mut self) -> Result<()>;
    fn bootstrapped(&mut self) -> Result<()>;   // enables signature verification

    /// `VerifyTransfer(tx, in, cred, utxo)`.
    fn verify_transfer(&self, tx: &dyn UnsignedTx, input: &dyn Any,
        cred: &dyn Any, utxo: &dyn Any) -> Result<()>;
    /// `VerifyPermission(tx, in, cred, owner)`.
    fn verify_permission(&self, tx: &dyn UnsignedTx, input: &dyn Any,
        cred: &dyn Any, owner: &dyn Any) -> Result<()>;
    /// `VerifyOperation(tx, op, cred, utxos)`.
    fn verify_operation(&self, tx: &dyn UnsignedTx, op: &dyn Any,
        cred: &dyn Any, utxos: &[&dyn Any]) -> Result<()>;
    /// `CreateOutput(amount, owner)`.
    fn create_output(&self, amount: u64, owner: &dyn Any) -> Result<Arc<dyn verify::State>>;
}

/// `secp256k1fx.VM` — the host callbacks an fx needs.
pub trait FxVm: Send + Sync {
    fn codec_registry(&self) -> Arc<CodecRegistry>;   // 03
    fn clock(&self) -> SystemTime;                     // for locktime checks
    fn logger(&self) -> Logger;
}
```

> **Why `&dyn Any`?** Go passes `interface{}` and type-asserts inside the fx
> (`txIntf.(UnsignedTx)`, `inIntf.(*TransferInput)`). The Rust port keeps that
> dynamic boundary with `&dyn Any` + `downcast_ref`, mapping a failed downcast to
> the matching Go sentinel (`ErrWrongTxType`, `ErrWrongInputType`,
> `ErrWrongCredentialType`, `ErrWrongOwnerType`, `ErrWrongUTXOType`,
> `ErrWrongOpType`). This preserves the exact error semantics while letting
> P/X-Chain pass their own tx/input/output concrete types through one fx.

### 4.2 `secp256k1fx` types (byte-exact codec)

Registered into the codec **in this exact order** (typeID = index, per `03`):
`TransferInput`(0), `MintOutput`(1), `TransferOutput`(2), `MintOperation`(3),
`Credential`(4).

```rust
/// `secp256k1fx.OutputOwners`. NOT a `verify::State` (Go `IsNotState`).
pub struct OutputOwners {
    pub locktime: u64,           // serialize
    pub threshold: u32,          // serialize
    pub addrs: Vec<ShortId>,     // serialize — sorted & unique
    // runtime-only ctx for JSON address formatting (not on codec path):
    ctx: Option<Arc<ChainContext>>,
}

/// `secp256k1fx.Input` — the SigIndices common to every spend.
pub struct Input { pub sig_indices: Vec<u32> }   // serialize — sorted & unique
impl Input {
    pub const COST_PER_SIGNATURE: u64 = 1000;
    pub fn cost(&self) -> Result<u64> { safemath::mul(self.sig_indices.len() as u64, Self::COST_PER_SIGNATURE) }
}

/// `secp256k1fx.TransferInput`.
pub struct TransferInput { pub amt: u64 /*serialize*/, pub input: Input /*serialize, embedded*/ }
/// `secp256k1fx.TransferOutput` (a `verify::State`).
pub struct TransferOutput { pub amt: u64 /*serialize*/, pub owners: OutputOwners /*serialize, embedded*/ }
/// `secp256k1fx.MintOutput` (a `verify::State`).
pub struct MintOutput { pub owners: OutputOwners }   // serialize, embedded
/// `secp256k1fx.Credential` — fixed-size 65-byte recoverable sigs.
pub struct Credential { pub sigs: Vec<[u8; 65]> }    // serialize
```

`OutputOwners::verify` reproduces Go exactly: `threshold > len(addrs)` ⇒
`ErrOutputUnspendable`; `threshold == 0 && !addrs.is_empty()` ⇒
`ErrOutputUnoptimized`; `!is_sorted_and_unique(addrs)` ⇒ `ErrAddrsNotSortedUnique`.

### 4.3 `OutputOwners` multisig verification (the heart)

`secp256k1fx.Fx.VerifyCredentials` — reproduced **bit-for-bit** because it gates
spending. Signature recovery uses the `secp256k1` crate (recoverable sigs, `00`
§4.3); the address is `ripemd160(sha256(compressed_pubkey))` (`03`); the message
is `sha256(unsigned_tx_bytes)`.

```rust
impl Fx {
    /// `VerifyCredentials(tx, in, cred, owner)`.
    pub fn verify_credentials(
        &self,
        unsigned_tx: &dyn UnsignedTx,
        input: &Input,
        cred: &Credential,
        owner: &OutputOwners,
    ) -> Result<()> {
        let num_sigs = input.sig_indices.len() as u32;
        // 1. locktime — must be matured against the fx clock.
        if owner.locktime > self.clock_unix() {
            return Err(Error::Timelocked);
        }
        // 2. threshold must equal the number of supplied sig indices, exactly.
        if owner.threshold < num_sigs { return Err(Error::TooManySigners); }
        if owner.threshold > num_sigs { return Err(Error::TooFewSigners); }
        // 3. one signature per sig index.
        if input.sig_indices.len() != cred.sigs.len() {
            return Err(Error::InputCredentialSignersMismatch);
        }
        // 4. during bootstrap, skip signature verification (Go parity).
        if !self.bootstrapped { return Ok(()); }

        let tx_hash = sha256(unsigned_tx.bytes());     // ComputeHash256
        for (i, &index) in input.sig_indices.iter().enumerate() {
            // 5. index must reference an existing owner address.
            if index as usize >= owner.addrs.len() {
                return Err(Error::InputOutputIndexOutOfBounds);
            }
            // 6. recover pubkey from (hash, sig) and check the derived address
            //    matches the owner address at `index`.
            let pk = self.recover_cache.recover(&tx_hash, &cred.sigs[i])?;
            let expected = owner.addrs[index as usize];
            if expected != pk.address() {
                return Err(Error::WrongSig { expected, got: pk.address() });
            }
        }
        Ok(())
    }
}
```

> **Invariant (proptest-guarded, `02`):** because indices are *sorted & unique*
> (`Input::verify`) and `threshold == num_sigs`, each accepted spend is a distinct
> set of `threshold` owner signatures — no signature can be reused for two indices,
> and an under/over-signed input is rejected. The `index` ordering is also tied to
> the `cred.sigs` ordering positionally (`cred.sigs[i]` for `sig_indices[i]`).

`VerifySpend`/`VerifyTransfer` additionally require `utxo.amt == in.amt`
(`ErrMismatchedAmounts`). `verifyOperation` (mint) requires the produced
`MintOutput.owners` to equal the consumed mint UTXO's owners (`ErrWrongMintCreated`)
and then runs `VerifyCredentials` against the mint input.

### 4.4 `nftfx` / `propertyfx` (brief)

Same `FxInstance` shape, registered by the X-Chain (`09`). `nftfx`
(`MintOutput`/`TransferOutput`/`MintOperation`/`TransferOperation`/`Credential`)
carries an NFT payload + group id and reuses `secp256k1fx.OutputOwners` for the
multisig; `propertyfx` (`MintOutput`/`OwnedOutput`/`MintOperation`/`BurnOperation`/
`Credential`) likewise. Both delegate signature checking to the same
`verify_credentials` logic over `OutputOwners`. Detailed in `09`.

---

## 5. `rpcchainvm` — out-of-process gRPC plugin (`ava-vm-rpc`)

A VM may run as a **separate process** speaking gRPC, so the node and the VM can be
different binaries (and different languages). The compatibility contract (`00`
§ table) is **byte-exact `proto/vm` + the avalanchego runtime handshake**: a Rust
plugin must be hostable by a Go node, and a Rust node must host Go plugins.

> **CRITICAL FINDING — it is NOT hashicorp go-plugin.** Modern avalanchego does
> **not** use go-plugin's stdout `magic|core|net|addr|cert` handshake line, magic
> cookie, or mTLS-over-stdout. It uses its **own reverse-dial handshake** over an
> extra gRPC service (`proto/vm/runtime`). The interop surface a Rust impl must
> match is therefore: (a) the env var, (b) the `Runtime.Initialize` RPC, (c) plain
> insecure gRPC over an ephemeral TCP port, (d) the `proto/vm` `VM` service, and
> (e) the proxied callback services. We document go-plugin compatibility only as a
> *legacy* note for old plugins (§5.5).

### 5.1 The handshake (verbatim from `vms/rpcchainvm/runtime/subprocess`)

```
HOST (node, the gRPC client of VM)                 GUEST (plugin, the gRPC server of VM)
──────────────────────────────────                ─────────────────────────────────────
1. bind ephemeral TCP listener R                   (process not yet started)
2. serve Runtime gRPC service on R
3. spawn plugin process with env
   AVALANCHE_VM_RUNTIME_ENGINE_ADDR = R.addr
   (+ forward GRPC_*/GODEBUG env)
   capture child stdout/stderr → log         ───►  4. read env AVALANCHE_VM_RUNTIME_ENGINE_ADDR
                                                    5. bind ephemeral TCP listener V
                                                    6. dial R, call
                                                       Runtime.Initialize(protocol_version=45,
                                                                          addr = V.addr)
7. Initialize handler: assert protocol_version
   == version.RPCChainVMProtocol (45) else
   ErrProtocolVersionMismatch; record V.addr  ◄───
8. handshake channel closed (or HandshakeTimeout
   = 5s ⇒ ErrHandshakeFailed, kill child)
9. dial V; create VM gRPC client                   10. serve VM + grpc.health on V (SERVING)
```

The plugin's `main()` calls `ava_vm_rpc::serve(vm).await` (mirrors Go `rpcchainvm.Serve`):
it reads the env addr, dials the runtime, binds its own listener, calls
`Initialize`, then serves `VM` + health. Graceful shutdown ignores SIGINT/SIGTERM
until the host has signalled shutdown (Go drops signals until `allowShutdown`),
then exits on SIGTERM.

Constants (copy verbatim into `ava-vm-rpc`):

```rust
pub const ENGINE_ADDRESS_KEY: &str = "AVALANCHE_VM_RUNTIME_ENGINE_ADDR";
pub const RPC_CHAIN_VM_PROTOCOL: u32 = 45;          // version/constants.go — bump together
pub const DEFAULT_HANDSHAKE_TIMEOUT: Duration = Duration::from_secs(5);
pub const DEFAULT_GRACEFUL_TIMEOUT: Duration = Duration::from_secs(5);
```

`Runtime.Initialize(protocol_version: u32, addr: String)` is the single
runtime RPC (`proto/vm/runtime`). On Linux the host sets `Pdeathsig = SIGTERM`
(child dies with parent) — in Rust via `nix`/`libc` `prctl(PR_SET_PDEATHSIG)` in a
`pre_exec` closure (the one isolated `unsafe`, `00` §7.6); non-Linux falls back to
kill-on-drop.

### 5.2 The host (VM client) side

`vm_client.go` → `ava-vm-rpc::host::RpcChainVm`, which implements the **full**
`ChainVm` trait (§2) by translating each method to a `proto/vm` RPC over the dialed
channel. Before sending `Initialize`, the host stands up **two** local gRPC
servers and passes their addresses in `InitializeRequest`:

- `db_server_addr` → serves `proto/rpcdb` `Database` (the per-chain VM DB, `04`).
- `server_addr` → serves the callback bundle: `proto/sharedmemory`
  (`SharedMemory`), `proto/aliasreader` (`AliasReader` = `bc_lookup`),
  `proto/appsender` (`AppSender`), `proto/validatorState` (`ValidatorState`,
  `06`), `proto/warp` (`Signer`), plus `grpc.health`.

`InitializeRequest` carries the `ChainContext` identity verbatim: `network_id`,
`subnet_id`, `chain_id`, `node_id`, BLS `public_key`, `x_chain_id`, `c_chain_id`,
`avax_asset_id`, `chain_data_dir`, `genesis_bytes`, `upgrade_bytes`,
`config_bytes`, and `network_upgrades` (JSON-encoded `upgrade::Config`, `03`).
`CreateHandlers`/`NewHTTPHandler` return server addresses that the host dials and
proxies HTTP→gRPC via `proto/http` (`ghttp` — `Handle`/`HandleSimple` streaming
request/response, with `proto/net`/`proto/io` for hijacked conns).

### 5.3 The guest (VM server) side

`vm_server.go` → `ava-vm-rpc::guest::VmServer<V: ChainVm>`: a tonic service
implementing the `proto/vm` `VM` service, delegating to the local `V`. At
`Initialize` it dials back `db_server_addr`/`server_addr` and constructs the
client-side proxies the inner VM consumes:

- `RpcDatabase` (`proto/rpcdb` client) implementing `DynDatabase` (`04`).
- `RpcSharedMemory`, `RpcAliasReader`, `RpcValidatorState`, `RpcWarpSigner`
  implementing the corresponding traits (§3.1, `06`).
- `RpcAppSender` (`proto/appsender` client) implementing `AppSender` (§2.6).

So whichever side is the *plugin* always **dials** the callback services; whichever
side is the *node* always **serves** them. This symmetry is what makes Rust↔Go
interop work in both directions.

### 5.4 Proto → tonic service map

| proto package | Direction | tonic service | Rust trait it bridges |
|---|---|---|---|
| `vm` | host→guest | `VM` | `ChainVm` (+ batched/statesync/withcontext RPCs) |
| `vm/runtime` | guest→host | `Runtime` | handshake `Initialize` |
| `rpcdb` | guest→host | `Database` | `DynDatabase` (`04`) — server-side iterator handles, batched `IteratorNext`, `ErrEnumToError` table |
| `appsender` | guest→host | `AppSender` | `AppSender` (§2.6) |
| `sharedmemory` | guest→host | `SharedMemory` | `SharedMemory` (§3.1) |
| `validatorState` | guest→host | `ValidatorState` | `ValidatorState` (`06`) |
| `warp` | guest→host | `Signer` | `warp::Signer` (`ava-crypto`) |
| `aliasreader` | guest→host | `AliasReader` | `AliaserReader` (`06`) |
| `messenger` | guest→host | `Messenger` | VM→engine `Notify` (legacy; modern path uses `WaitForEvent`) |
| `keystore` | guest→host | `Keystore` | deprecated user keystore (kept for proto parity) |
| `http`,`net`,`io` | host→guest | `HTTP`/`Conn`/`Reader`/`Writer` | proxying VM HTTP handlers (`ghttp`) |
| `grpc.health.v1` | both | `Health` | liveness; host waits for `SERVING` |

All generated by `tonic-build`/`prost` from the **shared `proto/` tree** (`00`
§3) — the same `.proto` files the Go node uses, guaranteeing wire identity. gRPC
options must match: max recv/send message size (the p2p message limit), the
`grpcutils` keepalive/dial defaults, and **insecure** transport (no TLS — the
channel is loopback TCP).

### 5.5 go-plugin legacy note

If interop with *very old* avalanchego plugins (pre-runtime-handshake) is ever
required, a separate `ava-vm-rpc::legacy::go_plugin` shim would implement
hashicorp go-plugin's handshake: read `AppProtoVersion`/`MagicCookieKey`/
`MagicCookieValue` from env, print the handshake line
`CORE-PROTOCOL-VERSION|APP-PROTOCOL-VERSION|network|address|protocol[|cert]` to
stdout, optionally serve mTLS. **This is out of scope for parity** with current
avalanchego (protocol 45) and is documented only so the boundary is understood.

---

## 6. VM middleware — metervm / tracedvm (`ava-vm::middleware`)

`metervm`/`tracedvm` are **decorators**: each wraps a `ChainVm` and *is* a
`ChainVm`, forwarding every call while recording metrics / opening tracing spans.
They compose with proposervm (`06`) — a `ChainVm` middleware — via the chain
pipeline (§8).

```rust
/// `metervm.blockVM`. Wraps a `ChainVm`, times every method into a
/// per-method Prometheus histogram (same metric names/labels as Go — golden
/// tested, 00 §7.3). Probes inner optional capabilities once at construction so
/// the wrapper re-exposes them (`as_batched`, etc.).
pub struct MeterVm<V: ChainVm> { inner: V, metrics: BlockMetrics }

/// `tracedvm.blockVM`. Wraps a `ChainVm`, opening an OTel span per method
/// (must `.End()` — `spancheck`/Drop guard, 00 §4.6). Span/attr names mirror Go.
pub struct TracedVm<V: ChainVm> { inner: V, name: String, tracer: Tracer }
```

Both implement `ChainVm`/`Vm`/`AppHandler`/`HealthCheck`/`Connector` by delegating.
Because Go uses interface embedding + type-assertion to re-expose `BatchedChainVM`/
`StateSyncableVM`/`BuildBlockWithContextChainVM`, the Rust wrappers must forward the
`as_batched`/`as_state_syncable`/`as_build_with_context` probes to the inner VM
(wrapping the returned trait object with the same metering/tracing) — otherwise a
wrapped proposervm would lose its batched/statesync capability. Avalanche-engine
(DAG) `vertexVM` variants exist too (`NewVertexVM`) but are legacy (`06` §1).

---

## 7. Mempool (`ava-vm::mempool`)

`vms/txs/mempool.Mempool[T]` → a generic `Mempool<T: MempoolTx>`. FIFO-ordered
(`Peek` = oldest), bounded by total bytes, dedups by txID, tracks drop reasons,
and wakes the builder via `wait_for_event`:

```rust
pub trait MempoolTx: Send + Sync { fn id(&self) -> Id; fn size(&self) -> usize;
    fn inputs(&self) -> &[Id]; }   // inputs for conflict removal

#[async_trait::async_trait]
pub trait Mempool<T: MempoolTx>: Send + Sync {
    fn add(&self, tx: T) -> Result<()>;
    fn get(&self, id: Id) -> Option<T>;
    fn remove(&self, txs: &[T]);                 // + conflicts
    fn peek(&self) -> Option<T>;                 // oldest
    fn iterate(&self, f: &mut dyn FnMut(&T) -> bool);
    fn mark_dropped(&self, id: Id, reason: Error);
    fn get_drop_reason(&self, id: Id) -> Option<Error>;
    fn len(&self) -> usize;
    async fn wait_for_event(&self, token: &CancellationToken) -> Result<VmEvent>; // PendingTxs
}
```

Backed by an insertion-ordered map (`indexmap`, `00` §4.7) + a per-input index for
conflict removal + a dropped-reason LRU. `wait_for_event` is a `tokio::sync::Notify`
that fires when the pool becomes non-empty (Go uses a channel). Shared by P-Chain,
X-Chain, and SAE C-Chain mempools (`08`/`09`/`11`).

---

## 8. `ava-chains` — the chain manager

### 8.1 Registry / `VmManager` / `Factory`

```rust
/// `vms.Factory`. Creates VM instances.
pub trait Factory: Send + Sync {
    fn new(&self, log: Logger) -> Result<Box<dyn Any + Send>>;  // returns a ChainVm (or legacy DAG VM)
}

/// `vms.Manager`. Registers factories by VM ID, tracks aliases + versions.
/// Embeds the `Aliaser` (03/06) so `vmID.String()` is always an alias.
pub struct VmManager { /* RwLock<{factories: HashMap<Id, Arc<dyn Factory>>, versions: HashMap<Id, String>}>, aliaser */ }
impl VmManager {
    pub fn get_factory(&self, vm_id: Id) -> Result<Arc<dyn Factory>>;  // Err(NotFound)
    pub async fn register_factory(&self, token: &CancellationToken,
        vm_id: Id, f: Arc<dyn Factory>) -> Result<()>;  // dup ⇒ Err; probes Version then Shutdown
    pub fn list_factories(&self) -> Vec<Id>;
    pub fn versions(&self) -> HashMap<String, String>;  // primaryAlias -> version
}

/// `vms/registry.VMRegistry` — installs plugin VMs discovered on disk.
pub struct VmRegistry { getter: Box<dyn VmGetter>, manager: Arc<VmManager> }
impl VmRegistry {
    /// `Reload` — register all not-yet-installed VMs; returns (installed, failed).
    pub async fn reload(&self, token: &CancellationToken) -> Result<(Vec<Id>, HashMap<Id, Error>)>;
}
```

The built-in VMs (platformvm/avm/evm/saevm) register static factories; plugin VMs
are discovered by `VmGetter` (scans the plugin dir, builds an `rpcchainvm` host
factory per binary — `12`).

### 8.2 The chain-creation pipeline (`createSnowmanChain`)

The single most important wiring in the node. Reproduce the exact construction
*and ordering*:

```rust
async fn create_snowman_chain(
    m: &ChainManager,
    ctx: Arc<ConsensusContext>,      // built in build_chain (see below)
    genesis: &[u8],
    vdrs: Arc<dyn ValidatorManager>,
    beacons: Arc<dyn ValidatorManager>,
    mut vm: Box<dyn ChainVm>,        // from factory.new()
    fxs: Vec<Fx>,
    sb: Subnet,
) -> Result<Chain> {
    let alias = m.primary_alias_or_default(ctx.chain.chain_id);

    // 1. DB stack:  base → meterdb → prefixdb(chainID) → { prefix(VM), prefix(bootstrapping) }
    let meter_db = MeterDb::new(m.metrics_for(&alias), m.db.clone())?;          // 04
    let prefix_db = PrefixDb::new(ctx.chain.chain_id.as_bytes(), meter_db);     // 04
    let vm_db: Arc<dyn DynDatabase> = Arc::new(PrefixDb::new(VM_DB_PREFIX, prefix_db.clone()));
    let bootstrapping_db          = PrefixDb::new(BOOTSTRAPPING_DB_PREFIX, prefix_db);

    // 2. Outbound message sender (the engine `Sender`, 06 §5.3).
    let mut sender: Arc<dyn Sender> = OutboundSender::new(ctx.clone(), m.msg_creator.clone(),
        m.net.clone(), m.router.clone(), m.timeout_manager.clone(), EngineType::Snowman, sb.clone())?;
    if m.tracing_enabled { sender = TracedSender::new(sender, m.tracer.clone()); }

    // 3. If this is the P-Chain, the VM *is* the ValidatorState; wire it up
    //    (cached → no-op-if-sybil-disabled) and unblock chain creation on bootstrap.
    //    (See 06 §6 / 08 — the P-Chain bootstraps the validator manager.)

    // 4. VM wrapping order (OUTERMOST last). MUST match Go:
    //      inner VM
    //   -> tracedvm        (if tracing)            "primaryAlias"
    //   -> proposervm      (always)                06 §7
    //   -> metervm         (if MeterVMEnabled)
    //   -> tracedvm        (if tracing)            "proposervm"
    //   -> ChangeNotifier  (always)               wakes engine on VM events
    if m.tracing_enabled { vm = Box::new(TracedVm::new(vm, alias.clone(), m.tracer.clone())); }
    vm = Box::new(ProposerVm::new(vm, m.proposervm_config(&ctx, &sb)));        // 06 §7
    if m.meter_vm_enabled { vm = Box::new(MeterVm::new(vm, m.metrics_for(&alias))); }
    if m.tracing_enabled { vm = Box::new(TracedVm::new(vm, "proposervm".into(), m.tracer.clone())); }
    let change_notifier = Arc::new(ChangeNotifier::new(vm));   // wraps + a notify channel
    let vm: Arc<dyn ChainVm> = change_notifier.clone();

    // 5. Initialize the (fully wrapped) VM with the per-chain ChainContext, DB,
    //    chain-config upgrade/config bytes, fxs, and the AppSender derived from `sender`.
    vm.initialize(token, ctx.chain.clone(), vm_db, genesis,
                  &chain_config.upgrade, &chain_config.config, fxs, sender.app_sender()).await?;

    // 6. Build the consensus instance (Topological), the Snowman engine, the
    //    bootstrapper, and (if supported) the state-syncer — all sharing `ctx`,
    //    `sender`, `vdrs`, the consensus `Parameters` from the subnet config.
    let consensus = Topological::new(/* factory, params */);
    let engines   = build_snowman_engines(ctx.clone(), vm.clone(), sender, vdrs,
                                          beacons, consensus, bootstrapping_db, &sb)?; // 06 §4
    // 7. Per-chain Handler actor (drains the mpsc queue, routes by EngineState/Type,
    //    forwards VM events from change_notifier.wait_for_event → engine.notify). 06 §5.2
    let handler = ChainHandler::new(ctx.clone(), engines, change_notifier.wait_for_event_fn(),
                                    vdrs, m.resource_tracker.clone(), sb, /* trackers */)?;

    // 8. Register chain with the timeout manager + router (06 §5.1).
    m.timeout_manager.register_chain(&ctx)?;
    m.router.add_chain(Arc::new(handler.clone()));
    Ok(Chain { ctx, vm, handler })
}
```

`build_chain` (the dispatcher) creates the per-chain `chain_data_dir`, the per-chain
`Logger`, and the `ConsensusContext`/`ChainContext` (`06` §3) populated with
`network_id`/`subnet_id`/`chain_id`/`node_id`/BLS key, the chain aliases
(`x/c/avax`), `shared_memory = atomic.new_shared_memory(chain_id)`,
`bc_lookup = self` (the aliaser), `warp_signer`, and `validator_state`. It then
`get_factory(vm_id).new()`, builds the `Vec<Fx>` from `fx_ids`, and dispatches to
`create_snowman_chain` (or the legacy `create_avalanche_chain` for DAG VMs).
Finally it registers the chain's metrics gatherers under `primaryAlias`.

`ChainParameters` (Go `chains.ChainParameters`) carries
`{id, subnet_id, genesis_data, vm_id, fx_ids, custom_beacons}`. The P-Chain is
special-cased: it must be created first (it initializes the shared
`validator_state`), and creating a non-P-Chain with the P-Chain VM ID is an error
(`errCreatePlatformVM`).

### 8.3 Aliasing & subnets

`ava-chains` embeds the `Aliaser` (`ids.Aliaser`, `03`/`06`): bidirectional
`alias ↔ chainID` with `primary_alias` (the canonical, e.g. `"X"`/`"C"`/`"P"`) used
in metrics namespaces, log file names, and the `/ext/bc/<alias>` API routes. It
also implements `AliaserReader` (`bc_lookup` in `ChainContext`). Subnet support
(`subnets.Subnet`): each chain belongs to a subnet that owns the consensus
`Parameters` (Snowball K/alpha/beta or Simplex params, `06`), an allowed-nodes
ACL (`Handler::should_handle`, `06` §5.1), and per-subnet config. The `Manager`
tracks subnet membership, gates chain creation on subnet validation, and supports
the special primary network (`PRIMARY_NETWORK_ID = Id::EMPTY`, `03`).

---

## 9. Error model

`ava-vm`/`ava-chains`/`ava-secp256k1fx` each define a `thiserror` `Error`
(`00` §7.1). Preserved Go sentinels (asserted via `matches!`, mirroring `errors.Is`):

- `Error::NotFound` ⇐ `database.ErrNotFound` (block/height/summary lookups).
- `Error::RemoteVmNotImplemented` ⇐ `ErrRemoteVMNotImplemented` (batched fallback).
- `Error::StateSyncableVmNotImplemented`, `Error::ProtocolVersionMismatch`,
  `Error::HandshakeFailed`, `Error::ProcessNotFound` (rpcchainvm).
- fx: `WrongVmType`, `WrongTxType`, `WrongInputType`, `WrongCredentialType`,
  `WrongOwnerType`, `WrongUtxoType`, `WrongOpType`, `MismatchedAmounts`,
  `WrongNumberOfUtxos`, `WrongMintCreated`, `Timelocked`, `TooManySigners`,
  `TooFewSigners`, `InputOutputIndexOutOfBounds`, `InputCredentialSignersMismatch`,
  `WrongSig`, `OutputUnspendable`, `OutputUnoptimized`, `AddrsNotSortedUnique`.
- `AppError` is its own typed error (`i32` code), `Is` by code (§2.2).

`AppError` codes cross the gRPC boundary in `proto/vm`/`proto/appsender` and must
keep Go integer values (`ErrUndefined=0`, `ErrTimeout=-1`).

---

## 10. Test plan (see `02`)

**VM conformance battery (`ava-vm`).** A generic `vm_conformance!(make_vm)` macro
exercising any `ChainVm`: init → genesis last-accepted; `build_block` →
`verify`/`accept` advances `last_accepted` + height index; `parse_block`
round-trips `bytes`; `get_block` of accepted/processing; `Err(NotFound)` for
unknown id/height; `set_preference`; the optional-capability probes; `set_state`
phase transitions; shutdown idempotence. Run against every concrete VM (`08`–`11`)
and against the rpcchainvm host+guest pair.

**rpcchainvm interop (differential, `02`).** Four-way matrix proving wire identity:
Rust-host ⇄ Rust-guest, Rust-host ⇄ **Go-guest**, **Go-host** ⇄ Rust-guest,
Go-host ⇄ Go-guest. Drive an identical block-build/verify/accept sequence through
each; assert identical block bytes, IDs, last-accepted, and `proto/vm` request
bytes (capture + diff). Also test the handshake: protocol-version mismatch ⇒
`ProtocolVersionMismatch`; handshake timeout ⇒ `HandshakeFailed`; the proxied
`rpcdb`/`appsender`/`sharedmemory` round-trips against the Go server.

**fx verification proptests.** Generate random `OutputOwners`
(threshold/addrs/locktime) + signer sets + sig-index permutations; assert
`verify_credentials` accepts iff exactly `threshold` valid owner sigs at sorted
unique indices and locktime matured; reject on over/under-sign, wrong sig, OOB
index, reordered indices, premature locktime. Differential against Go
`secp256k1fx` on shared vectors. `OutputOwners`/`Input` sort invariants
proptested. Codec round-trip golden vectors for all secp256k1fx/avax types
(typeID order — §4.2).

**avax/gas golden vectors.** UTXOID derivation, `FlowChecker` balances, transferable
sort order, `calculate_price` fixed-point output — all diffed against captured Go
outputs (`02`).

**chains pipeline.** Integration test: build a P-Chain + a custom subnet chain via
the `VmManager`/`ChainManager`, assert the DB prefixes, the VM wrapping order
(proposer/meter/trace), handler registration with the router, and alias/API routing.

---

## 11. Performance notes / improvements over Go

- **Zero-copy block bytes.** `Block::bytes()` returns `&[u8]` backed by
  `bytes::Bytes`; the rpcchainvm guest serializes from the same buffer it parsed
  (no re-marshal), and `parse_block` borrows from the channel buffer where the
  codec allows. *Safe because* bytes are immutable post-parse.
- **Parallel signature recovery in fx.** `verify_credentials` over a multi-input tx
  recovers each input's signatures independently. Batch them with `rayon` /
  `spawn_blocking` (`00` §7.2). *Safe because* per-signature verification is pure
  and order-independent; the accept/reject outcome is unchanged (the result is an
  AND-reduction). Guarded by the fx differential test.
- **Sharded mempool.** Replace Go's single-mutex mempool with an insertion-ordered
  structure under a sharded lock for the input-conflict index; `peek` order
  (consensus-visible for build order only, not for accept order) is preserved.
- **rpcchainvm channel reuse + pipelining.** tonic over a single HTTP/2 connection
  multiplexes the VM RPCs and the callback RPCs without head-of-line blocking;
  iterator/`GetAncestors` responses stream. *Safe because* it does not change
  message content or ordering of observable effects.
- **`chain::State` caches as `arc-swap` / sharded LRU** for lock-free
  `get_block`/`last_accepted` reads on the hot path. *Safe because* the data is
  content-addressed (block id ⇒ immutable bytes).

All improvements are gated behind the differential + conformance tests above; any
divergence in block bytes, IDs, accept ordering, or `proto/vm` wire bytes fails CI.
