# Audit Report — Snapshot `Declarations` ordering in blend membership

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/77`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `c3ff08e4a9cbc8344c58a18ef7992a41403c7f9c` — component(s): `services/blend/src/membership`, `blend/membership`, `blend/crypto/merkle`, `ledger/src/mantle/sdp/rewards/blend`, `core/src/sdp`
Date: `2026-09-07` — author: `claude-fable-5-1` — status: `final`

---

## 1. Summary

- Overall assessment: the ordering concern does not materialise. Both the blend service and the ledger sort the snapshot by `zk_id` with the same helper before any index is assigned, the two lists contain the same elements, and no `Declarations` byte encoding is ever hashed or compared across nodes. The review did surface one adjacent consensus problem in the shared helper: the membership Merkle tree has a hard cap of 2^20 leaves and both callers `expect` on it, so a snapshot larger than that panics every node at the epoch transition.
- Findings: 0 critical · 1 high · 0 medium · 0 low · 0 informational
- Key themes: "shared sort helper keeps ledger and blend indices aligned", "unbounded declaration count meets a fixed-size circuit tree"
- Must-fix before launch: LB-001 (cap the number of active blend declarations on chain, or fail closed without panicking).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/blend/src/membership/service.rs` L28-L119 | node list construction from the snapshot, sorting, Merkle proof, `Membership` construction |
| `services/blend/src/membership/chain.rs` L150-L164 | the single call site, per epoch |
| `services/blend/src/membership/node_id/libp2p.rs` L7-L11 | `ProviderId` → `PeerId` decoding |
| `blend/membership/src/lib.rs` L44-L150 | `Membership::new`, index assignment, `get_node_at`, `local_index`, random peer choice |
| `blend/crypto/src/merkle.rs` L13, L78-L127, L222-L228 | `sort_nodes_and_build_merkle_tree`, `MerkleTree::new_from_ordered`, leaf cap |
| `blend/provers/src/crypto/core_and_leader/{send,receive}.rs`, `leader/send.rs` | where `local_index()` / `get_node_at` feed proof of selection |
| `ledger/src/mantle/sdp/rewards/blend/current_epoch.rs` L132-L151, L188-L214 | ledger-side provider ordering and index assignment |
| `ledger/src/mantle/sdp/rewards/blend/target_epoch.rs` L98-L108 | ledger-side use of the index in activity-proof verification |
| `core/src/sdp/mod.rs` L341-L342, L427-L474 | `ProviderId`, `Declarations`, `TryFrom<Bytes>` |
| `core/src/codec/mod.rs` L22-L30 | the byte codec is `bincode` over `serde` |
| `ledger/src/cryptarchia/mod.rs` L64-L107 | `EpochState` derives |
| `services/chain/chain-service/src/service/mod.rs` L363-L380, L499-L508, L631-L707; `services/chain/chain-service/src/states.rs` L10-L28; `services/api/src/http/mantle.rs` L881-L935 | every other consumer of `Declarations` |
| `core/src/mantle/ops/sdp/declare.rs` L109-L132, L169; `kms/keys/src/keys/ed25519/public.rs` L42-L44; `utils/src/bounded/vec.rs` L33-L40 | invariants the blend side relies on (unique keys, valid keys, non-empty locators) |

