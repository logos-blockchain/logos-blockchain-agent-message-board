# Audit Report — Conformance fixtures pinning the Mantle and Cryptarchia §Test Vectors, and the `CLAIM_POW_REWARD` row

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/641`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `85a1620805e8b5697728a22abb9fbe6760145c21` — component(s): `core/src/mantle/ops`, `core/src/header`, `core/src/block`, `core/src/sdp`, `core/src/utils/merkle.rs`
Specs: `https://github.com/logos-co/logos-lips` @ `6637c791cf29985251bf67f73766f97c7512f824` — read: `bedrock-v1.1-mantle-specification.md` (in full), `mantle-transaction-encoding.md` (in full), `network-wire-format.md` (in full), `cryptarchia-v1-protocol.md` (§Block ID, §Block Header, §Block Header Validation, §Test Vectors), `bedrock-v1.1-block-construction.md` (§Hash, §Block Proposal, §Header, §Proof of Leadership, §Canonical Encoding, §Block), `bedrock-service-declaration-protocol.md` (§Service Types, §Identifiers, §Locators, §Declaration Message, §Declaration Storage, §Identifier Uniqueness), `proof-of-work.md` (§Protocol), `analysis-gas-cost-determination.md` (§Claim PoW Reward); core specs `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`
Date: 2026-09-22 — author: `claude-fable-5-1` — status: `final`

---

## 1. Summary

