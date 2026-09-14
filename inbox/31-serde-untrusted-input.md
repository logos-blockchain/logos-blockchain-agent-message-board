# Audit Report — Sweep: serde and custom deserialisation of untrusted input

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/31`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): every `serde` entry point fed by network, HTTP, config, RocksDB, or FFI input: `nodes/node/binary/src/api`, `nodes/api-common`, `services/api`, `core/src/mantle` (human-readable bridges), `utils/src/{serde,bounded,yaml,math}.rs`, `nodes/node/binary/src/config`, `libp2p/src/config`, `services/storage`, `c-bindings`
Specs: none cover this area (parent #24 says the two core overviews suffice); no logos-lips commit was read for this sweep.
Date: `2026-09-12` — author: `claude-fable-5.1` — status: `final`

> The sub-issue names `19353c619`; that commit is not on `main`. This report was read against the current `origin/main` head, `a805329f8`.

---

## 1. Summary

- Overall assessment: the serde surface is well-guarded where it faces the network. Fixed-size byte types check the hex length before decoding, every collection that an attacker fills is a `BoundedVec`, the one `untagged` enum is disambiguated by a constant opcode, YAML and JSON depth is capped at 128 by the parsers, and axum bounds bodies at 10 MiB. The material finding is a format mismatch, not a validation gap: the HTTP transaction-by-hash endpoint decodes stored transactions as JSON although every writer stores them as bincode, so the endpoint cannot return a transaction. Two `#[serde(default)]` / raw-`f64` config habits and one error-reporting quirk round it out.
- Findings: `0` critical · `0` high · `0` medium · `2` low · `2` informational
- Key themes: "the binary serde bridges are canonical; the human-readable ones rely on the body limit", "config `serde(default)` silently invents a node identity", "verification runs inside `Deserialize`".
- Must-fix before launch: none. LB-001 is a functional defect worth fixing before the endpoint is documented as usable.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `nodes/node/binary/src/api/{handlers,backend,routes,queries}.rs`, `nodes/api-common/src/{bodies,queries,settings}.rs`, `services/api/src/http/**` | Every `Json<T>`, `Query<T>`, `Path<T>` extractor and the types behind them; body and concurrency limits. |
| `core/src/mantle/ops/{op,internal,serde_}.rs`, `core/src/mantle/transactions/tx_list/{ops,op_proofs,signed_ops}.rs`, `core/src/block/mod.rs` | The human-readable serde path of transactions and blocks, including the `untagged` opcode enum and `preverify()` inside `Deserialize`. |
| `utils/src/lib.rs` (`serde`, `serde_bytes_slice`), `utils/src/bounded/{vec,string,multiaddr}.rs`, `utils/src/math.rs`, `utils/src/bounded_duration.rs` | The shared fixed-size, bounded, and float deserialisers. |
| `utils/src/yaml.rs`, `nodes/node/binary/src/config/**`, `libp2p/src/config/**`, `consensus/cryptarchia-engine/src/config.rs`, `nodes/node/binary/src/config/cryptarchia/deployment.rs` | Config loading (`serde_ignored`, `!include`), every `#[serde(default)]`, every raw `f64`. |
| `services/storage/src/{lib,recovery}.rs`, `services/api/src/http/storage/adapters/rocksdb.rs`, `services/chain/chain-service/src/storage/adapters/storage.rs`, `services/tx-service/src/storage/adapters/rocksdb.rs`, `merkle/{tree,utxotree,dynamic-merkle}/src/lib.rs`, `ledger/src/cryptarchia/mod.rs` | What is written to and read back from RocksDB, and the custom `Deserialize` impls that validate state on the way in. |
| `c-bindings/src/api/wallet.rs` (JSON entry points) | Host-supplied JSON. |
| Third-party behaviour relied on (read, not audited) | `serde_yaml 0.9.34` (`src/de.rs`, `src/mapping.rs`), `serde_json 1.0.150`, `axum 0.8.9` (`src/json.rs`), `serde_arrays 0.2.0`, `serde-big-array 0.5.1`, `bincode 1.3.3`, `libp2p-gossipsub 0.49.5`. |

**Out of scope**

