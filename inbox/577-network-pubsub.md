# Audit Report — Network pubsub fan-out and lag re-verification

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/577`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `3d5d419ec85c7b23e4d7e1455bd1cde845264a9e` — component(s): `services/network` pubsub fan-out, chain-network proposal ingestion, tx-service mempool ingestion
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `p2p-network-bootstrapping.md`, `p2p-nat-solution.md`, `network-wire-format.md`; read by section: `blend-protocol.md` — Failure Detection and Reaction, Detection and Direct Broadcast
Date: `2026-09-18` — author: `codex` — status: `final`

---

## 1. Summary

- Overall assessment: Static re-verification confirms that unrelated gossipsub topics share one 64-entry broadcast buffer and that both service consumers discard lagged messages; this is the same root cause already reported as `inbox/276-blend-delivery-observation-lag.md` LB-001.
- Findings: `0` critical · `0` high · `0` medium · `0` low · `0` informational
- Key themes: `cross-topic backpressure`, `lossy broadcast fan-out`, `conditional chain-sync recovery`
- Must-fix before launch: none new; do not treat the existing #276 finding as resolved until the requested burst/recovery experiment is run.

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

**Out of scope**

No devnet, peer-flood, or packet-level experiment was run in this iteration. The internals of vendored gossipsub peer scoring and Tokio's broadcast implementation were treated as specified dependencies, except for the documented `Lagged` behavior used by the callers. NAT parsing, Kademlia, chain-sync framing, and unrelated network limits were not audited.

**Assumptions**

The issue's pinned `logos-blockchain` revision `3d5d419ec…` is the source of truth. A receiver that does not poll its Tokio broadcast subscription while 65 or more messages are sent will observe a lag error and resume at the newest retained message. The Blend specification's sender-side direct-broadcast rule is normative: a sender that does not observe its payload within `T_M` treats it as lost and may broadcast it directly.

## 3. Method

- Read issue `#577`, parent `#11`, and repo-level context issue `#19`.
- Read the three parent-area specifications in full and the specified Blend failure-detection section.
- Re-read the exact pinned revision with `git show`, including all relevant fan-out, consumer, and recovery sites. Compared the same paths with the local audit checkout's newer `a805329…`; no relevant difference was found.
- Traced the message lifecycle: gossipsub event → network-service `broadcast(64)` → independent chain-network and mempool receivers → topic filtering → lag handling → proposal processing or mempool admission.
- Compared the result with the existing LB-001 in `inbox/276-blend-delivery-observation-lag.md`. This follow-up adds evidence about cross-topic sharing and conditional recovery, but not a new root cause or trust boundary.
- Automated tooling: none. Dynamic testing: none; the requested 65–200 message devnet burst and recovery timing remain outstanding.

## 4. Findings

No new `LB-NNN` finding is opened. The confirmed behavior is the same root cause as LB-001 in [inbox/276-blend-delivery-observation-lag.md](276-blend-delivery-observation-lag.md), submitted by PR #575 and still awaiting the requested dynamic validation. Creating a second finding for issue #577 would split one defect across two identifiers.

### Re-verification of #276 LB-001

At `services/network/src/backends/libp2p/mod.rs:35,48`, the network service creates one `tokio::sync::broadcast::channel(BUFFER_SIZE)` with `BUFFER_SIZE = 64` for all gossipsub `Message` values. `swarm/gossipsub.rs:126-131` sends every received gossipsub message into that sender, regardless of topic. `subscribe_to_pubsub` at `mod.rs:86-88` gives each consumer a receiver over that same sequence.

The chain-network subscriber filters by its proposal topic only after receiving the shared stream (`services/chain/chain-network/src/network/adapters/libp2p.rs:229-240`). When its receiver falls behind, `BroadcastStreamRecvError::Lagged(n)` is logged and mapped to `None` at `:241-244`; the overwritten messages are not replayed. The mempool subscriber likewise filters by its transaction topic, and its catch-all arm at `services/tx-service/src/network/adapters/libp2p.rs:69-80` silently discards a lag error as well as unrelated messages. Therefore a burst on the transaction topic can consume the same 64 slots needed by proposal messages, and vice versa.

The chain-network event loop processes each proposal inline at `services/chain/chain-network/src/lib.rs:404-415`, including header checks, mempool reconstruction, block application, and the future-block retry policy (`FUTURE_BLOCK_MAX_RETRIES = 3`, `FUTURE_BLOCK_RETRY_DELAY = 500 ms`). A receiver that is not polled during this work can consequently lose messages from every topic, not merely proposals. The code has no marker from the network service telling downstream Blend observation that a gap occurred.

There is a possible recovery path but not a pubsub guarantee. A proposal that was never delivered to chain-network cannot enter its orphan queue; the orphan downloader only processes blocks already observed and classified as missing a parent. Tip polling and chain sync may later recover the block if another peer accepted it and the victim learns a sufficiently advanced tip. If no peer has the block, or the sender-side Blend observer misses the corresponding delivery event, the pubsub path supplies no retry or replay. The requested devnet run is necessary to measure the actual burst threshold, whether the sender's direct-broadcast fallback fires, and how often chain sync repairs the loss.

This also matches the Blend specification's failure model: `blend-protocol.md` § Failure Detection and Reaction says a sender that does not observe its payload within `T_M` must treat it as lost and may direct-broadcast it. The current upstream loss is therefore security-sensitive only through the existing #276 delivery-observation path; this re-verification does not establish a separate impact or a new severity.

## 5. Suggestions

### S-001 · Run the requested burst and recovery experiment

On a devnet, publish 65–200 small messages on one topic while the victim's chain-network loop is busy. Record the first reliable `Lagged` threshold, exact proposal/transaction loss, Blend direct-broadcast observations, tip-poll/chain-sync recovery, and recovery time. Keep scenario directories and node logs separate so a concurrently healthy node is not used as evidence for the victim.

### S-002 · Isolate pubsub topics or propagate an explicit gap marker

Use one bounded channel per topic, or provide each subscriber a topic-specific bounded queue, so unrelated traffic cannot consume its retention budget. If a shared queue remains, propagate a typed gap marker to the delivery observer instead of silently continuing; the observer must abandon affected privacy-sensitive fallbacks rather than infer successful delivery from an incomplete stream.

