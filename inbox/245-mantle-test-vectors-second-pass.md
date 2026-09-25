# Audit Report · Mantle §Test Vectors, second pass: op_id, tx hash and declaration_id against the current specs, the SDP_ACTIVE Metadata production, and the vectors of the encoding and cryptographic specs

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/245`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `c4c86be18c58b5b09c3650e93871c8cfb624885b` · component(s): `core/src/mantle/ops`, `core/src/mantle/fixtures/ops`, `core/src/mantle/transactions/tx_list`, `core/src/sdp`, `zk/poseidon2`
Specs: `https://github.com/logos-co/logos-lips` @ `d788723992a805b395f377de6e6cf59859b47168` · read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `mantle-transaction-encoding.md`, `key-types-and-generation.md` (all four in full); `bedrock-v1.1-mantle-specification.md` §Revisions History, §Mantle Transaction, §Mantle Transaction Hash, §Test Vectors (Operation Id, Mantle Transaction Hash, Declaration Id); `bedrock-service-declaration-protocol.md` §Revision History, §Service Types, §Identifiers, §Locators, §Declaration Message, §Declaration Storage, §Identifier Uniqueness, §Active Message; `blend-protocol.md` §Active Message (Rewarding); `common-cryptographic-components.md` from §Revision History to the end (BLAKE2b, ChaCha20-Based PRNG Construction, Poseidon2, EdDSA, ZkSignature, Groth16, Annex: Poseidon2 Test Values)
Date: `2026-09-25` · author: `Claude Code (research agent)` · status: `final`

---

## 1. Summary

