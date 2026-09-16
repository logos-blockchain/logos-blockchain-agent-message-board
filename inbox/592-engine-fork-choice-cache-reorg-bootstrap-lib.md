# Audit Report — Engine fork-choice cache (#193 S-001): reorg cost with many tips, Bootstrapping state, and incremental LIB

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/592`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `bfcae04d25218d878eb77c3c072cb0e88524de82` — component(s): `consensus/cryptarchia-engine` (`lib.rs`), read-only: `services/chain/chain-service` (`lib.rs`, `service/mod.rs`, `tests/mod.rs`, `states.rs`)
Specs: `https://github.com/logos-co/logos-lips` @ `75d3d0382604d4a0d8e246c268935dd386ffc8ea` — read in full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `cryptarchia-v1-protocol.md`, `fork-choice.md`
Date: `2026-09-16` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: the #193 S-001 prototype (spec `on_block` short-circuit plus a per-tip divergence cache) still applies cleanly at this commit, passes the engine's 26 unit tests, and holds its per-block gain (4,000 tips at `k = 2160`: 278 ms → 0.64 ms per fork block, 564 ms → 0.57 ms per canonical block; 20,000 tips: 3.3 ms and 2.6 ms). Its one unmeasured path, the reorg, costs one divergence recompute of about 105 µs per tip: 0.37 s at 4,000 tips, 2.1 s at 20,000, which is half the unmodified engine's reorg (0.76 s at 4,000) and is paid at most once per pair of consecutive attacker wins, `120·α²` times per hour at `f = 1/30` for a stake share `α`. The Bootstrapping state gets nothing from the cache: `maxvalid_bg` walks every tip twice, 210 ms per block at 1,000 tips (3.5× the Online baseline for the same tips) with no pruning to bound the tip count, and the density rule needs the LCA slot and a slot-bounded walk that the cache does not hold. An incremental-LIB prototype for S-003 (a `k+1` queue of the canonical tail) removes the last fixed cost: 0.42 ms → 0.017 ms per canonical block with no tips, and about 0.17 ms off every block. The short-circuit should apply in both states: the spec's `on_block` does not distinguish, and re-running the Bootstrap tournament on a tip-extending block is the only way the code can move off a chain it just extended. The chain-service and chain-network suites could not be run here (disk); by reading, the only consumer that depends on the outcome of a tip-extending block is one assertion that `reorged_blocks` is empty, which the prototype preserves.
- Findings: 0 critical · 0 high · 0 medium · 1 low · 2 informational
- Key themes: per-tip cost bounded in Online but not in Bootstrapping; reorg is the residual `O(tips × depth)` path; spec `on_block` short-circuit.
- Must-fix before launch: none; the prototype is evidence for the maintainers' design, not a proposed patch.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `consensus/cryptarchia-engine/src/lib.rs:42-141` (`State::fork_choice`, `State::lib`, `maxvalid_bg`, `maxvalid_mc`), `:442-531` (`receive_block_with_canonical_change`, `update_lib`), `:560-649` (pruning), `:677-682` (`online`) | measured unmodified, with the #193 Appendix B diff, and with that diff plus the S-003 prototype of Appendix B below |
| `services/chain/chain-service/src/lib.rs:438-503`, `service/mod.rs:800-840,912-950`, `tests/mod.rs:95-104`, `states.rs:230-340,410-500` | read for what depends on `newly_canonical_blocks` / `reorged_blocks` / `lib()` of a tip-extending block; not run |
| `processed/39-fork-choice-reorg-bound-fork-state.md` LB-002, `inbox/193-fork-tip-spam-measurements.md` (PR #591) S-001, S-003, Appendix B | the prior results this report extends |

**Out of scope**

- The ledger-side cost of a fork block (`prepare_update`, PoL verification), measured in #193; the attacker's cost model of #193 LB-001 is taken as given.
- The chain-network orphan downloader and IBD (#135, #216, #270); Bootstrapping memory growth (#140) beyond noting that nothing prunes tips in that state.
- `chain-service` and `chain-network` test suites: not run (see Method).
- Third-party crates assumed correct: `rpds` (persistent map, set and queue), `std::time::Instant`.

**Assumptions**

- Workspace release profile (`lto = "fat"`, `codegen-units = 1`, `overflow-checks` off) for every measurement; the engine is single-threaded and its cost is dominated by `rpds` hash-trie lookups, so the numbers scale with per-lookup latency of the machine and should be read as ratios.
- Machine: 4-core aarch64 VM, 15 GB RAM, Linux 7.0.0, `rustc 1.98.1`; one measurement process at a time. #193 measured on an Apple M3 Pro; the two baselines agree within 10 % at the one shared configuration (`k = 2160`, 4,000 tips: 255/562 ms there, 278/564 ms here).
- Slot length 1 s and `f = 1/30` for the per-hour rates.

## 3. Method

- Manual review of the in-scope paths, working through #592's five items against `fork-choice.md` §Bootstrap Fork Choice Rule, §Online Fork Choice Rule and `cryptarchia-v1-protocol.md` §Chain Maintenance, §Commit, §Fork Pruning.
- Item 1 (test suites): `cargo test -p logos-blockchain-chain-service` and `-p logos-blockchain-chain-network` need the node's full dependency graph (rocksdb, libp2p, tokio) and roughly 20 GB of target directory; the environment had 6 GB free, so they were **not run**. The engine crate alone builds in 35 s (`cargo test -p logos-blockchain-cryptarchia-engine`, 889 MB debug, 1.3 GB with release), and its 26 unit tests were run at each of the three stages. The consumer question was answered by reading.
- Items 2–4 (measurements): one integration test, `fork_reorg_592` (Appendix C), placed in `consensus/cryptarchia-engine/tests/` of a scratch checkout and removed afterwards; the checkout was restored to `bfcae04d` with `git status` clean. It rebuilds the #193 `fork_spam` tree: a canonical chain of `CHAIN` blocks one every 30 slots, a competing branch diverging `k/2` blocks below the tip and as long as the local chain (a tie, not adopted), then `TIPS` single-block forks with parents spread uniformly over heights `2..CHAIN−1`. It then times, with `std::time::Instant` around one `receive_block` each: one more fork block; one canonical extension (`newly_canonical_blocks == [b]`, `reorged_blocks` empty, asserted); the second block on the competing branch that makes it longer than the local chain (reorg; `reorged_blocks == CHAIN − d + 1` and `newly_canonical_blocks == k/2 + 2` asserted, adoption asserted); then one more fork block and one more canonical extension after the reorg. `CHAIN = k` in the Online state (LIB at genesis, every fork admissible) and `CHAIN = 2k` in the Bootstrapping state, so that half the tips diverge more than `k` below the tip and take the density path. Each configuration is a separate process, `cargo test --release --offline --locked ... -- --nocapture`. Three stages: unmodified engine; #193 Appendix B applied with `git apply` (clean at this commit); that plus the S-003 prototype (Appendix B), applied by exact-string replacement.
- Item 5: reasoning against the spec text and #39 LB-002's cyclic example; the Bootstrapping measurements show the short-circuit's effect on the canonical path.
- Automated tooling: `cargo test --release` as above; no profiler.
- Dynamic testing: none against a running node.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | A Bootstrapping node pays `O(tips × depth)` on every block with no pruning to bound `tips`, and the #193 cache cannot serve `maxvalid_bg` | Denial of Service | Low | Medium | Open |
| LB-002 | Under the #193 cache a reorg recomputes every tip's divergence (≈105 µs per tip, 2.1 s at 20,000 tips), bounded by consecutive attacker wins at `120·α²` per hour | Denial of Service | Informational | — | Open |
| LB-003 | Spec deviation: the engine re-runs the fork choice on a block that extends the local chain, which the spec's `on_block` short-circuits; observable only in the Bootstrapping state | Consensus | Informational | — | Open |

### Measurements

Times are one `receive_block` call, in milliseconds, at `k = 2160` unless noted; "tips" is the count after the spam build (the local chain and the competing branch add two). "fork block" is the steady-state value (the insert after the reorg; the pre-reorg insert agrees within noise except where both are shown).

**Online state, `CHAIN = k`**

| tips | stage | fork block | canonical block | reorg | canonical build (2160 blocks) |
|---|---|---|---|---|---|
| 0 | baseline | 0.28 | 0.42 | 0.70 | 109 |
| 0 | S-001 | 0.22 | 0.13 | 0.56 | 137 |
| 0 | S-001 + S-003 | 0.08 | **0.017** | 0.68 | **1** |
| 1,000 | baseline | 60.4 / 81.3 | 120.1 | 150.0 | 111 |
| 1,000 | S-001 | 0.40 | 0.37 | 106.5 | 134 |
| 1,000 | S-001 + S-003 | 0.19 | 0.14 | 94.2 | 1 |
| 4,000 | baseline | 278 / 380 | 564 | **761** | 110 |
| 4,000 | S-001 | 0.64 | 0.57 | **375** | 131 |
| 4,000 | S-001 + S-003 | 0.46 | 0.48 | 360 | 1 |
| 20,000 | baseline | not run (spam build would take hours) | | | |
| 20,000 | S-001 | 3.3 / 2.5 | 2.6 | **2133** | 120 |
| 20,000 | S-001 + S-003 | 2.1 / 2.6 | 2.1 | 2075 | 1 |
| 4,000 (`k = 120`) | baseline | 10.5 / 14.8 | 21.9 | 27.5 | 0 |
| 4,000 (`k = 120`) | S-001 | 0.38 | 0.39 | 20.2 | 0 |
| 4,000 (`k = 120`) | S-001 + S-003 | 0.36 | 0.38 | 18.0 | 0 |

**Bootstrapping state, `CHAIN = 2k = 4320`, 1,000 tips (half of them more than `k` below the tip)**

| stage | fork block | canonical block | reorg | spam build (1,000 tips) |
|---|---|---|---|---|
| baseline | 220 / 225 | 216 | 215 | 153 s |
| S-001 | 210 / 224 | **0.004** | 351 | 150 s |
| S-001 + S-003 | 205 / 213 | 0.005 | 338 | 148 s |

Reading the tables:

- Baseline per-tip cost in Online is about 70 µs per tip per fork block (two `lca` walks of ~1,080 lookups each: `maxvalid_mc` and, when the LIB moves, `prunable_forks`) and twice that for a canonical block; the reorg adds the `lca`/`walk_back_to_block` of the reorg itself and lands at 2.7× the fork-block cost.
- S-001 makes the fork and canonical paths flat in `tips` (0.1 µs per tip, three hash lookups) and leaves the reorg at one recompute: 106 ms / 1,000 tips, 375 / 4,000, 2,133 / 20,000 — 100–107 µs per tip, i.e. one `lca` walk of `depth ≈ k/2` per tip at ~95 ns per `rpds` lookup. It is about half the baseline reorg because the baseline also re-walks every tip in `prunable_forks` when the LIB moves.
- S-003 removes the `k`-deep `nth_ancestor` walk from every block: 0.13 → 0.017 ms with no tips, 0.17 ms off every fork block (0.25 → 0.08) and canonical block (0.37 → 0.14 at 1,000 tips). With tips present the remaining canonical cost is the `prunable_forks` scan, 0.1 µs per tip. Building a 2,160-block chain drops from 110–137 ms to 1 ms.
- In Bootstrapping nothing is flat: 205–225 ms per fork block at 1,000 tips in all three stages, because `maxvalid_bg` (`lib.rs:81-115`) runs for every non-extending block and, for the ~500 tips more than `k` below `cmax`, walks `cmax` and the candidate back to the density slot (`walk_back_before`, `:316-325`, ~2,000 lookups each) on top of the `lca`. The S-001 short-circuit takes the canonical block from 216 ms to 0.004 ms (no LIB movement, no prune scan in this state).

### LB-001 · A Bootstrapping node pays `O(tips × depth)` on every block with no pruning to bound `tips`, and the #193 cache cannot serve `maxvalid_bg`

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | `consensus/cryptarchia-engine/src/lib.rs:81-115` (`maxvalid_bg`), `:316-325` (`walk_back_before`), `:60-66` (`State::lib`, Bootstrapping arm: LIB fixed), `:517-531` (`update_lib`: no pruning while the LIB does not move) |
| Status | Open |

**Description**

In the Bootstrapping state the LIB is the block the node started from (`State::lib`, `:60-66`) and never advances, so `update_lib` never prunes (`:520-521`) and every fork the node accepts stays a tip until `online()`. For each block that does not extend the local chain (`fork_choice()` at `:470`), `maxvalid_bg` iterates every tip: an `lca` walk (`:94-96`, ~`depth` lookups), and for every tip that diverges more than `k` blocks below `cmax`, two `walk_back_before` walks from `cmax` and from the candidate down to `lca.slot + s_gen` (`:106-108`), each ~`length − lca.length` lookups. Measured: 220 ms per fork block at 1,000 tips with `k = 2160` and a 4,320-block chain, versus 60 ms for the same tip count in Online (3.7×), and the per-tip cost grows with the chain length above the LCA, not only with `k`.

The #193 cache does not help here, for two reasons the item asks about. First, `maxvalid_bg` compares each candidate against a moving `cmax`, not against the local chain: once `cmax` has switched to a fork, `lca(cmax, chain)` is between two forks and the cached divergence against the local chain is the wrong quantity (in Online this is harmless because at most one fork can be both admissible and longer, #193 Method; in Bootstrapping the density branch can make `cmax` switch several times, #39 LB-002). Second, the density branch needs `lca.slot` (to place the `s_gen` window) and then the chain length at that slot on both sides; the cache holds only the LCA's *length*. Both could be cached per tip against the local chain (`(lca_length, lca_slot, local_density_after_lca, tip_density_after_lca)` — the two densities are fixed once both chains have passed `lca.slot + s_gen`), but that is only correct if the rule is restated as a comparison of each fork against the local chain, which is #39 S-001's proposal and a spec change.

**Exploit scenario**

The attacker of #193 LB-001 (one lottery win → one valid block on every parent with the same UTXO root) targets a node that is bootstrapping from a peer set the attacker is part of. Because the LIB is pinned, every block since the bootstrap start is a valid parent, not just the last `k`: a node that has been bootstrapping for a day at `f = 1/30` has ~2,900 parents, and each attacker win yields that many tips at one PoL verification each for the victim (#193: ~1 ms). At 1,000 tips each further block costs the victim 0.2 s of engine time; at 4,000 (two wins) about 0.9 s extrapolating linearly, i.e. slower than the 1 s slot, so the node falls behind the chain it is trying to catch and never reaches the `prolonged_bootstrap_period` condition to go Online. Requires stake (a win) and the ability to reach the victim with the fork blocks, hence Medium difficulty; affects only nodes in the Bootstrapping state, hence Low.

**Recommendation**
- *Short term*: bound the tip count in Bootstrapping independently of the LIB: prune forks whose divergence point is more than `k` blocks below the local tip even though the LIB does not move (they can never be selected by the longest-chain branch and, for the density branch, the spec's own §Fork Pruning is keyed on `B_imm`, so this needs a documented deviation or a Bootstrapping-specific bound); and cap `walk_back_before` at `s_gen` steps from the LCA rather than walking from the tip.
- *Long term*: restate the Bootstrap rule as "each fork against the local chain" (#39 S-001) so that the per-tip cache of #193 can carry `(lca_length, lca_slot, densities)` and the state becomes `O(tips)` per block like Online. Until then, keep the Bootstrapping window short.

**References**: `fork-choice.md` §Bootstrap Fork Choice Rule; `cryptarchia-v1-protocol.md` §Commit, §Fork Pruning; #39 LB-002 and S-001; #140; #193 LB-001.

### LB-002 · Under the #193 cache a reorg recomputes every tip's divergence (≈105 µs per tip, 2.1 s at 20,000 tips), bounded by consecutive attacker wins at `120·α²` per hour

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Denial of Service |
| Target | #193 Appendix B `recompute_tip_div` (called from the reorg branch of `receive_block_with_canonical_change` and from `online()`); unmodified `lib.rs:474-498` and `:574-600` for the baseline |
| Status | Open (design note for the prototype, not a defect in the shipped code) |

**Description**

When the local chain moves to another branch the prototype rebuilds `tip_div` with one `lca` walk per tip. Measured at `k = 2160`: 106 ms at 1,000 tips, 375 ms at 4,000, 2,133 ms at 20,000, linear at 100–107 µs per tip (the walk covers `k/2` blocks in this tree; it is `depth` in general). The unmodified engine's reorg at 4,000 tips is 761 ms, so the prototype halves it rather than adding to it, but it is the one place where the prototype's cost is still proportional to `tips × depth`.

How often an attacker can force it: a reorg needs a competing branch that becomes strictly longer than the local chain. With the #193 LB-001 attacker, two consecutive lottery wins with no honest block between them suffice: the first win puts a sibling next to the honest tip (same length, not adopted), the second extends that sibling (longer, adopted, depth-1 reorg). Wins arrive at `3600·f = 120` per hour; the probability that a given win is the attacker's and the next is too is `α²` for stake share `α`, so the rate is `120·α²` per hour: `0.012` at `α = 1 %`, `0.3` at `5 %`, `1.2` at `10 %`, `4.8` at `20 %`, `13` at `33 %`. At 20,000 tips that is 2.6 s of engine time per hour at 10 % stake and 28 s per hour at 33 %; natural depth-1 reorgs from propagation delay add a few per hour. For comparison, the same attacker spends each single win on thousands of fork blocks at 0.6–3.3 ms each under the prototype (LB-001 of #193 remains the lever, now bounded), so the reorg path is not where the budget goes.

**Exploit scenario**

None beyond the arithmetic above; the prototype is not deployed. If it is adopted, the residual risk is a 2 s stall per reorg on a node that has accumulated 20,000 tips, which the pruning of stale forks at each LIB advance already limits to tips younger than `k` blocks.

**Recommendation**
- *Short term* (for the design): keep the recompute as is; document the bound. If the reorg cost matters, make `tip_div` relative to the LCA of old and new local chains: tips whose cached divergence is below `lca.length` keep it (their LCA with the new chain is unchanged), and only tips diverging above the LCA on the old branch need a walk, which is bounded by the reorg depth times the tips on that branch. For a depth-1 reorg that is a handful of tips instead of all.
- *Long term*: the `(slot, entropy_contribution)` admission priority of #193 S-002 bounds the rate at which fork blocks reach the engine in the first place.

**References**: #193 S-001 and LB-001; `fork-choice.md` §Online Fork Choice Rule.

### LB-003 · Spec deviation: the engine re-runs the fork choice on a block that extends the local chain, which the spec's `on_block` short-circuits; observable only in the Bootstrapping state

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Consensus |
| Target | `consensus/cryptarchia-engine/src/lib.rs:469-470` (`apply_header` then unconditional `fork_choice()`), versus `cryptarchia-v1-protocol.md` §Chain Maintenance (`c_loc' = B if parent(B) = c_loc`) |
| Status | Open |

**Description**

Item 5. The spec's `on_block` sets `c_loc' = B` whenever `parent(B) = c_loc` and runs `fork_choice` only otherwise, in both fork-choice modes: the rule is written above the `if fork_choice_rule = ONLINE` branch and does not distinguish. The engine runs `fork_choice()` on every block (`:470`). In the Online state the outcomes coincide (#193 Method: after one block at most one fork is both admissible and longer, and the local chain is now longer than before, so `maxvalid_mc` returns the extension). In the Bootstrapping state they can differ: `maxvalid_bg` is a tournament whose result depends on the tip order (#39 LB-002 gives a tree `L`, `A`, `B` where `A` beats `L` on density, `L` beats `B`, and `B` beats `A` on length). After `L` is extended by one block the code re-runs the tournament and may end on `A` or `B`; the spec keeps `L`. So the code without the short-circuit can move the local chain *because* an honest block extended it, which is not what the spec says and is the less stable of the two behaviours. The prototype's short-circuit (`extends_local` in Appendix B of #193) applies in both states and is therefore the spec-conformant one; it also gives the 216 ms → 0.004 ms canonical-block figure in Bootstrapping.

Answer to the item: apply the short-circuit in both states. The spec requires it; the only outcomes it changes are the order-dependent ones of #39 LB-002, and it changes them in the direction of keeping the chain the node is on; the fork-block path in Bootstrapping is unaffected either way (LB-001).

**Exploit scenario**

None practical: constructing a cyclic tree needs a branch denser than the honest chain over `s_gen` slots (#39 LB-002). Recorded as a deviation so that the short-circuit is adopted as a spec rule, not as an optimisation.

**Recommendation**
- *Short term*: adopt the `extends_local` short-circuit for both states, with a test in `lib.rs`'s test module that a tip-extending block never changes the tip in Bootstrapping on #39 LB-002's `bootstrap_order` tree.
- *Long term*: with #39 S-001 (compare against the local chain only) the tournament order disappears and the two behaviours coincide everywhere.

**References**: `cryptarchia-v1-protocol.md` §Chain Maintenance; `fork-choice.md` §Bootstrap Fork Choice Rule (first-seen tie-break); #39 LB-002.

### Item 1 · The chain-service and chain-network suites (read, not run)

`chain-service` consumes the engine outcome in `try_apply_block_with_state_retention` (`lib.rs:480-503`): `pruned_blocks` go to `prune_ledger_states` and the storage index, `reorged_blocks` to the mempool/transaction reorg path (`service/mod.rs:862-907`) and `newly_canonical_blocks` only to `log_newly_canonical_blocks` (`service/mod.rs:829-833,912-950`, an INFO-level log that fetches each block from storage). For a block that extends the local tip the unmodified engine already produces `reorged_blocks = []` and `newly_canonical_blocks = [B]` (`lib.rs:474-498`: the LCA of the old and new local chain is the old tip), and the prototype returns exactly that pair from its `extends_local` branch. The one assertion in the suites on this path, `tests/mod.rs:98-103` (`reorged_blocks.is_empty()` after a straight chain), holds under the prototype. `states.rs:230-340` and `:410-500` rebuild an engine from the recovery state by replaying blocks from the LIB and compare `tip()`, `lib()` and individual `branches().get(..)` entries; those compare `Branches`, whose `PartialEq` is hand-written over `branches`, `tips` and `lib` (`lib.rs:164-171`), so the prototype's extra fields do not enter. What was **not** checked and needs the suites: `Cryptarchia: PartialEq` is derived and now includes `tip_div` and `lib_tail`, so any test comparing two engines whole after different histories (a replay versus the live engine) could see a difference in the cache even when the trees agree; and the chain-network tests that drive `Cryptarchia` through IBD with `online()` calls. Both are in follow-up #605's scope together with the suites themselves.

## 5. Suggestions (non-security)

### S-001 · Incremental LIB: a `k+1` queue of the canonical tail replaces the `k`-deep walk (prototype, measured)

| | |
|---|---|
| Target | `consensus/cryptarchia-engine/src/lib.rs:60-74` (`State::lib`), `:517-531` (`update_lib`), `:677-682` (`online`) |

#193 S-003, item 4. The prototype (Appendix B, 105 lines on top of #193's diff) keeps `lib_tail: rpds::QueueSync<Id>` holding the local chain from the LIB (front) to the tip (back), at most `k + 1` entries. A tip-extending block enqueues and, past `k + 1`, dequeues (`O(1)` amortised, persistent so the per-block engine clone in `chain-service/src/service/mod.rs:804` stays `O(1)`); a reorg and `online()` rebuild it with one walk of at most `k` parents; `State::lib` in Online reads the front. The 26 engine tests pass, including the LIB and pruning tests. Measured at `k = 2160`: canonical block 0.42 → 0.017 ms with no tips, fork block 0.25 → 0.08 ms; with tips the 0.17 ms saving is constant and the remaining cost is the `O(tips)` prune scan (0.1 µs per tip). The 2,160-block canonical build drops from 110 ms to 1 ms, which also shortens IBD replay proportionally. The reorg pays one extra `k` walk (~0.2 ms), invisible next to LB-002's recompute.

### S-002 · Cap the density walk at `s_gen` steps from the LCA

`walk_back_before` (`lib.rs:316-325`) starts at the tip and walks down to `lca.slot + s_gen`, i.e. `length − lca.length − density` lookups, most of which are outside the window it is measuring. Walking *up* from the LCA is not possible without child pointers, but the density can be computed as `length_at_or_before(density_slot)` by a binary search over cached `(slot → length)` checkpoints of the local chain, or simply by walking from the tip only until `slot <= density_slot` and stopping at `s_gen` steps of *counted* blocks. Either bounds the per-tip cost in Bootstrapping to `O(depth + s_gen)` instead of `O(length)`; it does not remove the `O(tips)` factor (LB-001).

### S-003 · Keep the measurement harness

`fork_reorg_592` (Appendix C) is parameterised by environment variables and asserts the reorg outcome; committed under `consensus/cryptarchia-engine/tests/` with `#[ignore]` it would give the maintainers a one-command regression check for all three paths at `k = 2160`.

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

## Appendix B — S-003 prototype diff (incremental LIB), on top of #193 Appendix B

Against `consensus/cryptarchia-engine/src/lib.rs` at `bfcae04d` with the #193 S-001 diff applied first (it applies cleanly at this commit with `git apply`). Evidence for the measurements, not a proposed upstream change.

```diff
@@ -12,7 +12,7 @@
 pub use config::*;
 use lb_log_targets::cryptarchia;
 use lb_utils::bounded::UpperBoundedVec;
-use rpds::{HashTrieMapSync, HashTrieSetSync};
+use rpds::{HashTrieMapSync, HashTrieSetSync, QueueSync};
 use thiserror::Error;
 pub use time::{Epoch, EpochConfig, Slot};
 
@@ -63,13 +63,13 @@
     {
         match cryptarchia.state {
             Self::Bootstrapping => cryptarchia.branches.lib,
+            // PROTOTYPE (#592 S-003): the LIB is the oldest entry of the
+            // maintained canonical tail instead of a `k`-deep parent walk.
             Self::Online => cryptarchia
-                .branches
-                .nth_ancestor(
-                    &cryptarchia.local_chain,
-                    cryptarchia.config.security_param().get().into(),
-                )
-                .id(),
+                .lib_tail
+                .peek()
+                .copied()
+                .unwrap_or(cryptarchia.branches.lib),
         }
     }
 }
@@ -176,6 +176,9 @@
     /// PROTOTYPE (#193): for every tip, the length of its lowest common
     /// ancestor with the local chain (its divergence height).
     tip_div: HashTrieMapSync<Id, u64>,
+    /// PROTOTYPE (#592 S-003): the local chain from the LIB (front) to the
+    /// tip (back), at most `k + 1` entries.
+    lib_tail: QueueSync<Id>,
 }
 
 #[derive(Clone, Debug)]
@@ -459,6 +462,7 @@
             config,
             state,
             tip_div: HashTrieMapSync::new_sync().insert(id, length),
+            lib_tail: QueueSync::new_sync().enqueue(id),
         }
     }
 
@@ -482,6 +486,29 @@
         self.tip_div = tip_div;
     }
 
+    /// PROTOTYPE (#592 S-003): rebuild the canonical tail (LIB..=tip) with one
+    /// walk of at most `k` parents. Called on a reorg and on `online()`.
+    fn rebuild_lib_tail(&mut self) {
+        let k = u64::from(self.config.security_param().get());
+        let mut ids = Vec::with_capacity(k as usize + 1);
+        let mut current = &self.local_chain;
+        ids.push(current.id);
+        let mut n = 0;
+        while n < k {
+            let Some(parent) = self.branches.parent(current) else {
+                break;
+            };
+            current = parent;
+            ids.push(current.id);
+            n += 1;
+        }
+        let mut tail = QueueSync::new_sync();
+        for id in ids.into_iter().rev() {
+            tail.enqueue_mut(id);
+        }
+        self.lib_tail = tail;
+    }
+
     /// Apply the given block.
     ///
     /// On success, returns the pruned/reorged blocks resulting from the update.
@@ -542,6 +569,18 @@
             self.fork_choice().clone()
         };
 
+        // PROTOTYPE (#592 S-003): maintain the canonical tail. Extension is
+        // O(1); a move to another branch rebuilds it with one `k` walk.
+        if extends_local {
+            self.lib_tail.enqueue_mut(id);
+            let max_len = self.config.security_param().get() as usize + 1;
+            while self.lib_tail.len() > max_len {
+                self.lib_tail.dequeue_mut();
+            }
+        } else if self.local_chain.id != old_local_chain.id {
+            self.rebuild_lib_tail();
+        }
+
         // Before `update_lib` which may prune blocks,
         // collect the reorged blocks in the old local chain.
         let (reorged_blocks, newly_canonical_blocks) = if self.local_chain.id == old_local_chain.id
@@ -763,6 +802,8 @@
         self.state = State::Online;
         // PROTOTYPE (#193): the cache is only trusted in the Online state.
         self.recompute_tip_div();
+        // PROTOTYPE (#592 S-003): as is the canonical tail.
+        self.rebuild_lib_tail();
         // Update the LIB to the current local chain's tip
         let pruned_blocks = self.update_lib();
         (self, pruned_blocks)
```

## Appendix C — Measurement harness `fork_reorg_592.rs`

Placed at `consensus/cryptarchia-engine/tests/fork_reorg_592.rs` of the scratch checkout for the runs and removed afterwards. Run as `LB592_K=2160 LB592_TIPS=4000 cargo test --release -p logos-blockchain-cryptarchia-engine --test fork_reorg_592 -- --nocapture`; `LB592_CHAIN`, `LB592_STATE=bootstrap`, `LB592_REORG=0` and `LB592_SAMPLE` select the other configurations.

```rust
use std::{
    num::NonZero,
    time::{Duration, Instant},
};

use lb_utils::math::NonNegativeRatio;
use logos_blockchain_cryptarchia_engine::{Config, Cryptarchia, Slot, State, UncleSlots};

type Id = [u8; 32];

fn id(n: u64) -> Id {
    let mut r = [0u8; 32];
    r[..8].copy_from_slice(&n.to_le_bytes());
    r[31] = 1;
    r
}

fn env_u64(key: &str, default: u64) -> u64 {
    std::env::var(key).ok().and_then(|v| v.parse().ok()).unwrap_or(default)
}

fn config(k: u32) -> Config {
    Config::new(
        NonZero::new(k).unwrap(),
        NonNegativeRatio::new(1, 30.try_into().unwrap()),
        1f64.try_into().expect("1 > 0"),
        NonZero::new(12).unwrap(),
    )
}

fn ms(d: Duration) -> f64 {
    d.as_secs_f64() * 1e3
}

#[test]
fn fork_reorg_592() {
    let k = env_u64("LB592_K", 2160) as u32;
    let tips = env_u64("LB592_TIPS", 4000);
    let chain_len = env_u64("LB592_CHAIN", u64::from(k));
    let bootstrap = std::env::var("LB592_STATE").as_deref() == Ok("bootstrap");
    let reorg = env_u64("LB592_REORG", 1) == 1;
    let sample = env_u64("LB592_SAMPLE", 10) as usize;
    let state = if bootstrap { State::Bootstrapping } else { State::Online };

    let genesis = [0u8; 32];
    let mut engine = Cryptarchia::from_lib(genesis, config(k), state, 0.into(), 0, UncleSlots::default());
    let mut next = 1u64;
    let mut canonical: Vec<Id> = vec![genesis];
    let mut canonical_slot: Vec<u64> = vec![0];

    // Canonical chain, one block every 30 slots.
    let t = Instant::now();
    for h in 1..=chain_len {
        let b = id(next);
        next += 1;
        let slot = 30 * h;
        engine
            .receive_block(b, canonical[(h - 1) as usize], Slot::from(slot), UncleSlots::default())
            .expect("canonical block");
        canonical.push(b);
        canonical_slot.push(slot);
    }
    let t_chain = t.elapsed();
    assert_eq!(engine.tip(), canonical[chain_len as usize]);

    // Competing branch diverging k/2 blocks below the tip (so it stays within
    // the k-deep admissibility bound after one canonical extension), same
    // length as the local chain (tie, not adopted). Its blocks reuse the
    // canonical slots.
    let d = chain_len - u64::from(k) / 2;
    let mut fork_tip = canonical[d as usize];
    let mut fork_len = 0u64;
    if reorg {
        for j in 1..=(chain_len - d) {
            let b = id(next);
            next += 1;
            let slot = canonical_slot[(d + j) as usize];
            engine
                .receive_block(b, fork_tip, Slot::from(slot), UncleSlots::default())
                .expect("fork chain block");
            fork_tip = b;
            fork_len = j;
        }
        assert_eq!(engine.tip(), canonical[chain_len as usize], "tie must not be adopted");
    }

    // Spam tips: parents spread uniformly over heights 2..CHAIN-1, one block each.
    let t = Instant::now();
    let mut last: Vec<Duration> = Vec::with_capacity(sample);
    for i in 0..tips {
        let h = 2 + (i * (chain_len - 3)) / tips.max(1);
        let b = id(next);
        next += 1;
        let slot = canonical_slot[h as usize] + 1;
        let t1 = Instant::now();
        engine
            .receive_block(b, canonical[h as usize], Slot::from(slot), UncleSlots::default())
            .expect("spam tip");
        let e = t1.elapsed();
        if tips - i <= sample as u64 {
            last.push(e);
        }
    }
    let t_spam_total = t.elapsed();
    let spam_avg = if last.is_empty() { 0.0 } else { ms(last.iter().sum::<Duration>()) / last.len() as f64 };
    let n_tips = engine.branches().branches().count();

    // One more fork block.
    let h = chain_len / 3;
    let b = id(next);
    next += 1;
    let t1 = Instant::now();
    engine
        .receive_block(b, canonical[h as usize], Slot::from(canonical_slot[h as usize] + 2), UncleSlots::default())
        .unwrap();
    let t_fork = t1.elapsed();

    // One canonical extension.
    let b = id(next);
    next += 1;
    let tip_slot = canonical_slot[chain_len as usize] + 30;
    let t1 = Instant::now();
    let out = engine
        .receive_block_with_canonical_change(b, engine.tip(), Slot::from(tip_slot), UncleSlots::default())
        .unwrap();
    let t_canonical = t1.elapsed();
    assert_eq!(out.newly_canonical_blocks, vec![b]);
    assert!(out.reorged_blocks.is_empty());
    let local_tip = b;

    // Reorg: two blocks on the competing branch make it longer than the local
    // chain (which is now CHAIN+1 long).
    let mut t_reorg = Duration::ZERO;
    let mut reorged = 0usize;
    let mut newly = 0usize;
    if reorg {
        let b1 = id(next);
        next += 1;
        engine
            .receive_block(b1, fork_tip, Slot::from(tip_slot), UncleSlots::default())
            .unwrap();
        assert_eq!(engine.tip(), local_tip, "tie must not be adopted");
        let b2 = id(next);
        next += 1;
        let t1 = Instant::now();
        let out = engine
            .receive_block_with_canonical_change(b2, b1, Slot::from(tip_slot + 30), UncleSlots::default())
            .unwrap();
        t_reorg = t1.elapsed();
        assert_eq!(engine.tip(), b2, "reorg must adopt the longer branch");
        reorged = out.reorged_blocks.len();
        newly = out.newly_canonical_blocks.len();
        assert_eq!(reorged as u64, chain_len - d + 1);
        assert_eq!(newly as u64, fork_len + 2);
    }

    // After the reorg: one more fork block and one more canonical extension.
    let b = id(next);
    next += 1;
    let t1 = Instant::now();
    engine
        .receive_block(b, canonical[h as usize], Slot::from(canonical_slot[h as usize] + 3), UncleSlots::default())
        .unwrap();
    let t_fork_after = t1.elapsed();
    let b = id(next);
    let t1 = Instant::now();
    engine
        .receive_block(b, engine.tip(), Slot::from(tip_slot + 60), UncleSlots::default())
        .unwrap();
    let t_canonical_after = t1.elapsed();

    println!(
        "LB592 k={k} chain={chain_len} state={} tips={n_tips} reorg={reorg} | build: chain {:.0} ms, spam {:.1} s (last {sample} inserts avg {spam_avg:.3} ms) | fork_block {:.3} ms | canonical {:.3} ms | reorg {:.3} ms (reorged={reorged}, newly_canonical={newly}) | fork_block_after {:.3} ms | canonical_after {:.3} ms | lib={:?}",
        if bootstrap { "bootstrap" } else { "online" },
        ms(t_chain), t_spam_total.as_secs_f64(), ms(t_fork), ms(t_canonical), ms(t_reorg),
        ms(t_fork_after), ms(t_canonical_after), &engine.lib()[..4],
    );
}
```

Raw output of the eighteen runs (baseline, S-001, S-001 + S-003; the `lib=` field is the first four bytes of the LIB id, `[3,0,0,0]` = block 3 after two canonical extensions past `k`, `[0,..]` = genesis in Bootstrapping):

```
baseline  k=2160 chain=2160 online    tips=2     | fork_block 0.281 | canonical 0.420   | reorg 0.702    | after: fork 0.326   canonical 0.553
baseline  k=2160 chain=2160 online    tips=1002  | fork_block 60.42 | canonical 120.06  | reorg 150.01   | after: fork 81.34   canonical 162.19
baseline  k=120  chain=120  online    tips=4002  | fork_block 10.51 | canonical 21.93   | reorg 27.55    | after: fork 14.79   canonical 29.87
baseline  k=2160 chain=2160 online    tips=4002  | fork_block 278.4 | canonical 564.2   | reorg 761.4    | after: fork 380.4   canonical 743.8   (spam build 705 s)
baseline  k=2160 chain=4320 bootstrap tips=1002  | fork_block 220.1 | canonical 216.4   | reorg 215.0    | after: fork 224.7   canonical 219.4   (spam build 153 s)
S-001     k=2160 chain=2160 online    tips=2     | fork_block 0.253 | canonical 0.132   | reorg 0.555    | after: fork 0.224   canonical 0.130
S-001     k=2160 chain=2160 online    tips=1002  | fork_block 0.506 | canonical 0.367   | reorg 106.47   | after: fork 0.401   canonical 0.275
S-001     k=120  chain=120  online    tips=4002  | fork_block 0.384 | canonical 0.385   | reorg 20.21    | after: fork 0.421   canonical 0.420
S-001     k=2160 chain=2160 online    tips=4002  | fork_block 0.641 | canonical 0.569   | reorg 374.58   | after: fork 0.694   canonical 0.601   (spam build 1.6 s)
S-001     k=2160 chain=2160 online    tips=20002 | fork_block 3.297 | canonical 2.571   | reorg 2133.2   | after: fork 2.454   canonical 2.542   (spam build 27 s)
S-001     k=2160 chain=4320 bootstrap tips=1002  | fork_block 209.8 | canonical 0.004   | reorg 351.3    | after: fork 223.6   canonical 0.005
S-001+S-003 k=2160 chain=2160 online  tips=2     | fork_block 0.083 | canonical 0.017   | reorg 0.682    | after: fork 0.082   canonical 0.007
S-001+S-003 k=2160 chain=2160 online  tips=1002  | fork_block 0.213 | canonical 0.141   | reorg 94.19    | after: fork 0.191   canonical 0.112
S-001+S-003 k=120  chain=120  online  tips=4002  | fork_block 0.363 | canonical 0.377   | reorg 18.04    | after: fork 0.358   canonical 0.388
S-001+S-003 k=2160 chain=2160 online  tips=4002  | fork_block 1.255 | canonical 0.480   | reorg 360.25   | after: fork 0.459   canonical 0.380
S-001+S-003 k=2160 chain=2160 online  tips=20002 | fork_block 2.149 | canonical 2.084   | reorg 2075.0   | after: fork 2.599   canonical 3.681
S-001+S-003 k=2160 chain=4320 bootstrap tips=1002 | fork_block 205.0 | canonical 0.005  | reorg 338.2    | after: fork 212.7   canonical 0.019
```
