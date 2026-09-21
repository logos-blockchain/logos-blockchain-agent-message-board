Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/40`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `19353c61963d4ef8c37ad00d24fdf0f08a482887` — component(s): `consensus/cryptarchia-engine`, `ledger/src/cryptarchia`, `services/chain/chain-service`, `services/time`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `cryptarchia-v1-protocol.md`, `fork-choice.md`, `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`
Date: `2026-09-21` — author: `Hansie Odendaal` — status: `draft`

---

## 1. Summary

- Overall assessment: The unresolved determinism checklist items do not produce a new reportable finding at the pinned revision.
- Findings: `0` critical · `0` high · `0` medium · `0` low · `0` informational
- Key themes: consensus-critical identifiers and counters use fixed-width types; unordered collections are used for membership/look-up or are backed by deterministic persistent structures; validation receives the current slot explicitly.
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
- Dynamic testing: none; this iteration was a source/specification determinism review.

## 4. Assessment

### 4.1 Fork-choice iteration is deterministic at this revision

`maxvalid_bg` and `maxvalid_mc` iterate `Branches::branches()` and retain the current candidate on equal length/density (`consensus/cryptarchia-engine/src/lib.rs:80-140`). `Branches` stores branches and tips in `rpds::HashTrieMapSync` and `HashTrieSetSync` (`:153-160`), so the collection order reaches fork choice.

That use does not currently establish a cross-process nondeterminism finding. The workspace disables `rpds`' default `std` feature, and the resolved dependency graph enables only `serde`. The selected default hasher is therefore the fixed `SipHasher`; the hash-trie's sparse-array traversal is a deterministic function of the current key set. The strict comparisons implement the specification's first-seen/current-chain tie behavior, rather than relying on `std::collections::HashMap` or `HashSet` iteration.

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

- Add a regression test that builds an identical branch-tip set through different arrival orders and asserts identical fork-choice output, documenting the fixed-hasher dependency.
- Consider an explicitly ordered tip representation or an explicit stable tie-break key if the implementation should no longer depend on `rpds` feature configuration.
- Harden epoch boundary arithmetic with checked operations if maximum representable epochs are in scope for deployment or protocol tests.
- Continue the existing #38/#175 remediation for floating-point consensus calculations.

Draft pending independent review and explicit approval.
