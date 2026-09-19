# Audit Report — Blend transaction sender: what a sender can learn about a lost transaction when `abstain_on_failure` is set

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/588`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `9ffddb30b9e6cf79465802953caedd010ff1cecd` — component(s): `services/blend/src/core`, `services/blend/src/edge`, `services/blend/src/delivery`, `services/blend/src/pending.rs`, `services/api/src/http/blend.rs`, `services/pow/src/service.rs`, `services/chain/chain-leader`, `services/chain/chain-network`
Specs: `https://github.com/logos-co/logos-lips` @ `75d3d0382604d4a0d8e246c268935dd386ffc8ea` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `blend-protocol.md` (all in full)
Date: `2026-09-19` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: the gap issue #588 describes is real at this commit and was reproduced at the service level. With `abstain_on_failure: true` the node switches off delivery *detection* together with the direct broadcast, so a transaction whose single Blend copy is lost leaves nothing behind: no log line above `trace`, no metric, and an empty pending-transactions query. The specification asks for detection unconditionally and makes only the reaction optional, so this is a spec deviation. The gap has one caller inside the node that the issue did not know about: the PoW service publishes reward claims through Blend, moves the claimed tickets to a pending set, and never returns them to the ready set, so a lost claim forfeits its rewards when the reward window closes.
- Findings: 0 critical · 0 high · 1 medium · 1 low · 1 informational
- Key themes: detection and reaction share one switch; fire-and-forget submission with no sender-side record; a caller that gives up rather than retrying; a proposer's own chain view is not a delivery signal.
- Must-fix before launch: LB-001 (PoW reward claims are not retried, so a lost claim forfeits the reward).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/blend/src/core/mod.rs` (`run`, `handle_service_message`, `queue_transaction_for_encapsulation`, `handle_local_transaction`, `schedule_local_encapsulated_message`, the release-round handlers) | where the failure detector is created or omitted; the life of a locally submitted transaction from queue to release |
| `services/blend/src/edge/mod.rs` (`run`), `services/blend/src/edge/current_epoch.rs`, `services/blend/src/edge/handlers.rs`, `services/blend/src/edge/backends/libp2p/` | the same on the edge side; what the first-hop send logs |
| `services/blend/src/delivery/mod.rs`, `services/blend/src/delivery/failure_detection.rs`, `services/blend/src/core/delivery.rs` | what the detector records and what it reports |
| `services/blend/src/core/dispatcher/libp2p.rs` | what counts as an observed delivery for a transaction and for a proposal |
| `services/blend/src/metrics.rs` | every Blend metric, checked for one that distinguishes a delivered payload from a lost one |
| `services/api/src/http/blend.rs`, `nodes/node/binary/src/api/handlers.rs`, `nodes/api-common/src/paths.rs`, `nodes/node/http-client/src/lib.rs` | the submission route, the pending-transactions query and their clients |
| `services/pow/src/service.rs` (`claim_ready_rewards`, `publish_reward_claim`, `prune_expired_tickets`, `prune_settled_pending`, `respond_claimable_rewards`, the run loop) | the in-node caller that submits transactions through Blend |
| `core/src/mantle/ops/pow.rs` (`validate_double_claiming`) | whether resubmitting a claim can pay twice |
| `services/chain/chain-leader/src/lib.rs` (`apply_and_publish_block_proposal`), `services/chain/chain-leader/src/blend.rs`, `services/chain/chain-network/src/lib.rs` (`note_received_proposal`, `handle_incoming_proposal`) | item 4: what a proposer learns about its own proposal |
| `nodes/node/binary/src/config/mod.rs`, `nodes/node/binary/src/config/blend/`, `deployment/ceremony/genesis/*/deployment-template.yaml` | how the flag is set; shipped `num_blend_layers`, `data_replication_factor`, PoW `slot_window` |
| `wallet/`, `services/wallet/`, `services/tx-service/`, `wallet-http-client/`, `zone-sdk/`, `c-bindings/src/api/blend.rs` | searched for any other caller of the Blend submission path |

**Out of scope**

- The failure detector's deadline arithmetic, the lag switch and the loop stalls that lengthen the deadline: covered by the #276 and #576 reports and only referenced here.
- How the mempool treats a duplicate `Add` (#277 report).
- Whether a reward-claim transaction can be delivered and still never be included in a block, for example because the execution base fee rose after the fee was estimated. The claim path has the same give-up behaviour in that case (see LB-001), but the fee logic was not reviewed.
- The PoW puzzle, the reward difficulty and the claim proof (`services/pow/src/` other than the claim bookkeeping; issue #243).
- The PoQ and encapsulation cryptography.
- Third-party crates assumed correct: `tokio`, `futures`, `libp2p`, `overwatch`.

**Assumptions**

- The specification at the commit above is the reference.
- The deployment templates under `deployment/ceremony/genesis/` carry the values each network will ship with.
- An adversary can run declared core nodes (it holds the minimum stake) and may also mine.

## 3. Method

- Manual review of the in-scope paths, working through the four items of issue `#588` under parent `#12`, starting from S-002 of the #279 report. Line numbers in the issue were taken at `3d5d419ec`; every citation below was re-located at `9ffddb30b`.
- Spec conformance against `blend-protocol.md`, read in full, with › Failure Detection and Reaction (Overview and Protocol), › Transition Period, › Generation, › Proof of Work Quota and › Global Parameters as the governing sections.
- Repository-wide search for producers of `DataPayload::Transaction` and for callers of `blend_transaction`, `BlendServiceApi::publish` and the `/blend/transactions/*` routes.
- Automated tooling: none beyond `cargo test`.
- Dynamic testing: one experiment test added to a scratch copy of the tree (never to the audited checkout), in `services/blend/src/edge/tests/mod.rs`, run with `rustc 1.98.1` as pinned by `rust-toolchain.toml`: `cargo test -p logos-blockchain-blend-service --lib -- edge::tests::audit_588`. It uses the crate's own edge-service harness (`spawn_run_without_direct_broadcast`, mock backend and mock proof generator). The test is reproduced in 3.2. It passes. The existing test `a_node_that_does_not_bypass_never_broadcasts_in_the_clear` was run alongside it and passes.
- Not done: the devnet run item 1 asks for. `tests/cucumber_tests/features/blend.feature` ("Transactions submitted through Blend come back through the mempool") is the scenario to extend, but it needs a multi-node cluster with the circuit artefacts, which this iteration did not set up. It is left as a follow-up issue.

### 3.1 Item by item

**Item 1 — does a lost transaction leave a trace?** No, confirmed at the service level rather than on a devnet.

The flag removes the detector entirely on both sides:

```rust
// services/blend/src/core/mod.rs:465-473
let mut failure_detector = if running_blend_config.abstain_on_failure {
    None
} else {
    Some(FailureDetector::new(…, payload_dispatcher.observe_broadcasts().await))
};

// services/blend/src/edge/mod.rs:362-372
// `None` when the operator has turned the fallback off, which records
// nothing, watches nothing and can reveal nothing.
let mut failure_detection = if settings.abstain_on_failure { None } else { Some(…) };
```

Every later use is guarded by that `Option`: `mark_payload_as_encapsulated` (`core/mod.rs:2010-2012`, `:2062-2064`), `mark_encapsulated_payload_as_released` (`core/mod.rs:2417-2419`, `:2508-2510`, `:2599-2602`), `mark_payload_as_blended` (`edge/mod.rs:443-445`). `next_undelivered_messages` (`services/blend/src/delivery/mod.rs:17-28`) parks forever on `None`. The subscription to the mempool's accepted stream (`observe_transactions`, `core/dispatcher/libp2p.rs:191-220`) is never opened.

What remains for a submitted transaction:

| Stage | Core node | Edge node |
|---|---|---|
| accepted | queued in `PendingTransactions` and written to the recovery state (`core/mod.rs:1476-1488`) | queued in memory only (`edge/mod.rs:411-413`); a restart loses it |
| waiting for a PoW solution | reported by `GetPendingTransactions` (`core/mod.rs:1271-1274`) | same (`edge/mod.rs:424-426`) |
| encapsulated | popped from the queue and from the recovery state (`core/mod.rs:1608-1611`); the wrapped message sits in the recovery state as an unsent data message until its release round | sent at once and popped (`edge/mod.rs:442-448`) |
| released | removed from the recovery state (`core/mod.rs:2412-2414`); `blend_messages_sent_total{action="publish"}` increments, as it does for every cover message | the libp2p backend logs `Message sent successfully to peer …` at `info` (`edge/backends/libp2p/swarm.rs:617`) or `No alternative peer available … Dropping the message` at `warn` (`:305`), neither naming the payload |
| after `T_M` | nothing | nothing |

The only log line that names the payload kind is the `trace` line in the prover (`blend/provers/src/crypto/leader/send.rs:203`, `core_and_leader/send.rs:244`), and it carries no transaction hash. The only payload-typed metric, `blend_payloads_bypassed_total` (`services/blend/src/metrics.rs:74-82`), is emitted from `broadcast_undelivered_messages`, which an abstaining node never reaches. So for an abstaining sender a delivered transaction and a lost one produce identical logs, metrics and query results. The first hop of an edge node is the exception: a failed dial or an exhausted peer list is logged, so a loss *before* the entry core node is visible as a generic send failure.

The experiment in 3.2 confirms the two API-visible halves: no dispatch after the deadline, and an empty `GetPendingTransactions` reply.

**Item 2 — do callers poll for their own hash?** There are two callers at this commit, and neither the issue nor the #279 report knew of the second.

1. The HTTP route `POST /blend/transactions/disperse` (`nodes/node/binary/src/api/handlers.rs:738-761`, `services/api/src/http/blend.rs:57-83`). Its only in-repo clients are the node HTTP client (`nodes/node/http-client/src/lib.rs:320`) and the test framework (`tests/src/cucumber/steps/mempool/actions.rs:108`). Both are a single POST. No wallet, `tx-service`, `zone-sdk`, `wallet-http-client` or FFI code calls it; `c-bindings/src/api/blend.rs` exposes only `blend_join_as_core_node` and `blend_info`. An external client that wants confirmation has to poll `GET /mempool/view` or the chain for its hash; nothing in the repository does so or documents a wait.
2. The PoW service. `publish_reward_claim` (`services/pow/src/service.rs:1502-1514`) hands every reward-claim transaction to `BlendServiceApi::publish`, which is documented "Fire-and-forget" (`services/blend/src/api.rs:77-86`). The service does watch for settlement, through `PoWRewardClaimed` block events (`prune_settled_pending`, `:1188-1221`), but it never resubmits: see LB-001. It waits until the ticket's reward window closes, which is `slot_window = 300` slots in every shipped template, and then drops the ticket.

**Item 3 — is an observation-only detector enough, and does resubmitting through Blend have a downside?**

Keeping the hash is enough to tell the operator, and it is safe in the two ways the current `None` was chosen to be safe:

- *It reveals nothing on the network.* The reveal comes from `broadcast_undelivered_messages`, not from watching. A detector that records and watches but whose expiry is reported instead of dispatched sends nothing.
- *The lag switch stops mattering.* `stop_observing_on_lag` (`core/dispatcher/libp2p.rs:101-118`) exists because a missed observation makes a delivered payload look lost and the node then reveals it (#276 LB-001). With no dispatch, a missed observation costs a false "unconfirmed" entry and, at worst, one redundant resubmission.
- *What it stores.* With the fallback on, the detector already keeps the full payload for `T_M` rounds, and a core node already persists queued transactions in its recovery state. Keeping a 32-byte hash per released transaction until it is confirmed or queried is a smaller local record than either. It is still a record of what this node originated, so it should be bounded (drop after a fixed number of rounds) and in memory only.

Resubmitting the same transaction bytes through Blend:

- costs one more PoW solution (`Q_W = ß_max`, one solution per message, `blend-protocol.md` › Proof of Work Quota);
- produces an independent message: fresh ephemeral keys, fresh PoQ nullifiers, a fresh path. No verifier can link it to the first copy before the exit node;
- is harmless if the first copy was only late. Both copies decode to the same transaction hash, and the mempool treats the second `Add` as a duplicate (#277 report). The exit node of the second copy logs the refusal at `debug` (`core/dispatcher/libp2p.rs:187`);
- shows the payload to a second exit node. Each exit node learns the transaction, which the mempool would gossip to it moments later anyway; neither learns the sender.

For a PoW claim the resubmission is a *new* transaction (the batch, the fee and the change outputs are rebuilt), so both can reach a mempool. The ledger rejects the second: `validate_double_claiming` returns `DoubleClaimed` when the puzzle ticket is already in `pow_nullifiers` (`core/src/mantle/ops/pow.rs:218-227`). No reward is paid twice.

The one real cost is timing. `T_M` rounds after a loss the sender emits a second message. Blend messages are indistinguishable on the wire and a transaction removes no cover message (› Releasing), so to a network observer this is one more message from a node that emits about one per round. For an edge node, which opens a connection per message, a retry is one more connection `T_M` rounds after the first; that is a weaker pattern than the direct broadcast it replaces, but it is a pattern, and a randomised retry delay would blunt it.

**Item 4 — do block proposals have the same gap?** Yes, and the chain view is a weaker signal than the #279 report assumed. See LB-003.

### 3.2 The experiment

Appended to `services/blend/src/edge/tests/mod.rs` in a scratch copy of the tree at the audited commit:

```rust
#[test_log::test(tokio::test(start_paused = true))]
async fn audit_588_lost_transaction_leaves_no_trace_when_abstaining() {
    let local_node = NodeId(99);
    let core_node = NodeId(0);
    let transaction = vec![3; 8];
    let RunningEdgeService { messages: msg_sender, blended_to: mut node_id_receiver,
        mut broadcasting_channel, .. } = spawn_run_without_direct_broadcast(
        local_node, 1, Some(membership(&[core_node], local_node))).await;

    msg_sender.send(DataPayload::Transaction(transaction.clone()).into()).await.expect("channel opened");
    // The single copy left for its first hop.
    assert_eq!(node_id_receiver.recv().await.expect("channel opened"), core_node);

    // Nothing comes back on the broadcasting channel, and nothing is dispatched after the deadline.
    assert!(timeout(TEST_ROUND * u32::try_from(TEST_DELIVERY_DEADLINE.get() + 4).unwrap(),
        broadcasting_channel.dispatched.recv()).await.is_err());

    // The only query a sender has reports nothing either.
    let (reply, pending) = oneshot::channel();
    msg_sender.send(ServiceMessage::GetPendingTransactions { reply }).await.expect("channel opened");
    let pending: Vec<Vec<u8>> = pending.await.expect("reply sent");
    assert!(pending.is_empty());
}
```

Result: `test edge::tests::audit_588_lost_transaction_leaves_no_trace_when_abstaining ... ok`. Run again with `RUST_LOG=trace --nocapture`, the service emitted four lines in total: membership ready (`info`), mode chosen (`debug`), handler created (`debug`), and `Encapsulating layer 0 of data message type Transaction for node at index 0.` (`trace`). Nothing after the send, nothing at the deadline. The harness uses a mock backend, so the real backend's first-hop `info` line is absent from this run.

### 3.3 Ruled out

- **A second copy or a retry inside the Blend service.** `PendingTransactions::mark_as_sent` pops the head; there is no copy counter for transactions as there is for proposals (`services/blend/src/pending.rs`), and no path re-queues a transaction.
- **The recovery state as a record.** On a core node the released message is removed from the recovery state in the same `inspect` closure that publishes it (`core/mod.rs:2410-2420`). Nothing survives a release.
- **The flag being reachable from the network.** `abstain_on_failure` is set only from the operator's configuration file or `--blend-abstain-on-failure` / `BLEND_ABSTAIN_ON_FAILURE` (`nodes/node/binary/src/config/mod.rs:264-269`, `:478-480`). It defaults to `false`.
- **A double reward from a retried claim.** Rejected by `validate_double_claiming`, above.
- **Other in-node producers of transaction payloads.** `DataPayload::try_from_transaction` has three call sites outside tests: the HTTP route, the PoW service and the dispatcher's observer. `BlendServiceApi::publish` has one caller, the PoW service.
- **`/pow/rewards/claimable` as a view of pending claims.** It reports `ready_to_claim` only (`services/pow/src/service.rs:1140`).

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | A PoW reward claim is published through Blend once and never retried, so a lost claim forfeits its rewards when the window closes | Economic / Incentive | Medium | Medium | Open |
| LB-002 | Spec deviation: `abstain_on_failure` disables delivery detection as well as the direct broadcast, so a lost transaction is indistinguishable from a delivered one | Auditing and Logging | Low | Low | Open |
| LB-003 | A proposer applies its own block before publishing it, so its chain view does not show whether the proposal was delivered | Auditing and Logging | Informational | Low | Open |

### LB-001 · A PoW reward claim is published through Blend once and never retried, so a lost claim forfeits its rewards when the window closes

| | |
|---|---|
| Severity | Medium |
| Difficulty | Medium |
| Category | Economic / Incentive |
| Target | `services/pow/src/service.rs:1046-1076` (`claim_ready_rewards`), `:1103-1115` (`prune_expired_tickets`), `:1188-1221` (`prune_settled_pending`), `:1502-1514` (`publish_reward_claim`) |
| Status | Open |

**Description**

`claim_ready_rewards` builds one claim transaction from the ready tickets, publishes it through Blend, and on a successful *hand-over to the Blend service* moves the claimed tickets to the pending set:

```rust
// services/pow/src/service.rs:1055-1076
publish_reward_claim(blend_api, signed_tx).await?;          // Ok(()) == the relay accepted the message
…
state.ready_to_claim = remaining;
info!(target: LOG_TARGET, "Claimed {} PoW reward(s); {} still ready", …);
state.pending_to_claim.extend(claimed);
```

`BlendServiceApi::publish` is fire-and-forget (`services/blend/src/api.rs:77-86`): `Ok(())` means the message reached the Blend service's inbox, not that it was sent, delivered or accepted by a mempool. From then on a pending ticket can leave `pending_to_claim` in exactly two ways: a `PoWRewardClaimed` event for its nullifier arrives in a processed block (`prune_settled_pending`), or its reward window closes and `prune_expired_tickets` drops it. There is no third path. Nothing moves a ticket from `pending_to_claim` back to `ready_to_claim`, the comment on the field says as much ("retained until their reward window closes", `:307-309`), and the claim functions read `ready_to_claim` only.

The window is `slot_window = 300` slots in all three shipped templates (`deployment/ceremony/genesis/*/deployment-template.yaml:80`), against a delivery deadline of `T_M = ß · (Δmax + η)` rounds: 5 rounds with the shipped `num_blend_layers: 1`, 15 with the spec's `ß = 3`. A retry after `T_M` fits in the window many times over. The service does not make one.

Whether the claim survives the network therefore rests entirely on the Blend service's direct-broadcast fallback, and that fallback is absent in two cases:

1. The operator set `abstain_on_failure`. The specification recommends exactly this for "nodes that value privacy above block rewards" (`blend-protocol.md` › Direct Broadcast). Such an operator accepts losing a block whose proposal is dropped. A reward claim is different: it could be resent through Blend at no privacy cost, and it is lost instead.
2. Under the default configuration, after the delivery observer has lagged once. `stop_observing_on_lag` ends the observation stream, the detector's stream ends with it, and the direct broadcast stays off "for the rest of this run" (`core/dispatcher/libp2p.rs:101-118`; #276 LB-002 and LB-004). From that point a default node loses claims the same way, with no metric or API state to show it.

A transaction travels as a single copy (`data_replication_factor` applies to proposals only, #279 report), so every blend node on its path is a single point of failure. With a fraction `f` of selected nodes unresponsive or malicious, a claim is lost with probability `f` at the shipped `ß = 1` and `1 − (1 − f)³` at `ß = 3`: 27% at `f = 0.1`, 87.5% at the `f = 0.5` the spec's minimal network size is dimensioned for (› Minimal Network Size).

The operator cannot see any of this. The log shows `Claimed N PoW reward(s)` at `info` when the claim is handed over, which reads as success. A settled claim logs `Retired N settled PoW claim(s)`; a lost one eventually logs `Pruned N expired PoW ticket(s)`, a line that does not distinguish tickets that were never claimed from claims that were lost. `GET /pow/rewards/claimable` lists ready tickets only, so pending ones are invisible.

**Exploit scenario**

A miner runs with `--blend-abstain-on-failure`. It mines two tickets and the auto-claim tick publishes one claim transaction carrying both. The transaction's single Blend copy is addressed to a core node that is offline. The miner's log says `Claimed 2 PoW reward(s); 0 still ready`. No mempool ever sees the transaction. 300 slots after the tickets' anchor blocks, the next call to `prune_expired_tickets` logs `Pruned 2 expired PoW ticket(s)`. Two epoch rewards stay in the pool.

The same happens without an outage if the exit node is hostile. The last blend node on the path decapsulates the payload and sees that it is a reward claim. An adversary that runs core nodes and also mines has a reason to discard it: an unclaimed reward stays in `R_PoW`, which funds later claims, its own included. Dropping a payload at the exit leaves no evidence; the blending token for the layer was already collected. Against a victim with the fallback on, the drop only costs the victim its unlinkability for that transaction, which is the trade the spec documents. Against an abstaining victim, or a default node whose observer has lagged, it costs the reward.

**Recommendation**

- *Short term*: give a pending ticket a deadline. If no `PoWRewardClaimed` event for it has been seen some margin after `T_M` (a small multiple of the expected inclusion time, well inside `slot_window`), move it back to `ready_to_claim` so the next claim run rebuilds and republishes it. The ledger's `DoubleClaimed` check makes a redundant retry harmless. Log the move at `warn` with the ticket count.
- *Short term*: report pending tickets, with their remaining window, from `GET /pow/rewards/claimable`, and word the hand-over log line as "published", not "claimed".
- *Long term*: have the Blend service report the fate of a locally submitted payload to its caller (delivered, unconfirmed after `T_M`, revealed by direct broadcast) instead of a fire-and-forget `publish`, so every caller can decide for itself whether to retry. LB-002's recommendation is the Blend half of this.

**References**: `blend-protocol.md` › Failure Detection and Reaction, › Proof of Work Quota, › Minimal Network Size; #279 report (S-002, single-copy transactions); #276 report LB-002, LB-004 (the direct broadcast turning itself off).

### LB-002 · Spec deviation: `abstain_on_failure` disables delivery detection as well as the direct broadcast, so a lost transaction is indistinguishable from a delivered one

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Auditing and Logging |
| Target | `services/blend/src/core/mod.rs:465-473`, `services/blend/src/edge/mod.rs:362-372`, `services/blend/src/delivery/mod.rs:17-28` (`next_undelivered_messages`), `services/api/src/http/blend.rs:57-119` |
| Status | Open |

**Description**

The specification separates two things. › Detection is unconditional: "If the sender does not observe that payload on the broadcasting channel within the message traversal time `T_M` … it **must** treat the message as lost." › Direct Broadcast is the reaction, and only the reaction is optional: "Nodes that value privacy above block rewards should disable this fallback and abstain from participation until Blend recovers."

The code implements both with one `Option`. `abstain_on_failure: true` makes the detector `None`, which removes the bookkeeping, the subscription to the broadcasting channel and the deadline clock along with the broadcast. The edge service's comment states the intent: "records nothing, watches nothing and can reveal nothing". Two consequences follow.

1. A node with the flag set never learns that a message was lost, so it has nothing to treat as lost. Item 1 of section 3.1 lists what is left: after release there is no log line, no metric and no query result that differs between a delivered and a lost transaction. `blend_transaction` returns the hash on acceptance ("accepted for blending, not that it was sent"), and `blend_pending_transactions` stops reporting the transaction once it is encapsulated. The experiment in 3.2 shows both.
2. The node never "abstains from participation until Blend recovers", because it cannot tell that Blend has failed or recovered. It keeps sending proposals and transactions into a network that may be dropping all of them. The flag's name describes behaviour the code does not have.

The code is the side that is wrong. Watching the broadcasting channel reveals nothing to the network; only the dispatch does. The lag hazard that justifies a one-way off switch for the *broadcast* (#276 LB-001) does not apply to a detector that never broadcasts. The spec could be clearer about what "abstain from participation" requires of a node (S-002).

**Exploit scenario**

Not an attack on its own; it removes the sender's ability to notice one. An external client submits a transfer through `POST /blend/transactions/disperse` on its own abstaining node and receives the transaction hash. The exit node on the path drops the payload. The client polls `GET /blend/transactions/pending`, sees an empty list, and reads that as "sent". The transaction never reaches a mempool, and nothing on the node will ever say so. A client that instead waits for the hash on chain cannot tell a loss from slow inclusion and has no guidance on when to resubmit. LB-001 is the same gap with a caller that loses money to it.

**Recommendation**

- *Short term*: split the switch. Keep the detector running when `abstain_on_failure` is set, and make only `broadcast_undelivered_messages` conditional. On expiry, log at `warn` that the payload was not delivered within `T_M` and was not revealed, and count it in a payload-typed metric next to `blend_payloads_bypassed_total` (for example `blend_payloads_undelivered_total`).
- *Short term*: extend the pending-transactions reply with the released-but-unconfirmed state, so a caller sees `waiting for PoW`, `released, unconfirmed` (with rounds since release) and, after `T_M`, `undelivered`. Keep the record to the transaction hash, in memory, with a bounded lifetime.
- *Long term*: in observation-only mode the detector can keep watching past a lag, since a missed observation can no longer cause a reveal; report the affected entries as `unknown` instead of ending the stream. Decide what "abstain from participation" means for the node (for example, hold locally originated transactions while recent payloads are going undelivered) and implement it behind the same flag.

**References**: `blend-protocol.md` › Failure Detection and Reaction › Detection, › Direct Broadcast; #279 report S-002; #276 report LB-001, LB-004.

### LB-003 · A proposer applies its own block before publishing it, so its chain view does not show whether the proposal was delivered

| | |
|---|---|
| Severity | Informational |
| Difficulty | Low |
| Category | Auditing and Logging |
| Target | `services/chain/chain-leader/src/lib.rs:706-729` (`apply_and_publish_block_proposal`), `services/chain/chain-network/src/lib.rs:803-807` (`note_received_proposal`) |
| Status | Open |

**Description**

Item 4 of the issue asks whether the chain view is a sufficient delivery signal for a block proposal under `abstain_on_failure`; the #279 report assumed it was ("a proposal sender in the same configuration at least learns from the chain"). It is not a direct signal. The leader applies its block to its own chain first and only then hands the proposal to Blend:

```rust
// services/chain/chain-leader/src/lib.rs:713-722
if let Err(e) = chain_network_api.apply_block_and_reconcile_mempool(block.clone()).await { …; return; }
blend_adapter.publish_proposal(block.to_proposal()).await;
```

The proposer's tip therefore contains the block whether or not any other node ever receives it. What the detector uses as the delivery signal is the proposal coming back over gossip (`observe_block_proposals` subscribes to `SubscribeToProposals`), and `note_received_proposal` publishes to that channel only when it has a subscriber. An abstaining node has none, so the echo is received and dropped; its only residue is the generic counter `consensus_proposals_ignored_total{reason="already_processed", source="network"}` (`chain-network/src/lib.rs`, `handle_incoming_proposal`), which also counts any other proposal for a block the node already holds. `metrics::consensus_proposals_created_local` counts the hand-over, not the delivery.

The proposer does learn indirectly: if the proposal was lost, no other node builds on the block, a competing branch overtakes it, and the block is orphaned. That signal arrives blocks later, reads the same as losing an ordinary fork race, and is not logged as a delivery failure. Proposals are better protected than transactions in the first place, because they are sent as `1 + R_D` copies (two on devnet and testnet), so the loss rate is lower; the observability is the same.

**Exploit scenario**

None beyond the loss itself, which the operator accepted by setting the flag. The impact is that an abstaining leader cannot measure how many of its slots Blend is costing it, which is the number it needs to decide whether to keep the flag set.

**Recommendation**

- *Short term*: the detector split recommended in LB-002 covers proposals with no extra work, since the detector already tracks both payload kinds. Expose the per-type undelivered count as a metric.
- *Long term*: document, next to the flag, that a lost proposal shows up only as an orphaned own block, and that a lost transaction does not show up at all until LB-002 is addressed.

**References**: `blend-protocol.md` › Failure Detection and Reaction; #279 report S-002; `services/blend/src/edge/tests/mod.rs:170-207` (the existing test, whose doc comment already notes that such a node "loses the slots the Blend network drops").

## 5. Suggestions (non-security)

### S-001 · An edge node keeps queued transactions in memory only

| | |
|---|---|
| Target | `services/blend/src/edge/mod.rs:360`, `:411-413` |

A core node writes a queued transaction to its recovery state (`core/mod.rs:1476-1488`) so a restart while waiting for a PoW solution does not lose it. The edge service keeps `PendingTransactions` in a local variable, so the same restart drops every queued transaction, again without a trace. Persisting the edge queue, or documenting that a restart clears it, would make the two services consistent. The wait for a solution can be long when `d_blend` is high, which is when a restart is most likely to fall inside it.

### S-002 · `blend-protocol.md`: say what detection an abstaining node still performs, and what it abstains from

| | |
|---|---|
| Target | `blend-protocol.md` › Failure Detection and Reaction › Direct Broadcast |

The sentence "Nodes that value privacy above block rewards should disable this fallback and abstain from participation until Blend recovers" leaves three things open that an implementation needs: (1) that › Detection still applies to such a node, which the "must" in › Detection implies but the implementation did not take from it; (2) what participation is withheld (proposing, sending transactions, or both) and what tells the node that Blend has recovered; (3) that the sentence speaks of block rewards only, although since v1.4.0 a lost data message may be a transaction, where the alternative to revealing is resending through Blend, not abstaining. A sentence allowing, or recommending, a Blend retry for a transaction after `T_M` would give callers such as the PoW claim path a specified behaviour. To be raised in logos-lips; not done from this audit, which writes to this repository only.

### S-003 · The hand-over log line of a reward claim reads as a completed claim

| | |
|---|---|
| Target | `services/pow/src/service.rs:1070-1075` |

`Claimed {} PoW reward(s); {} still ready` is logged when the transaction has been handed to the Blend service. `Published a claim for {} PoW reward(s)` would match what happened; the `Retired … settled PoW claim(s)` line is the one that reports a completed claim. Part of LB-001's recommendation, listed separately because it is a one-line change.

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
