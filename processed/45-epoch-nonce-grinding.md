# Audit Report — Epoch nonce derivation and grinding resistance: the node matches the specification; a block carries exactly one bit of influence; private forks at the snapshot give a one-third adversary ten or more bits in about one epoch in forty

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/45` (parent `#5`)
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `9ffddb30b9e6cf79465802953caedd010ff1cecd` — component(s): `ledger/src/cryptarchia/mod.rs`, `ledger/src/config.rs`, `core/src/proofs/leader_proof.rs`, `zk/proofs/pol/src/inputs.rs`, `zk/poseidon2/src/hasher.rs`, `zk/groth16/src/lib.rs`, `codec/src/numbers.rs`, `services/chain/chain-leader/src/leadership.rs`, `tools/blockchain-tools/src/genesis/inscription.rs`, `tools/blockchain-tools/src/bin/genesis.rs`, `deployment/ceremony/genesis/*/inscribe.yaml`, `nodes/node/binary/src/config/deployment/settings.yaml`
Circuits: `https://github.com/logos-blockchain/logos-blockchain-circuits` @ `ebf7ddf5b5b625ef4ea7c803171c670beebd05ef` (tag `v0.5.7`, the tag the node pins at `Cargo.toml:172-176`) — `mantle/pol.circom`, `mantle/pol_lib.circom`, `misc/constants.circom`. Source review only; no key or proof was generated.
Specs: `https://github.com/logos-co/logos-lips` @ `75d3d0382604d4a0d8e246c268935dd386ffc8ea` — read in full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md` (core), `cryptarchia-proof-of-leadership.md`, `bedrock-anonymous-leaders-reward.md`, `cryptarchia-v1-protocol.md` (area; the last one is listed by #45 as "by section" but it is the document that defines the nonce, so it was read whole); by section: `bedrock-genesis-block.md` (the `genesis_epoch_nonce` field at lines 122-123 and §Epoch Nonce Ceremony, lines 152-178), `common-cryptographic-components.md` (the DST, sponge-padding and byte-conversion rules at lines 82 and 147-157, located by search), `bedrock-v1.1-block-construction.md` (§Proof of Leadership, §Construction Procedure step 1, lines 143-160, 251-262), `bedrock-v1.1-mantle-specification.md` (`derive_op_id`, `derive_note_id`, lines 1860-1873)
Date: 2026-09-19 — author: `agent (Claude)` — status: `final`

---

## 1. Summary

- Overall assessment: the node derives, evolves and snapshots the epoch nonce exactly as `cryptarchia-v1-protocol.md` specifies, the entropy contribution is a circuit output bound by the Proof of Leadership and leaves the proposer no freedom, and the stake snapshot precedes the nonce snapshot by `6k/f` slots as the design requires. The grinding surface is therefore the classical Praos one (withhold or release, and which parent to build on), which was quantified: pure tail withholding is negligible, but withholding combined with short private forks is not, and a candidate nonce costs only 18 core-seconds to evaluate. One deployment-level spec deviation was found in the genesis nonce.
- Findings: `0` critical · `0` high · `0` medium · `0` low · `2` informational
- Key themes: "implementation conforms", "one bit per block, nothing from block content", "grinding is bounded by combinatorics, not by computation", "test-network genesis nonce is a public constant".
- Must-fix before launch: LB-001 before any network whose genesis stakeholders are not all the operator (the mainnet ceremony). Nothing else.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `ledger/src/cryptarchia/mod.rs:64-166, 247-456, 493-604, 702-796` | `EpochState::update_from_ledger` (the nonce and stake snapshots), `LedgerState::update_epoch_state` (three epoch cases), `try_apply_proof`, `try_apply_header`, `update_nonce`, `epoch_state_for_slot`, `from_utxos`; tests at `:1386-1475`, `:1572-1594`, `:2204-2271` |
| `ledger/src/config.rs:65-112` | `nonce_snapshot`, `nonce_contribution_period`, `stake_distribution_snapshot` |
| `core/src/proofs/leader_proof.rs:25-210, 230-279` | Header proof structure and decoding, `verify`, `entropy`, native `check_winning` and `ticket` |
| `zk/proofs/pol/src/inputs.rs:85-166` | Order of the nine public signals handed to the verifier |
| circuits `mantle/pol_lib.circom`, `mantle/pol.circom`, `misc/constants.circom:14-24` | `ticket_calculator`, `derive_entropy`, the `main` public list, the two DST constants |
| `zk/poseidon2/src/hasher.rs`, `zk/groth16/src/lib.rs:59-69`, `codec/src/numbers.rs:54-67` | Sponge mode and padding, DST-to-field conversion, canonical field decoding of `entropy_contribution` |
| `services/chain/chain-leader/src/leadership.rs:212-225, 284-395` | Which epoch state the proposer uses, and when it reads it |
| `tools/blockchain-tools/src/genesis/inscription.rs`, `bin/genesis.rs:226-231`, `deployment/ceremony/genesis/{devnet,testnet,standalone}/inscribe.yaml`, `deployment/README.md:108`, `.github/ISSUE_TEMPLATE/{release,release-candidate}.md`, `nodes/node/binary/src/config/deployment/settings.yaml:113` | How `η_GENESIS` is produced for the shipped networks |
| Dynamic | The four existing unit tests that pin the snapshot behaviour; a standalone program linking the node's `logos-blockchain-poseidon2` crate to measure ticket cost, reproduce the shipped genesis nonce and measure best-of-N gain; a Monte Carlo of the candidate count at the snapshot slot (Appendix B) |

**Out of scope**

- Fork choice (online `k`-deep rule and the bootstrap density rule) is taken as specified; two conclusions in §4.3 lean on it and say so. It belongs to `#39`.
- Total stake inference and the lottery approximation (`#38`, `#147`), schedule-parameter binding and validation (`#175`, finding `38-LB-001`), the PoQ leader branch and its use of `pol_epoch_nonce` (`#20`, `#283`), leader key handling and the KMS (parent `#5`), the PoW reward path that reads `previous_epoch_nonce`.
- No Groth16 proof was generated or verified; the circuit was read, not compiled. The trusted setup is `#151`/`#191`.
- Third-party components assumed correct: `jf-poseidon2` at the pinned jellyfish revision, `ark-bn254`, `ark-ff`, `rust-rapidsnark`, `circomlib`.

