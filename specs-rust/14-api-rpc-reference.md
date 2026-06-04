# 14 — API / RPC Endpoint Reference (Exhaustive Catalog)

> **Status:** Conforms to `00-overview-and-conventions.md` (binding) and to the
> `ava-api` design in `12-node-config-api-wallet.md`. This file is the
> **authoritative endpoint catalog**: it enumerates every exposed JSON-RPC method,
> Connect/gRPC RPC, and plain-HTTP endpoint, with exact method names, base paths,
> request (`Args`) fields, and response (`Reply`) fields extracted verbatim from
> the Go source. Specs `08` (P-Chain) and `09` (X-Chain) give summary method lists;
> where they differ, **this file wins**.

## Go source paths covered

| Go area | Section |
|---|---|
| `api/server/{server.go,router.go,allowed_hosts.go,metrics.go}` | §1 routing |
| `api/info/service.go` | §3 Info |
| `api/admin/service.go` | §4 Admin |
| `api/health/{service.go,handler.go,result.go}` | §5 Health |
| `api/metrics` (`promhttp`) | §6 Metrics |
| `indexer/{service.go,indexer.go,index.go}` | §7 Index |
| `vms/platformvm/service.go` + `vms/platformvm/api.go` + `api/common_args_responses.go` | §8 P-Chain |
| `vms/avm/service.go` | §9 X-Chain |
| `graft/coreth/eth/backend.go`, `internal/ethapi/{api.go,api.coreth.go,api_extra.go}`, `warp/service.go`, `plugin/evm/{vm.go,admin.go}`, `plugin/evm/atomic/vm/{vm.go,api.go}` | §10 C-Chain (EVM) |
| `vms/proposervm/service.go` + `connectproto/proposervm/service.proto` | §11 ProposerVM |
| `connectproto/xsvm/service.proto` | §11 xsvm Connect |
| `api/health/handler.go` GET branch; EVM `/ws` | §12 pub-sub / WS |

> **Keystore:** the historical `keystore.*` API (`/ext/keystore`) has been
> **removed** from this branch — there is no `api/keystore` package and `node.go`
> registers no keystore route. The Rust port therefore does **not** implement it.
> If a fixed Go release that still ships keystore must be matched, treat it as an
> optional, feature-gated shim; it is out of the default compatibility surface.

---

## 1. Wire format, base paths, transport

### 1.1 JSON-RPC 2.0 wire shape (`gorilla/rpc/v2` + `json` v1 codec)

All non-EVM avalanchego services use `gorilla/rpc/v2` with the **v1 `json`** codec
(not "json2"). The wire shape the Rust shim must reproduce exactly (see `12` §3.2):

**Request**
```json
{ "jsonrpc": "2.0", "id": 1, "method": "platform.getHeight", "params": [ { /* Args */ } ] }
```
- `method` = `"<service>.<Method>"` where `<Method>` is Go's exported **PascalCase**
  name (`info.GetNodeID`, `platform.GetHeight`, `health.Health`). Clients commonly
  lowercase the first letter (`platform.getHeight`); gorilla matching is
  case-insensitive on the method segment, so both are accepted.
- `params` is a **single-element array** wrapping the `Args` object (gorilla v1
  codec convention). Methods whose Go signature takes `*struct{}` accept `params:
  [{}]`, `params: []`, or an absent `params`.

**Success response**
```json
{ "jsonrpc": "2.0", "id": 1, "result": { /* Reply */ } }
```

**Error response**
```json
{ "jsonrpc": "2.0", "id": 1, "error": { "code": -32000, "message": "...", "data": null } }
```
JSON-RPC error codes (`12` §11): parse `-32700`, invalid request `-32600`, method
not found `-32601`, invalid params `-32602`, internal `-32603`; service-returned
`error` values map to a generic server error (gorilla uses `-32000`).

**Content-Type:** `application/json` or `application/json;charset=UTF-8` (both
codecs registered on every service). POST only for JSON-RPC.

### 1.2 Base-path scheme

Base URL constant `baseURL = "/ext"` (`api/server/server.go:32`). Routes:

| Path | Service prefix | Handler kind | Default enabled |
|---|---|---|---|
| `/ext/info` | `info.` | JSON-RPC | yes (`api-info-enabled`) |
| `/ext/admin` | `admin.` | JSON-RPC | **no** (`api-admin-enabled`) |
| `/ext/health` | `health.` | GET **or** JSON-RPC POST | yes (`api-health-enabled`) |
| `/ext/health/liveness` | — | GET-only | yes |
| `/ext/health/readiness` | — | GET-only | yes |
| `/ext/health/health` | — | GET-only | yes |
| `/ext/metrics` | — | plain HTTP (Prometheus text) | yes (`api-metrics-enabled`) |
| `/ext/bc/<chainID>/<ext>` | per-VM (`platform.`, `avm.`, …) | JSON-RPC | per chain |
| `/ext/bc/C/rpc` | EVM (`eth_`, `debug_`, …) | go-ethereum JSON-RPC | C-Chain |
| `/ext/bc/C/ws` | EVM | WebSocket JSON-RPC | C-Chain |
| `/ext/bc/C/avax` | `avax.` | JSON-RPC (atomic) | C-Chain |
| `/ext/bc/C/admin` | `admin.` (coreth) | JSON-RPC | if coreth admin enabled |
| `/ext/bc/<chainID>/proposervm` | `proposervm.` | JSON-RPC | proposervm-wrapped chains |
| `/ext/index/<alias>/{block,tx,vtx}` | `index.` | JSON-RPC | **no** (`index-enabled`) |

Chain mounting (`server.go:153 RegisterChain`): `vm.CreateHandlers()` returns
`map[extension]http.Handler`; each is mounted at `/ext/bc/<chainID>/<extension>`
(empty extension `""` ⇒ `/ext/bc/<chainID>`). `vm.NewHTTPHandler()` returns the
**header-route** handler (used by EVM `/rpc`,`/ws` and proposervm Connect),
dispatched via the `X-Avalanche-Vm-Route` header (`HTTPHeaderRoute`). Chain
**aliases** `P`,`X`,`C` are registered so `/ext/bc/P`, `/ext/bc/X`, `/ext/bc/C`
resolve to the primary-network chain IDs.

### 1.3 Transport, CORS, allowed-hosts, auth

- **HTTP server** (`api/server`): `gorilla/mux` + `h2c.NewHandler`
  (HTTP/2 cleartext, `MaxConcurrentStreams=64`) on `http-host:http-port`
  (default `127.0.0.1:9650`). TLS optional (`http-tls-enabled`).
- **CORS** (`rs/cors`): `allow_origins` = `http-allowed-origins` (default `*`),
  `allow_credentials = true`.
- **Allowed-Hosts** (`allowed_hosts.go::filterInvalidHosts`): rejects with `403`
  any request whose `Host` header is not in `http-allowed-hosts` (default
  `["localhost"]`); `*` accepts all; bare-IP and empty hosts always accepted.
- **`node-id` response header** set on every response.
- **Per-chain reject middleware:** until a chain reaches `NormalOp`, its routes
  return `503 "API call rejected because chain is not done bootstrapping"`.
