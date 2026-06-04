# 20 — Avalanche Warp Messaging / Interchain Messaging (ICM)

> Conforms to `00-overview-and-conventions.md`. Warp/ICM is the cross-chain
> BLS-multisignature primitive: a chain produces an `UnsignedMessage`, the source
> subnet's validators each BLS-sign it, an aggregator collects signatures into a
> `BitSetSignature`, and a verifying chain checks the aggregate against the source
> subnet's validator set at a pinned P-Chain height. It is consensus-critical and
> **byte-exact** with the Go node: the message format is the **avalanche linear
> codec** (NOT protobuf), the canonical validator ordering is fixed, and the quorum
> rule must match bit-for-bit. This spec gathers machinery that is diffuse across
> ~10 Go files into one `ava-warp` crate reused by P-Chain (`08`), the EVM warp
> precompile (`10`), and SAE (`11`).

## 0. Go source coverage

| Go source | What it provides | Rust home |
|---|---|---|
| `vms/platformvm/warp/{unsigned_message,message,signature,validator,signer,codec,constants}.go` | `UnsignedMessage`, `Message`, `Signature`/`BitSetSignature`, canonical-set helpers, local `Signer`, codec, `AnycastID` | `ava-warp` |
| `vms/platformvm/warp/payload/{payload,addressed_call,hash,codec}.go` | payload interface + `AddressedCall`, `Hash` (own codec) | `ava-warp::payload` |
| `vms/platformvm/warp/message/{payload,addressed_call?,subnet_to_l1_conversion,register_l1_validator,l1_validator_registration,l1_validator_weight,codec}.go` | ACP-77 registry payloads (own codec) | `ava-warp::registry` |
| `vms/platformvm/warp/gwarp/{client,server}.go`, `proto/warp/message.proto` | the `warp.Signer` gRPC service (plugin↔host); **not** the wire format | `ava-warp-rpc` (`07`/`15`) |
| `network/p2p/acp118/{aggregator,handler}.go`, `proto/sdk/sdk.proto` (`SignatureRequest`/`Response`) | the ACP-118 signature-request/response app protocol + aggregator | `ava-warp::acp118` (over `ava-network` p2p SDK, `05`) |
| `snow/validators/warp.go` (`WarpSet`, `Warp`, `FlattenValidatorSet`), `snow/validators/state.go` (`GetWarpValidatorSets`) | canonical validator set & ordering | `ava-validators` (`06`/`08`) — `ava-warp` consumes |
| `graft/{coreth,subnet-evm}/precompile/contracts/warp/{contract,config,module,contract_warp_handler}.go` | the C-Chain/EVM warp precompile, predicate verification, gas | `ava-evm` (`10`) — calls `ava-warp` |

> **Naming caveat (repeated from `15` §3.8):** `proto/warp/message.proto` defines a
> gRPC **`warp.Signer`** service, **NOT** the warp message wire format. The warp
> message/payload/signature bytes are linear-codec artifacts (`03` §2). Anything
> "on the wire between chains" is linear codec; the only protobuf in Warp is (a) the
> `warp.Signer` plugin RPC and (b) the ACP-118 `SignatureRequest`/`Response`
> envelopes that wrap the linear-codec `UnsignedMessage` bytes.

---

## 1. Crate layout

```
crates/ava-warp/
├── src/
│   ├── lib.rs
│   ├── message.rs        # UnsignedMessage, Message
│   ├── signature.rs      # Signature trait, BitSetSignature, VerifyWeight
│   ├── canonical.rs      # FilterValidators, SumWeight, AggregatePublicKeys, canonical-set glue
│   ├── signer.rs         # Signer trait + LocalSigner (BLS sign of an UnsignedMessage)
│   ├── codec.rs          # the warp linear codec (CodecVersion=0, type registry)
│   ├── payload.rs        # payload::{Payload, AddressedCall, Hash} + own codec
│   ├── registry.rs       # ACP-77 registry payloads + own codec
│   └── acp118.rs         # SignatureAggregator + signature-request Handler (over ava-network)
```

`ava-warp` depends on: `ava-types` (`Id`, `NodeId`, `ShortId`), `ava-codec`
(`#[derive(AvaCodec)]`), `ava-crypto` (`bls`), `ava-utils` (`set::Bits`),
`ava-validators` (`WarpSet`, `Warp`), and `ava-network` (the p2p app SDK `Client`
+ `Handler`, `05`). It is depended on by `ava-platformvm` (`08`), `ava-evm` (`10`),
and `ava-saevm` (`11`).

---

## 2. Message formats (byte-exact, linear codec)

The warp package owns **its own** `codec.Manager` (`CodecVersion = 0`), distinct
from the P/X-Chain codecs. `Codec.NewManager(math.MaxInt)` — no size cap at the
manager (the 24 KiB cap lives in the *payload* sub-codec, §3). One interface is
registered: the `Signature` interface, with `BitSetSignature` at **type ID 0**.