**Assumptions**

- Poseidon2 behaves as a random function on the inputs used here, so distinct `(η_parent, ρ, sl)` triples give independent nonces and a ticket under one candidate nonce says nothing about the ticket under another.
- Adversarial relative stake is below one half of the active stake; figures are given for 10% to 45%.
- Repo-level facts from `#19` that matter here: release builds do not enable `overflow-checks`, but every arithmetic step on the nonce path is either a field operation or `strict_*` (`config.rs:74-75, 81-88`), so nothing on this path can wrap silently.

## 3. Method

- Worked through both items of `#45` after reading the specifications in the header, the specifications first and the code second.
- Spec conformance of the nonce against `cryptarchia-v1-protocol.md` §Epoch Schedule, §Epoch Nonce, §Epoch State Pseudocode, and of `ρ_LEAD` and the ticket against `cryptarchia-proof-of-leadership.md` §Circuit Public Inputs and §Circuit Constraints, by reading the node and the pinned circuit source side by side.
- Existing tests, run from the read-only checkout with a separate target directory: `cargo test --release --locked -p logos-blockchain-ledger --lib -- cryptarchia::tests::test_epoch_transition cryptarchia::tests::test_epoch_state_for_slot_with_empty_epochs cryptarchia::tests::test_try_apply_header_with_proof_from_jumped_epoch cryptarchia::tests::previous_epoch_nonce_is_retained` — 4 passed, 0 failed.
- Dynamic, own programs (Appendix B): a standalone crate with a path dependency on the node's `zk/poseidon2`, so tickets are computed by the same code as `LeaderPublic::ticket`; `tail_sim.py`, a 20,000-trial Monte Carlo per stake level. Hardware: Apple M3 Pro, one thread. Versions: `rustc 1.98.1`, `cargo 1.98.1`, `Python 3.9.6`.
- Comparison material: Ouroboros Praos and Genesis as summarised by the specifications themselves, and Cardano's CIP-0161 "Ouroboros Phalanx — Breaking Grinding Incentives" (which answers CPS-0021, randomness manipulation), fetched on the day of the review. Its framing is used in LB-002: an adversary able to try `R` nonce candidates raises the probability of any rare bad event by up to a factor `R`, and the defence it proposes is to make each candidate expensive.
- Automated tooling: none beyond the above.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Spec deviation: every shipped genesis ceremony derives `η_GENESIS` from the constant `0x00…01`, so the first lottery's randomness is public before the stake distribution is frozen | Configuration | Informational | High | Open |
| LB-002 | Nonce candidates cost 18 core-seconds each and private forks at the snapshot slot give a one-third adversary ten or more bits in 2.6% of epochs; neither the specification nor the node bounds this | Consensus | Informational | High | Open |

