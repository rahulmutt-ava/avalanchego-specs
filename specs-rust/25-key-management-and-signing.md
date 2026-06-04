# 25 — Key Management & Signing

> Conforms to `00-overview-and-conventions.md`. This spec gathers the node's
> **identity-key lifecycle** (the staking TLS key/cert and the BLS staking key) and
> the **signer abstraction** that hides those keys behind a trait — including the
> in-process local signers and the **remote-signer gRPC services** a node may
> consume (or, for warp, expose). It connects the crypto primitives in `03` §3, the
> P2P handshake/signed-IP in `05`, the warp signer in `20`, the `proto/signer` +
> `proto/warp` services in `15`, and the staking-* flags in `13`.
>
> **Byte-exact requirement.** NodeID derivation, the BLS public key, and the
> Proof-of-Possession (PoP) are persisted on-chain and exchanged on the wire; they
> MUST match the Go node bit-for-bit (`00` §1). This spec fixes their formats and
> the loading precedence; the underlying crypto math lives in `03` §3.

---

## 0. Go source coverage

| Go source | What it provides | Rust home |
|---|---|---|
| `staking/tls.go` | TLS key/cert generation (`NewCertAndKeyBytes`, `NewTLSCert`), init-when-absent (`InitNodeStakingKeyPair`), PEM load (`LoadTLSCertFromFiles`/`FromBytes`) | `ava-crypto::staking` (`03` §3.6) |
| `staking/{parse,verify,certificate,asn1}.go` | strict cert parse, `CheckSignature`, `Certificate` | `ava-crypto::staking` (`03` §3.6) |
| `ids/node_id.go` | `NodeIDFromCert` = ripemd160(sha256(cert.DER)) | `ava-types` (`03` §1) |
| `utils/crypto/bls/signer.go` | `bls.Signer` interface (`PublicKey`, `Sign`, `SignProofOfPossession`, `Shutdown`) | `ava-crypto::bls::Signer` |
| `utils/crypto/bls/signer/localsigner/localsigner.go` | in-process BLS key: `New`, `FromBytes`, `FromFile`, `ToFile`, `FromFileOrPersistNew` | `ava-crypto::bls::LocalSigner` |
| `utils/crypto/bls/signer/rpcsigner/client.go` | remote BLS signer client over `proto/signer` | `ava-crypto::bls::RpcSigner` (tonic) |
| `proto/signer/signer.proto` | the `signer.Signer` gRPC service (raw BLS key proxy) | `ava-signer-rpc` (`15` §3.9) |
| `proto/warp/message.proto` | the `warp.Signer` gRPC service (sign an `UnsignedMessage`) | `ava-warp-rpc` (`20` §5.2, `15` §3.8) |
| `vms/platformvm/signer/proof_of_possession.go` | the `ProofOfPossession` registry type (`pk‖pop`) | `ava-platformvm::signer` (`08`) |
| `network/peer/{ip,ip_signer}.go` | signed-IP construction (TLS + BLS), re-sign cache | `ava-network::peer` (`05`) |
| `node/node.go` (`newStakingSigner`, `StakingSigner`, `StakingConfig`) | wires the configured signer + builds the PoP | `ava-node` (`12`) |
| `config/config.go` (`getStakingTLSCert`, `getStakingSignerConfig`) | flag → key precedence | `ava-config` (`12`/`13`) |

This spec **does not** redefine the crypto math (`03` §3.4/§3.5/§3.6) or the warp
message format (`20`); it defines the *lifecycle and the signer trait* that bind
them.

---

## 1. The node's two identity keys

An Avalanche node carries **two independent long-lived keys**. They are unrelated
cryptosystems with different roles, formats, and on-disk files. Do not conflate
them.

