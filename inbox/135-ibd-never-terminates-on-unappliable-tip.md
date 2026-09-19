# Audit Report — Initial Block Download never terminates when one configured peer's tip cannot be reached

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/135`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `9ffddb30b9e6cf79465802953caedd010ff1cecd` — component(s): `services/chain/chain-network` (`bootstrap/ibd.rs`, `sync/orphan_handler.rs`, `sync/rejected_blocks.rs`, `lib.rs`, `network/adapters/libp2p.rs`), `services/chain/chain-service` (`service/phases/*`, `service/mod.rs`, `lib.rs`, `sync/block_provider.rs`), `consensus/cryptarchia-engine`, `services/chain/chain-leader`, `nodes/node/binary` (IBD configuration)
Specs: `https://github.com/logos-co/logos-lips` @ `75d3d0382604d4a0d8e246c268935dd386ffc8ea` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `cryptarchia-v1-bootstr-sync.md` (all three in full)
Date: `2026-09-19` — author: `Claude Fable 5.1 (research agent)` — status: `final`

---

## 1. Summary

- Overall assessment: the defect the issue describes is still present at `9ffddb30b`, in a rewritten IBD. IBD completes only when the tip of *every* configured peer that answers is in the local tree, and nothing bounds how long one tip may stay unreachable. One peer is enough to hold a node in IBD for good. In an experiment on the node's own IBD test fixture, one good peer plus one peer with an unappliable tip kept IBD running for the whole simulated hour (3,600 rounds); the good peer's chain was fully applied after the first round. A second, new trigger needs no faulty peer at all: a configured peer that is merely *behind* the local node's latest immutable block (LIB) has the same effect, because the "do I have this tip" check reads a map that forgets blocks below the LIB.
- Findings: 0 critical · 0 high · 1 medium · 2 low · 0 informational
- Key themes: "IBD waits for all peers where the spec asks for one", "a tip the node can never apply is retried once a second, forever", "`has` is answered from a pruned map", "nodes that are still bootstrapping do not answer tip requests".
- Must-fix before launch: LB-001. The default configuration makes every initial peer an IBD peer, so one stale bootstrap node stalls every node that restarts.

Answers to the four checklist items, in order:

1. **No per-tip retry limit or time bound exists, and the rejected-blocks cache never removes a bad tip.** The loop at `bootstrap/ibd.rs:174-186` has one exit on success (`:176-179`) and one on error, `AllPeersFailed`, which only tip *fetching* can raise (`:237-241`, `:314-316`). The IBD code never calls `insert_rejected_block`; the only callers are in `lib.rs` (`:465`, `:484`, `:609`, `:694`, `:906`) and inside the downloader itself. Even if IBD did record the tip, `collect_unsynced_tips` (`:231-253`) does not consult the cache, so the tip would still count as unsynced and the loop would still not exit. The in-tree test comment still says "no per-tip retry limit yet" (`:510-516`).
2. **Rating: Medium. One such peer costs the node its whole service life until an operator edits the configuration**; it is not bounded. The node keeps following the chain by polling, but it never leaves the `InitialBlockDownload` phase, so it never starts the Prolonged Bootstrap Period, never switches to the Online rule, never proposes, and never serves sync requests (LB-001).
3. **A peer that is itself bootstrapping is treated as failed for that round only, and is asked again every round.** It never blocks IBD while another peer answers. If every configured peer is in that state, the node terminates: 4 tip requests per peer and 2.875 s of virtual time in my runs (LB-003).
4. **Fix shape**: count failed rounds per *peer* (not per tip), drop a peer after a small number, and finish with success if at least one peer's tip was reached, otherwise `AllPeersFailed`. A prototype of this passes the crate's 45 existing unit tests and ends both stuck experiments in 3 rounds (LB-001, Recommendation). A cap keyed by tip would not be enough: a peer that reports a new unknown tip each round would reset it.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/chain/chain-network/src/bootstrap/ibd.rs` (whole file), `bootstrap/config.rs` | the IBD loop, tip collection, tip-fetch retry, the drain loop, the in-tree tests |
| `services/chain/chain-network/src/sync/orphan_handler.rs` L1-L498, `sync/rejected_blocks.rs` (whole file) | the downloader IBD drives, its failure paths, the rejected cache |
| `services/chain/chain-network/src/lib.rs` L283-L375, L433-L435 | how the service runs IBD, what it does on success and on error, when it subscribes to proposals and sync events |
| `services/chain/chain-network/src/network/adapters/libp2p.rs` L384-L444 | which peers a block download is sent to |
| `services/chain/chain-service/src/service/phases/ibd.rs`, `pbp.rs`, `awaiting_genesis_time.rs`, `following.rs` L60-L90; `service/mod.rs` L1254-L1280 | what the chain service does in each phase, and which phases answer sync requests |
| `services/chain/chain-service/src/lib.rs` L425-L501, L528-L552, L576-L578 | `AlreadyApplied`, `ParentMissing`, ledger-state pruning |
| `services/chain/chain-service/src/sync/block_provider.rs` L82-L452 | what a provider answers for a target below the requester's LIB (read, not run) |
| `consensus/cryptarchia-engine/src/lib.rs` L31-L75, L228-L268, L437-L531, L639-L649 | LIB under the two fork-choice rules, pruning of immutable blocks, `ParentMissing` |
| `consensus/cryptarchia-sync/src/libp2p/provider.rs` L36-L62, `src/messages.rs` L14-L24 | the tip response and its failure form |
| `services/chain/chain-leader/src/lib.rs` L420-L444 | what waits for the chain to become online |
| `nodes/node/binary/src/cli/config/init.rs` L159-L185, `cli/config/update.rs` L132-L150, `config/cryptarchia/serde/network.rs` L20-L123, `config/cryptarchia/serde/service.rs` L41-L49, `config/cryptarchia/mod.rs` L112-L148, `nodes/node/standalone-node-config.yaml` L118-L149, `config/deployment/settings.yaml` L19 | where IBD peers come from, the shipped defaults, the sync protocol name |

**Out of scope**

- The libp2p stream protocol below the adapter (`consensus/cryptarchia-sync/src/libp2p/downloader.rs`, `behaviour.rs`) apart from the two excerpts above; `libp2p`, `backon`, `lru`, `tokio` and `rocksdb` are assumed correct.
- Block and proof validation itself. The experiments use the fixture's mock block processor, which runs the real `lb_cryptarchia_engine` but no ledger and no proofs.
- The orphan path after IBD, the tip-poll watchdog (`sync/tip_poll.rs`), and the races between download providers, which the #146 report covers.
- No multi-node network was run. Every dynamic result below comes from unit-test fixtures with virtual time.

**Assumptions**

- IBD peers are chosen by the operator or by the `config init` default, and are not authenticated beyond their libp2p peer ID.
- Repo-level facts from issue #19: I read the issue but did not re-verify its items at this commit. The findings here are about control flow and do not rest on any of them; the default values that matter are quoted where used.

## 3. Method

- Manual review of the in-scope paths, working through issue #135 (parent #2), after reading `cryptarchia-v1-bootstr-sync.md` in full. I did not consult `cryptarchia-v1-protocol.md`; the one fact I needed from Chain Maintenance (a block whose parent is unknown cannot be added) is restated in the bootstrapping spec's `listen_and_process_new_blocks`.
- Spec conformance of the IBD loop against §Initial Block Download and the `download_blocks` pseudocode of §Downloading Blocks.
- Prior reports read for overlap: #143 (`processed/143-rejected-cache-insertion-sites.md`, LB-004 defers the liveness gap to this issue), #146 (`processed/146-streamed-blocks-provider-trust.md`, LB-001 reaches the same endless loop from a provider race), #42.
- Automated tooling: `cargo test` with `rustc 1.94.1` / `cargo 1.94.1`. Baseline: `cargo test -p logos-blockchain-chain-network-service --lib -- bootstrap::ibd` on an unmodified copy of the tree: 10 passed, 0 failed.
- Dynamic testing: four experiment tests added to the `tests` module of `bootstrap/ibd.rs` in a scratch copy of the tree (the audited checkout was not modified). They reuse the fixture's `MockBlockProcessor`, `BlockProvider` and `MockNetworkAdapter`, with two counters added (download requests, successfully applied blocks), `max_rejected_cache_size: 1000` as shipped, and `#[tokio::test(start_paused = true)]` so that one hour of IBD runs in seconds. Each test prints its outcome; the lines are quoted verbatim in the findings. The experiments were run first on the unmodified IBD logic, then again on the prototype fix. With the prototype the full crate suite gives `49 passed; 0 failed` (45 existing tests plus the four experiments).

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Spec deviation: IBD requires every answering peer's tip and retries an unreachable tip forever | Denial of Service | Medium | High | Open |
| LB-002 | A configured peer whose tip is older than the local LIB keeps IBD running forever, because the tip check reads the pruned ledger-state map | Denial of Service | Low | High | Open |
| LB-003 | Nodes that are still in IBD or in the Prolonged Bootstrap Period do not answer tip requests, so nodes that list only each other cannot start together | Configuration | Low | High | Open |

### LB-001 · Spec deviation: IBD requires every answering peer's tip and retries an unreachable tip forever

| | |
|---|---|
| Severity | Medium |
| Difficulty | High |
| Category | Denial of Service |
| Target | `services/chain/chain-network/src/bootstrap/ibd.rs:L159-L187` (`download_blocks`), `:L192-L226` (`drain_downloader`), `:L231-L253` (`collect_unsynced_tips`) |
| Status | Open |

**Description**

Each IBD round fetches the tip of every configured peer, keeps the tips that are not in the local tree, hands them to an `OrphanBlocksDownloader`, drains it, sleeps `round_delay`, and starts again:

```rust
// bootstrap/ibd.rs:174-186
loop {
    let unsynced_tips = self.collect_unsynced_tips(&config).await?;
    if unsynced_tips.is_empty() {
        info!(target: LOG_TARGET, "IBD complete: all configured peer tips are present in the local tree");
        return Ok(());
    }
    let info = self.block_processor.info().await?;
    enqueue_tips(&mut downloader, unsynced_tips, &info);

    self.drain_downloader(&mut downloader).await;

    tokio::time::sleep(config.round_delay).await;
}
```

The only success exit is "no fetched tip is missing". The only error exit is `AllPeersFailed`, raised when no peer returned a tip in any of the fetch attempts (`:237-241`, `:314-316`). A tip that was fetched but cannot be reached has no exit:

- If a block of its chain fails to apply, `drain_downloader` logs a warning and calls `cancel_active_download` (`:213-216`). It does not record anything. The downloader goes idle, the round ends, and the next round enqueues the same tip again.
- If no queried peer can serve the tip, the request fails and the downloader returns to idle (`sync/orphan_handler.rs:381-387`); an empty stream ends the same way (`:457-469`). Again nothing is recorded.
- The downloader is built with the rejected-blocks cache (`ibd.rs:164-172`), but IBD never inserts into it, and `collect_unsynced_tips` does not read it. The cache cannot make a bad tip drop out. `enqueue_tips` passes `parent_id = None` (`:331`), so the cache's parent rule cannot fire either. This part was already recorded as #143 LB-004; the liveness consequence was left to this issue.

The specification asks for the opposite on both counts. `initial_block_download` counts successes and fails only `if num_success == 0`; `download_blocks` returns on the first block that fails (`except: return`), which makes that peer a failed peer, not a reason to continue. The text says: "If the node fails to catch up with at least one IBD peer (e.g., network error or invalid blocks), the node is terminated with an error, allowing the operator to restart the node with other IBD peers." The code neither succeeds on one good peer nor terminates on a bad one. I believe the code is the side that is wrong.

What the node does while it is held in IBD:

- `chain-network` has not yet subscribed to gossiped proposals or to sync requests; both subscriptions come after IBD returns (`lib.rs:364-365`). New blocks reach the node only through the next IBD round's tip poll.
- `chain-service` stays in the `InitialBlockDownload` phase, whose loop ends only on `ConsensusMsg::IbdCompleted` (`service/phases/ibd.rs:83-103`). The Prolonged Bootstrap Period never starts, so a node that started with the Bootstrap rule never switches to the Online rule.
- The leader service waits for the chain to become online before proposing (`chain-leader/src/lib.rs:437-444`), and the same file notes that the Blend service "becomes ready after chain becomes online" (`:423`). The node neither proposes nor takes part in Blend.
- Every round sends one tip request to each configured peer and one block request to up to 16 connected plus 16 discovered peers (`network/adapters/libp2p.rs:391-402`, defaults at `config/cryptarchia/serde/network.rs:121-122`), and writes at least one `warn!` or `error!` line. With the default `round_delay` of 1 s (`serde/network.rs:51`) that is up to 32 block requests and one log line per second, for as long as the process runs.

How a peer ends up in the configured set matters for the rating. `config init` and `config update` copy *every* initial peer into `ibd.peers` unless `--skip-ibd` is given (`cli/config/init.rs:172-181`, `cli/config/update.rs:143-150`). IBD also runs on every start, not only the first. So the set is normally the network's public bootstrap nodes, and one bad entry affects every node that lists it, each time one of them restarts.

Experiment EXP-A, unmodified IBD logic. Two configured peers: a good one with chain `G-1-2`, and one whose tip's parent is unknown, which is the same bad tip the in-tree test `block_apply_error_triggers_cancel` uses. Rejected cache enabled with capacity 1000. One hour of virtual time:

```
EXP-A outcome=TIMED OUT (IBD still running) virtual_elapsed=3600s good_tip_requests=3601 bad_tip_requests=3601 download_requests=3601 blocks_applied=2 apply_failures=3600
```

The good peer's two blocks were applied once, in the first round. The remaining 3,600 rounds each asked both peers for their tip, requested the bad tip's blocks, and failed to apply them. The fixture applies blocks instantly, so a round is one `round_delay`; on a real network a round also includes the download time.

I rate this Medium: it removes a node from consensus and from Blend under conditions that need no attacker, and it affects the nodes that share the bad peer rather than the whole network. I rate difficulty High because a deliberate attacker must already be one of the victim's configured peers. For such a peer the attack itself is trivial, since any 32 bytes that are not a known block ID will do as a tip.

**Exploit scenario**

1. *No attacker.* A testnet is reset with a new genesis. One of the published bootstrap nodes is still running the old chain. The sync protocol name is a configured string, not derived from the genesis (`nodes/node/binary/src/config/deployment/settings.yaml:19`), so unless the reset also changed it, the old node answers tip requests with a tip of the old chain. Every node that was initialised with the published peer list downloads towards that tip, the first streamed block has an unknown parent, and the round is cancelled. The node syncs the new chain from the other bootstrap nodes and then stays in IBD. Operators see a node that follows the chain but never proposes; the log shows one "failed to process block; cancelling the download" line per second.
2. *Malicious or compromised bootstrap node.* The node answers every `GetTip` with a random header ID. No peer can serve it, the download fails to start, and the effect is the same for every node that lists it. The bootstrap node stays otherwise useful, so the cause is not obvious from outside.
3. *A peer wedged by another defect.* The #143 and #184 reports describe ways a node's orphan pipeline can stop accepting an honest chain until restart (not re-verified here). Such a node keeps answering tip requests. If the branch it is on later fails to apply on a restarting node, or falls below that node's LIB (LB-002), the restarting node is held in IBD.

**Recommendation**

- *Short term*: bound the work per peer and apply the spec's success rule. Fetch tips as `(peer, tip)` pairs. After draining, a peer whose tip is still not local has failed that round. Drop a peer after a small number of failed rounds. Return `Ok` when every remaining peer's tip is local, or when all remaining peers have been dropped and at least one tip was reached. Return `AllPeersFailed` when all have been dropped and none was reached. The core of the prototype I ran:

  ```rust
  let mut active: HashSet<NetAdapter::PeerId> = config.peers.clone();
  let mut failed_rounds: HashMap<NetAdapter::PeerId, usize> = HashMap::new();
  let mut reached_any = false;
  loop {
      if active.is_empty() {
          return if reached_any { Ok(()) } else { Err(Error::AllPeersFailed(AllPeersFailed)) };
      }
      let mut unsynced = HashMap::new();
      for (peer, tip) in self.collect_tips(&config, &active).await? {
          if self.block_processor.has_processed_block(tip).await? { reached_any = true; }
          else { unsynced.insert(peer, tip); }
      }
      if unsynced.is_empty() { return Ok(()); }
      let info = self.block_processor.info().await?;
      enqueue_tips(&mut downloader, unsynced.values().copied().collect(), &info);
      self.drain_downloader(&mut downloader).await;
      for (peer, tip) in unsynced {
          if self.block_processor.has_processed_block(tip).await? {
              reached_any = true;
              failed_rounds.remove(&peer);
          } else {
              let rounds = failed_rounds.entry(peer).or_default();
              *rounds += 1;
              if *rounds >= MAX_FAILED_ROUNDS_PER_PEER { active.remove(&peer); }
          }
      }
      tokio::time::sleep(config.round_delay).await;
  }
  ```

  With `MAX_FAILED_ROUNDS_PER_PEER = 3`, the same experiments give:

  ```
  EXP-A outcome=completed Ok virtual_elapsed=3s good_tip_requests=4 bad_tip_requests=3 download_requests=4 blocks_applied=2 apply_failures=3
  EXP-D outcome=completed Err(All peers failed) virtual_elapsed=3s bad_tip_requests=3
  ```

  EXP-D is a single configured peer with the bad tip; it now ends with the error the spec asks for. The crate's 45 existing unit tests still pass. Two limits of the prototype: it counts a round as failed even when blocks were applied towards the tip, so with the provider race of #146 LB-001 an honest peer could be dropped after three unlucky rounds; counting only rounds that applied no block of that peer's chain would avoid this. And it keeps today's behaviour that a tip-fetch failure after a successful round still ends IBD with an error (`all_peers_go_dark_after_first_round`), which is stricter than the spec.
- *Long term*: key the bound by peer, not by tip, so that a peer reporting a fresh unknown tip each round cannot reset it. Give the limit a configuration field next to `round_delay`. Replace the comment at `ibd.rs:510-516` with tests for "one bad peer among good ones completes" and "only bad peers fails".

**References**: `cryptarchia-v1-bootstr-sync.md` §Initial Block Download, §Downloading Blocks; #143 LB-004 (issue #557); #146 LB-001.

### LB-002 · A configured peer whose tip is older than the local LIB keeps IBD running forever, because the tip check reads the pruned ledger-state map

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Denial of Service |
| Target | `services/chain/chain-network/src/bootstrap/ibd.rs:L73-L75` (`has_processed_block`), `services/chain/chain-service/src/lib.rs:L534-L552` (`prune_ledger_states`) |
| Status | Open |

**Description**

The spec's `download_blocks` starts with `if local_tree.has(target_block): return`. IBD implements `has` as "a ledger state exists for this block":

```rust
// bootstrap/ibd.rs:73-75
async fn has_processed_block(&self, block_id: HeaderId) -> Result<bool, Error> {
    Ok(self.cryptarchia.get_ledger_state(block_id).await?.is_some())
}
```

Ledger states do not exist for blocks older than the LIB. When the LIB advances, the engine removes every block below it from its branch map (`cryptarchia-engine/src/lib.rs:639-649`), and the chain service removes the ledger state of every pruned block, immutable ones included (`chain-service/src/lib.rs:306-313`, `:534-552`). So a block on the node's own canonical chain, below its LIB, is reported as not processed.

Such a block can never become processed again. It cannot be re-applied, because its parent's state is gone as well, and the apply path returns `ParentMissing` (`chain-service/src/lib.rs:471-476`). A peer whose tip is an ancestor of the local LIB therefore counts as unsynced in every round, and with the loop of LB-001 IBD never ends. The peer has done nothing wrong; the local node is simply ahead of it.

This needs the local LIB to be past the peer's tip, which means a restart. Under the Bootstrap rule the LIB stays where it was at start (`cryptarchia-engine/src/lib.rs:60-74`), so a node syncing from genesis is not affected: an old honest fork is just downloaded as a fork. A node that restarts with a saved LIB, under either rule, is affected when one configured peer answers tip requests while more than the LIB depth behind. A peer only answers tip requests in the `Following` phase, so this is a peer that has stopped keeping up (partitioned, wedged, or on a fork that diverged below the restarting node's LIB), not a peer that is still syncing.

Experiment EXP-C, unmodified IBD logic. The local processor runs the real engine under the Online rule with `k = 1` and holds the chain `G-1-2-3-4`. One peer is up to date (tip 4). The other is honest but behind (tip 1):

```
EXP-C local tip=HeaderId(0404040404040404040404040404040404040404040404040404040404040404) lib=HeaderId(0303030303030303030303030303030303030303030303030303030303030303) has_block(1)=false has_block(3)=true has_block(4)=true
EXP-C outcome=TIMED OUT (IBD still running) virtual_elapsed=3600s current_tip_requests=3601 lagging_tip_requests=3601 download_requests=3600 apply_failures=3600
```

Two limits of this experiment. The fixture's `has_processed_block` reads the engine's branch map rather than the ledger-state map; both are pruned from the same list, which I checked by reading (`chain-service/src/lib.rs:415`, `:534-552`), not by running a ledger. And the fixture's provider streams the peer's chain from genesis, where a real provider would, from my reading of `block_provider.rs:286-311` and `:403-436`, answer with a failure because the requester's known blocks are all newer than the target. I did not run a real provider. The outcome does not depend on which of the two happens: in neither case can block 1 regain a state.

I rate this Low. The impact is that of LB-001, but it needs a restart to coincide with a configured peer that is far behind while still in `Following`.

**Exploit scenario**

A bootstrap node loses its uplink and keeps running, for long enough that the network produces more blocks than the LIB depth. Its tip is now older than the LIB of every healthy node. It comes back, reconnects, and starts answering tip requests with its old tip while it catches up through the orphan path. Any node that lists it and restarts during that window sees a tip below its own LIB and stays in IBD. It leaves IBD only if a later round happens to see the bootstrap node's tip at or above the local LIB and present locally.

**Recommendation**

- *Short term*: the per-peer bound of LB-001 turns this into a dropped peer (EXP-C with the prototype: `outcome=completed Ok virtual_elapsed=3s ... lagging_tip_requests=3`). With a single configured peer it would turn into `AllPeersFailed`, which is still wrong for a node that is ahead of its peer.
- *Long term*: answer `has` from the chain, not from the ledger-state map. A tip counts as present if it has a ledger state or is an immutable block in storage (the block provider already has this lookup, `block_provider.rs:208-230`). The tip response also carries `slot` and `height` (`cryptarchia-sync/src/messages.rs:15-21`), which IBD discards (`ibd.rs:302`); a tip whose height is below the local LIB's could be skipped without any lookup, as the spec's orphan rule already does with `block.height ≤ local_tree.latest_immutable_block().height`.

**References**: `cryptarchia-v1-bootstr-sync.md` §Downloading Blocks (`download_blocks`), §Listening for New Blocks.

### LB-003 · Nodes that are still in IBD or in the Prolonged Bootstrap Period do not answer tip requests, so nodes that list only each other cannot start together

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Configuration |
| Target | `services/chain/chain-service/src/service/mod.rs:L1254-L1280` (`reject_chain_sync_event`), `service/phases/ibd.rs:L93`, `service/phases/pbp.rs:L99`, `service/phases/awaiting_genesis_time.rs:L148`; `services/chain/chain-network/src/lib.rs:L365` |
| Status | Open |

**Description**

This is the answer to checklist item 3, and one consequence of it.

A node answers `ProvideTipRequest` only in the `Following` phase (`following.rs:70-72`). In `AwaitingGenesisTime`, `InitialBlockDownload` and `ProlongedBootstrapPeriod` the chain service replies `Unavailable { reason: "Node is not in online mode" }` (`service/mod.rs:1265-1279`), which the provider sends as `GetTipResponse::Failure` (`cryptarchia-sync/src/libp2p/provider.rs:54-57`). While a node is in its own IBD, its `chain-network` service has not subscribed to sync events at all (`lib.rs:365` comes after IBD returns), so the request fails there without reaching the chain service; I did not trace which error the requester sees in that case.

On the requesting side both forms are an error for that peer in that round (`ibd.rs:301-308`): the peer is logged and left out of the tip set. It is asked again in the next round, because every round queries all of `config.peers` (`:275`). So a bootstrapping peer neither blocks IBD nor is given up on; IBD completes against the peers that answer, and the in-tree test `one_peer_fails_tip` covers this.

If no configured peer answers, the node terminates. Experiment EXP-B, two peers that always fail the tip request, default retry settings (`tips_fetch_max_attempts: 3`, 250 ms to 1 s backoff):

```
EXP-B result_is_all_peers_failed=true virtual_elapsed=2.875s peer0_tip_requests=4 peer1_tip_requests=4
```

Three runs printed the same 2.875 s. The backoff has jitter, so I expected the runs to differ; I did not look into why they do not. `chain-network` then shuts the node down (`lib.rs:344-361`). Terminating is what the spec asks for. The consequence is about who is able to answer:

- The shipped default for the Prolonged Bootstrap Period is one hour (`config/cryptarchia/serde/service.rs:49`; the spec's value is 24 hours, #42 LB-002). For that long after a restart under the Bootstrap rule, a node is of no use as an IBD peer.
- Nodes that list only each other as IBD peers cannot complete IBD at the same time. Each needs the other to be in `Following`, and neither gets there without completing IBD. Whichever asks first terminates within about three seconds. Under a process supervisor they restart and fail again. `config init` makes every initial peer an IBD peer by default (`cli/config/init.rs:172-181`), so this is the default shape for a set of bootstrap nodes that list each other.
- After an outage of the whole network, every node restarts into IBD. None answers tip requests, so every node with IBD peers terminates. Someone has to start a node with no IBD peers first. The spec says genesis nodes configure no IBD peer; it does not say what to do when restarting a network that already has a chain.

I rate this Low: the precondition is that all of a node's configured peers are bootstrapping at once. That is unlikely in normal operation, but it is exactly the situation during recovery from a network-wide outage.

**Exploit scenario**

A three-node devnet is deployed with each node listing the other two as initial peers, and all nodes are upgraded in one step after being down for longer than the offline grace period. Each node starts, asks the other two for their tip, gets two failures four times over, logs "Initial Block Download failed ... Retry with different bootstrap peers" and exits. The supervisor restarts them and the same thing happens. The network does not come back until an operator restarts one node with `--skip-ibd` or an empty `ibd.peers`.

**Recommendation**

- *Short term*: document that at least one node of any deployment must start with no IBD peers, and say so in the error message next to "Retry with different bootstrap peers". Distinguish "peer is bootstrapping" from "peer unreachable" in the tip response handling, and wait for a bootstrapping peer (with a bound) instead of counting it towards `AllPeersFailed`.
- *Long term*: let a node in the Prolonged Bootstrap Period answer tip requests. By then it has completed IBD and holds a validated chain; what the Bootstrap rule restricts is proposing, not serving. If that is not wanted, the spec should say that bootstrapping nodes do not serve, and what a restart of the whole network requires (S-001).

**References**: `cryptarchia-v1-bootstr-sync.md` §Initial Block Download, §Prolonged Bootstrap Period; #42 LB-002.

### 4.1 Checked and ruled out

- **A good peer whose tip keeps advancing does not keep IBD running.** A round drains the downloader completely, across as many streams as the chain needs (`orphan_handler.rs:433-455`), so after a round the local tree holds the tip fetched at its start. The next round sees at most the blocks produced during the round, and IBD completes in the first round that starts with no new block. The in-tree test `tip_advances_between_rounds` covers the two-step case.
- **Configured peers on different honest forks do not keep IBD running** while the local LIB is below the fork point. Each fork applies as a branch and its tip gets a ledger state. Under the Bootstrap rule the LIB does not move during IBD, so this always holds for a node syncing from genesis. Below the LIB it is LB-002.
- **A tip in a future slot is retried but bounded.** `FutureBlock` (`chain-service/src/lib.rs:443-448`) cancels the round like any other error; the next round succeeds once the slot has arrived.
- **`AlreadyApplied` does not cancel a download** (`ibd.rs:208-212`), so a provider that re-streams a known prefix does not cause a failed round. The in-tree test `already_applied_prefix_does_not_cancel_download` covers it.
- **The rejected cache cannot wedge IBD.** Since IBD never inserts, no honest tip can be refused at `enqueue_orphan` (`orphan_handler.rs:164-175`) during IBD, and the cache IBD uses is dropped when IBD returns; the online downloader is built fresh (`lib.rs:371-375`).
- **`delay_before_new_download`** is still in the node's config struct and in the shipped YAML, but it is documented as deprecated and unused (`serde/network.rs:31-32`) and is not mapped into the service settings (`config/cryptarchia/mod.rs:118-130`). Not a finding.
- **Empty peer set**: `run` returns at once with a warning (`ibd.rs:134-137`), as the spec requires for genesis nodes.

## 5. Suggestions (non-security)

### S-001 · Spec: say whether bootstrapping nodes serve tip and block requests, and how a network with an existing chain is restarted

`cryptarchia-v1-bootstr-sync.md` defines when a node may propose (after the Prolonged Bootstrap Period) but not when it may serve `DownloadBlocksRequest` or tip requests. The implementation refuses both until `Following`, which with a 24-hour period makes every recently restarted node unusable as an IBD peer for a day. The spec also covers starting a network (genesis nodes configure no IBD peer) but not restarting one after a full outage, where no node is online to be an IBD peer (LB-003). Both deserve a sentence upstream.

### S-002 · Spec: the tip request is used by the pseudocode but not specified

`download_blocks` calls `peer.tip()`, and the implementation has a `GetTip` request whose response carries `tip`, `slot` and `height` or a failure string. The spec defines only `DownloadBlocksRequest` and its response. Specifying the tip message, including the failure form and what a requester must do with `height` (LB-002), would make the termination rule checkable.

### S-003 · IBD rounds log at `warn!`/`error!` once per second while stuck

`ibd.rs:214` warns on every failed apply and `orphan_handler.rs:382` logs an error on every failed request; with the default `round_delay` that is one line per second with identical content. Once LB-001 is fixed the volume is bounded by the per-peer limit. Until then, logging the first occurrence per tip at `warn!` and the rest at `debug!` would keep a stuck node's log readable. This belongs with the log-volume review in #218.
