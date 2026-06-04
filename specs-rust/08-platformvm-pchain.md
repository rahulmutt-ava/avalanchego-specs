# 08 — Platform VM (P-Chain)

> **Status:** Conforms to `00-overview-and-conventions.md` (binding). Cross-refs:
> `03-core-primitives.md` (codec/type_id registry, ids, crypto/BLS), `04-storage-and-databases.md`
> (Database, merkledb/flat KV), `06-consensus.md` (ConsensusContext, `ValidatorState`/
> `ValidatorManager`, uptime, sampling), `07-vm-framework.md` (`ChainVm`/`Block`, avax UTXO
> components, fx/secp256k1fx, mempool). `09-avm-xchain.md` mirrors the shared avax tx patterns
> defined here.

## Go source covered

`vms/platformvm/` in full: `txs/` (+ `txs/executor`, `txs/mempool`, `txs/txheap`, `txs/fee`),
`block/` (+ `block/builder`, `block/executor`), `state/`, `reward/`, `validators/`, `warp/`,
`signer/`, `stakeable/`, `fx/`, `utxo/`, `config/`, `network/`, `genesis/`, `status/`,
`service.go`, `client.go`, `vm.go`, `factory.go`, `metrics/`, `health.go`.

The P-Chain is the **staking & metadata chain**: it owns the validator/delegator sets of every
subnet (incl. the Primary Network), subnets & their blockchains, the AVAX UTXO set on the
platform chain, import/export to other chains, the ACP-77 L1 validator lifecycle, BLS-signed Warp
messages, and — critically — it **serves the `ValidatorState` trait that all of consensus,
proposervm, uptime, and Warp depend on** (`06`).

---

## 1. Crate: `ava-platformvm`

Behind the `block::ChainVm` trait (`07`). Layout mirrors Go subpackages:

```
crates/ava-platformvm/
├── txs/            # UnsignedTx enum, Tx envelope, per-tx structs, executor visitors
│   ├── mod.rs      # codec registry (type_ids), Visitor trait, UnsignedTx
│   ├── executor/   # standard, proposal, atomic executors + staker verification
│   ├── mempool.rs  # gossipable tx mempool
│   ├── fee/        # dynamic-fee (gas) calculator + static (pre-Etna) calculator
│   └── txheap.rs   # time/fee-ordered tx heaps
├── block/          # block enum, builder, block executor (Accept/Verify/Reject), parse
├── state/          # State, Diff, stakers, L1 validators, metadata, diff iterators
├── reward/         # staking reward calculator (exact big-int math)
├── validators/     # Manager: the `ValidatorState` impl (serves 06) + diff windowing
├── warp/           # BLS-signed message production & verification
├── signer.rs       # ProofOfPossession + Empty signer enum
├── stakeable.rs    # LockIn / LockOut time-locked outputs
├── fx.rs           # fx::Owner trait (re-exports ava-secp256k1fx OutputOwners)
├── utxo.rs         # UTXO handler: spend/produce/verify (shared with avm via ava-vm components)
├── config.rs       # Internal/Validators config, dynamic-fee + validator-fee config
├── network.rs      # tx gossip (ava-network p2p Gossip)
├── genesis.rs      # P-Chain genesis build + parse
├── service.rs      # JSON-RPC methods (served via ava-api, 12)
├── client.rs       # JSON-RPC client SDK
└── vm.rs           # PlatformVm: impl block::ChainVm
```

External crates (per `00` §4): `ava-codec`/`ava-codec-derive`, `ava-crypto` (`bls`, `secp256k1`),
`ava-database`, `ava-validators`, `num-bigint::BigUint` for the reward formula (matches Go
`math/big`), `parking_lot`, `tokio`, `thiserror`. **No floats anywhere.**

---

## 2. Transaction model

### 2.1 Codec versions & type-ID registry (PROTOCOL CONSTANTS)

`txs.CodecVersion = 0` is the **only** codec version. Two managers exist:
`Codec` (default max size) and `GenesisCodec` (`MaxInt32` max size, used for oversized genesis
txs and for L1-validator value marshalling). Both register the **same type IDs** — they differ
only in max-slice limits. Mirror in Rust as two `Codec` instances over one `type_id` enum
(`03` §2.3).

The type IDs are assigned by **registration order** with two `SkipRegistrations` gaps. The block
codec (`block/codec.go`) and the tx codec (`txs/codec.go`) **share one numbering space** — the
block codec registers the 5 Apricot block types at 0–4, then the txs types, then the 4 Banff
block types at 29–32. The txs codec reproduces this exactly via `SkipRegistrations(5)` (reserve
block IDs 0–4) and `SkipRegistrations(4)` (reserve Banff-block IDs 29–32).

> **secp256k1fx alignment:** the secp256k1fx types are registered **here** (IDs 5–11) at the
> identical positions the AVM uses, so shared-memory UTXO bytes match across chains
> (`03` §2.3, `07` fx). The `SkipRegistrations(1)` after `TransferInput` and after
> `TransferOutput` reserve the (unused-on-P-Chain) `MintInput`/`MintOutput` slots that the AVM
> fills — **do not** collapse the gap.

**Complete shared `type_id` table** (block + tx combined registry, codec v0):

