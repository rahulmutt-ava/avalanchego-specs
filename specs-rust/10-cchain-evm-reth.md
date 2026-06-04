# 10 — C-Chain & EVM Subnets on reth

> **Status:** Conforms to `00-overview-and-conventions.md`. Binding choices from
> §4.5 (reth crates for EVM), §4.4 (Firewood with `ethhash` for EVM state), and
> the storage contract in `04-storage-and-databases.md` §4 are honored here.
> Consensus boundary is `06-consensus.md` (the `Block`/`ChainVm` traits) and the
> VM-framework / atomic-shared-memory boundary is `07-vm-framework.md`. Where this
> spec adds a binding decision (the **reth-as-library integration mode**, the
> **Firewood-EVM-state contract**, the **atomic-tx executor-hook model**) every
> other spec MUST conform.
>
> This is the most research-heavy spec. The grafted Go forks are **reference
> inputs**, not transliteration targets: we re-derive their *behavior* on reth.

This document specifies `ava-evm`: the crate that runs the **C-Chain** and
**EVM-based subnets** by embedding [**reth**](https://github.com/paradigmxyz/reth)
as a *library* (not as a standalone node), exposing them to Avalanche consensus
through the `ChainVm`/`Block` traits.

---

## 0. Go source covered (reference inputs)

| Go path | Subject | Re-derived as |
|---|---|---|
| `vms/evm/` (the avalanchego-side wiring) | predicate, sync, uptimetracker, database, metrics, acp176, acp226 | `ava-evm` wiring + reth integration |
| `graft/evm` (the shared Avalanche-EVM base, ex-libevm) | the extensible EVM base, hooks | reth `ConfigureEvm` + revm hooks |
| `graft/coreth` (C-Chain) | `plugin/evm/*`, `core`, `miner`, `params`, `warp`, atomic | `ava-evm` C-Chain profile |
| `graft/coreth/plugin/evm/atomic` | ImportTx/ExportTx, atomic trie, backend, txpool, shared memory | custom tx type + executor hook + atomic-trie side store |
| `graft/coreth/plugin/evm/customheader` | dynamic-fee window, base fee, block gas cost | custom fee calculator in `ConfigureEvm` |
| `graft/coreth/params/extras` | Avalanche fork schedule + `FeeConfig` | custom `Hardforks` + chain config |
| `graft/subnet-evm/precompile` | allowlist / fee-manager / reward-manager / nativeminter / warp precompile framework | revm `PrecompileProvider` + stateful-precompile registry |
| `graft/coreth/plugin/evm/wrapped_block.go`, `vm.go`, `block_builder.go`, `miner/` | Snowman block wrapper, on-demand build, Verify/Accept/Reject | `ChainVm` adapter + custom block builder |

**Rust crate produced:** `ava-evm` (with internal modules; an optional
sub-crate split mirrors the Go layout — see §13). External deps (per §4.5):
`reth-evm`, `reth-ethereum`, `revm`, `op-revm` (as a *pattern reference*),
`reth-provider`/`reth-storage-api`, `reth-chainspec`/`reth-ethereum-forks`,
`reth-rpc`/`reth-rpc-eth-api`, `reth-transaction-pool`, `reth-primitives`,
`alloy-primitives`/`alloy-consensus`/`alloy-rpc-types-eth`, plus `firewood`
(`features=["ethhash"]`, §4.2 of 04). Pin reth to a **single, vendored revision**
(reth has no stable semver for its library crates — see §12, gap G0).

---

## 1. Integration mode (BINDING DECISION)

reth is designed to be driven by the **Engine API** (`engine_newPayload` /
`engine_forkchoiceUpdated`) from an external consensus client, OR run as a full
node with its own staged sync. Avalanche consensus (Snowman, via the
`block.ChainVm` boundary in 06/07) is neither: it is a *DAG-of-blocks, one-block-
at-a-time linear* engine that (a) wants the **post-state root before voting**, and
(b) makes the **Accept/Reject** decision itself (no PoW/PoS fork choice). Coreth
already does this — it drives `core.BlockChain` directly, with no miner loop and
no engine API.

> **DECISION — reth-as-library executor, NOT the Engine API.** `ava-evm`
> consumes reth's **execution and storage crates as libraries** (`reth-evm`'s
> `ConfigureEvm` / `BlockExecutorFactory` / `BlockExecutor` / `BlockBuilder`,
> `reth-provider`'s `StateProvider*` traits, `revm`) and **does not** instantiate
> reth's `NodeBuilder`, engine, `PayloadBuilderService`, staged sync, or
> `BeaconConsensusEngine`. We build/execute/commit blocks **on command** from the
> `ChainVm` adapter. This is the same shape as coreth-on-libevm, and it is the
> only mode that gives us pre-commit state roots and consensus-owned fork choice.

**Why not the Engine API:** `forkchoiceUpdated` semantics (HEAD/SAFE/FINALIZED,
reorg-by-fork-choice, payload-id polling) actively fight Snowman's linear
accept/reject and would require a fake beacon client. `newPayload` does not return
a pre-execution opportunity to vote. We would also inherit reth's MDBX-as-truth
assumption, which conflicts with **Firewood-as-state-truth** (§5). Rejected.

**Why reth-as-library is viable:** reth's SDK explicitly supports "unbundling the
node into the components you need" — talking directly to the executor, the
state providers, and the DB. The `Executor`/`BlockExecutor` traits execute a
single block against a `State` DB and return a `BlockExecutionResult` (receipts +
state changes) with no node attached. `reth_provider`'s `BlockExecutorProvider`
and `NoopProvider`-style usage are documented for exactly this.

**Honest gap (G0, §12):** reth's *library* crates carry **no API-stability
guarantee**, and there is **no first-class "external consensus" entrypoint** —
this is the central risk of the whole spec, fully owned by `ava-evm`. We pin a
vendored reth revision and wrap every reth touch-point behind our own traits so a
reth bump is localized.

### 1.1 Layer diagram

```
  ┌─────────────────────────── ava-engine (Snowman, 06) ───────────────────────────┐
  │  ParseBlock / BuildBlock / SetPreference / Block::{verify,accept,reject}         │
  └───────────────────────────────────┬─────────────────────────────────────────────┘
                                       │  block.ChainVm boundary (07); proposervm wraps it (06)
  ┌────────────────────────────────────▼────────────────────────────────────────────┐
  │ ava-evm                                                                            │
  │  ┌──────────────┐  ┌───────────────┐  ┌───────────────┐  ┌──────────────────────┐ │
  │  │ ChainVm      │  │ Block builder │  │ Atomic backend│  │ JSON-RPC (eth_*/avax.*)│ │
  │  │ adapter      │  │ (on-demand)   │  │ + atomic trie │  │  reth-rpc + ava module │ │
  │  │ (§3)         │  │ (§4)          │  │ + shared mem  │  │  (§9)                  │ │
  │  └──────┬───────┘  └──────┬────────┘  │  (§6, 07)     │  └──────────────────────┘ │
  │         │                 │           └──────┬────────┘                            │
  │  ┌──────▼─────────────────▼──────────────────▼─────────────────────────────────┐  │
  │  │ reth (library): ConfigureEvm (§7 fees, §8 precompiles) → BlockExecutorFactory │  │
  │  │ → BlockExecutor / BlockBuilder (revm) ; AvaTxPool (§6.4)                       │  │
  │  └──────────────────────────────────┬───────────────────────────────────────────┘ │
  │  ┌──────────────────────────────────▼───────────────────────────────────────────┐ │
  │  │ FirewoodStateProvider : reth StateProvider/StateRootProvider/... (§5)          │ │
  │  └──────────────────────────────────┬───────────────────────────────────────────┘ │
  └─────────────────────────────────────┼─────────────────────────────────────────────┘
                                         │
                  ┌──────────────────────▼─────────────────────────┐
                  │ firewood (ethhash): propose→root, commit (04 §4) │  + reth-db (ancients:
                  └──────────────────────────────────────────────────┘   headers/bodies/receipts)
```

State **truth** is Firewood-ethhash. reth's MDBX/static-files are used only for
**block/receipt/header storage and indexing** (the "ancient" + history tables),
never as the state-root source of record. This is the **Firewood-EVM-state
contract** with 04 (§5, and the cross-spec summary).

---

## 2. The customization surface (what Avalanche changes vs. vanilla Ethereum)

Distilled from the Go reference, the code-agnostic divergences from vanilla
Ethereum, each mapped to its reth extension point:

| # | Avalanche customization | reth/revm extension point | Section |
|---|---|---|---|
| C1 | On-demand block building (no miner/PoW, consensus triggers) | custom `BlockBuilder` driver (not `PayloadBuilderService`) | §4 |
| C2 | Snowman fork choice (Accept→canonicalize, Reject→discard, no reorg race) | `ChainVm` adapter owns canonical head; bypass reth fork choice | §3 |
| C3 | Atomic Import/Export txs + shared memory + atomic trie | custom tx type + `BlockExecutor` pre/post hook + side trie | §6 |
| C4 | Avalanche dynamic fee (AP3 window, AP4 block gas cost, Fortuna/ACP-176 fee state) | custom base-fee/gas in `ConfigureEvm::next_evm_env` + a `feerules` module | §7 |
| C5 | Avalanche fork schedule (Apricot→Banff→Cortina→Durango→Etna→Fortuna→Granite) on top of Ethereum forks | custom `Hardforks` + `AvaChainSpec` | §7.4 |
| C6 | Warp precompile + subnet-evm stateful precompiles (allowlist, feemanager, nativeminter, rewardmanager) | revm `PrecompileProvider` + a precompile registry | §8 |
| C7 | EVM state root via Firewood-ethhash (not reth MPT/MDBX) | custom `StateProvider`/`StateRootProvider` | §5 |
| C8 | EVM state sync (snap-like accepted-state sync) | reth sync mapped onto Firewood range/change proofs | §10 |
| C9 | `eth_*` + `avax.*` RPC, block `bytes` wire format | reth-rpc `EthApi` + an `avax` RPC module | §9 |
| C10 | Predicates (precompile pre-tx checks: warp signature verify) | `BlockExecutor` pre-execution predicate pass | §6.5/§8 |

---

## 3. The `ChainVm` adapter (C2 — Snowman fork choice)

`ava-evm` implements the `ChainVm` and `Block` traits from `07-vm-framework.md`
(which re-export the `06` `Block` trait). The adapter owns the bridge between
Snowman's decision model and reth's execution. **No reth fork-choice, no reorg
logic** — Snowman is authoritative and acceptance is strictly linear (06 §"Safety":
siblings are rejected, two conflicting blocks never both accepted).

```rust
/// Implements the ChainVm boundary from 07 for the EVM. One per chain
/// (C-Chain or an EVM subnet); the profile differs only in config (§11).
pub struct EvmVm {
    chain_spec: Arc<AvaChainSpec>,                 // §7.4
    evm_config: AvaEvmConfig,                       // ConfigureEvm impl, §7/§8
    state: Arc<FirewoodStateProvider>,              // §5
    blocks: reth_db::DatabaseEnv,                   // headers/bodies/receipts only (§5)
    atomic: Arc<AtomicBackend>,                     // §6
    txpool: Arc<AvaTxPool>,                         // §6.4
    builder: BlockBuilderDriver,                    // §4
    // In-memory tree of processing (verified, not-yet-accepted) blocks,
    // mirroring coreth's `chain` + 06's processing set. Keyed by block hash.
    verified: DashMap<B256, Arc<EvmBlock>>,
    preferred: ArcSwap<B256>,                        // SetPreference target
    last_accepted: ArcSwap<EvmBlockId>,
}

#[async_trait]
impl ChainVm for EvmVm {
    async fn parse_block(&self, bytes: &[u8]) -> Result<Arc<dyn Block>> {
        // Decode the *Ethereum* block (alloy RLP) from the on-wire `bytes`
        // (§9.3). Recover sender, attach atomic txs (extracted from the
        // ExtraData/body per coreth's encoding). Do NOT execute yet.
        let eth = decode_ava_evm_block(bytes, &self.chain_spec)?;
        Ok(Arc::new(EvmBlock::unverified(eth, self.clone_handle())))
    }

    async fn build_block(&self, ctx: Option<&BlockContext>) -> Result<Arc<dyn Block>> {
        self.builder.build_on(self.preferred.load(), ctx).await   // §4
    }

    async fn get_block(&self, id: Id) -> Result<Arc<dyn Block>> { /* mem tree, else blocks db */ }

    async fn set_preference(&self, id: Id) -> Result<()> {
        // Snowman tells us the head to build on. Just record it; no reorg work —
        // unaccepted blocks live in `verified` and share the parent's state view.
        self.preferred.store(Arc::new(id.into()));
        self.txpool.on_head_change(id);   // re-target pending tx base fee/nonce
        Ok(())
    }

    fn last_accepted(&self) -> (Id, u64) { let a = self.last_accepted.load(); (a.id, a.height) }
}
```

### 3.1 `Block::verify / accept / reject`

```rust
impl Block for EvmBlock {
    async fn verify(&self, _t: &CancellationToken) -> Result<()> {
        // 1. syntactic verify (header well-formedness, gas limit per fee rules §7,
        //    extra-data, atomic-tx encoding) — coreth wrappedBlock::syntacticVerify.
        // 2. semantic verify against parent state (the verified-tree parent or
        //    last-accepted): execute the block via reth BlockExecutor (§3.2) on a
        //    *FirewoodStateProvider view at the parent root* + an in-memory overlay.
        // 3. atomic-tx semantic verify + conflict check vs shared memory (§6.5).
        // 4. predicate pass (warp signatures etc., §6.5/§8).
        // 5. compute the post-state root via Firewood `propose` (NOT committed),
        //    assert it equals header.state_root; assert receipts/gas/bloom match.
        // Store the un-committed Firewood proposal handle in `self` for accept.
        self.execute_into_overlay().await
    }

    async fn accept(&self, _t: &CancellationToken) -> Result<()> {
        // Linear accept (06 §accept_preferred_child): parent IS last_accepted.
        // 1. Firewood `proposal.commit()` → durably advances EVM state tip (§5).
        // 2. AtomicBackend: index atomic ops into the atomic trie at this height,
        //    then ApplyToSharedMemory (atomic batch with the state commit) (§6).
        // 3. Persist header/body/receipts to the blocks db; update canonical
        //    height→hash index; set last_accepted.
        // 4. handlePrecompileAccept callbacks (e.g. warp backend) (§8).
        // 5. txpool: drop now-included txs, re-price pending against new base fee.
        self.commit_accept().await
    }

    async fn reject(&self, _t: &CancellationToken) -> Result<()> {
        // Drop the un-committed Firewood proposal + overlay; evict from `verified`.
        // No state was committed, so reject is cheap (matches coreth).
        self.discard().await
    }
}
```

**Reorg handling (C2):** there is none in the EVM sense. Because acceptance is
linear and state is only committed on `accept`, a rejected block's proposal is
simply dropped. Sibling blocks of the same parent each hold an independent
Firewood proposal (proposal-on-proposal is supported, 04 §4.2). This eliminates
the entire class of reth `TreeState`/fork-choice reorg machinery.

### 3.2 Driving reth's executor for verify

```rust
// Inside execute_into_overlay():
let parent_root = self.parent_state_root();                 // [u8;32]
let state_view  = self.vm.state.history_by_state_root(parent_root)?;  // §5
let mut db      = State::builder().with_database(state_view).with_bundle_update().build();

let evm_env = self.vm.evm_config.evm_env(&self.eth.header)?;        // ConfigureEvm
let mut exec = self.vm.evm_config
    .block_executor_factory()
    .create_executor(self.vm.evm_config.evm_with_env(&mut db, evm_env), ctx);

exec.apply_pre_execution_changes()?;                  // + atomic Import pre-hook (§6.3)
for tx in self.eth.body.transactions() {
    exec.execute_transaction(tx)?;                    // revm; receipts accumulate
}
let (_evm, result) = exec.finish()?;                  // BlockExecutionResult
// Translate the revm BundleState → Firewood BatchOps, propose (no commit) → root:
let proposal = self.vm.state.propose_from_bundle(parent_root, db.take_bundle())?;
let computed_root = proposal.root_hash()?;            // pre-commit root (04 §4.2)
ensure_eq!(computed_root, self.eth.header.state_root);
self.pending_proposal.set(proposal);                  // committed on accept
```

---

## 4. On-demand block building (C1)

Vanilla reth builds payloads on a timer via `PayloadBuilderService` +
`PayloadJob` (engine-API driven). coreth instead builds **only when consensus
asks** (`BuildBlock`) and only when the mempool has work (`block_builder.go`:
`needToBuild`, `signalCanBuild`, `awaitSubmittedTxs`, a min-retry delay). We
reproduce coreth's model, **not** reth's `PayloadBuilderService`.

```rust
pub struct BlockBuilderDriver {
    evm_config: AvaEvmConfig,
    txpool: Arc<AvaTxPool>,
    atomic: Arc<AtomicBackend>,
    state: Arc<FirewoodStateProvider>,
    last_build: Mutex<Option<(B256 /*parent*/, Instant)>>,  // min-delay guard
}

impl BlockBuilderDriver {
    /// Called from ChainVm::build_block. Uses reth's BlockBuilder (the
    /// open→execute→finish flow) but seeded/triggered by us, not a job loop.
    pub async fn build_on(&self, parent: B256, ctx: Option<&BlockContext>)
        -> Result<Arc<dyn Block>>
    {
        let attrs = self.next_block_attrs(parent, ctx)?;       // timestamp, recipient, gas limit
        let parent_hdr = self.state.header(parent)?;
        let evm_env = self.evm_config.next_evm_env(&parent_hdr, &attrs)?;  // §7 fees
        let view = self.state.history_by_state_root(parent_hdr.state_root)?;
        let mut db = State::builder().with_database(view).with_bundle_update().build();
        let mut builder = self.evm_config
            .create_block_builder(&mut db, &parent_hdr, attrs.clone());

        builder.apply_pre_execution_changes()?;

        // 1. Atomic txs first (coreth includes one atomic tx set per block,
        //    gas-limited; §6). Pull from the atomic mempool, pre-flight, inject
        //    EVM state transfer outputs.
        let atomic_txs = self.atomic.mempool.next_batch(&evm_env)?;
        builder.apply_atomic_pre(&atomic_txs)?;                // §6.3 hook

        // 2. EVM txs from the pool, ordered by effective tip, until gas/blockgascost
        //    budget (§7.3) is hit.
        for tx in self.txpool.best_transactions(&evm_env) {
            match builder.execute_transaction(tx) {
                Ok(_) => {}
                Err(BlockExecutionError::Gas(_)) => break,     // block full
                Err(e) if e.is_invalid_tx() => { self.txpool.remove_invalid(&tx); }
                Err(e) => return Err(e.into()),
            }
        }
        let outcome = builder.finish(&self.state)?;            // assembles SealedBlock + root via §5
        let block = assemble_ava_block(outcome, atomic_txs)?;  // attach atomic txs to body/extra
        *self.last_build.lock() = Some((parent, Instant::now()));
        Ok(Arc::new(EvmBlock::built(block, /* proposal kept for accept */ )))
    }
}
```

`waitForEvent`/`signalCanBuild` (coreth) map to: the txpool/atomic-mempool emit a
`tokio::sync::Notify` when they go non-empty; `ava-engine` calls `build_block`
which respects `minBlockBuildingRetryDelay`. The "pending block" notion is served
read-only from the latest built-but-unaccepted block.

---

## 5. Firewood-ethhash as reth's state backend (C7 — the 04 contract)

This is the load-bearing integration. reth's `StateProvider` is a *super-trait*
bundle:
`BlockHashReader + AccountReader + BytecodeReader + StateRootProvider +
StorageRootProvider + StateProofProvider + HashedPostStateProvider + Send + Sync`.
The **state root is computed by `StateRootProvider`** — so by supplying our own
`StateProvider` impl, we route the root through **Firewood-ethhash** instead of
reth's MPT-over-MDBX.

```rust
/// Backs reth state reads/roots with Firewood (ethhash). 04 §4.3 is the contract.
pub struct FirewoodStateProvider {
    db: firewood::db::Db,            // features = ["ethhash"] (Keccak/MPT, 04 §4.2)
    bytecode: Arc<dyn Database>,     // code-hash → bytecode (ava-database, RocksDB)
    block_hashes: Arc<dyn Database>, // number → hash (for BLOCKHASH opcode window)
}

/// A read view pinned at a specific state root (parent or historical revision).
pub struct FirewoodStateView { rev: firewood::db::Revision }

impl AccountReader for FirewoodStateView {
    fn basic_account(&self, addr: &Address) -> ProviderResult<Option<Account>> {
        // Firewood ethhash stores the account node at the account depth keyed by
        // keccak(addr); value is the RLP account {nonce, balance, code_hash, storage_root}.
        Ok(self.rev.val(keccak(addr))?.map(decode_rlp_account))
    }
}
impl StateProvider for FirewoodStateView {
    fn storage(&self, addr: Address, slot: B256) -> ProviderResult<Option<StorageValue>> {
        Ok(self.rev.val(storage_key(addr, slot))?.map(decode_rlp_u256))
    }
    fn bytecode_by_hash(&self, code_hash: &B256) -> ProviderResult<Option<Bytecode>> { /* code db */ }
}

impl StateRootProvider for FirewoodStateView {
    /// reth hands us the post-execution HashedPostState; we translate to Firewood
    /// BatchOps, `propose` against this revision, and return the *pre-commit* root.
    fn state_root(&self, hashed: HashedPostState) -> ProviderResult<B256> {
        let ops = hashed_post_state_to_batchops(&hashed);   // RLP accounts + storage slots
        let proposal = self.rev.propose(ops)?;              // 04 §4.2 (no commit)
        Ok(proposal.root_hash()?.into())
    }
    fn state_root_with_updates(&self, hashed: HashedPostState)
        -> ProviderResult<(B256, TrieUpdates)> {
        // We return the root + an opaque handle wrapping the Firewood proposal so
        // accept() can commit it. TrieUpdates is reth-MPT-shaped; we keep our own
        // proposal in the EvmBlock instead and feed reth an empty/ignored TrieUpdates.
        let p = self.rev.propose(hashed_post_state_to_batchops(&hashed))?;
        Ok((p.root_hash()?.into(), TrieUpdates::default()))  // see gap G1
    }
}
// StorageRootProvider / StateProofProvider → Firewood sub-trie roots + range/change
// proofs (serve eth_getProof and state sync, §10). HashedPostStateProvider →
// our keccak hashing (matches Firewood ethhash key derivation).
```

### 5.1 What reth-DB still owns

- **Blocks/headers/bodies/receipts/logs** and the number↔hash index live in
  `reth-db` (MDBX) + `reth-nippy-jar` static files (04 §4.4 lists these as the
  EVM ancient/static-file store). These are *not* network-consensus on-disk
  formats, so reproducing Go's exact layout is **not required** (overview §1; a
  migrated Go data dir is an import concern, §10/§12).