- **Auth token:** the historical API-token middleware is removed; access control is
  CORS + allowed-hosts + loopback binding only (`12` §3.9).

The Rust mapping is in §13.

---

## 2. Common Args/Reply types (`api/common_args_responses.go`)

These shapes are shared across P-Chain, X-Chain, and the EVM `avax` namespace.
(`avajson.UintN`/`Uint64` serialize as **decimal strings** in JSON; `ids.ID`,
`NodeID` as CB58/`NodeID-…`; addresses as bech32.)

| Type | Fields (json tag : Go type) |
|---|---|
| `EmptyReply` | `{}` |
| `JSONTxID` | `txID: ids.ID` |
| `JSONAddress` | `address: string` |
| `JSONAddresses` | `addresses: []string` |
| `GetBlockArgs` | `blockID: ids.ID`, `encoding: formatting.Encoding` |
| `GetBlockByHeightArgs` | `height: Uint64`, `encoding: Encoding` |
| `GetBlockResponse` | `block: json.RawMessage`, `encoding: Encoding` |
| `GetHeightResponse` | `height: Uint64` |
| `FormattedBlock` | `block: string`, `encoding: Encoding` |
| `GetTxArgs` | `txID: ids.ID`, `encoding: Encoding` |
| `GetTxReply` | `tx: json.RawMessage`, `encoding: Encoding` |
| `FormattedTx` | `tx: string`, `encoding: Encoding` |
| `Index` | `address: string`, `utxo: string` |
| `GetUTXOsArgs` | `addresses: []string`, `sourceChain: string`, `limit: Uint32`, `startIndex: Index`, `encoding: Encoding` |
| `GetUTXOsReply` | `numFetched: Uint64`, `utxos: []string`, `endIndex: Index`, `encoding: Encoding` |

`formatting.Encoding` JSON values: `"hex"` (default), `"cb58"` (deprecated for
some), `"json"` (blocks/txs only).

---

## 3. Info API — `/ext/info`, prefix `info.` (`api/info/service.go`)

11 methods. All `Args` of `*struct{}` take no params.

| Method | Params (Args) | Returns (Reply) | Notes |
|---|---|---|---|
| `info.getNodeVersion` | — | `version: string`, `databaseVersion: string`, `rpcProtocolVersion: Uint32`, `gitCommit: string`, `vmVersions: map[string]string` | |
| `info.getNodeID` | — | `nodeID: NodeID`, `nodePOP: {publicKey: hex, proofOfPossession: hex}` | BLS PoP |
| `info.getNodeIP` | — | `ip: netip.AddrPort` (string `host:port`) | |
| `info.getNetworkID` | — | `networkID: Uint32` | |
| `info.getNetworkName` | — | `networkName: string` | `mainnet`/`fuji`/… |
| `info.getBlockchainID` | `alias: string` | `blockchainID: ids.ID` | resolves chain alias |
| `info.peers` | `nodeIDs: []NodeID` | `numPeers: Uint64`, `peers: []Peer` | `Peer` = `peer.Info{ip,publicIP,nodeID,version,lastSent,lastReceived,observedUptime,trackedSubnets,...}` + `benched: []string`. Empty `nodeIDs` ⇒ all peers |
| `info.isBootstrapped` | `chain: string` | `isBootstrapped: bool` | err if chain unknown |
| `info.upgrades` | — | `upgrade.Config` (all fork activation times: `apricotPhase1Time`…`etnaTime`,`fortunaTime`,`graniteTime`,…) | returns the whole upgrade schedule |
| `info.uptime` | — | `rewardingStakePercentage: Float64`, `weightedAveragePercentage: Float64` | |
| `info.acps` | — | `acps: map[uint32]{supportWeight: Uint64, supporters: set[NodeID], objectWeight: Uint64, objectors: set[NodeID], abstainWeight: Uint64}` | ACP support tally |
| `info.getTxFee` | — | `txFee, createAssetTxFee, createSubnetTxFee, transformSubnetTxFee, createBlockchainTxFee, addPrimaryNetworkValidatorFee, addPrimaryNetworkDelegatorFee, addSubnetValidatorFee, addSubnetDelegatorFee` (all `Uint64`) | **deprecated** (logs warn); static per-network |
| `info.getVMs` | — | `vms: map[ids.ID][]string`, `fxs: map[ids.ID]string` | registered VMs + fx (`secp256k1fx`,`nftfx`,`propertyfx`) |

---

## 4. Admin API — `/ext/admin`, prefix `admin.` (`api/admin/service.go`)

Disabled by default. 14 methods.

| Method | Params (Args) | Returns (Reply) | Notes |
|---|---|---|---|
| `admin.startCPUProfiler` | — | `EmptyReply` | writes pprof to `profile-dir` |
| `admin.stopCPUProfiler` | — | `EmptyReply` | |
| `admin.memoryProfile` | — | `EmptyReply` | |
| `admin.lockProfile` | — | `EmptyReply` | mutex profile |
| `admin.alias` | `endpoint: string`, `alias: string` | `EmptyReply` | alias an HTTP endpoint; `len(alias) ≤ 512` |
| `admin.aliasChain` | `chain: string`, `alias: string` | `EmptyReply` | alias a chain; updates `/ext/bc/<alias>` |
| `admin.getChainAliases` | `chain: string` | `aliases: []string` | `chain` parsed as ids.ID |
| `admin.stacktrace` | — | `EmptyReply` | writes `stacktrace.txt` (all goroutines) |
| `admin.setLoggerLevel` | `loggerName: string`, `logLevel: *Level`, `displayLevel: *Level` | `loggerLevels: map[string]{logLevel,displayLevel}` | empty name ⇒ all loggers; at least one level required |
| `admin.getLoggerLevel` | `loggerName: string` | `loggerLevels: map[string]{logLevel,displayLevel}` | empty name ⇒ all |
| `admin.getConfig` | — | resolved node config (interface{}) as JSON | |
| `admin.loadVMs` | — | `newVMs: map[ids.ID][]string`, `failedVMs: map[ids.ID]string` (omitempty) | rescan `plugin-dir` |
| `admin.dbGet` | `key: string` (HexNC) | `value: string` (HexNC), `errorCode: rpcdbpb.Error` | raw DB read |

`Level` values: `Verbo`,`Debug`,`Trace`,`Info`,`Warn`,`Error`,`Fatal`,`Off`.

---

## 5. Health API — `/ext/health` (`api/health/{service.go,handler.go}`)

**Dual handler** (`handler.go`): a **GET** request runs the GET handler; any other
method falls through to the JSON-RPC server.

GET endpoints (plain HTTP, query param `?tag=<subsystem>` repeatable):

| Path | Reports | HTTP status |
|---|---|---|
| `GET /ext/health` | overall `Health` | `200` healthy / `503` unhealthy |
| `GET /ext/health/health` | `Health` | 200/503 |
| `GET /ext/health/readiness` | `Readiness` (init complete) | 200/503 |
| `GET /ext/health/liveness` | `Liveness` (needs restart) | 200/503 |

