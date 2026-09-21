# Audit Report — Re-verification of empty-window stake-inference recovery

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/638`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `9ffddb30b9e6cf79465802953caedd010ff1cecd` — component(s): `ledger/src/cryptarchia`, `core/src/proofs`, `zk/proofs/pol`, `services/chain/chain-leader`, `zk/proofs/poq`
Specs: `https://github.com/logos-co/logos-lips` @ `75d3d0382604d4a0d8e246c268935dd386ffc8ea` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `cryptarchia-proof-of-leadership.md`, `cryptarchia-total-stake-inference.md` (in full); `cryptarchia-v1-protocol.md` sections `Epoch` and `Uncle References`, `fork-choice.md`
Date: `2026-09-21` — author: `Codex` — status: `final`

---

## 1. Summary

- Overall assessment: Static re-verification supports the existing `44-LB-001` finding: an empty inference window collapses the lottery difficulty `D` to `1` with the shipped `learning_rate: 1.0`, and the lottery approximation then permits stake-independent or highly irregular winning probabilities during recovery.
- Findings: `M` medium
- Key themes: `stake-inference floor`, `lottery approximation outside its monotone range`, `post-halt recovery load`
- Must-fix before launch: preserve the existing `44-LB-001` remediation priority and bound the empty-window recovery behavior before treating the finding as closed.

This iteration does not create a new independent finding or reclassify the canonical record. It re-verifies `44-LB-001`, filed as issue `#716`, retaining `Medium` severity, `High` difficulty, and `Consensus` category. It adds a host-like short-epoch three-node restart probe and a scratch bounded-decrease prototype. The private dust-note race and direct PoL/PoQ load measurements remain explicit follow-ups.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `ledger/src/cryptarchia/stake.rs` | Fixed-precision total-stake inference, floor, and unit tests. |
| `ledger/src/cryptarchia/mod.rs` | Epoch transitions, skipped-epoch updates, lottery constants, and synthesized epoch state. |
| `ledger/src/cryptarchia/block_density.rs` | Distinct occupied-slot accounting, including in-window uncle slots. |
| `zk/proofs/pol/src/lottery.rs` | `t0`/`t1` derivation and lottery constants. |
| `core/src/proofs/leader_proof.rs` | Field evaluation of the lottery threshold and ticket comparison. |
| `services/chain/chain-leader/src/leadership.rs` | Per-slot eligible-note scan, winning-proof generation, and blocking-task placement. |
| `zk/proofs/poq/src/chain_inputs.rs` | PoQ wiring of the same lottery constants. |
| `consensus/cryptarchia-engine` | Fork-choice behavior was consulted through the pinned implementation/spec context; no dynamic fork run was performed. |

**Out of scope**

The private dust-note race, direct PoL/PoQ generation and verification load measurement, and a production implementation of bounded `D` recovery were not performed. No source repository was modified. The bounded-decrease and restart tests were scratch-only evidence. The full fork-choice implementation was not independently re-audited beyond the paths needed to avoid duplicating issue `#634`.

**Assumptions**

The pinned Cryptarchia and Proof-of-Leadership specifications are authoritative. The prior report for issue `#44` and canonical finding issue `#716` are prior evidence used for deduplication, not a substitute for this source inspection. The scenario assumes an operational halt or another cause of a zero-density observation window; it does not assume that the reviewer has shown an attacker can force the halt.

## 3. Method

- Read issue `#638`, parent issue `#5`, issue `#44`, the prior report `processed/44-leader-threshold-stake-edge-cases.md`, canonical finding issue `#716`, and related follow-up issue `#634`.
- Read the mandatory architecture and Cryptoeconomics overviews and the issue-pinned Proof of Leadership and Total Stake Inference specifications in full. Read the requested Cryptarchia epoch/uncle-reference and fork-choice sections at the pinned revision.
- Manually inspected the pinned target revision with `git show`, `git grep`, and `git blame`, following the zero-density calculation into ordinary and skipped epoch transitions, lottery constant computation, leader-note scanning, proof generation, and PoQ input wiring.
- Automated tooling: scratch ledger tests covered the empty-density inference, skipped-epoch state, and bounded-decrease model. The prior `#44` targeted node tests and standalone lottery model remain prior evidence from the same target revision and were not relabeled as new measurements.
- Dynamic testing: a host-like pinned-revision local three-node probe used one-second slots, one-slot epoch phase parameters, `security_param=2`, and `slot_activation_coeff=1/2`. It stopped all nodes for 15 seconds, restarted their persisted state, sampled immediately, and sampled again after ten seconds of resumed operation.

