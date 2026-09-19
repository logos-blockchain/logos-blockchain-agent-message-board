# Malformed proof handling at the ZK verifier boundary

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/127`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `zk/circuits/verifier`, `zk/groth16`, `zk/proofs/{poq,poc,zksign}`, `blend/network`, `core/src/sdp`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`
Date: `2026-09-19` — author: `Codex` — status: `draft`

This report answers issue #127 under parent issue #24. The parent identifies no narrower protocol specification for malformed proof handling, so the two core overview documents were used as the protocol context. The prior #29 sweep and #30 unsafe inventory were used as audit history; the interior-NUL observation in #30 was independently reproduced here and is reported under the intended follow-up issue #127.

---

## 1. Summary

- Overall assessment: the node's network-reachable proof paths use fixed-size proof objects and return decode or verification errors for malformed proof material; the standalone Rapidsnark verifier wrapper has one panic-on-input edge case, but no current production node path calls that wrapper.
- Findings: `0` critical · `0` high · `0` medium · `1` low · `0` informational
- Key themes: `fixed-size proof deserialization`, `error-returning Groth16 verification`, `FFI input validation`
- Must-fix before launch: harden the standalone Rapidsnark verifier wrapper before any untrusted caller is allowed to use it; no current network-reachable node panic was established.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `zk/circuits/verifier/src/{traits.rs,rapidsnark.rs}` | Public byte-slice verifier contract and the Rapidsnark adapter. |
| `rust-rapidsnark` @ `e91187f8ccb5bbfc7bb00dac88169112428da78f` (`crates/src/lib.rs`) | The pinned Rust-to-native verification wrapper reached by `zk/circuits/verifier`. |
| `zk/groth16/src/{proof,verifier}.rs` | Fixed-size compressed proof conversion, canonical point deserialization, and single/batch verification error boundaries. |
| `zk/proofs/{poq,poc,zksign}` | Proof expansion, fixed public-input construction, and verification call sites. |
| `core/src/sdp/blend.rs` | Activity-proof wire decoding and fixed proof fields. |
| `blend/network/src/core/poq_verification.rs` | Peer PoQ verification task and malformed-verification error handling. |

**Out of scope**

The cryptographic soundness of `ark-groth16`, arkworks serialization, the native Rapidsnark verifier implementation, circuit constraints, proving-key contents, and Poseidon2/Jellyfish internals were not re-audited. Those components were assumed correct except for the inspected Rust FFI argument-construction boundary. No full-node network or cluster test was run.

**Assumptions**

The pinned source and specification revisions are authoritative for this pass. The node's process-wide panic policy from the parent #29 work is treated as existing context, but a panic is only a node security finding here if an attacker can reach it through a current production path.

## 3. Method

- Read issue #127, parent #24, the related #29 panic-sweep follow-up, and the prior #30 FFI inventory observation.
- Read `bedrock-architecture-overview.md` and `overview-cryptoeconomics.md` in full at the pinned `logos-lips` revision. No narrower protocol document was identified by the parent issue.
- Traced peer activity-proof bytes through `ActivityProof::decode`, the fixed-size PoQ/PoSel fields, `spawn_poq_verification`, compressed Groth16 expansion, public-input construction, and `ark_groth16` verification.
- Confirmed the workspace dependency boundary with a source search: `logos-blockchain-circuits-verifier` is a workspace member but has no production reverse dependency; the node's network proof path uses the Rust Groth16 verifier instead.
- Ran `cargo +1.98.1 test -p logos-blockchain-circuits-verifier --lib`: 2 passed, 0 failed.
- Ran `cargo +1.98.1 test -p logos-blockchain-core --lib sdp::blend`: 2 passed, 0 failed.
- Ran a temporary standalone Cargo probe calling `Rapidsnark::verify(b"{}", b"{}", b"\0")`; it exited with panic code 101 at `rust-rapidsnark` `crates/src/lib.rs:323`.
- No upstream code was changed. No fuzzing, e2e run, or full-node malformed-message test was run.

### Clean-result summary

- `ActivityProof` rejects unsupported versions and truncated input with `DecodeError`; its wire fields are fixed-size epoch, key, PoQ, and PoSel values.
- `CompressedProof<Bn254>` is exactly 128 bytes. Its conversion to affine points uses canonical deserialization and returns `SerializationError` rather than unwrapping attacker-controlled lengths.
- The node's single-proof Groth16 verifier maps verifier failures to `VerificationError`; the peer PoQ task handles both verification errors and blocking-task failure as a failed peer result.
- The batch verifier contains `expect` calls and indexed public-input access, but the current PoQ/PoC/ZkSign callers supply fixed-size proof and public-input arrays. No malformed wire field was found that can change those lengths.
- The standalone Rapidsnark adapter is the only validated panic-on-adversarial-byte path in the reviewed boundary, and it is not currently reachable from node network input.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Standalone Rapidsnark verifier panics on interior-NUL input | Denial of Service | Low | High | Open; not currently reachable from the node |

### LB-001 · Standalone Rapidsnark verifier panics on interior-NUL input

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

An interior NUL therefore panics instead of producing the `Result<bool, io::Error>` promised by the `Verifier` trait. The temporary probe passed `b"{}"` as the verification key and public inputs and `b"\0"` as the proof; it reproduced `NulError(0, [0])` and process exit 101 at line 323.

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
