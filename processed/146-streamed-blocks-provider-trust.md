# Audit Report — Streamed blocks: what a chainsync provider can make the requester do

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/146`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `services/chain/chain-network` (orphan downloader, IBD, libp2p adapter), `consensus/cryptarchia-sync`, `services/chain/chain-service` (block provider, apply path)
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `cryptarchia-v1-bootstr-sync.md`; by section: `cryptarchia-v1-protocol.md` §Block Header Validation, §Chain Maintenance
Date: `2026-09-12` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: the requester trusts whichever provider answers first and then trusts everything it streams. The provider is chosen by a race that a peer wins by sending the cheapest possible reply, the losers' streams are discarded at that moment, and the winner's stream is consumed with no bound on its length or duration, no check that it makes progress toward the target, and no retry with another peer when it ends short. One unauthenticated connected peer can therefore stop a node from ever completing an orphan download or IBD, at the cost of one message per attempt.
- Findings: `0` critical · `1` high · `1` medium · `1` low · `1` informational
- Key themes: "first responder wins, losers dropped", "a download never times out and never has to make progress", "every apply error, transient or not, ends the download and loses the orphan", "the requester enforces no bound of its own; the only cap is the provider's"
- Must-fix before launch: LB-001 (validate the first item against the request before declaring a winner, keep the other candidates as fallbacks, retry a short or empty stream with another peer); LB-002 (a per-download deadline or progress rule).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/chain/chain-network/src/network/adapters/libp2p.rs` L72-L128, L341-L478 | first-item validation, `select_ok`, peer choice, decode of streamed blocks |
| `services/chain/chain-network/src/sync/orphan_handler.rs` L86-L117, L233-L328, L347-L473 | download state machine, continuation requests, cancel, what is lost on failure |
| `services/chain/chain-network/src/lib.rs` L433-L473, L756-L782, L845-L936, L950-L990 | orphan-download arm, future-block retry, `should_process_block`, error classification |
| `services/chain/chain-network/src/bootstrap/ibd.rs` L34-L78, L126-L255 | IBD rounds, `drain_downloader`, tip collection |
| `services/chain/chain-network/src/sync/tip_poll.rs` L27-L69 | how a peer-reported tip becomes a download target |
| `consensus/cryptarchia-sync/src/libp2p/{downloader,provider,packing,behaviour,messages}.rs`, `src/config.rs`, `src/libp2p/mod.rs` | stream framing, per-item timeout, provider send loop, message caps |
| `services/chain/chain-service/src/sync/block_provider.rs` L82-L188, L331-L436, `src/sync/config.rs` | what a provider will serve and the provider-side cap |
| `services/chain/chain-service/src/lib.rs` L425-L501, `src/uncle.rs` L27-L142, `src/api.rs` L317-L352; `ledger/src/lib.rs` L183-L208; `ledger/src/cryptarchia/mod.rs` L518-L553 | what a streamed block costs before it is rejected |
| `core/src/block/mod.rs` L29-L32, L235-L285, L337-L378 | decode-time checks and size bounds |
| `nodes/node/standalone-node-config.yaml` L73-L74, L125, L145-L149; `nodes/node/binary/src/config/network/serde/chainsync.rs` L24-L25 | shipped values |

**Out of scope**

- The rejected-block cache semantics and which errors are recorded against a block ID: report #214 (issue #143) and its follow-ups #215, #216. This report cites them and does not re-derive them.
- Gossip ingress (relay before validation, the inline future-block sleep on the proposal arm): report #142 (issue #43) LB-001, LB-002.
- IBD termination when a configured peer's tip never applies: issue #135. This report shows two more ways to reach that state but leaves the fix to #135.
- The stale `tip`/`lib` captured in `OrphanInfo`: issue #270.
- Provider-side resource use (storage scans per request, `max_inbound_requests`): parent #2's provider question, not this sub-issue.
- `libp2p-stream`, `futures::select_ok`, `tokio`, `lru`, RocksDB: assumed correct.

