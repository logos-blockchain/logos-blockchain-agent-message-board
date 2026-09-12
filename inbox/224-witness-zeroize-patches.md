# Audit Report — Witness memory hygiene, fix pass: scrubbed witness copies on the Rust side, scrubbed C and C++ buffers, and the long-lived prover sized

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/224`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `zk/proofs/{pol,poq,zksign,poc}/src/inputs.rs`, `zk/circuits/prover`; and `https://github.com/logos-blockchain/logos-blockchain-circuits` @ `ebf7ddf5b5b625ef4ea7c803171c670beebd05ef` (tag `v0.5.7`, the revision the node pins) — `rust/logos-blockchain-circuits-types`, `src/circom_adapter.{cpp,hpp}`, `src/{pol,poq,poc,signature}/ffi.cpp`, `.github/resources/witness-generator/fix_calcwit_leak.sh`; `https://github.com/logos-blockchain/logos-blockchain-rust-rapidsnark` @ `e91187f8ccb5bbfc7bb00dac88169112428da78f` read, not patched
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`; by section: `common-cryptographic-components.md` §ZkSignature (Private Parameters), §Groth16
Date: 2026-09-12 — author: `agent (Claude)` — status: `fix-review`

---

## 1. Summary

- Overall assessment: items 1, 2 and 3 of #224 are done as two proposed patches, one per repository, given in full in Appendices B and C. Of the eight heap copies of the private witness inputs that #152 LB-003 counted per proof, six are now scrubbed before their memory is released (the JSON `String`, the C-string copy, the witness signal array, the `.wtns` staging vector, the C `malloc` buffer, and the Rust-side witness), the two `Vec`/`Bytes` copies collapse into one zeroizing buffer, and the two nlohmann JSON trees remain (§4.3 explains why, and what replaces them). Item 4, the long-lived prover, is sized and not implemented: the C API already exists in the vendored rapidsnark and only the `rust-rapidsnark` bindings are missing, but #152 S-001 puts the gain at a few percent and the issue makes it conditional on #97's numbers. Item 5 (opening PRs) is not part of a report.
- Findings: 0 critical · 0 high · 0 medium · 0 low · 0 informational (one prior finding moved to *Patched*)
- Key themes: "every buffer that held a witness signal is now overwritten before `free`/`delete`", "the Rust API change is source-compatible: `String` still converts into the new `Zeroizing<String>` parameter", "the one copy that cannot be scrubbed is the decimal-JSON parse tree, so the durable fix is binary inputs"
- Must-fix before launch: none from this issue (the finding was Low); the circuits patch is the one to land first, the node patch follows the tag bump.

## 2. Scope

**In scope**

| Repository / path | Notes |
|---|---|
| circuits `rust/logos-blockchain-circuits-types/src/native/{witness_input.rs, witness.rs, circuit_witness_input.rs}`, `Cargo.toml`, `rust/Cargo.toml` | copies 1, 2, 6, 7, 8 of the #152 table |
| circuits `src/circom_adapter.cpp` L105-L201 (`loadJson`, `writeBinWitness`), `circom_adapter.hpp`, `src/{pol,poq,poc,signature}/ffi.cpp` L89-L111 | copies 3, 5 |
| circuits `.github/resources/witness-generator/fix_calcwit_leak.sh` | copy 4: the generated `~Circom_CalcWit` that the CI build patches in |
| node `zk/proofs/{pol,poq,zksign,poc}/src/inputs.rs` (the four `TryFrom<..> for lbc_*_sys::*WitnessInput` impls), their `Cargo.toml` | copy 1 on the node side; the `Witness` API change |
| node `zk/circuits/prover/src/{rapidsnark.rs,traits.rs}`; rust-rapidsnark `crates/src/lib.rs` L40-L75; circuits `rapidsnark/src/prover.h` L56-L109 | item 4, read only |

**Out of scope**

The polynomial arrays `a`/`b`/`c` that rapidsnark derives from the witness inside `Prover::prove` (`groth16.cpp` L84-L86, `new[]`/`delete[]` per call, not scrubbed): they live in the vendored rapidsnark, not in either patched repository, and #152 lists them for completeness only. `serde_json::to_string`'s internal growth reallocations before the final `String` (§4.3). The secrets before they reach the witness (#123) and their `Debug` derives (#129, patched in report PR #271). Third-party crates assumed correct: `zeroize` 1.9 (its `Zeroize for [u8]`/`Vec<u8>`/`String` use volatile writes and a compiler fence), `libc`, nlohmann `json`.

**Assumptions**

The copy inventory of #152 LB-003 is taken as read and re-verified only where the patch touches it. The C++ changes are reviewed against the circom 2.2.2 runtime headers the node's cached checkout carries (`rapidsnark/depends/circom_runtime/c/calcwit.hpp`), but not compiled: producing the generated `calcwit.cpp`/`circom.hpp` requires `circom`, which is not installed here (§4.5).

## 3. Method

- Working through items 1–4 of `#224` (parent `#16`) against a detached worktree of the node at the target commit, a clone of the circuits repository at `v0.5.7` (the tag `Cargo.toml` L172-L176 pins) and a clone of rust-rapidsnark at the pinned rev.
- Spec conformance: `common-cryptographic-components.md` §ZkSignature names the secret key as the private witness input that "must remain confidential"; §Groth16 has no memory-hygiene requirement. The patch does not change any protocol behaviour.
- Automated tooling: circuits — `cargo test -p logos-blockchain-circuits-types` (3 new tests), `cargo clippy --workspace --all-targets` under the circuits workspace lints (nursery, pedantic, restriction; clean), `cargo +nightly fmt`, `cargo test` on the four `-sys` crates (§4.4). Node — the patched circuits crate substituted through `[patch."https://github.com/logos-blockchain/logos-blockchain-circuits.git"]` for verification only, then `cargo test --lib -p logos-blockchain-{pol,poq,zksign,poc}` (each crate's `test_full_flow` generates a real witness through the FFI and proves it) and `cargo check --workspace --all-targets`. Results in §4.4.
- Dynamic testing: the full-flow tests above are the dynamic check; no memory-inspection harness was written (§4.5).

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| #152 LB-003 | Witness secrets exist in eight heap copies across the FFI, none zeroized | Data Exposure | Low | High | **Patched** for copies 1, 2, 4, 5, 6, 7, 8; copy 3 open (§4.3) |
| #152 S-001 | Prover reuse | Performance | — | — | Sized, not implemented (§4.3) |

### 4.1 The circuits patch (items 1 and 3)

**Rust, `logos-blockchain-circuits-types`** (Appendix B, first four files):

- `WitnessInput` stores the inputs as `Zeroizing<Vec<u8>>` with the trailing NUL appended by us, instead of a `CString`. `new` takes `impl Into<Zeroizing<String>>`: a plain `String` converts (`zeroize` has `impl<Z: Zeroize> From<Z> for Zeroizing<Z>`), so the four `-sys` tests and any other caller compile unchanged, and the *caller's* buffer is now scrubbed on drop too instead of being left behind after the copy. The copy is made into a vector allocated at exactly `len + 1`, so no intermediate reallocation holds an unscrubbed copy — which `CString::new(String)` does not guarantee (it may reallocate for the NUL, #152 copy 2). An interior NUL is rejected with the same `Error::InvalidInput` message as before.
- `Witness` is `Zeroizing<Vec<u8>>` instead of `bytes::Bytes` (which has no mutable access on drop and so could never be scrubbed) and implements `AsRef<[u8]>`. `From<ffi::Bytes>` copies the C buffer out, then **zeroizes the C buffer** through `<[u8] as Zeroize>::zeroize` (volatile writes plus a fence) before `free_bytes` hands it to `libc::free` — copy 6, fixed from Rust without touching the C library, as #152 proposed. `From<bytes::Bytes>` is gone, `From<Vec<u8>>` replaces it; the `bytes` dependency is dropped and `zeroize` (`alloc` feature, no default features) added.
- `CircuitWitnessInput::new` takes the same `impl Into<Zeroizing<String>>`.
- Tests: the NUL termination and interior-NUL rejection; a `malloc`'d buffer round-trips through `From<ffi::Bytes>`; a null/empty `Bytes` is an empty witness.

**C++, `src/circom_adapter.cpp`** (copies 5 and the per-signal temporary):

- `writeBinWitness` reserves the final `.wtns` size up front (`4 + 4 + 4 + (4 + 8 + 4 + n8 + 4) + (4 + 8) + n8·Nwtns`), because a growing `std::vector` reallocates and each reallocation leaves an unscrubbed partial copy of the witness in freed memory. After the `memcpy` into the `malloc` buffer, `buf` is overwritten through a volatile pointer (`scrubBytes`) before its storage goes back to the allocator, and so is the `FrElement v` that held every signal in turn. The `malloc == nullptr` path now also scrubs.
- `scrubAndDeleteCalcWit` replaces the twelve `delete ctx` sites in the four `ffi.cpp` (three per circuit: the `loadJson` catch, the "not all inputs set" return, and the success path). It is a plain `delete` today; the scrub itself lives in the destructor, because `signalValues` is a private member of `Circom_CalcWit` (`calcwit.hpp` L28, before `public:` at L40) and cannot be reached from the adapter.

**Generated runtime, `fix_calcwit_leak.sh`** (copy 4): the CI build already rewrites circom's empty `~Circom_CalcWit()` with a body that frees `signalValues`, `inputSignalAssigned` and `componentMemory` (the script's own comment explains why). The patch adds, before `delete[] signalValues`, a volatile byte loop over `sizeof(FrElement) * circuit->NSignals` — `circuit` is a public member (`calcwit.hpp` L42) and `NSignals` is what the constructor sizes the array with (`calcwit.cpp` L25). Every signal of the circuit, private inputs included, is therefore zero when the array is freed. `inputSignalAssigned` (booleans) and `componentMemory` (per-component bookkeeping) carry no secret and are left as they were.

