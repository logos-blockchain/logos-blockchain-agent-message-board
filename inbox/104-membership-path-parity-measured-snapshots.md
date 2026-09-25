# Audit Report · Membership Merkle rebuild, second pass: path parity up to 2^20, measured snapshot memory, per-transition cost with the node's types, and where the root should come from

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/104`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `c4c86be18c58b5b09c3650e93871c8cfb624885b` · component(s): `blend/crypto/src/merkle.rs` (+ `rs-merkle-tree 0.1.0`), `ledger/src/mantle/sdp/{mod.rs,rewards/blend/current_epoch.rs}`, `ledger/src/cryptarchia/mod.rs`, `core/src/sdp/mod.rs`, `services/blend/src/{lib.rs,mode.rs,membership/*,core/mod.rs,edge/*,broadcast/mod.rs,orchestrator/instance.rs}`, `services/chain/{chain-service,chain-leader}`
Specs: `https://github.com/logos-co/logos-lips` @ `d788723992a805b395f377de6e6cf59859b47168` · read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `bedrock-service-declaration-protocol.md` (all three in full); `proof-of-quota.md` §Public values, §Witness, §Constraints, §Pseudocode; `blend-protocol.md` §Network (Bootstrapping, Minimal Network Size, Fallback, Maintenance), §Quota (Core Quota to Quota Application), §Proof of Quota (by section)
Date: `2026-09-25` · author: `Claude Code (research agent)` · status: `final`

This is the second pass on #104. The first report (`processed/104-membership-merkle-rebuild.md`, PR #120, at `b8c3c54f`; findings filed as #309 and #310) proved root parity of a bottom-up builder and left four things open: path parity and the switch of the blend path builder, a real memory measurement, a per-transition cost with the node's own types, and the decision on the old S-003. All four are answered here.

**What changed since `b8c3c54f`.** 52 commits. `blend/crypto/src/merkle.rs` and `ledger/src/mantle/sdp/rewards/blend/current_epoch.rs` are byte-identical; the builder has **not** been replaced upstream. `services/blend/src/membership/{service.rs,chain.rs}` changed only in logging (INFO to DEBUG) and in taking the Ed25519 key directly from `ProviderId` (`service.rs:106-108`). `core/src/sdp/mod.rs` gained `AsRef` impls, samples and the new codec module; `Declaration` and `Locators` are unchanged. `ledger/src/mantle/sdp/mod.rs` changed only in log targets and test helper names. Every line number below is at `c4c86be1`.

---

## 1. Summary

- Overall assessment: the bottom-up builder is a drop-in replacement for both call sites. Its root **and every `get_proof_for_key` output** are equal to the current builder's for 1, 2, 3, 4, 5, 7, 8, 9, 1,000, 1,023, 1,024, 1,025, 4,097 and **2^20** keys (all 1,048,576 paths compared), and it is 22.6 times faster at 10^5 (0.74 s against 16.7 s here) and 23.5 times at 2^20 (7.4 s against 172.8 s, with 64 MB of level vectors against a 1.7 GB peak). With the node's own types, one epoch transition at 10^5 active declarations costs 17.5 s in `SdpLedger::try_apply_header`, 98% of it the tree; syncing costs that once per past epoch (about 4.3 hours of tree building per synced year on this machine, about 12 hours on a Raspberry Pi 5, extrapolated). The memory estimate behind 104-LB-002 (#310) does not hold: `Multiaddr` is `Arc<Vec<u8>>`, so snapshots share the locator bytes. Measured, a maximal declaration costs 3,400 B once in the ledger map and 506 B per snapshot copy (48 MiB per snapshot at 10^5), not 2.8 KB times three; `Arc<Locators>` would save about 85 B of the 506. Nothing caches the root anywhere, and the root is not stored in `EpochState` (the first report said it was). The Blend side builds the full tree twice per node per epoch, and one of the two builds is thrown away unread.
- Findings: 0 critical · 0 high · 0 medium · 1 low · 1 informational
- Key themes: full trees built where only a root, or nothing, is read; per-copy cost from inline 304-byte `Declaration`s, not from locator bytes; one snapshot-scoped cache would serve the ledger, every fork and the Blend service.
- Must-fix before launch: none new. The existing #309 (104-LB-001) remains the fix that matters; LB-001 below multiplies its cost.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `blend/crypto/src/merkle.rs:74-228` | `new_from_ordered`, `root`, `get_proof_for_key`, `compute_selectors`, `sort_nodes_and_build_merkle_tree`; bottom-up replacement and parity |
| `rs-merkle-tree 0.1.0` `src/tree.rs:69-242`, `src/stores/memory_store.rs` | zero table, `add_leaves`, `proof` |
| `ledger/src/mantle/sdp/rewards/blend/current_epoch.rs:93-231` | where the ledger builds the tree and where the root goes |
| `ledger/src/mantle/sdp/mod.rs:40, 178-219, 380-422, 589-633` | snapshot functions, header application |
| `ledger/src/cryptarchia/mod.rs:64-166, 257-456, 702-714` | when snapshots are taken, `epoch_state_for_slot` |
| `core/src/sdp/mod.rs:108-270, 404-424, 500-501`; `multiaddr 0.18.2` `src/lib.rs:44-46, 361-372`; `kms/keys/src/keys/ed25519/public.rs:14` | what a `Declaration` and a `Locator` hold |
| `services/blend/src/{lib.rs:218-290, mode.rs:49-89, orchestrator/instance.rs:115-122, membership/service.rs:28-81, membership/chain.rs:133-227, core/mod.rs:395-402, 578-640, edge/mod.rs:241-249, edge/current_epoch.rs:206, broadcast/mod.rs:152-162}` | every Blend-side build and what each consumer reads |
| `services/chain/chain-service/src/service/mod.rs:334-350, 694-744, 893-940`; `services/chain/chain-leader/src/lib.rs:432-459, 657-664` | other snapshot and build sites |

**Out of scope**

The 2^20 cap and its panic (#92 LB-001), per-fork repetition of the whole transition, the proposer's double transition and the per-slot re-synthesis (report #178 LB-001, LB-003), pre-proof synthesis cost (report #147 LB-001), reward-note insertion (report #178 LB-002), pinning or inlining `rs-merkle-tree` (#432). They are cited where they meet this issue and not re-derived. No devnet and no multi-epoch sync were run (see Method). Third-party crates assumed correct: `rs-merkle-tree` (behaviour measured), `jf-poseidon2`, `ark-ff`/`ark-bn254`, `rpds`, `hashbrown`, `multiaddr`, `ed25519-dalek`.

**Assumptions**

Repo-level facts of #19 hold. The circuit's witness convention (siblings leaf-first, selectors root-first, `true` means the current node is the right child) is the one established against the circuit fixture by report #51 (second pass, Appendix B) and re-confirmed below. Timings are from a shared 4-core `Intel Xeon @ 2.10GHz` VM (x86_64, `rustc 1.98.1`, workspace release profile: fat LTO, `codegen-units = 1`) with four other jobs running; ratios, hash counts and byte counts are machine-independent, absolute seconds are not.

## 3. Method

- Specs first, as listed in the header. Checked against them: the tree shape (`proof-of-quota.md` §Constraints Step 2.1: depth 20, `zk_id`s sorted as naturals, empty leaves `0` appended after the sort), the witness (`core_path` of length 20 with `core_path_selectors`, §Witness), `core_root` as a public input (§Public values), `N = SDP(e)` (`blend-protocol.md` §Proof of Quota). The specs say nothing about where or how often the root is computed, so no spec deviation is possible on items 2 to 4; item 1 is checked for parity with the current builder and with the circuit fixture.
- Re-verification of #309 and #310 at `c4c86be1` (git log and diff of every file they cite since `b8c3c54f`, see the header note).
- Experiment 1 (`blend/crypto`, scratch clone, `cargo test --release`): a bottom-up builder working in `Fr` with `ZkHasher::compress`, a root-only variant, and a path extractor that emits the same `[(Fr, bool); 20]` layout as `get_proof_for_key`. A parity test compares the root and every key's path against `MerkleTree::new` + `get_proof_for_key` at the sizes listed in the Summary, a timing test runs 10^4 and 10^5, and a fixture test recomputes the PoQ circuit fixture root (`zk/proofs/poq/src/lib.rs` `test_core_node_full_flow`) under four sibling/selector orderings. Code: Appendix B.1; output: Appendix C.1.
- Experiment 2 (`ledger`, scratch clone, `cargo test --release`): a counting `#[global_allocator]` in the ledger's unit-test binary. It inserts 10^5 real `Declaration`s (distinct Ed25519 provider keys, distinct `zk_id`s, 8 locators of exactly 329 bytes each, built through the wire path `Locator::try_from(Vec<u8>)`) into a real `SdpLedger`, takes the two snapshots with `SdpLedger::active_declarations`, calls `declarations()`, checks by pointer whether locator bytes are shared, then times `SdpLedger::try_apply_header` across an epoch boundary (real `RealProofsVerifier`), the tree build alone, and a synthesized transition through `cryptarchia::LedgerState::epoch_state_for_slot`. Reported as live heap bytes (allocator counter) and RSS (`/proc/self/statm`). Run at 10^5 with 8 locators, 10^5 with 1 locator, and 10^4 with 8. Code: Appendix B.2; output: Appendix C.2.
- Dynamic testing on a node: none. A devnet sync of many epochs is out of reach here (shared machine, no multi-node harness); sync cost is per-transition cost measured with the node's types times the number of transitions, stated as an extrapolation.

Checked and ruled out:

| Check | Evidence | Result |
|---|---|---|
| Builder replaced upstream since `b8c3c54f` | `git log b8c3c54f..c4c86be1 -- blend/crypto/src/merkle.rs ledger/src/mantle/sdp/rewards/blend/current_epoch.rs` is empty | not replaced; #309 stands unchanged |
| The ledger stores the root in `EpochState` (first report S-001) | `EpochState` fields `cryptarchia/mod.rs:64-107`: no root; `finalize` puts `zk_root` only into the `RealProofsVerifier` of `TargetEpochState` (`current_epoch.rs:158-163, 217-231`) | false; nothing outside the verifier holds it |
| Anything caches the root between ledger and Blend, or across forks | grep of every `sort_nodes_and_build_merkle_tree` / `MerkleTree::new` in non-test code: `current_epoch.rs:196` and `membership/service.rs:53` only, both build from scratch | no cache anywhere |
| Ledger and Blend build over different sets | the ledger at boundary `e → e+1` builds over `last_epoch_state.active_declarations` (`current_epoch.rs:132-152`), the Blend service at the start of `e` over the same `EpochState`'s set (`membership/chain.rs:148-162`), both the `Arc` frozen at boundary `e-1 → e` (`cryptarchia/mod.rs:326-349`) | same set, same root, built one epoch apart |
| Blend builds during initial sync | orchestrator and leader wait for `Online` (`services/blend/src/lib.rs:227-230`, `chain-leader/src/lib.rs:435-438`) | no: during IBD only the ledger builds |
| Bottom-up paths follow the circuit's convention | fixture test (Appendix C.1): root reproduced only with siblings leaf-first and selectors root-first, which is what `compute_selectors` (`merkle.rs:200-216`) and the bottom-up extractor emit | yes; matches report #51 |
| Zero padding differs between builders | `rs-merkle-tree` `zeros[0] = Node::ZERO` (`tree.rs:72`) and bottom-up `zeros[0] = Fr::ZERO`; equal roots and paths at non-powers of two (1,000, 1,023, 1,025, 4,097) | identical, and equal to the spec's `0` leaves |

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | The Blend service builds the full membership tree twice per node per epoch, inline on async tasks, and the orchestrator's build is discarded unread | Denial of Service | Low | Medium | Open |
| LB-002 | Each active-declarations snapshot re-materialises every 304-byte `Declaration` (48 MiB per snapshot at 10^5); the locator bytes are already shared, so the `Arc<Locators>` fix of 104-LB-002 (#310) saves about a sixth | Denial of Service | Informational | Medium | Open |

### 4.1 Item 1: path parity, and whether the Blend path builder can switch

Results (Appendix C.1):

| Keys | Root equal | Paths compared | Current build | Bottom-up build (levels kept) |
|---|---|---|---|---|
| 1, 2, 3, 4, 5, 7, 8, 9 | yes | all | 0.30 to 1.4 ms | 0.26 to 0.31 ms (zero table dominates) |
| 1,000 / 1,023 / 1,024 / 1,025 | yes | all | 155 to 164 ms | 7.2 to 7.9 ms |
| 4,097 | yes | all | 672 ms | 32.7 ms |
| 2^20 (full tree, no padding) | yes | all 1,048,576 | 172.8 s, 1.7 GB peak RSS | 7.36 s, 64 MB of levels |

Timing, root only (Appendix C.1): 10^4: 1.63 s against 71.9 ms (22.7x); 10^5: 16.73 s against 741 ms (22.6x). Keeping all levels and extracting one path costs 788 ms at 10^5. The ratio exceeds the tree height (20) because the bottom-up builder also skips the `Fr`/byte round trip and the `HashMap` store of `add_leaves` (`tree.rs:104-170`).

The Blend path builder can switch. What a replacement must preserve, all covered by the test:

1. The output layout: sibling hashes leaf level first (`rs-merkle-tree` fills `proof[level]` from level 0, `tree.rs:212-231`), selectors root level first (`compute_selectors`, `merkle.rs:200-216`). This mixed order is what the circuit consumes (fixture test, and report #51 second pass Appendix B); it must be kept exactly.
2. Absent keys return `None` (`merkle.rs:154`). With sorted leaves in `levels[0]`, a binary search replaces the 10^5-entry `sorted_key_indices` `HashMap` (`merkle.rs:98-102`), and adjacent-equal comparison replaces its duplicate check (`:103-107`).
3. The `EmptyKeySet` and `TooManyKeys` checks (`:91-96`) stay.

A bottom-up tree that keeps its levels needs `2n` field elements (6.4 MB at 10^5, 64 MB at 2^20) and serves any path with 20 indexed reads, which is what item 4 needs.

### 4.2 Item 2: cost per epoch transition with the node's types, and sync

Measured at 10^5 active declarations, 8 maximal locators (Appendix C.2):

| Step (where it runs) | Time here | Transient heap |
|---|---|---|
| `sdp.active_declarations(e+2)` snapshot in `update_epoch_state` case 2 (`cryptarchia/mod.rs:345-348`) | 115 ms | +48 MiB retained |
| `SdpLedger::try_apply_header` across the boundary (`sdp/mod.rs:380-422` → `current_epoch.rs:93-186`) | **17.51 s** | 156 MiB peak |
| of which the tree alone (`current_epoch.rs:195-198`) | 17.22 s | 135 MiB peak |
| same tree plus one path (what `membership_info_from_epoch_state` does, `service.rs:53-66`) | 18.02 s | |
| bottom-up root instead (Experiment 1) | 0.74 s | about 3 MB |
| synthesized transition `epoch_state_for_slot` (`cryptarchia/mod.rs:702-714`) | 147 ms | 67 MiB peak, dropped |

With one locator per declaration the tree and header numbers are the same (16.1 s, 16.5 s): the cost is the hashing, independent of what else a declaration carries.

Sync extrapolation (not measured end to end). During IBD the Blend services and the leader are not running (§3), so a fresh node pays, per past boundary, the snapshot plus the header transition: about 17.6 s here at 10^5, of which about 1.1 s would remain with a bottom-up root. Testnet epochs are 36,000 one-second slots (`security_param: 120`, `slot_activation_coeff: 1/30`, ten base periods; `deployment/ceremony/genesis/testnet/deployment-template.yaml:25-31,109`), 2.4 per day:

| Epochs synced | Current builder, here | Bottom-up, here | Current, Raspberry Pi 5 (x3.0, from #92's 51.2 s) |
|---|---|---|---|
| 72 (one month) | 21 min | 1.3 min | 1.0 h |
| 876 (one year) | 4.3 h | 16 min | 12.4 h |
| per boundary at 2^20 | 173 s | 7.4 s | about 8.6 min |

The factor 3.0 is #92's Pi figure over the same tree measured here (51.2 s / 17.2 s); it assumes the rest of the transition scales the same way.

Does anything cache the root? No (§3). Once the node is online, the same root is computed, for one declaration set: by the Blend service at the start of epoch `e` (twice, LB-001), by the ledger at boundary `e → e+1` for every block that crosses it (report #178 LB-001), and twice more on a node that proposes such a block (report #178 LB-003). Forks that diverged after the snapshot `Arc` was created share that `Arc` (`cryptarchia/mod.rs:293-299` moves it unchanged through same-epoch blocks), so a cache attached to the `Arc` would serve all of them; forks from before it hold an equal but distinct set.

### LB-001 · The Blend service builds the full membership tree twice per node per epoch, inline on async tasks, and the orchestrator's build is discarded unread

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | `services/blend/src/lib.rs:233-249, 266-268, 280-290`; `services/blend/src/orchestrator/instance.rs:115-122`; `services/blend/src/mode.rs:49-76`; `services/blend/src/membership/service.rs:50-71`; `services/blend/src/membership/chain.rs:142-162`; `services/blend/src/{core/mod.rs:395-402, edge/mod.rs:241-249, broadcast/mod.rs:154-162}` |
| Status | Open |

**Description**

Every Blend-enabled node runs two membership streams at the same time. The orchestrator subscribes with no ZK key (`lib.rs:233-245`) and keeps consuming the stream for the node's lifetime (`:280-290`); whichever mode service is active subscribes again (core with its ZK key, `core/mod.rs:395-402`; edge and broadcast with `None`, `edge/mod.rs:241-249`, `broadcast/mod.rs:154-160`). Each stream, at the first slot tick of every epoch, calls `membership_info_from_epoch_state` (`chain.rs:158-162`), which builds the whole tree whenever the membership is non-empty, regardless of whether a key was given:

```rust
// services/blend/src/membership/service.rs
50  let zk_info = if nodes.is_empty() {
51      None
52  } else {
53      let zk_tree = sort_nodes_and_build_merkle_tree(&mut nodes, |ZkNode { zk_key, .. }| {
```

What each consumer reads:

| Stream | Uses | Needs a tree? |
|---|---|---|
| orchestrator | `ModeMembership::resolve(..).mode()` only (`lib.rs:268`, `instance.rs:118`); `resolve` looks at `zk` only as `Some`/`None` (`mode.rs:62-75`), which is `nodes.is_empty()` | no |
| broadcast | drops `zk` (`broadcast/mod.rs:162`) | no |
| edge | the root, as a PoQ public input (`edge/current_epoch.rs:206`) | root only |
| core | root and its own path (`core/mod.rs:595-620`) | root + one path |

So every node pays at least one full build that nothing reads (the orchestrator's), a broadcast node pays two, an edge node pays a full `MemoryStore` tree for a root. Both builds run inline in `unfold` stream closures polled by the services' async tasks (`chain.rs:133-226`); nothing uses `spawn_blocking`. At 10^5 declarations each is 18 s here (54 s on a Raspberry Pi 5, from #92), holding two Tokio worker threads at the same moment, the first tick of the epoch, on a 4-core target. This is on top of the ledger's own build and does not touch consensus, but it delays the epoch's first Blend messages (the core service cannot mint a proof of quota before its path exists) by the full build time.

**Exploit scenario**

No attacker action beyond what #309 already needs: with `n` active declarations (honest or funded as in #92 LB-001), every Blend node spends `2 × 20n` Poseidon2 compressions at each epoch start, one of them for a value it throws away, while two runtime workers are blocked. At 10^5 on the target hardware that is about 1.8 minutes of CPU per node per epoch, during which the node's core-mode message schedule for the new epoch has not started.

**Recommendation**

- *Short term*: split `membership_info_from_epoch_state` so the tree is built only when the caller asks for it: the orchestrator and broadcast streams need the membership only; edge needs the root; only core needs the path. Run the build in `spawn_blocking` while it remains expensive.
- *Long term*: take the root and the local path from the frozen snapshot rather than building at all (S-001); with #309's bottom-up builder the remaining cost is one 20-read path lookup.

**References**: `blend-protocol.md` §Proof of Quota (`N = SDP(e)`); #92 report S-003; report #178 item 3 and LB-003 (ledger-side repetitions); #309.

### 4.3 Item 3: real memory of the declaration snapshots

Measured with the node's types (Appendix C.2), live heap bytes from the allocator counter:

| 10^5 declarations | Ledger map (`rpds`, once) | Snapshot 1 (`epoch_state`) | Snapshot 2 (`next_epoch_state`) | `declarations()` | Locator bytes shared with the ledger map |
|---|---|---|---|---|---|
| 8 locators × 329 B | 324.2 MiB (3,400 B/decl) | 48.2 MiB (506 B/decl) | 48.2 MiB (506 B/decl) | 48.2 MiB | yes, all three (same pointer) |
| 1 locator × 329 B | 72.6 MiB (761 B/decl) | 42.9 MiB (450 B/decl) | 42.9 MiB | 42.9 MiB | yes |

RSS moved from 6.6 MiB to 358 MiB after the map and to 471 MiB after both snapshots (8 locators). The first report's figures (about 2.8 KB per declaration times three copies, 810 MB at 10^5) and #92 LB-004's (0.84 GB) are refuted: the resident total with both snapshots is 421 MiB. The ledger copy is larger than their per-declaration estimate (3,400 B, because each locator is a separate `Arc<Vec<u8>>` allocation of 329 + 40 bytes plus the `rpds` node), but it exists once. The snapshots are about 15% of it each.

Why the estimate was wrong: `Locator` wraps `Multiaddr` (`core/src/sdp/mod.rs:118-122`), and `multiaddr 0.18.2` stores `bytes: Arc<Vec<u8>>` (`src/lib.rs:44-46`), so `Declaration::clone` copies eight pointers and bumps eight reference counts. Two side observations: a locator built by parsing a string over-allocates its vector (5,984 B/decl in the ledger map at 10^4 against 3,400 B through the wire path, Appendix C.2), which matters for configuration or genesis paths but not for declarations decoded from blocks (`BinaryDecode for Locator`, `core/src/sdp/mod.rs:259-270` → `Multiaddr::try_from(Vec<u8>)`, `multiaddr` `src/lib.rs:361-372`, exact size); and `declarations()` is cloned at every boundary by the canonical-snapshot logging (S-003).

### LB-002 · Each active-declarations snapshot re-materialises every 304-byte `Declaration` (48 MiB per snapshot at 10^5); the locator bytes are already shared, so the `Arc<Locators>` fix of 104-LB-002 (#310) saves about a sixth

| | |
|---|---|
| Severity | Informational |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | `ledger/src/mantle/sdp/mod.rs:611-633` (`active_declarations`), `:589-603` (`declarations`); `core/src/sdp/mod.rs:404-424` (`Declaration`), `:451-452` (`Declarations`); `ledger/src/cryptarchia/mod.rs:106, 345-348, 421-423, 436-439` |
| Status | Open |

**Description**

`active_declarations` builds a fresh `HashMap<DeclarationId, Declaration>` per service and clones every active declaration into it (`sdp/mod.rs:620-625`). What a clone costs is the inline `Declaration`, 304 bytes (`size_of`, Appendix C.2), plus one 8-pointer `Vec` for `Locators` (64 B at 8 locators), plus the hash-table slot and its load-factor slack: 506 B per declaration measured. About 192 of the 304 bytes are the provider key: `ProviderId` wraps `UnverifiedPublicKey(VerifyingKey)` (`kms/keys/src/keys/ed25519/public.rs:14`), which keeps the decompressed point next to the 32 compressed bytes. The locator bytes (2,632 B at the maximum) are not copied.

104-LB-002 (#310) states that each snapshot deep-copies the locator heap and that `locators: Arc<Locators>` would collapse the copies. At `c4c86be1` the first half is false and the second would save the 64-byte `Vec` per copy: the one-locator run, whose `Vec` is 8 B, measures 450 B per snapshot entry against 506 B. `Arc<Locators>` would remove the 64-byte `Vec` and 16 bytes of inline size (about 21 B with table slack), bringing the 8-locator case to about 420 B, a sixth less. The resident cost of the two snapshots at 10^5 is 96 MiB, not 540 MB.

What does scale: a snapshot is taken at every epoch boundary per fork (`cryptarchia/mod.rs:345-348`, twice after skipped epochs `:421-423, :436-439`), on every synthesized query while the tip lags the boundary (report #178 LB-003, 67 MiB transient per call here), and a full `declarations()` copy is taken for logging (S-003). Each is a 48 MiB allocation at 10^5 and 0.5 GB at 10^6.

**Exploit scenario**

None beyond memory churn. The figure an operator should plan for at 10^5 maximal declarations is about 0.42 GiB resident plus about 50 MiB per transient snapshot, not 0.8 GiB.

**Recommendation**

- *Short term*: re-scope #310. `Arc<Locators>` is not worth a type change on its own. If the snapshot size matters, store `Arc<Declaration>` in the `rpds` map and in the snapshots (a snapshot entry becomes `DeclarationId` plus a pointer, about 40 B plus table slack, roughly 5 MiB at 10^5 instead of 48), or keep only `DeclarationId`s in the snapshot and read the frozen fields at the time of use.
- *Long term*: keep `ProviderId` as the 32 compressed bytes in stored declarations and decompress where a key is used, which removes 160 bytes from every stored and every snapshotted declaration.

**References**: `processed/104-membership-merkle-rebuild.md` LB-002 (#310); #92 report LB-004; report #147 LB-001 (the same snapshot on the pre-proof path).

### 4.4 Item 4: should the Blend service take the root and its path from the ledger's `EpochState`?

Yes, with one adjustment: the value should hang off the frozen snapshot, not be copied into `EpochState` as a plain field.

- The set is the same (§3): the root the Blend service needs at the start of `e` is exactly the root the ledger needs at `e → e+1`, over the same `Arc<Declarations>`.
- `EpochState` is cloned into every per-block ledger state (`cryptarchia/mod.rs:95-97`) and serialised with it. A plain `zk_root` field computed eagerly where the snapshot is created (`:345-348`) would run once per fork crossing the previous boundary, on the header path, a whole epoch before anybody reads it. A lazily filled cache inside the `Arc` (a `OnceLock` next to the declarations, skipped by serde and ignored by `PartialEq`) is filled once per distinct snapshot, by whichever reader comes first, and every fork holding that `Arc` shares it. This is report #178 S-001 and #92 S-003; the measurements here say what to put in it.
- What to cache: the bottom-up levels (`2n` field elements: 6.4 MB at 10^5, 64 MB at 2^20), not only the root. The root is `levels[20][0]`; any path is 20 reads (§4.1); the core service asks for its own path by `zk_id` with a binary search on `levels[0]`. Caching only the root would leave core with one more full O(n) pass per epoch; caching the rs-merkle-tree `MemoryStore` would be 135 MiB at 10^5 and 1.7 GB at 2^20.
- Consensus safety: the root stays a pure function of an immutable set, so a cache keyed by the `Arc` cannot diverge between nodes; after a restart the cache is empty and is rebuilt on first use. The PoQ public input is unchanged, so the circuit and the spec are unaffected.
- Consumers after the change: ledger `finalize` reads the root (`current_epoch.rs:148-163`), the orchestrator and broadcast streams read neither, edge reads the root, core reads root and path (LB-001). With #309 applied the one remaining build costs 0.74 s at 10^5 here, instead of three builds of 17 s per node per epoch (two more on a boundary proposer, one more per fork that re-crosses the boundary).

## 5. Suggestions (non-security)

### S-001 · Serve the root and paths from the frozen snapshot

Wrap `EpochState::active_declarations` (`cryptarchia/mod.rs:106`) in a snapshot type `{ declarations: Declarations, membership: OnceLock<BottomUpTree> }` with `root()` and `path_for(&ZkPublicKey) -> Option<CorePathAndSelectors>`; have `current_epoch.rs:188-199` and `membership/service.rs:50-71` call it. Keep `#[serde(skip)]` and exclude the cache from `PartialEq` on the derive (`cryptarchia/mod.rs:64`). Test: build the snapshot, read the root twice and from a cloned `EpochState`, and assert one build (a counter in test builds).

### S-002 · Replace `add_leaves` inside `MerkleTree` and land the parity test

The parity test of Appendix B.1 (with `parity_full_2_pow_20` ignored by default, 7 s bottom-up plus 173 s for the old builder) is ready to become the regression test for the switch; after the switch, keep the recorded 2^20 root as a constant so the test no longer needs the old builder. The currently ignored `full_keys` test (`merkle.rs:348-362`, "It takes too long") takes about 7 s with a bottom-up builder and can be enabled. This also removes the `rs-merkle-tree` store and most of the surface #432 asks to pin or inline.

### S-003 · The canonical-snapshot diagnostic clones every declaration at each boundary at the default log level

`log_canonical_blend_snapshots` (`chain-service/src/service/mod.rs:694-744`) is gated at INFO (`:912-914`) and calls `sdp_ledger().declarations()` (a copy of every declaration of every service, lapsed ones included) once per boundary and twice for the genesis block, to emit per-provider lines at TRACE (`:652-670`). At 10^5 that is 69 ms and 48 MiB per call here. Iterate `get_declarations_by_service(ServiceType::BlendNetwork)` (`sdp/mod.rs:647-649`) by reference instead, and gate the per-provider loop on TRACE.

### S-004 · `poq_interaction_four_keys` tests three keys

`blend/proofs/src/quota/tests.rs:237-251` calls `generate_inputs::<3>()`, duplicating `poq_interaction_three_keys`. A full 4-leaf level is the one bottom-level shape not otherwise covered by a circuit round trip; use `generate_inputs::<4>()`.

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

## Appendix B · Experiment code

Both files were added to a scratch clone at `c4c86be1` as test-only modules (`#[cfg(test)] #[path = "merkle_probe.rs"] mod probe;` at the end of `blend/crypto/src/merkle.rs`; `#[cfg(test)] mod memprobe;` at the end of `ledger/src/mantle/sdp/mod.rs`). They are evidence, not proposed patches.

### B.1 `blend/crypto/src/merkle_probe.rs` (builder, parity, timing; fixture test abridged)

```rust
use lb_groth16::{AdditiveGroup as _, Fr, fr_from_bytes_unchecked};
use lb_poq::{CORE_MERKLE_TREE_HEIGHT as DEPTH, CorePathAndSelectors};
use super::MerkleTree;
use crate::ZkHasher;

fn compress(l: &Fr, r: &Fr) -> Fr { <ZkHasher as lb_poseidon2::Digest>::compress(&[*l, *r]) }

fn zeros() -> [Fr; DEPTH + 1] {
    let mut z = [Fr::ZERO; DEPTH + 1];
    for i in 1..=DEPTH { z[i] = compress(&z[i - 1], &z[i - 1]); }
    z
}

pub struct BottomUp { levels: Vec<Vec<Fr>>, zeros: [Fr; DEPTH + 1] }

impl BottomUp {
    pub fn new(sorted_keys: &[Fr]) -> Self {
        let zeros = zeros();
        let mut levels = vec![sorted_keys.to_vec()];
        for l in 0..DEPTH {
            let next = levels[l].chunks(2)
                .map(|p| compress(&p[0], p.get(1).unwrap_or(&zeros[l])))
                .collect();
            levels.push(next);
        }
        Self { levels, zeros }
    }
    pub fn root(&self) -> Fr { self.levels[DEPTH][0] }
    /// Same layout as get_proof_for_key: hashes leaf level first,
    /// selectors as compute_selectors (index MSB first).
    pub fn path(&self, index: usize) -> CorePathAndSelectors {
        let mut out = [(Fr::ZERO, false); DEPTH];
        let mut idx = index;
        for (l, slot) in out.iter_mut().enumerate() {
            slot.0 = self.levels[l].get(idx ^ 1).copied().unwrap_or(self.zeros[l]);
            idx >>= 1;
        }
        let mut bits = index;
        for slot in out.iter_mut().rev() { slot.1 = (bits & 1) == 1; bits >>= 1; }
        out
    }
}

pub fn bottom_up_root(sorted_keys: &[Fr]) -> Fr {
    let zeros = zeros();
    let mut level = sorted_keys.to_vec();
    for z in zeros.iter().take(DEPTH) {
        level = level.chunks(2).map(|p| compress(&p[0], p.get(1).unwrap_or(z))).collect();
    }
    level[0]
}

// keys(n): n distinct full-width Fr (splitmix64 + index, times a 248-bit constant), sorted.

fn check_parity(n: usize) {
    let ks = keys(n);
    let tree = MerkleTree::new(ks.clone()).unwrap();           // current builder
    let bu = BottomUp::new(&ks);
    assert_eq!(tree.root(), bu.root());
    assert_eq!(bottom_up_root(&ks), bu.root());
    for (i, k) in ks.iter().enumerate() {
        assert_eq!(tree.get_proof_for_key(k).unwrap(), bu.path(i));   // every key
    }
}

#[test] fn parity_small() {
    for n in [1, 2, 3, 4, 5, 7, 8, 9, 1000, 1023, 1024, 1025, 4097] { check_parity(n); }
}
#[test] #[ignore] fn parity_full_2_pow_20() { check_parity(1 << DEPTH); }
// timing(): n in {10_000, 100_000}: MerkleTree::new(..).root() vs bottom_up_root vs BottomUp::new + path.
// circuit_fixture_convention(): leaf = compress([fr(b"KDF"), core_sk]) from
//   zk/proofs/poq/src/lib.rs test_core_node_full_flow; fold its 20 (sibling, selector)
//   pairs four ways (siblings/selectors each forward or reversed) and compare with core_root.
```

Run: `cargo test --release -p logos-blockchain-blend-crypto --lib probe:: -- --nocapture --test-threads=1` (add `--ignored` for the 2^20 and timing tests). Peak RSS of the 2^20 run was read from `VmHWM` in `/proc/<pid>/status`.

### B.2 `ledger/src/mantle/sdp/memprobe.rs` (abridged)

```rust
struct Counting;                                   // wraps System; LIVE and PEAK AtomicUsize
unsafe impl GlobalAlloc for Counting { /* alloc/dealloc/realloc update LIVE, PEAK */ }
#[global_allocator] static A: Counting = Counting;