### 2.1 `UnsignedMessage` and `Message`

`UnsignedMessage = (NetworkID, SourceChainID, Payload)`. Its `bytes` are
`Codec.Marshal(0, msg)` (so they include the 2-byte codec version `0x0000`), and
its `id = sha256(bytes)` (single-pass; `03` §1.1). The aggregate BLS signature is
over **these `bytes`** — i.e. over the version-prefixed linear-codec encoding, not
over the raw fields.

```rust
/// `vms/platformvm/warp/unsigned_message.go`. Wire = codec_version(0x0000)
/// ++ be_u32(network_id) ++ source_chain_id[32] ++ be_u32(len) ++ payload.
#[derive(AvaCodec, Clone, Debug)]
pub struct UnsignedMessage {
    #[codec] pub network_id: u32,
    #[codec] pub source_chain_id: Id,   // [u8;32], no length prefix
    #[codec] pub payload: Vec<u8>,       // u32 len + bytes
    // cached, not serialized:
    bytes: Bytes,                        // Codec.Marshal(0, self)
    id: Id,                              // sha256(bytes)
}

impl UnsignedMessage {
    pub fn new(network_id: u32, source_chain_id: Id, payload: Vec<u8>) -> Result<Self>;
    pub fn parse(b: &[u8]) -> Result<Self>;   // unmarshal, then id = sha256(b)
    pub fn bytes(&self) -> &Bytes { &self.bytes }
    pub fn id(&self) -> Id { self.id }
}

/// `vms/platformvm/warp/message.go`. The embedded UnsignedMessage is flattened
/// (Go uses struct embedding with serialize:"true"), so on the wire a Message is
/// the UnsignedMessage fields immediately followed by the Signature interface
/// (u32 typeID + body). Note: Message::bytes re-marshals at version 0, so the
/// 2-byte version prefix appears ONCE at the front of the whole Message.
#[derive(AvaCodec, Clone, Debug)]
pub struct Message {
    #[codec] pub unsigned: UnsignedMessage,  // embedded → fields inline (no version prefix here)
    #[codec] pub signature: Signature,       // interface enum: u32 typeID ++ body
    bytes: Bytes,
}
```

> **Embedding subtlety.** Go embeds `UnsignedMessage` with `serialize:"true"`,
> which the reflect codec flattens into the parent's fields (no nested version
> prefix). In Rust, model `unsigned` as a normal `#[codec]` field whose `AvaCodec`
> impl emits *only the body* (network_id ++ source_chain_id ++ payload) — the
> 2-byte version is added once by the `Manager` for the top-level `Message`. A
> golden test (§9) pins `Message::bytes == version(0) ++ unsigned_body ++ sig`.

### 2.2 `Signature` interface + `BitSetSignature`

```rust
/// `vms/platformvm/warp/signature.go`. The codec interface (type registry).
/// Only one concrete type exists today: BitSetSignature @ type_id 0.
#[derive(AvaCodec, Clone, Debug)]
#[codec(type_registry)]
pub enum Signature {
    #[codec(type_id = 0)] BitSet(BitSetSignature),
}

/// `Signers` is a big-endian set::Bits byte slice (which validators signed);
/// `Signature` is the 96-byte aggregate BLS signature (G2 compressed, `03` §3.5).
#[derive(AvaCodec, Clone, Debug)]
pub struct BitSetSignature {
    #[codec] pub signers: Vec<u8>,        // u32 len + bytes; == set::Bits.bytes()
    #[codec] pub signature: [u8; 96],     // bls::SIGNATURE_LEN; fixed array, no prefix
}
```

There is **no** deprecated per-validator signature type still registered — the Go
codec registers `BitSetSignature` only. (Historic per-validator signatures were
removed; do not register a second type or the type IDs shift.)

### 2.3 Wire-format summary (for the differential harness)

A full `Message` decodes as:

```
0x00 0x00                       # codec version 0 (Manager prefix)
be_u32(network_id)
source_chain_id[32]
be_u32(payload_len) payload…    # the inner AddressedCall/Hash/registry bytes
be_u32(signature_type_id = 0)   # Signature interface dispatch
be_u32(signers_len) signers…    # set::Bits big-endian byte vector
signature[96]                   # aggregate BLS (G2 compressed)
```

The `UnsignedMessage::bytes` that get signed are exactly the prefix through
`payload…` (i.e. `version ++ be_u32(network_id) ++ chainID ++ payload`).

---

## 3. Payload types (sub-codec, own type registry)

Payloads have their **own** `codec.Manager` with `MaxMessageSize = 24 * KiB` and
`CodecVersion = 0`. The payload type registry order is load-bearing — these IDs
are protocol constants:

| Payload | Type ID | Go (`payload/codec.go` registration order) |
|---|---|---|
| `Hash`          | **0** | registered first |
| `AddressedCall` | **1** | registered second |

