# Audit Report — Blend path selection, scheduling delays, and shutdown behaviour

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/58`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `blend/scheduling, blend/membership, blend/proofs/selection, blend/provers, services/blend/src/core, services/blend/src/delivery, services/blend/src/orchestrator, nodes/node/binary/src/config/blend, deployment/ceremony/genesis`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md, overview-cryptoeconomics.md, blend-protocol.md` (in full); `message-encapsulation.md` §Keys and Proof Generation, §Message Encapsulation; `message-formatting.md`, `payload-formatting.md` (in full, short); `proof-of-quota.md` §Constraints, §Pseudocode; `analysis-queuing-system-in-the-mix-node.md` (in full)
Date: `2026-09-11` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: the scheduling and selection code follows the specification closely in structure (uniform-subset cover schedule, queue-independent release rounds, verifiable node selection, shuffled releases), but the two random processes that the delaying design depends on are driven from one RNG state, and the shipped deployment parameters remove the timing obfuscation entirely.
- Findings: 0 critical · 0 high · 1 medium · 6 low · 0 informational
- Key themes: "correlated randomness between cover schedule and release delay", "deployment parameters below the spec's privacy floor", "locally originated payloads lost at restart, rotation and retirement without the direct-broadcast fallback"
- Must-fix before launch: LB-002 (deployment templates set `maximum_release_delay_in_rounds: 1`, `num_blend_layers: 1`, `minimum_network_size: 2`), LB-001 (independent RNG per scheduler component).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `blend/scheduling/src/{cover_traffic.rs, release_delayer.rs, epoch.rs, message_scheduler/mod.rs, message_scheduler/round_info.rs}` | cover schedule, release delaying, RNG handling, epoch rotation of schedulers |
| `blend/membership/src/lib.rs` | node index table, random peer choice |
| `blend/proofs/src/selection/mod.rs` | proof of selection index derivation and verification |
| `blend/crypto/src/lib.rs` | CSPRNG construction used by the selection |
| `blend/provers/src/provers/{core,leader,pow,core_and_leader,core_leader_and_pow}/mod.rs`, `blend/provers/src/crypto/core_and_leader/send.rs` | which proofs back a message, in which order, key-index resumption |
| `services/blend/src/core/{mod.rs, scheduler.rs, state.rs, delivery.rs, settings.rs, processor.rs}`, `services/blend/src/core/epoch_stages/*`, `services/blend/src/core/backends/libp2p/mod.rs` | event loop, release rounds, recovery state, epoch transition, retirement, backend shutdown |
| `services/blend/src/{pending.rs, delivery/mod.rs, delivery/failure_detection.rs, settings/mod.rs, settings/timing.rs, membership/chain.rs, membership/service.rs, mode.rs}` | local message queues, failure detection deadlines, epoch stream, membership ordering |
| `services/blend/src/orchestrator/{instance.rs, on_demand.rs}` | mode switching, stop and kill grace periods |
| `nodes/node/binary/src/config/blend/{deployment.rs, mod.rs}`, `core/src/blend/mod.rs` | how the scheduler parameters and the core quota are derived from deployment settings |
| `deployment/ceremony/genesis/{testnet,devnet,standalone}/deployment-template.yaml`, `nodes/node/binary/src/config/deployment/settings.yaml` | shipped values of the Blend parameters |
| `blend/network/src/core/with_core/behaviour/{message_cache.rs, utils.rs}`, `services/chain/chain-network/src/lib.rs` | consulted only for how a neighbour treats a re-sent nullifier and what the failure detector observes |

**Out of scope**

