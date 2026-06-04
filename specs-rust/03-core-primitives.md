# 03 — Core Primitives (`ava-types`, `ava-codec`, `ava-crypto`, `ava-utils`, `ava-version`)

> Conforms to `00-overview-and-conventions.md`. This spec is the foundation every
> other spec depends on. The codec, sampler, crypto address derivation, and
> NodeID derivation here MUST be **byte-for-byte** identical to the Go node; they
> are part of the network/consensus protocol surface (see `00` §1, §6.1).

## 0. Go source coverage

| Rust crate | Go source mirrored |
|---|---|
| `ava-types` | `ids/` (`id.go`, `short.go`, `node_id.go`, `aliases.go`, `bits.go`, `request_id.go`), `utils/constants/network_ids.go` |
| `ava-codec` (+ `ava-codec-derive`) | `codec/` (`codec.go`, `manager.go`, `registry.go`, `general_codec.go`, `linearcodec/`, `reflectcodec/`), `utils/wrappers/packing.go` |
| `ava-crypto` | `utils/crypto/secp256k1/`, `utils/crypto/bls/` (+`signer/localsigner`), `staking/` (`tls.go`, `parse.go`, `verify.go`, `certificate.go`), `utils/hashing/`, `utils/cb58/`, `utils/formatting/` (`encoding.go`, `address/`) |
| `ava-utils` | `utils/set/`, `utils/bag/`, `utils/sampler/`, `utils/linked/`, `utils/math/`, `utils/wrappers/`, `utils/bloom`, `utils/window`, `utils/timer`, `utils/units`, `utils/constants` |
| `ava-version` | `version/` (`application.go`, `constants.go`, `compatibility.go`), `upgrade/upgrade.go` |

The dependency order inside this layer: `ava-types` → `ava-codec` (depends on
`ava-types` only for `Packer`-free primitive newtypes; the `Packer` itself lives
in `ava-codec`) ; `ava-crypto` depends on `ava-types`; `ava-utils` is mostly
standalone; `ava-version` depends on `ava-types` (for `Id`).

> **See also (appended sections):** §10 **Deterministic RNG (R1 resolution)** —
> the exact gonum MT19937 / MT19937-64 port that the sampler (§4.1) and the
> proposervm windower depend on; and §11 **Network-upgrade fork gating** — the
> shared `Fork` / `UpgradeConfig` model and full activation table used identically
> by every VM (cross-ref `06`, `08`, `09`, `10`, `11`).

> **Packer placement decision.** Go puts `Packer` in `utils/wrappers`. We place it
> in **`ava-codec`** (it is the codec wire engine) and re-export nothing from
> `ava-utils`. `ava-types` newtypes that need to pack (e.g. `Id::prefix`) call
> into `ava-codec` — but to avoid a cycle, `Id::prefix`/`Id::append` are
> implemented with a tiny inline big-endian writer in `ava-types` (they only pack
> `u64`/`u32`/fixed-bytes, never the full codec). Document this in `ava-types`.

---

## 1. `ava-types` — IDs and primitive newtypes

### 1.1 Fixed-size IDs

Go: `type ID [32]byte`, `type ShortID [20]byte`, `type NodeID ShortID`. All are
value types with `Compare` = `bytes.Compare`, CB58 string form, JSON as a quoted
CB58 string.

```rust
/// 32-byte identifier (block IDs, tx IDs, chain IDs, subnet IDs, …). Mirrors `ids.ID`.
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord, Default)]
pub struct Id([u8; 32]);          // Ord derives lexicographic == bytes.Compare ✓

/// 20-byte identifier. Mirrors `ids.ShortID`.
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord, Default)]
pub struct ShortId([u8; 20]);

/// Node identifier. Distinct newtype (NOT a type alias) so the `NodeID-` prefix
/// and JSON form differ from ShortId. Mirrors `ids.NodeID`.
#[derive(Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord, Default)]
pub struct NodeId([u8; 20]);

pub const ID_LEN: usize = 32;
pub const SHORT_ID_LEN: usize = 20;
pub const NODE_ID_LEN: usize = 20;
pub const NODE_ID_PREFIX: &str = "NodeID-";  // ids/node_id.go:17
```

Constructors / invariants (mirror Go exactly):

- `Id::from_slice(&[u8]) -> Result<Id>` — errors `InvalidHashLen` if `len != 32`
  (Go `hashing.ToHash256`). Same for `ShortId` at 20.
- `Id::from_str` / `Display` use **CB58** (no prefix). `ShortId` likewise.
  `NodeId::Display` = `"NodeID-" + cb58(bytes)` (Go `NodeID.String`).
  `NodeId::from_str` requires the `NodeID-` prefix (Go `NodeIDFromString` →
  `ShortFromPrefixedString`).
- JSON: serialize as a quoted string of the `Display` form; deserialize the
  inverse. The Go behavior treats the literal `null` as a no-op (leaves value
  unchanged) — replicate for serde via a custom `Deserialize` (`null` ⇒ keep
  `Default`). Empty / wrong-prefix errors mirror `errMissingQuotes`,
  `errShortNodeID`.
- `Id::EMPTY = Id([0;32])` (`ids.Empty`); `ShortId::EMPTY`; `NodeId::EMPTY`.
- Hex: `Id::hex()` = lowercase hex, no `0x` (Go `id.Hex`).

**`Id::prefix` (consensus-relevant — `ids/id.go:97`).** Concatenate
`be_u64(prefix_i)` for each prefix, then the 32 id bytes, then `sha256` → new
`Id`:

```rust
impl Id {
    pub fn prefix(&self, prefixes: &[u64]) -> Id {
        let mut buf = Vec::with_capacity(prefixes.len() * 8 + 32);
        for p in prefixes { buf.extend_from_slice(&p.to_be_bytes()); }
        buf.extend_from_slice(&self.0);
        Id(sha256(&buf))
    }
    /// ACP-77 validationID derivation. `ids/id.go:116`.
    pub fn append(&self, suffixes: &[u32]) -> Id {
        let mut buf = Vec::with_capacity(32 + suffixes.len() * 4);
        buf.extend_from_slice(&self.0);
        for s in suffixes { buf.extend_from_slice(&s.to_be_bytes()); }
        Id(sha256(&buf))
    }
    pub fn xor(&self, other: &Id) -> Id { /* byte-wise xor */ }
    /// bit i: byteIndex = i/8, bit = (b >> (i%8)) & 1. `ids/id.go:140`.
    pub fn bit(&self, i: usize) -> u8 { (self.0[i / 8] >> (i % 8)) & 1 }
}
```

> `sha256` here is the raw single-pass `sha2::Sha256` digest (NOT double-hash).

### 1.2 Bit helpers (`ids/bits.go`)

Port `equal_subset(start, stop, &Id, &Id) -> bool` and
`first_difference_subset(...) -> Option<usize>` verbatim — used by the Avalanche
consensus DAG/patricia routing. `NUM_BITS = 256`, `BITS_PER_BYTE = 8`. Bit
indexing convention: index 7 is the MSB of byte 0 (LSB-first within a byte). Use
`u8::trailing_zeros` for `bits.TrailingZeros8`. **Keep the exact masking/shift
arithmetic** — this is consensus-affecting in the Avalanche engine.

### 1.3 RequestID and Aliaser

```rust
/// `ids.RequestID`. Plain value type, field order irrelevant (not serialized).
#[derive(Clone, Copy, PartialEq, Eq, Hash)]
pub struct RequestId { pub node_id: NodeId, pub chain_id: Id, pub request_id: u32, pub op: u8 }
```

`Aliaser` (`ids/aliases.go`): a bidirectional map `alias→id` and `id→Vec<alias>`,
thread-safe (`parking_lot::RwLock`). An alias maps to exactly one id; one id may
have many aliases; first alias is "primary". `primary_alias_or_default` falls
back to `id.to_string()`. `get_relevant_aliases` strips the redundant
`alias == id.String()` self-alias.

### 1.4 Network constants (`utils/constants/network_ids.go`)

Copy verbatim into `ava-types::constants` (doc-comment each with the Go path):

```rust
pub const MAINNET_ID: u32 = 1;
pub const FUJI_ID: u32 = 5;          // == TESTNET_ID
pub const LOCAL_ID: u32 = 12345;
pub const MAINNET_HRP: &str = "avax";
pub const FUJI_HRP: &str = "fuji";
pub const LOCAL_HRP: &str = "local";
pub const FALLBACK_HRP: &str = "custom";   // GetHRP default
// also: cascade/denali/everest/testing HRPs for historical networks
pub const PRIMARY_NETWORK_ID: Id = Id::EMPTY;   // ids.Empty
```

`get_hrp(network_id) -> &str` returns the mapped HRP or `FALLBACK_HRP`.
`network_name(id)` and `network_id(name)` mirror the bidirectional maps.

---

## 2. `ava-codec` — the linear codec (BYTE-EXACT)

This is the single most compatibility-critical module in this spec. Go's codec is
**reflection-driven**; ours is **derive-macro-driven** but must produce identical
bytes. The wire format is: big-endian integers, `u32`-length-prefixed slices and
maps, `u16`-length-prefixed strings, fixed arrays with no prefix, a 2-byte
**codec version** prefix on top-level `Marshal`, and a `u32` **typeID** prefix for
interface (trait-object) fields.

### 2.1 The Packer (`utils/wrappers/packing.go`)

Big-endian primitive reader/writer with an error-accumulation model (`Errs`). Port
exact constants and semantics:

```rust
pub const BYTE_LEN: usize = 1;
pub const SHORT_LEN: usize = 2;
pub const INT_LEN: usize = 4;
pub const LONG_LEN: usize = 8;
pub const BOOL_LEN: usize = 1;
pub const MAX_STRING_LEN: usize = u16::MAX as usize;   // math.MaxUint16

pub struct Packer<'a> {
    pub bytes: PackerBuf<'a>,   // owned Vec on write, borrowed &[u8] on read
    pub offset: usize,
    pub max_size: usize,
    err: Option<PackerError>,   // first error wins (sticky), mirrors Errs
}
```

Method-for-method parity (write side appends, read side advances `offset`):

