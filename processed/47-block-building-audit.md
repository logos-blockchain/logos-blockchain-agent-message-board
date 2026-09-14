# Audit Report — Block building: valid-only txs, ordering, limits, panics

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/47`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `b8c3c54ff6e808f247521befd7d103d9daa361aa` — component(s): `services/chain/chain-leader`, `ledger` (block assembly path)
Specs: `https://github.com/logos-co/logos-lips` @ `4b9d1ca1794f1e1b47aa40582f7793e2f84ed999` — read: `cryptarchia-proof-of-leadership.md`, `bedrock-anonymous-leaders-reward.md` (in full); `bedrock-v1.1-block-construction.md` (Proposal Construction, Block Proposal Validation, Block Execution); `cryptarchia-v1-protocol.md` (Constants, Block Header Validation, Uncle References); `overview-cryptoeconomics.md`, `bedrock-architecture-overview.md` (core)
Date: 2026-09-09 — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: Block assembly is broadly correct on size, count and validity, but it never enforces the per-block **execution-gas** limit that block validation enforces, so a leader can assemble a block that fails its own validation and silently forfeits its slot.
- Findings: 0 critical · 0 high · 1 medium · 1 low · 0 informational
- Key themes: proposer builds a block the network (and the proposer itself) will reject; quadratic re-verification of ZK proofs during assembly.
- Must-fix before launch: LB-001 (leaders lose slots under a cheap mempool flood).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/chain/chain-leader/src/lib.rs` | `propose_block`, the assembly retry loop, `txs_for_block`, `apply_and_publish_block_proposal` |
| `services/chain/chain-leader/src/leadership.rs` | winning-slot / proof build (read for context) |
| `ledger/src/lib.rs` | `try_apply_contents`, `EXECUTION_GAS_LIMIT` enforcement |
| `core/src/block/mod.rs` | `Block::create`, size/count bounds |
| `services/chain/chain-service/src/lib.rs` | `try_apply_block_with_state_retention` (self-apply of the proposed block) |
| `services/tx-service/src/backend/pool.rs`, `tx/service.rs` | mempool admission, `view`, eviction |

**Out of scope**

The ZK circuits and prover (`rust-rapidsnark`, `ark-groth16`, `jellyfish`/Poseidon2) are assumed correct. NTP/time-source robustness, KMS secret handling, and equivocation across restarts (parent issue #5 questions) were not audited here; this report covers only the block-building sub-issue. `libp2p`, `rocksdb`, `overwatch` assumed correct.

**Assumptions**

Repo-level facts from issue #19 hold at this commit: `overflow-checks` is off in release, so `+ - * <<` wrap silently; the panic/overflow clippy lints are allowed. Spec at the pinned `logos-lips` commit is the reference.

## 3. Method

- Manual review of the in-scope paths, working through issue `#47` under parent `#5`.
- Spec conformance against `bedrock-v1.1-block-construction.md` (Proposal Construction / Validation / Execution) and `cryptarchia-v1-protocol.md` (Constants, Block Header Validation) — in particular block-body limits `MAX_BLOCK_SIZE` (2 MiB), `MAX_BLOCK_TXS` (1024), and the execution-gas limit `limit_Ex = 3,193,460` from `overview-cryptoeconomics.md` / `execution-market.md`.
- Automated tooling: none run. Findings are from source reading against the spec.
- Dynamic testing: none.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Block assembly omits the per-block execution-gas limit; leader forfeits its slot | Denial of Service | Medium | Medium | Open |
| LB-002 | Assembly re-verifies each transaction's ZK proofs once per retry round | Denial of Service | Low | Medium | Open |

### LB-001 · `Block assembly omits the per-block execution-gas limit, so the leader builds a block that fails its own validation`

| | |
|---|---|
| Severity | Medium |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | `services/chain/chain-leader/src/lib.rs:673-743` (`propose_block`), `services/chain/chain-leader/src/lib.rs:881-911` (`txs_for_block`); limit enforced at `ledger/src/lib.rs:527-533` |
| Status | Open |

**Description**

