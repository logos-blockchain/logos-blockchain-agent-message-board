# Audit Report — Reward PoW difficulty adjustment and ticket validation

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/52` (parent `#7`)
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `ledger/src/config.rs`, `ledger/src/mantle/pow/{difficulty.rs,mod.rs}`, `ledger/src/lib.rs`, `core/src/mantle/ops/pow.rs`, `services/pow/src/{tickets.rs,service.rs}`, `zk/groth16/src/{lib.rs,modulus_shift.rs}`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `cryptarchia-total-stake-inference.md`, `block-rewards.md`, `cryptarchia-proof-of-leadership.md`, `proof-of-work.md`; relevant sections consulted in `cryptarchia-v1-protocol.md`
Date: `2026-09-16` — author: `agent (Codex)` — status: `final`

---

## 1. Summary

- Overall assessment: the pinned implementation avoids the requested arithmetic, replay, validation-cost, and target-encoding failures; one low-severity deployment-invariant gap accepts a zero reward-claim target that permanently disables claims after the next applied block.
- Findings: `0` critical · `0` high · `0` medium · `1` low · `0` informational
- Key themes: `wide-integer deterministic retargeting`, `consensus nullifier replay protection`, `deployment validation`
- Must-fix before launch: none from issue #52 alone; the positive deployment invariant remains the canonical recommendation for LB-001.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `ledger/src/config.rs` | Reward PoW deployment fields and load-time invariant checks |
| `ledger/src/mantle/pow/difficulty.rs`, `ledger/src/mantle/pow/mod.rs` | Integer difficulty adjustment, target clamping, and state update |
| `ledger/src/lib.rs` | Applied-block claim-event counting and difficulty update plumbing |
| `core/src/mantle/ops/pow.rs` | Ticket derivation, strict target comparison, claim validation, and nullifier execution |
| `services/pow/src/tickets.rs`, `services/pow/src/service.rs` | Ticket generation, acceptance-window pruning, and claim assembly |
| `zk/groth16/src/lib.rs`, `zk/groth16/src/modulus_shift.rs` | Field-byte endianness and initial target representation |

**Out of scope**

- Mining-key randomness and the cryptographic security of Poseidon, `arkworks`, and the field implementation were assumed correct; the review checked only their use and input ordering here.
- No changes were made to the audit source checkout, specifications checkout, or node repository. No e2e/devnet run was needed for the deterministic code paths in this issue.
- Third-party components assumed correct: `arkworks`, `rand`, `rpds`, and the Rust standard library.

**Assumptions**

- The reward difficulty is a strict field threshold: a ticket is valid only when `ticket < difficulty`.
- Reward PoW is intended to be enabled by a positive payout rate and a positive target claim count; `rate_num: 0` is the existing explicit disabling mechanism.
- Deployment configuration is deserialized through `RewardPoWConfig`, so the `RewardPoWConfigFields` conversion and `validate` method are the relevant admission boundary.

## 3. Method

- Manual review of the paths above, working through issue `#52` and parent issue `#7` after reading the applicable specifications, including the pinned `proof-of-work.md`.
- Spec conformance against `proof-of-work.md` §Parameters, §Puzzle Target, and §Reward Difficulty, which define `TARGET_CLAIMS_PER_BLOCK = 10`, strict target comparison, and the reward-difficulty update formula.
- Reviewed the recent PoW history, including `b8c3c54ff` (`chore: update pow ticket derivation`) and `13f0c2236` (`fix(pow): Use precomputed difficulty settings`), to distinguish current behavior from superseded implementation concerns.
- Traced the difficulty formula from applied-block claim events through `LedgerState::update_pow_reward_difficulty`, and traced a ticket from mining generation through validation and nullifier recording.
- Automated tooling: none beyond focused unit-test execution.
- Dynamic testing: focused Rust unit tests at the pinned source revision; no e2e/devnet execution.

Focused commands and results:

- `cargo test -p logos-blockchain-ledger difficulty --target-dir /tmp/logos-audit-52-target` — `29 passed; 0 failed`.
- `cargo test -p logos-blockchain-core pow --target-dir /tmp/logos-audit-52-target` — `19 passed; 0 failed`.
- `cargo test -p logos-blockchain-pow-service tickets --target-dir /tmp/logos-audit-52-target` — `12 passed; 0 failed`.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Zero reward-claim target accepted by deployment validation | Configuration | Low | High | Open |

### LB-001 · Zero reward-claim target accepted by deployment validation

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Configuration |
| Target | `ledger/src/config.rs:L164-L165,L245-L254` (`RewardPoWConfig::validate`); `ledger/src/mantle/pow/difficulty.rs:L41-L49` (`compute_new_reward_difficulty`) |
| Status | Open |

**Description**

`target_claims_per_block` is the target `T` used by the reward-difficulty controller, but it is represented as an unrestricted `u64` in both the runtime struct and the deserialization wire struct (`ledger/src/config.rs:L164-L165,L183-L193`). `RewardPoWConfig::validate` checks the EMA bounds and payout-rate multiplication overflow, but does not require `T > 0` (`ledger/src/config.rs:L245-L254`).

