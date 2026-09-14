# Audit Report — Sweep: non-determinism in anything hashed, signed, or agreed on

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/33`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `consensus/cryptarchia-engine, ledger, core, zk/proofs/pol, blend/message, blend/crypto, services/blend, services/chain`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md, overview-cryptoeconomics.md` (in full); by section: `cryptarchia-v1-protocol.md, fork-choice.md, cryptarchia-total-stake-inference.md, mantle-transaction-encoding.md, network-wire-format.md, blend-protocol.md, bedrock-service-reward-distribution.md, bedrock-v1.1-mantle-specification.md`
Date: `2026-09-12` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: no cross-node non-determinism was found on the paths that produce hashed, signed or agreed-on values at this commit; the three findings are one class of spec deviation (consensus constants evaluated in `f64` instead of exact integer arithmetic) and two latent hash-order dependencies that are harmless today but sit directly on consensus paths.
- Findings: 0 critical · 0 high · 0 medium · 3 low · 0 informational
- Key themes: `f64` used to derive consensus constants, `rpds` hash-trie iteration on consensus paths, tie-breaks that depend on `RandomState`.
- Must-fix before launch: none. LB-002 must be fixed before a second `ServiceType` is added.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `consensus/cryptarchia-engine/src/{lib.rs,config.rs,time.rs}` | fork choice, uncle selection, LIB/pruning, config-derived constants |
| `ledger/src/{lib.rs,config.rs}`, `ledger/src/cryptarchia/{mod.rs,stake.rs,block_density.rs}` | epoch state, total stake inference, reward UTXO insertion |
| `ledger/src/mantle/sdp/**` | declarations, activity proofs, reward calculation and distribution |
| `core/src/{header,block,mantle,sdp,blend,proofs}` | header/block hashing, note ids, tx encoding, SDP ops, core quota |
| `zk/proofs/pol/src/lottery.rs`, `zk/groth16/src/lib.rs` | lottery constants, field-element decoding |
| `blend/message/src/reward/**`, `blend/crypto/src/merkle.rs`, `blend/membership` | activity threshold, membership Merkle tree and index |
| `services/blend/src/{core/settings.rs,membership/**}` | prover-side derivation of quota and membership index |
| `services/chain/**`, `services/tx-service/src/backend/pool.rs`, `services/pow`, `services/time` | wall-clock and RNG use, HashSet/HashMap use around block processing and sync |
| `codec/`, `merkle/`, `mmr/` | width-dependent integer encodings |

**Out of scope**
`c-bindings/`, `wallet/`, `zone-sdk/`, `nodes/node/http-client`, `libp2p/` NAT code, `logos_sql`, test modules and `tests/`. Third-party crates assumed correct: `rpds`, `ark-*`/`lb_groth16` field arithmetic, `astro-float`, `blake2`, `bincode`, `serde`, `num-bigint`, `rust-rapidsnark`.

**Assumptions**
All nodes run the same binary on an IEEE-754 platform (x86_64 with SSE2 or aarch64; no x87 extended precision, no FMA contraction, which `rustc` does not perform implicitly). All nodes load the same genesis and deployment configuration. Specifications at the commit above are the reference.

## 3. Method

- Manual review of the in-scope paths, working through issue `#33` (parent `#24`), and issue `#19` for repo-level facts (`overflow-checks` off in release, `as_conversions`/`indexing_slicing`/`unwrap_used` allowed).
- Spec conformance against `cryptarchia-v1-protocol.md` §Constants, §Notation, §Epoch State, §Uncle References, §Block Header Validation; `fork-choice.md` §Protocol; `cryptarchia-total-stake-inference.md` §Algorithm; `blend-protocol.md` §Core Quota, §Reward Calculation, §Rewarding Distribution Logic; `bedrock-service-reward-distribution.md` §Protocol; `bedrock-v1.1-mantle-specification.md` `derive_note_id`; `mantle-transaction-encoding.md` and `network-wire-format.md` in full.
- Sweeps with ripgrep 14.1.1 over non-test code for each checklist item: `HashMap|HashSet|HashTrieMap|HashTrieSet` iteration, `f32|f64`, `usize|isize` fields in `Serialize`/`BinaryCodec` types, `Instant::now|SystemTime::now|UNIX_EPOCH`, `thread_rng|OsRng|random`, `partial_cmp|sort_by|sort_unstable_by`, `format!("{:?}")` outside logging.
- Numeric checks: Python 3 and `rustc 1.98.1` (the pinned toolchain) programs comparing the `f64` formulas in `config.rs` and `core/src/blend/mod.rs` against exact integer arithmetic over ranges of parameter values.
- Dynamic testing: none.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Spec deviation: consensus constants derived in `f64` differ from the exact integer values for some parameter choices | Determinism | Low | High | Open |
| LB-002 | Reward UTXOs of different services are inserted into the UTXO tree in hash-trie iteration order | Determinism | Low | High | Open |
| LB-003 | Fork choice iterates chain tips in hash-randomised order, so ties between competing forks are not first-seen | Consensus | Low | High | Open |

