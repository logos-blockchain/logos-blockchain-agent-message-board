# Audit Report — KMS `unsafe` feature: what production depends on it, and what the gate does not cover

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/85`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `kms/keys`, `kms/operators`, `services/key-management-system`, callers in `services/blend`, `services/wallet`, `services/chain/chain-leader`, `nodes/node/binary` (CLI and config)
Specs: `https://github.com/logos-co/logos-lips` @ `accd97d9bfd94ce1d542c41d0f95582f278e57db` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `key-types-and-generation.md`, `common-cryptographic-components.md` (all in full); `bedrock-anonymous-leaders-reward.md` § "Voucher creation and inclusion" and § "Preventing Double Claims" by section
Date: `2026-09-10` — author: `Claude Fable 5.1 (agent)` — status: `final`

---

## 1. Summary

- Overall assessment: the `unsafe` feature of the KMS keys crate is on in every build of the node, the node binary does not compile without it, and the feature is not the boundary it looks like: the ZK secret scalar is readable and exported through paths the feature never gated, while the one thing the feature reliably changes in release is that `ZkKey`'s `Debug` prints the secret in clear.
- Findings: 0 critical · 0 high · 0 medium · 3 low · 0 informational
- Key themes: "a cargo feature cannot separate the CLI from the node inside one binary", "the hardened ZK wrapper has an ungated secret accessor", "Debug redaction depends on a feature that is always on"
- Must-fix before launch: none. LB-001 and LB-003 should be closed together (redact unconditionally, replace the feature by explicit export types) before the KMS hardening is relied on or documented as such.

This report answers #85 (spun out of #36's LB-001, PR #84) and re-verifies it at the current head. What is new is the per-use inventory the issue asked for, with a disposition for each use (LB-001), the observation that the ZK key is not protected by the feature at all (LB-002), and a reproduced, inconsistent `Debug` behaviour for the two key types (LB-003).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `kms/keys/Cargo.toml`, `kms/keys/src/keys/{mod.rs,secured_key.rs}`, `keys/ed25519/{mod.rs,private.rs,x25519.rs}`, `keys/zk/{mod.rs,private.rs}` | what the `unsafe` feature gates; secret accessors; `Debug`, `Serialize`, `Zeroize` on every secret-bearing type |
| `kms/operators/Cargo.toml`, `kms/operators/src/{ed25519,zk,blend}/*` | all five operators, gated and ungated, and what each sends out of the KMS |
| `services/key-management-system/src/{lib.rs,message.rs,backend/mod.rs,backend/preload.rs}` | service message loop, log lines, `Debug` bounds, settings type |
| `services/blend/src/{core/mod.rs,core/settings.rs,core/backends/libp2p/settings.rs,core/state.rs,edge/mod.rs,edge/settings.rs,edge/handlers.rs,edge/backends/libp2p/mod.rs}` | the two `LeakSecretKeyOperator` callers and everything done with the raw key afterwards |
| `services/wallet/src/lib.rs` (`derive_voucher_from_kms`, `generate_new_voucher_secret`, `build_reserved_leader_claim_tx`) | the `UnsafeVoucherOperator` caller |
| `services/chain/chain-leader/{Cargo.toml,src/kms.rs,src/leadership.rs}` | feature request; how `LeaderPrivate` leaves the KMS |
| `core/src/proofs/leader_proof.rs` (`LeaderPrivate`) | derives on the struct that carries the ZK secret out |
| `nodes/node/binary/{Cargo.toml,src/config/mod.rs,src/config/kms/serde.rs,src/cli/keys.rs,src/cli/config/{keystore.rs,init.rs,update.rs},src/cli/participate.rs}` | `testing` feature, `Key: Serialize` requirement, every `into_unsecured()` call and what it feeds |
| `c-bindings/src/api/keys.rs` | the FFI `add_key` path into `AddKeyArgs::new` |
| `tools/config/src/kms.rs` | only to confirm it uses public-key accessors |
| Third-party sources read: `ed25519-dalek 2.2.0` (`signing.rs` `Debug`/`Drop`), `ark-ff 0.5.0` (`Fp` `Debug`, `Zeroize`; `BigInt` `Debug`), `libp2p-identity 0.2.14` (`ed25519.rs` `try_from_bytes`), `overwatch @ ae887f41` (settings logging) | to state exactly what is printed and what is zeroed |

