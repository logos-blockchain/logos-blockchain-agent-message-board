# Audit Report — Merkle, MMR, and UTXO tree: domain separation, edge cases, trusted roots, position hashing

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/51`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `c3ff08e4a9cbc8344c58a18ef7992a41403c7f9c` — component(s): `merkle/dynamic-merkle`, `merkle/tree`, `merkle/utxotree`, `merkle/blake2btree`, `mmr`, `core/src/proofs/merkle.rs`, and the ledger/wallet call sites that build or check roots and paths
Date: `2026-09-07` — author: `claude-fable-5-1` — status: `final`

Circuits: not in scope (the PoL/PoC circuits live outside this repository; see Appendix B item 7 and the follow-up issue).

---

## 1. Summary

- Overall assessment: the tree code is sound on the checklist's core questions. Fixed-depth paths, domain-tagged leaves and roots that always come from ledger state close the classic Merkle attacks. The weaknesses found are in the persistence layer: two deserialisation paths accept states that the in-memory API could never produce, and the generic tree silently corrupts itself on a duplicate key.
- Findings: 0 critical · 0 high · 0 medium · 2 low · 3 informational
- Key themes: consistency between the live tree and its serialised form; unvalidated MMR state on load; API footguns around proof length and leaf/inner-node separation.
- Must-fix before launch: none. LB-001 and LB-002 should be fixed before any code path can insert a UTXO twice or load state from an untrusted source.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `merkle/dynamic-merkle/src/lib.rs` | node model, insert/remove/update, `path`, `root`, `from_sorted_items`, serde |
| `merkle/tree/src/lib.rs` | `MerkleTree` key→position map, `compressed()`, `TryFrom<CompressedMerkleTree>`, serde |
| `merkle/utxotree/src/lib.rs` | `UtxoMerkleHasher`, `UtxoLeaf`, `UtxoTree` wrapper and serde |
| `merkle/blake2btree/src/lib.rs` | `Blake2bMerkleHasher`, `Blake2bLeaf`, `Blake2bTree` |
| `mmr/src/lib.rs`, `mmr/src/path.rs` | `MerkleMountainRange::{push, push_with_paths, frontier_root, len}`, `MerklePath::{root, verify}`, serde |
| `zk/poseidon2/src/hasher.rs` | `Digest::compress` vs `Digest::digest` construction |
| `core/src/proofs/merkle.rs`, `core/src/proofs/leader_proof.rs`, `core/src/proofs/leader_claim_proof.rs` | path→witness conversion, public inputs |
| `core/src/mantle/ledger.rs` (`Utxo::id`, `Outputs::{validate,execute}`), `core/src/mantle/ops/op.rs` (`OpId`), `core/src/mantle/ops/pow.rs` (reward UTXO), `ledger/src/mantle/sdp/rewards/mod.rs` (reward op id) | how leaves are derived and whether a key can be inserted twice |
| `ledger/src/cryptarchia/mod.rs` (`try_apply_proof`, `from_utxos`, `utxo_merkle_path`), `ledger/src/lib.rs:331-333`, `ledger/src/mantle/leader.rs`, `core/src/mantle/ops/leader_claim.rs` (`verify`) | which root each proof is checked against |
| `wallet/src/lib.rs` (`apply_voucher`), `services/wallet/src/lib.rs` (`generate_poc`), `services/chain/chain-leader/src/leadership.rs` (`build_proof_for`) | prover-side use of paths (read for root/path consistency only) |

**Out of scope**

The Circom circuits (PoL, PoC, PoQ) and their Poseidon2 Merkle gadgets: the node repository only holds the witness marshalling, so the leaf/node encoding parity item could be checked one-sided only (Appendix B item 7). Groth16 verification itself (parent #16). The blend `CORE_MERKLE_TREE_HEIGHT = 20` tree in `zk/proofs/poq` (parent #13). Fee, value, nullifier and reorg logic of the ledger (other sub-issues of #6). Third-party crates assumed correct: `ark-ff`/`ark-bn254`, `jf-poseidon2` (pinned git rev), `blake2` 0.10, `rpds` 1.2.1, `serde`.

**Assumptions**

Poseidon2 over BN254 (`Poseidon2ParamsBn3`, width 3) is collision- and second-preimage-resistant in both the `compress` (no padding, `[l, r, 0]` state) and `digest` (rate-1 sponge with `1` padding) modes. Repo-level facts from issue #19 hold at this commit: `[profile.release]` (`Cargo.toml:11-14`) sets no `overflow-checks`, so `+`/`-`/`<<` wrap in production; `arithmetic_side_effects`, `indexing_slicing`, `unwrap_used`, `expect_used` are allowed (`Cargo.toml:338,379,391,413`).

## 3. Method

- Manual review of the in-scope paths, working through issue `#51` (sub-issue of `#6`), with `#19` as context.
- Static reading via `rg`/`sed` against a detached worktree at `c3ff08e4a`. Every claim below carries a `file:line` at that commit.
- Automated tooling: a 40-line scratch program against `merkle/blake2btree` and `merkle/tree` (as path dependencies, `rustc 1.94.1`) to reproduce LB-001. Output is quoted in the finding. The workspace's own unit tests were not run.
- Dynamic testing: none against a running node.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | `MerkleTree::insert` with an existing key orphans the old leaf; root and `size()` diverge from the serialised form | Data Validation / Determinism | Low | High | Open |
| LB-002 | `MerkleMountainRange` deserialisation accepts any `roots` stack; heights of 0 or above the maximum panic on first use | Data Validation / Denial of Service | Low | High | Open |
| LB-003 | `lb_mmr::MerklePath::verify` accepts a path of any length and so "verifies" inclusion under an inner node | Data Validation | Informational | — | Open |
| LB-004 | No explicit leaf/inner-node domain separation in any tree; safety rests on fixed-depth paths and tagged leaf digests | Cryptography | Informational | — | Open |
| LB-005 | Direct `DynamicMerkleTree` deserialiser recurses before bounding depth | Denial of Service | Informational | — | Open |