### 4.1 Item 1 — derivation: inputs, domain separation, bits of influence per block

**What the node computes.** `LedgerState::update_nonce` (`ledger/src/cryptarchia/mod.rs:591-604`) absorbs, in this order, `EPOCH_NONCE_V1`, the parent state's `nonce`, the header's `entropy_contribution` and `Fr::from(u64::from(slot))` into the rate-1 Poseidon2 sponge and finalises with the `10*` padding element (`zk/poseidon2/src/hasher.rs:24-36, 70-77`). That is `η_B = zkHASH(D_epoch ‖ η_parent ‖ ρ_LEAD ‖ Fr(sl))` of §Epoch Nonce, input for input. The domain separator is `fr_from_bytes(b"EPOCH_NONCE_V1")`, the little-endian reading that `common-cryptographic-components.md:157` prescribes; it is the value `0x45504f43485f4e4f4e43455f5631` and is distinct from the other two tags on this path. The incremental form the node uses and the one-shot `digest` give the same output (checked, Appendix B.3, which also gives a reference vector since neither the specification nor the node has one for this hash; see S-002).

**Where `ρ_LEAD` comes from.** It is the single output signal of the PoL circuit, `entropy_contribution <== entropy.out` with `derive_entropy = Poseidon2(NONCE_CONTRIB_V1, sl, note_id, secret_key)` (`mantle/pol_lib.circom:30-44, 179, 199-205`), exactly constraint 7 of the PoL specification. The circuit constants equal the little-endian integers of their ASCII tags (`LEAD_V1` = 13887241025832268, `NONCE_CONTRIB_V1` = 65580641403957881555985426713123114830; recomputed). The node takes the value from the header (`leader_proof.rs:199-201`), places it first in the nine-signal vector (`leader_proof.rs:170-184`, `zk/proofs/pol/src/inputs.rs:128-140`), which is where snarkjs puts a circuit output ahead of the `public [sl, epoch_nonce, t0, t1, ledger_aged, ledger_latest, P_lead_part_one, P_lead_part_two]` list of `pol.circom:6`, and only then feeds it to `update_nonce` (`mod.rs:533-536`: epoch state, then proof, then nonce). A header whose `entropy_contribution` is not the circuit's output for the winning `(sl, note, sk)` fails `try_apply_proof` (`mod.rs:511-513`) and never reaches the nonce. The field is decoded canonically, values at or above the modulus being rejected (`codec/src/numbers.rs:61-64`), so there is no second encoding of the same contribution.

**Bits of adversarial influence per block.** Going through every header and body field:

| Lever | Bits | Why |
|---|---|---|
| Block body, transaction choice, uncle list | 0 | `η_B` reads only `η_parent`, `ρ_LEAD`, `sl`. `body_root` is not an input. |
| `leader_key`, `leader_voucher` | 0 | Not inputs to `η_B` or to `ρ_LEAD`. |
| Groth16 proof bytes (re-randomisable) | 0 | Not an input. This matters: a hash over the header would have made the nonce freely grindable by re-proving. |
| `ρ_LEAD` itself | 0 | Deterministic in `(sl, noteID, sk)`; the proposer cannot choose it, and the same note in the same slot gives the same value on every fork. |
| Slot | 0 | The slot must be one the note won. |
| Release or withhold the block | 1 | The only per-block choice. |
| Several notes winning the same slot | `log2(m+1)` for `m` winners | Each note has its own `ρ`. With stake share `α` the chance of two adversarial winners in one slot is about `(αf)²/2`, 6·10⁻⁵ at `α = 1/3`, `f = 1/30`; splitting stake does not raise it usefully. |
| Parent choice | one alternative `η_parent` per viable parent | Viable means the resulting chain is adopted, which costs adversarial blocks; quantified in LB-002. |
| Ordering of own blocks | 0 | Slots strictly increase along a chain (`mod.rs:264-269`), so a set of blocks has one order. |

So a block carries exactly one bit, and the count of candidate nonces is the count of distinct chains the adversary can get adopted at the snapshot slot. One property differs from Praos and is harmless: `ρ_LEAD` does not depend on the epoch nonce, whereas the Praos contribution is a VRF output over the current nonce. Because slots are numbered globally, `(sl, noteID, sk)` never repeats, so the contribution is still fresh per slot; the owner can compute its own future contributions at any time, but that gives nothing without knowing `η_parent`, which depends on honest contributions keyed by secrets the adversary does not hold.

### 4.2 Item 2 — is the nonce fixed before the adversary can see the resulting eligibility

