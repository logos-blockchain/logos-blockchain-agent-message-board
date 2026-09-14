# Audit Report — Mempool admission and SDPActive verification order: Groth16 PoQ check before signature and ledger validation

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/98`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a77c248f350014bfab7990233b35bcf61b450d05` — component(s): `core/src/mantle/ops/sdp`, `ledger/src/lib.rs`, `ledger/src/mantle/sdp`, `services/tx-service`, `services/chain/chain-leader`, `services/chain/chain-service`, `blend/message/src/reward`, `blend/proofs`, `zk/proofs/{poq,zksign}`
Specs: `https://github.com/logos-co/logos-lips` @ `d6e824293579e5f53b8d278d0f710467fcae5fb8` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `bedrock-service-declaration-protocol.md`, `bedrock-service-reward-distribution.md`, `bedrock-anonymous-leaders-reward.md` (all in full); `bedrock-v1.1-mantle-specification.md` §Validation, §SDP_ACTIVE; `bedrock-v1.1-block-construction.md` §Proposal Construction, §Block Proposal Validation, §Batch verification of ZK proofs; `blend-protocol.md` §Active Message; `analysis-gas-cost-determination.md` §Execution Gas, §SDP Activation, §Gas determination from measures (by section)
Date: `2026-09-09` — author: `claude-fable-5-1` — status: `final`

---

## 1. Summary

- Overall assessment: the four items of #98 are confirmed. An `SDPActive` operation runs one full, unbatched Groth16 proof-of-quota (PoQ) verification during *execution*, after only three cheap checks (declaration exists, not withdrawn, nonce increasing) and before its own ZkSignature is checked (deferred to the block batch), before the "one active message per provider per epoch" check, and before the fee balance check. The transaction mempool admits anything under 2 MiB without touching the ledger and has no count bound, no per-sender or per-op limit, and no eviction except a 24 h TTL and "the builder tried it and it failed". The block builder pulls the whole pool, applies every transaction against the ledger on every proposal, and retries every failure in every round. Together these let an unauthenticated peer, with no stake, no key and no fee, make every leader on the network spend one PoQ verification per garbage transaction per proposal, and the `SDP_ACTIVE` gas (590) does not fund that verification at all, so a block that is within the execution gas limit can still require three to seven times the CPU budget the fee market is designed to enforce (seven on the spec's own cost model, three measured here against the batched signature cost).
- Findings: 0 critical · 1 high · 2 medium · 1 low · 0 informational
- Key themes: "expensive check before cheap checks", "execution gas does not price the service-specific activity check", "mempool is a byte-size filter, not an admission policy", "builder cost is proportional to pool size, per proposal, per round"
- Must-fix before launch: LB-001 (builder DoS from unauthenticated mempool input) and LB-002 (PoQ verification unpriced by gas and unbatched). LB-003 and LB-004 are cheap to fix alongside.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `core/src/mantle/ops/sdp/active.rs` L44-L46, L63-L102, L105-L133 | gas constant, `verify` (what is checked before execution, what is deferred), `execute` |
| `core/src/mantle/ops/signed_op.rs` L335-L349 | verification context handed to `SDPActive` (`declarations`, `epoch`) |
| `core/src/mantle/batch.rs` L42-L57 | deferred ZkSignature batch |
| `core/src/mantle/transactions/mod.rs` L39; `core/src/block/mod.rs` L29-L32 | `MAX_OPS_PER_TX`, `MAX_BLOCK_TRANSACTIONS`, `MAX_BLOCK_TRANSACTIONS_SIZE` |
| `core/src/sdp/mod.rs` L564-L600; `core/src/sdp/blend.rs` L15-L62 | activity metadata encoding and decode-time checks |
| `ledger/src/lib.rs` L99, L466-L541, L731-L830, L910-L953 | gas limit, `try_apply_contents` (balance check after execution), `try_apply_op`, interleaved verify/execute loop |
| `ledger/src/mantle/sdp/mod.rs` L506-L545, L663-L680 | `apply_active_msg`, `get_service` |
| `ledger/src/mantle/sdp/rewards/blend/mod.rs` L63-L117 | `update_active` (order of `verify_proof` and duplicate `insert`) |
| `ledger/src/mantle/sdp/rewards/blend/target_epoch.rs` L79-L125, L152-L176 | `verify_proof` (order of checks around the pairing), `TargetEpochTracker::insert` |
| `ledger/src/mantle/sdp/rewards/blend/current_epoch.rs` L93-L185 | when a target epoch (and so a verifier) exists |
| `blend/message/src/reward/activity.rs` L41-L65; `blend/message/src/crypto/proofs.rs` L81-L112 | PoQ then PoSel order, `RealProofsVerifier` |
| `blend/proofs/src/quota/mod.rs` L76-L86; `blend/proofs/src/selection/mod.rs` L53-L105 | PoQ verify (Groth16), PoSel verify (two hashes and a modulo) |
| `zk/proofs/poq/src/lib.rs` L114-L135; `zk/proofs/zksign/src/lib.rs` L114-L145; `zk/groth16/src/verifier.rs` L10-L60 | single vs batched Groth16 verification |
| `services/tx-service/src/tx/service.rs` L393-L440, L536-L580 | HTTP add path, network add path, `validate_item_for_mempool` |
| `services/tx-service/src/backend/pool.rs` L26-L52, L140-L224; `backend/evictor.rs`; `backend/policy/ttl.rs` | pool structure, add, view, remove, TTL eviction |
| `services/tx-service/src/storage/adapters/rocksdb.rs` L49-L80 | where pool items live |
| `services/blend/src/core/dispatcher/libp2p.rs` L153-L190; `services/api/src/http/mempool.rs` L60-L85 | the other two ingress paths into the pool |
| `services/chain/chain-leader/src/lib.rs` L455-L530, L630-L760, L881-L910 | leader loop, `propose_block` (pool scan, retry rounds, per-tx deferred verification), `txs_for_block` |
| `services/chain/chain-service/src/lib.rs` L426-L500 | validator path: `prepare_update` then `verify_batch_proofs` |
| `libp2p/src/behaviour/mod.rs` L22, L77 | gossipsub message size limit |
| `services/sdp/src/mempool.rs` | the "SDP mempool" of parent #8 question 4 (it is an adapter to the transaction pool) |
| `nodes/node/binary/src/config/deployment/settings.yaml` L1-L30, L112-L125 | deployed blend, epoch, slot and TTL parameters |