### 4.2 The node patch (item 2)

Appendix C. In each of the four `TryFrom<..WitnessInputs> for lbc_*_sys::*WitnessInput` impls the `serde_json::to_string` result is wrapped in `Zeroizing::new(..)` before it is handed to `*WitnessInput::new` (copy 1; the wrap is what makes the intent visible at the call site even though the new `Into` bound would accept the bare `String`). `zeroize = { features = ["alloc"], workspace = true }` is added to the four proof crates. The `Witness` API change needs no node edit: every consumer passes `witness.as_ref()` to `Prover::prove(&[u8], &[u8])` (`pol/src/lib.rs` L77-L80 and the three twins), which already wanted a slice. What the node patch cannot contain yet is the `lbc-*` tag bump in `Cargo.toml` L172-L176 and `flake.nix` L16: it must point at whichever circuits release ships Appendix B, and that tag does not exist until the circuits patch is merged. The verification here used a path override instead (§3), which is not part of the delivered diff.

### 4.3 Item 4 and what the patch does not close

**Copy 3, the JSON parse trees.** `loadJson` (`circom_adapter.cpp` L105-L146) builds two nlohmann `json` values, `jin` from `json::parse` and `j` from `qualify_input`, whose leaves are `std::string`s holding the decimal scalars; the container frees them at scope exit without scrubbing, and there is no hook to scrub a `json` tree's internal strings in place. The durable fix is the one the issue names: pass the inputs as binary field elements (32 bytes each, in signal order) and drop the JSON path; then copies 1, 2 and 3 disappear rather than being scrubbed, and the Rust side sends a `Zeroizing<Vec<u8>>` it already owns. That is a circuits API change larger than this issue and is left as the follow-up.