### Dynamic confirmation and bounded-recovery prototype

The scratch bounded-recovery prototype was run with:

`cargo test -p logos-blockchain-ledger research_empty_window_bounded_recovery_prototype -- --nocapture`

Starting from `2,968,004,000,000,000`, the shipped zero-density update returned `1` immediately. A scratch bounded step using `max(unbounded_step, previous / 31)` produced:

`95,742,064,516,130; 3,088,453,694,069; 99,627,538,519; 3,213,791,566; 103,670,696; 3,344,216; 107,878; 3,480; 113; 4; 1; 1`

After ten unbounded recovery steps from `1`, the model reached `646,315,676,155,181`. This is a model of a candidate mitigation, not a recommendation to adopt the exact divisor or a production code change.

The affected pinned-ledger tests were also run successfully:

- `cargo test -p logos-blockchain-ledger test_total_stake_inference_zero_block_density -- --nocapture` — 1 passed.
- `cargo test -p logos-blockchain-ledger test_epoch_state_for_slot_with_empty_epochs -- --nocapture` — 1 passed.

The host-like local-network probe was run with:

`source ~/Code/logos/set_paths.sh && cargo test -p logos-blockchain-tests --test test_cli_restart research_empty_inference_window_restart_probe -- --nocapture`

The successful settled run captured these exact anchors. Before stopping, node 0 was height 3 / slot 13 / LIB slot 7 with tip `39a2ec3ac92c589c51a7edcee638c2e8da55f2f7af4976a98c1f9c943eb4058c` and LIB `bcda2e8909f127f9d1d3fd6d1533c79dd29590e6d0c06af532451b77054b061d`; node 1 was height 4 / slot 12 / LIB slot 9 with tip `02e32d6435f41b9a9e8701a16643e264bae0890405d61bc01b7edff36b9a9e3a` and LIB `7812067bc97304cc1bad241c50c5ece2e137a73a26349cbf92eeee35bba37122`; node 2 was height 3 / slot 12 / LIB slot 7 with tip `5aaee3d69a9c03c1b515b3efb7c80a5fb7c367b84bb1a928ea4ca68794b7f7a6` and LIB `bcda2e8909f127f9d1d3fd6d1533c79dd29590e6d0c06af532451b77054b061d`.

Immediately after restart, node 0 was at height 1 with tip/LIB `bcda2e8909f127f9d1d3fd6d1533c79dd29590e6d0c06af532451b77054b061d`; nodes 1 and 2 retained the above height/tip observations. Ten seconds later, node 0 was height 9 / slot 38 / LIB slot 36 with tip `b4450abf511dbe2bc51f8738b21275e24dcd016f1f5b1f8f4bca68df9b652690` and LIB `40faf056d68b5006ae5b71511fdd449d3cb6b272151c65e538411225a547edb7`; node 1 was height 11 / slot 37 / LIB slot 34 with tip `8bad9ab3a561b3ceedc7ca9135c9309f29ffae8fa8f73131400ecce5f2652307` and LIB `060c9ba652f8ea5f3f8117c322a0d498050b0512627a33130f9a53e2854993f0`; node 2 was height 9 / slot 39 / LIB slot 35 with tip `48e36683733f038fa14312340f6c682ef3eb78ede70acb6aa92ac8ee03664455c` and LIB `1eb681e19ac6fdaca73f8f2e9c0ad50a5277077d5fc46cab9b74c0f927c20856`.

The run therefore demonstrates the requested controlled restart and measurable post-restart tip/LIB divergence in the pinned harness. It does not by itself prove the private dust-note race or quantify PoL/PoQ proof-generation and verification load; those remain separate follow-ups.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| `44-LB-001` | One empty stake-inference window collapses `D` to one and leaves the lottery outside its weighted regime | Consensus | Medium | High | Re-verified; canonical issue `#716` remains open |

