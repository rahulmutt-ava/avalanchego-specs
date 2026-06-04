# 09 — AVM / X-Chain (`ava-avm`, + `nftfx`, `propertyfx`)

> **Status:** Conforms to `00-overview-and-conventions.md` (binding). Builds on the
> VM framework, the `avax` UTXO/fx components, and `secp256k1fx` defined in
> `07-vm-framework.md`, and mirrors the shared avax-tx patterns the P-Chain
> established in `08-platformvm-pchain.md`. Cross-references: `03-core-primitives.md`
> (ids, codec, crypto), `04-storage-and-databases.md` (Database/versiondb/prefixdb),
> `06-consensus.md` (Snowman block.ChainVM boundary, X-Chain linearization),
> `12-node-config-api-wallet.md` (JSON-RPC serving, genesis, wallet).

## 0. Go source covered

| Go path | What |
|---|---|
| `vms/avm/` | the X-Chain VM (asset exchange, UTXO model) |
| `vms/avm/txs/` | `BaseTx`, `CreateAssetTx`, `OperationTx`, `ImportTx`, `ExportTx`; `Tx` envelope, `Operation`, `InitialState`; codec/parser |
| `vms/avm/txs/executor/` | `SyntacticVerifier`, `SemanticVerifier`, `Executor` (UTXO state transitions), `Backend` |
| `vms/avm/state/` | UTXO set + tx/block/singleton persistence (`state.go`, `diff.go`, `versions.go`) |
| `vms/avm/block/` | linearized Snowman block (`StandardBlock`), `parser.go`; `block/builder`, `block/executor` |
| `vms/avm/network/` | tx gossip (`gossip.go`), `Atomic` app-handler switch, `tx_verifier.go` |
| `vms/avm/utxo/` | `Spender` (UTXO selection for wallet/spend building) |
| `vms/avm/{vm,service,client,wallet_service,wallet_client,genesis,config,factory,health}.go` | VM assembly, JSON-RPC, genesis, config |
| `vms/nftfx/`, `vms/propertyfx/` | the NFT and property feature extensions |
| `vms/secp256k1fx/` | base fx (covered in `07`; referenced here for codec ordering & verification) |
| `vms/components/avax/` | `BaseTx`, `UTXO`, `UTXOID`, `Asset`, `Transferable{In,Out}put`, `UTXOState` (covered in `07`) |

**Crate:** `ava-avm` (single library crate). It pulls `ava-vm` (traits, `avax`
components, `verify`), `ava-secp256k1fx`, `ava-codec`, `ava-types`, `ava-database`,
`ava-network` (gossip), `ava-api` (JSON-RPC serving). `nftfx` and `propertyfx` are
small modules **inside** `ava-avm` (`ava_avm::nftfx`, `ava_avm::propertyfx`) — they
are not separate crates because they exist only to register codec types and fx
verification used by the AVM; this matches Go where they are tiny leaf packages.

---

## 1. Historical note: vertex/DAG → Snowman linearization

The X-Chain originally ran **Avalanche DAG consensus** (vertices containing txs,
`snowstorm` conflict graph). It was **linearized**: a "stop vertex" froze the DAG,
and from that point the chain is a **linear Snowman chain** of `StandardBlock`s. The
Rust port targets the **post-linearization** chain only:

- `ava-avm` implements the Snowman `block.ChainVm` (07), **not** the Avalanche
  `LinearizableVMWithEngine`/`vertex.DAGVM` interfaces. We do **not** port
  `snowstorm`, `vertex`, the avalanche engine, or DAG bootstrap.
- **Still on the wire / on disk (must reproduce byte-exact):** the tx wire format
  (unchanged across linearization), the `StandardBlock` format, the genesis format,
  and the `InitializeChainState(stopVertexID, genesisTimestamp)` step — the genesis
  Snowman block's `Parent()` is the historical **stop vertex ID** (and `Height()==0`).
  Mainnet/Fuji X-Chains were linearized long ago; a syncing node parses the linear
  block history. The stop-vertex ID is a fixed per-network constant carried in
  `ava-genesis`.
- **Deprecated (not ported):** vertex parsing/issuance, `InputUTXOs()` (kept in Go
  as `// TODO: deprecate after x-chain linearization`), DAG conflict detection. We
  keep the data needed to *parse historical state* but never *produce* a vertex.

> **Decision:** `ava-avm` is a pure Snowman VM. Any vertex-era artifact appears only
> as fixed genesis/stop-vertex constants in `ava-genesis`, never as live code.

---

## 2. Codec & type-ID registry (byte-exact — this is the hard contract)

Type IDs in the avalanchego linear codec are assigned **sequentially in registration
order, per codec registry**, as a `uint32` written before each interface value
(see `03-core-primitives.md`). The AVM uses **two** registries built identically
(Go `parser.go` `NewCustomParser`): the **standard codec** `c` (max size
`DefaultMaxSize`) and the **genesis codec** `gc` (max size `MaxInt32`). Both get the
**same types in the same order**, so type IDs are identical between them; only the
max-length limit differs.

### 2.1 Registration order (must match `vms/avm/txs/parser.go` + `block/parser.go` + each fx `Initialize`)

