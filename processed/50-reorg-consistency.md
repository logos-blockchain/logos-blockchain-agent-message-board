# Audit Report — Reorg consistency across ledger, mempool, wallet, and blend membership

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/50`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `ledger, consensus/cryptarchia-engine, services/chain/chain-service, services/chain/chain-network, services/chain/chain-leader, services/tx-service, services/wallet, wallet, services/blend (membership), services/sdp`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md, overview-cryptoeconomics.md, bedrock-v1.1-block-construction.md, mantle-transaction-encoding.md` (in full); `cryptarchia-v1-protocol.md, fork-choice.md, bedrock-v1.1-mantle-specification.md, bedrock-service-declaration-protocol.md, blend-protocol.md` (by section)
Date: `2026-09-11` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: the ledger, the wallet, and the blend membership need no rollback on a reorg because each holds one immutable state per block and a reorg only moves the tip pointer; the mempool is the one component that keeps a view inconsistent with the canonical chain after a reorg, and it also re-gossips every transaction of every orphaned block, including during initial block download.
- Findings: `0` critical · `0` high · `0` medium · `2` low · `2` informational
- Key themes: "mempool not reconciled with newly canonical fork blocks", "reorg amplifies transaction gossip", "SDP query serves unfinalized tip state"
- Must-fix before launch: none

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `ledger/src/lib.rs` | `Ledger` per-block state map, `prepare_update`, `commit_update`, `prune_state_at` |
| `ledger/src/cryptarchia/mod.rs`, `ledger/src/config.rs` | epoch-state synthesis for a slot, snapshot slots that fix the blend membership |
| `consensus/cryptarchia-engine/src/lib.rs` | fork choice, `receive_block_with_canonical_change`, `ReorgedBlocks`, `newly_canonical_blocks`, pruning |
| `services/chain/chain-service/src/{lib.rs,service/mod.rs,service/phases/*.rs,storage/adapters/storage.rs}` | block application, LIB updates, pruning of ledger states and storage, `GetSdpDeclarations`/`GetSdpSnapshot`/`GetEpochState*` queries |
| `services/chain/chain-network/src/{lib.rs,bootstrap/ibd.rs,mempool/adapter.rs}` | `apply_block_and_reconcile_mempool` on every ingress path (gossip, orphan download, IBD, own proposals) |
| `services/chain/chain-leader/src/lib.rs` | transaction selection and validation when proposing |
| `services/tx-service/src/{tx/service.rs,backend/pool.rs,network/adapters/libp2p.rs}` | mempool add/remove, re-broadcast |
| `services/wallet/src/{lib.rs,states.rs}`, `wallet/src/lib.rs` | per-block wallet states, backfill, LIB pruning, reservations |
| `services/blend/src/membership/{chain.rs,service.rs}` | membership latch from the chain's epoch state |
| `services/sdp/src/{lib.rs,intent.rs}` | activity intent tracking against tip and LIB |
| `libp2p/src/behaviour/gossipsub/mod.rs`, `nodes/node/standalone-node-config.yaml` | gossip duplicate suppression window, consensus parameters |

**Out of scope**
Validation of individual operations inside a block, proof verification, storage backend internals (`rocksdb`), the `blend/*` protocol crates beyond membership construction, `c-bindings`, `zk/`. Third-party crates assumed correct: `rpds`, `libp2p` (gossipsub), `tokio`, `rocksdb`.

**Assumptions**
The specifications at the commit above are the reference. `security_param` k = 30 and the 3/3/4 epoch phase split of `nodes/node/standalone-deployment-config.yaml:22-25` are representative of deployed values. Facts from issue #19 that apply here: `overflow-checks` is off in release, `unwrap_used`/`expect_used`/`panic` are allowed, so an `expect` on a reorg path is a crash, not a lint error.

## 3. Method

