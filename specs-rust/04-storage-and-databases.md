# 04 — Storage & Databases

> **Status:** Conforms to `00-overview-and-conventions.md`. Binding decisions from
> §4.4 (storage crates), the **Pebble→RocksDB** decision, and the crate names
> (`ava-database`, `ava-merkledb`, `ava-blockdb`, `ava-archivedb`) are honored here.
> Where this spec adds a binding decision (the `Database` trait signature, the
> merkledb-vs-Firewood compatibility conclusion), every other spec MUST conform.

This spec covers the generic key/value layer and the merkleized state stores.

**Go source covered**

| Go path | Subject |
|---|---|
| `database/{database,batch,iterator,common,errors,helpers}.go` | the `Database` interface family |
| `database/{leveldb,pebbledb,memdb,prefixdb,versiondb,meterdb,corruptabledb,linkeddb,rpcdb,heightindexdb}` | KV backends |
| `database/dbtest`, `database/heightindexdb/dbtest` | conformance batteries |
| `database/merkle/{firewood,sync}` | Firewood binding + the MerkleDB state-sync protocol |
| `x/merkledb` | path-based Merkle radix trie (node/key/hash/proof/view/history) |
| `x/blockdb` | append-optimized block store |
| `x/archivedb` | height-versioned KV |
| `proto/rpcdb`, `proto/sync` | gRPC DB protocol + state-sync protocol |

**Rust crates produced:** `ava-database`, `ava-merkledb`, `ava-blockdb`,
`ava-archivedb`. External deps (per §4.4): `rust-rocksdb`, `firewood` (+
`firewood-storage`), `prost`/`tonic` (rpcdb, sync), `prometheus` (meterdb),
`bytes`, `parking_lot`, `lru`.

---

## 1. The `Database` trait family (`ava-database`)

### 1.1 Go shape (what we must reproduce)

The Go `database.Database` is the composition
`KeyValueReaderWriterDeleter + Batcher + Iteratee + Compacter + io.Closer +
health.Checker`. Semantics that are load-bearing and MUST be preserved exactly:

- **Ordered keys.** Iteration is in lexicographic byte order. Backends sort.
- **Key/value memory safety.** `Get` returns a slice the caller may read but not
  mutate; `key`/`value` args are safe to mutate after the call returns (callers
  rely on this — see `dbtest.TestMemorySafety*`). In Rust this is free: `Put`
  takes `&[u8]` and copies; `Get` returns an owned `Vec<u8>`.
- **`nil` ⇔ empty.** A `nil` key is the empty key; a `nil`/empty value round-trips
  as possibly-`nil`-or-empty. Rust: `&[]` and `Vec::new()`; `Get` of an
  empty-valued key returns `Ok(Vec::new())` (never `ErrNotFound`).
- **Error model (`database/errors.go`).** Exactly two sentinels:
  `ErrClosed = "closed"`, `ErrNotFound = "not found"`. Operations after `Close`
  return `ErrClosed`. `Get` of a missing key returns `ErrNotFound`.
- **Iterator contract (`iterator.go`).** `Next()` advances and returns `bool`;
  `Key`/`Value` return the current pair (or `nil` when done); `Error()` is read
  after exhaustion; `Release()` is idempotent and mandatory. A closed DB makes
  `Next()` return false and `Error()` return `ErrClosed`. Iterators are
  snapshots — not safe for concurrent use individually, but multiple iterators
  may run concurrently.
- **Batch contract (`batch.go`).** Write-only, replayable, resettable; `Size()`
  counts keys+values+deleted-keys bytes; `Replay(w)` re-applies ops in order;
  `Inner()` returns the batch against the base DB. Atomic on `Write()`.

### 1.2 Sync vs async decision (BINDING)

**Decision: the `Database` trait family is *synchronous*.** Backends
(RocksDB, Firewood) are themselves blocking C/Rust libraries with no async API,
and the Go interface is synchronous. Making the trait `async` would force a fake
async wrapper around blocking calls (the worst of both worlds) and break object
safety with `async-trait` allocation on the hot path.

Per §7.2, **blocking DB work runs on `spawn_blocking` (or a dedicated rayon /
DB thread pool) at the *call site*** — never inside the trait. Subsystem specs
that hold a `dyn Database` from async code wrap the call:
`tokio::task::spawn_blocking(move || db.get(&key)).await?`. `ava-saevm` (11) and
`ava-evm` (10) own dedicated commit/execution thread pools and therefore call
the DB synchronously inside those threads with no `spawn_blocking` overhead.

This matches the overview's "sync trait + spawn_blocking at call sites" guidance.

### 1.3 The traits

```rust
use bytes::Bytes;

/// The only two sentinel errors, mirroring database/errors.go.
#[derive(Debug, thiserror::Error)]
pub enum Error {
    #[error("closed")]
    Closed,
    #[error("not found")]
    NotFound,
    /// Any backend-specific failure (RocksDB, IO, gRPC, corruption).
    /// corruptabledb poisons on *these* only (not Closed/NotFound).
    #[error(transparent)]
    Other(#[from] anyhow::Error),
}
pub type Result<T> = std::result::Result<T, Error>;

pub trait KeyValueReader {
    fn has(&self, key: &[u8]) -> Result<bool>;
    /// Returns Err(Error::NotFound) when absent.
    fn get(&self, key: &[u8]) -> Result<Vec<u8>>;
}

pub trait KeyValueWriter {
    fn put(&self, key: &[u8], value: &[u8]) -> Result<()>;
}

pub trait KeyValueDeleter {
    fn delete(&self, key: &[u8]) -> Result<()>;
}

pub trait Compacter {
    /// `start = None` ⇒ before all keys; `limit = None` ⇒ after all keys.
    fn compact(&self, start: Option<&[u8]>, limit: Option<&[u8]>) -> Result<()>;
}

/// A point-in-time, ordered cursor. `Drop` is the implicit `Release()`,
/// but we keep an explicit `release()` for parity (idempotent).
pub trait Iterator {
    fn next(&mut self) -> bool;
    fn error(&self) -> Result<()>;
    fn key(&self) -> Option<&[u8]>;
    fn value(&self) -> Option<&[u8]>;
    fn release(&mut self) {}
}

pub trait Iteratee {
    type Iter<'a>: Iterator where Self: 'a;
    fn new_iterator(&self) -> Self::Iter<'_> {
        self.new_iterator_with_start_and_prefix(&[], &[])
    }
    fn new_iterator_with_start(&self, start: &[u8]) -> Self::Iter<'_> {
        self.new_iterator_with_start_and_prefix(start, &[])
    }
    fn new_iterator_with_prefix(&self, prefix: &[u8]) -> Self::Iter<'_> {
        self.new_iterator_with_start_and_prefix(&[], prefix)
    }
    fn new_iterator_with_start_and_prefix(&self, start: &[u8], prefix: &[u8]) -> Self::Iter<'_>;
}

pub trait Batch: KeyValueWriter + KeyValueDeleter {
    fn size(&self) -> usize;
    fn write(&mut self) -> Result<()>;
    fn reset(&mut self);
    fn replay(&self, w: &mut dyn WriteDelete) -> Result<()>;
    fn inner(&mut self) -> &mut dyn Batch;
}

/// Object-safe target for `Batch::replay` / `BatchOps`.
pub trait WriteDelete {
    fn put(&mut self, key: &[u8], value: &[u8]) -> Result<()>;
    fn delete(&mut self, key: &[u8]) -> Result<()>;
}

pub trait Batcher {
    fn new_batch(&self) -> Box<dyn Batch + '_>;
}

/// The full DB. `Send + Sync` so it can live behind `Arc<dyn Database>`,
/// matching how Go threads a single `database.Database` through chains.
pub trait Database:
    KeyValueReader + KeyValueWriter + KeyValueDeleter
    + Batcher + Iteratee<Iter<'_> = Self::DynIter> + Compacter + Send + Sync
{
    fn close(&self) -> Result<()>;
    fn health_check(&self) -> Result<serde_json::Value>;
}
```

**Object-safety note.** The generic `Iteratee::Iter<'a>` GAT is convenient for
concrete backends but not object-safe. `ava-database` therefore also defines a
**boxed** facade used wherever Go passes `database.Database` by interface:

```rust
pub type BoxIter<'a> = Box<dyn Iterator + 'a>;

/// Object-safe DB used behind `Arc<dyn DynDatabase>` across the workspace.
pub trait DynDatabase: Send + Sync {
    fn has(&self, key: &[u8]) -> Result<bool>;
    fn get(&self, key: &[u8]) -> Result<Vec<u8>>;
    fn put(&self, key: &[u8], value: &[u8]) -> Result<()>;
    fn delete(&self, key: &[u8]) -> Result<()>;
    fn new_batch(&self) -> Box<dyn Batch + '_>;
    fn new_iterator_with_start_and_prefix<'a>(&'a self, start: &[u8], prefix: &[u8]) -> BoxIter<'a>;
    fn compact(&self, start: Option<&[u8]>, limit: Option<&[u8]>) -> Result<()>;
    fn close(&self) -> Result<()>;
    fn health_check(&self) -> Result<serde_json::Value>;
}
```