### LB-001 · `MerkleTree::insert` with an existing key orphans the old leaf; root and `size()` diverge from the serialised form

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Data Validation / Determinism |
| Target | `merkle/tree/src/lib.rs:174-185` (`MerkleTree::insert`), `:229-237` (`compressed`), `:207-222` (`remove`) |
| Status | Open |

**Description**

`MerkleTree` keeps two structures that must agree: the `DynamicMerkleTree` of leaf hashes and the `items` map from key to `(item, position)`. `insert` never checks whether the key is already present:

```rust
// merkle/tree/src/lib.rs:174-185
pub fn insert(&self, key: Key, item: Item) -> (Self, usize) {
    let (merkle, pos) = self.merkle.insert(Leaf::leaf(&key, &item));   // new leaf at next free slot
    let items = self.items.insert(key, (item, pos));                   // overwrites the old (item, pos)
```

On a repeated key the old leaf stays in the hash tree but its position is no longer reachable from `items`. Consequences:

- `root()` commits to a leaf that `items`, `get`, `contains`, `path` and `remove` cannot see. `remove(&key)` (`:207-222`) clears only the newer position, so the orphan is permanent.
- `size()` (`:106-109`, delegates to `merkle.size()`) exceeds `items.size()`.
- `compressed()` (`:229-237`) is built from `items`, so the orphan is dropped on serialisation. `TryFrom<CompressedMerkleTree>` (`:291-324`) rebuilds a tree with a different root. `UtxoTree`'s serde goes through this path (`merkle/utxotree/src/lib.rs:227-254`), and `LedgerState` derives serde around `UtxoTree` (`ledger/src/cryptarchia/mod.rs:208-213`) for the chain-service recovery state.

Reproduced with a scratch program using `Blake2bTree` (same `MerkleTree` code path as `UtxoTree`):

```
positions 0 1; size()=2 items=1
serialized: {"1":[1,11]}
root before [c3, c3, 61, 4, da, c5, e1, 74]
root after  [1d, 1e, 7, 54, 8a, 1c, 8f, dc]
roots equal: false
after remove: size()=1 items=0 root==empty? false
```

**Exploit scenario**

Not reachable from network input at this commit. A UTXO key is `NoteId = Poseidon2(NOTE_ID_V1, op_id, output_index, value, pk)` (`core/src/mantle/ledger.rs:512-530`) and `op_id = blake2b("OPERATION_ID_V1" || op_bytes)` (`core/src/mantle/ops/op.rs:29-37`). Inserting the same key twice therefore needs the same operation applied twice with the same output index. Every producer of UTXOs prevents that: transfers remove their inputs before adding outputs (`core/src/mantle/ops/transfer.rs:158-162`, `core/src/mantle/ledger.rs:367-371` then `:175-180`), PoW reward claims record a nullifier and require the block hash to be inside the acceptance window (`core/src/mantle/ops/pow.rs:328-345`, `ledger/src/mantle/pow/mod.rs:114`), and SDP reward UTXOs use an `op_id` derived from `(epoch, service_type)` with one output index per recipient (`ledger/src/mantle/sdp/rewards/mod.rs:100-136`).