**Rust-side residual.** `serde_json::to_string` grows its output `Vec` while serialising; each growth reallocates and leaves the earlier partial JSON behind. The final `String` is now scrubbed; the partials are not. Sizing the output with `Vec::with_capacity` from a first pass, or writing the binary inputs above, removes them.

**Item 4, prover reuse.** The C API the issue asks to bind already exists in the vendored rapidsnark: `groth16_prover_create`, `groth16_prover_create_zkey_file`, `groth16_prover_prove` and `groth16_prover_destroy` are declared in circuits `rapidsnark/src/prover.h` L56-L99 next to the one-shot `groth16_prover` (L109). `rust-rapidsnark` (`crates/src/lib.rs` L40-L75) binds only the one-shot entry points, and the node's `zk/circuits/prover` calls `groth16_prover_zkey_buffer_wrapper` per proof. #152 S-001 sized what a long-lived prover saves at these circuit sizes — one section-table scan and one `2·domainSize` root table per call, on the order of a millisecond against tens of milliseconds of MSM and FFT — and the issue makes the change conditional on #97's measurements. Not implemented; the shape, if #97 says it is worth it, is: three `extern "C"` declarations plus a `Groth16Prover(*mut c_void)` newtype with `unsafe impl Send + Sync` (justified by `Prover::prove` having no mutable shared state, #152) and `Drop` calling `groth16_prover_destroy`, held in a `LazyLock` next to each `PROVING_KEY`; and a note that concurrent proves still share the process-wide `ThreadPool`, so reuse does not make them cheaper.

### 4.4 Verification

| Check | Result |
|---|---|
| circuits `cargo test -p logos-blockchain-circuits-types` | 3 passed (the 3 new tests), 0 failed |
| circuits `cargo clippy --workspace --all-targets` (nursery + pedantic + restriction) | clean, 0 warnings |
| circuits `cargo +nightly fmt` | clean |
| circuits `cargo test` on `pol-sys`, `poq-sys`, `poc-sys`, `signature-sys` | the three tests per crate that go through `WitnessInput::new` → `generate_witness` (the patched FFI path: invalid JSON, missing inputs, constraint violation) pass in all four. The fourth, `test_generate_witness`, fails in all four with `Witness generation [circom main()] failed with code: ...` from `generate_witness_from_files` — the file-based `circom_main` path, which the patch does not touch — and fails identically on the unmodified `v0.5.7` checkout on this machine, so it is an environment problem, not a regression |
| node `cargo test --lib -p logos-blockchain-{pol,poq,zksign,poc}` against the patched circuits crate | 4 + 1 + 3 + 1 passed, 0 failed; each `test_full_flow` generates a witness through the new `WitnessInput`/`Witness` and proves it |
| node `cargo check --workspace --all-targets` against the patched circuits crate | clean, 0 errors, 0 warnings (exit 0) |
| C++ | not compiled (§4.5) |

### 4.5 Not done, and why

- The C++ changes are unverified by a compiler. The witness-generator sources (`calcwit.cpp`, `circom.hpp`, `fr.hpp`, `<circuit>.cpp`) are generated by `circom --c` in CI (`justfile` L33-L48 for one circuit) and shipped to the Rust crates as prebuilt libraries; `circom` is not installed here and the cached checkout carries only the runtime's `calcwit.hpp`. What was checked by hand: every identifier the patch uses exists in those headers (`u32`, `u64`, `FrElement`, `Fr_N64`, `get_size_of_witness`, `Circom_CalcWit::circuit` public, `Circom_Circuit::NSignals`), the `perl` substitution in the fix script still matches the destructor it already matched, and the sed-free edits to the four `ffi.cpp` are mechanical. A CI run of the circuits repository is the verification.
- No process-memory inspection was done to demonstrate the scrub (e.g. dumping the heap after a proof and grepping for the decimal secret). The `-sys` and node tests show the new code path produces the same witnesses and proofs; they do not show the absence of the bytes. A test that proves with a recognisable secret and scans `/proc/self/mem` (Linux) would be the assertion; it belongs in the circuits repository's test crate.
- Item 4 (§4.3), item 5 (out of scope for a report).

## 5. Suggestions (non-security)

### S-001 · Binary witness inputs

Replace the decimal-JSON input contract of the four `*_generate_witness` C entry points with fixed-size binary field elements in main-input order. It removes copies 1–3 outright, deletes `loadJson`'s two `json` trees and `json2FrElements`, and makes the Rust side a `Zeroizing<Vec<u8>>` from the start. The `.dat` file already fixes the input order (`InputHashMap`), so the encoding is derivable from artefacts the crates ship.

### S-002 · A memory-scan test in the circuits test crate

After `generate_witness` with a secret of a recognisable pattern, scan the process heap for its decimal and 32-byte forms; assert zero hits. Linux-only (`/proc/self/mem`), gated behind a feature so CI can run it on one platform.

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

## Appendix B — The circuits patch

Proposed patch as `git diff` against `logos-blockchain-circuits` @ `ebf7ddf5b5b625ef4ea7c803171c670beebd05ef` (`v0.5.7`). The `rust/Cargo.lock` hunk swaps the `bytes` edge of the types crate for `zeroize`.