| type_id | Go type | Rust mapping | Phase |
|---|---|---|---|
| 0 | `ApricotProposalBlock` | `Block::ApricotProposal` | block |
| 1 | `ApricotAbortBlock` | `Block::ApricotAbort` | block |
| 2 | `ApricotCommitBlock` | `Block::ApricotCommit` | block |
| 3 | `ApricotStandardBlock` | `Block::ApricotStandard` | block |
| 4 | `ApricotAtomicBlock` | `Block::ApricotAtomic` | block |
| 5 | `secp256k1fx.TransferInput` | `ava-secp256k1fx::TransferInput` | apricot |
| 6 | *(skip — MintInput)* | reserved gap | apricot |
| 7 | `secp256k1fx.TransferOutput` | `ava-secp256k1fx::TransferOutput` | apricot |
| 8 | *(skip — MintOutput)* | reserved gap | apricot |
| 9 | `secp256k1fx.Credential` | `ava-secp256k1fx::Credential` | apricot |
| 10 | `secp256k1fx.Input` | `ava-secp256k1fx::Input` | apricot |
| 11 | `secp256k1fx.OutputOwners` | `ava-secp256k1fx::OutputOwners` | apricot |
| 12 | `AddValidatorTx` | `UnsignedTx::AddValidator` | apricot |
| 13 | `AddSubnetValidatorTx` | `UnsignedTx::AddSubnetValidator` | apricot |
| 14 | `AddDelegatorTx` | `UnsignedTx::AddDelegator` | apricot |
| 15 | `CreateChainTx` | `UnsignedTx::CreateChain` | apricot |
| 16 | `CreateSubnetTx` | `UnsignedTx::CreateSubnet` | apricot |
| 17 | `ImportTx` | `UnsignedTx::Import` | apricot |
| 18 | `ExportTx` | `UnsignedTx::Export` | apricot |
| 19 | `AdvanceTimeTx` | `UnsignedTx::AdvanceTime` | apricot |
| 20 | `RewardValidatorTx` | `UnsignedTx::RewardValidator` | apricot |
| 21 | `stakeable.LockIn` | `stakeable::LockIn` (Input enum) | apricot |
| 22 | `stakeable.LockOut` | `stakeable::LockOut` (Output enum) | apricot |
| 23 | `RemoveSubnetValidatorTx` | `UnsignedTx::RemoveSubnetValidator` | banff |
| 24 | `TransformSubnetTx` | `UnsignedTx::TransformSubnet` | banff |
| 25 | `AddPermissionlessValidatorTx` | `UnsignedTx::AddPermissionlessValidator` | banff |
| 26 | `AddPermissionlessDelegatorTx` | `UnsignedTx::AddPermissionlessDelegator` | banff |
| 27 | `signer.Empty` | `signer::Empty` | banff |
| 28 | `signer.ProofOfPossession` | `signer::ProofOfPossession` | banff |
| 29 | `BanffProposalBlock` | `Block::BanffProposal` | block |
| 30 | `BanffAbortBlock` | `Block::BanffAbort` | block |
| 31 | `BanffCommitBlock` | `Block::BanffCommit` | block |
| 32 | `BanffStandardBlock` | `Block::BanffStandard` | block |
| 33 | `TransferSubnetOwnershipTx` | `UnsignedTx::TransferSubnetOwnership` | durango |
| 34 | `BaseTx` | `UnsignedTx::Base` | durango |
| 35 | `ConvertSubnetToL1Tx` | `UnsignedTx::ConvertSubnetToL1` | etna (ACP-77) |
| 36 | `RegisterL1ValidatorTx` | `UnsignedTx::RegisterL1Validator` | etna |
| 37 | `SetL1ValidatorWeightTx` | `UnsignedTx::SetL1ValidatorWeight` | etna |
| 38 | `IncreaseL1ValidatorBalanceTx` | `UnsignedTx::IncreaseL1ValidatorBalance` | etna |
| 39 | `DisableL1ValidatorTx` | `UnsignedTx::DisableL1Validator` | etna |
| 40 | `AddAutoRenewedValidatorTx` | `UnsignedTx::AddAutoRenewedValidator` | helicon |
| 41 | `SetAutoRenewedValidatorConfigTx` | `UnsignedTx::SetAutoRenewedValidatorConfig` | helicon |
| 42 | `RewardAutoRenewedValidatorTx` | `UnsignedTx::RewardAutoRenewedValidator` | helicon |

> A golden test (`02`) dumps these 43 ids from the Go codec and asserts the Rust enum
> discriminants match. The `signer.Signer`, `fx.Owner` (= `secp256k1fx.OutputOwners`), and
> `verify.Verifiable` (credentials) interface fields embedded in txs each dispatch through the
> **same** shared registry, so e.g. a credential is `type_id=9`.

### 2.2 `UnsignedTx` enum

`UnsignedTx` is the Go interface registered into the codec; its concrete types become enum
variants. Block IDs are a separate enum (`Block`, §4) but share the numbering space, so the
`UnsignedTx` derive uses explicit `type_id`s with reserved gaps rather than auto-increment.

```rust
/// Go: vms/platformvm/txs/unsigned_tx.go (interface) + per-tx files.
/// type_ids share the block codec space (00 §2.3, 03 §2.3).
#[derive(AvaCodec, Clone, Debug)]
#[codec(type_registry)]
pub enum UnsignedTx {
    #[codec(type_id = 12)] AddValidator(AddValidatorTx),
    #[codec(type_id = 13)] AddSubnetValidator(AddSubnetValidatorTx),
    #[codec(type_id = 14)] AddDelegator(AddDelegatorTx),
    #[codec(type_id = 15)] CreateChain(CreateChainTx),
    #[codec(type_id = 16)] CreateSubnet(CreateSubnetTx),
    #[codec(type_id = 17)] Import(ImportTx),
    #[codec(type_id = 18)] Export(ExportTx),
    #[codec(type_id = 19)] AdvanceTime(AdvanceTimeTx),
    #[codec(type_id = 20)] RewardValidator(RewardValidatorTx),
    #[codec(type_id = 23)] RemoveSubnetValidator(RemoveSubnetValidatorTx),
    #[codec(type_id = 24)] TransformSubnet(TransformSubnetTx),
    #[codec(type_id = 25)] AddPermissionlessValidator(AddPermissionlessValidatorTx),
    #[codec(type_id = 26)] AddPermissionlessDelegator(AddPermissionlessDelegatorTx),
    #[codec(type_id = 33)] TransferSubnetOwnership(TransferSubnetOwnershipTx),
    #[codec(type_id = 34)] Base(BaseTx),
    #[codec(type_id = 35)] ConvertSubnetToL1(ConvertSubnetToL1Tx),
    #[codec(type_id = 36)] RegisterL1Validator(RegisterL1ValidatorTx),
    #[codec(type_id = 37)] SetL1ValidatorWeight(SetL1ValidatorWeightTx),
    #[codec(type_id = 38)] IncreaseL1ValidatorBalance(IncreaseL1ValidatorBalanceTx),
    #[codec(type_id = 39)] DisableL1Validator(DisableL1ValidatorTx),
    #[codec(type_id = 40)] AddAutoRenewedValidator(AddAutoRenewedValidatorTx),
    #[codec(type_id = 41)] SetAutoRenewedValidatorConfig(SetAutoRenewedValidatorConfigTx),
    #[codec(type_id = 42)] RewardAutoRenewedValidator(RewardAutoRenewedValidatorTx),
}

impl UnsignedTx {
    pub fn visit<V: Visitor>(&self, v: &mut V) -> Result<(), V::Error> { /* match dispatch */ }
    pub fn inputs(&self) -> &[TransferableInput];
    pub fn outputs(&self) -> &[TransferableOutput];
    pub fn input_ids(&self) -> BTreeSet<Id>;
    pub fn syntactic_verify(&self, ctx: &SnowContext) -> Result<()>;
}
```