- Overall assessment: nothing in the vectors moved upstream (the only spec change since the first pass's `7244d3b0`/`75d3d038` is the strict EdDSA rule in `common-cryptographic-components.md`), and at `c4c86be1` all 10 `op_id` vectors and both transaction-hash vectors still reproduce through the crate codec, while the `declaration_id` vector still fails for the same reason as before (#538 / #718: `DeclarationMessage::id` hashes ASCII `"BN"`, now at `core/src/sdp/mod.rs:514-531`). Neither side resolved the `SDP_ACTIVE` `Metadata = UINT32 *BYTE` disagreement. The node has since landed a mandatory golden-fixture framework, and it now pins the unprefixed `SDP_ACTIVE` form in committed tests. So the code side is fixed in place while the encoding spec still says otherwise; this report files that as a spec deviation (LB-001). The same framework pins crate-invented sample values rather than the published rows, and pins no identifier at all. So 31 commits after the first pass, no committed test asserts any Mantle §Test Vectors value, and the `declaration_id` deviation is still invisible to CI (LB-002). The encoding spec and the key-types spec publish no vectors. The common-crypto spec publishes 12 Poseidon2 values, all asserted by `zk/poseidon2` and passing.
- Findings: 0 critical · 0 high · 0 medium · 0 low · 2 informational
- Key themes: conformance-test coverage; a spec production that the vector, the Blend spec and the code all contradict; a defect whose fix is one line and still verified green.
- Must-fix before launch: the `declaration_id` fix already on file (#538 / #718). It must land before any `declaration_id` is persisted. I found none persisted at this commit.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `core/src/mantle/ops/mod.rs`, `ops/op.rs` | `OpId` preimage (`op.rs:29-37`), the `#[ignore]`d generators (`mod.rs:116-148`) |
| `core/src/mantle/transactions/tx_list/ops.rs`, `tx_list/hash.rs` | `Ops` codec and `Hashable` (`ops.rs:180-201`), `tx_hasher` |
| `core/src/mantle/fixtures/ops/{op_values.rs,op.rs,sdp.rs}`, `fixtures/transactions/tx_list/ops.rs` | the committed golden codec fixtures added after `9ffddb30b` |
| `binary-codec/src/canonical/fixtures.rs` | the `CodecExamples` / `codec_fixtures!` contract that makes a fixture mandatory for every codec |
| `core/src/sdp/mod.rs`, `core/src/sdp/blend.rs` | `DeclarationMessage::id` (`mod.rs:514-531`), `ActiveMessage` (`mod.rs:570-576`), `ActivityMetadata` codec (`mod.rs:601-638`), `ActivityProof` codec (`blend.rs:35-84`) |
| `zk/poseidon2/src/hasher.rs` | `test_hashes` / `test_compression` (L102-L180) against the Poseidon2 annex |
| `nodes/node/standalone-node-config.yaml`, `tools/config/src/sdp.rs` | whether any `declaration_id` is persisted |

**Out of scope**

The signed-transaction layer (`OpsProofs`, proof-variant derivation) is issue #743, assigned to someone else, and is not touched here. The Cryptarchia §Test Vectors were re-verified by #641 at `85a16208`. They were not re-run here: no commit since then touches `core/src/header` or `core/src/block`, and they are not a section this issue names. Operation validation and execution semantics are also out of scope. Third-party crates assumed correct: `blake2`, `jf-poseidon2`, `ark-*`, `ed25519-dalek`, `multiaddr`, `hex`.

**Assumptions**

The specs at `d7887239` are the reference. `Hash` is BLAKE2b-256, which the passing vectors confirm.

## 3. Method

- Worked through issue #245 under parent #10, the first report `processed/245-mantle-test-vector-conformance.md` (PR #640, at `9ffddb30b`) and its filed findings #718 and #719, the follow-up report `inbox/641-spec-test-vector-conformance-fixtures.md` (issue #641, closed), and the open issue #743, so as not to redo their work. I did not re-report #538 / #718 or #719 as new findings: their status is re-verified in §3.2.
- Spec reading as listed in the header, before any code. `git diff 7244d3b0..d7887239 -- docs/blockchain/raw/` (and from `75d3d038`, the commit the first report's header records) touches only `common-cryptographic-components.md`: revision 1.2.0 adds strict EdDSA verification (L196, L211). No vector, production or preimage rule in any of the specs this issue names changed.
- Independent recomputation with Python 3.11 `hashlib.blake2b(digest_size=32)`, parsed straight from the spec markdown: all 10 `op_id`, both tx hashes, the one-of-each payload equal to `0x0a || (opcode || payload)*` over the rows, the `declaration_id` from the published preimage (`7fb647c0…`) and from the `"BN"` preimage (`019e6a82…`), and a check that the Preimage row equals `service || provider_id || zk_id || locators` sliced from the `SDP_DECLARE` payload. The 12 Poseidon2 annex values were compared as integers with the decimal constants in `zk/poseidon2/src/hasher.rs` (12/12 equal).
- Dynamic testing on a scratch clone (`git clone --shared`) at `c4c86be1`, `rustc 1.98.1` per `rust-toolchain.toml`, release profile. I added the scratch module of Appendix B (`core/src/mantle/ops/spec_vectors.rs`, 6 tests) and ran `cargo test --release -p logos-blockchain-core --features test-utils --lib`: 479 passed, 1 failed (`mantle_spec_declaration_id_vector`), 3 ignored. With the one-line #538 fix applied: 480 passed, 0 failed. `-p logos-blockchain-ledger --lib` with the fix: 171 passed. `-p logos-blockchain-poseidon2`: 2 passed. I also ran the two Mantle generators with `--ignored --nocapture`. Without `--features test-utils` the core lib test target does not compile at this commit (S-002). The audited trees were not modified.

### 3.1 Conformance matrix at `c4c86be1`, specs `d7887239`

Decode with the crate codec, re-encode byte-exact, recompute the identifier, compare with the published value.

| Vector | Changed upstream since `7244d3b0`? | Result at `c4c86be1` |
|---|---|---|
| `op_id` `TRANSFER`, `CHANNEL_CONFIG`, `CHANNEL_INSCRIBE`, `CHANNEL_DEPOSIT`, `CHANNEL_WITHDRAW`, `CHANNEL_TRANSFER`, `SDP_DECLARE`, `SDP_WITHDRAW`, `SDP_ACTIVE`, `LEADER_CLAIM` | no | pass, 10/10; `OpId::op_id()` also agrees for the 5 rows whose type implements it |
| Tx hash, empty (`0x00`) | no | pass (`2eba3f66…`) |
| Tx hash, one of each operation (1215 bytes) | no | pass (`11e60138…`) |
| `declaration_id` Preimage row, rebuilt from the decoded op with the crate's own `ServiceType`, `ProviderId`, `fr_to_bytes`, `Locators` encoders | no | pass: equals the published preimage and hashes to `7fb647c0…` |
| `declaration_id` via `DeclarationMessage::id()` | no | **fail**: `019e6a82…` vs `7fb647c0…` (#538 / #718); passes with the one-line fix |
| `SDP_ACTIVE` metadata layout (Blend §Active Message, 230 bytes, no length prefix) | no | the vector decodes, the metadata re-encodes to bytes 40..270 of the payload; the `UINT32`-prefixed form the encoding spec describes is rejected (`UnknownDiscriminant { discriminant: 230 }`) (LB-001) |
| Proposed `CLAIM_POW_REWARD` row (#245 S-002 / #641 S-001) | still absent from the spec | node value unchanged, `5e0997fd…`, passes |
| Ed25519 keys inside the vectors (2 `CHANNEL_CONFIG` signers, the inscription signer, the `SDP_DECLARE` `provider_id`, the `SDP_ACTIVE` `signing_key`) | spec 1.2.0 added small-order rejection | all decode under the weak-key rejection added by node commit `874b7877c` (#3646) |
| Poseidon2 annex, 8 hash + 4 compression | no | asserted by `test_hashes` / `test_compression`, pass |

Generators (`--ignored --nocapture`): `generate_op_id_test_vectors` prints the 10 published rows plus the `CLAIM_POW_REWARD` row, `generate_mantle_tx_hash_test_vectors` prints the published empty vector and an 11-operation transaction (`e551ee97…`), not the published 10-operation one. This is unchanged from #641 LB-001, except that the label now reads `"(11 ops)"` (`mod.rs:147`) instead of `"(9 ops)"`. The `OpId` cross-check at `mod.rs:121-130` still falls through `_ => {}` for `ClaimPowReward`.

### 3.2 Status of earlier items

| Item | At `9ffddb30b` / `85a16208` | At `c4c86be1` |
|---|---|---|
| #538 / #718 `declaration_id` hashes `"BN"` | open, `sdp/mod.rs:490-507` | **still open**, the same code at `sdp/mod.rs:514-531` (the file was touched by `ecbb5d461`, `d2a134634`, `f1bfda40b`, `f1b9f0f41`, `874b7877c`, none of which changes the preimage) |
| `declaration_id` persisted anywhere | none | none: `standalone-node-config.yaml:157` is `declaration_id: null`, `tools/config/src/sdp.rs:20` derives `decl.id()` live, and the pinned `logos-blockchain-testing` rev is still `db880d61`. The fix still needs no migration |
| #719 `op_id` preimage omits the opcode | open | unchanged (`op.rs:29-37`; `op_bytes()` is `encode_to_vec()` of the payload, e.g. `transfer.rs:85-87`, `channel_transfer.rs:63-65`), spec unchanged |
| #245 S-001 `SDP_ACTIVE` `Metadata = UINT32 *BYTE` | Suggestion, not filed | unresolved on both sides, and the code side is now pinned by committed fixtures: LB-001 |
| #245 S-003 / #641 committed conformance test | not committed | not committed: no published value (`op_id`, tx hash, `declaration_id`, or any payload of the rows) appears anywhere in the tree (`git grep`); see LB-002 |
| #641 LB-001 generator drift | open (inbox) | unchanged apart from the label; not re-reported |

### 3.3 Vectors in the encoding, key-types and cryptographic specs (the last unchecked item)

- `mantle-transaction-encoding.md` (rev 1.8.0): no test vectors. Its productions are exercised only through the Mantle §Test Vectors. Each row decodes as the production says, apart from `SDPActive`/`Metadata` (LB-001).
- `key-types-and-generation.md` (rev 1.1.0): no test vectors. It defines key roles only (NQK = `zk_id`, NSK = `provider_id`).
- `common-cryptographic-components.md` (rev 1.2.0): the only vectors are the 12 Poseidon2 annex values (L300-L322). All 12 are asserted as ordinary, non-ignored tests in `zk/poseidon2/src/hasher.rs:102-180` and pass. No separate scratch assertion was needed; the decimal constants were matched to the annex hex one by one. There are no vectors for BLAKE2b with a DST, for the ChaCha20 PRNG (L96-L125), or for the new strict EdDSA rule (L196). The SDP operations depend on BLAKE2b through `op_id`, the tx hash and `declaration_id`, and those are covered by the Mantle vectors. The other two gaps are recorded in S-003.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Spec deviation: `SDP_ACTIVE` is encoded with no `UINT32` length before `Metadata`, as the vector and the Blend spec say, while the encoding spec's `Metadata = UINT32 *BYTE` requires one; the encoding spec is the side that is wrong | Determinism | Informational | High | Open |
| LB-002 | The new mandatory golden fixtures pin crate-invented operation values instead of the published rows and assert no identifier, so the Mantle vectors are still unasserted and the `declaration_id` deviation is invisible to CI | Determinism | Informational | High | Open |

### LB-001 · Spec deviation: `SDP_ACTIVE` is encoded with no `UINT32` length before `Metadata`, as the vector and the Blend spec say, while the encoding spec's `Metadata = UINT32 *BYTE` requires one; the encoding spec is the side that is wrong

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Determinism |
| Target | `core/src/sdp/mod.rs:570-576` (`ActiveMessage`), `core/src/sdp/mod.rs:601-638` (`ActivityMetadata` codec), `core/src/sdp/blend.rs:35-84` (`ActivityProof` codec); fixtures `core/src/mantle/fixtures/ops/sdp.rs:45-89`, `core/src/mantle/fixtures/ops/op_values.rs:96-105,135-136`; spec `mantle-transaction-encoding.md:129-130` |
| Status | Open at `c4c86be1` (spec side open at `d7887239`) |

**Description**

The encoding spec (rev 1.8.0, unchanged since the first pass) says:

```
SDPActive     = DeclarationId Nonce Metadata
Metadata      = UINT32 *BYTE  ; Service-specific node activeness metadata
```

The node encodes `ActiveMessage` as a plain concatenation with no length. The comment on the derive states the layout as `ActiveMessage = DeclarationId Nonce Metadata`, a plain field-order concatenation (`sdp/mod.rs:570`). `ActivityMetadata` writes the one-byte `metadata_type` (`ACTIVE_METADATA_BLEND_TYPE = 1`, L601) followed by `ActivityProof`: `version(1) · epoch(4, LE) · signing_key(32) · proof_of_quota(160) · proof_of_selection(32)` (`blend.rs:48-54`). That is 230 bytes, exactly the layout of `blend-protocol.md` §Active Message (L1091-L1102), which says the envelope encoding is defined elsewhere and restates no length. The Mantle `SDP_ACTIVE` vector also has no prefix: bytes 40..46 are `01 01 0a000000`. Read as the spec's `UINT32`, they would give a length of 655617.

Two things changed since the first pass flagged this as #245 S-001. First, neither side moved. Second, the node's new mandatory golden fixtures now pin the unprefixed form in committed tests: `ActiveMessage` (`fixtures/ops/sdp.rs:45-61`), `ActivityMetadata` (L76-L89), `ActivityProof` (L62-L75), and `Op::SDPActive` / `Ops` through `SDP_ACTIVE_HEX` / `ALL_OPS_COLUMN_HEX` (`op_values.rs:135-136,145`). Changing the node to match the encoding spec would now fail its own CI, which is the right outcome: three of the four sources agree. Because Suggestions are not filed, the first pass's S-001 never reached the tracker, and the spec text is still wrong. This report therefore files it as a finding, as the README asks for code/spec disagreements.

The scratch test `sdp_active_metadata_has_no_uint32_prefix` (Appendix B) shows the concrete incompatibility. It inserts `UINT32(230)` after the nonce, as a reader of the encoding spec would, and the node rejects the message with `UnknownDiscriminant { type_name: "…::ActivityMetadata", discriminant: 230 }`: the first length byte `0xe6` is read as `metadata_type`.

The ABNF has a second, related defect. `Metadata` is also defined at L99 for `ChannelDeposit`, where it genuinely is `UINT32 *BYTE`: the deposit vector carries `10000000` = 16 before 16 bytes, and `DepositOp.metadata` is a length-prefixed `UpperBoundedVec` (`ops/channel/deposit.rs:31-37`). One rule name is used for two shapes. Under ABNF a second `=` definition of the same rule is not allowed, so the SDP definition cannot differ from the channel one without being renamed.

**Exploit scenario**

Not exploitable. The impact is interoperability. An alternative implementation built from `mantle-transaction-encoding.md` alone emits `SDP_ACTIVE` operations that every node rejects at decode, so its Blend nodes never record activity and lose their rewards. It also fails to decode every `SDP_ACTIVE` in a block, and therefore the whole block. The published vector is the only tie-breaker, and a reader has to notice that it contradicts the production.

**Recommendation**

- *Short term* (spec): change the `SDPActive` production to `SDPActive = DeclarationId Nonce ActiveMetadata` with `ActiveMetadata = MetadataType *BYTE ; service-defined layout, see the service spec (Blend: 229 bytes after the type byte)`, and rename the channel rule to `DepositMetadata = UINT32 *BYTE`. No node change is needed.
- *Long term*: once the rule is fixed, add the published `SDP_ACTIVE` row as a fixture of `Op` (see LB-002) so that the vector, the production and the code are checked together.

**References**: `mantle-transaction-encoding.md` §SDP Operations (L129-L130), §Channel Operations (L96-L99); `blend-protocol.md` §Active Message (L1087-L1106); `bedrock-service-declaration-protocol.md` §Active Message ("The `metadata` layout is service-defined; the SDP does not parse it"); first pass #245 S-001; #155 S-004.

### LB-002 · The new mandatory golden fixtures pin crate-invented operation values instead of the published rows and assert no identifier, so the Mantle vectors are still unasserted and the `declaration_id` deviation is invisible to CI

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Determinism |
| Target | `core/src/mantle/fixtures/ops/op_values.rs:33-145`, `core/src/mantle/fixtures/ops/op.rs:14-27`, `core/src/mantle/fixtures/ops/sdp.rs:25-35`, `core/src/mantle/fixtures/transactions/tx_list/ops.rs`; `core/src/sdp/mod.rs:514-531` (`DeclarationMessage::id`) |
| Status | Open at `c4c86be1` |

**Description**

Since `9ffddb30b` the node has made a golden fixture mandatory for every codec. `CodecExamples` is a sealed supertrait of `BinaryEncode`/`BinaryDecode`, satisfiable only through `#[derive(BinaryCodec)]` or `codec_fixtures!` (`binary-codec/src/canonical/fixtures.rs:7-17`), and the generated test checks each fixture's value against its exact bytes. This is the mechanism #245 S-003 and #641 asked for, but the Mantle fixtures were written with a new set of values that no spec row uses:

- `op_values.rs:33-117` builds each operation from bytes `0x10…0xa2` (for example `SDP_DECLARE` with `provider_id` from seed `[0x60; 32]` and `zk_id = 0x61`), and `op_values.rs:119-145` pins their encodings.
- The published rows are built by the crate's own `sample()` constructors (seeds `7, 8, 9`, `14, 15`, `24, 25, 26`, …; for example `sdp/mod.rs:549-560`, `ops/channel/config.rs:59-72`), which the `#[ignore]`d generators print. The committed fixtures do not use them.
- No fixture or test asserts an identifier. `git grep` finds none of the 10 `op_id`s, the two tx hashes, `7fb647c0…`, or any row payload in the tree. The only `op_id` assertions compare the crate with itself (`ops/mod.rs:121-130`, `leader_claim.rs:650`). The only `declaration_id` tests are relational (`sdp/mod.rs:753-769` asserts two ids differ).

So the published vectors are still pinned by nothing, and the one of them that fails (`declaration_id`, #538 / #718) fails in no committed test. A change to any preimage (the opcode fold #719 recommends, a change to the `Locators` production, a new `ServiceType` discriminant) or to any payload shape that also appears in the sample values would move the golden bytes and the identifiers together. The reviewer would see only the fixture diff, never a disagreement with the spec.

The gap is now cheap to close. `codec_fixtures!` takes several `value => bytes` pairs, so each published row can be added next to the existing fixture as `Self::Transfer(TransferOp::sample()) => "<spec payload>"`, and so on for the other rows. The identifier side needs the separate test of Appendix B, which compiles and runs against this commit unchanged apart from being registered in `ops/mod.rs`. It is 5/6 green today, and 6/6 green with the #538 fix, with the whole core library (480 tests) and ledger library (171 tests) green.

**Exploit scenario**

Not a security issue. The impact is on conformance. The node's own documentation invites alternative implementations to check themselves against its generators (`ops/mod.rs:34-38`), and a silent preimage change in the node would carry them along or break them without any signal. The `declaration_id` fix, which everyone agrees on, keeps not landing partly because no test fails for it.

**Recommendation**

- *Short term*: add the 10 published Operation Id rows as extra `codec_fixtures!` entries of `Op` (built from the existing `sample()` constructors), the published one-of-each transaction as an `Ops` fixture, and the identifier test of Appendix B (`op_id`, tx hash, `declaration_id` preimage and id). Land it together with the #538 one-liner, or mark the `declaration_id` test `#[should_panic(expected = "declaration_id differs from the spec")]` with a comment naming #538 until the fix lands.
- *Long term*: make the `sample()` constructors and `op_values.rs` one source, so that the spec vectors, the generators and the golden fixtures cannot drift apart. Have the spec's §Test Vectors name the node commit and the fixture module they are pinned by (#641 S-004).

**References**: Mantle spec §Test Vectors; #245 S-003; #641 (closed) and its report's Appendix B; #538 / #718; #719; node commits `ecbb5d461`, `e9c8a5280`, `f1b9f0f41`.

## 5. Suggestions (non-security)

### S-001 · Encoding spec: `Locator = 2Byte *BYTE` does not state the byte order of the length, and several rules are defined twice

`mantle-transaction-encoding.md:120` writes the locator length as `2Byte`, not `UINT16`, so the ABNF does not say it is little-endian. Only the Mantle §Test Vectors / Declaration Id prose says so ("prefixed with its 2-byte little-endian byte length"), and the SDP spec §Declaration Storage says only "its byte length". The vector (`0b00` for an 11-byte locator) and the code agree on little-endian. Recommend `Locator = UINT16 *BYTE`. `Inputs` is also defined three times (L97, L110, L154), and `InputCount` (L98, L155), `Outputs` (L108, L156) and `OutputCount` (L109, L157) twice each. They are identical today, but a single definition in §Common Structures avoids the `Metadata` trap of LB-001.

### S-002 · `cargo test -p logos-blockchain-core --lib` does not compile without `--features test-utils`

`TransferOp::sample()` (`core/src/mantle/ops/transfer.rs:72-81`) and `ChannelTransferOp::sample()` (`core/src/mantle/ops/channel/channel_transfer.rs:50-60`) are gated `#[cfg(any(test, feature = "test-utils"))]`, but the imports they use (`Fr`, `ZkPublicKey`, `Note`, `NoteId`) are gated `#[cfg(feature = "test-utils")]` only (`transfer.rs:2-5`, `channel_transfer.rs:2-9`). A plain `cargo test -p logos-blockchain-core --lib` therefore fails with 11 E0425/E0433 errors, which is what a developer (or the #641 fixture instructions) would run. CI does not see it because nextest runs with `--all-features` (`.github/workflows/code-check.yml:171`). The imports were introduced by `13ab0f400` (#3557). Recommend gating the imports with the same `any(test, feature = "test-utils")` predicate, and adding one CI job that builds the test targets with default features.

### S-003 · Common cryptographic components: the annex prints two values without their leading zero, and the new strict EdDSA rule and the ChaCha20 PRNG have no test values

Hash `[0,1,0,1]` and compression `[1,0]` are printed with 63 hex digits (L313, L320), and the annex does not say that its values are big-endian integers, while the same document's byte convention for field elements is little-endian (L158). A reader who loads them as 32-byte strings gets a parse failure or the wrong element. Recommend printing all 64 digits and stating the convention. Revision 1.2.0 makes rejection of small-order `A` and `R` normative (L196). A few vectors would pin it for every implementation: a small-order public key, a small-order `R`, and a non-canonical `S`, each with the expected reject. The ChaCha20 PRNG (L96-L125) claims byte equality with `rand_chacha::ChaCha20Rng`, and would similarly benefit from one seed/output pair.

---

## Appendix A · Definitions

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

## Appendix B · Scratch conformance test and fix

Registered in the scratch clone with a two-line addition to `core/src/mantle/ops/mod.rs` after `pub mod signed_op;`:

```rust
#[cfg(test)]
mod spec_vectors;
```

Results at `c4c86be1` (`cargo test --release -p logos-blockchain-core --features test-utils --lib spec_vectors`):

```
test mantle::ops::spec_vectors::claim_pow_reward_proposed_op_id_vector ... ok
test mantle::ops::spec_vectors::mantle_spec_declaration_id_preimage_from_crate_encoders ... ok
test mantle::ops::spec_vectors::mantle_spec_declaration_id_vector ... FAILED
  left: "019e6a82692d5533fc162b137c11fd2ad1e2c4f2bc1e728237e00572e7331526"
 right: "7fb647c069bade94e06685b0825299d220e7cc14752cfc474773b6c4040e37b5"
test mantle::ops::spec_vectors::mantle_spec_op_id_vectors ... ok      (10/10)
test mantle::ops::spec_vectors::mantle_spec_tx_hash_vectors ... ok
test mantle::ops::spec_vectors::sdp_active_metadata_has_no_uint32_prefix ... ok
  UINT32-prefixed SDP_ACTIVE rejected: UnknownDiscriminant { type_name: "logos_blockchain_core::sdp::ActivityMetadata", discriminant: 230 }
```

Whole library: 479 passed, 1 failed, 3 ignored. With the fix below: 480 passed, 0 failed; `logos-blockchain-ledger --lib` 171 passed.

The fix (#538 / #718), as applied to the scratch clone:

```diff
--- a/core/src/sdp/mod.rs
+++ b/core/src/sdp/mod.rs
@@ -513,14 +513,11 @@ impl DeclarationMessage {
     pub fn id(&self) -> DeclarationId {
         let mut hasher = Blake2b::new();
-        let service = match self.service_type {
-            ServiceType::BlendNetwork => "BN",
-        };
 
         // declaration_id = Hash(service||provider_id||zk_id||locators)
-        hasher.update(service.as_bytes());
+        hasher.update(self.service_type.encode());
         hasher.update(self.provider_id.as_ref());
```

`core/src/mantle/ops/spec_vectors.rs`. The ten `OpVector` rows are the Mantle §Operation Id table verbatim (payload and `op_id` hex without `0x`), identical to the `OP_ID_VECTORS` constant printed in full in the #641 report, Appendix B.1. They are abbreviated here to their names to keep the appendix readable; the proposed `CLAIM_POW_REWARD` row is `payload 2300…00 2424…24 2500…00`, `op_id 5e0997fd…d8`, as in #641 S-001.

```rust
use lb_binary_codec::canonical::{BinaryDecode as _, BinaryEncode as _};
use lb_groth16::fr_to_bytes;

use crate::{
    crypto::{Digest as _, Hasher},
    mantle::{
        ops::{OPERATION_ID_V1, Op, OpId as _},
        traits::Hashable as _,
        transactions::tx_list::Ops,
    },
    sdp::{ActiveMessage, ActivityMetadata},
};

struct OpVector { name: &'static str, opcode: u8, payload: &'static str, op_id: &'static str }

const OP_ID_VECTORS: &[OpVector] = &[ /* TRANSFER 0x00, CHANNEL_CONFIG 0x10, CHANNEL_INSCRIBE 0x11,
    CHANNEL_DEPOSIT 0x12, CHANNEL_WITHDRAW 0x13, CHANNEL_TRANSFER 0x14, SDP_DECLARE 0x20,
    SDP_WITHDRAW 0x21, SDP_ACTIVE 0x22, LEADER_CLAIM 0x30: spec rows verbatim */ ];
const CLAIM_POW_REWARD_PROPOSED: OpVector = /* #641 S-001 row */;
const TX_HASH_EMPTY: &str = "2eba3f667b80a508f3d44d149a1c27a90ea365a51e4fc8209289088142b364e5";
const TX_HASH_ONE_OF_EACH: &str = "11e6013847824badf33aa383cfbdb4b5b74a621acefc8296c21f48c4072e0e92";
const DECLARATION_ID_PREIMAGE: &str = "0053470962558a6e0839022ae65c6b2723b32772e5c0c5f4776cb8e6a3e10ba2f31900000000000000000000000000000000000000000000000000000000000000010b00047f00000191020bb8cd03";
const DECLARATION_ID: &str = "7fb647c069bade94e06685b0825299d220e7cc14752cfc474773b6c4040e37b5";

fn unhex(s: &str) -> Vec<u8> { hex::decode(s).expect("hex") }

fn wire_bytes(v: &OpVector) -> Vec<u8> {
    let mut bytes = vec![v.opcode];
    bytes.extend(unhex(v.payload));
    bytes
}

fn op_id_from_payload(payload: &[u8]) -> [u8; 32] {
    let mut preimage = OPERATION_ID_V1.clone();
    preimage.extend_from_slice(payload);
    Hasher::digest(&preimage).into()
}

fn check_op_vector(v: &OpVector) -> Op {
    let wire = wire_bytes(v);
    let op = Op::decode_all(&wire, &()).unwrap_or_else(|e| panic!("{}: decode failed: {e:?}", v.name));
    assert_eq!(hex::encode(op.encode()), hex::encode(&wire), "{}: re-encoding", v.name);
    assert_eq!(hex::encode(op_id_from_payload(&op.encode()[1..])), v.op_id, "{}: op_id", v.name);
    let trait_op_id = match &op {
        Op::Transfer(o) => Some(o.op_id()),
        Op::ChannelDeposit(o) => Some(o.op_id()),
        Op::ChannelTransfer(o) => Some(o.op_id()),
        Op::ChannelWithdraw(o) => Some(o.op_id()),
        Op::LeaderClaim(o) => Some(o.op_id()),
        Op::ClaimPowReward(o) => Some(o.op_id()),
        Op::ChannelInscribe(_) | Op::ChannelConfig(_) | Op::SDPDeclare(_)
        | Op::SDPWithdraw(_) | Op::SDPActive(_) => None,
    };
    if let Some(id) = trait_op_id {
        assert_eq!(hex::encode(id), v.op_id, "{}: OpId::op_id differs", v.name);
    }
    op
}

#[test]
fn mantle_spec_op_id_vectors() {
    assert_eq!(OP_ID_VECTORS.len(), 10);
    for v in OP_ID_VECTORS { check_op_vector(v); }
}

#[test]
fn claim_pow_reward_proposed_op_id_vector() {
    assert!(matches!(check_op_vector(&CLAIM_POW_REWARD_PROPOSED), Op::ClaimPowReward(_)));
}

#[test]
fn mantle_spec_tx_hash_vectors() {
    let tx = Ops::decode_all(&unhex("00"), &()).expect("empty tx decodes");
    assert_eq!(hex::encode(tx.encode()), "00");
    assert_eq!(hex::encode(tx.hash().0), TX_HASH_EMPTY);

    let mut bytes = vec![u8::try_from(OP_ID_VECTORS.len()).unwrap()];
    for v in OP_ID_VECTORS { bytes.extend(wire_bytes(v)); }
    assert_eq!(bytes.len(), 1215);
    let tx = Ops::decode_all(&bytes, &()).expect("one-of-each tx decodes");
    assert_eq!(hex::encode(tx.encode()), hex::encode(&bytes));
    assert_eq!(hex::encode(tx.hash().0), TX_HASH_ONE_OF_EACH);
}

#[test]
fn mantle_spec_declaration_id_preimage_from_crate_encoders() {
    let row = OP_ID_VECTORS.iter().find(|v| v.name == "SDP_DECLARE").unwrap();
    let Op::SDPDeclare(d) = check_op_vector(row) else { panic!() };
    let mut preimage = d.service_type.encode_to_vec();
    preimage.extend_from_slice(d.provider_id.as_ref());
    preimage.extend_from_slice(&fr_to_bytes(d.zk_id.as_fr()));
    preimage.extend(d.locators.encode_to_vec());
    assert_eq!(hex::encode(&preimage), DECLARATION_ID_PREIMAGE);
    let id: [u8; 32] = Hasher::digest(&preimage).into();
    assert_eq!(hex::encode(id), DECLARATION_ID);
}

#[test]
fn mantle_spec_declaration_id_vector() {
    let row = OP_ID_VECTORS.iter().find(|v| v.name == "SDP_DECLARE").unwrap();
    let Op::SDPDeclare(d) = check_op_vector(row) else { panic!() };
    assert_eq!(hex::encode(d.id().0), DECLARATION_ID, "declaration_id differs from the spec");
}

#[test]
fn sdp_active_metadata_has_no_uint32_prefix() {
    let row = OP_ID_VECTORS.iter().find(|v| v.name == "SDP_ACTIVE").unwrap();
    let payload = unhex(row.payload);
    assert_eq!(payload.len(), 32 + 8 + 230);
    let msg = ActiveMessage::decode_all(&payload, &()).expect("vector decodes");
    let ActivityMetadata::Blend(proof) = &msg.metadata;
    assert_eq!(u32::from(proof.epoch), 10);
    assert_eq!(msg.metadata.encode_to_vec(), payload[40..].to_vec());

    // The encoding-spec reading: UINT32(230) after the nonce.
    let mut prefixed = payload[..40].to_vec();
    prefixed.extend(230u32.to_le_bytes());
    prefixed.extend_from_slice(&payload[40..]);
    let err = ActiveMessage::decode_all(&prefixed, &()).expect_err("prefixed form must not decode");
    println!("UINT32-prefixed SDP_ACTIVE rejected: {err:?}");
}
```
