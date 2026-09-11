# Audit Report — Fork choice rule vs spec, reorg depth bound, per-fork ledger state growth

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/39`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `consensus/cryptarchia-engine`, `ledger`, `services/chain/chain-service`, `services/chain/chain-network`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `cryptarchia-v1-protocol.md`, `fork-choice.md` (in full); `cryptarchia-v1-bootstr-sync.md` §Setting the Fork Choice Rule, §Prolonged Bootstrap Period (by section)
Date: `2026-09-11` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: the fork choice rules, the `k` reorg bound and the per-fork state pruning match the specification; the gaps are a per-block cost that grows with the number of fork tips a leader can cheaply create, and a Bootstrap-mode fork choice whose result depends on the hash order of block IDs rather than on the spec's first-seen order.
- Findings: `0` critical · `0` high · `1` medium · `1` low · `2` informational
- Key themes: "cost of block processing is linear in fork tips, and fork tips are cheap for a leader", "fork choice is order-sensitive during bootstrap", "silent stall after a divergence deeper than k"
- Must-fix before launch: none; LB-001 should be measured against the production `k` and the PoL proving cost before a decision.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `consensus/cryptarchia-engine/src/lib.rs` | block tree, `maxvalid_bg`, `maxvalid_mc`, LIB update, stale and immutable pruning, LCA, uncle selection (read, not audited) |
| `consensus/cryptarchia-engine/src/config.rs` | `k`, `f`, `s_gen`, uncle window derivation |
| `ledger/src/lib.rs`, `ledger/src/cryptarchia/mod.rs`, `ledger/src/cryptarchia/block_density.rs` | per-block `LedgerState` map, header application, slot ordering, proof-of-leadership public inputs, state pruning |
| `services/chain/chain-service/src/lib.rs`, `src/service/mod.rs`, `src/states.rs` | wallclock check, ledger-then-engine apply order, ledger-state pruning, recovery replay |
| `services/chain/chain-network/src/lib.rs`, `src/sync/orphan_handler.rs`, `src/sync/tip_poll.rs` | what happens to a block whose parent was pruned (fork deeper than `k`) |
| `Cargo.toml`, `rpds` 1.2.1 | hasher behind the tip set, feature resolution |

**Out of scope**

Proof-of-leadership circuit and verifier (`zk/`, `lb_pol`, `rust-rapidsnark`), transaction validation, the block download and IBD streams (#146, #135), the cost of epoch-state synthesis per block (#147), bootstrap-mode memory retention (#140), reorg effects on mempool, wallet and blend (#50, PR #133), slot and epoch arithmetic (#38), the rejected-block cache (#143). Third-party crates assumed correct: `rpds`, `archery`, `serde`, `tokio`.

**Assumptions**

- The specification at the commit above is correct unless stated otherwise.
- `k = 2160` and `f = 1/30` as in `cryptarchia-v1-protocol.md` §Constants. The deployment templates use `k = 120` (`deployment/ceremony/genesis/testnet/deployment-template.yaml:25`, `devnet/...:25`) and `k = 30` (`standalone/...:25`); where a rating depends on `k` both values are given.
- Release builds: `overflow-checks` off, `arithmetic_side_effects`, `indexing_slicing`, `unwrap_used`, `expect_used` allowed (issue #19, re-verified in `Cargo.toml` at this commit).

## 3. Method

- Manual review of the in-scope paths, working through issue `#39` under parent `#1`. All three checklist items were verified against both spec and code.
- Spec conformance against `fork-choice.md` §Bootstrap Fork Choice Rule, §Online Fork Choice Rule, §Definitions; `cryptarchia-v1-protocol.md` §Latest Immutable Block, §Block Header Validation (steps 5, 6, 7, 8), §Chain Maintenance, §Commit, §Fork Pruning; `cryptarchia-v1-bootstr-sync.md` §Setting the Fork Choice Rule.
- Automated tooling: two throw-away integration tests written against a scratch copy of the tree and run with `cargo test --release` (rustc 1.98.1, release profile of the workspace, `lto = "fat"`), on an Apple M-series laptop:
  - `fork_spam_bench`: a 2160-block canonical chain in the Online state, then N single-block forks whose parents are spread uniformly over the LIB subtree; measures the time to apply one more fork block and one more canonical block.
  - `bootstrap_order`: the same block tree built with 40 different block-ID sets in the Bootstrapping state; records the tip visiting order and the selected tip.
  Both are described in LB-001 and LB-002 and are reproducible from the description alone.
