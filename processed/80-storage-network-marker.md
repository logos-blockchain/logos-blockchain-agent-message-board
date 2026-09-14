# Audit Report — Storage: network-identity marker and schema version for the state directory

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/80`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `c3ff08e4a9cbc8344c58a18ef7992a41403c7f9c` — component(s): `services/storage` (recovery, RocksDB backend), `services/utils` (recovery operator), `services/chain/chain-service` (recovery, bootstrap, sync), `services/chain/chain-network` (IBD, orphan handling), `services/{blend,sdp,wallet,tx-service,pow}` (recoverable states), `nodes/node/binary` (startup, chain id API), `deployment/`, `.github/workflows`
Date: `2026-09-07` — author: `claude-fable-5-1` — status: `final`

---

## 1. Summary

- Overall assessment: every one of the seven persisted states is bound to one network, but only `recovery/cryptarchia` carries an identity (`genesis_id`) and nothing reads it; the new `/chain/id` endpoint reports the configured chain rather than the one the node is actually resuming, and no recovery blob carries a schema version, so the key-prefix migration recommended in report #63 currently has no hook and a layout change turns a restart into an unexplained startup failure. The identity that should be checked already exists: the genesis header id commits to the genesis transaction, which contains `chain_id`, `genesis_time`, the epoch nonce and the stake distribution.
- Findings: 0 critical · 0 high · 0 medium · 2 low · 1 informational
- Key themes: "recovered state silently outranks configuration", "no version or identity byte anywhere in the DB", "operational reuse of state directories is only prevented by discipline"
- Must-fix before launch: none on its own; LB-001's marker and version should land together with the key-prefix change from report #63 (LB-001 there), because that change cannot be shipped safely without them.

The issue was filed against `19353c619`, which is on a branch that has diverged from `master`. All in-scope files are identical or trivially refactored at `c3ff08e4`; line numbers below are for `c3ff08e4`.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/storage/src/recovery.rs` L22-L44, L98-L133; `services/storage/src/backends/rocksdb.rs` L60-L78, L93-L128 | recovery prefix, DB open, load of all `recovery/*` entries |
| `services/utils/src/overwatch/recovery/{data.rs,operators.rs}` | `RecoveryData`, `RecoveryOperator::try_load`; overwatch `resources.rs` L186-L196 and `service_runner.rs` L260-L262 at rev `ae887f4` (git dependency) |
| `nodes/node/binary/src/lib.rs` L138-L224; `config/state.rs`; `config/storage/mod.rs` L13-L19; `config/cryptarchia/deployment.rs` L35-L43; `config/deployment/mod.rs` L27-L32 | startup order, state dir, chain id source |
| `nodes/node/binary/src/api/handlers.rs` L500-L506; `nodes/api-common/src/paths.rs` L6; `c-bindings/src/api/chain.rs` L41-L56; `c-bindings/src/node.rs` L66-L68 | chain id exposure added by #3485 |
| `services/chain/chain-service/src/states.rs` L11-L30, L75-L112; `lib.rs` L410-L500, L582-L610, L697-L800, L966-L1070; `bootstrap/state.rs` L11-L28; `service/phases/awaiting_genesis_time.rs` L177-L193; `sync/block_provider.rs` L247-L311 | recovery vs configured genesis, replay, engine state, block serving |
| `services/chain/chain-network/src/bootstrap/ibd.rs` L233-L337; `sync/orphan_handler.rs` L152-L212 | what a node does with tips from another chain |
| `services/blend/src/core/state.rs` L21-L29, L580-L610; `core/mod.rs` L280-L300, L661-L722 | blend recovery adoption rule |
| `services/sdp/src/state.rs` L12-L18; `lib.rs` L170-L190 | SDP recovery vs settings |
| `services/wallet/src/states.rs` L200-L224, L240-L281, L306-L330 | wallet pinned to persisted LIB |
| `services/tx-service/src/backend/pool.rs` L55-L63, L278-L300; `tx/service.rs` L223-L232 | mempool recovery |
| `services/pow/src/service.rs` L300-L316 | PoW ticket recovery |
| `core/src/header/mod.rs` L174-L178, L222-L229; `core/src/mantle/transactions/genesis_tx.rs` L272-L281, L410-L416 | what the genesis id commits to |
| `deployment/{compose.yml,compose.setup.yml,compose.run.yml}`, `deployment/scripts/{cleanup_node_data.sh,run_node.sh}`, `deployment/systemd/logos-blockchain-node.service`, `.github/workflows/{genesis-ceremony.yml,nightly-cluster-fork-detector.yml}`, `tests/testing_framework/src/framework/local/provisioning.rs` L160-L200, `tools/config/src/unique.rs` L25-L40, `nodes/node/binary/src/config/deployment/settings.yaml` L5-L19, L85-L92, L122 | state-dir reuse across genesis regenerations; protocol names |