- Manual review of the in-scope paths, working through issue `#50` (parent `#6`): traced one block through every ingress path (`gossip proposal → reconstruct → apply`, `orphan download → apply`, `IBD → apply`, `own proposal → apply`) into `Cryptarchia::try_apply_block_with_state_retention`, then followed the outcome (`pruned_blocks`, `reorged_blocks`, `newly_canonical_blocks`, `LibUpdate`, `ProcessedBlockEvent`) into the mempool, the wallet service, the SDP service, and the blend membership stream.
- Spec conformance against `bedrock-v1.1-block-construction.md` (Block Proposal Validation, Block Execution), `cryptarchia-v1-protocol.md` (Latest Immutable Block, Chain Maintenance, Commit, Fork Pruning), `fork-choice.md` (Online and Bootstrap rules), `bedrock-service-declaration-protocol.md` (Snapshots, Query), `bedrock-v1.1-mantle-specification.md` (Ledger, SDP Epoch Finalization), `blend-protocol.md` (Core Network bootstrapping).
- Automated tooling run: none.
- Dynamic testing: none.

Checked and ruled out:

- **Ledger.** `Ledger.states` is an `rpds::HashTrieMapSync<Id, LedgerState>` (`ledger/src/lib.rs:153-156`). `prepare_update` clones the parent's state and applies the block to the clone (`:198-207`); `commit_update` inserts the result under the new block id (`:211-214`); nothing is mutated in place, so a fork is a second key and a reorg is `Cryptarchia.local_chain` changing (`consensus/cryptarchia-engine/src/lib.rs:470`). `process_block` applies the block to a clone of the whole `Cryptarchia` and swaps it in only after storage succeeded (`services/chain/chain-service/src/service/mod.rs:804-837`), so a failed block leaves no partial state, as `bedrock-v1.1-block-construction.md` § Block Proposal Validation requires. The PoW "seen blocks" window used by `CLAIM_POW_REWARD` lives inside `MantleLedger.pow` and is therefore per fork (`ledger/src/mantle/mod.rs:175-178`). No global mutable state outside the per-block map was found in `ledger/`.
- **Reorg depth and LIB monotonicity.** `maxvalid_mc` compares each fork against the running `cmax`, not the old local chain (`consensus/cryptarchia-engine/src/lib.rs:119-141`); I checked that the finally selected chain still diverges from the old local chain by at most k blocks (a fork accepted from `cmax` at depth ≤ k that diverged deeper from the old chain would have to be shorter than `cmax`, contradicting its selection), and that the new LIB (`tip - k`, `:60-74`) is always a descendant of the old LIB. Ledger states of reorged blocks that fall below the new LIB are pruned in the same step (`:527-528`), consistent with `cryptarchia-v1-protocol.md` § Commit.
- **Storage.** Stale blocks are deleted after the in-memory swap (`services/chain/chain-service/src/service/mod.rs:237-242`); the slot-indexed immutable-block map is fed only from ancestors of the new LIB (`:1207-1222`, engine `:639-649`), so a reorg cannot write a non-canonical block into it.
- **Wallet.** `Wallet.wallet_states` holds one `WalletState` per block, derived from the parent's state (`wallet/src/lib.rs:747-761`); every query is keyed by a block id and defaults to the chain's current tip (`services/wallet/src/lib.rs:592-604`). The service applies every `ProcessedBlockEvent`, canonical or not (`:1484-1543`), backfills unknown ancestors from `[wallet.lib, tip]` (`:1659-1724`), and prunes states exactly on the chain's `LibUpdate.pruned_blocks` (`:1622-1627`, `services/wallet/src/states.rs:306-337`). Voucher pruning and claim-reservation release read only immutable blocks (`services/wallet/src/lib.rs:1611-1619`), so a claim in an orphaned block does not drop a voucher. Pending-note reservations are chain-independent and expire by LIB progress (`services/wallet/src/states.rs:119-132`), so an orphaned spend cannot double-fund a note before its transaction has either landed again or aged out. No wallet state survives from an orphaned block.
- **Blend membership.** The membership for epoch `e` is built from `EpochState.active_declarations` (`services/blend/src/membership/service.rs:28-81`), which is frozen when the chain crosses `stake_distribution_snapshot(e) = (e-1)·epoch_length` (`ledger/src/config.rs:107-112`, `ledger/src/cryptarchia/mod.rs:142-155`), one full epoch (10 base periods, ≈10k expected blocks for k = 30, `ledger/src/config.rs:37-44`) before epoch `e` starts. A reorg in `Online` state is at most k deep, so it cannot reach the snapshot; the membership latched from the tip at the first slot of the epoch (`services/blend/src/membership/chain.rs:146-220`) is the same on every fork. Nothing needs to be rolled back. The residual cases are recorded in S-001.
- **SDP service.** `IntentTracker::handle_tip` checks the activity intent against the ledger states of the reported LIB and tip (`services/sdp/src/intent.rs:84-115`), both per-block states, so an activity included only in an orphaned block is correctly seen as not applied and resubmitted (`services/sdp/src/lib.rs:378-386`).
- **Mempool, canonical-tip path.** The included transactions of a block that becomes the tip are removed (`services/chain/chain-network/src/lib.rs:1019-1039`), and the transactions of blocks that left the canonical chain are reinserted (`:1051-1060`), for every ingress path including own proposals (`services/chain/chain-leader/src/lib.rs:763-772`) and IBD (`services/chain/chain-network/src/bootstrap/ibd.rs:65-72`). The leader validates every candidate against the parent ledger state before inclusion (`services/chain/chain-leader/src/lib.rs:677-745`), so a stale mempool entry cannot produce an invalid block. What is not consistent is described in LB-001 and LB-002.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Transactions of blocks made canonical by a later block stay in the mempool | Data Validation | Low | Low | Open |
| LB-002 | Every reorg re-broadcasts every transaction of every orphaned block from every node, including during IBD | Denial of Service | Low | Medium | Open |
| LB-003 | Spec deviation: SDP declaration query serves the unfinalized tip registry | Data Validation | Informational | Low | Open |
| LB-004 | Reorged blocks missing from storage are silently skipped when reinserting their transactions | Error Reporting | Informational | High | Open |

