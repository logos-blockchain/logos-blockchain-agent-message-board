# Audit Report — Binary witness inputs and secret-memory testing

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/273`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `zk/proofs/{pol,poq,poc,zksign}` and the circuits witness-generator FFI
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `cryptarchia-proof-of-leadership.md`, `proof-of-quota.md`, `common-cryptographic-components.md`, `trusted-setup-ceremony.md`
Circuits: `https://github.com/logos-blockchain/logos-blockchain-circuits` @ `ebf7ddf5b5b625ef4ea7c803171c670beebd05ef` (tag `v0.5.7`), verification-key hash not computed
Date: `2026-09-14` — author: `Codex` — status: `final`

---

## 1. Summary

- Overall assessment: The in-memory witness FFI still converts private inputs through decimal JSON and retains multiple native/Rust copies; its circuit-data boundary also permits out-of-bounds reads from truncated buffers.
- Findings: `C` 0 · `H` 0 · `M` 0 · `L` 2 · `I` 0
- Key themes: secret lifetime and serialization, unchecked FFI buffer lengths
- Must-fix before launch: define and validate a binary field-element input ABI; remove JSON from the in-memory path; scrub all temporary and returned witness buffers; validate circuit-data length before parsing.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `zk/proofs/{pol,poq,poc,zksign}/src/inputs.rs` | Rust conversion of prover inputs into the native witness FFI |
| `logos-blockchain-circuits/src/{types.hpp,circom_adapter.cpp}` | Witness-input ABI, circuit-data loading, JSON parsing, and `.wtns` construction |
| `logos-blockchain-circuits/src/{pol,poq,poc,signature}/ffi.cpp` | Four memory-based exported witness-generation entry points and argument validation |
| `logos-blockchain-circuits/rust/logos-blockchain-circuits-{types,tests}` | Rust FFI ownership and existing witness-generator tests |
| Logos LIPs listed above | Field encoding, PoL/PoQ private inputs, and trusted-setup context |

**Out of scope**

The verifier paths, Groth16 implementation, Circom constraint logic, generated circuit arithmetic, and the file-based Circom command path were not audited beyond their role in the FFI boundary. `serde_json`, `nlohmann::json`, Circom runtime code, and the allocator were treated as third-party dependencies; the findings concern the surrounding ownership and ABI contracts.

**Assumptions**

The four specifications at the recorded LIPs commit are the intended protocol reference, the trusted setup is honest, and the host process is not already compromised. The current node wrappers provide embedded circuit data, so the malformed-`dat` issue is not shown to be network-reachable through the node at this commit; it affects native FFI callers and future callers that accept caller-supplied circuit data.

## 3. Method

- Read parent issue `#16`, source issue `#273`, and the prior witness-hygiene report for `#224`.
- Read the four area specifications in full at the commit recorded above. In particular, the LIPs describe BN254 field elements as 32-byte little-endian values when serialized, and identify private note, quota, proof-of-leadership, and signing inputs that must remain secret.
- Reviewed the Rust input conversions, native bindings, C++ ABI, circuit-data loader, JSON loader, witness serializer, and circuits test crate at the pinned commits.
- Used targeted `git`, `sed`, `nl`, and `rg` inspection. Automated tooling: none. Dynamic testing: none; this was a report-only pass and no generated-circuit rebuild or memory scanner is present in the test crate.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Decimal JSON and witness buffers retain private inputs | Data Exposure | Low | High | Open; re-verified residual from `#224` |
| LB-002 | Truncated circuit-data buffers are read past their declared length | Undefined Behavior / Memory safety | Low | Medium | Open |

### LB-001 · Decimal JSON and witness buffers retain private inputs

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Data Exposure |
| Target | `zk/proofs/{pol,poq,poc,zksign}/src/inputs.rs:21-27,94-101` and circuits `src/types.hpp:69-75`, `src/circom_adapter.cpp:105-202` |
| Status | Open; the binary replacement and memory-scan test requested by `#224/#273` are not implemented |

**Description**

Each in-memory Rust prover path serializes typed inputs into a decimal JSON `String` and constructs a native `CString`. The C ABI exposes only a null-terminated `inputs_json` pointer. `loadJson` then parses that text into `jin`, qualifies it into a second JSON tree `j`, and converts each value into temporary field-element vectors. These copies are not scrubbed before their storage is released. The C++ `writeBinWitness` path additionally accumulates every witness field element, including private values, in `buf`, copies it to a `malloc` allocation, and `free_bytes` releases that allocation without wiping it.

