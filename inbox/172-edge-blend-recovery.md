# Audit Report — Blend edge state, delivery fallback, and mode transitions

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/172`  
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `services/blend/src/edge`, `services/blend/src/orchestrator`, `services/blend/src/delivery`, `services/blend/src/pending.rs`, `services/blend/src/lib.rs`, `services/api/src/http/blend.rs`, `services/tx-service`, `services/chain/chain-leader`  
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, and `blend-protocol.md` sections `Edge Network`, `Failure Detection and Reaction`, and `Transition Period`  
Date: `2026-09-14` — author: `Codex` — status: `final`

---

## 1. Summary

- Overall assessment: Edge mode accepts work into process-local queues and process-local failure detection, so restart and some mode transitions can lose payloads or suppress the required direct-broadcast fallback.
- Findings: `0 C` critical · `0 H` high · `1 M` medium · `2 L` low · `0 I` informational
- Key themes: non-durable accepted work, dropped epoch-bound proposals, and shutdown timing that is shorter than the delivery deadline.
- Must-fix before launch:
  - Make edge-accepted transactions, epoch-bound proposals, and outstanding delivery deadlines recoverable or transfer them to a component with equivalent recovery guarantees.
  - Define and implement the fate of proposals that are still queued when their epoch rotates.
  - Ensure edge shutdown during a mode transition completes its delivery-drain deadline, or transfers that responsibility before the old service is killed.

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
| `services/tx-service` and `services/chain/chain-leader` | Whether another recoverable service resubmits direct Blend requests or proposals. |

**Out of scope**

Core-specific recovery and retirement behavior already covered by issue `#154` and report PR `#171`, except where the edge-side interface or handover is relevant. Membership construction, Blend cryptographic proofs, swarm transport correctness, consensus rules outside the proposal handover, and third-party crates such as `tokio`, `overwatch`, `libp2p`, and `rust-rapidsnark` were not audited as independent components.

**Assumptions**

The target is the source tree at commit `a805329f8a186eb6989f09a7c49dee4a0e07473b`. The host is not compromised, and `abstain_on_failure = false` is the intended production configuration for the failure-detector findings. The Blend protocol text at the stated Logos LIPs commit is the governing behavior. A direct HTTP caller may retry, but the API's accepted response is treated as an application-level acceptance contract rather than proof that the transaction was sent.

## 3. Method

- Manual review of the issue `#172` scope, parent issue `#14`, and prior report PR `#171`.
- Spec conformance against the stated Logos LIPs commit for `Edge Network`, `Failure Detection and Reaction`, and `Transition Period`.
- Static tracing from the direct HTTP Blend endpoint through the edge queue, epoch encapsulation, delivery failure detector, mode orchestrator, and normal transaction mempool recovery.
- Automated tooling: `CARGO_TARGET_DIR=/tmp/logos-blockchain-agent-audit-target cargo test -p logos-blockchain-blend-service --lib edge::tests` — `15 passed; 0 failed; 1 ignored; 81 filtered out`.
- Dynamic testing: the existing edge unit tests passed. They cover normal encapsulation, delivery fallback, proposal queuing, and same-process epoch behavior, but do not provide dedicated restart or mode-handover regression coverage for LB-001 or LB-003.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Edge-local accepted Blend work and delivery deadlines are not recoverable after restart | Denial of Service | Medium | Medium | Open |
| LB-002 | Edge proposals waiting for proof are dropped at epoch rotation | Denial of Service | Low | Medium | Open |
| LB-003 | One-second mode-switch grace can abort the edge direct-broadcast drain | Denial of Service | Low | High | Open |

### LB-001 · Edge-local accepted Blend work and delivery deadlines are not recoverable after restart

| | |
|---|---|
| Severity | Medium |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | `services/blend/src/edge/mod.rs:L85-L114, L314-L465`; `services/blend/src/delivery/failure_detection.rs:L29-L136`; `services/api/src/http/blend.rs:L57-L118` |
| Status | Open |

**Description**

The edge service declares `State = NoState<Self::Settings>` and `StateOperator = NoOperator<Self::State>` in `ServiceData` (`edge/mod.rs:L85-L114`), and its `init` function ignores the initial state (`edge/mod.rs:L160-L167`). The running edge therefore owns its pending transaction queue, epoch-local proposal queue, and delivery detector only in memory:

- `pending_transactions` is created as `PendingTransactions::new()` when the edge run loop starts (`edge/mod.rs:L360`).
- `CurrentEpoch` owns a `PendingProposals` queue, and a new epoch creates a fresh queue (`current_epoch.rs:L47-L50, L125-L142`).
- `FailureDetector` stores unacknowledged payloads and its current round in memory (`failure_detection.rs:L29-L57`); its round starts from zero on every service start.

The direct Blend HTTP endpoint documents that a successful response means the transaction was “accepted for blending,” not that it was sent, and specifically avoids adding the transaction to the node's own mempool (`services/api/src/http/blend.rs:L57-L82`). The normal tx-service persistence path only snapshots the regular mempool after `MempoolMsg::Add` (`services/tx-service/src/tx/service.rs:L497-L515`, `tx/state.rs:L8-L47`). Direct Blend requests do not enter that path; they remain in the edge queue until encapsulated and later submitted by a core dispatcher. Consequently, tx-service recovery cannot re-submit a direct Blend request after an edge restart.

The fallback state is also volatile. The detector registers a payload only after edge encapsulation/send (`edge/mod.rs:L430-L446`), and the expired-payload queue is drained by waiting for the release deadline before direct dispatch (`failure_detection.rs:L111-L136`). A restart loses both the outstanding payload map and the deadline/round context, so a payload that was sent to a failed Blend path may never be directly broadcast.

**Exploit scenario**

An ordinary caller submits a direct Blend transaction and receives its accepted response. The node process is restarted while the transaction is queued, while it is being sent, or while its failure detector is waiting for the `T_M` observation deadline. The in-memory queue or detector entry disappears. If the transaction was not already accepted by a core, it is not re-submitted by tx-service, and if the Blend path failed it is not sent through the direct-broadcast fallback. A local block proposal can be lost in the same way after chain-leader hands it to Blend but before edge encapsulation or fallback. This is a liveness/availability failure requiring no special network capability beyond causing or timing a routine operator restart; the direct endpoint's accepted response makes the transaction loss externally surprising.

**Recommendation**

- *Short term*: persist the edge queues and the identity/deadline of every sent-but-unobserved payload. Remove a queued item only after the relevant handoff is durable, and restore outstanding deadlines using a persisted wall-clock/epoch reference rather than a `FailureDetector` round that restarts at zero. Define whether restored proposals are re-encapsulated only under a valid epoch or sent through the direct fallback.
- *Long term*: move the durable egress queue and failure detector into a service whose lifetime spans edge/core/broadcast mode changes, or introduce an explicit handover protocol. Add restart tests covering a queued transaction, a sent-but-unobserved transaction, and a proposal awaiting proof.

**References**: Logos LIPs `blend-protocol.md`, `Failure Detection and Reaction › Detection` and `Direct Broadcast`; issue `#154` and report PR `#171` for the corresponding core-side recovery gap.

### LB-002 · Edge proposals waiting for proof are dropped at epoch rotation

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | `services/blend/src/edge/mod.rs:L376-L396`; `services/blend/src/edge/current_epoch.rs:L125-L142`; `services/blend/src/pending.rs:L101-L169` |
| Status | Open |

**Description**

When an epoch event arrives, edge constructs a new `CurrentEpoch` (`edge/mod.rs:L376-L396`). `CurrentEpoch::try_new` always allocates a new `PendingProposals` queue for the new epoch (`current_epoch.rs:L125-L142`). The old queue is therefore dropped, and `PendingProposals::Drop` only logs a warning before discarding its entries (`pending.rs:L159-L169`). The queue is not handed to the failure detector, which only tracks payloads after they have been encapsulated and sent (`edge/mod.rs:L430-L446`).

This is distinct from pending transactions: the edge-level `PendingTransactions` value is outside `CurrentEpoch`, so a transaction can remain queued across a same-process edge epoch change. The proposal queue is epoch-bound because its proof material cannot simply be reused under a different epoch. The chain leader applies its proposal and hands it to Blend (`services/chain/chain-leader/src/blend.rs:L44-L60`), but there is no separate chain-leader direct-broadcast fallback for a proposal that Blend never releases.

**Exploit scenario**

