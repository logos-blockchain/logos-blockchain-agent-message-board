# Audit Report — SDP Active: the Proof of Quota is unpriced and unbatched; measured batched cost and a deferral prototype

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/124`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `core/src/mantle/batch.rs`, `core/src/mantle/ops/sdp/active.rs`, `ledger/src/lib.rs`, `ledger/src/update.rs`, `ledger/src/mantle/sdp`, `blend/message/src/reward`, `blend/message/src/crypto/proofs.rs`, `blend/proofs/src/{quota,selection}`, `zk/proofs/{poq,zksign}`, `zk/groth16/src/verifier.rs`, `services/chain/{chain-service,chain-leader}`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `bedrock-v1.1-block-construction.md` (all in full); `analysis-gas-cost-determination.md` §Execution Gas, §SDP Activation, §Gas determination from measures; `blend-protocol.md` §Activity Proof, §Activity Threshold, §Active Message, §Reward Calculation; `bedrock-v1.1-mantle-specification.md` §Validation, §SDP_ACTIVE, §Gas Determination; `bedrock-service-declaration-protocol.md` §Active Message, §Active; `mantle-transaction-encoding.md` §SDP Operations (by section)
Date: `2026-09-12` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: the four items of #124 are done. At `a805329f` the Blend activity Proof of Quota (PoQ) of every `SDPActive` op is still verified one proof at a time during execution, on every node, and nothing after that call needs a verified proof; `lb_poq::batch_verify` is still unused outside benchmarks. Measured the way the gas analysis measured the ZkSignature, the marginal cost of a batched PoQ equals the marginal cost of a batched ZkSignature to within 0.2 % (two runs: ratio 1.000 and 1.002), so the activity evaluation that `SDP_ACTIVE_GAS = 590` neglects is worth exactly another 590 gas once batched, and 2.9× that as the code runs it today. A prototype that defers the PoQ into the block's existing batch (a third variant of `DeferredZkpVerification`, verified with `lb_poq::batch_verify`) compiles and passes the `core`, `blend-message`, `blend-proofs` and `ledger` test suites; with it, a block filled with `SDPActive` ops costs 1.8× the one-second leader budget instead of 3.7×, and raising the constant to `1 180` brings it to 1.0×. The batching spec lists Proofs of Claim and ZkSignatures only, and the `limit_Ex` deduction for batch initialisation covers those two only; both need a PoQ entry.
- Findings: 0 critical · 0 high · 1 medium · 0 low · 0 informational
- Key themes: "execution gas does not price the service activity check", "a batch verifier exists and the ledger does not use it", "the block-construction batching section and the fee-market deduction do not know about the PoQ"
- Must-fix before launch: LB-001 (defer the PoQ into the block batch and set `SDP_ACTIVE_GAS = 1 180`), together with the spec changes in S-001 and S-002 so the constant can be recomputed from the spec.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `core/src/mantle/ops/sdp/active.rs` L43-L45, L64-L103, L111-L133 | `GAS_COST = 590`; what `verify` checks and defers; `execute` |
| `core/src/mantle/batch.rs` L10-L13, L42-L69 | the two deferred proof kinds and how the batch is verified |
| `core/src/mantle/ops/transfer.rs` L96-L98; `core/src/mantle/transactions/mod.rs` L40; `core/src/block/mod.rs` L29-L32 | `TRANSFER` gas, `MAX_OPS_PER_TX`, `MAX_BLOCK_TRANSACTIONS`, `MAX_BLOCK_TRANSACTIONS_SIZE` |
| `ledger/src/lib.rs` L99, L183-L207, L475-L553, L743-L905, L922-L973 | `EXECUTION_GAS_LIMIT`, `prepare_update`, `try_apply_contents`, `try_apply_op`, `try_apply_tx` |
| `ledger/src/update.rs` L31-L38 | `PreparedUpdate::verify_batch_proofs` |
| `ledger/src/mantle/mod.rs` L238-L250; `ledger/src/mantle/sdp/mod.rs` L94-L109, L506-L545 | `try_apply_sdp_active`, `update_rewards`, `apply_active_msg` |
| `ledger/src/mantle/sdp/rewards/mod.rs` L27-L40; `rewards/blend/mod.rs` L63-L118; `rewards/blend/target_epoch.rs` L79-L125, L152-L183 | `Rewards::update_active`, `verify_proof`, `TargetEpochTracker::insert` |
| `blend/message/src/reward/activity.rs` L41-L66; `blend/message/src/reward/token.rs` L37-L52; `blend/message/src/crypto/proofs.rs` L81-L120 | PoQ then PoSel; Hamming distance on the token bytes; `RealProofsVerifier` |
| `blend/proofs/src/quota/mod.rs` L76-L85, L108-L123; `blend/proofs/src/selection/mod.rs` L72-L105 | `ProofOfQuota::verify`, decoding; `ProofOfSelection::verify` |
| `zk/proofs/poq/src/lib.rs` L115-L140; `zk/proofs/zksign/src/lib.rs` L114-L145; `zk/groth16/src/verifier.rs` L30-L83 | single and batched Groth16 verification |
| `zk/proofs/poq/benches/verify.rs`, `zk/proofs/zksign/benches/verify.rs`, `zk/groth16/tests/zk_signature_cpu_cycles.rs`, `ledger/benches/zk_batch.rs` | how the repository measures verification; the only users of `lb_poq::batch_verify` |
| `services/chain/chain-service/src/lib.rs` L460-L491; `services/chain/chain-leader/src/lib.rs` L682-L722 | where the validator and the builder verify the deferred batch |

**Out of scope**
The PoQ, PoSel and ZkSignature circuits and their keys (assumed sound), `ark-groth16`/`ark-bn254`/`ark-ec`, `rust-rapidsnark`, the mempool admission path (#98 LB-001/LB-004, #55), the ordering of the cheap activity checks before the pairing (#125), Blend reward arithmetic (#54, #76, #89), the network-layer PoQ verification in `blend/network` and `services/blend` (they share the `ProofsVerifier` trait but verify one message at a time by design), and the builder's per-transaction batch verification (#98 S-002).

**Assumptions**
The gas analysis' reference machine and cost model (`analysis-gas-cost-determination.md` §Gas determination from measures: one ZkSignature costs `3 900 000 + n × 590 000` cycles in a batch of `n`, 1 gas = 1 000 cycles) are the protocol's own. Cycle figures below are calibrated on that model through the ZkSignature marginal cost measured on the same machine in the same run, so machine speed cancels. The deployed parameters and limits are the code constants named in §2. The batch verifier's random scalars (`zk/groth16/src/verifier.rs` L45-L49) are drawn from `thread_rng`, as they already are for the ZkSignature batch.

## 3. Method

- Manual review of the in-scope paths, working through sub-issue #124 under parent #8, building on the report for #98 (PR #108) whose finding LB-002 this issue comes from. Every item of #124 was re-verified at `a805329f`; §4 "Checklist answers" maps the items to the results.
- Spec conformance against `bedrock-v1.1-mantle-specification.md` §SDP_ACTIVE (validation: declaration, nonce, ZkSignature; execution: "if the service-specific activity logic rejects the message, the Operation is invalid"), `bedrock-service-declaration-protocol.md` §Active (steps 1-6), `blend-protocol.md` §Activity Proof (PoQ true, PoSel true, Hamming distance within threshold) and §Active Message (one message per node per attested epoch), `bedrock-v1.1-block-construction.md` §Block Proposal Validation step 6 and §Batch verification of ZK proofs, `overview-cryptoeconomics.md` §Execution Fee Market, and `analysis-gas-cost-determination.md` §Execution Gas and §SDP Activation.
- Automated tooling: none.
- Dynamic testing, in a private copy of the tree at `a805329f` (the shared checkout was not modified), built with `RUSTFLAGS=""` to bypass the macOS `rust-lld` override in `.cargo/config.toml`, release profile with `lto = false`, `codegen-units = 16`, `strip = false`, `rustc 1.98.1`, on an Apple M4 Pro (14 cores, 48 GB, macOS 26.6.2):
  1. **Timing harness** (`audit-timing`, a workspace member; source below). It proves 32 distinct core-node PoQs from the repository's own benchmark fixtures (`zk/proofs/poq/benches/common/mod.rs`, key indices 0..32) and 32 distinct ZkSignatures, checks that every proof verifies and that a proof verified against another proof's inputs is rejected, then times: one unbatched verification of each kind, proof decompression, and `batch_verify` of each kind at the batch sizes the gas analysis tabulates (1..10, 20..200 in steps of 10) plus 1 024 and 5 200, cycling the pool to fill a batch. A least-squares line `total = a·n + b` is fitted over the 29 tabulated sizes, as the analysis did. Run twice; both runs are reported.
  2. **Deferral prototype.** The change described under LB-001 was applied to the private copy (12 files, about +150/−30 lines; the load-bearing hunks are quoted under LB-001) and `cargo test --release` was run for `logos-blockchain-core` (`batch` tests), `logos-blockchain-blend-message`, `logos-blockchain-blend-proofs` and `logos-blockchain-ledger`, plus `cargo check` of `logos-blockchain-chain-leader-service` and `logos-blockchain-chain-service`. Results under LB-001.

```rust
// audit-timing/src/main.rs (deps: lb-groth16, lb-pol, lb-poq, lb-poseidon2, lb-utils, lb-zksign,
// num-bigint, all `workspace = true`; `common` is zk/proofs/poq/benches/common/mod.rs with the
// crate path renamed to `lb_poq`)
mod common;
use std::time::{Duration, Instant};
use lb_groth16::{Fr, Groth16Proof};
use lb_poq::{KeyIndex, PoQProof, PoQVerifierInput, batch_verify as poq_batch_verify,
             prove as poq_prove, verify as poq_verify};
