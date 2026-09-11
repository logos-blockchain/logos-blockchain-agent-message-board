# Audit Report — Blend activity threshold vs. per-node quota

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/110`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `blend/message/src/reward`, `core/src/blend`, `ledger/src/mantle/sdp/rewards/blend`, `nodes/node/binary/src/config/blend`, `deployment/ceremony/genesis/*/deployment-template.yaml`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `bedrock-service-declaration-protocol.md`, `bedrock-service-reward-distribution.md` (in full); `blend-protocol.md` §Minimal Network Size, §Rewarding (Overview and Protocol tiers), §Notation, §Global Parameters, §Core Quota, §Proof of Selection, §Activity Proof, §Activity Threshold, §Active Message, §Reward Calculation, §Rewarding Distribution Logic; `block-rewards.md` §Overview (reserve and pending pool) (by section)
Date: `2026-09-11` — author: `claude-fable-5.1` — status: `final`

The issue was filed against `a77c248f3`. Between that commit and `a805329f8` the in-scope files change only by the `derivative` → `educe` derive swap in `activity.rs` and the renaming of a diagnostic log target in `epoch.rs`, `reward/mod.rs` and `sdp/mod.rs`; every line cited below is identical at both commits.

---

## 1. Summary

- Overall assessment: confirmed at the ledger level. Driving the real `Rewards` state machine with `N` declared providers that each hold `Q_C` uniformly random tokens, the shipped standalone template pays the base reward to 100% of honest nodes at `N = 100`, 69% at `N = 200`, 12% at `N = 500` and 1.4% at `N = 1000` (40 simulated epochs each); the testnet template drops from 98% at `N = 1000` to 50% at `N = 2000` (10 epochs each) and, by the model the simulation confirmed, 1.7% at `N = 5000`. The collapse happens at powers of two (`N = 128`, `256`, `512`, `1024`, …) because the subtracted term `ν = ⌈log₂(N+1)⌉` steps by one bit there while `χ` is fixed by the epoch length and each step divides a token's chance by 3-5. The spec's condition `χ > ν + θ` is satisfied in every one of these cases and therefore does not protect the parameter choice. The largest network each template supports with `P(honest node paid) ≥ 0.99` is `N = 127` (standalone) and `N = 878` (testnet). When no submitted token clears the threshold, the epoch's blend income is discarded: it is neither carried into the next epoch nor returned to any pool (established by code reading; the simulated network sizes never produced such an epoch), and the spec does not define this case.
- Findings: 0 critical · 0 high · 1 medium · 1 low · 0 informational
- Key themes: "threshold subtracts `log₂ N` bits but each node only holds `E·β/N` tokens", "spec condition necessary, not sufficient", "undefined behaviour of the income when nobody is eligible"
- Must-fix before launch: LB-001 for any deployment expected to exceed `N = 127` on the standalone schedule or `N = 878` on the testnet schedule; at minimum document the supported `N` next to the template parameters.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `blend/message/src/reward/activity.rs` L71-L86 | `activity_threshold` = `χ − ν − θ`, saturating |
| `blend/message/src/reward/epoch.rs` L58-L75, L172-L207 | `BlendingTokenEvaluation::new` (digest width `⌈χ/8⌉` bytes, L72), `token_count_bit_len` (`χ`), `activity_threshold` (`ν`) |
| `blend/message/src/reward/token.rs` L37-L52, L93-L106 | ticket = `blake2b(token bytes)` truncated to the digest width; Hamming distance |
| `blend/message/src/reward/mod.rs` L124-L160 | node-side selection of the best token (`compute_activity_proof`) |
| `core/src/blend/mod.rs` L12-L36 | `Q_C = ⌈E·f·β / N⌉` |
| `ledger/src/mantle/sdp/rewards/blend/mod.rs` L119-L164, L235-L263 | `update_epoch` (what survives a transition), `RewardsParameters`, `core_quota_and_token_evaluation` |
| `ledger/src/mantle/sdp/rewards/blend/current_epoch.rs` L132-L146, L154-L156, L175-L185 | minimum network size gate, per-target-epoch evaluation parameters, income hand-over |
| `ledger/src/mantle/sdp/rewards/blend/target_epoch.rs` L79-L125, L185-L236 | proof acceptance (`HammingDistanceTooLarge`), payout, empty-epoch branch |
| `ledger/src/lib.rs` L88-L92, L419-L441 | the 60% blend share of each block reward and where the three shares go |
| `nodes/node/binary/src/config/blend/deployment.rs` L30-L34, L70-L84 | `rounds_per_epoch = slots_per_epoch`, `RewardsParameters` from the template |
| `consensus/cryptarchia-engine/src/config.rs` L117-L131; `consensus/cryptarchia-engine/src/time.rs` L278-L289 | `E = 10·⌊k/f⌋` slots |
| `services/blend/src/core/mod.rs` L705-L711; `services/blend/src/core/settings.rs` L48-L52, L132-L143 | node-side `EpochInfo::new` with the same `Q_C`, `N`, `θ` |
| `deployment/ceremony/genesis/{standalone,testnet,devnet}/deployment-template.yaml` L3-L4, L10, L15, L25-L28, L47-L49; `nodes/node/binary/src/config/deployment/settings.yaml` L3-L4, L10, L15, L25-L28 | the simulated parameters |

**Out of scope**
The Proof of Quota circuit and its keys, Groth16 (`lb_groth16`, `ark-*`), Blake2b and Poseidon2 are assumed correct. The ticket grind and the premium reward (report #97 LB-001, report #54), the PoW branch (report #97 LB-003), the withdrawal cut-off (report #76), the SDP declaration lifecycle and cap (reports #53, #92), the mempool admission of `SDPActive` (report #98), the network-layer meaning of "relaying", and the leader-branch tokens (they add to an honest node's tokens but are not counted in `χ`; ignoring them makes the numbers below slightly pessimistic for staked nodes).

**Assumptions**
Tokens received by an honest node are one per key index and their digests are independent and uniform, which is what the spec asserts ("the node does not control the process of generating blending tokens", `blend-protocol.md` §Activity Proof) and what the simulation reproduces with random preimages. Every node receives exactly its `Q_C` (the network-wide expectation; the actual count is Binomial and varies by `±√Q_C`). One round per one-second slot (`deployment.rs` L30-L34, `slot_duration: 1`). The templates' `minimum_network_size: 2` is used as shipped; report #92 S-005 already records that the spec says 32.

## 3. Method

- Manual review of the in-scope paths, working through sub-issue #110 under parent #8 (question 1 of the parent, "any path where a participant claims more than its share", is what LB-001 amounts to: the few eligible nodes split the whole epoch).
- Spec conformance against `blend-protocol.md` §Core Quota (`Q_C`, `Q_C^Total`), §Activity Proof (`ε`, the inclusive comparison), §Activity Threshold (`𝒜_ε = max(0, χ − ν − θ)`, the `χ > ν + θ` condition), §Reward Calculation (`R = I / (B + P)`), and against `bedrock-service-reward-distribution.md` §Service Reward Distribution (one note per rewarded `zk_id`, none for zero). The code matches the spec on every one of these; the findings are shared by both and are parameter and design problems.
- Automated tooling, with versions: `rustc 1.98.1`, `cargo 1.98.1` (the toolchain pinned by `rust-toolchain.toml`); `python3` 3.x with the closed-form model in Appendix C (no numpy). One test was added locally to `ledger/src/mantle/sdp/rewards/blend/mod.rs` (source in Appendix B) and run with `cargo test -p logos-blockchain-ledger --lib -- --ignored --nocapture issue110`.
- Dynamic testing: the ledger-level simulation of Appendix B, 40 epochs per `N` on the standalone parameters (`N = 100, 200, 500, 1000`) and 10 per `N` on the testnet parameters (`N = 200, 1000, 2000`). The test is a debug build (about 0.2 ms per token evaluation); the testnet `N = 5000` and spec `N = 10 000` rows did not complete within the time budget of this run and are given from the model, which the completed rows match within sampling error. No running network.

**Checklist items of #110**

| # | Item | Result |
|---|---|---|
| C1 | Simulate with `N` real providers and `Q_C` random tokens each on the standalone `RewardsParameters` | done, Appendix B; matches the closed-form model within sampling error (LB-001) |
| C2 | Decide the supported `N` per template, or derive the epoch length from the intended `N` | computed: standalone `N ≤ 127`, testnet `N ≤ 878` at `P ≥ 0.99`; `Q_C ≥ 100` gives `N ≤ 60` / `N ≤ 360`; text proposed in LB-001 |
| C3 | Evaluate a threshold rule from `Q_C` or from distinct nullifiers; raise the `χ` vs `ε` mismatch | S-001 (rule from `Q_C`, with numbers), S-002 (`χ` vs `ε`, quantified), distinct-nullifier rule discussed under S-001 |
| C4 | What happens to the epoch's blend income when no provider is eligible | discarded, never minted, not carried (LB-002), by code reading; the simulation's `paid ≤ INCOME` assertion held in every epoch but no zero-eligible epoch occurred at the sizes run |

**Parameters simulated**

| Set | Source | `k` | `f` | `E = 10·⌊k/f⌋` | `β` (`num_blend_layers`) | Epoch |
|---|---|---|---|---|---|---|
| standalone | `standalone/deployment-template.yaml`, `settings.yaml` | 30 | 1/20 | 6 000 | 1 | 100 min |
| testnet / devnet | `testnet/…`, `devnet/…` | 120 | 1/30 | 36 000 | 1 | 10 h |
| spec | `blend-protocol.md` §Global Parameters | — | — | 648 000 | 3 | 7.5 days |

All with `message_frequency_per_round = 1.0`, `activity_threshold_sensitivity θ = 1`, `data_replication_factor` irrelevant to `Q_C` (`R_C = 0`, `core/src/blend/mod.rs` L23-L25).

**Checked and ruled out**

| Check | Evidence | Result |
|---|---|---|
| Node and ledger disagree on the threshold or the digest width | ledger: `current_epoch.rs` L154-L156 → `core_quota_and_token_evaluation(providers.size())` (`blend/mod.rs` L245-L263); node: `core/mod.rs` L705-L711 with `membership.size()` and the quota from `settings.rs` L48-L52, both via `BlendingTokenEvaluation::new` | same parameters on both sides; the simulation submits the node's choice and the ledger accepts every one of them (`expect` at Appendix B never fires) |
| `activity_threshold` clamps at 0 rather than going negative | `activity.rs` L82-L85 `saturating_sub` twice | matches spec's `max(0, ·)`; at `𝒜 = 0` only an exact digest match qualifies, which the spec describes |
| The comparison is inclusive | `epoch.rs` L102 `distance <= self.activity_threshold`; `target_epoch.rs` L117-L122 | matches spec |
| `ε` computed from `χ` in bytes | `epoch.rs` L72 `div_ceil(8)`; `token.rs` L45-L49 hashes to that many bytes | matches spec's `⌈χ/8⌉·8`; the mismatch (S-002) is in the spec |
| `Q_C` rounding | `core/src/blend/mod.rs` L26-L30 `ceil` | matches spec (rounded up, `Q_C ≥ 1`) |
| `χ` uses the rounded `N·Q_C`, not `E·β` | `epoch.rs` L174-L178 `quota * num_core_nodes` | matches spec's `Q_C^Total = N·Q_C`; this is why `χ` steps from 13 to 14 at standalone `N = 5000` (`T = 10 000`) |
| Reward division when only one node is eligible | `target_epoch.rs` L203-L204: `I / (1 + 1)`, doubled at L209-L210 → the single node is paid the whole income | consistent with spec's `R = I/(B+P)`, `R(n) = 2R` |
| Anything else limits how many nodes are paid, or pays the unlucky ones | `finalize` L199-L215 iterates `submitted_proofs` only; nothing in `sdp/mod.rs` L176-L218 touches the amounts | nothing; an unlucky honest node gets 0 |
| Income of an epoch with no eligible proof survives the transition | `finalize` L189-L197 returns `(Self::new(), vec![])`; `update_epoch` L148-L159 drops `target_epoch_state` and builds the next target from `current_epoch_tracker.epoch_income` only (`current_epoch.rs` L181); no reserve or pool balance exists in the ledger to receive it (`lib.rs` L419-L441: leader share → `leaders.add_pending_rewards`, PoW share → `pow.add_reward_refill_rewards`, blend share → `sdp.add_blend_income` only) | dropped (LB-002); the Appendix B test asserts `paid ≤ INCOME` every epoch and prints "income carried" whenever a zero-eligible epoch occurs, which the completed sizes did not reach (model: `3.5·10⁻⁶` per epoch at standalone `N = 1000`) |

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Activity threshold steps down one bit at every power of two of `N` while each node's tokens fall as `1/N`; the shipped templates deny the base reward to most honest nodes beyond `N = 127` (standalone) and `N = 878` (testnet) | Economic / Incentive | Medium | — | Open |
| LB-002 | Epoch blend income is discarded when no submitted proof clears the threshold; neither carried over nor returned to a pool, and undefined in the spec | Economic / Incentive | Low | — | Open |

### LB-001 · Activity threshold steps down one bit at every power of two of `N` while each node's tokens fall as `1/N`; the shipped templates deny the base reward to most honest nodes beyond `N = 127` (standalone) and `N = 878` (testnet)

| | |
|---|---|
| Severity | Medium |
| Difficulty | — |
| Category | Economic / Incentive |
| Target | `blend/message/src/reward/activity.rs:L71-L86` (`activity_threshold`); `blend/message/src/reward/epoch.rs:L58-L75, L172-L207`; `core/src/blend/mod.rs:L12-L36`; `deployment/ceremony/genesis/standalone/deployment-template.yaml:L3, L25-L28`; `deployment/ceremony/genesis/testnet/deployment-template.yaml:L3, L25-L28`; `nodes/node/binary/src/config/blend/deployment.rs:L70-L84` |
| Status | Open |

**Description**
This is the ledger-level confirmation of report #97 LB-002. The threshold is

```rust
// blend/message/src/reward/activity.rs
82    token_count_bit_len            // χ = ⌈log₂(N·Q_C + 1)⌉        (epoch.rs L174-L185)
83        .saturating_sub(network_size_bit_len)   // ν = ⌈log₂(N + 1)⌉   (epoch.rs L193-L200)
84        .saturating_sub(activity_threshold_sensitivity)   // θ = 1
```

and a token qualifies when its `ε = 8·⌈χ/8⌉`-bit digest is within that many bits of the randomness digest (`epoch.rs` L72, L102). `N·Q_C` is `E·β` rounded up to a multiple of `N` (`core/src/blend/mod.rs` L26-L30), so `χ` is a constant of the deployment: 13 on the standalone schedule, 16 on the testnet schedule. Every doubling of `N` therefore removes one bit from the threshold, and `P(one token qualifies) = P(Binomial(ε, ½) ≤ 𝒜)` drops by a factor 3-5 per bit (16-bit digest: 0.227 → 0.105 → 0.038 → 0.011 → 0.0021 → 0.00026 for `𝒜 = 6 … 1`). At the same time each honest node holds `Q_C ≈ E·β/N` tokens, so its chance `1 − (1 − p)^{Q_C}` is squeezed from both sides. Simulated through the real `Rewards` state machine (Appendix B; model in Appendix C):

| Set | `N` | `Q_C` | `𝒜` (bits `ε`) | Model `P(node)` | Simulated eligible/epoch (mean, min-max) | Paid/epoch |
|---|---|---|---|---|---|---|
| standalone | 100 | 60 | 5 (16) | 0.999 | 100.0 (99-100) | 100.0 |
| standalone | 200 | 30 | 4 (16) | 0.691 | 138.6 (124-153) | 138.6 |
| standalone | 500 | 12 | 3 (16) | 0.120 | 59.9 (47-74) | 59.9 |
| standalone | 1000 | 6 | 2 (16) | 0.0125 | 13.5 (7-20) | 13.5 |
| testnet | 200 | 180 | 7 (16) | 1.000 | 200.0 (200-200) | 200.0 |
| testnet | 1000 | 36 | 5 (16) | 0.982 | 980.7 (976-985) | 980.7 |
| testnet | 2000 | 18 | 4 (16) | 0.506 | 1009.5 (989-1039) | 1009.5 |
| testnet | 5000 | 8 | 2 (16) | 0.017 | not completed in the time budget (model: 83) | — |
| spec | 10 000 | 195 | 6 (24) | 0.892 | not completed in the time budget (model: 8 916) | — |

The simulated counts are the number of providers whose best token the node-side selection (`compute_activity_proof`, `reward/mod.rs` L128-L145) accepts; the ledger accepted every one of them (`verify_proof`, `target_epoch.rs` L117-L122) and minted exactly that many notes (`finalize` L199-L215), so "paid" equals "eligible" and the rest received nothing for an epoch in which they relayed their full quota.

The collapse is at the powers of two (model, honest node with its full `Q_C`):

| Set | `N = 127` | `128` | `255` | `256` | `511` | `512` | `878` | `879` | `1023` | `1024` | `2047` | `2048` |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| standalone | 0.995 | 0.841 | 0.609 | 0.226 | 0.120 | 0.025 | 0.015 | 0.015 | 0.013 | 0.002 | 0.001 | 0.000 |
| testnet | 1.000 | 1.000 | 1.000 | 1.000 | 1.000 | 1.000 | 0.991 | 0.989 | 0.982 | 0.756 | 0.506 | 0.175 |

So the largest `N` such that every smaller network also keeps `P(honest node paid) ≥ 0.99` is **127 on the standalone template and 878 on the testnet template** (1023 at `≥ 0.98`); with the spec's own `E = 648 000`, `β = 3` it is 8191. The templates' own comment names "a target network of ~200 mining nodes" (`deployment-template.yaml` L48-L49); the testnet schedule supports that, the standalone schedule (`P = 0.69` at `N = 200`) does not, and the node binary's embedded `settings.yaml` is the standalone schedule.

The spec's guard, "Parameters must be chosen so that `χ > ν + θ`" (`blend-protocol.md` §Activity Threshold), holds in every row of both tables (the threshold is positive down to standalone `N = 2047`), so it is necessary and not sufficient. What the guard misses is that eligibility depends on the *per-node* token count `Q_C`, which the threshold formula never sees: `χ − ν = ⌈log₂(N·Q_C+1)⌉ − ⌈log₂(N+1)⌉ ≈ log₂ Q_C`, i.e. the threshold is roughly `log₂ Q_C − 1` bits out of `ε ≥ χ` bits, and `P(Binomial(ε,½) ≤ log₂ Q_C − 1)` shrinks far faster than `1/Q_C` once `log₂ Q_C` is small compared with `ε/2`. The rule only works while `Q_C` is large, i.e. while `N` is small compared with `E·β`.

Beyond the knee the epoch's whole blend income goes to the lucky few (`R = I/(B+P)`, `target_epoch.rs` L203-L204): on the standalone schedule an eligible node at `N = 1000` is paid about 74 times the fair share `I/N`, at `N = 2000` about 780 times; on testnet at `N = 5000` about 60 times. Combined with report #97 LB-001 (a grinder is eligible with certainty) the grinder is then one of the few paid at all.

**Exploit scenario**
Not an attack. A network started from the testnet template that grows to 2 000 declared core nodes pays about half of them per epoch, chosen by the lottery, and at 5 000 about 80; each of the others relayed 8-18 messages per epoch as instructed and receives nothing, while the income meant for `N` nodes is split among the winners. On the standalone schedule the same happens from about 130 nodes. Nothing distinguishes an honest unlucky node from a node that relayed nothing, so the reward stops selecting for the behaviour it is meant to pay.

**Recommendation**
- *Short term*: document the supported `N` in each template next to `num_blend_layers` (`deployment-template.yaml` L3) and `security_param` (L25): "the activity lottery pays honest nodes with `P ≥ 0.99` only up to `N = 127` core nodes at `security_param: 30`, `1/20` (standalone) and `N = 878` at `security_param: 120`, `1/30` (testnet); beyond that raise `security_param`, lower `slot_activation_coeff`, or raise `num_blend_layers`." Or size the schedule from the intended `N`: with the current rule `P ≥ 0.99` at `N = 200` needs `E·β ≥ 8 933` rounds, at `N = 1000` `E·β ≥ 42 494`, at `N = 5000` `E·β ≥ 2²⁰` (the last jump is S-002: `χ = 17` widens the digest to 24 bits). The simpler invariant `Q_C ≥ 100` (`E·β ≥ 100·N`) is within the safe region everywhere and is what the model in Appendix C recommends stating as a constraint on the parameters.
- *Long term*: replace the threshold rule by one derived from `Q_C` (S-001), which makes eligibility independent of `N` by construction, and state the constraint between `E`, `β` and `N` in `blend-protocol.md` §Activity Threshold together with what breaks when it is violated.

**References**: report #97 LB-002; `blend-protocol.md` §Core Quota, §Activity Threshold, §Reward Calculation; `overview-cryptoeconomics.md` §Blend Service ("only active nodes qualify for rewards").

### LB-002 · Epoch blend income is discarded when no submitted proof clears the threshold; neither carried over nor returned to a pool, and undefined in the spec

| | |
|---|---|
| Severity | Low |
| Difficulty | — |
| Category | Economic / Incentive |
| Target | `ledger/src/mantle/sdp/rewards/blend/target_epoch.rs:L185-L197` (`TargetEpochTracker::finalize`); `ledger/src/mantle/sdp/rewards/blend/mod.rs:L142-L162` (`update_epoch`); `ledger/src/mantle/sdp/rewards/blend/current_epoch.rs:L175-L185`; `ledger/src/lib.rs:L419-L441` |
| Status | Open |

**Description**
Checklist item 4. When the target epoch closes with no accepted proof, `finalize` returns no notes and a fresh tracker:

```rust
// ledger/src/mantle/sdp/rewards/blend/target_epoch.rs
189    if self.submitted_proofs.is_empty() {
194        "target epoch tracker finalized with no activity proofs. no rewards to distribute",
196        return (Self::new(), vec![]);
```

`update_epoch` then discards the `TargetEpochState` that held `epoch_income` (`blend/mod.rs` L148-L159) and the next target epoch is built with the *current* tracker's income only (`current_epoch.rs` L181). The blend share of each block reward is computed from the emission formula and handed to the SDP ledger as a number (`lib.rs` L419-L423, L440); unlike the leader share (`leaders.add_pending_rewards`, L436-L438) and the PoW share (L441) there is no balance it is debited from, so an unpaid blend income is simply never minted. Report #76 (C2) recorded the same for the case "nobody submits" as an edge case; LB-001 makes it a parameter regime: on the standalone schedule the probability that *no* token in the whole network qualifies is 21% per epoch at `N = 2000` and 86% at `N ≥ 5000` (Appendix C, `table_no_eligible`). The simulation ran up to `N = 1000` on that schedule, where the per-epoch probability is `3.5·10⁻⁶`, and did not observe one; the behaviour is established from the code path above, which has no other exit, and the run's `paid ≤ INCOME` assertion held in every epoch.

Neither `blend-protocol.md` §Reward Calculation (`R = I/(B+P)` is undefined at `B = 0`) nor `bedrock-service-reward-distribution.md` says what happens to `I` in this case. `block-rewards.md` §Overview describes the blend share as released from the reserve and distributed from the pending pool; an amount that is released and then never distributed is accounted for in neither.

**Exploit scenario**
Not exploitable by a third party; the effect is on supply accounting. Every epoch in which no node clears the threshold silently removes 60% of that epoch's block rewards from what the emission schedule says was released. On a standalone-schedule network of 2 000 nodes that is one epoch in five.

**Recommendation**
- *Short term*: define the case in the spec (S-003) and implement whichever rule it picks. The two obvious choices are to carry `epoch_income` into the next target epoch (add it to `current_epoch_tracker` at `blend/mod.rs` L148-L159 when `finalize` returns no notes) or to state explicitly that it stays in the reserve, in which case the ledger should not treat it as released (it currently has no reserve balance to return it to).
- *Long term*: with LB-001 fixed the case only arises when nobody submits, and the same rule covers it.

**References**: report #76 C2; `blend-protocol.md` §Reward Calculation; `bedrock-service-reward-distribution.md` §Service Reward Distribution; `block-rewards.md` §Overview.

## 5. Suggestions (non-security)

### S-001 · A threshold derived from `Q_C` keeps honest eligibility above 0.96 at every `N` and still discriminates lazy nodes; the current rule does neither

| | |
|---|---|
| Target | `blend/message/src/reward/activity.rs:L71-L86`; `blend-protocol.md` §Activity Threshold |

Checklist item 3. Issue #110 proposes choosing `𝒜` as the largest (in fact smallest sufficient) `d` with `Q_C · P(D_ε ≤ d) ≥ c`, i.e. a node holding its full quota expects at least `c` qualifying tokens. Evaluated with the model of Appendix C for `c = 3` (`P(node) ≥ 1 − e⁻³ ≈ 0.95` by Poisson approximation), against the current rule, and for a "lazy" node that holds only a fraction of `Q_C`:

| Set | `N` | `Q_C` | `𝒜_c` | `P(node)` full | lazy 10% | lazy 25% | lazy 50% | current `𝒜` | current `P(node)` | current lazy 10% |
|---|---|---|---|---|---|---|---|---|---|---|
| standalone | 100 | 60 | 5 | 0.999 | 0.49 | 0.81 | 0.96 | 5 | 0.999 | 0.49 |
| standalone | 200 | 30 | 5 | 0.964 | 0.28 | 0.54 | 0.81 | 4 | 0.691 | 0.11 |
| standalone | 500 | 12 | 7 | 0.998 | 0.40 | 0.79 | 0.95 | 3 | 0.120 | 0.01 |
| standalone | 1000 | 6 | 8 | 0.996 | 0.60 | 0.60 | 0.94 | 2 | 0.013 | 0.002 |
| testnet | 200 | 180 | 4 | 0.999 | 0.51 | 0.83 | 0.97 | 7 | 1.000 | 1.00 |
| testnet | 1000 | 36 | 5 | 0.982 | 0.28 | 0.63 | 0.86 | 5 | 0.982 | 0.28 |
| testnet | 2000 | 18 | 6 | 0.990 | 0.23 | 0.64 | 0.90 | 4 | 0.506 | 0.04 |
| testnet | 5000 | 8 | 7 | 0.984 | 0.40 | 0.64 | 0.87 | 2 | 0.017 | 0.002 |
| spec | 1000 | 1944 | 5 | 0.998 | 0.47 | 0.80 | 0.96 | 10 | 1.000 | 1.00 |
| spec | 5000 | 389 | 6 | 0.988 | 0.35 | 0.67 | 0.89 | 7 | 1.000 | 0.71 |

Two things the table shows beyond the fix itself. First, the current rule does not discriminate lazy nodes where it works: on the testnet schedule at `N ≤ 200` a node holding a *single* token is eligible with probability 0.40-0.77 (Appendix C, `table_lazy_current`), so the "minimal amount of work" the base reward is meant to certify (`blend-protocol.md` §Motivations) is one relayed message. The rule is mis-scaled in both directions, too loose at small `N` and impossible at large `N`, because it is anchored to `χ − ν ≈ log₂ Q_C` rather than to `Q_C`. Second, no rule over a Hamming lottery can work once `Q_C ≤ 3` (standalone `N ≥ 2000`): with `c = 3` the rule degenerates to `𝒜 = ε` (everyone qualifies) or to no valid `d` at all. `Q_C` has to be kept above a floor by the schedule regardless of the rule (LB-001).

Implementation cost is small: `𝒜_c` is a function of `(Q_C, ε, c)` computed once per epoch on both sides (`BlendingTokenEvaluation::new`, `epoch.rs` L58-L75), with the binomial CDF over at most 64 bits in integer arithmetic (`Σ C(ε,k) · Q_C ≥ c · 2^ε`), so determinism is not at risk. `c` replaces `θ` as the sensitivity parameter.

The other rule the issue names, a base reward for `≥ m` distinct nullifiers received, decouples eligibility from `N` completely but is a larger change: the active message carries one 230-byte token (`blend-protocol.md` §Active Message), so counting nullifiers means either submitting `m` tokens or proving possession of `m` in zero knowledge, and it only becomes meaningful after the ticket is bound to the nullifier (report #97 LB-001 / S-001), since today a node re-proving one witness has unboundedly many distinct tokens with one nullifier. It is the right long-term shape; the `Q_C` rule is the one that can ship.

### S-002 · Threshold is derived in `χ` bits but measured in `ε ≥ χ` bits; crossing a byte boundary costs up to 8 noise bits and makes the required epoch length non-monotone in `N`

| | |
|---|---|
| Target | `blend/message/src/reward/epoch.rs:L64-L72`; `blend-protocol.md` §Activity Proof, §Activity Threshold |

Reported in #97 as S-002; this quantifies it for the upstream discussion the issue asks for. `𝒜 = χ − ν − θ` is a distance budget in a `χ`-bit space, but the digest is `ε = 8·⌈χ/8⌉` bits and every extra bit is a fair coin that adds `½` to the expected distance and nothing to the budget. On the standalone schedule (`χ = 13`, `ε = 16`) measuring over 13 bits instead of 16 would give `P(node) = 0.99 / 0.43 / 0.07` at `N = 200 / 500 / 1000` against `0.69 / 0.12 / 0.01` today. The effect is worst just above a byte boundary: at `N = 2000` with `E·β = 100 000` (`χ = 17`, `ε = 24`, `𝒜 = 5`), `P(D_24 ≤ 5) = 0.0033` where `P(D_17 ≤ 5) = 0.072`, twenty times lower, and a node holding `Q_C = 50` tokens is eligible with probability 0.15. That is why the minimum `E·β` for `P ≥ 0.99` at `N = 2000` is about 286 000 (`2^18.1`) rather than the `~85 000` the trend from `N = 1000` suggests (Appendix C, `threshold110_b.py`). Either derive the threshold from `ε` (the S-001 rule does this automatically) or compare only the first `χ` bits of the digest. Code and spec agree, so this is a spec change first.

### S-003 · The spec should define the reward when `B = 0` and the constraint between `E`, `β` and `N`

| | |
|---|---|
| Target | `blend-protocol.md` §Activity Threshold, §Reward Calculation |

Two additions to the specification, following from LB-001 and LB-002:

- §Activity Threshold: replace "Parameters must be chosen so that `χ > ν + θ`" by a constraint that involves `Q_C`, for example `Q_C · P(D_ε ≤ 𝒜_ε) ≥ c` for a stated `c`, with the consequence when violated ("honest nodes holding their full quota are not paid"). The current sentence is satisfied by every configuration in which the reward fails.
- §Reward Calculation: state what happens to `I` when `B = 0` (carried into the next epoch's `I`, or retained by the reserve). Today `R = I/(B+P)` is undefined there and the implementation drops `I`.

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

## Appendix B — Ledger-level simulation

Added to `mod tests` in `ledger/src/mantle/sdp/rewards/blend/mod.rs` at `a805329f8`, directly before `struct AlwaysSuccessProofsVerifier`. It uses the module's own `AlwaysSuccessProofsVerifier` (the PoQ and PoSel are out of scope; the verifier reconstructs the token from the submitted bytes, so the ledger hashes exactly the bytes the node chose), the crate's `EpochState` constructor pattern from `test_utils.rs`, and `StdRng` with a fixed seed. For each `N` it declares `N` providers with distinct `provider_id`, `zk_id` and `DeclarationId`, then for each epoch gives every provider `Q_C` tokens with uniformly random `signing_key`, `ProofOfQuota` and `ProofOfSelection` bytes, selects the best with `BlendingTokenEvaluation::evaluate` (the function `compute_activity_proof` uses), submits it through `update_active`, adds a fixed income, and closes the epoch with `update_epoch`. Run: `cargo test -p logos-blockchain-ledger --lib -- --ignored --nocapture issue110`, `rustc 1.98.1`.

Output:

```
set          N   Q_C  thr epochs | eligible/epoch mean   min   max | paid/epoch | epochs with 0 eligible | income carried
standalone   100    60    5     40 |               100.0    99   100 |      100.0 |                      0 | n/a
standalone   200    30    4     40 |               138.6   124   153 |      138.6 |                      0 | n/a
standalone   500    12    3     40 |                59.9    47    74 |       59.9 |                      0 | n/a
standalone  1000     6    2     40 |                13.5     7    20 |       13.5 |                      0 | n/a
testnet      200   180    7     10 |               200.0   200   200 |      200.0 |                      0 | n/a
testnet     1000    36    5     10 |               980.7   976   985 |      980.7 |                      0 | n/a
testnet     2000    18    4     10 |              1009.5   989  1039 |     1009.5 |                      0 | n/a
(run stopped after the testnet N = 2000 row: the testnet N = 5000 and spec N = 10 000 rows did not complete within the time budget)
```

Source:

```rust
    /// Message-board issue #110: how many of `N` honest providers, each
    /// holding `Q_C` uniformly random blending tokens, obtain a base reward
    /// per epoch under the shipped `RewardsParameters`.
    ///
    /// Run with `cargo test -p logos-blockchain-ledger --lib -- --ignored
    /// --nocapture issue110`.
    #[test]
    #[ignore]
    fn issue110_base_reward_eligibility_simulation() {
        use std::sync::Arc;

        use lb_blend_message::reward::{ActivityProof as BlendActivityProof, BlendingToken};
        use lb_blend_proofs::{quota::PROOF_OF_QUOTA_SIZE, selection::PROOF_OF_SELECTION_SIZE};
        use lb_core::sdp::{Declaration, DeclarationId, Declarations, Locator};
        use lb_groth16::fr_from_bytes_unchecked;
        use lb_key_management_system_keys::keys::ZkPublicKey;
        use num_bigint::BigUint;
        use rand::{RngCore as _, SeedableRng as _, rngs::StdRng};

        use crate::UtxoTree;

        const INCOME: Value = 1_000_000;

        // (name, rounds_per_epoch, num_blend_layers, N values, epochs simulated)
        let sets: [(&str, u64, u64, &[usize], u32); 3] = [
            ("standalone", 6_000, 1, &[100, 200, 500, 1000], 40),
            ("testnet", 36_000, 1, &[200, 1000, 2000, 5000], 10),
            ("spec", 648_000, 3, &[10_000], 1),
        ];
        let config = create_service_parameters();

        let make_state = |provider_ids: &[ProviderId], epoch: u32, nonce: Fr| -> EpochState {
            let entries: HashMap<DeclarationId, Declaration> = provider_ids
                .iter()
                .enumerate()
                .map(|(i, provider_id)| {
                    let mut id = [0u8; 32];
                    id[..8].copy_from_slice(&(i as u64).to_le_bytes());
                    let declaration = Declaration {
                        service_type: ServiceType::BlendNetwork,
                        provider_id: *provider_id,
                        service_note_id: Fr::from(i as u64).into(),
                        locators: "/ip4/1.1.1.1/udp/0".parse::<Locator>().unwrap().into(),
                        zk_id: ZkPublicKey::new(BigUint::from(i as u64 + 1).into()),
                        created: 0.into(),
                        active: 2.into(),
                        withdraw_at: None,
                        nonce: 0,
                    };
                    (DeclarationId(id), declaration)
                })
                .collect();
            let active_declarations: Declarations =
                HashMap::from([(ServiceType::BlendNetwork, entries)]).into();
            EpochState {
                epoch: epoch.into(),
                nonce,
                utxos: UtxoTree::default(),
                total_stake: 0,
                lottery_0: Fr::ZERO,
                lottery_1: Fr::ZERO,
                blend_pow_difficulty: Fr::ZERO,
                active_declarations: Arc::new(active_declarations),
            }
        };

        println!();
        println!(
            "set          N   Q_C  thr epochs | eligible/epoch mean   min   max | paid/epoch | epochs with 0 eligible | income carried"
        );
        for (name, rounds, layers, ns, epochs) in sets {
            let params = RewardsParameters {
                rounds_per_epoch: rounds.try_into().unwrap(),
                message_frequency_per_round: PositiveF64::try_from(1.0).unwrap(),
                num_blend_layers: NonZeroU64::new(layers).unwrap(),
                data_replication_factor: 0,
                minimum_network_size: NonZeroU64::new(2).unwrap(),
                activity_threshold_sensitivity: 1,
            };
            for &n in ns {
                let mut rng = StdRng::seed_from_u64(110 + n as u64);
                let random_fr = |rng: &mut StdRng| {
                    let mut b = [0u8; 32];
                    rng.fill_bytes(&mut b);
                    fr_from_bytes_unchecked(&b)
                };
                let provider_ids: Vec<ProviderId> = (0..n)
                    .map(|i| {
                        let mut b = [0u8; 32];
                        b[..8].copy_from_slice(&(i as u64).to_le_bytes());
                        b[31] = 0x7a;
                        ProviderId(Ed25519Key::from_bytes(&b).public_key())
                    })
                    .collect();
                let (core_quota, token_evaluation) = params
                    .core_quota_and_token_evaluation(n as u64)
                    .unwrap();
                let q_c = core_quota.get();

                let epoch0 = make_state(&provider_ids, 0, random_fr(&mut rng));
                let epoch1 = make_state(&provider_ids, 1, random_fr(&mut rng));
                let (mut tracker, _) =
                    Rewards::<AlwaysSuccessProofsVerifier>::new(&params, &epoch0)
                        .add_income(INCOME)
                        .update_epoch(&epoch0, &epoch1, &config, &params);
                let mut current = epoch1;

                let mut eligible_counts = Vec::new();
                let mut paid_counts = Vec::new();
                let mut zero_epochs = 0usize;
                let mut income_carried = false;
                let mut previous_epoch_unpaid = false;
                for e in 1..=epochs {
                    let Rewards::WithTargetEpoch {
                        target_epoch_state,
                        current_epoch_state,
                        ..
                    } = &tracker
                    else {
                        panic!("target epoch must be set");
                    };
                    let target_epoch = target_epoch_state.epoch();
                    assert_eq!(target_epoch, Epoch::new(e - 1));
                    let randomness = current_epoch_state.epoch_randomness();

                    let mut eligible = 0usize;
                    for provider_id in &provider_ids {
                        // `Q_C` tokens, each a uniformly random digest preimage
                        // (the proof bytes are random; the signing key is the
                        // provider's, derived once, since ed25519 key derivation
                        // dominates a debug build).
                        let signing_key = provider_id.0;
                        let mut best: Option<(u64, BlendingToken)> = None;
                        for _ in 0..q_c {
                            let mut poq = [0u8; PROOF_OF_QUOTA_SIZE];
                            rng.fill_bytes(&mut poq);
                            let mut posel = [0u8; PROOF_OF_SELECTION_SIZE];
                            rng.fill_bytes(&mut posel);
                            let token = BlendingToken::new(
                                signing_key,
                                VerifiedProofOfQuota::from_bytes_unchecked(poq),
                                VerifiedProofOfSelection::from_bytes_unchecked(posel),
                            );
                            // Same function the node uses in `compute_activity_proof`.
                            if let Some(distance) = token_evaluation.evaluate(&token, randomness) {
                                let d = distance.value();
                                if best.as_ref().is_none_or(|(b, _)| d < *b) {
                                    best = Some((d, token));
                                }
                            }
                        }
                        if let Some((_, token)) = best {
                            let proof: blend::ActivityProof =
                                (&BlendActivityProof::new(target_epoch, token)).into();
                            tracker = tracker
                                .update_active(
                                    *provider_id,
                                    &ActivityMetadata::Blend(Box::new(proof)),
                                    &params,
                                )
                                .expect("ledger must accept the node's chosen token");
                            eligible += 1;
                        }
                    }
                    tracker = tracker.add_income(INCOME);
                    let next = make_state(&provider_ids, e + 1, random_fr(&mut rng));
                    let (new_tracker, utxos) =
                        tracker.update_epoch(&current, &next, &config, &params);
                    tracker = new_tracker;
                    current = next;
                    let paid: Value = utxos.iter().map(|u| u.note.value).sum();
                    assert_eq!(utxos.len(), eligible);
                    assert!(paid <= INCOME, "more than one epoch's income was paid");
                    if previous_epoch_unpaid && !utxos.is_empty() {
                        // If the unpaid income had been carried over, this epoch
                        // would pay from 2 * INCOME.
                        income_carried = paid > INCOME;
                    }
                    if eligible == 0 {
                        zero_epochs += 1;
                    }
                    previous_epoch_unpaid = eligible == 0;
                    eligible_counts.push(eligible);
                    paid_counts.push(utxos.len());
                }
                let mean =
                    eligible_counts.iter().sum::<usize>() as f64 / eligible_counts.len() as f64;
                let min = eligible_counts.iter().min().unwrap();
                let max = eligible_counts.iter().max().unwrap();
                let paid_mean = paid_counts.iter().sum::<usize>() as f64 / paid_counts.len() as f64;
                println!(
                    "{name:10} {n:5} {q_c:5} {:4} {epochs:6} | {mean:19.1} {min:5} {max:5} | {paid_mean:10.1} | {zero_epochs:22} | {}",
                    token_evaluation.activity_threshold().value(),
                    if zero_epochs > 0 {
                        if income_carried { "yes" } else { "no" }
                    } else {
                        "n/a"
                    },
                );
            }
        }
    }
```

## Appendix C — Closed-form model

`threshold110.py` (the tables quoted in §4 and §5) and `threshold110_b.py` (minimum `E·β` per `N`, boundaries, pay multiples). Both mirror the code: `Q_C = ⌈E·f·β/N⌉`, `χ = ⌈log₂(N·Q_C+1)⌉`, `ε = 8·⌈χ/8⌉`, `𝒜 = max(0, χ − ⌈log₂(N+1)⌉ − θ)`, one token's distance `~ Binomial(ε, ½)`, `P(node) = 1 − (1 − P(D_ε ≤ 𝒜))^{Q_C}`, `P(none) = (1 − P(D_ε ≤ 𝒜))^{N·Q_C}`.

```python
import math

def params(N, E, layers, theta=1):
    Q_C = math.ceil(E * 1.0 * layers / N)                 # core/src/blend/mod.rs
    T = N * Q_C
    chi = math.ceil(math.log2(T + 1))                     # token_count_bit_len
    eps = 8 * math.ceil(chi / 8)                          # digest bytes -> bits
    nu = math.ceil(math.log2(N + 1))
    thr = max(0, chi - nu - theta)                        # activity_threshold
    return Q_C, T, chi, eps, nu, thr

def cdf(bits, d):                                         # P(Binomial(bits, 1/2) <= d)
    return sum(math.comb(bits, k) for k in range(0, min(d, bits) + 1)) / 2.0**bits

def p_node(Q_C, p_one):
    return 1.0 - (1.0 - p_one) ** Q_C

def threshold_from_quota(Q_C, eps, c):                   # S-001 rule
    for k in range(0, eps + 1):
        if Q_C * cdf(eps, k) >= c:
            return k
    return -1
```

Model output for the current rule (excerpt; the full run also prints the S-001 and `χ`-only variants):

```
set        N      Q_C     T      chi eps nu thr  P(one)    P(node)   E[eligible]
standalone    100      60     6000  13  16  7   5  0.10506  0.9987       99.9
standalone    200      30     6000  13  16  8   4  0.03841  0.6912      138.2
standalone    500      12     6000  13  16  9   3  0.01064  0.1204       60.2
standalone   1000       6     6000  13  16 10   2  0.00209  0.0125       12.5
standalone   2000       3     6000  13  16 11   1  0.00026  0.0008        1.6
standalone   5000       2    10000  14  16 13   0  0.00002  0.0000        0.2
testnet       200     180    36000  16  16  8   7  0.40181  1.0000      200.0
testnet      1000      36    36000  16  16 10   5  0.10506  0.9816      981.6
testnet      2000      18    36000  16  16 11   4  0.03841  0.5059     1011.7
testnet      5000       8    40000  16  16 13   2  0.00209  0.0166       83.0
testnet     10000       4    40000  16  16 14   1  0.00026  0.0010       10.4
spec         1000    1944  1944000  21  24 10  10  0.27063  1.0000     1000.0
spec        10000     195  1950000  21  24 14   6  0.01133  0.8916     8915.6

== Largest N with P(honest node eligible) >= 0.99 ==
standalone N_max = 127 (Q_C = 48, thr = 5, P = 0.9951)
testnet    N_max = 878 (Q_C = 42, thr = 5, P = 0.9906)
spec       N_max = 8191 (Q_C = 238, thr = 7, P = 0.9996)

== P(no provider at all is eligible) per epoch ==
standalone   1000  3.5e-06      standalone   2000  0.21      standalone   5000  0.86
testnet     10000  3.1e-05
```
