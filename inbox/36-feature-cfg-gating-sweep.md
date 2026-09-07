# Audit Report — Sweep: feature and cfg gating of test fixtures and default features

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/36`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `c3ff08e4a9cbc8344c58a18ef7992a41403c7f9c` — component(s): every workspace `Cargo.toml` (feature tables and `[dependencies]` feature requests), the resolved feature graph of `nodes/node/binary`, and the code behind `#[cfg(feature = …)]` / `#[cfg(test)]` gates in `kms/*`, `services/key-management-system`, `blend/*`, `core`, `libp2p`, `utils`, `tracing`, `nodes/node/binary`
Date: `2026-09-07` — author: `claude-fable-5-1` — status: `final`

---

## 1. Summary

- Overall assessment: the workspace is disciplined about `default-features = false` (all 198 workspace dependency entries) and about `#[cfg(test)]` on test helpers, but three features that read as test-only are on in every release build of the node: the KMS `unsafe` feature (which production code depends on, so it is not a test feature at all), tokio's `test-util`, and tokio's `tracing`/`full`; and the node binary itself declares no tokio features, relying on `overwatch` and a third-party crate's `tokio = { features = ["full"] }` for everything it uses.
- Findings: 0 critical · 0 high · 0 medium · 3 low · 3 informational
- Key themes: "unsafe"/"testing" features that are unconditionally enabled by production `[dependencies]`; tokio features obtained by unification rather than declaration; test doubles compiled into the release binary.
- Must-fix before launch: none. LB-001 should be resolved before the KMS hardening claims are relied on (see follow-up on #17).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `Cargo.toml` (workspace) and every member `Cargo.toml` (66 files) | `[features]` tables, feature requests in `[dependencies]` vs `[dev-dependencies]`, `default-features = false` coverage |
| `cargo tree -e features -p logos-blockchain-node` at the target commit | resolved feature set of the release binary (`cargo build -p logos-blockchain-node --release --locked` is the release command, `.github/workflows/prepare-release.yml:88`) |
| `kms/keys/src/keys/{mod.rs,ed25519/mod.rs,ed25519/private.rs,zk/mod.rs,zk/private.rs}`, `kms/operators/src/{ed25519,zk}/*` | what the `unsafe` feature gates |
| `services/key-management-system/src/{lib.rs,message.rs,backend/preload.rs}` | `unsafe` re-export, Debug bounds |
| `nodes/node/binary/{Cargo.toml,src/main.rs,src/config/mod.rs,src/config/kms/serde.rs,src/cli/keys.rs,src/cli/config/keystore.rs,src/generic_services/blend/mod.rs}` | `testing` feature, config Serialize/Debug, key printing, ungated mock |
| `services/blend/src/{edge/mod.rs,core/mod.rs}`, `services/wallet/src/lib.rs` | production callers of `unsafe`-gated operators |
| `blend/{membership,message,network,proofs,provers,scheduling}/src/**` | `unsafe-test-functions` gates and their enablers |
| `core/src/mantle/**` | `test-utils` / `unsafe-test-functions` gates, `mock.rs` |
| `libp2p/Cargo.toml`, `libp2p/src/behaviour/nat/{gateway_monitor.rs,address_mapper/mod.rs}` | tokio `test-util` |
| `utils/{Cargo.toml,src/tokio/mod.rs}`, `blend/network/Cargo.toml` | `tokio-task-names` / `tokio/tracing` |
| `tracing/Cargo.toml`, `tracing/src/{logging,tracing,metrics}/otlp.rs` | OTLP exporter transport features |
| `codec/src/{lib.rs,fixtures.rs}` and every `fixtures` / `mock` / `test_utils` module in the workspace | `#[cfg(test)]` gating |
| `.github/workflows/{prepare-release,code-check,end-to-end-integration-tests,nightly-cluster-fork-detector}.yml`, `Dockerfile`, `flake.nix`, `.cargo-deny.toml` | which feature sets are actually built and shipped |

**Out of scope**

`tests/`, `tests/testing_framework/`, `tools/`, `deployment/faucet`, `logos_sql`, `zone-sdk`, `wallet-http-client`, `c-bindings` are not part of the node binary; they were only checked for feature requests that could unify into the node (none do) and for the reqwest TLS question (Appendix B). The correctness of what the gated code does (blend proof generation, wallet voucher derivation, KMS operator semantics) belongs to #17, #14 and #23. Third-party crates assumed correct: `tokio` 1.52.3, `ed25519-dalek` 2.2.0, `reqwest` 0.12.28, `rs-merkle-tree` 0.1.0, `overwatch` @ `ae887f41`, `opentelemetry-otlp`. `cargo tree` was run with the lockfile at the target commit; the feature graph is what cargo resolves for the default feature set of `logos-blockchain-node`, which is what `prepare-release.yml` builds.

**Assumptions**

Release artefacts are produced by `cargo build -p logos-blockchain-node --release --locked` with no `--features` (`prepare-release.yml:88`); `Dockerfile` does not build from source (it installs a pre-built module) and `flake.nix` only passes `-p logos-blockchain-c` (`flake.nix:78`). Repo-level facts from #19 hold at this commit. One #19 item does not: a `.cargo-deny.toml` exists and CI runs `cargo-deny --all-features` (`code-check.yml:60`); see S-003.

## 3. Method

- Manual review of the in-scope paths, working through issue `#36` (sub-issue of `#24`), with `#19` as context.
- Feature resolution: `cargo tree -e features -p logos-blockchain-node` (cargo 1.98.1) at the target commit, then tracing each suspicious feature line back to the crate that requested it. Separate `cargo tree -e features -p <crate>` runs for `logos-sql`, `logos-blockchain-faucet`, `logos-blockchain-zone-sdk`.
- Static grep (`rg`) for every `cfg(feature = …)`, `cfg(test)`, `mod fixtures|mock|test_utils`, tokio API use per crate, and `tempfile|proptest|mockall|…` in non-dev dependency tables.
- Automated tooling beyond `cargo tree`: none. Dynamic testing: none. No node build was performed; LB-003's runtime error is derived from reqwest's source, not reproduced.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | KMS `unsafe` feature is unconditionally enabled by production dependencies; the node's `testing` feature is a no-op for it; `ZkKey`'s Debug prints the secret scalar in release | Data Exposure | Low | High | Open |
| LB-002 | tokio `test-util` is a non-dev dependency feature of `lb-libp2p`, so the paused-clock machinery ships in the release node | Configuration | Low | High | Open |
| LB-003 | OTLP exporters are built with a reqwest/tonic that has no TLS backend; an `https://` collector URL fails at runtime | Configuration | Low | High | Open |
| LB-004 | `tokio/tracing` and `tokio/full` are on in release via `blend/network` and `rs-merkle-tree`; the node binary declares no tokio features and gets `macros`/`rt-multi-thread` only from `overwatch` | Configuration | Informational | — | Open |
| LB-005 | `MockLeaderProofsGenerator`, which fabricates zero-byte "verified" blend proofs, is compiled into the release node binary | Configuration | Informational | — | Open |
| LB-006 | Test-double network and mempool backends are compiled unconditionally in release | Configuration | Informational | — | Open |

### LB-001 · KMS `unsafe` feature is unconditionally enabled by production dependencies; the node's `testing` feature is a no-op for it; `ZkKey`'s Debug prints the secret scalar in release

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Data Exposure |
| Target | `kms/operators/Cargo.toml:20`, `nodes/node/binary/Cargo.toml:34` and `:95`, `services/chain/chain-leader/Cargo.toml:24`, `services/blend/Cargo.toml:27`, `services/wallet/Cargo.toml:26`; `kms/keys/src/keys/zk/mod.rs:28,63-66,76-84`, `kms/keys/src/keys/zk/private.rs:20-22`, `kms/keys/src/keys/mod.rs:28`; `nodes/node/binary/src/cli/keys.rs:294-297` |
| Status | Open |

**Description**

`lb-key-management-system-keys` has a feature `unsafe` (`kms/keys/Cargo.toml:46`) that (a) adds `into_unsecured()` to the hardened `Ed25519Key`/`ZkKey` wrappers, (b) derives `serde::Serialize` on them and on `Key`, (c) switches their `Debug` from `<redacted>` to the inner value, and (d) in `kms/operators` exposes `ed25519::exfiltrate_secret_key::LeakSecretKeyOperator` and `zk::voucher::UnsafeVoucherOperator` (`kms/operators/src/ed25519/mod.rs:2-3`, `zk/mod.rs:2-3`). The node's `testing` feature is documented as the way to turn it on (`nodes/node/binary/Cargo.toml:95`: `testing = ["lb-key-management-system-service/unsafe"]`).

It is on in every build. `cargo tree -e features -p logos-blockchain-node` (default features) shows `logos-blockchain-key-management-system-keys feature "unsafe"` enabled, requested from plain `[dependencies]` tables:

```toml
# kms/operators/Cargo.toml:20  (non-dev, unconditional)
lb-key-management-system-keys = { features = ["unsafe"], workspace = true }
# nodes/node/binary/Cargo.toml:34
lb-key-management-system-service = { features = ["unsafe"], workspace = true }
# services/chain/chain-leader/Cargo.toml:24, services/blend/Cargo.toml:27, services/wallet/Cargo.toml:26 — same line
```

and production code depends on it: the blend core and edge services pull the raw Ed25519 secret out of the KMS with `LeakSecretKeyOperator` (`services/blend/src/edge/mod.rs:227-235`, `services/blend/src/core/mod.rs:378-386`), the wallet derives vouchers with `UnsafeVoucherOperator` (`services/wallet/src/lib.rs:1178-1184`), the keystore CLI calls `into_unsecured()` five times (`nodes/node/binary/src/cli/config/keystore.rs:98,113,124,134,143`), and the node's own config type `PreloadKmsBackendSettings { keys: HashMap<KeyId, Key> }` derives `Serialize` unconditionally (`nodes/node/binary/src/config/kms/serde.rs:12-16`), which only compiles because `Key: Serialize` under `unsafe`. So `unsafe` cannot be turned off; the `testing` feature's only remaining effect is `Serialize`/`Deserialize` on `RunConfig` (`nodes/node/binary/src/config/mod.rs:541-546`).

The concrete consequence for this checklist item is (c): in release, `ZkKey`'s `Debug` is

```rust
// kms/keys/src/keys/zk/mod.rs:78-82
#[cfg(feature = "unsafe")]
write!(f, "ZkKey({:?})", self.0)?;
```

and `self.0` is `SecretKey(Fr)` with `#[derive(… Debug …)]` (`kms/keys/src/keys/zk/private.rs:20-22`), so the field element is printed. `Key` (`keys/mod.rs:28`), `PreloadKMSBackendSettings` (`services/key-management-system/src/backend/preload.rs:31`) and the node's `config::kms::serde::Config` (`config/kms/serde.rs:6`) all derive `Debug` through it. `Ed25519Key` is not affected: its inner `SigningKey` Debug is redacted by `ed25519-dalek` 2.2.0 (`signing.rs:552-557`, `finish_non_exhaustive`).

One production print site exists: `logos-blockchain-node keys generate`, when the operator declines to persist, does `println!("Key: {key:?}")` (`nodes/node/binary/src/cli/keys.rs:297`). For a `Zk` key this writes the raw scalar to stdout; for an `Ed25519` key it writes `Ed25519Key(SigningKey { verifying_key: … , .. })`, i.e. the public half only. No `{:?}` of a config, settings, message or key was found in the node or services outside `#[cfg(test)]` (`rg` over `nodes`, `c-bindings`, `tools`, `services`); `KMSMessage` does not derive `Debug` (`services/key-management-system/src/message.rs:18`).

**Exploit scenario**

No remote path. The gap is defence-in-depth: any future `tracing::debug!(?settings)` or `{config:?}` in the node, or a panic message that formats a `Key`, writes ZK secret keys to logs, and the type system no longer prevents it because the "hardened" wrappers are compiled in their unhardened form. An operator running `keys generate --zk` and answering "no" at the prompt gets the secret on their terminal (arguably intended, but inconsistent with the Ed25519 output, which is unusable for the same purpose).

**Recommendation**

- *Short term*: keep `Debug` redacted regardless of `unsafe` (delete the `#[cfg(feature = "unsafe")]` arm at `zk/mod.rs:78` and `ed25519/mod.rs:88`); print the key explicitly in `cli/keys.rs:297` via a dedicated encoding rather than `Debug`. Rename `unsafe` to what it is (e.g. `secret-export`) and drop the `testing = [".../unsafe"]` alias so nobody believes it is off in release.
- *Long term*: move the three production consumers off raw-secret export (KMS operators for blend signing and x25519 derivation, a `VoucherOperator` inside the KMS, keystore serialisation through a dedicated export type), then make `unsafe` a genuinely test-only feature enabled from `[dev-dependencies]` only. Filed as a follow-up under #17.

**References**: #17, #35 (derive hygiene), #37 (log leaks).

### LB-002 · tokio `test-util` is a non-dev dependency feature of `lb-libp2p`, so the paused-clock machinery ships in the release node

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Configuration |
| Target | `libp2p/Cargo.toml:46`; consumers only in `libp2p/src/behaviour/nat/gateway_monitor.rs:140-179` and `libp2p/src/behaviour/nat/address_mapper/mod.rs:431-666` (both under `#[cfg(test)]`) |
| Status | Open |

**Description**

```toml
# libp2p/Cargo.toml:46  ([dependencies])
tokio = { features = ["macros", "test-util", "time"], workspace = true }
```

`test-util` is tokio's test-only feature: it compiles the mockable `Clock` (`tokio-1.52.3/src/time/clock.rs:30`, `cfg_test_util!`), `tokio::time::pause/advance/resume`, and `#[tokio::test(start_paused)]`. The only users in the workspace are inside `#[cfg(test)] mod tests` (`gateway_monitor.rs:140-141,170`; `address_mapper/mod.rs:431-432,538,643`). Because features unify, `cargo tree -e features -p logos-blockchain-node` shows `tokio feature "test-util"` under `logos-blockchain-libp2p`, so every release node carries it. This is the checklist's "feature that is off in release" item, and it does not hold for this feature.

**Exploit scenario**

None from the network. Impact is (1) `tokio::time::pause()` is callable from any code in the node's process and would freeze every timer (a footgun for future code, not a current bug); (2) the time driver takes the `test-util` code path with its paused-clock check on each timer tick. The other crates that use `#[tokio::test(start_paused = true)]` (`services/blend`, `services/time`) correctly request `test-util` from `[dev-dependencies]`.

**Recommendation**

- *Short term*: move `test-util` to `[dev-dependencies]` in `libp2p/Cargo.toml` (`tokio = { features = ["test-util"], workspace = true }`).
- *Long term*: add a CI check that `cargo tree -e features -p logos-blockchain-node` contains none of `test-util`, `unsafe`, `unsafe-test-functions`, `test-utils`, `testing*` (a one-line grep; see S-001).

**References**: tokio feature docs, `test-util`.

### LB-003 · OTLP exporters are built with a reqwest/tonic that has no TLS backend; an `https://` collector URL fails at runtime

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Configuration |
| Target | `tracing/Cargo.toml:27-35`; `tracing/src/logging/otlp.rs:50,66`, `tracing/src/tracing/otlp.rs:67,83`, `tracing/src/metrics/otlp.rs:58,74` |
| Status | Open |

**Description**

The workspace pins `reqwest = { default-features = false, version = "0.12" }` (`Cargo.toml:257`) and `lb-tracing` requests only transport features:

```toml
# tracing/Cargo.toml:27-35
opentelemetry-http = { features = ["reqwest"], workspace = true }
opentelemetry-otlp = { features = ["grpc-tonic", "http", "http-proto", "logs", "metrics", "reqwest-blocking-client"], workspace = true }
```

In the node's resolved graph the only reqwest features are `blocking` (twice, both from `opentelemetry-otlp`/`opentelemetry-http`), and tonic has `channel` and `codegen` only; no `rustls-tls`, `native-tls`, `default-tls`, `tonic/tls*`, `hyper-rustls` appear anywhere in `cargo tree -e features -p logos-blockchain-node`. The `rustls` lines that do appear come from `libp2p-tls`/QUIC. `lb-common-http-client` does request `reqwest/rustls-tls` (`nodes/node/http-client/Cargo.toml:26`), but it is not a dependency of the node binary, so nothing unifies TLS in.

Without any `__tls` feature reqwest's connector is the plain `Inner::Http(HttpConnector)` (`reqwest-0.12.28/src/connect.rs:170,510-511`), and hyper-util's `HttpConnector` rejects non-`http` schemes. The exporters pass the operator's URL straight through (`.with_endpoint(config.service.url.to_string())`, six sites above), so a `tracing.otlp.service.url: https://collector…` config compiles, is accepted by config parsing, and fails on the first export.

**Exploit scenario**

No attacker needed. An operator who points logs/metrics/traces at a TLS-only collector gets an opaque exporter error after startup, and the natural workaround is a plaintext `http://` endpoint, which sends node logs (peer addresses, block ids, key ids) unencrypted over the network. The checklist's "required features (e.g. `rustls`) enabled explicitly rather than by accident" item does not hold for this transport: nothing enables TLS at all.

**Recommendation**

- *Short term*: decide whether OTLP over TLS is supported. If yes, add `reqwest/rustls-tls` (and `opentelemetry-otlp/tls-roots` or `tonic/tls-ring` for gRPC) to `tracing/Cargo.toml`. If no, reject `https` URLs in config validation with a clear message.
- *Long term*: for every client library, declare the TLS feature next to the crate that constructs the client, not somewhere else in the graph (the faucet, `zone-sdk` and `logos_sql` currently get `reqwest/rustls-tls` only because `lb-common-http-client` is in their graph, see Appendix B).

**References**: #26 (deployment defaults), #37 (log volume and content).

### LB-004 · `tokio/tracing` and `tokio/full` are on in release via `blend/network` and `rs-merkle-tree`; the node binary declares no tokio features and gets `macros`/`rt-multi-thread` only from `overwatch`

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Configuration |
| Target | `blend/network/Cargo.toml:28`, `utils/Cargo.toml:38`; `blend/crypto/Cargo.toml:22` → `rs-merkle-tree-0.1.0/Cargo.toml:120-122`; `nodes/node/binary/Cargo.toml` (tokio line) and `src/main.rs:11` |
| Status | Open |

**Description**

Three unification effects were traced in the node graph:

1. `blend/network` requests `lb-utils = { features = ["tokio", "tokio-task-names"] }` from `[dependencies]` (`blend/network/Cargo.toml:28`); `tokio-task-names = ["tokio", "tokio/rt", "tokio/tracing"]` (`utils/Cargo.toml:38`). So `tokio/tracing` is on in every build, and the node's `tokio-console` feature (`nodes/node/binary/Cargo.toml:105-119`), which is documented as the opt-in for exactly this, is partly redundant. Without `--cfg tokio_unstable` the named-task path is not compiled (`utils/src/tokio/mod.rs:69-83`), so the only effect is the extra `tracing` instrumentation inside tokio.
2. `blend/crypto` depends on `rs-merkle-tree` with `memory_store` (`blend/crypto/Cargo.toml:22`); that crate hard-codes `tokio = { version = "1", features = ["full"] }` (`registry …/rs-merkle-tree-0.1.0/Cargo.toml:120-122`). `full` brings `fs`, `process`, `signal`, `io-std`, `parking_lot`, `net`, … into the node. No workspace crate uses `tokio::fs`, `tokio::signal`, `tokio::process` or `tokio::io::stdin` in non-test code (grep), so nothing currently depends on it, but any future use would compile only because of this third-party crate.
3. `nodes/node/binary/Cargo.toml` declares `tokio = { workspace = true }` with no features, yet `src/main.rs:11` is `#[tokio::main]` (needs `macros` + `rt-multi-thread`) and the crate uses `tokio::net` and `tokio::sync`. Those come from `overwatch`'s `tokio = { features = ["macros", "rt-multi-thread", "sync", "time"] }` (`overwatch/Cargo.toml:27` at `ae887f41`). Fifteen other crates are in the same position (`tokio = { workspace = true }` while using `select!`, `spawn_blocking`, `time`, `sync`; list in Appendix B.3). `c-bindings` is the only binary-ish crate that declares `rt-multi-thread` itself.

**Exploit scenario**

None. This is build hygiene: a dependency bump in `overwatch` or `rs-merkle-tree` that drops a tokio feature breaks the node's build in a crate far from the change, and `tokio/tracing` in release is a small, unrequested overhead.

**Recommendation**

- *Short term*: declare `tokio = { features = ["macros", "rt-multi-thread", "net", "sync", "time"] }` in `nodes/node/binary/Cargo.toml`; move `tokio-task-names` out of `blend/network`'s `[dependencies]` request (it belongs behind `lb-blend/tokio-task-names`, which already forwards it).
- *Long term*: each crate declares the tokio features it uses; consider replacing or vendoring `rs-merkle-tree`'s tokio dependency (it is only needed for the `memory_store`), or filing upstream to make it optional. Filed as a follow-up under #24.

**References**: `cargo tree -e features -p logos-blockchain-node`, lines `tokio feature "tracing"` (via `logos-blockchain-utils feature "tokio-task-names"`), `tokio feature "full"` (via `rs-merkle-tree v0.1.0`), `tokio feature "test-util"` (via `logos-blockchain-libp2p`).

### LB-005 · `MockLeaderProofsGenerator`, which fabricates zero-byte "verified" blend proofs, is compiled into the release node binary

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Configuration |
| Target | `nodes/node/binary/src/generic_services/blend/mod.rs:61-79` |
| Status | Open |

**Description**

```rust
// nodes/node/binary/src/generic_services/blend/mod.rs:61-79
#[derive(Clone)]
pub struct MockLeaderProofsGenerator;
#[async_trait]
impl LeaderProofsGenerator for MockLeaderProofsGenerator {
    …
    async fn get_next_proof(&mut self) -> Option<BlendLayerProof> {
        Some(BlendLayerProof {
            proof_of_quota: VerifiedProofOfQuota::from_bytes_unchecked([0; _]),
            proof_of_selection: VerifiedProofOfSelection::from_bytes_unchecked([0; _]),
            ephemeral_signing_key: UnsecuredEd25519Key::generate_with_chacha_rng(),
        })
    }
}
```

This is a copy of the test double in `services/blend/src/edge/tests/utils.rs:63-66`, but it lives in the production node crate with no `#[cfg(test)]` and no feature gate. It is `pub`, so the dead-code lint does not fire, and nothing in the repository references it (`rg MockLeaderProofsGenerator` finds only its definition and the separate test copy). The real `BlendCoreService`/`BlendEdgeService` type aliases use `RealLeaderAndPowProofsGenerator`.

**Exploit scenario**

None today: unreferenced, and LTO will drop it. The risk is a one-line edit swapping it into the `BlendEdgeService` alias (`mod.rs:82-91`) for local testing and shipping it, which would make the node emit all-zero proofs of quota/selection. That it type-checks in release at all is the gap.

**Recommendation**

- *Short term*: delete it (the test copy in `services/blend` already exists) or wrap it in `#[cfg(test)]`.
- *Long term*: keep `from_bytes_unchecked` constructors of `Verified*` proof types behind `unsafe-test-functions` (currently `pub` and unconditional: `blend/proofs/src/quota/mod.rs:189`, `selection/mod.rs:173`; relevant to #61).

### LB-006 · Test-double network and mempool backends are compiled unconditionally in release

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Configuration |
| Target | `services/network/src/backends/mod.rs:7` (`pub mod mock;`, 469 lines), `services/tx-service/src/network/adapters/mod.rs:2` (`pub mod mock;`), `core/src/mantle/mod.rs:7` (`pub mod mock;`, `MockTransaction`) |
| Status | Open |

**Description**

Three mock backends are declared as plain `pub mod` with no `#[cfg(test)]` or feature: the network service's `backends::mock::Mock` backend, the mempool's `MockAdapter`, and `core::mantle::mock::{MockTransaction, MockTxId}`. Their only users are `#[cfg(test)]` modules (e.g. `services/chain/chain-network/src/sync/orphan_handler.rs:505`) and the tests crate. They are not selectable from the node config (`rg -i mock nodes/node/binary/src` finds only LB-005), so they are dead weight in release rather than reachable behaviour. Every other test helper in the workspace is gated correctly (Appendix B.2).

**Recommendation**

- *Short term*: `#[cfg(any(test, feature = "unsafe-test-functions"))]` on the three `mod mock;` lines, with the feature enabled from the consuming crates' `[dev-dependencies]`, matching the existing pattern in `blend/*`.

## 5. Suggestions (non-security)

### S-001 · Add a release-graph feature guard to CI

`code-check.yml` builds and tests with `--all-features` (`:112`, `:171`) and `--features testing` (`:153`), which is right for coverage but means no job checks what the default graph contains. A step running `cargo tree -e features -p logos-blockchain-node --locked | grep -E 'feature "(test-util|unsafe|unsafe-test-functions|test-utils|testing)'` and failing on any match would have caught LB-001 and LB-002.

### S-002 · `tempfile` is a non-dev dependency of `c-bindings`

`c-bindings/Cargo.toml:31` lists `tempfile` under `[dependencies]`; its only uses are inside `#[cfg(test)]` modules (`c-bindings/src/api/lifecycle.rs:209`, `c-bindings/src/api/config.rs:367`). Move it to `[dev-dependencies]`. No other test-only crate (`proptest`, `mockall`, `serial_test`, `rstest`, `insta`, `criterion`, …) appears in a non-dev table of a node-graph crate.

### S-003 · Issue #19 is stale on `cargo-deny`

`.cargo-deny.toml` exists at this commit (advisories with nine ignores, bans, licenses) and CI runs `cargo-deny --all-features check` (`code-check.yml:60`) with `exclude-dev = true`. Whoever takes #19 / #25 should update the "No `deny.toml`" line and review the ignore list (it includes `hickory-proto` DNS advisories reachable through libp2p).

### S-004 · Codec fixtures are compiled into release by design

`BinaryEncode: CodecExamples` and `BinaryDecode: CodecExamples` (`codec/src/lib.rs:50,81`) make every codec type carry its golden vectors in all builds, which is why `mod fixtures;` is ungated in `codec`, `core`, `kms/keys`, `blend/message`, `blend/proofs`, `consensus/cryptarchia-engine`, `logos_sql`. This is deliberate and the fixtures checked contain only public data (e.g. `kms/keys/src/fixtures.rs` has public keys and signatures of `0x01` bytes). It costs binary size and means fixture construction code (with `unwrap`s) is in the release image; if that is unwanted, the supertrait could be moved behind `cfg(any(test, feature = "fixtures"))` at the cost of the "no codec without a fixture" compile-time guarantee.

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

## Appendix B — Checklist items verified

### B.1 Feature tables in the workspace and where each test-flavoured feature is requested

| Feature | Declared in | Gates | Requested from `[dependencies]` (non-dev) | Requested from `[dev-dependencies]` | In node release graph? |
|---|---|---|---|---|---|
| `unsafe` | `kms/keys`, `kms/operators`, `services/key-management-system` | `into_unsecured`, `Serialize`, unredacted `Debug`, `LeakSecretKeyOperator`, `UnsafeVoucherOperator` | `kms/operators:20`, `nodes/node/binary:34`, `services/chain/chain-leader:24`, `services/blend:27`, `services/wallet:26` | `services/key-management-system:29` | **Yes** (LB-001) |
| `testing` (node) | `nodes/node/binary:95`, `tools/config:17` | `RunConfig` serde; forwards `unsafe` (already on) | `tests:44`, `tests/testing_framework:27,37` (not node-graph crates) | — | No (only CI e2e builds pass `--features testing`) |
| `testing-disable-proposal-publish` | `nodes/node/binary`, `services/chain/chain-leader` | skips block publish (`chain-leader/src/lib.rs:774-776`) | — | — | No (e2e CI only, `end-to-end-integration-tests.yml:134`) |
| `unsafe-test-functions` | `core`, `blend/{core,membership,message,network,proofs,provers,scheduling}` | 19 `#[cfg(any(test, feature = "unsafe-test-functions"))]` sites (`from_bytes_unchecked` fixtures, test constructors) | — | `ledger:39`, `blend/scheduling:31-32`, `blend/network:35-37`, `blend/provers:35-37`, `services/blend:54` | No ✔ |
| `test-utils` (core) | `core` | `op.rs:4-6,180`, `op_proof.rs:142`, `signed_ops.rs:4,73,90` | — | `ledger:39`, `tests:18`, `wallet:31`, `zone-sdk:35` | No ✔ |
| tokio `test-util` | — | tokio paused clock | `libp2p:46` | — | **Yes** (LB-002) |
| `tokio-task-names` / `tokio/tracing` | `utils:38`, `blend/*`, services, node `tokio-console` | named tasks (only with `tokio_unstable`) | `blend/network:28` | — | **Yes** (LB-004) |
| `profiling`, `tokio-console`, `dhat-heap`, `jemalloc` | `nodes/node/binary`, `nodes/api-common`, `services/tracing` | pprof routes, console subscriber, allocators | — | — | No ✔ |
| `openapi`, `ntp`, `rocksdb-backend`, `time`, `serde`, `tokio` (lb-utils) | various | production features | node binary and services, explicitly | — | Yes, intended ✔ |

### B.2 `#[cfg(test)]` gating of helper modules (all `mod fixtures|mock|mocks|test_utils|test_helpers` declarations in the workspace)

| Gated correctly | Ungated, by design (codec fixtures, S-004) | Ungated test doubles |
|---|---|---|
| `services/blend/src/lib.rs:74-75` (`test_utils`), `services/blend/src/delivery/mod.rs:8-9`, `ledger/src/mantle/sdp/mod.rs:2-3`, `ledger/src/mantle/sdp/rewards/mod.rs:2-3`, `blend/provers/src/crypto/mod.rs:18-19`, `blend/provers/src/provers/mod.rs:21-22`, `libp2p/src/behaviour/nat/state_machine/transitions/mod.rs:1-2`, `blend/proofs/src/quota/mod.rs:26-27` (feature-gated), every `mod tests` | `codec/src/lib.rs:21`, `kms/keys/src/lib.rs:3`, `core/src/block/mod.rs:2`, `core/src/header/mod.rs:10`, `core/src/proofs/mod.rs:2`, `core/src/mantle/mod.rs:4`, `blend/message/src/lib.rs:8`, `blend/proofs/src/lib.rs:8`, `consensus/cryptarchia-engine/src/lib.rs:4`, `logos_sql/src/protocol/mod.rs:15` | `services/network/src/backends/mod.rs:7`, `services/tx-service/src/network/adapters/mod.rs:2`, `core/src/mantle/mod.rs:7`, `nodes/node/binary/src/generic_services/blend/mod.rs:62` (LB-005, LB-006) |

### B.3 `default-features = false` and explicit feature requests

- All 198 entries in `[workspace.dependencies]` set `default-features = false`; every member crate uses `workspace = true` (two multi-line entries, `libp2p/Cargo.toml:26-37` and `tracing/Cargo.toml:28-35`, also end in `workspace = true`). Item holds ✔.
- **tokio**: crates that use tokio APIs (`select!`, `spawn_blocking`, `sync`, `time`, `net`) with a bare `tokio = { workspace = true }`: `blend/network`, `blend/provers`, `blend/scheduling`, `consensus/cryptarchia-sync`, `nodes/node/binary`, `services/blend`, `services/chain/{broadcast-service,chain-leader,chain-network,chain-service}`, `services/sdp`, `services/storage`, `services/time`, `services/tx-service`, `tracing`, `utils` (optional). They compile because `overwatch` requests `macros, rt-multi-thread, sync, time` and `rs-merkle-tree` requests `full` (LB-004). Crates that declare what they use: `c-bindings`, `deployment/faucet`, `kms/*`, `services/{api,network,pow,tracing,wallet}`, `libp2p`, `logos_sql`, `zone-sdk`, `tools/*`, `tests/testing_framework`.
- **rustls / TLS**: node graph has `libp2p-tls` (via `libp2p/quic`, `libp2p/Cargo.toml:26-37`, explicit ✔) and `rustls {ring, std, tls12, logging}` from it; no HTTP-client TLS at all (LB-003). Outside the node graph, `logos-sql`, `logos-blockchain-faucet` and `logos-blockchain-zone-sdk` resolve `reqwest/rustls-tls` only through `lb-common-http-client` (`nodes/node/http-client/Cargo.toml:26`), which they do depend on directly (faucet `:16`, zone-sdk `:20`, logos_sql via `lb-zone-sdk` `:23`), so it is reliable but implicit; they use `reqwest` themselves only for `Url`/`StatusCode` types.
- **axum / hyper**: node requests `axum {http1, http2, json, query, tokio}`, `tower-http {cors, limit, timeout, trace}` explicitly (`nodes/node/binary/Cargo.toml:72-76`) ✔. `utoipa-swagger-ui {axum, vendored}` explicit ✔.
- **tokio runtime features for `sntpc`** (`services/time`): `ntp = ["dep:sntpc", "dep:thiserror", "tokio/net"]` explicit ✔.

### B.4 Build commands that define "release"

| Job | Command | Feature set |
|---|---|---|
| `prepare-release.yml:88` | `cargo build -p logos-blockchain-node --release --locked --target …` | default (graph analysed here) |
| `code-check.yml:112` | `cargo clippy --workspace --all-targets --all-features` | all |
| `code-check.yml:153` | `cargo build -p logos-blockchain-node --features testing` | default + `testing` |
| `code-check.yml:171` | `cargo nextest … --all-features --exclude logos-blockchain-tests` | all |
| `end-to-end-integration-tests.yml:108,134` | `--release -p logos-blockchain-node --features testing` / `testing-disable-proposal-publish` | e2e only |
| `Dockerfile` | installs a pre-built module; no `cargo` invocation | n/a |
| `flake.nix:78` | `cargoExtraArgs = "-p logos-blockchain-c"` | default |
