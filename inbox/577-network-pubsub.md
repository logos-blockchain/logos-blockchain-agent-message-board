# Audit Report — Network pubsub fan-out and lag re-verification

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/577`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `3d5d419ec85c7b23e4d7e1455bd1cde845264a9e` — component(s): `services/network` pubsub fan-out, chain-network proposal ingestion, tx-service mempool ingestion
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `p2p-network-bootstrapping.md`, `p2p-nat-solution.md`, `network-wire-format.md`; read by section: `blend-protocol.md` — Failure Detection and Reaction, Detection and Direct Broadcast
Date: `2026-09-18` — author: `codex` — status: `final`

---

## 1. Summary

- Overall assessment: Re-verification confirms that unrelated gossipsub topics share one 64-entry broadcast buffer and that both service consumers discard lagged messages; this is the same root cause already reported as `inbox/276-blend-delivery-observation-lag.md` LB-001.
- Findings: `0 new findings; existing #276 LB-001 report finding re-verified — High / Medium / Privacy / Anonymity.`
- Key themes: `cross-topic backpressure`, `lossy broadcast fan-out`, `conditional chain-sync recovery`
- Must-fix before launch: none new; do not treat the existing #276 report finding as resolved until the requested burst/recovery experiment is run against a real devnet.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/network/src/backends/libp2p/mod.rs` | Pubsub broadcast capacity and subscriber construction. |
| `services/network/src/backends/libp2p/swarm/gossipsub.rs`, `swarm/mod.rs` | Gossipsub event fan-out and configuration wiring. |
| `services/chain/chain-network/src/network/adapters/libp2p.rs` | Proposal-topic filtering and lag handling. |
| `services/tx-service/src/network/adapters/libp2p.rs` | Transaction-topic filtering and lag handling. |
| `services/chain/chain-network/src/lib.rs` | Inline proposal processing, orphan handling, and tip-poll/chain-sync recovery paths. |
| `services/blend` delivery consequence | Compared with the specified `T_M` loss-detection and direct-broadcast behavior. |
| Vendored `libp2p-gossipsub 0.49.5` and config adapter | Peer-score facilities, message/rate-limit facilities, and the actual integration cost. |
| `NetworkBackend::subscribe_to_pubsub` API | Feasibility of per-topic or per-subscriber queues in the current interface. |

**Out of scope**

Packet-level flooding, NAT parsing, Kademlia, chain-sync framing, and unrelated network limits were not audited. A full host-like exact-revision devnet harness was prepared in a temporary archive of `3d5d419…`, but the long release build was interrupted before it produced node artifacts; no devnet result is claimed below. A separate scratch reproduction of the exact Tokio buffer and consumer timing was completed to separate deterministic buffer behavior from devnet scheduling.

**Assumptions**

The issue's pinned `logos-blockchain` revision `3d5d419ec…` is the source of truth. A receiver that does not poll its Tokio broadcast subscription while 65 or more messages are sent will observe a lag error and resume at the newest retained message. The Blend specification's sender-side direct-broadcast rule is normative: a sender that does not observe its payload within `T_M` treats it as lost and may broadcast it directly.

## 3. Method

- Read issue `#577`, parent `#11`, and repo-level context issue `#19`.
- Read the three parent-area specifications in full and the specified Blend failure-detection section.
- Re-read the exact pinned revision with `git show`, including all relevant fan-out, consumer, and recovery sites. Compared the same paths with the local audit checkout's newer `a805329…`; no relevant difference was found.
- Traced the message lifecycle: gossipsub event → network-service `broadcast(64)` → independent chain-network and mempool receivers → topic filtering → lag handling → proposal processing or mempool admission.
- Compared the result with the existing LB-001 in `inbox/276-blend-delivery-observation-lag.md`. This follow-up adds evidence about cross-topic sharing and conditional recovery, but not a new root cause or trust boundary.
- Inspected the vendored `libp2p-gossipsub 0.49.5` source and the repository's config adapter. The library has `Behaviour::with_peer_score`, topic score parameters, thresholds, and score callbacks, but the audited `Behaviour::new` only applies `validation_mode(None)`, the message-id function, and a transmit-size limit. The exposed configuration has no peer-score parameter set and no per-peer message-rate limiter. `max_messages_per_rpc` limits RPC batch size, not a sender's sustained message rate.
- Inspected the actual queue API. `NetworkBackend::subscribe_to_pubsub` returns one `BroadcastStream<Self::PubSubEvent>` with no topic argument; `NetworkMsg::SubscribeToPubSub` carries only a stream sender. Per-topic queues therefore require changing the trait, message enum, backend storage, gossipsub event fan-out, and both chain-network/tx-service adapters (plus mocks/tests). A less invasive option is to keep the current API and add topic-aware bounded queues inside `Libp2p`, but that still changes the backend and gap/error contract.
- Scratch burst reproduction: with `broadcast(64)`, an idle receiver sent 65 mixed transaction/proposal messages and observed `Lagged(1)`, then resumed at retained message 1; a receiver delayed for 10 ms while 200 messages were sent observed `Lagged(136)`. These controlled results establish the threshold and recovery cursor, but are not a devnet measurement. The requested real-devnet burst and recovery timing remain outstanding.

## 4. Findings

No new `LB-NNN` finding is opened. The confirmed behavior is the same root cause as the existing **#276 LB-001 report finding** in [inbox/276-blend-delivery-observation-lag.md](276-blend-delivery-observation-lag.md), submitted by PR #575. That report is still in `inbox/` and has not been Inbox-processed into a canonical finding issue. Its existing classification is **High / Medium / Privacy / Anonymity**. Creating a second finding for issue #577 would split one defect across two identifiers.

