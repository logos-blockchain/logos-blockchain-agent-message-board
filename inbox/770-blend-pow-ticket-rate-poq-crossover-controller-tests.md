# Audit Report · Blend PoW branch follow-up: the ticket rate and the proof-of-quota time measured on one x86 server core, the three long-run controller tests, and the grinder table still gated on the ticket fix at c4c86be1

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/770` (parent `#8`)
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `c4c86be18c58b5b09c3650e93871c8cfb624885b` · component(s): `blend/proofs/src/quota/pow.rs`, `zk/poseidon2/src/hasher.rs`, `zk/proofs/poq` (bench `prove`), `zk/circuits/prover/src/rapidsnark.rs`, `ledger/src/mantle/pow/{blend_difficulty.rs,difficulty.rs}`, `ledger/src/config.rs`, `blend/message/src/reward/token.rs`, `deployment/ceremony/genesis/testnet/deployment-template.yaml`
Specs: `https://github.com/logos-co/logos-lips` @ `d788723992a805b395f377de6e6cf59859b47168` · read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `proof-of-work.md` (all three in full); `blend-protocol.md` §Rewarding (Motivations, Mechanics), §Quota, §Proof of Quota, §Rewarding (Epoch Randomness to Rewarding Distribution Logic); `proof-of-quota.md` §Introduction to §Appendix; `common-cryptographic-components.md` §Poseidon2 (by section)
Date: `2026-09-25` · author: `Claude Code (research agent)` · status: `final`
Circuits: `https://github.com/logos-blockchain/logos-blockchain-circuits` @ tag `v0.5.7` (`ebf7ddf5b5b625ef4ea7c803171c670beebd05ef`, workspace `Cargo.toml` L176), prebuilt `logos-blockchain-circuits-v0.5.7-linux-x86_64`: `poq/proving_key.zkey` 12,578,709 bytes, sha256 `640315cc3ea5c4b3366c640ad6866afd5b43e19ad50b24dc1d788d354509321a`; `poq/verification_key.json` sha256 `a6898d3eb5353d1333c04f51c365edb669ab1f18c68bd9d9109fc7187bebe10f`. Prover: `rust-rapidsnark` rev `e91187f8` (`Cargo.toml` L269), rapidsnark v0.0.8, Logos `-fPIC` x86_64 build.

Follow-up to #111 (report `inbox/111-blend-pow-branch-difficulty-floor-and-token-eligibility.md`, PR #769, at `273658c7`), which built on #97 and #54 (`processed/97-poq-grinding-measurement.md`, `processed/54-blend-reward-claims.md`).

---

## 1. Summary

- Overall assessment: this machine has no GPU, so the GPU ticket rate stays unmeasured; everything else the issue asks for was measured or run at `c4c86be1`. One x86 server core computes 40,300 PoW candidates per second through the miner's own loop (13.0 s per solution at `base_difficulty: 19`, 2.6 s at the 5x ease, 0.81 s at 16x) and one proof of quota with the production v0.5.7 key in 1.08 s. The puzzle therefore becomes cheaper than the proof it gates at an ease of 12.1 on this core, against 12.7 to 12.9 on the #97 Raspberry Pi 5 core: the crossover barely moves between CPU classes, because both costs are BN254 field arithmetic. A PoW ticket is three Poseidon2 permutations, not one as #111 and this issue assumed, so #111's GPU band overstates a GPU's ticket rate by a factor of three (its conclusion stands). The three long-run controller tests of #111 S-004 were written in a scratch clone; the two that record today's behaviour pass (19 idle epochs to `p - 1`, 172 claimless blocks to `p - 1`) and the three that assert a floor fail, which confirms #111 LB-001 and LB-002 unchanged at `c4c86be1`. Neither the ticket fix nor a `d_blend` floor has landed (the three commits after `273658c7` touch only `logos_sql`, `zone-sdk` and cucumber tests), so item 4 stays gated. Fed into #111's model, the measured server-core constants already put 64 cores at 2.2% of the sole premium at the calibrated baseline with `N = 100`, so on this class of hardware item 4's "below 1%" bar cannot be met by a floor alone: it needs `base_difficulty - log2(ceiling) >= 21` (21 with the threshold pinned at the baseline, 24 with a ceiling of 8). Measuring the prover showed that every Groth16 proof the node makes appends 1,435 bytes to `MyLogFile.log` in the process's working directory, with no rotation.
- Findings: 0 critical · 0 high · 0 medium · 1 low · 1 informational
- Key themes: "the puzzle-to-proof ratio is a property of the arithmetic, not of the CPU", "the calibration is a Pi core and the adversary is a rack of server cores", "a C++ trace logger left on in the prover".
- Must-fix before launch: none new. #111 LB-001 and LB-002 (re-confirmed here by failing tests) and #97 LB-001 remain the blockers for the PoW branch.

Answers to the four checklist items:

1. **Ticket rate.** No GPU is attached (no `nvidia-smi`, no `/dev/dri` or `/dev/nvidia*`, and no PCI device of display class `0x03xxxx` among the eleven in `/sys/bus/pci/devices`; the machine is a KVM guest). On one core of an Intel Xeon (family 6 model 207) at 2.10 GHz, pinned with `taskset`: `PowTicket::derive` 42,100 to 44,400 tickets/s (median of 100 samples of 2,000, two runs), `solve_puzzle` 40,300/s (the miner's rate, including nonce sampling), one Poseidon2 permutation 127,000 to 130,000/s. Seconds per solution at the miner's rate: **13.0 s** at `2^19`, **2.60 s** at 5x, **0.81 s** at 16x (from `derive` alone: 12.2, 2.4, 0.76 s). This core is 3.8 times the #97 Pi 5 core (10,420/s through `solve_puzzle`). The GPU figure is not measured; it remains open in this issue (§4.1).
2. **Proof-of-quota time and the crossover.** `cargo bench -p logos-blockchain-poq --bench prove`, production key, pinned to one core: **1.076 s** median (10 samples, 98% of one core), 1.102 s for the leader branch, 0.323 s wall and 1.13 core-seconds per proof when rapidsnark may use all four cores. Crossover (solution cost equals proof cost): **ease 12.1** on this core, 12.7 to 12.9 on the Pi 5. The #111 LB-001 bound of "a floor at an ease of about 12" therefore holds for any CPU, and only non-CPU hardware moves it (§4.2).
3. **Long-run controller tests.** Written in `ledger/src/mantle/pow/{blend_difficulty.rs,difficulty.rs}` of a scratch clone (Appendix B), run with `cargo test --release -p logos-blockchain-ledger --lib s004`. Observed: 19 idle epochs from the baseline reach `p - 1` (the ease doubles every epoch, exactly); 172 claimless blocks from `p / 2^26` reach `p - 1`, and one block of 1,024 claims then hardens it to `p / 103`; at 1,000, 100, 10 and 1 transactions per 1,200-block epoch the threshold settles at 1.55, 4.90, 15.49 and 48.99 times the baseline after 1, 3, 4 and 6 epochs. With a ceiling of 8 times the baseline for `d_blend` and `2^8` times genesis for `d_reward`, the three floor tests fail (idle epoch 4 at 16x; loads 10 and 1; claimless block 53). Without a floor they cannot pass; they are the tests that pin one once it exists (§4.3).
4. **Grinder table.** Not re-run: `BlendingToken::hamming_distance` still hashes `self.to_bytes()` (`blend/message/src/reward/token.rs` L37-L52) and `compute_epoch_blend_difficulty` still has no lower clamp other than `previous / k` and no ceiling other than `p - 1` (`blend_difficulty.rs` L57-L64, L80); `git diff 273658c7..c4c86be1 -- blend ledger core zk` is empty. The item stays gated. §4.4 gives the #111 model with the measured constants as an interim, which is where LB-002 comes from.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `blend/proofs/src/quota/pow.rs` L42-L82 | `PowTicket::derive`, `solve_puzzle`: benchmarked |
| `zk/poseidon2/src/hasher.rs` L17-L78; `blend/proofs/src/lib.rs` L10-L24 | what `[pow_nonce, epoch_nonce].hash()` costs: rate-1 sponge, three permutations |
| `zk/proofs/poq/src/lib.rs` L65-L70; `zk/proofs/poq/benches/prove.rs`; `zk/circuits/prover/src/rapidsnark.rs` L10-L18 | proving time with the production key; the prover's side effects |
| `ledger/src/mantle/pow/blend_difficulty.rs` L44-L81; `difficulty.rs` L7-L53; `ledger/src/config.rs` L286-L319 | controllers under the long-run tests; re-verified unchanged since `273658c7` |
| `ledger/src/cryptarchia/mod.rs` L117-L139; `ledger/src/mantle/pow/mod.rs` L160-L163, L202-L235 | where each controller is fed (unchanged) |
| `blend/message/src/reward/token.rs` L37-L52 | ticket fix status |
| `deployment/ceremony/genesis/testnet/deployment-template.yaml` L36-L75 | parameters used in the tests and the model |

**Out of scope**

- No GPU measurement (no GPU on the machine). No GPU Groth16 proving measurement either.
- The PoQ circuit and its keys, Groth16, rapidsnark internals, Poseidon2 (assumed correct). The ticket grind itself (#54 LB-001, #97 LB-001), the missing floors' severity and recommendations (#111 LB-001, LB-002; re-verified, not re-rated), eager mining and `SDP_ACTIVE` counted as load (#243), the claim operation (#167, #636).
- No node was run; no devnet.

**Assumptions**

- The machine was shared with four other agents; load average during the runs was 0.6 to 3.9 on four vCPUs. Every measurement was pinned to one core, and the spread between fastest and median (about 7% for the ticket, 10% for the proof) is the contention noise. The numbers are a server-class x86 core, not a desktop; the issue's "desktop core" is taken to mean "a commodity CPU core other than the Pi".
- Pi 5 constants are #97's (10,560 tickets/s via `derive`, 10,420/s via `solve_puzzle`, 3.9 s per proof), not re-measured.

## 3. Method

- Worked through the four items of #770 (parent #8, source #111 S-004 and §4.3/§4.4). Read #111, #97 and #54 first; re-derived every line reference at `c4c86be1`. `git log 273658c7..c4c86be1` has three commits (`d92b17fb`, `8b34537a`, `c4c86be1`), none touching `blend/`, `ledger/`, `core/` or `zk/`; the `reward_target_floor` commit `badd079f` (#3562) predates `273658c7` and is the zero guard #111 already described. `reward_target_floor` moved to `config.rs` L292-L319 (was L283-L300 in #111) with no content change.
- Spec conformance: `proof-of-quota.md` §Pseudocode `pow_ticket = zkhash(pow_nonce, pol_epoch_nonce)` (L187) matches `pow.rs` L44-L46; `common-cryptographic-components.md` §Poseidon2 (rate 1, 10* padding, L144-L152) matches `hasher.rs` L24-L36, so `zkhash` of two elements is three permutations in both. `proof-of-work.md` §Blend Difficulty (L204-L240) and §Reward Difficulty (L185-L200) are what the tests exercise; code and pseudocode agree, as #111 found.
- Tooling: `rustc 1.98.1`, `cargo 1.98.1`, `divan 0.1.21`, `taskset`, Python 3.11.15. Builds: `CARGO_TARGET_DIR=/home/user/cargo-target CARGO_INCREMENTAL=0`, release/bench profile (fat LTO), only `-p logos-blockchain-blend-proofs --bench pow_ticket`, `-p logos-blockchain-poq --bench prove`, `-p logos-blockchain-ledger --lib`. The scratch clone added one bench file, a `divan` dev-dependency and the tests (Appendix B); the node checkout was not modified.
- Dynamic testing: benches and unit tests only (Appendix A has raw output). The existing 53 `mantle::pow` tests still pass alongside the new ones.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Every Groth16 proof appends a 1,435-byte trace block to `MyLogFile.log` in the node's working directory, never rotated | Auditing and Logging | Low | High | Open |
| LB-002 | `base_difficulty: 19` is calibrated to a Pi 5 core; on server cores the post-fix PoW grinder passes 1% of the sole premium at the baseline itself (`N = 100`), so no `d_blend` floor meets #770's bar | Economic / Incentive | Informational | Medium | Open |

### 4.1 Item 1: the ticket rate on one core, and what it says about a GPU

**No GPU here.** Checked `nvidia-smi` (absent), `lspci` (absent), `/dev/dri` and `/dev/nvidia*` (absent), and the PCI classes directly: one host bridge (`0x0600`), six virtio block, one virtio net and three virtio other devices, no display controller. No GPU number is given below as measured.

**CPU core, measured** (Appendix A.1; bench source in Appendix B.1):

| Operation | Median rate, one pinned core (two runs) | Per item |
|---|---|---|
| one Poseidon2 permutation (compression mode) | 127,300 / 129,600 per s | 7.8 µs |
| `PowTicket::derive` | 42,120 / 44,370 per s | 23 µs |
| `solve_puzzle` (sampling + ticket, no hit) | 40,300 / 40,240 per s | 25 µs |

The ticket costs 3.0 permutations, because `zkhash` is a rate-1 sponge with 10* padding (`hasher.rs` L31-L36; spec §Poseidon2 L144, L152): absorbing `pow_nonce`, `epoch_nonce` and the padding element are three calls to `permute_mut`. #111 §4.3 and this issue's first item describe the ticket as one permutation.

**Seconds per solution** (expected candidates `2^19 / ease`):

| Ease | This core, `solve_puzzle` rate | This core, `derive` rate | Pi 5 core (#97) |
|---|---|---|---|
| 1 (`base_difficulty: 19`) | 13.0 s | 12.2 s | 49.6 s |
| 5 | 2.60 s | 2.44 s | 9.9 s |
| 16 | 0.81 s | 0.76 s | 3.1 s |

**GPU, corrected assumption band (not a measurement).** If #111's band of 10^7 to 10^8 Poseidon2 permutations per second for a GPU is kept, a ticket rate is a third of it, 3.3 x 10^6 to 3.3 x 10^7 per second, so a solution at the baseline costs 157 ms to 16 ms (not 52 ms to 5.2 ms), 31 ms to 3.1 ms at 5x and 9.8 ms to 1.0 ms at 16x. The ratio to one CPU core is 80 to 800 server cores, or 300 to 3,000 Pi cores. The exponent at which the puzzle would cost one CPU proof-time on such a GPU is 25 to 26 (#111 said 27). #111's conclusion (on a GPU the proof, not the puzzle, is the rate limit) is unchanged, with one caveat #111 did not state: Groth16 proving also runs on GPUs (MSM and NTT kernels), so a GPU adversary's proof-time would not be the 1.08 s CPU figure either. Both numbers stay open in this issue.

### 4.2 Item 2: proving time with the production key, and the crossover

`zk/proofs/poq/benches/prove.rs` proves the repository's fixture (`benches/common/mod.rs`) with the embedded v0.5.7 key through `lb_poq::prove` → `lb_circuits_prover::Rapidsnark::prove` (`poq/src/lib.rs` L65-L70). The circuit is one three-branch circuit, so the PoW branch costs what the core and leader branches cost; there is no PoW-branch bench fixture.

| Run | Per proof | Notes |
|---|---|---|
| `bench_prove_core_node`, `taskset -c 3`, 5 samples | median 1.070 s (1.040 to 1.109) | |
| `bench_prove_core_node`, `taskset -c 2`, 10 samples | median 1.076 s (1.007 to 1.152) | 10.60 s user over 10.79 s real: 98% of one core |
| `bench_prove_leader`, `taskset -c 2`, 5 samples | median 1.102 s | same circuit |
| `bench_prove_core_node`, unpinned, 5 samples | 0.323 s wall | 5.66 s user for 5 proofs: 1.13 core-s per proof, no saving from threads |

**Crossover.** A solution costs less than the proof that accompanies it once `2^19 / (rate x ease) < t_prove`:

| Core | Candidates/s (`solve_puzzle`) | Proof | Ease at which the puzzle stops binding |
|---|---|---|---|
| Raspberry Pi 5 (#97) | 10,420 | 3.9 s | 12.9 (12.7 with #97's `derive` rate) |
| Xeon @ 2.1 GHz (this report) | 40,300 | 1.076 s | 12.1 |

The two costs scale together (3.8x for the ticket, 3.6x for the proof) because both are BN254 multiplications. #111 LB-001's recommended floor ("the puzzle at the ceiling still costs more than one proof on the target hardware, ease at most 12") is therefore not specific to the Pi: an ease ceiling of 8 keeps the puzzle binding on any CPU measured so far. The Pi-specific part is the absolute level, which is LB-002.

### 4.3 Item 3: the three long-run controller tests

Source in Appendix B.2, raw output in Appendix A.2. The deployed shapes: `d_blend` with `base_difficulty: 19`, `T_tx = 2`, `k = 2`, `alpha = 1/2`, 1,200 blocks per epoch; `d_reward` with `initial_difficulty: 26`, `F/P = 9/10`, `T = 1` (`deployment-template.yaml` L41-L49, L66-L71).

| Test | What it asserts | Result at `c4c86be1` |
|---|---|---|
| `s004_observe_idle_chain_epochs_to_field_max` | from `BASE`, zero-transaction epochs reach `p - 1` at epoch 19 | pass: 2, 4, 8, ... 262,144 x `BASE`, then `p - 1` at 19 and stays |
| `s004_idle_chain_stays_at_the_ceiling` (#111 S-004 test 1) | 25 idle epochs never exceed `8 x BASE` | **fail**: epoch 4 at 16.0 x |
| `s004_settling_point_at_small_load_stays_under_the_ceiling` (test 3) | fixed loads 1,000 / 100 / 10 / 1 tx per epoch settle at or below `8 x BASE` | **fail**: settles at 1.5491 (1 epoch), 4.8989 (3), 15.4919 (4), 48.9897 (6); loads 10 and 1 exceed |
| `s004_observe_claimless_blocks_to_field_max` | from `p / 2^26`, claimless blocks reach `p - 1` at block 172; then 1,024 claims | pass: 172; then `p / d = 103` |
| `s004_claimless_run_stays_at_the_ceiling` (test 2) | 200 claimless blocks never exceed `2^8 x` genesis | **fail**: block 53 |

The observations reproduce #111's analytical figures exactly (19 epochs, 172 blocks, 1.55 / 4.9 / 15.5 / 49 times, `p / 103`), so #111 LB-001 and LB-002 stand unchanged at `c4c86be1`. The floor tests fail for the reason #111 gives: `compute_epoch_blend_difficulty` returns `high = previous x k` on an empty epoch (`blend_difficulty.rs` L62-L64) and clamps only to `[previous / k, previous x k]` capped at `p - 1` (L57-L58, L80); `compute_new_reward_difficulty` multiplies by `P / F` per claimless block and caps only at `p - 1`, its floor of 9 units being a zero guard (`difficulty.rs` L43-L52; `config.rs` L292-L319). The ceiling constants (`MAX_EASE = 8`, `MAX_REWARD_EASE_LOG2 = 8`) are placeholders named at the top of each test; they are the knob the fix sets.

### 4.4 Item 4: the grinder table, gated; the model with measured constants

**Gate.** Not met at `c4c86be1`: the ticket is still Blake2b of the serialized token (`token.rs` L42-L45), so re-proving one witness still yields fresh tickets (#54 LB-001, #97 LB-001, filed as #334), and there is no `d_blend` floor. Item 4's re-run with the real `hamming_distance` and a chosen floor cannot be done; it stays in this issue.

**Interim.** #111 Appendix B's model with `HASH_RATE` and `T_PROVE` replaced by this core's measured `solve_puzzle` rate and proof time (script in Appendix B.3), testnet template, 1.4-epoch window, grinder's self-selecting PoW tokens after the fix:

| `N` | `Q_C` | Ease | s per self-token (Xeon core) | tokens, 1 core | P(unique premium), 1 core | tokens, 64 cores | P(unique premium), 64 cores | Pi 5 64 cores (#111) |
|---|---|---|---|---|---|---|---|---|
| 100 | 360 | 1 | 1,302 | 38 | 0.0003 | 2,477 | **0.0215** | 0.0057 |
| 100 | 360 | 5 | 261 | 192 | 0.0017 | 12,345 | 0.099 | 0.028 |
| 100 | 360 | 8 | 164 | 307 | 0.0027 | 19,704 | 0.150 | 0.044 |
| 100 | 360 | 16 | 82 | 611 | 0.0054 | 39,152 | 0.260 | 0.084 |
| 100 | 360 | `p - 1` | 1.1 | 46,732 | 0.29 | 2.99 M | 0.58 | 0.58 |
| 1000 | 36 | 1 | 13,011 | 3 | 0.0000 | 247 | 0.0022 | 0.0006 |
| 1000 | 36 | 5 | 2,603 | 19 | 0.0002 | 1,239 | **0.0108** | 0.0029 |
| 1000 | 36 | 8 | 1,627 | 30 | 0.0003 | 1,982 | 0.017 | 0.0046 |
| 1000 | 36 | 16 | 814 | 61 | 0.0005 | 3,961 | 0.034 | 0.0091 |
| 1000 | 36 | `p - 1` | 1.1 | 45,784 | 0.29 | 2.93 M | 0.58 | 0.58 |

Holding the ease at 1 and raising the exponent, 64 server cores fall under 1% at `N = 100` only from `base_difficulty: 21` (0.54%; 20 gives 1.08%), which is 52 s per solution on this core and 3.3 minutes on a Pi core. See LB-002.

### LB-001 · Every Groth16 proof appends a 1,435-byte trace block to `MyLogFile.log` in the node's working directory, never rotated

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Auditing and Logging |
| Target | `zk/circuits/prover/src/rapidsnark.rs:L10-L18` (`Rapidsnark::prove`, the single funnel into `rust_rapidsnark::groth16_prover_zkey_buffer_wrapper`); `Cargo.toml:L268-L269` (`rust-rapidsnark` rev `e91187f8`, rapidsnark v0.0.8 prebuilt `rapidsnark-linux-x86_64-pic-v0.0.8`); callers `zk/proofs/poq/src/lib.rs:L67`, `zk/proofs/pol/src/lib.rs:L78`, `zk/proofs/poc/src/lib.rs:L76`, `zk/proofs/zksign/src/lib.rs:L62` |
| Status | Open |

**Description**

Running the repository's PoQ bench left a file `MyLogFile.log` in the directory the bench was started from. A controlled rerun in an empty directory: 3 proofs wrote 87 lines, 4,305 bytes; a second process run of 3 more proofs appended to the same file (174 lines, 8,610 bytes). Each proof writes 29 lines, `<timestamp> [TRACE]: Start Multiexp A` through `Start Multiexp H` (Appendix A.3), 1,435 bytes. The strings (`[TRACE]: `, `Start Multiexp A`, and the file name) are in the statically linked prover inside the bench binary, not in the node's Rust code, and the node repository contains no reference to the file (`git grep MyLogFile` is empty). Every Groth16 proof the node generates goes through `Rapidsnark::prove` (PoQ, PoL, PoC, ZkSignature), so a running node appends to `./MyLogFile.log` relative to its working directory, forever: nothing rotates, truncates or configures it, and the node's tracing configuration (#37) does not govern it. In `deployment/Dockerfile` the runtime stage sets no `WORKDIR` (L23-L42), so the file lands in `/` of the container's writable layer.

Volume. A core node proves about `Q_C` proofs of quota per epoch, and `N x Q_C` is about the round count, so the network writes about `E x 1,435` bytes per epoch spread over `N` nodes: 52 MB per 10-hour testnet epoch, 8.6 MB per 100-minute standalone epoch. Per node that is 4.5 GB a year at `N = 10` on either template, 23 GB a year at the standalone `minimum_network_size: 2`, plus one block per PoL, per signed transaction and per PoW-branch proof. The content is not secret (step names and wall-clock seconds), but the timestamps of PoL proofs mark the slots a leader proved for, in a file an operator does not know exists and may ship with other logs.

**Exploit scenario**

No attacker lever: the node proves at its own pace. The impact is operational. A small deployment (the standalone template with a handful of core nodes, or a devnet run for weeks) fills the disk holding the node's working directory, which on a container is the same writable layer the node may write its database to, at a rate unrelated to anything the operator configured. A node whose disk fills fails its RocksDB writes (#81, #757).

**Recommendation**

- *Short term*: rebuild the pinned rapidsnark artifact without its file logger (the upstream logger is compiled in by a build flag), and add a test in `zk/circuits/prover` that runs one proof in a temporary working directory and asserts no file was created there. Until then, document it and set `WORKDIR` to a data directory in the runtime images.
- *Long term*: treat prebuilt native artifacts as a supply-chain surface (#151, #191 cover their provenance): pin their build flags alongside their hashes, and route any native logging into `tracing`.

**References**: iden3 rapidsnark v0.0.8 `groth16` prover trace points; #37 (logging volume and leaks), #81 and #757 (disk and durability), #224 (the same prover's witness handling).

### LB-002 · `base_difficulty: 19` is calibrated to a Pi 5 core; on server cores the post-fix PoW grinder passes 1% of the sole premium at the baseline itself (`N = 100`), so no `d_blend` floor meets #770's bar

| | |
|---|---|
| Severity | Informational |
| Difficulty | Medium |
| Category | Economic / Incentive |
| Target | `deployment/ceremony/genesis/testnet/deployment-template.yaml:L36-L41` (`base_difficulty: 19`, calibrated "on the target hardware itself (Raspberry Pi 5, one core)"); `nodes/node/binary/src/config/deployment/settings.yaml:L41`; `proof-of-work.md` L144 (`BLEND_DIFFICULTY_BASE = p // 2**19`), L240 |
| Status | Open |

**Description**

Item 4 of #770 sets the acceptance bar for the post-fix PoW branch: 64 cores stay below 1% at the sole premium for `N` in {100, 1000} on the testnet template. #111 §4.4 showed that a floor at the baseline meets it on Pi 5 cores (0.57% at `N = 100`). With the constants measured here for one commodity server core (40,300 candidates/s, 1.08 s per proof), the same model gives 2.15% at `N = 100` at the calibrated baseline, with no ease at all, and 1.08% at `N = 1000` at a 5x ease (§4.4 table). A floor cannot make the puzzle harder than the baseline, so no floor alone meets the bar on this hardware at `N = 100`; the baseline itself has to move to `base_difficulty: 21`, where an honest Pi miner then needs 3.3 minutes per solution. 64 server cores are 16 four-vCPU cloud instances. This is a model prediction for a code base with the ticket fix; today the grind is cheaper still (#97 LB-001), which is why this is Informational.

**Exploit scenario**

After the #97 S-001 ticket fix lands with a floor at `BASE`, a declared core node on a 100-provider testnet rents 64 server cores for the 14-hour window each epoch, mines PoW solutions whose proof of selection addresses its own index, proves each (1 s), and keeps the best ticket: 2,477 self-selecting tokens against an honest node's 360 received ones, and the sole premium in about one epoch in 47.

**Recommendation**

- *Short term*: state in `proof-of-work.md` §Blend Difficulty which adversary `BLEND_DIFFICULTY_BASE` is sized against, next to the Pi calibration (#111 S-002), and choose the base together with the floor. On this model, holding 64 server cores under 1% at `N = 100` with the threshold sitting at its ceiling needs `base_difficulty - log2(ceiling) >= 21`: with a ceiling of 8 times the baseline that is `base_difficulty: 24`, 27 minutes per solution for a Pi core at the reference load (`base_difficulty: 21` with a ceiling of 8 gives 4.2%). If that prices out honest Pi miners, the 1% bar or the premium rule has to give, not the floor.
- *Long term*: the grinder's share scales with `cores / N`, so a fixed exponent cannot serve every network size; tie the premium to more than one ticket (#97 LB-002 long term, "at least `m` distinct nullifiers") so that PoW tokens, which cost the grinder the same as anyone, stop being a lottery ticket per core.

**References**: #111 §4.4 and Appendix B (model), #97 S-001 (ticket fix), #97 LB-001 (#334), #97 LB-003 (#336); Appendix B.3.

## 5. Suggestions (non-security)

### S-001 · Count PoW work in tickets, not permutations, in the calibration and the GPU follow-up

| | |
|---|---|
| Target | `blend/proofs/src/quota/pow.rs:L44-L46`; `zk/poseidon2/src/hasher.rs:L31-L36`; `proof-of-quota.md` L187; `proof-of-work.md` L240 |

A ticket is `zkhash(pow_nonce, epoch_nonce)`, which in hash mode is three width-3 permutations (measured ratio 3.0). Anyone converting a published GPU Poseidon2 throughput into a ticket rate must divide by three, and a GPU kernel written for this puzzle must implement the padded sponge, not the bare permutation. The specification could also state the calibration as "`2^19` tickets of three permutations each". Using compression mode for the ticket (one permutation, as Merkle nodes and nullifiers do) would make both the puzzle and its in-circuit check cheaper, but it changes the circuit and gains nothing for the calibration, so it is noted, not recommended.

### S-002 · Spec: `common-cryptographic-components.md` §Poseidon2 gives capacity 3 for a width-3, rate-1 sponge

| | |
|---|---|
| Target | `common-cryptographic-components.md` L144-L148; `zk/poseidon2/src/hasher.rs:L7-L29` |

The section lists "rate = 1" and "capacity = 3" and a state "initialized with three 0s". A state of three elements with rate 1 has capacity 2, which is what the code implements (`state: [Fr; 3]`, absorbing into `state[0]`, `Poseidon2ParamsBn3`). The line should read "capacity = 2 (state width 3)".

### S-003 · Land the long-run tests with the floors

| | |
|---|---|
| Target | `ledger/src/mantle/pow/blend_difficulty.rs` tests; `ledger/src/mantle/pow/difficulty.rs` tests |

The five tests in Appendix B.2 are ready to paste. The two `observe` tests pin today's trajectory and should be updated when the floors land; the three `stays_at_the_ceiling` tests should replace `MAX_EASE` and `MAX_REWARD_EASE_LOG2` with the configured ceiling and then pass.

---

## Appendix A · Raw output

### A.1 PoW ticket bench (one core, 100 samples of 2,000)

```
$ taskset -c 3 pow_ticket-2e85feaa3cbc7df7 --bench --sample-count 100
pow_ticket                          fastest       │ slowest       │ median        │ mean          │ samples │ iters
├─ bench_poseidon2_one_permutation  14.86 ms      │ 21.8 ms       │ 15.7 ms       │ 15.89 ms      │ 100     │ 100
│                                   134.5 Kitem/s │ 91.73 Kitem/s │ 127.3 Kitem/s │ 125.8 Kitem/s │         │
├─ bench_pow_ticket_derive          44.57 ms      │ 74.14 ms      │ 47.47 ms      │ 48.99 ms      │ 100     │ 100
│                                   44.86 Kitem/s │ 26.97 Kitem/s │ 42.12 Kitem/s │ 40.82 Kitem/s │         │
╰─ bench_solve_puzzle_budget        47.42 ms      │ 82.74 ms      │ 49.62 ms      │ 50.81 ms      │ 100     │ 100
                                    42.17 Kitem/s │ 24.17 Kitem/s │ 40.3 Kitem/s  │ 39.36 Kitem/s │         │
$ taskset -c 2 ... (second run)
├─ bench_poseidon2_one_permutation  14.31 ms      │ 25.34 ms      │ 15.42 ms      │ 16.38 ms      │ 100     │ 100
│                                   139.7 Kitem/s │ 78.91 Kitem/s │ 129.6 Kitem/s │ 122 Kitem/s   │         │
├─ bench_pow_ticket_derive          41.42 ms      │ 67.18 ms      │ 45.07 ms      │ 46.37 ms      │ 100     │ 100
│                                   48.28 Kitem/s │ 29.76 Kitem/s │ 44.37 Kitem/s │ 43.12 Kitem/s │         │
╰─ bench_solve_puzzle_budget        45.1 ms       │ 69.43 ms      │ 49.68 ms      │ 51.03 ms      │ 100     │ 100
                                    44.33 Kitem/s │ 28.8 Kitem/s  │ 40.24 Kitem/s │ 39.18 Kitem/s │         │
```

PoQ prove bench:

```
$ taskset -c 3 prove-b6d08ba152d419eb --bench bench_prove_core_node --sample-count 5 --sample-size 1
╰─ bench_prove_core_node  1.04 s        │ 1.109 s       │ 1.07 s        │ 1.072 s       │ 5       │ 5
$ time taskset -c 2 prove-... --bench bench_prove_core_node --sample-count 10 --sample-size 1
╰─ bench_prove_core_node  1.007 s       │ 1.152 s       │ 1.076 s       │ 1.076 s       │ 10      │ 10
real 0m10.793s  user 0m10.595s  sys 0m0.068s
$ time taskset -c 2 prove-... --bench bench_prove_leader --sample-count 5 --sample-size 1
╰─ bench_prove_leader  1.072 s       │ 1.181 s       │ 1.102 s       │ 1.113 s       │ 5       │ 5
$ time prove-... --bench bench_prove_core_node --sample-count 5 --sample-size 1      # unpinned
╰─ bench_prove_core_node  304.6 ms      │ 366.9 ms      │ 323.1 ms      │ 326.1 ms      │ 5       │ 5
real 0m1.645s  user 0m5.659s  sys 0m0.072s
```

### A.2 Controller tests

```
$ logos_blockchain_ledger-846a1543b5bbd004 s004 --test-threads 1 --nocapture
test ...blend_difficulty::tests::s004_idle_chain_stays_at_the_ceiling ...
  panicked at ledger/src/mantle/pow/blend_difficulty.rs:313:13:
  idle epoch 4: d_blend is 16.0 x BASE, above the 8 x ceiling
FAILED
test ...blend_difficulty::tests::s004_observe_idle_chain_epochs_to_field_max ...
  idle epoch  1: d_blend / BASE = 2.0000
  idle epoch  2: d_blend / BASE = 4.0000
  ...
  idle epoch 18: d_blend / BASE = 262144.0000
  idle epoch 19: d_blend / BASE = 524288.0000
  idle epoch 20..25: d_blend / BASE = 524288.0000
  p - 1 first reached after Some(19) idle epochs
ok
test ...blend_difficulty::tests::s004_settling_point_at_small_load_stays_under_the_ceiling ...
   1000 tx/epoch: settles at 1.5491 x BASE after Some(1) epochs
    100 tx/epoch: settles at 4.8989 x BASE after Some(3) epochs
     10 tx/epoch: settles at 15.4919 x BASE after Some(4) epochs
      1 tx/epoch: settles at 48.9897 x BASE after Some(6) epochs
  panicked at ledger/src/mantle/pow/blend_difficulty.rs:349:9:
  settling points above 8 x BASE: [(10, 15.4919), (1, 48.9897)]
FAILED
test ...difficulty::tests::s004_claimless_run_stays_at_the_ceiling ...
  panicked at ledger/src/mantle/pow/difficulty.rs:291:13:
  claimless block 53: d_reward is 2^8 x genesis, above 2^8
FAILED
test ...difficulty::tests::s004_observe_claimless_blocks_to_field_max ...
  claimless block  20: log2(d / genesis) = 3
  ...
  claimless block 160: log2(d / genesis) = 24
  claimless block 172: log2(d / genesis) = 26
  p - 1 first reached after Some(172) claimless blocks
  after 1024 claims: p / d = 103
ok
test result: FAILED. 2 passed; 3 failed; 0 ignored; 0 measured; 172 filtered out
$ logos_blockchain_ledger-846a1543b5bbd004 mantle::pow --skip s004
test result: ok. 53 passed; 0 failed; 0 ignored; 0 measured; 124 filtered out
```

(The "2^8" in the reward failure message rounds by bit length; the value at block 53 is about `(10/9)^53 = 266` times genesis.)

### A.3 Prover log

```
$ mkdir logtest && cd logtest && prove-... --bench bench_prove_core_node --sample-count 3 --sample-size 1
$ wc -c -l MyLogFile.log          ->  87 lines, 4305 bytes (3 proofs)
$ (same again)                    -> 174 lines, 8610 bytes (appended)
Fri Sep 25 08:44:02 2026  [TRACE]: Start Multiexp A
Fri Sep 25 08:44:02 2026  [TRACE]: Start Multiexp B1
Fri Sep 25 08:44:02 2026  [TRACE]: Start Multiexp B2
Fri Sep 25 08:44:03 2026  [TRACE]: Start Multiexp C
Fri Sep 25 08:44:03 2026  [TRACE]: Start Initializing a b c A
Fri Sep 25 08:44:03 2026  [TRACE]: Processing coefs
Fri Sep 25 08:44:03 2026  [TRACE]: Calculating c
Fri Sep 25 08:44:03 2026  [TRACE]: Initializing fft
Fri Sep 25 08:44:03 2026  [TRACE]: Start iFFT A
Fri Sep 25 08:44:03 2026  [TRACE]: a After ifft:
... (Shift/FFT for a, b, c) ...
Fri Sep 25 08:44:03 2026  [TRACE]: Start ABC
Fri Sep 25 08:44:03 2026  [TRACE]: abc:
Fri Sep 25 08:44:03 2026  [TRACE]: Start Multiexp H
```

## Appendix B · Code

### B.1 PoW ticket bench (`blend/proofs/benches/pow_ticket.rs`, scratch only)

Manifest change: `divan = { workspace = true }` under `[dev-dependencies]` of `blend/proofs/Cargo.toml` and `[[bench]] harness = false, name = "pow_ticket"`. Built with `cargo bench -p logos-blockchain-blend-proofs --bench pow_ticket --no-run`, run with `taskset -c <core> <binary> --bench --sample-count 100`.

```rust
use core::num::NonZeroU64;

use divan::counter::ItemsCount;
use lb_groth16::Fr;
use lb_poseidon2::{Digest, Poseidon2Bn254Hasher};
use logos_blockchain_blend_proofs::quota::pow::{PowTarget, PowTicket, solve_puzzle};
use num_bigint::BigUint;
use rand::rngs::OsRng;

const BATCH: u64 = 2000;

fn main() { divan::main(); }

#[divan::bench]
fn bench_pow_ticket_derive(bencher: divan::Bencher) {
    let epoch_nonce: Fr = BigUint::from(42u64).into();
    bencher.counter(ItemsCount::new(BATCH as usize)).bench(|| {
        for i in 0..BATCH {
            divan::black_box(PowTicket::derive(epoch_nonce, BigUint::from(i).into()));
        }
    });
}

#[divan::bench]
fn bench_solve_puzzle_budget(bencher: divan::Bencher) {
    let epoch_nonce: Fr = BigUint::from(42u64).into();
    let difficulty: PowTarget = BigUint::from(1u64).into(); // no ticket wins
    bencher.counter(ItemsCount::new(BATCH as usize)).bench(|| {
        divan::black_box(solve_puzzle(epoch_nonce, difficulty, &mut OsRng,
                                      NonZeroU64::new(BATCH).unwrap()))
    });
}

#[divan::bench]
fn bench_poseidon2_one_permutation(bencher: divan::Bencher) {
    let b: Fr = BigUint::from(42u64).into();
    bencher.counter(ItemsCount::new(BATCH as usize)).bench(|| {
        for i in 0..BATCH {
            divan::black_box(<Poseidon2Bn254Hasher as Digest>::compress(&[BigUint::from(i).into(), b]));
        }
    });
}
```

### B.2 Long-run controller tests (appended to the existing `mod tests` of each file, scratch only)

`ledger/src/mantle/pow/blend_difficulty.rs`:

```rust
    fn deployed() -> BlendPoWConfig {
        BlendPoWConfig {
            base_difficulty: ModulusShift::new::<19>(),
            target_transactions_per_block: NonZeroU64::new(2).unwrap(),
            max_step: NonZeroU64::new(2).unwrap(),
            damping_num: NonZeroU32::new(1).unwrap(),
            damping_den_offset: 1,
        }
    }

    /// Largest ease a floor would allow; any value below the measured
    /// crossover (12.1 on the #770 Xeon core, 12.7 on the #97 Pi 5 core).
    const MAX_EASE: u32 = 8;

    fn ease(target: PowTarget, base: PowTarget) -> f64 {
        let scaled: BigUint = (as_int(target) * 10_000u32) / as_int(base);
        scaled.to_string().parse::<f64>().unwrap() / 10_000.0
    }

    #[test]
    fn s004_observe_idle_chain_epochs_to_field_max() {
        let config = deployed();
        let base = PowTarget::from(config.base_difficulty);
        let max_target = fr_from_biguint_saturating(fr_modulus());
        let mut d = base;
        let mut reached = None;
        for epoch in 1..=25u32 {
            d = compute_epoch_blend_difficulty(ClosedEpochLoad::new(0, 1_200), d, &config);
            eprintln!("idle epoch {epoch:2}: d_blend / BASE = {:.4}", ease(d, base));
            if reached.is_none() && d == max_target { reached = Some(epoch); }
        }
        assert_eq!(reached, Some(19));
    }

    #[test]
    fn s004_idle_chain_stays_at_the_ceiling() {
        let config = deployed();
        let base = PowTarget::from(config.base_difficulty);
        let ceiling = as_int(base) * MAX_EASE;
        let mut d = base;
        for epoch in 1..=25u32 {
            d = compute_epoch_blend_difficulty(ClosedEpochLoad::new(0, 1_200), d, &config);
            assert!(as_int(d) <= ceiling,
                "idle epoch {epoch}: d_blend is {:.1} x BASE, above the {MAX_EASE} x ceiling",
                ease(d, base));
        }
    }

    #[test]
    fn s004_settling_point_at_small_load_stays_under_the_ceiling() {
        let config = deployed();
        let base = PowTarget::from(config.base_difficulty);
        let ceiling = as_int(base) * MAX_EASE;
        let mut failures = Vec::new();
        for txs in [1_000u64, 100, 10, 1] {
            let mut d = base;
            let mut epochs_to_settle = None;
            for epoch in 1..=40u32 {
                let next = compute_epoch_blend_difficulty(ClosedEpochLoad::new(txs, 1_200), d, &config);
                if next == d && epochs_to_settle.is_none() { epochs_to_settle = Some(epoch - 1); }
                d = next;
            }
            eprintln!("{txs:5} tx/epoch: settles at {:.4} x BASE after {epochs_to_settle:?} epochs",
                      ease(d, base));
            if as_int(d) > ceiling { failures.push((txs, ease(d, base))); }
        }
        assert!(failures.is_empty(), "settling points above {MAX_EASE} x BASE: {failures:?}");
    }
```

`ledger/src/mantle/pow/difficulty.rs`:

```rust
    fn deployed() -> RewardPoWConfig { difficulty_config(9, 10, 1) }   // initial_difficulty 26

    const MAX_REWARD_EASE_LOG2: u32 = 8;

    fn genesis(config: &RewardPoWConfig) -> PowTarget { PowTarget::from(config.initial_difficulty) }
    fn int(t: PowTarget) -> BigUint { BigUint::from_bytes_le(&fr_to_bytes(&t)) }

    #[test]
    fn s004_observe_claimless_blocks_to_field_max() {
        let config = deployed();
        let max_target = -PowTarget::ONE;
        let mut d = genesis(&config);
        let mut reached = None;
        for block in 1..=200u32 {
            d = compute_new_reward_difficulty(0, d, &config);
            if reached.is_none() && d == max_target { reached = Some(block); }
        }
        assert_eq!(reached, Some(172));
        let after = compute_new_reward_difficulty(1_024, max_target, &config);
        eprintln!("after 1024 claims: p / d = {}", int(-PowTarget::ONE) / int(after));
    }

    #[test]
    fn s004_claimless_run_stays_at_the_ceiling() {
        let config = deployed();
        let ceiling = int(genesis(&config)) << MAX_REWARD_EASE_LOG2;
        let mut d = genesis(&config);
        for block in 1..=200u32 {
            d = compute_new_reward_difficulty(0, d, &config);
            assert!(int(d) <= ceiling,
                "claimless block {block}: d_reward is 2^{} x genesis, above 2^{MAX_REWARD_EASE_LOG2}",
                int(d).bits() - int(genesis(&config)).bits());
        }
    }
```

### B.3 Model with measured constants

#111 Appendix B's `binom_cdf_table`, `params`, `min_dist_pmf` and `outcome`, unchanged, with this driver:

```python
BASE_EXP = 19
MACHINES = {"pi5 (#97)": (10_560.0, 3.9), "xeon (#770)": (40_300.0, 1.076)}
for mname, (rate, tprove) in MACHINES.items():
    for N in (100, 1000):
        for ease in (1, 5, 8, 16, 2**19):
            secs = N * 2.0**BASE_EXP / ease / rate + tprove     # one self-selecting token
            window = 1.4 * 36_000                                # testnet
            m1, m64 = int(window / secs), int(64 * window / secs)
            Q_C, thr, pb1, pu1 = outcome(N, 36_000, 1, m1)
            _, _, pb64, pu64 = outcome(N, 36_000, 1, m64)
            print(mname, N, Q_C, ease, round(secs, 1), m1, round(pu1, 4), m64, round(pu64, 4))
for base in (19, 20, 21, 22):                                    # xeon, ease 1, 64 cores
    for N in (100, 1000):
        m64 = int(64 * 1.4 * 36_000 / (N * 2.0**base / 40_300.0 + 1.076))
        print(base, N, m64, round(outcome(N, 36_000, 1, m64)[3], 4))
```

The Pi rows reproduce #111 §4.4 to the printed precision (e.g. `N = 100`, ease 1, 64 cores: 649 tokens, 0.0057). Xeon sweep: base 19 → 0.0215 / 0.0022; 20 → 0.0108 / 0.0011; 21 → 0.0054 / 0.0005; 22 → 0.0027 / 0.0003 (`N = 100` / `N = 1000`).

---

## Appendix C · Definitions

### C.1 Severity

| Level | Definition |
|---|---|
| **Critical** | Loss of funds, chain halt, consensus split, or deanonymisation of users, exploitable by an unprivileged network participant with modest resources. |
| **High** | As above but requires significant resources, stake, timing, or a second weakness; or a remote crash/DoS of any node from a single unauthenticated peer. |
| **Medium** | Degrades safety/liveness/privacy guarantees under realistic conditions, or DoS requiring many peers / high cost; incorrect behaviour affecting a subset of users. |
| **Low** | Limited impact or unlikely preconditions; defence-in-depth gaps; reliability issues with a security flavour. |
| **Informational** | No immediate risk but relevant to best practice, maintainability, or future changes. |
| **Undetermined** | Needs more information from the team to rate. |

### C.2 Difficulty (to exploit)

| Level | Definition |
|---|---|
| **Low** | Well-known flaw; public tools exist or exploitation can be scripted. |
| **Medium** | Attacker must write an exploit or needs in-depth knowledge of the system. |
| **High** | Requires privileged access, complex technical details, or discovery of another weakness. |

### C.3 Categories

`Access Controls` · `Auditing and Logging` · `Authentication` · `Configuration` · `Cryptography` · `Data Exposure` · `Data Validation` · `Denial of Service` · `Error Reporting` · `Patching / Supply chain` · `Session Management` · `Timing` · `Undefined Behavior / Memory safety` · `Consensus` · `Economic / Incentive` · `Privacy / Anonymity` · `ZK Soundness` · `ZK Completeness` · `Determinism`
