# Audit Report — Blend core recovery state: full-state clone, serialise and write per collected token

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/250`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `3d5d419ec85c7b23e4d7e1455bd1cde845264a9e` — component(s): `services/blend/src/core/{mod.rs,state.rs}`, `services/storage/src/recovery.rs`, `services/storage/src/{lib.rs,backends/rocksdb.rs}`, `services/utils/src/overwatch/recovery`, `blend/message/src/reward`, `core/src/blend/mod.rs`, overwatch `services/state/*` (git `ae887f41`)
Specs: `https://github.com/logos-co/logos-lips` @ `75d3d0382604d4a0d8e246c268935dd386ffc8ea` — read: `blend-protocol.md` (› Quota, › Processing, › Activity Proof, › Activity Threshold), plus the parent #13 set read for earlier reports
Date: `2026-09-15` — author: `Claude (research workflow, iteration 7)` — status: `final`

The Blend/services code paths named in #250 are unchanged between `a805329f` (the commit named in the issue) and `3d5d419e` (`git log a805329f..3d5d419e -- services/blend blend/message services/storage` is empty for the files cited here); line numbers below are for `3d5d419e`.

---

## 1. Summary

- Overall assessment: the per-message clone reported as S-003 in #72 is real, but the overwatch state channel already coalesces updates, so the expensive part (serialise + RocksDB write) runs off the service task and at most once per in-flight save. What remains is (a) a write-amplification term that is quadratic in the number of tokens a node collects per epoch and (b) the fact that persistence is asynchronous and happens *after* the cover message that spent the quota has been published, so a crash reuses PoQ key indices on restart. With the deployment defaults (`message_frequency_per_round: 1.0`, `num_blend_layers: 1`) the per-node state stays small unless the core membership is tiny, so the practical rating is Low.
- Findings: 0 critical · 0 high · 0 medium · 2 low · 1 informational
- Key themes: "snapshot-per-event persistence", "write-after-send durability ordering", "cost that scales as 1/N²"
- Must-fix before launch: none. Recommended before raising `message_frequency_per_round` or `num_blend_layers`: LB-001 (incremental or per-round persistence). Recommended before relying on the recovery state for quota accounting: LB-002.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/blend/src/core/state.rs` | `ServiceState`, its manual `Clone`, `save`, `StateUpdater::commit_changes`, `SerializableServiceState`, `RecoveryServiceState` |
| `services/blend/src/core/mod.rs` | All 10 `commit_changes()` call sites; epoch bootstrap that restores `spent_core_quota`; release round; cover-message generation |
| overwatch `src/services/state/{updater.rs,handle.rs,mod.rs}` @ `ae887f41` | `StateUpdater::update` (watch send), `StateHandle::run` (WatchStream → operator), spawn in `runner/service_runner.rs:336-341` |
| `services/utils/src/overwatch/recovery/operators.rs` | `RecoveryOperator::run` → `RecoveryBackend::save_state` |
| `services/storage/src/recovery.rs` | Storage-backed `RecoveryBackend`: bincode `to_bytes`, `StorageMsg::Store` under one key |
| `services/storage/src/{lib.rs,backends/rocksdb.rs}` | `handle_store` → synchronous `rocks.put` on the storage service task |
| `blend/message/src/reward/{mod.rs,token.rs,epoch.rs}` | `EpochBlendingTokenCollector`, `OldEpochBlendingTokenCollector`, `BlendingToken` layout |
| `core/src/blend/mod.rs`, `services/blend/src/settings/timing.rs`, `consensus/cryptarchia-engine/src/{config.rs,time.rs}`, `nodes/node/binary/src/config/deployment/settings.yaml` | Parameters used to size a full epoch of tokens |
| `services/blend/src/edge/` | Checked for the same pattern (item 4 of the issue) |

**Out of scope**
The RocksDB engine itself, `bincode`, `tokio::sync::watch` / `tokio-stream` (assumed correct; their documented semantics are relied on). Proof verification and reward evaluation logic. The edge service's *lack* of recovery state, already filed as #172 / #540. Storage error reporting (#403) and schema versioning (#524).

**Assumptions**
Deployment parameters as in `nodes/node/binary/src/config/deployment/settings.yaml` at the target commit: slot 1 s, `security_param` 30, activation coefficient 1/20, epoch config 3+3+4, `message_frequency_per_round` 1.0, `num_blend_layers` 1, `data_replication_factor` 0, `normalization_constant` 1.03, `maximum_release_delay_in_rounds` 1. Block leaders and data traffic are a small fraction of cover traffic (about 300 block proposals per epoch at one block per 20 slots versus about 6 000 cover messages).

## 3. Method

- Manual review of every `commit_changes()` site in the core service and the full persistence path from `ServiceState::save` to `rocksdb::DB::put`.
- Spec conformance against `blend-protocol.md` › Processing (what a token is and that it is stored per processed message), › Activity Proof, › Quota (`Q_C`, `Q_C^Total`) to size the token set.
- Automated tooling: none applicable.
- Dynamic testing: a throw-away `#[test]` added to `blend/message` (removed afterwards; the worktree is clean) that fills an `EpochBlendingTokenCollector` with N distinct tokens built from `from_bytes_unchecked` proofs, then times 20 clones and one `bincode::serialize`, release profile, Apple Silicon laptop:

| tokens | clone (µs) | serialise (µs) | serialised bytes | bytes/token |
|---|---|---|---|---|
| 1 000 | 69 | 163 | 224 028 | 224 |
| 10 000 | 516 | 2 028 | 2 240 028 | 224 |
| 50 000 | 2 104 | 13 016 | 11 200 028 | 224 |
| 100 000 | 4 770 | 27 509 | 22 400 028 | 224 |

  Clone is about 48 ns per token, serialise about 275 ns per token, and the wire size is exactly 224 B per token (32 B Ed25519 key + 32 B key nullifier + 128 B Groth16 proof + 32 B selection randomness) plus a 28 B header.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Every token collected and every release round re-clones, re-serialises and rewrites the whole recovery state: O(T²) bytes per epoch, scaling as 1/N² | Denial of Service | Low | Low | Open |
| LB-002 | Quota spend is persisted asynchronously and after the message is published, so a crash reuses PoQ key indices on restart | Data Validation | Low | Medium | Open |
| LB-003 | Three resident copies of the state per commit and a synchronous RocksDB put on the storage service task | Denial of Service | Informational | — | Open |

### LB-001 · Every token collected and every release round re-clones, re-serialises and rewrites the whole recovery state: O(T²) bytes per epoch, scaling as 1/N²

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `services/blend/src/core/state.rs:L242-L244` (`save`), `L565-L570` (`commit_changes`), `L120-L134` (manual `Clone`); `services/blend/src/core/mod.rs:L2245-L2246`, `L2280-L2282`, `L2467`; `services/storage/src/recovery.rs:L111-L133`; `services/storage/src/backends/rocksdb.rs:L130-L136` |
| Status | Open |

**Description**

What happens per commit, in order:

1. Service task: `commit_changes` → `save` → `self.state_updater.update(Some(self.clone().into()))` (`state.rs:243`). The manual `Clone` deep-copies both token sets, both unsent-message maps and the pending-transaction queue (`state.rs:120-134`). `into()` moves the clone into a `SerializableServiceState` without a second copy.
2. `StateUpdater::update` is a `tokio::sync::watch` send (overwatch `updater.rs:34-38`, channel created at `mod.rs:16-20`). The slot holds only the latest value, so if the operator is still busy the intermediate snapshots are dropped unseen. This is the answer to the issue's first checklist item: the operator neither writes every update nor debounces; it processes whatever value is current when it next polls.
3. State task (spawned per service in `runner/service_runner.rs:336-341`): `StateHandle::run` drives a `WatchStream`, which clones the slot's value again (`tokio-stream-0.1.19/src/wrappers/watch.rs:108`) and awaits `RecoveryOperator::run` → `save_state` (`services/utils/src/overwatch/recovery/operators.rs:61-66`).
4. `save_state` (`services/storage/src/recovery.rs:111-133`) bincode-serialises the whole state and sends `StorageMsg::Store { key: recovery_key(suffix), value }`, with no reply channel.
5. Storage task: `handle_store` calls `rocks.put` synchronously (`rocksdb.rs:130-136`; only `bulk_store` uses `spawn_blocking`). The put is a full-value overwrite of one key, so every commit is a fresh WAL record plus an SST entry of the whole state.

The commit sites that fire per event rather than per round are `handle_decapsulated_incoming_message_from_current_epoch` (`mod.rs:2245`), the old-epoch equivalent (`mod.rs:2280`), the local-encapsulation path (`mod.rs:2027`, `2089`), and the transaction queue/dequeue paths (`mod.rs:1487`, `1558`, `1611`). The release round commits once after `join_all(message_futures)` (`mod.rs:2467`), and only when `changed` is set (a message was released or a cover message spent quota).

Size of the state late in an epoch (second checklist item). Per `blend-protocol.md` › Processing step 2.2, a node stores one token per layer it successfully decapsulates, and with `data_replication_factor = 0` the network emits about `1.03 · rounds_per_epoch · message_frequency_per_round` cover messages per epoch (`core/src/blend/mod.rs:12-40`, normalisation applied in the scheduler). With the deployment defaults `rounds_per_epoch = epoch_length = (3+3+4) · floor(30 / (1/20)) = 6 000` (`cryptarchia-engine/src/time.rs:278-288`, `config.rs:117-130`, `timing.rs:30-31`), so the network produces about 6 180 cover-message layers per epoch and each of N core nodes collects `T ≈ 6 180 / N` tokens, plus a few hundred from block proposals. The old-epoch collector keeps the whole previous epoch's set for the transition period (`state.rs:114`, `reward/mod.rs:83-95`), so for that window the state holds about `2T` tokens. Unsent messages are bounded by the release delay (1 round) and are about 18.7 KB each (`PUBLIC_HEADER_ENCODED_SIZE` 257 B + one 289 B blending header + `PAYLOAD_ENCODED_SIZE` 18 195 B); they are few and short-lived.

Bytes written to RocksDB per epoch. Each token collection triggers one save of `28 + 224·k` bytes when the operator is idle, and each release round that changed anything triggers another. The number of release-round commits is about `2T` (one per cover message sent, one per processed message released), so the total is about `Σ_{k≤T} 224k · 3 ≈ 336·T²` bytes if no coalescing happens, less when saves overlap. Applying the measured 224 B/token:

| core nodes N | tokens/node/epoch T | state at epoch end | writes/epoch (upper bound) | clone on service task at epoch end |
|---|---|---|---|---|
| 2 | 3 090 | 692 KB (1.4 MB during transition) | ≈ 3.2 GB | ≈ 150 µs |
| 10 | 618 | 138 KB | ≈ 128 MB | ≈ 30 µs |
| 100 | 62 | 14 KB | ≈ 1.3 MB | ≈ 3 µs |
| any N, `message_frequency_per_round = N` (one cover per node per round) | 6 180 | 1.4 MB | ≈ 12.8 GB | ≈ 300 µs |

So with the shipped parameters and a membership of tens of nodes or more, the cost is negligible; it is a concern only for very small memberships (dev/testnets, the `minimum_network_size: 2` floor) or if `message_frequency_per_round · num_blend_layers` is raised so that per-node quota grows. The scaling law is the finding: cost per event is O(state), events per epoch are O(state), and the state is the whole epoch's history.

Third checklist item (can a per-round or incremental snapshot replace the clone without weakening recovery?). What the on-disk state guarantees after a crash today is already only "whatever the last completed put contained", because `commit_changes` returns before the watch send is observed, before serialisation, and before the put. Nothing in the service waits for durability. Under that model:

- Tokens: losing the last round's tokens costs at most one round of reward lottery entries. A per-round snapshot loses exactly the same set as today's worst case (the operator lagging one save).
- Unsent processed/data messages: they are released within `maximum_release_delay_in_rounds = 1` round anyway; a per-round snapshot taken at the end of the release round captures exactly the set that survived the round, which is what the release round already writes (`mod.rs:2467`).
- `spent_core_quota`: this is the one field with an ordering requirement, and it is already violated (LB-002). A per-round snapshot changes nothing there.
- `pending_transactions`: queued/dequeued at most a few times per epoch; per-event commits for those are cheap and can stay.

Fourth checklist item: the edge service has no state operator at all (`services/blend/src/edge/mod.rs:112` uses `NoOperator`; `grep -rn 'commit_changes\|state_updater' services/blend/src/edge` returns nothing), so the pattern does not exist there. Its missing recovery is #172 / #540.

**Exploit scenario**

Not attacker-triggerable beyond the message rate the quota already caps: a peer cannot make a node collect more tokens than the network's total quota allows. The impact is self-inflicted I/O and, in the transition period of a small network, a state of a few MB rewritten several times per second. On a Raspberry Pi 5 class target with the deployment quota this is not measurable at N ≥ 10.

**Recommendation**
- *Short term*: drop the per-token commit in `handle_decapsulated_incoming_message_from_current_epoch` / old-epoch variant and rely on the release-round commit (`mod.rs:2467`), which already runs once per round; keep per-event commits for the quota and transaction fields. This bounds saves to one per round and clones to one per round.
- *Long term*: make the token collector append-only on disk: persist each `BlendingToken` under `recovery_key(suffix) ++ epoch ++ token_hash` via `bulk_store`, and keep only the small mutable header (`last_seen_epoch`, `spent_core_quota`, unsent-message keys, pending transactions) in the single-key snapshot. Alternatively hold the token sets in a persistent structure (`rpds::HashTrieSet`, already a workspace dependency) so `Clone` is O(1) and the serialiser walks the shared structure.

**References**: `blend-protocol.md` › Processing step 2.2, › Quota (`Q_C`, `Q_C^Total`); #72 S-003; #516 (the same snapshot-per-event pattern in the Cryptarchia recovery pipeline).

### LB-002 · Quota spend is persisted asynchronously and after the message is published, so a crash reuses PoQ key indices on restart

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Data Validation |
| Target | `services/blend/src/core/mod.rs:L2645-L2660` (`generate_and_try_to_decapsulate_cover_message`), `L2440-L2467` (release round), `L765-L787` (restore); `blend/provers/src/crypto/core_and_leader/send.rs:L101-L122`; `services/storage/src/recovery.rs:L122-L133` |
| Status | Open |

**Description**

On restart the service reads `spent_core_quota` from the recovered state and hands it to the cryptographic processor, which logs "resuming core key indices from {spent_core_quota}" (`send.rs:105`) and continues the PoQ key sequence from there; the scheduler's remaining quota is `epoch_core_quota - spent_core_quota` (`mod.rs:787`). So correctness of quota accounting across a crash depends on the spend having reached disk before the corresponding cover message left the node.

It does not. In the release round, `consume_core_quota` is recorded in the in-memory updater (`mod.rs:2651`), the cover message is pushed to `message_futures`, the futures are awaited (`join_all`, `mod.rs:2465`, the message is now on the wire), and only then does `commit_changes` (`mod.rs:2467`) hand the snapshot to the watch channel. The write itself happens later on the state task and the storage task with no acknowledgement (`StorageMsg::Store` has no reply channel, `recovery.rs:122-133`; see also #403). A crash anywhere between `join_all` and the put restarts the node with the previous `spent_core_quota`, so the next cover messages carry the same PoQ key indices as messages already sent.

Consequences: the reused key index derives the same key nullifier, which peers that still hold it in their message cache drop as already seen (`blend/network/src/core/with_core/behaviour/message_cache.rs:69-71`, spec › Processing step 2.4.1.1), so those cover messages produce no cover traffic and no tokens for the recipients; peers whose cache has expired accept the duplicate nullifier, which the spec says the node "was not allowed to use". The node also believes it has more quota left than it does, so its cover schedule for the rest of the epoch is off by the reused count. The same ordering applies to `remove_sent_*` (a message can be released and then re-released after restart, which is harmless as peers deduplicate) and to token collection (tokens collected after the last completed put are lost, costing at most a round of lottery entries).

**Exploit scenario**

Not externally triggerable; it needs a crash or kill of the node in the window between publishing a cover message and the RocksDB put completing (tens of microseconds to milliseconds, longer when the state task is busy with a large snapshot per LB-001). Impact is bounded by one round's spend per crash. Listed because it shows that the recovery state's quota field cannot be relied on for exact accounting, which matters if PoQ nullifier reuse is later made a peer-penalisable offence.

**Recommendation**
- *Short term*: commit the quota spend before pushing the cover message to `message_futures`, and add a reply channel to the recovery `Store` so `save_state` can await the put (this also gives #403 its error path). Accept the one-round latency this adds to cover emission, or reserve indices one round ahead.
- *Long term*: persist `spent_core_quota` (and the small header) in its own key with an awaited write, separate from the token history, so the ordering requirement is cheap to honour.

**References**: `blend-protocol.md` › Processing step 2.4.1.1 (duplicate nullifier), › Core Quota; #403; #413 (mid-epoch quota spreading).

### LB-003 · Three resident copies of the state per commit and a synchronous RocksDB put on the storage service task

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Denial of Service |
| Target | `services/blend/src/core/state.rs:L243`; overwatch `services/state/handle.rs:L90-L100`; `tokio-stream/src/wrappers/watch.rs:L108`; `services/storage/src/backends/rocksdb.rs:L130-L136` |
| Status | Open |

**Description**

Per commit the state exists in the service (`ServiceState`), in the watch slot (the `save` clone), and in the operator (the `WatchStream` clone), plus the serialised `Bytes`. At the deployment defaults this is at most a few MB even at N = 2. Separately, `rocks.put` runs inline on the storage service's single message loop (`services/storage/src/lib.rs:339-340`), so a large recovery-state put delays every other storage message (block and transaction stores, chain recovery writes) by the put's duration; `bulk_store` already uses `spawn_blocking` (`rocksdb.rs:144-151`), `store` does not. Worth fixing when the state is made incremental, not before.

**Recommendation**
Route `store` through `spawn_blocking` like `bulk_store`, and pass the serialised value by `Bytes` (already the case) rather than re-cloning.

## 5. Suggestions (non-security)

### S-001 · `unsent_data_messages` is a `HashMap` keyed by the 18.7 KB encapsulated message

`state.rs:110`. Every insert, remove and clone hashes and copies the whole message. Keying by the message id (`remaining.id()` is already used at `mod.rs:2061`) and storing the message as the value would make the map cheaper to clone and serialise and would stop the recovery snapshot from carrying the key twice conceptually. Cosmetic today because the map holds at most one round of messages.

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
