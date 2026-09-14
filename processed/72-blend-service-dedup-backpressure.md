# Audit Report — Blend service: dedup and back-pressure for messages reported by the libp2p backend

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/72`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `services/blend/src/core`, `services/blend/src/core/backends/libp2p`, `blend/network/src/core`, `blend/message`, `blend/provers`, `blend/scheduling`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `proof-of-quota.md`, `message-formatting.md`, `payload-formatting.md` (in full); `blend-protocol.md` by section (listed in Method)
Date: `2026-09-12` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: the Blend service trusts the backend to deliver each message once and never checks a message identifier before doing the expensive work, while the backend's edge path (issue #60, LB-001) and the core path's verification window both deliver copies; every copy is fully decapsulated on the single service task, including two inline Groth16 verifications per layer addressed to the node, and a burst of copies overwrites honest messages in the 64-slot `broadcast` channel, which for a message whose layer belongs to this node is a permanent, network-wide loss.
- Findings: 0 critical · 0 high · 2 medium · 1 low · 2 informational
- Key themes: "no service-side dedup", "drop-oldest channel in front of a slow single consumer", "duplicate handling that logs a drop but does not drop".
- Must-fix before launch: LB-001 (dedup by message identifier before decapsulation, or in the backend before reporting) and LB-002 (make the report path lossless for unique messages, or at least never lossy for messages that only this node can process).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/blend/src/core/backends/libp2p/mod.rs` | channel construction, `listen_to_incoming_messages`, lag handling |
| `services/blend/src/core/backends/libp2p/swarm.rs` | `report_message_to_service`, core and edge event handlers, publish/forward paths |
| `services/blend/src/core/mod.rs` | event loops (`run_event_loop`, `run_during_transition`, `retire`), `handle_incoming_blend_message*`, `schedule_decapsulated_incoming_message`, release rounds |
| `services/blend/src/core/state.rs`, `services/blend/src/message.rs` | `unsent_processed_messages` set, `ProcessedMessage`, state snapshots |
| `services/blend/src/core/dispatcher/libp2p.rs` | what a released decapsulated payload does downstream |
| `services/blend/src/metrics.rs` | `inbound_messages_dropped` and neighbours |
| `blend/network/src/core/{poq_verification.rs, with_core/behaviour/{mod,utils,message_cache,old_epoch}.rs, with_edge/behaviour/mod.rs}` | only what decides how many times one message is reported (re-verified at this commit) |
| `blend/provers/src/crypto/core_and_leader/receive.rs`, `blend/message/src/encap/{validated,encapsulated}.rs`, `blend/proofs/src/selection/mod.rs`, `blend/message/src/crypto/proofs.rs` | what one decapsulation costs |
| `blend/scheduling/src/release_delayer.rs`, `blend/message/src/reward/mod.rs` | where a duplicate ends up after decapsulation |

