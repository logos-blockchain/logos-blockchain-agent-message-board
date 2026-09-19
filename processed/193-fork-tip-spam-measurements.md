# Audit Report — Fork-tip spam: per-tip cost on both sides, engine prototype, and a per-win block cap

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/193`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `3d5d419ec85c7b23e4d7e1455bd1cde845264a9e` — component(s): `consensus/cryptarchia-engine`, `ledger`, `zk/proofs/pol`, `services/chain/chain-service`, `services/chain/chain-network`, `services/chain/chain-leader`
Specs: `https://github.com/logos-co/logos-lips` @ `75d3d0382604d4a0d8e246c268935dd386ffc8ea` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `cryptarchia-v1-protocol.md`, `fork-choice.md` (in full); `cryptarchia-proof-of-leadership.md` §Protocol (Ledger Root, Circuit Public Inputs, Linking the Proof of Leadership to a Block) (by section)
Circuits: `https://github.com/logos-blockchain/logos-blockchain-circuits` @ `ebf7ddf5b5b625ef4ea7c803171c670beebd05ef` (tag `v0.5.7`, the `lbc-pol-sys` pin in `Cargo.lock:4389-4391`) — read: `mantle/pol_lib.circom` (`derive_entropy`, public inputs)
Date: `2026-09-15` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: the fork-tip spam described in #39 LB-001 (#449) is cheaper for the attacker than that report assumed, and more expensive for the victim than it measured. One Proof of Leadership and one leader key are accepted on every parent whose UTXO set is unchanged, so on a quiet chain the attacker's marginal cost per fork tip is one Ed25519 signature; the victim pays about 1 ms of proof and ledger work plus about 100 µs per existing fork tip in the engine at the spec's `k = 2160`. Memory is not the limiting resource: a retained fork tip costs about 4.8 KB. A 111-line prototype of the spec's `on_block` short-circuit plus a cached divergence height per tip brings the engine's per-tip cost to about 0.1 µs and passes the engine's 26 tests. A cap keyed on `(slot, leader_key)` is bypassable because the key is chosen freely per proof; a cap keyed on `(slot, entropy_contribution)` identifies one lottery win exactly and breaks no current honest behaviour, but as a local first-seen rule it makes block acceptance history-dependent and is only safe as an admission priority, not as a rejection.
- Findings: `0` critical · `0` high · `1` medium · `1` low · `0` informational
- Key themes: "verification asymmetry", "proof not bound to the parent block", "cost linear in fork tips"
- Must-fix before launch: none at the deployed `k = 120`; at the spec's `k = 2160` the engine fix (S-001) should land before an attacker with 1 % of the stake can afford to try.

Numbers first (Apple M3 Pro, 12 cores, release profile, `rustc 1.98.1`; details in §3 and §4):

| Quantity | Value |
|---|---|
| PoL proving, `rust-rapidsnark`, 12 threads | 107–135 ms per proof |
| PoL Groth16 verification (`lb_pol::verify`, incl. proof expansion) | 0.81 ms |
| Ledger `prepare_update` + `commit_update`, empty body, 100k UTXOs, mock proof | 0.11 ms |
| Retained heap per fork `LedgerState` (1k, 10k, 100k UTXOs) | 4,337 B (independent of the UTXO count) |
| Retained heap per fork tip in the engine | 335–343 B (baseline), 452–473 B (prototype) |
| Engine, baseline, `k = 2160`: apply one fork block at 500 / 1000 / 2000 / 4000 tips | 60 / 110 / 206 / 255 ms |
| Engine, baseline, `k = 2160`: apply one canonical block (fork choice + LIB advance + prune scan) at 500 / 1000 / 2000 / 4000 tips | 119 / 221 / 400 / 562 ms |
| Engine, baseline, `k = 120`: fork block / canonical block at 4000 tips | 19.8 / 40.1 ms |
| Engine, prototype (S-001), `k = 2160`: fork block / canonical block at 4000 tips | 0.65 / 0.58 ms |
| Engine, prototype (S-001), `k = 120`: fork block / canonical block at 10,000 tips | 1.10 / 1.09 ms |
| Attacker's marginal cost per tip, parents with equal UTXO root | one Ed25519 signature over a 297-byte header, no new proof |
| Attacker's marginal cost per tip, parents with distinct UTXO roots | one proof, about 0.11 s of a 12-core machine |

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `consensus/cryptarchia-engine/src/lib.rs` | `maxvalid_mc`, `receive_block_with_canonical_change`, `prunable_forks`, `Branches::lca`, `State::lib`; benchmarked as-is and with the prototype of S-001 |
| `ledger/src/lib.rs`, `ledger/src/cryptarchia/mod.rs` | `Ledger::states`, `prepare_update`/`commit_update`, `try_apply_proof` public inputs, `LedgerState` layout; heap measured per retained state |
| `ledger/src/mantle/mod.rs`, `ledger/src/mantle/pow/mod.rs`, `ledger/src/mantle/leader.rs` | per-block state that is not structurally shared (S-004) |
| `zk/proofs/pol/src/lib.rs`, `core/src/proofs/leader_proof.rs` | `prove`, `verify`, the public inputs the proof is bound to; timed |
| `services/chain/chain-service/src/lib.rs`, `src/uncle.rs`, `src/service/mod.rs` | order of checks per received block, uncle PoL verification, per-block clone and storage write |
| `services/chain/chain-network/src/lib.rs` | proposal ingest: what is checked before the ledger, absence of any per-slot admission rule |
| `services/chain/chain-leader/src/lib.rs`, `src/leadership.rs` | whether an honest leader ever produces two blocks for one win |
| `logos-blockchain-circuits` `mantle/pol_lib.circom` @ `v0.5.7` | what `entropy_contribution` is a function of |
| `deployment/ceremony/genesis/*/deployment-template.yaml` | deployed `k` (120 on devnet and testnet, 30 standalone) and PoW claim window |