```rust
/// `vms/platformvm/warp/payload/`. Separate Manager (24 KiB cap), version 0.
#[derive(AvaCodec, Clone, Debug)]
#[codec(type_registry)]
pub enum WarpPayload {
    #[codec(type_id = 0)] Hash(HashPayload),
    #[codec(type_id = 1)] AddressedCall(AddressedCall),
}

/// AddressedCall: a call across VMs. The destination, if any, is encoded inside
/// `payload` by the consumer (Go comment: "If a destination address is expected,
/// it should be encoded in the payload").
#[derive(AvaCodec, Clone, Debug)]
pub struct AddressedCall {
    #[codec] pub source_address: Vec<u8>,  // u32 len + bytes
    #[codec] pub payload: Vec<u8>,         // u32 len + bytes
    bytes: Bytes,
}

/// Hash: a single 32-byte commitment (e.g. a block hash for block-hash proofs).
#[derive(AvaCodec, Clone, Debug)]
pub struct HashPayload {
    #[codec] pub hash: Id,                 // [u8;32]
    bytes: Bytes,
}
```

`AnycastID = Id([0xff; 32])` (`constants.go`): a sentinel destination-chain ID
meaning "any chain may receive this message"; carried inside an `AddressedCall`
payload by some consumers, not a field of `UnsignedMessage`.

### 3.1 ACP-77 registry payloads (P-Chain, third codec)

`vms/platformvm/warp/message/` is yet another `codec.Manager` (version 0) used by
the P-Chain L1 lifecycle (`08`). Registration order (= type IDs):

| Registry payload | Type ID | Fields (all `serialize:"true"`) |
|---|---|---|
| `SubnetToL1Conversion`     | **0** | `id: Id` (a hash of `SubnetToL1ConversionData`) |
| `RegisterL1Validator`      | **1** | `subnet_id, node_id(JSONByteSlice), bls_public_key:[u8;48], expiry:u64, remaining_balance_owner:PChainOwner, disable_owner:PChainOwner, weight:u64` |
| `L1ValidatorRegistration`  | **2** | `validation_id: Id, registered: bool` |
| `L1ValidatorWeight`        | **3** | `validation_id: Id, nonce: u64, weight: u64` |

```rust
/// `vms/platformvm/warp/message/codec.go` — third, independent registry.
#[derive(AvaCodec, Clone, Debug)]
#[codec(type_registry)]
pub enum RegistryPayload {
    #[codec(type_id = 0)] SubnetToL1Conversion(SubnetToL1Conversion),
    #[codec(type_id = 1)] RegisterL1Validator(RegisterL1Validator),
    #[codec(type_id = 2)] L1ValidatorRegistration(L1ValidatorRegistration),
    #[codec(type_id = 3)] L1ValidatorWeight(L1ValidatorWeight),
}

#[derive(AvaCodec, Clone, Debug)]
pub struct PChainOwner {                   // register_l1_validator.go
    #[codec] pub threshold: u32,
    #[codec] pub addresses: Vec<ShortId>,  // u32 count + each [u8;20]
}
```

These registry payloads are wrapped inside an `AddressedCall.payload`, which is the
`UnsignedMessage.payload` — i.e. three codec layers nest. The L1 justifications
(`proto/platformvm` `L1ValidatorRegistrationJustification`) ride the ACP-118
`justification` field (§5), not the message body (`15` §3.x; `08`).

> **Three independent registries (do not merge).** warp `Signature` (§2),
> `WarpPayload` (§3), and `RegistryPayload` (§3.1) each have their own `Manager`
> and their own `type_id` numbering starting at 0. Keep them as three separate
> derive-enums; reusing IDs across them is fine because they never share a codec.

---

## 4. The canonical validator set (deterministic ordering + bit-set)

The bit-set in `BitSetSignature.signers` indexes into a **canonical ordering** of
the source subnet's validators. Signer and verifier MUST derive the identical
ordering or the bits select the wrong public keys. This is `WarpSet` /
`FlattenValidatorSet` (`snow/validators/warp.go`), owned by `ava-validators` and
cross-referenced from `08` (the `GetWarpValidatorSets` BTreeMap contract).

### 4.1 `WarpSet` / `Warp`

```rust
/// snow/validators/warp.go. Lives in ava-validators; re-exported by ava-warp.
pub struct WarpSet {
    /// Canonical-ordered validators that HAVE a BLS public key.
    pub validators: Vec<Warp>,
    /// Total weight of ALL validators in the subnet — INCLUDING those without a
    /// public key (they cannot sign, but they count toward the denominator).
    pub total_weight: u64,
}

pub struct Warp {
    pub public_key: bls::PublicKey,
    /// UNCOMPRESSED (96-byte) form — this is the ordering key.
    pub public_key_bytes: [u8; 96],
    pub weight: u64,
    pub node_ids: Vec<NodeId>,
}
```

### 4.2 Flatten + canonical ordering (byte-exact)