**Assumptions**

- The node is past `AwaitingGenesisTime`; both the `InitialBlockDownload` and `Following` phases are considered, and the report says which applies to each finding.
- Attacker model: one unprivileged peer that the victim is connected to or has discovered, holding no stake and no valid proof of leadership. Every attack below also works for a merely buggy or stale provider.
- Repo-level facts from #19 at this commit: `[profile.release]` has no `overflow-checks`; the arithmetic on these paths is `checked_*`/`saturating_*` (`orphan_handler.rs` L394-L397, `block_provider.rs` L338-L342), so no wrapping site is reported.

## 3. Method

- Manual review of the in-scope paths against `cryptarchia-v1-bootstr-sync.md` §Downloading Blocks, §Initial Block Download and §Listening for New Blocks, and `cryptarchia-v1-protocol.md` §Block Header Validation, §Chain Maintenance, working through issue #146's five items in order; parent #2 read for context.
- Every claim about control flow is tied to a line at the commit above; the existing unit tests in the same files were used as evidence of intended behaviour where they exist (`libp2p.rs` L516-L523, `orphan_handler.rs` L1078-L1109, `ibd.rs` L512-L544).
- Costs are taken from report #221 (issue #152, Appendix B table 3: 1.29 ms per Groth16 verify, 0.46 ms per proof batched at 1,000) and report #176 (issue #147: 0.83-1.01 ms per verify); not re-measured here.
- Automated tooling: none. Dynamic testing: none; the two-node reproduction is left as a follow-up item in §4 (LB-001).

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | The fastest provider wins the download, honest candidates are dropped, and an empty or aborted stream loses the orphan without retry | Denial of Service | High | Low | Open |
| LB-002 | A download has no length, byte or time bound and never has to make progress; already-applied blocks are skipped without ending it | Denial of Service | Medium | Low | Open |
| LB-003 | A streamed future-slot block runs the inline 1.5 s retry sleep on the chain-network loop | Denial of Service | Low | Low | Open |
| LB-004 | What an unrelated streamed block costs before rejection, and why every rejection ends the download | Denial of Service | Informational | — | Open |

### LB-001 · The fastest provider wins the download, honest candidates are dropped, and an empty or aborted stream loses the orphan without retry

| | |
|---|---|
| Severity | High |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `services/chain/chain-network/src/network/adapters/libp2p.rs:L82-L90` (`check_first_block_response_ready`), `:L121-L127`, `:L443-L445` (`select_ok`); `sync/orphan_handler.rs:L233-L251` (`dequeue_next_orphan`), `:L253-L281` (no retry), `:L428-L467` (stream end); `bootstrap/ibd.rs:L176-L189`, `:L206-L227` |
| Status | Open |

**Description**

Answering items 3 and 5. `request_blocks_from_peers` asks up to 16 connected and 16 discovered peers at once (`libp2p.rs` L397-L403, config L148-L149) and resolves with `select_ok` (L445): the first future to return `Ok` wins and every other future is dropped, which closes the streams the other peers had just started serving. A candidate returns `Ok` as soon as its *first item* is not an `Err` (L121-L127). Two replies pass that gate at zero cost to the sender:

- `NoMoreBlocks` as the first message. `check_first_block_response_ready(None)` is `Ok(None)` (L88), and the test at L516-L523 pins this as intended. The empty stream wins.
- One decodable block, then anything. Only the first item is inspected.

The reply that wins is the cheapest one to produce: an honest provider must go through the chain-service actor (`following.rs` L84-L106), compute a path over the engine or a RocksDB scan (`block_provider.rs` L191-L205, L552-L567) and read the first block from storage before its first frame goes out; a peer that sends a pre-built frame the moment the request arrives is ahead of every honest peer by at least that much. With the default gossipsub mesh of 6-12 (`standalone-node-config.yaml` L10-L12) the connected set is usually smaller than 16, so a connected attacker is asked on every download, not sampled.