/// A 329-byte locator: /dns/<323 chars>/tcp/<port> = 1 + 2 + 323 + 1 + 2 bytes,
/// rebuilt through the wire path so the byte vector is exactly 329 long.
fn locator(decl: usize, j: usize) -> Locator {
    let head = format!("d{decl}l{j}x");
    let name = format!("{head}{}", "a".repeat(323 - head.len()));
    let parsed: Locator = format!("/dns/{name}/tcp/{}", 1000 + j).parse().unwrap();
    let wire: &[u8] = parsed.as_ref();
    let l = Locator::try_from(wire.to_vec()).unwrap();   // PROBE_STRING_LOCATORS keeps `parsed`
    assert_eq!(l.len(), 329);
    l
}

fn declaration(i: usize, n_locators: usize) -> (DeclarationId, Declaration) {
    // distinct Ed25519 provider key from seed i+1, zk_id = (i+1) * 0x123456789abcdef1,
    // created 0, active 2, withdraw_at None, n_locators locators from locator(i, j)
}

fn run(n: usize, n_locators: usize) {
    let config = cryptarchia::tests::config();          // inactivity_period 2, min network size 1
    let mut sdp = SdpLedger::new(Epoch::new(2)).with_blend_service(/* .. */, &es2);
    // insert n declarations straight into the BlendNetwork ServiceState's rpds map
    // measure: LIVE delta, RSS; then
    let snap_a = sdp.active_declarations(Epoch::new(2), &params);   // epoch_state
    let snap_b = sdp.active_declarations(Epoch::new(3), &params);   // next_epoch_state
    let all = sdp.declarations();
    // pointer equality of the first locator's bytes: ledger vs snap_a / snap_b / all
    let (_, _) = sdp.try_apply_header(sdp_config, &epoch_state(2, snap_a.clone()),
                                      &epoch_state(3, snap_b.clone())).unwrap();
    // sort_nodes_and_build_merkle_tree over snap_a's (provider_id, zk_id): root, and root + one path
    // cryptarchia::tests::genesis_state(..).epoch_state_for_slot(first slot of epoch 1, &sdp, ..)
}
```

Run: `cargo test --release -p logos-blockchain-ledger --lib memprobe -- --ignored --nocapture --test-threads=1`.

## Appendix C · Raw output

### C.1 `blend/crypto`

```
FIXTURE selectors_reversed=false hashes_reversed=false: matches_root=false
FIXTURE selectors_reversed=true hashes_reversed=false: matches_root=true
FIXTURE selectors_reversed=false hashes_reversed=true: matches_root=false
FIXTURE selectors_reversed=true hashes_reversed=true: matches_root=false
PARITY n=1: root and all 1 paths equal; build current=300.863µs bottom_up=264.684µs; path compare=8.588µs
PARITY n=2: root and all 2 paths equal; build current=433.313µs bottom_up=255.605µs; path compare=9.028µs
PARITY n=3: root and all 3 paths equal; build current=584.193µs bottom_up=258.145µs; path compare=13.045µs
PARITY n=4: root and all 4 paths equal; build current=713.142µs bottom_up=258.786µs; path compare=17.745µs
PARITY n=5: root and all 5 paths equal; build current=857.297µs bottom_up=274.678µs; path compare=22.134µs
PARITY n=7: root and all 7 paths equal; build current=1.140455ms bottom_up=294.02µs; path compare=29.968µs
PARITY n=8: root and all 8 paths equal; build current=1.341963ms bottom_up=310.613µs; path compare=33.131µs
PARITY n=9: root and all 9 paths equal; build current=1.413092ms bottom_up=312.799µs; path compare=37.573µs
PARITY n=1000: root and all 1000 paths equal; build current=155.092233ms bottom_up=7.160722ms; path compare=4.181623ms
PARITY n=1023: root and all 1023 paths equal; build current=157.211256ms bottom_up=7.487737ms; path compare=7.760203ms
PARITY n=1024: root and all 1024 paths equal; build current=161.512708ms bottom_up=7.459872ms; path compare=4.490001ms
PARITY n=1025: root and all 1025 paths equal; build current=163.917889ms bottom_up=7.89294ms; path compare=5.28821ms
PARITY n=4097: root and all 4097 paths equal; build current=671.858447ms bottom_up=32.708469ms; path compare=19.779332ms
PARITY n=1048576: root and all 1048576 paths equal; build current=172.829390988s bottom_up=7.355699895s; path compare=5.839706721s
VmHWM_kB=1736360
TIMING n=10000: current(add_leaves)=1.629391125s bottom_up_root_only=71.867751ms bottom_up_with_levels+1path=78.425834ms speedup_root=22.7x
TIMING n=100000: current(add_leaves)=16.725801897s bottom_up_root_only=741.497089ms bottom_up_with_levels+1path=788.542085ms speedup_root=22.6x
```

### C.2 `ledger`

```
== n=100000 locators/decl=8 | size_of Declaration=304 Locator=8 Locators=24 DeclarationId=32
LEDGER map: 324.2 MiB live (3400 B/decl), RSS 6.6->358.1 MiB, build 5.259022296s
SNAPSHOT 1 active_declarations: 48.2 MiB (506 B/decl) in 114.512388ms; RSS 408.0 MiB
SNAPSHOT 2 active_declarations: 48.2 MiB (506 B/decl) in 112.880266ms; RSS 471.2 MiB
declarations(): 48.2 MiB (506 B/decl) in 68.768114ms; RSS 513.3 MiB
LOCATOR BYTES SHARED: ledger==snap1 true ledger==snap2 true ledger==declarations() true
TRANSITION SdpLedger::try_apply_header: 17.510854079s, peak transient 156.3 MiB, retained 29.8 MiB
TREE current builder (root only kept): 17.223848338s, peak transient 134.9 MiB
TREE current builder + 1 path (blend side): 18.018045289s
TRANSITION cryptarchia epoch_state_for_slot (1 new active_declarations snapshot): 147.028035ms, peak transient 66.7 MiB, retained by returned EpochState 0.0 MiB