All txs embed `BaseTx` (`avax::BaseTx`: `network_id: u32`, `blockchain_id: Id`,
`outs: Vec<TransferableOutput>`, `ins: Vec<TransferableInput>`, `memo: Vec<u8>`). `SyntacticVerify`
checks: outputs sorted (`avax::is_sorted_transferable_outputs`), inputs sorted & unique, each
in/out verifies, plus tx-specific rules (e.g. stake outputs sorted & summing to `Validator.weight`,
`delegation_shares <= reward::PERCENT_DENOMINATOR = 1_000_000`, BLS signer present iff Primary
Network). `SyntacticallyVerified` is a non-serialized cache field → in Rust a memoized
`OnceCell<()>` (not on the wire).

#### Per-tx field summary (load-bearing fields; see Go for full)

| Tx | Key fields beyond `BaseTx` |
|---|---|
| `AddValidatorTx` *(deprecated, parse-only)* | `Validator{node_id, start, end, wght}`, `stake_outs`, `rewards_owner`, `delegation_shares` |
| `AddDelegatorTx` *(deprecated)* | `Validator`, `stake_outs`, `delegation_rewards_owner` |
| `AddSubnetValidatorTx` *(deprecated)* | `SubnetValidator{Validator, subnet}`, `subnet_auth` |
| `AddPermissionlessValidatorTx` | `Validator`, `subnet`, `signer: Signer`, `stake_outs`, `validator_rewards_owner`, `delegator_rewards_owner`, `delegation_shares: u32` |
| `AddPermissionlessDelegatorTx` | `Validator`, `subnet`, `stake_outs`, `delegation_rewards_owner` |
| `RemoveSubnetValidatorTx` | `node_id`, `subnet`, `subnet_auth` |
| `TransformSubnetTx` *(no-op post-Etna)* | full elastic-subnet params (`asset_id`, rates, min/max stake, …) |
| `CreateSubnetTx` | `owner: fx::Owner` |
| `CreateChainTx` | `subnet`, `chain_name`, `vm_id`, `fx_ids`, `genesis_data`, `subnet_auth` |
| `TransferSubnetOwnershipTx` | `subnet`, `subnet_auth`, `owner` |
| `ImportTx` | `source_chain: Id`, `imported_inputs: Vec<TransferableInput>` |
| `ExportTx` | `destination_chain: Id`, `exported_outputs: Vec<TransferableOutput>` |
| `BaseTx` | (just the avax base — fee burn / transfer) |
| `AdvanceTimeTx` | `time: u64` (proposal; Apricot-only, banned post-Banff) |
| `RewardValidatorTx` | `tx_id: Id` (proposal; staker to reward) |
| `ConvertSubnetToL1Tx` | `subnet`, `chain_id`, `address: Vec<u8>`, `validators: Vec<ConvertSubnetToL1Validator>`, `subnet_auth` |
| `RegisterL1ValidatorTx` | `balance: u64`, `proof_of_possession: [u8;96]`, `message: Vec<u8>` (Warp `RegisterL1Validator`) |
| `SetL1ValidatorWeightTx` | `message: Vec<u8>` (Warp `L1ValidatorWeight`) |
| `IncreaseL1ValidatorBalanceTx` | `validation_id: Id`, `balance: u64` |
| `DisableL1ValidatorTx` | `validation_id: Id`, `disable_auth: Verifiable` |
| `AddAutoRenewedValidatorTx` / `SetAutoRenewedValidatorConfigTx` / `RewardAutoRenewedValidatorTx` | Helicon auto-renew lifecycle (see codec v2 metadata, §3.4) |

### 2.3 Signed `Tx` envelope

```rust
/// Go: vms/platformvm/txs/tx.go
#[derive(AvaCodec, Clone)]
pub struct Tx {
    #[codec] pub unsigned: UnsignedTx,
    #[codec] pub creds: Vec<Credential>,      // verify.Verifiable; secp256k1fx::Credential = type_id 9
    pub tx_id: Id,                            // = sha256(signed_bytes); not serialized
    pub bytes: Bytes,                         // cached signed bytes; not serialized
}
```

`initialize`: marshal whole `Tx` → `signed_bytes`; the unsigned prefix length is
`Codec::size(&unsigned)`; `unsigned_bytes = signed_bytes[..unsigned_len]`; `tx_id =
sha256(signed_bytes)`. Signing: hash = `sha256(unsigned_bytes)`, each signer group produces a
`Credential{sigs: Vec<[u8;65]>}` of recoverable secp256k1 signatures over the hash. **Parsing
must reproduce the prefix-length trick** to recover unsigned bytes for caching.

### 2.4 Executor visitors (semantic verification + state transition)

Three executor entry points, each a Go `txs.Visitor`. Port the visitor as a Rust trait with a
default `ErrWrongTxType` for unsupported variants, dispatched by `UnsignedTx::visit`.

```rust
pub trait Visitor {
    type Error;
    // default impls return ErrWrongTxType; each executor overrides the txs it accepts
    fn add_permissionless_validator(&mut self, tx: &AddPermissionlessValidatorTx) -> Result<(), Self::Error> { Err(wrong_type()) }
    fn import(&mut self, tx: &ImportTx) -> Result<(), Self::Error> { Err(wrong_type()) }
    /* … one method per UnsignedTx variant … */
}
```

