# Audit Report — Blend delivery observation after validation and lag

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/571`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `54328b50eb47ab4b8fd8044f8db5fbcebed718c0` — component(s): `services/blend` delivery observation, `services/chain/chain-network` proposal observation, `services/tx-service` accepted-item observation
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `blend-protocol.md`, `bedrock-v1.1-block-construction.md`
Date: `2026-09-18` — author: `codex` — status: `final`

---

## 1. Summary

- Overall assessment: the #571 checklist re-verifies the delivery-observation behavior reported in #276; no distinct finding should be opened.
- Findings: `0 new findings; #276 LB-002 and LB-003 re-verified.`
- Key themes: `terminal observation loss`, `cross-type fallback fate`, `pre-validation delivery evidence`
- Must-fix before launch: the existing #276 findings remain open; independently review and process them rather than creating duplicate identifiers for #571.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/chain/chain-network/src/lib.rs` | `note_received_proposal`, its position in the proposal event loop, and the 64-entry observation channel. |
| `services/chain/chain-network/src/network/adapters/libp2p.rs` | Proposal decoding, topic filtering, and lag handling. |
| `services/tx-service/src/tx/service.rs` | The 64-entry accepted-item channel and its notification points. |
| `services/blend/src/core/dispatcher/libp2p.rs` | Proposal/transaction observation, lag termination, and the merged stream. |
| `services/blend/src/delivery/failure_detection.rs`, `services/blend/src/delivery/mod.rs` | Detector state, expiry, terminal-stream behavior, and direct broadcast. |
| `services/blend/src/core/delivery.rs`, `services/blend/src/edge/mod.rs`, `services/blend/src/core/mod.rs` | Core/edge wrappers and the service-loop branches that consume the detector. |

**Out of scope**

The Blend cryptographic constructions, scheduler correctness, epoch-transition state machine, gossipsub implementation internals, and mempool eviction policy were not re-audited. Third-party crates assumed correct: `tokio`, `tokio-stream`, `futures`, and `libp2p-gossipsub`.

**Assumptions**

The pinned `logos-lips` specifications are the reference. The failure detector is intended to implement `blend-protocol.md` § Failure Detection and Reaction: a sender waits `T_M` for its payload on the broadcasting channel and directly broadcasts it if the payload is not observed. A missing observation caused by a lag is not evidence of a failed delivery.

## 3. Method

- Read issue `#571`, parent `#12`, repo-context issue `#19`, and the existing report for #276.
- Read `bedrock-architecture-overview.md` and `overview-cryptoeconomics.md` in full before source review.
- Read `blend-protocol.md` and `bedrock-v1.1-block-construction.md` in full. The relevant sections were Blend § Failure Detection and Reaction, § Detection, § Direct Broadcast, § Relaying, § Processing, and § Releasing; block-construction § Block Proposal Reconstruction and § Block Proposal Validation.
- Issue #571 was spawned from #145 / PR #570, whose audit target is `54328b50…`; the source paths were therefore re-read at that exact revision in an isolated checkout rather than using the newer shared checkout. A focused comparison of `54328b50…` through `a805329f…` found four intervening commits, none of which modifies the audited Blend delivery-observation paths. A focused diff of the exact target against the #276 target `3d5d419ec85c7b23e4d7e1455bd1cde845264a9e` also showed no relevant changes.
- Traced the full observation path: network proposal/transaction input → chain-network or mempool broadcast channel → Blend dispatcher → merged stream → `FailureDetector` → direct-broadcast branch.
- Ran the same command from the exact `54328b50…` checkout: `cargo +nightly-2026-07-05 test -p logos-blockchain-blend-service delivery::failure_detection::tests:: --lib --target-dir /tmp/logos-audit-571-54328-target`; all 10 existing failure-detector unit tests passed. No devnet or throughput measurement was run.

## 4. Findings

No new `LB-NNN` finding is opened. The two material behaviors below are the same root causes already reported in **#276 LB-002** and **#276 LB-003** in [processed/276-blend-delivery-observation-lag.md](../processed/276-blend-delivery-observation-lag.md). Reusing those findings avoids splitting one defect across duplicate identifiers. Their preserved classifications are:

- **#276 LB-002** — Low / Low / Denial of Service.
- **#276 LB-003** — Low / Low / Denial of Service.

### Re-verification of #276 LB-002 — cross-type observation streams share terminal fate