| Go | Rust | Wire |
|---|---|---|
| `PackByte`/`UnpackByte` | `pack_byte`/`unpack_byte` | 1 byte |
| `PackShort`/`UnpackShort` | `pack_u16`/`unpack_u16` | 2 bytes BE |
| `PackInt`/`UnpackInt` | `pack_u32`/`unpack_u32` | 4 bytes BE |
| `PackLong`/`UnpackLong` | `pack_u64`/`unpack_u64` | 8 bytes BE |
| `PackBool` | `pack_bool` | 1 byte; **unpack rejects any value ≠ 0/1** (`errBadBool`) |
| `PackFixedBytes` | `pack_fixed_bytes` | raw bytes, no length |
| `PackBytes`/`UnpackBytes` | `pack_bytes`/`unpack_bytes` | `u32` len **prefix** then bytes |
| `UnpackLimitedBytes(limit u32)` | `unpack_limited_bytes` | rejects `len > limit` (`errOversized`) |
| `PackStr`/`UnpackStr` | `pack_str`/`unpack_str` | `u16` len prefix then UTF-8 bytes; pack rejects `len > MAX_STRING_LEN` |
| `UnpackLimitedStr(limit u16)` | `unpack_limited_str` | rejects `len > limit` |

**Sticky error semantics (critical):** once an error is set, every subsequent
operation is a no-op returning a zero value (Go checks `p.Errored()`). The
`max_size`/`expand` growth logic only matters for writes; on overflow set
`ErrInsufficientLength`. On read, `check_space` sets `ErrInsufficientLength` when
`len(bytes) - offset < n`. Reproduce these error identities as a `PackerError`
enum (`InsufficientLength`, `NegativeOffset` — unreachable in Rust but kept for
parity, `InvalidInput`, `BadBool`, `Oversized`).

Negative-offset and `bytes < 0` branches are unreachable with `usize`; document
that and skip them.

### 2.2 Codec / Manager / Registry API

```rust
/// Mirrors `codec.Codec`. One concrete codec = one type registry + tag set.
pub trait Codec {
    fn marshal_into(&self, value: &dyn Serializable, p: &mut Packer) -> Result<()>;
    fn unmarshal_from(&self, p: &mut Packer, dst: &mut dyn Deserializable) -> Result<()>;
    fn size(&self, value: &dyn Serializable) -> Result<usize>;
}

/// Mirrors `codec.Manager`. Holds versioned codecs; prepends the 2-byte version.
pub struct Manager { max_size: usize, codecs: RwLock<HashMap<u16, Arc<dyn Codec>>> }

pub const VERSION_SIZE: usize = SHORT_LEN;          // codec/manager.go:16
pub const DEFAULT_MAX_SIZE: usize = 256 * 1024;     // 256 KiB
pub const INITIAL_SLICE_CAP: usize = 128;

impl Manager {
    pub fn register(&self, version: u16, codec: Arc<dyn Codec>) -> Result<()>; // ErrDuplicatedVersion
    pub fn marshal(&self, version: u16, v: &dyn Serializable) -> Result<Vec<u8>>;
    pub fn unmarshal(&self, src: &[u8], dst: &mut dyn Deserializable) -> Result<u16>;
    pub fn size(&self, version: u16, v: &dyn Serializable) -> Result<usize>;
}
```

**`marshal`:** new `Packer{max_size, bytes: Vec::with_capacity(128)}`, `pack_u16(version)`,
then `codec.marshal_into`. Returns full buffer (version prefix included).
**`unmarshal`:** reject `src.len() > max_size` (`ErrUnmarshalTooBig`); read `u16`
version (`ErrCantUnpackVersion`); dispatch to the registered codec
(`ErrUnknownVersion`); after `unmarshal_from`, **require `offset == src.len()`**
or return `ErrExtraSpace` (Go `manager.go:165`). This trailing-byte check is part
of consensus validation — do not skip it.

Error identities to preserve as a `CodecError` enum (cite Go names):
`UnsupportedType, MaxSliceLenExceeded, DoesNotImplementInterface,
UnexportedField` (→ N/A in Rust; keep variant for parity tests),
`MarshalZeroLength, UnmarshalZeroLength, UnknownVersion, MarshalNil,
UnmarshalNil, UnmarshalTooBig, CantPackVersion, CantUnpackVersion,
DuplicatedVersion, ExtraSpace, DuplicateType, UnknownTypeId`.

### 2.3 Type registry (`linearcodec`)

Go's `linearCodec` assigns sequential `u32` typeIDs in **registration order**
starting at 0; `SkipRegistrations(n)` bumps the counter (reserves IDs). Interface
fields are encoded as `pack_u32(typeID)` followed by the concrete value
(`PrefixSize = INT_LEN = 4`). On unmarshal, the typeID selects the concrete type
and it must implement the target interface (`DoesNotImplementInterface`).

Rust port: we cannot reflect over registered Go types, so we model each Go
interface as a **Rust enum** whose variants are the registered concrete types, in
the exact Go registration order. The derive macro emits the `u32` typeID
dispatch:

```rust
/// One per Go interface that uses RegisterType (e.g. the secp256k1fx `Owners`,
/// avm/platformvm `UnsignedTx`, `verify.Verifiable`). Generated/annotated so the
/// discriminant == the Go typeID.
#[derive(AvaCodec)]
#[codec(type_registry)]              // marks this as an interface dispatch enum
pub enum TransferOutput_etc {
    #[codec(type_id = 0)] Variant0(ConcreteA),
    #[codec(type_id = 1)] Variant1(ConcreteB),
    // #[codec(skip_ids = 3)] to reproduce SkipRegistrations gaps
}
```

> **Binding rule for downstream specs (07/08/09):** every Go `codec.RegisterType`
> call site must be reproduced as an explicit `#[codec(type_id = N)]` with N equal
> to the Go assignment order (accounting for `SkipRegistrations`). These are
> protocol constants. A golden test (§7) asserts each typeID against a Go-dumped
> table.

The Manager+version pattern: each Go VM registers a `linearcodec.New(...)` per
codec version (often v0 only) into a `codec.Manager`. In Rust each version is an
`Arc<dyn Codec>` whose typeID enums are fixed at that version.

### 2.4 `reflectcodec` encoding rules → `ava-codec-derive`

The derive macro `#[derive(AvaCodec)]` replaces Go's `serialize:"true"` struct
tags. Encoding rules transcribed verbatim from `reflectcodec/type_codec.go`:

**Field selection.** Only fields tagged `#[codec]` are serialized, **in
declaration order** (Go iterates struct fields by index, including only tagged
ones — `struct_fielder.go`). Untagged fields are skipped. The macro records the
ordered list of serialized fields.

```rust
#[derive(AvaCodec)]
pub struct TransferableOutput {
    #[codec] pub asset_id: Id,           // tagged → serialized
    #[codec] pub output: Output,         // interface enum → u32 typeID + value
    pub cache: OnceCell<...>,            // untagged → never serialized
}
```

**Per-kind wire format (must match byte-for-byte):**

| Rust type | Wire encoding (Go rule) |
|---|---|
| `u8`/`i8` | 1 byte |
| `u16`/`i16` | 2 bytes BE |
| `u32`/`i32` | 4 bytes BE |
| `u64`/`i64` | 8 bytes BE |
| `bool` | 1 byte (0/1; decode rejects other) |
| `String` | `u16` len + UTF-8 bytes |
| `[u8; N]` (fixed array) | **N raw bytes, no length prefix** |
| `[T; N]`, `T≠u8` | each element back-to-back, **no length prefix** |
| `Vec<u8>` | `u32` len + raw bytes |
| `Vec<T>`, `T≠u8` | `u32` count + each element |
| struct | concatenation of serialized fields in order |
| interface enum | `u32` typeID + concrete value |
| `Box<T>`/`Option`-pointer | see below |

**Slices:**
- count is packed as `u32`; **reject `len > i32::MAX`** (`ErrMaxSliceLenExceeded`;
  Go uses `math.MaxInt32`).