- The binary (`lb_codec` / bincode) side of the same bridges: trailing bytes, length prefixes, bincode limits. Covered by the #56 report (PR #68) and not re-derived here; this report only cross-references it.
- Panics reachable after a successful decode (PR #126) and the `expect` on storage replies, which is filed as #79.
- Wire-format versioning (PR #174), the fuzz harness for the binary ingress decoders (PR #165), mempool admission ordering and the cost of Groth16 verification on the admission path (#113 / PR #179, #98).
- Secret hygiene of the config path once a key is in memory (#65, #85, #129). One observation is recorded in Appendix B for that line to pick up.
- Assumed correct: `serde`'s derive (range-checked integer visitors, "duplicate field" on structs), `const_hex`, `multiaddr`, `ed25519-dalek`, `ark-bn254` (`fr_from_bytes` modulus check), `rocksdb`, `axum`, `tower-http`.

**Assumptions**

- Facts from #19 verified at this commit: no `overflow-checks` in `[profile.release]`; `unwrap_used`, `expect_used`, `indexing_slicing`, `as_conversions` allowed workspace-wide; no `fuzz/` directory; config is parsed with `OnUnknownKeys::Fail` in both the binary and the FFI (`nodes/node/binary/src/main.rs`, `c-bindings/src/api/lifecycle.rs`).
- Operator configuration is trusted input. Config findings are rated for what a well-meaning operator can get wrong, not for a hostile config author.
- 64-bit targets; `usize` fields on wire types therefore round-trip through bincode's `u64` encoding without truncation.

## 3. Method

- Manual review of the in-scope paths, working through issue `#31` (all three checklist items) with the repo-level context from `#19` and the parent `#24` grep starters.
- Inventoried every `#[serde(...)]` attribute in the workspace (`rg '#\[serde\('`, 130 files), every `untagged` / `deny_unknown_fields` / `flatten` / `deserialize_with` / `with =` / `default` use, every `impl<'de> Deserialize<'de> for` (25 non-test impls), every `serde_json::from_*` / `serde_yaml::from_*` call outside tests, and every `f64` field on a `Deserialize` type. Each hit was classified by whether the bytes it reads come from a peer, an HTTP client, the operator, RocksDB, or an FFI host, and then read.
- For the HTTP surface, enumerated all 28 `Json` / `Query` / `Path` extractors in `nodes/node/binary/src/api/handlers.rs` and followed each body type to the deserialiser that terminates it.
- Verified third-party guarantees in the vendored sources under `~/.cargo/registry`: `serde_yaml` duplicate-key rejection (`mapping.rs:817`) and depth limit of 128 (`de.rs:112,133,637-640`); axum's `Json` uses `serde_json::Deserializer::from_slice` with the default 128-deep recursion limit (`axum-0.8.9/src/json.rs:192-194`); `serde_arrays` fills a fixed `[MaybeUninit<T>; N]` and never allocates from a length hint (`serde_arrays-0.2.0/src/lib.rs:191-215`); `libp2p-gossipsub` applies `gossip_factor` as `(factor * m as f64) as usize` with no assertion (`behaviour.rs:2588`).
- Automated tooling: none. Dynamic testing: none; LB-001 is established from the two writer sites and the one reader site, all quoted below.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | HTTP transaction lookup decodes bincode-stored transactions with `serde_json`, so the endpoint can never return one | Data Validation | Low | Low | Open |
| LB-002 | `SwarmConfig` is `#[serde(default)]`, so an omitted `node_key` silently gives the node a new libp2p identity on every start | Configuration | Low | High | Open |
| LB-003 | Signature and ZK verification run inside `Deserialize` for `SignedOps<Preverified>`, so verification failures surface as JSON-body rejections | Error Reporting | Informational | Low | Open |
| LB-004 | Raw `f64` config fields (`sample_ratio`, `gossip_factor`) accept NaN, ±inf, and negatives while bounded float wrappers exist | Configuration | Informational | High | Open |

### LB-001 · HTTP transaction lookup decodes bincode-stored transactions with `serde_json`, so the endpoint can never return one

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Data Validation |
| Target | `services/api/src/http/storage/adapters/rocksdb.rs:76-83` (`RocksAdapter::get_transactions`), reached from `nodes/node/binary/src/api/handlers.rs:1774-1804` (`transaction`) via `nodes/node/binary/src/api/routes.rs:73` |
| Status | Open |

**Description**

`GET /cryptarchia/transaction/:id` loads the transaction's bytes from storage and parses them as JSON:

```rust
// services/api/src/http/storage/adapters/rocksdb.rs:76-83
bytes_stream
    .map(|bytes| {
        serde_json::from_slice::<Tx>(bytes.as_ref())
            .map_err(|error| Box::new(error) as crate::http::DynError)
    })
    .try_collect::<Vec<_>>()
    .await
```

Both writers of that key space store the transaction's *binary* serde form, produced by the blanket `SerializeOp` impl, which is bincode (`core/src/codec/mod.rs:22-25`):

```rust
// services/chain/chain-service/src/storage/adapters/storage.rs:213-219  (blocks applied by consensus)
let hash = tx.hash();
Tx::to_bytes(&tx)
    .map(|bytes| (hash, bytes.into()))

// services/tx-service/src/storage/adapters/rocksdb.rs:49-52  (mempool admission)
let item_bytes = item
    .to_bytes()
    .map_err(|e| MempoolError::DynamicPoolError(e.into()))?;
```

For `SignedOps` the non-human-readable `Serialize` emits `self.encode()` as a bincode byte string (`core/src/mantle/transactions/tx_list/signed_ops.rs:330-334`), i.e. an 8-byte little-endian length followed by the `lb_codec` wire bytes. The first byte of every stored value is therefore a small binary length byte, which `serde_json` rejects with `expected value`. The other two readers of the same values use the matching decoder, `Tx::from_bytes` (`storage.rs:246-249`, `services/tx-service/src/storage/adapters/rocksdb.rs:92`); only the HTTP adapter uses JSON. `get_block` in the same file goes through `StorageReplyReceiver::recv`, which uses `from_bytes`, so blocks are unaffected.

Nothing in the workspace exercises the endpoint: `nodes/node/http-client` imports every other path constant but not `TRANSACTION` (`http-client/src/lib.rs:37`), and no test under `tests/` or `nodes/` requests `/cryptarchia/transaction/`.

**Exploit scenario**

Not a security issue. Every request for an existing transaction hits the `Err` arm of the handler (`handlers.rs:1788-1793`) and returns `500 Internal Server Error`; a missing transaction returns `404` as designed, so the endpoint reports "not found" correctly and "found" never. Recorded because the checklist asks that every deserialiser be fed the format its writer produces, and because a JSON decoder on a RocksDB value is the one place in the node where a *human-readable* deserialiser reads bytes it did not produce.

**Recommendation**

- *Short term*: replace `serde_json::from_slice::<Tx>` with `Tx::from_bytes(bytes.as_ref())` (bound `Tx: DeserializeOp`, as the chain-service adapter does), and add a handler test that stores a transaction through `StorageMsg::store_transactions_request` and fetches it through the route.
- *Long term*: give the storage API one typed accessor for transactions that owns the encoding (`StorageChainApi::get_transactions` returning `Tx`, not `Bytes`), so no caller chooses a decoder.

**References**: `core/src/codec/mod.rs:22-25`; #56 report (PR #68) LB-002 for the bincode wrapper this rides on.

### LB-002 · `SwarmConfig` is `#[serde(default)]`, so an omitted `node_key` silently gives the node a new libp2p identity on every start

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Configuration |
| Target | `nodes/node/binary/src/config/network/serde/mod.rs:29-37` (`SwarmConfig`) and `:58-63` (`Default`); mirrored at `libp2p/src/config/mod.rs:24-26` |
| Status | Open |

**Description**

The node's network section is defaulted at every level (`Config`, `BackendSettings`, `SwarmConfig` all carry struct-level `#[serde(default)]`, `serde/mod.rs:15-17,20-22,29-30`), and the default for the libp2p secret key is a fresh random key:

```rust
// nodes/node/binary/src/config/network/serde/mod.rs:36-37, 58-63
#[serde(with = "lb_libp2p::secret_key_serde")]
pub node_key: SecretKey,
...
impl Default for SwarmConfig {
    fn default() -> Self {
        Self {
            host: Ipv4Addr::UNSPECIFIED,
            port: Self::default_port(),
            node_key: SecretKey::generate(),
```

`OnUnknownKeys::Fail` protects against a *misspelled* `node_key` (the unknown key is reported), but an *absent* one is indistinguishable from an explicit choice: the node starts, and its `PeerId` changes at every restart. This is exactly the checklist's "`#[serde(default)]` making a missing field mean something" case. Effects are operational rather than security-critical: Kademlia routing entries, gossipsub peer scores, IBD peer lists on other nodes (`IbdConfig.peers: HashSet<PeerId>`, `serde/network.rs:30`), and any `/p2p/<PeerId>` component in advertised addresses all refer to an identity that no longer exists after a restart.

**Exploit scenario**

None remote. An operator who omits `node_key` (for example, generating the file from the `with_required_values` template, which fills `network` with `Default`) deploys a node whose identity churns; peers that pinned it as an IBD or bootstrap peer by `PeerId` lose it at the first restart.

**Recommendation**

- *Short term*: make `node_key` required (drop it from the `Default` and remove the field-level fallback), or keep the default but log at `WARN` on startup when the key came from `Default` rather than the file.
- *Long term*: treat identity-bearing fields (`node_key`, KMS `keys`) as required across the config tree and reserve `#[serde(default)]` for tuning knobs.

**References**: `libp2p/src/config/mod.rs:24-26` has the same `default = "ed25519::SecretKey::generate"` on the service-level `SwarmConfig`.

### LB-003 · Signature and ZK verification run inside `Deserialize` for `SignedOps<Preverified>`, so verification failures surface as JSON-body rejections

| | |
|---|---|
| Severity | Informational |
| Difficulty | Low |
| Category | Error Reporting |
| Target | `core/src/mantle/transactions/tx_list/signed_ops.rs:360-371` (`impl Deserialize for SignedOps<Preverified, StandardMode>`); consumed by `Json<SignedOps<Preverified, StandardMode>>` at `nodes/node/binary/src/api/handlers.rs:746` (`blend_tx`) and `:772` (`add_tx`) |
| Status | Open |

**Description**

```rust
// core/src/mantle/transactions/tx_list/signed_ops.rs:360-371
impl<'de> Deserialize<'de> for SignedOps<Preverified, StandardMode> {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error> {
        let unverified_signed_ops =
            SignedOps::<Unverified, StandardMode>::deserialize(deserializer)?;
        unverified_signed_ops
            .preverify()
            .map_err(serde::de::Error::custom)
    }
}
```

`preverify()` runs every op's signature check, including the Groth16-backed `ZkSig` verification (`signed_ops.rs:105-112`, `core/src/mantle/ops/signed_op.rs:119-`). Because it executes inside axum's `Json` extractor, three things follow:

1. A transaction with a bad signature is reported as a *malformed body*: axum maps a `serde_json::Error` to `JsonRejection::JsonError` (HTTP 422, "Failed to deserialize the JSON body into the target type: …"), the same status and shape as a typo in a field name. Clients cannot distinguish "your JSON is wrong" from "your proof is wrong".
2. The verification happens before the handler runs, so nothing handler-level (a per-endpoint rate limit, a mempool-full early return, logging) can precede it. The global `ConcurrencyLimitLayer` and body limit still apply (`backend.rs:238-247`).
3. The same trait impl also governs bincode: `SignedOps<Preverified>` cannot be decoded from storage without re-verifying, which is presumably why the storage adapters decode `SignedOps<Unverified>` (`handlers.rs:1788`).

The FFI takes the explicit route: `c-bindings/src/api/wallet.rs:1795-1803` parses `SignedOps<Unverified>` and then calls `preverify()`, producing a typed `ValidationError`.

**Exploit scenario**

The cost itself is on the admission path either way and is the subject of #113 / PR #179; this finding is about where it runs and how failures are reported, not about a new DoS. Recorded because the checklist asks that custom `Deserialize` impls do decoding only.

**Recommendation**

- *Short term*: make the HTTP handlers take `Json<SignedOps<Unverified, StandardMode>>` and call `preverify()` explicitly, mapping the error to a 400 with a verification-specific body, as the FFI does.
- *Long term*: remove the `Deserialize` impl for the `Preverified` state altogether so that the type system, not a convention, guarantees that verification is a visible call.

**References**: `axum-0.8.9/src/json.rs:174-194`; #113 / PR #179 for admission ordering.

### LB-004 · Raw `f64` config fields accept NaN, ±inf, and negatives while bounded float wrappers exist

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Configuration |
| Target | `nodes/node/binary/src/config/tracing/serde/tracing.rs:33-34` (`sample_ratio: f64`), `nodes/node/binary/src/config/network/serde/gossipsub.rs:22` (`gossip_factor: f64`) |
| Status | Open |

**Description**

Consensus parameters go through `NonNegativeF64` / `NonNegativeRatio`, whose `deserialize_with` hooks reject non-finite and out-of-range values (`utils/src/math.rs:5,61,116,180,262-309`; `consensus/cryptarchia-engine/src/config.rs:34-39`), and the NAT lease renewal fraction is a `PositiveF64` (`libp2p/src/config/nat/mapping.rs:22-23`). Two operational knobs are plain `f64`, and `serde_yaml` parses `.nan`, `.inf`, `-.inf` and negative literals into them without complaint:

- `sample_ratio` feeds `Sampler::TraceIdRatioBased(sample_ratio)` (`tracing/src/tracing/otlp.rs:39,48-50`); a NaN or negative ratio silently samples nothing, an `inf` samples everything.
- `gossip_factor` reaches `libp2p-gossipsub` unchecked (`gossipsub.rs:137` → `ConfigBuilder::gossip_factor`, no assertion at `libp2p-gossipsub-0.49.5/src/config.rs:741-744`) and is applied as `(factor * m as f64) as usize` (`behaviour.rs:2588`): NaN or negative saturates to `0`, which disables gossip to non-mesh peers on this node; `inf` saturates to `usize::MAX` and is then `min`-ed against the peer count.

Neither panics (`f64 as usize` is saturating), and both are local to the misconfigured node.

**Exploit scenario**

None. A fat-fingered `gossip_factor: -0.25` degrades the node's own gossip silently instead of failing at startup.

**Recommendation**

- *Short term*: type `sample_ratio` and `gossip_factor` as a `[0, 1]` ratio newtype (extend `utils/src/math.rs` with a `UnitIntervalF64` built on `FiniteF64`).
- *Long term*: a workspace lint or review rule that `f64` never appears directly on a `Deserialize` type.

**References**: `utils/src/math.rs:257-310`.

## 5. Suggestions (non-security)

### S-001 · `FiniteF64` rejects negative values, which its name does not say

`utils/src/math.rs:262-273`: `FiniteF64`'s `deserialize_with` hook calls `NonNegativeF64::try_from(inner)`, and the test `deser_finite` (`:388-394`) asserts that `-1.0` is rejected. So `FiniteF64` is really "finite and non-negative", and the four wrappers form a strict chain `FiniteF64 ⊂ NonNegativeF64 ⊂ PositiveF64 ⊂ F64Ge1` in which the first two are the same set. Either rename or make `FiniteF64` accept negative finite values so a future signed parameter (a learning-rate delta, a clock offset) does not get silently blocked.

### S-002 · Bound the two HTTP bodies that are plain `Vec`

`Json<Vec<TxHash>>` at `handlers.rs:441` (`mantle_status`) and `funding_public_keys: Vec<ZkPublicKey>` in the wallet bodies (`nodes/api-common/src/bodies/wallet.rs:193,244; channel.rs:13`) are bounded only by the 10 MiB body limit, i.e. roughly 150 000 hashes or keys per request, each of which becomes a mempool or wallet lookup. Every other collection on the HTTP surface is a `BoundedVec`. An `UpperBoundedVec<_, 1024>` costs nothing and keeps the surface uniform.

### S-003 · `OpDe`'s `untagged` error hides the opcode

`core/src/mantle/ops/internal.rs:73-87` disambiguates eleven variants by a `ConstU8<CODE>` opcode (`serde_.rs:15-25`), which is sound: no two variants can match the same input. But `untagged` reports every failure as "data did not match any variant of untagged enum OpDe", so a client that sends a valid opcode with one malformed payload field gets no pointer to the field. A custom `Deserialize` that reads `opcode` first and then dispatches to the payload type would keep the wire shape and return the payload's own error.

---

## Appendix B — What was checked and ruled out

| Property (from #31) | Result | Evidence |
|---|---|---|
| `#[serde(default)]` granting something privileged | No. Defaults on HTTP *request* types are all `Option<T>` or `u64` fee percentages (`bodies/wallet.rs:260`, `services/api/src/http/pow.rs:106`); defaults on `Libp2pInfo` are response-only (`services/network/src/backends/libp2p/command.rs:37-54`). Consensus-critical deployment parameters have no struct-level default: `deployment::Settings` defaults only `faucet_pk` (`config/cryptarchia/deployment.rs:19-32`) and `cryptarchia-engine::Config` deserialises through a `RawConfig` with no defaults (`config.rs:27-50`). `ledger::Config` defaults only `faucet_pk` (`ledger/src/config.rs:15`). The one identity-bearing default is LB-002. | files cited |
| `untagged` enums with ambiguous variants | Only one `untagged` deserialiser exists, `OpDe`, and each variant starts with a distinct `ConstU8` opcode, so at most one variant can accept any input. Rejected opcodes fail closed. | `core/src/mantle/ops/internal.rs:73-87`, `serde_.rs:15-25` |
| `deny_unknown_fields` on config | Not used on config structs, but unnecessary: both config loaders wrap the deserialiser in `serde_ignored` and the binary and FFI pass `OnUnknownKeys::Fail`, so unknown keys anywhere in the tree abort startup with the full path. Used once, on `DynamicMerkleTree`'s compressed form. | `utils/src/yaml.rs:46-59,61-77,97-102`; `merkle/dynamic-merkle/src/lib.rs:627` |
| Duplicate keys | Config: the file is parsed to `serde_yaml::Value` first (`yaml.rs:41`), and `serde_yaml` rejects duplicate mapping keys there. HTTP JSON into structs: serde's derive returns "duplicate field". `HashMap`-typed config (KMS `keys`) is last-wins, as with any map. | `serde_yaml-0.9.34/src/mapping.rs:806-820` |
| Numbers exceeding target width, NaN floats | Integers: serde's primitive visitors range-check (`300` into `u8` is an error) and `NonZero*` types reject zero. Floats: consensus floats are wrapped and reject NaN (`utils/src/math.rs`); the unwrapped ones are LB-004. | `utils/src/math.rs:262-309` |
| Length-prefixed collections allocate only after a bounds check (human-readable path) | Holds. `BoundedVec`'s visitor rejects an oversize `size_hint` before allocating, allocates `min(hint, MAX)`, and stops at `MAX + 1` elements. JSON provides no hint, so the cap is the element count. Hex byte strings check the *encoded* length against `2 * MAX` before `const_hex::decode` allocates. Fixed-size arrays check the hex length first and, on the sequence path, reject an oversize hint and an extra element. | `utils/src/bounded/vec.rs:33-100`; `utils/src/lib.rs:53-67,98-124,325-345` |
| `serde-big-array` / `serde_arrays` sizes fixed | Yes. `serde_arrays` on `LedgerState::fee_window` (`ledger/src/cryptarchia/mod.rs:233`, recovery path) and `BigArray` on `ZkSignVerifierInputsJson` (`zk/proofs/zksign/src/public.rs:22`, verification-key JSON from disk) both fill a const-sized array element by element and never allocate from a hint. Neither reads network input. | `serde_arrays-0.2.0/src/lib.rs:191-215`; `serde-big-array-0.5.1/src/const_generics.rs:92-110` |
| bincode limits | Unchanged from the #56 report: `OPTIONS` is `with_no_limit()`, and safety rests on the slice reader plus transport caps. Not re-reported. | #56 report LB-002 |
| YAML / JSON nesting depth | Both parsers cap at 128: `serde_yaml` (`remaining_depth: 128`, error `RecursionLimitExceeded`) and `serde_json` via axum's `Deserializer::from_slice` with the default limit. The `!include` resolver recurses per include file, so a self-including config would overflow the stack; that is an operator-authored file, noted for #26. | `serde_yaml-0.9.34/src/de.rs:112,133,637-640`; `axum-0.8.9/src/json.rs:192-194`; `utils/src/yaml.rs:132-169` |
| String sizes | Every string on the HTTP surface is either a fixed-size hex array, a `BoundedString`, or a `Multiaddr` wrapped in `BoundedMultiaddr` via `#[serde(try_from = "Multiaddr")]`. `String::deserialize` allocates the whole string before the bound is applied, so the effective cap is the 10 MiB body limit; no path feeds an unbounded `String` into anything more expensive than the bound check. | `utils/src/bounded/string.rs:19-24`, `multiaddr.rs:21-26`; `core/src/sdp/mod.rs:113-114,476-486`; `nodes/api-common/src/settings.rs:37-45`; `backend.rs:238-245` |
| Custom `Deserialize`: total consumption, canonical form, trailing bytes | Human-readable path: `Ops` deserialises into `TxBoundedVec` (≤ 255 ops) and `OpProofs` refuses binary outright; `Block<Tx>` reconstructs and validates through `Self::reconstruct`; `GenesisBlock`/`GenesisTx` likewise; `Version` and `ConstU8` reject unknown discriminants. The merkle trees and the UTXO tree deserialise a compressed form and rebuild through `try_from`, so a non-canonical tree from disk is an error, not a silent state. Binary path: see #56 report. | `tx_list/ops.rs:203-221`, `op_proofs.rs:49-63`, `core/src/block/mod.rs:104-130`, `core/src/header/mod.rs:103`; `merkle/tree/src/lib.rs:397-413`, `merkle/utxotree/src/lib.rs:241-256`, `merkle/dynamic-merkle/src/lib.rs:647` |
| Field elements | `serde_fr` reads exactly 32 bytes (hex length checked) and `fr_from_bytes` rejects values ≥ the modulus; tests cover 33-byte and all-`ff` inputs. `serde_fr_vec` is only used behind `MerklePath`, whose `siblings` are an `UpperBoundedVec<Fr, MAX_MERKLE_PATH_SIBLINGS>` with a regression test for one-too-many. | `zk/groth16/src/serde.rs:16-22,49-59`; `mmr/src/path.rs:8-26`, `mmr/src/lib.rs:444-464` |
| RocksDB read-back | Recovery state is decoded with `from_bytes` and mapped to `RecoveryError` (`services/storage/src/recovery.rs:98-108`). The generic `StorageReplyReceiver::recv` still `expect`s on `from_bytes` (`services/storage/src/lib.rs:92-101`) and blocks and transactions share the unprefixed 32-byte key space; that is #79 (from the #63 report) and is not re-reported. | files cited |
| Query and path parameters | `Path<HeaderId>` / `Path<TxHash>` / `Path<ZkPublicKey>` are fixed-size hex; `BlocksStreamQuery` carries `validator` bounds on `blocks_limit` and `server_batch_size` and rejects zero via `NonZero<usize>`, with unit tests. | `nodes/api-common/src/queries.rs:13-70`; `nodes/node/binary/src/api/queries.rs:176-190` |
| FFI JSON | Host-supplied JSON goes through `serde_json::from_str` (128-deep limit) into the same bounded types as HTTP, then explicit `preverify()`. Host is trusted (PR #105). | `c-bindings/src/api/wallet.rs:1722,1795-1803` |
| Secret-bearing deserialisers (recorded for #65 / #129) | `UnsecuredEd25519Key` wraps its intermediate array in `Zeroizing` (`kms/keys/src/keys/ed25519/private.rs:67-75`). `secret_key_serde::deserialize` for the libp2p node key decodes into a plain `String` and `Vec<u8>`; libp2p zeroises the `Vec` slice on success but the hex `String` is dropped unzeroised, and the whole config also lives in a `serde_yaml::Value` tree (`yaml.rs:41-43`) for the duration of parsing. | `libp2p/src/config/mod.rs:70-76` |

---

## Appendix A — Definitions

### A.1 Severity

| Level | Definition |
|---|---|
| **Critical** | Loss of funds, chain halt, consensus split, or deanonymisation of users, exploitable by an unprivileged network participant with modest resources. |
| **High** | As above but requires significant resources, stake, timing, or a second weakness; or a remote crash/DoS of any node from a single unauthenticated peer. |
| **Medium** | Degrades safety/liveness/privacy guarantees under realistic conditions, or DoS requiring many peers / high cost; incorrect behaviour affecting a subset of users. |
| **Low** | Limited impact or unlikely preconditions; defence-in-depth gaps; reliability issues with a security flavour. |
| **Informational** | No immediate risk but relevant to best practice, maintainability, or future changes. |
| **Undetermined** | Needs more information from the team to rate. |

### A.2 Difficulty (to exploit)

| Level | Definition |
|---|---|
| **Low** | Well-known flaw; public tools exist or exploitation can be scripted. |
| **Medium** | Attacker must write an exploit or needs in-depth knowledge of the system. |
| **High** | Requires privileged access, complex technical details, or discovery of another weakness. |

### A.3 Categories

`Access Controls` · `Auditing and Logging` · `Authentication` · `Configuration` · `Cryptography` · `Data Exposure` · `Data Validation` · `Denial of Service` · `Error Reporting` · `Patching / Supply chain` · `Session Management` · `Timing` · `Undefined Behavior / Memory safety` · `Consensus` · `Economic / Incentive` · `Privacy / Anonymity` · `ZK Soundness` · `ZK Completeness` · `Determinism`