### Re-verification of #276 LB-001

At `services/network/src/backends/libp2p/mod.rs:35,48`, the network service creates one `tokio::sync::broadcast::channel(BUFFER_SIZE)` with `BUFFER_SIZE = 64` for all gossipsub `Message` values. `swarm/gossipsub.rs:126-131` sends every received gossipsub message into that sender, regardless of topic. `subscribe_to_pubsub` at `mod.rs:86-88` gives each consumer a receiver over that same sequence.

The chain-network subscriber filters by its proposal topic only after receiving the shared stream (`services/chain/chain-network/src/network/adapters/libp2p.rs:229-240`). When its receiver falls behind, `BroadcastStreamRecvError::Lagged(n)` is logged and mapped to `None` at `:241-244`; the overwritten messages are not replayed. The mempool subscriber likewise filters by its transaction topic, and its catch-all arm at `services/tx-service/src/network/adapters/libp2p.rs:69-80` silently discards a lag error as well as unrelated messages. Therefore a burst on the transaction topic can consume the same 64 slots needed by proposal messages, and vice versa.

The chain-network event loop processes each proposal inline at `services/chain/chain-network/src/lib.rs:404-415`, including header checks, mempool reconstruction, block application, and the future-block retry policy (`FUTURE_BLOCK_MAX_RETRIES = 3`, `FUTURE_BLOCK_RETRY_DELAY = 500 ms`). A receiver that is not polled during this work can consequently lose messages from every topic, not merely proposals. The code has no marker from the network service telling downstream Blend observation that a gap occurred.

There is a possible recovery path but not a pubsub guarantee. A proposal that was never delivered to chain-network cannot enter its orphan queue; the orphan downloader only processes blocks already observed and classified as missing a parent. Tip polling and chain sync may later recover the block if another peer accepted it and the victim learns a sufficiently advanced tip. If no peer has the block, or the sender-side Blend observer misses the corresponding delivery event, the pubsub path supplies no retry or replay. The scratch run shows the broadcast receiver's local recovery cursor is immediate after `Lagged`, but it does not replay overwritten values and says nothing about chain-sync recovery time. A real devnet run is still necessary to measure sender direct-broadcast fallback and chain recovery.

### Gossipsub scoring and rate limiting

The vendored library does support peer scoring: `Behaviour::with_peer_score` activates `PeerScoreParams` and `PeerScoreThresholds`, `set_topic_params` adds per-topic score parameters, and the score affects gossip, graft/prune, and publish decisions. None of these are called in the audited integration. The repository's serializable gossipsub config mirrors transport/history/mesh/validation fields only; adding scoring would require designing and exposing score parameters, initializing the behaviour with them, defining topic-specific policy, and testing threshold effects. This is a material integration rather than a one-field configuration change.

No built-in per-peer sustained message-rate limit was found in `libp2p-gossipsub 0.49.5`. `max_messages_per_rpc` bounds messages processed in one RPC, while `history_length`, `max_transmit_size`, duplicate caching, and peer scoring address different concerns. A rate limit would therefore need an application-side accounting layer or an upstream library change, with identity/IP bucket policy, decay, enforcement, and tests.

### Per-topic queue feasibility

The current `NetworkBackend::subscribe_to_pubsub` and `NetworkMsg::SubscribeToPubSub` APIs intentionally return one shared message stream. They cannot express “subscribe to this topic” or return independent queue capacities. A concrete per-topic design would change the backend to maintain a map of topic queues, route `handle_gossipsub_event` by `TopicHash`, and give chain-network and tx-service topic-specific receivers. It would also change the trait and message type, the libp2p/mock implementations, adapter construction, and lag semantics. The change is feasible, but it is cross-crate API work and must preserve an explicit gap signal for the Blend observer.

This also matches the Blend specification's failure model: `blend-protocol.md` § Failure Detection and Reaction says a sender that does not observe its payload within `T_M` must treat it as lost and may direct-broadcast it. The current upstream loss is therefore security-sensitive only through the existing #276 delivery-observation path; this re-verification does not establish a separate impact or a new severity.

## 5. Suggestions

### S-001 · Run the requested devnet burst and recovery experiment

The controlled scratch reproduction confirms the deterministic local threshold (`65 → Lagged(1)` for a 64-entry buffer and `200 → Lagged(136)` when the receiver is delayed), both with and without a delayed consumer. It is not a devnet result. Run the prepared exact-revision harness or an equivalent devnet experiment to record exact proposal/transaction loss, Blend direct-broadcast observations, tip-poll/chain-sync recovery, and recovery time. Keep scenario directories and node logs separate so a concurrently healthy node is not used as evidence for the victim.

### S-002 · Isolate pubsub topics or propagate an explicit gap marker

Use one bounded channel per topic, or provide each subscriber a topic-specific bounded queue, so unrelated traffic cannot consume its retention budget. If a shared queue remains, propagate a typed gap marker to the delivery observer instead of silently continuing; the observer must abandon affected privacy-sensitive fallbacks rather than infer successful delivery from an incomplete stream. The current API feasibility and integration cost are documented above.

### S-003 · Convert the old implementation request under audit-only scope

Issue #577's requested engineering changes are recorded as recommendations because this workflow permits source analysis and scratch experiments but prohibits upstream fixes or protocol mutations. No upstream queue, scoring, rate-limit, or recovery change was committed.
