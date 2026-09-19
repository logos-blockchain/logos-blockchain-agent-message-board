# Audit Report — Blend old-epoch recovery state, second pass

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/278`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `54328b50eb47ab4b8fd8044f8db5fbcebed718c0` — component(s): `services/blend/src/core` (`mod.rs`, `state.rs`, `delivery.rs`, `epoch_stages/`), `blend/scheduling`, `blend/network/src/core/with_core/behaviour`, `services/blend/src/orchestrator`, `services/sdp` (activity submission only)
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `blend-protocol.md`; by section: `bedrock-service-declaration-protocol.md` (› Message Timing, › Active Message, › Active)
Date: `2026-09-14` — author: `Claude Fable 5.1 (Claude Code)` — status: `final`

---

## 1. Summary

- Overall assessment: the old-epoch exclusion from the recovery state is deliberate and consistent for the transition period, but two adjacent recovery boundaries are not: fully decapsulated payloads that need no epoch context are thrown away at rotation and at stale-state start-up, and the retiring stage drops the recovery state entirely, so a restart in that window forfeits the epoch's Blend reward.
- Findings: `0` critical · `0` high · `0` medium · `2` low · `1` informational
- Key themes: recovery-state lifetime at epoch boundaries, retirement as an untracked path, activity-proof timing.
- Must-fix before launch: none. LB-002 costs an operator one epoch's reward on an unlucky restart and should be fixed before mainnet rewards are live.

This is the second report on #278. The first (PR #295, corrected by PR #569) established that old-epoch processed messages are scheduled without being recorded and that the old-epoch paths have no duplicate check; both are now tracked as #504 and #501. This pass answers the five checklist items one by one at the current head, which is byte-identical to the previously audited `a805329f8` in every Blend path (`git diff a805329f8..54328b50 -- services/blend blend` is empty), and reports what the first pass did not cover: the rotation code itself, the retiring stage, and the start-up recovery branch.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/blend/src/core/mod.rs` | start-up recovery (`initialize`, L680-L830), rotation (`handle_epoch_event`, L1694-L1872), transition expiry (L1208-L1228), retirement (`retire`, L1625-L1683), incoming-message handlers (L2100-L2355), release rounds (L2365-L2616), activity-proof submission (L1917-L1928, L2679-L2718) |
| `services/blend/src/core/state.rs` | the persisted fields and `with_epoch` |
| `services/blend/src/core/epoch_stages/{transitioning,retiring}.rs` | what outlives a rotation and where it lives |
| `services/blend/src/core/delivery.rs` | what the failure detector logs at transition expiry |
| `blend/scheduling/src/message_scheduler/mod.rs`, `release_delayer.rs` | scheduler rotation and the old-epoch scheduler's queue |
| `blend/network/src/core/with_core/behaviour/{mod,utils,message_cache,old_epoch}.rs` | when a message identifier is marked processed relative to service delivery |
| `services/blend/src/orchestrator/instance.rs`, `services/blend/src/mode.rs` | which service runs when the node is no longer core |
| `services/blend/src/edge/mod.rs` | recovery state of the edge service (item 5 only) |
| `services/sdp/src/lib.rs` | whether `PostActivity` is gated on the transition period (LB-003 only) |

**Out of scope**

Message encapsulation and decapsulation correctness, proof verification, the KMS, the libp2p transport, consensus, the ledger side of `SDP_ACTIVE`, and the mempool. `libp2p`, `serde`, `overwatch` (including its file-based state operator) and `rand_chacha` are assumed correct. No circuits were in scope.

**Assumptions**

The node is not compromised. The operator can restart a node at any time; an adversary cannot. Blend parameters are the spec's (`TP = 30` rounds, `T_M = 15`, round = 1 s) unless the deployment sets otherwise. The overwatch state operator writes every `StateUpdater::update` to disk before the process can die.

## 3. Method