**Out of scope**

The correctness of the operations the KMS performs (PoQ, PoL and PoC proving, ZkSignature) belongs to #22 and #21. Storage-at-rest of the keystore file, its permissions, and the config layout that duplicates keys are #65 and #26; they are noted under Suggestions only where this review had to touch them. The `unsafe-test-functions` features of the blend crates and the other gating results of #36 were not re-checked. Third-party crates assumed correct: `ed25519-dalek`, `x25519-dalek`, `zeroize`, `ark-ff`, `libp2p-identity`, `serde_yaml`.

**Assumptions**

The host is not compromised and no other process reads the node's memory; the question is what the node hands to which of its own components, and what can reach a log, a file, or a terminal. The `logos-blockchain-node` release build is `cargo build -p logos-blockchain-node --release --locked` with default features, as in `.github/workflows/prepare-release.yml`.

## 3. Method

- Manual review of the in-scope paths, working through #85 and its four items, with #17 for context.
- Spec conformance against `key-types-and-generation.md` (which key does what, and that the NSK is the network-level identity) and `bedrock-anonymous-leaders-reward.md` § "Voucher creation and inclusion" (how a voucher secret is obtained).
- `cargo tree --locked -e features -p logos-blockchain-node -i logos-blockchain-key-management-system-keys` and the same with `-e features,no-dev`, at the target commit, to confirm the resolved feature set of the release binary (cargo 1.9x from the workspace toolchain).
- Dynamic testing: one integration test added locally to `kms/keys/tests/` that prints `{:?}` of a `Key::Zk` and a `Key::Ed25519`, run with and without `--features unsafe` (output quoted in LB-003). Nothing else was executed.
- Every `grep` below was run over the whole workspace excluding `target/`, and the hits inside `#[cfg(test)]` modules were discarded by checking the enclosing module.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | The `unsafe` feature is enabled by five plain `[dependencies]` entries and the node binary cannot be built without it; four production uses depend on what it gates | Configuration | Low | High | Open |
| LB-002 | `ZkKey::as_fr()` is an ungated secret accessor, and the ZK secret leaves the KMS through ungated operators; the gate covers only the Ed25519 export and serde/`Debug` | Data Exposure | Low | High | Open |
| LB-003 | With the feature on, `Debug` of a `ZkKey` prints the secret scalar in decimal while `Debug` of an `Ed25519Key` prints only the public key; the CLI's declined-generate path prints the former and loses the latter | Data Exposure | Low | High | Open |

