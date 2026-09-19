# Audit Report — Storage state-directory identity and schema re-verification

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/80`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `services/storage`, `services/chain/chain-service`, `services/blend`, `services/sdp`, `services/wallet`, `services/tx-service`, `services/pow`, `nodes/node/binary`, `deployment/`, `.github/workflows/`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `docs/blockchain/raw/bedrock-architecture-overview.md`, `docs/blockchain/raw/overview-cryptoeconomics.md`
Date: `2026-09-18` — author: `codex` — status: `final`

The issue's historical source hint was `19353c619`; that revision is not present in the permitted audit checkout. This pass uses the available clean `logos-blockchain` revision above.

---

## 1. Summary

- Overall assessment: the third pass finds no remediation in the current source; state directories still have no network marker or schema version, recovered chain state still outranks the configured genesis, and the chain-id API still reports configuration rather than the resumed state.
- Findings: `0 new` · `1` existing medium re-verified · `2` existing low re-verified · `1` existing informational re-verified
- Key themes: `wrong-network state reuse`, `unversioned positional recovery`, `configuration-only network identity`
- Must-fix before launch: the canonical open findings `63-LB-002` / `#402` (network identity), `181-LB-001` / `#524` (schema migration), and `80-LB-002` / `#362` (accurate chain identity) remain unresolved. No new finding ID is assigned.

The historical marker reports were already processed into tracker issues. This report is a third-pass re-verification of those issues, not a duplicate assignment of the same defects.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/storage/src/recovery.rs`, `services/storage/src/backends/rocksdb.rs` | The pre-service recovery open, RocksDB creation, recovery key space, and positional decoding. |
| `nodes/node/binary/src/lib.rs`, `nodes/node/binary/src/api/handlers.rs`, `c-bindings/src/api/chain.rs` | Startup ordering and the HTTP/FFI chain identity values. |
| `services/chain/chain-service/src/{states.rs,lib.rs,service/mod.rs,sync/block_provider.rs}` | Configured versus recovered genesis, replay behavior, and the immutable genesis index. |
| `services/blend/src/core/{mod.rs,state.rs}`, `services/sdp/src/lib.rs`, `services/wallet/src/states.rs`, `services/tx-service/src/backend/pool.rs`, `services/pow/src/service.rs` | Whether recoverable service state is network-bound and how it is adopted. |
| `deployment/`, `.github/workflows/`, `tests/testing_framework/src/framework/local/provisioning.rs`, `tools/config/src/unique.rs` | State-directory reuse and fresh-state behavior in deployment and test paths. |

**Out of scope**

The implementation of the fixes tracked by `#402`, `#524`, `#362`, and `#364`; a live two-node devnet run; RocksDB internals; consensus and ledger validation; and third-party crates assumed correct, including `rocksdb`, `bincode`, `overwatch`, and `tempfile`.

**Assumptions**

The operator configuration and local state filesystem are trusted. A network is identified by its genesis header ID, not only by the human-readable `chain_id`; two ceremonies with the same chain-id string but different genesis material are different networks. The existing issue and prior reports are used as evidence for the full enumeration of recoverable service state; this pass rechecked the relevant adoption guards and startup paths.

## 3. Method

- Manual re-verification of issue `#80` under parent `#15`, including the processed reports `80-storage-network-marker.md` and `80-storage-network-marker-second-pass.md`.
- Re-read the pinned core specifications listed in the header before inspecting source. Neither specification defines a state-directory marker or migration format, so this pass treats the implementation's network identity and recovery behavior as the audit subject rather than claiming a direct protocol-spec violation.
- Re-checked the current clean source checkout at `a805329f` with targeted `rg` and line-focused source reads. The relevant source paths are unchanged from the second pass; no `meta/network`, `meta/schema_version`, `SCHEMA_VERSION`, or recovered-genesis comparison exists.
- Automated tooling: none.
- Dynamic testing: none. The requested two-node mismatched-state run remains unperformed; the outcome below is a static code trace and the deployment/test-state inspection is static.

### Rechecked state and deployment behavior

`load_recovery_data` opens the database at `services/storage/src/recovery.rs:33-36`, while `RocksBackend::new` uses `create_if_missing(true)` at `services/storage/src/backends/rocksdb.rs:113-120` and writes no marker. The node calls this loader before starting services at `nodes/node/binary/src/lib.rs:161`.

The recoverable states remain network-bound: Cryptarchia stores `genesis_id`; mempool entries, Blend quotas/tokens, SDP declaration state, wallet state at a LIB, PoW tickets, and stored blocks/indexes all refer to a particular chain. The only explicit Blend adoption check remains epoch-number equality at `services/blend/src/core/mod.rs:690-706`; it does not identify a network, but the related informational tracker issue `#363` is already closed as low-value duplicate coverage.

The configured starting state is constructed in `services/chain/chain-service/src/states.rs:82-108`. When recovery exists, `initialize_cryptarchia` instead takes `genesis_id` and ledger state from that recovered object at `services/chain/chain-service/src/lib.rs:975-990`; there is still no comparison with the configured starting state. Recovery replay logs processing errors and continues at `services/chain/chain-service/src/lib.rs:1039-1056`.

The immutable slot index remains the only durable fallback identity examined by the marker design: `immutable_block/slot/0` is read by `find_immutable_genesis_block` at `services/chain/chain-service/src/sync/block_provider.rs:313-329`, and the index is produced from pruned immutable blocks at `services/chain/chain-service/src/service/mod.rs:1207-1222`. A recovery blob can exist before that index is written, leaving the pre-marker adoption hole tracked by `#364`.