== n=100000 locators/decl=1 | size_of Declaration=304 Locator=8 Locators=24 DeclarationId=32
LEDGER map: 72.6 MiB live (761 B/decl), RSS 6.7->86.4 MiB, build 4.251811327s
SNAPSHOT 1 active_declarations: 42.9 MiB (450 B/decl) in 100.103909ms; RSS 131.7 MiB
SNAPSHOT 2 active_declarations: 42.9 MiB (450 B/decl) in 97.880258ms; RSS 216.0 MiB
declarations(): 42.9 MiB (450 B/decl) in 57.343786ms; RSS 258.1 MiB
LOCATOR BYTES SHARED: ledger==snap1 true ledger==snap2 true ledger==declarations() true
TRANSITION SdpLedger::try_apply_header: 16.067167377s, peak transient 156.3 MiB, retained 29.8 MiB
TREE current builder (root only kept): 16.470916052s, peak transient 134.9 MiB
TREE current builder + 1 path (blend side): 16.558470974s
TRANSITION cryptarchia epoch_state_for_slot (1 new active_declarations snapshot): 90.155119ms, peak transient 63.6 MiB, retained by returned EpochState 0.0 MiB

== n=10000 locators/decl=8 (wire-path locators)
LEDGER map: 32.4 MiB live (3400 B/decl), RSS 6.7->41.9 MiB, build 602.870756ms
SNAPSHOT 1 active_declarations: 5.9 MiB (616 B/decl) in 10.829993ms; RSS 48.1 MiB
SNAPSHOT 2 active_declarations: 5.9 MiB (616 B/decl) in 9.258205ms; RSS 58.6 MiB
TRANSITION SdpLedger::try_apply_header: 1.739726563s, peak transient 16.5 MiB, retained 3.0 MiB
TREE current builder (root only kept): 1.79257482s, peak transient 14.4 MiB

== n=10000 locators/decl=8, PROBE_STRING_LOCATORS=1 (string-parsed locators)
LEDGER map: 57.1 MiB live (5984 B/decl), RSS 6.5->66.4 MiB, build 499.710675ms
```

The per-entry snapshot cost differs between 10^4 (616 B) and 10^5 (506 B) because of the hash table's power-of-two capacity (16,384 slots for 10^4 entries, 131,072 for 10^5). "Retained 29.8 MiB" after `try_apply_header` is the new `SdpLedger`'s target-epoch state (the provider map and verifier), which the probe kept alive until the next line.