| | **Staking TLS key** | **BLS staking key** |
|---|---|---|
| Algorithm | ECDSA P-256 (self-signed X.509 cert) | BLS12-381 `min_pk` (`03` §3.5) |
| Derives | **NodeID** = ripemd160(sha256(cert.DER)) | BLS **public key** (48-byte compressed) + **PoP** |
| Used for | mutual-TLS P2P handshake (`05`); **TLS signature** over the signed-IP | warp/ICM message signing (`20`); **BLS PoP signature** over the signed-IP; registered on-chain as a validator |
| Default file | `<data>/staking/staker.key` + `staker.crt` | `<data>/staking/signer.key` |
| File format | PEM `PRIVATE KEY` (PKCS#8) + PEM `CERTIFICATE` | raw 32-byte big-endian secret-key serialization (`SecretKey::serialize`) — **not** PEM |
| Generated when absent | yes (TLS cert+key pair) | yes (`FromFileOrPersistNew`) |
| Rotatable | rotating ⇒ **new NodeID** (re-staking required) | rotating ⇒ new validator pubkey; on-chain update via `TransformSubnetTx`/L1 ops (`08`) |

> **Signed-IP uses BOTH keys, not one.** The handshake's signed IP carries a TLS
> signature *and* a BLS signature (`network/peer/ip.go::Sign`): the TLS signature is
> over `SHA256(ipBytes)` with the staking key (`crypto.SHA256`); the BLS signature
> is a **proof-of-possession** signature over the *raw* `ipBytes`. So the BLS key is
> not only for warp — it also proves liveness/ownership of the validator's BLS key
> in the handshake. See `05` §1.6 and §4 below.

### 1.1 On-disk layout (the `staking/` subdir)

```
<data-dir>/                     # --data-dir (default ~/.avalanchego)
└── staking/                    #   dir perms 0o700 (perms.ReadWriteExecute on create)
    ├── staker.key              #   --staking-tls-key-file   (PEM PKCS#8)  0o400
    ├── staker.crt              #   --staking-tls-cert-file  (PEM cert)    0o400
    └── signer.key              #   --staking-signer-key-file (32B raw)    0o400
```

All three are written **read-only** (`perms.ReadOnly` = `0o400`) after creation and
the parent dir `0o700` (`perms.ReadWriteExecute`). The Rust port replicates the
exact mode bits (mirror with `std::fs::set_permissions` on Unix; on Windows the
mode is advisory as in Go). `ava-config` resolves the paths via `getExpandedArg`
(env-var expansion, `12`/`13`).

---

## 2. NodeID & PoP derivation (cross-ref `03`)

### 2.1 NodeID from the TLS cert

```rust
/// NodeID = ripemd160(sha256(cert.DER)) over the ENTIRE DER-encoded cert
/// (cert.Raw), NOT the public key. ids/node_id.go::NodeIDFromCert.
pub fn node_id_from_cert(cert_der: &[u8]) -> NodeId {
    NodeId::from(pubkey_bytes_to_address(cert_der)) // ripemd160(sha256(·)), 03 §3.1
}
```

The input is the whole certificate, so any change to the cert (new key, new
validity window, re-generation) yields a **different NodeID**. An **ephemeral** TLS
cert (`--staking-ephemeral-cert-enabled`) therefore gives the node a fresh,
throwaway identity each boot.

### 2.2 Proof-of-Possession (`ava-platformvm::signer`)

The validator publishes `(pk, pop)` where the **message signed is the compressed
public-key bytes themselves**:

```rust
/// Mirrors vms/platformvm/signer/ProofOfPossession.
/// Layout (linear codec / on-chain): PublicKey[48] ‖ ProofOfPossession[96].
pub struct ProofOfPossession {
    pub public_key: [u8; bls::PUBLIC_KEY_LEN],  // 48, compressed G1
    pub proof:      [u8; bls::SIGNATURE_LEN],   // 96, compressed G2
}

impl ProofOfPossession {
    pub fn new(signer: &dyn bls::Signer) -> Result<Self> {
        let pk_bytes = signer.public_key().compress();        // 48 bytes
        let sig = signer.sign_proof_of_possession(&pk_bytes)?; // POP ciphersuite
        Ok(Self { public_key: pk_bytes, proof: sig.compress() })
    }
    /// verify_pop(pk, pop, pk_bytes) — message is the pubkey bytes. POP DST.
    pub fn verify(&self) -> Result<()> { /* 03 §3.5 verify_pop */ }
}
```

- **JSON form** (`MarshalJSON`): `{"publicKey": "0x…", "proofOfPossession": "0x…"}`
  using `formatting.HexNC` (`0x`-prefixed hex, **no** checksum — `03` §3.2). Must
  match Go's serialization exactly (consumed by the P-Chain `addValidator` APIs,
  `14`).