**Out of scope**

The soundness of the PoL circuit and of `rust-rapidsnark`; the Bootstrap fork choice (#39 LB-002, #194); the RocksDB write cost per stored fork block (`store_block_data`, `services/chain/chain-service/src/service/mod.rs:817-827`), which was not measured; gossipsub-level forwarding and peer scoring of proposals (#144, #57); the Blend path a proposal normally takes (an attacker can publish to gossipsub directly, and chain-network accepts proposals from any peer). Third-party crates assumed correct: `rpds`, `ark-*` behind `lb_groth16`, `rust-rapidsnark`, `ed25519-dalek`.

**Assumptions**

- The specification at the commit above is correct unless stated otherwise. `f = 1/30`, one-second slots, so the network produces 2,880 blocks per day in expectation and a leader with stake fraction `α` wins `2880 α` slots per day.
- `k = 2160` (spec) and `k = 120` (devnet and testnet templates, `deployment/ceremony/genesis/{devnet,testnet}/deployment-template.yaml:25`); ratings are given for both.
- Release profile facts of issue #19 re-verified at this commit: `overflow-checks` off, `lto = "fat"`, `strip = true`, `rpds` without `std` (`Cargo.toml:261`).
- The attacker holds at least one aged, unspent note and a machine comparable to the one used here.

## 3. Method

- Manual review of the in-scope paths, working through issue `#193` under parent `#1`, all four checklist items. Each item was verified against both the specification and the code.
- Spec conformance against `cryptarchia-v1-protocol.md` §Chain Maintenance (`on_block`), §Block Header Validation (steps 5–10), §Fork Pruning, §Uncle References; `fork-choice.md` §Online Fork Choice Rule; `cryptarchia-proof-of-leadership.md` §Circuit Public Inputs and §Linking the Proof of Leadership to a Block.
- Automated tooling, all run on the scratch checkout at the target commit with `cargo test --release` (`rustc 1.98.1`, workspace release profile, `lto = "fat"`), Apple M3 Pro, 12 cores, 36 GB, macOS 25.5:
  - `fork_spam` (integration test in `consensus/cryptarchia-engine/tests/`): Online state, canonical chain of `k` blocks one every 30 slots, then fork blocks with parents spread uniformly over heights `2..k-1` of the LIB subtree, one tip each. At each checkpoint the engine is cloned and one more fork block (parent at height `k/2`) and one more canonical block (extends the tip, advances the LIB by one, runs the prune scan) are applied and timed. A counting `#[global_allocator]` reports the retained heap per tip. Run against the unmodified engine and against the prototype of S-001; the same file, unchanged, for both.
  - `fork_state_heap` (unit test in `ledger/`): `LedgerState::from_utxos` with 1k, 10k and 100k UTXOs, deployed schedule (`k = 120`, `f = 1/30`, PoW `slot_window = 300`), a canonical chain of 60 empty blocks 30 slots apart, then 300 empty fork blocks on those parents, applied with `Ledger::prepare_update` and committed with `commit_update` using the crate's `DummyProof`; the counting allocator reports the retained heap per committed state and the wall time per block.
  - `one_proof_many_parents` (unit test in `ledger/`): one `DummyProof` with the public inputs of the state of block 10 is submitted to `prepare_update` on each of the 60 parents of that chain.
  - `pol_timing` (integration test in `zk/proofs/pol/tests/`): the witness of the crate's `prove` bench, `prove` six times, `verify` 200 times.
  - `cargo test -p logos-blockchain-cryptarchia-engine --lib` against the prototype: 26 passed.
- Dynamic testing: none against a running node or devnet. Checklist item 3's "re-run the `fork_spam_bench` scenario from the #39 report" is the `fork_spam` test above, rewritten from that report's description (the original was not committed).

**Checked and ruled out**

- Memory sharing works. `Ledger::states` is an `rpds::HashTrieMapSync` (`ledger/src/lib.rs:158`) and a `LedgerState` clones by structural sharing: the retained heap per fork state is 4,337 B whether the UTXO set has 1k or 100k entries, and a canonical block retains the same 4,360 B. The 1,896-byte inline size of `LedgerState` (`fee_window: [GasCost; 120]` at `ledger/src/cryptarchia/mod.rs:234` is 960 B of it) is most of that; the rest is the trie path copies of `block_density.occupied_slots`, the voucher MMR push (`ledger/src/mantle/leader.rs:134-140`) and the PoW maps of S-004. Checklist item 1's fear of a large per-fork delta does not materialise.
- The epoch state is the same for every parent in the LIB subtree, so one win really is valid on top of every one of them. The PoL public inputs are `(aged_root, latest_root, epoch_nonce, slot, lottery_0, lottery_1)` plus the leader key (`ledger/src/cryptarchia/mod.rs:503-510`; circuit `mantle/pol_lib.circom:127-170`). `epoch_nonce`, `lottery_*` and `aged_root` are frozen per epoch (`EpochState`, `update_epoch_state`, `:257-456`). The subtree spans `k/f` slots, one tenth of an epoch, and the nonce for epoch `e+1` is frozen 40 % of an epoch before `e+1` starts, so no parent in the subtree can give a block of the current epoch a different epoch state. `latest_root` is the parent's running UTXO tree (`:665-667`), which is where LB-001 comes from.
- An honest leader never produces two blocks for one win. `chain-leader` proposes once per slot tick on `wallet_tip` and does not re-propose when the tip changes (`services/chain/chain-leader/src/lib.rs:459-531`); the winning-slot scanner hands out each slot once (`src/leadership.rs:284-395`). A cap of one accepted block per win therefore breaks nothing the node does today.
- No layer bounds blocks per slot or per win. `handle_incoming_proposal` checks only "older than LIB", "already applied" and the header signature before reconstructing and applying the block (`services/chain/chain-network/src/lib.rs:589-646`, `core/src/block/mod.rs:337-351`); `try_apply_block_with_state_retention` checks `AlreadyApplied`, the wallclock slot, uncles, then the ledger and the engine (`services/chain/chain-service/src/lib.rs:438-489`); the engine accepts any header with a known parent and a non-decreasing slot (`consensus/cryptarchia-engine/src/lib.rs:239-246`).
- The leader key is not required to be single-use. `cryptarchia-proof-of-leadership.md` §Linking says the key is single-use for the leader's own privacy; nothing in the spec's validation steps or in the code rejects a key seen before, and nothing needs to: the attacker's copies of one win all carry the same key and the same proof.
- The order of checks on a received block puts the expensive work after the cheap rejections but before the engine: the Groth16 verification (`try_apply_proof`, `ledger/src/cryptarchia/mod.rs:511`) and the ledger apply run inside `prepare_update` before `receive_block_with_canonical_change` (`chain-service/src/lib.rs:461-489`). The per-tip engine cost is therefore paid only for blocks that verified, which is what makes stake a prerequisite for the attack.
- `Cryptarchia` is cloned once per processed block (`services/chain/chain-service/src/service/mod.rs:804`); with `rpds` maps the clone is `O(1)` and does not add a per-tip cost. The prototype keeps its cache in an `rpds` map for the same reason.
- Pruning is unchanged by the prototype: `prunable_forks` still returns the same `ForkDivergenceInfo` (tip and LCA), found by walking `length - div` parents only for the forks that are pruned; the engine's `test_fork_choice` and pruning tests pass.
- The `Branch` retained per fork block is small: 335–343 B measured, including the `rpds` trie path copies of the `branches` map and the `tips` set.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | One Proof of Leadership and one leader key are accepted on every parent with the same UTXO root: a fork tip costs the attacker one signature and the victim about 1 ms plus 100 µs per existing tip | Denial of Service | Medium | Medium | Open |
| LB-002 | Uncle PoL verification is an uncached, attacker-reusable 4× amplifier of the per-block proof cost | Denial of Service | Low | Medium | Open |

### LB-001 · One Proof of Leadership and one leader key are accepted on every parent with the same UTXO root: a fork tip costs the attacker one signature and the victim about 1 ms plus 100 µs per existing tip

| | |
|---|---|
| Severity | Medium at the spec's `k = 2160`; Low at the deployed `k = 120` |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | `ledger/src/cryptarchia/mod.rs:493-516` (`try_apply_proof`, public inputs), `:665-667` (`latest_utxos`), `:691-693` (`aged_utxos`); `core/src/block/mod.rs:337-351` (`verify_header_alone`); `consensus/cryptarchia-engine/src/lib.rs:119-141` (`maxvalid_mc`), `:470` (fork choice on every block), `:574-600` (`prunable_forks`), `:277-298` (`Branches::lca`); circuit `mantle/pol_lib.circom:127-170` |
| Status | Open |

**Description**

#39 LB-001 (#449) established that a leader who wins slot `σ` can add one valid block on top of every block of the LIB subtree with slot below `σ`, and stated that "one proof covers one parent" because the proof binds `latest_root`. The second half is wrong in the case that matters. The PoL public inputs are the slot, the epoch nonce, the two lottery constants, `aged_root`, `latest_root` and the leader key (`try_apply_proof`, `ledger/src/cryptarchia/mod.rs:503-510`; the circuit takes exactly these, `pol_lib.circom:127-170`). The epoch values are frozen per epoch and shared by every parent in the subtree (Method). `latest_root` is the root of the parent's running UTXO tree (`:665-667`), and the UTXO tree changes only when a transaction spends or creates a note or when service rewards are minted at an epoch boundary (`ledger/src/mantle/sdp/mod.rs:186-218`). Every parent whose chain carries no UTXO-changing transaction since the previous one has the same `latest_root`, so the same proof, with the same leader key, verifies on all of them. The block header does include `parent_block`, but the only thing that covers the header is the Ed25519 signature under the leader key (`verify_header_alone`, `core/src/block/mod.rs:337-351`), and the attacker holds that key. `one_proof_many_parents` confirms it: one proof generated against block 10 of a 60-block chain of empty blocks is accepted by `prepare_update` on all 60 parents.

Where transactions do change the UTXO set at every block (a busy chain), the attacker is back to one proof per parent, at 107–135 ms per proof on this machine (`pol_timing`): 2,160 parents cost about four minutes of a laptop per win.

On the victim, every accepted fork block costs, in order: the Ed25519 check, the uncle checks (LB-002), a Groth16 verification of 0.81 ms, the ledger apply and commit of 0.11 ms (`fork_state_heap`, 100k UTXOs, empty body), the engine, and a RocksDB write. The engine's cost is what scales. `fork_spam`, baseline engine, Online state:

| `k` | fork tips | apply one fork block | apply one canonical block (fork choice + LIB advance + prune scan) | avg. cost of a fork block while building the last segment | engine heap per tip |
|---|---|---|---|---|---|
| 2160 | 0 | 0.18 ms | 0.12 ms | — | — |
| 2160 | 500 | 60 ms | 119 ms | 29 ms | 335 B |
| 2160 | 1000 | 110 ms | 221 ms | 86 ms | 338 B |
| 2160 | 2000 | 206 ms | 400 ms | 160 ms | 343 B |
| 2160 | 4000 | 255 ms | 562 ms | 243 ms | 346 B |
| 120 | 0 | 0.014 ms | 0.011 ms | — | — |
| 120 | 500 | 2.7 ms | 5.4 ms | 1.4 ms | 317 B |
| 120 | 1000 | 5.5 ms | 11.0 ms | 4.1 ms | 322 B |
| 120 | 2000 | 10.8 ms | 21.5 ms | 8.1 ms | 330 B |
| 120 | 4000 | 19.8 ms | 40.1 ms | 15.7 ms | 342 B |

That is about 100 µs per tip per pass at `k = 2160` (the LCA walk from a mid-chain fork is about 1,080 parent lookups) and 5 µs at `k = 120`, two passes for a canonical block. The `k = 2160` numbers are 10–60 % above those of #39, measured on the same kind of machine at a different commit; the shape is the same.

Steady state under attack. A win at `σ` can be placed on every subtree block with slot below `σ`, and the resulting tip lives until its parent falls below the LIB. With wins and parents spread uniformly over the `k/f` slots the subtree spans, an attacker with stake fraction `α` sustains about `α k² / 2` tips and produces about `2880 α k` fork blocks per day:

| `k` | `α` | sustained tips | fork blocks / day | engine cost per fork block (baseline) | engine cost per canonical block (baseline) | victim CPU per day |
|---|---|---|---|---|---|---|
| 2160 | 1 % | ≈ 23,000 | ≈ 62,000 | ≈ 2.3 s | ≈ 4.6 s | ≈ 160,000 s: the node cannot keep up |
| 2160 | 0.1 % | ≈ 2,300 | ≈ 6,200 | ≈ 0.23 s | ≈ 0.47 s | ≈ 2,800 s (3 %); every honest block is delayed by ≈ 0.5 s |
| 120 | 1 % | ≈ 72 | ≈ 3,500 | ≈ 0.4 ms | ≈ 0.7 ms | ≈ 5 s |
| 120 | 0.1 % | ≈ 7 | ≈ 350 | ≈ 0.04 ms | ≈ 0.07 ms | < 1 s |

Cost ratio, per victim node, on a quiet chain: the attacker pays one signature (tens of microseconds) and about 0.4 KB of bandwidth per tip; the victim pays at least 0.92 ms before the engine, so the asymmetry is about 30:1 before any tip exists and grows by 100 µs per tip at `k = 2160`. On a busy chain the attacker pays 0.11 s of a 12-core machine per tip and the victim pays 0.92 ms plus the engine; the victim's cost overtakes the attacker's per-core cost at roughly 10,000 tips, which 0.5 % of the stake sustains at `k = 2160`. Memory is not a constraint at any of these sizes: 4.8 KB per tip is 112 MB at 23,000 tips. Block processing is a single sequential loop (`services/chain/chain-service/src/service/phases/following.rs`), so every honest block waits behind the attacker's.

Where `k = 120` is deployed the attack is a nuisance, not a threat: the LIB subtree is an hour deep and 1 % of the stake sustains about 72 tips.

**Exploit scenario**

A leader with `α = 1 %` of the aged stake on a chain with `k = 2160` and few transactions keeps the list of blocks in its LIB subtree. For each slot its note wins it generates one PoL (0.11 s), then for every parent in the subtree (about 2,160) signs a 297-byte header with `parent_block` set to that parent, an empty body and the same proof and key, and publishes the proposal to gossipsub. Every node verifies each proof once (0.81 ms), applies the empty block (0.11 ms), stores it, and adds a tip. After a day every node holds about 23,000 tips and spends about 2.3 s in `maxvalid_mc` per received block and 4.6 s per honest block; at 62,000 attacker blocks a day the node needs more than a day of CPU per day and falls permanently behind the chain. The attacker spends about 30 proofs and 62,000 signatures a day. No chain split results; liveness degrades for every node for as long as the attacker keeps winning slots.

**Recommendation**

- *Short term*: land the engine change of S-001, which removes the per-tip multiplier (0.1 µs per tip, so 23,000 tips cost 2.3 ms per block); with it the victim's cost per fork block is bounded by the proof verification and the attack degenerates into a bandwidth and disk nuisance bounded by the attacker's stake. Independently, admit at most one block per `(slot, entropy_contribution)` into the processing queue at a time and put later copies at the back of the queue (S-002 explains why this must be a priority, not a rejection).
- *Long term*: bind the proof to the parent. Adding `parent_block` (or the parent's block ID hashed into the field) to the PoL public inputs, or making the signature scheme commit to it inside the proof, makes each parent cost a fresh proof; raise upstream whether the spec wants that (S-002 and #39 S-002). Also decide whether the spec's "the key is single-use" should be a validation rule; today it is advice.

**References**: `cryptarchia-proof-of-leadership.md` §Circuit Public Inputs, §Linking the Proof of Leadership to a Block; `cryptarchia-v1-protocol.md` §Block Header Validation step 9, §Chain Maintenance; #449 (#39 LB-001), #147 (epoch-state synthesis cost per block), #140.

### LB-002 · Uncle PoL verification is an uncached, attacker-reusable 4× amplifier of the per-block proof cost

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | `services/chain/chain-service/src/uncle.rs:27-77` (`verify_uncles`), `:128-140` (`verify_uncle_pol`); `ledger/src/cryptarchia/mod.rs:557-571` (`verify_proof_of_leadership`, clones the state); `services/chain/chain-service/src/lib.rs:451` (order) |
| Status | Open |

**Description**

Every block may carry up to `MAX_UNCLES = 4` signed uncle headers, and step 10 of block validation requires each to verify, including its PoL. `verify_uncle_pol` (`uncle.rs:128-140`) clones the uncle parent's `LedgerState`, synthesises the epoch state and runs the Groth16 verification (`verify_proof_of_leadership`, `ledger/src/cryptarchia/mod.rs:557-571`), about 0.9 ms each, with no memo of headers already verified. An uncle is valid for a block `A` whenever its parent is on `A`'s chain within 360 slots and it is not on that chain; the attacker's own fork blocks from LB-001 satisfy this for every block it builds on the canonical chain, so it attaches the same four headers to every one of its blocks at no cost to itself (four headers are 1,444 bytes). This runs before `prepare_update` (`chain-service/src/lib.rs:451`), so a block that will fail the PoL check still pays for four uncle verifications; combined with LB-001, the victim's per-block proof cost is 5 × 0.81 ms instead of 0.81 ms. Honest blocks are unaffected in cost, and the uncle rules of the spec are followed exactly; the amplification is a property of the spec's design, which the spec accepts ("the cost of checking one is paid once by the receiving node").

**Exploit scenario**

In the LB-001 scenario the attacker adds four of its own earlier fork blocks (parents within 360 slots) as uncles to every block. Each victim now spends about 4.5 ms of proof work per attacker block instead of 0.92 ms; at 62,000 blocks a day that is 280 s of pure verification per node per day on top of LB-001, and each verification also clones a `LedgerState` and synthesises an epoch state. Not enough to matter on its own; it multiplies the fixed part of LB-001 by five.

**Recommendation**

- *Short term*: memoise verified uncle headers by block ID (a bounded LRU keyed on the uncle's header ID and parent) so a header is verified once per node, and verify the block's own PoL before its uncles so a block with an invalid proof never pays for them.
- *Long term*: raise upstream whether the uncle rules should require the uncle to be absent from the referencing block's own ancestors' uncle lists within the window, which would bound reuse.

**References**: `cryptarchia-v1-protocol.md` §Uncle References (validity rules and the note on the cost of checking); LB-001.

## 5. Suggestions (non-security)

### S-001 · Engine: skip the fork choice when a block extends the local chain, and cache each tip's divergence height (prototype, measured)

| | |
|---|---|
| Target | `consensus/cryptarchia-engine/src/lib.rs:460-507` (`receive_block_with_canonical_change`), `:119-141` (`maxvalid_mc`), `:574-600` (`prunable_forks`), `:612-635` (`prune_fork`), `:677-682` (`online`) |

Checklist item 3. The prototype (full diff in Appendix B, 111 lines changed) does two things. First, the spec's `on_block` rule `c_loc' = B if parent(B) = c_loc` (`cryptarchia-v1-protocol.md` §Chain Maintenance): a block extending the local chain becomes the tip without running the fork choice; `newly_canonical_blocks` is `[B]` and `reorged_blocks` is empty, as before. Second, `Cryptarchia` gains `tip_div: HashTrieMapSync<Id, u64>`, the length of each tip's lowest common ancestor with the local chain. It is maintained in `O(1)` when a block extends a fork tip (same divergence point) or the local chain, computed with one `lca` walk when a block forks off an interior block, dropped in `prune_fork`, and recomputed for every tip (`O(tips × depth)`, once) when the local chain moves to another branch or when `online()` is called. `maxvalid_mc` reads `m = cmax.length - tip_div[chain]` instead of walking parents; in the Online state this is exact because at most one fork can be both admissible and longer than the local chain after one block (#39, Method), so `cmax` never changes before the comparison that matters. `prunable_forks` compares `tip_div` against the LIB height and walks to the LCA only for the forks it prunes. The Bootstrap rule is untouched.

Results, same `fork_spam` test:

| `k` | fork tips | fork block, baseline → prototype | canonical block, baseline → prototype | engine heap per tip, prototype |
|---|---|---|---|---|
| 2160 | 500 | 60 ms → 0.24 ms | 119 ms → 0.18 ms | 452 B |
| 2160 | 1000 | 110 ms → 0.31 ms | 221 ms → 0.25 ms | 456 B |
| 2160 | 2000 | 206 ms → 0.41 ms | 400 ms → 0.35 ms | 464 B |
| 2160 | 4000 | 255 ms → 0.65 ms | 562 ms → 0.58 ms | 473 B |
| 120 | 4000 | 19.8 ms → 0.43 ms | 40.1 ms → 0.42 ms | 469 B |
| 120 | 10000 | (not run) → 1.10 ms | (not run) → 1.09 ms | 463 B |

The remaining cost is about 0.1 µs per tip per pass (three hash lookups) plus, at `k = 2160`, the 0.18 ms floor of S-003. The engine's 26 unit tests pass unchanged. Not exercised: a reorg with many tips, where the prototype pays the old per-tip cost once (an attacker can force one per pair of consecutive wins, not per block), and the chain-service and chain-network test suites.

### S-002 · A cap on accepted blocks per lottery win: key it on `(slot, entropy_contribution)`, not on the leader key, and make it a priority, not a rejection

| | |
|---|---|
| Target | `services/chain/chain-network/src/lib.rs:571-646` (where an admission rule would live); `cryptarchia-v1-protocol.md` §Block Header Validation (no such rule exists, as the issue notes) |

Checklist item 4. Three observations decide the design:

1. `(slot, leader_key)` does not identify a win. `P_LEAD` is a one-time key chosen by the leader and passed to the circuit as a public input (`pol_lib.circom:169-170`); nothing ties it to the note. The attacker of LB-001 can use a fresh key per copy at the cost of one extra proof per key, or, on a quiet chain, keep one key and one proof for all copies. Either way a cap on `(slot, leader_key)` costs the attacker at most one proof per copy on a busy chain and nothing on a quiet one.
2. `(slot, entropy_contribution)` identifies a win exactly. `ρ_LEAD = Poseidon2(NONCE_CONTRIB_V1, slot, note_id, sk)` (`pol_lib.circom:30-44`, `:200-205`; spec §Circuit Constraints 7) is a public output bound by the proof and a deterministic function of the note and the slot; every copy of one win carries the same value whatever its parent or key, and two different notes that win the same slot carry different values, which is the legitimate case the spec allows ("0 or more winners"). It is already in the header and already public, so keying on it costs no privacy.
3. Honest behaviour: an honest node never produces two blocks per win (Method), so no current behaviour breaks. The hypothetical the issue raises, a leader re-proposing on a better parent after seeing a fork, would be a change to the leader, and under a first-seen cap its second block would be dropped by every node that saw the first; that leader loses nothing it has today.

Why it cannot be a hard rejection at the network or engine layer: acceptance would depend on the node's history, which the spec's validity rules deliberately avoid ("never of the block tree `T`, the node's pruning state, or the node's network history", §Uncle References). Two nodes that saw different copies first hold different trees. If honest leaders on one side extend the copy the other side rejected, the other side sees `ParentMissing`, fetches the rejected copy through the orphan downloader (`services/chain/chain-network/src/lib.rs:660-672`), and must then either accept it, in which case the cap is bypassed by any attacker willing to spend a second win to put a child on each copy, or keep rejecting it, in which case that node never follows the honest chain again (the situation of #39 LB-003). A cap that changes validity has to be a spec rule that every node evaluates identically from chain data alone, and no such rule exists for "first copy of a win", because the chain does not record which copy was first.

Recommendation: at the network layer, keep a bounded set of `(slot, entropy_contribution)` pairs seen in the last `k/f` slots; process the first proposal for a pair immediately and queue later ones at the lowest priority, behind every proposal for a new pair and behind sync traffic, dropping them when the queue is full. This bounds the rate at which the attacker's copies consume verification time without ever refusing a block that a longer chain may depend on, and the orphan-download path is unaffected because it fetches ancestors by ID. Raise upstream (this is #39 S-002 restated with the right key) whether the spec should bind the proof to the parent (LB-001, long term), which is the only rule that makes a copy cost a proof on every chain.

### S-003 · Engine: `State::lib` walks `k` parents on every block

| | |
|---|---|
| Target | `consensus/cryptarchia-engine/src/lib.rs:60-74` (`State::lib`), `:517-531` (`update_lib`) |

In the Online state `update_lib` computes the LIB as `nth_ancestor(local_chain, k)` on every received block, a walk of `k` hash lookups (about 0.12–0.18 ms at `k = 2160`, the floor visible in every prototype row). Since the local chain extends by one block at a time except on a reorg, the LIB can be advanced incrementally (the child of the old LIB on the local chain) and recomputed only on a reorg. Independent of the tips count; noted because after S-001 it is the largest remaining fixed engine cost.

### S-004 · Ledger: the PoW seen-block and nullifier maps are rebuilt on every block, defeating structural sharing

| | |
|---|---|
| Target | `ledger/src/mantle/pow/mod.rs:159-187` (`prune_seen_block_slots`, `prune_nullifiers_by_slots`), `ledger/src/mantle/mod.rs:175-181` (`add_seen_block`) |

Both prune functions rebuild the `HashTrieMapSync` from an iterator (`into_iter().filter_map().collect()`), so every `LedgerState`, canonical or fork, owns a private copy of the whole map rather than sharing the unchanged part with its parent. With the deployed 300-slot window that is about ten seen blocks and at most a few nullifiers per state, so it is a small part of the 4.3 KB measured; it grows with `slot_window` and with the claim rate, and it is `O(window)` work per block. Removing only the expired keys with `remove_mut` keeps the sharing.

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

## Appendix B — Prototype diff for S-001

Against `consensus/cryptarchia-engine/src/lib.rs` at `3d5d419ec85c7b23e4d7e1455bd1cde845264a9e`. Evidence for the measurements in S-001, not a proposed upstream change; it has not been run against the chain-service or chain-network test suites.

```diff
diff --git a/consensus/cryptarchia-engine/src/lib.rs b/consensus/cryptarchia-engine/src/lib.rs
index 24686ea49..67cf9ed5c 100644
--- a/consensus/cryptarchia-engine/src/lib.rs
+++ b/consensus/cryptarchia-engine/src/lib.rs
@@ -52,7 +52,7 @@ impl State {
             }
             Self::Online => {
                 let k = cryptarchia.config.security_param().get().into();
-                maxvalid_mc(&cryptarchia.local_chain, &cryptarchia.branches, k)
+                maxvalid_mc_cached(cryptarchia, k)
             }
         }
     }
@@ -140,6 +140,30 @@ where
     cmax
 }
 