```rust
/// FlattenValidatorSet (snow/validators/warp.go). Determinism rules (00 §6.1):
///   1. dedup validators by uncompressed-pubkey bytes (validators sharing a key
///      are merged: weights summed, node_ids concatenated);
///   2. total_weight sums EVERY validator (checked_add → error on overflow),
///      even keyless ones, BEFORE the public-key filter;
///   3. sort the merged list by `public_key_bytes` (bytes.Compare = lexicographic
///      unsigned == [u8;96] Ord).
pub fn flatten_validator_set(
    vdr_set: &BTreeMap<NodeId, GetValidatorOutput>,
) -> Result<WarpSet> {
    let mut by_pk: BTreeMap<[u8; 96], Warp> = BTreeMap::new();
    let mut total_weight: u64 = 0;
    for (node_id, vdr) in vdr_set {
        total_weight = total_weight.checked_add(vdr.weight).ok_or(Error::WeightOverflow)?;
        let Some(pk) = &vdr.public_key else { continue };       // keyless: counted, can't sign
        let pk_bytes = pk.uncompressed_bytes();                 // 96-byte form
        let entry = by_pk.entry(pk_bytes).or_insert_with(|| Warp {
            public_key: pk.clone(), public_key_bytes: pk_bytes,
            weight: 0, node_ids: Vec::new(),
        });
        entry.weight += vdr.weight;                             // cannot overflow (subset of total)
        entry.node_ids.push(*node_id);
    }
    // BTreeMap<[u8;96], _> already yields validators sorted by pubkey bytes.
    Ok(WarpSet { validators: by_pk.into_values().collect(), total_weight })
}
```

> **BTreeMap ordering contract (cross-ref `08`).** Go iterates a `map` then sorts
> via `utils.Sort` on `Warp.Compare` (`bytes.Compare` of `public_key_bytes`).
> Iterating a Go map is unordered, but because the *output* is sorted by pubkey,
> the result is deterministic. We key the dedup map by `[u8;96]` so the merge AND
> the final order both fall out of `BTreeMap` iteration — but the *node_ids* within
> one merged `Warp` are appended in **source-map iteration order**, so the
> `GetValidatorSet`/`GetWarpValidatorSets` provider in `08` MUST hand us a
> `BTreeMap<NodeId, _>` (sorted by NodeId) to make `node_ids` deterministic.
> `node_ids` ordering is not consensus-load-bearing for verification (only the
> pubkey ordering and weights are), but it IS part of the JSON/gRPC response, so
> pin it for `15` parity.

### 4.3 Bit-set semantics (`utils/set::Bits`)

`signers` is the big-endian byte encoding of a `set::Bits` (`03` §4.2). Bit `i`
set ⇒ "validator at canonical index `i` signed". Construction and reading:

```rust
// Build (aggregator, §5): start empty, Add(index) per signer, then Bytes().
let mut bits = Bits::new();
for idx in signer_indices { bits.add(idx); }
let signers: Vec<u8> = bits.bytes();   // minimal big-endian, no trailing zero padding

// Read (verify, §6): the no-padding invariant is enforced — a parsed Bits must
// re-serialize to the IDENTICAL bytes, else ErrInvalidBitSet.
let bits = Bits::from_bytes(&sig.signers);
if bits.bytes() != sig.signers { return Err(Error::InvalidBitSet); }
```

The aggregate public key is built from exactly the set bits:

```rust
/// AggregatePublicKeys (validator.go) over the FILTERED, canonical-ordered subset.
/// blst aggregate; empty set → error (NoPublicKeys).
pub fn aggregate_public_keys(vdrs: &[&Warp]) -> Result<bls::PublicKey> {
    bls::aggregate_public_keys(&vdrs.iter().map(|v| &v.public_key).collect::<Vec<_>>())
}
```

`FilterValidators(bits, vdrs)` returns the canonical-ordered subset whose bit is
set; it errors `ErrUnknownValidator` if `bits.bit_len() > vdrs.len()` (a bit set
beyond the validator count). `SumWeight` sums the filtered weights with
`checked_add` (`ErrWeightOverflow`).

---

## 5. Signing flow

### 5.1 Local sign (chain → UnsignedMessage → BLS sign)

```rust
/// vms/platformvm/warp/signer.go. The local node's authority to sign a message
/// for its own chain at its own network.
pub trait Signer {
    /// BLS signature (96-byte compressed) over msg.bytes(). Errors if this node
    /// lacks authority (wrong chain/network).
    fn sign(&self, msg: &UnsignedMessage) -> Result<[u8; 96]>;
}

pub struct LocalSigner { sk: Arc<dyn bls::Signer>, network_id: u32, chain_id: Id }

impl Signer for LocalSigner {
    fn sign(&self, msg: &UnsignedMessage) -> Result<[u8; 96]> {
        if msg.source_chain_id != self.chain_id { return Err(Error::WrongSourceChainId); }
        if msg.network_id != self.network_id    { return Err(Error::WrongNetworkId); }
        Ok(self.sk.sign(msg.bytes())?.compress())   // sign over the version-prefixed bytes
    }
}
```

