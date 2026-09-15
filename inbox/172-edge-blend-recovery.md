# Audit Report — Blend edge state, delivery fallback, and mode transitions

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/172`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `services/blend/src/edge`, `services/blend/src/orchestrator`, `services/blend/src/delivery`, `services/blend/src/pending.rs`, `services/blend/src/lib.rs`, `services/api/src/http/blend.rs`, `services/tx-service`, `services/chain/chain-leader`, `services/chain/chain-network`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, and `blend-protocol.md` sections `Edge Network`, `Failure Detection and Reaction`, and `Transition Period`
Pinned dependency: `https://github.com/logos-co/Overwatch` @ `ae887f41f5a626c341179026ad7f03953ff2072e`
Date: `2026-09-14` — author: `Codex` — status: `final`

---

## 1. Summary

- Overall assessment: Edge mode has a real durability gap for accepted direct-Blend work and a separate mode-handover race that can suppress direct fallback; epoch-bound proposal discard is currently consistent with the stated protocol design but needs clarification.
- Findings: `0 C` critical · `0 H` high · `2 L` low · `0 I` informational
- Key themes: non-durable accepted work, explicit handover sequencing, and a protocol boundary around never-released epoch-bound proposals.
- Must-fix before launch: LB-003, if direct fallback is a production invariant, requires an explicit edge drain/handover acknowledgement before the old mode can be killed.
- Launch decisions: LB-001 requires a documented durability contract for “accepted for blending” and a recovery fix if that guarantee is intended; S-001 is not a launch blocker until the protocol team decides whether never-released epoch-bound proposals require a separate recovery rule.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/blend/src/edge/mod.rs` | Edge service state, pending transaction queue, epoch handling, and service shutdown. |
| `services/blend/src/edge/current_epoch.rs` | Epoch-local proposal queue and encapsulation order. |
| `services/blend/src/pending.rs` | Pending proposal and transaction queue ownership and drop behavior. |
| `services/blend/src/delivery/failure_detection.rs` and `services/blend/src/delivery/mod.rs` | In-memory release deadlines and direct-broadcast fallback. |
| `services/blend/src/orchestrator/{instance,on_demand}.rs` and `services/blend/src/lib.rs` | Mode handover, shutdown grace, and independent epoch subscriptions. |
| `services/api/src/http/blend.rs` and `nodes/node/binary/src/api/handlers.rs` | Direct Blend transaction acceptance and pending-transaction API contract. |
| `services/tx-service`, `services/chain/chain-leader`, and `services/chain/chain-network` | Whether another recoverable service resubmits direct Blend requests or provides eventual proposal dissemination/recovery. |

**Out of scope**

Core-specific recovery and retirement behavior already covered by issue `#154` and report PR `#171`, except where the edge-side interface or handover is relevant. Membership construction, Blend cryptographic proofs, swarm transport correctness, consensus rules outside the proposal handover, and third-party crates such as `tokio`, `overwatch`, `libp2p`, and `rust-rapidsnark` were not audited as independent components.

**Assumptions**

The target is the source tree at commit `a805329f8a186eb6989f09a7c49dee4a0e07473b`. The host is not compromised, and `abstain_on_failure = false` is the intended production configuration for the failure-detector findings. The Blend protocol text at the stated Logos LIPs commit is the governing behavior. A direct HTTP caller may retry, but the API's accepted response is treated as an application-level acceptance contract rather than proof that the transaction was sent. No remote primitive to restart the target service was demonstrated; timing a known operator restart is a separate precondition from causing one.

## 3. Method

- Manual review of the issue `#172` scope, parent issue `#14`, and prior report PR `#171`.
- Spec conformance against the stated Logos LIPs commit for `Edge Network`, `Failure Detection and Reaction`, and `Transition Period`.
- Static tracing from the direct HTTP Blend endpoint through the edge queue, epoch encapsulation, delivery failure detector, mode orchestrator, normal transaction mempool recovery, and chain-network tip/orphan recovery path.
- Automated tooling: `CARGO_TARGET_DIR=/tmp/logos-blockchain-agent-audit-target cargo test -p logos-blockchain-blend-service --lib edge::tests` — `15 passed; 0 failed; 1 ignored; 81 filtered out`.
- Dynamic testing: the existing edge unit tests passed. They cover normal encapsulation, delivery fallback, proposal queuing, and same-process epoch behavior, but do not provide dedicated restart or mode-handover regression coverage for LB-001 or LB-003.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Edge-local accepted Blend work and delivery deadlines are not recoverable after restart | Denial of Service | Low | High | Open |
| LB-003 | One-second mode-switch grace can abort the edge direct-broadcast drain | Denial of Service | Low | High | Open |