### LB-001 · Spec deviation: consensus constants derived in `f64` differ from the exact integer values for some parameter choices

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Determinism |
| Target | `consensus/cryptarchia-engine/src/config.rs:107-113` (`Config::s_gen`), `consensus/cryptarchia-engine/src/config.rs:125-131` (`average_slots_for_blocks`), `core/src/blend/mod.rs:20-29` (`core_quota`) |
| Status | Open |

**Description**

`slot_activation_coeff` is stored exactly as a `NonNegativeRatio { numerator: u32, denominator: NonZeroU32 }`, but the three consensus-visible lengths derived from it are computed by converting it to `f64` first:

```rust
// config.rs:107-113
NonZero::new(((self.security_param.get() as f64) / (4.0 * self.slot_activation_coeff.as_f64())).floor() as u64)
// config.rs:129
NonZero::new((num_blocks.get() as f64 / slot_activation_coeff.as_f64()).floor() as u64)
```

`average_slots_for_blocks` yields the base period length (`⌊k/f⌋`, from which `epoch_length`, the TSI period and every epoch boundary are computed through `ledger/src/config.rs:28-34,81`) and the uncle reference window `⌊W/f⌋`; `s_gen` is the density window of the bootstrap fork choice. The specification defines all three as integer floors of a rational (`cryptarchia-v1-protocol.md` §Constants and §Notation, `fork-choice.md` §Definitions). `f64` division rounds, and `floor` of a value that landed one ulp below an integer loses 1. At the pinned toolchain:

| `f` | `⌊k/f⌋` exact | `config.rs` | `⌊k/(4f)⌋` exact | `config.rs` |
|---|---|---|---|---|
| 1/30 (deployment templates) | 64800 | 64800 | 16200 | 16200 |
| 27/50 | 4000 | 3999 | 1000 | 999 |
| 9/14 | 3360 | 3359 | 840 | 839 |

Over all ratios `n/d` with `d ≤ 1000` and `k = 2160`, 2697 values of `f` give a base period or `s_gen` one slot short of the specified value. The same file already computes `expected_blocks_per_epoch` (`config.rs:145-164`) with exact `u128` arithmetic, so two conventions coexist for the same parameter.

`core_quota` (`core/src/blend/mod.rs:20-29`) has the same shape: `ceil(E * F_C * β_C / N)` evaluated in `f64`, where `E`, `β_C`, `N` are integers and `F_C` is a `PositiveF64` from configuration. The result is a public input of every activity proof (`ledger/src/mantle/sdp/rewards/blend/current_epoch.rs:217-231`, `CoreInputs { zk_root, quota }`), so a one-unit difference makes every `SDPActive` operation of the epoch invalid. Over `E ∈ {2160, …, 648000}`, `F_C ∈ {0.01, …, 1.00}`, `β_C ∈ {1,2,3}`, `N < 200`, 1102 combinations give a `f64` ceiling one above the exact ceiling (for example `E = 12960, F_C = 0.55, β_C = 1, N = 2`: `3565` vs `3564`).

All nodes running this binary agree with each other, because IEEE-754 division, multiplication, `floor` and `ceil` are correctly rounded and identical on every supported target. The gap is between this implementation and the specification (or any second implementation that follows it), and it only opens for parameter values other than the ones in the deployment templates.

**Exploit scenario**

Not exploitable by a network participant. Impact: a deployment that sets `slot_activation_coeff` to one of the affected ratios has epoch boundaries one slot earlier than the specification derives, and a node implementing the specification's integer formula would compute a different epoch state from the same chain (different `⌊k/f⌋` shifts every `EpochState` snapshot) and reject the other's blocks. For `core_quota`, an activity proof produced by a prover using exact arithmetic would not verify on this node, and vice versa.

