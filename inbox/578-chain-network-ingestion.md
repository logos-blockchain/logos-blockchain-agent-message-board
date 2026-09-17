# Audit Report — Chain-network proposal ingestion and recovery

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/578`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `3d5d419ec85c7b23e4d7e1455bd1cde845264a9e` — component(s): `services/chain/chain-network`, `services/network`, `consensus/cryptarchia-sync`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `docs/blockchain/raw/cryptarchia-v1-bootstr-sync.md`, `docs/blockchain/raw/fork-choice.md`; consulted by section: `docs/blockchain/raw/cryptarchia-v1-protocol.md` (Block Chain, Block Header Validation, Chain Maintenance, Commit, Fork Pruning)
Date: `2026-09-17` — author: `Codex` — status: `draft`

---

## 1. Summary

- Overall assessment: The chain-network loop still processes each gossiped proposal inline, so proposal reconstruction or future-slot retries stop all other ingress; the resulting 64-slot broadcast overruns and recovery behavior are already covered by existing canonical findings.
- Findings: `0` new findings; existing `LB-001` from #276, `43-LB-001` from #439, and `146-LB-001` from #550 remain applicable.
- Key themes: serialized ingress; lossy bounded subscriptions; indirect tip-poll recovery; orphan-download retry boundaries.
- Must-fix before launch: the existing canonical findings cover the required fixes; this re-verification introduces no additional must-fix.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/chain/chain-network/src/lib.rs` | Main `tokio::select!` loop, proposal reconstruction/application, future-slot retry, tip-poll handoff, and proposal observation. |
| `services/chain/chain-network/src/network/adapters/libp2p.rs` | Proposal and ChainSync subscriptions, lag behavior, and parallel block-download requests. |
| `services/chain/chain-network/src/sync/orphan_handler.rs` | Orphan queue bounds, download state transitions, and failed-stream handling. |
| `services/chain/chain-network/src/sync/tip_poll.rs`, `sync/config.rs` | Lag threshold, cadence, peer sampling, and polled-tip selection. |
| `services/network/src/backends/libp2p/mod.rs`, `swarm/gossipsub.rs`, `swarm/chainsync.rs` | Shared 64-slot pubsub/ChainSync channels and event production. |
| `consensus/cryptarchia-sync/src/libp2p` | Provider/requester response timeout behavior relevant to dropped ChainSync events. |
| Deployment and block-size configuration | Reference slot duration, active-slot coefficient, and proposal size limits used for bounds. |

**Out of scope**

The audit did not inspect unrelated consensus execution, ledger correctness, or cryptographic validity beyond the checks needed to establish proposal-processing cost. No source changes, devnet, e2e run, fuzzing, or benchmark was performed. Third-party dependencies, including `tokio`, `libp2p`, `lru`, `backon`, and `overwatch`, were assumed correct for their documented channel, timeout, and stream semantics.

**Assumptions**

The pinned Cryptarchia specifications are authoritative. A proposal that is dropped before `handle_incoming_proposal` is not recoverable by the node unless the corresponding block is later available from another peer. The deployment reference uses a one-second slot duration and `f = 1/20`; configuration can change the resulting timing.

## 3. Method

- Manual review of issue `#578`, parent issue `#3`, the prior report for #276, and the canonical related findings #439 and #550.
- Spec conformance review against the pinned bootstrapping/synchronization and fork-choice specifications, plus the relevant Cryptarchia block-validation and chain-maintenance sections.
- Automated tooling: none.
- Dynamic testing: none. Focused exact-target unit tests were run with the following results:
  - `CARGO_TARGET_DIR=/tmp/logos-audit-578-target cargo test -p logos-blockchain-chain-network-service --lib retry_future_block_apply` — `3 passed; 0 failed`.
  - `CARGO_TARGET_DIR=/tmp/logos-audit-578-target cargo test -p logos-blockchain-chain-network-service --lib sync::` — `22 passed; 0 failed`.

## 4. Findings

No new finding IDs are assigned. The existing canonical findings were independently re-verified at the target revision:

| Existing ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| `276-LB-001` | Upstream lag drops observations and can make a delivered payload appear lost | Privacy / Anonymity | High | Medium | Open; canonical report for #276 |
| `43-LB-001` | Future-slot proposals stall the chain-network event loop for 1.5 s each | Denial of Service | High | Low | Open; canonical report for #43 |
| `146-LB-001` | A failed or empty orphan download loses the orphan without retry | Denial of Service | High | Low | Open; canonical report for #146 |

These ratings were checked against the current report-template definitions. The #578 follow-up is a re-verification and does not create duplicate `LB-NNN` identifiers.

### Re-verification evidence and checklist answers

**1. Stall length and honest trigger conditions.**

The proposal arm at `services/chain/chain-network/src/lib.rs:407-414` calls `handle_incoming_proposal(...).await` before the loop can poll any other arm. That handler first performs chain-service checks, verifies the header, reconstructs the block by resolving references sequentially against the mempool (`:571-645`, `:1079-1140`), and then applies the block (`:695-755`). The source permits up to 1024 transaction references and a 2 MiB transaction-body limit; there is no wall-clock bound around reconstruction, chain-service application, or mempool reconciliation. Therefore a full block does not have a source-derived millisecond duration: it stalls the loop for however long those downstream calls take. This was not measured on a devnet.

The future-slot case has a deterministic added delay. `FUTURE_BLOCK_MAX_RETRIES` is `3` and `FUTURE_BLOCK_RETRY_DELAY` is `500 ms` (`lib.rs:78-79`); the retry helper sleeps after each of the first three `FutureBlock` responses (`:950-990`). It can therefore add up to `1.5 s`, plus the application-attempt overhead. The three exact-target retry tests all pass.

