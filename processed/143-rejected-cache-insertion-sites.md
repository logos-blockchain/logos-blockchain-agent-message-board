# Audit Report — Rejected-block cache: which apply errors may condemn a block ID, on every insertion site

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/143`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `services/chain/chain-network` (`lib.rs`, `sync/orphan_handler.rs`, `sync/rejected_blocks.rs`, `bootstrap/ibd.rs`, `network/adapters/libp2p.rs`), `services/chain/chain-service` (`api.rs`, `lib.rs`, `uncle.rs`, `service/mod.rs`, `service/phases/*`), `core/src/block`, `core/src/mantle/transactions/tx_list`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `cryptarchia-v1-bootstr-sync.md`, `fork-choice.md`; by section: `cryptarchia-v1-protocol.md` §Uncle References, §Block Header Validation, §Chain Maintenance; `bedrock-v1.1-block-construction.md` §Block, §Block Proposal Reconstruction, §Reference Resolution, §Binding of the reference list, §Block Proposal Validation
Date: 2026-09-12 — author: `agent (Claude)` — status: `final`

---

## 1. Summary

- Overall assessment: the rejected-block cache is written from seven sites, and the classification that guards five of them is an allowlist of four *recoverable* errors, so every other failure, including failures that read bytes the block ID does not commit to, is recorded as a permanent verdict on the block. The most consequential case is new: a chainsync provider can stream a copy of a genuine block whose transaction proofs are mangled; the copy passes every binding check, fails batch proof verification, and the genuine block ID is condemned on the downloading node, after which every descendant is refused by the orphan pipeline.
- Findings: 0 critical · 1 high · 0 medium · 2 low · 1 informational
- Key themes: "`ApiError::Unexpected(String)` erases the error type before the classification runs", "`mantle_txhash` excludes `op_proofs`, so proof failures are never a verdict on a block ID", "block IDs enter the cache before any byte of the block is authenticated", "the cache is inert during IBD"
- Must-fix before launch: LB-001 (a single unauthenticated peer can wedge a syncing node's orphan pipeline until restart); the same fix closes #184.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/chain/chain-network/src/lib.rs` L404-L509, L571-L727, L775-L796, L845-L936, L1001-L1063 | every `insert_rejected_block` call, `should_process_block`, `is_recoverable_apply_error`, the apply and reconstruction paths |
| `services/chain/chain-network/src/sync/orphan_handler.rs` L119-L328, L331-L473, L725-L800 | `enqueue_orphan`, `dequeue_next_orphan`, `cancel_active_download`, the stream state machine, existing cache tests |
| `services/chain/chain-network/src/sync/rejected_blocks.rs` (whole file) | the LRU wrapper and the capacity-0 switch |
| `services/chain/chain-network/src/sync/config.rs`, `nodes/node/binary/src/config/cryptarchia/serde/network.rs` L70-L83, `nodes/node/standalone-node-config.yaml` L143-L149, `tests/testing_framework/src/framework/local/provisioning.rs` L827 | configured capacity and download fan-out |
| `services/chain/chain-network/src/bootstrap/ibd.rs` L34-L78, L161-L228, L323-L339 | the IBD copy of the downloader and how it reacts to apply errors |
| `services/chain/chain-network/src/network/adapters/libp2p.rs` L72-L128, L341-L446 | how streamed blocks are decoded and which peers are asked |
| `services/chain/chain-service/src/api.rs` L33-L49, L317-L352 | the `ApiError` mapping every apply error passes through |
| `services/chain/chain-service/src/lib.rs` L91-L126, L425-L500, L889-L947; `uncle.rs` L20-L170; `service/mod.rs` L186-L247, L773-L838; `service/phases/awaiting_genesis_time.rs` L141-L170; `service/phases/ibd.rs` L85-L103 | every producer of every `Error` variant and the phase handlers of `ApplyBlock` |
| `core/src/block/mod.rs` L104-L130, L215-L285, L335-L379; `core/src/utils/merkle.rs` L14-L26; `core/src/mantle/transactions/tx_list/signed_ops.rs` L222-L231; `core/src/mantle/batch.rs` L89-L98 | what a block ID, `body_root` and the transaction hash commit to |
| `ledger/src/lib.rs` L117-L150; `consensus/cryptarchia-engine/src/lib.rs` L232-L246, L385-L392 | the ledger and engine error variants that reach the cache |

**Out of scope**

The chainsync provider side (`ChainSync` handling in chain-service, `consensus/cryptarchia-sync`), gossipsub scoring and message validation (report #142 LB-002, issue #144), mempool admission (report #179), the correctness of ledger and proof verification themselves, and the tip-poll peer sampling beyond its enqueue call. Third-party crates assumed correct: `lru` 0.18.2, `libp2p`, `tokio`, `futures`. No dynamic reproduction was run; issue #184 item 4 keeps that task.

**Assumptions**

The spec text at the `logos-lips` commit above is authoritative for what a rejection may establish about a block ID. The node is past IBD and in the `Following` or `PBP` phase unless stated. Repo-level facts from #19 apply (release profile without overflow checks, panics allowed).

## 3. Method

- Manual review of the in-scope paths, working through issue `#143` (parent `#3`) and its four checklist items, building on report #142 (`inbox/43-block-validation-ordering.md`, LB-004 and LB-005), report #179 (`inbox/113-mempool-admission-validation.md`, LB-001) and issue #184. Findings of those reports are cited, not repeated.
- Spec conformance against `bedrock-v1.1-block-construction.md` §Block Proposal Validation (the classification of what a failed check establishes, master revision, which is longer than the local rfc-branch copy and was read from `origin/master`), §Reference Resolution, §Binding of the reference list, and `cryptarchia-v1-protocol.md` §Block Header Validation rules 4, 9, 10 and §Uncle References; `cryptarchia-v1-bootstr-sync.md` §Initial Block Download, §Listening for New Blocks, §Downloading Blocks.
- Automated tooling: none. Every claim is anchored to a line at the target SHA.
- Dynamic testing: none.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | A chainsync provider can condemn a genuine block ID by streaming a copy with mangled transaction proofs | Consensus | High | Medium | Open |
| LB-002 | Block IDs enter the rejected cache before any byte of the block is authenticated, so an unauthenticated peer can flush it | Denial of Service | Low | Low | Open |
| LB-003 | `AwaitingGenesisTime` and storage failures are recorded as permanent verdicts on the block | Data Validation | Low | High | Open |
| LB-004 | The rejected cache is inert during IBD and the IBD downloader never records an invalid tip chain | Denial of Service | Informational | Low | Open |

### 4.0 The seven insertion sites and what reaches them (checklist items 1 and 2)

`RejectedBlocks::insert` (`sync/rejected_blocks.rs` L43-L51) is reached from seven places. S1 to S5 are in `chain-network/src/lib.rs`, S6 and S7 in `sync/orphan_handler.rs`:

| Site | Line | Trigger | Bytes examined before the ID is recorded |
|---|---|---|---|
| S1 | L449 | orphan-download stream: `should_process_block` returns `OlderThanLib` | header slot only; the block was decoded by `Block::try_from` (`libp2p.rs` L375, `core/src/block/mod.rs` L366-L378), which checks the header's own signature and `body_root`, but nothing about the leader or the chain |
| S2 | L467-L468 | orphan-download stream: `apply_block_with_future_block_retry` fails and `is_recoverable_apply_error` is false | whatever the chain-service error read (table below) |
| S3 | L593 | gossip proposal: `OlderThanLib` | header slot only; runs *before* `verify_header_alone` (L608) |
| S4 | L615 | gossip proposal: `verify_header_alone` fails | genesis-slot check and the Ed25519 signature over the header, which is outside the header and so outside the block ID (report #142 LB-004) |
| S5 | L688-L689 | reconstructed proposal: apply fails, not recoverable | as S2, on transactions taken from the local mempool |
| S6 | `orphan_handler.rs` L159-L170 | `enqueue_orphan`: the block or its parent is cached | none, cascade |
| S7 | `orphan_handler.rs` L237-L248 | `dequeue_next_orphan`: the block or its parent is cached | none, cascade |

The IBD phase constructs its own downloader with the same capacity (`bootstrap/ibd.rs` L166-L174) but never inserts: `drain_downloader` cancels on any error other than `AlreadyApplied` (L208-L218) and `enqueue_tips` passes `parent_id = None` (L323-L339), so S6 and S7 cannot fire either (LB-004). The tip poll enqueues with `parent_id = None` too (`lib.rs` L783), so it is gated by S6 on the tip ID alone and never inserts by itself.

S2 and S5 are the sites the issue is about. Both pass through `CryptarchiaServiceApi::apply_block` (`chain-service/src/api.rs` L317-L352), which keeps `ParentMissing`, `FutureBlock` and `AlreadyApplied` as typed variants and turns **every other** `chain_service::Error` into `ApiError::Unexpected(format!("Failure while applying block: {err:?}"))` (L350). `is_recoverable_apply_error` (`chain-network/src/lib.rs` L926-L936) then allowlists those three plus `CommsFailure`; anything else is terminal. The table classifies each variant of `chain_service::Error` (`chain-service/src/lib.rs` L91-L126) by what its failure is a verdict on. "Committed" means the bytes the check read are bound to the block ID through `header.body_root` (`core/src/block/mod.rs` L355-L364), whose transaction leaves are `mantle_txhash` over the operations only (`merkle.rs` L14-L15; `signed_ops.rs` L222-L231 hashes `op_refs()`, which excludes `op_proofs`).

| `chain_service::Error` variant | Produced at | Reaches chain-network as | Recorded? | Verdict on | Spec says |
|---|---|---|---|---|---|
| `ParentMissing` | `lib.rs` L472-L475, L484-L487; `uncle.rs` L34-L41 | `ParentMissing` | no | block-tree state | correct (rule 7 is a tree question) |
| `FutureBlock` | `lib.rs` L443-L448 | `FutureBlock`, after 3 retries of 500 ms (`chain-network/src/lib.rs` L78-L79, L938-L999) | no | local clock | correct |
| `AlreadyApplied` | `lib.rs` L438-L440 | `AlreadyApplied` | no | — | correct |
| relay failure | `api.rs` L331-L336 | `CommsFailure` | no | local | correct |
| `Ledger(InvalidSlot)`, `Consensus(InvalidSlot)` | `ledger/src/lib.rs` L118-L119; `cryptarchia-engine` L244-L246 | `Unexpected` | **yes** | header vs. parent (rule 5), committed | correct |
| `Ledger(InvalidProof)` | `ledger/src/lib.rs` L122-L123 | `Unexpected` | **yes** | header PoL (rule 9), committed | correct |
| `InvalidUncle{NotStrictlyOlder, ParentNotOnChain, OnChain, InvalidSlot, InvalidSignature, InvalidProof}` | `uncle.rs` L44-L50, L100-L122, L66-L74 | `Unexpected` | **yes** | the carried uncle entries, committed by `body_root`, which `Block::reconstruct`/`try_from` confirmed before apply (`block/mod.rs` L230, L246, L280) | correct (rule 10; the spec's proposal-path caveat in §Block Proposal Validation step 3 does not apply because the code checks `body_root` first; the cost side is #142 LB-005) |
| `Ledger(...)` operation-level: `InsufficientBalance`, `BalanceOverflow`, `UnbalancedTransaction`, `GasOverflow`, `Mantle`, `Inputs`, `Outputs`, `InputInGenesis`, `MissingTransferGenesis`, `InsufficientExecutionFee`, `TooMuchExecutionGas`, `InvalidStoragePrice`, `VerificationError::{ChannelNotFound, KeyNotFound}` | `ledger/src/lib.rs` L124-L149 via `prepare_update` (`chain-service/src/lib.rs` L461-L477) | `Unexpected` | **yes** | the operations, committed | correct |
| `Ledger(VerificationError)` multisig variants (`core/src/mantle/transactions/errors.rs` L14-L16 and following) | as above | `Unexpected` | **yes** | `op_proofs` bytes, **not committed** | **wrong**: a copy-level failure |
| `BatchZkpVerification{InvalidZkSignatures, MalformedZkSignature, InvalidLeaderClaimProofs, MalformedLeaderClaimProof}` | `chain-service/src/lib.rs` L478 → `core/src/mantle/batch.rs` L89-L98 | `Unexpected` | **yes** | `op_proofs` bytes, **not committed** | **wrong**: LB-001 (streamed copy), #184 (mempool copy) |
| `Storage(String)` | `service/mod.rs` L817-L827 | `Unexpected` | **yes** | local I/O; the in-memory state is untouched (`candidate` clone L804, committed at L837 only after the store) | **wrong**: #142 LB-004, restated in LB-003 |
| `AwaitingGenesisTime` | `phases/awaiting_genesis_time.rs` L145-L147, L160-L170 | `Unexpected` | **yes** | service phase / local clock | **wrong**: LB-003 |
| `Serialisation`, `InvalidBlock`, `Mempool` | no producer on the apply path at this SHA (`grep` over `chain-service/src`; `Serialisation` has only its `#[from]`) | — | — | — | unreachable |
| `HeaderIdNotFound`, `ParentIdNotFound` | recovery-time loaders only (`lib.rs` L889-L947; `service/mod.rs` L1011-L1074) | — | — | — | unreachable via `ApplyBlock` |
| `Consensus(OrphanMissing)` | no producer (`cryptarchia-engine/src/lib.rs` L389 is the only non-test mention) | — | — | — | unreachable |

On the proposal path the errors of chain-network itself are handled before apply: `InvalidHeader` is S4; `UnresolvedReference`, `CollidingReference`, `NoMatchingReconstruction`, `Mempool` and `BoundedError` from `reconstruct_block_from_proposal` are deliberately not recorded (`lib.rs` L628-L641), which matches §Reference Resolution.

The mismatch is structural rather than a list of missed variants: the classification is an allowlist of *recoverable* errors applied to a `String`, so every new `chain_service::Error` variant is terminal by default, and the two variants that read uncommitted bytes (`BatchZkpVerification`, multisig `VerificationError`) cannot be told apart from committed ones once they have been formatted (S-001).

### LB-001 · A chainsync provider can condemn a genuine block ID by streaming a copy with mangled transaction proofs

| | |
|---|---|
| Severity | High |
| Difficulty | Medium |
| Category | Consensus |
| Target | `services/chain/chain-network/src/lib.rs:L457-L471` (orphan-download arm), `:L926-L936` (`is_recoverable_apply_error`); `services/chain/chain-service/src/api.rs:L350`; `core/src/mantle/transactions/tx_list/signed_ops.rs:L222-L231`; `core/src/block/mod.rs:L276-L285`, `:L366-L378` |
| Status | Open |

**Description**

Checklist item 3 asked whether a malicious chainsync provider can get a genuine block ID inserted. It can, and it does not need to stream out of order.

A streamed block is decoded by `Block::try_from(bytes)` (`libp2p.rs` L373-L377), which runs `into_verified` (`core/src/block/mod.rs` L235-L249): the header's own signature (`verify_header_alone`, L337-L351), the transaction size bound, and `body_root` (L276-L285). `body_root` commits to the uncle headers and to the Merkle root over `tx.hash()` (`merkle.rs` L14-L15). For the node's transaction type that hash is `mantle_txhash`, computed over `op_refs()` — the operations — and not over `op_proofs` (`signed_ops.rs` L222-L231; report #179 §LB-001 established the same fact for the mempool key). A copy of a genuine block in which every ZkSignature or LeaderClaim proof is replaced by other bytes of the same fixed size therefore has the same `body_root`, the same header, the same signature and the same block ID as the genuine block, and passes decoding.

The orphan-download arm then calls `apply_block_with_future_block_retry` (`lib.rs` L457). Inside chain-service, `try_apply_block_with_state_retention` runs `verify_uncles`, `prepare_update` and `verify_batch_proofs` (`chain-service/src/lib.rs` L451-L478); the last fails with `BatchZkpVerification` (`batch.rs` L89-L98), which `apply_block` formats into `ApiError::Unexpected` (`api.rs` L350). Back in chain-network, `is_recoverable_apply_error` returns false (L926-L936), and `insert_rejected_block(header_id)` records the **genuine** block's ID (L467-L468). `cancel_active_download` (L470) then drops only the active orphan from the queue (`orphan_handler.rs` L297-L308); nothing marks the provider.

The cache is consulted on every later `enqueue_orphan` (`orphan_handler.rs` L159-L170) and `dequeue_next_orphan` (L237-L248) with `contains_block_or_parent`, which also promotes the entry (`rejected_blocks.rs` L39). A gossiped descendant fails apply with `ParentMissing`, is handed to `enqueue_orphan(child, Some(parent))`, is refused and is itself inserted (S6); the tip poll goes through the same gate with `parent_id = None` (`lib.rs` L783), so it is refused as soon as the polled tip itself has been inserted by the cascade. Report #179 §LB-001 and issue #184 describe this cascade for the mempool-copy variant; the stream variant reaches the same site with a different attacker and without needing the victim to hold any transaction.

The spec is explicit that this is not allowed. §Block Proposal Validation (master): "every byte outside the header ... can be altered in a copy without changing the `block_id` it names ... A failure detected in them is a property of the received copy, not of the block ... Treating any of the former as final would let an attacker censor a genuine block by circulating tampered copies of it". `op_proofs` are such bytes, because `mantle_txhash` does not cover them (`bedrock-v1.1-mantle-specification.md` §Mantle Transaction Hash, as cited by report #179). §Downloading Blocks assumes a failed download simply returns and may be retried with other peers; it does not contemplate a verdict being kept.

**Exploit scenario**

Preconditions: the victim has an orphan download in flight, and the attacker's stream is the first to deliver a valid first item (`select_ok`, `libp2p.rs` L443-L444; `check_first_block_response_ready` L82-L90 only rejects a first item that is an `Err`). The victim asks up to 16 connected and 16 discovered peers per download (`standalone-node-config.yaml` L148-L149), so an attacker that is merely *discovered* and answers fast is a candidate. The attacker can also create the download: a proposal with a self-signed header whose `parent` is a fresh random ID passes `verify_header_alone` (`lib.rs` L608), reconstructs with zero references, and fails apply with `ParentMissing` before any PoL check (`chain-service/src/lib.rs` L434-L441 via `verify_uncles`, then `prepare_update`), which enqueues it (`lib.rs` L660-L672); the victim then requests `target = fake_id` from its peers, honest peers have nothing to serve, and the attacker serves whatever it likes, since the stream arm never checks that a streamed block is on the path to the target.

1. The attacker holds the genuine block `B` (the newest honest block, or any block the victim has not applied) with its transactions, and streams `B'`: the same header, signature and uncle headers, and the same transactions with every proof replaced.
2. The victim decodes `B'` (passes), applies it, fails at `verify_batch_proofs`, records `id(B)` (S2), cancels the download.
3. Every later proposal whose ancestry passes through `B` fails with `ParentMissing` and is refused at S6, each refused ID being inserted in turn. With the cascade at one insertion per honest block and touch-on-hit on the parent, the newest rejected descendant is always resident, so the 1000-entry LRU never evicts the frontier.

Exits: a restart (the cache is in-memory); a tip poll whose chosen tip the victim has not yet seen through gossip, which succeeds at S6 because `parent_id = None` and lets an honest provider stream `B` (downloaded ancestors are not checked against the cache); or LB-002, which lets anyone flush the cache. The first is operator action; the second is a race the attacker re-enters by answering the next download first; the third is a second bug. Issue #184 items 2 and 4 keep the measurement and the two-node reproduction. The result is a liveness denial of service against any node that needs `B` from synchronisation, from one unauthenticated peer.

**Recommendation**

- *Short term*: in `is_recoverable_apply_error`, stop treating `ApiError::Unexpected` as terminal. Until the error type is carried across the API (S-001), match the formatted string for `BatchZkpVerification` and the multisig `VerificationError` variants and treat them as copy-level: do not record the ID, do not cancel the queue on their account, and re-request the block from another peer (issue #184's fix covers the reconstructed-proposal site with the same rule). On the stream arm, require that each streamed block's parent is either the previously streamed block or already in the tree before applying it, so an unrelated block cannot be injected through a download the attacker provoked.
- *Long term*: make `ApiError` carry `chain_service::Error` (boxed) and invert the classification into an allowlist of block-level verdicts: `InvalidSlot`, `InvalidProof`, `InvalidUncle`, and the operation-level ledger errors, which read only committed bytes. Everything else is a verdict on the copy or on local state. Add a test per site that asserts a `BatchZkpVerification` failure leaves the cache empty. Separately, decide at the spec level whether `mantle_txhash` should commit to `op_proofs`; while it does not, no proof failure can ever be attributed to a block ID by any node, and the download requester needs a provider-level penalty instead.

**References**: `bedrock-v1.1-block-construction.md` §Block Proposal Validation (closing paragraph), §Binding of the reference list; `cryptarchia-v1-protocol.md` §Block Header Validation rule 4; `cryptarchia-v1-bootstr-sync.md` §Downloading Blocks; report #179 LB-001; report #142 LB-004; issue #184.

### LB-002 · Block IDs enter the rejected cache before any byte of the block is authenticated, so an unauthenticated peer can flush it

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `services/chain/chain-network/src/lib.rs:L440-L451` (S1), `:L589-L595` (S3), `:L845-L868` (`should_process_block`), `:L877-L907` (`is_after_lib`) |
| Status | Open |

**Description**

`should_process_block` reads only the header slot (`is_at_or_before_lib`, L910-L912) and asks whether a ledger state exists for the ID (L859-L861). On the gossip path it runs before `verify_header_alone` (S3 at L593 precedes L608), and on the stream path (S1) the only prior check is `Block::try_from`, which verifies a signature under whatever key the header itself names. A block ID is `hash(header)` and the slot is a header field, so a peer can mint an unbounded number of distinct 297-byte headers with `slot <= lib_slot`, sign each with its own key, and have each ID inserted. `RejectedBlocks::insert` counts a fresh insertion per ID (L47-L50), and `LruCache::put` evicts the least recently used entry once 1000 are held (`lru` 0.18.2). One thousand such messages on the block topic, or one download stream carrying them, evict every entry the cache holds, including the ones that protect the node from re-downloading a known-invalid chain, which is the purpose the type documents (`rejected_blocks.rs` L9-L10).

`is_after_lib` is also fail-open: a failed `info()` call returns `true` (L903-L906), which report #202 (issue #34) already records; it means S1 and S3 are skipped, not that anything wrong is recorded, so it is only noted here.

**Exploit scenario**

An attacker wants a syncing victim to keep re-fetching a chain it has already rejected, or, conversely, wants to lift a rejection it previously planted (LB-001) at a moment of its choosing. It publishes 1000 self-signed pre-LIB headers on the block topic; each node in the mesh inserts 1000 IDs and evicts everything else. Cost: 1000 Ed25519 signatures and about 370 KiB. The gossipsub layer forwards these before validation (report #142 LB-002), so one message stream reaches the whole mesh.

**Recommendation**

- *Short term*: move S3 after `verify_header_alone`, and on the stream arm do not record `OlderThanLib` at all; a pre-LIB block from a download is simply skipped, since nothing will ever enqueue it (S6 refuses any orphan whose slot is behind the LIB through `ParentMissing`'s `info.lib`, and the tip poll only offers tips ahead of the local height, `tip_poll.rs` L54-L66).
- *Long term*: record an ID only after a verdict that needed the leader's proof (rule 9), so that every entry cost its author a lottery win; count evictions of never-hit entries in `metrics::orphan_blocks_rejected_inserted_total` so a flush is visible.

**References**: report #142 LB-002; report #202 (issue #34) on `is_after_lib`; `lru` 0.18.2 `LruCache::put`.

### LB-003 · `AwaitingGenesisTime` and storage failures are recorded as permanent verdicts on the block

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Data Validation |
| Target | `services/chain/chain-service/src/service/phases/awaiting_genesis_time.rs:L145-L147`, `:L160-L170`; `services/chain/chain-service/src/service/mod.rs:L817-L827`; `services/chain/chain-service/src/api.rs:L350`; `services/chain/chain-network/src/lib.rs:L467-L468`, `:L688-L689` |
| Status | Open |

**Description**

Two `chain_service::Error` variants that describe the node rather than the block reach S2 and S5 as `Unexpected` and are recorded:

- `AwaitingGenesisTime`: while chain-service is in its first phase, every `ApplyBlock` is answered with this error (`awaiting_genesis_time.rs` L145-L147, L160-L170). Chain-network's loop starts after IBD, and IBD is skipped when no peers are configured (`ibd.rs` L136-L139), so a node with no IBD peers whose clock is behind the network's genesis time receives genuine slot-1.. proposals and records each ID. After genesis, `FutureBlock` covers the same clock skew and is correctly classified as recoverable; before genesis, the phase error is not.
- `Storage`: `process_block` stores the block before committing the candidate state (`service/mod.rs` L804-L837), so a RocksDB write failure returns `Storage(..)` with the in-memory state unchanged. Report #142 LB-004 already lists this; it is restated because the consequence differs from the others in the table: the block was never applied, a retry would succeed once storage recovers, and the cache is exactly what prevents the retry.

Neither is a property of the block; §Block Proposal Validation permits only header-implied failures to condemn a block ID.

**Exploit scenario**

Not attacker-driven. A validator that starts a few minutes early with a slow clock, or one whose disk is briefly full at the moment the newest block arrives, condemns that block and, through S6, every block after it until restart. The difficulty rating reflects that the trigger is an operational fault, not a message.

**Recommendation**

- *Short term*: add `AwaitingGenesisTime` and `Storage` to the recoverable set; for `Storage`, cancel the download (as today) but do not record.
- *Long term*: S-001.

**References**: report #142 LB-004; `cryptarchia-v1-bootstr-sync.md` §Setting the Fork Choice Rule (the node is expected to start before or at genesis).

### LB-004 · The rejected cache is inert during IBD and the IBD downloader never records an invalid tip chain

| | |
|---|---|
| Severity | Informational |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `services/chain/chain-network/src/bootstrap/ibd.rs:L166-L174`, `:L206-L227`, `:L323-L339` |
| Status | Open |

**Description**

Checklist item 1 named IBD as an insertion site. It is not: `download_blocks` builds a downloader with `orphan_config.max_rejected_cache_size` (L166-L174), but `drain_downloader` only distinguishes `AlreadyApplied` (L210-L214) from "any other error", on which it calls `cancel_active_download` (L215-L218) and never `insert_rejected_block`; `enqueue_tips` passes `parent_id = None` (L333), so the cascade sites S6 and S7 cannot fire either. During IBD the capacity setting has no effect, and the downloader's own test fixture sets it to 0 (L720-L725). The comment on the `block_apply_error_triggers_cancel` test (L512-L516) records the consequence: IBD "keeps retrying that same tip (no per-tip retry limit yet)". `cryptarchia-v1-bootstr-sync.md` §Initial Block Download says a peer whose chain does not apply should count as a failed peer and IBD should terminate when none succeeds; the code loops with `round_delay` instead. That liveness gap is issue #135 and is not restated as a finding here.

The observation for this issue is the asymmetry: the one phase in which a bounded negative cache would stop a malicious IBD peer from being re-fetched every round is the phase that does not use it, while the online phase uses it on unauthenticated input (LB-002).

**Recommendation**

- *Short term*: none needed for the cache itself; fix #135 in IBD.
- *Long term*: when the classification of S-001 exists, let IBD record block-level verdicts and count them per peer, so a peer serving an invalid chain is dropped from `config.peers` after one round.

**References**: issue #135; `cryptarchia-v1-bootstr-sync.md` §Initial Block Download.

### 4.1 Checklist item 3: what `cancel_active_download` does with the rest of the queue

`cancel_active_download` (`orphan_handler.rs` L297-L308) removes the active download's orphan from `pending_orphans_queue`, sets the state to `Idle`, updates the pending gauge and wakes the stream. Blocks already yielded and applied stay applied; blocks not yet yielded are lost with the stream; the other queued orphans are untouched and the next `poll_next` (L351-L367) starts a download for whichever the `HashMap` iterator returns first (L235, unordered). The cancelled orphan is **not** inserted in the cache, so it is re-enqueued the next time a child arrives by gossip (`ParentMissing` → S6, now with a parent that is not cached) or the tip poll offers it, and a new peer set is asked. On the stream arm, a stream `Err` (L419-L427) and an empty stream (L452-L464) also return to `Idle` without recording. So a malicious provider that streams garbage costs the victim one download round per attempt; the lasting damage comes only from what was *recorded* before the cancel, which is LB-001.

### 4.2 Checklist item 4: the bound and the `capacity = 0` switch

`RejectedBlocks::new(capacity)` wraps `NonZeroUsize::new(capacity).map(LruCache::new)` (`rejected_blocks.rs` L20-L24); with 0 the option is `None`, `contains_block_or_parent` returns `false` (L36-L38) and `insert` returns early (L44-L46). All seven sites go through these two methods, so capacity 0 disables every insertion and every check; `test_disabled_rejected_cache_is_noop` (`orphan_handler.rs` L725-L741) pins it. With a non-zero capacity the `lru` crate evicts the least-recently-used key on `put` once full and promotes on `get`; `contains_block_or_parent` calls `get` on both the block and the parent (L39), which is the touch-on-hit that keeps a cascade frontier resident (LB-001). The value is 1000 in every shipped configuration: the serde default (`config/cryptarchia/serde/network.rs` L76-L83), the standalone template (`standalone-node-config.yaml` L146) and the testing framework (`provisioning.rs` L827); `sync/config.rs` L27 has no serde default, so a user config that omits the field fails to parse rather than silently getting a value. Memory is 1000 × 32 B plus map overhead.

### 4.3 Ruled out

- `InvalidUncle` is a correct verdict at both S2 and S5: `body_root` is confirmed before apply on both paths, so the uncle entries are the committed ones, and every rule in `verify_uncles` reads the chain being extended plus the entry (`uncle.rs` L27-L125), matching §Uncle References' "function of the chain being extended and of the carried entry alone".
- Operation-level ledger errors are correct verdicts: `op_refs` is what `mantle_txhash` covers, so two copies with the same hash have the same operations.
- Out-of-order streaming by a provider produces `ParentMissing`, which is recoverable and not recorded; the only lasting effect is a cancelled round (§4.1).
- Reconstruction failures on the proposal path are not recorded (L628-L641), as §Reference Resolution requires.
- `Consensus(OrphanMissing)`, `Serialisation`, `InvalidBlock`, `Mempool`, `HeaderIdNotFound`, `ParentIdNotFound` cannot reach either site at this SHA.
- The mempool reconcile step after a successful apply (`lib.rs` L1030-L1060) logs and swallows its errors, so no mempool failure turns a successful apply into a recorded rejection.
- `AlreadyApplied` on the IBD path (`ibd.rs` L210-L214) is handled before the cancel, so a re-streamed known block does not abort a download.

## 5. Suggestions (non-security)

### S-001 · `ApiError::Unexpected(String)` erases the error type that the cache classification needs

`api.rs` L350 formats every non-allowlisted `chain_service::Error` into a `String`. `is_recoverable_apply_error` (`chain-network/src/lib.rs` L926-L936) therefore matches on four typed variants and treats the string as terminal, and the doc comment above it (L914-L925) states that policy as intended. Carry the error (`Box<chain_service::Error>`) in `ApiError`, and write the chain-network side as an allowlist of block-level verdicts so that a new variant is *not recorded* until someone classifies it. Every finding in this report is a consequence of the current default.

### S-002 · The stream arm does not tie streamed blocks to the download that requested them

`poll_next` yields whatever the provider sends (`orphan_handler.rs` L392-L418) and `lib.rs` L433-L473 applies it; nothing checks that the block's parent is the previous streamed block or an already-known block, or that the stream is heading toward `orphan_info.orphan_id`. Enforcing parent continuity would confine a malicious provider to blocks that extend the victim's tree and make LB-001 require a genuine block the victim actually needs, instead of any block at all.

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