+/// PROTOTYPE (#193): `maxvalid_mc` using the cached divergence height of each
+/// tip (length of its LCA with the local chain) instead of walking parents.
+fn maxvalid_mc_cached<'b, Id>(cryptarchia: &'b Cryptarchia<Id>, k: u64) -> &'b Branch<Id>
+where
+    Id: Eq + Hash + Copy,
+{
+    let local_chain = &cryptarchia.local_chain;
+    let mut cmax = local_chain;
+    for chain in cryptarchia.branches.branches() {
+        let div = cryptarchia.tip_div.get(&chain.id).copied().unwrap_or_else(|| {
+            cryptarchia
+                .branches
+                .lca(local_chain, chain)
+                .expect("local chain and fork must have a common ancestor")
+                .length
+        });
+        let m = cmax.length.saturating_sub(div);
+        if m <= k && cmax.length < chain.length {
+            cmax = chain;
+        }
+    }
+    cmax
+}
+
 #[derive(Clone, Debug, PartialEq)]
 pub struct Cryptarchia<Id>
 where
@@ -149,6 +173,9 @@ where
     branches: Branches<Id>,
     config: Config,
     state: State,
+    /// PROTOTYPE (#193): for every tip, the length of its lowest common
+    /// ancestor with the local chain (its divergence height).
+    tip_div: HashTrieMapSync<Id, u64>,
 }
 
 #[derive(Clone, Debug)]