The AVM registers tx types first, then each fx in the order the VM passes them
(`secp256k1fx`, `nftfx`, `propertyfx`), each fx registering its own types in its
`Initialize`, then `block` registers `StandardBlock` last. The resulting **global
type-ID table** (this is load-bearing — wrong IDs = wrong bytes = consensus fork):

| Type ID | Type | Registered by |
|---:|---|---|
| 0 | `BaseTx` | `txs` |
| 1 | `CreateAssetTx` | `txs` |
| 2 | `OperationTx` | `txs` |
| 3 | `ImportTx` | `txs` |
| 4 | `ExportTx` | `txs` |
| 5 | `secp256k1fx::TransferInput` | secp256k1fx |
| 6 | `secp256k1fx::MintOutput` | secp256k1fx |
| 7 | `secp256k1fx::TransferOutput` | secp256k1fx |
| 8 | `secp256k1fx::MintOperation` | secp256k1fx |
| 9 | `secp256k1fx::Credential` | secp256k1fx |
| 10 | `nftfx::MintOutput` | nftfx |
| 11 | `nftfx::TransferOutput` | nftfx |
| 12 | `nftfx::MintOperation` | nftfx |
| 13 | `nftfx::TransferOperation` | nftfx |
| 14 | `nftfx::Credential` | nftfx |
| 15 | `propertyfx::MintOutput` | propertyfx |
| 16 | `propertyfx::OwnedOutput` | propertyfx |
| 17 | `propertyfx::MintOperation` | propertyfx |
| 18 | `propertyfx::BurnOperation` | propertyfx |
| 19 | `propertyfx::Credential` | propertyfx |
| 20 | `block::StandardBlock` | `block` |

> **Invariant CODEC-AVM-1:** The Rust registry MUST produce exactly this table.
> A golden test asserts `type_id(T) == N` for all 21 entries against both registries.
> Codec version is **0** (`CodecVersion = 0`) everywhere on the X-Chain.

### 2.2 Rust representation of the registry

Go uses reflection + a shared `codecRegistry` writing into multiple `linearcodec`
instances, plus a `typeToFxIndex` map (`reflect.Type → fx index`) used by the
verifier to route an output/input/credential/operation to the right fx. In Rust we
replace reflection with an explicit enum-keyed table.

`ava-codec` (03) provides a `TypeId(u32)` newtype and a `CodecRegistry` that maps a
Rust type → its registered `TypeId` for serialization, and `TypeId` → constructor
for deserialization. `ava-avm` builds two `CodecRegistry` instances (standard +
genesis) and, crucially, also builds the `type_id → fx_index` map (`TypeToFxIndex`)
in the **same pass** so the registry and the fx-routing table can never drift:

```rust
/// fx index assigned in VM-registration order. Stable & on-disk-meaningful:
/// it appears in CreateAssetTx::InitialState.fx_index.
#[repr(u32)]
pub enum FxIndex { Secp256k1 = 0, Nft = 1, Property = 2 }
```

---

## 3. Tx model

### 3.1 The signed envelope (`Tx`)

Mirrors `vms/avm/txs/tx.go`. Two `serialize:"true"` fields: the interface-typed
`Unsigned` (so a `TypeId` prefix is written) and `Creds`.

```rust
/// X-Chain signed transaction. Wire layout (codec v0):
///   [unsigned: typeid-prefixed UnsignedTx][creds: []FxCredential]
pub struct Tx {
    pub unsigned: UnsignedTx,         // serialize: true (interface → typeid-prefixed)
    pub creds:    Vec<FxCredential>,  // serialize: true
    // derived, not serialized:
    tx_id: Id,        // = sha256(signed_bytes)
    bytes: Bytes,     // full signed bytes
}
```