```diff
diff --git a/.github/resources/witness-generator/fix_calcwit_leak.sh b/.github/resources/witness-generator/fix_calcwit_leak.sh
index 7f65abe..a5c7fbc 100755
--- a/.github/resources/witness-generator/fix_calcwit_leak.sh
+++ b/.github/resources/witness-generator/fix_calcwit_leak.sh
@@ -10,7 +10,8 @@
 # megabytes), plus `componentMemory` (and its per-component sub-buffers) and
 # `inputSignalAssigned`, on EVERY call.
 #
-# This rewrites the destructor to free those allocations. The per-component
+# This rewrites the destructor to scrub the signal array and free those
+# allocations. The per-component
 # frees mirror circom's own `release_memory_component` (guarded `if(ptr)
 # delete[]`), so they are safe even if the generated run already released some
 # components.
@@ -39,6 +40,15 @@ perl -0777 -i -pe '
   // (subcomponents/outputIsSet/mutexes/...) are already released during the
   // witness computation by circoms own release_memory_component, so they must
   // NOT be freed here -- doing so double-frees.
+  // logos: scrub the signal array before freeing it. It holds every signal of
+  // the circuit, the private inputs (secret keys, secret vouchers) included,
+  // and delete[] leaves the bytes in place for the allocator to hand out again.
+  {
+    volatile unsigned char* p = reinterpret_cast<volatile unsigned char*>(signalValues);
+    for (size_t i = 0; i < sizeof(FrElement) * (size_t)circuit->NSignals; i++) {
+      p[i] = 0;
+    }
+  }
   delete[] signalValues;
   delete[] inputSignalAssigned;
   delete[] componentMemory;
diff --git a/rust/Cargo.lock b/rust/Cargo.lock
index 686dab6..e4b4626 100644
--- a/rust/Cargo.lock
+++ b/rust/Cargo.lock
@@ -255,8 +255,8 @@ dependencies = [
 name = "logos-blockchain-circuits-types"
 version = "0.5.7"
 dependencies = [
- "bytes",
  "libc",
+ "zeroize",
 ]
 
 [[package]]
diff --git a/rust/Cargo.toml b/rust/Cargo.toml
index 6ad2bff..fee969f 100644
--- a/rust/Cargo.toml
+++ b/rust/Cargo.toml
@@ -41,7 +41,7 @@ libc = "^0.2"
 serde_json = "^1"
 tar = "^0.4"
 ureq = "^3.3"
-bytes = "^1.11"
+zeroize = { version = "^1.8", default-features = false, features = ["alloc"] }
 
 # Sourced from https://github.com/logos-co/nomos/blob/main/Cargo.toml
 [workspace.lints.clippy]
diff --git a/rust/logos-blockchain-circuits-types/Cargo.toml b/rust/logos-blockchain-circuits-types/Cargo.toml
index 3a85314..9d59047 100644
--- a/rust/logos-blockchain-circuits-types/Cargo.toml
+++ b/rust/logos-blockchain-circuits-types/Cargo.toml
@@ -13,5 +13,5 @@ version.workspace = true
 workspace = true
 
 [dependencies]
-bytes = { workspace = true }
 libc = { workspace = true }
+zeroize = { workspace = true }
diff --git a/rust/logos-blockchain-circuits-types/src/native/circuit_witness_input.rs b/rust/logos-blockchain-circuits-types/src/native/circuit_witness_input.rs
index 48db1cc..c79b302 100644
--- a/rust/logos-blockchain-circuits-types/src/native/circuit_witness_input.rs
+++ b/rust/logos-blockchain-circuits-types/src/native/circuit_witness_input.rs
@@ -1,5 +1,7 @@
 use std::{marker::PhantomData, ops::Deref};
 
+use zeroize::Zeroizing;
+
 use crate::native::{Error, WitnessInput};
 
 pub trait CircuitDat<'dat> {
@@ -12,7 +14,7 @@ pub struct CircuitWitnessInput<'dat, Dat> {
 }
 
 impl<'dat, Dat: CircuitDat<'dat>> CircuitWitnessInput<'dat, Dat> {