The spec caps a block at `limit_Ex = 3,193,460` execution gas (`overview-cryptoeconomics.md`, Execution Fee Market; enforced in the node at `ledger/src/lib.rs:527`, constant at `ledger/src/lib.rs:99`). Block validation accumulates execution gas across *all* transactions of the block and rejects the block with `TooMuchExecutionGas` once the running total exceeds the limit.

Block *assembly* does not apply this check. In `propose_block` the candidate transactions are each applied individually:

```rust
// services/chain/chain-leader/src/lib.rs
for tx in pending {
    match ledger_state
        .clone()
        .try_apply_contents::<_, HeaderId, MainnetGasProfile>(
            ledger_config,
            iter::once(tx.clone()),   // one tx per call
        ) { ... valid_txs.push(tx); ... }
}
```

Each call to `try_apply_contents` starts its `total_block_execution_gas` at zero (`ledger/src/lib.rs:474`) and only sums the single transaction it is given, so the per-block limit at `ledger/src/lib.rs:527` is evaluated against one transaction at a time and never against the accumulated block. The final selection step, `txs_for_block` (`lib.rs:881-911`), truncates the block only by **storage size** (`MAX_BLOCK_TRANSACTIONS_SIZE`, 2 MiB) and by **count** (`BlockTransactions::MAX`, 1024). Nothing bounds the *cumulative execution gas* of the selected set.

The proposer therefore assembles a block whose transactions are each individually valid but whose summed execution gas exceeds `limit_Ex`. `Block::create` (`core/src/block/mod.rs:171`) does not check gas either. The block is then self-applied before publishing:

```rust
// apply_and_publish_block_proposal
if let Err(e) = chain_network_api.apply_block_and_reconcile_mempool(block.clone()).await {
    error!(... "Failed to apply our own proposed block ...");
    return;               // <-- never published
}
```

`apply_block` runs the block through `try_apply_block_with_state_retention` → `prepare_update` → `try_apply_contents` over the **whole** transaction list, where the running gas total now crosses `EXECUTION_GAS_LIMIT` and returns `TooMuchExecutionGas`. The leader logs the error and returns without broadcasting. The winning proof of leadership for that slot is wasted.

Crucially, the offending transactions are **not** evicted: `propose_block` only removes transactions that never became applicable (`invalid_tx_hashes`, `lib.rs:729`). These transactions *are* applicable individually, so they stay in the mempool for the full 24 h TTL (`DEFAULT_TX_TTL`, `services/tx-service/src/backend/pool.rs`), and every subsequent leader that draws them repeats the failure.

**Exploit scenario**

An unprivileged peer floods the gossip mempool with valid, well-funded Mantle transactions that are individually cheap in execution gas but collectively heavy. A single transaction may carry up to `MAX_OPS_PER_TX = 255` operations (`core/src/mantle/transactions/mod.rs:39`); a Transfer op costs 590 execution gas (`core/src/mantle/ops/transfer.rs:98`), so one transaction reaches ~150,450 gas and stays far under the 3,193,460 per-*call* limit seen during assembly. About 22 such transactions sum past `limit_Ex`. Mempool admission runs no gas or balance check — only a size check (`validate_item_for_mempool`, `services/tx-service/src/tx/service.rs`) — so the transactions are admitted and gossiped network-wide. Because every block that includes them is rejected at self-apply, they are never mined and never pay, so the attacker re-uses the same funded notes indefinitely. Every honest leader whose selection prefix includes enough of them forfeits its slot; sustained flooding depresses the block rate across the network. This is a liveness degradation, not a full halt: leaders whose selected prefix happens to stay under the limit still produce.

**Recommendation**

- *Short term*: enforce `EXECUTION_GAS_LIMIT` during selection. Thread the running execution-gas total through the assembly loop (or through `txs_for_block`) and stop including transactions once the next one would cross `limit_Ex`, exactly as `bytes(transactions) <= MAX_BLOCK_SIZE` is already handled. Apply candidate transactions against a single advancing state and check the cumulative gas, rather than calling `try_apply_contents` with `iter::once`.
- *Long term*: make the proposer construct the block by the same single-pass `try_apply_contents` the validator uses, so "assembled" and "valid" cannot diverge on any block-level limit. A proposer-side assertion that its own block passes full validation before publishing would catch this class (gas, and any future per-block bound) at the source.