- A nil/empty slice marshals as just the `u32` count = 0 (Go: "nil slices are
  marshaled as empty slices").
- **Zero-length-element guard:** Go errors `ErrMarshalZeroLength` /
  `ErrUnmarshalZeroLength` if an element serializes to 0 bytes (detected by
  `startOffset == offset`). The derive macro must reproduce this check for
  non-`u8` element slices, arrays-of-zero-size, and maps. (In practice this guards
  against e.g. `Vec<()>`.)
- `Vec<u8>` / `[u8;N]` use the fast bulk-copy path (matches Go's `PackFixedBytes`
  / `value.Bytes()` specialization) — same bytes, just faster.

**Maps (`HashMap`/`BTreeMap`) — determinism-critical (`type_codec.go:423`):**
- count packed as `u32` (reject `> i32::MAX`).
- **Entries MUST be emitted sorted by the *serialized key bytes*** via
  `bytes.Compare` (lexicographic, unsigned). Go serializes each key, then sorts
  key byte-ranges, then writes key+value pairs in that order. Port exactly:
  serialize all keys into a scratch buffer, sort by serialized bytes, then write
  `key_bytes ++ marshal(value)` per entry.
- On unmarshal, Go **enforces strictly increasing key bytes** (`bytes.Compare <= 0`
  ⇒ error "keys aren't sorted"). Reproduce: reject equal or out-of-order keys.
  → **Recommendation:** model serialized maps as `BTreeMap<K, V>` where `K`'s
  `Ord` matches serialized-byte order, but still emit via the serialize-then-sort
  path to be safe for multi-byte/interface keys. Document that consensus structs
  rarely use maps (§6.1 of `00`).

**Pointers / Option:** Go dereferences `Ptr` transparently (a `*T` field encodes
exactly as `T`; nil pointer ⇒ `ErrMarshalNil`). In Rust, `Box<T>` encodes as `T`.
There is **no "optional present" byte** — Avalanche structs that are logically
optional use explicit presence flags or interface typeIDs, never Go's pointer
nilability on the wire. The macro therefore treats `Box<T>` as transparent and
**rejects `Option<T>` on a serialized field unless** annotated `#[codec(...)]`
with an explicit user-provided presence scheme (none exist in the protocol; flag
at compile time).

**Recursion guard:** Go tracks a `typeStack` to reject recursive interface types
(`errRecursiveInterfaceTypes`). With enum-dispatch this is structurally
impossible for our types; keep a runtime depth/seen-set only in debug for the
generic path.

**`size()`:** the derive macro also generates an exact `size()` (Go computes it to
pre-allocate). It returns the marshaled length **excluding** the 2-byte version
(the `Manager` adds `VERSION_SIZE`). Used for `Manager::size`.

### 2.5 Derive macro surface (`ava-codec-derive`)

```rust
#[derive(AvaCodec)]                 // generates Serializable + Deserializable + size
struct Foo {
    #[codec]                         // include this field (== serialize:"true")
    a: u32,
    #[codec(version = 0)]            // optional: field present only from codec ver N (rare)
    b: Vec<u8>,
    skipped: Cache,                  // no attribute ⇒ skipped
}

#[derive(AvaCodec)]
#[codec(type_registry)]              // interface-dispatch enum
enum Output {
    #[codec(type_id = 7)] Transfer(TransferOutput),
    #[codec(type_id = 8)] Mint(MintOutput),
}
```

Attributes: `#[codec]` (include field), `#[codec(type_id = N)]` (enum variant
typeID), `#[codec(type_registry)]` (enum is an interface dispatch table),
`#[codec(skip_ids = N)]` / explicit gaps for `SkipRegistrations`,
`#[codec(version = N)]` (field gated on codec version — only if a protocol
actually versions a field; default all-versions).

`Serializable`/`Deserializable` are object-safe traits (`marshal_into`,
`unmarshal_from`, `size`) so the `Manager` can hold `&dyn`. The generated impls
call `Packer` methods directly (no reflection, no allocation beyond the output
buffer).

---

## 3. `ava-crypto`

### 3.1 Hashing (`utils/hashing`)

```rust
pub const HASH_LEN: usize = 32;   // sha256.Size
pub const ADDR_LEN: usize = 20;   // ripemd160.Size
pub fn sha256(b: &[u8]) -> [u8; 32];                 // sha2::Sha256
pub fn ripemd160(b: &[u8]) -> [u8; 20];              // ripemd::Ripemd160
/// `hashing.Checksum(b, n)` = the LAST n bytes of sha256(b). Panics if n > 32.
pub fn checksum(b: &[u8], n: usize) -> Vec<u8> { sha256(b)[32 - n..].to_vec() }
/// `hashing.PubkeyBytesToAddress` = ripemd160(sha256(pubkey)). Address scheme.
pub fn pubkey_bytes_to_address(key: &[u8]) -> [u8; 20] { ripemd160(&sha256(key)) }
```

> Note the Go double-hash for addresses (`ripemd160(sha256(pubkey))`) and for the
> CB58/hex checksum (`last 4 bytes of sha256(payload)`). Both byte-exact.

### 3.2 CB58 and `formatting` encodings

**CB58** (`utils/cb58/cb58.go`): there is no off-the-shelf CB58 crate — hand-roll
it on top of a Bitcoin-alphabet base58. CB58 = `base58( payload ++ checksum(payload, 4) )`.

```rust
const CHECKSUM_LEN: usize = 4;
pub fn cb58_encode(b: &[u8]) -> Result<String> {
    // reject len > i32::MAX - 4 (errEncodingOverFlow)
    let mut checked = b.to_vec();
    checked.extend_from_slice(&checksum(b, CHECKSUM_LEN));
    Ok(base58_encode(&checked))     // mr-tron/base58 == Bitcoin alphabet
}
pub fn cb58_decode(s: &str) -> Result<Vec<u8>> {
    let d = base58_decode(s)?;                              // ErrBase58Decoding
    if d.len() < CHECKSUM_LEN { return Err(MissingChecksum) }
    let (raw, ck) = d.split_at(d.len() - CHECKSUM_LEN);
    if ck != checksum(raw, CHECKSUM_LEN) { return Err(BadChecksum) }
    Ok(raw.to_vec())
}
```

Use the `bs58` crate (Bitcoin alphabet, the same alphabet `mr-tron/base58` uses)
for the raw base58 layer; CB58 = `bs58` + the 4-byte sha256-tail checksum. **Do
not** use `bs58`'s built-in `::with_check` (that is double-sha256 Bitcoin-style;
CB58 is single-sha256-tail). Golden-vector this against Go.

**`formatting.Encode/Decode`** (`encoding.go`): the `Encoding` enum `{Hex, HexNC,
HexC, Json}`. `Hex`/`HexC` = `"0x" + hex(payload ++ checksum4)`; `HexNC` = `"0x" +
hex(payload)`; decode requires the `0x` prefix and (for Hex/HexC) verifies the
4-byte checksum. `Json` is unsupported in this call path (returns an error in Go).
Default API encoding is `Hex`.

### 3.3 Bech32 addresses (`utils/formatting/address`)

Chain-prefixed address format: `"<chainAlias>-<bech32(hrp, addr)>"`, e.g.
`X-avax1...`, `P-fuji1...`. Go uses `btcd/btcutil/bech32` with the
8↔5-bit `ConvertBits(..., pad=true)` regrouping.

```rust
// crate: bech32 = "0.11"
pub fn format_bech32(hrp: &str, payload: &[u8]) -> Result<String>;   // 8→5 bits, pad
pub fn parse_bech32(s: &str) -> Result<(String, Vec<u8>)>;           // returns (hrp, 8-bit bytes)
pub fn format(chain_alias: &str, hrp: &str, addr: &[u8]) -> Result<String>; // "alias-bech32"
pub fn parse(addr: &str) -> Result<(String /*alias*/, String /*hrp*/, Vec<u8>)>; // split on first '-'
```

Use **bech32 (not bech32m)** with the standard checksum; the separator within the
chain-prefixed form is `-` (split into ≤2 parts; `ErrNoSeparator` if none). The
HRP per network comes from `ava-types::constants::get_hrp(network_id)`.

> **bech32 crate nuance:** verify the `bech32 0.11` API produces the same
> `ConvertBits(8,5,pad=true)` grouping as btcd. The `bech32::encode` with
> `Bech32` variant + `Hrp::parse` path matches; cover with golden vectors against
> known mainnet/fuji addresses.

### 3.4 secp256k1 (`utils/crypto/secp256k1`)

Constants & wire layout (`secp256k1.go`):

```rust
pub const SIGNATURE_LEN: usize = 65;    // [r||s||v]
pub const PRIVATE_KEY_LEN: usize = 32;
pub const PUBLIC_KEY_LEN: usize = 33;   // COMPRESSED
const COMPACT_SIG_MAGIC_OFFSET: u8 = 27;
pub const PRIVATE_KEY_PREFIX: &str = "PrivateKey-";
```

Crate: `secp256k1 = "0.31"` (Bitcoin Core libsecp256k1; same C lib lineage as
`dcrd/secp256k1`, so DER/compact/recovery semantics match). Use feature
`recovery`.

**Signature byte order is the load-bearing detail.** Go stores `[r || s || v]`
(64 bytes r/s + 1 byte recovery id in `0..=3`). The decred library’s internal
"compact" form is `[v' || r || s]` with `v' = v + 27`. Conversions
(`rawSigToSig` / `sigToRawSig`) must be reproduced exactly:

```rust
/// Avalanche sig [r||s||v]  →  libsecp recoverable (recid v, compact 64-byte r||s).
fn ava_sig_to_recoverable(sig: &[u8; 65]) -> Result<RecoverableSignature> {
    // verify low-S first (see below); recid = sig[64]; compact = sig[0..64]
}
/// libsecp compact+recid  →  Avalanche [r||s||v] (v in 0..=3, the recovery id).
fn recoverable_to_ava_sig(...) -> [u8; 65];
```

**Sign** (`SignHash`): `ecdsa.SignCompact(sk, hash, compressed=false)` produces RFC6979
deterministic, **low-S–normalized**, recoverable signatures. With the `secp256k1`
crate use `sign_ecdsa_recoverable` (RFC6979 + low-S by default) then reorder to
`[r||s||v]`. The recovery id maps to Go’s `v` (0..=3); Go’s "uncompressed"
(`compressed=false`) means the recid does NOT have the +4 compressed flag.

**Verify / recover** (`RecoverPublicKeyFromHash`): first
`verify_secp256k1_r_signature_format`:

```rust
/// Reject high-S signatures (malleability). `secp256k1.go:316`.
fn verify_sig_format(sig: &[u8; 65]) -> Result<()> {
    // s = scalar(sig[32..64]); if s.is_high() { return Err(MutatedSig) }
}
```

This **low-S enforcement is consensus-critical** — a high-S signature is rejected
(`errMutatedSig`) before recovery. The `secp256k1` crate exposes
`Signature::normalize_s` and you can detect non-normalized via comparing
normalized vs original; implement an explicit `is_high_s` check on the 32-byte S
scalar to match Go bit-for-bit. Recovery rejects compressed-flagged recids
(`errCompressed`). On success, `RecoverPublicKey` returns an **uncompressed** key
internally but `Bytes()` serializes **compressed (33 bytes)**.

**Address derivation:**
```rust
impl PublicKey {
    pub fn bytes(&self) -> [u8; 33];                 // SerializeCompressed
    pub fn address(&self) -> ShortId {               // ripemd160(sha256(compressed_pubkey))
        ShortId::from_slice(&pubkey_bytes_to_address(&self.bytes())).unwrap()
    }
    pub fn eth_address(&self) -> [u8; 20];           // keccak256(uncompressed[1..])[12..] (EVM)
}
```
`VerifyHash` recovers the pubkey from the sig and compares **addresses**
(`recovered.address() == self.address()`), matching Go.

**PrivateKey string form:** `"PrivateKey-" + cb58(32-byte sk)`; JSON quoted.
`NewPrivateKey` = random 32-byte scalar.

### 3.5 BLS12-381 (`utils/crypto/bls`)

Go uses `supranational/blst` with **public keys in G1 (`P1Affine`)** and
**signatures in G2 (`P2Affine`)** → this is the blst **`min_pk`** variant. Use
`blst = "0.3"` and `use blst::min_pk::*`.

```rust
pub const PUBLIC_KEY_LEN: usize = 48;    // BLST_P1_COMPRESS_BYTES (compressed G1)
pub const SIGNATURE_LEN: usize  = 96;    // BLST_P2_COMPRESS_BYTES (compressed G2)
pub const SECRET_KEY_LEN: usize = 32;
```

