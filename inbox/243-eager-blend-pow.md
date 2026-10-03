# Audit Report — Eager Blend PoW work and transaction-density retarget

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/243`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `blend/provers/src/provers/pow`, `utils/src/tokio/stream.rs`, `ledger/src/lib.rs`, `ledger/src/mantle/pow`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `proof-of-work.md`, `proof-of-quota.md` (in full); `blend-protocol.md` §Proof of Work Quota, §Blend Difficulty, §Generation, §Active Message; `bedrock-v1.1-mantle-specification.md` §SDP_ACTIVE, §SDP_DECLARE, §CLAIM_POW_REWARD; `bedrock-service-declaration-protocol.md` §Active Message
Date: `2026-10-03` — author: `codex` — status: `final`

This report extends report #159 (`processed/159-leader-pow-tokens-eligibility.md`, S-002 and S-003) with dynamic measurements and an explicit review of the pinned specifications' intent. It does not re-file either observation as a security finding.

---

## 1. Summary

- Overall assessment: the pinned implementation eagerly starts two PoW solution/proof jobs per `RealPowProofsGenerator` instance even without queued Blend traffic; the retarget counts every transaction in applied blocks, including protocol-internal Mantle transactions, but not genesis transactions.
- Findings: 0 critical · 0 high · 0 medium · 0 low · 0 informational
- Key themes: eager PoW work is bounded by the two-item buffer; observed proof waits and throughput are proportional to the retargeted difficulty; `d_blend` follows all applied transaction density, including `SDP_ACTIVE`.
- Must-fix before launch: none established by this issue. Consider the non-security follow-ups S-002 and S-003 below.

The direct production-generator experiment confirmed that both buffered proofs are ready after 45 seconds without a consumer request, at both the standalone base target and a target twice as hard. On this host, six successive one-message proof grants took 15.662 seconds at base difficulty and 34.582 seconds at twice-hard difficulty. Extrapolated across the standalone schedule's 6,000 one-second slots, that is about 2,298 and 1,041 grants per node per epoch, respectively; these are single-run generator-stage estimates, not end-to-end `POST /blend/transactions/disperse` guarantees. The target template's own calibration comment estimates about 50 seconds per solution on one Raspberry Pi 5 core, implying roughly 240 grants per 6,000-second epoch with two saturated search workers at base difficulty, or roughly 120 at twice the difficulty, before other service overhead.

For a closed standalone epoch containing only one `SDP_ACTIVE` per core node, the observed load is `N / (2 × 300)`. At `N = 200`, 600, and 2,000, this is `1/3`, `1`, and `10/3` of the configured reference load, producing ideal next retargets of approximately `1.732 × BASE`, `BASE`, and `0.548 × BASE`, respectively. The `2×` per-epoch clamp does not bind for these three cases. As specified by the epoch schedule, this load is read later: the threshold for epoch `n` is fixed during epoch `n−1` from epoch `n−2`.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `blend/provers/src/provers/pow/mod.rs` | Two-thread search pool, `BUFFER_SIZE`, buffered proof stream, solution mining and per-solution PoQ creation |
| `blend/provers/src/provers/core_leader_and_pow/mod.rs`; `blend/provers/src/provers/leader_and_pow/mod.rs` | Production construction sites for the PoW proof generator |
| `utils/src/tokio/stream.rs` | `Buffered::new` eager pre-poll behavior |
| `services/blend/src/core/mod.rs` | PoW proof retrieval in the transaction dispersal flow and queueing behavior |
| `ledger/src/lib.rs`; `ledger/src/mantle/pow/{mod.rs,blend_difficulty.rs,tx_density.rs}` | Applied-transaction counting, epoch closure, `d_blend` retarget, genesis state |
| `nodes/node/standalone-deployment-config.yaml`; `deployment/ceremony/genesis/standalone/deployment-template.yaml` | Standalone target, density reference, epoch schedule and slot duration |

**Out of scope**

No network-level node throughput, multi-node deployment, GPU implementation, adversarial benchmarking, or end-to-end HTTP latency was measured. PoQ circuit correctness, Groth16, Poseidon2, Blake2b, Rayon/Tokio internals, the transaction queue's network behavior, and the validity of the target-hardware timing comment are assumed correct. The separate Blend leader-branch eager-buffer behavior is discussed only for comparison.

**Assumptions**

The pinned code and pinned LIPS revisions listed above are the authoritative revisions for this report. The standalone deployment values are used for the requested `N` calculations (`base_difficulty = 19`, `target_transactions_per_block = 2`, `max_step = 2`, damping exponent `1/2`, one Blend layer). Each provider contributes one `SDP_ACTIVE` transaction in the epoch under consideration. For the throughput extrapolation, the standalone schedule has 6,000 slots and `slot_duration = 1s`; a solution yields one transaction proof at `pow_quota = 1`.

## 3. Method

- Reviewed issue #243 and its parent #8, report #159, comments and linked follow-up #770. #770 concerns adjacent ticket/controller work and does not complete #243's eager-mining or load-measurement questions. No canonical `LB-NNN` finding is associated with S-002/S-003; their existing classifications are preserved.
- Inspected the exact target revision in a detached scratch worktree at `/tmp/logos-blockchain-243` and the exact pinned LIPS revision in `/home/pluto/Code/logos/internal-audit/logos-lips`. The message-board checkout and the shared audit source checkout were not used as mutable source evidence.
- Compared implementation behavior with the listed LIPS sections, including the full `proof-of-work.md` and `proof-of-quota.md` documents.
- Built the pinned release node successfully with `rtk cargo build --release -p logos-blockchain-node --target-dir /tmp/target-243`.
- Ran a temporary, uncommitted release-mode test against the actual `RealPowProofsGenerator`, standalone `pow_quota = 1`, and production two-thread mining pool. With no `get_next_proof` request for 45 seconds, the first two proof requests each returned in 0 ms at both `2^19` and `2^20` thresholds. Following a fresh generator start and 10 ms of in-flight work, the six proof-call waits were 1,402, 1,710, 0, 1,166, 11,382, and 0 ms at `2^19` (15.662 s total), and 246, 16,091, 0, 7,557, 1,766, and 8,920 ms at `2^20` (34.582 s total). The test passed. The measurement includes PoQ generation and is one stochastic run on an AMD Ryzen 9 9950X host.
- A standalone node launch could not bind its QUIC listener in the sandbox; the host-like retry was stopped before execution. Consequently, no node trace-line count, per-thread CPU-time sample, or HTTP request latency is claimed. The in-process test exercises the same production generator and directly confirms eager work and its proof-call timing, but is not a substitute for those node-level measurements.
- Computed the throughput estimates as six grants divided by the observed saturated-call duration, multiplied by 6,000 seconds. They are estimates for the proof-generator stage, not service-level guarantees.
- No new target-repository source changes were made; the temporary test harness exists only in the scratch worktree.

## 4. Findings

No security findings were identified. The observations below retain the non-security `S-002` and `S-003` classifications from report #159; neither is assigned a new `LB-NNN` identifier.

## 5. Suggestions (non-security)

### S-002 · Each PoW proof generator eagerly mines and proves two solutions before Blend traffic asks for one

| | |
|---|---|
| Prior report | #159 S-002; classification unchanged |
| Target | `blend/provers/src/provers/pow/mod.rs`: `BUFFER_SIZE`, `create_proof_stream`, `spawn_solution_proofs`; `utils/src/tokio/stream.rs`: `Buffered::new` |

`new_mining_pool` creates two named `logos/blend/pow-puzzle-search-*` workers. For nonzero difficulty and quota, `create_proof_stream` wraps an unbounded `repeat_with(spawn_solution_proofs)` stream in `Buffered::new(..., 2)`. `Buffered::new` pre-polls the stream; creating each item immediately spawns the search-and-proof task. The in-process test showed that the first two standalone-quota proofs had both completed by 45 seconds with no consumer request, at both base and twice-hard difficulty. Since the stream can have only two items buffered, an idle consumer causes two jobs to run and then retains their finished proofs; it does not cause unbounded mining. When the epoch's generator is dropped, its cancellable work is dropped as well.

At standalone `pow_quota = 1`, one solution grants one message. A transaction arriving while both jobs are active can consume the next completed proof, then wait for subsequent proof generation if demand continues. In the sampled release run, six successive calls took 15.662 s at base and 34.582 s at twice-hard. With the configured 6,000-second epoch, those correspond to approximately 2,298 and 1,041 grants per epoch on this host. Applying the template comment's approximately 50 s per solution on one Raspberry Pi 5 core to two saturated workers gives a rough 240 grants per epoch at base and 120 after one 2× hardness step. Realized transaction throughput can be lower due to proof verification, message processing, queueing, and network delivery.

The LIPS proof-of-work specification defines the PoW branch's solution/quota relationship and epoch difficulty but does not require eager background mining. Eager prefetch can be useful to a leader proof on a proposal's critical path; the PoW-backed transaction path queues the transaction while awaiting its proof, so starting before a transaction exists does not remove a proposal-path stall. The observed behavior is therefore an implementation choice, not a spec requirement. Lazy-start on the first `get_next_proof` is the simplest option; alternatively, couple the producer to a non-empty transaction queue. Retain two-way buffering only if measurements show it materially lowers dispersal latency.

### S-003 · Blend difficulty counts protocol-internal transactions as load

| | |
|---|---|
| Prior report | #159 S-003; classification unchanged |
| Target | `ledger/src/lib.rs` (`try_apply_block` transaction count); `ledger/src/mantle/pow/{blend_difficulty.rs,tx_density.rs,mod.rs}` |

The canonical block-application path increments `txs_in_block` for each transaction passed through successful `try_apply_contents`, then records that total in `TxDensity`. The count is not filtered by opcode, so accepted `SDP_ACTIVE`, `SDP_DECLARE`, `CLAIM_POW_REWARD`, and user transactions all contribute. A rejected block does not commit the resulting ledger state. Genesis transactions do not enter this density: genesis `PowState` starts with `BlendPowState::default()`, and density is accumulated only by later applied-block recording. A test-only helper also records synthetic test blocks but is not a production call path.

With the standalone template, the reference is two transactions per block and the expected schedule is 300 blocks per epoch. If a closed epoch contains only `N` active messages, its load is `N / (2 × 300)`. The retarget formula is `BASE / sqrt(load)`, clamped to `[previous/2, previous×2]`:

| Active messages `N` | Load | Retarget from `BASE = p/2^19` |
|---:|---:|---:|
| 200 | `1/3` | `sqrt(3) × BASE ≈ 1.732 × BASE` (easier) |
| 600 | `1` | `BASE` |
| 2,000 | `10/3` | `BASE / sqrt(10/3) ≈ 0.548 × BASE` (harder) |

None of these ideal values reaches the `2×` clamp. The result is applied with the protocol's two-epoch lag: an epoch's traffic is not reflected until the relevant closed-epoch load is read at a later nonce snapshot.

At the pinned LIPS revision, `proof-of-work.md` defines `num_transactions(b)` over the transactions in each block without excluding system or protocol transactions, so counting accepted Mantle transactions is consistent with the formula as written. The spec does not explicitly call out internal transactions. Also, the LIPS parameter table gives `TARGET_TXS_PER_BLOCK = 512`, while the pinned standalone deployment uses `target_transactions_per_block: 2` and documents two as the actual reference load. The calculations above intentionally use the issue-requested standalone configuration; the spec/config reference-value divergence should be reconciled or clearly scoped in the specifications. It matters to the resulting difficulty, but does not change the conclusion that active messages count.

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
