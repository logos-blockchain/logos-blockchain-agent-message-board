# Audit Report — Mempool duplicate `Add`: callers, re-gossip effect, and the spawned adapter

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/277`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `54328b50eb47ab4b8fd8044f8db5fbcebed718c0` — component(s): `services/tx-service` (`tx/service.rs`, `network/adapters/libp2p.rs`, `backend/pool.rs`), `services/network/src/backends/libp2p/swarm/gossipsub.rs`, `libp2p/src/behaviour/gossipsub`, the six `MempoolMsg::Add` callers (`services/api`, `services/blend/src/core/dispatcher`, `services/chain/chain-network`, `services/chain/chain-leader`, `services/sdp`, `nodes/node/binary`)
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `mantle-transaction-encoding.md`, `bedrock-v1.1-block-construction.md`, `blend-protocol.md`; by section: `bedrock-v1.1-mantle-specification.md` (› Mantle Transaction Hash, › Validation)
Date: `2026-09-14` — author: `Claude Fable 5.1 (Claude Code)` — status: `final`

---

## 1. Summary

- Overall assessment: the `ExistingItem` arm of `handle_add_message` is reached by five callers with different intentions and the message cannot tell them apart; the re-gossip it spawns is a silent no-op for 60 s and a silent whole-network re-flood after that, and the adapter it constructs inside a detached task panics, and so exits the node, if the network service is gone.
- Findings: `0` critical · `0` high · `0` medium · `1` low · `1` informational
- Key themes: origin-blind duplicate handling, unobservable re-gossip, panic-to-exit from a detached task.
- Must-fix before launch: none. LB-001 is one more instance of the process-wide panic hook class (#491) and should be fixed with the others.

The core observation of this issue, that a duplicate `Add` re-announces and re-gossips, is already filed as #502 (`248-LB-007`) and is not restated as a finding. This report answers the five checklist items against the current head and adds the two things #502 does not cover: the adapter constructor's `expect` inside the spawned tasks, and the absence of any signal for what the re-gossip does.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/tx-service/src/tx/service.rs` | `handle_add_message` (L393-L437), `handle_add_success` (L497-L515), `notify_about_accepted_item` (L527-L534), `handle_network_item` (L548-L599), the `select!` loop (L303-L322) |
| `services/tx-service/src/network/adapters/libp2p.rs` | `new` (L35-L56), `send` (L83-L101) |
| `services/tx-service/src/backend/pool.rs` | `add_item` (L140-L174), `remove`/`retire`/`prune_removed_items` (L207-L224, L346-L383) |
| `services/tx-service/src/lib.rs` | `MempoolMsg` (L21-L55) |
| `services/network/src/backends/libp2p/swarm/gossipsub.rs` | `handle_pubsub_command`, `broadcast_and_retry` (L33-L124) |
| `libp2p/src/behaviour/gossipsub/mod.rs`, `libp2p/src/config/gossipsub.rs`, `nodes/node/standalone-node-config.yaml` | message id, duplicate cache, flood publish |
| `libp2p-gossipsub 0.49.5` (`behaviour.rs`) | `publish` L582-L756, `subscribe` L529-L551, receive-side duplicate check L1827-L1834 — read from the registry source, as the behaviour under audit depends on it |
| `MempoolMsg::Add` callers | `services/api/src/http/mempool.rs:L62-L85`; `services/blend/src/core/dispatcher/libp2p.rs:L153-L189`; `services/chain/chain-network/src/mempool/adapter.rs:L31-L46` and `lib.rs:L1001-L1063`, `bootstrap/ibd.rs:L65-L72`; `services/chain/chain-leader/src/mempool/adapter.rs:L60-L74` and `lib.rs:L831-L860`; `nodes/node/binary/src/generic_services/sdp/mempool.rs:L81-L95` and `services/sdp/src/lib.rs:L369-L387` |
| `nodes/node/binary/src/panic.rs`, `lib.rs:L228` | the process-wide panic hook |
| `overwatch` @ `ae887f41` | `OutboundRelay::send` (`services/relay/outbound.rs:L35-L40`), `SERVICE_RELAY_BUFFER_SIZE` (`services/mod.rs:L41`) |

