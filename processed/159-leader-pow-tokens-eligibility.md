# Audit Report — Honest activity eligibility with leader-branch and PoW-branch tokens

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/159`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `blend/message/src/reward`, `blend/provers/src/provers`, `blend/provers/src/crypto/core_and_leader`, `blend/proofs/src/quota`, `services/blend/src/core`, `services/blend/src/membership`, `ledger/src/mantle/sdp/rewards/blend`, `ledger/src/mantle/pow`, `ledger/src/cryptarchia`, `consensus/cryptarchia-engine/src/config.rs`, `deployment/ceremony/genesis/*/deployment-template.yaml`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `bedrock-service-declaration-protocol.md`, `bedrock-service-reward-distribution.md`, `proof-of-quota.md` (in full); `blend-protocol.md` §Rewarding (Overview, Protocol), §Notation, §Global Parameters, §Core Quota, §Leadership Quota, §Proof of Work Quota, §Blend Difficulty, §Quota Application, §Proof of Quota, §Proof of Selection, §Generation, §Processing, §Activity Proof, §Activity Threshold, §Active Message, §Reward Calculation; `proof-of-work.md` §Introduction, §Overview, §Protocol, §Notation, §Parameters, §Puzzle Target, §Blend Difficulty (by section)
Date: `2026-09-12` — author: `claude-fable-5.1` — status: `final`

This report extends report #110 (`inbox/110-activity-threshold-vs-quota.md`, PR #157) with the two token sources that report left out of its model, and builds on report #97 (PR #109) for the PoW branch. The issue was filed against `a805329f8`, which is the commit reviewed.

---

## 1. Summary

- Overall assessment: the supported network sizes of report #110 LB-001 stand. Leader-branch tokens add `10·k·β·(1+R_D)/N` tokens per node per epoch, which is 5% of `Q_C` on the standalone template and 6.7% on testnet, at every `N`; PoW-branch tokens add `X·β/N` where `X` is the number of transactions dispersed through Blend per epoch, at most 10% (standalone) and 6.7% (testnet) of `Q_C` if every transaction of the template's reference load goes through Blend. Both kinds are addressed to a uniformly random node by the Proof of Selection, so every node receives them in expectation whatever its stake, contrary to the "unevenly distributed" premise of the issue. With both sources added, the largest `N` with `P(honest node paid) ≥ 0.99` is unchanged at 127 on the standalone template and moves from 878 to 947 (leader tokens) or 999 (leader tokens and PoW tokens at the reference load) on testnet; the collapse at `N = 128` and `N = 1024` is untouched because restoring `P ≥ 0.99` past a knee needs 2 to 400 times more tokens than these sources supply. The node collects tokens of all three branches into one collector and the ledger verifies all three with the public inputs of the attested epoch, so a node never discards a token the ledger would accept, and never submits one the ledger rejects. Code and spec agree on every item checked.
- Findings: 0 critical · 0 high · 0 medium · 0 low · 0 informational
- Key themes: "extra tokens are a constant fraction of `Q_C`, the collapse is exponential in the threshold", "receivers of leader and PoW tokens are uniform", "eager PoW mining that produces no token"
- Must-fix before launch: none from this issue; LB-001 of report #110 remains the item to fix.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `consensus/cryptarchia-engine/src/config.rs` L133-L166 | `expected_blocks_per_epoch = epoch_length·f = 10·k` |
| `ledger/src/mantle/sdp/rewards/blend/mod.rs` L235-L298 | `RewardsParameters`, `core_quota_and_token_evaluation`, `encapsulations_per_message` (`β·(1+R_D)`), `leader_inputs`, `pow_inputs` (`Q_W = β`) |
| `ledger/src/mantle/sdp/rewards/blend/current_epoch.rs` L43-L51, L148-L186, L217-L231 | the target epoch's verifier is built from the attested epoch's `EpochState` |
| `ledger/src/mantle/sdp/rewards/blend/target_epoch.rs` L79-L125 | `verify_proof`: PoQ over all three branches, PoSel index, Hamming distance |
| `ledger/src/cryptarchia/mod.rs` L117-L140 | `blend_pow_difficulty` frozen with the nonce, from the last closed epoch's load |
| `ledger/src/mantle/pow/blend_difficulty.rs` L44-L81; `ledger/src/mantle/pow/tx_density.rs`; `ledger/src/lib.rs` L276-L285 | `d_blend` retarget and what it counts as load |
| `blend/proofs/src/quota/pow.rs` L44-L52, L63-L82 | ticket `Poseidon2(pow_nonce, epoch_nonce) < d_blend`, `solve_puzzle` |
| `blend/provers/src/crypto/core_and_leader/send.rs` L201-L222, L279-L285 | which branch backs each payload type; `β` proofs per message |
| `blend/provers/src/provers/leader/mod.rs` L91-L150 | `message_quota` proofs per winning slot, one slot per data message |
| `blend/provers/src/provers/pow/mod.rs` L38-L49, L93-L140, L144-L169, L212-L250 | one solution per `pow_quota` proofs; mining loop; eager buffer of two solutions |
| `blend/provers/src/provers/core_leader_and_pow/mod.rs`; `blend/provers/src/provers/mod.rs` L24-L30 | the combined generator; winning-slot stream |
| `blend/message/src/reward/mod.rs` L60-L105, L124-L160 | `EpochBlendingTokenCollector`, `compute_activity_proof` |
| `blend/message/src/reward/epoch.rs` L58-L75, L172-L207; `blend/message/src/reward/activity.rs` L41-L65 | `BlendingTokenEvaluation::new`, `verify_and_build` |
| `services/blend/src/core/mod.rs` L578-L640, L681-L685, L705-L711, L1252-L1265, L1760-L1770, L2176-L2200, L2207-L2283 | node epoch info, verifier inputs, `EpochInfo::new`, proposal copies, token collection sites |
| `services/blend/src/core/settings.rs` L47-L77 | node-side `Q_C`, `Q_W`, `Q_L` |
| `services/blend/src/membership/chain.rs` L205-L215 | node-side `BlendEpochState` from the ledger's `EpochState` |
| `services/blend/src/edge/mod.rs` L406-L414 | edge nodes send transactions on the PoW branch and proposals on the leader branch too |
| `services/chain/chain-leader/src/blend.rs` L50-L60; `nodes/node/binary/src/generic_services/blend/pol.rs` L53-L96 | every proposal goes through Blend; winning slots feed the leader branch |
| `nodes/node/binary/src/api/routes.rs` L42; `nodes/node/binary/src/api/handlers.rs` L744-L760 | the only transaction path into Blend is `POST /blend/transactions/disperse` |
| `utils/src/tokio/stream.rs` L8-L55; `utils/src/tokio/mod.rs` L42-L46 | `Buffered` pre-polls; `CancellableHandle` aborts on drop |
| `deployment/ceremony/genesis/{standalone,testnet,devnet}/deployment-template.yaml` L3, L6, L25-L28, L38-L46; `nodes/node/binary/src/config/deployment/settings.yaml` L3, L6, L25-L28, L38-L46 | the modelled parameters |

**Out of scope**
The PoQ circuit and its keys, Groth16 (`lb_groth16`, `ark-*`), rapidsnark, Blake2b and Poseidon2 are assumed correct. The ticket grind and the premium reward (report #97 LB-001), the `d_blend` ease on an idle chain (report #97 LB-003), the threshold rule itself and the byte-boundary effect (report #110 LB-001, S-001, S-002), the income accounting (report #110 LB-002, issue #158), the SDP declaration lifecycle (reports #53, #92), and the network-layer relaying of leader- and PoW-backed messages (whether a message reaches its addressee at all).

**Assumptions**
The Proof of Selection addresses each layer of a leader- or PoW-backed message to a uniformly random core node (`blend-protocol.md` §Proof of Selection; `selection_randomness = zkhash(sk, index, period_nonce)` in `proof-of-quota.md` §Constraints step 5), so the expected number of such tokens per node is the network total divided by `N`. Every proposed block, canonical or not, produces one data message; forks add a few percent and are ignored. `f = 1/20` (standalone) and `1/30` (testnet) blocks per slot, one round per one-second slot, as in report #110. The number of transactions dispersed through Blend per epoch is not fixed by the protocol; it is modelled at 0, at the template's reference load (`target_transactions_per_block: 2`, all through Blend) and at ten times that.

## 3. Method

- Manual review of the in-scope paths, working through sub-issue #159 under parent #8.
- Spec conformance against `blend-protocol.md` §Leadership Quota (`Q_L = β_D(1+R_D)`, one quota per proof of leadership), §Proof of Work Quota (`Q_W = β_max`, one solution per message), §Blend Difficulty and `proof-of-work.md` §Blend Difficulty (`d_blend` from the load of epoch `N-2`, fixed with the nonce), §Activity Proof (`χ` counts `Q_C^Total` only; the proof holds for any true PoQ), §Activity Threshold, and `proof-of-quota.md` (three branches, private selector, `pow_quota` public input). The code matches the spec on every one of these.
- Automated tooling, with versions: `python3` 3.x with the closed-form model of report #110 Appendix C extended by the two token sources (Appendix B). No Rust build was needed: every quantity entering the model is a configuration value or a closed-form count read from the code, and the ledger-level simulation of report #110 already established that the model matches the real `Rewards` state machine within sampling error.
- Dynamic testing: none.

**Checklist items of #159**

| # | Item | Result |
|---|---|---|
| C1 | Leader-branch tokens per node per epoch vs `Q_C` at `N = 128 … 2048` | `10·k·β·(1+R_D)/N`: 300/N (standalone), 2400/N (testnet); a constant 5.0% / 6.7% of `Q_C` at every `N` (table below) |
| C2 | PoW-branch solutions per epoch at `base_difficulty: 19` and the retargeted `d_blend`; tokens per node | the count is demand-driven, not difficulty-driven: one solution per transaction dispersed through Blend, plus two eagerly mined and discarded per node per epoch (S-002); tokens per node `X·β/N`; `d_blend` changes the cost per solution, not the count, until mining throughput (`2 solutions in flight`, about 50 s each on a Pi 5 core at base difficulty) binds |
| C3 | Re-run the #110 model with `Q_C + leader + PoW` | standalone `N_max` stays 127 with every source; testnet 878 → 947 (leader) → 999 (leader + PoW at reference load) → 1023 (ten times the reference load); the power-of-two knees are unchanged (tables below, Appendix B) |
| C4 | Same collector for all branches; `EpochInfo::new` uses the ledger's `Q_C` and `N` | confirmed: one `HashSet<BlendingToken>` per epoch fed from every decapsulation; the node verifies incoming PoQs with the same core, leader and PoW inputs the ledger later uses for the attested epoch; `Q_C`, `N`, `θ` and the randomness match (checked-and-ruled-out table) |

**Token sources per node per epoch (C1, C2)**

`B = 10·k` blocks per epoch (`config.rs` L145-L166; 300 standalone, 1200 testnet), `L = B·β·(1+R_D)/N` leader tokens, `W = X·β/N` PoW tokens with `X = 2·B` at the reference load.

| Set | `N` | `Q_C` | `L` | `W` (ref. load) | `L/Q_C` | `W/Q_C` |
|---|---|---|---|---|---|---|
| standalone | 128 | 47 | 2.34 | 4.69 | 0.050 | 0.100 |
| standalone | 256 | 24 | 1.17 | 2.34 | 0.049 | 0.098 |
| standalone | 512 | 12 | 0.59 | 1.17 | 0.049 | 0.098 |
| standalone | 1024 | 6 | 0.29 | 0.59 | 0.049 | 0.098 |
| standalone | 2048 | 3 | 0.15 | 0.29 | 0.049 | 0.098 |
| testnet | 128 | 282 | 18.75 | 18.75 | 0.066 | 0.066 |
| testnet | 256 | 141 | 9.38 | 9.38 | 0.066 | 0.066 |
| testnet | 512 | 71 | 4.69 | 4.69 | 0.066 | 0.066 |
| testnet | 1024 | 36 | 2.34 | 2.34 | 0.065 | 0.065 |
| testnet | 2048 | 18 | 1.17 | 1.17 | 0.065 | 0.065 |

Why the ratio is constant: `Q_C = E·β/N` and `L = E·f·β·(1+R_D)/N`, so `L/Q_C = f·(1+R_D)` (1/20 on standalone, 2/30 on testnet) independent of `N`; likewise `W/Q_C = f·(X/B)`. With the spec's own parameters (`E = 648 000`, `β = 3`, `f = 1/30`, `R_D = 0`) the ratio is 3.3%.

Where the counts come from: a leader draws one winning slot per data message and `message_quota = β·(1+R_D)` proofs from it (`leader/mod.rs` L99-L109, `provers/mod.rs` L24-L30); the node queues `1 + R_D` copies of each proposal (`core/mod.rs` L1261-L1263) and each copy is encapsulated under `β` layers (`send.rs` L201-L222), so one proposed block spends exactly one slot's quota and yields `β·(1+R_D)` tokens network-wide. A transaction is encapsulated on the PoW branch only (`send.rs` L283), under `β` layers, from a solution worth `pow_quota = β` proofs (`pow/mod.rs` L109, L155-L161; ledger `mod.rs` L291-L292), and reaches Blend only through `POST /blend/transactions/disperse` (`routes.rs` L42) or an edge node's queue (`edge/mod.rs` L408-L410); the mempool path `add_tx` does not touch Blend.

**Eligibility with the extra tokens (C3)**

`P(node) = 1 − (1 − p)^{T}` with `p = P(Binomial(ε, ½) ≤ 𝒜)` and `T` the node's token count; threshold and `ε` as in report #110 (they do not see `L` or `W`, `epoch.rs` L172-L186).

| Set | `N` | `𝒜` | `p` | core only | + leader | + leader + PoW (ref.) | + leader + PoW (10×) |
|---|---|---|---|---|---|---|---|
| standalone | 127 | 5 | 0.10506 | 0.995 | 0.996 | 0.998 | 1.000 |
| standalone | 128 | 4 | 0.03841 | 0.841 | 0.855 | 0.879 | 0.977 |
| standalone | 200 | 4 | 0.03841 | 0.691 | 0.709 | 0.741 | 0.910 |
| standalone | 256 | 3 | 0.01064 | 0.226 | 0.236 | 0.255 | 0.405 |
| standalone | 512 | 2 | 0.00209 | 0.025 | 0.026 | 0.028 | 0.050 |
| standalone | 1000 | 2 | 0.00209 | 0.012 | 0.013 | 0.014 | 0.025 |
| standalone | 1024 | 1 | 0.00026 | 0.002 | 0.002 | 0.002 | 0.003 |
| testnet | 878 | 5 | 0.10506 | 0.991 | 0.993 | 0.995 | 1.000 |
| testnet | 1000 | 5 | 0.10506 | 0.982 | 0.986 | 0.989 | 0.999 |
| testnet | 1024 | 4 | 0.03841 | 0.756 | 0.777 | 0.797 | 0.911 |
| testnet | 2000 | 4 | 0.03841 | 0.506 | 0.529 | 0.550 | 0.705 |
| testnet | 2048 | 3 | 0.01064 | 0.175 | 0.185 | 0.195 | 0.281 |
| testnet | 5000 | 2 | 0.00209 | 0.017 | 0.018 | 0.019 | 0.027 |

| Set | core only (#110) | + leader | + leader + PoW (ref.) | + leader + PoW (10×) |
|---|---|---|---|---|
| standalone `N_max` at `P ≥ 0.99` | 127 | 127 | 127 | 127 |
| testnet `N_max` at `P ≥ 0.99` | 878 | 947 | 999 | 1023 |

The testnet gain is inside the flat region between the 512 and 1024 knees, where `P` sits just under 0.99 and 7-13% more tokens lift it over; it never crosses a knee. The tokens a node would need to be back at `P ≥ 0.99` just past each knee are 118 at `𝒜 = 4`, 431 at `𝒜 = 3`, 2 200 at `𝒜 = 2` and 17 750 at `𝒜 = 1` (Appendix B), against `Q_C + L = 49` at standalone `N = 128`, 25 at 256, 13 at 512 and 6 at 1024, and 38 at testnet `N = 1024`, 19 at 2048. The shortfall is 2.4×, 17×, 170× and 2 800× respectively; leader and PoW traffic supplies 5-17%. `P(no node at all eligible)` per epoch moves from 0.21 to 0.17 at standalone `N = 2000` and from 0.86 to 0.85 at `N = 5000` (report #110 LB-002 unchanged). This is the byte-boundary and threshold-step behaviour already described in #110 S-002, not a new effect.

**Checked and ruled out (C4 and the premises of the issue)**

| Check | Evidence | Result |
|---|---|---|
| Leader- and PoW-branch tokens are collected into the same collector as core tokens | every decapsulation returns its tokens to `collect_current_epoch_tokens` (`core/mod.rs` L2245) or `collect_old_epoch_tokens` (L2279-L2281) or the old collector directly (L2195-L2199); the collector is one `HashSet<BlendingToken>` (`reward/mod.rs` L62-L79) with no branch information in the token; the backend verified the incoming PoQ with `PoQVerificationInputsMinusSigningKey { core, leader, pow }` (`core/mod.rs` L681-L685) | one collector, three branches |
| The node keeps tokens the ledger would accept, and only those | node: `compute_activity_proof` keeps every token with `distance ≤ 𝒜` (`reward/mod.rs` L128-L145 via `epoch.rs` L102) and submits the minimum; ledger: `verify_proof` applies the same `token_evaluation.evaluate` (`target_epoch.rs` L117-L122) after the same PoQ + PoSel checks (`activity.rs` L50-L59); both `BlendingTokenEvaluation::new(Q_C, N, θ)` from `core_quota(rounds, F_C, β, N)` with `N = membership.size()` (node, `core/mod.rs` L705-L711, `settings.rs` L48-L52, L132-L144) and `N = providers.size()` (ledger, `current_epoch.rs` L154-L156) | identical decision on both sides |
| The ledger verifies a leader- or PoW-branch token with the public inputs the sender used | ledger: the target epoch's verifier is `create_proof_verifier(current_reward_epoch_state.leader_input, .pow_input, zk_root, core_quota)` (`current_epoch.rs` L158-L163) where `CurrentEpochState::new(epoch_state)` (L43-L51) takes `utxos.root()`, `nonce`, `lottery_0/1`, `blend_pow_difficulty` from the `EpochState` of the attested epoch (`mod.rs` L271-L298); node: `BlendEpochState { nonce, aged: utxo_merkle_root(), lottery_0, lottery_1, pow_difficulty: blend_pow_difficulty }` from the same `EpochState` (`membership/chain.rs` L206-L213, `cryptarchia/mod.rs` L190-L192), `message_quota = β·(1+R_D)` (`settings.rs` L61-L77 vs ledger `mod.rs` L265-L269), `pow_quota = β` (`settings.rs` L54-L59 vs ledger L291-L292) | same inputs; a token the node accepted at message time verifies on the ledger |
| `d_blend` seen by provers equals `d_blend` the ledger verifies with | one value per epoch, frozen with the nonce at the snapshot slot from the load of epoch `N-2` (`cryptarchia/mod.rs` L117-L140, `blend_difficulty.rs` L44-L81); read by the node from the same `EpochState` | same value; matches `proof-of-work.md` §Blend Difficulty |
| Leader- and PoW-branch tokens reach only some nodes ("unevenly distributed", issue text) | the sender needs stake or a solution, the receiver is `expected_index(membership_size)` from the PoSel of a fresh key (`send.rs` L235-L238; `selection_randomness = zkhash(sk, index, period_nonce)`), uniform over the membership | receivers are uniform; every node gains `L + W` in expectation, staked or not |
| PoW solutions per epoch depend on `d_blend` | mining is on demand: `create_proof_stream` is `repeat_with(spawn_solution_proofs)` buffered to 2 (`pow/mod.rs` L125-L139), consumed one solution per `pow_quota` proofs, i.e. one per transaction message; `d_blend` scales the expected candidates per solution (`2^19` at base, `solve_puzzle` L76-L81) and therefore latency, not the number of messages | count is `X` (transactions dispersed) plus 2 per node (S-002); no token depends on `d_blend` |
| `Q_W`, `Q_L` and blocks per epoch match the spec | `Q_W = β_max` (spec) = `num_blend_layers` (code, the only `β`); `Q_L = β_D(1+R_D)` both sides; `N_b = 10·k` (`config.rs` L133-L166; `proof-of-work.md` §Parameters `EXPECTED_BLOCKS_PER_EPOCH = 10 k`) | no deviation |
| `χ` should count leader and PoW tokens | `blend-protocol.md` §Activity Proof defines `ε` and `χ` over `Q_C^Total`; `token_count_bit_len(Q_C, N)` (`epoch.rs` L174-L186) does the same | code matches spec; adding `L + W` to `χ` would raise it by `log₂(1 + f(1+R_D) + f·X/B) < 0.25` bit, i.e. change nothing at the shipped parameters (S-001) |
| A proposal sent in the last slots of an epoch loses its leader tokens | the node drops queued proposals and in-flight mining at rotation (`core/mod.rs` L1742-L1748); a message of epoch `e` received during the transition period is decapsulated by the old processor and its tokens go to the old collector (L2176-L2200) | at most one proposal's tokens per node per epoch; no effect on the model |

## 4. Findings

None. Every checklist item resolves to code that matches the specification, and the quantitative question of the issue is answered in §3: leader- and PoW-branch tokens are a constant 5-17% of `Q_C` and do not move the boundaries of report #110 LB-001 across any power-of-two knee. Two observations made on the way are recorded as suggestions.

## 5. Suggestions (non-security)

### S-001 · State in the spec that leader- and PoW-branch tokens count towards a node's activity proof but not towards `χ`

| | |
|---|---|
| Target | `blend-protocol.md` §Activity Proof, §Activity Threshold; `blend/message/src/reward/epoch.rs:L172-L186` |

§Activity Proof accepts any token whose PoQ is true, which includes the leader and PoW branches, and defines `ε` and `χ` from `Q_C^Total` alone with the wording "the expected number of blending tokens generated during an epoch". The two statements are consistent only if the reader knows that the other two branches are deliberately left out. Suggested sentence for §Activity Threshold: "`χ` counts core tokens only. Leader-branch and PoW-branch tokens are accepted by the activity proof and are excluded from `χ`; they add `f·(1+R_D)` and `f·X/B` of `Q_C` to a node's tokens, which raises the eligibility of every node equally." Including them in `χ` instead would change nothing at the shipped parameters (`log₂(1.15) < 0.25` bit) and would make `χ` depend on the epoch's traffic, so the exclusion is the right choice and should be stated. This is the numeric complement to report #110 S-003: the fix to LB-001 has to come from `Q_C`, not from the other branches.

### S-002 · Every core and edge node mines two Blend PoW solutions and proves their PoQs at each epoch start with no transaction to spend them on

| | |
|---|---|
| Target | `blend/provers/src/provers/pow/mod.rs:L38, L125-L139, L144-L169`; `utils/src/tokio/stream.rs:L25-L39`; `blend/provers/src/provers/core_leader_and_pow/mod.rs:L79-L82`; `blend/provers/src/provers/leader_and_pow/mod.rs:L58` |

`RealPowProofsGenerator::new` is called for every epoch's cryptographic processor (core and edge). It wraps `stream::repeat_with(spawn_solution_proofs)` in `Buffered::new(_, BUFFER_SIZE = 2)`, and `Buffered` pre-polls the inner `buffered(2)` stream so that two `spawn_solution_proofs` futures are created at once (`stream.rs` L25-L36); each one spawns a task immediately (`pow/mod.rs` L152) that mines a solution (`2^19` Poseidon2 evaluations in expectation at `base_difficulty: 19`, about 50 s on one Pi 5 core per report #97) and then proves `pow_quota` PoQs from it (L155-L162, one Groth16 proof each). Nothing consumes them unless a transaction is dispersed through Blend, and at the next rotation the generator is dropped and `CancellableHandle` aborts the tasks (`utils/src/tokio/mod.rs` L42-L46), which also discards any completed solution and its proofs. On a node with no dispersed transactions this is 2 solutions and 2 proofs per epoch of wasted work, network-wide `2N` solutions per epoch, and it does not produce a single blending token; on a loaded chain `d_blend` hardens by up to `2×` per epoch (`blend_difficulty.rs` L57-L58) and the two pool threads stay busy for correspondingly longer. The eager buffer makes sense for the leader branch, where the first proof is on the critical path of a block proposal; for the PoW branch the transaction can wait for the solution anyway (`core/mod.rs` L1474-L1488 already queues it). Start the first search on the first `get_next_proof` (a plain `buffered(2)` without the pre-poll, or `BUFFER_SIZE = 1` with lazy start), or gate the generator on a non-empty transaction queue.

### S-003 · The `d_blend` load counts the reward protocol's own transactions

| | |
|---|---|
| Target | `ledger/src/lib.rs:L280-L285`; `ledger/src/mantle/pow/blend_difficulty.rs:L62-L66`; `proof-of-work.md` §Blend Difficulty |

`txs_in_block` counts every applied Mantle transaction, which includes the `N` `SDP_ACTIVE` transactions the Blend reward protocol itself requires per epoch and every `CLAIM_POW_REWARD`. With `target_transactions_per_block: 2` and `B = 300` blocks per epoch (standalone), a network of `N = 600` core nodes reaches the reference load from its own active messages alone, and at `N = 2000` the load is `3.3×` the reference, so `d_blend` is hardened to `BASE/√3.3` by traffic that has nothing to do with Blend admission demand. This is what `num_transactions(b)` in `proof-of-work.md` §Blend Difficulty says, so the code is correct; the spec should either say that protocol-internal transactions are counted on purpose or define the load over user transactions only. Not a security issue at the template's network sizes; recorded because the same count decides how expensive S-002's wasted mining is.

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

## Appendix B — Closed-form model with leader- and PoW-branch tokens

`threshold159.py`, the model of report #110 Appendix C with `T = Q_C + L + W` in place of `Q_C` for the per-node count and `N·Q_C + B·β·(1+R_D) + X·β` for the network-wide count. `python3`, no dependencies.

```python
import math

def params(N, E, layers, theta=1):
    Q_C = math.ceil(E * 1.0 * layers / N)                 # core/src/blend/mod.rs
    T = N * Q_C
    chi = math.ceil(math.log2(T + 1))                     # token_count_bit_len
    eps = 8 * math.ceil(chi / 8)
    nu = math.ceil(math.log2(N + 1))
    thr = max(0, chi - nu - theta)
    return Q_C, T, chi, eps, nu, thr

def cdf(bits, d):
    return sum(math.comb(bits, k) for k in range(0, min(d, bits) + 1)) / 2.0**bits

def p_node(tokens, p_one):
    return 1.0 - (1.0 - p_one) ** tokens

# (name, E rounds/epoch, beta = num_blend_layers, k, R_D = data_replication_factor, txs/block at reference load)
sets = [("standalone", 6_000, 1, 30, 0, 2), ("testnet", 36_000, 1, 120, 1, 2)]

for name, E, beta, k, R_D, tpb in sets:
    B = 10 * k                       # expected_blocks_per_epoch
    X = tpb * B                      # transactions per epoch at the reference load
    for label, extra in (("core only", lambda N: 0.0),
                         ("core+leader", lambda N: B*beta*(1+R_D)/N),
                         ("core+leader+PoW(ref load)", lambda N: B*beta*(1+R_D)/N + X*beta/N),
                         ("core+leader+PoW(10x load)", lambda N: B*beta*(1+R_D)/N + 10*X*beta/N)):
        nmax = None
        for N in range(2, 20000):
            Q_C, T, chi, eps, nu, thr = params(N, E, beta)
            if p_node(Q_C + extra(N), cdf(eps, thr)) < 0.99:
                nmax = N - 1
                break
        print(f"{name:10} {label:28} N_max = {nmax}")

for name, E, beta, k, R_D, tpb in sets:
    B = 10 * k
    for N in (128, 256, 512, 1024, 2048):
        Q_C, T, chi, eps, nu, thr = params(N, E, beta)
        p1 = cdf(eps, thr)
        need = math.log(0.01) / math.log(1 - p1)
        L = B*beta*(1+R_D)/N
        print(f"{name:10} N={N:5} Q_C={Q_C:4} thr={thr} tokens needed={need:8.1f} Q_C+L={Q_C+L:.1f}")
```

Output (excerpt; the eligibility and ratio tables of §3 are the other sections of the same run):

```
== Largest N with P >= 0.99 for every smaller N (as in #110), by token source ==
standalone core only                    N_max = 127
standalone core+leader                  N_max = 127
standalone core+leader+PoW(ref load)    N_max = 127
standalone core+leader+PoW(10x load)    N_max = 127
testnet    core only                    N_max = 878
testnet    core+leader                  N_max = 947
testnet    core+leader+PoW(ref load)    N_max = 999
testnet    core+leader+PoW(10x load)    N_max = 1023

== Extra tokens per node needed to restore P >= 0.99 at the first failing N ==
standalone N=  128 Q_C=  47 thr=4 P(one)=0.03841 tokens needed=   117.6  (Q_C+L=49.3; extra needed=68.2 per node = 8735 extra blend messages network-wide per epoch)
standalone N=  256 Q_C=  24 thr=3 P(one)=0.01064 tokens needed=   430.7  (Q_C+L=25.2; extra needed=405.5 per node = 103815 extra blend messages network-wide per epoch)
standalone N=  512 Q_C=  12 thr=2 P(one)=0.00209 tokens needed=  2200.6  (Q_C+L=12.6; extra needed=2188.1 per node = 1120288 extra blend messages network-wide per epoch)
standalone N= 1024 Q_C=   6 thr=1 P(one)=0.00026 tokens needed= 17750.9  (Q_C+L=6.3; extra needed=17744.6 per node = 18170477 extra blend messages network-wide per epoch)
standalone N= 2048 Q_C=   3 thr=0 P(one)=0.00002 tokens needed=301802.1  (Q_C+L=3.1; extra needed=301799.0 per node = 618084320 extra blend messages network-wide per epoch)
testnet    N= 1024 Q_C=  36 thr=4 P(one)=0.03841 tokens needed=   117.6  (Q_C+L=38.3; extra needed=79.2 per node = 81147 extra blend messages network-wide per epoch)
testnet    N= 2048 Q_C=  18 thr=3 P(one)=0.01064 tokens needed=   430.7  (Q_C+L=19.2; extra needed=411.5 per node = 842806 extra blend messages network-wide per epoch)

== P(no provider at all eligible) with leader tokens included (network-wide token count) ==
standalone N= 1000 core only=3.52e-06  with L+W=5.36e-07
standalone N= 2000 core only=2.11e-01  with L+W=1.67e-01
standalone N= 5000 core only=8.58e-01  with L+W=8.47e-01

== Spec parameters (E=648000, beta=3, R_D=0 per spec; blocks 21600) ==
spec N=  8191 Q_C=  238 L=    7.9 L/Q_C=0.033 thr=7 eps=24 core=0.9996 +L=0.9997
spec N=  8192 Q_C=  238 L=    7.9 L/Q_C=0.033 thr=6 eps=24 core=0.9336 +L=0.9393
spec N= 10000 Q_C=  195 L=    6.5 L/Q_C=0.033 thr=6 eps=24 core=0.8916 +L=0.8993
```
