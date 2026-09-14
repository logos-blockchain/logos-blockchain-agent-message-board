# Audit Report — Rejected-block cache: the mempool-copy variant reproduced on two nodes, and the exits from the rejection cascade

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/184`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `services/chain/chain-network` (`lib.rs`, `sync/orphan_handler.rs`, `sync/rejected_blocks.rs`, `sync/tip_poll.rs`, `sync/config.rs`, `bootstrap/ibd.rs`), `services/chain/chain-service` (`api.rs`, `lib.rs`, `service/mod.rs`), `core/src/block/mod.rs`, `core/src/mantle/ops/transfer.rs`, `core/src/mantle/transactions/tx_list/signed_ops.rs`, `tests/testing_framework` (reproduction only)
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `cryptarchia-v1-bootstr-sync.md`, `fork-choice.md`; by section: `cryptarchia-v1-protocol.md` §Block Header Validation, §Chain Maintenance; `bedrock-v1.1-block-construction.md` §Block Proposal Reconstruction, §Reference Resolution, §Binding of the reference list, §Block Proposal Validation
Date: 2026-09-12 — author: `agent (Claude)` — status: `final`

---

## 1. Summary

- Overall assessment: issue #184 is confirmed end to end, on two live nodes. A node holding a proof-mangled copy of a transaction admits it (the transfer op's `preverify` never reads the proof), reconstructs the honest block with that copy, fails batch proof verification with `BatchZkpVerification(InvalidZkSignatures)`, records the genuine block ID as rejected, and refuses the next honest blocks as descendants (§4.4, run 3). The cache has no exit of its own — LRU eviction never frees the frontier and the tip poll is refused once the polled tip is cached — and the only recovery is accidental: a proposal that never receives a cache entry, whose orphan download streams the condemned range past the cache. That accident is common on a lossy Blend deployment (it ended the reproduction's cascade after 5 s) and absent on a gap-free one, where the node stays behind until restart. Two further gates found by the reproduction: a victim that wins a slot while holding the bad copy evicts it at block assembly, and a catch-up download that beats the proposal pre-empts the attack; neither protects a non-proposing node on a normally-paced network.
- Findings: 0 critical · 1 high · 0 medium · 0 low · 1 informational
- Key themes: "mempool admission does not establish proof validity, so a reconstructed block's proofs are this node's bytes, not the block's", "a rejection cascade keeps its own frontier fresh in the LRU", "the recovery path is the one place the cache is not consulted", "outage length is set by the network's delivery gaps, not by the cache"
- Must-fix before launch: LB-001 (the same defect as report #179 LB-001 and report #214 LB-001, now reproduced; one fix closes #113, #143 and #184).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/chain/chain-network/src/lib.rs` L404-L509, L571-L727, L775-L796, L845-L936 | event loop arms, proposal path, apply-error handling, tip-poll enqueue, `is_recoverable_apply_error` |
| `services/chain/chain-network/src/sync/orphan_handler.rs` L141-L251, L297-L312, L385-L470 | cache insertion, `enqueue_orphan`, `dequeue_next_orphan`, `cancel_active_download`, the stream arm |
| `services/chain/chain-network/src/sync/rejected_blocks.rs` (whole) | LRU semantics, touch-on-hit |
| `services/chain/chain-network/src/sync/tip_poll.rs` L40-L66 | when a poll fires and which tip it hands over |
| `services/chain/chain-network/src/sync/config.rs`; `nodes/node/standalone-node-config.yaml` L146; `nodes/node/binary/src/config/cryptarchia/serde/network.rs` L80; `tests/testing_framework/src/framework/local/provisioning.rs` L827 | capacity in every shipped configuration |
| `services/chain/chain-network/src/bootstrap/ibd.rs` L161-L221 | contrast: IBD never records |
| `services/chain/chain-service/src/api.rs` L33-L50, L317-L352; `chain-service/src/lib.rs` L91-L127, L443-L478; `service/mod.rs` L773-L837 | error taxonomy, the `Unexpected` collapse, where storage and proof failures arise |
| `core/src/block/mod.rs` L215-L285, L337-L364; `core/src/mantle/transactions/tx_list/signed_ops.rs` L105-L112, L222-L231, L360-L371; `core/src/mantle/ops/transfer.rs` L102-L115 | what `body_root` and `mantle_txhash` commit to; what `preverify` checks |
| `tests/testing_framework` (local deployer, `node/http_client.rs`, `workloads/transaction/workload.rs`) | used to build the reproduction harness in Appendix B |

**Out of scope**