### `44-LB-001` · One empty stake-inference window collapses `D` to one and leaves the lottery outside its weighted regime

| | |
|---|---|
| Severity | Medium |
| Difficulty | High |
| Category | Consensus |
| Target | `ledger/src/cryptarchia/stake.rs:27-69` (`total_stake_inference`); `ledger/src/cryptarchia/mod.rs:257-449` (`update_epoch_state`); `zk/proofs/pol/src/lottery.rs:131-138` (`compute_lottery_values`); `core/src/proofs/leader_proof.rs:261-274` (`check_winning`, `phi_approx`) |
| Status | Open; canonical tracker issue `#716`; original source issue `#44` |

**Description**

At the pinned target revision, `StakeInference::total_stake_inference` calculates the density correction in fixed-point `i128` arithmetic and then clamps the result with `.max(1)` (`ledger/src/cryptarchia/stake.rs:27-69`). With `learning_rate: 1.0`, an observed density of zero makes the correction equal to the previous estimate, so the returned value is exactly `1`, regardless of the prior `D`.

The testnet and devnet deployment templates at this revision ship `security_param: 120`, `slot_activation_coeff: 1/30`, and `learning_rate: 1.0`. The ordinary epoch transition calls inference once using the current `BlockDensity` (`ledger/src/cryptarchia/mod.rs:300-310`). If a block arrives after one or more epochs, the skipped-epoch branch first applies the current density and then calls the same inference once with density `0` for each skipped epoch (`ledger/src/cryptarchia/mod.rs:367-383`). Thus a single empty inference window reaches `D = 1`, and multiple skipped epochs remain at the floor.

`LotteryConstants::compute_lottery_values` divides the fixed `t0` constant by `D` and the fixed `t1` constant by `D²` (`zk/proofs/pol/src/lottery.rs:131-138`). `LeaderPublic::check_winning` then evaluates `v·(t0 + t1·v)` in the BN254 field and compares the result with a Poseidon2 ticket (`core/src/proofs/leader_proof.rs:261-274`). The pinned Proof of Leadership specification describes this as a second-order approximation intended for small `v/D`, and explicitly documents a pathological regime beyond the second-order peak. At `D = 1`, ordinary note values can enter that regime immediately.

The same lottery constants are also passed into the Proof-of-Quota chain inputs (`zk/proofs/poq/src/chain_inputs.rs:70-88`), so the low `D` state is not confined to the local leader scan. The static review does not claim a new PoQ-specific vulnerability; it confirms that any downstream quota calculation using those inputs inherits the same recovery-state parameters.

**Recovery-load observations**

The leader implementation checks eligible UTXOs serially for each slot. `build_proof_for` returns after the first successful proof, so the static code does not support the stronger claim that one leader always generates one Groth16 proof for every winning note. It can perform multiple checks, and it can continue to later winners if private-input or proof generation fails. The Groth16 proof itself is dispatched through `spawn_blocking` (`services/chain/chain-leader/src/leadership.rs:120-145`). The epoch-wide winning-slot stream is lazy and creates one per-slot future; that future scans the aged UTXOs until it finds a winner (`services/chain/chain-leader/src/leadership.rs:465-550`). These facts identify the load path, but no node run measured its recovery throughput, verifier queue, or proof counts.

**Skipped-epoch ageing observation**

The skipped-epoch branch constructs both the synthesized current and next epoch states with `self.utxos.clone()` (`ledger/src/cryptarchia/mod.rs:407-440`). Here `self.utxos` is the current unspent tree, while `EpochState::utxos` is the aged snapshot used by the leader path. Therefore a note created in the last block before a halt can be present in the synthesized aged tree when the chain resumes across skipped epochs. This is an important static observation for the separate note-ageing/fork-choice follow-up `#634`; it is not promoted to a second finding in this report.

**Exploit scenario**

