# Malformed proof handling at the ZK verifier boundary

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/127`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `zk/circuits/verifier`, `zk/groth16`, `zk/proofs/{pol,poc,poq,zksign}`, `blend/network`, `core/src/proofs`, `core/src/sdp`, `ledger/src/cryptarchia`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`
Date: `2026-09-19` — author: `Codex` — status: `final`

This report answers issue #127 under parent issue #24. The parent identifies no narrower protocol specification for malformed proof handling, so the two core overview documents were used as the protocol context. The prior #29 sweep and #30 unsafe inventory were used as audit history; the interior-NUL observation in #30 was independently reproduced here and is reported under the intended follow-up issue #127.

---

## 1. Summary

- Overall assessment: the node's network-reachable proof paths use fixed-size proof objects and return decode or verification errors for malformed proof material; the standalone Rapidsnark verifier wrapper has one panic-on-input edge case, but no current production node path calls that wrapper.
- Findings: `0` critical · `0` high · `0` medium · `1` low (`127-LB-001`, new under #127) · `0` informational
- Key themes: `fixed-size proof deserialization`, `error-returning Groth16 verification`, `FFI input validation`
- Must-fix before launch: harden the standalone Rapidsnark verifier wrapper before any untrusted caller is allowed to use it; no current network-reachable node panic was established.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `zk/circuits/verifier/src/{traits.rs,rapidsnark.rs}` | Public byte-slice verifier contract and the Rapidsnark adapter. |
| `rust-rapidsnark` @ `e91187f8ccb5bbfc7bb00dac88169112428da78f` (`crates/src/lib.rs`) | The pinned Rust-to-native verification wrapper reached by `zk/circuits/verifier`. |
| `core/src/proofs/leader_proof.rs`, `core/src/proofs/leader_claim_proof.rs` | Proof-of-Leadership block/uncle verification and leader-claim/PoC verification, including wire decoding and error-to-false handling. |
| `ledger/src/cryptarchia/mod.rs` | Block and uncle leadership-proof application path. |
| `zk/groth16/src/{proof,verifier}.rs` | Fixed-size compressed proof conversion, canonical point deserialization, and single/batch verification error boundaries. |
| `zk/proofs/{pol,poc,poq,zksign}` | Proof expansion, fixed public-input construction, and verification call sites. |
| `core/src/sdp/blend.rs` | Activity-proof wire decoding and fixed proof fields. |
| `blend/network/src/core/poq_verification.rs` | Peer PoQ verification task and malformed-verification error handling. |

**Out of scope**

The cryptographic soundness of `ark-groth16`, arkworks serialization, the native Rapidsnark verifier implementation, circuit constraints, proving-key contents, and Poseidon2/Jellyfish internals were not re-audited. The native implementation was exercised only through the isolated malformed-input subprocess matrix; the Rust FFI argument-construction boundary was inspected directly. No full-node network or cluster test was run.

**Assumptions**

The pinned source and specification revisions are authoritative for this pass. The node's process-wide panic policy from the parent #29 work is treated as existing context, but a panic is only a node security finding here if an attacker can reach it through a current production path.

## 3. Method

- Read issue #127, parent #24, the related #29 panic-sweep follow-up, and the prior #30 FFI inventory observation.
- Read `bedrock-architecture-overview.md` and `overview-cryptoeconomics.md` in full at the pinned `logos-lips` revision. No narrower protocol document was identified by the parent issue.
- Traced the Proof-of-Leadership block/uncle path through `Groth16LeaderProof::decode`, `CryptarchiaLedger::try_apply_header` / `verify_proof_of_leadership`, and `lb_pol::verify`; traced leader-claim proofs through `Groth16LeaderClaimProof::decode` and deferred `lb_poc::batch_verify`.
- Traced peer activity-proof bytes through `ActivityProof::decode`, the fixed-size PoQ/PoSel fields, `spawn_poq_verification`, compressed Groth16 expansion, public-input construction, and `ark_groth16` verification.
- Confirmed the workspace dependency boundary with a source search: `logos-blockchain-circuits-verifier` is a workspace member but has no production reverse dependency; the node's network proof path uses the Rust Groth16 verifier instead.
- Ran `cargo +1.98.1 test -p logos-blockchain-circuits-verifier --lib`: 2 passed, 0 failed.
- Ran `cargo +1.98.1 test -p logos-blockchain-core --lib sdp::blend`: 2 passed, 0 failed.
- Ran an isolated subprocess matrix against the pinned `rust-rapidsnark` `e91187f8ccb5bbfc7bb00dac88169112428da78f`: malformed proof JSON, malformed public-input JSON, valid JSON with wrong proof/input structure or lengths, malformed verification-key JSON, and an invalid curve/proof value all returned an error or `Ok(false)`; no abort, signal termination, or index-out-of-bounds result was observed. An interior-NUL proof exited with panic code 101 at `crates/src/lib.rs:323`, before the native call.
- No upstream code was changed. No fuzzing, e2e run, or full-node malformed-message test was run.

### Clean-result summary

- `ActivityProof` rejects unsupported versions and truncated input with `DecodeError`; its wire fields are fixed-size epoch, key, PoQ, and PoSel values.
- `CompressedProof<Bn254>` is exactly 128 bytes. Its conversion to affine points uses canonical deserialization and returns `SerializationError` rather than unwrapping attacker-controlled lengths.
- `Groth16LeaderProof` decodes a fixed-size PoL proof plus typed entropy, leader-key, and voucher-commitment fields. The block/uncle path builds typed `LeaderPublic` inputs and maps `lb_pol::verify` errors to `false`; `Groth16LeaderClaimProof` likewise decodes a fixed-size PoC proof and maps `lb_poc::verify` errors to `false` before its deferred batch path.
- The node's single-proof Groth16 verifier maps verifier failures to `VerificationError`; the peer PoQ task handles both verification errors and blocking-task failure as a failed peer result.
- The batch verifier contains `expect` calls and indexed public-input access, but the current PoQ/PoC/ZkSign callers supply fixed-size proof and public-input arrays. No malformed wire field was found that can change those lengths.
- The standalone Rapidsnark adapter is the only validated panic-on-adversarial-byte path in the reviewed boundary, and it is not currently reachable from node network input.

### Native FFI malformed-input matrix

Each case was executed in a fresh subprocess so an abort or segmentation fault could not terminate the audit harness. The malformed JSON/shape cases reached the native verifier and returned an error; the invalid curve/proof case reached native verification and returned `Ok(false)`.

| Case | Observed result |
|---|---|
| Malformed JSON proof | `Err(Proof verification failed: invalid proof data)` |
| Malformed public-input JSON | `Err(Proof verification failed: invalid inputs data)` |
| Valid JSON with wrong proof structure | `Err(Proof verification failed: invalid proof data)` |
| Valid JSON with wrong public-input length | `Err(Proof verification failed: invalid inputs data)` |
| Malformed verification key JSON | `Err(Proof verification failed: invalid verification key data)` |
| Invalid curve/proof value | `Ok(false)` |
| Interior NUL in proof | Rust panic, exit `101`, at `rust-rapidsnark` `crates/src/lib.rs:323`; this fails before native verification |

This matrix did not observe an assertion, abort, signal, or index-out-of-bounds failure for the NUL-free native-boundary cases. It does not establish that every possible native malformed input is safe; it establishes the requested representative boundary behavior and preserves the separately reported Rust-wrapper panic.

### Public-input canonicality

| Production path | Public-input construction | Canonicality result |
|---|---|---|
| PoL block/uncle proof | `Groth16LeaderProof::verify` derives a typed `PolVerifierInput` from `LeaderPublic` state, slot/nonce/lottery values, and the typed Ed25519 leader key; `PolVerifierInput::to_inputs` returns a fixed `[Fr; 9]`. | Derived from strongly typed values; no attacker-controlled JSON reaches the verifier. `Fr` values are canonical at construction. |
| Leader claim / PoC | `LeaderClaimOp::verify` constructs typed `PoCVerifierInput` from the checked voucher nullifier/root and transaction hash; `to_inputs` returns a fixed `[Fr; 3]`. | Derived from strongly typed ledger values; no direct attacker-byte public-input parser is used in production. |
| Blend PoQ | `RealProofsVerifier` combines current epoch state and the typed signing key with the proof's fixed-size key nullifier; `PoQVerifierInputData` converts to fixed typed `Groth16Input` values, while `ProofOfQuota::decode` parses the nullifier with canonical `fr_from_bytes`. | Public inputs are derived internally and parsed as canonical field elements; attacker bytes cannot choose the public-input vector length. |
| ZkSign | Operation verification derives `ZkSignVerifierInputs` from the typed transaction hash and declaration/public keys; `as_inputs` returns a fixed `[Fr; 33]`. | Derived from strongly typed values and canonical field wrappers; no raw attacker-controlled public-input JSON is accepted on the node path. |

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| 127-LB-001 | Standalone Rapidsnark verifier panics on interior-NUL input | Denial of Service | Low | High | Open; not currently reachable from the node |

### 127-LB-001 · Standalone Rapidsnark verifier panics on interior-NUL input

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Denial of Service |
| Target | `zk/circuits/verifier/src/rapidsnark.rs:10-21` (`Rapidsnark::verify`); pinned `rust-rapidsnark` `crates/src/lib.rs:321-325` (`groth16_verify_wrapper`) |
| Status | Open; not currently reachable from the node |

**Description**

`Rapidsnark::verify` accepts arbitrary byte slices and first converts each input with `std::str::from_utf8`. A NUL byte is valid UTF-8, so this conversion does not reject an input such as `b"\0"`. The adapter then calls `rust_rapidsnark::groth16_verify_wrapper`.

At the pinned `rust-rapidsnark` revision, the wrapper constructs all three C strings with unconditional `unwrap()` calls:

```rust
let proof_cstr = std::ffi::CString::new(proof).unwrap();
let inputs_cstr = std::ffi::CString::new(inputs).unwrap();
let verification_key_cstr = std::ffi::CString::new(verification_key).unwrap();
```

An interior NUL therefore panics instead of producing the `Result<bool, io::Error>` promised by the `Verifier` trait. The subprocess matrix passed valid-UTF-8 proof bytes containing an interior NUL and reproduced `NulError` and process exit 101 at line 323.

This is not a current remote node-crash finding. The workspace search found no production crate that depends on `logos-blockchain-circuits-verifier`; its only references are the verifier crate itself and its README/test target. The node's peer PoQ path instead decodes fixed-size proof fields, expands them with `TryFrom`, and calls the Rust `ark-groth16` verifier. Thus an adversarial peer can reach malformed proof bytes in the node path, but no path from those bytes to these `CString::new(...).unwrap()` calls was identified at `a805329f`.

**Exploit scenario**

A future or external caller exposes `Rapidsnark::verify` to an untrusted party and forwards raw proof, public-input, or verification-key bytes. Supplying a valid-UTF-8 byte slice containing `0x00` causes the process to panic before native verification. If that caller installs the node's fail-stop panic policy, the caller process exits; today, however, the workspace does not wire this verifier into the node's network or block-validation path.

**Recommendation**

- *Short term*: replace each `CString::new(...).unwrap()` with error propagation, mapping `NulError` to the wrapper's `anyhow::Result` (and then to the adapter's `io::Error`). Add regression tests for NUL in proof, public-input, and verification-key bytes, alongside invalid UTF-8 and malformed JSON cases.
- *Long term*: prefer a length-aware native FFI API, or validate the exact JSON/byte contract before crossing the FFI boundary, so C-string termination is not an implicit input invariant. Keep the verifier crate's dependency boundary explicit until it has complete adversarial-input tests.

**References**: issue #127; parent issue #24; prior #29 follow-up; prior #30 S-004; Rust `CString::new` interior-NUL contract.

## 5. Suggestions

### S-001 · Add malformed fixed-proof regression coverage

Add tests for all node-facing proof decoders using truncated bytes, extra trailing bytes, all-zero fixed-size proofs, non-canonical field encodings, and invalid curve encodings. Assert that each case returns `DecodeError`, `SerializationError`, or a verification error rather than panicking. The existing activity-proof tests cover an unsupported version and a short input, and the standalone verifier tests cover one valid proof and one invalid proof, but they do not cover the full malformed-input matrix.

### S-002 · Preserve the verifier dependency boundary

Add a lightweight dependency/reverse-dependency check or documentation stating that `logos-blockchain-circuits-verifier` is a standalone utility and is not part of the node's untrusted network path. If a production caller is added later, make the adversarial-input tests and the NUL fix a prerequisite for that integration.

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