- **State (accounts/storage) is Firewood-only.** We never write reth's
  `PlainState`/`HashedState`/`Trie` MDBX tables; `FirewoodStateProvider` shadows
  those reads.

### 5.2 Gaps (see §12)

- **G1:** reth's `state_root_with_updates` returns reth-shaped `TrieUpdates`; we
  bypass them (commit via our own Firewood proposal). We must ensure no reth
  code path *writes* those `TrieUpdates` back to MDBX as state-of-record. Wrapper:
  use the bare `Executor`/`BlockBuilder` flow (we control commit) and never call
  reth's `StateWriter::write_state`/`UnifiedStorageWriter` for the trie tables.
- **G2:** reth assumes `StateProviderFactory::history_by_block_number` can
  reconstruct any historical state from MDBX. We back it with Firewood's bounded
  **revision window** (04 §4.1/§4.4). Outside the window → `ProviderError` mapped
  to the same "pruned/unavailable" RPC error coreth returns.

---

## 6. Atomic transactions, shared memory, atomic trie (C3 — the 07 contract)

The biggest divergence from vanilla Ethereum. AVAX moves between C-Chain and X/P
via **atomic ImportTx/ExportTx** through **shared memory** (the `chains/atomic`
package — `07-vm-framework.md` owns the shared-memory + `atomic.Requests`
{`PutRequests`/`RemoveRequests`} contract). We re-derive coreth's
`plugin/evm/atomic` on reth.

### 6.1 Atomic tx types (`ava-evm::atomic`)

Byte-exact with coreth (these are network-consensus formats — overview §1,
compatibility table):

```rust
/// EVMOutput/EVMInput: linear-codec (ava-codec) serialized, NOT RLP.
pub struct EvmOutput { pub address: Address, pub amount: u64, pub asset_id: Id }       // serialize order: addr, amount, asset
pub struct EvmInput  { pub address: Address, pub amount: u64, pub asset_id: Id, pub nonce: u64 }

pub struct UnsignedImportTx {
    pub network_id: u32,
    pub blockchain_id: Id,
    pub source_chain: Id,                       // X or P
    pub imported_inputs: Vec<TransferableInput>, // avax components (07/09)
    pub outs: Vec<EvmOutput>,                    // credited into EVM state
}
pub struct UnsignedExportTx {
    pub network_id: u32,
    pub blockchain_id: Id,
    pub destination_chain: Id,
    pub ins: Vec<EvmInput>,                      // debited from EVM state
    pub exported_outputs: Vec<TransferableOutput>,
}
pub enum AtomicTx { Import(SignedTx<UnsignedImportTx>), Export(SignedTx<UnsignedExportTx>) }
```

Constants reproduced verbatim (cite Go path in doc-comments): `X2CRate =
1_000_000_000` (nAVAX→wei, 9→18 decimals), the per-byte gas constants
`TxBytesGas`, `EVMOutputGas`, `EVMInputGas` (used by the atomic dynamic-fee cost,
§7.3), `secp256k1fx::CostPerSignature`.

`AtomicTx::atomic_ops()` → `(chain_id, atomic::Requests)` exactly as Go:
ImportTx → `RemoveRequests = utxoIDs` on the source chain; ExportTx →
`PutRequests = elems` on the destination chain. These feed the shared-memory
atomic batch on accept (07).

### 6.2 Where atomic txs live in a block

coreth packs atomic txs into the block (encoded in the EVM block's `ExtraData` /
a side region of the body, version-gated by fork). `ava-evm` reproduces that
encoding in `decode_ava_evm_block`/`assemble_ava_block` (§9.3) so block hashes
match. Atomic txs are **not** revm transactions — they are applied via a hook.

### 6.3 Executor hook — `EVMStateTransfer`

revm/reth have no notion of "credit an account from outside the EVM." coreth does
this with `EVMStateTransfer(state)`: ImportTx adds balance to `Outs` addresses;
ExportTx debits `Ins` addresses (and bumps nonces). We implement it as a
**pre-execution state mutation** on the `BlockExecutor`'s `State` DB, before any
EVM transaction runs:

```rust
impl AtomicStateHook {
    /// Applied in apply_pre_execution_changes (verify) and apply_atomic_pre (build).
    fn apply(&self, txs: &[AtomicTx], db: &mut impl reth_revm::Database) -> Result<()> {
        for tx in txs {
            match tx {
                AtomicTx::Import(t) => for o in &t.outs {
                    // credit wei = amount * X2CRate (AVAX asset only; other assets
                    // via the nativeasset path, coreth/nativeasset).
                    db.increment_balance(o.address, o.amount as u128 * X2C_RATE)?;
                }
                AtomicTx::Export(t) => for i in &t.ins {
                    db.decrement_balance(i.address, i.amount as u128 * X2C_RATE)?;
                    db.set_nonce(i.address, db.nonce(i.address)? .max(i.nonce + 1))?; // matches coreth
                }
            }
        }
        Ok(())
    }
}
```