**Out of scope**
The edge-path amplification itself (#60 LB-001, #74), connection limits (#60 LB-002), the connection monitor (#73), PoQ circuit soundness, message encapsulation correctness, the failure detector's own logic (#154), reward accounting (#89). Third-party crates assumed correct: `tokio` (`sync::broadcast`, `sync::mpsc`), `tokio-stream`, `libp2p` 0.56, `lb_groth16` / `rust-rapidsnark`, `ed25519-dalek`, `x25519-dalek`, `chacha20`, `blake2`, `overwatch` @ `ae887f41`.

**Assumptions**
- Release-profile facts from issue #19 hold: `overflow-checks` off, `unwrap_used` / `expect_used` / `panic` allowed. Nothing in this report depends on wrapping arithmetic.
- The spec's traffic model: about one new message per round network-wide (`α ≈ 1.03`, blend-protocol.md › Releasing), `β_max = 3` layers, `Δ_max = 3` rounds, one round = 1 s.
- Shipped defaults as in #60: `core_peering_degree: 3..=5`, `edge_node_connection_timeout: 1s`, `max_edge_node_incoming_connections: 300` (`nodes/node/binary/src/config/blend/serde/core.rs:54-56`).
- Any party that can produce one valid PoQ can replay the message carrying it through the edge path indefinitely (#60 LB-001, re-verified below). The PoW branch of the PoQ (proof-of-quota.md › Step 4) makes such a proof available to anyone without stake or declaration.
- A Groth16 verification on BN254 takes a few milliseconds; it was not measured here (see Method). Where a number matters it is kept as the parameter `t_v`.

## 3. Method

- Manual review of the in-scope paths, working through issue `#72` (all four items) with parent `#13` for context and the `#60` report (PR #71) as the starting point. Both `⚑`-style claims inherited from #60 (edge path reports without dedup; core path reports once per peer within the verification window) were re-verified at `a805329f`.
- Spec conformance against `blend-protocol.md` › Networking, Time, Messages (Generation / Relaying / Processing / Broadcasting), Protocol › Message Lifecycle and Failure Detection and Reaction, Details › Global Parameters, Connectivity Maintenance, Transition Period, Quota, Quota Application, Proof of Quota, Proof of Selection, Message Structure, Relaying, Processing, Delaying, Releasing, Broadcasting, Activity Proof; `proof-of-quota.md`, `message-formatting.md`, `payload-formatting.md` in full.
- Automated tooling: none. The shared checkout must not be built; `zk/proofs/poq/benches/verify.rs` exists but needs the circuit artefacts referenced from `circuits.env`, which are not on this machine, so Groth16 timings are left parametric.
- Dynamic testing: none.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | The service decapsulates every reported copy of a message: no identifier check before decapsulation, two inline Groth16 verifications per replayed layer addressed to the node | Denial of Service | Medium | Low | Open |
| LB-002 | A burst of reported copies evicts honest messages from the 64-slot drop-oldest channel, and an evicted message whose layer belongs to this node is lost network-wide | Denial of Service | Medium | Medium | Open |
| LB-003 | A duplicate decapsulated message is scheduled for release before the duplicate check, so it is released, published, and dispatched a second time | Data Validation | Low | Low | Open |
| LB-004 | Messages verified before the service subscribes to the backend channel are relayed but never processed locally, and cannot be recovered | Timing | Informational | High | Open |
| LB-005 | Spec deviation: the nullifier of a decapsulated header is checked at release time against the forwarded set, not at processing time against the seen set | Data Validation | Informational | High | Open |

### LB-001 · The service decapsulates every reported copy of a message: no identifier check before decapsulation, two inline Groth16 verifications per replayed layer addressed to the node

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `services/blend/src/core/mod.rs:L2098-L2149` (`handle_incoming_blend_message`), `L2153-L2172` (`try_decapsulate`), `L2176-L2200` (`handle_incoming_blend_message_from_old_epoch`), `L1097-L1100`, `L1191-L1194`, `L1652-L1655` (the three `select!` arms); `blend/provers/src/crypto/core_and_leader/receive.rs:L116-L171`; `blend/message/src/encap/encapsulated.rs:L239-L266`, `L339-L352`, `L86-L110` |
| Status | Open |

**Description**

Item 1 of the issue. The answer is no: nothing on the service side deduplicates. `handle_incoming_blend_message` takes the `(message, epoch)` pair off the stream and goes straight to decapsulation:

```rust
// services/blend/src/core/mod.rs
2121   if epoch == cryptographic_processor.epoch() {
2122       let Some(output) = try_decapsulate(verified_message, cryptographic_processor, epoch) else {
2123           return current_recovery_checkpoint;
2124       };
```

`try_decapsulate` calls `decapsulate_message_recursive` (`receive.rs:116-171`). The service holds no set of message identifiers (`MessageIdentifier` is the PoQ key nullifier, `blend/message/src/encap/encapsulated.rs:39`), and the backend's `MessageCache` is private to the swarm task. The only duplicate check downstream is `add_unsent_processed_message` on the recovery state (`state.rs:335-344`), which runs after decapsulation and only for messages that decapsulated (LB-003).

What one copy costs, all of it inline on the service task (no `spawn_blocking`, unlike the backend's `poq_verification.rs:59-66`):

1. Outer layer not addressed to this node (the common case, `N-1` out of `N`): X25519 shared-key derivation (`validated.rs:260`), ChaCha decryption of the three blending headers and of the 18 KB payload (`encapsulated.rs:518-521`, `524-529`), one deserialisation attempt and one PoSel check (`encapsulated.rs:527-541`, `selection/mod.rs:72-104`: blake2b, a modulo, a Poseidon2 hash). Sub-millisecond, but never zero.
2. Outer layer addressed to this node and not the last one: all of the above, then an Ed25519 signature check and a Groth16 PoQ verification of the reconstructed inner header inside `decapsulate` (`encapsulated.rs:259-266` → `verify_intermediate_reconstructed_public_header`, `L339-L352`), then a second signature check and a second Groth16 verification of the same header in `decapsulate_message_recursive` (`receive.rs:152-156` → `validate_message_header` → `encapsulated.rs:86-110`), then the next-layer attempt. That is `2·t_v` plus two Ed25519 verifications per copy (S-001).
3. Outer layer addressed to this node and last: item 1 plus payload parsing, then the duplicate is scheduled and dispatched again (LB-003).

How many copies the backend delivers, re-verified at this commit:

- Edge path: `with_edge/behaviour/mod.rs:195-225` deserialises, checks the signature, and spawns a PoQ verification with no cache lookup (the struct at `L77-L100` has no cache); every verified copy becomes an `Event::Message`, and `swarm.rs:959-969` publishes it (deduplicated by the core cache, `utils.rs:45-47`) and then unconditionally reports it (`L968`). One report per QUIC connection that carries the message.
- Core path: `utils.rs:112-116` skips a message already marked in the cache, but the mark is set only when the verification outcome comes back (`with_core/behaviour/mod.rs:967-976`), so copies arriving from different peers within one verification latency are each verified and each reported (`swarm.rs:464-469`). Bounded by the peering degree, at most 5 per message with shipped defaults. A single core peer cannot replay (`utils.rs:108-110` → `SpamReason::DuplicateMessage`).
- Old-epoch path: `old_epoch.rs:245-278` uses the same `handle_received_serialized_encapsulated_message_and_update_cache`, same window.

The attacker's per-copy price is a QUIC handshake, a 20 KB upload, and one Groth16 verification on the victim's blocking pool. The victim's per-copy price on the service task is item 1 or item 2 above. Item 2 is reachable on purpose: the PoSel index is a deterministic function of the PoQ (`proof-of-quota.md` › Step 5; `selection/mod.rs:60-70`), so about `1/N` of any key pool selects the victim as first hop, and the PoW branch lets anyone mint keys at the cost of a puzzle solution. The attacker builds one message whose outer layer selects the victim, obtains one valid PoQ for it, and replays.

**Exploit scenario**

The attacker holds a message `m` (about 20 KB) with a valid PoQ whose first hop is the victim, with at least two layers so the victim's decapsulation reaches item 2. From one host it opens edge connections to the victim's blend listener in a loop and sends `m` on each. Per copy the victim pays: TLS handshake and Ed25519 on the swarm task; `t_v` on the blocking pool; then `2·t_v` plus two Ed25519 verifications plus a full decryption on the service task, serialised behind every other thing the service task does (release rounds, cover generation, epoch handling). The service task's throughput on such copies is about `1 / (2·t_v)` per second, independent of core count. With `t_v ≈ 2 ms` that is about 250 copies/s, which is one host's QUIC handshake budget with margin. The first copy is legitimately processed; every further copy is pure loss. The same replay also drives LB-002, which is where the damage to other users comes from; on its own this finding is a CPU-time sink on the one task that keeps the node's Blend hop alive.

**Recommendation**
- *Short term*: keep a per-epoch `HashSet<MessageIdentifier>` of identifiers already handed to `try_decapsulate` (current epoch and old epoch, dropped with the old epoch's processor at transition end), and return before decapsulation on a hit. The bound is the same as the spec's nullifier cache (`blend-protocol.md` › Relaying, `(E + TP)·(F_C + F_D)·β_max` entries, about 65 MB worst case, about 21 MB at the modelled rate); it can share the backend's `MessageCache` if the check is moved to the backend instead (below). Move the per-layer verification work (`decapsulate_message_recursive`) to `spawn_blocking`, as the backend already does for the outer PoQ.
- *Long term*: deduplicate in the backend before reporting, so the service can keep its "verified once, delivered once" assumption (documented at `backends/mod.rs:53-57`): check the edge message against the core `MessageCache` before signature and PoQ verification (also closes #60 LB-001's replay component), and keep an in-flight set of identifiers whose verification has been spawned so a copy from a second peer is neither verified nor reported (S-002 of #60). Then a duplicate report becomes a bug, not a load path.

**References**: `blend-protocol.md` › Relaying (steps 1.4, 1.5, 1.6: nullifier check before signature and PoQ) and › Processing; #60 LB-001 and S-002; #74.

### LB-002 · A burst of reported copies evicts honest messages from the 64-slot drop-oldest channel, and an evicted message whose layer belongs to this node is lost network-wide

| | |
|---|---|
| Severity | Medium |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | `services/blend/src/core/backends/libp2p/mod.rs:L56`, `L75`, `L149-L157`, `L181-L194`; `services/blend/src/core/backends/libp2p/swarm.rs:L916-L933`, `L464-L469`, `L959-L969`; `services/blend/src/core/mod.rs:L1089-L1121` (the `select!` loop), `L2463-L2464`; `blend/network/src/core/with_core/behaviour/utils.rs:L112-L116` |
| Status | Open |

**Description**

Items 2 and 3 of the issue. The channel is `tokio::sync::broadcast` with capacity 64 (`mod.rs:56`, `75`). `report_message_to_service` calls `send`, which never blocks and overwrites the oldest buffered value when the buffer is full (`swarm.rs:927`). The consumer is `BroadcastStream` in `listen_to_incoming_messages` (`mod.rs:149-157`); when it has fallen behind it receives `Lagged(n)`, which `handle_incoming_broadcast_event` turns into a warning and a counter increment and then continues from the oldest value still buffered (`mod.rs:181-194`). So the eviction policy is drop-oldest, and what is dropped is whatever had been waiting longest: under a burst of copies of one message, that is the honest messages queued ahead of the burst.

What is lost when a report is evicted:

- Relaying is not affected. The backend forwards or publishes before it reports (`swarm.rs:466` then `468`; `L966` then `L968`), so the rest of the network still gets the message.
- The local layer is lost for good. PoSel selects exactly one node per layer (`blend-protocol.md` › Proof of Selection), so if the evicted message had a layer addressed to this node, no other node can strip it: the message dies here. Nothing retries: the backend's cache already holds the identifier as `Forwarded`, so a second copy from another peer is dropped before verification (`utils.rs:112-116`) and never reaches the service again.
- The sender observes a delivery failure after the message traversal time and, unless `abstain_on_failure` is set, broadcasts its payload in the clear (`blend-protocol.md` › Failure Detection and Reaction › Direct Broadcast; `services/blend/src/delivery/failure_detection.rs`). The anonymity of that sender is what the eviction actually costs.
- The blending token of that layer is not collected, so the node's activity-proof lottery ticket for that message is gone (`blend-protocol.md` › Activity Proof).
- Observability is the counter `blend_inbound_messages_dropped_total` (`metrics.rs:64-66`) and one `warn!`. Nothing downstream is told which messages were lost, and nothing consults the counter.

Quantification against `CHANNEL_SIZE = 64`. Let `t_v` be one Groth16 verification, `C` the number of cores the blocking pool can use, and consider the replay from LB-001 of a message whose outer layer selects the victim.

- Producer rate: the backend verifies copies in parallel on the blocking pool, up to about `C / t_v` per second, capped by the attacker's handshake rate `h`.
- Consumer rate: the service task drains copies at about `1 / (2·t_v)` per second (LB-001, item 2), and drains nothing at all while it is inside an `.await` of the same `select!` loop — a release round waits for every `backend.publish` to enter a 64-slot `mpsc` (`mod.rs:2463-2464`, `backends/libp2p/mod.rs:105-119`), for the gossipsub relay (`dispatcher/libp2p.rs:65-79`) and for the mempool's `Add` round trip (`dispatcher/libp2p.rs:170-187`).
- The buffer overflows once the producer is 64 reports ahead. With producer `min(h, C/t_v)` and consumer `1/(2·t_v)`, the lead grows at `min(h, C/t_v) − 1/(2·t_v)` per second, so the 64-slot buffer is exceeded after `64 / (min(h, C/t_v) − 1/(2·t_v))` seconds: for `t_v = 2 ms`, `C = 8`, `h = 500/s`, that is about `64 / (500 − 250) ≈ 0.26 s` of sustained replay, and for `h = 300/s` about `1.3 s`. From then on the channel stays full and each honest report arriving during the attack survives only if it is consumed before 64 further reports arrive, i.e. with probability about `consumer / producer ≈ 1/(2C)` to `1/2` depending on `h`; the rest are evicted.
- With a message not addressed to the victim the consumer cost drops to sub-millisecond (LB-001, item 1) and the attacker needs thousands of handshakes per second to win, which one host cannot sustain; the victim-addressed message is the realistic vector.
- Honest traffic alone does not lag the channel: the network emits about one new message per round (`α ≈ 1.03`), each reaching a node once plus at most `peering_degree − 1` extra copies within a verification latency, so 64 slots hold roughly a minute of honest input, and the consumer would have to stall for that long. The stalls listed above are shorter than that in normal operation.

Because the replay needs only one valid PoQ and the edge path (a second weakness, #60 LB-001), and because the damage is to the liveness of every message routed through the victim and to the anonymity of their senders rather than a crash, this is rated Medium.

**Exploit scenario**

The attacker from LB-001 sustains `h ≈ 300-500` edge connections per second to the victim, each carrying the same victim-addressed message. Within about a second the channel is permanently full. Every honest message the victim's backend verifies and forwards during the attack is reported into a full channel and, with high probability, overwritten before the service task polls it. The victim keeps relaying, so no peer sees a fault; the victim's own logs show `Incoming blend message channel lagged behind` and the counter climbs. Every honest message that had a layer for the victim in that period is dead; each of its senders, after `T_M`, broadcasts its block proposal or transaction directly, linking itself to it. The attacker can hold this for as long as it can keep up the handshake rate, and needs no stake, no declaration, and one PoW-backed PoQ.

**Recommendation**
- *Short term*: (a) deduplicate before reporting (LB-001 long-term), which removes the only cheap way to generate a burst; (b) make the channel drop-newest instead of drop-oldest, so a burst cannot evict what was already accepted (S-002); (c) raise the capacity to cover a realistic stall of the consumer (the per-epoch bound of honest input over the longest expected `.await` in the loop, not 64); (d) never let a message that only this node can process be dropped silently — at minimum log its identifier at `warn!` so an operator can correlate with the sender's direct broadcasts.
- *Long term*: separate the consumer from the release/dispatch work (a dedicated task that only decapsulates, feeding the scheduler through a bounded queue), so the report path is never blocked by a mempool or gossipsub round trip, and move decapsulation to the blocking pool (S-001). With dedup in place, the remaining input rate is bounded by the network quota and a lossless bounded `mpsc` becomes viable.

**References**: `blend-protocol.md` › Proof of Selection, › Releasing (`μ`, `α`), › Failure Detection and Reaction; `tokio::sync::broadcast` documentation on `RecvError::Lagged`; #60 LB-001; #154.

### LB-003 · A duplicate decapsulated message is scheduled for release before the duplicate check, so it is released, published, and dispatched a second time

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Data Validation |
| Target | `services/blend/src/core/mod.rs:L2293-L2355` (`schedule_decapsulated_incoming_message`, `L2333`, `L2351`), `L2226-L2247` (`handle_decapsulated_incoming_message_from_current_epoch`), `L2253-L2283`, `L2176-L2200` (old-epoch variants), `L2564-L2616` (`build_futures_to_release_processed_messages`); `blend/scheduling/src/release_delayer.rs:L30`, `L87-L90`; `services/blend/src/core/tests/mod.rs:L379-L473` |
| Status | Open |

**Description**

`schedule_decapsulated_incoming_message` pushes the `ProcessedMessage` into the scheduler unconditionally and returns it:

```rust
// services/blend/src/core/mod.rs
2332   let processed_message = ProcessedMessage::from(data_message);
2333   scheduler.schedule_processed_message(processed_message.clone());
2334   (Some(processed_message), blending_tokens.into_iter())
...
2351   scheduler.schedule_processed_message(processed_message.clone());
```

Only afterwards does the caller consult the recovery state, and on a duplicate it logs a drop that does not happen:

```rust
2234   if let Some(processed_message) = maybe_processed_message
2235       && state_updater
2236           .add_unsent_processed_message(processed_message)
2237           .is_err()
2238   {
2239       tracing::trace!(
2240           target: LOG_TARGET,
2241           "Dropping a duplicate decapsulated replica already pending release."
2242       );
2243   }
```

The message is already in `unreleased_messages: Vec<ProcessedMessage>` (`release_delayer.rs:30`, `87-90`) and is released at the next round. At release, `build_futures_to_release_processed_messages` fails to remove it from the state, warns for the encapsulated variant (`L2594-L2597`: "Previously processed message should be present in the recovery state but was not found."), and sends it anyway: an `Encapsulated` duplicate goes to `backend.publish`, where the swarm's cache rejects it as `DuplicateMessage` (`utils.rs:45-47`, traced at `swarm.rs:761`); a `Decapsulated` duplicate goes to `dispatch`, which broadcasts a block proposal to gossipsub again (`dispatcher/libp2p.rs:65-79`) or submits a transaction to the mempool again and waits for the reply (`L153-L189`). The old-epoch variants (`L2253-L2283`, `L2176-L2200`) have no duplicate check at all.

This is reached by honest traffic, not only by replay: the service emits `data_replication_factor + 1` replicas of each data message with distinct outer layers, and the last hop decapsulates all of them to the same `DataPayload`, which is what the regression test `test_duplicate_decapsulated_replica_handled_gracefully` (`core/tests/mod.rs:379-473`) exercises. The test asserts only that the recovery state holds one entry; the scheduler holds two, and the doc comment's claim that the second replica is "dropped gracefully" is not what the code does.

**Exploit scenario**

None beyond LB-001: each replayed copy that fully decapsulates is dispatched once more, so a replayed transaction is re-submitted to the mempool and a replayed proposal re-published on gossipsub at every release round, at the victim's cost and with the misleading warning above in its logs. Impact is wasted work and noise; both downstream consumers reject duplicates on their own.

**Recommendation**
- *Short term*: check `add_unsent_processed_message` before `schedule_processed_message` and skip scheduling on `Err`; apply the same check in the old-epoch paths (the old-epoch state has no set today; a local `HashSet` in `RetiringEpoch` / `CurrentEpochDuringTransition` is enough); make the test assert `scheduler.release_delayer().unreleased_messages().len() == 1`.
- *Long term*: LB-001's identifier check makes the replay case unreachable; the replica case still needs this fix because replicas have distinct identifiers by design.

**References**: `blend-protocol.md` › Releasing; #60 S-002.

### LB-004 · Messages verified before the service subscribes to the backend channel are relayed but never processed locally, and cannot be recovered

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Timing |
| Target | `services/blend/src/core/backends/libp2p/mod.rs:L75`, `L149-L157`; `services/blend/src/core/backends/libp2p/swarm.rs:L927-L929`; `services/blend/src/core/mod.rs:L804-L815`, `L452-L463` |
| Status | Open |

**Description**

The backend creates the channel with `let (incoming_message_sender, _) = broadcast::channel(CHANNEL_SIZE)` (`mod.rs:75`) and drops the only receiver; `send` therefore fails until `listen_to_incoming_messages` calls `subscribe()` (`L149-L157`). `Backend::new` is called inside `initialize` (`core/mod.rs:804-815`) and immediately spawns the swarm, which listens and dials. The subscription happens at `core/mod.rs:463`, after `notify_ready` (`L452`) and `post_initialize` (`L461`, which awaits the PoL-info subscription). A message verified in between is forwarded by the swarm (`swarm.rs:466`, `966`) and then dropped at `report_message_to_service` with a `trace!` ("No active listeners yet", `L927-L929`) and `inbound_message_err`. Because the cache marks it `Forwarded`, a later copy is discarded before verification (`utils.rs:112-116`), so the local layer of that message is lost in the same permanent way as in LB-002. The window is short (a dial, a handshake and a message have to complete first), which is why this is informational; it is the same fire-and-forget report path as LB-002, seen at start-up.

**Exploit scenario**

None. Impact is limited to the first messages after a start or restart.

**Recommendation**
- *Short term*: create the receiver in `Backend::new` and hand it out from `listen_to_incoming_messages`, so nothing sent by the swarm is ever unsubscribed; or delay `swarm.run()` until the service has subscribed.
- *Long term*: covered by LB-002's long-term change (a bounded queue owned by the consumer).

**References**: `tokio::sync::broadcast::Sender::send` returns `Err` when there are no receivers.

### LB-005 · Spec deviation: the nullifier of a decapsulated header is checked at release time against the forwarded set, not at processing time against the seen set

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Data Validation |
| Target | `blend/provers/src/crypto/core_and_leader/receive.rs:L152-L156`; `services/blend/src/core/mod.rs:L2336-L2353`; `blend/network/src/core/with_core/behaviour/utils.rs:L45-L47`; `blend/network/src/core/with_core/behaviour/message_cache.rs:L107-L121` |
| Status | Open |

**Description**

`blend-protocol.md` › Processing, step 2.4.1: after decapsulation, "If the PoQ nullifier from the public header of the message was already seen, then the node was not allowed to use the same PoQ nullifier and the message must be discarded. The cache is the one maintained by Relaying." The code, after decapsulation, verifies the inner header's signature and PoQ (`receive.rs:152-156`) but consults no cache; the message is delayed (`core/mod.rs:2351`) and the check happens only when it is published, in `forward_validated_message_and_update_cache`, and there against `is_message_forwarded` (`utils.rs:45`, `message_cache.rs:116-121`), which matches only the `Forwarded` status and not `Processed`. In practice every identifier that reaches the cache through the core or edge path becomes `Forwarded` right after it is reported, so the two sets differ only for a message whose forwarding failed with `NoPeers`. The deviation is the order: a duplicated inner layer costs the node the full inner verification and up to `Δ_max` rounds in the delayer before it is discarded, and the discard is logged as a send failure rather than as the duplicate it is. The code is the side to change; the spec is consistent with its Relaying section.

**Exploit scenario**

None on its own; it is the reason the service cannot short-circuit a replayed inner layer before doing LB-001's item 2 work.

**Recommendation**
- *Short term*: expose an identifier check from the backend (or the shared set from LB-001) and call it in `decapsulate_message_recursive` before `validate_message_header`.
- *Long term*: none beyond LB-001.

**References**: `blend-protocol.md` › Processing (2.4.1), › Relaying (1.4).

## 5. Suggestions (non-security)

### S-001 · The inner header's PoQ is verified twice per decapsulated layer, inline on the service task

| | |
|---|---|
| Target | `blend/message/src/encap/encapsulated.rs:L259-L266`, `L339-L352`; `blend/provers/src/crypto/core_and_leader/receive.rs:L152-L156`; `blend/message/src/encap/encapsulated.rs:L86-L110` |

`EncapsulatedPart::decapsulate` calls `verify_intermediate_reconstructed_public_header`, which verifies the signature and the PoQ of the reconstructed inner header; `decapsulate_message_recursive` then calls `validate_message_header` on the same header, which verifies both again. Every layer this node strips therefore costs two Groth16 verifications and two Ed25519 verifications where one of each is enough, and all of it runs on the service task rather than on the blocking pool the backend uses for the outer header. Return an already-verified header from `decapsulate` (the type `EncapsulatedMessageWithVerifiedPublicHeader` exists for exactly this), and run the recursive decapsulation under `spawn_blocking`. This halves the per-copy cost in LB-001 and doubles the consumer rate in LB-002.

### S-002 · `broadcast` is the wrong channel for one consumer; the choice of drop policy matters less than dedup

| | |
|---|---|
| Target | `services/blend/src/core/backends/libp2p/mod.rs:L49-L56`, `L75`, `L149-L157`, `L181-L194` |

Item 4 of the issue. The channel has exactly one subscriber for its whole life (`listen_to_incoming_messages` is called once, `core/mod.rs:463`), so `broadcast` buys nothing except the ability to subscribe after construction (which LB-004 shows to be a liability). The candidates:

- `mpsc::channel(N)` with `try_send`, dropping the newest on `Full`: a burst can no longer evict what was already accepted; honest messages arriving during the burst are still dropped. Keep the same metric and warning.
- `mpsc::channel(N)` with `send().await` from the swarm task: lossless, but back-pressures the libp2p swarm task, which also services every peer's stream and the connection monitor; a slow service would then stall relaying for the whole network, which is worse than a local loss. Not advisable as long as the swarm task and the report path share a task.
- Keep `broadcast` and drop-oldest: today's behaviour, LB-002.

Whichever is chosen, the policy only decides which honest message is lost under a burst; only deduplication (LB-001) removes the burst. A reasonable end state is dedup in the backend plus a drop-newest `mpsc` sized from the per-round honest bound (`μ`-derived, `blend-protocol.md` › Global Parameters) times the longest `.await` the consumer can be inside, plus a `warn!` naming the identifier on every drop.

### S-003 · The whole recovery state is cloned and pushed to the state updater on every decapsulated message

| | |
|---|---|
| Target | `services/blend/src/core/state.rs:L242-L244`, `L565-L570`; `services/blend/src/core/mod.rs:L2245-L2246`, `L2278-L2282` |

`commit_changes` calls `save`, which does `self.state_updater.update(Some(self.clone().into()))`: a deep clone of `ServiceState`, including both epochs' blending-token sets and every unsent message (each about 20 KB), on every incoming message that decapsulated and on every release round. The cost grows through the epoch with the number of tokens collected and is paid on the service task, so it is another term in LB-002's consumer cost, and under LB-001's replay it is paid per copy. Batch the snapshot per release round, or snapshot a diff.

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

## Appendix B — Checklist items verified

| Item (#72) | Evidence | Result |
|---|---|---|
| Does the service deduplicate by message id / key nullifier before decapsulation? | No identifier set anywhere in `services/blend/src/core`; `handle_incoming_blend_message` (`core/mod.rs:2098-2149`) goes straight to `try_decapsulate`; the only later check is `add_unsent_processed_message` (`state.rs:335-344`), after decapsulation and after scheduling. Per copy: decryption + PoSel, plus two Ed25519 and two Groth16 verifications per layer addressed to the node (`encapsulated.rs:259-266`, `339-352`; `receive.rs:152-156`), inline on the service task | **LB-001**, **S-001** |
| What is dropped on lag; can a replay of one valid message evict legitimate messages? Quantify against 64 | `broadcast::channel(64)` (`backends/libp2p/mod.rs:75`), `send` overwrites the oldest (`swarm.rs:927`), consumer skips to the oldest retained on `Lagged` (`mod.rs:181-194`). Producer up to `C/t_v` (blocking pool, `poq_verification.rs:59-66`), consumer about `1/(2·t_v)` on victim-addressed copies and zero during release-round awaits (`core/mod.rs:2463-2464`, `dispatcher/libp2p.rs:170-187`). Buffer exceeded after `64 / (min(h, C/t_v) − 1/(2·t_v))` s; about 0.3-1.3 s at `t_v = 2 ms`, `C = 8`, `h = 300-500/s`. Honest input alone is about a minute per 64 slots | **LB-002** |
| Is the drop observable beyond the metric; does anything downstream assume delivery? | Only `blend_inbound_messages_dropped_total` (`metrics.rs:64-66`) and a `warn!`. Delivery is assumed by: the sender's failure detector (direct broadcast after `T_M`, `blend-protocol.md` › Direct Broadcast, `delivery/failure_detection.rs`), the reward lottery (blending token of the lost layer never collected, `reward/mod.rs:62-78`), and the backend's own cache, which marks the identifier `Forwarded` so no retry can reach the service (`utils.rs:112-116`). Relaying itself is not affected (forward precedes report, `swarm.rs:466/468`, `966/968`) | **LB-002** (permanent local loss), **LB-004** (same at start-up) |
| Would a bounded `mpsc` with an explicit drop policy be a better fit than `broadcast` for a single consumer? | One subscriber for the channel's lifetime (`core/mod.rs:463`); `broadcast` only provides late subscription, which drops pre-subscription reports (`swarm.rs:927-929`). Drop-newest `mpsc` stops eviction of accepted messages; blocking `mpsc` would stall the swarm task; neither removes the burst without dedup | **S-002**, **LB-004** |
| (re-verified from #60) Edge path reports every verified copy without dedup | `with_edge/behaviour/mod.rs:195-225` (no cache), `swarm.rs:959-969` (publish deduped, report unconditional) | Holds at `a805329f` |
| (re-verified from #60 S-002) Core path reports once per peer within the verification window | `utils.rs:112-116` checks the cache, `with_core/behaviour/mod.rs:967-976` marks it only on outcome; `utils.rs:108-110` blocks single-peer replays | Holds at `a805329f`; bounded by peering degree |
| Where a duplicate ends up after decapsulation | Scheduled before the state check (`core/mod.rs:2333`, `2351` vs `2234-2243`); released and re-sent (`2564-2616`); tokens deduplicated by `HashSet` (`reward/mod.rs:65`, `78`); old-epoch paths unchecked (`2176-2200`, `2253-2283`) | **LB-003** |
| Spec: where the nullifier check on the decapsulated header happens | Spec › Processing 2.4.1 says at processing, against the Relaying cache; code checks at release against `Forwarded` only (`utils.rs:45`, `message_cache.rs:116-121`) | **LB-005** |