`services/blend/src/core/dispatcher/libp2p.rs:290-306` creates proposal and accepted-transaction streams, appends a `None` end marker to each, selects between them, and stops the merged stream at the first end marker. Both individual streams end when `stop_observing_on_lag` at `:101-119` sees `BroadcastStreamRecvError::Lagged`.

The detector receives that merged stream at `services/blend/src/delivery/failure_detection.rs:29-55`. When the stream ends, `poll_next` returns `Poll::Ready(None)` at `:151-157`. The core and edge loops consume the detector only through `Some(undelivered) = next_undelivered_messages(...)` (`services/blend/src/core/mod.rs:1094`, `:1188`; `services/blend/src/edge/mod.rs:399`), so the direct-broadcast path no longer produces expiry events. A transaction observation gap therefore also terminates proposal observation, even though `DataPayload::BlockProposal` and `DataPayload::Transaction` remain distinct keys (`services/blend/src/delivery/failure_detection.rs:35`; test at `:271-284`).

The existing unit test `losing_sight_of_the_broadcasting_channel_stops_the_detection` confirms the terminal behavior. This re-verification adds no new impact or trust boundary beyond #276 LB-002: the transaction stream is still the cheaper unrelated stream to lag, and the proposal fallback remains disabled for the process lifetime after the shared stream ends.

### Re-verification of #276 LB-003 — terminal detector retains later releases

After the observation stream ends, `FailureDetector::poll_next` never reaches `take_expired_payloads` again. The service loops nevertheless retain the detector and continue calling `mark_payload_as_blended` when locally generated messages are released (`services/blend/src/edge/mod.rs:430-445`; `services/blend/src/core/delivery.rs:68-71`). That method inserts each distinct payload into `unacknowledged_blended_payloads` (`services/blend/src/delivery/failure_detection.rs:59-71`), with no bound or cleanup path after the observer has ended. The detector also logs the terminal error each time it is polled because the fused stream remains at `Ready(None)` (`:147-158`).

This is the same terminal-state retention and repeated-log behavior as #276 LB-003. The ten passing unit tests cover normal expiry, delivery, and the intentional terminal behavior, but do not exercise a lagged proposal or transaction stream followed by later releases.

### Checklist conclusions

- Moving `note_received_proposal` after `should_process_block` and `verify_header_alone` is not a separate finding. `note_received_proposal` at `services/chain/chain-network/src/lib.rs:407-414,798-801` records that the proposal appeared on the broadcasting channel, before local consensus processing. `AlreadyApplied` is still delivery evidence, and a local reconstruction or state failure does not mean the sender’s payload failed to traverse Blend. Moving the notification would make the observation depend on validator-local state and could suppress valid delivery evidence. Add an explicit regression test for this contract before changing the order.
- The observer currently carries full `Proposal` and transaction values through bounded broadcast channels. Replacing these with a typed delivery key (`HeaderId`/proposal bytes and transaction hash/bytes as appropriate) would reduce clone and queue memory, but the key must preserve the `DataPayload` variant so equal proposal and transaction bytes cannot acknowledge one another. This is a hardening recommendation, not a new defect.
- On lag, the safest recovery is to abandon all deadlines that could have crossed the observation gap, keep the detector armed for payloads released afterward, and mark the detector degraded. Resubscribing or continuing without an explicit gap would risk false direct broadcasts; permanently ending detection is the existing #276 liveness tradeoff.
- The accepted-item rate and devnet stall timing were not measured here. The static trigger remains a 64-entry channel overrun; a separate devnet experiment should measure the actual mempool acceptance rate and Blend-loop stall before choosing capacities.

## 5. Suggestions

### S-001 · Make observation gaps typed and recoverable

Replace the terminal `take_while` behavior with an explicit `Observation::Gap(StreamType, count)` event. Drop or abandon only deadlines whose evidence could have crossed that gap, keep the other payload type under observation, and resume after the receiver’s post-`Lagged` cursor. Expose a one-shot metric and degraded state so operators can distinguish “no payload arrived” from “the observer was blind.”

### S-002 · Add integration coverage for the validation boundary

Test that a proposal is observable when it is already applied, fails local reconstruction, or fails header validation, and document which cases count as delivery evidence. This prevents a future move of `note_received_proposal` from silently changing the failure-detector contract.

### S-003 · Bound detector memory after terminal failure

If the implementation keeps a terminal state, stop accepting new entries into `unacknowledged_blended_payloads` and emit one diagnostic. If it resumes, explicitly abandon the entries that overlap the gap before rearming expiry.
