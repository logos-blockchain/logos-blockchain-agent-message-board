# Audit Report — Note ageing across block-free windows: fork choice holds while the chain produces blocks, and a chain halt is the one case that breaks it, first by collapsing the inferred stake to 1, then by letting the last block before the halt fix the notes and the nonce of the epochs that follow

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/634` (parent `#5`; follow-up to `#45`, report `processed/45-epoch-nonce-grinding.md`, PR #632)
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `35a4a666e22a51eb98fe8a050854e57fe3420899` — component(s): `consensus/cryptarchia-engine/src/lib.rs`, `consensus/cryptarchia-engine/src/config.rs`, `ledger/src/cryptarchia/mod.rs`, `ledger/src/cryptarchia/stake.rs`, `ledger/src/cryptarchia/block_density.rs`, `ledger/src/config.rs`, `core/src/proofs/leader_proof.rs`, `zk/proofs/pol/src/lottery.rs`, `services/chain/chain-service/src/bootstrap/*`, `services/chain/chain-service/src/service/phases/pbp.rs`, `services/chain/chain-leader/src/lib.rs`, `nodes/node/binary/src/config/deployment/settings.yaml`, `deployment/ceremony/genesis/*/deployment-template.yaml`, `deployment/ceremony/genesis/*/stakeholders.yaml`
Specs: `https://github.com/logos-co/logos-lips` @ `6637c791cf29985251bf67f73766f97c7512f824` — read in full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md` (core), `cryptarchia-v1-protocol.md`, `fork-choice.md` (area); by section: `cryptarchia-v1-bootstr-sync.md` (§Overview, §Constants, §Setting the Fork Choice Rule, §Prolonged Bootstrap Period, §Proposing New Blocks, §Offline Grace Period, §Offline Duration Measurement), `cryptarchia-proof-of-leadership.md` (§Ledger Root, §Zero-knowledge Proof Statement, §Lottery Approximation, §Corner Case: Note Value Exceeding Inferred Total Stake), `cryptarchia-total-stake-inference.md` (§Definitions, §Algorithm; consulted because the halt case turns on it)
Date: 2026-09-23 — author: `agent (Claude)` — status: `final`

---

## 1. Summary

- Overall assessment: items 1 and 2 of the issue are confirmed on the code. A fork whose nonce the adversary could predict when it created its notes is, by construction, a fork containing only the adversary's blocks from the note-creating block up to the nonce snapshot, so its dense part cannot begin before `10k/f` slots after the divergence point; the bootstrap rule measures density in the `k/(4f)` slots right after the divergence and the online rule needs the honest chain to have grown by at most `k` blocks since it, so both reject it, at the specified and at every shipped constant set, **as long as the honest chain keeps producing blocks**. The block-free stretch the issue describes is not even what makes the nonce predictable; any adversary-only fork has that property, and the protection is fork choice rather than the ledger, as the #45 report said. The one situation in which that protection lapses is a chain halt, item 3, and there the node behaves worse than the issue anticipated: a halt that empties one stake-inference window sets the inferred total stake to `1` (the specification's `β = 1.0` and floor of `1`, both implemented faithfully), after which the leadership threshold wraps in the field and each note wins a value-determined fraction of every slot, `17%` to `41%` per slot for the shipped genesis notes, for about ten epochs; any note whose value is chosen for that regime wins every slot, and no nonce needs to be known for it. The nonce-and-notes grind the issue asks about is real too, but it only matters for the producer of the last block before the halt, needs foreknowledge or speculative grinding, and on the `β = 1.0` networks it is dominated by the value grind. Nothing in the node or the specifications re-randomises the nonce, re-ages the notes, or resets the stake estimate on restart, and no restart procedure is documented.
- Findings: `0` critical · `2` high · `0` medium · `1` low · `0` informational
- Key themes: "fork choice counts blocks, not slots, so a halt suspends the `k`-deep rule", "the stake inference has no floor but 1", "the lottery threshold wraps once a note exceeds 58 times the inferred stake", "the last block before a halt fixes three epoch variables at once", "no restart procedure".
- Must-fix before launch: LB-001 (a floor on the inferred stake, or a bounded learning rate) before any network that could ever go a full inference window without a block; that is 6 hours on the devnet and testnet templates. LB-002 should be settled by the same restart design.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `consensus/cryptarchia-engine/src/lib.rs:43-74, 81-138, 316-326, 350-360, 517-533, 574-598, 677-681` | `State::fork_choice` and `State::lib`, `maxvalid_bg`, `maxvalid_mc`, `walk_back_before`, `nth_ancestor`, `update_lib`, `prunable_forks`, `online` |
| `consensus/cryptarchia-engine/src/config.rs:104-113, 117-131` | `s_gen`, `base_period_length` |
| `ledger/src/cryptarchia/mod.rs:110-166, 257-456, 493-536, 591-604, 702-720` | `EpochState::update_from_ledger`, the three cases of `update_epoch_state`, `try_apply_proof`, `try_apply_header`, `update_nonce`, `epoch_state_for_slot`; tests at `:1386`, `:2204-2269`, `:2271-2320` |
| `ledger/src/cryptarchia/stake.rs:27-70` | `total_stake_inference`, the `β` update and the floor |
| `ledger/src/cryptarchia/block_density.rs:29-58` | the inference window and the occupied-slot count |
| `ledger/src/config.rs:65-119` | `nonce_snapshot`, `stake_distribution_snapshot`, `total_stake_inference_period` |
| `core/src/proofs/leader_proof.rs:242-291` | native `check_winning`, `phi_approx`, `ticket` |
| `zk/proofs/pol/src/lottery.rs:22-140` | `LotteryConstants`, `compute_lottery_values` |
| `services/chain/chain-service/src/bootstrap/state.rs:11-55`, `bootstrap/config.rs:1-45`, `service/phases/pbp.rs:66-100`, `services/chain/chain-leader/src/lib.rs:439-444`, `nodes/node/binary/src/config/cryptarchia/serde/service.rs:39-54` | What the node does when it restarts: fork-choice state selection, offline grace period, prolonged bootstrap period, proposal gating |
| `nodes/node/binary/src/config/deployment/settings.yaml:22-33`, `deployment/ceremony/genesis/{standalone,devnet,testnet}/deployment-template.yaml:22-33`, `deployment/ceremony/genesis/*/stakeholders.yaml` | Shipped `k`, `f`, `learning_rate`, and the genesis note values used in the numbers |
| Dynamic | Own programs, Appendix B: an exact-integer replica of the threshold and stake-inference arithmetic (Python, big integers), and a ticket-cost benchmark linking the node's `zk/poseidon2` crate |

**Out of scope**

- The uncle-reference rules and their effect on the inference count beyond what the window code does (`#183`-area issues); the Blend PoW difficulty snapshot that shares the nonce snapshot; the leader's KMS and proof generation (parent `#5`); the correctness of `η` derivation itself, established in `#45`.
- Schedule-parameter validation at load (`#175`), the checkpoint bootstrap path (`#1454` upstream TODO), chain-sync and IBD (`#135`, `#643`, `#644`).
- No Groth16 proof was generated; the circuit's threshold arithmetic is taken from `cryptarchia-proof-of-leadership.md` §Circuit Constraints and matched against the node's native `phi_approx`, which the #45 report already matched against the circuit source.
- Third-party components assumed correct: `ark-bn254`, `ark-ff`, `jf-poseidon2` at the pinned revision, `astro-float`, `rpds`.

**Assumptions**

- Poseidon2 behaves as a random function, so tickets are uniform in `[0, p)` and independent across slots and notes.
- Adversarial stake below one half; honest nodes are well connected; a halt means no block on the canonical chain for the stated span, whatever the cause.
- Repo-level facts from `#19`: release builds do not enable `overflow-checks`; the arithmetic on this path is `strict_*` or field arithmetic, and the one wrap that matters, the threshold wrapping modulo `p`, is a property of the circuit and is intended by the specification.

## 3. Method

- Worked through the five items of `#634` in order, specifications first, then code, at the commits in the header. The two fork-choice rules were traced from `fork-choice.md` into `consensus/cryptarchia-engine`, the epoch-state cases from `cryptarchia-v1-protocol.md` §Epoch State Pseudocode into `ledger/src/cryptarchia/mod.rs`, the restart path from `cryptarchia-v1-bootstr-sync.md` into `services/chain/chain-service`, and the threshold from `cryptarchia-proof-of-leadership.md` into `core/src/proofs/leader_proof.rs` and `zk/proofs/pol/src/lottery.rs`.
- Searched the node repository for any restart, halt or outage procedure (`README.md`, `deployment/`, `.github/`, config comments). None exists beyond "try removing the `state` directory" (`README.md:73`).
- Dynamic, own programs (Appendix B): `calc.py` and `calc2.py` reproduce `compute_lottery_values`, `phi_approx` and `total_stake_inference` with Python big integers, using the two constants of the PoL specification's table for `f = 1/30` (recomputed with 200-digit precision and found equal) and the same derivation for `f = 1/20`; `ticketbench`, a crate with a path dependency on the node's `zk/poseidon2`, measures the cost of one ticket on the audit machine. Hardware: Raspberry Pi 5, one core (the hardware the shipped Blend difficulty is calibrated on, `settings.yaml:36-40`). Versions: `rustc 1.97.1`, `Python 3`.
- The existing ledger tests that pin the behaviour were read, not run: `test_epoch_state_for_slot_with_empty_epochs` (`mod.rs:2204-2269`) asserts that the stake drops to 1 after an empty epoch and that nonce and notes are carried over; `test_try_apply_header_with_proof_from_jumped_epoch` (`mod.rs:2271-2320`) shows a proof built from the synthesised state verifying.
- Automated tooling: none.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | A halt that empties one inference window collapses the inferred stake to 1, after which leadership is a function of note value, every slot has several leaders, and any note of a chosen value wins every slot for about ten epochs | Consensus | High | Low | Open |
| LB-002 | The producer of the last block before a halt fixes the eligible notes, the nonce and the difficulty of every epoch whose snapshots fall inside the halt, and a note set ground at that block wins every slot of those epochs on the canonical chain | Consensus | High | High | Open |
| LB-003 | Every whole-network restart after an outage longer than 20 minutes adds a further hour without blocks, since all nodes come back in bootstrap state and none proposes until the prolonged bootstrap period ends | Denial of Service | Low | Low | Open |

### 4.1 Item 1 — bootstrap fork choice

**What the code does.** `maxvalid_bg` (`lib.rs:81-117`) iterates over the tips. For each, it takes the lowest common ancestor with the current best chain (`:92-95`, `lca` at `:277-297` walks both branches back to equal length and then in step), sets `m` to the number of blocks of the best chain after that ancestor (`:97`), and if `m ≤ k` applies the longest-chain rule (`:98-103`, strict inequality, so the first-seen chain keeps ties). Otherwise it compares densities: `density_slot = lca.slot + s_gen` (`:106`) and each chain's density is the `length` of the last block whose slot is `≤ density_slot` (`:107-108`; `walk_back_before` at `:316-326` walks back while `slot > target`, so the window is inclusive of its last slot, as the figure caption in `fork-choice.md` §Definitions says). Because both chains share the prefix up to the ancestor, comparing lengths compares the number of blocks each chain placed in the `s_gen` slots after the divergence. `s_gen = ⌊k/(4f)⌋` (`config.rs:107-113`): 16,200 slots at the specified constants, 900 on the devnet and testnet templates (`k = 120`, `f = 1/30`), 150 on the standalone template and `settings.yaml` (`k = 30`, `f = 1/20`). This is `maxvalid-bg` of `fork-choice.md` §Bootstrap Fork Choice Rule line for line.

**Why the fork in question always loses.** Let the adversary create ground notes in a block `B_a` at slot `sl_a` of epoch `e`, so that they are in `C_LEAD^{e+2}` (frozen at `sl_{e+1}`, `config.rs:110-113`, `mod.rs:146-155`). For the notes to be ground against the nonce the fork will actually use, `η^{e+2}` must be computable when `B_a` is built, and `η^{e+2}` is the nonce after the last fork block before `sl_{e+1} + 6k/f` (`config.rs:71-77`, `mod.rs:118-140`). Every fork block between `B_a` and that snapshot therefore has to be the adversary's own, since `ρ_LEAD` of an honest block is unknown until the block is published while the adversary's own `ρ_LEAD` values are deterministic in `(sl, noteID, sk)` (`leader_proof.rs:285-287`, PoL §Circuit Constraints step 7) and its winning slots for epoch `e+1` are already fixed by the honest chain before `B_a`. So the fork diverges from the honest chain at `B_a` or before it, and its ground notes cannot produce a block before `sl_{e+2} = sl_{e+1} + 10k/f > sl_a + 10k/f`. Within the density window, which ends `k/(4f)` slots after the divergence, the fork holds at most `B_a` plus the adversary's ordinary wins, about `αk/4`, against the honest chain's `(1−α)k/4`; for `α < 1/2` the honest chain is denser. This is exactly the sparse-then-dense long-range fork that `fork-choice.md` §The Long Range Attack describes and the density rule was designed for, and the block-free stretch only makes the fork sparser.

**Every placement of the window.** The window has one placement per comparison: it is anchored at the lowest common ancestor, and the ancestor is the divergence point. The two edge cases the issue names resolve as follows.

- *Divergence close to the tip*: if the honest chain has `m ≤ k` blocks after the ancestor, the density rule is skipped and the longest chain wins. The fork cannot be longer, since its dense part has not started, unless the honest chain has produced fewer than `k` blocks in more than `10k/f` slots, which is a halt, item 3.
- *Dense part inside the `s_gen` window*: impossible for this fork type, since the dense part starts at least `10k/f` slots after the divergence and the window is `k/(4f)` slots long, forty times shorter.
- *The bootstrapping node fetched the fork first*: then `cmax` is the fork. If the fork's dense part has given it more than `k` blocks after the ancestor, the density rule runs and the honest chain wins it; if not, the longest-chain rule runs and the honest chain, with about `10k` blocks after the ancestor, wins it.

The density rule cannot be fooled by uncles either: `density` counts `Branch::length`, which counts blocks of the chain only (`lib.rs:174-183, 195-197`), as `fork-choice.md` 1.1.0 requires.

**Generalisation.** The nonce of an adversary-only fork is always predictable to the adversary, block-free stretch or not; nothing in the ledger prevents that and nothing could. What `cryptarchia-v1-protocol.md` §Epoch Schedule's buffer phase guarantees is that the **canonical** chain's nonce receives at least one honest contribution, and what keeps a predictable-nonce fork from becoming canonical is fork choice. The rest of this report is about the one case where the canonical chain itself has no honest contribution.

### 4.2 Item 2 — online fork choice

`maxvalid_mc` (`lib.rs:119-138`) admits a fork only if the local chain has `m ≤ k` blocks after the common ancestor and the fork is strictly longer (`:134-135`). In `Online` state the LIB is the `k`-th ancestor of the tip (`State::lib`, `:60-73`, via `nth_ancestor` at `:350-360`), `update_lib` (`:517-533`) prunes every fork whose ancestor is below it (`prunable_forks`, `:574-598`, condition `lca.length < tip.length − k` at `:595`), and a later block of a pruned fork fails `apply_header` with `ParentMissing` (`:239-242`). The fork of §4.1 can be released with a length advantage only from `sl_{e+2}` on, at least `10k/f` slots after the divergence. Over that span the honest chain produces about `10k` blocks; the probability that it produces fewer than `k` is that of a binomial with mean `10k(1−α)` falling below `k`, which is below `2^{−k}` for every `α < 1/2` and every shipped `k` (`k = 30`: mean 200 at `α = 1/3`, tail below `10^{−40}`). So the fork is always more than `k` deep when it can be released, on the specified constants and on the shipped ones, and online nodes never see it as a candidate.

The rule counts **blocks**, not slots (`m = cmax.length − lca.length`). That is what the specification prescribes, and it is the exact reason a halt disables it: however many slots pass, a fork whose divergence point is within the last `k` blocks stays admissible, and the LIB does not move. The issue's item 2 holds under the premise "by the time it can be released the honest chain has grown"; the premise is what a halt removes.

### 4.3 Item 3 — halted-chain restart

**Is there a restart procedure?** No. The node repository has no document, script, flag or CI job for restarting a halted chain; the only related text is `README.md:73` ("If you encounter issues on restart, try removing [the `state` directory]"), and the systemd unit restarts the process on failure (`deployment/systemd/logos-blockchain-node.service:15-17`). What is coded is the generic restart path of `cryptarchia-v1-bootstr-sync.md`:

1. `choose_engine_state` (`bootstrap/state.rs:11-28`): `Bootstrapping` if the LIB is genesis or `force_bootstrap` is set; otherwise, if a recorded engine state exists, `Bootstrapping` when more than `grace_period` has elapsed since it was recorded (`:30-55`; default 20 minutes, `bootstrap/config.rs:30`, recorded every minute, `:34`), else the recorded state; otherwise `Online`.
2. In `Bootstrapping`, the prolonged bootstrap period runs for `prolonged_bootstrap_period` (`pbp.rs:66-100`, timer at `:86`), default **1 hour** in the binary (`serde/service.rs:49`; the specification says 24 hours) and unset in every shipped deployment file, so 1 hour everywhere. During it the LIB does not advance (`State::lib`, `lib.rs:60-73`) and the leader does not propose: `chain-leader/src/lib.rs:439-444` waits for the chain to become online before its first proposal.
3. The ledger synthesises the epoch state for whatever slot the first post-halt block lands in. If it is in the next epoch, case 2 of `update_epoch_state` (`mod.rs:301-366`): the new `epoch_state` is the parent's `next_epoch_state` as refreshed by the parent, so its `nonce` and `utxos` are the parent's if the parent was before the two snapshot slots (`:118-140, :146-155`), and the new `next_epoch_state` starts from `self.nonce` and `self.utxos` (`:335, :339`). If it is two or more epochs later, case 3 (`:368-455`): both `epoch_state` and `next_epoch_state` take `nonce: self.nonce` and `utxos: self.utxos` (`:410, :415, :428, :430`), and the stake estimate is updated once with the density the last epoch had and then once per skipped epoch with density `0` (`:376-380`).

Nothing re-randomises the nonce, re-ages the notes, or resets the stake estimate. Neither does the specification ask for any of this; `compute_epoch_state` is the same recursion with the same inputs.

**What a halt does to the three epoch variables.** Let `B_last` be the last block before the halt, at slot `sl_last` in epoch `e`, and let the first post-halt block be in epoch `r`. For every epoch `j` with `e+2 ≤ j ≤ r`, and for `j = r+1` if the restart came after `sl_r + 6k/f`:

- `C_LEAD^{j}` is the note set after `B_last`;
- `η^{j}` is the nonce after `B_last`;
- `D^{j}` is `D^{e+1}` reduced by the inference: once with the (possibly partial) density of epoch `e`'s window, then `(1−β)` per empty window (`stake.rs:27-70`: `new = D − β·D·(expected − measured)/expected`, floored at `1` at `:67`). With the specification's `β = 1.0` (`cryptarchia-total-stake-inference.md` §Definitions), shipped on the devnet and testnet templates (`deployment-template.yaml:32`), one empty window is enough: `D = 1`. With the standalone `learning_rate: 0.5` (`settings.yaml:32`), `D` halves per empty window.

**What a stakeholder gains.** Two distinct things, treated as two findings.

- LB-001: once `D` is far below the value of the eligible notes, the threshold `v·(t₀ + t₁·v)` computed in `F_p` (`leader_proof.rs:280-283`, PoL §Circuit Constraints step 5) no longer approximates `1 − (1−f)^{v/D}`; it is a quadratic in `v` reduced modulo `p`, which the PoL specification's own §Corner Case describes as an "oscillation between near-certain win and near-certain loss". Every eligible note then wins a value-determined fraction of every slot, and a note whose value is chosen for it wins every slot. No nonce knowledge is needed. This is available to **every** holder of an aged note, not only to the producer of `B_last`, and it lasts until `D` climbs back, about ten epochs.
- LB-002: independently of `D`, the producer of `B_last` knew `η(B_last)` before choosing the block's transactions, so notes ground in `B_last` against `η(B_last)` win their chosen slots of epochs `e+2 … r(+1)` on the canonical chain, without any fork. That needs `B_last` to be the last block, which the producer knows only for an announced halt or by grinding speculatively in every block it produces.

**Can the stakeholder cause the halt?** Not through the protocol: a stakeholder below one half cannot stop honest leaders from producing blocks on the canonical chain, and a halt on a private fork is worth nothing (§4.1). A halt is an operational event: an outage, a bug, a coordinated stop for an upgrade, or a test network left idle. What the stakeholder can do is prepare for one, which LB-002 quantifies, and profit from one, which anyone can do under LB-001.

### 4.4 Item 4 — quantifying the grind

A note ground to win a chosen slot needs, in expectation, `p / threshold(v)` candidate tickets, where `threshold(v) = v·(t₀ + t₁·v) mod p` with `t₀ = ⌊t₀_const / D⌋`, `t₁ = p − ⌊t₁_const / D²⌋` (`lottery.rs:131-140`). Each candidate is a fresh secret key (a new `pk`, hence a new `noteID`, `core/src/mantle/ledger.rs:512-527`) and costs one ticket hash plus the key derivation and note-id hash, so a small constant times the ticket cost. Ticket cost measured with the node's own hasher (Appendix B.2): **141 µs on one Raspberry Pi 5 core**; the #45 report measured 28 µs on an Apple M3 Pro core. With stake share `α` of the inferred stake `D` split into `k` notes of value `αD/k`, one per slot of a `k`-slot run:

| Constants | Stake share | Value per note | Tickets per note | Tickets for `k` slots | Core-time at 28 µs | Core-time at 141 µs |
|---|---|---|---|---|---|---|
| Specified, `k = 2160`, `f = 1/30`, `D = 2.03·10¹⁵` | 10% | 9.4·10¹⁰ | 6.4·10⁵ | 1.4·10⁹ | 10.7 h | 54 h |
| | 1% | 9.4·10⁹ | 6.4·10⁶ | 1.4·10¹⁰ | 4.5 d | 22 d |
| | 0.1% | 9.4·10⁸ | 6.4·10⁷ | 1.4·10¹¹ | 45 d | 225 d |
| Devnet/testnet, `k = 120`, `f = 1/30`, same `D` | 10% | 1.7·10¹² | 3.5·10⁴ | 4.2·10⁶ | 2 min | 10 min |
| | 1% | 1.7·10¹¹ | 3.5·10⁵ | 4.2·10⁷ | 20 min | 1.7 h |
| | 0.1% | 1.7·10¹⁰ | 3.5·10⁶ | 4.2·10⁸ | 3.3 h | 17 h |
| | 0.01% | 1.7·10⁹ | 3.5·10⁷ | 4.2·10⁹ | 33 h | 7 d |
| Standalone, `k = 30`, `f = 1/20`, `D = 1.0·10¹⁴` | 1% | 3.3·10¹⁰ | 5.8·10⁴ | 1.8·10⁶ | 49 s | 4 min |
| | 0.1% | 3.3·10⁹ | 5.8·10⁵ | 1.8·10⁷ | 8 min | 41 min |
| | 0.01% | 3.3·10⁸ | 5.8·10⁶ | 1.8·10⁸ | 1.4 h | 7 h |

`D` is the sum of the shipped genesis stakes (`stakeholders.yaml`; 2.032·10¹⁵ on devnet and testnet, 1.002·10¹⁴ on standalone). The work is proportional to `k²/(f·α)`, parallelises perfectly, and the stake is not consumed: the notes are ordinary UTXOs the adversary can spend again, so the only recurring cost is fees. A block in every slot of a **whole epoch** costs `10/f` times a `k`-slot run, since the epoch has `10k/f` slots. When `D` has collapsed (LB-001) the table is moot: at `D = 1` a note of value `59` wins 99.98% of slots with no grinding at all.

### 4.5 Item 5 — a conservative alternative, and what it would cost

Four candidates were costed. None was implemented.

1. **Refuse eligibility to notes created in the block that fixes the nonce** (the issue's proposal). Specification: one clause in §Epoch State Pseudocode, "if the block fixing `η^{ep}` is also the last block before `sl_{ep−1}`, `C_LEAD^{ep}` is the root after its parent", plus a sentence in §Note Ageing. Code: `LedgerState` would need to carry the previous block's `utxos` root alongside its own (one more `UtxoTree` handle per state; the tree is persistent, so the clone is cheap), and cases 2 and 3 of `update_epoch_state` would pick it when the two snapshot blocks coincide. A consensus change, so a version bump. Benefit: small. It moves the attacker from "produce the last block" to "produce the last two blocks" (`α²` instead of `α` per halt), does nothing for LB-001, and does nothing for private forks, which fork choice already handles.
2. **Age notes by block count instead of slot count.** Define `C_LEAD^{ep}` as the root after the block `N` blocks before the one that fixes `η^{ep}` (with `N` on the order of `3k`), whichever is earlier of that and the current slot rule. Then a single producer can never fix both unless it produced `N` consecutive canonical blocks. Specification: a new helper in §Epoch State Pseudocode. Code: the ledger must reach the root `N` blocks back; online, states are retained down to the LIB, `k` blocks deep (`prune_immutable_blocks`, `lib.rs:639-649`), so `N ≤ k` is available for free and larger `N` needs a ring of roots. Moderate. Still does nothing for LB-001.
3. **Floor the inferred stake.** Keep `D ≥ max(v)` over the eligible notes, or `D ≥ c · Σv` for a small `c`, at the epoch transition. The ledger has both values at hand (`from_utxos` already sums `utxos`, `mod.rs:743-749`). This removes the wrapping regime entirely, since `v/D ≤ 1` keeps the Taylor threshold inside its valid domain (PoL §Error Analysis). Specification: one line in `cryptarchia-total-stake-inference.md` §Algorithm and an update to the PoL corner-case section. Code: a running maximum in `EpochState` and one `max` in `update_epoch_state`. Cheap, and it is the change that matters. A lower `β` (the standalone `0.5`) only slows the collapse.
4. **A restart ceremony.** For a halt long enough to empty a window, treat the restart like genesis: an out-of-band nonce (the genesis ceremony's three-source construction, `bedrock-genesis-block.md` §Epoch Nonce Ceremony) and an explicit `D` written into the first post-halt epoch by a version-gated rule. That closes LB-002 as well, since the nonce after `B_last` stops being the nonce anyone lotteries under. Cost: a documented procedure, a config field, and a rule in `update_epoch_state` keyed on `bedrock_version`. It is the only candidate that addresses the case where the producer of `B_last` prepared.

### 4.6 Checked and ruled out

- **A block exactly at the snapshot slot.** Excluded from both snapshots (`ledger.slot < snapshot_slot`, `mod.rs:118, :146`), as #45 found; nothing in this issue changes that.
- **Density counts uncles.** No: `Branch::length` is blocks only (`lib.rs:243-247`), and `walk_back_before` follows `parent` links. The inference window does count uncle slots (`block_density.rs:44-55`), so a post-halt block cannot inflate a past window either, since a block outside the window is skipped entirely (`:45-47`).
- **A fork admitted by the density rule because the honest chain's window is empty too.** If the halt begins right after `B_last` and `B_last` is public, both chains hold exactly the same blocks in the window, the densities tie, and `cmax` keeps the first-seen chain (`lib.rs:109`). A fork therefore never wins the density rule on a halt; it wins the longest-chain branch of either rule once its ground notes give it blocks, which is the LB-002 scenario and needs `m ≤ k`.
- **The proposer and the validator disagree on the synthesised state.** `epoch_state_for_slot` (`mod.rs:702-720`) and `try_apply_header` (`:518-536`) run the same `update_epoch_state`; the test at `:2271-2320` pins it.
- **Integer wrap on the halt path.** `strict_*` throughout `config.rs`; the skipped-epoch loop bounds are `u32` epochs (`mod.rs:376`); the inference is `i128` with a floor (`stake.rs:37-67`). Nothing wraps silently. The `epochs_skipped` field is only logged.
- **The standalone constants (`k = 30`, `f = 1/20`) as cited by the issue.** The devnet and testnet templates now carry `k = 120`, `f = 1/30`, `β = 1.0`; only `settings.yaml` and the standalone template carry `k = 30`, `f = 1/20`, `β = 0.5`. All three sets were checked; the conclusions of §4.1 and §4.2 hold for each, and the numbers above name the set they belong to.

### LB-001 · A halt that empties one inference window collapses the inferred stake to 1, after which leadership is a function of note value, every slot has several leaders, and any note of a chosen value wins every slot for about ten epochs

| | |
|---|---|
| Severity | High |
| Difficulty | Low |
| Category | Consensus |
| Target | `ledger/src/cryptarchia/stake.rs:27-70` (`fn total_stake_inference`, `β` update and `.max(1)`); `ledger/src/cryptarchia/mod.rs:304-307, 371-380` (the zero-density transitions); `core/src/proofs/leader_proof.rs:273-283` (`fn check_winning`, `fn phi_approx`); `zk/proofs/pol/src/lottery.rs:131-140`; `deployment/ceremony/genesis/{devnet,testnet}/deployment-template.yaml:32` (`learning_rate: 1.0`) |
| Status | Open |

**Description**

The inference update is `D' = D − β·D·(expected − measured)/expected`, floored at 1 (`stake.rs:37-67`), with `β = learning_rate`, `expected = 6k/f · f = 6k`, and `measured` the number of occupied slots in the first `6k/f` slots of the epoch (`block_density.rs:29-36`). A window with no block gives `D' = D(1−β)`: `1` at the specification's and the devnet/testnet `β = 1.0`, `D/2` at the standalone `0.5`. Case 3 of `update_epoch_state` applies it once per skipped epoch (`mod.rs:376-380`), and case 2 applies it with whatever the last epoch's window measured (`:304-307`), which is `0` when the halt began before the window opened. The test `test_epoch_state_for_slot_with_empty_epochs` documents the outcome ("With 0 density and LEARNING_RATE=1, total stake drops to minimum (1)", `mod.rs:2241-2245`).

At `D = 1`, `t₀ = t₀_const ≈ 0.0339·p` and `t₁ = p − t₁_const ≈ p − 0.000575·p`, so the threshold is `v·t₀ − v²·t₁_const mod p`. It is the parabola the PoL specification's §Corner Case describes: it peaks at `v ≈ 29` and crosses zero at `v ≈ 58`, and beyond that it is, for practical purposes, a pseudo-random fraction of `p` determined by `v`. Reproduced exactly with the node's integer formulas (Appendix B.1), for `f = 1/30` and the shipped genesis note values:

| Note value | `P(win)` per slot at `D = 1` | Notes with this value on devnet/testnet |
|---|---|---|
| `10¹⁴` | 0.170 | 4 |
| `2·10¹⁴` | 0.409 | 8 |
| `8·10¹²` | 0.328 | 4 |
| `10⁹` | 0.261 | 4 |
| `59` | 0.9998 | any |

Ordinary operation gives the `10¹⁴` note `0.0017` per slot. With the twenty genesis notes eligible, the expected number of leaders per slot is **6.3** and the probability that a slot has no leader is `4·10⁻⁴`. Of the first million note values, 10,078 (1.0%) win more than 99% of slots. The regime persists: with nearly every slot occupied the next update multiplies `D` by about `1 + β(1/f − 1) = 30`, so `D` goes `1, 30, 909, 2.8·10⁴, 8.3·10⁵, 2.5·10⁷, 7.7·10⁸, 2.3·10¹⁰, 7.0·10¹¹, 2.1·10¹³, 6.1·10¹⁴, 2.0·10¹⁵` (Appendix B.1), eleven epochs, with 6 to 14 expected leaders per slot through the eighth and the `2·10¹⁴` notes still in the wrapping regime (`v > 58·D`) until `D` passes `3.4·10¹²`. On the devnet and testnet templates that is 110 hours; at the specified constants, eleven weeks. On the standalone template the collapse is gradual: the `10¹⁴` note goes from `0.05` per slot to `0.10`, `0.18`, `0.33`, `0.48` over four empty windows and `0.91` at six (Appendix B.1).

The node implements both specifications faithfully (`β = 1.0` and `max(…, 1)` are in `cryptarchia-total-stake-inference.md` §Algorithm; the wrapping threshold is the circuit's `t := v(t₀ + t₁·v)` in `F_p`). The PoL specification's §Corner Case names "chain halt and partial restart" as a cause and concludes the effect is "arguably beneficial" for liveness and that "no circuit-level mitigation is strictly necessary". That conclusion is what this finding disputes: the effect is not one big note winning aggressively, it is every note winning at a rate unrelated to its stake, several leaders in every slot, and a lottery anyone can win outright by picking a note value.

**Exploit scenario**

A devnet or testnet stops producing blocks for six hours (an outage that spans the first `6k/f` slots of an epoch; the 20-minute grace period plus the one-hour prolonged bootstrap period of LB-003 count toward it). On restart, the first block's `update_epoch_state` sets `D = 1` for the new epoch. From then on: (a) about six of the twenty genesis notes win each slot, so every slot produces several competing blocks; forks of equal length are broken by first-seen only (`lib.rs:135`), grow at one block per slot on every side, and after `k = 120` blocks, two minutes at one block per slot, nodes that saw different first blocks have different `k`-deep LIBs and `maxvalid_mc` rejects each other's chains for good, a consensus split with no adversary; (b) any participant who anticipated the outage (an announced upgrade window, a weekend) has, before the halt, sent itself a note of value `59`, or any of the 1% of values that win, and that note, being older than one epoch, wins 99.98% of the slots of every epoch until `D` exceeds `v/58`; a note of value `10¹²`, one half of one percent of a `2·10¹⁴` genesis note, stays in the winning regime through the eighth post-halt epoch; (c) any participant who did not anticipate it creates, in the first post-halt epoch, a hundred notes of distinct values above `58·D_{+2}` (a few million units in total; at `D = 900`, 1.1% of such values win more than 99% of slots, Appendix B.1) and holds an always-win note two epochs later, since `D_{+2} = 30·D_{+1}` is predictable once every slot is occupied. Whoever holds an always-win note collects every block reward and censors every transaction for as long as the regime lasts, and, producing a block in every slot, also supplies every contribution to the epoch nonce, which is LB-002's perpetuation mechanism.

**Recommendation**

- *Short term*: floor the estimate at the largest eligible note value, `D' = max(D', max_{u ∈ C_LEAD} value(u))`, or at a fixed fraction of the eligible stake, in `update_epoch_state` at both transitions; both quantities are available from `epoch_state.utxos`. Independently, cap the per-window correction (for example `β·(expected − measured)/expected ≤ 0.9`) so that one empty window cannot take `D` below a tenth of its value.
- *Long term*: raise the corner-case conclusion with the specification editors (S-001) and define the restart procedure (S-002) so that an outage longer than a window is followed by an explicit reset of `D` rather than by the inference's floor. A test that restarts a two-node network after an idle window and asserts a bounded leader count per slot would keep the regime from returning.

**References**: `cryptarchia-total-stake-inference.md` §Definitions (`β = 1.0`), §Algorithm (`max(…, 1)`); `cryptarchia-proof-of-leadership.md` §Corner Case: Note Value Exceeding Inferred Total Stake, §Error Analysis; `fork-choice.md` §Online Fork Choice Rule (first-seen tie-break); `cryptarchia-v1-protocol.md` §Epoch State Pseudocode. Related: `38-LB-001` (#453) on schedule parameters; `#147` on the lottery approximation.

### LB-002 · The producer of the last block before a halt fixes the eligible notes, the nonce and the difficulty of every epoch whose snapshots fall inside the halt, and a note set ground at that block wins every slot of those epochs on the canonical chain

| | |
|---|---|
| Severity | High |
| Difficulty | High |
| Category | Consensus |
| Target | `ledger/src/cryptarchia/mod.rs:335-339` (case 2, `nonce: self.nonce`, `utxos: self.utxos`), `:408-431` (case 3, both epoch states from the parent), `:118-155` (`update_from_ledger`); `consensus/cryptarchia-engine/src/lib.rs:97-98, 134-135` (`m` counts blocks); `services/chain/chain-service/src/bootstrap/state.rs:11-55` (no restart-specific logic) |
| Status | Open |

**Description**

When no block exists between `B_last` (slot `sl_last`, epoch `e`) and the first post-halt block (epoch `r`), every epoch `j` in `e+2 … r`, and `r+1` if the restart came after `sl_r + 6k/f`, derives `C_LEAD^{j}`, `η^{j}` and (through LB-001) `D^{j}` from `B_last` alone (§4.3). Its producer knew `η(B_last)` before choosing the block's transactions, since `η_B = H(η_parent, ρ_LEAD, sl)` depends on nothing it did not already hold (`mod.rs:591-604`, #45 §4.1), and the notes it creates in `B_last` are members of `C_LEAD^{j}` because they exist at `sl_{e+1}`. Notes ground against `η(B_last)` and the predictable `D^{j}` (`1` after one empty window at `β = 1`, otherwise `D(1−β)^n`) therefore win their chosen slots of epochs `e+2 … r(+1)` **on the canonical chain**; no fork, no density comparison and no `k`-deep check is involved, because the honest chain has nothing after `B_last`. This is the #45 report's §4.3 second bullet with the fork-choice defence removed, which is exactly what the halt removes: `m` in both rules counts blocks (`lib.rs:97, :134`), the LIB does not move during a halt, and the honest chain produces no block to make either quantity grow.

The cost of the grind is the table of §4.4, on the pre-halt `D` if the notes are ground before the collapse is known; if the producer expects `D = 1` it needs no grind at all and only picks note values (LB-001). The gain is every slot of at least one full epoch: all block rewards, full censorship, and, since it then supplies every `ρ_LEAD` in the buffer phase of the epochs it monopolises, knowledge of the next nonces as well. Honest blocks in those epochs are produced at rate `f` (or at the LB-001 rate) and are orphaned each time the monopolist extends its own block at the next slot, since a chain with a block in every slot is always the longest one; the honest contributions never reach the canonical nonce, the monopolist grinds fresh notes in each captured epoch for the epoch after next, and the capture perpetuates until a coordinated hard fork. The foreknowledge requirement bounds who can do this: the producer of the last block before an **announced** halt (an upgrade freeze at a known slot, a scheduled maintenance window) with probability about `α` of being that producer, or any producer who grinds a fresh note set into every block it makes, at the §4.4 cost per block (20 core-minutes per block for a 1% holder on the devnet template; 4.5 core-days per block at the specified constants, which a 1% holder produces every 54 minutes, so about 120 cores running continuously), spending the previous block's ground notes into the next block's so that the capital is reused.

**Exploit scenario**

A testnet announces that nodes will be stopped at slot `X` for an upgrade and restarted the next day, longer than the ten-hour epoch of the template. A holder of 5% of the stake computes its winning slots of the current epoch (all fixed since the previous nonce snapshot), and for each of its slots in the last hour before `X` prepares a block carrying 120 self-transfers whose output keys were ground so that, under the nonce that block will produce and under `D = 1`, one output wins each of the first 120 slots of epoch `e+2`; two core-hours per candidate block. It publishes the block for its last winning slot before `X`; with probability about 5% no honest block follows before `X`, and the network stops. On restart, all nodes synthesise epoch `e+2` (or later) from that block, the ground notes are in `C_LEAD`, `η` is the one they were ground against, and the holder proposes a block in every slot. Every honest block is orphaned within one slot. In the buffer phase of epoch `e+3` only the holder's `ρ_LEAD` values enter the nonce, so `η^{e+4}` is known to it before `sl_{e+3}`, and the notes it creates in epoch `e+2` blocks for `C_LEAD^{e+4}` are ground against it. The chain has one leader until the operators fork it.

**Recommendation**

- *Short term*: document that a halt longer than an inference window must be followed by a coordinated restart that re-seeds the nonce and resets the stake estimate (S-002), and that the last block before an announced halt is a privileged position. Until a restart rule exists, the standalone `β = 0.5` and a floor on `D` (LB-001) at least keep the ground notes from winning every slot, since their thresholds are computed against a `D` the grinder had to guess.
- *Long term*: option 4 of §4.5, a version-gated restart rule in `update_epoch_state` that, for the first post-halt epoch, takes `η` from a configured ceremony value and `D` from a configured estimate, so that no single block ever fixes the eligible notes and the nonce of the same epoch; or option 2, ageing by block count, which makes the position unreachable without `N` consecutive canonical blocks.

**References**: `cryptarchia-v1-protocol.md` §Epoch Schedule (buffer phase, "at least one honest leader"), §Note Ageing, §Epoch State Pseudocode; `fork-choice.md` §Online Fork Choice Rule; `cryptarchia-v1-bootstr-sync.md` §Setting the Fork Choice Rule; `processed/45-epoch-nonce-grinding.md` §4.3 and S-003.

### LB-003 · Every whole-network restart after an outage longer than 20 minutes adds a further hour without blocks, since all nodes come back in bootstrap state and none proposes until the prolonged bootstrap period ends

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `services/chain/chain-service/src/bootstrap/state.rs:30-55` (`check_offline_grace_period`); `bootstrap/config.rs:30` (20-minute default); `nodes/node/binary/src/config/cryptarchia/serde/service.rs:49` (`prolonged_bootstrap_period` default 1 h); `services/chain/chain-leader/src/lib.rs:439-444` (no proposal before online) |
| Status | Open |

**Description**

A node restarted more than `grace_period` after its last recorded state enters `Bootstrapping` (`state.rs:37-39`), stays there for `prolonged_bootstrap_period` (`pbp.rs:86`), and does not propose in the meantime (`chain-leader/src/lib.rs:441`). The specification intends this for a single node rejoining a live network, whose peers keep producing blocks. When **every** node restarts after the same outage, none of them proposes for the whole period: the outage is extended by at least one hour, unconditionally, and by 24 hours if the specification's `T_boot` were configured. On the standalone template, where an inference window is one hour, the extension alone empties a window if the restart falls at an epoch boundary, and on every template it moves a restart closer to the six-hour threshold of LB-001. No shipped configuration sets `force_bootstrap`, `prolonged_bootstrap_period` or the grace period, so the defaults apply everywhere.

**Exploit scenario**

Not an attack. A twenty-five-minute network-wide outage on a standalone network at slot `sl_e − 100` becomes an eighty-five-minute one, covering `[sl_e, sl_e + 3600)`, and the next epoch starts with `D/2`; two such restarts a day apart put the largest note at `0.18` wins per slot instead of `0.05`.

**Recommendation**

- *Short term*: document that a whole-network restart should be done with `force_bootstrap: false` and a zero `prolonged_bootstrap_period` on the nodes that hold the canonical tip, or that the operators must accept the added hour when reasoning about LB-001's threshold.
- *Long term*: let the prolonged bootstrap period end early when the node sees that its IBD peers' tips are its own tip and no peer has a longer chain, which is the case for a coordinated restart; the specification's rationale for `T_boot` (comparing with a broader peer set) is satisfied trivially then.

**References**: `cryptarchia-v1-bootstr-sync.md` §Constants, §Prolonged Bootstrap Period, §Proposing New Blocks, §Offline Grace Period.

## 5. Suggestions (non-security)

### S-001 · Revise the conclusion of the PoL specification's corner-case section

`cryptarchia-proof-of-leadership.md` §Corner Case: Note Value Exceeding Inferred Total Stake describes the wrapping threshold correctly and lists "chain halt and partial restart" as a cause, but concludes that the effect is "arguably beneficial" and that no mitigation is necessary. With `β = 1.0` and a floor of `1` in `cryptarchia-total-stake-inference.md`, a halt does not put one large note above the estimate; it puts every note there, gives every slot several leaders, and makes the win rate a function of the note's value. The section should state the leader-count consequence, the value-selection consequence, and the recovery time (about eleven epochs at full occupancy), and the stake-inference specification should carry the floor of LB-001 or a bounded correction.

### S-002 · Document a restart procedure for a halted chain

Nothing in the node repository or the specifications says what operators do after the chain has produced no block for longer than an inference window. The pieces exist: the genesis ceremony tool produces a nonce from external entropy, `bedrock_version` schedules rule changes by height, and `update_epoch_state` is the single place where the post-halt epoch is synthesised. A one-page procedure that names the threshold (one inference window), the values to agree on (nonce, stake estimate, restart slot), and the flag that applies them would turn LB-001 and LB-002 from silent behaviour into an operational rule. The devnet and testnet templates should state it next to `learning_rate`.

### S-003 · Say in the specification that the `k`-deep rule and the ageing argument are stated in blocks and honest liveness

`fork-choice.md` §How This Attack is Mitigated by the Praos Fork Choice Rule assumes "nodes see honest blocks reasonably quickly", and `cryptarchia-v1-protocol.md` §Epoch Schedule assumes at least one honest leader in the buffer phase. Both are liveness assumptions, and both are what a halt removes at once. A sentence next to each, saying that after any period without canonical blocks longer than `6k/f` slots the two assumptions no longer hold and the restart procedure of S-002 applies, would keep the dependency visible; the #45 report's S-003 asked for the same at the ledger's two epoch-transition branches, and this report adds the fork-choice side.

---

## Appendix A — Definitions

Severity, difficulty and categories are those of `docs/REPORT_TEMPLATE.md`, Appendix A, unchanged.

## Appendix B — Reproduction

All programs lived outside the node checkout, which was left unmodified.

### B.1 Threshold and inference arithmetic (`calc.py`, `calc2.py`)

Exact replicas of `LotteryConstants::compute_lottery_values` (`lottery.rs:131-140`: `t₀ = t₀_const // D`, `t₁ = p − t₁_const // D²`), `LeaderPublic::phi_approx` (`leader_proof.rs:280-283`: `v·(t₀ + t₁·v) mod p`) and `StakeInference::total_stake_inference` (`stake.rs:27-70`, with `PRECISION = 1000` and the same truncations), in Python big integers. `t₀_const` and `t₁_const` for `f = 1/30` were recomputed from `p·(−ln(1−f))` and `p·ln²(1−f)/2` at 200 decimal digits and compared equal to the two hex constants in `cryptarchia-proof-of-leadership.md` §Lottery Approximation; the `f = 1/20` pair was derived the same way.

```python
p = 0x30644e72e131a029b85045b68181585d2833e84879b9709143e1f593f0000001
def lottery(t0c, t1c, D): return t0c // D, p - t1c // (D * D)
def threshold(v, t0, t1): return (v * (t0 + t1 * v)) % p          # phi_approx in F_p
def win_probability(v, t0c, t1c, D):
    t0, t1 = lottery(t0c, t1c, D); return Fraction(threshold(v, t0, t1), p)
def inference(D, measured, beta, f_num, f_den, k):                   # stake.rs, PRECISION = 1000
    P = 1000; period = 6 * (k * f_den); expected = period * int(f_num / f_den * P)
    err = D * P * (expected - measured * P) // expected
    corr = int(beta * P) * err // P
    return max((D * P - corr) // P, 1)
```

Output used in the report (`f = 1/30` unless stated):

```
D=1: v=1 -> 0.033327 | v=29 -> 0.499858 | v=58 -> 0.033142 | v=59 -> 0.999808
     v=1e9 -> 0.260642 | v=8e12 -> 0.327808 | v=1e14 -> 0.169597 | v=2e14 -> 0.408741
values in [1, 1e6] with P(win) > 0.99 at D=1: 10078 (first: 59, 118, 171, 177, 236)
sanity, normal regime: D=2.032e15, v=1e14 -> 0.001667 (exact 1-(1-f)^(v/D) = 0.001667)
devnet genesis set, D=1: expected leaders/slot 6.30, P(no leader) 4.33e-4
recovery from D=1 at full occupancy (k=120, beta=1): 1, 30, 909, 2.75e4, 8.35e5, 2.53e7,
     7.67e8, 2.32e10, 7.04e11, 2.11e13, 6.14e14, 1.97e15; leaders/slot 6.3..14.3 through epoch +7
standalone (f=1/20, beta=0.5), 1e14 note after n empty windows:
     n=0 0.0499 | 1 0.0971 | 2 0.1838 | 3 0.3257 | 4 0.4836 | 5 0.2964 | 6 0.9095
values in [58*900+1, 58*900+100000] with P(win) > 0.99 at D=900: 1130
```

The grind table of §4.4 is `p / threshold(αD/k)` times `k`, at `D` equal to the sum of the shipped genesis stakes.

### B.2 Ticket cost (`ticketbench`)

A crate with `logos-blockchain-poseidon2 = { path = "<node>/zk/poseidon2" }`, computing `Poseidon2Bn254Hasher::digest(&[LEAD_V1, η, Fr(slot), noteID, sk])`, the expression of `LeaderPublic::ticket` (`leader_proof.rs:285-287`), for 200,000 slots, `--release`, one thread, Raspberry Pi 5:

```
tickets: 200000 in 28.232s -> 7084 tickets/s, 141.16 us/ticket
tickets: 200000 in 28.213s -> 7089 tickets/s, 141.06 us/ticket
```

The 28 µs figure in §4.4 is the #45 report's measurement of the same expression on an Apple M3 Pro core, quoted for comparison.
