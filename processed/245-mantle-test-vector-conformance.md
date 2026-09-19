# Audit Report — Mantle §Test Vectors conformance: op_id, tx hash, declaration_id, and the SDP_ACTIVE Metadata production

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/245`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `9ffddb30b9e6cf79465802953caedd010ff1cecd` — component(s): `core/src/mantle/ops`, `core/src/sdp`, `core/src/mantle/ledger.rs`, `ledger/src/mantle/sdp`
Specs: `https://github.com/logos-co/logos-lips` @ `75d3d0382604d4a0d8e246c268935dd386ffc8ea` — read: `bedrock-v1.1-mantle-specification.md` (§Test Vectors, §Mantle Transaction Hash, §Operations, §Mantle Ledger in full), `mantle-transaction-encoding.md` (in full), `bedrock-service-declaration-protocol.md` (§Service Types, §Locators, §Declaration Storage, §Active Message), `blend-protocol.md` (§Active Message), `bedrock-service-reward-distribution.md` (§Service Reward Distribution), `common-cryptographic-components.md` (§Poseidon2 annex), `cryptarchia-v1-protocol.md` (§Test Vectors); core specs `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`
Date: 2026-09-19 — author: `claude-fable-5-1` — status: `final`

---

## 1. Summary

- Overall assessment: the ten `op_id` vectors and both Mantle transaction-hash vectors reproduce exactly at this commit; the `declaration_id` vector still fails (confirming #155 LB-001 is open here). The published vectors are asserted by no committed test — the node ships only `#[ignore]`d generators that regenerate the values rather than pin them against the spec.
- Findings: 0 critical · 0 high · 0 medium · 1 low · 1 informational
- Key themes: determinism / conformance-test coverage; a hash preimage (`op_id`) that omits the opcode.
- Must-fix before launch: LB-001 (the `declaration_id` fix from #155 LB-001, a one-liner) must land before any `declaration_id` is persisted anywhere that survives a reset. It is not persisted anywhere today (see LB-001), so there is no migration.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `core/src/mantle/ops/op.rs`, `ops/mod.rs` | `Op` decode/encode; `OpId` trait and `op_id` preimage; the `#[ignore]`d generators `generate_op_id_test_vectors` / `generate_mantle_tx_hash_test_vectors` |
| `core/src/mantle/ops/**` | per-op payload codecs (`transfer`, `channel/*`, `sdp/*`, `leader_claim`, `pow`) |
| `core/src/sdp/mod.rs` | `DeclarationMessage::id`, `ServiceType`/`Locator`/`ActivityMetadata` codecs |
| `core/src/sdp/blend.rs` | `ActivityProof` codec (the SDP_ACTIVE metadata body) |
| `core/src/mantle/ledger.rs` | `Utxo::id` (`NoteId` derivation) |
| `core/src/header/mod.rs` | `#[ignore]`d `generate_body_root_test_vectors` (Cryptarchia §Test Vectors) |
| `zk/poseidon2/src/hasher.rs` | Poseidon2 annex test values |
| `ledger/src/mantle/sdp/rewards/mod.rs` | reward `op_id` preimage (`hash(ServiceType || epoch_number)`) |

Verification was done against the vectors published in `bedrock-v1.1-mantle-specification.md` §Test Vectors and `cryptarchia-v1-protocol.md` §Test Vectors, and the Poseidon2 annex of `common-cryptographic-components.md`.

**Out of scope**

Runtime validation/execution semantics of the operations (covered by #155 and others); the ZK circuits and proving keys; wallet, mempool, and networking. Third-party crates assumed correct: `blake2`/`hasher`, `jf-poseidon2` (jellyfish rev `8d80230358…`), `ark-*`, `rust-rapidsnark`, `multiaddr`, `bincode`, `serde`, `hex`.

**Assumptions**

The specs at the pinned `logos-lips` commit are the reference. `Hasher` = BLAKE2b-256 (confirmed by the passing `op_id`/`tx_hash` vectors). BN254 is the field for `Fr`.

## 3. Method

- Manual review of the in-scope paths, working through issue #245 (spun out of #155) and the parent #10.
- Spec conformance against the documents listed above. Every field of the SDP_ACTIVE/Blend metadata layout, the `declaration_id` preimage, and the `Metadata` production was read in the encoding spec, the Mantle spec, the SDP spec and the Blend spec and cross-checked against the code.
- Automated tooling: `cargo 1.90`-toolchain per `rust-toolchain.toml`; independent recomputation of every vector with Python 3 `hashlib.blake2b(digest_size=32)`.
- Dynamic testing: **on a copy of the node tree** (`core` only), I added a committed-style fixture test `core/src/mantle/ops/spec_vectors.rs` that, for every published vector, decodes the payload bytes with the crate codec, re-encodes and compares bytes, and compares the computed `op_id` / `tx_hash` / `declaration_id` against the spec value. I ran it at `9ffddb30b` unmodified, then with the #155 LB-001 one-line fix applied, and ran the full `logos-blockchain-core` (295 tests) and `logos-blockchain-ledger` (171 tests) libraries. I also ran the node's own `#[ignore]`d generators (`--ignored --nocapture`) and `logos-blockchain-poseidon2`. No change was made to the audited tree; the node repositories were left untouched.

### 3.1 Conformance matrix at `9ffddb30b`

Decode → re-encode (byte-exact) → recompute id:

| Vector | Decodes | Byte round-trip | Computed id == spec |
|---|---|---|---|
| `op_id` × 10 (`TRANSFER … LEADER_CLAIM`) | ✅ | ✅ | ✅ (all 10) |
| Mantle tx hash — empty tx | ✅ | ✅ | ✅ |
| Mantle tx hash — one of each op | ✅ | ✅ | ✅ |
| `declaration_id` | ✅ | ✅ | ❌ → `019e6a82…` vs spec `7fb647c0…` (**LB-001**) |
| Poseidon2 annex hash+compression (12 rows) | — | — | ✅ asserted by `zk/poseidon2` `test_hashes` / `test_compression` |
| Cryptarchia §Test Vectors (tx root, body_root, HeaderId) | — | — | reproduced by the node's `#[ignore]`d `generate_body_root_test_vectors`; no committed spec-comparison test |

With the #155 LB-001 fix applied to the copy, the `declaration_id` vector passes and all 295 core + 171 ledger tests stay green.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | `declaration_id` conformance vector fails: `SDP_DECLARE` id hashes ASCII `"BN"`, not the `ServiceType` discriminant | Determinism | Low | High | Open |
| LB-002 | `op_id` preimage omits the opcode: one byte string yields the same `op_id` and `NoteId` under two operations | Data Validation | Informational | High | Open |

### LB-001 · `declaration_id` conformance vector fails: `SDP_DECLARE` id hashes ASCII `"BN"`, not the `ServiceType` discriminant

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Determinism |
| Target | `core/src/sdp/mod.rs:490-507` (`DeclarationMessage::id`) |
| Status | Open at `9ffddb30b` |

**Description**

The Mantle §Test Vectors / Declaration Id vector specifies `declaration_id = 0x7fb647c0…` for a preimage whose first field is the one-byte `ServiceType` discriminant `0x00` ([Declaration Storage](https://github.com/logos-co/logos-lips/blob/master/docs/blockchain/raw/bedrock-service-declaration-protocol.md#declaration-storage), [Service Types](https://github.com/logos-co/logos-lips/blob/master/docs/blockchain/raw/bedrock-service-declaration-protocol.md#service-types): "This byte is the canonical encoding of a `ServiceType` and is used wherever a `ServiceType` is serialized or hashed … the `declaration_id` preimage").

`DeclarationMessage::id` instead hashes the ASCII service name:

```rust
let service = match self.service_type {
    ServiceType::BlendNetwork => "BN",
};
hasher.update(service.as_bytes());   // 0x42 0x4e, not the 0x00 discriminant
```

This is finding **#155 LB-001**, filed against `a805329f`. This report confirms it is still present at `9ffddb30b` and pins it with a decoding conformance test: the crate decodes the published `SDP_DECLARE` payload, computes `019e6a82692d5533fc162b137c11fd2ad1e2c4f2bc1e728237e00572e7331526`, while the spec value is `7fb647c069bade94e06685b0825299d220e7cc14752cfc474773b6c4040e37b5`. Python `blake2b(digest_size=32)` reproduces the spec value from the `0x00`-prefixed preimage and the code's value from the `"BN"`-prefixed preimage, so the only difference is the two service bytes.

The other 30 Mantle vectors (10 `op_id`, 2 tx-hash) all pass, so the `Locators` encoding, the field order, and the hash function are all correct; only the service prefix differs.

**Exploit scenario**

Not directly exploitable. The `declaration_id` is deterministic and every node computes it the same wrong way, so consensus does not split. The impact is a spec deviation that (a) makes any external or future re-implementation that follows the spec disagree with the node on every `declaration_id`, and (b) would harden into a migration hazard the moment a `declaration_id` is persisted across a format change. I checked for such persistence at this commit and found none: `nodes/node/standalone-node-config.yaml` ships `sdp.declaration_id: null`; the pinned `logos-blockchain-testing` checkout (rev `db880d61…`) contains no `declaration_id`/`DeclarationId` literal; and `tools/config/src/sdp.rs:20` derives it live from `decl.id()` rather than storing a constant. So the fix can land with no migration today, which is exactly why it should land now.

**Recommendation**

- *Short term*: replace the `"BN"` match with the canonical encoding, as proposed in #155 — `hasher.update(self.service_type.encode());`. Verified on a copy of the tree: this makes the `declaration_id` vector pass and leaves all 295 `core` and 171 `ledger` library tests green.
- *Long term*: adopt the conformance test in S-003 so this vector (and the others) are pinned in CI and a future edit cannot silently drift.

**References**: Mantle spec §Test Vectors / Declaration Id (rev 1.10.1); SDP spec §Service Types, §Declaration Storage; prior finding #155 LB-001.

### LB-002 · `op_id` preimage omits the opcode: one byte string yields the same `op_id` and `NoteId` under two operations

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Data Validation |
| Target | `core/src/mantle/ops/op.rs:28-37` (`OpId::op_id`), `core/src/mantle/ops/mod.rs:199-207`; `core/src/mantle/ledger.rs:511-530` (`Utxo::id`) |
| Status | Open at `9ffddb30b` |

**Description**

`op_id = blake2b256("OPERATION_ID_V1" || op_bytes)`, where `op_bytes` is the operation encoding **without** the 1-byte opcode tag (`op.encode()[1..]`; every `OpId` impl returns `self.encode_to_vec()`, which is the payload only). The spec preimage is the same: §Test Vectors states the payload column is "the canonical operation encoding without the leading opcode byte". So the opcode is a domain separator on the wire but is absent from both the `op_id` preimage and, transitively, from `NoteId = zkhash(NOTE_ID_V1, op_id, output_index, value, pk)`.

Because the payload productions of different operations are structurally similar (a `TRANSFER` is `Inputs Outputs`; a `CHANNEL_TRANSFER` is `ChannelId Inputs Outputs`), a single byte string can be a valid payload under two opcodes and then hash identically. On a copy of the tree I decoded one payload as both a `TRANSFER` (1 input, 5 outputs) and a `CHANNEL_TRANSFER` (5 inputs, 1 output); `TransferOp::op_id() == ChannelTransferOp::op_id()`, and their output-0 `NoteId`s were equal (`NoteId(6878257237144814680251414356296802668429947275269324876046030367770534­59083)`).

**Exploit scenario**

Not reachable as a live attack at this commit, hence Informational. Realising a `NoteId` collision on-chain requires two operations that both **validate and execute** to insert a note under the shared id: a `CHANNEL_TRANSFER` needs the channel to exist and a threshold of accredited-key signatures, a `TRANSFER` needs a valid `ZkSignature` over the inputs, and the ledger would have to insert the second colliding key. This finding records the latent gap rather than a working exploit, and it is the concrete instance of the class flagged in the #51 report ("an op type whose id does not cover all inputs") — there, inserting a duplicate ledger key is noted as a silent consensus-split trigger for any future path that re-inserts a key. Folding the opcode into the `op_id` preimage removes the cross-opcode branch of that class entirely.

**Recommendation**

- *Short term*: none required for safety today; treat as defence-in-depth.
- *Long term*: include the opcode byte in the `op_id` preimage (`"OPERATION_ID_V1" || opcode || op_bytes`) in both the spec and the node, so `op_id` is domain-separated by operation kind. This is a hard format change (it moves every `op_id`, `NoteId`, and tx hash) and must be coordinated with the spec; it is cheap now and expensive after genesis.

**References**: Mantle spec §Operations / Opcodes, §Note Id; §Test Vectors / Operation Id; prior finding #51 LB (duplicate-key latent split).

## 5. Suggestions (non-security)

### S-001 · Spec deviation: encoding spec `Metadata = UINT32 *BYTE` for `SDP_ACTIVE` disagrees with the vector and the code; the `Metadata` name is overloaded

`mantle-transaction-encoding.md` §SDP Operations defines `SDPActive = DeclarationId Nonce Metadata` with `Metadata = UINT32 *BYTE`. That production is wrong for `SDP_ACTIVE`, and the code and the vector agree it is wrong:

- The `SDP_ACTIVE` vector payload has no `UINT32` length prefix after the nonce: byte 40 onward is `01 01 0a000000 …` = `metadata_type(0x01) · version(0x01) · epoch_number(0x0a000000)`. Reading those first four bytes as a `UINT32` length would give `655617`.
- `blend-protocol.md` §Active Message pins the Blend `metadata` to a fixed 230-byte layout (`metadata_type(1) · version(1) · epoch_number(4) · signing_key(32) · proof_of_quota(160) · proof_of_selection(32)`), with **no** length prefix.
- `core/src/sdp/mod.rs:564-581` (`ActivityMetadata::encode_into`) and `core/src/sdp/blend.rs:37-54` (`ActivityProof`) encode exactly that: a 1-byte `metadata_type` discriminant followed by the version byte and the fixed fields, no `UINT32`.

Two of the three agree; the encoding spec is the outlier. Recommend the encoding spec drop the `UINT32` prefix for `SDP_ACTIVE` and describe `Metadata` as a service-tagged, service-defined body (`metadata_type` byte then a service-specific layout), matching [SDP §Active Message](https://github.com/logos-co/logos-lips/blob/master/docs/blockchain/raw/bedrock-service-declaration-protocol.md#active-message) ("The `metadata` layout is service-defined; the SDP does not parse it"). This is the spec side of #155 S-004.

Separately, the same `Metadata` production name is reused for `CHANNEL_DEPOSIT`, where it genuinely *is* `UINT32 *BYTE` (the deposit vector carries a 4-byte length `0x10000000 = 16` before 16 metadata bytes, and `DepositOp` uses a length-prefixed `Metadata` type). Giving one name to two different shapes is a readability trap; consider distinct names (e.g. `ChannelMetadata` vs `ActiveMetadata`).

### S-002 · `CLAIM_POW_REWARD` has no §Test Vectors row

The `CLAIM_POW_REWARD` operation (opcode `0x40`, added in Mantle spec 1.15.0 / encoding 1.8.0) ships in the node, and the node's own `#[ignore]`d generator emits an `op_id` for it (`5e0997fddce4a431…`), but the Mantle §Test Vectors / Operation Id table has no row for it. The table therefore covers 10 of the 11 live opcodes. Add a `CLAIM_POW_REWARD` row (payload + `op_id`) so the newest operation is pinned like the rest.

### S-003 · No committed conformance test pins the published vectors; add one

The node has three `#[ignore]`d generators — `generate_op_id_test_vectors`, `generate_mantle_tx_hash_test_vectors` (`core/src/mantle/ops/mod.rs`) and `generate_body_root_test_vectors` (`core/src/header/mod.rs`) — but they *regenerate* values and cross-check them only against the crate's own `hash()`/`op_id()`; none compares against the spec's published bytes, and all are skipped by default. So a change that alters an `op_id` would move the generator output in lockstep and never fail CI. The Poseidon2 annex is the counter-example done right: `zk/poseidon2` `test_hashes`/`test_compression` pin all 12 annex values as ordinary (non-ignored) tests.

Recommend a committed, non-ignored fixture test that embeds the spec's published payload/id strings as constants and, for each, decodes → re-encodes (byte-exact) → recomputes the id and asserts equality. The test I used for this audit does exactly this for the 10 `op_id`, 2 tx-hash and the `declaration_id` vectors; it is reproduced in Appendix B and is ready to drop into `core` (it fails today only on `declaration_id`, i.e. it encodes LB-001 as a regression guard that will go green once the LB-001 fix lands). The Cryptarchia §Test Vectors (tx root, `body_root`, `HeaderId`) deserve the same treatment against `generate_body_root_test_vectors`.

---

## Appendix B — Reproduction

Environment: node at `9ffddb30b`, specs at `75d3d0382`. All commands run against a copy of the node tree; the audited tree was not modified.

**B.1 No committed test asserts the published values.** At `9ffddb30b`, grepping the tree (excluding `target/`) for each published `op_id`/tx-hash/`declaration_id` value returns nothing; the only matches for the domain tags `OPERATION_ID_V1`/`MANTLE_TXHASH_V1`/`NOTE_ID_V1` are the implementations and the `#[ignore]`d generators.

**B.2 Independent hash reproduction (Python).** For every `op_id` row, `blake2b(b"OPERATION_ID_V1" + payload, digest_size=32)` equals the published `op_id` (10/10). For the tx-hash rows, `blake2b(b"MANTLE_TXHASH_V1" + payload, digest_size=32)` equals the published hash (both). The "one of each operation" tx payload equals the opcode-ordered concatenation `count || (opcode||payload)*` of the individual op vectors (count byte `0x0a` = 10). For `declaration_id`, `blake2b` of the spec's `0x00`-prefixed preimage gives `7fb647c0…` (spec); of the `"BN"`-prefixed preimage gives `019e6a82…` (code).

**B.3 Crate conformance test.** `core/src/mantle/ops/spec_vectors.rs` (added to the copy) with three tests:

- `spec_op_id_vectors`: for each of the 10 rows, `Op::decode_all(opcode||payload)` succeeds, `op.encode()` reproduces the wire bytes, and `blake2b("OPERATION_ID_V1" || encode()[1..])` equals the row's `op_id`. Result: all pass.
- `spec_tx_hash_vectors`: for both rows, `Ops::decode_all(payload)` succeeds, re-encodes byte-exact, and `Ops::hash()` equals the row's hash. Result: both pass.
- `spec_declaration_id_vector`: decodes the `SDP_DECLARE` payload and asserts `DeclarationMessage::id()` equals `7fb647c0…`. Result: **fails** at `9ffddb30b` (`019e6a82…`); **passes** after applying the #155 LB-001 fix (`hasher.update(self.service_type.encode());`), with `cargo test -p logos-blockchain-core --lib` (295 passed) and `-p logos-blockchain-ledger --lib` (171 passed) both green.

**B.4 Cross-opcode collision (LB-002).** `cross_opcode_op_id_collision` builds one payload that `TransferOp::decode_all` reads as 1 input / 5 outputs and `ChannelTransferOp::decode_all` reads as 5 inputs / 1 output; `op_id()` is equal for both, and the output-0 `NoteId` is equal for both.

**B.5 Other spec vectors.** The 12 Poseidon2 annex values in `common-cryptographic-components.md` are all asserted by `zk/poseidon2` tests (verified by decimal-matching the constants in `hasher.rs`). The Cryptarchia §Test Vectors leaves and roots are all emitted by `generate_body_root_test_vectors` (10/10 non-trivial values present in its output) but not pinned by any non-ignored test.

---

## Appendix A — Definitions

Severity, difficulty and category use the definitions in `docs/REPORT_TEMPLATE.md` Appendix A.
