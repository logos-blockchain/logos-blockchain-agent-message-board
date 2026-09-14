# Audit Report — Storage: network-identity marker for the state directory, second pass (prototype and verification)

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/80`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `services/storage` (recovery, RocksDB backend), `services/utils` (recovery data), `services/chain/chain-service` (recovery state, block provider), `nodes/node/binary` (startup), `c-bindings` (FFI startup), `tests` (cucumber recovery helper)
Date: `2026-09-12` — author: `claude-fable-5-1` — status: `final`

Builds on the first-pass report for this issue (PR #131, `inbox/80-storage-network-marker.md`, verified at `c3ff08e4`). Findings from that report are referred to by their ids (`#131 LB-001` etc.) and are not restated beyond what this pass adds.

---

## 1. Summary

- Overall assessment: nothing has landed upstream since `c3ff08e4` that records or checks a network identity or a schema version in the state directory; the design proposed in #131 LB-001 holds at `a805329f` with one correction (the genesis is not stored as a block, so a pre-marker directory must be identified through the `recovery/cryptarchia` blob first and the slot index second). The design is now a working patch: 7 files, +603/−3 lines (542 of them the new module with its tests), `cargo check` clean for the storage crate and the node binary, 11 unit tests passing, covering stamp, verify, refuse, adopt, read-only, newer-schema and migration.
- Findings: 0 critical · 0 high · 0 medium · 0 low · 1 informational (new); the three #131 findings are all still open at `a805329f`, anchors re-checked.
- Key themes: "nothing upstream yet", "adoption of pre-marker directories has one hole", "one open site, three readers".
- Must-fix before launch: none on its own; same position as #131: land the marker together with the key-prefix migration from report #63.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/storage/src/recovery.rs` L22-L44, L98-L109, L136-L292 | recovery prefix, single open site, test conventions |
| `services/storage/src/backends/rocksdb.rs` L14-L19, L93-L128 | settings, open modes (read-only vs read-write, column family) |
| `services/storage/src/api/backend/rocksdb/chain.rs` L27-L29, L45-L70, L144-L166 | key layout, what `store_block_data` writes, slot-index scan |
| `services/storage/src/lib.rs` L305-L318; `services/storage/src/bin/rocks.rs` | second open by the storage service; demo binary |
| `services/utils/src/overwatch/recovery/data.rs` | `RecoveryData` API |
| `services/chain/chain-service/src/states.rs` L10-L30, L79-L112; `lib.rs` L581-L617, L745-L770, L965-L1069; `service/mod.rs` L490-L497, L1207-L1222; `service/phases/{awaiting_genesis_time,ibd,pbp,following}.rs` (recording timer arms); `sync/block_provider.rs` L286-L329` | what identifies a directory and when it is written |
| `consensus/cryptarchia-engine/src/lib.rs` L639-L649 | what enters the immutable slot index |
| `nodes/node/binary/src/lib.rs` L140-L162; `config/state.rs`; `config/storage/{mod,serde}.rs`; `config/mod.rs` L305-L310 | node startup and the `STATE_PATH` override |
| `c-bindings/src/api/lifecycle.rs` L74-L106 | FFI startup path |
| `tests/src/cucumber/steps/mempool/actions.rs` L231-L262 | the one out-of-process reader of a live state directory |
| `tools/`, `deployment/scripts/`, `scripts/`, `logos-blockchain-block-explorer-template` | checked for further openers (none) |
| overwatch `ae887f41` `services/resources.rs` L186-L196, `services/runner/service_runner.rs` L262 | error path of a failed blob decode (re-read from the cargo git checkout) |

**Out of scope**
Everything rated in #131 and #63 that this pass does not touch: the wrong-network resume itself (#63 LB-002), the raw-key collision (#63 LB-001), replay-error handling (#131 S-001), deployment tooling (#131 S-002). Per-value-type schema versioning and migration of individual blobs is issue #181 and is being worked in parallel; this report defines the directory-level marker and the single schema-version integer that #181's migrations would key on, not the per-blob format. RocksDB internals, the libp2p handshake, ledger rules. Third-party crates assumed correct: `rocksdb` 0.24 / `librocksdb-sys` 0.17.3, `bincode` 1.3, `overwatch` (read, not audited), `tempfile`.

**Assumptions**
Same as #131: the operator's config is trusted, the state directory is local, and "network" means one genesis block. The prototype was compiled and unit-tested, not run inside a node; item 2 of the issue (a live two-node mismatch) is still a code trace (#131, Method) and is left as the remaining work.

## 3. Method

- Re-verification of #131 at `a805329f`. 17 commits separate `c3ff08e4` from `a805329f`; the ones touching in-scope files are `#3498` (node version in the HTTP API), `#3505` (cargo-deny, chain-service imports), `#3456` (channel config proof, chain-service), `#3489`/`#3487` (API handlers), `#3490` (blend service wiring in `nodes/.../lib.rs`), `#3481` (PoW settings, one line of `states.rs`). None adds a marker, a version, or a genesis comparison; `grep -rn 'meta/\|schema_version\|SCHEMA_VERSION\|network_marker'` over the tree is empty. Every #131 anchor was re-read; the ones that moved are listed under Findings.
- Enumeration of every place that opens a state directory (Table 1), from `grep -rn 'RocksBackend::new\|load_recovery_data\|DB::open'` over the tree plus the FFI, tools, test-framework and block-explorer sources.
- Prototype (Appendix B): the patch was applied to a private copy of the tree at `a805329f` and built with the pinned toolchain (`rustc`/`cargo` 1.98.1, release profile, `CARGO_TARGET_DIR` outside the tree). Tooling: `cargo check --release -p logos-blockchain-storage-service --features rocksdb-backend`, `cargo test --release -p logos-blockchain-storage-service --features rocksdb-backend state_dir`, `cargo check --release -p logos-blockchain-node`. Output in Appendix C. `cargo clippy` was not run.
- Specifications: none cover the state directory (parent #15). No logos-lips commit was read for this pass.
- Dynamic testing: none beyond the unit tests; the node was not started against a foreign directory.

### Table 1 — every path that opens a state directory at `a805329f`

| # | Path | Mode | Identity check today | Where the marker check lands |
|---|---|---|---|---|
| 1 | `nodes/node/binary/src/lib.rs` L161 `load_recovery_data` → `recovery.rs` L33-L36 `RocksBackend::new` | rw (or ro if `storage.backend.read_only`) | none | here: the only open that happens before any service consumes `RecoveryData` |
| 2 | `services/storage/src/lib.rs` L310 `Backend::new` in `StorageService::init` | same settings, second open after (1) has dropped its handle | none | none needed; (1) has already refused or stamped |
| 3 | `c-bindings/src/api/lifecycle.rs` L100 `run_node_from_config` | as (1); config from `get_user_config` + `STATE_PATH` env (`config/mod.rs` L308) | none | inherits (1) |
| 4 | `tests/src/cucumber/steps/mempool/actions.rs` L231-L251 `load_recovery_data` with `read_only: true` on a running node's directory | ro, out of process | none | must stay unchecked: the helper has no genesis in hand; the prototype leaves `load_recovery_data` untouched and adds a new entry point |
| 5 | unit tests in `services/storage` (`recovery.rs`, `rocksdb.rs`, `chain.rs`), `services/chain/chain-service/src/tests/mod.rs` L610, `sync/block_provider.rs` L816, `services/tx-service/tests/mock.rs` L110, L746 | rw, temp dirs | none | none needed |
| 6 | `services/storage/src/bin/rocks.rs` (`logos-blockchain-rocksdb`) | rw/ro on a hard-coded `rocks` path with a `blocks` column family | none | S-003: a demo binary, not a node path |
| — | `tools/blockchain-tools`, `tools/config`, `deployment/scripts/*.sh`, `scripts/*.sh`, `logos-blockchain-block-explorer-template` (HTTP client, Python) | — | do not open the DB; `cleanup_node_data.sh` L3 deletes `state*` wholesale | — |

Conclusion: one site to guard (1); (3) inherits it; (4) is the only reader that must keep working without an expected identity, which fixes the API shape (a new function next to `load_recovery_data`, not a change to it).

### What identifies a pre-marker directory

#131 proposed adopting an unstamped directory by reading the genesis block from `immutable_block/slot/0`. Two corrections from the code:

- The genesis block body is never written. `store_block_data` (`chain.rs` L45-L70) is called only from `process_block` (`service/mod.rs` L819) and the orphan/IBD path (`block_provider.rs` L952) for received blocks; `CryptarchiaConsensusState::from_settings` (`states.rs` L79-L112) builds the genesis ledger in memory and stores nothing. What does exist is the slot-index entry: `prune_immutable_blocks` (`cryptarchia-engine/src/lib.rs` L639-L649) walks from `lib.parent` down to and including the genesis branch when the LIB first advances, and `immutable_blocks_index` (`service/mod.rs` L1207-L1222) writes every returned `(slot, id)` under `immutable_block/slot/<slot BE u64>`. So `immutable_block/slot/0` holds the genesis *id* (32 raw bytes, `chain.rs` L150 for the encoding), and only once the LIB has moved at least once, i.e. after `security_param` blocks. `find_immutable_genesis_block` (`block_provider.rs` L313-L329) already relies on exactly this entry.
- The `recovery/cryptarchia` blob appears much earlier: `record_recovery_state` (`service/mod.rs` L490-L497) is armed on `state_recording_timer` (`lib.rs` L763-L767, default one minute, `bootstrap/state.rs` L131) in every phase including `awaiting_genesis_time` (`awaiting_genesis_time.rs` L155, `ibd.rs` L100, `pbp.rs` L80/L105, `following.rs` L78). A directory whose node ran for one interval is bound to its network through `genesis_id` (`states.rs` L23) even if no block was ever seen.

Adoption rule used by the prototype, in order: (a) `recovery/cryptarchia` decodes and its `genesis_id` matches; (b) else `immutable_block/slot/0` matches; (c) else the directory holds data but cannot be identified and is refused (LB-004). Case (b) is what makes a layout change to `CryptarchiaConsensusState` survivable: the blob no longer decodes, the slot index still names the network, and the migration that follows the adoption can delete the stale blob.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| #131 LB-001 | Recovery blobs carry no schema version or network marker | Configuration | Low | High | Open at `a805329f`; anchors unchanged (`recovery.rs` L98-L109, L33-L44; `rocksdb.rs` L93-L128; `nodes/.../lib.rs` L156-L161); prototype in Appendix B |
| #131 LB-002 | `/chain/id` and `get_chain_id` report the configured chain, not the resumed one | Auditing and Logging | Low | High | Open; handler moved to `handlers.rs` L506-L508 (`#3489`, `#3487`, `#3498` inserted above it), `nodes/.../lib.rs` L147, `c-bindings/src/api/chain.rs` L41 unchanged |
| #131 LB-003 | Blend core adopts a recovered state on epoch-number equality alone | Configuration | Informational | — | Open; `services/blend/src/core/mod.rs` not touched since `c3ff08e4` (`#3490` only moved service wiring in the node crate) |
| LB-004 | A pre-marker directory younger than one LIB advance whose `recovery/cryptarchia` blob no longer decodes has no recoverable identity | Configuration | Informational | — | Open |

### LB-004 · A pre-marker directory younger than one LIB advance whose `recovery/cryptarchia` blob no longer decodes has no recoverable identity

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Configuration |
| Target | `services/chain/chain-service/src/states.rs:L10-L30` (`CryptarchiaConsensusState`, positional bincode); `services/chain/chain-service/src/service/mod.rs:L1207-L1222` (`immutable_blocks_index`); `services/storage/src/api/backend/rocksdb/chain.rs:L27` (`IMMUTABLE_BLOCK_PREFIX`) |
| Status | Open |

**Description**
The two identities a directory written before the marker can offer are the `genesis_id` field inside the `recovery/cryptarchia` blob and the `immutable_block/slot/0` index entry (Method). The first exists after one recording interval but is only readable by a binary whose `CryptarchiaConsensusState` layout matches the writer's, because the blob is positional `bincode` with no version (#131 LB-001). The second exists only after the LIB has advanced past genesis. A directory that is older than a minute, younger than `security_param` blocks, and written by a binary with a different `CryptarchiaConsensusState` layout therefore has data but no identity that the new binary can read. The prototype refuses such a directory (`StateDirError::Unidentifiable`) rather than guess; that is the safe choice, but it means the first release that ships the marker cannot adopt every existing directory, and the adoption path must be shipped in the same release as any change to `CryptarchiaConsensusState` or before it, not after.

**Exploit scenario**
Operator, not attacker. A testnet node started shortly before an upgrade that both adds the marker and changes the recovery struct refuses to start after the upgrade with the `Unidentifiable` message; the remedy is deleting the directory, which for such a young directory costs nothing, but the message must say so.

**Recommendation**
- *Short term*: ship the marker before, or in the same release as, the next change to `CryptarchiaConsensusState`; keep the `Unidentifiable` error text explicit about the age of the directory (it is in the prototype).
- *Long term*: write the genesis id under its own key at first start (`meta/network` is exactly that once the marker lands), so that no future identity depends on a service blob's layout.

**References**: #131 LB-001; issue #181 (per-value versioning); `find_immutable_genesis_block`, `block_provider.rs` L313-L329, for the existing reliance on the slot-index entry.

## 5. Suggestions (non-security)

### S-003 · `logos-blockchain-rocksdb` is a demo binary shipped from the storage crate

| | |
|---|---|
| Target | `services/storage/Cargo.toml` (`[[bin]] name = "logos-blockchain-rocksdb"`, `required-features = ["rocksdb-backend"]`); `services/storage/src/bin/rocks.rs` L1-L50 |

The binary opens a hard-coded relative path `rocks` with a `blocks` column family, writes one fixed key, and loops forever; its "read-only" mode still sets `create_if_missing(true)` (L7). It is not reachable from any deployment or test path (Table 1) but it is a workspace bin target that a `cargo install`/`cargo build --bins` picks up. Either move it under `examples/` or delete it. Noted because the enumeration for this report had to rule it out as a state-directory opener.

### S-004 · The prototype's `RecoveryData::peek` should become the way startup checks read blobs

| | |
|---|---|
| Target | `services/utils/src/overwatch/recovery/data.rs` L26-L31 (`take`) |

`RecoveryData` only offers `take`, which removes the entry, because each service consumes its own blob exactly once. Any check that runs before the services (the marker, a future per-blob version check from #181) needs to read without consuming; the prototype adds a `peek` (Appendix B) and that is the API the #181 work should build on rather than re-loading the prefix.

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

## Appendix B — Prototype patch against `a805329f`

Design, as implemented:

| Key | Value | Written | Checked |
|---|---|---|---|
| `meta/network` (12 bytes) | `[format=1 u8][genesis id 32 bytes][chain id len u16 LE][chain id]` | once, when the directory is empty (`Stamped`) or adopted (`Adopted`) | on every open through `open_recovery_data`; genesis id is the identity, chain id is for messages |
| `meta/schema_version` (19 bytes) | `u32` LE, currently `1` | with the marker; after each migration step | lower than `SCHEMA_VERSION` runs `migrate` step by step and rewrites the key after each; higher refuses (`NewerSchema`) |

Outcomes: `Stamped` (empty directory), `Verified`, `Migrated { from }`, `Adopted(CryptarchiaRecovery | ImmutableIndex)`. Refusals: `WrongNetwork` (names the directory, both chain ids and both genesis ids), `NewerSchema`, `Unidentifiable` (LB-004), `ReadOnly` (unstamped or outdated directory opened with `read_only: true`), `Corrupt`. `load_recovery_data` is unchanged so the cucumber helper (Table 1, row 4) keeps working. Migration `0 → 1` is the marker itself; the `block/`/`tx/` prefix rewrite from #63 LB-001 would be `1 → 2`.

Files: `services/storage/src/state_dir.rs` (new, 542 lines including 11 tests), `services/storage/src/lib.rs` (+2), `services/storage/src/recovery.rs` (+1/−1, `recovery_data_from_backend` made `pub(crate)`), `services/storage/src/backends/rocksdb.rs` (+6, `raw_db`), `services/utils/src/overwatch/recovery/data.rs` (+9, `peek`), `services/chain/chain-service/src/states.rs` (+7, `genesis_id()` accessor), `nodes/node/binary/src/lib.rs` (+36/−2, the call).

```diff
--- a/services/storage/src/state_dir.rs
+++ b/services/storage/src/state_dir.rs
@@ -0,0 +1,542 @@
+//! Network-identity marker and schema version for the `RocksDB` state
+//! directory.
+//!
+//! Every value persisted under the state directory (blocks, the slot index,
+//! every `recovery/*` blob) belongs to exactly one network, identified by its
+//! genesis header id, and to one key layout, identified by
+//! [`SCHEMA_VERSION`]. Neither was recorded before this module existed, so a
+//! directory from another network was resumed silently and a layout change
+//! could only fail at startup without saying why.
+//!
+//! [`open_recovery_data`] is the one place the node opens the directory before
+//! any service consumes it. It stamps a fresh directory, verifies a stamped
+//! one, adopts a pre-marker directory whose genesis can still be recovered,
+//! and refuses everything else with an error that names the directory and
+//! both networks.
+
+use std::path::{Path, PathBuf};
+
+use lb_services_utils::overwatch::recovery::RecoveryData;
+use overwatch::DynError;
+use rocksdb::{DB, IteratorMode, WriteBatch};
+use tracing::info;
+
+use crate::{
+    backends::{
+        StorageBackend as _,
+        rocksdb::{RocksBackend, RocksBackendSettings},
+    },
+    recovery::recovery_data_from_backend,
+};
+
+/// Key of the network marker. Twelve ASCII bytes, so it can never alias a raw
+/// 32-byte block or transaction key, and outside every existing prefix.
+pub const NETWORK_KEY: &[u8] = b"meta/network";
+/// Key of the little-endian `u32` schema version.
+pub const SCHEMA_VERSION_KEY: &[u8] = b"meta/schema_version";
+/// The key layout this binary reads and writes. Bump it whenever a recovery
+/// struct or a key scheme changes, and add the matching step to [`migrate`].
+pub const SCHEMA_VERSION: u32 = 1;
+
+/// Version byte of the marker value itself, independent of the schema version.
+const MARKER_FORMAT: u8 = 1;
+const GENESIS_ID_LEN: usize = 32;
+/// The slot index entry of the genesis block: prefix from
+/// `api/backend/rocksdb/chain.rs` plus `Slot::genesis().to_be_bytes()`.
+const IMMUTABLE_BLOCK_PREFIX: &[u8] = b"immutable_block/slot/";
+
+/// What identifies a network: the genesis header id, which commits to the
+/// genesis transaction (chain id, genesis time, epoch nonce, stake
+/// distribution), plus the human-readable chain id for messages.
+#[derive(Clone, Debug, PartialEq, Eq)]
+pub struct NetworkIdentity {
+    pub genesis_id: [u8; GENESIS_ID_LEN],
+    pub chain_id: Vec<u8>,
+}
+
+impl NetworkIdentity {
+    /// `[format u8][genesis id 32 bytes][chain id length u16 LE][chain id]`.
+    #[must_use]
+    pub fn encode(&self) -> Vec<u8> {
+        let chain_id_len =
+            u16::try_from(self.chain_id.len()).expect("chain ids are bounded far below u16::MAX");
+        let mut bytes = Vec::with_capacity(1 + GENESIS_ID_LEN + 2 + self.chain_id.len());
+        bytes.push(MARKER_FORMAT);
+        bytes.extend_from_slice(&self.genesis_id);
+        bytes.extend_from_slice(&chain_id_len.to_le_bytes());
+        bytes.extend_from_slice(&self.chain_id);
+        bytes
+    }
+
+    pub fn decode(bytes: &[u8]) -> Result<Self, String> {
+        let (&format, rest) = bytes.split_first().ok_or("empty marker")?;
+        if format != MARKER_FORMAT {
+            return Err(format!("unknown marker format {format}"));
+        }
+        if rest.len() < GENESIS_ID_LEN + 2 {
+            return Err(format!("marker too short ({} bytes)", bytes.len()));
+        }
+        let (genesis_id, rest) = rest.split_at(GENESIS_ID_LEN);
+        let (len, chain_id) = rest.split_at(2);
+        let len = usize::from(u16::from_le_bytes([len[0], len[1]]));
+        if chain_id.len() != len {
+            return Err(format!(
+                "chain id length {len} does not match {} trailing bytes",
+                chain_id.len()
+            ));
+        }
+        Ok(Self {
+            genesis_id: genesis_id
+                .try_into()
+                .expect("split_at yields exactly GENESIS_ID_LEN bytes"),
+            chain_id: chain_id.to_vec(),
+        })
+    }
+
+    fn genesis_hex(&self) -> String {
+        hex(&self.genesis_id)
+    }
+
+    fn chain_id_lossy(&self) -> String {
+        String::from_utf8_lossy(&self.chain_id).into_owned()
+    }
+}
+
+/// Where the genesis id of a pre-marker directory was recovered from.
+#[derive(Clone, Copy, Debug, PartialEq, Eq)]
+pub enum LegacySource {
+    /// The `genesis_id` field of the `recovery/cryptarchia` blob.
+    CryptarchiaRecovery,
+    /// The `immutable_block/slot/0` index entry.
+    ImmutableIndex,
+}
+
+#[derive(Clone, Copy, Debug, PartialEq, Eq)]
+pub enum Outcome {
+    /// The directory was empty and has been stamped with the expected
+    /// identity and the current schema version.
+    Stamped,
+    /// The marker matched and the schema version is current.
+    Verified,
+    /// The marker matched and the directory was migrated from an older
+    /// schema version.
+    Migrated { from: u32 },
+    /// A pre-marker directory whose recovered genesis matched; it has been
+    /// stamped and migrated from schema version 0.
+    Adopted(LegacySource),
+}
+
+#[derive(Debug, thiserror::Error)]
+pub enum StateDirError {
+    #[error(
+        "state directory {path} belongs to another network: it holds chain `{found_chain}` \
+         (genesis {found_genesis}) but this node is configured for chain `{expected_chain}` \
+         (genesis {expected_genesis}); move or delete the directory, or point `state.base_folder` elsewhere"
+    )]
+    WrongNetwork {
+        path: PathBuf,
+        expected_genesis: String,
+        expected_chain: String,
+        found_genesis: String,
+        found_chain: String,
+    },
+    #[error(
+        "state directory {path} was written by a newer node (schema version {found}, this binary \
+         supports {supported}); upgrade the node or delete the directory"
+    )]
+    NewerSchema {
+        path: PathBuf,
+        found: u32,
+        supported: u32,
+    },
+    #[error(
+        "state directory {path} holds data but no network marker, and no genesis id could be \
+         recovered from it (no `recovery/cryptarchia` entry and no genesis slot-index entry); \
+         delete the directory if it belongs to this network"
+    )]
+    Unidentifiable { path: PathBuf },
+    #[error(
+        "state directory {path} is open read-only and has no network marker (or an outdated \
+         schema), so it cannot be stamped or migrated; open it read-write once"
+    )]
+    ReadOnly { path: PathBuf },
+    #[error("network marker in state directory {path} is corrupt: {reason}")]
+    Corrupt { path: PathBuf, reason: String },
+    #[error(transparent)]
+    Backend(#[from] rocksdb::Error),
+}
+
+/// Open the state directory, check it against `expected`, and load the
+/// `recovery/*` entries for the services.
+///
+/// `legacy_genesis` is consulted only for a non-empty directory without a
+/// marker; the node passes a closure that decodes the `recovery/cryptarchia`
+/// blob, which this crate cannot name. The slot index is tried after it.
+pub fn open_recovery_data(
+    settings: RocksBackendSettings,
+    expected: &NetworkIdentity,
+    legacy_genesis: impl FnOnce(&RecoveryData) -> Option<[u8; GENESIS_ID_LEN]>,
+) -> Result<(RecoveryData, Outcome), DynError> {
+    let path = settings.db_path.to_path_buf();
+    let read_only = settings.read_only;
+    let backend = RocksBackend::new(settings)?;
+    let recovery_data = recovery_data_from_backend(&backend)?;
+    let outcome = check(backend.raw_db(), &path, read_only, expected, || {
+        legacy_genesis(&recovery_data)
+    })?;
+    info!(
+        target: "storage::state_dir",
+        path = %path.display(),
+        ?outcome,
+        chain_id = %expected.chain_id_lossy(),
+        genesis_id = %expected.genesis_hex(),
+        "state directory checked"
+    );
+    Ok((recovery_data, outcome))
+}
+
+fn check(
+    db: &DB,
+    path: &Path,
+    read_only: bool,
+    expected: &NetworkIdentity,
+    legacy_genesis: impl FnOnce() -> Option<[u8; GENESIS_ID_LEN]>,
+) -> Result<Outcome, StateDirError> {
+    if let Some(bytes) = db.get(NETWORK_KEY)? {
+        let found = NetworkIdentity::decode(&bytes).map_err(|reason| StateDirError::Corrupt {
+            path: path.to_path_buf(),
+            reason,
+        })?;
+        if found.genesis_id != expected.genesis_id {
+            return Err(wrong_network(path, expected, &found));
+        }
+        let version = read_schema_version(db, path)?;
+        return match version {
+            v if v == SCHEMA_VERSION => Ok(Outcome::Verified),
+            v if v > SCHEMA_VERSION => Err(StateDirError::NewerSchema {
+                path: path.to_path_buf(),
+                found: v,
+                supported: SCHEMA_VERSION,
+            }),
+            v => {
+                if read_only {
+                    return Err(StateDirError::ReadOnly { path: path.to_path_buf() });
+                }
+                migrate(db, v)?;
+                Ok(Outcome::Migrated { from: v })
+            }
+        };
+    }
+
+    if read_only {
+        return Err(StateDirError::ReadOnly { path: path.to_path_buf() });
+    }
+
+    let is_empty = db.iterator(IteratorMode::Start).next().is_none();
+    if is_empty {
+        stamp(db, expected, SCHEMA_VERSION)?;
+        return Ok(Outcome::Stamped);
+    }
+
+    // Pre-marker directory: recover its genesis from what it already holds.
+    let (found_genesis, source) = match legacy_genesis() {
+        Some(genesis) => (genesis, LegacySource::CryptarchiaRecovery),
+        None => match db.get(immutable_genesis_key())? {
+            Some(value) => (
+                <[u8; GENESIS_ID_LEN]>::try_from(value.as_slice()).map_err(|_| {
+                    StateDirError::Corrupt {
+                        path: path.to_path_buf(),
+                        reason: format!(
+                            "genesis slot-index entry is {} bytes, expected {GENESIS_ID_LEN}",
+                            value.len()
+                        ),
+                    }
+                })?,
+                LegacySource::ImmutableIndex,
+            ),
+            None => return Err(StateDirError::Unidentifiable { path: path.to_path_buf() }),
+        },
+    };
+    if found_genesis != expected.genesis_id {
+        let found = NetworkIdentity {
+            genesis_id: found_genesis,
+            chain_id: b"<unknown: pre-marker directory>".to_vec(),
+        };
+        return Err(wrong_network(path, expected, &found));
+    }
+    // Stamp at version 0 and run every migration, so a layout change that
+    // predates the marker is applied exactly like any later one.
+    stamp(db, expected, 0)?;
+    migrate(db, 0)?;
+    Ok(Outcome::Adopted(source))
+}
+
+/// Apply every migration from `from` up to [`SCHEMA_VERSION`], rewriting the
+/// version key after each step so an interrupted run resumes where it stopped.
+fn migrate(db: &DB, from: u32) -> Result<(), StateDirError> {
+    for version in from..SCHEMA_VERSION {
+        match version {
+            // 0 -> 1 introduced the marker itself; nothing else changed.
+            0 => {}
+            other => unreachable!("no migration registered from schema version {other}"),
+        }
+        db.put(SCHEMA_VERSION_KEY, (version + 1).to_le_bytes())?;
+    }
+    Ok(())
+}
+
+fn stamp(db: &DB, identity: &NetworkIdentity, version: u32) -> Result<(), StateDirError> {
+    let mut batch = WriteBatch::default();
+    batch.put(NETWORK_KEY, identity.encode());
+    batch.put(SCHEMA_VERSION_KEY, version.to_le_bytes());
+    db.write(batch)?;
+    Ok(())
+}
+
+fn read_schema_version(db: &DB, path: &Path) -> Result<u32, StateDirError> {
+    let bytes = db
+        .get(SCHEMA_VERSION_KEY)?
+        .ok_or_else(|| StateDirError::Corrupt {
+            path: path.to_path_buf(),
+            reason: "network marker present but schema version missing".into(),
+        })?;
+    let bytes: [u8; 4] = bytes
+        .as_slice()
+        .try_into()
+        .map_err(|_| StateDirError::Corrupt {
+            path: path.to_path_buf(),
+            reason: format!("schema version is {} bytes, expected 4", bytes.len()),
+        })?;
+    Ok(u32::from_le_bytes(bytes))
+}
+
+fn wrong_network(path: &Path, expected: &NetworkIdentity, found: &NetworkIdentity) -> StateDirError {
+    StateDirError::WrongNetwork {
+        path: path.to_path_buf(),
+        expected_genesis: expected.genesis_hex(),
+        expected_chain: expected.chain_id_lossy(),
+        found_genesis: found.genesis_hex(),
+        found_chain: found.chain_id_lossy(),
+    }
+}
+
+fn immutable_genesis_key() -> Vec<u8> {
+    let mut key = IMMUTABLE_BLOCK_PREFIX.to_vec();
+    key.extend_from_slice(&0u64.to_be_bytes());
+    key
+}
+
+fn hex(bytes: &[u8]) -> String {
+    bytes.iter().map(|byte| format!("{byte:02x}")).collect()
+}
+
+#[cfg(test)]
+mod tests {
+    use std::collections::HashMap;
+
+    use super::*;
+    use crate::recovery::recovery_key;
+
+    fn identity(tag: u8) -> NetworkIdentity {
+        NetworkIdentity {
+            genesis_id: [tag; GENESIS_ID_LEN],
+            chain_id: format!("chain-{tag}").into_bytes(),
+        }
+    }
+
+    fn settings(dir: &tempfile::TempDir, read_only: bool) -> RocksBackendSettings {
+        RocksBackendSettings {
+            db_path: dir.path().join("db"),
+            read_only,
+            column_family: None,
+        }
+    }
+
+    fn open(
+        dir: &tempfile::TempDir,
+        expected: &NetworkIdentity,
+        legacy: Option<[u8; GENESIS_ID_LEN]>,
+    ) -> Result<Outcome, String> {
+        open_recovery_data(settings(dir, false), expected, |_| legacy)
+            .map(|(_, outcome)| outcome)
+            .map_err(|error| error.to_string())
+    }
+
+    fn raw_put(dir: &tempfile::TempDir, entries: &[(&[u8], &[u8])]) {
+        let backend = RocksBackend::new(settings(dir, false)).unwrap();
+        for (key, value) in entries {
+            backend.raw_db().put(key, value).unwrap();
+        }
+    }
+
+    fn raw_get(dir: &tempfile::TempDir, key: &[u8]) -> Option<Vec<u8>> {
+        let backend = RocksBackend::new(settings(dir, false)).unwrap();
+        backend.raw_db().get(key).unwrap()
+    }
+
+    #[test]
+    fn marker_round_trips() {
+        let identity = identity(7);
+        assert_eq!(
+            NetworkIdentity::decode(&identity.encode()).unwrap(),
+            identity
+        );
+        assert_eq!(identity.encode().len(), 1 + 32 + 2 + 7);
+        assert!(NetworkIdentity::decode(&[]).is_err());
+        assert!(NetworkIdentity::decode(&[MARKER_FORMAT; 10]).is_err());
+        let mut truncated = identity.encode();
+        truncated.pop();
+        assert!(NetworkIdentity::decode(&truncated).is_err());
+    }
+
+    #[test]
+    fn fresh_directory_is_stamped_then_verified() {
+        let dir = tempfile::tempdir().unwrap();
+        let expected = identity(1);
+
+        assert_eq!(open(&dir, &expected, None), Ok(Outcome::Stamped));
+        assert_eq!(
+            raw_get(&dir, NETWORK_KEY).as_deref(),
+            Some(expected.encode().as_slice())
+        );
+        assert_eq!(
+            raw_get(&dir, SCHEMA_VERSION_KEY).as_deref(),
+            Some(SCHEMA_VERSION.to_le_bytes().as_slice())
+        );
+
+        assert_eq!(open(&dir, &expected, None), Ok(Outcome::Verified));
+    }
+
+    #[test]
+    fn stamped_directory_verifies_read_only() {
+        let dir = tempfile::tempdir().unwrap();
+        let expected = identity(1);
+        assert_eq!(open(&dir, &expected, None), Ok(Outcome::Stamped));
+
+        let (_, outcome) =
+            open_recovery_data(settings(&dir, true), &expected, |_| None).unwrap();
+        assert_eq!(outcome, Outcome::Verified);
+    }
+
+    #[test]
+    fn mismatched_marker_refuses_and_leaves_the_directory_alone() {
+        let dir = tempfile::tempdir().unwrap();
+        assert_eq!(open(&dir, &identity(1), None), Ok(Outcome::Stamped));
+
+        let error = open(&dir, &identity(2), None).unwrap_err();
+        assert!(error.contains("belongs to another network"), "{error}");
+        assert!(error.contains("chain-1"), "{error}");
+        assert!(error.contains("chain-2"), "{error}");
+        assert_eq!(
+            raw_get(&dir, NETWORK_KEY).as_deref(),
+            Some(identity(1).encode().as_slice())
+        );
+    }
+
+    #[test]
+    fn legacy_directory_is_adopted_from_the_cryptarchia_blob() {
+        let dir = tempfile::tempdir().unwrap();
+        let expected = identity(3);
+        raw_put(&dir, &[(&recovery_key(b"cryptarchia"), b"opaque blob")]);
+
+        assert_eq!(
+            open(&dir, &expected, Some(expected.genesis_id)),
+            Ok(Outcome::Adopted(LegacySource::CryptarchiaRecovery))
+        );
+        assert_eq!(
+            raw_get(&dir, SCHEMA_VERSION_KEY).as_deref(),
+            Some(SCHEMA_VERSION.to_le_bytes().as_slice())
+        );
+        assert_eq!(open(&dir, &expected, None), Ok(Outcome::Verified));
+    }
+
+    #[test]
+    fn legacy_directory_is_adopted_from_the_slot_index() {
+        let dir = tempfile::tempdir().unwrap();
+        let expected = identity(4);
+        raw_put(
+            &dir,
+            &[
+                (&recovery_key(b"mempool"), b"opaque blob"),
+                (&immutable_genesis_key(), &expected.genesis_id),
+            ],
+        );
+
+        assert_eq!(
+            open(&dir, &expected, None),
+            Ok(Outcome::Adopted(LegacySource::ImmutableIndex))
+        );
+    }
+
+    #[test]
+    fn legacy_directory_from_another_network_refuses() {
+        let dir = tempfile::tempdir().unwrap();
+        raw_put(&dir, &[(&recovery_key(b"cryptarchia"), b"opaque blob")]);
+
+        let error = open(&dir, &identity(5), Some(identity(6).genesis_id)).unwrap_err();
+        assert!(error.contains("belongs to another network"), "{error}");
+        assert!(raw_get(&dir, NETWORK_KEY).is_none());
+    }
+
+    #[test]
+    fn legacy_directory_without_a_genesis_refuses() {
+        let dir = tempfile::tempdir().unwrap();
+        raw_put(&dir, &[(&recovery_key(b"mempool"), b"opaque blob")]);
+
+        let error = open(&dir, &identity(5), None).unwrap_err();
+        assert!(error.contains("no network marker"), "{error}");
+        assert!(raw_get(&dir, NETWORK_KEY).is_none());
+    }
+
+    #[test]
+    fn newer_schema_refuses() {
+        let dir = tempfile::tempdir().unwrap();
+        let expected = identity(1);
+        assert_eq!(open(&dir, &expected, None), Ok(Outcome::Stamped));
+        raw_put(
+            &dir,
+            &[(SCHEMA_VERSION_KEY, &(SCHEMA_VERSION + 1).to_le_bytes())],
+        );
+
+        let error = open(&dir, &expected, None).unwrap_err();
+        assert!(error.contains("newer node"), "{error}");
+    }
+
+    #[test]
+    fn older_schema_is_migrated() {
+        let dir = tempfile::tempdir().unwrap();
+        let expected = identity(1);
+        assert_eq!(open(&dir, &expected, None), Ok(Outcome::Stamped));
+        raw_put(&dir, &[(SCHEMA_VERSION_KEY, &0u32.to_le_bytes())]);
+
+        assert_eq!(
+            open(&dir, &expected, None),
+            Ok(Outcome::Migrated { from: 0 })
+        );
+        assert_eq!(open(&dir, &expected, None), Ok(Outcome::Verified));
+    }
+
+    #[test]
+    fn recovery_entries_are_still_loaded() {
+        let dir = tempfile::tempdir().unwrap();
+        let expected = identity(1);
+        raw_put(
+            &dir,
+            &[
+                (&recovery_key(b"cryptarchia"), b"blob"),
+                (&immutable_genesis_key(), &expected.genesis_id),
+            ],
+        );
+
+        let (data, outcome) =
+            open_recovery_data(settings(&dir, false), &expected, |_| None).unwrap();
+        assert_eq!(outcome, Outcome::Adopted(LegacySource::ImmutableIndex));
+        let entries: HashMap<_, _> = [(recovery_key(b"cryptarchia").to_vec(), b"blob".to_vec())]
+            .into_iter()
+            .collect();
+        for (key, value) in entries {
+            assert_eq!(data.take(&key).unwrap().as_deref(), Some(value.as_slice()));
+        }
+        assert!(data.take(NETWORK_KEY).unwrap().is_none());
+    }
+}
--- a/services/storage/src/lib.rs
+++ b/services/storage/src/lib.rs
@@ -2,6 +2,8 @@
 pub mod backends;
 mod metrics;
 pub mod recovery;
+#[cfg(feature = "rocksdb-backend")]
+pub mod state_dir;
 
 use std::{
     fmt::{Debug, Display, Formatter},
--- a/services/storage/src/recovery.rs
+++ b/services/storage/src/recovery.rs
@@ -36,7 +36,7 @@
 }
 
 #[cfg(feature = "rocksdb-backend")]
-fn recovery_data_from_backend(backend: &RocksBackend) -> Result<RecoveryData, DynError> {
+pub(crate) fn recovery_data_from_backend(backend: &RocksBackend) -> Result<RecoveryData, DynError> {
     backend
         .load_prefix_entries(RECOVERY_PREFIX)
         .map(RecoveryData::new)
--- a/services/storage/src/backends/rocksdb.rs
+++ b/services/storage/src/backends/rocksdb.rs
@@ -55,6 +55,12 @@
             rocks: Arc::clone(&self.rocks),
             executor: Box::new(executor),
         }
+    }
+
+    /// Direct handle for the startup checks in `state_dir`, which run before
+    /// the storage service exists.
+    pub(crate) fn raw_db(&self) -> &DB {
+        &self.rocks
     }
 
     pub(crate) fn load_prefix_entries(
--- a/services/utils/src/overwatch/recovery/data.rs
+++ b/services/utils/src/overwatch/recovery/data.rs
@@ -23,6 +23,15 @@
         Self(Arc::new(Mutex::new(entries)))
     }
 
+    /// Read an entry without removing it, for checks that run before the
+    /// owning service takes it.
+    pub fn peek(&self, key: &[u8]) -> RecoveryResult<Option<Bytes>> {
+        self.0
+            .lock()
+            .map_err(|error| RecoveryError::Backend(error.to_string()))
+            .map(|entries| entries.get(key).cloned())
+    }
+
     pub fn take(&self, key: &[u8]) -> RecoveryResult<Option<Bytes>> {
         self.0
             .lock()
--- a/services/chain/chain-service/src/states.rs
+++ b/services/chain/chain-service/src/states.rs
@@ -35,6 +35,13 @@
         self.tip
     }
 
+    /// The genesis this state was built on; the network identity of a
+    /// pre-marker state directory.
+    #[must_use]
+    pub const fn genesis_id(&self) -> HeaderId {
+        self.genesis_id
+    }
+
     /// Re-create the [`CryptarchiaConsensusState`]
     /// given the cryptarchia engine and ledger state.
     ///
--- a/nodes/node/binary/src/lib.rs
+++ b/nodes/node/binary/src/lib.rs
@@ -21,7 +21,12 @@
     SerdeOp, StorageBackend,
     rocksdb::{RocksBackend, RocksBackendSettings},
 };
-use lb_storage_service::recovery::load_recovery_data;
+use lb_chain_service::CryptarchiaConsensusState;
+use lb_core::codec::DeserializeOp as _;
+use lb_storage_service::{
+    recovery::recovery_key,
+    state_dir::{NetworkIdentity, open_recovery_data},
+};
 pub use lb_system_sig_service::SystemSig;
 use lb_time_service::backends::NtpTimeBackend;
 pub use lb_tracing_service::Tracing;
@@ -158,7 +163,36 @@
     }
     .into_rocks_backend_settings(&config.user.state);
 
-    let recovery_data = load_recovery_data(storage_config.clone())?;
+    // Refuse a state directory that belongs to another network, or one written
+    // by a newer binary, before any service consumes what is in it. The
+    // genesis header id commits to the whole genesis transaction, so it is the
+    // network identity; the chain id is carried for messages only.
+    let expected_network = NetworkIdentity {
+        genesis_id: config
+            .deployment
+            .cryptarchia
+            .genesis_block
+            .header()
+            .id()
+            .into(),
+        chain_id: AsRef::<[u8]>::as_ref(&chain_id).to_vec(),
+    };
+    let (recovery_data, state_dir_outcome) =
+        open_recovery_data(storage_config.clone(), &expected_network, |recovery_data| {
+            // A directory written before the marker existed identifies its
+            // network through the chain service's recovery blob.
+            let bytes = recovery_data
+                .peek(&recovery_key(b"cryptarchia"))
+                .ok()
+                .flatten()?;
+            let state = CryptarchiaConsensusState::from_bytes(&bytes).ok()?;
+            Some(state.genesis_id().into())
+        })?;
+    tracing::info!(
+        path = %storage_config.db_path.display(),
+        outcome = ?state_dir_outcome,
+        "state directory bound to chain {chain_id}"
+    );
 
     let (blend_config, blend_core_config, blend_edge_config) = BlendConfig {
         user: config.user.blend,
```

## Appendix C — Build and test output

Toolchain `1.98.1` (pinned by `rust-toolchain.toml`), release profile, target dir outside the tree. `cargo clippy` not run.

```text
$ cargo check --release -p logos-blockchain-storage-service --features rocksdb-backend
=== cargo check storage (rocksdb-backend) Sat 12 Sep 2026 00:36:20 +04
=== exit 0 Sat 12 Sep 2026 00:38:25 +04

$ cargo test --release -p logos-blockchain-storage-service --features rocksdb-backend state_dir
test state_dir::tests::marker_round_trips ... ok
test state_dir::tests::stamped_directory_verifies_read_only ... ok
test state_dir::tests::legacy_directory_is_adopted_from_the_slot_index ... ok
test state_dir::tests::recovery_entries_are_still_loaded ... ok
test state_dir::tests::mismatched_marker_refuses_and_leaves_the_directory_alone ... ok
test state_dir::tests::newer_schema_refuses ... ok
test state_dir::tests::legacy_directory_from_another_network_refuses ... ok
test state_dir::tests::legacy_directory_without_a_genesis_refuses ... ok
test state_dir::tests::fresh_directory_is_stamped_then_verified ... ok
test state_dir::tests::legacy_directory_is_adopted_from_the_cryptarchia_blob ... ok
test state_dir::tests::older_schema_is_migrated ... ok
test result: ok. 11 passed; 0 failed; 0 ignored; 0 measured; 14 filtered out; finished in 0.03s

$ cargo check --release -p logos-blockchain-node   # first attempt
error[E0282]: type annotations needed
   --> nodes/node/binary/src/lib.rs:178:28
178 |         chain_id: chain_id.as_ref().to_vec(),
    (ChainId implements both AsRef<str> and AsRef<[u8]>; fixed with AsRef::<[u8]>::as_ref(&chain_id), which is what the patch above carries)

$ cargo check --release -p logos-blockchain-node   # after the fix
=== cargo check node (retry) Sat 12 Sep 2026 00:47:13 +04
=== exit 0 Sat 12 Sep 2026 00:48:47 +04
```
