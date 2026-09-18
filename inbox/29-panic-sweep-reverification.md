# Audit Report — Workspace panic-surface re-verification

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/29`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): workspace-wide panic paths, node panic policy, HTTP/storage/wallet paths, Blend membership transition
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`
Date: `2026-09-18` — author: `codex` — status: `final`

---

## 1. Summary

- Overall assessment: the #29 sweep has already been completed twice at the exact available node revision; this pass re-verifies the recorded anchors and found no distinct panic finding.
- Findings: `0 new findings; existing #126/#206 findings and follow-ups remain applicable.`
- Key themes: `fail-stop panic policy`, `input-facing expect sites`, `fixed-capacity structures`, `proof-verification FFI boundary`
- Must-fix before launch: the existing finding issues remain the canonical records; #127 is still the focused unresolved proof-verification boundary.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `nodes/node/binary/src/{panic.rs,lib.rs}` | Process-wide panic hook and installation. |
| `nodes/api-common/src/pprof.rs` | Profiling query parameters and the existing profiling-build finding. |
| `blend/crypto/src/merkle.rs`, `ledger/src/mantle/sdp/rewards/blend/current_epoch.rs` | Fixed Merkle capacity and epoch-transition `expect`. |
| `services/wallet/src/lib.rs`, `wallet/src/lib.rs` | Stored event fallback and leader-claim event assumption. |
| Workspace panic inventory | Existing non-test `unwrap`/`expect`/`panic!`/`unreachable!` sites and the spawned-task boundary, with prior reports as the baseline. |

**Out of scope**

Malformed proof behavior inside `rust-rapidsnark`, `ark-groth16`, and Poseidon2/Jellyfish verification remains issue #127. The full network, storage, codec, HTTP-authentication, and FFI sweeps already covered by the prior #29 reports were not repeated line by line. Third-party runtimes and storage libraries were assumed correct.

**Assumptions**

The two core Logos specifications are the reference, and the workspace’s process-wide `exit(1)` panic hook remains intentional fail-stop behavior. The issue’s older `19353c619` hint is not present in the clean audit checkout; `a805329f…` is the exact revision inspected.

## 3. Method

- Read issue `#29`, parent `#24`, repo-context issue `#19`, and the two prior #29 reports: `processed/29-panics-reachable-from-input.md` and `processed/29-panic-sweep-second-pass.md`.
- Read both core overview specifications in full before source review.
- Re-read the prior report’s material anchors at `a805329f…`: `set_hook`/`exit(1)`, profiling frequency input, the 2^20-leaf Blend Merkle limit, wallet event loading and `LeaderClaim` conversion, and detached subscription/retry task sites.
- Re-ran targeted searches over the node checkout for panic sites and the named anchors. The prior report’s findings are still present at the same revision; no newer source revision was available in the clean shared audit checkout.
- No source changes or dynamic tests were needed for this re-verification. The previous reports’ finding issues are canonical: #491–#496, with #127 still open for the proof-verification FFI boundary.

## 4. Findings

No new `LB-NNN` finding is opened. The process-wide panic hook, profiling query panic, fixed-capacity Blend tree, wallet/storage `expect` paths, and detached-task observations are already documented in the two prior reports for #29. Reusing those findings avoids duplicate identifiers for the same defects.

### Existing finding status

- The fail-stop panic hook remains at `nodes/node/binary/src/lib.rs:228` and `nodes/node/binary/src/panic.rs:10-16`; it still amplifies any reachable panic to a whole-node exit.
- `nodes/api-common/src/pprof.rs:88-91` still passes an unvalidated optional `i32` frequency to `pprof::ProfilerGuard::new`; the `frequency=0` profiling-build case remains the previously reported finding.
- `blend/crypto/src/merkle.rs:13,94-95` still bounds the tree at `1 << CORE_MERKLE_TREE_HEIGHT`, while `ledger/src/mantle/sdp/rewards/blend/current_epoch.rs:197` still `expect`s construction; this remains the previously reported chain-state capacity finding.
- `services/wallet/src/lib.rs:1555-1570` still converts storage-event failures to an empty set, and `wallet/src/lib.rs:600` still expects a leader-claim event; this remains the previously reported storage/error-propagation finding.
- The proof-verification FFI boundary has not been resolved by this pass. It remains the explicit scope of #127 and should not be given a duplicate finding ID from #29.

The prior #29 reports’ recommendations and classifications remain the applicable record. No evidence was found that would justify reopening, reclassifying, or duplicating them.

## 5. Suggestions

### S-001 · Complete the proof-verification boundary review under #127

Trace malformed proof bytes through the fixed-size deserializers, canonical field-element checks, `ark-groth16`/Poseidon2 verification, and the `rust-rapidsnark` FFI. Confirm whether every failure returns `false`/`Err` rather than panicking or aborting. Any validated result should be reported under #127, not as a new #29 finding.

### S-002 · Keep the prior #29 report issues as the source of truth

The prior reports were audited at the same node revision and their findings are already filed as #491–#496. This pass should close or unassign #29 only after the workflow owner confirms that the #127 follow-up is the sole remaining line; no duplicate #29 report finding should be created.

