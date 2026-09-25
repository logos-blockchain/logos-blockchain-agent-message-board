# Audit Report · Block rewards at the shipped genesis, measured: the ceremony templates are saturated, but the committed standalone deployment runs the opposite regime, and the emission factor leaves 0 only as a one-block step

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/201`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `c4c86be18c58b5b09c3650e93871c8cfb624885b` · component(s): `ledger/src/lib.rs` (`compute_block_rewards`, reward constants, `from_genesis_tx`), `ledger/src/cryptarchia/{mod.rs,stake.rs}`, `ledger/src/gas_and_fees.rs`, `zk/proofs/pol/src/lottery.rs`, `core/src/block/genesis.rs`, `services/chain/chain-service/src/states.rs`, `tools/blockchain-tools/src/{bin/genesis.rs,genesis/distribution.rs}`, `deployment/ceremony/genesis/{testnet,devnet,standalone}/`, `nodes/node/standalone-deployment-config.yaml`, `nodes/node/binary/src/config/deployment/settings.yaml`
Specs: `https://github.com/logos-co/logos-lips` @ `d788723992a805b395f377de6e6cf59859b47168` · read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `block-rewards.md` (all three in full); `analysis-block-reward-parameter-calibration.md` §Introduction, §The Parameter α_d, §The Parameter α_a, §The Inferred Total Stake, §The Burning Rate Average Factor, §Maximum Emission Rate, §Minimum Emission Rate; `cryptarchia-v1-protocol.md` §Constants, §Notation, §Slot, §Epoch (Epoch Schedule, Epoch State, Eligible Leader Notes, Epoch Nonce, Total Stake Inference, Epoch State Pseudocode), §Leadership Lottery (Proof of Leadership, Leader Rewards); `cryptarchia-proof-of-leadership.md` §Appendix › Lottery Approximation, Error Analysis, Corner Case (all by section)
Date: `2026-09-25` · author: `Claude Code (research agent)` · status: `final`

Second pass on #201 (parent #7). The first pass is PR #763, `inbox/201-block-reward-constants-genesis-stakes.md`, at `273658c7`; it could not build the node and derived item 1 by hand. This pass runs item 1 through the node's own genesis path, re-checks the fee-regime question (first report LB-002) and the overflow item with tests, and says which hand-derived numbers hold. It does not do the work of follow-up #764 (pinning the token unit, re-deriving the constants from `S_cap`, committing a per-template test); findings of the first report are cited as "PR #763 LB-00N" and are not re-filed.

---

## 1. Summary

- Overall assessment: every stake total and first-block value the first report derived by hand for the three ceremony templates is confirmed by execution at `c4c86be1`, which changed nothing in the audited paths since `273658c7`. Two of its conclusions need correcting. First, the network the repository actually starts in standalone mode is not built from the standalone template: the committed `nodes/node/standalone-deployment-config.yaml` predates the 10^9 stake increase of 2026-08-04, seeds `total_stake = 100,201` with `min_stake = 1`, and runs from block 1 in the bootstrap regime (`a_numerator = A_SCALE`, 57 + 38 units per block whatever the fees), the opposite of every ceremony template (LB-001). Second, `A_t` does not rise gradually once fees are high enough: at the shipped stakes the linear band of the emission factor is 11,415 units of window fees wide against a window that grows by 3.4·10^10 per block under sustained demand, so `a_numerator` jumps from 0 to `A_SCALE` in one block (block 76 of a run of full blocks on testnet) and the block reward falls from 3.0·10^10 to 95 units (LB-002). The last-fee versus window-average deviation (PR #763 LB-002) is unchanged, and nothing overflows at `total_stake = 10^15`.
- Findings: 0 critical · 0 high · 0 medium · 1 low · 1 informational
- Key themes: "the generated file is not what its inputs generate", "a controller band millions of times narrower than one block's change in its input", "hand-derived numbers confirmed, one ratio and one bound corrected".
- Must-fix before launch: none from this pass. Regenerating the standalone deployment file (LB-001) is a one-command fix; the unit question behind LB-002 stays with #764.

Answers to the four tasks of this pass, in order:

1. **What changed between `273658c7` and `c4c86be1` (§4.1).** Nothing in scope. The three commits (`d92b17fb4`, `8b34537a0`, `c4c86be18`) touch only `logos_sql/`, `zone-sdk/`, `tests/` and `Cargo.lock`; `git diff 273658c7 c4c86be1` over `ledger/`, `core/`, `zk/`, `tools/`, `deployment/`, `nodes/`, `services/pow` is empty. Every line number of the first report is valid at `c4c86be1`.
2. **Item 1, run for real (§4.2).** Through `GenesisBlockBuilder` and `LedgerState::from_genesis_tx` exactly as `CryptarchiaConsensusState::from_settings` calls it: testnet and devnet `(2,968,004,000,000,000, 0, 0, 0)`, standalone template `(100,201,000,000,000, 0, 0, 0)`, as the first report derived. A full block at genesis prices (5,290,612 units, 4,761,551 pooled after the 10 % PoW share) still gives `a_numerator = 0` and pays `(blend, leader) = (2,856,930, 1,904,620)`. At constant genesis prices `A_t` stays 0 for the 130 blocks run and forever after: leaving 0 needs 494 times a full block's pooled fee (the first report's "445" compares a net threshold with a gross fee; 445 is the gross ratio). With the execution market moving under sustained full blocks it leaves 0 at block 76 (testnet, devnet) or 47 (standalone template), directly to 1. The committed standalone file gives `(100,201, 120,000,000, 57, 38)`, the embedded default `settings.yaml` `(1, 120,000,000, 57, 38)`.
3. **Fee regime, last fee versus average (§4.3).** Unchanged at both commits: the prose (L199, L213, L246, L259) distributes `R̄_t`, the integer section (L447, L510-L511, L524-L525, L544, L553) and the node (`lib.rs:416-423`) distribute `D_{1,t}`. One more sentence supports the first report's reading: the integer section says of itself that its goal "is not to change the reward mechanism" and that the reward "remains driven by ... the moving average of pooled fees" (L434). Measured on a fee ramp, the two forms differ by a factor of 13 per block (block 75 of the testnet ramp: 29,976,022,343 paid against a window average of 2,250,149,902).
4. **Overflow at `10^15` (§4.4).** None, tested: `compute_lottery_values` at `10^15`, `2.968·10^15` and `u64::MAX` returns in-field values whose threshold matches `1 − (1 − f)^{v/D}` to within the approximation error; `compute_block_rewards` at `D = 10^15` and `D = u64::MAX` with a `u64::MAX` fee window stays below 2^87; the inference step at `2.968·10^15` stays below 2^57 even with every slot occupied. The first report's `u128` bound for a base-unit rescaling (`d ≤ 8`) is conservative: `a_numerator < A_SCALE` forces `Σ fees ≤ D/10,512`, so the reachable product is `1.38·10^26 · b` and `u128` holds up to `d = 12`; the binding limit is `S_cap` in `u64` (`d ≤ 9`).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `ledger/src/lib.rs:63-97, 387-449, 575-608` | reward constants, `compute_block_rewards`, `from_genesis_tx` |
| `ledger/src/cryptarchia/mod.rs:225-245, 459-479, 610-630, 716-796` | fee window, execution market update, `from_genesis_tx` / `from_utxos` |
| `ledger/src/cryptarchia/stake.rs:27-70` | `total_stake_inference` at large `D` |
| `ledger/src/gas_and_fees.rs:5-50`, `core/src/block/mod.rs:34`, `core/src/mantle/transactions/gas.rs:5-12` | block limits and genesis prices that bound a block's fee |
| `zk/proofs/pol/src/lottery.rs:131-139`, `core/src/proofs/leader_proof.rs:274-285` | lottery values and the threshold they feed |
| `services/chain/chain-service/src/states.rs:79-92`, `nodes/node/binary/src/config/cryptarchia/mod.rs:35-86`, `nodes/node/binary/src/cli/mod.rs:443-446`, `nodes/node/binary/src/config/deployment/mod.rs:16, 45-48` | how the node turns a deployment file into a genesis `LedgerState` and a ledger `Config` |
| `tools/blockchain-tools/src/bin/genesis.rs:226-267, 349-378`, `src/genesis/distribution.rs:37-122`, `scripts/standalone-genesis-ceremony.sh`, `.github/workflows/genesis-ceremony.yml` | the ceremony and what it writes where |
| `deployment/ceremony/genesis/{testnet,devnet,standalone}/*.yaml`, `nodes/node/standalone-deployment-config.yaml`, `nodes/node/binary/src/config/deployment/settings.yaml` | the inputs and the two committed outputs |

**Out of scope**

- The work of #764 (unit decision, constant re-derivation, committed test, ceremony-time checks), the missing reward pool and reserve (#89 LB-001, filed as #349), stake-inference correctness at small `D` (#44, filed as #716; #634), the faucet key handling (#204 LB-001, #688), the `trunc(f · 1000)` bias of the inference (#38, #33).
- The storage price's epoch-boundary update: the fee ramps in §4.2 hold the storage price at its genesis value, which is what the ledger does within one epoch (testnet epoch: 36,000 slots, about 1,200 blocks, longer than every ramp run).
- Third-party crates assumed correct: `num-bigint`, `astro-float`, `serde_yaml`, `ark-ff`.

