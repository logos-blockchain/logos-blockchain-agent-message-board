# Audit Report — Long-range attacks on bootstrap and partial sync state across restart

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/42`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `services/chain/chain-service, services/chain/chain-network, consensus/cryptarchia-engine`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md, overview-cryptoeconomics.md, cryptarchia-v1-bootstr-sync.md, fork-choice.md (in full); cryptarchia-v1-protocol.md (Constants, Notation, Latest Immutable Block, Fork Choice Rule, Block Header Validation, Chain Maintenance, Commit, Fork Pruning)`
Date: `2026-09-11` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: the long-range defences the spec relies on (genesis as the only trust anchor, Bootstrap fork choice with the Genesis density rule, LIB frozen while bootstrapping, validate-before-persist) are implemented as specified; the gaps are around restart handling and the checkpoint path, which the spec requires and the node does not implement.
- Findings: `0` critical · `0` high · `1` medium · `3` low · `1` informational
- Key themes: "bootstrap progress is not durable across restart", "spec deviation: checkpoint bootstrap and 24h bootstrap period", "recovery errors are logged, not acted on"
- Must-fix before launch: LB-001 (a node restarted inside the Prolonged Bootstrap Period restarts it from zero and replays every block since LIB; a node restarted more often than the period never goes online); LB-002 (default Prolonged Bootstrap Period is 1h, spec says 24h).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/chain/chain-service/src/bootstrap/{config.rs,state.rs}` | fork-choice selection at startup, offline grace period |
| `services/chain/chain-service/src/states.rs` | recovery state contents, what is persisted |
| `services/chain/chain-service/src/lib.rs` | `initialize_cryptarchia`, replay of stored blocks, `StartingState` |
| `services/chain/chain-service/src/service/mod.rs` | `process_block` (validate, persist, commit), `persist_recovery_state` |
| `services/chain/chain-service/src/service/phases/*.rs` | phase machine, Prolonged Bootstrap Period timer, switch to online |
| `services/chain/chain-service/src/storage/adapters/storage.rs` | `store_block_data` semantics |
| `consensus/cryptarchia-engine/src/{lib.rs,config.rs}` | `maxvalid_bg`, `maxvalid_mc`, LIB handling per state, `s_gen` |
| `services/chain/chain-network/src/bootstrap/ibd.rs`, `src/lib.rs` (IBD start/finish, `should_process_block`), `src/sync/orphan_handler.rs` (`enqueue_orphan`) | how IBD feeds blocks into chain-service and what happens on failure |
| `ledger/src/cryptarchia/mod.rs` (`update_epoch_state` slot check only) | strict slot ordering, needed for the density rule |
| `nodes/node/binary/src/config/cryptarchia/{mod.rs,serde/service.rs}`, `nodes/node/standalone-node-config.yaml` | what the binary actually configures |
| `services/utils/src/overwatch/recovery/*`, `services/storage/src/recovery.rs`, Overwatch `services/state/*` @ `ae887f41` | when the recovery state is written |

**Out of scope**

- The provider side of sync (`chain-service/src/sync/block_provider.rs`, `cryptarchia-sync/src/libp2p/*`), stream bounds, per-request caps, stall timeouts, and IBD termination when peers disagree: these are the parent issue's (#2) questions and are only cross-referenced here.
- Proof of Leadership, uncle validation, ledger transaction rules, ZK proof verification: assumed correct as validation steps; only their position relative to persistence was checked.
- Third-party crates assumed correct: `rocksdb`, `libp2p`, `rpds`, `tokio`, `backon`, `overwatch` (behaviour of `StateHandle` was read, not audited).

**Assumptions**

- The genesis block in the deployment config is trusted (it is the only trust anchor the node has today).
- The specs at the commit above are the reference; where the spec is `raw` and silent, this report says so rather than inventing a rule.
- Repo-level facts from issue #19 apply: `overflow-checks` is off in release; `arithmetic_side_effects`, `unwrap_used`, `expect_used`, `panic` lints are allowed. The one addition on the audited path (`u64::from(lca.slot) + s_gen` at `consensus/cryptarchia-engine/src/lib.rs:106`) cannot wrap because block slots are bounded by the wall clock (`services/chain/chain-service/src/lib.rs:443`).

## 3. Method

- Manual review of the in-scope paths, working through issue `#42` (both checklist items) with parent `#2` and `#19` for context.
- Spec conformance against `cryptarchia-v1-bootstr-sync.md` (Setting the Fork Choice Rule, Initial Block Download, Prolonged Bootstrap Period, Downloading Blocks, Bootstrapping from Checkpoint, Offline Grace Period, Offline Duration Measurement), `fork-choice.md` (Definitions, Bootstrap and Online rules), `cryptarchia-v1-protocol.md` (Latest Immutable Block, Block Header Validation rules 5–8, Chain Maintenance, Commit).
- Automated tooling run: none.
- Dynamic testing: none.

What was checked and ruled out (item 1, long-range / fake long chain):

- Genesis trust: the node binary only ever constructs `StartingState::Genesis` (`nodes/node/binary/src/config/cryptarchia/mod.rs:110`), and `choose_engine_state` returns `Bootstrapping` whenever LIB is genesis (`services/chain/chain-service/src/bootstrap/state.rs:17-19`). A fresh node therefore always uses the Bootstrap rule. Matches spec.
- Density-based comparison: `maxvalid_bg` (`consensus/cryptarchia-engine/src/lib.rs:81-115`) compares `cmax.length - lca.length` with `k` and, when deeper, compares the chain heights at the last block with slot `<= lca.slot + s_gen` on each side. Heights count chain blocks only, so uncles carry no weight, matching `fork-choice.md` 1.1.0. `s_gen = floor(k / (4f))` (`config.rs:107-113`). Matches spec.
- LIB frozen while bootstrapping: `State::lib` returns the stored LIB in `Bootstrapping` and the `k`-th ancestor only in `Online` (`lib.rs:60-74`); `online()` is the only transition (`lib.rs:677-682`). Blocks whose parent is below LIB cannot attach (`apply_header` requires the parent in the tree, `lib.rs:239-242`; immutable ancestors are pruned on commit, `lib.rs:639-649`). The spec's key invariant (never roll back below `B_imm`) holds structurally.
- Equal-slot chaining, which would let an attacker inflate density: the engine only rejects `parent.slot > slot` (`lib.rs:244-246`), but the ledger, which runs first in `try_apply_block_with_state_retention`, rejects `slot <= parent.slot` (`ledger/src/cryptarchia/mod.rs:264-269`). Ruled out; see S-003.
- Future slots: rejected against the local clock before any ledger work (`services/chain/chain-service/src/lib.rs:442-448`), so an attacker cannot pre-build blocks for future slots to win density.
- Offline grace period: implemented as the spec recommends, with a timestamp written every minute and on every applied block (`states.rs:67-70`, `service/mod.rs:244`, `lib.rs:763-768`); clock going backwards falls back to `Bootstrapping` (`bootstrap/state.rs:45-52`). Matches spec; see S-001 for a spec-level gap.
- Blocks at or before LIB arriving by gossip are dropped before any work (`services/chain/chain-network/src/lib.rs:589-595`, `877-900`).
- IBD failure when no configured peer answers ends in a graceful shutdown (`services/chain/chain-network/src/lib.rs:328-345`), as the spec requires.

What was checked and ruled out (item 2, validate before persisting; partial state across restart):

- Every block, from IBD, orphan download, or gossip, goes through `process_block`, which applies the block to a clone of `Cryptarchia` (ledger rules, batch ZK verification, uncle checks, consensus engine) and only then calls `store_block_data`, which awaits the storage service's reply (`service/mod.rs:804-827`, `storage/adapters/storage.rs:93-127`). The in-memory state is replaced only after the store succeeds (`service/mod.rs:837`). Unit test `process_block_does_not_mutate_state_when_storage_send_fails` covers the failure side. An invalid block is never persisted.
- Recovery state (`tip`, `lib`, LIB ledger state, LIB slot/height/uncle slots, pending deletions, last engine state + timestamp) is pushed to the Overwatch state operator after every processed block and every minute, and the operator writes it to storage on every update (`services/utils/src/overwatch/recovery/operators.rs:61-66`, `services/storage/src/recovery.rs:111-133`, Overwatch `StateHandle::run`). Block write precedes the state write, so a crash between them leaves a stored block that is simply not referenced and is re-downloaded later. Safe.
- On restart, blocks in `(LIB, tip]` are re-read from storage and re-validated through `process_block` (`lib.rs:1015-1057`); a broken parent chain falls back to LIB and re-persists (`lib.rs:913-948`, test `recovery_blocks_fall_back_to_lib_when_tip_missing_from_storage`). Partial sync state therefore survives restart, and is re-validated rather than trusted. The cost of that replay is LB-001.

Noted but left to the parent issue (#2, "Can it loop forever?"): `InitialBlockDownload::download_blocks` loops until every configured peer's tip is in the local tree (`bootstrap/ibd.rs:176-189`); a configured peer whose tip chain never applies is retried without limit (test comment at `ibd.rs:512-516`), so the node never leaves the IBD phase and never goes online. The spec instead says IBD succeeds if at least one peer succeeds and otherwise terminates with an error. Filed as a follow-up sub-issue rather than rated here.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Prolonged Bootstrap Period restarts from zero on every restart, and every restart replays the whole chain since LIB | Consensus | Medium | High | Open |
| LB-002 | Spec deviation: default Prolonged Bootstrap Period is 1 hour, spec says 24 hours | Configuration | Low | Low | Open |
| LB-003 | Spec deviation: checkpoint bootstrap is absent, and the only non-genesis starting state would start with the Online rule | Consensus | Low | High | Open |
| LB-004 | A block that fails validation during recovery replay is skipped, leaving a truncated in-memory chain behind a stale recovery pointer | Error Reporting | Low | High | Open |
| LB-005 | Fork blocks persisted before shutdown are neither replayed nor deleted after restart | Denial of Service | Informational | Medium | Open |

### LB-001 · Prolonged Bootstrap Period restarts from zero on every restart, and every restart replays the whole chain since LIB

| | |
|---|---|
| Severity | Medium |
| Difficulty | High |
| Category | Consensus |
| Target | `services/chain/chain-service/src/service/phases/pbp.rs:86` (`run_event_loop`), `services/chain/chain-service/src/bootstrap/state.rs:21-23,36-43` (`choose_engine_state`), `services/chain/chain-service/src/lib.rs:1015-1057` (`initialize_cryptarchia`), `consensus/cryptarchia-engine/src/lib.rs:60-74` (`State::lib`) |
| Status | Open |

**Description**

The Prolonged Bootstrap Period (PBP) is an in-process `tokio::time::sleep` created when the phase is entered:

```rust
// pbp.rs:86
let mut pbp_timer = Box::pin(tokio::time::sleep(self.prolonged_bootstrap_period));
```

Nothing about the period (start time, remaining time) is part of `CryptarchiaConsensusState`. The recovery state records only `state: Bootstrapping` plus a timestamp (`states.rs:67-70`). On restart `choose_engine_state` returns `last_state.state` if the node was down for less than the grace period and `Bootstrapping` otherwise (`bootstrap/state.rs:36-43`), so a node that was bootstrapping always comes back bootstrapping and enters a fresh, full-length PBP after IBD. There is no path that resumes a partially elapsed period.

While bootstrapping the LIB does not move (`State::lib`, engine `lib.rs:65`), so for a node syncing from genesis the LIB stays at genesis for the entire IBD plus PBP. `initialize_cryptarchia` replays every stored block in `(LIB, tip]` through `process_block` (`lib.rs:1038-1057`), which re-runs ledger validation and batch ZK verification for each block and re-stores it. The replay is O(blocks since LIB) in CPU and, because nothing is pruned in `Bootstrapping`, the engine tree and the per-block ledger states for the whole chain are rebuilt in memory.

The spec (`cryptarchia-v1-bootstr-sync.md`, Prolonged Bootstrap Period) says the timer starts "when Listening for New Blocks is started after Initial Block Download is completed" and is 24 hours; it does not say what a restart does. The implementation resolves that silence in the direction that costs liveness: any restart during the 24 hours discards the elapsed time.

**Exploit scenario**

Operational: a node bootstrapping from genesis on a chain of a few hundred thousand blocks is restarted (upgrade, config change, OOM, crash) 20 hours into its 24-hour PBP. On restart it re-verifies every block since genesis, runs IBD (which completes immediately, all tips are local), then waits another 24 hours. A node whose restart cadence is shorter than the PBP never switches to `Online`, never advances its LIB, and never proposes blocks (`Proposing New Blocks` requires the Online rule), while still answering `Info` queries as healthy in `ProlongedBootstrapPeriod` phase.

Adversarial: any remotely triggerable crash of a bootstrapping node (a separate weakness; none is claimed here) turns into "this node never comes online, and every trigger costs it a full-chain re-verification". The attacker needs no stake.

**Recommendation**

- *Short term*: persist the PBP start time (or the deadline) in `CryptarchiaConsensusState` alongside `last_engine_state`, and on restart within the offline grace period resume the remaining time instead of restarting it. Log at `warn` when a restart resets the period so operators can see why the node is not online.
- *Long term*: let the LIB advance under a bounded rule during bootstrap (for example commit blocks that are both `k`-deep and older than `s_gen` slots, the point after which the density rule can no longer prefer another fork), so recovery replay and in-memory tree size are bounded by `k + s_gen` rather than by the chain length. This also needs a spec change; see S-002.

**References**: `cryptarchia-v1-bootstr-sync.md` § Prolonged Bootstrap Period, § Setting the Fork Choice Rule; `cryptarchia-v1-protocol.md` § Latest Immutable Block ("`B_imm` does not advance as new blocks are added unless the Online fork choice rule is used").

### LB-002 · Spec deviation: default Prolonged Bootstrap Period is 1 hour, spec says 24 hours

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Configuration |
| Target | `nodes/node/binary/src/config/cryptarchia/serde/service.rs:46-54` (`impl Default for BootstrapConfig`) |
| Status | Open |

**Description**

```rust
// serde/service.rs:49
prolonged_bootstrap_period: Duration::from_hours(1),
```

The spec fixes `T_boot` at 24 hours (`cryptarchia-v1-bootstr-sync.md` § Constants and § Prolonged Bootstrap Period). The node's user-config default is 1 hour, and the struct is `#[serde(default)]`, so an operator who does not set the field gets a 24x shorter period. The offline grace period default (20 minutes, `serde/service.rs:71`) does match the spec. The shipped `nodes/node/standalone-node-config.yaml:118-121` sets 5 seconds for both, which is fine for a single dev node but is the file an operator is most likely to copy.

The side the report believes is wrong: the code. The spec's argument for 24 hours (time to discover peers outside an isolated set and compare chains under the Genesis rule) is not weakened by anything in the implementation, and no code comment justifies the shorter value.

**Exploit scenario**

A node that synced from a partition of colluding or eclipsed peers switches to the Online rule after one hour and commits to that chain up to `k`-deep; from then on the honest chain is rejected as a fork deeper than `k`. The spec's 24-hour window exists precisely to reduce this.

**Recommendation**

- *Short term*: change the default to `Duration::from_hours(24)`; make the standalone config comment that its values are development-only.
- *Long term*: derive the default from the same constants table the spec uses (a single source shared with `k`, `f`) so spec and code cannot drift silently.

**References**: `cryptarchia-v1-bootstr-sync.md` § Constants (`T_boot` = 24 hours).

### LB-003 · Spec deviation: checkpoint bootstrap is absent, and the only non-genesis starting state would start with the Online rule

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Consensus |
| Target | `services/chain/chain-service/src/bootstrap/state.rs:17-27` (`choose_engine_state`), `services/chain/chain-service/src/lib.rs:599-609` (`StartingState`), `services/chain/chain-service/src/states.rs:94-111` (`from_settings`) |
| Status | Open |

**Description**

The spec lists three starting points, genesis, checkpoint, and local block tree, and requires the Bootstrap rule for the first two. The node implements genesis and local tree only. Checkpoint import (`Checkpoint Provider HTTP API`, ledger-state import) does not exist; the code marks it as a TODO pointing at logos-blockchain issue 1454 (`bootstrap/state.rs:25-26`). That alone is a gap, not a bug.

The bug is in the code that would serve the checkpoint case. `StartingState::Lib { lib_id, lib_ledger_state, genesis_id }` (`lib.rs:604-608`) is exactly a checkpoint: an arbitrary LIB with its ledger state. If used, `from_settings` produces a recovery state with `last_engine_state: None`, `lib_block_length: 0` and `lib_block_slot: 0` (`states.rs:101-111`), and `choose_engine_state` then falls through to:

```rust
// bootstrap/state.rs:17-27
if lib_id == genesis_id || config.force_bootstrap { return Bootstrapping; }
if let Some(last_state) = last_engine_state { return check_offline_grace_period(...); }
// TODO: Implement other criteria for bootstrapping
//       - Checkpoint: ...
lb_cryptarchia_engine::State::Online
```

so a node started from a checkpoint would use the Online rule and commit to the first `k`-deep chain it downloads, which is the case the spec's Bootstrap rule is meant to protect. The zeroed LIB slot also makes the LIB look like slot 0 to `maxvalid_bg`'s density window when the LIB is the LCA. Today the node binary never builds `StartingState::Lib` (`nodes/node/binary/src/config/cryptarchia/mod.rs:110`), so the path is dead, but it is public API of the service crate and the natural hook for issue 1454.

**Exploit scenario**

Not exploitable at this commit (dead path). If checkpoint support is added by wiring `StartingState::Lib` without touching `choose_engine_state`, a bootstrapping node would run the Online rule and a long-range fork served by the first peers it meets becomes immutable after `k` blocks.

**Recommendation**

- *Short term*: make the unknown case conservative: return `Bootstrapping` when `last_engine_state` is `None`, matching the clock-error branch at `bootstrap/state.rs:45-52`. Add a unit test for `StartingState::Lib`. Have `StartingState::Lib` carry the LIB slot and height so the engine is not seeded with zeros.
- *Long term*: implement checkpoint bootstrap as specified (trusted HTTP provider, import of block and ledger state, Bootstrap rule forced, termination if no peer chain connects to the checkpoint), and remove `StartingState::Lib` if it is not the vehicle for it.

**References**: `cryptarchia-v1-bootstr-sync.md` § Setting the Fork Choice Rule (first bullet), § Bootstrapping from Checkpoint, § Checkpoint Provider HTTP API.

### LB-004 · A block that fails validation during recovery replay is skipped, leaving a truncated in-memory chain behind a stale recovery pointer

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Error Reporting |
| Target | `services/chain/chain-service/src/lib.rs:1038-1057` (`initialize_cryptarchia`, phase 2), `services/chain/chain-service/src/lib.rs:750-756` (`run`) |
| Status | Open |

**Description**

```rust
// lib.rs:1039-1056
match process_block(&mut cryptarchia, block, current_slot, ...).await {
    Ok(outcome) => { ... }
    Err(e) => { error!(target: LOG_TARGET, "Error processing block: {:?}", e); }
}
```

Replay walks `(LIB, tip]` in order. If any block fails (`FutureBlock` when the clock moved back past that block's slot, a ledger rule that changed in an upgrade, a storage read that returned a block for the wrong id), the loop logs and continues; every later block then fails with `ParentMissing`, so the node comes up with its tip at the last good block while the persisted recovery state still names the old tip. The state is only re-persisted here when `fell_back_to_lib` is true (`lib.rs:750-756`), which this path does not set, so until the next block is applied a further restart repeats the same partial replay. The node then reports the shorter chain as its tip and re-downloads the rest from peers as if it had never had it.

**Exploit scenario**

Not remotely triggerable; the input is the node's own storage and clock. Impact is silent divergence between what the node persisted and what it runs on, and a repeat of the same errors on every restart, with `error!` lines as the only signal.

**Recommendation**

- *Short term*: on the first replay error, stop the loop, log which block failed and why, and persist the recovery state with the actual tip so the next restart is consistent. Treat `FutureBlock` during replay separately (clock moved back: wait for the slot rather than dropping the chain).
- *Long term*: distinguish "invalid block in storage" (corruption, should not happen; fail loudly) from "transient" (clock, storage I/O) errors with distinct handling, and add a test that injects a failing block mid-replay.

**References**: `cryptarchia-v1-bootstr-sync.md` § Introduction (key invariant), § Setting the Fork Choice Rule (local block tree case).

### LB-005 · Fork blocks persisted before shutdown are neither replayed nor deleted after restart

| | |
|---|---|
| Severity | Informational |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | `services/chain/chain-service/src/lib.rs:889-911` (`load_recovery_blocks_from_storage`), `services/chain/chain-service/src/service/mod.rs:1086-1132` (`delete_stale_blocks_from_storage`) |
| Status | Open |

**Description**

Every accepted block is persisted (`service/mod.rs:817-827`), including blocks on non-canonical forks. Recovery reloads only the parent chain from `tip` back to `LIB` (`lib.rs:894-908`), and deletion from storage is driven only by `PrunedBlocks` computed by the engine when the LIB moves (`service/mod.rs:237-242`, `phases/pbp.rs:131-136`). Forks that existed in the engine at shutdown are not in the tree after restart, so the engine never reports them as stale and they are never deleted; they stay in the block store, the parent index, and the events store for the life of the database. In `Bootstrapping`, where no pruning happens at all, every fork downloaded during IBD survives a restart on disk.

A second effect: after restart the Bootstrap fork choice has forgotten every competing fork it had already compared, and only re-learns them if a peer serves them again. This does not weaken the rule (the local chain was chosen against them once), but it does mean a restarted node cannot re-evaluate a previously seen fork against new information.

**Exploit scenario**

An attacker with enough stake to produce valid blocks on a fork can grow the on-disk footprint of a bootstrapping node by one block per fork block, and the growth is permanent across restarts. The cost to the attacker (a valid Proof of Leadership per block) makes this expensive, hence Informational.

**Recommendation**

- *Short term*: on restart, reload fork tips from storage as well (the parent index is enough to find blocks not on the canonical chain), or at least schedule the unreferenced blocks for deletion once the LIB passes their height.
- *Long term*: keep a storage-level index of block height so pruning below LIB can be a range delete independent of what the engine remembers.

**References**: `cryptarchia-v1-protocol.md` § Commit, § Fork Pruning.

## 5. Suggestions (non-security)

### S-001 · Spec: the offline-duration recommendation does not cover a running but partitioned node

Target: `cryptarchia-v1-bootstr-sync.md` § Offline Duration Measurement; implementation at `services/chain/chain-service/src/states.rs:67-70`, `services/chain/chain-service/src/service/phases/following.rs:78`.

The spec's recommended method (periodically record the wall clock while the Online rule is in use) is what the node does: the timestamp is refreshed every minute and on every applied block regardless of whether any block has arrived. A node that is running but cut off from the network for hours therefore restarts with the Online rule, although the threat the grace period is defined against (an adversary extends a `k-d` fork while the node is not watching) applies to it equally. The spec could define the measure as time since the last block was accepted under the Online rule, which the node already has in its recovery state.

### S-002 · Spec: define restart behaviour inside the Prolonged Bootstrap Period

Target: `cryptarchia-v1-bootstr-sync.md` § Prolonged Bootstrap Period; § Setting the Fork Choice Rule.

The spec starts `T_boot` "when Listening for New Blocks is started after Initial Block Download is completed" and is silent about a restart during the period, and about whether the offline grace period applies to a node that was last running the Bootstrap rule. LB-001 is the implementation's reading. A rule such as "the period is measured from its first start and persists across restarts; a restart after more than `T_offline` restarts it" would make the intended behaviour checkable.

### S-003 · Engine accepts a child in the same slot as its parent

Target: `consensus/cryptarchia-engine/src/lib.rs:244-246` (`Branches::apply_header`).

`apply_header` rejects only `parent.slot > slot`; spec rule 5 requires `slot > parent.slot`. The ledger enforces the strict form first (`ledger/src/cryptarchia/mod.rs:264`), so no block reaches the engine with an equal slot today. The engine's density rule depends on this strictness (equal-slot chains would inflate density), so the engine should enforce it itself and the `test_slot_increasing` test should cover the equal case.

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