- Dynamic testing: none against a running node.

**Checked and ruled out**

- `maxvalid_mc` (`lib.rs:119-141`) is `online_fork_choice` of the spec: `m = cmax.length - lca.length` is `depth_max`, `m <= k` selects the longest-chain branch, `cmax.length < chain.length` is the spec's strict `depth_max < depth_fork` (both depths share the LCA), forks with `m > k` are skipped.
- `maxvalid_bg` (`lib.rs:81-115`) is `bootstrap_fork_choice`: the density window is `(lca.slot, lca.slot + s_gen]` inclusive on the right, as the spec's diagram note says; `walk_back_before(..).length` is an absolute length but both operands share the LCA, so the comparison equals the spec's `density` comparison. `s_gen` (`config.rs:107-113`) is `⌊k / 4f⌋`.
- Ties against the local chain: both rules start with `cmax = local_chain` and use strict comparisons, so the local chain wins a tie against any single fork. In the Online state this makes the result independent of the tip order (proof sketch: after a fork-choice run no fork admissible w.r.t. the local chain is strictly longer than it, and one new block cannot create two such forks). The Bootstrap state is not order-independent: LB-002.
- The reorg bound is `k` inclusive, as the spec's `depth_max <= k`. In the Online state the LIB is the `k`-th ancestor of the tip (`lib.rs:60-74`), `update_lib` (`lib.rs:517-531`) prunes forks whose LCA with the tip is shallower than the LIB (`prunable_forks`, `lib.rs:574-600`, `lca.length < deepest_div_block` with `deepest_div_block = lib.length`) and every ancestor of the LIB (`prune_immutable_blocks`, `lib.rs:639-649`); the tree is therefore exactly the LIB subtree and a block on a deeper fork fails with `ParentMissing` at `apply_header` (`lib.rs:239-242`). This is the spec's `is_ancestor(B_imm, B)` variant of validation step 8.
- The LIB never moves backwards: the new tip is strictly longer and within `k` of the old one, so its `k`-th ancestor is a descendant of the old LIB.
- Per-fork state is pruned: engine pruning returns `PrunedBlocks`; `process_block` (`service/mod.rs:773-910`) stores the block and the immutable-block index first, then `prune_ledger_states` (`chain-service/src/lib.rs:534-551`) removes each pruned block's `LedgerState`, and `delete_stale_blocks_from_storage` (`service/mod.rs:237-242`) removes stale fork blocks from the store. On a storage failure the candidate clone is dropped and no state is mutated (`service/mod.rs:804-837`). The `online()` transition prunes the same way (`chain-service/src/lib.rs:553-565`). While Bootstrapping nothing is pruned, by design (#140 covers the consequences).
- Arrival-order independence of state: a block's `LedgerState` is a function of its parent's state and the block only (`ledger/src/lib.rs:183-208` and `309-355`); `BlockDensity` is a set of slots (`block_density.rs:9-14`, test `test_mark_occupied_slots` states "slot order doesn't matter"); the engine `Branch` is a function of the parent branch. Nothing in either reads the tree or wall time except the wallclock slot check.
- Recovery replays only the `(LIB, tip]` path from storage (`chain-service/src/lib.rs:889-909`, `965-1070`), so a restart never re-applies fork blocks; the replayed state equals the state that was persisted.
- Panics: the three `expect`s in fork choice and pruning (`lib.rs:96`, `133`, `484`, `593`) require two branches without a common ancestor in the tree; every branch descends from the root inserted by `from_lib` (`lib.rs:207-226`) and `Branches::parent` returns `None` at the root, so `lca` cannot return `None`. `checked_add(1).expect(..)` on the length (`lib.rs:248-251`) needs 2^64 blocks.
- Overflow: `u64::from(lca.slot) + s_gen` (`lib.rs:106`) wraps in release only if the LCA's slot is within `s_gen` of `u64::MAX`; every accepted block's slot is `<= current_slot` (`chain-service/src/lib.rs:440-446`).
- The wallclock check (spec step 6) is applied before the ledger and the engine (`chain-service/src/lib.rs:440-446`); the ledger's strict slot check (`ledger/src/cryptarchia/mod.rs:264-269`) runs before the engine's, so LB-004 has no effect on the node.
- A block whose parent was pruned (a fork deeper than `k`) never enters the engine: LB-003 describes the observable behaviour.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Per-block cost of fork choice and pruning is linear in the number of fork tips, and one lottery win lets a leader add one tip per block of the LIB subtree | Denial of Service | Medium | Medium | Open |
| LB-002 | Spec deviation: Bootstrap fork choice visits tips in block-ID hash order, not first-seen order, and its result depends on that order | Determinism | Low | High | Open |
| LB-003 | A node that has diverged more than `k` blocks in the Online state never rejoins and raises no alert | Consensus | Informational | High | Open |
| LB-004 | Spec deviation: the engine accepts a child block at its parent's slot | Data Validation | Informational | High | Open |