use lb_poseidon2::{Digest as _, Poseidon2Bn254Hasher};
use lb_zksign::{ZkSignPrivateKeysData, ZkSignProof, ZkSignVerifierInputs, ZkSignWitnessInputs,
                batch_verify as zk_batch_verify, prove as zk_prove, verify as zk_verify};
use num_bigint::BigUint;

const POOL: usize = 32;
const FIT_SIZES: [usize; 29] = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 20, 30, 40, 50, 60, 70, 80, 90,
    100, 110, 120, 130, 140, 150, 160, 170, 180, 190, 200];
const EXTRA_SIZES: [usize; 2] = [1024, 5200];

fn time<F: FnMut()>(iters: usize, mut f: F) -> Duration {
    for _ in 0..2 { f(); }
    let start = Instant::now();
    for _ in 0..iters { f(); }
    start.elapsed() / iters as u32
}
fn iters_for(k: usize) -> usize { (400 / k).clamp(3, 100) }
fn fit(points: &[(f64, f64)]) -> (f64, f64) {
    let n = points.len() as f64;
    let (sx, sy) = (points.iter().map(|p| p.0).sum::<f64>(), points.iter().map(|p| p.1).sum::<f64>());
    let (sxx, sxy) = (points.iter().map(|p| p.0 * p.0).sum::<f64>(), points.iter().map(|p| p.0 * p.1).sum::<f64>());
    let a = (n * sxy - sx * sy) / (n * sxx - sx * sx);
    (a, (sy - a * sx) / n)
}

