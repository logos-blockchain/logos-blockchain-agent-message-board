# Audit Report — Blend core connections are not bound to an epoch: the transition window, the verdict path, and a negotiation-level binding

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/116`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `blend/network` (core-to-core behaviour and handler), `services/blend` (libp2p core backend, membership/epoch stream), `blend/message` (proof verifier), `services/time`, `ledger` and `services/chain/chain-service` (epoch-state query only)
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `proof-of-quota.md`, `message-formatting.md`, `payload-formatting.md` (all in full); `blend-protocol.md` by section (see Method)
Date: `2026-09-12` — author: `Claude Fable 5.1 (agent)` — status: `final`

---

## 1. Summary

- Overall assessment: the premise of #116 holds unchanged at `a805329f`: a core connection is assigned to an epoch by the receiver's local state at upgrade time, nothing on the wire says which epoch the sender is in, and a PoQ generated for the other epoch is a spam verdict. This report decomposes the transition window between two honest nodes into its four components (clock offset, local latency, a failed chain query, and one newly identified deterministic cause: a block at the epoch's first slot), traces the verdict path in both directions, and designs the binding the issue asks for. The design chosen is to append the epoch number to the stream protocol name; it is prototyped in Appendix B with a behaviour test, and it needs no message-format change, no extra round trip, and no change to the dial-retry logic, which already treats a failed negotiation as "try again later, then try another member" rather than as a strike.
- Findings: 0 critical · 0 high · 1 medium (carried from #101 LB-002, re-verified) · 1 low (new) · 1 informational (new)
- Key themes: "the epoch is a receiver-side guess", "a verifier that cannot verify is not evidence of spam", "bind at negotiation, not after the first message"
- Must-fix before launch: LB-001 (bind connections to an epoch; Appendix B), together with the epoch-bounded blocklist of #117 (upstream PR #3544) which caps the damage of any remaining misclassification to one epoch.

This is the report for #116, spun out of #101 (PR #115, LB-002) and its second pass (PR #136). It does not re-derive the traffic-model estimates of #115; it re-verifies the mechanism at the current commit, adds what the first two passes did not cover, and delivers the design and prototype. Item 1 of the issue (a devnet measurement of the inter-node skew) was not done; the exact recipe is in §5 and filed as a follow-up.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `blend/network/src/core/with_core/behaviour/mod.rs` | connection upgrade and epoch assignment, `start_new_epoch`, forwarding, PoQ outcome handling, `ConnectionClosed` handling |
| `blend/network/src/core/with_core/behaviour/{old_epoch.rs,utils.rs,handler/mod.rs}` | old-epoch receive path and its verifier, PoQ dispatch, stream protocol negotiation and substream lifecycle |
| `blend/network/src/core/{mod.rs,poq_verification.rs}` | protocol name plumbing, verification outcome and its diagnostics |
| `services/blend/src/core/backends/libp2p/{swarm.rs,behaviour.rs,mod.rs,settings.rs}` | `StartNewEpoch` handling, dial selection, dial retry and failure classification, blocklist feed |
| `services/blend/src/core/mod.rs` (epoch handoff only), `services/blend/src/membership/chain.rs`, `blend/scheduling/src/epoch.rs` | when a node changes epoch, how the backend is told, the transition-period timer |
| `blend/message/src/crypto/proofs.rs`, `blend/message/src/encap/mod.rs`, `blend/message/src/message/public_header.rs` | what a verifier holds; what the public header carries |
| `services/time/src/{lib.rs,backends/common.rs,backends/ntp/mod.rs}` | how the slot clock that triggers the transition is derived |
| `ledger/src/cryptarchia/mod.rs` (`update_epoch_state`), `ledger/src/lib.rs`, `services/chain/chain-service/src/lib.rs` (`epoch_state_for_slot_with_source`) | what the epoch-state query returns and when it fails |
| `nodes/node/binary/src/config/blend/{deployment.rs,mod.rs}`, `nodes/node/binary/src/config/deployment/settings.yaml`, `deployment/ceremony/genesis/*/deployment-template.yaml` | shipped protocol name, transition period, slot parameters |
| `libp2p-swarm` 0.47.1 `src/connection.rs`, `libp2p-core` 0.43.2 `src/upgrade/ready.rs`, `libp2p/src/dial_error_ext.rs` | what a failed protocol negotiation produces on each side |

**Out of scope**
The blocklist lifetime and where blocks are enforced (#101, #117, upstream PR #3544), the observation-window verdicts (#73, #100), the old-epoch receive path's missing verdict (#205), the edge behaviour (edge connections are single-message and never blocked), PoQ circuit soundness and verifier cost (#16, #127, #152), membership derivation and the stale-membership attack (#62; its interplay with this issue is discussed under LB-001), and the traffic-model estimates of #115 LB-002 (taken as given). `libp2p` transport, `multistream-select`, `tokio`, `arkworks` and `rust-rapidsnark` are assumed correct.

**Assumptions**
Release-profile facts from #19 hold. Core peers are authenticated by the QUIC/TLS handshake, so a verdict is attributable to the peer it is issued against. The shipped parameters are those of `nodes/node/binary/src/config/deployment/settings.yaml` at the audited commit (1 s slots, `slot_activation_coeff` 1/20, `num_blend_layers` 1) unless a template value is named.

## 3. Method

- Specifications first, per the README: `bedrock-architecture-overview.md` and `overview-cryptoeconomics.md` in full; `proof-of-quota.md`, `message-formatting.md`, `payload-formatting.md` in full; `blend-protocol.md` §Time (L108-L112), §Core Network (L303-L346), §Connection Details, §Neighbor Distinction Process, §Connectivity Maintenance, §Transition Period (L535-L604), §Relaying and §Processing (L878-L931).
- Manual review of the in-scope paths at `a805329f`, working through the five items of #116 with parent #13, the originating report (#115 LB-002) and its second pass (#136), and the #117 (PR #164), #154 (PR #171) and #205 reports for context.
- Re-verification of #115 at `a805329f`: the two commits between `7b5e48b0` and `a805329f` that touch `blend/network` or the core backend (`d96e7ef99`, `ce3e2504d`) do not change any path cited here; every anchor of #115 LB-002 was re-read and is cited at its current line.
- Third-party code read from the local registry: how `libp2p-swarm` reports a failed substream negotiation to each side.
- Automated tooling run: none.
- Dynamic testing: one behaviour test written for this report and run against a scratch copy of the node tree at the audited commit with the Appendix B patch applied (`cargo test -p logos-blockchain-blend-network --lib core::with_core::behaviour::tests::epoch`); result in Appendix B. No multi-node devnet was run, so the inter-node skew is not measured (issue item 1; §5 S-001).

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Core connections carry no epoch: two honest nodes that cross the boundary at different times classify each other as spammers (re-verified; #101 LB-002) | Data Validation | Medium | Low | Open; design and prototype in Appendix B |
| LB-002 | A block at the epoch's first slot makes the Blend epoch-state query fail with `InvalidSlot`, holding the node in the old epoch for a full slot | Denial of Service | Low | High | Open |
| LB-003 | `RealProofsVerifier` documents a fallback to the previous epoch's public inputs that is not implemented | Data Validation | Informational | — | Open |

### LB-001 · Core connections carry no epoch: two honest nodes that cross the boundary at different times classify each other as spammers (re-verified; #101 LB-002)

| | |
|---|---|
| Severity | Medium (unchanged from #115) |
| Difficulty | Low |
| Category | Data Validation |
| Target | `blend/network/src/core/with_core/behaviour/mod.rs:L285-L319` (`start_new_epoch`), `L1076-L1119` / `L1125-L1168` (`handle_established_{inbound,outbound}_connection`), `L924-L954` (`forward_maybe_excluding`), `L984-L992` (`handle_poq_verification_outcome`, `Failed`); `blend/network/src/core/with_core/behaviour/handler/mod.rs:L160-L167`, `L319-L330` (single constant protocol name); `services/blend/src/core/backends/libp2p/swarm.rs:L625-L639` (`StartNewEpoch`), `L427-L435` (block on `Spammy`); `services/blend/src/membership/chain.rs:L146-L226` |
| Status | Open |

**Description**

Re-verified at `a805329f`; the mechanism is exactly as #115 described it. The facts, at their current lines:

1. *The wire carries no epoch.* The core handler negotiates a single constant protocol, `ReadyUpgrade<StreamProtocol>` built from `protocol_name` (`handler/mod.rs` L160-L167 for inbound, L319-L330 for the outbound substream request); the name is the deployment constant `blend.common.protocol_name` (`settings.yaml` L5; templates L5 carry the environment and release version in it, not the epoch). The public header is `version, public_key, proof_of_quota, signature` (`message-formatting.md` L51-L63; `public_header.rs` L10-L36): no epoch field. The PoQ public inputs that do change per epoch (`core_root`, `pol_ledger_aged`, `pol_epoch_nonce`, `pow_blend_difficulty`; `proof-of-quota.md` L57-L73) are not transmitted; they are whatever the receiver's verifier holds.
2. *The epoch of a connection is the receiver's current epoch at upgrade time.* Both upgrade paths check membership of the *current* epoch (`mod.rs` L1103, L1152), insert the connection into `connections_waiting_upgrade` (L1108-L1109, L1157-L1158) and hand the handler the constant name (L1110-L1114, L1159-L1163). `FullyNegotiated` then registers the peer in `negotiated_peers` (L597-L604), the current epoch's map.
3. *Transition order.* The service builds the new verifier from the new epoch's public inputs (`services/blend/src/core/mod.rs` L1771-L1782), sends `StartNewEpoch` over the backend channel (`backends/libp2p/mod.rs` L129-L137), and the swarm applies it (`swarm.rs` L625-L639): `start_new_epoch` closes every pending upgrade (L297-L300), installs the new verifier (L302-L303), moves the negotiated peers and their cache into `OldEpoch` with the *old* verifier (L307-L316), and the swarm immediately dials new members (L638). From that moment every connection upgraded is a new-epoch connection verified with the new verifier (`utils.rs` L125-L132 with `self.proofs_verifier`, `mod.rs` L1028-L1037); old-epoch connections are verified with the old verifier (`old_epoch.rs` L255-L263) until `CompleteEpochTransition` (`swarm.rs` L640-L642) after `epoch_transition_period` (`epoch.rs` L98-L121). That period is `slot_duration × average_slots_per_block` = 20 s with the default `slot_activation_coeff` of 1/20 (`config/blend/deployment.rs` L59-L65, `config/blend/mod.rs` L78-L80, `cryptarchia-engine/src/config.rs` L125-L131, `settings.yaml` L26-L28), not the spec's 30 rounds (`blend-protocol.md` L594).
4. *Forwarding is by epoch of connection, verification too.* Every current-epoch message is forwarded to every non-spammy current-epoch peer except its sender (L924-L954); a message arriving on a current-epoch connection is verified with the current verifier, and a Groth16 proof over the other epoch's public inputs fails by construction. The failure is `PoQVerificationOutcome::Failed` (`poq_verification.rs` L87-L103, logged at *debug* as `blend_poq_verification_failed`), which `handle_poq_verification_outcome` turns into `close_spammy_connection(.., InvalidProofOfQuota)` (L984-L992) → `Spammy` state (L729-L751) → `CloseSubstreams` → `ConnectionClosed` → `PeerDisconnected(peer, Spammy)` (L1206-L1221) → `block_peer` in the swarm (`swarm.rs` L429-L433) → excluded from every later dial (L247-L253). At `a805329f` the block is for the life of the process (#101 LB-001; epoch-bounded by upstream PR #3544, unmerged at this commit).
5. *Both directions.* Take `P` ahead of `L` by δ, and `P` dialing `L` within δ (P dials `Φ_min` members at once, L638). `L`, still in `e`, upgrades the connection as an epoch-`e` peer; `P` holds it as an epoch-`e+1` peer. `L` forwards its epoch-`e` traffic to `P`, which cannot verify it → `P` blocks `L`. `P` forwards its epoch-`e+1` traffic to `L`, which has no `e+1` inputs at all → `L` blocks `P`. The spec's transition rule ("validates message proofs against both new and past epoch-related public input", L600) cannot help `L`, which has no new input yet, and does not help `P` either, because `P` applies its old verifier only to connections it registered in the old epoch (`mod.rs` L1011-L1026), never to a current-epoch connection whose peer turns out to be behind. Once `L` transitions, its connection to `P` moves into `L`'s `OldEpoch`; any later verdict on it is ignored (`update_state_for_negotiated_peer` L763-L770), so the damage is confined to the window but is permanent (or one epoch with #3544).

*What the window δ is made of* (issue item 1, analytically; the measurement is S-001). δ is the difference between the instants the two nodes process `StartNewEpoch`, and it has four components:

| Component | Where | Size |
|---|---|---|
| Clock offset between the two slot clocks | Slot ticks fire at slot start on a timer rebased from NTP (`backends/common.rs` L15-L33; `ntp/mod.rs` L82-L100 initial local clock, L139-L200 rebase with `roundtrip/2` correction, L163-L167) | tens of ms with working NTP; the full local drift between resyncs, or seconds, if NTP fails (L67-L79 logs and continues) |
| Local latency: tick → `watch` channel (`time/src/lib.rs` L150-L160) → chain query (`chain.rs` L152) → membership rebuild → service handoff → `mpsc` of capacity 64 (`backends/libp2p/mod.rs` L56) → swarm | in-process relay and channel hops | ms, longer if the swarm task is busy or the channel holds queued `Publish` messages |
| A failed chain query | `chain.rs` L221-L226: "will retry on next slot" | exactly one slot (1 s) per failure |
| A block at the boundary slot already applied when the query runs | LB-002 | exactly one slot |

The first two are sub-slot and hit every pair at every transition; the last two add whole slots, and with the shipped `λ ≈ 1 message/s` per connection (#115 LB-002's model) one slot is enough for a message to cross with probability ≈ 64 %. #115's table stands.

*Interplay with #62 (issue item 5).* A node whose chain view lags does not fail the query; it gets an answer synthesised from its own tip: `epoch_state_for_slot_with_source` takes the tip's ledger state (`chain-service/src/lib.rs` L508-L512) and `update_epoch_state` derives the requested epoch from it (`ledger/src/cryptarchia/mod.rs` L300-L360 for the next epoch, from L361 for skipped epochs). So a lagging node latches an `e+1` state *on time* but possibly with a different `core_root`, nonce or aged root than the rest of the network, and then fails everyone's proofs and has its own proofs failed by everyone, for as long as it lags. Binding connections to the epoch *number* (Appendix B) does not catch this case; both sides agree on the number and disagree on the inputs. Binding to a digest of the public inputs would (see Recommendation, optional step). Either way the symptom is the same as this finding's and the fix for the verdict is the same: a proof the receiver cannot verify because its inputs differ from the sender's is not evidence of spam.

**Exploit scenario**

No attacker is needed; #115 gives the honest-collision rates. An attacker that can make one victim's chain query fail or return late at the boundary (a burst of blocks to apply, a relay stall) widens δ to whole slots for that victim; an attacker that is the leader of the boundary slot can trigger LB-002 on every node that receives its block before their own tick. Neither is demonstrated here.

**Recommendation**

- *Short term* (issue item 3, either side can adopt it independently):
  1. **Leading side, re-verify against the other verifier.** In `handle_poq_verification_outcome`, when `Failed` arrives for a *current-epoch* connection while `old_epoch` is `Some`, verify the proof once more with `old_epoch.proofs_verifier` (the `Failed` outcome must carry the message for this; today it carries only `sender` and `connection_id`, `poq_verification.rs` L32-L35). If it passes, move the connection from `negotiated_peers` into `old_epoch.negotiated_peers` (an insertion API is needed in `old_epoch.rs`), mark the message processed in the old cache, and emit `Event::Message { epoch: old }`. Cost: one extra Groth16 verification per failed proof, only during the 20 s transition period, so an attacker gains at most one additional verification per message for 20 s per epoch.
  2. **Lagging side, pre-compute the next verifier.** The ledger can already answer `epoch_state_for_slot` for a slot in the *next* epoch: case 2 of `update_epoch_state` builds it from `next_epoch_state` (`ledger/src/cryptarchia/mod.rs` L300-L360) for any slot after the tip. `chain.rs` can therefore issue a second query, for the first slot of `e+1`, during the last slots of `e`, and hand the backend a `next_proofs_verifier` to try when a proof fails on a current-epoch connection. The values are the ones the ledger has frozen for the next epoch; whether they can still move in the last slots depends on `EpochState::update_from_ledger` (called at L279-L282) and on reorgs, so the boundary query must still be made and must override.
  3. **Either side, the cheapest rule of all:** for the first `epoch_transition_period` after `StartNewEpoch`, treat `InvalidProofOfQuota` on a current-epoch connection as a plain close rather than a `Spammy` verdict, and never issue the verdict while `chain.rs` is in its retry state (clock epoch advanced, query not yet answered). This is what #115 proposed; it costs an attacker-free window of 20 s per epoch per connection, which a per-connection counter bounds.
- *Long term* (issue items 2 and 4): make the epoch part of the connection at negotiation time by appending it to the stream protocol name, `<protocol_name>/epoch/<n>`. Appendix B has the analysis of what each side sees, the 30-line patch and its test. Optionally append a short digest of the epoch's PoQ public inputs as well (`.../epoch/<n>/<hex8>`), which also covers the #62 case; the digest is over public values, so it leaks nothing. Put the wire-format version (`public_header.rs` L10) and `num_blend_layers` (`settings.yaml` L3) in the name too, so a version-skewed peer fails negotiation instead of producing `UndeserializableMessage` verdicts (`utils.rs` L102-L104 → `mod.rs` L1042; #115 S-004). The spec should state it: `blend-protocol.md` §Connection Details (L539) names one protocol string for the whole deployment, and §Transition Period (L598-L603) implies but never says that a connection belongs to one epoch (#136 S-002; logos-lips issue #451 from the #117 report).

**References**: `blend-protocol.md` L303-L320 (bootstrapping each epoch), L539 (protocol name), L554-L557 and L888 (invalid PoQ → malicious), L598-L603 (transition period); `proof-of-quota.md` L57-L73, L113; `message-formatting.md` L51-L63; #101 LB-001/LB-002 (PR #115), #136 LB-003 and S-002, #117 (PR #164, upstream PR #3544), #62, #205.

### LB-002 · A block at the epoch's first slot makes the Blend epoch-state query fail with `InvalidSlot`, holding the node in the old epoch for a full slot

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Denial of Service |
| Target | `services/blend/src/membership/chain.rs:L146-L152`, `L221-L226`; `services/chain/chain-service/src/lib.rs:L507-L512` (`epoch_state_for_slot_with_source`); `ledger/src/cryptarchia/mod.rs:L257-L269` (`update_epoch_state`); `ledger/src/lib.rs:L623-L633` |
| Status | Open |

**Description**

On the first tick of a new epoch the Blend membership stream queries the chain service for the epoch state *at the tick's slot* `s₀` (`chain.rs` L148-L152). The chain service answers from the tip's ledger state (`chain-service/src/lib.rs` L508-L512), and the ledger refuses any slot that is not strictly after the tip's slot:

```rust
// ledger/src/cryptarchia/mod.rs L264-L269
if slot <= self.slot {
    return Err(LedgerError::InvalidSlot { parent: self.slot, block: slot });
}
```

So if a block *for slot `s₀`* has already been applied when the query runs, the query fails, `chain.rs` logs a warning and waits for the next tick (L221-L226), and the node stays in epoch `e` for one more slot: δ = 1 s for that node, deterministically, against every peer that transitioned on time. That is the widest of the honest windows in LB-001 (≈ 64 % chance of a cross-epoch message per new connection per direction under #115's model).

When does the tip hold a block for `s₀` before the tick for `s₀` is processed? The block's leader must have produced it early relative to this node's clock, by more than the propagation delay: a clock offset of a few hundred milliseconds (NTP unavailable, LB-001 table), or a tick delivered late on this node (the time service broadcasts ticks over a `watch` channel, `time/src/lib.rs` L150-L160, and the timer skips missed ticks, `backends/common.rs` L26-L28). With the shipped `slot_activation_coeff` of 1/20 the boundary slot carries a block one time in twenty; the coincidence is rare, but every transition of a node whose clock is behind by more than the block propagation time hits it whenever the boundary slot has a block. It was not reproduced dynamically; the path is static.

A node that is itself the leader of `s₀` is on the same tick; its proposal has to be proven and applied first, so the Blend query normally wins that race, but nothing orders it.

**Exploit scenario**

The leader of a boundary slot runs its clock ahead by a few hundred milliseconds and publishes its block for `s₀` as early as the consensus tolerance allows. Every node that receives the block before its own `s₀` tick spends the first slot of the epoch in the old epoch, dialing and being dialed by nodes that have transitioned, and blocks or is blocked by them under LB-001. The effect is bounded by one slot per epoch and by how many nodes the block reaches before their ticks; the attacker's identity is the leader's, which is private by design.

**Recommendation**

- *Short term*: in `epoch_state_for_slot_with_source`, answer `slot == state.slot()` from the tip's own `epoch_state()` (the block at that slot has already run `update_epoch_state` for it) instead of propagating `InvalidSlot`; or in `chain.rs`, on `Ok(Err(InvalidSlot))` re-query immediately with `slot + 1` rather than waiting a slot. The chain-leader makes the same query for its own slot (`chain-leader/src/leadership.rs` L434) and would benefit from the same answer.
- *Long term*: the epoch-state query should take an epoch, not a slot; the membership stream only ever wants "the state for epoch `e`" and should not have to pick a slot that is guaranteed to be after the tip.

**References**: `blend-protocol.md` L303-L305 (bootstrapping "at the beginning of each epoch"); LB-001 window table.

### LB-003 · `RealProofsVerifier` documents a fallback to the previous epoch's public inputs that is not implemented

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Data Validation |
| Target | `blend/message/src/crypto/proofs.rs:L67-L69` (struct), `L74-L79` (`new`), `L86-L103` (`verify_proof_of_quota`); `blend/message/src/encap/mod.rs:L17-L37` (`ProofsVerifier` trait) |
| Status | Open |

**Description**

The production verifier's `verify_proof_of_quota` is annotated:

```rust
// blend/message/src/crypto/proofs.rs L88-L89
// Try with current input, and if it fails, try with the previous one, if any
// (i.e., within the epoch transition period).
```

but the struct holds one set of inputs (`current_inputs`, L67-L69), the trait's constructor takes one set (`encap/mod.rs` L22), and the function verifies against that set only (L86, L96-L103). The dual verification the spec asks for during the transition period (`blend-protocol.md` L600) was evidently intended to live here and was dropped when verifiers became per-epoch objects (one per `OldEpoch`, `old_epoch.rs` L48; one per behaviour, `mod.rs` L130). Nothing is wrong with the per-epoch design, but the comment now describes a protection that does not exist, and a reader auditing the transition period from this file would conclude the case of LB-001 is handled.

**Exploit scenario**

None; impact is covered by LB-001.

**Recommendation**

- *Short term*: delete the comment, or implement it where it now belongs (LB-001 short-term item 1 keeps the per-epoch verifiers and does the fallback in the behaviour, which knows which epoch a connection was registered in).
- *Long term*: a `ProofsVerifier` that holds `(current, Option<previous>, Option<next>)` inputs and reports *which* set verified would let the behaviour re-classify a connection from one verification instead of two.

**References**: `blend-protocol.md` L598-L603.

## 5. Suggestions (non-security)

### S-001 · Measure δ on a devnet (issue item 1, not done here)

The two log events needed already exist and carry the right fields: `blend_epoch_state_latched` (`chain.rs` L176-L197: `clock_epoch`, `clock_slot`, `source_tip_slot`, `component`) and `blend_poq_verification_failed` (`poq_verification.rs` L88-L99: `blend_epoch`, `peer_id`, `sender_signing_key`), plus `blend_peer_negotiation_failure` (`swarm.rs` L722-L749). Both are at `debug`, so the run needs `debug` for `lb_log_targets::blend::*` and the `BLEND_REACHABILITY` diagnostic. Recipe: run `deployment/compose.yml` with ≥ 4 core nodes for ≥ 3 epoch transitions (shorten the epoch via `security_param`/`slot_activation_coeff` in the deployment template); per transition, δ between nodes is the spread of `blend_epoch_state_latched` timestamps for the same `clock_epoch`; a chain-query retry shows as `clock_slot` > first slot of the epoch together with the `will retry on next slot` warning (`chain.rs` L222, L225), whereas clock offset shows as sub-slot spread with the same `clock_slot`; then count `blend_poq_verification_failed` in the 30 s after each boundary and join on `peer_id` to see which pairs blocked each other. Report the distribution of δ and the fraction of transitions with at least one cross-epoch failure. Filed as a follow-up.

### S-002 · Make cross-epoch failures visible without `debug` logging

`blend_poq_verification_failed` is the only trace of the honest collision and it is at `debug`. A counter metric `blend_poq_verification_failed_total{epoch_of_connection, reason}` next to `core_peer_blocked` (`swarm.rs` L432) would show the problem on any node's dashboard, and a `PeerBlocked`-style event carrying the epoch would let #117's observability follow-up cover it.

### S-003 · Spec: state that a core connection belongs to one epoch and that the protocol string carries it

`blend-protocol.md` §Connection Details (L539) fixes one protocol string per deployment; §Core Network (L303-L320) rebuilds connections every epoch; §Transition Period (L602-L603) says a node opens new connections for the new epoch and keeps the old ones for the period. The rule that follows, "a connection belongs to the epoch both ends agreed at negotiation, a node sends on it only messages of that epoch, and it is verified with that epoch's inputs", is nowhere stated, and L600 reads as a per-message rule that the behind node cannot satisfy (#136 S-002). Appendix B's naming (`<name>/epoch/<n>`) is the smallest change that makes the rule enforceable and should be written into §Connection Details. Raised upstream as part of logos-lips issue #451 by the #117 report; this adds the concrete wording.

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

## Appendix B — Design and prototype: the epoch in the stream protocol name (issue items 2 and 4)

### B.1 Why the protocol name and not a first frame

Two options were in the issue. Sending the epoch as the first frame after upgrade needs the handler to hold back `FullyNegotiated` (emitted today as soon as a substream is negotiated, `handler/mod.rs` L377, L393) until the frame has been exchanged, a new `ToBehaviour::EpochMismatch` event, a new `ConnectionUpgradeFailureReason` variant, one extra round trip, and it touches the substream state machine that #60 is about. Putting the epoch in the protocol name needs none of that: `multistream-select` already refuses a protocol the listener does not offer, and the existing failure path already does the right thing on both sides.

### B.2 What each side sees on a mismatch (verified in `libp2p-swarm` 0.47.1)

Let `P` be at epoch `e+1` and `L` at `e`; `P` dials `L`.

- `P`'s handler requests an outbound substream with `<name>/epoch/e+1` (L319-L330). `L` offers only `<name>/epoch/e` for inbound streams (`listen_protocol`, L165-L167), so `multistream-select` answers "na": `P`'s connection task delivers `ConnectionEvent::DialUpgradeError(StreamUpgradeError::NegotiationFailed)` (`libp2p-swarm/src/connection.rs` L338-L342), and `P`'s handler closes its substreams (L396-L400).
- On `L`'s side, an inbound stream whose negotiation fails is logged at debug and dropped without any handler event (`connection.rs` L366-L368). Symmetrically, `L`'s own handler requested an outbound substream with `<name>/epoch/e` (L319-L330), which `P` refuses; `L` gets the same `DialUpgradeError` and closes.
- With no substream on either side the connection idles out (`with_idle_connection_timeout(Duration::ZERO)`, `swarm.rs` L186-L191). Each behaviour sees `ConnectionClosed` for an entry still in `connections_waiting_upgrade` and emits `…ConnectionUpgradeFailed { reason: ConnectionFailure }` (`mod.rs` L1187-L1204). Nothing is registered in `negotiated_peers`, so no message is ever exchanged and no verdict can be issued.
- `P`'s swarm handles `OutboundConnectionUpgradeFailed { ConnectionFailure }` by `schedule_retry` (`swarm.rs` L482-L502): attempt 2 after 2 s, attempt 3 after 4 s (L826-L832), then the peer goes into the cycle's `failed_peers` and another member is dialed (L496-L501). `max_dial_attempts_per_peer` defaults to 3 (`config/blend/serde/core.rs` L57). A lagging `L` therefore costs `P` at most 6 s on that one slot while the other `Φ_min − 1` dials proceed, and `L` is usually reachable again at the first retry, since δ is normally below 2 s. `DialErrorExt::is_recoverable` (`libp2p/src/dial_error_ext.rs` L8-L34) is not involved: this is a substream failure, not a dial error. Issue item 2's check, "the refusal is treated as try another member, not as a strike", holds.
- `L`'s swarm handles `InboundConnectionUpgradeFailed` with a trace log only (L526-L534).

Two connections to the same peer in different epochs already coexist by design (`PeerCondition::Always`, L348-L350; the duplicate checks L1095, L1144 look only at the current epoch's maps), so per-epoch names change nothing there. Pending handlers of the old epoch are already closed at the transition (L297-L300), so a handler can be given its epoch's name at creation and never needs to change it.

Optional refinement, not prototyped: during the transition period the leading node `P` could also *offer* `<name>/epoch/e` for inbound streams and register a connection negotiated with it directly in `OldEpoch` (it holds the `e` verifier, `old_epoch.rs` L48). That needs an upgrade type whose `UpgradeInfo::protocol_info` lists two names and whose `upgrade_inbound` returns the negotiated `Info` (`libp2p-core/src/upgrade/ready.rs` L54-L66 shows where the name is available), an insertion API in `old_epoch.rs`, and the old membership for the check at L1103. It saves a lagging dialer the 2 s retry and matches L602-L603 of the spec literally. Do the simple version first.

Naming: `<protocol_name>/epoch/<n>` keeps the deployment string (environment and release version, templates L5) as the prefix. Adding the wire-format version and `num_blend_layers` (`<protocol_name>/v1/l1/epoch/<n>`) is a one-line extension of `epoch_protocol` below and closes #115 S-004 (issue item 4). Adding a digest of the epoch's PoQ public inputs (`.../epoch/<n>/<hex8>`) would extend the binding to the #62 case (two nodes on the same epoch number with different inputs); it needs the digest computed once per epoch from `PoQVerificationInputsMinusSigningKey`, which is `Serialize` (`proofs.rs` L26).

### B.3 Patch (against `a805329f`, `blend/network` only)

```diff
--- a/blend/network/src/core/with_core/behaviour/mod.rs
+++ b/blend/network/src/core/with_core/behaviour/mod.rs
@@ fn try_wake(&mut self) { ... }
+    /// The stream protocol a core connection of the current epoch is upgraded
+    /// with: the deployment protocol name with the epoch number appended.
+    ///
+    /// Two nodes that are not in the same epoch therefore fail stream
+    /// negotiation instead of upgrading a connection on which neither side can
+    /// verify the other's proofs of quota, and the dialer moves on to another
+    /// member (or retries this one once it has transitioned) without any spam
+    /// verdict being issued in either direction.
+    fn current_epoch_protocol(&self) -> StreamProtocol {
+        epoch_protocol(&self.protocol_name, self.current_epoch_info.1)
+    }
@@ fn handle_established_inbound_connection ... (L1110-L1114)
             Either::Left(ConnectionHandler::new(
                 ConnectionMonitor::new(self.observation_window_clock_provider.interval_stream()),
-                self.protocol_name.clone(),
+                self.current_epoch_protocol(),
                 (peer_id, connection_id),
             ))
@@ fn handle_established_outbound_connection ... (L1159-L1163)
             Either::Left(ConnectionHandler::new(
                 ConnectionMonitor::new(self.observation_window_clock_provider.interval_stream()),
-                self.protocol_name.clone(),
+                self.current_epoch_protocol(),
                 (peer_id, connection_id),
             ))
@@ before `update_connection_id_and_direction`
+/// Builds the stream protocol name for core connections of `epoch`:
+/// `<protocol_name>/epoch/<n>`.
+pub(crate) fn epoch_protocol(protocol_name: &StreamProtocol, epoch: Epoch) -> StreamProtocol {
+    StreamProtocol::try_from_owned(format!(
+        "{}/epoch/{}",
+        protocol_name.as_ref(),
+        u32::from(epoch)
+    ))
+    .expect("a valid protocol name with a numeric path segment appended is a valid protocol name")
+}
```

The edge behaviour keeps the bare `protocol_name` for edge connections (`core/mod.rs` L57-L62), which are single-message and epoch-checked by their verifier; its name is therefore distinct from every core name.

Test added to `blend/network/src/core/with_core/behaviour/tests/epoch.rs`:

```rust
/// A dialer that has already moved to epoch 1 cannot upgrade a connection to a
/// listener still in epoch 0: the two sides offer different stream protocol
/// names, so negotiation fails, the connection closes as `ConnectionFailure`
/// on the dialer's side, and neither side registers a negotiated peer or has
/// any message to issue a verdict on. Once the listener transitions, the same
/// dial upgrades.
#[test(tokio::test)]
async fn epoch_mismatch_fails_negotiation_instead_of_upgrading() {
    let (mut identities, nodes) = new_nodes_with_empty_address(2);
    let mut dialer = TestSwarm::new(&identities.next().unwrap(), |id| {
        BehaviourBuilder::new(id).with_membership(&nodes).build()
    });
    let mut listener = TestSwarm::new(&identities.next().unwrap(), |id| {
        BehaviourBuilder::new(id).with_membership(&nodes).build()
    });
    listener.listen().with_memory_addr_external().await;

    // The dialer moves to epoch 1; the listener stays in epoch 0.
    let memberships = build_memberships(&[&dialer, &listener]);
    dialer.behaviour_mut().start_new_epoch(
        (memberships[0].clone(), Epoch::new(1)),
        TestProofsVerifier::accepting(),
    );

    dialer.connect(&mut listener).await;

    let mut dialer_failed = false;
    let mut listener_closed = false;
    loop {
        select! {
            event = dialer.select_next_some() => match event {
                SwarmEvent::Behaviour(Event::OutboundConnectionUpgradeFailed {
                    peer,
                    reason: ConnectionUpgradeFailureReason::ConnectionFailure,
                }) => {
                    assert_eq!(peer, *listener.local_peer_id());
                    dialer_failed = true;
                }
                SwarmEvent::Behaviour(Event::OutboundConnectionUpgradeSucceeded(_)) => {
                    panic!("a connection must not be upgraded across epochs");
                }
                _ => {}
            },
            event = listener.select_next_some() => match event {
                SwarmEvent::Behaviour(Event::InboundConnectionUpgradeSucceeded(_)) => {
                    panic!("a connection must not be upgraded across epochs");
                }
                SwarmEvent::ConnectionClosed { peer_id, .. } => {
                    assert_eq!(peer_id, *dialer.local_peer_id());
                    listener_closed = true;
                }
                _ => {}
            },
        }
        if dialer_failed && listener_closed {
            break;
        }
    }
    assert!(dialer.behaviour().negotiated_peers.is_empty());
    assert!(listener.behaviour().negotiated_peers.is_empty());
    assert!(dialer.behaviour().connections_waiting_upgrade.is_empty());
    assert!(listener.behaviour().connections_waiting_upgrade.is_empty());

    // The listener catches up: the same dial now upgrades.
    listener.behaviour_mut().start_new_epoch(
        (memberships[1].clone(), Epoch::new(1)),
        TestProofsVerifier::accepting(),
    );
    dialer.connect_and_wait_for_upgrade(&mut listener).await;
    assert_eq!(dialer.behaviour().negotiated_peers.len(), 1);
    assert_eq!(listener.behaviour().negotiated_peers.len(), 1);
}
```

### B.4 Verification

Run on a scratch copy of the node tree at `a805329f`, toolchain `rustc 1.98.1 (48a229cea 2026-09-01)`, macOS arm64, `cargo test -p logos-blockchain-blend-network --lib`:

| Step | Command filter | Result |
|---|---|---|
| Negative control: new test on the **unpatched** `mod.rs` | `…tests::epoch::epoch_mismatch_fails_negotiation_instead_of_upgrading` | FAILED as expected: panicked at the `OutboundConnectionUpgradeSucceeded` arm ("a connection must not be upgraded across epochs"), i.e. today a dialer one epoch ahead upgrades and registers the peer |
| Patched: epoch and bootstrapping tests | `…tests::epoch`, `…tests::bootstrapping` | 27 passed, 0 failed (the new test included; the mismatch path takes ≈ 10 s, the test swarm's default idle timeout, before `ConnectionClosed`) |
| Patched: whole crate | (no filter) | 63 passed, 0 failed, 3 ignored (the three pre-existing observation-window tests) |
| Patched: swarm-level backend tests | `logos-blockchain-blend-service --lib core::backends::libp2p` | 10 passed, 2 ignored, **1 failed**: `redials::core_does_not_give_up_below_minimum_peering_degree` (see below) |

The one swarm-level failure is the patch doing its job on a fixture that is itself cross-epoch: the test builds the reachable listener at the fixture default epoch 1 (`tests/utils.rs` L140-L145) and sends `StartNewEpoch` with epoch 2 to the dialer only (`tests/redials.rs` L399-L409), then asserts the dialer negotiated the listener (L437). With the patch the dialer proposes `…/epoch/2`, the listener offers `…/epoch/1`, negotiation fails, and the dialer (built with one dial attempt) records the peer as failed and keeps looking, which is the behaviour B.2 predicts. Confirmed deterministic: three runs each, the test passes on the unpatched behaviour and fails on the patched one at the same assertion. The fixture needs the listener rotated to epoch 2 as well; that one-line change belongs with the upstream PR and is listed in the implementation follow-up. The other ten backend tests, which either use a single epoch or start no connections across epochs, pass.

The test binaries and scratch tree were not kept; the patch above and the test are sufficient to reproduce.

### B.5 What the patch does not do

It does not change what happens to a cross-epoch message that arrives on an already-upgraded connection; with the patch there is no such connection, but the short-term rules of LB-001 remain worth having as defence in depth for the #62 case. It does not change the edge path. It does not touch the swarm-level tests (`services/blend/src/core/backends/libp2p/tests`), whose behaviours are built by the same constructor and therefore agree on the name. Upstreaming should add the wire-format version and `num_blend_layers` to the name in the same change and update the spec (S-003).
