# Audit Report — SDPActive proof-check ordering and deferred PoQ verification

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/266`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `ledger/src/mantle/sdp/rewards/blend`, `blend/message/src/reward`, `core/src/mantle/batch.rs`, `services/chain/chain-leader/src/lib.rs`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `bedrock-service-declaration-protocol.md`, `bedrock-service-reward-distribution.md`, `bedrock-anonymous-leaders-reward.md`, `blend-protocol.md`
Date: `2026-09-15` — author: `Codex (GPT-5)` — status: `draft`

---

## 1. Summary

- Overall assessment: the pinned upstream target remains unfixed, but this audit found no new finding: canonical issue [#562](https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/562) (`125-LB-001`) was re-verified at the same source commit. Upstream [PR #3545](https://github.com/logos-blockchain/logos-blockchain/pull/3545) implemented most of the requested change on that revision and was closed unmerged for process/ownership reasons.
- Findings: `0` critical · `0` high · `0` medium · `0` low · `0` informational
- Previously identified finding re-verified: [#562](https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/562) (`125-LB-001`) — Low severity · Low difficulty · Denial of Service
- Key themes: `expensive proof before cheap rejection`, `public API seam for deferred PoQ verification`, `test-double drift`
- Follow-up before upstreaming: coordinate the eventual implementation with #124's deferred-PoQ batching seam, retain the duplicate guard inside `TargetEpochTracker::insert`, and add a builder-level regression test showing one PoQ verification for the first accepted message and none for later duplicate attempts.

The canonical issue is #562; this report records a re-verification rather than a new finding under #266. PR #3545 is the current implementation evidence: against the same `a805329f8` revision it implemented the duplicate → PoSel → threshold → PoQ order, split `verify_selection_and_evaluate` from `verify_quota`, consolidated the ledger and blend-message verifier doubles, and added stronger call-counting tests. It was reviewed but closed without merge for process/ownership reasons, not because the implementation was technically disproved. The audited target therefore remains unfixed. The older Appendix B prototype in PR #264 is retained only as historical context; the eventual upstream shape should coordinate directly with #124's deferred-PoQ work.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `ledger/src/mantle/sdp/rewards/blend/mod.rs` | `Rewards::update_active`, the call order, and the existing verifier doubles |
| `ledger/src/mantle/sdp/rewards/blend/target_epoch.rs` | `TargetEpochState::verify_proof` and `TargetEpochTracker::insert` |
| `blend/message/src/reward/{activity,epoch,token,mod}.rs` | `verify_and_build`, token evaluation, and the unverified-token API needed by the prototype |
| `blend/message/src/crypto/proofs.rs` | The `ProofsVerifier` abstraction and its private PoQ public-input state |
| `core/src/mantle/batch.rs`, `core/src/mantle/ops/sdp/active.rs` | The existing deferred-proof variant and `SDPActive` validation boundary |
| `services/chain/chain-leader/src/lib.rs` | Candidate transaction trial execution and retry loop in `propose_block` |
| test utilities in `blend/message`, `blend/network`, `blend/provers`, `services/blend` | Inventory of PoQ verifier doubles |

**Out of scope**

No source change was made to the audit checkout, no upstream `logos-blockchain` PR was opened, and no new timing harness or builder benchmark was added in this iteration. The PoQ and PoSel circuits, trusted setup, `lb_groth16`, `lb_poq`, `ark-*`, `rpds`, transaction-pool policy outside the duplicate test, and the ZkSignature circuit are assumed correct. #124's gas re-derivation is not repeated here.

**Assumptions**

The target commit and the specification commit in the header are the sources of truth. The host is not compromised. The timing observations from #125 and #98 are used as prior measurements rather than re-measured here. Reordering pure rejection checks does not change acceptance or ledger state; it can change only the error returned for an operation that is rejected by more than one check.

## 3. Method

- Claimed checklist issue `#266` after reading the two mandatory core specifications; re-read parent `#8`, repo-context issue `#19`, and the related issues `#98`, `#124`, and `#125`.
- Confirmed the audit source of truth once: `/home/pluto/Code/logos/internal-audit/logos-blockchain` at `a805329f8a186eb6989f09a7c49dee4a0e07473b`, clean working tree; `/home/pluto/Code/logos/internal-audit/logos-lips` at `7244d3b05ddec91a4a7b565bd5a9340ab77ededd`, clean working tree.
- Read the specifications listed in the header in full, including `blend-protocol.md` §Active Message and Rewarding, `bedrock-service-declaration-protocol.md` §Active, and the service-reward distribution timing.
- Manually traced `update_active` → `TargetEpochState::verify_proof` → `ActivityProof::verify_and_build` → `TargetEpochTracker::insert`, then traced the PoQ/ZkSignature boundary through `SDPActiveOp::verify`, `try_apply_contents`, `DeferredZkpVerifications::verify`, and `chain-leader::propose_block`.
- Inventoried every `ProofsVerifier` test implementation in the scoped crates and compared the current source against the Appendix B patch recorded in PR #264.
- Reviewed upstream PR #3545 against the same pinned target, including its cheap-check reorder, `verify_selection_and_evaluate` / `verify_quota` split, `ScriptedProofsVerifier` consolidation, stronger call-counting tests, and the review discussion about keeping the split API private until batching requires it. Its closure was recorded as a process/ownership decision, not a technical rejection.
- Recorded PR #3545's reported validation: 166 ledger tests, 43 blend-message tests, clippy clean, and fmt clean. These are upstream PR results, not additional tests independently run in this iteration.
- Automated validation, toolchain `rustc 1.98.1` / `cargo 1.98.1`: `cargo test -p logos-blockchain-ledger --lib mantle::sdp::rewards::blend` — 15 passed; `cargo test -p logos-blockchain-blend-message --lib reward` — 7 passed.
- Dynamic testing: none. The current-head tests exercise the pre-fix order; the call-counting tests in PR #264 were run against this same target commit in the prior report.