**Ciphersuites (DST strings — byte-exact, `ciphersuite.go`):**
```rust
pub const CIPHERSUITE_SIGNATURE: &[u8] = b"BLS_SIG_BLS12381G2_XMD:SHA-256_SSWU_RO_POP_";
pub const CIPHERSUITE_POP: &[u8]       = b"BLS_POP_BLS12381G2_XMD:SHA-256_SSWU_RO_POP_";
```

API parity:

| Go | Rust (`ava_crypto::bls`) | Notes |
|---|---|---|
| `localsigner.New` | `SecretKey::new()` | `blst::min_pk::SecretKey::key_gen(ikm32)`; ikm from CSPRNG, zeroized after |
| `localsigner.FromBytes` | `SecretKey::from_bytes(&[u8;32])` | big-endian `Deserialize`; zeroize on drop |
| `Signer.Sign(msg)` | `sk.sign(msg)` | `sig.sign(sk, msg, CIPHERSUITE_SIGNATURE)` (hash-to-G2) |
| `Signer.SignProofOfPossession(msg)` | `sk.sign_pop(msg)` | uses `CIPHERSUITE_POP` |
| `PublicKeyToCompressedBytes` | `pk.compress()` → 48 bytes | network/genesis form |
| `PublicKeyFromCompressedBytes` | `PublicKey::from_compressed` | `uncompress` + **`key_validate()`** (subgroup check); reject zero/invalid |
| `PublicKeyToUncompressedBytes` | `pk.serialize()` → 96 bytes | uncompressed storage form |
| `SignatureToBytes` | `sig.compress()` → 96 bytes | |
| `SignatureFromBytes` | `Signature::from_bytes` | `uncompress` + **`sig_validate(false)`** |
| `AggregatePublicKeys` | `aggregate_public_keys(&[&PublicKey])` | `P1Aggregate::aggregate(pks, false)`; error on empty |
| `AggregateSignatures` | `aggregate_signatures(&[&Signature])` | `P2Aggregate`; error on empty |
| `Verify(pk, sig, msg)` | `verify(pk, sig, msg)` | `sig.verify(false, pk, false, msg, CIPHERSUITE_SIGNATURE)` |
| `VerifyProofOfPossession` | `verify_pop(pk, sig, msg)` | uses `CIPHERSUITE_POP` |

> The `false` flags in blst `verify`/`aggregate` mean "do not re-validate the
> already-validated group element" — Go validates on parse
> (`PublicKeyFromCompressedBytes`/`SignatureFromBytes`) then passes `false` to
> verify. Mirror this: validate on `from_*`, skip re-validation in `verify`.

`Signer` is a trait (`PublicKey()`, `Sign`, `SignProofOfPossession`) with
`LocalSigner` (in-process key) and an RPC signer (`12`/`05`). PoP usage: a
validator publishes `(pk, pop)` where `pop = sign_pop(pk_compressed)`; verified
with `verify_pop(pk, pop, pk.compress())` (the message is the pubkey bytes — see
platformvm `08`).

### 3.6 Staking TLS certs & NodeID derivation (`staking/`)

**NodeID derivation (consensus identity — `ids/node_id.go:79`):**
```rust
/// NodeID = ripemd160(sha256(cert.DER))  — i.e. ShortID over the raw cert bytes.
pub fn node_id_from_cert(cert_der: &[u8]) -> NodeId {
    NodeId::from(pubkey_bytes_to_address(cert_der))   // ripemd160(sha256(DER))
}
```
**Important:** the hash input is the **entire DER-encoded certificate** (`cert.Raw`),
not the public key. `pubkey_bytes_to_address` happens to be the same
`ripemd160(sha256(·))` function. Byte-exact.

**Cert generation (`NewCertAndKeyBytes`):** self-signed X.509 with these exact
template values (so behavior matches; the cert bytes themselves are not consensus
constants but the parsing/derivation must round-trip):
- ECDSA P-256 key (`ecdsa.GenerateKey(P256)`).
- `SerialNumber = 0`, `NotBefore = 2000-01-01T00:00:00Z` (Go literally writes
  `time.January, 0` ⇒ Dec 31 1999 — replicate the exact instant), `NotAfter = now
  + 100 years`, `KeyUsage = DigitalSignature`, `BasicConstraintsValid = true`, no
  SAN/subject. Self-signed.