- **`StandardTx`** (`standard_tx_executor.go`): non-proposal decision txs (Base, Create*,
  Import/Export, Add/RemovePermissionless*, all L1 txs, TransferSubnetOwnership). Mutates a
  `state::Diff` in place; returns `(consumed_input_ids, atomic_requests, on_accept_fn)`. Atomic
  txs (Import/Export) emit `atomic::Requests` against shared memory of the peer chain.
- **`ProposalTx`** (`proposal_tx_executor.go`): the **oracle** txs `AdvanceTimeTx` (Apricot-only)
  and `RewardValidatorTx`. Produces **two** diffs — `on_commit_state` and `on_abort_state` —
  representing the two branches of the proposal/commit/abort block model (§4.2). Returns whether
  the commit branch is preferred.
- **`AtomicTx`** (`atomic_tx_executor.go`): wraps Import/Export for the Apricot atomic-block path
  (pre-Banff). Post-Banff these are ordinary standard txs inside a `BanffStandardBlock`.

Shared helpers (`staker_tx_verification.go`, `subnet_tx_verification.go`, `state_changes.go`,
`warp_verifier.go`) port to free functions over `&state::Diff`:
- `verify_add_permissionless_validator` / `_delegator`: stake bounds, duration bounds
  (`MaxStakeDuration`), subnet existence, overlap with existing stakers, fee charge, BLS uniqueness
  rules, `MaxFutureStartTime = 24*7*2h` (pre-Durango start-time), `SyncBound = 10s`.
- `verify_subnet_authorization`: resolves the subnet owner (`CreateSubnetTx.owner` or after
  `TransferSubnetOwnershipTx`) and checks the `subnet_auth` credential.
- **Fee charging**: pre-Etna uses static fees from config; **post-Etna uses the dynamic gas-based
  fee** (`txs/fee`, `vms/components/gas`) — see §6.
- `advance_time_to`: moves chain time forward, promoting pending→current stakers whose
  `NextTime <= newTime` (in `Staker::Less` order) and removing expired permissioned subnet
  validators; charges L1 continuous fees and deactivates L1 validators whose balance is exhausted
  (ACP-77). `RewardValidatorTx` pays out the staker's `PotentialReward` (commit) or not (abort),
  updates supply, and restakes for auto-renewed validators (Helicon).

---

## 3. State

### 3.1 The state interfaces

`state::Chain` (read+write surface), `state::Diff` (a layered, in-memory overlay over a parent
`Versions`), and `state::State` (the persisted base). Port the Go interface stack as:

```rust
pub trait Chain: Send {
    fn timestamp(&self) -> SystemTime;                 fn set_timestamp(&mut self, t: SystemTime);
    fn current_supply(&self, subnet: Id) -> Result<u64>; fn set_current_supply(&mut self, subnet: Id, s: u64);
    fn fee_state(&self) -> gas::State;                  fn set_fee_state(&mut self, s: gas::State);
    fn l1_validator_excess(&self) -> gas::Gas;          // ACP-77 validator-fee accumulator
    fn accrued_fees(&self) -> u64;                      fn set_accrued_fees(&mut self, v: u64);
    // UTXOs
    fn get_utxo(&self, id: Id) -> Result<Utxo>;         fn add_utxo(&mut self, u: Utxo);
    fn delete_utxo(&mut self, id: Id);
    // stakers
    fn get_current_validator(&self, subnet: Id, node: NodeId) -> Result<Staker>;
    fn put_current_validator(&mut self, s: Staker) -> Result<()>;
    fn delete_current_validator(&mut self, s: &Staker);
    fn put_pending_validator(&mut self, s: Staker) -> Result<()>;  /* + delegator/iterator variants */
    // L1 (ACP-77)
    fn get_l1_validator(&self, vid: Id) -> Result<L1Validator>;
    fn put_l1_validator(&mut self, v: L1Validator) -> Result<()>;
    fn weight_of_l1_validators(&self, subnet: Id) -> Result<u64>;
    // subnets / chains / supply / reward utxos / subnet owners & managers …
}

pub trait Versions: Send + Sync { fn get_state(&self, block_id: Id) -> Option<Arc<dyn Chain>>; }
```

`Diff` stacks on a `parent_id` resolved through `Versions`; `apply(&self, base: &mut dyn Chain)`
flushes the overlay. This is the **versioned/diff** model the block executor uses: each accepted
block has an associated `Diff`; on `Accept` the diff chain is applied down to `State` and
committed.

### 3.2 Persistence (flat KV, NOT merkledb)

The P-Chain state is stored in **flat, prefixed key/value spaces** over the base `Database`
(`04`) — **not** a merkledb (unlike SAE/Firewood). There is no state Merkle root in P-Chain
consensus; block IDs commit the state transitively. Prefixes (each its own `prefixdb`):
`utxoDB`, `subnetDB`, `subnetOwnerDB`, `subnetManagerDB`/`subnetToL1ConversionDB`, `chainDB`
(subnetID→chains), `txDB` (txID→{tx,status}), `rewardUTXOsDB`, `blockDB`, `blockIDDB`
(height→blockID), `currentValidator/Delegator/SubnetValidator` lists, `pending*` lists,
`l1ValidatorDB` (validationID→bytes), `weightDiffDB`, `pkDiffDB` (BLS-key diffs), `singletonDB`
(timestamp, current supply, last accepted, height, fee state). LRU caches sit in front of each.

Port the **byte-exact key layouts** as protocol-relevant only where shared memory / cross-chain
matters (UTXO bytes); the generic on-disk layout is a migration concern (`00` §4.4 — RocksDB on
disk; importing a Go datadir is a migration path, not in-place read).

### 3.3 The staker model

```rust
/// Go: vms/platformvm/state/staker.go — ordering key for the current & pending btrees.
#[derive(Clone, Debug)]
pub struct Staker {
    pub tx_id: Id,
    pub node_id: NodeId,
    pub public_key: Option<bls::PublicKey>,   // None for non-primary subnet stakers
    pub subnet_id: Id,
    pub weight: u64,
    pub start_time: SystemTime,
    pub end_time: SystemTime,
    pub potential_reward: u64,
    pub next_time: SystemTime,     // pending ⇒ == start_time; current ⇒ == end_time
    pub priority: Priority,        // ties between equal next_time (see priorities.go)
}

impl Ord for Staker {            // Go (*Staker).Less — the btree.LessFunc
    fn cmp(&self, o: &Self) -> Ordering {
        self.next_time.cmp(&o.next_time)
            .then(self.priority.cmp(&o.priority))      // Priority is u8; lower first
            .then(self.tx_id.as_ref().cmp(o.tx_id.as_ref()))   // bytes.Compare
    }
}
```