### LB-001 · The `unsafe` feature is enabled by five plain `[dependencies]` entries and the node binary cannot be built without it; four production uses depend on what it gates

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Configuration |
| Target | `kms/keys/Cargo.toml:46`; requesters `kms/operators/Cargo.toml:20`, `services/blend/Cargo.toml:27`, `services/wallet/Cargo.toml:26`, `services/chain/chain-leader/Cargo.toml:25`, `nodes/node/binary/Cargo.toml:34` (and the `testing` alias at `:96`); uses listed in the table below |
| Status | Open (re-verification of #36 LB-001 at `a805329f8`) |

**Description**

`unsafe = []` in `kms/keys/Cargo.toml:46` gates four things: `into_unsecured()` on the two hardened wrappers (`kms/keys/src/keys/ed25519/mod.rs:67-72`, `zk/mod.rs:63-67`), `serde::Serialize` on `Key`, `Ed25519Key` and `ZkKey` (`keys/mod.rs:28`, `ed25519/mod.rs:31`, `zk/mod.rs:28`), the unredacted `Debug` branch of both wrappers (`ed25519/mod.rs:88-89`, `zk/mod.rs:78-79`), and, through `kms/operators`' own `unsafe` feature (`kms/operators/Cargo.toml:30`), the modules `ed25519::exfiltrate_secret_key` and `zk::voucher` (`kms/operators/src/ed25519/mod.rs:2-3`, `zk/mod.rs:2-3`).

At `a805329f8` the feature is requested from plain `[dependencies]` tables in five crates, all of which are in the release binary's dependency graph. `cargo tree -e features,no-dev -p logos-blockchain-node -i logos-blockchain-key-management-system-keys` shows five `feature "unsafe"` edges reaching the keys crate, via `logos-blockchain-key-management-system-service feature "unsafe"` and `…-operators feature "unsafe"`. The node's `testing` feature (`nodes/node/binary/Cargo.toml:96`, `testing = ["lb-key-management-system-service/unsafe"]`) is therefore a no-op for the KMS: the feature is on whether or not `testing` is.

The feature cannot simply be turned off, because production code compiles against what it gates. This is the inventory the issue asked for, with what each use actually needs from the raw key and whether an existing, ungated KMS operation already provides it:

| # | Use | Where | What the raw key is used for | Ungated alternative today? | Disposition |
|---|---|---|---|---|---|
| 1 | `LeakSecretKeyOperator` on the blend core NSK | `services/blend/src/core/mod.rs:379-392`; result stored in `RunningBlendConfig.non_ephemeral_signing_key` (`core/settings.rs:38`) | (a) `public_key()` for membership and node id (`core/mod.rs:398`); (b) `derive_x25519()` for the NEK on every epoch processor (`core/mod.rs:775`, `:1794`); (c) the raw 32 bytes into `libp2p::identity::Keypair::ed25519_from_bytes` for the blend swarm identity (`core/backends/libp2p/settings.rs:32-37`) | (a) yes, `KMSMessage::PublicKey`; (b) yes, `DeriveX25519Operator` (`kms/operators/src/ed25519/derive_x25519.rs`, ungated); (c) no | (a) and (b) can move to the ungated API now. (c) cannot become an operator: `key-types-and-generation.md` § "Non-ephemeral Signing Key" makes the NSK the network-level identity, and rust-libp2p's Noise handshake needs the `Keypair` in-process. What can change is the type: a dedicated operator that hands out a `libp2p_identity::Keypair` (or a `Zeroizing<[u8; 32]>`) to the blend crate only, instead of the general-purpose `UnsecuredEd25519Key` kept for the life of the service (see S-005) |
| 2 | `LeakSecretKeyOperator` on the blend edge NSK | `services/blend/src/edge/mod.rs:223-235`; stored in `RunningSettings`/`RunningBlendConfig` (`edge/settings.rs:34`) | `public_key()` (`edge/mod.rs:238`, `:244`); `to_bytes()` into `Keypair::ed25519_from_bytes` (`edge/backends/libp2p/mod.rs:49-50`) | public key yes; keypair no | Same as #1 |
| 3 | `UnsafeVoucherOperator` on the wallet's voucher master ZK key | `services/wallet/src/lib.rs:1257-1279` (`derive_voucher_from_kms`), called from `generate_new_voucher_secret` (`:1231-1249`) and `build_reserved_leader_claim_tx` (`:1008-1027`) | The voucher secret `zkhash(master_sk, index)` (`kms/operators/src/zk/voucher.rs:37`) is used for `VoucherCm::from_secret` and `VoucherNullifier::from_secret` (`lib.rs:1017`, `:1182`, `:1247-1248`) and as the witness of the proof of claim in `generate_poc` (`lib.rs:1026`) | Commitment and nullifier: yes, computable inside the operator (the TODO at `voucher.rs:13-14` cites a `kms-keys <> core` cycle that no longer exists: `kms/operators` already depends on `lb-core`, `kms/operators/Cargo.toml:18`, and `zk/leader.rs:3-6` imports it). Proof of claim: no, unless proving moves inside the KMS the way `PoQOperator` already does it (`kms/operators/src/blend/poq.rs:57-77`, `spawn_blocking` inside `execute`) | Split into a `VoucherOperator` returning `(VoucherCm, VoucherNullifier)` and a `ProveClaimOperator` returning the proof. Both are ungated by construction |
| 4 | `into_unsecured()` in the CLI keystore | `nodes/node/binary/src/cli/config/keystore.rs:98`, `:113`, `:124`, `:134`, `:143`; `cli/keys.rs:127-128` | `init.rs:125-134` and `update.rs:106` need the raw NETWORK_SWARM bytes for `network.backend.swarm.node_key` (a libp2p `SecretKey` written into the config file); `init.rs:164`, `:188`, `:217`, `update.rs:139`, `:160`, `:181`, `participate.rs:43-45`, `:88` call only `to_public_key()`; `keys.rs:127-128` (`AddKeyArgs::new`, the FFI path from `c-bindings/src/api/keys.rs:155`) unwraps a `Key` only for `AddKeyArgs::key()` (`keys.rs:143-154`) to wrap it again; `keys.rs:239-246` unwraps the freshly generated key only to rewrap it | Public keys: yes, `ZkKey::to_public_key`, `Ed25519Key::public_key` are ungated. The rewrap round-trips: no raw access needed at all. The swarm key export: a raw export is inherent to a keystore tool | Drop the round-trips; keep one explicit, named export (`Ed25519Key::export_for_libp2p` or a `Keystore::export_swarm_key`) for the swarm key. That export lives in the same binary as the node, so no cargo feature can hide it; the honest gate is a type with a loud name, not `cfg` |
| 5 | `Key: Serialize` | `nodes/node/binary/src/config/kms/serde.rs:12-16` (`PreloadKmsBackendSettings` derives `Serialize`, and `UserConfig` at `config/mod.rs:57-71` derives it too); `cli/config/keystore.rs:60-64` (`Keystore { secret_keys: HashMap<KeyTitle, Key> }` derives `Serialize`) | Writing `user_config.yaml` and the keystore file (`cli/keys.rs:223-227`, `init.rs:55-58`, `migrate.rs:38`, `migrate_0_1_2/mod.rs:129`) | No: the hardened `Key` is the type stored in both files | Serialize a dedicated on-disk type (`StoredKey`/`KeyExport`, `#[serde(transparent)]` over `Zeroizing` bytes) that converts *into* `Key`, and never implement `Serialize` for `Key` |

Two requesters need nothing gated: `services/chain/chain-leader` (`Cargo.toml:25`) uses only `Ed25519Key`, `ZkKey` and the ungated leader operators (`src/kms.rs`, `src/leadership.rs`; its tests use `UnsecuredZkKey`, which is exported unconditionally from `kms/keys/src/keys/mod.rs:11-21`), and `kms/operators` requests `keys/unsafe` from `[dependencies]` (`Cargo.toml:20`) although only its own `unsafe`-gated module `exfiltrate_secret_key.rs:37` calls `into_unsecured()`.

**Exploit scenario**

Not directly exploitable. The impact is that every property the `unsafe` feature is meant to switch off in production is on: `Debug` of a `ZkKey` prints the scalar (LB-003), `Key`, `UserConfig` and `RunConfig` are `Serialize`, and the two exfiltrating operators are part of the release `KeyOperators` enum. Any future `{config:?}`, `serde_yaml::to_string(&config)` or `warn!(?settings)` added anywhere in the node writes the ZK secrets in clear; this review found no such site today (see LB-003 for what was searched).

**Recommendation**

- *Short term*: make `Debug` redaction unconditional (LB-003); move `kms/operators`' request for `keys/unsafe` into its own `unsafe` feature (`unsafe = ["lb-key-management-system-keys/unsafe"]`); drop the request from `chain-leader`; replace uses 1(a), 1(b), 2 (public key) and the two rewrap round-trips of use 4 by the ungated API; compute `VoucherCm`/`VoucherNullifier` inside the voucher operator.
- *Long term*: retire the feature. What remains after the short-term step is three genuine raw exports (libp2p identity in blend, swarm key in the CLI, proof-of-claim witness in the wallet) and one on-disk format. Give each an explicit type and function name (`LibP2pIdentity`, `KeyExport`, `ProveClaimOperator`) with `Zeroizing` bytes and no `Debug`, and let the compiler enforce that nothing else can reach the secret. Rename or delete `unsafe`: it is not `unsafe` in the Rust sense and, as shown, not test-only. Keep the node's `testing` feature for `RunConfig`'s serde derives (`config/mod.rs:541-547`), but remove the `lb-key-management-system-service/unsafe` entry from it.

**References**: #36 LB-001 (PR #84); `key-types-and-generation.md` § "Non-ephemeral Signing Key"; `.github/workflows/prepare-release.yml` (release build command).

### LB-002 · `ZkKey::as_fr()` is an ungated secret accessor, and the ZK secret leaves the KMS through ungated operators; the gate covers only the Ed25519 export and serde/`Debug`

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Data Exposure |
| Target | `kms/keys/src/keys/zk/mod.rs:42-45` (`as_fr`); `kms/operators/src/zk/leader.rs:111-134` (`BuildPrivateInputsWithLeaderKey`), `core/src/proofs/leader_proof.rs:281-285` (`LeaderPrivate`); `kms/keys/src/keys/zk/private.rs:89-99` |
| Status | Open |

**Description**

`ZkKey` is documented as "an hardened ZK secret key that only exposes methods to retrieve public information" (`zk/mod.rs:22-26`), and `into_unsecured()` is behind the feature. But `pub const fn as_fr(&self) -> &Fr` at `zk/mod.rs:42-45` returns the secret scalar unconditionally, and `Fr` is `Copy`. Anything holding a `&ZkKey` has the secret; the feature gate on `into_unsecured` changes nothing for ZK keys. The Ed25519 wrapper has no such accessor (its only ungated methods are `public_key`, `sign_payload`, `derive_x25519`, `ed25519/mod.rs:52-65`), so for Ed25519 the gate is real.

The accessor is what the operators use, and one of them exports the secret out of the KMS on the ungated path. `BuildPrivateInputsWithLeaderKey::execute` (`leader.rs:111-134`) builds `LeaderPrivate::new(…, *key.as_fr(), …)` and sends it over the reply channel; the chain-leader service receives it (`services/chain/chain-leader/src/kms.rs:79-108`) and runs the Groth16 prover on it in its own `spawn_blocking` (`leadership.rs:132-135`). `LeaderPrivate` derives `Debug` and `Clone` (`leader_proof.rs:281`) and wraps `lb_pol::PolWitnessInputsData`, which holds the scalar. So on every won slot the leader's ZK secret is copied into the chain-leader service, inside a printable, clonable, non-zeroizing struct. `PoQOperator` shows the alternative: it copies the scalar into `ProofOfCoreQuotaInputs` (`blend/poq.rs:58-64`) but proves inside `execute`, so nothing secret is sent back. `CheckLotteryWinning` (`leader.rs:49-67`) also keeps the secret inside.

Zeroization of ZK secrets is nominal for the same reason. `ZkKey` and `UnsecuredZkKey` derive `ZeroizeOnDrop` and `ark_ff::Fp` implements `Zeroize` (`ark-ff-0.5.0/src/fields/models/fp/mod.rs:984-990`), so the wrapper's own field is cleared; but every use copies the `Copy` scalar out first: into the `[Fr; 32]` stack buffer of `try_from_secret_keys` (`zk/private.rs:94-97`, not zeroized), into `ZkSignWitnessInputs`, into `LeaderPrivate`, into `ProofOfCoreQuotaInputs`, and `into_inner(self) -> Fr` (`private.rs:46-48`) returns a plain copy. None of those are zeroized.

**Exploit scenario**

Not exploitable remotely. The impact is on what the KMS boundary guarantees: for ZK keys it guarantees nothing that a plain `Arc<ZkKey>` would not, and the leader secret is present in the chain-leader service's memory (and in any `Debug` output of `LeaderPrivate`, of which there is none today: `grep -rn "private_inputs:?\|?private_inputs" services/ core/` is empty) every time the node proposes a block.

**Recommendation**

- *Short term*: make `as_fr` `pub(crate)` and give the operators a crate-private accessor; remove `Debug` and `Clone` from `LeaderPrivate` (or implement `Debug` by hand without the witness) and make it `ZeroizeOnDrop`.
- *Long term*: prove the leader proof inside the KMS as `PoQOperator` does, so `LeaderPrivate` never leaves `execute`; audit `lb_pol`, `lb_poc` and `lb_zksign` witness types for `Zeroize` and drop the `Copy` on any wrapper that carries a secret scalar (a `Zeroizing<Fr>` newtype for the secret would make each copy explicit). Relates to #35 (derive hygiene) and #64 (prover inputs).

**References**: `common-cryptographic-components.md` § "ZkSignature" (the secret key is the private witness and "must remain confidential"); #17 question "Does the service hand out raw key bytes to callers, or only perform operations?"

### LB-003 · With the feature on, `Debug` of a `ZkKey` prints the secret scalar in decimal while `Debug` of an `Ed25519Key` prints only the public key; the CLI's declined-generate path prints the former and loses the latter

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Data Exposure |
| Target | `kms/keys/src/keys/zk/mod.rs:76-86`, `ed25519/mod.rs:86-96`, `zk/private.rs:20` (`UnsecuredZkKey` derives `Debug`); `nodes/node/binary/src/cli/keys.rs:279-300` (`run_generate_key`) |
| Status | Open |

**Description**

Under `feature = "unsafe"` (always on, LB-001) both wrappers format their inner unsecured key. `UnsecuredZkKey` derives `Debug` (`zk/private.rs:20`), so it prints the `Fr`, which `ark-ff` renders as the decimal integer (`fp/mod.rs:154-158` → `BigInt` `Debug` → `BigUint`). `UnsecuredEd25519Key` also derives `Debug` (`ed25519/private.rs:16`), but `ed25519_dalek::SigningKey`'s `Debug` deliberately omits the secret (`ed25519-dalek-2.2.0/src/signing.rs:552-558`, `finish_non_exhaustive() // avoids printing secret_key`). Verified with a one-test crate added locally to `kms/keys/tests/`, run twice:

```
$ cargo test -p logos-blockchain-key-management-system-keys --test debug_repro -- --nocapture
REPRO unsafe=false zk=Zk(ZkKey(<redacted>))
REPRO unsafe=false ed=Ed25519(Ed25519Key(<redacted>))
$ cargo test -p logos-blockchain-key-management-system-keys --features unsafe --test debug_repro -- --nocapture
REPRO unsafe=true zk=Zk(ZkKey(SecretKey(7719472615821079694904732333912527190217998977709370935963838933860875309329)))
REPRO unsafe=true ed=Ed25519(Ed25519Key(UnsecuredEd25519Key(SigningKey { verifying_key: VerifyingKey(CompressedEdwardsY: [160, 154, …]), EdwardsPoint{ … }), .. })))
```

(The ZK key in the test is the scalar `0x11…11`; the Ed25519 key is `0x22…22` and does not appear in its own output.)

Where `Key: Debug` is reachable in the release node: `Key` derives `Debug` (`keys/mod.rs:27`) and so do `PreloadKMSBackendSettings` (`backend/preload.rs:31`), the node's `PreloadKmsBackendSettings`, `KmsConfig`, `UserConfig` and `RunConfig` (`config/kms/serde.rs:6`, `:12`, `config/mod.rs:57`, `:541`), and `KMSService` requires `Backend::Key: Debug` (`services/key-management-system/src/lib.rs:32`, `:43`, `:58`, `:112`). No production site prints them today: the KMS service logs only `key_id` and the error (`lib.rs:200-203`), `grep -rn '{config:?}\|?config\|{settings:?}\|?settings\|:#?}' nodes/ services/ c-bindings/` finds nothing outside tests, overwatch never logs settings, and the blend `sender_signing_key = ?…` fields (`blend/network/src/core/poq_verification.rs:76`, `:94`, `:112`) are the sender's public key.

One CLI path does print a key. `run_generate_key` (`cli/keys.rs:279-300`): when the user answers no to "Write key to keystore?", the command generates the key anyway and prints `Key: {key:?}` (`:297`). For a ZK key the terminal receives the secret as a decimal integer, which `--zk` cannot take back (`parse_hex_zk_key` at `config/mod.rs:564-570` wants little-endian hex). For an Ed25519 key the terminal receives the public key only, and since nothing was persisted the secret is gone. The path is therefore both a leak and a data loss, and behaves differently per key type. `AddKeyArgs` derives `Debug` too (`cli/keys.rs:85`) and holds an `Option<UnsecuredZkKey>`, so a future `{args:?}` would print a ZK secret passed on the command line; none exists today.

**Exploit scenario**

An operator runs `keys generate --type zk` on a shared terminal or under a session recorder, declines to write to the keystore, and the secret scalar is on screen and in the scrollback; or a future `debug!(?config)` at startup lands ZK secrets in the log at the level most deployments enable. For Ed25519, the same command silently discards the key it was asked to show.

**Recommendation**

- *Short term*: make both `Debug` impls print `<redacted>` unconditionally and delete the `#[cfg(feature = "unsafe")]` branches (`zk/mod.rs:78-82`, `ed25519/mod.rs:88-92`); implement `Debug` for `UnsecuredZkKey` by hand the way `ed25519-dalek` does, or drop the derive; replace `println!("Key: {key:?}")` with an explicit, documented encoding that is the same for both types and round-trips through `--zk`/`--ed25519` (hex of the 32 secret bytes, with the same WARNING string the keystore file carries), or remove the declined path and require `--yes`.
- *Long term*: a workspace lint or test that formats every `Key`, `Unsecured*Key`, `X25519PrivateKey`, `SharedKey` and `LeaderPrivate` with `{:?}` and asserts the secret bytes are absent (#35, #37 cover the sweep).

**References**: #36 LB-001 (the release-`Debug` observation); `ed25519-dalek` 2.2.0 `signing.rs:552-558`.

## 5. Suggestions (non-security)

### S-001 · Feature requests that do nothing, and an alias that documents the wrong thing

`services/chain/chain-leader/Cargo.toml:25` requests `lb-key-management-system-service/unsafe` and uses nothing gated. `kms/operators/Cargo.toml:20` requests `keys/unsafe` for all consumers when only its own gated module needs it. `nodes/node/binary/Cargo.toml:96` presents `testing` as the switch for `unsafe`, which is misleading since the feature is on regardless; `testing` still gates `RunConfig`'s serde derives (`config/mod.rs:541-547`) and is used by `tests/` and CI (`code-check.yml:153`, `end-to-end-integration-tests.yml:108`), so keep the feature and remove the `unsafe` entry from it.

### S-002 · The voucher operator's TODO is stale

`kms/operators/src/zk/voucher.rs:13-14` says the operator will be made secure "once resolving cyclic dep: kms-keys <> core". `kms/operators` has depended on `lb-core` since `45266a26f` (2026-02-03, `Cargo.toml:18`) and `zk/leader.rs` uses `lb_core::proofs::leader_proof`. Returning `(VoucherCm, VoucherNullifier)` from the operator is possible today; see LB-001 use 3.

### S-003 · Deterministic voucher secrets versus the spec's random draw

`bedrock-anonymous-leaders-reward.md` § "Voucher creation and inclusion" step 1 draws `voucher ←$ 𝔽p` at random. The node derives it as `zkhash(master_sk, index)` in hash mode with no domain-separation tag (`voucher.rs:37`), with `index` taken from wallet state (`services/wallet/src/states.rs:287-293`). Deterministic derivation is a reasonable choice for recovery, and unpredictable as long as `master_sk` is secret, so this is recorded as an observation rather than a deviation. Two things worth confirming in the wallet area (#14): that `next_new_voucher_index` is persisted and never rewinds (a reused index yields a duplicate `voucher_cm` and a shared nullifier, so only one of the two blocks' rewards can ever be claimed), and that the master key is never also used as a ZkSignature key (the public key there is `compress(KDF, sk)`, so there is no collision today, but a DST such as `b"VOUCHER_KDF"` would make that structural). The spec could say explicitly that a deterministic derivation from a dedicated master key is acceptable.

### S-004 · The keystore is written with default permissions, and every key is copied into the user config

`init` copies all keystore keys into `user_config.yaml` under `kms.backend.keys` (`cli/config/init.rs:198-206`, `build_kms_config`), so the config file, not only the keystore file, holds every secret; both are written with `std::fs::write` and no mode (`cli/keys.rs:223-227`, `init.rs:55-58`; `grep -rn "Permissions\|0o6\|umask" nodes/node/binary/src` is empty), so they get the process umask, typically `0644`. This is #65's item "KMS storage at rest" and is only noted here.

### S-005 · The blend services keep the raw NSK for the life of the process and clone it on every epoch

`RunningBlendConfig` (`services/blend/src/core/settings.rs:32-38`, `edge/settings.rs:30-34`) keeps the `UnsecuredEd25519Key` after start, although after start the core needs only the NEK (`derive_x25519()`, recomputed on every epoch at `core/mod.rs:775` and `:1794`) and the libp2p `Keypair` (`core/backends/libp2p/settings.rs:32-37`). The edge `run` loop clones its settings, key included, on every epoch and PoL event (`edge/mod.rs:396`, `:404`). Each copy is zeroized on drop (`UnsecuredEd25519Key` derives `ZeroizeOnDrop`; `SigningKey` zeroizes in `Drop`, `signing.rs:659-666`; `to_bytes()` returns `Zeroizing`, `private.rs:48-50`; `libp2p_identity::ed25519::Keypair::try_from_bytes` zeroes its input on success, `ed25519.rs:49-58`; the core's `*self.non_ephemeral_signing_key.as_bytes()` copy is likewise consumed by `ed25519_from_bytes(&mut …)`), and neither the recovery state (`core/state.rs:17-21`, `:585-594`, no key field) nor any log line receives it, so this is hygiene, not a leak. Derive the NEK and the `Keypair` once at start, keep those, and drop the raw key.

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

## Appendix B — Items of #85 and where they are answered

| Item | Where |
|---|---|
| Inventory every production use of `unsafe`-gated API and decide per use | LB-001 table (uses 1–5), S-001 for the two requesters that need nothing |
| Make `Debug` unconditionally redacted; fix `println!("Key: {key:?}")` | LB-003 |
| Rename the feature or move it to dev-dependencies; delete the `testing` alias | LB-001 recommendation, S-001 (keep `testing`, drop the alias entry) |
| Raw secret pulled out by blend: zeroized on drop, never cloned into logs or recovery state | S-005 (verified: yes, yes, yes); the ZK counterpart is LB-002 |