- PEM blocks: `CERTIFICATE` and `PRIVATE KEY` (PKCS#8).

Use `rcgen` for generation and PEM via `rustls-pemfile`. The key/cert files are
written `0o400` (read-only) after creation (`perms.ReadOnly`); the parent dir
`0o700`.

**Cert parsing (`staking/parse.go`) — strict, custom ASN.1.** Go does NOT use the
full `crypto/x509` parser; it walks the ASN.1 manually (`cryptobyte`) and is
deliberately stricter than TLS:
- `MAX_CERTIFICATE_LEN = 2 KiB` (`2 * units.KiB`); reject larger
  (`ErrCertificateTooLarge`).
- Parse `Certificate ::= SEQUENCE { tbsCertificate, ... }`; skip version, serial,
  sigAlg, issuer, validity, subject; read SubjectPublicKeyInfo; require parsing
  the public key (unlike std x509 which defers).
- **RSA keys:** modulus bit length MUST be **exactly 2048 or 4096**; exponent MUST
  be **65537**; modulus positive and odd. Reject otherwise (the
  `ErrUnsupportedRSAModulusBitLen` / `ErrUnsupportedRSAPublicExponent` family).
- **ECDSA keys:** P-256 only; unmarshal the curve point or
  `ErrFailedUnmarshallingEllipticCurvePoint`.
- Unknown algorithm ⇒ `ErrUnknownPublicKeyAlgorithm`.

Port with `x509-parser` for the SPKI walk **plus** the explicit RSA/ECDSA
well-formedness checks above (the std-lib-lenient path is not acceptable — these
checks gate which peers may join). `Certificate { raw: Vec<u8>, public_key:
PublicKey }`.

**Signature verification (`staking/verify.go`):** `CheckSignature(cert, msg, sig)`
= `sha256(msg)` then RSA `PKCS1v15`/SHA-256 or ECDSA `VerifyASN1` depending on key
type; unsupported ⇒ error. Used in the P2P handshake (`05`).

> **TLS provider parity:** `rustls` must offer the same cipher suites / TLS 1.3
> (and the legacy 1.2 suites Go offers for staking) that avalanchego’s
> `peer`/`network` TLS config uses — pin the `rustls` crypto provider and suite
> list in `05`. This spec only fixes cert *content* and NodeID derivation.

---

## 4. `ava-utils` — sampler (CONSENSUS-CRITICAL), sets, bags, linked map, math

### 4.1 Sampler determinism contract

The validator sampler output is **stored in chain state** (`rand.go:36`: "any
modifications are considered breaking"). The Rust sampler MUST reproduce Go’s
output bit-for-bit given the same RNG source.

**RNG source (`utils/sampler/rand.go`).** Go’s deterministic path uses
`gonum/v1/gonum/mathext/prng.MT19937` — the **64-bit Mersenne Twister**,
seeded with a single `uint64`, yielding `Uint64()`. The non-deterministic global
RNG seeds MT19937 with `time.Now().UnixNano()`; that path is **not**
consensus-relevant (only the `NewDeterministic*` constructors are). Port:

```rust
/// Trait mirroring `sampler.Source`. Only `Uint64` is consensus-relevant.
pub trait Source { fn uint64(&mut self) -> u64; }

/// gonum-compatible MT19937-64. MUST match gonum's seed schedule & output stream.
/// crate: rand_mt::Mt64  (verify against gonum golden vectors before trusting it;
/// if it diverges, hand-port gonum's MT19937 — it is ~120 lines).
pub struct Mt19937_64(rand_mt::Mt64);
```

> **OPEN VERIFICATION ITEM (must pass before merge):** confirm `rand_mt::Mt64`'s
> output stream for a given `u64` seed is identical to gonum
> `prng.MT19937.Seed(seed)` + `Uint64()`. MT19937-64 is standardized
> (Matsumoto–Nishimura 2004 init `6364136223846793005`), so they *should* match,
> but gonum's `Seed(uint64)` init must be matched exactly. Golden vectors in `02`.
> If divergent, hand-port gonum. This is the single highest-risk crate choice in
> this spec.

**`Uint64Inclusive(n)` (`rand.go:39`) — port the three branches verbatim:**
```rust
fn uint64_inclusive(src: &mut impl Source, n: u64) -> u64 {
    if n & n.wrapping_add(1) == 0 {            // n+1 is a power of two
        src.uint64() & n
    } else if n > i64::MAX as u64 {            // n > MaxInt64
        let mut v = src.uint64(); while v > n { v = src.uint64(); } v
    } else {
        // max = (1<<63) - 1 - (1<<63) % (n+1); rejection-sample uint63
        let max = ((1u64 << 63) - 1) - ((1u64 << 63) % (n + 1));
        let mut v = src.uint64() & i64::MAX as u64;
        while v > max { v = src.uint64() & i64::MAX as u64; }
        v % (n + 1)
    }
}
```
`uint63 = uint64 & MaxInt64`. Match the rejection loop exactly (it consumes the
same number of RNG draws as Go → identical downstream state).

**Uniform-without-replacement (`uniform_replacer.go`).** Lazy partial
Fisher–Yates over `[0, length)`:
```rust
// state: length, drawn: HashMap<u64,u64> (defaultMap: get(k, default=k)), draws_count
fn next(&mut self) -> Option<u64> {
    if self.draws_count >= self.length { return None }
    let draw = uint64_inclusive(rng, self.length - 1 - self.draws_count) + self.draws_count;
    let ret = self.drawn.get(draw).copied().unwrap_or(draw);
    self.drawn.insert(draw, self.drawn.get(&self.draws_count).copied().unwrap_or(self.draws_count));
    self.draws_count += 1;
    Some(ret)
}
```
`Sample(count)` resets then calls `next` count times; `false` if range exhausted.
**The `drawn` map semantics (`get(k, default=k)`) and draw formula are exact.**

**Weighted (`weighted_heap.go`).** Build a heap of `{weight, cumulative_weight,
index}` elements. Init: stable-sort by **(weight desc, original index asc)**
(`weightedHeapElement.Compare`), then accumulate `cumulative_weight` from leaves
to root using `parent = (i-1) >> 1` and **checked add** (overflow ⇒ error). Sample
walks: at each node, if `value < weight` return `index`; else subtract weight,
go to left child `2i+1`; if `left.cumulative_weight <= value` subtract it and go
right (`++index`). Port arithmetic and traversal exactly (the sampled index feeds
consensus). `NewWeighted` = heap (Go default).

**Weighted-without-replacement (`weighted_without_replacement_generic.go`).**
Compose: `Initialize` sums weights with **checked add** (overflow ⇒ error),
`uniform.initialize(total)`, `weighted.initialize(weights)`. `Sample(count)`:
reset uniform, then for each of `count`: `w = uniform.next()` (a weight position),
`index = weighted.sample(w)`. Returns indices (duplicates impossible because
uniform is without-replacement over the weight space). This is what the validator
set / consensus poll sampler uses with `NewDeterministicWeightedWithoutReplacement(source)`.

```rust
pub trait Uniform { fn initialize(&mut self, n: u64); fn sample(&mut self, k: usize) -> Option<Vec<u64>>;
                    fn next(&mut self) -> Option<u64>; fn reset(&mut self); }
pub trait Weighted { fn initialize(&mut self, w: &[u64]) -> Result<()>; fn sample(&mut self, v: u64) -> Option<usize>; }
pub trait WeightedWithoutReplacement { fn initialize(&mut self, w: &[u64]) -> Result<()>; fn sample(&mut self, k: usize) -> Option<Vec<usize>>; }
pub fn new_deterministic_weighted_without_replacement(src: Box<dyn Source>) -> impl WeightedWithoutReplacement;
```

### 4.2 set / bag / Bits

- **`Set<T>`** (`utils/set`): a `HashSet`-like with the convenience API Go uses
  (`Of`, `Add`, `Contains`, `Overlaps`, `List`, ordered `SortedList` via explicit
  sort). Where Go serializes set contents, callers sort first (§6.1).
- **`Bits`** (`utils/set/bits.go`): a big-int-backed bitset
  (`Add/Remove/Contains/Union/Intersection/Difference/Len(=popcount)/BitLen`,
  `Bytes()`/`from_bytes` big-endian via `num-bigint::BigUint`,
  `String()`=hex). Used by consensus polls. `Bits64` (`bits_64.go`) is a `u64`
  fast-path variant — port as a separate type.
- **`Bag<T>`** (`utils/bag`): a multiset `HashMap<T,usize>` with a `threshold`
  and a `met_threshold` set tracking elements whose count ≥ threshold (used by
  Snowball vote counting). `Of`, `Add(n)`, `Count`, `Mode`, `Threshold`. Port the
  threshold bookkeeping exactly (consensus poll logic).
- **`UniqueBag`** (`unique_bag.go`): `HashMap<T, set::Bits>` mapping a vote to the
  set of voters by index.

### 4.3 linked hashmap, math, units, misc

- **`LinkedHashmap<K,V>`** (`utils/linked/hashmap.go`): ordered O(1) map preserving
  **insertion order**; `Put` on an existing key **moves it to the back**.
  Implement with `indexmap::IndexMap` plus explicit move-to-back, or a hand-rolled
  intrusive list + `HashMap` (Go uses a free-list-backed doubly linked list).
  Behavior, not the free-list, is what matters.
- **`safemath`** (`utils/math/safe_math.go`): generic checked `add/sub/mul` over
  unsigned ints returning `Result` with `Overflow`/`Underflow` errors; `abs_diff`;
  `max_uint::<T>()`. Map to `checked_add/sub/mul` with these typed errors. This is
  the `safemath` alias from `00` §4.7 / §6.1.
- **`units`** (`utils/units`): `KiB/MiB/GiB`, `NanoAvax/MicroAvax/...Avax` (1 AVAX
  = 1e9 nAVAX). Copy constants.
- **`window`** (sliding time window), **`timer`** (`Clock`, adaptive timeout),
  **`bloom`** (Go's `utils/bloom` — used by gossip; serialized form is a protocol
  detail, port byte-exact when `05` needs it). Stubs here; detailed in their
  consumer specs.

---

## 5. `ava-version` and upgrade schedule (PROTOCOL CONSTANTS)

### 5.1 Version

```rust
pub struct Application { pub name: String, pub major: u32, pub minor: u32, pub patch: u32 }
impl Application {
    pub fn display(&self) -> String { format!("{}/{}.{}.{}", self.name, self.major, self.minor, self.patch) }
    pub fn semantic(&self) -> String { format!("v{}.{}.{}", self.major, self.minor, self.patch) }
    pub fn compare(&self, o: &Self) -> Ordering;   // major, then minor, then patch
}
pub const CLIENT: &str = "avalanchego";              // version/constants.go
pub const RPC_CHAIN_VM_PROTOCOL: u32 = 45;           // bump must match Go
pub const CURRENT_DATABASE: &str = "v1.4.5";
pub const CURRENT: Application = /* avalanchego 1.14.2 */;
pub const MINIMUM_COMPATIBLE: Application = /* 1.14.0 */;
pub const PREV_MINIMUM_COMPATIBLE: Application = /* 1.13.0 */;
```
`Compatibility` (`compatibility.go`): given a peer version and the upgrade time,
decide if a peer is compatible (peer must be ≥ `MinCompatible`, or ≥
`MinCompatibleAfterUpgrade` once `UpgradeTime` passed). Port the comparison logic;
used in the handshake (`05`).

> These version numbers track the Go release; pin to the Go tree at port time and
> guard with a golden test that re-reads `version/constants.go`.

### 5.2 Upgrade schedule (`upgrade/upgrade.go`) — copy verbatim

```rust
pub struct UpgradeConfig {
    pub apricot_phase_1_time: DateTime<Utc>,
    pub apricot_phase_2_time: DateTime<Utc>,
    pub apricot_phase_3_time: DateTime<Utc>,
    pub apricot_phase_4_time: DateTime<Utc>,
    pub apricot_phase_4_min_p_chain_height: u64,
    pub apricot_phase_5_time: DateTime<Utc>,
    pub apricot_phase_pre_6_time: DateTime<Utc>,
    pub apricot_phase_6_time: DateTime<Utc>,
    pub apricot_phase_post_6_time: DateTime<Utc>,
    pub banff_time: DateTime<Utc>,
    pub cortina_time: DateTime<Utc>,
    pub cortina_x_chain_stop_vertex_id: Id,
    pub durango_time: DateTime<Utc>,
    pub etna_time: DateTime<Utc>,
    pub fortuna_time: DateTime<Utc>,
    pub granite_time: DateTime<Utc>,
    pub granite_epoch_duration: Duration,
    pub helicon_time: DateTime<Utc>,
}
```

**Activation rule:** `is_X_activated(t) = !(t < X_time)` i.e. `t >= X_time`
(Go `!t.Before(X)`). Port one `is_*_activated` per phase.

**Mainnet / Fuji activation instants (verbatim — these ARE protocol constants):**

| Upgrade | Mainnet (UTC) | Fuji (UTC) |
|---|---|---|
| ApricotPhase1 | 2021-03-31 14:00 | 2021-03-26 14:00 |
| ApricotPhase2 | 2021-05-10 11:00 | 2021-05-05 14:00 |
| ApricotPhase3 | 2021-08-24 14:00 | 2021-08-16 19:00 |
| ApricotPhase4 | 2021-09-22 21:00 | 2021-09-16 21:00 |
| ApricotPhase4MinPChainHeight | 793005 | 47437 |
| ApricotPhase5 | 2021-12-02 18:00 | 2021-11-24 15:00 |
| ApricotPhasePre6 | 2022-09-05 01:30 | 2022-09-06 20:00 |
| ApricotPhase6 | 2022-09-06 20:00 | 2022-09-06 20:00 |
| ApricotPhasePost6 | 2022-09-07 03:00 | 2022-09-07 06:00 |
| Banff | 2022-10-18 16:00 | 2022-10-03 14:00 |
| Cortina | 2023-04-25 15:00 | 2023-04-06 15:00 |
| CortinaXChainStopVertexID | `jrGWDh5Po9FMj54depyunNixpia5PN4aAYxfmNzU8n752Rjga` | `2D1cmbiG36BqQMRyHt4kFhWarmatA1ighSpND3FeFgz3vFVtCZ` |
| Durango | 2024-03-06 16:00 | 2024-02-13 16:00 |
| Etna | 2024-12-16 17:00 | 2024-11-25 16:00 |
| Fortuna | 2025-04-08 15:00 | 2025-03-13 15:00 |
| Granite | 2025-11-19 16:00 | 2025-10-29 15:00 |
| GraniteEpochDuration | 5m | 5m |
| Helicon | Unscheduled (9999-12-01) | Unscheduled (9999-12-01) |

`Default` (local/custom): all phases = `InitiallyActiveTime` (2020-12-05 05:00
UTC) except `Helicon = Unscheduled`, `GraniteEpochDuration = 30s`,
`CortinaXChainStopVertexID = Id::EMPTY`. `InitiallyActiveTime = 2020-12-05
05:00:00 UTC`; `UnscheduledActivationTime = 9999-12-01 00:00:00 UTC`.

`get_config(network_id)` ⇒ Mainnet / Fuji / Default. `validate()` checks the
upgrade times are monotonically non-decreasing in the listed order (the
`apricot_phase_4_min_p_chain_height` and the two epoch durations are excluded from
the ordering check, matching Go's `upgrades` slice).

> **Time representation.** Store as integer Unix nanoseconds / `chrono` `DateTime<Utc>`;
> activation comparison must be against the consensus block timestamp. Per `00`
> §4.7 the consensus time math is integer-based; `chrono` is only a formatting
> convenience here.

---

## 6. Go → Rust mapping table (non-obvious)

| Go | Rust | Note |
|---|---|---|
| `ID [32]byte` value type | `Id([u8;32])` `Copy` newtype | `Ord` == `bytes.Compare` |
| `NodeID = ShortID` (type alias) | `NodeId([u8;20])` distinct newtype | needs different Display/JSON prefix |
| `wrappers.Packer` + `Errs` sticky error | `Packer` with `Option<PackerError>` | once errored, ops are no-ops |
| reflection `serialize:"true"` | `#[derive(AvaCodec)]` + `#[codec]` | fields in declaration order |
| `linearCodec` typeID registry (bimap) | `#[codec(type_registry)]` enum w/ `type_id = N` | N = Go registration order |
| `SkipRegistrations(n)` | gap in `type_id` values | reserve IDs |
| map marshal: serialize keys, sort by bytes | serialize-then-sort identical | + strict increasing on decode |
| `gonum prng.MT19937` (64-bit) | `rand_mt::Mt64` (verify!) | consensus RNG; golden-vector gated |
| `Uint64Inclusive` 3-branch | port exactly incl. rejection loops | same RNG draw count |
| decred secp256k1 `[v\|\|r\|\|s]` compact | `secp256k1` crate recoverable | reorder to/from `[r\|\|s\|\|v]`, low-S enforce |
| blst `P1Affine`/`P2Affine` | `blst::min_pk::{PublicKey,Signature}` | pk∈G1 (48B), sig∈G2 (96B) |
| `NodeIDFromCert` = ripemd160(sha256(DER)) | `node_id_from_cert(der)` | hashes the whole cert |
| `cb58` (single-sha256 tail checksum) | hand-rolled `bs58` + `checksum4` | NOT bs58 `with_check` |
| `safemath.Add/Sub/Mul` | `checked_*` + typed error | `safemath` alias |
| `time.Time` upgrade consts | `DateTime<Utc>` / unix-nanos consts | activation = `t >= X` |

## 7. Error model

Each crate exposes `pub type Result<T> = core::result::Result<T, Error>` with a
`thiserror` enum. Preserve Go sentinel-error identities as variants (asserted in
tests where Go uses `errors.Is`, per `00` §7.1):

- `ava_codec::CodecError` / `PackerError` — see §2.2 list.
- `ava_crypto::Error` — `InvalidSig, MutatedSig (high-S), InvalidSigLen,
  InvalidPublicKeyLength, InvalidPrivateKeyLength, Compressed, BadChecksum,
  MissingChecksum, Base58Decoding, MissingHexPrefix, CertificateTooLarge,
  Malformed* (cert ASN.1 family), UnsupportedRSAModulusBitLen,
  UnsupportedRSAPublicExponent, UnknownPublicKeyAlgorithm, FailedPublicKeyDecompress,
  FailedSignatureDecompress, NoPublicKeys, NoSignatures, ...`.
- `ava_types::Error` — `InvalidHashLen, NoIdWithAlias, AliasAlreadyMapped,
  ShortNodeId, MissingQuotes`.
- `ava_utils::Error` — `Overflow, Underflow`.
- `ava_version::Error` — `InvalidUpgradeTimes`.

## 8. Test plan (see `02-testing-strategy.md`)

**Byte-exact golden vectors (highest priority — differential vs. Go):**
1. **Codec round-trips & golden bytes.** Dump `Manager.Marshal` output (hex) from
   Go for a representative struct from every protocol struct family (fixed array,
   `Vec<u8>`, `Vec<struct>`, interface/typeID, map, nested). Assert Rust
   `marshal` produces identical bytes and `unmarshal` round-trips. Include
   negative cases: trailing bytes (`ExtraSpace`), oversize slice
   (`MaxSliceLenExceeded`), bad bool, unsorted map keys, unknown typeID, version
   mismatch.
2. **Packer.** Property test (`proptest`) every primitive: pack→unpack identity;
   golden BE bytes for fixed inputs; sticky-error behavior.
3. **typeID tables.** Golden table of every registered type’s `(interface,
   type_id)` dumped from Go; assert the derive-macro discriminants match.
4. **Sampler determinism (consensus).** Seed the deterministic source with fixed
   `u64` seeds; compare full `Sample` outputs (and intermediate `Uint64Inclusive`
   sequences) against Go-dumped vectors for uniform, weighted, and
   weighted-without-replacement. **Gate the `rand_mt::Mt64` choice on a
   golden-vector match with gonum MT19937** before any other sampler test.
5. **CB58 / bech32 / hex.** Golden vectors: known `Id`/`NodeId`/address strings
   from Mainnet & Fuji; round-trip; checksum-failure cases.
6. **secp256k1.** RFC6979 deterministic-signature vectors (reuse Go's
   `rfc6979_test.go`); low-S rejection of a hand-mutated high-S sig; recover →
   address equals expected; `PrivateKey-` string round-trip.
7. **BLS.** Sign/verify, aggregate verify, PoP verify; compress/uncompress
   round-trip; cross-verify a Go-produced signature/pubkey (and vice-versa);
   ciphersuite DST byte-equality.
8. **NodeID-from-cert.** Feed Go-generated staking certs; assert identical
   `NodeId`. Assert the strict ASN.1 parser rejects RSA-3072 / exponent≠65537 /
   oversize certs exactly as Go does.
9. **Upgrade schedule.** Golden table of all Mainnet/Fuji activation instants;
   `is_*_activated` boundary tests (exactly-at-time = activated).

**Property tests (`proptest`):** round-trip for all `Id`/`ShortId`/`NodeId`
string & JSON forms; codec marshal∘unmarshal == identity for arbitrary instances
of derived types; safemath checked ops vs. `u128` reference; `Bits` set algebra.

## 9. Performance notes / improvements over Go

- **Const-generic fixed IDs.** `[u8; 32]`/`[u8; 20]` newtypes are stack `Copy`
  with no heap; comparison via `Ord` on arrays. Avoids Go’s per-`String()` CB58
  allocation on hot consensus paths (cache the string only where logged).
- **Zero-copy / single-allocation Packer.** Write path allocates one `Vec`
  (`with_capacity(size())` from the generated `size()`); read path borrows
  `&[u8]` (`bytes::Bytes` on the network path) — no per-field allocation. Matches
  Go’s `PackFixedBytes` fast path and improves on its `reflect` overhead.
- **No reflection.** The derive macro emits straight-line `Packer` calls; the Go
  codec pays reflection + per-type field-index cache lookups on every
  marshal/unmarshal. This is pure win with **identical output bytes** (safe per
  §6.1: encoding rules are fixed by the macro, not data-dependent).
- **BLS batch verify.** Where Go calls `Verify` per signature, `blst` supports
  batched/multi-point verification; use it for independent signature sets
  (e.g. Warp/ICM aggregate checks) — **safe only when verification has no
  ordering side effects** (it does not). Back with a differential test (`00` §9).
- **secp256k1 recovery cache.** Port `RecoverCache` (keyed by
  `sha256(hash ++ sig)`) as a `lru`/`moka` cache; same semantics, lock-free reads.
- **Sampler:** the deterministic sampler is single-threaded by contract (RNG state
  is sequential) — do not parallelize. The only safe optimization is avoiding the
  `HashMap` allocation in `uniformReplacer` for small `count` via a `SmallVec`
  scratch, preserving the exact draw sequence.
- **Validator-set sampling reads** can use `arc-swap`/sharded snapshots
  (`ava-validators`, `06`) but the sample *computation* stays deterministic.

---

## 10. Deterministic RNG (R1 resolution)

> Referenced from §4.1 (sampler) and `00` §11.2 (R1). This is the **single
> highest-risk** item in the whole port: the validator/consensus sampler output
> is **stored in chain state** (`utils/sampler/rand.go:36–38`: "any modifications
> are considered breaking"), and the proposervm windower's proposer schedule is
> consensus-affecting. The RNG must reproduce Go's output stream **bit-for-bit**.

### 10.1 What Go actually uses (confirmed from source)

avalanchego does **not** use Go's `math/rand`. Both consensus consumers pull
`gonum.org/v1/gonum/mathext/prng`, which is a direct port of the canonical
Matsumoto–Nishimura reference C code:

| Consumer | Generator | Go cite |
|---|---|---|
| Deterministic samplers (`NewDeterministic*`) | **MT19937-64** (`prng.MT19937_64`) | `utils/sampler/rand.go:19` (global uses `NewMT19937()`; the sampler `Source` interface only needs `Uint64()`) |
| proposervm `Proposers` (Pre-Durango, legacy) | **MT19937 (32-bit)**, `Uint64()` = two `Uint32()` calls `h<<32 \| l` | `vms/proposervm/proposer/windower.go:122` (comment: "The 32-bit prng is used here for legacy reasons") |
| proposervm `ExpectedProposer`/`MinDelayForProposer` (Post-Durango) | **MT19937-64** (`prng.NewMT19937_64()`) | `windower.go:177`, `windower.go:202` |

**Both** 32-bit and 64-bit variants must be ported — the 32-bit one is *not*
dead code; it gates the Pre-Durango proposer list (`Proposers`/`Delay`), and its
`Uint64()` derives a 64-bit value from **two** 32-bit draws, so its state
advancement and bit layout differ from MT19937-64. Do not substitute one for the
other.

> **Verdict on existing crates:** **do NOT rely on `rand_mt`/`mersenne_twister`
> defaults.** `rand_mt::Mt` and `rand_mt::Mt64` implement the *same algorithm
> family*, but (a) their default seed differs (`rand_mt` uses `5489` like gonum,
> but this is unverified per-release), (b) `rand_mt::Mt64::next_u64` and gonum's
> `Uint64` must be confirmed to apply identical tempering and identical
> `Seed(uint64)` state-init, and (c) the 32-bit `Uint64 = h<<32|l` composition is
> an avalanchego/gonum convention not guaranteed by any crate's `next_u64`.
> **Mandate: hand-port both gonum generators** (each ≈120 lines, transcribed
> below) and golden-gate them against Go before any consensus code merges. A crate
> may be used *only* if it passes the §10.4 golden vectors byte-for-byte; absent
> that proof, the hand-port is normative.

### 10.2 Algorithm — exact constants & schedule (transcribed from gonum)

**MT19937-64** (`mt19937_64.go`). State = 312 `u64` words + index `mti`.

- `NN = 312`, `MM = 156`, `MATRIX_A = 0xB5026F5AA96619E9`,
  `UPPER_MASK = 0xFFFFFFFF80000000`, `LOWER_MASK = 0x7FFFFFFF`.
- **Seed schedule** (`Seed(seed: u64)`): `mt[0] = seed`; for `i` in `1..312`:
  `mt[i] = 6364136223846793005 * (mt[i-1] ^ (mt[i-1] >> 62)) + i` (wrapping mul +
  wrapping add). Sets `mti = 312`.
- **Default seed** = `5489` (applied lazily on first `Uint64` if `Seed` never
  called; `mti` sentinel `NN+1 = 313`). avalanchego always calls `Seed` first, so
  this path is for parity only.
- **Refill** (when `mti >= 312`): the standard twist using `mag01 = [0, MATRIX_A]`
  and `x = (mt[i] & UPPER_MASK) | (mt[i+1] & LOWER_MASK)` split into the two
  ranges `[0, NN-MM)` and `[NN-MM, NN-1)` plus the wrap word; then `mti = 0`.
- **Tempering** (applied to `x = mt[mti]; mti += 1`):
  ```text
  x ^= (x >> 29) & 0x5555555555555555
  x ^= (x << 17) & 0x71D67FFFEDA60000
  x ^= (x << 37) & 0xFFF7EEE000000000
  x ^= (x >> 43)
  ```

**MT19937 (32-bit)** (`mt19937.go`). State = 624 `u32` words + index `mti`.

- `N = 624`, `M = 397`, `MATRIX_A = 0x9908b0df`, `UPPER_MASK = 0x80000000`,
  `LOWER_MASK = 0x7fffffff`.
- **Seed schedule**: `mt[0] = seed as u32` (**only the low 32 bits of the seed
  are used** — `windower.go` seeds with `chainSource ^ blockHeight`, a `u64`, so
  the high half is silently discarded; replicate the truncation exactly); for `i`
  in `1..624`: `mt[i] = 1812433253 * (mt[i-1] ^ (mt[i-1] >> 30)) + i` (wrapping).
- **Tempering**: `y ^= y>>11; y ^= (y<<7) & 0x9d2c5680; y ^= (y<<15) & 0xefc60000;
  y ^= y>>18`.
- **`Uint64()` = `((Uint32() as u64) << 32) | (Uint32() as u64)`** — high word
  drawn first. (Two state advances per `Uint64`.)

### 10.3 Rust design (`ava-utils::rng`)

```rust
/// Mirrors `sampler.Source` (utils/sampler/rand.go:29). Only `uint64` is on the
/// consensus path. Implemented by both MT variants below.
pub trait Source {
    /// Random value in [0, u64::MAX]; advances generator state. == Go `Uint64()`.
    fn uint64(&mut self) -> u64;
}

/// gonum `prng.MT19937_64` — hand-port, byte-for-byte. Used by all deterministic
/// samplers and the Post-Durango windower.
pub struct Mt19937_64 {
    mt: [u64; 312],
    mti: usize, // sentinel 313 == "Seed never called"
}

impl Mt19937_64 {
    const NN: usize = 312;
    const MM: usize = 156;
    const MATRIX_A: u64 = 0xB502_6F5A_A966_19E9;
    const UPPER: u64 = 0xFFFF_FFFF_8000_0000;
    const LOWER: u64 = 0x0000_0000_7FFF_FFFF;

    /// == NewMT19937_64(): unseeded sentinel (lazy default-seed 5489 on first draw).
    pub fn new() -> Self { Self { mt: [0; 312], mti: Self::NN + 1 } }

    /// == (*MT19937_64).Seed. Wrapping arithmetic is REQUIRED (Go integer overflow).
    pub fn seed(&mut self, seed: u64) {
        self.mt[0] = seed;
        let mut i = 1;
        while i < Self::NN {
            let prev = self.mt[i - 1];
            self.mt[i] = 6364136223846793005u64
                .wrapping_mul(prev ^ (prev >> 62))
                .wrapping_add(i as u64);
            i += 1;
        }
        self.mti = Self::NN;
    }

    fn refill(&mut self) {
        const MAG01: [u64; 2] = [0, Mt19937_64::MATRIX_A];
        if self.mti == Self::NN + 1 { self.seed(5489); } // lazy default seed
        let mut x;
        let mut i = 0;
        while i < Self::NN - Self::MM {
            x = (self.mt[i] & Self::UPPER) | (self.mt[i + 1] & Self::LOWER);
            self.mt[i] = self.mt[i + Self::MM] ^ (x >> 1) ^ MAG01[(x & 1) as usize];
            i += 1;
        }
        while i < Self::NN - 1 {
            x = (self.mt[i] & Self::UPPER) | (self.mt[i + 1] & Self::LOWER);
            self.mt[i] = self.mt[i + Self::MM - Self::NN] ^ (x >> 1) ^ MAG01[(x & 1) as usize];
            i += 1;
        }
        x = (self.mt[Self::NN - 1] & Self::UPPER) | (self.mt[0] & Self::LOWER);
        self.mt[Self::NN - 1] = self.mt[Self::MM - 1] ^ (x >> 1) ^ MAG01[(x & 1) as usize];
        self.mti = 0;
    }
}

impl Source for Mt19937_64 {
    fn uint64(&mut self) -> u64 {
        if self.mti >= Self::NN { self.refill(); }
        let mut x = self.mt[self.mti];
        self.mti += 1;
        // Tempering — exact gonum constants.
        x ^= (x >> 29) & 0x5555_5555_5555_5555;
        x ^= (x << 17) & 0x71D6_7FFF_EDA6_0000;
        x ^= (x << 37) & 0xFFF7_EEE0_0000_0000;
        x ^= x >> 43;
        x
    }
}

/// gonum `prng.MT19937` (32-bit). Used ONLY by the Pre-Durango `Proposers` path.
/// Its `Source::uint64` draws TWO u32s (high first); seed truncates to low 32 bits.
pub struct Mt19937 {
    mt: [u32; 624],
    mti: usize, // sentinel 625
}

impl Mt19937 {
    const N: usize = 624;
    const M: usize = 397;
    const MATRIX_A: u32 = 0x9908_b0df;
    const UPPER: u32 = 0x8000_0000;
    const LOWER: u32 = 0x7fff_ffff;

    pub fn new() -> Self { Self { mt: [0; 624], mti: Self::N + 1 } }

    /// == (*MT19937).Seed — NOTE the `as u32` truncation of the u64 seed.
    pub fn seed(&mut self, seed: u64) {
        self.mt[0] = seed as u32;
        let mut i = 1;
        while i < Self::N {
            let prev = self.mt[i - 1];
            self.mt[i] = 1812433253u32
                .wrapping_mul(prev ^ (prev >> 30))
                .wrapping_add(i as u32);
            i += 1;
        }
        self.mti = Self::N;
    }

    fn uint32(&mut self) -> u32 {
        const MAG01: [u32; 2] = [0, Mt19937::MATRIX_A];
        if self.mti >= Self::N {
            if self.mti == Self::N + 1 { self.seed(5489); }
            let mut y;
            let mut kk = 0;
            while kk < Self::N - Self::M {
                y = (self.mt[kk] & Self::UPPER) | (self.mt[kk + 1] & Self::LOWER);
                self.mt[kk] = self.mt[kk + Self::M] ^ (y >> 1) ^ MAG01[(y & 1) as usize];
                kk += 1;
            }
            while kk < Self::N - 1 {
                y = (self.mt[kk] & Self::UPPER) | (self.mt[kk + 1] & Self::LOWER);
                self.mt[kk] = self.mt[kk + Self::M - Self::N] ^ (y >> 1) ^ MAG01[(y & 1) as usize];
                kk += 1;
            }
            y = (self.mt[Self::N - 1] & Self::UPPER) | (self.mt[0] & Self::LOWER);
            self.mt[Self::N - 1] = self.mt[Self::M - 1] ^ (y >> 1) ^ MAG01[(y & 1) as usize];
            self.mti = 0;
        }
        let mut y = self.mt[self.mti];
        self.mti += 1;
        y ^= y >> 11;
        y ^= (y << 7) & 0x9d2c_5680;
        y ^= (y << 15) & 0xefc6_0000;
        y ^= y >> 18;
        y
    }
}

impl Source for Mt19937 {
    /// == (*MT19937).Uint64: high word drawn first, low word second.
    fn uint64(&mut self) -> u64 {
        let h = self.uint32() as u64;
        let l = self.uint32() as u64;
        (h << 32) | l
    }
}
```

The `Uint64Inclusive` rejection-sampling wrapper (the `rng` struct in
`utils/sampler/rand.go`) sits **on top** of any `Source` and is already specified
verbatim in §4.1 (`uint64_inclusive`). Reproduce its three branches exactly so
each variant consumes the **same number of draws** as Go — drift in draw count,
not just bit layout, desyncs all downstream state. (The Go `rng` also holds a
`sync.Mutex`; in Rust the deterministic sampler is `&mut`-single-threaded by
contract — see §9 — so no lock is needed.)

> **`MarshalBinary` parity (optional):** gonum can serialize the full state
> (312/624 BE words + `mti`). Not used on any consensus path; port only if a
> snapshot/debug consumer needs it.

### 10.4 Golden-vector plan (gating — cross-ref `02`)

Store vectors under `tests/vectors/rng/` and `tests/vectors/sampler/`. A small Go
helper (built in the avalanchego tree, output committed) dumps:

1. **Raw generator stream** — for each variant and a fixed seed set
   `{0, 1, 5489, 0xDEADBEEF, u64::MAX, time-like 1_700_000_000_000_000_000}`:
   `(seed, [first 320 Uint64 outputs])`. 320 > NN forces at least one refill so
   the twist is exercised. Separate file per variant
   (`mt19937_64.json`, `mt19937_32.json`); the 32-bit file must show the
   high-word-first `Uint64` composition.
2. **`Uint64Inclusive`** — `(seed, n, [k outputs])` covering each of the three
   branches: `n+1` power-of-two (e.g. `n=255`), `n > MaxInt64`, and the default
   rejection branch (e.g. `n=10`), to verify draw-count parity through rejections.
3. **Samplers** — `(seed, weights, count) -> sampled_indices` for
   `NewDeterministicUniform`, `NewDeterministicWeighted`, and
   `NewDeterministicWeightedWithoutReplacement` (the validator sampler). Use the
   MT19937-64 source.
4. **Windower** — `(chainID, validatorSet{nodeID,weight}, pChainHeight,
   blockHeight, slot) -> ExpectedProposer` and
   `(..., maxWindows) -> Proposers[]` (exercising BOTH the 64-bit and the 32-bit
   path), plus the seed derivations `chainSource ^ blockHeight` (32-bit) and
   `chainSource ^ blockHeight ^ bits.Reverse64(slot)` (64-bit). `chainSource` is
   the first 8 bytes of the chainID read big-endian (`UnpackLong`).

The Rust differential test (`02`) **must** match all four byte-for-byte. CI gate:
**no consensus crate (`06`, `08`, proposervm in `07`) may merge until the RNG and
sampler vectors pass.** If a candidate crate (`rand_mt`) is proposed, it is
admissible only after passing vector set (1) for both variants; otherwise the
hand-port in §10.3 is the implementation.

---

## 11. Network-upgrade fork gating

> Referenced from §5.2. The activation schedule is a set of **protocol
> constants**; every VM (`08` P-Chain, `09` X-Chain, `10` C-Chain, `11` SAE, and
> proposervm in `07`) gates behavior on it identically. This section specifies the
> single shared model in `ava-version` so the gating logic is written **once**.

### 11.1 What Go does (confirmed from `upgrade/upgrade.go`)

There are **16 fork phases**, in this strict chronological/registration order
(the order `Config.Validate` enforces as monotonically non-decreasing):

1. ApricotPhase1
2. ApricotPhase2
3. ApricotPhase3
4. ApricotPhase4 (+ side param `ApricotPhase4MinPChainHeight`, **not a time**)
5. ApricotPhase5
6. ApricotPhasePre6
7. ApricotPhase6
8. ApricotPhasePost6
9. Banff
10. Cortina (+ side param `CortinaXChainStopVertexID`, an `ids.ID`, **not a time**)
11. Durango
12. Etna
13. Fortuna
14. Granite (+ side param `GraniteEpochDuration`, a `time.Duration`, **not a time**)
15. Helicon (currently `UnscheduledActivationTime` on all networks)

…where `ApricotPhase4MinPChainHeight`, `CortinaXChainStopVertexID`,
`GraniteEpochDuration`, and `HeliconTime` are carried in the same `Config` struct
but **excluded from the ordering check** in `Validate` (it lists exactly the 15
time fields above; the three non-time side params are skipped). The full struct is
already reproduced in §5.2's `UpgradeConfig`.

**Activation rule (uniform, time-based):** a fork is active at consensus
timestamp `t` **iff `t >= ForkTime`**. Go implements this as
`func (c *Config) IsXActivated(t time.Time) bool { return !t.Before(c.XTime) }`
(`upgrade.go:144–202`) — one method per phase, all identical in shape. "Exactly
at the boundary instant = activated."

**Height-based exception (the only one):** `ApricotPhase4MinPChainHeight` is a
**block-height** gate, not a timestamp gate. It is consumed *only* in proposervm
pre-fork blocks — `vms/proposervm/pre_fork_block.go:120`
(`if childPChainHeight < b.vm.Upgrades.ApricotPhase4MinPChainHeight`) and
`:170` (`selectChildPChainHeight(...)`). It is **not** a `Is*Activated` predicate;
model it as a plain `u64` field, never as part of `is_active`. All other forks are
purely time-gated. (`CortinaXChainStopVertexID` and `GraniteEpochDuration` are
not activation gates at all — they are post-activation behavioral parameters.)

`GetConfig(networkID)` returns `Mainnet` / `Fuji` / `Default` (the
local/custom genesis-time-0 config where every phase = `InitiallyActiveTime`
except Helicon = unscheduled). Verbatim timestamps are in §5.2's table.

### 11.2 Rust design (`ava-version::upgrade`)

The `UpgradeConfig` struct + per-phase `is_*_activated` already appear in §5.2.
This section adds the **ordered enum + generic API** so VMs gate uniformly instead
of each calling a different bespoke predicate.

```rust
/// Ordered list of network upgrades. Discriminant order == chronological order ==
/// the order `upgrade.Config.Validate` enforces. Only the 15 TIME-gated phases
/// are forks; the height/id/duration side-params live on `UpgradeConfig` (§5.2),
/// NOT here. Mirrors upgrade/upgrade.go.
#[derive(Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash, Debug)]
pub enum Fork {
    ApricotPhase1,
    ApricotPhase2,
    ApricotPhase3,
    ApricotPhase4,
    ApricotPhase5,
    ApricotPhasePre6,
    ApricotPhase6,
    ApricotPhasePost6,
    Banff,
    Cortina,
    Durango,
    Etna,
    Fortuna,
    Granite,
    Helicon,
}

impl Fork {
    /// All forks in activation order (== `Ord`). Used by `fork_at` and `Validate`.
    pub const ALL: [Fork; 15] = [
        Fork::ApricotPhase1, Fork::ApricotPhase2, Fork::ApricotPhase3,
        Fork::ApricotPhase4, Fork::ApricotPhase5, Fork::ApricotPhasePre6,
        Fork::ApricotPhase6, Fork::ApricotPhasePost6, Fork::Banff,
        Fork::Cortina, Fork::Durango, Fork::Etna, Fork::Fortuna,
        Fork::Granite, Fork::Helicon,
    ];
}

impl UpgradeConfig {
    /// Activation time for a given fork (the matching `*_time` field).
    pub fn fork_time(&self, fork: Fork) -> DateTime<Utc> {
        match fork {
            Fork::ApricotPhase1 => self.apricot_phase_1_time,
            Fork::ApricotPhase2 => self.apricot_phase_2_time,
            // … one arm per fork, mapping to the §5.2 fields …
            Fork::Granite       => self.granite_time,
            Fork::Helicon       => self.helicon_time,
            _ => unreachable!("covered above"),
        }
    }

    /// THE canonical gate. `t >= fork_time` (== Go `!t.Before(forkTime)`).
    /// Every VM calls THIS — not a bespoke per-phase predicate.
    /// `t` is the consensus block timestamp.
    pub fn is_active(&self, fork: Fork, t: DateTime<Utc>) -> bool {
        t >= self.fork_time(fork)
    }

    /// The most-recent fork active at `t` (highest `Fork` whose time <= t), or
    /// `None` if `t` precedes ApricotPhase1. Forks form a total order by time
    /// (guaranteed by `validate`), so a linear/last-match scan over `Fork::ALL`
    /// is exact.
    pub fn fork_at(&self, t: DateTime<Utc>) -> Option<Fork> {
        Fork::ALL.iter().rev().copied().find(|&f| self.is_active(f, t))
    }

    /// == upgrade.Config.Validate: the 15 TIME fields must be monotonically
    /// non-decreasing in `Fork::ALL` order. The u64/id/duration side-params are
    /// NOT checked. Errors `InvalidUpgradeTimes`.
    pub fn validate(&self) -> Result<()> {
        for w in Fork::ALL.windows(2) {
            if self.fork_time(w[0]) > self.fork_time(w[1]) {
                return Err(Error::InvalidUpgradeTimes);
            }
        }
        Ok(())
    }
}

/// == upgrade.GetConfig(networkID). MAINNET_ID / FUJI_ID / else Default.
pub fn get_config(network_id: u32) -> UpgradeConfig { /* §5.2 constants */ }
```

The per-phase `is_durango_activated(t)` etc. helpers from §5.2 are kept as thin
forwarders (`self.is_active(Fork::Durango, t)`) for call-site readability and to
mirror Go's method names one-to-one, but `is_active`/`fork_at` are the primitives.

**Constants are the §5.2 table — verbatim.** Reproduced here with Go cites for the
two networks that ship real timestamps (Default = all `InitiallyActiveTime =
2020-12-05 05:00:00 UTC`, Helicon = `UnscheduledActivationTime =
9999-12-01 00:00:00 UTC`):

| Fork | Mainnet UTC (`upgrade.go:20–41`) | Fuji UTC (`upgrade.go:44–65`) |
|---|---|---|
| ApricotPhase1 | 2021-03-31 14:00 | 2021-03-26 14:00 |
| ApricotPhase2 | 2021-05-10 11:00 | 2021-05-05 14:00 |
| ApricotPhase3 | 2021-08-24 14:00 | 2021-08-16 19:00 |
| ApricotPhase4 | 2021-09-22 21:00 | 2021-09-16 21:00 |
| ApricotPhase5 | 2021-12-02 18:00 | 2021-11-24 15:00 |
| ApricotPhasePre6 | 2022-09-05 01:30 | 2022-09-06 20:00 |
| ApricotPhase6 | 2022-09-06 20:00 | 2022-09-06 20:00 |
| ApricotPhasePost6 | 2022-09-07 03:00 | 2022-09-07 06:00 |
| Banff | 2022-10-18 16:00 | 2022-10-03 14:00 |
| Cortina | 2023-04-25 15:00 | 2023-04-06 15:00 |
| Durango | 2024-03-06 16:00 | 2024-02-13 16:00 |
| Etna | 2024-12-16 17:00 | 2024-11-25 16:00 |
| Fortuna | 2025-04-08 15:00 | 2025-03-13 15:00 |
| Granite | 2025-11-19 16:00 | 2025-10-29 15:00 |
| Helicon | 9999-12-01 00:00 | 9999-12-01 00:00 |

Non-time side-params (carried, not gated): `ApricotPhase4MinPChainHeight` =
`793005` (Mainnet) / `47437` (Fuji) / `0` (Default);
`GraniteEpochDuration` = `5m` (Mainnet/Fuji) / `30s` (Default);
`CortinaXChainStopVertexID` = the §5.2 mainnet/fuji IDs / `Id::EMPTY` (Default).

### 11.3 Golden test (cross-ref `02`)

A Go helper dumps, for `networkID ∈ {Mainnet, Fuji}` and **each** fork, the
activation decision at the boundary instants `{forkTime - 1ns, forkTime,
forkTime + 1ns}` (i.e. `is_X_activated`), plus `fork_at(t)` for those instants.
The Rust test asserts identical booleans/fork for every row — pinning the
`t >= forkTime` (inclusive-at-boundary) semantics and the verbatim constants. A
second table asserts `validate()` accepts each shipped config and rejects a
hand-swapped (out-of-order) one, matching `upgrade_test.go`.
