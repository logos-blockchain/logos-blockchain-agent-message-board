# Audit Report — Sweep: declare tokio features per crate and gate test doubles instead of relying on feature unification

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/86`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): every workspace `Cargo.toml` that requests `tokio` or `lb-utils`, the resolved feature graph of `nodes/node/binary`, `libp2p`, `blend/network`, `utils`, `services/{network,tx-service,time}`, `core/src/mantle`, `nodes/node/binary/src/generic_services/blend`, `c-bindings`, `.github/workflows/code-check.yml`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md` (no specification covers build configuration; parent #24)
Date: `2026-09-11` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: every ⚑ item from #36 still holds at `a805329`; a fix set covering all seven checklist items was written, applied to a private copy and verified (`cargo tree` before/after, `cargo check --workspace --all-targets --all-features`, the new CI guard), and is reproduced in Appendix C for an upstream PR. The sweep also found two feature declarations that are wrong in the other direction (a feature that compiles `rt`-gated code without asking for `tokio/rt`) and established that `rs-merkle-tree`'s `tokio/full` is dead weight upstream, so the fix there is a one-line upstream PR rather than vendoring.
- Findings: 0 critical · 0 high · 0 medium · 1 low · 5 informational
- Key themes: tokio features obtained by unification instead of declaration (16 crates); test-only tokio features in release; test doubles compiled unconditionally; a third-party crate requesting `tokio/full` it never uses.
- Must-fix before launch: none. LB-001 (tokio `test-util` in the release node) and the CI guard (S-001) are the two items worth landing first, because they are the only ones that regress silently.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| Every `Cargo.toml` with a `tokio` or `lb-utils` request (44 `tokio` lines, listed in Appendix B.1) | which section (`[dependencies]` / `[dev-dependencies]` / feature table) and which features |
| `cargo tree -e features -p logos-blockchain-node --locked` at the target commit, before and after the fix set | the release feature graph; `prepare-release.yml:88` builds `-p logos-blockchain-node --release --locked` with default features |
| Non-test source of the 16 crates named in the issue plus `libp2p`, `utils`, `core`, `services/utils`, `services/network`: every `tokio::` path and `#[tokio::main|test]` attribute, classified as production or `#[cfg(test)]` | what each crate actually needs (Appendix B.2) |
| `tokio` 1.52.3 `Cargo.toml` and `src/lib.rs`, `src/task/mod.rs`, `src/sync/mod.rs`, `src/time/mod.rs` | which feature gates each API |
| `rs-merkle-tree` 0.1.0 (registry source) and upstream `master` `Cargo.toml` | whether tokio is used at all |
| `overwatch` @ `ae887f41` `overwatch/Cargo.toml` | what it supplies by unification |
| `services/network/src/backends/{mod,mock}.rs`, `services/tx-service/src/network/adapters/{mod,mock}.rs`, `services/tx-service/tests/mock.rs`, `core/src/mantle/mod.rs`, `services/chain/chain-network/src/sync/orphan_handler.rs`, `nodes/node/binary/src/generic_services/blend/mod.rs` | test doubles and their users |
| `c-bindings/Cargo.toml`, `c-bindings/src/api/{config,lifecycle}.rs` | `tempfile` |
| `.github/workflows/{code-check,nightly-cargo-hack,prepare-release}.yml`, `.github/actions/cargo-hack-check-cached/action.yml`, `scripts/check-workspace-dependency-policies.sh`, `.taplo.toml` | where a guard fits and what CI already checks |

**Out of scope**

Runtime behaviour of any crate; the correctness of the mocks themselves; the KMS `unsafe` feature (#36 LB-001, follow-up under #17) beyond noting its effect on the CI guard; `tests/`, `tests/testing_framework/`, `tools/`, `deployment/faucet`, `logos_sql`, `zone-sdk`, `wallet-http-client` (not in the node graph; their `tokio` lines are listed in Appendix B.1 for completeness only); `tokio-stream` and `tokio-util` feature requests. Third-party crates assumed correct: `tokio` 1.52.3, `overwatch`, `rs-merkle-tree` 0.1.0, `libp2p` 0.56, `axum` 0.7.9. The fix set was compiled with `cargo check`, not `cargo test`; no test was executed.

**Assumptions**

Release artefacts come from `cargo build -p logos-blockchain-node --release --locked` (`prepare-release.yml:88`). `[workspace.dependencies]` sets `default-features = false` on every entry and members may not override it (`scripts/check-workspace-dependency-policies.sh`), so a member crate's `default` feature is never enabled by another member; this is what makes the `lb-tx-service` `default` trick in Appendix C safe. Repo-level facts from #19 hold, with the #36 S-003 correction (`.cargo-deny.toml` exists, `code-check.yml:60`).

## 3. Method

- Manual review working through issue `#86` (sub-issue of `#24`), with `#19` and the #36 report (PR #84) as context. Every ⚑ item was re-verified at `a805329`; the source lines moved for `c-bindings/Cargo.toml` (31 → 33) and `libp2p/Cargo.toml` (46, unchanged).
- Feature resolution: `cargo tree -e features -p logos-blockchain-node --locked` (cargo 1.97.1) on a private copy of the checkout, before and after the fix set; for each tokio feature line, the nearest ancestor in the tree was extracted to list every requester (Appendix B.3).
- Static classification of tokio API use: a script walked each crate's `src/`, tracked `#[cfg(test)] mod … { … }` blocks by brace depth, excluded `lb_utils::tokio::*` and `libp2p::…::tokio::*` paths, and recorded every `tokio::` path and `#[tokio::main|test]` attribute as production or test. Each production path was mapped to its tokio feature from tokio's own `cfg_*!` gates (`tokio-1.52.3/src/lib.rs:507-567,644`, `src/task/mod.rs:277-281`, `src/sync/mod.rs:491`, `src/time/mod.rs:93-96`).
- Fix set (Appendix C) applied to the private copy; verified with `cargo tree` (after), the new `scripts/check-release-feature-graph.sh`, `cargo check -p logos-blockchain-node --locked`, `cargo check --workspace --all-targets --all-features --locked`, and `cargo check --all-targets --locked` on the six crates whose gating changed with their default features. Baseline `cargo check --workspace --all-targets --all-features --locked` at the unmodified commit passed first (3m21s), so a failure after the patch would be attributable to the patch.
- Automated tooling beyond `cargo tree`/`cargo check`: `taplo fmt` 0.10 on the touched manifests. `cargo-machete`, `cargo-hack` and `cargo-deny` were not run locally.
- Dynamic testing: none.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | tokio `test-util` is requested from `lb-libp2p`'s `[dependencies]` and ships in the release node (re-verified #36 LB-002); `lb-libp2p` also uses `net` and `sync` it never declares | Configuration | Low | High | Open — fixed in Appendix C |
| LB-002 | Sixteen crates use tokio APIs with a bare `tokio = { workspace = true }`; `tokio/tracing` and `tokio/full` are in release via `blend/network` and `rs-merkle-tree` (re-verified #36 LB-004) | Configuration | Informational | — | Open — fixed in Appendix C |
| LB-003 | `lb-utils`'s `tokio` feature and `lb-time-service`'s `ntp` feature compile `rt`- and `time`-gated code without requesting `tokio/rt` / `tokio/time` | Configuration | Informational | — | Open — fixed in Appendix C |
| LB-004 | `rs-merkle-tree` 0.1.0 requests `tokio = { features = ["full"] }` and contains no use of tokio | Patching / Supply chain | Informational | — | Open — upstream |
| LB-005 | `MockLeaderProofsGenerator` is compiled into the release node (re-verified #36 LB-005) | Configuration | Informational | — | Open — fixed in Appendix C |
| LB-006 | Three test doubles are `pub mod` with no gate; one of them is the only production reason `lb-tx-service` needs `tokio/rt` (re-verified #36 LB-006) | Configuration | Informational | — | Open — fixed in Appendix C |

### LB-001 · tokio `test-util` is requested from `lb-libp2p`'s `[dependencies]` and ships in the release node; `lb-libp2p` also uses `net` and `sync` it never declares

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Configuration |
| Target | `libp2p/Cargo.toml:46`; users of `test-util` only at `libp2p/src/behaviour/nat/gateway_monitor.rs:170,179` and `libp2p/src/behaviour/nat/address_mapper/mod.rs:538,554,643,666`, all inside `#[cfg(test)] mod tests` (`gateway_monitor.rs:140-141`, `address_mapper/mod.rs:431-432`) |
| Status | Open — fixed in Appendix C |

**Description**

Unchanged since #36:

```toml
# libp2p/Cargo.toml:46  ([dependencies])
tokio = { features = ["macros", "test-util", "time"], workspace = true }
```

`cargo tree -e features -p logos-blockchain-node --locked` at `a805329` contains exactly one `tokio feature "test-util"` line, whose only requester is `logos-blockchain-libp2p` (Appendix B.3). `test-util = ["rt", "sync", "time"]` in tokio (`tokio-1.52.3/Cargo.toml`), and this is where the checklist's "confirm it disappears" step bites: `lb-libp2p`'s production code uses `tokio::sync::mpsc` (`nat/inner.rs:21,101`, `nat/state_machine/mod.rs:4`), `tokio::sync::oneshot` (`chainsync/swarm_ext.rs:6`) and `tokio::net::UdpSocket` (`nat/address_mapper/protocols/nat_pmp.rs:5`, `pcp_core/connection.rs:8-11`), but declares neither `sync` nor `net`. Today `sync` arrives through `test-util` itself and `net` through `igd-next`, `natpmp` and `libp2p-quic`. Moving `test-util` to `[dev-dependencies]` alone leaves `lb-libp2p` compiling only by unification; the fix must also add `net` and `sync`. `macros` is used only in tests (`tokio::join!` at `nat/mod.rs:294`, inside `mod tests` from line 160-161; ten `#[tokio::test]`), so it moves to `[dev-dependencies]` too.

**Exploit scenario**

None from the network; as in #36 LB-002, the impact is that `tokio::time::pause()` is linkable from any production code path and the timer driver runs its paused-clock check. The new observation is that the fix as phrased in the issue ("move `test-util`") would have silently replaced one unification dependency with another.

**Recommendation**

- *Short term*: `libp2p/Cargo.toml`: `[dependencies] tokio = { features = ["net", "sync", "time"] }`, `[dev-dependencies] tokio = { features = ["macros", "rt", "test-util"] }` (Appendix C). After the change the node graph has no `tokio feature "test-util"` line (Appendix B.3, after).
- *Long term*: S-001.

**References**: #36 LB-002; tokio `test-util` feature definition.

### LB-002 · Sixteen crates use tokio APIs with a bare `tokio = { workspace = true }`; `tokio/tracing` and `tokio/full` are in release via `blend/network` and `rs-merkle-tree`

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Configuration |
| Target | `blend/network/Cargo.toml:28,30`; `utils/Cargo.toml:38`; `blend/crypto/Cargo.toml:22` → `rs-merkle-tree-0.1.0/Cargo.toml:120-122`; the bare `tokio` lines listed in Appendix B.2 (`nodes/node/binary/Cargo.toml:62`, `blend/provers:31`, `blend/scheduling:26`, `consensus/cryptarchia-sync:27`, `services/blend:49`, `services/chain/broadcast-service:22`, `services/chain/chain-leader:39`, `services/chain/chain-network:39`, `services/chain/chain-service:38`, `services/sdp:28`, `services/storage:29`, `services/time:29`, `services/tx-service:37`, `tracing:40`, `core:51` (dev), `services/utils:36` (dev)) |
| Status | Open — fixed in Appendix C |

**Description**

All three unification effects from #36 LB-004 hold at `a805329`:

1. `tokio feature "tracing"` has exactly one requester, `logos-blockchain-utils feature "tokio-task-names"`, which is requested from a `[dependencies]` table only by `blend/network/Cargo.toml:28` (`lb-utils = { features = ["tokio", "tokio-task-names"] }`). `blend/network` uses `lb_utils::tokio::task::spawn_blocking` (`core/poq_verification.rs:13`), which needs only `lb-utils/tokio`; the naming path is `#[cfg(all(feature = "tokio-task-names", tokio_unstable))]` (`utils/src/tokio/mod.rs:69,97,126`) and is what the node's `tokio-console` feature turns on (`nodes/node/binary/Cargo.toml:106-119`, via `lb-blend/tokio-task-names` → `lb-blend-provers`, `lb-blend-scheduling`; `blend/core/Cargo.toml:25`). Dropping it from `blend/network` removes `tokio feature "tracing"` from the default graph (verified, Appendix B.3) and it reappears with `--features tokio-console` because that feature lists `tokio/tracing` directly (`nodes/node/binary/Cargo.toml:119`).
2. `tokio feature "full"` has exactly one requester, `rs-merkle-tree v0.1.0` (LB-004). Through it the node compiles tokio's `fs`, `io-std`, `process`, `signal`, `parking_lot` and `rt-multi-thread`; the tree also shows `process` and `signal` pulling `signal-hook-registry`. Nothing in the workspace uses `tokio::fs`, `tokio::process`, `tokio::signal` or `tokio::io::stdin` outside the `tests` crate (Appendix B.2).
3. Sixteen crates use tokio APIs that are gated behind a feature they do not request. Appendix B.2 gives, per crate, the production paths found, the feature each needs, and the line to change. The features arrive from `overwatch` (`macros, rt-multi-thread, sync, time`; `overwatch/Cargo.toml:27`), `rs-merkle-tree` (`full`), `lb-libp2p` (`test-util` → `rt, sync, time`), and third parties (`axum/tokio` → `rt, net`; `libp2p-quic` → `rt, net, time`; `hyper-util` → `rt, time`; `tracing-gelf` → `io-util, net, time`; full list in Appendix B.3).

**Exploit scenario**

None. Build hygiene: a change in `overwatch`, `rs-merkle-tree` or `lb-libp2p` that drops a feature breaks an unrelated crate; `tokio/tracing` and `tokio/full` are unrequested code in the release image.

**Recommendation**

- *Short term*: the per-crate declarations in Appendix C. They are derived from Appendix B.2, so a reviewer can check each line against the paths listed there. Two of them deserve a note: `blend/network` has no production use of tokio at all (its only non-`lb_utils` tokio paths are inside `#[cfg(test)] mod tests`, `conn_maintenance.rs:119-120,187-205`, and the `tests/` directories), so its `tokio` entry moves to `[dev-dependencies]` outright; and `services/chain/chain-service` tests use `#[tokio::test(flavor = "multi_thread")]`, so its dev entry requests `rt-multi-thread`.
- *Long term*: S-001 catches the two named tokio features regressing; there is no cheap check for "feature obtained by unification" short of `cargo hack check --each-feature -p <crate>` with a patched `overwatch`, which is out of proportion for this.

**References**: #36 LB-004; Appendix B.2, B.3.

### LB-003 · `lb-utils`'s `tokio` feature and `lb-time-service`'s `ntp` feature compile `rt`- and `time`-gated code without requesting `tokio/rt` / `tokio/time`

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Configuration |
| Target | `utils/Cargo.toml:29,37-38`, `utils/src/lib.rs:14-15`, `utils/src/tokio/mod.rs:9-12,80,108,137`; `services/time/Cargo.toml:29,37`, `services/time/src/backends/mod.rs:2-3`, `services/time/src/backends/ntp/mod.rs:16`, `services/time/src/backends/ntp/async_client.rs:10-13` |
| Status | Open — fixed in Appendix C |

**Description**

Two feature definitions are the inverse of the issue's pattern: they are the crate's own opt-in, yet they still rely on the rest of the graph.

```toml
# utils/Cargo.toml:37-38
tokio            = ["dep:futures", "dep:tokio"]
tokio-task-names = ["tokio", "tokio/rt", "tokio/tracing"]
```

`#[cfg(feature = "tokio")] pub mod tokio;` (`utils/src/lib.rs:14-15`) compiles `use tokio::{runtime::Handle, task::{JoinError, JoinHandle}}` (`mod.rs:9-12`), `tokio::spawn` (`:80`), `tokio::task::spawn_blocking` (`:108`) and `Handle::spawn` (`:137`). All of these are behind `cfg_rt!` (`tokio-1.52.3/src/lib.rs:530-531,561-562`, `src/task/mod.rs:277-281`). Only `tokio-task-names` asks for `tokio/rt`; the plain `tokio` feature, which ten crates request (`blend/network:28`, `blend/provers:28`, `services/network:22`, `services/storage:24`, `services/chain/chain-leader:33`, `services/chain/chain-network:32`, `services/tx-service:32`, and others), does not. It compiles because `lb-utils` itself depends on `overwatch` (`utils/Cargo.toml:22`), which requests `rt-multi-thread`.

```toml
# services/time/Cargo.toml:37
ntp = ["dep:sntpc", "dep:thiserror", "tokio/net"]
```

The `ntp` module (`backends/mod.rs:2-3`) uses `tokio::time::{MissedTickBehavior, interval}` (`ntp/mod.rs:16`) and `tokio::time::{error::Elapsed, timeout}` (`async_client.rs:10-13`) but the feature adds only `tokio/net`. Note that `services/time`'s `[dependencies]` tokio line is bare as well (`:29`), so `time` comes from `overwatch`.

**Exploit scenario**

None; same class as LB-002, but in a place where the declaration is explicit and therefore looks complete.

**Recommendation**

- *Short term*: `tokio = ["dep:futures", "dep:tokio", "tokio/rt"]` in `utils/Cargo.toml`; `ntp = [..., "tokio/net", "tokio/time"]` in `services/time/Cargo.toml` (Appendix C).

### LB-004 · `rs-merkle-tree` 0.1.0 requests `tokio = { features = ["full"] }` and contains no use of tokio

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Patching / Supply chain |
| Target | `blend/crypto/Cargo.toml:22` (`rs-merkle-tree = { features = ["memory_store"], workspace = true }`), `Cargo.toml:261` (`rs-merkle-tree = { default-features = false, version = "0.1" }`); registry `rs-merkle-tree-0.1.0/Cargo.toml:120-122`; users `blend/crypto/src/merkle.rs:7,33,57,111` |
| Status | Open — upstream |

**Description**

The issue asks for a decision between an upstream PR, vendoring `memory_store`, or accepting the dependency. The facts that settle it:

- `rg -n tokio` over `rs-merkle-tree-0.1.0/src/` (`errors.rs`, `hasher.rs`, `lib.rs`, `node.rs`, `tree.rs`, `stores/{memory_store,rocksdb_store,sled_store,sqlite_store,store}.rs`) returns nothing. The crate's `[dependencies.tokio] features = ["full"]` is unused by the crate's own code; `criterion = { features = ["async_tokio"] }` in `[dev-dependencies]` is the only consumer of a tokio runtime, and it is dev-only.
- 0.1.0 is the only published version (crates.io). Upstream `master` at `github.com/bilinearlabs/rs-merkle-tree` still has `tokio = { version = "1", features = ["full"] }` under `[dependencies]` (line 49) alongside a new `file_store` feature, so no unreleased fix exists.
- `blend/crypto` uses `MerkleTree<InnerTreeZkHasher, MemoryStore, CORE_MERKLE_TREE_HEIGHT>`, `Node`, `MerkleProof` and the `Hasher` trait (`merkle.rs:7,33,57,111`), i.e. the in-memory store only.

`tokio feature "full"` in the node graph has this single requester (Appendix B.3), and it is the only source of `fs`, `io-std`, `process`, `signal` and `parking_lot`, and one of three sources of `rt-multi-thread`.

**Exploit scenario**

None. It enlarges the release image with tokio subsystems the node does not use (`process`/`signal` link `signal-hook-registry`), and it makes any accidental future use of `tokio::fs`/`tokio::process` compile.

**Recommendation**

- *Short term*: upstream PR deleting the `[dependencies.tokio]` entry (the crate compiles without it; the benches keep `criterion/async_tokio`). Until it is released, the node can carry `[patch.crates-io] rs-merkle-tree = { git = "<fork>", rev = "<sha>" }`; `flake.nix` and `Cargo.lock` need the rev too. Vendoring `memory_store` (about 100 lines plus the `MerkleTree`/`MerkleProof` types) is the fallback if upstream is unresponsive; accepting the dependency as-is is not recommended because it defeats the CI guard's `tokio full` check (S-001).
- *Long term*: none beyond S-001.

**References**: `https://github.com/bilinearlabs/rs-merkle-tree` (`master` `Cargo.toml:49` at the time of writing); #36 LB-004 item 2.

### LB-005 · `MockLeaderProofsGenerator` is compiled into the release node

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Configuration |
| Target | `nodes/node/binary/src/generic_services/blend/mod.rs:61-80`; its imports `:1,3-4,6-7,12` |
| Status | Open — fixed in Appendix C |

**Description**

Unchanged since #36 LB-005: a `pub struct MockLeaderProofsGenerator` whose `get_next_proof` returns `VerifiedProofOfQuota::from_bytes_unchecked([0; _])` and `VerifiedProofOfSelection::from_bytes_unchecked([0; _])` with a fresh ephemeral key, ungated, in the production node crate. `rg MockLeaderProofsGenerator` at `a805329` finds only this definition and the separate test copy in `services/blend/src/edge/tests/utils.rs:63-66` (used by `edge/tests/mod.rs`). Nothing references the node copy; `BlendEdgeService` uses `RealLeaderAndPowProofsGenerator` (`mod.rs:82-91`).

Deleting it removes nine imports that exist only for it (`axum::async_trait`, `Ed25519SecretKeyExt`, `VerifiedProofOfQuota`, `VerifiedProofOfSelection`, `BlendLayerProof`, `ProofsGeneratorSettings`, `WinningPolInfoStream`, `LeaderProofsGenerator`, `UnsecuredEd25519Key`); the crate still compiles with `CARGO_BUILD_WARNINGS=deny`-equivalent settings (no unused-import warning in the `cargo check` run, Appendix B.4). `from_bytes_unchecked` on both `Verified*` types remains `pub` and unconditional (`blend/proofs/src/quota/mod.rs:188-189`, `selection/mod.rs:172-173`), as noted in #36; that is #61's territory.

**Recommendation**

- *Short term*: delete the struct and its impl, plus the now-unused imports (Appendix C).

### LB-006 · Three test doubles are `pub mod` with no gate; one of them is the only production reason `lb-tx-service` needs `tokio/rt`

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Configuration |
| Target | `services/network/src/backends/mod.rs:7` (`pub mod mock;`), `services/tx-service/src/network/adapters/mod.rs:2` (`pub mod mock;`), `core/src/mantle/mod.rs:7` (`pub mod mock;`); users `services/tx-service/src/network/adapters/mock.rs:2,6,26-27,39,65,79`, `services/tx-service/tests/mock.rs:17,23,43`, `services/chain/chain-network/src/sync/orphan_handler.rs:505` |
| Status | Open — fixed in Appendix C |

**Description**

Unchanged since #36 LB-006. The dependency chain between the three is what shapes the gate: `lb-tx-service::network::adapters::mock::MockAdapter` is built on `lb-network-service::backends::mock::{Mock, MockBackendMessage, …}` and `lb-core::mantle::mock::{MockTransaction, MockTxId}` (`adapters/mock.rs:2,6`). Its users are `lb-tx-service`'s own integration test `tests/mock.rs` (which also uses the core and network mocks directly, `:17,23,43`) and `chain-network`'s `#[cfg(test)]` module (`orphan_handler.rs:505`, network mock only). Nothing else in the workspace, including the `tests` crate, references any of them.

Two of the mocks have their own tokio needs, which today are met by unification: `adapters/mock.rs:39` calls `tokio::spawn` (`rt`; the only `rt`-gated call in `lb-tx-service`'s non-test code), and `backends/mock.rs:188,211` call `tokio::time::sleep` (`time`; `lb-network-service` declares only `macros, sync`).

A plain `#[cfg(test)]` is not enough: an integration test compiles the library without `cfg(test)`. `lb-tx-service` already solved the same problem for `rocksdb-backend` with a default-on feature that the node never receives because `[workspace.dependencies]` sets `default-features = false` and members may not override it (`services/tx-service/Cargo.toml:44-52`, comment). The fix follows that precedent.

**Recommendation**

- *Short term* (Appendix C): `#[cfg(any(test, feature = "unsafe-test-functions"))]` on the three `mod mock;` lines; `lb-network-service` gains `unsafe-test-functions = ["tokio/time"]`; `lb-tx-service` gains `unsafe-test-functions = ["lb-core/unsafe-test-functions", "lb-network-service/unsafe-test-functions", "tokio/rt"]`, added to its `default` list next to `rocksdb-backend`; `chain-network` requests `lb-network-service/unsafe-test-functions` from `[dev-dependencies]`. `lb-core` already has the feature. After the change the node graph contains no `unsafe-test-functions` line (Appendix B.3), and `cargo check -p logos-blockchain-tx-service --all-targets` with default features still compiles `tests/mock.rs` (Appendix B.4).
- *Long term*: if the default-on trick is disliked, the alternative is `#![cfg(feature = "unsafe-test-functions")]` at the top of `tests/mock.rs` plus running that crate's tests with `--all-features` (which `code-check.yml:171` already does); the cost is that `cargo test -p logos-blockchain-tx-service` then silently runs zero integration tests.

**References**: #36 LB-006; `services/tx-service/Cargo.toml:44-52`.

## 5. Suggestions (non-security)

### S-001 · CI guard on the release feature graph

`code-check.yml` checks `--all-features` (`:112`, `:171`) and `--features testing` (`:153`); `nightly-cargo-hack.yml` checks feature combinations per package; `cargo-machete` checks unused dependencies. None of them looks at what the default graph of the node contains, which is why LB-001 survived. Appendix C adds `scripts/check-release-feature-graph.sh`, which runs `cargo tree -e features -p logos-blockchain-node --locked` and fails on any crate feature named `test-util`, `test-utils`, `unsafe-test-functions`, `testing`, `testing-disable-proposal-publish`, or any tokio feature in `full|tracing|test-util|process|signal|fs|io-std|parking_lot`. It fits the `workspace-deps` job next to `scripts/check-workspace-dependency-policies.sh` (`code-check.yml:62-69`) and needs only a lockfile-resolving `cargo tree` (no build).

Two deliberate omissions, both to be added back when their blockers land:

- `unsafe` is not in the list. The KMS `unsafe` feature is on in every build (#36 LB-001, follow-up under #17) and would fail the guard today for a reason this issue does not fix.
- With the fix set applied but `rs-merkle-tree` unchanged, the guard fails on `tokio feature "full"` (LB-004). That is the intended behaviour; the run in Appendix B.4 shows the exact output. Until the upstream fix or a `[patch.crates-io]` lands, `full` (and `fs|io-std|process|signal|parking_lot`, which it implies) should be dropped from `forbidden_tokio` so the guard can be enabled on the remaining patterns.

### S-002 · `tempfile` is a non-dev dependency of `c-bindings`

Re-verified at `a805329`: `c-bindings/Cargo.toml:33` lists `tempfile` under `[dependencies]`; both uses are inside `#[cfg(test)] mod test` (`src/api/config.rs:363-367`, `src/api/lifecycle.rs:203-209`). Moved to `[dev-dependencies]` in Appendix C; `cargo check -p logos-blockchain-c --all-targets` passes.

### S-003 · The `tests` crate has the same bare `tokio` line

`tests/Cargo.toml:72` (`tokio = { workspace = true }`) while the crate uses `tokio::process::Command`, `select!`, `join!`, `sync::{Mutex, broadcast, watch}`, `task::JoinSet`, `time::*` and `net::TcpListener`. It is not in the node graph and gets everything from `tests/testing_framework` (`macros, process, rt-multi-thread, time`) and the node. Out of scope here; left as is, noted so a later sweep of test crates does not miss it.

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

## Appendix B — Evidence

### B.1 Every `tokio` request in the workspace at `a805329` (grep `^tokio\s*=` over `**/Cargo.toml`)

| Crate | Section | Features requested | In node graph |
|---|---|---|---|
| `Cargo.toml:286` (workspace) | `[workspace.dependencies]` | none, `default-features = false` | — |
| `nodes/node/binary:62` | deps | none | yes |
| `blend/network:30` / `:42` | deps / dev | none / none | yes |
| `blend/provers:31` | deps | none | yes |
| `blend/scheduling:26` | deps | none | yes |
| `consensus/cryptarchia-sync:27` | deps | none | yes |
| `services/blend:49` / `:58` | deps / dev | none / `test-util` | yes |
| `services/chain/broadcast-service:22` | deps | none | yes |
| `services/chain/chain-leader:39` | deps | none | yes |
| `services/chain/chain-network:39` | deps | none | yes |
| `services/chain/chain-service:38` | deps | none | yes |
| `services/sdp:28` / `:35` | deps / dev | none / `macros, rt` | yes |
| `services/storage:29` | deps | none | yes |
| `services/time:29` / `:34` | deps / dev | none / `macros, rt, test-util, time` | yes |
| `services/tx-service:37` | deps | none | yes |
| `tracing:40` | deps | none | yes |
| `utils:29` | deps (optional) | none; feature `tokio-task-names` adds `tokio/rt, tokio/tracing` | yes |
| `libp2p:46` | deps | `macros, test-util, time` | yes |
| `c-bindings:34` | deps | `rt-multi-thread` | no |
| `consensus/cryptarchia-engine:29` | deps (optional) | `time` | yes |
| `nodes/api-common:26` | deps (optional) | `time` | yes |
| `kms/keys:33`, `kms/operators:25`, `services/key-management-system:24` | deps | `rt` | yes |
| `services/api:38` | deps | `net` | yes |
| `services/network:28` | deps | `macros, sync` | yes |
| `services/pow:19` / `:40` | deps / dev | `sync, time` / `macros, rt` | yes |
| `services/tracing:30` | deps | `sync` | yes |
| `services/wallet:19` | deps | `sync` | yes |
| `core:51` | dev | none | (dev) |
| `services/utils:36` | dev | none | (dev) |
| `deployment/faucet:24`, `logos_sql:28`, `zone-sdk:31`, `tools/blockchain-tools:29`, `tests/testing_framework:56` | deps | explicit | no |
| `tests:72` | deps | none | no |

### B.2 Production tokio use per crate, the feature it needs, and the declaration added

"Production" means outside `#[cfg(test)]` modules and `tests/` directories. Paths through `lb_utils::tokio::*` are excluded (they are `lb-utils`'s responsibility, LB-003). Feature gates from `tokio-1.52.3/src/lib.rs` (`cfg_rt!` → `runtime`, `task`, `spawn`; `cfg_sync!` → `sync`; `cfg_time!` → `time`; `cfg_net!` → `net`; `cfg_macros!` → `select!`, `join!`, `#[tokio::main]`, `#[tokio::test]`) and `src/task/mod.rs:277-281` (`JoinError`, `JoinHandle`, `spawn_blocking` under `cfg_rt!`).

| Crate | Production paths (file:line) | Needs | Declared after fix (`[dependencies]` / `[dev-dependencies]`) |
|---|---|---|---|
| `nodes/node/binary` | `#[tokio::main]` `main.rs:11`; `runtime::Handle` `lib.rs:40,142`; `net::TcpListener` `api/backend.rs:37`; `sync::oneshot` `api/handlers.rs:81`, `api/tracing.rs:55`, `generic_services/sdp/mempool.rs:25`, `generic_services/blend/pol.rs:11` | `macros`, `rt-multi-thread`, `net`, `sync` | `macros, net, rt-multi-thread, sync` / — (ten `#[tokio::test]` covered) |
| `blend/network` | none (`core/poq_verification.rs:13` is `lb_utils::tokio`) | — | removed / `macros, rt, time` |
| `blend/provers` | `time::Instant` `provers/core/mod.rs:16`, `provers/leader/mod.rs:20`, `provers/pow/mod.rs:30`; `sync::oneshot` `provers/pow/mod.rs:30` | `sync`, `time` | `sync, time` / `macros, rt` |
| `blend/scheduling` | `time::{MissedTickBehavior, interval}` `message_scheduler/mod.rs:14`; `time::{Sleep, sleep}` `epoch.rs:9` | `time` | `time` / `macros, rt` |
| `consensus/cryptarchia-sync` | `sync::mpsc` `libp2p/provider.rs:4`, `libp2p/behaviour.rs:20`; `sync::oneshot`, `time`, `time::error::Elapsed` `libp2p/downloader.rs:5`, `libp2p/errors.rs:3` | `sync`, `time` | `sync, time` / `macros, rt` |
| `services/blend` | `select!` `broadcast/mod.rs:215`, `core/backends/libp2p/swarm.rs:668`, `core/epoch_stages/running.rs:150`; `sync::{broadcast, mpsc, oneshot, watch}` `core/backends/libp2p/{mod.rs:17,swarm.rs:35-38}`, `core/mod.rs:81`, `core/kms.rs:16`, `core/dispatcher/libp2p.rs:27`, `edge/backends/libp2p/{mod.rs:17,swarm.rs:26}`; `time::*` `core/backends/libp2p/{settings.rs:9,tokio_provider.rs:9,53}`, `delivery/failure_detection.rs:16`, `edge/backends/libp2p/swarm.rs:360` | `macros`, `sync`, `time` | `macros, sync, time` / `rt, test-util` |
| `services/chain/broadcast-service` | `sync::{broadcast, oneshot}` `lib.rs:16` | `sync` | `sync` / — |
| `services/chain/chain-leader` | `select!` `leadership.rs:317`, `lib.rs:457`; `sync::{mpsc, oneshot}` `lib.rs:55`, `api.rs:5`, `kms.rs:16`, `mempool/adapter.rs:10`; `task::JoinError` `leadership.rs:35-38` | `macros`, `rt`, `sync` | `macros, rt, sync` / — |
| `services/chain/chain-network` | `select!` `lib.rs:406`; `sync::{broadcast, mpsc, oneshot}` `lib.rs:53-57`, `api.rs:5`, `mempool/adapter.rs:10`, `network/adapters/libp2p.rs:30`; `task::JoinHandle` `lib.rs:55`; `time::{sleep, timeout}` `lib.rs:56`, `bootstrap/ibd.rs:187,207`, `network/adapters/libp2p.rs:211` | `macros`, `rt`, `sync`, `time` | `macros, rt, sync, time` / `lb-network-service/unsafe-test-functions` |
| `services/chain/chain-service` | `select!` `service/phases/{awaiting_genesis_time.rs:136,following.rs:62,ibd.rs:85}`; `sync::{broadcast, mpsc, oneshot, watch}` `lib.rs:62`, `api.rs:13`, `notifier.rs:1`, `service/mod.rs:38`, `sync/block_provider.rs:19`, `storage/adapters/storage.rs:23`; `time::{Interval, interval, sleep, Sleep}` `lib.rs:764`, `service/mod.rs:135`, `service/phases/{awaiting_genesis_time.rs:21-24,90,188,pbp.rs:86}` | `macros`, `sync`, `time` | `macros, sync, time` / `rt-multi-thread` (three `flavor = "multi_thread"` tests) |
| `services/sdp` | `select!` `lib.rs:229`; `sync::oneshot` `lib.rs:38`, `api.rs:12`, `mempool.rs:11` | `macros`, `sync` | `macros, sync` / (unchanged `macros, rt`) |
| `services/storage` | `sync::OnceCell` `recovery.rs:16`; `sync::oneshot` `lib.rs:26,64,70,78,88,106`, `api/chain/requests.rs:12` | `sync` | `sync` / `macros, rt, time` |
| `services/time` | `select!` `lib.rs:142`; `sync::{oneshot, watch}` `lib.rs:18`; under `ntp`: `time::{MissedTickBehavior, interval}` `backends/ntp/mod.rs:16`, `net::*`, `time::{error::Elapsed, timeout}` `backends/ntp/async_client.rs:10-13` | `macros`, `sync`; `ntp` → `net`, `time` | `macros, sync`; `ntp = [..., "tokio/net", "tokio/time"]` / (unchanged) |
| `services/tx-service` | `select!` `tx/service.rs:304`; `sync::{broadcast, oneshot}` `lib.rs:15`, `tx/service.rs:34`, `network/adapters/libp2p.rs:62`, `storage/adapters/rocksdb.rs:76`; behind the new gate: `tokio::spawn` `network/adapters/mock.rs:39` | `macros`, `sync`; gate → `rt` | `macros, sync`; `unsafe-test-functions = [..., "tokio/rt"]` / — |
| `tracing` | `runtime::Handle` `logging/gelf.rs:4`, `logging/loki.rs:4`; `time::sleep` `logging/gelf.rs:28` | `rt`, `time` | `rt, time` / — |
| `utils` (feature `tokio`) | `runtime::Handle`, `task::{JoinError, JoinHandle}` `tokio/mod.rs:9-12`; `tokio::spawn` `:80`; `task::spawn_blocking` `:108` | `rt` | `tokio = ["dep:futures", "dep:tokio", "tokio/rt"]` / `macros, rt, time` |
| `libp2p` | `sync::mpsc` `nat/inner.rs:21,101`, `nat/state_machine/mod.rs:4`; `sync::oneshot` `chainsync/swarm_ext.rs:6`; `net::UdpSocket` `nat/address_mapper/protocols/{nat_pmp.rs:5,pcp_core/connection.rs:8-11}`; `time::*` `nat/gateway_monitor.rs:59,65,90`, `nat/inner.rs:259`, `nat/address_mapper/{mod.rs:24,protocols/pcp.rs:9,protocols/pcp_core/client.rs:37}` | `net`, `sync`, `time` | `net, sync, time` / `macros, rt, test-util` |
| `services/network` (mock, gated) | `time::sleep` `backends/mock.rs:188,211` | gate → `time` | `unsafe-test-functions = ["tokio/time"]` / `rt, time` (its `mod tests` uses `tokio::spawn`, `runtime::Handle`) |
| `core` | none (dev: `#[tokio::test]`, `task::spawn`) | — | — / `macros, rt` |
| `services/utils` | none (dev: `time::sleep` `overwatch/status/macros.rs:188`) | — | — / `macros, rt, time` |

### B.3 Before/after feature graph of `logos-blockchain-node` (default features, `--locked`)

Requesters are the nearest parent of each `tokio feature "…"` line in `cargo tree -e features`. Before = `a805329` unmodified; after = fix set in Appendix C applied, `Cargo.lock` unchanged.

| tokio feature | Before: requesters | After: requesters |
|---|---|---|
| `test-util` | `logos-blockchain-libp2p` | **none** |
| `tracing` | `logos-blockchain-utils feature "tokio-task-names"` (from `blend/network`) | **none** |
| `full` | `rs-merkle-tree v0.1.0` | `rs-merkle-tree v0.1.0` (LB-004, upstream) |
| `macros` | `axum/tokio`, `hickory-proto`, `logos-blockchain-libp2p`, `logos-blockchain-network-service`, `overwatch`, `tokio/full` | **`logos-blockchain-node` (root)**, `axum/tokio`, `hickory-proto`, `logos-blockchain-{blend,chain-leader,chain-network,chain,network,sdp,time,tx}-service`, `overwatch`, `tokio/full` (`lb-libp2p` no longer: test-only) |
| `rt-multi-thread` | `hickory-proto/tokio`, `overwatch`, `tokio/full` | **root**, `hickory-proto/tokio`, `overwatch`, `tokio/full` |
| `rt` | `axum/tokio`, `hickory-proto/tokio`, `hickory-resolver/tokio`, `hyper-util/tokio`, `if-watch`, `libp2p-quic`, `libp2p-swarm`, `logos-blockchain-key-management-system-{keys,operators,service}`, `logos-blockchain-utils/tokio-task-names`, `opentelemetry-otlp`, `opentelemetry_sdk/rt-tokio`, `quinn/runtime-tokio`, `tokio/{full,rt-multi-thread,test-util}`, `tower/buffer` | same third parties; workspace requesters now `logos-blockchain-{chain-leader,chain-network}-service`, `logos-blockchain-key-management-system-{keys,operators,service}`, `logos-blockchain-tracing`, **`logos-blockchain-utils feature "tokio"`** (LB-003); `tokio/test-util` and `logos-blockchain-utils/tokio-task-names` gone |
| `sync` | `hyper`, `hyper-util/client-legacy`, `logos-blockchain-{network,pow,tracing,wallet}-service`, `opentelemetry-otlp`, `overwatch`, `quinn`, `reqwest/blocking`, `tokio/{full,test-util}`, `tokio-stream`, `tokio-util`, `tower/{buffer,limit,ready-cache}`, `tracing-loki` | same third parties; workspace requesters now **root**, `logos-blockchain-{blend-provers,cryptarchia-sync,libp2p}`, `logos-blockchain-{blend,chain-broadcast,chain-leader,chain-network,chain,network,pow,sdp,storage,time,tracing,tx,wallet}-service`; `tokio/test-util` gone |
| `time` | `axum`, `backon/tokio-sleep`, `hickory-proto/tokio`, `hyper-util/tokio`, `libp2p-quic`, `logos-blockchain-{cryptarchia-engine,libp2p,pow-service}`, `opentelemetry_sdk/rt-tokio`, `overwatch`, `quinn/runtime-tokio`, `reqwest` (×2), `tokio/{full,test-util}`, `tokio-stream/time`, `tonic/channel`, `tower/{limit,load,retry,timeout}`, `tower-http/timeout`, `tracing-gelf` | same third parties; workspace requesters now `logos-blockchain-{blend-provers,blend-scheduling,cryptarchia-engine,cryptarchia-sync,libp2p,tracing}`, `logos-blockchain-{blend,chain-network,chain,pow}-service`, `logos-blockchain-time-service/ntp`; `tokio/test-util` gone |
| `net` | `axum/tokio`, `hickory-proto/tokio`, `hyper-util/client`, `igd-next`, `libp2p-quic`, `logos-blockchain-api-service`, `logos-blockchain-time-service/ntp`, `natpmp`, `quinn/runtime-tokio`, `reqwest` (×2), `sntpc`, `tokio/full`, `tokio-util/net`, `tracing-gelf` | same, plus **root** and `logos-blockchain-libp2p` |
| other test-flavoured features (`unsafe-test-functions`, `test-utils`, `testing*`) | none | none |
| KMS `unsafe` | `logos-blockchain-key-management-system-{keys,operators,service} feature "unsafe"` | unchanged (#17) |

Distinct tokio feature names in the graph: 22 before, 20 after (`test-util` and `tracing` removed; the `full` group — `fs`, `io-std`, `parking_lot`, `process`, `signal`, `rt-multi-thread` — remains through `rs-merkle-tree`). The root package's own edges before the fix contained no `tokio feature` line at all (`nodes/node/binary` requested nothing); after, `cargo tree` lists `├── tokio feature "macros" | "net" | "rt-multi-thread" | "sync"` directly under the root. The guard pattern from S-001 applied to the before graph matches `tokio feature "test-util"`, `tokio feature "tracing"` and the six `full`-group lines; applied to the after graph it matches only the six `full`-group lines.

### B.4 Verification runs on the patched copy

All commands run in the private copy at `a805329` with the fix set applied, cargo 1.97.1, rustc 1.98.1 (`rust-toolchain.toml`), `Cargo.lock` untouched.

| Command | Result |
|---|---|
| `cargo tree -e features -p logos-blockchain-node --locked` | exit 0; graph summarised in B.3, root-level lines `tokio feature "macros" / "net" / "rt-multi-thread" / "sync"` |
| `sh scripts/check-release-feature-graph.sh` | exit 1 with exactly six hits: `tokio feature "fs"`, `"full"`, `"io-std"`, `"parking_lot"`, `"process"`, `"signal"` — all from `rs-merkle-tree` (LB-004). `test-util` and `tracing` no longer match. Same script over the unmodified graph additionally matches `tokio feature "test-util"` and `tokio feature "tracing"`. |
| `cargo check -p logos-blockchain-node --locked` | exit 0, 0 warnings |
| `cargo check --workspace --all-targets --all-features --locked` | exit 0, 0 warnings (baseline at the unmodified commit: exit 0, 0 warnings, 3m21s) |
| `cargo check -p logos-blockchain-tx-service -p logos-blockchain-network-service -p logos-blockchain-core -p logos-blockchain-libp2p -p logos-blockchain-blend-network -p logos-blockchain-c --all-targets --locked` (default features, so the gated mocks are exercised as their own crates see them, and `tests/mock.rs` of `lb-tx-service` compiles through the default-on feature) | exit 0, 0 warnings (run with `RUSTFLAGS=""` and a separate target dir because the repo's `.cargo/config.toml` selects `rust-lld` on macOS and that linker cannot read this machine's SDK `.tbd` files when it links freshly built build scripts; the other runs reused build scripts already linked by the baseline run) |
| `taplo fmt --check` on the 21 touched manifests | clean after `taplo fmt` reordered three entries (`libp2p`, `services/network`, `services/tx-service`); the diff in Appendix C is post-format |

Not run: `cargo test` (no behaviour changed; the gate only decides whether a module is compiled), `cargo-machete` (not installed locally; the change removes one dependency line, `blend/network`'s `[dependencies] tokio`, and moves two, so it can only reduce machete's work), `cargo-hack` (its per-package feature powerset would additionally confirm that `lb-tx-service` and `lb-network-service` build with `unsafe-test-functions` off and on; covered here by the default-features and `--all-features` runs).

## Appendix C — Fix set

Unified diff against `a805329f8a186eb6989f09a7c49dee4a0e07473b`, `taplo fmt` clean. `Cargo.lock` is unchanged (no dependency added or removed; `tempfile` stays in the lock as a dev-dependency). The generated `c-bindings/logos_blockchain.h` is rewritten by `c-bindings/build.rs` on every build and is excluded.

```diff
--- a/blend/network/Cargo.toml
+++ b/blend/network/Cargo.toml
@@ -25,9 +25,8 @@
 lb-key-management-system-keys = { workspace = true }
 lb-libp2p                     = { workspace = true }
 lb-log-targets                = { workspace = true }
-lb-utils                      = { features = ["tokio", "tokio-task-names"], workspace = true }
+lb-utils                      = { features = ["tokio"], workspace = true }
 libp2p                        = { workspace = true }
-tokio                         = { workspace = true }
 tracing                       = { workspace = true }
 
 [dev-dependencies]
@@ -39,7 +38,7 @@
 libp2p-stream       = { workspace = true }
 libp2p-swarm-test   = { features = ["tokio"], workspace = true }
 test-log            = { features = ["trace"], workspace = true }
-tokio               = { workspace = true }
+tokio               = { features = ["macros", "rt", "time"], workspace = true }
 tokio-stream        = { workspace = true }
 
 [features]
--- a/blend/provers/Cargo.toml
+++ b/blend/provers/Cargo.toml
@@ -28,7 +28,7 @@
 lb-utils                      = { features = ["tokio"], workspace = true }
 rand                          = { features = ["alloc"], workspace = true }
 rayon                         = { workspace = true }
-tokio                         = { workspace = true }
+tokio                         = { features = ["sync", "time"], workspace = true }
 tracing                       = { workspace = true }
 
 [dev-dependencies]
@@ -39,6 +39,7 @@
 libp2p              = { workspace = true }
 multiaddr           = { workspace = true }
 test-log            = { features = ["trace"], workspace = true }
+tokio               = { features = ["macros", "rt"], workspace = true }
 
 [features]
 tokio-task-names      = ["lb-utils/tokio-task-names"]
--- a/blend/scheduling/Cargo.toml
+++ b/blend/scheduling/Cargo.toml
@@ -23,7 +23,7 @@
 lb-log-targets        = { workspace = true }
 rand                  = { features = ["alloc"], workspace = true }
 thiserror             = { workspace = true }
-tokio                 = { workspace = true }
+tokio                 = { features = ["time"], workspace = true }
 tokio-stream          = { workspace = true }
 tracing               = { workspace = true }
 
@@ -32,6 +32,7 @@
 lb-blend-proofs  = { features = ["unsafe-test-functions"], workspace = true }
 rand             = { features = ["getrandom"], workspace = true }
 rand_chacha      = { workspace = true }
+tokio            = { features = ["macros", "rt"], workspace = true }
 
 [features]
 tokio-task-names      = []
--- a/c-bindings/Cargo.toml
+++ b/c-bindings/Cargo.toml
@@ -30,12 +30,12 @@
 overwatch                     = { workspace = true }
 serde                         = { features = ["derive"], workspace = true }
 serde_json                    = { workspace = true }
-tempfile                      = { workspace = true }
 tokio                         = { features = ["rt-multi-thread"], workspace = true }
 
 [dev-dependencies]
 serde_yaml  = { workspace = true }
 serial_test = { workspace = true }
+tempfile    = { workspace = true }
 
 [build-dependencies]
 cbindgen = { workspace = true }
--- a/consensus/cryptarchia-sync/Cargo.toml
+++ b/consensus/cryptarchia-sync/Cargo.toml
@@ -24,9 +24,10 @@
 serde                 = { workspace = true }
 serde_with            = { workspace = true }
 thiserror             = { workspace = true }
-tokio                 = { workspace = true }
+tokio                 = { features = ["sync", "time"], workspace = true }
 tracing               = { workspace = true }
 
 [dev-dependencies]
 libp2p            = { features = ["dns", "quic", "tokio"], workspace = true }
 libp2p-swarm-test = { workspace = true }
+tokio             = { features = ["macros", "rt"], workspace = true }
--- a/core/Cargo.toml
+++ b/core/Cargo.toml
@@ -48,7 +48,7 @@
 hasher     = { workspace = true }
 rand       = { workspace = true }
 serde_json = { features = ["alloc"], workspace = true }
-tokio      = { workspace = true }
+tokio      = { features = ["macros", "rt"], workspace = true }
 
 [[bench]]
 harness = false
--- a/core/src/mantle/mod.rs
+++ b/core/src/mantle/mod.rs
@@ -4,6 +4,7 @@
 mod fixtures;
 pub mod gas;
 pub mod ledger;
+#[cfg(any(test, feature = "unsafe-test-functions"))]
 pub mod mock;
 pub mod ops;
 pub mod traits;
--- a/libp2p/Cargo.toml
+++ b/libp2p/Cargo.toml
@@ -43,11 +43,12 @@
 serde = { features = ["derive"], workspace = true }
 serde_with = { workspace = true }
 thiserror = { workspace = true }
-tokio = { features = ["macros", "test-util", "time"], workspace = true }
+tokio = { features = ["net", "sync", "time"], workspace = true }
 tracing = { workspace = true }
 zerocopy = { features = ["derive"], workspace = true }
 
 [dev-dependencies]
 libp2p-swarm-test  = { features = ["tokio"], workspace = true }
 serde_json         = { workspace = true }
+tokio              = { features = ["macros", "rt", "test-util"], workspace = true }
 tracing-subscriber = { features = ["env-filter", "fmt"], workspace = true }
--- a/nodes/node/binary/Cargo.toml
+++ b/nodes/node/binary/Cargo.toml
@@ -59,7 +59,7 @@
 serde_with                       = { workspace = true }
 serde_yaml                       = { workspace = true }
 thiserror                        = { workspace = true }
-tokio                            = { workspace = true }
+tokio                            = { features = ["macros", "net", "rt-multi-thread", "sync"], workspace = true }
 tokio-stream                     = { workspace = true }
 tracing                          = { workspace = true }
 url                              = { workspace = true }
--- a/nodes/node/binary/src/generic_services/blend/mod.rs
+++ b/nodes/node/binary/src/generic_services/blend/mod.rs
@@ -1,15 +1,8 @@
-use axum::async_trait;
-use lb_blend::{
-    message::crypto::key_ext::Ed25519SecretKeyExt as _,
-    proofs::{quota::VerifiedProofOfQuota, selection::VerifiedProofOfSelection},
-    scheduling::message_blend::provers::{
-        BlendLayerProof, ProofsGeneratorSettings, WinningPolInfoStream,
-        core_leader_and_pow::RealCoreLeaderAndPowProofsGenerator, leader::LeaderProofsGenerator,
-        leader_and_pow::RealLeaderAndPowProofsGenerator,
-    },
+use lb_blend::scheduling::message_blend::provers::{
+    core_leader_and_pow::RealCoreLeaderAndPowProofsGenerator,
+    leader_and_pow::RealLeaderAndPowProofsGenerator,
 };
 use lb_blend_service::{RealProofsVerifier, core::kms::PreloadKMSBackendCorePoQGenerator};
-use lb_key_management_system_service::keys::UnsecuredEd25519Key;
 use lb_storage_service::{backends::rocksdb::RocksBackend, recovery::StorageRecoveryBackend};
 use lb_time_service::backends::NtpTimeBackend;
 use libp2p::PeerId;
@@ -57,27 +50,6 @@
     BlendCoreRecoveryBackend<RuntimeServiceId>,
     RuntimeServiceId,
 >;
-
-#[derive(Clone)]
-pub struct MockLeaderProofsGenerator;
-
-#[async_trait]
-impl LeaderProofsGenerator for MockLeaderProofsGenerator {
-    fn new(
-        _settings: ProofsGeneratorSettings,
-        _winning_pol_info_stream: WinningPolInfoStream,
-    ) -> Self {
-        Self
-    }
-
-    async fn get_next_proof(&mut self) -> Option<BlendLayerProof> {
-        Some(BlendLayerProof {
-            proof_of_quota: VerifiedProofOfQuota::from_bytes_unchecked([0; _]),
-            proof_of_selection: VerifiedProofOfSelection::from_bytes_unchecked([0; _]),
-            ephemeral_signing_key: UnsecuredEd25519Key::generate_with_chacha_rng(),
-        })
-    }
-}
 
 pub type BlendEdgeService<RuntimeServiceId> = lb_blend_service::edge::BlendService<
     lb_blend_service::edge::backends::libp2p::Libp2pBlendBackend,
--- a/scripts/check-release-feature-graph.sh
+++ b/scripts/check-release-feature-graph.sh
@@ -0,0 +1,31 @@
+#!/usr/bin/env sh
+set -eu
+
+# Fails if the release feature graph of the node binary (what
+# `.github/workflows/prepare-release.yml` builds: default features, `--locked`)
+# contains a feature that only tests should turn on. Feature unification means
+# one stray `features = [...]` in any `[dependencies]` table of the graph is
+# enough to ship it, and neither cargo-machete nor cargo-hack notices.
+#
+# Exit codes: 0 clean, 1 a forbidden feature is enabled, 255 tooling error.
+
+repo_root="$(cd "$(dirname "$0")/.." && pwd)"
+
+# Any crate's feature with one of these names.
+forbidden_features='test-util|test-utils|unsafe-test-functions|testing|testing-disable-proposal-publish'
+# tokio features that nothing in the node uses; `full` is the union of them.
+forbidden_tokio='full|tracing|test-util|process|signal|fs|io-std|parking_lot'
+
+graph="$(cd "$repo_root" && cargo tree -e features -p logos-blockchain-node --locked)" || exit 255
+
+hits="$(printf '%s\n' "$graph" \
+  | grep -E "feature \"(${forbidden_features})\"|tokio feature \"(${forbidden_tokio})\"" \
+  | sed -E 's/^[│ ├└─]+//' | sort -u || true)"
+
+if [ -n "$hits" ]; then
+  echo "error: the release feature graph of logos-blockchain-node enables test-only or unrequested features:" >&2
+  printf '%s\n' "$hits" | sed 's/^/  /' >&2
+  echo "run: cargo tree -e features -p logos-blockchain-node --locked -i <crate> to find who requests it" >&2
+  exit 1
+fi
+echo "release feature graph clean"
--- a/services/blend/Cargo.toml
+++ b/services/blend/Cargo.toml
@@ -46,7 +46,7 @@
 serde                            = { workspace = true }
 serde_with                       = { workspace = true }
 thiserror                        = { workspace = true }
-tokio                            = { workspace = true }
+tokio                            = { features = ["macros", "sync", "time"], workspace = true }
 tokio-stream                     = { workspace = true }
 tracing                          = { workspace = true }
 
@@ -55,7 +55,7 @@
 libp2p            = { features = ["plaintext", "tcp", "yamux"], workspace = true }
 libp2p-swarm-test = { workspace = true }
 test-log          = { features = ["trace"], workspace = true }
-tokio             = { features = ["test-util"], workspace = true }
+tokio             = { features = ["rt", "test-util"], workspace = true }
 
 [features]
 default          = []
--- a/services/chain/broadcast-service/Cargo.toml
+++ b/services/chain/broadcast-service/Cargo.toml
@@ -19,5 +19,5 @@
 lb-log-targets = { workspace = true }
 overwatch      = { workspace = true }
 serde          = { features = ["derive"], workspace = true }
-tokio          = { workspace = true }
+tokio          = { features = ["sync"], workspace = true }
 tracing        = { workspace = true }
--- a/services/chain/chain-leader/Cargo.toml
+++ b/services/chain/chain-leader/Cargo.toml
@@ -36,7 +36,7 @@
 rand                             = { workspace = true }
 serde                            = { workspace = true }
 thiserror                        = { workspace = true }
-tokio                            = { workspace = true }
+tokio                            = { features = ["macros", "rt", "sync"], workspace = true }
 tokio-stream                     = { workspace = true }
 tracing                          = { workspace = true }
 tracing-futures                  = { workspace = true }
--- a/services/chain/chain-network/Cargo.toml
+++ b/services/chain/chain-network/Cargo.toml
@@ -36,14 +36,14 @@
 serde                 = { workspace = true }
 serde_with            = { workspace = true }
 thiserror             = { workspace = true }
-tokio                 = { workspace = true }
+tokio                 = { features = ["macros", "rt", "sync", "time"], workspace = true }
 tokio-stream          = { workspace = true }
 tracing               = { workspace = true }
 tracing-futures       = { workspace = true }
 
 [dev-dependencies]
 lb-core            = { workspace = true }
-lb-network-service = { workspace = true }
+lb-network-service = { features = ["unsafe-test-functions"], workspace = true }
 lb-utils           = { workspace = true }
 
 [features]
--- a/services/chain/chain-service/Cargo.toml
+++ b/services/chain/chain-service/Cargo.toml
@@ -35,7 +35,7 @@
 serde_with                 = { workspace = true }
 thiserror                  = { workspace = true }
 time                       = { workspace = true }
-tokio                      = { workspace = true }
+tokio                      = { features = ["macros", "sync", "time"], workspace = true }
 tracing                    = { workspace = true }
 tracing-futures            = { features = ["std-future"], workspace = true }
 utoipa                     = { features = ["macros"], optional = true, workspace = true }
@@ -48,6 +48,7 @@
 lb-utxotree                   = { workspace = true }
 rand                          = { workspace = true }
 tempfile                      = { workspace = true }
+tokio                         = { features = ["rt-multi-thread"], workspace = true }
 
 [features]
 default = []
--- a/services/network/Cargo.toml
+++ b/services/network/Cargo.toml
@@ -33,9 +33,11 @@
 [dev-dependencies]
 chrono             = { features = ["now"], workspace = true }
 lb-utils           = { workspace = true }
+tokio              = { features = ["rt", "time"], workspace = true }
 tracing-subscriber = { features = ["env-filter", "fmt", "std"], workspace = true }
 
 [features]
-default          = []
-openapi          = ["dep:utoipa"]
-tokio-task-names = ["lb-utils/tokio-task-names"]
+default               = []
+openapi               = ["dep:utoipa"]
+tokio-task-names      = ["lb-utils/tokio-task-names"]
+unsafe-test-functions = ["tokio/time"]
--- a/services/network/src/backends/mod.rs
+++ b/services/network/src/backends/mod.rs
@@ -4,6 +4,7 @@
 use super::Debug;
 
 pub mod libp2p;
+#[cfg(any(test, feature = "unsafe-test-functions"))]
 pub mod mock;
 
 #[async_trait::async_trait]
--- a/services/sdp/Cargo.toml
+++ b/services/sdp/Cargo.toml
@@ -25,7 +25,7 @@
 serde                         = { features = ["derive"], workspace = true }
 thiserror                     = { workspace = true }
 time                          = { workspace = true }
-tokio                         = { workspace = true }
+tokio                         = { features = ["macros", "sync"], workspace = true }
 tracing                       = { workspace = true }
 
 [dev-dependencies]
--- a/services/storage/Cargo.toml
+++ b/services/storage/Cargo.toml
@@ -26,11 +26,12 @@
 rocksdb               = { features = ["bindgen-runtime"], optional = true, workspace = true }
 serde                 = { workspace = true }
 thiserror             = { workspace = true }
-tokio                 = { workspace = true }
+tokio                 = { features = ["sync"], workspace = true }
 tracing               = { workspace = true }
 
 [dev-dependencies]
 tempfile = { workspace = true }
+tokio    = { features = ["macros", "rt", "time"], workspace = true }
 
 [features]
 default          = []
--- a/services/time/Cargo.toml
+++ b/services/time/Cargo.toml
@@ -26,7 +26,7 @@
 sntpc                 = { features = ["std", "tokio-socket"], optional = true, workspace = true }
 thiserror             = { optional = true, workspace = true }
 time                  = { features = ["std"], workspace = true }
-tokio                 = { workspace = true }
+tokio                 = { features = ["macros", "sync"], workspace = true }
 tokio-stream          = { workspace = true }
 tracing               = { workspace = true }
 
@@ -34,4 +34,4 @@
 tokio = { features = ["macros", "rt", "test-util", "time"], workspace = true }
 
 [features]
-ntp = ["dep:sntpc", "dep:thiserror", "tokio/net"]
+ntp = ["dep:sntpc", "dep:thiserror", "tokio/net", "tokio/time"]
--- a/services/tx-service/Cargo.toml
+++ b/services/tx-service/Cargo.toml
@@ -34,7 +34,7 @@
 serde              = { workspace = true }
 serde_json         = { optional = true, workspace = true }
 thiserror          = { workspace = true }
-tokio              = { workspace = true }
+tokio              = { features = ["macros", "sync"], workspace = true }
 tokio-stream       = { workspace = true }
 tracing            = { workspace = true }
 utoipa             = { features = ["macros"], optional = true, workspace = true }
@@ -51,8 +51,12 @@
 # its graph (`krates` panics on the resulting cycle). Workspace consumers are
 # unaffected: `[workspace.dependencies]` sets `default-features = false` and
 # members may not override it, so each still opts in explicitly.
-default         = ["rocksdb-backend"]
+default         = ["rocksdb-backend", "unsafe-test-functions"]
 rocksdb-backend = ["lb-storage-service/rocksdb-backend"]
+# Test doubles (`network::adapters::mock`, and the mock network backend and
+# mock transaction it is built on). Default-on for the same reason as
+# `rocksdb-backend`; never requested by the node.
+unsafe-test-functions = ["lb-core/unsafe-test-functions", "lb-network-service/unsafe-test-functions", "tokio/rt"]
 
 # enable to help generate OpenAPI
 openapi          = ["dep:serde_json", "dep:utoipa"]
--- a/services/tx-service/src/network/adapters/mod.rs
+++ b/services/tx-service/src/network/adapters/mod.rs
@@ -1,2 +1,3 @@
 pub mod libp2p;
+#[cfg(any(test, feature = "unsafe-test-functions"))]
 pub mod mock;
--- a/services/utils/Cargo.toml
+++ b/services/utils/Cargo.toml
@@ -33,5 +33,5 @@
 [dev-dependencies]
 overwatch-derive = { workspace = true }
 serde            = { features = ["derive"], workspace = true }
-tokio            = { workspace = true }
+tokio            = { features = ["macros", "rt", "time"], workspace = true }
 tracing          = { workspace = true }
--- a/tracing/Cargo.toml
+++ b/tracing/Cargo.toml
@@ -37,7 +37,7 @@
 opentelemetry_sdk = { features = ["logs", "rt-tokio"], workspace = true }
 rand = { workspace = true }
 serde = { workspace = true }
-tokio = { workspace = true }
+tokio = { features = ["rt", "time"], workspace = true }
 tonic = { workspace = true }
 tracing = { workspace = true }
 tracing-appender = { workspace = true }
--- a/utils/Cargo.toml
+++ b/utils/Cargo.toml
@@ -34,7 +34,7 @@
 openapi          = ["dep:utoipa"]
 serde            = ["const-hex/alloc", "serde/alloc"]
 time             = ["dep:humantime", "dep:serde_with", "dep:time"]
-tokio            = ["dep:futures", "dep:tokio"]
+tokio            = ["dep:futures", "dep:tokio", "tokio/rt"]
 tokio-task-names = ["tokio", "tokio/rt", "tokio/tracing"]
 
 [dev-dependencies]
@@ -42,3 +42,4 @@
 serde_json = { features = ["alloc"], workspace = true }
 serde_with = { features = ["macros"], workspace = true }
 tempfile   = { workspace = true }
+tokio      = { features = ["macros", "rt", "time"], workspace = true }
```

