# Audit Report — Binary witness inputs and secret-memory testing

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/273`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `zk/proofs/{pol,poq,poc,zksign}` and the circuits witness-generator FFI
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `cryptarchia-proof-of-leadership.md`, `proof-of-quota.md`, `common-cryptographic-components.md`, `trusted-setup-ceremony.md`
Circuits: `https://github.com/logos-blockchain/logos-blockchain-circuits` @ `ebf7ddf5b5b625ef4ea7c803171c670beebd05ef` (tag `v0.5.7`), verification-key hash: not applicable to this FFI/secret-lifetime review; verifier and verification-key consistency were out of scope
Circom: `v2.2.2` (the version used by the pinned circuits build)
Date: `2026-09-14` — author: `Codex` — status: `final`

---

## 1. Summary

- Overall assessment: In-memory witness generation retains secret-bearing representations across decimal-JSON/native/runtime stages, and memory-based FFI circuit-data loading lacks pre-read length validation.
- Findings: `C` 0 · `H` 0 · `M` 0 · `L` 2 · `I` 0
- Key themes: post-use secret remanence, unsafe initial circuit-data loading, and an unambiguous binary input contract
- Must-fix before launch: No unconditional launch blocker was demonstrated in this review. LB-001 is recommended before production where crash dumps, post-use memory disclosure, forensic capture, or equivalent local-memory threats are in scope. LB-002 must be fixed before exposing any caller-controlled circuit-data path.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `zk/proofs/{pol,poq,poc,zksign}/src/inputs.rs` | Rust conversion of prover inputs into the native witness FFI |
| `zk/groth16/src/public_input/{mod.rs,deserialize.rs}` | Per-field decimal deserialization representation used by witness inputs |
| `logos-blockchain-circuits/src/{types.hpp,circom_adapter.cpp}` | Witness-input ABI, circuit-data loading, JSON parsing, and `.wtns` construction |
| `logos-blockchain-circuits/src/{pol,poq,poc,signature}/ffi.cpp` | Four memory-based exported witness-generation entry points and argument validation |
| `logos-blockchain-circuits/rust/logos-blockchain-circuits-{types,tests}` | Rust FFI ownership and existing witness-generator tests |
| `logos-blockchain-circuits/.github/resources/witness-generator/fix_calcwit_leak.sh` | Pinned generated-runtime destructor hook and per-call `signalValues` allocation lifetime |
| Logos LIPs listed above | Field encoding, PoL/PoQ private inputs, and trusted-setup context |

**Out of scope**

The verifier paths, Groth16 implementation, Circom constraint logic, generated circuit arithmetic, and the file-based Circom command path were not audited beyond their role in the FFI boundary. Generated Circom runtime internals beyond the pinned allocation/destructor hook were not independently audited. `serde_json`, `nlohmann::json`, and the allocator were treated as third-party dependencies; the findings concern the surrounding ownership and ABI contracts.

**Assumptions**

The four specifications at the recorded LIPs commit are the intended protocol reference, the trusted setup is honest, and the host process is not already compromised. The current node wrappers provide embedded circuit data, so the malformed-`dat` issue is not shown to be network-reachable through the node at this commit; it affects native FFI callers and future callers that accept caller-supplied circuit data. The report treats arbitrary live-process memory inspection during witness generation as outside the post-use remanence property; such an inspector can observe secrets that necessarily exist while computation is active.

## 3. Method

- Read parent issue `#16`, source issue `#273`, and the prior witness-hygiene report for `#224`.
- Read the four area specifications in full at the commit recorded above. In particular, the LIPs describe BN254 field elements as 32-byte little-endian values when serialized, and identify private note, quota, proof-of-leadership, and signing inputs that must remain secret.
- Re-verified only the claims corrected here: Rust per-field and aggregate serialization, Rust/C++ ownership, generated-runtime allocation hooks, cache initialization, safe embedded-circuit bindings, circuit-data parsing, and the witness-output failure path at the pinned commits.
- Reviewed the Circom v2.2.2 build configuration and the `InputHashMap` metadata contract needed to define binary input ordering; raw open-addressed table storage/iteration order is not treated as an ABI order.
- Used targeted `git`, `sed`, `nl`, and `rg` inspection. Automated tooling: none. Dynamic testing: none; this was a report-only pass and no generated-circuit rebuild or memory scanner is present in the test crate.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Decimal JSON and witness buffers retain private inputs | Data Exposure | Low | High | Open; re-verified residual from `#224` |
| LB-002 | Truncated circuit-data buffers are read past their declared length | Undefined Behavior / Memory safety | Low | High | Open |