**Recommendation**
- *Short term*: compute `average_slots_for_blocks` and `s_gen` as `(num_blocks * denominator) / numerator` and `(k * denominator) / (4 * numerator)` in `u128`, as `expected_blocks_per_epoch` already does. For `core_quota`, replace `PositiveF64` `message_frequency_per_round` with a ratio and compute `(E * F_num * β_C).div_ceil(F_den * N)` in `u128`.
- *Long term*: forbid `f64` in `lb_cryptarchia_engine::Config`, `lb_ledger::Config` and `RewardsParameters` (a `clippy::disallowed_types` entry for `f64` in those crates), and add a test that derives each constant both ways over a sweep of ratios.

**References**: `cryptarchia-v1-protocol.md` §Constants (`W·f⁻¹`), §Notation (`s = 3⌊k/f⌋`); `fork-choice.md` §Definitions (`s_gen = ⌊k/(4f)⌋`); `blend-protocol.md` §Core Quota.

### LB-002 · Reward UTXOs of different services are inserted into the UTXO tree in hash-trie iteration order

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Determinism |
| Target | `ledger/src/mantle/sdp/mod.rs:390-408` (`SdpLedger::try_apply_header`), `ledger/src/lib.rs:340-343` |
| Status | Open |

**Description**

`SdpLedger::services` is an `rpds::HashTrieMapSync<ServiceType, Service>` (`sdp/mod.rs:295`). `rpds` 1.2.1 defaults its hasher to `std::collections::hash_map::RandomState` (`rpds-1.2.1/src/utils/mod.rs:4`), so iteration order is per-process random. At every epoch transition `try_apply_header` iterates that map and appends each service's reward UTXOs to one vector in iteration order:

```rust
// sdp/mod.rs:390-408
let services = self.services.iter().map(|(service, service_state)| {
    ...
    all_reward_utxos.extend(reward_utxos);
```

and `ledger/src/lib.rs:340-343` inserts them into the UTXO Merkle tree in that order:

```rust
for utxo in effect.reward_utxos {
    cryptarchia_ledger.utxos = cryptarchia_ledger.utxos.insert(utxo.id(), utxo).0;
}
```

Leaf position in the UTXO tree depends on insertion order, and the tree root is the `ledger_LATEST` public input of every Proof of Leadership. Today `ServiceType` has one variant (`core/src/sdp/mod.rs:266-269`), so the vector always holds one service's notes, already sorted by `zk_id` inside `distribute_rewards` (`rewards/mod.rs:116-138`), and the order is fixed. The dependency on hash order is latent: it becomes a chain split the moment a second service type is added, with nothing at the insertion site to warn the author.

**Exploit scenario**

Not triggerable at this commit. With two services, on the first block of an epoch in which both distribute rewards, half of the nodes insert service A's notes before service B's and half the reverse; their UTXO tree roots differ, every subsequent Proof of Leadership verifies on only one side, and the network splits.

**Recommendation**
- *Short term*: sort the collected `all_reward_utxos` by `(service_type discriminant, zk_id)` before returning `HeaderEffect`, or iterate services in `ServiceType` discriminant order.
- *Long term*: keep `services` in an ordered map (`rpds::RedBlackTreeMapSync`) so every iteration over it is canonical; audit the other `HashTrieMapSync` iterations in the ledger (`declarations.iter()` at `sdp/mod.rs:236-265` only collects ids to remove, which is order-independent) with the same rule.

**References**: `bedrock-service-reward-distribution.md` §Service Reward Distribution ("executed identically by every node ... inserting notes in the ledger in ascending order of `zk_id`" is specified per service only).

### LB-003 · Fork choice iterates chain tips in hash-randomised order, so ties between competing forks are not first-seen

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Consensus |
| Target | `consensus/cryptarchia-engine/src/lib.rs:92` and `:129` (`maxvalid_bg`, `maxvalid_mc`), `consensus/cryptarchia-engine/src/lib.rs:270-272` (`Branches::branches`) |
| Status | Open |

**Description**

`Branches::tips` is an `rpds::HashTrieSetSync<Id>` with `RandomState`. Both fork choice rules walk it:

```rust
// lib.rs:129-138
let forks = branches.branches();          // self.tips.iter()
for chain in forks {
    ...
    if m <= k && cmax.length < chain.length { cmax = chain; }
}
```

`cmax` starts at the local chain, so an equal-length fork never displaces the local tip (the strict `<` in the specification). But when two or more forks are both longer than the local chain and equal to each other, the first one in iteration order wins, and that order is a function of the process's random hash seed. The same holds for equal densities in the `m > k` branch of `maxvalid_bg` (`lib.rs:105-109`). `fork-choice.md` comments the strict inequality with "to ensure we choose first-seen chain as our tie break"; first-seen would require iterating tips in arrival order, which the set does not preserve.