> **BINDING (cross-spec):** every subsystem that today takes a
> `database.Database` takes `Arc<dyn DynDatabase>`. Backends implement both the
> typed `Database` (for monomorphized hot paths like merkledb's intermediate
> node DB) and `DynDatabase` (a thin blanket impl). The sentinel-error contract
> (`Error::Closed`, `Error::NotFound`) is part of the contract — match on these
> variants exactly where Go uses `errors.Is`.

### 1.4 Shared helpers (`helpers.go`)

Free functions in `ava-database::helpers`, byte-exact with Go:
`put_id/get_id`, `put_u64/get_u64` (8-byte big-endian — `PackUInt64`),
`put_u32/get_u32`, `put_bool/get_bool` (`0x00`/`0x01`), `put_timestamp/
get_timestamp` (Go `time.Time.MarshalBinary` format — must be reproduced bit-for-
bit since some persisted records embed it), `with_default`, `count`, `size`
(`kvPairOverhead = 8`), `atomic_clear[_prefix]`, `clear[_prefix]` (batched delete
with `write_size` threshold and iterator re-seek to release deleted-key refs).

`BatchOps` (the reusable Put/Delete recorder backing `memdb`/`versiondb`
batches) ports directly: a `Vec<BatchOp{ key, value, delete }>` with `size`
accounting; `reset()` reproduces the `MaxExcessCapacityFactor=4` /
`CapacityReductionFactor=2` shrink heuristic (a perf-parity nicety, not a
correctness requirement).

---

## 2. KV backends (`ava-database::*`)

Each backend implements `Database` + `DynDatabase`. Behavior is verified by the
shared conformance battery (§6).

### 2.1 `rocksdb` — the on-disk default (replaces `leveldb` *and* `pebbledb`)

Per §4.4, RocksDB is the production on-disk backend (closest ordered/prefix/
batch/snapshot semantics to goleveldb + Pebble). Wraps `rust-rocksdb`.

- **Open options** chosen to match avalanchego's leveldb/pebble tuning
  (`database/leveldb/db.go`, `database/pebbledb/db.go`): block cache size, write
  buffer size, max open files, bloom filters, compression (LZ4/Snappy),
  `OptimizeForPointLookup`/level compaction. These are perf knobs, not protocol —
  expose them through `ava-config` mirroring the Go JSON DB-config keys.
- `get` → `ErrNotFound` on `Ok(None)`. `put`/`delete` direct. `has` via
  `get_pinned` (zero-copy) returning `is_some()`.
