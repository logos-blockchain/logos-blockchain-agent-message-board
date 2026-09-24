# Audit Report — Block-reward constants against the shipped genesis stakes: the emission model is inert on every template, the token unit is defined nowhere, and the fee regime pays the last fee rather than the average

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/201`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `273658c765e0be1eb373883c19e7f1fb0320170a` — component(s): `ledger/src/lib.rs` (`compute_block_rewards`, the reward constants), `ledger/src/cryptarchia/mod.rs` (`from_utxos`, `update_epoch_state`, the fee window), `ledger/src/cryptarchia/stake.rs`, `ledger/src/config.rs`, `zk/proofs/pol/src/lottery.rs`, `core/src/mantle/transactions/gas.rs`, `tools/blockchain-tools/src/bin/genesis.rs`, `tools/blockchain-tools/src/genesis/distribution.rs`, `deployment/ceremony/genesis/{testnet,devnet,standalone}/`
Specs: `https://github.com/logos-co/logos-lips` @ `d788723992a805b395f377de6e6cf59859b47168` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `block-rewards.md` (all three in full); `analysis-block-reward-parameter-calibration.md` (§The Inferred Total Stake, §The Burning Rate Average Factor, §Maximum Emission Rate, by section); `overview-cryptoeconomics.md` §Minimum Stake, §Gas; `analysis-static-minimum-stake-estimation-for-service-declaration-protocol.md` §Overview, §Minimum Stake, §Minimum Stake in FIAT Terms; `bedrock-genesis-block.md` §Initial Stake Distribution, §Initial Proof of Work Reward Pool, §Initial Epoch State; `proof-of-work.md` §Reward Pool; `storage-markets.md` §Parameters; `execution-market.md` §Parameters (all by section)
Date: `2026-09-24` — author: `Claude Code (research agent)` — status: `final`

---

## 1. Summary

- Overall assessment: the observation of #89 S-004 holds at `273658c7` and is worse than stated. `compute_block_rewards` implements the integer reference of `block-rewards.md` with `STAKE_TARGET = 3·10^9`, while every shipped genesis template seeds the inferred total stake at `2,968,004,000,000,000` (testnet and devnet) or `100,201,000,000,000` (standalone), between `3·10^4` and `10^6` times the target. The emission factor `A_t` is therefore `0` at genesis on every template, the reserve-release branch never executes, and it cannot start executing until the fees pooled over a 120-block window exceed about `2.8·10^11` units on testnet, roughly 445 times what a full block can pay at genesis prices. The reason is not a bug in either arithmetic but the absence of a token unit: no constant in the node, the wallet, the FFI or the specifications says how many ledger units make one LGO, the block-rewards specification counts in whole LGO with `S_cap = 10^10`, the storage-market specification calls a ledger price of `1` "1 LGO per byte", the minimum-stake analysis derives `10^5` LGO where the templates set `10^9` units, and the genesis templates size their proof-of-work pool by the minimum stake rather than by the `5/1000 · S_cap` the genesis specification requires. Separately, the specification contradicts itself on what the fee-regime reward is: its prose and pool accounting distribute the window average `R̄_t`, its integer rewrite and reference code distribute the last block's fee `D_{1,t}`, and the node follows the latter.
- Findings: 0 critical · 0 high · 0 medium · 1 low · 1 informational
- Key themes: "an integer without a unit", "a bootstrap mechanism that no shipped network can enter", "two halves of one specification disagree and the code picked one".
- Must-fix before launch: LB-001 is a configuration matter, not a code defect, but it has to be settled before any genesis ceremony that is meant to run the emission model: decide the unit, restate the five reward constants (and `min_stake`, the PoW pool seed and the genesis stakes) in it, and add a genesis-time check that the seeded stake is below `STAKE_TARGET` unless that is intended.

Answers to the five checklist items, in order:

1. **Saturation with the actual genesis (§4.1).** Confirmed by tracing the ceremony tool and the ledger's genesis constructor, not by running them (this sandbox could not build the node; the arithmetic is deterministic and is shown in full). `distribute` (`tools/blockchain-tools/src/genesis/distribution.rs` L41-L53) emits one note per `stakeholders.yaml` entry plus the faucet note, and `run_ceremony` (`bin/genesis.rs` L260-L262) writes the faucet's `zk_id` into `cryptarchia.faucet_pk`; `LedgerState::from_utxos` (`ledger/src/cryptarchia/mod.rs` L743-L749) sums every note except the faucet's. The sums are `2,968,004,000,000,000` (testnet: 4 notes of `10^14`, 8 of `2·10^14`, 4 provider notes of `10^9`, 121 notes of `8·10^12`), the same for devnet (identical amounts, different keys), and `100,201,000,000,000` for standalone. With `sum_fees = 0` at block 1, `a_numerator = min(A_SCALE, sat(3·10^9 + 0 − total_stake)) = 0` (`ledger/src/lib.rs` L411-L414), `reward_numerator = 62,500·0 + 657·(1.2·10^8 − 0)·0 = 0` (L416-L423), `blend_reward = 0`, `leader_reward = 0 + tips` (L428-L435). Expected `0, 0, 0`; obtained `0, 0, 0` on all three.
2. **The unit (§4.2).** There is none. `Value` is `u64` (`core/src/mantle/ledger.rs`), no `decimals`, `denomination` or `10^10` constant exists in any Rust, TypeScript or C source of the repository, the genesis gas prices are the bare integer `1` (`core/src/mantle/transactions/gas.rs` L5-L12) with a doc comment quoting the storage-market specification's "1 LGO/gas", and the end-to-end test vocabulary calls one ledger unit "1 LGO" throughout (`tests/src/cucumber/steps/**`). The specifications do not agree with one another: `block-rewards.md` counts in whole LGO (`S_cap = 10^10`, "10 billion LGO", L165, L449), `storage-markets.md` says a ledger price of `1` is "1 LGO per permanently stored byte" (L120, L139), and the minimum-stake analysis puts the stake at `0.001 % · S_TGE = 10^5` LGO (L57-L61) where the templates set `min_stake.threshold = 10^9`. If one ledger unit is one LGO, the templates distribute 300 times the hard cap; if one LGO is `10^8` units, the reward constants are `10^8` too small and the templates' `10^9` minimum stake is `10^4` times too small. A genesis note of `10^14` is therefore either `10^14` LGO (ten thousand times the supply) or `10^6` LGO, depending on a decision nobody has written down.
3. **Base-unit constants and overflow (§4.3).** If the constants are in whole LGO and one LGO is `b = 10^d` units, the integer rewrite carries over with `STAKE_TARGET_b = 3·10^9 · b`, `A_SCALE_b = 1.2·10^8 · b`, `INFLATION_NUMERATOR_b = 62,500 · b`, `INFLATION_DENOMINATOR` and `FEE_AVG_NUM` unchanged (the fee term is already homogeneous in the unit), and `reward = (62,500·b·A' + 657·(A_SCALE·b − A')·D_{1,t}) / (657·A_SCALE·b)`. In `u128` the product `657 · A_SCALE·b · D_{1,t}` is the binding term: with `D_{1,t}` bounded only by `u64` it fits for `d ≤ 8` (`1.4·10^38 < 3.4·10^38`) and overflows at `d = 9`; with `D_{1,t}` bounded by the supply it fits up to `d = 9`, which is also where `S_cap` itself (`10^19`) stops fitting `u64`. `d = 8` is the largest safe choice without restructuring the formula. If instead the templates are wrong, a testnet allocation scaled by `1/(3·10^6)` with `min_stake = 10^5` (the analysis's own number) seeds `D ≈ 9.9·10^8`, a third of the target, puts `A_t = 1` from block 1 with a 95-unit per-block release, and the lottery is unaffected because it depends only on the ratio of note value to `D` (`zk/proofs/pol/src/lottery.rs` L131-L139); the #44 report's `29·D` caveat needs every note below `2.9·10^10`, which such an allocation satisfies by five orders of magnitude.
4. **Last fee versus window average (§4.4, LB-002).** Confirmed: the reference Python uses `last_pooled_fee = pooled_fees_window[-1]` (`block-rewards.md` L544, L553), the integer derivation substitutes `R_block = D_{1,t}` into equation (1) (L447, L510-L511), and the node reads `get_fee_from_index(window_index)`, the fee of the block being applied (`lib.rs` L419-L423). The prose says the opposite three times: equation (1) itself (L199), the sentence that "the recycled component distributes the average pooled reward `R̄_t`, rather than the single-block fee `R_block`, which smooths it across the window `T`" (L213), and the pool accounting `R_t = (1 − A_t)·R̄_t + ι_t` (L246). The specification means the average: the whole of §Pool Accounting and Supply Dynamics, added in revision 1.1.0, is built on it, and the pool `P_t` exists precisely to fund the difference `R̄_t − D_{1,t}` in a block whose fees are below the average. The integer section is the part that is wrong, and the node inherits the error. The practical consequence today is small because the node also has no pool (#89 LB-001): paying `R̄_t` without `P_t` could pay out more than a block collected, so the last-fee form is the only one the current ledger can conserve tokens under. Fixing the spec's integer section without implementing `P_t` would trade one deviation for another.
5. **Interaction with #188 / #147 and overflow at `10^15` (§4.5).** None, as expected. `total_stake` is read in exactly two places, `compute_block_rewards` (saturating `u128`, L411-L414) and `compute_lottery_values` (512-bit `BigUint`, `lottery.rs` L131-L139), plus the inference step (`stake.rs` L27-L70, `i128` with a `u64::MAX` unit test at L155-L169). Nothing overflows at `2.97·10^15`; the largest intermediate is `657 · 1.2·10^8 · D_{1,t} ≤ 1.5·10^30`. The genesis sum itself is unchecked `u64` arithmetic (`mod.rs` L748), which #44 already reports; the shipped templates total `2.97·10^15`, four orders of magnitude below the wrap.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `ledger/src/lib.rs` L63-L97, L387-L449, L527-L560 | the five reward constants, `compute_block_rewards`, where it is called |
| `ledger/src/cryptarchia/mod.rs` L64-L107, L225-L245, L300-L349, L367-L440, L615-L630, L738-L796 | `EpochState::total_stake`, the fee window, epoch transitions and inference, `from_utxos` |
| `ledger/src/cryptarchia/stake.rs` L27-L70 | `total_stake_inference` arithmetic and bounds |
| `ledger/src/config.rs` L22-L24, L44-L49, L350-L366 | lottery constants accessor, expected blocks per epoch, `pow_fee_share` |
| `zk/proofs/pol/src/lottery.rs` L20-L139 | `LotteryConstants`, `compute_lottery_values` precision |
| `core/src/mantle/transactions/gas.rs` L1-L13 | genesis gas prices |
| `tools/blockchain-tools/src/bin/genesis.rs` L226-L267; `src/genesis/distribution.rs` L15-L53, L81-L122 | how a template becomes a genesis block and a `faucet_pk` |
| `deployment/ceremony/genesis/{testnet,devnet,standalone}/{stakeholders,faucet,providers,deployment-template}.yaml` | the shipped allocations and parameters |
| `ledger/src/lib.rs` L3267-L3324 | the two existing `compute_block_rewards` unit tests |

**Out of scope**

- The stake-inference algorithm's correctness (#44, #634), the blend reward distribution downstream of `add_blend_income` (#89), the leader voucher pool (#639), and the PoW reward pool's own controllers (`proof-of-work.md`; only its genesis seed is compared here). `astro_float`, `num_bigint`, `serde_yaml` are assumed correct.
- Whether any deployed network uses the shipped templates unchanged; the compose deployments generate configuration outside this repository (#644 §4.2).

**Assumptions**

- Facts from issue #19, re-verified at `273658c7` where used: release builds do not set `overflow-checks`, so the `u64` sum at `mod.rs` L748 wraps silently in production; everything in `compute_block_rewards` is `u128` and either saturating or bounded (§4.5).
- The specifications at `d788723` are the reference. Where they disagree with each other (§4.2, §4.4) the report says which part it believes is authoritative and why, and files the disagreement under Suggestions for upstream.
- The ceremony is run with the shipped templates and `run_ceremony`, so `faucet_pk` is set. A hand-assembled config with `faucet_pk` unset or mismatched would count the `u64::MAX` faucet note in `total_stake` and wrap; that path is #44's finding, not repeated here.

## 3. Method

- Manual review of the in-scope paths, working through issue #201 (parent #7) after reading the three documents listed in the header in full and the named sections of the other six. The line numbers cited in the issue (`a805329f`) were re-derived at `273658c7`: the constants moved from L64-L81 to L63-L97 and `compute_block_rewards` from L389-L443 to L390-L449, with the PoW refill (L398-L405) added in between.
- Spec conformance against `block-rewards.md` §Parametrization, §Block Rewards, §Pool Accounting and Supply Dynamics, §Float Precision for Implementation and the reference Python; `bedrock-genesis-block.md` §Initial Proof of Work Reward Pool, §Initial Epoch State; `analysis-static-minimum-stake-estimation-for-service-declaration-protocol.md` §Minimum Stake.
- Prior reports read: `processed/89-blend-income-accounting.md` (LB-001 and S-004, the origin), `processed/44-leader-threshold-stake-edge-cases.md` (lottery precision, the genesis sum, the `29·D` corner case), `processed/147-epoch-state-synthesis-cost.md` and `processed/188-recovery-state-write-cost.md` (for item 5).
- Automated tooling: none. Dynamic testing: none. Item 1 asks for a test that builds each genesis and prints the values; the sandbox could not compile the node during this session, so §4.1 derives them by hand from the templates and the constructor, with every input quoted. The follow-up in §6 asks for the test to be added to the tree, where it belongs anyway.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | The reward constants and every shipped genesis template are in different units, so the emission factor is pinned at 0 and the reserve-release branch is dead code on every network the repository can build | Economic / Incentive | Low | — | Open |
| LB-002 | Spec deviation: the fee-regime reward is the applied block's own fee, not the window average the specification's prose and pool accounting define | Economic / Incentive | Informational | — | Open |

### 4.1 Item 1: the genesis stake on each template, and what `compute_block_rewards` makes of it

**How a template becomes `total_stake`.** `run_ceremony` (`bin/genesis.rs` L226-L267) loads `stakeholders.yaml` into `Vec<StakeHolderInfo>` (`{zk_id, stake: Value}`), `providers.yaml` and `faucet.yaml` (`{zk_id, funds: Value}`), and calls `distribute` (`distribution.rs` L81-L122), which builds one `Note` per stakeholder entry and appends one for the faucet (L42-L46, bounded by `Outputs::try_new` at 255 outputs, L48-L49), and one `SDPDeclareOp` per provider whose `zk_id` matches a stakeholder note (L97-L119). The notes go into the genesis `Transfer` with no inputs; the faucet's key goes into `cryptarchia.faucet_pk` (L260-L262). At node start `LedgerState::from_genesis_tx` (`mod.rs` L716-L736) rejects any input and calls `from_utxos` (L738-L796), which computes:

```rust
let total_stake = utxos
    .utxos()
    .iter()
    .filter(|(_, (utxo, _))| config.faucet_pk.is_none_or(|fpk| utxo.note.pk != fpk))
    .map(|(_, (utxo, _))| utxo.note.value)
    .sum::<Value>()
    .max(1);
```

and stores it in both `epoch_state` and `next_epoch_state` (L772, L782), exactly as `bedrock-genesis-block.md` §Initial Epoch State item 3 says: "the initial estimate of total stake will be the total tokens distributed at genesis" (L336). Later epochs replace it with the density-inferred value (`mod.rs` L304-L307, L371-L380), which starts from this seed and moves by the measured block density, so on a healthy network it stays at this order of magnitude.

**The templates.**

| Template | Stakeholder notes | Sum, excluding the faucet | Faucet note (excluded) | `min_stake.threshold` | PoW `reward_pool_genesis` |
|---|---|---|---|---|---|
| testnet | 4 × `10^14`, 8 × `2·10^14`, 4 × `10^9` (the providers), 121 × `8·10^12` (one key) | **2,968,004,000,000,000** | `18,446,744,073,709,551,615` (`u64::MAX`) | `10^9` | `3·10^11` |
| devnet | same amounts, different keys | **2,968,004,000,000,000** | `u64::MAX` | `10^9` | `3·10^11` |
| standalone | `10^14`, `10^11`, `10^11`, `10^9` | **100,201,000,000,000** | (faucet file present, same shape) | `10^9` | `3·10^11` |

The devnet and testnet sums: `4·10^14 + 1.6·10^15 + 4·10^9 + 9.68·10^14 = 2.968004·10^15`. The 121 notes of `8·10^12` are all paid to the same `zk_id`, which is what the genesis specification suggests for a stakeholder who wants to hide their allocation (L54); they still count once each. The faucet note is the entire remaining `u64` range and is excluded from `D` by the filter, but it is a real note in the ledger: the sum of all genesis notes exceeds `u64::MAX`, which is why any code that sums the whole UTXO set must do so in `u128` (the ledger does not, outside the filtered sum: #44 §S).

**The computation at block 1, zero fees.** From `lib.rs` L390-L435 with `total_fee_burned = 0`:

| Step | testnet / devnet | standalone |
|---|---|---|
| `pow_refill = 0 · 10 / 100` (L401) | 0 | 0 |
| `fee_window[1] = 0` (L404-L405); `sum_fees` (L410) | 0 | 0 |
| `STAKE_TARGET + FEE_AVG_NUM·sum_fees` | `3,000,000,000` | `3,000,000,000` |
| `.saturating_sub(total_stake)` (L413) | `sat(3·10^9 − 2.968·10^15) = 0` | `sat(3·10^9 − 1.002·10^14) = 0` |
| `a_numerator = min(0, A_SCALE)` (L414) | 0 | 0 |
| `reward_numerator = 62,500·0 + 657·(1.2·10^8 − 0)·0` (L416-L423) | 0 | 0 |
| `blend_reward` (L428-L430) | 0 | 0 |
| `leader_reward` (L431-L435) | `0 + tips = 0` | 0 |

So `epoch_state.total_stake, a_numerator, blend_reward, leader_reward` = `(2,968,004,000,000,000, 0, 0, 0)`, `(2,968,004,000,000,000, 0, 0, 0)` and `(100,201,000,000,000, 0, 0, 0)`. The issue's expectation is met on every template.

**Not only at block 1.** `a_numerator` becomes positive only when `FEE_AVG_NUM · sum_fees > total_stake − STAKE_TARGET`, that is when the fees pooled over the 120-block window (net of the 10 % PoW refill) exceed `(2.968·10^15 − 3·10^9) / 10,512 ≈ 2.82·10^11` units on testnet, `9.25·10^9` on standalone. A block at genesis prices (execution and storage prices both `1`, `gas.rs` L9-L12) can pay at most `3,193,460 + 2,097,152 ≈ 5.3·10^6` units in mandatory fees, so 120 full blocks pool `6.4·10^8`, 445 times short on testnet and 15 times short on standalone. The execution base fee has to rise by that factor, and stay there for a full window, before the reserve release contributes a single unit. On the shipped templates the emission model has one regime, `A_t = 0`, and the reward of every block is its own fee (LB-002).

### LB-001 · The reward constants and every shipped genesis template are in different units, so the emission factor is pinned at 0 and the reserve-release branch is dead code on every network the repository can build

| | |
|---|---|
| Severity | Low |
| Difficulty | — (not an attack; a configuration and specification gap) |
| Category | Economic / Incentive |
| Target | `ledger/src/lib.rs:L63-L97` (the constants), `L411-L414` (saturation); `deployment/ceremony/genesis/{testnet,devnet,standalone}/stakeholders.yaml`, `deployment-template.yaml` L56, L90; `core/src/mantle/transactions/gas.rs:L5-L12` |
| Status | Open |

**Description**

`block-rewards.md` derives its five integer constants from `S_cap = 10^10`, "10 billion LGO" (L165), `D_{0,target} = 3·10^9` (L170, L448), `I_max = 1 %`, `T = 120` and one block per 30 s (L442-L451): `STAKE_TARGET = 3·10^9`, `A_SCALE = 1.2·10^8`, `FEE_AVG_NUMERATOR = 10,512`, `INFLATION_NUMERATOR / INFLATION_DENOMINATOR = 62,500 / 657` (L535-L539). The node copies them verbatim (`lib.rs` L69-L86) and applies them to `epoch_state.total_stake` and to the fee window, both of which are sums of `Value = u64` note values and gas costs. The constants are meaningful only if one ledger unit is one LGO. The genesis templates then distribute `2.968·10^15` units to non-faucet stakeholders, three hundred times `S_cap` under that reading, plus a faucet note of `u64::MAX`, and set the SDP minimum stake at `10^9` units where the analysis that fixes it (`analysis-static-minimum-stake-estimation-for-service-declaration-protocol.md` L57-L61, L113-L123) gives `0.001 % · S_TGE = 10^5` LGO. The PoW reward pool seed shows the templates are not derived from `S_cap` at all: `bedrock-genesis-block.md` L76-L83 and `proof-of-work.md` L136, L161 require `5/1000 · S_cap` (`5·10^7` under the whole-LGO reading, `5·10^15` at `10^8` units per LGO), and the template's `3·10^11` is explained in its own comment as "sized to put 200 nodes over the minimum stake" (`deployment-template.yaml` L51-L56).

No source fixes the unit. The repository has no decimals constant in Rust, TypeScript or C (`grep -rn 'decimals\|denomination\|10_000_000_000\|1e10'` finds nothing relevant); `Value` is a bare `u64`; the test suite's step definitions call one unit "1 LGO" (`tests/src/cucumber/steps/transactions/steps/wallets.rs` L98-L216 and siblings); the storage-market specification calls the genesis price `1` "1 LGO per permanently stored byte" (`storage-markets.md` L120, L139), which the node's doc comment repeats (`gas.rs` L7-L8). Under the whole-LGO reading the templates are impossible (they exceed the hard cap) and the constants are right; under a `10^8`-unit reading the templates are plausible (`2.97·10^7` LGO distributed, `10` LGO minimum stake) and every economic constant in the ledger, the genesis price and the minimum stake are wrong by the same factor. Whichever is intended, the shipped combination disables the mechanism that `block-rewards.md` §Lifecycle Phases calls the Bootstrap Phase, and the numbers in `analysis-block-rewards.md` and the calibration document describe a network that this repository cannot instantiate.

**Exploit scenario**

None. The impact is that every network built from the shipped ceremony runs from block 1 in the fee regime with zero reserve release: leaders and blend providers are paid only their blocks' own fees (LB-002), which at genesis prices are a few thousand units per block, and the "3.33 % APY at target stake" of `block-rewards.md` L172 is never approached. On testnet and devnet this is presumably intended as a placeholder. On a mainnet genesis produced by the same tooling with the same reading, the economic model would be off by the unit factor from the first block, with no error, warning or test to say so: `from_utxos` accepts any stake and `compute_block_rewards` has no test at a realistic stake (the two tests at `lib.rs` L3267-L3324 exercise only the PoW refill on a single-note ledger).

**Recommendation**
- *Short term*: write the unit down once, in the specification (the `TokenValue` definition in `bedrock-v1.1-mantle-specification.md` §Notes is the natural place, or `common-cryptographic-components.md`) and once in the code (`pub const LGO_BASE_UNITS: Value = 10^d` next to `Value`), and restate every economic constant in base units: the five reward constants (§4.3 gives them for `10^d`), `GENESIS_STORAGE_GAS_PRICE` and `GENESIS_EXECUTION_GAS_PRICE`, `min_stake.threshold`, `reward_pool_genesis` and `epoch_reward_genesis`. Regenerate the three templates from `S_cap` (stakes below `D_{0,target}` if the bootstrap phase is meant to be exercised, PoW seed at `5/1000 · S_cap`). Add to `from_utxos`, or to the ceremony tool, a check that logs at `warn` (ceremony: refuses without `--allow-saturated-stake`) when the seeded `total_stake` is at or above `STAKE_TARGET`, since that silently disables the emission model. Add the unit test the issue asks for: build `LedgerState` from each template and assert the first-block rewards, so the next parameter change is visible.
- *Long term*: express the reward constants as `Config` fields derived at load time from `S_cap`, the unit and the consensus schedule (`expected_blocks_per_epoch`, `config.rs` L44-L49, already derives `f` from the schedule), rather than five literals that must be kept consistent by hand with `block-rewards.md`; then the same derivation can print the human-readable values (`STAKE_TARGET` in LGO, per-block cap in LGO) at startup.

**References**: `block-rewards.md` §Parametrization (L163-L177), §Float Precision for Implementation (L440-L561); `bedrock-genesis-block.md` §Initial Proof of Work Reward Pool (L74-L87), §Initial Epoch State (L328-L336); `proof-of-work.md` §Reward Pool (L159-L171); `analysis-static-minimum-stake-estimation-for-service-declaration-protocol.md` §Minimum Stake (L101-L123); `storage-markets.md` §Parameters (L115-L139); `analysis-block-reward-parameter-calibration.md` §The Inferred Total Stake (L108-L129); #89 S-004; #44 (the genesis sum, the `29·D` corner case); parent #7.

### 4.2 Item 2: where the unit could have been, and what each candidate implies

I looked for a denomination in five places the issue names and two it does not:

| Source | What it says | Implied unit |
|---|---|---|
| Node: `Value` (`core/src/mantle/ledger.rs`), `Note::value` | `u64`, no scaling constant anywhere in the workspace | undefined |
| Node: genesis gas prices (`gas.rs` L5-L12) | `1` and `1`, doc comment "`P_STR(0)` = 1 LGO/gas" | 1 unit = 1 LGO |
| Wallet and FFI (`wallet/`, `c-bindings/`) | balances and note values are `u64` passed through; no formatting layer | undefined |
| Tests (`tests/src/cucumber/steps/**`) | "{int} LGO" is the raw `u64` | 1 unit = 1 LGO |
| `block-rewards.md` (L165, L170-L173, L448-L449) | `S_cap = 10^10` "10 billion LGO", target `3·10^9`, reserve `10^9` LGO | constants in whole LGO |
| `storage-markets.md` (L120, L139) | initial price `1 LGO/gas`, "1 LGO per permanently stored byte" | 1 unit = 1 LGO |
| `analysis-static-minimum-stake-…` (L57-L61, L151-L153) | stake `= 0.001 % · S_TGE`; worked example with `S_max = 10^8` LGO gives `10^3` LGO | `10^5` LGO at `S_cap = 10^10` |
| Templates (`deployment-template.yaml` L90, L56) | `min_stake.threshold = 10^9`, PoW seed `3·10^11` | neither `10^5` nor `5·10^7`; consistent with `10^4` units per LGO for the stake and with nothing for the seed |
| Templates (`stakeholders.yaml`) | `2.97·10^15` distributed | at 1 unit = 1 LGO, 297 × `S_cap`; at `10^8`, `2.97·10^7` LGO |

No two rows agree once the templates are included. The specifications among themselves are consistent with "one ledger unit is one LGO" (block rewards, storage market, minimum stake all read that way; the minimum-stake analysis then puts the SDP stake at `10^5` units), and under that reading `u64` holds `1.8·10^9` times the hard cap, which is more headroom than needed but harmless. The templates alone are inconsistent with every reading, which is the strongest evidence that they are placeholders. What one genesis note of `10^14` is worth in LGO therefore has no answer today; the report treats the whole-LGO reading as the specifications' intent and the templates as unscaled.

### 4.3 Item 3: the constants in base units, the `u128` bound, and an allocation that enters the bootstrap phase

**If the constants are in whole LGO and one LGO is `b = 10^d` units.** Writing `D_0'`, `D_{1,τ}'` for the ledger quantities in base units (`D = D'/b`), the emission factor of `block-rewards.md` L492 becomes

`A_t = min(1, max(0, (3·10^9·b − D_0' + 10,512·Σ D_{1,τ}') / (1.2·10^8·b)))`,

because the stake term scales with `b` and the fee term `(α_a/I_max)·γ_t = 10,512·ΣD_1 / (1.2·10^8)` is a ratio of two token quantities and is unchanged. So:

| Constant | whole LGO (today) | base units, `b = 10^d` |
|---|---|---|
| `STAKE_TARGET` | `3,000,000,000` | `3·10^9 · b` |
| `A_SCALE` | `120,000,000` | `1.2·10^8 · b` |
| `FEE_AVG_NUM` | `10,512` | `10,512` (unchanged) |
| `INFLATION_NUMERATOR` | `62,500` | `62,500 · b` (per-block cap `95.13 · b` units) |
| `INFLATION_DENOMINATOR` | `657` | `657` |
| `reward` | `(62,500·A' + 657·(1.2·10^8 − A')·D_1) / (657·1.2·10^8)` | `(62,500·b·A' + 657·(1.2·10^8·b − A')·D_1') / (657·1.2·10^8·b)` |

`u128` bound of `reward_numerator` (`lib.rs` L416-L423): the second term dominates, `657 · 1.2·10^8 · b · D_1'`. With `D_1'` a `u64` `GasCost` (no other bound: fees are what a block collects, and the base fee is unbounded above), the product is `1.45·10^18 · b`, so:

| `d` | `657·A_SCALE·b·u64::MAX` | fits `u128` (`3.4·10^38`)? | `S_cap = 10^10·b` fits `u64`? |
|---|---|---|---|
| 0 (today) | `1.45·10^30` | yes | yes |
| 6 | `1.45·10^36` | yes | yes |
| 8 | `1.45·10^38` | yes | yes (`10^18`) |
| 9 | `1.45·10^39` | **no** | yes (`10^19 < 1.8·10^19`) |
| 10 | `1.45·10^40` | no | no |

`d = 8` is the largest exponent at which the current formula is safe with an unbounded per-block fee; `d = 9` works only if `D_1'` is bounded by the supply (`10^19`: `657·1.2·10^17·10^19 = 7.9·10^38`, still over) so it does not work at all without dividing before multiplying. `FEE_AVG_NUM · sum_fees` (L412) is `10,512 · 120 · u64::MAX = 2.3·10^25` at any `d` and is saturating anyway.

**If the templates are wrong.** Scaling the testnet allocation by `1/(3·10^6)` and setting `min_stake.threshold` to the analysis's `10^5`:

| Note class | today | proposed | count | subtotal |
|---|---|---|---|---|
| large stakeholders | `10^14` | `3.33·10^7` | 4 | `1.33·10^8` |
| large stakeholders | `2·10^14` | `6.67·10^7` | 8 | `5.33·10^8` |
| providers (service notes) | `10^9` | `3.33·10^5` (≥ `10^5`) | 4 | `1.33·10^6` |
| split notes | `8·10^12` | `2.67·10^6` | 121 | `3.23·10^8` |
| total `D_0` | `2.968·10^15` | | | **`9.9·10^8`**, a third of `STAKE_TARGET` |

At `D_0 = 9.9·10^8`: `δ_t = 0.67`, `A' = min(1.2·10^8, 3·10^9 − 9.9·10^8) = 1.2·10^8`, `A_t = 1`, reward per block `= 62,500/657 = 95` units (`57` blend, `38` leader), release `95 · 2880 · 365 = 10^8` per year, a 10 % return on the seeded stake, falling to the calibrated 3.33 % as `D` approaches the target (L172). The faucet note must then also be sized inside `S_cap − D_0` rather than at `u64::MAX`, or the "hard cap" of §Core Variables is fiction from genesis. The PoW seed becomes `5·10^7`; with `epoch_reward_genesis` scaled the same way (`25·10^6 → 8`) the pool still funds one claim per block for the ten epochs the template comments describe.

**Does the lottery still work at those stakes?** Yes: `compute_lottery_values` (`lottery.rs` L131-L139) divides 512-bit constants by `D` and `D^2`, and `check_winning` compares a note's ticket to `lottery_0 · v + lottery_1 · v^2`, a function of `v/D` only; scaling every note and `D` by the same factor leaves every win probability unchanged. The one corner case is #44's: the quadratic approximation is monotone only for `v ≤ 29·D`, i.e. here `v ≤ 2.9·10^10`; the largest proposed note is `6.7·10^7`. The inference step (`stake.rs` L27-L70) is also scale-free apart from its floor at `1`, which a `D` of `10^9` is far from. The two existing consensus tests that pin `D = 1` after an empty window (#44 LB-001) are unaffected by the allocation.

### 4.4 Item 4: which reward the fee regime pays

Three statements of the same equation in `block-rewards.md`:

| Where | Form | Fee term |
|---|---|---|
| Equation (1), L197-L201, and L213 ("distributes the average pooled reward `R̄_t`, rather than the single-block fee `R_block`, which smooths it across the window `T`") | `R_t = A_t · cap + (1 − A_t) · R̄_t` | window average |
| §Pool Accounting, L246 (`R_t = (1 − A_t)·R̄_t + ι_t`), L259 (`P_t = P_{t−1} + R_block − (1 − A_t)·R̄_t`) | | window average |
| §Float Precision, L447 (`R_block = D_{1,t}`), L510-L511 (`(1 − A_t)·R_block`), L524-L525, reference code L544 and L553 (`last_pooled_fee = pooled_fees_window[-1]`) | `Rewards_t = A_t · cap + (1 − A_t) · D_{1,t}` | last block's fee |

The node (`lib.rs` L416-L423) multiplies `(A_SCALE − a_numerator)` by `get_fee_from_index(window_index)`, the entry just written for the block being applied (L404-L405): the last block's fee. The average enters only through `sum_fees` in `A_t` (L410-L412), which is the `γ_t` term and is correct in both readings.

Which does the specification mean? The average. Revision 1.1.0 (L25) rewrote the model around a pool `P_t` whose only purpose is to bank `D_{1,t} − R̄_t` when a block collects more than the average and to fund the shortfall when it collects less (L256-L262); with the last-fee form `P_t` changes by `A_t · D_{1,t}` only and the smoothing that L213 and §Benefits ("built-in safeguards against manipulation through moving averages", L120) advertise does not exist. The integer section predates that rewrite in substance: it still substitutes `R_block` for `R̄_t` at L447 and its variable naming (`last_pooled_fee`) is deliberate, so the section was internally consistent with an older model and was not updated.

Consequences of the node's choice, in the fee regime every template is in: each block's `blend_reward` is `0.6·D_{1,t}` and its `leader_reward` is `0.4·D_{1,t}` plus tips; nothing is smoothed. Over an epoch the totals `Σ_t D_{1,t}` and `Σ_t R̄_t` differ only by the window's edge effects (about `120/L` of one epoch's fees, `L` the epoch length in blocks), so the amounts the blend and leader pools receive per epoch are nearly the same under both forms, and a leader gains nothing by stuffing fees into its own block since its reward is a voucher's share of a pool. The material difference is structural: the node has no `P_t` (#89 LB-001, fees are burned and rewards created), so it cannot pay `R̄_t` in a block whose fee is below the average without creating tokens that no fee backed. The last-fee form is the only one the current ledger can implement while conserving tokens; it is also not what the specification says.

### LB-002 · Spec deviation: the fee-regime reward is the applied block's own fee, not the window average the specification's prose and pool accounting define

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Economic / Incentive |
| Target | `ledger/src/lib.rs:L416-L423` (`reward_numerator` uses `get_fee_from_index(window_index)`); `block-rewards.md` L447, L510-L511, L544, L553 (the integer section) against L199, L213, L246, L259 (the model) |
| Status | Open |

**Description**

`block-rewards.md` defines the recycled component of the block reward as `(1 − A_t) · R̄_t`, the moving average of pooled fees over `T = 120` blocks, in its equation (1), its explanatory prose and its pool accounting. Its integer rewrite and reference implementation replace `R̄_t` by `D_{1,t}`, the current block's fee, and the node implements the rewrite. I believe the integer section is the side that is wrong: it contradicts the model that the same document's revision 1.1.0 introduced, and the pool `P_t` that revision defines has no role under the last-fee form.

**Exploit scenario**

None. In the fee regime the epoch totals paid to the blend and leader pools are the same to within the window's edge effect, and no participant can profit from the difference. The impact is that a reader of the specification and a reader of the code disagree about the reward of any single block, and that the "moving average" safeguard the specification advertises is not implemented.

**Recommendation**
- *Short term*: decide upstream (S-001). If the average is meant, the node needs `P_t` first (#89 LB-001's long-term recommendation) and then `reward_numerator` uses `sum_fees / WINDOW_SIZE` (with the window partially filled during the first 120 blocks, which the reference code also ignores) in place of `get_fee_from_index(window_index)`. If the last fee is meant, correct L199, L213, L246 and L259 of the specification and drop the smoothing claim at L120.
- *Long term*: keep one normative statement of the reward in the specification and generate the reference implementation's test vectors from it, so that the prose, the integer rewrite and the node are checked against the same numbers.

**References**: `block-rewards.md` §Block Rewards (L180-L236), §Pool Accounting and Supply Dynamics (L238-L278), §Float Precision for Implementation (L430-L561); `overview-cryptoeconomics.md` §Blend Service and Consensus Leaders; #89 LB-001 (no pool balance in the node).

### 4.5 Item 5: `total_stake` at `10^15` and the interaction with #188 / #147

`epoch_state.total_stake` is read by three consumers at `273658c7`:

| Consumer | Arithmetic | Largest intermediate at `D = 2.97·10^15` | Bound |
|---|---|---|---|
| `compute_block_rewards` (`lib.rs` L411-L423) | `u128`, `saturating_add/mul/sub` for `A'`, plain `*`/`+` for `reward_numerator` | `657 · 1.2·10^8 · D_1 ≤ 1.45·10^30` (`D_1 ≤ u64::MAX`) | `3.4·10^38` |
| `compute_lottery_values` (`lottery.rs` L131-L139) | `BigUint`, 512-bit constants, `t1 / D^2` | `D^2 = 8.8·10^30`, exact | unbounded; `t1/D^2 < p` so `checked_sub` cannot fail (#44) |
| `total_stake_inference` (`stake.rs` L27-L70) | `i128` | `D · 1000 · expected_density·1000 ≈ 2.97·10^15 · 10^3 · 10^7 = 3·10^25` | `1.7·10^38`; unit test at `u64::MAX` (L155-L169) |

Nothing overflows, and none of these depend on the recovery-state or epoch-state-synthesis paths that #188 and #147 audit: those serialise and rebuild `EpochState` (whose `total_stake` is a `u64` field, `mod.rs` L85) but do no arithmetic on it. The plain `*` at L416-L423 relies on the `u64` bound of `D_1` and the size of `A_SCALE`; it is safe today and stays safe up to `d = 8` (§4.3), after which it must become checked or restructured. The genesis sum at `mod.rs` L748 is the one unchecked `u64` operation on the path; #44 recommends a `checked_add` fold and the shipped templates are four orders of magnitude below the wrap, so it is not repeated as a finding here.

## 5. Suggestions (non-security)

### S-001 · Spec: reconcile the integer section of `block-rewards.md` with its model, and fix the unit in one place

Two edits to `block-rewards.md`: (a) replace `R_block = D_{1,t}` at L447 and the `(1 − A_t)·R_block` terms at L510-L511 and L524-L525 by `R̄_t = (1/T) Σ D_{1,τ}`, and `last_pooled_fee` at L544 and L553 by `sum_fees // T` (or state explicitly that the last fee is intended and delete L213's smoothing claim); (b) state the unit of every quantity in §Parametrization: whether `S_cap = 10^10` is whole LGO or the ledger's `TokenValue`, and if the former, the number of `TokenValue` units per LGO. `storage-markets.md` L120 and L139 ("1 LGO per byte" for a ledger price of `1`) and `analysis-static-minimum-stake-…` L57-L61 should then be checked against the same definition. The `TokenValue` definition in `bedrock-v1.1-mantle-specification.md` §Notes (`uint64`) is where the unit belongs, so that every document inherits it.

### S-002 · Genesis templates: derive the economic parameters from `S_cap`, and have the ceremony refuse a saturated stake

`deployment-template.yaml` carries `reward_pool_genesis: 300000000000` with a comment deriving it from the minimum stake, where `bedrock-genesis-block.md` L76-L83 and `proof-of-work.md` L161 fix it at `5/1000 · S_cap`, and `min_stake.threshold: 1000000000` where the analysis gives `0.001 % · S_TGE`. Once the unit is fixed, both are one-line derivations from `S_cap`; the ceremony tool (`bin/genesis.rs` `run_ceremony`) could compute them and check the stakeholder sum against `STAKE_TARGET` and the faucet against `S_cap − Σ stakes`, refusing (or warning loudly) when a template would start the chain outside the model. The faucet at `u64::MAX` should be sized inside the cap for the same reason.

### S-003 · Doc comments on the reward constants refer to `S_TGE`, which revision 1.1.0 of the specification removed

`lib.rs` L76-L84 documents `INFLATION_NUMERATOR` and `INFLATION_DENOMINATOR` as "`I_max` * `S_TGE` * `DELTA_t` / `f`" and "`MAX_INFLATION` * `TOKEN_GENESIS`", the revision 1.0.0 vocabulary. `block-rewards.md` L25 replaced `S_tge` by `S_cap` and the reserve model; the comments should follow, and the `WINDOW_SIZE = 120` at L63 should cite `T` and the 30-second block assumption (`Δ_t = 1/(365·2880)`, L176) that the two existing tests at L3267-L3324 do not exercise. A test that builds each shipped template's genesis and asserts `(total_stake, a_numerator, blend_reward, leader_reward)` at block 1, as the issue asks, would make the next constant or template change visible in CI.

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