A locally produced proposal reaches edge near an epoch boundary and remains queued while proof-of-life or proof-of-qualification information is unavailable. The epoch changes before edge encapsulates it. The old queue is dropped, so the proposal is not delivered by Blend and is not registered for the existing delivery fallback. This can suppress a valid local proposal during routine delayed proof delivery or a boundary race. The impact is limited to proposals in this timing window, hence Low severity.

**Recommendation**

- *Short term*: explicitly transfer an old proposal to a terminal fallback path at rotation. It must not be re-encapsulated under the new epoch unless the protocol confirms that its proof material remains valid. A direct broadcast may be scheduled at the old proposal's deadline, provided it cannot occur earlier than the protocol's `T_M` safety condition.
- *Long term*: specify the fate of never-released epoch-bound proposals in `blend-protocol.md`, and add an epoch-rotation test proving either direct fallback or an intentional, documented discard with an upstream recovery mechanism.

**References**: Logos LIPs `blend-protocol.md`, `Failure Detection and Reaction › Detection` and `Direct Broadcast`; prior report PR `#171`, finding LB-003, which identified the analogous core queue drop.

### LB-003 · One-second mode-switch grace can abort the edge direct-broadcast drain

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Denial of Service |
| Target | `services/blend/src/orchestrator/instance.rs:L50-L55, L136-L179, L205-L225`; `services/blend/src/orchestrator/on_demand.rs:L71-L105`; `services/blend/src/edge/mod.rs:L376-L390`; `services/blend/src/delivery/failure_detection.rs:L111-L136` |
| Status | Open |

**Description**

On a mode transition away from edge, the orchestrator stops the old instance and waits only `SHUTDOWN_GRACE = 1s` for edge/broadcast services (`orchestrator/instance.rs:L50-L55, L205-L225`). The on-demand stop path waits for a status change and then kills the service if the deadline expires (`orchestrator/on_demand.rs:L71-L105`).

Separately, edge reacts to a new membership/epoch event by calling `failure_detection.drain_pending_message_queue(...)` before returning (`edge/mod.rs:L376-L390`). That drain waits for the detector's configured maximum delivery deadline (`failure_detection.rs:L111-L136`), which is computed from the Blend layer count and release delay (`services/blend/src/settings/mod.rs:L143-L167`). The one-second orchestrator grace is not tied to that deadline and can expire first. The top-level Blend service and edge also maintain independent epoch subscriptions (`services/blend/src/lib.rs:L236-L315` and `edge/mod.rs:L241-L281`), so the orchestrator kill can race the edge's own drain rather than waiting for its completion.

**Exploit scenario**

An edge sends a payload into a Blend network that does not deliver it, then the node changes to core or broadcast mode before the `T_M` observation deadline. If edge begins its drain, the orchestrator can kill it after one second, before direct broadcast occurs. If the kill wins the race, the detector entry is lost with the process. The same transition also loses any not-yet-encapsulated queue described in LB-001. The precondition requires a mode transition during an outstanding delivery window, so exploitation is difficult and severity is Low.

**Recommendation**

- *Short term*: make edge shutdown wait at least the configured delivery deadline plus a margin, or make the stop path explicitly await completion of the detector drain before killing the service. Do not use a fixed one-second grace for a deadline-driven protocol.
- *Long term*: transfer the durable queue and detector to the long-lived Blend proxy before changing modes, so the outgoing mode owns the same recovery/fallback obligations. Add a mode-transition test with an intentionally undelivered payload and a deadline longer than the current grace period.

**References**: Logos LIPs `blend-protocol.md`, `Failure Detection and Reaction › Detection`, `Direct Broadcast`, and `Transition Period`; prior report PR `#171`, finding LB-002, for the related core shutdown timing issue.

## 5. Suggestions

### S-001 · Make accepted Blend API semantics durable and observable

The direct transaction endpoint returns a hash after handing the payload to the Blend relay, while `GetPendingTransactions` exposes only the current in-memory queue and does not promise restart recovery. Consider returning an acceptance identifier tied to durable queue state, exposing queue/recovery status, or changing the response wording and retry contract until durable acceptance exists.

### S-002 · Centralize delivery ownership across Blend modes

The edge, core, and broadcast modes each have different local queues and shutdown behavior. A single long-lived delivery owner would reduce the number of handover races and make the `T_M`/transition-period obligations explicit in one place.