- Overall assessment: every published vector of the Mantle §Test Vectors (10 `op_id`, 2 transaction hashes) and of the Cryptarchia §Test Vectors (10 transaction leaves, transaction root, `body_root` with and without uncles, the two carried uncle headers with their signatures, and the `block_id`) reproduces byte-for-byte at `85a16208`, both through the crate codec and through an independent Python recomputation. The one exception is unchanged: the `declaration_id` vector still fails because `DeclarationMessage::id` hashes ASCII `"BN"` (#155 LB-001, issue #538; re-confirmed as #245 LB-001, issue #718). The node still commits no test that pins any of these values; its three `#[ignore]`d generators regenerate them, and one of them has already drifted away from the published vector without any signal (LB-001). This report delivers the committed fixture the issue asks for: two `cfg(test)` modules, 407 lines, that decode every published payload with the crate codec, re-encode it byte-exact and recompute its identifier. At `85a16208` they pass 7/8 (the failing one is the `declaration_id` regression guard) and go 8/8 with the one-line #155 fix applied; they are `cargo clippy` clean under the CI `CARGO_BUILD_WARNINGS=deny` setting and `rustfmt` clean under the CI nightly pin.
- Findings: 0 critical · 0 high · 0 medium · 0 low · 1 informational
- Key themes: conformance-test coverage; generators that move in lockstep with the code and therefore cannot fail; a spec table one opcode behind the node.
- Must-fix before launch: none from this report. The `declaration_id` fix (#538 / #718) remains the one open deviation these vectors expose, and it must land before any `declaration_id` is persisted anywhere that survives a reset.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `core/src/mantle/ops/mod.rs` | the `#[ignore]`d generators `generate_op_id_test_vectors` (L245) and `generate_mantle_tx_hash_test_vectors` (L270) and their `sample_ops` (L109) |
| `core/src/mantle/ops/op.rs`, `ops/pow.rs`, `ops/**` | `Op` opcode dispatch and per-op canonical codecs; `OpId` preimage (L30-36); the `CLAIM_POW_REWARD` op (`pow.rs`) |
| `core/src/mantle/transactions/tx_list/ops.rs`, `transactions/codec.rs` | `Ops` codec and `hash()`; the only committed `SignedOps` byte vector (`codec.rs` L146-188) |
| `core/src/header/mod.rs` | `Header` codec, `Header::id()` (block id preimage), `Header::sign` (L227-228), the `#[ignore]`d `generate_body_root_test_vectors` (L653) and its `one_tx_per_op` (L515) |
| `core/src/block/mod.rs`, `block/uncle.rs` | `body_root` (L381), `verify_header_signature` (L370), `SignedHeader` / `UncleHeaders` codecs |
| `core/src/utils/merkle.rs` | `calculate_transactions_root` (L14) |
| `core/src/sdp/mod.rs` | `DeclarationMessage::id` (L508-524), `ServiceType` and `Locator` codecs |

Every value was checked against the vectors published in `bedrock-v1.1-mantle-specification.md` §Test Vectors (rev 1.15.0) and `cryptarchia-v1-protocol.md` §Test Vectors at the pinned `logos-lips` commit.

**Out of scope**

Validation and execution semantics of the operations; the `SignedMantleTx` proof encodings beyond what the codec vector in `codec.rs` covers (the spec publishes no signed-transaction vector, see S-002); the ZK circuits; mempool, networking, storage. Third-party crates assumed correct: `blake2`, `ed25519-dalek` (through `lb-key-management-system-keys`), `jf-poseidon2`, `ark-*`, `multiaddr`, `bincode`, `serde`, `hex`.

**Assumptions**

The specs at the pinned `logos-lips` commit are the reference. `Hasher` is BLAKE2b-256 and `merkle_root` is the unkeyed BLAKE2b binary tree with zero-leaf padding, both confirmed by the passing vectors. The Cryptarchia vectors are stated to have been generated at node commit `e14c7fd1`; this report shows they still hold at `85a16208`.

## 3. Method

- Manual review of the in-scope paths, working through issue #641 (follow-up of #245, S-002 and S-003) under parent #10, with the repo-level facts of #19 (release profile without overflow checks, `unwrap_used`/`indexing_slicing`/`panic` allowed, `restriction` group otherwise on with `CARGO_BUILD_WARNINGS=deny` in CI) carried into the review of the fixture itself.
- Spec conformance against the documents listed in the header. The Mantle spec and the encoding spec were read in full before any code; the Cryptarchia, block-construction and SDP specs by the sections named, which are the ones defining the header wire order, the `block_id` preimage order (deliberately different from the wire order, and both normative), `body_root`, the Merkle root, the `Locators` production and the `declaration_id` preimage.
- Independent recomputation, Python 3 `hashlib.blake2b(digest_size=32)` plus `cryptography`'s Ed25519, of every published value from its published preimage: the 10 `op_id`s, both tx hashes (and that the one-of-each payload is `count || (opcode || payload)*` of the `op_id` rows), the `declaration_id` (from the spec's `0x00`-prefixed preimage, and from the `"BN"`-prefixed one the code uses), the 10 leaves, both roots, both `body_root`s, the `block_id`, and the two uncle signatures (verified over the 297-byte header under the key derived from the seed byte). All reproduce (Appendix A).
- Dynamic testing on the scratch checkout of the node at `85a16208` (`rustc 1.98.1` per `rust-toolchain.toml`, `cargo-clippy` of the same toolchain, `rustfmt` from `nightly-2026-07-05` per `code-check.yml`): added the two fixture modules of Appendix B, ran `cargo test -p logos-blockchain-core --lib` (310 tests, 3 of them the ignored generators: at `85a16208` 305 pass and 2 of the 8 new ones fail as described in §3.1; with the #155 fix applied to the scratch copy all 307 pass; the fix was reverted afterwards), `cargo clippy -p logos-blockchain-core --tests --all-features` with `CARGO_BUILD_WARNINGS=deny` (clean), `rustfmt --check --edition 2024` under the CI nightly (clean), and the three `#[ignore]`d generators with `--ignored --nocapture` to capture their current output. The node and spec repositories were not modified beyond that scratch checkout; nothing was pushed or opened against them.

### 3.1 Conformance matrix at `85a16208`

Decode with the crate codec → re-encode (byte-exact) → recompute the identifier → compare with the published value. "Fixture" names the test in Appendix B.

| Vector (spec) | Fixture | Result |
|---|---|---|
| Mantle `op_id` × 10 (`TRANSFER` … `LEADER_CLAIM`) | `mantle_spec_op_id_vectors` | ✅ all 10; `OpId::op_id()` agrees for the 5 ops that implement it |
| Mantle tx hash, empty transaction | `mantle_spec_tx_hash_vectors` | ✅ |
| Mantle tx hash, one of each operation (10 ops) | `mantle_spec_tx_hash_vectors` | ✅ (payload rebuilt as `0x0a || (opcode||payload)*` from the rows, decoded, re-encoded, hashed) |
| Mantle `declaration_id` | `mantle_spec_declaration_id_vector` | ❌ `019e6a82…` computed vs `7fb647c0…` published (#155 LB-001 / #245 LB-001); ✅ with the one-line fix |
| Cryptarchia leaves × 10 and transaction root | `cryptarchia_spec_transaction_leaves_and_root` | ✅; empty root `0x00…00` ✅ |
| Cryptarchia `body_root`, no uncles | `cryptarchia_spec_body_root_without_uncles` | ✅; empty list encodes as `0x00` ✅ |
| Cryptarchia `body_root`, two uncles | `cryptarchia_spec_body_root_with_two_uncles` | ✅; each 361-byte uncle re-encodes byte-exact, decodes back to the same value, and its signature verifies |
| Cryptarchia `Header` / `block_id` | `cryptarchia_spec_header_id` | ✅; the 297-byte wire form decodes to the constructed header and `Header::id()` matches |
| `CLAIM_POW_REWARD` (no spec row) | `claim_pow_reward_proposed_op_id_vector` | ✅ against the node generator's value, pinned as the proposed row (S-001) |

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | The transaction-hash generator has silently diverged from the published vector: it now emits an 11-operation transaction the spec does not have, while the header generator still builds 10 | Determinism | Informational | High | Open |

### LB-001 · The transaction-hash generator has silently diverged from the published vector: it now emits an 11-operation transaction the spec does not have, while the header generator still builds 10

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Determinism |
| Target | `core/src/mantle/ops/mod.rs:109-197` (`sample_ops`), `:270-280` (`generate_mantle_tx_hash_test_vectors`); `core/src/header/mod.rs:515-627` (`one_tx_per_op`) |
| Status | Open at `85a16208` |

**Description**

This is the failure mode #245 S-003 predicted, already realised. `sample_ops` (`ops/mod.rs:109-197`) gained an `Op::ClaimPowReward` entry when the operation was added, so `generate_mantle_tx_hash_test_vectors` now prints, under the label `"one of each operation (9 ops)"` (`:277`, stale twice over), an 11-operation transaction:

```
encoding 0b00 0201…   (count byte 0x0b, 1312 bytes)
tx_hash  e551ee978b8b45a46b8a28386d01790cd530a74bdf13b9b756a307fb9413099f
```

The spec's "Transaction with one of each operation" vector is the 10-operation transaction (`0x0a…`, 1215 bytes, hash `11e60138…`). The generator's own assertion (`assert_eq!(tx.hash().0, tx_hash)`, `:232`) compares the hand-rolled hash with `Ops::hash()` of the same object, so it passed, printed a value the spec does not contain, and nothing failed. Meanwhile `one_tx_per_op` in the header module (`header/mod.rs:515-627`), whose comment says its instances "mirror those used by the `OpId` test vectors so the two vector sets stay consistent", was not updated and still builds 10 transactions, so the two generators disagree with each other as well: the header generator reproduces the published Cryptarchia vectors exactly (Appendix A), the op generator no longer reproduces the published Mantle tx-hash vector.

Nothing is wrong with the codec: the fixture in Appendix B rebuilds the 10-operation transaction from the published rows, and the crate decodes, re-encodes and hashes it to the published value. The finding is that the only mechanism the node has for these vectors cannot detect drift in either direction, and did not.

The same generator match at `:251-259` cross-checks `OpId::op_id()` for five variants and falls through `_ => {}` for the rest, including `ClaimPowReward`, which does implement `OpId` (`pow.rs:241-245`) and is what `execute` uses to derive the reward note id (`pow.rs:338`).

**Exploit scenario**

Not a security issue. The impact is on conformance: an alternative implementation that regenerates the vectors from this generator, as its module documentation invites ("so that alternative implementations (e.g. the nim implementation) can be checked"), obtains a transaction hash for a transaction shape the spec does not publish and gets no vector for the published one; and a future change that moves any `op_id`, tx hash, leaf or `body_root` would move the generator output in lockstep and still pass CI. The concrete class this guards against is already on file: #245 LB-002 (issue #719, `op_id` preimage omits the opcode) is exactly a preimage change that, if made on one side only, would be caught by a pinned vector and by nothing else in the tree today.

**Recommendation**

- *Short term*: commit the fixture of Appendix B (two `cfg(test)` modules, no production code touched, CI-clean at `85a16208`). It fails today only on `mantle_spec_declaration_id_vector`; either land it after the #155 fix, or land it now with that one test marked `#[should_panic(expected = "declaration_id differs from the spec")]` and a comment naming #538, so the guard flips to a failure the day the fix lands and the marker is removed. Fix the `"(9 ops)"` label, and make the `OpId` cross-check exhaustive over the variants that implement it.
- *Long term*: have the spec publish the `CLAIM_POW_REWARD` rows (S-001) and regenerate the two dependent vectors with 11 operations, or keep the published 10-operation vectors and add the 11-operation ones alongside; then bring `one_tx_per_op` and `sample_ops` back to one shared list so the two generators cannot diverge again. Treat the generators as what they are, a way to *propose* new vectors, and the fixture as the thing CI runs.

**References**: Mantle spec §Test Vectors; Cryptarchia spec §Test Vectors ("generated by the implementation's `generate_body_root_test_vectors` at commit `e14c7fd1`"); #245 S-002, S-003; #245 LB-002 (issue #719).

## 5. Suggestions (non-security)

### S-001 · Spec: add the `CLAIM_POW_REWARD` rows to the Mantle and Cryptarchia §Test Vectors

The Mantle §Operation Id table covers 10 of the 11 live opcodes (Mantle rev 1.15.0 added `CLAIM_POW_REWARD`, `0x40`, without a row), the "one of each operation" transaction has 10 operations, and the Cryptarchia "one transaction per operation kind" vector has 10 leaves. The values below are what the node generator emits for its sample instance (`epoch_nonce = Fr(35)`, `block_hash = [0x24; 32]`, `public_key = Fr(37)`); each was recomputed independently in Python from the published preimage rules and is pinned by the Appendix B fixture, so it can be adopted as is.

| Field | Value |
|---|---|
| `CLAIM_POW_REWARD` payload (`EpochNonce BlockHash PublicKey`) | `0x2300000000000000000000000000000000000000000000000000000000000000` `2424242424242424242424242424242424242424242424242424242424242424` `2500000000000000000000000000000000000000000000000000000000000000` |
| `op_id` | `0x5e0997fddce4a431c2197e2480561c0de82b3db220626fe40f6186d0166241d8` |
| Single-operation transaction hash (`leaf[10]`) | `0x73b869deeeabc96a2c7e0723b9d3bd0f82e28a7e41791b9eb63bf67ab7555a87` |
| "One of each operation" transaction, 11 ops (`0x0b || …`, 1312 bytes) | hash `0xe551ee978b8b45a46b8a28386d01790cd530a74bdf13b9b756a307fb9413099f` |
| `merkle_root(transactions)` over 11 leaves | `0xfa038c91e8130b08311d59d5172c5549e835fa329d03b2ffa19c86b1ab7e550c` |
| `body_root`, empty `uncle_headers`, 11 leaves | `0x437c2694960194a134ff70029e6932dc73cdf4781870dc07a4c82453715d4a3d` |
| `body_root`, the two published uncles, 11 leaves | `0x1308a1f1718cfd2811a6c3d780de43365d54647ee0dd22bf18368b072290f707` |
| `block_id`, the published `Header` vector with the 11-leaf `body_root` | `0x99c619f0f59c24a7dc012b6196fd0dd6b53c338f149078b94c85ca61da5e16de` |

Whether the 10-operation vectors are replaced or kept alongside the 11-operation ones is a spec choice; the fixture pins the published 10-operation values and the proposed `op_id` row, and would pin the rest once published.

### S-002 · Spec and node: publish a `SignedMantleTx` vector, so the proof-variant derivation is pinned too

Every published vector stops at the unsigned `MantleTx`. The `OpsProofs` layer of the encoding spec, where the proof variant of each `OpProof` is derived from the operation, is pinned by no spec vector, and the only committed byte vector for a signed transaction in the node (`transactions/codec.rs:146-188`, a `CHANNEL_INSCRIBE` with an Ed25519 signature) was generated from the crate itself. That is the layer where the node currently deviates: `CLAIM_POW_REWARD` is bound to `NoOpProof` (`pow.rs:276`, zero bytes) while both specs require a `ZkSigProof` (#167 LB-001). A published `SignedMantleTx` vector with one operation of every kind, using the deterministic Ed25519 keys the existing vectors already use and a fixed placeholder for the 128-byte Groth16 proofs, would have flagged that deviation as a byte-length mismatch, and would fix the proof order and variants for other implementations. Recommend adding it to the Mantle §Test Vectors and to the fixture.

### S-003 · Node: the uncle signature is over `Header::to_bytes()` (bincode) while the spec signs the canonical header; the equality is unguarded

`Header::sign` (`header/mod.rs:227-228`) and `verify_header_signature` (`block/mod.rs:370-371`) both sign and verify `header.to_bytes()`, the bincode form, whereas the spec's uncle vectors carry signatures over the 297-byte canonical header. They are the same bytes today (the fixture asserts `header.to_bytes() == header.encode_to_vec()` and both uncle signatures verify), because bincode's fixed-width little-endian encoding of these particular field types happens to coincide with the canonical encoding. Nothing in the tree states or tests that coincidence outside the new fixture; a bincode configuration change (varint integers, a length prefix on a newly added field) would silently change what is signed while the canonical encoding stayed put. Recommend signing and verifying `encode_to_vec()` directly, or at least keeping a test asserting the two encodings are equal for `Header`.

### S-004 · Spec: the Cryptarchia §Test Vectors state the generating commit; the Mantle ones do not

`cryptarchia-v1-protocol.md` §Test Vectors names the node commit (`e14c7fd1`) and generator its values came from. The Mantle §Test Vectors name neither, which makes S-001 style drift harder to notice. Recommend stating the commit for the Mantle table as well, and, once the fixture is committed, pointing both sections at it rather than at the `#[ignore]`d generators.

---

## Appendix A — Independent reproduction

Environment: specs at `6637c791`, node at `85a16208`, Python 3 with `hashlib` and `cryptography`. Every published value was recomputed from the published inputs without the node:

- `blake2b256(b"OPERATION_ID_V1" || payload)` equals the published `op_id` for all 10 rows; `blake2b256(b"MANTLE_TXHASH_V1" || payload)` equals both published transaction hashes; the "one of each" payload equals `0x0a || (opcode || payload)*` over the rows in table order.
- `declaration_id`: the published preimage (`0x00 || provider_id || zk_id || Locators`) hashes to the published `7fb647c0…`; the same preimage with `"BN"` (`0x424e`) in place of `0x00` hashes to `019e6a82…`, which is what `DeclarationMessage::id` (`sdp/mod.rs:508-524`) returns and what the fixture reports.
- Cryptarchia: `leaf[i] = blake2b256(b"MANTLE_TXHASH_V1" || 0x01 || opcode_i || payload_i)` matches all 10 published leaves; the BLAKE2b binary Merkle tree over the leaves padded to 16 with zero leaves gives the published root; `blake2b256(b"BODY_ROOT_V1" || 0x00 || root)` and `blake2b256(b"BODY_ROOT_V1" || 0x02 || uncle0 || uncle1 || root)` give both published `body_root`s; `blake2b256(b"BLOCK_ID_V1" || 0x01 || parent || slot_le || body_root || leader_voucher || entropy || proof || leader_key)` gives the published `block_id`.
- Uncles: each published entry is 361 bytes; bytes `233..265` of each header equal the Ed25519 public key of the seed key `[0x66; 32]` / `[0x77; 32]`, and the trailing 64 bytes are that key's signature over the leading 297 bytes. The `Header` vector's `leader_key` equals the public key of seed `[0x33; 32]`.
- Generators at `85a16208` (`--ignored --nocapture`): `generate_op_id_test_vectors` prints the 10 published rows plus the `CLAIM_POW_REWARD` row of S-001; `generate_mantle_tx_hash_test_vectors` prints the published empty-transaction vector and the 11-operation transaction of LB-001; `generate_body_root_test_vectors` prints exactly the published Cryptarchia vectors (10 leaves).

## Appendix B — The conformance fixture

Two `cfg(test)` modules added to `logos-blockchain-core` on the scratch checkout at `85a16208`, registered with the two-line `mod` additions below. No production code changes. Results at `85a16208`: 7 of 8 pass, `mantle_spec_declaration_id_vector` fails with `019e6a82…` vs `7fb647c0…`; with `hasher.update(self.service_type.encode())` in place of the `"BN"` match in `DeclarationMessage::id`, 8 of 8 pass and all 307 non-ignored library tests of the crate, the 8 new ones included, are green. `cargo clippy -p logos-blockchain-core --tests --all-features` with `CARGO_BUILD_WARNINGS=deny`: clean. `rustfmt --check` (nightly-2026-07-05, edition 2024): clean.

Registration:

```diff
--- a/core/src/mantle/ops/mod.rs
+++ b/core/src/mantle/ops/mod.rs
@@ -11,6 +11,8 @@ pub mod proof_zk_and_ed;
 pub mod sdp;
 mod serde_;
 pub mod signed_op;
+#[cfg(test)]
+pub(crate) mod spec_vectors;
 pub mod signed_op_error;
--- a/core/src/header/mod.rs
+++ b/core/src/header/mod.rs
@@ -11,6 +11,8 @@ use lb_key_management_system_keys::keys::{Ed25519Key, Ed25519Signature};
 mod fixtures;
+#[cfg(test)]
+mod spec_vectors;
```

### B.1 `core/src/mantle/ops/spec_vectors.rs`

```rust
//! Conformance fixtures pinning the Mantle specification §Test Vectors
//! (`bedrock-v1.1-mantle-specification.md`, rev 1.15.0, logos-lips
//! `6637c791`). Unlike the `#[ignore]`d generators in `mantle_test_vectors`,
//! these tests embed the *published* bytes and fail if the crate drifts away
//! from them: every payload is decoded with the crate codec, re-encoded
//! byte-exact, and its identifier recomputed and compared to the spec value.

use lb_binary_codec::canonical::{BinaryDecode as _, BinaryEncode as _};

use crate::{
    crypto::{Digest as _, Hasher},
    mantle::{
        ops::{OPERATION_ID_V1, Op, OpId as _},
        traits::Hashable as _,
        transactions::tx_list::Ops,
    },
};

/// One row of the spec's §Test Vectors / Operation Id table.
pub struct OpVector {
    pub name: &'static str,
    pub opcode: u8,
    /// Canonical operation encoding without the leading opcode byte.
    pub payload: &'static str,
    pub op_id: &'static str,
}

/// Mantle spec §Test Vectors / Operation Id, in table order.
pub const OP_ID_VECTORS: &[OpVector] = &[
    OpVector {
        name: "TRANSFER",
        opcode: 0x00,
        payload: "0201000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000020300000000000000040000000000000000000000000000000000000000000000000000000000000005000000000000000600000000000000000000000000000000000000000000000000000000000000",
        op_id: "5e5e1b318aa0c2aec93fbb327e6af5f705e5684269a34e0c1319539d00d06cdb",
    },
    OpVector {
        name: "CHANNEL_CONFIG",
        opcode: 0x10,
        payload: "0707070707070707070707070707070707070707070707070707070707070707000000000000000000000000000000000000000000000000000000000000000002001398f62c6d1a457c51ba6a4b5f3dbd2f69fca93216218dc8997e416bd17d93cafd1724385aa0c75b64fb78cd602fa1d991fdebf76b13c58ed702eac835e9f6180a0000000b0000000c000d00",
        op_id: "8bac7efe4c3ef10745c0d509ac88e2abaf1d3cda94987ed2eeb9ad71dd31d056",
    },
    OpVector {
        name: "CHANNEL_INSCRIBE",
        opcode: 0x11,
        payload: "0e0e0e0e0e0e0e0e0e0e0e0e0e0e0e0e0e0e0e0e0e0e0e0e0e0e0e0e0e0e0e0e0b00000068656c6c6f206c6f676f730000000000000000000000000000000000000000000000000000000000000000d9bf2148748a85c89da5aad8ee0b0fc2d105fd39d41a4c796536354f0ae2900c",
        op_id: "fb9af7fb1384fff51780ec8c5afbcba76449ab7603484f797df3a472e48826c1",
    },
    OpVector {
        name: "CHANNEL_DEPOSIT",
        opcode: 0x12,
        payload: "1010101010101010101010101010101010101010101010101010101010101010011100000000000000000000000000000000000000000000000000000000000000100000006465706f7369742d6d65746164617461",
        op_id: "f14ff0aad9bc5e8e30c5d1aa3710aaa1c1cc1f47c2c256e7d9e73104cb17ccaf",
    },
    OpVector {
        name: "CHANNEL_WITHDRAW",
        opcode: 0x13,
        payload: "1212121212121212121212121212121212121212121212121212121212121212011300000000000000000000000000000000000000000000000000000000000000",
        op_id: "503d0d08f9faef971864943103965d13be7159fe6e0361c8ea614c6d0431e59c",
    },
    OpVector {
        name: "CHANNEL_TRANSFER",
        opcode: 0x14,
        payload: "14141414141414141414141414141414141414141414141414141414141414140115000000000000000000000000000000000000000000000000000000000000000116000000000000001700000000000000000000000000000000000000000000000000000000000000",
        op_id: "fb24c17731954e8bbe1b0dedd69e4857c8083d1689aff331ba16f3ed5883f0ce",
    },
    OpVector {
        name: "SDP_DECLARE",
        opcode: 0x20,
        payload: "00010b00047f00000191020bb8cd0353470962558a6e0839022ae65c6b2723b32772e5c0c5f4776cb8e6a3e10ba2f319000000000000000000000000000000000000000000000000000000000000001a00000000000000000000000000000000000000000000000000000000000000",
        op_id: "42e93fdce121a5ab4da3201a6fd2da1d42ca8b7d8c1a8c9e2a657a6cdc7aa468",
    },
    OpVector {
        name: "SDP_WITHDRAW",
        opcode: 0x21,
        payload: "1b1b1b1b1b1b1b1b1b1b1b1b1b1b1b1b1b1b1b1b1b1b1b1b1b1b1b1b1b1b1b1b1d000000000000001c00000000000000000000000000000000000000000000000000000000000000",
        op_id: "c95aea0e46f60c12a8b29b259ca1b39947093c0d88a1ea8400c49e392ca491a0",
    },
    OpVector {
        name: "SDP_ACTIVE",
        opcode: 0x22,
        payload: "1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1e1f0000000000000001010a0000008a88e3dd7409f195fd52db2d3cba5d72ca6709bf1d94121bf3748801b40f6f5c020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020202020303030303030303030303030303030303030303030303030303030303030303",
        op_id: "76afa55f5733db75a982dc5ccabb5c6a7dab992eda78cdfd5f657f314e388354",
    },
    OpVector {
        name: "LEADER_CLAIM",
        opcode: 0x30,
        payload: "200000000000000000000000000000000000000000000000000000000000000021000000000000000000000000000000000000000000000000000000000000002200000000000000000000000000000000000000000000000000000000000000",
        op_id: "0dc1a007fdd184b4553a83d166b749a621f5be2de4b3b0429ebf0520d1dd9a51",
    },
];

/// `CLAIM_POW_REWARD` (opcode `0x40`) has no row in the spec table yet. This
/// is the row the crate's own generator (`generate_op_id_test_vectors`) emits
/// for its sample instance: `epoch_nonce = Fr(35)`, `block_hash = [0x24; 32]`,
/// `public_key = Fr(37)`. Pinned here so the value cannot move silently
/// before the spec adopts it.
pub const CLAIM_POW_REWARD_PROPOSED: OpVector = OpVector {
    name: "CLAIM_POW_REWARD",
    opcode: 0x40,
    payload: "230000000000000000000000000000000000000000000000000000000000000024242424242424242424242424242424242424242424242424242424242424242500000000000000000000000000000000000000000000000000000000000000",
    op_id: "5e0997fddce4a431c2197e2480561c0de82b3db220626fe40f6186d0166241d8",
};

/// Mantle spec §Test Vectors / Mantle Transaction Hash.
const TX_HASH_EMPTY: (&str, &str) = (
    "00",
    "2eba3f667b80a508f3d44d149a1c27a90ea365a51e4fc8209289088142b364e5",
);
const TX_HASH_ONE_OF_EACH: &str =
    "11e6013847824badf33aa383cfbdb4b5b74a621acefc8296c21f48c4072e0e92";

/// Mantle spec §Test Vectors / Declaration Id (reuses the `SDP_DECLARE` row).
const DECLARATION_ID: &str = "7fb647c069bade94e06685b0825299d220e7cc14752cfc474773b6c4040e37b5";

pub fn unhex(s: &str) -> Vec<u8> {
    hex::decode(s).expect("spec vectors are valid hex")
}

/// `opcode || payload`: the full canonical `Op` encoding of a row.
pub fn wire_bytes(v: &OpVector) -> Vec<u8> {
    let mut bytes = vec![v.opcode];
    bytes.extend(unhex(v.payload));
    bytes
}

fn op_id_from_payload(payload: &[u8]) -> [u8; 32] {
    let mut preimage = OPERATION_ID_V1.clone();
    preimage.extend_from_slice(payload);
    Hasher::digest(&preimage).into()
}

/// Decode → re-encode (byte-exact) → recompute `op_id` for one row.
fn check_op_vector(v: &OpVector) -> Op {
    let wire = wire_bytes(v);
    let op =
        Op::decode_all(&wire, &()).unwrap_or_else(|e| panic!("{}: decode failed: {e:?}", v.name));
    assert_eq!(
        hex::encode(op.encode()),
        hex::encode(&wire),
        "{}: re-encoding is not byte-exact",
        v.name
    );
    assert_eq!(
        hex::encode(op_id_from_payload(&op.encode()[1..])),
        v.op_id,
        "{}: op_id differs from the spec",
        v.name
    );
    // Where the production trait is implemented it must agree with the spec too.
    let trait_op_id = match &op {
        Op::Transfer(o) => Some(o.op_id()),
        Op::ChannelDeposit(o) => Some(o.op_id()),
        Op::ChannelTransfer(o) => Some(o.op_id()),
        Op::ChannelWithdraw(o) => Some(o.op_id()),
        Op::LeaderClaim(o) => Some(o.op_id()),
        Op::ClaimPowReward(o) => Some(o.op_id()),
        Op::ChannelInscribe(_)
        | Op::ChannelConfig(_)
        | Op::SDPDeclare(_)
        | Op::SDPWithdraw(_)
        | Op::SDPActive(_) => None,
    };
    if let Some(id) = trait_op_id {
        assert_eq!(hex::encode(id), v.op_id, "{}: OpId::op_id differs", v.name);
    }
    op
}

#[test]
fn mantle_spec_op_id_vectors() {
    assert_eq!(OP_ID_VECTORS.len(), 10, "the spec table has 10 rows");
    for v in OP_ID_VECTORS {
        check_op_vector(v);
    }
}

#[test]
fn claim_pow_reward_proposed_op_id_vector() {
    let op = check_op_vector(&CLAIM_POW_REWARD_PROPOSED);
    assert!(matches!(op, Op::ClaimPowReward(_)));
}

#[test]
fn mantle_spec_tx_hash_vectors() {
    // Empty transaction.
    let (payload, hash) = TX_HASH_EMPTY;
    let bytes = unhex(payload);
    let tx = Ops::decode_all(&bytes, &()).expect("empty tx decodes");
    assert_eq!(hex::encode(tx.encode()), payload);
    assert_eq!(hex::encode(tx.hash().0), hash);

    // One of each operation: the spec payload is the opcode-ordered
    // concatenation `count || (opcode || payload)*` of the op_id rows.
    let mut bytes = vec![u8::try_from(OP_ID_VECTORS.len()).expect("fits")];
    for v in OP_ID_VECTORS {
        bytes.extend(wire_bytes(v));
    }
    let tx = Ops::decode_all(&bytes, &()).expect("one-of-each tx decodes");
    assert_eq!(hex::encode(tx.encode()), hex::encode(&bytes));
    assert_eq!(tx.len(), OP_ID_VECTORS.len());
    assert_eq!(hex::encode(tx.hash().0), TX_HASH_ONE_OF_EACH);
}

#[test]
fn mantle_spec_declaration_id_vector() {
    let row = OP_ID_VECTORS
        .iter()
        .find(|v| v.name == "SDP_DECLARE")
        .expect("row present");
    let Op::SDPDeclare(declaration) = check_op_vector(row) else {
        panic!("SDP_DECLARE row must decode as a declaration");
    };
    assert_eq!(
        hex::encode(declaration.id().0),
        DECLARATION_ID,
        "declaration_id differs from the spec (see #155 LB-001 / #245 LB-001)"
    );
}
```

### B.2 `core/src/header/spec_vectors.rs`

```rust
//! Conformance fixtures pinning the Cryptarchia specification §Test Vectors
//! (`cryptarchia-v1-protocol.md`, logos-lips `6637c791`): the transaction
//! Merkle root, `body_root` with and without uncles, and the `HeaderId`
//! (`block_id`). The transaction leaves reuse the Mantle `op_id` rows pinned in
//! `crate::mantle::ops::spec_vectors`, one single-operation transaction each.

use lb_poseidon2::Fr;

use super::*;
use crate::{
    block::{SignedHeader, UncleHeaders, body_root, verify_header_signature},
    mantle::{
        ops::{
            leader_claim::VoucherCm,
            spec_vectors::{OP_ID_VECTORS, unhex, wire_bytes},
        },
        traits::Hashable as _,
        transactions::tx_list::Ops,
    },
    utils::merkle,
};

const EMPTY_TX_ROOT: &str = "0000000000000000000000000000000000000000000000000000000000000000";
const TX_LEAVES: [&str; 10] = [
    "6ab0046084f3ce8dad90eb28afe5692ad92d5d0588a4e868ad38d0d841d7a60e",
    "15c4361f33089c446b8f2f7747ec211d5d031da529a3ee6b62e9df670f996dbf",
    "50e5674eea7fa17f531a51159ea7c3cab843fb1c8e8bf9bd5518a8aad08865d3",
    "d52da59d9db42391363d6c4f96447536e5dfff747b91b88320310b07581a8dee",
    "6f57c77dc872cc3f01380fbd57a97e9f7998a1cd8b24e84594ceba796cfa0822",
    "2c04be946507e2b8c239b85b03cf476a8be5af8e4de853660d0447a46ea460fc",
    "9ce9fa694b4c801eca6c9a1d3dca6401952404bda8c144fb16e03e3872fd475e",
    "3555b3d8f5d05ea5d69efb17aab7639474738bcb4bfee8d354107433d781ef9c",
    "0a91ab8271016f212061e6b45ea35c95cfa0f9a70c5225508f284b2657f4d931",
    "c992f1a63a7ea665a3766fae6b032df3db12ef386caf0ef1f3654afedbc51c6c",
];
const TX_ROOT: &str = "65f481d9f0cdb38f2166299c40f4e74bec7332df72281daec7a6547a098ff08b";
const BODY_ROOT_NO_UNCLES: &str =
    "d279012d8ce1c7db4812b900c29174f2657b3fa243270fc8ebeeb5f1c0a29cec";
const BODY_ROOT_TWO_UNCLES: &str =
    "7f3854d9c24cfdb5e30f37a74ace03d06adacbf807d55a89e4935d90f286e831";
/// The two carried uncles, each a 297-byte header followed by its 64-byte
/// signature; built by `uncle(0x66)` and `uncle(0x77)`.
const UNCLE_HEADERS: [&str; 2] = [
    "016666666666666666666666666666666666666666666666666666666666666666660000000000000066666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666666660000000000000000000000000000000000000000000000000000000000000034b4d9043156cb6dcf0beb0a2949b7559c940d2bcb6dbe8c53a9b30278e3a7466600000000000000000000000000000000000000000000000000000000000000563913f1ba7ad4129a077acd56278e743fd45120226dd315fa49f3a9c5d07af6a174ab84d4555a279afe053e79c8bb794be3f7d2e71e92b8da1b490687cb8306",
    "0177777777777777777777777777777777777777777777777777777777777777777700000000000000777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777777700000000000000000000000000000000000000000000000000000000000000c853ad0f0cd2b619aea92ceec4fd56a24d6499d584ce79257e45cfd8139b60a77700000000000000000000000000000000000000000000000000000000000000ad17e45d503a16fb41c25c4b3025956c63b31015871e957f3562b47cebce784e5b392ce3dd05214afe09102e0d2ed8211a83b81f18231963a226198fd528df0c",
];
/// `Header` vector: wire-order fields and the resulting `block_id`.
const HEADER_PARENT: &str = "1111111111111111111111111111111111111111111111111111111111111111";
const HEADER_SLOT: u64 = 42;
const HEADER_LEADER_VOUCHER: &str =
    "4444000000000000000000000000000000000000000000000000000000000000";
const HEADER_ENTROPY: &str = "5555000000000000000000000000000000000000000000000000000000000000";
const HEADER_PROOF: &str = "2222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222";
const HEADER_LEADER_KEY: &str = "17cb79fb2b4120f2b1ec65e4198d6e08b28e813feb01e4a400839b85e18080ce";
const BLOCK_ID: &str = "b5232b5462d6d802b2e77185e3fd7124af713e36818db1438d921e8c980232bb";

fn array32(s: &str) -> [u8; 32] {
    unhex(s).try_into().expect("32 bytes")
}

/// One single-operation transaction per Mantle `op_id` row, in table order.
fn one_tx_per_op() -> Vec<Ops> {
    OP_ID_VECTORS
        .iter()
        .map(|v| {
            let mut bytes = vec![1u8];
            bytes.extend(wire_bytes(v));
            let tx = Ops::decode_all(&bytes, &()).unwrap_or_else(|e| panic!("{}: {e:?}", v.name));
            assert_eq!(hex::encode(tx.encode()), hex::encode(&bytes), "{}", v.name);
            tx
        })
        .collect()
}

/// The generator's uncle: every field set from one byte, signed by the key
/// derived from that byte.
fn uncle(byte: u8) -> (Header, Ed25519Signature) {
    let signing_key = Ed25519Key::from_bytes(&[byte; 32]);
    let header = Header::new(
        HeaderId([byte; 32]),
        ContentId([byte; 32]),
        Slot::from(u64::from(byte)),
        Groth16LeaderProof::from_parts(
            lb_pol::PoLProof::from_bytes(&[byte; 128]),
            Fr::from(u64::from(byte)),
            signing_key.public_key(),
            VoucherCm::from(Fr::from(u64::from(byte))),
        ),
    );
    let signature = header.sign(&signing_key).expect("header serializes");
    (header, signature)
}

#[test]
fn cryptarchia_spec_transaction_leaves_and_root() {
    let empty: Vec<Ops> = vec![];
    assert_eq!(
        hex::encode(merkle::calculate_transactions_root(&empty)),
        EMPTY_TX_ROOT
    );

    let txs = one_tx_per_op();
    assert_eq!(txs.len(), TX_LEAVES.len());
    for (i, (tx, leaf)) in txs.iter().zip(TX_LEAVES).enumerate() {
        assert_eq!(hex::encode(tx.hash().0), leaf, "leaf[{i}]");
    }
    assert_eq!(
        hex::encode(merkle::calculate_transactions_root(&txs)),
        TX_ROOT
    );
}

#[test]
fn cryptarchia_spec_body_root_without_uncles() {
    let root = body_root(&UncleHeaders::empty(), &one_tx_per_op());
    assert_eq!(hex::encode(root.0), BODY_ROOT_NO_UNCLES);
    // The empty list encodes as the single byte 0x00.
    assert_eq!(UncleHeaders::empty().encode_to_vec(), vec![0u8]);
}

#[test]
fn cryptarchia_spec_body_root_with_two_uncles() {
    let mut signed = Vec::new();
    for (byte, expected) in [(0x66u8, UNCLE_HEADERS[0]), (0x77u8, UNCLE_HEADERS[1])] {
        let (header, signature) = uncle(byte);
        // The signature covers the canonical 297-byte header, which must be
        // the same bytes `Header::sign` signs (its bincode form).
        assert_eq!(header.encode_to_vec().len(), Header::CANONICAL_ENCODED_SIZE);
        assert_eq!(
            header.to_bytes().expect("bincode").as_ref(),
            header.encode_to_vec().as_slice()
        );
        verify_header_signature(&header, &signature).expect("uncle signature verifies");

        let uncle = SignedHeader::new(header, signature);
        assert_eq!(
            hex::encode(uncle.encode_to_vec()),
            expected,
            "uncle {byte:#x}"
        );
        assert_eq!(
            SignedHeader::decode_all(&unhex(expected), &()).expect("uncle decodes"),
            uncle
        );
        signed.push(uncle);
    }
    let uncles = UncleHeaders::new([signed[0].clone(), signed[1].clone()]);
    let mut expected_list = vec![2u8];
    expected_list.extend(unhex(UNCLE_HEADERS[0]));
    expected_list.extend(unhex(UNCLE_HEADERS[1]));
    assert_eq!(uncles.encode_to_vec(), expected_list);
    assert_eq!(
        hex::encode(body_root(&uncles, &one_tx_per_op()).0),
        BODY_ROOT_TWO_UNCLES
    );
}

#[test]
fn cryptarchia_spec_header_id() {
    let proof_bytes: [u8; 128] = unhex(HEADER_PROOF).try_into().expect("128 bytes");
    let header = Header::new(
        HeaderId(array32(HEADER_PARENT)),
        ContentId(array32(BODY_ROOT_NO_UNCLES)),
        Slot::from(HEADER_SLOT),
        Groth16LeaderProof::from_parts(
            lb_pol::PoLProof::from_bytes(&proof_bytes),
            Fr::from(0x5555u64),
            Ed25519Key::from_bytes(&[0x33u8; 32]).public_key(),
            VoucherCm::from(Fr::from(0x4444u64)),
        ),
    );
    // Wire order: version, parent, slot, body_root, proof, entropy, leader_key,
    // voucher.
    let mut wire = vec![BEDROCK_VERSION];
    wire.extend(unhex(HEADER_PARENT));
    wire.extend(HEADER_SLOT.to_le_bytes());
    wire.extend(unhex(BODY_ROOT_NO_UNCLES));
    wire.extend(unhex(HEADER_PROOF));
    wire.extend(unhex(HEADER_ENTROPY));
    wire.extend(unhex(HEADER_LEADER_KEY));
    wire.extend(unhex(HEADER_LEADER_VOUCHER));
    assert_eq!(wire.len(), Header::CANONICAL_ENCODED_SIZE);
    assert_eq!(hex::encode(header.encode_to_vec()), hex::encode(&wire));
    assert_eq!(
        Header::decode_all(&wire, &()).expect("header decodes"),
        header
    );
    assert_eq!(hex::encode(header.id().0), BLOCK_ID);
}
```

---

## Appendix C — Definitions

Severity, difficulty and category use the definitions in `docs/REPORT_TEMPLATE.md` Appendix A.
