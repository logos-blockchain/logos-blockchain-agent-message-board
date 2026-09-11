# Audit Report — Blend core blocklist: epoch-bounded blocks, enforcement inside the blend behaviour, dial-retry and observability follow-ups

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/117`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `blend/network` (core-to-core behaviour), `services/blend` (libp2p core backend, metrics), `libp2p` (`DialErrorExt`)
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md` (in full); `blend-protocol.md` by section (see Method)
Date: `2026-09-11` — author: `Claude Fable 5.1 (agent)` — status: `fix-review`

---

## 1. Summary

- Overall assessment: the five items of #117 are done against today's upstream head, as one verified patch (Appendix B) that supersedes the #101 patch: the block lifetime is bound to epochs, the blocklist is owned by the blend behaviour so a verdict closes only the offending connection, a blocked peer is refused before the transport handshake, `DialError::Denied` is no longer retried, and the blocklist is observable. The patch is not yet submitted upstream and the spec gap is not yet filed in logos-lips; both texts are ready (Appendix B, Appendix C) and need a human to open them.
- Findings: 0 critical · 0 high · 1 medium (re-verified, fixed in the patch) · 1 low (new, fixed in the patch) · 0 informational
- Key themes: "a block outlives its evidence", "a block is issued by the component that did not issue the verdict", "a verdict racing the epoch transition is lost"
- Must-fix before launch: LB-001 (from #101, still open upstream at `a805329f8`; the patch here closes it together with LB-002)

This report is the follow-up of `inbox/101-blend-blocklist-lifetime.md` (PR #115). It does not re-derive that report's analysis of which verdicts are live and how honest peers trigger them; it re-verifies its patch on the current head, then does the structural work the issue asks for. The epoch-binding of connections (#101 LB-002) is out of scope and remains open (#116).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `blend/network/src/core/with_core/behaviour/mod.rs` | new owner of the blocklist: `block_peer`, `lift_expired_blocks`, denial in the three connection hooks, `PeerBlocked` / `PeerUnblocked` events; `start_new_epoch`, `close_spammy_connection`, `on_swarm_event` (verdict and close ordering) |
| `blend/network/src/core/with_core/behaviour/old_epoch.rs` | what happens to a connection, and to a verdict, that belong to the outgoing epoch |
| `services/blend/src/core/backends/libp2p/{behaviour.rs,swarm.rs}` | removal of `allow_block_list`; dial exclusion; `StartNewEpoch`; retry path (`dial`, `execute_retry`, `schedule_retry`, `abandon_peer`); the new warning |
| `services/blend/src/metrics.rs` | `blend_core_peers_blocked_total`, new `blend_core_peers_unblocked_total` and `blend_core_peers_blocked` gauge |
| `libp2p/src/dial_error_ext.rs` | classification of `DialError::Denied` |
| `libp2p-swarm` 0.47.1, `libp2p-allow-block-list` 0.6.0 (`~/.cargo/registry`) | hook signatures (`handle_pending_outbound_connection`, `ConnectionDenied`), what `block_peer` closes |
| `blend-protocol.md` §Core Network, §Connectivity Maintenance, §Transition Period | the reference for items 2 and 5 |

**Out of scope**
The edge behaviour and edge swarm (edge peers are never blocked), the connection handler's substream state machine (#60), the observation-window verdicts that are disabled today (#73, #100), PoQ verification itself and its cost (#13, #74), the epoch assignment of connections (#116), the chain service's epoch-state query, and the Groth16 verifier. `libp2p` (swarm, QUIC, memory transport), `tokio`, `futures` and `lb_tracing` are assumed correct. No multi-node devnet was run.

**Assumptions**
Release-profile facts from #19 hold. Core peers are authenticated by the QUIC/TLS handshake, so a verdict is attributable to the peer it is issued against, and a Sybil core identity costs the SDP minimum stake, so escalating the block per identity is the right unit. The traffic model and the estimates of #101 LB-002 are taken as given.

## 3. Method

- Manual review of the in-scope paths, working through the five items of #117, with parent #13 and the #101 report as context.
- Spec conformance against `blend-protocol.md` @ `7244d3b0`: §Protocol / Network Maintenance / Core Network (L303-L345), §Details / Connectivity Maintenance (L548-L576, the blacklist rule at L559), §Transition Period (L578-L604), and §Failure Detection and Reaction (L442-L456, new in this spec revision, checked for any interaction with the blacklist: none). The two core overview documents were read in full. The spec changed between the #101 report (`4b9d1ca1`) and now (`7244d3b0`): the blacklist sentence is unchanged, only renumbered from L527 to L559.
- Item 1 was re-verified first, on the unmodified head: `git apply` of the #101 Appendix B patch on `a805329f8`, then `cargo test -p logos-blockchain-blend-service --lib backends::libp2p::tests`.
- Automated tooling, with versions: rustc 1.98.1 (aarch64, Raspberry Pi 5); `cargo clippy --tests` on `logos-blockchain-blend-network`, `logos-blockchain-blend-service`, `logos-blockchain-libp2p`; `cargo +nightly-2026-07-05 fmt --check` on the same three crates (nightly-2026-07-05 is the toolchain the upstream `code-check.yml` fmt job uses); `cargo test` on the same three crates (see the results table under Findings, "Verification").
- Dynamic testing: three new behaviour-level tests in `blend/network/src/core/with_core/behaviour/tests/blocklist.rs` and the two swarm-level tests of #101, adapted. Each test was run at least twice. No devnet.
- Not done: the upstream PR and the logos-lips issue were drafted but not opened (outward-facing actions, left to the human operator); the patch was not run against the disabled `TooManyMessages` verdict, which stays disabled.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | A spammy verdict blocks a core peer for the life of the process, and closes every connection to it (#101 LB-001, S-002) | Denial of Service | Medium | Low | Fixed in the patch of Appendix B (not yet upstream) |
| LB-002 | A spammy verdict issued just before an epoch transition never becomes a block | Data Validation | Low | Medium | Fixed in the patch of Appendix B (not yet upstream) |

### LB-001 · A spammy verdict blocks a core peer for the life of the process, and closes every connection to it

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `services/blend/src/core/backends/libp2p/swarm.rs:L427-L433` (`handle_disconnected_peer`), `L248` (dial filter), `L625-L640` (`StartNewEpoch`); `services/blend/src/core/backends/libp2p/behaviour.rs:L4`, `L13`, `L60` (`allow_block_list`); at `a805329f8` |
| Status | Fixed in Appendix B; open upstream |

**Description**

Unchanged from #101 LB-001 and S-002, re-verified at `a805329f8`: the block is issued in `handle_disconnected_peer` (L431) through `libp2p::allow_block_list::Behaviour::block_peer`, nothing calls `unblock_peer` (`git grep unblock_peer a805329f8` finds no caller in the workspace), `StartNewEpoch` (L625-L640) clears every other per-epoch set but not the blocklist, and `block_peer` queues `CloseConnection::All`, so the draining old-epoch connection to the same peer is closed too. The blend service was restructured between `7b5e48b0` and `a805329f8` (commits `ce3e2504d`, `d96e7ef99`), but none of the touched lines moved, and the #101 patch applies without offsets.

**Item 1 — re-verification of the #101 patch on the current head**

| Step | Result |
|---|---|
| `git apply --3way inbox/101 Appendix B` on `a805329f8` | all four files applied cleanly, no conflicts, no offsets |
| `cargo test -p logos-blockchain-blend-service --lib backends::libp2p::tests` | 20 passed, 0 failed, 2 ignored (`peer_with_invalid_poq_is_blocked`, `blocked_peer_is_dialable_again_in_next_epoch` both pass) |

That patch is now superseded by Appendix B of this report, which contains it (the strike/expiry logic, the two swarm tests and the `TestProofsVerifier` change) moved to the place items 2 to 4 need it.

**Item 2 — block enforcement moved into the blend behaviour**

The blocklist is now a field of the core behaviour (`blend/network/src/core/with_core/behaviour/mod.rs`, patched tree): `blocked_peers: HashMap<PeerId, Epoch>` (expiry epoch, L176) and `spam_strikes: HashMap<PeerId, u32>` (L180), with `MAX_BLOCK_DURATION_IN_EPOCHS = 4` (L65). The behaviour:

- issues the block at verdict time, in `set_connection_to_spammy` (L857-L871): only when `update_state_for_negotiated_peer` actually updated the peer's current-epoch connection, and only if that connection was not already `Spammy`, so a second failed verification on a connection that is being torn down does not add a strike. `block_peer` (L411-L426) computes `until = current_epoch + min(strikes, 4)` and emits `Event::PeerBlocked { peer_id, reason, until_epoch }`;
- lifts expired blocks in `start_new_epoch` (`lift_expired_blocks`, L386-L407, called at L352 right after the new epoch info is installed), emitting `Event::PeerUnblocked`, and forgets the strikes of peers that are neither blocked nor in the new membership, so both maps stay bounded by the membership size;
- denies a blocked peer in `handle_pending_outbound_connection` (L1317-L1333, before any transport handshake for the node's own dials), `handle_established_inbound_connection` (L1203-L1212) and `handle_established_outbound_connection` (L1266-L1275), returning `ConnectionDenied` with a `BlockedPeer` cause (L69-L86).

Denial is by `Err(ConnectionDenied)` rather than by the `DummyConnectionHandler` the issue suggested (the edge-peer path at L1116/L1165 of the unpatched file). Both close only the offending connection, but a dummy handler on an *outbound* connection leaves the swarm's `ongoing_dials` entry for that peer dangling until the epoch ends, because no `OutboundConnectionUpgradeFailed` is ever produced for a connection that was never upgraded, whereas `ConnectionDenied` surfaces as `OutgoingConnectionError { error: Denied }`, which the swarm handles (item 3). For inbound connections the two are equivalent; `ConnectionDenied` was kept for symmetry and because it is what `allow_block_list` did, so the peer on the other side sees the same thing as before.

`close_spammy_connection` (L836-L850) still closes the one connection the verdict was issued on, through the handler's `CloseSubstreams`. A blocked peer's old-epoch connection is untouched and keeps being served during the transition period, which is what `blend-protocol.md` §Transition Period L603 requires ("maintain old connections and process all messages received from these connections for the duration of TP"). The service side (`swarm.rs`) no longer composes `allow_block_list` (`behaviour.rs`), reads the blocklist from the behaviour for its dial filter (L243-L250), and reacts to the two new events (L509-L518) for logging and metrics only.

**Item 3 — `DialError::Denied` unrecoverable**

`libp2p/src/dial_error_ext.rs` L20 now classes `Denied` with `WrongPeerId`, `LocalPeerId` and `NoAddresses`. With the pending-outbound hook above, a retry fired by `execute_retry` for a peer blocked in the meantime fails synchronously in `Swarm::dial` with `Denied`, and `abandon_peer` removes the dial and looks for a replacement at once; the same happens to a dial that was already in flight when the verdict landed, through `OutgoingConnectionError`. No separate blocklist check in `execute_retry` was needed. `Denied` is also returned when a behaviour other than blend refuses a dial; the edge swarm's retry path (`services/blend/src/edge/backends/libp2p/swarm.rs`) shares `DialErrorExt` and gains the same, correct, behaviour (a refused dial is not going to succeed on retry). The crate's 65 tests pass.

**Item 4 — observability**

`services/blend/src/metrics.rs`: `blend_core_peers_unblocked_total` counter (L75) and `blend_core_peers_blocked` gauge (L81), set from the swarm on every `PeerBlocked` / `PeerUnblocked` next to the existing `blend_core_peers_blocked_total{reason}`. `swarm.rs` L265-L277 logs at `warn`, under the `blend_reachability` diagnostic with event `blend_no_dialable_peers`, when no member is dialable and at least one peer is blocked, with the membership size, the blocked count and the number of connections needed. Seen firing in the test log of `blocked_peer_is_dialable_again_in_next_epoch`: `epoch=1 membership_size=2 blocked_peers=1 needed_connections=1`.

**Item 5 — spec gap**

`blend-protocol.md` @ `7244d3b0` L559 still defines the blacklist without a lifetime, without a rule for a blacklist covering the membership, and without saying whether a verdict on one connection closes the other connection to the same neighbour during the transition period. The issue text for logos-lips is in Appendix C; it was not filed.

**Verification** (patched tree, `cd8393083` on top of `a805329f8`)

| Check | Result |
|---|---|
| `cargo test -p logos-blockchain-blend-network --lib with_core::behaviour::tests` | 41 passed, 0 failed, 3 ignored (the 3 new tests included) |
| `cargo test -p logos-blockchain-blend-service --lib backends::libp2p::tests` | 20 passed, 0 failed, 2 ignored |
| `cargo test -p logos-blockchain-libp2p` | 65 passed, 0 failed, 2 ignored |
| `cargo clippy -p … --tests` (three crates) | no warnings |
| `cargo +nightly-2026-07-05 fmt --check` (three crates) | clean |

New tests (`blend/network/src/core/with_core/behaviour/tests/blocklist.rs`):

- `block_duration_escalates_per_strike_and_expires_with_epochs` (L30): first strike until `e+1`, second until `e+2`, cap at `+4`, strikes kept while the peer is blocked or a member, forgotten otherwise.
- `blocked_peer_is_denied_until_its_block_expires` (L95): garbage over the current-epoch connection produces `PeerBlocked { UndeserializableMessage, until: 1 }` and closes the connection; a re-dial by the blocked peer is denied (`IncomingConnectionError` on the listener, `ConnectionClosed` on the dialer, no upgrade); `start_new_epoch(1)` emits `PeerUnblocked` and the peer negotiates again.
- `verdict_on_new_epoch_connection_leaves_old_epoch_connection_open` (L247): two connections with the same peer during a transition; a verdict on the new-epoch one closes exactly that connection id, the peer stays in `old_epoch_peer_ids()`, and an epoch-0 message it then sends over the old connection is delivered as `Event::Message { epoch: 0 }`. Under `allow_block_list` this test cannot pass, since `block_peer` closes `CloseConnection::All` (`libp2p-allow-block-list` 0.6.0 `src/lib.rs` L148-L158, L283-L291).

The two swarm-level tests of #101 were adapted to the new API (`is_blocked`) and to the fact that the block now precedes the disconnect (the helper waits for both).

**Recommendation**
- *Short term*: land Appendix B upstream (the branch `audit/117-blocklist-behaviour` at `cd8393083` in the reviewer's checkout is the same content as the patch, with a commit message).
- *Long term*: bind connections to epochs (#116), after which a PoQ failure is again a strong spam signal and the escalation cap can be revisited; re-enable the observation-window verdicts only with the measurement of #100, since every verdict now also costs the peer one to four epochs of exclusion at every node that issued it.

**References**: #101 LB-001/S-002/S-003/S-005; `blend-protocol.md` L559, L578-L604; `libp2p-allow-block-list` 0.6.0 `src/lib.rs` L136-L172, L233-L291; `libp2p-swarm` 0.47.1 `src/behaviour.rs` L181-L189.

### LB-002 · A spammy verdict issued just before an epoch transition never becomes a block

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Data Validation |
| Target | `blend/network/src/core/with_core/behaviour/mod.rs:L285-L319` (`start_new_epoch`), `L1171-L1184` (`on_swarm_event`, old-epoch short-circuit); `services/blend/src/core/backends/libp2p/swarm.rs:L427-L433`; at `a805329f8` |
| Status | Fixed in Appendix B; open upstream |

**Description**

In the unpatched design the block is issued by the swarm on `Event::PeerDisconnected(peer, Spammy(_))`, which the behaviour emits from `on_swarm_event` when the closed connection is the peer's *current-epoch* negotiated connection (L1206-L1221). The verdict itself only sets the state to `Spammy` and asks the handler to close the substreams (`close_spammy_connection`, L729-L740); the actual `ConnectionClosed` arrives one or more polls later. If `StartNewEpoch` is processed in between, `start_new_epoch` (L307-L316) moves *every* negotiated peer into `OldEpoch`, `Spammy` state included, and when the close finally arrives `old_epoch.handle_closed_connection` swallows it (L1180-L1184), so no `PeerDisconnected` is emitted and the peer is never blocked. The swarm's `StartNewEpoch` handler then dials the same peer again immediately (`check_and_dial_new_peers`, L638), and the spammer starts the new epoch with a fresh connection and a clean record.

This was observed rather than derived: the first run of `blocked_peer_is_dialable_again_in_next_epoch` against an intermediate version of the patch (block issued at verdict time, test helper returning before the disconnect) sent `StartNewEpoch` in exactly this window, and the sequence above is what the logs showed for the unpatched ordering of events.

**Exploit scenario**

A spammer that can time its abuse to the last round of an epoch (epoch boundaries are on the slot clock and public) loses the connection but not its standing, and is re-dialed by the very node that convicted it. With the shipped parameters that is one free strike every five hours per victim. The impact is bounded by the epoch length and by the fact that the verdicts live today are the ones honest peers trigger anyway (#101), hence Low; it becomes relevant once the observation-window verdicts are re-enabled.

**Recommendation**
- *Short term*: issue the block at verdict time, inside the behaviour, as Appendix B does (`set_connection_to_spammy` → `block_peer`); the block is then independent of when, and in which epoch, the connection is actually torn down.
- *Long term*: none beyond LB-001's.

**References**: `blend-protocol.md` L559 ("The neighbor is added to a black list"); #101 LB-001 last paragraph of its Description, which described the swallow as harmless because it only concerned old-epoch traffic, and missed this ordering.

## 5. Suggestions (non-security)

### S-001 · Spec: blacklist lifetime, exhaustion, and verdict scope (carried from #101 S-001)

Still open in `blend-protocol.md` @ `7244d3b0` L559. Appendix C is the issue text for logos-lips, with three concrete proposals: expiry at the next epoch with per-strike escalation and a global cap; a log-and-cooldown rule when fewer than `Φ_CC^Min` members are selectable, without early lifting; and a statement that a verdict closes the connection it was issued on and that the blacklist only prevents new connections.

### S-002 · `Event::PeerBlocked` is emitted before the connection closes

Consumers that count blocks and disconnects (metrics, tests) must not assume the peer is already out of `negotiated_peers` when `PeerBlocked` arrives; the swarm-level test helper was changed for this. The alternative, emitting the event on disconnect, would reintroduce LB-002 for the *event* (not for the block), so the ordering is deliberate. Documented on the event variant.

### S-003 · Test toolchain drift

The #101 report ran `cargo fmt` with nightly-2025-08-19; upstream's `code-check.yml` now pins nightly-2026-07-05, whose `wrap_comments` output differs from both the older and the current nightly on untouched files. Agents verifying patches should install exactly the pinned nightly (`rustup toolchain install nightly-2026-07-05 --profile minimal -c rustfmt`), otherwise `fmt --check` reports dozens of pre-existing hunks and the real ones are lost in the noise.

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

## Appendix B — Patch (items 1 to 4, against `a805329f8`)

Applies with `git apply` on `logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b`; verified as described in Method and under LB-001 "Verification". It supersedes Appendix B of `inbox/101-blend-blocklist-lifetime.md`. Proposed commit title: `fix(blend): bound spammy-peer blocks to epochs and own the blocklist in the blend behaviour`.

```diff
diff --git a/blend/network/src/core/with_core/behaviour/mod.rs b/blend/network/src/core/with_core/behaviour/mod.rs
index 0039b658e..4a191a15b 100644
--- a/blend/network/src/core/with_core/behaviour/mod.rs
+++ b/blend/network/src/core/with_core/behaviour/mod.rs
@@ -57,6 +57,32 @@ mod tests;
 
 const LOG_TARGET: &str = blend::network::core::core::BEHAVIOUR;
 
+/// Longest a spammy peer stays blocked, in epochs. A peer's first spammy
+/// verdict blocks it until the next epoch; every further verdict adds an
+/// epoch, up to this cap. The membership, and the `PoQ` public inputs the
+/// verdict may have been based on, are rebuilt every epoch, so a block that
+/// outlives the epoch it was issued in outlives its evidence.
+const MAX_BLOCK_DURATION_IN_EPOCHS: u32 = 4;
+
+/// The reason a connection with a blocked peer is denied.
+#[derive(Debug)]
+pub struct BlockedPeer {
+    peer_id: PeerId,
+    until_epoch: Epoch,
+}
+
+impl core::fmt::Display for BlockedPeer {
+    fn fmt(&self, f: &mut core::fmt::Formatter<'_>) -> core::fmt::Result {
+        write!(
+            f,
+            "peer {:?} is blocked for spamming until epoch {:?}",
+            self.peer_id, self.until_epoch
+        )
+    }
+}
+
+impl core::error::Error for BlockedPeer {}
+
 #[derive(Debug)]
 pub struct Config {
     /// The [minimum, maximum] peering degree of this node.
@@ -143,6 +169,15 @@ pub struct Behaviour<ObservationWindowClockProvider, ProofsVerifier> {
     /// States for processing messages from the old epoch
     /// before the transition period has passed.
     old_epoch: Option<OldEpoch<ProofsVerifier>>,
+    /// Peers caught spamming on a current-epoch connection, with the epoch at
+    /// which they may connect again. No new connection is upgraded with a
+    /// blocked peer in either direction; connections it already holds (an
+    /// old-epoch one draining during the transition period) are unaffected.
+    blocked_peers: HashMap<PeerId, Epoch>,
+    /// Number of spammy verdicts issued against each peer, which sets how many
+    /// epochs its next block lasts. Entries are dropped at the epoch
+    /// transition for peers that are neither blocked nor members any more.
+    spam_strikes: HashMap<PeerId, u32>,
 }
 
 #[derive(Debug, Eq, PartialEq, Clone, Copy)]
@@ -230,6 +265,16 @@ pub enum Event {
     /// A connection with a peer has dropped. The last state that was negotiated
     /// with the peer is also returned.
     PeerDisconnected(PeerId, NegotiatedPeerState),
+    /// A peer was caught spamming on its current-epoch connection and no new
+    /// connection will be upgraded with it before `until_epoch`.
+    PeerBlocked {
+        peer_id: PeerId,
+        reason: SpamReason,
+        until_epoch: Epoch,
+    },
+    /// A peer's block has expired at an epoch transition and it may connect
+    /// again.
+    PeerUnblocked(PeerId),
     /// An outbound connection request was successfully negotiated with the
     /// remote peer.
     OutboundConnectionUpgradeSucceeded(PeerId),
@@ -279,6 +324,8 @@ impl<ObservationWindowClockProvider, ProofsVerifier>
             minimum_network_size: config.minimum_network_size,
             num_blend_layers: config.num_blend_layers,
             old_epoch: None,
+            blocked_peers: HashMap::new(),
+            spam_strikes: HashMap::new(),
         }
     }
 
@@ -302,6 +349,7 @@ impl<ObservationWindowClockProvider, ProofsVerifier>
         let current_epoch_proofs_verifier =
             mem::replace(&mut self.proofs_verifier, Arc::new(new_proofs_verifier));
 
+        self.lift_expired_blocks();
         self.stop_old_epoch();
 
         self.old_epoch = Some(OldEpoch::new(
@@ -333,6 +381,67 @@ impl<ObservationWindowClockProvider, ProofsVerifier>
         }
     }
 
+    /// Unblocks the peers whose block has run its course at the current epoch,
+    /// and forgets the strikes of peers that are neither blocked nor members.
+    fn lift_expired_blocks(&mut self) {
+        let current_epoch = self.current_epoch_info.1;
+        let expired_blocks: Vec<PeerId> = self
+            .blocked_peers
+            .iter()
+            .filter(|(_, until_epoch)| **until_epoch <= current_epoch)
+            .map(|(peer_id, _)| *peer_id)
+            .collect();
+        for peer_id in expired_blocks {
+            self.blocked_peers.remove(&peer_id);
+            tracing::debug!(target: LOG_TARGET, "Unblocking peer {peer_id:?}: its block expired at epoch {current_epoch:?}.");
+            self.events
+                .push_back(ToSwarm::GenerateEvent(Event::PeerUnblocked(peer_id)));
+        }
+        let blocked_peers = &self.blocked_peers;
+        let membership = &self.current_epoch_info.0;
+        self.spam_strikes.retain(|peer_id, _| {
+            blocked_peers.contains_key(peer_id) || membership.contains(peer_id)
+        });
+        self.try_wake();
+    }
+
+    /// Blocks a peer caught spamming on its current-epoch connection: its
+    /// first strike blocks it until the next epoch, every further strike adds
+    /// an epoch, up to [`MAX_BLOCK_DURATION_IN_EPOCHS`].
+    fn block_peer(&mut self, peer_id: PeerId, reason: SpamReason) {
+        let strikes = self.spam_strikes.entry(peer_id).or_insert(0);
+        *strikes = strikes.saturating_add(1);
+        let until_epoch: Epoch = u32::from(self.current_epoch_info.1)
+            .saturating_add((*strikes).min(MAX_BLOCK_DURATION_IN_EPOCHS))
+            .into();
+        tracing::debug!(target: LOG_TARGET, "Blocking spammy peer {peer_id:?} for reason {reason:?} until epoch {until_epoch:?} (strike {strikes}).");
+        self.blocked_peers.insert(peer_id, until_epoch);
+        self.events
+            .push_back(ToSwarm::GenerateEvent(Event::PeerBlocked {
+                peer_id,
+                reason,
+                until_epoch,
+            }));
+        self.try_wake();
+    }
+
+    /// Whether no new connection may be upgraded with the peer because it was
+    /// caught spamming.
+    #[must_use]
+    pub fn is_blocked(&self, peer_id: &PeerId) -> bool {
+        self.blocked_peers.contains_key(peer_id)
+    }
+
+    /// The peers currently blocked for spamming.
+    pub fn blocked_peers(&self) -> impl Iterator<Item = &PeerId> {
+        self.blocked_peers.keys()
+    }
+
+    #[must_use]
+    pub fn num_blocked_peers(&self) -> usize {
+        self.blocked_peers.len()
+    }
+
     #[must_use]
     pub fn num_healthy_peers(&self) -> usize {
         self.negotiated_peers
@@ -726,6 +835,12 @@ impl<ObservationWindowClockProvider, ProofsVerifier>
 
     /// Mark the connection with the sender of a malformed message as malicious
     /// and instruct its connection handler to drop the substream.
+    ///
+    /// Only the offending connection is closed. If it is the peer's
+    /// current-epoch connection, the peer is also blocked from opening or
+    /// receiving new connections until its block expires; an old-epoch
+    /// connection with the same peer keeps draining during the transition
+    /// period.
     fn close_spammy_connection(
         &mut self,
         (peer_id, connection_id): (PeerId, ConnectionId),
@@ -744,10 +859,15 @@ impl<ObservationWindowClockProvider, ProofsVerifier>
         (peer_id, connection_id): (PeerId, ConnectionId),
         reason: SpamReason,
     ) {
-        self.update_state_for_negotiated_peer(
+        // A verdict counts once per connection: a second failure on the same
+        // connection before it is torn down must not add a strike.
+        if let Some(previous_state) = self.update_state_for_negotiated_peer(
             (peer_id, connection_id),
             NegotiatedPeerState::Spammy(reason),
-        );
+        ) && !previous_state.is_spammy()
+        {
+            self.block_peer(peer_id, reason);
+        }
     }
 
     /// Update the state of an already negotiated peer if exists,
@@ -1080,6 +1200,18 @@ where
         _: &Multiaddr,
         remote_addr: &Multiaddr,
     ) -> Result<THandler<Self>, ConnectionDenied> {
+        // A peer caught spamming in this or a recent epoch gets no new
+        // connection. Denying (rather than a dummy handler) closes this
+        // connection only: the peer's old-epoch connection, if any, is not
+        // touched.
+        if let Some(until_epoch) = self.blocked_peers.get(&peer_id) {
+            tracing::debug!(target: LOG_TARGET, "Denying inbound connection {connection_id:?} with peer {peer_id:?} with addr {remote_addr:?} blocked until epoch {until_epoch:?}.");
+            return Err(ConnectionDenied::new(BlockedPeer {
+                peer_id,
+                until_epoch: *until_epoch,
+            }));
+        }
+
         // If the new peer makes the set of established connections too large, do not
         // try to upgrade the connection.
         if self.negotiated_peers.len() >= *self.peering_degree.end() {
@@ -1130,6 +1262,19 @@ where
         _: Endpoint,
         _: PortUse,
     ) -> Result<THandler<Self>, ConnectionDenied> {
+        // Reached only by a dial that was in flight when the verdict landed:
+        // `handle_pending_outbound_connection` refuses later dials before the
+        // transport handshake. The denial surfaces as `DialError::Denied`, an
+        // unrecoverable error, so the swarm picks another peer instead of
+        // retrying this one.
+        if let Some(until_epoch) = self.blocked_peers.get(&peer_id) {
+            tracing::debug!(target: LOG_TARGET, "Denying outbound connection {connection_id:?} with peer {peer_id:?} with addr {remote_addr:?} blocked until epoch {until_epoch:?}.");
+            return Err(ConnectionDenied::new(BlockedPeer {
+                peer_id,
+                until_epoch: *until_epoch,
+            }));
+        }
+
         // If the new peer makes the set of established connections too large, do not
         // try to upgrade the connection.
         if self.negotiated_peers.len() >= *self.peering_degree.end() {
@@ -1167,6 +1312,27 @@ where
         })
     }
 
+    /// Refuses to dial a blocked peer at all, so the node does not pay a
+    /// transport handshake for a connection it would deny once established.
+    fn handle_pending_outbound_connection(
+        &mut self,
+        connection_id: ConnectionId,
+        maybe_peer: Option<PeerId>,
+        _: &[Multiaddr],
+        _: Endpoint,
+    ) -> Result<Vec<Multiaddr>, ConnectionDenied> {
+        if let Some(peer_id) = maybe_peer
+            && let Some(until_epoch) = self.blocked_peers.get(&peer_id)
+        {
+            tracing::debug!(target: LOG_TARGET, "Refusing to dial peer {peer_id:?} on connection {connection_id:?}: blocked until epoch {until_epoch:?}.");
+            return Err(ConnectionDenied::new(BlockedPeer {
+                peer_id,
+                until_epoch: *until_epoch,
+            }));
+        }
+        Ok(vec![])
+    }
+
     /// Informs the behaviour about an event from the [`Swarm`].
     fn on_swarm_event(&mut self, event: FromSwarm) {
         if let FromSwarm::ConnectionClosed(ConnectionClosed {
diff --git a/blend/network/src/core/with_core/behaviour/tests/blocklist.rs b/blend/network/src/core/with_core/behaviour/tests/blocklist.rs
new file mode 100644
index 000000000..84ad6a680
--- /dev/null
+++ b/blend/network/src/core/with_core/behaviour/tests/blocklist.rs
@@ -0,0 +1,367 @@
+//! The list of peers blocked for spamming, which the behaviour owns: what a
+//! block denies, what it leaves alone, and how long it lasts.
+
+use core::time::Duration;
+
+use futures::StreamExt as _;
+use lb_blend_membership::Membership;
+use lb_blend_message::serialize_encapsulated_message_with_verified_public_header;
+use lb_cryptarchia_engine::Epoch;
+use lb_libp2p::SwarmEvent;
+use libp2p::swarm::dial_opts::{DialOpts, PeerCondition};
+use libp2p_swarm_test::SwarmExt as _;
+use test_log::test;
+use tokio::{select, time::sleep};
+
+use crate::core::{
+    tests::utils::{TestEncapsulatedMessageWithEpoch, TestProofsVerifier, TestSwarm},
+    with_core::behaviour::{
+        Event, MAX_BLOCK_DURATION_IN_EPOCHS, SpamReason,
+        tests::utils::{
+            BehaviourBuilder, SwarmExt as _, build_memberships, new_nodes_with_empty_address,
+        },
+    },
+};
+
+/// A first strike blocks until the next epoch, each further strike adds an
+/// epoch up to the cap, an expired block is lifted at the epoch transition, and
+/// strikes are forgotten once the peer is neither blocked nor a member.
+#[test(tokio::test)]
+async fn block_duration_escalates_per_strike_and_expires_with_epochs() {
+    let (mut identities, nodes) = new_nodes_with_empty_address(2);
+    let local_identity = identities.next().unwrap();
+    let remote_peer = nodes[1].id;
+    // The behaviour starts at epoch 0.
+    let mut behaviour = BehaviourBuilder::new(&local_identity)
+        .with_membership(&nodes)
+        .build();
+    let membership = Membership::new_without_local(&nodes);
+
+    behaviour.block_peer(remote_peer, SpamReason::InvalidProofOfQuota);
+    assert_eq!(behaviour.blocked_peers.get(&remote_peer), Some(&1.into()));
+    assert!(behaviour.is_blocked(&remote_peer));
+
+    // Lifted at epoch 1, but the strike is remembered while the peer is a member.
+    behaviour.start_new_epoch(
+        (membership.clone(), 1.into()),
+        TestProofsVerifier::accepting(),
+    );
+    assert!(!behaviour.is_blocked(&remote_peer));
+    assert_eq!(behaviour.spam_strikes.get(&remote_peer), Some(&1));
+
+    // Second strike: two epochs, so still blocked at epoch 2 and lifted at 3.
+    behaviour.block_peer(remote_peer, SpamReason::DuplicateMessage);
+    assert_eq!(behaviour.blocked_peers.get(&remote_peer), Some(&3.into()));
+    behaviour.start_new_epoch(
+        (membership.clone(), 2.into()),
+        TestProofsVerifier::accepting(),
+    );
+    assert!(behaviour.is_blocked(&remote_peer));
+    behaviour.start_new_epoch((membership, 3.into()), TestProofsVerifier::accepting());
+    assert!(!behaviour.is_blocked(&remote_peer));
+
+    // The escalation is capped.
+    for _ in 0..(MAX_BLOCK_DURATION_IN_EPOCHS + 2) {
+        behaviour.block_peer(remote_peer, SpamReason::UndeserializableMessage);
+    }
+    assert_eq!(
+        behaviour.blocked_peers.get(&remote_peer),
+        Some(&(3 + MAX_BLOCK_DURATION_IN_EPOCHS).into())
+    );
+
+    // A peer that leaves the membership while blocked stays blocked until its
+    // block expires, and its strikes are forgotten once it is neither.
+    let empty_membership = Membership::new_without_local(&[]);
+    behaviour.start_new_epoch(
+        (empty_membership.clone(), 4.into()),
+        TestProofsVerifier::accepting(),
+    );
+    assert!(behaviour.is_blocked(&remote_peer));
+    assert!(behaviour.spam_strikes.contains_key(&remote_peer));
+    behaviour.start_new_epoch(
+        (empty_membership, (3 + MAX_BLOCK_DURATION_IN_EPOCHS).into()),
+        TestProofsVerifier::accepting(),
+    );
+    assert!(!behaviour.is_blocked(&remote_peer));
+    assert!(!behaviour.spam_strikes.contains_key(&remote_peer));
+    assert_eq!(behaviour.num_blocked_peers(), 0);
+}
+
+/// A peer caught spamming is blocked when the verdict is issued, its new
+/// connections are denied in the same epoch, and it connects again once its
+/// block has expired at the next epoch.
+#[expect(clippy::too_many_lines, reason = "Test function.")]
+#[test(tokio::test)]
+async fn blocked_peer_is_denied_until_its_block_expires() {
+    let (mut identities, nodes) = new_nodes_with_empty_address(2);
+    let mut dialing_swarm = TestSwarm::new(&identities.next().unwrap(), |id| {
+        BehaviourBuilder::new(id).with_membership(&nodes).build()
+    });
+    let mut listening_swarm = TestSwarm::new(&identities.next().unwrap(), |id| {
+        BehaviourBuilder::new(id).with_membership(&nodes).build()
+    });
+    let dialing_peer_id = *dialing_swarm.local_peer_id();
+    let listening_peer_id = *listening_swarm.local_peer_id();
+
+    listening_swarm.listen().with_memory_addr_external().await;
+    dialing_swarm
+        .connect_and_wait_for_upgrade(&mut listening_swarm)
+        .await;
+
+    // Garbage over the current-epoch connection: a spammy verdict.
+    dialing_swarm
+        .behaviour_mut()
+        .force_send_serialized_message_to_current_epoch_peer(b"garbage".to_vec(), listening_peer_id)
+        .unwrap();
+
+    let mut blocked_event = false;
+    let mut connection_closed = false;
+    loop {
+        select! {
+            () = sleep(Duration::from_secs(10)) => { break; }
+            _ = dialing_swarm.select_next_some() => {}
+            event = listening_swarm.select_next_some() => {
+                match event {
+                    SwarmEvent::Behaviour(Event::PeerBlocked { peer_id, reason, until_epoch }) => {
+                        assert_eq!(peer_id, dialing_peer_id);
+                        assert_eq!(reason, SpamReason::UndeserializableMessage);
+                        // Behaviours start at epoch 0: a first strike lasts one epoch.
+                        assert_eq!(until_epoch, Epoch::from(1));
+                        blocked_event = true;
+                    }
+                    SwarmEvent::ConnectionClosed { peer_id, .. } if peer_id == dialing_peer_id => {
+                        connection_closed = true;
+                    }
+                    _ => {}
+                }
+            }
+        }
+        if blocked_event && connection_closed {
+            break;
+        }
+    }
+    assert!(blocked_event, "a spammy verdict must block the peer");
+    assert!(connection_closed, "the spammy connection must be closed");
+    assert!(listening_swarm.behaviour().is_blocked(&dialing_peer_id));
+    assert_eq!(listening_swarm.behaviour().num_blocked_peers(), 1);
+
+    // The blocked peer dials again: the connection is denied before any
+    // upgrade, and the dialer sees it closed.
+    let listening_address = listening_swarm
+        .external_addresses()
+        .next()
+        .cloned()
+        .expect("listening swarm must have an external address");
+    dialing_swarm
+        .dial(
+            DialOpts::peer_id(listening_peer_id)
+                .addresses(vec![listening_address.clone()])
+                .condition(PeerCondition::Always)
+                .build(),
+        )
+        .unwrap();
+
+    let mut denied = false;
+    let mut closed_on_dialer = false;
+    loop {
+        select! {
+            () = sleep(Duration::from_secs(10)) => { break; }
+            event = dialing_swarm.select_next_some() => {
+                if let SwarmEvent::ConnectionClosed { peer_id, .. } = event && peer_id == listening_peer_id {
+                    closed_on_dialer = true;
+                }
+            }
+            event = listening_swarm.select_next_some() => {
+                match event {
+                    SwarmEvent::IncomingConnectionError { .. } => {
+                        denied = true;
+                    }
+                    SwarmEvent::Behaviour(Event::InboundConnectionUpgradeSucceeded(peer_id)) => {
+                        panic!("connection with blocked peer {peer_id:?} must not be upgraded");
+                    }
+                    _ => {}
+                }
+            }
+        }
+        if denied && closed_on_dialer {
+            break;
+        }
+    }
+    assert!(
+        denied,
+        "the listening swarm must deny the blocked peer's connection"
+    );
+    assert!(
+        closed_on_dialer,
+        "the blocked peer must see its connection closed"
+    );
+    assert!(listening_swarm.behaviour().negotiated_peers.is_empty());
+
+    // The next epoch lifts the block, and the peer connects again.
+    let memberships = build_memberships(&[&dialing_swarm, &listening_swarm]);
+    listening_swarm.behaviour_mut().start_new_epoch(
+        (memberships[1].clone(), 1.into()),
+        TestProofsVerifier::accepting(),
+    );
+    assert!(!listening_swarm.behaviour().is_blocked(&dialing_peer_id));
+
+    let mut unblocked_event = false;
+    loop {
+        select! {
+            () = sleep(Duration::from_secs(5)) => { break; }
+            _ = dialing_swarm.select_next_some() => {}
+            event = listening_swarm.select_next_some() => {
+                if let SwarmEvent::Behaviour(Event::PeerUnblocked(peer_id)) = event {
+                    assert_eq!(peer_id, dialing_peer_id);
+                    unblocked_event = true;
+                    break;
+                }
+            }
+        }
+    }
+    assert!(
+        unblocked_event,
+        "an expired block must be reported to the swarm"
+    );
+
+    tokio::time::timeout(
+        Duration::from_secs(10),
+        dialing_swarm.connect_and_wait_for_upgrade(&mut listening_swarm),
+    )
+    .await
+    .expect("a peer whose block expired must be able to connect again");
+    assert!(
+        listening_swarm
+            .behaviour()
+            .negotiated_peers
+            .contains_key(&dialing_peer_id)
+    );
+}
+
+/// During the transition period a node holds two connections with the same
+/// peer. A spammy verdict on the new-epoch one closes that connection only:
+/// the old-epoch connection keeps carrying, and the node keeps processing, the
+/// previous epoch's messages until the transition period ends.
+#[expect(clippy::too_many_lines, reason = "Test function.")]
+#[test(tokio::test)]
+async fn verdict_on_new_epoch_connection_leaves_old_epoch_connection_open() {
+    let (mut identities, nodes) = new_nodes_with_empty_address(2);
+    let mut dialing_swarm = TestSwarm::new(&identities.next().unwrap(), |id| {
+        BehaviourBuilder::new(id).with_membership(&nodes).build()
+    });
+    let mut listening_swarm = TestSwarm::new(&identities.next().unwrap(), |id| {
+        BehaviourBuilder::new(id).with_membership(&nodes).build()
+    });
+    let dialing_peer_id = *dialing_swarm.local_peer_id();
+    let listening_peer_id = *listening_swarm.local_peer_id();
+
+    listening_swarm.listen().with_memory_addr_external().await;
+    dialing_swarm
+        .connect_and_wait_for_upgrade(&mut listening_swarm)
+        .await;
+    let old_epoch_connection = listening_swarm
+        .behaviour()
+        .negotiated_peers
+        .get(&dialing_peer_id)
+        .unwrap()
+        .connection_id;
+
+    // Both sides move to epoch 1; the connection above becomes old-epoch on
+    // both, and a new one is opened for the new epoch.
+    let memberships = build_memberships(&[&dialing_swarm, &listening_swarm]);
+    dialing_swarm.behaviour_mut().start_new_epoch(
+        (memberships[0].clone(), 1.into()),
+        TestProofsVerifier::accepting(),
+    );
+    listening_swarm.behaviour_mut().start_new_epoch(
+        (memberships[1].clone(), 1.into()),
+        TestProofsVerifier::accepting(),
+    );
+    dialing_swarm
+        .connect_and_wait_for_upgrade(&mut listening_swarm)
+        .await;
+    let new_epoch_connection = listening_swarm
+        .behaviour()
+        .negotiated_peers
+        .get(&dialing_peer_id)
+        .unwrap()
+        .connection_id;
+    assert_ne!(old_epoch_connection, new_epoch_connection);
+    assert!(
+        listening_swarm
+            .behaviour()
+            .old_epoch_peer_ids()
+            .unwrap()
+            .any(|peer_id| *peer_id == dialing_peer_id)
+    );
+
+    // Garbage over the new-epoch connection: the peer is blocked and that
+    // connection is closed.
+    dialing_swarm
+        .behaviour_mut()
+        .force_send_serialized_message_to_current_epoch_peer(b"garbage".to_vec(), listening_peer_id)
+        .unwrap();
+
+    let mut new_epoch_connection_closed = false;
+    loop {
+        select! {
+            () = sleep(Duration::from_secs(10)) => { break; }
+            _ = dialing_swarm.select_next_some() => {}
+            event = listening_swarm.select_next_some() => {
+                if let SwarmEvent::ConnectionClosed { peer_id, connection_id, .. } = event && peer_id == dialing_peer_id {
+                    assert_eq!(connection_id, new_epoch_connection, "only the offending connection may be closed");
+                    new_epoch_connection_closed = true;
+                    break;
+                }
+            }
+        }
+    }
+    assert!(new_epoch_connection_closed);
+    assert!(listening_swarm.behaviour().is_blocked(&dialing_peer_id));
+    assert!(
+        listening_swarm
+            .behaviour()
+            .old_epoch_peer_ids()
+            .unwrap()
+            .any(|peer_id| *peer_id == dialing_peer_id),
+        "the old-epoch connection must survive the verdict"
+    );
+
+    // An old-epoch message from the blocked peer is still delivered over the
+    // old-epoch connection.
+    let old_epoch_message = TestEncapsulatedMessageWithEpoch::new(0.into(), b"last-epoch");
+    dialing_swarm
+        .behaviour_mut()
+        .force_send_serialized_message_to_peer_at_epoch(
+            serialize_encapsulated_message_with_verified_public_header(&old_epoch_message),
+            listening_peer_id,
+            0.into(),
+        )
+        .unwrap();
+
+    let mut old_epoch_message_delivered = false;
+    loop {
+        select! {
+            () = sleep(Duration::from_secs(10)) => { break; }
+            _ = dialing_swarm.select_next_some() => {}
+            event = listening_swarm.select_next_some() => {
+                match event {
+                    SwarmEvent::Behaviour(Event::Message { sender, epoch, .. }) => {
+                        assert_eq!(sender, dialing_peer_id);
+                        assert_eq!(epoch, Epoch::from(0));
+                        old_epoch_message_delivered = true;
+                        break;
+                    }
+                    SwarmEvent::ConnectionClosed { peer_id, connection_id, .. } if peer_id == dialing_peer_id => {
+                        panic!("old-epoch connection {connection_id:?} must not be closed by a verdict on another connection");
+                    }
+                    _ => {}
+                }
+            }
+        }
+    }
+    assert!(
+        old_epoch_message_delivered,
+        "messages of the previous epoch must keep flowing over the old-epoch connection"
+    );
+}
diff --git a/blend/network/src/core/with_core/behaviour/tests/mod.rs b/blend/network/src/core/with_core/behaviour/tests/mod.rs
index 1f5b00bc3..601e68ee6 100644
--- a/blend/network/src/core/with_core/behaviour/tests/mod.rs
+++ b/blend/network/src/core/with_core/behaviour/tests/mod.rs
@@ -1,3 +1,4 @@
+mod blocklist;
 mod bootstrapping;
 mod connection_maintenance;
 mod epoch;
diff --git a/blend/network/src/core/with_core/behaviour/tests/utils.rs b/blend/network/src/core/with_core/behaviour/tests/utils.rs
index 7048f511c..f2e7183f3 100644
--- a/blend/network/src/core/with_core/behaviour/tests/utils.rs
+++ b/blend/network/src/core/with_core/behaviour/tests/utils.rs
@@ -175,6 +175,8 @@ impl BehaviourBuilder {
             message_cache: MessageCache::new(),
             proofs_verifier: Arc::new(self.proofs_verifier),
             pending_poq_verifications: PendingPoQVerifications::new(),
+            blocked_peers: HashMap::new(),
+            spam_strikes: HashMap::new(),
         }
     }
 }
diff --git a/libp2p/src/dial_error_ext.rs b/libp2p/src/dial_error_ext.rs
index 14c526ee4..a93cfb8aa 100644
--- a/libp2p/src/dial_error_ext.rs
+++ b/libp2p/src/dial_error_ext.rs
@@ -14,7 +14,10 @@ impl DialErrorExt for DialError {
             // The address handed resolves back to this node.
             | Self::LocalPeerId { .. }
             // There is no address to dial for this peer.
-            | Self::NoAddresses => false,
+            | Self::NoAddresses
+            // A behaviour refused the connection (the peer is blocked). Nothing
+            // we retry changes that verdict; a new epoch may.
+            | Self::Denied { .. } => false,
 
             // Per-address transport failures. Permanent only when every address
             // failed because we cannot speak its protocol at all; a timeout,
@@ -27,9 +30,7 @@ impl DialErrorExt for DialError {
             }
 
             // Local, transient conditions.
-            Self::Denied { .. }
-            | Self::Aborted
-            | Self::DialPeerConditionFalse(_) => true,
+            Self::Aborted | Self::DialPeerConditionFalse(_) => true,
         }
     }
 }
diff --git a/services/blend/src/core/backends/libp2p/behaviour.rs b/services/blend/src/core/backends/libp2p/behaviour.rs
index f6b0efaa4..336aac805 100644
--- a/services/blend/src/core/backends/libp2p/behaviour.rs
+++ b/services/blend/src/core/backends/libp2p/behaviour.rs
@@ -1,16 +1,20 @@
 use lb_blend::scheduling::membership::Membership;
 use lb_chain_service::Epoch;
 use lb_libp2p::NetworkBehaviour;
-use libp2p::{PeerId, allow_block_list::BlockedPeers};
+use libp2p::PeerId;
 
 use crate::core::{
     backends::libp2p::Libp2pBlendBackendSettings, settings::RunningBlendConfig as BlendConfig,
 };
 
+/// The blend behaviour owns the list of peers blocked for spamming (see
+/// `lb_blend::network::core::with_core::behaviour::Behaviour::is_blocked`), so
+/// no `allow_block_list` behaviour is composed here: a verdict closes only the
+/// offending connection, and the block's lifetime is bound to epochs by the
+/// component that issues the verdicts.
 #[derive(NetworkBehaviour)]
 pub struct BlendBehaviour<ObservationWindowProvider, ProofsVerifier> {
     pub blend: lb_blend::network::core::NetworkBehaviour<ObservationWindowProvider, ProofsVerifier>,
-    pub blocked_peers: libp2p::allow_block_list::Behaviour<BlockedPeers>,
 }
 
 impl<ObservationWindowProvider, ProofsVerifier>
@@ -57,7 +61,6 @@ where
                 config.peer_id(),
                 config.backend.protocol_name.clone().into_inner(),
             ),
-            blocked_peers: libp2p::allow_block_list::Behaviour::default(),
         }
     }
 }
diff --git a/services/blend/src/core/backends/libp2p/swarm.rs b/services/blend/src/core/backends/libp2p/swarm.rs
index f6f932324..e753455c8 100644
--- a/services/blend/src/core/backends/libp2p/swarm.rs
+++ b/services/blend/src/core/backends/libp2p/swarm.rs
@@ -19,7 +19,7 @@ use lb_blend::{
         with_core::{
             behaviour::{
                 ConnectionUpgradeFailureReason, Event as CoreToCoreEvent, IntervalStreamProvider,
-                NegotiatedPeerState,
+                NegotiatedPeerState, SpamReason,
             },
             error::SendError,
         },
@@ -238,14 +238,16 @@ where
             return;
         }
 
-        let negotiated_peers = self.behaviour().blend.with_core().negotiated_peers().keys();
+        let core_behaviour = self.behaviour().blend.with_core();
+        let negotiated_peers = core_behaviour.negotiated_peers().keys();
+        let num_blocked_peers = core_behaviour.num_blocked_peers();
 
         // We need to clone else we would not be able to call `self.dial` below, which
         // requires access to `&mut self`.
         let current_membership = self.current_epoch_info.membership.clone();
 
         let exclude_peers: HashSet<PeerId> = negotiated_peers
-            .chain(self.swarm.behaviour().blocked_peers.blocked_peers())
+            .chain(core_behaviour.blocked_peers())
             .chain(self.ongoing_dials.keys())
             .chain(self.unrecoverable_peers.iter())
             .chain(except.iter())
@@ -261,6 +263,21 @@ where
 
         let no_more_peers_to_dial = peers_to_dial.peek().is_none();
 
+        // A node whose blocklist has swallowed its membership would otherwise
+        // look idle: it needs connections, and finds nobody to dial.
+        if no_more_peers_to_dial && num_blocked_peers > 0 {
+            tracing::warn!(
+                target: LOG_TARGET,
+                diagnostic = BLEND_REACHABILITY,
+                event = "blend_no_dialable_peers",
+                epoch = u32::from(self.current_epoch_info.epoch),
+                membership_size = current_membership.size(),
+                blocked_peers = num_blocked_peers,
+                needed_connections = amount,
+                "No member is dialable: every member is negotiated, being dialed, unreachable, or blocked for spamming."
+            );
+        }
+
         // When no membership peer is eligible to be dialed but we still have peers
         // we gave up on earlier in this dial cycle (`except`), we want to clear
         // that memory and retry the whole membership from scratch. Rather than
@@ -424,16 +441,26 @@ where
         self.dial_random_peers_except(connections_to_establish, except);
     }
 
+    /// A spammy peer was already blocked by the behaviour when the verdict was
+    /// issued (`Event::PeerBlocked`); here only its connection slot is
+    /// refilled.
     fn handle_disconnected_peer(&mut self, peer_id: PeerId, peer_state: NegotiatedPeerState) {
         tracing::trace!(target: LOG_TARGET, "Peer {peer_id} disconnected with state {peer_state:?}.");
-        if let NegotiatedPeerState::Spammy(reason) = peer_state {
-            tracing::debug!(target: LOG_TARGET, "Blocking spammy peer {peer_id} for reason {reason:?}.");
-            self.swarm.behaviour_mut().blocked_peers.block_peer(peer_id);
-            metrics::core_peer_blocked(reason.as_str());
-        }
         self.check_and_dial_new_peers_except(&HashSet::from([peer_id]));
     }
 
+    fn handle_blocked_peer(&self, peer_id: PeerId, reason: SpamReason, until_epoch: Epoch) {
+        tracing::debug!(target: LOG_TARGET, "Peer {peer_id} blocked for reason {reason:?} until epoch {until_epoch:?}.");
+        metrics::core_peer_blocked(reason.as_str());
+        metrics::core_peers_blocked(self.num_blocked_peers());
+    }
+
+    fn handle_unblocked_peer(&self, peer_id: PeerId) {
+        tracing::debug!(target: LOG_TARGET, "Peer {peer_id} unblocked: its block expired.");
+        metrics::core_peer_unblocked();
+        metrics::core_peers_blocked(self.num_blocked_peers());
+    }
+
     fn collect_network_info(&self) -> NetworkInfo<PeerId> {
         let core_behaviour = self.swarm.behaviour().blend.with_core();
         let current_epoch_peers = core_behaviour
@@ -479,6 +506,16 @@ where
             ) => {
                 self.handle_disconnected_peer(peer_id, peer_state);
             }
+            lb_blend::network::core::with_core::behaviour::Event::PeerBlocked {
+                peer_id,
+                reason,
+                until_epoch,
+            } => {
+                self.handle_blocked_peer(peer_id, reason, until_epoch);
+            }
+            lb_blend::network::core::with_core::behaviour::Event::PeerUnblocked(peer_id) => {
+                self.handle_unblocked_peer(peer_id);
+            }
             lb_blend::network::core::with_core::behaviour::Event::OutboundConnectionUpgradeFailed { peer, reason } => {
                 match reason {
                     reason @ ConnectionUpgradeFailureReason::ConnectionFailure => {
@@ -635,6 +672,8 @@ where
                 self.pending_retries.clear();
                 self.unrecoverable_peers.clear();
                 self.pending_full_membership_retry = None;
+                // Expired blocks are lifted by the behaviour in `start_new_epoch`,
+                // so the peers concerned are dialable again right here.
                 self.check_and_dial_new_peers();
             }
             BlendSwarmMessage::CompleteEpochTransition => {
@@ -944,6 +983,10 @@ where
         self.swarm.behaviour().blend.with_core().num_healthy_peers()
     }
 
+    fn num_blocked_peers(&self) -> usize {
+        self.swarm.behaviour().blend.with_core().num_blocked_peers()
+    }
+
     fn available_connection_slots(&self) -> usize {
         self.swarm
             .behaviour()
diff --git a/services/blend/src/core/backends/libp2p/tests/network_maintenance.rs b/services/blend/src/core/backends/libp2p/tests/network_maintenance.rs
index 6dbe8ee0c..943bf92e4 100644
--- a/services/blend/src/core/backends/libp2p/tests/network_maintenance.rs
+++ b/services/blend/src/core/backends/libp2p/tests/network_maintenance.rs
@@ -5,12 +5,22 @@ use lb_blend::{
 };
 use libp2p::core::Endpoint;
 use test_log::test;
-use tokio::{select, time::sleep};
+use tokio::{
+    select,
+    time::{sleep, timeout},
+};
 
 use crate::{
-    core::backends::libp2p::{
-        core_swarm_test_utils::{new_nodes_with_empty_address, update_nodes},
-        tests::utils::{BlendBehaviourBuilder, SwarmBuilder, SwarmExt as _, TestSwarm},
+    core::backends::{
+        BackendEpochInfo,
+        libp2p::{
+            core_swarm_test_utils::{new_nodes_with_empty_address, update_nodes},
+            swarm::BlendSwarmMessage,
+            tests::utils::{
+                BlendBehaviourBuilder, InnerSwarm, SwarmBuilder, SwarmExt as _, TestProofsVerifier,
+                TestSwarm, build_membership,
+            },
+        },
     },
     test_utils::TestEncapsulatedMessage,
 };
@@ -194,9 +204,9 @@ async fn on_malicious_peer() {
     assert!(
         listening_swarm
             .behaviour()
-            .blocked_peers
-            .blocked_peers()
-            .contains(malicious_swarm.local_peer_id())
+            .blend
+            .with_core()
+            .is_blocked(malicious_swarm.local_peer_id())
     );
 
     // We check that the malicious peer has no entry in the set of negotiated peers.
@@ -224,3 +234,206 @@ async fn on_malicious_peer() {
     );
     assert_eq!(second_swarm_connection_details.role(), Endpoint::Listener);
 }
+
+/// Polls both swarms until `condition` holds on the first one or `duration`
+/// elapses, returning whether it held.
+async fn poll_both_until(
+    first: &mut InnerSwarm,
+    second: &mut InnerSwarm,
+    duration: Duration,
+    condition: impl Fn(&InnerSwarm) -> bool + Send + Sync,
+) -> bool {
+    timeout(duration, async {
+        // Checked before every poll rather than after the first swarm's
+        // events only: the condition may already hold, or become true while
+        // the other swarm is the one making progress.
+        while !condition(first) {
+            select! {
+                () = first.poll_next() => {}
+                () = second.poll_next() => {}
+            }
+        }
+    })
+    .await
+    .is_ok()
+}
+
+/// Two core nodes. The listening one rejects every `PoQ`, which is what a
+/// node does to the messages of a peer whose proofs were generated for a
+/// different epoch than the one the connection was accepted under. The dialing
+/// one connects and sends a single message, and ends up blocked.
+async fn block_dialing_peer_for_invalid_poq() -> (
+    InnerSwarm,
+    InnerSwarm,
+    tokio::sync::mpsc::Sender<BlendSwarmMessage<TestProofsVerifier>>,
+    Vec<Node<libp2p::PeerId>>,
+) {
+    let (mut identities, mut nodes) = new_nodes_with_empty_address(2);
+
+    let TestSwarm {
+        swarm: mut dialing_swarm,
+        ..
+    } = SwarmBuilder::new(identities.next().unwrap(), &nodes)
+        .build(|id, membership| BlendBehaviourBuilder::new(id, membership).build());
+    let (dialing_node, _) = dialing_swarm.listen_and_return_membership_entry(None).await;
+    update_nodes(&mut nodes, &dialing_node.id, dialing_node.address.clone());
+
+    let TestSwarm {
+        swarm: mut listening_swarm,
+        swarm_message_sender,
+        ..
+    } = SwarmBuilder::new(identities.next().unwrap(), &nodes).build(|id, membership| {
+        BlendBehaviourBuilder::new(id, membership)
+            .with_rejecting_proofs_verifier()
+            .build()
+    });
+    let (listening_node, _) = listening_swarm
+        .listen_and_return_membership_entry(None)
+        .await;
+    update_nodes(
+        &mut nodes,
+        &listening_node.id,
+        listening_node.address.clone(),
+    );
+
+    dialing_swarm.dial_peer_at_addr(listening_node.id, listening_node.address.clone());
+    assert!(
+        poll_both_until(
+            &mut listening_swarm,
+            &mut dialing_swarm,
+            Duration::from_secs(5),
+            |swarm| {
+                swarm
+                    .behaviour()
+                    .blend
+                    .with_core()
+                    .negotiated_peers()
+                    .contains_key(&dialing_node.id)
+            }
+        )
+        .await,
+        "connection was not negotiated by the listening swarm"
+    );
+    // The handler events are asynchronous, so the dialing side may learn of
+    // the negotiation a little later than the listening side.
+    assert!(
+        poll_both_until(
+            &mut dialing_swarm,
+            &mut listening_swarm,
+            Duration::from_secs(5),
+            |swarm| {
+                swarm
+                    .behaviour()
+                    .blend
+                    .with_core()
+                    .negotiated_peers()
+                    .contains_key(&listening_node.id)
+            }
+        )
+        .await,
+        "connection was not negotiated by the dialing swarm"
+    );
+
+    dialing_swarm
+        .behaviour_mut()
+        .blend
+        .with_core_mut()
+        .force_send_message_to_current_epoch_peer(
+            &TestEncapsulatedMessage::new(b"proof-for-another-epoch"),
+            listening_node.id,
+        )
+        .unwrap();
+
+    // The block is issued with the verdict, before the connection is torn
+    // down, so wait for both.
+    assert!(
+        poll_both_until(
+            &mut listening_swarm,
+            &mut dialing_swarm,
+            Duration::from_secs(10),
+            |swarm| {
+                let core_behaviour = swarm.behaviour().blend.with_core();
+                core_behaviour.is_blocked(&dialing_node.id)
+                    && !core_behaviour
+                        .negotiated_peers()
+                        .contains_key(&dialing_node.id)
+            }
+        )
+        .await,
+        "a peer whose PoQ fails verification must be blocked and disconnected"
+    );
+
+    (listening_swarm, dialing_swarm, swarm_message_sender, nodes)
+}
+
+#[test(tokio::test)]
+async fn peer_with_invalid_poq_is_blocked() {
+    let (listening_swarm, dialing_swarm, _sender, _nodes) =
+        block_dialing_peer_for_invalid_poq().await;
+    assert!(
+        listening_swarm
+            .behaviour()
+            .blend
+            .with_core()
+            .is_blocked(dialing_swarm.local_peer_id())
+    );
+    // The spammy connection was closed, and nothing else is negotiated.
+    assert_eq!(
+        listening_swarm
+            .behaviour()
+            .blend
+            .with_core()
+            .num_negotiated_peers(),
+        0
+    );
+}
+
+/// A block must not outlive the epoch it was issued in: the next epoch
+/// rebuilds the membership and brings new `PoQ` public inputs, under which the
+/// rejected peer's messages may well verify. Here the new epoch's verifier
+/// accepts everything and the blocked peer is the only other member, so the
+/// listening swarm must unblock it and negotiate with it again.
+#[test(tokio::test)]
+async fn blocked_peer_is_dialable_again_in_next_epoch() {
+    let (mut listening_swarm, mut dialing_swarm, swarm_message_sender, nodes) =
+        block_dialing_peer_for_invalid_poq().await;
+    let dialing_peer_id = *dialing_swarm.local_peer_id();
+    let listening_peer_id = *listening_swarm.local_peer_id();
+
+    swarm_message_sender
+        .send(BlendSwarmMessage::StartNewEpoch(BackendEpochInfo {
+            membership: build_membership(&nodes, Some(listening_peer_id)),
+            epoch: 2.into(),
+            proofs_verifier: TestProofsVerifier::default(),
+        }))
+        .await
+        .unwrap();
+
+    let renegotiated = poll_both_until(
+        &mut listening_swarm,
+        &mut dialing_swarm,
+        Duration::from_secs(5),
+        |swarm| {
+            swarm
+                .behaviour()
+                .blend
+                .with_core()
+                .negotiated_peers()
+                .contains_key(&dialing_peer_id)
+        },
+    )
+    .await;
+
+    assert!(
+        !listening_swarm
+            .behaviour()
+            .blend
+            .with_core()
+            .is_blocked(&dialing_peer_id),
+        "a peer blocked in epoch 1 is still blocked in epoch 2"
+    );
+    assert!(
+        renegotiated,
+        "the listening swarm did not reconnect to the peer it blocked in the previous epoch"
+    );
+}
diff --git a/services/blend/src/core/backends/libp2p/tests/redials.rs b/services/blend/src/core/backends/libp2p/tests/redials.rs
index 3bc22bc18..8c9ea4221 100644
--- a/services/blend/src/core/backends/libp2p/tests/redials.rs
+++ b/services/blend/src/core/backends/libp2p/tests/redials.rs
@@ -325,7 +325,7 @@ async fn core_epoch_rotation_clears_pending_retries() {
     let new_epoch_info = BackendEpochInfo {
         membership: Membership::new_without_local(&[]),
         epoch: 2.into(),
-        proofs_verifier: TestProofsVerifier,
+        proofs_verifier: TestProofsVerifier::default(),
     };
     swarm_message_sender
         .send(BlendSwarmMessage::StartNewEpoch(new_epoch_info))
@@ -403,7 +403,7 @@ async fn core_does_not_give_up_below_minimum_peering_degree() {
         .send(BlendSwarmMessage::StartNewEpoch(BackendEpochInfo {
             membership: new_membership,
             epoch: 2.into(),
-            proofs_verifier: TestProofsVerifier,
+            proofs_verifier: TestProofsVerifier::default(),
         }))
         .await
         .unwrap();
diff --git a/services/blend/src/core/backends/libp2p/tests/utils.rs b/services/blend/src/core/backends/libp2p/tests/utils.rs
index 7b32dd446..c27d96416 100644
--- a/services/blend/src/core/backends/libp2p/tests/utils.rs
+++ b/services/blend/src/core/backends/libp2p/tests/utils.rs
@@ -22,9 +22,7 @@ use lb_blend::{
 use lb_chain_service::Epoch;
 use lb_key_management_system_service::keys::UnsecuredEd25519Key;
 use lb_libp2p::{Protocol, SwarmEvent};
-use libp2p::{
-    Multiaddr, PeerId, Swarm, allow_block_list, core::transport::ListenerId, identity::Keypair,
-};
+use libp2p::{Multiaddr, PeerId, Swarm, core::transport::ListenerId, identity::Keypair};
 use libp2p_swarm_test::SwarmExt as _;
 use rand::SeedableRng as _;
 use rand_chacha::ChaCha20Rng;
@@ -46,18 +44,21 @@ use crate::{
 };
 
 /// A `PoQ` verifier for the swarm tests, which accepts every proof it is
-/// handed.
+/// handed unless `reject_all` is set.
 ///
-/// What a node does with a *failed* verification is decided by the behaviour,
-/// so it is tested there rather than here.
-#[derive(Debug, Clone, Copy)]
-pub struct TestProofsVerifier;
+/// Rejecting every proof is what a verifier does to messages whose proofs
+/// were generated against another epoch's public inputs, which is how the
+/// swarm's reaction to a failed verification (blocking the sender) is tested.
+#[derive(Debug, Clone, Copy, Default)]
+pub struct TestProofsVerifier {
+    pub reject_all: bool,
+}
 
 impl ProofsVerifier for TestProofsVerifier {
     type Error = ();
 
     fn new(_public_inputs: PoQVerificationInputsMinusSigningKey) -> Self {
-        Self
+        Self::default()
     }
 
     fn verify_proof_of_quota(
@@ -65,6 +66,9 @@ impl ProofsVerifier for TestProofsVerifier {
         proof: ProofOfQuota,
         _signing_key: &lb_key_management_system_service::keys::Ed25519PublicKey,
     ) -> Result<VerifiedProofOfQuota, Self::Error> {
+        if self.reject_all {
+            return Err(());
+        }
         Ok(VerifiedProofOfQuota::from_proof_of_quota_unchecked(proof))
     }
 
@@ -141,7 +145,7 @@ impl SwarmBuilder {
         let public_info = BackendEpochInfo {
             membership: build_membership(membership, Some(identity.public().into())),
             epoch: 1.into(),
-            proofs_verifier: TestProofsVerifier,
+            proofs_verifier: TestProofsVerifier::default(),
         };
         Self {
             identity,
@@ -213,7 +217,7 @@ impl BlendBehaviourBuilder {
             membership,
             observation_window: None,
             peering_degree: None,
-            proofs_verifier: TestProofsVerifier,
+            proofs_verifier: TestProofsVerifier::default(),
         }
     }
 
@@ -231,6 +235,13 @@ impl BlendBehaviourBuilder {
         self
     }
 
+    /// Every `PoQ` this behaviour verifies fails, as if the sender's proofs
+    /// were generated for a different epoch.
+    pub fn with_rejecting_proofs_verifier(mut self) -> Self {
+        self.proofs_verifier = TestProofsVerifier { reject_all: true };
+        self
+    }
+
     pub fn build(self) -> BlendBehaviour<TestObservationWindowProvider, TestProofsVerifier> {
         let observation_window_values = self
             .observation_window
@@ -261,7 +272,6 @@ impl BlendBehaviourBuilder {
                 self.peer_id,
                 PROTOCOL_NAME,
             ),
-            blocked_peers: allow_block_list::Behaviour::default(),
         }
     }
 }
diff --git a/services/blend/src/metrics.rs b/services/blend/src/metrics.rs
index 360786d6b..733d43781 100644
--- a/services/blend/src/metrics.rs
+++ b/services/blend/src/metrics.rs
@@ -71,6 +71,17 @@ mod imp {
         lb_tracing::increase_counter_u64!(blend_core_peers_blocked_total, 1, reason = reason);
     }
 
+    /// Reports a core peer whose block expired at an epoch transition.
+    pub fn core_peer_unblocked() {
+        lb_tracing::increase_counter_u64!(blend_core_peers_unblocked_total, 1);
+    }
+
+    /// Number of core peers currently blocked for spamming. A value close to
+    /// the membership size means the node has nobody left to dial.
+    pub fn core_peers_blocked(count: usize) {
+        lb_tracing::metric_observable_gauge_u64_set!(blend_core_peers_blocked, count as u64);
+    }
+
     /// Reports a data payload the Blend network failed to deliver within the
     /// delivery deadline, and that this node therefore broadcast in the clear.
     pub fn data_payload_bypassed_blend(payload_type: DataPayloadType) {
```

## Appendix C — Draft issue for logos-lips (item 5, not filed)

> Title: blend-protocol: the Connectivity Maintenance blacklist has no lifetime and no rule for a blacklist covering the membership
> 
> Document: `docs/blockchain/raw/blend-protocol.md` @ 7244d3b05ddec91a4a7b565bd5a9340ab77ededd
> 
> §Connectivity Maintenance, item 3.3 (L559) says of a neighbour marked spammy:
> 
> > The neighbor is added to a black list, and its selection must be avoided.
> 
> Three things are left open, and the reference implementation (logos-blockchain `services/blend/src/core/backends/libp2p/swarm.rs`, `handle_disconnected_peer`) currently resolves them in the most restrictive way, which produces a self-inflicted partition:
> 
> 1. **Lifetime.** Nothing says when a blacklisted neighbour may be selected again. The same document rebuilds the membership and all connections at the start of every epoch (§Core Network, Bootstrapping, L307-L321; §Transition Period, L602), and every verdict a node can issue is evaluated against epoch-scoped data: the PoQ public inputs (`core_root`, `pol_epoch_nonce`, `pol_ledger_aged`, `d_blend`) change per epoch, and the observation-window thresholds are derived from the epoch's membership size. A block issued in epoch `e` therefore outlives its evidence in epoch `e+1`. The node implements the blacklist as permanent for the life of the process, and since honest nodes can be classified as spammy across an epoch boundary (a message generated under the new epoch's inputs arriving over a connection the receiver still holds as old-epoch, or vice versa), the blacklist grows monotonically until a node has nobody left to dial.
> 
>    Proposal: state that a blacklist entry expires at the start of the next epoch, and that a neighbour blacklisted `k` times in a row stays blacklisted for `min(k, K_max)` epochs, `K_max` being a global parameter (the implementation under review uses 4). A Sybil core identity costs the SDP minimum stake, so escalation per identity is the right unit.
> 
> 2. **Exhaustion.** Nothing says what a node does when every eligible member is blacklisted (or unreachable). §Connectivity Maintenance item 5 requires opening a connection with "a new randomly selected core node", which is impossible in that state; the node then stops dialing silently.
> 
>    Proposal: state that when fewer than `Φ_CC^Min` members are selectable the node logs the condition, keeps the connections it has, and retries the full membership after a fixed cooldown; and that it must not lift blacklist entries early to get out of that state, since that would let a spammer force its own re-selection by exhausting the rest.
> 
> 3. **Scope of the verdict.** §Transition Period (L602-L603) requires a node to open new connections for the new epoch while maintaining the old ones for `T` rounds, so a node has two connections to the same neighbour during the transition. The text does not say whether a spammy verdict on one of them (the new-epoch one, typically) also closes the other. Closing both discards the last-epoch messages the old connection was carrying, which §Transition Period says must be processed. The reference implementation closes both (`libp2p::allow_block_list::block_peer` closes every connection to the peer).
> 
>    Proposal: state that the verdict closes the connection it was issued on, that the blacklist only prevents *new* connections with the neighbour, and that an old-epoch connection is left to expire with the transition period.
> 
> Related: item 3.1 says the neighbour "is the true source of messages", which is true of the transport but not of the epoch the messages were generated for; a note that a PoQ failure during the transition period must be re-checked against the other epoch's public inputs before it counts as a spammy verdict would close the false-positive path described above.
> 
> Reported from an audit of logos-blockchain @ a805329f8a186eb6989f09a7c49dee4a0e07473b (message-board issues #101 and #117).
