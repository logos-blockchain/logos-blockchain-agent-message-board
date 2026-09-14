# Audit Report — ZK circuits: constraint soundness and completeness

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/20`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `zk/poseidon2`, `zk/proofs/{pol,poq,poc,zksign}`, `core/src/proofs`, `core/src/mantle/ledger.rs`, `blend/proofs/src/quota` (node side, for parity only)
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `cryptarchia-proof-of-leadership.md`, `proof-of-quota.md`, `bedrock-anonymous-leaders-reward.md`, `common-cryptographic-components.md`, `trusted-setup-ceremony.md`; by section: `bedrock-v1.1-mantle-specification.md` (§Note Id, §ZkSignature, §Proof of Claim, §LEADER_CLAIM), `blend-protocol.md` (§Leadership Quota, §Proof of Quota), `proof-of-work.md` (§Overview, §Blend Difficulty), `cryptarchia-v1-protocol.md` (§Epoch)
Date: `2026-09-12` — author: `claude-fable-5.1` — status: `final`

Circuits: `https://github.com/logos-blockchain/logos-blockchain-circuits` @ `ebf7ddf5b5b625ef4ea7c803171c670beebd05ef` (tag `v0.5.7`, the tag pinned by `Cargo.toml:172-176` and `flake.nix:16`); `circomlib` submodule @ `35e54ea21da3e8762557234298dbb553c175ea8d`; compiler `circom 2.2.2` (the version pinned in `justfile:6`). Verification-key hashes (sha256 of `verification_key.json` in release bundle `logos-blockchain-circuits-v0.5.7-linux-aarch64.tar.gz`, which is what `lbc-*-sys` embed): PoL `973513d9a15e0641fa8d153ef678713f5f3dbb9883cae2320832d469b01d44a5`, PoQ `a6898d3eb5353d1333c04f51c365edb669ab1f18c68bd9d9109fc7187bebe10f`, PoC `94457cb5317dba09f96df09e69aa0be19a38c07d69a17cec8b0be6549606dae3`, ZkSignature `b6f8a7861308f0df1a7ead87f4d8158f59104738bb24a24b18904f0bbbbe0df9`.

---

## 1. Summary

