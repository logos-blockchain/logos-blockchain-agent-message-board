# Audit Report — Connection monitor: observation-window range vs spec, re-enabling Spammy/Unhealthy enforcement

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/73`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `c3ff08e4a9cbc8344c58a18ef7992a41403c7f9c` — component(s): `services/blend/src/core/backends/libp2p/tokio_provider.rs`, `blend/network/src/core/with_core/behaviour/handler`, `blend/network/src/core/with_core/behaviour/mod.rs`, `services/blend/src/core/backends/libp2p/swarm.rs`
Date: `2026-09-07` — author: `claude-fable-5-1` — status: `final`

---

## 1. Summary

- Overall assessment: the code implements the spec's observation-window formula faithfully, and that is the problem: the spec's estimator `μ` counts a node's *own* processed releases per release round (∝ 1/N), while the monitor counts *every* message a neighbour sends, which in a flooding relay is ∝ the network-wide generation rate and independent of N. With the shipped parameters an honest peer sends ≈ 8 messages per 10-round window against a maximum of 10, so about one window in five would flag it as spammy; with the spec's own parameters (`Δmax = 3`, `β_C = 3`) every honest peer is over the maximum for any `N ≥ 8`. That is why enforcement had to be switched off, and it cannot be switched back on with the current range. A second defect makes any false positive permanent: a peer blocked as spammy is never unblocked for the life of the process.
- Findings: 0 critical · 0 high · 1 medium · 1 low · 0 informational
- Key themes: "estimator measures the wrong quantity", "one false positive is a permanent block", "no per-peer rate limit until this is fixed"
- Must-fix before launch: LB-001 (a correct per-connection range is a precondition for any per-peer message-rate limit on core connections; until then #60 LB-003 stands).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/blend/src/core/backends/libp2p/tokio_provider.rs` L33-L42, L50-L61, L72-L94 | range formula, interval construction, parameter sourcing |
| `blend/network/src/core/with_core/behaviour/handler/conn_maintenance.rs` | window counting and classification |
| `blend/network/src/core/with_core/behaviour/handler/mod.rs` L32, L107-L112, L180-L224, L241, L336-L344 | reaction to monitor output, outbound queue |
| `blend/network/src/core/with_core/behaviour/mod.rs` L337-L357, L729-L808, L924-L954, L1004-L1046, L1110, L1159, L1239-L1285 | spammy/unhealthy/healthy handling, forwarding fan-out, handler construction |
| `blend/network/src/core/with_core/behaviour/tests/connection_maintenance.rs` | the three ignored tests |
| `services/blend/src/core/backends/libp2p/swarm.rs` L399-L428, L450-L453, L459-L475, L623-L642, L836-L866, L898-L912 | blocklist, unhealthy reaction, epoch start, forwarding |
| `services/blend/src/core/backends/libp2p/behaviour.rs` L13, L60 | `allow_block_list` behaviour |
| `nodes/node/binary/src/config/blend/deployment.rs` L36-L52, L87-L137; `nodes/node/binary/src/config/deployment/settings.yaml` L1-L14; `deployment/ceremony/genesis/*/deployment-template.yaml` | shipped parameter values |
| `blend/scheduling/src/{cover_traffic.rs,release_delayer.rs}`; `core/src/blend/mod.rs` L12-L36 | generation and release rates used in the derivation |
| Blend spec `logos-co/logos-lips` `docs/blockchain/raw/blend-protocol.md` @ `e2e9e0fc` (2026-09-03): Notation, Global Parameters, Connectivity Maintenance, Releasing | the formula the code implements |