**Out of scope**
Everything already rated in report #63 (`inbox/63-storage-network-keys-growth-sql-permissions.md`): the block/transaction key collision (LB-001 there), the wrong-network resume itself (LB-002 there, Medium, restated here only as far as needed to design the marker), write error reporting, file permissions, orphaned blocks, the unused column family. RocksDB internals, the libp2p handshake, the ledger's block validation rules, the blend membership and reward logic, and the k8s testing-framework path (`framework/k8s`). Third-party crates assumed correct: `rocksdb`, `bincode`, `overwatch` (read, not audited).

**Assumptions**
The operator's config file is trusted. The state directory is on a local disk owned by the node. "Network" means one genesis block: two ceremonies with the same `chain_id` string but different entropy or stakeholders are two networks. No devnet was run for this report (see Method); the mismatch behaviour is a code trace.

## 3. Method

- Manual review of the in-scope paths, working through sub-issue #80 under parent #15 and the four items it lists, with report #63 (LB-001, LB-002) as the starting point. Every statement below was traced to code at `c3ff08e4`; the `overwatch` lines were read from the pinned checkout (`ae887f4`) in the cargo cache.
- Spec conformance: none; there is no written spec for the state directory.
- Automated tooling: none.
- Dynamic testing: none. Item 2 of the issue asks for a two-node devnet run with a mismatched state directory; that requires a node build (not attempted on this host) and is left as the remaining work on #80. The expected outcome is derived below from the recovery, sync and orphan-handling code, with the exact branches cited.

Enumeration of persisted state (item 1). All entries live in one flat RocksDB key space (`rocksdb.rs` L93-L128); the recovery blobs are loaded in one prefix scan at startup (`recovery.rs` L33-L44) and handed to each service, which uses the stored blob whenever it exists and calls `from_settings` only when it does not (overwatch `resources.rs` L186-L196).