The signature ciphersuite is the **signature** DST (`BLS_SIG_…_POP_`, `03` §3.5) —
*not* the PoP DST. `bls.Signer` may be a `LocalSigner` (in-process key) or the
RPC-backed `warp.Signer`/`signer.Signer` (a plugin VM signs via the host through
`proto/warp` or `proto/signer`, `07`/`15`).

### 5.2 The `warp.Signer` gRPC service (plugin ↔ host)

`proto/warp/message.proto` (`gwarp`): `Sign(network_id, source_chain_id, payload)
→ signature`. The plugin's `Client` rebuilds an `UnsignedMessage` from the three
fields and calls the host's local signer; the host's `Server` verifies authority
and returns the 96-byte signature. This is the *only* protobuf in the message
path, and it carries fields (not the marshaled `UnsignedMessage`) so the host
re-marshals canonically.

```rust
// ava-warp-rpc — tonic service mirroring proto/warp. Distinct from proto/signer
// (the raw BLS local-signer proxy, 15 §3.x).
#[tonic::async_trait]
impl warp_pb::signer_server::Signer for WarpSignerServer {
    async fn sign(&self, req: Request<SignRequest>) -> Result<Response<SignResponse>, Status> {
        let msg = UnsignedMessage::new(req.network_id, Id::from_slice(&req.source_chain_id)?, req.payload)?;
        let sig = self.inner.sign(&msg)?;            // local Signer (§5.1)
        Ok(Response::new(SignResponse { signature: sig.to_vec() }))
    }
}
```

### 5.3 The aggregator (ACP-118 signature-request/response)

The aggregator gathers per-validator signatures over the app-message protocol and
combines them into a `BitSetSignature`. Protocol identity:

- **App protocol handler ID = `2`** (`p2p.SignatureRequestHandlerID`; iota order:
  `0`=TxGossip, `1`=AtomicTxGossip, **`2`=SignatureRequest**, `3`=FirewoodProof).
  ACP-118.