This is a residual of the issue `#224` witness-hygiene review, not a claim that the earlier report's proposed patch landed. The four Rust call sites still use `serde_json::to_string`, and the audited circuits checkout still has the original JSON loader and unscrubbed serializer buffer.

**Exploit scenario**

An operator, crash-dump reader, or other actor with sufficiently privileged access to a prover process can inspect heap pages after the caller has dropped its typed input and recover recognizable decimal or field-element representations of note secrets, quota secrets, or signing keys. This is not a demonstrated remote or unauthenticated node exploit, but it defeats the intended short lifetime of private witness material and can turn a local memory disclosure into key compromise.

**Recommendation**

- *Short term*: add a separate binary in-memory ABI carrying an exact sequence of 32-byte little-endian BN254 field elements, with the length checked before parsing and each element checked to be below the field modulus. Keep JSON only for `generate_witness_from_files`. Build the Rust input buffer once in `Zeroizing<Vec<u8>>`, and ensure the native output/witness representation is also wiped on release.
- *Long term*: make input order a deterministic artifact-level contract derived from the generated circuit metadata (rather than map iteration order), expose or version that manifest with the circuit, and test the full lifetime: caller buffer, FFI temporaries, generated signal storage, `.wtns` buffer, and Rust wrapper all contain no marker after drop. Generated-runtime and C++ temporary storage must be handled as part of the same secret boundary.

**References**: PoL private inputs and field serialization in `cryptarchia-proof-of-leadership.md`; PoQ witness/private inputs in `proof-of-quota.md`; field-element encoding in `common-cryptographic-components.md`; prior report `#224`.

### LB-002 · Truncated circuit-data buffers are read past their declared length

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Undefined Behavior / Memory safety |
| Target | `logos-blockchain-circuits/src/{poc,pol,poq,signature}/ffi.cpp:65-110` and `src/circom_adapter.cpp:19-49` |
| Status | Open |

**Description**

The four memory-based validators reject a null or zero-length `dat` buffer but do not check a minimum or exact size. `loadCircuit` immediately copies the generated input-hash map, witness list, and constants with fixed-size `memcpy` operations from `circuit_bytes.data`. It only uses the declared size later while parsing the optional I/O map. Consequently, a non-null buffer whose declared length is shorter than the first fixed region is read beyond its contract before an error can be returned. `getCachedCircuit` also caches the first buffer supplied for the process lifetime.

**Exploit scenario**

A native caller, or a future API accepting untrusted circuit data, supplies a truncated `.dat` buffer. The loader may segfault, read unrelated mapped memory as circuit metadata, or continue into invalid allocations and crash. The current Rust wrappers pass compile-time embedded circuit data, so I found no current network-to-this-primitive path; the exported C ABI still exposes the unsafe boundary.

**Recommendation**

- *Short term*: validate the complete expected serialized layout, including checked arithmetic for each variable-length section, before any read; reject truncated and trailing data according to a documented exact-size policy. Add tests for one-byte, each fixed-region truncation, and malformed variable-length maps.
- *Long term*: do not let callers select the process-wide cached circuit with arbitrary bytes. Bind each exported generator to its compiled circuit artifact (or a validated immutable handle and artifact hash), and make cache initialization independent of caller-controlled input.

**References**: the FFI contract in `src/types.hpp`; generated `.dat` loading in `src/circom_adapter.cpp`.

## 5. Suggestions

### S-001 · Add the requested Linux memory-scan regression test

The circuits test crate has no `/proc/self/mem` or equivalent memory-hygiene test. Add a Linux-only test that uses a unique marker represented both as the decimal JSON value and as its 32-byte little-endian encoding, runs witness generation, drops and zeroizes all caller-side buffers and the returned witness, then scans only a controlled process/subprocess heap region. The test must not retain the marker in its own assertion data while scanning, and should document that allocator reuse and unrelated mappings can make a broad process-wide scan noisy.

### S-002 · Propagate witness-output allocation failure

`writeBinWitness` returns `void`; if `malloc` fails at `src/circom_adapter.cpp:197-201`, the caller still returns `StatusCode_Ok` from the FFI implementation at `src/{poc,pol,poq,signature}/ffi.cpp:107-110`, leaving the output empty. Return an explicit out-of-memory status and test that the output invariant holds on failure.

## Appendix A — Definitions

Severity and difficulty use the definitions in `docs/REPORT_TEMPLATE.md`. `Low` reflects privileged/local preconditions or a non-current call path; it does not mean the secret-memory gap is harmless.