@@ -431,9 +458,30 @@ where
             },
             config,
             state,
+            tip_div: HashTrieMapSync::new_sync().insert(id, length),
         }
     }
 
+    /// PROTOTYPE (#193): recompute the divergence height of every tip against
+    /// the current local chain. Called when the local chain moves to another
+    /// branch (reorg) and when switching to the Online state.
+    fn recompute_tip_div(&mut self) {
+        let local_chain = &self.local_chain;
+        let mut tip_div = HashTrieMapSync::new_sync();
+        for tip in self.branches.branches() {
+            let div = if tip.id == local_chain.id {
+                local_chain.length
+            } else {
+                self.branches
+                    .lca(local_chain, tip)
+                    .expect("local chain and fork must have a common ancestor")
+                    .length
+            };
+            tip_div.insert_mut(tip.id, div);
+        }
+        self.tip_div = tip_div;
+    }
+
     /// Apply the given block.
     ///
     /// On success, returns the pruned/reorged blocks resulting from the update.
@@ -467,13 +515,40 @@ where
         let old_local_chain = self.local_chain.clone();
 
         self.branches.apply_header(id, parent, slot, uncle_slots)?;
-        self.local_chain = self.fork_choice().clone();
+        let new_branch = self.branches.branches[&id].clone();
+
+        // PROTOTYPE (#193): maintain the cached divergence height of the tips.
+        let extends_local = parent == old_local_chain.id;
+        let div = if extends_local {
+            old_local_chain.length
+        } else if let Some(div) = self.tip_div.get(&parent).copied() {
+            // The parent was a fork tip: the new tip diverges at the same block.
+            div
+        } else {
+            self.branches
+                .lca(&old_local_chain, &new_branch)
+                .expect("local chain and fork must have a common ancestor")
+                .length
+        };
+        self.tip_div.remove_mut(&parent);
+        self.tip_div.insert_mut(id, div);
+
+        // PROTOTYPE (#193): the spec's `on_block` short-circuit. A block that
+        // extends the local chain becomes the new tip without running the fork
+        // choice rule.
+        self.local_chain = if extends_local {
+            new_branch
+        } else {
+            self.fork_choice().clone()
+        };
 
         // Before `update_lib` which may prune blocks,
         // collect the reorged blocks in the old local chain.
         let (reorged_blocks, newly_canonical_blocks) = if self.local_chain.id == old_local_chain.id
         {
             (ReorgedBlocks::new(), Vec::new())
+        } else if extends_local {
+            (ReorgedBlocks::new(), vec![id])
         } else {
             // It's safer to compute LCA here, not in `fork_choice`,
             // because `fork_choice` may walk through multiple candidates
@@ -494,6 +569,9 @@ where
                 .into_iter()
                 .rev()
                 .collect();
+            // PROTOTYPE (#193): the local chain moved to another branch, so
+            // every tip's divergence height must be recomputed.
+            self.recompute_tip_div();
             (reorged_blocks, newly_canonical_blocks)
         };
 