**Snapshot slot and boundary.** `Config::nonce_snapshot(epoch)` is `(epoch-1)·epoch_length + base_period·(stake_stabilization + nonce_buffer)` (`config.rs:71-89`), i.e. `sl_{ep-1} + 6k/f` with the shipped 3/3/4 split, which is the argument of `epoch_nonce_at_slot` in §Epoch State Pseudocode. `update_from_ledger` is called on the **parent** state at the start of every header application and copies `ledger.nonce` while `ledger.slot < nonce_snapshot_slot` (`mod.rs:117-140, 279-282`). The next epoch's nonce is therefore the nonce after the last block whose slot is strictly below the snapshot slot; a block exactly at the snapshot slot is excluded. That is the specification's "last block before the start of the Lottery Constants Finalization phase", and `test_epoch_transition` pins it (`mod.rs:1414, 1462-1467`: a block at slot 60 of a 100-slot epoch does not contribute).

**The three epoch cases** of `update_epoch_state` (`mod.rs:293-455`) were each checked against the pseudocode:

| Case | Code | Specification value | Result |
|---|---|---|---|
| Same epoch | `next_epoch_state` refreshed from the parent, `epoch_state` untouched | — | Conforms |
| Next epoch | `epoch_state = next_epoch_state` as refreshed from the parent (`:326-331`); the new `next_epoch_state` starts from `self.nonce` (`:335`) and is refreshed by later blocks | `η^{ep}` = nonce at the last block before `sl_{ep-1}+6k/f`. If the parent is itself before that slot, the refresh at `:279-282` picks it up; if the first `6k/f` slots of the new epoch stay empty, the initial `self.nonce` is already the right value | Conforms |
| Skipped epochs | `epoch_state.nonce = self.nonce`, `utxos = self.utxos` (`:408-424`) | No block exists between the parent and `sl_{new-1}+6k/f`, so both the nonce and the commitment root "at" those slots are the parent's | Conforms; see §4.3 for what this case gives up |

The genesis path seeds both `epoch_state` and `next_epoch_state` with the genesis nonce and the genesis notes (`mod.rs:738-796`), and `stake_distribution_snapshot(1) = 0` keeps epoch 1 on the genesis notes, as `C_LEAD^{1} = commitment_root_at_slot(0)` requires. `nonce_snapshot` cannot be called for epoch 0: `next_epoch_state.epoch` is never below 1, and the `strict_sub` would panic rather than wrap (test at `config.rs:675`).

**Order of events for epoch `ep`.** Notes are frozen at `sl_{ep-1}`; the nonce is frozen `6k/f` slots later; it is used `4k/f` slots after that. A note created after the adversary learns anything about `η^{ep}` is not in `C_LEAD^{ep}`; it enters `C_LEAD^{ep+1}` at the earliest, whose nonce is settled `6k/f` slots after that note set is frozen. Both roots are enforced inside the proof (`aged_root`, `latest_root` at `mod.rs:503-510`). So the secret-key grinding the specification's §Note Ageing warns about is closed in normal operation, on the code as well as on paper.

**The honest answer to the question as asked is "no, and no Praos-family protocol can say yes".** The party that controls the tail of the contribution window sees its own eligibility under each candidate before choosing which chain to release. What the design guarantees is that the choice is among few candidates. How few is LB-002.

**Proposer and validator agree.** The proposer obtains its epoch state through the same function (`epoch_state_for_slot`, `mod.rs:702-714`, which is `update_epoch_state` on a clone) and builds its public inputs from it (`leadership.rs:212-225`). The per-epoch winning-slot scan reads the state at the first tick of the epoch (`leadership.rs:323-344`), by which time the nonce has had `4k/f` slots, about `4k` blocks, to settle; a reorganisation that changed it would have to be deeper than `k`.

### 4.3 Checked and ruled out