- **Ciphersuite:** PoP uses `CIPHERSUITE_POP` (`BLS_POP_…`); plain `Sign` uses
  `CIPHERSUITE_SIGNATURE` (`BLS_SIG_…`). Byte-exact DST strings in `03` §3.5. The
  PoP and the signed-IP BLS signature both go through `sign_proof_of_possession`.

---

## 3. The `Signer` abstraction

### 3.1 BLS `Signer` trait

The Go `bls.Signer` interface is object-safe and has three signing operations plus
`Shutdown`. The Rust trait mirrors it; every consumer holds an `Arc<dyn Signer>` so
the key is **never** copied out — only signatures and the public key leave the
boundary.

```rust
// ava-crypto::bls
use std::sync::Arc;

/// Mirrors utils/crypto/bls/signer.go::Signer. Object-safe; the concrete key
/// (local secret key, or a remote endpoint) lives behind it and never escapes.
#[async_trait::async_trait]
pub trait Signer: Send + Sync {
    /// The BLS public key (already subgroup-validated). Cheap; callers cache it.
    fn public_key(&self) -> &PublicKey;

    /// Sign `msg` with the SIGNATURE ciphersuite (warp / app messages).
    async fn sign(&self, msg: &[u8]) -> Result<Signature>;

    /// Sign `msg` with the POP ciphersuite (PoP + signed-IP).
    async fn sign_proof_of_possession(&self, msg: &[u8]) -> Result<Signature>;

    /// Release any held resources (closes the gRPC channel for the remote impl).
    async fn shutdown(&self) -> Result<()> { Ok(()) }
}
```