- Overall assessment: the four circuits are well constrained and match the Rust node bit-for-bit on every hash construction, but the Proof of Quota leader branch leaves its slot witness completely unbounded, so any holder of an aged note can mint an unlimited number of Blend message keys by grinding slots instead of winning them.
- Findings: 0 critical · 1 high · 0 medium · 0 low · 1 informational
- Key themes: "a private witness that the specification treats as a public fact", "circuit ⇄ node parity is otherwise exact"
- Must-fix before launch: LB-001 (bound `pol_sl` to the epoch inside the PoQ circuit, and add the same bound to `proof-of-quota.md`). This is a circuit change, so it requires a new phase-2 key for PoQ (see PR #148, LB-001 there, for the ceremony gap).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| circuits `hash_bn/poseidon2_perm.circom`, `poseidon2_sponge.circom`, `poseidon2_hash.circom` | Permutation structure, matrices, round constants, sponge padding, compression mode |
| circuits `hash_bn/merkle.circom` | Path/selector convention, root binding, fixed depth |
| circuits `misc/comparator.circom`, `misc/constants.circom` | Full-field comparator, n-bit comparator, domain-separation tags, inverse of 2 |
| circuits `ledger/notes.circom`, `mantle/pol_lib.circom`, `mantle/pol.circom` | Public-key derivation, note id, ticket, threshold, entropy, PoL main |
| circuits `mantle/poc.circom`, `mantle/signature.circom`, `blend/poq.circom` | PoC, ZkSignature, PoQ (three branches) |
| circomlib `bitify.circom`, `comparators.circom`, `aliascheck.circom`, `compconstant.circom` | Only the templates the circuits instantiate (`Num2Bits`, `Num2Bits_strict`, `Bits2Num`, `LessThan`, `IsEqual`, `IsZero`, `AliasCheck`) |
| node `zk/poseidon2/src/hasher.rs`, jellyfish `poseidon2/src/{lib,external,internal}.rs`, `constants/bn254.rs` @ `8d80230` | Parity of parameters with the circuit |
| node `core/src/mantle/ledger.rs:497-531`, `kms/keys/src/keys/zk/private.rs:14,63-65`, `core/src/proofs/leader_proof.rs:260-279`, `core/src/proofs/merkle.rs`, `merkle/utxotree/src/lib.rs`, `mmr/src/lib.rs`, `zk/proofs/*/src/*_inputs.rs` | Node-side derivations the circuits are checked against |
| circuits release bundle `v0.5.7` (`proving_key.zkey`, `verification_key.json`) and `ptau/` | Provenance of the embedded keys |

**Out of scope**

- Public-signal ordering, proof deserialisation, verify-before-work, prover FFI, artefact download integrity and the phase-2 ceremony: covered by PR #148 (issue #64) and not repeated.
- Groth16 itself, `rust-rapidsnark`, `arkworks`, `snarkjs`, `circom`'s code generator, and the C++ witness generator: assumed correct.
- The cryptanalytic strength of Poseidon2 with the chosen parameters (8 external / 56 internal rounds over BN254): assumed as stated in `common-cryptographic-components.md`.
- Independent regeneration of the Poseidon2 round constants from a parameter-generation script (see S-002).
- Node-side use of the proofs beyond input construction (ledger validation, blend nullifier caches, epoch handling).

**Assumptions**

- The trusted setup is honest (the ceremony gap is PR #148 LB-001).
- The specifications at the recorded logos-lips commit are the reference; where two specs disagree, the report says which one the circuit should follow.
- The ledger enforces `0 < value ≤ 2^64-1` on every note it inserts (`bedrock-v1.1-mantle-specification.md` §Output Notes Validation), so a note id present in the ledger tree binds a well-formed value.

## 3. Method

- Manual review of every `.circom` file in the circuits repository (1037 lines outside circomlib) and of the circomlib templates they instantiate, working through the five checklist items of #20. The circuit-side items of #83 (Poseidon2 leaf/node encoding, sibling order, selector order, empty-leaf value, depth 32) were verified in passing and are recorded below.
- Spec conformance: every hash construction, domain-separation tag, comparison and Merkle convention was checked against `common-cryptographic-components.md`, `cryptarchia-proof-of-leadership.md`, `proof-of-quota.md`, `bedrock-anonymous-leaders-reward.md`, `bedrock-v1.1-mantle-specification.md` and `blend-protocol.md`, and against the node's Rust derivations.
- Tooling: `circom 2.2.2` built from `iden3/circom` tag `v2.2.2` (`--r1cs --O2 --inspect`); `snarkjs 0.7.5` (`wtns calculate`, `zkey export verificationkey`, `zkey verify`); `cargo test --release` on `logos-blockchain-{poseidon2,pol,poq,poc,zksign}` at the target commit with the `v0.5.7` prebuilt artefacts; Python 3 for the constant and test-vector comparisons.
- Dynamic testing:
  - A test circuit instantiating `Poseidon2_hash(1..4)` and `Compression()` was compiled and its witness computed for all twelve test values in `common-cryptographic-components.md` §Annex: every output matches (hash mode `[0]`, `[1]`, `[0,0]`, `[1,2]`, `[2,1]`, `[1,0,0]`, `[0,0,1]`, `[0,1,0,1]`; compression `[0,0]`, `[1,0]`, `[0,1]`, `[1,1]`). The node's `hasher.rs:103-180` tests carry the same twelve values in decimal and pass.
  - The node's full prove/verify round-trips (`test_full_flow` for PoL, PoC, ZkSignature; `test_core_node_full_flow`, `test_leader_full_flow`, `test_pow_full_flow` for PoQ) all pass at the target commit, which is a functional proof that note id, ticket, entropy, KDF, selection randomness, key nullifier, voucher commitment/nullifier and both Merkle conventions agree between the node and the circuits.
  - A slot-grinding test against the real PoQ prover and verifier (LB-001).
  - `snarkjs zkey verify` of each bundled `proving_key.zkey` against the `.r1cs` compiled from the `v0.5.7` sources and the vendored `powersOfTau28_hez_final_17.ptau` (sha256 `6b662a32…83ae0`, equal to the pin in `ptau/README.md` and `ci.yml`); and `zkey export verificationkey` compared byte-for-byte with the bundled `verification_key.json`. Results in "Checked and ruled out".

### Checked and ruled out

**Under-constrained signals.** No `<--` or `-->` occurs outside circomlib; every intermediate in the four circuits is assigned with `<==`. Public inputs that no constraint would otherwise touch are pinned with a quadratic dummy (`P_lead_part_one/two` at `pol_lib.circom:175-176`, `K_part_one/two` at `poq.circom:30-33`, `mantle_tx_hash` at `poc.circom:71`, `msg` at `signature.circom:21`); `circom --O2` reports the expected public-input counts (PoL 8+1 output, PoQ 11+1, PoC 2+1, ZkSignature 1+32), so none is optimised away. `--inspect` warns only about unused permutation lanes (`perm.out[1..2]` in `Compression`), unused bit arrays inside `LessThan`/`CompConstant`, and the unused `would_win.note_identifier` output in PoQ; none of these is a soundness gap. Constraint counts: PoL 20460, PoQ 20110, PoC 8257, ZkSignature 7681 non-linear, 0 linear. Membership templates return a 0/1 signal rather than asserting, and every caller asserts it: `pol_lib.circom:196` (`lottery_checker.out * unspent_membership.out === 1`, with `out` itself the product of membership and win at `:120`), `poc.circom:59`, `poq.circom:134` (branch equation). `IsZero`/`IsEqual` are the sound circomlib forms (`in*out === 0`, `out = 1 - in*inv`).

**Range checks and aliasing.** All 84 Merkle selectors are constrained boolean before use (`pol_lib.circom:86-88`, `:184-186`; `poc.circom:47-49`; `poq.circom:94`), as the `/!\` comment in `merkle.circom:8,33` requires. `SafeFullLessThan` (`comparator.circom:10-58`) decomposes both operands with `Num2Bits_strict` (254 bits plus `AliasCheck`, so the decomposition is the canonical representative `< p`), compares the top 252 bits with circomlib `LessThan(252)` (inputs `< 2^252`, so no wrap) and the two low bits by hand; the truth table was checked for all cases of `a_hi ? b_hi` and the four low-bit combinations and is correct. It is used for `ticket < threshold` (`pol_lib.circom:115-117`) and `pow_ticket < pow_blend_difficulty` (`poq.circom:126-128`), both full-field comparisons as `proof-of-work.md` §Puzzle Target and `proof-of-quota.md` §Step 4 require. `SafeLessThan(20)` (`comparator.circom:64-79`) range-checks both `index` and the selected quota to 20 bits before `LessThan(20)` (`poq.circom:80-83`), matching the 20-bit widths in `proof-of-quota.md` §Public values and §Witness; the node types both as `CircuitInteger<20>` (`zk/proofs/poq/src/lib.rs:35-42`), so honest provers cannot hit the over-constraint. The PoQ selector is constrained to `{0,1,2}` by `(s²−s)(s−2) = 0` (`poq.circom:67`) and the Lagrange terms `L1 = −s²+2s`, `L2 = (s²−s)/2` (`:72-73`, `INV_2` verified equal to `2⁻¹ mod p`) evaluate to the intended one-hot for all three values, so the quota selection (`:81`), branch predicate (`:134`), key selection (`:142-143`) and period selection (`:146`) are exact. Note value and output number carry no range check in the circuit; they need none, because both are bound into the note id (`pol_lib.circom:74-80`) that must sit in the ledger tree, and the ledger only inserts values in `(0, 2^64−1]`.

**Nullifier and commitment binding.** PoQ `key_nullifier = compress(KEY_NULLIFIER_V1, hash(SELECTION_RANDOMNESS_V1, sk, index, period))` (`poq.circom:137-153`) binds the branch secret (`core_sk`, the note's `pol_secret_key`, or `pow_nonce`), the index and the period (`pol_epoch_nonce`, or `pol_sl` for the leader branch), exactly as `proof-of-quota.md` §Step 5. PoC `reward_voucher = compress(REWARD_VOUCHER, secret)` and `voucher_nullifier = compress(VOUCHER_NF, secret)` (`poc.circom:8-30`) match `bedrock-anonymous-leaders-reward.md` §Voucher creation and §Preventing Double Claims, and both use compression mode as `common-cryptographic-components.md` prescribes for "nullifier derivation and reward voucher derivation". The note id `hash(NOTE_ID_V1, tx_hash, output_number, value, pk)` (`pol_lib.circom:74-80`) binds every note field and the owner key, in the order of `bedrock-v1.1-mantle-specification.md` §Note Id, and `core/src/mantle/ledger.rs:512-531` computes the same five-element digest. `pk = compress(KDF, sk)` (`notes.circom:8-17`) matches `kms/keys/src/keys/zk/private.rs:63-65`. Opening a compression or hash two ways reduces to a collision in a 254-bit-capacity sponge, which is the assumed security of the primitive. The DST integers in `constants.circom` were recomputed with `int.from_bytes(tag, "little")` for all eight tags and are correct.

**Poseidon2 parameters.** The circuit permutation (`poseidon2_perm.circom`) is: initial `circ(2,1,1)` layer (`:152-158`), 4 external rounds (full S-box `x⁵`, `circ(2,1,1)`), 56 internal rounds (S-box on lane 0, matrix `[[2,1,1],[1,2,1],[1,1,3]]` at `:89-91`), 4 external rounds. jellyfish `permute_mut` (`lib.rs:98-125`) applies `matmul_external` first, then `EXT_ROUNDS/2 = 4`, then 56 internal with `MAT_DIAG3_M_1 = [1,1,2]` (so `M_I = I + diag(1,1,2) + J`, the same matrix), then 4 external. All 24 external and 56 internal round constants are identical between `poseidon2_perm.circom:28-84,102-136` and jellyfish `constants/bn254.rs:29-135` (compared programmatically). Sponge: state `[0,0,0]`, rate 1, capacity 2, `10*` padding with a single `1` (`poseidon2_sponge.circom:47-52`), output `state[0]`; the node's `Poseidon2Hasher::update`/`finalize` (`hasher.rs:24-36,48-50,74-77`) is the same absorb-one-permute-per-element with a trailing `1`. Compression: `perm([a, b, 0])[0]` (`poseidon2_perm.circom:204-214`) equals `hasher.rs:40-46`. The twelve spec test values match both implementations (Method). Domain separation between uses: every hash-mode use carries a distinct DST as first element and a distinct arity (note id 5, ticket 5, entropy 4, selection randomness 4), compression uses carry a DST (`KDF`, `KEY_NULLIFIER_V1`, `REWARD_VOUCHER`, `VOUCHER_NF`) except Merkle nodes, and the blend PoW ticket `hash(pow_nonce, pol_epoch_nonce)` (`poq.circom:123-125`) is the only DST-less sponge call; it has arity 2 and therefore a different padding position from every other call, and the node's reward-side puzzle ticket has arity 3 (`core/src/mantle/ops/pow.rs:97-103`), so no two uses of the same construction share a preimage domain.

**Merkle parity (the circuit half of #83).** `compute_merkle_root` (`merkle.circom:9-28`) consumes `nodes[i]` leaf-to-root and `selector[n-1-i]` root-to-leaf, and selector `1` puts the sibling on the left (`inp[0] = nodes[i]`, `inp[1] = current`); `merkle_path_to_witness` (`core/src/proofs/merkle.rs:6-21`) emits selectors reversed and `true` for `MerkleNode::Left`, and `mmr_path_to_witness` (`:25-56`) emits `true` when the leaf is a right child, both of which mean "sibling on the left". Inner nodes are `compress([left, right])` in the circuit and in `merkle/utxotree/src/lib.rs:31-32` and `mmr/src/lib.rs:231-255`; the leaf is the raw `NoteId` with no tag (`utxotree/src/lib.rs:52-54`, `pol_lib.circom:95`) and the raw voucher commitment (`poc.circom:56`); the empty leaf is `0` (`utxotree/src/lib.rs:29`, `cryptarchia-proof-of-leadership.md` §Ledger Root). Depths are hard-coded to 32/32/32/20 in both the circuits and `zk/proofs/{pol,poc,poq}/src/*_inputs.rs`. The root is re-derived over all levels and compared with the public root (`merkle.circom:49-53`), never accepted from the witness. Fixed depth also rules out the leaf/inner-node confusion that an untagged tree would otherwise allow. The passing round-trip tests confirm the conventions agree in practice.

**Completeness (over-constraint).** No honest input was found that a circuit rejects. Boundary values: `value = 2^64−1` and `t1 = p − ⌊c/S²⌋` (the "negative" constant of `cryptarchia-proof-of-leadership.md` §Lottery Approximation, computed that way in `zk/proofs/pol/src/lottery.rs:131-140`) give a threshold that is simply a field element, and `Num2Bits_strict` accepts any element `< p`; the "note value ≫ inferred stake" wrap is the documented corner case in the spec, not a circuit defect. Quotas and index at `2^20−1` pass `SafeLessThan(20)`. Selectors for the unselected PoQ branches are filled with `0` by the node and satisfy the boolean constraints. ZkSignature pads to 32 keys with `sk = 0` (`kms/keys/src/keys/zk/private.rs:89-99`) and rejects more than 32, as the spec's padding rule requires. Public inputs are supplied by the verifier, so no range constraint on them can make an honest proof fail.

**Key provenance.** `snarkjs zkey verify` accepts every bundled `proving_key.zkey` against the `.r1cs` compiled here from the `v0.5.7` sources with the pinned ptau, and `zkey export verificationkey` of each `.zkey` is byte-identical to the bundled `verification_key.json` (hashes in the header). The embedded keys therefore correspond to the reviewed sources at `ebf7ddf5`. Each `.zkey` records exactly one phase-2 contribution, named `RELEASE`, confirming PR #148 LB-001.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | PoQ leader branch: the slot is a private witness with no bound, so any aged note yields unlimited leader quota by grinding slots | ZK Soundness / Denial of Service | High | Low | Open |
| LB-002 | The two 128-bit halves of the one-time signing key are not range-constrained in PoL or PoQ | ZK Soundness | Informational | High | Open |

### LB-001 · PoQ leader branch: the slot is a private witness with no bound, so any aged note yields unlimited leader quota by grinding slots

| | |
|---|---|
| Severity | High |
| Difficulty | Low |
| Category | ZK Soundness / Denial of Service |
| Target | circuits `blend/poq.circom:47` (`signal input pol_sl`), `:107` (`would_win.slot <== pol_sl`), `:146` (nullifier period), `:158` (`main`, `pol_sl` not public); `mantle/pol_lib.circom:12-27` (`ticket_calculator`), `:46-122` (`would_win_leadership`) |
| Status | Open |

**Description**

`blend-protocol.md` §Leadership Quota defines the leadership quota of a node as `Q_L^n = x · Q_L` where `x` is "the exact number of leader elections won by the node `n` in an epoch" (`blend-protocol.md:665-671`), and §Proof of Quota says the leader index is bounded "per won slot" (`:739`). The PoQ circuit is what makes that bound real, since the verifier cannot see the slot. In the leader branch the slot is the private input `pol_sl` (`poq.circom:47`). It is used in exactly two places: as the `slot` element of the lottery ticket `hash(LEAD_V1, epoch_nonce, slot, note_id, sk)` (`:107` → `pol_lib.circom:12-27`), and as the nullifier period `pol_epoch_nonce + (pol_sl − pol_epoch_nonce)·L1` (`:146`). Nothing relates it to the epoch of `pol_epoch_nonce`, to the current slot, or even to a 64-bit range. The only epoch binding is through the ticket, which mixes in `pol_epoch_nonce`; but for a fixed epoch nonce the ticket is a fresh pseudo-random field element for every value of `pol_sl`, so the prover simply searches `pol_sl` over the whole field until `ticket < threshold`. Each hit satisfies `would_win.out = 1` and produces a nullifier `compress(KEY_NULLIFIER_V1, hash(SELECTION_RANDOMNESS_V1, sk, index, pol_sl))` that differs from every other hit's, with the index free to run over `0 .. leader_quota−1` again for each of them.

`proof-of-quota.md` has the same gap: `pol_sl` is a private witness (`:92`) and the pseudocode (`:172-203`) never constrains it. `cryptarchia-proof-of-leadership.md` avoids the problem because PoL makes `sl` public and the block header fixes it. The spec side that is wrong is `proof-of-quota.md`, which drops the public slot when it reuses the PoL relation; `blend-protocol.md` states the intended semantics.

The node's honest prover chooses real winning slots from a stream computed by the chain service (`services/blend/src/epoch_info.rs:14-21`, `blend/proofs/src/quota/inputs/prove/private.rs:84,137`, `zk/proofs/poq/src/wallet_inputs.rs:19,86`), so honest traffic is unaffected; a modified prover is a one-line change to that `u64`.

**Exploit scenario**

The attacker holds one note that is present in `pol_ledger_aged` for the current epoch (any aged note, spent or not, since the PoQ leader branch deliberately checks only the aged root). The win probability per slot is `≈ −ln(1−f)·v/S` (`cryptarchia-proof-of-leadership.md` §Lottery Approximation); with `f = 1/30`, a note holding 0.01 % of the inferred stake wins one slot in about 300 000 candidates, so one 5-element Poseidon2 sponge evaluation at a time (six permutations, tens of microseconds in Rust) yields a new winning `pol_sl` every few seconds on one core, whereas the same note honestly expects about `648000 · 0.034 · 10⁻⁴ ≈ 2` wins per epoch. Every ground slot gives the attacker `leader_quota` fresh, verifier-accepted key nullifiers, so the total quota is bounded by hashing power, not by stake. The keys carry valid PoQs, so blend core nodes relay the messages and do not drop them as duplicates; the attacker can saturate the Blend network's data-message budget, dilute cover traffic, and force every core node to verify one Groth16 proof per message, at a cost to themselves of one note and a laptop.

Reproduced against the real prover and verifier at the target commit by adding a test next to `test_leader_full_flow` (`zk/proofs/poq/src/lib.rs:317-502`) that reuses its fixture (note value 50 of an inferred stake of 5000, `f = 1/10`), recomputes the note id and threshold as the node does, and grinds `pol_sl` downwards from `u64::MAX`:

```rust
let wins = |slot: u64| {
    let t = Poseidon2Bn254Hasher::digest(&[lead, chain_data.pol_epoch_nonce, F::from(slot), note_id, sk]);
    BigUint::from(t) < thr
};
let mut s = u64::MAX;
while found.len() < 3 { if wins(s) { found.push(s); } s -= 1; }
for slot in found {
    let wd = PoQWalletInputsData { slot, ..same note, same path, same key.. };
    let mut cd = common_data; cd.index = KeyIndex::new::<0>();
    let (proof, inputs) = prove(PoQWitnessInputs::from_leader_data(chain_data, cd, wd)).unwrap();
    assert!(verify(&proof, inputs).unwrap());
}
```

Output (the full patch is 275 lines and adds only `lb-poseidon2` as a dev-dependency):

```text
grinding: 3 winning slots after 5249 hashes: [18446744073709549594, 18446744073709547746, 18446744073709546367]
slot 18446744073709549594: accepted, nullifier 16613966071293272684843536501686122450669310695879688913276810311818284632510
slot 18446744073709547746: accepted, nullifier 11706992131724859022301632359797189642352951037211151069241548109698473753418
slot 18446744073709546367: accepted, nullifier 16922427591506393024432357028141267592445730851857386473471115074090133184673
test tests::audit_poq_leader_slot_is_unbounded ... ok
```

Three slots that lie about 1.8 × 10¹⁹ slots in the future, the same note, the same index `0`, three distinct nullifiers, three accepted proofs, in under five seconds including proving.

**Recommendation**

- *Short term*: add two public inputs, `pol_epoch_first_slot` and `pol_epoch_slot_count` (or the last slot), taken from the same epoch state that supplies `pol_epoch_nonce`, and constrain inside the leader branch `SafeLessThan(64)(pol_sl − pol_epoch_first_slot, pol_epoch_slot_count)` with the result multiplied into the `L1` term of the branch equation (`poq.circom:134`), so core and PoW provers, who fill `pol_sl` with `0`, are unaffected. Update `proof-of-quota.md` §Public values and the pseudocode accordingly, and add the slot range to `PoQChainInputsData` and the verifier inputs on the node side. Re-run phase 2 for PoQ after the change.
- *Long term*: for every private witness that a spec elsewhere treats as a bounded quantity (slot, index, output number), the circuit should either range-check it or bind it to a public input; a table of witnesses with their intended domain, kept in the circuits repository and checked by a test that grinds each unconstrained one, would have caught this. Consider also making the verifier's nullifier cache reject more than `leader_quota × (expected wins + margin)` leader-shaped nullifiers per epoch as a defence in depth, although the branch is hidden and this can only cap the aggregate.

**References**: `blend-protocol.md` §Leadership Quota (`:636-671`), §Proof of Quota (`:735-745`); `proof-of-quota.md` §Witness (`:84-103`), §Step 3, §Pseudocode (`:172-203`); `cryptarchia-proof-of-leadership.md` §Circuit Public Inputs item 1, §Lottery Approximation; PR #148 LB-001 (phase-2 ceremony) for the re-keying that a fix implies.

### LB-002 · The two 128-bit halves of the one-time signing key are not range-constrained in PoL or PoQ

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | ZK Soundness |
| Target | circuits `mantle/pol_lib.circom:168-176` (`P_lead_part_one/two`), `blend/poq.circom:24-33` (`K_part_one/two`) |
| Status | Open |

**Description**

`cryptarchia-proof-of-leadership.md` §Circuit Public Inputs item 6 and `proof-of-quota.md` §Step 6 define the key as "two public inputs, each of 16 bytes in little endian". The circuits pin the two inputs only with `dummy <== part * part`, so any field element is accepted for each half. This is not exploitable today: both inputs are public, the verifier derives them from the 32 Ed25519 key bytes itself (`core/src/proofs/leader_proof.rs:360-367`, `blend/proofs/src/quota/inputs/mod.rs:9-21`, checked in PR #148), and a 16-byte little-endian integer is always `< 2^128 < p`, so the encoding is injective. The binding is therefore exactly as strong as the verifier-side encoding and no stronger; a future change that lets a prover influence the encoding, or that reduces a longer value mod `p`, would not be caught by the circuit.

**Exploit scenario**

None at this commit. Recorded so that the property "the key halves are 128-bit" is known to live in the Rust encoding, not in the constraint system.

**Recommendation**

- *Short term*: none required.
- *Long term*: replace the dummy squares with `Num2Bits(128)` on each half (256 extra constraints per circuit), which documents the intended domain in the circuit and makes the public-input encoding self-checking.

**References**: `cryptarchia-proof-of-leadership.md` §Linking the Proof of Leadership to a Block; PR #148 "Public-input ordering".

## 5. Suggestions (non-security)

### S-001 · `common-cryptographic-components.md` states capacity 3 for a width-3 permutation

`common-cryptographic-components.md:143-144` lists "The rate = 1. The capacity = 3." for Poseidon2 over BN254. The permutation has width `t = 3`, so rate 1 implies capacity 2, which is what both the circuit (`PoseidonSponge(3, 2, n, 1)`, `poseidon2_hash.circom:14`) and the node (state `[Fr; 3]`, one lane absorbed) implement. The spec should say capacity 2. Raise upstream in logos-lips.

### S-002 · Poseidon2 round constants are not reproducible from the script the spec cites

The spec says the round constants "are derived following the original Poseidon paper following their implementation" (`generate_params_poseidon.sage`). The constants actually in use are, per the jellyfish source comment (`constants/bn254.rs:19`), "adapted from HorizenLabs/poseidon2 … poseidon2_instance_bn256.rs", which that project generated with its own `poseidon2_rust_params.sage`. This report verified that the circuit and the node use the same 80 constants and that they reproduce the spec's test values, but did not regenerate them. Pin the generator script, its parameters (field, `t = 3`, `d = 5`, `R_F = 8`, `R_P = 56`) and the expected output in the circuits repository, with a CI job that regenerates and diffs, and cite that script in the spec instead of the Poseidon-1 one.

### S-003 · The PoQ leader nullifier binds the note owner's key, not the note

`key_nullifier` in the leader branch is derived from `pol_secret_key`, `index` and `pol_sl` (`poq.circom:142-146`, as `proof-of-quota.md` §Step 5 specifies). A leader who holds several aged notes under one ZK key and whose notes win the same slot obtains one quota for that slot, not one per note, because the nullifiers coincide; and, conversely, a nullifier cannot be attributed to a note even by its owner. Neither direction is a security problem (the loss falls only on the owner and is astronomically unlikely), but if the intent is "one quota per winning note per slot", `note_id` should be added to the selection-randomness preimage. This should be decided in the spec first.

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