Body (all): `{ "checks": map[string]Result, "healthy": bool }`. `Result` =
`{message: json, error: json, timestamp: time.Time, duration: int64, contiguousFailures: int64, timeOfFirstFailure: *time.Time}`.

JSON-RPC POST on `/ext/health`, prefix `health.`:

| Method | Params (`APIArgs`) | Returns (`APIReply`) |
|---|---|---|
| `health.health` | `tags: []string` | `checks: map[string]Result`, `healthy: bool` |
| `health.readiness` | `tags: []string` | `checks`, `healthy` |
| `health.liveness` | `tags: []string` | `checks`, `healthy` |

---

## 6. Metrics — `/ext/metrics` (`api/metrics`, `promhttp`)

Plain `GET`. Prometheus text exposition format from the node's `MultiGatherer`
(prefix + label gatherers, plus the Go process/`GoCollector`). No JSON-RPC. Metric
names/labels must match Go (golden test, `02`). Rust: `prometheus` crate
`TextEncoder` (`12` §3.6).

---

## 7. Index API — `/ext/index/<alias>/{block,tx,vtx}`, prefix `index.` (`indexer/service.go`)

Disabled unless `index-enabled`. Per primary-network chain, up to three index
endpoints are mounted (`indexer.go`: prefixes `tx=0x01`, `vtx=0x02`,
`block=0x03`). Known mounts: `/ext/index/X/tx`, `/ext/index/X/block`,
`/ext/index/C/block`, `/ext/index/P/block`, `/ext/index/X/vtx` (legacy Avalanche
vertices). 6 methods, all returning `FormattedContainer` or wrappers.

`FormattedContainer` = `{id: ids.ID, bytes: string, timestamp: time.Time, encoding: Encoding, index: Uint64}`.

| Method | Params | Returns | Notes |
|---|---|---|---|
| `index.getLastAccepted` | `encoding: Encoding` | `FormattedContainer` | most recently accepted container |
| `index.getContainerByIndex` | `index: Uint64`, `encoding: Encoding` | `FormattedContainer` | |
| `index.getContainerByID` | `id: ids.ID`, `encoding: Encoding` | `FormattedContainer` | |
| `index.getContainerRange` | `startIndex: Uint64`, `numToFetch: Uint64`, `encoding: Encoding` | `containers: []FormattedContainer` | capped at `MaxFetchedByRange (1024)` |
| `index.getIndex` | `id: ids.ID` | `index: Uint64` | |
| `index.isAccepted` | `id: ids.ID` | `isAccepted: bool` | |

---

## 8. P-Chain — `/ext/bc/P`, prefix `platform.` (`vms/platformvm/service.go`)

31 exported methods. `Height` (in `platformapi`) is a union accepting `"proposed"`,
a numeric height, or omitted (latest). Validator entries are `[]any` polymorphic
objects (primary/subnet/L1).

| Method | Params (Args) | Returns (Reply) | Notes |
|---|---|---|---|
| `platform.getHeight` | — | `height: Uint64` | last-accepted P-Chain height |
| `platform.getProposedHeight` | — | `height: Uint64` | height that would be proposed now |
| `platform.getBalance` | `addresses: []string` | `balance, unlocked, lockedStakeable, lockedNotStakeable: Uint64`, `balances, unlockeds, lockedStakeables, lockedNotStakeables: map[ids.ID]Uint64`, `utxoIDs: []UTXOID` | |
| `platform.getUTXOs` | `GetUTXOsArgs` (§2) | `GetUTXOsReply` | paginated; cross-chain via `sourceChain` |
| `platform.getSubnet` | `subnetID: ids.ID` | `isPermissioned: bool`, `controlKeys: []string`, `threshold: Uint32`, `locktime: Uint64`, `subnetTransformationTxID: ids.ID`, `conversionID: ids.ID`, `managerChainID: ids.ID`, `managerAddress: JSONByteSlice` | |
| `platform.getSubnets` | `ids: []ids.ID` | `subnets: []APISubnet{id, controlKeys: []string, threshold: Uint32}` | empty `ids` ⇒ all |
| `platform.getStakingAssetID` | `subnetID: ids.ID` | `assetID: ids.ID` | |
| `platform.getCurrentValidators` | `subnetID: ids.ID`, `nodeIDs: []NodeID` | `validators: []any` | primary/subnet/L1 validator objects |
| `platform.getL1Validator` | `validationID: ids.ID` | `subnetID: ids.ID`, `nodeID, publicKey, remainingBalanceOwner, deactivationOwner, startTime, weight, minNonce, balance`, `height: Uint64` | ACP-77 L1 validator |
| `platform.getCurrentSupply` | `subnetID: ids.ID` | `supply: Uint64`, `height: Uint64` | |
| `platform.sampleValidators` | `size: Uint16`, `subnetID: ids.ID` | `validators: []NodeID` | |
| `platform.getBlockchainStatus` | `blockchainID: string` | `status: BlockchainStatus` (`Validating`/`Created`/`Preferred`/`Syncing`/`UnknownChain`) | |
| `platform.validatedBy` | `blockchainID: ids.ID` | `subnetID: ids.ID` | |
| `platform.validates` | `subnetID: ids.ID` | `blockchainIDs: []ids.ID` | |
| `platform.getBlockchains` | — | `blockchains: []APIBlockchain{id, name, subnetID, vmID}` | |
| `platform.issueTx` | `FormattedTx` (§2) | `txID: ids.ID` | submit signed tx |
| `platform.getTx` | `GetTxArgs` (§2) | `GetTxReply` (`tx`, `encoding`) | hex or JSON |
| `platform.getTxStatus` | `txID: ids.ID` | `status: status.Status` (`Committed`/`Aborted`/`Processing`/`Unknown`/`Dropped`), `reason: string` (omitempty) | |
| `platform.getStake` | `addresses: []string`, `validatorsOnly: bool`, `encoding: Encoding` | `staked: Uint64`, `stakeds: map[ids.ID]Uint64`, `stakedOutputs: []string`, `encoding: Encoding` | |
| `platform.getMinStake` | `subnetID: ids.ID` | `minValidatorStake: Uint64`, `minDelegatorStake: Uint64` | |
| `platform.getTotalStake` | `subnetID: ids.ID` | `stake: Uint64`, `weight: Uint64` | |
| `platform.getRewardUTXOs` | `GetTxArgs` (`txID`,`encoding`) | `numFetched: Uint64`, `utxos: []string`, `encoding: Encoding` | |
| `platform.getTimestamp` | — | `timestamp: time.Time` | |
| `platform.getValidatorsAt` | `height: Height`, `subnetID: ids.ID` | `validators: map[NodeID]{publicKey: *string, weight: Uint64}` | |
| `platform.getAllValidatorsAt` | `height: Height` | `validatorSets: map[ids.ID]validators.WarpSet` | all subnets |
| `platform.getBlock` | `GetBlockArgs` (§2) | `GetBlockResponse` | hex or JSON block |
| `platform.getBlockByHeight` | `GetBlockByHeightArgs` (§2) | `GetBlockResponse` | |
| `platform.getFeeConfig` | — | `gas.Config` (dynamic-fee params) | |
| `platform.getFeeState` | — | `gas.State` (capacity, excess) + `price: gas.Price`, `timestamp: time.Time` | |
| `platform.getValidatorFeeConfig` | — | `fee.Config` (validator-fee params) | |
| `platform.getValidatorFeeState` | — | `excess: gas.Gas`, `price: gas.Price`, `timestamp: time.Time` | |

