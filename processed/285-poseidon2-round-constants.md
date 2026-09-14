# Audit Report — Poseidon2 BN254 round constants: regeneration and the spec's derivation claim

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/285`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `zk/poseidon2` (via `jf-poseidon2`, git rev `8d80230358e900f8d63765a937f63f4978ca1daa` pinned at `Cargo.toml:108`)
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `common-cryptographic-components.md`
Date: `2026-09-12` — author: `claude-fable-5.1` — status: `final`

Circuits: `https://github.com/logos-blockchain/logos-blockchain-circuits` @ `ebf7ddf5b5b625ef4ea7c803171c670beebd05ef` (tag `v0.5.7`, pinned at `Cargo.toml:172-176`). Generator scripts: `https://github.com/HorizenLabs/poseidon2` @ `055bde3f4782731ba5f5ce5888a440a94327eaf3` (`poseidon2_rust_params.sage`), `https://extgit.isec.tugraz.at/krypto/hadeshash` @ `208b5a164c6a252b137997694d90931b2bb851c5` (`code/generate_params_poseidon.sage`). Paper: Poseidon2, ePrint 2023/323.

---

## 1. Summary

- Overall assessment: the 80 Poseidon2 round constants in the circuit, the node and the three Python input generators are exactly the output of the Grain LFSR of the Poseidon paper for `(GF(p), x^5, n=254, t=3, R_F=8, R_P=56)`; both scripts, the one the spec cites and the one HorizenLabs used, reproduce them, and the round numbers and matrices are the Poseidon2 paper's choices with the paper's security margin.
- Findings: 0 critical · 0 high · 0 medium · 0 low · 0 informational
- Key themes: "constants are reproducible from either script", "the spec's derivation sentence is imprecise, not wrong", "the spec's capacity value is off by one"
- Must-fix before launch: none

This closes the Scope exclusion and S-002 of the issue #20 report (PR #282). That suggestion's title, "not reproducible from the script the spec cites", is withdrawn by this report: the hadeshash script does reproduce them (§4, item 2).

## 2. Scope

**In scope**

| Path | Notes |
|---|---|
| circuits `hash_bn/poseidon2_perm.circom:28-84,102-136` | The 56 internal and 24 external round constants; `:89-91,144-146,155-157` the matrices |
| circuits `mantle/generate_inputs_for_pol.py:27,95`, `mantle/generate_inputs_for_proof_of_claim.py:27,95`, `blend/generate_inputs_for_poq.py:26,94` | Three further copies of the same 80 constants |
| jellyfish `poseidon2/src/constants/bn254.rs` @ `8d80230`, `poseidon2/src/{lib,external,internal}.rs` | Node-side constants, `MAT_DIAG3_M_1`, hard-coded `t = 3` matrix code |
| node `zk/poseidon2/src/hasher.rs` | Confirms the node instantiates `Poseidon2ParamsBn3` (t=3, d=5, 8/56) |
| HorizenLabs `poseidon2_rust_params.sage`, `plain_implementations/src/poseidon2/poseidon2_instance_bn256.rs` | The script and the instance jellyfish says it adapted |
| hadeshash `code/generate_params_poseidon.sage` | The script the spec cites |
| `common-cryptographic-components.md` §Poseidon2, §Annex | Derivation claim, parameters, twelve test values |

**Out of scope**

- Cryptanalysis of Poseidon2 beyond checking the round numbers against the paper's own bounds. The Gröbner-basis and statistical bounds implemented in the scripts are taken as the paper states them.
- The sponge, padding, Merkle and compression templates around the permutation, and the circuits' constraint soundness: PR #282.
- `circom`, `snarkjs`, `arkworks`, SageMath 10.6 and CPython 3.12 are assumed correct.

**Assumptions**

- The Grain LFSR construction of the Poseidon paper (ePrint 2019/458 §Round constants) is an acceptable nothing-up-my-sleeve source, as the spec and both papers assume.