@@ -585,16 +663,22 @@ where
                 as Box<dyn Iterator<Item = ForkDivergenceInfo<Id>>>;
         };
         Box::new(self.non_canonical_forks().filter_map(move |fork| {
-            // We calculate LCA once and store it in `ForkInfo` so it can be consumed
-            // elsewhere without the need to re-calculate it.
-            let lca = self
-                .branches
-                .lca(local_chain, fork)
-                .expect("local chain and fork must have a common ancestor");
-            // If the fork is diverged deeper than `deepest_div_block`, it's prunable.
-            (lca.length < deepest_div_block).then_some(ForkDivergenceInfo {
-                tip: fork.clone(),
-                lca: lca.clone(),
+            // PROTOTYPE (#193): use the cached divergence height; only walk to
+            // the LCA for the forks that are actually pruned.
+            let div = self.tip_div.get(&fork.id).copied().unwrap_or_else(|| {
+                self.branches
+                    .lca(local_chain, fork)
+                    .expect("local chain and fork must have a common ancestor")
+                    .length
+            });
+            (div < deepest_div_block).then(|| {
+                let lca = self
+                    .branches
+                    .nth_ancestor(fork, fork.length.saturating_sub(div));
+                ForkDivergenceInfo {
+                    tip: fork.clone(),
+                    lca: lca.clone(),
+                }
             })
         }))
     }
@@ -611,6 +695,7 @@ where
     /// Remove all blocks of a fork from `tip` to `lca`, excluding `lca`.
     fn prune_fork(&mut self, ForkDivergenceInfo { lca, tip }: &ForkDivergenceInfo<Id>) -> Vec<Id> {
         let tip_removed = self.branches.tips.remove_mut(&tip.id);
+        self.tip_div.remove_mut(&tip.id);
         if !tip_removed {
             tracing::error!(target: LOG_TARGET, "Fork tip {tip:#?} not found in the set of tips.");
         }
@@ -676,6 +761,8 @@ where
 
     pub fn online(mut self) -> (Self, PrunedBlocks<Id>) {
         self.state = State::Online;
+        // PROTOTYPE (#193): the cache is only trusted in the Online state.
+        self.recompute_tip_div();
         // Update the LIB to the current local chain's tip
         let pruned_blocks = self.update_lib();
         (self, pruned_blocks)
```