- Manual review of the paths above, working through issue `#278` under parent `#12`, with the first-pass report, its correction PR #569, and the canonical findings #501 and #504 open alongside.
- Spec conformance against `blend-protocol.md` › Transition Period, › Processing, › Releasing, › Failure Detection and Reaction › Detection, › Rewarding › Active Message and › Rewarding Distribution Logic; and `bedrock-service-declaration-protocol.md` › Message Timing › Active message.
- Tracker search for prior coverage before rating anything as new: failure-detector persistence is #540/#415/#319; retirement shutdown grace is #416/#541; the SDP-side reward forfeiture on withdrawal is #368; none covers the retiring stage's token collector or the start-up submission timing.
- Automated tooling: none. Dynamic testing: none. The Blend crates at this head are byte-identical to `a805329f8`, where the first pass ran `test_handle_incoming_blend_message`, `the_previous_epoch_keeps_releasing_under_its_own_epoch` and `duplicate_message_from_old_epoch_after_epoch_rotation_is_suppressed` and reported them passing; those results were not re-run here and no test exercises a restart, so every conclusion below is code-path analysis.

**Checklist items, verified**

1. *Is the exclusion deliberate; could an old-epoch entry be restored?* Deliberate, and stated in three places: `blend/scheduling/src/message_scheduler/mod.rs:L191-L198` ("whatever is still queued when the transition period expires is dropped with them"), `L318-L325`, and `services/blend/src/core/mod.rs:L2499-L2504`. At rotation the old `ServiceState` is destructured and its `unsent_processed_messages` and `unsent_data_messages` are discarded (`mod.rs:L1749-L1758`), and the new state starts empty (`ServiceState::with_epoch`, `state.rs:L221-L240`, called at `mod.rs:L1825-L1832`). The premise that a saved entry "would need the old epoch's processor to be rebuilt" holds only for `ProcessedMessage::Encapsulated`; the `Decapsulated` variant is released through `PayloadDispatcher::dispatch(payload)` (`mod.rs:L2607-L2609`, `dispatcher/mod.rs:L41`), which takes no epoch and needs no old peers. That is LB-001.
2. *What is actually lost.* Confirmed against `blend/network/src/core/with_core/behaviour/utils.rs:L108-L116`: a copy whose identifier is already in the cache is dropped before signature or proof checks. The identifier is entered into the cache the moment its `PoQ` verifies (`behaviour/mod.rs:L961-L976`, into the old epoch's own cache when it verified against the old epoch, `old_epoch.rs:L183-L187`), which is before the event reaches the swarm; the swarm then forwards to peers first and reports to the service second (`backends/libp2p/swarm.rs:L462-L469`). Both caches are in memory, so a restart empties them, but no peer will resend: the spec forbids it (› Connectivity Maintenance item 10, › Relaying step 1.4) and the peers' own caches enforce it. A processed message lost from the old scheduler is therefore this node's only copy. What remains is the sender's redundancy (`data_replication_factor + 1` copies, proposals only) and the sender's failure detector (› Detection), whose own persistence gap is #540.
3. *Should the transition end log what it drops?* Nothing does. At `TransitionPeriodExpired` the transitioning loop calls `failure_detector.drop_unreleased_payloads_for_epoch` (`mod.rs:L1216-L1218`, logs at `delivery.rs:L78-L88`) and moves on (`L1219-L1222`); the `TransitioningEpoch` holding the old scheduler is dropped by `end_transition` with no count of its `release_delayer` queue or `data_messages`. The retiring loop is the same (`L1662-L1680`). With `abstain_on_failure = true` there is no failure detector at all (`L465-L473`), so even the locally generated leftovers go unlogged. See S-001.
4. *Do the old-epoch paths need the duplicate check?* Confirmed as #501 describes: `handle_decapsulated_incoming_message_from_old_epoch` (`L2272-L2276`) and the retiring path (`L2195-L2196`) schedule with no set at all, and the current-epoch check at `L2234-L2243` runs after `schedule_processed_message` (`L2333`, `L2351`) so its "Dropping" message is not true either. Nothing new to file; the fix belongs to #501 and must precede the scheduling call on all three paths.
5. *Interaction with #172.* The edge service declares `NoState` (`edge/mod.rs:L111`) and keeps its failure detector in memory (`L364-L372`). Both #172 and #278 rely on the same backstop, the sender's direct broadcast after `T_M`, and that backstop is itself unpersisted on both services (#540, #415, #319). They want one answer in this order: persist the failure detector first, since it is what makes every other queue loss recoverable at protocol level; then decide queue persistence, where the core's `Decapsulated` subset (LB-001) is the cheap part and the edge's proposals (#172) the expensive one.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Rotation and stale-state start-up discard fully decapsulated payloads that need no epoch context to release | Denial of Service | Low | High | Open |
| LB-002 | Retiring stage drops the recovery state, so a restart during retirement forfeits the epoch's activity proof and Blend reward | Economic / Incentive | Low | High | Open |
| LB-003 | Spec deviation: start-up recovery releases the old epoch's active message before the new epoch's transition period has elapsed | Timing | Informational | Low | Open |

### LB-001 · Rotation and stale-state start-up discard fully decapsulated payloads that need no epoch context to release

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Denial of Service |
| Target | `services/blend/src/core/mod.rs:L1749-L1758` (`handle_epoch_event`, rotation), `L725-L739` (`initialize`, stale-state branch), `L2564-L2616` (`build_futures_to_release_processed_messages`), `L785-L802` (start-up seeding) |
| Status | Open |

**Description**

`unsent_processed_messages` holds two kinds of entry (`services/blend/src/message.rs`, `ProcessedMessage`): `Encapsulated`, a message with layers left that must be published under the epoch whose `PoQ` it carries, and `Decapsulated`, a `DataPayload` whose Blend processing is finished and which only has to be handed to the payload dispatcher. The release path treats them differently:

```rust
// services/blend/src/core/mod.rs
2606   match processed_message_to_release {
2607       ProcessedMessage::Decapsulated(payload) => {
2608           payload_dispatcher.dispatch(payload).boxed()
2609       }
2610       ProcessedMessage::Encapsulated(encapsulated_message) => {
2611           backend.publish(*encapsulated_message, epoch).boxed()
2612       }
```

`dispatch` takes no epoch (`dispatcher/mod.rs:L41`). A `Decapsulated` entry is therefore releasable from any epoch, by any scheduler, with no old-epoch processor and no old-epoch peers. The recovery code does not distinguish the two and discards both at two points:

```rust
// rotation: the whole old state is destructured, the message sets go to `_`
1749   let (
1750       _,
1751       _,
1752       _,
1753       _,
1754       pending_transactions,
1755       current_epoch_blending_token_collector,
1756       _,
1757       state_updater,
1758   ) = current_recovery_checkpoint.into_components();
```

```rust
// start-up with a state saved under a previous epoch: same fields, same `_`
731    let (_, _, _, _, pending_transactions, token_collector, ..) =
732        state.into_components();
```

The rotation case is self-consistent while the process lives: the messages stay in the old scheduler (`rotate_epoch`, `blend/scheduling/src/message_scheduler/mod.rs:L183-L201`) and are released by `handle_release_round_for_old_epoch` with a `None` state updater (`mod.rs:L2520-L2527`). It stops being consistent at the first restart inside the transition period: the matching-epoch state (`L689-L697`) has no entry for them, and this is the #504 class. The start-up case is the one #504 does not cover: a node that dies with fully decapsulated payloads pending, in the last rounds of an epoch or with downtime spanning the boundary, comes back to a state one epoch old, takes the `is_previous_epoch` branch to rescue the token collector (`L729-L736`), and drops the payloads in the same statement, although seeding them into the new scheduler is exactly what the matching-epoch branch does two lines later (`L785-L802`).

The first-pass correction (PR #569) and the follow-up #296 say that "persisting only `{epoch, ProcessedMessage}` is not sufficient". That is true of the `Encapsulated` variant and not of the `Decapsulated` one, which already is persisted and is dropped by the two destructurings above.

**Exploit scenario**

Not attacker-triggerable. A core node fully decapsulates a transaction addressed to it as the last hop, queues it with a release delay of up to `Δ_max = 3` rounds, and crashes; it is back up in the next epoch. The stale state holds the payload; L731 discards it. The network cache of every neighbour still holds the identifier (item 2 above), so no copy is resent. The transaction reaches the mempool only if the sender's failure detector fires after `T_M` (and only if that sender did not itself restart, #540). For a block proposal the loss is usually moot because the slot has passed, but a proposal from the last seconds of the epoch is exactly the one that lands in this window.

**Recommendation**

- *Short term*: in the stale-state branch (`L725-L739`), when `is_previous_epoch` holds, keep the `Decapsulated` entries of `unsent_processed_messages` and pass them to `new_with_initial_messages` alongside the current-epoch ones, the way the token collector is kept. Add the crash/restart test #296 asks for with this exact shape: a fully decapsulated payload saved under epoch `e`, a restart at epoch `e+1`, one dispatch.
- *Long term*: split the persisted set by variant, so the recovery policy chosen under #296 can be stated per variant: `Decapsulated` is epoch-agnostic and durable; `Encapsulated` is bound to its epoch's peers and dropped with them. At rotation, keep the `Decapsulated` entries in the new state and have the old-epoch release round remove them (it currently passes `None` at `L2524`), so the two never diverge.

**References**: `blend-protocol.md` › Transition Period, › Broadcasting (a payload leaving Blend has no epoch of its own); #504, #296, #540.

### LB-002 · Retiring stage drops the recovery state, so a restart during retirement forfeits the epoch's activity proof and Blend reward

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Economic / Incentive |
| Target | `services/blend/src/core/mod.rs:L1846-L1870` (`handle_epoch_event`, `NotCore` branch), `L1649-L1683` (`retire`), `L648-L652` (`initialize`, first-epoch assertion), `L723-L739` (stale-state age check); `services/blend/src/core/epoch_stages/retiring.rs:L9-L19` |
| Status | Open |

**Description**

While the node stays core across a rotation, the old epoch's blending tokens are part of the recovery state: the new state is created with `Some(old_epoch_blending_token_collector)` (`L1825-L1832`), every old-epoch decapsulation saves into it (`L2278-L2282`), and a restart inside the transition period finds it and submits the activity proof (`L689-L697`, `L756-L760`). When the node leaves core mode, none of that happens. The `NotCore` branch destructures the state and drops the `state_updater` without creating or saving a successor:

```rust
// services/blend/src/core/mod.rs
1848   let old_cryptographic_processor = current_cryptographic_processor.rotate_epoch();
1849   let (_, _, _, _, _, current_epoch_blending_token_collector, _, _) =
1850       current_recovery_checkpoint.into_components();
...
1859   let (_, old_epoch_blending_token_collector) =
1860       current_epoch_blending_token_collector.rotate_epoch(&new_reward_epoch_info);
1861   HandleEpochEventOutput::Retiring {
1862       retiring_epoch: Box::new(RetiringEpoch::new(
1863           TransitioningEpoch::new(
1864               old_cryptographic_processor,
1865               current_scheduler.consume(),
1866           ),
1867           old_epoch_blending_token_collector,
1868       )),
1869   }
```

`retiring.rs:L12-L15` says so in as many words: "While the service keeps running the tokens live in the recovery state, so `TransitioningEpoch` alone is what the event loop carries" — in retirement they do not. The retiring loop collects tokens into the in-memory collector (`L1653-L1654`, `handle_incoming_blend_message_from_old_epoch`, `L2195-L2199`) and submits the proof once, at `TransitionPeriodExpired` (`L1669`).

The file on disk is left at the last save before rotation: `last_seen_epoch = e`, the epoch-`e` collector as `current_epoch_token_collector`. Nothing can read it back. The core service is not started when the node is not core (`mode.rs:L49-L96`, `orchestrator/instance.rs:L136-L179`), and if it were, it panics when its first epoch is `NotCore`:

```rust
648   let CoreEpochStateInfo::Core(core_epoch_info) =
649       epoch_info.unwrap_or_else(|error| panic!("{error}"))
650   else {
651       panic!("First retrieved epoch for Blend core startup must be available.");
652   };
```

The edge service has no state (`edge/mod.rs:L111`). If the node becomes core again it is at epoch `e+2` or later, and a state two or more epochs old is dropped as "past submitting for" (`L723-L724`, `L729-L736`). The spec requires the active message for epoch `e` to be included in a block of epoch `e+1` (`blend-protocol.md` › Active Message), so that judgement is correct; the tokens were simply never reachable.

**Exploit scenario**

Not attacker-triggerable. A core node's declaration expires or is withdrawn, or it drops out of the membership snapshot, so epoch `e+1` starts with it as an edge node. Within the 30-round retirement window the process restarts, for any reason. The epoch-`e` collector, holding a full epoch of blending tokens, is gone: the on-disk copy is never loaded, the in-memory copy died with the process. No active message for epoch `e` is sent, and the node receives neither the base nor the premium reward for its last epoch of service, `R = I / (B + P)` per `blend-protocol.md` › Reward Calculation. The tokens collected during retirement itself are lost in every restart, but they are at most 30 rounds' worth.

The same window is already fragile in a second way: the orchestrator kills the retiring service `CORE_SHUTDOWN_GRACE = 3 s` after its own `TransitionPeriodExpired` (#416, #541). That finding concerns delivery deadlines; the activity proof is submitted at `L1669` before the drain and is not affected by the kill unless the two expiry events are skewed by more than 3 s.

**Recommendation**

- *Short term*: in the `NotCore` branch, keep the `state_updater` and save a state for epoch `e+1` with an empty current collector and `Some(old_epoch_blending_token_collector)`, exactly as the `Core` branch does; have the retiring loop write collected tokens through that state. Then give the on-disk state a reader: on start-up, if a saved state is one epoch behind and holds an `old_epoch_token_collector`, compute and submit its proof before deciding whether core mode can start (the existing `L756-L760` does this, but only after the `Core` assertion at `L648-L652`; the submission has to move ahead of it, or the orchestrator has to run it).
- *Long term*: make the activity proof's lifecycle independent of the core service's: hand the `OldEpochBlendingTokenCollector` (it is `Serialize`, `blend/message/src/reward/mod.rs:L108-L112`) to the SDP service, or to a small persisted "pending activity proofs" store, at rotation, and let that owner submit at transition expiry whatever the Blend mode is. Add a test: rotate to `NotCore`, drop the service, recreate it from the saved state, assert one `PostActivity`.

**References**: `blend-protocol.md` › Rewarding › Active Message ("must be included in a block of epoch `e+1`"), › Transition Period; `bedrock-service-declaration-protocol.md` › Message Timing › Active message; #416, #541 (retirement grace), #368 (the SDP-side forfeiture on withdrawal, a different path to the same loss).

### LB-003 · Spec deviation: start-up recovery releases the old epoch's active message before the new epoch's transition period has elapsed

| | |
|---|---|
| Severity | Informational |
| Difficulty | Low |
| Category | Timing |
| Target | `services/blend/src/core/mod.rs:L756-L761` (`initialize`), `L1917-L1928` (`compute_and_submit_activity_proof`), `L2679-L2697` (`submit_activity_proof`); `services/sdp/src/lib.rs:L569-L604` (`handle_post_activity`), `L611-L723` (`submit_activity`) |
| Status | Open |

**Description**

`blend-protocol.md` › Rewarding Distribution Logic, step 2, reads: "The Active Message must not be released before the epoch transition period of epoch `e+1` has elapsed". The running service complies: `complete_transition_period` submits at `TransitionPeriodExpired` (`L1897-L1901`, `L1212-L1222`), as does the retiring loop (`L1669`). The start-up path does not:

```rust
// services/blend/src/core/mod.rs
756   let mut state_updater = current_recovery_checkpoint.start_updating();
757   if let Some(old_epoch_token_collector) = state_updater.clear_old_epoch_token_collector() {
758       tracing::debug!(target: LOG_TARGET, "Old epoch token collector loaded. Computing activity proof");
759       compute_and_submit_activity_proof(old_epoch_token_collector, sdp_relay).await;
760   }
```

This runs as soon as the first epoch is ready, whatever the slot. A node that restarts in the first 30 rounds of epoch `e+1` with a state from epoch `e` (or a state from `e+1` still holding the old collector) sends `SdpMessage::PostActivity` at once (`L2692-L2696`). The SDP service builds and posts the transaction immediately and does no timing check of its own (`sdp/lib.rs:L586-L589`, `L657-L723`); `chain_epoch` and `chain_slot` at `L624-L628` are fetched for logging only.

Which side is wrong: the code, as written; but the deviation is harmless in effect and the spec gives no rationale for the rule. The only rationale visible in the surrounding text is that a node keeps collecting old-epoch tokens during the transition period and should submit its best one; after a restart the node opens no old-epoch connections (`L752-L755`) and can collect nothing more, so waiting gains it nothing. The ledger's own window is the whole of epoch `e+1` (› Active Message: "must be included in a block of epoch `e+1`"), so the early message is accepted.

**Exploit scenario**

None. The effect is a message sent up to 30 rounds earlier than the specification allows, by a node that has just restarted, with the same content it would have sent later.

**Recommendation**

- *Short term*: either defer the start-up submission to the epoch stream's `TransitionPeriodExpired` (already available to the service, `L660-L665`), which also removes a code path that behaves differently from the running one; or, if early release is judged acceptable, raise it upstream so the spec's "must not" becomes "should not, unless no further tokens can be collected".
- *Long term*: give the spec rule a stated reason (see S-003), so that implementers can tell whether a shortcut like this one is safe.

**References**: `blend-protocol.md` › Rewarding Distribution Logic step 2, › Active Message; `bedrock-service-declaration-protocol.md` › Active.

## 5. Suggestions (non-security)

### S-001 · Count and log what the old-epoch scheduler drops at transition expiry

Both expiry sites (`mod.rs:L1212-L1222`, `L1662-L1680`) drop the `TransitioningEpoch` without inspecting it. `EpochProcessedMessageDelayer::unreleased_messages()` already exists (`blend/scheduling/src/release_delayer.rs:L81-L83`) but `OldEpochMessageScheduler::release_delayer()` is `cfg(test)` only (`message_scheduler/mod.rs:L373-L379`); a non-test `pending_counts() -> (processed, data)` on the old scheduler would let both sites log one line at `debug` with the two counts, matching `drop_unreleased_payloads_for_epoch` (`delivery.rs:L78-L88`). Do it independently of the failure detector, which is absent under `abstain_on_failure` (`L465-L473`). This is the first-pass S-001 with the exact hooks named.

### S-002 · Document the old-epoch exception on the persisted fields

`unsent_processed_messages` and `unsent_data_messages` (`state.rs:L108-L109`) carry no doc comment, and `with_epoch` (`L211-L220`) explains why `pending_transactions` survives a rotation but not why the two message sets do not. One sentence on each, pointing at the scheduler comment (`message_scheduler/mod.rs:L191-L198`), closes the gap #504 calls out and makes LB-001's variant distinction visible to the next reader.

### S-003 · Spec: state why the active message must wait for the transition period

`blend-protocol.md` › Rewarding Distribution Logic step 2 is a bare "must not" with no reason given, and › Active Message says only that the message "is constructed after the current epoch, when the next epoch randomness is known". If the reason is token collection during the transition period, say so and allow an exception when no further tokens can arrive; if it is something else (for example, not revealing through the timing of the message that a node has stopped serving), say that, because it changes whether LB-003 is a bug.
