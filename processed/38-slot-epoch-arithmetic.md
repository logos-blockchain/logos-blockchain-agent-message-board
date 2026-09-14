# Audit Report — Slot and epoch arithmetic, stake snapshot distance, security parameters

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/38`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `consensus/cryptarchia-engine` (`time.rs`, `config.rs`, `lib.rs`), `ledger/src/config.rs`, `ledger/src/cryptarchia` (`mod.rs`, `stake.rs`, `block_density.rs`), `services/time`, `services/chain/chain-service` (`lib.rs`, `uncle.rs`), `nodes/node/binary/src/config`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `cryptarchia-v1-protocol.md`, `fork-choice.md` (in full); `bedrock-genesis-block.md` §Cryptarchia Parameters, §Initializing Bedrock; `cryptarchia-proof-of-leadership.md` §Ledger Root, §Circuit Public Inputs; `cryptarchia-total-stake-inference.md` §Construction; `cryptarchia-v1-bootstr-sync.md` §Constants, §Setting the Fork Choice Rule (by section)
Date: `2026-09-11` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: the slot-to-epoch conversion, the epoch schedule, the stake-distribution and nonce snapshot distances, and the total-stake inference match `cryptarchia-v1-protocol.md`; no consensus-affecting arithmetic defect was found. The weaknesses are at the configuration and clock boundary: the consensus schedule parameters are per-deployment values that nothing binds to the chain or validates, and the node's slot counter is a tick counter that can lag the clock.
- Findings: `0` critical · `0` high · `0` medium · `2` low · `2` informational
- Key themes: "consensus parameters unbound and unvalidated", "slot clock derived from ticks, not time", "floating-point derivation of integer schedule constants"
- Must-fix before launch: none

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `consensus/cryptarchia-engine/src/time.rs` | `Slot`, `Epoch`, `EpochConfig::{epoch, starting_slot, last_slot, epoch_length}`, `Slot::from_offset_and_config`, `SlotTimer::slot_interval` |
| `consensus/cryptarchia-engine/src/config.rs` | `Config` deserialisation, `base_period_length`, `s_gen`, `uncle_reference_window_in_slot`, `expected_blocks_per_epoch` |
| `consensus/cryptarchia-engine/src/lib.rs` | slot checks in `apply_header`, `k` and `s_gen` use in `maxvalid_bg` / `maxvalid_mc` / `lib` |
| `ledger/src/config.rs` | `nonce_snapshot`, `stake_distribution_snapshot`, `total_stake_snapshot`, `total_stake_inference_period`, `epoch` |
| `ledger/src/cryptarchia/mod.rs` | `EpochState::update_from_ledger`, `LedgerState::update_epoch_state` (same / next / skipped epoch), `try_apply_proof` public inputs, `from_utxos` genesis state |
| `ledger/src/cryptarchia/stake.rs`, `block_density.rs` | total-stake inference arithmetic, density window |
| `services/time/src` | slot tick stream (`backends/common.rs`, `backends/ntp/mod.rs`, `backends/system_time.rs`) |
| `services/chain/chain-service/src/lib.rs`, `uncle.rs` | ordering of the future-slot check, uncle slot checks, ledger update and engine insertion |
| `nodes/node/binary/src/config/{cryptarchia,deployment,time}`, `nodes/node/binary/src/cli/mod.rs`, `deployment/ceremony/genesis/*/deployment-template.yaml`, `tools/config/src/deployment.rs` | where `k`, `f`, `beta`, `W` and the phase lengths come from, per network |
| `zk/proofs/pol/src/lottery.rs` | lottery constants derived from `f` (determinism only) |

**Out of scope**

Fork choice and block-tree pruning (parent #1 asks for them separately), proof-of-leadership circuit and verification, uncle validity rules beyond their slot checks, PoW difficulty and reward windows, storage and execution markets, mempool, networking. Third-party crates assumed correct: `tokio` (interval semantics as documented), `time`, `astro_float`, `num_bigint`, `rpds`, `sntpc`.

**Assumptions**

The specifications at the commit above are the reference. Repo-level facts from issue #19 were re-verified at this commit and hold: `[profile.release]` sets no `overflow-checks` (`Cargo.toml:11-14`), so `+ - *` wrap silently in release; `arithmetic_side_effects`, `as_conversions`, `cast_possible_truncation`, `cast_possible_wrap`, `indexing_slicing`, `unwrap_used`, `expect_used` are allowed (`Cargo.toml:339-414`). Operators run the shipped binary, whose deployment settings are embedded (`nodes/node/binary/src/config/deployment/mod.rs:16`), unless they pass `--deployment` (`nodes/node/binary/src/cli/mod.rs:443-446`).

## 3. Method

- Manual review of the in-scope paths, working through issue `#38` (sub-issue of `#1`) item by item; issue `#19` read first.
- Spec conformance against `cryptarchia-v1-protocol.md` §Constants, §Notation, §Epoch Schedule, §Epoch Nonce, §Epoch State Pseudocode, §Block Header Validation (steps 5, 6), `cryptarchia-total-stake-inference.md` §Algorithm, `bedrock-genesis-block.md` §Cryptarchia Parameters and §Initial Epoch State, `fork-choice.md` §Definitions (`s_gen`), for slot/epoch arithmetic, snapshot slots and parameter provenance.
- Automated tooling: none against the code. `python3` (3.x, IEEE-754 doubles) was used to evaluate the floating-point derivations in `config.rs` for the deployed and for arbitrary `(k, f)` pairs (LB-003).
- Dynamic testing: none. Existing unit tests were read for coverage: `ledger/src/config.rs` (`epoch_snapshots`, `slot_to_epoch`, the two `*_panics_at_epoch_zero` tests), `ledger/src/cryptarchia/mod.rs` (`test_epoch_transition`, `test_epoch_state_for_slot_with_empty_epochs`, `test_try_apply_header_with_proof_from_jumped_epoch`, `test_total_stake_inference_chain_across_epoch_transitions`), `ledger/src/cryptarchia/block_density.rs`, `consensus/cryptarchia-engine/src/config.rs`.

### Checklist verification

**Slot to epoch conversion, epoch length, boundaries, wrap, genesis.**

- Epoch length: `epoch_length = (3 + 3 + 4) * base_period_length` with `base_period_length = floor(k / f)` (`time.rs:278-288`, `config.rs:117-131`), which is the spec's `10 * floor(k/f)`. The phase split is configuration (`EpochConfig`, three `NonZero<u8>`), spec-conformant at the shipped `3/3/4`.
- Conversion: `epoch(slot) = slot / epoch_length` in `u64`, then `try_into::<u32>().expect(..)` (`time.rs:260-264`). The first slot of epoch `e` is `e * epoch_length` (`time.rs:267-269`), confirmed by `slot_to_epoch` (`ledger/src/config.rs:581-641`: slot 100 is epoch 1 at length 100). No off-by-one.
- The `expect` panics only when `slot / epoch_length > u32::MAX`, i.e. after `2^32` epochs. Every path that feeds a slot into it is bounded by wall-clock time: network blocks are rejected when `slot > current_slot` before the ledger is touched (`services/chain/chain-service/src/lib.rs:443-447`, then `:449` uncles, `:458-478` ledger, `:480-482` engine); uncle slots must be `< block slot` (`uncle.rs:44-51`) before their proof is checked against the parent's ledger state (`uncle.rs:128-142`); the internal `GetEpochState` query is issued only by the leader and Blend with tick slots (`services/chain/chain-leader/src/leadership.rs:434`, `services/blend/src/membership/chain.rs:152`) and has no HTTP route. `current_slot` itself comes from `Slot::from_offset_and_config` (`time.rs:151-169`), bounded by `now - genesis_time` where `genesis_time` is a `u32` unix timestamp. Ruled out as a remote panic.
- `Epoch` is `u32`, encoded little-endian on the wire (`time.rs:102-122`), matching spec revision 1.2.1. `Epoch::strict_add` / `strict_sub` panic on overflow regardless of profile (`time.rs:51-63`); the only `strict_sub(1)` callers (`ledger/src/config.rs:75, 112`) receive `next_epoch_state.epoch >= 1` (`ledger/src/cryptarchia/mod.rs:117, 145, 768`) or `epoch + 1` (`block_density.rs:30`), and the logging call at `services/chain/chain-service/src/service/mod.rs:743-748` skips epochs `<= 1`. The epoch-0 panic the two `*_panics_at_epoch_zero` tests document is unreachable in production. Ruled out.
- `Slot` arithmetic on network input: the ledger rejects `slot <= parent.slot` (`ledger/src/cryptarchia/mod.rs:264-269`) before anything else in the block path (`ledger/src/lib.rs:269`), so slot 0 and equal slots never reach the engine; `verify_header_alone` and `Block::new` also reject slot 0 (`core/src/block/mod.rs:180-182, 341-343`). `maxvalid_bg` adds `s_gen` to an accepted slot with a plain `+` (`consensus/cryptarchia-engine/src/lib.rs:107`); it cannot wrap because accepted slots are bounded by the clock. Remaining unchecked or truncating slot arithmetic is confined to code that is dead or needs `2^32` slots (S-003).
- Genesis: the genesis state is at slot 0, epoch 0, with `next_epoch_state` for epoch 1 seeded from the genesis note set and nonce (`ledger/src/cryptarchia/mod.rs:753-786`), as `bedrock-genesis-block.md` §Initial Epoch State requires; `D_GENESIS` is the sum of the distributed notes minus the faucet note (`:743-749`), a deliberate deviation the code comments and that the spec does not mention (S-001).

**Which epoch's stake distribution and nonce feed the lottery; snapshot distance.**

- For epoch `e` the eligible-note tree is the UTXO tree of the ledger state after the last block whose slot is `< (e-1) * epoch_length` (`ledger/src/config.rs:110-112`, `ledger/src/cryptarchia/mod.rs:145-155`, and at the transition `:339, 415`), i.e. the commitment root at the start of the previous epoch: spec `C_LEAD^ep = commitment_root_at_slot(sl_{ep-1}, tip)`. Distance to the epoch start: `10 * floor(k/f)` slots, above `s = 3 * floor(k/f)`.
- The nonce for epoch `e` is the ledger nonce after the last block whose slot is `< (e-1) * epoch_length + 6 * floor(k/f)` (`ledger/src/config.rs:71-90`, `mod.rs:117-140`): spec `eta^ep = epoch_nonce_at_slot(sl_{ep-1} + floor(6k/f))`, the last block before the Lottery Constants Finalization phase. Distance to the epoch start: `4 * floor(k/f)` slots, above `s`. The `<` on the *parent's* slot gives exactly "last block before the phase start": a block at the snapshot slot itself contributes to the next epoch's nonce only when it is a parent, and then its slot is not `<` the snapshot, so it is excluded. Verified against `test_epoch_transition` (`mod.rs:1382-1508`), which pins the nonce of epoch 1 to the block at slot 10 and ignores the blocks at 60 and 90.
- `D^e` is inferred at the first block of `e` from the distinct occupied slots in `[start(e-1), start(e-1) + 6 * floor(k/f))` (`block_density.rs:29-35, 44-54`, `mod.rs:304-307`), including uncle slots but only for referencing blocks inside the window, so no block after the window closes changes the estimate. Skipped epochs recurse with zero density (`mod.rs:371-380`), the spec's recursion with `N_BLOCKS = 0`. Public inputs of the proof use `epoch_state.utxos.root()`, `latest_utxos().root()`, `epoch_state.nonce` and the lottery values of the same epoch state (`mod.rs:503-510`), guarded by `assert_eq!(config.epoch(slot), self.epoch_state.epoch)` (`:502`), which holds by construction after `update_epoch_state`.
- Lottery constants `t0`, `t1` are computed from the rational `f` with 512-bit `astro_float` arithmetic (`zk/proofs/pol/src/lottery.rs:43-82`), not `f64`, so every node derives the same values. Ruled out as a determinism risk.
- The total-stake update uses `i128` intermediates with `PRECISION = 1000` (`stake.rs:32-51`). Bounds: `total_stake * 1000 <= 1.8e22`, `density_difference <= period * 1000`; with `period = 6 * floor(k/f) <= 6 * 2^32 * 2^32 / 1` the product stays below `i128::MAX`. The final `expect` (`:66-69`) needs `new_estimate > u64::MAX`, which with `beta <= 1` needs measured density about thirty times the expected one at an estimate above `6e17`, impossible while the estimate is at least a thirtieth of the true stake and the stake fits a `u64`. Ruled out for the shipped `beta` values; reachable only through configuration (LB-001).

**Security parameters per network.**

- `k` (`security_param`), `f` (`slot_activation_coeff`), `beta` (`learning_rate`), `W` (`uncle_reference_window_in_block`) and the three phase lengths are deployment settings (`nodes/node/binary/src/config/cryptarchia/deployment.rs:19-33`), embedded in the binary from `nodes/node/binary/src/config/deployment/settings.yaml` and regenerated per network by the genesis ceremony (`.github/workflows/genesis-ceremony.yml:32-47`). Values found: embedded/standalone `k = 30`, `f = 1/20`, `beta = 0.5` (`settings.yaml:20-30`); testnet and devnet templates `k = 120`, `f = 1/30`, `beta = 1.0` (`deployment/ceremony/genesis/{testnet,devnet}/deployment-template.yaml:20-30`); integration-test tool `k = 20`, `f = 1/10`, `beta = 0.1` (`tools/config/src/deployment.rs:50-62`). The spec states `k = 2160`, `f = 1/30`, `beta = 1.0` as constants.
- Nothing binds these values to the chain: the genesis inscription carries only `chain_id`, `genesis_time` and the genesis nonce (`core/src/mantle/transactions/genesis_tx.rs:381-385`, spec `bedrock-genesis-block.md` §Cryptarchia Parameters), and `Config::deserialize` (`config.rs:27-50`) and `EpochConfig` perform no cross-parameter validation. See LB-001.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Consensus schedule parameters are unbound to the chain and unvalidated at load | Configuration | Low | High | Open |
| LB-002 | The node's slot counter is a tick counter that lags the clock after missed ticks | Timing | Low | Medium | Open |
| LB-003 | Base period, `s_gen` and uncle window are derived in `f64` and differ from the spec's integer floor for some `(k, f)` | Determinism | Informational | High | Open |
| LB-004 | Spec deviation: the block-tree engine accepts a child at the same slot as its parent | Consensus | Informational | High | Open |

### LB-001 · Consensus schedule parameters are unbound to the chain and unvalidated at load

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Configuration |
| Target | `nodes/node/binary/src/config/cryptarchia/deployment.rs:L19-L33` (`Settings`), `consensus/cryptarchia-engine/src/config.rs:L27-L50` (`Config::deserialize`), `nodes/node/binary/src/cli/mod.rs:L443-L446`, `ledger/src/cryptarchia/stake.rs:L34-L46`, `consensus/cryptarchia-engine/src/config.rs:L107-L113` |
| Status | Open |

**Description**

Every quantity the epoch schedule and the lottery derive from — `k`, `f`, `beta`, `W`, and the three phase lengths — is a deployment setting:

```rust
// nodes/node/binary/src/config/cryptarchia/deployment.rs:19-33
pub struct Settings {
    pub epoch_config: EpochConfig,
    pub security_param: NonZeroU32,
    pub slot_activation_coeff: NonNegativeRatio,
    pub learning_rate: NonNegativeF64,
    pub uncle_reference_window_in_block: NonZeroU32,
    ...
    pub genesis_block: GenesisBlock,
```

Two nodes agree on them only because they run the same binary (the settings are embedded) and the same `--deployment` file if one is given (`cli/mod.rs:443-446`). Nothing on the chain commits to them: the genesis inscription carries `chain_id`, `genesis_time` and the nonce only (`core/src/mantle/transactions/genesis_tx.rs:381-385`), and the gossipsub topic name carries a version string, not the parameters. A node started with the correct genesis block and topic but a different `security_param` or `slot_activation_coeff` computes a different `epoch_length`, takes its snapshots at different slots and, from the first epoch boundary on, rejects every block of the network (`LedgerError::InvalidProof` at `ledger/src/cryptarchia/mod.rs:511-513`) without any log line naming the cause.

The values are also not sanity-checked when loaded. `Config::deserialize` (`config.rs:27-50`) only computes the lottery constants; `RewardPoWConfig` shows the pattern that is missing here (`ledger/src/config.rs:196-256`). Consequences of values the types accept:

- `f < 1/1000`: `trunc(f * PRECISION)` is `0` (`stake.rs:34-35`), `expected_density_with_precision` is `0` (`:40-41`) and `:44-46` divides by it. Integer division by zero panics in release too, at every epoch transition, on every node.
- `k < 4 / f` (for example `k = 1`, `f = 1/2`): `s_gen()` is `0` and its `expect` fires (`config.rs:107-113`) the first time the Bootstrap fork-choice rule runs.
- `beta > 1`: the correction exceeds the estimate; the result is clamped to `1` (`stake.rs:66-67`) on the way down but on the way up `learning_rate_with_precision * slot_activation_error` (`:47-49`) wraps in `i128` for large `beta`, and `:66-69` panics when the estimate exceeds `u64::MAX`.
- `W > 0.6 k`: the spec's virtual upper bound on the uncle window (`cryptarchia-v1-protocol.md` §Uncle Selection note) is not evaluated anywhere; the shipped values (`W = 12`, `k >= 20`) satisfy it.
- Phase lengths other than `3/3/4` change the observation window and the snapshot distances silently.

**Exploit scenario**

Not exploitable by an unprivileged participant: the parameters are read from the operator's own configuration. The realistic failure is operational. An operator on testnet passes `--deployment` with a file derived from `nodes/node/standalone-deployment-config.yaml` (`k = 30`, `f = 1/20`) instead of the testnet template (`k = 120`, `f = 1/30`). The node syncs epoch 0 (the genesis epoch state is hard-coded), then at slot `36000` the network moves to epoch 1 while the node, with `epoch_length = 6000`, is at epoch 6 with a nonce snapshotted at slot `3600` instead of `21600`: every subsequent block fails proof verification and the node is stuck at the boundary, reporting only `InvalidProof`.

**Recommendation**

- *Short term*: validate at deserialisation, as `RewardPoWConfig::validate` does: `slot_activation_coeff.numerator * PRECISION >= denominator` (so `f >= 1/PRECISION`), `k >= 4 / f`, `learning_rate <= 1`, `W <= 0.6 k`, and log the derived `epoch_length`, `base_period_length`, `s_gen` and snapshot offsets at startup so a mismatch is visible.
- *Long term*: commit the consensus parameters to the chain — either in the genesis inscription (extend `CryptarchiaParameter`) or as a hash of the canonical parameter set that peers exchange on the chainsync/identify handshake — so a node with different values cannot join the network silently. Raise the specification question in S-001.

**References**: `cryptarchia-v1-protocol.md` §Constants and §Uncle Selection (virtual bound on `W`); `cryptarchia-total-stake-inference.md` §Algorithm (`PRECISION = 1e3`); `bedrock-genesis-block.md` §Cryptarchia Parameters.

### LB-002 · The node's slot counter is a tick counter that lags the clock after missed ticks

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Timing |
| Target | `services/time/src/backends/common.rs:L15-L33` (`slot_timer`), `consensus/cryptarchia-engine/src/time.rs:L319-L331` (`SlotTimer::slot_interval`), `services/time/src/backends/ntp/mod.rs:L56-L100` (`tick_stream`), `L148-L213` (`handle_ntp_update`), `consensus/cryptarchia-engine/src/time.rs:L151-L169` (`Slot::from_offset_and_config`) |
| Status | Open |

**Description**

The slot ticks the whole node consumes (`current_slot` in the chain service phases, `services/chain/chain-service/src/service/phases/following.rs:77` and siblings; the leader's epoch scans; Blend) are produced by zipping a `tokio` interval with a `+1` counter:

```rust
// services/time/src/backends/common.rs:24-31
IntervalStream::new(SlotTimer::new(slot_config).slot_interval(datetime))
    .zip(futures::stream::iter(std::iter::successors(
        Some(current_slot.strict_add(1.into())),
        |&slot| Some(slot.strict_add(1.into())),
    )))
```

`slot_interval` sets `MissedTickBehavior::Skip` (`time.rs:329`). When the time service's task is not polled for longer than one slot — a starved runtime, a process suspension, a clock step — the interval skips the missed ticks and fires once, but the counter advances by exactly one. Each missed tick therefore lags `current_slot` behind the clock by one slot, permanently for the system backend and, for the NTP backend the node uses (`nodes/node/binary/src/config/time/mod.rs:22-59`), until the next successful NTP response rebuilds the timer from `Slot::from_offset_and_config` (`ntp/mod.rs:184-192`). The rebase runs every `update_interval` (default 15 s, `nodes/node/binary/src/config/time/serde.rs:31-35`) and is skipped with a warning when the server does not answer (`ntp/mod.rs:70-76`), so the lag is unbounded while NTP is unreachable.

A lagging node treats valid blocks as `FutureBlock` (`services/chain/chain-service/src/lib.rs:443-447`); the network layer retries only `3 x 500 ms` (`services/chain/chain-network/src/lib.rs:78-79, 950-990`) and then drops the block with a warning, without enqueuing it as an orphan (`:674-681`). The leader scans and proposes at stale slot numbers.

A second inconsistency in the same code: the clock-to-slot conversion divides whole seconds by `slot_duration.as_secs()` (`time.rs:163-167`), while `slot_interval` ticks with the full `Duration`. `slot_duration` is bounded below by one second but not required to be whole (`MinimalBoundedDuration<1, SECOND>`, `time.rs:293`; `nodes/node/binary/src/config/time/deployment.rs:10-11`). With `slot_duration = 1.5 s` every NTP rebase resets the slot to `seconds / 1` while ticks advanced one per `1.5 s`, so the slot number jumps forward at each rebase. The shipped configurations use exactly `1 s`.

**Exploit scenario**

Not attacker-driven. A node whose runtime stalls for `n` seconds (heavy proof verification on all worker threads, a paused container, a VM migration) with the NTP pool temporarily unreachable runs `n` slots behind. Blocks arriving at the true slot are rejected as future for `1.5 s` and dropped; the node falls behind the tip until the next successful NTP rebase, then fetches the missing blocks through tip-poll or orphan download. If the node is a leader it proposes blocks whose slot is `n` behind; they are valid for peers (their slot is in the past) but may lose the race to blocks proposed at the true slot.

**Recommendation**

- *Short term*: derive the slot of each tick from the clock (`Slot::from_offset_and_config(now, config)`) instead of a counter, and use `MissedTickBehavior::Burst` or an explicit "emit every slot up to now" loop so no slot number is skipped or repeated; reject `slot_duration` values that are not a whole number of seconds, or divide in nanoseconds.
- *Long term*: make the time service the only place that computes `Slot` from wall-clock time, expose it as a `watch` of the latest slot rather than a stream, and add a metric for the difference between the clock-derived slot and the emitted slot.

**References**: `cryptarchia-v1-protocol.md` §Block Header Validation step 6 (`wallclock_time().to_slot() >= header.slot`), §Clocks.

### LB-003 · Base period, `s_gen` and uncle window are derived in `f64` and differ from the spec's integer floor for some `(k, f)`

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Determinism |
| Target | `consensus/cryptarchia-engine/src/config.rs:L107-L113` (`s_gen`), `L125-L131` (`average_slots_for_blocks`), `L145-L164` (`expected_blocks_per_epoch`) |
| Status | Open |

**Description**

`floor(k / f)`, `floor(k / (4 f))` and `floor(W / f)` are computed as `(k as f64 / (numerator as f64 / denominator as f64)).floor() as u64` (`config.rs:109-112, 129`). For a ratio whose reciprocal is not exactly representable, the double division rounds below the integer and `floor` loses one: `k = 1, f = 1/93` gives `92` (`1/93 = 0.010752688...`, `1 / that = 92.99999999999999`), `k = 2, f = 1/186` gives `185`, `s_gen` for `k = 4, f = 1/93` gives `92`. Checked with IEEE-754 doubles: the deployed pairs (`k in {20, 30, 120, 2160}` with `f in {1/10, 1/20, 1/30}`) all produce the exact value (`600`, `3600`, `64800`; `s_gen` `150`, `900`, `16200`), so no shipped network is affected. The result is identical on every node (IEEE-754 division is deterministic and the Rust `const fn` is evaluated the same way everywhere), so there is no split risk; the defect is that `epoch_length` then disagrees with the spec formula, and with `expected_blocks_per_epoch` (`config.rs:145-164`), which is computed exactly in `u128` and no longer equals `10 k` — the value the PoW payout rate is denominated in (`ledger/src/config.rs:37-63`).

**Exploit scenario**

None. A future parameter change to a ratio such as `1/93` would shorten every base period by one slot and mis-scale `expected_blocks_per_epoch` relative to the spec, without any test catching it (`test_config` and `test_expected_blocks_per_epoch_is_ten_k` use `1/5`, exact in binary at `k = 10`).

**Recommendation**

- *Short term*: compute `floor(k * denominator / numerator)` in `u64`/`u128` (as `expected_blocks_per_epoch` already does) for `base_period_length`, `s_gen` and `uncle_reference_window_in_slot`.
- *Long term*: keep `f64` out of every quantity that enters consensus state; `NonNegativeRatio::as_f64` (`utils/src/math.rs:250-253`) should not be reachable from the schedule derivation or from `StakeInference::new` (`ledger/src/cryptarchia/mod.rs:754-758`, `stake.rs:15`), which re-truncates the same ratio.

**References**: `cryptarchia-v1-protocol.md` §Notation (`s = 3 floor(k/f)`), §Epoch Schedule; `fork-choice.md` §Definitions (`s_gen = floor(k / 4f)`); `cryptarchia-total-stake-inference.md` §Parameters (`PERIOD = 6 floor(k/f)`).

### LB-004 · Spec deviation: the block-tree engine accepts a child at the same slot as its parent

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Consensus |
| Target | `consensus/cryptarchia-engine/src/lib.rs:L245-L247` (`Branches::apply_header`) |
| Status | Open |

**Description**

```rust
// consensus/cryptarchia-engine/src/lib.rs:245-247
if parent_branch.slot > slot {
    return Err(Error::InvalidSlot(parent));
}
```

Spec step 5 of Block Header Validation requires `header.slot > parent.slot`. The engine's own guard allows equality. It is not reachable from the network at this commit: the ledger rejects `slot <= parent.slot` (`ledger/src/cryptarchia/mod.rs:264-269`) and `prepare_update` runs before `receive_block_with_canonical_change` (`services/chain/chain-service/src/lib.rs:458-482`), so I believe the code is wrong only in the sense that the engine relies on a check it does not own. Any future caller that inserts headers into the engine without the ledger (sync tooling, tests, a headers-first path) inherits the gap.

**Exploit scenario**

None at this commit.

**Recommendation**

- *Short term*: change the guard to `parent_branch.slot >= slot`.
- *Long term*: add an engine unit test for equal slots so the invariant is owned where the block tree is.

**References**: `cryptarchia-v1-protocol.md` §Block Header Validation step 5.

## 5. Suggestions (non-security)

### S-001 · Specification: consensus parameters are constants in the spec but per-network in the implementation, and their constraints are unstated

| | |
|---|---|
| Target | `cryptarchia-v1-protocol.md` §Constants; `bedrock-genesis-block.md` §Cryptarchia Parameters; `nodes/node/binary/src/config/deployment/settings.yaml:L20-L30` |

`cryptarchia-v1-protocol.md` fixes `k = 2160` and `f = 1/30`, `cryptarchia-total-stake-inference.md` fixes `beta = 1.0`, and the genesis inscription carries no consensus parameter. The implementation ships `k = 30, f = 1/20, beta = 0.5` embedded and `k = 120, f = 1/30, beta = 1.0` for testnet and devnet, and derives `D_GENESIS` excluding the faucet note (`ledger/src/cryptarchia/mod.rs:743-749`), which `bedrock-genesis-block.md` §Initial Epoch State does not mention. The specification should say which of these are per-network parameters, how nodes agree on them (LB-001 recommends the genesis inscription), and the constraints between them where each is defined: `f >= 1/PRECISION` (else the inference divides by zero), `k >= 4 / f` (else `s_gen = 0`), `beta <= 1`, `W <= 0.6 k`, and whether the faucet note is excluded from `D_GENESIS`.

### S-002 · Truncating `f` to three decimals biases the total-stake estimate by about one percent at `f = 1/30`

| | |
|---|---|
| Target | `ledger/src/cryptarchia/stake.rs:L34-L41`; `cryptarchia-total-stake-inference.md` §Algorithm |

Both the spec pseudocode and the code use `trunc(f * 1000) = 33` for `f = 1/30`, so the expected density is `period * 0.033` while the lottery targets `period * 0.0333...`. The update's fixed point is the estimate at which the measured density equals the truncated expectation, `D = S * (1/30) / 0.033 = 1.0101 S`: the estimate settles one percent above the true stake and the block rate one percent below `f`. The code holds `f` as an exact ratio (`NonNegativeRatio`); the expected density can be `period * numerator / denominator` with no truncation, and the spec's `PRECISION` applied only to `beta`. A spec change is needed first, since the two must agree.

### S-003 · Unchecked or dead slot arithmetic outside the consensus path

| | |
|---|---|
| Target | `consensus/cryptarchia-engine/src/time.rs:L267-L274, L322`; `consensus/cryptarchia-engine/src/lib.rs:L107` |

- `EpochConfig::last_slot` (`time.rs:272-274`) adds `1` to a `u32` epoch and subtracts `1` from a `u64` product with plain operators; it wraps in release at `u32::MAX`. It has no production caller (`ledger/src/config.rs:121-124` wraps it, also uncalled). Delete it or use checked arithmetic.
- `EpochConfig::starting_slot` (`time.rs:267-269`) multiplies `u32 as u64 * epoch_length` unchecked; it wraps only for an `epoch_length` above `2^32`, which `average_slots_for_blocks` can produce for a tiny `f` (it saturates to `u64::MAX`). Tie to the validation in LB-001 or use `checked_mul`.
- `SlotTimer::slot_interval` (`time.rs:322`) casts the next slot number to `u32` before multiplying the slot duration: after `2^32` slots (136 years at 1 s) the next tick is scheduled in the past and `Duration::try_from(delay)` fails its `expect` (`:326`). Multiply as `u64` seconds instead.
- `maxvalid_bg` (`lib.rs:107`) adds `s_gen` to an accepted slot with `+`; use `saturating_add` so the invariant does not depend on the future-slot check living in another crate.

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