fn main() {
    let poqs: Vec<(PoQProof, PoQVerifierInput)> = (0..POOL as u64)
        .map(|i| poq_prove(common::core_node_inputs(KeyIndex::try_new(i).unwrap())).unwrap()).collect();
    let zks: Vec<(ZkSignProof, ZkSignVerifierInputs)> = (0..POOL as u64).map(|i| {
        let sks: [Fr; 32] = core::array::from_fn(|k| BigUint::from(i * 32 + k as u64 + 1).into());
        let msg = Poseidon2Bn254Hasher::digest(&[BigUint::from(i).into()]);
        zk_prove(ZkSignWitnessInputs::from_witness_data_and_message_hash(ZkSignPrivateKeysData::from(sks), msg)).unwrap()
    }).collect();
    for (p, i) in &poqs { assert!(poq_verify(p, i.clone()).unwrap()); }
    for (p, i) in &zks { assert!(zk_verify(p, i).unwrap()); }
    assert!(!poq_verify(&poqs[1].0, poqs[0].1.clone()).unwrap());
    assert!(!zk_verify(&zks[1].0, &zks[0].1).unwrap());

    let poq_single = time(100, || assert!(poq_verify(&poqs[0].0, poqs[0].1.clone()).unwrap()));
    let poq_single_rej = time(100, || assert!(!poq_verify(&poqs[1].0, poqs[0].1.clone()).unwrap()));
    let zk_single = time(100, || assert!(zk_verify(&zks[0].0, &zks[0].1).unwrap()));
    let poq_expand = time(1000, || { let _ = std::hint::black_box(Groth16Proof::try_from(&poqs[0].0).unwrap()); });
    let zk_expand = time(1000, || { let _ = std::hint::black_box(Groth16Proof::try_from(&zks[0].0).unwrap()); });
    // ... printed as in the tables below ...

    let (mut poq_points, mut zk_points) = (Vec::new(), Vec::new());
    for &k in FIT_SIZES.iter().chain(EXTRA_SIZES.iter()) {
        let poq_batch: Vec<_> = (0..k).map(|j| poqs[j % POOL].clone()).collect();
        let zk_batch: Vec<_> = (0..k).map(|j| zks[j % POOL].clone()).collect();
        let iters = iters_for(k);
        let tp = time(iters, || assert!(poq_batch_verify(&poq_batch).unwrap()));
        let tz = time(iters, || assert!(zk_batch_verify(&zk_batch).unwrap()));
        let (mp, mz) = (tp.as_secs_f64() * 1e3, tz.as_secs_f64() * 1e3);
        if FIT_SIZES.contains(&k) { poq_points.push((k as f64, mp)); zk_points.push((k as f64, mz)); }
    }
    let ((ap, bp), (az, bz)) = (fit(&poq_points), fit(&zk_points));
    // marginal ratio ap/az; calibrated PoQ marginal = 590_000 * ap / az cycles; fixed = 3_900_000 * bp / bz
}
```

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | `SDP_ACTIVE_GAS = 590` pays for the ZkSignature only; the Blend PoQ that `apply_active_msg` verifies unbatched costs the same again batched and 2.9× that as run today, so a gas-valid block needs 3.7× the leader budget | Economic / Incentive | Medium | Medium | Open |

### What runs at `a805329f`

For one `SDPActive` op, on the builder and on every validator:

1. `ledger/src/lib.rs` L944-L970 (`try_apply_tx`): the op is verified (`verified_operations.next`, L944) and then executed (`try_apply_op`, L964) before the next op is verified. Verification of the op (`core/src/mantle/ops/sdp/active.rs` L64-L103) checks the declaration exists (L71), is not withdrawn (L78-L85), the nonce increases (L88-L93), and **defers** the ZkSignature into the block batch (L95-L102).
2. Execution (`active.rs` L111-L133, reached through `try_apply_op` L822-L842 → `ledger/src/mantle/mod.rs` L238-L250 → `ledger/src/mantle/sdp/mod.rs` L506-L545 `apply_active_msg`) sets `active` and `nonce`, then `update_rewards` (L538-L542) → `rewards/blend/mod.rs` L95-L100 `verify_proof` → `target_epoch.rs` L79-L125: epoch equals the target epoch (L86-L91), provider in the target set (L98-L101), then **`verify_and_build`** (L103-L109) → `blend/message/src/reward/activity.rs` L50-L51 `verify_proof_of_quota` → `crypto/proofs.rs` L95-L103 `ProofOfQuota::verify` → `blend/proofs/src/quota/mod.rs` L79 **`lb_poq::verify`** (`zk/proofs/poq/src/lib.rs` L115-L120) → `groth16_verify` (`zk/groth16/src/verifier.rs` L30-L38), one full pairing check per op. Then PoSel (`activity.rs` L52-L59), Hamming distance (`target_epoch.rs` L117-L122), and the duplicate-per-provider check (`rewards/blend/mod.rs` L102-L107 → `target_epoch.rs` L159-L164).
3. The validator verifies the deferred batch once per block, after every transaction has been applied (`chain-service/src/lib.rs` L460-L478 `prepare_update` → `ledger/src/update.rs` L31-L38 `verify_batch_proofs` → `core/src/mantle/batch.rs` L42-L45), and commits only then (L491). The builder verifies the batch once per candidate transaction (`chain-leader/src/lib.rs` L698). `lb_poq::batch_verify` (`zk/proofs/poq/src/lib.rs` L122-L140) is called from `zk/proofs/poq/benches/verify.rs` only.

Unchanged from the #98 report at `a77c248f`, except that the line numbers moved.

### LB-001 · `SDP_ACTIVE_GAS = 590` pays for the ZkSignature only; the Blend PoQ that `apply_active_msg` verifies unbatched costs the same again batched and 2.9× that as run today, so a gas-valid block needs 3.7× the leader budget

| | |
|---|---|
| Severity | Medium |
| Difficulty | Medium |
| Category | Economic / Incentive |
| Target | `core/src/mantle/ops/sdp/active.rs:L43-L45` (`GAS_COST = 590`); `ledger/src/mantle/sdp/rewards/blend/target_epoch.rs:L103-L109` (`verify_and_build`, one `lb_poq::verify` per op); `core/src/mantle/batch.rs:L10-L13` (no PoQ variant); `zk/proofs/poq/src/lib.rs:L122-L140` (`batch_verify`, unused by the ledger); `ledger/src/lib.rs:L99` (`EXECUTION_GAS_LIMIT = 3 193 460`) |
| Status | Open |

**Description**
`analysis-gas-cost-determination.md` §SDP Activation derives the 590 gas from "Verification of the ZK signature: 590,000 cycles", the marginal cost of one more ZkSignature in a batch (§Execution Gas: `3,900,000 + n × 590,000`), and states "Evaluation of the activity depends on the service and is neglected here". The Blend activity evaluation is a Groth16 PoQ verification, and the code runs it unbatched at execution (§"What runs"). This report measures what that evaluation costs, batched and unbatched, by the analysis' own method.

Measured on the audit machine (two runs, §3):

| Operation | Run 1 (ms) | Run 2 (ms) |
|---|---|---|
| PoQ `verify`, single, valid proof | 0.969 | 0.983 |
| PoQ `verify`, single, well-formed proof for other inputs (rejected) | 1.096 | 0.972 |
| ZkSignature `verify`, single, valid | 3.214 | 2.612 |
| PoQ / ZkSignature proof decompression (`Groth16Proof::try_from`) | 0.095 / 0.106 | 0.090 / 0.090 |
| PoQ `batch_verify`, 1 024 proofs (total) | 332.7 | 333.8 |
| ZkSignature `batch_verify`, 1 024 proofs (total) | 335.5 | 341.7 |
| PoQ `batch_verify`, 5 200 proofs (total) | 1 627.8 | 1 681.7 |
| ZkSignature `batch_verify`, 5 200 proofs (total) | 1 635.0 | 1 673.7 |

Linear fit over batch sizes 1..10 and 20..200, `total_ms = a·n + b`:

| | PoQ | ZkSignature | ratio PoQ / ZkSignature |
|---|---|---|---|
| Run 1: `a` (marginal, ms/proof) | 0.3267 | 0.3267 | 1.000 |
| Run 1: `b` (fixed, ms) | 2.650 | 3.006 | 0.881 |
| Run 2: `a` (marginal, ms/proof) | 0.3400 | 0.3392 | 1.002 |
| Run 2: `b` (fixed, ms) | 0.952 | 1.457 | 0.653 |

Calibrated on the analysis' ZkSignature line (590 000 cycles marginal, 3 900 000 fixed):

- **Batched PoQ, marginal: 590 000 cycles (run 1: 589 960; run 2: 591 321) = 590 gas.** The two circuits differ in public-input count (12 vs 33), but the per-proof work of the batch verifier (`verifier.rs` L51-L79: one G1 scalar multiplication of `pi_a`, one term in the multi-pairing, one term in the `pi_c` MSM) does not depend on it, so the equality is expected, not a coincidence.
- **Batched PoQ, fixed (batch initialisation): 2.5-3.4 M cycles** (run 1: 3 437 767; run 2: 2 547 451), i.e. about 2 550-3 440 gas. This is the third initialisation cost that `overview-cryptoeconomics.md` §Execution Fee Market does not deduct (it deducts 6 540 for ZkSignature and Proof of Claim only).
- **Unbatched PoQ, as the code runs it: 2.9× the batched marginal** (run 1: 2.96; run 2: 2.89), about 1.7 M cycles. The #98 report estimated this at 4.1 M cycles by using the analysis' single-ZkSignature figure as a proxy; the measured single PoQ is cheaper than a single ZkSignature (0.98 ms vs 2.6-3.2 ms here, the 33-term input MSM of the ZkSignature dominating the unbatched path), so #98's "≈ 21 G cycles, 7×" for a full block is corrected below to 12 G cycles, 3.7×.

Capacity of one block in `SDPActive` ops, from the code constants (`MAX_OPS_PER_TX = 255`, `EXECUTION_GAS_LIMIT = 3 193 460`, `MAX_BLOCK_TRANSACTIONS_SIZE = 2 097 152`, one `TRANSFER` of 590 gas per transaction to pay fees; 403 bytes per `SDPActive` op with its 128-byte signature per `mantle-transaction-encoding.md` §SDP Operations, 164 bytes per one-input `TRANSFER` with the op-count byte):

| Gas per `SDPActive` | Binding limit | Ops per block |
|---|---|---|
| 590 (today) | bytes | 5 194 (gas would allow 5 390) |
| 1 180 (proposed) | gas | 2 700 (11 transactions) |

Cost of validating a block full of `SDPActive` ops, on this machine, against the budget-equivalent of `limit_Ex` on this machine (3 193 460 gas × measured ms per 590 gas = 1.77 s in run 1, 1.84 s in run 2; the spec's "1 second on the reference machine"):

| Configuration | Work | This machine | × budget | Reference model |
|---|---|---|---|---|
| Today: PoQ inline, ZkSignatures batched, 5 194 ops | 5 194 × 0.98 ms + batch of 5 215 ZkSignatures | ≈ 6.8 s | 3.7 | 5 194 × 1.7 M + 5 215 × 0.59 M ≈ 12 G cycles |
| PoQ deferred into the batch, gas unchanged, 5 194 ops | batch of 5 194 PoQ + batch of 5 215 ZkSignatures | ≈ 3.4 s | 1.8 | 5 194 × 2 × 0.59 M ≈ 6.1 G cycles |
| PoQ deferred, `SDP_ACTIVE_GAS = 1 180`, 2 700 ops | batch of 2 700 PoQ + batch of 2 711 ZkSignatures | ≈ 1.8 s | 1.0 | 3.19 G cycles |

Who fills such a block is as in #98 LB-002: 5 194 valid ops need 5 194 distinct providers in the target epoch (a legitimate load the fee market then under-prices, every epoch, by 3.7× today and 1.8× once batched), and invalid ops can only be put in a block by a leader, who then makes every validator pay the full cost before the block is rejected (today: 5 194 separate pairing checks before the ZkSignature batch is even reached; batched: one batch that fails, but only after the whole block has been applied).

**Exploit scenario**
As #98 LB-002: not an attack on its own for the honest case; for the leader case, a small-stake leader publishes a block of ≈ 5 200 `SDPActive` ops with well-formed but invalid PoQs at every slot it wins, and every validator spends ≈ 6.8 s (this machine) or ≈ 3.7 s (reference model) rejecting each, in the chain service's critical path.

**Recommendation**
- *Short term*: (1) defer the PoQ into the block's batch, as prototyped below; (2) set `SDP_ACTIVE_GAS = 1 180` (`core/src/mantle/ops/sdp/active.rs` L44) and the spec constants with it (S-001); (3) lower `EXECUTION_GAS_LIMIT` by the PoQ batch initialisation, about 3 000 gas (S-002). With (1) and (2) the `SDPActive` op is priced exactly as `SDP_DECLARE` is priced today: the sum of the marginal batched costs of the proofs it carries.
- *Long term*: make every service's activity evaluation batchable and priced by construction (S-001), extend `ledger/benches/zk_batch.rs` to a block filled to `EXECUTION_GAS_LIMIT` with each op kind and compare it with the budget (S-003), and give the deferred-but-unverified PoQ its own type rather than borrowing `VerifiedProofOfQuota` (see the prototype note).

**Prototype (applied to a private copy of `a805329f`; tests below).** Twelve files, about +150/−30 lines. The pairing check leaves `apply_active_msg` and joins the block batch; the cheap checks (epoch, provider, PoSel, Hamming distance, duplicate) stay where they are. The `ProofsVerifier` trait gets a second entry point with an inline default, so the Blend network layer and the ledger test doubles are untouched; the `Rewards` trait gets the same; the deferred pair travels up through `apply_active_msg`, `try_apply_sdp_active` and `try_apply_op` into `DeferredZkpVerifications`.

```diff
--- a/blend/proofs/src/quota/mod.rs
+++ b/blend/proofs/src/quota/mod.rs
@@ -30,6 +30,10 @@
 pub use lb_poq::{KeyIndex, Quota};