While the loop is stalled, the network service publishes every gossipsub event into one `broadcast::channel(64)` (`services/network/src/backends/libp2p/mod.rs:35-49`). The chain-network proposal subscription turns `BroadcastStreamRecvError::Lagged(n)` into a log and `None` (`services/chain/chain-network/src/network/adapters/libp2p.rs:220-246`), so the overwritten events are lost. The chain-network-to-observer proposal channel is another capacity-64 broadcast channel (`lib.rs:80, 259`), and `note_received_proposal` is called only after a proposal has survived the first subscription (`:407-408, 798-800`). Thus 65 unread events are enough to overrun either channel; they need not all be valid proposals because the upstream channel is shared across topics.

**2. Recovery of a proposal lost at the lag point.**

A proposal lost by `proposals_stream` never reaches `handle_incoming_proposal`, so it is not reconstructed, applied, or enqueued in `OrphanBlocksDownloader`. The orphan downloader only receives a block after a proposal was successfully processed far enough to produce `ParentMissing` (`lib.rs:655-675`). The tip poll is an indirect recovery mechanism only: with default settings it checks the local tip every `ceil(1/f)=20` slots, triggers once the tip is more than `3 × 20 = 60` slots behind, samples up to five peers, and enqueues the most advanced reported tip (`sync/config.rs:30-57`, `sync/tip_poll.rs:27-65, 131-178`). With the reference one-second slots this is roughly a three-block-interval detection delay plus up to one cadence interval, followed by peer sampling and block download; it is not a fixed recovery latency.

Tip polling can recover a canonical block that another peer already accepted, because the orphan downloader then fetches full blocks. It cannot recover a dropped proposal that no peer turned into a canonical descendant, and it does nothing when disabled, when no sampled peer reports a higher tip, or when the lag threshold is not reached. There is no proposal-ID replay request. A restart/IBD is a later full-sync opportunity, not an immediate recovery path.

The orphan downloader itself has a bounded 1000-entry default queue and passes its request to multiple peers. However, if every candidate request fails, the dequeued orphan is removed and the `Requesting` error path returns to `Idle` without re-enqueuing it (`sync/orphan_handler.rs:233-251, 347-381`). The same loss on empty or aborted streams is covered by canonical `146-LB-001` and was not assigned a second ID here.

**3. Moving the future wait or draining into a local queue.**

Moving the future-slot retry out of the `select!` body would remove the known event-loop stall, but it needs a bounded, slot-aware pending queue to avoid unbounded memory and to preserve parent-before-child processing. An alternative is a dedicated ingress task that continuously drains the network subscription into a bounded queue; that protects the main loop only if the queue has an explicit overflow policy. It does not repair the upstream network-service `broadcast(64)` loss unless the network service itself provides a dedicated queue or an explicit lag marker. These are recommendations already captured by `43-LB-001` and `276-LB-001`.

**4. ChainSync has the same starvation shape, with a different immediate effect.**

`chainsync_events.next()` is the second arm at `lib.rs:417-421`. It is not polled while proposal handling is awaiting. The network backend uses a separate capacity-64 `chainsync_events_tx` (`services/network/src/backends/libp2p/mod.rs:48-49`), and `chainsync_events_stream` filters lag errors to `None` after logging (`services/chain/chain-network/src/network/adapters/libp2p.rs:248-258`). A dropped `ProvideBlocksRequest` or `ProvideTipRequest` therefore receives no response. The chain-sync configuration gives a requester a five-second peer-response timeout (`services/network/src/backends/libp2p/swarm/mod.rs:411-414`); block downloads fan out to candidate peers with `select_ok` (`chain-network/src/network/adapters/libp2p.rs:397-445`), so another responsive provider often masks one dropped event. If all candidates fail, the orphan loss is the already-tracked #550 behavior. Tip-poll samples similarly ignore failed peer responses and return no catch-up tip when none succeeds. This is a confirmed secondary manifestation of the existing starvation/recovery findings, not a distinct new root cause in this iteration.

**5. Items checked and ruled out.**

- The rejected-block cache is bounded LRU state and only records terminal/older-than-LIB outcomes; transient `ParentMissing`, `FutureBlock`, `AlreadyApplied`, and communication errors are intentionally not poisoned into that cache (`lib.rs:845-930`, `sync/rejected_blocks.rs:1-55`).
- Orphan descendants are processed through the downloader’s streamed block path, and the focused sync suite passed all 22 tests. The unresolved problem is failed-download retention/retry, already recorded as #550.
- Fork-choice and chain-maintenance rules require valid blocks to be applied parent-to-child and do not provide a replay mechanism for a proposal that was discarded before validation.

## 5. Suggestions

### S-001 · Add a bounded ingress/recovery regression harness

Add a deterministic test or benchmark that holds `handle_incoming_proposal` for a controlled duration while injecting 65+ pubsub and ChainSync events. Assert which events are lost, that lag is surfaced to the relevant observer, and that a failed orphan download remains retryable. This would quantify implementation-specific full-block costs without conflating them with the fixed 1.5-second future-slot delay.

**References**: issue #578; existing reports #276, #439, and #550; `cryptarchia-v1-bootstr-sync.md` sections “Listening for New Blocks”, “Downloading Blocks”, and “Initial Block Download”; `cryptarchia-v1-protocol.md` sections “Block Header Validation” and “Chain Maintenance”.