> Removed legacy wallet methods (key import/export, createAddress, send*, add*Tx
> via keystore) are **not present** in this service — tx creation is client-side
> (`ava-wallet`, `12` §13). Do not invent them.

---

## 9. X-Chain — `/ext/bc/X`, prefix `avm.` (`vms/avm/service.go`)

11 exported methods.

| Method | Params (Args) | Returns (Reply) | Notes |
|---|---|---|---|
| `avm.getBlock` | `GetBlockArgs` (§2) | `GetBlockResponse` | |
| `avm.getBlockByHeight` | `GetBlockByHeightArgs` (§2) | `GetBlockResponse` | |
| `avm.getHeight` | — | `height: Uint64` | |
| `avm.issueTx` | `FormattedTx` (§2) | `txID: ids.ID` | parse + mempool + gossip |
| `avm.getTxStatus` | `txID: ids.ID` (`JSONTxID`) | `status: choices.Status` | **deprecated** (only `Accepted`/`Unknown`); use `getTx` |
| `avm.getTx` | `GetTxArgs` (§2) | `GetTxReply` | |
| `avm.getUTXOs` | `GetUTXOsArgs` (§2) | `GetUTXOsReply` | cross-chain via `sourceChain` |
| `avm.getAssetDescription` | `assetID: string` | `assetID: ids.ID`, `name: string`, `symbol: string`, `denomination: Uint8` | |
| `avm.getBalance` | `address: string`, `assetID: string`, `includePartial: bool` | `balance: Uint64`, `utxoIDs: []UTXOID` | |
| `avm.getAllBalances` | `address: string`, `includePartial: bool` | `balances: []Balance{assetID: ids.ID, balance: Uint64}` | |
| `avm.getTxFee` | — | `txFee: Uint64`, `createAssetTxFee: Uint64` | |

X-Chain `nftfx`/`propertyfx` add no JSON-RPC methods — they are fx type registrations only.

---

## 10. C-Chain (EVM) — go-ethereum RPC at `/ext/bc/C/{rpc,ws}` + avalanche extras

The C-Chain uses **go-ethereum's `rpc` server** (graft/coreth), not gorilla. Wire
format = standard Ethereum JSON-RPC 2.0 (`{"jsonrpc":"2.0","id":N,"method":"eth_blockNumber","params":[...]}`),
method names `<namespace>_<method>` (snake_case after the underscore). Mounts:

- `/ext/bc/C/rpc` — HTTP JSON-RPC. Header-route handler (via `NewHTTPHandler`).
- `/ext/bc/C/ws` — WebSocket JSON-RPC (supports `eth_subscribe`/`eth_unsubscribe`).
- `/ext/bc/C/avax` — the **`avax.*`** atomic API (gorilla JSON-RPC).
- `/ext/bc/C/admin` — coreth `admin.*` (gorilla JSON-RPC), if enabled.

Enabled namespaces (`eth/backend.go` + `plugin/evm/vm.go`): **`eth`**, **`debug`**,
**`net`**, plus **`warp`** (if `WarpAPIEnabled`), **`web3`**, **`txpool`**,
**`personal`** (subject to `eth-apis` config). `admin` (geth) namespace is the
coreth node-admin, separate from the `/avax` and `/admin` gorilla handlers.

### 10.1 `eth` namespace (selected; **★ = Avalanche-modified/added**)

Standard go-ethereum methods (full set per geth): `eth_blockNumber`,
`eth_getBalance`, `eth_getProof`, `eth_getCode`, `eth_getStorageAt`,
`eth_getBlockByNumber`, `eth_getBlockByHash`, `eth_getBlockReceipts`,
`eth_getHeaderByNumber`, `eth_getHeaderByHash`, `eth_getUncleByBlockNumberAndIndex`,
`eth_getUncleByBlockHashAndIndex`, `eth_getUncleCountByBlockNumber`,
`eth_getUncleCountByBlockHash`, `eth_call`, `eth_estimateGas`,
`eth_createAccessList`, `eth_chainId`, `eth_gasPrice`, `eth_maxPriorityFeePerGas`,
`eth_feeHistory`, `eth_syncing`, `eth_getBlockTransactionCountByNumber/Hash`,
`eth_getTransactionByBlockNumberAndIndex`, `eth_getTransactionByBlockHashAndIndex`,
`eth_getRawTransactionByBlock*Index`, `eth_getTransactionCount`,
`eth_getTransactionByHash`, `eth_getRawTransactionByHash`,
`eth_getTransactionReceipt`, `eth_sendTransaction`, `eth_sendRawTransaction`,
`eth_fillTransaction`, `eth_sign`, `eth_signTransaction`, `eth_pendingTransactions`,
`eth_resend`. Filters (`eth-filter`): `eth_newFilter`, `eth_newBlockFilter`,
`eth_newPendingTransactionFilter`, `eth_getFilterChanges`, `eth_getFilterLogs`,
`eth_getLogs`, `eth_uninstallFilter`, `eth_subscribe`, `eth_unsubscribe` (WS).

Avalanche-specific / divergent:
- **★ `eth_baseFee`** (`EthereumAPI.BaseFee`, `api.go:92`) — current base fee from
  the ACP-176 dynamic-fee mechanism. Not in stock geth.
- **★ `eth_suggestPriceOptions`** (`api.coreth.go:46`) — returns
  `PriceOptions{slow, normal, fast: *hexutil.Big}` derived from `EstimateBaseFee` +
  tip; coreth-only fee UX helper.
- **★ `eth_getChainConfig`** (`api_extra.go:28`, `BlockChainAPI.GetChainConfig`) —
  returns the full coreth `params.ChainConfig` (incl. the Avalanche `extras`:
  fee config, network upgrades). Not in stock geth.
- **★ `eth_gasPrice` / `eth_maxPriorityFeePerGas` / `eth_feeHistory`** — same names
  as geth but values come from the **ACP-176** mechanism, not EIP-1559 London
  defaults. Semantics diverge.
- **Accepted-block semantics:** geth's `latest` block tag maps to the
  **last-accepted** (final) block, not a tentative head; coreth adds the
  `accepted`/`pending` handling consistent with Snowman finality. There are no
  reorgs, so `finalized`/`safe` track `accepted`.

### 10.2 `debug` namespace

`debug_getRawHeader`, `debug_getRawBlock`, `debug_getRawReceipts`,
`debug_getRawTransaction`, `debug_printBlock`, plus the **tracer** methods
(`eth/tracers/api.go`, registered as `debug`): `debug_traceTransaction`,
`debug_traceBlockByNumber`, `debug_traceBlockByHash`, `debug_traceBlock`,
`debug_traceCall`, `debug_traceChain`, `debug_standardTraceBlockToFile`,
`debug_intermediateRoots`, etc. Standard geth set; tracers operate over accepted
state.