This mutates the same `State` overlay whose `BundleState` becomes the Firewood
proposal (§3.2/§5), so the EVM state root *includes* the atomic effects — exactly
as coreth.

### 6.4 Atomic mempool + atomic backend + atomic trie

- **`AtomicMempool`** (coreth `atomic/txpool`): a separate pool from the EVM
  txpool; ordered, dedup by source UTXO, conflict-checked against pending. The
  builder pulls **one atomic batch per block** (§4). A `gossip::Gossipable` so
  atomic txs gossip over the p2p SDK (05). Reproduce the heap ordering + the
  `discardedTxs`/`issuedTxs` lifecycle.
- **`AtomicBackend` + `AtomicState`** (coreth `atomic/state`): on accept, indexes
  the block's atomic ops into the **atomic trie** at the block height and applies
  the shared-memory batch.
- **`AtomicTrie`** (coreth `atomic/state/atomic_trie.go`): a **separate
  Merkle-Patricia trie** keyed `height(8B) || blockchainID(32B)` →
  serialized atomic requests, with `TrieKeyLength = 8 + 32`, periodic commits at
  `commitInterval`, `lastCommittedRoot`/`lastAcceptedRoot`. **Decision:** back the
  atomic trie with **Firewood-ethhash** (a second, small Firewood instance) so its
  root matches Go's MPT root and it shares the propose/commit discipline; its root
  is checkpointed and used for atomic-state sync (§10). Keep the exact key
  encoding and `EmptyRootHash` initialization.

### 6.5 Atomic semantic verify, conflicts, predicates (C10)

- **Conflict check:** an atomic tx is invalid if its input UTXOs are already
  spent in shared memory or by another atomic tx in the same/ancestor block. Port
  coreth's conflict set (`set.Set[ids.ID]` of consumed UTXO IDs) checked across
  the verified-block ancestry.
- **Bonus blocks:** reproduce the `bonusBlocks` height→ID skip-set (historical
  mainnet quirk) verbatim.
- **Predicates** (warp): before EVM execution, verify per-tx precompile
  predicates (e.g. a warp message's BLS aggregate signature) using the
  `PredicateContext` (proposervm block context / P-Chain height). This is the
  coreth `predicate` + `verifyPredicates` pass; in `ava-evm` it runs in
  `apply_pre_execution_changes` and stores results for the warp precompile to read
  (§8). The proposervm block context arrives via `Block::verify_with_context`
  (06 §`ShouldVerifyWithContext`).

---

## 7. Dynamic fees & gas (C4/C5)

### 7.1 Fee history per fork

Avalanche replaced Ethereum's fee model in stages (reproduce *exactly* — these
set block validity):

- **Pre-AP3:** base fee is `nil` (legacy gas pricing).
- **AP3+ (`customheader/base_fee.go` `baseFeeFromWindow`):** an EIP-1559-like base
  fee computed from a **rolling fee window** (`dynamic_fee_windower.go`) over the
  last ~10s of gas usage vs `TargetGas`, bounded by `MinBaseFee`, adjusted by
  `BaseFeeChangeDenominator`. Port the window math bit-for-bit (integer only).
- **AP4+ (`block_gas_cost.go`):** an additional **block gas cost** (`MinBlockGasCost`,
  `MaxBlockGasCost`, `BlockGasCostStep`, `TargetBlockRate`) the block producer must
  cover from priority fees — anti-spam tied to block production rate.
- **Fortuna / ACP-176 (`vms/evm/acp176`, `customheader/dynamic_fee_state.go`):** a
  new gas-price *state machine* (`feeStateBeforeBlock(...).GasPrice()`) replacing
  the window; carries a serialized fee state in the header. ACP-226
  (`vms/evm/acp226`) adds the min-delay-excess mechanism. Port both modules.

### 7.2 Where it plugs into reth

reth derives base fee from `EthereumHardforks`/EIP-1559 inside `next_evm_env`. We
**override** `ConfigureEvm::evm_env`/`next_evm_env` to call our `feerules` module:

```rust
pub struct AvaEvmConfig {
    chain_spec: Arc<AvaChainSpec>,
    executor_factory: AvaBlockExecutorFactory,   // wraps revm + atomic hook + precompiles
    assembler: AvaBlockAssembler,
}
impl ConfigureEvm for AvaEvmConfig {
    type Primitives = AvaEvmPrimitives;          // Ethereum block/receipt + atomic side-data
    type Error = AvaEvmError;
    type NextBlockEnvCtx = AvaNextBlockCtx;      // timestamp(ms), recipient, gas limit, p-chain height
    type BlockExecutorFactory = AvaBlockExecutorFactory;
    type BlockAssembler = AvaBlockAssembler;

    fn next_evm_env(&self, parent: &Header, attrs: &AvaNextBlockCtx)
        -> Result<EvmEnvFor<Self>, Self::Error> {
        let mut env = self.eth_like_env(parent, attrs)?;
        // Override base fee + gas limit with Avalanche fee rules for the active fork:
        env.block_env.basefee = feerules::base_fee(&self.chain_spec, parent, attrs.timestamp_ms)?;
        env.block_env.gas_limit = feerules::gas_limit(&self.chain_spec, parent, attrs)?;
        Ok(env)
    }
    fn block_executor_factory(&self) -> &Self::BlockExecutorFactory { &self.executor_factory }
    fn block_assembler(&self) -> &Self::BlockAssembler { &self.assembler }
    // evm_env, context_for_block, context_for_next_block similarly delegate to feerules.
}
```

### 7.3 Atomic tx cost

Atomic txs consume gas too: `tx.GasUsed(...)` = byte cost (`TxBytesGas`,
`EVMOutputGas`, `EVMInputGas`) and the tx must "pay" the dynamic base fee
(coreth `tx.go` `dynamicFee`, guarded against `nil baseFee` overflow). The builder
counts atomic gas against the block gas/blockGasCost budget (§4). Reproduce the
overflow checks as typed errors (`ErrFeeOverflow`, §11 error model).

### 7.4 Fork schedule as a custom `Hardforks` (C5)

reth's `EthChainSpec`/`Hardforks` is generic over hardfork enums with
`ForkCondition` (timestamp/block). We define an `AvaHardfork` enum interleaving
Ethereum forks (London, Shanghai, Cancun, …) with Avalanche phases and build an
`AvaChainSpec`:

```rust
pub enum AvaHardfork {
    // Ethereum (inherited where coreth maps them):
    Eth(EthereumHardfork),
    // Avalanche phases (all timestamp-activated; cite params/extras):
    ApricotPhase1, ApricotPhase2, ApricotPhase3, ApricotPhase4, ApricotPhase5,
    ApricotPhasePre6, ApricotPhase6, ApricotPhasePost6,
    Banff, Cortina, Durango, Etna, Fortuna, Granite,
}
pub struct AvaChainSpec {
    inner: ChainHardforks,                 // reth ordered fork list (ForkCondition::Timestamp)
    fee_config: FeeConfig,                 // GasLimit, TargetBlockRate, MinBaseFee, TargetGas, BaseFeeChangeDenominator, Min/MaxBlockGasCost, BlockGasCostStep
    is_subnet: bool,                       // C-Chain vs EVM-subnet profile
    network_upgrades: NetworkUpgrades,     // the *uint64 timestamps from params/extras
    // ... chain id, genesis, allocations
}
impl EthChainSpec for AvaChainSpec { /* fork conditions, base_fee_params, etc. */ }
impl AvaChainSpec {
    pub fn is_apricot_phase3(&self, t: u64) -> bool { /* mirrors extras.IsApricotPhase3 */ }
    pub fn is_fortuna(&self, t: u64) -> bool { /* ... */ }
    // one predicate per phase; these gate feerules + precompile activation.
}
```

`network_upgrades.go`'s `checkCompatible` (incompatibility on already-activated
forks) → a Rust `check_compatible` returning the same error shapes. Fork
*timestamps* for Mainnet/Fuji are protocol constants embedded in `ava-version`
(overview §5) and re-exported here.

---

## 8. Warp + subnet-evm stateful precompiles (C6/C10)

### 8.1 The precompile model

subnet-evm's framework (`precompile/`): each precompile is a **`Module`**
(`{ConfigKey, Address, Contract: StatefulPrecompiledContract, Configurator}`),
registered in a global registry, **enabled/configured via genesis + upgrade
JSON**, and able to read/write EVM state (allowlist roles, fee config, minted
balances). coreth/C-Chain ships the **Warp** precompile.

revm exposes precompiles via the **`PrecompileProvider`** trait
(address → precompile fn, on the revm handler). Stateful precompiles need access
to the EVM `State` + the configuration + (for warp) the predicate results. We
implement a custom `PrecompileProvider` that overlays Avalanche precompiles on top
of revm's standard set:

```rust
pub struct AvaPrecompiles {
    base: EthPrecompiles,                       // revm standard precompiles for the active spec
    modules: Arc<PrecompileRegistry>,           // address → Arc<dyn StatefulPrecompile>
    activated: HashSet<Address>,                // gated by fork + upgrade config at this height
}
pub trait StatefulPrecompile: Send + Sync {
    /// `state` is the live revm journaled state; `ctx` exposes block ctx,
    /// caller, value, and predicate results (for warp).
    fn run(&self, input: &[u8], gas: u64, state: &mut dyn PrecompileState, ctx: &PrecompileCtx)
        -> PrecompileResult;
}
impl<CTX> PrecompileProvider<CTX> for AvaPrecompiles {
    fn run(&self, ctx: &mut CTX, addr: &Address, input: &PrecompileInput, gas: u64)
        -> Result<Option<PrecompileOutput>, ...> {
        if let Some(p) = self.activated.get(addr).and_then(|a| self.modules.get(a)) {
            return Ok(Some(p.run(input.data, gas, ctx.state_mut(), &ctx.into())?));
        }
        self.base.run(ctx, addr, input, gas)     // fall through to standard precompiles
    }
    fn contains(&self, addr: &Address) -> bool { self.activated.contains(addr) || self.base.contains(addr) }
}
```

`AvaBlockExecutorFactory` installs `AvaPrecompiles` into the revm handler when
creating each `BlockExecutor` (the `EvmFactory` it owns sets `precompiles =
AvaPrecompiles::for_height(...)`).

### 8.2 The ported precompiles

| Precompile | Source | State it touches |
|---|---|---|
| **Warp** (send/verify Avalanche Warp messages) | coreth/subnet-evm `precompiles/contracts/warp` | reads predicate results (BLS aggregate over a P-Chain validator set at a height); emits/verifies `AddressedCall`/`BlockHashPayload` |
| **AllowList** (deployer/tx allowlists) | `precompile/allowlist` | role storage slots; admin/manager/enabled |
| **FeeManager** | `contracts/feemanager` | live `FeeConfig` overrides |
| **NativeMinter** | `contracts/nativeminter` | mints native balance |
| **RewardManager** | `contracts/rewardmanager` | fee-reward address config |
| **GasPriceManager** | `contracts/gaspricemanager` | gas price params |

Each maps to a `StatefulPrecompile` + a `Module` registered by `ConfigKey`.
**Warp predicate:** verification happens in the pre-execution predicate pass (§6.5)
— the precompile `run` only *reads* the cached verified result (matching coreth's
split between predicate verification and precompile execution). `handlePrecompileAccept`
(§3.1 accept) fires module accept-hooks (e.g. the warp backend records sent
messages for signature serving, coreth `warp/backend.go`).

### 8.3 Configuration

Precompile enablement + params come from **genesis JSON** (`config` block) and
**upgrade JSON** (`upgrades`/`precompileUpgrades`), byte-compatible with
subnet-evm/coreth (§11). At each block, `AvaPrecompiles::for_height` computes the
activated set from the timestamp-keyed upgrade schedule.

---

## 9. RPC + wire format (C9)

### 9.1 `eth_*` namespace

Use reth's `EthApi`/`reth-rpc` for the standard namespaces (`eth`, `net`, `web3`,
`debug`, `txpool`, websocket `eth_subscribe`). reth's `EthApi` is generic over the
provider/pool — we instantiate it over `FirewoodStateProvider` + `AvaTxPool` so
`eth_getBalance`/`eth_call`/`eth_getProof`/`eth_estimateGas` read Avalanche state
and use Avalanche fee rules. Override the fee-related helpers
(`eth_gasPrice`, `eth_feeHistory`, `eth_maxPriorityFeePerGas`) to use `feerules`
(§7). `debug_traceTransaction`/`*_traceBlock*` use revm inspectors (reth's
tracing stack) — reproduce coreth's tracer outputs (incl. the prestate tracer,
`prestate_tracer_test.go`) for parity.

### 9.2 `avax.*` namespace