`FxCredential` mirrors `fxs/fx.go`: `{ fx_id: Id (serialize:false), credential:
Verifiable (serialize:true → typeid-prefixed) }`. The `fx_id` is **not serialized**;
it is filled in post-parse by looking up the credential's concrete type in
`TypeToFxIndex` and resolving to the fx's asset ID. The credential variants on the
wire are exactly `secp256k1fx::Credential` (id 9), `nftfx::Credential` (id 14),
`propertyfx::Credential` (id 19) — all are `{ sigs: Vec<[u8; 65]> }` structurally
(nft/property embed secp's `Credential`), differing only by type ID.

**Initialization (byte/ID derivation, must match Go `Tx.Initialize`/`SetBytes`):**
1. `signed_bytes = codec.marshal(0, &tx)`.
2. `unsigned_len = codec.size(0, &tx.unsigned)`.
3. `unsigned_bytes = signed_bytes[..unsigned_len]`.
4. `tx_id = sha256(signed_bytes)`; `unsigned.set_bytes(unsigned_bytes)`.

Signature hash is `sha256(unsigned_bytes)` (per-fx `Sign*` helpers). The signed-tx
ID is `sha256(signed_bytes)` — used as both tx ID and gossip ID.

### 3.2 `UnsignedTx` enum

The Go `UnsignedTx` interface + `Visitor` becomes a Rust enum; the visitor methods
(`BaseTx`, `CreateAssetTx`, …) become `match` arms in each verifier/executor. All
embed `avax::BaseTx` (so they share inputs/outputs/memo/network/blockchain id).

```rust
pub enum UnsignedTx {
    /// type id 0 — pure UTXO transfer (and the embedded base of every other tx).
    Base(BaseTx),
    /// type id 1 — defines a new asset + its initial fx state.
    CreateAsset(CreateAssetTx),
    /// type id 2 — fx operations (mint/transfer NFT, mint/burn property, secp mint).
    Operation(OperationTx),
    /// type id 3 — import UTXOs from another chain via shared memory.
    Import(ImportTx),
    /// type id 4 — export UTXOs to another chain via shared memory.
    Export(ExportTx),
}

/// `avax.BaseTx` — the shared body. Byte layout (codec v0), in order:
///   network_id u32 | blockchain_id [32] | outs []TransferableOutput
///   | ins []TransferableInput | memo []u8 (<=256)
pub struct BaseTx {
    pub network_id:    u32,
    pub blockchain_id: Id,
    pub outs: Vec<TransferableOutput>, // sorted, typeid-prefixed Out
    pub ins:  Vec<TransferableInput>,  // sorted, typeid-prefixed In
    pub memo: Vec<u8>,                 // MaxMemoSize = 256
}

/// type id 1
pub struct CreateAssetTx {
    pub base: BaseTx,           // serialize: true (embedded)
    pub name: String,           // 1..=128, ASCII letters/digits/space, no edge ws
    pub symbol: String,         // 1..=4, ASCII uppercase
    pub denomination: u8,       // <= 32
    pub states: Vec<InitialState>, // sorted+unique by fx_index, non-empty
}

/// type id 2
pub struct OperationTx {
    pub base: BaseTx,
    pub ops: Vec<Operation>,    // non-empty, sorted+unique by marshalled bytes
}

/// type id 3
pub struct ImportTx {
    pub base: BaseTx,
    pub source_chain: Id,
    pub imported_ins: Vec<TransferableInput>, // non-empty
}

/// type id 4
pub struct ExportTx {
    pub base: BaseTx,
    pub destination_chain: Id,
    pub exported_outs: Vec<TransferableOutput>, // non-empty, sorted
}
```

> **Field-order invariant TX-AVM-1:** field order above is the serialization order
> and MUST NOT change. `CreateAssetTx`/`OperationTx`/`ImportTx`/`ExportTx` serialize
> the embedded `BaseTx` fields **inline first** (Go struct embedding), then their own
> fields in declared order. The embedded `avax.BaseTx` inside avm's `txs.BaseTx` is
> itself flattened (no extra prefix).

### 3.3 `InitialState` & asset-definition model

`CreateAssetTx` defines an asset whose **asset ID == the CreateAssetTx's tx ID**
(see executor §6). Its `states` declare, per fx, the initial UTXO outputs the asset
starts with:

```rust
/// avm/txs/initial_state.go
pub struct InitialState {
    pub fx_index: u32,            // serialize:true; must be < num_fxs
    pub fx_id: Id,                // serialize:false (derived)
    pub outs: Vec<VerifyState>,   // serialize:true; typeid-prefixed fx outputs, sorted by bytes
}
```

`states` is sorted+unique by `fx_index` (`Compare` = `cmp(fx_index)`); within a
state, `outs` is sorted by marshalled bytes. Each out must verify and be a known fx
output type. `fx_index` is the on-disk-meaningful fx ordinal from §2.2.

### 3.4 `Operation` (the OperationTx body)

```rust
/// avm/txs/operation.go
pub struct Operation {
    pub asset:    Asset,          // serialize:true { id: Id }
    pub utxo_ids: Vec<UTXOID>,    // serialize:true; sorted+unique
    pub fx_id:    Id,             // serialize:false (derived from op concrete type)
    pub op:       FxOperation,    // serialize:true; typeid-prefixed
}
```

`FxOperation` (Go `fxs.FxOperation`) is `Verifiable + ContextInitializable + Coster +
Outs() -> Vec<VerifyState>`. Concrete variants on the wire: `secp256k1fx::Mint` (8),
`nftfx::Mint` (12), `nftfx::Transfer` (13), `propertyfx::Mint` (17),
`propertyfx::Burn` (18). Operations in an `OperationTx` are sorted+unique by their
**marshalled bytes** (`SortOperations`/`IsSortedAndUniqueOperations`).

---

## 4. fx integration (`Fx` dispatch from 07)

The fx framework lives in `ava-vm`/`ava-secp256k1fx` (07). The Go `Fx` interface is
`{ Initialize, Bootstrapping, Bootstrapped, VerifyTransfer(tx,in,cred,utxo),
VerifyOperation(tx,op,cred,utxos) }`. In Rust (07) this is a `trait Fx` with
strongly-typed dispatch; `ava-avm` holds `fxs: Vec<ParsedFx { id: Id, fx: Box<dyn
Fx> }>` and routes via `TypeToFxIndex` (the verifier looks up the concrete output /
input / credential / operation type, gets the fx index, and calls that fx).

`bootstrapped` gating matches Go: **signature verification is disabled until the VM
is bootstrapped** (`VerifyCredentials` returns `Ok(())` early when `!bootstrapped`).
Hold the Rust port to that exact branch.

### 4.1 secp256k1fx (the base — fully in 07; verification summary)

`VerifyCredentials(unsigned_tx, in: &Input, cred: &Credential, out: &OutputOwners)`
(`vms/secp256k1fx/fx.go`), reproduced bit-for-bit:

```rust
fn verify_credentials(
    &self, utx: &dyn UnsignedTxBytes, input: &Input,
    cred: &Credential, out: &OutputOwners, now: u64,
) -> Result<()> {
    let num_sigs = input.sig_indices.len();
    if out.locktime > now              { return Err(Err::Timelocked); }
    if (out.threshold as usize) < num_sigs { return Err(Err::TooManySigners); }
    if (out.threshold as usize) > num_sigs { return Err(Err::TooFewSigners); }
    if num_sigs != cred.sigs.len()     { return Err(Err::SignerCredMismatch); }
    if !self.bootstrapped              { return Ok(()); }   // gate matches Go

    let tx_hash = sha256(utx.bytes());                      // hash of UNSIGNED bytes
    for (i, &index) in input.sig_indices.iter().enumerate() {
        if index as usize >= out.addrs.len() { return Err(Err::IndexOOB); }
        let pk = self.recover_cache.recover(&tx_hash, &cred.sigs[i])?; // recoverable secp
        if out.addrs[index as usize] != pk.address() { return Err(Err::WrongSig); }
    }
    Ok(())
}
```

`VerifyTransfer` checks `utxo.amt == in.amt` then `verify_credentials` against the
`TransferOutput`'s owners. `VerifyOperation` handles exactly one `MintOutput` UTXO,
requires `utxo.owners == op.mint_output.owners`, then verifies against `op.mint_input`.
Note `recover_cache` (Go `secp256k1.RecoverCache`) — a bounded LRU keyed by
`(hash, sig)`; in Rust a `dashmap`/`lru` of cap 256, same default.

### 4.2 nftfx verification

NFT outputs carry a `group_id` (u32) and (for transfer) a `payload` (`<= 1 KiB`).
`VerifyTransfer` is **disallowed** (`errCantTransfer`) — NFTs move only via
operations. `VerifyOperation` requires exactly one input UTXO and dispatches on op
type:

```rust
// nftfx::Fx::verify_operation(tx, op, cred: &nftfx::Credential, utxos: &[Output])
match op {
    NftOp::Mint(op) => {                       // type id 12
        let out: &nftfx::MintOutput = utxos[0].try_into()?; // else WrongUTXOType
        verify_all(&[op, cred, out])?;
        if out.group_id != op.group_id { return Err(WrongUniqueID); }
        secp.verify_credentials(tx, &op.mint_input, &cred.0, &out.owners)
    }
    NftOp::Transfer(op) => {                    // type id 13
        let out: &nftfx::TransferOutput = utxos[0].try_into()?;
        verify_all(&[op, cred, out])?;
        if out.group_id != op.output.group_id { return Err(WrongUniqueID); }
        if out.payload   != op.output.payload  { return Err(WrongBytes); }
        secp.verify_credentials(tx, &op.input, &cred.0, &out.owners)
    }
}
```

`nftfx` types (field order = serialization order):
- `MintOutput` (10): `{ group_id: u32, owners: OutputOwners }`
- `TransferOutput` (11): `{ group_id: u32, payload: Vec<u8> (<=1KiB), owners }`
- `MintOperation` (12): `{ mint_input: secp::Input, group_id: u32, payload, outputs: Vec<OutputOwners> }`; `outs()` synthesizes one `TransferOutput` per owner with the shared `group_id`/`payload`.
- `TransferOperation` (13): `{ input: secp::Input, output: TransferOutput }`; `outs()` = `[output]`.
- `Credential` (14): newtype around `secp::Credential`.

### 4.3 propertyfx verification

Property assets: mint produces a new `MintOutput` (continuing mint authority) plus
an `OwnedOutput`; burn consumes an `OwnedOutput`. `VerifyTransfer` disallowed.

```rust
match op {
    PropOp::Mint(op) => {                       // type id 17
        let out: &propertyfx::MintOutput = utxos[0].try_into()?;
        verify_all(&[op, cred, out])?;
        if !out.owners.eq(&op.mint_output.owners) { return Err(WrongMintOutput); }
        secp.verify_credentials(tx, &op.mint_input, &cred.0, &out.owners)
    }
    PropOp::Burn(op) => {                        // type id 18 (BurnOperation)
        let out: &propertyfx::OwnedOutput = utxos[0].try_into()?;
        verify_all(&[op, cred, out])?;
        secp.verify_credentials(tx, &op.input, &cred.0, &out.owners)
    }
}
```

`propertyfx` types:
- `MintOutput` (15): `{ owners: OutputOwners }`
- `OwnedOutput` (16): `{ owners: OutputOwners }` (structurally identical to MintOutput; distinct type ID is what matters)
- `MintOperation` (17): `{ mint_input: secp::Input, mint_output: MintOutput, owned_output: OwnedOutput }`; `outs()` = `[mint_output, owned_output]`.
- `BurnOperation` (18): `{ input: secp::Input }` (embeds `secp::Input`); `outs()` = `[]` (burns, produces nothing).
- `Credential` (19): newtype around `secp::Credential`.

> **fx-routing invariant FX-AVM-1:** the verifier resolves fx-index by the **concrete
> type** of the routed value (output for `verify_fx_usage`, credential for transfer,
> operation for op). The Rust `try_into()?` on the typed-enum payload reproduces Go's
> `interface{}` type-assert + `errWrong*Type`. `verify_all(&[..])` = Go `verify.All`
> (calls `.verify()` left-to-right, first error wins).

---

## 5. State (`ava_avm::state`)

Mirrors `vms/avm/state/state.go`. A `versiondb` over the chain's base DB with five
`prefixdb`-namespaced sub-stores; the in-memory diff layer holds pending writes until
`commit`. See `04` for `Database`/`VersionDb`/`PrefixDb`.

```
VMDB (versiondb over base)
├─ "utxo"      → UTXOState (avax.UTXOState: id→UTXO + addr-index for GetUTXOs)
├─ "tx"        → tx_id[32] → signed tx bytes (parsed with GENESIS codec on read)
├─ "blockID"   → height(u64 BE) → block_id[32]
├─ "block"     → block_id[32] → block bytes
└─ "singleton" → 0x00 initialized | 0x01 timestamp | 0x02 lastAccepted
```

```rust
pub trait ReadOnlyChain: UtxoGetter {
    fn get_tx(&self, id: Id) -> Result<Arc<Tx>>;
    fn get_block_id_at_height(&self, h: u64) -> Result<Id>;
    fn get_block(&self, id: Id) -> Result<Block>;
    fn get_last_accepted(&self) -> Id;
    fn get_timestamp(&self) -> Time;
}
pub trait Chain: ReadOnlyChain + UtxoAdder + UtxoDeleter {
    fn add_tx(&mut self, tx: Arc<Tx>);
    fn add_block(&mut self, b: Block);
    fn set_last_accepted(&mut self, id: Id);
    fn set_timestamp(&mut self, t: Time);
}
pub trait State: Chain + UtxoReader {
    fn is_initialized(&self) -> Result<bool>;
    fn set_initialized(&mut self) -> Result<()>;
    /// Post-linearization startup hook; seeds genesis Snowman block if no
    /// lastAccepted is stored. stop_vertex_id becomes the genesis block's parent.
    fn initialize_chain_state(&mut self, stop_vertex_id: Id, genesis_ts: Time) -> Result<()>;
    fn abort(&mut self);
    fn commit(&mut self) -> Result<()>;
    fn commit_batch(&mut self) -> Result<Batch>;
    fn checksum(&self) -> Id;   // = UTXOState checksum (optional, trackChecksums)
}
```

### 5.1 UTXO model

The canonical UTXO (from `vms/components/avax`, defined in 07; reproduced here for
clarity since it is the heart of the X-Chain):

```rust
/// avax.UTXO — id = TxID.prefix(OutputIndex) (sha256 of txid||u64(index))
pub struct Utxo {
    pub utxo_id: UtxoId,          // { tx_id: Id, output_index: u32 }
    pub asset:   Asset,           // { id: Id }
    pub out:     VerifyState,     // typeid-prefixed fx output (one of ids 6,7,10,11,15,16)
}
pub struct UtxoId { pub tx_id: Id, pub output_index: u32 }
```

`input_id` = `sha256(tx_id ++ be_u64(output_index))` (Go `UTXOID.InputID`/`Prefix`).
The address index (`UTXOState`, 07) lets `GetUTXOs` enumerate by owner address.

### 5.2 `diff.go` (in-memory layer)

`Diff` is the verifier/executor's working set over a parent `Chain` (07's pattern,
same as P-Chain in 08): tracks modified UTXOs (`Some(utxo)` = added, `None` =
deleted), added txs/blocks, and pending timestamp/lastAccepted. `Apply` flushes into
the parent state; `commit` writes the `versiondb` batch to the base DB. `Abort`
discards the versiondb diff. Map iteration on flush is **sorted** (`BTreeMap`) to
keep writes deterministic (§6.1 of 00).