### LB-001 · Decimal JSON and witness buffers retain private inputs

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Data Exposure |
| Target | `zk/proofs/{pol,poq,poc,zksign}/src/inputs.rs:21-27,94-101`; `zk/groth16/src/public_input/{mod.rs,deserialize.rs}`; circuits `rust/logos-blockchain-circuits-types/src/native/{witness_input.rs,witness.rs}`, `src/types.hpp:69-75`, and `src/circom_adapter.cpp:105-202` |
| Status | Open; the binary replacement and memory-scan test requested by `#224/#273` are not implemented |

**Description**

The problem here is post-use secret remanence, not protection against an inspector that can read arbitrary live prover memory while witness generation is running. The audited lifetime crosses more secret-bearing representations than only the final aggregate JSON string:

`typed field inputs → per-field decimal Strings → aggregate JSON String → CString → nlohmann JSON trees / temporary native field vectors → Circom signal storage → C++ .wtns staging buffer → C malloc result → Rust witness copy`

At the Rust boundary, `Groth16Input` values are converted through `Groth16InputDeser(pub String)` (`zk/groth16/src/public_input/{mod.rs,deserialize.rs}`). The four in-memory prover paths build their typed input structures, convert them into per-field decimal strings, serialize the aggregate object with `serde_json::to_string`, and pass the resulting `String` to `WitnessInput::new`, which stores a `CString` (`zk/proofs/{pol,poq,poc,zksign}/src/inputs.rs:21-27,94-101`; circuits `rust/logos-blockchain-circuits-types/src/native/witness_input.rs:5-20`). The C ABI exposes only the null-terminated `inputs_json` pointer.

`loadJson` parses that text into `jin`, qualifies it into a second JSON tree `j`, and converts each value into a temporary native field-element vector before setting Circom signals (`src/circom_adapter.cpp:105-146`). These secret-bearing representations are not scrubbed before their storage is released. The C++ `writeBinWitness` path additionally accumulates witness field elements, including private values, in `buf`, uses a local `FrElement v` while serializing them, copies the result to a `malloc` allocation, and `free_bytes` releases that allocation without wiping it (`src/circom_adapter.cpp:148-202`; circuits Rust `native/witness.rs` and `ffi/bytes.rs`). The pinned Circom build also patches `Circom_CalcWit` to free its per-call `signalValues`; that allocation must be wiped before the patched destructor frees it. The generated runtime was not otherwise independently audited, so this report does not assert additional unverified copies.

This is a residual of the issue `#224` witness-hygiene review, not a claim that the earlier report's proposed patch landed. The four Rust call sites still use `serde_json::to_string`, and the audited circuits checkout still has the original JSON loader and unscrubbed serializer buffer.

**Exploit scenario**

After proof generation, a crash dump, forensic memory capture, allocator reuse, or another local memory disclosure could expose stale decimal or canonical field-element representations of note secrets, quota secrets, or signing keys. The relevant scenario is a later disclosure or another local weakness reading memory after intended use; post-use zeroization cannot prevent an attacker who can inspect arbitrary live prover memory during computation. No remote or unauthenticated node exploit was demonstrated.

**Recommendation**

- *Immediate hardening*: zeroize the aggregate Rust JSON `String` and native `CString`; eliminate or zeroize secret-bearing per-field decimal `String`s where practical; store returned witness bytes in zeroizing mutable storage; wipe the C-allocated witness buffer before `free`; wipe the C++ `writeBinWitness` staging buffer and its verified `FrElement` temporary; wipe Circom `signalValues` before the patched destructor frees it; and cover any other native temporary verified to contain witness material. Keep the existing file-based JSON path available for CLI tooling if needed.
- *Structural fix*: replace the in-memory decimal-JSON ABI with a binary field-element ABI. The required ABI must carry canonical 32-byte little-endian BN254 field elements, validate exact length, and reject non-canonical values with `value < modulus`. Its ordering must be an explicit, deterministic, versioned per-circuit input manifest generated during circuit compilation, mapping every element unambiguously to a circuit input and respecting `signalsize`. Do not use raw `InputHashMap` storage or iteration order: at Circom v2.2.2 it is an open-addressed hash table containing entries such as `hash`, `signalid`, and `signalsize`, not an ABI ordering contract. If signal metadata is used instead of a generated manifest, the deterministic derivation and its `signalsize` handling must be explicitly documented. Keep the file-based JSON path for CLI tooling if needed.
- *Verification*: extend the lifetime tests across the caller buffer, per-field/aggregate strings, FFI temporaries, generated signal storage, `.wtns` buffer, C result, and Rust wrapper, while acknowledging allocator, stack/register, compiler, unrelated-mapping, and false-positive limitations.