### 10.3 `net`, `web3`, `txpool`, `personal`

- `net`: `net_listening`, `net_peerCount`, `net_version`.
- `web3`: `web3_clientVersion`, `web3_sha3` (geth standard).
- `txpool`: `txpool_content`, `txpool_contentFrom`, `txpool_status`,
  `txpool_inspect`.
- `personal` (deprecated, off by default): `personal_listAccounts`,
  `personal_listWallets`, `personal_openWallet`, `personal_deriveAccount`,
  `personal_newAccount`, `personal_importRawKey`, `personal_unlockAccount`,
  `personal_lockAccount`, `personal_sendTransaction`, `personal_signTransaction`,
  `personal_sign`, `personal_ecRecover`, `personal_initializeWallet`,
  `personal_unpair`.

### 10.4 `warp` namespace (★ Avalanche, `warp/service.go`)

ICM / Warp messaging. Off unless `WarpAPIEnabled`.
- `warp_getMessage(messageID: ids.ID) → hexutil.Bytes`
- `warp_getMessageSignature(messageID: ids.ID) → hexutil.Bytes`
- `warp_getBlockSignature(blockID: ids.ID) → hexutil.Bytes`
- `warp_getMessageAggregateSignature(messageID, quorumNum: uint64, subnetID: string) → hexutil.Bytes`
- `warp_getBlockAggregateSignature(blockID, quorumNum: uint64, subnetID: string) → hexutil.Bytes`

### 10.5 `avax.*` atomic API — `/ext/bc/C/avax` (★ Avalanche, gorilla JSON-RPC)

`plugin/evm/atomic/vm/api.go`, service name **`avax`**:

| Method | Params | Returns |
|---|---|---|
| `avax.getUTXOs` | `GetUTXOsArgs` (§2) | `GetUTXOsReply` |
| `avax.issueTx` | `FormattedTx` (§2) | `txID: ids.ID` (`JSONTxID`) |
| `avax.getAtomicTxStatus` | `txID: ids.ID` (`JSONTxID`) | `GetAtomicTxStatusReply{status: atomic.Status, blockHeight: *Uint64}` |
| `avax.getAtomicTx` | `GetTxArgs` (§2) | `FormattedTx` (atomic tx) |

These handle C↔X / C↔P atomic import/export UTXOs (the ATOMIC-1 byte contract,
`00` §11.1.7).

### 10.6 coreth `admin.*` — `/ext/bc/C/admin` (gorilla JSON-RPC, `plugin/evm/admin.go`)

`admin.startCPUProfiler`, `admin.stopCPUProfiler`, `admin.memoryProfile`,
`admin.lockProfile`, `admin.setLogLevel(level: string)`,
`admin.getVMConfig → {config: config.Config}`. (Distinct from the geth `admin`
namespace and from the node-level `/ext/admin`.)

> **Reth note (`10`):** `ava-evm` reproduces all of §10.1–§10.3 via reth's
> `reth-rpc` (standard namespaces) plus custom modules for `eth_baseFee`,
> `eth_suggestPriceOptions`, `eth_getChainConfig`, the ACP-176 fee semantics, the
> `warp` and `avax` namespaces, and the accepted-block tag mapping.

---

## 11. ProposerVM & Connect/gRPC endpoints

### 11.1 ProposerVM (`vms/proposervm/service.go`, `vm.go`)

The proposervm wrapper exposes **two** transports on every wrapped chain:

**JSON-RPC** at `/ext/bc/<chainID>/proposervm` (httpPathEndpoint `/proposervm`),
service name `proposervm`:

| Method | Params | Returns |
|---|---|---|
| `proposervm.getProposedHeight` | — | `height: Uint64` (`GetHeightResponse`) |
| `proposervm.getCurrentEpoch` | — | `number: Uint64`, `startTime: Uint64`, `pChainHeight: Uint64` |

**Connect (connectrpc)** via the header-route handler (`HTTPHeaderRoute =
"proposervm"`), service `proposervm.ProposerVM` (`connectproto/proposervm/service.proto`):

| RPC | Request | Reply |
|---|---|---|
| `ProposerVM.GetProposedHeight` | `GetProposedHeightRequest{}` | `GetProposedHeightReply{height: uint64}` |
| `ProposerVM.GetCurrentEpoch` | `GetCurrentEpochRequest{}` | `GetCurrentEpochReply{number: uint64, p_chain_height: uint64, start_time: int64}` |

Connect requests are routed by the `X-Avalanche-Vm-Route` header on the chain's
header-route mount (the same h2c port). This is the open-risk **R5** Connect
enumeration item from `00` §11.2 — resolved here for proposervm.

### 11.2 xsvm Connect Ping (`connectproto/xsvm/service.proto`)

Test/example VM `xsvm` exposes a Connect `Ping` service (used in load/e2e tests,
not on primary-network nodes):

| RPC | Request | Reply | Kind |
|---|---|---|---|
| `Ping.Ping` | `PingRequest{message: string}` | `PingReply{message: string}` | unary |
| `Ping.StreamPing` | stream `StreamPingRequest{message}` | stream `StreamPingReply{message}` | bidi stream |

Served over the same h2c port via the chain's header-route handler. Documented for
completeness; the Rust port wires it only behind the xsvm test crate.

> No other `connectproto/` services are exposed by a default node. The
> `rpcchainvm`/plugin gRPC services (`vm.proto`, `runtime.proto`, rpcdb, appsender,
> sharedmemory, validatorstate, warp) are the **plugin protocol** (spec `07`),
> not public node APIs, and are not catalogued here.

---

## 12. Pub-sub / WebSocket

- **EVM `/ext/bc/C/ws`** — full Ethereum `eth_subscribe` over WebSocket:
  `newHeads`, `logs`, `newPendingTransactions`, `syncing`. Same JSON-RPC method
  names as geth. Rust: `tokio-tungstenite` upgrade on the header-route handler
  (`12` §3.8), reth's pub-sub for subscriptions.
- **Health GET** is the load-balancer/k8s probe surface (§5), not a stream.
- There is **no** X-Chain/P-Chain `pubsub` (bloom-filter address feed) package on
  this branch — the legacy `/ext/bc/X/events` filtered feed has been removed. The
  Rust port does not implement it (a `tokio::sync::broadcast` fan-out shim is kept
  feature-gated should a matched Go release require it; `12` §3.8).

---

## 13. Rust `ava-api` mapping & `register_chain` contract