### LB-001 · Edge-local accepted Blend work and delivery deadlines are not recoverable after restart

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Denial of Service |
| Target | `services/blend/src/edge/mod.rs:L85-L114, L314-L465`; `services/blend/src/delivery/failure_detection.rs:L29-L136`; `services/api/src/http/blend.rs:L57-L118`; `services/chain/chain-leader/src/lib.rs:L761-L784`; `services/chain/chain-network/src/sync/tip_poll.rs:L13-L26` |
| Status | Open |

**Description**

The edge service declares `State = NoState<Self::Settings>` and `StateOperator = NoOperator<Self::State>` in `ServiceData` (`edge/mod.rs:L85-L114`), and its `init` function ignores the initial state (`edge/mod.rs:L160-L167`). The running edge therefore owns its pending transaction queue, epoch-local proposal queue, and delivery detector only in memory:

- `pending_transactions` is created as `PendingTransactions::new()` when the edge run loop starts (`edge/mod.rs:L360`).
- `CurrentEpoch` owns a `PendingProposals` queue, and a new epoch creates a fresh queue (`current_epoch.rs:L47-L50, L125-L142`).
- `FailureDetector` stores unacknowledged payloads and its current round in memory (`failure_detection.rs:L29-L57`); its round starts from zero on every service start.

The direct Blend HTTP endpoint documents that a successful response means the transaction was “accepted for blending,” not that it was sent, and specifically avoids adding the transaction to the node's own mempool (`services/api/src/http/blend.rs:L57-L82`). The normal tx-service persistence path only snapshots the regular mempool after `MempoolMsg::Add` (`services/tx-service/src/tx/service.rs:L497-L515`, `tx/state.rs:L8-L47`). Direct Blend requests do not enter that path; they remain in the edge queue until encapsulated and later submitted by a core dispatcher. Consequently, tx-service recovery cannot re-submit a direct Blend request after an edge restart.

There is also a documentation/invariant mismatch in `PendingTransactions`: its type documentation says the queue “is persisted so it survives a restart,” but the type is an in-memory `VecDeque`. Persistence is supplied by callers, and edge supplies no such caller-side state. Core explicitly mirrors its normal transaction path into recoverable tx-service state; edge direct-Blend requests do not follow that path.

The fallback state is also volatile. The detector registers a payload only after edge encapsulation/send (`edge/mod.rs:L430-L446`), and the expired-payload queue is drained by waiting for the release deadline before direct dispatch (`failure_detection.rs:L111-L136`). A restart loses both the outstanding payload map and the deadline/round context, so a payload that was sent to a failed Blend path may never be directly broadcast.

Block proposals have a different impact profile. Chain leader applies its own block before handing the proposal to Blend (`services/chain/chain-leader/src/lib.rs:L761-L784`), so a restart at that point does not erase the locally applied block. When an ahead peer has the block or a descendant, chain-network tip polling can sample that peer's tip and hand it to the orphan downloader for eventual catch-up (`services/chain/chain-network/src/sync/tip_poll.rs:L13-L26`). These routes are not equivalent to prompt Blend dissemination or its `T_M` fallback, but the proposal itself does not disappear in the same sense as an unsent direct transaction.

**Exploit scenario**

An operator restarts the node while a direct Blend transaction is queued, while it is being sent, or while its failure detector is waiting for the `T_M` observation deadline. The in-memory queue or detector entry disappears. If the transaction was not already accepted by a core, it is not re-submitted by tx-service, and if the Blend path failed it is not sent through the direct-broadcast fallback. A remote primitive to cause that restart was not demonstrated; a caller can only submit its own transaction before a separately occurring restart. For proposals, the locally applied block may later be recovered through chain synchronization, but the timely Blend dissemination and fallback path are lost. This is a restart/durability/liveness failure with security consequences, not a demonstrated remotely induced DoS.

**Recommendation**

- *Short term*: separate the recovery policy for (1) unreleased direct transactions, (2) released payloads with outstanding deadlines, and (3) unreleased epoch-bound proposals. Persist the first two, remove a queued item only after the relevant handoff is durable, and restore outstanding deadlines using a persisted wall-clock/epoch reference rather than a `FailureDetector` round that restarts at zero. Do not blindly restore proposals until the protocol question in S-001 is resolved.
- *Long term*: move the durable egress queue and failure detector into a service whose lifetime spans edge/core/broadcast mode changes, or introduce an explicit handover protocol. Add restart tests covering a queued transaction, a sent-but-unobserved transaction, and a proposal awaiting proof.

**References**: Logos LIPs `blend-protocol.md`, `Failure Detection and Reaction › Detection` and `Direct Broadcast`; issue `#154` and report PR `#171` for the corresponding core-side recovery gap; `PendingTransactions` documentation in `services/blend/src/pending.rs`.