`Priority` (`txs/priorities.go`) is a `u8` enum, values **1..=11** in this exact order (pending
group first, then current group). The ordering drives time-advancement promotion/removal order:
permissioned subnet validators are removed first (by time), permissionless are removed via
`RewardValidatorTx`. **The discriminant values are protocol-load-bearing** (they tie-break
serialization order and the diff-iterator) — pin them:

```rust
#[repr(u8)]
pub enum Priority {
    PrimaryNetworkDelegatorApricotPending = 1,
    PrimaryNetworkValidatorPending = 2,
    PrimaryNetworkDelegatorBanffPending = 3,
    SubnetPermissionlessValidatorPending = 4,
    SubnetPermissionlessDelegatorPending = 5,
    SubnetPermissionedValidatorPending = 6,
    SubnetPermissionedValidatorCurrent = 7,
    SubnetPermissionlessDelegatorCurrent = 8,
    SubnetPermissionlessValidatorCurrent = 9,
    PrimaryNetworkDelegatorCurrent = 10,
    PrimaryNetworkValidatorCurrent = 11,
}
```

Stakers live in two `BTreeSet<Staker>`-backed sets (current, pending) plus per-(subnet,node)
lookup maps, mirroring `state/stakers.go`. The in-memory `baseStakers`/`diffStakers` structure
becomes an overlay map in `Diff`.

### 3.4 Validator metadata — codec v2 (ACP-236)

`MetadataCodec` is a **separate manager** (`state/metadata_codec.go`) with **three versions**
v0/v1/v2 selected by the value's serialized version tag (`03` §2 version-prefixed records). Fields
are gated by version tag — port via `#[codec(version = N)]` on the struct fields:

```rust
/// Go: vms/platformvm/state/metadata_validator.go (validatorMetadata)
#[derive(AvaCodec)]
pub struct ValidatorMetadata {
    #[codec(version = 0)] pub up_duration: u64,        // ns
    #[codec(version = 0)] pub last_updated: u64,        // unix s
    #[codec(version = 0)] pub potential_reward: u64,
    #[codec(version = 0)] pub potential_delegatee_reward: u64,
    #[codec(version = 1)] pub staker_start_time: u64,
    // ACP-236 / Helicon auto-renew (codec v2):
    #[codec(version = 2)] pub accrued_validation_rewards: u64,
    #[codec(version = 2)] pub accrued_delegatee_rewards: u64,
    #[codec(version = 2)] pub auto_compound_reward_shares: u32,
    #[codec(version = 2)] pub next_period: u64,
    #[codec(version = 2)] pub staker_end_time: u64,
    // not serialized:
    pub tx_id: Id,
}
```

`parse_validator_metadata` must reproduce the **length-based legacy fallbacks**: 0 bytes (nil),
8 bytes (potential reward only), `VERSION_SIZE + 3*8` bytes (`preDelegateeRewardMetadata`), else
full codec decode. Auto-renewed validators compute effective weight as
`tx.weight + accrued_validation_rewards + accrued_delegatee_rewards`.

`L1Validator` (`state/l1_validator.go`, ACP-77) is marshalled with **`block::GenesisCodec`** (not
MetadataCodec) keyed by `ValidationID`; ordering (`Compare`) is by `(end_accumulated_fee,
validation_id)` — an `IsActive` (`weight!=0 && end_accumulated_fee!=0`) validator participates in
the active iterator used to charge continuous fees in `EndAccumulatedFee` order. Port the struct
verbatim (see Go for the 9 serialized fields); `immutable_fields_are_unmodified` guards mutation.

### 3.5 Supply & reward state

Per-subnet current supply (singleton key for Primary Network seeded from genesis
`InitialSupply`). `RewardValidatorTx` mints `PotentialReward` into supply on commit; reward UTXOs
are written to `rewardUTXOsDB` keyed by the staker's tx ID and surfaced via `GetRewardUTXOs`.

---

## 4. Blocks

### 4.1 Block enum & byte-exact codec

`Block` is the registered interface; variants & type IDs per §2.1 (0–4 Apricot, 29–32 Banff).
`CommonBlock` = `{ parent_id: Id (PrntID), height: u64 (Hght) }`. Banff blocks add a `time: u64`
prefix and (for standard/proposal) a `Vec<Tx>`.

```rust
#[derive(AvaCodec, Clone)]
#[codec(type_registry)]
pub enum Block {
    #[codec(type_id = 0)]  ApricotProposal(ApricotProposalBlock), // CommonBlock + Tx
    #[codec(type_id = 1)]  ApricotAbort(ApricotAbortBlock),       // CommonBlock
    #[codec(type_id = 2)]  ApricotCommit(ApricotCommitBlock),     // CommonBlock
    #[codec(type_id = 3)]  ApricotStandard(ApricotStandardBlock), // CommonBlock + Vec<Tx>
    #[codec(type_id = 4)]  ApricotAtomic(ApricotAtomicBlock),     // CommonBlock + Tx
    #[codec(type_id = 29)] BanffProposal(BanffProposalBlock),     // time + Vec<Tx>(decision) + ApricotProposal(Tx)
    #[codec(type_id = 30)] BanffAbort(BanffAbortBlock),           // time + ApricotAbort
    #[codec(type_id = 31)] BanffCommit(BanffCommitBlock),         // time + ApricotCommit
    #[codec(type_id = 32)] BanffStandard(BanffStandardBlock),     // time + Vec<Tx>
}
```

`BanffProposalBlock` embeds the Apricot proposal (a single proposal `Tx`) plus a leading
`Vec<Tx>` of decision txs and a `time`; its `Txs()` returns `decision_txs ++ [proposal_tx]`.
**Field order is byte-exact** (Banff fields precede the embedded Apricot struct in the Go struct
literal: `time`, then `transactions`, then the embedded `ApricotProposalBlock`). The block ID is
`sha256` of the codec bytes. `block/parse.go` → `Block::parse(codec, bytes)`.