**Out of scope**
The PoQ, PoSel and ZkSignature circuits and their keys (assumed sound), `ark-groth16`/`ark-bn254`, `rust-rapidsnark`, `libp2p` gossipsub internals, RocksDB, the SDP declare/withdraw paths (#103 covers declare-time validation order), blend reward correctness (#54, #76, #89), the HTTP API surface (#66), and the mempool's general admission policy beyond what #98 asks (#55 is the sub-issue for that; findings here that touch it are cross-referenced rather than developed).

**Assumptions**
Deployed parameters from `settings.yaml`: `slot_duration = 1 s`, `security_param = 30`, `slot_activation_coeff = 1/20`, epoch split 3/3/4, so one epoch is 6 000 slots (≈ 1 h 40 min) and a block is expected every 20 s on average; `minimum_network_size = 2`; `tx_ttl = 24 h`. The gas analysis' reference machine and cycle counts (`analysis-gas-cost-determination.md` §Gas determination from measures) are taken as the protocol's own cost model. The attacker controls one ordinary peer with no stake and no funds.

## 3. Method

- Manual review of the in-scope paths, working through sub-issue #98 under parent #8 (question 4 of the parent, "is the SDP mempool bounded, what evicts stale declarations", is answered under LB-004 and in §4 "Checked and ruled out").
- Spec conformance against `bedrock-v1.1-mantle-specification.md` §SDP_ACTIVE (validation order: declaration, nonce, signature, then service logic at execution), `bedrock-v1.1-block-construction.md` §Proposal Construction step 2 and §Block Proposal Validation step 6 (batching covers proof checks only), `blend-protocol.md` §Active Message (one message per node per attested epoch, epoch `e` reported in `e+1`), and `analysis-gas-cost-determination.md` §SDP Activation (what the 590 gas is meant to pay for).
- Automated tooling: none.
- Dynamic testing: one timing harness, built in release mode (`lto = false`, `codegen-units = 16`, otherwise the workspace profile) from a worktree at `a77c248f` and run on the audit machine (Raspberry Pi 5, Cortex-A76 at 2.4 GHz, `rustc 1.98.1`). It proves one core-node PoQ and one ZkSignature with the repository's own benchmark fixtures, then times: a PoQ verification of a valid proof, of a well-formed proof for other public inputs (the rejection path an attacker triggers), and of garbage bytes; PoQ `batch_verify` for 1/16/256 proofs; a single ZkSignature verification; and ZkSignature `batch_verify` for 1/16/256/1024 proofs. Results are in LB-002. Source (the `common` module is a copy of `zk/proofs/poq/benches/common/mod.rs`):

```rust
// audit-timing/src/main.rs (added as a workspace member; deps: lb-groth16, lb-pol, lb-poq,
// lb-poseidon2, lb-utils, lb-zksign, num-bigint, all `workspace = true`)
mod common;
use std::time::{Duration, Instant};
use lb_groth16::Fr;
use lb_poq::{KeyIndex, PoQProof, PoQVerifierInput, batch_verify as poq_batch_verify,
             prove as poq_prove, verify as poq_verify};
use lb_poseidon2::{Digest as _, Poseidon2Bn254Hasher};
use lb_zksign::{ZkSignPrivateKeysData, ZkSignProof, ZkSignVerifierInputs, ZkSignWitnessInputs,
                batch_verify as zk_batch_verify, prove as zk_prove, verify as zk_verify};
use num_bigint::BigUint;

fn time<F: FnMut()>(label: &str, iters: usize, mut f: F) -> Duration {
    for _ in 0..3 { f(); }
    let start = Instant::now();
    for _ in 0..iters { f(); }
    let per = start.elapsed() / iters as u32;
    println!("{label:60} {:>10.3} ms  (n={iters})", per.as_secs_f64() * 1e3);
    per
}

fn main() {
    let (poq0, in0) = poq_prove(common::core_node_inputs(KeyIndex::try_new(0).unwrap())).unwrap();
    let (poq1, _) = poq_prove(common::core_node_inputs(KeyIndex::try_new(1).unwrap())).unwrap();
    let sks: [Fr; 32] = core::array::from_fn(|k| BigUint::from(k as u64 + 1).into());
    let msg = Poseidon2Bn254Hasher::digest(&[BigUint::from(7u64).into()]);
    let (zsig, zin) = zk_prove(ZkSignWitnessInputs::from_witness_data_and_message_hash(
        ZkSignPrivateKeysData::from(sks), msg)).unwrap();
    let msg2 = Poseidon2Bn254Hasher::digest(&[BigUint::from(8u64).into()]);
    let (zsig2, _) = zk_prove(ZkSignWitnessInputs::from_witness_data_and_message_hash(
        ZkSignPrivateKeysData::from(sks), msg2)).unwrap();

    let n = 50;
    time("PoQ verify, valid proof", n, || assert!(poq_verify(&poq0, in0.clone()).unwrap()));
    time("PoQ verify, well-formed proof for other inputs (rejected)", n,
         || assert!(!poq_verify(&poq1, in0.clone()).unwrap()));
    let garbage = PoQProof::from_bytes(&[0x11u8; 128]);
    time("PoQ verify, garbage bytes (expansion error path)", n, || { let _ = poq_verify(&garbage, in0.clone()); });
    for k in [1usize, 16, 256] {
        let batch: Vec<(PoQProof, PoQVerifierInput)> = (0..k).map(|_| (poq0, in0.clone())).collect();
        time(&format!("PoQ batch_verify, {k} valid proofs (total)"), if k >= 256 { 5 } else { 20 },
             || assert!(poq_batch_verify(&batch).unwrap()));
    }
    time("ZkSign verify, valid, single", n, || assert!(zk_verify(&zsig, &zin).unwrap()));
    time("ZkSign verify, well-formed wrong proof, single (rejected)", n,
         || assert!(!zk_verify(&zsig2, &zin).unwrap()));
    for k in [1usize, 16, 256, 1024] {
        let batch: Vec<(ZkSignProof, ZkSignVerifierInputs)> = (0..k).map(|_| (zsig, zin.clone())).collect();
        time(&format!("ZkSign batch_verify, {k} valid proofs (total)"), if k >= 256 { 3 } else { 20 },
             || assert!(zk_batch_verify(&batch).unwrap()));
    }
}
```

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Unauthenticated mempool input makes every block builder run one Groth16 PoQ verification per garbage `SDPActive` transaction, per proposal, per retry round | Denial of Service | High | Low | Open |
| LB-002 | `SDP_ACTIVE` execution gas funds the ZkSignature only; the unbatched PoQ verification it triggers is unpriced, so a gas-valid block can need 3–7× the CPU budget | Economic / Incentive | Medium | Medium | Open |
| LB-003 | In `apply_active_msg` the pairing check runs before the duplicate-per-provider, proof-of-selection and activity-threshold checks, all of which are hash-cheap | Denial of Service | Low | Low | Open |
| LB-004 | The transaction pool, which is also the SDP mempool, has no count or byte bound, no per-sender or per-op limit, and no admission validation; stale `SDPActive` ops leave only by TTL or by failing at a leader | Denial of Service | Medium | Low | Open |

### What runs, in what order

For one `SDPActive` op inside a transaction, on both the builder and the validator:

1. `ledger/src/lib.rs` L910-L953 (`try_apply_tx`): for each op, `verified_operations.next(&helper)` (L932) then `try_apply_op` (L943). Verification and execution are interleaved per op; the fee balance check is in the caller, after the whole transaction has executed (`try_apply_contents` L507).
2. `core/src/mantle/ops/sdp/active.rs` L63-L102 (`verify`): declaration exists (L70), not withdrawn (L77-L84), nonce increasing (L87-L92), then the ZkSignature is **deferred** (L95-L101) into the block's batch.
3. `active.rs` L110-L132 (`execute`), reached through `ledger/src/mantle/sdp/mod.rs` L506-L545 (`apply_active_msg`): sets `active` and `nonce`, then `update_rewards` (L538) → `rewards/blend/mod.rs` L95-L100 `verify_proof` → `target_epoch.rs` L79-L125: epoch equals target epoch (L86), provider known (L98-L101), **`verify_and_build`** (L103-L109) which calls `RealProofsVerifier::verify_proof_of_quota` (`proofs.rs` L81-L105) → `ProofOfQuota::verify` (`quota/mod.rs` L76-L86) → `lb_poq::verify` (`poq/src/lib.rs` L114-L119) → `ark_groth16::verify_proof` (`verifier.rs` L16), a single three-pairing check; then proof of selection (`activity.rs` L52-L59; two Blake2b hashes and a modulo), then the Hamming-distance threshold (`target_epoch.rs` L117-L122), then back in `mod.rs` L102-L107 the duplicate check `TargetEpochTracker::insert` (`target_epoch.rs` L157-L163).
4. Only after every transaction of the block has been applied does the validator check the batch of ZkSignatures (`chain-service/src/lib.rs` L464-L479; `batch.rs` L42-L57). The builder checks the deferred batch per transaction (`chain-leader/src/lib.rs` L696), i.e. a batch of one.

So a transaction carrying an `SDPActive` op with a real `declaration_id`, any nonce above the declaration's, `epoch = target epoch`, a proof of quota made of three well-formed curve points, and 128 arbitrary bytes as signature, costs whoever applies it one full Groth16 verification before it fails (at `verify_and_build`), and it needs no funds because the balance is checked after execution.

### LB-001 · Unauthenticated mempool input makes every block builder run one Groth16 PoQ verification per garbage `SDPActive` transaction, per proposal, per retry round

| | |
|---|---|
| Severity | High |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `services/chain/chain-leader/src/lib.rs:L646-L722` (`propose_block`), `L455-L520` (leader loop); `services/tx-service/src/tx/service.rs:L536-L546` (`validate_item_for_mempool`); `ledger/src/mantle/sdp/rewards/blend/target_epoch.rs:L103-L109` |
| Status | Open |

**Description**
The pool admits any item whose encoded size is at most `MAX_BLOCK_TRANSACTIONS_SIZE` (2 MiB), from gossip (`service.rs` L548-L571), from the HTTP API (L393-L440, which also re-gossips duplicates) and from the blend dispatcher (`dispatcher/libp2p.rs` L153-L190); `validate_item_for_mempool` (L536-L546) is the only check and it never touches the ledger. Gossipsub accepts messages up to 16 MiB (`libp2p/src/behaviour/mod.rs` L22, L77), so the 2 MiB per item is reachable from the network.

On every winning slot the leader fetches the whole pool (`propose_block` L646-L647, `pool.rs` L176-L182 returns every pending item), collects it into a `Vec` (L673), and applies each transaction to a clone of the tip ledger state (L682-L722). A transaction that fails goes to `still_pending` (L709, L719) and is retried in the next round; rounds continue while any round adds at least one transaction (L682). Only transactions that never applied are removed from the pool (L727-L737), after the cost has been paid, and nothing stops the same peer from re-sending fresh ones (a different nonce or different proof bytes gives a new hash). The proposal is built inline in the leader's `select!` loop (L455, L506-L507, with the comment `TODO: spawn as a separate task?`), so while it runs the leader processes no slot ticks.

Per the "What runs" section, a garbage `SDPActive` transaction reaches `verify_and_build` and costs one Groth16 verification there. The attacker needs only public data: a `declaration_id` that is in the target epoch's provider set (any active blend node, readable from the chain), and the current target epoch number. The transaction needs no `Transfer`, no note and no valid signature; a single-op transaction is 404 bytes (`1 + (1 + 32 + 8 + 4 + 230) + 128`).

Cost per garbage transaction per proposal per round on this audit machine: 6.17 ms (LB-002 table); on the gas analysis' reference machine, about 4.1 M cycles. With `M` garbage transactions in the pool and `R` rounds, each proposal costs `M · R` verifications on top of the honest work. `R` is at least 1, and an attacker who also holds a few funded notes can make it large: a chain of `N` transactions each spending the previous one's output, gossiped in reverse order, applies exactly one transaction per round (the pool preserves insertion order, `pool.rs` L69, and `still_pending` preserves it too), giving `R = N + 1`.

**Exploit scenario**
A peer with no stake connects to the mempool topic and publishes 100 000 garbage `SDPActive` transactions (≈ 40 MB, under a minute on a home connection). Every node stores them (pool items live in RocksDB via `rocksdb.rs` L49-L63, keys in memory) for up to 24 h. From then on, every leader that wins a slot spends `100 000 × 6.17 ms ≈ 617 s` (this machine) or about `100 000 × 4.1 M cycles ≈ 130 s` on a 3.2 GHz core before it can publish, while its own slot timer is blocked. Expected block interval is 20 s, so the chain stops producing timely blocks network-wide until the TTL expires or every leader has tried, failed, and evicted the batch — at which point the peer re-sends. With the dependency-chain variant the same 100 000 items are verified `N + 1` times per proposal. No fee is ever paid: the balance check (`ledger/src/lib.rs` L507) is after execution, and the garbage never gets that far.

**Recommendation**
- *Short term*: (1) In `propose_block`, verify the deferred proofs and execute the service-specific activity logic only for transactions that have passed every cheap ledger check and the fee balance check, and stop retrying a transaction whose failure was not a missing dependency (a `Sdp`/`InvalidProof`/`DuplicateActiveMessage`/`InsufficientBalance` error is final for this block; only "input not found" style errors deserve a retry). (2) Move the proposal off the leader's select loop, with a deadline tied to the slot. (3) Cap the pool (count and bytes) and evict oldest-first when full (see LB-004).
- *Long term*: validate at admission against the tip ledger, before gossiping (validate-before-forward): decode, check the fee note exists and covers `total_gas_cost` at current prices, and for `SDPActive` check declaration exists, nonce, `epoch == target epoch`, provider in the target set, and that no accepted or pending message already exists for that `(declaration_id, target epoch)`, keeping at most one pending `SDPActive` per declaration (replace on higher nonce). Verify the ZkSignature once at admission (one Groth16, LB-002 table) so that a transaction in the pool is known to be authorised by the declaration's `zk_id`; garbage then costs the attacker a signature it cannot produce. Parent #8 question 4 and #55 ask for the same policy.

**References**: `bedrock-v1.1-block-construction.md` §Proposal Construction step 2 ("choose up to `MAX_BLOCK_TXS` **valid** transactions from the local mempool", which the code implements by trial execution of the whole pool); report #54 LB-002 (this finding is that item, re-rated after reading the builder).

### LB-002 · `SDP_ACTIVE` execution gas funds the ZkSignature only; the unbatched PoQ verification it triggers is unpriced, so a gas-valid block can need 3–7× the CPU budget

| | |
|---|---|
| Severity | Medium |
| Difficulty | Medium |
| Category | Economic / Incentive |
| Target | `core/src/mantle/ops/sdp/active.rs:L44-L46` (`GAS_COST = 590`); `ledger/src/mantle/sdp/rewards/blend/target_epoch.rs:L103-L109`; `zk/proofs/poq/src/lib.rs:L114-L119` (single `verify`, while `batch_verify` L121-L135 exists and is unused by the ledger) |
| Status | Open |

**Description**
`analysis-gas-cost-determination.md` §SDP Activation derives the 590 gas as "Verification of the ZK signature: 590,000 cycles" (the marginal cost of one more proof in a batch) and states "Evaluation of the activity depends on the service and is neglected here". The Blend activity evaluation is a full Groth16 verification (three pairings plus a 12-term MSM), run one proof at a time, on every node, at execution (`target_epoch.rs` L103-L109 → `lb_poq::verify`). The same document measures one *single* Groth16 verification of the comparable ZkSignature circuit at 4 126 177 cycles (§ZkSignature, batch size 1). The op is therefore priced at about one seventh of what it costs on that model, and the part left out is the unbatched part. Measured here the unbatched PoQ costs 3.3× the batched marginal ZkSignature (6.18 ms vs 1.88 ms), the gap the 590 gas was calibrated on.

Measured on the audit machine (release build, see §3):

| Operation | ms per call |
|---|---|
| PoQ `verify`, valid proof | 6.18 |
| PoQ `verify`, well-formed proof for other public inputs (rejected) | 6.17 |
| PoQ `verify`, garbage bytes (rejected at decompression) | 0.62 |
| PoQ `batch_verify`, per proof at batch 16 / 256 | 2.27 / 1.90 |
| ZkSignature `verify`, single (what the builder pays per transaction at L696) | 14.7 |
| ZkSignature `batch_verify`, per proof at batch 256 / 1024 | 1.91 / 1.88 |

Capacity of one block in `SDPActive` ops, from the code's limits (`MAX_OPS_PER_TX = 255`, `EXECUTION_GAS_LIMIT = 3 193 460`, `MAX_BLOCK_TRANSACTIONS_SIZE = 2 MiB`, one `Transfer` per transaction to pay fees, 403 bytes per op with its proof):

| Bound | Ops per block |
|---|---|
| gas: 21 transactions of 255 ops + 1 of 35 | 5 390 |
| bytes: `(2 097 152 − 22 · 164) / 403` | ≈ 5 195 |

So a block that satisfies every consensus limit can contain ≈ 5 200 `SDPActive` ops, i.e. ≈ 5 200 unbatched Groth16 verifications: about `5 200 × 4.1 M ≈ 21 G cycles` on the reference cost model, against the `limit_Ex` budget of 3.2 G cycles per second that `overview-cryptoeconomics.md` §Execution Fee Market sets so that a leader can validate a block in one second; on this machine that is `5 200 × 6.18 ms ≈ 32 s`. Batched, the same 5 200 proofs would cost `5 200 × 1.90 ms ≈ 9.9 s`.

Who can fill such a block: (a) valid ops need one distinct provider each (duplicates are rejected, LB-003), so a block of 5 200 valid ops needs a network of 5 200 declarations, each paying `min_stake` — a legitimate load the fee market would then under-price by 7× every epoch; (b) invalid ops make the block invalid, so only a leader can put them in a block, and every validator then pays the full cost before rejecting it (the batch of signatures at `chain-service/src/lib.rs` L479 is never reached; the PoQ failures are hit one by one in `prepare_update`). A leader who wins a slot with a small stake can do this at every slot they win, and a block rejected for this reason is not remembered, so it can be re-sent.

**Exploit scenario**
Not an exploit on its own for (a); for (b), a staked leader publishes a block of 5 200 garbage `SDPActive` ops; every validator spends ≈ 32 s (this machine) or ≈ 7 s (reference model) rejecting it, during which that validator's chain service processes nothing else. Repeated per won slot.

**Recommendation**
- *Short term*: defer the PoQ verification into the block's deferred batch, next to the ZkSignatures: add a `PoQ(PoQProof, PoQVerifierInput)` variant to `DeferredZkpVerification`, have `verify_proof` return the public inputs instead of calling `verify`, and use the existing `lb_poq::batch_verify` in `DeferredZkpVerifications::verify`. Nothing in `apply_active_msg` after L103 needs the proof to be *verified*: the proof of selection binds the key nullifier carried in the proof bytes, the Hamming distance is computed on the token bytes, and the duplicate check keys on `provider_id`. Then raise `SDP_ACTIVE_GAS` to the marginal batched cost of a PoQ (measure it as the spec did for ZkSignature; on this machine the per-proof batched cost is 1.90 ms vs 1.91 ms for a ZkSignature, so the batched PoQ and the batched ZkSignature cost the same within 2 %, and `SDP_ACTIVE_GAS` should be about `2 × 590`).
- *Long term*: make the gas analysis account for every service's activity evaluation explicitly (S-001), and have a test that measures the CPU time of a block filled to `EXECUTION_GAS_LIMIT` with each op kind and asserts it stays under the 1 s budget on the reference hardware.

**References**: `analysis-gas-cost-determination.md` §SDP Activation ("Evaluation of the activity depends on the service and is neglected here") and §Gas determination from measures (ZkSignature table, 4 126 177 cycles at batch size 1); `overview-cryptoeconomics.md` §Execution Fee Market (`limit_Ex := 3 193 460`, one-second leader validation); `bedrock-v1.1-block-construction.md` §Batch verification of ZK proofs (defines batching for Proofs of Claim and ZkSignatures only).

### LB-003 · In `apply_active_msg` the pairing check runs before the duplicate-per-provider, proof-of-selection and activity-threshold checks, all of which are hash-cheap

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `ledger/src/mantle/sdp/rewards/blend/mod.rs:L95-L107`; `ledger/src/mantle/sdp/rewards/blend/target_epoch.rs:L103-L122`, `L157-L163`; `blend/message/src/reward/activity.rs:L50-L59` |
| Status | Open |

**Description**
Inside one op, the order is: PoQ Groth16 (`activity.rs` L50-L51), then proof of selection (L52-L59: index from a Blake2b hash mod `membership_size`, nullifier from a second hash), then Hamming distance against the threshold (`target_epoch.rs` L117-L122), then `TargetEpochTracker::insert`, which rejects a second message from the same provider for the same target epoch (L157-L163). The spec puts the one-message-per-node rule at the ledger (`blend-protocol.md` §Active Message: "The ledger must only accept a single active message per node per attested epoch. Any duplicate must be rejected") and says nothing about ordering, but the three later checks cost two hashes and a popcount, and they are the ones that a message from a *real* provider fails on: a provider that already has an accepted message this epoch (duplicate), a proof of quota not selecting this node (PoSel index mismatch), or a token above the threshold. Report #54 LB-001 shows a provider can produce unlimited distinct valid proofs of quota, so an insider can submit any number of second messages that each cost a pairing check before being rejected as duplicates. For an outsider the PoSel check is also a free filter: the attacker cannot produce a selection randomness that both hashes to its chosen node index and matches the nullifier it put in the PoQ bytes, unless it copies a real provider's published pair, which then fails the duplicate check.

**Exploit scenario**
Limited on its own (needs either an insider or LB-001's mempool path, which this ordering makes cheaper to defend against); listed because it is the precondition for the short-term fix of LB-001 and LB-002 to be effective.

**Recommendation**
- *Short term*: in `update_active`, check `submitted_proofs.contains_key(&provider_id)` before `verify_proof`; in `verify_proof`/`verify_and_build`, run the proof-of-selection check and the Hamming threshold on the unverified token before the PoQ verification (the token hash does not depend on verification). Keep the `epoch` and `providers.get` checks first as they are.
- *Long term*: with LB-002's batching, the PoQ leaves this path entirely and the remaining checks are all cheap; add a unit test that a duplicate or mis-selected message is rejected without calling the verifier (the `AlwaysSuccessProofsVerifier` test double can count calls).

**References**: `blend-protocol.md` §Active Message; report #54 LB-001.

### LB-004 · The transaction pool, which is also the SDP mempool, has no count or byte bound, no per-sender or per-op limit, and no admission validation; stale `SDPActive` ops leave only by TTL or by failing at a leader

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `services/tx-service/src/backend/pool.rs:L26-L52`, `L140-L173`, `L207-L224`; `services/tx-service/src/tx/service.rs:L536-L546`; `services/sdp/src/mempool.rs` |
| Status | Open |

**Description**
Parent #8 question 4 asks whether the SDP mempool is bounded and what evicts stale declarations. There is no separate SDP mempool: `services/sdp/src/mempool.rs` is a 24-line adapter trait whose only method is `post_tx` into the transaction pool. That pool:

- has one setting, `tx_ttl` (default and deployed value 24 h, `pool.rs` L27, `settings.yaml` L123); the `Evictor` composes exactly one policy, `TtlPolicy` (`evictor.rs` L11-L13);
- has no maximum number of items or bytes; `add_item` (L140-L173) rejects only an existing key;
- runs eviction only inside `remove` (L207-L224), which is called once per applied canonical block, so a node that sees no blocks evicts nothing;
- never re-checks an item against the ledger. An `SDPActive` for target epoch `E` is useless from the first block of `E+2` on (`InvalidEpoch`, `target_epoch.rs` L86-L91) but stays for the rest of its 24 h (about 14 epochs at 6 000 one-second slots) unless a leader tries it, fails, and removes it (`chain-leader` L727-L737); non-leaders keep it;
- has no per-sender, per-declaration or per-op-kind limit, so one declaration can have any number of `SDPActive` messages pending at once, of which at most one can ever be accepted per epoch.

Items are stored through the chain storage service into RocksDB (`rocksdb.rs` L49-L63) and every key is held in memory (`pending_items`, `by_prefix`, TTL map), and every proposal loads the whole pool into a `Vec` (`chain-leader` L673). A peer can therefore fill each node's disk at 2 MiB per item and up to 16 MiB per gossip message, for 24 h at a time, and make every leader's proposal allocate the whole pool.

**Exploit scenario**
Same peer as LB-001, but sending 2 MiB transactions instead of small ones: 10 000 of them (20 GB) sit on every node's disk for 24 h, and each leader materialises 20 GB in memory when it wins a slot (`L673`), which on an 8 GB node is an out-of-memory kill of the leader process. This is the resource-exhaustion counterpart of LB-001's CPU cost; #55 is the sub-issue for the general admission policy and should carry the full analysis.

**Recommendation**
- *Short term*: bound the pool by count and bytes with oldest-first eviction; stream the pool into the builder instead of collecting it (L673), with a cap on how many transactions a proposal examines; in `remove`, also drop `SDPActive` items whose `epoch` is below the tip's target epoch and `SDPDeclare`/`SDPWithdraw` items whose nonce is already spent.
- *Long term*: admission validation against the tip ledger as in LB-001, plus at most one pending `SDPActive` per `declaration_id` (replace on higher nonce), so the pool holds at most one message per provider per epoch, which is also all a block can use.

**References**: parent #8 question 4; #55; `bedrock-service-declaration-protocol.md` §Active Message ("The `nonce` must increase monotonically by every message sent for the `declaration_id`"), which already makes any second pending message for the same declaration and nonce dead.

### Checked and ruled out

- **Replay of an included `SDPActive`** fails at the nonce check (`active.rs` L87-L92) before any expensive work; the nonce is updated at execution (L122).
- **Withdrawn declarations** are rejected at `verify` (L77-L84) for `withdraw_at < epoch`, matching the spec's "the report attesting `withdraw_at − 1` is due during `withdraw_at`".
- **No target epoch** (fewer than `minimum_network_size` declarations, or a multi-epoch jump): `Rewards::WithoutTargetEpoch` rejects every message with `TargetEpochNotSet` before any verification (`rewards/blend/mod.rs` L70-L78). The `assert!(num_providers >= minimum_network_size)` at `target_epoch.rs` L94-L97 is guarded by `current_epoch.rs` L137-L143 and is not reachable from input.
- **Wrong target epoch** is rejected first and cheaply (`target_epoch.rs` L86-L91), so an attacker cannot make a node verify a proof against last epoch's verifier.
- **Metadata encoding**: the spec's 230-byte layout (`metadata_type`, `version`, epoch, key, PoQ, PoSel) is what the code writes: the type byte is emitted by `ActivityMetadata` (`core/src/sdp/mod.rs` L573-L580) and the version byte by `ActivityProof` (`core/src/sdp/blend.rs` L28-L35); both are checked at decode (`mod.rs` L590-L596, `blend.rs` L44-L49). No deviation.
- **Validator batching** of ZkSignatures matches `bedrock-v1.1-block-construction.md` §Batch verification: one batch per block after all transactions (`chain-service` L479), with public inputs taken from the state each op was reached in (`active.rs` L95-L97). The interleaved verify/execute order matches the Mantle spec §Validation.
- **Validators are not exposed to the pool**: a validator only ever applies transactions that a leader put in a block, so LB-001's cost lands on leaders (and LB-002's on everyone, once per block).
- **Garbage proof bytes** are rejected at decompression (`Groth16Proof::try_from`, `zk/groth16/src/proof/mod.rs` L156-L168) in 0.62 ms, so an attacker must supply valid curve points to force the pairing; any real proof's points, or generator multiples, do.

### Checklist answers

- **Cost of one PoQ verification vs one ZkSignature verification; how many garbage `SDPActive` ops fit in a block; does a builder verify candidates.** One unbatched PoQ: 6.18 ms here / ≈ 4.1 M cycles on the reference model. One ZkSignature: 14.7 ms unbatched, 1.88 ms per proof batched at 1 024 (the batched figure is the one the 590 gas pays for). ≈ 5 200 `SDPActive` ops fit in one block (bytes bound; 5 390 by gas). The builder verifies every candidate by full trial execution against the tip state, on every proposal, in every retry round (LB-001); garbage ops never reach a block because the builder discards them, at a cost of one PoQ each per round.
- **Per-sender or per-op rate limit, size bound, eviction of `SDPActive` ops whose target epoch has passed.** None, none, none, and only the 24 h TTL or a leader's failed attempt (LB-004).
- **Verifying the op signature before executing `SDPActive`, or checking cheap ledger conditions first, in the interleaved loop.** The signature's public inputs are already built at `verify` (`active.rs` L95-L97), so verifying it synchronously is a one-line change, but it is the most expensive option: a single ZkSignature verification costs 14.7 ms here, 2.4× the PoQ it would guard. The cheap conditions can all move ahead of the pairing without touching the loop (LB-003), and the PoQ itself can join the deferred batch (LB-002), after which the interleaved loop needs no change.
- **Admission-time validation of SDP ops against the tip ledger.** Proposed in LB-001 (long term) and LB-004: declaration exists, nonce above the ledger's, `epoch == target epoch`, provider in the target set, no accepted or pending message for `(declaration_id, target epoch)`, fee note covers `total_gas_cost`, ZkSignature verified once; one pending `SDPActive` per declaration; pool bounded by count and bytes; validate before forwarding.

## 5. Suggestions (non-security)

### S-001 · The gas analysis should price each service's activity evaluation, not neglect it

| | |
|---|---|
| Target | `logos-lips/docs/blockchain/raw/analysis-gas-cost-determination.md` §SDP Activation |

The sentence "Evaluation of the activity depends on the service and is neglected here" is where LB-002 originates. The Blend service is the only service and its evaluation is the dominant cost of the op. Either the SDP spec should require every service's activity check to be batchable and priced (add a `SERVICE_ACTIVITY_GAS[service]` term to `SDP_ACTIVE_GAS`), or the analysis should carry the measured cost for Blend, so that the `limit_Ex` reasoning in `overview-cryptoeconomics.md` holds for blocks full of `SDPActive` ops. Raise upstream.

### S-002 · The builder verifies deferred proofs one transaction at a time

| | |
|---|---|
| Target | `services/chain/chain-leader/src/lib.rs:L696` |

`deferred_zkps.verify()` is called per transaction, so a block of 1 024 fee-paying transactions costs the builder 1 024 single Groth16 verifications (`14.7 ms` each here) where the validator pays one batch (`1.88 ms` per proof). Collect the deferred proofs of accepted transactions and verify them once at the end of the selection (dropping the whole candidate set only if the batch fails, then bisecting), or at least once per round.

### S-003 · Move block proposal off the leader's select loop

| | |
|---|---|
| Target | `services/chain/chain-leader/src/lib.rs:L455-L520` |

The `TODO: spawn as a separate task?` at L506 is right: a proposal that takes longer than a slot delays every subsequent tick, and with LB-001 the proposal time is attacker-controlled. Spawn it with a deadline and drop the proposal if the slot has passed.

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
