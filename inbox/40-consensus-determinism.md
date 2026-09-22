# Audit Report — Cryptarchia fork-choice determinism under forced hash collisions

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/40`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `19353c61963d4ef8c37ad00d24fdf0f08a482887` — component(s): `consensus/cryptarchia-engine`, `ledger/src/cryptarchia`, `services/chain/chain-service`, `services/time`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `cryptarchia-v1-protocol.md`, `fork-choice.md`, `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`
Date: `2026-09-21` — author: `Hansie Odendaal` — status: `final`

---

## 1. Summary

- Overall assessment: The unresolved determinism checklist items do not produce a new reportable finding at the pinned revision. A focused adversarial test now confirms the missing qualification: fixed SipHasher does not erase insertion history from a full-collision bucket, but both `maxvalid_mc` and `maxvalid_bg` retain the supplied current chain for equal-length/equal-density candidates. Different construction histories selected different current chains exactly as the specification's first-seen strict-tie rule permits.
- Findings: `0` new critical · `0` new high · `0` new medium · `0` new low · `0` new informational
- Key themes: consensus-critical identifiers and counters use fixed-width types; unordered collections are used for membership/look-up or are backed by deterministic persistent structures; the hash-trie collision path preserves insertion history; validation receives the current slot explicitly.
- Preserved follow-up: the previously reported floating-point behavior remains tracked by #38 and follow-up #175 and is not duplicated here.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `consensus/cryptarchia-engine/src/lib.rs` | Fork choice, branch/tip storage, uncle selection, and consensus counters. |
| `consensus/cryptarchia-engine/src/config.rs` | Consensus configuration arithmetic, including the known floating-point path. |
| `consensus/cryptarchia-engine/src/time.rs` | Slot/epoch representation and epoch arithmetic. |
| `ledger/src/cryptarchia` | Consensus ledger state and service-parameter collections. |
| `services/chain/chain-service` | Block validation entry point and startup state selection. |
| `services/time` | Local-clock conversion to the current slot. |

**Out of scope**

The already reported floating-point discrepancy in `Config::s_gen` and stake inference was not re-reported. Network behavior, cryptographic primitive correctness, and unrelated service collection ordering were not audited beyond their consensus-facing call sites.

## 3. Method

- Read the parent review direction `#1` and the complete checklist issue `#40`, including its comment linking the existing float finding to #38/#175.
- Read the required core specifications and the complete `cryptarchia-v1-protocol.md` and `fork-choice.md` at the pinned `logos-lips` revision.
- Inspected the pinned `logos-blockchain` revision with revision-qualified `git show` and `git grep`.
- Checked Cargo feature resolution: the workspace declares `rpds` with `default-features = false`; `cargo tree --workspace -i rpds -e features --offline` shows the `serde` feature but no `std` feature. In `rpds` 1.2.1, this selects the fixed `BuildHasherDefault<SipHasher>` rather than process-randomized `RandomState`.
- Automated tooling: static source searches and Cargo dependency-tree inspection.
- Dynamic testing: a focused test was added only in a temporary linked worktree at the pinned target revision. It defines `CollidingId(u64)` whose `Hash` implementation always hashes `0`, builds the same equal-length/equal-density two-tip fork through opposite arrival orders, and invokes the real `maxvalid_mc` and `maxvalid_bg` implementations. Command: `cargo test -p logos-blockchain-cryptarchia-engine --lib forced_collisions_preserve_first_seen_ties_in_fork_choice -- --nocapture`. Result: `1 passed`; collision-tip iteration was `[CollidingId(4), CollidingId(2)]` and `[CollidingId(2), CollidingId(4)]`, while both fork-choice functions retained `CollidingId(2)` when that was supplied as the current chain. No audited source checkout was modified.

## 4. Assessment

### 4.1 Fork-choice tie behavior under forced full collisions

`maxvalid_bg` and `maxvalid_mc` iterate `Branches::branches()` and retain the current candidate on equal length/density (`consensus/cryptarchia-engine/src/lib.rs:80-140`). `Branches` stores branches and tips in `rpds::HashTrieMapSync` and `HashTrieSetSync` (`:153-160`), so the collection order reaches fork choice.

