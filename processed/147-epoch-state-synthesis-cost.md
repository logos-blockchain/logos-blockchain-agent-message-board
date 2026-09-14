# Audit Report — Cost of epoch-state synthesis before the proof of leadership is verified

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/147`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `ledger` (`cryptarchia`, `mantle/sdp`), `services/chain/chain-service`, `merkle/*`, `zk/groth16`, `zk/proofs/pol`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `cryptarchia-v1-protocol.md`, `cryptarchia-proof-of-leadership.md`, `cryptarchia-total-stake-inference.md`
Date: `2026-09-11` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: the premise of the issue does not hold at this commit. For a same-epoch block, `update_epoch_state` costs about 100 ns, independent of the UTXO-set size and of the declaration count, because every collection it touches is structurally shared and the in-epoch snapshot branch that would clone the UTXO set and recompute the active declarations is unreachable. The only pre-proof work that scales is the epoch-boundary case, which deep-copies every active declaration once (twice after skipped epochs): about 0.13 µs per active declaration, 1.3 ms at 10,000, against 0.9 ms for one Groth16 verification on the same machine.
- Findings: `0` critical · `0` high · `0` medium · `1` low · `1` informational
- Key themes: pre-proof work at epoch boundaries proportional to the active declaration count; dead snapshot code; a panic instead of an error on a restored state under a changed epoch configuration.
- Must-fix before launch: none.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `ledger/src/cryptarchia/mod.rs` | `EpochState::update_from_ledger`, `LedgerState::update_epoch_state`, `try_apply_proof`, `try_apply_header`, `verify_proof_of_leadership`, `epoch_state_for_slot`, `from_utxos` |
| `ledger/src/lib.rs` | `Ledger::prepare_update`, `LedgerState::try_update`, `try_apply_header`, `verify_proof_of_leadership` |
| `ledger/src/mantle/sdp/mod.rs` | `SdpLedger::active_declarations` and `is_active` |
| `ledger/src/mantle/pow/*` | `last_closed_epoch_load`, `compute_epoch_blend_difficulty` (only their cost on the header path) |
| `ledger/src/cryptarchia/{stake.rs,block_density.rs}` | cost of the stake inference and the density counter |
| `ledger/src/config.rs`, `consensus/cryptarchia-engine/src/time.rs` | `epoch`, `stake_distribution_snapshot`, `nonce_snapshot` |
| `merkle/utxotree`, `merkle/tree`, `merkle/dynamic-merkle` | clone and `root()` cost of `UtxoTree` |
| `services/chain/chain-service/src/lib.rs`, `uncle.rs`, `states.rs` | block application order, uncle proof verification, ledger-state restore |
| `zk/groth16/src/verifier.rs`, `zk/proofs/pol/src/lib.rs` | which verifier runs for a proof of leadership |

**Out of scope**

Transaction application (`try_apply_contents`), the SDP and PoW epoch transitions in `mantle_ledger.try_apply_header` (they run after the proof is verified), the storage-market and stake-inference arithmetic itself (issue #38), fork choice, the network layer (issue #43 covers the gossip-side amplification), the PoL circuit. Assumed correct: `ark-groth16`/`ark-bn254`/`ark-ec` 0.5, `rpds`, `num-bigint`.

**Assumptions**

The specifications at the commit above are the reference. The repo-level facts of issue #19 hold at this commit and were re-checked where used: the release profile has no `overflow-checks`, and `panic`, `expect_used`, `unwrap_used` are allowed lints, so an `assert!` or `expect` on the block path is a crash, not an error. The Blend network size assumed for the boundary estimate is 1,000 to 10,000 active declarations; the cost scales linearly beyond that.

## 3. Method

- Manual review of the in-scope paths, working through issue `#147` (all four items) with its parent `#6` and the context issue `#19`.
- Spec conformance against `cryptarchia-v1-protocol.md` §Epoch (Epoch Schedule, Epoch State, Eligible Leader Notes, Epoch Nonce, Total Stake Inference, Epoch State Pseudocode), §Leadership Lottery, §Uncle References, §Block Header Validation, §Chain Maintenance; `cryptarchia-proof-of-leadership.md` §Ledger Root, §Circuit Public Inputs, §Linking the Proof of Leadership to a Block; `cryptarchia-total-stake-inference.md` in full. The two core overview documents were read in full first.
- Automated tooling: `cargo 1.98.1` / `rustc 1.98.1` release build of a throwaway timing test added to a private copy of the tree (the copy's `.cargo/config.toml` `rust-lld` override was removed because the local macOS SDK cannot be parsed by that linker; no other change to the build). Hardware: Apple M4 Pro, single thread, mean of 200 iterations. The test builds `CryptarchiaLedger::from_utxos` with N notes and an `SdpLedger` with M declarations all active at epoch 0, using the ledger crate's own test `config()` (`k = 1`, `f = 1/10`, epoch length 100 slots, nonce phase 60 slots, `inactivity_period = 2`). Groth16 timing uses `lb_pol::verify` with the production verification key and a well-formed random proof, which costs the same pairings as a valid one.
- Dynamic testing: none beyond the timing test.

Measured, per call:

| UTXOs | declarations | `LedgerState::clone` | same-epoch, nonce phase | same-epoch, after nonce phase | epoch boundary | 3 skipped epochs | `aged.root()` + `latest.root()` | `active_declarations` alone | Groth16 verify |
|---|---|---|---|---|---|---|---|---|---|
| 1,000 | 0 | 66 ns | 107 ns | 97 ns | 0.62 µs | 0.65 µs | 2 ns | 17 ns | 1.01 ms |
| 1,000 | 1,000 | 67 ns | 106 ns | 97 ns | 98 µs | 7.7 µs | 1 ns | 99 µs | 0.87 ms |
| 1,000 | 10,000 | 66 ns | 107 ns | 97 ns | 1.50 ms | 104 µs | 2 ns | 1.43 ms | 0.98 ms |
| 100,000 | 0 | 63 ns | 102 ns | 93 ns | 0.52 µs | 0.62 µs | 1 ns | 16 ns | 0.83 ms |
| 100,000 | 1,000 | 66 ns | 107 ns | 97 ns | 93 µs | 7.2 µs | 2 ns | 94 µs | 0.87 ms |
| 100,000 | 10,000 | 66 ns | 106 ns | 97 ns | 1.24 ms | 118 µs | 3 ns | 1.29 ms | 0.87 ms |

Reading the table against the code:

- Same-epoch blocks (`update_epoch_state` case 1, `ledger/src/cryptarchia/mod.rs:293-299`) cost about 100 ns whatever N and M are. The work is one `EpochState::clone` (an `Arc` root pointer for the tree, an `rpds` map handle for the items, an `Arc` for the declarations, a few field elements: `merkle/dynamic-merkle/src/lib.rs:379-385`, `merkle/tree/src/lib.rs:53-65`) and `update_from_ledger` (`:110-166`), whose stake/declarations branch never runs (S-001). During the nonce phase it also recomputes the Blend PoW difficulty with `BigUint` arithmetic (`:118-146`); that is the 10 ns difference between the two same-epoch columns.
- Epoch-boundary blocks (case 2, `:300-366`) add one stake inference (`i128` arithmetic, `stake.rs:27-71`), one lottery-constant derivation (`BigUint` division), one storage-market update and one `sdp.active_declarations(next_epoch_state_epoch)` (`:345-348`). The last is the only term that scales: `active_declarations` (`ledger/src/mantle/sdp/mod.rs:611-633`) iterates every declaration of every service and deep-copies each active one, `Locators` included, into a fresh `HashMap`. Measured about 11 ns per declaration visited plus about 120 ns per declaration copied; at 10,000 active declarations, 1.3 ms, 1.5 times one Groth16 verification. The `self.utxos.clone()` at `:339` is an `Arc` clone.
- Skipped-epoch blocks (case 3, `:367-455`) call `active_declarations` twice (`:421-423`, `:436-439`) and loop once per skipped epoch over the stake inference and the storage market (`:376-380`, `:393-395`), both cheap. The table shows this case cheaper than the boundary only because, in the test set-up, no declaration is still active at epoch 4 or 5 (`is_active`, `sdp/mod.rs:279-287`), so nothing is copied; the per-declaration iteration cost remains (104-118 µs at 10,000). With declarations active at the target epochs it is twice the boundary cost. The number of skipped epochs is bounded by the wall clock: the chain service rejects `slot > current_slot` before the ledger is touched (`services/chain/chain-service/src/lib.rs:443-448`).
- The two roots the proof needs are stored in the tree's root node (`merkle/dynamic-merkle/src/lib.rs:503-511`); reading them is free.
- One Groth16 verification is 0.83-1.01 ms here (`ark_groth16` with `LibsnarkReduction`, `zk/groth16/src/verifier.rs:30-38`, called from `zk/proofs/pol/src/lib.rs:125-130`).

Checked and ruled out:

- Item 1, the issue's claim that every block "clones `utxos` and recomputes `sdp.active_declarations(...)` over the whole SDP ledger": false at this commit. The branch that would do so, `update_from_ledger` `:145-155`, requires `ledger.slot < stake_distribution_snapshot(next_epoch_state.epoch)`. `next_epoch_state.epoch` is always `epoch_state.epoch + 1` (set at `:332`, `:425`, and at genesis `:768`/`:778`), `epoch_state.epoch` is always `config.epoch(ledger.slot)`, and `stake_distribution_snapshot(e + 1) = e * epoch_length` (`ledger/src/config.rs:110-112`) is by definition of `config.epoch` (`consensus/cryptarchia-engine/src/time.rs:260-264`) at most `ledger.slot`. The condition is never true; the `else` arm at `:154` moves the existing handles. The measurement confirms it: the same-epoch cost is identical with 0 and 10,000 declarations. The snapshot the field doc describes is actually taken at the transition (`:339`, `:415`, `:430`), as the `Arc` clone of the parent's set after the last block of the previous epoch. This matches the spec's $`\mathbb{C}_\text{LEAD}^{ep} := \textbf{commitment\_root\_at\_slot}(sl_{ep-1}, tip)`$ (§Epoch State Pseudocode).
- Item 1, cloning: the outer `parent_state.clone()` in `Ledger::prepare_update` (`ledger/src/lib.rs:203-205`), the `epoch_state().clone()` in `LedgerState::try_apply_header` (`:319`) and the `self.clone()` in `verify_proof_of_leadership` (`cryptarchia/mod.rs:568`) are all constant-time: every collection in `CryptarchiaLedger`, `MantleLedger`, `SdpLedger`, `PowState`, `LeaderState` and `Channels` is an `rpds` persistent structure or an `Arc` (`ledger/src/mantle/mod.rs:60-66`, `pow/mod.rs:56-64`, `leader.rs:19-25`, `core/src/mantle/channel.rs:103-106`), except the 120-entry `fee_window` array (`cryptarchia/mod.rs:234`), which is a 1 KiB memcpy.
- Item 2: for a same-epoch block, every public input of the proof is readable from the parent state unchanged. Case 1 replaces only `slot` and `next_epoch_state` (`:295-298`); `try_apply_proof` reads `epoch_state.utxos.root()`, `utxos.root()`, `epoch_state.nonce`, `epoch_state.lottery_0/1` (`:503-510`), none of which case 1 touches. For a boundary block they are equally cheap to derive without the SDP and storage-market work: the aged root is `next_epoch_state.utxos` (frozen since the previous transition), the nonce is `ledger.nonce` if `ledger.slot < nonce_snapshot` and `next_epoch_state.nonce` otherwise (`:117-140`), the total stake is one stake inference over the parent's density counter, the lottery values follow from it. See S-002.
- Item 3: the `assert_eq!(config.epoch(slot), self.epoch_state.epoch)` at `:502` is not reachable through the block path. `update_epoch_state` always runs first (`:551-552`) and leaves `epoch_state.epoch == config.epoch(slot)` in all three cases: case 1 by the invariant above, case 2 because it promotes `next_epoch_state` whose epoch is `current_epoch + 1 == new_epoch` (`:300`, `:326-331`), case 3 because it sets `epoch: new_epoch` (`:400`). `epoch_state_for_slot` (`:702-714`) uses the same function. The genesis constructor seeds slot 0, epoch 0, next epoch 1 (`:753`, `:768`, `:778`). The one way to break the invariant is a `LedgerState` produced under one `Config` and applied under another, which the node allows at restart (LB-002). A second panic on the same line, `EpochConfig::epoch`'s `expect` for `slot / epoch_length > u32::MAX`, is unreachable while `slot <= current_slot` is enforced from the slot timer (`chain-service/src/service/phases/following.rs:77`).
- Item 4: `verify_uncle_pol` (`chain-service/src/uncle.rs:128-142`) does repeat, per uncle, the `CryptarchiaLedger` clone, the synthesis and a Groth16 verification, on the ledger state of the uncle's parent. For an uncle in the same epoch as its parent the synthesis is the 100 ns case; only an uncle whose slot crosses an epoch boundary from its parent pays the boundary cost (up to four times per block). Caching per `(parent, epoch)` would work, because the public inputs of every block or uncle extending the same parent in the same epoch are identical (`latest_root` is the parent's UTXO root, the rest is the epoch state), but S-002 removes the need for it. The `expect("ledger state of a block on the chain must exist")` at `:131-134` holds: the parent was found by walking `consensus.branches()` (`:96-113`), and ledger states are pruned only for the blocks the engine prunes (`chain-service/src/lib.rs:534-551`, `:553-565`), after the caller has observed the transition (`:425-429`).
- Spec consistency of the per-uncle derivation: the spec verifies an uncle's proof against "the epoch state of the epoch of $`sl_U`$ as derived on the chain of $`A`$" (§Uncle References); the implementation derives it from the uncle's parent $`P`$ on that chain. The two coincide because the uncle window bounds $`sl_U - sl_P < W f^{-1} = 360`$ slots, far less than the $`4\lfloor k/f \rfloor`$ slots of the Lottery Constants Finalization phase, so for any uncle crossing a boundary the parent lies after the nonce snapshot and after the density window, where nonce, aged set and inferred stake are already frozen and identical on every block of the chain.
- Ordering against the spec: §Block Header Validation step 9 verifies the proof against `(T, parent, sl, ...)`, that is, a function of the parent state and the slot. The implementation derives the child epoch state first (`:530-536`, `:551-552`), which is a spec-neutral implementation choice as long as the derived inputs equal the ones the spec prescribes; they do. The uncle-before-block ordering is already reported as LB-003 of the issue #43 report (PR #142) and is not repeated here.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Epoch-boundary state synthesis proportional to the active declaration count runs before the block's proof of leadership is checked | Denial of Service | Low | Low | Open |
| LB-002 | Ledger state restored under a changed epoch configuration panics in `try_apply_proof` instead of failing with an error | Error Reporting | Informational | High | Open |

### LB-001 · Epoch-boundary state synthesis proportional to the active declaration count runs before the block's proof of leadership is checked

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `ledger/src/cryptarchia/mod.rs:540-553` (`update_epoch_state_and_apply_proof`), `:345-348`, `:421-423`, `:436-439` (`sdp.active_declarations` calls); `ledger/src/mantle/sdp/mod.rs:611-633` (`active_declarations`) |
| Status | Open |

**Description**

`update_epoch_state_and_apply_proof` synthesises the child's epoch state before it verifies the proof:

```rust
// ledger/src/cryptarchia/mod.rs:551-552
self.update_epoch_state(slot, sdp, pow, config)?
    .try_apply_proof(slot, proof, config)
```

For a block in the same epoch as its parent the synthesis costs about 100 ns and is not worth reordering. For a block in the next epoch it runs case 2 of `update_epoch_state`, which includes `sdp.active_declarations(next_epoch_state_epoch, ..)` (`:345-348`): a pass over every declaration in the SDP ledger that deep-copies each declaration still active at that epoch, `Locators` included, into a new `HashMap` (`sdp/mod.rs:616-632`). After one or more skipped epochs (case 3) it runs twice (`:421-423`, `:436-439`). The result is only needed if the block is valid; it is computed whether or not the proof verifies. Measured: 0.13 µs per active declaration, 1.3 ms at 10,000 (1.5 times a Groth16 verification), 13 ms at 100,000; the lapsed declarations that issue #92 (LB-004 of PR #102) found are never removed still cost 11 ns each to visit.

**Exploit scenario**

While the last canonical block of the previous epoch is still shallower than the latest immutable block (up to `k = 2160` blocks, about 18 hours at `f = 1/30`), a peer takes any block `P` of the previous epoch from the block tree, builds headers with `parent = P`, a slot in the current epoch not after the wall clock, empty body, a random 128-byte proof and a valid self-signature, and publishes one per second. Each receiving node, for each header, clones the parent state, runs the case-2 synthesis including the active-declaration copy, then fails the Groth16 verification and drops the header. With 10,000 active Blend declarations that is 1.3 ms of synthesis plus 0.9 ms of pairing per header, 2.5 times the cost of rejecting the same header with the proof checked first, and the gossip amplification of LB-002 in the issue #43 report applies. Cost to the attacker: one Ed25519 signature per header. The factor grows linearly with the declaration count and is not bounded by the protocol, since the SDP ledger has no cap (issue #92).

**Recommendation**

- *Short term*: derive the proof's public inputs from the parent state (S-002) and call `try_apply_proof` before `update_epoch_state`, or at least move the `active_declarations` and storage-market work out of `update_epoch_state` into a step that runs after the proof verifies.
- *Long term*: keep the active-declaration snapshot as a persistent map filtered incrementally at the transition instead of a per-epoch deep copy, so the boundary cost stops scaling with the network size (the same structure would serve issue #104).

**References**: issue #147 items 1 and 2; `cryptarchia-v1-protocol.md` §Block Header Validation step 9; issue #43 report (PR #142) LB-002, LB-003; issue #92 report (PR #102) LB-004.

### LB-002 · Ledger state restored under a changed epoch configuration panics in `try_apply_proof` instead of failing with an error

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Error Reporting |
| Target | `ledger/src/cryptarchia/mod.rs:502` (`try_apply_proof`); `services/chain/chain-service/src/states.rs:94-98` (`StartingState::Lib`) |
| Status | Open |

**Description**

```rust
// ledger/src/cryptarchia/mod.rs:502
assert_eq!(config.epoch(slot), self.epoch_state.epoch);
```

Through the block path the assertion cannot fail (see §3, item 3). It can fail when the `LedgerState` was not produced by this node's `Config`: `CryptarchiaConsensusState::from_settings` accepts a serialized `lib_ledger_state` from the settings (`states.rs:94-98`) and pairs it with `settings.config`. If the epoch configuration (`epoch_config`, `base_period_length`) differs from the one the state was produced under, `config.epoch(slot)` no longer equals the stored `epoch_state.epoch`; the first block applied takes case 1 of `update_epoch_state` (both epochs computed with the new config agree), keeps the stale `epoch_state`, and the assertion panics. The `panic` lint is allowed workspace-wide (issue #19), so nothing flags it.

**Exploit scenario**

Not exploitable by a network participant. An operator who changes the epoch parameters in the node settings and restarts from a saved LIB state gets a crash loop at the first received block with no message naming the mismatch, rather than a startup error.

**Recommendation**

- *Short term*: return `LedgerError::InvalidSlot`-style error from `try_apply_proof` (or debug-assert), and validate in `from_settings` that `config.epoch(lib_ledger_state.slot()) == lib_ledger_state.epoch_state().epoch`.
- *Long term*: persist the config hash next to the saved ledger state and refuse to start on a mismatch.

**References**: issue #147 item 3; issue #19 (allowed `panic` lint).

## 5. Suggestions (non-security)

### S-001 · The in-epoch stake and declarations snapshot in `update_from_ledger` is unreachable

| | |
|---|---|
| Target | `ledger/src/cryptarchia/mod.rs:145-155` (`EpochState::update_from_ledger`); `:81-84` and `:91-97` (field docs) |

The branch guarded by `ledger.slot < config.stake_distribution_snapshot(self.epoch)` cannot run: `self.epoch` is `config.epoch(ledger.slot) + 1` by construction, so the snapshot slot is the first slot of the ledger's own epoch and never after `ledger.slot` (§3, item 1). The `utxos` and `active_declarations` of `next_epoch_state` are set once, at the transition (`:339`, `:345-348`, `:415-423`, `:430-439`), and then carried unchanged. The field comments describe a snapshot "taken at the beginning of the epoch" and "frozen at the same slot as the stake distribution", which the transition code, not this branch, provides. Remove the branch and the `sdp` parameter of `update_from_ledger`, and say in the field docs where the snapshot is taken. This also removes the misleading `TODO` at `:276-278` and makes the answer to issue #147 item 1 visible in the code.

### S-002 · Derive the proof-of-leadership public inputs from the parent state and verify the proof first

| | |
|---|---|
| Target | `ledger/src/cryptarchia/mod.rs:493-516` (`try_apply_proof`), `:540-553`, `:557-571` (`verify_proof_of_leadership`); `services/chain/chain-service/src/uncle.rs:128-142` |

All six public inputs are a function of the parent state and the slot alone (§3, item 2): `latest_root = self.utxos.root()`; and, with `e = config.epoch(slot)` and `e_p = epoch_state.epoch`: if `e == e_p`, the aged root, nonce and lottery values of `epoch_state`; if `e == e_p + 1`, the aged root of `next_epoch_state`, the nonce `self.nonce` when `self.slot < nonce_snapshot(e)` and `next_epoch_state.nonce` otherwise, and the lottery values from one stake inference over `block_density`; if `e > e_p + 1`, the current `utxos.root()`, `self.nonce`, and the inference iterated over the skipped epochs. A `fn leader_public_inputs(&self, slot, config) -> LeaderPublic` of this shape lets `try_apply_header` verify the proof before `update_epoch_state`, lets `verify_proof_of_leadership` drop its `self.clone()` and synthesis, and lets `verify_uncle_pol` verify each uncle without any state synthesis, which answers item 4 without a cache. `update_epoch_state` then only needs to assert that the values it produces match the ones the proof was checked against, which the existing tests (`:2200-2260`) can cover.

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