## 4. Previously identified finding re-verified

| Canonical ID | Tracker issue | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|---|
| `125-LB-001` | [#562](https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/562) | `The Groth16 pairing still runs before the duplicate, proof-of-selection and activity-threshold checks in SDPActive execution` | Denial of Service | Low | Low | Open; reverified |

### Re-verification — #562 (125-LB-001)

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `ledger/src/mantle/sdp/rewards/blend/mod.rs:L95-L107` (`Rewards::update_active`); `ledger/src/mantle/sdp/rewards/blend/target_epoch.rs:L103-L122,L157-L163`; `blend/message/src/reward/activity.rs:L41-L65` |
| Status | Open; reverified at `a805329f8`; the implementation evidence in [PR #3545](https://github.com/logos-blockchain/logos-blockchain/pull/3545) was closed unmerged and is not present in the target |

**Description**

The upstream target still calls `target_epoch_state.verify_proof` at `rewards/blend/mod.rs:95-100` and only then calls `target_epoch_tracker.insert` at `:102-107`. `TargetEpochState::verify_proof` first calls `ActivityProof::verify_and_build` at `target_epoch.rs:103-109`. That function runs `verify_proof_of_quota` — the Groth16 pairing — at `blend/message/src/reward/activity.rs:50-51`, then the cheaper proof-of-selection check at `:52-59`. The activity-threshold evaluation follows at `target_epoch.rs:117-122`, and the duplicate-per-provider check remains last at `target_epoch.rs:159-164`.

The ordering is not a consensus divergence: every check is deterministic and a failed operation is discarded. It is nevertheless an avoidable cost asymmetry. A genuine provider can submit a second message for the same target epoch, and a mis-selected or over-threshold activity proof can be rejected without a pairing. The current path pays the pairing first. The prior #125 report measured the resulting builder cost as proportional to the number of pending candidates and showed that the Appendix B reorder removes those pairings from the cheap rejection paths.

**Exploit scenario**

A provider submits one accepted activity message followed by many messages for the same declaration and target epoch, using increasing operation nonces and fresh PoQ proofs. During proposal construction, `chain-leader/src/lib.rs:673-727` collects the pending pool and repeatedly trial-applies candidates. The first message records the provider; each later message reaches the current PoQ pairing before `TargetEpochTracker::insert` rejects it as a duplicate. The same candidates can be revisited in the retry round because the first message made progress. This consumes builder CPU without changing ledger state. The broader unauthenticated mempool amplification is tracked by the earlier #98 report; this re-verification concerns the ordering defect that keeps that amplification expensive.

**Recommendation**

- *Short term*: use the implementation and tests in [PR #3545](https://github.com/logos-blockchain/logos-blockchain/pull/3545) as the primary evidence for the cheap-check order: check the target-epoch duplicate map before proof verification; verify proof of selection from the unverified PoQ nullifier; evaluate the activity threshold from a byte-identical unverified token view; then verify the PoQ. Keep the duplicate check inside `TargetEpochTracker::insert` as an invariant backstop.
- *Upstream shape*: coordinate PR #3545's `verify_selection_and_evaluate` / `verify_quota` split directly with #124's deferred-PoQ batching work, including the review question of whether the split API should remain private until batching needs it.
- *Regression*: add the builder-level test in S-003. It must make the operation nonce increase so later candidates pass the outer `SDPActiveOp` nonce check, and it must account for the builder's second retry pass. The assertion is still linear work and exactly one PoQ verification, not zero: the first activity message is valid and must be checked.

**References**: `blend-protocol.md` §Active Message; `bedrock-service-declaration-protocol.md` §Active; canonical finding [#562](https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/562); implementation and review evidence [PR #3545](https://github.com/logos-blockchain/logos-blockchain/pull/3545); historical prototype [PR #264](https://github.com/logos-blockchain/logos-blockchain-agent-message-board/pull/264); issue `#124`.

## 5. Suggestions (non-security)

### S-001 · Split the activity proof API at the deferred-PoQ boundary

The current `ActivityProof::verify_and_build` combines proof-of-selection verification, construction of a verified `BlendingToken`, and PoQ verification. PR #3545 implemented the corresponding `verify_selection_and_evaluate` / `verify_quota` split on the same target revision while retaining `verify_and_build` as their composition. Its review discussion considered keeping the split API private until batching requires it. The #124 design needs the ledger to perform the first two cheap decisions and return a PoQ proof plus its public inputs to `core::mantle::batch::DeferredZkpVerification`; the eventual upstream implementation should coordinate these boundaries directly.

The current `ProofsVerifier` trait at `blend/message/src/encap/mod.rs:10-35` exposes both verification methods, while `RealProofsVerifier` stores the PoQ public inputs privately at `blend/message/src/crypto/proofs.rs:65-69`. A clean seam is:

1. A `verify_selection_and_evaluate` phase consumes the unverified activity proof, the provider index, the membership size, the token evaluation, and epoch randomness. It verifies PoSel using the PoQ's plain key-nullifier field and computes Hamming distance from the unverified token bytes. It returns the provider's `zk_id`, distance, and a typed PoQ verification request.
2. `DeferredZkpVerification` gains a PoQ variant carrying the PoQ circuit proof and `PoQVerifierInput`. `DeferredZkpVerifications` batches those requests with `lb_poq::batch_verify` after block execution. The target tracker stores only the provider, `zk_id`, and distance, so no verified PoQ is needed to update the candidate state.
3. If a synchronous path remains temporarily, a small wrapper can call the same two phases and construct `BlendingToken` only after PoQ succeeds. This keeps the fast checks and the eventual batch path identical.

This seam must preserve the transition-period public inputs used by `RealProofsVerifier`: the deferred request must carry the exact current/previous-epoch input selected for the operation, not merely the proof bytes. The proof-of-selection and threshold checks are safe before the PoQ because they read only the serialized proof fields; a failed later batch rejects the whole candidate update.

### S-002 · Consolidate test doubles without erasing scenario-specific behaviour

The audited source has three reward-local doubles in `ledger/src/mantle/sdp/rewards/blend/mod.rs:1072-1164`; three encap doubles in `blend/message/src/encap/tests.rs:29-109`; one configurable accept/reject double in `blend/network/src/core/tests/utils.rs:36-76`; one unconditional double in `blend/provers/src/crypto/test_utils.rs:78-105`; and three service doubles in `services/blend/src/test_utils/crypto.rs:88-128`, `services/blend/src/core/tests/utils.rs:544-578`, and `services/blend/src/core/backends/libp2p/tests/utils.rs:54-80`. The audited source therefore has eleven verifier implementations. PR #3545 consolidated the three ledger doubles and three blend-message doubles on its branch; the five network/prover/service doubles remain the follow-up tracked by #291.

PR #3545's `ScriptedProofsVerifier` provides the shared accept/reject modes and PoQ/PoSel call counters behind the existing `unsafe-test-functions` feature. It should not force every remaining wrapper into a boolean: `StaticFetchVerifier` models a finite number of successful PoSel layers, and `MockProofsVerifier` binds dummy proofs to an epoch. Those behaviours can remain thin wrappers around the shared counter/failure policy when the replacement implementation lands.

### S-003 · Add a builder-level duplicate benchmark/regression test

`services/chain/chain-leader/src/lib.rs:632-759` is the relevant boundary: it collects all candidates, trial-applies each transaction against a cloned ledger state, verifies deferred proofs for each successful trial, and retries `still_pending` while a pass makes progress. PR #3545 adds stronger ledger call-count tests but does not provide this builder-level retry regression. A focused test or extracted proposal-selection helper should:

- construct `N` `SDPActive` candidates for one declaration and target epoch with monotonically increasing nonces, so they reach the service-specific duplicate check;
- place the first candidate first, let it be accepted, and leave the remaining candidates as duplicates;
- run the candidate loop with a call-counting PoQ verifier;
- assert that the first candidate causes one PoQ verification and all duplicate candidates, including the second retry pass, cause zero additional PoQ verifications; and
- assert that the duplicate work remains linear in `N` (the current retry loop may inspect the duplicates twice) and that only the first transaction is selected.

This complements, rather than replaces, the ledger unit tests from PR #264: those tests establish the local ordering, while this test proves the block builder does not reintroduce pairing work through its retry loop.

## Ruled out

- **Rebase drift:** the audit checkout is exactly `a805329f8`, the same full node commit named by #125, PR #264, and PR #3545. The current six target files retain the pre-patch contents; PR #3545's implementation branch was not merged, so the target remains unfixed.
- **State mutation from the reorder:** `TargetEpochState::verify_proof` borrows state and `TargetEpochTracker::insert` creates persistent state only after successful verification. Evaluating the unverified token before PoQ cannot admit it; a failed deferred PoQ rejects the candidate update.
- **PoSel dependence on a verified PoQ:** `ProofOfQuota::key_nullifier()` is a plain field accessor, and `ProofOfSelection::verify` uses that field plus its own selection randomness. The key-nullifier relation is exactly what PoSel checks; the Groth16 pairing is not needed for this check.
- **Threshold-byte drift:** the `UnverifiedTokenRef` used by PR #3545 must serialize the same three fields in the same order as `BlendingToken`; its byte-equality test preserves the guard first recorded in PR #264. Without that test, the pre-pairing threshold optimization should not be accepted.
- **Consensus determinism:** the specs require duplicate rejection but do not prescribe check order. All nodes still make the same accept/reject decision; only rejected-error precedence changes.

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
