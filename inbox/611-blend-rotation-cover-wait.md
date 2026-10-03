# Blend release wait and accepted-item lag across epoch rotations

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/611`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `bfcae04d25218d878eb77c3c072cb0e88524de82` — component(s): `services/blend/src/core`, `services/key-management-system`, `zk/proofs/poq`
Specs: `https://github.com/logos-co/logos-lips` @ `75d3d0382604d4a0d8e246c268935dd386ffc8ea` — read: `blend-protocol.md` (in full; relevant sections: Releasing, Delaying, Failure Detection and Reaction, Proof of Quota)
Date: `2026-10-03` — author: `Codex` — status: `draft`

---

## 1. Summary

- Overall assessment: the pinned source still awaits cover-message encapsulation inside the release round, and the KMS worker handles requests serially; a one-sample in-process PoQ benchmark completed, but none of the requested epoch-rotation devnet measurements were performed.
- Findings: no new finding; no existing finding was independently re-verified or reclassified.
- Key themes: the release wait, accepted-item lag, and KMS queue depth remain unmeasured across epoch rotations on a devnet.
- Must-fix before launch: no new disposition in this follow-up; see the existing open LB-001 classification in report #576.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/blend/src/core/mod.rs` | `handle_release_round` and its awaited cover-message path. |
| `services/key-management-system/src/lib.rs` | KMS service receive/execute loop. |
| `zk/proofs/poq/benches/prove.rs` | Existing in-process core-node PoQ proving benchmark, used only as a lower-level timing baseline. |
| `nodes/node/binary/src/config/deployment/settings.yaml` | Confirmed the requested deployment values at the pinned target: 1 Blend layer, message frequency 1.0 per round, maximum release delay 1 round. |

**Out of scope**

No three-node devnet was started. This report therefore does not assess actual epoch rotation behavior, cover-wait distributions in `handle_release_round`, accepted-item lag from real libp2p gossip, or KMS queue depth following rotation. It also does not re-run the `#576` mempool/RocksDB stall probe or assess third-party implementations such as Groth16, libp2p, or RocksDB.

**Assumptions**

The source report #576 remains the canonical record for LB-001 through LB-005. Their classifications below are carried forward unchanged, not independently re-verified here. The pinned LIPS specification is assumed authoritative for this review.

## 3. Method

- Reviewed the exact target commit `bfcae04d25218d878eb77c3c072cb0e88524de82` in a detached, clean worktree at `/tmp/logos-blockchain-611`. The benchmark wrote build artifacts to `/tmp/target-794`; it did not modify the source worktree.
- Inspected `handle_release_round`, `generate_and_try_to_decapsulate_cover_message`, the KMS service loop, and the requested deployment settings. The release-round function awaits cover encapsulation before it joins the futures that send that round's messages; the KMS loop awaits each operator before receiving the next request.
- Read the exact pinned `blend-protocol.md` at LIPS commit `75d3d0382604d4a0d8e246c268935dd386ffc8ea`, including the sections named above. The existing report #576 discusses the specification gap around precomputing core-quota proofs; this follow-up makes no new specification finding.
- Dynamic testing: ran the existing `logos-blockchain-poq` core-node prove benchmark in-process, at the pinned source revision, with one sample and one iteration. It completed in `105.4 ms` on an AMD Ryzen 9 9950X. This is a single lower-level PoQ sample, not a measurement of cover-message wait, KMS queueing, or epoch rotation; it is not statistically representative.
- No devnet, cluster-spawning test, transaction burst, or KMS queue instrumentation was run. Consequently, this iteration has no observation for the two-rotation distribution, the frequency of waits over one second, correlation with `Missed {n}` or dropped-message metrics, minimum burst that causes accepted-item lag, or the second core-quota proof's wait behind rotation work.
- The comment on issue #611 reports that the prior #576 second-pass devnet run at `c4c86be1` had no epoch rotation. Its wait figures are historical context only and are not treated as evidence for this requested measurement.

## 4. Findings

This follow-up adds no canonical finding and does not claim independent re-verification. The existing findings from report #576 remain recorded as follows:

| ID | Category | Severity | Difficulty | Status in report #576 |
|---|---|---|---|---|
| LB-001 | Privacy / Anonymity | Medium | High | Open |
| LB-002 | Denial of Service | Low | High | Open |
| LB-003 | Timing | Low | High | Open |
| LB-004 | Auditing and Logging | Informational | — | Open |
| LB-005 | Denial of Service | Low | Medium | Open |

The 105.4 ms benchmark result is only a new local baseline for one core-node PoQ operation. It cannot establish whether the cover-message wait after an epoch rotation exceeds one second, or whether that wait is the dominant contributor to a KMS queue or accepted-item lag on a devnet.

## 5. Follow-up and issue disposition

The requested dynamic work remains open: run the specified three-node devnet across two epoch rotations, collect the per-cover wait and lag/`Missed {n}` observations, inject a burst of structurally valid transactions during a wait, and measure KMS queue depth and the second core-quota proof wait. Issue #611 remains open and assigned; this report PR is not intended to close it. No new LB identifier or classification is proposed pending those measurements.

---

Draft pending independent review and explicit approval.
