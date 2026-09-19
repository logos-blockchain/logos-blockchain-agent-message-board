# Audit Report — Fate of never-released block proposals at Blend epoch rotation

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/320`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `bfcae04d25218d878eb77c3c072cb0e88524de82` — component(s): `services/blend/src/edge`, `services/blend/src/core`, `services/blend/src/pending.rs`, `services/blend/src/delivery`, `services/blend/src/orchestrator`, `blend/provers`, `services/chain/chain-leader`
Specs: `https://github.com/logos-co/logos-lips` @ `75d3d0382604d4a0d8e246c268935dd386ffc8ea` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `blend-protocol.md`, `bedrock-service-declaration-protocol.md` (all in full); `bedrock-v1.1-block-construction.md` §Proposal Construction, §Block Proposal Reconstruction, §Block Proposal Validation; `cryptarchia-v1-protocol.md` §Constants, §Epoch, §Uncle References, §Block Header Validation; `proof-of-quota.md` §Overview, §Zero-Knowledge Proof Statement; `cryptarchia-proof-of-leadership.md` §Zero-knowledge Proof Statement, §Linking the Proof of Leadership to a Block (by section)
Date: `2026-09-16` — author: `Claude Fable 5.1 (Claude Code)` — status: `final`

---

## 1. Summary