### LB-003 · One-second mode-switch grace can abort the edge direct-broadcast drain

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Denial of Service |
| Target | `services/blend/src/orchestrator/instance.rs:L50-L55, L136-L179, L205-L225`; `services/blend/src/orchestrator/on_demand.rs:L71-L105`; `services/blend/src/edge/mod.rs:L376-L390`; `services/blend/src/delivery/failure_detection.rs:L111-L136` |
| Status | Open |

**Description**

On a mode transition away from edge, the orchestrator stops the old instance and waits only `SHUTDOWN_GRACE = 1s` for edge/broadcast services (`orchestrator/instance.rs:L50-L55, L205-L225`). It does not initially tell edge to enter a graceful drain. The on-demand stop path first waits for the service to report that it stopped; only when the deadline expires does it kill the service (`orchestrator/on_demand.rs:L71-L105`). In the pinned Overwatch revision, stopping a still-running service task aborts that task.

Separately, edge reacts to a new membership/epoch event by calling `failure_detection.drain_pending_message_queue(...)` before returning (`edge/mod.rs:L376-L390`). That drain waits for the detector's configured maximum delivery deadline (`failure_detection.rs:L111-L136`), which is computed from the Blend layer count and release delay (`services/blend/src/settings/mod.rs:L143-L167`). The pinned specification's parameterization gives `T_M = 15` one-second rounds, while the orchestrator grace is only one second. The one-second orchestrator grace is not tied to the delivery deadline and can expire first. The top-level Blend service and edge also maintain independent epoch subscriptions (`services/blend/src/lib.rs:L236-L315` and `edge/mod.rs:L241-L281`), each performing asynchronous work before emitting its event. There is no ordering guarantee that edge observes the transition and begins draining before the orchestrator's grace timer starts.

**Exploit scenario**

An edge sends a payload into a Blend network that does not deliver it, then the node changes to core or broadcast mode before the `T_M` observation deadline. The orchestrator begins waiting for the old edge to stop, while edge may not yet have observed the same transition or may already be waiting out its legitimate drain. After one second, the orchestrator can abort the still-running edge task before direct broadcast occurs. The detector entry is then lost with the task. The same transition also loses any not-yet-encapsulated queue described in LB-001. The precondition requires useful non-delivery and a mode transition during an outstanding delivery window, so exploitation is difficult and severity is Low.

**Recommendation**

- *Short term*: stop routing new work to the old edge instance, explicitly instruct it to enter drain mode, and await a drain-complete/stopped acknowledgement before killing it. A watchdog may enforce an upper bound, but it must start after drain initiation and exceed the maximum outstanding delivery deadline with margin; merely changing `1s` to `T_M + margin` is insufficient if the timer starts before edge has observed the transition.
- *Long term*: transfer the durable queue and detector to the long-lived Blend proxy before changing modes, so the outgoing mode owns the same recovery/fallback obligations. Add an orchestrator-level mode-transition test with an intentionally undelivered payload and an outstanding deadline, rather than only an isolated edge test.

**References**: Logos LIPs `blend-protocol.md`, `Failure Detection and Reaction › Detection`, `Direct Broadcast`, and `Transition Period`; pinned Overwatch revision `ae887f41f5a626c341179026ad7f03953ff2072e`; prior report PR `#171`, finding LB-002, for the related core shutdown timing issue.

## 5. Suggestions

### S-001 · Clarify the fate of never-released epoch-bound proposals

At epoch rotation, edge drops proposals that never reached Blend's message-generation/release stage. This appears consistent with the current design: `CurrentEpoch` and `PendingProposals` are epoch-bound, leadership quota and proof material are tied to the old epoch, and the protocol starts `T_M` from the round in which a payload was released. A never-released proposal therefore has no normative `T_M` deadline or current direct-broadcast requirement. The protocol should state whether this is intentional.

If the protocol team decides that such a proposal must eventually receive direct fallback, specify the synthetic reference time, privacy implications of revealing it directly, consensus freshness/validity checks, and interaction with old-epoch leadership quota before changing the implementation. The locally applied block may also be recovered through chain synchronization, so this is a protocol/design question rather than an established security defect.

**References**: Logos LIPs `blend-protocol.md`, `Failure Detection and Reaction › Detection`, `Direct Broadcast`, and `Transition Period`; prior report PR `#171`, which described synthetic release at epoch end as a proposal to spec editors.

### S-002 · Make accepted Blend API semantics durable and observable

The direct transaction endpoint returns a hash after handing the payload to the Blend relay, while `GetPendingTransactions` exposes only the current in-memory queue and does not promise restart recovery. Consider returning an acceptance identifier tied to durable queue state, exposing queue/recovery status, or changing the response wording and retry contract until durable acceptance exists.

### S-003 · Centralize delivery ownership across Blend modes

The edge, core, and broadcast modes each have different local queues and shutdown behavior. A single long-lived delivery owner would reduce the number of handover races and make the `T_M`/transition-period obligations explicit in one place.