**References**: PoL private inputs and field serialization in `cryptarchia-proof-of-leadership.md`; PoQ witness/private inputs in `proof-of-quota.md`; field-element encoding in `common-cryptographic-components.md`; prior report `#224`.

### LB-002 · Truncated circuit-data buffers are read past their declared length

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Undefined Behavior / Memory safety |
| Target | `logos-blockchain-circuits/src/{poc,pol,poq,signature}/ffi.cpp:65-110` and `src/circom_adapter.cpp:9-49`; circuits Rust `rust/logos-blockchain-circuits-types/src/native/circuit_witness_input.rs:5-20` |
| Status | Open |

**Description**

The four memory-based validators reject a null or zero-length `dat` buffer but do not check a minimum or exact size. `loadCircuit` immediately copies the generated input-hash map, witness list, and constants with fixed-size `memcpy` operations from `circuit_bytes.data`. It only uses the declared size later while parsing the optional I/O map. Consequently, a non-null buffer whose declared length is shorter than the first fixed region is read beyond its contract before an error can be returned.

The cache behavior narrows the timing: `getCachedCircuit` initializes the circuit once with `std::call_once` and returns that cached circuit for the process lifetime. A malformed buffer must therefore affect the initial circuit load; later differing `dat` arguments are not reparsed as new circuits. The audited safe Rust wrappers do not expose arbitrary circuit data: each `CircuitWitnessInput` passes its circuit type's compile-time `Dat::DAT` to the FFI. No current node or network path to caller-selected circuit data was demonstrated.

**Exploit scenario**

A low-level native/FFI caller able to control the `.dat` buffer used for first circuit initialization supplies a truncated buffer. The loader may segfault, read unrelated mapped memory as circuit metadata, or continue into invalid allocations and crash. A future API accepting caller-controlled circuit data could create the same condition. The audited safe Rust/node path uses compile-time embedded circuit data, and no current network path to this primitive was demonstrated.

**Recommendation**

- *Short term*: validate the complete expected serialized layout, including checked arithmetic for each variable-length section, before any read and before one-time cache initialization; reject truncated and trailing data according to a documented exact-size policy. Define and reject inconsistent later `dat` arguments instead of silently ignoring them after cache initialization. Add tests for one-byte, each fixed-region truncation, and malformed variable-length maps.
- *Long term*: do not let callers select the process-wide cached circuit with arbitrary bytes. Bind each exported generator directly to its compiled circuit artifact (or a validated immutable circuit handle/hash), and make cache identity explicit rather than dependent on whichever caller initializes it first.

**References**: the FFI contract in `src/types.hpp`; generated `.dat` loading in `src/circom_adapter.cpp`.

## 5. Suggestions

### S-001 · Add the requested Linux memory-scan regression test

The circuits test crate has no `/proc/self/mem` or equivalent memory-hygiene test. Prefer a parent/child design so the process being scanned does not retain the test's comparison needle:

1. The parent generates and retains a unique marker and its decimal and canonical 32-byte little-endian representations, then passes only the input needed for the child to perform witness generation over a controlled IPC channel.
2. The child performs witness generation, drops or zeroizes all intended secret-containing objects, and pauses at a synchronization point.
3. The parent scans selected writable mappings in the child process and searches for both the decimal representation and the canonical field representation.
4. The child exits after the scan, allowing the parent to report the result without placing the marker in the child's assertion data.

The test should document limitations from allocator behavior, unrelated mappings, stack/register/compiler copies, optimization, and false positives from intentionally retained values. It should demonstrate the intended post-use secret-lifetime property, not claim that a secret never existed anywhere in process memory.

### S-002 · Propagate witness-output allocation failure

`writeBinWitness` returns `void`; if `malloc` fails at `src/circom_adapter.cpp:197-201`, the caller still returns `StatusCode_Ok` from the FFI implementation at `src/{poc,pol,poq,signature}/ffi.cpp:107-110`, leaving the output empty. Return an explicit out-of-memory status and test that the output invariant holds on failure.

## Appendix A — Definitions

Severity and difficulty use the definitions in `docs/REPORT_TEMPLATE.md`. `Low` reflects privileged/local preconditions or a non-current call path; it does not mean the secret-memory gap is harmless.