The actual impact is latent: the first future code path that re-inserts a key (a replayed genesis output, an op type whose id does not cover all inputs, a bug elsewhere) turns into a consensus split between nodes that stayed up and nodes that restarted from recovery state, since they will disagree on `aged_root`/`latest_root` in `LeaderPublic` (`ledger/src/cryptarchia/mod.rs:503-511`). Nothing panics, nothing logs.

**Recommendation**
- *Short term*: make `insert` return `Err` (or route through `update`) when `items.contains_key(&key)`; add a `debug_assert_eq!(self.merkle.size(), self.items.size())` invariant in `insert`/`remove`.
- *Long term*: a property test that serde round-trips preserve `root()` for arbitrary insert/remove/duplicate sequences; consider making `MerkleTree` hold `items` as the single source of truth for occupancy.

**References**: `merkle/tree/src/lib.rs:275-281` (`RecoveryError::DuplicateKey`) shows the deserialiser already treats duplicates as invalid; the live API should match.

### LB-002 · `MerkleMountainRange` deserialisation accepts any `roots` stack; heights of 0 or above the maximum panic on first use

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Data Validation / Denial of Service |
| Target | `mmr/src/lib.rs:53-70` (`Deserialize`), `:90-95` (`Root`), `:266-273` (`num_leaves`, `len`), `:231-259` (`frontier_root`), `:281-292` (`empty_subtree_root`) |
| Status | Open |

**Description**

The MMR is stored as a stack of `Root { root: Fr, height: u8 }`. The deserialiser validates only the const generic `MAX_HEIGHT`, then takes the stack as-is:

```rust
// mmr/src/lib.rs:60-69
validate_supported_max_height(MAX_HEIGHT).map_err(serde::de::Error::custom)?;
let MerkleMountainRangeSerde { roots } = ...deserialize(deserializer)?;
Ok(Self { roots, _hash: PhantomData })
```

Neither the per-peak `height` range (`1..=MAX_HEIGHT`) nor the strictly-decreasing height order that `push` (`:121-154`) maintains is checked. With a `height` of 0, `num_leaves` (`:266-268`, `1 << (height - 1)`) underflows: a panic under overflow checks, `1 << 255` masked to `1 << 63` in release. With a height above 33, `frontier_root` hits `assert!(height <= MAX_HEIGHT)` (`:251`) or the `assert_acceptable_height` inside `empty_subtree_root` (`:284`). A non-monotone stack yields a wrong root silently. By contrast, both `DynamicMerkleTree` (`merkle/dynamic-merkle/src/lib.rs:647-671`) and `CompressedMerkleTree` (`merkle/tree/src/lib.rs:291-324`) validate on load.

**Exploit scenario**

The MMR is deserialised as part of `LeaderState` (`ledger/src/mantle/leader.rs:18-36`) inside the chain-service recovery state, and inside the wallet service's persisted state (`services/wallet/src/states.rs:200-262`), both loaded through `RecoveryOperator::try_load` (`services/utils/src/overwatch/recovery/operators.rs:48-52`) from local storage. An attacker needs write access to the state directory, which already implies a compromised host, so the realistic impact is a corrupted or hand-edited recovery file that makes the node panic at the first epoch transition (`snapshot_vouchers`, `leader.rs:146-153`) rather than fail at load with a clear error.

**Recommendation**
- *Short term*: in `Deserialize`, reject any `Root` whose height is outside `1..=MAX_HEIGHT` and any stack whose heights are not strictly decreasing from bottom to top.
- *Long term*: a `TryFrom<Vec<Root>>` constructor as the single validated entry point, shared by serde and any future sync path.

**References**: none.

### LB-003 · `lb_mmr::MerklePath::verify` accepts a path of any length and so "verifies" inclusion under an inner node

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Data Validation |
| Target | `mmr/src/path.rs:66-85` (`MerklePath::root`, `MerklePath::verify`) |
| Status | Open |

**Description**