| Surface | Base path | Rust crate / handler |
|---|---|---|
| `info.*` | `/ext/info` | `ava-api::info` (JSON-RPC shim, `#[rpc_service("info")]`) |
| `admin.*` | `/ext/admin` | `ava-api::admin` |
| `health.*` + GET probes | `/ext/health[/…]` | `ava-api::health` (axum method-branching handler) |
| metrics | `/ext/metrics` | `ava-api::metrics` (`prometheus` TextEncoder) |
| `index.*` | `/ext/index/<alias>/{block,tx,vtx}` | `ava-indexer::service` |
| `platform.*` | `/ext/bc/P` | `ava-platformvm::service` (mounted via `register_chain`) |
| `avm.*` | `/ext/bc/X` | `ava-avm::service` |
| `eth.*`/`debug.*`/`net.*`/`web3.*`/`txpool.*`/`warp.*` | `/ext/bc/C/rpc`,`/ws` | `ava-evm` (reth `reth-rpc` + custom modules) |
| `avax.*` (atomic) | `/ext/bc/C/avax` | `ava-evm::atomic_api` |
| coreth `admin.*` | `/ext/bc/C/admin` | `ava-evm::admin` |
| `proposervm.*` (JSON-RPC + Connect) | `/ext/bc/<id>/proposervm` | `ava-proposervm::service` (+ tonic/Connect) |

**`register_chain` contract** (from `12` §3.1, binding): the chain manager calls
`ApiServer::register_chain(name, ctx, vm)`, which:
1. calls `vm.create_handlers()` → `HashMap<extension, Handler>` and mounts each at
   `/ext/bc/<chainID>/<extension>` (validate extension like Go's
   `url.ParseRequestURI`; `""` ⇒ `/ext/bc/<chainID>`);
2. calls `vm.new_http_handler()` and, if non-nil, registers it as the
   **header-route** handler for `<chainID>` (EVM `/rpc`,`/ws`; proposervm Connect);
3. registers chain aliases (`P`,`X`,`C`) as path aliases;
4. wraps each handler in the per-chain metrics + OTel-trace + not-bootstrapped-503
   middleware (`wrapMiddleware`).

JSON-RPC dispatch reproduces the gorilla v1 codec exactly: `params` is a
single-element array, method = `Service.Method` (PascalCase, case-insensitive on
the method segment), error objects `{code, message, data}` (`12` §3.2).

---

## 14. Parity test plan (ref `02`, `12` §12.4)

1. **Per-method request/response snapshots.** For every method in §3–§11, drive an
   identical JSON-RPC request at a Go node and the Rust node in the differential
   harness (`02`) and assert byte/shape-equal responses (modulo non-deterministic
   fields — timestamps, peer lists — which are diffed structurally, not by value).
2. **Method-set completeness.** A generated test enumerates the registered method
   names from each Rust `JsonRpcService` and asserts set-equality against a snapshot
   extracted from the Go services (gorilla reflection dump) — fails on any drift in
   either direction. Same for EVM namespaces vs reth's registered modules.
3. **Wire-shape conformance.** Golden tests assert the single-element `params`
   array, `Service.Method` naming, and the `{code,message,data}` error object for
   the gorilla services; standard Ethereum JSON-RPC shape for `/rpc`.
4. **HTTP semantics.** Allowed-hosts `403`, not-bootstrapped `503`, CORS headers,
   `node-id` response header, and health GET `200`/`503` status codes are asserted
   against the Go node.
5. **EVM divergence checks.** Explicit tests for `eth_baseFee`,
   `eth_suggestPriceOptions`, `eth_getChainConfig`, ACP-176 fee semantics on
   `eth_gasPrice`/`eth_feeHistory`, the `warp_*` and `avax.*` methods, and
   accepted-block tag mapping (`latest`/`finalized`/`safe` → last-accepted).
6. **Connect parity.** `proposervm.ProposerVM` and `xsvm.Ping` exercised over the
   h2c port with a Connect client; responses diffed against Go.
7. **Metrics-name golden test** for `/ext/metrics` (`00` §7.3).

---

## 15. Summary counts

| API | Path | Methods |
|---|---|---|
| Info | `/ext/info` | 13 |
| Admin | `/ext/admin` | 13 |
| Health | `/ext/health[/…]` | 3 JSON-RPC + 4 GET probes |
| Metrics | `/ext/metrics` | 1 (HTTP) |
| Index | `/ext/index/<alias>/{block,tx,vtx}` | 6 |
| P-Chain | `/ext/bc/P` | 31 (incl. fee/L1) |
| X-Chain | `/ext/bc/X` | 11 |
| C-Chain | `/ext/bc/C/{rpc,ws,avax,admin}` | geth `eth`/`debug`/`net`/`web3`/`txpool`/`personal` + ★`warp`(5) + ★`avax`(4) + coreth `admin`(6) |
| ProposerVM | `/ext/bc/<id>/proposervm` | 2 JSON-RPC + 2 Connect |
| xsvm | header-route | 2 Connect (test-only) |

**Keystore (`/ext/keystore`) is removed on this branch** and not implemented.

---

## 16. Error taxonomy & status codes

This section is the authoritative error contract for §3–§12, satisfying the `00`
requirement that the Rust port surface the **same error codes** (and, where clients
parse them, the same error **messages**) as the Go node. Three independent wire
conventions are in play and must not be conflated:

1. **gorilla `json2`** — all non-EVM services (info/admin/health/index/platform/avm
   /avax-atomic/coreth-admin/proposervm-JSON-RPC).
2. **go-ethereum `rpc`** — the C-Chain `eth`/`debug`/`net`/`web3`/`txpool`/`personal`
   /`warp` namespaces on `/ext/bc/C/{rpc,ws}`.
3. **HTTP transport layer** — `api/server` middleware (routing, allowed-hosts,
   not-bootstrapped, health probes) and the gRPC plugin boundary (`15`).

> **Codec correction (binding for this section).** §1.1 calls the non-EVM codec the
> "v1 `json`" codec; the actual registration is **`json2`** wrapped by a
> first-letter-uppercasing shim (`utils/json/codec.go` → `json2.NewCodec()`). The
> error object is `json2`'s `{code, message, data}` (`v2/json2/error.go`). The
> error **codes below are normative**; where §1.1 and §16 differ, §16 wins.

### 16.1 gorilla `json2` JSON-RPC 2.0 error codes

Defined in `gorilla/rpc@v1.2.0/v2/json2/error.go`:

| Code | json2 name | When raised |
|---|---|---|
| `-32700` | `E_PARSE` | request body is not valid JSON (`server.go:107`) |
| `-32600` | `E_INVALID_REQ` | valid JSON but not a valid JSON-RPC request, or wrong `jsonrpc` version (`server.go:113,166`) |
| `-32601` | `E_NO_METHOD` | `Service.Method` not registered; also raised by the uppercase-method guard (`utils/json/codec.go::errUppercaseMethod`) |
| `-32602` | `E_BAD_PARAMS` | params present but unmarshal fails — surfaced as `errInvalidArg` ("couldn't unmarshal an argument…") |
| `-32603` | `E_INTERNAL` | reserved internal error |
| `-32000` | `E_SERVER` | **default for any handler-returned `error`** that is not already a `*json2.Error` (`server.go:191`) |

**Mapping rule (the one that matters for parity):** an avalanchego service method
returns a plain Go `error`; gorilla's main server sees `errResult != nil`, sets the
HTTP status to `400` and calls `codecReq.WriteError`, which wraps the error as
`{code: -32000, message: err.Error(), data: null}`. The services almost never
construct `*json2.Error` themselves, so **every domain error from platform/avm/info
/etc. is code `-32000` with the message = the Go error string verbatim.**