- **Uncle proofs verified under a different nonce than the uncle's leader used.** `verify_proof_of_leadership` derives the epoch state from the uncle's parent state (`mod.rs:557-571`), while the specification says "as derived on the chain of `A`". The two differ only if the referencing chain has blocks between the uncle's parent and a snapshot slot that lies before the uncle's slot, which needs `sl_U − sl_parent(U) > 4k/f`. The reference window caps `sl_A − sl_parent(U)` at `W/f`, and the virtual bound `W ≤ 0.6k` keeps `W/f` far below `4k/f` (360 against 259,200 slots at the specified constants; 240 against 2,400 at the shipped `k = 30`, `f = 1/20`, `W = 12`). Not reachable. Whether `W` is validated against `k` at load is `#175`.
- **Note ageing on a chain with a block-free stretch.** Whenever a chain has no block in `[sl_{e+1}, sl_{e+1}+6k/f)`, the last block before `sl_{e+1}` fixes both `C_LEAD^{e+2}` and `η^{e+2}`; this is the skipped-epochs case and a sub-case of the next-epoch case. The producer of that block knows `η_B` before it chooses the block's transactions, so it can create notes ground against `η_B` that will be eligible under exactly that nonce: ageing is void. With many tiny notes each ground to win one slot, such a fork can be given a block in every slot at a cost of roughly `1/(f·v/D)` tickets per note. This is not exploitable on the public chain, where honest leaders fill the window (the buffer phase exists for this), and a private fork built this way has zero density for at least `6k/f` slots after its divergence point, so it loses under the bootstrap density rule and is more than `k` deep for online nodes. The protection is fork choice, not the ledger; recorded as S-003.
- **Wrap-around on the nonce path.** None: field arithmetic and `strict_*` only.
- **Non-canonical `entropy_contribution`.** Rejected at decode.
- **Withheld blocks recovering value as uncles.** A withheld tail block can be referenced as an uncle later and then counts for the stake inference, but uncles earn no voucher and do not touch the nonce (`mod.rs:533-536` applies `update_nonce` for the header only). Withholding costs one block reward per block.
- **Influence on the lottery difficulty as a side channel.** The occupied-slot window and the nonce window are the same `6k/f` slots, so tail withholding also lowers `D^{ep}` slightly. That raises everyone's win rate in proportion and gives no relative gain.

### LB-001 · Spec deviation: every shipped genesis ceremony derives `η_GENESIS` from the constant `0x00…01`, so the first lottery's randomness is public before the stake distribution is frozen

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Configuration |
| Target | `deployment/ceremony/genesis/{devnet,testnet,standalone}/inscribe.yaml:1-3`; `tools/blockchain-tools/src/genesis/inscription.rs:26-51` (`fn inscribe`); `.github/ISSUE_TEMPLATE/release.md:31`, `release-candidate.md:33`; `deployment/README.md:108` |
| Status | Open |

**Description**

`bedrock-genesis-block.md` §Epoch Nonce Ceremony requires that the genesis nonce "must be revealed AFTER the initial stake distribution has been frozen", and builds it from a Bitcoin block hash, an Ethereum block hash and a drand round, all taken after a pre-announced time `t`. The reason is the one §Note Ageing gives: a stakeholder who knows the nonce can grind a secret key.

The three shipped ceremonies instead carry, under a header that says `TEMPLATE: shared default, do not edit per release`:

```yaml
entropy_sources:
  - "0000000000000000000000000000000000000000000000000000000000000001"
```

and both release checklists tell the release manager "No need to change the `entropy_sources`". `inscribe` hashes whatever list it is given, requires only that it is non-empty (`inscription.rs:33-41`), and records nothing about where the values came from. The 32 bytes are read little-endian (`fr_from_bytes`), so the nonce is `zkhash(2^248)`. Recomputing that with the node's hasher gives `2d2ddf918544bca603c5a291c7dd1b902d6769ff4b00021506780e075c06051a`, which is byte for byte the last 32 bytes of the inscription checked in at `nodes/node/binary/src/config/deployment/settings.yaml:113` (Appendix B.2). Every devnet and testnet release therefore starts epoch 0 with the same, permanently public nonce. Epoch 1 also runs on the genesis notes, under a nonce produced by epoch 0's blocks.

The implementation is the side that deviates. For networks where one operator generates every stakeholder key this has no consequence, which is why it is Informational. It matters because the tool and the checklists are what a mainnet ceremony would start from, and nothing in the tool distinguishes a test value from a ceremony output.

**Exploit scenario**

Genesis note identifiers depend only on the stake-distribution operation (`derive_note_id` takes the `OpId` of the transfer, `core/src/mantle/ledger.rs:512-527`), not on `chain_id` or `genesis_time`. So once `stakeholders.yaml` is final, every ticket of epoch 0 is a public function of each holder's secret key. A participant who contributes its key last, or the person assembling the file, replaces its own key, recomputes the identifiers and counts its wins in epoch 0, at 28 µs per ticket. At the shipped devnet schedule one trial of a whole epoch costs 0.17 s; a million trials, two core-days, buy about five standard deviations, which for a 10% holder is roughly 60 wins against an expected 31, and the blocks of epoch 0 are the ones that decide epoch 1's nonce. Under the specified ceremony the same participant learns the nonce only after the file is frozen and can do nothing.