- Overall assessment: The specification does not define what happens to a block proposal that a node has handed to Blend but never released before the epoch turns, and the implementation resolves the gap by discarding it silently in every mode. The discard is keyed on the Blend epoch current when the proposal *arrived*, not on the block's slot, which at the epoch boundary produces the quota misuse the discard is meant to prevent. In one case the discard also contradicts the specification's own fallback rule: a proposal still queued when the membership drops below the minimum network size is dropped instead of being broadcast directly.
- Findings: `0` critical · `0` high · `0` medium · `2` low · `0` informational
- Key themes: epoch binding by arrival rather than by slot, no fallback for payloads that never reached the network, a fallback the spec mandates and the code skips.
- Must-fix before launch: none. LB-002 is a small, self-contained deviation; LB-001 and S-001 need a protocol decision first.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/blend/src/edge/mod.rs`, `edge/current_epoch.rs`, `edge/handlers.rs` | Edge event loop, per-epoch proposal queue, handler rebuild on epoch and secret-info events. |
| `services/blend/src/pending.rs` | `PendingProposals` (epoch-bound) and `PendingTransactions` (not epoch-bound). |
| `services/blend/src/core/mod.rs`, `core/epoch_stages/*.rs`, `core/delivery.rs` | Core-mode rotation, transition period, and the encapsulated-but-unreleased bookkeeping, for parity with edge. |
| `services/blend/src/delivery/failure_detection.rs`, `delivery/mod.rs` | When a payload starts being watched for direct broadcast. |
| `services/blend/src/orchestrator/instance.rs`, `mode.rs`, `broadcast/mod.rs` | Mode switch when the membership falls below the minimum. |
| `services/blend/src/membership/chain.rs`, `blend/scheduling/src/epoch.rs` | When Blend's epoch turns relative to the slot clock. |
| `blend/provers/src/provers/leader/mod.rs`, `blend/provers/src/crypto/leader/send.rs`, `services/blend/src/edge/settings.rs` | How leadership proofs are drawn per winning slot and per proposal. |
| `services/chain/chain-leader/src/lib.rs`, `leadership.rs`, `blend.rs` | When a proposal is built, applied locally, and handed to Blend; the winning-slot scan Blend consumes. |

**Out of scope**

Restart durability and the mode-switch drain race are covered by issue #172 (report PR #318, LB-001 and LB-003) and are not re-derived here. Membership derivation, the PoQ and PoSel circuits, the libp2p swarms, gossipsub, chain synchronisation internals, and the third-party crates `tokio`, `overwatch`, `libp2p`, `ark-groth16`, `rust-rapidsnark` are assumed correct.

**Assumptions**

The Logos LIPs text at the stated commit is the reference. The host is not compromised. `abstain_on_failure = false` where the direct-broadcast fallback is discussed. Shipped parameters are taken from `nodes/node/binary/src/config/deployment/settings.yaml` and `deployment/ceremony/genesis/testnet/deployment-template.yaml` (one blend layer, replication factor 0 or 1, one-round maximum release delay, transition period of one expected block interval, i.e. 30 rounds).

## 3. Method

- Manual review of the in-scope paths, working through issue `#320` (parent `#14`, source audit `#172`, report PR `#318` suggestion S-001).
- Spec conformance against `blend-protocol.md` §Fallback (L163-166), §Failure Detection and Reaction (L442-457), §Transition Period (L578-604), §Leadership Quota (L636-671), §Proof of Work Quota (L673-685), §Quota Application (L689-706), §Generation (L858-876), §Releasing (L953-996); `bedrock-v1.1-block-construction.md` §Block Proposal Reconstruction (L299-347) and §Block Proposal Validation (L349-381); `cryptarchia-v1-protocol.md` §Constants (L95-105), §Uncle References (L312-391), §Block Header Validation (L393-456); `proof-of-quota.md` §Overview (L37-49) and §Constraints (L105-147).
- Automated tooling: `cargo test -p logos-blockchain-blend-service --lib edge::tests` (Rust 1.98.1, aarch64) on the target commit plus two local evidence tests appended to `services/blend/src/edge/tests/mod.rs` in the scratch checkout (reproduced in Appendix B). Result: see Appendix B.
- Dynamic testing: none beyond the unit tests above.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Queued proposals are bound to the Blend epoch at arrival, not to the block's slot, so an epoch-boundary proposal is either blended under the next epoch's leadership allowance or discarded | Timing | Low | High | Open |
| LB-002 | Spec deviation: proposals still queued when the membership falls below the minimum network size are discarded instead of broadcast directly | Consensus | Low | High | Open |

### LB-001 · Queued proposals are bound to the Blend epoch at arrival, not to the block's slot, so an epoch-boundary proposal is either blended under the next epoch's leadership allowance or discarded

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Timing |
| Target | `services/blend/src/edge/mod.rs:L376-L397, L414-L417` (`run`); `services/blend/src/edge/current_epoch.rs:L125-L143` (`CurrentEpoch::try_new`); `services/blend/src/pending.rs:L101-L112, L159-L170` (`PendingProposals`); `services/blend/src/core/mod.rs:L1186, L1446-L1448, L1742-L1748` (`rotate`, `handle_epoch_event`); `blend/provers/src/provers/leader/mod.rs:L99-L109` (`create_proof_stream`); `services/blend/src/edge/settings.rs:L52-L66` (`epoch_leadership_quota`); `services/chain/chain-leader/src/lib.rs:L459-L531, L711-L733`; `services/blend/src/membership/chain.rs:L146-L152` |
| Status | Open |

**Description**

Both Blend modes keep locally produced proposals in a `PendingProposals` queue that belongs to the epoch the service is on. The queue is created with the epoch of the public epoch info that started it (`current_epoch.rs:L137-L141`, `core/epoch_stages/running.rs:L61-L73`) and every proposal that arrives is pushed into *whatever queue is current* (`edge/mod.rs:L414-L417`; `core/mod.rs:L1186, L1260-L1263`). On the next `EpochEvent::NewEpoch` the old queue is dropped together with the old handler, logging a warning (`edge/mod.rs:L392-L396`; `pending.rs:L159-L170`; `core/mod.rs:L1446-L1448`). The stated reason is quota accounting:

```
pending.rs:104   /// **Bound to the epoch it was queued in.** A proposal is built for one slot,
pending.rs:105   /// and leadership quota is one message's worth per winning slot, so a proposal
pending.rs:106   /// still waiting when the epoch turns would spend the quota that the *new*
pending.rs:107   /// epoch's block needs.
```

That reasoning is about the *slot the proposal was built for*, but the binding is decided by *when the proposal reaches the Blend service*, and the two do not agree at the epoch boundary:

1. Blend rotates its epoch on the first slot tick of the new epoch whose chain query succeeds (`membership/chain.rs:L84-L89, L148-L152`), and the rotation is applied as soon as the event is polled (`blend/scheduling/src/epoch.rs:L98-L107`).
2. The chain leader learns that it won slot `s` on the slot tick for `s`, then fetches the tip ledger state, builds a Groth16 proof of leadership in a blocking task, builds the block, applies it locally, and only then relays the proposal to Blend (`chain-leader/src/lib.rs:L459-L531, L711-L725`). Whenever this takes longer than the remaining slots of epoch `e` (a Groth16 proof plus block build, apply and relay; this report did not measure it, but nothing bounds it to one slot), the proposal reaches Blend after Blend has already rotated to `e+1`, and is queued in `e+1`'s queue.
3. Leadership proofs are drawn from a lazy stream that yields exactly `message_quota` proofs per winning slot, consumed in slot order from the start of the scan (`blend/provers/src/provers/leader/mod.rs:L99-L109`; `chain-leader/src/leadership.rs:L272-L279, L479-L497`), where `message_quota = num_blend_layers · (1 + data_replication_factor)` is also exactly what one proposal consumes (`edge/settings.rs:L52-L66`; `crypto/leader/send.rs:L159-L180`). The stream does not match proofs to the proposal's slot.

So the epoch-`e` proposal that arrives after rotation is encapsulated under a PoQ whose witness is the node's *first* winning slot of `e+1`, and goes out as a valid `e+1` message. This is exactly the outcome the queue's documentation says must not happen: the allowance of that `e+1` slot is spent on an `e` block, every later `e+1` proposal shifts to the following winning slot's allowance, and the proposal for the node's *last* win of `e+1` finds the stream ended (`leader/mod.rs:L74-L88`), stays queued with `ProofNotAvailable` retries, and is discarded at the `e+2` rotation. If the node wins nothing in `e+1`, the `e` proposal itself is the one discarded. The specification defines the leadership allowance per epoch as `Q_L^n = x · (β_D + β_D · R_D)` with `x` the wins in that epoch (`blend-protocol.md:L666-L671`), so one block per boundary crossing is sent outside the allowance the protocol intends for it, and one block is lost.

The mirror case exists but is rarer: if Blend's rotation lags (the chain query on the first slot of `e+1` fails and is retried on the next slot, `membership/chain.rs:L221-L226`), a proposal for the first slot of `e+1` is queued under `e`, encapsulated under `e`'s leadership allowance if any proofs are left, and otherwise dropped at the rotation.

Whichever way the binding goes, a proposal that is dropped at rotation is never registered with the delivery failure detector, because registration happens only after a copy has been sent (`edge/mod.rs:L442-L445`; `core/delivery.rs:L47-L72`), so the direct-broadcast fallback of `blend-protocol.md` §Direct Broadcast never fires for it. The chain leader has already applied the block locally (`chain-leader/src/lib.rs:L716-L722`), so the node sits on a tip nobody else has seen until chain synchronisation, tip polling by peers, or a reorg away from it resolves the situation. The block can still be picked up as an uncle by a later block only while its parent is within `W · f⁻¹ = 360` slots of that block (`cryptarchia-v1-protocol.md:L104, L326`).

**Exploit scenario**

No attacker is needed. A node with a modest number of wins per epoch proposes for a slot in the last few seconds of epoch `e`; the proposal reaches Blend after the rotation to `e+1`; it is blended under the allowance of the node's first `e+1` win; at the end of `e+1` the node's last proposal of that epoch cannot obtain proofs and is dropped at the `e+2` rotation with only a warning in the log. Over many epochs a leader that regularly wins near epoch ends loses about one block per affected boundary, without any signal other than the log line. The impact is a lost block for that leader and a temporary local fork (the block was applied locally); it degrades liveness and the leader's rewards, not safety.

**Recommendation**

- *Short term*: derive the epoch a proposal belongs to from the block header's slot (available in the proposal bytes the leader hands over) rather than from the Blend epoch at arrival, and route a proposal whose slot epoch is already behind Blend's epoch to whatever the protocol decides in S-001 (drop, release under old-epoch keys during the transition period, or direct broadcast) instead of into the new epoch's queue. Emit a metric alongside the existing warning so operators can see drops.
- *Long term*: the specification should say how a leader's allowance is consumed at the boundary (see S-001). If the answer is "an epoch's allowance is only for that epoch's blocks", the proof stream should draw proofs keyed by the proposal's winning slot rather than by scan order, so an `e` block can never consume an `e+1` slot's allowance. Add a test that hands a proposal to the service after a rotation and asserts which epoch's proofs back it.

**References**: `blend-protocol.md` §Leadership Quota (L636-671), §Transition Period (L578-604), §Direct Broadcast (L450-457); `proof-of-quota.md` §Overview (L44-46), §Constraints Step 3 and Step 5 (L116-145); `cryptarchia-v1-protocol.md` §Constants (L104), §Uncle References (L326); report PR #318 (issue #172) S-001.

### LB-002 · Spec deviation: proposals still queued when the membership falls below the minimum network size are discarded instead of broadcast directly

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Consensus |
| Target | `services/blend/src/edge/mod.rs:L377-L384` (`run`, `Err(Error::NetworkIsTooSmall)` arm); `services/blend/src/edge/current_epoch.rs:L129-L135`; `services/blend/src/orchestrator/instance.rs:L136-L157, L169-L171` (`transition`); `services/blend/src/mode.rs:L59-L60`; `services/blend/src/broadcast/mod.rs:L243-L254` |
| Status | Open |

**Description**

`blend-protocol.md` §Fallback (L163-166) is unconditional: "If the minimal network size is not reached, nodes must not use the Blend protocol. In such cases, nodes must broadcast data messages directly, bypassing the Blend network." The edge bootstrapping logic repeats it (L359).

When a new epoch's membership is smaller than the minimum, the edge service maps it to `ModeMembership::Broadcast` (`mode.rs:L59-L60`), `CurrentEpoch::try_new` returns `Error::NetworkIsTooSmall` (`current_epoch.rs:L129-L135`), and the run loop drains only the failure detector, i.e. only payloads that were already *sent*, then returns (`edge/mod.rs:L377-L384`). The `current_epoch` value holding the queued proposals is dropped on return, with the `PendingProposals::drop` warning (`pending.rs:L159-L170`). Meanwhile the orchestrator stops the edge instance and starts a broadcast instance for the new epoch (`instance.rs:L151-L157, L169-L171`), which would have broadcast exactly those proposals directly (`broadcast/mod.rs:L254`). The queued proposals are never handed over. The same holds for the core service leaving core mode into broadcast: `rotate` drops the queue (`core/mod.rs:L1446-L1448`) and the retiring stage carries no proposals.

Unlike the general "never-released at rotation" question (S-001), there is no privacy argument here: from this epoch on the node broadcasts everything in the clear anyway, and the specification already requires it. The proposal that is lost is the one built in the last slots of the epoch before the membership shrank, or one that waited for leadership proofs that never came.

**Exploit scenario**

Not attacker-triggered in practice, since an attacker would need to push the SDP membership below the minimum. Operationally: the membership for epoch `e+1` falls below `minimum_network_size` (a small testnet, or several withdrawals landing at once); a leader that won one of the last slots of `e` still has that proposal queued when the epoch turns; the edge service exits, the broadcast service starts, and the proposal is discarded. The block was applied locally, so the node forks until it reorgs, and the block is lost to the chain unless peers fetch it through sync. The local evidence test `audit_320_queued_proposal_is_dropped_when_membership_falls_below_minimum` in Appendix B reproduces the discard with the fallback enabled.

**Recommendation**

- *Short term*: in the `NetworkIsTooSmall` arm (and in the equivalent core-to-broadcast path), dispatch every queued proposal through the payload dispatcher before returning, since the node is entering direct-broadcast mode for that epoch anyway. Something along the lines of:

  ```rust
  // edge/mod.rs, in the `Err(Error::NetworkIsTooSmall(_))` arm, before returning
  for proposal in current_epoch.proposals_mut().drain() {
      payload_dispatcher.dispatch(DataPayload::BlockProposal(proposal)).await;
  }
  ```

  (`PendingProposals` has no `drain` today; it would need one.) Whether pending *transactions* should be dispatched too follows from the same spec sentence, but transactions are not epoch-bound and are the subject of #172 LB-001, so they are left out here.
- *Long term*: make the mode switch hand the outgoing mode's unsent queue to the incoming mode (edge to broadcast, core to broadcast, core to edge), so the fallback rule holds regardless of which mode held the queue. This is the "single delivery owner" direction of report PR #318 S-003.

**References**: `blend-protocol.md` §Fallback (L163-166), §Edge Network › Bootstrapping step 2 (L359); report PR #318 (issue #172) LB-003 and S-003.

## 5. Suggestions (non-security)

### S-001 · The specification should state the fate of a data message that was generated for an epoch but never released before the epoch turned

This is the clarification issue #320 asks for. What the current text establishes, and what it leaves open:

- **Established.** `T_M` and the direct broadcast are counted "from the round in which the message was released" (`blend-protocol.md:L448`), so a payload that was never released has no deadline and no direct-broadcast obligation. The transition period exists so that "messages for the past epoch" can "safely exit the network" and after it "messages for the past epoch must not be processed anymore" (L580-594). The leadership allowance is per epoch and per win (L666-671). The PoQ leader branch is bound to the epoch's nonce and aged ledger root and to a winning slot, not to the proposed block (`proof-of-quota.md:L116-L145`), and "the set of PoQ for leaders must be precomputed for each epoch" (`blend-protocol.md:L773`).
- **Open.** Nothing says whether a *sender* may still release a message built under epoch `e`'s keys during `e+1`'s transition period. Verifiers accept both epochs' public inputs for the whole period (L600-601), so a release up to `T − T_M = 30 − 15 = 15` rounds into the period would still be processed by every honest node. Nothing says whether a block of epoch `e` may be sent under epoch `e+1`'s leadership allowance (the implementation does so by accident, LB-001), or under a proof-of-work allowance (`Q_W`, L673-685, is "attached to no node"; §Generation step 1.1 (L869) says "each key uses a message-type-specific allowance" without saying which payload types a PoW key may back). And nothing says what a node should do with a proposal that never obtained proofs at all.

The three behaviours available are, with what a specification would have to fix for each:

1. **Discard (current implementation).** Cheapest and privacy-neutral, but a valid block is lost. The spec should say so explicitly ("a data message not released before the end of its epoch is discarded") so that the implementation's warning is a documented outcome, and should say whether the allowance drawn for a partially built message is considered spent (it is not: the nullifiers never reach the network).
2. **Release under the old epoch's keys during the transition period.** Protocol-valid today from the verifier side. The spec would have to bound the release to the first `T − T_M` rounds of the period (so the sender's own `T_M` detection still lands inside the window peers keep the old inputs for), and the implementation would have to keep the old epoch's prover, membership, and, for edge nodes, the old epoch's stream protocol alive for that long instead of dropping them at rotation (`core/mod.rs:L1742-L1745`; `edge/current_epoch.rs:L171-L182`). The leadership allowance consumed is the correct epoch's.
3. **Direct broadcast at rotation.** If chosen, the spec needs: (a) a *synthetic reference time*: to preserve "a directly broadcast payload never reaches the network earlier than a delivered one" and the transaction-maturity assumption of `bedrock-v1.1-block-construction.md` §Block Proposal Reconstruction (L301-303), the broadcast must not happen before `build_time + T_M`, and at the rotation otherwise, i.e. `max(rotation, build_time + T_M)`; (b) a *privacy statement*: a payload that never entered the network has no cover at all, so this reveals the leader immediately; the existing `abstain_on_failure` opt-out (L452) should cover it; (c) *freshness*: header validation accepts a past slot (`cryptarchia-v1-protocol.md:L429`), so the block stays valid, but it only remains useful while a descendant or an uncle reference is still possible, i.e. within `W · f⁻¹ = 360` slots of its parent (L104, L326); the spec should say whether a broadcast older than that is still worth doing; (d) *quota*: no allowance is consumed by a message that was never encapsulated, so no accounting rule is needed.

Whatever is chosen, the edge and core services should behave the same way, and the epoch a proposal belongs to should be its slot's epoch (LB-001).

**References**: `blend-protocol.md` L448, L452-456, L578-604, L636-685, L869, L773; `bedrock-v1.1-block-construction.md` L301-303, L349-381; `cryptarchia-v1-protocol.md` L104, L326, L429.

### S-002 · Dropped proposals are only visible as a warning

`PendingProposals::drop` logs a count per epoch (`pending.rs:L159-L170`) and `drop_unreleased_payloads_for_epoch` logs at debug (`core/delivery.rs:L78-L88`), while the direct-broadcast path has a metric (`delivery/mod.rs:L46-L47`). A `data_payload_dropped` counter by payload type and reason (epoch rotation, membership below minimum, unencapsulatable) would let operators notice LB-001 and LB-002 in the field, and logging the block id of a dropped proposal (as the TODO in `delivery/mod.rs:L38-L40` already asks for the fallback path) would make the log line actionable.

## 6. What was checked and ruled out

- **Core mode after a proposal is encapsulated.** A message encapsulated under epoch `e` and queued in `e`'s scheduler is still released during the transition period under `e`'s protocol (`core/mod.rs:L2499-L2513`) and registered with the failure detector on release (`core/delivery.rs:L68-L72`), so the fallback covers it. Only a message still unreleased when the period *expires* is dropped (`core/mod.rs:L1212-L1221, L1662-L1668`), which with a one-round generated-message release delay and a 30-round period requires a stalled node. Consistent with the spec.
- **Proposals that can never be encapsulated.** `Error::PayloadTooLarge` discards the head without a fallback (`edge/mod.rs:L453`; `crypto/leader/send.rs:L137-L139`). `MAX_PAYLOAD_BODY_SIZE = 18_192` (`blend/message/src/message/payload.rs:L18`) is exactly the maximum proposal encoding that `bedrock-v1.1-block-construction.md` §Canonical Encoding derives at `MAX_BLOCK_TXS` references and `MAX_UNCLES` uncles (L209), so a spec-conformant proposal always fits. Not a finding.
- **Secret PoL info arriving late.** A proposal queued before the epoch's secret info arrives is kept and blended once the handler exists (`current_epoch.rs:L145-L169`), covered by the existing test `a_proposal_arriving_before_the_pol_info_is_still_blended`. Correct.
- **Second copy after rotation.** If the first copy of a proposal was sent before the rotation and the second was not, the payload is already registered with the detector, so the fallback still applies to it (`failure_detection.rs:L59-L71`). Fine.
- **Transition period expiry in edge mode.** `EpochEvent::TransitionPeriodExpired` is a no-op for edge (`edge/mod.rs:L400`); edge keeps no old-epoch state, so there is nothing to drop there.
- **Transactions.** Not epoch-bound, held outside `CurrentEpoch`, and redrawn under the current epoch's PoW allowance (`pending.rs:L172-L177`); they survive rotation. Their restart durability is #172 LB-001.
- **Whether the block is lost to the network entirely.** The chain leader applies the block before publishing (`chain-leader/src/lib.rs:L716-L722`); chain-network's tip poll (`services/chain/chain-network/src/sync/tip_poll.rs`) lets a lagging peer pull it, as #172 already noted. The dropped proposal is therefore a lost *dissemination*, and usually a lost block, not corrupted state.

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

## Appendix B — Local evidence tests

Appended to `services/blend/src/edge/tests/mod.rs` in a scratch checkout of `bfcae04d25218d878eb77c3c072cb0e88524de82`; they use only the crate's existing test harness (`spawn_run_with_pol`, `GatedPolStreamProvider`, `PolGate`, `TestPayloadDispatcher`). Both tests *pass*, which documents the discard; they are evidence, not a proposed upstream change.

```rust
/// Audit #320: a proposal still queued (no leadership proofs yet) when the
/// epoch turns is dropped with the old epoch and is never handed to the direct
/// broadcast, even though the operator left the fallback on.
#[test_log::test(tokio::test(start_paused = true))]
async fn audit_320_never_released_proposal_is_dropped_at_epoch_rotation_without_fallback() {
    let local_node = NodeId(99);
    let old_core = NodeId(0);
    let new_core = NodeId(1);
    let proposal = b"proposal-from-old-epoch".to_vec();

    let pol_gate = PolGate::setup();
    let RunningEdgeService {
        handle: _join_handle,
        epochs: epoch_sender,
        messages: msg_sender,
        blended_to: mut node_id_receiver,
        mut broadcasting_channel,
    } = spawn_run_with_pol::<GatedPolStreamProvider>(
        local_node, 1, Some(membership(&[old_core], local_node)),
        false, // abstain_on_failure = false: the direct broadcast is enabled.
    ).await;

    // Queue the proposal while no handler exists (secret PoL info withheld).
    msg_sender.send(ServiceMessage::Blend(DataPayload::BlockProposal(proposal.clone()))).await.unwrap();
    let (reply, answered) = oneshot::channel();
    msg_sender.send(ServiceMessage::GetPendingTransactions { reply }).await.unwrap();
    answered.await.unwrap();

    // The epoch turns before any proof was available.
    epoch_sender.send(membership(&[new_core], local_node)).await.unwrap();
    let (reply, answered) = oneshot::channel();
    msg_sender.send(ServiceMessage::GetPendingTransactions { reply }).await.unwrap();
    answered.await.unwrap();

    // Proofs become available only now, for the new epoch.
    pol_gate.release();

    // Wait out more than a full delivery deadline.
    let waited = timeout(
        TEST_ROUND * u32::try_from(TEST_DELIVERY_DEADLINE.get() + 4).unwrap(),
        node_id_receiver.recv(),
    ).await;
    assert!(waited.is_err(), "the old epoch's proposal must not be blended under the new epoch: {waited:?}");
    assert!(broadcasting_channel.dispatched.try_recv().is_err(), "and it is never broadcast directly either: it is simply lost");
}

/// Audit #320: when the membership falls below the minimum the edge service
/// shuts down (the orchestrator switches to broadcast mode), but the proposals
/// it still holds are dropped instead of being broadcast directly, although
/// direct broadcast is exactly what the node switches to.
#[test_log::test(tokio::test(start_paused = true))]
async fn audit_320_queued_proposal_is_dropped_when_membership_falls_below_minimum() {
    let local_node = NodeId(99);
    let core_node = NodeId(0);
    let proposal = b"proposal-before-fallback".to_vec();

    let pol_gate = PolGate::setup();
    let RunningEdgeService {
        handle: join_handle,
        epochs: epoch_sender,
        messages: msg_sender,
        blended_to: mut node_id_receiver,
        mut broadcasting_channel,
    } = spawn_run_with_pol::<GatedPolStreamProvider>(
        local_node, 1, Some(membership(&[core_node], local_node)), false,
    ).await;

    msg_sender.send(ServiceMessage::Blend(DataPayload::BlockProposal(proposal.clone()))).await.unwrap();
    let (reply, answered) = oneshot::channel();
    msg_sender.send(ServiceMessage::GetPendingTransactions { reply }).await.unwrap();
    answered.await.unwrap();

    // Membership below the minimum: the service returns Ok(()) and stops.
    epoch_sender.send(membership(&[], local_node)).await.unwrap();
    assert!(matches!(join_handle.await.unwrap(), Ok(())));
    pol_gate.release();

    assert!(node_id_receiver.try_recv().is_err(), "nothing was blended");
    assert!(broadcasting_channel.dispatched.try_recv().is_err(), "and the queued proposal was not broadcast directly on the way out");
}
```

Test run output (`cargo test -p logos-blockchain-blend-service --lib edge::tests`, Rust 1.98.1, aarch64, debug profile, 4 min 51 s build):

```
running 20 tests
test edge::tests::audit_320_never_released_proposal_is_dropped_at_epoch_rotation_without_fallback ... ok
test edge::tests::audit_320_queued_proposal_is_dropped_when_membership_falls_below_minimum ... ok
test edge::tests::a_proposal_arriving_before_the_pol_info_is_still_blended ... ok
test edge::tests::a_proposal_the_network_never_delivers_is_broadcast_in_the_clear ... ok
test edge::tests::run_shuts_down_if_new_membership_is_small ... ok
test edge::tests::run_with_epoch_transition ... ignored, We need a different test setup since we are not blocking the edge tokio task until the secret PoL info is fetched, which makes this test flaky.
... (14 more pre-existing tests, all ok)

test result: ok. 19 passed; 0 failed; 1 ignored; 0 measured; 81 filtered out; finished in 0.05s
```