> **HTTP status nuance.** The outer gorilla server passes `400`
> (`StatusBadRequest`) to `WriteError`, but `json2.writeServerResponse`
> (`server.go:211`) **ignores the status argument** — it only sets `Content-Type`
> and encodes the body. Net effect: handler/domain errors return **HTTP 200** with a
> JSON-RPC error body. Only the pre-dispatch failures use real HTTP status codes:
> `405` (non-POST), `415` (unrecognized `Content-Type`), `400` (method/get/read
> failure before dispatch). The Rust shim MUST reproduce this: HTTP 200 + error
> body for domain errors, not 400/500.

**Representative client-contract messages** (returned verbatim as the `-32000`
message; preserve byte-for-byte):

| Service | Go sentinel / message |
|---|---|
| platform | `"the primary network isn't a subnet"` (`errPrimaryNetworkIsNotASubnet`) |
| platform | `"no addresses provided"` (`errNoAddresses`) |
| platform | `"argument 'blockchainID' not given"` (`errMissingBlockchainID`) |
| platform | `"should have a decision block within the past two blocks"` (`errMissingDecisionBlock`) |
| platform | `"number of addresses given, %d, exceeds maximum, %d"` (`getUTXOs`) |
| avm | `"transaction doesn't create an asset"` (`errTxNotCreateAsset`) |
| avm | `"nil transaction ID"` (`errNilTxID`) |
| avm | `"chain is not linearized"` (`errNotLinearized`) |
| codec | `"couldn't unmarshal an argument. Ensure arguments are valid and properly formatted. See documentation for example calls"` (`errInvalidArg`, but code `-32602`) |

### 16.2 C-Chain / go-ethereum RPC error codes

From `graft/evm/rpc/errors.go` and `graft/coreth/internal/ethapi/errors.go`.
**★ = Avalanche/coreth-specific** behavior (the *code* is geth-standard; the
*trigger* or message diverges).

| Code | Meaning | Source / notes |
|---|---|---|
| `-32700` | parse error | `parseError`, `invalidMessageError` |
| `-32600` | invalid request | `invalidRequestError` |
| `-32601` | method not found / not available | `methodNotFoundError` ("the method %s does not exist/is not available"); also subscription-not-found and `notificationsUnsupportedError` |
| `-32602` | invalid params | `invalidParamsError` |
| `-32603` | internal error | `errcodePanic`, `errcodeMarshalError` |
| `-32000` | default server error (`errcodeDefault`) | catch-all for backend errors incl. ★`TxIndexingError` ("transaction indexing is in progress") |
| `-32001` | notifications unsupported (legacy) | `legacyErrcodeNotificationsUnsupported` |
| `-32002` | request timed out | `errcodeTimeout` |
| `-32003` | response too large | `errcodeResponseTooLarge` |
| `3` | **execution reverted** | `revertError.ErrorCode()` = `3`; `message` = `"execution reverted[: <decoded reason>]"`, `data` = `0x…` hex of the raw revert blob (`ErrorData()`). Emitted by `eth_call`/`eth_estimateGas` on `vm.ErrExecutionReverted` |

**Avalanche-specific error cases** (still surfaced under the codes above, usually
`-32000`, but with Avalanche messages clients key on):
- ★ Fee-cap rejection on `eth_sendRawTransaction`/`eth_sendTransaction`/`eth_resend`:
  `"tx fee (%.2f ether) exceeds the configured cap (%.2f ether)"` (`checkTxFee`,
  `RPCTxFeeCap`).
- ★ Gas-estimation: `"gas required exceeds allowance (%d)"` and geth's
  `"insufficient funds for gas * price + value"` (values from the **ACP-176**
  dynamic-fee mechanism, not London — see §10.1).
- ★ Atomic-tx (`avax.*`) errors cross the `avax` namespace as **gorilla `json2`**
  errors (code `-32000`), not geth codes — e.g. `"tx has wrong chain ID"`,
  `"import input cannot contain non-AVAX in Banff"`,
  `"insufficient AVAX funds to pay transaction fee"`
  (`plugin/evm/atomic/import_tx.go`). The `avax` endpoint uses gorilla, so it
  follows §16.1, not this table.
- ★ `eth_getChainConfig`/`eth_*` extras that pre-build a reply with an embedded
  `ErrCode` (`api_extra.go:45,52`) propagate the underlying `Error.ErrorCode()`.

### 16.3 HTTP transport-layer status codes

These come from `api/server` middleware and the health handler, **before/around**
the JSON-RPC layer:

| Status | When | Source |
|---|---|---|
| `403 Forbidden` | `Host` header not in `http-allowed-hosts` ("invalid host specified") | `allowed_hosts.go:75` (`filterInvalidHosts`) |
| `404 Not Found` | header-route value present but no matching header-route handler; unknown base URL / endpoint | `router.go:70`, `errUnknownBaseURL` |
| `400 Bad Request` | header-route key present with empty value | `router.go:64` |
| `503 Service Unavailable` | chain not yet `NormalOp`: "API call rejected because chain is not done bootstrapping" | `server.go:263` |
| `503 Service Unavailable` | health/liveness/readiness probe failing (any check unhealthy) | `health/handler.go:59` |
| `405 / 415` | non-POST to a JSON-RPC mount / unrecognized `Content-Type` | gorilla `v2/server.go:149,165` |
| CORS | `rs/cors`: `Access-Control-Allow-Origin` from `http-allowed-origins` (default `*`), `Allow-Credentials: true`; disallowed origins get no CORS headers (browser blocks) | `server.go` |

There is **no** API-token auth middleware on this branch (§1.3), so no `401`.
Payload-size limits are not enforced by a dedicated `413` middleware in `api/server`
(the EVM `rpc` server enforces its own batch/response-size limits via codes
`-32003`/`errMsgBatchTooLarge`, §16.2). A response header `node-id` (a.k.a. the
`<networkID>` node identifier) is set on **every** response (§1.3) — assert its
presence in parity tests even on error responses.

### 16.4 gRPC plugin services — `tonic::Status` mapping (cross-ref `15`)

The plugin protocol (`proto/vm`, `rpcdb`, `appsender`, `sharedmemory`,
`validatorstate`, …; spec `07`/`15`) is **not** a public node API, but the Rust
port must reproduce its error semantics across the gRPC boundary. Two distinct
mechanisms coexist:

1. **Error-in-response-message (preferred, dominant).** Most RPCs return an
   explicit error field *inside* a successful gRPC response rather than a non-OK
   gRPC status:
   - `rpcdb.proto`: `enum Error { ERROR_UNSPECIFIED=0, ERROR_CLOSED=1,
     ERROR_NOT_FOUND=2 }` carried as `err` on `GetResponse`/`PutResponse`/etc.
   - `vm.proto`: `enum Error { ERROR_UNSPECIFIED=0, ERROR_CLOSED=1,
     ERROR_NOT_FOUND=2, ERROR_STATE_SYNC_NOT_IMPLEMENTED=3 }` carried as `err` on
     block/state-sync responses; `AppRequestFailed` carries app-defined
     `error_code: sint32` + `error_message: string`.
   - The client side maps these enums back to Go sentinels (`database.ErrClosed`,
     `database.ErrNotFound`, `block.ErrStateSyncableVMNotImplemented`). The Rust
     port maps them to the corresponding `thiserror` variants — **not** to a
     `tonic::Status`. A `0`/`UNSPECIFIED` value means success.