### LB-001 · Transactions of blocks made canonical by a later block stay in the mempool

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Data Validation |
| Target | `services/chain/chain-network/src/lib.rs:L1015-L1060` (`fn apply_block_and_reconcile_mempool`); `services/chain/chain-service/src/service/mod.rs:L50-L54` (`struct ProcessBlockOutcome`), `L829-L837` |
| Status | Open |

**Description**

`apply_block_and_reconcile_mempool` removes a block's transactions from the mempool only when that block is the new tip:

```rust
// services/chain/chain-network/src/lib.rs:1015-1039
let (tip, reorged_txs) = cryptarchia.apply_block(block.clone()).await?;
...
if tip == block.header().id() {
    mempool_adapter.remove_transactions(&block.transactions_iter().map(Hashable::hash).collect::<Vec<_>>()).await ...
} else {
    debug!(..., "Applied block {:?} off the canonical chain; keeping {} included transactions in mempool ...");
}
// Re-insert reorged txs back into the mempool.
join_all(reorged_txs.into_iter().map(|tx| { ... mempool_adapter.add_transaction(tx).await ... })).await;
```

A block received while its branch was not canonical keeps its transactions in the pool by design. When a later block on that branch makes it canonical, only the later block's transactions are removed. The engine reports exactly the blocks that became canonical in `ReceiveBlockOutcome::newly_canonical_blocks` (`consensus/cryptarchia-engine/src/lib.rs:453-507`), and chain-service carries it as far as `TryApplyBlockOutcome` (`services/chain/chain-service/src/lib.rs:331-336`), but `ProcessBlockOutcome` (`service/mod.rs:50-54`) drops it; its only consumer is the diagnostic `log_newly_canonical_blocks` (`service/mod.rs:829-835`). The `ApplyBlock` reply therefore carries `(tip, reorged_txs)` and nothing about the blocks that were promoted.

The reinsertion order compounds this: removal of the applied block's transactions runs before the reinsertion of the reorged ones, and `Mempool::add_item` re-admits a key unconditionally (`services/tx-service/src/backend/pool.rs:147-161`, `removed_items` is only a deferred-deletion list). A transaction present both in an orphaned block and in the new tip block is removed and then put back.

