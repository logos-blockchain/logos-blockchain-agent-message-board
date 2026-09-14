# Audit Report — Storage schema versioning: every RocksDB value shares the wire codec with no format version

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/181`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `services/storage` (backend, chain API, recovery), `services/utils/src/overwatch/recovery`, `services/chain/chain-service/src/{states.rs,storage/}`, `services/tx-service/src/storage`, `services/api/src/http/storage`, `core/src/{codec,block,header,events}`, the recovery states of `chain-service`, `tx-service`, `wallet`, `sdp`, `pow`, `blend/core`; Overwatch `ae887f41f5a626c341179026ad7f03953ff2072e` (git dependency) for the load-failure path
Specifications: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd`
Date: `2026-09-12` — author: `claude-fable-5.1` — status: `final`

Source: report for #70 (PR #174, LB-003). Related and not repeated here: PR #131 (#80, state-directory network marker), PR #209 / #211 (recovery record size and `LedgerState` wire form), #79 (`expect` on bytes read back from RocksDB), #63 (block/transaction key collision), #180 (λSQL payload version), #197 (HTTP transaction lookup decodes bincode as JSON).

---

## 1. Summary

- Overall assessment: the node persists eleven kinds of value in one RocksDB. Nine of them are the bincode encoding of a live Rust type; none carries a format tag and the database has no schema key, so the only "version" is the binary that wrote the value. Between the two most recent release lines (`0.2.4`, 2026-09-04, and `0.3.0-rc.2`, 2026-09-02) seven of those nine types changed layout, an eighth retyped a field, and only the wallet state was untouched, so an in-place upgrade between them cannot start: the first service whose `recovery/*` entry fails to decode aborts the start sequence, and the process exits with a channel error that names neither the key nor the cause. The one compatibility measure in the tree, `#[serde(default)]` on `SdpState::pending_activity`, does nothing under bincode (verified empirically). The picture for blocks is better than PR #174 LB-003 stated: appended `Op` codes and appended event variants are readable by a newer binary, and the raw-id index keys never change; the fragile values are the six `recovery/*` blobs and the block-events value.
- Findings: 0 critical · 0 high · 0 medium · 3 low · 1 informational
- Key themes: "the wire type is the storage type", "compatibility assumptions that bincode does not honour", "a start failure with no name attached", "every automated path wipes state, so nothing in CI can see this"
- Must-fix before launch: none for launch itself, since every deployment path today starts from an empty state directory. Before the first post-launch release that changes any persisted type: LB-001 (schema key, refusal, migration hook), and the fixture test in §4 so that a layout change cannot land without a version bump.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/storage/src/api/backend/rocksdb/chain.rs` L27-L262 | every key prefix and the value written under it |
| `services/storage/src/backends/rocksdb.rs` L60-L78, L93-L128, L130-L167 | DB open (`create_if_missing`), prefix scan, `store`/`load` |
| `services/storage/src/recovery.rs` L22-L44, L98-L133 | `recovery/` prefix, load of all entries at startup, per-service `load_state`/`save_state` |
| `services/storage/src/lib.rs` L86-L102 | `StorageReplyReceiver::recv` `expect` |
| `services/utils/src/overwatch/recovery/{operators.rs,data.rs}` | `RecoveryOperator::try_load`, `RecoveryData::take` |
| Overwatch `ae887f4`: `overwatch/src/services/resources.rs` L186-L196, `overwatch/src/services/runner/service_runner.rs` L190-L204, L260-L262, `overwatch-derive/src/lib.rs` L505-L513 | what a `try_load` error does to the node |
| `nodes/node/binary/src/lib.rs` L156-L161, `main.rs` L89-L107, `config/storage/{mod.rs,serde.rs}`, `config/state.rs` | startup order, DB path, defaults |
| `core/src/codec/{mod.rs,bincode/mod.rs}` | the bincode options every stored value uses |
| `core/src/block/mod.rs` L95-L134, L235-L249, L366-L389; `core/src/header/mod.rs` L51-L115, L141-L143; `core/src/mantle/ops/op.rs` L74-L162; `core/src/events/mod.rs` L64-L174; `core/src/mantle/transactions/tx_list/signed_ops.rs` L360-L370 | how each stored type serialises, and what a reader does on the way back |
| `services/chain/chain-service/src/storage/adapters/storage.rs` L76-L91, L143-L187, L232-L251; `states.rs` L10-L30, L115-L119; `lib.rs` L581-L598, L643-L645, L750-L756, L899-L946; `service/mod.rs` L473-L483, L890-L900, L1163-L1194 | chain-service readers and the recovery replay |
| `services/tx-service/src/storage/adapters/rocksdb.rs` L49-L65, L91; `tx/state.rs` L9-L15; `backend/pool.rs` L54-L63; `tx/settings.rs` L4 | mempool items and pool recovery state |
| `services/api/src/http/storage/adapters/rocksdb.rs` L46-L83; `services/api/src/http/mantle.rs` L283, L360-L366, L453-L461, L755-L763, L801, L839 | HTTP readers |
| `services/wallet/src/lib.rs` L378, L1546-L1569, L1635; `states.rs` L199-L211 | wallet recovery state and block reads |
| `services/sdp/src/state.rs` L11-L18; `lib.rs` L76, L887-L896 | SDP recovery state |
| `services/pow/src/service.rs` L287-L307 | PoW recovery state |
| `services/blend/src/core/state.rs` L17-L27; `core/mod.rs` L290-L306; `core/settings.rs` L102 | blend core recovery state and its adoption rule |
| `deployment/{compose.setup.yml,scripts/cleanup_node_data.sh,systemd/logos-blockchain-node.service}`, `tests/testing_framework/src/framework/local/provisioning.rs` L936-L996 | whether any deployment path migrates or wipes |
| `git` history between tags `0.2.4`, `0.3.0-rc.2` and `a805329f8` for the persisted types | measured layout churn |

**Out of scope**
The λSQL SQLite file (`logos_sql/src/db.rs`), listed in the inventory for completeness and owned by #180. The block/transaction key collision and the `expect` on the HTTP block path (#63, #79). The state-directory network marker (PR #131, being re-verified under #80 in parallel; this report only says how the two mechanisms should share a namespace). The size and restore cost of the recovery record (#209, #211). The RocksDB SST format: `librocksdb-sys 0.17.3+10.4.2` reads the files its predecessors wrote, and that is assumed, not checked. The bincode crate itself (`1.3.3`) is assumed to behave as its source reads; the two behaviours this report depends on were also executed (Appendix C).

**Assumptions**
The operator upgrades a node by replacing the binary and restarting it on the same state directory, which is what `deployment/systemd/logos-blockchain-node.service` L13 does. Values in the database were written by an earlier build of this same code base, not by an attacker; a corrupt value is treated under #79. "Layout change" means a change to the byte sequence bincode produces for a type: a field added, removed, reordered or retyped, or an enum variant removed or reordered. A rename alone is not one.

## 3. Method

- Manual review of the in-scope paths, working through issue `#181` (all five checklist items) against parent `#10` and the storage parent `#15`.
- Specifications read: `bedrock-architecture-overview.md` and `overview-cryptoeconomics.md` in full (core), `network-wire-format.md` in full, `bedrock-v1.1-block-construction.md` §Canonical Encoding and §Block. The first three say nothing about storage; §Block says a block is "the unit that nodes store, execute and serve to their peers", which fixes what a stored block must preserve (header, signature, uncle headers, transactions) but not how. §Encoding and Decoding scopes bincode to "transport-level message framing". So storage is unspecified, and the findings below are against the node's own behaviour, not a spec.
- Read the bincode `1.3.3` deserializer (`src/de/mod.rs` L307-L316, L402-L411) to establish which type changes alter the encoding, then confirmed the two load-bearing claims by running them (Appendix C): a trailing `#[serde(default)]` field does not make an older blob readable, and an appended enum variant does.
- `git diff 0.2.4 0.3.0-rc.2` and `git log 0.3.0-rc.2..a805329f8` over the files that define persisted types, to measure how often the layouts actually change.
- Automated tooling: none beyond the Appendix C program (`cargo 1.98` toolchain, `bincode 1.3.3`, `serde 1`).
- Dynamic testing: none against the node. The start-failure path (LB-001) is a code trace through Overwatch; the exact text of the exit message is derived from `main.rs` L98-L104 and `overwatch-derive` L505-L513, not observed.