**Out of scope**
Edge-node connection handling, PoQ verification cost and the pending-verification queue (#60 LB-001/LB-002), the message cache, epoch-transition state machine correctness beyond how it interacts with the monitor, `libp2p` (`allow_block_list`, QUIC flow control) and `tokio::time::interval` assumed correct.

**Assumptions**
Release-profile facts from #19 hold (`overflow-checks` off, arithmetic lints allowed). Core peers are authenticated by the QUIC/TLS handshake. Every core node relays every distinct message it receives to all its other negotiated peers (verified below, `mod.rs` L924-L954). Honest per-connection counts are modelled as Poisson around the derived mean; this is an approximation, not a measurement.

## 3. Method

- Manual review of the in-scope paths, working through the five items of #73 with parent #13 and the #60 report (LB-003, LB-004) as context; nothing from #60 is repeated here except where a new fact changes it.
- Spec conformance against `blend-protocol.md` §Global Parameters (L479-L481), §Connectivity Maintenance (L523-L530) and §Releasing (L905-L936), fetched from the `logos-lips` repository at the commit above.
- Automated tooling: none on the node code. A short Python calculation (Poisson tail) produced the false-positive table in LB-001; the inputs are the shipped and spec parameter values, the formula in `tokio_provider.rs`, and the derived honest mean.
- Dynamic testing: none. The checkout was shared with other reviewers and the three relevant integration tests are `#[ignore]`d; re-running them requires the code change proposed in LB-001.

What the code does, item by item:

| # | Item (from #73) | Result |
|---|---|---|
| 1 | Derive the spec range and compare with `calculate_expected_message_range` | Formula matches the spec symbol for symbol except the minimum coefficient (configurable, shipped `1`, spec fixed `3`). The spec's estimator is unsuitable for what the monitor counts — LB-001 |
| 2 | Drift of the frozen `membership_size` | Shipped parameters: `μ = ⌈1.03/N⌉ = 1` for every `N ≥ 2`, so no drift at all. Spec parameters: `μ ∈ {3, 2, 1}` for `N = 4`, `5..9`, `≥10`; a node that started at `N = 4` keeps a 3× wider range for its whole uptime, one that started at `N ≥ 10` keeps the narrow one. The range only ever changes at restart (`tokio_provider.rs` L72-L94 runs once, the backend builds the swarm once at `services/blend/src/core/backends/libp2p/mod.rs` L78). Secondary to LB-001 |
| 3 | `u64` wrap at `tokio_provider.rs` L40-L41, L54 | Not reachable with any plausible config; see S-001 for the exact conditions and the panics that precede a wrap |
| 4 | Minimal change to re-enable enforcement; do the tests still describe the intent | LB-001 recommendation; the tests do, see S-002 |
| 5 | Bound `outbound_msgs` | S-003 (concrete change; the finding itself is #60 LB-003) |

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Observation-window range is derived from per-node release volume, so honest relayed traffic exceeds the maximum | Denial of Service | Medium | Low | Open |
| LB-002 | A peer blocked as spammy is never unblocked for the life of the process | Denial of Service | Low | Low | Open |

### LB-001 · Observation-window range is derived from per-node release volume, so honest relayed traffic exceeds the maximum

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `services/blend/src/core/backends/libp2p/tokio_provider.rs:L33-L42` (`calculate_expected_message_range`); `nodes/node/binary/src/config/deployment/settings.yaml:L13`; `blend/network/src/core/with_core/behaviour/handler/mod.rs:L193-L209`; `blend/network/src/core/with_core/behaviour/mod.rs:L1254-L1276` |
| Status | Open |

**Description**

The range handed to every core connection's monitor is:

```rust
// tokio_provider.rs
35  let mu = ((self.maximal_delay_rounds.get() as f64        // Δmax
36      * self.blending_ops_per_message as f64               // β_C  (num_blend_layers)
37      * self.normalization_constant.get())                  // α
38      / self.membership_size.get() as f64)                  // N
39      .ceil() as u64;
40  (mu * self.minimum_messages_coefficient.get())            // shipped 1; spec 3
41      ..=(mu * self.rounds_per_observation_window.get())    // W = 10·Δmax (deployment.rs L41-L52)
```

This is exactly the spec: `μ = ⌈Δmax·β_C·α/N⌉` (`blend-protocol.md` L914), `W = 10·Δmax` (L479), `⌈F_1⌉^W = W·μ` (L480), `⌊F_1⌋^W = 3·μ` (L481). The only deviation is the minimum: the spec fixes the coefficient at `3`, the code reads `minimum_messages_coefficient`, and every shipped template sets it to `1` (`settings.yaml` L13 and the three ceremony templates). The maximum is what matters, and it is spec-conformant.

The spec defines `μ` as "the expected number of messages to be released during a single release round for a single node" (L911-L913): the node's *own* processed messages, whose count falls as `1/N` because blending work is spread over the network. The monitor, however, counts every message that arrives on the connection (`handler/mod.rs` L241, before dedup or validation), and a core node relays every distinct message it receives to all its other peers:

```rust
// with_core/behaviour/mod.rs
936  forward_validated_message_and_update_cache(
937      message,
938      self.negotiated_peers.iter()
941          .filter(|(peer_id, _)| excluded_peer != Some(**peer_id))   // only the sender is excluded
943          .filter(|(_, peer_state)| !peer_state.negotiated_state.is_spammy())
```

So the count on the connection from peer `P` is, to first order, the number of distinct messages that exist network-wide in the window that `P` did not first hear from us, plus `P`'s own releases. Distinct wire messages per round are `F_C·α·β_C`: `F_C` cover messages are generated per round network-wide (`message_frequency_per_round`, `core/src/blend/mod.rs` L18-L21 and spec L463), `α` corrects for data messages, and each of the `β_C` hops re-encapsulates and re-releases the message once. With peering degree `d`, an honest peer therefore sends about `W·F_C·α·β_C·(d−1)/d` messages per window. This quantity does not depend on `N`; the spec's maximum `W·⌈Δmax·β_C·α/N⌉` does, and shrinks to `W` as soon as `N > Δmax·β_C·α`.

Putting numbers to it (`d = 5`, the top of the shipped `core_peering_degree: 3..=5`, `serde/core.rs` L54; Poisson tail for the last column):

| Parameters | `N` | `μ` | range | honest mean / connection / window | P(honest peer flagged spammy per window) |
|---|---|---|---|---|---|
| shipped: `Δmax=1, β_C=1, α=1.03, F_C=1, W=10`, coeff `1` | any `≥2` | 1 | `[1, 10]` | 8.2 | ≈ 0.21 |
| spec: `Δmax=3, β_C=3, α=1.03, F_C=1, W=30`, coeff `3` | 4 | 3 | `[9, 90]` | 74 | ≈ 0.03 |
| spec | 8 | 2 | `[6, 60]` | 74 | ≈ 0.95 |
| spec | ≥ 10 | 1 | `[3, 30]` | 74 | ≈ 1.00 |

With the shipped parameters the mean sits just under the cap and ordinary burstiness (release rounds batch up to `Δmax` rounds of processed messages, `release_delayer.rs` L144-L156; `MissedTickBehavior::Skip` lets a window absorb a late tick) pushes an honest peer over it in roughly one window out of five. With the spec's parameters the range is unusable for any realistic `N`. This is consistent with the history: enforcement was removed in `398d81243` ("fix: bypass Blend when proposing first block of a new session", 2026-03-19) with the note "Re-enable this once we have fixed Blend observation window range values", and the three integration tests were `#[ignore]`d in the same commit.

Consequences today, with the reactions commented out (`handler/mod.rs` L195-L201, `mod.rs` L1254-L1276): the monitor still runs on every core connection and emits `SpammyPeer` / `UnhealthyPeer` / `HealthyPeer`, of which only `HealthyPeer` is acted on (`mod.rs` L1277-L1279, L796-L808). There is no per-peer message-rate limit on core connections at all (#60 LB-003), and `Unhealthy` never triggers the spec's "open an additional connection" reaction (`swarm.rs` L450-L453 is dead: `Event::UnhealthyPeer` is never generated because `handle_unhealthy_connection` is `dead_code`, `mod.rs` L776-L779).

Ruled out while deriving this:
- `maximal_delay_rounds`, `blending_ops_per_message`, `normalization_constant`, `membership_size`, `rounds_per_observation_window` are wired to the right settings (`tokio_provider.rs` L79-L92, `deployment.rs` L41-L52, L89-L91, L116-L117, L136), so this is not a plumbing bug.
- The first interval tick fires immediately (`tokio::time::interval`), and the monitor uses it only to install the range (`conn_maintenance.rs` L90-L99), so the first evaluated window is a full one; there is no off-by-one in the counting.
- Epoch transitions do not double the per-connection rate: old-epoch traffic goes over old-epoch connections and new-epoch traffic over new ones (`mod.rs` L871-L877, L910-L921). What transitions do add is a fresh, empty window on every new connection while the network is still reconnecting, which yields a burst of `Unhealthy` verdicts, not `Spammy` ones. I could not identify a transition-specific mechanism beyond the false-positive rate above; the commit message's "epoch transition issues" are consistent with LB-002 turning false positives into a growing blocklist across epochs.

**Exploit scenario**

Not an attack by itself. The impact is that the protection cannot be enabled: if the reactions at `handler/mod.rs` L198 and `mod.rs` L1259-L1262 were restored today, each honest neighbour of a node running the shipped parameters would be disconnected with probability ≈ 0.21 per 10-round window and then blocked for the rest of the process (LB-002). A node with 5 peers would lose its first peer within a couple of windows and, since replacement dials exclude the blocklist (`swarm.rs` L241), would exhaust the membership within an epoch or two, partitioning itself from the blend network. While the reactions stay off, a member node (stake required) can send an arbitrary number of well-formed messages per window to any neighbour; the only per-message defences are dedup, signature and PoQ checks, so every such message costs the receiver a signature verification and, if the PoQ is valid or reused, a Groth16 verification.

**Recommendation**
- *Short term*: replace the estimator with one that models what the monitor counts. The expected number of distinct wire messages per window is `M_W = W · F_C · α · β_C` (all already available as `rounds_per_observation_window`, `message_frequency_per_round`, `normalization_constant`, `num_blend_layers`; `data_replication_factor` multiplies `β_C` if replication is used). A peer may legitimately send us every one of them, so the maximum must be at least `M_W`; use `max = ⌈c_max · M_W⌉` with a burst factor `c_max ≈ 2` and `min = ⌊M_W / c_min⌋` with `c_min ≈ 4` (shipped parameters: `M_W = 10.3`, range `[2, 21]`; spec parameters: `M_W = 93`, range `[23, 186]`). Neither bound depends on `N`, which also removes item 2 (frozen `membership_size`) from the picture. Then re-enable enforcement on the behaviour side rather than in the handler: in `on_connection_handler_event` handle `SpammyPeer` with `if self.update_state_for_negotiated_peer((peer_id, connection_id), NegotiatedPeerState::Spammy(SpamReason::TooManyMessages)).is_some() { self.close_connection((peer_id, connection_id)); }` and `UnhealthyPeer` with the existing `handle_unhealthy_connection`. Because `update_state_for_negotiated_peer` returns `None` for a connection id that is no longer the peer's current one (`mod.rs` L760-L770), verdicts from old-epoch connections are ignored automatically, which the handler-side `close_substreams()` cannot do. Un-ignore the three tests in `tests/connection_maintenance.rs` and the two in `services/blend/.../tests/network_maintenance.rs`; they need no change (S-002). Add a unit test for `calculate_expected_message_range` with the shipped values.
- *Long term*: fix the spec: §Global Parameters L480-L481 should bound the per-connection count by the network-wide generation rate, not by `μ`, and should state whether the minimum coefficient is `3` or `Δmax`. Feed the provider from the epoch stream (logos-blockchain#1533) only if an `N`-dependent term survives the spec fix. Consider counting only messages that pass `is_message_processed` dedup for the *unhealthy* bound while keeping the raw count for the *spammy* bound, so a peer cannot look healthy by replaying.

**References**: `blend-protocol.md` L455-L481, L523-L530, L905-L936 (`logos-co/logos-lips` @ `e2e9e0fc`); node commit `398d81243`; #60 LB-003, LB-004; logos-blockchain#1533.

### LB-002 · A peer blocked as spammy is never unblocked for the life of the process

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `services/blend/src/core/backends/libp2p/swarm.rs:L420-L428` (`handle_disconnected_peer`), `L241`; `services/blend/src/core/backends/libp2p/behaviour.rs:L13`, `L60` |
| Status | Open |

**Description**

Every disconnection of a peer in `Spammy` state adds it to a `libp2p::allow_block_list` behaviour:

```rust
// swarm.rs
422  if let NegotiatedPeerState::Spammy(reason) = peer_state {
424      self.swarm.behaviour_mut().blocked_peers.block_peer(peer_id);
```

Nothing ever calls `unblock_peer`; the only other reference is the dial filter at L241. `StartNewEpoch` (L628-L642) resets dial bookkeeping but not the blocklist. So a block lasts until the process restarts, across epochs and membership changes. The spec asks for a blacklist "and its selection must be avoided" (L527) but says nothing about its lifetime; the SDP membership, however, is re-derived every epoch and a blocked member remains a legitimate member.

Today the blocklist is fed only by verdicts that are hard to trigger by accident (`DuplicateMessage`, `InvalidHeaderSignature`, `UndeserializableMessage`, `InvalidProofOfQuota`; `mod.rs` L1039-L1044, L984-L992). Even so, an honest peer that hits one of them once through a bug or a version skew is gone until one side restarts. Once the monitor's `TooManyMessages` verdict is re-enabled, this is what turns LB-001's per-window false-positive rate into a monotonically growing blocklist.

**Exploit scenario**

A peer that can make a victim classify it as spammy once is blocked by that victim permanently; if it is one of the few well-connected members, the victim's reachable membership shrinks for good. No third party can force the classification on an honest peer, so the impact is reliability rather than security, hence Low.

**Recommendation**
- *Short term*: clear the blocklist, or expire entries, on `StartNewEpoch`; at minimum unblock peers that are in the new membership. Keep a per-peer strike counter so a repeat offender is blocked for longer.
- *Long term*: make the blocklist a bounded, time-decaying structure owned by the swarm and covered by a test that a blocked peer is dialable again in the next epoch.

**References**: `blend-protocol.md` L527; #60 LB-001 (edge peers are, conversely, never blocked).

## 5. Suggestions (non-security)

### S-001 · The `u64` products in `tokio_provider.rs` cannot wrap under any plausible configuration; the interval construction panics first

| | |
|---|---|
| Target | `services/blend/src/core/backends/libp2p/tokio_provider.rs:L40-L41, L53-L55`; `nodes/node/binary/src/config/blend/deployment.rs:L43-L51` |

`mu` is at most `⌈Δmax·β_C·α⌉` (for `N = 1`) and saturates at `u64::MAX` through the `as u64` cast. `mu * minimum_messages_coefficient` and `mu * rounds_per_observation_window` wrap only if the product exceeds `2^64`, which needs e.g. `α ≥ 10^18` or `Δmax ≥ 10^17`. `rounds_per_observation_window * round_duration_seconds` (L54) is `10·Δmax·slot_seconds`; with `slot_duration: 1s` it needs `Δmax ≥ 1.8·10^18`, at which point `10 * Δmax` in `deployment.rs` L44 has already wrapped and, if it wrapped to `0`, panicked at `NonZeroU64::new(..).unwrap()` L51 during config load. Between those two thresholds a wrapped period is either huge (`tokio::time::interval` then panics on `Instant + Duration` overflow when the first connection is established) or, only for exact multiples of `2^64`, zero (`interval` panics on a zero period). None of this is reachable by a deployment that intends to run. Use `checked_mul` and reject at config load anyway, since the current failure mode is a panic in the swarm task rather than a config error.

### S-002 · The ignored tests still describe the intended behaviour and can be un-ignored with the LB-001 change

| | |
|---|---|
| Target | `blend/network/src/core/with_core/behaviour/tests/connection_maintenance.rs:L19-L200`; `services/blend/src/core/backends/libp2p/tests/network_maintenance.rs:L18, L97` |

`detect_spammy_peer` expects `PeerDisconnected(_, Spammy(TooManyMessages))`, an empty `negotiated_peers`, the peer's cache entries removed on `ConnectionClosed`, and the listener-side close — all of which the behaviour-side reaction in LB-001 produces (`close_connection` → handler `CloseSubstreams` → swarm `ConnectionClosed` → `handle_disconnected_peer`). `detect_unhealthy_peer` and `restore_healthy_peer` exercise the event dedup in `handle_unhealthy_connection` / `handle_healthy_connection` (`mod.rs` L780-L808), which is unchanged. The tests inject a fixed range through `IntervalProviderBuilder` (`tests/utils.rs` L55-L68), so they do not cover `calculate_expected_message_range`; the service-level test helper explicitly never calls it (`services/blend/.../tests/utils.rs` L283). A unit test on the provider is the missing piece.

### S-003 · Bound `outbound_msgs` at the handler independently of the monitor

| | |
|---|---|
| Target | `blend/network/src/core/with_core/behaviour/handler/mod.rs:L32, L336-L340` |

Concrete change for #60 LB-003: in `on_behaviour_event`, before `self.outbound_msgs.push_back(msg)`, check `self.outbound_msgs.len() >= MAX_PENDING_OUTBOUND` and, if so, call `close_substreams()` and push `ToBehaviour::IOError(io::Error::new(WriteZero, "peer not draining"))` so the behaviour treats it like any other stalled connection and the swarm re-dials. `MAX_PENDING_OUTBOUND` can be derived from the same `M_W` as LB-001 (e.g. `2·M_W`, ≈ 21 messages ≈ 400 KB with the shipped parameters); a constant in the low hundreds is also fine. Dropping the oldest message instead of closing would silently break relaying for that peer, so closing is preferable.

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