What happens to the victim once the attacker has won:

- Empty stream: `poll_next` sees `Ready(None)` with `last_block_id == None`, logs "No blocks received for this orphan, ending sync", removes the orphan and goes `Idle` (`orphan_handler.rs` L452-L464). The orphan had already been taken out of the queue at dequeue (L236). Nothing retries with another peer: `request_blocks_stream` says so in its own comment (L259-L261), and `poll_next` has no retry path. The block the node was missing is simply forgotten.
- One block then a stream error or a garbage frame: `Ready(Some(Err))` sets `Idle` (L419-L427); same loss.
- One block then a decodable but unrelated block: the stream arm applies it, gets any error, and calls `cancel_active_download` for every error, recoverable or not (`lib.rs` L465-L471); same loss. LB-004 gives the per-case cost.

In the `Following` phase the consequence is that any block the node missed on gossip stays missing. Every later gossiped block fails with `ParentMissing`, is enqueued (`lib.rs` L660-L672), becomes the next download target, and the race is run again, once per gossiped block, with the same winner; the tip-poll watchdog (three block intervals of lag by default, `sync/config.rs` L50-L56) adds one more attempt per poll through `lib.rs` L783 with the same outcome. The node follows the chain only as long as it never misses a block, and once it has missed one it never catches up while the attacker is connected.

In `InitialBlockDownload` the consequence is worse: `drain_downloader` returns when the downloader goes `Idle` (`ibd.rs` L206-L227), the round sleeps `round_delay`, `collect_unsynced_tips` re-enqueues the same tip (L177-L183) and the race is run again, forever. IBD neither completes nor fails: the spec's "terminated with an error, allowing the operator to restart the node with other IBD peers" (§Initial Block Download) is never reached because `AllPeersFailed` only covers tip fetching (L233-L255). This is the same terminal state as #135 reached from a different entry point, and it needs no misconfigured IBD peer: block downloads "are not restricted to the configured IBD peer set" (L87-L89), so any connected or discovered peer can be the winner.

The spec's `download_blocks` returns on the first exception and §Listening for New Blocks says "If the request fails, the node may retry with different peers before abandoning the orphan block"; the retry is optional in the spec and absent in the code (S-003).

**Exploit scenario**

1. The attacker connects to the victim (any peer in the mesh will do) and answers every `DownloadBlocksRequest` with a single `NoMoreBlocks` frame (`messages.rs` L131-L139; 5 bytes plus the length prefix).
2. The victim misses one gossiped block, or is bootstrapping. Its next download fans out to its connected peers; the attacker's frame arrives first; the honest streams are dropped; the orphan is discarded.
3. In `Following`, the victim's tip freezes; every subsequent gossiped block is an orphan and step 2 repeats for each of them, as it does for every tip poll. In IBD, step 2 repeats every `round_delay` with the same tip.

Cost to the attacker: one connection and one frame per download. Cost to the victim: it cannot synchronise while the attacker is connected. No crash, no state corruption; a restart does not help while the peer is still there. Variant: instead of an empty stream, send one genuine block followed by one self-signed block with an unknown parent (LB-004 case A); same outcome, and the victim does one decode more. Not reproduced on two nodes here; the reproduction belongs with #216's fix.

**Recommendation**

