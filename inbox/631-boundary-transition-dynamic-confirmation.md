# Audit Report — Re-verification of boundary-transition recomputation

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/631`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `9ffddb30b9e6cf79465802953caedd010ff1cecd` — component(s): `ledger`, `services/chain/chain-service`, `services/chain/chain-network`
Specs: `https://github.com/logos-co/logos-lips` @ `75d3d0382604d4a0d8e246c268935dd386ffc8ea` — read: `cryptarchia-v1-protocol.md`, `bedrock-service-reward-distribution.md`, `fork-choice.md`
Date: `2026-09-20` — author: `Codex` — status: `draft`

---

## 1. Summary

- Overall assessment: Static re-verification supports the existing `178-LB-001` finding: a boundary-transition update is recomputed for each distinct valid block and no `(parent, new epoch)` transition cache is present in the reviewed path.
- Findings: `H` high · `M` medium · `D` denial of service
- Key themes: `unbounded consensus-state recomputation`, `proof reuse across distinct headers`, `missing transition-result cache`
- Must-fix before launch: preserve the existing `178-LB-001` remediation priority and complete the dynamic confirmation requested by issue #631 before treating the finding as closed.

This iteration does not introduce a new independent finding or reclassify the canonical record. It re-verifies `178-LB-001` as tracked by issue #708, retaining its existing `High` severity, `Medium` difficulty, and `Denial of Service` category. The local multi-node reproduction, packet/path observation, and prototype cache requested by issue #631 were not run in this iteration; no dynamic result is represented as completed evidence below.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `ledger/src/lib.rs` | Parent-state preparation, header application, proof verification, reward UTXO insertion. |
| `ledger/src/mantle/mod.rs` | Mantle header application and service-distribution processing. |
| `ledger/src/mantle/sdp/rewards/blend/current_epoch.rs` | Provider declaration traversal, Merkle-root construction, and provider-index enumeration at an epoch transition. |
| `ledger/src/cryptarchia/mod.rs` | Proof verification state handling. |
| `services/chain/chain-service/src/lib.rs` | Block admission, ledger update invocation, consensus acceptance, and retained-state pruning. |
| `services/chain/chain-network/src/lib.rs` | Network proposal admission, older-than-LIB/already-applied filtering, reconstruction, orphan handling, and apply dispatch. |

**Out of scope**

The requested live local-network reproduction, packet capture, timing benchmark, memory measurement, and implementation of a `(parent id, new epoch)` cache were not performed. Consensus cryptography and third-party networking implementations were not independently re-audited. No circuit code was in scope.

**Assumptions**

The pinned Cryptarchia, reward-distribution, and fork-choice specifications are authoritative for this review. The existing report and canonical tracker record for `178-LB-001` are treated as prior evidence, not as a substitute for the source inspection in this iteration. The attacker is a network participant able to cause multiple distinct valid blocks for a slot/parent combination to be received by a node; the report does not assume control of honest leader keys.

## 3. Method

- Read issue `#631`, parent issue `#6`, the original issue `#178`, the prior report `processed/178-mantle-epoch-transition-cost.md`, and canonical tracker issue `#708`.
- Re-read the core Bedrock architecture and Cryptoeconomics specifications, then read the pinned `cryptarchia-v1-protocol.md`, `bedrock-service-reward-distribution.md`, and `fork-choice.md` at the revisions listed above.
- Manually inspected the pinned target revision with `git show`, `git grep`, and `git blame`, following the path from network admission to `ChainService::try_apply_block_with_state_retention`, `Ledger::prepare_update`, epoch-state construction, reward insertion, and consensus acceptance.
- Automated tooling: none.
- Dynamic testing: none in this iteration. The prior report's isolated measurements are cited only as prior evidence and were not repeated here.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| `178-LB-001` | Boundary-transition recomputation is repeated for distinct valid blocks | Denial of Service | High | Medium | Re-verified; canonical issue `#708` remains open |

### `178-LB-001` · Boundary-transition recomputation is repeated for distinct valid blocks

| | |
|---|---|
| Severity | High |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | `ledger/src/lib.rs:183-205,310-375` (`prepare_update`, `try_apply_header`); `ledger/src/mantle/sdp/rewards/blend/current_epoch.rs:148-215`; `services/chain/chain-service/src/lib.rs:425-501` (`try_apply_block_with_state_retention`) |
| Status | Open; canonical tracker issue `#708`; original finding `178-LB-001` |

**Description**

For a block whose parent is retained, `Ledger::prepare_update` obtains and clones the parent's ledger state and invokes the update path with the candidate block's slot, leader proof, uncle slots, and transactions (`ledger/src/lib.rs:183-205`). `try_apply_header` then applies the Cryptarchia header, the Mantle header, and service-distribution processing before inserting reward UTXOs (`ledger/src/lib.rs:310-375`).