2. **gRPC status (transport/unexpected failures).** Genuine transport errors,
   panics, or codec failures cross as a real gRPC status; tonic surfaces these as
   `tonic::Status { code, message }`. Standard mapping:

| gRPC code | tonic | When |
|---|---|---|
| `OK` | (Ok response) | normal; domain error is in the `err`/`error_code` field |
| `Unavailable` | `Code::Unavailable` | plugin process down / dial failure |
| `Internal` | `Code::Internal` | server panic, marshal failure |
| `Unimplemented` | `Code::Unimplemented` | RPC not registered (vs the in-band `ERROR_STATE_SYNC_NOT_IMPLEMENTED`) |
| `DeadlineExceeded` | `Code::DeadlineExceeded` | context timeout |
| `Canceled` | `Code::Cancelled` | context cancellation |

> **Distinction to preserve (cross-ref `15` §3, §5):** an *expected* domain
> condition (key not found, DB closed, state-sync not implemented) is an **in-band
> enum/`error_code`**, returned with gRPC status `OK`. Only *infrastructure*
> failures use a non-OK `tonic::Status`. Reversing this would break parity with Go
> clients that pattern-match the enum.

### 16.5 Rust error → wire mapping at the API boundary

Per `00` §7.1 each crate owns a `thiserror` `Error` enum (Go sentinels → variants).
The mapping into a wire error happens **only at the API boundary** (the JSON-RPC
shim, the reth RPC module, or the tonic service impl), keeping domain crates
transport-agnostic. Message strings are preserved verbatim where §16.1/§16.2 flags
them client-sensitive.

```rust
/// JSON-RPC 2.0 error object on the wire (gorilla json2 shape).
pub struct JsonRpcError {
    pub code: i32,
    pub message: String,
    pub data: Option<serde_json::Value>,
}

/// Standard json2 codes (utils/json + gorilla v2/json2/error.go).
pub mod json2_code {
    pub const PARSE: i32       = -32700; // E_PARSE
    pub const INVALID_REQ: i32 = -32600; // E_INVALID_REQ
    pub const NO_METHOD: i32   = -32601; // E_NO_METHOD
    pub const BAD_PARAMS: i32  = -32602; // E_BAD_PARAMS
    pub const INTERNAL: i32    = -32603; // E_INTERNAL
    pub const SERVER: i32      = -32000; // E_SERVER (handler-error default)
}

/// Boundary trait: each service's domain Error declares how it surfaces.
/// The DEFAULT mirrors gorilla exactly — any handler error becomes
/// {code: -32000, message: err.to_string(), data: null}.
pub trait IntoJsonRpcError {
    fn to_json_rpc(&self) -> JsonRpcError;
}

impl<E: std::error::Error> IntoJsonRpcError for E {
    fn to_json_rpc(&self) -> JsonRpcError {
        JsonRpcError {
            code: json2_code::SERVER,        // -32000, like json2 WriteError
            message: self.to_string(),       // BYTE-STABLE: must equal Go err.Error()
            data: None,                       // json2 emits explicit null
        }
    }
}

/// C-Chain / geth boundary: reth RPC modules implement geth's `ErrorCode`/`ErrorData`.
/// `latest` tag → last-accepted block (§10.1); fee errors carry ACP-176 values.
pub enum EvmRpcError {
    Reverted { reason: String, data: Vec<u8> },   // code 3, data = 0x<hex>
    TxFeeCapExceeded { fee_eth: f64, cap_eth: f64 },// code -32000, geth message
    MethodNotFound(String),                        // code -32601
    InvalidParams(String),                         // code -32602
    Internal(String),                              // code -32603
    Server(String),                                // code -32000
}

impl EvmRpcError {
    pub fn code(&self) -> i32 {
        match self {
            EvmRpcError::Reverted { .. }         => 3,
            EvmRpcError::MethodNotFound(_)       => -32601,
            EvmRpcError::InvalidParams(_)        => -32602,
            EvmRpcError::Internal(_)             => -32603,
            EvmRpcError::TxFeeCapExceeded { .. }
            | EvmRpcError::Server(_)             => -32000,
        }
    }
    pub fn data(&self) -> Option<serde_json::Value> {
        match self {
            // revert blob hex-encoded, exactly like coreth ErrorData()
            EvmRpcError::Reverted { data, .. } =>
                Some(format!("0x{}", hex::encode(data)).into()),
            _ => None,
        }
    }
}

/// gRPC plugin boundary (§16.4): expected domain conditions go IN-BAND as the
/// proto `err` enum with gRPC status OK; only transport failures become Status.
pub fn db_err_to_proto(e: &DatabaseError) -> rpcdb::Error {
    match e {
        DatabaseError::Closed   => rpcdb::Error::Closed,    // ERROR_CLOSED = 1
        DatabaseError::NotFound => rpcdb::Error::NotFound,  // ERROR_NOT_FOUND = 2
    } // success -> ERROR_UNSPECIFIED = 0
}
// Reserve tonic::Status::{unavailable,internal,unimplemented,...} for genuine
// transport/panic failures only (never for NotFound/Closed/StateSyncNotImpl).
```

**Byte-stability rule.** Variants whose `Display` (`#[error("…")]`) feeds a
client-parsed message MUST reproduce the Go string exactly, including format
arguments and punctuation (the §16.1/§16.2 tables enumerate the load-bearing ones).
Use a golden test per such message; do not "improve" wording.

### 16.6 Parity test plan (ref `02`, extends §14)

8. **Error-response snapshots.** For each surface, drive known bad inputs at a Go
   node and the Rust node through the differential harness (`02` §differential) and
   diff the full error object:
   - gorilla: missing required arg, unknown method, malformed JSON, uppercase
     method segment, non-POST, bad `Content-Type` → assert `code`, `message`
     (byte-equal), `data: null`, and the **HTTP status** (200 for domain errors;
     405/415/400 pre-dispatch).
   - geth: `eth_call` to a reverting contract (code `3` + `data` hex), unknown
     method (`-32601`), bad params (`-32602`), over-cap raw tx (the fee-cap
     message), oversized batch (`-32003`).
   - HTTP middleware: disallowed `Host` → `403`; not-bootstrapped chain → `503` +
     exact message; failing health probe → `503` body `{checks, healthy:false}`;
     `node-id` header present on error responses.
   - gRPC: assert `NotFound`/`Closed`/`StateSyncNotImplemented` arrive as the
     **in-band enum with status OK**, and that a killed plugin yields a non-OK
     `tonic::Status` (`Unavailable`).
9. **Message-string golden vectors.** Commit the `#[error("…")]` strings for every
   client-sensitive variant (§16.1/§16.2) as `insta` snapshots extracted from Go
   `err.Error()`; CI fails on drift in either direction.
</content>
</invoke>
