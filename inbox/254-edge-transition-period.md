# Audit Report — Blend core-to-edge behaviour applies no Transition Period

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/254`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `3d5d419ec85c7b23e4d7e1455bd1cde845264a9e` — component(s): `blend/network/src/core/with_edge/behaviour/{mod.rs,tests/epoch.rs,tests/message_handling.rs}`, `blend/network/src/core/poq_verification.rs`, `blend/network/src/core/with_core/behaviour/mod.rs` (for comparison), `services/blend/src/core/backends/libp2p/swarm.rs`, `services/blend/src/edge/{mod.rs,current_epoch.rs,handlers.rs,backends/libp2p/{mod.rs,swarm.rs}}`, `services/blend/src/settings/mod.rs`, `nodes/node/standalone-node-config.yaml`
Specs: `https://github.com/logos-co/logos-lips` @ `75d3d0382604d4a0d8e246c268935dd386ffc8ea` — read by section: `blend-protocol.md` › Edge Network (L347-L368), › Transition Period (L578-L603), › Detection (L446-L448), › Direct Broadcast (L450-L456); parent #13 set read for earlier reports
Date: `2026-09-15` — author: `Claude (research workflow, iteration 10)` — status: `final`

`blend/network` and `services/blend` are unchanged between `a805329f` (the commit #183 LB-001 / #521 was filed against) and `3d5d419e`; line numbers below are for `3d5d419e` and match the ones in #521.

---

## 1. Summary

- Overall assessment: the deviation filed as #521 (183-LB-001) is confirmed at the current commit, with two additions. First, the window has two directions and the larger one is the reverse of the one the issue describes: an edge node that has rotated sends an epoch-`e` message to a core node whose core service is still on `e − 1`, and that core drops it too; the #234 measurement puts the core service's lag behind its own slot tick at 0.2-9.6 s under load, against sub-second for the direction the issue names. Second, a behaviour-level test shows that a message already on the wire when the core rotates is dropped as well: `start_new_epoch` closes the substream and, if the bytes are read at all, verifies them with the new verifier only. In every case the edge node sees a successful send, waits out the delivery deadline, and then broadcasts the proposal in the clear (or abstains). The spec rule in § Transition Period is written for "the node" and its "old connections"; it does not say whether the single-message edge connections are covered, and the code's own test asserts they are not ("no dual-epoch for edge"). This report drafts the sentence for both readings and recommends the covering one.
- Findings: 0 critical · 0 high · 0 medium · 1 low · 1 informational
- Key themes: "transition period applies to core connections only", "the reverse direction is the bigger window", "edge sender cannot tell"
- Must-fix before launch: none on its own; LB-001 should land with the #235 protocol-name change so that both directions are handled once.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `blend/network/src/core/with_edge/behaviour/mod.rs` L104-L140, L195-L250, L318-L340 | `new`, `start_new_epoch`, receive path, verification outcome, handler event dispatch |
| `blend/network/src/core/poq_verification.rs` L45-L112 | Verification spawned with the epoch and verifier captured at spawn time |
| `blend/network/src/core/with_core/behaviour/mod.rs` L285-L320, L865-L880 | Core-to-core counterpart (`old_epoch`, `publish_message_with_validated_header` by epoch) |
| `services/blend/src/core/backends/libp2p/swarm.rs` L855-L880, L959-L970 | What the swarm does with an edge message and its epoch |
| `services/blend/src/edge/mod.rs` L370-L400, L440-L465; `edge/current_epoch.rs` L120-L145; `edge/backends/libp2p/mod.rs` L60-L92; `edge/backends/libp2p/swarm.rs` L220-L360 | Edge rotation, backend abort on drop, dial/retry ladder |
| `services/blend/src/settings/mod.rs` L100-L110, L150-L170 | Edge delivery deadline |
| `blend/network/src/core/with_edge/behaviour/tests/{epoch.rs,message_handling.rs}` | Existing coverage (item 4) |
| #234 report (`inbox/234-blend-epoch-skew-devnet.md`, PR #599) | Measured core-service latch lag, used for the window bound |

**Out of scope**
The core-to-core transition (#116, #322, #235); the transition-period length (#535); the delivery-failure detector's own gaps (#540, #541, #586); the edge service's missing recovery state (#172); PoQ circuit and verifier correctness. `libp2p-stream`, `libp2p-swarm-test` assumed correct.

**Assumptions**
Shipped configuration: `edge_node_connection_timeout: 1 s`, `max_edge_node_incoming_connections: 300`, edge `replication_factor: 1`, `max_dial_attempts_per_peer_per_message: 1` (`standalone-node-config.yaml` L106-L115); `epoch_transition_period = slots_per_block · slot_duration` (`config/blend/deployment.rs` L59-L65), 20 s with the deployment defaults; delivery deadline `num_blend_layers · maximum_release_delay_in_rounds` rounds (`settings/mod.rs` L150-L170).

## 3. Method

- Manual review of the receive, rotation and verification paths on both sides, and of the swarm's use of the `epoch` carried on `Event::Message`.
- Spec conformance against § Transition Period (the four rules for "when a new epoch begins"), § Edge Network (connect, send one message, close), § Detection and § Direct Broadcast (what the sender does when nothing comes back).
- Dynamic testing: two throw-away behaviour-level tests appended to `with_edge/behaviour/tests/epoch.rs` (removed afterwards; worktree clean), release build, `cargo test -p logos-blockchain-blend-network`. Both use the crate's `TestSwarm`, `StreamBehaviour` edge, `BehaviourBuilder` core and `TestEncapsulatedMessage`; the core rotates with `start_new_epoch((membership, 1), TestProofsVerifier::rejecting())`, which stands in for "the new epoch's public inputs, under which an epoch-0 proof does not verify". (a) *In flight at rotation*: the edge connects, upgrades and writes the message; the core rotates before it polls the connection. (b) *After rotation*: the core rotates first; the edge then connects and sends. In both cases the core is polled for 3 s.

| case | `Event::Message` reported | notes |
|---|---|---|
| (a) message on the wire when the core rotates | 0 | `start_new_epoch` closed the substream (`upgraded_edge_peers` emptied, L134-L139); the bytes were never verified under the old inputs |
| (b) message sent after the core rotated | 0 | verified with the new verifier only (L217-L223), dropped at L242-L249 |

Against the same setup with an accepting verifier and no rotation the existing `receive_valid_message` test reports the message, so the drops are the rotation's doing.

- Automated tooling: none applicable.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | The core-to-edge path has no transition window in either direction: an edge message whose epoch differs from the core service's current one is dropped, whether the edge or the core is the one that rotated first, and the edge cannot tell | Data Validation | Low | Low | Open — confirms #521 at `3d5d419e`, adds the reverse direction and the in-flight case |
| LB-002 | `blend-protocol.md` § Transition Period does not say whether edge connections are covered; the code's test asserts they are not | Data Validation | Informational | — | Open |

### LB-001 · The core-to-edge path has no transition window in either direction: an edge message whose epoch differs from the core service's current one is dropped, whether the edge or the core is the one that rotated first, and the edge cannot tell

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Data Validation |
| Target | `blend/network/src/core/with_edge/behaviour/mod.rs:L126-L140` (`start_new_epoch`), `L195-L225` (verify against `self.current_epoch` / `self.proofs_verifier` only), `L232-L250` (drop); `services/blend/src/edge/mod.rs:L370-L400` (edge rotation), `edge/backends/libp2p/mod.rs:L87-L92` (backend abort on drop); `services/blend/src/core/backends/libp2p/swarm.rs:L855-L880` |
| Status | Open — same defect as #521 |

**Description**

*What the code does.* The core-to-edge behaviour holds one epoch and one verifier. `start_new_epoch` (L126-L140) replaces both and closes every upgraded edge substream, with the comment "without waiting for the transition period, so that edge nodes can retry with the new membership". A received message is verified with `self.current_epoch` and `self.proofs_verifier` captured at spawn time (L217-L223, `poq_verification.rs` L45-L57); on failure the substream is closed and nothing is reported (L242-L249). The only concession to epochs is downstream: a verification spawned *before* the rotation completes with the old epoch, and the swarm publishes the message to the old epoch's core peers if that epoch is still served (`swarm.rs` L855-L880, `with_core` L865-L880). The core-to-core behaviour, by contrast, keeps the previous verifier and peers in `old_epoch` for `epoch_transition_period` (`with_core` L305-L316).

*First checklist item: the window.* Three cases, with the edge node's rotation at `t_E`, the core node's orchestrator tick at `t_C` and the core *service's* rotation at `t_C + lag` (the lag measured in #234):

1. *Edge behind* (the issue's case): a message encapsulated under `e − 1` before `t_E` and read by the core after `t_C + lag`. The edge aborts its old backend on its own tick (`edge/mod.rs` L395, `Drop` at `backends/libp2p/mod.rs` L87-L92 aborts the swarm task), so only sends already handed to the swarm are exposed; with `max_dial_attempts_per_peer_per_message: 1` and `replication_factor: 1` there is one dial and one write per message. Window: `(t_C + lag) − t_E` plus dial-and-write latency, i.e. clock offset between the two hosts plus tens of milliseconds, *minus* the core's lag (a late core accepts old messages longer). Sub-second with working NTP.
2. *Core behind* (not in the issue): a message encapsulated under `e` after `t_E` and read by a core whose service has not yet rotated, i.e. before `t_C + lag`. The core verifies it against `e − 1` inputs and drops it. Window: `(t_C + lag) − t_E`, which is the core service's lag when clocks agree. #234 measured that lag at 0-5 ms at the shipped cover rate and 0.2-9.6 s at four times that rate on a loaded host, against an edge service whose own latch lagged by at most 5 ms in every run. This direction therefore dominates whenever the core node is busy at the boundary, and it is the one an edge leader that wins the first slot of an epoch hits.
3. *In flight*: bytes written before the core's rotation and polled after it (test (a) in §3). Dropped; the substream is closed first.

In all three cases the edge node's write succeeds (`edge/backends/libp2p/swarm.rs` L618-L620 logs "Message sent successfully" once the stream is written and closed; the core's silent drop or the close of the substream arrives after that or not at all), the failure detector waits `num_blend_layers · maximum_release_delay_in_rounds` rounds (`settings/mod.rs` L150-L170; 1 round with the shipped `num_blend_layers: 1`, `maximum_release_delay_in_rounds: 1`), and then `broadcast_undelivered_messages` publishes the proposal in the clear from the leader's own node (`edge/mod.rs` L396-L398), or the node abstains if `abstain_on_failure` is set. The spec's rule that a sender must treat an undelivered payload as lost (§ Detection) is followed; the loss itself is the deviation from § Transition Period, whose first bullet says the node "validates message proofs against both new and past epoch-related public input for the duration of TP".

*Third checklist item: the #235 protocol name.* #235 keeps the bare `protocol_name` for edge streams by design (its §2 table, "the edge behaviour keeps the bare name", and Appendix C: "Edge-to-core connections use `<protocol_name>` alone"), and in §4.3 decided against offering the previous epoch's name for inbound *core* streams during the transition because a 2 s dial retry covers the sub-6-second skews it expected, with the note "revisit only if #234's devnet measurement shows skews beyond 6 s". #234 measured 9.6 s under load. For edge streams the calculus differs: an edge node has no retry that re-encapsulates under the other epoch (a failed write at most re-dials with the same message, `swarm.rs` L326-L360, and the shipped config allows one attempt), so a name mismatch would only turn a silent drop into a visible dial failure. Offering both `…/epoch/e−1` and `…/epoch/e` for inbound edge streams during `T` would let the core learn the message's epoch at negotiation and verify with the matching verifier without a second Groth16; it needs the two-name upgrade type #235 §4.3 describes plus the previous verifier kept for `T`, which is the same state LB-001's short-term fix needs anyway. Direction 2 is not helped by any of this: a core that has not rotated cannot verify an epoch-`e` proof whatever name the stream carries. That direction is closed only by LB-002 of #234 (making the core service rotate on the tick) or by the core keeping the *next* epoch's inputs once its chain query for them succeeds.

*Fourth checklist item: tests.* `tests/epoch.rs` has two tests: `start_new_epoch_closes_all_edge_connections` asserts the close and documents "no dual-epoch for edge"; `epoch_transition_updates_membership_for_new_connections` covers membership. `tests/message_handling.rs` covers valid, invalid-PoQ, wrong-layer-count and oversized messages, all in one epoch. No test sends a message verified under `e − 1` within `epoch_transition_period` after `start_new_epoch` and expects it reported; test (b) of §3 is that test with the current expectation inverted, and it passes today as a drop.

**Exploit scenario**

Not attacker-triggered. An edge leader wins the first slot of epoch `e`, encapsulates under `e`, dials a core node whose core service is 2 s behind its tick (the #234 loaded-host figure), and sends; the core drops it; one round later the leader broadcasts the block in the clear, linking its network identity to the block, or abstains and forfeits the slot. Under the issue's original direction the same happens to a leader whose clock is behind the core's by more than the dial latency at the last slot of `e − 1`. Anything that widens the core service's lag (#72, #234 LB-002) widens direction 2 for every edge sender of the first slots of an epoch.

**Recommendation**
- *Short term*: in `with_edge`, keep the previous epoch's verifier for `epoch_transition_period` after `start_new_epoch`, do not close upgraded substreams at rotation (let their one message arrive; the `connection_timeout` of 1 s already bounds them), and on a current-epoch verification failure during the period verify once against the previous verifier and report the message under `e − 1`, which `Event::Message { epoch }` and `publish_received_edge_message` already carry through to the old-epoch peers and processor. One extra Groth16 per failed edge message for `T` per epoch. Add the test described under item 4 with the reported expectation. For direction 2, have the core service switch its edge verifier to the new epoch's inputs as soon as its own chain query has produced them, before the rest of its rotation, or fix #234 LB-002.
- *Long term*: with #235's name, offer `…/epoch/e−1` and `…/epoch/e` for inbound edge streams during `T` and pick the verifier by the negotiated name; write the rule into the spec (LB-002).

**References**: `blend-protocol.md` § Transition Period L598-L603, § Edge Network L347-L368, § Detection L446-L448, § Direct Broadcast L450-L456; #521 / #183 LB-001; #235 §4.3 and Appendix C; #234 LB-002 (core-service lag); #116 LB-001 (core-to-core counterpart).

### LB-002 · `blend-protocol.md` § Transition Period does not say whether edge connections are covered; the code's test asserts they are not

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Data Validation |
| Target | `blend-protocol.md` L598-L603; `blend/network/src/core/with_edge/behaviour/tests/epoch.rs` L18-L19 |
| Status | Open |

**Description**

§ Transition Period's four bullets say "the node validates message proofs against both new and past epoch-related public input", "must open new connections", "needs to maintain old connections and process all messages received from these connections". § Edge Network describes edge connections as opened per message and closed after it (L365-L366), so "old connections" reads naturally as the long-lived core connections of § Core Network, while "validates message proofs against both … public input" reads as a property of the node whatever the connection. The implementation picked the narrow reading and wrote it into a test comment ("no dual-epoch for edge"). Either reading is defensible; the spec should choose. Second checklist item, both drafts:

- *Covering reading (recommended, because the sender is the party with the most to lose and has no way to retry under the other epoch):* append to the first bullet: "This applies to every message the node receives during the Transition Period, including messages delivered over edge connections (§ Edge Network): an edge node's message is verified against the public input of the epoch its proofs were generated under, whether that is the current epoch or the previous one, and is forwarded to the connections of that epoch. A core node that has not yet obtained the public input of the new epoch cannot verify proofs generated under it; the Transition Period gives it the time to do so, and an edge node should therefore prefer core nodes that have announced the new epoch when one is known to it."
- *Excluding reading:* append: "Edge connections are outside this rule: they carry one message each and are closed after it, so a core node verifies an edge node's message against the current epoch only, and an edge node whose message crosses an epoch boundary must expect it to be dropped and treat it as undelivered (§ Detection)." Under this reading the edge node's direct-broadcast fallback becomes the specified outcome for every boundary-crossing proposal, which is worth stating in § Direct Broadcast as a known deanonymisation point.

The covering reading is the one #183 S-001 and #235 Appendix C assumed; #235's sentence "a connection belongs to the epoch both ends agreed at negotiation" covers core connections only and should say so ("core connection").

**Recommendation**
Raise on logos-lips (issue #451 already collects § Transition Period wording) with the covering draft; keep the excluding draft as the fallback text if the editors decide edge messages are not worth a second verification.

**References**: `blend-protocol.md` L578-L603, L347-L368, L450-L456; #183 S-001; #235 Appendix C; logos-lips #451.

## 5. Suggestions (non-security)

### S-001 · Make the edge sender's loss observable

Today the only signal of a boundary drop on the edge side is the direct broadcast one round later. A `trace` on the core when it drops an edge message for a PoQ failure during `epoch_transition_period`, with `blend_epoch`, and a counter next to `core_peer_blocked`, would let #234-style measurements count edge losses per transition; nothing in the current logs distinguishes them from a genuinely invalid proof (`poq_verification.rs` L88-L104 logs both identically).

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