Deployment inspection still distinguishes fresh and reusable paths. The local testing framework assigns each run a fresh scenario directory (`tests/testing_framework/src/framework/local/provisioning.rs:168-192`), and the fork-detector workflow uploads those run artifacts rather than reusing them. The container setup script deletes `state*` before copying a deployment file (`deployment/scripts/cleanup_node_data.sh:3-10`), but the systemd unit runs from a stable working directory and does not clean `./state` (`deployment/systemd/logos-blockchain-node.service:8-11`, `config/state.rs`). Replacing only the deployment configuration can therefore reuse an old directory; the genesis ceremony itself updates the deployment file and does not migrate or stamp node state.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| `63-LB-002` / `#402` | No network identity is stored in or checked against the state directory | Configuration | Medium | High | Open; re-verified |
| `181-LB-001` / `#524` | No schema version or migration path exists for persisted state | Denial of Service | Low | Low | Open; re-verified |
| `80-LB-002` / `#362` | Chain identity APIs report configured rather than resumed network identity | Auditing and Logging | Low | High | Open; re-verified |
| `80-LB-004` / `#364` | Some pre-marker directories have no identity that a future binary can recover | Configuration | Informational | — | Open; re-verified |

No additional finding is warranted. The earlier Blend observation is still visible in `services/blend/src/core/mod.rs`, but issue `#363` was closed as duplicate/low-value coverage of the state-marker problem.

### Existing finding `63-LB-002` / `#402` — No network identity is stored in or checked against the state directory

The current code still opens any existing RocksDB directory, loads all `recovery/*` entries, and lets Overwatch use a decoded recovery state in preference to `from_settings`. `CryptarchiaConsensusState::from_settings` derives the configured genesis at `states.rs:82-108`, but `initialize_cryptarchia` uses the recovered `genesis_id`, LIB, ledger state, and engine state at `lib.rs:975-990` without comparing them. The node can therefore resume network B while its deployment configuration and chain-id API describe network A.

**Exploit scenario:** an operator regenerates or replaces a deployment genesis and restarts a node without deleting its state directory. The node can remain on the old recovered tip, apply the new configuration around it, and connect to peers using the same protocol namespace. Static tracing predicts rejected cross-chain blocks or a private fork; a live two-node run was not performed in this pass.

**Recommendation:** implement the marker and fail-closed genesis comparison described in `#402`, using the genesis header ID as the network identity. Check it before any service consumes `RecoveryData`, and make a mismatch identify both the state directory and the two network identities. Keep this work coordinated with the schema and key-layout migrations in `#524`.

### Existing finding `181-LB-001` / `#524` — No schema version or migration path exists for persisted state

`RocksBackend::new` still writes no schema key. Each service decodes its recovery value through `State::from_bytes` at `services/storage/src/recovery.rs:98-109`; a changed positional `bincode` layout becomes a recovery error rather than a controlled migration or fallback. The startup path still has no directory-level dispatch point before service initialization. The recommended `meta/schema_version` key and migration registry remain absent.

**Impact:** a binary upgrade that changes a recovery struct can make an otherwise usable node fail startup until the operator deletes state and re-syncs. The same missing version prevents safely rolling out the block/transaction key-prefix migration identified by the storage reports.

**Recommendation:** use a fixed-format directory schema key, reject newer versions, and run explicit atomic migrations before service state is consumed. Preserve or deliberately invalidate each affected persisted type; do not silently interpret old bytes as the new layout.

### Existing finding `80-LB-002` / `#362` — Chain identity APIs report configured rather than resumed network identity

`run_node_from_config` captures `config.deployment.chain_id()` at `nodes/node/binary/src/lib.rs:147`, and the HTTP handler returns that injected value at `nodes/node/binary/src/api/handlers.rs:506-507`. The FFI function reads the same node-level value at `c-bindings/src/api/chain.rs:41-56`. None of these values is derived from the recovered `genesis_id` or a state-directory marker.

**Impact:** after wrong-network state reuse, `/chain/id` and `get_chain_id` can make monitoring or a wallet believe the node is on the configured network while the recovered consensus state is from another one.

**Recommendation:** make the API identity come from the same verified marker/recovered identity used at startup, or expose both the configured chain ID and verified genesis identity with unambiguous names. The startup refusal in `#402` is the primary control.

### Existing finding `80-LB-004` / `#364` — Some pre-marker directories have no recoverable identity

Before a marker exists, the proposed adoption design relies first on `recovery/cryptarchia` and otherwise on `immutable_block/slot/0`. The recovery timer is active in early chain phases, but the immutable slot index is populated only after immutable blocks are available. If a future binary changes the positional recovery layout before the marker is written, a young directory may contain data but neither identity can be decoded by the new binary.

**Impact:** a safe marker rollout can refuse such a directory and require a re-sync. This is an operational migration boundary, not a new attacker-controlled vulnerability.

**Recommendation:** ship the marker before any recovery-layout change, retain an explicit `Unidentifiable` outcome, and document when deleting the directory is safe. Keep the existing `#364` tracker item as the canonical record.

## 5. Suggestions (non-security)

### S-001 · Add a state-directory mismatch scenario to the integration test framework

Provision a node, persist at least one recovery interval and LIB/index entry, then restart it with a different genesis configuration and assert a clear refusal. Repeat with a pre-marker directory too young to expose `immutable_block/slot/0`. This directly closes the one dynamic item left unverified by the prior reports.

### S-002 · Close checklist issue #80 only after the canonical fixes are independently re-verified

The checklist issue currently has canonical downstream records `#402`, `#524`, `#362`, and `#364`. Its deliverable is complete as an audit record, but the implementation and any live mismatch test remain follow-up work.

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