-    pub fn new(inputs_json: String) -> Result<Self, Error> {
+    pub fn new(inputs_json: impl Into<Zeroizing<String>>) -> Result<Self, Error> {
         let inner = WitnessInput::new(Dat::DAT, inputs_json)?;
         Ok(Self {
             inner,
diff --git a/rust/logos-blockchain-circuits-types/src/native/witness.rs b/rust/logos-blockchain-circuits-types/src/native/witness.rs
index 773a429..b8b0c48 100644
--- a/rust/logos-blockchain-circuits-types/src/native/witness.rs
+++ b/rust/logos-blockchain-circuits-types/src/native/witness.rs
@@ -1,19 +1,25 @@
+use zeroize::{Zeroize as _, Zeroizing};
+
 use crate::ffi;
 
-/// Byte buffer
+/// Witness bytes (the `.wtns` contents).
+///
+/// The witness holds every signal of the circuit, private inputs included, so
+/// the buffer is scrubbed on drop; the C buffer it was copied from is scrubbed
+/// before it is returned to the allocator.
 ///
 /// When constructing from [`From<ffi::Bytes>`], it takes ownership of the
 /// underlying value and frees it.
-pub struct Witness(bytes::Bytes);
+pub struct Witness(Zeroizing<Vec<u8>>);
 
-impl From<bytes::Bytes> for Witness {
-    fn from(bytes: bytes::Bytes) -> Self {
-        Self(bytes)
+impl From<Vec<u8>> for Witness {
+    fn from(bytes: Vec<u8>) -> Self {
+        Self(Zeroizing::new(bytes))
     }
 }
 
-impl AsRef<bytes::Bytes> for Witness {
-    fn as_ref(&self) -> &bytes::Bytes {
+impl AsRef<[u8]> for Witness {
+    fn as_ref(&self) -> &[u8] {
         &self.0
     }
 }
@@ -25,12 +31,42 @@ impl From<ffi::Bytes> for Witness {
         } else {
             // SAFETY: `ffi_value.data` is non-null and `ffi_value.size > 0` (checked
             // above), pointing to a valid C-allocated buffer of at least `size`
-            // bytes.
-            unsafe { std::slice::from_raw_parts(ffi_value.data, ffi_value.size).to_vec() }
+            // bytes that nothing else references until `free_bytes` below.
+            let c_buffer =
+                unsafe { std::slice::from_raw_parts_mut(ffi_value.data, ffi_value.size) };
+            let vec = c_buffer.to_vec();
+            // Scrub the C copy before `free` hands its pages back to the allocator.
+            c_buffer.zeroize();
+            vec
         };
         // SAFETY: `ffi_value` is a local variable, so the raw pointer is valid for this
         // call.
         unsafe { ffi::free_bytes(&raw mut ffi_value) };
-        Self::from(bytes::Bytes::from(vec))
+        Self::from(vec)
+    }
+}
+
+#[cfg(test)]
+mod tests {
+    use super::*;
+
+    #[test]
+    fn a_c_buffer_is_copied_out_and_freed() {
+        let size = 16;
+        // SAFETY: a fresh allocation of `size` bytes, owned by the `Bytes` below
+        // and freed by `Witness::from`.
+        let data = unsafe { libc::malloc(size) }.cast::<u8>();
+        assert!(!data.is_null());
+        // SAFETY: `data` points to `size` writable bytes.
+        unsafe { std::ptr::write_bytes(data, 0xAB, size) };
+
+        let witness = Witness::from(ffi::Bytes { data, size });
+        assert_eq!(witness.as_ref(), &[0xAB; 16]);
+    }
+
+    #[test]
+    fn a_null_or_empty_c_buffer_is_an_empty_witness() {
+        let witness = Witness::from(ffi::Bytes::null());
+        assert!(witness.as_ref().is_empty());
     }
 }
diff --git a/rust/logos-blockchain-circuits-types/src/native/witness_input.rs b/rust/logos-blockchain-circuits-types/src/native/witness_input.rs
index d5e30cc..61bde36 100644
--- a/rust/logos-blockchain-circuits-types/src/native/witness_input.rs
+++ b/rust/logos-blockchain-circuits-types/src/native/witness_input.rs
@@ -1,22 +1,28 @@
-use std::ffi::CString;
+use std::ffi::c_char;
+
+use zeroize::Zeroizing;
 
 use crate::{ffi, native::Error};
 
 /// Input for witness generators
+///
+/// The JSON carries the circuit's private inputs (a note secret key, a voucher
+/// secret, signing keys), so the buffer is scrubbed when this value is
+/// dropped.
 pub struct WitnessInput<'dat> {
     /// The circuit's dat file contents.
     dat: &'dat [u8],
-    /// The JSON string containing the circuit inputs.
-    inputs_json: CString,
+    /// The NUL-terminated JSON string containing the circuit inputs.
+    inputs_json: Zeroizing<Vec<u8>>,
 }
 
 impl<'dat> WitnessInput<'dat> {
-    pub fn new(dat: &'dat [u8], inputs_json: String) -> Result<Self, Error> {
-        let inputs_json = CString::new(inputs_json).map_err(|error| {
-            Error::InvalidInput(Some(format!(
-                "The parameter inputs_json could not be converted to C string: {error}"
-            )))
-        })?;
+    /// Takes the inputs as a `Zeroizing<String>`; a plain `String` converts
+    /// into one, so the caller's buffer is scrubbed on drop as well instead of
+    /// being left behind after the copy.
+    pub fn new(dat: &'dat [u8], inputs_json: impl Into<Zeroizing<String>>) -> Result<Self, Error> {
+        let inputs_json = inputs_json.into();
+        let inputs_json = nul_terminated(inputs_json.as_bytes())?;
         Ok(Self { dat, inputs_json })
     }
 
@@ -27,6 +33,22 @@ impl<'dat> WitnessInput<'dat> {
     }
 }
 
+/// Copies `bytes` into a buffer with exactly one trailing NUL byte.
+///
+/// The buffer is allocated at its final size, so no intermediate allocation
+/// ever holds an unscrubbed copy of the inputs.
+fn nul_terminated(bytes: &[u8]) -> Result<Zeroizing<Vec<u8>>, Error> {
+    if let Some(position) = bytes.iter().position(|byte| *byte == 0) {
+        return Err(Error::InvalidInput(Some(format!(
+            "The parameter inputs_json could not be converted to C string: nul byte found in provided data at position: {position}"
+        ))));
+    }
+    let mut buffer = Zeroizing::new(Vec::with_capacity(bytes.len().saturating_add(1)));
+    buffer.extend_from_slice(bytes);
+    buffer.push(0);
+    Ok(buffer)
+}
+
 /// Temporary FFI view of a [`WitnessInput`], which makes [`ffi::WitnessInput`]
 /// lifetime-aware.
 pub struct WitnessInputFfiGuard<'dat> {
@@ -40,7 +62,7 @@ impl<'dat> WitnessInputFfiGuard<'dat> {
             data: inner.dat.as_ptr(),
             size: inner.dat.len(),
         };
-        let inputs_json = inner.inputs_json.as_ptr();
+        let inputs_json = inner.inputs_json.as_ptr().cast::<c_char>();
         let ffi = ffi::WitnessInput { dat, inputs_json };
         Self {
             ffi,
@@ -54,3 +76,23 @@ impl AsRef<ffi::WitnessInput> for WitnessInputFfiGuard<'_> {
         &self.ffi
     }
 }
+
+#[cfg(test)]
+mod tests {
+    use super::*;
+
+    #[test]
+    fn inputs_are_nul_terminated_and_rejected_when_they_contain_nul() {
+        let Ok(input) = WitnessInput::new(&[], "{\"a\":1}".to_owned()) else {
+            panic!("a JSON string without NUL is a valid input");
+        };
+        assert_eq!(input.inputs_json.as_slice(), b"{\"a\":1}\0");
+
+        let Err(error) = WitnessInput::new(&[], "{\0}".to_owned()) else {
+            panic!("an interior NUL must be rejected");
+        };
+        assert!(
+            matches!(error, Error::InvalidInput(Some(message)) if message.contains("nul byte"))
+        );
+    }
+}
diff --git a/src/circom_adapter.cpp b/src/circom_adapter.cpp
index 0560753..31fac3f 100644
--- a/src/circom_adapter.cpp
+++ b/src/circom_adapter.cpp
@@ -6,6 +6,22 @@
 #include <stdexcept>
 #include <mutex>
 
+// Overwrites `size` bytes at `data` with zeros through a volatile pointer, so
+// the store is not elided as a dead write to memory about to be freed. Used on
+// every buffer that held witness signals (the private inputs among them).
+static void scrubBytes(void* data, size_t size) {
+    volatile unsigned char* p = static_cast<volatile unsigned char*>(data);
+    while (size--) {
+        *p++ = 0;
+    }
+}
+
+void scrubAndDeleteCalcWit(Circom_CalcWit* ctx) {
+    // The signal array is private to Circom_CalcWit; the generated destructor
+    // is patched by fix_calcwit_leak.sh to scrub it before freeing.
+    delete ctx;
+}
+
 Circom_Circuit* getCachedCircuit(const ConstBytes& circuit_bytes) {
     // The circuit is immutable, compiled-in data and has no destructor that
     // frees its internal buffers, so load it once and keep it for the process
@@ -147,6 +163,13 @@ void loadJson(Circom_CalcWit *ctx, const char* inputs_json) {
 
 void writeBinWitness(Circom_CalcWit *ctx, Bytes* output_witness) {
     std::vector<uint8_t> buf;
+    // Reserve the final size up front: a growing vector reallocates and leaves
+    // unscrubbed partial copies of the witness behind.
+    {
+        const u32 n8_reserve = Fr_N64*8;
+        const u64 nwtns_reserve = (u64)get_size_of_witness();
+        buf.reserve(4 + 4 + 4 + (4 + 8 + 4 + n8_reserve + 4) + (4 + 8) + (u64)n8_reserve * nwtns_reserve);
+    }
 
     auto write = [&](const void* data, size_t size) {
         const uint8_t* p = (const uint8_t*)data;
@@ -194,9 +217,16 @@ void writeBinWitness(Circom_CalcWit *ctx, Bytes* output_witness) {
         write(v.longVal, Fr_N64*8);
     }
 
+    // `v` held every signal in turn, the private inputs included.
+    scrubBytes(&v, sizeof(v));
+
     size_t size = buf.size();
     output_witness->data = static_cast<uint8_t*>(malloc(size));
-    if (output_witness->data == nullptr) return;
-    output_witness->size = size;
-    memcpy(output_witness->data, buf.data(), size);
+    if (output_witness->data != nullptr) {
+        output_witness->size = size;
+        memcpy(output_witness->data, buf.data(), size);
+    }
+    // The caller's copy is now the only one; scrub ours before the vector's
+    // storage goes back to the allocator.
+    scrubBytes(buf.data(), buf.size());
 }
diff --git a/src/circom_adapter.hpp b/src/circom_adapter.hpp
index bd4798f..f4adb58 100644
--- a/src/circom_adapter.hpp
+++ b/src/circom_adapter.hpp
@@ -20,4 +20,8 @@ Circom_Circuit* getCachedCircuit(const ConstBytes& circuit);
 void loadJson(Circom_CalcWit *ctx, const char* inputs_json);
 void writeBinWitness(Circom_CalcWit *ctx, Bytes* output_witness);
 
+// Deletes a Circom_CalcWit; its signal array (which holds the private inputs)
+// is scrubbed by the destructor patched in by fix_calcwit_leak.sh.
+void scrubAndDeleteCalcWit(Circom_CalcWit* ctx);
+
 #endif
diff --git a/src/poc/ffi.cpp b/src/poc/ffi.cpp
index 5ec2244..1128015 100644
--- a/src/poc/ffi.cpp
+++ b/src/poc/ffi.cpp
@@ -95,17 +95,17 @@ static Status generate_witness_impl(const WitnessInput* input, Bytes* output) {
     try {
         loadJson(ctx, input->inputs_json);
     } catch (...) {
-        delete ctx;
+        scrubAndDeleteCalcWit(ctx);
         throw;
     }
     if (ctx->getRemaingInputsToBeSet()!=0) {
         const std::string message = "Not all inputs have been set. Only " + std::to_string(get_main_input_signal_no()-ctx->getRemaingInputsToBeSet()) + " out of " + std::to_string(get_main_input_signal_no()) + ".";
-        delete ctx;
+        scrubAndDeleteCalcWit(ctx);
         return status_new(StatusCode_InvalidInput, message.c_str());
     }
 
     writeBinWitness(ctx, output);
-    delete ctx;
+    scrubAndDeleteCalcWit(ctx);
 
     return status_ok();
 }
diff --git a/src/pol/ffi.cpp b/src/pol/ffi.cpp
index 176f3f9..e44f1d1 100644
--- a/src/pol/ffi.cpp
+++ b/src/pol/ffi.cpp
@@ -95,17 +95,17 @@ static Status generate_witness_impl(const WitnessInput* input, Bytes* output) {
     try {
         loadJson(ctx, input->inputs_json);
     } catch (...) {
-        delete ctx;
+        scrubAndDeleteCalcWit(ctx);
         throw;
     }
     if (ctx->getRemaingInputsToBeSet()!=0) {
         const std::string message = "Not all inputs have been set. Only " + std::to_string(get_main_input_signal_no()-ctx->getRemaingInputsToBeSet()) + " out of " + std::to_string(get_main_input_signal_no()) + ".";
-        delete ctx;
+        scrubAndDeleteCalcWit(ctx);
         return status_new(StatusCode_InvalidInput, message.c_str());
     }
 
     writeBinWitness(ctx, output);
-    delete ctx;
+    scrubAndDeleteCalcWit(ctx);
 
     return status_ok();
 }
diff --git a/src/poq/ffi.cpp b/src/poq/ffi.cpp
index 44c69ab..91022d5 100644
--- a/src/poq/ffi.cpp
+++ b/src/poq/ffi.cpp
@@ -95,17 +95,17 @@ static Status generate_witness_impl(const WitnessInput* input, Bytes* output) {
     try {
         loadJson(ctx, input->inputs_json);
     } catch (...) {
-        delete ctx;
+        scrubAndDeleteCalcWit(ctx);
         throw;
     }
     if (ctx->getRemaingInputsToBeSet()!=0) {
         const std::string message = "Not all inputs have been set. Only " + std::to_string(get_main_input_signal_no()-ctx->getRemaingInputsToBeSet()) + " out of " + std::to_string(get_main_input_signal_no()) + ".";
-        delete ctx;
+        scrubAndDeleteCalcWit(ctx);
         return status_new(StatusCode_InvalidInput, message.c_str());
     }
 
     writeBinWitness(ctx, output);
-    delete ctx;
+    scrubAndDeleteCalcWit(ctx);
 
     return status_ok();
 }
diff --git a/src/signature/ffi.cpp b/src/signature/ffi.cpp
index 43a792b..9a6abd4 100644
--- a/src/signature/ffi.cpp
+++ b/src/signature/ffi.cpp
@@ -95,17 +95,17 @@ static Status generate_witness_impl(const WitnessInput* input, Bytes* output) {
     try {
         loadJson(ctx, input->inputs_json);
     } catch (...) {
-        delete ctx;
+        scrubAndDeleteCalcWit(ctx);
         throw;
     }
     if (ctx->getRemaingInputsToBeSet()!=0) {
         const std::string message = "Not all inputs have been set. Only " + std::to_string(get_main_input_signal_no()-ctx->getRemaingInputsToBeSet()) + " out of " + std::to_string(get_main_input_signal_no()) + ".";
-        delete ctx;
+        scrubAndDeleteCalcWit(ctx);
         return status_new(StatusCode_InvalidInput, message.c_str());
     }
 
     writeBinWitness(ctx, output);
-    delete ctx;
+    scrubAndDeleteCalcWit(ctx);
 
     return status_ok();
 }
```

## Appendix C — The node patch

Proposed patch as `git diff` against `logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b`. To be applied together with a bump of the five `lbc-*` tags in `Cargo.toml` L172-L176 and the circuits flake input in `flake.nix` L16 to the release that carries Appendix B; the `Cargo.lock` hunk adds the `zeroize` edges for the four proof crates.

```diff
diff --git a/Cargo.lock b/Cargo.lock
index 0ecedfd37..7846f7f35 100644
--- a/Cargo.lock
+++ b/Cargo.lock
@@ -4932,6 +4932,7 @@ dependencies = [
  "serde",
  "serde_json",
  "tracing",
+ "zeroize",
 ]
 
 [[package]]
@@ -4952,6 +4953,7 @@ dependencies = [
  "serde",
  "serde_json",
  "tracing",
+ "zeroize",
 ]
 
 [[package]]
@@ -4971,6 +4973,7 @@ dependencies = [
  "serde",
  "serde_json",
  "tracing",
+ "zeroize",
 ]
 
 [[package]]
@@ -5401,6 +5404,7 @@ dependencies = [
  "serde_json",
  "thiserror 2.0.18",
  "tracing",
+ "zeroize",
 ]
 
 [[package]]
diff --git a/Cargo.toml b/Cargo.toml
index 3bd8db8bf..ebfcd9dfb 100644
--- a/Cargo.toml
+++ b/Cargo.toml
@@ -456,3 +456,4 @@ trivial_casts                          = { level = "allow" }
 unreachable_pub                        = { level = "allow" }
 unsafe_code                            = { level = "allow" }
 variant_size_differences               = { level = "allow" }
+
diff --git a/zk/proofs/poc/Cargo.toml b/zk/proofs/poc/Cargo.toml
index 41d5393c0..94de8767e 100644
--- a/zk/proofs/poc/Cargo.toml
+++ b/zk/proofs/poc/Cargo.toml
@@ -10,6 +10,7 @@ repository  = { workspace = true }
 version     = { workspace = true }
 
 [dependencies]
+zeroize = { features = ["alloc"], workspace = true }
 lb-circuits-prover = { workspace = true }
 lb-groth16         = { workspace = true }
 lb-log-targets     = { workspace = true }
diff --git a/zk/proofs/poc/src/inputs.rs b/zk/proofs/poc/src/inputs.rs
index fb42ecfb5..4ef4fd86a 100644
--- a/zk/proofs/poc/src/inputs.rs
+++ b/zk/proofs/poc/src/inputs.rs
@@ -1,5 +1,6 @@
 use lb_groth16::{Fr, Groth16Input, Groth16InputDeser};
 use serde::{Deserialize, Serialize};
+use zeroize::Zeroizing;
 
 use crate::{
     PoCChainInputsData, PoCWalletInputsData,
@@ -32,7 +33,7 @@ impl TryFrom<PoCWitnessInputs> for lbc_poc_sys::PocWitnessInput<'_> {
 
     fn try_from(value: PoCWitnessInputs) -> Result<Self, Self::Error> {
         let inputs_json: PoCInputsJson = value.into();
-        let inputs_str: String = serde_json::to_string(&inputs_json)?;
+        let inputs_str = Zeroizing::new(serde_json::to_string(&inputs_json)?);
         let witness_input = lbc_poc_sys::PocWitnessInput::new(inputs_str)?;
         Ok(witness_input)
     }
diff --git a/zk/proofs/pol/Cargo.toml b/zk/proofs/pol/Cargo.toml
index 6dc4f19a3..05f286509 100644
--- a/zk/proofs/pol/Cargo.toml
+++ b/zk/proofs/pol/Cargo.toml
@@ -10,6 +10,7 @@ repository  = { workspace = true }
 version     = { workspace = true }
 
 [dependencies]
+zeroize = { features = ["alloc"], workspace = true }
 astro-float        = { workspace = true }
 lb-circuits-prover = { workspace = true }
 lb-groth16         = { workspace = true }
diff --git a/zk/proofs/pol/src/inputs.rs b/zk/proofs/pol/src/inputs.rs
index 6db1e15d5..15d55f9e2 100644
--- a/zk/proofs/pol/src/inputs.rs
+++ b/zk/proofs/pol/src/inputs.rs
@@ -1,5 +1,6 @@
 use lb_groth16::{Fr, Groth16Input, Groth16InputDeser};
 use serde::{Deserialize, Serialize};
+use zeroize::Zeroizing;
 
 use crate::{
     PolChainInputsData, PolWalletInputsData,
@@ -20,7 +21,7 @@ impl TryFrom<PolWitnessInputs> for lbc_pol_sys::PolWitnessInput<'_> {
 
     fn try_from(value: PolWitnessInputs) -> Result<Self, Self::Error> {
         let inputs_json: PolInputsJson = value.into();
-        let inputs_str: String = serde_json::to_string(&inputs_json)?;
+        let inputs_str = Zeroizing::new(serde_json::to_string(&inputs_json)?);
         let witness_input = lbc_pol_sys::PolWitnessInput::new(inputs_str)?;
         Ok(witness_input)
     }
diff --git a/zk/proofs/poq/Cargo.toml b/zk/proofs/poq/Cargo.toml
index f5a9f69c0..c9d88e3d9 100644
--- a/zk/proofs/poq/Cargo.toml
+++ b/zk/proofs/poq/Cargo.toml
@@ -10,6 +10,7 @@ repository  = { workspace = true }
 version     = { workspace = true }
 
 [dependencies]
+zeroize = { features = ["alloc"], workspace = true }
 lb-circuits-prover = { workspace = true }
 lb-groth16         = { workspace = true }
 lb-log-targets     = { workspace = true }
diff --git a/zk/proofs/poq/src/inputs.rs b/zk/proofs/poq/src/inputs.rs
index 17e487632..3898e465e 100644
--- a/zk/proofs/poq/src/inputs.rs
+++ b/zk/proofs/poq/src/inputs.rs
@@ -1,5 +1,6 @@
 use lb_groth16::{AdditiveGroup as _, Fr, Groth16Input, Groth16InputDeser};
 use serde::{Deserialize, Serialize};
+use zeroize::Zeroizing;
 
 use crate::{
     PoQChainInputsData, PoQCommonInputsData, Quota,
@@ -96,7 +97,7 @@ impl TryFrom<PoQWitnessInputs> for lbc_poq_sys::PoqWitnessInput<'_> {
 
     fn try_from(value: PoQWitnessInputs) -> Result<Self, Self::Error> {
         let inputs_json: PoQInputsJson = value.into();
-        let inputs_str: String = serde_json::to_string(&inputs_json)?;
+        let inputs_str = Zeroizing::new(serde_json::to_string(&inputs_json)?);
         let witness_input = lbc_poq_sys::PoqWitnessInput::new(inputs_str)?;
         Ok(witness_input)
     }
diff --git a/zk/proofs/zksign/Cargo.toml b/zk/proofs/zksign/Cargo.toml
index f9ac8843d..1648241bd 100644
--- a/zk/proofs/zksign/Cargo.toml
+++ b/zk/proofs/zksign/Cargo.toml
@@ -10,6 +10,7 @@ repository  = { workspace = true }
 version     = { workspace = true }
 
 [dependencies]
+zeroize = { features = ["alloc"], workspace = true }
 lb-circuits-prover = { workspace = true }
 lb-groth16         = { workspace = true }
 lb-log-targets     = { workspace = true }
diff --git a/zk/proofs/zksign/src/inputs.rs b/zk/proofs/zksign/src/inputs.rs
index 261f4e73f..7cac0a1fc 100644
--- a/zk/proofs/zksign/src/inputs.rs
+++ b/zk/proofs/zksign/src/inputs.rs
@@ -1,5 +1,6 @@
 use lb_groth16::{Fr, Groth16Input, Groth16InputDeser};
 use serde::Serialize;
+use zeroize::Zeroizing;
 
 use crate::private::{ZkSignPrivateKeysData, ZkSignPrivateKeysInputs, ZkSignPrivateKeysInputsJson};
 
@@ -23,7 +24,7 @@ impl TryFrom<ZkSignWitnessInputs> for lbc_signature_sys::SignatureWitnessInput<'
 
     fn try_from(value: ZkSignWitnessInputs) -> Result<Self, Self::Error> {
         let inputs_json: ZkSignWitnessInputsJson = (&value).into();
-        let inputs_str: String = serde_json::to_string(&inputs_json)?;
+        let inputs_str = Zeroizing::new(serde_json::to_string(&inputs_json)?);
         let witness_input = lbc_signature_sys::SignatureWitnessInput::new(inputs_str)?;
         Ok(witness_input)
     }
```
