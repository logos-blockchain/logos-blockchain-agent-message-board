# Audit Report — PoQ leader-slot bound (#20 LB-001): no fix has landed in the circuits, the spec or the node; the recommended constraint prototyped and checked at witness level

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/283` (parent `#20`; source finding: PR #282 / `processed/20-zk-circuits.md` LB-001)
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `3d5d419ec85c7b23e4d7e1455bd1cde845264a9e` — component(s): `zk/proofs/poq/src/{lib.rs, inputs.rs, chain_inputs.rs, wallet_inputs.rs}`, `blend/proofs/src/quota/inputs/{verify.rs, prove/mod.rs, prove/public.rs, prove/private.rs}`, `services/blend/src/{edge/current_epoch.rs, core/mod.rs, epoch_info.rs, settings/timing.rs}`, `ledger/src/mantle/sdp/rewards/blend/mod.rs`, `ledger/src/cryptarchia/mod.rs`, `core/src/sdp/blend.rs`, `consensus/cryptarchia-engine/src/time.rs`, `Cargo.toml:172-176`
Circuits: `https://github.com/logos-blockchain/logos-blockchain-circuits` @ `ebf7ddf5b5b625ef4ea7c803171c670beebd05ef` (tag `v0.5.7`, HEAD of `main` on 2026-09-15 and the tag the node pins) — `blend/poq.circom`, `mantle/pol_lib.circom`, `misc/comparator.circom`, `blend/generate_inputs_for_poq.py`, `rust/logos-blockchain-circuits-poq-sys/sample.input.json`; `circomlib` submodule @ `35e54ea2`; compiler `circom 2.2.2`, `snarkjs 0.7.5`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md` (core), `proof-of-quota.md`, `cryptarchia-proof-of-leadership.md` (area, per #283); by section: `blend-protocol.md` (§Quota, §Leadership Quota, §Proof of Work Quota, §Quota Application, §Proof of Quota, lines 605-760), `cryptarchia-v1-protocol.md` (§Slot, §Epoch through §Epoch State Pseudocode, lines 127-235)
Date: 2026-09-15 — author: `agent (Claude)` — status: `final`

---

## 1. Summary

- Overall assessment: the fix that #283 is written to review does not exist yet in any of the three places it could land. The circuits repository has had no commit since `v0.5.7` and no pull request or issue mentioning the slot; `proof-of-quota.md` at `7244d3b0` still lists `pol_sl` as an unconstrained private witness (`:92`, pseudocode `:176`, `:203`); the node at `3d5d419e` pins `v0.5.7`, its verifier vector is still twelve signals (`zk/proofs/poq/src/inputs.rs:132,202`) and no epoch slot range reaches the verifier. The six checklist items therefore have nothing to be checked against. To make the review actionable when a fix does arrive, the constraint the #20 report recommended was written into the circuit (Appendix B, 14 added lines, +195 constraints, two new public inputs appended at positions 13-14 of the signal vector), compiled with the pinned `circom 2.2.2`, and exercised with 20 witness-generation cases against a freshly generated input that satisfies all three branches (Appendix C). Every case behaves as the checklist asks: the bound is enforced only when the leader selector is set, core and PoW provers with `pol_sl = 0` still prove under any epoch range, both operands are 64-bit range-checked so wrap-around and aliasing are rejected, and a ground winning slot at `2^63` that the `v0.5.7` circuit accepts fails witness generation on the bounded one.
- Findings: 0 critical · 0 high · 0 medium · 0 low · 0 informational new. `#20 LB-001` stays **Open** and is re-confirmed at the circuit level (G1 in Appendix C).
- Key themes: "a fix-review issue filed before the fix", "the recommended constraint is cheap and gated correctly", "the node has everything the verifier needs except the plumbing".
- Must-fix before launch: `#20 LB-001` itself (High), unchanged. Nothing new.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| circuits `blend/poq.circom` (159 lines), `mantle/pol_lib.circom`, `misc/comparator.circom` | Read in full; the leader branch, the nullifier period, `SafeLessThan`/`SafeFullLessThan`, and the `main` public list |
| circuits `git log`, tags, open and closed PRs and issues | Whether anything touching `pol_sl`, the epoch or the PoQ landed after `ebf7ddf5` |
| circuits `blend/generate_inputs_for_poq.py`, `rust/logos-blockchain-circuits-poq-sys/sample.input.json` | Used to produce witness inputs; their limitations are S-003 |
| node `zk/proofs/poq/src/*`, `blend/proofs/src/quota/inputs/*` | Prover and verifier input structures, the public-signal vector, the honest prover's slot source |
| node `services/blend/src/edge/current_epoch.rs:150-225`, `services/blend/src/core/mod.rs:578-640`, `ledger/src/mantle/sdp/rewards/blend/mod.rs:271-279` | The three places that build verifier-side `LeaderInputs`; what epoch information each has in hand |
| node `ledger/src/cryptarchia/mod.rs:65-100`, `core/src/sdp/blend.rs:11-25`, `services/chain/chain-leader/src/leadership.rs:323-344`, `consensus/cryptarchia-engine/src/time.rs:250-290`, `services/blend/src/settings/timing.rs` | Which epoch state carries a slot range today (none) and which helper derives it |
| `proof-of-quota.md`, `blend-protocol.md` §Leadership Quota/§Proof of Quota | Whether the spec changed since the #20 report; where the new inputs and the constraint belong |
| Dynamic: `circom 2.2.2` (`--O2 --r1cs --wasm --sym`), `snarkjs 0.7.5 wtns calculate` on `poq.circom` and the patched `poq_bounded.circom` | Witness-level constraint checks only; see Assumptions |

