# Blend release wait and accepted-item lag across epoch rotations

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/611`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `bfcae04d25218d878eb77c3c072cb0e88524de82` — component(s): `services/blend/src/core`, `services/key-management-system`, `zk/proofs/poq`
Specs: `https://github.com/logos-co/logos-lips` @ `75d3d0382604d4a0d8e246c268935dd386ffc8ea` — read: `blend-protocol.md` (in full; relevant sections: Releasing, Delaying, Failure Detection and Reaction, Proof of Quota)
Date: `2026-10-05` — author: `Codex` — status: `draft`

---

## 1. Summary

- Overall assessment: a three-node local network crossed two epoch rotations and measured release waits and KMS service-loop queueing, but the transaction burst remained in local PoW and did not reach the real gossip path.
- Findings: no new finding; the existing #576 classifications are carried forward and were not independently re-verified or reclassified.
- Key themes: no release wait exceeded one second in the measured windows; the tested burst did not produce an observed `Missed {n}` event, but did not reach gossip and therefore cannot establish a safe burst threshold.
- Must-fix before launch: no new disposition in this follow-up; see the existing open LB-001 classification in report #576.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/blend/src/core/mod.rs` | `handle_release_round` and its awaited cover-message path. |
| `services/key-management-system/src/lib.rs` | KMS service receive/execute loop. |
| `zk/proofs/poq/benches/prove.rs` | Existing in-process core-node PoQ proving benchmark, retained only as a lower-level timing baseline. |
| `nodes/node/binary/src/config/deployment/settings.yaml` | Confirmed the requested deployment values at the pinned target: 1 Blend layer, message frequency 1.0 per round, maximum release delay 1 round. |
| Scratch-only instrumentation in `services/blend/src/core/mod.rs` and `services/key-management-system` | Timed cover-generation awaits and KMS `Execute` queue/execution intervals; temporary edits were confined to `/tmp/logos-blockchain-611` and removed after the run. |

**Out of scope**

This was a standalone local three-node network, not a public or geographically distributed devnet. The test shortened chain epoch parameters to observe rotations promptly. A follow-up burst attempt also lowered only the scratch configuration's Blend PoW base difficulty to 1; it did not change the requested Blend settings, but the burst still failed to reach message publication. These measurements do not characterize production-scale load, and do not re-run the `#576` mempool/RocksDB stall probe. Third-party implementations including Groth16, libp2p, and RocksDB are assumed correct.

**Assumptions**

The source report #576 remains the canonical record for LB-001 through LB-005. Their classifications below are carried forward unchanged, not independently re-verified here. The pinned LIPS specification is assumed authoritative for this review.

## 3. Method

- Reviewed the exact target commit `bfcae04d25218d878eb77c3c072cb0e88524de82` and LIPS commit `75d3d0382604d4a0d8e246c268935dd386ffc8ea` in a detached source worktree at `/tmp/logos-blockchain-611`.
- Inspected `handle_release_round`, `generate_and_try_to_decapsulate_cover_message`, the KMS service loop, and the requested deployment settings. The release-round function awaits cover encapsulation before joining the futures that send that round's messages; the KMS loop awaits each operator before receiving the next request.
- Read the exact pinned `blend-protocol.md` at LIPS commit `75d3d0382604d4a0d8e246c268935dd386ffc8ea`, including the sections named above. The existing report #576 discusses the specification gap around precomputing core-quota proofs; this follow-up makes no new specification finding.
- Dynamic testing: ran `.agents/cucumber_scripts/e2e-integration-test.sh test_mantle_sdp_blend --features mantle_sdp blend_rotation_cover_wait_experiment -- --exact --nocapture` through the repo-aware host-like wrapper. The standalone three-node local network ran on Linux/x86_64 (AMD Ryzen 9 9950X), with the requested Blend settings (`num_blend_layers=1`, `message_frequency_per_round=1.0`, `maximum_release_delay_in_rounds=1`). Chain slot duration was 1 second and epoch parameters were shortened for the run. The unmodified-PoW run observed two rotations on all three nodes. Across the first two 10-second post-rotation windows, cover-await durations (ms) were: epoch 1 — node 0 `[356, 0]`, node 1 `[0, 0]`, node 2 `[183, 0]`; epoch 2 — node 0 `[169, 0]`, node 1 `[0, 0]`, node 2 `[183, 0]`. None exceeded 1 second. No `Missed {n} transactions` event or inbound-channel lag/drop was observed in those windows.
- In the same rotation windows, KMS `Execute` instrumentation counted queued operator type, queue wait, execution time, and queued depth at dequeue. Per node, epoch 1/epoch 2 `Execute` counts were node 0 `2037/2063`, node 1 `1682/1474`, node 2 `1703/1145`; maximum queue depth was 2–3. `CheckConditionWithLeaderKey` (per-slot lottery) had maximum queue waits of `356/340 ms`, `341/366 ms`, and `364/357 ms` respectively. `PoQOperator` maximum queue wait/execution durations were node 0 `185/185` and `169/171 ms`, node 1 `173/173` and `188/187 ms`, node 2 `183/183` in both windows. `BuildWithLeaderKey` and `UnsafeVoucherOperator` also appeared. Queue wait is reported separately from operator execution; leadership PoQ itself is not a KMS operation at this revision.
- Burst attempt: submitted 100 distinct structurally valid transfers to the real Blend transaction ingress during the rotation run; all 100 were accepted with no HTTP-level rejection in 78.9 ms. Immediately afterward, 98 remained pending local PoW. A second scratch run lowered only Blend PoW base difficulty to 1 and kept the requested Blend settings; all 100 were again accepted, but node logs still contained zero completed local transaction PoW/encapsulation events, zero outgoing data-message publications, and zero remote transaction decapsulations before the network crossed a third rotation. The run's explicit assertion that at least one transaction arrive at another core through Blend/libp2p therefore failed. No missed/drop event was observed, but because no burst item reached gossip, this is not evidence that the requested real-pubsub burst is safe and does not determine a minimum triggering burst.
- The one-sample in-process `logos-blockchain-poq` benchmark completed in `105.4 ms` on the same host. It remains only a lower-level baseline and is not substituted for the network observations above.
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

The measured waits and KMS queues are observations from this local accelerated network, not production-scale estimates. The burst result is inconclusive for accepted-item lag: endpoint acceptance was measured, but no transaction completed the local PoW/encapsulation stage or reached libp2p publication. The data therefore neither confirms nor contradict a burst-triggered `Missed {n}` condition.

## 5. Follow-up and issue disposition

The rotation wait and KMS queue observations are now recorded, but the burst still needs to traverse local PoW, real libp2p publication, and remote decapsulation during a cover wait before accepted-item lag can be assessed. A subsequent run should arrange a completed, observable transaction burst (or an explicitly documented supported way to hold KMS work busy) without treating ingress acceptance as gossip delivery. Issue #611 remains open and assigned; this report PR is not intended to close it. No new LB identifier or classification is proposed pending that follow-up.

---

Draft pending independent review and explicit approval.