The libp2p swarm and connection monitoring (`services/blend/src/core/backends/libp2p/swarm.rs`, `blend/network`, tracked by #13), membership derivation from SDP state (#14, #62), the edge service, the PoQ circuit and the Groth16 verifier, the blending-token collector and reward computation (#8), and the nullifier cache lifetime (#59). Third-party crates assumed correct: `rand 0.8.6`, `rand_chacha 0.3.1` (`ChaCha20Rng::from_entropy` seeds from the OS), `tokio`, `fork_stream`, `libp2p`, `overwatch` (rev `ae887f41`).

**Assumptions**

The specification at the stated commit is the reference. The host is not compromised; the recovery file on disk is readable only by the operator. Facts from issue #19 apply: the release profile has no overflow checks (the scheduler code reviewed here uses `checked_*`/`strict_*`/`saturating_*` for every arithmetic operation that matters, so no wrap was found in scope).

## 3. Method

- Manual review of the in-scope paths, working through issue `#58` (all three checklist items) with parent `#12` for context.
- Spec conformance against `blend-protocol.md` §Cover Message Schedule, §Delaying, §Releasing, §Proof of Selection, §Transition Period, §Failure Detection and Reaction, §Minimal Network Size, §Global Parameters; `message-encapsulation.md` §Keys and Proof Generation (node selection pseudocode); `proof-of-quota.md` §Constraints step 5 (nullifier derivation).
- Automated tooling run: none. The sampling arithmetic in LB-001 was verified by reading `rand 0.8.6` (`src/distributions/bernoulli.rs` `Bernoulli::sample`, `src/distributions/uniform.rs` `UniformInt::sample_single_inclusive` and `UniformFloat::sample`) from the local cargo registry.
- Dynamic testing: none.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Cover-traffic scheduler and release delayer draw from one RNG state | Privacy / Anonymity | Low | High | Open |
| LB-002 | Spec deviation: deployment templates set `Δmax = 1`, `β = 1` and a minimal network size of 2, and the node accepts them | Configuration | Medium | Low | Open |
| LB-003 | Spec deviation: a node started mid-epoch spreads its remaining cover quota over a full epoch of rounds | Privacy / Anonymity | Low | Low | Open |
| LB-004 | Spec deviation: a local proposal whose outermost layer is self-addressed removes no cover message | Privacy / Anonymity | Low | High | Open |
| LB-005 | Spec deviation: delivery-failure deadlines do not survive a restart | Denial of Service | Low | Medium | Open |
| LB-006 | Retiring core service is killed 3 s after the transition period, before its delivery deadlines are drained | Denial of Service | Low | High | Open |
| LB-007 | Block proposals still waiting for proofs at an epoch rotation are dropped without the direct-broadcast fallback | Denial of Service | Low | Medium | Open |

### LB-001 · Cover-traffic scheduler and release delayer draw from one RNG state

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Privacy / Anonymity |
| Target | `blend/scheduling/src/message_scheduler/mod.rs:L139-L159` (`EpochMessageScheduler::new`), `L199` (`rotate_epoch`); `blend/scheduling/src/cover_traffic.rs:L91-L93`, `L115-L119`; `blend/scheduling/src/release_delayer.rs:L114-L126` |
| Status | Open |

**Description**

`EpochMessageScheduler::new` receives one `ChaCha20Rng` and hands a clone of it to the cover scheduler and the original to the release delayer:

```rust
// blend/scheduling/src/message_scheduler/mod.rs
139        let cover_traffic = EpochCoverTraffic::new(
...
150            rng.clone(),
...
153        let release_delayer = EpochProcessedMessageDelayer::new(
...
157            rng,
```

`ChaCha20Rng: Clone` copies the full state, so both components start on the same keystream position. Each decision consumes exactly one `u64` under `rand 0.8.6`: `gen_bool(p)` draws `v` and returns `v < p·2^64` (`Bernoulli::sample`); `gen_range(1..=Δ)` draws `v` and returns `1 + hi(v·Δ)` (`UniformInt<u64>::sample_single_inclusive`, widening multiply, rejection probability ≈ 2^-63); `gen_range(0f64..=1f64)` draws one `u64`. Hence the k-th value the cover scheduler consumes (`cover_traffic.rs:91`, one per round tick at round k−1) equals the k-th value the delayer consumes (`release_delayer.rs:121`, the first in `new`, then one per release round):

- `delay_k = 1` ⇔ `v_k < 2^64/Δmax`;
- cover message decided at round k−1 ⇔ `v_k < p_k·2^64`, with `p_k = remaining_messages / remaining_rounds`.

With the spec parameters, `p ≈ F_C/N = 1/N` per round (`core/src/blend/mod.rs:12-36` gives `Q_C = ⌈E·F_C·β/N⌉`, and `mod.rs:148` schedules `Q_C/β` messages), so `p < 1/Δmax` for every `N ≥ 4`. Then a cover emission at round k−1 implies the k-th release delay is exactly one round, and a k-th release delay of 2 or 3 rounds implies no cover message was decided at round k−1. The alignment is shifted by one whenever the skip coin flip runs (`cover_traffic.rs:118`, only while a proposal is unaccounted for) and skips no draw when `p = 1`, but it is never broken.

`rotate_epoch` (`mod.rs:199`) seeds the new epoch's scheduler from `self.release_delayer.rng().clone()`, so for the length of the transition period the old epoch's delayer (still draining), the new epoch's cover scheduler and the new epoch's delayer all continue from one state.

The spec's delaying design (`blend-protocol.md` §Delaying) relies on the release round being unpredictable from anything observable, and §Generation states that the cover and data generation processes are independent. Sharing one state ties the delayer's choices to the cover schedule.

**Exploit scenario**

An adversary neighbour of node X that can time X's processed-message releases (for example by addressing its own messages to X and recognising the next-layer public keys when X emits them) recovers a subset of X's release rounds and therefore some delays `d_j`. Every `d_j ≥ 2` tells it that X decided no cover message at round j−1, so any fresh message X emitted at that round was a data or processed message, not cover. Continuous observation would need roughly one message addressed to X per round, which the quota makes impractical (a core sender reaches X with about `3E/N²` messages per epoch), so in practice the adversary obtains sparse samples and a partial reduction of the delay entropy. The impact is a defence-in-depth loss rather than a linkage on its own, hence Low.

**Recommendation**
- *Short term*: seed each component from its own entropy: replace `rng.clone()` with `ChaCha20Rng::from_entropy()` for the cover scheduler (or take two RNGs in `new`), and reseed the new epoch's schedulers in `rotate_epoch` instead of copying the delayer's state.
- *Long term*: make the scheduler generic over a `SeedableRng` factory rather than a cloned instance, so that a shared stream cannot be reintroduced; add a test asserting that the cover decisions and the delay draws are uncorrelated for a fixed seed.

**References**: `blend-protocol.md` §Generation ("Both types of messages are created at random by independent processes"), §Delaying, §Releasing; `rand 0.8.6` `Bernoulli::sample`, `UniformInt::sample_single_inclusive`.

### LB-002 · Spec deviation: deployment templates set `Δmax = 1`, `β = 1` and a minimal network size of 2, and the node accepts them

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Configuration |
| Target | `deployment/ceremony/genesis/testnet/deployment-template.yaml:L3-L12`; `deployment/ceremony/genesis/devnet/deployment-template.yaml:L3-L12`; `nodes/node/binary/src/config/deployment/settings.yaml:L3-L12`; `nodes/node/binary/src/config/blend/deployment.rs:L97-L101`, `L133-L137`; `blend/scheduling/src/release_delayer.rs:L114-L126` |
| Status | Open |

**Description**

Every shipped deployment template, including the testnet genesis-ceremony template, sets:

```yaml
    num_blend_layers: 1
    minimum_network_size: 2
    ...
      delayer:
        maximum_release_delay_in_rounds: 1
```

The specification fixes `Δmax = 3`, `β_max = 3` (`blend-protocol.md` §Global Parameters) and a minimal network size of 32 (§Minimal Network Size). The node validates only `minimum_network_size ≥ 2` (`deployment.rs:97-101`) and `maximum_release_delay_in_rounds` as `NonZeroU64` (`deployment.rs:133-137`).

With `maximum_release_delay_in_rounds = 1`, `schedule_next_release_round_offset` (`release_delayer.rs:121`) evaluates `gen_range(1..=1)` and always returns 1: every round is a release round, and a processed message leaves the node at the tick after it arrived. The delaying step that §Delaying introduces to break the timing link between incoming and outgoing messages has no randomness left; the only remaining protection is the number of other messages the node happens to release in the same round. With `num_blend_layers = 1` a message is blended by one node, which the spec's own analysis (§Impact of the Blend Protocol on the Time to Link) rates at a time-to-link of under one epoch for a 1% stake node. With `minimum_network_size = 2` the direct-broadcast fallback of §Fallback never engages at the sizes where the spec says Blend is unsafe.

The `core_quota` derivation (`core/src/blend/mod.rs:12-36`) and `max_data_message_delay_in_rounds` (`services/blend/src/settings/mod.rs:153-168`) both follow these values, so the network is internally consistent but outside the parameter region the spec analysed.

**Exploit scenario**

A passive neighbour of a core node records, for each message it forwards to the node, the round of the next fresh message the node emits. Under `Δmax = 1` the processed message always appears exactly one round later, so the neighbour links incoming and outgoing messages whenever the node releases a single message in that round (the common case at the spec's traffic rate of about one message per round network-wide). With a single layer the linked outgoing message is the block proposal itself.

**Recommendation**
- *Short term*: set `maximum_release_delay_in_rounds: 3`, `num_blend_layers: 3` and `minimum_network_size: 32` in the testnet template, or document explicitly that the testnet does not provide the spec's anonymity guarantee.
- *Long term*: validate the deployment settings at load time against the spec's bounds (`Δmax ≥ 2`, `β_max = 3`, minimal network size 32, and `epoch_transition_period ≥ T_M`, see S-002) and refuse to start, or log at error level, when they are not met.

**References**: `blend-protocol.md` §Global Parameters, §Minimal Network Size, §Fallback, §Delaying, §Impact of the Blend Protocol on the Time to Link and Time to Infer the Stake.

### LB-003 · Spec deviation: a node started mid-epoch spreads its remaining cover quota over a full epoch of rounds

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Privacy / Anonymity |
| Target | `blend/scheduling/src/message_scheduler/mod.rs:L125-L152`; `blend/scheduling/src/cover_traffic.rs:L41-L65`, `L89-L93`; `services/blend/src/core/mod.rs:L785-L792`; `services/blend/src/membership/chain.rs:L148-L151` |
| Status | Open |

**Description**

The cover scheduler is a selection-sampling process: at every round it emits with probability `remaining_messages / remaining_rounds` (`cover_traffic.rs:91-93`), which yields a uniformly random subset of `message_count` rounds out of `rounds_per_epoch`. `remaining_rounds` is initialised from `settings.rounds_per_epoch` (`cover_traffic.rs:61`) whenever a scheduler is created (`mod.rs:139-152`), and a scheduler is created when the epoch stream yields. On a (re)start the epoch stream yields the *current* epoch at the first slot tick (`chain.rs:148-151`), so a node that starts `r` rounds into an epoch schedules `(Q_C − spent)/β` messages over `E` rounds while only `E − r` remain. Its per-round rate is `(E − r)/E` of its peers' rate, it under-uses its quota by the same factor, and its cover emissions stop at the epoch end without the tail of the schedule.

The spec requires cover messages to be "evenly distributed across the duration of an epoch" (§Cover Message Schedule) and the count to keep the node "indistinguishable from all other nodes" (§Motivations). `EpochInfo` (`message_scheduler/epoch_info.rs`) carries no notion of the current round within the epoch, so the scheduler cannot correct for it.

**Exploit scenario**

A neighbour counting fresh messages per observation window distinguishes a node that restarted early in the epoch (rate close to `1/N`) from one that restarted late (rate close to zero), and both from nodes that were up at the epoch start. This is a passive observation and also lowers the connection's measured frequency towards the "unhealthy" threshold of §Connectivity Maintenance.

**Recommendation**
- *Short term*: pass the number of rounds already elapsed in the epoch (available from the `SlotTick` the epoch stream consumes) into `EpochInfo`, and initialise `remaining_rounds` to `rounds_per_epoch − elapsed`.
- *Long term*: persist the scheduler's `remaining_messages`/`remaining_rounds` pair in the recovery state alongside `spent_core_quota`, so that a restart resumes the same schedule rather than a new one.

**References**: `blend-protocol.md` §Cover Message Schedule, §Motivations (1), §Connectivity Maintenance.

### LB-004 · Spec deviation: a local proposal whose outermost layer is self-addressed removes no cover message

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Privacy / Anonymity |
| Target | `services/blend/src/core/mod.rs:L1961-L2089` (`schedule_local_encapsulated_message`), specifically `L2006-L2021` versus `L2054-L2070` |
| Status | Open |

**Description**

`schedule_local_encapsulated_message` first tries to peel the layers of a locally encapsulated data message that are addressed to the node itself. Only when the outermost layer is *not* self-addressed does it call `scheduler.queue_data_message_and_skip_cover_message` (`mod.rs:2015`), which is what removes one future cover message per block proposal copy as §Releasing requires. When one or more outer layers are self-addressed (probability `1/N` per layer, since the selection is uniform over the membership), the remainder is scheduled as a *processed* message (`mod.rs:2070`) and no cover message is skipped. The node then emits one more fresh message in the epoch than a node that did not win the slot, which is exactly the count difference the rule exists to hide. The same asymmetry does not exist for cover messages, whose self-addressed remainder is published in the same round as the cover would have been (`mod.rs:2442-2457`).

**Exploit scenario**

None on its own. The effect is one extra emission per `N` proposals a leader sends, observable only in an aggregate count of fresh messages over an epoch. It is recorded because the rule it violates is stated as a must.

**Recommendation**
- *Short term*: call `scheduler.queue_data_message_and_skip_cover_message`'s cover-skip half (`cover_traffic.notify_new_data_message`) for block proposals in the self-decapsulation branch as well, whatever the remainder is.
- *Long term*: move the cover-skip decision to the point where a proposal copy is handed to the encapsulator, independent of how many layers the node peels off itself.

**References**: `blend-protocol.md` §Releasing ("As soon as a data message carrying a block proposal is generated, one random unreleased (future) cover message must be removed"), §Motivations (1).

### LB-005 · Spec deviation: delivery-failure deadlines do not survive a restart

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | `services/blend/src/core/mod.rs:L465-L473` (`FailureDetector::new`), `L2410-L2420`; `services/blend/src/core/scheduler.rs:L34-L43`; `services/blend/src/core/delivery.rs:L68-L72`; `services/blend/src/delivery/failure_detection.rs:L29-L36` |
| Status | Open |

**Description**

The `FailureDetector` is built in memory when the service starts and is not part of the recovery state (`state.rs:21-29` lists the persisted fields). Two classes of locally originated payloads therefore lose their deadline on a restart:

1. Messages released before the restart are removed from `unsent_data_messages` on release (`mod.rs:2412`), so the restarted service knows nothing about them; if the network fails to deliver them, nothing broadcasts them directly.
2. Messages recovered from `unsent_data_messages` are re-queued by `SchedulerWrapper::new_with_initial_messages` (`scheduler.rs:36-43`) without `mark_payload_as_encapsulated`; when they are released, `mark_encapsulated_payload_as_released` (`delivery.rs:68-72`) finds no entry and, by design, ignores the identifier. The payload is on the wire but is never watched.

§Detection requires the sender to treat the message as lost after `T_M` rounds from the release. A restart is an operational event, not an attack, so the severity is Low.

**Exploit scenario**

A leader restarts within `T_M` rounds of releasing its proposal, or restarts with a copy in `unsent_data_messages`. If the Blend network drops the message, the block is never proposed, without any log line saying so.

**Recommendation**
- *Short term*: persist the pending deadlines (payload, release round) in the recovery state and re-arm them at start; for recovered unsent messages, register the payload with the detector before re-queueing (the persisted `DataPayloadType` is enough to reconstruct the `DataPayload` only if the payload bytes are stored too, so store them).
- *Long term*: make the failure detector a persisted component with the same epoch-scoping rules as the scheduler state.

**References**: `blend-protocol.md` §Failure Detection and Reaction › Detection.

### LB-006 · Retiring core service is killed 3 s after the transition period, before its delivery deadlines are drained

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Denial of Service |
| Target | `services/blend/src/orchestrator/instance.rs:L50-L55`, `L191-L203`; `services/blend/src/core/mod.rs:L1661-L1681` (`retire`, `TransitionPeriodExpired` arm); `services/blend/src/delivery/failure_detection.rs:L111-L136` |
| Status | Open |

**Description**

When a node leaves core mode, the retiring core service keeps running for the transition period and, once `TransitionPeriodExpired` fires, calls `failure_detector.drain_pending_message_queue` (`mod.rs:1674-1678`), which waits up to `T_M` rounds for every payload released during the transition period and broadcasts the undelivered ones. The orchestrator handles its own `TransitionPeriodExpired` at the same slot and gives the service `CORE_SHUTDOWN_GRACE = 3 s` (`instance.rs:53`, `194-198`) before `stop_service` drops it. With the spec parameters `T_M = 15` rounds of one second, so any payload released in the last 12 rounds of the transition period and not delivered is never broadcast directly. With the shipped parameters (`T_M = 1·(1+2) = 3` rounds) the two windows coincide only by accident.

**Exploit scenario**

A leader whose last proposal of the epoch is released near the end of the transition period, and whose node stops being a core node in the new epoch, loses the proposal if the Blend network drops it; the direct-broadcast fallback it configured (`abstain_on_failure: false`) does not run.

**Recommendation**
- *Short term*: set the grace to at least `max_data_message_delay_in_rounds · round_duration` plus a margin, computed from the same settings the service uses.
- *Long term*: let the orchestrator wait for the service's own `Stopped` status with no fixed timeout when the previous mode is core, and kill only on a watchdog well above `T_M`.

**References**: `blend-protocol.md` §Transition Period, §Detection.

### LB-007 · Block proposals still waiting for proofs at an epoch rotation are dropped without the direct-broadcast fallback

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | `services/blend/src/core/mod.rs:L1446-L1449` (`rotate`), `L1744-L1748`; `services/blend/src/pending.rs:L104-L112`, `L159-L169`; `services/blend/src/core/delivery.rs:L74-L88` |
| Status | Open |

**Description**

`PendingProposals` is owned by the epoch (`pending.rs:104-112`) and is dropped in `rotate` (`mod.rs:1448`, the `_` of the components tuple), logging a warning for every proposal it still holds (`pending.rs:159-169`). A proposal waits there until a leadership proof is available, which depends on the secret PoL stream having delivered the winning slot (`mod.rs:1379-1409`); a proposal for one of the last slots of an epoch can therefore still be queued when the next epoch's event arrives. Encapsulated-but-unreleased proposals are likewise dropped from the failure detector at the end of the transition period (`delivery.rs:78-88`). In both cases the payload never reaches a peer, so §Detection does not cover it, and the fallback of §Direct Broadcast is not applied either: the block is lost silently apart from the log line.

The code comment explains the rotation drop by the new epoch's leadership quota, which is correct for encapsulation, but does not justify skipping the direct broadcast that the operator opted into with `abstain_on_failure: false`.

**Exploit scenario**

None required; the loss happens under normal timing for a leader elected in the last few slots of an epoch whose PoL info arrives late.

**Recommendation**
- *Short term*: when `abstain_on_failure` is false, hand the proposals dropped at rotation, and the encapsulated-but-unreleased ones dropped at the end of the transition period, to `broadcast_undelivered_messages` instead of discarding them.
- *Long term*: define in the spec what a sender does with a payload it could not release before the epoch changed (see S-001 for the spec side), and implement that.

**References**: `blend-protocol.md` §Failure Detection and Reaction, §Transition Period.

## 5. Suggestions (non-security)

### S-001 · Spec: the delay interval and the fate of unreleased payloads at an epoch change are underspecified

| | |
|---|---|
| Target | `blend/scheduling/src/release_delayer.rs:L114-L126`; `services/blend/src/pending.rs:L159-L169` |

§Delaying writes the delay as `δ ∈ (1, Δmax)`, which reads as an open interval; the implementation samples the closed interval `[1, Δmax]`, which is the only reading consistent with `T_M = β_max · (Δmax + η)` in §Transition Period. The spec should write `[1, Δmax]`. §Failure Detection counts the deadline "from the round in which the message was released" and says nothing about a payload that was never released before its epoch ended (LB-007); the spec should state whether such a payload is broadcast directly, re-encapsulated under the new epoch, or dropped.

### S-002 · The transition period is derived from the block time, not from `T_M`, and the constraint `T ≥ T_M` is not checked

| | |
|---|---|
| Target | `nodes/node/binary/src/config/blend/deployment.rs:L54-L65`; `services/blend/src/settings/mod.rs:L153-L168` |

`epoch_transition` returns `slot_duration · slots_per_block` (30 s at the testnet's `1/30` activation coefficient), while `max_data_message_delay_in_rounds` computes `T_M = β·(Δmax + η)` from independent settings. The spec makes `T ≥ T_M` a release constraint (§Transition Period). Nothing enforces it: the shipped values satisfy it (`T = 30`, `T_M = 3`; with the spec's `β = 3`, `Δmax = 3`, `T_M = 15`), but a deployment with `Δmax = 8` and the same block time would give `T_M = 30 = T` with no margin, and any shorter block time breaks it. Compute the transition period from `T_M` with the spec's margin, or assert the inequality at config load.

### S-003 · Recovered unsent block proposals re-trigger a cover skip

| | |
|---|---|
| Target | `services/blend/src/core/scheduler.rs:L36-L43`; `blend/scheduling/src/cover_traffic.rs:L155-L161` |

`new_with_initial_messages` calls `queue_data_message_and_skip_cover_message` for every recovered `BlockProposal`, but the `unprocessed_data_messages` counter is not persisted, so a skip that already happened before the restart is applied a second time. The node then emits one cover message fewer than its quota per recovered proposal. Persist the counter, or re-queue recovered proposals with `queue_data_message_without_skipping_cover_message`.

### S-004 · Verify that a node which is the last hop of its own proposal does not report a false delivery failure

| | |
|---|---|
| Target | `services/blend/src/core/dispatcher/libp2p.rs:L121-L149`; `services/chain/chain-network/src/lib.rs:L797-L801` |

The failure detector observes `received_proposals`, which the chain network fills from proposals it receives (`note_received_proposal`). When the local node is itself the last blend hop of its own proposal (probability `1/N`), it dispatches the proposal with `broadcast_block_proposal`, which goes straight to the network service; if gossipsub does not echo a locally published message back as received, the detector never sees the delivery and, `T_M` rounds later, broadcasts the same bytes again "directly", incrementing `data_payload_bypassed_blend`. Confirm the echo behaviour with a test; if there is none, mark the payload as delivered when the node dispatches it itself.

## 6. Checked and ruled out

- **Node selection derivation** (`blend/proofs/src/selection/mod.rs:53-70`): `index = LE(ChaCha20(BLAKE2b-256("BlendNode" ‖ ρ))[0..8]) mod N`, matching §Proof of Selection and the `select_nodes` pseudocode of `message-encapsulation.md`; `verify` rejects an index `≥ N` before computing and checks `ν = Poseidon2("KEY_NULLIFIER_V1", ρ)`. The selection randomness is `Poseidon2("SELECTION_RANDOMNESS_V1", sk, index, period_nonce)` inside the circuit (`proof-of-quota.md` §Pseudocode), so neither the sender nor an observer can steer the index for a given key; the sender can only choose *which* of its quota keys to use, which the spec permits (§Proof of Selection, note). The implementation does not exercise that freedom: proofs are consumed in key-index order (`blend/provers/src/provers/core/mod.rs:113-146`, `crypto/core_and_leader/send.rs:202-222`).
- **Leader nullifiers across slots**: the leader branch uses `pol_sl` as the period nonce, so two winning slots with the same message index yield distinct nullifiers; the `assert!` in `delivery.rs:53-56` cannot be triggered by two proposals.
- **Membership index order** (`services/blend/src/membership/service.rs:53`, `blend/crypto/src/merkle.rs:222-228`): nodes are sorted by ZK public key before `Membership::new`, so every node derives the same index table; the closed issue #77 covers the ordering of the SDP snapshot itself.
- **Peer choice RNGs**: the backend (`core/mod.rs:814`), the release shuffle (`core/mod.rs:818`, `2461`) and the edge handler (`edge/handlers.rs:72`) each use their own `ChaCha20Rng::from_entropy()`; `filter_and_choose_remote_nodes` uses reservoir sampling (`membership/src/lib.rs:92-114`). Only the two scheduler components share a state (LB-001).
- **Delay distribution**: uniform on `[1, Δmax]` and re-drawn at every release round regardless of the queue (`release_delayer.rs:144-156`), as §Delaying requires; an empty queue still yields a (silent) release round, so the release schedule does not reveal the queue state. Generated messages go out at the next tick (`message_scheduler/mod.rs:68-95`), processed ones at the next release round, and everything due in a round is shuffled before publishing (`core/mod.rs:2461`).
- **Cover schedule**: selection sampling gives a uniform random subset of rounds (`cover_traffic.rs:89-97`); the per-proposal skip removes a uniformly random *future* cover slot (`cover_traffic.rs:110-124`, threshold `1/remaining` per release round); `gen_bool` cannot panic because `remaining_messages ≤ remaining_rounds` is an invariant of the sampler. Cover traffic does not depend on incoming traffic, so an idle node emits at the same rate as a busy one.
- **Cover traffic on retirement and shutdown**: `consume()` drops the cover scheduler (`message_scheduler/mod.rs:169-181`); the old-epoch scheduler only drains; killing the service drops all timers, and `Libp2pBlendBackend::drop` aborts the swarm task (`core/backends/libp2p/mod.rs:196-204`). No timer outlives its owner.
- **Restart and key-index reuse**: `spent_core_quota` is persisted and the core proof stream resumes from it (`core/mod.rs:763-783`, `send.rs:105-124`); the recovery state is committed after each release round (`core/mod.rs:2467`). A crash between publishing and committing re-mints at most `β` nullifiers; a neighbour that already processed them drops the copies silently (`blend/network/.../utils.rs:108-116`: the per-peer seen set is cleared on disconnect at `behaviour/mod.rs:1215`, so the re-send is not classified as `DuplicateMessage`). Nothing isolates the node. Issue #61 tracks the broader quota-reuse question.
- **What a restart reveals**: recovered `unsent_processed_messages` and `unsent_data_messages` are re-queued and released in the first round(s) after the restart as one shuffled batch; the burst coincides with the reconnection the neighbours see anyway. The recovery file holds only material that is either public once sent or bound to this node's own rewards.
- **Transition period versus in-flight messages**: old-epoch messages are decapsulated with the old processor and released through the old scheduler until the period expires (`core/mod.rs:1183-1231`), matching §Transition Period; leftovers at expiry are dropped, which is what LB-007 records.

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
