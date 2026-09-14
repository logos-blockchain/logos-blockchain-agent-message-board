# Audit Report — KMS `unsafe` feature retired: one on-disk key type, three named exports, and `Debug` that redacts in every build

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/122`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `kms/keys` (`src/keys/{mod.rs, stored.rs, zk/{mod.rs, private.rs}, ed25519/{mod.rs, private.rs}}`, `tests/redaction.rs`, `Cargo.toml`), `kms/operators` (`src/ed25519/{mod.rs, derive_x25519.rs, libp2p_identity.rs}`, `src/zk/{mod.rs, voucher.rs}`, `Cargo.toml`), `services/key-management-system` (`src/backend/preload.rs`, `Cargo.toml`), `services/blend/src/{core, edge}`, `services/wallet/src/lib.rs`, `services/chain/chain-leader/Cargo.toml`, `nodes/node/binary/src/{cli/keys.rs, cli/config/{keystore.rs, init.rs, update.rs, migrate_0_1_2/config.rs}, cli/participate.rs, config/{mod.rs, kms/}}`, `c-bindings/src/api/keys.rs`, `tools/config/src/{blend.rs, consensus.rs, kms.rs, node.rs}`, `tests/testing_framework/src/{node/configs/{wallet.rs, dynamic.rs, postprocess.rs}, framework/local/provisioning.rs, workloads/transaction/workload.rs}`, `tests/src/{common/wallet/**, cucumber/steps/nodes/**, tests/mantle/sdp/ops.rs}`; read: `kms/macros/src/lib.rs`, `services/key-management-system/src/{lib.rs, api.rs, message.rs}`, `kms/operators/src/{zk/leader.rs, blend/poq.rs}`, `core/src/proofs/leader_claim_proof.rs`, `core/src/mantle/ops/leader_claim.rs`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md` (core), `key-types-and-generation.md`, `common-cryptographic-components.md` (area, per #17 and #122)
Date: 2026-09-13 — author: `agent (Claude)` — status: `fix-review`

---

## 1. Summary

- Overall assessment: all seven items of #122 are delivered as one proposed patch against the target commit (Appendix B, 62 files, +947/−501), and the `unsafe` feature no longer exists: `cargo tree -e features,no-dev -p logos-blockchain-node -i logos-blockchain-key-management-system-keys` shows zero `unsafe` edges (§4.7), against five at the target commit (PR #121 LB-001). What replaces it is what PR #121 said should: `Debug` on every secret-bearing key type is `<redacted>` unconditionally and a test formats each of them; the keystore file and `kms.backend.keys` are a dedicated on-disk type, `StoredKey`, in the exact encoding the files already use, and the hardened `Key` no longer implements `Serialize` at all; the three genuine raw uses each have a named, single-purpose type or operator — `Ed25519Key::libp2p_identity()` → `LibP2pIdentity` (consumed once by the Blend swarm), `Keystore::swarm_secret_key()` (the one secret the CLI copies into a config in the clear), and `ProveClaimOperator` (the proof of claim is now generated inside the KMS, off the KMS loop, so the voucher secret never leaves it); the blend services get the public key and the encryption key through the existing ungated API and no longer keep the raw signing key for the life of the process. The declined `generate-key` path prints an explicit hex export that round-trips through `add-key`, for both key types.
- Findings: 0 critical · 0 high · 0 medium · 0 low · 0 informational new; prior findings #85 LB-001, LB-003 and #123 LB-001 move to *Patched*, #85 LB-002 and #123 LB-002 partly (§4).
- Key themes: "the honest gate is a type with a loud name, not a cargo feature", "a key that has been loaded stays loaded: `Key` deserializes and never serializes", "what the wallet needs from the voucher master key is a commitment, a nullifier and a proof, and the KMS can hand out all three".
- Must-fix before launch: none from this issue. The remaining KMS boundary work is #123's (the leader ZK key in the chain-leader and Blend provers) and #128/#129's (zeroizing the witness copies).

### Item-by-item

| # | Item in #122 | Done | Where (§) |
|---|---|---|---|
| 1 | Unconditional `<redacted>` `Debug` for `ZkKey`, `Ed25519Key`; hand-written `Debug` for `UnsecuredZkKey`; a test formatting every secret type | yes | §4.1 |
| 2 | Replace `println!("Key: {key:?}")` with a hex export that round-trips through `--zk`/`--ed25519` | yes (export, with a round-trip test) | §4.2 |
| 3 | Blend core and edge: public key via `KMSMessage::PublicKey`, NEK via `DeriveX25519Operator`, `LeakSecretKeyOperator` replaced by a keypair export consumed once, `non_ephemeral_signing_key` dropped from `RunningBlendConfig` | yes | §4.3 |
| 4 | Wallet: voucher operator returning `(VoucherCm, VoucherNullifier)` and a `ProveClaimOperator` proving inside `execute` | yes; the prover is scheduled from `execute`, not awaited in it (#123 LB-003) | §4.4 |
| 5 | CLI keystore: no `into_unsecured()` round-trips; public-key getters where only the public key is needed; one named export for `NETWORK_SWARM` | yes | §4.5 |
| 6 | Dedicated on-disk type for the keystore and `kms.backend.keys`; `Key` never `Serialize` | yes | §4.6 |
| 7 | Feature bookkeeping: operators' request behind its own feature, chain-leader's request dropped, `testing` no longer aliases it, feature deleted or renamed | yes: deleted everywhere, nothing left to gate | §4.7 |

## 2. Scope

**In scope**

| Path (target-commit line numbers) | Notes |
|---|---|
| `kms/keys/Cargo.toml` L46; `src/keys/mod.rs` L27-L32; `zk/mod.rs` L27-L29, L63-L67, L76-L86, L100-L104; `zk/private.rs` L20-L22, L62-L72; `ed25519/mod.rs` L30-L32, L67-L71, L86-L96; `ed25519/private.rs` L16-L17, L42-L50 | the feature, what it gated, and the two unsecured types the on-disk form is built from |
| `kms/operators/Cargo.toml` L20, L28-L30; `src/ed25519/{mod.rs, exfiltrate_secret_key.rs, derive_x25519.rs}`; `src/zk/{mod.rs, voucher.rs}`; `src/blend/poq.rs` L57-L77 (the pattern) | the two exfiltrating operators and the two they become |
| `services/key-management-system/Cargo.toml` L29, L37; `src/backend/preload.rs` L28-L35, L237-L262; `src/lib.rs` L28-L34 (`Backend::Key: Debug`) | the settings type and the `Debug` bound the service imposes |
| `kms/macros/src/lib.rs` L116-L131 | the `UnsupportedKeyOperator` arm that formats the key (#123 LB-001) |
| `services/blend/src/core/mod.rs` L370-L420, L768-L780, L1788-L1800; `core/settings.rs` L30-L45; `core/backends/libp2p/settings.rs` L31-L42; `edge/mod.rs` L219-L275; `edge/settings.rs` L28-L42; `edge/handlers.rs` L68-L75; `edge/backends/mod.rs` L19-L29; `edge/backends/libp2p/{mod.rs L37-L52, settings.rs L19-L27}`; `core/tests/{mod.rs, utils.rs}`; `edge/tests/utils.rs` | every use of the raw NSK after the KMS hands it out |
| `services/wallet/src/lib.rs` L1001-L1031 (`sign_leader_claim`), L1173-L1190 (`generate_poc`), L1231-L1280 (`generate_new_voucher_secret`, `derive_voucher_from_kms`), L119-L125 (errors) | the `UnsafeVoucherOperator` caller and the prover it runs |
| `nodes/node/binary/src/cli/keys.rs` L115-L155, L232-L300; `cli/config/keystore.rs` (whole file); `cli/config/init.rs` L125-L225; `cli/config/update.rs` L100-L190; `cli/participate.rs` L43-L50, L88-L96; `cli/config/migrate_0_1_2/config.rs` L80-L93; `config/kms/{serde.rs, mod.rs}`; `config/mod.rs` L112-L133 | every `into_unsecured()` site and every place `Key: Serialize` was required |
| `c-bindings/src/api/keys.rs` L130-L187; `tools/config/src/{blend.rs, consensus.rs, kms.rs, node.rs}`; the testing-framework and cucumber files listed in the header | the other constructors of keys that end up in a config file |

**Out of scope**

`ZkKey::as_fr()` and the leader ZK key's path through the chain-leader and Blend provers (PR #121 LB-002, #123 LB-002/LB-003 and its `ProveLeadership` design); zeroizing the PoL/PoC witness copies (#128, #129); `recording`/permissions of the keystore file (#65); `PoQOperator`'s awaited `spawn_blocking` (#123 LB-003, not touched, though `ProveClaimOperator` follows the non-blocking shape #123 asks for); the `unsafe-test-functions` features of the Blend crates (#36). Third-party crates assumed correct: `ed25519-dalek`, `x25519-dalek`, `zeroize`, `libp2p-identity` (`ed25519_from_bytes` zeroes its input, as PR #121 S-005 verified), `serde_yaml`.

**Assumptions**

The PR #121 inventory (its LB-001 table: uses 1-5 and the two requesters that need nothing gated) is complete; this report's grep of `into_unsecured`, `LeakSecretKeyOperator`, `UnsafeVoucherOperator`, `Key::Zk(`, `Key::Ed25519(` and `HashMap<KeyId, Key>` across the workspace found no site outside it. The keystore and user-config files written by the target commit are the compatibility baseline: the patch must read them unchanged.

## 3. Method

- Working through the seven items of `#122` (parent `#17`, spun out of PR #121 LB-001/LB-003) on a detached worktree at the target commit, with PR #121's disposition per use and the #123 report's rule for operators (schedule the prover, do not await it in `execute`) as the design inputs.
- Spec conformance: `key-types-and-generation.md` § Non-ephemeral Signing Key ("used to authenticate the node on the network level") is why the NSK export stays an export and becomes `LibP2pIdentity` rather than an operation; § Non-ephemeral Encryption Key ("derived from the NSK") is why the NEK is a KMS operation (`DeriveX25519Operator`) and nothing else about the NSK is needed after start; `common-cryptographic-components.md` § ZkSignature (the secret key "must remain confidential") for the voucher master key.
- Every constructor of a config-bound key in the workspace was enumerated so `StoredKey` has a deliberate producer in each (§4.6): the CLI keystore, the migration, the c-bindings `add_key`, `tools/config`, the testing framework's provisioning/dynamic/post-processing, the cucumber steps.
- Automated tooling: `cargo check --workspace --all-targets`; `cargo check -p logos-blockchain-tests --all-targets --features cucumber,testing_framework,mantle_sdp,tip_poll_self_heal`; `cargo clippy --workspace --all-targets -- -D warnings` and the same for the tests crate with those features; `cargo +nightly fmt`; `cargo tree -e features,no-dev`; the unit tests in §4.8. Toolchain `rustc 1.98.1`, macOS arm64.
- Dynamic testing: the end-to-end leader-claim test of the repository (`tests/src/tests/mantle/leader.rs`) on a node built from the patch, which starts a Blend core node from a config holding `StoredKey`s, drives the wallet's voucher and proof-of-claim path through the two new operators, and checks the claim is included (§4.8).

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| #85 LB-001 | The `unsafe` feature is enabled by five plain `[dependencies]` entries and the node binary cannot be built without it | Configuration | Low | High | **Patched** (§4.7: the feature is gone) |
| #85 LB-003 | With the feature on, `Debug` of a `ZkKey` prints the secret scalar; the CLI's declined-generate path prints it and loses the Ed25519 key | Data Exposure | Low | High | **Patched** (§4.1, §4.2) |
| #123 LB-001 | A key/operator type mismatch puts the ZK secret scalar into the KMS error string, which is logged at `error` level | Data Exposure | Low | High | **Patched** as a consequence of §4.1: the macro's `format!("{key:?}")` now yields `Zk(ZkKey(<redacted>))` |
| #85 LB-002 / #123 LB-002 | `as_fr()` is ungated and the ZK secret leaves the KMS through `BuildPrivateInputsWithLeaderKey`; the voucher secret through `UnsafeVoucherOperator` | Data Exposure | Low | High | **Partly patched**: the voucher secret no longer leaves the KMS (§4.4); the leader key path is #123's |
| #85 S-005 | The blend services keep the raw NSK for the life of the process and clone it on every epoch | — | — | — | **Patched** (§4.3) |

### 4.1 Item 1 — `Debug` redacts unconditionally, with a test

`ZkKey` and `Ed25519Key` write `ZkKey(<redacted>)` / `Ed25519Key(<redacted>)` in every build (the `#[cfg(feature = "unsafe")]` branches at `zk/mod.rs` L78-L82 and `ed25519/mod.rs` L88-L92 are gone). `UnsecuredZkKey` (`zk/private.rs` L20) loses its derived `Debug`, which printed the scalar in decimal through `ark-ff`, and gets `UnsecuredZkKey(<redacted>)`; `UnsecuredEd25519Key` (`ed25519/private.rs` L16) likewise gets `UnsecuredEd25519Key(<redacted>)` instead of relying on `ed25519_dalek::SigningKey`'s `finish_non_exhaustive`. The new `StoredKey` (§4.6) prints `StoredKey::Zk(<redacted>)` / `StoredKey::Ed25519(<redacted>)`. `Key` keeps its derive, which now composes only redacting impls.

The test is `kms/keys/tests/redaction.rs`: it builds an Ed25519 key from `[0x22; 32]` and a ZK key from the scalar `12648430`, formats `UnsecuredEd25519Key`, `Ed25519Key`, `Key::Ed25519`, `StoredKey::Ed25519` and the four ZK counterparts with `{:?}`, and asserts each rendering contains `<redacted>` and none contains the secret in any of its renderings (hex, `{:?}` of the byte array, `{:?}` of the slice; decimal, `Display` and `Debug` of the `Fr`, hex of `fr_to_bytes`). It is an integration test, so it exercises the public API exactly as a log macro would.

**A finding closed on the way.** #123 LB-001 showed the `KmsEnumKey` macro building `KeyError::UnsupportedKeyOperator { key: format!("{key:?}") }` (`kms/macros/src/lib.rs` L124-L127) on a key/operator mismatch, and the service logging that error. `{key:?}` is `Key`'s derived `Debug`, so at the target commit the string carried `Zk(ZkKey(SecretKey(<decimal scalar>)))`; with this patch it carries `Zk(ZkKey(<redacted>))`. The macro is unchanged.

### 4.2 Item 2 — the declined `generate-key` path exports, and the export round-trips

`run_generate_key` (`cli/keys.rs` L279-L300) kept the "declined" branch that generated a key and printed `Key: {key:?}`: a decimal scalar `--zk` could not take back, or, for Ed25519, the public key only, with the secret discarded (PR #121 LB-003). It now prints the key id, the keystore's `WARNING` line with the exact `add-key --ed25519|--zk` invocation to keep the key, and `export_secret_hex(key)`: the 32 secret bytes for Ed25519, `fr_to_bytes` of the scalar for ZK, both hex. Those are the encodings `parse_hex_ed25519_key` and `parse_hex_zk_key` (`config/mod.rs` L564-L577) read, so `add-key` accepts the printed value for both types; `keystore::tests::exported_secret_hex_round_trips_through_the_add_key_parsers` proves it for a freshly generated key of each type. The export is returned as a `Zeroizing<String>` and is the only function in the node binary that renders a secret; it lives next to the keystore, which is the only caller.

`AddKeyArgs` (`cli/keys.rs` L85-L155) still derives `Debug` for clap and still holds an `Option<UnsecuredZkKey>` / `Option<UnsecuredEd25519Key>`; with §4.1 a `{args:?}` now prints `<redacted>` for both.

### 4.3 Item 3 — the Blend services

At the target commit both services ran `LeakSecretKeyOperator` at start (`core/mod.rs` L379-L392, `edge/mod.rs` L223-L235), stored the `UnsecuredEd25519Key` in `RunningBlendConfig` (`core/settings.rs` L38, `edge/settings.rs` L34), and used it for three things: the public key (`core/mod.rs` L398; `edge/mod.rs` L238, L244), the NEK on every epoch (`core/mod.rs` L775, L1794), and the libp2p keypair (`core/backends/libp2p/settings.rs` L32-L37; `edge/backends/libp2p/mod.rs` L48-L51).

Now, at start, each service does three KMS round trips on the signing key id and keeps only the results:

- the public key through `KmsServiceApi::public_key` (`KMSMessage::PublicKey`), matched as `PublicKeyEncoding::Ed25519` the way the same file already matches the ZK key (`core/mod.rs` L371-L377);
- the NEK through `DeriveX25519Operator` (which gains the `new` constructor it lacked: it had no caller at the target commit), stored as `RunningBlendConfig::non_ephemeral_encryption_key: X25519PrivateKey` and cloned into each epoch's `EpochCryptographicProcessorSettings` instead of re-derived (`core/mod.rs` L775, L1794; the 15 test fixtures in `core/tests/mod.rs` follow);
- the swarm identity through `LibP2pIdentityOperator`, a new operator that sends `Ed25519Key::libp2p_identity()`: a `LibP2pIdentity(Zeroizing<[u8; 32]>)` with no `Clone`, no `Debug`, no serde and one method, `into_bytes`, which the service feeds to `Keypair::ed25519_from_bytes` once (that call zeroes its input) and stores as `RunningBlendConfig::swarm_identity: Keypair`. `BlendConfig<Libp2pBlendBackendSettings>::keypair()` returns a clone of it; the edge `BlendBackend::new` takes the `Keypair` instead of the key.

`RunningBlendConfig` in both services has no field of a secret-key type any more; the doc comment that described it as "the secret key exfiltrated from the KMS" is replaced. `LeakSecretKeyOperator` is deleted. The `Keypair` is the one raw-bytes holder left, which is the NSK's job by `key-types-and-generation.md` § Non-ephemeral Signing Key (network-level authentication), and it is a libp2p type the swarm needs in-process; PR #121's table (use 1(c)) already said this part cannot become an operation.

The edge service's public-key fetch uses `?` rather than `expect`, since `edge/mod.rs`'s `init` returns a `Result`; the core service's `init` panics on KMS failure as it did before.

### 4.4 Item 4 — the wallet's voucher path

`UnsafeVoucherOperator` returned the voucher secret `zkhash(master_sk, index)` (`voucher.rs` L37) to the wallet, which derived the commitment and nullifier from it (`lib.rs` L1017, L1245-L1246) and, on a claim, ran the PoC prover on it in its own `spawn_blocking` (`lib.rs` L1024-L1027, "TODO: This should happen in KMS"). Two operators replace it, in `kms/operators/src/zk/voucher.rs`, both ungated by construction:

- `VoucherOperator { index }` → `(VoucherCm, VoucherNullifier)`. The secret is computed and consumed inside `execute`.
- `ProveClaimOperator { index, voucher_path: MerklePath, voucher_root: Fr, mantle_tx_hash: Fr }` → `Result<Groth16LeaderClaimProof, ProveClaimError>`. `execute` clones the `ZkKey` (it is `Clone + ZeroizeOnDrop`), schedules `prove_claim` on the blocking pool under the task name `logos/kms/leader-claim-proof-blocking`, and returns without awaiting the join handle; the closure recomputes the secret, builds `LeaderClaimPublic` (the nullifier from the secret, the root and the tx hash from the caller), `LeaderClaimPrivate::try_new`, `Groth16LeaderClaimProof::prove`, drops the key, and sends the result on the operator's channel. This is the shape #123 §5 item 2 prescribes so that the KMS loop is not held for the proving time (its LB-003), rather than the awaited `spawn_blocking` of `PoQOperator`. `ProveClaimError` wraps the two errors the prover can return (`MerklePathWitnessError`, `leader_claim_proof::Error`); `kms/operators` gains `lb-mmr` (for the path type) and `thiserror`.

The wallet's `sign_leader_claim` becomes: `VoucherOperator` for the commitment → `voucher_path_snapshot(tip, &cm)` → `ProveClaimOperator` → `OpProof::PoC`. `generate_new_voucher_secret` uses `VoucherOperator` and stores the pair. `derive_voucher_from_kms`, `generate_poc`, the `VoucherSecret`, `LeaderClaimPrivate`, `LeaderClaimPublic`, `Groth16LeaderClaimProof` and `MerklePath` imports and the `PoCGenerationFailed` / `MerklePathWitness` error variants are gone from the wallet; `WalletServiceError::ProveClaim(#[from] ProveClaimError)` replaces the last two. The stale TODO at `voucher.rs` L13-L14 (PR #121 S-002) is gone with the file it was in.

### 4.5 Item 5 — the CLI keystore

`Keystore` (`cli/config/keystore.rs`) holds `HashMap<KeyTitle, StoredKey>` (§4.6). Its API is now: `set`, `get`, `get_all`, `remove` over `StoredKey`; `public_zk(title)` and `zk_public_keys()` for the public halves; `swarm_secret_key()` for the one named export; `generate_ed25519` / `generate_zk` returning the key id. `get_ed25519`, `get_zk` and `get_all_zk`, which each cloned a key and called `into_unsecured()` (L98, L113, L124), are gone, as are the two `into_unsecured()` calls of `generate_*` (L134, L143) and the two of `AddKeyArgs::new` (`cli/keys.rs` L127-L128), which unwrapped a `Key` only for `key()` to wrap it again; `AddKeyArgs::new` takes a `&StoredKey` and `key()` returns one.

Callers: `init.rs` and `update.rs` obtain `funding_pk` for cryptarchia and SDP through `public_zk`, `known_keys` through `zk_public_keys`, and `network.backend.swarm.node_key` through `swarm_secret_key()`, which returns the `lb_libp2p::ed25519::SecretKey` the network config stores (the target commit did `*unsecured_key.as_bytes()` into a stack array in both files, L129-L134 and L109-L114); `participate.rs` obtains its four public keys through `public_zk`. The migration (`migrate_0_1_2/config.rs` L86-L93) builds the swarm entry as `StoredKey::Ed25519(UnsecuredEd25519Key::from_bytes(..))`; the other five entries it copies were already the config's own key values and stay `StoredKey`s.

`generate_zk` uses `UnsecuredZkKey::from_rng(&mut OsRng)` (`zk/private.rs` L68-L72) instead of the local `generate_zk_key_from_random_bytes` (L184-L188), which built a `BigUint` from random bytes and reduced it; the two are the same sampler (`fr_from_mod_bytes`), so this is a de-duplication, and it removes the file's `num-bigint` use. Three tests cover the export round trip (§4.2), that `swarm_secret_key()` is the `NetworkSwarm` entry, and that the public getters agree with the stored keys and reject the wrong type.

### 4.6 Item 6 — `StoredKey`, and `Key` never `Serialize`

`kms/keys/src/keys/stored.rs` adds

```rust
#[derive(Clone, Deserialize, Serialize, ZeroizeOnDrop, PartialEq, Eq)]
pub enum StoredKey { Ed25519(UnsecuredEd25519Key), Zk(UnsecuredZkKey) }
impl StoredKey { pub fn public_key(&self) -> PublicKeyEncoding }
impl From<StoredKey> for Key            // loading; no impl in the other direction
impl From<UnsecuredEd25519Key> for StoredKey; impl From<UnsecuredZkKey> for StoredKey
```

with a redacting `Debug`. Its serde encoding is byte-for-byte the one `Key` had: `Key::Ed25519(Ed25519Key(UnsecuredEd25519Key))` and `Key::Zk(ZkKey(UnsecuredZkKey))` serialised through transparent newtypes to `!Ed25519 <hex bytes>` / `!Zk <hex scalar>`, and `StoredKey`'s variants wrap the same inner types under the same names. `stored::tests::stored_key_yaml_is_the_keystore_encoding` serialises a `StoredKey`, checks the tag, reloads it as a `StoredKey` *and* as a `Key` (which keeps `Deserialize`) and compares public keys, so an existing `keystore.yaml` or `user_config.yaml` reads unchanged, and `kms.backend.keys` written by the patched CLI reads on the target commit. The `#[cfg_attr(feature = "unsafe", derive(serde::Serialize))]` on `Key`, `Ed25519Key` and `ZkKey` (`keys/mod.rs` L28, `ed25519/mod.rs` L31, `zk/mod.rs` L28) and the `any(test, feature = "unsafe")` one on `PreloadKMSBackendSettings` (`preload.rs` L32) are deleted; there is no `Serialize` impl for a hardened key anywhere, so `UserConfig`, `RunConfig` (`#[cfg(feature = "testing")]`) and the keystore serialise `StoredKey`s or do not compile.

Where the type lands: `config::kms::serde::PreloadKmsBackendSettings { keys: HashMap<KeyId, StoredKey> }`, converted key by key into the service's `PreloadKMSBackendSettings { keys: HashMap<KeyId, Key> }` in `config/kms/mod.rs` (the service type keeps `Deserialize` only, and its test now loads a serialised map of `StoredKey`s); `UserConfig::blend_provider_id` / `blend_zk_key` (`config/mod.rs` L112-L133) match on `StoredKey`; the `Keystore` (§4.5); `AddKeyArgs` and the c-bindings' `parse_key_hex` (`c-bindings/src/api/keys.rs` L173-L187), which builds a `StoredKey` directly from the parsed unsecured key instead of a `Key` it then had to unwrap.

The producers outside the node are the test tooling, and this is the one place the patch reaches beyond the node's own code: `tools/config` derives keys from seeds (`consensus.rs` L398-L428, `blend.rs` L28-L30) and held them as `ZkKey` / `Ed25519Key`, which after this patch cannot be written into a config. `GeneralConsensusConfig`, `ServiceNote`, `ProviderInfo`, `GeneralBlendConfig`, the testing framework's `WalletAccount` and the cucumber steps hold `UnsecuredZkKey` / `UnsecuredEd25519Key` instead, which the keys crate documents as the type "to be used in contexts where a KMS-like key is required, but it's not possible to go through the KMS roundtrip" (`zk/private.rs` L16-L19); every method they used (`to_public_key`, `public_key`, `multi_sign`, `sign_payload`, `from_bytes`, `From<BigUint>`) exists on the unsecured types under the same name, and `key_id_for_preload_backend` takes a `&StoredKey`. The change is type-level only; no test logic changed.

### 4.7 Item 7 — the feature is deleted

After §4.1-§4.6 nothing is gated: the operators crate's `exfiltrate_secret_key` and `voucher` modules are replaced by ungated ones, the keys crate has no `into_unsecured` and no conditional derive. So instead of moving the operators' request behind its own feature, the feature is removed from `kms/keys/Cargo.toml` (L46), `kms/operators/Cargo.toml` (L20 request, L30 definition), `services/key-management-system/Cargo.toml` (L29 dev-dependency request, L37 definition), and the requests are dropped from `nodes/node/binary/Cargo.toml` L34, `services/blend/Cargo.toml` L27, `services/wallet/Cargo.toml` L26 and `services/chain/chain-leader/Cargo.toml` L25. The node's `testing` feature (`Cargo.toml` L96) becomes `testing = []`: it still gates `RunConfig`'s serde derives and is still what CI and the test crates enable, so it stays.

The check the issue asks for, on the patched worktree:

```text
$ cargo tree -e features,no-dev -p logos-blockchain-node -i logos-blockchain-key-management-system-keys | grep -c unsafe
0
$ cargo tree -e features,no-dev -p logos-blockchain-node -i logos-blockchain-key-management-system-keys | grep feature | sort -u
logos-blockchain-key-management-system-keys feature "openapi"
… (chain-service, core, network-service, tx-service "openapi"; storage/tx-service "rocksdb-backend"; node "default")
```

Against the five `feature "unsafe"` edges PR #121 LB-001 printed at the same commit. A `grep -rn unsafe --include=Cargo.toml --include='*.rs'` over `kms/`, `services/key-management-system`, `services/blend`, `services/wallet`, `services/chain/chain-leader` and `nodes/node/binary` returns only the Rust keyword, the Blend crates' `unsafe-test-functions` and one unrelated comment (`tokio_provider.rs` L34, #60).

### 4.8 Verification

| What | Command (worktree at `a805329f8` + Appendix B) | Result |
|---|---|---|
| Type check, all crates and targets | `cargo check --workspace --all-targets` | clean |
| Type check, tests crate with the CI feature sets | `cargo check -p logos-blockchain-tests --all-targets --features cucumber,testing_framework,mantle_sdp,tip_poll_self_heal` | clean |
| Lints | `cargo clippy --workspace --all-targets -- -D warnings`; the same for the tests crate with those features | clean, both |
| Format | `cargo +nightly-2026-01-18 fmt --all -- --check` | clean |
| Feature graph | `cargo tree -e features,no-dev -p logos-blockchain-node -i logos-blockchain-key-management-system-keys` | 0 `unsafe` edges (§4.7) |
| KMS crates | `cargo test -p logos-blockchain-key-management-system-{keys,operators,service}` | keys 12 unit + 2 redaction (new), service 3 (1 rewritten) |
| Node config and CLI | `cargo test -p logos-blockchain-node --lib -- config:: cli::` | 22 passed (3 new keystore tests, 1 rewritten config test) |
| Config tool, testing framework | `cargo test -p logos-blockchain-config -p testing-framework` | 7 + 15 passed |
| Blend service | `cargo test -p logos-blockchain-blend-service --lib` | 94 passed, 3 ignored (the 15 adapted core fixtures and both edge fixtures included) |
| Wallet service | `cargo test -p logos-blockchain-wallet-service --lib` | 7 passed |
| End to end, real nodes | `LOGOS_BLOCKCHAIN_NODE_BIN=<release build of the patch, --features testing> cargo test -p logos-blockchain-tests --test test_mantle_leader` | `leader_claim` and `concurrent_leader_claims` passed (27.5 s): nodes start from configs holding `StoredKey`s, the Blend core comes up through `DeriveX25519Operator` + `LibP2pIdentityOperator`, the wallet's `/leader/claim` goes through `VoucherOperator` and `ProveClaimOperator`, and the claim transaction is included |

The end-to-end run is the check that matters for items 3, 4 and 6 together: a node cannot reach a claim without loading its keys from the on-disk type, joining Blend with the exported identity, and proving the claim inside the KMS.

### 4.9 Not done, and why

- **`ZkKey::as_fr()`** stays public: the three remaining operators (`CheckLotteryWinning`, `BuildPrivateInputsWithLeaderKey`, `PoQOperator`) and the two new ones need it, and #123 §5 item 1 explains why `pub(crate)` is not available across the `kms/keys` ↔ `kms/operators` split; its rename-and-guard proposal belongs with the leader-key work there.
- **The witness copies** inside `LeaderClaimPrivate` / `PoCWalletInputsData` are still `Copy` `Fr`s and not zeroized; `ProveClaimOperator` confines them to one blocking closure, which is what #123 LB-002's long-term design asks for, and #129's patch (`Debug` redaction on those types) applies on top. The two patches touch nine files in common (`Cargo.lock`, `kms/keys/src/keys/zk/{mod.rs, private.rs}`, `keystore.rs`, `config/mod.rs`, `tools/config/src/{blend.rs, consensus.rs}`, `tests/testing_framework/src/node/configs/wallet.rs`, `tests/src/tests/mantle/sdp/ops.rs`) and `git apply --check` of #129 on top of this one fails in five of them: both rewrite the same `ZkKey::from(BigUint::from_bytes_le(..))` lines (#129 to a checked `Fr`, this one to the unsecured type). Whichever lands second rebases those lines to `UnsecuredZkKey::new(fr)`; nothing else overlaps.
- **`PoQOperator`** still awaits its `spawn_blocking` inside `execute` (#123 LB-003); the new operator deliberately does not, and changing the existing one is #123's item.
- **A CLI `export-key` command** (the inverse of `add-key`) was not added; `export_secret_hex` exists and is tested, so it is a ten-line addition if wanted.

## 5. Suggestions (non-security)

### S-001 · Print the key's public id on the declined path in the same line as the export

`run_generate_key` prints `KeyID:` and the export on separate lines; a single line `add-key --zk <hex> --title <title>` that can be pasted as-is would remove the last way to lose the key the operator asked for.

### S-002 · Let `KmsServiceApi::public_key` say which key type it returned

Both Blend services and `UserConfig::blend_provider_id` match the returned `PublicKeyEncoding` and fail on the other variant with their own message. A typed accessor (`public_ed25519_key(id)`, `public_zk_key(id)`) on the API would fold the three matches into one error.

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

## Appendix B — The patch

Against `logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b`; applies with `git apply`. `Cargo.lock` changes because `kms/operators` gains `lb-mmr` and `thiserror` and the node binary gains `zeroize`, all workspace members or already-resolved crates; no new external dependency.

~~~~diff
diff --git a/Cargo.lock b/Cargo.lock
index 0ecedfd37..c34994698 100644
--- a/Cargo.lock
+++ b/Cargo.lock
@@ -4695,9 +4695,11 @@ dependencies = [
  "logos-blockchain-groth16",
  "logos-blockchain-key-management-system-keys",
  "logos-blockchain-log-targets",
+ "logos-blockchain-mmr",
  "logos-blockchain-poseidon2",
  "logos-blockchain-utils",
  "logos-blockchain-utxotree",
+ "thiserror 2.0.18",
  "tokio",
  "tracing",
 ]
@@ -4915,6 +4917,7 @@ dependencies = [
  "utoipa",
  "utoipa-swagger-ui",
  "validator",
+ "zeroize",
 ]
 
 [[package]]
diff --git a/c-bindings/src/api/keys.rs b/c-bindings/src/api/keys.rs
index 943365d58..bd040c8a2 100644
--- a/c-bindings/src/api/keys.rs
+++ b/c-bindings/src/api/keys.rs
@@ -1,6 +1,6 @@
 use std::ffi::{CStr, CString, c_char};
 
-use lb_key_management_system_keys::keys::{Ed25519Key, Key, UnsecuredEd25519Key, ZkKey};
+use lb_key_management_system_keys::keys::{StoredKey, UnsecuredEd25519Key};
 use lb_node::cli::keys::{
     AddKeyArgs, GenerateKeyArgs, KeyType as NodeKeyType, RemoveKeyArgs, run_add_key, run_remove_key,
 };
@@ -169,8 +169,8 @@ pub unsafe extern "C" fn add_key(
     }
 }
 
-/// Parses a hex-encoded secret key of the given type into a [`Key`].
-fn parse_key_hex(key_type: KeyType, key_hex: &str) -> Result<Key, String> {
+/// Parses a hex-encoded secret key of the given type into a [`StoredKey`].
+fn parse_key_hex(key_type: KeyType, key_hex: &str) -> Result<StoredKey, String> {
     match key_type {
         KeyType::Ed25519 => {
             let secret_key = lb_node::config::parse_hex_ed25519_key(key_hex)?;
@@ -178,11 +178,11 @@ fn parse_key_hex(key_type: KeyType, key_hex: &str) -> Result<Key, String> {
                 .as_ref()
                 .try_into()
                 .map_err(|_| "Invalid ed25519 secret key length".to_owned())?;
-            Ok(Ed25519Key::from(UnsecuredEd25519Key::from_bytes(&bytes)).into())
+            Ok(StoredKey::Ed25519(UnsecuredEd25519Key::from_bytes(&bytes)))
         }
         KeyType::Zk => {
             let zk_key = lb_node::config::parse_hex_zk_key(key_hex)?;
-            Ok(ZkKey::from(zk_key).into())
+            Ok(StoredKey::Zk(zk_key))
         }
     }
 }
diff --git a/kms/keys/Cargo.toml b/kms/keys/Cargo.toml
index 147243454..10c3091bb 100644
--- a/kms/keys/Cargo.toml
+++ b/kms/keys/Cargo.toml
@@ -43,4 +43,3 @@ serde_yaml = { workspace = true }
 
 [features]
 openapi = ["dep:utoipa", "lb-utils/openapi"]
-unsafe  = []
diff --git a/kms/keys/src/keys/ed25519/mod.rs b/kms/keys/src/keys/ed25519/mod.rs
index a8ab4f550..e228774ed 100644
--- a/kms/keys/src/keys/ed25519/mod.rs
+++ b/kms/keys/src/keys/ed25519/mod.rs
@@ -6,7 +6,7 @@ use ed25519_dalek::SigningKey;
 use lb_codec::{BinaryDecode, BinaryEncode, DecodeError};
 use rand_core::CryptoRngCore;
 use serde::Deserialize;
-use zeroize::ZeroizeOnDrop;
+use zeroize::{ZeroizeOnDrop, Zeroizing};
 
 use crate::keys::{errors::KeyError, secured_key::SecuredKey};
 
@@ -28,9 +28,24 @@ pub use self::{
 /// It is a secured variant of a [`UnsecuredEd25519Key`] and used within the set
 /// of supported KMS keys.
 #[derive(Deserialize, ZeroizeOnDrop, Clone, PartialEq, Eq)]
-#[cfg_attr(feature = "unsafe", derive(serde::Serialize))]
 pub struct Ed25519Key(UnsecuredEd25519Key);
 
+/// The raw bytes of a non-ephemeral signing key, exported once so a libp2p
+/// swarm can present the node's network identity in its handshake.
+///
+/// This is the one raw export the hardened key offers. It has no `Clone`, no
+/// `Debug` and no serde; the only thing to do with it is
+/// [`into_bytes`](Self::into_bytes), which the Blend backends feed to
+/// `libp2p::identity::Keypair::ed25519_from_bytes` (which zeroes its input).
+pub struct LibP2pIdentity(Zeroizing<[u8; ED25519_SECRET_KEY_SIZE]>);
+
+impl LibP2pIdentity {
+    #[must_use]
+    pub fn into_bytes(self) -> Zeroizing<[u8; ED25519_SECRET_KEY_SIZE]> {
+        self.0
+    }
+}
+
 impl Ed25519Key {
     #[must_use]
     pub const fn new(signing_key: SigningKey) -> Self {
@@ -64,10 +79,11 @@ impl Ed25519Key {
         self.0.derive_x25519()
     }
 
-    #[cfg(feature = "unsafe")]
+    /// Exports the key as the node's libp2p identity. See [`LibP2pIdentity`]
+    /// for why this export exists and what may consume it.
     #[must_use]
-    pub fn into_unsecured(self) -> UnsecuredEd25519Key {
-        self.0.clone()
+    pub fn libp2p_identity(&self) -> LibP2pIdentity {
+        LibP2pIdentity(self.0.to_bytes())
     }
 }
 
@@ -85,13 +101,7 @@ impl From<SigningKey> for Ed25519Key {
 
 impl Debug for Ed25519Key {
     fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
-        #[cfg(feature = "unsafe")]
-        write!(f, "Ed25519Key({:?})", self.0)?;
-
-        #[cfg(not(feature = "unsafe"))]
-        write!(f, "Ed25519Key(<redacted>)")?;
-
-        Ok(())
+        f.write_str("Ed25519Key(<redacted>)")
     }
 }
 
diff --git a/kms/keys/src/keys/ed25519/private.rs b/kms/keys/src/keys/ed25519/private.rs
index 4ba48768f..ade3e4dc4 100644
--- a/kms/keys/src/keys/ed25519/private.rs
+++ b/kms/keys/src/keys/ed25519/private.rs
@@ -13,9 +13,15 @@ pub const KEY_SIZE: usize = SECRET_KEY_LENGTH;
 ///
 /// To be used in contexts where a KMS-like key is required, but it's not
 /// possible to go through the KMS roundtrip of executing operators.
-#[derive(ZeroizeOnDrop, Clone, Debug)]
+#[derive(ZeroizeOnDrop, Clone)]
 pub struct UnsecuredEd25519Key(pub(super) SigningKey);
 
+impl core::fmt::Debug for UnsecuredEd25519Key {
+    fn fmt(&self, f: &mut core::fmt::Formatter<'_>) -> core::fmt::Result {
+        f.write_str("UnsecuredEd25519Key(<redacted>)")
+    }
+}
+
 impl UnsecuredEd25519Key {
     #[must_use]
     pub fn from_bytes(bytes: &[u8; SECRET_KEY_LENGTH]) -> Self {
diff --git a/kms/keys/src/keys/mod.rs b/kms/keys/src/keys/mod.rs
index 0c53aab22..32a38359f 100644
--- a/kms/keys/src/keys/mod.rs
+++ b/kms/keys/src/keys/mod.rs
@@ -2,6 +2,7 @@ pub mod errors;
 pub mod secured_key;
 
 mod ed25519;
+mod stored;
 mod zk;
 
 use lb_key_management_system_macros::KmsEnumKey;
@@ -11,9 +12,10 @@ use zeroize::ZeroizeOnDrop;
 pub use crate::keys::{
     ed25519::{
         ED25519_PUBLIC_KEY_SIZE, ED25519_SECRET_KEY_SIZE, ED25519_SIGNATURE_SIZE, Ed25519Key,
-        PublicKey as Ed25519PublicKey, SharedKey, Signature as Ed25519Signature,
+        LibP2pIdentity, PublicKey as Ed25519PublicKey, SharedKey, Signature as Ed25519Signature,
         UnsecuredEd25519Key, X25519PrivateKey, X25519PublicKey,
     },
+    stored::StoredKey,
     zk::{
         MAX_ZK_SIGNING_KEYS, PublicKey as ZkPublicKey, PublicKeys as ZkPublicKeys,
         Signature as ZkSignature, UnsecuredZkKey, ZkKey, public_inputs_from_pks,
@@ -23,9 +25,9 @@ pub use crate::keys::{
 /// Entity that gathers all keys provided by the KMS crate.
 ///
 /// Works as a [`SecuredKey`] over [`Encoding`], delegating requests to the
-/// appropriate key.
+/// appropriate key. It deserializes from the on-disk [`StoredKey`] encoding
+/// but never serializes: a key that has been loaded stays loaded.
 #[derive(Deserialize, ZeroizeOnDrop, PartialEq, Eq, Clone, Debug, KmsEnumKey)]
-#[cfg_attr(feature = "unsafe", derive(serde::Serialize))]
 pub enum Key {
     Ed25519(Ed25519Key),
     Zk(ZkKey),
diff --git a/kms/keys/src/keys/stored.rs b/kms/keys/src/keys/stored.rs
new file mode 100644
index 000000000..1f990e501
--- /dev/null
+++ b/kms/keys/src/keys/stored.rs
@@ -0,0 +1,101 @@
+use core::fmt::{self, Debug, Formatter};
+
+use serde::{Deserialize, Serialize};
+use zeroize::ZeroizeOnDrop;
+
+use crate::keys::{Ed25519Key, Key, PublicKeyEncoding, UnsecuredEd25519Key, UnsecuredZkKey, ZkKey};
+
+/// The on-disk form of a KMS key: what the keystore file and the node's
+/// `kms.backend.keys` section hold.
+///
+/// This is the only serialisable form of a secret key. A [`Key`] is built
+/// from it when the KMS loads and never converts back, so a hardened key
+/// cannot reach a file, a log line or a wire through serde. The encoding is
+/// the one the keystore has always used: an externally tagged variant over
+/// the hex-encoded secret bytes.
+#[derive(Clone, Deserialize, Serialize, ZeroizeOnDrop, PartialEq, Eq)]
+pub enum StoredKey {
+    Ed25519(UnsecuredEd25519Key),
+    Zk(UnsecuredZkKey),
+}
+
+impl StoredKey {
+    #[must_use]
+    pub fn public_key(&self) -> PublicKeyEncoding {
+        match self {
+            Self::Ed25519(key) => PublicKeyEncoding::Ed25519(key.public_key()),
+            Self::Zk(key) => PublicKeyEncoding::Zk(key.to_public_key()),
+        }
+    }
+}
+
+impl Debug for StoredKey {
+    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
+        match self {
+            Self::Ed25519(_) => f.write_str("StoredKey::Ed25519(<redacted>)"),
+            Self::Zk(_) => f.write_str("StoredKey::Zk(<redacted>)"),
+        }
+    }
+}
+
+impl From<StoredKey> for Key {
+    fn from(value: StoredKey) -> Self {
+        match &value {
+            StoredKey::Ed25519(key) => Self::Ed25519(Ed25519Key::from(key.clone())),
+            StoredKey::Zk(key) => Self::Zk(ZkKey::from(key.clone())),
+        }
+    }
+}
+
+impl From<UnsecuredEd25519Key> for StoredKey {
+    fn from(value: UnsecuredEd25519Key) -> Self {
+        Self::Ed25519(value)
+    }
+}
+
+impl From<UnsecuredZkKey> for StoredKey {
+    fn from(value: UnsecuredZkKey) -> Self {
+        Self::Zk(value)
+    }
+}
+
+#[cfg(test)]
+mod tests {
+    use lb_groth16::Fr;
+
+    use super::*;
+    use crate::keys::secured_key::SecuredKey as _;
+
+    fn sample_keys() -> [StoredKey; 2] {
+        [
+            StoredKey::Ed25519(UnsecuredEd25519Key::from_bytes(&[0x22; 32])),
+            StoredKey::Zk(UnsecuredZkKey::new(Fr::from(12_648_430u64))),
+        ]
+    }
+
+    #[test]
+    fn stored_key_loads_into_the_same_hardened_key() {
+        for stored in sample_keys() {
+            let expected = stored.public_key();
+            let key = Key::from(stored);
+            assert_eq!(key.as_public_key(), expected);
+        }
+    }
+
+    #[test]
+    fn stored_key_yaml_is_the_keystore_encoding() {
+        for stored in sample_keys() {
+            let yaml = serde_yaml::to_string(&stored).expect("stored key serializes");
+            let expected_tag = match &stored {
+                StoredKey::Ed25519(_) => "!Ed25519",
+                StoredKey::Zk(_) => "!Zk",
+            };
+            assert!(yaml.starts_with(expected_tag), "{yaml}");
+
+            let reloaded: StoredKey = serde_yaml::from_str(&yaml).expect("round trip");
+            assert_eq!(reloaded, stored);
+            let hardened: Key = serde_yaml::from_str(&yaml).expect("loads as a hardened key");
+            assert_eq!(hardened.as_public_key(), stored.public_key());
+        }
+    }
+}
diff --git a/kms/keys/src/keys/zk/mod.rs b/kms/keys/src/keys/zk/mod.rs
index be5f65cb6..6e8f89265 100644
--- a/kms/keys/src/keys/zk/mod.rs
+++ b/kms/keys/src/keys/zk/mod.rs
@@ -25,7 +25,6 @@ pub use self::signature::Signature;
 /// It is a secured variant of a [`UnsecuredZkKey`] and used within the set of
 /// supported KMS keys.
 #[derive(Deserialize, ZeroizeOnDrop, Clone, PartialEq, Eq)]
-#[cfg_attr(feature = "unsafe", derive(serde::Serialize))]
 pub struct ZkKey(UnsecuredZkKey);
 
 impl ZkKey {
@@ -59,12 +58,6 @@ impl ZkKey {
     pub fn to_public_key(&self) -> PublicKey {
         self.0.to_public_key()
     }
-
-    #[cfg(feature = "unsafe")]
-    #[must_use]
-    pub fn into_unsecured(self) -> UnsecuredZkKey {
-        self.0.clone()
-    }
 }
 
 impl Hash for ZkKey {
@@ -75,13 +68,7 @@ impl Hash for ZkKey {
 
 impl Debug for ZkKey {
     fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
-        #[cfg(feature = "unsafe")]
-        write!(f, "ZkKey({:?})", self.0)?;
-
-        #[cfg(not(feature = "unsafe"))]
-        write!(f, "ZkKey(<redacted>)")?;
-
-        Ok(())
+        f.write_str("ZkKey(<redacted>)")
     }
 }
 
diff --git a/kms/keys/src/keys/zk/private.rs b/kms/keys/src/keys/zk/private.rs
index 01bf4f3c9..66a700b91 100644
--- a/kms/keys/src/keys/zk/private.rs
+++ b/kms/keys/src/keys/zk/private.rs
@@ -17,10 +17,16 @@ static KDF: LazyLock<Fr> = LazyLock::new(|| fr_from_bytes_unchecked(b"KDF"));
 ///
 /// To be used in contexts where a KMS-like key is required, but it's not
 /// possible to go through the KMS roundtrip of executing operators.
-#[derive(ZeroizeOnDrop, Clone, Debug, Serialize, Deserialize)]
+#[derive(ZeroizeOnDrop, Clone, Serialize, Deserialize)]
 #[serde(transparent)]
 pub struct SecretKey(#[serde(with = "lb_groth16::serde::serde_fr")] Fr);
 
+impl core::fmt::Debug for SecretKey {
+    fn fmt(&self, f: &mut core::fmt::Formatter<'_>) -> core::fmt::Result {
+        f.write_str("UnsecuredZkKey(<redacted>)")
+    }
+}
+
 impl SecretKey {
     #[must_use]
     pub const fn zero() -> Self {
diff --git a/kms/keys/tests/redaction.rs b/kms/keys/tests/redaction.rs
new file mode 100644
index 000000000..0df22cd5d
--- /dev/null
+++ b/kms/keys/tests/redaction.rs
@@ -0,0 +1,65 @@
+//! Every secret-bearing type formats as `<redacted>`, in every build.
+
+use lb_groth16::{Fr, fr_to_bytes};
+use logos_blockchain_key_management_system_keys::keys::{
+    Ed25519Key, Key, StoredKey, UnsecuredEd25519Key, UnsecuredZkKey, ZkKey,
+};
+
+const ED25519_SECRET: [u8; 32] = [0x22; 32];
+const ZK_SECRET: u64 = 12_648_430;
+
+fn ed25519_secret_renderings() -> Vec<String> {
+    vec![
+        hex::encode(ED25519_SECRET),
+        format!("{ED25519_SECRET:?}"),
+        format!("{:?}", &ED25519_SECRET[..]),
+    ]
+}
+
+fn zk_secret_renderings() -> Vec<String> {
+    let secret = Fr::from(ZK_SECRET);
+    vec![
+        ZK_SECRET.to_string(),
+        format!("{secret:?}"),
+        format!("{secret}"),
+        hex::encode(fr_to_bytes(&secret)),
+    ]
+}
+
+fn assert_redacted(rendered: &str, secret_renderings: &[String]) {
+    assert!(rendered.contains("<redacted>"), "{rendered}");
+    for secret in secret_renderings {
+        assert!(
+            !rendered.contains(secret.as_str()),
+            "{rendered} leaks {secret}"
+        );
+    }
+}
+
+#[test]
+fn ed25519_types_redact_the_secret() {
+    let unsecured = UnsecuredEd25519Key::from_bytes(&ED25519_SECRET);
+    let renderings = [
+        format!("{unsecured:?}"),
+        format!("{:?}", Ed25519Key::from(unsecured.clone())),
+        format!("{:?}", Key::Ed25519(Ed25519Key::from(unsecured.clone()))),
+        format!("{:?}", StoredKey::Ed25519(unsecured)),
+    ];
+    for rendered in renderings {
+        assert_redacted(&rendered, &ed25519_secret_renderings());
+    }
+}
+
+#[test]
+fn zk_types_redact_the_secret() {
+    let unsecured = UnsecuredZkKey::new(Fr::from(ZK_SECRET));
+    let renderings = [
+        format!("{unsecured:?}"),
+        format!("{:?}", ZkKey::from(unsecured.clone())),
+        format!("{:?}", Key::Zk(ZkKey::from(unsecured.clone()))),
+        format!("{:?}", StoredKey::Zk(unsecured)),
+    ];
+    for rendered in renderings {
+        assert_redacted(&rendered, &zk_secret_renderings());
+    }
+}
diff --git a/kms/operators/Cargo.toml b/kms/operators/Cargo.toml
index 2dd521b48..7517c8557 100644
--- a/kms/operators/Cargo.toml
+++ b/kms/operators/Cargo.toml
@@ -17,14 +17,15 @@ async-trait                   = { workspace = true }
 lb-blend-proofs               = { workspace = true }
 lb-core                       = { workspace = true }
 lb-groth16                    = { workspace = true }
-lb-key-management-system-keys = { features = ["unsafe"], workspace = true }
+lb-key-management-system-keys = { workspace = true }
 lb-log-targets                = { workspace = true }
+lb-mmr                        = { workspace = true }
 lb-poseidon2                  = { workspace = true }
 lb-utils                      = { features = ["tokio"], workspace = true }
 lb-utxotree                   = { workspace = true }
+thiserror                     = { workspace = true }
 tokio                         = { features = ["rt"], workspace = true }
 tracing                       = { workspace = true }
 
 [features]
 tokio-task-names = ["lb-utils/tokio-task-names"]
-unsafe           = []
diff --git a/kms/operators/src/ed25519/derive_x25519.rs b/kms/operators/src/ed25519/derive_x25519.rs
index 32510cba0..6b5d1bf1f 100644
--- a/kms/operators/src/ed25519/derive_x25519.rs
+++ b/kms/operators/src/ed25519/derive_x25519.rs
@@ -19,6 +19,13 @@ impl Debug for DeriveX25519Operator {
     }
 }
 
+impl DeriveX25519Operator {
+    #[must_use]
+    pub const fn new(response_channel: oneshot::Sender<X25519PrivateKey>) -> Self {
+        Self { response_channel }
+    }
+}
+
 #[async_trait::async_trait]
 impl SecureKeyOperator for DeriveX25519Operator {
     type Key = Ed25519Key;
diff --git a/kms/operators/src/ed25519/exfiltrate_secret_key.rs b/kms/operators/src/ed25519/exfiltrate_secret_key.rs
deleted file mode 100644
index 0f0fb3ba5..000000000
--- a/kms/operators/src/ed25519/exfiltrate_secret_key.rs
+++ /dev/null
@@ -1,41 +0,0 @@
-use core::fmt::{self, Debug, Formatter};
-
-use lb_key_management_system_keys::keys::{
-    Ed25519Key, UnsecuredEd25519Key, errors::KeyError, secured_key::SecureKeyOperator,
-};
-use lb_log_targets::kms;
-use tokio::sync::oneshot;
-use tracing::debug;
-
-const LOG_TARGET: &str = kms::operators::ED25519;
-
-pub struct LeakSecretKeyOperator {
-    response_channel: oneshot::Sender<UnsecuredEd25519Key>,
-}
-
-impl Debug for LeakSecretKeyOperator {
-    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
-        f.debug_struct("LeakSecretKeyOperator").finish()
-    }
-}
-
-impl LeakSecretKeyOperator {
-    #[must_use]
-    pub const fn new(response_channel: oneshot::Sender<UnsecuredEd25519Key>) -> Self {
-        Self { response_channel }
-    }
-}
-
-#[async_trait::async_trait]
-impl SecureKeyOperator for LeakSecretKeyOperator {
-    type Key = Ed25519Key;
-    type Error = KeyError;
-
-    async fn execute(mut self: Box<Self>, key: &Self::Key) -> Result<(), Self::Error> {
-        let _ = self
-            .response_channel
-            .send(key.clone().into_unsecured())
-            .map_err(|_| debug!(target: LOG_TARGET, "Error sending Ed25519 key to requester."));
-        Ok(())
-    }
-}
diff --git a/kms/operators/src/ed25519/libp2p_identity.rs b/kms/operators/src/ed25519/libp2p_identity.rs
new file mode 100644
index 000000000..5a4d32b61
--- /dev/null
+++ b/kms/operators/src/ed25519/libp2p_identity.rs
@@ -0,0 +1,48 @@
+use core::fmt::{self, Debug, Formatter};
+
+use lb_key_management_system_keys::keys::{
+    Ed25519Key, LibP2pIdentity, errors::KeyError, secured_key::SecureKeyOperator,
+};
+use lb_log_targets::kms;
+use tokio::sync::oneshot;
+use tracing::debug;
+
+const LOG_TARGET: &str = kms::operators::ED25519;
+
+/// Hands the non-ephemeral signing key to the Blend swarm as its libp2p
+/// identity, once, at service start.
+///
+/// The NSK is the node's network-level identity
+/// (`key-types-and-generation.md` § Non-ephemeral Signing Key) and the Noise
+/// handshake needs the keypair in-process, so this cannot become an operation
+/// the KMS performs on the caller's behalf. What it can be is a single-use
+/// export of a type that nothing else accepts.
+pub struct LibP2pIdentityOperator {
+    response_channel: oneshot::Sender<LibP2pIdentity>,
+}
+
+impl Debug for LibP2pIdentityOperator {
+    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
+        f.write_str("LibP2pIdentityOperator")
+    }
+}
+
+impl LibP2pIdentityOperator {
+    #[must_use]
+    pub const fn new(response_channel: oneshot::Sender<LibP2pIdentity>) -> Self {
+        Self { response_channel }
+    }
+}
+
+#[async_trait::async_trait]
+impl SecureKeyOperator for LibP2pIdentityOperator {
+    type Key = Ed25519Key;
+    type Error = KeyError;
+
+    async fn execute(self: Box<Self>, key: &Self::Key) -> Result<(), Self::Error> {
+        if self.response_channel.send(key.libp2p_identity()).is_err() {
+            debug!(target: LOG_TARGET, "Error sending libp2p identity to requester.");
+        }
+        Ok(())
+    }
+}
diff --git a/kms/operators/src/ed25519/mod.rs b/kms/operators/src/ed25519/mod.rs
index 107631cac..2c3804c85 100644
--- a/kms/operators/src/ed25519/mod.rs
+++ b/kms/operators/src/ed25519/mod.rs
@@ -1,3 +1,2 @@
 pub mod derive_x25519;
-#[cfg(feature = "unsafe")]
-pub mod exfiltrate_secret_key;
+pub mod libp2p_identity;
diff --git a/kms/operators/src/zk/mod.rs b/kms/operators/src/zk/mod.rs
index 20bb8eb1a..1e33f2ad2 100644
--- a/kms/operators/src/zk/mod.rs
+++ b/kms/operators/src/zk/mod.rs
@@ -1,3 +1,2 @@
 pub mod leader;
-#[cfg(feature = "unsafe")]
 pub mod voucher;
diff --git a/kms/operators/src/zk/voucher.rs b/kms/operators/src/zk/voucher.rs
index e4740cfb9..b3cc91fd1 100644
--- a/kms/operators/src/zk/voucher.rs
+++ b/kms/operators/src/zk/voucher.rs
@@ -1,26 +1,53 @@
+use core::fmt::{self, Debug, Formatter};
+
+use lb_core::{
+    mantle::ops::leader_claim::{VoucherCm, VoucherNullifier, VoucherSecret},
+    proofs::{
+        MerklePathWitnessError,
+        leader_claim_proof::{
+            self, Groth16LeaderClaimProof, LeaderClaimPrivate, LeaderClaimPublic,
+        },
+    },
+};
 use lb_groth16::Fr;
 use lb_key_management_system_keys::keys::{
     ZkKey, errors::KeyError, secured_key::SecureKeyOperator,
 };
 use lb_log_targets::kms;
+use lb_mmr::MerklePath;
 use lb_poseidon2::{Digest as _, Poseidon2Bn254Hasher as ZkHasher};
+use lb_utils::tokio::task::spawn_blocking;
 use tokio::sync::oneshot;
 use tracing::debug;
 
 const LOG_TARGET: &str = kms::operators::ZK_VOUCHER;
 
-/// Derives/returns a voucher secret from key and index.
-// TODO: Make this secure by embedding actual logic
-//       once resolving cyclic dep: kms-keys <> core
-#[derive(Debug)]
-pub struct UnsafeVoucherOperator {
+/// The voucher secret for `index` under the voucher master key. Never leaves
+/// the KMS: both operators below derive what the wallet needs from it.
+fn voucher_secret(master_key: &ZkKey, index: Fr) -> VoucherSecret {
+    ZkHasher::digest(&[*master_key.as_fr(), index]).into()
+}
+
+/// Derives the commitment and nullifier of the voucher at `index`.
+pub struct VoucherOperator {
     index: Fr,
-    result_channel: oneshot::Sender<Fr>,
+    result_channel: oneshot::Sender<(VoucherCm, VoucherNullifier)>,
+}
+
+impl Debug for VoucherOperator {
+    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
+        f.debug_struct("VoucherOperator")
+            .field("index", &self.index)
+            .finish_non_exhaustive()
+    }
 }
 
-impl UnsafeVoucherOperator {
+impl VoucherOperator {
     #[must_use]
-    pub const fn new(index: Fr, result_channel: oneshot::Sender<Fr>) -> Self {
+    pub const fn new(
+        index: Fr,
+        result_channel: oneshot::Sender<(VoucherCm, VoucherNullifier)>,
+    ) -> Self {
         Self {
             index,
             result_channel,
@@ -29,15 +56,120 @@ impl UnsafeVoucherOperator {
 }
 
 #[async_trait::async_trait]
-impl SecureKeyOperator for UnsafeVoucherOperator {
+impl SecureKeyOperator for VoucherOperator {
     type Key = ZkKey;
     type Error = KeyError;
 
     async fn execute(self: Box<Self>, key: &Self::Key) -> Result<(), Self::Error> {
-        let voucher_secret = ZkHasher::digest(&[*key.as_fr(), self.index]);
-        if self.result_channel.send(voucher_secret).is_err() {
+        let secret = voucher_secret(key, self.index);
+        let voucher = (
+            VoucherCm::from_secret(secret),
+            VoucherNullifier::from_secret(secret),
+        );
+        if self.result_channel.send(voucher).is_err() {
             debug!(target: LOG_TARGET, "Failed to send voucher via channel");
         }
         Ok(())
     }
 }
+
+#[derive(Debug, thiserror::Error)]
+pub enum ProveClaimError {
+    #[error(transparent)]
+    MerklePath(#[from] MerklePathWitnessError),
+    #[error(transparent)]
+    Proof(#[from] leader_claim_proof::Error),
+}
+
+/// Proves the leader claim for the voucher at `index`, inside the KMS.
+///
+/// The proof runs on the blocking pool with a clone of the key and `execute`
+/// returns as soon as it is scheduled, so the KMS loop is not held for the
+/// proving time; the result arrives on the operator's own channel.
+pub struct ProveClaimOperator {
+    index: Fr,
+    voucher_path: MerklePath,
+    voucher_root: Fr,
+    mantle_tx_hash: Fr,
+    result_channel: oneshot::Sender<Result<Groth16LeaderClaimProof, ProveClaimError>>,
+}
+
+impl Debug for ProveClaimOperator {
+    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
+        f.debug_struct("ProveClaimOperator")
+            .field("index", &self.index)
+            .field("voucher_root", &self.voucher_root)
+            .field("mantle_tx_hash", &self.mantle_tx_hash)
+            .finish_non_exhaustive()
+    }
+}
+
+impl ProveClaimOperator {
+    #[must_use]
+    pub const fn new(
+        index: Fr,
+        voucher_path: MerklePath,
+        voucher_root: Fr,
+        mantle_tx_hash: Fr,
+        result_channel: oneshot::Sender<Result<Groth16LeaderClaimProof, ProveClaimError>>,
+    ) -> Self {
+        Self {
+            index,
+            voucher_path,
+            voucher_root,
+            mantle_tx_hash,
+            result_channel,
+        }
+    }
+}
+
+fn prove_claim(
+    master_key: &ZkKey,
+    index: Fr,
+    voucher_path: &MerklePath,
+    voucher_root: Fr,
+    mantle_tx_hash: Fr,
+) -> Result<Groth16LeaderClaimProof, ProveClaimError> {
+    let secret = voucher_secret(master_key, index);
+    let public = LeaderClaimPublic {
+        voucher_nullifier: VoucherNullifier::from_secret(secret).into(),
+        voucher_root,
+        mantle_tx_hash,
+    };
+    let witness = LeaderClaimPrivate::try_new(public, voucher_path, secret)?;
+    Ok(Groth16LeaderClaimProof::prove(witness)?)
+}
+
+#[async_trait::async_trait]
+impl SecureKeyOperator for ProveClaimOperator {
+    type Key = ZkKey;
+    type Error = KeyError;
+
+    async fn execute(self: Box<Self>, key: &Self::Key) -> Result<(), Self::Error> {
+        let Self {
+            index,
+            voucher_path,
+            voucher_root,
+            mantle_tx_hash,
+            result_channel,
+        } = *self;
+        let master_key = key.clone();
+        drop(spawn_blocking(
+            "logos/kms/leader-claim-proof-blocking",
+            move || {
+                let result = prove_claim(
+                    &master_key,
+                    index,
+                    &voucher_path,
+                    voucher_root,
+                    mantle_tx_hash,
+                );
+                drop(master_key);
+                if result_channel.send(result).is_err() {
+                    debug!(target: LOG_TARGET, "Failed to send proof of claim via channel");
+                }
+            },
+        ));
+        Ok(())
+    }
+}
diff --git a/nodes/node/binary/Cargo.toml b/nodes/node/binary/Cargo.toml
index ad1d3eb75..d40af1a71 100644
--- a/nodes/node/binary/Cargo.toml
+++ b/nodes/node/binary/Cargo.toml
@@ -31,7 +31,7 @@ lb-core                          = { features = ["openapi"], workspace = true }
 lb-cryptarchia-engine            = { features = ["openapi"], workspace = true }
 lb-groth16                       = { workspace = true }
 lb-http-api-common               = { workspace = true }
-lb-key-management-system-service = { features = ["unsafe"], workspace = true }
+lb-key-management-system-service = { workspace = true }
 lb-ledger                        = { workspace = true }
 lb-libp2p                        = { workspace = true }
 lb-log-targets                   = { workspace = true }
@@ -75,6 +75,7 @@ time       = { workspace = true }
 tower      = { features = ["limit"], workspace = true }
 tower-http = { features = ["cors", "limit", "timeout", "trace"], workspace = true }
 validator  = { features = ["derive"], workspace = true }
+zeroize    = { workspace = true }
 
 # Jemalloc
 [target.'cfg(not(target_env = "msvc"))'.dependencies]
@@ -93,7 +94,7 @@ jemalloc  = ["dep:tikv-jemallocator"]
 # Existing broad profiling feature. Keep this as an umbrella feature for
 # backwards compatibility: HTTP pprof + tokio-console support.
 profiling                        = ["lb-http-api-common/profiling", "tokio-console"]
-testing                          = ["lb-key-management-system-service/unsafe"]
+testing                          = []
 testing-disable-proposal-publish = ["lb-chain-leader-service/testing-disable-proposal-publish"]
 
 # Tokio task/resource profiling via tokio-console.
diff --git a/nodes/node/binary/src/cli/config/init.rs b/nodes/node/binary/src/cli/config/init.rs
index e2a6bfd69..0ebfcca2d 100644
--- a/nodes/node/binary/src/cli/config/init.rs
+++ b/nodes/node/binary/src/cli/config/init.rs
@@ -123,16 +123,12 @@ pub fn build_user_config(keystore: &Keystore, args: InitArgs) -> UserConfig {
 }
 
 fn build_network_config(keystore: &Keystore, network_args: NetworkArgs) -> NetworkConfig {
-    let unsecured_key = keystore
-        .get_ed25519(KeyTitle::NETWORK_SWARM)
-        .map(|(_, key)| key)
+    let node_key = keystore
+        .swarm_secret_key()
         .expect("Network key set by default");
-    let mut network_secret_key_bytes: [u8; 32] = *unsecured_key.as_bytes();
 
     let mut network_config = NetworkConfig::default();
-    network_config.backend.swarm.node_key =
-        lb_libp2p::ed25519::SecretKey::try_from_bytes(&mut network_secret_key_bytes)
-            .expect("Valid default secret key structure");
+    network_config.backend.swarm.node_key = node_key;
     update_network(&mut network_config, network_args)
         .expect("Network configuration should update from cli args");
 
@@ -160,12 +156,12 @@ fn build_cryptarchia_config(
     initial_peers: Option<Vec<Multiaddr>>,
     cryptarchia_args: CryptarchiaArgs,
 ) -> CryptarchiaConfig {
-    let (_, cryptarchia_funding_key) = keystore
-        .get_zk(KeyTitle::LEADER_FUNDING)
+    let (_, cryptarchia_funding_pk) = keystore
+        .public_zk(KeyTitle::LEADER_FUNDING)
         .expect("Cryptarchia funding key set by default");
     let mut cryptarchia_config =
         CryptarchiaConfig::with_required_values(CryptarchiaConfigRequiredValues {
-            funding_pk: cryptarchia_funding_key.to_public_key(),
+            funding_pk: cryptarchia_funding_pk,
         });
     if !cryptarchia_args.skip_ibd
         && let Some(initial_peers) = initial_peers
@@ -184,11 +180,11 @@ fn build_cryptarchia_config(
 }
 
 fn build_sdp_config(keystore: &Keystore, sdp_args: SdpArgs) -> SdpConfig {
-    let (_, sdp_funding_key) = keystore
-        .get_zk(KeyTitle::SDP_FUNDING)
+    let (_, sdp_funding_pk) = keystore
+        .public_zk(KeyTitle::SDP_FUNDING)
         .expect("Sdp funding key set by default");
     let mut sdp_config = SdpConfig::with_required_values(SdpConfigRequiredValues {
-        funding_pk: sdp_funding_key.to_public_key(),
+        funding_pk: sdp_funding_pk,
     });
     update_sdp(&mut sdp_config, sdp_args);
 
@@ -213,10 +209,7 @@ fn build_wallet_config(keystore: &Keystore) -> WalletConfig {
     let mut wallet_config = WalletConfig::with_required_values(WalletConfigRequiredValues {
         voucher_master_key_id,
     });
-    wallet_config.known_keys = keystore
-        .get_all_zk()
-        .map(|(id, key)| (id, key.to_public_key()))
-        .collect();
+    wallet_config.known_keys = keystore.zk_public_keys().collect();
 
     wallet_config
 }
diff --git a/nodes/node/binary/src/cli/config/keystore.rs b/nodes/node/binary/src/cli/config/keystore.rs
index 3ee9fb281..f17323461 100644
--- a/nodes/node/binary/src/cli/config/keystore.rs
+++ b/nodes/node/binary/src/cli/config/keystore.rs
@@ -3,16 +3,14 @@ use std::collections::HashMap;
 use lb_groth16::fr_to_bytes;
 use lb_key_management_system_service::{
     backend::preload::KeyId,
-    keys::{
-        Ed25519Key, Key, UnsecuredEd25519Key, UnsecuredZkKey, ZkKey, secured_key::SecuredKey as _,
-    },
+    keys::{PublicKeyEncoding, StoredKey, UnsecuredEd25519Key, UnsecuredZkKey, ZkPublicKey},
 };
-use num_bigint::BigUint;
 use rand::rngs::OsRng;
 use serde::{Deserialize, Serialize};
 use thiserror::Error;
+use zeroize::Zeroizing;
 
-const WARNING: &str = "Do not share your secret keys";
+pub const WARNING: &str = "Do not share your secret keys";
 
 #[derive(Serialize, Deserialize, Hash, Eq, PartialEq, Clone, Debug)]
 #[serde(transparent)]
@@ -57,96 +55,103 @@ pub enum KeystoreError {
     ZkExpected(KeyTitle),
 }
 
+/// The keystore file: every key the node's config refers to, in the on-disk
+/// [`StoredKey`] encoding.
+///
+/// Secrets leave this type in exactly two ways: as [`StoredKey`]s copied into
+/// `kms.backend.keys` of the user config (which is the same file format), and
+/// as the libp2p swarm key through
+/// [`swarm_secret_key`](Self::swarm_secret_key), which the network config needs
+/// in the clear. Everything else the CLI does with a key needs only its public
+/// half.
 #[derive(Serialize, Deserialize)]
 pub struct Keystore {
     // Convenience mapping for users to inspect when serialized.
     public_keys: HashMap<KeyTitle, KeyId>,
-    secret_keys: HashMap<KeyTitle, Key>,
+    secret_keys: HashMap<KeyTitle, StoredKey>,
 
     #[serde(rename = "WARNING")]
     warning: String,
 }
 
 impl Keystore {
-    pub fn set(&mut self, name: impl Into<KeyTitle>, key: Key) {
+    pub fn set(&mut self, name: impl Into<KeyTitle>, key: StoredKey) {
         let key_name = name.into();
         self.public_keys.insert(key_name.clone(), key_id(&key));
         self.secret_keys.insert(key_name, key);
     }
 
     #[must_use]
-    pub fn get(&self, name: impl Into<KeyTitle>) -> Option<(KeyId, &Key)> {
+    pub fn get(&self, name: impl Into<KeyTitle>) -> Option<(KeyId, &StoredKey)> {
         self.secret_keys
             .get_key_value(&name.into())
             .map(|(_, v)| (key_id(v), v))
     }
 
-    pub fn get_all(&self) -> impl Iterator<Item = (KeyId, &Key)> {
+    pub fn get_all(&self) -> impl Iterator<Item = (KeyId, &StoredKey)> {
         self.secret_keys.values().map(|key| (key_id(key), key))
     }
 
-    pub fn get_ed25519(
+    /// The public half of the ZK key stored under `title`.
+    pub fn public_zk(
         &self,
         title: impl Into<KeyTitle>,
-    ) -> Result<(KeyId, UnsecuredEd25519Key), KeystoreError> {
+    ) -> Result<(KeyId, ZkPublicKey), KeystoreError> {
         let title = title.into();
-        let (key_id, generic_key) = self
+        let (id, key) = self
             .get(title.clone())
             .ok_or_else(|| KeystoreError::NotFound(title.clone()))?;
 
-        match generic_key {
-            Key::Ed25519(inner_key) => Ok((key_id, inner_key.clone().into_unsecured())),
-            Key::Zk(_) => Err(KeystoreError::Ed25519Expected(title)),
+        match key {
+            StoredKey::Zk(inner_key) => Ok((id, inner_key.to_public_key())),
+            StoredKey::Ed25519(_) => Err(KeystoreError::ZkExpected(title)),
         }
     }
 
-    pub fn get_zk(
-        &self,
-        title: impl Into<KeyTitle>,
-    ) -> Result<(KeyId, UnsecuredZkKey), KeystoreError> {
-        let title = title.into();
-        let (id, generic_key) = self
-            .get(title.clone())
-            .ok_or_else(|| KeystoreError::NotFound(title.clone()))?;
-
-        match generic_key {
-            Key::Zk(inner_key) => Ok((id, inner_key.clone().into_unsecured())),
-            Key::Ed25519(_) => Err(KeystoreError::ZkExpected(title)),
-        }
+    /// The public halves of every ZK key in the store.
+    pub fn zk_public_keys(&self) -> impl Iterator<Item = (KeyId, ZkPublicKey)> + '_ {
+        self.secret_keys.values().filter_map(|key| match key {
+            StoredKey::Zk(inner_key) => Some((key_id(key), inner_key.to_public_key())),
+            StoredKey::Ed25519(_) => None,
+        })
     }
 
-    pub fn get_all_zk(&self) -> impl Iterator<Item = (KeyId, UnsecuredZkKey)> + '_ {
-        self.secret_keys
-            .values()
-            .filter_map(|generic_key| match generic_key {
-                Key::Zk(inner_key) => {
-                    let id = key_id(generic_key);
-                    let unsecured = inner_key.clone().into_unsecured();
-                    Some((id, unsecured))
-                }
-                Key::Ed25519(_) => None,
-            })
+    /// The `NetworkSwarm` key in the form the libp2p network config stores it.
+    ///
+    /// This is the one place a secret leaves the keystore other than as a
+    /// [`StoredKey`].
+    pub fn swarm_secret_key(&self) -> Result<lb_libp2p::ed25519::SecretKey, KeystoreError> {
+        let title = KeyTitle::from(KeyTitle::NETWORK_SWARM);
+        let (_, key) = self
+            .get(title.clone())
+            .ok_or_else(|| KeystoreError::NotFound(title.clone()))?;
+        let StoredKey::Ed25519(inner_key) = key else {
+            return Err(KeystoreError::Ed25519Expected(title));
+        };
+        let mut bytes = inner_key.to_bytes();
+        Ok(lb_libp2p::ed25519::SecretKey::try_from_bytes(&mut *bytes)
+            .expect("an Ed25519 secret key is a valid libp2p secret key"))
     }
 
-    pub fn generate_ed25519(&mut self, title: impl Into<KeyTitle>) -> (KeyId, UnsecuredEd25519Key) {
+    pub fn generate_ed25519(&mut self, title: impl Into<KeyTitle>) -> KeyId {
         let title = title.into();
-        let secure_key = Ed25519Key::generate(&mut OsRng);
-        let unsecured = secure_key.clone().into_unsecured();
-
-        self.set(title.clone(), Key::Ed25519(secure_key));
-        (key_id(&self.secret_keys[&title]), unsecured)
+        self.set(
+            title.clone(),
+            StoredKey::Ed25519(UnsecuredEd25519Key::generate(&mut OsRng)),
+        );
+        key_id(&self.secret_keys[&title])
     }
 
-    pub fn generate_zk(&mut self, title: impl Into<KeyTitle>) -> (KeyId, UnsecuredZkKey) {
+    pub fn generate_zk(&mut self, title: impl Into<KeyTitle>) -> KeyId {
         let title = title.into();
-        let secure_key = generate_zk_key_from_random_bytes();
-        let unsecured = secure_key.clone().into_unsecured();
-
-        self.set(title.clone(), Key::Zk(secure_key));
-        (key_id(&self.secret_keys[&title]), unsecured)
+        self.set(
+            title.clone(),
+            StoredKey::Zk(UnsecuredZkKey::from_rng(&mut OsRng)),
+        );
+        key_id(&self.secret_keys[&title])
     }
 
-    pub fn remove(&mut self, title: impl Into<KeyTitle>) -> Option<(KeyId, Key)> {
+    pub fn remove(&mut self, title: impl Into<KeyTitle>) -> Option<(KeyId, StoredKey)> {
         let title = title.into();
         self.public_keys.remove(&title);
         self.secret_keys.remove(&title).map(|v| (key_id(&v), v))
@@ -173,16 +178,97 @@ impl Default for Keystore {
     }
 }
 
-fn key_id(key: &Key) -> KeyId {
-    let key_id_bytes = match key {
-        Key::Ed25519(ed25519_secret_key) => ed25519_secret_key.as_public_key().to_bytes(),
-        Key::Zk(zk_secret_key) => fr_to_bytes(zk_secret_key.as_public_key().as_fr()),
+fn key_id(key: &StoredKey) -> KeyId {
+    let key_id_bytes = match key.public_key() {
+        PublicKeyEncoding::Ed25519(public_key) => public_key.to_bytes(),
+        PublicKeyEncoding::Zk(public_key) => fr_to_bytes(public_key.as_fr()),
     };
     hex::encode(key_id_bytes)
 }
 
-fn generate_zk_key_from_random_bytes() -> ZkKey {
-    let mut bytes = [0u8; 32];
-    rand::RngCore::fill_bytes(&mut OsRng, &mut bytes);
-    ZkKey::from(BigUint::from_bytes_le(&bytes))
+/// The secret of `key` as the hex string `add-key --ed25519` / `--zk` take
+/// back: the 32 secret bytes for Ed25519, the little-endian scalar for ZK.
+#[must_use]
+pub fn export_secret_hex(key: &StoredKey) -> Zeroizing<String> {
+    match key {
+        StoredKey::Ed25519(inner_key) => Zeroizing::new(hex::encode(inner_key.to_bytes())),
+        StoredKey::Zk(inner_key) => Zeroizing::new(hex::encode(fr_to_bytes(inner_key.as_fr()))),
+    }
+}
+
+#[cfg(test)]
+mod tests {
+    use lb_key_management_system_service::keys::secured_key::SecuredKey as _;
+
+    use super::*;
+    use crate::config::{parse_hex_ed25519_key, parse_hex_zk_key};
+
+    #[test]
+    fn exported_secret_hex_round_trips_through_the_add_key_parsers() {
+        let keystore = Keystore::default();
+
+        let (_, zk) = keystore
+            .get(KeyTitle::STAKE)
+            .expect("stake key is generated");
+        let parsed = parse_hex_zk_key(&export_secret_hex(zk)).expect("zk export parses");
+        assert_eq!(StoredKey::Zk(parsed).public_key(), zk.public_key());
+
+        let (_, ed25519) = keystore
+            .get(KeyTitle::NETWORK_SWARM)
+            .expect("swarm key is generated");
+        let parsed = parse_hex_ed25519_key(&export_secret_hex(ed25519)).expect("ed25519 parses");
+        let parsed = UnsecuredEd25519Key::from_bytes(
+            parsed
+                .as_ref()
+                .try_into()
+                .expect("an ed25519 secret key is 32 bytes"),
+        );
+        assert_eq!(
+            StoredKey::Ed25519(parsed).public_key(),
+            ed25519.public_key()
+        );
+    }
+
+    #[test]
+    fn swarm_secret_key_is_the_network_swarm_entry() {
+        let keystore = Keystore::default();
+        let (_, stored) = keystore.get(KeyTitle::NETWORK_SWARM).expect("generated");
+        let PublicKeyEncoding::Ed25519(expected) = stored.public_key() else {
+            panic!("swarm key is Ed25519");
+        };
+
+        let swarm_key = keystore.swarm_secret_key().expect("swarm key exports");
+        let exported = UnsecuredEd25519Key::from_bytes(
+            swarm_key
+                .as_ref()
+                .try_into()
+                .expect("an ed25519 secret key is 32 bytes"),
+        );
+        assert_eq!(exported.public_key(), expected);
+    }
+
+    #[test]
+    fn public_getters_match_the_stored_keys() {
+        let keystore = Keystore::default();
+        let (id, public_key) = keystore
+            .public_zk(KeyTitle::LEADER_FUNDING)
+            .expect("zk key");
+        let (stored_id, stored) = keystore.get(KeyTitle::LEADER_FUNDING).expect("stored");
+        assert_eq!(id, stored_id);
+        assert_eq!(PublicKeyEncoding::Zk(public_key), stored.public_key());
+        assert!(matches!(
+            keystore.public_zk(KeyTitle::NETWORK_SWARM),
+            Err(KeystoreError::ZkExpected(_))
+        ));
+        assert_eq!(
+            keystore.zk_public_keys().count(),
+            KeyTitle::PREDEFINED_ZK.len()
+        );
+        assert_eq!(
+            keystore.get_all().count(),
+            KeyTitle::PREDEFINED_ZK.len() + KeyTitle::PREDEFINED_ED25519.len()
+        );
+        let _: PublicKeyEncoding =
+            lb_key_management_system_service::keys::Key::from(stored.clone()).as_public_key();
+    }
 }
diff --git a/nodes/node/binary/src/cli/config/migrate_0_1_2/config.rs b/nodes/node/binary/src/cli/config/migrate_0_1_2/config.rs
index 80c52a3e7..a58aa28bc 100644
--- a/nodes/node/binary/src/cli/config/migrate_0_1_2/config.rs
+++ b/nodes/node/binary/src/cli/config/migrate_0_1_2/config.rs
@@ -1,4 +1,7 @@
-use lb_key_management_system_service::{backend::preload::KeyId, keys::Ed25519Key};
+use lb_key_management_system_service::{
+    backend::preload::KeyId,
+    keys::{StoredKey, UnsecuredEd25519Key},
+};
 use lb_libp2p::ed25519::SecretKey;
 use serde::{Deserialize, Deserializer};
 
@@ -84,13 +87,16 @@ impl OldConfig {
         let mut old_kms = self.kms.backend.keys;
 
         let network_secret_key = self.network.backend.swarm.node_key;
-        let network_secret_key = Ed25519Key::from_bytes(
+        let network_secret_key = UnsecuredEd25519Key::from_bytes(
             network_secret_key
                 .as_ref()
                 .try_into()
                 .expect("SecretKey is guaranteed to be 32 bytes"),
         );
-        keystore.set(KeyTitle::NETWORK_SWARM, network_secret_key.into());
+        keystore.set(
+            KeyTitle::NETWORK_SWARM,
+            StoredKey::Ed25519(network_secret_key),
+        );
 
         if let Some(blend_signing) = old_kms.remove(&self.blend.non_ephemeral_signing_key_id) {
             keystore.set(KeyTitle::BLEND_SIGNING, blend_signing);
diff --git a/nodes/node/binary/src/cli/config/update.rs b/nodes/node/binary/src/cli/config/update.rs
index 2c863a98d..948c4c76e 100644
--- a/nodes/node/binary/src/cli/config/update.rs
+++ b/nodes/node/binary/src/cli/config/update.rs
@@ -102,15 +102,11 @@ fn update_network_config(
     network_config: &mut NetworkConfig,
     network_args: NetworkArgs,
 ) {
-    let unsecured_key = keystore
-        .get_ed25519(KeyTitle::NETWORK_SWARM)
-        .map(|(_, key)| key)
+    let node_key = keystore
+        .swarm_secret_key()
         .expect("Network key set by default");
-    let mut network_secret_key_bytes: [u8; 32] = *unsecured_key.as_bytes();
 
-    network_config.backend.swarm.node_key =
-        lb_libp2p::ed25519::SecretKey::try_from_bytes(&mut network_secret_key_bytes)
-            .expect("Valid default secret key structure");
+    network_config.backend.swarm.node_key = node_key;
     update_network(network_config, network_args)
         .expect("Network configuration should update from cli args");
 }
@@ -135,10 +131,10 @@ fn update_cryptarchia_config(
     cryptarchia_config: &mut CryptarchiaConfig,
     cryptarchia_args: CryptarchiaArgs,
 ) {
-    let (_, cryptarchia_funding_key) = keystore
-        .get_zk(KeyTitle::LEADER_FUNDING)
+    let (_, cryptarchia_funding_pk) = keystore
+        .public_zk(KeyTitle::LEADER_FUNDING)
         .expect("Cryptarchia funding key set by default");
-    cryptarchia_config.set_funding_pk(cryptarchia_funding_key.to_public_key());
+    cryptarchia_config.set_funding_pk(cryptarchia_funding_pk);
 
     if !cryptarchia_args.skip_ibd
         && let Some(initial_peers) = initial_peers
@@ -156,10 +152,10 @@ fn update_cryptarchia_config(
 }
 
 fn update_sdp_config(keystore: &Keystore, sdp_config: &mut SdpConfig, sdp_args: SdpArgs) {
-    let (_, sdp_funding_key) = keystore
-        .get_zk(KeyTitle::SDP_FUNDING)
+    let (_, sdp_funding_pk) = keystore
+        .public_zk(KeyTitle::SDP_FUNDING)
         .expect("Sdp funding key set by default");
-    sdp_config.set_funding_pk(sdp_funding_key.to_public_key());
+    sdp_config.set_funding_pk(sdp_funding_pk);
 
     update_sdp(sdp_config, sdp_args);
 }
@@ -177,8 +173,5 @@ fn update_wallet_config(keystore: &Keystore, wallet_config: &mut WalletConfig) {
         .expect("Vaucher master key set by default");
 
     wallet_config.voucher_master_key_id = voucher_master_key_id;
-    wallet_config.known_keys = keystore
-        .get_all_zk()
-        .map(|(id, key)| (id, key.to_public_key()))
-        .collect();
+    wallet_config.known_keys = keystore.zk_public_keys().collect();
 }
diff --git a/nodes/node/binary/src/cli/keys.rs b/nodes/node/binary/src/cli/keys.rs
index 1389770c2..3ca440524 100644
--- a/nodes/node/binary/src/cli/keys.rs
+++ b/nodes/node/binary/src/cli/keys.rs
@@ -4,7 +4,7 @@ use clap::{Parser, ValueEnum};
 use color_eyre::eyre::Result;
 use lb_key_management_system_service::{
     backend::preload::KeyId,
-    keys::{Ed25519Key, Key, UnsecuredEd25519Key, UnsecuredZkKey, ZkKey},
+    keys::{StoredKey, UnsecuredEd25519Key, UnsecuredZkKey},
 };
 use thiserror::Error;
 
@@ -14,7 +14,7 @@ use crate::{
         UpdateArgs,
         config::{
             confirm_overwrite,
-            keystore::{KeyTitle, Keystore},
+            keystore::{KeyTitle, Keystore, WARNING, export_secret_hex},
             update::update_user_config,
         },
     },
@@ -120,12 +120,12 @@ impl AddKeyArgs {
         user_config: PathBuf,
         keystore: PathBuf,
         key_title: Option<String>,
-        key: &Key,
+        key: &StoredKey,
         auto_approve: bool,
     ) -> Self {
         let (ed25519_key, zk_key) = match key {
-            Key::Ed25519(key) => (Some(key.clone().into_unsecured()), None),
-            Key::Zk(key) => (None, Some(key.clone().into_unsecured())),
+            StoredKey::Ed25519(key) => (Some(key.clone()), None),
+            StoredKey::Zk(key) => (None, Some(key.clone())),
         };
 
         Self {
@@ -138,12 +138,12 @@ impl AddKeyArgs {
         }
     }
 
-    /// Returns the key provided via the `--ed25519`/`--zk` flags as a [`Key`],
-    /// validating that exactly one was given.
-    pub fn key(&self) -> Result<Key> {
+    /// Returns the key provided via the `--ed25519`/`--zk` flags as a
+    /// [`StoredKey`], validating that exactly one was given.
+    pub fn key(&self) -> Result<StoredKey> {
         match (&self.ed25519_key, &self.zk_key) {
-            (Some(ed_secret), None) => Ok(Ed25519Key::from(ed_secret.clone()).into()),
-            (None, Some(zk_secret)) => Ok(ZkKey::from(zk_secret.clone()).into()),
+            (Some(ed_secret), None) => Ok(StoredKey::Ed25519(ed_secret.clone())),
+            (None, Some(zk_secret)) => Ok(StoredKey::Zk(zk_secret.clone())),
             (Some(_), Some(_)) => Err(color_eyre::eyre::eyre!(
                 "Please provide either --ed25519 or --zk, not both."
             )),
@@ -233,16 +233,10 @@ fn generate_key_into_keystore(
     keystore: &mut Keystore,
     key_title: String,
     key_type: &KeyType,
-) -> (KeyId, Key) {
+) -> KeyId {
     match key_type {
-        KeyType::Ed25519 => {
-            let (id, secret_key) = keystore.generate_ed25519(key_title);
-            (id, Ed25519Key::from(secret_key).into())
-        }
-        KeyType::Zk => {
-            let (id, secret_key) = keystore.generate_zk(key_title);
-            (id, ZkKey::from(secret_key).into())
-        }
+        KeyType::Ed25519 => keystore.generate_ed25519(key_title),
+        KeyType::Zk => keystore.generate_zk(key_title),
     }
 }
 
@@ -264,7 +258,7 @@ pub fn generate_key(args: GenerateKeyArgs) -> Result<KeyId> {
         .as_ref()
         .map_or_else(|| next_user_key_title(&keystore), Clone::clone);
 
-    let (key_id, _) = generate_key_into_keystore(&mut keystore, user_key_title, &key_type);
+    let key_id = generate_key_into_keystore(&mut keystore, user_key_title, &key_type);
 
     persist_user_config_and_keystore(
         &mut user_config,
@@ -283,7 +277,8 @@ pub fn run_generate_key(args: GenerateKeyArgs) -> Result<()> {
         return Ok(());
     }
 
-    // Declined: generate and show the key without persisting it.
+    // Declined: generate and show the key without persisting it. The secret
+    // is printed in the form `add-key` takes back, so it is not lost.
     let (_, mut keystore) = load_user_config_and_keystore(&args.user_config, &args.keystore)?;
 
     let user_key_title = args
@@ -291,10 +286,18 @@ pub fn run_generate_key(args: GenerateKeyArgs) -> Result<()> {
         .as_ref()
         .map_or_else(|| next_user_key_title(&keystore), Clone::clone);
 
-    let (key_id, key) = generate_key_into_keystore(&mut keystore, user_key_title, &args.key_type);
+    let key_id = generate_key_into_keystore(&mut keystore, user_key_title.clone(), &args.key_type);
+    let (_, key) = keystore
+        .get(user_key_title)
+        .expect("the key was generated into the keystore above");
+    let flag = match args.key_type {
+        KeyType::Ed25519 => "--ed25519",
+        KeyType::Zk => "--zk",
+    };
 
     println!("KeyID: {key_id}");
-    println!("Key: {key:?}");
+    println!("WARNING: {WARNING}. Nothing was written; keep it with `add-key {flag} <key>`.");
+    println!("Key ({flag}): {}", export_secret_hex(key).as_str());
 
     Ok(())
 }
diff --git a/nodes/node/binary/src/cli/participate.rs b/nodes/node/binary/src/cli/participate.rs
index 57657c0bd..89a21c744 100644
--- a/nodes/node/binary/src/cli/participate.rs
+++ b/nodes/node/binary/src/cli/participate.rs
@@ -40,15 +40,11 @@ pub fn run(args: &ParticipateArgs) -> Result<()> {
     let keystore_yaml = std::fs::read_to_string(&args.keystore)?;
     let keystore: Keystore = serde_yaml::from_str(&keystore_yaml)?;
 
-    let (_, stake_key) = keystore.get_zk(KeyTitle::STAKE)?;
-    let (_, leader_funding_key) = keystore.get_zk(KeyTitle::LEADER_FUNDING)?;
-    let (_, sdp_funding_key) = keystore.get_zk(KeyTitle::SDP_FUNDING)?;
+    let (_, stake_pk) = keystore.public_zk(KeyTitle::STAKE)?;
+    let (_, leader_funding_pk) = keystore.public_zk(KeyTitle::LEADER_FUNDING)?;
+    let (_, sdp_funding_pk) = keystore.public_zk(KeyTitle::SDP_FUNDING)?;
 
-    let mut stakeholder_identities = vec![
-        stake_key.to_public_key(),
-        leader_funding_key.to_public_key(),
-        sdp_funding_key.to_public_key(),
-    ];
+    let mut stakeholder_identities = vec![stake_pk, leader_funding_pk, sdp_funding_pk];
 
     let blend = build_blend_data(&user_config, &keystore, args.external_address)?;
     if let Some(blend) = &blend {
@@ -85,7 +81,7 @@ fn build_blend_data(
     let nat_config = &user_config.network.backend.swarm.nat;
     let locator_addr = resolve_locator_addr(listen_addr, nat_config, external_address)?;
     let locators = Locators::from(Locator::try_from(locator_addr).map_err(|e| eyre!("{e}"))?);
-    let (_, blend_key) = keystore.get_zk(KeyTitle::BLEND_ZK)?;
+    let (_, blend_zk_id) = keystore.public_zk(KeyTitle::BLEND_ZK)?;
 
     // Declaration ID is not required when providing participation information for
     // genesis ceremony, but it is still useful to have when configuring the
@@ -95,14 +91,14 @@ fn build_blend_data(
         service_type: ServiceType::BlendNetwork,
         locators: locators.clone(),
         provider_id: ProviderId(provider_id),
-        zk_id: blend_key.to_public_key(),
+        zk_id: blend_zk_id,
         service_note_id: NoteId::from(Fr::ZERO),
     }
     .id();
 
     Ok(Some(BlendParticipationData {
         provider_id,
-        zk_id: blend_key.to_public_key(),
+        zk_id: blend_zk_id,
         locators,
         service_type: ServiceType::BlendNetwork,
         declaration_id,
diff --git a/nodes/node/binary/src/config/kms/mod.rs b/nodes/node/binary/src/config/kms/mod.rs
index 51090bc8b..c12bd85f2 100644
--- a/nodes/node/binary/src/config/kms/mod.rs
+++ b/nodes/node/binary/src/config/kms/mod.rs
@@ -11,7 +11,13 @@ pub struct ServiceConfig {
 impl From<ServiceConfig> for PreloadKMSBackendSettings {
     fn from(value: ServiceConfig) -> Self {
         Self {
-            keys: value.user.backend.keys,
+            keys: value
+                .user
+                .backend
+                .keys
+                .into_iter()
+                .map(|(key_id, key)| (key_id, key.into()))
+                .collect(),
         }
     }
 }
diff --git a/nodes/node/binary/src/config/kms/serde.rs b/nodes/node/binary/src/config/kms/serde.rs
index 573dcc764..059855544 100644
--- a/nodes/node/binary/src/config/kms/serde.rs
+++ b/nodes/node/binary/src/config/kms/serde.rs
@@ -1,6 +1,6 @@
 use std::collections::HashMap;
 
-use lb_key_management_system_service::{backend::preload::KeyId, keys::Key};
+use lb_key_management_system_service::{backend::preload::KeyId, keys::StoredKey};
 use serde::{Deserialize, Serialize};
 
 #[derive(Clone, Debug, Serialize, Deserialize, Default)]
@@ -9,15 +9,19 @@ pub struct Config {
     pub backend: PreloadKmsBackendSettings,
 }
 
+/// The keys the KMS preloads, in the on-disk encoding.
+///
+/// They become hardened [`Key`](lb_key_management_system_service::keys::Key)s
+/// when the service starts and are never written back from that form.
 #[derive(Clone, Debug, Serialize, Deserialize, Default)]
 #[serde(default)]
 pub struct PreloadKmsBackendSettings {
-    pub keys: HashMap<KeyId, Key>,
+    pub keys: HashMap<KeyId, StoredKey>,
 }
 
 #[cfg(test)]
 mod tests {
-    use lb_key_management_system_service::keys::{Ed25519Key, Key, ZkKey};
+    use lb_key_management_system_service::keys::{StoredKey, UnsecuredEd25519Key, UnsecuredZkKey};
     use num_bigint::BigUint;
     use rand::rngs::OsRng;
 
@@ -29,11 +33,11 @@ mod tests {
             keys: [
                 (
                     "test1".into(),
-                    Key::Ed25519(Ed25519Key::generate(&mut OsRng)),
+                    StoredKey::Ed25519(UnsecuredEd25519Key::generate(&mut OsRng)),
                 ),
                 (
                     "test2".into(),
-                    Key::Zk(ZkKey::new(BigUint::from_bytes_le(&[1u8; 32]).into())),
+                    StoredKey::Zk(UnsecuredZkKey::from(BigUint::from_bytes_le(&[1u8; 32]))),
                 ),
             ]
             .into(),
diff --git a/nodes/node/binary/src/config/mod.rs b/nodes/node/binary/src/config/mod.rs
index 4329dad42..6228bf4a6 100644
--- a/nodes/node/binary/src/config/mod.rs
+++ b/nodes/node/binary/src/config/mod.rs
@@ -10,7 +10,7 @@ use lb_core::sdp::ProviderId;
 use lb_groth16::fr_from_bytes;
 use lb_key_management_system_service::{
     backend::preload::KeyId,
-    keys::{Key, UnsecuredZkKey, ZkPublicKey},
+    keys::{StoredKey, UnsecuredZkKey, ZkPublicKey},
 };
 use lb_libp2p::{Multiaddr, ed25519::SecretKey};
 use lb_tracing::{
@@ -116,7 +116,7 @@ impl UserConfig {
                 "Blend non-ephemeral signing key '{key_id}' not found in KMS"
             ));
         };
-        let Key::Ed25519(secret_key) = key else {
+        let StoredKey::Ed25519(secret_key) = key else {
             return Err("Blend non-ephemeral signing key must be Ed25519".to_owned());
         };
         Ok(ProviderId(secret_key.public_key()))
@@ -127,7 +127,7 @@ impl UserConfig {
         let Some(key) = self.kms.backend.keys.get(key_id) else {
             return Err(format!("Blend ZK signing key '{key_id}' not found in KMS"));
         };
-        let Key::Zk(secret_key) = key else {
+        let StoredKey::Zk(secret_key) = key else {
             return Err("Blend ZK signing key must be Zk".to_owned());
         };
         Ok((key_id.to_owned(), secret_key.to_public_key()))
diff --git a/services/blend/Cargo.toml b/services/blend/Cargo.toml
index 71de6c1c4..109aa3169 100644
--- a/services/blend/Cargo.toml
+++ b/services/blend/Cargo.toml
@@ -24,7 +24,7 @@ lb-codec                         = { workspace = true }
 lb-core                          = { workspace = true }
 lb-cryptarchia-engine            = { workspace = true }
 lb-groth16                       = { workspace = true }
-lb-key-management-system-service = { features = ["unsafe"], workspace = true }
+lb-key-management-system-service = { workspace = true }
 lb-ledger                        = { workspace = true }
 lb-libp2p                        = { workspace = true }
 lb-log-targets                   = { workspace = true }
diff --git a/services/blend/src/core/backends/libp2p/settings.rs b/services/blend/src/core/backends/libp2p/settings.rs
index 3abde5ca1..5fa521de3 100644
--- a/services/blend/src/core/backends/libp2p/settings.rs
+++ b/services/blend/src/core/backends/libp2p/settings.rs
@@ -31,9 +31,7 @@ pub struct Libp2pBlendBackendSettings {
 impl BlendConfig<Libp2pBlendBackendSettings> {
     #[must_use]
     pub fn keypair(&self) -> Keypair {
-        let mut secret_key_bytes = *self.non_ephemeral_signing_key.as_bytes();
-        Keypair::ed25519_from_bytes(&mut secret_key_bytes)
-            .expect("Cryptographic secret key should be a valid Ed25519 private key.")
+        self.swarm_identity.clone()
     }
 
     #[must_use]
diff --git a/services/blend/src/core/mod.rs b/services/blend/src/core/mod.rs
index 0e6823b71..55904423a 100644
--- a/services/blend/src/core/mod.rs
+++ b/services/blend/src/core/mod.rs
@@ -56,7 +56,9 @@ use lb_core::sdp::ActivityMetadata;
 use lb_key_management_system_service::{
     api::KmsServiceApi,
     keys::{KeyOperators, PublicKeyEncoding},
-    operators::ed25519::exfiltrate_secret_key::LeakSecretKeyOperator,
+    operators::ed25519::{
+        derive_x25519::DeriveX25519Operator, libp2p_identity::LibP2pIdentityOperator,
+    },
 };
 use lb_log_targets::{blend, diagnostic::BLEND_REACHABILITY};
 use lb_network_service::NetworkService;
@@ -67,6 +69,7 @@ use lb_services_utils::{
     wait_until_services_are_ready,
 };
 use lb_time_service::TimeService;
+use libp2p::identity::Keypair;
 use overwatch::{
     OpaqueServiceResourcesHandle,
     overwatch::OverwatchHandle,
@@ -376,26 +379,48 @@ where
             panic!("Key with specified ID is not a ZK key.");
         };
 
-        // TODO: This will go once we do not need to pass the secret key anymore, i.e.,
-        // when we have libp2p integration with KMS.
-        let non_ephemeral_signing_key = {
+        let PublicKeyEncoding::Ed25519(non_ephemeral_signing_public_key) = kms_api
+            .public_key(blend_config.non_ephemeral_signing_key_id.clone())
+            .await
+            .expect("Non-ephemeral signing public key for provided ID should be stored in KMS.")
+        else {
+            panic!("Key with specified ID is not an Ed25519 key.");
+        };
+
+        let non_ephemeral_encryption_key = {
             let (sender, receiver) = oneshot::channel();
             kms_api
                 .execute(
                     blend_config.non_ephemeral_signing_key_id.clone(),
-                    KeyOperators::Ed25519(Box::new(LeakSecretKeyOperator::new(sender))),
+                    KeyOperators::Ed25519(Box::new(DeriveX25519Operator::new(sender))),
                 )
                 .await
-                .expect("Failed to interact with KMS to fetch non-ephemeral signing key.");
+                .expect("Failed to interact with KMS to derive the non-ephemeral encryption key.");
             receiver
                 .await
-                .expect("Failed to retrieve non-ephemeral signing key from KMS.")
+                .expect("Failed to retrieve non-ephemeral encryption key from KMS.")
+        };
+
+        let swarm_identity = {
+            let (sender, receiver) = oneshot::channel();
+            kms_api
+                .execute(
+                    blend_config.non_ephemeral_signing_key_id.clone(),
+                    KeyOperators::Ed25519(Box::new(LibP2pIdentityOperator::new(sender))),
+                )
+                .await
+                .expect("Failed to interact with KMS to fetch the swarm identity.");
+            let identity = receiver
+                .await
+                .expect("Failed to retrieve swarm identity from KMS.");
+            Keypair::ed25519_from_bytes(&mut *identity.into_bytes())
+                .expect("Non-ephemeral signing key should be a valid Ed25519 secret key.")
         };
 
         let public_epoch_stream =
             membership::chain::subscribe::<ChainService, NodeId, TimeBackend, RuntimeServiceId>(
                 overwatch_handle,
-                non_ephemeral_signing_key.public_key(),
+                non_ephemeral_signing_public_key,
                 Some(zk_public_key),
                 "blend_core_service",
             )
@@ -409,7 +434,8 @@ where
         // Initialize components for the service.
         let running_blend_config = RunningBlendConfig {
             backend: blend_config.backend,
-            non_ephemeral_signing_key,
+            non_ephemeral_encryption_key,
+            swarm_identity,
             num_blend_layers: blend_config.num_blend_layers,
             minimum_network_size: blend_config.minimum_network_size,
             scheduler: blend_config.scheduler,
@@ -772,7 +798,7 @@ where
     >::new(
         current_epoch_public_info.membership.clone(),
         EpochCryptographicProcessorSettings {
-            non_ephemeral_encryption_key: blend_config.non_ephemeral_signing_key.derive_x25519(),
+            non_ephemeral_encryption_key: blend_config.non_ephemeral_encryption_key.clone(),
             num_blend_layers: blend_config.num_blend_layers,
             pow_mining_pool: Arc::clone(&blend_config.pow_mining_pool),
             spent_core_quota,
@@ -1790,9 +1816,7 @@ where
                 CurrentEpochCryptographicProcessor::new(
                     new_epoch_info.membership.clone(),
                     EpochCryptographicProcessorSettings {
-                        non_ephemeral_encryption_key: settings
-                            .non_ephemeral_signing_key
-                            .derive_x25519(),
+                        non_ephemeral_encryption_key: settings.non_ephemeral_encryption_key.clone(),
                         num_blend_layers: settings.num_blend_layers,
                         pow_mining_pool: Arc::clone(&settings.pow_mining_pool),
                         spent_core_quota: Quota::ZERO,
diff --git a/services/blend/src/core/settings.rs b/services/blend/src/core/settings.rs
index 8d4384eb5..2b3354cfc 100644
--- a/services/blend/src/core/settings.rs
+++ b/services/blend/src/core/settings.rs
@@ -1,10 +1,11 @@
 use std::{num::NonZeroU64, sync::Arc};
 
 use lb_core::blend::core_quota;
-use lb_key_management_system_service::{backend::preload::KeyId, keys::UnsecuredEd25519Key};
+use lb_key_management_system_service::{backend::preload::KeyId, keys::X25519PrivateKey};
 use lb_poq::Quota;
 use lb_services_utils::overwatch::{RecoveryData, StorageRecoverySettings};
 use lb_utils::math::PositiveF64;
+use libp2p::identity::Keypair;
 use rayon::ThreadPool;
 use serde::{Deserialize, Serialize};
 
@@ -27,15 +28,20 @@ pub struct StartingBlendConfig<BackendSettings, NetworkSettings> {
     pub abstain_on_failure: bool,
 }
 
-/// Same values as [`StartingBlendConfig`] but with the secret key exfiltrated
-/// from the KMS.
+/// Same values as [`StartingBlendConfig`], plus what the service needs from
+/// the non-ephemeral signing key after start.
+///
+/// That is the encryption key derived from it and the swarm identity built
+/// from it, both obtained from the KMS once; the signing key itself never
+/// reaches this type.
 #[derive(Clone)]
 pub struct RunningBlendConfig<BackendSettings> {
     pub backend: BackendSettings,
     pub scheduler: SchedulerSettings,
     pub time: TimingSettings,
     pub zk: ZkSettings,
-    pub non_ephemeral_signing_key: UnsecuredEd25519Key,
+    pub non_ephemeral_encryption_key: X25519PrivateKey,
+    pub swarm_identity: Keypair,
     pub num_blend_layers: NonZeroU64,
     pub minimum_network_size: NonZeroU64,
     pub data_replication_factor: u64,
diff --git a/services/blend/src/core/tests/mod.rs b/services/blend/src/core/tests/mod.rs
index c943f4ed4..d23068935 100644
--- a/services/blend/src/core/tests/mod.rs
+++ b/services/blend/src/core/tests/mod.rs
@@ -155,7 +155,7 @@ async fn test_handle_incoming_blend_message() {
     let public_info = new_epoch_info(epoch, membership.clone(), &settings);
     let mut processor = new_crypto_processor(
         EpochCryptographicProcessorSettings {
-            non_ephemeral_encryption_key: settings.non_ephemeral_signing_key.derive_x25519(),
+            non_ephemeral_encryption_key: settings.non_ephemeral_encryption_key.clone(),
             num_blend_layers: settings.num_blend_layers,
             pow_mining_pool: Arc::new(ThreadPoolBuilder::new().build().unwrap()),
             spent_core_quota: Quota::ZERO,
@@ -209,7 +209,7 @@ async fn test_handle_incoming_blend_message() {
     let public_info = new_epoch_info(epoch, membership.clone(), &settings);
     let mut new_processor = new_crypto_processor(
         EpochCryptographicProcessorSettings {
-            non_ephemeral_encryption_key: settings.non_ephemeral_signing_key.derive_x25519(),
+            non_ephemeral_encryption_key: settings.non_ephemeral_encryption_key.clone(),
             num_blend_layers: settings.num_blend_layers,
             pow_mining_pool: Arc::new(ThreadPoolBuilder::new().build().unwrap()),
             spent_core_quota: Quota::ZERO,
@@ -310,7 +310,7 @@ async fn test_handle_incoming_blend_message() {
     epoch = epoch.strict_add(1.into());
     let mut future_processor = new_crypto_processor(
         EpochCryptographicProcessorSettings {
-            non_ephemeral_encryption_key: settings.non_ephemeral_signing_key.derive_x25519(),
+            non_ephemeral_encryption_key: settings.non_ephemeral_encryption_key.clone(),
             num_blend_layers: settings.num_blend_layers,
             pow_mining_pool: Arc::new(ThreadPoolBuilder::new().build().unwrap()),
             spent_core_quota: Quota::ZERO,
@@ -396,7 +396,7 @@ async fn test_duplicate_decapsulated_replica_handled_gracefully() {
     let public_info = new_epoch_info(epoch, membership.clone(), &settings);
     let mut processor = new_crypto_processor(
         EpochCryptographicProcessorSettings {
-            non_ephemeral_encryption_key: settings.non_ephemeral_signing_key.derive_x25519(),
+            non_ephemeral_encryption_key: settings.non_ephemeral_encryption_key.clone(),
             num_blend_layers: settings.num_blend_layers,
             pow_mining_pool: Arc::new(ThreadPoolBuilder::new().build().unwrap()),
             spent_core_quota: Quota::ZERO,
@@ -491,7 +491,7 @@ async fn test_handle_incoming_blend_message_with_invalid_poq() {
     let public_info_0 = new_epoch_info(epoch_0, membership.clone(), &settings);
     let mut processor_0 = new_crypto_processor(
         EpochCryptographicProcessorSettings {
-            non_ephemeral_encryption_key: settings.non_ephemeral_signing_key.derive_x25519(),
+            non_ephemeral_encryption_key: settings.non_ephemeral_encryption_key.clone(),
             num_blend_layers: settings.num_blend_layers,
             pow_mining_pool: Arc::new(ThreadPoolBuilder::new().build().unwrap()),
             spent_core_quota: Quota::ZERO,
@@ -512,7 +512,7 @@ async fn test_handle_incoming_blend_message_with_invalid_poq() {
     let public_info_1 = new_epoch_info(epoch_1, membership, &settings);
     let processor_1 = new_crypto_processor(
         EpochCryptographicProcessorSettings {
-            non_ephemeral_encryption_key: settings.non_ephemeral_signing_key.derive_x25519(),
+            non_ephemeral_encryption_key: settings.non_ephemeral_encryption_key.clone(),
             num_blend_layers: settings.num_blend_layers,
             pow_mining_pool: Arc::new(ThreadPoolBuilder::new().build().unwrap()),
             spent_core_quota: Quota::ZERO,
@@ -655,7 +655,7 @@ async fn test_handle_epoch_event_discards_queued_proposals() {
     let public_info = new_epoch_info(epoch, membership, &settings);
     let crypto_processor = new_crypto_processor(
         EpochCryptographicProcessorSettings {
-            non_ephemeral_encryption_key: settings.non_ephemeral_signing_key.derive_x25519(),
+            non_ephemeral_encryption_key: settings.non_ephemeral_encryption_key.clone(),
             num_blend_layers: settings.num_blend_layers,
             pow_mining_pool: Arc::new(ThreadPoolBuilder::new().build().unwrap()),
             spent_core_quota: Quota::ZERO,
@@ -679,7 +679,7 @@ async fn test_handle_epoch_event_discards_queued_proposals() {
     let (old_crypto_processor, old_scheduler) = (
         new_crypto_processor(
             EpochCryptographicProcessorSettings {
-                non_ephemeral_encryption_key: settings.non_ephemeral_signing_key.derive_x25519(),
+                non_ephemeral_encryption_key: settings.non_ephemeral_encryption_key.clone(),
                 num_blend_layers: settings.num_blend_layers,
                 pow_mining_pool: Arc::new(ThreadPoolBuilder::new().build().unwrap()),
                 spent_core_quota: Quota::ZERO,
@@ -742,7 +742,7 @@ async fn test_handle_epoch_event() {
     let public_info = new_epoch_info(epoch, membership.clone(), &settings);
     let crypto_processor = new_crypto_processor(
         EpochCryptographicProcessorSettings {
-            non_ephemeral_encryption_key: settings.non_ephemeral_signing_key.derive_x25519(),
+            non_ephemeral_encryption_key: settings.non_ephemeral_encryption_key.clone(),
             num_blend_layers: settings.num_blend_layers,
             pow_mining_pool: Arc::new(ThreadPoolBuilder::new().build().unwrap()),
             spent_core_quota: Quota::ZERO,
@@ -894,7 +894,7 @@ async fn test_handle_epoch_event_membership_change_rewires_backend_and_generator
     let public_info = new_epoch_info(epoch, membership.clone(), &settings);
     let crypto_processor = new_crypto_processor(
         EpochCryptographicProcessorSettings {
-            non_ephemeral_encryption_key: settings.non_ephemeral_signing_key.derive_x25519(),
+            non_ephemeral_encryption_key: settings.non_ephemeral_encryption_key.clone(),
             num_blend_layers: settings.num_blend_layers,
             pow_mining_pool: Arc::new(ThreadPoolBuilder::new().build().unwrap()),
             spent_core_quota: Quota::ZERO,
@@ -994,7 +994,7 @@ async fn transition_to_new_epoch_with_secret(secret_epoch: Epoch) -> Vec<Epoch>
     let public_info = new_epoch_info(epoch, membership.clone(), &settings);
     let crypto_processor = new_crypto_processor(
         EpochCryptographicProcessorSettings {
-            non_ephemeral_encryption_key: settings.non_ephemeral_signing_key.derive_x25519(),
+            non_ephemeral_encryption_key: settings.non_ephemeral_encryption_key.clone(),
             num_blend_layers: settings.num_blend_layers,
             pow_mining_pool: Arc::new(ThreadPoolBuilder::new().build().unwrap()),
             spent_core_quota: Quota::ZERO,
@@ -1105,7 +1105,7 @@ async fn test_handle_epoch_event_empty_epoch_retires() {
     let public_info = new_epoch_info(epoch, membership.clone(), &settings);
     let crypto_processor = new_crypto_processor(
         EpochCryptographicProcessorSettings {
-            non_ephemeral_encryption_key: settings.non_ephemeral_signing_key.derive_x25519(),
+            non_ephemeral_encryption_key: settings.non_ephemeral_encryption_key.clone(),
             num_blend_layers: settings.num_blend_layers,
             pow_mining_pool: Arc::new(ThreadPoolBuilder::new().build().unwrap()),
             spent_core_quota: Quota::ZERO,
@@ -1180,7 +1180,7 @@ async fn test_handle_epoch_event_non_empty_without_local_core_path_retires() {
     let public_info = new_epoch_info(epoch, membership, &settings);
     let crypto_processor = new_crypto_processor(
         EpochCryptographicProcessorSettings {
-            non_ephemeral_encryption_key: settings.non_ephemeral_signing_key.derive_x25519(),
+            non_ephemeral_encryption_key: settings.non_ephemeral_encryption_key.clone(),
             num_blend_layers: settings.num_blend_layers,
             pow_mining_pool: Arc::new(ThreadPoolBuilder::new().build().unwrap()),
             spent_core_quota: Quota::ZERO,
@@ -1674,7 +1674,7 @@ async fn test_proof_generator_epoch_binding() {
 
     let mut generator_0 = new_crypto_processor(
         EpochCryptographicProcessorSettings {
-            non_ephemeral_encryption_key: settings.non_ephemeral_signing_key.derive_x25519(),
+            non_ephemeral_encryption_key: settings.non_ephemeral_encryption_key.clone(),
             num_blend_layers: settings.num_blend_layers,
             pow_mining_pool: Arc::new(ThreadPoolBuilder::new().build().unwrap()),
             spent_core_quota: Quota::ZERO,
@@ -1685,7 +1685,7 @@ async fn test_proof_generator_epoch_binding() {
 
     let mut generator_1 = new_crypto_processor(
         EpochCryptographicProcessorSettings {
-            non_ephemeral_encryption_key: settings.non_ephemeral_signing_key.derive_x25519(),
+            non_ephemeral_encryption_key: settings.non_ephemeral_encryption_key.clone(),
             num_blend_layers: settings.num_blend_layers,
             pow_mining_pool: Arc::new(ThreadPoolBuilder::new().build().unwrap()),
             spent_core_quota: Quota::ZERO,
diff --git a/services/blend/src/core/tests/utils.rs b/services/blend/src/core/tests/utils.rs
index 061cfd6d7..d568a52d9 100644
--- a/services/blend/src/core/tests/utils.rs
+++ b/services/blend/src/core/tests/utils.rs
@@ -84,6 +84,14 @@ pub fn new_membership(size: u8) -> (Membership<NodeId>, UnsecuredEd25519Key) {
     )
 }
 
+/// The libp2p identity the running config carries in place of the raw key.
+pub fn swarm_identity(local_private_key: UnsecuredEd25519Key) -> libp2p::identity::Keypair {
+    let mut bytes = local_private_key.to_bytes();
+    drop(local_private_key);
+    libp2p::identity::Keypair::ed25519_from_bytes(&mut *bytes)
+        .expect("test key is a valid Ed25519 secret key")
+}
+
 /// Creates a [`BlendConfig`] with the given parameters and reasonable defaults
 /// for the rest.
 pub fn settings<BackendSettings>(
@@ -106,7 +114,8 @@ pub fn settings<BackendSettings>(
         zk: ZkSettings {
             secret_key_kms_id: "test-key".to_owned(),
         },
-        non_ephemeral_signing_key: local_private_key,
+        non_ephemeral_encryption_key: local_private_key.derive_x25519(),
+        swarm_identity: swarm_identity(local_private_key),
         num_blend_layers: NonZeroU64::try_from(1).unwrap(),
         minimum_network_size,
         data_replication_factor,
diff --git a/services/blend/src/edge/backends/libp2p/mod.rs b/services/blend/src/edge/backends/libp2p/mod.rs
index cdcca0a06..0bfbe4e8b 100644
--- a/services/blend/src/edge/backends/libp2p/mod.rs
+++ b/services/blend/src/edge/backends/libp2p/mod.rs
@@ -6,7 +6,6 @@ use lb_blend::{
     message::encap::validated::EncapsulatedMessageWithVerifiedPublicHeader,
     scheduling::membership::Membership,
 };
-use lb_key_management_system_service::keys::UnsecuredEd25519Key;
 use lb_libp2p::identity::Keypair;
 use lb_log_targets::blend;
 use libp2p::PeerId;
@@ -39,17 +38,12 @@ impl<RuntimeServiceId> BlendBackend<PeerId, RuntimeServiceId> for Libp2pBlendBac
         overwatch_handle: OverwatchHandle<RuntimeServiceId>,
         membership: Membership<PeerId>,
         rng: Rng,
-        non_ephemeral_signing_key: UnsecuredEd25519Key,
+        swarm_identity: Keypair,
     ) -> Self
     where
         Rng: RngCore + Send + 'static,
     {
         let (swarm_command_sender, swarm_command_receiver) = mpsc::channel(CHANNEL_SIZE);
-        let swarm_identity = {
-            let mut non_ephemeral_signing_key_bytes = non_ephemeral_signing_key.to_bytes();
-            Keypair::ed25519_from_bytes(&mut non_ephemeral_signing_key_bytes)
-                .expect("Cryptographic secret key should be a valid Ed25519 private key.")
-        };
         let swarm = BlendSwarm::new(
             settings,
             membership,
diff --git a/services/blend/src/edge/backends/libp2p/settings.rs b/services/blend/src/edge/backends/libp2p/settings.rs
index b202e3c3e..cbd125a95 100644
--- a/services/blend/src/edge/backends/libp2p/settings.rs
+++ b/services/blend/src/edge/backends/libp2p/settings.rs
@@ -19,8 +19,6 @@ pub struct Libp2pBlendBackendSettings {
 impl BlendConfig<Libp2pBlendBackendSettings> {
     #[must_use]
     pub fn keypair(&self) -> Keypair {
-        let mut secret_key_bytes = *self.non_ephemeral_signing_key.as_bytes();
-        Keypair::ed25519_from_bytes(&mut secret_key_bytes)
-            .expect("Cryptographic secret key should be a valid Ed25519 private key.")
+        self.swarm_identity.clone()
     }
 }
diff --git a/services/blend/src/edge/backends/mod.rs b/services/blend/src/edge/backends/mod.rs
index d1a0b6490..196bd30f9 100644
--- a/services/blend/src/edge/backends/mod.rs
+++ b/services/blend/src/edge/backends/mod.rs
@@ -1,8 +1,8 @@
+use ::libp2p::identity::Keypair;
 use lb_blend::{
     message::encap::validated::EncapsulatedMessageWithVerifiedPublicHeader,
     scheduling::membership::Membership,
 };
-use lb_key_management_system_service::keys::UnsecuredEd25519Key;
 use overwatch::overwatch::handle::OverwatchHandle;
 use rand::RngCore;
 
@@ -21,8 +21,7 @@ where
         overwatch_handle: OverwatchHandle<RuntimeServiceId>,
         membership: Membership<NodeId>,
         rng: Rng,
-        // TODO: This should go once we find a way to integrate KMS into libp2p.
-        non_ephemeral_signing_key: UnsecuredEd25519Key,
+        swarm_identity: Keypair,
     ) -> Self
     where
         Rng: RngCore + Send + 'static;
diff --git a/services/blend/src/edge/handlers.rs b/services/blend/src/edge/handlers.rs
index 58eead559..921454337 100644
--- a/services/blend/src/edge/handlers.rs
+++ b/services/blend/src/edge/handlers.rs
@@ -70,7 +70,7 @@ where
             overwatch_handle,
             membership,
             ChaCha20Rng::from_entropy(),
-            settings.non_ephemeral_signing_key,
+            settings.swarm_identity,
         );
         Self {
             cryptographic_processor,
diff --git a/services/blend/src/edge/mod.rs b/services/blend/src/edge/mod.rs
index 2ded50a70..9834cc0f1 100644
--- a/services/blend/src/edge/mod.rs
+++ b/services/blend/src/edge/mod.rs
@@ -22,13 +22,15 @@ use lb_blend::scheduling::{
 };
 use lb_chain_service::api::CryptarchiaServiceData;
 use lb_key_management_system_service::{
-    api::KmsServiceApi, keys::KeyOperators,
-    operators::ed25519::exfiltrate_secret_key::LeakSecretKeyOperator,
+    api::KmsServiceApi,
+    keys::{KeyOperators, PublicKeyEncoding},
+    operators::ed25519::libp2p_identity::LibP2pIdentityOperator,
 };
 use lb_log_targets::blend;
 use lb_network_service::NetworkService;
 use lb_services_utils::wait_until_services_are_ready;
 use lb_time_service::TimeService;
+use libp2p::identity::Keypair;
 use overwatch::{
     OpaqueServiceResourcesHandle,
     overwatch::OverwatchHandle,
@@ -220,28 +222,34 @@ where
             overwatch_handle.relay::<PreloadKmsService<_>>().await?,
         );
 
-        // TODO: This will go once we do not need to pass the secret key anymore, i.e.,
-        // when we have libp2p integration with KMS.
-        let non_ephemeral_signing_key = {
+        let PublicKeyEncoding::Ed25519(non_ephemeral_signing_public_key) = kms
+            .public_key(settings.non_ephemeral_signing_key_id.clone())
+            .await?
+        else {
+            return Err("Key with specified ID is not an Ed25519 key.".into());
+        };
+        let swarm_identity = {
             let (sender, receiver) = oneshot::channel();
             kms.execute(
                 settings.non_ephemeral_signing_key_id,
-                KeyOperators::Ed25519(Box::new(LeakSecretKeyOperator::new(sender))),
+                KeyOperators::Ed25519(Box::new(LibP2pIdentityOperator::new(sender))),
             )
             .await
-            .expect("Failed to interact with KMS to fetch non-ephemeral signing key.");
-            receiver
+            .expect("Failed to interact with KMS to fetch the swarm identity.");
+            let identity = receiver
                 .await
-                .expect("Failed to retrieve non-ephemeral signing key from KMS.")
+                .expect("Failed to retrieve swarm identity from KMS.");
+            Keypair::ed25519_from_bytes(&mut *identity.into_bytes())
+                .expect("Non-ephemeral signing key should be a valid Ed25519 secret key.")
         };
         let local_node_id =
-            NodeId::try_from_provider_id(&non_ephemeral_signing_key.public_key().to_bytes())
+            NodeId::try_from_provider_id(&non_ephemeral_signing_public_key.to_bytes())
                 .expect("non-ephemeral signing key should decode into a valid node id");
 
         let public_epoch_stream =
             membership::chain::subscribe::<ChainService, NodeId, TimeBackend, RuntimeServiceId>(
                 &overwatch_handle,
-                non_ephemeral_signing_key.public_key(),
+                non_ephemeral_signing_public_key,
                 // No ZK stuff needs to be computed by edge nodes, so no ZK key is specified here.
                 None,
                 "blend_edge_service",
@@ -258,7 +266,7 @@ where
             RunningSettings::<Backend, _, _> {
                 backend: settings.backend,
                 cover: settings.cover,
-                non_ephemeral_signing_key,
+                swarm_identity,
                 num_blend_layers: settings.num_blend_layers,
                 minimum_network_size: settings.minimum_network_size,
                 time: settings.time,
diff --git a/services/blend/src/edge/settings.rs b/services/blend/src/edge/settings.rs
index 5db944788..f9a86693c 100644
--- a/services/blend/src/edge/settings.rs
+++ b/services/blend/src/edge/settings.rs
@@ -1,8 +1,9 @@
 use core::num::NonZeroU64;
 use std::sync::Arc;
 
-use lb_key_management_system_service::{backend::preload::KeyId, keys::UnsecuredEd25519Key};
+use lb_key_management_system_service::backend::preload::KeyId;
 use lb_poq::Quota;
+use libp2p::identity::Keypair;
 use rayon::ThreadPool;
 
 use crate::{
@@ -25,13 +26,16 @@ pub struct StartingBlendConfig<BackendSettings, NetworkSettings> {
     pub abstain_on_failure: bool,
 }
 
-/// Same values as [`StartingBlendConfig`] but with the secret key exfiltrated
-/// from the KMS.
+/// Same values as [`StartingBlendConfig`], plus the swarm identity built
+/// from the non-ephemeral signing key.
+///
+/// The identity is obtained from the KMS once; the signing key itself never
+/// reaches this type.
 #[derive(Clone)]
 pub struct RunningBlendConfig<BackendSettings> {
     pub backend: BackendSettings,
     pub time: TimingSettings,
-    pub non_ephemeral_signing_key: UnsecuredEd25519Key,
+    pub swarm_identity: Keypair,
     pub num_blend_layers: NonZeroU64,
     pub minimum_network_size: NonZeroU64,
     pub cover: CoverTrafficSettings,
diff --git a/services/blend/src/edge/tests/utils.rs b/services/blend/src/edge/tests/utils.rs
index 7084483f1..55b0e6469 100644
--- a/services/blend/src/edge/tests/utils.rs
+++ b/services/blend/src/edge/tests/utils.rs
@@ -17,7 +17,6 @@ use lb_blend::{
         },
     },
 };
-use lb_key_management_system_service::keys::UnsecuredEd25519Key;
 use overwatch::overwatch::{OverwatchHandle, commands::OverwatchCommand};
 use rand::{RngCore, rngs::OsRng};
 use rayon::ThreadPoolBuilder;
@@ -190,7 +189,10 @@ pub fn settings(
             rounds_per_observation_window: NonZeroU64::new(1).unwrap(),
             epoch_transition_period: Duration::from_secs(1),
         },
-        non_ephemeral_signing_key: key(local_id).0,
+        swarm_identity: libp2p::identity::Keypair::ed25519_from_bytes(
+            &mut *key(local_id).0.to_bytes(),
+        )
+        .expect("test key is a valid Ed25519 secret key"),
         num_blend_layers: TEST_BLEND_LAYERS,
         max_blend_delay_in_rounds: TEST_MAX_BLEND_DELAY,
         backend: msg_sender,
@@ -221,7 +223,7 @@ where
         _: OverwatchHandle<RuntimeServiceId>,
         membership: Membership<NodeId>,
         _: Rng,
-        _: UnsecuredEd25519Key,
+        _: ::libp2p::identity::Keypair,
     ) -> Self
     where
         Rng: RngCore + Send + 'static,
diff --git a/services/chain/chain-leader/Cargo.toml b/services/chain/chain-leader/Cargo.toml
index e0ab58e97..550e4a5b0 100644
--- a/services/chain/chain-leader/Cargo.toml
+++ b/services/chain/chain-leader/Cargo.toml
@@ -22,7 +22,7 @@ lb-codec                         = { workspace = true }
 lb-core                          = { workspace = true }
 lb-cryptarchia-engine            = { workspace = true }
 lb-groth16                       = { workspace = true }
-lb-key-management-system-service = { features = ["unsafe"], workspace = true }
+lb-key-management-system-service = { workspace = true }
 lb-ledger                        = { workspace = true }
 lb-log-targets                   = { workspace = true }
 lb-services-utils                = { workspace = true }
diff --git a/services/key-management-system/Cargo.toml b/services/key-management-system/Cargo.toml
index d5525caa0..748c88fbe 100644
--- a/services/key-management-system/Cargo.toml
+++ b/services/key-management-system/Cargo.toml
@@ -26,7 +26,6 @@ tracing                            = { workspace = true }
 
 [dev-dependencies]
 bytes                         = { workspace = true }
-lb-key-management-system-keys = { features = ["unsafe"], workspace = true }
 num-bigint                    = { workspace = true }
 rand                          = { features = ["getrandom"], workspace = true }
 serde                         = { features = ["std"], workspace = true }
@@ -34,4 +33,3 @@ serde_yaml                    = { workspace = true }
 
 [features]
 tokio-task-names = ["lb-key-management-system-operators/tokio-task-names"]
-unsafe           = ["lb-key-management-system-keys/unsafe", "lb-key-management-system-operators/unsafe"]
diff --git a/services/key-management-system/src/backend/preload.rs b/services/key-management-system/src/backend/preload.rs
index fd57785c0..90e205223 100644
--- a/services/key-management-system/src/backend/preload.rs
+++ b/services/key-management-system/src/backend/preload.rs
@@ -25,11 +25,12 @@ pub struct PreloadKMSBackend {
     keys: HashMap<KeyId, Key>,
 }
 
-/// This setting contains all [`Key`]s to be loaded into the
-/// [`PreloadKMSBackend`]. This implements [`serde::Serialize`] for users to
-/// populate the settings from bytes.
+/// The [`Key`]s to load into the [`PreloadKMSBackend`].
+///
+/// Deserializes from the on-disk
+/// [`StoredKey`](lb_key_management_system_keys::keys::StoredKey) encoding and
+/// is never serialized: once loaded, keys do not leave the backend.
 #[derive(Deserialize, Clone, Debug)]
-#[cfg_attr(any(test, feature = "unsafe"), derive(serde::Serialize))]
 pub struct PreloadKMSBackendSettings {
     pub keys: HashMap<KeyId, Key>,
 }
@@ -121,7 +122,8 @@ mod tests {
 
     use bytes::{Bytes as RawBytes, Bytes};
     use lb_key_management_system_keys::keys::{
-        Ed25519Key, PayloadEncoding, ZkKey, secured_key::SecureKeyOperator,
+        Ed25519Key, PayloadEncoding, StoredKey, UnsecuredEd25519Key, UnsecuredZkKey,
+        secured_key::SecureKeyOperator,
     };
     use num_bigint::BigUint;
     use rand::rngs::OsRng;
@@ -234,30 +236,39 @@ mod tests {
         assert!(matches!(backend.register(&key_id, key), Ok(())));
     }
 
+    /// The settings load from the on-disk key encoding, which is what the
+    /// node's config and keystore write.
     #[test]
-    fn serde_keys_from_yaml() {
-        let preloaded_keys = PreloadKMSBackendSettings {
+    fn settings_load_from_stored_keys_yaml() {
+        #[derive(serde::Serialize)]
+        struct StoredSettings {
+            keys: HashMap<KeyId, StoredKey>,
+        }
+
+        let stored = StoredSettings {
             keys: [
                 (
-                    "test1".into(),
-                    Key::Ed25519(Ed25519Key::generate(&mut OsRng)),
+                    "test1".to_owned(),
+                    StoredKey::Ed25519(UnsecuredEd25519Key::generate(&mut OsRng)),
                 ),
                 (
-                    "test2".into(),
-                    Key::Zk(ZkKey::new(BigUint::from_bytes_le(&[1u8; 32]).into())),
+                    "test2".to_owned(),
+                    StoredKey::Zk(UnsecuredZkKey::from(BigUint::from_bytes_le(&[1u8; 32]))),
                 ),
             ]
             .into(),
         };
 
-        let mut serialized_ouput = Vec::new();
-        serde_yaml::to_writer(&mut serialized_ouput, &preloaded_keys).unwrap();
+        let yaml = serde_yaml::to_string(&stored).unwrap();
+        let loaded: PreloadKMSBackendSettings = serde_yaml::from_str(&yaml).unwrap();
 
-        let deserialized_keys: PreloadKMSBackendSettings =
-            serde_yaml::from_slice(&serialized_ouput).unwrap();
-
-        assert_eq!(preloaded_keys.keys.len(), deserialized_keys.keys.len());
-        let original_key = preloaded_keys.keys.keys().next().unwrap();
-        assert!(deserialized_keys.keys.contains_key(original_key));
+        assert_eq!(loaded.keys.len(), stored.keys.len());
+        for (key_id, stored_key) in &stored.keys {
+            let loaded_key = loaded
+                .keys
+                .get(key_id)
+                .expect("key id survives the round trip");
+            assert_eq!(loaded_key.as_public_key(), stored_key.public_key());
+        }
     }
 }
diff --git a/services/wallet/Cargo.toml b/services/wallet/Cargo.toml
index e6691e696..526f96da4 100644
--- a/services/wallet/Cargo.toml
+++ b/services/wallet/Cargo.toml
@@ -23,7 +23,7 @@ lb-chain-service                 = { workspace = true }
 lb-core                          = { workspace = true }
 lb-cryptarchia-engine            = { workspace = true }
 lb-groth16                       = { workspace = true }
-lb-key-management-system-service = { features = ["unsafe"], workspace = true }
+lb-key-management-system-service = { workspace = true }
 lb-ledger                        = { workspace = true }
 lb-log-targets                   = { workspace = true }
 lb-mmr                           = { workspace = true }
diff --git a/services/wallet/src/lib.rs b/services/wallet/src/lib.rs
index 8bcafb805..ddc4cc2eb 100644
--- a/services/wallet/src/lib.rs
+++ b/services/wallet/src/lib.rs
@@ -22,9 +22,7 @@ use lb_core::{
         ops::{
             NoOpProof, OpRef, ZkAndEd25519Proof,
             channel::{ChannelId, config::ChannelConfigOp, inscribe::InscriptionOp},
-            leader_claim::{
-                LeaderClaimOp, RewardsRoot, VoucherCm, VoucherNullifier, VoucherSecret,
-            },
+            leader_claim::{LeaderClaimOp, RewardsRoot, VoucherCm, VoucherNullifier},
             sdp::{SDPActiveOp, SDPDeclareOp, SDPWithdrawOp},
         },
         traits::{Hashable as _, MantleTx, SignedMantleTx},
@@ -33,7 +31,6 @@ use lb_core::{
             tx_list::ops::OpsContext,
         },
     },
-    proofs::leader_claim_proof::{Groth16LeaderClaimProof, LeaderClaimPrivate, LeaderClaimPublic},
 };
 use lb_key_management_system_service::{
     api::{KmsServiceApi, KmsServiceData},
@@ -42,11 +39,10 @@ use lb_key_management_system_service::{
         Ed25519Key, KeyOperators, PayloadEncoding, SignatureEncoding, ZkPublicKey, ZkPublicKeys,
         ZkSignature, secured_key::SecuredKey,
     },
-    operators::zk::voucher::UnsafeVoucherOperator,
+    operators::zk::voucher::{ProveClaimError, ProveClaimOperator, VoucherOperator},
 };
 use lb_ledger::LedgerState;
 use lb_log_targets::wallet;
-use lb_mmr::MerklePath;
 use lb_services_utils::{
     overwatch::{RecoveryData, RecoveryOperator, StorageRecoverySettings},
     wait_until_services_are_ready,
@@ -54,7 +50,7 @@ use lb_services_utils::{
 use lb_storage_service::{
     api::chain::StorageChainApi, backends::StorageBackend, recovery::StorageRecoveryBackend,
 };
-use lb_utils::{bounded::BoundedError, tokio::task::spawn_blocking};
+use lb_utils::bounded::BoundedError;
 use lb_wallet::{WalletBalance, WalletBlock, WalletError};
 use overwatch::{
     DynError, OpaqueServiceResourcesHandle,
@@ -118,11 +114,8 @@ pub enum WalletServiceError {
     #[error("Transaction fee exceeded the configured max fee. tx_fee={tx_fee} > max_fee={max_fee}")]
     TxFeeExceedsMaxFee { max_fee: GasCost, tx_fee: GasCost },
 
-    #[error("PoC generation failed: {0:?}")]
-    PoCGenerationFailed(#[from] lb_core::proofs::leader_claim_proof::Error),
-
-    #[error(transparent)]
-    MerklePathWitness(#[from] lb_core::proofs::MerklePathWitnessError),
+    #[error("Proof of claim failed in KMS: {0}")]
+    ProveClaim(#[from] ProveClaimError),
 
     #[error("No claimable voucher found")]
     NoClaimableVoucher,
@@ -1010,22 +1003,30 @@ where
             .ok_or(WalletServiceError::VoucherNotFound(
                 leader_claim_op.voucher_nullifier,
             ))?;
-        let voucher_secret =
-            Self::derive_voucher_from_kms(kms, voucher_master_key_id.clone(), *voucher_index)
-                .await?;
+        let (voucher_cm, _) =
+            Self::voucher_from_kms(kms, voucher_master_key_id.clone(), *voucher_index).await?;
 
-        let voucher_cm = VoucherCm::from_secret(voucher_secret);
         let path = wallet
             .voucher_path_snapshot(tip, &voucher_cm)
             .map_err(WalletServiceError::WalletError)?
             .ok_or(WalletServiceError::VoucherMerklePathNotFound(voucher_cm))?;
-        let rewards_root = leader_claim_op.rewards_root;
 
-        // TODO: This should happen in KMS
-        let poc = spawn_blocking("logos/wallet/leader-claim-proof-blocking", move || {
-            Self::generate_poc(voucher_secret, &path, rewards_root, tx_hash)
-        })
-        .await??;
+        let (output_tx, output_rx) = oneshot::channel();
+        kms.execute(
+            voucher_master_key_id.clone(),
+            KeyOperators::Zk(Box::new(ProveClaimOperator::new(
+                (*voucher_index).into(),
+                path,
+                leader_claim_op.rewards_root.into(),
+                tx_hash.to_fr(),
+                output_tx,
+            ))),
+        )
+        .await
+        .map_err(|error| WalletServiceError::KmsApi(Box::new(error)))?;
+        let poc = output_rx.await.map_err(|_| {
+            WalletServiceError::KmsApi("KMS API did not respond with proof of claim".into())
+        })??;
 
         Ok(OpProof::PoC(poc))
     }
@@ -1170,25 +1171,6 @@ where
             .collect()
     }
 
-    fn generate_poc(
-        voucher_secret: VoucherSecret,
-        path: &MerklePath,
-        rewards_root: RewardsRoot,
-        tx_hash: TxHash,
-    ) -> Result<Groth16LeaderClaimProof, WalletServiceError> {
-        Ok(Groth16LeaderClaimProof::prove(
-            LeaderClaimPrivate::try_new(
-                LeaderClaimPublic {
-                    voucher_nullifier: VoucherNullifier::from_secret(voucher_secret).into(),
-                    voucher_root: rewards_root.into(),
-                    mantle_tx_hash: tx_hash.to_fr(),
-                },
-                path,
-                voucher_secret,
-            )?,
-        )?)
-    }
-
     /// Resolves the wallet-owned UTXOs that are eligible to lead at `tip`
     /// (or at the current tip when `tip` is `None`), paired with the key ids
     /// needed to build a leadership proof for them.
@@ -1235,15 +1217,13 @@ where
         resp_tx: Sender<Result<VoucherCm, WalletServiceError>>,
     ) {
         let index = state.next_new_voucher_index();
-        let secret = match Self::derive_voucher_from_kms(kms, master_key_id.clone(), index).await {
-            Ok(secret) => secret,
+        let (cm, nf) = match Self::voucher_from_kms(kms, master_key_id.clone(), index).await {
+            Ok(voucher) => voucher,
             Err(err) => {
                 Self::send_err(resp_tx, err);
                 return;
             }
         };
-        let cm = VoucherCm::from_secret(secret);
-        let nf = VoucherNullifier::from_secret(secret);
 
         state.add_next_known_voucher(cm, nf, master_key_id);
 
@@ -1252,30 +1232,24 @@ where
         }
     }
 
-    /// Derive voucher secret from KMS given master key and index.
-    // TODO: Use secure KMS operator that returns `VoucherCm` and `VoucherNullifier`
-    async fn derive_voucher_from_kms(
+    /// The commitment and nullifier of the voucher at `index` under the
+    /// master key, derived inside the KMS.
+    async fn voucher_from_kms(
         kms: &KmsServiceApi<Kms, RuntimeServiceId>,
         key_id: KeyId,
         index: u64,
-    ) -> Result<VoucherSecret, WalletServiceError> {
+    ) -> Result<(VoucherCm, VoucherNullifier), WalletServiceError> {
         let (output_tx, output_rx) = oneshot::channel();
         kms.execute(
             key_id,
-            KeyOperators::Zk(Box::new(UnsafeVoucherOperator::new(
-                index.into(),
-                output_tx,
-            ))),
+            KeyOperators::Zk(Box::new(VoucherOperator::new(index.into(), output_tx))),
         )
         .await
         .map_err(|error| WalletServiceError::KmsApi(Box::new(error)))?;
 
-        Ok(output_rx
-            .await
-            .map_err(|_| {
-                WalletServiceError::KmsApi("KMS API did not respond with voucher_cm".into())
-            })?
-            .into())
+        output_rx.await.map_err(|_| {
+            WalletServiceError::KmsApi("KMS API did not respond with the voucher".into())
+        })
     }
 
     async fn build_leader_claim_tx(
diff --git a/tests/src/common/wallet/funding.rs b/tests/src/common/wallet/funding.rs
index 1e7f80e00..69e08ab37 100644
--- a/tests/src/common/wallet/funding.rs
+++ b/tests/src/common/wallet/funding.rs
@@ -5,7 +5,7 @@ use lb_core::mantle::{
     ledger::{Inputs, Outputs},
     ops::transfer::TransferOp,
 };
-use lb_key_management_system_service::keys::{ZkKey, ZkPublicKey};
+use lb_key_management_system_service::keys::{UnsecuredZkKey, ZkPublicKey};
 use lb_testing_framework::configs::wallet::WalletAccount;
 use lb_wallet::WalletError;
 
@@ -124,7 +124,7 @@ impl WalletFundingSource {
     }
 
     #[must_use]
-    pub(crate) const fn signing_key(&self) -> &ZkKey {
+    pub(crate) const fn signing_key(&self) -> &UnsecuredZkKey {
         &self.account.secret_key
     }
 
diff --git a/tests/src/common/wallet/transaction/signing.rs b/tests/src/common/wallet/transaction/signing.rs
index 5cc739c18..66ed54abe 100644
--- a/tests/src/common/wallet/transaction/signing.rs
+++ b/tests/src/common/wallet/transaction/signing.rs
@@ -8,12 +8,12 @@ use lb_core::mantle::{
     traits::Hashable as _,
     transactions::{MantleTxBuilder, OpProofs, Ops, tx_list::ops::OpsContext},
 };
-use lb_key_management_system_service::keys::ZkKey;
+use lb_key_management_system_service::keys::UnsecuredZkKey;
 
 use super::{error::WalletTransactionError, signed::SignedWalletTransaction};
 use crate::common::wallet::WalletReservedInputs;
 
-pub(super) type WalletTransferSigners = HashMap<NoteId, ZkKey>;
+pub(super) type WalletTransferSigners = HashMap<NoteId, UnsecuredZkKey>;
 
 pub(super) fn sign_prepared_wallet_transaction(
     funded_builder: MantleTxBuilder,
@@ -67,7 +67,7 @@ pub(super) fn sign_prepared_wallet_transaction(
 /// funding inputs all come from a single wallet account.
 pub fn transfer_proofs_for_funded_wallet_tx(
     tx: &Ops,
-    signing_key: &ZkKey,
+    signing_key: &UnsecuredZkKey,
 ) -> Result<OpProofs, WalletTransactionError> {
     let tx_hash = tx.hash();
     let proofs = tx
@@ -82,7 +82,7 @@ pub fn transfer_proofs_for_funded_wallet_tx(
                 .iter()
                 .map(|_| signing_key.clone())
                 .collect::<Vec<_>>();
-            Ok(OpProof::ZkSig(ZkKey::multi_sign(
+            Ok(OpProof::ZkSig(UnsecuredZkKey::multi_sign(
                 &signing_keys,
                 &tx_hash.to_fr(),
             )?))
@@ -114,7 +114,7 @@ pub(super) fn build_transfer_proofs(
                 })
                 .collect::<Result<Vec<_>, _>>()?;
 
-            Ok(OpProof::ZkSig(ZkKey::multi_sign(
+            Ok(OpProof::ZkSig(UnsecuredZkKey::multi_sign(
                 &signing_keys,
                 &tx_hash.to_fr(),
             )?))
diff --git a/tests/src/cucumber/steps/nodes/operations/lifecycle.rs b/tests/src/cucumber/steps/nodes/operations/lifecycle.rs
index fbba248ee..5e9e03a58 100644
--- a/tests/src/cucumber/steps/nodes/operations/lifecycle.rs
+++ b/tests/src/cucumber/steps/nodes/operations/lifecycle.rs
@@ -637,7 +637,7 @@ fn prepare_config_patch(
 }
 
 pub(super) fn wallet_account_key_id(account: &WalletAccount) -> KeyId {
-    let key: Key = account.secret_key.clone().into();
+    let key: StoredKey = account.secret_key.clone().into();
     key_id_for_preload_backend(&key)
 }
 
@@ -654,7 +654,7 @@ fn remove_external_scenario_wallet_keys(
 
 pub(super) fn remove_external_scenario_wallet_keys_from_maps(
     known_keys: &mut HashMap<KeyId, lb_key_management_system_service::keys::ZkPublicKey>,
-    kms_keys: &mut HashMap<KeyId, Key>,
+    kms_keys: &mut HashMap<KeyId, StoredKey>,
     scenario_wallet_key_ids: &HashSet<KeyId>,
 ) {
     for key_id in scenario_wallet_key_ids {
diff --git a/tests/src/cucumber/steps/nodes/operations/mod.rs b/tests/src/cucumber/steps/nodes/operations/mod.rs
index bea223738..a4f04e535 100644
--- a/tests/src/cucumber/steps/nodes/operations/mod.rs
+++ b/tests/src/cucumber/steps/nodes/operations/mod.rs
@@ -14,7 +14,7 @@ use lb_chain_service::{ChainServiceInfo, CryptarchiaInfo, PhaseTag};
 use lb_config::kms::key_id_for_preload_backend;
 use lb_core::mantle::{Utxo, ops::OpId as _};
 use lb_http_api_common::paths::CRYPTARCHIA_INFO;
-use lb_key_management_system_service::{backend::preload::KeyId, keys::Key};
+use lb_key_management_system_service::{backend::preload::KeyId, keys::StoredKey};
 use lb_libp2p::PeerId;
 use lb_node::config::{
     DeploymentSettings, RunConfig,
diff --git a/tests/src/cucumber/steps/nodes/operations/tests.rs b/tests/src/cucumber/steps/nodes/operations/tests.rs
index 1750f66c3..163333396 100644
--- a/tests/src/cucumber/steps/nodes/operations/tests.rs
+++ b/tests/src/cucumber/steps/nodes/operations/tests.rs
@@ -73,7 +73,7 @@ mod wallet_name_tests {
 
 #[cfg(test)]
 mod scenario_wallet_key_tests {
-    use lb_key_management_system_service::keys::{Ed25519Key, secured_key::SecuredKey as _};
+    use lb_key_management_system_service::keys::UnsecuredEd25519Key;
 
     use super::*;
 
@@ -99,31 +99,31 @@ mod scenario_wallet_key_tests {
             .iter()
             .chain(std::iter::once(&sponsored_fee_account))
         {
-            let key: Key = account.secret_key.clone().into();
+            let key: StoredKey = account.secret_key.clone().into();
             let key_id = key_id_for_preload_backend(&key);
             kms_keys.insert(key_id.clone(), key);
-            known_keys.insert(key_id, account.secret_key.as_public_key());
+            known_keys.insert(key_id, account.secret_key.to_public_key());
         }
 
         let unrelated = WalletAccount::deterministic(102, 0, true).expect("account");
-        let unrelated_key: Key = unrelated.secret_key.clone().into();
+        let unrelated_key: StoredKey = unrelated.secret_key.clone().into();
         let unrelated_key_id = key_id_for_preload_backend(&unrelated_key);
         kms_keys.insert(unrelated_key_id.clone(), unrelated_key);
         known_keys.insert(
             unrelated_key_id.clone(),
-            unrelated.secret_key.as_public_key(),
+            unrelated.secret_key.to_public_key(),
         );
 
-        let ed25519_key: Key = Ed25519Key::from_bytes(&[9; 32]).into();
+        let ed25519_key: StoredKey = UnsecuredEd25519Key::from_bytes(&[9; 32]).into();
         let ed25519_key_id = key_id_for_preload_backend(&ed25519_key);
         kms_keys.insert(ed25519_key_id.clone(), ed25519_key);
         let voucher_master = WalletAccount::deterministic(104, 0, true).expect("account");
-        let voucher_key: Key = voucher_master.secret_key.clone().into();
+        let voucher_key: StoredKey = voucher_master.secret_key.clone().into();
         let voucher_master_key_id = key_id_for_preload_backend(&voucher_key);
         kms_keys.insert(voucher_master_key_id.clone(), voucher_key);
         known_keys.insert(
             voucher_master_key_id.clone(),
-            voucher_master.secret_key.as_public_key(),
+            voucher_master.secret_key.to_public_key(),
         );
 
         remove_external_scenario_wallet_keys_from_maps(
@@ -153,10 +153,10 @@ mod scenario_wallet_key_tests {
         let mut kms_keys = HashMap::new();
         let mut known_keys = HashMap::new();
         let account = WalletAccount::deterministic(105, 0, true).expect("account");
-        let key: Key = account.secret_key.clone().into();
+        let key: StoredKey = account.secret_key.clone().into();
         let key_id = key_id_for_preload_backend(&key);
         kms_keys.insert(key_id.clone(), key);
-        known_keys.insert(key_id, account.secret_key.as_public_key());
+        known_keys.insert(key_id, account.secret_key.to_public_key());
         let before_kms = kms_keys.clone();
         let before_known_keys = known_keys.clone();
 
diff --git a/tests/src/cucumber/steps/nodes/steps/lifecycle.rs b/tests/src/cucumber/steps/nodes/steps/lifecycle.rs
index f1c2c3668..1e190f4e1 100644
--- a/tests/src/cucumber/steps/nodes/steps/lifecycle.rs
+++ b/tests/src/cucumber/steps/nodes/steps/lifecycle.rs
@@ -1,7 +1,7 @@
 use super::{
     AutoClaimSettings, AutoClaimTick, ClaimTarget, ConfigOverride, CucumberWorld, Duration,
-    GenesisTokens, HashMap, Instant, Key, ManualClusterKind, ManualClusterSpec,
-    NodesToStartUnordered, NonZeroU64, Step, StepError, StepResult, TARGET, WalletAccount,
+    GenesisTokens, HashMap, Instant, ManualClusterKind, ManualClusterSpec, NodesToStartUnordered,
+    NonZeroU64, Step, StepError, StepResult, StoredKey, TARGET, WalletAccount,
     assert_manual_node_has_peers, connect_manual_node_to_node,
     ensure_fee_sponsorship_and_fork_groups_are_not_mixed, given, install_local_manual_cluster,
     key_id_for_preload_backend, non_zero, parse_genesis_wallet_tokens_row,
@@ -231,7 +231,7 @@ fn step_configure_pow_auto_claim(
     // account has to be one the node holds: preload the secret into the KMS and
     // list its public key under `wallet.known_keys`, keyed the same way the
     // preload backend derives ids.
-    let key: Key = account.secret_key.clone().into();
+    let key: StoredKey = account.secret_key.clone().into();
     let key_id = key_id_for_preload_backend(&key);
     let key_value = serde_yaml::to_value(&key).map_err(|source| StepError::InvalidArgument {
         message: format!(
diff --git a/tests/src/cucumber/steps/nodes/steps/mod.rs b/tests/src/cucumber/steps/nodes/steps/mod.rs
index 072705788..fba58537a 100644
--- a/tests/src/cucumber/steps/nodes/steps/mod.rs
+++ b/tests/src/cucumber/steps/nodes/steps/mod.rs
@@ -9,7 +9,7 @@ use cucumber::{gherkin::Step, given, then, when};
 use lb_common_http_client::CommonHttpClient;
 use lb_config::kms::key_id_for_preload_backend;
 use lb_core::{codec::DeserializeOp as _, mantle::GenesisTime};
-use lb_key_management_system_service::keys::{Key, ZkPublicKey};
+use lb_key_management_system_service::keys::{StoredKey, ZkPublicKey};
 use lb_libp2p::{Multiaddr, PeerId};
 use lb_pow_service::{AutoClaimSettings, AutoClaimTick, ClaimTarget};
 use lb_testing_framework::{
diff --git a/tests/src/tests/mantle/sdp/ops.rs b/tests/src/tests/mantle/sdp/ops.rs
index fab4c9b8d..10ed96701 100644
--- a/tests/src/tests/mantle/sdp/ops.rs
+++ b/tests/src/tests/mantle/sdp/ops.rs
@@ -22,7 +22,7 @@ use lb_core::{
         WithdrawMessage,
     },
 };
-use lb_key_management_system_service::keys::{Ed25519Key, Ed25519Signature, ZkKey};
+use lb_key_management_system_service::keys::{Ed25519Key, Ed25519Signature, UnsecuredZkKey};
 use lb_node::config::{
     RunConfig, blend::deployment::MinimumNetworkSize, cryptarchia::deployment::EpochConfig,
 };
@@ -91,7 +91,7 @@ async fn sdp_ops_e2e() {
     );
 
     let provider_signing_key = Ed25519Key::from_bytes(&[7u8; 32]);
-    let provider_zk_key = ZkKey::from(BigUint::from(7u64));
+    let provider_zk_key = UnsecuredZkKey::from(BigUint::from(7u64));
     let provider_id = ProviderId::try_from(provider_signing_key.public_key().to_bytes())
         .expect("provider signing key should yield a provider id");
     let zk_id = provider_zk_key.to_public_key();
@@ -120,7 +120,7 @@ async fn sdp_ops_e2e() {
                     .sign_payload(tx_hash.as_signing_bytes().as_ref())
                     .to_bytes(),
             );
-            let zk_sig = ZkKey::multi_sign(
+            let zk_sig = UnsecuredZkKey::multi_sign(
                 &[spare_note_secret_key.clone(), provider_zk_key.clone()],
                 &tx_hash.to_fr(),
             )
@@ -166,7 +166,7 @@ async fn sdp_ops_e2e() {
         Op::SDPWithdraw(withdraw_message),
         |tx_hash| {
             OpProof::ZkSig(
-                ZkKey::multi_sign(
+                UnsecuredZkKey::multi_sign(
                     &[spare_note_secret_key.clone(), provider_zk_key.clone()],
                     &tx_hash.to_fr(),
                 )
@@ -321,7 +321,7 @@ async fn start_sdp_manual_cluster(
     NodeHttpClient,
     Vec<Utxo>,
     WalletAccount,
-    ZkKey,
+    UnsecuredZkKey,
     NoteId,
     u64,
 ) {
diff --git a/tests/testing_framework/src/framework/local/provisioning.rs b/tests/testing_framework/src/framework/local/provisioning.rs
index 342589b01..ca59498e7 100644
--- a/tests/testing_framework/src/framework/local/provisioning.rs
+++ b/tests/testing_framework/src/framework/local/provisioning.rs
@@ -12,7 +12,7 @@ use config::{api, sdp, state, storage, wallet};
 use flate2::read::GzDecoder;
 use lb_config::kms::key_id_for_preload_backend;
 use lb_core::mantle;
-use lb_key_management_system_service::keys::{Key, secured_key::SecuredKey as _};
+use lb_key_management_system_service::keys::StoredKey;
 use lb_libp2p::Multiaddr;
 use lb_node::{
     UserConfig,
@@ -726,7 +726,7 @@ fn build_run_config(config: Config, deployment_settings: &DeploymentSettings) ->
             declaration_id: config.sdp_config.declaration_id,
             wallet: sdp::serde::WalletConfig {
                 max_tx_fee: mantle::Value::MAX.into(),
-                funding_pk: config.consensus_config.funding_sk.as_public_key(),
+                funding_pk: config.consensus_config.funding_sk.to_public_key(),
             },
             active_message_tracker: ActiveMessageTrackerConfig {
                 status_check_interval_in_tip_changes: NonZeroU64::new(3).unwrap(),
@@ -735,21 +735,23 @@ fn build_run_config(config: Config, deployment_settings: &DeploymentSettings) ->
         wallet: {
             let known_keys: HashMap<_, _> = [
                 (
-                    key_id_for_preload_backend(&Key::Zk(config.consensus_config.known_key.clone())),
-                    config.consensus_config.known_key.as_public_key(),
+                    key_id_for_preload_backend(&StoredKey::Zk(
+                        config.consensus_config.known_key.clone(),
+                    )),
+                    config.consensus_config.known_key.to_public_key(),
                 ),
                 (
-                    key_id_for_preload_backend(&Key::Zk(
+                    key_id_for_preload_backend(&StoredKey::Zk(
                         config.consensus_config.funding_sk.clone(),
                     )),
-                    config.consensus_config.funding_sk.as_public_key(),
+                    config.consensus_config.funding_sk.to_public_key(),
                 ),
             ]
             .into_iter()
             .chain(config.consensus_config.other_keys.iter().map(|sk| {
                 (
                     key_id_for_preload_backend(&sk.clone().into()),
-                    sk.as_public_key(),
+                    sk.to_public_key(),
                 )
             }))
             .chain(
@@ -759,11 +761,11 @@ fn build_run_config(config: Config, deployment_settings: &DeploymentSettings) ->
                     .keys
                     .values()
                     .filter_map(|key| match key {
-                        Key::Zk(sk) => Some((
-                            key_id_for_preload_backend(&Key::Zk(sk.clone())),
-                            sk.as_public_key(),
+                        StoredKey::Zk(sk) => Some((
+                            key_id_for_preload_backend(&StoredKey::Zk(sk.clone())),
+                            sk.to_public_key(),
                         )),
-                        Key::Ed25519(_) => None,
+                        StoredKey::Ed25519(_) => None,
                     }),
             )
             .collect();
@@ -771,7 +773,7 @@ fn build_run_config(config: Config, deployment_settings: &DeploymentSettings) ->
             wallet::serde::Config {
                 known_keys,
                 ..wallet::serde::Config::with_required_values(wallet::serde::RequiredValues {
-                    voucher_master_key_id: key_id_for_preload_backend(&Key::Zk(
+                    voucher_master_key_id: key_id_for_preload_backend(&StoredKey::Zk(
                         config.consensus_config.known_key.clone(),
                     )),
                 })
diff --git a/tests/testing_framework/src/node/configs/dynamic.rs b/tests/testing_framework/src/node/configs/dynamic.rs
index ef60bb2c1..58b77bb05 100644
--- a/tests/testing_framework/src/node/configs/dynamic.rs
+++ b/tests/testing_framework/src/node/configs/dynamic.rs
@@ -1,6 +1,6 @@
 use lb_config::kms::key_id_for_preload_backend;
 use lb_core::mantle::GenesisTime;
-use lb_key_management_system_service::keys::Key;
+use lb_key_management_system_service::keys::StoredKey;
 use lb_libp2p::Multiaddr;
 use lb_node::config::KmsConfig;
 use thiserror::Error;
@@ -110,23 +110,25 @@ fn build_kms_config_for_node(
             keys: [
                 (
                     blend_conf.non_ephemeral_signing_key_id.clone(),
-                    Key::Ed25519(private_key.clone()),
+                    StoredKey::Ed25519(private_key.clone()),
                 ),
                 (
                     blend_conf.core.zk.secret_key_kms_id.clone(),
-                    Key::Zk(secret_zk_key.clone()),
+                    StoredKey::Zk(secret_zk_key.clone()),
                 ),
                 (
-                    key_id_for_preload_backend(&Key::Zk(consensus_config.blend_note.sk.clone())),
-                    Key::Zk(consensus_config.blend_note.sk.clone()),
+                    key_id_for_preload_backend(&StoredKey::Zk(
+                        consensus_config.blend_note.sk.clone(),
+                    )),
+                    StoredKey::Zk(consensus_config.blend_note.sk.clone()),
                 ),
                 (
-                    key_id_for_preload_backend(&Key::Zk(consensus_config.known_key.clone())),
-                    Key::Zk(consensus_config.known_key.clone()),
+                    key_id_for_preload_backend(&StoredKey::Zk(consensus_config.known_key.clone())),
+                    StoredKey::Zk(consensus_config.known_key.clone()),
                 ),
                 (
-                    key_id_for_preload_backend(&Key::Zk(consensus_config.funding_sk.clone())),
-                    Key::Zk(consensus_config.funding_sk.clone()),
+                    key_id_for_preload_backend(&StoredKey::Zk(consensus_config.funding_sk.clone())),
+                    StoredKey::Zk(consensus_config.funding_sk.clone()),
                 ),
             ]
             .into(),
diff --git a/tests/testing_framework/src/node/configs/postprocess.rs b/tests/testing_framework/src/node/configs/postprocess.rs
index eef071e02..8a5c68d33 100644
--- a/tests/testing_framework/src/node/configs/postprocess.rs
+++ b/tests/testing_framework/src/node/configs/postprocess.rs
@@ -6,7 +6,7 @@ use lb_core::{
     mantle::{GenesisTime, Note, ledger::Outputs},
     sdp::{Locator, ServiceType},
 };
-use lb_key_management_system_service::keys::{Key, ZkKey};
+use lb_key_management_system_service::keys::{StoredKey, UnsecuredZkKey};
 
 use super::{
     Config,
@@ -152,8 +152,8 @@ pub fn apply_wallet_genesis_overrides(
     general_configs: &mut [Config],
     genesis_block: &GenesisBlock,
     n_blend_core_nodes: usize,
-    wallet_accounts: &[(ZkKey, u64)],
-    key_id_for_preload_backend: impl Fn(&Key) -> String,
+    wallet_accounts: &[(UnsecuredZkKey, u64)],
+    key_id_for_preload_backend: impl Fn(&StoredKey) -> String,
     test_context: Option<&str>,
     sdp_funding_config: SdpFundingConfig,
     genesis_time: GenesisTime,
@@ -217,7 +217,7 @@ pub fn apply_wallet_genesis_overrides(
 
     for general in general_configs.iter_mut() {
         for (secret_key, _) in wallet_accounts {
-            let key = Key::Zk(secret_key.clone());
+            let key = StoredKey::Zk(secret_key.clone());
             let key_id = key_id_for_preload_backend(&key);
             general.kms_config.backend.keys.entry(key_id).or_insert(key);
         }
diff --git a/tests/testing_framework/src/node/configs/wallet.rs b/tests/testing_framework/src/node/configs/wallet.rs
index 864bd4733..756dd2548 100644
--- a/tests/testing_framework/src/node/configs/wallet.rs
+++ b/tests/testing_framework/src/node/configs/wallet.rs
@@ -2,7 +2,7 @@ use std::{collections::HashSet, num::NonZeroUsize};
 
 use hex::ToHex as _;
 use lb_core::codec::SerializeOp as _;
-use lb_key_management_system_service::keys::{ZkKey, ZkPublicKey};
+use lb_key_management_system_service::keys::{UnsecuredZkKey, ZkPublicKey};
 use num_bigint::BigUint;
 use rand::Rng as _;
 use thiserror::Error;
@@ -107,14 +107,14 @@ fn allocation_for(index: usize, base_allocation: u64, remainder: usize) -> u64 {
 #[derive(Clone, Debug, serde::Serialize, serde::Deserialize)]
 pub struct WalletAccount {
     pub label: String,
-    pub secret_key: ZkKey,
+    pub secret_key: UnsecuredZkKey,
     pub value: u64,
 }
 
 impl WalletAccount {
     pub fn new(
         label: String,
-        secret_key: ZkKey,
+        secret_key: UnsecuredZkKey,
         value: u64,
         allow_zero_value_genesis_tokens: bool,
     ) -> Result<Self, WalletConfigError> {
@@ -138,7 +138,7 @@ impl WalletAccount {
         seed[..2].copy_from_slice(b"wl");
         seed[2..10].copy_from_slice(&index.to_le_bytes());
 
-        let secret_key = ZkKey::from(BigUint::from_bytes_le(&seed));
+        let secret_key = UnsecuredZkKey::from(BigUint::from_bytes_le(&seed));
         Self::new(
             format!("wallet-user-{index}"),
             secret_key,
@@ -151,7 +151,7 @@ impl WalletAccount {
         let mut seed = [0u8; 32];
         rand::thread_rng().fill(&mut seed);
 
-        let secret_key = ZkKey::from(BigUint::from_bytes_le(&seed));
+        let secret_key = UnsecuredZkKey::from(BigUint::from_bytes_le(&seed));
         let index = u64::from_le_bytes(seed[..8].try_into().expect("seed has len 32"));
         Self::new(format!("wallet-r-user-{index}"), secret_key, 0, true)
     }
diff --git a/tests/testing_framework/src/workloads/transaction/workload.rs b/tests/testing_framework/src/workloads/transaction/workload.rs
index 1c01d087b..0ed7e2f7a 100644
--- a/tests/testing_framework/src/workloads/transaction/workload.rs
+++ b/tests/testing_framework/src/workloads/transaction/workload.rs
@@ -18,7 +18,7 @@ use lb_core::mantle::{
         GasPrices, MantleTxBuilder, OpProofs, states::Preverified, tx_list::ops::OpsGasContext,
     },
 };
-use lb_key_management_system_service::keys::{ZkKey, ZkPublicKey};
+use lb_key_management_system_service::keys::{UnsecuredZkKey, ZkPublicKey};
 use rand::{seq::SliceRandom as _, thread_rng};
 use testing_framework_core::scenario::{
     DynError, Expectation, RunContext, RunMetrics, Workload as ScenarioWorkload,
@@ -309,7 +309,7 @@ fn build_wallet_transaction(
         .build()
         .map_err(|err| format!("failed to build tx: {err}"))?;
 
-    let signature = ZkKey::multi_sign(
+    let signature = UnsecuredZkKey::multi_sign(
         slice::from_ref(&input.account.secret_key),
         &tx.hash().to_fr(),
     )
diff --git a/tools/config/src/blend.rs b/tools/config/src/blend.rs
index 358126896..834bf30ef 100644
--- a/tools/config/src/blend.rs
+++ b/tools/config/src/blend.rs
@@ -1,13 +1,13 @@
 use std::str::FromStr as _;
 
-use lb_key_management_system_service::keys::{Ed25519Key, ZkKey};
+use lb_key_management_system_service::keys::{UnsecuredEd25519Key, UnsecuredZkKey};
 use lb_libp2p::Multiaddr;
 use lb_node::config::blend::serde as blend;
 use num_bigint::BigUint;
 
 use crate::kms::key_id_for_preload_backend;
 
-pub type GeneralBlendConfig = (blend::Config, Ed25519Key, ZkKey);
+pub type GeneralBlendConfig = (blend::Config, UnsecuredEd25519Key, UnsecuredZkKey);
 
 const DEFAULT_BLEND_LISTENING_HOST: &str = "127.0.0.1";
 
@@ -25,9 +25,9 @@ pub fn create_blend_configs_with_listening_host(
     ids.iter()
         .zip(ports)
         .map(|(id, port)| {
-            let private_key = Ed25519Key::from_bytes(id);
+            let private_key = UnsecuredEd25519Key::from_bytes(id);
             let secret_zk_key =
-                ZkKey::from(BigUint::from_bytes_le(private_key.public_key().as_bytes()));
+                UnsecuredZkKey::from(BigUint::from_bytes_le(private_key.public_key().as_bytes()));
             let mut base_config = blend::Config::with_required_values(blend::RequiredValues {
                 non_ephemeral_signing_key_id: key_id_for_preload_backend(
                     &private_key.clone().into(),
diff --git a/tools/config/src/consensus.rs b/tools/config/src/consensus.rs
index 1bcd0d708..18ad7f30e 100644
--- a/tools/config/src/consensus.rs
+++ b/tools/config/src/consensus.rs
@@ -19,7 +19,7 @@ use lb_core::{
 };
 use lb_groth16::{AdditiveGroup as _, CompressedGroth16Proof, Fr};
 use lb_key_management_system_service::keys::{
-    Ed25519Key, Ed25519Signature, ZkKey, ZkPublicKey, ZkSignature,
+    Ed25519Signature, UnsecuredEd25519Key, UnsecuredZkKey, ZkPublicKey, ZkSignature,
 };
 use lb_node::{Hashable as _, SignedOps};
 use num_bigint::BigUint;
@@ -85,8 +85,8 @@ impl Default for SdpFundingConfig {
 #[derive(Clone)]
 pub struct ProviderInfo {
     pub service_type: ServiceType,
-    pub provider_sk: Ed25519Key,
-    pub zk_sk: ZkKey,
+    pub provider_sk: UnsecuredEd25519Key,
+    pub zk_sk: UnsecuredZkKey,
     pub locator: Locator,
     pub note: ServiceNote,
 }
@@ -107,25 +107,25 @@ impl ProviderInfo {
 /// be converted into a specific service or services configuration.
 #[derive(Clone, Debug)]
 pub struct GeneralConsensusConfig {
-    pub known_key: ZkKey,
+    pub known_key: UnsecuredZkKey,
     pub blend_note: ServiceNote,
-    pub funding_sk: ZkKey,
+    pub funding_sk: UnsecuredZkKey,
     pub funding_pk: ZkPublicKey,
-    pub other_keys: Vec<ZkKey>,
+    pub other_keys: Vec<UnsecuredZkKey>,
     pub prolonged_bootstrap_period: Duration,
 }
 
 #[derive(Clone, Debug)]
 pub struct ServiceNote {
     pub pk: ZkPublicKey,
-    pub sk: ZkKey,
+    pub sk: UnsecuredZkKey,
     pub note: Note,
     pub note_id: NoteId,
     pub output_index: usize,
 }
 
 pub struct BaseConsensusMaterial {
-    pub regular_note_keys: Vec<ZkKey>,
+    pub regular_note_keys: Vec<UnsecuredZkKey>,
     pub blend_notes: Vec<ServiceNote>,
     pub sdp_notes: Vec<ServiceNote>,
     pub utxos: Vec<Utxo>,
@@ -360,7 +360,7 @@ pub fn create_base_consensus_material_with_additional_wallet_outputs_and_sdp_fun
 
 fn create_utxos(
     ids: &[[u8; 32]],
-    regular_note_keys: &mut Vec<ZkKey>,
+    regular_note_keys: &mut Vec<UnsecuredZkKey>,
     blend_notes: &mut Vec<ServiceNote>,
     sdp_notes: &mut Vec<ServiceNote>,
     additional_wallet_outputs: usize,
@@ -395,7 +395,7 @@ fn create_utxos(
 
     for &id in ids {
         let sk_data = derive_key_material(LEADER_KEY_PREFIX, &id);
-        let sk = ZkKey::from(BigUint::from_bytes_le(&sk_data));
+        let sk = UnsecuredZkKey::from(BigUint::from_bytes_le(&sk_data));
         let pk = sk.to_public_key();
         regular_note_keys.push(sk);
         utxos.push(Utxo {
@@ -406,7 +406,7 @@ fn create_utxos(
         output_index += 1;
 
         let sk_blend_data = derive_key_material(BLEND_KEY_PREFIX, &id);
-        let sk_blend = ZkKey::from(BigUint::from_bytes_le(&sk_blend_data));
+        let sk_blend = UnsecuredZkKey::from(BigUint::from_bytes_le(&sk_blend_data));
         let pk_blend = sk_blend.to_public_key();
         let note_blend = Note::new(BLEND_NOTE_VALUE, pk_blend);
         let utxo = Utxo {
@@ -425,7 +425,7 @@ fn create_utxos(
         output_index += 1;
 
         let sk_sdp_data = derive_key_material(SDP_KEY_PREFIX, &id);
-        let sk_sdp = ZkKey::from(BigUint::from_bytes_le(&sk_sdp_data));
+        let sk_sdp = UnsecuredZkKey::from(BigUint::from_bytes_le(&sk_sdp_data));
         let pk_sdp = sk_sdp.to_public_key();
         for sdp_note_index in 0..sdp_notes_per_node {
             let note_value = base_sdp_note_value + u64::from(sdp_note_index < sdp_value_remainder);
@@ -490,9 +490,11 @@ pub fn create_genesis_block_with_declarations(
     ]);
 
     for provider in providers {
-        let zk_sig =
-            ZkKey::multi_sign(&[provider.note.sk, provider.zk_sk], &mantle_tx_hash.to_fr())
-                .unwrap();
+        let zk_sig = UnsecuredZkKey::multi_sign(
+            &[provider.note.sk, provider.zk_sk],
+            &mantle_tx_hash.to_fr(),
+        )
+        .unwrap();
         let ed25519_sig = provider
             .provider_sk
             .sign_payload(mantle_tx_hash.as_signing_bytes().as_ref());
diff --git a/tools/config/src/kms.rs b/tools/config/src/kms.rs
index 53d951094..27be4c465 100644
--- a/tools/config/src/kms.rs
+++ b/tools/config/src/kms.rs
@@ -1,17 +1,17 @@
 use lb_groth16::fr_to_bytes;
 use lb_key_management_system_service::{
     backend::preload::KeyId,
-    keys::{Key, secured_key::SecuredKey as _},
+    keys::{PublicKeyEncoding, StoredKey},
 };
 use lb_node::config::{KmsConfig, kms::serde::PreloadKmsBackendSettings};
 
 use crate::{blend::GeneralBlendConfig, consensus::GeneralConsensusConfig};
 
 #[must_use]
-pub fn key_id_for_preload_backend(key: &Key) -> KeyId {
-    let key_id_bytes = match key {
-        Key::Ed25519(ed25519_secret_key) => ed25519_secret_key.as_public_key().to_bytes(),
-        Key::Zk(zk_secret_key) => fr_to_bytes(zk_secret_key.as_public_key().as_fr()),
+pub fn key_id_for_preload_backend(key: &StoredKey) -> KeyId {
+    let key_id_bytes = match key.public_key() {
+        PublicKeyEncoding::Ed25519(public_key) => public_key.to_bytes(),
+        PublicKeyEncoding::Zk(public_key) => fr_to_bytes(public_key.as_fr()),
     };
     hex::encode(key_id_bytes)
 }
@@ -20,7 +20,7 @@ pub fn key_id_for_preload_backend(key: &Key) -> KeyId {
 pub fn create_kms_configs(
     blend_configs: &[GeneralBlendConfig],
     consensus_configs: &[GeneralConsensusConfig],
-    shared_keys: Option<&[Key]>,
+    shared_keys: Option<&[StoredKey]>,
 ) -> Vec<KmsConfig> {
     let mut kms_configs: Vec<KmsConfig> = blend_configs
         .iter()
@@ -30,25 +30,29 @@ pub fn create_kms_configs(
                 keys: [
                     (
                         blend_conf.non_ephemeral_signing_key_id.clone(),
-                        private_key.clone().into(),
+                        StoredKey::Ed25519(private_key.clone()),
                     ),
                     (
                         blend_conf.core.zk.secret_key_kms_id.clone(),
-                        zk_secret_key.clone().into(),
+                        StoredKey::Zk(zk_secret_key.clone()),
                     ),
                     (
-                        key_id_for_preload_backend(
-                            &consensus_configs[i].blend_note.sk.clone().into(),
-                        ),
-                        consensus_configs[i].blend_note.sk.clone().into(),
+                        key_id_for_preload_backend(&StoredKey::Zk(
+                            consensus_configs[i].blend_note.sk.clone(),
+                        )),
+                        StoredKey::Zk(consensus_configs[i].blend_note.sk.clone()),
                     ),
                     (
-                        key_id_for_preload_backend(&consensus_configs[i].known_key.clone().into()),
-                        consensus_configs[i].known_key.clone().into(),
+                        key_id_for_preload_backend(&StoredKey::Zk(
+                            consensus_configs[i].known_key.clone(),
+                        )),
+                        StoredKey::Zk(consensus_configs[i].known_key.clone()),
                     ),
                     (
-                        key_id_for_preload_backend(&consensus_configs[i].funding_sk.clone().into()),
-                        consensus_configs[i].funding_sk.clone().into(),
+                        key_id_for_preload_backend(&StoredKey::Zk(
+                            consensus_configs[i].funding_sk.clone(),
+                        )),
+                        StoredKey::Zk(consensus_configs[i].funding_sk.clone()),
                     ),
                 ]
                 .into(),
diff --git a/tools/config/src/node.rs b/tools/config/src/node.rs
index 5c0c3c061..ae5282411 100644
--- a/tools/config/src/node.rs
+++ b/tools/config/src/node.rs
@@ -1,9 +1,6 @@
 use std::{collections::HashMap, net::SocketAddr};
 
-use lb_key_management_system_service::{
-    backend::preload::KeyId,
-    keys::{Key, secured_key::SecuredKey as _},
-};
+use lb_key_management_system_service::{backend::preload::KeyId, keys::StoredKey};
 use lb_node::{
     UserConfig,
     config::{
@@ -35,7 +32,7 @@ pub fn create_node_user_config(config: GeneralConfig) -> UserConfig {
         .prolonged_bootstrap_period = config.consensus_config.prolonged_bootstrap_period;
 
     let mut sdp_config = SdpConfig::with_required_values(SdpConfigRequiredValues {
-        funding_pk: config.consensus_config.funding_sk.as_public_key(),
+        funding_pk: config.consensus_config.funding_sk.to_public_key(),
     });
     sdp_config.declaration_id = config.sdp_config.declaration_id;
 
@@ -73,36 +70,38 @@ fn create_axum_backend_settings(listen_address: SocketAddr) -> AxumBackendSettin
 
 fn create_wallet_config(
     consensus: &GeneralConsensusConfig,
-    kms_keys: &HashMap<KeyId, Key>,
+    kms_keys: &HashMap<KeyId, StoredKey>,
 ) -> WalletConfig {
     let known_keys = [
         (
-            key_id_for_preload_backend(&Key::Zk(consensus.known_key.clone())),
-            consensus.known_key.as_public_key(),
+            key_id_for_preload_backend(&StoredKey::Zk(consensus.known_key.clone())),
+            consensus.known_key.to_public_key(),
         ),
         (
-            key_id_for_preload_backend(&Key::Zk(consensus.funding_sk.clone())),
-            consensus.funding_sk.as_public_key(),
+            key_id_for_preload_backend(&StoredKey::Zk(consensus.funding_sk.clone())),
+            consensus.funding_sk.to_public_key(),
         ),
     ]
     .into_iter()
     .chain(consensus.other_keys.iter().map(|sk| {
         (
-            key_id_for_preload_backend(&Key::Zk(sk.clone())),
-            sk.as_public_key(),
+            key_id_for_preload_backend(&StoredKey::Zk(sk.clone())),
+            sk.to_public_key(),
         )
     }))
     .chain(kms_keys.values().filter_map(|key| match key {
-        Key::Zk(sk) => Some((
-            key_id_for_preload_backend(&Key::Zk(sk.clone())),
-            sk.as_public_key(),
+        StoredKey::Zk(sk) => Some((
+            key_id_for_preload_backend(&StoredKey::Zk(sk.clone())),
+            sk.to_public_key(),
         )),
-        Key::Ed25519(_) => None,
+        StoredKey::Ed25519(_) => None,
     }))
     .collect();
 
     let mut config = WalletConfig::with_required_values(WalletConfigRequiredValues {
-        voucher_master_key_id: key_id_for_preload_backend(&Key::Zk(consensus.known_key.clone())),
+        voucher_master_key_id: key_id_for_preload_backend(&StoredKey::Zk(
+            consensus.known_key.clone(),
+        )),
     });
     config.known_keys = known_keys;
     config
~~~~