| Key | Contents | Network-bound? | With a blob from another network |
|---|---|---|---|
| `recovery/cryptarchia` | `tip`, `lib`, full `lib_ledger_state`, `genesis_id`, `last_engine_state` (`states.rs` L11-L30) | yes; the only blob that names its network | resumed verbatim; `genesis_id` never compared (`lib.rs` L980-L997) |
| `recovery/mempool` | pending tx keys with timestamps, removed keys (`pool.rs` L55-L63) | yes (hashes of that chain's txs, bytes under raw 32-byte keys in the same DB) | re-admitted (`pool.rs` L278-L300); harmless alone, but a leader includes them in blocks of whatever chain it is on |
| `recovery/blend/core` | `last_seen_epoch`, `spent_core_quota`, unsent messages, pending txs, blending-token collectors (`state.rs` L21-L29) | yes (quota and tokens are per epoch of one membership) | adopted whenever `last_seen_epoch == current epoch` (`core/mod.rs` L661-L669), an integer that is not chain-specific; a collector one epoch old is even rotated into an activity-proof submission (L697-L711) |
| `recovery/sdp` | `declaration_id`, `updated`, `pending_activity` (`state.rs` L12-L18) | yes (declaration ids are ledger state) | the stored id outranks the configured one (`sdp/lib.rs` L177-L180); activity and withdrawal are then posted for a declaration the configured chain does not have |
| `recovery/wallet` | vouchers, next voucher index, `(lib header, WalletState)`, pending claims (`states.rs` L200-L210) | yes | wallet starts from the persisted LIB (`states.rs` L260-L264); `advance_lib` refuses to move to any LIB it has not applied (L319-L327), so it stays pinned with only a `warn!` |
| `recovery/pow` | mined tickets ready or pending to claim (`service.rs` L300-L307) | yes (tickets name slots and nullifiers) | claims are built and rejected |
| raw 32-byte keys, `block_parent/`, `block_events/`, `immutable_block/slot/` | blocks, parents, events, slot index (`chain.rs` L27-L29) | yes | served to peers by the block provider, which falls back to the stored genesis when it shares no block with the requester (`block_provider.rs` L286-L311) |

Conclusion for item 1: all seven are network-bound, none can be reused across networks, and only one carries an identity. The marker therefore belongs to the directory, not to a blob. The identity to store is the genesis header id: `Header::genesis` hashes `parent = 0`, the body root over the genesis transaction, slot 0 and the genesis proof (`header/mod.rs` L174-L178, L222-L229), and the genesis transaction carries `chain_id`, `genesis_time` and `epoch_nonce` (`genesis_tx.rs` L410-L416) plus the stake distribution. `chain_id` alone is a per-release string (`"X.Y.Z-rc.N"`, `deployment/ceremony/genesis/devnet/inscribe.yaml`) that two ceremonies can share.

What the node does on a mismatch (item 2, code trace). Let the configured genesis be A and the state directory be from B.

1. Recovery outranks configuration: `CryptarchiaConsensusState::from_settings` (`states.rs` L79-L112) is never called when the blob exists. A's genesis block is only used for the genesis-time timer (`awaiting_genesis_time.rs` L177-L193) and for the chain id handed to the API (`nodes/.../lib.rs` L145). Everything else in `CryptarchiaSettings` (ledger config, bootstrap, sync; `lib.rs` L582-L590, L704-L714) is A's and is applied to B's ledger state (`lib.rs` L988-L997).
2. Engine state: `choose_engine_state` (`bootstrap/state.rs` L11-L28) sees `lib != genesis` and uses the persisted `last_engine_state` and grace period; a node restarted within the grace period comes up `Online` at once and, if it holds a leader key, proposes on B's tip with A's parameters.
3. Replay: blocks in `(LIB, tip]` are re-applied under A's slot clock (`lib.rs` L1039-L1057); with a later `genesis_time` they fail `FutureBlock` (L443-L447). Errors are logged and skipped (L1054-L1056), so the node silently sits at B's LIB.
4. Network: every protocol name and gossip topic carries only the release version (`settings.yaml` L5, L17-L19, L85, L122), not the genesis, so A's peers connect. IBD fetches their tips, finds them unknown and enqueues them as orphans (`ibd.rs` L233-L250, L323-L337). The peer's block provider shares no block with us and streams from its own genesis (`block_provider.rs` L286-L311); the first block fails with `ParentMissing` (`lib.rs` L462-L478) and is cached as rejected together with its descendants (`orphan_handler.rs` L159-L170). The same happens in reverse to any block we propose.

Net effect: no block ever crosses between A and B; peers are not rejected, the node does not crash, and it either idles on B or, as a leader, extends a private fork of B with A's ledger rules. `/chain/id` reports A throughout (LB-002). The wallet, SDP and PoW services keep acting on B's state (table above). This matches the prose of #63 LB-002; the new facts are the blend adoption rule, the SDP precedence, and that the wrong-chain condition is invisible to the one endpoint an operator would poll.

State-directory reuse (item 3).

| Path | Reuses a state dir across genesis regenerations? | Evidence |
|---|---|---|
| `nightly-cluster-fork-detector.yml` | no | runs the testing framework (L46-L55); each node gets `scenario_base_dir/<node>` (`provisioning.rs` L168, L192-L193) under a fresh `tests/.tmp*` per run, and each run mints a unique `chain_id` from thread, workspace, runner and attempt (`unique.rs` L25-L40). Artifacts are only uploaded (L64-L67). |
| `genesis-ceremony.yml` | n/a | rewrites `settings.yaml` and pushes (L38-L62); never touches a node. The reuse happens at deploy time. |
| `deployment/compose.*` | not by default, unenforced | `compose.setup.yml` wipes `/node-data/*/state*` and copies the deployment file onto the volume (`cleanup_node_data.sh` L3-L9); `compose.run.yml` reads `STATE_PATH=/node-data/<n>/state` (L49) and `run_node.sh` reads `/node-data/deployment.yaml` (L5-L7). Config and state live on the same external volume and are replaced together, so `docker compose up` after a ceremony without re-running setup keeps the old network consistently. Nothing stops an operator from replacing only `deployment.yaml`. |
| `deployment/systemd` | yes | `ExecStart=… config.yaml` (service L13) with the state dir defaulting to `./state` (`config/state.rs` L14) relative to the working directory; swapping the config after a ceremony reuses the directory. This is the realistic path for testnet resets. |

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Recovery blobs carry no schema version or network marker, so a layout change or a wrong directory is either resumed silently or fails startup without saying why | Configuration | Low | High | Open |
| LB-002 | `/chain/id` and `get_chain_id` report the configured chain, not the chain being resumed from storage | Auditing and Logging | Low | High | Open |
| LB-003 | Blend core adopts a recovered state on epoch-number equality alone | Configuration | Informational | — | Open |

### LB-001 · Recovery blobs carry no schema version or network marker, so a layout change or a wrong directory is either resumed silently or fails startup without saying why

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Configuration |
| Target | `services/storage/src/recovery.rs:L98-L109` (`load_state`), `:L33-L44` (`load_recovery_data`); `services/storage/src/backends/rocksdb.rs:L93-L128` (`new`); `nodes/node/binary/src/lib.rs:L154-L159`; overwatch `services/runner/service_runner.rs:L260-L262`, `services/resources.rs:L186-L196` (rev `ae887f4`) |
| Status | Open |

**Description**
Each recovery blob is a positional `bincode` encoding of a service struct, decoded with no version or magic:

```rust
// services/storage/src/recovery.rs
 98 fn load_state(settings: &Settings) -> RecoveryResult<Option<Self::State>> {
 99     let Some(bytes) = settings.recovery_data().take(&recovery_key(Settings::RECOVERY_KEY_SUFFIX))? else {
103         return Ok(None);
104     };
106     State::from_bytes(&bytes).map(Some)
108         .map_err(|error| RecoveryError::Backend(error.to_string()))
```

A decode error is not a fallback to `from_settings`; overwatch turns it into a service start failure, and the node exits:

```rust
// overwatch/src/services/resources.rs (ae887f4)
186 match StateOperator::try_load(&settings) {
187     Ok(Some(loaded_state)) => { .. Ok(loaded_state) }
191     Ok(None) => { .. State::from_settings(&settings) .. }
195     Err(error) => Err(InitialStateError::Operator(error)),
// overwatch/src/services/runner/service_runner.rs
260 let initial_state = service_resources.get_service_initial_state()
262     .map_err(|error| format!("Failed to create the initial state: {error}"))?;
```

The DB itself is opened with `create_if_missing(true)` and nothing else (`rocksdb.rs` L111-L122); no key records which binary wrote it, which key layout it uses, or which genesis it belongs to. Two consequences:

- Any field added, removed or reordered in `CryptarchiaConsensusState`, `PoolRecoveryState`, `SerializableServiceState`, `SdpState`, `RecoveryState` (wallet) or `PoWServiceState` changes the encoding. The next start after an upgrade fails with "Failed to create the initial state: …" and the only remedy is deleting `state/db`, which also deletes every block. `#[serde(default)]` on `SdpState::pending_activity` (`state.rs` L16) does not help: `bincode` is not self-describing, so a missing trailing field is an EOF, not a default. The blend service's `Option` wrapper and `EpochMismatch` handling (`core/mod.rs` L290-L300) run after `from_bytes` and cannot catch this either.
- The key-prefix migration recommended in #63 LB-001 (prefix blocks with `block/` and transactions with `tx/`) has no way to tell an old directory from a new one, so it cannot be rolled out without a wipe.

The network side is #63 LB-002 and is not re-rated here; the point of this finding is that both problems have one fix, and that the check must run before any service consumes `RecoveryData`.

**Exploit scenario**
Operator, not attacker. (a) A release changes a recovery struct; every upgraded node stops at startup with the overwatch error, which names neither the blob nor the cure. (b) A systemd operator swaps `config.yaml` for a new testnet genesis and restarts; the node resumes the old network as traced in Method, and `/chain/id` reports the new one (LB-002).

**Recommendation**
- *Short term*: add a marker check at the single point where the directory is opened for recovery, `load_recovery_data` (`recovery.rs` L33-L36, called from `nodes/.../lib.rs` L159 before any service starts), taking the expected identity from the deployment config that is already in hand at that point (`config.deployment.cryptarchia.genesis_block.header().id()` for `StartingState::Genesis`, the configured `genesis_id` for `StartingState::Lib`, and `config.deployment.chain_id()`, `lib.rs` L145). Proposed keys, in the existing flat key space (no prefix extractor, `rocksdb.rs` L178-L190), chosen so they cannot alias a raw 32-byte block or transaction key:

  | Key | Value | Written | Checked |
  |---|---|---|---|
  | `meta/network` (12 bytes) | `bincode(NetworkMarker { genesis_id: HeaderId, chain_id: ChainId })` | once, when the directory is created (DB opened and no `recovery/*` entry and no `immutable_block/slot/0` entry exist) | on every open, including `read_only` opens (compare only, never write) |
  | `meta/schema_version` (19 bytes) | `u32` little-endian, starting at `1` | with the marker, and after each successful migration | on every open: lower than the binary's `SCHEMA_VERSION` runs the migrations in order and then rewrites the key; higher refuses ("written by a newer node") |

  Rules: marker present and `genesis_id` differs → refuse to start with a message that names the directory, both chain ids and both genesis ids, and says to move or delete the directory. Marker absent but the directory is non-empty (every directory written before this change) → read the genesis block from `immutable_block/slot/0` (`chain.rs` L27), compare its id with the expected one, and on match stamp `meta/network` and `meta/schema_version = 0` so that the prefix migration can run next; on mismatch refuse as above. Marker present and equal → proceed. Record the outcome in one `info!` line and a metric. Add three tests next to `missing_recovery_state_returns_none` (`recovery.rs` L225): fresh directory is stamped; mismatched marker refuses; legacy directory with a matching genesis is adopted.
- *Long term*: make the version the dispatch point for every layout change: bump `SCHEMA_VERSION` whenever a recovery struct or key scheme changes, and give each bump a migration that either rewrites the affected keys (the `block/` and `tx/` prefixes from #63) or deletes the affected `recovery/*` entry so that the service falls back to `from_settings`, which every service already supports (`states.rs` L79-L112; `state.rs` L35; `service.rs` L313; wallet `states.rs` L216; blend `state.rs` L604; mempool `tx/service.rs` L225-L226). Prefer that over per-blob version bytes, because all seven layouts ship in one binary and one integer describes them together. Expose `meta/network` through `/chain/id` (LB-002).

**References**: report #63 LB-001 (prefixing needs a migration hook), LB-002 (wrong-network resume, Medium), S-002 (`logos_sql` lacks `user_version`, same class); parent #15.

### LB-002 · `/chain/id` and `get_chain_id` report the configured chain, not the chain being resumed from storage

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Auditing and Logging |
| Target | `nodes/node/binary/src/lib.rs:L142-L145`, `:L219-L224`; `nodes/node/binary/src/api/handlers.rs:L500-L506` (`chain_id`); `c-bindings/src/api/chain.rs:L41-L56` (`get_chain_id`); `nodes/node/binary/src/config/cryptarchia/deployment.rs:L38-L43` |
| Status | Open |

**Description**
Commit `c3ff08e4` ("Expose chain id on http and ffi", #3485) reads the chain id off the configured genesis block and hands it to the API and FFI as a constant:

```rust
// nodes/node/binary/src/lib.rs
142 // Read before the deployment settings are consumed piecewise below. The
143 // chain ID is fixed by the deployment, so the API backend is handed it up
144 // front rather than querying a service for a value that cannot change.
145 let chain_id = config.deployment.chain_id();
// nodes/node/binary/src/api/handlers.rs
505 pub async fn chain_id(Extension(chain_id): Extension<ChainId>) -> Response {
506     Json(ChainIdResponseBody { chain_id }).into_response()
```

The comment's premise holds for the configuration, not for the node: the chain it runs is the one in `recovery/cryptarchia` (Method, item 2), which the configuration never overrides. The endpoint is exactly what a fleet check or a wallet would use to confirm which network a node is on, and it cannot detect the case the marker is meant to catch. The FFI copy (`c-bindings/src/api/chain.rs` L41-L56) is populated from the same value.

**Exploit scenario**
Operator, not attacker. After a testnet reset, a node left on the old state directory answers the new `chain_id` on `/chain/id` while gossiping the old chain; monitoring built on the endpoint reports it healthy.

**Recommendation**
- *Short term*: once LB-001's check exists, the value is trustworthy because startup refuses a mismatch; until then, add the recovered `genesis_id` (`Cryptarchia::genesis_id`, `lib.rs` L329, already reachable from `CryptarchiaInfo`) to the response body next to `chain_id`, so a client can compare it with the expected genesis.
- *Long term*: keep the response derived from a single source, the marker read at open (`meta/network`), and delete the config-side constant.

**References**: `nodes/api-common/src/bodies/chain.rs` L10; report #53 B19 and issue #49 (transactions carry no chain id either).

### LB-003 · Blend core adopts a recovered state on epoch-number equality alone

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Configuration |
| Target | `services/blend/src/core/mod.rs:L661-L669`, `:L697-L711` |
| Status | Open |

**Description**
The blend core service keeps its recovered `SerializableServiceState` when `saved_state.last_seen_epoch() == current_epoch_public_info.epoch` (L662) and otherwise starts fresh, except that a state exactly one epoch old has its token collector rotated and submitted as an activity proof (L701-L707). The epoch is a plain counter (`Epoch`), so a directory from another network whose epoch counter happens to match is adopted: its `spent_core_quota` throttles this node's sending under a different membership, and its tokens are turned into an activity-proof transaction that the ledger rejects. With LB-001 in place this cannot happen; without it, the comparison is the only guard and it is not one. Noted here because the enumeration in Method is the deliverable of item 1 and this is the only blob with an explicit, but insufficient, adoption rule.

**Exploit scenario**
None from the network; same operator scenario as LB-001.

**Recommendation**
- *Short term*: none beyond LB-001.
- *Long term*: include the epoch nonce (`pol_epoch_nonce`, already in `current_epoch_public_info`) in `SerializableServiceState` and compare it together with the epoch number.

**References**: none.

## 5. Suggestions (non-security)

### S-001 · Replay errors during chain recovery are logged and skipped, leaving the persisted tip ahead of the engine

| | |
|---|---|
| Target | `services/chain/chain-service/src/lib.rs:L1039-L1057`, `:L751-L757` |

`initialize_cryptarchia` applies each stored block in `(LIB, tip]` and on error only logs `"Error processing block"` (L1054-L1056). `fell_back_to_lib` is set only when blocks are missing from storage (L932-L942), not when they fail to apply, so the recovery snapshot is not rewritten (L751-L757) and still names the old tip until the next block arrives. A block that stops applying after a restart (a `FutureBlock` when `genesis_time` moved, or a ledger-rule change) therefore produces a silent rewind to LIB with one `error!` line per block. This answers parent #15's question "what does recovery do if the persisted tip cannot be reapplied": it rewinds and carries on. Suggest treating the first apply failure like the missing-block case (log at `warn!` with the reason, set `fell_back_to_lib`, persist), and exposing the rewind as a metric.

### S-002 · Deployment tooling does not tie the state directory to the deployment file it was created with

| | |
|---|---|
| Target | `deployment/compose.setup.yml:L5-L12`; `deployment/scripts/cleanup_node_data.sh:L3-L9`; `deployment/scripts/run_node.sh:L5-L7`; `deployment/systemd/logos-blockchain-node.service:L13`; `deployment/README.md` |

The compose setup wipes state and installs the deployment file in one step, but nothing in `compose.yml` or the README says that a ceremony requires re-running it, and the systemd unit has no setup step at all (Method, item 3 table). Until LB-001 lands, add to `deployment/README.md` and `deployment/systemd/README.md` that a genesis ceremony invalidates `state/` and must be followed by `docker compose -f deployment/compose.setup.yml up` or an `rm -rf state/`, and have `run_node.sh` record the sha256 of `deployment.yaml` next to the state directory and refuse to start when it changed.

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