`root` folds over `self.siblings` whatever its length (0 to 32 after `deserialize_bounded_sequence`, `:128-142`); `verify` compares the result with `expected_root`. A path with `k < 32` siblings therefore proves inclusion under the height-`k+1` node on its way to the root, and an empty path "verifies" any leaf against itself. The sibling count is enforced only by the two other consumers: `push_with_paths` calls `validate_for_height` (`mmr/src/lib.rs:170-173`) and `mmr_path_to_witness` requires exactly `VOUCHER_MERKLE_PATH_LEN = 32` (`core/src/proofs/merkle.rs:34-41`). At this commit no production code calls `verify` or `root` (the only `.siblings()` users are `core/src/proofs/merkle.rs:34-36` and a wallet test), so this is an API hazard, not a live bug.

**Exploit scenario**

None today. If a future light-client, wallet or API endpoint used `verify` on a client-supplied path, an attacker could present a short path against an intermediate node whose value they can compute from public chain data.

**Recommendation**
- *Short term*: make `verify` and `root` take the tree height (or a const generic) and return `false`/`Err` unless `siblings.len() == MAX_HEIGHT - 1`.
- *Long term*: keep `MerklePath` bound to its `MerkleMountainRange<_, _, MAX_HEIGHT>` type so the check is structural.

**References**: none.

### LB-004 · No explicit leaf/inner-node domain separation in any tree; safety rests on fixed-depth paths and tagged leaf digests

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Cryptography |
| Target | `merkle/utxotree/src/lib.rs:29-35,52-54`; `mmr/src/lib.rs:13,126-129,136`; `merkle/blake2btree/src/lib.rs:34-41` |
| Status | Open |

**Description**

- UTXO tree: the leaf is the raw `NoteId` field element (`UtxoLeaf::leaf` returns `*key.as_ref()`, `merkle/utxotree/src/lib.rs:52-54`); inner nodes are `Poseidon2::compress([left, right])` (`:31-33`); an empty position is `Fr::ZERO` (`:29`).
- Voucher MMR: the leaf is the raw `VoucherCm` (`mmr/src/lib.rs:126-129`), inner nodes `compress([left, right])` (`:136`), empty subtree seed `Fr::ZERO` (`:13`).
- `Blake2bTree`: inner nodes are `blake2b-256(left || right)` over 64 bytes (`merkle/blake2btree/src/lib.rs:36-41`); the leaf is whatever `LeafHash` returns, so an implementor hashing 64 bytes would make leaves and inner nodes indistinguishable. This tree has no users outside its own tests (S-001).

No `0x00`/`0x01` prefix or distinct compression function separates the two layers. The CVE-2012-2459 class of attack (a "leaf" that is really an inner node, or a duplicated last leaf) needs the verifier to accept variable-depth proofs. It does not here: `MerklePath<T>` is `[MerkleNode<T>; 32]` at type level (`merkle/dynamic-merkle/src/lib.rs:693`), the PoL circuit consumes exactly 32 levels (`zk/proofs/pol/src/wallet_inputs.rs:6-11`), and the PoC witness requires 32 siblings (`core/src/proofs/merkle.rs:34-41`). A forged membership would need a second preimage of an inner node under `compress`, and leaves are outputs of tagged digests (`NOTE_ID_V1`, `core/src/mantle/ledger.rs:497-530`; `VoucherCm` is produced inside the PoL circuit and only enters the ledger through a verified proof, `ledger/src/lib.rs:323-328`). The `compress` and `digest` modes differ structurally (`zk/poseidon2/src/hasher.rs:31-36,40-46`: sponge with `1` padding vs. raw two-input permutation), so a leaf cannot be re-interpreted as a node output.

Residual: a leaf whose value is exactly `Fr::ZERO` is indistinguishable from an empty slot. That needs a digest collision on zero and cannot be forced.

**Exploit scenario**

None at the current parameters. The observation matters if the tree height ever becomes variable, if `Blake2bTree` is adopted with a 64-byte leaf preimage, or if a leaf type is introduced that is not a digest.

**Recommendation**
- *Short term*: document the invariant (fixed depth, tagged leaves) at the `MerkleHasher` trait and in the spec section the circuits follow.
- *Long term*: when the circuits are next regenerated, consider a leaf domain tag (e.g. `compress([LEAF_TAG, leaf])`) so that safety does not depend on every caller keeping depth fixed.

