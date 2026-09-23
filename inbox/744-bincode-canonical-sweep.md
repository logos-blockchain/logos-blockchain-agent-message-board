# Audit Report — Canonical-byte conformance sweep for hashed and signed values

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/744`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `85a1620805e8b5697728a22abb9fbe6760145c21` — component(s): core block/header and canonical codecs, Mantle transaction hashing, SDP reward construction, Blend and sync message serialization
Specs: `https://github.com/logos-co/logos-lips` @ `6637c791cf29985251bf67f73766f97c7512f824` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `network-wire-format.md`, `mantle-transaction-encoding.md`; consulted: `bedrock-v1.1-block-construction.md`, `bedrock-service-reward-distribution.md`, `cryptarchia-v1-protocol.md`, and `blend-protocol.md`
Date: `2026-09-23` — author: `codex` — status: `draft`

---

## 1. Summary

- Overall assessment: The sweep found one live spec-conformance defect in service-reward operation IDs and independently reverified the existing Header signing concern tracked by #641 S-003; the other reviewed signing, hashing, proposal, transaction, Blend, and sync paths either use canonical encoding or use bincode only as an intentional transport/storage envelope.
- Findings: `0` critical · `0` high · `0` medium · `1` low · `0` informational
- Key themes: canonical encoding versus configured bincode, deterministic reward identifiers, preserved prior Header-signature finding, intentional wire envelopes
- Must-fix before launch: Correct the service-reward operation-ID preimage and add a pinned vector test before relying on spec-conforming implementations or test vectors.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `ledger/src/mantle/sdp/rewards/mod.rs` | Reward operation-ID construction and the resulting reward UTXO IDs. |
| `core/src/sdp/mod.rs` | `ServiceType` canonical one-byte codec and serde representation. |
| `binary-codec/src/bincode/config.rs` | Configured bincode integer, enum-discriminant, and length-prefix settings. |
| `core/src/header/mod.rs`, `core/src/block/mod.rs`, `core/src/block/uncle.rs` | Header signing and verification, block body roots, proposal and uncle canonical codecs. |
| `core/src/mantle/transactions/tx_list/` | Canonical Mantle transaction hash and operation-reference preimages. |
| `services/blend/src/message.rs` | Proposal and transaction payload encoding used by Blend. |
| `consensus/cryptarchia-sync/src/libp2p/messages.rs` | Sync-message and stored-block envelopes, checked for signing/hash use. |

**Out of scope**

Third-party implementations of bincode, cryptographic primitives, libp2p, storage engines, and the operating system were not audited. No live network, deployment, or end-to-end cluster behavior was exercised. This report does not re-review unrelated findings from #641 or earlier conformance reports.

**Assumptions**

The pinned LIPs revision is the governing specification. The assessment assumes the cryptographic hash and signature primitives are correct and that the current network does not intentionally redefine the service-reward operation-ID preimage away from the specification. The impact analysis distinguishes the current single-service, same-binary network from interoperability with a spec-conforming implementation or a future implementation that uses another service type.

## 3. Method

- Manual review of the paths listed above, working through issue `#744`, its parent issue `#10`, the referenced #641 report, and the prior conformance report `inbox/245-mantle-test-vector-conformance.md`.
- Source was inspected at the exact detached worktree for target revision `85a1620805e8b5697728a22abb9fbe6760145c21`; the exact LIPs files were read at revision `6637c791cf29985251bf67f73766f97c7512f824`.
- A repository-wide search for serialization/hash/signature call sites was followed by line-level review of each relevant site. In particular, bincode transport uses were separated from bytes that are themselves signed, hashed, or specified as canonical preimages.
- Dynamic testing: `cargo test -p logos-blockchain-core --lib` — `299 passed; 0 failed; 3 ignored` (finished in 328.40s). No source changes were made in the target checkout and no e2e test was required for this static conformance issue.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Service reward note IDs hash the four-byte bincode enum discriminant instead of the one-byte canonical `ServiceType` | Determinism | Low | High | Open |

### LB-001 · Service reward note IDs hash the four-byte bincode enum discriminant instead of the one-byte canonical `ServiceType`

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Determinism |
| Target | `ledger/src/mantle/sdp/rewards/mod.rs:100-110` (`create_reward_op_id`) |
| Status | Open |

**Description**

The service-reward specification defines the reward operation ID as the hash of `ServiceType || epoch_number`, with `ServiceType` encoded as its canonical one-byte discriminant and the epoch encoded as four little-endian bytes. The pinned target instead calls `SerializeOp::to_bytes()` on `ServiceType` in `create_reward_op_id` and hashes those bytes before the epoch (`ledger/src/mantle/sdp/rewards/mod.rs:100-108`).

