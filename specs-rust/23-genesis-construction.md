# 23 — Genesis Construction (byte/ID-exact, per chain)

> **Status:** Canonical, byte-exact algorithm for the Rust port. Conforms to
> `00-overview-and-conventions.md` (binding: §1 compatibility surface, §6.1
> determinism). This is the **early interop gate**: a Rust node must produce
> genesis bytes, genesis block IDs, AVAX asset ID, and per-VM CreateChain tx IDs
> that are **byte-identical** to Go for Mainnet, Fuji, and Local, or it cannot
> join those networks.
>
> Cross-refs: `03` (linear codec, ids, CB58/bech32, hashing), `08` (P-Chain
> tx/UTXO/state types), `09` (X-Chain / AVM asset/UTXO types), `10` (C-Chain eth
> genesis via reth/alloy), `12` §6 (the `ava-genesis` crate design + "genesis is
> the source of truth" decision), `15` (genesis bytes use the linear codec),
> `22` (golden-vector test suite — this spec supplies the strongest single
> vector).

## Go source paths covered

| Go file | Role |
|---|---|
| `genesis/genesis.go` | `FromConfig` / `FromFile` / `FromFlag` build pipeline; `splitAllocations`; `VMGenesis`; `AVAXAssetID` |
| `genesis/config.go` | `Config`, `Allocation`, `LockedAmount`, `Staker`; `InitialSupply`; embedded-config `init()`; `getRecentStartTime` |
| `genesis/unparsed_config.go` | JSON ⇄ parsed conversion (bech32/hex string ↔ `ShortID`) |
| `genesis/genesis_mainnet.json` / `_fuji.json` / `_local.json` | embedded network configs (the `cChainGenesis` blob is a JSON-string field) |
| `genesis/bootstrappers.json` + `bootstrappers.go` | beacon list per network |
| `genesis/genesis_test.go` | the golden IDs (copied verbatim in §7) |
| `vms/avm/genesis.go` | `Genesis` / `GenesisAsset` / `NewGenesis` — X-Chain genesis bytes |
| `vms/platformvm/genesis/genesis.go` | `Genesis` / `New` / `Bytes` — P-Chain genesis bytes |
| `vms/platformvm/genesis/codec.go` | `Codec = block.GenesisCodec`, `CodecVersion = 0` |
| `vms/avm/txs/parser.go` | AVM `CodecVersion = 0`; genesis codec (secp256k1fx, nftfx, propertyfx) |
| `vms/platformvm/state/state.go` `init()` | P-Chain genesis **block** = `ApricotCommitBlock(genesisID, 0)`, `genesisID = ComputeHash256Array(genesisBytes)` |
| `vms/avm/state/state.go` `initializeChainState` | X-Chain genesis **block** = `StandardBlock(stopVertexID, 0, ts, nil)` |

---

## 1. Inputs — the genesis `Config`

Genesis is fully determined by one struct. The on-disk/embedded form is JSON
with string-encoded addresses (the **unparsed** config); the build pipeline
operates on the **parsed** form with `ShortID` addresses. Field names/JSON tags
are **protocol constants** — they must match Go exactly.

```rust
// genesis/config.go::Config  ->  ava_genesis::Config (parsed form)
pub struct Config {
    pub network_id: u32,                       // "networkID"
    pub allocations: Vec<Allocation>,          // "allocations"
    pub start_time: u64,                       // "startTime"  (unix seconds)
    pub initial_stake_duration: u64,           // "initialStakeDuration"        (seconds)
    pub initial_stake_duration_offset: u64,    // "initialStakeDurationOffset"  (seconds)
    pub initial_staked_funds: Vec<ShortId>,    // "initialStakedFunds"  (bech32 in JSON)
    pub initial_stakers: Vec<Staker>,          // "initialStakers"
    pub c_chain_genesis: String,               // "cChainGenesis"  (a JSON string holding eth genesis)
    pub message: String,                       // "message"
}

// genesis/config.go::Allocation
pub struct Allocation {
    pub eth_addr: ShortId,                     // "ethAddr"   (0x-hex in JSON, 20 bytes)
    pub avax_addr: ShortId,                    // "avaxAddr"  (bech32 in JSON, 20 bytes)
    pub initial_amount: u64,                   // "initialAmount"   -> X-Chain UTXO
    pub unlock_schedule: Vec<LockedAmount>,    // "unlockSchedule"  -> P-Chain UTXOs
}

// genesis/config.go::LockedAmount
pub struct LockedAmount {
    pub amount: u64,                           // "amount"
    pub locktime: u64,                         // "locktime"  (unix seconds; 0 = unlocked)
}

// genesis/config.go::Staker
pub struct Staker {
    pub node_id: NodeId,                       // "nodeID"
    pub reward_address: ShortId,               // "rewardAddress"  (bech32 in JSON)
    pub delegation_fee: u32,                   // "delegationFee"  (PerMille*1000; e.g. 20000 = 2%)
    pub signer: Option<ProofOfPossession>,     // "signer"  (BLS PoP; nil => legacy AddValidatorTx)
}
```

**Unparsed ⇄ parsed** (`unparsed_config.go`): JSON stores `ethAddr` as
`"0x"+hex(20 bytes)`, `avaxAddr` / `rewardAddress` / `initialStakedFunds[i]` as
bech32 `X-<hrp>1...`. Parse strips the chain alias + HRP and decodes to a 20-byte
`ShortID`. `cChainGenesis` is a **JSON string** whose content is a go-ethereum
`core.Genesis` document (see 10).

**Derived quantities** (`Config` methods):
- `initial_supply()` = Σ over allocations of `initial_amount` + Σ `unlock_schedule[].amount`, with checked `u64` add (overflow → error). Used for the P-Chain `InitialSupply` field.
- HRP = `constants::get_hrp(network_id)` (`avax` mainnet, `fuji`, `local`, `custom`).

---

## 2. Validation (`validateConfig`) — gate before build

Run before `from_config` for **custom** networks (embedded Mainnet/Fuji/Local are
pre-validated). Each check maps to a `GenesisError` variant (§6); errors are
matched by identity in tests (`require.ErrorIs`):

1. `network_id == config.network_id` else `ConflictingNetworkIds`.
2. `initial_supply() > 0` else `NoSupply` (computation error → wrapped).
3. `start_time` not in the future (`time.Since(start) >= 0`) else `FutureStartTime`.
4. `initial_stake_duration != 0` else `NoStakeDuration`.
5. `initial_stake_duration <= staking_cfg.max_stake_duration_secs` else `StakeDurationTooHigh`.
6. `len(initial_stakers) > 0` else `NoStakers`.
7. `initial_stake_duration_offset * (len(stakers) - 1) <= initial_stake_duration` else `InitialStakeDurationTooLow`.
8. `validate_initial_staked_funds`: non-empty; each entry unique (`DuplicateInitiallyStakedAddress`) and present in the set of allocation `avax_addr`s (`NoAllocationToStake`). Duplicate `avax_addr` **across allocations is allowed** (different `eth_addr`s can map to the same `avax_addr`).
9. `validate_allocations_locked_amount`: Σ of all `unlock_schedule[].amount` ≥ `len(initial_stakers)` else `AllocationsLockedAmountTooLow`.
10. `c_chain_genesis` non-empty else `NoCChainGenesis`.

> `FromFile`/`FromFlag` additionally **reject** Mainnet/Fuji/Local network IDs with
> `OverridesStandardNetworkConfig` (those use the embedded config, never a file).

---

## 3. Construction algorithm — `from_config(&Config) -> (Vec<u8>, Id)`

Returns `(p_chain_genesis_bytes, avax_asset_id)`. The order below is **load-bearing**
for byte parity. Mirrors `genesis.go::FromConfig`.

### 3.0 Codec & hashing facts (constants)

- Both the AVM genesis codec and the PlatformVM genesis codec are the **linear
  codec** (03 §2), **version `0`** (`avm/txs.CodecVersion = 0`,
  `platformvm/genesis.CodecVersion = block.CodecVersion = txs.CodecVersion = 0`).
  The 2-byte BE version prefix on the produced bytes is therefore `00 00`.
- The genesis codecs use a **`MaxInt32` max-slice-len** manager (not the default
  256 KiB) so oversized genesis blobs serialize; otherwise identical to 03.
- The AVM genesis codec registers fxs in order **secp256k1fx, nftfx, propertyfx**
  (type-ID assignment order — matters for any fx-typed interface field).
- All hashing is `sha256` → `ComputeHash256` (32-byte) / `ComputeHash256Array`
  (→ `Id`), 03 §3.

### 3.1 Build the X-Chain (AVM) genesis → bytes → AVAX asset ID

1. Start an `AssetDefinition { name:"Avalanche", symbol:"AVAX", denomination:9, initial_state: empty }`.
2. Collect `x_allocations` = allocations with `initial_amount > 0`.
3. **Sort `x_allocations`** by `Allocation::compare`: primary key `initial_amount`
   ascending, tie-break `avax_addr` ascending (byte compare). *(This is the
   sort that fixes the X-Chain UTXO order — critical for the asset ID.)*
4. For each sorted allocation, in order:
   - append `Holder { amount: initial_amount, address: bech32(hrp, avax_addr) }` to `FixedCap`;
   - append the raw 20 `eth_addr` bytes to `memo` (concatenated).
5. Set `asset.memo = memo`.
6. `avm::new_genesis(network_id, {"AVAX": asset})`:
   - builds one `GenesisAsset { alias:"AVAX", CreateAssetTx{ BaseTx{network_id, blockchain_id: Empty, memo}, name, symbol, denomination } }`;
   - `FixedCap` holders → `secp256k1fx::TransferOutput { amt, owners{threshold:1, addrs:[holder_addr]} }` in `InitialState{ fx_index: 0, outs }`;
   - **sort the `InitialState.outs`** with the genesis codec (`InitialState::sort`, codec-byte order of serialized outputs);
   - **sort `asset.states`** and **sort `g.txs`** by `GenesisAsset::compare` = `cmp(alias)` (single asset here, but the sort is part of the contract).
7. `avm_genesis_bytes = genesis_codec.marshal(0, &g)`.
8. **`avax_asset_id`** = re-parse `avm_genesis_bytes` with the genesis codec, take
   `genesis.txs[0]`, wrap its `CreateAssetTx` in a `Tx{ unsigned }`,
   `tx.initialize(genesis_codec)` (computes `unsigned_bytes`/`bytes`), then
   `tx.id() = ComputeHash256Array(tx.bytes)`. *(Equivalently the hash of the
   initialized CreateAssetTx bytes; do **not** shortcut — match `AVAXAssetID`.)*

### 3.2 Build the P-Chain genesis UTXOs (non-staked allocations)

1. `initially_staked = set(config.initial_staked_funds)`.
2. Iterate `config.allocations` **in config order** (not sorted). For each:
   - if `avax_addr ∈ initially_staked`: push to `skipped_allocations`, skip;
   - else for each `unlock` in `unlock_schedule` with `amount > 0`, emit a
     `platform::Allocation { locktime: unlock.locktime, amount: unlock.amount, address: bech32(hrp, avax_addr), message: eth_addr.bytes() }`.
3. These become P-Chain genesis `UTXO`s (built in §3.4 by `genesis::New`), one per
   emitted allocation, in **this iteration order** (config order × unlock-schedule order).

### 3.3 Build the P-Chain initial validators

1. `all_node_allocations = split_allocations(skipped_allocations, len(initial_stakers))`
   — distributes the staked funds across stakers (see §3.3.1).
2. `end_staking_time = genesis_time + initial_stake_duration secs` (genesis_time = `unix(start_time)`).
3. `staking_offset = 0`. For each staker `i` (config order):
   - `this_end = end_staking_time - staking_offset`; then `staking_offset += initial_stake_duration_offset secs` (post-increment).
   - For each allocation in `all_node_allocations[i]`, for each `unlock`, emit a stake `platform::Allocation { locktime, amount, address: bech32(hrp, avax_addr), message: eth_addr.bytes() }`.
   - Build a `PermissionlessValidator { Validator{ start: start_time, end: this_end_unix, node_id }, reward_owner: Owner{ threshold:1, addresses:[bech32(hrp, reward_address)] }, staked: allocations, exact_delegation_fee: delegation_fee, signer }`.

#### 3.3.1 `split_allocations` (exact)

Greedy split of the staked allocations into `num_splits` buckets of roughly
`node_weight = total_staked_amount / num_splits` each (integer division;
remainder lands in the last bucket). Walk allocations in `skipped_allocations`
order; within each, walk `unlock_schedule` in order; carve a current bucket's
unlock entries until it reaches `node_weight`, then start the next bucket
(splitting an `unlock.amount` across the boundary when needed). The last bucket
absorbs the remainder. Each emitted sub-allocation has `initial_amount=0` and a
freshly built `unlock_schedule`. **Reproduce the loop verbatim** — the exact
split determines each validator's stake outputs and weight, hence the validator
tx IDs.

### 3.4 Assemble the P-Chain genesis (`platformvm/genesis::New`)

Inputs: `avax_asset_id`, `network_id`, `platform_allocations` (§3.2),
`validators` (§3.3), `chains` (§3.5), `start_time`, `initial_supply`, `message`.

**UTXOs** (one per allocation, in input order — index = `output_index`):
- `amount == 0` → error `UtxoHasNoValue`.
- `UTXO { UTXOID{ tx_id: Id::EMPTY, output_index: i as u32 }, Asset{ id: avax_asset_id }, out }` where
  `out = secp256k1fx::TransferOutput { amt: amount, owners{ locktime:0, threshold:1, addrs:[addr_id] } }`.
- If `allocation.locktime > start_time`: wrap `out` in `stakeable::LockOut { locktime, transferable_out: out }`.
- Wrap as `genesis::UTXO { utxo, message }`.

**Validators** (each → a `Tx`, then ordered by a **by-end-time heap**):
- For each validator, build `stake` = `TransferableOutput`s from `staked`
  allocations **after sorting `vdr.staked`** by `Allocation::compare` (locktime, then
  amount, then address); `LockOut`-wrap when `locktime > start_time`; sum `weight`
  (checked add; overflow → `StakeOverflow`).
- `weight == 0` → `ValidatorHasNoWeight`; `vdr.end <= start_time` → `ValidatorAlreadyExited`.
- Build `reward_owner` = `secp256k1fx::OutputOwners { locktime, threshold, addrs }`
  with **`addrs` sorted** (`utils.Sort`).
- `BaseTx { network_id, blockchain_id: Empty }`, `Validator { node_id, start, end, weight }`.
- `signer == None` → `AddValidatorTx { base, validator, stake_outs, rewards_owner, delegation_shares: delegation_fee }`.
  `signer == Some` → `AddPermissionlessValidatorTx { base, validator, signer, stake_outs, validator_rewards_owner: owner, delegator_rewards_owner: owner, delegation_shares: delegation_fee }`.
- `tx.initialize(genesis_codec)`, then **add to a `txheap.ByEndTime`**. The final
  `Validators` slice is `heap.list()` — **ordered by end time** (tie-break by tx
  ID; reproduce the heap's `Less`). This re-orders validators away from config
  order, so the heap ordering is a hard parity requirement.

**Chains** (in this fixed order — see §3.5), each → `CreateChainTx`
`{ BaseTx{network_id, blockchain_id:Empty}, subnet_id, chain_name, vm_id, fx_ids, genesis_data, subnet_auth: secp256k1fx::Input{} }`, `tx.initialize(genesis_codec)`.

**Final struct** (linear-codec field order is struct-declaration order — do not
reorder):
```
Genesis { UTXOs, Validators, Chains, Timestamp = start_time, InitialSupply = initial_supply, Message = message }
```
`p_chain_genesis_bytes = Codec.marshal(0, &genesis)` (linear codec, version 0,
`MaxInt32` manager). Return `(p_chain_genesis_bytes, avax_asset_id)`.

### 3.5 The chains list (fixed order)

```rust
[
  Chain { genesis_data: avm_genesis_bytes, subnet_id: PRIMARY_NETWORK_ID, vm_id: AVM_ID,
          fx_ids: [secp256k1fx::ID, nftfx::ID, propertyfx::ID], name: "X-Chain" },
  Chain { genesis_data: c_chain_genesis.as_bytes(), subnet_id: PRIMARY_NETWORK_ID, vm_id: EVM_ID,
          fx_ids: [], name: "C-Chain" },
]
```
X-Chain **first**, C-Chain **second** — never reorder; the CreateChainTx IDs (and
hence the X/C blockchain IDs the rest of the node uses) depend on it. `fx_ids` for
X are exactly `[secp256k1fx, nftfx, propertyfx]`; C has none.

### 3.6 C-Chain genesis (embedded; block derived by reth/alloy)

The C-Chain genesis is **not** computed here: `cChainGenesis` is carried verbatim
(as UTF-8 bytes) into the C-Chain `CreateChainTx.GenesisData`. The C-Chain's
genesis **block hash** is computed by reth/alloy from that go-ethereum genesis
JSON (`core.Genesis`) — see 10. `ava-genesis` only validates it is non-empty and
parseable JSON, and (for the timestamp golden test) that `Timestamp` matches the
network's expected value (Mainnet/Fuji = `0`; Local = `upgrade::InitiallyActiveTime`).

---

## 4. Genesis **blocks** and their IDs (per chain)

The "genesis bytes" above are the VM genesis **state**, not blocks. Each chain
derives its accepted genesis block deterministically:

### 4.1 P-Chain genesis block

```text
genesis_id    = ComputeHash256Array(p_chain_genesis_bytes)     // == hash of the bytes
genesis_block = ApricotCommitBlock { parent_id: genesis_id, height: 0 }
```
`platformvm/state.init`: the block's *parent* is set to the hash of the genesis
state bytes, height 0; the block is stored as last-accepted without `Accept()`.
The **`expectedID` in `genesis_test.go::TestGenesis`** (§7) is exactly
`ComputeHash256Array(p_chain_genesis_bytes).String()` — i.e. the hash of the
P-Chain genesis bytes. This is **the** primary golden value.

### 4.2 X-Chain genesis block

`avm/state.initializeChainState` builds `StandardBlock { parent_id: stop_vertex_id,
height: 0, timestamp: genesis_timestamp, txs: nil }` and uses `block.id()` as last
accepted. (`stop_vertex_id` comes from linearization; the X-Chain genesis **state**
bytes are `avm_genesis_bytes` from §3.1, and the **X-Chain blockchain ID** is the
`CreateChainTx` ID from `VMGenesis(p_bytes, AVM_ID)`.)

### 4.3 Per-VM chain (blockchain) IDs

`vm_genesis(p_chain_genesis_bytes, vm_id)` parses the P-Chain genesis, finds the
`CreateChainTx` whose `vm_id` matches, and returns its tx; `tx.id()` is the
**blockchain ID** for that chain (X-Chain ID = AVM CreateChainTx ID; C-Chain ID =
EVM CreateChainTx ID). These are golden values too (§7, `TestVMGenesis`).

---

## 5. Embedded network configs & bootstrappers

### 5.1 Embedded JSON

`genesis_mainnet.json`, `genesis_fuji.json`, `genesis_local.json` are embedded
verbatim (`include_str!`), parsed once into `MAINNET_CONFIG` / `FUJI_CONFIG` /
`LOCAL_CONFIG` (the unparsed → parsed path of §1).

```rust
pub static MAINNET_GENESIS_CONFIG_JSON: &str = include_str!("../data/genesis_mainnet.json");
pub static FUJI_GENESIS_CONFIG_JSON:    &str = include_str!("../data/genesis_fuji.json");
pub static LOCAL_GENESIS_CONFIG_JSON:   &str = include_str!("../data/genesis_local.json");

pub fn get_config(network_id: u32) -> Config {  // mirror genesis.go::GetConfig
    match network_id {
        MAINNET_ID => MAINNET_CONFIG.clone(),
        FUJI_ID    => FUJI_CONFIG.clone(),
        LOCAL_ID   => LOCAL_CONFIG.clone(),
        other      => { let mut c = LOCAL_CONFIG.clone(); c.network_id = other; c } // local template
    }
}
```

> **Local `start_time` quirk (must replicate):** `LocalConfig` is *not* used
> as-embedded. Go advances the embedded `startTime` in `getRecentStartTime` by
> chunks of `9 months` (`9*30*24h`) until the latest value `<= now`. So the live
> `LocalConfig.start_time` is time-dependent (and the live local genesis ID is
> **not** a fixed golden value). The golden test uses **`unmodified_local_config`**
> (the pre-advance config) for the Local ID in §7. Implement both: keep
> `UNMODIFIED_LOCAL_CONFIG` for the golden test, and the advanced `LOCAL_CONFIG`
> for actually running a local node.

### 5.2 Bootstrappers / beacons

`bootstrappers.json` embedded as `map<network_name, Vec<Bootstrapper{id: NodeId,
ip: SocketAddr}>>`. `bootstrappers(network_id)` looks up by
`network_id → network_name`. `sample_bootstrappers(network_id, count)` mirrors
`SampleBootstrappers` using the uniform sampler (05) — order/selection must match
for the manual-track wiring in `ava-node` (12 §2.3). Custom networks have no
embedded beacons (fed from `--bootstrap-ips/-ids`, 12 §1.6).

---

## 6. Rust design — the `ava-genesis` crate (12 §6)

```rust
/// genesis.go::FromConfig — the build pipeline. Reuses tx/UTXO types from 08/09.
pub fn from_config(config: &Config) -> Result<(Vec<u8> /*p-chain genesis*/, Id /*avaxAssetID*/)>;

/// FromFile / FromFlag for custom networks (validate + build). Rejects std networks.
pub fn from_file(network_id: u32, path: &Path, staking: &StakingConfig) -> Result<(Vec<u8>, Id)>;
pub fn from_flag(network_id: u32, content_b64: &str, staking: &StakingConfig) -> Result<(Vec<u8>, Id)>;

/// Convenience for the node / golden tests.
pub enum Chain { P, X, C }
pub fn genesis_bytes(network_id: u32, custom: Option<&Config>) -> Result<(Vec<u8>, Id)>; // P-Chain bytes + asset ID
pub fn genesis_block_id(network_id: u32, chain: Chain) -> Result<Id>;                     // §4
pub fn vm_genesis(p_bytes: &[u8], vm_id: Id) -> Result<p_chain::Tx>;                      // blockchain ID source
pub fn avax_asset_id(avm_genesis_bytes: &[u8]) -> Result<Id>;

pub fn bootstrappers(network_id: u32) -> Vec<Bootstrapper>;
```

- Reuses `ava-codec` (15), `ava-crypto::address` bech32 (03), `ava-platformvm`
  tx/UTXO/genesis types (08), `ava-avm` genesis/tx types (09), and the eth genesis
  parse from `ava-cchain`/reth (10) for the timestamp check only.
- `from_config` is pure (no I/O) and cached per `(network_id)` for the embedded
  configs; the result is computed once at node start (12 §14).

### 6.1 Error model (`GenesisError`, `thiserror`)

One variant per Go sentinel so `require.ErrorIs`-style parity tests map 1:1:
`ConflictingNetworkIds`, `NoSupply`, `FutureStartTime`, `NoStakeDuration`,
`StakeDurationTooHigh`, `NoStakers`, `InitialStakeDurationTooLow`,
`NoInitiallyStakedFunds`, `DuplicateInitiallyStakedAddress`, `NoAllocationToStake`,
`AllocationsLockedAmountTooLow`, `NoCChainGenesis`, `OverridesStandardNetworkConfig`,
`InvalidGenesisJson`, `NoTxs`, `UtxoHasNoValue`, `ValidatorHasNoWeight`,
`ValidatorAlreadyExited`, `StakeOverflow`, plus `Codec(..)` / `Address(..)` wraps.

---

## 7. Expected genesis IDs (golden values — copied from `genesis/genesis_test.go`)

These are the **golden vectors** (extracted, not "to be extracted"; they live in
`TestGenesis`/`TestVMGenesis`/`TestAVAXAssetID` and the custom hash in
`TestGenesisFromFile`). The Local row uses `unmodified_local_config` (§5.1).

**P-Chain genesis block ID** = `ComputeHash256Array(p_chain_genesis_bytes)` (§4.1):

| Network | P-Chain genesis block ID |
|---|---|
| Mainnet | `UUvXi6j7QhVvgpbKM89MP5HdrxKm9CaJeHc187TsDNf8nZdLk` |
| Fuji | `MSj6o9TpezwsQx4Tv7SHqpVvCbJ8of1ikjsqPZ1bKRjc9zBy3` |
| Local (unmodified) | `2nRRoR76HuEk1JjDpRdN8FKvZFvUXWxY3b9C5rZRPFjcgEh7S7` |

**Per-VM blockchain IDs** = `VMGenesis(p_bytes, vm_id).ID()` (§4.3):

| Network | X-Chain ID (AVM CreateChainTx) | C-Chain ID (EVM CreateChainTx) |
|---|---|---|
| Mainnet | `2oYMBNV4eNHyqk2fjjV5nVQLDbtmNJzq5s3qs3Lo6ftnC6FByM` | `2q9e4r6Mu3U68nU1fYjgbR6JvwrRx36CohpAX5UQxse55x1Q5` |
| Fuji | `2JVSBoinj9C2J33VntvzYtVJNZdN2NKiwwKjcumHUWEb5DbBrm` | `yH8D7ThNJkxmtkuv2jgBa4P1Rn3Qpr4pPr7QYNfcdoS6k6HWp` |
| Local | `2eNy1mUFdmaxXNj1eQHUe7Np4gju9sJsEtWQ4MX3ToiNKuADed` | `2owdGqyG6FFzTHy5qhenDXQcEghvr571KZE3gSfRJERSJinuwC` |

**AVAX asset ID** = `AVAXAssetID(avm_genesis_bytes)` (§3.1):

| Network | AVAX asset ID |
|---|---|
| Mainnet | `FvwEAhmxKfeiG8SnEvq42hc6whRyY3EFYAvebMqDNDGCgxN5Z` |
| Fuji | `U8iRqJoiJm8xZHAacmvYyZVwqQx6uDNtQeP3CQ6fcgQk3JqnK` |
| Local | `2fombhL7aGPwj3KH4bfrmJwW6PVnMobf9Y2fn9GwxiAAJyFDbe` |

**Custom-config hash** (hex `ComputeHash256(p_bytes)` for `genesis_test.json`,
networkID 9999): `a1d1838586db85fe94ab1143560c3356df9ba2445794b796bba050be89f4fcb4`.

**C-Chain genesis timestamp** (parsed from `cChainGenesis`): Mainnet `0`, Fuji `0`,
Local `unix(upgrade::InitiallyActiveTime)`.

> The X/C **genesis block hashes** (the eth/standard-block IDs from §4.2/§4.3 vs.
> the *blockchain* IDs above) are cross-referenced from 09/10; the table above is
> the avalanchego-side identity set, which is the interop gate.

---

## 8. Go → Rust mapping (non-obvious)

| Go | Rust |
|---|---|
| `utils.Sort(xAllocations)` (Sortable.Compare) | stable sort by `(initial_amount, avax_addr)` |
| `txheap.NewByEndTime()` for validators | min-heap / sort by `(end_time, tx_id)`; emit `list()` |
| `address.FormatBech32(hrp, bytes)` | `ava_crypto::address::format_bech32` (03) |
| `block.GenesisCodec` (MaxInt32) | `ava_codec` manager with `max_slice_len = i32::MAX` |
| `ComputeHash256Array(bytes)` → `ids.ID` | `Id::from(sha256(bytes))` (03) |
| `core.Genesis` (libevm) for timestamp | reth/alloy `Genesis` JSON parse (10) — validate only |
| `getRecentStartTime` (9-month chunks) | same loop; only for live Local config |
| `set.Of(initialStakedFunds...)` | `HashSet<ShortId>` membership test |
| `math.Add` (checked) for supply/weight | `u64::checked_add` → error on overflow |

---

## 9. Test plan (ref `02`, `22`)

1. **THE genesis-block-ID golden test (CI-blocking).** For Mainnet, Fuji, Local
   (unmodified) and the custom `genesis_test.json`: assert `from_config` →
   `genesis_block_id(P)`, the X/C `vm_genesis` IDs, the AVAX asset ID, and the
   custom hex hash **byte-match the §7 table**. This is the strongest single
   compatibility check; failure means the node cannot join.
2. **Round-trip / rebuild parity.** Parse each embedded JSON (`unparsed → parsed`),
   `from_config`, then re-serialize and assert byte-identity against bytes dumped
   from Go `genesis.FromConfig` (not just the ID hash — the full byte stream),
   guarding the intermediate orderings (X-alloc sort, validator end-time heap,
   reward-addr sort, chain order).
3. **`validateConfig` parity** (table test mirroring `TestValidateConfig`): every
   error variant matched by identity, plus the "duplicate avaxAddr across
   allocations is allowed" / "empty message is allowed" cases.
4. **`split_allocations` unit vectors**: fixed staked-allocation sets × num_splits,
   asserting per-bucket unlock schedules + resulting validator weights match Go.
5. **Bootstrapper parity**: count + IDs + IPs per network vs `bootstrappers.json`;
   `sample_bootstrappers` selection determinism vs Go's uniform sampler.
6. **C-Chain timestamp** golden (`TestCChainGenesisTimestamp`): Mainnet/Fuji 0,
   Local = `InitiallyActiveTime`.
7. **Local start-time advance** unit test for `getRecentStartTime` (fixed `now`
   inputs → expected advanced start).

## 10. Performance notes

Genesis is computed **once** at startup and cached; cost is dominated by two
sha256 passes and a few-KB codec marshal — sub-millisecond, no hot path. The only
correctness-sensitive concern is determinism (sorts/heap), which the golden test
in §9.1 guards. No optimization is warranted or attempted.