**Recommendation**

- *Short term*: say in the three templates and the two checklists that the constant is for test networks only, and make `genesis ceremony` refuse a single-source or all-constant list unless an explicit `--insecure-test-entropy` flag is passed.
- *Long term*: give `inscribe` the ceremony's actual shape, three named sources with their heights and round number and the announced time `t`, write them into the ceremony output so that anyone can recheck the nonce, and have the ceremony fail if the stake-distribution file changed after `t`.

**References**: `bedrock-genesis-block.md` §Epoch Nonce Ceremony (lines 152-178); `cryptarchia-v1-protocol.md` §Note Ageing. Related but distinct: `95-LB-004` (#340), the checked-in inscriptions' encoding.

### LB-002 · Nonce candidates cost 18 core-seconds each and private forks at the snapshot slot give a one-third adversary ten or more bits in 2.6% of epochs; neither the specification nor the node bounds this

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Consensus |
| Target | `ledger/src/cryptarchia/mod.rs:591-604` (`fn update_nonce`), `:117-140` (nonce snapshot); `core/src/proofs/leader_proof.rs:273-275` (`fn ticket`); `cryptarchia-v1-protocol.md` §Epoch Nonce |
| Status | Open |

**Description**

This is a property of the design, which the node implements faithfully; it is recorded as a finding because #45 asks for the number and because the specification gives none. `cryptarchia-proof-of-leadership.md` 1.1.0 removed the adaptive-adversary protection, and no raw-status document contains a grinding bound.

*How many candidates.* Each distinct chain ending below the snapshot slot gives a distinct nonce (§4.1). Two strategies were counted, per epoch, by Monte Carlo (Appendix B.4; 20,000 trials per row). "Tail only" is the textbook one: release any subset of the adversarial blocks that follow the last honest block, `2^t` chains. "With forks" adds what withholding makes possible: the adversary keeps all its blocks private, honest leaders extend the honest-only chain, and just before the snapshot the adversary releases the honest chain up to some honest block followed by a subset of its own later blocks that is strictly longer than the honest blocks it orphans. Ties are given to the honest side and network delay is ignored, so the second column is a lower bound for an adversary with good connectivity.

| Stake `α` | Tail only: mean bits | Tail only: P(≥10 bits) | With forks: mean bits | With forks: P(≥10 bits) | With forks: P(≥20 bits) |
|---|---|---|---|---|---|
| 0.10 | 0.11 | 0 | 0.13 | 0 | 0 |
| 0.20 | 0.25 | 0 | 0.44 | 0.0001 | 0 |
| 0.30 | 0.44 | 0.00005 | 1.16 | 0.010 | 0.00015 |
| 0.33 | 0.50 | 0 | 1.72 | 0.026 | 0.0016 |
| 0.40 | 0.67 | 0.00015 | 4.40 | 0.14 | 0.038 |
| 0.45 | 0.82 | 0.0004 | 13.6 | 0.41 | 0.24 |

The tail-only column agrees with the closed form `E[2^t] = (1−α)/(1−2α)`, two candidates on average at one third. The fork column does not stay small: at one third, one epoch in about forty offers a thousand or more candidates and one in six hundred offers a million. These counts do not depend on `k`, `f` or the epoch length, because only the order of adversarial and honest block events before the snapshot matters.

*What a candidate costs.* Evaluating a candidate means computing the adversary's tickets for the next epoch. The node's own Poseidon2 does one ticket in 28 µs on one M3 Pro core (35,600 per second, Appendix B.1). Stake can sit in one note without losing win probability, since the threshold is close to linear in value, so a candidate costs one ticket per slot:

| Schedule | Slots per epoch | One candidate | 2¹⁰ candidates | 2²⁰ candidates |
|---|---|---|---|---|
| Specified, `k = 2160`, `f = 1/30` | 648,000 | 18.5 s measured | 5.2 core-hours | 224 core-days |
| Shipped devnet, `k = 30`, `f = 1/20` | 6,000 | 0.17 s measured | 3 core-minutes | 2 core-days |

These are for unoptimised CPU code; sharing the first two sponge steps across slots removes a third of the work, and the search is embarrassingly parallel. Nothing else has to be computed per candidate. For comparison, the Praos evaluation is one VRF per slot per pool, and CIP-0161 judges that too cheap and proposes to raise the cost of grinding by about ten orders of magnitude. Here the computation is no obstacle up to about 2²⁰ and only a modest one beyond.

*What the candidates buy.* Choosing the best of `N` nonces by own win count, measured with real tickets for a 33% holder (Appendix B.1): on the devnet schedule +10.7% blocks at `N = 4`, +17.5% at 16, +23.1% at 64, +30.1% at 1,024; on the specified schedule +1.9% at 4 and +2.8% at 16 (a single group of 16, so noisy; the normal approximation gives +2.1%, and +3.8% at 1,024). The relative gain scales as one over the square root of the adversary's blocks per epoch, which is why the short devnet epoch is so much more sensitive. The win count is the mild objective. The general statement, which is the one CIP-0161 uses, is that `N` candidates multiply the probability of any rare event the adversary is waiting for, such as holding a run of slots long enough to out-build the honest chain, by up to `N`. A settlement bound computed for `k` without that factor is optimistic by 2¹⁰ in one epoch in forty at one-third stake. The strategy is not free: every withheld or orphaned block forfeits its voucher, and for the tail-only strategy the candidates do not compound, since the longest adversarial tail among `N` nonces grows only like `log N / log(1/α)`, which for `α < 1/2` gives fewer candidates in the next epoch than were spent in this one. Whether the fork strategy compounds was not analysed.

**Exploit scenario**

A holder of one third of the active stake withholds its blocks over the last few hundred slots before `sl_e + 6k/f`. In about one epoch in forty the block pattern lets it choose among a thousand or more chains, each of which honest nodes adopt because it is the longest they see. It evaluates them, a few core-hours of work that parallelises freely and that must finish before honest blocks outgrow the chains it holds, so in practice it is spread over many cores or run against a cheaper objective than the whole next epoch; it then releases the chain that maximises its objective for epoch `e+1`, and pays for it with the rewards of the blocks it dropped. On the shipped devnet schedule the same procedure, in such an epoch, raises its block count for the next epoch by roughly 30%.

**Recommendation**

- *Short term*: add a grinding section to `cryptarchia-v1-protocol.md` that states the candidate count for the fork strategy and the factor it puts on the settlement bound, so that `k` is chosen with it in view; state the assumption under which the buffer phase's "at least one honest leader" argument holds.
- *Long term*: decide whether the per-candidate cost should be raised, in the way CIP-0161 does with a delay function over the nonce, given that a whole candidate costs 18 core-seconds here. For test networks, note in the deployment documentation that `k = 30` makes best-of-N selection worth tens of percent, so lottery statistics gathered there do not transfer.

**References**: `cryptarchia-v1-protocol.md` §Epoch Schedule (buffer phase), §Epoch Nonce; `cryptarchia-proof-of-leadership.md` revision 1.1.0; Cardano CIP-0161 and CPS-0021; `38-LB-001` (#453) for the schedule parameters.

## 5. Suggestions (non-security)

### S-001 · Say in the specification that the snapshot excludes the block at the snapshot slot

`epoch_nonce_at_slot(sl, tip)` is used but not defined. The prose ("the last block before the start of the … phase") and the node agree that a block whose slot equals `sl_{ep-1}+6k/f` does not contribute, and a test pins it, but an implementer reading only the pseudocode could include it and fork at the first such block. Define the helper as "the nonce after the last ancestor of `tip` with slot strictly below `sl`".

### S-002 · Add a test vector for the nonce update

`EPOCH_NONCE_V1` exists only in the node; the circuit never sees it, so no proof would catch a second implementation that encodes the tag or orders the inputs differently, and the divergence would surface only at the first epoch boundary. Neither the specification's §Test Vectors nor the node has a vector for this hash. One, computed with the node's hasher: tag as field bytes `45504f43485f4e4f4e43455f5631` followed by 18 zero bytes; `η_parent = 1`, `ρ_LEAD = 2`, `sl = 3`; `η_B` (little-endian) = `b6267031acbb624024c906d16e0a1aee4da4551eef3ec5673a5d038cf72e042e`. A unit test asserting it beside `update_nonce`, and the same row in the specification, would close the gap.

### S-003 · Record that note ageing depends on fork choice across block-free windows

The skipped-epochs branch and the empty-first-window sub-case of the next-epoch branch (`mod.rs:326-349, 408-440`) let one block fix both the eligible note set and the nonce of a later epoch (§4.3). That is what the specification prescribes and it is safe only because such a fork cannot win under either fork-choice rule. A comment at those branches and a sentence in §Note Ageing would keep a future change to fork choice, or a recovery procedure for a halted chain, from removing the protection unnoticed. A chain restarted after a halt longer than `6k/f` slots is exactly this case, with the producer of the last block before the halt as the favoured party.

---

## Appendix A — Definitions

Severity, difficulty and categories are those of `docs/REPORT_TEMPLATE.md`, Appendix A, unchanged.

## Appendix B — Reproduction

All programs lived outside the node checkout, which was left unmodified (`git status` clean before and after).

### B.1 Ticket cost and best-of-N gain

A standalone crate with `logos-blockchain-poseidon2 = { path = "<node>/zk/poseidon2" }`, `ark-ff 0.5`, `num-bigint 0.4`, `rand 0.8`, built `--release`. The ticket is `Poseidon2Bn254Hasher::digest(&[LEAD_V1, nonce, Fr::from(slot), note_id, sk])`, the expression at `core/src/proofs/leader_proof.rs:273-275`. The threshold is `p·(1−(1−f)^α)` to 2⁻⁶⁰ relative precision instead of the node's `t0`/`t1` form; the difference is far below the sampling noise. One random note, random candidate nonces, wins counted over one epoch, and for each `N` the mean over disjoint groups of `N` candidates of the group maximum, relative to the expected count.

```
ticket hashes: 200000 in 5.615s -> 35622 hashes/s, 28.073 us/hash
devnet k=30 f=1/20 alpha=0.33 expected=100.7 mean=101.0 | N=1:+0.33% N=4:+10.74% N=16:+17.51% N=64:+23.07% N=256:+27.35% N=1024:+30.08% | 0.17s per candidate
spec  k=2160 f=1/30 alpha=0.33 expected=7209.1 mean=7219.0 | N=1:+0.14% N=4:+1.88% N=16:+2.83% | 18.45s per candidate
```

(1,024 candidates on the devnet schedule, 16 on the specified one. A first run of 2,000,000 tickets gave 27.6 µs.)

### B.2 The shipped genesis nonce

```rust
let mut src = [0u8; 32];
src[31] = 0x01;                                  // hex "00…01", read little-endian by fr_from_bytes
let eta = Poseidon2Bn254Hasher::digest(&[Fr::from_le_bytes_mod_order(&src)]);
// little-endian hex: 2d2ddf918544bca603c5a291c7dd1b902d6769ff4b00021506780e075c06051a
```

`settings.yaml:113` holds `…6c6f63616c` (`chain_id`), four bytes of genesis time, then `2d2ddf91…5c06051a`.

### B.3 Nonce update vector

`digest(&[dst, 1, 2, 3])` and the incremental `new`/`update`×4/`finalize` sequence that `update_nonce` uses return the same element, the one quoted in S-002.

### B.4 Candidate count at the snapshot slot (`tail_sim.py`)

```python
def trial(alpha, window, rng):
    ev = [rng.random() < alpha for _ in range(window)]      # True = adversarial block event, oldest first
    h = sum(1 for e in ev if not e)
    a_after, a = [], 0                                       # a_after[j]: adversarial blocks after honest block j
    for e in reversed(ev):
        if e: a += 1
        else: a_after.append(a)
    a_after.append(a); a_after.reverse()
    t = a_after[h]
    total = 2 ** t                                           # extend the honest tip with any subset
    for j in range(h - 1, -1, -1):                           # orphan h-j honest blocks: need a strictly longer chain
        need, aj = h - j + 1, a_after[j]
        if aj >= need:
            total += sum(comb(aj, m) for m in range(need, aj + 1))
    return t, math.log2(total)
```

Seed 45, 20,000 trials per row, windows of 100 to 600 block events before the snapshot (longer for larger `α`; deeper forks add nothing measurable at the smaller values). Output:

```
alpha window | no-fork: E[bits] P(>=5) P(>=10) | with forks: E[bits] median P(>=5) P(>=10) P(>=20)
 0.10    100 |     0.11  0.0000 0.00000 |     0.13   0.00  0.0003 0.00000 0.00000
 0.20    100 |     0.25  0.0002 0.00000 |     0.44   0.00  0.0070 0.00010 0.00000
 0.30    200 |     0.44  0.0026 0.00005 |     1.16   0.00  0.0592 0.00975 0.00015
 0.33    200 |     0.50  0.0046 0.00000 |     1.72   1.00  0.1078 0.02560 0.00160
 0.40    300 |     0.67  0.0105 0.00015 |     4.40   1.58  0.2935 0.14325 0.03820
 0.45    600 |     0.82  0.0191 0.00040 |    13.57   6.74  0.5629 0.40875 0.23570
```

The model treats block events as independent with adversarial probability `α`, ignores slots won by both sides, gives ties to the honest chain and ignores propagation delay. It is a model of the protocol, not a run of the node.