### 4.2 The proposal/commit/abort oracle model

The P-Chain is the only VM using Snowman **oracle** blocks (`06`). A `*ProposalBlock` is an
oracle: on verify it produces **two child options** — a `*CommitBlock` and an `*AbortBlock` (same
parent = the proposal). The proposal executor (`ProposalTx`, §2.4) precomputes `on_commit_state`
and `on_abort_state`. Consensus decides which child is accepted; the corresponding diff is the one
applied. Post-Banff, the *only* proposal tx produced is `RewardValidatorTx` (issued automatically
when a staker's end time is reached); `AdvanceTimeTx` is forbidden (time advances via the Banff
block timestamp instead). `block/executor/` ports `Verify`/`Accept`/`Reject`/`Options` and the
acceptor that flushes diffs and notifies `validators::Manager::on_accepted_block_id`.

### 4.3 Block builder + mempool

`txs/mempool` is a `gossip::Gossipable` tx pool (FIFO + drop-on-full, deduped by tx ID), shared
with `network.rs` (gossip via `ava-network` p2p, `05`). `block/builder/builder.go` builds the next
block:

1. If the tip is a proposal needing options → build commit/abort.
2. If a staker's `next_time` has arrived → build a `BanffProposalBlock` carrying a
   `RewardValidatorTx` proposal (+ any ready decision txs).
3. Else if mempool non-empty or time must advance → build a `BanffStandardBlock` with decision
   txs ordered by the mempool, capped by block size and (post-Etna) gas.
4. Else → no block (`ErrNoPendingBlocks`).

The builder advances chain time to `min(now, next_staker_change_time)` (clamped by `SyncBound`).

---

## 5. Reward formula (exact)

`reward/calculator.go`. **`PercentDenominator = 1_000_000`**; `consumptionRateDenominator =
1_000_000`. Config: `MaxConsumptionRate`, `MinConsumptionRate` (per year, scaled by
PercentDenominator), `MintingPeriod` (Duration), `SupplyCap`. All math is **`big.Int`** → Rust
`num_bigint::BigUint` to match exactly (intermediate products overflow u128).

```rust
/// Reward = RemainingSupply * adjustedConsumptionRate * stakedAmount * stakedDuration
///          / (mintingPeriod * consumptionRateDenominator) / currentSupply / mintingPeriod
pub fn calculate(&self, staked_duration: Duration, staked_amount: u64, current_supply: u64) -> u64 {
    let d  = BigUint::from(staked_duration.as_nanos() as u64); // Go uses int64 ns of Duration
    let amt = BigUint::from(staked_amount);
    let cur = BigUint::from(current_supply);
    // adjustedConsumptionRateNumerator = maxSubMin*d + minRate*mintingPeriod
    let acr_num = &self.max_sub_min_rate * &d + &self.min_rate * &self.minting_period;
    let acr_den = &self.minting_period * &self.consumption_rate_denominator; // = mintingPeriod * 1e6
    let remaining = self.supply_cap - current_supply;                        // u64 (cap >= supply invariant)
    let mut reward = BigUint::from(remaining);
    reward = reward * acr_num * amt * d / acr_den / cur / &self.minting_period;
    match reward.to_u64() { Some(r) => r.min(remaining), None => remaining } // !IsUint64 ⇒ clamp
}
```

`Split(total, shares)` (delegator/validator fee split) ports exactly, including the
"delay-rounding" optimization: try `shares * total / 1e6` via checked mul; on overflow fall back to
`(PercentDenominator - shares) * (total / PercentDenominator)`. **Reward math has zero tolerance**
— differential + proptest against Go (§9).

---

## 6. Dynamic fees (gas) — Etna / ACP-103

Post-Etna, fees are charged by a **gas meter** (`vms/components/gas`, `txs/fee`). A `gas::State`
(`capacity`, `excess`) is part of P-Chain state; each block updates excess from gas consumed,
and the per-byte/per-credential/per-complexity gas price is an exponential function of excess
(EIP-1559-style). The `fee::Calculator` has two impls: a **static** one (pre-Etna constants from
config) and a **dynamic** one (gas × price). Tx complexity is computed per tx type
(`txs/fee/complexity.go`). Port the gas arithmetic with `u64` checked ops (no float; matches
`gas` package). The **validator fee** (`validators/fee`) is a *separate* gas dimension tracking
the ACP-77 continuous fee charged to active L1 validators against their `EndAccumulatedFee`.

### ACP-77 L1 validator lifecycle

`ConvertSubnetToL1Tx` converts a permissioned subnet into an L1 with an off-chain manager:
existing permissioned validators are removed and the tx's `Validators` become L1 validators.
Then: `RegisterL1ValidatorTx` (adds an L1 validator, funding `balance` → `EndAccumulatedFee`, via
a signed Warp `RegisterL1Validator` message + PoP), `SetL1ValidatorWeightTx` (updates weight via a
Warp `L1ValidatorWeight` message with a monotonic `nonce` ≥ `MinNonce`; weight 0 removes — but
cannot remove the last validator), `IncreaseL1ValidatorBalanceTx` (tops up balance),
`DisableL1ValidatorTx` (owner deactivates, refunding remaining balance to
`RemainingBalanceOwner`). Time-advancement charges each **active** L1 validator the continuous fee
in `EndAccumulatedFee` order; exhausted validators are **deactivated** (weight kept, becomes
inactive). All L1 mutations go through `Diff::put_l1_validator` with the immutable-field guard.

---

## 7. Validator set — serving `ValidatorState` (the contract 06 consumes)

`vms/platformvm/validators/manager.go` is the **single implementation** of the
`ValidatorState` trait declared in `06`. It is the most cross-cutting deliverable of this crate:
proposervm windowing, Warp verification, uptime/reward, and the snow engine all read it.

