# Audit Report — Block validation ordering: cheap checks first, dedup before work, no relay before validation

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/43`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `services/chain/chain-network, services/chain/chain-service, core/src/block, ledger, consensus/cryptarchia-engine, libp2p, services/network`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md, overview-cryptoeconomics.md, cryptarchia-v1-bootstr-sync.md` (in full); `cryptarchia-v1-protocol.md` §Constants, §Block Chain (Block ID … Fork Pruning); `bedrock-v1.1-block-construction.md` §High-level Flow, §Block Proposal Reconstruction, §Block Proposal Validation, §Block Execution
Date: `2026-09-11` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: the per-block check order inside the chain service is mostly right (dedup, wallclock, parent, slot ordering and header signature all precede any Groth16 work), but the network ingress around it is not: every gossip message on the block topic is forwarded to mesh peers before it is even decoded, and a self-signed proposal for a far-future slot stalls the receiving node's whole chain-network loop for 1.5 s, which the relay turns into a network-wide stall.
- Findings: `0` critical · `1` high · `1` medium · `2` low · `1` informational
- Key themes: "relay before validation", "inline retry sleeps on the ingress event loop", "verdicts recorded against a block ID for failures that only condemn a copy", "uncle proofs verified before the block's own proof"
- Must-fix before launch: LB-001 (future-slot stall), LB-002 (gossipsub forwards unvalidated messages, no validation reporting, no peer scoring)

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/chain/chain-network/src/lib.rs` | proposal ingress: dedup gate, header check, reconstruction, apply, future-block retry, rejected-cache updates |
| `services/chain/chain-network/src/network/adapters/libp2p.rs` | proposal stream (decode, lag handling), chainsync block decode |
| `services/chain/chain-network/src/sync/orphan_handler.rs`, `sync/rejected_blocks.rs` | rejected-block cache semantics and bounds |
| `services/chain/chain-service/src/lib.rs`, `src/service/mod.rs`, `src/uncle.rs`, `src/api.rs` | `try_apply_block_with_state_retention` check order, uncle rules, error mapping |
| `ledger/src/lib.rs`, `ledger/src/cryptarchia/mod.rs` | `prepare_update`, slot ordering, epoch-state synthesis, PoL verification order |
| `consensus/cryptarchia-engine/src/lib.rs` | `apply_header` parent/slot checks |
| `core/src/block/mod.rs`, `core/src/header/mod.rs` | decode-time bounds, `verify_header_alone`, `Block::reconstruct` |
| `libp2p/src/behaviour/mod.rs`, `libp2p/src/config/gossipsub.rs`, `services/network/src/backends/libp2p/swarm/gossipsub.rs` | gossipsub validation mode, forwarding, message id, transmit size |
| `nodes/node/standalone-node-config.yaml`, `nodes/node/binary/src/config/network/serde/gossipsub.rs` | shipped gossipsub defaults |

**Out of scope**

- Phase transitions, restart consistency, uncle tracking bounds and notifier back-pressure (parent #4's own questions).
- The chainsync *provider* side (`sync/block_provider.rs`) and IBD peer selection.
- Correctness of the Groth16 verifier, Poseidon2, Ed25519 (`lb-groth16`, `rust-rapidsnark`, `ed25519-dalek`), `libp2p-gossipsub` 0.49.5 internals beyond the forwarding rule quoted below, `rpds`, `lru`, `tokio`.
- Blend-layer validation of proposals before they are handed to gossipsub.

**Assumptions**

- Specs at the commit above are the reference; where they disagree with the code the finding says which side I believe is wrong.
- Repo-level facts from issue #19 verified at this commit and relevant here: `[profile.release]` in `Cargo.toml:11-14` sets `lto = "fat"`, `strip = true` and does not set `overflow-checks`; the arithmetic on the paths reviewed here uses `checked_*`/`saturating_*`/`strict_*` so no wrapping site is reported. No gossipsub peer scoring and no connection or rate limits exist anywhere in `libp2p/src` or `services/network/src`.
- Attacker model for the DoS findings: a single unprivileged peer connected to the gossipsub mesh, no stake, no valid proof of leadership.

## 3. Method

- Manual review of the in-scope paths, working through issue `#43` (parent `#4`, context `#19`). Every checklist item was traced from the network socket to the ledger commit.
- Spec conformance against `cryptarchia-v1-protocol.md` §Block Header Validation and §Chain Maintenance, and `bedrock-v1.1-block-construction.md` §Block Proposal Validation (the section that fixes the check order for proposals) and §Reference Resolution (what a rejection may record).
- Confirmed the gossipsub forwarding rule in the pinned dependency source (`libp2p-gossipsub` 0.49.5, `src/behaviour.rs:1881-1889`).
- Automated tooling run: none.
- Dynamic testing: none.

