# SDP Active PoQ pricing and batching — current-master re-verification

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/124`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `9ffddb30b9e6cf79465802953caedd010ff1cecd` — component(s): `core/src/mantle/ops/sdp`, `core/src/mantle/batch.rs`, `ledger/src/mantle/sdp`, `ledger/src/mantle/sdp/rewards/blend`, `blend/message/src/reward/activity.rs`, `zk/proofs/poq`, `zk/proofs/zksign`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-v1.1-block-construction.md`; by section `analysis-gas-cost-determination.md` (§ SDP Activation, § Gas determination from measures), `blend-protocol.md` (§ Active Message)
Date: `2026-09-19` — author: `Codex` — status: `final`

This is an evidence-reset re-verification of the canonical finding `124-LB-001` (#564), performed at the current `logos-blockchain` master `9ffddb30b9e6cf79465802953caedd010ff1cecd` rather than the superseded filing revision `a77c248f350014bfab7990233b35bcf61b450d05`. The finding identifier and classification are preserved.

---

## 1. Summary

- Overall assessment: the original pricing gap remains present at current master: `SDP_ACTIVE_GAS` charges for one deferred ZkSignature while the Blend activity proof's PoQ is verified synchronously and is not included in the block proof batch.
- Findings: `0 new findings; canonical 124-LB-001 / #564 re-verified — Medium / Medium / Economic / Incentive`
- Key themes: unpriced proof work, missing deferred PoQ path, block-validation budget
- Must-fix before launch: remeasure and price both proof components, or otherwise ensure the activity-proof cost is charged; batch PoQ verification before adopting block state.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `core/src/mantle/ops/sdp/active.rs` | `SDPActive` gas constant, cheap validation, and deferred ZkSignature construction |
| `core/src/mantle/batch.rs` | Deferred proof variants and block-level ZK verification |
| `ledger/src/mantle/sdp/rewards/blend/target_epoch.rs` | Blend activity-proof verification and reward-state update |
| `blend/message/src/reward/activity.rs` | PoQ and PoSel verification order in `ActivityProof::verify_and_build` |
| `zk/proofs/poq` and `zk/proofs/zksign` | Single and batched verification implementations and existing benchmarks |
| `ledger/src/lib.rs`, `tests/src/common/fee_spec.rs` | Execution-gas limit enforcement and block-gas test coverage |

**Out of scope**

The implementation of the Groth16 verifier and circuit artifacts, the cryptographic soundness of PoQ/PoSel, mempool admission and eviction, SDP declaration semantics outside active-message processing, and network/e2e behavior were not re-audited here. The third-party cryptographic dependencies and the trusted setup are assumed correct.

**Assumptions**

The pinned `logos-lips` revision is the intended specification for this review. A block's deferred proofs are checked before the resulting state is accepted, as required by the block-construction flow.

## 3. Method

- Re-read parent issue #8, issue #124, and the existing report `processed/124-sdp-active-poq-batching-and-gas.md`; reset the evidence to current master `9ffddb30b9e6cf79465802953caedd010ff1cecd` after confirming it is 24 commits ahead of the superseded `a77c248f` filing revision.
- Re-read `bedrock-v1.1-block-construction.md` in full and the specified sections of the gas-determination and Blend protocol documents at the pinned `logos-lips` revision.
- Manually traced current master from `SDPActive::verify` (`core/src/mantle/ops/sdp/active.rs:60-103`) through `ledger/src/mantle/sdp/mod.rs:506-545`, `TargetEpochState::verify_proof` (`ledger/src/mantle/sdp/rewards/blend/target_epoch.rs:79-124`), `ActivityProof::verify_and_build` (`blend/message/src/reward/activity.rs:41-65`), and `DeferredZkpVerifications::verify` (`core/src/mantle/batch.rs:10-68`).
- Ran `cargo bench -p logos-blockchain-poq --bench verify -- --sample-count 3` at current master: batch medians were 1.566 ms (1), 9.556 ms (16), and 13.07 ms (32); sequential verification of 32 was 43.1 ms.
- Ran `cargo bench -p logos-blockchain-zksign --bench verify -- --sample-count 3` at current master: batch medians were 1.745 ms (1), 6.731 ms (16), and 333.7 ms (1024); sequential verification of 1024 was 2.555 s.
- Ran `cargo test -p logos-blockchain-ledger --lib sdp`: 26 matching tests passed.
- No upstream code was changed. No full-node or cluster benchmark was run.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| 124-LB-001 / #564 | `SDP_ACTIVE_GAS` omits the Blend PoQ verification | Economic / Incentive | Medium | Medium | Open; canonical finding re-verified; no new finding |

### 124-LB-001 / #564 · `SDP_ACTIVE_GAS` omits the Blend PoQ verification

| | |
|---|---|
| Severity | Medium |
| Difficulty | Medium |
| Category | Economic / Incentive |
| Target | `core/src/mantle/ops/sdp/active.rs:43-44,60-103` (`GAS_COST`, `verify`); `ledger/src/mantle/sdp/rewards/blend/target_epoch.rs:79-124` (`verify_proof`); `blend/message/src/reward/activity.rs:41-65` (`verify_and_build`); `core/src/mantle/batch.rs:10-68`; `zk/proofs/poq/src/lib.rs:114-139`; `ledger/src/lib.rs:484-559` |
| Status | Open; canonical `124-LB-001` / #564 re-verified; no new finding |

**Description**

`SDPActiveOp` still has a fixed execution-gas cost of `590`. Its `verify` method performs the declaration, withdrawal, and nonce checks, then returns only a deferred `ZkSig` proof. The Blend activity proof is processed later through `update_rewards`: `TargetEpochState::verify_proof` calls `ActivityProof::verify_and_build`, which synchronously verifies the PoQ and PoSel and then evaluates the activity threshold. `DeferredZkpVerifications` still has variants for ZkSignatures and leader-claim PoC proofs, but no PoQ variant; the available `lb_poq::batch_verify` function is still not used by the ledger.

The specification's gas analysis assigns `SDP_ACTIVE_GAS = 590` to the ZkSignature and explicitly says that service-specific activity evaluation is neglected. Its measured batch table likewise contains only ZkSignatures and Proofs of Claim. The block-construction specification describes batched PoC and ZkSignature checks, but not the PoQ carried by a Blend Active Message.

The current-master benchmarks confirm that the omitted work remains material. With valid proofs and three samples, the PoQ benchmark reported medians of `1.566 ms` for one batched proof, `9.556 ms` for 16, and `13.07 ms` for 32; sequential verification of the same 32 proofs took `43.1 ms`. The corresponding ZkSignature benchmark reported `1.745 ms`, `6.731 ms`, and `333.7 ms` for batches of 1, 16, and 1024; sequential verification of 1024 took `2.555 s`. Exact gas values still need the specification's CPU-cycle calibration on reference hardware, but charging `590` for one proof while executing two proof systems leaves the PoQ unpriced. The earlier recommendation to rederive the charge for both proof components remains valid; the current-head evidence does not justify downgrading the canonical finding.

`ledger/src/lib.rs` still rejects a block once accumulated execution gas exceeds `EXECUTION_GAS_LIMIT = 3,193,460`, but that check uses the declared per-operation cost. A block can therefore contain thousands of `SDPActive` operations under the gas limit while every validator performs the additional synchronous PoQ checks. The fee-test helper maps only transfers and has no block-fill test for `SDPActive`, so the repository does not currently demonstrate that a gas-valid block stays within the one-second validation budget.

**Exploit scenario**

An eligible leader includes a gas-valid block containing a large number of `SDPActive` operations, or includes well-formed operations with invalid activity proofs so the block is rejected only after validation work has been performed. Every validator executes the Blend PoQ checks synchronously, then performs the deferred ZkSignature batch. The operation's fee and block gas accounting charge only the ZkSignature component, allowing the block to consume substantially more CPU than the execution-gas budget models. The exact wall-clock amplification depends on hardware and the number of valid activity messages, so this report does not claim a new full-node timing result.

**Recommendation**

- *Short term*: split Blend activity-proof processing at the PoQ pairing check. Run the cheap, state-binding checks that do not require a verified PoQ, return the PoQ proof and verifier inputs as a distinct pending value, and add a `PoQ` variant to `DeferredZkpVerification`. Batch it with `lb_poq::batch_verify` and reject the block before adopting any state when the batch fails. Do not construct a type named `Verified` before the batch has actually passed.
- *Short term*: remeasure the PoQ and ZkSignature marginal and fixed batch costs using the reference hardware and revise `SDP_ACTIVE_GAS` to charge for both. Account for any additional batch-initialization cost in the block budget, following the gas-analysis rule for shared initialization.
- *Long term*: extend the block-construction and gas-analysis specifications to cover service-specific activity proofs, and add a regression benchmark/test that fills the execution-gas limit with each proof-bearing operation and checks the validation budget.

**References**: `analysis-gas-cost-determination.md` § SDP Activation, § Execution Gas, and § Gas determination from measures; `bedrock-v1.1-block-construction.md` § Batch verification of ZK proofs; `blend-protocol.md` § Active Message; prior report PR #256 and canonical finding #564.

## 5. Suggestions

### S-001 · Make the deferred proof state explicit

The natural deferral seam is `ActivityProof::verify_and_build`, but a pending PoQ must not be represented by an already-verified proof type. A dedicated pending proof type, or a state parameter on the Blend token, would make it harder to accidentally consume the PoQ before the block batch passes. This is a maintainability and correctness guard, not a separate security finding.

### S-002 · Add an SDP Active execution-gas budget test

The current fee fixture intentionally maps only transfer operations and rejects other operation kinds. Add a focused test or benchmark that constructs a block near `EXECUTION_GAS_LIMIT` with `SDPActive` operations, records the PoQ and ZkSignature work separately, and keeps the measured assumptions synchronized with the gas-analysis document.