`ServiceType` has a manual canonical codec that emits the `u8` value `0` for `BlendNetwork` (`core/src/sdp/mod.rs:292-320`). It does not have a `BinaryCodec` derive or a custom bincode serializer. The repository's configured bincode format uses fixed-width encoding and a four-byte enum discriminant (`binary-codec/src/bincode/config.rs:11-42`). Consequently, the current `to_bytes()` call supplies four bytes for the unit enum variant (`00 00 00 00`) where the specification requires one byte (`00`). The epoch is correctly emitted as little-endian bytes by the current code, but the preimage is already different before the epoch is appended.

The resulting hash is used as the single `op_id` for every non-zero reward UTXO in `distribute_rewards` (`ledger/src/mantle/sdp/rewards/mod.rs:119-138`). The current Blend-only network remains internally deterministic because all nodes run the same implementation, but its reward note IDs and downstream UTXO identifiers do not match the pinned specification or an independent implementation that follows the canonical preimage. The defect is therefore an interoperability and conformance risk rather than a demonstrated same-version consensus split.

**Exploit scenario**

There is no direct unauthenticated exploit against the current one-service network claimed here. A spec-conforming implementation, an independent test-vector generator, or a future mixed-version implementation computes a different reward operation ID for the same epoch and service. It then derives different reward note IDs and can reject or diverge from the target implementation's reward UTXO set. If a second service type is added, the existing bincode enum representation also makes the serialized prefix depend on bincode's four-byte discriminant rather than the explicitly specified one-byte service code.

**Recommendation**

- *Short term*: Build the reward preimage from the canonical byte directly, for example by hashing `service_type.encode()`/`encode_to_vec()` (or the explicitly defined `AsRef<u8>` value) followed by the four-byte little-endian epoch. Add a test vector covering `BlendNetwork` and a non-zero epoch that asserts the exact preimage and resulting hash.
- *Long term*: Keep consensus and identifier preimages on an explicit canonical-encoding API and prohibit generic bincode `to_bytes()` calls in code that computes IDs, signatures, roots, or other consensus commitments. Add a conformance test that compares the canonical reward preimage with the specification vector and a regression test that fails if bincode and canonical bytes are accidentally assumed equal.

**References**: LIPs `bedrock-service-reward-distribution.md` §Service Reward Distribution; `mantle-transaction-encoding.md` canonical encoding rules; `core/src/sdp/mod.rs:292-320`; `binary-codec/src/bincode/config.rs:11-42`.

## 5. Suggestions (non-security)

### S-001 · Add an explicit Header canonical-equivalence guard

The sweep independently reverified the existing #641 S-003 concern: `Header::sign` uses `header.to_bytes()` (`core/src/header/mod.rs:227-230`) and `verify_header_signature` verifies those same configured-bincode bytes (`core/src/block/mod.rs:369-375`), while the specification defines the canonical 297-byte Header encoding. At the pinned target, the current fixed-width little-endian bincode representation happens to equal the canonical Header representation for the present fields, and the core test suite passes. That equality is not enforced by the type or a regression test that compares the two encodings.

This report does not assign a second finding identifier. The canonical record remains #641 S-003, with its existing classification and history preserved. Add a direct equality test or, preferably, make signing and verification call the canonical encoder explicitly. The same review found that proposal/body-root and Mantle transaction hash paths already use `BinaryEncode`, while Blend transaction and sync bincode uses are transport/storage envelopes rather than hash or signature preimages.

---

## Appendix: Reviewed rule-outs

- `Header::update_hasher`, `Proposal`, `UncleHeaders`, and `SignedHeader` expose canonical codec paths. `body_root` hashes canonical uncle-header and transaction encodings.
- Mantle transaction hashes prepend the specified domain and use the canonical `OpRefs` encoding; operation-ID implementations reviewed in the transaction list use `encode_to_vec()` for their canonical preimages.
- Blend proposals use `BinaryEncode::encode_to_vec()`. Blend transactions use configured bincode around the transaction's serialized canonical bytes as a message envelope, and the receiver decodes that envelope; this is not a hash or signature preimage.
- Cryptarchia sync messages and stored-block messages use bincode as transport/storage serialization. No reviewed sync-message bytes are directly signed or hashed as a consensus commitment.
- The prior #245 report included the reward path in its scope but did not record this `ServiceType`/reward-op-ID mismatch. This is an independent re-verification at the authoritative #744 target, not a duplicate of a canonical filed finding.
