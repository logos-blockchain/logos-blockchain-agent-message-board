# Audit Report — `BlendBehaviour` composition order: the block list is consulted after the blend behaviour has registered the connection

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/141`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `3d5d419ec85c7b23e4d7e1455bd1cde845264a9e` — component(s): `services/blend` (libp2p core backend), `blend/network` (core-to-core and core-to-edge behaviours), `libp2p` (node behaviour composition)
Specs: `https://github.com/logos-co/logos-lips` @ `75d3d0382604d4a0d8e246c268935dd386ffc8ea` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `proof-of-quota.md`, `message-formatting.md`, `payload-formatting.md` (in full); `blend-protocol.md` §Network (Bootstrapping, Maintenance), §Network Maintenance (Connection Details, Neighbor Distinction Process, Connectivity Maintenance, Transition Period) (by section)
Date: `2026-09-15` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: the ordering defect #101 S-001 described is real and reproduced with a swarm test: an inbound connection from a blocked core peer leaves one entry in the blend behaviour's `connections_waiting_upgrade` map, and if the peer is unblocked in the same epoch its next inbound connection is not upgraded until the listener starts a new epoch. Both candidate fixes close it, and the one that does not depend on field order (handling `ListenFailure` and `DialFailure`) is the one to keep. Today the impact is bounded and mostly latent: the node never unblocks a peer (`allow_block_list::unblock_peer` has no caller), so the stale entry costs one map slot and one extra `O(pending)` scan per inbound connection until the next epoch, and only a future design that lifts blocks mid-epoch would hit the connectivity effect. The edge behaviour and the node's main libp2p behaviour do not have the same shape: neither records state before a sibling can deny, and no sibling of the main behaviour can deny at all.
- Findings: `0` critical · `0` high · `0` medium · `1` low · `0` informational
- Key themes: "side effect before a sibling's cheap check", "a denied connection is not a closed connection"
- Must-fix before launch: none; land the `ListenFailure`/`DialFailure` handling before any block-lifting logic (#115, #100) ships.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/blend/src/core/backends/libp2p/behaviour.rs` | the composed `BlendBehaviour` (`blend`, then `blocked_peers`) |
| `blend/network/src/core/with_core/behaviour/mod.rs` | `connections_waiting_upgrade`, `handle_established_{inbound,outbound}_connection`, `on_swarm_event`, `start_new_epoch`, `has_*_connection_with_peer`, `available_connection_slots` |
| `blend/network/src/core/with_edge/behaviour/mod.rs` | the same hooks on the edge side |
| `blend/network/src/core/mod.rs` | the inner composition (`with_core`, then `with_edge`) |
| `services/blend/src/core/backends/libp2p/swarm.rs` | where peers are blocked, whether they are unblocked, what the block list is used for |
| `libp2p/src/behaviour/mod.rs`, `src/behaviour/nat/mod.rs` | the node's main composed behaviour and its custom NAT behaviour, for the same shape |
| `consensus/cryptarchia-sync/src/libp2p/behaviour.rs` | chain-sync behaviour hooks |
| `libp2p-swarm-derive` 0.35.1 `src/lib.rs`, `libp2p-swarm` 0.47.1 `src/lib.rs`, `libp2p-allow-block-list` 0.6.0 `src/lib.rs` | the generated composition, the swarm's handling of a denied established connection, the block list's enforcement points (all from the local cargo registry at the versions pinned in `Cargo.lock`) |

**Out of scope**

The message cache, PoQ verification cost, the old-epoch path and the spam verdicts (#13's other sub-issues: #59, #100, #205, #116); the Blend membership and connectivity-maintenance policy beyond the ordering question; the libp2p swarm's own connection accounting. Third-party crates assumed correct: `libp2p-swarm`, `libp2p-swarm-derive`, `libp2p-allow-block-list`, `tokio`.

**Assumptions**

- The specification at the commit above is correct unless stated otherwise. `blend-protocol.md` §Connectivity Maintenance 3.3 ("The neighbor is added to a black list, and its selection must be avoided") is the rule the block list implements; the spec sets no lifetime for the black list (#115 S-001).
- Release build facts of issue #19 re-verified at this commit; none of them bears on this issue.

## 3. Method

- Manual review of the in-scope paths, working through issue `#141` under parent `#13`, all four checklist items, against the versions of the three libp2p crates pinned in `Cargo.lock` (`libp2p-swarm` 0.47.1, `libp2p-swarm-derive` 0.35.1, `libp2p-allow-block-list` 0.6.0). The in-scope node files are unchanged between `a805329f8` (where #101 read them) and the target commit.
- Spec conformance against `blend-protocol.md` §Connectivity Maintenance 3 and §Transition Period, and §Neighbor Distinction Process for which peers the core behaviour registers.
- Automated tooling: one throw-away swarm test in `services/blend/src/core/backends/libp2p/tests/` (`cargo test --release -p logos-blockchain-blend-service blocked_peer`, `rustc 1.98.1`), with a test-only accessor `connections_waiting_upgrade()` added to the core behaviour under `#[cfg(any(test, feature = "unsafe-test-functions"))]`, the gate the crate already uses for its other test accessors. Two memory-transport swarms, both core members of a two-node membership. The listener blocks the dialer's peer ID, the dialer dials; the listener unblocks, the dialer dials again on a fresh connection; the listener starts epoch 2, the dialer dials a third time. After each step the test prints the number of pending-upgrade entries the listener holds for the dialer and whether the dialer is in the listener's `negotiated_peers`. Run three times: unmodified code, with fix A (Appendix B), and with fix B (field reorder) on the unmodified core behaviour.
- The blend service's and blend network's full test suites (`cargo test --release -p logos-blockchain-blend-service -p logos-blockchain-blend-network`) run against fix A: `logos-blockchain-blend-network` 62 passed, 0 failed, 3 ignored; `logos-blockchain-blend-service` 94 passed, 3 ignored, and the single failure is the audit test's own last assertion, which encodes the baseline behaviour and is expected to flip once the fix is in (Appendix B).
- Dynamic testing: none against a running node.

**Checked and ruled out**

- The generated composition is exactly what #101 said. `libp2p-swarm-derive` 0.35.1 builds `handle_established_inbound_connection` as `ConnectionHandler::select(self.blend.handle_established_inbound_connection(..)?, self.blocked_peers.handle_established_inbound_connection(..)?)` in field order (`src/lib.rs:289-310`); Rust evaluates the arguments left to right, so the blend behaviour's side effects happen before the block list's `enforce` returns `ConnectionDenied` (`libp2p-allow-block-list` 0.6.0 `src/lib.rs:233-243`, `:217-223`). `on_swarm_event` is fanned out to every field (`src/lib.rs:212-225`), so the blend behaviour does receive the `ListenFailure`; it ignores everything but `ConnectionClosed` (`with_core/behaviour/mod.rs:1171-1178`).
- What the swarm does with a denied established connection: `FromSwarm::ListenFailure { connection_id, peer_id: Some(..), error: Denied }` for inbound (`libp2p-swarm` 0.47.1 `src/lib.rs:736-766`) and `FromSwarm::DialFailure` for outbound (`:704-726`); no `ConnectionEstablished` and no `ConnectionClosed` follow. Both events carry the `connection_id` and the `peer_id` the blend behaviour keyed its entry on (`src/behaviour.rs:534-540`), which is what makes fix A possible.
- Outbound is not affected: the block list enforces in `handle_pending_outbound_connection` when the peer is known (`allow_block_list` `:245-257`), before any `handle_established_*` runs, and the swarm never dials a blocked peer anyway because the dial candidates exclude `blocked_peers()` (`swarm.rs:247-253`).
- Growth is bounded to one entry per blocked peer per epoch: the second inbound attempt from the same peer meets `has_incoming_connection_with_peer` (`:812-815`, `:844-850`) and gets a `DummyConnectionHandler` (`:1095-1098`) before the block list denies it again, so nothing new is inserted. `start_new_epoch` takes the whole map (`:297-300`).
- The stale entry does not affect the node's dialing budget: `available_connection_slots` counts `negotiated_peers` only (`:353-357`), and the peering-degree check in the swarm uses that (`swarm.rs:416-424`). Its only readers are `has_pending_{incoming,outgoing}_connection_with_peer`, so the effect is confined to the blocked peer and to the `O(pending)` cost of those scans.
- Unblocking does not exist today. `allow_block_list::Behaviour::unblock_peer` has no caller in the node (`grep -rn unblock services/`); `block_peer` is called once, from `handle_disconnected_peer` when a peer leaves in the `Spammy` state (`swarm.rs:427-435`). The scenario the issue describes therefore needs the block-lifting logic #115 and #100 discuss before it can happen on a real node. `StartNewEpoch` does not clear the block list either (`swarm.rs:620-637`), so a blocked peer stays blocked for the process lifetime.
- The `ConnectionHandler` the blend behaviour builds for the denied connection (`:1110-1114`, with a fresh `ConnectionMonitor` interval stream) is dropped by the swarm with the `Err`; no task is spawned for it.
- Checklist item 3, the edge behaviour: `with_edge::handle_established_inbound_connection` (`with_edge/behaviour/mod.rs:264-293`) records nothing; it returns a handler or a dummy. State (`upgraded_edge_peers`) is written only in `handle_negotiated_connection` on `SubstreamOpened` (`:154-179`), which cannot happen for a denied connection. The inner composition runs `with_core` before `with_edge` (`blend/network/src/core/mod.rs:24-27`), and `with_core` registers only peers that are in the membership (`:1103-1109`), so an edge peer is never registered by either.
- Checklist item 4, other compositions: the node's main `Behaviour` (`libp2p/src/behaviour/mod.rs:44-54`: gossipsub, kademlia, identify, chain_sync, autonat server, NAT) contains no behaviour that can deny a connection; `ConnectionDenied::new` is not called anywhere in the workspace, the NAT behaviour delegates its hooks to the autonat client (`nat/mod.rs:66-110`), chain-sync delegates to `libp2p-stream` (`cryptarchia-sync/src/libp2p/behaviour.rs:373-385`), and neither `libp2p::connection_limits` nor `allow_block_list` is composed there. The only denying behaviour in the node is the one in `BlendBehaviour`, and it is the last field.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | An inbound connection from a blocked core peer leaves a stale pending-upgrade entry in the blend behaviour for the rest of the epoch, because the block list denies after the blend behaviour has registered the connection and a denied connection is never reported as closed | Denial of Service | Low | High | Open |

### LB-001 · An inbound connection from a blocked core peer leaves a stale pending-upgrade entry in the blend behaviour for the rest of the epoch, because the block list denies after the blend behaviour has registered the connection and a denied connection is never reported as closed

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Denial of Service |
| Target | `services/blend/src/core/backends/libp2p/behaviour.rs:10-14`; `blend/network/src/core/with_core/behaviour/mod.rs:1076-1119` (`handle_established_inbound_connection`, insert at `:1108-1109`), `:1171-1204` (`on_swarm_event`), `:812-815`, `:844-850` (`has_incoming_connection_with_peer`), `:297-300` (`start_new_epoch`); `libp2p-swarm-derive` 0.35.1 `src/lib.rs:289-310`; `libp2p-swarm` 0.47.1 `src/lib.rs:736-766`; `libp2p-allow-block-list` 0.6.0 `src/lib.rs:233-243` |
| Status | Open |

**Description**

`BlendBehaviour` declares `blend` before `blocked_peers`. For an inbound connection from a peer `P` that is in the block list and in the current membership, the generated `handle_established_inbound_connection` first runs the core blend behaviour, which inserts `(P, connection_id) → Endpoint::Dialer` into `connections_waiting_upgrade` and builds a `ConnectionHandler`, and then runs the block list, which returns `ConnectionDenied`. The swarm turns the `Err` into `FromSwarm::ListenFailure` and drops the connection; it never emits `ConnectionClosed` for it. The blend behaviour handles only `ConnectionClosed`, so the entry stays until `start_new_epoch` takes the map.

Measured with the swarm test (unmodified code):

| Step | listener's pending entries for the dialer | dialer in listener's `negotiated_peers` |
|---|---|---|
| dialer blocked, dialer dials | 1 | no |
| dialer unblocked, dialer dials again (new connection) | 1 | **no** |
| listener starts epoch 2, dialer dials a third time | 0 before the dial | yes |

The second row is the consequence: `has_incoming_connection_with_peer(P)` is true because of the stale entry, so the fresh inbound connection gets a `DummyConnectionHandler` (`:1095-1098`) and is never upgraded, and this holds for every inbound attempt by `P` until the listener's next epoch. The map is bounded by that same check to one stale entry per blocked peer, and the entry is not counted by `available_connection_slots`, so on the node as shipped, where no code path ever unblocks a peer and blocks last for the process lifetime, the observable cost is one map entry per blocked peer per epoch and one more element in the `O(pending)` scans of `has_pending_*_connection_with_peer`. The defect becomes a connectivity bug the moment a block is lifted mid-epoch, which is what #115 (a lifetime for the black list) and #100 (re-enabling the spam verdict with a corrected window) are heading towards: a peer wrongly marked spammy and then pardoned could not reconnect inbound to any node that had seen its blocked attempt.

`blend-protocol.md` §Connectivity Maintenance 3.3 asks only that a black-listed neighbour's "selection must be avoided"; it says nothing about inbound handling or about the black list's lifetime, so the code is not in conflict with the spec. The defect is in the composition, not the protocol.

Both fixes the issue proposed close the gap. With fix A (Appendix B: on `ListenFailure` or `DialFailure` carrying a `peer_id`, remove the matching entry from `connections_waiting_upgrade`) the first row reads 0 entries and the second row reads "yes". With fix B (declare `blocked_peers` before `blend`, no change in `blend/network`) the result is the same. A is the one to keep: it makes the core behaviour correct for any composition and for any future sibling that denies (a connection-limits behaviour, an allow list), whereas B relies on a field order that nothing enforces and that the next contributor can silently change. B on top of A costs nothing and also puts the `HashSet::contains` before the `O(pending)` scan.

**Exploit scenario**

None on the node as shipped. A remote peer cannot cause more than one stale entry per blocked identity per epoch, cannot be blocked without first being marked spammy by the observation window (currently disabled, #100), and cannot use the entry against anyone but itself. Once blocks can be lifted mid-epoch, a node that pardons a peer keeps refusing its inbound connections until the next epoch, an unexplained loss of a healthy core connection on both sides; an adversary who can get honest peers briefly marked spammy on many nodes (the attack #100 is measuring) would then have those peers stay disconnected inbound for up to an epoch after the pardon.

**Recommendation**

- *Short term*: apply fix A in `with_core::Behaviour::on_swarm_event`, and add the swarm test of §3 (with the `connections_waiting_upgrade` accessor under the existing test feature) so the composition order is covered. Optionally reorder the fields as well.
- *Long term*: a rule for composed behaviours in this workspace: a field that only denies goes first, and any behaviour that records state in `handle_established_*` must also handle `ListenFailure` and `DialFailure` for that `connection_id`.

**References**: `blend-protocol.md` §Connectivity Maintenance 3, §Transition Period; #101 S-001 (source), #115, #100, #116 (epoch-bound connections).

## 5. Suggestions (non-security)

### S-001 · The `has_pending_*_connection_with_peer` scans are linear in the number of pending connections

| | |
|---|---|
| Target | `blend/network/src/core/with_core/behaviour/mod.rs:840-862` (the two `TODO: Find a different data structure` comments) |

Every inbound and outbound `handle_established_*` call walks the whole pending map. The map is small in normal operation (at most the peering degree, plus one stale entry per blocked peer under LB-001), so this is a code-quality note: a `HashMap<PeerId, SmallVec<(ConnectionId, Endpoint)>>` or two `HashSet<PeerId>` side indexes make the checks `O(1)` and remove the second reason the issue gave for reordering the fields.

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

## Appendix B — Fix A and the swarm test

Against `blend/network/src/core/with_core/behaviour/mod.rs` at `3d5d419ec85c7b23e4d7e1455bd1cde845264a9e`. The accessor is test-only; the `on_swarm_event` hunk is the fix.

```diff
diff --git a/blend/network/src/core/with_core/behaviour/mod.rs b/blend/network/src/core/with_core/behaviour/mod.rs
index 0039b658e..baf85f62c 100644
--- a/blend/network/src/core/with_core/behaviour/mod.rs
+++ b/blend/network/src/core/with_core/behaviour/mod.rs
@@ -23,7 +23,8 @@ use libp2p::{
     Multiaddr, PeerId, StreamProtocol,
     core::{Endpoint, transport::PortUse},
     swarm::{
-        ConnectionClosed, ConnectionDenied, ConnectionId, FromSwarm, NetworkBehaviour,
+        ConnectionClosed, ConnectionDenied, ConnectionId, DialFailure, FromSwarm, ListenFailure,
+        NetworkBehaviour,
         NotifyHandler, THandler, THandlerInEvent, THandlerOutEvent, ToSwarm,
         dummy::ConnectionHandler as DummyConnectionHandler,
     },
@@ -434,6 +435,15 @@ impl<ObservationWindowClockProvider, ProofsVerifier>
         Ok(())
     }
 
+    /// AUDIT (#141): test-only view of the connections that were registered
+    /// by `handle_established_*_connection` but not yet upgraded.
+    #[cfg(any(test, feature = "unsafe-test-functions"))]
+    pub const fn connections_waiting_upgrade(
+        &self,
+    ) -> &HashMap<(PeerId, ConnectionId), Endpoint> {
+        &self.connections_waiting_upgrade
+    }
+
     pub const fn negotiated_peers(&self) -> &HashMap<PeerId, RemotePeerConnectionDetails> {
         &self.negotiated_peers
     }
@@ -1169,6 +1179,29 @@ where
 
     /// Informs the behaviour about an event from the [`Swarm`].
     fn on_swarm_event(&mut self, event: FromSwarm) {
+        // AUDIT (#141) prototype A: a sibling behaviour denied a connection that
+        // this behaviour had already registered as waiting for upgrade. The
+        // swarm never opens it, so no `ConnectionClosed` will follow: forget it.
+        if let FromSwarm::ListenFailure(ListenFailure {
+            connection_id,
+            peer_id: Some(peer_id),
+            ..
+        })
+        | FromSwarm::DialFailure(DialFailure {
+            connection_id,
+            peer_id: Some(peer_id),
+            ..
+        }) = event
+        {
+            if self
+                .connections_waiting_upgrade
+                .remove(&(peer_id, connection_id))
+                .is_some()
+            {
+                tracing::debug!(target: LOG_TARGET, "Connection {connection_id:?} with peer {peer_id:?} was denied after being registered for upgrade: forgetting it.");
+            }
+            return;
+        }
         if let FromSwarm::ConnectionClosed(ConnectionClosed {
             peer_id,
             connection_id,
```

The swarm test, `services/blend/src/core/backends/libp2p/tests/blocked_peer.rs` (registered in `tests/mod.rs`); the final assertion holds on the unmodified code and is the step that becomes redundant once the fix is in, because the dialer is already negotiated before the epoch change:

```rust
//! AUDIT (#141): what an inbound connection from a blocked peer leaves behind
//! in the core blend behaviour.
use core::time::Duration;

use lb_blend::scheduling::membership::Node;
use libp2p::PeerId;
use test_log::test;
use tokio::{
    select,
    time::{self, sleep_until},
};

use crate::core::backends::{
    BackendEpochInfo,
    libp2p::{
        core_swarm_test_utils::{SwarmExt as _, new_nodes_with_empty_address, update_nodes},
        swarm::BlendSwarmMessage,
        tests::utils::{
            BlendBehaviourBuilder, InnerSwarm, SwarmBuilder, TestProofsVerifier, TestSwarm,
            build_membership,
        },
    },
};

async fn drive(a: &mut InnerSwarm, b: &mut InnerSwarm, for_: Duration) {
    let deadline = time::Instant::now() + for_;
    loop {
        select! {
            () = sleep_until(deadline) => break,
            () = a.poll_next() => {}
            () = b.poll_next() => {}
        }
    }
}

fn pending_entries_for(swarm: &InnerSwarm, peer: PeerId) -> usize {
    swarm
        .behaviour()
        .blend
        .with_core()
        .connections_waiting_upgrade()
        .keys()
        .filter(|(p, _)| *p == peer)
        .count()
}

fn is_negotiated(swarm: &InnerSwarm, peer: PeerId) -> bool {
    swarm
        .behaviour()
        .blend
        .with_core()
        .negotiated_peers()
        .contains_key(&peer)
}

/// Baseline at the audited commit: the entry stays, the unblocked peer is not
/// upgraded until the listener's next epoch.
#[test(tokio::test)]
async fn inbound_from_blocked_peer_leaves_pending_entry_until_next_epoch() {
    let (mut identities, mut nodes) = new_nodes_with_empty_address(2);

    let TestSwarm {
        swarm: mut listening_swarm,
        swarm_message_sender: listening_sender,
        ..
    } = SwarmBuilder::new(identities.next().unwrap(), &nodes)
        .build(|id, membership| BlendBehaviourBuilder::new(id, membership).build());
    let (
        Node {
            address: listening_address,
            id: listening_peer_id,
            ..
        },
        _,
    ) = listening_swarm
        .listen_and_return_membership_entry(None)
        .await;
    update_nodes(&mut nodes, &listening_peer_id, listening_address.clone());

    let TestSwarm {
        swarm: mut dialing_swarm,
        ..
    } = SwarmBuilder::new(identities.next().unwrap(), &nodes)
        .build(|id, membership| BlendBehaviourBuilder::new(id, membership).build());
    let dialing_peer_id = *dialing_swarm.local_peer_id();

    // The listener blocks the dialer before any connection exists.
    listening_swarm
        .behaviour_mut()
        .blocked_peers
        .block_peer(dialing_peer_id);

    // 1. Inbound connection from the blocked peer.
    dialing_swarm.dial_peer_at_addr(listening_peer_id, listening_address.clone());
    drive(&mut listening_swarm, &mut dialing_swarm, Duration::from_secs(2)).await;

    let stale = pending_entries_for(&listening_swarm, dialing_peer_id);
    println!("AUDIT after blocked dial: pending_entries={stale} negotiated={}", is_negotiated(&listening_swarm, dialing_peer_id));
    assert!(!is_negotiated(&listening_swarm, dialing_peer_id));

    // 2. Unblock and dial again on a fresh connection.
    listening_swarm
        .behaviour_mut()
        .blocked_peers
        .unblock_peer(dialing_peer_id);
    dialing_swarm.dial_peer_at_addr(listening_peer_id, listening_address.clone());
    drive(&mut listening_swarm, &mut dialing_swarm, Duration::from_secs(3)).await;
    let negotiated_after_unblock = is_negotiated(&listening_swarm, dialing_peer_id);
    println!(
        "AUDIT after unblock+dial: pending_entries={} negotiated={negotiated_after_unblock}",
        pending_entries_for(&listening_swarm, dialing_peer_id)
    );

    // 3. New epoch on the listener clears the pending map; dial once more.
    listening_sender
        .send(BlendSwarmMessage::StartNewEpoch(BackendEpochInfo {
            membership: build_membership(&nodes, Some(listening_peer_id)),
            epoch: 2.into(),
            proofs_verifier: TestProofsVerifier,
        }))
        .await
        .unwrap();
    drive(&mut listening_swarm, &mut dialing_swarm, Duration::from_secs(1)).await;
    let pending_after_epoch = pending_entries_for(&listening_swarm, dialing_peer_id);
    dialing_swarm.dial_peer_at_addr(listening_peer_id, listening_address);
    drive(&mut listening_swarm, &mut dialing_swarm, Duration::from_secs(3)).await;
    let negotiated_after_epoch = is_negotiated(&listening_swarm, dialing_peer_id);
    println!(
        "AUDIT after new epoch: pending_before_dial={pending_after_epoch} negotiated={negotiated_after_epoch}"
    );
    println!(
        "AUDIT SUMMARY stale_entry_after_block={stale} negotiated_after_unblock={negotiated_after_unblock} negotiated_after_epoch={negotiated_after_epoch}"
    );
    assert!(negotiated_after_epoch);
}
```