> **Sync vs async.** The Go local signer is synchronous; the RPC signer makes a
> blocking gRPC call. We make the trait `async` so the **remote** impl is a
> first-class member without spawning blocking threads per signature. The local
> impl is CPU-only (sub-millisecond) and simply does the work inline (it may also
> be offered through a sync convenience method for hot non-async call sites such as
> the warp aggregator's per-message signing, wrapped via the runtime handle).

### 3.2 Local in-process signer

```rust
// ava-crypto::bls — wraps a blst min_pk::SecretKey; the key is zeroized on drop.
pub struct LocalSigner {
    sk: zeroize::Zeroizing<SecretKeyBytes>, // 32B, big-endian; zeroized
    pk: PublicKey,                          // cached, derived once
}

impl LocalSigner {
    /// localsigner.New: 32 bytes of CSPRNG IKM -> key_gen; IKM zeroized after.
    pub fn generate() -> Result<Self> { /* rand IKM -> SecretKey::key_gen; zeroize ikm */ }
    /// localsigner.FromBytes: big-endian SecretKey::deserialize; reject malformed.
    pub fn from_bytes(b: &[u8]) -> Result<Self> { /* ErrFailedSecretKeyDeserialize */ }
    /// localsigner.FromFile: read the 32-byte file then from_bytes.
    pub fn from_file(path: &Path) -> Result<Self> { /* read + from_bytes */ }
    /// localsigner.ToFile: write 32B, dir 0o700, then chmod 0o400.
    pub fn to_file(&self, path: &Path) -> Result<()> { /* mkdir 0o700; write; chmod 0o400 */ }
    /// localsigner.FromFileOrPersistNew: load if present, else generate + persist.
    pub fn from_file_or_persist_new(path: &Path) -> Result<Self> { /* stat -> from_file | (generate+to_file) */ }
}

#[async_trait::async_trait]
impl Signer for LocalSigner {
    fn public_key(&self) -> &PublicKey { &self.pk }
    async fn sign(&self, msg: &[u8]) -> Result<Signature> { /* sign(sk, msg, SIG_DST) */ }
    async fn sign_proof_of_possession(&self, msg: &[u8]) -> Result<Signature> { /* sign(sk, msg, POP_DST) */ }
}
```

The 32-byte file format is `SecretKey::serialize()` (big-endian, `blst`), **not**
PEM — matching `LocalSigner.ToBytes/FromBytes`. A Go-written `signer.key` opens
identically and vice-versa.

### 3.3 Remote signer (`proto/signer`, tonic)

The remote BLS signer proxies the three operations over the **`signer.Signer`**
gRPC service (`15` §3.9). The connection is **plaintext loopback** (Go uses
`insecure.NewCredentials()`): the node is expected to talk to a *local proxy* on
the same machine that forwards to the real (possibly HSM-backed) signer — the
trust boundary is the loopback socket, not TLS. The public key is fetched once at
construction and cached.

```rust
// ava-crypto::bls — tonic client for proto/signer (signer.SignerClient).
pub struct RpcSigner {
    client: SignerClient<tonic::transport::Channel>,
    pk: PublicKey, // fetched once via PublicKey() at connect; cached
}

impl RpcSigner {
    /// rpcsigner.NewClient: dial (insecure loopback), call PublicKey(), parse+validate.
    pub async fn connect(endpoint: String) -> Result<Self> {
        let channel = tonic::transport::Channel::from_shared(endpoint)?
            .connect_lazy(); // grpc handles transient reconnects
        let mut client = SignerClient::new(channel);
        let resp = client.public_key(PublicKeyRequest {}).await?;
        let pk = PublicKey::from_compressed(&resp.into_inner().public_key)?; // subgroup-validated
        Ok(Self { client, pk })
    }
}

#[async_trait::async_trait]
impl Signer for RpcSigner {
    fn public_key(&self) -> &PublicKey { &self.pk }
    async fn sign(&self, msg: &[u8]) -> Result<Signature> {
        let r = self.client.clone().sign(SignRequest { message: msg.to_vec() }).await?;
        Signature::from_bytes(&r.into_inner().signature) // uncompress + sig_validate
    }
    async fn sign_proof_of_possession(&self, msg: &[u8]) -> Result<Signature> {
        let r = self.client.clone()
            .sign_proof_of_possession(SignProofOfPossessionRequest { message: msg.to_vec() }).await?;
        Signature::from_bytes(&r.into_inner().signature)
    }
    async fn shutdown(&self) -> Result<()> { Ok(()) /* channel dropped */ }
}
```

> **Ciphersuite responsibility.** `proto/signer` carries only the raw `message`;
> the *server* applies the DST. The client must therefore route `Sign`/`SignPoP`
> to the matching RPC method (it does — distinct RPCs), and the server must use the
> SIGNATURE vs POP ciphersuite per method. The Rust server impl (for a Rust-hosted
> signer) mirrors `LocalSigner` behind the tonic service.

### 3.4 The `warp.Signer` service (distinct from `proto/signer`)

`proto/warp/message.proto` defines a **different** service, `warp.Signer`, used in
the **rpcchainvm plugin↔host** direction (`07`/`20` §5.2): a plugin VM that wants a
warp signature sends `Sign(network_id, source_chain_id, payload)` and the **host**
rebuilds the `UnsignedMessage`, signs it with its local `bls::Signer` over the
**SIGNATURE** ciphersuite, and returns the 96-byte signature (`20` §5.1). This is
the only place the node *exposes* a signer (to its own plugins), gated by the
host's authority checks. The two services share no proto package:

| Service | Package | Operations | Direction | Defined |
|---|---|---|---|---|
| `signer.Signer` | `signer` | `PublicKey`, `Sign`, `SignProofOfPossession` (raw BLS proxy) | node → external signer | `15` §3.9 |
| `warp.Signer` | `warp` | `Sign(network_id, chain_id, payload)` (signs an `UnsignedMessage`) | plugin → host | `15` §3.8, `20` §5.2 |

`ava-signer-rpc` (client+server for `signer.Signer`) lives in `ava-crypto`;
`ava-warp-rpc` (the `warp.Signer` client+server) lives with warp (`20`).

### 3.5 How consumers obtain a signer handle

The node constructs **one** `Arc<dyn bls::Signer>` (`StakingSigner`) at startup and
hands clones to:

- **the IP signer** (`05` §3.5): `NetworkConfig.BLSKey = StakingSigner`; the
  `IpSigner` calls `sign_proof_of_possession(ip_bytes)` on each re-sign.
- **the PoP builder** (`ProofOfPossession::new(&*staking_signer)`), used by the
  `info.getNodeID`/health/registration paths.
- **warp** (`20`): the local `warp::Signer` wraps `Arc<dyn bls::Signer>` plus the
  `network_id`/`chain_id`; the ACP-118 handler signs requests through it.
- **the C-Chain / plugins** via the `warp.Signer` host service.

The **TLS** key is *not* funneled through `bls::Signer`; it is consumed directly as
a `rustls`/`crypto::Signer` by the TLS stack (`05`) and by `UnsignedIP::sign` for
the TLS signature. We keep a parallel narrow handle (`Arc<dyn TlsSigner>` wrapping
the loaded `tls::Certificate` private key) so the raw ECDSA key likewise never
escapes the abstraction.

---

## 4. The signed-IP (cross-ref `05`)

`UnsignedIP { addr_port, timestamp }` is serialized as
`ip.As16() (16) ‖ port (u16 BE) ‖ timestamp (u64 BE)` via the `wrappers.Packer`
(`05` §1.6). Signing produces a `SignedIp` with **two** signatures:

```rust
// ava-network::peer — mirrors network/peer/ip.go::Sign
pub async fn sign_ip(
    ip: &UnsignedIp,
    tls: &dyn TlsSigner,          // ECDSA P-256 staking key
    bls: &dyn bls::Signer,        // BLS staking key
) -> Result<SignedIp> {
    let bytes = ip.bytes();
    let tls_sig = tls.sign(&sha256(&bytes))?;          // ECDSA over SHA256(ipBytes), crypto.SHA256
    let bls_sig = bls.sign_proof_of_possession(&bytes).await?; // BLS POP over raw ipBytes
    Ok(SignedIp { unsigned: ip.clone(), tls_sig, bls_sig: bls_sig.compress() })
}
```

`SignedIp::verify(cert, max_timestamp)` checks `timestamp <= max_timestamp` then
`staking::check_signature(cert, ip.bytes(), tls_sig)` (`03` §3.6). The BLS
signature is verified separately for registered validators in
`should_disconnect` (`05` §3.3). The `IpSigner` caches the `SignedIp` behind
`arc-swap` and re-signs only when the dynamic IP changes (`05` §3.5).

---

## 5. Config surface & loading precedence (cross-ref `13`)

### 5.1 Staking TLS cert precedence (`getStakingTLSCert`)

```rust
// ava-config — mirrors config/config.go::getStakingTLSCert
fn load_staking_tls(cfg: &Viper) -> Result<TlsCertificate> {
    if cfg.get_bool(STAKING_EPHEMERAL_CERT_ENABLED) {
        return staking::new_tls_cert(); // fresh ephemeral cert+key -> ephemeral NodeID
    }
    let key_set  = cfg.is_set(STAKING_TLS_KEY_CONTENT);   // base64 PEM key
    let cert_set = cfg.is_set(STAKING_CERT_CONTENT);      // base64 PEM cert
    match (key_set, cert_set) {
        (true, false) => Err(Error::StakingCertContentUnset), // both-or-neither
        (false, true) => Err(Error::StakingKeyContentUnset),
        (true, true)  => staking::load_from_bytes(           // *-content wins
            base64_decode(cfg.get_string(STAKING_TLS_KEY_CONTENT))?,
            base64_decode(cfg.get_string(STAKING_CERT_CONTENT))?,
        ),
        (false, false) => {
            // file path. If the *path flags were explicitly set, the files MUST
            // exist; otherwise generate+persist when absent (InitNodeStakingKeyPair).
            let (kp, cp) = (expand(cfg, STAKING_TLS_KEY_PATH), expand(cfg, STAKING_CERT_PATH));
            if cfg.is_set(STAKING_TLS_KEY_PATH) || cfg.is_set(STAKING_CERT_PATH) {
                require_exists(&kp)?; require_exists(&cp)?;
            } else {
                staking::init_node_staking_key_pair(&kp, &cp)?; // generate if missing
            }
            staking::load_from_files(&kp, &cp)
        }
    }
}
```

Precedence (highest first): **ephemeral flag** > **`*-content` (base64) pair** >
**file path** (explicit path ⇒ must exist; default path ⇒ generate-when-absent).
The key+cert `-content` pair is **both-or-neither** (`errStakingKeyContentUnset` /
`errStakingCertContentUnset`).

### 5.2 BLS signer precedence (`getStakingSignerConfig` + `newStakingSigner`)

The four signer options are **mutually exclusive — at most one** may be set
(`errInvalidSignerConfig`, `13` §5). `newStakingSigner` resolves them in this
order:

```rust
// ava-node — mirrors node/node.go::newStakingSigner
async fn new_staking_signer(cfg: &StakingSignerConfig) -> Result<Arc<dyn bls::Signer>> {
    if cfg.ephemeral_signer_enabled {                        // --staking-ephemeral-signer-enabled
        return Ok(Arc::new(LocalSigner::generate()?));       // throwaway key, never persisted
    }
    if !cfg.key_content.is_empty() {                         // --staking-signer-key-file-content (base64)
        let raw = base64_decode(&cfg.key_content)?;
        return Ok(Arc::new(LocalSigner::from_bytes(&raw)?));
    }
    if !cfg.rpc_endpoint.is_empty() {                         // --staking-rpc-signer-endpoint
        return Ok(Arc::new(RpcSigner::connect(cfg.rpc_endpoint.clone()).await?)); // 5s connect timeout
    }
    if cfg.key_path_is_set {                                  // --staking-signer-key-file (explicit)
        return Ok(Arc::new(LocalSigner::from_file(&cfg.key_path)?)); // MUST exist
    }
    // default path: load, or generate+persist when absent
    Ok(Arc::new(LocalSigner::from_file_or_persist_new(&cfg.key_path)?))
}
```

> **Subtle ordering difference vs TLS.** For the BLS key, an **explicitly set**
> `--staking-signer-key-file` (`key_path_is_set`) uses `from_file` (the file MUST
> exist — no generation), whereas the **default** path uses
> `from_file_or_persist_new` (generate-when-absent). This matches Go exactly. The
> `key_path` field is only populated when none of the other three options are set
> (`getStakingSignerConfig`), so the precedence is unambiguous.

### 5.3 First-boot behavior

On a fresh data dir with no flags overriding defaults, the node:
1. generates `staking/staker.key` + `staker.crt` (→ a stable NodeID) and
2. generates `staking/signer.key` (→ a stable BLS key + PoP),
then writes all three `0o400`. Subsequent boots load them. This is the default
"just works" path and MUST produce a NodeID/PoP that a Go node would accept (and
that round-trips if the same files are later loaded by a Go node).

---

## 6. Security posture

- **Keys at rest.** Both private keys are `0o400`, parent dir `0o700`. The Rust
  port sets identical permissions and refuses to weaken them. The TLS key is PKCS#8
  PEM; the BLS key is raw 32 bytes — neither is encrypted at rest (matching Go), so
  filesystem permissions are the only protection. Operators wanting HSM/KMS use the
  **remote signer** (`--staking-rpc-signer-endpoint`).
- **The key never leaves the signer.** The secret key material is owned solely by
  the concrete `Signer` impl. Public API exposes only the public key and produced
  signatures. No `to_bytes`/serialize on the trait — only `LocalSigner` (which owns
  the file lifecycle) exposes it, and only for `to_file`.
- **Zeroization.** `LocalSigner` wraps the 32-byte secret in `zeroize::Zeroizing`
  (and key-gen IKM is zeroized immediately after use, as Go zeroizes `ikm` and sets
  a `blst` finalizer that calls `Zeroize`). This is the Rust equivalent of Go's
  `runtime.SetFinalizer(sk, Zeroize)` — but deterministic (on drop, not GC).
- **`forbid(unsafe_code)` boundary.** `blst` and `secp256k1`/`rustls` require FFI;
  per `00` §7.6 they are isolated behind `ava-crypto`'s safe wrapper modules with a
  `// SAFETY:` rationale. The `Signer` trait surface itself is 100% safe Rust.
- **Remote-signer trust model.** The gRPC channel is **insecure loopback** by
  design (Go uses `insecure.NewCredentials()`): the node trusts a same-host proxy
  that brokers to the real signer/HSM. The Rust client uses a plaintext tonic
  channel; we do **not** add TLS (it would diverge from the documented deployment
  model). The connection retries transient failures (tonic `connect_lazy` +
  reconnect backoff, mirroring grpc's `ConnectParams`); the public key is fetched
  once and is the identity anchor.
- **Validation on parse.** Remote-returned public keys and signatures are subgroup-
  validated on `from_compressed`/`from_bytes` (`03` §3.5), so a malicious signer
  cannot inject invalid group elements.

---

## 7. Go → Rust mapping

| Go | Rust | Notes |
|---|---|---|
| `bls.Signer` (interface) | `bls::Signer` (`#[async_trait]`) | async to host the remote impl natively |
| `localsigner.LocalSigner` | `bls::LocalSigner` | `Zeroizing` secret; same 32B file format |
| `rpcsigner.Client` | `bls::RpcSigner` | tonic client over `proto/signer`; insecure loopback |
| `localsigner.FromFileOrPersistNew` | `LocalSigner::from_file_or_persist_new` | stat → load | (generate+persist) |
| `signer.ProofOfPossession` | `platformvm::signer::ProofOfPossession` | `pk[48]‖pop[96]`; `HexNC` JSON |
| `staking.NewCertAndKeyBytes` | `staking::new_cert_and_key_bytes` (`rcgen`) | ECDSA P-256, the exact cert template (`03` §3.6) |
| `staking.InitNodeStakingKeyPair` | `staking::init_node_staking_key_pair` | generate-when-absent, `0o400`/`0o700` |
| `staking.LoadTLSCertFromFiles/Bytes` | `staking::load_from_files/bytes` | PEM via `rustls-pemfile` |
| `tls.Certificate` (Go) | `TlsCertificate` + `Arc<dyn TlsSigner>` | TLS key behind its own narrow signer handle |
| `node.StakingSigner` (`bls.Signer`) | `Arc<dyn bls::Signer>` | one instance, cloned to all consumers |
| `node.newStakingSigner` | `ava_node::new_staking_signer` | the 4-way precedence (§5.2) |
| `config.getStakingTLSCert` | `ava_config::load_staking_tls` | the 4-way precedence (§5.1) |
| `Signer.Shutdown` | `Signer::shutdown` | closes the gRPC channel for `RpcSigner` |

### 7.1 Error model

`ava-crypto` defines (mirroring Go sentinels, asserted with `matches!` per `00`
§7.1):

- `Error::FailedSecretKeyDeserialize` (`ErrFailedSecretKeyDeserialize`).
- `Error::InvalidProofOfPossession` (`ErrInvalidProofOfPossession`).
- TLS parse errors carry through from `03` §3.6 (`CertificateTooLarge`,
  `UnsupportedRsaModulusBitLen`, …).

`ava-config` defines `StakingKeyContentUnset` / `StakingCertContentUnset` /
`InvalidSignerConfig` (the one-of-four guard). The remote-signer connect path
surfaces a `RpcSigner` error wrapping the tonic/transport status.

---

## 8. Test plan (cross-ref `02`)

1. **NodeID-from-cert golden vectors.** Feed Go-generated staking certs (`tests/`
   fixtures from the Go tree); assert `node_id_from_cert` matches the Go NodeID
   bit-for-bit (`03` §6/§9, `22`-style golden corpus). Include the `large_rsa_key`
   reject case (`staking/large_rsa_key.cert`).
2. **PoP golden vectors.** Given a fixed 32-byte BLS key, assert `(pk, pop)` equals
   the Go `ProofOfPossession` bytes and that the `HexNC` JSON matches. Round-trip
   `MarshalJSON`/`UnmarshalJSON`. Cross-check `verify()`.
3. **Signed-IP cross-sign.** Sign an `UnsignedIp` with a Rust node's keys; assert a
   Go node's `SignedIP.Verify` accepts the TLS signature *and* the BLS PoP, and
   vice-versa (differential, `02`/`05`).
4. **Key-file format round-trip.** A `signer.key` written by Go loads in Rust and
   produces the same public key; a Rust-written `signer.key` loads in Go.
   Likewise the PEM `staker.key`/`staker.crt`.
5. **Precedence tables.** `rstest` tables over the §5.1/§5.2 matrices (each branch
   incl. the mutual-exclusion errors), mirroring `config/config_test.go`'s
   `expectedSignerConfig` cases. Assert the one-of-four guard
   (`errInvalidSignerConfig`) and the both-or-neither TLS-content guard.
6. **Remote-signer interop (both directions).** (a) A Rust node configured with
   `--staking-rpc-signer-endpoint` signs warp/PoP via a **Go** `proto/signer`
   server; verify the resulting signatures. (b) A **Rust** `proto/signer` server
   (wrapping `LocalSigner`) services a **Go** `rpcsigner.Client`. Run under the
   differential harness (`02`). Repeat for `warp.Signer` (plugin↔host) under the
   rpcchainvm interop suite (`07`).
7. **Generate-when-absent.** Fresh `t.TempDir()` data dir: assert the three files
   are created with mode `0o400` (dir `0o700`), a stable NodeID/PoP results, and a
   second boot reuses them unchanged.
8. **Zeroization.** Assert (via a test hook / `Drop` instrumentation) that
   `LocalSigner` zeroizes its secret on drop and that key-gen IKM does not linger.

---

## 9. Performance notes / improvements over Go

- **Signer-result caching.** The BLS public key is derived once at construction and
  cached (as in Go). The `IpSigner` already caches the `SignedIp` (`arc-swap`,
  `05`); we extend nothing beyond Go's behavior on the consensus path.
- **No per-signature allocation on the warp hot path.** `sign`/`sign_pop` borrow
  `&[u8]`; the local impl writes into a reused output buffer where the caller can
  provide one. Compressed-signature output is a fixed `[u8; 96]` (no heap).
- **Remote-signer pipelining.** tonic's HTTP/2 multiplexing lets the warp
  aggregator issue concurrent `Sign` calls to a remote signer without head-of-line
  blocking — an improvement over Go's `context.TODO()` one-at-a-time client, **safe
  because** each signature is independent and order-free (the aggregator reassembles
  by validator index, `20`). Gated on the §8.6 interop test.
- **Lazy connect.** `RpcSigner` uses `connect_lazy` so node startup is not blocked
  on signer reachability beyond the initial `PublicKey()` probe (Go uses a 5s
  connect timeout for the probe; we keep that bound on the probe call).
- All signer work that is CPU-bound (local `blst`) stays off the async reactor only
  if it ever became non-trivial; current sign cost is sub-millisecond, so it runs
  inline (no `spawn_blocking`), matching Go.