**Assumptions**

- Release builds have no `overflow-checks` (#19); the tests ran in the release profile.
- The ledger `Config` used by the tests carries the deployment file's `epoch_config`, consensus parameters, `min_stake`, `faucet_pk` and PoW reward block, mapped as `into_cryptarchia_services_settings` maps them; the Blend reward parameters and Blend PoW block are test defaults, which neither the genesis path nor `compute_block_rewards` reads.

## 3. Method

- Re-read issue #201, parent #7, the first report (PR #763) and the issue comment, follow-up #764, and the prior reports the first one cites (`processed/44-…`, `processed/89-…`) plus `processed/204-config-silent-defaults.md`, `processed/38-slot-epoch-arithmetic.md` and `processed/92-sdp-declaration-cap.md` for what they already cover of the same files.
- `git log` / `git diff 273658c7..c4c86be1` restricted to the in-scope paths (§4.1); `git log -S` on the templates and the committed standalone file to date when their stakes diverged.
- Dynamic testing, in a scratch clone at `c4c86be1` (`/home/user/scratch/201`, the pristine checkout untouched): a test-only module `ledger/src/issue201.rs` plus an 18-line `#[cfg(test)]` trace of the intermediates at the end of `compute_block_rewards` (Appendix B). It reproduces `run_ceremony` (`distribute` + `build_genesis_block`, faucet key into `faucet_pk`) with `lb_core`'s `GenesisBlockBuilder` from each template's YAML, deserialises the committed deployment files' `genesis_block` as the node does, and calls `LedgerState::from_genesis_tx(genesis_tx, &config, epoch_nonce)` as `services/chain/chain-service/src/states.rs:83-91` does. The `logos-blockchain-tools` crate itself was not linked because it depends on `lb-node`, which the brief says not to build. Every first-block result is asserted equal to a `u128` port of the specification's reference Python. `cargo 1.98.1`, `rustc 1.98.1`, `cargo test --release -p logos-blockchain-ledger --lib issue201` (5 tests, all pass; output in Appendix C).
- Automated tooling: none beyond the tests.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | The committed standalone deployment was not regenerated when its ceremony inputs were scaled by 10^9, so the standalone node seeds `total_stake = 100,201` and `min_stake = 1` and runs the reserve-release regime that no ceremony template reaches | Configuration | Low | n/a | Open |
| LB-002 | At the shipped stakes the emission factor is a one-block step: once the fee window crosses `(D − STAKE_TARGET)/10,512`, `a_numerator` jumps from 0 to `A_SCALE` and the block reward falls from about 0.9 of the block's fee to 95 units | Economic / Incentive | Informational | n/a | Open |

### 4.1 What changed between `273658c7` and `c4c86be1`

| Commit | Paths | In scope? |
|---|---|---|
| `d92b17fb4` feat(logos-sql): support read-only replicas | `logos_sql/**`, `tests/**` | no |
| `8b34537a0` feat(logos-sql): compress inscription payloads | `logos_sql/**`, `Cargo.lock`, `tests/**` | no |
| `c4c86be18` feat(zone-sdk): channel update enum | `zone-sdk/**`, `tests/**` | no |

`git diff --stat 273658c7 c4c86be1 -- ledger core deployment tools zk services/pow nodes` is empty. So at `c4c86be1`: the constants are at `ledger/src/lib.rs:63-97` (`STAKE_TARGET` L86, `A_SCALE` L69, `FEE_AVG_NUM` L74, `INFLATION_NUMERATOR/DENOMINATOR` L79/L84), `compute_block_rewards` at L390-L449 with the saturating `a_numerator` at L411-L414 and the plain-`*` `reward_numerator` at L416-L423; `from_utxos` sums at `ledger/src/cryptarchia/mod.rs:743-749`; `min_stake.threshold` and `reward_pool_genesis` are at L90 and L56 of each `deployment-template.yaml`; the genesis prices at `core/src/mantle/transactions/gas.rs:9-12`. All as the first report cites them.

### 4.2 Item 1, measured through the node's genesis path

**The path.** A node reads `cryptarchia.genesis_block` from its deployment file (or, with no `--deployment`, from the embedded `settings.yaml`, `nodes/node/binary/src/cli/mod.rs:443-446`) and calls `LedgerState::from_genesis_tx(genesis_tx.clone(), &settings.config, epoch_nonce)` (`services/chain/chain-service/src/states.rs:83-91`), which runs `CryptarchiaLedger::from_genesis_tx` → `from_utxos` (`ledger/src/cryptarchia/mod.rs:716-796`) for the stake and `MantleLedger::from_genesis_tx` for the declarations. The ceremony (`tools/blockchain-tools/src/bin/genesis.rs:226-267`) builds that block from the four input files and writes the faucet key into `faucet_pk`. Per `deployment/README.md` L96-L99 its committed outputs are `nodes/node/binary/src/config/deployment/settings.yaml` (devnet/testnet, overwritten by `.github/workflows/genesis-ceremony.yml` at release) and `nodes/node/standalone-deployment-config.yaml` (standalone, `scripts/standalone-genesis-ceremony.sh`).

**Results.** `block 1` means `block_number = 1` (the header step increments it before `compute_block_rewards`, `lib.rs:347-351`), so the fee lands at window index 1. "Full block" is `EXECUTION_GAS_LIMIT = 3,193,460` execution gas (`gas_and_fees.rs:5`) plus `MAX_BLOCK_TRANSACTIONS_SIZE = 2,097,152` storage gas (`core/src/block/mod.rs:34`) at the state's prices.

| Genesis source | `total_stake` | `min_stake` | block 1, zero fees: `a`, blend, leader | block 1, full block (fee 5,290,612, pooled 4,761,551): `a`, blend, leader |
|---|---|---|---|---|
| ceremony `testnet` (138 notes incl. faucet) | 2,968,004,000,000,000 | 10^9 | 0, 0, 0 | 0, 2,856,930, 1,904,620 |
| ceremony `devnet` (same amounts, other keys) | 2,968,004,000,000,000 | 10^9 | 0, 0, 0 | 0, 2,856,930, 1,904,620 |
| ceremony `standalone` (5 notes) | 100,201,000,000,000 | 10^9 | 0, 0, 0 | 0, 2,856,930, 1,904,620 |
| committed `nodes/node/standalone-deployment-config.yaml` | **100,201** | **1** | **120,000,000, 57, 38** | **120,000,000, 57, 38** |
| committed `settings.yaml` (placeholder, zero notes) | 1 | 1 | 120,000,000, 57, 38 | 120,000,000, 57, 38 |

`next_epoch_state.total_stake` equals `epoch_state.total_stake` in every row. In the fee regime the reward is the block's own pooled fee split 60/40 with both shares floored separately, so `2,856,930 + 1,904,620 = 4,761,550`, one unit below the pooled 4,761,551. In the reserve regime the reward is `62,500/657 = 95.13` whatever the fee: `floor(57.08) + floor(38.05) = 95`.

**Forward runs (when, if ever, `A_t` leaves 0).**

| Run | testnet / devnet | standalone template |
|---|---|---|
| 130 full blocks at constant genesis prices: `Σ` pooled over the window, `a` at block 130 | 571,386,120, 0 | 571,386,120, 0 |
| constant pooled fee per block for `a > 0` with a full window: `(D − 3·10^9)/(10,512·120)` | > 2,352,867,358 (494.1 × a full block's pooled fee) | > 79,431,444 (16.7 ×) |
| sustained full blocks, base fee compounding (`update_execution_market(EXECUTION_GAS_LIMIT)` after each block): first block with `a > 0` | block 76, execution price 11,732, block fee 37,467,769,872, `a` = 120,000,000 at once | block 47, price 387, fee 1,237,966,172, `a` = 120,000,000 at once |

So at genesis prices `A_t` never leaves 0 on a ceremony network, however full the blocks; with the base fee rising 12.5 % per full block it leaves 0 after about 76 consecutive full blocks, and it leaves straight to 1 (LB-002). Appendix C has the per-block trajectory from block 70 to 80.

**Corrections to the first report's hand derivation.**

| First report | This pass |
|---|---|
| §4.1 totals `2,968,004,000,000,000` (twice) and `100,201,000,000,000`; `(total_stake, a, blend, leader) = (…, 0, 0, 0)` | confirmed by execution, for the ceremony templates |
| "every network the repository can build" runs `A_t = 0` (LB-001 title, §4.1) | not the standalone network the repository ships: its committed genesis seeds 100,201 (LB-001 below) |
| §4.1 "window fees must exceed about `2.82·10^11`", "445 times what a full block can pay" | threshold confirmed (`2.8234·10^11` pooled); the ratio is 444.7 against the gross fee and **494.1** against the pooled fee, which is what the threshold is measured in. Standalone: 15.0 gross, **16.7** pooled |
| §4.1 full block `≈ 5.3·10^6` | exactly 5,290,612 (3,193,460 + 2,097,152) |
| §4.3 "`d = 8` is the largest safe exponent" for a base-unit rescaling | conservative: `a < A_SCALE_b` requires `10,512 · Σ ≤ D ≤ u64::MAX`, so `fee ≤ 1.75·10^15` whenever the fee term is non-zero, and `657 · 1.2·10^8 · b · 1.75·10^15 = 1.38·10^26 · b` fits `u128` for `b ≤ 2.46·10^12` (`d ≤ 12`). Measured at `D = u64::MAX`, fee `1.754·10^15`: 87-bit numerator. `S_cap` in `u64` (`d ≤ 9`) is the binding limit |
| §4.3 an allocation with `D_0 ≈ 10^9` gives 95 units per block, 57 blend and 38 leader | confirmed by the committed standalone genesis (`D = 100,201`) |
| §4.5 largest intermediate `≤ 1.45·10^30` | true as a bound; the reachable maximum is `1.38·10^26` (above) |

### LB-001 · The committed standalone deployment was not regenerated when its ceremony inputs were scaled by 10^9, so the standalone node seeds `total_stake = 100,201` and `min_stake = 1` and runs the reserve-release regime that no ceremony template reaches

| | |
|---|---|
| Severity | Low |
| Difficulty | n/a (not an attack; a stale generated file) |
| Category | Configuration |
| Target | `nodes/node/standalone-deployment-config.yaml:L90` (`threshold: 1`), `:L113-L122` (genesis outputs 100,000 / 100 / 100 / 1 / `u64::MAX`); inputs `deployment/ceremony/genesis/standalone/stakeholders.yaml:L1-L8`, `deployment-template.yaml:L90`; generator `scripts/standalone-genesis-ceremony.sh:L1-L7` |
| Status | Open |

**Description**

`deployment/README.md` L96-L99 and L124-L126 say `nodes/node/standalone-deployment-config.yaml` is generated by `scripts/standalone-genesis-ceremony.sh` from `deployment/ceremony/genesis/standalone/*` and is "not hand-edited". It is not what those inputs generate. Commit `9544e94c8` (#3252, 2026-08-04, "Increase stake in all deployments") multiplied every standalone stake by 10^9 and the minimum stake from 1 to 10^9:

```
-  stake: 100000        +  stake: 100000000000000
-  stake: 100           +  stake: 100000000000
-  stake: 100           +  stake: 100000000000
-  stake: 1             +  stake: 1000000000
-      threshold: 1     +      threshold: 1000000000
```

The committed output was regenerated afterwards for other reasons (`d4be59653`, `a6a98bd8f`, `4e1039770`), but its genesis block still carries the pre-increase notes (`standalone-deployment-config.yaml` L113-L120: 100,000, 100, 100, 1) and `threshold: 1` (L90). CI checks only that the file deserialises (`.github/workflows/code-check.yml` `--check-config`), not that it is the ceremony's output.

Executed through `LedgerState::from_genesis_tx`, this file seeds `total_stake = 100,201`, below `STAKE_TARGET = 3·10^9`, so `a_numerator = A_SCALE` from block 1 and every block pays `blend = 57`, `leader = 38` plus tips, independent of the fees it collects (§4.2). Every ceremony template seeds `≥ 10^14` and pays the block's own fee with `a = 0`. The two regimes are what `block-rewards.md` §Lifecycle Phases calls Bootstrap and fee-driven, and the standalone node is the one the root `README.md` L67-L71 tells users to run, the one the FFI lifecycle tests start (`c-bindings/src/api/lifecycle.rs:227-231`), and the one CI validates. The embedded default `settings.yaml` (used when `--deployment` is omitted) is a separate case: a placeholder genesis with no notes, `total_stake = max(0, 1) = 1`, overwritten by the release ceremony; no block can be produced from it.

The history also dates the saturation the first report describes. Before `9544e94c8` the testnet stakeholders totalled 2,000,004 (4 × 10^5, 8 × 2·10^5, 4 × 1), inside the bootstrap regime; `9544e94c8` and `d6b1cf58e` (#3255, the 121 notes of `8·10^12`) moved testnet and devnet into saturation on the same day. The constants did not change; the deployment chore did.

**Exploit scenario**

None. Impact: anything learned about rewards on a standalone node (block explorer figures, wallet balances of leaders and Blend providers, FFI tests that exercise claims) reflects the reserve-release regime with a 1-unit minimum stake, while testnet and devnet run the fee regime with a `10^9` minimum stake. On the standalone chain the 95 units per block are minted with no reserve behind them (#349) against a seeded stake of 100,201, about 1,500 times the stake per year at `f = 1/20`, while every fee a block collects is removed from circulation apart from the PoW share. Regenerating the file will silently switch the standalone node into the fee regime.

**Recommendation**
- *Short term*: rerun `scripts/standalone-genesis-ceremony.sh` and commit the result, or revert the standalone inputs if the small allocation was intended (it is the only shipped genesis that exercises the reserve-release branch, which has value for testing; if so, say so in the template).
- *Long term*: add a CI step that reruns the ceremony on the committed inputs with the committed inscription and diffs `genesis_block` and `sdp_config` against the committed output, for the standalone file on every change and for `settings.yaml` on release branches; and the first-block reward test of #764 over the committed files as well as the templates.

**References**: `deployment/README.md` §genesis ceremony; `block-rewards.md` §Lifecycle Phases; PR #763 LB-001; #92 LB-005 (the `threshold: 1` of the committed files, rated there as a Sybil-cost issue); #764.

### LB-002 · At the shipped stakes the emission factor is a one-block step: once the fee window crosses `(D − STAKE_TARGET)/10,512`, `a_numerator` jumps from 0 to `A_SCALE` and the block reward falls from about 0.9 of the block's fee to 95 units

| | |
|---|---|
| Severity | Informational |
| Difficulty | n/a (not an attack) |
| Category | Economic / Incentive |
| Target | `ledger/src/lib.rs:L411-L423` (`a_numerator`, `reward_numerator`); `ledger/src/cryptarchia/mod.rs:L459-L479` (`update_execution_market`, up to +12.5 % per block) |
| Status | Open |

**Description**

`a_numerator = min(A_SCALE, sat(STAKE_TARGET + 10,512 · Σ − D))` is linear only on a band of width `A_SCALE = 1.2·10^8` in `10,512 · Σ`, i.e. 11,415 units of window fees, or 95 units per block averaged over 120 blocks. In the specification's unit that band is meant to be comparable to the fees themselves: with stake near target, `A_t = R̄_t / cap` and the reward moves smoothly from `R̄_t` to the 95-LGO cap. At the shipped stakes the window only reaches the band when it holds `2.8·10^11` units, and a fee ramp crosses it in far less than one block: on the testnet ramp the window grows from `2.70·10^11` to `3.04·10^11` between blocks 75 and 76, about 3·10^6 times the band. The test trajectory (Appendix C):

| testnet block | block fee | pooled | `10,512·Σ − (D − 3·10^9)` | `a` | blend + leader |
|---|---|---|---|---|---|
| 74 | 29,608,664,812 | 26,647,798,331 | −4.45·10^14 | 0 | 26,647,798,330 |
| 75 | 33,306,691,492 | 29,976,022,343 | −1.30·10^14 | 0 | 29,976,022,342 |
| 76 | 37,467,769,872 | 33,720,992,885 | +2.25·10^14 | 120,000,000 | 95 |
| 77 | 42,149,382,232 | 37,934,444,009 | +6.24·10^14 | 120,000,000 | 95 |

The reward is therefore not monotone in fees: it tracks the block fee up to `≈ 3·10^10` and then drops by eight orders of magnitude in one block. The fees of blocks 76 onward are removed from circulation (the node has no pool `P_t` to bank them, #349) and nothing replaces them but the 95-unit release. The drop itself is the specification's design at over-target stake (with `δ_t < 0`, `A_t = 0` until `γ_t` offsets `δ_t`, then the reward is capped at the release cap); the unit mismatch of PR #763 LB-001 makes the offset `2.35·10^9` units per block and the band 95 units, which turns a ramp into a cliff.

**Exploit scenario**

None profitable. A user who wanted to deny leaders and Blend providers their fee income for a window would have to fill about 76 consecutive blocks to lift the base fee to about 11,700 and pay the `≈ 3·10^11` of fees that crossing requires; those fees are what leaders would otherwise have received, and the effect ends one window (120 blocks) after the demand stops. The relevant consequence is for operators: under sustained congestion on a ceremony network, leader and Blend income collapses to 95 units per block at the moment fees are highest, which is the opposite of what an operator reading `block-rewards.md` §Lifecycle Phases expects.

**Recommendation**
- *Short term*: none in code; this is resolved by the unit decision of #764, after which `D − STAKE_TARGET` and the band are in the same unit as the fees. Record in #764 that the test must include a fee ramp, not only block 1.
- *Long term*: upstream, state in `block-rewards.md` §Emission Rate Factor Function that the reward is non-monotone in `R̄_t` when `δ_t < 0` (it follows `R̄_t` up to `R̄* = (α_d/α_a) · ((D − D_{0,target})/D_{0,target}) · S_cap · Δ_t` per block, 2.35·10^9 at the testnet `D` with the shipped constants, and is then capped at `I_max · S_cap · Δ_t / f`), so the calibration can decide whether that is intended.

**References**: `block-rewards.md` §Block Rewards (L182-L219), §Emission Rate Factor Function (L280-L313), §Lifecycle Phases (L105-L112), §Float Precision (L489-L505); `overview-cryptoeconomics.md` §Execution Fee Market; PR #763 LB-001, LB-002; #349.

### 4.3 Task 3: last fee versus window average, re-checked

At `d788723` the specification text is the one the first report quoted, line for line: equation (1) at L197-L201 and the sentence at L213 ("The recycled component distributes the average pooled reward `R̄_t`, rather than the single-block fee `R_block`, which smooths it across the window `T`"), the pool accounting at L246 and L259, all with `R̄_t`; the integer rewrite at L447 (`R_block = D_{1,t}`), L510-L511, L524-L525 and the reference code at L544 and L553 (`last_pooled_fee = pooled_fees_window[-1]`), all with the last fee. At `c4c86be1` the node's `reward_numerator` still multiplies by `get_fee_from_index(window_index)` (`lib.rs:416-423`), the entry just written for the block being applied (L404-L405). The node matches the integer section exactly: for block 1 of every genesis the `u128` port of the reference Python and the node agree on `a`, blend and leader (asserted in the test).

What this pass adds:

- The integer section contradicts its own statement of purpose. L432-L434: "The goal of this section is not to change the reward mechanism. … the reward logic remains driven by the same two KPI components … the inferred total stake relative to its target, and the moving average of pooled fees over the look-back window." The substitution `R̄_t → D_{1,t}` at L447 and L510 does change the mechanism. That supports the first report's view that the prose is normative and the integer section is the side to fix.
- The difference is large exactly when it matters. On the testnet ramp at block 75 the node pays `0.9 · fee = 29,976,022,343` where the window average is `270,017,988,261 / 120 = 2,250,149,902`, 13 times less. With constant fees the two forms coincide once the window is full.
- The reference Python cannot serve as a test oracle over that range: in `numpy.int64`, `INFLATION_DENOMINATOR · (A_SCALE − a) · last_pooled_fee` overflows for `last_pooled_fee > 116,988,483` (between blocks 20 and 30 of the testnet ramp, Appendix C) and `FEE_AVG_NUMERATOR · sum_fees` for `sum_fees > 8.77·10^14` (S-001).

PR #763 LB-002 stands as filed; the node needs the pool `P_t` (#349) before it can pay `R̄_t` without creating tokens.

### 4.4 Task 4: `total_stake` around `10^15`, tested

| Consumer | Test | Result |
|---|---|---|
| `compute_lottery_values` (`lottery.rs:131-139`) | `f = 1/30` and `1/20`; `D ∈ {1, 10^15, 1.00201·10^14, 2.968004·10^15, u64::MAX}` | both values below `p`; `t0` 185 to 250 bits, `p − t1` 115 to 245 bits; `v·(t0 + t1·v) mod p` over `p` equals `1 − (1 − f)^{v/D}` within `−7.7·10^-6` (`v = 2·10^14`, `D = 10^15`, `f = 1/30`) and `−1.9·10^-4` (`v = D`); `checked_sub` cannot fail since `t1_constant < p` |
| `compute_block_rewards` (`lib.rs:411-423`) | `D = 10^15` and `u64::MAX`, fee `u64::MAX` for 121 blocks; `D = u64::MAX`, fee `1.754·10^15` | `a` saturates to `A_SCALE` whenever the fee is large, so the fee term vanishes; largest `reward_numerator` 1.38·10^26 (87 bits). `657 · A_SCALE · u64::MAX = 1.45·10^30` fits `u128` in any case. With the 10 % PoW share a `u64::MAX` fee is refused by the PoW pool refill (`PowRewardPoolOverflow`, a `LedgerError`, not a panic) |
| `total_stake_inference` (`stake.rs:27-70`) | testnet schedule (period 21,600 slots, `f = 1/30`, `β = 1`), `D = 2.968·10^15` | empty window → 1 (#716); on-target (720) → 2.998·10^15; every slot occupied → 8.99·10^16, 57 bits |

No overflow at `10^15`. The only panic seen is outside that range: `D = 1.7·10^19` with 800 blocks against 712.8 expected overflows `u64` at `stake.rs:69` (`expect("After precision it should fit in a u64")`). #38 ruled this out on the premise that stake fits a `u64`; the shipped testnet genesis does not satisfy it, since its faucet note alone is `u64::MAX` (S-002). As in the first report, `total_stake` has no other arithmetic consumer: `leadership.rs:96` only logs it, and the recovery and epoch-state synthesis paths of #188 and #147 copy the field.

## 5. Suggestions (non-security)

### S-001 · Spec: the reference implementation of `block-rewards.md` overflows `int64` inside the range the node accepts

`block-rewards.md` L531-L561 computes in `numpy.int64`. `INFLATION_DENOMINATOR * (A_SCALE - a_numerator) * last_pooled_fee` overflows once `last_pooled_fee > 116,988,483` (with `a = 0`), and `FEE_AVG_NUMERATOR * sum_fees` once `sum_fees > 8.77·10^14`; numpy wraps silently. A full block at genesis prices is 5.3·10^6, but fewer than 30 consecutive full blocks of rising base fee exceed the first bound. Use Python integers (or state `u128`) in the reference, and give test vectors at a large fee.

### S-002 · `stake.rs:69` can panic on a chain whose genesis supply exceeds `u64::MAX`, which the shipped testnet genesis does

`total_stake_inference` computes in `i128` and converts back with `expect`. The new estimate is about `D · measured/expected`; once participating stake approaches `u64::MAX`, normal Poisson noise in `measured` (one standard deviation is about 4 % at 712 expected blocks) takes it past `u64::MAX` and every node panics at the same epoch transition. The testnet and devnet geneses total `u64::MAX + 2.968·10^15` because of the faucet note (`faucet.yaml`), so this is reachable in principle once the faucet has drained into active stake, about `1.8·10^7` drips of `10^12` (`deployment/scripts/run_faucet.sh:11`), hence not in practice. Saturate at `u64::MAX` instead of `expect`, and size the faucet note so that the genesis total fits `u64` with margin (#44 and #204 LB-001 already recommend a checked genesis sum).

### S-003 · Spec: `analysis-block-reward-parameter-calibration.md` gives `α_d = 1/6`, `block-rewards.md` and the node use `1/4`

Calibration §The Parameter α_d L80 chooses `α_d = 1/6` ("off target by 16.6 %"); `block-rewards.md` §Parametrization L168 and §Float Precision L443 use `1/4`, from which `A_SCALE = 3·10^9 / 25 = 1.2·10^8` follows. With `1/6` the constants would be `A_SCALE = 1.8·10^8`, `FEE_AVG_NUM = 15,768`. The same analysis still defines `D_{1,target} = S_tge` (§The Burning Rate Average Factor L133) and the stake target as "30 % of the TGE supply" (L110), vocabulary that `block-rewards.md` 1.1.0 removed. Worth settling in the same upstream pass as PR #763 S-001, since #764 re-derives the constants from these documents.

### S-004 · Spec: the `cryptarchia-proof-of-leadership.md` error table is labelled `f = 1/30` but holds the `f = 1/20` values

§Error Analysis L254-L281 says "For `f = 1/30`" and lists an order-2 error of −0.0444 % at 100 % stake and −0.0004 % at 10 %. Measured through the node's constants: `f = 1/20` gives −0.0444 % and −0.00044 %, `f = 1/30` gives −0.0193 % and −0.00019 %. Relabel the table or recompute it for `1/30`.

---

## Appendix A · Definitions

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

## Appendix B · Scratch test harness (not for upstream)

Applied to a clone at `c4c86be1`. Run with `CARGO_TARGET_DIR=… cargo test --release -p logos-blockchain-ledger --lib issue201 -- --nocapture --test-threads=1`.

Changes outside the new file (`serde_yaml` is already in the dependency graph through `lb-utils`):

```diff
--- a/ledger/Cargo.toml
+++ b/ledger/Cargo.toml
@@ [dev-dependencies]
 serde_json = { features = ["alloc"], workspace = true }
+serde_yaml = { workspace = true }
--- a/ledger/src/cryptarchia/mod.rs
+++ b/ledger/src/cryptarchia/mod.rs
-mod stake;
+pub(crate) mod stake;
--- a/ledger/src/lib.rs
+++ b/ledger/src/lib.rs
 mod intent;
+#[cfg(test)]
+mod issue201;
 pub mod mantle;
@@ fn compute_block_rewards (end, before `Ok(self)`)
+        #[cfg(test)]
+        issue201::LAST_REWARD.with(|c| {
+            c.set(issue201::RewardTrace {
+                sum_fees,
+                a_numerator,
+                reward_numerator,
+                blend_reward,
+                leader_reward: leader_reward.into_inner(),
+                pooled_fee: self
+                    .cryptarchia_ledger
+                    .get_fee_from_index(window_index)
+                    .into_inner(),
+            });
+        });
```

`ledger/src/issue201.rs`:

```rust
//! ISSUE-201 scratch tests (not for upstream). Builds the genesis
//! `LedgerState` of every shipped template through the node's own genesis
//! path (`GenesisBlock` -> `LedgerState::from_genesis_tx`) and prints the
//! first-block reward values.

use core::{cell::Cell, num::NonZero};
use std::path::PathBuf;

use lb_core::{
    block::genesis::{GenesisBlock, GenesisBlockBuilder},
    mantle::{
        Note, Value,
        gas::{Gas, GasCost},
        ledger::{Inputs, Outputs},
        ops::{sdp::SDPDeclareOp, transfer::TransferOp},
        traits::GenesisTx as _,
    },
    sdp::{Locators, MinStake, ServiceType},
};
use lb_key_management_system_keys::keys::{Ed25519PublicKey, ZkPublicKey};
use lb_utils::math::{NonNegativeF64, NonNegativeRatio};
use num_bigint::BigUint;

use crate::{Config, LedgerState, config::RewardPoWConfig, gas_and_fees::EXECUTION_GAS_LIMIT};

#[derive(Clone, Copy, Debug, Default)]
pub struct RewardTrace {
    pub sum_fees: u128,
    pub a_numerator: u128,
    pub reward_numerator: u128,
    pub blend_reward: Value,
    pub leader_reward: Value,
    pub pooled_fee: Value,
}

thread_local! {
    pub static LAST_REWARD: Cell<RewardTrace> = const { Cell::new(RewardTrace {
        sum_fees: 0, a_numerator: 0, reward_numerator: 0, blend_reward: 0, leader_reward: 0, pooled_fee: 0,
    }) };
}

type Id = [u8; 32];

// ---- mirrors of the ceremony / deployment YAML (same field names) ----------

#[derive(serde::Deserialize, Debug, Clone)]
struct StakeHolderInfo {
    zk_id: ZkPublicKey,
    stake: Value,
}

#[derive(serde::Deserialize, Debug, Clone)]
struct ProviderInfo {
    provider_id: Ed25519PublicKey,
    zk_id: ZkPublicKey,
    locators: Locators,
    service_type: ServiceType,
}

#[derive(serde::Deserialize, Debug, Clone)]
struct Faucet {
    zk_id: ZkPublicKey,
    funds: Value,
}

#[derive(serde::Deserialize)]
struct DeploymentYaml {
    cryptarchia: CryptarchiaYaml,
}

#[derive(serde::Deserialize)]
struct CryptarchiaYaml {
    epoch_config: lb_cryptarchia_engine::EpochConfig,
    security_param: NonZero<u32>,
    slot_activation_coeff: NonNegativeRatio,
    learning_rate: NonNegativeF64,
    uncle_reference_window_in_block: NonZero<u32>,
    sdp_config: SdpYaml,
    genesis_block: serde_yaml::Value,
    #[serde(default)]
    faucet_pk: Option<ZkPublicKey>,
    pow_config: PowYaml,
}

#[derive(serde::Deserialize)]
struct SdpYaml {
    min_stake: MinStake,
}

#[derive(serde::Deserialize)]
struct PowYaml {
    reward: RewardPoWConfig,
}

fn repo(path: &str) -> PathBuf {
    PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("..").join(path)
}

fn load<T: serde::de::DeserializeOwned>(path: &str) -> T {
    let s = std::fs::read_to_string(repo(path)).unwrap_or_else(|e| panic!("{path}: {e}"));
    serde_yaml::from_str(&s).unwrap_or_else(|e| panic!("{path}: {e}"))
}

/// Same mapping as `nodes/node/binary/src/config/cryptarchia/mod.rs`
/// `into_cryptarchia_services_settings` for every field the reward and
/// genesis paths read; blend reward params and blend PoW are the test
/// defaults (not read by these paths).
fn ledger_config(c: &CryptarchiaYaml) -> Config {
    let mut config = crate::cryptarchia::tests::config();
    config.epoch_config = c.epoch_config;
    config.consensus_config = lb_cryptarchia_engine::Config::new(
        c.security_param,
        c.slot_activation_coeff,
        c.learning_rate,
        c.uncle_reference_window_in_block,
    );
    config.sdp_config.min_stake = c.sdp_config.min_stake;
    config.faucet_pk = c.faucet_pk;
    config.pow_config.reward = c.pow_config.reward.clone();
    config
}

/// Any valid inscription: taken from the committed standalone genesis. It
/// carries the chain id / genesis time / nonce, none of which enters the
/// stake or reward arithmetic.
fn some_inscription() -> lb_core::mantle::ops::channel::inscribe::InscriptionOp {
    let d: DeploymentYaml = load("nodes/node/standalone-deployment-config.yaml");
    let gb: GenesisBlock = serde_yaml::from_value(d.cryptarchia.genesis_block).unwrap();
    gb.genesis_tx()
        .clone()
        .into_genesis_ops()
        .inscription
        .operation()
        .clone()
}

/// Replicates `run_ceremony`: `distribution::distribute` + `build_genesis_block`
/// (`tools/blockchain-tools/src/{genesis/distribution.rs,bin/genesis.rs}`),
/// then writes the faucet key into `faucet_pk` exactly as the ceremony does.
fn ceremony(env: &str) -> (GenesisBlock, Config, u64) {
    let dir = format!("deployment/ceremony/genesis/{env}");
    let stakeholders: Vec<StakeHolderInfo> = load(&format!("{dir}/stakeholders.yaml"));
    let providers: Vec<ProviderInfo> = load(&format!("{dir}/providers.yaml"));
    let faucet: Faucet = load(&format!("{dir}/faucet.yaml"));
    let mut deployment: DeploymentYaml = load(&format!("{dir}/deployment-template.yaml"));
    deployment.cryptarchia.faucet_pk = Some(faucet.zk_id);

    let mut notes: Vec<Note> = stakeholders
        .iter()
        .map(|s| Note::new(s.stake, s.zk_id))
        .collect();
    notes.push(Note::new(faucet.funds, faucet.zk_id));
    let transfer = TransferOp::new(Inputs::empty(), Outputs::try_new(notes.clone()).unwrap());
    let declarations: Vec<SDPDeclareOp> = providers
        .iter()
        .filter_map(|p| {
            transfer
                .utxos()
                .find(|u| u.note.pk == p.zk_id)
                .map(|utxo| SDPDeclareOp {
                    service_type: p.service_type,
                    locators: p.locators.clone(),
                    provider_id: p.provider_id.into(),
                    zk_id: p.zk_id,
                    service_note_id: utxo.id(),
                })
        })
        .collect();

    let mut it = notes.into_iter();
    let mut b = GenesisBlockBuilder::new().add_note(it.next().unwrap());
    for n in it {
        b = b.try_add_note(n).unwrap();
    }
    let mut decls = declarations.into_iter();
    let mut b = b
        .set_inscription(some_inscription())
        .add_declaration(decls.next().unwrap());
    for d in decls {
        b = b.add_declaration(d).unwrap();
    }
    let gb = b.build().unwrap();
    let config = ledger_config(&deployment.cryptarchia);
    (gb, config, faucet.funds)
}

/// A committed, generated deployment file: its genesis block and faucet key
/// exactly as the node reads them.
fn committed(path: &str) -> Option<(GenesisBlock, Config)> {
    let d: DeploymentYaml = load(path);
    let config = ledger_config(&d.cryptarchia);
    match serde_yaml::from_value::<GenesisBlock>(d.cryptarchia.genesis_block) {
        Ok(gb) => Some((gb, config)),
        Err(e) => {
            println!("  {path}: genesis_block does not deserialize: {e}");
            None
        }
    }
}

/// The node's own path: `CryptarchiaConsensusState::from_settings`
/// (`services/chain/chain-service/src/states.rs:83-91`).
fn node_genesis(gb: &GenesisBlock, config: &Config) -> Result<LedgerState, String> {
    let genesis_tx = gb.genesis_tx();
    let epoch_nonce = genesis_tx.cryptarchia_parameter().epoch_nonce;
    LedgerState::from_genesis_tx::<Id>(genesis_tx.clone(), config, epoch_nonce)
        .map(|(l, _)| l)
        .map_err(|e| format!("{e:?}"))
}

/// Pure integer reward function of `block-rewards.md` (reference Python,
/// L531-L561) in `u128`; `last_fee` is `pooled_fees_window[-1]`.
fn pure_reward(total_stake: u128, window: &[u128], last_fee: u128) -> (u128, u128, u128) {
    const A_SCALE: u128 = 120_000_000;
    let sum: u128 = window.iter().sum();
    let a = (3_000_000_000u128 + 10_512 * sum)
        .saturating_sub(total_stake)
        .min(A_SCALE);
    let num = 62_500 * a + 657 * (A_SCALE - a) * last_fee;
    let den = 657 * A_SCALE;
    (a, num * 6 / (den * 10), num * 4 / (den * 10))
}

/// Mandatory fees of a block that fills both limits at the state's prices:
/// `EXECUTION_GAS_LIMIT` (3,193,460) execution gas and
/// `MAX_BLOCK_TRANSACTIONS_SIZE` (2 MiB) storage gas.
fn full_block_fee(state: &LedgerState) -> GasCost {
    let p = state.get_gas_prices();
    let ex = GasCost::calculate(EXECUTION_GAS_LIMIT, p.execution_base_gas_price).unwrap();
    let st = GasCost::calculate(
        Gas::new(lb_core::block::MAX_BLOCK_TRANSACTIONS_SIZE as u64),
        p.storage_gas_price,
    )
    .unwrap();
    ex.checked_add(st).unwrap()
}

fn apply_block(state: LedgerState, block_number: u64, fee: GasCost, config: &Config) -> (LedgerState, RewardTrace) {
    let mut s = state;
    s.block_number = block_number;
    let s = s
        .compute_block_rewards::<Id>(fee, 0.into(), &config.pow_config.reward)
        .unwrap();
    (s, LAST_REWARD.with(Cell::get))
}

fn report(name: &str, state: &LedgerState, config: &Config) {
    let d = state.epoch_state().total_stake;
    println!("=== {name}");
    println!(
        "  total_stake (epoch_state) = {d}; next_epoch_state = {}; faucet_pk set = {}; min_stake = {}; pow_share = {}/{}",
        state.next_epoch_state().total_stake,
        config.faucet_pk.is_some(),
        config.sdp_config.min_stake.threshold,
        config.pow_config.reward.pow_share,
        config.pow_config.reward.share_den,
    );
    // Block 1, zero fees.
    let (_, t0) = apply_block(state.clone(), 1, 0.into(), config);
    let (a0, b0, l0) = pure_reward(u128::from(d), &[0], 0);
    assert_eq!((t0.a_numerator, u128::from(t0.blend_reward), u128::from(t0.leader_reward)), (a0, b0, l0));
    println!(
        "  block 1, zero fees:        a_numerator = {}, blend_reward = {}, leader_reward = {}",
        t0.a_numerator, t0.blend_reward, t0.leader_reward
    );
    // Block 1, full block at genesis prices.
    let fee = full_block_fee(state);
    let (_, t1) = apply_block(state.clone(), 1, fee, config);
    let (a1, b1, l1) = pure_reward(u128::from(d), &[u128::from(t1.pooled_fee)], u128::from(t1.pooled_fee));
    assert_eq!((t1.a_numerator, u128::from(t1.blend_reward), u128::from(t1.leader_reward)), (a1, b1, l1));
    println!(
        "  block 1, full block fee {} (pooled {} after PoW share): a_numerator = {}, blend_reward = {}, leader_reward = {}",
        fee.into_inner(), t1.pooled_fee, t1.a_numerator, t1.blend_reward, t1.leader_reward
    );

    // 130 consecutive full blocks at constant genesis prices.
    let mut s = state.clone();
    let mut first_positive = None;
    let mut last = RewardTrace::default();
    for n in 1..=130u64 {
        let (ns, t) = apply_block(s, n, fee, config);
        s = ns;
        if t.a_numerator > 0 && first_positive.is_none() {
            first_positive = Some(n);
        }
        last = t;
    }
    println!(
        "  130 full blocks at constant genesis prices: sum_fees = {}, a_numerator after block 130 = {}, first block with A>0 = {:?}, rewards at 130 = ({}, {})",
        last.sum_fees, last.a_numerator, first_positive, last.blend_reward, last.leader_reward
    );
    let d128 = u128::from(d);
    let pooled_needed = d128.saturating_sub(3_000_000_000).div_ceil(10_512 * 120);
    println!(
        "  constant pooled fee per block needed for A>0 once the window is full: > {} (x{:.1} a full genesis-price block)",
        pooled_needed,
        pooled_needed as f64 / t1.pooled_fee.max(1) as f64
    );

    // Sustained full blocks with the execution market moving (storage price
    // held at its epoch value; the chain would cross an epoch boundary only
    // after epoch_length slots).
    let mut s = state.clone();
    let mut found = None;
    for n in 1..=5_000u64 {
        let fee = full_block_fee(&s);
        let (ns, t) = apply_block(s, n, fee, config);
        s = ns.update_execution_market(EXECUTION_GAS_LIMIT);
        if t.a_numerator > 0 {
            found = Some((n, fee.into_inner(), t));
            break;
        }
    }
    match found {
        Some((n, fee, t)) => println!(
            "  sustained full blocks, execution market moving: A>0 first at block {n} (block fee {fee}, base fee then {}): a_numerator = {}, blend = {}, leader = {}",
            s.get_gas_prices().execution_base_gas_price.into_inner(), t.a_numerator, t.blend_reward, t.leader_reward
        ),
        None => println!("  sustained full blocks: A stays 0 for 5000 blocks"),
    }
}

#[test]
fn issue201_genesis_rewards_per_template() {
    for env in ["testnet", "devnet", "standalone"] {
        let (gb, config, faucet_funds) = ceremony(env);
        let state = node_genesis(&gb, &config).expect("genesis");
        let notes: Vec<Value> = gb
            .genesis_tx()
            .clone()
            .into_genesis_ops()
            .transfer
            .operation()
            .utxos()
            .map(|u| u.note.value)
            .collect();
        println!(
            "[ceremony {env}] {} genesis notes (incl. faucet {faucet_funds}), {} non-faucet",
            notes.len(),
            notes.len() - 1
        );
        report(&format!("ceremony templates: {env}"), &state, &config);
    }
    for path in [
        "nodes/node/standalone-deployment-config.yaml",
        "nodes/node/binary/src/config/deployment/settings.yaml",
    ] {
        println!("[committed {path}]");
        if let Some((gb, config)) = committed(path) {
            match node_genesis(&gb, &config) {
                Ok(state) => report(&format!("committed: {path}"), &state, &config),
                Err(e) => println!("  from_genesis_tx failed: {e}"),
            }
        }
    }
}

/// The spec's reference Python uses `numpy.int64`. Where does it overflow?
#[test]
fn issue201_reference_int64_bounds() {
    let a_scale: i64 = 120_000_000;
    let max_last_fee = i64::MAX / (657 * a_scale);
    let max_sum = i64::MAX / 10_512;
    println!(
        "int64 reference: 657*A_SCALE*last_fee overflows for last_fee > {max_last_fee}; 10512*sum_fees overflows for sum_fees > {max_sum}"
    );
    // Node side, u128. PoW share 0 here (a u64::MAX fee with a 10 % share
    // is rejected by the PoW pool refill, checked below).
    let config = crate::cryptarchia::tests::config();
    for (d, fee) in [
        (1_000_000_000_000_000u64, u64::MAX),
        (u64::MAX, 1_754_000_000_000_000u64),
        (u64::MAX, u64::MAX),
    ] {
        let mut s = LedgerState::from_utxos([crate::cryptarchia::tests::utxo()], &config);
        s.cryptarchia_ledger.epoch_state.total_stake = d;
        for n in 1..=121u64 {
            let (ns, t) = apply_block(s, n, GasCost::from(fee), &config);
            s = ns;
            if n == 1 || n == 121 {
                println!(
                    "node u128, D={d}, fee={fee}, block {n}: sum_fees = {}, a = {}, reward_numerator = {} ({} bits), blend = {}, leader = {}",
                    t.sum_fees, t.a_numerator, t.reward_numerator, 128 - t.reward_numerator.leading_zeros(), t.blend_reward, t.leader_reward
                );
            }
        }
    }
    println!(
        "657*A_SCALE*u64::MAX = {:?} (checked_mul in u128)",
        657u128.checked_mul(120_000_000).and_then(|x| x.checked_mul(u128::from(u64::MAX)))
    );
    // With the shipped 10 % PoW share: fee u64::MAX.
    let mut config10 = crate::cryptarchia::tests::config();
    config10.pow_config.reward.pow_share = 10;
    config10.pow_config.reward.share_den = NonZero::new(100).unwrap();
    let mut s = LedgerState::from_utxos([crate::cryptarchia::tests::utxo()], &config10);
    s.block_number = 1;
    let r = s.compute_block_rewards::<Id>(GasCost::from(u64::MAX), 0.into(), &config10.pow_config.reward);
    println!("pow_share 10/100, fee u64::MAX: {:?}", r.map(|_| ()).map_err(|e| format!("{e:?}")));
}

#[test]
fn issue201_transition_trajectory() {
    let (gb, config, _) = ceremony("testnet");
    let mut s = node_genesis(&gb, &config).unwrap();
    for n in 1..=80u64 {
        let fee = full_block_fee(&s);
        let (ns, t) = apply_block(s, n, fee, &config);
        s = ns.update_execution_market(EXECUTION_GAS_LIMIT);
        if n % 10 == 0 || (70..=80).contains(&n) {
            println!(
                "testnet block {n}: fee {} pooled {} sum_fees {} 10512*sum-(D-3e9) = {} a = {} blend = {} leader = {}",
                fee.into_inner(), t.pooled_fee, t.sum_fees,
                10_512i128 * t.sum_fees as i128 - (2_968_004_000_000_000i128 - 3_000_000_000),
                t.a_numerator, t.blend_reward, t.leader_reward
            );
        }
    }
}

#[test]
fn issue201_lottery_at_1e15() {
    use lb_pol::P;
    for (num, den) in [(1u32, 30u32), (1, 20)] {
        let lc = lb_pol::LotteryConstants::new(NonNegativeRatio::new(num, den.try_into().unwrap()));
        let f = f64::from(num) / f64::from(den);
        for d in [
            1u64,
            1_000_000_000_000_000,
            2_968_004_000_000_000,
            100_201_000_000_000,
            u64::MAX,
        ] {
            let (l0, l1) = lc.compute_lottery_values(d);
            let l0: BigUint = l0.into();
            let l1: BigUint = l1.into();
            assert!(l0 < *P && l1 < *P);
            // threshold(v) = v*(l0 + l1*v) mod p, and the exact
            // p*(1-(1-f)^(v/D)) for v = largest testnet note, v = D/10, v = D.
            let mut line = format!("f={num}/{den} D={d}: t0={} bits, p-t1={} bits;", l0.bits(), (&*P - &l1).bits());
            for v in [200_000_000_000_000u64, d / 10, d] {
                if v == 0 || v > d {
                    continue;
                }
                let vb = BigUint::from(v);
                let thr = (&vb * ((&l0 + &l1 * &vb) % &*P)) % &*P;
                let approx = thr.to_string().parse::<f64>().unwrap()
                    / P.to_string().parse::<f64>().unwrap();
                let exact = 1.0 - (1.0 - f).powf(v as f64 / d as f64);
                line += &format!(" v={v}: approx={approx:.6e} exact={exact:.6e} rel_err={:.3e};", (approx - exact) / exact);
            }
            println!("{line}");
        }
    }
}

#[test]
fn issue201_inference_at_1e15() {
    // total_stake_inference at the testnet genesis D with the testnet
    // schedule (k=120, f=1/30, lr=1): empty window, expected, and every slot
    // occupied.
    let si = crate::cryptarchia::stake::StakeInference::new(1.0, 1.0 / 30.0, 21_600);
    for d in [2_968_004_000_000_000u64, 1_000_000_000_000_000] {
        for measured in [0u64, 720, 21_600] {
            let new = si.total_stake_inference::<1000>(d, measured);
            println!("inference D={d} measured={measured} -> {new}");
        }
    }
    for (d, measured) in [(17_000_000_000_000_000_000u64, 712u64), (17_000_000_000_000_000_000, 800)] {
        let r = std::panic::catch_unwind(|| si.total_stake_inference::<1000>(d, measured));
        println!("inference D={d} measured={measured} -> {:?}", r.map_err(|e| e.downcast_ref::<&str>().map(|s| (*s).to_owned()).or_else(|| e.downcast_ref::<String>().cloned())));
    }
    let largest_safe = u64::MAX / 30;
    println!("u64::MAX / 30 = {largest_safe} (a full window at lr=1, f=1/30 multiplies D by ~30)");
}
```

## Appendix C · Test output (`c4c86be1`, release profile)

```text

running 5 tests
test issue201::issue201_genesis_rewards_per_template ... [ceremony testnet] 138 genesis notes (incl. faucet 18446744073709551615), 137 non-faucet
=== ceremony templates: testnet
  total_stake (epoch_state) = 2968004000000000; next_epoch_state = 2968004000000000; faucet_pk set = true; min_stake = 1000000000; pow_share = 10/100
  block 1, zero fees:        a_numerator = 0, blend_reward = 0, leader_reward = 0
  block 1, full block fee 5290612 (pooled 4761551 after PoW share): a_numerator = 0, blend_reward = 2856930, leader_reward = 1904620
  130 full blocks at constant genesis prices: sum_fees = 571386120, a_numerator after block 130 = 0, first block with A>0 = None, rewards at 130 = (2856930, 1904620)
  constant pooled fee per block needed for A>0 once the window is full: > 2352867358 (x494.1 a full genesis-price block)
  sustained full blocks, execution market moving: A>0 first at block 76 (block fee 37467769872, base fee then 13198): a_numerator = 120000000, blend = 57, leader = 38
[ceremony devnet] 138 genesis notes (incl. faucet 18446744073709551615), 137 non-faucet
=== ceremony templates: devnet
  total_stake (epoch_state) = 2968004000000000; next_epoch_state = 2968004000000000; faucet_pk set = true; min_stake = 1000000000; pow_share = 10/100
  block 1, zero fees:        a_numerator = 0, blend_reward = 0, leader_reward = 0
  block 1, full block fee 5290612 (pooled 4761551 after PoW share): a_numerator = 0, blend_reward = 2856930, leader_reward = 1904620
  130 full blocks at constant genesis prices: sum_fees = 571386120, a_numerator after block 130 = 0, first block with A>0 = None, rewards at 130 = (2856930, 1904620)
  constant pooled fee per block needed for A>0 once the window is full: > 2352867358 (x494.1 a full genesis-price block)
  sustained full blocks, execution market moving: A>0 first at block 76 (block fee 37467769872, base fee then 13198): a_numerator = 120000000, blend = 57, leader = 38
[ceremony standalone] 5 genesis notes (incl. faucet 18446744073709551615), 4 non-faucet
=== ceremony templates: standalone
  total_stake (epoch_state) = 100201000000000; next_epoch_state = 100201000000000; faucet_pk set = true; min_stake = 1000000000; pow_share = 10/100
  block 1, zero fees:        a_numerator = 0, blend_reward = 0, leader_reward = 0
  block 1, full block fee 5290612 (pooled 4761551 after PoW share): a_numerator = 0, blend_reward = 2856930, leader_reward = 1904620
  130 full blocks at constant genesis prices: sum_fees = 571386120, a_numerator after block 130 = 0, first block with A>0 = None, rewards at 130 = (2856930, 1904620)
  constant pooled fee per block needed for A>0 once the window is full: > 79431444 (x16.7 a full genesis-price block)
  sustained full blocks, execution market moving: A>0 first at block 47 (block fee 1237966172, base fee then 435): a_numerator = 120000000, blend = 57, leader = 38
[committed nodes/node/standalone-deployment-config.yaml]
=== committed: nodes/node/standalone-deployment-config.yaml
  total_stake (epoch_state) = 100201; next_epoch_state = 100201; faucet_pk set = true; min_stake = 1; pow_share = 10/100
  block 1, zero fees:        a_numerator = 120000000, blend_reward = 57, leader_reward = 38
  block 1, full block fee 5290612 (pooled 4761551 after PoW share): a_numerator = 120000000, blend_reward = 57, leader_reward = 38
  130 full blocks at constant genesis prices: sum_fees = 571386120, a_numerator after block 130 = 120000000, first block with A>0 = Some(1), rewards at 130 = (57, 38)
  constant pooled fee per block needed for A>0 once the window is full: > 0 (x0.0 a full genesis-price block)
  sustained full blocks, execution market moving: A>0 first at block 1 (block fee 5290612, base fee then 1): a_numerator = 120000000, blend = 57, leader = 38
[committed nodes/node/binary/src/config/deployment/settings.yaml]
=== committed: nodes/node/binary/src/config/deployment/settings.yaml
  total_stake (epoch_state) = 1; next_epoch_state = 1; faucet_pk set = true; min_stake = 1; pow_share = 10/100
  block 1, zero fees:        a_numerator = 120000000, blend_reward = 57, leader_reward = 38
  block 1, full block fee 5290612 (pooled 4761551 after PoW share): a_numerator = 120000000, blend_reward = 57, leader_reward = 38
  130 full blocks at constant genesis prices: sum_fees = 571386120, a_numerator after block 130 = 120000000, first block with A>0 = Some(1), rewards at 130 = (57, 38)
  constant pooled fee per block needed for A>0 once the window is full: > 0 (x0.0 a full genesis-price block)
  sustained full blocks, execution market moving: A>0 first at block 1 (block fee 5290612, base fee then 1): a_numerator = 120000000, blend = 57, leader = 38
ok
test issue201::issue201_inference_at_1e15 ... inference D=2968004000000000 measured=0 -> 1
inference D=2968004000000000 measured=720 -> 2997983838383838
inference D=2968004000000000 measured=21600 -> 89939515151515151
inference D=1000000000000000 measured=0 -> 1
inference D=1000000000000000 measured=720 -> 1010101010101010
inference D=1000000000000000 measured=21600 -> 30303030303030303
inference D=17000000000000000000 measured=712 -> Ok(16980920314253647586)

thread 'issue201::issue201_inference_at_1e15' (4116) panicked at ledger/src/cryptarchia/stake.rs:69:14:
After precision it should fit in a u64: TryFromIntError(PosOverflow)
inference D=17000000000000000000 measured=800 -> Err(Some("After precision it should fit in a u64: TryFromIntError(PosOverflow)"))
u64::MAX / 30 = 614891469123651720 (a full window at lr=1, f=1/30 multiplies D by ~30)
ok
test issue201::issue201_lottery_at_1e15 ... f=1/30 D=1: t0=249 bits, p-t1=243 bits; v=1: approx=3.332689e-2 exact=3.333333e-2 rel_err=-1.932e-4;
f=1/30 D=1000000000000000: t0=199 bits, p-t1=144 bits; v=200000000000000: approx=6.757324e-3 exact=6.757376e-3 rel_err=-7.675e-6; v=100000000000000: approx=3.384409e-3 exact=3.384415e-3 rel_err=-1.917e-6; v=1000000000000000: approx=3.332689e-2 exact=3.333333e-2 rel_err=-1.932e-4;
f=1/30 D=2968004000000000: t0=198 bits, p-t1=141 bits; v=200000000000000: approx=2.281859e-3 exact=2.281861e-3 rel_err=-8.703e-7; v=296800400000000: approx=3.384409e-3 exact=3.384415e-3 rel_err=-1.917e-6; v=2968004000000000: approx=3.332689e-2 exact=3.333333e-2 rel_err=-1.932e-4;
f=1/30 D=100201000000000: t0=203 bits, p-t1=150 bits; v=10020100000000: approx=3.384409e-3 exact=3.384415e-3 rel_err=-1.917e-6; v=100201000000000: approx=3.332689e-2 exact=3.333333e-2 rel_err=-1.932e-4;
f=1/30 D=18446744073709551615: t0=185 bits, p-t1=115 bits; v=200000000000000: approx=3.675613e-7 exact=3.675613e-7 rel_err=1.165e-10; v=1844674407370955161: approx=3.384409e-3 exact=3.384415e-3 rel_err=-1.917e-6; v=18446744073709551615: approx=3.332689e-2 exact=3.333333e-2 rel_err=-1.932e-4;
f=1/20 D=1: t0=250 bits, p-t1=245 bits; v=1: approx=4.997779e-2 exact=5.000000e-2 rel_err=-4.441e-4;
f=1/20 D=1000000000000000: t0=200 bits, p-t1=145 bits; v=200000000000000: approx=1.020604e-2 exact=1.020622e-2 rel_err=-1.759e-5; v=100000000000000: approx=5.116174e-3 exact=5.116197e-3 rel_err=-4.391e-6; v=1000000000000000: approx=4.997779e-2 exact=5.000000e-2 rel_err=-4.441e-4;
f=1/20 D=2968004000000000: t0=198 bits, p-t1=142 bits; v=200000000000000: approx=3.450443e-3 exact=3.450450e-3 rel_err=-1.993e-6; v=296800400000000: approx=5.116174e-3 exact=5.116197e-3 rel_err=-4.391e-6; v=2968004000000000: approx=4.997779e-2 exact=5.000000e-2 rel_err=-4.441e-4;
f=1/20 D=100201000000000: t0=203 bits, p-t1=152 bits; v=10020100000000: approx=5.116174e-3 exact=5.116197e-3 rel_err=-4.391e-6; v=100201000000000: approx=4.997779e-2 exact=5.000000e-2 rel_err=-4.441e-4;
f=1/20 D=18446744073709551615: t0=186 bits, p-t1=117 bits; v=200000000000000: approx=5.561229e-7 exact=5.561229e-7 rel_err=3.860e-11; v=1844674407370955161: approx=5.116174e-3 exact=5.116197e-3 rel_err=-4.391e-6; v=18446744073709551615: approx=4.997779e-2 exact=5.000000e-2 rel_err=-4.441e-4;
ok
test issue201::issue201_reference_int64_bounds ... int64 reference: 657*A_SCALE*last_fee overflows for last_fee > 116988483; 10512*sum_fees overflows for sum_fees > 877413626032608
node u128, D=1000000000000000, fee=18446744073709551615, block 1: sum_fees = 18446744073709551615, a = 120000000, reward_numerator = 7500000000000 (43 bits), blend = 57, leader = 38
node u128, D=1000000000000000, fee=18446744073709551615, block 121: sum_fees = 2213609288845146193800, a = 120000000, reward_numerator = 7500000000000 (43 bits), blend = 57, leader = 38
node u128, D=18446744073709551615, fee=1754000000000000, block 1: sum_fees = 1754000000000000, a = 0, reward_numerator = 138285360000000000000000000 (87 bits), blend = 1052400000000000, leader = 701600000000000
node u128, D=18446744073709551615, fee=1754000000000000, block 121: sum_fees = 210480000000000000, a = 120000000, reward_numerator = 7500000000000 (43 bits), blend = 57, leader = 38
node u128, D=18446744073709551615, fee=18446744073709551615, block 1: sum_fees = 18446744073709551615, a = 120000000, reward_numerator = 7500000000000 (43 bits), blend = 57, leader = 38
node u128, D=18446744073709551615, fee=18446744073709551615, block 121: sum_fees = 2213609288845146193800, a = 120000000, reward_numerator = 7500000000000 (43 bits), blend = 57, leader = 38
657*A_SCALE*u64::MAX = Some(1454341302771261049326600000000) (checked_mul in u128)
pow_share 10/100, fee u64::MAX: Ok(())
ok
test issue201::issue201_transition_trajectory ... testnet block 10: fee 14870992 pooled 13383893 sum_fees 64860194 10512*sum-(D-3e9) = -2967319189640672 a = 0 blend = 8030335 leader = 5353557
testnet block 20: fee 53192512 pooled 47873261 sum_fees 365397736 10512*sum-(D-3e9) = -2964159938999168 a = 0 blend = 28723956 leader = 19149304
testnet block 30: fee 171350532 pooled 154215479 sum_fees 1326981498 10512*sum-(D-3e9) = -2954051770493024 a = 0 blend = 92529287 leader = 61686191
testnet block 40: fee 544985352 pooled 490486817 sum_fees 4392416708 10512*sum-(D-3e9) = -2921827915565504 a = 0 blend = 294292090 leader = 196194726
testnet block 50: fee 1758500152 pooled 1582650137 sum_fees 14272376212 10512*sum-(D-3e9) = -2817969781259456 a = 0 blend = 949590082 leader = 633060054
testnet block 60: fee 5699229792 pooled 5129306813 sum_fees 46248524148 10512*sum-(D-3e9) = -2481836514156224 a = 0 blend = 3077584087 leader = 2051722725
testnet block 70: fee 18489037092 pooled 16640133383 sum_fees 149930942270 10512*sum-(D-3e9) = -1391926934857760 a = 0 blend = 9984080029 leader = 6656053353
testnet block 71: fee 20797908672 pooled 18718117805 sum_fees 168649060075 10512*sum-(D-3e9) = -1195162080491600 a = 0 blend = 11230870683 leader = 7487247122
testnet block 72: fee 23397385112 pooled 21057646601 sum_fees 189706706676 10512*sum-(D-3e9) = -973804099421888 a = 0 blend = 12634587960 leader = 8423058640
testnet block 73: fee 26319401012 pooled 23687460911 sum_fees 213394167587 10512*sum-(D-3e9) = -724801510325456 a = 0 blend = 14212476546 leader = 9474984364
testnet block 74: fee 29608664812 pooled 26647798331 sum_fees 240041965918 10512*sum-(D-3e9) = -444679854269984 a = 0 blend = 15988678998 leader = 10659119332
testnet block 75: fee 33306691492 pooled 29976022343 sum_fees 270017988261 10512*sum-(D-3e9) = -129571907400368 a = 0 blend = 17985613405 leader = 11990408937
testnet block 76: fee 37467769872 pooled 33720992885 sum_fees 303738981146 10512*sum-(D-3e9) = 224903169806752 a = 120000000 blend = 57 leader = 38
testnet block 77: fee 42149382232 pooled 37934444009 sum_fees 341673425155 10512*sum-(D-3e9) = 623670045229360 a = 120000000 blend = 57 leader = 38
testnet block 78: fee 47415397772 pooled 42673857995 sum_fees 384347283150 10512*sum-(D-3e9) = 1072257640472800 a = 120000000 blend = 57 leader = 38
testnet block 79: fee 53339266072 pooled 48005339465 sum_fees 432352622615 10512*sum-(D-3e9) = 1576889768928880 a = 120000000 blend = 57 leader = 38
testnet block 80: fee 60004017092 pooled 54003615383 sum_fees 486356237998 10512*sum-(D-3e9) = 2144575773834976 a = 120000000 blend = 57 leader = 38
ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 172 filtered out; finished in 0.22s

```