A custom RPC module (axum/JSON-RPC 2.0 per overview §4.2; reth's RPC uses
`jsonrpsee` — we mount our module alongside or via `ava-api`'s router, §12-node).
Methods mirror coreth's `avax` service:
`avax.issueTx` (submit an atomic tx), `avax.getAtomicTx`, `avax.getAtomicTxStatus`,
`avax.getUTXOs`, `avax.export`/`avax.import` helpers, `avax.getBlockByHeight`. The
admin namespace (`admin.go`) and health (`health.go`) likewise ported.

### 9.3 Block `bytes` wire format

`ChainVm::parse_block`/`Block::bytes` use the **exact coreth block encoding**:
the RLP Ethereum block plus atomic txs (encoded in `ExtraData`/body per the active
fork's rules). This is consensus-critical (block IDs must match across Go/Rust
nodes — overview compatibility table). `decode_ava_evm_block`/`assemble_ava_block`
are golden-tested against Go-produced bytes (§12 test plan).

---

## 10. State sync (C8)

coreth has a snap-like **EVM state sync** (`vms/evm/sync`, coreth
`plugin/evm/syncervm`): a syncing node fetches the accepted state at a recent
block via range/leaf requests, plus the **atomic trie** state. Mapping:

- **EVM account/storage state →** served from **Firewood range/change proofs**
  (04 §4.2/§4.3, the Go `firewood/syncer` is the reference). The wire protocol
  (leaf request/response over the p2p SDK, 05) is reproduced byte-exact; the
  server answers from a Firewood historical revision, the client reconstructs a
  Firewood trie and verifies the root.
- **Atomic trie state →** synced the same way over the atomic Firewood instance
  (§6.4), then `ApplyToSharedMemory` from the synced cursor.
- **Block sync →** headers/bodies/receipts backfilled into reth-db.
- **Accepted-block index / metadata →** ported from coreth's sync adapter.

reth's own staged sync / snap sync is **not** used (no engine, §1). We implement
the Go protocol directly on top of Firewood proofs.

---

## 11. Genesis, upgrade config, error model, Go→Rust map

### 11.1 Genesis / upgrade JSON

- **C-Chain genesis** (`genesis/`): the embedded genesis JSON (chain config +
  alloc) must reproduce identical genesis block IDs for Mainnet/Fuji (overview
  compatibility table). `ava-genesis` (12) owns embedding; `ava-evm` parses the
  `config` (chain id, fork timestamps, `feeConfig`, precompile configs).
- **Upgrade config** (`upgrade.json`, `precompileUpgrades`): timestamp-keyed
  precompile enable/disable + param changes; same schema as subnet-evm/coreth.
  Parsed into the `AvaChainSpec` upgrade schedule (§7.4/§8.3).

### 11.2 Error model

Per overview §7.1: `ava-evm::Error` (thiserror) preserves coreth/atomic sentinels
as variants and matches on them where Go uses `errors.Is`:
`ErrWrongNetworkID`, `ErrNilTx`, `ErrNoValueOutput/Input`, `ErrNoGasUsed`,
`errNilBaseFee`, `ErrFeeOverflow`, `ErrConflictingAtomicInputs`, predicate errors,
plus reth/revm execution errors wrapped via `#[from]`. All arithmetic on
balances/fees is **checked** (overflow → typed error, never wrap — overview §6.1).

### 11.3 Go→Rust mapping (non-obvious)

| Go (coreth/subnet-evm) | Rust (`ava-evm`) |
|---|---|
| `core.BlockChain` (consensus-driven) | reth `BlockExecutor`/`BlockBuilder` + our canonical-head tracking (§3) |
| `miner.Miner.GenerateBlock` | `BlockBuilderDriver::build_on` (§4) |
| `wrappedBlock` Verify/Accept/Reject | `EvmBlock` impl of 06 `Block` (§3.1) |
| `EVMStateTransfer(stateDB)` | `AtomicStateHook::apply` on revm `State` (§6.3) |
| `atomic.Requests{Put,Remove}` | `atomic::Requests` (07 shared-memory contract) |
| `triedb.Database` (atomic trie) | second Firewood-ethhash instance (§6.4) |
| `params.ChainConfig` + `extras` | `AvaChainSpec` + `AvaHardfork` (§7.4) |
| `precompile/modules.Module` | `StatefulPrecompile` + `PrecompileRegistry` (§8) |
| `PredicateContext` | `PrecompileCtx` predicate results (§6.5) |
| `eth.APIs()` | reth `EthApi` over Firewood/AvaTxPool (§9.1) |
| `txpool.TxPool` | `AvaTxPool` (reth pool + Avalanche `PoolTransaction` ordering) |
| `gossip.Gossipable` (atomic tx) | `ava-network` p2p SDK gossip (05) |

---

## 12. Honest gaps & wrapper designs (reth limits) — summary

This is the make-or-break part of the EVM port: for every Avalanche behavior reth
does **not** natively cover, `ava-evm` builds a concrete *gap wrapper* around a
reth/revm primitive. The table below is the index; **§17 is the normative,
implementable design** for each wrapper (struct fields, the exact trait methods
overridden, the type conversions, and a per-gap effort/risk + "pure wrapper vs
needs-upstream" verdict). Where §17 and this table differ in detail, §17 governs.

> **reth revision assumption (binding for §17 signatures).** All trait
> signatures in §17 are written against **reth ≈ v2.x (`reth` 2.2.0, April 2026)
> + `revm` ≥ 24** — the post-1.3 `ConfigureEvm` unification (one trait owning the
> `BlockExecutorFactory`, `BlockAssembler`, and `BlockBuilder` flow), the post-2.0
> Storage-V2 + `SparseTrieCache` provider stack, and the revm `PrecompileProvider`
> shape `run(&mut self, ctx, &CallInputs) -> Result<Option<Output>, String>`. The
> **exact revision is pinned and vendored** (G0); these signatures are the *shape*
> we wrap, and the `AvaEvm` facade (§17.1) is precisely the seam that absorbs the
> next bump. Version-sensitive lines are flagged inline with `// vN-sensitive`.

| ID | Gap (one-liner) | Verdict | Detailed design |
|---|---|---|---|
| **G0** | reth library crates have **no stable API**; no first-class external-consensus entrypoint | **pure wrapper** (vendoring + facade) | §17.1 |
| **G1** | `StateRootProvider`/`StateWriter` route the root + commit through reth-MPT/MDBX | **pure wrapper** | §17.2 |
| **G2** | dynamic fees: reth base-fee is EIP-1559-fixed; gas charged for atomic txs | **pure wrapper** | §17.3 |
| **G3** | atomic Import/Export are **not** revm txs; shared memory + atomic trie | **pure wrapper** | §17.4 |
| **G4** | revm `PrecompileProvider` is stateless; warp needs predicate results + block ctx | **pure wrapper** | §17.5 |
| **G5** | on-demand build path is the engine `PayloadBuilderService` | **pure wrapper** | §17.6 |
| **G6** | Snowman owns fork choice; no reth pipeline/reorg/canonicalization | **pure wrapper** | §17.7 |
| **G7** | Avalanche fork schedule layered on Ethereum forks; per-block revm spec id | **pure wrapper** | §17.8 |
| **G8** | EVM + atomic state-sync protocol; `avax.*` RPC + `eth_*` overrides | **pure wrapper** (+1 *soft* upstream ask) | §17.9 |

**Net verdict:** every gap is closeable with **pure wrappers** against the
vendored reth revision — *no hard upstream reth change is required to ship*. Two
**soft** upstream asks would reduce maintenance (G1: a public "bring-your-own
state-root/commit" seam so we needn't keep `TrieUpdates` empty by convention; G8:
a stable `EthApi` builder over a third-party provider). Both are *nice-to-haves*,
not blockers; until then the facade (G0) carries the churn. The dominant residual
risk is **G0/R3** (reth API drift), not any single missing primitive.

> Previously-listed "G7 = reading a Go data dir" is **not** a reth gap — it is the
> cross-cutting migration concern tracked as **R2** (overview §11.2) and §5.1/§10;
> it has been removed from the G-series so each Gn maps to one reth wrapper. The
> renumbered G6/G7/G8 above (fork choice / hardforks / sync+RPC) replace it.

---

## 13. Crate layout (`ava-evm`)

Internal modules mirroring the Go reference (optionally sub-crates if build times
demand it, per overview §3 SAE precedent):

```
ava-evm/
├── vm.rs            # EvmVm: ChainVm/Block adapter (§3)
├── block.rs         # EvmBlock, decode/assemble wire bytes (§3, §9.3)
├── builder.rs       # BlockBuilderDriver, on-demand build (§4)
├── evmconfig.rs     # AvaEvmConfig (ConfigureEvm), AvaBlockExecutorFactory (§7/§8)
├── feerules/        # AP3 window, AP4 block gas cost, Fortuna/ACP-176, ACP-226 (§7)
├── chainspec.rs     # AvaChainSpec, AvaHardfork, upgrade schedule (§7.4, §11)
├── state.rs         # FirewoodStateProvider + views (§5)
├── atomic/          # tx types, mempool, backend, atomic trie, state hook (§6)
├── precompile/      # AvaPrecompiles + registry + warp/allowlist/feemanager/… (§8)
├── rpc/             # eth_* wiring + avax.* module + admin/health (§9)
├── sync/            # EVM + atomic-trie state sync over Firewood proofs (§10)
└── error.rs         # Error enum, sentinels (§11.2)
```

---

## 14. Test plan (references `02-testing-strategy.md`)

1. **EVM conformance:** run the **Ethereum execution-spec state tests &
   blockchain tests** through reth's existing test harness against
   `AvaEvmConfig` (with Avalanche forks mapped to their Ethereum equivalents) —
   guarantees revm-level correctness for free.
2. **State-root parity (differential vs Go):** replay a range of Mainnet/Fuji
   C-Chain blocks; assert every block's Firewood-ethhash state root equals
   coreth's, block-for-block. This is the single most important test (the 04
   contract). Property-test `HashedPostState → BatchOps` translation.
3. **Atomic-tx differential:** for a corpus of ImportTx/ExportTx, assert
   byte-identical tx serialization, identical `atomic.Requests`, identical
   post-`EVMStateTransfer` balances/nonces, and identical atomic-trie roots vs Go.
   Shared-memory effects checked against the 07 harness.
4. **Block-bytes golden:** `decode_ava_evm_block`/`assemble_ava_block` round-trip
   Go-produced block bytes; block IDs match (consensus-critical).
5. **Fee parity:** golden vectors for base fee (AP3 window), block gas cost (AP4),
   Fortuna/ACP-176 gas-price state across fork boundaries vs `customheader`.
6. **Precompile parity:** warp message verify/produce, allowlist role
   transitions, feemanager/nativeminter/rewardmanager state changes vs subnet-evm.
7. **RPC parity:** golden `eth_*`/`debug_*`/`avax.*` request→response vs a Go node
   (incl. prestate tracer, `eth_getProof` from Firewood).
8. **State-sync interop:** a Rust node syncs EVM + atomic state from a Go node and
   vice-versa; resulting roots match.
9. **Consensus integration:** drive `EvmVm` from the `ava-engine` test harness
   (06); verify/accept/reject ordering, linear-acceptance invariant, reject drops
   the Firewood proposal without committing.

---

## 15. Performance notes / improvements over Go

Each is gated on a differential test (#2/#3 above) proving identical external
behavior (overview §9, §6.1):

- **Firewood async commit / pipelined propose:** compute the next block's state
  root via `propose` while a background thread `commit`s the accepted block
  (04 §4.2). Decouples consensus voting latency from durable trie writes — the
  same lever SAE (11) uses. Safe: roots are deterministic and computed pre-commit.
- **reth/revm parallelism:** revm's optimized interpreter + reth's parallel
  state-root and (where applicable) parallel tx execution outrun coreth/libevm's
  geth-derived serial path. Tx-level parallelism only where independence is
  provable (disjoint state) — otherwise serial to preserve ordering.
- **Zero-copy block decode:** `alloy` RLP + `bytes::Bytes` on the parse path; no
  re-encode round-trips that Go incurs.
- **Sharded txpool reads / lock-free best-tx iteration** for build (§4), bounded
  by determinism of inclusion order.
- **Batch BLS verification** for warp predicates (cross-ref `ava-crypto`, `blst`
  aggregate verify) — independent signatures verified in parallel.

---

## 16. Cross-spec contracts (binding)

- **With 04 (Firewood):** EVM state root = Firewood `propose().root_hash()` with
  `features=["ethhash"]`; commit on `Block::accept`; reads via Firewood revisions;
  range/change proofs serve `eth_getProof` + state sync. reth-db owns only
  block/receipt storage, never state-of-record. (§5)
- **With 07 (VM framework / shared memory):** `ava-evm` implements the `ChainVm`/
  `Block` traits; atomic Import/Export produce `atomic.Requests`
  ({`PutRequests`/`RemoveRequests`}) applied to **shared memory** in the same
  atomic batch as the state commit on accept; the atomic mempool gossips via the
  p2p SDK. (§3, §6)
- **With 06 (consensus):** acceptance is linear; Reject drops an un-committed
  Firewood proposal; proposervm block context feeds the predicate/warp pass via
  `verify_with_context`. (§3, §6.5)
- **Integration mode (binding):** reth is embedded **as a library executor**, not
  via the Engine API; Snowman owns fork choice. (§1)
- **Reuse contract with 11 (SAE):** there is **one EVM engine, two drivers**. SAE's
  `cchain` (spec 11) is the asynchronous (ACP-194) driver and reuses this crate's
  revm executor and Firewood-ethhash state layout. Therefore `ava-evm` MUST expose,
  as stable public APIs:
  1. the revm `BlockExecutor`/block-executor-factory (`AvaEvmConfig` and the
     execution entrypoint) — usable to execute an ordered batch of txs against a
     given parent state and obtain receipts + the post-state root, **decoupled** from
     this crate's synchronous `ChainVm` lifecycle; and
  2. the `FirewoodStateProvider` (the reth `StateProvider`/`StateRootProvider`
     wrapper, §5) and the propose/commit handles,
  so `ava-saevm-exec` can drive execution behind the consensus frontier without
  re-implementing the EVM. The two drivers differ only in block lifecycle, never in
  EVM semantics or on-disk state format. (cross-ref `11` §"cchain reuse"; `00` §11.1.5)

---

## 17. Gap wrappers — concrete designs (G0–G8)

This section is the **normative, implementable** elaboration of §12. Each gap has:
*the gap* (what reth does not provide), *the reth primitive(s)* (exact crate/trait
paths + the v2.x version assumption), *the wrapper design* (Rust sketches detailed
enough to implement), and an *effort/risk + verdict* note. §17.10 lists the public
**reusable API surface** for SAE (11), satisfying §16's reuse contract.

> **Convention.** `// vN-sensitive` marks a line whose exact spelling tracks the
> pinned reth/revm revision; the `AvaEvm` facade (§17.1) is where those re-spell on
> a bump. `ProviderResult<T> = Result<T, reth_storage_errors::ProviderError>`.
> `B256 = alloy_primitives::B256`. Imports are abbreviated in sketches.

### 17.1 G0 — no stable external-consensus entrypoint → the `AvaEvm` facade crate

**The gap.** reth's library crates (`reth-evm`, `reth-provider`/`reth-storage-api`,
`reth-rpc`, `reth-transaction-pool`, `reth-chainspec`, `revm`) carry **no semver
stability guarantee**, and reth ships **no first-class "external consensus" entry
point** — it expects either the Engine API or its own staged-sync/`NodeBuilder`.
Between the trait set the rest of this spec was first drafted against and reth
v2.x, real churn already happened: `ConfigureEvm` absorbed `ConfigureEvmEnv` +
`BlockExecutionStrategyFactory` (1.3), `BlockExecutor` split into
`execute_transaction_without_commit` + `commit_transaction`, `revm`'s
`PrecompileProvider::run` changed shape, and Storage-V2/`SparseTrieCache` reshuffled
the provider stack (2.0). Any of these can move again.

**reth primitives involved.** *All of them* — this gap is meta. The facade pins:
`reth-evm`, `reth-ethereum`, `reth-provider`, `reth-storage-api`,
`reth-chainspec`, `reth-ethereum-forks`, `reth-rpc`, `reth-rpc-eth-api`,
`reth-transaction-pool`, `reth-primitives`, `revm`, `alloy-*`, all at **one
git SHA** (not a version range).

**Wrapper design — `ava-evm-reth` facade sub-crate.** All reth touch-points go
through a thin facade crate that (a) depends on the *pinned* reth SHA, (b)
re-exports a **minimal, stable internal API surface** (`ava-evm` and
`ava-saevm-exec` depend only on the facade, never on `reth_*` directly), and (c)
contains every `impl Trait for reth::…` so a reth bump localizes here.

```toml
# ava-evm-reth/Cargo.toml — the ONLY crate allowed to name reth/revm directly.
[dependencies]
reth-evm         = { git = "https://github.com/paradigmxyz/reth", rev = "<PINNED_SHA>" }
reth-provider    = { git = "https://github.com/paradigmxyz/reth", rev = "<PINNED_SHA>" }
reth-storage-api = { git = "https://github.com/paradigmxyz/reth", rev = "<PINNED_SHA>" }
reth-chainspec   = { git = "https://github.com/paradigmxyz/reth", rev = "<PINNED_SHA>" }
revm             = { git = "https://github.com/bluealloy/revm",   rev = "<PINNED_SHA>" }
# ... all other reth-* / alloy-* pinned to the same matching set.
```

```rust
//! ava-evm-reth/src/lib.rs — the stable internal seam (G0 boundary).
//! Re-export ONLY the reth/revm items the rest of ava-evm is allowed to see,
//! under our own names, so an upstream rename is a one-line edit here.
pub use reth_evm::{ConfigureEvm, ConfigureEvmFor, EvmEnvFor, ExecutionCtxFor};
pub use reth_evm::execute::{BlockExecutor, BlockExecutorFactory, BlockBuilder,
    BlockBuilderOutcome, BlockExecutionResult, BlockExecutionError};
pub use reth_storage_api::{StateProvider, StateRootProvider, StorageRootProvider,
    StateProofProvider, HashedPostStateProvider, AccountReader, BytecodeReader,
    BlockHashReader, StateProviderFactory};
pub use revm::context::PrecompileProvider;                 // vN-sensitive (was handler::)
pub use revm::database::{State, BundleState, StateBuilder}; // vN-sensitive
pub use revm::state::{Account as RevmAccount, AccountInfo, Bytecode};

/// The ONE entrypoint SAE (11) and the sync ChainVm (§3) both call. This is the
/// "external consensus executor" reth doesn't ship: execute an ordered batch of
/// txs (+ an atomic pre-hook) against a parent state view, return receipts and a
/// revm BundleState — with NO node, NO engine, NO fork choice attached.
pub trait ExternalConsensusExecutor: Send + Sync {
    type State;          // a reth StateProvider-backed `State<DB>` view (§17.2)
    /// Pure function of (env, parent view, ordered txs, pre-hook). Used by BOTH
    /// the sync verify path (§3.2) and SAE's streaming executor (11 §6.1).
    fn execute_batch(
        &self,
        env: AvaEvmEnv,
        parent: &mut Self::State,
        pre_hook: &dyn PreExecutionHook,      // atomic Import/Export (§17.4)
        txs: &[Recovered<TxEnvelope>],
    ) -> Result<ExecOutcome, AvaEvmError>;
}

/// Receipts + the unflushed state delta; the caller turns `bundle` into a
/// Firewood proposal (§17.2). Decoupled from any block lifecycle.
pub struct ExecOutcome {
    pub result: BlockExecutionResult<Receipt>,   // receipts, gas_used, requests
    pub bundle: BundleState,                      // revm state delta (→ Firewood)
}
```

`AvaEvmConfig` (§7.2) implements `ExternalConsensusExecutor` by driving the
reth `BlockExecutor` (§17.4 sketch). Crucially, **the facade is the reuse seam of
§16**: spec 11's `ava-saevm-exec` depends on `ava-evm-reth` (the facade) +
`ava-evm`'s executor, *not* on reth — so SAE inherits the same pinned revision and
the same wrappers (§17.10).

**A reth bump checklist** lives in `ava-evm-reth/UPGRADING.md`: (1) move the SHA;
(2) fix compile errors *only inside the facade* (re-export renames + the ~9 trait
impls below); (3) re-run the §14 differential suite (state-root parity is the
gate). The blast radius is one crate.

**Effort/risk.** Effort: **moderate-ongoing** (the facade itself is small; the cost
is the per-bump maintenance tax). Risk: **highest in the spec (R3)** — but it is
*contained*, not eliminated. **Verdict: pure wrapper** (vendoring + facade); a
*soft* upstream ask is reth publishing a blessed "library executor" entrypoint, at
which point the facade shrinks to re-exports.

### 17.2 G1 — bypass reth TrieUpdates/StateWriter → Firewood state root & commit

**The gap.** reth computes the post-state root via `StateRootProvider` over its own
MPT and **persists** it (and the trie/plain/hashed tables) via `StateWriter` /
`UnifiedStorageWriter` into MDBX (Storage-V2 now via `SparseTrieCache`). Avalanche's
EVM state root and durable commit MUST instead go through **Firewood-ethhash** (04
§4.2/§4.3; the 04 contract, §16). reth provides the *seam* (we supply a
`StateProvider`) but not the behavior (it still wants to own the commit).

**reth primitives involved (`reth-storage-api`, `reth-trie`, `revm` v2.x).**

```rust
// reth-storage-api: the super-trait we must fully implement.
pub trait StateProvider:
    BlockHashReader + AccountReader + BytecodeReader
    + StateRootProvider + StorageRootProvider + StateProofProvider
    + HashedPostStateProvider + Send + Sync {
    fn storage(&self, account: Address, key: StorageKey)
        -> ProviderResult<Option<StorageValue>>;
    // + provided: account_code / account_balance / account_nonce
}
pub trait StateRootProvider: Send + Sync {
    fn state_root(&self, hashed: HashedPostState) -> ProviderResult<B256>;
    fn state_root_from_nodes(&self, input: TrieInput) -> ProviderResult<B256>;
    fn state_root_with_updates(&self, hashed: HashedPostState)
        -> ProviderResult<(B256, TrieUpdates)>;                      // vN-sensitive
    fn state_root_from_nodes_with_updates(&self, input: TrieInput)
        -> ProviderResult<(B256, TrieUpdates)>;
}
pub trait HashedPostStateProvider: Send + Sync {
    fn hashed_post_state(&self, bundle: &BundleState) -> HashedPostState;
}
// revm State<DB>: the journaled overlay whose delta we convert.
// let mut db = State::builder().with_database(view).with_bundle_update().build();
// after execution: let bundle: BundleState = db.take_bundle();        // vN-sensitive
```

**Wrapper design — `FirewoodStateProvider` + `FirewoodStateCommitter`.**
The provider answers reads/roots from a Firewood revision; the committer owns the
propose→commit lifecycle. The crux: we **never** let reth write trie tables — we
keep our own Firewood `Proposal` and feed reth an **empty `TrieUpdates`** (or hand
`BlockBuilder::finish` a *precomputed* `(root, TrieUpdates::default())`, see §17.6).

```rust
/// Factory: opens read views at any retained revision and proposes/commits.
pub struct FirewoodStateProvider {
    db: firewood::db::Db,             // features=["ethhash"] (04 §4.2)
    bytecode: Arc<dyn DynDatabase>,   // code_hash -> bytecode (ava-database)
    block_hashes: Arc<dyn DynDatabase>, // number -> hash (BLOCKHASH window)
}

/// A read view pinned at one state root (parent / historical revision).
pub struct FirewoodStateView {
    rev: firewood::db::Revision,      // db.view() (tip) or db.revision(root) (04 §4.2)
    provider: Arc<FirewoodStateProvider>,
}

impl AccountReader for FirewoodStateView {
    fn basic_account(&self, addr: &Address) -> ProviderResult<Option<Account>> {
        // ethhash stores the account node keyed by keccak(addr); value = RLP
        // {nonce, balance, code_hash, storage_root}.
        Ok(self.rev.val(account_key(addr)).map_err(map_fw_err)?
              .map(|rlp| decode_rlp_account(&rlp)))
    }
}
impl StateProvider for FirewoodStateView {
    fn storage(&self, addr: Address, slot: StorageKey)
        -> ProviderResult<Option<StorageValue>> {
        Ok(self.rev.val(storage_key(&addr, &slot)).map_err(map_fw_err)?
              .map(|rlp| decode_rlp_u256(&rlp)))   // slots are RLP(U256), ethhash-MPT
    }
}
impl BytecodeReader for FirewoodStateView {
    fn bytecode_by_hash(&self, h: &B256) -> ProviderResult<Option<Bytecode>> {
        Ok(self.provider.bytecode.get(h.as_slice())?.map(Bytecode::new_raw_decoded))
    }
}
impl BlockHashReader for FirewoodStateView { /* number -> hash from block_hashes db */ }

impl HashedPostStateProvider for FirewoodStateView {
    /// Our keccak hashing MUST match Firewood-ethhash key derivation exactly so
    /// the HashedPostState keys line up with our BatchOps keys.
    fn hashed_post_state(&self, bundle: &BundleState) -> HashedPostState {
        HashedPostState::from_bundle_state::<KeccakKeyHasher>(&bundle.state) // vN-sensitive
    }
}

impl StateRootProvider for FirewoodStateView {
    fn state_root(&self, hashed: HashedPostState) -> ProviderResult<B256> {
        // pre-commit root only — translate to Firewood BatchOps, propose, read root.
        let ops = hashed_post_state_to_batchops(&hashed);     // §17.2.1
        let proposal = self.rev.propose(ops).map_err(map_fw_err)?;
        Ok(proposal.root_hash().map_err(map_fw_err)?.expect("non-empty").into())
    }
    fn state_root_with_updates(&self, hashed: HashedPostState)
        -> ProviderResult<(B256, TrieUpdates)> {
        // We return the real Firewood root but an EMPTY TrieUpdates: reth must
        // never persist trie nodes — Firewood is state-of-record (G1 invariant).
        let p = self.rev.propose(hashed_post_state_to_batchops(&hashed)).map_err(map_fw_err)?;
        let root = p.root_hash().map_err(map_fw_err)?.expect("non-empty").into();
        // Stash the proposal so accept() can commit THIS one (no recompute):
        self.provider.stash_proposal(root, p);                // keyed by root (§17.2.2)
        Ok((root, TrieUpdates::default()))                    // ← the G1 trick
    }
    fn state_root_from_nodes(&self, input: TrieInput) -> ProviderResult<B256> {
        // reth's intermediate-node fast path is meaningless for Firewood; fold the
        // TrieInput's HashedPostState in and ignore the prefix-set/nodes hint.
        self.state_root(input.into_sorted().0 /* HashedPostState */)
    }
    fn state_root_from_nodes_with_updates(&self, input: TrieInput)
        -> ProviderResult<(B256, TrieUpdates)> {
        self.state_root_with_updates(input.into_sorted().0)
    }
}
// StorageRootProvider::storage_root -> Firewood sub-trie root for keccak(addr);
// storage_proof / storage_multiproof + StateProofProvider::proof/multiproof ->
// Firewood range/inclusion proofs (serve eth_getProof + state sync, §10/§17.9).
```

#### 17.2.1 The `BundleState`/`HashedPostState` → Firewood `BatchOp` conversion

This is the single most correctness-critical conversion in the port (the §14 #2
differential gate). reth gives us either a `BundleState` (raw revm delta) or a
`HashedPostState` (keccak-keyed). We normalize to Firewood ethhash `BatchOp`s:

```rust
/// HashedPostState is { accounts: B256(=keccak(addr)) -> Option<Account>,
///                      storages: B256 -> HashedStorage { wiped, slots: B256->U256 } }.
fn hashed_post_state_to_batchops(h: &HashedPostState) -> Vec<firewood::db::BatchOp> {
    let mut ops = Vec::with_capacity(h.accounts.len() + h.storages.len() * 4);
    // 1. Storage first (so an account's storage_root reflects updated slots).
    for (hashed_addr, storage) in h.storages.iter_sorted() {     // DETERMINISTIC order
        if storage.wiped {
            ops.push(firewood::db::BatchOp::DeleteRange { prefix: storage_prefix(hashed_addr) });
        }
        for (hashed_slot, value) in storage.slots.iter_sorted() {
            let key = storage_node_key(hashed_addr, hashed_slot); // ethhash MPT key
            if value.is_zero() {
                ops.push(firewood::db::BatchOp::Delete { key });
            } else {
                ops.push(firewood::db::BatchOp::Put { key, value: rlp_u256(value) });
            }
        }
    }
    // 2. Accounts. ethhash stores the account leaf with storage_root folded in;
    //    Firewood-ethhash computes storage_root from the sub-trie automatically,
    //    so we encode {nonce, balance, code_hash} and let Firewood supply root.
    for (hashed_addr, account) in h.accounts.iter_sorted() {
        match account {
            None => ops.push(firewood::db::BatchOp::Delete { key: account_node_key(hashed_addr) }),
            Some(acc) => ops.push(firewood::db::BatchOp::Put {
                key: account_node_key(hashed_addr),
                value: rlp_account(acc.nonce, acc.balance, acc.bytecode_hash()),
            }),
        }
    }
    ops
}
```

Determinism note (overview §6.1): `iter_sorted()` everywhere — Firewood's root is
order-independent, but we sort to make the conversion itself reproducible and to
ease golden-vector debugging. New bytecode in the bundle is written to the
`bytecode` KV in the *same* accept batch (not into Firewood).

#### 17.2.2 `FirewoodStateCommitter` — propose stash + commit on accept

```rust
impl FirewoodStateProvider {
    /// verify(): proposal computed during state_root_with_updates is stashed by
    /// root. accept(): commit exactly that proposal (no recompute, no MDBX trie write).
    fn stash_proposal(&self, root: B256, p: firewood::db::Proposal) { /* DashMap<B256,_> */ }

    pub fn commit(&self, root: B256) -> Result<(), AvaEvmError> {
        let p = self.take_stashed(root).ok_or(AvaEvmError::MissingProposal(root))?;
        p.commit().map_err(map_fw_err)?;     // durably advances the EVM tip (04 §4.2)
        Ok(())                                // proposal-on-proposal => siblings unaffected
    }
    pub fn discard(&self, root: B256) { self.take_stashed(root); /* drop => reject is free */ }

    /// G2 of §5.2: history bounded by Firewood's RevisionManager window.
    pub fn history_by_state_root(self: &Arc<Self>, root: B256)
        -> ProviderResult<FirewoodStateView> {
        let rev = self.db.revision(root.into())
            .map_err(|_| ProviderError::StateForHashNotFound(root))?; // = coreth "pruned"
        Ok(FirewoodStateView { rev, provider: self.clone() })
    }
}
```

**The G1 invariant (must hold in CI).** No code path may call reth
`StateWriter::write_state` / `UnifiedStorageWriter::write_to_storage` for *state /
trie / hashed* tables. We assert it two ways: (1) we only ever use the bare
`BlockExecutor`/`BlockBuilder` flow and our own committer — we never construct
reth's `BlockchainProvider`/`UnifiedStorageWriter` for state; (2) a unit test
opens the MDBX env after a block and asserts the `PlainState`/`HashedState`/`Trie`
tables stay empty (only `Headers`/`Bodies`/`Receipts`/`CanonicalHeaders` grow,
§17.7).

**Effort/risk.** Effort: **high** (the conversion + proof serving + parity gate).
Risk: **medium** — the seam exists and is stable-ish; the empty-`TrieUpdates`
convention is slightly fragile (a future reth could assume non-empty). **Verdict:
pure wrapper**; *soft* upstream ask = a `StateRootProvider` mode that says "root is
externally computed, do not persist nodes."

### 17.3 G2 — dynamic fees → `next_evm_env` override + atomic gas charge

**The gap.** reth derives base fee from `EthereumHardforks`/EIP-1559 inside
`next_evm_env`, with a fixed `BaseFeeParams`. Avalanche replaced this in stages
(AP3 rolling fee-window, AP4 block-gas-cost, Fortuna/ACP-176 gas-price state
machine, ACP-226 min-delay-excess) — §7.1. reth has no hook for a *stateful*,
fork-switching fee calculator, and no concept of a non-revm (atomic) tx paying gas.

**reth primitives involved (`reth-evm`, `reth-chainspec`).**
`ConfigureEvm::{evm_env, next_evm_env}` (return `EvmEnvFor<Self>` = `EvmEnv {
cfg_env, block_env }`); `EthChainSpec::base_fee_params_at_timestamp`;
`Self::NextBlockEnvCtx` (the CL-supplied attributes we repurpose).

**Wrapper design — `feerules` + `AvaNextBlockCtx`.** Override `next_evm_env` to set
`block_env.basefee`/`gas_limit` from `feerules` keyed on the active fork, and carry
the Avalanche-specific inputs (timestamp-ms, P-Chain height, the serialized fee
state) in `NextBlockEnvCtx`.

```rust
#[derive(Debug, Clone)]
pub struct AvaNextBlockCtx {                 // = ConfigureEvm::NextBlockEnvCtx
    pub timestamp: u64,
    pub timestamp_ms: u64,                    // ACP-226 sub-second cadence
    pub suggested_fee_recipient: Address,
    pub gas_limit_hint: Option<u64>,
    pub pchain_height: u64,                   // for warp predicate ctx (§17.5)
    pub parent_fee_state: AvaFeeState,        // Fortuna/ACP-176 carried state (§7.1)
}

impl ConfigureEvm for AvaEvmConfig {
    type NextBlockEnvCtx = AvaNextBlockCtx;
    // ...
    fn next_evm_env(&self, parent: &Header, attrs: &AvaNextBlockCtx)
        -> Result<EvmEnvFor<Self>, AvaEvmError> {
        let mut env = self.eth_baseline_env(parent, attrs)?;  // spec id via §17.8
        let fork = self.chain_spec.fork_at(attrs.timestamp);
        env.block_env.basefee = feerules::base_fee(&self.chain_spec, fork, parent, attrs)?;
        env.block_env.gas_limit = feerules::gas_limit(&self.chain_spec, fork, parent, attrs)?;
        // Pre-AP3: legacy pricing => basefee MUST be absent (errNilBaseFee parity).
        if fork < AvaFork::ApricotPhase3 { env.block_env.basefee = 0; /* treated as nil */ }
        Ok(env)
    }
}

pub mod feerules {
    /// Dispatch by fork — each arm is a bit-for-bit integer port of customheader.
    pub fn base_fee(cs: &AvaChainSpec, fork: AvaFork, parent: &Header, a: &AvaNextBlockCtx)
        -> Result<u64, AvaEvmError> {
        match fork {
            f if f < AvaFork::ApricotPhase3 => Ok(0),                    // nil
            f if f < AvaFork::Fortuna =>
                window::base_fee_from_window(&cs.fee_config, parent, a.timestamp), // AP3
            _ => acp176::fee_state_before_block(&a.parent_fee_state, parent, a)?    // Fortuna/176
                     .gas_price(),
        }
    }
    pub mod window  { /* dynamic_fee_windower.go: rolling 10s window, MinBaseFee bound,
                         BaseFeeChangeDenominator — integer only, checked arithmetic */ }
    pub mod acp176  { /* dynamic_fee_state.go state machine; AvaFeeState (canoto blob in
                         header extra), GasPrice(), feeStateBeforeBlock(...) */ }
    pub mod blockgas { /* AP4 block_gas_cost.go: Min/MaxBlockGasCost, BlockGasCostStep,
                         TargetBlockRate; the producer must cover it from priority fees */ }
}
```

**Atomic-tx gas (the non-revm charge).** Atomic txs aren't in `block_env`, but they
consume gas and must pay the dynamic base fee. The builder (§17.6) and verify
(§17.4) compute `atomic_gas = TxBytesGas*len + EVMOutputGas*outs + EVMInputGas*ins
+ CostPerSignature*sigs` and `atomic_fee = atomic_gas * base_fee` (coreth
`tx.go::dynamicFee`, with the `nil baseFee` overflow guard → `ErrFeeOverflow`). The
atomic gas is **added to the block gas counter** before EVM txs are packed and
**counts against the AP4 block-gas-cost budget** (`blockgas`), so a block carrying
atomic txs has correspondingly less room for EVM txs — matching coreth exactly.

**Effort/risk.** Effort: **high** (three fee regimes + fork-boundary parity +
golden vectors, §14 #5). Risk: **medium** (pure integer math; risk is fidelity, not
API). **Verdict: pure wrapper** — `next_evm_env` is a clean, stable seam.

### 17.4 G3 — atomic txs as a `BlockExecutor` pre/post hook + atomic-trie commit

**The gap.** revm/reth have **no** notion of (a) a transaction that isn't an EVM
tx, (b) crediting/debiting an account from *outside* the EVM ("EVMStateTransfer"),
or (c) committing a *second* trie + a shared-memory batch atomically with the state
commit. coreth's atomic Import/Export (§6) is exactly this.

**reth primitives involved (`reth-evm`, `revm`).**
`BlockExecutor` (we wrap its pre-execution phase); `BlockExecutorFactory` (we own
it, so we can inject the hook); `revm::database::State<DB>` (the journaled overlay
we mutate directly). No revm change needed — atomic effects are applied as plain
state mutations on the `State` before EVM txs run, so they land in the same
`BundleState` → Firewood proposal (§17.2).

```rust
// reth-evm: BlockExecutor (v2.x shape) — note the split commit.
pub trait BlockExecutor {
    type Transaction; type Receipt; type Evm; type Result;
    fn apply_pre_execution_changes(&mut self) -> Result<(), BlockExecutionError>;
    fn execute_transaction_without_commit(&mut self, tx: impl ExecutableTx<Self>)
        -> Result<Self::Result, BlockExecutionError>;                 // vN-sensitive
    fn commit_transaction(&mut self, out: Self::Result) -> GasOutput; // vN-sensitive
    fn finish(self) -> Result<(Self::Evm, BlockExecutionResult<Self::Receipt>), BlockExecutionError>;
    fn evm_mut(&mut self) -> &mut Self::Evm;
    // + provided execute_transaction / execute_block
}
```

**Wrapper design — `AvaBlockExecutor<E>` decorator + `AtomicStateHook`.**
We do **not** reimplement execution; we *decorate* the reth executor `E` so that
`apply_pre_execution_changes` first runs the standard pre-changes, then applies the
atomic state transfer for the block's atomic txs.

```rust
pub struct AvaBlockExecutor<'a, E: BlockExecutor> {
    inner: E,
    atomic_txs: &'a [AtomicTx],          // attached to the block (§6.2)
    atomic_hook: AtomicStateHook,
    predicates: PredicateResults,        // for warp precompile (§17.5), populated here
}

impl<'a, E: BlockExecutor> BlockExecutor for AvaBlockExecutor<'a, E> {
    type Transaction = E::Transaction; type Receipt = E::Receipt;
    type Evm = E::Evm; type Result = E::Result;

    fn apply_pre_execution_changes(&mut self) -> Result<(), BlockExecutionError> {
        self.inner.apply_pre_execution_changes()?;        // beacon-root etc. (no-op on Ava)
        // (1) Predicate pass (warp BLS verify) over EVM txs -> cache results (§17.5/§6.5).
        self.predicates = run_predicates(self.inner.evm_mut(), &self.predicate_ctx)?;
        // (2) Atomic EVMStateTransfer on the journaled State (mutates same bundle).
        let state = self.inner.evm_mut().db_mut();        // &mut State<DB>  // vN-sensitive
        self.atomic_hook.apply(self.atomic_txs, state)
            .map_err(|e| BlockExecutionError::other(e))?;
        Ok(())
    }
    fn execute_transaction_without_commit(&mut self, tx: impl ExecutableTx<Self>)
        -> Result<Self::Result, BlockExecutionError> { self.inner.execute_transaction_without_commit(tx) }
    fn commit_transaction(&mut self, out: Self::Result) -> GasOutput { self.inner.commit_transaction(out) }
    fn finish(self) -> Result<(Self::Evm, BlockExecutionResult<Self::Receipt>), BlockExecutionError> {
        self.inner.finish()
    }
    fn evm_mut(&mut self) -> &mut Self::Evm { self.inner.evm_mut() }
}

impl AtomicStateHook {
    fn apply(&self, txs: &[AtomicTx], db: &mut impl revm::Database) -> Result<(), AvaEvmError> {
        for tx in txs {
            match tx {
                AtomicTx::Import(t) => for o in &t.outs {
                    let wei = (o.amount as u128).checked_mul(X2C_RATE)            // 1e9, checked
                        .ok_or(AvaEvmError::FeeOverflow)?;
                    increment_balance(db, o.address, wei)?;    // AVAX asset; other assets => nativeasset path
                }
                AtomicTx::Export(t) => for i in &t.ins {
                    let wei = (i.amount as u128).checked_mul(X2C_RATE).ok_or(AvaEvmError::FeeOverflow)?;
                    decrement_balance(db, i.address, wei)?;
                    let n = nonce(db, i.address)?.max(i.nonce + 1);  // matches coreth bump
                    set_nonce(db, i.address, n)?;
                }
            }
        }
        Ok(())
    }
}
```

**The atomic trie + shared memory (post-execution, on accept).** This is *not* in
the executor — it runs in `Block::accept` (§3.1) so it shares the commit batch with
the Firewood state commit. The atomic trie is a **second Firewood-ethhash
instance** (§6.4):

```rust
pub struct AtomicBackend {
    trie: firewood::db::Db,              // ethhash; keys = height(8B)||blockchainID(32B)
    shared_memory: Arc<dyn SharedMemory>, // 07 contract: atomic.Requests{Put,Remove}
    last_committed_root: ArcSwap<B256>,
    commit_interval: u64,               // periodic checkpoint (coreth atomic_trie.go)
}
impl AtomicBackend {
    /// Called from EvmBlock::accept AFTER FirewoodStateCommitter::commit, in the
    /// same logical batch. Indexes ops at this height + applies shared-memory batch.
    pub fn accept(&self, height: u64, txs: &[AtomicTx]) -> Result<(), AvaEvmError> {
        let (mut put, mut remove) = (RequestMap::new(), RequestMap::new());
        for tx in txs {
            let (chain, reqs) = tx.atomic_ops();   // Import=>Remove(srcUTXOs); Export=>Put(elems)
            reqs.merge_into(chain, &mut put, &mut remove);
        }
        let ops = vec![firewood::db::BatchOp::Put {
            key: atomic_trie_key(height),                 // 8B height || 32B blockchainID
            value: serialize_requests(&put, &remove) }];  // ava-codec (linear), byte-exact
        let proposal = self.trie.propose(ops)?;
        let trie_root = proposal.root_hash()?;            // checkpointed for atomic-state sync (§10)
        // ONE atomic batch: shared-memory apply + atomic-trie commit together (07).
        self.shared_memory.apply(put, remove, /*and*/ move || proposal.commit())?;
        if height % self.commit_interval == 0 { self.last_committed_root.store(Arc::new(trie_root.into())); }
        Ok(())
    }
}
```

The **atomic mempool** is a *sidecar* (`AtomicMempool`, §6.4): NOT a reth
`TransactionPool`, because its items aren't revm txs. It gossips via the p2p SDK
(05) and the builder pulls **one atomic batch per block** (§17.6). Conflict/`bonusBlocks`
checks per §6.5. Cross-ref **07** for the shared-memory `atomic.Requests` contract.

**Effort/risk.** Effort: **high** (byte-exact tx codec, atomic trie root parity,
shared-memory batch atomicity). Risk: **low-medium** (decorator over a stable
trait; no revm change). **Verdict: pure wrapper.**

### 17.5 G4 — warp predicate results into the revm precompile context

**The gap.** revm's `PrecompileProvider` is **stateless by default** and has no
channel for *pre-verified* off-EVM data (a warp message's BLS aggregate signature
verified against a P-Chain validator set at a given height, via the proposervm
block context). coreth splits this: predicates are verified *before* execution; the
warp precompile only *reads* the cached result.

**reth/revm primitives involved (`revm` v2.x, `reth-evm`).**

```rust
// revm: the precompile seam (v2.x — note run() takes &CallInputs, returns Option).
pub trait PrecompileProvider<CTX: ContextTr> {
    type Output;
    fn set_spec(&mut self, spec: SpecId) -> bool;                       // vN-sensitive
    fn run(&mut self, ctx: &mut CTX, inputs: &CallInputs)               // vN-sensitive
        -> Result<Option<Self::Output>, String>;
    fn warm_addresses(&self) -> &HashSet<Address, FbBuildHasher<20>>;   // vN-sensitive
    fn contains(&self, addr: &Address) -> bool { /* default over warm_addresses */ }
}
```

**Wrapper design — `AvaPrecompiles` provider + a context extension carrying
predicate results.** Because `run` only gets `ctx`, we thread predicate results
through a **revm context extension** (the `CTX` type carries an
`Arc<PredicateResults>`); `AvaBlockExecutorFactory` installs both when building the
EVM. The warp precompile reads from that extension; all other Avalanche precompiles
are stateful over the live journaled state.

```rust
/// Attached to the revm Context as a typed extension (DB stays Firewood-backed).
#[derive(Clone, Default)]
pub struct AvaCtxExt {
    pub predicates: Arc<PredicateResults>,   // tx_index -> precompile_addr -> verified bytes
    pub block_ctx: AvaBlockCtx,              // proposervm ctx, P-Chain height, timestamp
}

pub struct AvaPrecompiles {
    base: EthPrecompiles,                     // revm standard set for the active spec
    modules: Arc<PrecompileRegistry>,         // addr -> Arc<dyn StatefulPrecompile>
    warm: HashSet<Address, FbBuildHasher<20>>,// activated set (fork + upgrade gated, §8.3)
}

impl<CTX: ContextTr<…Ext = AvaCtxExt>> PrecompileProvider<CTX> for AvaPrecompiles {
    type Output = InterpreterResult;
    fn set_spec(&mut self, spec: SpecId) -> bool { self.base.set_spec(spec) }
    fn run(&mut self, ctx: &mut CTX, inputs: &CallInputs)
        -> Result<Option<Self::Output>, String> {
        let addr = inputs.target_address;
        if let Some(p) = self.warm.contains(&addr).then(|| self.modules.get(&addr)).flatten() {
            let pctx = PrecompileCtx {
                caller: inputs.caller_address, value: inputs.call_value(),
                predicates: ctx.ext().predicates.clone(),     // ← warp reads cached BLS result
                block: ctx.ext().block_ctx.clone(),
                tx_index: ctx.ext().block_ctx.current_tx_index,
            };
            return p.run(inputs.input.bytes(ctx), inputs.gas_limit, ctx.journal_mut(), &pctx)
                    .map(Some).map_err(|e| e.to_string());
        }
        self.base.run(ctx, inputs)                            // fall through to standard
    }
    fn warm_addresses(&self) -> &HashSet<Address, FbBuildHasher<20>> { &self.warm }
}

pub trait StatefulPrecompile: Send + Sync {
    fn run(&self, input: &[u8], gas: u64, journal: &mut dyn JournalExt, ctx: &PrecompileCtx)
        -> Result<InterpreterResult, PrecompileError>;
}
```

The **predicate pass** that *populates* `AvaCtxExt::predicates` runs in
`AvaBlockExecutor::apply_pre_execution_changes` (§17.4), driven by the proposervm
block context delivered via `Block::verify_with_context` (06). Warp verification =
BLS aggregate over the P-Chain validator set at `block_ctx.pchain_height` (batched,
`ava-crypto`/`blst`, §15). `handlePrecompileAccept` (accept, §3.1) fires module
accept-hooks (warp backend records sent messages, coreth `warp/backend.go`).
`AvaBlockExecutorFactory::create_executor` is where `AvaPrecompiles::for_height(t)`
+ `AvaCtxExt` are wired into the revm handler.

**Effort/risk.** Effort: **medium-high** (the ctx-extension plumbing + per-precompile
parity, §14 #6). Risk: **medium** (revm `PrecompileProvider`/context-ext shape is
the part most likely to churn — `set_spec`/`run` already moved once). **Verdict:
pure wrapper.**

### 17.6 G5 — on-demand block building (bypass `PayloadBuilderService`)

**The gap.** reth builds payloads on the engine's schedule via
`PayloadBuilderService`/`PayloadJob`. Avalanche builds **only when consensus asks**
(`BuildBlock`) and only when there is work (coreth `block_builder.go`:
`needToBuild`, `signalCanBuild`, min-retry delay).

**reth primitives involved (`reth-evm`).** `ConfigureEvm::builder_for_next_block`
→ `BlockBuilder`:

```rust
pub trait BlockBuilder {
    type Primitives; type Executor: BlockExecutor;
    fn apply_pre_execution_changes(&mut self) -> Result<(), BlockExecutionError>;
    fn execute_transaction(&mut self, tx: impl ExecutorTx<Self::Executor>)
        -> Result<GasOutput, BlockExecutionError>;
    /// KEY for G1: we may pass a PRECOMPUTED (root, TrieUpdates) so reth assembles
    /// the header with the Firewood root and does NOT compute/persist its own.
    fn finish(self, state: impl StateProvider, precomputed: Option<(B256, TrieUpdates)>)
        -> Result<BlockBuilderOutcome<Self::Primitives>, BlockExecutionError>;  // vN-sensitive
    fn into_executor(self) -> Self::Executor;
    fn executor_mut(&mut self) -> &mut Self::Executor;
}
```

**Wrapper design — `BlockBuilderDriver`.** Already sketched in §4; the §17 additions
are the two load-bearing seams: (1) we wrap the builder's executor in
`AvaBlockExecutor` (atomic pre-hook), and (2) we exploit `finish`'s
`precomputed: Option<(B256, TrieUpdates)>` to inject the **Firewood root** so reth
never runs its own state-root/commit:

```rust
impl BlockBuilderDriver {
    pub async fn build_on(&self, parent: B256, ctx: Option<&BlockContext>)
        -> Result<Arc<dyn Block>, AvaEvmError> {
        let attrs = self.next_block_attrs(parent, ctx)?;        // AvaNextBlockCtx (§17.3)
        let parent_hdr = self.state.header(parent)?;
        let view = self.state.history_by_state_root(parent_hdr.state_root)?;
        let mut db = State::builder().with_database(view).with_bundle_update().build();
        let mut builder = self.evm_config.builder_for_next_block(&mut db, &parent_hdr, attrs.clone())?;

        builder.apply_pre_execution_changes()?;                 // runs AtomicStateHook + predicates
        // 1. one atomic batch first (gas-budgeted, §17.3); atomic txs already applied
        //    to state by the pre-hook — here we just reserve their gas.
        let atomic_txs = self.atomic.mempool.next_batch(&attrs)?;
        let mut gas_used = atomic_gas_total(&atomic_txs);
        // 2. EVM txs by effective tip until gas / blockGasCost budget hit.
        for tx in self.txpool.best_transactions(&attrs) {
            if gas_used + tx.gas_limit() > attrs.gas_limit { break; }
            match builder.execute_transaction(tx.clone()) {
                Ok(out) => gas_used += out.gas_used,
                Err(e) if e.is_invalid_tx() => { self.txpool.remove_invalid(tx.hash()); }
                Err(BlockExecutionError::Gas(_)) => break,
                Err(e) => return Err(e.into()),
            }
        }
        // 3. Compute the Firewood root from the accumulated bundle, then hand it to
        //    reth's assembler so the header carries it (G1):
        let bundle = builder.executor_mut().evm_mut().db_mut().take_bundle(); // vN-sensitive
        let proposal = self.state.propose_from_bundle(parent_hdr.state_root, &bundle)?;
        let root: B256 = proposal.root_hash()?.into();
        self.state.stash_proposal(root, proposal);              // committed on accept (§17.2.2)
        let outcome = builder.finish(self.state.view_tip()?, Some((root, TrieUpdates::default())))?;
        let block = assemble_ava_block(outcome, atomic_txs)?;   // attach atomic txs (§9.3)
        *self.last_build.lock() = Some((parent, Instant::now()));
        Ok(Arc::new(EvmBlock::built(block, root)))
    }
}
```

**Build-then-verify symmetry (must hold).** `build_on` and `verify` (§3.2) drive the
*same* `AvaEvmConfig` executor over the *same* parent view with the *same* atomic
pre-hook, so a self-built block re-verifies to the identical Firewood root — the
determinism contract. The mempool `Notify`-on-nonempty + `minBlockBuildingRetryDelay`
(§4) replaces the payload-job loop.

**Effort/risk.** Effort: **medium**. Risk: **low** (`builder_for_next_block` +
`finish(precomputed)` are exactly the seams we need). **Verdict: pure wrapper.**

### 17.7 G6 — Snowman fork choice → Accept=commit+canonicalize, Reject=drop

**The gap.** reth owns canonicalization/reorg through its blockchain tree
(`TreeState`), staged-sync pipeline, and `forkchoiceUpdated`. Snowman owns fork
choice: acceptance is **linear**, there are **no reorgs**, and Reject simply drops
an uncommitted proposal. We must keep reth-db's block/receipt storage consistent
**without** reth's pipeline.

**reth primitives involved.** *Avoided:* `reth_blockchain_tree`, `BeaconConsensusEngine`,
`Pipeline`, `UnifiedStorageWriter` for state. *Used directly:* the raw `reth-db`
(MDBX) tables + `reth-static-file` for headers/bodies/receipts/logs and the
canonical number↔hash index — written by us, not by a stage.

**Wrapper design — `CanonicalStore`.** A thin writer over the block tables that the
`ChainVm` adapter drives on Accept; Reject is a pure in-memory drop.

```rust
pub struct CanonicalStore { db: reth_db::DatabaseEnv, static_files: StaticFileProvider }

impl CanonicalStore {
    /// EvmBlock::accept, AFTER FirewoodStateCommitter::commit + AtomicBackend::accept.
    /// One MDBX rw-tx: append header/body/receipts + advance CanonicalHeaders + tip.
    pub fn append_canonical(&self, block: &SealedBlock, receipts: &[Receipt]) -> Result<(), AvaEvmError> {
        let mut tx = self.db.tx_mut()?;
        let n = block.number;
        tx.put::<tables::Headers>(n, block.header().clone())?;
        tx.put::<tables::CanonicalHeaders>(n, block.hash())?;       // number -> hash
        tx.put::<tables::HeaderNumbers>(block.hash(), n)?;          // hash -> number
        tx.put::<tables::BlockBodyIndices>(n, body_indices(block))?;
        write_transactions(&mut tx, n, block.body())?;             // Transactions table
        self.static_files.append_receipts(n, receipts)?;           // receipts -> static file
        tx.put::<tables::ChainState>(LAST_CANONICAL, n.into())?;    // tip pointer
        tx.commit()?;                                              // NO state/trie tables touched (G1)
        Ok(())
    }
}
```

`EvmBlock::reject` writes **nothing** here — it only calls
`FirewoodStateCommitter::discard(root)` and evicts from the `verified` map (§3.1).
Because siblings hold independent Firewood proposals (proposal-on-proposal, 04
§4.2), no canonical rewrite is ever needed. **Consistency invariant:** the only
writer of the canonical tables is `append_canonical` on accept, and it advances
strictly by +1 height (linear), so the number↔hash index can never disagree with
Firewood's committed tip (asserted: `LAST_CANONICAL == last_accepted.height`).
History reads (`get_block`, RPC) go through reth's `BlockReader` over these tables
read-only — fine, since we wrote them in reth's own format.

**Effort/risk.** Effort: **low-medium**. Risk: **low** (we bypass the complex parts
entirely; the table writes are reth's documented schema). **Verdict: pure wrapper.**

### 17.8 G7 — Avalanche fork schedule → custom `Hardforks`/`EthChainSpec` + per-block spec id

**The gap.** reth's `EthChainSpec`/`EthereumHardforks` model Ethereum's fork list;
Avalanche interleaves **timestamp-activated** Avalanche phases
(Apricot→…→Granite) with the Ethereum forks coreth maps in, and the revm `SpecId`
must be selected per block from *both*.

**reth primitives involved (`reth-chainspec`, `reth-ethereum-forks`).**
`EthChainSpec` (methods: `chain`, `base_fee_params_at_timestamp`,
`blob_params_at_timestamp`, `genesis_hash/header/genesis`, `display_hardforks`,
`final_paris_total_difficulty`, `bootnodes`, …); `Hardforks`/`EthereumHardforks`;
`ChainHardforks` (ordered `(Hardfork, ForkCondition)` list); `ForkCondition::Timestamp`.

**Wrapper design — `AvaChainSpec` + `AvaHardfork`.** Build a `ChainHardforks` that
interleaves `EthereumHardfork::*` and `AvaHardfork::*` ordered by activation
timestamp; map each block to a revm `SpecId` via the highest active fork.

```rust
pub enum AvaHardfork {
    Eth(EthereumHardfork),                 // London, Shanghai, Cancun, Prague…
    ApricotPhase1, ApricotPhase2, ApricotPhase3, ApricotPhase4, ApricotPhase5,
    ApricotPhasePre6, ApricotPhase6, ApricotPhasePost6,
    Banff, Cortina, Durango, Etna, Fortuna, Granite,
}
impl Hardfork for AvaHardfork { fn name(&self) -> &'static str { /* … */ } }

pub struct AvaChainSpec {
    inner: ChainHardforks,                 // ordered ForkCondition::Timestamp list
    eth_genesis_header: Header,            // for genesis_hash parity (§11.1)
    genesis: Genesis,
    fee_config: FeeConfig,                 // §7.4
    network_upgrades: NetworkUpgrades,     // *uint64 timestamps from params/extras
    is_subnet: bool,
    chain: Chain,
}
impl EthChainSpec for AvaChainSpec {
    type Header = alloy_consensus::Header;
    fn chain(&self) -> Chain { self.chain }
    fn base_fee_params_at_timestamp(&self, t: u64) -> BaseFeeParams {
        // Avalanche overrides base fee in feerules (§17.3); return params consistent
        // with the active fork so reth's own paths don't disagree where they run.
        self.fee_config.base_fee_params_at(self.fork_at(t))
    }
    fn genesis_header(&self) -> &Header { &self.eth_genesis_header }
    fn genesis(&self) -> &Genesis { &self.genesis }
    fn final_paris_total_difficulty(&self) -> Option<U256> { Some(U256::ZERO) } // no PoW
    fn bootnodes(&self) -> Option<Vec<NodeRecord>> { None }                     // Avalanche p2p (05)
    // display_hardforks / blob_params_at_timestamp / prune_delete_limit per schedule
}
impl EthereumHardforks for AvaChainSpec {
    fn ethereum_fork_activation(&self, fork: EthereumHardfork) -> ForkCondition {
        self.inner.fork(AvaHardfork::Eth(fork))   // coreth maps each Eth fork to a phase ts
    }
}
impl AvaChainSpec {
    pub fn fork_at(&self, t: u64) -> AvaFork { /* highest fork with Timestamp <= t */ }
    pub fn is_apricot_phase3(&self, t: u64) -> bool { self.fork_at(t) >= AvaFork::ApricotPhase3 }
    pub fn is_fortuna(&self, t: u64) -> bool { self.fork_at(t) >= AvaFork::Fortuna }
    /// revm SpecId for a block: the Ethereum spec coreth pins for this Avalanche phase.
    pub fn revm_spec_id(&self, t: u64) -> SpecId {
        match self.fork_at(t) {
            f if f >= AvaFork::Granite  => SpecId::PRAGUE,    // (example mapping; cite extras)
            f if f >= AvaFork::Durango  => SpecId::SHANGHAI,
            f if f >= AvaFork::Cortina  => SpecId::LONDON,
            _                           => SpecId::BERLIN,
        }
    }
    /// network_upgrades.checkCompatible parity (incompatibility on activated forks).
    pub fn check_compatible(&self, other: &NetworkUpgrades, head_ts: u64) -> Result<(), AvaEvmError> { /* … */ }
}
```

`revm_spec_id` is consumed by `AvaEvmConfig::eth_baseline_env` (§17.3) to set
`cfg_env.spec` per block. Fork *timestamps* for Mainnet/Fuji are protocol constants
embedded in `ava-version` (overview §5) and re-exported via `network_upgrades`.

**Effort/risk.** Effort: **medium** (the schedule + the exact Eth↔phase spec
mapping must match coreth, §14 #5 boundary tests). Risk: **low** (`EthChainSpec`/
`Hardforks` are designed to be implemented by alt-chains; op-reth/bsc-reth do it).
**Verdict: pure wrapper.**

### 17.9 G8 — EVM/atomic state sync + RPC (`avax.*` namespace, `eth_*` overrides)

**The gap (two parts).** (a) reth's staged/snap sync assumes MDBX-as-truth and the
engine; we need the **coreth EVM state-sync protocol** mapped onto Firewood
proofs + a parallel **atomic-trie** sync. (b) reth's RPC is `jsonrpsee`-based and
`EthApi` is generic over reth's *own* provider/pool; we must (i) add an `avax.*`
namespace, and (ii) override `eth_*` fee/tag behavior (accepted-block tag, base
fee) while reading Firewood state.

**reth primitives involved (`reth-rpc`, `reth-rpc-eth-api`, `reth-rpc-builder`).**
`EthApi<Provider, Pool, …>` (generic — instantiate over `FirewoodStateProvider` +
`AvaTxPool`); the `EthApiServer`/`FullEthApi` trait family; `RpcModule` +
`merge_configured`/`extend_rpc_modules`; `jsonrpsee` proc-macro server traits.

**Wrapper design — (a) sync.** A direct port of coreth's protocol on the p2p SDK
(05), served from Firewood proofs (no reth sync):

```rust
pub struct EvmStateSyncServer { state: Arc<FirewoodStateProvider>, atomic: Arc<AtomicBackend> }
impl EvmStateSyncServer {
    /// leaf request -> Firewood range proof at a historical revision (04 §4.2/§4.3).
    fn handle_leafs(&self, req: LeafsRequest) -> LeafsResponse {
        let rev = self.state.history_by_state_root(req.root).unwrap();
        let proof = rev.range_proof(req.start, req.end, req.limit); // wire-exact (Go firewood/syncer)
        LeafsResponse { keys: proof.keys, vals: proof.vals, proof: proof.nodes }
    }
}
// Client: reconstruct a Firewood trie from range/change proofs, verify root, then
// AtomicBackend syncs the atomic trie the same way and ApplyToSharedMemory from the
// synced cursor. Blocks/headers/receipts backfill into CanonicalStore (§17.7).
```

**Wrapper design — (b) RPC.** Instantiate reth's `EthApi` over our provider/pool,
override the fee/tag helpers, and merge a hand-written `avax.*` module:

```rust
type AvaEthApi = EthApi<Arc<FirewoodStateProvider>, AvaTxPool, /* Network, EvmConfig */>;

// Override fee + accepted-tag behavior by wrapping the relevant EthApi helper traits.
#[async_trait]
impl EthFees for AvaEthApiWrapper {                       // vN-sensitive trait name
    async fn gas_price(&self) -> RpcResult<U256> { Ok(feerules::suggested_price(&self.cs, self.head()).into()) }
    async fn fee_history(&self, n: u64, newest: BlockNumberOrTag, p: Option<Vec<f64>>)
        -> RpcResult<FeeHistory> { feerules::fee_history(/* AP3 window / ACP-176 */).await }
    async fn max_priority_fee_per_gas(&self) -> RpcResult<U256> { /* Avalanche tip rule */ }
}
// "accepted" block tag: map BlockNumberOrTag::{Latest,Safe,Finalized} all to the
// last-accepted height (Snowman has no pending/unsafe head) — coreth `rpc_accepted`.

// avax.* namespace via jsonrpsee macro, merged into the shared module.
#[rpc(server, namespace = "avax")]
pub trait AvaxApi {
    #[method(name = "issueTx")]        async fn issue_tx(&self, tx: Hex) -> RpcResult<TxId>;
    #[method(name = "getAtomicTx")]    async fn get_atomic_tx(&self, id: TxId) -> RpcResult<AtomicTxReply>;
    #[method(name = "getAtomicTxStatus")] async fn status(&self, id: TxId) -> RpcResult<Status>;
    #[method(name = "getUTXOs")]       async fn get_utxos(&self, args: GetUtxosArgs) -> RpcResult<UtxosReply>;
    #[method(name = "getBlockByHeight")] async fn block_by_height(&self, h: u64) -> RpcResult<Hex>;
}
// Mount: modules.merge_configured(AvaxApiImpl::new(atomic, state).into_rpc())?;
//        modules.merge_configured(eth_api.into_rpc())?;   // eth_/net_/web3_/debug_/txpool_
// Per §9.2/G8 in §12: decide in 12-node whether to bridge jsonrpsee under the axum
// router or run jsonrpsee for eth_* + axum for avax.*/admin.* — both serve identical JSON.
```

**Effort/risk.** Effort: **high** (sync protocol byte-exactness + RPC parity,
§14 #7/#8). Risk: **medium** — `EthApi`'s generic instantiation over a *third-party*
provider is the part reth keeps refactoring; the *soft* upstream ask is a stable
"EthApi over my provider/pool" builder. **Verdict: pure wrapper (+1 soft upstream
ask).**

### 17.10 Reusable API surface for SAE (11) — the §16 reuse contract

These wrapper types are **public, stable (behind the facade) APIs** of `ava-evm` /
`ava-evm-reth`, so `ava-saevm-exec` (11 §6) reuses the EVM engine without
re-implementing it. This is the concrete fulfillment of §16's reuse contract and
`00` §11.1.5:

| Public item | Crate | What SAE reuses it for |
|---|---|---|
| `trait ExternalConsensusExecutor` + `ExecOutcome` (§17.1) | `ava-evm-reth` | the single batch-execute entrypoint; SAE's executor calls `execute_batch` per ordered block (11 §6.1 step 6) decoupled from `ChainVm` |
| `AvaEvmConfig` (`impl ConfigureEvm` + `ExternalConsensusExecutor`) (§7.2/§17.3) | `ava-evm` | same revm executor, fee rules, precompiles, spec-id selection |
| `FirewoodStateProvider` / `FirewoodStateView` / `FirewoodStateCommitter` (§17.2) | `ava-evm` | SAE's `Tracker` (11 §7.1) holds `Arc<FirewoodStateProvider>`; opens views by root, proposes, defers commit on the interval |
| `hashed_post_state_to_batchops` + `propose_from_bundle` (§17.2.1) | `ava-evm` | identical `BundleState`→Firewood conversion ⇒ identical state roots across both drivers |
| `AvaPrecompiles` / `PrecompileRegistry` / `AtomicStateHook` (§17.4/§17.5) | `ava-evm` | SAE C-Chain (`ava-saevm-cchain`) reuses warp/atomic semantics via hooks |
| `AvaChainSpec` / `revm_spec_id` (§17.8) | `ava-evm` | shared fork schedule + spec-id |

**Boundary that is NOT shared:** the block lifecycle. `EvmVm`/`EvmBlock` (§3, the
synchronous `ChainVm`/verify-then-vote) and `BlockBuilderDriver` (§17.6) are
sync-C-Chain-only; SAE supplies its *own* streaming lifecycle (order→execute→settle,
11 §6) but drives the *same* `ExternalConsensusExecutor` + `FirewoodStateCommitter`
underneath. "One EVM engine, two drivers" (00 §11.1.5) is enforced by SAE depending
only on the items above — never on `EvmVm`, `BlockBuilderDriver`, or reth directly.

### 17.11 Newly discovered gaps (not in the original G0–G8)

Surfaced while pinning the reth v2.x API:

- **G9 (provider Storage-V2 / `SparseTrieCache` coupling).** reth 2.0 made
  Storage-V2 + an in-memory `SparseTrieCache` the default state-root path. Several
  newer reth call sites assume that cache exists. Since we replace the entire
  state-root path with Firewood (§17.2), we must ensure no reth component we *do*
  use (the `BlockBuilder::finish` assembler, RPC `eth_getProof`) silently routes
  through `SparseTrieCache`. **Mitigation:** the empty-`TrieUpdates` + precomputed-root
  path (§17.2/§17.6) keeps reth off its trie entirely; add a CI assertion (the §17.2
  empty-table check) that the sparse-trie/state tables stay empty. **Verdict: pure
  wrapper**, folded into G1's invariant. (This is the concrete 2.x face of G0.)
- **G10 (revm context-extension typing churn).** Threading `AvaCtxExt` (§17.5)
  through revm's `ContextTr` associated types is the API most exposed to revm's
  ongoing generics refactors (the `set_spec`/`run` change already bit us). **Mitigation:**
  the facade (§17.1) owns the single `impl PrecompileProvider`; if revm drops typed
  context extensions we fall back to an `Arc<PredicateResults>` carried in the
  `EvmFactory` closure capture rather than the context. **Verdict: pure wrapper**,
  but flagged as the second-most-likely churn point after G0.