**References**: `overview-cryptoeconomics.md` §Execution Fee Market (`limit_Ex = 3,193,460`); `bedrock-v1.1-block-construction.md` §Block Proposal Validation (rule that transactions validate against an advancing state); node enforcement at `ledger/src/lib.rs:527`.

### LB-002 · `Assembly re-verifies each transaction's ZK proofs once per retry round`

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | `services/chain/chain-leader/src/lib.rs:681-725` (retry loop), `lib.rs:696` (`deferred_zkps.verify()`) |
| Status | Open |

**Description**

The assembly loop retries the whole pending set each round while any transaction applied:

```rust
let mut applied_any = true;
while applied_any {
    applied_any = false;
    for tx in pending {
        match ledger_state.clone().try_apply_contents(..., iter::once(tx.clone())) {
            Ok((new_state, _events, deferred_zkps)) => match deferred_zkps.verify() { ... }
            ...
        }
    }
}
```

Two costs compound here. First, `deferred_zkps.verify()` (`lib.rs:696`) is called per transaction, defeating the batch verification the ledger is built around (`core/src/mantle/batch.rs`, `verify_batch_proofs`): each transaction's Groth16 proofs are verified in isolation rather than in one batched pairing check. Second, a transaction that fails to apply this round is pushed back to `still_pending` and its proofs are verified again next round. A dependency chain of length N (transaction *i* spends an output of *i-1*) applies exactly one transaction per round, so the loop runs N rounds and performs O(N²) `try_apply_contents` calls, each including a fresh ZK verification of the transaction it touches. With `MAX_BLOCK_TXS = 1024`, a crafted chain drives on the order of hundreds of thousands of full apply-and-verify operations.

This work runs on the leader service's main task: `propose_block` is awaited directly in the `tokio::select!` loop (`lib.rs:507`), not on `spawn_blocking` (the `// TODO: spawn as a separate task?` at `lib.rs:506` acknowledges this). It does not block the separate chain-service that validates inbound blocks, but it stalls the leader's own event loop (winning-slot handling, claim requests) and burns CPU during the 1-second slot the proposer is trying to hit.

**Exploit scenario**

A peer seeds the mempool with a long chain of interdependent valid transactions (each spending the prior one's output). When this node wins a slot, assembly enters the O(N²) regime and the ZK proofs on the chain are re-verified many times over, delaying or missing the block. Impact is limited to the proposer's own slot and bounded by mempool size, hence Low.

**Recommendation**

- *Short term*: collect the candidate set with a single advancing state and one batched `verify()` at the end, instead of per-transaction `verify()` inside the retry loop. Cap the number of retry rounds.
- *Long term*: topologically order candidates once (by input/output dependency) so a single pass suffices, removing the retry loop entirely.

**References**: `bedrock-v1.1-block-construction.md` §Batch verification of ZK proofs; `core/src/mantle/batch.rs`.

## 5. Suggestions (non-security)

- **S-001 · Ordering policy is undocumented.** Selection order is the mempool's `IndexSet` iteration order (`services/tx-service/src/backend/pool.rs`, `view`), i.e. insertion order, with no fee-priority or documented MEV policy. This is deterministic per node but arbitrary across nodes and is not consensus-relevant (validators only re-derive `body_root`). Worth stating the intended ordering policy explicitly, since today a leader cannot prioritise by tip even though the execution market defines a priority fee.
- **S-002 · Assembly ledger state diverges from canonical.** The `ledger_state` advanced in the assembly loop applies transactions one at a time, so `compute_block_rewards` runs once per transaction with per-transaction fee totals rather than once with block totals. This state is used only for the debug log `log_sdp_activity_selected_for_proposal` (`lib.rs:745`), so it is harmless today, but it is a latent trap if that state is ever read for anything load-bearing. Building the proposal against the same single-pass application the validator uses (see LB-001 long-term) removes the divergence.

---

## Appendix A — Definitions

Severity, difficulty and category ratings use the definitions in `docs/REPORT_TEMPLATE.md` Appendix A. No new scales were introduced.