## 3. Method

- Extracted the 24 + 56 constants from `poseidon2_perm.circom`, `bn254.rs` and `poseidon2_instance_bn256.rs` by script and compared the three lists element-wise (all 80 are `< p`, and all three lists are identical).
- Ran `poseidon2_rust_params.sage` at HorizenLabs `055bde3` with the `p` line switched from BLS12-381 to `21888242871839275222246405745257275088548364400416034343698204186575808495617` (the only edit; `t = 3` is the file's default) under SageMath 10.6. Its own round-number search printed `R_F = 8, R_P = 56`, its matrix subspace-trail checks passed (the script exits otherwise), and its 192-entry `RC3` (partial rounds padded `[c, 0, 0]`) was compared with the circuit.
- Ran `generate_params_poseidon.sage` at hadeshash `208b5a1` as `sage generate_params_poseidon.sage 1 0 254 3 5 128 0x30644e72e131a029b85045b68181585d2833e84879b9709143e1f593f0000001` (field `GF(p)`, S-box `x^alpha`, `n = 254`, `t = 3`, `alpha = 5`, `M = 128`, BN254 modulus), with no edits, and compared its 192 constants with the circuit. A second run with `R_F, R_P` forced to `8, 56` was identical to the default run because the script already chose `8, 56`.
- Re-derived the round numbers with the HorizenLabs `find_FD_round_numbers` with `security_margin=False` and `=True` to state the margin.
- Read Poseidon2 ePrint 2023/323 §3.2 (round numbers and margin), §5.1-5.2 and §6 "Case t ∈ {2,3}" (matrices), §6 (round constants, Table 1), §7.1 (margin rationale).
- Wrote an independent 40-line Python implementation of the Grain LFSR (Appendix B) and of the permutation; regenerated the 80 constants from the six parameters alone, then recomputed the twelve Annex test values of the spec from the regenerated constants (8 hash-mode, 4 compression-mode: all match).
- Tooling: SageMath 10.6 (local, not the `sagemath/sagemath` Docker image of `justfile:27`), CPython 3.12.
- Dynamic testing: none.

## 4. Findings

None. Every item of the issue checks out; the record of what was verified follows, in the issue's order.

**1. HorizenLabs `poseidon2_rust_params.sage` reproduces the constants.** With `p` set to the BN254 scalar field and `t = 3`, the script's 192-entry output has rows 0-3 and 60-63 equal to `poseidon2_perm.circom:102-136` (the 8 external rounds, in order) and rows 4-59 equal to `[c_i, 0, 0]` with `c_i` the 56 constants of `poseidon2_perm.circom:28-84`, in order. Its `state_out` for input `(0, 1, 2)` equals the independent implementation's permutation of the same input. The script hard-codes `alpha = 5` via `get_alpha(p)` (smallest `alpha ≥ 3` coprime to `p − 1`; for BN254 that is 5).

**2. hadeshash `generate_params_poseidon.sage` reproduces the constants too; the spec's sentence is imprecise, not wrong.** Poseidon2 keeps the Poseidon round-constant generator unchanged ("All round constants are generated as in Poseidonπ", 2023/323 §6), and both scripts seed the same 80-bit Grain state `(field=1 | sbox=0 | n=254 | t=3 | R_F=8 | R_P=56 | 1^30)`, apply the same 160-step warm-up, the same one-bit-discard rule, and the same rejection sampling of 254-bit words `≥ p`. The only difference is how many words are drawn and where they are placed: hadeshash draws `(R_F + R_P)·t = 192` words in Poseidon layout (3 per round, partial rounds included); Poseidon2 draws `R_F·t + R_P = 80` and uses one word per internal round (`poseidon2_rust_params.sage:176-192`). So the Logos constants are exactly the first 80 words of the stream the spec's script prints, re-laid out as `[0:12] → external rounds 0-3`, `[12:68] → internal rounds 0-55`, `[68:80] → external rounds 4-7`. Verified: `hades_default[0:80] == circom_stream` for both hadeshash runs. hadeshash's own round-number search at `208b5a1` also returns `R_F = 8, R_P = 56` for these inputs, so no override is needed to reach the seed. What the spec's sentence omits is (a) that the cited script prints 192 constants of which the first 80 are used, and (b) the six seed parameters; see S-001.

**3. Round numbers.** Poseidon2 Table 1 lists `(n, t, d) = (256, 3, 5) → R_F = 8, R_P = 56`. The HorizenLabs search (which includes the Gröbner bound added after ePrint 2023/537) returns `(R_F, R_P) = (6, 52)` without margin and `(8, 56)` with the paper's margin of `+2` external rounds and `+7.5 %` internal rounds (`ceil(52 · 1.075) = 56`), matching 2023/323 §3.2 ("The security level consists of 2 external/full rounds and 7.5% more internal/partial rounds"). The spec's "8 external rounds and 56 internal rounds" is therefore the paper's 128-bit instance including its margin. jellyfish's `sanity_check` (`lib.rs:67-80`) checks only shapes, not the round-number bound; its own `TODO` says so. That is acceptable because the parameters are a fixed constant, not user input.

**4. Matrices.** `poseidon2_perm.circom:144-146` and `:155-157` apply `circ(2, 1, 1)` before the first round and in every external round; `:89-91` apply `[[2,1,1],[1,2,1],[1,1,3]]` in internal rounds. jellyfish hard-codes the same two matrices for `T = 3` (`external.rs:50-62`, `internal.rs:27-38`) and its `MAT_DIAG3_M_1 = [1, 1, 2]` is `diag(M_I) − 1`, consistent. These are the choices of 2023/323 §5.2 and §6 "Case t ∈ {2, 3}" (`M_I = ones + diag(μ)` with small `μ_i ∈ {2, …, p/4}`, `M_E` MDS), both MDS by the paper's `t = 3` conditions (`μ0μ1μ2 − μ0 − μ1 − μ2 + 2 ≠ 0`, `μiμj ≠ 1`: `4` and `7` respectively), and they are what `generate_matrix_full` and `generate_matrix_partial` in the HorizenLabs script hard-code for `t = 3` (`poseidon2_rust_params.sage:414-415,443-444`). The paper permits `M_E = M_I` for `t = 3`; the implementation's `M_E = circ(2,1,1) ≠ M_I` is also permitted (the paper's §5.1 `t = 4t′` matrix does not apply at `t = 3`) and is the reference authors' own instance. The script's subspace-trail checks (`algorithm_1..3`) on `M_I` passed during the run.

**5. All copies agree, and the test values close the loop.** The five copies (circuit, jellyfish, three Python generators) are identical. The 80 regenerated constants, fed to an independent permutation, reproduce all twelve Annex values of `common-cryptographic-components.md` (hash mode with the `10*` padding, compression mode `[a, b, 0] → state[0]`), which is what `zk/poseidon2/src/hasher.rs:24-47` computes and `hasher.rs:102-180` asserts against the same twelve values.

## 5. Suggestions (non-security)

### S-001 · Make the spec's derivation sentence exact and pin the generator

`common-cryptographic-components.md` §Poseidon2 says the constants "are derived following the original Poseidon paper following their implementation" and links `generate_params_poseidon.sage`. That is true of the generator but not of the script's output layout, which is why #20 read it as unreproducible. Proposed wording, to raise in logos-lips:

> The round constants are the first `R_F·t + R_P = 80` outputs of the Grain LFSR of the Poseidon paper (hadeshash `code/generate_params_poseidon.sage` @ `208b5a1`, invoked with `field = 1, s_box = 0, n = 254, t = 3, alpha = 5, M = 128` and the BN254 scalar modulus, which selects `R_F = 8, R_P = 56`), assigned in stream order to external rounds 0-3 (3 each), internal rounds 0-55 (1 each) and external rounds 4-7 (3 each), exactly as `poseidon2_rust_params.sage` in HorizenLabs/poseidon2 @ `055bde3` produces them. `M_E = circ(2,1,1)`, `M_I = [[2,1,1],[1,2,1],[1,1,3]]` (Poseidon2 §5).

Also fix the parameter list in the same section: with `t = 3` and rate 1 the capacity is 2, not 3 ("The capacity = 3" is a typo for the state width).

### S-002 · CI check that regenerates the constants (issue item 5, #20 S-002)

Add the dependency-free script of Appendix B to the circuits repository as `hash_bn/check_poseidon2_constants.py` and run it in `ci.yml` before `generate-proving-keys` (one `python3 hash_bn/check_poseidon2_constants.py` step; ~1 s, no Sage or Docker). It regenerates the 80 constants from the six seed parameters and diffs them against `poseidon2_perm.circom`; a one-digit change in any constant fails it (tested). Extend it to the three Python generators, or make them import the constants from one file, so the copies at `generate_inputs_for_*.py:26-27,94-95` cannot drift. On the node side, jellyfish is a pinned rev and its constants are already tied to the spec by the twelve Annex values asserted in `zk/poseidon2/src/hasher.rs:102-180` (`test_hashes`, `test_compression`); no extra check is needed there beyond keeping the rev pinned.

---

## Appendix A — Definitions

Ratings follow `docs/REPORT_TEMPLATE.md` Appendix A (no findings were rated in this report).

## Appendix B — Regeneration script

Reproduces the 80 constants from the seed parameters alone (verified against the circuit; a one-digit change in the circuit fails it).

```python
#!/usr/bin/env python3
"""Regenerate the Poseidon2 BN254 t=3 round constants with the Grain LFSR of the
Poseidon paper (hadeshash generate_params_poseidon.sage, HorizenLabs
poseidon2_rust_params.sage) and diff them against hash_bn/poseidon2_perm.circom."""
import re, sys, pathlib

P   = 0x30644e72e131a029b85045b68181585d2833e84879b9709143e1f593f0000001
N, T, R_F, R_P = 254, 3, 8, 56          # field bits, state size, external rounds, internal rounds
FIELD, SBOX = 1, 0                       # GF(p), x^alpha

def grain():
    bits = [int(b) for b in (f"{FIELD:02b}{SBOX:04b}{N:012b}{T:012b}{R_F:010b}{R_P:010b}" + "1" * 30)]
    def step():
        b = bits[62] ^ bits[51] ^ bits[38] ^ bits[23] ^ bits[13] ^ bits[0]
        bits.pop(0); bits.append(b); return b
    for _ in range(160): step()
    while True:
        while step() == 0: step()        # discard one bit after every 0 control bit
        yield step()

def constants():
    g = grain(); out = []
    while len(out) < R_F * T + R_P:
        v = int("".join(str(next(g)) for _ in range(N)), 2)
        if v < P: out.append(v)          # rejection sampling, as in both Sage scripts
    return out

def circom():
    src = (pathlib.Path(__file__).parent / "poseidon2_perm.circom").read_text()
    def block(name):
        i = src.index(name); return [int(h, 16) for h in re.findall(r"0x([0-9a-f]{64})", src[i:src.index("];", i)])]
    ext, itn = block("round_consts[8][3]"), block("round_consts[56]")
    return ext[:12] + itn + ext[12:]     # stream order: 4 external rounds, 56 internal, 4 external

if __name__ == "__main__":
    gen, src = constants(), circom()
    if gen != src:
        bad = [i for i, (a, b) in enumerate(zip(gen, src)) if a != b]
        sys.exit(f"MISMATCH at stream positions {bad[:5]}{'...' if len(bad) > 5 else ''} (of {len(src)})")
    print(f"OK: {len(src)} Poseidon2 round constants match the Grain LFSR stream (n={N}, t={T}, R_F={R_F}, R_P={R_P})")
```
