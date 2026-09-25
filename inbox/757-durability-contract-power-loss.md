# Audit Report · Durability contract for persisted service state: what must survive a power loss, what the RocksDB write path actually guarantees, and a test that shows the difference

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/757`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `c4c86be18c58b5b09c3650e93871c8cfb624885b` · component(s): `services/storage` (`rocksdb/mod.rs`, `recovery.rs`, `api/mod.rs`, `lib.rs`), `services/utils/src/overwatch/recovery`, the six recovery states (`chain-service`, `tx-service`, `sdp`, `wallet`, `blend/core`, `pow`), `services/chain/chain-leader/src/leadership.rs`, `ledger/src/mantle/{leader.rs,pow/mod.rs}`, `nodes/node/binary/src/cli/{keys.rs,participate.rs,config/*.rs}`, `logos_sql/src/db.rs` (for contrast); Overwatch @ `8f06c685948620d5d9931ed8f5bdb86e6cbffcd9` (`services/state`)
Specs: `https://github.com/logos-co/logos-lips` @ `d788723992a805b395f377de6e6cf59859b47168` · read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md` (both in full); by section: `cryptarchia-v1-bootstr-sync.md`, `cryptarchia-v1-protocol.md`, `proof-of-work.md`, `blend-protocol.md`, `proof-of-quota.md`, `bedrock-anonymous-leaders-reward.md`, `bedrock-service-declaration-protocol.md` (sections listed in Method)
Date: `2026-09-25` · author: `Claude Code (research agent)` · status: `final`

Parent: #15 (storage). Source: #81 LB-003 (`inbox/81-rocksdb-options-durability-blocking-calls.md`), which asked for the contract to be decided. Related and cited, not repeated: #181 (schema versioning), #188 / #210 / #211 (recovery record size, envelope split, `LedgerState` wire form), #403 (63-LB-003, storage writes without a reply channel), #695 (250-LB-002, Blend quota spend persisted after publish), #636 (PoW claim batches, report in `inbox/`), #639 LB-003 (voucher index persisted after the header is signed, report in `inbox/`), #404 (63-LB-004, keystore file permissions), #648 (secrets at rest outside the keystore).

---

## 1. Summary

- Overall assessment: nothing has changed since #81 LB-003 on the write path (`RocksBackend::new` and `store` are byte-identical at `c4c86be1`), and the test in Appendix B shows what that means. The node survives a process kill with every acknowledged write intact, and every block batch is atomic. An acknowledged write only becomes power-loss durable when RocksDB next syncs a file. With the node's two column families, that happens at the next memtable flush (every 64 MiB written) or when the kernel writes back dirty pages. In the strict power-loss model (only fsynced bytes survive), 50 of 50 acknowledged rounds were lost when no flush had happened, 162 of 800 were lost after one flush, and a clean `drop` of the DB changed nothing. Writing the recovery keys with `WriteOptions::set_sync(true)` lost nothing, at 150 to 250 µs per write on this disk. What survives is always a prefix, so the services stay consistent with each other; they just go back in time. For most persisted state that is harmless, because it can be re-derived from the chain or from peers. It is not harmless for four things whose effect is already public, or already spent, before they are written: the wallet's voucher index, the Blend core quota index, the Blend-to-SDP activity hand-off, and PoW tickets. The same holds for the keystore file, which the CLI rewrites in place with no fsync.
- Findings: `0` critical · `0` high · `0` medium · `4` low · `0` informational
- Key themes: "acknowledged is not durable, and nothing in the node can ask for durable", "effects published before the state that makes them unique is persisted", "the only irreplaceable file is written with truncate-then-write"
- Must-fix before launch: LB-004 (atomic, fsynced keystore writes; cheap and prevents permanent key loss). LB-002 and LB-003 should be fixed together with #695 and #639 LB-003 through one mechanism (S-001: an awaited, synced store). LB-001 is the recorded contract (Section 4.2) plus that mechanism.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/storage/src/rocksdb/mod.rs` | `RocksBackend::new` (L340-375), `store` (L377-383), `bulk_store` (L385-410), `execute` (L538-544), `store_block_data` (L120-145), `remove_block` (L147-166), `store_immutable_block_ids` (L189-202), `remove_transactions` (L311-325) |
| `services/storage/src/recovery.rs`, `api/mod.rs:72-76`, `api/mod.rs:193-199` | recovery prefix (L26), startup load (L36-46), `save_state` (L111-125), `StorageApi::store` returns once the message is queued |
| `services/utils/src/overwatch/recovery/operators.rs:61-66`; Overwatch `services/state/handle.rs:82-110`, `updater.rs:37-41` | how a service's state reaches the storage task (watch channel, no acknowledgement) |
| `services/chain/chain-service/src/{lib.rs:592, states.rs:10-30, service/mod.rs:199-236, 798-808, bootstrap/state.rs:11-54, service/phases/pbp.rs:66-67}` | block write, recovery record, offline-duration timestamp |
| `services/pow/src/service.rs:319-345, 614-628, 1063-1152, 1217-1287`; `tickets.rs:53-63, 215-245` | ticket state, the random claim key, publish-then-persist |
| `services/wallet/src/{states.rs:198-296, 405-421, lib.rs:377, 1217-1239}`; `services/chain/chain-leader/src/leadership.rs:121-137`; `ledger/src/mantle/leader.rs:105-140` | voucher index and voucher set |
| `services/blend/src/core/{state.rs:21-29, 259-264, mod.rs:756-787, 1880-1928, 2440-2467, 2679-2697}`; `blend/provers/src/crypto/core_and_leader/send.rs:101-122` | quota index, token collectors, activity-proof hand-off |
| `services/sdp/src/{state.rs:11-40, lib.rs:63-80, 168-190, 386-456, 566-601, 608-723, 885-896}` | declaration id, nonce, pending activity |
| `services/tx-service/src/{tx/settings.rs:4, tx/state.rs:9-15, tx/service.rs:487-499, storage/adapters/rocksdb.rs:45-50}` | mempool snapshot and items |
| `nodes/node/binary/src/cli/{keys.rs:215-230, config/init.rs:37-62, config/migrate.rs:39, config/update.rs:58, config/keystore.rs:61-68, 128-173, participate.rs:63-65}` | every non-RocksDB file the node's binary writes |
| `logos_sql/src/db.rs:265-273` | the one other database in the workspace (not linked by the node binary) |
| librocksdb-sys `0.17.3+10.4.2` `include/rocksdb/options.h`, `db/db_impl/db_impl_compaction_flush.cc:168-172`, `db/db_impl/db_impl.cc:470-476` | option defaults, closed-WAL sync on flush, shutdown flush |

**Out of scope**

- Recovery record size, write rate and restart cost: #188, #210, #211. Schema versioning: #181. RocksDB open-file limits, compression and blocking calls: #81 LB-001, LB-002, LB-004.
- Whether Overwatch's orderly shutdown (SIGINT via `services/system-sig`, `KillSignal=SIGINT` in the shipped unit) hands every service's last state to the storage service before the storage service stops. Not traced; proposed as a follow-up.
- A real power cut. The test emulates one (Appendix B); the kernel may write back more than the emulation keeps, never less. Measuring the practical window on ext4 or XFS needs `dm-log-writes` or a VM hard reset, which this shared machine does not allow; proposed as a follow-up.
- Third-party crates assumed correct: `rocksdb 0.24.0`, `librocksdb-sys 0.17.3+10.4.2`, `tokio`, Overwatch, `serde_yaml`, `rusqlite`.

**Assumptions**

- "Power loss" means kernel crash, power cut or hypervisor reset: the process, the page cache and any unsynced device cache are gone. "Process crash" means SIGKILL, OOM kill or panic with `panic = abort`: the page cache survives.
- Linux defaults: `vm.dirty_expire_centisecs = 3000`, `vm.dirty_writeback_centisecs = 500` (read on this machine), ext4 with `data=ordered`.
- Block interval 1 s slots and `f = 1/30` as in #188 / #210; `WINDOW = 300` slots for PoW tickets (`proof-of-work.md` § Acceptance Window).

## 3. Method

- Manual review, working through issue #757: every `RECOVERY_KEY_SUFFIX` implementation (`grep -rn RECOVERY_KEY_SUFFIX`: six, listed in 4.1), every raw key family in `rocksdb/mod.rs`, and every file write in non-test code of the node binary and services (`grep` for `fs::write`, `File::create`, `OpenOptions`, `sync_all`, `sync_data`, `rename`, `persist(`, `rusqlite`, `sled`, `to_writer`). For each item: who writes it, what it holds, whether it can be re-derived from the chain or from peers, and what a lost tail or a torn pair of keys does. The three reports the issue names (#81, #181, #188) and the two reports for #210 and #211 were read first. #636, #639, #695, #404 and #648 were found by searching the tracker and prior reports, so as not to duplicate them.
- Specifications. Storage itself has no specification (parent #15 says so), so there is no area spec to read in full. The two core overviews were read in full. The following were read by section, for the state a node must keep: `cryptarchia-v1-bootstr-sync.md` § Constants, § Setting the Fork Choice Rule, § Offline Grace Period, § Offline Duration Measurement; `cryptarchia-v1-protocol.md` § Latest Immutable Block; `proof-of-work.md` § Introduction, § Overview, § Protocol, § Parameters, § Puzzle Target, § Reward Pool, § Acceptance Window, § Reward Difficulty; `blend-protocol.md` § Quota (Core Quota to Quota Application), § Proof of Quota, § Relaying (details), § Processing (details), § Failure Detection and Reaction, § Rewarding (Epoch Randomness to Rewarding Distribution Logic); `proof-of-quota.md` § Constraints (Steps 1 to 5); `bedrock-anonymous-leaders-reward.md` § Overview, § Protocol, § Details; `bedrock-service-declaration-protocol.md` § Message Timing, § Declaration Storage, § Identifier Uniqueness, § Active Message, § Withdraw Message, § Protocol (Declare, Active, Withdraw).
- Tooling: `cargo 1.98.1` / `rustc 1.98.1` (pinned toolchain), release profile, shared target dir; `strace 6.x`; Python 3 for the driver.
- Dynamic testing (Appendix B), in a scratch clone of `c4c86be1`, with no change to node code; one new integration test file `services/storage/tests/durability_757.rs` and a driver script:
  - (a) process kill: a child process writes rounds through the node's own path (`RocksBackend::new` with the node-default settings, one `store_block_data` of a 100 KiB block, then one `store` per recovery key), prints `ACK i` after each round, and is SIGKILLed after the parent has read `ACK 300`. The DB is reopened and checked.
  - (b) emulated power loss: the same writer runs under `strace -f -y`. For every file the driver records the length it had at its last `fsync`/`fdatasync`. A copy of the DB is then cut back to those lengths (written but never synced files are emptied) and reopened with `RocksBackend::new`. That is exactly what POSIX guarantees after a power loss. Four variants: 50 rounds of 4 KiB blocks (no memtable flush) and 800 rounds of 100 KiB blocks (one flush), each with the recovery keys written as the node does (`sync = false`) or with `WriteOptions::set_sync(true)` through the node's own `DB` handle. One more variant ends with a clean `drop` of the DB instead of an abort.
  - (c) latency of a 200 B put, unsynced and synced, 300 each.
  - (d) the keystore write path (`std::fs::write`) reproduced in a 15-line program under `RLIMIT_FSIZE` (standing in for a full disk) and under `strace`.
- Not run: the full node, since the brief rules out building the node binary; a real power cut (see Out of scope).

## 4. Findings

### 4.0 Settings in effect (re-verified at `c4c86be1`)

`RocksBackend::new` (`rocksdb/mod.rs:340-375`) builds `Options::default()` and sets only `create_if_missing` and `create_missing_column_families`. Every write goes through `DB::put` (`:382`) or `DB::write` (`:140, :161, :197, :319, :406`), which rust-rocksdb forwards with `WriteOptions::default()`. The `OPTIONS-*` file of the test DB (Appendix B.3) and the RocksDB v10.4.2 headers give:

| Setting | Value in effect | Where | What it means here |
|---|---|---|---|
| `WriteOptions::sync` | `false` on every write | `options.h:2109` | a write returns once it is in the WAL's page-cache pages |
| `WriteOptions::disableWAL` | `false` | `options.h:2117` | WAL on; a process crash loses nothing that was acknowledged (measured, B.2 a) |
| `manual_wal_flush` | `false` | `options.h:1379`, OPTIONS file | each write group is handed to the kernel with `write(2)` before the call returns |
| `use_fsync` | `false` | `options.h:812` | syncs use `fdatasync` |
| `bytes_per_sync`, `wal_bytes_per_sync` | `0`, `0` | `options.h:1146, 1156` | no background `sync_file_range`; not a durability guarantee even if set |
| `wal_recovery_mode` | `kPointInTimeRecovery` | `options.h:1312` | replay stops at the first missing or torn WAL record: what survives is a prefix of the write sequence |
| `atomic_flush` | `false` | `options.h:1409` | irrelevant while the WAL is on; only the default column family is written |
| column families | `default` + `blocks` (created, never written, #63 LB-006) | `config/storage/serde.rs:19-26` | because there are two, RocksDB `fdatasync`s every closed WAL before each memtable flush (`db_impl_compaction_flush.cc:168-172`, seen in B.2 b): a flush is also a durability point |
| `write_buffer_size` | 64 MiB | OPTIONS file | durability point every 64 MiB written, if nothing else syncs |
| `avoid_flush_during_shutdown` | `false` | `options.h:1352` | the shutdown flush runs only for data written with the WAL disabled (`db_impl.cc:470-476`); a clean close with the WAL on syncs nothing (measured, B.2 b, clean-close row) |
| `track_and_verify_wals_in_manifest` | `false` | OPTIONS file | a missing WAL tail is not detected as corruption; it is silently truncated |

No code in the workspace calls `set_sync`, `flush_wal`, `sync_wal` or `DB::flush`.

### 4.1 Inventory of persisted state

One RocksDB (`<state>/db`), default column family, plus files outside it. "Lost tail" means that the last seconds of acknowledged writes are missing after a power loss. Because recovery is point-in-time, all keys roll back to the same point in the single WAL (measured: every recovery key ended at the same round, B.2 b). "Torn" means two related writes of which only the earlier one survived.

| # | Key / file | Writer | Holds | Re-derivable? | Lost tail | Torn pair |
|---|---|---|---|---|---|---|
| 1 | `<header id>`, `block_parent/<id>`, `block_events/<id>`, `immutable_block/slot/<u64 BE>` | chain service, `process_block` → `store_block_data` (`service/mod.rs:798-808`, awaited; one `WriteBatch`, `rocksdb/mod.rs:134-140`) | block bytes, parent link, events, immutable index | Yes: blocks from peers, the rest by re-applying | blocks re-downloaded; nothing lost for the chain, since finality is the network's | block and index cannot tear (one batch; the test asserts it on every round). Block durable, record not: the normal case after a loss (measured: 638 blocks, records at 637). Blocks above the record's tip are orphans (#63 LB-005) and are re-applied from peers. Record ahead of its block: impossible, since the block is awaited before the record is queued (`service/mod.rs:798-808`, then `:233`) and the WAL is a prefix |
| 2 | `<tx hash>` (mempool items) | tx-service via `store_transaction` → `bulk_store` (`api/mod.rs:193-199`, `rocksdb/mod.rs:385-410`) | pending transaction bytes | Yes: gossiped at acceptance (`tx/service.rs:487-499`); peers and the submitter hold them | pending txs forgotten | item without its key in `recovery/mempool` or the reverse: an item nobody lists is dead weight; a listed key with no item is dropped on load (#181 LB-003 describes the silent drop) |
| 3 | `recovery/mempool` | tx-service operator (`tx/settings.rs:4`, `tx/state.rs:9-15`) | pool snapshot (keys, order) | Yes | as #2 | as #2 |
| 4 | `recovery/cryptarchia` | chain service after every block (`service/mod.rs:233`) and on the one-minute timer (#210 LB-002) | tip, LIB, LIB `LedgerState`, uncle slots, blocks to delete, `last_engine_state` (engine state + timestamp) | Yes, from stored blocks and genesis (absent: `from_settings` and IBD from genesis; older: replay of `(lib, tip]`) | older tip and LIB, replayed. Older timestamp: the measured offline time is longer, which only ever chooses the Bootstrap rule sooner (`bootstrap/state.rs:30-54`); that is the safe direction for § Offline Duration Measurement. A lost Bootstrapping → Online switch (`pbp.rs:66-67`) restarts under the Bootstrap rule for another Prolonged Bootstrap Period (24 h): liveness only. `B_imm` moves back by the blocks of the lost tail (§ Latest Immutable Block says it never should). Exploiting that needs a competing chain from within those few blocks below the old LIB, i.e. about `k` deep; not practical | block/record, as #1 |
| 5 | `recovery/wallet` | wallet (`states.rs:198-209`, `405-421`; `lib.rs:377`) | `next_new_voucher_index`, the voucher set, the `WalletState` at LIB, pending claim reservations | `WalletState`: yes, rebuilt from the LIB ledger when absent (`states.rs:258-268`). Voucher index and set: **not by the code**; the secrets are `zkhash(master_key, index)` and could be found by scanning indices against the chain, but nothing does | voucher index rewinds: the next proposal reuses a commitment already in a public header (**LB-002**). Lost reservations: a second claim for a voucher already in flight is built and rejected on chain, harmless | voucher index (wallet) vs the block that carries its commitment (chain, gossiped): **LB-002** |
| 6 | `recovery/sdp` | SDP (`state.rs:11-18`, `lib.rs:885-896`) | declaration id, `updated`, `pending_activity` | Declaration id: from config or `POST` (`lib.rs:177-180`, `798-833`). The nonce is not persisted locally and is read from the ledger (`lib.rs:386-456`). Pending activity: **no**, once Blend has dropped its token collector | declaration id forgotten until the operator sets it again (no active messages meanwhile). Pending activity lost | Blend `recovery/blend/core` (collector cleared) vs `recovery/sdp` (activity recorded): **LB-003** |
| 7 | `recovery/blend/core` | Blend core after every release round and epoch event (`state.rs:21-29`, `mod.rs:2467`) | epoch, `spent_core_quota`, unsent messages, pending transactions, current and old token collectors | Quota index: no. Tokens: no (they are the reward-lottery inputs, `blend-protocol.md` § Activity Proof). Unsent messages: no, but losing them is only a missed release | quota index rewinds and the node reuses PoQ key nullifiers, which peers drop as duplicates (#695, window widened from milliseconds to the writeback window by LB-001). Tokens of the tail lost: marginally lower lottery chance | quota index vs messages already on the wire (#695); collector vs SDP (**LB-003**) |
| 8 | `recovery/pow` | PoW (`service.rs:319-345`) on every mined ticket (`:614-628`), claim and settlement | `ready_to_claim` and `pending_to_claim`, each ticket carrying a fresh random secret key (`tickets.rs:53-63, 215-245`) | **No**: the key is random and exists only here; a ticket is claimable for `WINDOW = 300` slots (§ Acceptance Window) | tickets mined in the tail are lost with their reward (bounded by the node's mining rate × the tail). `pending_to_claim` is informational: never resubmitted, only retired or window-pruned (`:1254-1287`) | claim published (`:1117`) before the state moves the ticket to pending (`:1134-1142`, persisted later): after a rewind the ticket is back in `ready_to_claim`, the next batch re-claims a spent nullifier and the whole transaction is rejected. #636 LB-001 already recommends filtering against `pow.nullifiers()` (§3.4 step 3, "or a restart"); not re-filed |
| 9 | column family `blocks` | created by `RocksBackend::new`, never written | nothing | n/a | n/a | its only effect is the closed-WAL sync on flush noted in 4.0 |
| 10 | `keystore.yaml`, `user_config.yaml` | CLI only: `config init` (`init.rs:57-61`), `keys generate/add/remove` (`keys.rs:215-230`), `config migrate/update` (user config only, `migrate.rs:39`, `update.rs:58`) | every secret key: stake, leader and SDP funding, voucher master, PoW claim, Blend, network (`keystore.rs:24-40`, generated from `OsRng`, `:128-173`) | **No** | new key lost; with the in-place rewrite, the old keys too (**LB-004**) | user config names a key that the keystore does not hold (written first, `keys.rs:224` then `:227`) |
| 11 | `participation_data.yaml` | CLI `participate` (`participate.rs:63-65`) | public keys for the genesis ceremony | Yes, from the keystore | none that matters | n/a |
| 12 | λSQL `LIB.db`, `LIVE.db`, `control.db` | `logos_sql`, a zone library not linked by the node binary | zone SQL state | n/a | n/a | already durable: `journal_mode = WAL`, `synchronous = FULL` for writers (`db.rs:265-268`); the rebuild DB uses `synchronous = OFF` (`:270-273`) |

Services that persist nothing: KMS (keys are loaded from the keystore at start; voucher and PoQ secrets are derived), network (the identity comes from the keystore), time, API. The chain service's in-memory ledger states are rebuilt from the record and the blocks.

### 4.2 The contract (the choice #757 asks for)

Recorded here so that LB-001's recommendation has something concrete to implement:

1. **Process-crash durable, the default for everything.** Every acknowledged RocksDB write survives SIGKILL, OOM and panic. That already holds (B.2 a) and needs no change. It is enough for every family that is re-derivable: #1 to #4, `WalletState` and pending claims in #5, the declaration id in #6, token collectors and unsent messages in #7, #11.
2. **Power-loss durable and ordered before the effect, for state whose effect is public or spent before it is written.** The state must be synced to disk *before* the effect leaves the node, and a service must be able to wait for that:
   - wallet voucher index (#5): before the commitment is returned to the leader (LB-002);
   - Blend core quota index (#7): before a message using the index is published (#695);
   - SDP pending activity (#6): before Blend drops the token collector it came from (LB-003);
   - PoW winning tickets (#8): synced when mined, since the key exists nowhere else. The claim transition should be written before publishing, or reconciled against `pow.nullifiers()` as #636 recommends.
   For the two counters, "reserve ahead" is cheaper than a sync per use: persist `index + R` synced, hand out indices from memory, and on restart start from the persisted reserve. The cost is at most `R` unused indices per crash, never a reuse.
3. **Optional: power-loss durable for one chain-record write.** The Bootstrapping → Online switch (`pbp.rs:66-67`), so that a power loss does not cost another 24 h under the Bootstrap rule. That is one synced write per node lifetime. The per-block envelope needs no sync: its loss is replayed, and the timestamp errs safe.
4. **Files outside RocksDB that hold secrets** are written as temp file, `fsync`, `rename`, `fsync` of the directory (LB-004).
5. **No reliance on `bytes_per_sync` / `wal_bytes_per_sync` / memtable flushes** for any of the above. They bound the loss statistically but guarantee nothing, and the flush interval grows after #210's split (a 177 B envelope instead of a whole ledger state means 64 MiB of writes takes 64 or more blocks, over 30 minutes at full 1 MiB blocks, instead of 2).

Cost of a synced write on this machine: 150 to 250 µs, against 2.5 to 3 µs unsynced (B.2 c). The items in point 2 are written a few times per epoch (vouchers, activity), once per release round at most (quota, or once per reserve with the reserve scheme), and once per mined ticket, so the total is negligible. A synced write also makes every earlier WAL record durable, including the block batches before it (B.2 b, sync rows: all 800 blocks survived although only the recovery writes were synced).

### 4.3 Findings table

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Acknowledged writes are not power-loss durable and nothing in the node can ask for durability: under emulated power loss every write since the last memtable flush is lost (50 of 50 rounds, 162 of 800), and the recovery path has no acknowledgement to wait on | Configuration | Low | High | Open |
| LB-002 | The wallet voucher index rolls back on power loss after the commitment is already in a gossiped header, so the next proposal publishes the same `voucher_cm` again: two blocks become linkable to one leader and one reward is forfeited | Privacy / Anonymity | Low | High | Open |
| LB-003 | The activity-proof hand-off from Blend to SDP crosses two recovery keys with no ordering: Blend's cleared collector reaches disk before SDP records the pending activity, so a crash or power loss in between loses that epoch's Blend reward | Economic / Incentive | Low | High | Open |
| LB-004 | The CLI rewrites `keystore.yaml` in place (`O_TRUNC`, no fsync, no rename): a full disk, a kill or a power loss during `keys add/generate/remove` or `config init` destroys the only copy of every secret key | Configuration | Low | High | Open |

### LB-001 · Acknowledged writes are not power-loss durable and nothing in the node can ask for durability: under emulated power loss every write since the last memtable flush is lost (50 of 50 rounds, 162 of 800), and the recovery path has no acknowledgement to wait on

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Configuration |
| Target | `services/storage/src/rocksdb/mod.rs:340-375` (`RocksBackend::new`), `:377-383` (`store`), `:134-140` (`store_block_data` batch), `:385-410` (`bulk_store`); `services/storage/src/api/mod.rs:72-76` (`StorageApi::store` returns after `relay.send`); `services/storage/src/recovery.rs:111-125` (`save_state`); `services/utils/src/overwatch/recovery/operators.rs:61-66`; Overwatch `services/state/updater.rs:37-41` |
| Status | Open (re-verifies #81 LB-003 at `c4c86be1`: no line of the storage crate, the recovery operator or the persisting services' state files changed since `3088316`; `git diff --stat` is empty) |

**Description**

#81 LB-003 derived the durability of the write path from the RocksDB headers and a syscall count. This finding measures it and adds the part that makes it hard to fix locally.

What an acknowledged write means (Section 4.0): it is in the WAL's page-cache pages. It becomes power-loss durable only when RocksDB syncs a file, and nothing in the node asks for that. RocksDB does it on its own in two places: at open (MANIFEST, CURRENT, OPTIONS), and before every memtable flush. The flush sync comes from the node's second, never-written column family, which makes RocksDB `fdatasync` every closed WAL before flushing (`db_impl_compaction_flush.cc:168-172`). A clean close adds nothing (`db_impl.cc:470-476`: the shutdown flush is only for data written with the WAL disabled). Between those points, durability depends on kernel writeback, which is a timing behaviour (30 s dirty expiry, 5 s ext4 commit), not a guarantee.

Measured (Appendix B.2), node write path, strict power-loss model (only fsynced bytes survive):

| Run | Acknowledged rounds | Survived process kill / abort | Survived emulated power loss |
|---|---|---|---|
| 50 rounds, 4 KiB blocks, no flush, `sync = false` | 50 | 50 | **0** (WAL `000004.log` 301,659 B, never synced) |
| same, ending with a clean `drop` of the DB | 50 | 50 | **0** |
| 800 rounds, 100 KiB blocks, one memtable flush, `sync = false` | 800 | 800 | **638 blocks, 637 of every recovery key** (the flush synced the closed WAL; 16.9 MB of the new WAL was never synced, and the half-written SST was discarded) |
| 50 rounds, recovery keys with `set_sync(true)` | 50 | 50 | 50 |
| 800 rounds, recovery keys with `set_sync(true)` | 800 | 800 | 800 |
| SIGKILL after `ACK 300` (no strace) | 300 read | all 300, every key at 300 | n/a |

Three properties hold in every run and are what make a contract possible: block batches never tore (the verifier asserts block, parent and index together on every round), the surviving state is a prefix, and every recovery key stopped at the same round. A power loss therefore takes the node back in time consistently.

The node cannot adopt the contract of Section 4.2 as it stands, for a structural reason. No service can learn that its state is on disk, let alone synced. A service hands its state to `StateUpdater::update`, which is a `watch::Sender::send` (`updater.rs:37-41`). The operator runs later on its own task and coalesces (#188). `save_state` awaits `StorageApi::store`, which returns once the message is in the relay (`api/mod.rs:72-76`, #403). The storage task writes it later with `sync = false`. Every service that must persist before acting (LB-002, LB-003, #695, the PoW tickets of 4.1 #8) has no primitive to do so.

**Exploit scenario**

Not attacker-triggered. A node loses power, or its VM is reset, some seconds after it proposed a block, spent Blend quota, handed an activity proof to SDP or mined a PoW ticket. It restarts from a state older than what it had already made public. What that costs is family-specific (LB-002, LB-003, #695, 4.1 #8). For every other family it costs only replay and re-download (4.1). On a young node, or after #210's envelope split, the RocksDB-side durability point can be far behind (64 MiB of writes). The loss window is then whatever the kernel had not yet written back, and under this emulation every write since the last flush.

**Recommendation**

- *Short term*: adopt the contract of Section 4.2 and write it down next to `RocksBackend::new`. Give the storage API one durable write, `StorageMsg::Store { key, value, durability: Durability::{Default, Synced}, reply }`, which the storage task executes with `WriteOptions::set_sync(synced)` and answers after the write (this is #403's reply channel plus one flag, S-001). Give `RecoveryBackend` / the services an awaited `persist(state).await` that bypasses the watch channel for the four items of 4.2 point 2. Use it there only. Leave block batches and the per-block envelope unsynced.
- *Long term*: keep the harness of Appendix B as an `#[ignore]` test (S-002), so that the contract is re-checked when the storage layer changes. Drop the unused `blocks` column family (#63 LB-006) only together with an explicit decision on flush-time WAL syncs, since it is what currently syncs closed WALs.

**References**: #81 LB-003; #403; #188 (coalescing); #210 (envelope split lengthens the flush interval); RocksDB v10.4.2 `options.h:1146-1156, 1312, 1352, 1379, 1409, 2093-2118`, `db_impl_compaction_flush.cc:168-172`, `db_impl.cc:470-476`; `cryptarchia-v1-bootstr-sync.md` § Offline Duration Measurement (the only persisted value the specifications explicitly ask for; its loss errs safe, 4.1 #4).

### LB-002 · The wallet voucher index rolls back on power loss after the commitment is already in a gossiped header, so the next proposal publishes the same `voucher_cm` again: two blocks become linkable to one leader and one reward is forfeited

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Privacy / Anonymity |
| Target | `services/wallet/src/states.rs:287-296` (`add_next_known_voucher`: increment, then `update_state`), `:405-421` (the `StateUpdater` write); `services/wallet/src/lib.rs:1217-1239` (`generate_new_voucher_secret` returns the commitment); `services/chain/chain-leader/src/leadership.rs:121-137` (commitment into the leader proof, then the block is published); `ledger/src/mantle/leader.rs:105-106, 134-140` (`add_voucher` pushes into the MMR with no duplicate check) |
| Status | Open (extends #639 LB-003, same mechanism; this finding adds the power-loss window and the linkability consequence) |

**Description**

The voucher secret is `zkhash(master_key, index)` (#639, #85) with `index = next_new_voucher_index`. `add_next_known_voucher` increments the index in memory and queues the recovery state. `generate_new_voucher_secret` returns the commitment to the leader, which proves and publishes the block. #639 LB-003 described the process-crash window: the operator's write lands milliseconds later. Under power loss (LB-001) the window is the unsynced tail, seconds to minutes. A leader that proposed a block in that tail restarts with the index one or more behind. Its block is not lost: it was gossiped, and the node re-syncs it from peers.

On its next win it derives the same secret again. The same `voucher_cm` then appears in a second block header. Three consequences follow:

1. **Linkability.** `bedrock-anonymous-leaders-reward.md` § Voucher creation and inclusion step 1 requires "a one-time random secret", and § Unlinking Block Rewards from Proposals relies on the commitment revealing nothing about the producer. Two headers carrying the same commitment are publicly linkable to one leader by anyone who reads the chain. That leaks the kind of information the stake-inference analyses treat as sensitive (two wins by the same stake).
2. **Forfeited reward.** Both leaves share one nullifier; only one `LEADER_CLAIM` can succeed (#639 LB-003).
3. **A permanently unclaimable leaf.** `add_voucher` pushes the duplicate into the MMR without a check (`leader.rs:134-140`). The unclaimable leaf stays in `|voucher_cm| − |voucher_nf|`, the denominator of every later share (§ Leaders Reward). The effect is small, but it is permanent.

The wallet does not detect the rewind. It never compares the commitments of indices at or above `next_new_voucher_index` with the chain's voucher set, and `vouchers` is not re-derived on start (4.1 #5).

**Exploit scenario**

A validator with enough stake to win a few slots an hour runs on a VM that is hard-reset (host maintenance, power cut) 20 s after it proposed block `B1`. The recovery record for the wallet was still in the page cache. The node restarts, re-syncs `B1` from peers, and wins again an hour later: `B2` carries `B1`'s `voucher_cm`. Any observer linking the two headers learns that `B1` and `B2` share a leader. One of the two rewards can never be claimed. No attacker is needed, and none can force the reset at the right moment, hence the High difficulty.

**Recommendation**

- *Short term*: persist the index before returning the commitment, synced, with the durable write of LB-001. Or, cheaper, reserve ahead: persist `next_new_voucher_index + R` synced, allocate from memory, and on restart continue from the persisted reserve, wasting at most `R` indices per crash. Independently, on start, derive the commitments for indices `next..next+R` and skip any that are already in the chain's voucher set. That also repairs a rewind that already happened.
- *Long term*: as #639 LB-003 suggests, bind the derivation to something the block fixes (slot, parent), so that a reused index cannot produce an equal commitment. Reject a duplicate `voucher_cm` in header validation as defence in depth (a consensus change; raise it upstream).

**References**: #639 LB-003; #85 (asked whether the index "never rewinds"); `bedrock-anonymous-leaders-reward.md` § Voucher creation and inclusion, § Leaders Reward, § Unlinking Block Rewards from Proposals; LB-001.

### LB-003 · The activity-proof hand-off from Blend to SDP crosses two recovery keys with no ordering: Blend's cleared collector reaches disk before SDP records the pending activity, so a crash or power loss in between loses that epoch's Blend reward

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Economic / Incentive |
| Target | `services/blend/src/core/mod.rs:1880-1901` (`complete_transition_period`: clear the old collector, submit, `commit_changes`), `:756-761` (the same at start-up), `:1917-1928`, `:2679-2697` (`submit_activity_proof` is a relay `send`); `services/sdp/src/lib.rs:566-601` (`handle_post_activity`: build and post the transaction, then `update_state`), `:608-723` (`submit_activity`) |
| Status | Open |

**Description**

At the end of the transition period, Blend takes the old epoch's token collector out of its state and computes the activity proof. It sends the proof to SDP as a relay message and commits its state with the collector gone (`mod.rs:1897-1900`). SDP receives the message and first builds the active-message transaction: a ledger lookup and a wallet-built, ZK-signed transaction, `lib.rs:608-700`. It then posts the transaction to the mempool, and only after that records `pending_activity` with `update_state` (`lib.rs:600`). The `IntentTracker` resubmits it until it is included (`lib.rs:334-384`).

The two writes travel through independent watch channels and operators into the same WAL. Blend's arrives first, since SDP is still building a transaction. Between the two, the proof exists only in SDP's memory. The token collector it came from is gone from Blend's persisted state, and Blend does not reload it.

A process crash in that window (seconds: the ZK signature) or a power loss whose tail covers it (LB-001) loses the proof for good, unless the transaction had already reached the mempool and been gossiped. `blend-protocol.md` § Active Message requires the message for epoch `e` to be included during epoch `e+1`, and § Rewarding Distribution Logic step 4 says "If a node does not send the Active Message on time, then it will not receive a reward". The operator loses the whole epoch's Blend reward (60 % of block rewards shared among active nodes, `overview-cryptoeconomics.md` § Blend Service and Consensus Leaders).

**Exploit scenario**

Operational: a Blend provider is restarted by a deploy (`systemctl restart` sends SIGINT; whether the orderly shutdown saves SDP's in-flight state is the follow-up in Section 2), crashes, or loses power in the few seconds after the transition period of epoch `e+1` ends. On restart, Blend has no old collector and SDP has no pending activity. No active message for epoch `e` is ever sent, and the epoch's reward is forfeited. The window is once per epoch per node, so the rate is low but not zero.

**Recommendation**

- *Short term*: make the hand-off acknowledged. SDP persists the received `Activity` with a synced, awaited write (LB-001's primitive) before building the transaction, and replies to Blend. Blend clears the collector only after the reply. Alternatively, keep the old collector in Blend's state until SDP reports that the activity is tracked or included.
- *Long term*: treat every cross-service hand-off of unrecoverable data as a two-phase transfer (the receiver persists, then the sender forgets), and list those hand-offs next to the contract of Section 4.2.

**References**: `blend-protocol.md` § Active Message, § Rewarding Distribution Logic; `bedrock-service-declaration-protocol.md` § Active, § Message Timing; #278 LB-003 (the same start-up path, timing only); LB-001.

### LB-004 · The CLI rewrites `keystore.yaml` in place (`O_TRUNC`, no fsync, no rename): a full disk, a kill or a power loss during `keys add/generate/remove` or `config init` destroys the only copy of every secret key

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Configuration |
| Target | `nodes/node/binary/src/cli/keys.rs:215-230` (`persist_user_config_and_keystore`, used by generate/add/remove at `:269`, `:323`, `:362`); `nodes/node/binary/src/cli/config/init.rs:57-61`; `config/migrate.rs:39`, `config/update.rs:58` (user config only) |
| Status | Open |

**Description**

`keystore.yaml` holds every secret key the node has. Stake, leader and SDP funding, voucher master, PoW claim, Blend signing and ZK, and the network identity are all generated from `OsRng` (`keystore.rs:24-40, 128-173`). Nothing else holds them. Every write of the file is `std::fs::write(path, yaml)`, which is `open(O_WRONLY|O_CREAT|O_TRUNC)` followed by `write` and `close`, with no `fsync`, no temporary file and no `rename` (B.4, strace). The old content is gone at `open`. The new content is in the page cache until writeback. So:

- **Full disk (ENOSPC) or any write error:** the old keystore is truncated, the new one is partial, and the command returns an error. Reproduced with `RLIMIT_FSIZE` standing in for ENOSPC (B.4): an 8,205 B keystore became 4,096 B of the new content, which is invalid YAML. A node whose disk filled with chain data (#81 LB-001) is exactly where an operator runs `keys add`.
- **Kill or crash during the write:** same result.
- **Power loss within the writeback window after any of these commands:** the file can be empty or partial after reboot (ext4's `auto_da_alloc` heuristic starts writeback on close for truncate-and-rewrite, which narrows the window but does not close it; XFS has no such heuristic).
- **Torn pair:** `user_config.yaml` is written first (`keys.rs:224`) and names the new key, then the keystore (`:227`). If the second write fails, the config refers to a key that no longer exists. The node then fails to start, and the operator may "fix" it by running `config init --overwrite`, which generates fresh keys.

`config init` has the same shape for a keystore it has just generated. The window is short, but a power cut right after `init`, before the operator has copied the keys anywhere, loses stake keys that `participate` (`participate.rs:63-65`) may already have published to the genesis ceremony.

**Exploit scenario**

An operator whose disk is full runs `keys add --title Extra --zk <hex>` to import one more key. `std::fs::write` truncates `keystore.yaml` and fails with ENOSPC after the first page. The command prints an I/O error. The stake, funding and voucher-master keys are gone. Without a backup, the stake and every unclaimed voucher are unrecoverable. No attacker is involved, and a backup mitigates it, hence Low / High. The impact, loss of funds for that operator, is the most severe of any durability failure in this report.

**Recommendation**

- *Short term*: write both files through one helper: create `<path>.tmp` in the same directory with mode `0600` (also closes #404's permission issue), write, `sync_all`, `rename` over the target, `fsync` the directory. Write the keystore before the user config, so that a failure leaves an unused key rather than a dangling reference. Refuse to proceed if the existing keystore fails to parse, instead of letting a later command overwrite it.
- *Long term*: keep the previous keystore as `keystore.yaml.bak` on every rewrite, and document backup of the keystore as part of `config init` output.

**References**: #404 (63-LB-004, keystore permissions); #648 (secrets at rest outside the keystore: `recovery/pow` holds each ticket's secret key, 4.1 #8); `rename(2)`, `fsync(2)`.

## 5. Suggestions (non-security)

### S-001 · One durable-write primitive for the storage API, shared by #403, #695, #639 LB-003 and LB-001 to LB-003 here

Target: `services/storage/src/api/{mod.rs,requests.rs}`, `services/storage/src/rocksdb/handlers.rs`, `services/storage/src/recovery.rs:111-125`, `services/utils/src/overwatch/recovery/operators.rs`.

Four reports now ask for the same thing in different words: a store that reports its result (#403), a quota spend persisted before publishing (#695), a voucher index persisted before the header (#639), and the contract here. One `Store { key, value, synced: bool, reply: oneshot::Sender<Result<(), _>> }` and one awaited `persist_now(state)` on the recovery backend cover all of them. Batching the synced recovery writes of one release round or one block into a single `WriteBatch` keeps the cost to one `fdatasync` (150 to 250 µs here).

### S-002 · Keep the power-loss emulation as a test

Target: `services/storage/tests/` (new). The harness in Appendix B needs `strace` and ran in 10 s. A lighter permanent form, with no strace, is a unit test asserting that the keys of Section 4.2 point 2 are written with `sync = true`: wrap `WriteOptions` creation in one function and count calls. A second test SIGKILLs a child and checks that every acknowledged write is present (B.2 a).

### S-003 · Specifications: say which local state must survive a crash

`proof-of-quota.md` § Constraints Step 1 requires an index "not already used", and `bedrock-anonymous-leaders-reward.md` § Voucher creation and inclusion requires "a one-time random secret". Both are persistence requirements across restarts for any implementation that derives the value from a counter, as this node does for both. The specs could state it, and allow the "skip ahead by a reserve on restart" rule of Section 4.2. `cryptarchia-v1-bootstr-sync.md` § Offline Duration Measurement could state the safety direction: the recorded time must never be later than the node's last Online time. With that rule, losing recent timestamps is harmless and needs no sync, and an implementation that writes the timestamp ahead of time is wrong.

## 6. Checked and ruled out

- **Block batch atomicity.** Block, parent link, events and immutable index are one `WriteBatch` (`rocksdb/mod.rs:134-140`). The verifier asserted all-or-nothing on every round of every run, including the emulated power losses. `remove_block` deletes its three keys in one batch (`:156-161`).
- **Record ahead of its block.** The chain service awaits `store_block_data` before queueing the record (`service/mod.rs:798-808`, then `:233`), and the WAL is replayed as a prefix. Measured: after the torn run, blocks reached 638 and every record 637, never the reverse.
- **Cross-service consistency after a loss.** All six recovery keys ended at the same round in every run. There is one WAL and one storage task, and `bulk_store`'s `spawn_blocking` is awaited by that task (`rocksdb/mod.rs:393-409`). The node never restarts with, say, a wallet newer than the chain.
- **DB opens after emulated power loss.** In all four emulated runs `RocksBackend::new` succeeded under `kPointInTimeRecovery` with the unused second column family. The half-written SST of the flush in progress was discarded because the MANIFEST did not reference it yet. No "column family inconsistency" error.
- **Offline Duration Measurement.** A lost timestamp makes the measured offline time longer, which can only select the Bootstrap rule sooner (`bootstrap/state.rs:30-54`, including the clock-error branch). Conforms to § Offline Duration Measurement and § Setting the Fork Choice Rule without a sync.
- **`B_imm` regression.** After a loss, `B_imm` is the recorded LIB, a few blocks below the LIB the node had reached. § Latest Immutable Block says blocks deeper than `B_imm` are never reorganized. After the restart, the blocks in the lost range are reorganizable again, but only by a competing chain forking about `k` blocks deep. Not practical; no finding.
- **SDP nonce.** Not persisted locally. Read from the ledger at every submission (`sdp/lib.rs:386-456, 634`), so it cannot rewind (`bedrock-service-declaration-protocol.md` § Active Message: "must increase monotonically").
- **PoW claim reward.** The claim and the transfer of its reward note to the configured claim address are in one transaction (`service.rs:1392-1499`, `build_reward_claim_tx` and `transfer_ops`). Losing local state after settlement strands nothing: the reward is already at a keystore address. The reverse torn case (ticket back in `ready_to_claim`) is #636's.
- **Wallet notes.** A missing or older `WalletState` is rebuilt from the LIB ledger state (`states.rs:258-268`); owned notes are chain state.
- **Other files.** No node service writes files at run time (grep over `services/*/src`: no `std::fs` or `tokio::fs` outside tests). The λSQL database is not linked by the node binary, and its writers already use `synchronous = FULL`.
- **Shutdown flush.** `KillSignal=SIGINT` (`deployment/systemd/logos-blockchain-node.service`) reaches `services/system-sig` (`lib.rs:55-66`), which calls `overwatch_handle.shutdown()`. Overwatch's state handle forwards the last state on its fuse signal (`handle.rs:104-108`). Whether the storage service is still running to write it was not traced (Section 2). Either way, a clean RocksDB close does not sync the WAL (B.2 b), so an orderly stop followed by a power cut within the writeback window loses the same tail.

---

## Appendix A · Definitions

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

## Appendix B · Experiments

### B.1 Setup and commands

Scratch clone of `c4c86be18c58b5b09c3650e93871c8cfb624885b`; no node source file changed. Added: `services/storage/tests/durability_757.rs` (B.5) and `audit757.py` (B.6). Built and run on a shared 4-core x86-64 VM, ext4, `vm.dirty_expire_centisecs = 3000`:

```sh
CARGO_TARGET_DIR=<shared> CARGO_INCREMENTAL=0 \
  cargo test --release -p logos-blockchain-storage-service --test durability_757 --no-run
python3 audit757.py <target>/release/deps/durability_757-<hash> <workdir>
```

The emulated power loss is the strictest POSIX model: for every file, only the bytes present at its last `fsync`/`fdatasync` survive, and a file written but never synced is emptied. Real hardware keeps at least that much, and usually more, depending on writeback timing. The model does not reorder writes within a synced range and does not model device caches that ignore flushes.

### B.2 Raw output

(a) process kill, (b) emulated power loss (five variants), (c) latency. Paths shortened; the test harness's own `running 1 test` lines removed. The run was repeated three times with identical survival counts; the synced latency was 154.1, 252.4 and 152.4 µs, the unsynced 2.5, 3.0 and 2.6 µs.

```
process-kill: SIGKILL after reading ACK 300
  VERIFY blocks_prefix=300 blocks_after_gap=0 cryptarchia=300 mempool=300 sdp=300 wallet=300 blend/core=300 pow=300
power-loss[small-nosync]: rounds=50 block=4KiB sync=none acked=50 fsync+fdatasync calls=11
  process-crash view (dir as left by abort): VERIFY blocks_prefix=50 blocks_after_gap=0 cryptarchia=50 mempool=50 sdp=50 wallet=50 blend/core=50 pow=50
  truncated: 000004.log:301659->0
  power-loss view (only fsynced bytes kept):  VERIFY blocks_prefix=0 blocks_after_gap=0 cryptarchia=0 mempool=0 sdp=0 wallet=0 blend/core=0 pow=0
power-loss[large-nosync]: rounds=800 block=100KiB sync=none acked=800 fsync+fdatasync calls=14
  process-crash view (dir as left by abort): VERIFY blocks_prefix=800 blocks_after_gap=0 cryptarchia=800 mempool=800 sdp=800 wallet=800 blend/core=800 pow=800
  truncated: 000008.log:16907631->0, 000009.sst:65649018->0
  power-loss view (only fsynced bytes kept):  VERIFY blocks_prefix=638 blocks_after_gap=0 cryptarchia=637 mempool=637 sdp=637 wallet=637 blend/core=637 pow=637
power-loss[small-syncrec]: rounds=50 block=4KiB sync=recovery acked=50 fsync+fdatasync calls=312
  process-crash view (dir as left by abort): VERIFY blocks_prefix=50 blocks_after_gap=0 cryptarchia=50 mempool=50 sdp=50 wallet=50 blend/core=50 pow=50
  truncated: nothing
  power-loss view (only fsynced bytes kept):  VERIFY blocks_prefix=50 blocks_after_gap=0 cryptarchia=50 mempool=50 sdp=50 wallet=50 blend/core=50 pow=50
power-loss[large-syncrec]: rounds=800 block=100KiB sync=recovery acked=800 fsync+fdatasync calls=4817
  process-crash view (dir as left by abort): VERIFY blocks_prefix=800 blocks_after_gap=0 cryptarchia=800 mempool=800 sdp=800 wallet=800 blend/core=800 pow=800
  truncated: nothing
  power-loss view (only fsynced bytes kept):  VERIFY blocks_prefix=800 blocks_after_gap=0 cryptarchia=800 mempool=800 sdp=800 wallet=800 blend/core=800 pow=800
power-loss[small-nosync-cleanclose]: rounds=50 block=4KiB sync=none acked=50 fsync+fdatasync calls=11
  process-crash view (dir as left by abort): VERIFY blocks_prefix=50 blocks_after_gap=0 cryptarchia=50 mempool=50 sdp=50 wallet=50 blend/core=50 pow=50
  truncated: 000004.log:301659->0
  power-loss view (only fsynced bytes kept):  VERIFY blocks_prefix=0 blocks_after_gap=0 cryptarchia=0 mempool=0 sdp=0 wallet=0 blend/core=0 pow=0
LATENCY synced=false n=300 mean_us=2.6
LATENCY synced=true n=300 mean_us=152.4
```

Syncs issued in the 800-round unsynced run (thread id, call, file relative to the DB directory). The first eleven are the open (MANIFEST, CURRENT via `*.dbtmp` and rename, OPTIONS, directory). The last three are the memtable flush: the closed WAL `000004.log` is synced first (the two-column-family rule, `db_impl_compaction_flush.cc:168-172`), then the directory, then the new SST, whose sync the abort interrupted (`= ?`). Nothing syncs the live WAL `000008.log`, whose 16.9 MB (rounds 638/639 to 800) the emulation discards:

```
4409  fdatasync(6</000000.dbtmp>) = 0
4409  fsync(6<>) = 0
4409  fdatasync(6</MANIFEST-000001>) = 0
4409  fdatasync(6</000001.dbtmp>) = 0
4409  fsync(4<>) = 0
4409  fdatasync(8</MANIFEST-000005>) = 0
4409  fdatasync(9</000005.dbtmp>) = 0
4409  fsync(4<>) = 0
4409  fdatasync(8</MANIFEST-000005>) = 0
4409  fsync(10</OPTIONS-000006.dbtmp>) = 0
4409  fsync(10<>) = 0
4411  fdatasync(7</000004.log> <unfinished ...>
4411  <... fdatasync resumed>)          = 0
4411  fsync(4<>) = 0
4411  fdatasync(7</000009.sst> <unfinished ...>
4411  <... fdatasync resumed>)          = ?
```

### B.3 Effective options (OPTIONS file of the test DB, node settings, both column families identical)

```
atomic_flush=false
avoid_flush_during_recovery=false
avoid_flush_during_shutdown=false
bytes_per_sync=0
manual_wal_flush=false
max_open_files=-1
max_total_wal_size=0
paranoid_checks=true
track_and_verify_wals_in_manifest=false
use_fsync=false
wal_bytes_per_sync=0
wal_recovery_mode=kPointInTimeRecovery
write_buffer_size=67108864
column families: [CFOptions "default"], [CFOptions "blocks"]
```

### B.4 Keystore write path

`persist_user_config_and_keystore` (`keys.rs:215-230`) and `config init` (`init.rs:57-61`) call `std::fs::write`. The same call in a standalone program, run on an 8 KiB file that stands in for an existing keystore; the new content is 4 KiB larger (one key added). `RLIMIT_FSIZE` of 4 KiB stands in for a full disk: the kernel returns `EFBIG` at the same point where it would return `ENOSPC`.

```rust
use std::os::unix::fs::MetadataExt;
fn main() {
    let path = std::env::args().nth(1).unwrap();
    let before = std::fs::metadata(&path).unwrap().size();
    let new_yaml = "k".repeat(before as usize + 4096);
    let res = std::fs::write(&path, new_yaml.as_bytes());
    let after = std::fs::metadata(&path).unwrap().size();
    println!("before={before} B  write_result={res:?}  after={after} B");
}
```

```
$ bash -c 'trap "" XFSZ; ulimit -f 4; ./ks keystore.yaml'
before=8205 B  write_result=Err(Os { code: 27, kind: FileTooLarge, message: "File too large" })  after=4096 B
$ head -c 20 keystore.yaml
kkkkkkkkkkkkkkkkkkkk
$ strace -e trace=openat,write,fsync,fdatasync,rename,close ./ks keystore.yaml   # without the limit
openat(AT_FDCWD, "keystore.yaml", O_WRONLY|O_CREAT|O_TRUNC|O_CLOEXEC, 0666) = 3
write(3, "kkkkkkkkkkkkkkkkkkkkkkkkkkkkkkkk"..., 12301) = 12301
close(3)                                = 0
```

The old 8,205 B are gone at `openat`; no `fsync` and no `rename` follow the write.

### B.5 `services/storage/tests/durability_757.rs`

```rust
//! AUDIT #757 (scratch checkout only, not part of the node): durability of the
//! node's own RocksDB write path under a process crash and under an emulated
//! power loss. Driven by `audit757.py`; every mode is `#[ignore]`d.
//!
//! Modes (env `A757_MODE`):
//! - `write`: open the DB exactly as the node does (`RocksBackend::new` with
//!   the node-default settings), then for round i = 1..=`A757_ROUNDS`:
//!   `store_block_data(block i)` and one `store(recovery/<svc>, i)` per
//!   recovery key of the node, then print `ACK i`. `A757_SYNC=recovery` writes
//!   the recovery keys with `WriteOptions::set_sync(true)` through the node's
//!   own `DB` handle instead. `A757_END=abort` aborts after the last round (no
//!   destructor, no clean close); `A757_END=hang` sleeps so the driver can
//!   SIGKILL the process mid-stream.
//! - `verify`: reopen with `RocksBackend::new` and report what survived.
//! - `latency`: mean latency of a 200 B recovery-sized put, unsynced vs synced.
#![allow(clippy::all, clippy::pedantic, clippy::nursery, missing_docs)]

use std::{
    collections::BTreeMap,
    io::Write as _,
    path::PathBuf,
    time::{Duration, Instant},
};

use bytes::Bytes;
use lb_core::header::HeaderId;
use lb_cryptarchia_engine::Slot;
use logos_blockchain_storage_service::{
    backend::StorageBackend as _,
    recovery::recovery_key,
    rocksdb::{RocksBackend, RocksBackendSettings},
};

/// Every `RECOVERY_KEY_SUFFIX` at c4c86be1.
const SUFFIXES: [&[u8]; 6] = [
    b"cryptarchia",
    b"mempool",
    b"sdp",
    b"wallet",
    b"blend/core",
    b"pow",
];

fn env(name: &str, default: &str) -> String {
    std::env::var(name).unwrap_or_else(|_| default.to_owned())
}

fn settings() -> RocksBackendSettings {
    // Node defaults: nodes/node/binary/src/config/storage/serde.rs:19-26.
    RocksBackendSettings {
        db_path: PathBuf::from(env("A757_DB", "/nonexistent")),
        read_only: false,
        column_family: Some("blocks".to_owned()),
    }
}

fn id(i: u64) -> HeaderId {
    let mut raw = [0u8; 32];
    raw[..8].copy_from_slice(&i.to_be_bytes());
    raw[31] = 0x57;
    HeaderId::from(raw)
}

fn value(i: u64, len: usize) -> Bytes {
    let mut v = vec![(i % 251) as u8; len.max(8)];
    v[..8].copy_from_slice(&i.to_le_bytes());
    Bytes::from(v)
}

fn rt() -> tokio::runtime::Runtime {
    tokio::runtime::Builder::new_current_thread()
        .build()
        .unwrap()
}

#[test]
#[ignore = "AUDIT #757 harness"]
fn a757() {
    match env("A757_MODE", "").as_str() {
        "write" => write(),
        "verify" => verify(),
        "latency" => latency(),
        other => panic!("unknown A757_MODE {other:?}"),
    }
}

fn write() {
    let rounds: u64 = env("A757_ROUNDS", "100").parse().unwrap();
    let block_len: usize = env("A757_BLOCK_KB", "100").parse::<usize>().unwrap() * 1024;
    let sync_recovery = env("A757_SYNC", "none") == "recovery";
    let rt = rt();
    let mut backend = RocksBackend::new(settings()).unwrap();
    let mut out = std::io::stdout().lock();
    for i in 1..=rounds {
        let immutable = BTreeMap::from([(Slot::from(i), id(i))]);
        rt.block_on(backend.store_block_data(
            id(i),
            id(i - 1),
            value(i, block_len),
            value(i, 256),
            immutable,
        ))
        .unwrap();
        for suffix in SUFFIXES {
            // The chain envelope after #210's split is about 177 B; the other
            // records are small too. Size does not change the durability model.
            let (key, val) = (recovery_key(suffix), value(i, 200));
            if sync_recovery {
                backend
                    .txn(move |db| {
                        let mut wo = rocksdb::WriteOptions::default();
                        wo.set_sync(true);
                        db.put_opt(key, val, &wo)?;
                        Ok(None)
                    })
                    .execute()
                    .unwrap();
            } else {
                rt.block_on(backend.store(key, val)).unwrap();
            }
        }
        writeln!(out, "ACK {i}").unwrap();
        out.flush().unwrap();
    }
    match env("A757_END", "abort").as_str() {
        "hang" => loop {
            std::thread::sleep(Duration::from_secs(60));
        },
        // Clean close: drop the handle (RocksDB `~DB`), then exit normally.
        "drop" => {
            drop(backend);
            println!("DROPPED");
        }
        _ => std::process::abort(),
    }
}

fn verify() {
    let rt = rt();
    let mut backend = match RocksBackend::new(settings()) {
        Ok(b) => b,
        Err(e) => {
            println!("VERIFY open_failed={e}");
            return;
        }
    };
    let max: u64 = env("A757_ROUNDS", "100000").parse().unwrap();
    let mut last_contiguous = 0;
    let mut present_after_gap = 0;
    let mut gap = false;
    for i in 1..=max {
        let block = rt.block_on(backend.get_block(id(i))).unwrap();
        let parent = rt.block_on(backend.get_block_parent(id(i))).unwrap();
        let index = rt
            .block_on(backend.get_immutable_block_id(Slot::from(i)))
            .unwrap();
        // Atomicity of the block batch: all three or none.
        assert_eq!(block.is_some(), parent.is_some(), "torn block batch at {i}");
        assert_eq!(block.is_some(), index.is_some(), "torn block batch at {i}");
        match (block.is_some(), gap) {
            (true, false) => last_contiguous = i,
            (false, false) => gap = true,
            (true, true) => present_after_gap += 1,
            (false, true) => {}
        }
    }
    let mut line = format!(
        "VERIFY blocks_prefix={last_contiguous} blocks_after_gap={present_after_gap}"
    );
    for suffix in SUFFIXES {
        let v = rt.block_on(backend.load(&recovery_key(suffix))).unwrap();
        let i = v.map_or(0, |b| u64::from_le_bytes(b[..8].try_into().unwrap()));
        line.push_str(&format!(" {}={i}", String::from_utf8_lossy(suffix)));
    }
    println!("{line}");
}

fn latency() {
    let n: u32 = env("A757_N", "200").parse().unwrap();
    let backend = RocksBackend::new(settings()).unwrap();
    let rt = rt();
    for synced in [false, true] {
        let mut b = backend.clone();
        let t0 = Instant::now();
        for i in 0..n {
            let (key, val) = (recovery_key(b"pow"), value(u64::from(i), 200));
            if synced {
                b.txn(move |db| {
                    let mut wo = rocksdb::WriteOptions::default();
                    wo.set_sync(true);
                    db.put_opt(key, val, &wo)?;
                    Ok(None)
                })
                .execute()
                .unwrap();
            } else {
                rt.block_on(b.store(key, val)).unwrap();
            }
        }
        let us = t0.elapsed().as_secs_f64() * 1e6 / f64::from(n);
        println!("LATENCY synced={synced} n={n} mean_us={us:.1}");
    }
}
```

### B.6 `audit757.py`

```python
#!/usr/bin/env python3
"""AUDIT #757 driver (scratch only).

process-kill: run the writer, SIGKILL it after ACK K, reopen, verify.
power-loss:   run the writer under strace, record for every file the length it
              had at its last fsync/fdatasync, then build a copy of the DB in
              which every file is cut back to that length (files written but
              never synced are emptied). That is the state POSIX guarantees
              after a power loss; the kernel may have written back more, never
              less. Reopen the copy with the node's open path and verify.
"""
import os, re, shutil, signal, subprocess, sys

BIN = sys.argv[1]
WORK = sys.argv[2]
os.makedirs(WORK, exist_ok=True)


def run(mode, db, extra=None, strace=None):
    env = dict(os.environ, A757_MODE=mode, A757_DB=db)
    env.update(extra or {})
    cmd = [BIN, "a757", "--ignored", "--nocapture", "--test-threads=1", "-q"]
    if strace:
        cmd = ["strace", "-f", "-y", "-qq", "-s", "0", "-o", strace,
               "-e", "trace=openat,write,writev,pwrite64,pwritev,fsync,fdatasync,"
                     "ftruncate,rename,renameat,renameat2"] + cmd
    return subprocess.run(cmd, env=env, capture_output=True, text=True)


def last_ack(text):
    acks = [int(l.split()[1]) for l in text.splitlines() if l.startswith("ACK ")]
    return acks[-1] if acks else 0


def verify(db):
    out = run("verify", db).stdout
    return next(l for l in out.splitlines() if l.startswith("VERIFY"))


FD = re.compile(r"^(\d+)\s+(\w+)\((\d+)<([^>]*)>(.*)\)\s+=\s+(-?\d+)")
REN = re.compile(r'^(\d+)\s+(rename\w*)\((?:[^,]*,\s*)?"([^"]+)",\s*(?:[^,]*,\s*)?"([^"]+)"')


def merged_lines(trace):
    """Join strace's `<unfinished ...>` / `<... resumed>` halves per pid."""
    pending = {}
    for line in open(trace):
        line = line.rstrip("\n")
        pid = line.split(None, 1)[0]
        if line.endswith("<unfinished ...>"):
            pending[pid] = line[: -len("<unfinished ...>")].rstrip()
            continue
        m = re.match(r"^(\d+)\s+<\.\.\. \w+ resumed>(.*)$", line)
        if m:
            head = pending.pop(pid, None)
            if head is None:
                continue
            line = head + m.group(2)
        yield line


def emulate_power_loss(trace, src, dst):
    hi, cur, synced, touched = {}, {}, {}, set()
    for line in merged_lines(trace):
        m = REN.match(line)
        if m:
            old, new = m.group(3), m.group(4)
            for d in (hi, cur, synced):
                if old in d:
                    d[new] = d.pop(old)
            if old in touched:
                touched.discard(old)
                touched.add(new)
            continue
        m = FD.match(line)
        if not m:
            continue
        call, path, rest, ret = m.group(2), m.group(4), m.group(5), int(m.group(6))
        if ret < 0:
            continue
        if call in ("write", "writev"):
            cur[path] = cur.get(path, 0) + ret
            hi[path] = max(hi.get(path, 0), cur[path]); touched.add(path)
        elif call in ("pwrite64", "pwritev"):
            off = int(rest.rsplit(",", 1)[1])
            hi[path] = max(hi.get(path, 0), off + ret); touched.add(path)
        elif call == "ftruncate":
            hi[path] = int(rest.rsplit(",", 1)[1]); touched.add(path)
        elif call in ("fsync", "fdatasync"):
            synced[path] = hi.get(path, 0)
    shutil.rmtree(dst, ignore_errors=True)
    shutil.copytree(src, dst)
    report = []
    for name in sorted(os.listdir(dst)):
        if name.startswith("LOG") or name == "LOCK":
            continue
        path_src = os.path.join(os.path.realpath(src), name)
        if path_src not in touched:
            continue
        keep = synced.get(path_src, 0)
        size = os.path.getsize(os.path.join(dst, name))
        if keep < size:
            os.truncate(os.path.join(dst, name), keep)
            report.append(f"{name}:{size}->{keep}")
    return report


def process_kill(rounds_before_kill):
    db = os.path.join(WORK, "kill")
    shutil.rmtree(db, ignore_errors=True)
    env = dict(os.environ, A757_MODE="write", A757_DB=db, A757_ROUNDS="100000",
               A757_BLOCK_KB="100", A757_END="hang")
    p = subprocess.Popen([BIN, "a757", "--ignored", "--nocapture", "--test-threads=1", "-q"],
                         env=env, stdout=subprocess.PIPE, text=True)
    acked = 0
    for line in p.stdout:
        if line.startswith("ACK "):
            acked = int(line.split()[1])
            if acked >= rounds_before_kill:
                p.send_signal(signal.SIGKILL)
                break
    p.wait()
    # A few more rounds may have been written but not yet read by us.
    print(f"process-kill: SIGKILL after reading ACK {acked}")
    print("  " + verify(db))


def power_loss(tag, rounds, block_kb, sync, end="abort"):
    db = os.path.join(WORK, tag)
    shutil.rmtree(db, ignore_errors=True)
    trace = os.path.join(WORK, tag + ".strace")
    r = run("write", db, {"A757_ROUNDS": str(rounds), "A757_BLOCK_KB": str(block_kb),
                          "A757_SYNC": sync, "A757_END": end}, strace=trace)
    acked = last_ack(r.stdout)
    syncs = sum(1 for l in open(trace) if re.search(r"\b(fsync|fdatasync)\(", l))
    print(f"power-loss[{tag}]: rounds={rounds} block={block_kb}KiB sync={sync} acked={acked} "
          f"fsync+fdatasync calls={syncs}")
    # Build the power-loss copy first: `verify` reopens read-write, which
    # replays the WAL into an SST and deletes the old WAL.
    cut = emulate_power_loss(trace, db, db + ".powerloss")
    print("  process-crash view (dir as left by abort): " + verify(db))
    print("  truncated: " + (", ".join(cut) or "nothing"))
    print("  power-loss view (only fsynced bytes kept):  " + verify(db + ".powerloss"))


if __name__ == "__main__":
    process_kill(300)
    power_loss("small-nosync", 50, 4, "none")
    power_loss("large-nosync", 800, 100, "none")
    power_loss("small-syncrec", 50, 4, "recovery")
    power_loss("large-syncrec", 800, 100, "recovery")
    power_loss("small-nosync-cleanclose", 50, 4, "none", end="drop")
    lat = os.path.join(WORK, "latency")
    shutil.rmtree(lat, ignore_errors=True)
    print(run("latency", lat, {"A757_N": "300"}).stdout.strip())
```