+
+/// A Proof of Quota pairing check left for a batch: the compressed proof and
+/// the public inputs it must verify against.
+pub type DeferredProofOfQuotaVerification = (PoQProof, PoQVerifierInput);
@@ -82,6 +86,19 @@ impl ProofOfQuota {
+    /// Splits verification: returns the pairing check to run later in a batch,
+    /// together with the proof marked verified on the caller's promise that the
+    /// batch is checked before the enclosing state is adopted.
+    #[must_use]
+    pub fn defer_verification(
+        self,
+        public_inputs: &PublicInputs,
+    ) -> (VerifiedProofOfQuota, DeferredProofOfQuotaVerification) {
+        let verifier_input =
+            VerifyInputs::from_prove_inputs_and_nullifier(*public_inputs, self.key_nullifier);
+        (VerifiedProofOfQuota(self), (self.proof, verifier_input.into()))
+    }
--- a/blend/message/src/encap/mod.rs
+++ b/blend/message/src/encap/mod.rs
@@ -27,6 +27,19 @@ pub trait ProofsVerifier {
+    /// Proof of Quota verification split for batching: the cheap part runs
+    /// now and the pairing check is returned for the caller to batch. The
+    /// default verifies inline and defers nothing.
+    fn defer_proof_of_quota(
+        &self,
+        proof: ProofOfQuota,
+        signing_key: &Ed25519PublicKey,
+    ) -> Result<(VerifiedProofOfQuota, Option<DeferredProofOfQuotaVerification>), Self::Error> {
+        self.verify_proof_of_quota(proof, signing_key)
+            .map(|verified| (verified, None))
+    }
--- a/blend/message/src/crypto/proofs.rs
+++ b/blend/message/src/crypto/proofs.rs
@@ -109,6 +109,21 @@ impl ProofsVerifier for RealProofsVerifier {
+    fn defer_proof_of_quota(
+        &self,
+        proof: ProofOfQuota,
+        signing_key: &Ed25519PublicKey,
+    ) -> Result<(VerifiedProofOfQuota, Option<DeferredProofOfQuotaVerification>), Self::Error> {
+        let PoQVerificationInputsMinusSigningKey { core, leader, pow } = self.current_inputs;
+        let (verified, deferred) = proof.defer_verification(&PublicInputs {
+            core,
+            leader,
+            pow,
+            signing_key: *signing_key.as_inner(),
+        });
+        Ok((verified, Some(deferred)))
+    }
--- a/blend/message/src/reward/activity.rs
+++ b/blend/message/src/reward/activity.rs
@@ -43,12 +45,19 @@ impl ActivityProof {
         node_index: u64,
         membership_size: u64,
-    ) -> Result<Self, ProofsVerifier::Error>
+        defer_proof_of_quota: bool,
+    ) -> Result<(Self, Option<DeferredProofOfQuotaVerification>), ProofsVerifier::Error>
     {
-        let proof_of_quota =
-            verifier.verify_proof_of_quota(proof.proof_of_quota, &proof.signing_key)?;
+        let (proof_of_quota, deferred) = if defer_proof_of_quota {
+            verifier.defer_proof_of_quota(proof.proof_of_quota, &proof.signing_key)?
+        } else {
+            (verifier.verify_proof_of_quota(proof.proof_of_quota, &proof.signing_key)?, None)
+        };
         let proof_of_selection = verifier.verify_proof_of_selection( /* unchanged */ )?;
-        Ok(Self::new(proof.epoch, BlendingToken::new(proof.signing_key, proof_of_quota, proof_of_selection)))
+        Ok((Self::new(proof.epoch, BlendingToken::new(proof.signing_key, proof_of_quota, proof_of_selection)), deferred))
--- a/core/src/mantle/batch.rs
+++ b/core/src/mantle/batch.rs
@@ -1,4 +1,5 @@
 use lb_poc::{PoCProof, PoCVerifierInput};
+use lb_poq::{PoQProof, PoQVerifierInput};
@@ -10,6 +11,8 @@ pub enum DeferredZkpVerification {
     ZkSig(ZkSignProof, ZkSignVerifierInputs),
     LeaderClaim(PoCProof, PoCVerifierInput),
+    /// The Blend activity Proof of Quota of an `SDPActive` operation.
+    PoQ(PoQProof, PoQVerifierInput),
 }
@@ -17,6 +20,7 @@ pub struct DeferredZkpVerifications {
     zk_sigs: Vec<(ZkSignProof, ZkSignVerifierInputs)>,
     leader_claims: Vec<(PoCProof, PoCVerifierInput)>,
+    poqs: Vec<(PoQProof, PoQVerifierInput)>,
 }
@@ -31,19 +35,36 @@ impl DeferredZkpVerifications {
+            DeferredZkpVerification::PoQ(proof, public) => {
+                self.poqs.push((proof, public));
+            }
     pub fn verify(self) -> Result<(), Error> {
         Self::verify_zk_sigs(&self.zk_sigs)?;
-        Self::verify_leader_claims(&self.leader_claims)
+        Self::verify_leader_claims(&self.leader_claims)?;
+        Self::verify_poqs(&self.poqs)
     }
+    fn verify_poqs(proofs: &[(PoQProof, PoQVerifierInput)]) -> Result<(), Error> {
+        if proofs.is_empty() {
+            return Ok(());
+        }
+        match lb_poq::batch_verify(proofs) {
+            Ok(true) => Ok(()),
+            Ok(false) => Err(Error::InvalidProofsOfQuota),
+            Err(e) => Err(Error::MalformedProofOfQuota(format!("{e:?}"))),
+        }
+    }
```

The remaining hunks are plumbing: `ledger/src/mantle/sdp/rewards/mod.rs` adds `Rewards::update_active_deferred` (default: `update_active` and `None`); `rewards/blend/mod.rs` moves the body of `update_active` into `update_active_inner(.., defer_proof_of_quota: bool)` and implements both entry points with it; `rewards/blend/target_epoch.rs` `verify_proof` takes the flag and returns the deferred pair as a third tuple element; `ledger/src/mantle/sdp/mod.rs` `update_rewards` returns `Option<DeferredZkpVerification>` (wrapping the pair in `DeferredZkpVerification::PoQ`) and `apply_active_msg` returns it as a third element; `ledger/src/mantle/mod.rs` `try_apply_sdp_active` passes it through; `ledger/src/lib.rs` `try_apply_op` returns it as a fourth element and `try_apply_tx` pushes it into `deferred_zkps` next to the op's ZkSignature; `core/Cargo.toml` adds `lb-poq`. Four ledger tests that destructure `apply_active_msg`'s result and three that destructure `try_apply_op`'s were widened by one element; no assertion changed.

Test results on the prototype (`cargo test --release`): `logos-blockchain-core` `batch` tests 9 passed; `logos-blockchain-blend-message` 42 passed; `logos-blockchain-blend-proofs` 27 passed; `logos-blockchain-ledger` 162 passed, 1 ignored (the pre-existing ignored test); `cargo check` of `logos-blockchain-chain-leader-service` and `logos-blockchain-chain-service` clean. No behaviour changes for a block whose PoQs are all valid; a block with an invalid PoQ is now rejected at `verify_batch_proofs` (`batch::Error::InvalidProofsOfQuota`) instead of at `apply_active_msg` (`Error::InvalidProof`), which is where an invalid ZkSignature of the same op is already rejected, and in both cases nothing of the block is committed (`chain-service/src/lib.rs` L478-L491; `chain-leader/src/lib.rs` L698-L711).

Prototype note: the deferred path hands `BlendingToken::new` a `VerifiedProofOfQuota` built by the private constructor before the pairing check has run. That is the same promise `SignedOperation<_, Verified, _>` already makes for the deferred ZkSignature, but #223 is gating the unchecked `Verified*` constructors, so the upstream change should introduce a `PendingProofOfQuota` (or make `BlendingToken` generic over the proof's state) instead of reusing `VerifiedProofOfQuota`.

**References**: `analysis-gas-cost-determination.md` §SDP Activation, §Execution Gas ("the initialization cost for batch verification is paid by everyone and deduced from the block directly"), §Gas determination from measures (ZkSignature table); `overview-cryptoeconomics.md` §Execution Fee Market (`limit_Ex := 3,193,460`, 6 540 deducted for ZkSignature and Proof of Claim); `bedrock-v1.1-block-construction.md` §Block Proposal Validation step 6 ("batching covers the proof checks alone and does not change the state a transaction is validated in") and §Batch verification of ZK proofs; report #98 LB-002 (PR #108).

### Checked and ruled out

- **Nothing after the pairing check needs a verified PoQ** (item 1 of #124). The PoSel check reads the key nullifier from the proof bytes (`activity.rs` L57 `proof.proof_of_quota.key_nullifier()`, the field decoded at `quota/mod.rs` L112-L113) and binds it to the selection randomness (`selection/mod.rs` L94-L102); the Hamming distance is computed over the serialised token bytes (`token.rs` L42-L45 `self.to_bytes()`); the tracker keys on `provider_id` and stores `(zk_id, hamming_distance)` (`target_epoch.rs` L159-L182). `VerifiedProofOfQuota` carries no data the unverified proof does not (`quota/mod.rs` L152, a newtype).
- **State exposure before the batch is checked.** The validator holds the new state in `PreparedUpdate` and commits only after `verify_batch_proofs` (`update.rs` L31-L38; `chain-service/src/lib.rs` L478, L491); the builder adopts `new_state` only on `deferred_zkps.verify() == Ok` (`chain-leader/src/lib.rs` L698-L702). `prepare_update` is the only production caller of `try_update` (grep over `services/`, `consensus/`, `nodes/`), so the deferred PoQ has the same guarantee as the deferred ZkSignature.
- **Garbage proof bytes.** Inline, they fail at `Groth16Proof::try_from` inside `lb_poq::verify` (L117) in 0.09 ms; batched, `batch_verify` short-circuits on the same decompression (L131-L134) and the block is rejected as `MalformedProofOfQuota`. Same outcome, same cost.
- **Public inputs of the deferred PoQ** are built from the target epoch's `RealProofsVerifier` (`proofs.rs` L86, `self.current_inputs`), exactly as the inline path builds them (L96-L102), so a proof is checked against the epoch it was reached in, as `bedrock-v1.1-block-construction.md` §Block Proposal Validation step 6 requires for batched proofs.
- **The `expect` on the split signing key** (`blend/proofs/src/quota/inputs/verify.rs` L40-L43) is not reachable from input: each half of a 32-byte Ed25519 key is 16 bytes, below the BN254 scalar modulus.
- **`SDP_DECLARE` and `SDP_WITHDRAW`** carry no service-specific proof; their gas (646, 590) matches the batched marginal costs they trigger. Only `SDP_ACTIVE` has the gap.
- **The gas the code charges equals the spec's** (`active.rs` L44 = `EXECUTION_SDP_ACTIVE_GAS = 590` in `bedrock-v1.1-mantle-specification.md` §Gas Determination), so this is not a `Spec deviation`; the spec's own cost model is what under-prices the op (S-001).

### Checklist answers

- **Re-verify at head that `apply_active_msg` still calls the single `verify` and that nothing after it needs a verified proof.** Confirmed at `a805329f` (§"What runs", "Checked and ruled out" first bullet).
- **Defer the PoQ into the block's batch; record the spec gap.** Prototyped and tested (LB-001). Spec gap: `bedrock-v1.1-block-construction.md` §Batch verification of ZK proofs defines batching for Proofs of Claim and ZkSignatures only, and `overview-cryptoeconomics.md` §Execution Fee Market deducts initialisation for those two only (S-002).
- **Measure the marginal batched PoQ cost as the spec measured the ZkSignature; re-derive `SDP_ACTIVE_GAS`.** Marginal batched PoQ = 590 000 cycles calibrated (ratio to the ZkSignature 1.000 and 1.002 in two runs); `SDP_ACTIVE_GAS = 590 + 590 = 1 180`; PoQ batch initialisation 2.5-3.4 M cycles to deduct from `limit_Ex` (S-001, S-002).
- **A test that fills a block to `EXECUTION_GAS_LIMIT` with each op kind and asserts the one-second budget.** Not added as code: `ledger/benches/zk_batch.rs` benchmarks `TRANSFER`-only blocks, and a block of 2 700-5 200 `SDPActive` ops needs that many distinct PoQs (about 0.1 s each to prove here, and one declaration each). The block-level numbers were measured directly on the proof batches instead (LB-001 tables: 5 200-proof batches of each kind), and the benchmark extension is filed as S-003 and a follow-up issue.

## 5. Suggestions (non-security)

### S-001 · Price the Blend activity evaluation in the gas analysis: `SDP_ACTIVE_GAS = 1 180`

| | |
|---|---|
| Target | `logos-lips/docs/blockchain/raw/analysis-gas-cost-determination.md` §SDP Activation, §Execution Gas; `bedrock-v1.1-mantle-specification.md` §Gas Determination |

Replace "Evaluation of the activity depends on the service and is neglected here" with a measured line for the Blend service: "Verification of the Proof of Quota (batched): 590,000 cycles", add a `Proof of Quota batch verification | b + number_of_proof × 590,000` row to the measures table with `b` measured on the reference machine (2.5-3.4 M cycles calibrated here), and set `SDP_ACTIVE_GAS = 1 180` in both documents. The PoSel check (two Blake2b hashes, one Poseidon2 compression, one modulo) and the Hamming distance (two Blake2b hashes) fall under the analysis' "hashes ... are neglected". Structurally, `SDP_ACTIVE_GAS` should be written as `ZKSIGNATURE_GAS + SERVICE_ACTIVITY_GAS[service_type]` so that a second service cannot reintroduce the gap. Raise upstream.

### S-002 · Add Proofs of Quota to the batching section and to the `limit_Ex` deduction

| | |
|---|---|
| Target | `logos-lips/docs/blockchain/raw/bedrock-v1.1-block-construction.md` §Batch verification of ZK proofs; `overview-cryptoeconomics.md` §Execution Fee Market; `logos-blockchain/ledger/src/lib.rs:L99` |

§Batch verification of ZK proofs lists Proofs of Claim and ZkSignatures. Add "Proofs of Quota: the verifier follows the same procedure with the Groth16 proofs carried in the `metadata` of `SDP_ACTIVE` operations, with the public inputs of the target epoch each operation is reached in". §Execution Fee Market deducts 6 540 gas for the initialisation of two batches; a third batch adds its own fixed cost (2.5-3.4 M cycles measured here, to be measured on the reference machine), so `limit_Ex` becomes `3,200,000 − 6,540 − PoQ_batch_init`, about 3 190 000, and `EXECUTION_GAS_LIMIT` in the code with it. Raise upstream.

### S-003 · Extend `ledger/benches/zk_batch.rs` to blocks filled to `EXECUTION_GAS_LIMIT` with each op kind

| | |
|---|---|
| Target | `logos-blockchain/ledger/benches/zk_batch.rs` |

The benchmark builds `TRANSFER`-only blocks "because the ZK cost of other operations is not so different from `Transfer`" (file header), which is exactly the assumption LB-001 falsifies. Add a scenario per op kind that fills a block to `EXECUTION_GAS_LIMIT` (for `SDPActive`: `N` declarations in a target-epoch snapshot and `N` distinct PoQs generated once, as `zk/proofs/poq/benches/verify.rs` does with a proof pool), report the wall time of `try_apply_contents` plus `DeferredZkpVerifications::verify`, and compare it with a per-machine budget calibrated on the same run's batched ZkSignature marginal cost (`EXECUTION_GAS_LIMIT / 590 × ms_per_batched_zksig`), so the check is machine-independent. The harness in §3 is the proof-level half of this.

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