**References**: CVE-2012-2459; `merkle/dynamic-merkle/src/lib.rs:41-43` (`TREE_HEIGHT_EXCEPT_ROOT = 32`).

### LB-005 · Direct `DynamicMerkleTree` deserialiser recurses before bounding depth

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Denial of Service |
| Target | `merkle/dynamic-merkle/src/lib.rs:83-104` (`Node` derives `Deserialize`), `:166-193` (`rebuild_and_validate`), `:647-671` |
| Status | Open |

**Description**

`Node<Hash>` is a recursive `enum` with `Arc<Self>` children and a derived `Deserialize`. The height bound (`height > TREE_HEIGHT_EXCEPT_ROOT` rejected, `:172-174,189-191`) is applied by `rebuild_and_validate` after the whole node graph has been built, and `rebuild_and_validate` itself recurses. A serialised chain of nested `Inner` nodes deeper than the stack allows overflows before any check runs. Only the benchmark `tests/benches/voucher.rs:23,60` uses `DynamicMerkleTree` directly; every ledger and wallet type goes through `CompressedMerkleTree` (`merkle/tree/src/lib.rs:327-376`), which is a flat map and is not affected.

**Exploit scenario**

None: no untrusted input reaches this deserialiser at this commit.

**Recommendation**
- *Short term*: none required; if the direct form is ever persisted or transmitted, add a depth counter to the visitor or deserialise via `Raw` into the compressed form.

**References**: none.

## 5. Suggestions (non-security)

### S-001 · `merkle/blake2btree` is compiled into the workspace but has no users

`lb-blake2btree` is declared at `Cargo.toml:53,109` and referenced by no crate outside `merkle/` (`rg lb_blake2btree|Blake2bTree` finds only its own tests). Either remove it or mark it clearly as experimental; a second tree family with a different leaf contract is one more surface to keep in sync with LB-004.

### S-002 · `Outputs::validate` comment promises a duplicate check it does not perform

`core/src/mantle/ledger.rs:165-172`: the comment reads "Check that there is no duplicate" but the loop only rejects zero-value notes. Duplicate `Note`s in one op are harmless (each gets its own `output_index`), so fix the comment rather than the code.

### S-003 · Capacity arithmetic assumes a 64-bit `usize`

