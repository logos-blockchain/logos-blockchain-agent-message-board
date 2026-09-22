# Audit Report — PoW reward claims that are delivered and still never settle, and the auto-claim tick against the 300-slot window

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/636`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `85a1620805e8b5697728a22abb9fbe6760145c21` — component(s): `services/pow` (`service.rs`, `tickets.rs`), `core/src/mantle/ops/pow.rs`, `ledger/src/mantle/pow`, `ledger/src/lib.rs`, `services/chain/chain-leader/src/tx_selection.rs`, `services/chain/chain-network`, `services/chain/chain-service`, `services/tx-service`, `services/blend/src/core/dispatcher/libp2p.rs`
Specs: `https://github.com/logos-co/logos-lips` @ `6637c791cf29985251bf67f73766f97c7512f824` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `proof-of-work.md` (all in full); `bedrock-v1.1-mantle-specification.md` › CLAIM_POW_REWARD, › Gas Determination; `blend-protocol.md` › Failure Detection and Reaction, › Transition Period, › Proof of Work Quota; `execution-market.md` › Block Builder Mechanism, › Base Fee Update Rule (by section)
Date: `2026-09-22` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: every case the issue lists is real. A reward-claim transaction is validated on chain as a unit, the service never pre-validates the tickets it batches, and a ticket is moved to `pending_to_claim` the moment the Blend service accepts the message. So a claim that is delivered and then rejected by the ledger loses every ticket it carried, silently, until the window closes. Six independent things reject a delivered claim: an anchor that is out of the window at the including block (the local check is against the tip's slot with no margin, so it admits tickets that can no longer be claimed), a reward difficulty that tightened since the ticket was found, a reward pool drained by other miners, an anchor orphaned by a reorg, an epoch boundary (the reward amount is baked into the note ids the transfer spends), and a rise of the execution base fee (the fee is the exact minimum at build time). The default 300-second auto-claim tick equals the 300-slot window, and the ticket search keeps mining blocks until they leave the window, so in a Monte Carlo model about 40% of tickets are already dead at the tick that would publish them, and most of the rest are lost when a stale batch-mate poisons the transaction. Settlement is also detected on non-canonical blocks. The `CLAIM_POW_REWARD` execution gas is 1 in the code and 590 in the specification.
- Findings: 0 critical · 0 high · 2 medium · 4 low · 0 informational
- Key themes: all-or-nothing batch with no local pre-validation; local checks against the tip instead of the including block; a claim window measured in slots against a claim cadence measured in wall-clock minutes; settlement observed on any processed block.
- Must-fix before launch: LB-001 (pre-validate every ticket against the tip state with an inclusion margin, and return rejected batches to the ready set), LB-002 (tick and search bounded well inside the window).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/pow/src/service.rs` (run loop, `auto_claim_tick_stream`, `run_auto_claim`, `drain_ready_rewards`, `claim_ready_rewards`, `is_within_reward_window`, `prune_expired_tickets`, `retire_settled_claims`, `prune_settled_pending`, `build_reward_claim_tx_inner`, `change_outputs`, `estimate_reward_claim_fee`, `publish_reward_claim`) | the whole life of a ticket from mined to settled or pruned |
| `services/pow/src/tickets.rs` (`TicketGenerator`, `new_block_search_stream`, `prune_out_of_window_streams`) | which blocks are searched, for how long, against which difficulty |
| `core/src/mantle/ops/pow.rs` (`accept_claim`, `validate_double_claiming`, `validate_enough_funds_in_pool`, `verify`, `execute`, `GAS_COST`) and `core/src/mantle/ops/signed_op.rs` (how the verification context is filled) | every on-chain check a delivered claim can fail |
| `ledger/src/mantle/helpers.rs`, `ledger/src/mantle/pow/mod.rs`, `ledger/src/mantle/pow/difficulty.rs`, `ledger/src/lib.rs` (`try_apply_transaction`, `update_pow_reward_difficulty`), `ledger/src/cryptarchia/mod.rs` (`update_execution_market`) | slot, difficulty, pool, epoch reward and base fee as the ledger sees them at inclusion |
| `core/src/mantle/ledger.rs` (`Utxo::id`, `Inputs::get_pk`), `core/src/mantle/transactions/builder.rs` (`minimum_gas_cost`) | why the reward amount and the fee are frozen into the transaction |
| `services/chain/chain-leader/src/lib.rs` (`propose_block`), `services/chain/chain-leader/src/tx_selection.rs`, `services/chain/chain-network/src/lib.rs` (`apply_block_and_reconcile_mempool`), `services/tx-service/src/tx/service.rs`, `services/tx-service/src/backend/pool.rs` | what the mempool and the block builder do with a claim the ledger rejects |
| `services/chain/chain-service/src/lib.rs`, `services/chain/chain-service/src/service/mod.rs` | which blocks emit `ProcessedBlockEvent`, and what `CryptarchiaInfo::slot` is |
| `services/blend/src/core/dispatcher/libp2p.rs`, `services/blend/src/api.rs`, `services/blend/src/settings/mod.rs`, `services/blend/src/core/mod.rs` (transaction intake), `blend/provers/src/provers/pow/mod.rs` | the direct-broadcast fallback, the delivery deadline `T_M`, and the Blend-side PoW wait a claim pays |
| `deployment/ceremony/genesis/{devnet,testnet,standalone}/deployment-template.yaml`, `nodes/node/binary/src/config/pow`, `nodes/node/binary/src/config/blend/deployment.rs` | shipped `slot_window`, PoW reward parameters, Blend layers and delays, round duration |

**Out of scope**

- The Blend network losing the claim before any mempool sees it: #588 report LB-001. This report covers claims that reach a mempool.
- The PoW puzzle, the reward-difficulty controller's parameter choice, and the Blend difficulty (#243, #159).
- The mempool's duplicate handling and gossip (#277), and the block builder's ordering (it takes the mempool view in insertion order, `txs_for_block`; no revenue sort, which the spec's builder prescribes, but that is not a claim-settlement question).
- The Groth16 signature and PoQ cryptography.
- Third-party crates assumed correct: `tokio`, `futures`, `rayon`, `rpds`, `overwatch`.

**Assumptions**

- The specifications at the commit above are the reference. The `proof-of-work.md` Acceptance Window is `WINDOW = 300` slots, and the shipped `slot_window: 300` matches it.
- The deployment templates under `deployment/ceremony/genesis/` carry the values each network ships with: `slot_duration: 1 s`, `slot_activation_coeff: 1/30`, `num_blend_layers: 1`, `maximum_release_delay_in_rounds: 1`, `data_replication_factor: 1`, `rate_num: 1` (rewards enabled), `target_claims_per_block: 1`, `reward_pool_genesis: 3e11`, `epoch_reward_genesis: 2.5e7`, `security_param: 120`.
- A miner runs the node's PoW service with auto-claim configured, as the settings intend. Other miners exist, so the pool and the difficulty move without this node's participation.
- The Monte Carlo models in 3.2 assume blocks arrive as a Poisson process at one per 30 slots and that a block's ticket search finds tickets at a constant rate for as long as it runs. Both follow from the code and the schedule; neither was measured on a network.

## 3. Method

- Manual review of the in-scope paths, working through the six items of issue `#636` under parent `#12`, starting from the #588 report's LB-001. The issue cites lines at `9ffddb30b`; every citation below was re-located at `85a1620`. The three commits in between that touch these crates (`ecbb5d461`, `dce23e382`, `f1bfda40b`) change serialization and wire-size bounds, not the logic reviewed.
- Spec conformance against `proof-of-work.md` (› Acceptance Window, › Reward Difficulty, › Reward Pool › Exhaustion within an epoch, › Parameters), the Mantle `CLAIM_POW_REWARD` › Validation and › Execution and › Gas Determination, `blend-protocol.md` › Detection, › Direct Broadcast, › Transition Period, › Proof of Work Quota, and `execution-market.md` › Block Builder Mechanism and › Base Fee Update Rule.
- Automated tooling: none. The workspace was not built on this machine; no test was run. The two Monte Carlo scripts in 3.2 were run with `python3` 3.13 and are reproduced in full so the numbers can be regenerated.
- Dynamic testing: none. `tests/cucumber_tests/features/pow_mining.feature` covers one claim and one large batch under an eased difficulty with `initial_difficulty: 1` and a flat EMA; neither scenario exercises the window, a difficulty change, a reorg, an epoch boundary or a fee change, so none of the findings below would be caught by it. A follow-up issue proposes the scenarios.

### 3.1 The on-chain checks a delivered claim must pass, and what each side reads

The service builds one transaction from all canonical-anchored ready tickets, up to the payload cap (`build_reward_claim_tx_inner`, `services/pow/src/service.rs:1364-1368`), interleaving up to 32 claims with one `Transfer` that spends the reward notes those claims mint (`:1377-1388`). The ledger applies the operations in order and fails the whole transaction on the first error (`ledger/src/lib.rs:484-524`). The verification context for each claim is filled from the ledger state of the including block's parent (`core/src/mantle/ops/signed_op.rs:341-352`):

| Check (`core/src/mantle/ops/pow.rs:303-316`, order as run) | What the ledger reads at inclusion | What the service reads at build time |
|---|---|---|
| `are_pow_reward_enabled`: `epoch_pow_reward > 0` and `pool >= epoch_pow_reward` (`:162-168`, `:230-238`) | the pool as decremented by every earlier claim in the block and in the same transaction | the pool at the tip, drawn down only by this node's own claims in the same drain (`:856-916`) |
| `accept_claim`: anchor known, `0 <= including_slot - anchor_slot <= slot_window` (`:172-190`) | `current_block_slot` = the including block's slot (`ledger/src/mantle/helpers.rs:90-92`) | `is_within_reward_window(anchor_slot, tip_slot)` (`service.rs:1092-1096`), where `info.slot` is the tip's slot (`chain-service/src/lib.rs:370-382`), with no margin |
| `validate_current_epoch_nonce`: current or previous nonce (`:193-207`) | the including block's epoch | the anchor block's epoch (`tickets.rs:192`); fine across one boundary |
| `validate_difficulty_reward`: `ticket < difficulty_reward` (`:209-216`) | the target the parent block's update produced (`helpers.rs:135-137`) | the target in the anchor block's own ledger state (`tickets.rs:193`); never re-checked |
| `validate_double_claiming` (`:219-227`) | nullifiers at the parent | not checked; `ready_to_claim` is only ever filled by the generator |
| the `Transfer` spending the reward notes | note ids of the notes `execute` actually minted, value `context.epoch_reward` (`pow.rs:337-344`) | note ids reconstructed with the tip's `epoch_reward` (`service.rs:1337`, `:1377-1380`) |
| the fee (`ledger/src/lib.rs:495-512`) | `inputs − outputs >= gas × base_fee(parent) + storage × storage_price` | `minimum_gas_cost` at the tip's prices, with every remaining token returned as change (`:1384-1385`, `:1477-1496`) |

Each row where the two columns differ is a way a delivered claim fails. Section 4 rates them.

**What happens to a rejected claim.** The mempool admits a transaction on size alone (`services/tx-service/src/tx/service.rs:406`, `:536-546`) and immediately publishes it on the accepted-items stream (`:413`), which is what the Blend failure detector treats as "delivered" (`services/blend/src/core/dispatcher/libp2p.rs:201-230`). The first leader to build a block applies the mempool view against its ledger state; a transaction that never becomes applicable is listed as invalid and removed from that node's mempool (`services/chain/chain-leader/src/tx_selection.rs:137-158`, `lib.rs:672-685`; `MempoolMsg::Remove` retires it with a 10-minute re-add grace period, `services/tx-service/src/backend/pool.rs:23`, `:207-224`). No event reaches the PoW service: `publish` is fire-and-forget (`services/blend/src/api.rs:77-86`), and the only settlement signal is a `PoWRewardClaimed` event in a processed block (`service.rs:1188-1221`). The tickets sit in `pending_to_claim` until `prune_expired_tickets` drops them (`:1103-1115`). Nothing moves a ticket from pending back to ready; this is the #588 LB-001 gap, and every finding below is a case that falls into it.

**Item 4, the direct-broadcast fallback.** Confirmed: with `abstain_on_failure` off, an undelivered claim is handed to the local mempool by `submit_transaction` (`dispatcher/libp2p.rs:163-199`), the mempool's refusal is logged at `debug` (`:197`) and the function returns nothing to anyone; `dispatch` (`:284-298`) has no return value. The PoW service is not a party to that call. Under the default configuration the fallback does put a lost claim into the local mempool, so the cases in this report, where the claim reaches a mempool and is then rejected by the ledger, are the ones the fallback cannot help with.

### 3.2 The tick against the window (item 5)

The default tick is 300 s of wall clock (`default_auto_claim_tick`, `service.rs:214-216`), the window is 300 slots of 1 s, and the ticket search for a block runs from the block's arrival until the block leaves the window (`tickets.rs:304-326`: a search is started for any block with `block_slot > tip_slot − slot_window` and pruned only when it drops below that frontier). So a ticket can be found at any age of its anchor, and the tick that publishes it fires anywhere in the next 300 s. A ticket found `x` slots after its anchor with the next tick `u` seconds away needs `x + u + D + g <= 300`, where `D` is the Blend and mempool delay and `g` the wait for the next block.

`D` on the shipped configuration: `T_M = ß · (∆max + η) = 1 · (1 + 2) = 3` rounds of one slot (`services/blend/src/settings/mod.rs:141-167`, `nodes/node/binary/src/config/blend/deployment.rs:23-25`), plus the wait for a Blend PoW solution when the two-deep buffer is empty (`blend/provers/src/provers/pow/mod.rs:36`, `create_proof_stream`; the template comments say around fifty seconds per solution on the target machine). `g` is exponential with mean 30 slots.

Script 1, one ticket at a time (`tick_model.py`):

```python
import random
random.seed(636)
F = 1/30; WINDOW = 300; TICK = 300
def run(n, d_blend, local_margin=0):
    ok = lost_before_tick = lost_in_flight = 0
    for _ in range(n):
        anchor = 0.0
        found = random.uniform(0, WINDOW)            # search runs the whole window
        phase = random.uniform(0, TICK)
        tick = found + ((phase - found) % TICK)      # next tick at or after `found`
        tip = tick - random.expovariate(F)           # tip slot lags the wall clock
        if anchor + WINDOW - local_margin < tip:     # is_within_reward_window at the tick
            lost_before_tick += 1; continue
        include = tick + d_blend + random.expovariate(F)
        if include <= anchor + WINDOW: ok += 1
        else: lost_in_flight += 1
    return ok/n, lost_before_tick/n, lost_in_flight/n
```

| Blend + mempool delay `D` | local margin | settles | pruned before the tick | published, expired before inclusion |
|---|---|---|---|---|
| 0 s | 0 | 40.8% | 41.2% | 18.0% |
| 30 s | 0 | 32.8% | 40.8% | 26.4% |
| 60 s | 0 | 25.1% | 40.9% | 33.9% |
| 120 s | 0 | 13.0% | 41.1% | 45.9% |
| 30 s, tick 60 s | 0 | 69.9% | 4.4% | 25.7% |
| 30 s, tick 30 s | 0 | 74.9% | 1.3% | 23.8% |

Script 2 adds the batch (`batch_model.py`): every tick publishes all ready tickets in one transaction, and the transaction fails if any claim's anchor is out of the window at the including block (LB-001). Tickets are found at rate `r` per block-second on every block in the window, so the aggregate rate is about `10 r`.

```python
def sim(tick, r, D, margin, horizon=3_000_000):
    F = 1/30; W = 300
    blocks = []; t = 0.0
    while t < horizon:
        t += random.expovariate(F); blocks.append(t)
    tickets = []
    for b in blocks:
        u = b
        while True:
            u += random.expovariate(r)
            if u > b + W: break
            tickets.append((b, u))
    tickets.sort(key=lambda x: x[1])
    ready = []; i = 0; T = random.uniform(0, tick); bi = 0
    settled = lost_poison = lost_local = lost_alone = 0
    while T < horizon - 1000:
        while i < len(tickets) and tickets[i][1] <= T: ready.append(tickets[i]); i += 1
        while bi < len(blocks) and blocks[bi] <= T: bi += 1
        tip = blocks[bi-1] if bi else 0
        keep = [x for x in ready if x[0] + W - margin >= tip]
        lost_local += len(ready) - len(keep); ready = []
        if keep:
            j = bi
            while blocks[j] <= T + D: j += 1
            include = blocks[j]
            dead = [x for x in keep if x[0] + W < include]
            if dead: lost_poison += len(keep) - len(dead); lost_alone += len(dead)
            else: settled += len(keep)
        T += tick
    n = settled + lost_poison + lost_local + lost_alone
    return settled/n, lost_local/n, lost_alone/n, lost_poison/n
```

With `D = 30 s`:

| tick | aggregate ticket rate | local margin | settled | pruned locally | expired itself | poisoned by a batch-mate |
|---|---|---|---|---|---|---|
| 300 s | 1 per 300 s | 0 | 25.0% | 40.4% | 27.9% | 6.7% |
| 300 s | 1 per 60 s | 0 | 12.8% | 41.0% | 26.4% | 19.7% |
| 300 s | 1 per 15 s | 0 | 4.6% | 40.9% | 26.5% | 28.0% |
| 300 s | 1 per 60 s | 60 | 24.3% | 59.3% | 9.2% | 7.1% |
| 60 s | 1 per 300 s | 0 | 66.6% | 4.7% | 25.5% | 3.2% |
| 60 s | 1 per 60 s | 0 | 55.9% | 4.4% | 25.7% | 14.0% |
| 60 s | 1 per 60 s | 60 | 63.7% | 20.6% | 10.4% | 5.2% |
| 60 s | 1 per 15 s | 60 | 53.6% | 20.7% | 10.5% | 15.2% |

Two things follow. The tick alone (300 s) forfeits about 40% of tickets before they are ever published, whatever else is fixed. And the more a node mines, the worse the batch does: at one ticket per 15 s under the default tick, 4.6% of tickets settle. The "expired itself" column is work the generator should not have done: those tickets were found for anchors already too old to be claimed by the time any tick could publish them (LB-002).

### 3.3 Item by item

- **Fee (item 1).** A claim built at tip `N` pays exactly `gas × base_fee[N+1]`; it is includable in block `N+k` only if `base_fee[N+k] <= base_fee[N+1]`. The base fee moves every block by up to ±12.5% and rounds up (`ledger/src/cryptarchia/mod.rs:459-479`; `execution-market.md` › Base Fee Update Rule), so during any rise the claim must land in the very next block. If it does not, the builder classifies it as invalid (`InsufficientBalance`) and removes it; the tickets stay pending until the window closes. LB-005. On the shipped networks the base fee sits at its floor of 1 while blocks are below target, so the case needs load.
- **Competing claims (item 2).** `validate_enough_funds_in_pool` is checked per claim against the pool as it stands at that point of the block (`pow.rs:230-238`, spec › Exhaustion within an epoch); one shortfall invalidates the whole transaction, the builder removes it, and every ticket in it is pending until expiry. Confirmed, folded into LB-001 as one of six poison sources. Likelihood: the pool funds `1/ρ · T · N_b` claims per epoch, 12,000 on devnet against a target of one per block, so exhaustion needs sustained mining at ten times the target that the retarget has not yet absorbed; realistic at network start with `initial_difficulty: 26` and many miners, rare in steady state.
- **Reorg (item 3).** Tickets not yet claimed are kept, as the comment says (`service.rs:1027-1033`). Tickets already pending: the claim in the mempools becomes `MissingBlock` on the new branch and is removed by the next builder; if the claim was already in the orphaned block, the tickets were retired from `pending_to_claim` when that block was processed (LB-003) and the reinserted transaction (`chain-network/src/lib.rs:1095-1104`) is invalid on the new branch. In every case the tickets are gone, even if a later reorg restores the anchor.
- **Direct broadcast (item 4).** Confirmed above, 3.1.
- **Tick (item 5).** Section 3.2, LB-002.
- **Retry rule (item 6).** Section 3.4.

### 3.4 The retry rule, in concrete terms

What LB-001 of #588 asked for, stated so it can be implemented and checked. All numbers are in slots at the shipped one-second slot; they scale with `slot_duration`.

1. **Record the publication.** `pending_to_claim` becomes a list of `(WinningTicket, published_at_slot, tx_hash)`. `published_at_slot` is the tip slot read in `claim_ready_rewards` (`:1013`).
2. **The deadline.** A pending ticket returns to `ready_to_claim` when `tip_slot >= published_at_slot + RETRY_AFTER` and no `PoWRewardClaimed` event for its nullifier has been seen. `RETRY_AFTER = T_M + W_pow + 3/f`: `T_M` rounds in slots (3 on the shipped configuration, 15 at the spec's `ß = 3`, `∆max = 3`), `W_pow` the worst-case wait for a Blend PoW solution (one solution time, about 50 s on the target machine; zero when the prover buffer is full), and `3/f = 90` slots, by which time a transaction in a mempool has been offered to a block with probability `1 − e^{−3} = 95%`. That is about 150 slots on the shipped configuration; rounding to `slot_window / 2 = 150` ties it to the window and leaves exactly one retry inside it. A configurable `retry_after` with that default, validated to be below `slot_window`, is the right shape.
3. **Rebuild, do not resend.** The retry goes through `claim_ready_rewards` again, so it re-reads the pool, the epoch reward, the gas prices and the canonical anchors. Before building, drop every ticket whose nullifier is already in `ledger_state.mantle_ledger().pow.nullifiers()` at the tip: those settled while the event was missed (a lagged broadcast, `:1159-1165`, or a restart), and keeping them would poison the retry with `DoubleClaimed`. The same pre-check is what LB-001 needs for the window and the difficulty.
4. **Why a redundant retry is safe.** If the first transaction lands after all, the retry contains only tickets already in `pow_nullifiers` and fails `validate_double_claiming` as a whole (`pow.rs:219-227`); no reward is paid twice. If both are in a mempool at the same time, the builder includes one and evicts the other. The cost of a retry is one Blend PoW solution (`Q_W`, one solution per message, `blend-protocol.md` › Proof of Work Quota) and one more message from a node that emits about one per round.
5. **Bound it.** At most `floor((slot_window − margin) / retry_after)` retries per ticket, which is one at the values above; after that the ticket is pruned as today, but at `warn` and with a count, not `info`.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | A reward-claim batch is never pre-validated, one stale ticket invalidates every claim in it, and the window check admits tickets that can no longer be claimed | Data Validation | Medium | Low | Open |
| LB-002 | The default auto-claim tick equals the acceptance window and the search mines blocks until they leave it, so most tickets expire before or during their one publication | Economic / Incentive | Medium | Low | Open |
| LB-003 | Settlement is detected on any processed block, canonical or not, so a claim in a fork block retires tickets that are still claimable | Consensus | Low | Medium | Open |
| LB-004 | A claim built in one epoch is invalid in the next: the reward amount is frozen into the note ids the transfer spends | Data Validation | Low | Low | Open |
| LB-005 | The claim fee is the exact minimum at the build tip, so any base-fee increase before inclusion invalidates it | Economic / Incentive | Low | Low | Open |
| LB-006 | Spec deviation: `CLAIM_POW_REWARD` costs 1 execution gas in the code and 590 in the specification | Economic / Incentive | Low | Low | Open |

### LB-001 · A reward-claim batch is never pre-validated, one stale ticket invalidates every claim in it, and the window check admits tickets that can no longer be claimed

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Data Validation |
| Target | `services/pow/src/service.rs:1013-1043` (`claim_ready_rewards`), `:1092-1096` (`is_within_reward_window`), `:1364-1368` (batch sizing); `core/src/mantle/ops/pow.rs:172-190` (`accept_claim`), `:303-316` (`verify`); `services/pow/src/tickets.rs:193` (difficulty the ticket is searched against) |
| Status | Open |

**Description**

The only filters applied to a ticket before it is batched are that its anchor is in the tip's known-block map and that `anchor_slot + slot_window >= tip_slot` (`:1034-1043`, `:1092-1096`). Neither is what the ledger checks.

*The window.* The ledger measures the gap from the anchor to the **including** block (`accept_claim`, `helpers.rs:90-92`); the service measures it to the **tip**. The including block is at least one slot after the tip, and in practice `D + g` slots after it (3.2). A ticket with `anchor_slot + 300 == tip_slot` passes the local check and cannot be accepted by any block. The margin the service leaves for delivery and inclusion is zero.

*The difficulty.* A ticket is searched against the target in its anchor block's own ledger state (`tickets.rs:193`) and validated against the target the including block's parent produced (`helpers.rs:135-137`). The target moves every block by `T · P / max(1, (P − F) · c + F · T)` (`ledger/src/mantle/pow/difficulty.rs:7`; `proof-of-work.md` › Reward Difficulty). On the shipped `T = 1, F = 9, P = 10` that is `10 / (c + 9)`: two claims in one block tighten the target by 9%, five by 29%, eleven by half. Tickets are uniform below the target they were found under, so after a tightening from `D` to `D'` a fraction `1 − D'/D` of the tickets found earlier are invalid, and the service never re-checks them.

*The rest.* The pool (3.3, item 2), a reorged anchor after publication (3.3, item 3), the epoch boundary (LB-004) and the fee (LB-005) are the other four ways a batched claim fails on chain.

*The batch.* Every failure above is per claim, and the ledger fails the transaction at the first one (`ledger/src/lib.rs:484-524`). The batch is every canonical ready ticket up to `MAX_CLAIMS_BY_PAYLOAD_SIZE` (`:1300-1307`, `:1364-1368`), so the claim inherits the deadline of its stalest ticket and the difficulty of its weakest one, and a single bad ticket forfeits all of them. The second model in 3.2 puts that at 7% to 28% of all tickets under the default tick, on top of the ones that die alone. The reorg comment at `:1027-1033` shows the authors knew one non-canonical anchor "would be rejected and poison the whole tx"; the same is true of the other five checks, and only that one is filtered.

*Afterwards.* The tickets are in `pending_to_claim`, the transaction has been evicted from the mempools, and nothing retries (#588 LB-001). The log shows `Claimed N PoW reward(s)` and, up to 300 slots later, `Pruned N expired PoW ticket(s)`.

**Exploit scenario**

No attacker is needed. A devnet miner with auto-claim on has three ready tickets at a tick: anchors 40, 180 and 299 slots behind the tip. All three pass the local check. The claim reaches a mempool 5 s later; the next block comes 25 s after that, at a gap of 329 slots from the oldest anchor. `accept_claim` returns `OutOfWindowSlot`, the leader removes the transaction, and all three rewards, including the one anchored 40 slots ago that had 260 slots of life left, stay in the pool.

An adversary that also mines can make it worse without touching the victim: including several of its own claims in one block tightens the target by 9% per extra claim on devnet, invalidating the corresponding share of every other miner's already-found tickets, and any batch that carries one of them.

**Recommendation**

- *Short term*: before building, run each ticket through the ledger's own `ClaimPoWRewardVerificationContext::verify` (or the same checks) against the tip's ledger state with `current_block_slot = tip_slot + margin`, where `margin >= RETRY_AFTER` of 3.4, and drop, not just skip, tickets that fail the window or the difficulty (they cannot recover) while keeping the ones that fail on the pool. Set `slot_window − margin` as the local window everywhere the service reads `slot_window`, including `claimable_rewards_info` and the generator's frontier.
- *Short term*: split the batch so that one rejected claim cannot take others with it: sort ready tickets by anchor age and put tickets whose remaining life is below a second, tighter threshold in their own transaction.
- *Long term*: the retry rule of 3.4, so that every rejected batch is rebuilt from fresh state instead of waiting out the window. Add the five rejection paths to `pow_mining.feature`.

**References**: `proof-of-work.md` › Acceptance Window, › Reward Difficulty; Mantle › CLAIM_POW_REWARD › Validation steps 1, 2 and 4; #588 report LB-001.

### LB-002 · The default auto-claim tick equals the acceptance window and the search mines blocks until they leave it, so most tickets expire before or during their one publication

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Economic / Incentive |
| Target | `services/pow/src/service.rs:214-216` (`default_auto_claim_tick`), `:640-697` (`auto_claim_tick_stream`); `services/pow/src/tickets.rs:304-326` (search lifetime) |
| Status | Open |

**Description**

The tick is 300 s of wall clock and the window is 300 slots of 1 s (`deployment-template.yaml:80`). A ticket found just after a tick is published 300 s later, when its anchor is at least 300 slots old, and cannot be claimed; a ticket found `x` slots after its anchor has, at the next tick, a uniformly distributed `0..300` s more of age. The first model in 3.2 shows 41% of tickets pruned by the service itself at the tick that would have published them, before any network delay, and a further 18% to 46% expiring between publication and inclusion depending on the Blend delay. Shortening the tick to 60 s or 30 s moves the settled fraction from 33% to 70% to 75% at a 30 s delay.

The other half is the generator. `poll_next` starts a search for every processed block younger than the window and stops it only when the block leaves the window (`tickets.rs:304-326`). A ticket found for a block 250 slots old has 50 slots to be published, delivered and included. In the model, a quarter of all tickets are found too late to be claimable under any tick, which is mining work the node pays for and the pool never pays out; and each of those tickets, if it reaches a batch, poisons it (LB-001).

The `Slots` variant of `AutoClaimTick` exists (`:208-211`) but the default is `Seconds(300)`, and the templates do not set `pow.auto_claim.tick`. The template comment on `slot_window` ("ten block-intervals of tolerance for a claim to reach a block") describes a budget that the tick alone can consume in full.

**Exploit scenario**

None needed. A miner on devnet with the defaults, finding one ticket per minute, settles about 13% of its tickets (3.2, second table, tick 300 s, rate 1/60 s). The remaining 87% of its electricity buys nothing, and the pool, which is sized to put 200 nodes over the minimum stake, distributes at a fraction of the intended rate.

**Recommendation**

- *Short term*: default the tick to `AutoClaimTick::Slots(slot_window / 10)` (30 slots, one expected block), and bound any configured tick to at most `slot_window / 5`. Reject a wall-clock tick longer than that at startup.
- *Short term*: stop searching a block once it is older than `slot_window − RETRY_AFTER − tick` (about 120 slots at the values of 3.4), so the generator only produces tickets that can still settle. Prefer the newest blocks when the pool is shared: the search buffer is per block, so today a block at age 290 gets the same share of the pool as the tip.
- *Long term*: pace claiming on the chain, not on the clock: publish when a ticket's remaining life crosses a threshold, or on every processed block while the ready set is non-empty, rather than on an interval.

**References**: `proof-of-work.md` › Acceptance Window ("a solution cannot be spent long after it was found"); `services/pow/src/tickets.rs:247-261`.

### LB-003 · Settlement is detected on any processed block, canonical or not, so a claim in a fork block retires tickets that are still claimable

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Consensus |
| Target | `services/pow/src/service.rs:1151-1180` (`retire_settled_claims`), `:1188-1221` (`prune_settled_pending`); `services/chain/chain-service/src/service/mod.rs:840-854` (event emission) |
| Status | Open |

**Description**

The chain service emits a `ProcessedBlockEvent` for every block it applies, whether or not that block is the new tip; the event carries `block_id` and, separately, `tip` (`service/mod.rs:840-854`, `lib.rs:294-304`). The PoW service reads the events of `block_id` (`:1159-1166`, `get_block_events`) and retires every pending ticket whose nullifier they carry (`:1202-1219`). It never compares `block_id` with `tip`, and never revisits a retirement.

So a claim included in a block on a losing branch retires its tickets the moment that block is processed. The claim is still in the mempool on the canonical side (`chain-network/src/lib.rs:1085-1093` keeps a non-tip block's transactions), so usually it lands there too and the retirement was merely early. It is wrong when the canonical branch never includes it: the anchor is on the losing branch (a claim anchored to a fork block, mined by a node that followed that fork), or the claim is evicted for any of LB-001's reasons before a canonical leader takes it. The tickets are then in neither set while still inside the window. The same happens on a reorg after inclusion: the tickets were retired at inclusion, and the reinserted transaction (`:1095-1104`) is invalid on the new branch if its anchor was reorged out.

With the retry rule of 3.4 this becomes load-bearing: a retirement on a fork block would cancel a retry the ticket still needs.

**Exploit scenario**

Two leaders win adjacent slots and the miner's node hears the losing block first. Its own claim, anchored to that block, is included by that leader (the leader's mempool has it from gossip), the event retires the tickets, the other branch wins, and the claim is `MissingBlock` there. The rewards are lost, and `Retired N settled PoW claim(s)` was logged.

**Recommendation**

- *Short term*: retire a ticket only from events of blocks on the canonical chain, at minimum `block_id == tip`, and re-check `pow_nullifiers` at the tip when a `LibUpdate` or a reorg is observed; the cheapest correct rule is to treat a ticket as settled only once its nullifier is in the tip ledger state's `pow.nullifiers()` (3.4 step 3), which is exactly the set the ledger keeps for the window.
- *Long term*: keep settled tickets in a third set until their claim is below the LIB, so a reorg can move them back to ready.

**References**: `services/chain/chain-service/src/lib.rs:286-291` (the event's own note that clients must handle it idempotently); `proof-of-work.md` › Acceptance Window (a nullifier is kept while its anchor is in the window, so the tip's set is the settled set).

### LB-004 · A claim built in one epoch is invalid in the next: the reward amount is frozen into the note ids the transfer spends

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Data Validation |
| Target | `services/pow/src/service.rs:1337`, `:1377-1380` (note ids from the tip's `epoch_reward`); `ledger/src/mantle/pow/mod.rs:133-145`, `:217-235` (`epoch_reward` recomputed at the boundary); `core/src/mantle/ledger.rs:512-524` (`Utxo::id` hashes the value) |
| Status | Open |

**Description**

The transfer that spends the reward notes references `Utxo::new(claim.op_id(), 0, Note::new(reward_value, pk)).id()` with `reward_value` read from the tip (`:1337`), and the note id commits to the value (`ledger.rs:519-520`). At an epoch boundary the ledger moves the epoch's refill into the pool and recomputes `epoch_reward = pool · ρ / (T · N_b)` (`pow/mod.rs:133-145`, `:217-235`); the pool has changed by every claim of the epoch and by the refill, so the new reward differs. `execute` mints the note with the ledger's `context.epoch_reward` (`pow.rs:337-344`), the transfer's input does not exist, and `Inputs::get_pk` fails with `InexistingNote` (`ledger.rs:390-397`). The whole batch is rejected and the tickets are pending until expiry. The claim's `epoch_nonce` would have been accepted (step 3 allows the previous nonce), so the specification lets a claim cross the boundary and the implementation's transaction shape does not.

The exposure per claim is `(D + g) / epoch_length`: with about 60 slots of delivery and inclusion, 0.2% on devnet (36,000-slot epochs at `k = 120`) and 0.01% on the spec schedule (648,000 slots). The cost when it hits is a whole batch.

**Exploit scenario**

None beyond timing. A miner's tick fires 20 s before an epoch boundary; the claim is delivered in 5 s and the next block is the epoch's first. Every ticket in the batch is lost.

**Recommendation**

- *Short term*: do not publish within `RETRY_AFTER` slots of an epoch boundary (the epoch length and the tip slot are both known to the service), or rebuild and republish once the boundary has passed. With the retry rule of 3.4 the rebuild happens on its own.
- *Long term*: none needed once claims are retried.

**References**: `proof-of-work.md` › Reward Pool; Mantle › CLAIM_POW_REWARD › Execution step 2, › TRANSFER.

### LB-005 · The claim fee is the exact minimum at the build tip, so any base-fee increase before inclusion invalidates it

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Economic / Incentive |
| Target | `services/pow/src/service.rs:1384-1385` (fee and change), `:1521-1534` (`estimate_reward_claim_fee`); `ledger/src/lib.rs:495-512` (fee check); `ledger/src/cryptarchia/mod.rs:459-479` (base fee update) |
| Status | Open |

**Description**

There is no per-transaction gas price in this ledger: the fee is `inputs − outputs`, and a transaction is valid only if that covers `execution_gas × base_fee + storage_gas × storage_price` at the state it is applied to (`lib.rs:495-512`). `estimate_reward_claim_fee` prices the batch at `minimum_gas_cost` with the tip's prices, and `change_outputs` returns every remaining token (`:1477-1496`), so the transaction pays exactly `base_fee[N+1]` per gas unit. `update_execution_market` raises the base fee by up to 12.5% per block whenever the smoothed usage is above target, rounding up (`cryptarchia/mod.rs:467-472`; `execution-market.md` › Base Fee Update Rule), and at the floor of 1 the first rise is to 2. Under a rising fee a claim built at tip `N` is valid in block `N+1` only. The builder's retry loop cannot make it applicable (`tx_selection.rs:137-158`), so it is removed, and the tickets are pending until expiry. The transaction builder already supports a priority reserve (`funding_delta_with_priority_fee`, `builder.rs:254-262`); the claim path does not use it.

On the shipped networks blocks are far below the 1.6 M-gas target, so the base fee stays at 1 and this does not trigger. It triggers exactly when the network is busy, which is when the spec's execution market says fees should rise.

**Exploit scenario**

A period of sustained load raises the base fee one step per block for twenty blocks. Every claim published during it, from every miner, is built at the previous block's price, arrives one or two blocks later, and is evicted. No miner settles a claim until the fee turns; the tickets published during the rise are lost.

**Recommendation**

- *Short term*: reserve a fee margin of at least two base-fee steps (`ceil(fee × 1.27)`) when the reward covers it, via the builder's `priority_fee_percent`; the surplus is a tip to the leader, not a loss. Rebuild instead of resending on retry (3.4).
- *Long term*: none once claims are retried with fresh prices.

**References**: `execution-market.md` › Block Builder Mechanism step 2, › Base Fee Update Rule; `overview-cryptoeconomics.md` › Fee Markets (upward rounding).

### LB-006 · Spec deviation: `CLAIM_POW_REWARD` costs 1 execution gas in the code and 590 in the specification

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Economic / Incentive |
| Target | `core/src/mantle/ops/pow.rs:280-282` (`GAS_COST = Gas::new(1)`) |
| Status | Open |

**Description**

`bedrock-v1.1-mantle-specification.md` › CLAIM_POW_REWARD › Execution gas says "Claim Operations have a fixed Execution Gas cost of `EXECUTION_CLAIM_POW_REWARD_GAS`", and › Gas Determination lists `EXECUTION_CLAIM_POW_REWARD_GAS | 590`. The code charges 1 (`pow.rs:280-282`). Every other operation's constant matches its table row (`transfer.rs:97` 590, `leader_claim.rs:204` 580, `sdp/declare.rs:169` 646, `sdp/active.rs:44` 590, channel ops 56 or 590). A claim's on-chain work is a Poseidon2 hash, a map lookup and a nullifier insert, so the 590 is the spec's figure for that; the analysis document this table comes from was not reviewed here. The code is the side that is wrong: it is a consensus constant, so every node must agree, and the figure in force is not the specified one.

The consequence for this issue is the fee: a batch of `c` claims pays `c + 590 · ceil(c/32)` execution gas instead of `590 · c + 590 · ceil(c/32)`, so the fee the claim must cover is about 30 times smaller than specified for a full group, and the `epoch_pow_reward > fee` requirement of `proof-of-work.md` › Parameters is checked against the wrong number. It also means a transaction of 247 claims and their 8 transfers (`MAX_OPS_PER_TX = 255`) costs under 5,000 execution gas against a 3.19 M-gas block limit, so claim operations are not bounded by the execution market at all, only by the storage fee and the reward-difficulty controller.

**Exploit scenario**

Not exploitable on its own: each claim still needs a valid puzzle solution. It changes the economics silently, and any node that adopts the specified constant forks from the rest.

**Recommendation**

- *Short term*: set `GAS_COST` to 590, or change the specification if 1 is intended, and add a test that pins every `GAS_COST` to the table.
- *Long term*: generate the gas table from one source shared by the spec and the code.

**References**: Mantle › CLAIM_POW_REWARD › Execution gas, › Gas Determination; `proof-of-work.md` › Parameters (reward must exceed the claim's fee).

## 5. Suggestions (non-security)

### S-001 · The pruning on ticket arrival uses the anchor's slot as "the current slot"

| | |
|---|---|
| Target | `services/pow/src/service.rs:596-601` |

The comment says "The new ticket's slot tracks the tip", but `winning_ticket.block_slot` is the anchor block's slot, which can be up to `slot_window` behind the tip since searches run for the whole window (`tickets.rs:304-326`). Pruning against it removes less than intended; harmless today, misleading once a margin is introduced (LB-001). Use the generator's cached tip slot, which `ProcessedBlockEvent` already carries.

### S-002 · Pending tickets and their publication are invisible

| | |
|---|---|
| Target | `services/pow/src/service.rs:1140` (`claimable_rewards_info`), `:1070-1075` |

`GET /pow/rewards/claimable` reports `ready_to_claim` only, and the hand-over log line says `Claimed`. With six ways for a published claim to fail silently, the operator needs the pending count, each pending ticket's remaining life, and the transaction hash it went out in. The #588 report's S-003 covers the log line; this adds the API.

### S-003 · The generator's frontier and the service's window are the same number

| | |
|---|---|
| Target | `services/pow/src/tickets.rs:304`, `services/pow/src/service.rs:1092-1096`, `nodes/node/binary/src/config/pow/mod.rs:15-33` |

Both read `slot_window`, the consensus constant. The node needs a second, smaller number, the local claimable window, derived from it (`slot_window − RETRY_AFTER`), and both call sites should read that one. Deriving it in `into_pow_service_settings` keeps the consensus value single-sourced.

### S-004 · `proof-of-work.md`: say what a claimant may assume about inclusion delay

| | |
|---|---|
| Target | `proof-of-work.md` › Acceptance Window |

The window is stated as a validation rule. Nothing tells an implementer how much of it to reserve for delivery and inclusion, or that the difficulty a claim is validated against is the one at inclusion, not at the anchor (that is in › Reward Difficulty, but the claim-side consequence, that found tickets go stale, is not drawn). One sentence on each would have avoided LB-001 and LB-002. To be raised in logos-lips; not done from this audit.

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