```rust
/// Go: vms/platformvm/validators.Manager — impl of `ava_validators::ValidatorState` (06 §6.1).
pub struct PChainValidatorManager {
    cfg: config::Internal,                 // TrackedSubnets, UseCurrentHeight, the in-mem Validators set
    state: Arc<State>,
    metrics: Metrics,
    clk: Clock,
    caches: Mutex<HashMap<Id, Lru<u64, BTreeMap<NodeId, GetValidatorOutput>>>>, // per-subnet, size 64
    recently_accepted: Window<Id>,         // 06 sliding window: MaxSize 64, TTL 30s
}

#[async_trait::async_trait]
impl ValidatorState for PChainValidatorManager {
    async fn get_minimum_height(&self) -> Result<u64> {
        if self.cfg.use_current_height { return self.current_height(); }
        match self.recently_accepted.oldest() {
            Some(oldest) => Ok(self.state.get_block(oldest)?.height() - 1), // parent of oldest in window
            None => self.current_height(),
        }
    }
    async fn get_current_height(&self) -> Result<u64> { self.current_height() } // height of last accepted block

    async fn get_subnet_id(&self, chain: Id) -> Result<Id> {
        if chain == PLATFORM_CHAIN_ID { return Ok(PRIMARY_NETWORK_ID); }
        match self.state.get_tx(chain)?.unsigned { UnsignedTx::CreateChain(c) => Ok(c.subnet), _ => Err(..) }
    }

    /// Reconstruct the set at `target_height` by walking diffs backward from current.
    async fn get_validator_set(&self, target_height: u64, subnet: Id)
        -> Result<BTreeMap<NodeId, GetValidatorOutput>>
    {
        if let Some(hit) = self.cache_get(subnet, target_height) { return Ok(hit); }
        let (mut set, current_height) = self.current_validator_set(subnet)?; // from in-mem cfg.Validators
        if current_height < target_height { return Err(ErrUnfinalizedHeight); }
        // apply diffs over (target_height, current_height]  ==  [target_height+1, current_height]
        self.state.apply_validator_weight_diffs(&mut set, current_height, target_height + 1, subnet)?;
        self.state.apply_validator_public_key_diffs(&mut set, current_height, target_height + 1, subnet)?;
        self.cache_put(subnet, target_height, &set);
        Ok(set) // BTreeMap ⇒ canonical NodeId order (06 determinism rule)
    }

    async fn get_current_validator_set(&self, subnet: Id)
        -> Result<(BTreeMap<Id, GetCurrentValidatorOutput>, u64)>
    {
        // base stakers (IsL1Validator=false, MinNonce=0, IsActive=true) keyed by tx_id
        // + L1 validators keyed by validation_id (IsActive from weight/fee), height = current
    }

    async fn get_warp_validator_sets(&self, height: u64) -> Result<HashMap<Id, WarpSet>> {
        // build all subnet sets at `height`, flatten each via FlattenValidatorSet (BLS-key dedup);
        // skip subnets that fail to flatten (disallows warp from them)
    }
}
```

### 7.1 Validator-diff windowing (the reconstruction algorithm)

This is the load-bearing detail. Current validator weights & BLS keys are held **in memory**
(`cfg.Validators`, the `ava-validators::Manager` from `06`) at the **current** (last-accepted)
height. Historical sets are reconstructed by **applying weight diffs and public-key diffs
backward**:

- **Weight diffs** (`state/disk_staker_diff_iterator.go`): keyed
  `[subnetID(32)] ++ [inverseHeight: u64 BE] ++ [nodeID(20)]`; value `[isNegative: bool] ++
  [weight: u64 BE]`. `inverseHeight = MaxU64 - height` so iteration in key order walks **newest
  height first** — exactly what backward reconstruction needs. There is a parallel "by height"
  index (`[inverseHeight] ++ [subnetID] ++ [nodeID]`) for the all-subnets path.
- **Public-key diffs** (`pkDiffDB`): record the BLS key a node *had* before a change at a height,
  so reconstruction can restore prior keys.
- `apply_*_diffs(set, from=current_height, to=target_height+1, subnet)` iterates diffs in
  `[to, from]` and **un-applies** each: subtract added weight / add removed weight; restore prior
  keys. Diffs are written by the block acceptor when stakers enter/leave the current set.

> **Determinism contract for `06`:** the returned map is a `BTreeMap<NodeId,_>`; every consumer
> (sampler, proposervm windower, Warp canonical ordering) iterates in `NodeId` byte order. The
> `inverseHeight` encoding and the bool+weight diff value are **byte-exact protocol** (they're on
> disk but also define reconstruction equivalence) — differential-test reconstruction against Go
> (§9). `errUnfinalizedHeight` must be returned (not a panic) when `current < target`.

`GetValidatorOutput { node_id, public_key: Option<bls::PublicKey>, weight }` and
`GetCurrentValidatorOutput { validation_id, node_id, public_key, weight, start_time, min_nonce,
is_active, is_l1_validator }` are defined in `06` — this crate populates them.

---

## 8. Warp (BLS-signed cross-subnet messages)

`vms/platformvm/warp/` (the platform-side; the generic primitives live in `ava-crypto`/the warp
component). Components:
- **`UnsignedMessage { network_id: u32, source_chain_id: Id, payload: Vec<u8> }`**; ID = sha256.
- **`Signer`** (`signer.go` + `gwarp` gRPC): signs an unsigned message with the node's BLS key,
  used by the Warp API/Signature backend.
- **`Signature`**: `BitSetSignature { signers: Vec<u8> (bitset over the canonically-ordered
  validator set), signature: [u8;96] (aggregate) }`. Verification: take the height's validator set
  (via §7 `get_warp_validator_sets`), order canonically by BLS key, select signers by the bitset,
  aggregate public keys, verify the BLS aggregate over the message, and check the signing weight
  exceeds the required quorum (`quorum_num/quorum_den` of total weight).
- **`message/`, `payload/`**: typed ACP-77 payloads (`RegisterL1Validator`, `L1ValidatorWeight`,
  `SubnetToL1Conversion`, `L1ValidatorRegistration`) that the L1 txs consume/produce.
- **`ProofOfPossession`** (`signer/proof_of_possession.go`): `{ public_key: [u8;48] (compressed),
  proof: [u8;96] }`; `Verify` parses both and checks `bls::verify_proof_of_possession(pk, sig,
  pk_compressed_bytes)`. The `Empty` signer (type_id 27) is the non-primary-subnet case.