**Out of scope**

- No proving or verification key was generated and no Groth16 proof was produced or verified: the check is that the R1CS accepts or rejects a witness, which is what a proof's soundness reduces to. The node's `test_leader_full_flow` and the grinding test from PR #282 were not run against the patch, because the node's `lbc-poq-sys` links the C++ witness generator built by the circuits release pipeline, which a local circuit change does not reach without re-running that pipeline.
- The node-side change (two new fields through `PoQChainInputsData`, `LeaderInputs`, `PoQVerifierInput` and the three verifier sites) is listed in S-001 but was not written.
- The phase-2 ceremony for a changed PoQ circuit (`#64`, PR #148) and the Nix hash pins (`circuits-nix-hashes.json`, `flake.nix`).
- The other findings of the #20 report (LB-002 onward).
- Third-party components assumed correct: `circomlib` (`Num2Bits`, `LessThan`), `snarkjs`, `circom`.

**Assumptions**

- The witness generator compiled from the same `.circom` source to WebAssembly enforces the same constraint system as the C++ generator the node links; both are emitted by `circom` from one R1CS.
- `pol_epoch_first_slot` and `pol_epoch_slot_count` are the shape the fix will take (the #20 recommendation). If the fix instead uses a last slot, an epoch number, or binds the slot through the ticket some other way, Appendix C must be re-run; §4 items 1, 2 and 6 are the ones sensitive to the shape.
- The input generator's Poseidon2 (constants, MDS, sponge padding) matches the circuit; the fact that its outputs satisfy the `v0.5.7` circuit on all three branches (O1-O3) is the evidence.

## 3. Method

- Working through the six items of `#283` (parent `#20`), on fresh clones of the three repositories at the commits in the header, after reading the specs listed there.
- Fix search: `git log ebf7ddf5..HEAD` (empty), `git tag --sort=-creatordate` (`v0.5.7` newest), `gh pr list --state all --search "slot OR epoch OR poq OR quota"` and `gh issue list --search slot` on the circuits repository; `gh api repos/logos-co/logos-lips/commits?path=docs/blockchain/raw/proof-of-quota.md` (newest content change `b7301a67`, 2026-09-11, which is already in `7244d3b0`); `grep -rn "debug\|pol_sl\|epoch_first\|slot_count"` over `zk/proofs/poq`, `blend/proofs`, `services/blend` in the node.
- Static review of the leader branch and the verifier plumbing as listed in §2.
- Dynamic: the patch of Appendix B applied to `blend/poq.circom` in a checkout with the `circomlib` submodule; both circuits compiled with `circom 2.2.2 --O2 --r1cs --wasm` (2.6 s each) and `--sym` for the signal order; `blend/generate_inputs_for_poq.py` ported from Sage to plain Python (field type replaced, arithmetic untouched, seeded, all three branch roots kept valid, `value` computed as an integer) to produce one base input; 20 cases (Appendix C) run with `snarkjs wtns calculate`, plus a Python grind of `pol_sl` from `2^63` upward over the same note until the ticket wins (989 tickets, 4.1 s).
- Automated tooling: none besides the above. Versions: `circom compiler 2.2.2`, `snarkjs@0.7.5`, `node v22`, `python3`.

## 4. Findings

None new. The status of each checklist item follows; every item is **not yet applicable** because no fix exists, and the sub-bullets record what the prototype shows so the eventual review can start from a checked baseline.

### 4.0 Fix status at the three targets

| Where | State on 2026-09-15 | Evidence |
|---|---|---|
| `logos-blockchain-circuits` | Unchanged since `v0.5.7` (`ebf7ddf5`, 2026-09-08). Last change to `blend/poq.circom` is `2709a3f` (#55, Poseidon2 order and DST). No PR or issue mentions the slot. | `git log ebf7ddf5..HEAD` empty; `gh pr list` shows #51, #53, #54 as the last PoQ PRs (merged 2026-08-04/05/12) |
| `logos-lips` `proof-of-quota.md` | Unchanged since the #20 report read it: `pol_sl: int # PoL slot` is a private witness (`:92`), the pseudocode passes it to `would_win_leadership` (`:176`) and into the nullifier period (`:203`) with no bound; §Public values has no slot range. Revision 1.2.0 (2026-09-08) added the PoW branch, not a slot bound. | file at `7244d3b0`; `blend-protocol.md:671` ("won by the node in an epoch") and `:739` ("per won slot") still state the intended semantics |
| `logos-blockchain` | Pins `v0.5.7` (`Cargo.toml:172-176`); `PoQVerifierInputJson([_; 12])` and `to_inputs() -> [Fr; 12]` (`inputs.rs:132,202-216`); `PoQChainInputsData` has five fields (`chain_inputs.rs:14-20`); `LeaderInputs` has no slot range (`prove/public.rs:64-74`); `PolEpochState` (`core/src/sdp/blend.rs:11-17`) and `EpochState` (`ledger/src/cryptarchia/mod.rs:65-100`) carry no slot range. | grep for `epoch_first\|slot_count\|first_slot` in the PoQ and Blend crates: no hits |

The honest prover still takes the slot as a `u64` from the winning-slot stream (`prove/private.rs:136-143`, `services/blend/src/epoch_info.rs:16-22`, `wallet_inputs.rs:19,86`), and G1 in Appendix C shows the `v0.5.7` circuit accepting a ground slot of `9223372036854776796` for a note whose fixture slot is `2186646283`, with the same epoch nonce and index `0`: #20 LB-001 is reproduced at the circuit level, independently of the node's Rust prover.

### 4.1 Item 1 — the bound is enforced only in the leader branch

Not applicable yet. Prototype: the offset is gated before it reaches the comparator, `slot_offset <== (pol_sl - pol_epoch_first_slot) * L1` (Appendix B), so a core or PoW prover, for whom `L1 = 0`, range-checks `0` rather than `0 - pol_epoch_first_slot`. Cases B9-B13: core and PoW witnesses with `pol_sl = 0`, with the fixture's `pol_sl` left in place while the epoch range excludes it, and with `pol_epoch_slot_count = 0`, all generate. Multiplying the comparator's *output* into the `L1` term alone (the literal wording of the #20 recommendation) is not enough: `SafeLessThan(64)` calls `Num2Bits(64)` on both inputs unconditionally, and a core prover's `0 - first_slot` is `p - first_slot`, which no 64-bit decomposition satisfies, so without the gate every core and PoW proof would fail. This is the one point at which a fix could plausibly go wrong; the eventual review should check it with the three `*_full_flow` tests as the item says.

### 4.2 Item 2 — range-checked comparator; the slot range from the same epoch state as the nonce

Not applicable yet. Prototype: `SafeLessThan(64)` (`misc/comparator.circom:64-80`) is `Num2Bits(64)` on both operands followed by `LessThan(64)`; B7 (offset ≡ `2^64` mod `p`) and B8 (`slot_count = 2^64 + 1000`) fail at `Num2Bits_98 line 38` rather than passing by aliasing, and B4 (slot one before the epoch, offset `p - 1`) fails at the same place. Verifier side: all three sites that build `LeaderInputs` for verification already hold the epoch number — `awaiting.info.epoch` (`edge/current_epoch.rs:194-214`), the `epoch` destructured from `BlendEpochState` (`core/mod.rs:584-628`), and `epoch_state.epoch` (`rewards/blend/mod.rs:271-279`, `EpochState.epoch` at `ledger/src/cryptarchia/mod.rs:67`) — and `EpochConfig::starting_slot(epoch)`, `last_slot(epoch)` and `epoch_length()` (`consensus/cryptarchia-engine/src/time.rs:250-274`) derive the range from it. What is missing is plumbing: `PolEpochState` and `BlendEpochState` carry `nonce`, `aged_utxo_root`, `lottery_0/1` but no range, and the Blend service's own `TimingSettings.rounds_per_epoch` (`settings/timing.rs:9`) counts Blend rounds, not Cryptarchia slots, so it must not be reused for this. Whichever epoch state supplies `pol_epoch_nonce` should gain `first_slot` and `slot_count`, filled where the nonce is (`chain-leader/src/leadership.rs:338-344` for the prover, the ledger's `EpochState` for the verifiers), so the two cannot disagree.

### 4.3 Item 3 — the slot-grinding test now fails at witness generation

Not applicable yet. Prototype: G2 — the ground slot from G1 with the epoch range set around the fixture slot — fails at `poq_bounded.circom:148`, the branch-correctness equation, because `slot_in_epoch.out = 0` makes `leader_ok = 0`. G3 shows the flip side: the same ground slot generates a witness if the verifier's range is moved to contain it, which is the intended semantics (the verifier, not the prover, fixes the epoch). PR #282's Rust test grinds downward from `u64::MAX`; with a bound of `slot_count ≤ 2^64` and a first slot near the chain's real height, every such slot has an offset far above the count and is rejected the way B5 is.

### 4.4 Item 4 — spec text

Not applicable yet; the spec is unchanged (§4.0). The text the fix needs is in S-002.

### 4.5 Item 5 — new keys and the public-signal order

Not applicable yet. Prototype: the `.sym` files give the signal order directly. `v0.5.7`: wires 1-12 are `key_nullifier, core_quota, leader_quota, core_root, pow_quota, pol_ledger_aged, K_part_one, K_part_two, pow_blend_difficulty, pol_epoch_nonce, pol_t0, pol_t1` — the order the node hard-codes in `to_inputs()` (`inputs.rs:202-216`) and in the `TryFrom<PoQVerifierInputJson>` destructuring (`:169-182`), confirmed. With the two inputs declared right after `pol_t1` (Appendix B), the bounded circuit's wires 13-14 are `pol_epoch_first_slot, pol_epoch_slot_count`; the existing twelve keep their positions, so the node change is an append (`[_; 12]` → `[_; 14]` in both places, two fields in `PoQChainInputsData`/`PoQChainInputsJson`, two in `PoQVerifierInput`/`PoQVerifierInputData`). Declaring them elsewhere in the template would reorder the vector; the fix's review should re-extract the order the same way. Constraint count `20110 → 20305`; a new phase-2 `proving_key.zkey` and `verification_key.json` are required regardless, and the node's tag pin (`Cargo.toml:172-176`) and `flake.nix` input must move together.

### 4.6 Item 6 — epoch boundary

Not applicable yet. Today the only cross-epoch protection is the nonce: a leader proof carries `pol_epoch_nonce` as a public input and the ticket hashes it (`pol_lib.circom:19-25`), so a proof made with epoch `e`'s nonce is rejected by a verifier supplying epoch `e+1`'s. The bound does not loosen that and adds a second, independent rejection: with `e+1`'s `first_slot`, any slot of epoch `e` has a negative offset and fails `Num2Bits(64)` (B4's shape). Conversely nothing in the bound lets a proof for the last slot of `e` pass with `e+1`'s inputs, since both the nonce and the range would have to match. Confirmed by construction; B4/B5 are the witness-level checks.

## 5. Suggestions (non-security)

### S-001 · Land the bound as one change across the three repositories, using the gated form

| | |
|---|---|
| Category | ZK Soundness |
| Target | circuits `blend/poq.circom:47-50,131-134,158`; node `zk/proofs/poq/src/{inputs.rs:132,169-182,202-216, chain_inputs.rs:5-29}`, `blend/proofs/src/quota/inputs/{prove/mod.rs:28-34, prove/public.rs:64-74, verify.rs:37-52}`, `core/src/sdp/blend.rs:11-17`, `services/chain/chain-leader/src/leadership.rs:338-344`, `ledger/src/mantle/sdp/rewards/blend/mod.rs:271-279`, `services/blend/src/{edge/current_epoch.rs:208-214, core/mod.rs:623-629}` |

The circuit side is Appendix B, checked as in Appendix C. The node side: add `pol_epoch_first_slot: u64` and `pol_epoch_slot_count: u64` to `PoQChainInputsData`, `PoQChainInputsJson`, `PoQVerifierInput(Data)` (positions 13 and 14), and `LeaderInputs`; fill them from `EpochConfig::starting_slot` and `epoch_length` at the point where each site already has the epoch number, and from the same `EpochState` that supplies the nonce; extend `PolEpochState` and `BlendEpochState` so the prover and the three verifier sites read one source. Keep `unused_wallet_inputs()` at `slot: 0` (`inputs.rs:76-85`): the gate in the circuit is what makes that safe. Add PR #282's grinding test to `zk/proofs/poq/src/lib.rs` as a negative test once the bounded key is in, and re-run `test_core_node_full_flow`, `test_leader_full_flow`, `test_pow_full_flow`.

### S-002 · Spec text for `proof-of-quota.md`

| | |
|---|---|
| Category | Documentation |
| Target | `proof-of-quota.md` §Public values (`:53-75`), §Witness (`:92`), §Step 3, §Pseudocode (`:148-205`); `blend-protocol.md` §Proof of Quota (`:711-760`) |

Add to `ProofOfQuotaPublic` after `pol_t1`: `pol_epoch_first_slot: int  # first slot of the epoch (64 bits)` and `pol_epoch_slot_count: int  # slots in the epoch (64 bits)`, and state that the signal vector order is the listing order so the two come last. In §Step 3 add "3. The slot is in the current epoch: `pol_epoch_first_slot ≤ pol_sl < pol_epoch_first_slot + pol_epoch_slot_count`, as 64-bit integers." In the pseudocode, before the branch assertion: `slot_offset = (pol_sl - pol_epoch_first_slot) * L1` (a comment that the gate keeps core and PoW witnesses range-checkable), `is_leader = is_leader and (0 <= slot_offset < pol_epoch_slot_count)  # both operands range-checked to 64 bits`. In `blend-protocol.md` §Proof of Quota, the leadership part's public input list gains the epoch's slot range next to `e`. The #20 report's `Spec deviation` framing stands: `blend-protocol.md` states the semantics, `proof-of-quota.md` dropped them when it reused the PoL relation.

### S-003 · The circuits repository's input fixtures cannot exercise the core or leader branch

| | |
|---|---|
| Category | Testing |
| Target | circuits `rust/logos-blockchain-circuits-poq-sys/sample.input.json`, `blend/generate_inputs_for_poq.py:209-326` (main section; PoW difficulty `:277-280`, root randomisation `:317-320`) |

`sample.input.json` carries `selector: 2`, `index: 19` with `core_quota: 10`, `leader_quota: 15`: with any other selector it fails the quota check (`poq.circom:83`), and with `index` lowered it fails the branch equation (`:134`) because the generator replaces the roots of the branches it was not asked for with random values (`generate_inputs_for_poq.py:317-320`) and sets `pow_blend_difficulty` below the ticket unless the PoW role is chosen (`:277-280`). A reviewer who wants to check a change to the leader branch therefore has no fixture in the repository. The port used here (Appendix C) shows the cost of fixing this is nil: keep all three roots valid, always set the PoW difficulty above the ticket, compute `value` as an integer (`F(total_stake / 100)` is a float in Python 3), and commit one input per branch plus the all-branch one. A Sage-free version of the generator would also let it run in CI, where `docker run sagemath/sagemath` does not.

### S-004 · Choose the range encoding before the ceremony, and make the count non-zero by construction

| | |
|---|---|
| Category | Configuration |
| Target | circuits `blend/poq.circom` (Appendix B lines 137-142); node `consensus/cryptarchia-engine/src/time.rs:278-290` |

`(first_slot, slot_count)` and `(first_slot, last_slot)` are interchangeable for the circuit but not for the node: `epoch_length()` is `saturating_mul` of three `NonZero<u8>` and a `NonZero<u64>`, so the count is never zero and never overflows `u64`, whereas a last slot at `(epoch + 1) * length - 1` can be one past `u64::MAX` for a pathological configuration. `slot_count` is the safer encoding and is what Appendix B uses. B6 shows what a zero count does (every leader proof fails, core and PoW unaffected): worth a debug assertion where the inputs are built, not a circuit change.

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

## Appendix B — The prototype constraint (`blend/poq.circom` @ `ebf7ddf5`)

```diff
--- blend/poq.circom
+++ blend/poq_bounded.circom
@@ -48,6 +48,8 @@
     signal input pol_epoch_nonce;
     signal input pol_t0;
     signal input pol_t1;
+    signal input pol_epoch_first_slot;   // public: first slot of the epoch of pol_epoch_nonce
+    signal input pol_epoch_slot_count;   // public: number of slots in that epoch

     signal input pol_noteid_path[32];
     signal input pol_noteid_path_selectors[32];
@@ -128,9 +130,21 @@
     is_winning_pow.b <== pow_blend_difficulty;


+    // Leader branch only: the slot must lie in the epoch of pol_epoch_nonce,
+    // i.e. 0 <= pol_sl - pol_epoch_first_slot < pol_epoch_slot_count as 64-bit
+    // integers. The difference is gated by L1 so that core and PoW provers,
+    // who fill pol_sl with 0, are range-checked on 0 rather than on -first_slot.
+    signal slot_offset;
+    slot_offset <== (pol_sl - pol_epoch_first_slot) * L1;
+    component slot_in_epoch = SafeLessThan(64);
+    slot_in_epoch.in[0] <== slot_offset;
+    slot_in_epoch.in[1] <== pol_epoch_slot_count;
+    signal leader_ok;
+    leader_ok <== would_win.out * slot_in_epoch.out;
+
     // Enforce the selected role is correct
     signal lh_correctness_selector;
-    lh_correctness_selector <== (would_win.out - is_registered.out) * L1;
+    lh_correctness_selector <== (leader_ok - is_registered.out) * L1;
     is_registered.out + lh_correctness_selector + (is_winning_pow.out - is_registered.out) * L2 === 1;


@@ -155,5 +169,5 @@
 }

 // Instantiate with chosen depths: 20 for core PK tree
-component main { public [ core_quota, leader_quota, pow_quota, core_root, K_part_one, K_part_two, pol_epoch_nonce, pol_t0, pol_t1, pol_ledger_aged, pow_blend_difficulty] }
+component main { public [ core_quota, leader_quota, pow_quota, core_root, K_part_one, K_part_two, pol_epoch_nonce, pol_t0, pol_t1, pol_epoch_first_slot, pol_epoch_slot_count, pol_ledger_aged, pow_blend_difficulty] }
     = ProofOfQuota(20, 20);
```

`circom 2.2.2 --O2`: `v0.5.7` 20110 non-linear constraints, 11 public inputs, 20168 wires; bounded 20305 constraints, 13 public inputs, 20362 wires. Public-signal order from `--sym`: positions 1-12 unchanged, 13 `pol_epoch_first_slot`, 14 `pol_epoch_slot_count`.

## Appendix C — Witness-generation results

Base input: `generate_inputs_for_poq.py` ported to plain Python (seed 283, quotas 10/15/20, note value 50 of stake 5000, `f = 1/10`), all three branch roots valid, fixture slot `2186646283`, index `0`. Epoch range for the bounded cases: `E = 1000` slots. `slot` denotes the fixture slot, `s` the ground slot.

| Case | Circuit | Selector | `pol_sl` | `first_slot` | `count` | Witness | Where it fails |
|---|---|---|---|---|---|---|---|
| O1 | v0.5.7 | core | slot | – | – | ok | |
| O2 | v0.5.7 | leader | slot | – | – | ok | |
| O3 | v0.5.7 | PoW | slot | – | – | ok | |
| O4 | v0.5.7 | leader | slot + 1 (loses lottery) | – | – | fail | `:134` branch equation |
| G1 | v0.5.7 | leader | `s = 9223372036854776796` (wins, ground from `2^63` in 989 tickets) | – | – | **ok** | — #20 LB-001 reproduced |
| B1 | bounded | leader | slot | slot − 500 | 1000 | ok | |
| B2 | bounded | leader | slot | slot | 1000 | ok | |
| B3 | bounded | leader | slot | slot − 999 | 1000 | ok | |
| B4 | bounded | leader | slot | slot + 1 | 1000 | fail | `Num2Bits` (offset ≡ p − 1) |
| B5 | bounded | leader | slot | slot − 1000 | 1000 | fail | `:148` branch equation |
| B6 | bounded | leader | slot | slot | 0 | fail | `:148` branch equation |
| B7 | bounded | leader | slot | (slot − 2^64) mod p | 1000 | fail | `Num2Bits` (offset = 2^64) |
| B8 | bounded | leader | slot | slot + 1 | 2^64 + 1000 | fail | `Num2Bits` (count) |
| G2 | bounded | leader | `s` | slot − 500 | 1000 | **fail** | `:148` branch equation |
| G3 | bounded | leader | `s` | `s` − 500 | 1000 | ok | (verifier's range contains `s`: intended) |
| B9 | bounded | core | 0 | slot + 10^6 | 1000 | ok | |
| B10 | bounded | PoW | 0 | slot + 10^6 | 1000 | ok | |
| B11 | bounded | core | slot | slot + 10^6 | 1000 | ok | |
| B12 | bounded | PoW | slot | slot + 10^6 | 1000 | ok | |
| B13 | bounded | core | 0 | 0 | 0 | ok | |

Line numbers refer to `poq.circom` (`:134`) and `poq_bounded.circom` (`:148`) respectively; `Num2Bits` failures are reported by `snarkjs` as `Num2Bits_98 line 38` inside `SafeLessThan_101 line 70`. Scripts (`gen_poq_inputs.py`, `wtns_tests.py`, `grind_slot.py`) and the raw output were kept by the author; each is under 80 lines and reproducible from the description above.