- **Iterators:** RocksDB `DBRawIterator`. `new_iterator_with_start_and_prefix`
  combines `set_mode(From(start, Forward))` with a prefix-bound filter applied in
  the wrapper (RocksDB prefix-seek alone is insufficient for arbitrary prefixes;
  emulate Go's `start ≥` AND `HasPrefix` predicate). The iterator holds a RocksDB
  **snapshot** so it is point-in-time, matching `TestIteratorSnapshot`.
- **Batch:** `rust_rocksdb::WriteBatch`; `write()` is atomic.
- **Compact:** `compact_range(start, limit)`.
- **Close:** RocksDB closes on drop; we gate with an `AtomicBool` so post-close
  ops return `ErrClosed` (Go returns `ErrClosed`, not a panic).
- **`health_check`:** report `rocksdb.estimate-live-data-size` / errors as a JSON
  blob (parity with the Go leveldb health stats shape is best-effort, not
  protocol).

### 2.2 `memdb` — in-memory (`BTreeMap`)

`parking_lot::RwLock<Option<BTreeMap<Vec<u8>, Vec<u8>>>>` (the `Option` models
Go's `db == nil` after `Close`). `BTreeMap` gives ordered iteration for free
(Go sorts a `map[string][]byte` snapshot per-iterator; we snapshot the relevant
range into a `Vec` to keep iterators independent of later mutation, matching
`TestIteratorSnapshot`). `get` clones the value (memory-safety contract).

### 2.3 `prefixdb` — key namespacing

Partitions a parent DB by prepending a fixed prefix. **Critical detail to copy
exactly:** `New(prefix, db)` stores `MakePrefix(prefix) = SHA-256(prefix)` (a
32-byte hashed prefix), and `New` on an already-`prefixdb` *joins* by hashing the
concatenation (`JoinPrefixes`); `NewNested` always hashes without compression.
Because real chains' on-disk keys are `SHA256(prefix) || key`, **`ava-database`
must reproduce `MakePrefix`/`JoinPrefixes` byte-for-byte** for a Rust node to read
a Go-written shared DB (see §7 migration). `dbLimit = increment(prefix)` for
range-bounded `Compact`. Iterators strip the prefix from returned keys. Use a
reusable byte buffer pool (mirror `utils.BytesPool`) to avoid per-op allocation.

### 2.4 `versiondb` — in-memory overlay with commit

An overlay over a base DB: `mem: HashMap<Vec<u8>, ValueOrDelete>`. Reads consult
`mem` first (a tombstone ⇒ `ErrNotFound`), else the base. `Put`/`Delete` only
touch `mem`. `Commit()` flushes `mem` into a base batch, writes it atomically,
then clears `mem`; `Abort()` clears `mem`. Also expose `CommitBatch()` (return
the batch without writing — used by callers that compose multi-DB atomic commits)
and `SetDatabase`/`GetDatabase`. The **merge iterator** (`iterator` in `db.go`)
walks the sorted `mem` snapshot and the base iterator in lockstep, preferring
`mem` on key ties and skipping tombstones — port the exact merge state machine
(`Next()` cases: exhausted-mem, exhausted-base, `memKey < dbKey`, `dbKey < memKey`,
equal). This backs P-Chain/X-Chain block-atomic state writes (cross-ref 08/09).

### 2.5 `meterdb` — Prometheus wrapper

Wraps any `Database`; times every method and records counts under the exact Go
metric names (the `method` label set: `has`, `get`, `put`, `delete`, `new_batch`,
`new_iterator`, `compact`, `close`, `health_check`, `batch_*`, `iterator_*`).
Uses `prometheus` histograms/counters. A metrics-name golden test guards parity
(cross-ref 02, §7.3).

### 2.6 `corruptabledb` — poison-on-error

Wraps any `Database`. On any error **other than** `Closed`/`NotFound`, it latches
an `initial_error` (under an `RwLock`) and every subsequent op returns it,
"closing the database to avoid possible corruption." Rust: `parking_lot::RwLock<
Option<Error>>`; the `handle_error` matcher latches only on `Error::Other(_)`. The
node wraps its base on-disk DB in this so a single IO fault halts writes rather
than corrupting state.

### 2.7 `linkeddb` — iteration over an unordered KV

Implements an in-DB doubly-linked list so a non-iterating store gains LIFO
iteration (`headKey = 0x01`; nodes carry `value/hasNext/next/hasPrevious/
previous`, **serialized with the linearcodec — cross-ref 03**, since these node
bytes are persisted). Port the head-key + node LRU caches (`lru` crate),
`updatedNodes` staging map, and the batch write of head + touched nodes. Used by
P-Chain (e.g. pending-staker lists). Provides its own `Iterator`.

### 2.8 `rpcdb` — a `Database` over gRPC (plugin protocol)

`proto/rpcdb/rpcdb.proto` defines a `Database` service
(`Has/Get/Put/Delete/Compact/Close/HealthCheck/WriteBatch/
NewIteratorWithStartAndPrefix/IteratorNext/IteratorError/IteratorRelease`). This
is how a `rpcchainvm` plugin reaches the host's DB (cross-ref 07 `ava-vm-rpc`).

- **Client (`DatabaseClient`)** — implements `Database` by calling a tonic-
  generated `DatabaseClient`. Errors come back as a `rpcdb.Error` enum
  (`ERROR_CLOSED`/`ERROR_NOT_FOUND`) mapped back to `Error::Closed`/`NotFound`
  (the `ErrEnumToError` table); any transport error is `Error::Other`. Iterators
  are server-side handles addressed by id; `IteratorNext` batches multiple pairs
  per RPC (port the batching the Go client uses to amortize round-trips). A
  client batch buffers `BatchOp`s and sends one `WriteBatch`.
- **Server (`DatabaseServer`)** — a tonic service wrapping a host `Database`;
  maps `Error::Closed/NotFound` → the enum (`ErrorToErrEnum`), passes other
  errors through as gRPC errors. Holds an iterator registry keyed by id.

Built on `tonic`; the `.proto` is the shared wire contract — **byte-exact** so a
Rust plugin and a Go host interoperate (overview §1 gRPC-plugin requirement).

### 2.9 `heightindexdb` — `HeightIndex`

A distinct, simpler interface (`database/heightindexdb`): `put/get/has(height:
u64)`, `sync(start,end)`, `close`. Backends: `memdb` (`HashMap<u64,Vec<u8>>`) and
`meterdb` (Prometheus wrapper). Has its **own** dbtest battery
(`heightindexdb/dbtest`). Used to index block bytes by height (e.g. proposervm).
Rust trait:

```rust
pub trait HeightIndex: Send + Sync {
    fn put(&self, height: u64, value: &[u8]) -> Result<()>;
    fn get(&self, height: u64) -> Result<Vec<u8>>;     // Err(NotFound) if absent
    fn has(&self, height: u64) -> Result<bool>;
    fn sync(&self, start: u64, end: u64) -> Result<()>;
    fn close(&self) -> Result<()>;
}
```

---

## 3. `ava-merkledb` — the path-based Merkle radix trie

> **This is protocol-critical:** `merkledb` **root hashes and proofs are on the
> network wire** (P-Chain merkle state-sync; the X/P state root tooling) and the
> persisted node bytes are part of certain chains' on-disk format. The Rust impl
> must produce **identical 32-byte root IDs and byte-identical proof encodings**.

### 3.1 Build-on-Firewood vs reimplement (BINDING DECISION)

**Decision: `ava-merkledb` is a faithful *reimplementation* of `x/merkledb`'s
node/key/hash/proof scheme** (not a thin Firewood wrapper) for the
avalanche-trie use cases (P-Chain/X-Chain merkle state, the `database/merkle/
sync` protocol). Rationale below in §3.6 and §4.

The scheme is small, fully specified, and its hash must match Go bit-for-bit; a
direct reimplementation is the lowest-risk path to byte-compatibility and lets us
keep the exact codec/hash without depending on a Firewood feature flag matching at
all times. **Firewood is used for EVM state and for new SAE state where Go itself
uses Firewood** (§4).

### 3.2 Key / `Path` representation

`merkledb` keys are bit-paths over a configurable **branch factor**
(2/4/16/256 ⇒ token sizes 1/2/4/8 bits; `BranchFactor256` = byte-per-token is
the production default). Port `x/merkledb/key.go`:

```rust
/// A bit-path. `value` holds packed tokens; `length` is in *bits*.
#[derive(Clone, PartialEq, Eq, Hash)]
pub struct Key { value: Bytes, length: usize }

pub enum BranchFactor { Two, Four, Sixteen, TwoFiftySix } // token_size: 1,2,4,8 bits
```

Reproduce exactly: `token(bit_index, token_size)` bit-extraction,
`ToToken`, `has_prefix`/`iterated_has_prefix`, `Skip`/`Take`, longest-common-
prefix, and partial-byte zero-padding rules. The unsafe `string`↔`[]byte`
aliasing in Go is just an allocation trick; Rust uses `Bytes`/`&[u8]`.

### 3.3 Node model & on-disk codec (`node.go`, `codec.go`)

```rust
struct DbNode { value: Maybe<Bytes>, children: BTreeMap<u8, Child> }
struct Child { compressed_key: Key, id: Id, has_value: bool }
/// In-memory enrichment:
struct Node { db_node: DbNode, key: Key, value_digest: Maybe<Bytes> }
```

- `value_digest` = the value itself if `len < HashLength(32)`, else
  `HashValue(value)` — reproduce `setValueDigest`.
- **Codec (byte-exact):** `encodeDBNode` writes `MaybeBytes(value)`, then
  `Uvarint(num_children)`, then for each child **in ascending index order**:
  `Uvarint(index)`, `Key(compressed_key)`, `ID(child_id)`, `Bool(has_value)`.
  Primitives: `Bool` = 1 byte `0x00/0x01`; `Uvarint` = `binary.PutUvarint` with a
  **no-leading-zeroes** decode check (`errLeadingZeroes`); `Bytes`/`Key` are
  uvarint-length-prefixed; `Key` packs `length` (bits) + `bytesNeeded(length)`
  bytes with the **partial-byte-must-be-zero-padded** rule (`errNonZeroKeyPadding`).
  Decode rejects: child index ≥ branch factor, too many children, int overflow,
  trailing bytes (`errExtraSpace`). These error conditions are part of the
  conformance surface (a Go node a Rust node both reject the same bad bytes).

### 3.4 Hashing (`hashing.go`) — MUST match Go byte-for-byte

`HashNode` (SHA-256, the `DefaultHasher`) feeds the hasher in this exact order:
1. `Uvarint(num_children)`.
2. For each child **in ascending byte-index order**: `Uvarint(index)` then the
   child's 32-byte `id`.
3. If the node has a value digest: `0x01`, then `Uvarint(len(digest))`, then the
   digest bytes. Else `0x00`.
4. `Uvarint(key.length)` (bits) then `key.Bytes()`.

`HashValue(v) = SHA-256(v)`. `HashLength = 32`. The root ID is `HashNode(root)`;
an empty trie hashes to `ids::EMPTY`. We provide the `Hasher` trait so the
SHA-256 hasher is swappable but default and protocol-fixed. A **golden-vectors
test** (real mainnet/fuji root IDs and hand-built tries) gates this (§6).

### 3.5 Views (`view.go`, `trie.go`), history (`history.go`), caches

- **`View`/`TrieView`** = an immutable proposal layering changes over a parent
  (DB or another view); lazily computes node IDs / root only when queried or
  committed. Reproduce the validity model: committing a view **invalidates
  siblings and their descendants** (`ErrInvalid`); a view commits only if its
  parent is the DB and only once. Rust: `Arc`-linked parent pointers + an
  `AtomicBool` validity flag + `arc_swap` for the committed-root swap. Change
  computation parallelizes node hashing across independent subtries
  (`wait_group.go` → rayon scope) — safe because hashing is pure (§6.1).
- **`history`** keeps a bounded ring of recent change-sets keyed by root ID so
  the node can answer change-proof requests; port the trim/size bound.
- **Two node stores:** `intermediate_node_db` (LRU-cached, hashed-key) and
  `value_node_db`, both over a base `Database`. `bytes_pool` and `cache` (LRU)
  port to `bytes`/`lru`.

### 3.6 Proofs (`proof.go`) — wire-critical

Reproduce, with byte-exact protobuf encodings (`proto/sync` + the merkledb proto):

```rust
pub struct ProofNode { key: Key, value_or_hash: Maybe<Bytes>, children: BTreeMap<u8, Id> }
pub struct Proof { path: Vec<ProofNode>, key: Key, value: Maybe<Bytes> }
pub struct KeyValue { key: Vec<u8>, value: Vec<u8> }
pub struct RangeProof { start_proof: Vec<ProofNode>, end_proof: Vec<ProofNode>, key_values: Vec<KeyValue> }
pub struct KeyChange { key: Vec<u8>, value: Maybe<Bytes> }
pub struct ChangeProof { start_proof: Vec<ProofNode>, end_proof: Vec<ProofNode>, key_changes: Vec<KeyChange> }
```

- **Single proof** — inclusion/exclusion; verify by rebuilding a trie from `path`
  and checking the recomputed root == expected root (the README's one-way-hash
  argument). `ProofNode.value_or_hash` = value if `len < 32`, else its hash.
- **`RangeProof`** — contiguous `[start,end]` slice; verify by building a trie
  from `key_values`, inserting the `start_proof`/`end_proof` boundary nodes, and
  checking the root. Invariants (sorted, no gaps, omit-from-end-on-truncation)
  ported verbatim.
- **`ChangeProof`** — key changes between `start_root` (exclusive) and `end_root`
  (inclusive); `KeyChange.value = Nothing` ⇒ deletion. Verification applies the
  changes to the local (partial) trie and checks the root equals
  `expected_end_root`. All the ordering/subset invariants in the Go doc comment
  are enforced.

These proof types and their protobuf marshalers are the payloads of the state-
sync protocol — see §3.7 and **cross-ref 06 (consensus/state-sync)**.

### 3.7 State-sync protocol (`database/merkle/sync`, `proto/sync`)

Port the client/server (`Syncer`, `network_server`, `workheap`) and the four
messages (`SyncGetRangeProofRequest` → `RangeProof`,
`SyncGetChangeProofRequest` → `SyncGetChangeProofResponse{ChangeProof|RangeProof}`)
defined in `proto/sync.proto`. The generic Go `sync.DB[R, C]` interface
(`GetMerkleRoot`, `GetChangeProof`, `GetRangeProof`, `VerifyChangeProof`, `Clear`,
`CommitRangeProof`, `CommitChangeProof`) becomes a Rust trait:

```rust
pub trait SyncDb: Send + Sync {
    type RangeProof; type ChangeProof;
    fn merkle_root(&self) -> Result<Id>;
    fn change_proof(&self, start_root: Id, end_root: Id,
        start: Option<&[u8]>, end: Option<&[u8]>, max_len: usize) -> Result<Self::ChangeProof>;
    fn range_proof(&self, root: Id, start: Option<&[u8]>, end: Option<&[u8]>,
        max_len: usize) -> Result<Self::RangeProof>;
    fn verify_change_proof(&self, p: &Self::ChangeProof, start: Option<&[u8]>,
        end: Option<&[u8]>, expected_end_root: Id) -> Result<()>;
    fn commit_range_proof(&self, start: Option<&[u8]>, end: Option<&[u8]>, p: Self::RangeProof) -> Result<()>;
    fn commit_change_proof(&self, p: Self::ChangeProof) -> Result<()>;
    fn clear(&self) -> Result<()>;
}
```

`ErrInsufficientHistory`/`ErrNoEndRoot` map to enum variants. The `workheap`
priority queue and `SimultaneousWorkLimit` concurrency port to a tokio task
set + `BinaryHeap`. **Both `ava-merkledb` and the Firewood path implement
`SyncDb`** — note that the Go tree already has a Firewood-backed `sync.DB`
(`database/merkle/firewood/syncer`, generic over `*RangeProof`/`struct{}`,
`EmptyRoot = types.EmptyRootHash`), proving the sync protocol is reused for EVM
state sync (§4.3).

---

## 4. Firewood integration (`firewood` crate, direct dep)

### 4.1 What Firewood is, and the hashing-compatibility conclusion (BINDING)

Firewood is Ava Labs' Rust Merkle-trie database and the **successor to merkledb**.
The decisive fact for this port, confirmed from Firewood's README and the Go
`firewood-go-ethhash` bindings:

> **Firewood has two hashing modes, selected by a Cargo feature:**
> - **default** → SHA-256 hashing **compatible with `x/merkledb`** (no
>   account/storage-trie semantics).
> - **`ethhash` feature** → Keccak-256 + Ethereum-MPT (RLP) node encoding, where
>   an "account" is a node at a fixed depth — i.e. the **C-Chain / EVM state
>   root** (`types.EmptyRootHash` is Firewood's empty root in the Go EVM
>   bindings).

So: **merkledb roots and Firewood-default roots are the *same scheme*; EVM
(ethhash) roots are a *different* scheme.** Overview §38 ("`merkledb` root hashes;
Firewood ethhash for EVM state") is exactly this split.

**Per-chain mapping (BINDING):**

| State | Hashing | Rust backing |
|---|---|---|
| P-Chain / X-Chain merkle state, `merkle/sync` protocol | merkledb SHA-256 | `ava-merkledb` (reimplementation) — authoritative for on-wire roots/proofs |
| C-Chain / EVM-subnet account+storage state | Firewood `ethhash` (Keccak/MPT) | `firewood` crate with `features=["ethhash"]`, via `ava-evm` (cross-ref 10) |
| SAE / new state where Go uses Firewood | Firewood (mode per Go) | `firewood` crate, via `ava-saevm` (cross-ref 11) |

We therefore **do not** try to make `ava-merkledb` a Firewood wrapper: the wire
roots for P/X come from our reimplementation (guaranteed byte-match via golden
vectors), and EVM state roots come from Firewood-ethhash (guaranteed by using the
same engine Go uses). A future re-genesis could collapse P/X onto Firewood-
default (the README notes the layouts are interchangeable) — out of scope now,
but the `SyncDb` trait keeps it swappable.

### 4.2 Firewood Rust API (as used by `ava-evm`/`ava-saevm`)

Firewood's `Db` API is **synchronous** (no async); we call it under a dedicated
commit thread / `spawn_blocking` (§1.2). Sketch:

```rust
use firewood::db::{Db, DbConfig, BatchOp};
use firewood::v2::api::{Db as _, DbView, Proposal as _}; // trait imports

// Open (revision history bounded by config).
let db = Db::new(path, DbConfig::builder().truncate(false).build())?;

// Propose a batch of changes against the current tip.
let batch = vec![
    BatchOp::Put { key: acct_key, value: rlp_account },
    BatchOp::Delete { key: dead_key },
];
let proposal = db.propose(batch)?;

// The proposal's root *before* commit — this is the block's state root.
let state_root: [u8; 32] = proposal.root_hash()?.expect("non-empty").into();

// Commit (durably writes nodes; advances the tip).
proposal.commit()?;

// Read at the current tip, or at any retained historical revision by root.
let view  = db.view()?;                 // latest
let hist  = db.revision(old_root.into())?;   // historical (if within retained window)
let value = view.val(&key)?;            // Option<Box<[u8]>>
```

Key properties leveraged:
- **`propose` is pipelineable:** proposals can be created and even stacked
  (proposal-on-proposal) before commit, and `commit` is decoupled from
  proposal/hashing. This maps directly onto **SAE's streaming async execution**
  (ACP-194): consensus computes the next state root via `propose` while a
  background thread `commit`s accepted blocks — **cross-ref 11**. The root hash
  must be available pre-commit (it is) so consensus can vote on it.
- **Historical revisions by root** back archival/`eth_getProof`-style queries and
  the EVM state-sync server (cross-ref 10), and replace the bespoke Go
  `x/archivedb` Firewood path mentioned in recent commits.

### 4.3 EVM-state wrapping (`ava-evm`, cross-ref 10)

`ava-evm` adapts Firewood-ethhash to reth's `StateProvider`/`HashedPostState`
abstractions: account/storage reads go through `db.view().val(...)`, block
execution produces a reth `BundleState` that is translated into Firewood
`BatchOp`s (RLP-encoded accounts at the account depth, storage slots in the
sub-trie), `propose` yields the EVM state root that goes into the block header,
and `commit` runs on accept. Firewood's range/change proofs serve `eth_getProof`
and EVM state sync (the Go `firewood/syncer` is the reference).

### 4.4 Safety / FFI

The `firewood` crate is pure Rust (no CGO shim — overview §1), so `ava-merkledb`/
`ava-evm` link it directly. `#![forbid(unsafe_code)]` stays on in our crates;
Firewood's internal `unsafe` is its own concern. Bounded revision retention
(Firewood's `RevisionManager`) is configured to cover the consensus
reorg/sync window.

---

## 5. `ava-blockdb` and `ava-archivedb`

### 5.1 `ava-blockdb` (`x/blockdb`)

Append-optimized, **height-indexed** block store with O(1) read/write, concurrent
access, out-of-order writes (bootstrap), optional fsync durability, zstd block
compression, and crash recovery. **On-disk format is reproduced exactly** (it is a
local persisted format; reproduce so a migrated data dir is readable — §7):

- One **index file** (`.idx`): 64-byte header `{version, max_data_file_size,
  min_height, max_height, next_write_offset, reserved[24]}` + fixed **16-byte
  entries** `{data_offset:u64, block_size:u32, reserved:u32}` → O(1) seek by
  height. All little/big-endian widths copied from the README/code.
- Multiple **data files** (`.dat`): each block = 22-byte entry header
  `{height:u64, size:u32, checksum:u64, version:u16}` + raw bytes; split across
  files at `max_data_file_size`.
- **Recovery:** on open, if `data_file_size > indexed_size`, scan from
  `next_write_offset`, validate header+checksum per block, rewrite index entries,
  recompute `max_height`.
- **Durability:** `sync_to_disk` ⇒ fsync data after each write + fsync index every
  `checkpoint_interval`; else rely on OS buffering.

Rust: memory-map or positioned `pread`/`pwrite` (`std::os::unix::fs::FileExt` /
`positioned-io`) under an `RwLock`-free design (atomic `next_write_offset`), an
`lru` block cache, `zstd` crate for compression, `crc`/`xxhash` matching the Go
checksum algorithm. Concurrency uses per-file handles so reads don't block writes.

### 5.2 `ava-archivedb` (`x/archivedb`)

Append-only, **height-versioned** KV over a base `Database`. The encoding is the
load-bearing part (reproduce byte-exact so a migrated dir reads):

- **User key →** `uvarint(len(key)) || key || BigEndian(^height)`. The bitwise-
  **negated** height suffix makes RocksDB's ascending order yield **descending
  height**, so a forward seek from a target height lands on the most-recent
  version at/below it.
- **Metadata key →** `uvarint(len(key)+1) || key` (the `+1` guarantees no overlap
  with any user-key prefix; `heightKey` stores the last written height).
- **Reads:** `Open(height)` returns a `Reader`; `reader.get(key)` seeks
  `prefix || ^height` forward and takes the first entry with that user-prefix
  (newest ≤ height); a tombstone value ⇒ `ErrNotFound`. `get_height(key)` also
  returns the height the value was set at.
- **Writes:** `NewBatch(height)` buffers `Put`/`Delete` (delete = a tombstone
  value, see `value.go`) all stamped with `height`, committed atomically.

```rust
pub struct ArchiveDb<D: Database> { db: D }
impl<D: Database> ArchiveDb<D> {
    pub fn height(&self) -> Result<u64>;
    pub fn new_batch(&self, height: u64) -> ArchiveBatch<'_, D>;
    pub fn open(&self, height: u64) -> Reader<'_, D>;
}
```

> **Note on Firewood archival.** Recent Go work moves archival queries onto
> Firewood (`CopyTrie()` for reconstructed tries; "Firewood archival queries").
> `ava-archivedb` is the generic-KV reference model; for EVM/SAE archival we use
> **Firewood historical revisions** (§4.2) rather than this KV scheme. Keep
> `ava-archivedb` for any chain that uses the KV-archive model in Go.

---

## 6. Test plan (cross-ref 02)

1. **`dbtest` conformance battery as a generic Rust fn.** Port every Go test in
   `database/dbtest` (`SimpleKeyValue`, `Overwrite`, `EmptyKey`, `KeyEmptyValue`,
   `*Closed`, `MemorySafety*`, `Batch*` incl. `BatchInner`/`BatchReplay`/
   `BatchLargeSize`, `Iterator*` incl. `IteratorSnapshot`/`IteratorError(AfterRelease)`,
   `Compact`, `Clear*`, `ModifyValueAfter*`, `ConcurrentBatches`,
   `ManySmallConcurrentKVPairBatches`, `PutGetEmpty`) plus the benchmarks as
   criterion benches. Expose as:
   ```rust
   pub fn run_database_conformance<D: Database>(make: impl Fn() -> D);
   pub fn run_basic_conformance<R: KeyValueReaderWriterDeleter>(make: impl Fn() -> R);
   pub fn run_heightindex_conformance<H: HeightIndex>(make: impl Fn() -> H);
   ```
   Run for `memdb`, `rocksdb`, `prefixdb`, `versiondb`, `meterdb`,
   `corruptabledb`, `rpcdb` (client↔server over an in-process tonic channel), and
   the heightindex memdb/meterdb. This *is* the differential test for the KV layer.
2. **Golden roots.** A vectors file of `(operations → merkledb root ID)` captured
   from the Go `x/merkledb` (incl. real P/X state roots) — `ava-merkledb` must
   reproduce every root and the byte-exact node/codec encoding.
3. **Proof verification.** Differential: generate range/change/single proofs with
   the Go node, verify with Rust (and vice-versa); proptest random tries + random
   ranges and assert verify-accepts-valid / verify-rejects-tampered.
4. **proptest trie invariants:** insert/remove/get against a `BTreeMap` oracle;
   root invariance (same kv-set ⇒ same root regardless of insertion order); view
   layering equals direct application; commit-invalidates-siblings.
5. **prefixdb/archivedb encoding goldens:** `SHA256(prefix)` namespacing and the
   `^height` key encoding must byte-match Go output.
6. **Firewood:** golden EVM state roots (ethhash) vs the Go
   `firewood-go-ethhash` bindings on identical batches; propose/commit/revision
   round-trips; proof interop with the Go EVM syncer.
7. **rpcdb wire interop:** a Rust `DatabaseClient` against a Go `rpcdb` server and
   vice-versa (overview gRPC-plugin requirement).
8. **blockdb recovery:** simulate torn writes (data file ahead of index) and
   assert the recovery scan rebuilds the index identically to Go.
9. **metrics-name golden** for `meterdb` (§7.3).

---

## 7. Migration from a Go data directory (Pebble/LevelDB concern)

Per overview §4.4: there is no mature pure-Rust Pebble, and the *generic* KV
on-disk layout is **not** part of the network protocol — so a node started
against an existing Go data dir is a **migration**, not in-place reuse, for the
base KV engine:

| Go backend | Rust handling |
|---|---|
| `leveldb` (goleveldb) | **Import tool.** `ava-database::migrate` opens the goleveldb dir read-only (via a `rust-rocksdb` *LevelDB-format* open path if viable, else a small goleveldb-format reader) and bulk-loads into RocksDB. RocksDB can open many classic LevelDB dirs directly; we attempt that first and fall back to copy. |
| `pebble` | **Import only — no in-place.** No Rust Pebble. Document non-support of in-place open; provide an offline converter that reads a Pebble dir (using a Pebble reader, or by running a one-shot Go export sidecar producing an SST/stream) and bulk-ingests into RocksDB. Call this out explicitly in `ava-config` docs. |
| `memdb` | n/a (ephemeral). |

**What MUST be byte-exact regardless of engine** (because it is layered *inside*
the KV values/keys and is read after migration): `prefixdb` `SHA256` namespacing,
`linkeddb` node codec, `archivedb` `^height` keys, `blockdb` file format, and all
`merkledb` node/proof encodings + roots. The migration only re-houses opaque
`(key,value)` pairs into RocksDB; the bytes inside are preserved verbatim, so the
higher layers keep working. A migration integration test loads a captured Go
mainnet snapshot subset and verifies merkledb roots match post-migration.

---

## 8. Go → Rust mapping

| Go | Rust |
|---|---|
| `database.Database` (interface) | `trait Database` + object-safe `trait DynDatabase`; `Arc<dyn DynDatabase>` |
| `KeyValueReaderWriterDeleter` | `KeyValueReader + KeyValueWriter + KeyValueDeleter` |
| `database.Batch` / `Batcher` / `BatchOps` | `trait Batch` / `trait Batcher` / `struct BatchOps` |
| `database.Iterator` / `Iteratee` | `trait Iterator` (+ `Drop` = release) / `trait Iteratee` (GAT) |
| `ErrClosed` / `ErrNotFound` | `Error::Closed` / `Error::NotFound` (matched like `errors.Is`) |
| `health.Checker.HealthCheck` | `fn health_check(&self) -> Result<serde_json::Value>` |
| `leveldb` + `pebbledb` | `ava-database::rocksdb` (`rust-rocksdb`) + migration (§7) |
| `memdb` (`map[string][]byte`) | `RwLock<Option<BTreeMap<Vec<u8>,Vec<u8>>>>` |
| `prefixdb` (`SHA256` prefix) | `PrefixDb` (reproduce `MakePrefix`/`JoinPrefixes`) |
| `versiondb` (`mem` + `Commit`) | `VersionDb` (`HashMap` overlay + merge iterator) |
| `meterdb` / `corruptabledb` | `MeterDb` (prometheus) / `CorruptableDb` (`RwLock<Option<Error>>`) |
| `linkeddb` | `LinkedDb` (linearcodec nodes + LRU caches) |
| `rpcdb` client/server | tonic `DatabaseClient`/`DatabaseServer` (`proto/rpcdb`) |
| `heightindexdb.HeightIndex` | `trait HeightIndex` (+ memdb/meterdb) |
| `x/merkledb` `Key`/`node`/`Hasher`/`View`/`Proof`/`RangeProof`/`ChangeProof` | `ava-merkledb` ports (byte-exact codec+hash) |
| `database/merkle/sync` `DB[R,C]`/`Syncer` | `trait SyncDb` + `Syncer` (tokio) |
| `firewood-go-ethhash/ffi` | `firewood` crate (`features=["ethhash"]`) directly |
| `x/blockdb` | `ava-blockdb` (same file format) |
| `x/archivedb` | `ava-archivedb` (same `^height` encoding) |
| `maybe.Maybe[[]byte]` | `enum Maybe<T> { Nothing, Some(T) }` (in `ava-types`/`ava-utils`, cross-ref 03) |

---

## 9. Performance notes / improvements over Go

All are behavior-preserving (§6.1); each is gated by the differential/golden tests
in §6.

- **Zero-copy reads.** RocksDB `get_pinned` and `bytes::Bytes`-backed iterator
  values avoid the per-`Get` clone Go does (`slices.Clone`) on read-only paths;
  we still copy where the trait contract promises a mutable-safe owned `Vec`.
- **Pipelined Firewood commits.** SAE (11) `propose`s the next root for consensus
  on the hot path and `commit`s accepted blocks on a background thread — Firewood
  is built for this; Go's libevm path is more serial. "Safe because" the root is
  computed at propose time, so the value consensus votes on is unchanged.
- **Parallel proof verification & node hashing.** Range/change-proof verification
  rebuilds independent subtries; merkledb view hashing is per-subtree — both fan
  out over rayon. "Safe because" hashing is a pure function of node contents.
- **Lock-granularity.** `memdb`/`versiondb` use `parking_lot` and snapshot ranges
  into per-iterator `Vec`s, allowing concurrent readers without the global
  re-sort Go performs on every `NewIterator` over the whole map.
- **RocksDB > LevelDB.** Column families, prefix bloom filters, and tuned write
  buffers give better compaction behavior than goleveldb under chain write loads,
  without changing observable semantics.
- **Bounded allocation.** Reusable byte pools for prefix construction and codec
  writers (mirroring Go's `BytesPool`) plus `smallvec` for child arrays keep the
  trie hot path allocation-light.

---

## 10. On-disk key-schema catalog

> **Why this section exists.** §7 establishes that the *generic* KV on-disk format
> (goleveldb/pebble vs RocksDB) is **not** a compatibility surface — a migration
> re-houses opaque `(key, value)` pairs. But the **keys and values themselves** —
> the prefix-namespacing, the byte-layout of every typed key, and the value codec —
> **are** a compatibility surface: they survive a migration verbatim and a Rust node
> must produce/read byte-identical bytes (cross-ref §7, and the R2 tool in §11). This
> section is the exhaustive catalog the Rust ports (`ava-platformvm` 08, `ava-avm` 09,
> `ava-evm` 10, `ava-indexer`, `ava-node`) must reproduce.

### 10.1 The namespacing model (how a key is built)

A node opens **one** base on-disk `Database` (leveldb/pebble in Go, RocksDB in Rust;
`node/node.go:initDatabase`). Everything else is `prefixdb` namespacing on top:

```
on-disk key = SHA256( … SHA256(SHA256(p1) ‖ p2) … ‖ pN )  ‖  typed_key
              \__________________ namespace (always 32 bytes) ____________/
```

- `prefixdb.New(prefix, db)` stores `MakePrefix(prefix) = SHA256(prefix)` (32 bytes).
  Nesting a prefixdb inside another `prefixdb` calls `JoinPrefixes(parentPrefix,
  childPrefix) = SHA256(parentPrefix32 ‖ childPrefix)` — i.e. the parent's *already-
  hashed* 32-byte prefix is concatenated with the **raw** child prefix and re-hashed.
  `prefixdb.NewNested(prefix, db)` **always** does `SHA256(prefix)` (no join
  compression) — used where the child prefix is itself variable/derived (e.g. the
  per-address UTXO index, the per-subnet chain DB).
- The **final** on-disk namespace is therefore always a 32-byte SHA256 digest,
  regardless of nesting depth. `ava-database` reproduces `MakePrefix`/`JoinPrefixes`
  byte-for-byte (§2.3). **Wrappers do not alter keys:** `meterdb`, `corruptabledb`,
  `versiondb`, `heightindexdb`, and `linkeddb` are all key-passthrough (versiondb
  overlays in memory then flushes the *same* keys; linkeddb adds its own node keys
  *inside* an already-namespaced prefixdb — see §10.6).

The tables below give the **prefix chain** (the human-readable raw prefixes that get
hashed, in order) and the **typed key** appended after the 32-byte namespace.

> **Reading convention for the tables:** "Prefix chain" lists raw prefixes
> outermost→innermost; the real namespace is the iterated SHA256 of that chain.
> "Key body" is what follows the 32-byte namespace. All multi-byte integers are
> **big-endian** (`database.PackUInt64`) unless noted. `bool` = 1 byte `0x00`/`0x01`.

### 10.2 Shared / node-level

Base prefixes applied directly on the node's base DB (`node/node.go`, `chains/manager.go`).

| Prefix chain (raw) | Key body | Value codec | Meaning |
|---|---|---|---|
| *(none — base DB root)* | `"genesisID"` | 32-byte hash | Genesis hash sanity key (`genesisHashKey`). |
| *(none — base DB root)* | `"ungracefulShutdown"` | nil | Present iff node is mid-run; deleted on clean shutdown. |
| `0x00` (`indexerDBPrefix`) | *(indexer subtree, §10.7)* | — | Tx/block/vtx indexer root. |
| `"shared memory"` | atomic shared-memory keys | atomic codec | Cross-chain shared memory (`atomic.NewMemory`); keyed by `(chainID, key)` pairs internal to `chain/atomic`. |
| `chainID[32]` | *(per-chain subtree)* | — | **Every chain** gets `prefixdb.New(ctx.ChainID[:], meterDB)`. All VM/bootstrap state below is nested under this. |
| `chainID[32]` → `"vm"` (`VMDBPrefix`) | *(VM subtree — §10.3/10.4/10.5)* | — | The VM's own DB handle (`vmDB`). |
| `chainID[32]` → `"vertex"` (`VertexDBPrefix`) | vertex bytes | avm vertex codec | DAG vertex store (Avalanche/X-Chain linearization). |
| `chainID[32]` → `"vertex_bs"` / `"tx_bs"` / `"interval_block_bs"` | bootstrap job keys | bootstrapper codec | Bootstrapping queues for `LinearizableVM`s. |
| `chainID[32]` → `"interval_bs"` (`ChainBootstrappingDBPrefix`) | interval keys | bootstrapper codec | Bootstrapping interval tree for `ChainVM`s. |

> The validator/uptime manager persists **inside the P-Chain's** `vm` subtree (the
> `validators` prefix, §10.3) — there is no separate top-level validator DB.

### 10.3 P-Chain (`vms/platformvm/state`) — flat prefixed KV, **not** merkledb

**Confirmed (cross-ref 08 §3.2):** P-Chain state is flat prefixed key/value over the
base `Database`; there is **no** merkledb / state Merkle root. All prefixes below are
nested under `chainID[P] → "vm"` (i.e. `baseDB` in `state.go` is the P-Chain `vmDB`,
itself wrapped in a `versiondb` for block-atomic commits). Raw prefix string is from
`state.go`'s `var` block.

| Prefix chain (under P-Chain `vm`) | Key body | Value codec | Meaning |
|---|---|---|---|
| `"blockID"` | `PackUInt64(height)` (8B BE) | `ID` (32B) via `database.PutID` | Height → accepted block ID index. |
| `"block"` | `blockID` (32B) | raw block bytes | Block store. |
| `"txs"`† | `txID` (32B) | tx bytes ‖ status (`stateBlk`/tx codec) | Accepted tx store. († raw prefix is `"tx"`.) |
| `"utxo"` | `utxoID` (32B) | UTXO bytes (avax codec) | The flat UTXO set. |
| `"rewardUTXOs"` → *(nested `prefixdb.New(txID)`)* → linkeddb | `utxoID` (32B) | UTXO bytes | Per-tx reward UTXOs (linkeddb list inside a per-txID nested prefixdb). |
| `"subnet"` → linkeddb | `txID` (32B) | nil | Created subnets (existence set). |
| `"subnetOwner"` | `subnetID` (32B) | owner bytes (codec) | Subnet owner `fx.Owner`. |
| `"subnetToL1Conversion"` | `subnetID` (32B) | `conversionID ‖ chainID ‖ addr` (codec) | ACP-77 subnet→L1 conversion record. |
| `"transformedSubnet"` | `subnetID` (32B) | `TransformSubnetTx` bytes | Elastic-subnet transform record. |
| `"supply"` | `subnetID` (32B) | `PackUInt64` (8B BE) | Per-subnet current supply. |
| `"chain"` → *(nested `prefixdb.New(subnetID)`)* → linkeddb | `chainID` (32B) | nil | Chains created per subnet. |
| `"expiryReplayProtection"` | `timestamp(8B BE) ‖ validationID(32B)` | nil | ACP-77 expiry replay set (`ExpiryEntry.Marshal`). |
| `"singleton"` | one of the singleton keys ↓ | per-key (below) | Chain-wide scalars. |

**Validators subtree** — root `"validators"`, then `"current"`/`"pending"`, then the
staker-type prefix, then a **linkeddb** list keyed by `txID`:

| Prefix chain (under P-Chain `vm`) | Key body | Value codec | Meaning |
|---|---|---|---|
| `"validators"`→`"current"`→`"validator"`→list | `txID` (32B) | metadata (MetadataCodec, **v2**) — `UpDuration ‖ LastUpdated ‖ PotentialReward ‖ PotentialDelegateeReward`… | Current primary/subnet validator metadata. |
| `"validators"`→`"current"`→`"delegator"`→list | `txID` (32B) | `PackUInt64(potentialReward)` | Current delegator. |
| `"validators"`→`"current"`→`"subnetValidator"`→list | `txID` (32B) | metadata (MetadataCodec v2) | Current subnet validator (uptime + rewards). |
| `"validators"`→`"current"`→`"subnetDelegator"`→list | `txID` (32B) | `PackUInt64(potentialReward)` | Current subnet delegator. |
| `"validators"`→`"pending"`→`{validator,delegator,subnetValidator,subnetDelegator}`→list | `txID` (32B) | nil | Pending stakers (value carries no metadata). |

**Validator diffs** (the `inverseHeight` trick — note the **bit-flipped** height so
ascending RocksDB order yields descending height; `packIterableHeight = ^height`):

| Prefix chain (under P-Chain `vm`) | Key body | Value codec | Meaning |
|---|---|---|---|
| `"validators"`→`"flatValidatorDiffs"` | `subnetID(32B) ‖ ^height(8B BE) ‖ nodeID(20B)` | `isNegative(1B) ‖ weight(8B BE)` | Weight diffs, by-subnet seek order. |
| `"validators"`→`"flatValidatorDiffsByHeight"` | `^height(8B BE) ‖ subnetID(32B) ‖ nodeID(20B)` | `isNegative(1B) ‖ weight(8B BE)` | Weight diffs, by-height seek order. |
| `"validators"`→`"flatPublicKeyDiffs"` | `subnetID(32B) ‖ ^height(8B BE) ‖ nodeID(20B)` | uncompressed BLS pubkey or nil | Public-key diffs, by-subnet. |
| `"validators"`→`"flatPublicKeyDiffsByHeight"` | `^height(8B BE) ‖ subnetID(32B) ‖ nodeID(20B)` | uncompressed BLS pubkey or nil | Public-key diffs, by-height. |

**L1 / ACP-77 validators** — root `"validators"`→`"l1"`:

| Prefix chain (under P-Chain `vm`) | Key body | Value codec | Meaning |
|---|---|---|---|
| `"validators"`→`"l1"`→`"weights"` | `subnetID` (32B) | `PackUInt64(weight)` | Total L1 validator weight per subnet. |
| `"validators"`→`"l1"`→`"subnetIDNodeID"` | `subnetID(32B) ‖ nodeID(20B)` | `validationID` (32B) | (subnet,node) → validationID lookup. |
| `"validators"`→`"l1"`→`"active"` | `validationID` (32B) | `L1Validator` (`GenesisCodec`) | Active L1 validators. |
| `"validators"`→`"l1"`→`"inactive"` | `validationID` (32B) | `L1Validator` (`GenesisCodec`) | Inactive (weight-0/PoA) L1 validators. |

**Singletons** (`"singleton"` namespace, fixed string keys):

| Key body | Value codec | Meaning |
|---|---|---|
| `"initialized"` | nil | Genesis applied. |
| `"blocks reindexed.3"` | nil | Block-reindex migration marker. |
| `"timestamp"` | `time.Time` binary | Chain time. |
| `"fee state"` | fee-state codec | Dynamic fee state. |
| `"l1Validator excess"` | codec | ACP-77 fee excess. |
| `"accrued fees"` | codec | Accrued fees. |
| `"current supply"` | `PackUInt64` | Primary-network supply. |
| `"last accepted"` | `ID` (32B) | Last accepted block ID. |
| `"heights indexed"` | `startHeight ‖ endHeight` | Range safe for native height index. |

### 10.4 X-Chain (`vms/avm/state`) — flat prefixed KV

Nested under `chainID[X] → "vm"`. Note the X-Chain uses **single-byte** singleton
keys (`0x00/0x01/0x02`), unlike the P-Chain's string keys.

| Prefix chain (under X-Chain `vm`) | Key body | Value codec | Meaning |
|---|---|---|---|
| `"utxo"` | `utxoID` (32B) | UTXO bytes (avax codec) | Flat UTXO set (`utxoState`, §10.5). |
| `"index"` → *(nested `prefixdb.NewNested(addr)`)* → linkeddb | `utxoID` (32B) | nil | Per-address UTXO index. |
| `"tx"` | `txID` (32B) | tx bytes (avm codec) | Accepted tx store. |
| `"blockID"` | `PackUInt64(height)` (8B BE) | `ID` (32B, `PutID`) | Height → block ID. |
| `"block"` | `blockID` (32B) | raw block bytes | Block store. |
| `"singleton"` | `0x00` `isInitialized` | nil | Genesis applied. |
| `"singleton"` | `0x01` `timestamp` | `time.Time` binary | Chain time. |
| `"singleton"` | `0x02` `lastAccepted` | `ID` (32B) | Last accepted block ID. |

> The `"utxo"`/`"index"` pair above is the shared `vms/components/avax.utxoState`
> (`utxoPrefix="utxo"`, `indexPrefix="index"`) used by **both** P-Chain and X-Chain;
> the P-Chain version is the `"utxo"` row in §10.3.

### 10.5 C-Chain (Avalanche side — `graft/coreth/plugin/evm`)

The EVM **execution** state (accounts/storage) and blocks/receipts use a *different*
scheme — Firewood-ethhash for state, reth-db for chain data — documented in
**spec 10**; do **not** treat those as prefixdb keys. The keys below are the
**avalanche-side** bookkeeping the coreth VM keeps in the node's base DB (under
`chainID[C] → "vm"`), all reproduced byte-exact for migration.

| Prefix chain (under C-Chain `vm`) | Key body | Value codec | Meaning |
|---|---|---|---|
| `"ethdb"` (`prefixdb.NewNested`) | go-ethereum `rawdb` keys | rlp / ethdb | The wrapped EVM chain DB namespace (state/blocks live here in Go; the Rust port replaces the *contents* per spec 10 but the namespace prefix is the compatibility boundary). |
| `"snowman_accepted"` (`acceptedPrefix`, over versiondb) | `"last_accepted_key"` | block hash | Snowman last-accepted EVM block (`acceptedBlockDB`). |
| `"metadata"` (over versiondb) | metadata keys | codec | VM metadata (`metadataDB`). |
| `"warp"` | warp message keys | warp codec | Warp/AWM message DB. |

**Atomic trie & atomic-tx index** (`atomic/state/atomic_repository.go`,
`atomic_trie.go`) — the cross-chain (shared-memory) trie the C-Chain commits over:

| Prefix chain (under C-Chain `vm`) | Key body | Value codec | Meaning |
|---|---|---|---|
| `"atomicTxDB"` | `txID` (32B) | `height(8B BE) ‖ txBytes` | Accepted atomic tx → height+tx. |
| `"atomicHeightTxDB"` | `PackUInt64(height)` (8B BE) | `height(8B BE) ‖ packed txs` | Height → atomic txs at that height. |
| `"atomicRepoMetadataDB"` | `"maxIndexedAtomicTxHeight"` | `PackUInt64` | Last height the repo indexed. |
| `"atomicTrieDB"` | merkle-trie node keys | trie node bytes | The atomic-trie node store (merkleized; root committed every `commitInterval`). |
| `"atomicTrieMetaDB"` | `PackUInt64(height)` + `"atomicTrieLastCommittedBlock"` | trie root (32B) | Height→root index + last-committed pointer. |
| *(atomic-trie leaf keys, inside `"atomicTrieDB"`)* | `height(8B BE) ‖ blockchainID(32B)` (`TrieKeyLength=40`) | atomic `Requests` bytes | Per (height, peer-chain) atomic requests — the trie's logical KV. |
| `"atomicTrieLastAppliedToSharedMemory"` (metadata) | — | cursor | Shared-memory apply cursor. |

### 10.6 `linkeddb` node keys (applies to every "→ list" above)

Wherever a table says **"→ list"**, the bytes physically written inside that prefixdb
are **linkeddb** nodes, not bare values (cross-ref §2.7). For each logical entry the
store writes `key → linearcodec(node{ value, hasNext, next, hasPrevious, previous })`,
plus a head pointer at the fixed key `0x01` (`headKey`). The Rust `LinkedDb` must
reproduce this node codec byte-exact so a migrated list iterates identically.

### 10.7 Indexer (`indexer/`)

Root is the node-level `0x00` (`indexerDBPrefix`). Per indexed chain the indexer
builds a prefix `chainID(32B) ‖ kind(1B)` where `kind ∈ {0x01 tx, 0x02 vtx,
0x03 block}` (`registerChainHelper`), then nests two sub-spaces under a `versiondb`:

| Prefix chain | Key body | Value codec | Meaning |
|---|---|---|---|
| `0x00` → `chainID‖kind` | `0x00` (`nextAcceptedIndexKey`) | `PackUInt64` | Next accepted index counter. |
| `0x00` → `chainID‖kind` → `0x01` (`indexToContainerPrefix`) | `PackUInt64(index)` (8B BE) | `Container` (indexer `Codec`) | index → container (bytes + ID + timestamp). |
| `0x00` → `chainID‖kind` → `0x02` (`containerToIDPrefix`) | `containerID` (32B) | `PackUInt64(index)` | container ID → index reverse lookup. |

### 10.8 merkledb on-disk node/value keys (reference)

For chains that *do* use `x/merkledb` on disk (SAE/new state; **not** P/X), the base
DB holds three flat key spaces (`x/merkledb/db.go`), distinguished by a 1-byte
prefix prepended to the trie key (`addPrefixToKey`):

| 1-byte prefix | Key body | Value | Meaning |
|---|---|---|---|
| `0x00` (`metadataPrefix`) | `"cleanShutdown"` / `"root"` | `0x01`/`0x00` ; root-node key bytes | Clean-shutdown flag (`rebuild` trigger) + persisted root key. |
| `0x01` (`valueNodePrefix`) | `Key.Bytes()` (packed bit-path) | node bytes (encodeDBNode) | Value-node store (always durable). |
| `0x02` (`intermediateNodePrefix`) | `Key.Bytes()` | node bytes | Intermediate-node store (rebuildable from value nodes). |

These are governed by the byte-exact node/codec/hash rules in §3 (the **wire**
compatibility surface), and are independent of the prefixdb namespacing above (a
merkledb is itself opened over a `prefixdb`-namespaced base DB).

---

## 11. Migrating a Go data dir (R2)

> Closes **overview §11.2 risk R2**. Builds on §7 (which states the policy: import,
> not in-place open). This section is the **tool design**.

### 11.1 The problem

A Go avalanchego node persists its base DB as either **goleveldb** (`db/v1.4.5/`,
pre-v1.10.15 default) or **Pebble** (`pebble/`). The Rust node's only on-disk backend
is **RocksDB** (§2.1, per overview §4.4 — there is no production-grade pure-Rust
Pebble, and RocksDB was chosen for closest leveldb/pebble semantics). The two engines
share neither file format nor SSTable layout, so a Rust node **cannot open a Go data
dir in place**. But — crucially — **everything layered *inside* the KV pairs is
byte-identical** (the entire §10 catalog: prefixdb SHA256 namespaces, linkeddb node
codec, archivedb `^height` keys, blockdb file format, merkledb node/proof bytes). So
the migration problem reduces to: **copy every `(key, value)` pair from the Go engine
into RocksDB, verbatim.** No transformation of key or value bytes is required.

### 11.2 Decision: offline import tool (not in-place, not a runtime shim)

**Decision (BINDING):** ship an **offline, one-shot import tool** that reads the Go DB
and bulk-loads RocksDB, run before first Rust-node start. Rejected alternatives:

- *In-place open of the Go dir* — impossible for Pebble (no Rust reader), fragile for
  leveldb; rejected.
- *A runtime translation layer* (Rust node reads leveldb/pebble live) — would require
  embedding a pebble reader forever and forgo RocksDB's tuning; rejected.
- *Re-deriving everything from genesis by replay* — far slower than network sync; the
  network-bootstrap alternative (§11.5) is the supported "no tool" path instead.

### 11.3 Reading the Go DB

| Go backend | Reader strategy |
|---|---|
| **goleveldb** (`db/v1.4.5/`) | **Primary:** RocksDB can open many classic LevelDB directories directly — attempt `rust-rocksdb` open in LevelDB-compat mode first and, if it succeeds, the "migration" is just an in-place compaction/ingest (fast path). **Fallback:** a pure-Rust leveldb reader — evaluate **`rusty-leveldb`** (maintained, reads SSTables + MANIFEST + log) for a streaming `Iterator` over all pairs. |
| **Pebble** (`pebble/`) | **No pure-Rust Pebble reader exists.** Two options: (a) a small **Go export sidecar** (`avalanchego-db-export`, ~80 LOC, links the real `database/pebbledb`) that streams every pair as a length-prefixed `(key,value)` framing on stdout / into RocksDB SSTs via the SST writer; or (b) shell out to that sidecar from the Rust CLI. The sidecar reuses Go's own pebble open path, so it is guaranteed to read any dir a Go node wrote. |
| **memdb** | n/a (ephemeral). |

The CLI auto-detects the backend from the directory layout (`v1.4.5/` ⇒ leveldb,
`pebble/` ⇒ pebble) but `--db-type` overrides.

### 11.4 The tool: iterate + write + verify

```rust
// `avalanchego db migrate --from <godir> --to <rocksdir> --db-type {leveldb|pebble}`
//                          [--verify {none|roots|full}] [--resume]

use ava_database::{rocksdb::RocksDb, Database, KeyValueReader};

/// A backend-agnostic source: every Go pair, in lexicographic key order.
trait GoDbSource {
    /// Yields (key, value) verbatim — NO transformation. Lexicographic order
    /// lets us drive a RocksDB SstFileWriter for a bulk ingest (fastest path).
    fn iter_all(&self) -> anyhow::Result<Box<dyn Iterator<Item = (Vec<u8>, Vec<u8>)>>>;
}
// impls: RustyLevelDbSource (rusty-leveldb), PebbleSidecarSource (spawns the Go
// export sidecar and parses its length-prefixed stream), RocksDbCompatSource
// (rust-rocksdb opened in leveldb-compat mode, for the leveldb fast path).

fn migrate(src: &dyn GoDbSource, dst: &RocksDb, resume_after: Option<&[u8]>) -> anyhow::Result<u64> {
    let mut count = 0u64;
    let mut batch = dst.new_batch();
    let mut last_key: Vec<u8> = Vec::new();
    for (key, value) in src.iter_all()? {
        // Idempotency / --resume: skip anything already migrated.
        if let Some(after) = resume_after {
            if key.as_slice() <= after { continue; }
        }
        // Byte-for-byte: prefixdb namespaces, codec values, ^height keys, etc.
        // are ALL preserved because we never touch the bytes (§10).
        batch.put(&key, &value)?;
        last_key = key;
        count += 1;
        if batch.size() >= 64 * 1024 * 1024 {   // 64 MiB flush window
            batch.write()?;
            dst.put(MIGRATION_CURSOR_KEY, &last_key)?; // resume checkpoint
            batch = dst.new_batch();
        }
    }
    batch.write()?;
    dst.put(MIGRATION_CURSOR_KEY, &last_key)?;
    Ok(count)
}

/// Verify the migration without trusting the copy loop.
fn verify(dst: &RocksDb, level: VerifyLevel) -> anyhow::Result<()> {
    // roots: cheap structural re-derivation of the high-value compatibility
    // surfaces — re-read the singletons and re-hash any merkleized state.
    if level >= VerifyLevel::Roots {
        // P-Chain / X-Chain: re-read "singleton"→"last accepted" and confirm the
        // referenced block parses, and that "blockID"→PackUInt64(height) chains
        // back from it (these are flat KV — no state root to recompute, §10.3/10.4).
        check_last_accepted_chain(dst)?;
        // SAE / EVM chains: re-open the merkledb / Firewood over the migrated DB
        // and assert merkle_root() equals the root stored in the last block header
        // (the on-wire compatibility surface, §3 / §4).
        for chain in merkleized_chains(dst)? {
            let recomputed = chain.merkle_root()?;
            anyhow::ensure!(recomputed == chain.expected_root_from_header()?,
                "merkle root mismatch after migration for {chain:?}");
        }
    }
    // full: count + per-pair re-read of a random/sampled subset against the source.
    Ok(())
}
```

Key properties:

- **Byte-for-byte.** The loop never decodes a key or value; §10's entire catalog
  rides along untouched. This is what makes a one-pass copy correct.
- **Bulk-ingest fast path.** Because `iter_all` yields keys in lexicographic order,
  the leveldb/pebble source can feed a RocksDB `SstFileWriter` and `ingest_external_
  file` instead of batched `Put`s — multi-GB dirs migrate in minutes.
- **Idempotent / resumable.** A `MIGRATION_CURSOR_KEY` records the last-written key;
  `--resume` re-runs skip already-copied pairs. A clean re-run over a complete dir is
  a no-op (every `put` rewrites identical bytes).
- **Verification tiers.** `--verify roots` (default) re-derives the load-bearing
  surfaces: for flat-KV P/X it walks the last-accepted block chain; for merkleized
  SAE/EVM state it re-opens the trie and checks `merkle_root()` against the block
  header — the same golden-root guarantee as §6.2/§6.6, now applied post-migration.
  `--verify full` additionally samples random pairs back against the source.

### 11.5 Alternative: bootstrap fresh from the network (often simpler)

A node does **not** need the old data dir at all — it can start empty and obtain state
from peers. For most operators this is the **recommended** path because it avoids the
import tool entirely and yields a clean RocksDB:

- **State sync** (cross-ref **19**/06): C-Chain/SAE chains state-sync the merkle/
  Firewood state to a recent root, then accept forward — minutes-to-hours, independent
  of history depth.
- **Bootstrapping**: P/X (flat KV, no state sync) bootstrap by downloading and
  re-executing accepted blocks from peers.

**Recommendation per chain size:**

| Situation | Recommended path |
|---|---|
| Validator with deep C-Chain/SAE archive, downtime-sensitive | **Import tool** (`db migrate`, SST bulk-ingest + `--verify roots`) — avoids re-syncing terabytes. |
| Fresh/pruned node, or small P/X-only validator | **Network bootstrap / state-sync** — simpler, no tool, clean DB. |
| goleveldb dir, RocksDB opens it in compat mode | **Import fast path** (in-place ingest) — trivially cheap. |
| Pebble dir, archival, very large | **Import via Go export sidecar** (only correctness-guaranteed Pebble reader). |

Either way the byte-exactness of §10 is the safety net: a network-bootstrapped Rust
node and an import-migrated Rust node converge on identical on-disk bytes, so the two
paths are interchangeable for correctness — only operationally different.
