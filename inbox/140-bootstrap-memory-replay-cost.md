# Audit Report — Bootstrap-mode memory and replay cost

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/140`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `services/chain/chain-service, consensus/cryptarchia-engine, ledger, merkle/utxotree`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md, overview-cryptoeconomics.md, cryptarchia-v1-bootstr-sync.md, fork-choice.md (in full); cryptarchia-v1-protocol.md (Constants, Notation, Latest Immutable Block, Fork Choice Rule, Chain Maintenance, Commit, Fork Pruning)`
Date: `2026-09-11` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: the honest-chain baseline for "how much does a bootstrapping node hold and recompute" is unbounded in the chain length, and the measured constants make a one-year chain unsyncable on ordinary hardware: about 4.4 kB of resident memory per empty block and 30–90 kB per block carrying 16–64 transfers, retained until the node goes Online, and 0.8–44 ms of single-core CPU per block replayed on every restart.
- Findings: `0` critical · `0` high · `2` medium · `2` low · `0` informational
- Key themes: "memory grows with the chain, not with `k`, while bootstrapping", "every restart re-syncs the whole chain from local storage", "the proposed bounding rule (commit `k`-deep and older than `s_gen`) contradicts the Bootstrap fork choice rule and needs a spec-level decision"
- Must-fix before launch: LB-001 (a node syncing a one-year chain from genesis needs tens of GB of RAM before it can switch to Online) and LB-002 (each restart of a bootstrapping node replays and re-writes the whole chain; hours for a one-year chain).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `consensus/cryptarchia-engine/src/lib.rs` | `State::lib`, `update_lib`, `Branches` (what is retained per block, what is pruned and when) |
| `services/chain/chain-service/src/lib.rs` | `Cryptarchia::try_apply_block_with_state_retention`, `prune_ledger_states`, `online`, `initialize_cryptarchia`, `load_recovery_blocks_from_storage` |
| `services/chain/chain-service/src/service/mod.rs` | `process_block` (clone, validate, store, prune), `process_block_and_update_state`, `record_recovery_state`, `persist_recovery_state` |
| `services/chain/chain-service/src/service/phases/{ibd.rs,pbp.rs}` | where blocks enter during IBD and PBP, switch to Online |
| `services/chain/chain-service/src/states.rs` | recovery state contents (`lib_ledger_state`) |
| `services/chain/chain-service/src/uncle.rs` | per-uncle PoL verification cost on replay |
| `services/storage/src/api/backend/rocksdb/chain.rs` | `store_block_data` writes on replay |
| `ledger/src/lib.rs`, `ledger/src/cryptarchia/mod.rs`, `ledger/src/mantle/*`, `ledger/src/update.rs` | `Ledger::states`, `LedgerState` layout and which collections are structurally shared |
| `merkle/utxotree`, `merkle/tree`, `merkle/dynamic-merkle` | per-update path-copy cost of the UTXO tree |
| `zk/proofs/pol/src/lib.rs`, `core/src/proofs/leader_proof.rs` | PoL verification cost per block |

**Out of scope**