### 5.3 Persistence details (byte-exact)

- height key = `database.PackUInt64` = 8-byte **big-endian** (04).
- timestamp = Go `database.PutTimestamp` (Unix seconds, codec-packed `u64`).
- txs read back with the **genesis codec** (`ParseGenesisTx`) — unbounded size — since
  some historical/genesis assets exceed the standard max; `ava-avm` must do the same.
- genesis Snowman block: `StandardBlock { parent = stop_vertex_id, height = 0,
  time = genesis_ts, txs = [] }`; committed on first `initialize_chain_state`.

---

## 6. Semantic verification & UTXO state transitions

Three visitors over `UnsignedTx` (a `Backend` carries `Ctx`, `Config`, `Codec`,
`FeeAssetID`, `Fxs`, `TypeToFxIndex`, `Bootstrapped`):

### 6.1 `SyntacticVerify` (`syntactic_verifier.go`) — stateless

Per tx type: verify `avax.BaseTx` (network id, memo ≤ 256, ins/outs sorted &
typed), run `avax::verify_tx(fee, fee_asset, ins_groups, outs_groups, codec)`
(conservation + fee, 08's shared helper), verify every credential, and assert
`num_creds == num_inputs` (where "inputs" includes op-count for OperationTx and
imported-ins for ImportTx). Type-specific extra checks:
- `CreateAssetTx`: name 1..=128 (ASCII letter/digit/space, no leading/trailing ws),
  symbol 1..=4 (ASCII uppercase), `denomination <= 32`, `states` non-empty &
  sorted-unique, each `InitialState.verify(codec, num_fxs)`. Uses `CreateAssetTxFee`.
- `OperationTx`: `ops` non-empty; each op verifies; op `utxo_ids` must not collide
  with base `ins` (`errDoubleSpend`); ops sorted-unique by bytes.
- `ImportTx`: `imported_ins` non-empty; fee computed over `ins ++ imported_ins`.
- `ExportTx`: `exported_outs` non-empty; fee over `outs ++ exported_outs`.

### 6.2 `SemanticVerify` (`semantic_verifier.go`) — against state

- `BaseTx`: for each input, fetch the UTXO from state, check `utxo.asset == in.asset`
  (`errAssetIDMismatch`), resolve fx by credential type, check the asset actually
  enables that fx (`verify_fx_usage`), then `fx.verify_transfer(tx, in, cred, utxo.out)`.
  For each output, resolve fx by output type and `verify_fx_usage`.
- `CreateAssetTx` = `BaseTx` (the new asset's own initial-state outs aren't UTXO-spent
  here; they're produced in execution).
- `OperationTx`: `BaseTx`, then for each op fetch all its input UTXOs (asset must
  match), resolve fx by op type, `verify_fx_usage`, `fx.verify_operation(tx, op, cred,
  utxos)`. Credential index = `len(ins) + op_index`.
  - **Bug-compat quirk (must replicate):** Go skips operation verification when
    `!bootstrapped` **or** when `tx.id == "MkvpJS13eCnEYeYi9B5zuWrU9goG9RBj7nr83U7BjrFV22a12"`
    (a grandfathered mainnet tx). Reproduce this exact string check — it is part of the
    accepted chain history. Encode as a named const `GRANDFATHERED_OPERATION_TX`.
- `ImportTx`: `BaseTx`, then (if bootstrapped) `verify.SameSubnet(source_chain)`, then
  fetch imported UTXO bytes from **shared memory** (`Ctx.SharedMemory.Get(source_chain,
  utxo_ids)`), unmarshal each as `avax.UTXO`, verify transfer. Cred index = `len(ins)+i`.
- `ExportTx`: `BaseTx`, then (if bootstrapped) `verify.SameSubnet(destination_chain)`,
  and `verify_fx_usage` for each exported out.

`verify_fx_usage(fx_index, asset_id)`: load the asset's `CreateAssetTx` from state
(asset_id is its tx id); error `errNotAnAsset` if it isn't one; succeed iff some
`InitialState.fx_index == fx_index`, else `errIncompatibleFx`.

### 6.3 `Executor` (`executor.go`) — apply to `Chain`

```rust
// BaseTx: consume inputs, produce outputs at indices [0..len(outs)).
avax::consume(state, &tx.ins);             // DeleteUTXO per input
avax::produce(state, tx_id, &tx.outs);     // AddUTXO, output_index = i
```

- `CreateAssetTx`: run `BaseTx`, then for each `InitialState` out, `AddUTXO` with
  `tx_id = self_tx_id`, `asset.id = self_tx_id`, **`output_index` continuing from
  `len(outs)`** across all states in order. **The asset ID is the CreateAssetTx ID.**
- `OperationTx`: run `BaseTx`, then for each op delete its input UTXOs and add the
  op's `outs()` (asset = op.asset), `output_index` continuing from `len(outs)`.
- `ImportTx`: run `BaseTx`; record imported `input_id`s in `Inputs` and build
  `AtomicRequests { source_chain: { remove_requests: [input_id…] } }`.
- `ExportTx`: run `BaseTx`; for each exported out, build a UTXO at
  `output_index` continuing from `len(outs)`, marshal it, and emit an atomic
  `Element { key: utxo_input_id, value: utxo_bytes, traits: out.addresses() }` into
  `AtomicRequests { destination_chain: { put_requests: [elem…] } }`.

> **Output-index invariant EXEC-AVM-1:** indices are assigned `outs` first, then
> per-state / per-op / exported outputs in declared order, monotonically. Any
> deviation changes UTXO IDs → state divergence.

---

## 7. Blocks (linearized Snowman)

Only one block type: `StandardBlock` (type ID 20). Byte layout (codec v0), in field
order (`block/standard_block.go`):

```rust
/// type id 20. block_id = sha256(block_bytes).
pub struct StandardBlock {
    pub parent_id: Id,        // serialize:true  ("PrntID")
    pub height:    u64,       // serialize:true  ("Hght"); genesis = 0
    pub time:      u64,       // serialize:true  Unix seconds
    pub merkle_root: Id,      // serialize:true  ("Root") — currently zero/unused on X-Chain
    pub txs: Vec<Tx>,         // serialize:true  ("Transactions")
    // derived:
    block_id: Id, bytes: Bytes,
}
```

Block is serialized **as the `Block` interface** (so a type-ID prefix of 20 is
written): `cm.marshal(0, &(block as &dyn Block))`. `block_id = sha256(bytes)`. On
parse, `initialize(bytes)` recomputes the id and initializes every contained tx.
`timestamp() = Unix(time)`.

`ava-avm` implements the Snowman `Block` trait from 06/07: `parent()`, `height()`,
`bytes()`, `verify()` (syntactic+semantic over a `Diff` on the parent), `accept()`
(commit the diff, apply atomic requests via shared memory, mark txs accepted, advance
lastAccepted/timestamp), `reject()`.

### 7.1 Builder + mempool

- **Mempool** (`avm/txs/mempool` over the generic `vms/txs/mempool`): an
  `IndexedList` of verified, unissued txs keyed by tx id, bounded by total bytes
  (`maxMempoolSize`), with a "dropped" LRU recording why a tx was dropped. Same
  eviction/ordering as Go. In Rust: `indexmap` + byte accounting + `parking_lot`.
- **Builder** (`block/builder`): on `BuildBlock`, drain mempool txs in order,
  re-verify each against a running `Diff` (drop & remember failures), pack until the
  block size cap, and produce a `StandardBlock { parent = last_accepted, height =
  parent.height + 1, time = max(parent.time, now), txs }`. The X-Chain advances
  timestamp monotonically to wall-clock-ish `now` (clamped ≥ parent), mirroring Go.

---

## 8. Network: tx gossip & atomic app-handler

- **Gossip** (`network/gossip.go`): uses the generic p2p **push/pull gossip**
  (`network/p2p/gossip`, see 05) over `Tx` (a `Gossipable` with `gossip_id = tx_id`).
  A `Bloom`-filter-backed `Set` tracks known txs; the VM serves `AppRequest`
  (pull) and `AppGossip` (push) for txs, adding valid ones to the mempool. The
  `tx_verifier.go` wraps semantic verification for the gossip path. Reuse 05's gossip
  machinery; `ava-avm` only supplies the marshaller and verify hook.
- **`Atomic`** (`network/atomic.go`): an `arc-swap`-style switch of the live
  `AppHandler`. Before linearization Go routes app messages to a no-op/handshake
  handler, then swaps in the real `network.Network` once linearized. The Rust port
  keeps an `ArcSwap<dyn AppHandler>`; since we only run post-linearization, it is
  initialized once to the real handler but the indirection is preserved for the
  gRPC-plugin path (07) where linearization timing is observable.

---

## 9. Cross-chain atomic import/export (X↔P, X↔C)

This is the **shared-memory contract** that 07 defines and 08 also consumes; `09`
restates it so the three specs agree:

- **Shared memory** (`chains/atomic`, spec'd in 07): a per-(chainA,chainB) keyed
  store of `Element { key, value, traits }`. An **export** on the source chain issues
  a `PutRequest` (the exported UTXO bytes, keyed by the UTXO's `input_id`, with owner
  addresses as `traits`); an **import** on the destination chain issues a
  `RemoveRequest` for those keys. Both are bundled as `Requests { put_requests,
  remove_requests }` keyed by the **peer chain id**, applied **atomically with the
  block's state commit** (one batch).
- **Byte format:** the exported value is `codec.marshal(0, &avax.UTXO)` using the
  **X-Chain (avm) codec** — but the importing chain (P or C) unmarshals with **its
  own** codec; the `avax.UTXO` + `secp256k1fx` output type IDs must therefore be
  **identical across all three VMs**. (They are, because each VM registers
  `secp256k1fx` after its base tx types; the *output* type IDs differ per VM but the
  serialized UTXO carries the **producing chain's** type IDs and is decoded by a codec
  that registered the same fx types — see 07 for the shared `avax`/`secp256k1fx`
  registration the three VMs must agree on for atomic UTXOs).

  > **CROSS-SPEC DECISION ATOMIC-1 (binding on 07/08/09):** atomic UTXOs are encoded
  > with codec **version 0** and the **`secp256k1fx` output type IDs as registered by
  > the *exporting* VM's codec**. For interop, 07 MUST define the canonical
  > `avax.UTXO` + `secp256k1fx.{TransferOutput,MintOutput}` encoding once, and
  > P/X/C MUST register those types such that a UTXO exported by one decodes on the
  > others. The differential test in §11 pins this with vectors exported X→P, X→C,
  > P→X, C→X.
- **`SameSubnet`** check gates imports/exports to chains in the same subnet (07's
  `verify.SameSubnet`), skipped when `!bootstrapped`.

---

## 10. API (`ava_avm::service`, served via `ava-api` — see 12)

JSON-RPC 2.0 methods on the `avm` endpoint (`/ext/bc/X` or `/ext/bc/<chainID>`),
mirroring `vms/avm/service.go` exactly (names, args, replies, error codes). Served by
`ava-api`'s JSON-RPC router (00 §4.2; details in 12). Methods:

| Method | Purpose |
|---|---|
| `avm.getBlock` / `avm.getBlockByHeight` | block by id / height, formatted (hex/json) |
| `avm.getHeight` | last-accepted height |
| `avm.issueTx` | parse + add a signed tx to mempool, gossip it |
| `avm.getTxStatus` | accepted / processing / unknown (legacy `Status`) |
| `avm.getTx` | tx by id (formatted) |
| `avm.getUTXOs` | paginated UTXOs by address set, incl. cross-chain `sourceChain` |
| `avm.getAssetDescription` | name/symbol/denomination/assetID for an asset |
| `avm.getBalance` | balance of one (address, assetID) |
| `avm.getAllBalances` | all asset balances for an address |
| `avm.getTxFee` | `txFee` and `createAssetTxFee` |

Plus the **wallet** service (`wallet_service.go`): `wallet.issueTx`,
`wallet.send`, `wallet.sendMultiple` — server-side spend building via the
`utxo::Spender` and the deprecated keystore (port behind a feature flag; 12).
Addresses are Bech32 with the chain's HRP and `X-` prefix; assets are CB58/hex ids.
Formatting helpers (`avax.AddressManager`, `formatting/address`) come from `ava-vm`
/ `ava-types` (03/07).

---

## 11. Go → Rust mapping summary

| Go | Rust |
|---|---|
| `txs.UnsignedTx` interface + `Visitor` | `enum UnsignedTx` + `match` in verifiers/executor |
| reflection codec + `typeToFxIndex map[reflect.Type]int` | explicit `CodecRegistry` + `TypeToFxIndex: HashMap<TypeId, FxIndex>` built in one pass |
| `fxs.Fx` (`interface{}` args) | `trait Fx` with typed dispatch (07); `try_into()?` replaces type-asserts |
| `secp256k1.RecoverCache` | `lru`/`dashmap` cache cap 256 behind `ava-crypto` |
| `versiondb` + `prefixdb` + `metercacher`/`lru` | `ava-database::{VersionDb,PrefixDb}` + `ava-utils::lru` + prom metrics |
| `chains/atomic.{Requests,Element}` + `SharedMemory` | 07's `SharedMemory` trait + `Requests`/`Element` types |
| `block.Block` interface, single `StandardBlock` | `enum Block { Standard(StandardBlock) }` (room for future types) impl Snowman `Block` |
| gossip via `network/p2p/gossip` | 05's gossip; avm supplies marshaller + verify hook |
| `mockable.Clock` | injectable `Clock` (07) for tests/determinism |

### Error model

Per-crate `ava_avm::Error` (`thiserror`) with variants mirroring every Go sentinel
(`ErrAssetIdMismatch`, `ErrNotAnAsset`, `ErrIncompatibleFx`, `ErrUnknownFx`,
`ErrWrongNumberOfCredentials`, `ErrDoubleSpend`, `ErrNoImportInputs`,
`ErrNoExportOutputs`, name/symbol/denomination errors, …) plus fx errors re-exported
from `ava-secp256k1fx`/`nftfx`/`propertyfx`. Tests assert on variants where Go uses
`errors.Is` (matches the `ErrorIs` lint posture in 00/02).

---

## 12. Test plan (see `02-testing-strategy.md`)

- **Tx codec golden vectors:** for every tx type and every fx output/operation/credential,
  embed Go-produced hex (from `*_test.go` fixtures) and assert byte-exact
  marshal/unmarshal + `tx_id` (= `sha256(signed_bytes)`). **Type-ID table test** for
  all 21 entries (§2.1) against both standard & genesis registries.
- **fx verification proptests:** `proptest`-generate owners/threshold/sig-index/locktime
  combinations and assert `verify_credentials` accepts iff exactly-threshold valid
  signatures from listed addrs and not timelocked; nft/property group-id/payload
  mismatches rejected with the right error. Reproduce the `!bootstrapped` skip.
- **UTXO state-transition tests:** table-driven over each tx type — given a UTXO set,
  apply `Executor`, assert produced/consumed UTXO IDs and `output_index` assignment
  (EXEC-AVM-1), incl. the multi-state `CreateAssetTx` indexing and asset-id = tx-id.
- **Block round-trip & genesis-id test:** parse/serialize `StandardBlock`; assert the
  Mainnet/Fuji X-Chain genesis block id and the stop-vertex parent match Go
  (`ava-genesis` constants).
- **Atomic differential vs Go:** export a UTXO X→P and X→C, capture the shared-memory
  `Element` bytes, and assert a Go node imports them (and vice-versa) — pins
  ATOMIC-1. Run inside the differential harness (02) against a Go node.
- **Grandfather quirk test:** assert the hardcoded tx id `Mkvp…12a12` bypasses op
  verification exactly as Go does.
- **API conformance:** golden request/response JSON for every `avm.*` method vs Go
  `service_test.go` fixtures.

---

## 13. Performance notes / improvements over Go

- **Parallel credential/signature verification.** Per-input secp256k1 recovery is
  independent; verify a tx's credentials with `rayon` and aggregate the first error
  deterministically. *Safe because* the result (accept/reject + which error) is
  order-independent — the only observable effect is pass/fail, and we still return the
  lowest-index error to match Go's first-failure message. (§6.1, §9 of 00.)
- **Zero-copy parse.** Keep `Tx.bytes`/`Block.bytes` as `bytes::Bytes` slices and
  borrow sub-ranges (`unsigned_bytes`) instead of Go's reslice+copy; the unsigned-hash
  is computed over a borrowed slice.
- **Shared recover-cache across the gossip + builder + verifier** (one `dashmap`),
  vs Go's per-fx `RecoverCache`, cutting duplicate recoveries when a tx is gossiped
  then built into a block.
- **Sorted-by-bytes operations/initial-states:** precompute and cache each value's
  marshalled bytes once during build/parse to avoid the O(n log n · marshal) re-marshal
  Go does in its `Less` comparators (`SortOperations`, `sortState`). Behavior identical
  (same total order); only the constant factor improves.
- **Mempool:** `indexmap` + byte accounting avoids Go's separate `linked.List` +
  map bookkeeping; same eviction order.

All improvements are gated by the differential harness (§12) proving identical
external behavior versus the Go node.