**Check order as implemented (gossip path)**, for reference in the findings:

1. `chain-network/src/lib.rs:589-600` `should_process_block`: `slot <= lib_slot` → ignore; ledger state exists → already applied (dedup keyed on acceptance).
2. `:608` `verify_header_alone`: slot ≠ genesis, Ed25519 signature under the header's own `leader_key`.
3. `:620` `reconstruct_block_from_proposal`: one mempool prefix lookup per reference, then `Block::reconstruct` (`core/src/block/mod.rs:215-249`): signature again, `Σ storage_size ≤ 2 MiB`, `body_root`.
4. `:714` `apply_block_with_future_block_retry` → chain-service `try_apply_block_with_state_retention` (`chain-service/src/lib.rs:425-501`): already applied → future slot → uncles (ancestry, then per uncle: signature, PoL Groth16) → ledger `prepare_update` (parent known, `slot > parent.slot`, epoch-state synthesis, block's own PoL Groth16, transactions with batched ZK) → engine `apply_header` (parent known, slot) → commit.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Future-slot proposals stall the chain-network event loop for 1.5 s each | Denial of Service | High | Low | Open |
| LB-002 | Block-topic messages are forwarded to mesh peers before any validation | Denial of Service | Medium | Low | Open |
| LB-003 | Spec deviation: uncle proofs of leadership are verified before the block's own | Denial of Service | Low | Medium | Open |
| LB-004 | Spec deviation: copy-level and transient failures are recorded against the block ID | Data Validation | Low | Medium | Open |
| LB-005 | Spec deviation: reconstruction precedes uncle validation | Data Validation | Informational | High | Open |

### LB-001 · Future-slot proposals stall the chain-network event loop for 1.5 s each

| | |
|---|---|
| Severity | High |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `services/chain/chain-network/src/lib.rs:78-79`, `:404-415`, `:729-755`, `:950-990` (`retry_future_block_apply_with_delay`); `services/chain/chain-service/src/lib.rs:442-448` |
| Status | Open |

**Description**

When the chain service answers `FutureBlock`, chain-network retries the apply three times with a 500 ms `sleep` between attempts, and it does so *inline* in the body of the `select!` arm that consumes the proposal stream:

```rust
// chain-network/src/lib.rs:78-79
const FUTURE_BLOCK_MAX_RETRIES: usize = 3;
const FUTURE_BLOCK_RETRY_DELAY: Duration = Duration::from_millis(500);

// :407-415  (inside `loop { tokio::select! { ... } }`)
Some(proposal) = incoming_proposals.next() => {
    self.note_received_proposal(&proposal);
    self.handle_incoming_proposal(proposal, orphan_downloader.as_mut().get_mut(), &relays).await;
}

// :962-984
for attempt in 0..=max_retries {
    match apply_block().await {
        Ok(()) => return Ok(()),
        Err(Error::Cryptarchia(ApiError::FutureBlock { .. })) if attempt < max_retries => {
            ...
            sleep(retry_delay).await;
        }
```

While that future is pending, no other arm of the loop is polled: no other proposals, no orphan-download progress (`:433`), no chainsync events are forwarded to chain-service (`:417-422`), no polled tips are enqueued. Everything a peer needs from this node's chain-network service waits.

The check that produces `FutureBlock` is the second thing the chain service does (`chain-service/src/lib.rs:442-448`), before any Groth16 work, so the attacker's proposal never has to carry a valid proof of leadership. What it *does* have to pass before the sleep is cheap for the attacker:

- `should_process_block` (`:845-868`): `block_slot > lib_slot` holds trivially for a future slot; no ledger state exists for a fresh ID.
- `verify_header_alone` (`core/src/block/mod.rs:337-351`) verifies the signature under `header.leader_proof().leader_key()`, a key *inside the header the attacker wrote*, so a self-signed header passes. The proof bytes are never looked at here.
- `reconstruct_block_from_proposal` (`:1079-1098`) with zero references does no mempool lookup; `body_root` over an empty uncle list and an empty transaction list is a constant the attacker computes once.

Each such message therefore costs the attacker one Ed25519 signature and ~370 bytes, and costs every node that processes it 1.5 s of chain-network wall-clock. The proposal stream is a `broadcast` channel of capacity 64 (`services/network/src/backends/libp2p/mod.rs:35,48`); once it lags, messages are dropped, not delayed (`adapters/libp2p.rs:241-244` logs `lagged messages` and returns `None`). Genuine proposals arriving during the stall are among those dropped. Because LB-002 forwards the attacker's messages network-wide, every node in the mesh is stalled by the same stream.

**Exploit scenario**

A peer publishes, on the block topic, a fresh `Proposal` every ~1 s whose header has `slot = current_slot + 10^6`, a random parent, an empty reference list, an empty uncle list, the matching constant `body_root`, any 128-byte `proof`, its own Ed25519 public key as `leader_key`, and a signature under that key. Every node receives it (directly or via LB-002), passes it through steps 1-3 above, sends it to chain-service, gets `FutureBlock`, and sleeps 3 × 500 ms. At 1 message/s the chain-network loop of every node is asleep ~100% of the time; at ~100 messages/s the 64-slot ingress buffer overflows and genuine proposals are discarded. Block propagation across the network degrades to whatever fits between stalls; no node crashes, no state is corrupted, and the attack stops the moment the flood stops. Bandwidth needed: tens of KB/s from one peer.

**Recommendation**

- *Short term*: never `sleep` inside the ingress loop. Reject a block whose slot is more than a small tolerance (one or two slots) ahead of the local clock outright, before reconstruction, in `handle_incoming_proposal`; park blocks within the tolerance in a bounded, slot-keyed map and re-inject them from the slot-tick arm. Drop the inline retry.
- *Long term*: make the wallclock rule (`cryptarchia-v1-protocol.md` §Block Header Validation rule 6) part of `verify_header_alone`'s caller with an explicit tolerance parameter so that it is applied once, first, and identically on the gossip, orphan-download and sync paths; add a test that a far-future proposal is discarded without a chain-service round trip.

**References**: `cryptarchia-v1-protocol.md` §Block Header Validation rule 6; `bedrock-v1.1-block-construction.md` §Block Proposal Validation ("checked in the order given so that the cheapest checks discard a malformed or unauthenticated proposal first").

### LB-002 · Block-topic messages are forwarded to mesh peers before any validation

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `libp2p/src/behaviour/mod.rs:22`, `:74-78`; `libp2p/src/config/gossipsub.rs:118-120`; `services/network/src/backends/libp2p/swarm/gossipsub.rs:126-132`; `nodes/node/standalone-node-config.yaml:29`; `nodes/node/binary/src/config/network/serde/gossipsub.rs:66` |
| Status | Open |

**Description**

The gossipsub behaviour is built with `ValidationMode::None` and a 16 MiB `max_transmit_size` (`libp2p/src/behaviour/mod.rs:74-78`, `DATA_LIMIT` at `:22`). Application-level validation (`validate_messages`) is only enabled if the operator sets it (`libp2p/src/config/gossipsub.rs:118-120`); the shipped config sets it to `false` (`standalone-node-config.yaml:29`) and the binary's default is `gossipsub::Config::default().validate_messages()`, also `false` (`serde/gossipsub.rs:66`). Nothing in the workspace calls `report_message_validation_result`, and no peer-scoring parameters are configured.

With `validate_messages` off, the pinned `libp2p-gossipsub` 0.49.5 forwards every received message to the mesh immediately after the duplicate check, in the same call that emits it to the application (`behaviour.rs:1881-1889`):

```rust
// forward the message to mesh peers, if no validation is required
if !self.config.validate_messages() {
    self.forward_msg(&msg_id, raw_message, Some(propagation_source), HashSet::new());
```

The node's swarm handler only relays the event to a broadcast channel (`swarm/gossipsub.rs:126-132`); decoding (`Proposal::decode_all`, `adapters/libp2p.rs:232-239`) and every check in §3 happen after the bytes have already left for the mesh. So the checklist item "blocks are not relayed before they are validated" does not hold: the node relays before it has even established that the bytes are a proposal.

Two consequences:

- Any bytes up to 16 MiB published on the block topic by one peer are re-sent by every node to its `mesh_n` peers, so a single sender obtains network-wide bandwidth amplification without producing a valid header. (A well-formed proposal is at most ~18 KB: 297 + 64 + 1 + 4·361 + 2 + 1024·16 bytes.)
- The verdict a node reaches on a proposal cannot be fed back into gossipsub, so a peer that floods invalid proposals (LB-001, LB-003) is neither pruned from the mesh nor scored down, and its messages keep being amplified.

The design intent is that proposals reach gossipsub through Blend exit nodes (`services/blend/src/core/dispatcher/libp2p.rs:63-78`), which cannot sign messages without deanonymising the leader; `ValidationMode::None` and `MessageAuthenticity::Author` are therefore deliberate. That does not require forwarding before validation: gossipsub's `validate_messages` mode holds the message until the application reports `Accept`/`Reject`/`Ignore`, and `Reject` feeds peer scoring.

**Exploit scenario**

A peer with one mesh connection publishes 16 MiB of random bytes on the block topic at 1 message/s. Each node that receives it forwards it to ~`mesh_n` peers before `decode_all` fails, so the whole mesh carries ~16 MiB · mesh_n per second per node from a 16 MiB/s source. Nodes with constrained links fall behind on genuine proposals; the origin peer is never penalised. The same amplification applies to the cheap-to-mint invalid proposals of LB-001 and LB-003.

**Recommendation**

- *Short term*: set `validate_messages: true` by default and in the shipped config; make chain-network call `report_message_validation_result` with `Accept` once a proposal has passed the header-alone checks and reconstruction, `Reject` when the header itself is at fault (bad version, genesis slot, invalid signature under its own key, invalid proof of leadership, far-future slot), and `Ignore` for copy-level and mempool-dependent outcomes (LB-004). Lower `max_transmit_size` to the largest valid proposal plus margin.
- *Long term*: enable gossipsub peer scoring with an invalid-message penalty on the block topic, and add a network test that an undecodable message is not forwarded.

**References**: issue #43 checklist item 3; `bedrock-v1.1-block-construction.md` §Block Proposal Validation; `libp2p-gossipsub` 0.49.5 `src/behaviour.rs:1881-1889`.

### LB-003 · Spec deviation: uncle proofs of leadership are verified before the block's own

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | `services/chain/chain-service/src/lib.rs:450-478`; `services/chain/chain-service/src/uncle.rs:27-77`, `:128-142` |
| Status | Open |

**Description**

`try_apply_block_with_state_retention` verifies every carried uncle, including its Groth16 proof of leadership, before it applies the header to the ledger, which is where the block's *own* proof is verified:

```rust
// chain-service/src/lib.rs:450-478
// A block is valid only if every uncle it carries is valid.
self.verify_uncles(&block)?;
...
let update = self.ledger.prepare_update::<_, _, MainnetGasProfile>(id, parent, slot, &leader_proof, ...)
```

`verify_uncles` (`uncle.rs:64-76`) runs `uncle.verify()` (Ed25519) and then `verify_uncle_pol` (`:128-142`, a full `verify_proof_of_leadership` including epoch-state synthesis and a Groth16 verification) for each of up to `MAX_UNCLES = 4` entries. The test helper in the same file records the order explicitly: "uncles are verified before the block's own proof" (`uncle.rs:464-465`).

The specification orders these the other way. `cryptarchia-v1-protocol.md` §Block Header Validation lists the leader's own proof as step 9 and the uncle rules as step 10; `bedrock-v1.1-block-construction.md` §Block Proposal Validation puts "Header Validation … rules 1 and 5 through 9" in step 2 and "Uncle Validation" in step 3, "so that the cheapest checks discard a malformed or unauthenticated proposal first". I believe the spec side is right: the block's own proof is one verification and binds the proposer to the slot; the uncle proofs are up to four verifications that only matter once the block itself is authenticated.

**Exploit scenario**

Fork blocks are public, so an attacker holds genuine signed uncle headers. It publishes a proposal (LB-001 preconditions, but with `slot ≤ current_slot`, a parent on the local chain and `uncle_headers` filled with four genuine first-fork headers whose parents lie within the 360-slot window) and a garbage own proof. Each receiving node performs four *successful* Groth16 verifications plus four epoch-state syntheses before the fifth, failing, verification rejects the block: five times the cost of rejecting the same block with the spec's order, and the amplification of LB-002 applies. Cost to the attacker: one Ed25519 signature and one `body_root` hash per message.

**Recommendation**

- *Short term*: in `try_apply_block_with_state_retention`, verify the block's own proof of leadership against the parent state (`LedgerState::verify_proof_of_leadership` already exists for this purpose, `ledger/src/lib.rs:360-377`) before `verify_uncles`; keep the cheap uncle structural checks (`NotStrictlyOlder`, ancestry, window) where they are.
- *Long term*: batch the block's PoL with the uncle PoLs and the transaction proofs into the existing deferred verification (`verify_batch_proofs`, `ledger/src/update.rs:31`) so a single failure aborts before the batch is run, and add a test asserting the ledger's proof is checked before any uncle proof.

**References**: `cryptarchia-v1-protocol.md` §Block Header Validation steps 9-10; `bedrock-v1.1-block-construction.md` §Block Proposal Validation steps 2-3.

### LB-004 · Spec deviation: copy-level and transient failures are recorded against the block ID

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Data Validation |
| Target | `services/chain/chain-network/src/lib.rs:602-617`, `:683-691`, `:926-936`; `services/chain/chain-service/src/api.rs:337-345`; `services/chain/chain-network/src/sync/orphan_handler.rs:144-170`, `:233-247`; `services/chain/chain-network/src/sync/rejected_blocks.rs:31-50` |
| Status | Open |

**Description**

Three kinds of outcome are written into the rejected-block LRU (`RejectedBlocks`, capacity `max_rejected_cache_size`, default 1000) keyed by `block_id`, although the specification says they must not be:

1. A header signature failure. `chain-network/src/lib.rs:608-617` calls `orphan_downloader.insert_rejected_block(block_id)` when `verify_header_alone` fails, and the comment says "A bad signature is a property of the proposal itself — identical at every node — so it is final". `bedrock-v1.1-block-construction.md` §Block Proposal Validation says the opposite: "a frame that does not decode (step 1), a bad `signature` (step 2), an invalid uncle entry (step 3) and a failure to reconstruct (step 4) each discard the copy **without** recording a verdict against `block_id`", because the signature lies outside the 297 header bytes the ID is computed from, so one flipped bit produces a rejected copy with the genuine block's ID. I believe the spec is right and the comment is wrong.
2. Every other apply error. `handle_proposal_processing_error` (`:683-691`) records any error that is not `ParentMissing`/`FutureBlock`/`AlreadyApplied`/`CommsFailure` (`is_recoverable_apply_error`, `:926-936`). `api.rs:337-345` folds *all* remaining chain-service errors into `ApiError::Unexpected`, including `Error::Storage` from a failed `store_block_data` (`service/mod.rs:817-827`) and `Error::Mempool`. A transient local storage failure therefore condemns a genuine block ID.
3. Inherited rejection. `enqueue_orphan` (`orphan_handler.rs:158-170`) and `dequeue_next_orphan` (`:233-247`) refuse a block whose *parent* is in the cache and insert the block itself, so a wrongly recorded ID propagates down every descendant that arrives orphaned.

The cache is consulted only by the orphan pipeline (enqueue/dequeue), not by the gossip fast path, so a poisoned block that arrives while its parent is already known is still applied; the damage is confined to catch-up.

**Exploit scenario**

An attacker watching the block topic re-publishes each genuine proposal with one signature bit flipped, as fast as it can. The copy has the same `block_id` and a different gossipsub message ID (the ID is a hash of the full bytes, `libp2p/src/behaviour/gossipsub/mod.rs:7-11`), so it is forwarded. Any node that receives the tampered copy first and does not yet hold the parent records the genuine `block_id` as rejected; when the genuine copy arrives it is applied only if the parent is present, and otherwise `enqueue_orphan` refuses it and every orphaned descendant after it, until a tip poll enqueues a tip not yet in the cache. Nodes briefly disconnected or slow to receive the parent lag behind for as long as the attacker keeps racing. No consensus safety impact; delayed synchronisation for a subset of nodes.

**Recommendation**

- *Short term*: in `handle_incoming_proposal`, treat a `verify_header_alone` failure like a reconstruction failure (log, count, return; no cache insert). In `handle_proposal_processing_error`, insert into the cache only for errors that the header bytes alone imply (`Error::Ledger(InvalidProof | InvalidSlot)`, `Error::Consensus(InvalidSlot)`, `Error::InvalidUncle`, `Error::InvalidBlock`); make `api.rs` carry the distinction instead of collapsing to `Unexpected`.
- *Long term*: give the rejected cache a small TTL and do not propagate rejection to children whose own bytes have not been judged; add a test that a proposal with a corrupted signature leaves the cache untouched.

**References**: `bedrock-v1.1-block-construction.md` §Reference Resolution ("It **must not** record that outcome as a verdict on `block_id`"), §Binding of the reference list ("Duplicate suppression on `block_id` is therefore keyed on acceptance, never on receipt"), §Block Proposal Validation closing paragraph.

### LB-005 · Spec deviation: reconstruction precedes uncle validation

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Data Validation |
| Target | `services/chain/chain-network/src/lib.rs:619-645`; `services/chain/chain-service/src/lib.rs:450-451` |
| Status | Open |

**Description**

`bedrock-v1.1-block-construction.md` §Block Proposal Validation orders uncle validation (step 3) before reconstruction (step 4). The node reconstructs first (`chain-network/src/lib.rs:620`, one mempool prefix lookup per reference, then `body_root`), sends the reconstructed block to chain-service, and only then validates uncles (`chain-service/src/lib.rs:451`). The spec's stated reason for its order is that both steps need no mempool access for step 3; the reason the node's order is *safe* is that by the time uncles are judged their bytes are bound to the header by the `body_root` check in `Block::reconstruct` (`core/src/block/mod.rs:235-247`), which is exactly what lets the node treat an invalid uncle as a verdict on the block (LB-004 item 2 lists `InvalidUncle` as legitimately final under this order). Under the spec's order, an uncle failure must *not* be recorded, since the entries are still unauthenticated at step 3. The two orders are therefore each internally consistent, but the code and the spec disagree, and the code's order is the cheaper one for a defender (a prefix lookup costs less than a Groth16 verification). I believe the spec should adopt the code's order; see S-002.

**Exploit scenario**

None. Recorded so that the ordering difference is not mistaken for a bug when the spec and code are next compared.

**Recommendation**

- *Short term*: none in code.
- *Long term*: align the spec (S-002) and add a comment at `chain-network/src/lib.rs:619` stating that the `body_root` check is what makes the later uncle verdict final.

**References**: `bedrock-v1.1-block-construction.md` §Block Proposal Validation steps 3-4.

### Checked and ruled out

- **Decode-time bounds.** `Proposal.references` is `UpperBoundedVec<_, 1024>` and `uncle_headers` is `UpperBoundedVec<_, 4>` (`core/src/block/mod.rs:70`, `core/src/block/uncle.rs:11`), so counts are rejected while decoding, before allocation proportional to them. `Block` transactions are `BoundedVec<_, 0, 1024>` and `Σ storage_size ≤ 2 MiB` is enforced in `validate_total_transactions_size` (`core/src/block/mod.rs:251-274`) on every construction/decode path; `bedrock_version` is checked at decode (`core/src/header/mod.rs:134-137`). Synced blocks go through `TryFrom<Bytes> for Block` → `into_verified` (`core/src/block/mod.rs:366-378`): signature, size and `body_root` are checked at decode.
- **Cheap header checks before proofs.** Parent known (`ledger/src/lib.rs:198-201`, and `uncle.rs:36-42` when uncles are present), `slot > parent.slot` (`ledger/src/cryptarchia/mod.rs:264-269`), wallclock (`chain-service/src/lib.rs:443`), signature (`chain-network/src/lib.rs:608`, `core/src/block/mod.rs:240`) all precede `try_apply_proof` (`ledger/src/cryptarchia/mod.rs:493-513`). The engine's `apply_header` (`consensus/cryptarchia-engine/src/lib.rs:232-268`) repeats the parent and slot checks after the ledger; both are cheap map lookups. Rule 8 (height above LIB) is implemented as the `slot ≤ lib_slot` gate in chain-network (`:910-912`) plus fork pruning in the engine; no separate ancestry check exists in chain-service, which matches the spec's "assuming that T prunes all forks diverged deeper than B_imm".
- **Duplicate detection.** Gossipsub deduplicates by Blake2b of the full message bytes (`libp2p/src/behaviour/gossipsub/mod.rs:7-11`, `duplicate_cache_time` 60 s); the node deduplicates by ledger-state presence, in chain-network (`should_process_block`, `:859-861`) and again in chain-service (`AlreadyApplied`, `chain-service/src/lib.rs:438-440`). Both are keyed on acceptance, as the spec requires, and bounded by ledger-state pruning. Tampered copies with the same `block_id` bypass the gossipsub cache by design and are stopped by the ledger-state check once the block is accepted.
- **Signature before mempool scan.** Holds (`:602-617` before `:620`), and `resolve_reference` takes at most two candidates per prefix (`:1111-1136`).
- **State mutation on error.** `process_block` applies to a clone and swaps on success (`service/mod.rs:804-837`); `Ledger` and the engine use persistent maps (`HashTrieMapSync`, `rpds`), so the per-block clone is cheap and a rejected block leaves no state.
- **Overflow sites** on the reviewed paths use `checked_add`/`saturating_*`/`strict_*` (`consensus/cryptarchia-engine/src/lib.rs:248-251`, `ledger/src/lib.rs:347-350`, `core/src/block/mod.rs:259`).
- **Uncle ancestry walk** (`uncle.rs:83-121`) is bounded by the 360-slot window and touches only in-memory branches.

## 5. Suggestions (non-security)

### S-001 · The specification does not state a forwarding rule for received proposals

| | |
|---|---|
| Target | `libp2p/src/behaviour/mod.rs:74-78` (code side); `bedrock-v1.1-block-construction.md` §Block Proposal Validation (spec side) |

**Description**
Checklist item 3 ("blocks are not relayed before they are validated") has no counterpart in the specifications read: `cryptarchia-v1-bootstr-sync.md` only says a node listens for "new blocks relayed by its peers", and the block-construction spec describes dissemination through Blend without saying what a receiving node may re-send and when. LB-002 is therefore a finding against the checklist and against gossipsub's semantics, not against a written rule.

**Recommendation**
Add one rule to §Block Proposal Validation: a node re-sends a received proposal copy only after step 2 (header validation) has passed on that copy, and never a copy it has rejected. Raise upstream.

### S-002 · Spec order of uncle validation and reconstruction should be swapped

| | |
|---|---|
| Target | `bedrock-v1.1-block-construction.md` §Block Proposal Validation steps 3-4 |

**Description**
Step 3 (uncle validation: up to four Groth16 verifications) precedes step 4 (reconstruction: a mempool prefix lookup per reference plus one hash). The cheaper step is the later one, and step 3 as written must discard its verdict because the entries are unauthenticated until step 4. Placing reconstruction first authenticates the uncle entries through `body_root`, lets a failure in the uncle rules condemn the block, and matches what the node does (LB-005).

**Recommendation**
Swap steps 3 and 4, move the `InvalidUncle` case from the "copy-level" list to the "condemns the block" list, and delete the sentence explaining why an uncle failure at step 3 cannot be recorded.

### S-003 · Inconsistent maximum block size across specs

| | |
|---|---|
| Target | `overview-cryptoeconomics.md` §Fee Markets and §Permanent Storage Fee Market ("1MiB"); `cryptarchia-v1-protocol.md` §Constants (`MAX_BLOCK_SIZE` = 2 MiB); `core/src/block/mod.rs:32` (2 MiB) |

**Description**
The overview states the block body limit as 1 MiB in two places; the protocol spec's constants table and the code say 2 MiB. The code follows the protocol spec.

**Recommendation**
Change the overview to cite `MAX_BLOCK_SIZE` from the protocol spec instead of restating a value. Raise upstream.

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