At an epoch boundary, the Blend reward path derives the provider set and ZK root from declarations and constructs the provider Merkle structure (`ledger/src/mantle/sdp/rewards/blend/current_epoch.rs:148-215`). The inspected target revision contains no memoized transition result keyed by the parent state and new epoch. Consequently, two distinct candidate blocks that share the same eligible parent and transition slot enter the same expensive state-transition path independently.

The network admission path does not eliminate this multiplicity. Before proposal reconstruction it checks whether the block is older than the last irreversible block or already applied; the block-ID check cannot collapse distinct headers. The chain service separately rejects a future-slot block, then calls `prepare_update` and batch proof verification before consensus acceptance (`services/chain/chain-service/src/lib.rs:438-489`). The reviewed network path also dispatches reconstructed blocks to the normal apply path and has orphan handling for missing parents, but no transition-specific per-parent/epoch bound was found.

The proof path does not provide the needed deduplication invariant. Cryptarchia proof verification works from a cloned consensus state (`ledger/src/cryptarchia/mod.rs:555-570`), while the protocol specifies that proof of leadership is not directly bound to a particular block body. A valid leader can therefore produce distinct headers for the same eligible parent/slot; changing transactions or other block content also changes the block ID and defeats the already-applied filter. This is the same source-to-sink condition recorded by the existing `178-LB-001` report, not a newly discovered issue.

**Exploit scenario**

An eligible leader or a party able to relay multiple valid candidate blocks causes a node to receive distinct blocks sharing a parent immediately across an epoch boundary. Each block has a distinct ID and is not rejected by the already-applied check. For every candidate that reaches the chain-service apply path, the node can rebuild the boundary transition and its reward/provider structures from the parent state before consensus selection decides whether the block is canonical. Repeating this over the retained-parent window consumes CPU and retains intermediate state proportional to the number of distinct candidates. The exact throughput, memory impact, and whether the network layer materially limits delivery require the dynamic experiment requested by issue #631; this static pass does not claim those measurements.

**Recommendation**

- *Short term*: Add an explicit bound or admission policy for repeated boundary-transition candidates keyed by the eligible parent and transition slot/epoch, while preserving correctness for legitimately different blocks and reorgs. Instrument transition duration, queue depth, rejection counts, and retained-state growth.
- *Long term*: Cache or share the deterministic transition result for `(parent state, new epoch)` and make reward/provider computation and UTXO insertion reusable across candidate headers. Add tests proving that distinct valid blocks reuse the transition result, while a different parent, epoch input, or relevant state invalidates the cache. Complement this with a network-level resource limit so a valid-proof sender cannot enqueue unbounded equivalent work.

**References**: issue `#631`; original issue `#178`; canonical tracker issue `#708`; prior report `processed/178-mantle-epoch-transition-cost.md`; `cryptarchia-v1-protocol.md` sections on slots, proof of leadership, block validation, and chain maintenance; `bedrock-service-reward-distribution.md`; `fork-choice.md`.

## 5. Suggestions

### S-001 · Run the requested local-network benchmark

Use a short-epoch local network with controlled declarations and multiple distinct candidate blocks sharing the same eligible parent. Record per-node transition latency, CPU, memory, queue depth, block IDs, parent IDs, transition epochs, LIB movement, and whether candidates are dropped before or after ledger application. Keep the result separate from the canonical static re-verification until the exact network path is traced.

### S-002 · Confirm network-path and orphan behavior

Trace the same candidate identifiers through proposal admission, reconstruction, orphan download, and chain-service apply. Confirm whether gossipsub, peer scoring, per-peer quotas, or orphan limits impose a practical bound not visible in the reviewed chain-network functions. A concurrently passing scenario or aggregate block count should not be used as evidence for this path without matching parent, slot, block-ID, and node anchors.

### S-003 · Validate the cache invariant and proof reuse

Prototype the proposed `(parent id, new epoch)` cache in a test harness. Verify that the same transition result is reused for distinct valid headers with the same transition inputs, and that cache hits do not bypass block-specific validation, transaction effects, uncle validation, or consensus fork-choice bookkeeping. Test a same-epoch equivocation separately so the cache cannot accidentally suppress valid safety checks.

---

## Appendix A — Classification basis

The `High` / `Medium` / `Denial of Service` classification is preserved from canonical finding `178-LB-001` in issue `#708`. Under `docs/REPORT_TEMPLATE.md`, High covers a remote DoS requiring significant resources, timing, or a second weakness; Medium covers realistic high-cost or multi-peer degradation. This iteration did not obtain new dynamic evidence warranting a severity or difficulty change.

Draft pending independent review and explicit approval.