Use `ava-crypto::bls` (`blst`) for aggregate verify and PoP — guarantees byte-compatible
signatures (`00` §4.3). Signing is async (may call out to a `gwarp` gRPC backend); verification can
be **batched/parallelized** (§9).

---

## 9. API (JSON-RPC, mirrors `service.go`)

Served via `ava-api` (`12`) at `/ext/bc/P`. Method set (mirror `service.go` exactly — names,
request/response shapes, error codes). Notable methods:

`getHeight`, `getProposedHeight`, `getBalance`, `getUTXOs`, `getSubnet`, `getSubnets`,
`getStakingAssetID`, `getCurrentValidators` (incl. L1 validators), `getL1Validator`,
`getCurrentSupply`, `sampleValidators`, `getBlockchainStatus`, `validatedBy`, `validates`,
`getBlockchains`, `issueTx`, `getTx`, `getTxStatus`, `getStake`, `getMinStake`, `getTotalStake`,
`getRewardUTXOs`, `getTimestamp`, `getValidatorsAt`, `getAllValidatorsAt`, `getBlock`,
`getBlockByHeight`, `getFeeConfig`, `getFeeState`, `getValidatorFeeConfig`, `getValidatorFeeState`.

`client.rs` ports `client.go` (typed async wrappers over `reqwest`). JSON encodings of addresses
use bech32 (`P-…`); BLS keys hex (`0x…`, `HexNC`). `status/` ports the tx status enum
(`Committed`/`Aborted`/`Processing`/`Unknown`/`Dropped`).

---

## 10. Go → Rust mapping (non-obvious)

| Go | Rust |
|---|---|
| `txs.Visitor` interface (per-tx methods) | `Visitor` trait w/ default `ErrWrongTxType`; `UnsignedTx::visit` matches |
| `btree.BTreeG[*Staker]` ordered by `Less` | `BTreeSet<Staker>` w/ `Ord` = Go `Less` |
| `state.Diff` over `Versions` | `Diff` overlay; `Versions::get_state(block_id) -> Arc<dyn Chain>` |
| `*big.Int` reward math | `num_bigint::BigUint` (exact) |
| `gas` package u64 meter | `u64` checked arithmetic, no float |
| `verify.Verifiable` creds | `Credential` enum (secp256k1fx, type_id 9) |
| `fx.Owner` | `ava-secp256k1fx::OutputOwners` behind a `fx::Owner` trait |
| `window.Window` (recently accepted) | `06` `Window<Id>` reused |
| `MetadataCodec` v0/v1/v2 | one `Codec` manager, fields `#[codec(version=N)]` |
| `cache.Cacher` LRU | `ava-utils` `Lru` |
| `mockable.Clock` | injected `Clock` (test-controllable) |

**Error model:** `thiserror` `enum Error` per subpackage; preserve sentinels as variants and
assert via `matches!` where Go uses `errors.Is` (`00` §7.1): e.g. `ErrRemoveStakerTooEarly`,
`ErrMutatedL1Validator`, `ErrConflictingL1Validator`, `ErrWrongTxType`, `errUnfinalizedHeight`,
`ErrInvalidProofOfPossession`, `ErrNilTx`.

---

## 11. Test plan (per `02`)

1. **Codec golden vectors:** dump the 43-entry type_id table from Go; assert Rust enum
   discriminants match. Round-trip every tx type & block type against Go-produced bytes; assert
   identical `tx_id`/`block_id` hashes (incl. genesis-codec oversized txs, the L1 JSON test
   vectors `convert_subnet_to_l1_tx_test_*.json`, `register_l1_validator_tx_test.json`, etc.).
2. **Executor state-transition tests:** port `txs/executor/*_test.go` (standard/proposal/atomic),
   `advance_time_test.go`, `reward_validator_test.go`, `staker_tx_verification_test.go`,
   `warp_verifier_test.go` as table tests producing diffs; assert resulting state.
3. **Reward math proptests:** proptest `Calculate` & `Split` over random `(duration, amount,
   supply)` and **differential-test** against the Go calculator (exact equality; this is the
   highest-risk numeric path).
4. **Validator-set-at-height differential:** the marquee test — build a chain of staker
   add/remove blocks, then for every height assert `get_validator_set(h, subnet)` (weight + BLS
   key) **exactly equals** the Go `Manager.GetValidatorSet` output, including the diff-iterator
   `inverseHeight` reconstruction (`validator_set_property_test.go`, `manager_test.go`).
5. **Metadata codec v2:** round-trip `ValidatorMetadata` at v0/v1/v2 + the length-based legacy
   fallbacks against Go (`metadata_validator_test.go`).
6. **Warp:** PoP verify vectors; aggregate-signature verification against Go `signature_test.go`.
7. **Block oracle model:** proposal→commit/abort option generation & diff selection.
8. **API parity:** snapshot `service.go` request/response JSON; diff against Rust handlers.

---

## 12. Performance notes / improvements over Go

All gated by the differential tests above (`00` §9 — must not change observable behavior).

- **Parallel signature verification.** Tx credential (secp256k1) verification and Warp BLS
  aggregate verification are independent across txs/messages → verify on a rayon pool
  (`spawn_blocking`). Safe: verification is pure and order-independent; only the *accept* ordering
  (deterministic) is preserved.
- **BLS aggregate batch verification** for Warp (`ava-crypto`, `00` §9): batch-verify multiple
  signatures from one validator set.
- **Validator-diff windowing cache & lock-free reads.** The per-subnet height→set LRU (Go size 64)
  plus an `ArcSwap` over the current in-memory validator set lets `get_validator_set` for the
  current height be lock-free; historical reconstruction memoizes. Safe: reads are snapshots of an
  already-agreed state.
- **Diff-iterator over RocksDB prefix scans** with bounded reverse iterators (the `inverseHeight`
  key trick maps directly to a RocksDB forward iterator), avoiding Go's per-call allocation.
- **Zero-copy tx parsing:** keep `bytes::Bytes` for signed/unsigned slices (the prefix-length
  trick is a sub-slice, not a copy).
- **Sharded mempool** keyed by tx ID for concurrent gossip ingest, draining deterministically into
  the (single-threaded) block builder.