The chainsync provider side and the streamed-copy variant (report #214 LB-001 covers it), mempool admission beyond the one `preverify` fact used here (report #179), gossipsub validation and scoring (#144), the correctness of `lb_zksign::batch_verify`, and the IBD liveness gap (#135). Third-party crates assumed correct: `lru` 0.18.2, `libp2p`, `tokio`, `rocksdb`. Findings of reports #179 (`inbox/113-mempool-admission-validation.md`) and #214 (`inbox/143-rejected-cache-insertion-sites.md`), both at this same target and spec commit, are cited, not repeated; the per-variant classification table is #214 §4.0.

**Assumptions**

The spec text at the `logos-lips` commit above is authoritative for what a rejection may establish about a block ID (§Block Proposal Validation, closing paragraph). The node is past IBD. The reproduction uses the testing framework's local deployer with the shipped defaults except the four consensus parameters listed in §4.4, and a node binary built at the target commit with `--features testing` (needed for the `/mempool` view route only).

## 3. Method

- Manual review of the in-scope paths against the four checklist items of `#184` (parent `#3`), building on report #179 LB-001 (the mempool-key fact) and report #214 §4.0 (the insertion-site and error-variant tables).
- Spec conformance against `bedrock-v1.1-block-construction.md` §Reference Resolution and §Block Proposal Validation (which failures may be recorded against `block_id`), `cryptarchia-v1-bootstr-sync.md` §Listening for New Blocks and §Downloading Blocks (an orphan may be abandoned and retried; a failed download returns), and `cryptarchia-v1-protocol.md` §Chain Maintenance (`on_block` ignores an invalid block and keeps no memory of it; the rejected cache is an implementation optimisation with no spec counterpart).
- Automated tooling: none.
- Dynamic testing: a two-node reproduction on the local testing-framework deployer (Appendix B), node binary `cargo build --release -p logos-blockchain-node --features testing` at the target commit (release profile with `lto=off`, `codegen-units=16` to shorten the build; no functional difference), nodes at `LOG_LEVEL=debug`. Results in §4.4.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | A proof-mangled mempool copy makes a node condemn the honest block it is reconstructed into, and the rejection cascades to every descendant until the next missed proposal or a restart (reproduced) | Consensus | High | Medium | Open |
| LB-002 | Streamed ancestors bypass the rejected cache, and that bypass is currently the only in-protocol recovery from the cascade | Denial of Service | Informational | — | Open |

### 4.1 Checklist item 1 — which apply errors on a *reconstructed* block are properties of the block

Report #214 §4.0 classifies every `chain_service::Error` variant by the bytes it reads, for both insertion sites. Two facts specific to the reconstructed-proposal path (S5, `lib.rs` L683-L691) are added here.

**The proofs of a reconstructed block are this node's bytes.** `reconstruct_block_from_proposal` (L1079-L1098) resolves each 16-byte reference to the single local mempool item with that prefix (L1111-L1136) and hands the resolved items to `Block::reconstruct` (`core/src/block/mod.rs` L215-L247), which checks `verify_header_alone`, the size bound and `body_root` (L276-L284). `body_root` commits to `merkle_root(tx.hash())` (L355-L364), and for the node's transaction type `hash()` is `tx_hasher` over `op_refs()` only (`signed_ops.rs` L222-L231): `op_proofs` are not hashed. So after a successful reconstruction every byte of the block is bound to `block_id` **except** the proof column, which is whatever this node's mempool stored.

**Admission never establishes that those proofs verify.** Both admission entry points build the pool item through `SignedOps::<Unverified>::preverify` — the HTTP route deserialises straight into `SignedOps<Preverified>` (`signed_ops.rs` L360-L371 runs `preverify()` inside `Deserialize`), and the gossip adapter decodes the same type (report #179 §"What admission actually checks"). `preverify` (L105-L112) calls each op's `into_preverified`, and for the ledger transfer that is `transfer.rs` L102-L115:

```rust
fn preverify(&self, _context: &Self::Context<'_>) -> Result<(), Self::Error> {
    let operation = self.operation();
    operation.inputs.preverify()?;   // non-empty
    operation.outputs.validate()?;   // well-formed
    Ok(())
}
```

The ZkSignature is not read; its verification is deferred to `verify` (L117-L140) and batched at block application (`chain-service/src/lib.rs` L461-L478 → `batch.rs` L47-L56). A copy whose 128-byte proof is any other 128 bytes is therefore admitted under the genuine transaction's key (`pool.rs` L147-L149, report #179 LB-001), and the reproduction in §4.4 admits one through the HTTP route with a one-bit flip.

**Consequence for the classification.** On the reconstructed path, `BatchZkpVerification` (and the multisig `VerificationError` variants, per #214) are verdicts on *this node's copy*, never on the block: another node holding the genuine proofs reconstructs the identical `block_id` and applies it. Every other error `prepare_update` can return reads committed bytes against the shared parent state and is a legitimate block-level verdict, with the two node-local exceptions #214 LB-003 lists (`Storage`, `AwaitingGenesisTime`). The code cannot make this distinction today because `apply_block` formats all of them into `ApiError::Unexpected(String)` (`api.rs` L338-L351) and `is_recoverable_apply_error` (`lib.rs` L926-L936) allowlists four typed variants and treats the string as terminal — #214 S-001. The spec's rule is that only failures "implied by the header bytes themselves" condemn `block_id`, and that a "failure to reconstruct" — which the mempool-dependence of step 6 extends to a failure *in what was reconstructed* — "may resolve at another node" (§Block Proposal Validation, closing paragraph; §Reference Resolution).

### LB-001 · A proof-mangled mempool copy makes a node condemn the honest block it is reconstructed into, and the rejection cascades to every descendant until the next missed proposal or a restart (reproduced)

| | |
|---|---|
| Severity | High |
| Difficulty | Medium |
| Category | Consensus |
| Target | `services/chain/chain-network/src/lib.rs:L683-L691` (S5, `handle_proposal_processing_error`), `:L926-L936` (`is_recoverable_apply_error`); `services/chain/chain-service/src/api.rs:L350`; `services/chain/chain-network/src/sync/orphan_handler.rs:L159-L170` (cascade); `core/src/mantle/ops/transfer.rs:L102-L115` (`preverify` skips the proof) |
| Status | Open |

**Description**

This is the finding of report #179 LB-001 seen from the cache side, and the reconstructed-proposal twin of report #214 LB-001; the rating is theirs and the mechanism is not restated beyond §4.1. What this report adds is the reproduction (§4.4) and the analysis of whether the node can ever leave the state (§4.2).

**Exploit scenario**

Report #179 LB-001 gives the attacker model (one funded note, direct connections; publish `T` to one node and `T'` to every other). The reproduction below plays the two-node core of it: node A receives `T'`, node B receives `T`, a block containing `T` is proposed, A condemns it and stalls until restarted.

**Recommendation**

- *Short term*: as #179 LB-001 (1) and #214 LB-001: on the reconstructed-proposal site, classify `BatchZkpVerification` and multisig `VerificationError` as copy-level — do not insert `block_id`, evict the resolved transactions whose proofs failed from the mempool (they are demonstrably bad copies), and re-request the block in full from a peer. The harness in Appendix B, with its assertions inverted (A must end up with the block on its chain without a restart), is the regression test for that change.
- *Long term*: #214 S-001 (typed `ApiError`, allowlist of block-level verdicts) and #179 S-001 (`body_root` over the full signed transaction), either of which makes this class impossible rather than handled.

**References**: `bedrock-v1.1-block-construction.md` §Reference Resolution, §Block Proposal Validation; `cryptarchia-v1-protocol.md` §Block Header Validation rule 4; report #179 LB-001; report #214 LB-001, §4.0, S-001; issue #186 (the mempool-side repro and fix prototype).

### 4.2 Checklist item 2 — can a node ever leave the cascade, and can an attacker close the exits

**How the cascade sustains itself.** After `B` is cached, every honest proposal `P_i` (child of `P_{i-1}`, with `P_0 = B`) reconstructs normally (its transactions are in the mempool), fails apply with `ParentMissing` because `P_{i-1}` was never applied, and is handed to `enqueue_orphan(P_i, Some(P_{i-1}))` (`lib.rs` L660-L672). `contains_block_or_parent` (`rejected_blocks.rs` L31-L40) finds `P_{i-1}` and the block is itself inserted (`orphan_handler.rs` L159-L170). Only the `Rejected` outcome inserts; `AlreadyDownloading` (L172-L177), `AlreadyInQueue` (L179-L182) and `QueueFull` (L199-L212) do not. So the cascade inserts exactly one entry per honest block, and the gate for `P_i` is `P_{i-1}`, never `B`.

**LRU eviction is not an exit.** Capacity is 1000 in every shipped configuration (`sync/config.rs` has no serde default; `standalone-node-config.yaml` L146, `serde/network.rs` L80, `provisioning.rs` L827). `LruCache::put` evicts the least recently used key; `get` promotes. The entry the next proposal is gated on is always the one inserted last and promoted by the check that inserted the newest, so the frontier is never the least recently used. `B` itself stops being promoted once its direct children stop arriving and is evicted after roughly a thousand honest blocks, but evicting `B` unblocks nothing: `P_{i-1}` still gates `P_i`. The cascade survives eviction indefinitely.

**The tip poll is refused.** `poll_peer_tips_if_behind` fires on the cadence when the local tip lags by `lag_threshold_blocks` (`tip_poll.rs` L40-L66) and selects the most advanced sampled tip. `enqueue_polled_tip` (`lib.rs` L775-L796) calls `enqueue_orphan(tip, None, ..)`, which is gated on the tip ID alone. A stuck node still receives every proposal by gossip and cascades each into the cache, so the polled tip is, with overwhelming likelihood, already cached and the enqueue is refused (`debug!` "tip poll: did not enqueue polled tip", L793). The poll can only succeed if the sampled tip reaches the victim before that block's proposal does.

**The exits, enumerated.**

| Exit | Mechanism | Works? | Can an attacker close it? |
|---|---|---|---|
| Restart | `RejectedBlocks` is in-memory only (`rejected_blocks.rs` L15-L24); after restart the orphan pipeline re-fetches from the LIB/tip and the download stream is not cache-checked | yes, reliably (reproduced in §4.4) | no; but the attacker re-enters with a new `T'` (#179 exploit scenario) |
| Cascade gap | any proposal that never gets a cache entry — missed delivery, a reconstruction failure (`lib.rs` L628-L641 deliberately does not record), or `QueueFull` — leaves its child's parent un-cached; the child is enqueued, the download starts from the LCA with the local tip (`orphan_handler.rs` L262-L268), and the streamed range, `B` included, is applied without consulting the cache (LB-002) | yes, when it occurs; the streamed copy of `B` carries the provider's proofs, which verify. Observed in run 3 (§4.4): a missed proposal 5 s after the condemnation ended the cascade at three entries | the attacker does not need to close it: under the spec's transaction-maturity assumption and healthy delivery every proposal reconstructs and cascades, so the gap does not occur on its own. The outage lasts until the next missed proposal |
| Tip-poll race | as above, entered when the polled tip is not yet cached | rarely; requires the poll response to beat the proposal | same |
| LRU eviction | see above | **no** | — |
| Cache flush | report #214 LB-002: 1000 unauthenticated pre-LIB headers evict everything | yes, but it is the attacker's tool, not the victim's | — |

**What the victim does meanwhile.** A stuck node that is also a leader keeps proposing on `B`'s parent. Its proposals reference its own mempool, and peers that hold the genuine transactions reconstruct and apply them as a fork of the honest chain. In §4.4 this is visible on node B: it applies A's blocks as a competing branch and, with two equal-stake nodes, its canonical tip switches between the two branches. On a real network the stuck node's branch is a persistent minority fork that the honest majority evaluates on every block, with the fork-choice cost that issue #193 measures.

### 4.3 Checklist item 3 — downloaded ancestors are not checked against the cache

Confirmed. The stream arm yields every block the provider sends (`orphan_handler.rs` L391-L417) with no cache lookup, and the event-loop arm that applies it (`lib.rs` L433-L473) checks only `should_process_block` (LIB and already-applied) before `apply_block_with_future_block_retry`. The cache is consulted at exactly two points, `enqueue_orphan` and `dequeue_next_orphan`, both of which look at the orphan and its parent only.

Is it intended? The type's own documentation says the cache exists so the orphan pipeline can "skip (known-invalid or older-than-LIB)" blocks (`rejected_blocks.rs` L9-L10) and the insertion helper says the pipeline "will drop it if it surfaces from the queue later" (`orphan_handler.rs` L141-L143); a streamed ancestor that is cached is neither skipped nor dropped, so the bypass reads as an omission, not a decision. It is nonetheless load-bearing today: it is the only reason the "cascade gap" and tip-poll exits in §4.2 work at all, because the stream re-applies the condemned `B` from a copy whose proofs are the provider's.

**Recommendation (LB-002).** Do not make the cache "consistent" now — checking streamed blocks against it would remove the only in-protocol recovery from LB-001 while the cache still holds copy-level verdicts. Make the bypass deliberate instead: a comment at the stream arm stating that the cache is a gossip-path optimisation and that a full block obtained by synchronisation is "validated on its merits" (`bedrock-v1.1-block-construction.md` §Reference Resolution, last paragraph), and a unit test that a cached ID streamed by a provider is still applied. Once the cache holds only header-implied verdicts (#214 S-001), a cache check on the stream arm becomes sound and saves one apply per re-streamed invalid block; revisit then.

### 4.4 Checklist item 4 — two-node reproduction

**Setup.** Two local nodes from the testing framework's manual cluster (Appendix B): node B is a normal leader; node A is the victim. Consensus parameters as in the existing `tip_poll_self_heal` test: 1 s slots, `security_param = 5`, `slot_activation_coeff = 1/2`, prolonged bootstrap period 0. One funded wallet account builds a transfer `T` (one input, one output, fee 794) and signs it; `T'` is the same `SignedOps` with bit 0 of byte 100 of the 128-byte compressed ZkSignature proof flipped; both preverify. `T'` is submitted to A and `T` to B over HTTP in the same instant, so each node admits its own copy before the other's gossip arrives (the later gossiped copy is a duplicate key and is dropped). The harness then finds the first block on B's canonical chain containing the hash, samples both nodes every 5 s for 90 s, counts the relevant log lines on A, restarts A, and checks whether the block reaches A's canonical chain. Node logs at `LOG_LEVEL=debug`. Three runs were needed; each of the first two failed to reach the cache for a reason that is itself a result.

**Run 1 — both nodes leaders: the victim's own leader evicts `T'` first.** B proposed block `1d3cd742` (height 4, containing `T`) at slot 13. In the *same* slot A's leader assembled its own block, and block assembly runs the deferred ZKP verification per transaction (`chain-leader/src/lib.rs` L675-L724): `T'` failed, stayed in `pending`, and was evicted from the mempool as "genuinely invalid" (L729-L738, logged at L752; A's log: `proposed block 458d3dbd with 0 transactions (1 removed)` at 06:22:58.119, `removed: 1 items from mempool`). When B's proposal reached A four seconds later, reconstruction failed with `UnresolvedReference` (`lib.rs` L628-L641, not recorded), B's next block arrived as an orphan with an *un-cached* parent, and the download streamed `1d3cd742` with B's proofs; A applied it. The cache was never touched by the block. Consequence: a node that wins a slot between admitting `T'` and receiving the honest block heals itself; the #184 victim is a node that does not propose in that window — every non-leader, and every leader at production `f`, where the window is many blocks long. (The run also showed a second, unrelated effect: with two equal-stake leaders and `k = 5`, A's own branch became immutable before B's did, and the two nodes never converged — A at height 116, B at 118, four minutes after a restart. That is the online fork-choice rule doing what §Online Fork Choice Rule says on a 50/50 two-node fork, not the cache.)

**Run 2 — A a non-leader (empty wallet key list): the tip poll fetches the block before its proposal arrives.** B applied `ecec9e90` (height 4) at slot 13; A's first sight of it was `Processing block from orphan downloader` at slot 20, streamed by a tip-poll catch-up (`tip poll: enqueued peer tip for catch-up`), with B's proofs, so it applied. The proposal, delivered through Blend later, was `AlreadyApplied` and ignored. At 1 s slots and `f = 1/2` the watchdog's default `lag_threshold_blocks = 3` is six seconds, shorter than Blend's proposal delivery delay, so *every* block reaches A by download first (17 of A's 29 applies came from the downloader). At production block intervals the Blend delay is a small fraction of one block and the proposal arrives first, so run 3 disables the tip poll on A to restore that ordering.

**Run 3 — A a non-leader, tip poll off on A: the cache path fires.** B applied `64f4e5e1` (height 4, containing `T`) at slot 13; the proposal reached A at slot 22 (06:36:41.809, Blend delay). A's log, in order:

```
06:36:41.826 ERROR chain::network  Error processing reconstructed block
             err=Unexpected Error: Failure while applying block: BatchZkpVerification(InvalidZkSignatures)
             block_id=64f4e5e1…                                       ← S5, lib.rs L683-L691
06:36:41.826 DEBUG chain::network::sync  inserted rejected block into cache block_id=64f4e5e1…
06:36:41.826 ERROR chain::service  Failed to process block: ParentMissing { parent: 64f4e5e1…, … }   ← child B5 (7af5c0e6)
06:36:41.826 DEBUG chain::network::sync  Orphan block (or its parent) is in the rejected cache, skipping enqueue
             block_id=7af5c0e6… parent_id=Some(64f4e5e1…)              ← S6, orphan_handler.rs L159-L170
06:36:41.826 DEBUG chain::network::sync  inserted rejected block into cache block_id=7af5c0e6…
06:36:43.786 … ParentMissing { parent: 7af5c0e6… }  → skipping enqueue → inserted 6f3022b5…   ← B6, cascade
06:36:46.790 DEBUG chain::network  Parent block missing … block_id=670bf4b8… parent=19e0d286…   ← B8; B7 (19e0d286) never arrived
06:36:46.790 DEBUG chain::network::sync  Orphan block enqueued for sync block_id=670bf4b8…       ← parent not cached: the gap
06:36:46.796 DEBUG chain::network  Processing block from orphan downloader: 64f4e5e1…
06:36:46.805 DEBUG chain::network  Applied block 64f4e5e1…                                      ← streamed copy, B's proofs, no cache check
06:36:46.810 … Applied 7af5c0e6…, 6f3022b5…, 19e0d286…, 670bf4b8…  (total_blocks_received=6)
```

Sampled state (A height / B height / block on A's chain): 0 s 3/4/no · 5 s 3/7/no · 10 s 9/10/**yes** · … · 85 s 29/30/yes. Final: A 30, B 32; the block is on both canonical chains. Log counts on A over the run: `inserted rejected block into cache` 3, `is in the rejected cache, skipping enqueue` 2, `tip poll: did not enqueue` 0 (poll disabled). After the restart A was, as expected, still caught up.

**What the runs establish.**

1. The mechanism of LB-001 is real at this commit: a one-bit change in a proof column that nothing hashes is admitted (§4.1), reconstructs the honest block, and condemns the honest block's ID on the reconstructing node, with the cascade immediately refusing the next two honest blocks.
2. The cascade *did* end without a restart, through exactly the mechanism §4.2 predicts: a proposal (B7) that never got a cache entry because it never arrived, whose child was enqueued and whose download re-applied the condemned range from the provider's copy. In this two-node Blend deployment that gap occurred after 5 s and 3 blocks; A missed 6 of the 32 proposals it should have received during the run (6 `ParentMissing` on the gossip path), so gaps are frequent here. The duration of the outage is therefore the time to the next delivery gap. It is not a property of the cache, which has no exit of its own, but of the network: on a deployment where every proposal is delivered and reconstructs, the state persists until restart. The "until restart" wording of the issue is the gap-free case; "until the next missed proposal" is the general one.
3. Two conditions gate the attack on this commit that neither report #179 nor #214 states: the victim must not assemble a block while holding `T'` (run 1), and its proposal must be the first copy of the block it sees (run 2 shows a catch-up download pre-empting it). Both hold for a non-proposing node on a normally-paced network.

**Ruled out by the runs.** The orphan download applies a cached ID without consulting the cache (LB-002) — seen at 06:36:46.796. Stale cache entries after recovery are harmless: `should_process_block` answers `AlreadyApplied` for them before any cache lookup.

### 4.5 Ruled out

- The proposal path never consults the cache before applying (`handle_incoming_proposal`, `lib.rs` L585-L646): a cached `block_id` does not stop the genuine proposal from being applied if it arrives and reconstructs. The cache bites only through the orphan pipeline, which is why the damage is a *descendant* cascade rather than a direct refusal.
- `cancel_active_download` (`orphan_handler.rs` L297-L308) does not insert; report #214 §4.1 covers what it does with the queue.
- IBD never inserts and its downloader's cache is inert (`ibd.rs` L161-L221; report #214 LB-004), so the cascade cannot begin during IBD.
- A leader applying its *own* block goes through `Message::ApplyBlockAndReconcileMempool` (`lib.rs` L826-L840), which reports the error to the leader and does not touch the cache; the leader's own proposal, when it comes back by gossip, goes through the same S5 path as any other.
- Mempool reconciliation after a successful apply (`lib.rs` L1030-L1060) swallows its errors; no successful apply is turned into a recorded rejection.

## 5. Suggestions (non-security)

### S-001 · A two-node regression test for the copy-level classification

The harness in Appendix B takes about two minutes and needs only the local deployer and a `--features testing` node binary. Inverted (assert that no `inserted rejected block into cache` line is written for the block and that A applies it from the proposal path, not a download), it is the acceptance test for the fix of LB-001 at the reconstructed-proposal site; a second variant that mangles the proof in a chainsync-served copy covers #214 LB-001. Both belong under `tests/src/tests/cryptarchia/`. Two things the harness had to do that a regression test must keep: start the victim with an empty `wallet.known_keys` so it never assembles a block, and disable its tip poll so the proposal is the first copy it sees (§4.4, runs 1 and 2).

### S-002 · Consecutive orphans re-stream the ancestors the previous download just applied

`OrphanInfo` captures `tip` and `lib` at enqueue time (`orphan_handler.rs` L214-L217) and the request is built from them (L262-L268). In run 3, B8 and B9 were enqueued 1 ms apart while A's tip was still `a3dd66da`; the download for B8 streamed and applied six blocks (`total_blocks_received=6`), then the download for B9 asked from the same stale tip and streamed seven, the first six of which were `AlreadyApplied` (06:36:46.828, six `Processing block from orphan downloader` lines in 1 ms). The parent-replacement rule (L184-L197) does not help because B8 was already downloading, not queued (L172-L177). This is the "chain of orphans triggers repeated re-fetching of the same parent" question in parent issue #3. Refresh `tip`/`lib` from chain-service when the request is built, or pass the last applied block of the previous download in `known_blocks`; cost is one extra query per download.

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

## Appendix B — Reproduction harness

The harness is a `logos-blockchain-tests` integration test at the target commit. It is a reproduction, not a regression test: its assertions pass when the bug is observed (§5 S-001 describes the inverted form).

Register it in `tests/Cargo.toml`:

```toml
[[test]]
name = "test_cryptarchia_rejected_cache_mempool_copy"
path = "src/tests/cryptarchia/rejected_cache_mempool_copy.rs"
```

Build the node and run (the `testing` feature is only needed for the `/mempool` view route; `LOG_LEVEL=debug` is needed for the `debug!` cache lines the harness counts; `E2E_KEEP_LOGS=1` keeps the scenario directory with both node logs):

```sh
cargo build --locked --release -p logos-blockchain-node --features testing
LOGOS_BLOCKCHAIN_NODE_BIN=$(pwd)/target/release/logos-blockchain-node \
LOG_LEVEL=debug E2E_KEEP_LOGS=1 E2E_TESTS_BASE_DIR_OVERRIDE=/tmp/lb-184 \
  cargo test --locked -p logos-blockchain-tests \
  --test test_cryptarchia_rejected_cache_mempool_copy -- --nocapture
```

`tests/src/tests/cryptarchia/rejected_cache_mempool_copy.rs`:

```rust
//! Reproduction harness for message-board issue #184 (chain-network rejected
//! cache, reconstructed-proposal variant).
//!
//! Node A is given a copy `T'` of a valid transaction `T` whose ZkSignature
//! proof has one flipped bit. `T'` has the same `mantle_txhash` as `T`, so it
//! is admitted under the same mempool key and resolves the same proposal
//! reference. Node B is given the genuine `T`. When a block containing `T` is
//! proposed, A reconstructs it with `T'`, fails batch proof verification,
//! records the genuine block ID in its rejected cache, and refuses every
//! descendant.
//!
//! This is a reproduction, not a regression test: it passes when the bug is
//! observed.

use std::{
    fs,
    path::{Path, PathBuf},
    slice,
    time::{Duration, Instant},
};

use lb_core::{
    header::HeaderId,
    mantle::{
        Note, OpProof, SignedOps, Utxo,
        gas::{MainnetGasProfile, TxGasCalculator as _},
        ledger::verification_mode::StandardMode,
        ops::OpId as _,
        traits::Hashable as _,
        transactions::{
            GasPrices, MantleTxBuilder, OpProofs, hash::TxHash, states::Preverified,
            tx_list::ops::OpsGasContext,
        },
    },
};
use lb_groth16::CompressedGroth16Proof;
use lb_key_management_system_service::keys::{ZkKey, ZkSignature};
use lb_node::config::RunConfig;
use lb_testing_framework::{
    DeploymentBuilder, LbcEnv, NodeHttpClient, TopologyConfig as TfTopologyConfig,
    configs::wallet::{WalletAccount, WalletConfig},
};
use lb_utils::math::NonNegativeRatio;
use logos_blockchain_tests::{
    common::manual_cluster::{LocalManualClusterHarnessBase, build_local_manual_cluster},
    cucumber::defaults::E2E_ARTIFACTS_DIR,
};
use serial_test::serial;
use testing_framework_core::scenario::{DynError, PeerSelection, StartNodeOptions, StartedNode};

const NODE_COUNT: usize = 2;
const WALLET_USERS: usize = 2;
const WALLET_FUNDS: u64 = 1_000_000_000;
const WARMUP_HEIGHT: u64 = 3;
const OBSERVE: Duration = Duration::from_secs(90);
const POLL: Duration = Duration::from_millis(500);

#[tokio::test]
#[serial]
async fn node_holding_proof_mangled_copy_rejects_block_and_descendants() {
    let (base, nodes) = start_cluster("rejected_cache_mempool_copy").await;
    let node_a = &nodes[0];
    let node_b = &nodes[1];
    println!("node A = {} ({}), node B = {} ({})", node_a.name, node_a.client.base_url(), node_b.name, node_b.client.base_url());

    wait_for_height(&node_a.client, WARMUP_HEIGHT, Duration::from_mins(3)).await;
    wait_for_height(&node_b.client, WARMUP_HEIGHT, Duration::from_mins(3)).await;

    let (genuine, mangled) = build_tx_pair(&base);
    let tx_hash = genuine.hash();
    assert_eq!(tx_hash, mangled.hash(), "copies must share mantle_txhash");
    println!("tx hash (shared by T and T'): {tx_hash:?}");

    let (res_a, res_b) = tokio::join!(
        node_a.client.submit_transaction(&mangled),
        node_b.client.submit_transaction(&genuine),
    );
    println!("submit T' -> A: {res_a:?}; submit T -> B: {res_b:?}");
    res_a.expect("A must admit the proof-mangled copy (preverify skips the proof)");
    res_b.expect("B must admit the genuine transaction");

    let view_a = node_a.client.test_mempool_view().await.unwrap();
    let view_b = node_b.client.test_mempool_view().await.unwrap();
    println!("mempool view A contains hash: {}; B contains hash: {}", view_a.contains(&tx_hash), view_b.contains(&tx_hash));

    let start = Instant::now();
    let mut including_block: Option<(HeaderId, u64)> = None;
    while start.elapsed() < OBSERVE {
        if let Some(found) = find_block_including(&node_b.client, &tx_hash).await {
            including_block = Some(found);
            break;
        }
        tokio::time::sleep(POLL).await;
    }
    let (block_id, block_height) =
        including_block.expect("B never produced or applied a block containing T");
    println!("B applied block {block_id:?} at height {block_height} containing T");

    let mut samples = Vec::new();
    let observe_start = Instant::now();
    while observe_start.elapsed() < OBSERVE {
        let a = info(&node_a.client).await;
        let b = info(&node_b.client).await;
        let a_has_block = node_a.client.block(&block_id).await.unwrap().is_some();
        let b_has_block = node_b.client.block(&block_id).await.unwrap().is_some();
        samples.push((observe_start.elapsed().as_secs(), a.height, b.height, a_has_block, b_has_block, a.tip, b.tip));
        tokio::time::sleep(Duration::from_secs(5)).await;
    }
    println!("t(s) | A height | B height | A has block | B has block | A tip | B tip");
    for (t, ah, bh, ahb, bhb, at, bt) in &samples {
        println!("{t:>4} | {ah:>8} | {bh:>8} | {ahb:>11} | {bhb:>11} | {at:?} | {bt:?}");
    }

    let a_final = info(&node_a.client).await;
    let b_final = info(&node_b.client).await;
    let a_has_block = node_a.client.block(&block_id).await.unwrap().is_some();
    let b_chain_has_block = chain_contains(&node_b.client, &block_id).await;
    let a_chain_has_block = chain_contains(&node_a.client, &block_id).await;
    println!("final: A height {} tip {:?}; B height {} tip {:?}", a_final.height, a_final.tip, b_final.height, b_final.tip);
    println!("A stored block: {a_has_block}; block on A canonical chain: {a_chain_has_block}; block on B canonical chain: {b_chain_has_block}");

    let log_counts = count_log_lines(base.scenario_base_dir(), node_a.name.as_str());
    println!("node A log counts: {log_counts:#?}");

    let restart_outcome = restart_and_observe(&base, node_a, node_b, &block_id).await;
    println!("after restart of A: {restart_outcome}");

    let log_counts_after = count_log_lines(base.scenario_base_dir(), node_a.name.as_str());
    println!("node A log counts after restart: {log_counts_after:#?}");

    assert!(b_chain_has_block, "B must keep the block containing T on its canonical chain");
    assert!(log_counts.batch_zkp_error > 0, "reproduction: A must have failed batch proof verification on the reconstructed block");
    assert!(log_counts.inserted_rejected > 0, "reproduction: A must have inserted the genuine block ID into its rejected cache");
    assert!(log_counts.cascade_skipped_enqueue > 0, "reproduction: A must have refused at least one descendant");
    // Whether A recovers before the restart depends on a proposal being missed
    // (a "cascade gap"); it is reported, not asserted.
    let _ = a_chain_has_block;
}

struct LogCounts {
    batch_zkp_error: usize,
    inserted_rejected: usize,
    cascade_skipped_enqueue: usize,
    tip_poll_not_enqueued: usize,
}

impl std::fmt::Debug for LogCounts {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("LogCounts")
            .field("batch_zkp_error", &self.batch_zkp_error)
            .field("inserted_rejected", &self.inserted_rejected)
            .field("cascade_skipped_enqueue", &self.cascade_skipped_enqueue)
            .field("tip_poll_not_enqueued", &self.tip_poll_not_enqueued)
            .finish()
    }
}

fn count_log_lines(scenario_dir: &Path, node_name: &str) -> LogCounts {
    let node_dir = fs::read_dir(scenario_dir)
        .ok()
        .and_then(|entries| {
            entries.flatten().map(|e| e.path()).find(|p| {
                p.is_dir() && p.file_name().and_then(|n| n.to_str()).is_some_and(|n| n.starts_with(node_name))
            })
        })
        .unwrap_or_else(|| scenario_dir.join(node_name));
    let mut counts = LogCounts {
        batch_zkp_error: 0,
        inserted_rejected: 0,
        cascade_skipped_enqueue: 0,
        tip_poll_not_enqueued: 0,
    };
    for path in walk_files(&node_dir) {
        let Ok(content) = fs::read_to_string(&path) else {
            continue;
        };
        for line in content.lines() {
            if line.contains("BatchZkpVerification") {
                counts.batch_zkp_error += 1;
            }
            if line.contains("inserted rejected block into cache") {
                counts.inserted_rejected += 1;
            }
            if line.contains("is in the rejected cache, skipping enqueue") {
                counts.cascade_skipped_enqueue += 1;
            }
            if line.contains("did not enqueue polled tip") {
                counts.tip_poll_not_enqueued += 1;
            }
        }
    }
    counts
}

fn walk_files(dir: &Path) -> Vec<PathBuf> {
    let mut out = Vec::new();
    let Ok(entries) = fs::read_dir(dir) else {
        return out;
    };
    for entry in entries.flatten() {
        let path = entry.path();
        if path.is_dir() {
            if path.file_name().is_some_and(|n| n == "db") {
                continue;
            }
            out.extend(walk_files(&path));
        } else if path.extension().is_some_and(|e| e == "log") || path.to_string_lossy().contains("__logs") {
            out.push(path);
        }
    }
    out
}

async fn restart_and_observe(
    base: &LocalManualClusterHarnessBase,
    node_a: &StartedNode<LbcEnv>,
    node_b: &StartedNode<LbcEnv>,
    block_id: &HeaderId,
) -> String {
    if let Err(error) = base.cluster().restart_node(&node_a.name).await {
        return format!("restart failed: {error}");
    }
    let deadline = Instant::now() + Duration::from_mins(3);
    loop {
        if let Ok(ready) = tokio::time::timeout(Duration::from_secs(5), node_a.client.consensus_info()).await
            && ready.is_ok()
        {
            break;
        }
        if Instant::now() > deadline {
            return "A did not come back after restart".to_owned();
        }
        tokio::time::sleep(POLL).await;
    }
    let deadline = Instant::now() + Duration::from_mins(4);
    loop {
        let on_chain = chain_contains(&node_a.client, block_id).await;
        let a = info(&node_a.client).await;
        let b = info(&node_b.client).await;
        if on_chain {
            return format!("A recovered: block on A canonical chain; A height {} B height {}", a.height, b.height);
        }
        if Instant::now() > deadline {
            return format!("A did not recover within 4 min; A height {} B height {}", a.height, b.height);
        }
        tokio::time::sleep(Duration::from_secs(2)).await;
    }
}

async fn info(client: &NodeHttpClient) -> lb_chain_service::CryptarchiaInfo {
    client.consensus_info().await.expect("consensus info").cryptarchia_info
}

async fn wait_for_height(client: &NodeHttpClient, target: u64, timeout: Duration) {
    let deadline = Instant::now() + timeout;
    loop {
        if let Ok(i) = client.consensus_info().await
            && i.cryptarchia_info.height >= target
        {
            return;
        }
        assert!(Instant::now() < deadline, "node did not reach height {target}");
        tokio::time::sleep(POLL).await;
    }
}

/// Walk the canonical chain from the tip and return the first block that
/// contains `tx_hash`, with its height.
async fn find_block_including(client: &NodeHttpClient, tx_hash: &TxHash) -> Option<(HeaderId, u64)> {
    let i = client.consensus_info().await.ok()?.cryptarchia_info;
    let mut id = i.tip;
    let mut height = i.height;
    for _ in 0..64 {
        let block = client.block(&id).await.ok().flatten()?;
        if block.transactions.iter().any(|tx| tx.hash() == *tx_hash) {
            return Some((id, height));
        }
        if block.header.parent_block == id || height == 0 {
            return None;
        }
        id = block.header.parent_block;
        height = height.saturating_sub(1);
    }
    None
}

async fn chain_contains(client: &NodeHttpClient, block_id: &HeaderId) -> bool {
    let Ok(i) = client.consensus_info().await else {
        return false;
    };
    let mut id = i.cryptarchia_info.tip;
    for _ in 0..256 {
        if id == *block_id {
            return true;
        }
        let Some(block) = client.block(&id).await.ok().flatten() else {
            return false;
        };
        if block.header.parent_block == id {
            return false;
        }
        id = block.header.parent_block;
    }
    false
}

fn build_tx_pair(
    base: &LocalManualClusterHarnessBase,
) -> (SignedOps<Preverified, StandardMode>, SignedOps<Preverified, StandardMode>) {
    let config = base.deployment().config();
    let account: &WalletAccount = config
        .wallet_config
        .accounts
        .first()
        .expect("wallet accounts seeded");
    let genesis_tx = config
        .genesis_block
        .as_ref()
        .expect("genesis block")
        .transactions_iter()
        .next()
        .expect("genesis tx");
    let transfer_op = genesis_tx.transfer().operation().clone();
    let op_id = transfer_op.op_id();
    let (idx, note) = transfer_op
        .outputs
        .iter()
        .enumerate()
        .find(|(_, note)| note.pk == account.public_key())
        .expect("wallet account has a genesis UTXO");
    let utxo = Utxo::new(op_id, idx, *note);

    let gas_context = OpsGasContext::new(Default::default(), Default::default(), GasPrices::default());
    let receiver = account.public_key();
    let provisional = MantleTxBuilder::new()
        .add_ledger_input(utxo)
        .unwrap()
        .add_ledger_output(Note::new(utxo.note.value, receiver))
        .unwrap()
        .build()
        .unwrap();
    let fee = provisional
        .by_ref()
        .total_gas_cost::<MainnetGasProfile>(&gas_context)
        .unwrap()
        .into_inner();
    let output_value = utxo.note.value.checked_sub(fee).expect("note covers fee");

    let build = || {
        MantleTxBuilder::new()
            .add_ledger_input(utxo)
            .unwrap()
            .add_ledger_output(Note::new(output_value, receiver))
            .unwrap()
            .build()
            .unwrap()
    };
    let tx = build();
    let signature = ZkKey::multi_sign(slice::from_ref(&account.secret_key), &tx.hash().to_fr())
        .expect("sign");

    let mut mangled_bytes = signature.as_proof().to_bytes();
    mangled_bytes[100] ^= 0x01;
    let mangled_signature = ZkSignature::new(CompressedGroth16Proof::from_bytes(&mangled_bytes));

    let genuine = SignedOps::from_parts(build(), OpProofs::from([OpProof::ZkSig(signature)]))
        .unwrap()
        .preverify()
        .expect("genuine preverifies");
    let mangled = SignedOps::from_parts(build(), OpProofs::from([OpProof::ZkSig(mangled_signature)]))
        .unwrap()
        .preverify()
        .expect("mangled copy preverifies: preverify does not check the ZkSig proof");
    (genuine, mangled)
}

/// Node A (index 0) is started with an empty wallet key list so its leader
/// never finds an eligible note: a leader that wins a slot while holding `T'`
/// evicts it at block assembly (`chain-leader/src/lib.rs`, "N removed"), which
/// turns the scenario into a plain reconstruction failure. The victim of #184
/// is a node that does *not* propose between admitting `T'` and receiving the
/// honest block.
async fn start_cluster(test_name: &str) -> (LocalManualClusterHarnessBase, Vec<StartedNode<LbcEnv>>) {
    let wallet = WalletConfig::uniform(WALLET_FUNDS, WALLET_USERS.try_into().unwrap()).unwrap();
    let base = build_local_manual_cluster(
        test_name,
        "rejected-cache-mempool-copy",
        DeploymentBuilder::new(
            TfTopologyConfig::with_node_numbers(NODE_COUNT).with_test_context(Some(test_name.to_owned())),
        )
        .with_wallet_config(wallet),
        Some(PathBuf::from(E2E_ARTIFACTS_DIR)),
    );

    let mut nodes: Vec<StartedNode<LbcEnv>> = Vec::with_capacity(NODE_COUNT);
    for node_index in 0..NODE_COUNT {
        let peers = if node_index == 0 {
            PeerSelection::None
        } else {
            PeerSelection::Named(vec![nodes[0].name.clone()])
        };
        let patch = move |cfg: RunConfig| {
            let mut cfg = config(cfg);
            if node_index == 0 {
                cfg.user.wallet.known_keys.clear();
                // At 1 s slots and f = 1/2 the watchdog's 3-block lag threshold
                // (6 s) is shorter than Blend's proposal delivery delay, so the
                // poll would fetch every block by download before its proposal
                // arrives. Production block intervals are far longer than the
                // Blend delay, so proposals arrive first; emulate that ordering.
                cfg.user.cryptarchia.network.sync.tip_poll.enabled = false;
            }
            Ok::<_, DynError>(cfg)
        };
        let node = Box::pin(base.cluster().start_node_with(
            &node_index.to_string(),
            StartNodeOptions::default()
                .with_peers(peers)
                .with_persist_dir(base.scenario_base_dir().join(format!("node-{node_index}")))
                .create_patch(patch),
        ))
        .await
        .unwrap_or_else(|error| panic!("starting node-{node_index} should succeed: {error}"));
        nodes.push(node);
    }
    base.cluster().wait_network_ready().await.expect("manual cluster should become ready");
    (base, nodes)
}

fn config(mut config: RunConfig) -> RunConfig {
    config.deployment.time.slot_duration = Duration::from_secs(1);
    config.user.cryptarchia.service.bootstrap.prolonged_bootstrap_period = Duration::ZERO;
    config.deployment.cryptarchia.security_param = 5.try_into().unwrap();
    config.deployment.cryptarchia.slot_activation_coeff = NonNegativeRatio::new(1, 2.try_into().unwrap());
    config
}
```