- The provider side of sync, stream bounds, stall timeouts, IBD termination (parent issue #2 questions).
- Adversarial long fake chains (the parent's question); this report is the honest-chain baseline the parent asked for. Fork blocks add to every number here at the same per-block rate.
- RocksDB write throughput: the storage cost of replay is identified but not measured.
- Third-party crates assumed correct: `rpds`, `archery`, `rocksdb`, `ark-*`/`lb-groth16`, `rust-rapidsnark`, `tokio`, `overwatch`.

**Assumptions**

- Mainnet constants from `cryptarchia-v1-protocol.md`: `k = 2160`, `f = 1/30`, slot = 1 s, 3/3/4 epoch phases (epoch = 648,000 slots), `W = 12`, `MAX_UNCLES = 4`. A year is `31,536,000` slots, so the expected honest chain length is `f · slots ≈ 1,051,200` blocks; the extrapolations below use `1.05 M` blocks per year.
- Repo-level facts from issue #19 apply (release profile without overflow checks; restriction lints allowed). Nothing in this report depends on integer overflow.
- Measurements were taken on one machine (Apple Silicon, 14 cores, macOS; rustc 1.98.1, `--release`, single-threaded) and are order-of-magnitude constants, not benchmarks of the target hardware. The spec's minimum node CPU is slower than this machine, so the CPU estimates are optimistic.
- The transaction workload used for the per-transaction numbers is one `Transfer` with one input, one output and one ZK signature per transaction (the ledger's own `zk_batch` bench workload). Real transactions with more inputs/outputs cost proportionally more UTXO updates.

## 3. Method

- Manual review of the in-scope paths, working through issue `#140` (all four checklist items) with parent `#2`, `#19` and the prior report on `#42` (PR #132, LB-001 and S-002) for context. Re-verified the ⚑ items from #140 at the commit above: `State::lib` returns the stored LIB in `Bootstrapping` (`consensus/cryptarchia-engine/src/lib.rs:60-74`), `update_lib` prunes only when the LIB changes (`lib.rs:517-531`), `prune_ledger_states` is fed only by `PrunedBlocks` (`services/chain/chain-service/src/lib.rs:534-551`, `service/mod.rs:836`), and `initialize_cryptarchia` replays every stored block in `(LIB, tip]` through `process_block` (`lib.rs:1035-1057`). All still hold.
- Spec conformance against `cryptarchia-v1-bootstr-sync.md` (Setting the Fork Choice Rule, Initial Block Download, Prolonged Bootstrap Period), `fork-choice.md` (Definitions, Bootstrap Fork Choice Rule), `cryptarchia-v1-protocol.md` (Latest Immutable Block, Chain Maintenance, Commit, Fork Pruning).
- Automated tooling run: none.
- Dynamic testing: three measurement programs built from the audited tree in a private copy (upstream `linker=rust-lld` override in `.cargo/config.toml` removed because it fails against this machine's macOS SDK; no other change to the tree). Sources are reproduced in Appendix B.
  1. `ledger/examples/bootstrap_cost.rs` (new): a counting `GlobalAlloc` measures live heap before and after applying `N = 20,000` header-only blocks at 30-slot spacing (crossing one epoch boundary) to (a) the engine alone (`Cryptarchia::receive_block`, `Bootstrapping`) and (b) the ledger alone (`Ledger::prepare_update` → `verify_batch_proofs` → `commit_update`, the node's exact path, with a `LeaderProof` whose `verify` is a no-op so the Groth16 cost is measured separately). The delta divided by `N` is the marginal retained cost per block; run for genesis sets of 10,000 and 100,000 UTXOs. Section (c) retains 200 versions of a `UtxoTree` after `t` removals and `t` insertions each, for `t ∈ {1, 2, 4, 16, 64, 256}`. Section (d) times `Ledger::clone()` and drop with 20,000 states retained.
  2. `zk/proofs/pol/examples/pol_verify_time.rs` (new, derived from the crate's `prove` bench): one real PoL proof, then 200 timed `lb_pol::verify` calls.
  3. The existing `ledger/benches/zk_batch.rs` (`single_block`, pool reduced to 64 transactions to keep proving time short): block application with batched ZK-signature verification, without proofs, and with sequential verification, for 1, 4, 16 and 64 transfers per block.

**Measured constants** (mean of the runs; both genesis sizes unless a range is given):

| Quantity | Value | Source |
|---|---|---|
| `size_of::<LedgerState>()` (inline, before heap) | 1,896 B | harness |
| `size_of::<Branch<HeaderId>>()` | 104 B | harness |
| Engine: retained per block (`Branch` in `Branches.branches` + `tips`) | 216 B | harness A |
| Engine: `receive_block` | 0.40 µs | harness A |
| Ledger: retained per header-only block (`LedgerState` in `Ledger.states`) | 3,970–4,320 B (checkpoints at 5k/10k/15k/20k blocks) | harness B |
| Ledger: `prepare_update` + `commit_update`, header-only, no Groth16 | 28 µs | harness B |
| UTXO tree: retained per version, `t = 1` remove+insert | 5,106 B (10k set) / 5,520 B (100k set) | harness C |
| UTXO tree: retained per remove+insert, `t = 16` per version | 1,551 B / 1,955 B | harness C |
| UTXO tree: retained per remove+insert, `t = 64` per version | 1,025 B / 1,391 B | harness C |
| UTXO tree: retained per remove+insert, `t = 256` per version | 662 B / 989 B | harness C |
| UTXO tree: time per remove+insert (Poseidon2 path rehash) | 353 µs | harness C |
| `Ledger::clone()` / engine clone with 20,000 states retained | 0.04 µs each | harness A, D |
| Drop of a ledger with 20,000 retained states | 27 ms | harness D |
| PoL Groth16 verify (single, one core) | 0.805 ms | pol example |
| PoL prove (for reference) | 94 ms | pol example |
| Block apply, 1 / 4 / 16 / 64 transfers, batched ZK verify | 2.21 / 4.21 / 12.6 / 43.6 ms | zk_batch |
| Block apply, 1 / 4 / 16 / 64 transfers, no proof verification | 0.27 / 1.16 / 5.05 / 20.5 ms | zk_batch |
| Block apply, 64 transfers, sequential ZK verify | 186 ms | zk_batch |

What was checked and ruled out:

- `process_block` clones the whole `Cryptarchia` for every block (`service/mod.rs:804`). Both `Ledger::states` and the engine's `Branches` are `rpds` tries, so the clone is O(1) (0.04 µs with 20,000 states) and is not a cost term. Ruled out.
- The engine's own retention is small: 216 B/block, 227 MB for a one-year chain. The engine is not the memory problem; the ledger states are.
- `LedgerState` collections are structurally shared as the source comments require (`ledger/src/lib.rs:242-244`): `UtxoTree` (`rpds::HashTrieMapSync` + `Arc` Merkle nodes), `BlockDensity.occupied_slots`, `PowState.{nullifiers,block_slots}`, `LeaderState.nfs`, `SdpLedger`, `Channels`, `EpochState.active_declarations: Arc`. The remaining per-state cost is the inline struct (1,896 B; breakdown in LB-001) plus the trie path copies of each insert. No unshared `Vec`/`HashMap` was found on the per-block path.
- Pruning when the node finally goes Online works and is cheap: `online()` → `update_lib` → `prune_immutable_blocks` walks from LIB to genesis once, and dropping 20,000 states took 27 ms. In `Online`, memory is bounded by `k` states plus forks (about 2,160 × per-block cost, i.e. 9 MB for empty blocks, 200 MB at 64 transfers per block). The problem is specific to `Bootstrapping`.
- Blocks are fed to `process_block` one at a time during IBD (`chain-network/src/bootstrap/ibd.rs:208`); IBD itself does not buffer the chain in memory. The recovery path does (LB-003).
- Equal-slot chaining and future slots cannot inflate the chain length beyond the honest rate (ledger rejects `slot <= parent.slot`, `ledger/src/cryptarchia/mod.rs:264-269`; future slots rejected at `chain-service/src/lib.rs:443`), so the honest-chain length used for extrapolation is the correct baseline; an adversary with stake adds fork blocks at the same per-block cost.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Bootstrap mode retains one `LedgerState` per block for the whole chain: 4.4 kB per empty block, 30–90 kB per block with transfers, tens of GB for a one-year chain | Denial of Service | Medium | Low | Open |
| LB-002 | Every restart of a bootstrapping node re-validates, re-verifies and re-writes every block since genesis: 0.8–44 ms of CPU per block, 15 minutes to 13 hours for a one-year chain | Denial of Service | Medium | Low | Open |
| LB-003 | Recovery materialises every block in `(LIB, tip]` in one `Vec<Block>` before replaying | Denial of Service | Low | Low | Open |
| LB-004 | The recovery state, including the full LIB `LedgerState`, is serialised and written to storage after every processed block, including during IBD | Denial of Service | Low | Low | Open |

### LB-001 · Bootstrap mode retains one `LedgerState` per block for the whole chain: 4.4 kB per empty block, 30–90 kB per block with transfers, tens of GB for a one-year chain

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `services/chain/chain-service/src/lib.rs:534-551` (`prune_ledger_states`), `services/chain/chain-service/src/service/mod.rs:836` (only caller on the block path), `consensus/cryptarchia-engine/src/lib.rs:60-74` (`State::lib`) and `:517-531` (`update_lib`), `ledger/src/lib.rs:211-213` (`commit_update`) |
| Status | Open |

**Description**

`commit_update` inserts one `LedgerState` per applied block into `Ledger::states` (`ledger/src/lib.rs:211-213`). States are removed only by `prune_ledger_states`, which is fed the `PrunedBlocks` the engine returns from `update_lib` (`service/mod.rs:836`). In `Bootstrapping`, `State::lib` returns the stored LIB (`engine lib.rs:65`), so `update_lib` sees no change and returns an empty `PrunedBlocks` (`engine lib.rs:520-521`). Nothing is pruned from IBD start until the Prolonged Bootstrap Period ends, and for a node starting from genesis that is the entire chain.

Measured marginal cost of one retained block (Method table):

- engine `Branch` in the `rpds` tries: 216 B;
- `LedgerState` for an empty block: 3,970–4,320 B. The inline struct is 1,896 B, of which `fee_window: [GasCost; 120]` (`ledger/src/cryptarchia/mod.rs:232-233`) is 960 B and the two inline `EpochState` snapshots' scalar fields (`nonce`, `lottery_0`, `lottery_1`, `blend_pow_difficulty` at 32 B each) most of the rest; the remainder is the path copy of the insert into `Ledger::states` and into the per-block sets `BlockDensity.occupied_slots` and `PowState.block_slots`;
- each UTXO remove+insert adds 5.1–5.5 kB when it is the only update in the block, 1.6–2.0 kB each at 16 updates per block, 1.0–1.4 kB at 64 and 0.7–1.0 kB at 256 (path copies through the 32-level `DynamicMerkleTree` and the `HashTrieMap` are shared between updates in the same block; the higher figures are for the 100,000-UTXO set).

Extrapolated to one year at `f = 1/30` (`1.05 M` blocks), resident memory a bootstrapping node holds before it can switch to Online:

| Workload (per block) | Retained per block | One-year chain |
|---|---|---|
| empty blocks | 4.4 kB | 4.6 GB |
| 16 single-input transfers | 29–36 kB | 31–37 GB |
| 64 single-input transfers | 70–93 kB | 74–98 GB |
| 256 single-input transfers | 174–258 kB | 182–271 GB |

For comparison, the same node in `Online` mode holds about `k = 2,160` states: 9 MB (empty blocks) to 200 MB (64 transfers per block). The `Bootstrapping` figure is 500× larger and grows without bound. Blocks with more inputs and outputs per transaction, uncle-carrying blocks, and fork blocks all add at the same per-update rate.

**Exploit scenario**

No attacker is needed. A node that joins a network whose chain is one year old with modest transaction load (16 transfers per block on average) must hold 31–37 GB of ledger states in RAM until it has finished IBD and the 24-hour PBP; on the spec's minimum hardware it is killed by the OOM killer during IBD, restarts, and (LB-002) starts again from genesis. A stake-holding adversary can accelerate this only linearly (fork blocks with full transaction bodies), so the finding is rated on the honest baseline.

**Recommendation**

- *Short term*: keep the engine `Branch` for every block (227 MB/year is affordable and is what the density rule and `lca` need), but do not keep a `LedgerState` for every block. Retain states only for a window behind each tip (for example `k` blocks) plus periodic in-memory or on-disk checkpoints every `C` blocks, and re-derive the state at a deeper attach point from the nearest checkpoint by replaying stored blocks when a fork actually arrives there. Bound the re-derivation by `C` so one valid fork block cannot force an O(chain) replay. Shrink the inline state: `fee_window` (1,920 B per state) can be a shared `rpds` structure or a ring in an `Arc`.
- *Long term*: decide at the spec level what a bootstrapping node may treat as final (see S-001), so that the LIB can advance during bootstrap and the `Online` pruning path applies. Add a metric for the number of retained ledger states (`metrics.rs` reports `forks_count` but not state count) so operators can see the growth.

**References**: `cryptarchia-v1-protocol.md` § Latest Immutable Block ("`B_imm` does not advance as new blocks are added unless the Online fork choice rule is used"), § Commit; `fork-choice.md` § Bootstrap Fork Choice Rule; PR #132 LB-001.

### LB-002 · Every restart of a bootstrapping node re-validates, re-verifies and re-writes every block since genesis: 0.8–44 ms of CPU per block, 15 minutes to 13 hours for a one-year chain

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `services/chain/chain-service/src/lib.rs:1035-1057` (`initialize_cryptarchia`, phase 2), `services/chain/chain-service/src/service/mod.rs:804-827` (`process_block`: `try_apply_block_with_state_retention` then `store_block_data`), `services/chain/chain-service/src/lib.rs:451-478` (`verify_uncles`, `prepare_update`, `verify_batch_proofs`), `services/storage/src/api/backend/rocksdb/chain.rs:45-70` (`store_block_data`) |
| Status | Open |

**Description**

On start, `initialize_cryptarchia` rebuilds the tree from the persisted LIB by pushing every stored block in `(LIB, tip]` through `process_block` (`lib.rs:1038-1047`). `process_block` is the same function the network path uses: it verifies uncles (one PoL Groth16 verify per uncle, `uncle.rs:66-75`), applies the ledger update (`prepare_update`, which verifies the block's own PoL inside `try_apply_proof`, `ledger/src/cryptarchia/mod.rs:493-516`), batch-verifies the transaction ZK proofs (`verify_batch_proofs`), and then calls `store_block_data`, which unconditionally re-writes the block bytes, the parent index and the events to RocksDB (`rocksdb/chain.rs:59-68`). For a bootstrapping node the LIB is genesis (LB-001), so `(LIB, tip]` is the whole chain.

Measured CPU per replayed block, one core (Method table):

| Block | CPU per block | of which avoidable ZK (PoL + tx proofs) |
|---|---|---|
| empty | 0.83 ms (0.805 PoL + 0.028 ledger + 0.0004 engine) | 0.805 ms |
| 16 transfers | 13.4 ms | 8.4 ms |
| 64 transfers | 44.4 ms | 24.0 ms |
| each uncle | +0.805 ms | +0.805 ms |

The non-ZK part is dominated by the Poseidon2 Merkle path re-hash of each UTXO update (353 µs per remove+insert, `merkle/dynamic-merkle`), not by bookkeeping.

Extrapolated to a one-year chain (`1.05 M` blocks), CPU only, single core, excluding storage:

| Workload | Full replay (current) | Replay without ZK verification |
|---|---|---|
| empty blocks | 14.5 min | 30 s |
| 16 transfers per block | 3.9 h | 1.5 h |
| 64 transfers per block | 13.0 h | 6.0 h |

To this add one RocksDB write batch per block (block + parent + events; the whole chain is re-written on every restart) and one recovery-state write per block (LB-004). The replay runs before the service reports ready and before IBD, so during those hours the node serves nothing.

**Exploit scenario**

Operational, no attacker: a bootstrapping node on a one-year chain with 16 transfers per block that is restarted (upgrade, crash, OOM from LB-001) spends about four hours replaying before it can even resume IBD, and PR #132 LB-001 showed the PBP then restarts from zero as well. A node restarted more often than replay + IBD + 24 h never goes Online. Any remotely triggerable crash of a bootstrapping node (none claimed here) turns into hours of lost time per trigger.

**Recommendation**

- *Short term*: persist enough state to avoid the replay: write the tip `LedgerState` (or a `LedgerState` checkpoint every `C` blocks) to storage together with the recovery state, and on restart load the newest checkpoint and replay at most `C` blocks. Do not call `store_block_data` for blocks that were read from the same store; split `process_block` so replay uses a "validate and apply" path without the write.
- *Long term*: with LB-001 fixed so the LIB (or an internal checkpoint) advances during bootstrap, `(LIB, tip]` is bounded and the replay cost becomes O(`k`) as it already is for Online nodes.

**References**: `cryptarchia-v1-bootstr-sync.md` § Setting the Fork Choice Rule (local block tree case); PR #132 LB-001, LB-004.

### LB-003 · Recovery materialises every block in `(LIB, tip]` in one `Vec<Block>` before replaying

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `services/chain/chain-service/src/lib.rs:889-911` (`load_recovery_blocks_from_storage`), `services/chain/chain-service/src/lib.rs:1020-1033` |
| Status | Open |

**Description**

```rust
// lib.rs:898-910
let mut blocks = Vec::new();
for id in ids.into_iter().rev().skip(1) {
    let block = storage.get_block(&id).await.ok_or(Error::HeaderIdNotFound(id))?;
    blocks.push(block);
}
Ok(blocks)
```

Phase 1 first collects every block id from tip back to LIB (`load_block_ids_from_storage`, one storage round trip per block), then loads and deserialises every block into a `Vec<Block<Tx>>` before phase 2 applies any of them. For a bootstrapping node this is the entire chain's block bodies at once: with the spec's `MAX_BLOCK_SIZE` of 2 MiB the bound is 2 TB per year, and even the honest average (16 transfers of a few hundred bytes each) is about 5 GB per year of chain, on top of LB-001's retained states, which are rebuilt while the `Vec` is still alive.

**Exploit scenario**

Not remotely triggerable on its own; it multiplies the memory peak of LB-001 at the moment of restart. A node that survived IBD with its states resident may not survive the restart, because the restart needs the block bodies and the states at the same time.

**Recommendation**

- *Short term*: stream the replay: walk the ids from LIB to tip and load each block immediately before `process_block`, dropping it afterwards (the id list can be built in reverse by a first pass that keeps only 32-byte ids, or by storing a height index).
- *Long term*: same as LB-002; a bounded `(LIB, tip]` bounds this too.

**References**: PR #132 LB-004 (error handling in the same loop).

### LB-004 · The recovery state, including the full LIB `LedgerState`, is serialised and written to storage after every processed block, including during IBD

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `services/chain/chain-service/src/service/mod.rs:244` (`process_block_and_update_state` → `record_recovery_state`), `services/chain/chain-service/src/service/mod.rs:1235-1250` (`persist_recovery_state`), `services/chain/chain-service/src/states.rs:11-30` (`lib_ledger_state: LedgerState`), `services/storage/src/recovery.rs:111-133` (`save_state` serialises and stores) |
| Status | Open |

**Description**

`process_block_and_update_state` calls `record_recovery_state` after every block it applies (`service/mod.rs:244`), on the IBD, PBP and Following paths alike (`apply_block_and_reply`, `service/mod.rs:186-206`). The recovery state embeds the whole LIB `LedgerState` (`states.rs:14`), whose `UtxoTree` serialises as a `CompressedUtxoTree` of every UTXO (`merkle/utxotree/src/lib.rs:233-251`), plus the two `EpochState` snapshots, each with its own UTXO tree. The state is handed to the Overwatch state updater, whose operator serialises it and issues a storage write on every update (`recovery.rs:111-133`; PR #132 established that the operator writes on each update; not re-verified here).

The cost per block is therefore O(|UTXO set at LIB|) serialisation plus a write of that size, while the useful information that changes per block is the tip id and a timestamp. During bootstrap the LIB is genesis, so it is the genesis UTXO set that is re-serialised for every one of the `1.05 M` blocks of a one-year IBD; after the switch to Online it is the live UTXO set at the `k`-deep block.

**Exploit scenario**

Not an attack; a throughput ceiling. IBD speed is bounded by one full serialisation and write of the LIB ledger state per block. The size was not measured (the serialised form depends on the deployment's genesis and UTXO set); filed as follow-up.

**Recommendation**

- *Short term*: write the `LedgerState` only when the LIB changes, and write the small per-block part (tip, timestamp, `storage_blocks_to_remove`) separately; or rate-limit the per-block write to the existing one-minute timer, since a lost tip is recovered by IBD anyway.
- *Long term*: store ledger-state checkpoints keyed by block id in their own storage key space (this is also what LB-002 needs), and keep the recovery record to ids.

**References**: `cryptarchia-v1-bootstr-sync.md` § Offline Duration Measurement (the timestamp is the only thing that must be recorded periodically).

## 5. Suggestions (non-security)

### S-001 · Spec: bounding the bootstrap window needs a rule for what a bootstrapping node may commit; "k-deep and older than s_gen" is not that rule

Target: `fork-choice.md` § Bootstrap Fork Choice Rule; `cryptarchia-v1-protocol.md` § Latest Immutable Block, § Commit; implementation `consensus/cryptarchia-engine/src/lib.rs:81-115` (`maxvalid_bg`). Corrects the premise of PR #132 S-002 and the long-term recommendation of PR #132 LB-001.

Checklist item 4 asked whether blocks that are both `k`-deep and older than `s_gen` slots could be committed during bootstrap because "the density rule can no longer overturn" them. They cannot, under the spec as written: `bootstrap_fork_choice` compares densities whenever `depth_max > k`, at any depth, and the denser fork wins (`fork-choice.md`, the `else` branch; `maxvalid_bg` lines 103-111). A fork that diverges a month back and is denser in the `s_gen` window after the divergence beats the local chain regardless of how old the divergence is; that is precisely the long-range case the Genesis rule exists for. Committing any block during bootstrap therefore changes the fork choice rule, and no local condition on the block's own age or depth makes it safe. The ledger states of deep blocks are needed only to validate fork blocks attaching there (the engine's `Branch` data suffices for the density comparison), which is why the implementation-side recommendation in LB-001 keeps `Branch` for every block and bounds only the states.

If the specification wants bootstrap memory bounded by `k` rather than by chain length, it has to say what a bootstrapping node commits and when. One shape, used by Cardano's Genesis implementation, is to resolve density comparisons on candidate chains before adopting blocks more than `k` past the intersection with any candidate, so that the node's own selection is never rolled back more than `k` even under the Genesis rule. That is a design decision for the spec editors, not something the node can adopt on its own.

### S-002 · Recovery replay does not need to re-verify ZK proofs of blocks the node itself accepted, but skipping them only halves the cost

Target: `services/chain/chain-service/src/lib.rs:1035-1057`; checklist item 3.

The recovery state's `lib_ledger_state` is loaded from local storage without any verification (`states.rs:14`, `lib.rs:987-996`); whoever can alter stored blocks can alter the LIB state the replay starts from. Re-verifying PoL, uncle PoL and transaction proofs for blocks read from the same store therefore protects against nothing that the trust already placed in the store does not, provided the block id is recomputed from the stored bytes (`Block::header().id()` is a content hash, so `ParentMissing` catches tampering with the chain structure). A "trusted local replay" that skips `verify_batch_proofs`, the PoL check inside `try_apply_proof`, and `verify_uncle_pol` is consistent with the existing trust model.

It is not sufficient as a fix: the measured non-ZK cost is 0.32 ms per single-input transfer (UTXO Merkle re-hash) and still 6 hours for a one-year chain at 64 transfers per block (LB-002 table). The replay itself has to be bounded (state checkpoints), after which whether the bounded part re-verifies proofs is a minor choice; keeping full verification for the bounded tail is the simpler and safer default.

### S-003 · Bench and metric gaps

Target: `ledger/benches/zk_batch.rs`; `services/chain/chain-service/src/metrics.rs:5-20`.

- The only per-block benchmark in the tree (`zk_batch`) measures block application without the PoL verify and without retained-state memory; the harness in Appendix B could be added as a bench so that the per-block memory constant (LB-001) is tracked, since a single unshared collection added to `LedgerState` would multiply it.
- `emit_consensus_metrics` exposes tip height, LIB height and fork count but not the number of retained ledger states or the resident size; during bootstrap the first is the number that matters.

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

## Appendix B — Measurement harness

Built and run from a copy of `logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` with `cargo build --release -p logos-blockchain-ledger --example bootstrap_cost`, `cargo build --release -p logos-blockchain-pol --example pol_verify_time`, and `cargo bench -p logos-blockchain-ledger --bench zk_batch -- single_block` (with `TXS_PER_BLOCK = [1, 4, 16, 64]`). Runs: `bootstrap_cost 20000 10000` and `bootstrap_cost 20000 100000`.

`ledger/examples/bootstrap_cost.rs`:

```rust
#![allow(clippy::all, clippy::pedantic, clippy::restriction, unused_qualifications)]
use std::{alloc::{GlobalAlloc, Layout, System}, collections::HashMap, num::NonZero,
    sync::{Arc, atomic::{AtomicUsize, Ordering}}, time::Instant};
use lb_core::{header::HeaderId, mantle::{Note, SignedOps, Utxo, gas::MainnetGasProfile,
    ledger::verification_mode::StandardMode, ops::leader_claim::VoucherCm,
    transactions::states::Preverified}, proofs::leader_proof::{LeaderProof, LeaderPublic}};
use lb_cryptarchia_engine::{Cryptarchia, EpochConfig, Slot, State, UncleSlots};
use lb_groth16::Fr;
use lb_key_management_system_keys::keys::{Ed25519PublicKey, ZkKey};
use lb_utils::math::{NonNegativeRatio, PositiveF64};
use logos_blockchain_ledger::{Config, Ledger, LedgerState, UtxoTree,
    config::{BlendPoWConfig, ModulusShift, PoWConfig, RewardPoWConfig},
    mantle::sdp::{ServiceRewardsParameters, rewards}};
use num_bigint::BigUint;

struct Counting;
static LIVE: AtomicUsize = AtomicUsize::new(0);
static PEAK: AtomicUsize = AtomicUsize::new(0);
fn bump(size: usize) {
    let now = LIVE.fetch_add(size, Ordering::Relaxed) + size;
    let mut peak = PEAK.load(Ordering::Relaxed);
    while now > peak {
        match PEAK.compare_exchange_weak(peak, now, Ordering::Relaxed, Ordering::Relaxed) {
            Ok(_) => break, Err(p) => peak = p }
    }
}
unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 { bump(l.size()); unsafe { System.alloc(l) } }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        LIVE.fetch_sub(l.size(), Ordering::Relaxed); unsafe { System.dealloc(p, l) } }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        if new >= l.size() { bump(new - l.size()); } else { LIVE.fetch_sub(l.size() - new, Ordering::Relaxed); }
        unsafe { System.realloc(p, l, new) } }
}
#[global_allocator] static A: Counting = Counting;
fn live() -> usize { LIVE.load(Ordering::Relaxed) }

struct DummyProof { leader_key: Ed25519PublicKey, voucher_cm: VoucherCm }
impl LeaderProof for DummyProof {
    fn verify(&self, _: &LeaderPublic) -> bool { true }
    fn verify_genesis(&self) -> bool { true }
    fn entropy(&self) -> Fr { Fr::from(7u8) }
    fn leader_key(&self) -> &Ed25519PublicKey { &self.leader_key }
    fn voucher_cm(&self) -> &VoucherCm { &self.voucher_cm }
}

/// k = 2160, f = 1/30, 3/3/4 epoch phases, W = 12; other values as in `zk_batch`.
fn config() -> Config {
    let epoch_config = EpochConfig {
        epoch_stake_distribution_stabilization: NonZero::new(3).unwrap(),
        epoch_period_nonce_buffer: NonZero::new(3).unwrap(),
        epoch_period_nonce_stabilization: NonZero::new(4).unwrap() };
    let consensus_config = lb_cryptarchia_engine::Config::new(NonZero::new(2160).unwrap(),
        NonNegativeRatio::new(1, 30.try_into().unwrap()), 1f64.try_into().expect("1 > 0"),
        NonZero::new(12).unwrap());
    let epoch_length = epoch_config.epoch_length(consensus_config.base_period_length());
    Config { epoch_config, consensus_config,
        sdp_config: logos_blockchain_ledger::mantle::sdp::Config {
            service_params: Arc::new(HashMap::from([(lb_core::sdp::ServiceType::BlendNetwork,
                lb_core::sdp::ServiceParameters { inactivity_period: 2.try_into().unwrap(), epoch: 0.into() })])),
            service_rewards_params: ServiceRewardsParameters { blend: rewards::blend::RewardsParameters {
                rounds_per_epoch: epoch_length.try_into().unwrap(),
                message_frequency_per_round: PositiveF64::try_from(1.0).unwrap(),
                num_blend_layers: NonZero::new(3).unwrap(), minimum_network_size: NonZero::new(1).unwrap(),
                data_replication_factor: 0, activity_threshold_sensitivity: 1 } },
            min_stake: lb_core::sdp::MinStake { threshold: 1, timestamp: 0 } },
        faucet_pk: None,
        pow_config: PoWConfig {
            blend: BlendPoWConfig { base_difficulty: ModulusShift::new::<234>(),
                target_transactions_per_block: NonZero::new(10).unwrap(), max_step: NonZero::new(4).unwrap(),
                damping_num: NonZero::new(1).unwrap(), damping_den_offset: 1 },
            reward: RewardPoWConfig { reward_pool_genesis: 1_000_000_000, epoch_reward_genesis: 1_000_000,
                initial_difficulty: ModulusShift::new::<26>(), ema_smoothing_factor: 9,
                ema_smoothing_precision: NonZero::new(10).unwrap(), target_claims_per_block: 100,
                rate_num: 0, rate_den: NonZero::<u64>::MIN, target_claim_per_block: NonZero::<u64>::MIN,
                slot_window: NonZero::new(100).unwrap() } } }
}
fn id(i: u64) -> HeaderId { let mut b = [0u8; 32]; b[..8].copy_from_slice(&i.to_le_bytes()); b[8] = 1; HeaderId::from(b) }
fn utxo(i: u64, key: &ZkKey) -> Utxo { let mut op_id = [0u8; 32]; op_id[..8].copy_from_slice(&i.to_le_bytes());
    Utxo { op_id, output_index: 0, note: Note::new(1_000_000, key.to_public_key()) } }

fn main() {
    let args: Vec<String> = std::env::args().collect();
    let n_blocks: u64 = args.get(1).and_then(|s| s.parse().ok()).unwrap_or(20_000);
    let n_genesis: u64 = args.get(2).and_then(|s| s.parse().ok()).unwrap_or(10_000);
    let slot_step: u64 = 30;
    let config = config();
    let key = ZkKey::from(BigUint::from(1u8));
    let genesis_utxos: Vec<Utxo> = (0..n_genesis).map(|i| utxo(i, &key)).collect();
    let genesis = LedgerState::from_utxos(genesis_utxos.iter().cloned(), &config);
    let genesis_id = id(0);
    println!("size_of LedgerState={}B, engine Branch<HeaderId>={}B",
        std::mem::size_of::<LedgerState>(), std::mem::size_of::<lb_cryptarchia_engine::Branch<HeaderId>>());
    { // A. engine only
        let mut engine = Cryptarchia::<HeaderId>::from_lib(genesis_id, config.consensus_config.clone(),
            State::Bootstrapping, Slot::from(0u64), 0, UncleSlots::default());
        let before = live(); let t0 = Instant::now();
        for i in 1..=n_blocks {
            engine.receive_block(id(i), id(i - 1), Slot::from(i * slot_step), UncleSlots::default()).expect("engine apply");
        }
        println!("A engine: {:.2} us/block, {:.1} B/block retained", t0.elapsed().as_secs_f64() * 1e6 / n_blocks as f64,
            (live() - before) as f64 / n_blocks as f64);
    }
    let proof = DummyProof { leader_key: Ed25519PublicKey::from_bytes(&[0u8; 32]).unwrap(), voucher_cm: VoucherCm::default() };
    { // B. ledger states, header-only blocks
        let mut ledger = Ledger::<HeaderId>::new(genesis_id, genesis.clone(), config.clone());
        let before = live(); let t0 = Instant::now();
        for i in 1..=n_blocks {
            let update = ledger.prepare_update::<SignedOps<Preverified, StandardMode>, _, MainnetGasProfile>(
                    id(i), id(i - 1), Slot::from(i * slot_step), &proof, &UncleSlots::default(), std::iter::empty())
                .expect("ledger apply").verify_batch_proofs().expect("no proofs");
            let _events = ledger.commit_update(update);
        }
        println!("B ledger header-only: {:.2} us/block, {:.1} B/block retained",
            t0.elapsed().as_secs_f64() * 1e6 / n_blocks as f64, (live() - before) as f64 / n_blocks as f64);
        let t0 = Instant::now(); let mut clones = 0u64;
        while t0.elapsed().as_millis() < 500 { let c = ledger.clone(); std::hint::black_box(&c); clones += 1; }
        println!("D Ledger clone: {:.2} us", t0.elapsed().as_secs_f64() * 1e6 / clones as f64);
        let t0 = Instant::now(); drop(ledger); println!("   drop: {:.1} ms", t0.elapsed().as_secs_f64() * 1e3);
    }
    { // C. UTXO tree churn
        let base: UtxoTree = genesis_utxos.iter().map(|u| (u.id(), u.clone())).collect();
        for t in [1usize, 2, 4, 16, 64, 256] {
            let versions = 200usize.min(n_genesis as usize / t / 2).max(1);
            let mut retained: Vec<UtxoTree> = Vec::with_capacity(versions);
            let mut tree = base.clone(); let mut next_new = n_genesis; let mut next_old = 0u64;
            let before = live(); let t0 = Instant::now();
            for _ in 0..versions {
                for _ in 0..t {
                    let old = utxo(next_old, &key); next_old += 1;
                    let (nt, _) = tree.remove(&old.id()).expect("remove"); tree = nt;
                    let new = utxo(next_new, &key); next_new += 1;
                    let (nt, _) = tree.insert(new.id(), new); tree = nt;
                }
                retained.push(tree.clone());
            }
            println!("C utxo churn t={t}: {:.1} B/version retained, {:.1} us/version",
                (live() - before) as f64 / versions as f64, t0.elapsed().as_secs_f64() * 1e6 / versions as f64);
            drop(retained);
        }
    }
}
```

`zk/proofs/pol/examples/pol_verify_time.rs` is the crate's `benches/prove.rs` with `main` replaced by: prove once with `make_inputs()`, then 5 warm-up and 200 timed `verify(&proof, &verifier_inputs)` calls, printing the mean.