- **Request** = protobuf `sdk.SignatureRequest { message: UnsignedMessage.bytes(),
  justification }` (field 1 = the linear-codec unsigned-message bytes, field 2 =
  opaque justification used by the signer's `Verify`). Sent as an **AppRequest**
  over the `ava-network` p2p `Client` (`05`).
- **Response** = protobuf `sdk.SignatureResponse { signature }` (a single 96-byte
  BLS sig over `message`), returned as an **AppResponse**.

Server side (`acp118::Handler`): unmarshal request → `UnsignedMessage::parse` →
cache lookup by `msg.id()` → `Verifier::verify(msg, justification)` (the VM
decides whether this node *should* sign; the `Signer` separately enforces
network/chain) → `Signer::sign` → cache → respond. The `Verifier` is told **not**
to re-check `NetworkID`/`SourceChainID` (the `Signer` does that).

```rust
pub struct SignatureAggregator { client: Arc<p2p::Client>, log: Logger }

impl SignatureAggregator {
    /// AggregateSignatures (network/p2p/acp118/aggregator.go). Blocks until quorum
    /// or ctx cancel; returns the assembled Message + aggregated/total stake.
    pub async fn aggregate(
        &self,
        cancel: &CancellationToken,
        message: &Message,                    // seed: may already carry some signers
        justification: &[u8],
        validators: &[Warp],                  // canonical set (caller-provided, §4)
        quorum_num: u64, quorum_den: u64,
    ) -> Result<(Message, U256 /*aggregated*/, U256 /*total*/)> {
        // 1. Read the seed BitSetSignature's existing signer bits; pre-credit their stake.
        // 2. Build node_id → indexed-validator map for the NON-signers; collect their node_ids.
        // 3. AppRequest(set_of(non_signers), proto(SignatureRequest{msg, justification))).
        // 4. min_threshold = total * quorum_num / quorum_den   (big-int, floor div).
        // 5. For each response: parse + bls::verify(vdr.pk, sig, msg.bytes());
        //    on success add the validator's canonical index to the bitset, append the
        //    sig, add its weight. Drop dup-pubkey signers (shared keys). When
        //    aggregated >= min_threshold → assemble & return early.
        // 6. On ctx cancel → return best-effort partial Message.
    }
}
```

Assembly (`newWarpMessage`): `agg = bls::aggregate_signatures(&sigs)`, then
`BitSetSignature { signers: bits.bytes(), signature: agg.compress() }`, then
`Message::new(unsigned, Signature::BitSet(...))`. **Peer selection** = the set of
node IDs that map to currently-unsigned validators (a validator with multiple node
IDs is asked at all of them). **Retry-until-quorum** is the response loop; there
is no per-peer retry inside the aggregator — the p2p `Client` owns request IDs,
timeouts, and the AppRequestFailed path (`05`).

> Note the threshold here uses **floor division** (`total*num/den` then compare
> `>=`), whereas verification (§6) uses the cross-multiplied form. They agree on
> the boundary; we keep both exactly as Go does (aggregator is best-effort, verify
> is canonical).

---

## 6. Verification flow

```rust
/// BitSetSignature::Verify (signature.go). Returns Ok(()) iff a >= quorum subset
/// of `validators` (at the pinned P-Chain height) signed `msg`.
pub fn verify(
    sig: &BitSetSignature,
    msg: &UnsignedMessage,
    network_id: u32,
    validators: &WarpSet,
    quorum_num: u64, quorum_den: u64,
) -> Result<()> {
    // 1. NetworkID must match the verifying node's network.
    if msg.network_id != network_id { return Err(Error::WrongNetworkId); }

    // 2. Parse the bit-set; enforce the no-padding invariant (§4.3).
    let bits = Bits::from_bytes(&sig.signers);
    if bits.bytes() != sig.signers { return Err(Error::InvalidBitSet); }

    // 3. Select the canonical-ordered signers; error if a bit exceeds the set size.
    let signers = filter_validators(&bits, &validators.validators)?;   // ErrUnknownValidator

    // 4. Quorum check against TOTAL weight (incl. keyless validators).
    let sig_weight = sum_weight(&signers)?;                            // checked_add
    verify_weight(sig_weight, validators.total_weight, quorum_num, quorum_den)?;

    // 5. Aggregate pubkey from exactly the set bits; parse the aggregate sig.
    let agg_pk  = aggregate_public_keys(&signers)?;                    // blst aggregate
    let agg_sig = bls::Signature::from_bytes(&sig.signature)?;         // validates on parse

    // 6. Verify the aggregate BLS signature over msg.bytes() (signature DST).
    if !bls::verify(&agg_pk, &agg_sig, msg.bytes()) { return Err(Error::InvalidSignature); }
    Ok(())
}

/// VerifyWeight (signature.go): Ok iff quorum_num*total <= quorum_den*sig_weight,
/// using 128-bit math to avoid overflow (Go uses big.Int; u128 suffices since all
/// inputs are u64). NOTE: scaled comparison, not floor-division — exact boundary.
pub fn verify_weight(sig_w: u64, total_w: u64, num: u64, den: u64) -> Result<()> {
    let lhs = (total_w as u128) * (num as u128);
    let rhs = (sig_w  as u128) * (den as u128);
    if lhs > rhs { return Err(Error::InsufficientWeight); }
    Ok(())
}
```

### 6.1 Obtaining the source-subnet validator set at a height

The verifier needs the source subnet's `WarpSet` at the **P-Chain height that
pins the message's validity window**. The chain of lookups (cross-ref `08`/`06`):

1. `subnet_id = ValidatorState::get_subnet_id(source_chain_id)` (which subnet owns
   the source chain).
2. `sets = ValidatorState::get_warp_validator_sets(p_chain_height)` →
   `BTreeMap<Id /*subnetID*/, WarpSet>`; pick `sets[&subnet_id]`.
3. `p_chain_height` comes from the **proposervm block context**
   (`ProposerVMBlockCtx.PChainHeight`) of the *verifying* block — this is what pins
   determinism: every node verifying the same block uses the same height and thus
   the same validator set. For the EVM precompile this arrives via the
   `PredicateContext` (§7).

> **Why a height pins the set.** Validator sets evolve; without a fixed height,
> nodes could disagree on quorum. The proposervm P-Chain height makes warp
> verification a pure function of `(message, height)`. `ava-validators` serves
> `get_warp_validator_sets` from the P-Chain state at that height (`08`).

### 6.2 Quorum constants

| Constant | Value | Source |
|---|---|---|
| `WarpDefaultQuorumNumerator` | **67** | subnet-evm `warp/config.go` |
| `WarpQuorumNumeratorMinimum` | **33** | min configurable numerator |
| `WarpQuorumDenominator` | **100** | fixed denominator |

P-Chain warp verification (`08`) uses its own quorum (often the same 67/100). The
EVM precompile defaults to 67/100 but allows a per-deployment `quorumNumerator`
(validated: `0 < n` ⇒ must be in `[33, 100]`; `0` means "use default").

---

## 7. C-Chain / EVM integration

The EVM exposes Warp through a **stateful precompile** at
`0x0200000000000000000000000000000000000005` (`module.go`), reused by C-Chain
(coreth) and EVM subnets (subnet-evm). Cross-ref `10` §8 (precompile model on
revm's `PrecompileProvider`) and `10` §6.5/G4 (predicates).

### 7.1 Precompile ABI

| Selector fn | Effect |
|---|---|
| `getBlockchainID() → bytes32` | the snow context `ChainID` (this chain's blockchain ID) |
| `sendWarpMessage(bytes payload) → bytes32` | constructs an `AddressedCall { source_address = caller, payload }`, wraps in an `UnsignedMessage(networkID, thisChainID, addressedCall)`, **emits a `SendWarpMessage` log** (3 topics + data), returns the unsigned-message ID. No signing here — off-chain aggregators sign later. |
| `getVerifiedWarpMessage(uint32 index) → (WarpMessage, bool valid)` | reads the **predicate result** for the warp message at predicate `index` (set during block verify); returns the parsed `AddressedCall` fields + `valid`. |
| `getVerifiedWarpBlockHash(uint32 index) → (WarpBlockHash, bool valid)` | same, for `Hash` payloads (block-hash proofs). |

### 7.2 Predicate verification (the BLS check happens here, not in EVM)

The aggregate-signature verification is a **predicate** evaluated during block
verification, *before* EVM execution, and its boolean result is stored for
`getVerifiedWarpMessage` to read. This is `10` G4 ("predicate results into the
precompile context"). `Config::verify_predicate` (subnet-evm `warp/config.go`):

1. `warp::Message::parse(predicate_bytes)`.
2. `source_subnet = ValidatorState::get_subnet_id(msg.source_chain_id)`.
3. If `source_subnet == PrimaryNetworkID` **and** (`!require_primary_network_signers`
   **or** `source_chain_id == PlatformChainID`) ⇒ substitute `source_subnet =
   SnowCtx.SubnetID` (verify against the local, smaller subnet set; the P-Chain is
   always trusted because it is always synced).
4. `sets = ValidatorState::get_warp_validator_sets(ProposerVMBlockCtx.PChainHeight)`;
   `validator_set = sets[source_subnet]`.
5. `msg.signature.verify(&msg.unsigned, SnowCtx.NetworkID, validator_set,
   quorum_num, WarpQuorumDenominator)` — i.e. §6.

The `PChainHeight` is taken from the **proposervm block context** of the block
being verified, so all validators agree. `ava-evm` runs this in
`apply_pre_execution_changes` and stashes `Vec<bool>` predicate results keyed by
index (`10` §6.5).

### 7.3 Gas costs

| Param | Pre-Granite | Granite | Notes |
|---|---|---|---|
| `getBlockchainID` | 2 | **200** | |
| `getVerifiedWarpMessageBase` | 2 | **750** | base for `getVerifiedWarpMessage` |
| `sendWarpMessageBase` | `LogGas + 3*LogTopicGas + 20_000 + WriteGasCostPerSlot` | same | `addWarpMessageBaseGasCost = 20_000` (cost of producing/serving a BLS sig) |
| `perWarpMessageByte` | `LogDataGas` | same | per payload byte on send |
| per-signer / per-chunk verify gas | — | — | `PredicateGas` charges per warp **signer** in the set + per 32-byte chunk + per-BLS-verify; `SafeMul`/`SafeAdd` with overflow → error |

Granite raised read/verify costs "to target a worst-case verification cost". Copy
both `GasConfig` tables verbatim into `ava-evm` and select by active fork (`10`).

---

## 8. Go → Rust mapping (non-obvious)

| Go | Rust | Note |
|---|---|---|
| `warp.Codec` (`NewManager(MaxInt)`) | `ava-warp::codec::manager()` | version 0; one type: `BitSetSignature@0` |
| embedded `UnsignedMessage serialize:"true"` | `#[codec] unsigned` emitting body-only | version prefix added once for `Message` |
| `Signature` interface | `enum Signature { #[codec(type_id=0)] BitSet }` | type-registry enum |
| `set.BitsFromBytes` / `.Bytes()` no-padding check | `Bits::from_bytes` + `bits.bytes()==signers` | `ErrInvalidBitSet` |
| `FlattenValidatorSet` (map→sort by pubkey) | `BTreeMap<[u8;96], Warp>` keyed dedup | total_weight counts keyless vdrs |
| `big.Int` quorum cross-multiply | `u128` mul-compare | inputs are u64; no overflow |
| big-int floor-div threshold (aggregator) | `U256`/`u128` floor-div | best-effort only |
| `bls.Verify(aggPk, aggSig, unsignedBytes)` | `bls::verify` (signature DST) | over `msg.bytes()` |
| `proto/warp` `Signer.Sign` | `ava-warp-rpc` tonic `Signer` | carries fields, host re-marshals |
| `sdk.SignatureRequest/Response` (protobuf) | `prost` messages over p2p `Client` | handler ID `2`, ACP-118 |
| `acp118.SignatureAggregator` | `SignatureAggregator` (tokio) | `mpsc` results channel; `select!` on cancel |
| `acp118.Handler` + `Verifier` + `cache` | `Handler` + `Verifier` trait + `moka` cache | cache by `msg.id()` |
| `precompile/.../warp` predicate | `ava-evm` predicate pass | result `Vec<bool>` → precompile ctx |

## 9. Error model

`ava-warp::Error` (thiserror), preserving Go sentinel identities (asserted where Go
uses `errors.Is`, `00` §7.1):

- Signature/verify: `InvalidBitSet`, `InsufficientWeight`, `InvalidSignature`,
  `ParseSignature`, `WrongNetworkId` (`ErrWrongNetworkID`), `WrongSourceChainId`.
- Canonical set: `UnknownValidator`, `WeightOverflow`.
- BLS/codec parse errors flow up from `ava-crypto::Error` / `ava-codec::CodecError`
  (e.g. `ExtraSpace`, `UnknownTypeId`).
- Payload: `WrongType` (`ErrWrongType`), `MaxMessageSize` exceeded (24 KiB cap on
  the payload sub-codec).

## 10. Test plan (see `02-testing-strategy.md`)

**Byte-exact golden vectors (highest priority — differential vs Go):**
1. **Message bytes.** For a fixed `(network_id, source_chain_id, payload)`, dump
   Go `UnsignedMessage.Bytes()` (hex) and assert Rust `marshal` matches; same for a
   full `Message` (with a `BitSetSignature`). Pin the `0x0000` version prefix
   position and the embedded-flatten layout (§2.1).
2. **Type-ID tables.** Golden table of `(registry, type_id)` for all three
   registries (`Signature@0`; `Hash@0`,`AddressedCall@1`; the four registry
   payloads `@0..3`). Assert derive discriminants.
3. **Payload nesting.** Round-trip `AddressedCall` ⊂ `UnsignedMessage` and the
   ACP-77 registry payloads ⊂ `AddressedCall`; oversize (>24 KiB) rejection.
4. **Canonical ordering.** Given a Go-dumped validator map, assert `WarpSet`
   ordering, dedup-by-pubkey merging, and `total_weight` (incl. keyless) match
   `FlattenValidatorSet` exactly. Property test: shuffling the input map yields an
   identical canonical order.
5. **Bit-set.** No-padding invariant (a padded `signers` rejects with
   `InvalidBitSet`); `FilterValidators` selects the right subset; out-of-range bit
   → `UnknownValidator`.
6. **Quorum.** Boundary tests on `verify_weight` (exactly-at-threshold passes);
   compare `u128` cross-multiply vs a `big`-int reference; randomized proptest vs
   Go's `VerifyWeight`.
7. **Sign/verify round-trip.** Local sign N validators → aggregate → `verify` Ok;
   flip one bit / one weight → fails. Aggregate-verify proptest over random
   subsets and quorums.
8. **Signature golden vector.** A Go node signs an `UnsignedMessage`; assert the
   Rust node's `verify` accepts the resulting `Message`. **And vice-versa** (Rust
   signs, Go verifies) — the load-bearing differential test (`02`): a Go-signed
   warp message verifies on the Rust node and a Rust-signed one verifies on Go.
9. **ACP-118 protocol.** Encode/decode `sdk.SignatureRequest`/`Response`; an
   `acp118::Handler` (Rust) answers a Go aggregator's request and vice-versa over a
   loopback p2p `Client`; assert handler ID `2`; cache hit path; `Verifier`
   rejection → `AppError`.
10. **Precompile.** `getBlockchainID`/`sendWarpMessage` log+return; predicate-verify
    feeds a `Vec<bool>` that `getVerifiedWarpMessage` reads; gas costs match both
    pre-Granite and Granite `GasConfig` tables; `requirePrimaryNetworkSigners`
    subnet-substitution branch (§7.2 step 3).

## 11. Performance notes / improvements over Go

- **BLS aggregate batch verify (cross-ref `03` §9).** Verification aggregates
  pubkeys then does ONE pairing check — already aggregate, not per-signature. The
  win is in the **aggregator's response loop** and in **verifying many independent
  warp messages in a block**: use `blst`'s multi-point/`Pairing` batch API to
  verify a batch of `(agg_pk, agg_sig, msg)` triples in parallel on a rayon pool.
  **Safe because** verification has no ordering side effects (`00` §6.1/§9); back
  with the differential test (§10.8).
- **Zero-copy parse.** `UnsignedMessage.bytes` / `Message.bytes` are `bytes::Bytes`
  borrowed from the predicate/app-message buffer; the signed message body is a
  sub-slice, no re-marshal on the verify hot path.
- **Canonical-set cache.** `get_warp_validator_sets(height)` results are
  `arc-swap`/`moka`-cached per `(subnet_id, p_chain_height)`; reads are lock-free
  (cross-ref `06` sharded validator reads). The set is immutable for a fixed
  height, so caching is sound.
- **Signature cache.** The `acp118::Handler` caches by `msg.id()` (Go caches too);
  use `moka` for concurrent reads. The aggregator's per-response BLS verify runs on
  `spawn_blocking`/rayon, never on the reactor (`00` §7.2).
- **One `ava-warp` crate, three consumers.** P-Chain (`08`), EVM precompile (`10`),
  and SAE (`11`) share the identical verify path — no divergent reimplementations,
  so the differential guarantee holds uniformly.