The node's own choice is stable once made (later evaluations start from the chosen tip), so this does not oscillate. It is a local decision and cannot make nodes disagree on validity, but it changes which competing block a restarting or freshly syncing node extends, and it makes multi-node simulations and the `tsi_simulation` test path non-reproducible across runs.

**Exploit scenario**

Not exploitable. Impact: given two competing forks of equal length both longer than the local chain (the normal state right after a node has been offline for a few slots and receives both branches), the branch adopted is random per process rather than the one received first, so the "first-seen" heuristic that lets honest nodes converge faster is not implemented.

**Recommendation**
- *Short term*: iterate tips in a canonical order, for example `tips` sorted by `(length desc, slot asc, id)` collected into a `Vec` before the loop, or store `tips` in an ordered set keyed by insertion sequence to obtain first-seen.
- *Long term*: make `Branches` generic over a deterministic hasher (`rpds` accepts `H: BuildHasher`) or use `RedBlackTreeSetSync`, so every iteration over the block tree is reproducible; the same applies to `uncle_candidates` (`lib.rs:796`), which is currently rescued by the explicit `sort_unstable_by_key` on `(parent_slot, slot, id)` at `lib.rs:740`.

**References**: `fork-choice.md` §Online Fork Choice Rule, §Bootstrap Fork Choice Rule.

## 5. Suggestions (non-security)

### S-001 · Total stake inference truncates `f` to three decimals, exactly as the specification does

| | |
|---|---|
| Category | Determinism |
| Target | `ledger/src/cryptarchia/stake.rs:32-35`, `ledger/src/cryptarchia/mod.rs:754-758` |

`StakeInference::new` receives `slot_activation_coeff().as_f64()` and `total_stake_inference` uses `trunc(f * 1000)` as `f_p`, which is what `cryptarchia-total-stake-inference.md` §Algorithm prescribes (`let f_p: u64 = truncate(f * PRECISION)`). For the deployed `f = 1/30` this makes `f_p = 33`, so the expected density is `PERIOD · 0.033` instead of `PERIOD / 30`, and the inferred stake converges to a value about 1% above the true one. The learning rate goes through the same truncation. Both are deterministic (verified over every `n/d` with `d ≤ 1000` and every `β = i/1000`: no `f64` truncation differs from the exact integer). Suggest raising upstream that the specification state `f` as the rational the configuration already holds and use `expected_density_p = PERIOD · PRECISION · f_num / f_den`; the code can then drop the two `f64` fields.

### S-002 · `Utxo::output_index` is a `usize` inside the `NoteId` preimage

| | |
|---|---|
| Category | Determinism |
| Target | `core/src/mantle/ledger.rs:489-493`, `core/src/mantle/ledger.rs:512-528` |

`Utxo::id` hashes `self.output_index.to_le_bytes()`, 8 bytes on 64-bit and 4 on 32-bit targets. It is width-independent only because `lb_groth16::fr_from_bytes` (`zk/groth16/src/lib.rs:59-68`) goes through `BigUint::from_bytes_le`, which ignores trailing zero bytes. The specification (`bedrock-v1.1-mantle-specification.md` `derive_note_id`) takes an integer, and the transaction encoding bounds `OutputCount` to a byte. Suggest storing `output_index` as `u64` (or `u8`) so the preimage does not depend on the platform word size through an incidental property of the decoder.

### S-003 · `Version` serializes its `Debug` representation in human-readable formats

| | |
|---|---|
| Category | Determinism |
| Target | `core/src/header/mod.rs:90-100` |

`serializer.serialize_str(format!("{self:?}"))` couples the JSON/YAML form of the header version to the derived `Debug` output; renaming the variant silently changes the API format while `TryFrom<&str>` (`:75-87`) keeps accepting only `"bedrock"`. Binary encoding uses `as_byte()` and is unaffected. Suggest a `const fn as_str()` used by both directions.

## 6. Ruled out

Checked and found deterministic or off the consensus path at this commit:

- `HashMap`/`HashSet`/`HashTrie*` iteration feeding hashed or agreed data: `Declarations` (`HashMap<ServiceType, HashMap<DeclarationId, Declaration>>`) is only iterated to build `providers_and_zk_root` (`current_epoch.rs:188-215`), which sorts by `zk_id` through `sort_nodes_and_build_merkle_tree` (`blend/crypto/src/merkle.rs:222-228`); `zk_id` and `provider_id` are unique per service by `validate_service_scoped_uniqueness` (`core/src/mantle/ops/sdp/declare.rs:109-131`), so equal keys cannot occur and both the Merkle leaf order and the provider index are canonical. The prover side (`services/blend/src/membership/service.rs:36-71`) builds the same sorted tree. `TargetEpochTracker::finalize` (`target_epoch.rs:185-236`) writes a `HashMap<ZkPublicKey, _>` that `distribute_rewards` (`rewards/mod.rs:116-138`) sorts by `zk_id`, matching `bedrock-service-reward-distribution.md`; overwrites are impossible by the same uniqueness. `unlock_and_remove_withdrawn_declarations` (`sdp/mod.rs:226-265`) is order-independent. `uncle_candidates` (`engine lib.rs:788-812`) walks tips in hash order but the result is sorted with a total key at `:740`. `collect_chain_within_window`, `LedgerTxs` uniqueness (`core/src/mantle/ledger.rs:342`), channel signer uniqueness (`ops/channel/verification.rs:34`), `CompressedMerkleTree` recovery (`merkle/tree/src/lib.rs:293`) use sets for membership only. `KnownBlocks.additional_blocks` (`cryptarchia-sync/src/libp2p/messages.rs:43`) and IBD `tips` (`chain-network/src/bootstrap/ibd.rs:302-325`) are transport-level sets; their order changes only download scheduling.
- `f32`/`f64`: no floats in `ledger/src/mantle`, `core/src/mantle`, `core/src/header`, `core/src/block`, `blend/message`, `blend/proofs`, `services/pow`. `LotteryConstants::new` (`zk/proofs/pol/src/lottery.rs:43-79`) uses `astro-float` at fixed 512-bit precision with `RoundingMode::ToEven` from the exact ratio, which is software arithmetic and platform-independent. `BlendingTokenEvaluation` and `activity_threshold` (`blend/message/src/reward/epoch.rs:56-77`) are integer. `blend/scheduling/src/cover_traffic.rs:92-117` and `services/blend/.../tokio_provider.rs:35-38` are local scheduling only. `core/src/proofs/leader_proof.rs:382-400` is test code.
- `usize`/`isize` in serialised data: the canonical `BinaryCodec` has no `usize` encoding; length prefixes are fixed-width from the `MAX` bound (`codec/src/bounded_vec.rs:17-72`). `serde` `usize` fields (`MerklePath::leaf_index`, `CompressedMerkleTree`, `Utxo::output_index`, sync/config settings) go through `serialize_u64` under bincode and are storage or API only; header and body hashing use `to_bytes()`/`encode_to_vec()` of the canonical codec (`core/src/block/mod.rs:355-362`, `core/src/header/mod.rs:160-184`).
- `Instant`/`SystemTime` in logic: `check_offline_grace_period` (`chain-service/src/bootstrap/state.rs:34`) and `LastEngineState.timestamp` (`states.rs:68`) decide only whether the local node bootstraps; `current_timestamp_millis` (`tx-service/src/backend/pool.rs:387`) expires local mempool entries; `cryptarchia-engine/src/time.rs:325` is the slot clock, which the specification allows to be local (`cryptarchia-v1-protocol.md` step 6, §Clocks). All other `Instant::now` hits are metrics.
- `rand::thread_rng`/`OsRng` in consensus paths: every hit in `ledger`, `consensus`, `merkle`, `services/pow` is inside `#[cfg(test)]` (`services/pow/src/tickets.rs:347`, `services/pow/src/service.rs:1538`, `ledger/src/cryptarchia/mod.rs:851`).
- Sorting with non-total `PartialOrd`: no `partial_cmp`-based sorts; `ProviderId` and `IndexedSignature` implement `Ord` byte-wise and delegate `partial_cmp` to it; `Epoch`'s `PartialOrd<u32>` is over integers.
- `Debug` formatting as a canonical string: every `format!("{:?}")` outside logging builds an error message (`kms/macros`, `ops/channel/inscribe.rs:155`, `c-bindings`, `zone-sdk`) except `Version` (S-003). No storage key or hash preimage is built from `Debug` output.
- Duplicated blend parameters: `message_frequency_per_round` and `rounds_per_epoch` appear both in `RewardsParameters` (ledger) and `CoverTrafficSettings`/`TimingSettings` (blend service), but `nodes/node/binary/src/config/cryptarchia/mod.rs:67` derives the ledger copy from the blend deployment section, so a single node cannot hold two values.
- ⚑ repo items: none listed on this issue; the head cited in the issue (`19353c619`) has moved to `a805329f`, and every location above was checked at the latter.

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