The pinned [`proof-of-work.md` §Parameters](https://github.com/logos-co/logos-lips/blob/7244d3b05ddec91a4a7b565bd5a9340ab77ededd/docs/blockchain/raw/proof-of-work.md#parameters) defines `TARGET_CLAIMS_PER_BLOCK: uint64 = 10`. Its [`Puzzle Target`](https://github.com/logos-co/logos-lips/blob/7244d3b05ddec91a4a7b565bd5a9340ab77ededd/docs/blockchain/raw/proof-of-work.md#puzzle-target) section defines a valid ticket as strictly below the target, and [`Reward Difficulty`](https://github.com/logos-co/logos-lips/blob/7244d3b05ddec91a4a7b565bd5a9340ab77ededd/docs/blockchain/raw/proof-of-work.md#reward-difficulty) uses `T` in the retarget formula. Thus the positive target is a normative protocol parameter, not an assumption inferred only from the implementation.

The retarget formula multiplies the next target by `T` (`ledger/src/mantle/pow/difficulty.rs:L41-L44`). With `T = 0`, the numerator floor still keeps the division defined, but the result is always zero. The subsequent cap and field conversion preserve zero (`ledger/src/mantle/pow/difficulty.rs:L46-L49`).

**Exploit scenario**

An operator supplies a deployment configuration with `rate_num > 0` and `target_claims_per_block: 0`. The configuration loads successfully. On the next applied block, `ledger/src/lib.rs:L286-L295` retargets the difficulty and sets it to `PowTarget::ZERO`, regardless of the number of accepted claims. Since the protocol's strict target rule and claim validation require a lower ticket (`core/src/mantle/ops/pow.rs:L51-L64,L307-L314`), no future ticket can satisfy `ticket < 0`; because the zero target is absorbing in the current controller, reward claiming remains disabled until the consensus state is explicitly repaired or reset. The shipped deployment values are positive, so this is an operator-misconfiguration path rather than an unauthenticated network attack. The demonstrated trigger is privileged deployment configuration, which makes Difficulty High under Appendix A.

**Recommendation**

- *Short term*: make `target_claims_per_block` a `NonZeroU64`, or add an explicit `target_claims_per_block == 0` validation error. Continue using `rate_num: 0` as the way to disable rewards.
- *Long term*: add deserialization and controller tests proving that zero is rejected, and retain a regression test that a positive target remains positive under an empty block and an extreme claim count unless an explicit protocol rule permits a zero target.

**References**: [`proof-of-work.md` §Parameters](https://github.com/logos-co/logos-lips/blob/7244d3b05ddec91a4a7b565bd5a9340ab77ededd/docs/blockchain/raw/proof-of-work.md#parameters); [`proof-of-work.md` §Puzzle Target](https://github.com/logos-co/logos-lips/blob/7244d3b05ddec91a4a7b565bd5a9340ab77ededd/docs/blockchain/raw/proof-of-work.md#puzzle-target); [`proof-of-work.md` §Reward Difficulty](https://github.com/logos-co/logos-lips/blob/7244d3b05ddec91a4a7b565bd5a9340ab77ededd/docs/blockchain/raw/proof-of-work.md#reward-difficulty); [`block-rewards.md` §Parametrization](https://github.com/logos-co/logos-lips/blob/7244d3b05ddec91a4a7b565bd5a9340ab77ededd/docs/blockchain/raw/block-rewards.md#parametrization); [`cryptarchia-v1-protocol.md` §Leadership Lottery](https://github.com/logos-co/logos-lips/blob/7244d3b05ddec91a4a7b565bd5a9340ab77ededd/docs/blockchain/raw/cryptarchia-v1-protocol.md#leadership-lottery)

### Requested checks with no confirmed finding

- **Adjustment overflow:** `compute_new_reward_difficulty` converts the field target to `BigUint`, performs the products and division as integer arithmetic, and caps the result at `p - 1` before converting back. The applied-block claim count is derived from the bounded transaction event list, and no fixed-width intermediate overflow was found.
- **Ticket validation cost:** mining and validation both derive one Poseidon ticket from the public key, block hash, and epoch nonce. The claim has no proof-verification work hidden in `preverify`; the reviewed code did not establish a practical validation-amplification defect.
- **Ticket replay:** validation checks the ticket against the consensus nullifier map before execution, and execution records the same derived ticket. Existing core tests cover duplicate-claim rejection.
- **Target comparison endianness:** field serialization/deserialization and `fr_from_mod_bytes` use little-endian byte order consistently for the difficulty conversion and ticket hash input. Existing core tests cover deterministic ticket binding and strict comparison boundaries.

## 5. Suggestions (non-security)

### S-001 · Keep the controller’s target invariant explicit in the type

| | |
|---|---|
| Category | Configuration |
| Target | `ledger/src/config.rs:L164-L165`; `ledger/src/mantle/pow/difficulty.rs:L7-L49` |

The controller’s comments and tests treat a positive target as its fixed point, while the field currently permits zero. Encoding the invariant with `NonZeroU64` would make the intended state machine clearer and prevent future callers from constructing an absorbing zero target directly in Rust tests or helper code.