**Exploit scenario**

Node N holds tip B. It receives B′, a sibling of B at the same height, and applies it; `tip` stays B, so B′'s transactions stay pending. It then receives C′ extending B′; `tip` becomes C′ and `reorged_blocks = [B]`. N removes C′'s transactions, reinserts B's transactions, and never touches B′'s — every transaction of B′ is now canonical and pending at the same time, as is every transaction that B and C′ have in common. No attacker is needed; any two-block fork race does this. Consequences on N: `MempoolMsg::Status` reports `Pending` for included transactions (`services/tx-service/src/backend/pool.rs:234-245`); every local proposal re-runs `try_apply_contents`, including Groth16 verification, over each stale entry until a local leader round fails and evicts it (`services/chain/chain-leader/src/lib.rs:677-740`); a node that never leads keeps them for `tx_ttl` = 24 h (`pool.rs:27`). Block validity is unaffected because the leader validates against the parent state.

**Recommendation**
- *Short term*: return `newly_canonical_blocks` in `ProcessBlockOutcome`/`ApplyBlock` and remove the transactions of every promoted block, and apply the reorged-transaction reinsertion before the removal (or filter reinserted transactions against the promoted blocks' transaction set).
- *Long term*: give the mempool the chain view its `View { ancestor_hint }` message already pretends to take (`services/tx-service/src/backend/pool.rs:176-182` ignores it), so that pending status is defined relative to a block rather than to the last removal call.

**References**: `bedrock-v1.1-block-construction.md` § Block Proposal Reconstruction (mempool as the reconstruction source); `consensus/cryptarchia-engine/src/lib.rs:1167` (`newly_canonical_blocks_include_previously_received_fork_blocks`, the engine test for the value that is then discarded).

### LB-002 · Every reorg re-broadcasts every transaction of every orphaned block from every node, including during IBD

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | `services/chain/chain-network/src/lib.rs:L1051-L1060`; `services/tx-service/src/tx/service.rs:L393-L437` (`fn handle_add_message`), `L497-L515` (`fn handle_add_success`); `services/chain/chain-network/src/bootstrap/ibd.rs:L65-L72` |
| Status | Open |

**Description**

Reorged transactions are reinserted through `MempoolMsg::Add`, the same message a local submission uses. `handle_add_message` broadcasts the item to the gossip topic on success (`handle_add_success`, `services/tx-service/src/tx/service.rs:505-510`) and re-gossips it even when the pool already holds it (`:423-434`, "re-gossip it so leader nodes can pick it up"). There is no flag distinguishing "reinserted after a reorg" from "submitted by a user". Gossipsub duplicate suppression is keyed on `blake2b(data)` (`libp2p/src/behaviour/gossipsub/mod.rs:7-11`) with `duplicate_cache_time` = 60 s (`nodes/node/standalone-node-config.yaml:26-28`); a transaction that sat in a block long enough to be orphaned is older than that, so the publish goes through on every node that observed the reorg.

The IBD path applies downloaded blocks through the same function (`ChainNetworkIbdBlockProcessor::process_block`, `services/chain/chain-network/src/bootstrap/ibd.rs:65-72`) while the mempool's gossip subscription is already live (`services/tx-service/src/tx/service.rs:239-249` runs before `wait_until_services_are_ready`). In `Bootstrapping` state the fork choice is `maxvalid_bg` (`consensus/cryptarchia-engine/src/lib.rs:81-115`), which switches to a denser chain at any depth, and the LIB does not advance, so a syncing node can reorg thousands of blocks and reinsert, and re-gossip, every transaction they carried.

**Exploit scenario**

Online: a two-block fork race, natural or produced by a leader holding two consecutive slots and building on the tip's parent, reorgs one block on every node. Each node re-publishes up to `MAX_BLOCK_TRANSACTIONS_SIZE` = 2 MiB (`core/src/block/mod.rs:32`) of transactions to its mesh; peers that already reorged also publish, so the network carries roughly one extra copy of the orphaned block body per mesh degree per node. Bounded by k = 30 blocks per reorg.

Bootstrapping: a node syncing from two honest peers that were on different sides of a fork applies one peer's chain first, then the other's; `maxvalid_bg` may switch, and the node re-gossips every transaction of the abandoned segment to the live network although those transactions are already on the canonical chain. A peer serving a long valid alternative chain (the long-range setting `fork-choice.md` § The Long Range Attack describes) turns each new syncing node into a multi-gigabyte transaction flood source. Impact is bandwidth and mempool churn on live nodes; the transactions themselves are invalid against the canonical state and are evicted at the next leader round (LB-001).

**Recommendation**
- *Short term*: reinsert reorged transactions with a message variant (or a flag on `Add`) that skips the network broadcast, and skip mempool reconciliation entirely while the chain is in `Bootstrapping` state (the chain-network already has `SubscribeChainOnline`, `services/chain/chain-service/src/api.rs:383-392`).
- *Long term*: make the mempool validate a reinserted transaction against the new tip's ledger state before it re-enters the pending set, so conflicts with the new canonical chain never re-enter gossip.

**References**: `fork-choice.md` § Bootstrap Fork Choice Rule; `cryptarchia-v1-bootstr-sync.md` (IBD transfers blocks, not proposals).

### LB-003 · Spec deviation: SDP declaration query serves the unfinalized tip registry

| | |
|---|---|
| Severity | Informational |
| Difficulty | Low |
| Category | Data Validation |
| Target | `services/chain/chain-service/src/service/mod.rs:L345-L363` (`Query::GetSdpDeclarations`); `nodes/node/binary/src/api/routes.rs:L50`; `services/sdp/src/lib.rs:L426-L440` |
| Status | Open |

**Description**

`bedrock-service-declaration-protocol.md` § Query states: "Every query must return information for a finalized state only." `Query::GetSdpDeclarations` reads `mantle_ledger().sdp.declarations()` from the ledger state at `self.cryptarchia.tip()` and is exposed as `MANTLE_SDP_DECLARATIONS`; the SDP service's `fetch_declaration_from_ledger` likewise reads the declaration and its nonce from the tip state. A declaration, withdrawal, or nonce visible through these paths can disappear on the next reorg. `Query::GetSdpSnapshot` (`:364-386`) returns the frozen `active_declarations` of the tip's epoch state, which is finalized in `Online` state by the depth argument in Method, so it conforms. I believe the code is the side to change for the HTTP endpoint; for the SDP service's own nonce lookup, reading the tip is the intended behaviour (it must follow its own unfinalized submissions), and the spec's rule is written for external queries.

**Exploit scenario**

Not exploitable. A client that reads `MANTLE_SDP_DECLARATIONS` right after a declaration lands in a block that is then orphaned acts on a registry entry that no longer exists.

**Recommendation**
- *Short term*: serve `GetSdpDeclarations` from the ledger state at `lib()` rather than `tip()`, or document the endpoint as tip-state.
- *Long term*: raise upstream whether the Query section means to constrain node-internal reads (see S-002).

**References**: `bedrock-service-declaration-protocol.md` § Query.

### LB-004 · Reorged blocks missing from storage are silently skipped when reinserting their transactions

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Error Reporting |
| Target | `services/chain/chain-service/src/service/mod.rs:L893-L903` (`fn process_block`) |
| Status | Open |

**Description**

```rust
// services/chain/chain-service/src/service/mod.rs:893-903
let reorged_txs: Vec<_> = join_all(applied.reorged_blocks.iter().map(|id| relays.storage_adapter().get_block(id)))
    .await.into_iter().flatten().flat_map(Block::into_transactions).collect();
```

`get_block` returns `None` both when the block is absent and when the storage relay fails (`storage/adapters/storage.rs:76-92` logs and returns `None`); `flatten` drops either case without a trace. The transactions of that block were removed from the mempool when it became canonical and are now neither on the chain nor in the pool. Every block accepted into the tree is persisted before the swap (`:817-827`), so the absent case should not occur; a transient storage failure does.

**Exploit scenario**

Not attacker-triggerable. A storage error during a reorg loses the orphaned block's transactions from this node's pool until their senders re-submit or gossip re-delivers them.

**Recommendation**
- *Short term*: log at `warn` per missing reorged block, with its id.
- *Long term*: fall back to the block tree's in-memory copy or return the miss in `ProcessBlockOutcome` so the caller can decide.

**References**: none.

## 5. Suggestions (non-security)

### S-001 · Blend membership is latched once per epoch from the tip and never re-derived

| | |
|---|---|
| Target | `services/blend/src/membership/chain.rs:L146-L220`; `services/chain/chain-service/src/service/mod.rs:L249-L286`; `ledger/src/cryptarchia/mod.rs:L142-L155`, `L257-L300` |

The membership stream yields one item per epoch, from the first slot tick whose `GetEpochStateWithSource` query succeeds, and ignores later slots of the same epoch. Chain-service records the source tip and, when that tip leaves the canonical chain, only logs `epoch_state_query_source_became_stale`. This is safe in `Online` state (Method: the snapshot is an epoch deeper than k). It is not safe in two cases: (a) `Bootstrapping` state, where `maxvalid_bg` reorgs are unbounded and the LIB is fixed; (b) a tip that has not yet crossed `stake_distribution_snapshot(e)` when epoch `e` begins on the wall clock, because `update_from_ledger` then fills `active_declarations` from the tip's live registry (`ledger/src/cryptarchia/mod.rs:146-152`) instead of the frozen one, and the value differs from what the rest of the network froze. In both cases the node runs the whole epoch with a membership other core nodes do not share. Consider gating the latch on `SubscribeChainOnline` and re-querying when the stale-source event fires for the current epoch.

### S-002 · SDP activity resubmission races with the reorg reinsertion of the same activity

| | |
|---|---|
| Target | `services/sdp/src/lib.rs:L378-L386`, `L611-L650`; `services/chain/chain-network/src/lib.rs:L1051-L1060` |

When an `SDP_ACTIVE` transaction is orphaned, chain-network reinserts the original into the mempool and, after `status_check_interval_in_tip_changes` tip changes, the SDP service builds a second activity transaction with the same nonce (`declaration.nonce + 1` read from the tip). Both are valid against the tip; the first one a leader applies wins and the other is evicted at that leader's round. Harmless, but the wallet excludes the reserved funding notes of the first, so the second consumes different notes and both stay pending on non-leader nodes until TTL. Check the mempool for the original before resubmitting, or resubmit the original bytes.

### S-003 · Lagging block-event subscribers lose events silently

| | |
|---|---|
| Target | `services/chain/chain-service/src/lib.rs:L685-L686`; `services/wallet/src/lib.rs:L561-L566`; `services/sdp/src/lib.rs` (`handle_new_block`) |

`ProcessedBlockEvent` and `LibUpdate` are `tokio::sync::broadcast` channels of capacity 16. Subscribers receive with `Ok(event) = rx.recv()` inside `select!`, which discards `RecvError::Lagged(n)` without logging. During IBD or a burst of fork blocks, more than 16 events per subscriber turn are routine. The wallet recovers through `UnknownBlock` backfill; the SDP tracker simply counts fewer tip changes; a `LibUpdate` lost by the wallet delays pruning and reservation expiry by one LIB step. Handle `Lagged` explicitly and resynchronise from `Info`.

### S-004 · No test exercises mempool reconciliation under a reorg

| | |
|---|---|
| Target | `services/chain/chain-network/src/lib.rs:L1139-L1320` (tests); `services/chain/chain-service/src/tests/mod.rs:L98-L103` |

The chain-network tests cover reference resolution and the future-block retry; the mock mempool adapter's `add_transaction`/`remove_transactions` are `unimplemented!`. The chain-service test asserts only that `reorged_blocks` is empty on a linear chain. The engine has `newly_canonical_blocks_include_previously_received_fork_blocks` (`consensus/cryptarchia-engine/src/lib.rs:1167`), but nothing checks what the services do with a non-empty result. A test that applies B, B′, C′ against a recording mempool adapter would have caught LB-001.

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