**Out of scope**
The proof-of-quota and proof-of-selection circuits and their verifiers, the Poseidon Merkle hasher (`rs_merkle_tree`, `InnerTreeZkHasher`), path-selection randomness quality (#58), stale-membership routing (#62), the SDP mempool, and the timing of the snapshot freeze (covered in #53 B3). Third-party crates assumed correct: `ed25519-dalek`, `libp2p-identity`, `rs_merkle_tree`, `rpds`, `bincode`.

**Assumptions**
The ledger enforces the invariants it claims: `provider_id` and `zk_id` unique per service (`declare.rs` L109-L132), every stored `ProviderId` is a valid Ed25519 key (it must have produced a valid signature in `preverify`), and locators are non-empty by type (`NonEmptyBoundedVec` validates on deserialisation, `utils/src/bounded/vec.rs` L33-L40). `minimum_network_size ≥ 1`.

## 3. Method

- Manual review of the in-scope paths, working through sub-issue #77 under parent #8, with #14, #33 and #62 read for overlap. Every consumer of `lb_core::sdp::Declarations` and of `EpochState.active_declarations` outside test code was enumerated with `grep` and read; the table below records each one.
- Spec conformance: `MerkleTree::new_from_ordered` cites the PoQ specification for the sorted-leaf order (`merkle.rs` L85); the spec text itself was not consulted.
- Automated tooling: none.
- Dynamic testing: none. Existing unit tests in `blend/membership/src/tests.rs` and `blend/crypto/src/merkle.rs` L230+ were read, not run.

Consumers of the snapshot and whether position matters:

| Consumer | Iterates | Sorted before use? | Position-dependent output |
|---|---|---|---|
| `membership_info_from_epoch_state`, `services/blend/src/membership/service.rs` L36-L48 | `declarations.values()` | yes, L53-L56 (`sort_nodes_and_build_merkle_tree`, key `zk_id`) | `Membership.node_indices` (L72-L76 → `blend/membership/src/lib.rs` L46-L53), Merkle root and local leaf proof (L57-L70) |
| `CurrentEpochTracker::providers_and_zk_root`, `ledger/.../current_epoch.rs` L188-L214 | `declarations.values()` (L132-L134, L148-L151) | yes, L196-L199, same helper and key | provider index `i` (L201-L212), `zk_root` |
| `Query::GetSdpSnapshot`, `chain-service/src/service/mod.rs` L363-L380 | `iter()` over both maps | no | `HashMap<DeclarationId, Declaration>` returned as JSON (`mantle.rs` L909-L935) |
| `log_epoch_state_query` L499-L508, `log_blend_snapshot_provider_decisions` L631-L707 | `iter()` / `values()` | no | log lines only |
| `CryptarchiaConsensusState.lib_ledger_state`, `states.rs` L10-L28 | serde | no | on-disk `bincode` of the LIB state, reloaded, never hashed |
| `EpochState: PartialEq` (`cryptarchia/mod.rs` L64) | — | — | `HashMap` equality is order-independent |
| `Declarations: TryFrom<Bytes>` / `TryFrom<Declarations> for Bytes`, `core/src/sdp/mod.rs` L460-L474 | serde | no | no call site outside `core` |

Checked and ruled out:

| Check | Evidence | Result |
|---|---|---|
| Blend index ≠ ledger index for the same provider | both lists are built from the same `active_declarations` map for the same epoch (blend: the `EpochState` returned by the chain query, `chain.rs` L150-L164; ledger: `last_epoch_state` at the transition, `current_epoch.rs` L132) and sorted by `zk_id.into_inner()` with one stable `sort_by_key` (`merkle.rs` L226); `zk_id` is unique per service, so there are no ties | holds |
| Blend drops an element the ledger keeps | `node_from_provider` (`service.rs` L86-L119) only drops a declaration if `PeerId::from_public_key` or `Ed25519PublicKey::from_bytes` fails on the 32 key bytes; both go through `ed25519_dalek::VerifyingKey::from_bytes` (`node_id/libp2p.rs` L7-L11, `public.rs` L42-L44), and a `ProviderId` that fails there could never have signed its `Declare` | holds |
| `expect("Locators set cannot be empty")` (`service.rs` L97-L100) reachable | `Locators = NonEmptyBoundedVec<Locator, 8>`; `Bounded<Vec<T>, MIN, MAX>` rejects out-of-range lengths on deserialisation (`utils/src/bounded/vec.rs` L33-L40) | unreachable |
| `assert!` on duplicate node in `Membership::new` (`lib.rs` L49-L52) reachable | `NodeId` is derived from `provider_id`, unique per service; only `BlendNetwork` is read (`service.rs` L38) | unreachable |
| Proof of selection verified against a different index than the sender used | sender: `expected_index(membership_size)` → `get_node_at` (`core_and_leader/send.rs` L235-L250, `leader/send.rs` L193-L211); receiver: `local_index()` (`receive.rs` L71, L82-L88); ledger: `providers.get(provider_id)` index (`target_epoch.rs` L98-L108). All three read the same sorted order | holds |
| Random peer choice biased by map order | `filter_and_choose_remote_nodes` samples uniformly with `choose_multiple` over `node_indices` (`lib.rs` L92-L113); order of the candidate list does not change the distribution, and the `HashMap` behind `core_nodes` is only used for lookups | holds |
| Any `Declarations` byte string hashed, signed, or compared across nodes | consumer table above; no call site of the `Bytes` conversions; the LIB state on disk is deserialised, never compared | none |

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | More than 2^20 active blend declarations panics every node at the epoch transition | Denial of Service | High | Medium | Open |

### LB-001 · More than 2^20 active blend declarations panics every node at the epoch transition

| | |
|---|---|
| Severity | High |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | `ledger/src/mantle/sdp/rewards/blend/current_epoch.rs:L148-L151, L196-L199` (`CurrentEpochTracker::finalize`, `providers_and_zk_root`); `blend/crypto/src/merkle.rs:L13, L90-L96` (`MerkleTree::new_from_ordered`); `services/blend/src/membership/service.rs:L53-L56` |
| Status | Open |

**Description**
The membership Merkle tree has the fixed height of the PoQ circuit, so it holds at most `1 << 20` keys:

```rust
// blend/crypto/src/merkle.rs
13  const TOTAL_MERKLE_LEAVES: usize = 1 << CORE_MERKLE_TREE_HEIGHT;   // height 20, zk/proofs/poq/src/blend_inputs.rs L4
94  if keys.len() > TOTAL_MERKLE_LEAVES {
95      return Err(Error::TooManyKeys);
96  }
```

Both callers turn that error into a panic. On the ledger side it runs inside header application at every epoch transition, for every node:

```rust
// ledger/src/mantle/sdp/rewards/blend/current_epoch.rs
196 let zk_root =
197     sort_nodes_and_build_merkle_tree(&mut providers, |(_, zk_id)| zk_id.into_inner())
198         .expect("Should not fail to build merkle tree of core nodes' zk public keys")
199         .root();
```

and on the blend side at the start of every epoch (`service.rs` L53-L56, same `expect`). Nothing bounds the number of declarations that reach the snapshot: `SDPDeclare` only requires a note of at least `min_stake` and unique `provider_id`/`zk_id` per service (`declare.rs` L62-L67, L109-L132); the deployment template sets `min_stake.threshold: 1` (`nodes/node/binary/src/config/deployment/settings.yaml` L82-L84); the `Declare` op costs 646 execution gas at a genesis price of 1 (`declare.rs` L169, `core/src/mantle/transactions/gas.rs` L9-L12) plus storage gas. A declaration created in epoch `C` enters the snapshot at `C+2` and stays for `inactivity_period` epochs without any further transaction (`is_active`, `ledger/src/mantle/sdp/mod.rs` L278-L286).

Well below the cap the same code is already a cost multiplier: every node rebuilds an `n`-leaf Poseidon tree twice per epoch (ledger and blend), and `validate_service_scoped_uniqueness` scans all `n` declarations for each `Declare` in a block (`declare.rs` L114-L131), so block verification is quadratic in the declaration count.

**Exploit scenario**
An attacker funds 1,048,577 notes of one unit each, generates that many Ed25519 and ZK key pairs, and submits one `SDPDeclare` per note over some epochs, keeping each declaration within `inactivity_period` of its creation so that all of them are in the snapshot for the same epoch `E` (re-declaring after lapse is possible since the note is only locked, not spent). At the first slot of `E`, `membership_info_from_epoch_state` panics in every blend service. At the first block of `E+1`, `CurrentEpochTracker::finalize` panics inside `try_apply_header` on every node; restarting re-applies the same block and panics again. The chain cannot advance past that block without a code change. Cost is roughly 10^6 transactions worth of gas and 10^6 units of stake, which is why this is rated High rather than Critical; if the smallest unit is cheap relative to fees the rating should be raised.

**Recommendation**
- *Short term*: reject `SDPDeclare` for a service once the number of live declarations reaches a configured maximum well under 2^20 (a new `SdpError`), so the cap is enforced at the transaction boundary, and replace both `expect`s with an explicit fallback (`WithoutTargetEpoch` on the ledger side, empty membership on the blend side) so an oversized snapshot degrades instead of halting.
- *Long term*: make `min_stake` and the declaration cap part of the economic design (a cap of, say, 2^16 with a stake that makes filling it cost more than the network is worth), and add a test that builds the tree at the cap and at cap+1.

**References**: none.

## 5. Suggestions (non-security)

### S-001 · Give `Declarations` a canonical encoding, or mark it as never hashable

| | |
|---|---|
| Target | `core/src/sdp/mod.rs:L427-L428, L460-L474` |

`Declarations` wraps two `HashMap`s and derives `Serialize`, so `to_bytes()` (`bincode`, `core/src/codec/mod.rs` L22-L30) yields a different byte string on every process for the same set. Today nothing hashes or compares those bytes (Method table), but the `TryFrom<Bytes>` / `TryFrom<Declarations> for Bytes` impls invite it, and #33's sweep exists because this pattern has bitten before. Either switch the inner maps to `BTreeMap` (keys are `ServiceType` and `DeclarationId`, both `Ord`-able; iteration cost is irrelevant at snapshot size) or remove the `Bytes` conversions and add a doc comment stating the encoding is not canonical.

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
