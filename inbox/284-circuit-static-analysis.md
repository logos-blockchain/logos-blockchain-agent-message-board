# Audit Report — Automated under-constraint analysis of the four circuits (circomspect, Picus) and unused-signal review

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/284`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `3d5d419ec85c7b23e4d7e1455bd1cde845264a9e` — component(s): `zk/proofs/{pol,poq,poc,zksign}` (consumers of the keys; parity only)
Specs: `https://github.com/logos-co/logos-lips` @ `75d3d0382604d4a0d8e246c268935dd386ffc8ea` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `common-cryptographic-components.md` (in full); `cryptarchia-proof-of-leadership.md` §Circuit Public Inputs, §Circuit Constraints; `proof-of-quota.md` §Zero-Knowledge Proof Statement; `bedrock-anonymous-leaders-reward.md` §Voucher creation and inclusion, §Claiming the reward, §Validation, §Preventing Double Claims (by section)
Circuits: `https://github.com/logos-blockchain/logos-blockchain-circuits` @ `ebf7ddf5b5b625ef4ea7c803171c670beebd05ef` (tag `v0.5.7`, the `lbc-*-sys` pin in `Cargo.lock:4389-4391`), `circomlib` submodule @ `35e54ea21da3e8762557234298dbb553c175ea8d`. Verification-key hashes (sha256 of `verification_key.json` in the prebuilt bundle `logos-blockchain-circuits-v0.5.7-macos-aarch64`, the bundle `lbc-*-sys` embeds on this platform): PoL `973513d9a15e0641fa8d153ef678713f5f3dbb9883cae2320832d469b01d44a5`, PoQ `a6898d3eb5353d1333c04f51c365edb669ab1f18c68bd9d9109fc7187bebe10f`, PoC `94457cb5317dba09f96df09e69aa0be19a38c07d69a17cec8b0be6549606dae3`, ZkSignature `b6f8a7861308f0df1a7ead87f4d8158f59104738bb24a24b18904f0bbbbe0df9` (identical to the linux-aarch64 bundle hashes in the #20 report).
Date: `2026-09-15` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: clean. circomspect 0.9.0 reports no unconstrained signal, no unconstrained division, no non-strict binary conversion and no unconstrained comparison in any of the four circuits; its only warnings are the four deliberate dummy squares on public inputs and the unused `note_identifier` port in PoQ, both already recorded in the #20 report. Picus (z3 backend) proves every output of all four full circuits, and of the Poseidon2 compression and hash, the 32-level Merkle root, `SafeLessThan(20)` and `IsEqual` templates, uniquely determined by the inputs; the standalone `SafeFullLessThan` and `proof_of_membership(32)` wrappers were undecided by the z3 integer encoding within the caps, and both are decided inside the full circuits, where their outputs are pinned. `--O2` removes 722 (PoL), 726 (PoQ), 2 (PoC) and 0 (ZkSignature) non-linear constraints relative to `--O0`; all of them are constant folding of the first Poseidon2 permutation of the hashes whose first input is a domain-separation constant, and every public input, including the four dummy-squared ones, still appears in the `--O2` R1CS, with `nPublic` in the four embedded verification keys equal to public inputs plus outputs. The unused `note_identifier` port has no effect on the R1CS. The pinned circomlib commit is the current upstream `master` head; there are no upstream fixes to `LessThan`, `Num2Bits_strict` or `AliasCheck` to pick up.
- Findings: `0` critical · `0` high · `0` medium · `0` low · `0` informational
- Key themes: "a clean tool run is a result", "constant folding is not constraint loss"
- Must-fix before launch: none.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `mantle/pol.circom`, `mantle/pol_lib.circom`, `mantle/poc.circom`, `mantle/signature.circom`, `blend/poq.circom` | the four `main` components, analysed with circomspect and compiled at `--O0`, `--O1`, `--O2` |
| `misc/comparator.circom` (`SafeFullLessThan`, `SafeLessThan(20)`), `hash_bn/merkle.circom` (`compute_merkle_root(32)`, `proof_of_membership(32)`), `hash_bn/poseidon2_perm.circom` (`Compression`), `hash_bn/poseidon2_hash.circom` (`Poseidon2_hash(5)`), `ledger/notes.circom` (`derive_public_key`) | sub-templates wrapped as their own `main` for Picus and for the O0/O2 comparison |
| `circomlib` @ `35e54ea2`: `bitify.circom`, `comparators.circom`, `aliascheck.circom`, `compconstant.circom` | pin compared with upstream `iden3/circomlib` `master` |
| `~/Library/Caches/logos/blockchain/logos-blockchain-circuits-v0.5.7-macos-aarch64/{pol,poq,poc,signature}/verification_key.json` | the keys the node embeds, for `nPublic` and `IC` |

**Out of scope**

The semantic review of the constraints (done in the #20 report and not repeated: hash constructions, Merkle conventions, nullifier binding, the #20 LB-001 slot-grinding issue and LB-002 key halves); the proving-key provenance (`zkey verify`, also done in #20); the Rust witness generators and prover FFI (#127); `--O2` optimisations other than constraint counts and public-input survival. Third-party tools assumed correct: `circom`, `circomspect`, `snarkjs`, `Picus`, `z3`.

**Assumptions**

- The specification at the commit above is correct unless stated otherwise.
- The R1CS produced here from the `v0.5.7` sources with `circom 2.2.2 --O2` is the R1CS the bundled proving keys were built from; the #20 report verified this with `snarkjs zkey verify`, and the constraint counts here (20460 / 20110 / 8257 / 7681) equal the ones it recorded.

## 3. Method

- Manual review of the `.circom` sources listed in §2 to triage every tool result, working through issue `#284` under parent `#20`, all five checklist items.
- Spec conformance: the public-input sets checked against `cryptarchia-proof-of-leadership.md` §Circuit Public Inputs (8 inputs + `entropy_contribution`), `proof-of-quota.md` §Public values (11 inputs + `key_nullifier`), `bedrock-anonymous-leaders-reward.md` §Claiming the reward (`voucher_root`, `mantle_tx_hash` + `voucher_nullifier`), and the ZkSignature's 32 public keys + `msg`.
- Tooling, all on Apple M3 Pro, macOS 25.5:
  - `circomspect 0.9.0` (`cargo install --locked circomspect`), run as `circomspect -l INFO -L circomlib/circuits <main>` on each of the four `main` files, so every included template is analysed. Passes in this version: `bitwise_complement`, `bn254_specific_circuit`, `constant_conditional`, `definition_complexity`, `field_arithmetic`, `field_comparisons`, `nonstrict_binary_conversion`, `side_effect_analysis`, `signal_assignments`, `taint_analysis`, `unconstrained_division`, `unconstrained_less_than`, `under_constrained_signals`, `unused_output_signal`.
  - `circom 2.2.2` built from `iden3/circom` tag `v2.2.2` (`cargo install --locked --git … --tag v2.2.2`; the version `justfile:6` pins), `--r1cs --sym --inspect` at `--O0`, `--O1` and `--O2` for the four circuits and for eight wrapper `main`s around the sub-templates.
  - `snarkjs 0.7.5` (`npx snarkjs@0.7.5 r1cs print <r1cs> <sym>`) to list every `--O2` constraint with signal names and grep the public inputs.
  - Picus (`Veridise/Picus` @ `138b151`, 2024-03-11) on Minimal Racket 8.15 with `--solver z3` (Z3 5.1.0; the official cvc5 1.2.1 macOS binary is built without CoCoA and rejects `QF_FF`, so the cvc5 backend was not available without a source build), `--timeout 20000` per query, run on the `--O2` `.r1cs` of each wrapper and of each full circuit.
  - `git log 35e54ea2..HEAD` on a fresh clone of `iden3/circomlib`.
- Dynamic testing: none beyond the tool runs; no witness was computed in this pass (the #20 report did that against the spec test values).

### Checked and ruled out

**circomspect (checklist item 1).** PoL: "No issues found" at `INFO` level. PoC: 7 results, ZkSignature: 4, PoQ: many at `INFO`; every one triaged:

| Result | Where | Verdict |
|---|---|---|
| `under_constrained_signals`: "Intermediate signals should typically occur in at least two separate constraints" | `poc.circom:70-71` (`dummy`), `signature.circom:20-21` (`dummy`), `poq.circom:30-33` (`dummy_one`, `dummy_two`) | the deliberate quadratic pins of `mantle_tx_hash`, `msg`, `K_part_one`, `K_part_two`, whose only purpose is to keep the public input in the R1CS; #20 "Checked and ruled out" lists them. PoL's `dummy_one`/`dummy_two` (`pol_lib.circom:174-177`) are not reported because circomspect's heuristic sees the input used elsewhere in that template. Benign. |
| `unused_output_signal`: "The output signal `note_identifier` defined by the template `would_win_leadership` is not constrained in `ProofOfQuota`" | `poq.circom:106` | the `--inspect` warning of #20; checklist item 4 below shows it has no R1CS effect. Benign. |
| `field_arithmetic` (`note`): "Field element arithmetic could overflow" | every `*`/`+` on signals, including loop counters `i++` | circomspect flags all field arithmetic at `INFO`; each site is either a boolean check `s * (1 - s) === 0`, a Lagrange term of the PoQ selector, or the threshold `v * (t0 + t1 * v)`, whose wrap is the documented behaviour of `cryptarchia-proof-of-leadership.md` §Corner Case. Benign. |
| `field_comparisons` (`note`): "Comparisons with field elements greater than `p/2`" | loop conditions `i < 32`, `i < maxInput` | compile-time loop bounds on `var`s. Benign. |

No `unconstrained_division`, `unconstrained_less_than`, `nonstrict_binary_conversion`, `signal_assignments` (no `<--`), `bn254_specific_circuit` or `taint_analysis` result in any circuit. The `unconstrained_less_than` pass is the one that would flag a `LessThan` whose inputs are not range-checked; it is silent because `SafeLessThan(20)` range-checks both operands with `Num2Bits(20)` and `SafeFullLessThan` decomposes both with `Num2Bits_strict` (`comparator.circom:15-27`, `:69-72`). SARIF and text output for the four runs are in the scratch directory of this run and reproducible from the command above.

**Picus (checklist item 2).** Picus checks *weak safety*: that every output signal of the R1CS is uniquely determined by the input signals (public and private), by propagation lemmas first and SMT queries for whatever is left. It was run on the `--O0` R1CS rather than `--O2`: `--O2` substitutes every S-box output into three-unknown linear combinations of the next round's inputs, which defeats propagation and turns a 240-constraint permutation into hundreds of 20-second queries (measured: no result in 12 minutes for `Compression`), whereas at `--O0` the same permutation propagates in under a second. `--O2` is a solution-set-preserving projection of `--O0` (it eliminates linear constraints by substitution), so uniqueness of the outputs at `--O0` implies it at `--O2`. Results, `--solver z3 --timeout 5000`:

| R1CS (`--O0`) | non-linear / linear constraints | Picus | wall time |
|---|---|---|---|
| `Compression()` | 240 / 672 | properly constrained | 0.8 s |
| `Poseidon2_hash(5)` | 1440 / 4066 | properly constrained | 1.1 s |
| `compute_merkle_root(32)` | 7744 / 21505 | properly constrained | 1.9 s |
| `SafeLessThan(20)` | 61 / 10 | properly constrained | 0.6 s |
| `IsEqual()` | 2 / 6 | properly constrained | 0.7 s |
| `Num2Bits_strict()` | 516 / 769 | undecided: no verdict within the 600 s cap (same encoding) | 600 s |
| `SafeFullLessThan()` | 1300 / 2071 | undecided: no verdict within the 1800 s cap (z3 `QF_NIA` encoding of the 254-bit `Num2Bits_strict` decompositions) | 1800 s |
| `proof_of_membership(32)` | 7746 / 21507 | "Cannot determine whether the circuit is properly constrained" (the unpinned `IsEqual` output at the end of a 32-level path) | 24 s |
| **`zkSignature(32)`** (full) | 7681 / 21696 | **properly constrained** | 2.2 s |
| **`proof_of_claim`** (full) | 8259 / 22997 | **properly constrained** | 2.0 s |
| **`proof_of_leadership`** (full) | 21182 / 57654 | **properly constrained** | 4.7 s |
| **`ProofOfQuota(20,20)`** (full) | 20836 / 55012 | **properly constrained** | 4.5 s |

The two undecided wrappers are the two whose *output* is a comparison result left free at the top level. In every full circuit the same templates appear with their outputs pinned by an equality (`lottery_checker.out * unspent_membership.out === 1`, `reward_membership.out === 1`, `cmp.out === 1`, the PoQ branch equation), and the circuit outputs (`entropy_contribution`, `key_nullifier`, `voucher_nullifier`, the 32 public keys) are hashes of input signals, which is why Picus resolves all four full circuits by propagation alone. So the tool result is: no output of any deployed circuit can take two values for one assignment of the inputs. What it does not establish is the uniqueness of the comparator's *internal* bit signals when the comparison is taken as a free output; the #20 report's manual argument (`Num2Bits_strict` plus `AliasCheck` fixes the canonical decomposition, so the bits are unique) covers that, and a cvc5 build with CoCoA (`QF_FF`) is the way to have a tool confirm it (S-001). cvc5 1.2.1's official macOS binary answers `cvc5 can't solve field problems since it was not configured with --cocoa`, so it was not used.

**`--O2` and the public inputs (checklist item 3).** Constraint counts by optimisation level (`circom 2.2.2`):

| Circuit | `--O0` non-linear / linear | `--O1` | `--O2` | public inputs | outputs |
|---|---|---|---|---|---|
| PoL | 21182 / 57654 | 21180 / 22585 | 20460 / 0 | 8 | 1 |
| PoQ | 20836 / 55012 | 20831 / 20686 | 20110 / 0 | 11 | 1 |
| PoC | 8259 / 22997 | 8258 / 9352 | 8257 / 0 | 2 | 1 |
| ZkSignature | 7681 / 21696 | 7681 / 8800 | 7681 / 0 | 1 | 32 |

The public-input counts are the same at every level and equal the spec's, and `nPublic` in the four embedded verification keys is 9 / 12 / 3 / 33 with `IC` of length 10 / 13 / 4 / 34, i.e. inputs + outputs + 1 for every circuit. `--O2` does remove non-linear constraints, 722 in PoL and 726 in PoQ, which `--O1` (linear simplification only) does not. Isolating templates in their own `main` shows where they go: `Poseidon2_hash(5)` with five signal inputs has 1440 non-linear constraints at every level, but the same template with the `NOTE_ID_V1` constant as first input has 1440 at `--O0` and 1200 at `--O2`. With rate 1 the sponge absorbs the constant into the all-zero state and runs a full permutation on a state that is entirely constant, so its 240 S-box constraints fold away. PoL has three such hashes (note id, ticket, entropy: 720) and PoQ three (note id, ticket, selection randomness: 720); the remaining 2 and 6 come from `Num2Bits_strict` (516 → 515 per instance, the constant side of `AliasCheck`'s `CompConstant`) and from the `Compression` calls whose first lane is a constant (`derive_public_key`, PoC's two derivations). None of these constraints involves a public input, and none is a constraint whose removal changes the solution set, since a folded constraint is one whose every term is a compile-time constant. Confirmed directly on the `--O2` R1CS printed by `snarkjs r1cs print`: every public input appears in at least one constraint, with the dummy squares present as `[-x] * [x] - [-dummy] = 0`:

| Circuit | occurrences of each public input in the `--O2` R1CS |
|---|---|
| PoL | `sl` 12, `epoch_nonce` 6, `t0` 1, `t1` 1, `ledger_aged` 2, `ledger_latest` 2, `P_lead_part_one` 1, `P_lead_part_two` 1; output `entropy_contribution` 3 |
| PoQ | `core_quota` 2, `leader_quota` 1, `core_root` 2, `pow_quota` 1, `pol_ledger_aged` 2, `K_part_one` 1, `K_part_two` 1, `pow_blend_difficulty` 4, `pol_epoch_nonce` 13, `pol_t0` 1, `pol_t1` 1; output `key_nullifier` 3 |
| PoC | `voucher_root` 3, `mantle_tx_hash` 1; output `voucher_nullifier` 3 |
| ZkSignature | `msg` 1; the 32 `public_keys` outputs 96 |

**The unused `note_identifier` port (checklist item 4).** A copy of the sources with a second template `would_win_leadership_noport`, identical except for the removed `signal output note_identifier` and its assignment, used by `ProofOfQuota` compiles at `--O2` to the same 20110 non-linear constraints and the same 20168 wires; only the label count changes (75901 → 75900). The port is a linear alias of `note_id.out` that `--O2` substitutes away, so it costs nothing in the R1CS or the keys; removing it is a readability change only and would change the `.sym` layout but not the proving or verification keys. `circom --inspect` at 2.2.2 reports exactly the same three classes of warning the #20 report recorded (`Compression.perm.out[1..2]`, the unused bit arrays inside `LessThan`/`CompConstant`/`SafeLessThan`, and this port).

**circomlib pin (checklist item 5).** A fresh clone of `iden3/circomlib` has `35e54ea21da3e8762557234298dbb553c175ea8d` (2025-04-03) as the head of `master`; `git log 35e54ea2..HEAD` is empty for the repository and therefore for `comparators.circom`, `bitify.circom`, `aliascheck.circom` and `compconstant.circom`. There is nothing newer to compare against; the `LessThan`, `Num2Bits_strict` and `AliasCheck` the circuits use are the current upstream versions.

## 4. Findings

None. All five checklist items produced clean results or confirmed the #20 report's triage; the tool outputs and the reasons each result is benign are recorded in §3 so the next pass does not redo them.

## 5. Suggestions (non-security)

### S-001 · Make the tool runs part of the circuits repository's CI

| | |
|---|---|
| Target | `logos-blockchain-circuits` `justfile`, `.github/workflows/ci.yml` |

`circomspect -l WARNING` on the four `main` files completes in seconds and currently reports exactly six warnings, all expected; a CI step with `--allow` for those six IDs and a failing exit on anything new is cheap insurance for the next circuit edit. The `--O0`/`--O2` non-linear-constraint delta (722 / 726 / 2 / 0) and the per-public-input occurrence counts of §3 are likewise stable numbers worth asserting. Picus on the `--O0` R1CS of each `main` and of the sub-templates takes seconds with z3 and would take less with a `--cocoa` cvc5; the CI already builds each circuit, so the `.r1cs` is at hand. The one thing to add to the tool's environment is a cvc5 built with CoCoA, so that `SafeFullLessThan` and `proof_of_membership` get a verdict standalone rather than only inside the full circuits.

### S-002 · Remove the unused `note_identifier` port from `would_win_leadership`, or use it

| | |
|---|---|
| Target | `mantle/pol_lib.circom:66`, `:122`; `blend/poq.circom:106` |

It has no R1CS effect (§3), so this is housekeeping: either drop the output and have `proof_of_leadership` read `note_id` through a dedicated small template, or keep it and silence the `--inspect` warning with a comment. Not worth a key regeneration on its own; fold into the next circuit change.

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