The workspace disables `rpds`' default `std` feature, and the resolved dependency graph enables only `serde`. The selected default hasher is therefore the fixed `SipHasher`, so the hash-trie's sparse-array traversal is deterministic for a fixed key set. That is not, by itself, a proof that iteration is independent of construction history: `rpds` stores full-hash collisions in a persistent list, and its `push_front_mut` insertion path makes collision-bucket order history-dependent.

The focused adversarial test exercised that exact path. It built the same two equal-length/equal-density fork tips, `CollidingId(2)` and `CollidingId(4)`, through opposite arrival orders. The collision-list iteration order changed from `[4, 2]` to `[2, 4]`. The corresponding live engines selected tips `2` and `4`, respectively, because each engine's current chain was the first-arriving branch and equal candidates do not replace `cmax`. When the same current chain (`CollidingId(2)`) was supplied to both branch sets, both `maxvalid_mc` and `maxvalid_bg` selected `CollidingId(2)` despite the different collision-list orders.

This distinguishes legitimate first-seen behavior from a collection-order artifact: the strict comparison preserves the current chain on ties, and the forced collision did not substitute collision-list order for that tie break. The synthetic type is not a production identifier and the test does not demonstrate that real block IDs can be made to collide under SipHash. No new consensus finding or production exploitability claim is therefore filed.

The uncle path also explicitly sorts candidates by parent slot, uncle slot, and uncle ID before selecting them (`:733-746`). Its `HashMap` and `HashSet` values at `:755-778` are membership maps/sets and are not iterated to construct a consensus result.

### 4.2 Other unordered collections do not feed order-sensitive consensus state

The ledger's `service_params` is looked up by service type, while service/declaration state is maintained through persistent ordered maps. Active declarations are collected into maps whose equality/content is order-independent; no map iteration is used to serialize, hash, or otherwise order a block or fork-choice result at the inspected call sites.

The remaining `HashSet` uses in chain-service and storage track known, stale, or removable IDs. They do not determine block contents, ledger transition order, or fork choice.

### 4.3 Numeric widths and wall-clock use

`Epoch` is a `u32` with explicit binary encoding and `Slot` is a `u64` (`consensus/cryptarchia-engine/src/time.rs:15-123`). Consensus branch lengths are `u64` (`lib.rs:172-181`); `usize` occurrences are collection capacities, counts, or test-only conversion, not serialized or hashed consensus fields.

Block validation receives `current_slot` as an argument and rejects a future block against that value (`services/chain/chain-service/src/lib.rs:423-445`). Local wall-clock reads in the inspected paths produce time-service ticks or select the node's restart/bootstrap mode (`services/chain/chain-service/src/bootstrap/state.rs:29-50`); they are not read inside block validation to derive a node-local acceptance rule.

One hardening note remains: `EpochConfig::last_slot` uses `epoch.into_inner() + 1` and `starting_slot` multiplies the epoch by the epoch length (`consensus/cryptarchia-engine/src/time.rs:266-274`). These are configuration/boundary arithmetic concerns and were not shown to create a cross-node determinism or consensus-safety failure at the pinned deployment parameters.

## 5. Existing finding disposition

The floating-point use in `Config::s_gen`, stake inference, and related ratio conversions remains an existing issue from #38 with follow-up #175, as recorded in issue #40. This report preserves that classification and does not create a duplicate `LB-NNN`.

## 6. Follow-ups

- Retain or port the focused forced-collision test as a regression test if the implementation wants an executable guard for first-seen tie behavior; the test passed at the pinned revision in the temporary linked worktree.
- Consider an explicitly ordered tip representation or an explicit stable tie-break key if the implementation should no longer depend on `rpds` feature configuration.
- Harden epoch boundary arithmetic with checked operations if maximum representable epochs are in scope for deployment or protocol tests.
- Continue the existing #38/#175 remediation for floating-point consensus calculations.

Independent review completed; this report is finalized for the Draft PR handoff.