- *Short term*: in `request_available_blocks_stream_from_peer`, treat an empty first response as a failure of that candidate (`Ok(None)` → `Err`), and require the first block's parent to be one of the `known_blocks` the request named or a block already in the tree before returning `Ok`. Replace `select_ok` with a selection that keeps the runners-up: on the winner's stream ending before the target or erroring, continue from `last_block_id` with the next candidate instead of going `Idle`, bounded by the number of candidates. In `poll_next`, do not drop the orphan on `Ready(None)` with no blocks received; re-queue it with a per-orphan attempt counter (a small constant, e.g. 3) and only then discard it.
- *Long term*: give the downloader a provider-fault record (issue #216 items 1 and 4) so that a peer whose stream ended short, erred, or delivered a non-descendant is not asked again for that target; make IBD count a tip whose download failed N times as a failed peer and apply the spec's "succeed if at least one configured peer's tip was reached, otherwise terminate" rule (#135). Add a unit test in `orphan_handler.rs` that an empty first stream is followed by a request to a second peer, and one in `ibd.rs` that a peer answering `NoMoreBlocks` forever makes IBD fail rather than loop.

**References**: `cryptarchia-v1-bootstr-sync.md` §Downloading Blocks (`download_blocks` pseudocode, `except: return`), §Listening for New Blocks (retry "may"), §Initial Block Download (termination rule); issues #135, #216; report #214 (issue #143) LB-001 preconditions and S-002.

### LB-002 · A download has no length, byte or time bound and never has to make progress; already-applied blocks are skipped without ending it

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `consensus/cryptarchia-sync/src/libp2p/downloader.rs:L95-L144` (`receive_blocks`), `src/libp2p/packing.rs:L58-L76`, `src/libp2p/mod.rs:L9`; `services/chain/chain-network/src/sync/orphan_handler.rs:L86-L117`, `:L392-L418`; `src/lib.rs:L440-L453`; `src/bootstrap/ibd.rs:L206-L214`; `services/chain/chain-service/src/sync/block_provider.rs:L338-L342` |
| Status | Open |

**Description**

Answering item 1. The only cap on a `DownloadBlocksResponse` is the provider's: `batch_size` (default 1,000, `standalone-node-config.yaml` L125) is applied when the *serving* node computes its path (`block_provider.rs` L338-L342, L376-L378). The requester enforces nothing of its own:

- `receive_blocks` is an unbounded `try_unfold` over the libp2p stream (`downloader.rs` L107-L133). Each frame is capped at `MAX_MSG_LEN = 16 MiB` (`packing.rs` L67-L72, `mod.rs` L9) and each item is waited for with `peer_response_timeout` (5 s, config L73) — a *per-item* timeout, not a per-stream one. A provider that sends one frame every 4.9 s never trips it.
- `ActiveDownload` counts `total_blocks_received` and records `download_started_at` (`orphan_handler.rs` L93-L96) but uses them only for logs and metrics (L399-L410); nothing ends a download because it is long or old.
- Nothing requires the stream to advance. A block whose ID is already applied is skipped and the download continues: `should_process_block` returns `AlreadyApplied` and the arm does `continue` (`lib.rs` L452); in IBD `process_block`'s `AlreadyApplied` is "continuing" (`ibd.rs` L210-L214). This is deliberate, so that a stream starting below the local tip works (`ibd.rs` L571-L628 tests it), but it also means a provider can send the same already-applied block, or the same genuine ancestor, forever, and the requester will decode and skip every copy without ending the download.

Because the orphan downloader is a single-slot state machine (`DownloaderState`, `orphan_handler.rs` L24-L31), the one download in flight is the only one: every other queued orphan waits (queue cap 1,000, L199-L212), and in IBD `drain_downloader` loops on `should_poll()` while the state is `Downloading` (`ibd.rs` L206). Memory does not grow (items are consumed one at a time), but the sync pipeline is held for as long as the provider likes.

What each looped item costs the victim: `Block::try_from` (`libp2p.rs` L373-L377 → `core/src/block/mod.rs` L235-L249: one Ed25519 verification, the transaction-size sum, and a `body_root` over up to 1,024 transactions of up to 2 MiB total) plus, in `Following`, the two chain-service round trips of `should_process_block` (`lib.rs` L855-L866), all on the chain-network task. At one 2 MiB block per item that is a few milliseconds of hashing and deserialisation per frame that the provider can repeat at line rate.

The spec bounds the *responder* ("the peer limits the number of blocks to be returned", §Downloading Blocks) and says nothing about the requester; the requester's `download_blocks` pseudocode reads "until the stream returns NoMoreBlock". A requester that inherits its only bound from the peer it is defending against has no bound.

**Exploit scenario**

The attacker wins one download (LB-001) with a genuine first block, then either (a) sends one genuine, already-applied block every 4.9 s, or (b) re-sends the same 2 MiB already-applied block as fast as the link allows. In (a) the victim's orphan downloader is occupied indefinitely at no CPU cost; in (b) it additionally spends a decode per frame. In `Following` the queue behind it fills with every orphan the node meets and, at 1,000 entries, new orphans are dropped (`orphan_handler.rs` L199-L212, `QueueFull`). In `InitialBlockDownload` the node never leaves IBD. Recovery requires the attacker to stop or the node to restart. Cost to the attacker: one connection, one frame per 4.9 s, or bandwidth.

**Recommendation**

- *Short term*: in `receive_blocks` or in `ActiveDownload`, add a per-download deadline (e.g. `batch_size × peer_response_timeout` with a low absolute cap) and a per-download item cap equal to the local `batch_size + 1`, and end the stream as a provider fault when either is hit. Count consecutive `AlreadyApplied` items per download and end the download after a small number (the honest case is a prefix of a few overlapping blocks, not an unbounded run); on the stream arm, require that each applied or skipped block's parent is the previous streamed block or a block in the tree (issue #216 item 1), which also rules out a repeated block.
- *Long term*: move the bounds into the requester's `DownloadBlocksRequest` handling so they are independent of the provider's configuration, expose them as `SyncConfig` fields, and add a test that a stream of `N > batch_size` items is cut at the bound with the provider marked.

**References**: `cryptarchia-v1-bootstr-sync.md` §Downloading Blocks; issue #216 items 1 and 4; issue #135.

### LB-003 · A streamed future-slot block runs the inline 1.5 s retry sleep on the chain-network loop

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `services/chain/chain-network/src/lib.rs:L78-L79`, `:L457-L471`, `:L756-L782`, `:L977-L990`; `services/chain/chain-service/src/lib.rs:L442-L448`; `services/chain/chain-network/src/bootstrap/ibd.rs:L65-L73` |
| Status | Open |

**Description**

Answering item 4. The orphan-download arm applies every streamed block through `apply_block_with_future_block_retry` (`lib.rs` L457), the same wrapper as the proposal arm: three attempts with a 500 ms `sleep` between them, awaited inline inside the `select!` (L983). The future-slot check is the second thing chain-service does, before any parent or proof check (`chain-service/src/lib.rs` L442-L448), so a provider needs only a decodable header with `slot > current_slot`: a self-signed header, empty body, constant `body_root` (`core/src/block/mod.rs` L235-L249). Report #142 LB-001 established the stall for the gossip path; the stream path reaches it without gossipsub in between, from a direct 1:1 stream, and the first such block also ends the download (L465-L471, `FutureBlock` is recoverable so nothing is recorded, but `cancel_active_download` runs regardless).

The stall is therefore bounded at 1.5 s per download attempt rather than per block: a provider cannot chain future blocks inside one stream because the first one aborts it. Combined with LB-001 it is 1.5 s per re-run of the race, which recurs every `round_delay` in IBD and on every gossiped block in `Following`. During the sleep the loop polls nothing else (proposals, chainsync provide requests, polled tips), as in #142 LB-001.

The IBD path does not have this: `ChainNetworkIbdBlockProcessor::process_block` calls `apply_block_and_reconcile_mempool` directly (`ibd.rs` L65-L73), so a future block there is an immediate error and a cancel.

**Exploit scenario**

The attacker wins a download (LB-001) with one genuine block and follows it with a self-signed header at `slot = current + 10^6`. The victim decodes it, sends it to chain-service, gets `FutureBlock`, sleeps 500 ms, repeats twice, then aborts the download. 1.5 s of chain-network wall-clock per attempt, on top of LB-001's effect. Cost to the attacker: one Ed25519 signature.

**Recommendation**

- *Short term*: do not wrap the stream arm in the retry: a block streamed as an *ancestor* of a target the node already holds (or as a peer's *current* tip) is never legitimately in the future, so treat `FutureBlock` on the stream arm as a provider fault and end the download without sleeping. This is one line at `lib.rs` L457 (call `apply_block_and_reconcile_mempool` directly, as IBD does).
- *Long term*: the fix in report #142 LB-001 (no sleep on the ingress loop; park near-future blocks in a slot-keyed map) covers both arms.

**References**: report #142 (issue #43) LB-001; `cryptarchia-v1-protocol.md` §Block Header Validation rule 6.

### LB-004 · What an unrelated streamed block costs before rejection, and why every rejection ends the download

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Denial of Service |
| Target | `services/chain/chain-network/src/lib.rs:L440-L473`, `:L953-L963`; `services/chain/chain-service/src/lib.rs:L438-L478`; `src/uncle.rs:L27-L41`, `:L128-L142`; `ledger/src/lib.rs:L198-L205`; `ledger/src/cryptarchia/mod.rs:L518-L553`; `services/chain/chain-service/src/api.rs:L338-L351` |
| Status | Open |

**Description**

Answering item 2 (the continuity check itself is issue #216 and report #214 S-002; this entry records the cost side, which neither states). A provider may stream any block; `should_process_block` (`lib.rs` L855-L866) only rejects blocks at or before the LIB slot and blocks already applied. Everything else is decoded (one Ed25519 verify, size sum, `body_root`) and handed to chain-service. The order there is: `AlreadyApplied`, `FutureBlock`, `verify_uncles`, `prepare_update` (parent lookup, then epoch-state synthesis, then the PoL Groth16 verify, then the transactions), then the batched transaction proofs (`chain-service/src/lib.rs` L438-L478). Per case, at report #221's figures (1.29 ms per Groth16 verify, 0.46 ms per proof batched at 1,000):

| Case | Where it fails | Victim's cost beyond decode | Recorded against the streamed ID? | Download |
|---|---|---|---|---|
| A. Parent unknown, no uncles | `prepare_update` → `ParentNotFound` (`ledger/src/lib.rs` L198-L201) | two `rpds` lookups | no (`ParentMissing` is recoverable, `lib.rs` L953-L963) | cancelled (L470) |
| B. Parent unknown, with uncles | `verify_uncles` → `ParentMissing` (`uncle.rs` L34-L41) | one lookup | no | cancelled |
| C. Parent known, bogus PoL, no uncles | `try_apply_proof` → `InvalidProof` (`cryptarchia/mod.rs` L551-L552) | epoch-state synthesis (about 100 ns same-epoch, up to 1.3 ms at an epoch boundary with 10,000 declarations, per report #176) + 1 Groth16 | yes, as `ApiError::Unexpected` (`api.rs` L350) | cancelled |
| D. Parent known, uncles whose parents lie on the chain within the window, bogus uncle PoL | `verify_uncle_pol` → `InvalidUncle` (`uncle.rs` L128-L142) | the walk-back over the window + 1 Groth16 for the first bad uncle (the loop stops at the first failure, L66-L75) | yes | cancelled |
| E. Parent known, valid PoL (attacker holds a winning aged note), up to 1,024 copied transactions with mangled proofs | `verify_batch_proofs` → `BatchZkpVerification` | 1 Groth16 + ledger application of the transactions + one batched multi-pairing over up to 1,024 proofs, about 0.5 s (report #221 S-002), inline on the chain-service actor | yes, and the ID is the *genuine* block's if the header is genuine: report #214 LB-001 | cancelled |

Two things follow. First, without stake the worst a provider can impose per streamed block is cases C/D, about 1-3 ms plus the decode; case E needs a valid proof of leadership and is bounded by the batch (and is the subject of #214 LB-001 for its verdict, not its cost). The cost side of item 2 is therefore modest at this commit; the damage is in the *outcome*, not the CPU. Second, the outcome is the same in every row: `cancel_active_download` (`lib.rs` L470) runs for recoverable and terminal errors alike, and the orphan is already out of the queue (`orphan_handler.rs` L236), so one unrelated block ends the download and forgets the orphan. `cancel_active_download` itself records nothing about the provider (L297-L308). In IBD the same holds (`ibd.rs` L215-L218) and the tip is retried next round with the same peers.

Item 5's `Error::Storage` mid-stream: on the *provider* side a storage miss ends the path early through `take_while` (`block_provider.rs` L185) and the stream closes with `NoMoreBlocks`, so the requester sees a short stream and re-requests from `last_block_id`; the next path hits the same missing block at once and comes back empty, which is LB-001's empty-stream exit (orphan dropped; in IBD, retried forever). On the *requester* side a chain-service `Error::Storage` (`chain-service/src/lib.rs` L112-L113) reaches chain-network as `ApiError::Unexpected` (`api.rs` L350), is terminal for the rejected cache (report #214 LB-003) and cancels the download.

**Recommendation**

The continuity rule of #216 (apply a streamed block only if its parent is the previous streamed block or in the tree; otherwise it is a provider fault, not a block verdict) removes rows A-D from the apply path entirely and makes row E attributable to the provider. Until then, at least stop cancelling the download on recoverable errors (`lib.rs` L465-L471): skip the item as `should_process_block` already does for `AlreadyApplied`, and let the download continue.

**References**: issue #216; report #214 (issue #143) LB-001, LB-003, S-002; report #221 (issue #152) S-002, Appendix B table 3; report #176 (issue #147).

## 5. Suggestions (non-security)

### S-001 · One download contract for both consumers

`OrphanBlocksDownloader` is driven by two loops with different error handling: `lib.rs` L460-L473 (retry wrapper, rejected-cache insert on terminal errors, cancel on every error) and `ibd.rs` L206-L227 (no retry wrapper, no cache, cancel on every error but `AlreadyApplied`). LB-001 to LB-004 have to be fixed twice. Put "what to do with a streamed block and its error" into the downloader (or a shared `apply_streamed_block` helper) so the IBD and `Following` paths cannot diverge again; the IBD test at `ibd.rs` L512-L544 already documents one such divergence.

### S-002 · `select_ok` is the wrong primitive for "first *validated* response"

`select_ok` returns the first `Ok` and drops the rest. What the adapter needs is "the first candidate whose first block passes a check, keeping the others as fallbacks". `futures::stream::FuturesUnordered` over the candidates, taking the first that passes and holding the remainder for continuation, gives that with no new dependency. The comment at `libp2p.rs` L444 ("First peer with a validated first response wins") describes the intent; the validation is currently "is not an `Err`".

### S-003 · Spec: make the requester's obligations normative

`cryptarchia-v1-bootstr-sync.md` §Downloading Blocks bounds the responder and leaves the requester with "read the stream until NoMoreBlock"; §Listening for New Blocks says the node "may retry with different peers" and that "the retry policy can be configured by implementers". Two of the three findings above are exactly the requester doing what the pseudocode says. Suggested additions, to raise in logos-lips: the requester MUST bound the number of blocks and the wall-clock it accepts per request independently of the responder; MUST treat a stream whose first block does not extend one of its `known_blocks` (or whose later block does not extend the previous one) as a failed request from that peer; and SHOULD retry a failed or empty request with a different peer before abandoning the target, with the number of attempts a protocol parameter rather than an implementer choice.

---

## Appendix A — Definitions

Ratings follow `docs/REPORT_TEMPLATE.md` Appendix A.