`1usize << TREE_HEIGHT_EXCEPT_ROOT` (`merkle/tree/src/lib.rs:292`, `merkle/dynamic-merkle/src/lib.rs:129,573`) and `1 << (33 - 1)` in `mmr/src/lib.rs:266-268` do not fit a 32-bit `usize`: a compile-time overflow error where const-evaluated, a panic or `0` otherwise, after which `DynamicMerkleTree::insert` (`:438-441`) fails its capacity assertion on the first insert. No 32-bit target is configured in CI (`rg armv7|i686|wasm32` finds nothing), so this is a portability note for the C bindings (#67): either state `target_pointer_width = "64"` as a requirement with a `compile_error!`, or use `u64` for positions and capacities. Positions are also serialised as `usize` in `CompressedMerkleTree` (`merkle/tree/src/lib.rs:330`).

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

## Appendix B — Checklist items verified

| # | Item | Result | Evidence |
|---|---|---|---|
| 1 | Leaf vs internal node domain separation (second-preimage / CVE-2012-2459) | No explicit tag; not exploitable at fixed depth 32 with digest leaves | LB-004 |
| 2 | Empty tree root | Well defined and consistent between the two implementations | `DynamicMerkleTree::root` on an `Empty` root returns `empty_subtree_root(32)` (`merkle/dynamic-merkle/src/lib.rs:503-511`); `frontier_root` of an empty MMR returns `empty_subtree_root(MAX_HEIGHT)` (`mmr/src/lib.rs:234-240`); both equal the root of a full tree of `Fr::ZERO` leaves, which is also what a partially filled tree pads with (`:242-255`). A genesis with no UTXOs (`from_utxos`, `ledger/src/cryptarchia/mod.rs:738-742`) therefore has a valid, non-degenerate root. |
| 3 | Proof with out-of-range index | Rejected or unreachable at every entry point | Tree: positions are only ever assigned by `insert` (`merkle/dynamic-merkle/src/lib.rs:437-455`) and looked up through `items` (`merkle/tree/src/lib.rs:139-142`); the internal `path` asserts `index < capacity` (`:324-329`); recovery rejects `position >= 2^32` (`merkle/tree/src/lib.rs:294-300`), duplicate keys (`:301-305`) and duplicate positions (`:362-369`). MMR: `push_with_paths` rejects tracked paths with `leaf_index >= len()` (`mmr/src/lib.rs:172`, `path.rs:52-64`); `mmr_path_to_witness` rejects `leaf_index >= 2^32` (`core/src/proofs/merkle.rs:42-49`). |
| 4 | Proof length vs tree depth | Enforced at the type level for the UTXO tree; enforced by both production consumers for the MMR; not enforced by `MerklePath::verify` | `MerklePath<T> = [MerkleNode<T>; 32]` (`merkle/dynamic-merkle/src/lib.rs:693`), `path` returns `None` rather than a longer array (`:334-336,342-344`); circuit constants are 32 (`zk/proofs/pol/src/wallet_inputs.rs:6-9`, `zk/proofs/poc/src/wallet_inputs.rs:4`); MMR siblings bounded to 32 on deserialise (`mmr/src/path.rs:128-142`), exact count checked in `validate_for_height` (`:39-49`) and `mmr_path_to_witness` (`core/src/proofs/merkle.rs:34-41`). Gap: LB-003. |
| 5 | Verification uses caller-supplied root only when trusted | Roots always come from ledger state; proof-carried roots are compared, never used | PoL: `LeaderPublic` is built from `self.aged_utxos().root()` and `self.latest_utxos().root()` of the parent state (`ledger/src/cryptarchia/mod.rs:503-511`), the proof carries no root. PoC: `LeaderClaimOp.rewards_root` must equal `context.claimable_vouchers_root` (`core/src/mantle/ops/leader_claim.rs:238-240`), which is the ledger's epoch snapshot (`ledger/src/mantle/leader.rs:144-155`), and the verifier input is built from the context value, not the op (`:244-248`). The genesis proof is compared field-by-field against a constant (`core/src/proofs/leader_proof.rs:191-197`). The prover side reads the path and the root from the same `UtxoTree` snapshot (`services/chain/chain-leader/src/leadership.rs:63-103`, `ledger/src/cryptarchia/mod.rs:198-200`). |
| 6 | `dynamic-merkle`: insertion order determinism | Deterministic; live mutation and sparse recovery produce the same canonical structure | Insertion always takes the lowest free index (`first_empty_index`, `merkle/dynamic-merkle/src/lib.rs:132-148`); empty siblings collapse on removal (`new_inner`, `:195-230`) so a remove/re-insert restores the root; `from_sorted_items` (`:535-601`) is fed from a `BTreeMap` ordered by position (`merkle/tree/src/lib.rs:308-314`) and yields the same shape (tests `test_mutation_and_sparse_recovery_have_same_structure`, `test_remove_and_reinsert_restores_root`). The root does depend on the order in which UTXOs were first inserted, which is the block's operation order and hence part of consensus. `MerkleTree::iter` order is unspecified (`:130-134`) but only feeds commutative sums. Exception: LB-001. |
| 7 | Hash of `usize` positions | Positions are never hashed; only used as left/right selectors | Tree: side chosen by index comparison against subtree capacity (`merkle/dynamic-merkle/src/lib.rs:331-346`). MMR: `is_left_child` is a bit test (`mmr/src/path.rs:223-225`); selectors are derived from `leaf_index` in `mmr_path_to_witness` (`core/src/proofs/merkle.rs:52-55`). The only `usize` that enters a hash is `Utxo.output_index` inside `NoteId` via `to_le_bytes` (`core/src/mantle/ledger.rs:517-518`); `fr_from_bytes` parses little-endian of any length (`zk/groth16/src/lib.rs:59-67`), so a 4-byte and an 8-byte encoding of the same index give the same field element. Not verified: that the circuits' Poseidon2 leaf/node encoding, `[left, right]` ordering, zero padding and the root-to-leaf selector order assumed by `merkle_path_to_witness` (`core/src/proofs/merkle.rs:12-20`) match; the circuits repository was not available. Filed as a follow-up issue. |
| 8 | (#19 context) wrapping arithmetic in release | One site affected by shape, none exploitable | `mmr/src/lib.rs:266-268` (`height - 1`, reachable only through LB-002); `+ 1` on heights in `dynamic-merkle` is bounded by the fixed height; `left_subtree_size + right_subtree_size` (`:121`) is bounded by `2^32`. |