**Out of scope**

Transaction validity and admission policy (#55, #316, #333), the pool's bounds and eviction, RocksDB persistence, the chain-network reorg logic itself (#138, #434), the Blend failure detector's lag policy (#276, #571), and gossipsub peer scoring, which is not configured (`grep` finds no `with_peer_score` in `libp2p/src`). `libp2p`, `tokio`, `overwatch`, `blake2` and `serde` are assumed correct; their sources were read only to establish behaviour, not to audit them.

**Assumptions**

The shipped configuration (`nodes/node/standalone-node-config.yaml`): `duplicate_cache_time = 60 s` (L26-L28), `flood_publish = true` (L40), gossipsub message id = Blake2b-256 of the data (`libp2p/src/behaviour/gossipsub/mod.rs:L7-L11`), API bound to `127.0.0.1:8080` with no authentication layer in `services/api` (L168-L172). Whoever can reach the HTTP API is trusted by the operator.

## 3. Method

- Manual review of the paths above, working through issue `#277` under parent `#9`, with the #248 report's LB-007 (#502) and the neighbouring follow-ups #276 and #138 open alongside.
- Spec conformance: `blend-protocol.md` › Processing step 2.3.2 and › Broadcasting ("A transaction extracted from a data message is not broadcast by this protocol. It is submitted to the mempool of the node that received it, and the mempool disseminates it as its own rules specify"); `bedrock-v1.1-block-construction.md` › Block Proposal Reconstruction steps 1-3 (a transaction is gossiped once when it enters a mempool). Neither spec says anything about resubmission, so nothing here is a spec deviation; the specs leave dissemination policy to the mempool.
- Tracker search before rating: the duplicate re-announce/re-gossip is #502; lag on the accepted-item channel is #276/#571; reorg re-broadcast is #138/#434; the panic hook is #491/#339 with per-site instances #259, #365, #401; no issue names the `NetworkAdapter::new` `expect`, and none covers observability of the re-gossip.
- Automated tooling: none. Dynamic testing: none. `services/tx-service` at this head differs from the `a805329f8` cited in the issue only by a test-target registration in `Cargo.toml`; the source is unchanged, so the issue's line numbers still hold. No test in `services/tx-service/tests` or `src` exercises the `ExistingItem` arm of `handle_add_message`.

**Checklist items, verified**

1. *Every caller that can reach `Add` with a key already held.* There are five, plus one non-caller the issue lists:

   | Caller | Site | Second `Add` of a held key means | Re-gossip wanted? |
   |---|---|---|---|
   | HTTP `add_tx` | `services/api/src/http/mempool.rs:L72-L79` | a user resubmitted | only if the first publish failed or fell out of peers' pools; harmless otherwise |
   | Blend exit path `submit_transaction` | `services/blend/src/core/dispatcher/libp2p.rs:L170-L176` | this node decapsulated the same payload twice, or received it by gossip first | no: the network already has it (item 2) |
   | Reorg reinsertion `add_transaction` | `chain-network/src/mempool/adapter.rs:L33-L40`, called from `lib.rs:L1052-L1060`, also during IBD (`bootstrap/ibd.rs:L65-L72`) | a reorged transaction is still pending, because `remove` retires the key (`pool.rs:L346-L353`) but `add_item` lets a retired key back in (`L147-L149`, `L158`), so a gossip copy arriving after inclusion re-pends it | no (#138, #434) |
   | Leader claim `post_tx` | `chain-leader/src/mempool/adapter.rs:L62-L68`, from `build_and_submit_claim_tx` (`lib.rs:L844-L860`) | the operator asked for the same claim twice | neutral |
   | SDP `post_tx` | `nodes/node/binary/src/generic_services/sdp/mempool.rs:L83-L89` | the intent tracker resubmitted an activity not yet applied (`services/sdp/src/lib.rs:L378-L382`, every `status_check_interval_in_tip_changes` tip changes, `intent.rs:L31-L34`) | yes: this is the one caller whose purpose is "re-gossip so leader nodes can pick it up", and only if its bytes are identical to the pooled copy (a rebuilt transaction with different fee inputs is a new key and takes the success arm) |
   | Block reconstruction | `chain-network/src/lib.rs:L1079+` | never calls `Add`; it reads through `GetTransactionsByPrefix` | not applicable |

   `MempoolMsg::Add` carries `payload`, `key` and `reply_channel` only (`tx-service/src/lib.rs:L25-L29`), so the arm cannot distinguish the rows. The SDP row is the only one the comment at `service.rs:L424-L425` describes correctly.

2. *Whether the re-gossip achieves anything.* `NetworkAdapter::send` issues `PubSubCommand::Broadcast` (`adapters/libp2p.rs:L88-L96`); the network service calls `gossipsub.publish` (`swarm/gossipsub.rs:L68`); the id is Blake2b-256 of the bytes alone (`libp2p/src/behaviour/gossipsub/mod.rs:L7-L11`), so a resubmitted transaction has the id of its first publication. Inside `duplicate_cache_time` the library refuses it (`behaviour.rs:L617-L625`, `PublishError::Duplicate`) and the network service logs that at `trace` (`swarm/gossipsub.rs:L111-L116`): the re-gossip does nothing. Outside the window the publish proceeds, and with `flood_publish = true` it goes to every connected peer on the topic, not only the mesh (`behaviour.rs:L643-L650`). Each receiving peer whose own 60 s cache has also expired accepts it as new (`L1827-L1834`), forwards it, and hands it to its mempool, where `handle_network_add_error` logs `ExistingItem` at `trace` (`service.rs:L584-L591`). So a duplicate `Add` more than 60 s after the previous publication re-floods the whole network with the transaction once; between 0 and 60 s it is dropped at the first hop. The cost to the caller is nil: admission checks size only (`L536-L546`), and the transaction is already pooled, so no fee is ever at stake. That is not a new attack surface, since the same caller can flood brand-new never-includable transactions at the same cost (#316, #333), but it is the second, silent, publish path that LB-002 is about. What it achieves honestly: recovery from a first publish that exhausted the `NoPeersSubscribedToTopic` retries (`swarm/gossipsub.rs:L84-L110`), or from peers that evicted the transaction by TTL; both are the SDP row's case.

3. *`NetworkAdapter::new` inside the spawned tasks.* Both `handle_add_success` (`L507-L510`) and the `ExistingItem` arm (`L427-L430`) construct the adapter inside the task. `new` sends `PubSubCommand::Subscribe` for the mempool topic (`adapters/libp2p.rs:L46-L51`) through the 16-slot service relay (`overwatch` `SERVICE_RELAY_BUFFER_SIZE = 16`, `services/mod.rs:L41`), then `send` issues the `Broadcast`: two relay messages and one swarm-loop wakeup per accepted or duplicated transaction where one would do. The subscribe itself is cheap at the swarm, since gossipsub returns `Ok(false)` as soon as the topic is in the mesh (`behaviour.rs:L535-L538`). The `expect("Network backend should be ready")` at `L51` is reachable: `OutboundRelay::send` fails only when the receiving service's channel is closed (`outbound.rs:L35-L40`), so the task panics exactly when the network service has stopped, and the node's panic hook turns that into `process::exit(1)`. That is LB-001.

4. *Whether the duplicate-path `notify_about_accepted_item` should be there.* Its only subscriber is the Blend dispatcher's `observe_transactions`; `SubscribeToAccepted` is sent from one place (`dispatcher/libp2p.rs:L200`). Any key already pending was announced when it was first added, by the success arm (`L413`) or the network path (`L571`), unless no subscriber existed at that moment (`receiver_count() > 0`, `L531`), which is start-up only. The duplicate announcement therefore never tells the failure detector anything new, and each one consumes a slot of the 64-slot channel (`L45`, `L265`) whose overflow ends the direct-broadcast fallback for the run (#276, #571). It should not be there; #502's short-term fix removes it.

5. *Whether #138's bound must account for this arm.* A reorged transaction's reinsertion publishes once whatever the pool's state: the success arm broadcasts (`L507-L510`), the `ExistingItem` arm re-gossips (`L427-L430`). The arm therefore does not double the count, but it removes the suppression that "already pooled, nothing to do" would give, so the bound is one publish attempt per reorged transaction per node per reorg, regardless of whether peers had already re-gossiped it back. Inside 60 s the second attempt is dropped at `publish`; a long IBD or repeated deep reorgs of the same blocks more than 60 s apart re-flood each time. Both attempts are invisible (LB-002), so the bound cannot be measured in production.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | A mempool `Add` after the network service has stopped panics inside a detached task, and the process-wide hook exits the node | Denial of Service | Low | High | Open |
| LB-002 | The duplicate re-gossip has no counter or log above `trace` in either of its outcomes, so re-flood volume cannot be seen | Auditing and Logging | Informational | Low | Open |

### LB-001 · A mempool `Add` after the network service has stopped panics inside a detached task, and the process-wide hook exits the node

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Denial of Service |
| Target | `services/tx-service/src/network/adapters/libp2p.rs:L46-L51` (`Libp2pAdapter::new`), `services/tx-service/src/tx/service.rs:L427-L430` and `L507-L510` (the two `spawn` sites); `nodes/node/binary/src/panic.rs:L10-L36`, `nodes/node/binary/src/lib.rs:L228` |
| Status | Open |

**Description**

Both publish paths of the mempool construct the network adapter inside a detached task:

```rust
// services/tx-service/src/tx/service.rs
427   spawn("logos/mempool/transaction-regossip", async move {
428       let adapter = NetworkAdapter::new(settings, network_relay).await;
429       adapter.send(item).await;
430   });
```

and the constructor unwraps the relay send:

```rust
// services/tx-service/src/network/adapters/libp2p.rs
46    network_relay
47        .send(NetworkMsg::Process(Command::PubSub(
48            PubSubCommand::Subscribe(settings.topic.clone()),
49        )))
50        .await
51        .expect("Network backend should be ready");
```

`OutboundRelay::send` returns `Err` only when the receiving service's inbound channel is closed (`overwatch/src/services/relay/outbound.rs:L35-L40`), so the `expect` fires exactly when the network service has stopped: during shutdown, if it stops before the mempool has drained the tasks it spawned, or whenever the network service exits on its own. In the same file, `send` handles the same failure by logging (`L88-L99`); only `new` panics. The panic is not confined to the task: the node installs a process-wide hook (`lib.rs:L228`) that logs and calls `std::process::exit(1)` (`panic.rs:L35`), so the task's panic ends the process, skipping whatever orderly teardown the other services were doing.

This is one more site of the class filed as #491 and #339; the per-site instances filed so far (#259, #365, #401) do not include it. It is listed here because item 3 of the issue asks for exactly this reachability, and because the `ExistingItem` arm doubles the number of tasks that can hit it.

**Exploit scenario**

Not attacker-triggerable from the network: a peer cannot stop the network service. The honest path is a shutdown race: the operator stops the node while an HTTP submission, a Blend decapsulation or a reorg reinsertion is in flight; the network service closes its relay; the spawned task wakes, panics at `L51`, and the process exits with status 1 before the remaining services finish stopping. The visible effect is an error-level "A panic occurred" log and a non-zero exit at every such shutdown, and a teardown cut short for the services still running.

**Recommendation**

- *Short term*: make `new` infallible in the same way `send` already is, by logging the failed `Subscribe` instead of unwrapping it, or return `Result` and drop the task on `Err`.
- *Long term*: construct the adapter once in `run` (it is already built there for `payload_stream`, `L240-L249`) and hand a clone to the tasks, so no task sends a `Subscribe` at all. That removes both the panic site and the second relay message per transaction (item 3). Add the general fix of #491 for the class.

**References**: #491, #339 (the hook); #259 (the same pattern on the chain-service storage relay); issue #277 item 3.

### LB-002 · The duplicate re-gossip has no counter or log above `trace` in either of its outcomes, so re-flood volume cannot be seen

| | |
|---|---|
| Severity | Informational |
| Difficulty | Low |
| Category | Auditing and Logging |
| Target | `services/tx-service/src/tx/service.rs:L423-L434` (no log, no metric), `services/tx-service/src/metrics.rs` (three counters: added, removed, pending); `services/network/src/backends/libp2p/swarm/gossipsub.rs:L111-L116` (`PublishError::Duplicate` at `trace`); `services/tx-service/src/tx/service.rs:L584-L591` (`ExistingItem` from gossip at `trace`) |
| Status | Open |

**Description**

The `ExistingItem` arm emits nothing when it spawns a re-gossip, and the two things that can then happen are each logged only at `trace`:

- inside `duplicate_cache_time` (60 s), the network service is refused by `publish` and writes "not publishing duplicate message to topic" at `trace` (`swarm/gossipsub.rs:L111-L116`);
- outside it, every node in the network receives the transaction again, forwards it, and its mempool writes "network item already exists in the mempool" at `trace` (`service.rs:L586-L591`).

The mempool's metrics (`metrics.rs`) count additions, removals and pending items; a re-gossip changes none of them. Consequently an operator cannot tell how often the arm fires, how much of the mempool topic's traffic is re-floods, or whether a reorg storm (#138, #434) or a resubmission loop is under way, and the #138 amplification bound has no production measurement to check against. The gossip-receive `ExistingItem` count is also the one signal that would show, network-wide, that some node is re-publishing.

**Exploit scenario**

None. This is observability: the re-gossip arm's honest use (item 1, SDP row) and its unwanted uses (Blend, reorg, resubmission loops) are indistinguishable in logs and metrics, so neither #502's fix nor #138's bound can be verified after deployment.

**Recommendation**

- *Short term*: a counter on the `ExistingItem` arm of `handle_add_message` (`mempool_transactions_regossiped_total`) and one on the gossip-side `ExistingItem` (`mempool_network_duplicates_total`), next to the three existing counters; raise the network service's `Duplicate` refusal to `debug`.
- *Long term*: once `Add` carries an origin (#502), label the counter by origin, so the SDP resubmissions and the Blend and reorg duplicates can be read separately.

**References**: #502, #138, #434; issue #277 items 2 and 5.

## 5. Suggestions (non-security)

### S-001 · Give `MempoolMsg::Add` an origin and decide the duplicate policy per caller

#502 recommends passing the origin; the table in Method item 1 is the input it needs. Five callers, three policies: re-gossip on duplicate for the SDP intent tracker and, if wanted, the HTTP API; return `Ok(())` and do nothing for Blend and reorg reinsertion; either for the leader claim. A `pub enum AddOrigin { Api, Blend, Reorg, Leader, Sdp }` on the message keeps the decision in `handle_add_message` and out of the callers.

### S-002 · Test the `ExistingItem` arm

No test in `services/tx-service` reaches it. Two assertions would pin #502's fix: a second `Add` of a pooled key produces no `accepted_items` message and no `Broadcast` command for a Blend/reorg origin, and exactly one `Broadcast` for an SDP origin.