### LB-001 · Per-block cost of fork choice and pruning is linear in the number of fork tips, and one lottery win lets a leader add one tip per block of the LIB subtree

| | |
|---|---|
| Severity | Medium at the spec's `k = 2160`; Low at the deployed `k = 120` |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | `consensus/cryptarchia-engine/src/lib.rs:92-113` (`maxvalid_bg`), `:129-139` (`maxvalid_mc`), `:277-298` (`Branches::lca`), `:587-599` (`prunable_forks`), `:232-268` (`apply_header`); `ledger/src/cryptarchia/mod.rs:503-510` (`try_apply_proof`) |
| Status | Open |

**Description**

Both fork choice rules iterate over every tip of the tree and compute the LCA of the tip with `cmax` (`lib.rs:92-97`, `129-134`). `Branches::lca` (`lib.rs:277-298`) walks parent pointers one block at a time, so its cost is the depth of the fork's divergence point below the longer chain's tip. `receive_block_with_canonical_change` runs the fork choice on every block (`lib.rs:470`), including a block that merely extends the local chain, where the spec's `on_block` short-circuits (`cryptarchia-v1-protocol.md` §Chain Maintenance, `c_loc' = B if parent(B) = c_loc`). When the LIB advances, `prunable_forks` (`lib.rs:587-599`) computes the LCA of every non-canonical tip a second time. The cost of processing one block is therefore `O(tips × divergence depth)`, twice for a canonical block.

Nothing bounds the number of tips a single leader can create. The engine accepts any header whose parent is in the tree and whose slot is not below the parent's (`apply_header`, `lib.rs:239-246`). The Proof of Leadership binds the block to its parent through `latest_root` (`ledger/src/cryptarchia/mod.rs:503-510`), so one proof covers one parent, but the lottery result for a slot does not depend on the parent: a note that wins slot `σ` wins it on top of every block of the same epoch. A leader who wins one slot can therefore produce a valid block for every block of the LIB subtree whose slot is below `σ` (up to `k` canonical blocks plus every fork block already present), each with its own proof. Each such block is a new tip. Past wins can be used too: the wallclock check only requires `σ <= now` (`chain-service/src/lib.rs:440-446`). Neither spec nor code limits blocks per `(slot, leader)`.

Each tip also retains a `LedgerState` (`ledger/src/lib.rs:153-156`, `211-214`) until the LIB passes its divergence point, at most `k` canonical blocks later. The state is persistent-structure based, so the retained delta per fork block is small (the 120-entry `fee_window` array at `ledger/src/cryptarchia/mod.rs:234`, two `EpochState` values, and the path copies of the `states`, `utxos`, `occupied_slots` and `block_slots` maps); it was not measured. The engine's `Branch` is about 100 bytes plus a trie path.

Measured with `fork_spam_bench` (Online state, 2160 canonical blocks, forks attached to canonical blocks at uniformly spread heights, release build):

| fork tips | apply one fork block | apply one canonical block (fork choice + LIB advance + prune scan) | average cost of the N-th fork block while building |
|---|---|---|---|
| 0 | 0.15 ms | 0.20 ms | — |
| 500 | 45 ms | 91 ms | 23 ms |
| 1000 | 83 ms | 167 ms | 44 ms |
| 2000 | 122 ms | 245 ms | 76 ms |
| 4000 | 250 ms | 250 ms (fork choice only: in this run one fork already extended the tip, so the block did not advance the LIB and no prune scan ran; with the scan, about 500 ms) | 136 ms |

The cost is linear in the tip count, 60–90 µs per tip per fork-choice pass at this depth (≈1080 parent lookups per tip). Block processing in chain-service is a single sequential loop (`service/phases/following.rs:61-80`), so this time is added to the latency of every block, honest or not.

Scaling: with `f = 1/30` a leader holding a fraction `α` of the stake wins about `2880 α` slots per day. At `k = 2160` each win yields up to ~2160 tips that live up to `k` canonical blocks (18 h), so `α = 1 %` sustains tens of thousands of tips, at which point the measured per-tip cost puts every block at several seconds of fork choice on every node; `α = 0.1 %` sustains a few thousand tips and a few hundred milliseconds per block. The attacker pays one Groth16 PoL proof per block. At the deployed `k = 120` the subtree has 120 parents, the LCA walk is ~60 steps and a tip lives one hour, so the same stake sustains roughly 300× less work on the victim; the rating is Low there.

**Exploit scenario**

A leader with `α = 1 %` of the aged stake and a few cores for proving keeps a list of every block in the LIB subtree. For every slot its note wins (past or present, `σ <= now`), it signs and publishes one block per parent whose slot is `< σ`, each carrying a fresh proof against that parent's `latest_root` and an empty body. Every node accepts each block (valid proof, known parent, slot in range), stores it, keeps a `LedgerState` for it and adds a tip. After a day every node holds tens of thousands of tips inside its LIB subtree, and `receive_block` spends seconds in `maxvalid_mc` for each block it processes, so block processing lags the chain. No node is crashed and no chain split results; the impact is degraded liveness of every node for as long as the attacker keeps winning slots, at a cost of one proof per block.

**Recommendation**

- *Short term*: skip the fork choice when the block extends the local chain in the Online state, as the spec's `on_block` does; cache the LCA-with-tip (or the divergence height) in `Branch` so that a fork's admissibility is `O(1)`; bound blocks per `(slot, leader_key)` at the network layer, since the spec gives one lottery win per slot per leader.
- *Long term*: keep, per tip, the height of its divergence from the canonical chain and update it on reorg, so that both fork choice and pruning are `O(tips)` with no parent walk; raise upstream whether the spec should cap valid blocks per slot per leader (S-002).

**References**: `fork-choice.md` §Online Fork Choice Rule; `cryptarchia-v1-protocol.md` §Chain Maintenance, §Fork Pruning; related: #147 (cost of epoch-state synthesis per block), #140 (bootstrap-state retention).

### LB-002 · Spec deviation: Bootstrap fork choice visits tips in block-ID hash order, not first-seen order, and its result depends on that order

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Determinism |
| Target | `consensus/cryptarchia-engine/src/lib.rs:92-113` (`maxvalid_bg` loop), `:160` (`tips: HashTrieSetSync<Id>`), `:270-272` (`branches()`); `Cargo.toml:260` (`rpds` with `default-features = false`) |
| Status | Open |

**Description**

`fork-choice.md` §Bootstrap Fork Choice Rule iterates `for c_fork in forks` and comments that the strict inequality "ensure[s] we choose first-seen chain as our tie break": the order of `forks` is the order in which the node saw them. The code iterates the `tips` set of an `rpds::HashTrieSetSync` (`lib.rs:160`, `270-272`), whose order is the order of the hashed block IDs. With `rpds` built without its `std` feature (`Cargo.toml:260`; no other crate in `Cargo.lock` depends on `rpds`, so feature unification does not enable it) the hasher is `BuildHasherDefault<SipHasher>` with fixed keys (`rpds-1.2.1/src/utils/mod.rs:5-7`), so the order is a deterministic function of the block IDs, equal on every node, but unrelated to arrival order.

The order matters because the Bootstrap rule is not a pairwise-consistent maximum. Two forks are compared by length when they diverge within `k` of `cmax` and by density after their LCA otherwise, and these comparisons can form a cycle. Example verified with `bootstrap_order` (`k = 10`, `f = 1/10`, `s_gen = 25`):

- `L`: blocks at slots 1..=44, then every fifth slot to 200 (24 blocks in the window (20, 45]).
- `A`: forks at `L20`, blocks at slots 21..=45, no tail (25 blocks in the window).
- `B`: forks at `A38` (7 blocks below `A`'s tip), blocks at slots 40, 42, 44, then every second slot to 100 (21 blocks in the window, and longer than `A`).

`A` beats `L` (density 25 > 24), `L` beats `B` (24 > 21), and `B` beats `A` (within `k`, longer). Across 40 block-ID sets of this tree the visiting orders `[A, L, B]`, `[B, L, A]`, `[L, A, B]`, ... all occurred, and the final tip was `L` 15 times, `A` 14 times and `B` 11 times. The spec's rule is also order-sensitive, but under the spec the order is the node's own arrival order; under the code it is the hash of the IDs, so two nodes with the same tree agree while a node's choice differs from what the spec's first-seen rule would give.

The determinism across nodes is an accident of the feature set. If any dependency ever enables `rpds/std`, `DefaultBuildHasher` becomes `std::collections::hash_map::RandomState` (`rpds-1.2.1/src/utils/mod.rs:3-4`), the tip order becomes per-process random, and two bootstrapping nodes holding the same tree can commit to different chains, with the choice changing across restarts. Nothing in the build pins the feature or tests the order.

**Exploit scenario**

An adversary who can present a bootstrapping node with two forks of its chain whose pairwise comparisons form a cycle with the honest chain (a dense short branch `A` and a long sparse branch `B` off `A`) makes the node's choice depend on the IDs of the forks, which the adversary controls through the block contents. Under the current hasher this lets the adversary pick, by grinding IDs, which of the three chains the node adopts among those that beat the local chain pairwise; it does not let the adversary make an honest node adopt a chain that loses to the honest chain in every comparison. Building a branch denser than the honest chain in the `s_gen` window after the fork requires a majority of the active stake in that window, so the practical impact is limited to periods of low honest participation.

**Recommendation**

- *Short term*: iterate tips in a defined order (insertion order, or `(length desc, id)`), and add a test that the Bootstrap result is stable under permutation of the tip set for trees where it is meant to be; pin `rpds` without `std` with a comment on why, or construct the maps with an explicit fixed `BuildHasher`.
- *Long term*: define the Bootstrap rule as a comparison against the local chain only, or as a tournament with a specified order (S-001).

**References**: `fork-choice.md` §Bootstrap Fork Choice Rule (tie-break comment); `cryptarchia-v1-protocol.md` §Chain Maintenance; related: #40 (determinism of consensus-critical code).

### LB-003 · A node that has diverged more than `k` blocks in the Online state never rejoins and raises no alert

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Consensus |
| Target | `consensus/cryptarchia-engine/src/lib.rs:129-139` (`maxvalid_mc`), `:239-242` (`ParentMissing`); `services/chain/chain-network/src/lib.rs:660-672` (orphan enqueue on `ParentMissing`), `:845-868` (`should_process_block`), `:433-473` (downloaded blocks); `services/chain/chain-service/src/service/mod.rs:1282-1289` (`log_process_block_error`) |
| Status | Open |

**Description**

The checklist asks what the node does on a fork deeper than `k`. In the Online state the answer is: ignore it, forever, with no dedicated signal. Because the tree is exactly the LIB subtree, a block on such a fork is rejected as `ParentMissing` by the engine (`lib.rs:239-242`), logged at `error` level per block by chain-service (`service/mod.rs:1282-1289`), and handed to the orphan downloader by chain-network (`chain-network/src/lib.rs:660-672`). Downloaded ancestors at or before the LIB's slot are dropped as `OlderThanLib` and recorded in the rejected cache (`chain-network/src/lib.rs:440-451`, `855-857`); their descendants are then refused at enqueue because their parent is rejected (`orphan_handler.rs:159-170`). The tip watchdog (`tip_poll.rs:27-69`) re-submits the best peer tip about once per expected block interval while the local tip lags, so the sequence repeats indefinitely at that cadence.

This matches `fork-choice.md` §Online Fork Choice Rule ("Ignore this fork") and the bootstrap spec, which only switches to the Bootstrap rule at startup (`cryptarchia-v1-bootstr-sync.md` §Setting the Fork Choice Rule). A running node that was on the minority side of a partition lasting more than `k` blocks stays on its own chain until an operator restarts it after `T_offline` or with the bootstrap flag. There is no metric or log line that says "the network's tip diverges from ours deeper than `k`"; the operator sees a stream of `ParentMissing` errors and a lagging height.

**Exploit scenario**

Not exploitable by itself. Impact: after a network partition longer than `k` blocks (18 h at spec parameters, 1 h at `k = 120`), every node on the minority side keeps producing and following its own chain and never rejoins without operator action, and nothing tells the operator that this is the state the node is in.

**Recommendation**

- *Short term*: count `ParentMissing` rejections whose parent is not in the tree and whose slot is above the LIB slot, and emit a warning plus a metric when the watchdog's best peer tip is ahead by more than `k` and cannot be applied.
- *Long term*: raise upstream whether a running node should re-enter the Bootstrap rule on such a signal (PR #132 lists the spec side of this).

**References**: `fork-choice.md` §Online Fork Choice Rule; `cryptarchia-v1-bootstr-sync.md` §Setting the Fork Choice Rule; related: PR #132 (report for #42), #146.

### LB-004 · Spec deviation: the engine accepts a child block at its parent's slot

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Data Validation |
| Target | `consensus/cryptarchia-engine/src/lib.rs:244-246` (`apply_header`); `ledger/src/cryptarchia/mod.rs:264-269` (`update_epoch_state`) |
| Status | Open |

**Description**

Validation step 5 of `cryptarchia-v1-protocol.md` requires `header.slot > parent.slot`. `apply_header` rejects only `parent_branch.slot > slot` (`lib.rs:244`), so a child at its parent's slot is accepted by the engine. The ledger rejects `slot <= self.slot` (`ledger/src/cryptarchia/mod.rs:264-269`) and runs first in `try_apply_block_with_state_retention` (`chain-service/src/lib.rs:461-489`), so no such block reaches the engine in the node. The engine's own invariant is nonetheless weaker than the spec's: `walk_back_before` and the density count treat a same-slot chain of blocks as several blocks in one slot, and the uncle-window arithmetic in `select_uncles` assumes strictly increasing slots. Any other user of the engine (the sync block provider in `services/chain/chain-service/src/sync/block_provider.rs` reads `Cryptarchia<HeaderId>` directly; the crate is public) inherits the weaker check. PR #132 raised the same point as a spec suggestion; it is recorded here as the code-side deviation.

**Exploit scenario**

Not reachable through the node. Impact limited to a future caller of the engine that does not run the ledger check first.

**Recommendation**

- *Short term*: change the check to `parent_branch.slot >= slot`.
- *Long term*: a unit test in the engine that a same-slot child is rejected.

**References**: `cryptarchia-v1-protocol.md` §Block Header Validation step 5.

## 5. Suggestions (non-security)

### S-001 · `fork-choice.md`: the iteration order of `forks` is unspecified but load-bearing

| | |
|---|---|
| Target | `fork-choice.md` §Bootstrap Fork Choice Rule (`for c_fork in forks`, tie-break comment); `cryptarchia-v1-protocol.md` §Chain Maintenance (`fork_choice(c_loc, F_T', k, s)`) |

Both pseudocode rules iterate an unordered set `F_T` and rely on strict inequalities for the tie-break "first-seen". The Bootstrap rule's result depends on the order (LB-002 gives a three-chain cycle), so the spec should state the order (arrival order of the tips, or a total order on block IDs) or restate the rule so that the result is order-independent, for example by comparing each fork against `c_loc` only and adopting the best. The Online rule is order-independent given the invariant maintained by `on_block`; the spec could state that invariant.

### S-002 · `cryptarchia-v1-protocol.md`: no bound on valid blocks per slot per leader

| | |
|---|---|
| Target | `cryptarchia-v1-protocol.md` §Block Header Validation, §Fork Pruning |

A lottery win at slot `σ` yields a valid Proof of Leadership for every parent of the same epoch with slot below `σ` (the proof's chain inputs are the epoch state and the parent's `latest_root`). The validation rules accept every such block, and forks are pruned only at commit. This is the root of LB-001. If the intent is one block per win, the spec should say so and give validators a rule to enforce it (for example, at most one block per `(slot, leader_key)` accepted into the tree, with the first-seen kept); if not, the specification should state that the block tree is unbounded below `k` and leave the cost to implementers.

### S-003 · Engine: run the fork choice only when the block does not extend the local chain

| | |
|---|---|
| Target | `consensus/cryptarchia-engine/src/lib.rs:470` |

The spec's `on_block` sets `c_loc' = B` when `parent(B) = c_loc` without running the fork choice, and the Online invariant (Method, "Ties against the local chain") makes the two equivalent. Applying the short-circuit removes one `O(tips × depth)` pass from every canonical block; the prune scan on LIB advance remains.

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