The precondition is a complete inference window with no occupied canonical or referenced-uncle slots, such as an outage or partial restart. On the next transition, the node computes `D = 1`; on later skipped epochs it continues applying the zero-density update. A stakeholder that already holds aged notes can then evaluate the lottery using constants derived from `D = 1`. Because the approximation is no longer monotone in note value in this range, a small pre-positioned note can win an unexpectedly large fraction of slots, while honest nodes may see an elevated number of winning candidates and proof checks during recovery. The controlled restart probe reproduced divergent post-restart tip/LIB progress, but did not isolate the private dust-note race or measure proof workload.

The prior `#44` report measured the node's own `check_winning` over 20,000 slots and recorded, at `D = 1`, a 59-unit note winning 19,996 slots and a 58-unit note winning 675 slots. It also modelled recovery from `D = 1` using the shipped testnet stake list. Those values are prior evidence from the same pinned target, not measurements repeated by this iteration.

**Recommendation**

- *Short term*: Bound the maximum downward change of `D` per epoch independently of `learning_rate`, and apply the same bound for each skipped epoch. Add regression tests that an empty window cannot move `D` directly from a realistic estimate to the floor, and that the recovery curve remains within the intended lottery domain. The scratch bounded model demonstrates the needed shape but does not select a protocol constant.
- *Long term*: Make the lottery threshold monotone over the supported note-value domain, or explicitly constrain the relation between eligible note values and `D` before exposing lottery constants to PoL/PoQ. Define the halt/restart semantics for note ageing and ensure synthesized epoch state cannot treat newly created notes as aged unless the protocol intentionally allows that behavior.
- *Validation*: Run issue #638's short-epoch multi-node test with a controlled empty window. Capture `new_total_stake`, winners per slot, blocks received per slot, tip growth, PoL generation/verifications, chain-service progress, and the exact note/nonce anchors. Run the bounded-decrease prototype only in a scratch copy and compare the existing zero-density tests and recovery model.

**References**: issue `#638`; parent `#5`; original issue `#44`; canonical finding `#716` (`44-LB-001`); related follow-up `#634`; `cryptarchia-proof-of-leadership.md` §Lottery Approximation and §Corner Case; `cryptarchia-total-stake-inference.md` §Algorithm; `cryptarchia-v1-protocol.md` §Epoch, §Total Stake Inference, and §Uncle References; `fork-choice.md`.

## 5. Suggestions

### S-001 · Complete the recovery race experiment

The short-epoch restart probe is now complete for tip/LIB observations. Add a controlled private dust-note created in the last pre-halt block, then trace its presence through the synthesized epoch state, PoL aged root, first post-restart proof, and fork choice. Record the `epoch transition` / `skipped epochs` values, winners per slot, blocks received per slot, and exact note/nonce anchors.

### S-002 · Measure the actual leader and verifier work

Instrument or collect metrics for eligible-note checks, successful and failed proof generations, `spawn_blocking` queue time, incoming PoL verification, chain-service application time, and proof backlog. The static implementation returns after the first successful block-proposal proof, so the experiment should distinguish note checks from generated proofs. The current restart probe intentionally reports no such load measurement.

### S-003 · Coordinate note-ageing semantics with issue #634

Test a note created in the last pre-halt block and follow its presence in `self.utxos`, the synthesized `EpochState::utxos`, the PoL aged root, and the first post-restart proof. Decide whether immediate eligibility is intended; if not, refuse or re-age those notes at the protocol/state-transition boundary.

### S-004 · Verify the PoQ parameter boundary

Using the same recovered epoch state, check that PoQ's `lottery_0` and `lottery_1` inputs are derived from the intended bounded `D` and that any mitigation applied to PoL is applied consistently to PoQ. Add a conformance test for the minimum supported `D` and the largest eligible note.

---

## Appendix A — Classification basis

The `Medium` / `High` / `Consensus` classification is preserved from canonical finding `44-LB-001` in issue `#716`. Under `docs/REPORT_TEMPLATE.md`, Medium covers realistic safety/liveness degradation or a costly denial of service, while High difficulty means the precondition requires privileged access, complex technical details, or discovery of another weakness. The dynamic restart and bounded-model evidence supports the existing classification and does not warrant a severity, difficulty, or category change. The report remains an independent re-verification of the canonical finding, not a new finding.

Draft pending independent review and explicit approval.