## 4. Findings

### 4.0 Inventory of persisted values

All keys live in one RocksDB opened at `<state.base_folder>/<storage.backend.folder_name>` (`./state/./db` by default, `config/state.rs` L14, `config/storage/serde.rs` L21-L26). The configured column family (`Some("blocks")` by default) is created but every read and write goes to the default column family (`backends/rocksdb.rs` L130-L136, L165-L167), as report #63 already noted. Every bincode value below is produced by the blanket `SerializeOp` (`core/src/codec/mod.rs` L22-L28) with the options at `core/src/codec/bincode/mod.rs` L23-L29: little-endian, fixed-width integers, no size limit, trailing bytes rejected.

| # | Key | Written by | Value type and encoding | Version tag | Read by → on decode failure |
|---|---|---|---|---|---|
| 1 | `<32-byte header id>` (no prefix) | `chain.rs` L53-L62 via `storage.rs` L101-L103 | `Block<SignedOps<Preverified, StandardMode>>` bincode (`block/mod.rs` L381-L389); `Op` fields are a length-prefixed byte string of the canonical op encoding (`op.rs` L74-L86) | the header carries `Version` (`header/mod.rs` L51-L53, L143) but no reader dispatches on it | chain-service `storage.rs` L84-L87: `try_into().ok()` → `None`, **no log**; recovery replay `lib.rs` L903-L907 → `HeaderIdNotFound` → falls back to LIB (`lib.rs` L932-L943); uncle selection `service/mod.rs` L473-L483 → logged, block proposed with fewer uncles; reorg re-insertion `service/mod.rs` L890-L900 → transactions silently absent; wallet `lib.rs` L1546-L1553 → `BlockNotFoundInStorage`; HTTP `api/.../rocksdb.rs` L46-L54 → `recv().expect` → process exit (#79); HTTP range endpoints `mantle.rs` L360-L366 skip with a warning, L755-L763 `flatten` silently, L453-L461 "canonical chain inconsistency" error |
| 2 | `<32-byte tx hash>` (no prefix, same key space as #1) | block transactions: `storage.rs` L210-L230 → `chain.rs` L191-L201; mempool items: `tx-service/.../rocksdb.rs` L49-L65 | `SignedOps<Preverified, StandardMode>` bincode (`generic_services/mod.rs` L53-L57) | none | chain-service `storage.rs` L247-L248 `filter_map(.ok())` → item dropped, **no log**; tx-service `rocksdb.rs` L91 same; HTTP `api/.../rocksdb.rs` L76-L83 decodes as JSON → always 500 (#197) |
| 3 | `block_parent/<id>` | `chain.rs` L55-L56, L62 | raw 32 bytes | none needed | `chain.rs` L100-L106 `try_into` → `Error::BlockHeaderError` → "Storage request failed" log, adapter `storage.rs` L137-L140 → `None` |
| 4 | `block_events/<id>` | `chain.rs` L57, L63 via `storage.rs` L105-L107 | `Events` bincode (`events/mod.rs` L168-L174): `Vec<Event>`, two enum levels | none | chain-service `storage.rs` L157-L160 → logged, `None`; wallet `lib.rs` L1555-L1569 → `Events::new()` (fail open; PR #206 LB-003); HTTP `handlers.rs` L1579-L1583 → 404 "Block not found" |
| 5 | `immutable_block/slot/<u64 BE>` | `chain.rs` L255-L262 | raw 32 bytes | none needed | `chain.rs` L137-L142 → error; scans yield a per-item error |
| 6 | `recovery/cryptarchia` | `recovery.rs` L122-L127 on every state update | `CryptarchiaConsensusState` bincode (`states.rs` L10-L30): tip, LIB, the full `LedgerState`, `UncleSlots`, `HashSet<HeaderId>`, `Option<LastEngineState>` | none | `recovery.rs` L106-L108 → `RecoveryError` → Overwatch `resources.rs` L195 → `service_runner.rs` L260-L262 → start sequence fails → **node exits** (LB-001) |
| 7 | `recovery/mempool` | same | `TxMempoolState<PoolRecoveryState<TxHash>, …>` (`tx/state.rs` L9-L15, `pool.rs` L54-L63): `IndexMap<Key, u64>`, `BTreeMap`, `u64` | none | same as #6 |
| 8 | `recovery/wallet` | same | wallet `RecoveryState` (`states.rs` L199-L211): voucher index, `Vouchers`, `Option<(HeaderId, WalletState)>`, `PendingClaims` | none | same as #6 |
| 9 | `recovery/sdp` | same, written by `update_state` (`sdp/lib.rs` L887-L896) on every declaration or activity change | `SdpState` (`sdp/state.rs` L11-L18): two `Option`s plus `#[serde(default)] pending_activity: Option<Activity>` | none; the `serde(default)` is inert (LB-002) | same as #6 |
| 10 | `recovery/pow` | same | `PoWServiceState` (`pow/service.rs` L299-L307): two `Vec<WinningTicket>` | none | same as #6 |
| 11 | `recovery/blend/core` | same | `RecoveryServiceState { service_state: Option<SerializableServiceState> }` (`blend/core/state.rs` L17-L27): epoch, quota, two hash collections of message types, `VecDeque<Vec<u8>>`, two token collectors | none | same as #6 at decode; after a successful decode an epoch mismatch is discarded with a log (`core/mod.rs` L290-L306), the only tolerant path |
| — | λSQL SQLite file | `logos_sql/src/db.rs` L37-L75 | six tables created `IF NOT EXISTS`, no `PRAGMA user_version`; the inscription payload carries `PAYLOAD_VERSION` (`protocol/mod.rs` L19, L284) | payload only | #180 |

**Which changes alter a stored value** (checklist item 2), from the bincode source and Appendix C:

| Change to the Rust type | Newer binary reads older value | Older binary reads newer value |
|---|---|---|
| Struct field appended, even with `#[serde(default)]` | fails: `deserialize_struct` visits `fields.len()` elements (`de/mod.rs` L402-L411) and hits end of input | fails: trailing bytes rejected |
| Struct field removed, reordered or retyped | fails or, worse, decodes into the wrong fields if sizes line up | same |
| Enum variant appended (derived serde: `u32` index) | reads | fails on the new index only |
| Enum variant removed or reordered | fails or mis-decodes | same |
| New `Op` code (`op.rs` L74-L108: opaque byte string with a 1-byte opcode) | reads | fails only on a block that contains the new op (`op.rs` L161) |
| New `Header::Version` byte (`header/mod.rs` L62-L74, L103-L115) | reads only if the header's field list is unchanged; the version byte selects nothing | fails on every block of the new version |
| `Vec`/map/`Option` contents change | as for the element type | same |
| Field rename | reads | reads |

Measured churn, so the table is not hypothetical. `git diff 0.2.4 0.3.0-rc.2` over the persisted types: `Block` gained `uncle_headers`; `LedgerState`'s epoch state gained `blend_pow_difficulty` and `previous_epoch_nonce`; `CryptarchiaConsensusState` gained `lib_block_uncle_slots`; `PoolRecoveryState::pending_items` went from `IndexSet<Key>` to `IndexMap<Key, u64>`; blend `SerializableServiceState` retyped `spent_core_quota` and `unsent_processed_messages` and gained `pending_transactions`; `TxEventPayload::Deposit::notes` changed element type (`BoundedInputs` → `UpperBoundedVec<DepositNote, …>`); `SdpState` gained `pending_activity`; `recovery/pow` did not exist. The transaction type retyped its `mantle_tx` field (`MantleTx` → `RawMantleTx`), a layout change unless the two serialise identically, which was not checked. The wallet's `WalletState` only renamed `locked_notes` to `service_notes`, which is not a layout change. The two raw-id key families (#3, #5) are unaffected by construction. Since `0.3.0-rc.2`, twelve more commits touched these files (`08205efec`, `471dbfc9e`, `b8c3c54ff` and others, 2026-09-03 to 2026-09-10).

**What hides this today** (checklist item 5). `deployment/compose.setup.yml` L12 runs `cleanup_node_data.sh`, whose L3 is `rm -rf /node-data/*/state*`, so every compose deployment starts empty. The testing framework provisions every node into `tempfile::tempdir()` (`provisioning.rs` L936, L957, L993). The systemd unit (`logos-blockchain-node.service` L13) restarts the same binary path on the same directory with no pre-start step, and no document in the repository mentions the state directory or upgrades. Nothing exercises an upgrade on a populated directory.

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | No schema version anywhere: an in-place upgrade across any persisted-layout change aborts node start, with no migration path and no named cause | Denial of Service | Low | Low | Open |
| LB-002 | `#[serde(default)]` on `SdpState::pending_activity` is inert under bincode, so the only compatibility measure in the tree does not work | Data Validation | Low | Low | Open |
| LB-003 | A block or transaction that fails to decode from the node's own database is reported as absent with no log line, on three read paths | Auditing and Logging | Low | Low | Open |
| LB-004 | `Header::Version` is stored but no storage reader dispatches on it, so two header versions cannot coexist in one database | Configuration | Informational | — | Open |

### LB-001 · No schema version anywhere: an in-place upgrade across any persisted-layout change aborts node start, with no migration path and no named cause

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `services/storage/src/backends/rocksdb.rs:L93-L128` (`RocksBackend::new`); `services/storage/src/recovery.rs:L98-L109` (`load_state`); `nodes/node/binary/src/lib.rs:L156-L161` (`load_recovery_data`, the only pre-service open); Overwatch `overwatch/src/services/resources.rs:L186-L196`, `overwatch/src/services/runner/service_runner.rs:L196-L204,L260-L262`; `nodes/node/binary/src/main.rs:L98-L104` |
| Status | Open |

**Description**

PR #174 LB-003 rated the class and stopped at "the block, state or item is silently absent or the node fails to recover". This finding pins down what actually happens and how often it will.

The database has no schema key: `RocksBackend::new` (L111-L116) opens with `create_if_missing(true)` and writes nothing, and `grep -rn "schema\|version" services/storage/src` returns nothing. Each service's recovery blob is loaded in `load_state` (`recovery.rs` L98-L109): a decode error becomes `RecoveryError::Backend(error.to_string())`, where the string is bincode's, for example `io error: unexpected end of file` or `Slice had bytes remaining after deserialization`. Overwatch turns that into `InitialStateError::Operator` (`resources.rs` L195) and explicitly does not fall back to `from_settings` (its own test `operator_load_error_does_not_fall_back_to_settings`, `resources.rs` L298-L299). `handle_start` returns the error as a string (`service_runner.rs` L260-L262), the runner logs `Failed to start service.` and `continue`s (L202-L203), which drops the `finished_signal_sender` without sending. The derive-generated `start` is awaiting that oneshot (`overwatch-derive/src/lib.rs` L505-L513), so it returns the receive error, `start_service_sequence` fails, and `main.rs` L98-L104 prints `Exiting... start_service_sequence failed: <RecvError>` and exits. The key that failed, the type, and the bincode message appear only in the tracing log at error level, and nothing says "schema mismatch".

Which values break at the next release is not a guess. Between `0.2.4` and `0.3.0-rc.2` seven of the nine bincode-encoded types changed layout (§4.0). A node that ran `0.2.4` and is restarted on `0.3.0-rc.2` with its state directory has at least `recovery/cryptarchia` (written on every block) and `recovery/mempool`; both are unreadable, so the chain service is the first to fail and the node exits. A node that had declared as a Blend provider also has `recovery/sdp` (LB-002). The stored blocks themselves also gained `uncle_headers`, so after a state wipe of only `recovery/*` the block store would be dead weight too: every `get_block` returns `None` silently (LB-003).

`read_only: true` (`config/storage/serde.rs` L23) makes even a future migration impossible without a separate refusal, and the sequence in `main.rs` gives no opportunity to run one: `load_recovery_data` (`lib.rs` L161) is the only place the DB is opened before services start, which is also where the check belongs.

**Exploit scenario**

Not attacker-triggered. Operators run `0.2.4` under the systemd unit. A release replaces the binary; systemd restarts it on the same directory. The chain service's `try_load` fails on `recovery/cryptarchia`, the node prints a channel error and exits, systemd restarts it, and the loop repeats until the operator deletes the state directory and re-syncs from peers. Every Blend provider that upgrades this way is offline for the duration; a coordinated upgrade takes the mix network down until enough operators find the log line. The same holds for any later release that touches `LedgerState`, which has changed in each of the last two release lines.

**Recommendation**

- *Short term* (one PR, storage crate plus the startup hook):
  1. A `meta/schema_version` key holding a fixed 4-byte little-endian `u32`, deliberately not a bincode struct so it is decodable by every future binary. Written in the same `WriteBatch` that first populates a new database.
  2. `open_and_migrate` called from `load_recovery_data` (`nodes/node/binary/src/lib.rs` L161), before any service exists: missing key on an empty DB → write `CURRENT`; missing key on a non-empty DB → treat as version 0; `v < CURRENT` → run each registered migration `v → v+1` inside one `WriteBatch` that ends with the version bump, so a crash mid-way leaves the old version; `v > CURRENT` → refuse with `database schema {v} is newer than this binary ({CURRENT}); upgrade the node or restore the state directory from backup`; `read_only && v != CURRENT` → refuse with the same shape of message. The message must name the key, both versions, and the remedy, and go to stderr as well as the log.
  3. Migration `0 → 1`, `drop-unversioned-recovery-state`: delete every `recovery/*` entry. That is exactly what operators do by hand today, made explicit, atomic and logged; blocks and indexes are kept. The chain service then starts from `StartingState` in the settings and re-syncs, as it does on a fresh directory.
  4. A byte-exact fixture test per persisted type (§ Test plan) so that a layout change fails CI unless `CURRENT` is bumped and a migration is added.
- *Long term*:
  - Separate the storage type from the live type for the values that change most. `CryptarchiaConsensusState` should be written as a `StoredConsensusStateV{N}` with explicit fields and `From` conversions, and `LedgerState` should get the same treatment together with the size work in #211 (that issue already lists "decide how an old recovery record is detected and migrated" as its last item; this report supplies the mechanism).
  - Keep the previous version's storage types as frozen `v{N}` modules, so a migration `N → N+1` is `decode as v{N}, convert, encode as v{N+1}` instead of "drop and re-sync". Do this first for `recovery/cryptarchia`, where re-sync is the expensive alternative.
  - Fold the two other pending storage changes into the same registry: the key-prefix change from report #63 (`block/`, `tx/`) as migration `1 → 2`, and the network marker from PR #131 as a second `meta/` key checked in the same `open_and_migrate`. PR #131 proposed carrying version and marker inside the recovery blobs; a `meta/` key is preferable because it also covers blocks, events and transactions, and stays readable whatever the blobs look like.
  - Add a "reuse state directory across a binary swap" mode to the testing framework so an upgrade on a populated directory is exercised in CI.

**Diff sketch** (new file `services/storage/src/schema.rs`, wiring shown for the startup hook only):

```rust
pub const SCHEMA_VERSION: u32 = 1;
const SCHEMA_KEY: &[u8] = b"meta/schema_version";

pub struct Migration {
    pub from: u32,
    pub name: &'static str,
    pub run: fn(&DB, &mut WriteBatch) -> Result<(), SchemaError>,
}

pub const MIGRATIONS: &[Migration] = &[Migration {
    from: 0,
    name: "drop-unversioned-recovery-state",
    run: |db, batch| {
        for item in db.iterator(IteratorMode::From(b"recovery/", Direction::Forward)) {
            let (key, _) = item?;
            if !key.starts_with(b"recovery/") { break; }
            batch.delete(key);
        }
        Ok(())
    },
}];

pub fn open_and_migrate(settings: &RocksBackendSettings) -> Result<RocksBackend, SchemaError> {
    let backend = RocksBackend::new(settings.clone())?;
    let stored = match backend.rocks.get(SCHEMA_KEY)? {
        Some(v) => u32::from_le_bytes(v.as_slice().try_into().map_err(|_| SchemaError::Corrupt)?),
        None if backend.is_empty()? => { backend.rocks.put(SCHEMA_KEY, SCHEMA_VERSION.to_le_bytes())?; SCHEMA_VERSION }
        None => 0,
    };
    if stored > SCHEMA_VERSION {
        return Err(SchemaError::Newer { stored, binary: SCHEMA_VERSION });
    }
    if stored < SCHEMA_VERSION && settings.read_only {
        return Err(SchemaError::ReadOnlyNeedsMigration { stored, binary: SCHEMA_VERSION });
    }
    for version in stored..SCHEMA_VERSION {
        let migration = MIGRATIONS.iter().find(|m| m.from == version).ok_or(SchemaError::NoMigration(version))?;
        tracing::warn!(target: LOG_TARGET, from = version, to = version + 1, name = migration.name, "migrating storage schema");
        let mut batch = WriteBatch::default();
        (migration.run)(&backend.rocks, &mut batch)?;
        batch.put(SCHEMA_KEY, (version + 1).to_le_bytes());
        backend.rocks.write(batch)?;
    }
    Ok(backend)
}
```

`load_recovery_data` (`nodes/node/binary/src/lib.rs` L161) becomes `let backend = open_and_migrate(&storage_config)?; let recovery_data = recovery_data_from_backend(&backend)?;`, and the `?` surfaces `SchemaError`'s `Display` through the existing `eyre!("Error encountered: {}", e)` at L255, which already reaches stderr. Note that migrations which re-encode blocks must not go through `Block::try_from(Bytes)`: that path re-runs header signature, size and body-root checks (`block/mod.rs` L366-L379, L235-L249) and, through `SignedOps<Preverified>::deserialize` (`signed_ops.rs` L360-L370), transaction pre-verification, for every block (S-001).

**Test plan**

1. `schema::tests`: a fresh directory gets `SCHEMA_VERSION`; a directory containing a `recovery/cryptarchia` entry and one block but no key is migrated to 1 with the block intact and the entry gone; a key of `SCHEMA_VERSION + 1` is refused with the expected message; `read_only` on a version-0 directory is refused; a migration that returns an error leaves the key unchanged.
2. Fixtures under `services/storage/fixtures/v1/`: one encoded value per row of §4.0 (#1, #2, #4, #6 to #11), generated once by a `#[ignore]`d test and checked in. A test decodes each with the current type and compares against a checked-in `Debug` snapshot. Any layout change fails here, and the fix is to bump `SCHEMA_VERSION`, add a migration, and regenerate under `v2/`.
3. The testing framework: start a node, produce a few blocks, stop it, restart the same state directory with a binary whose `SCHEMA_VERSION` is one higher and whose migration is the drop, and assert that the node comes up and re-syncs to the same tip.
4. Negative: the same restart with the version-newer binary first and the older one second must exit with the "newer than this binary" message, verified from stderr.

**References**: PR #174 LB-003 (rated the class); PR #131 LB-001 (no version in recovery blobs, migration hook missing); #211 item 4; `network-wire-format.md` §Encoding and Decoding (bincode is for transport framing).

### LB-002 · `#[serde(default)]` on `SdpState::pending_activity` is inert under bincode, so the only compatibility measure in the tree does not work

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Data Validation |
| Target | `services/sdp/src/state.rs:L11-L18` (`SdpState`), added in `e5f880c8a` (2026-09-01, "persist/restore pending active message", #3453); `core/src/codec/bincode/mod.rs:L23-L29` (the options); bincode `1.3.3` `src/de/mod.rs:L307-L316,L402-L411` |
| Status | Open |

**Description**

Commit `e5f880c8a` appended `#[serde(default)] pub pending_activity: Option<Activity>` to `SdpState` (`state.rs` L15-L17). The attribute is the standard way to add a field compatibly, and it works for self-describing formats. Under the node's bincode options it does nothing: `deserialize_struct` delegates to `deserialize_tuple(fields.len(), …)` (`de/mod.rs` L402-L411), whose `SeqAccess` yields exactly `fields.len()` elements and only returns `None` once that count is reached (L307-L316), so `serde`'s `default` arm is never taken; the third element is read from the end of the old value and fails with `io error: unexpected end of file`. Appendix C reproduces this with the same options: a two-field blob fails to decode as the three-field struct, and the three-field blob fails to decode as the two-field struct with `Slice had bytes remaining`.

`recovery/sdp` is written by `update_state` (`sdp/lib.rs` L887-L896) whenever the declaration id or the tracked activity changes, so every node that has ever declared a service has one. `0.2.4` does not contain the field (`git show 0.2.4:services/sdp/src/state.rs` has no `pending_activity`); `0.3.0-rc.1` and later do. On the upgrade the SDP service's `try_load` fails and the node exits as in LB-001, and the failure is specifically on the nodes the network depends on: Blend providers.

The same misconception will recur, because nothing in the code base says which serde attributes are honoured by the storage codec. `TxMempoolState` uses `#[serde(skip)]` on a `PhantomData` (`tx/state.rs` L13-L14), which is fine, but a reviewer seeing both will assume `default` is fine too.

**Exploit scenario**

Not attacker-triggered. A Blend provider on `0.2.4` with a live declaration restarts on `0.3.0-rc.2`. `load_state` for `recovery/sdp` returns `Backend("io error: unexpected end of file")`, the SDP service fails to start, the node exits.

**Recommendation**
- *Short term*: remove the attribute, since it promises something the codec cannot deliver, and rely on LB-001's migration to drop the old entry. Add a comment on `lb_core::codec::bincode::OPTIONS` stating that stored and wire types must not use `serde(default)`, `skip_serializing_if`, `flatten` or `untagged` to evolve, and that a field change is a schema change.
- *Long term*: the fixture test in LB-001 makes this class of change fail CI; a clippy-style lint over `#[serde(default)]` inside types that implement `ServiceState` would catch it earlier, but the fixture test is enough.

**References**: Appendix C; `e5f880c8a`; bincode `1.3.3` source.

### LB-003 · A block or transaction that fails to decode from the node's own database is reported as absent with no log line, on three read paths

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Auditing and Logging |
| Target | `services/chain/chain-service/src/storage/adapters/storage.rs:L84-L87` (`get_block`), `:L247-L248` (`get_transactions`); `services/tx-service/src/storage/adapters/rocksdb.rs:L91` (`get_items`) |
| Status | Open |

**Description**

`get_block` (L84-L87) does `let block = maybe_block?; block.try_into().ok()`: a value that exists but does not decode becomes `None`, indistinguishable from a missing key, and nothing is logged. The same `filter_map(… .ok())` shape drops undecodable transactions in the chain-service adapter (L247-L248) and in the mempool adapter (L91). By contrast `get_block_events` (L157-L160) logs the conversion failure, `remove_block` (L182-L184) returns an error string, and the backend logs RocksDB errors (`chain.rs` L224-L229).

The consumers then act on "absent". The recovery replay treats it as `HeaderIdNotFound` and falls back to LIB with a warning that names the tip and LIB but not the cause (`lib.rs` L932-L943). Uncle selection proposes with fewer uncles (`service/mod.rs` L473-L483, logged as "not found in storage"). Reorg handling re-inserts no transactions from a reorged block (`service/mod.rs` L890-L900, no log). The HTTP range endpoints skip or flatten (`mantle.rs` L360-L366, L755-L763) and the canonical-chain walk reports an "inconsistency" (L453-L461). The mempool loses pending items on restart with no trace (#203 covers the read-back semantics). PR #174 LB-003 stated the outcome ("the block does not exist as far as the chain service is concerned"); this finding is about the missing evidence: after LB-001's schema check, a decode failure means either corruption or a bug, and both deserve a log line with the key and the error.

**Exploit scenario**

Operational, not adversarial. A node whose block store was written by an older layout (LB-001's second half, if only `recovery/*` is cleared by hand) serves no blocks over sync and re-downloads its own chain, and the only symptom is a bootstrap that takes as long as a fresh one.

**Recommendation**
- *Short term*: log at `error` with the key and the decode error in the three sites, and return `Result<Option<_>, _>` from `get_block` so callers can tell "absent" from "unreadable"; #79 and #203 ask for the same distinction on the `expect` and mempool paths.
- *Long term*: once LB-001 exists, a decode failure on a versioned database should increment a metric (`storage_request_failed` already exists, `storage/src/lib.rs` L199-L203) so an operator dashboard shows it.

**References**: PR #174 LB-003; #79; #203; PR #206 LB-003 (wallet fails open to `Events::new()`).

### LB-004 · `Header::Version` is stored but no storage reader dispatches on it, so two header versions cannot coexist in one database

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Configuration |
| Target | `core/src/header/mod.rs:L51-L53,L62-L74,L103-L115,L141-L143`; `core/src/block/mod.rs:L104-L134` (`RawBlock` deserialisation) |
| Status | Open |

**Description**

`Header` carries `version: Version` as its first field (L143) and rejects any byte other than `BEDROCK_VERSION` on the way in (L62-L74, L113). But `Header` derives `Deserialize` (L141), so the field list is fixed by the current struct, and `Block`'s custom `Deserialize` (`block/mod.rs` L104-L134) reads a `RawBlock` with the current `Header` before it looks at anything. A future `Version::V2` header with a different field set cannot be stored next to v1 blocks in one database, because the reader has no way to choose a layout by version: it either decodes every block as v2 or none. PR #174 covered this for the network side (surface 1, "a new header version"); on the storage side the consequence is that a header upgrade is also a full-database migration, and it should be planned as one (LB-001's registry is where it goes). No change is needed now; the note is here so the header version is not mistaken for storage versioning.

**Recommendation**
- *Short term*: none.
- *Long term*: when a second header version is introduced, give the stored block an explicit envelope `(version, bytes)` or key it by version, and migrate.

**References**: PR #174 (surface 1 and LB-003); `bedrock-v1.1-block-construction.md` §Canonical Encoding (`Version = Byte ; fixed to 0x01`).

## 5. Suggestions (non-security)

### S-001 · Every read of a stored block re-runs proposal validation

`Block::try_from(Bytes)` (`block/mod.rs` L366-L379) calls `into_verified` (L235-L249): header signature check, total-size check and body-root recomputation over every transaction. Inside it, each transaction is a `SignedOps<Preverified, StandardMode>` whose `Deserialize` runs `preverify()` (`signed_ops.rs` L360-L370; PR #196 LB-003 has the detail). So `get_block` on the node's own database pays the cost of validating a proposal it already validated when it stored it, on the wallet's LIB scans (`wallet/lib.rs` L1635), the HTTP range endpoints (`mantle.rs` L755-L763), reorg handling and, relevant here, any migration that touches blocks. #199 proposes moving `preverify` out of `Deserialize`; storage reads should go through an unverified decode (the `RawBlock` shape already exists at `block/mod.rs` L112-L117) and trust the database, or verify once per key and cache.

### S-002 · The recovery values have no owner

Six services write under `recovery/` with a suffix each (`recovery.rs` L25-L30) and the node loads the whole prefix into a `HashMap` before any service starts (`recovery.rs` L33-L44, `data.rs` L26-L31). There is no list of suffixes, no test that two services do not share one, and no place a new service is required to register. A `const RECOVERY_SUFFIXES: &[&[u8]]` next to `RECOVERY_PREFIX`, with a test that every `StorageRecoverySettings` impl in the node's service set is in it, would make LB-001's fixture list and migration `0 → 1` complete by construction.

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

## Appendix B — Checked and ruled out

- **Raw-id keys are version-proof.** `block_parent/*` and `immutable_block/slot/*` store 32 raw bytes (`chain.rs` L56, L260); a length mismatch is a typed error (`chain.rs` L104, L140), not a mis-decode.
- **Appending an `Op` code or an event variant does not break old data.** `Op` is stored as an opaque byte string with its own opcode (`op.rs` L74-L86); derived enums encode a `u32` index. Verified for the enum case in Appendix C. PR #174 LB-003's "a new `Op` variant … every stored block then fails" holds only in the old-binary-reads-new-block direction.
- **A stale blend recovery state is tolerated.** `core/mod.rs` L290-L306 discards a state whose epoch does not match instead of panicking; this is a semantic check after a successful decode, not a layout check, so it does not help with LB-001.
- **Pruning survives a decode failure.** `remove_block` deletes the three keys before the adapter tries to convert the returned bytes (`chain.rs` L82-L93, then `storage.rs` L182-L184), so an undecodable stale block is deleted, the error is logged (`service/mod.rs` L1185-L1191), the id stays in `storage_blocks_to_remove` for one more round, and the next round sees `Ok(None)` and drops it.
- **`RejectTrailing` is on** (`bincode/mod.rs` L28), so an old binary cannot silently accept a longer new value; both directions fail loudly rather than mis-decode, except where field sizes happen to line up (a removed `u64` followed by an added `u64`), which no current change did.
- **The recovery blob is not the source of truth for the chain.** If only `recovery/*` is dropped, blocks remain, `StartingState` from settings is used, and the node re-syncs; that is why migration `0 → 1` in LB-001 can be a drop rather than a converter.
- **No fixture, compatibility or schema test exists** in `services/storage`, `tests/`, or the persisting services (`grep -rln "fixture\|compat\|schema"` finds only an unrelated cucumber step file).
- **λSQL** keeps its own SQLite file with `IF NOT EXISTS` tables and no `user_version`; its payload version and legacy readers are #180's.

## Appendix C — bincode behaviour, executed

A 60-line program (`bincode = "=1.3.3"`, `serde` derive) using the same options as `lb_core::codec::bincode::OPTIONS` (little-endian, no limit, fixint, reject trailing), run on 2026-09-12:

| Case | Result |
|---|---|
| 2-field struct value decoded as the same struct plus a trailing `#[serde(default)] Option` field (the `SdpState` change) | `Err("io error: unexpected end of file")` |
| Same with both original fields `None` | `Err("io error: unexpected end of file")` |
| 3-field value decoded as the 2-field struct (old binary, new value) | `Err("Slice had bytes remaining after deserialization")` |
| Enum value `Tx(3)` decoded as the same enum with a third variant appended | `Ok(Tx(3))` |
| New third variant decoded as the two-variant enum | `Err("invalid value: integer `2`, expected variant index 0 <= i < 2")` |

The program is not part of the node tree; it depends only on the two crates named and reproduces from the description above.
