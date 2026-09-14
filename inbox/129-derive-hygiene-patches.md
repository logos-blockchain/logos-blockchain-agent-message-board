# Audit Report — Derive hygiene, fix pass: redacting `Debug` on the PoL/PoC witness types, `TryFrom<BigUint>` for the ZK keys, a checked key parser, `VoucherSecret` and `X25519PrivateKey`

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/129`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `zk/proofs/pol`, `zk/proofs/poc`, `zk/groth16`, `kms/keys`, `core/src/proofs`, `core/src/mantle/ops/leader_claim.rs`, `nodes/node/binary/src/{config/mod.rs,cli/config/keystore.rs}`, `tools/config`, `tests/testing_framework`; test-only rewrites in `core`, `ledger`, `wallet`, `zone-sdk`, `tools/blockchain-tools`, `tests`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md` (parent #24: no specification covers this area)
Date: 2026-09-12 — author: `agent (Claude)` — status: `fix-review`

---

## 1. Summary

- Overall assessment: items 4, 5 and 6 of #129 are done as one proposed patch against the target commit (42 files, +440/−103), given in full in Appendix B. It closes #128 LB-001, LB-002 and LB-004 and folds #252 S-002 into the same change. No new finding. The workspace type-checks with all targets, clippy is clean on every touched crate under the workspace's nursery+pedantic lints, and 12 new unit tests plus the existing `core`, `ledger`, `pol`, `poc`, `kms/keys` and `groth16` suites pass.
- Findings: 0 critical · 0 high · 0 medium · 0 low · 0 informational (three prior findings moved to *Patched*)
- Key themes: "a secret's `Debug` is now a decision, not a derive, on every type the leader key or voucher secret passes through", "a key given as bytes is the scalar the caller supplied or an error, never a silent reduction", "one random-key sampler instead of two"
- Must-fix before launch: none from this issue; the patch in Appendix B is ready for a maintainer to apply and open as a `logos-blockchain` PR.

## 2. Scope

**In scope** (what the patch touches, at the target commit's line numbers)

| Crate / path | Notes |
|---|---|
| `zk/proofs/pol/src/wallet_inputs.rs` L14-L24, L27-L37; `zk/proofs/pol/src/inputs.rs` L11-L16, L29-L33; `zk/proofs/poc/src/wallet_inputs.rs` L13-L17 | the five witness types that carry `secret_key`/`secret_voucher` |
| `core/src/proofs/leader_proof.rs` L281-L285; `core/src/proofs/leader_claim_proof.rs` L124-L127 | `LeaderPrivate`, `LeaderClaimPrivate` |
| `zk/groth16/src/lib.rs` L59-L68 | `fr_from_bytes`, split into a reusable checked `BigUint → Fr` |
| `kms/keys/src/keys/zk/{private.rs L83-L87, mod.rs L94-L98, public.rs L77-L81}` | the three `From<BigUint>` impls |
| `nodes/node/binary/src/config/mod.rs` L564-L570; `nodes/node/binary/src/cli/config/keystore.rs` L184-L188 | `parse_hex_zk_key`; the `generate-key` sampler |
| `core/src/mantle/ops/leader_claim.rs` L45-L46; `kms/keys/src/keys/ed25519/x25519.rs` L8-L9 | `VoucherSecret`, `X25519PrivateKey` |
| `tools/config/src/{consensus.rs L398,L409,L428, blend.rs L30}`; `tests/testing_framework/src/node/configs/wallet.rs` L141, L154 | seed-derived keys, reduction made explicit |
| 21 test/bench/fixture files (§4.2) | `ZkKey::from(BigUint::from(n))` → `ZkKey::from(Fr::from(n))` |
| `tools/config/Cargo.toml`, `tests/Cargo.toml`, `tests/testing_framework/Cargo.toml`, `core/Cargo.toml`, `Cargo.lock` | `num-bigint` dropped where it became unused; `subtle` added to `core` |

**Out of scope**

Zeroizing the witness copies and moving proving into the KMS operator (#123, the long-term half of #128 LB-001); `UnsecuredZkKey`'s derived `Debug`/`Serialize` and the `unsafe` feature (#128 LB-003, tracked in #122); the PoW winning-ticket keys (#252 LB-001, tracked in #255); the bias of the mod-`p` sampler itself (#128 S-002: `SecretKey::from_rng` still reduces 32 random bytes; this patch makes it the *only* sampler, not a uniform one). Third-party crates assumed correct: `ark-ff`, `subtle` 2.6.1, `x25519-dalek`, `zeroize`.

**Assumptions**

The classification of #128 §4 and the grep-complete inventory of #252 §6 are taken as read: no formatting or serialisation site of the patched types exists outside tests at the target commit, so the patch changes what a future log line *could* print, not what any current one prints.

## 3. Method

- Working through items 4–6 of `#129` (parent `#24`) on a detached worktree at the target commit; items 1–3 are report #252 §6 and were not redone.
- Automated tooling, with versions: `rustc 1.98.1` (the pinned toolchain), `cargo check --workspace --all-targets` (clean: no errors, no warnings), `cargo clippy --all-targets` on the 13 touched crates under the workspace `[workspace.lints]` (nursery, pedantic, cargo; clean, zero warnings), `cargo +nightly-2026-01-18 fmt --all` with the repository's `rustfmt.toml` (CI pins `nightly-2026-07-05`; the two produced no difference on these files), `cargo test --lib` on `logos-blockchain-{pol,poc,groth16,key-management-system-keys,core,ledger}` and the three parser tests in `logos-blockchain-node`. Results in §4.4.
- Dynamic testing: none beyond the unit tests; nothing here changes runtime behaviour except the parser (§4.2) and `generate-key` (§4.2).

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| #128 LB-001 | PoL and PoC witness types derive `Debug` around the leader secret key and the voucher secret | Data Exposure | Low | High | **Patched** (§4.1) |
| #128 LB-002 | CLI ZK secret-key parser uses a lossy `From<BigUint>` that reduces modulo `p` and accepts any length | Data Validation | Low | Medium | **Patched** (§4.2) |
| #128 LB-004 | `VoucherSecret` and `X25519PrivateKey` derive `Serialize`/`Debug`/`Default` unconditionally | Data Exposure | Informational | High | **Patched** (§4.3) |
| #252 S-002 | `generate_zk_key_from_random_bytes` is a second copy of the mod-`p` sampler | — | — | — | **Patched** (§4.2) |

### 4.1 Item 4 — redacting `Debug` on the witness types

Every derived `Debug` on the chain from `LeaderPrivate`/`LeaderClaimPrivate` down to the raw witness structs is replaced by a hand-written impl, following `ProofType` (`blend/proofs/src/quota/inputs/prove/private.rs` L102-L110):

| Type | Before | After |
|---|---|---|
| `PolWalletInputsData`, `PolWalletInputs` (`pol/src/wallet_inputs.rs`) | `derive(Debug)`; `secret_key: Fr` / `Groth16Input` printed in full | `note_value`, `transaction_hash`, `output_number`, then `secret_key: "<redacted>"`, `finish_non_exhaustive()` (the four Merkle path/selector arrays are omitted) |
| `PolWitnessInputsData`, `PolWitnessInputs` (`pol/src/inputs.rs`) | `derive(Debug)` | `debug_struct` over `wallet` and `chain`; the wallet half redacts. `PolWitnessInputs` keeps `Serialize` through `PolInputsJson`, which the prover needs (#128 LB-001, "keep `Serialize` only on the `*Json` structs") |
| `PoCWalletInputsData` (`poc/src/wallet_inputs.rs`) | `derive(Debug)`; `secret_voucher` printed | `secret_voucher: "<redacted>"`, path printed |
| `LeaderPrivate` (`core/src/proofs/leader_proof.rs`), `LeaderClaimPrivate` (`leader_claim_proof.rs`) | `derive(Debug)` | `debug_struct` over the private fields; the inner witness redacts |

`PoCWitnessInputsData` and `PolChainInputsData`/`PoCChainInputsData` keep their derives: the former now redacts transitively, the latter hold public inputs only. `finish_non_exhaustive` is what satisfies clippy's pedantic `missing_fields_in_debug` on the two types that omit the arrays.

Seven tests pin the property, one per type. Each builds the value around `Fr::from(12_648_430)` and asserts that the rendering contains `<redacted>` and does not contain any of the secret's `Debug` rendering, its `Display` rendering, or the `Debug` rendering of `Groth16Input::new(secret)` — the three forms in which the scalar could reappear if a derive came back:

- `pol`: `wallet_inputs::tests::{wallet_inputs_data_debug_redacts_secret_key, wallet_inputs_debug_redacts_secret_key}`, `inputs::tests::{witness_inputs_data_debug_redacts_secret_key, witness_inputs_debug_redacts_secret_key}`
- `poc`: `wallet_inputs::tests::wallet_inputs_data_debug_redacts_secret_voucher`
- `core`: `proofs::leader_proof::redaction_tests::leader_private_debug_redacts_secret_key` (a one-note `UtxoTree` so `LeaderPrivate::new` gets a real path), `proofs::leader_claim_proof::redaction_tests::leader_claim_private_debug_redacts_secret_voucher`

Coordination with #123: the types are unchanged in layout and still `Clone`/`Copy`-carrying, so the zeroize work there applies on top without conflict; the only new items are the `Debug` impls and the tests.

### 4.2 Item 5 — `TryFrom<BigUint>`, the parser, and the 48 call sites

**The impls.** `lb_groth16` gains `fr_try_from_biguint(BigUint) -> Result<Fr, FrFromBytesError>`, the check that `fr_from_bytes` already made (L59-L68), factored out so a `BigUint` that never came from bytes can use it; `fr_from_bytes` now calls it. The three `From<BigUint>` impls become `TryFrom<BigUint>` with `Error = FrFromBytesError`: `SecretKey` (`private.rs`), `ZkKey` delegating to it (`mod.rs`), `PublicKey` (`public.rs`). `From<Fr>` on all three is untouched and is the infallible constructor tests use.

**The parser.** `parse_hex_zk_key` (`config/mod.rs` L564-L570) now rejects any decoded length other than `size_of::<FrBytes>()` (32) with an explicit message, then goes through `fr_from_bytes`, so a value `>= p` is an error, mirroring `parse_hex_public_key` (L555-L562) and the Ed25519 parser beside it (L572-L577). `add-key --zk` uses it as its `value_parser` (`cli/keys.rs` L102-L106), so the CLI inherits the check. Three tests in `config::zk_key_parser_tests`: a 32-byte scalar below `p` round-trips to the same `Fr`; `0xff × 32` (above `p`) is rejected without reduction; lengths 0, 1, 31, 33 and 64 are rejected as length errors.

**The sampler.** `generate_zk_key_from_random_bytes` (`keystore.rs` L184-L188) is now `ZkKey::from(UnsecuredZkKey::from_rng(&mut OsRng))`: same distribution as before (32 bytes reduced mod `p`, `private.rs` L68-L72), one implementation instead of two (#252 S-002). The bias of that distribution is #128 S-002 and is not changed here.

**The 48 call sites** of the removed impls, classified by what the conversion was doing:

| Class | Sites | Rewrite |
|---|---|---|
| Operator input that must not be reduced | `config/mod.rs` L564-L570 (`parse_hex_zk_key`) | length check + `fr_from_bytes` → error on `>= p` |
| Random sampling where reduction is the design | `keystore.rs` L184-L188 | `UnsecuredZkKey::from_rng` |
| Deterministic key material derived from a hash or seed, where reduction is the design | `tools/config/src/consensus.rs` L398, L409, L428 (`derive_key_material`), `tools/config/src/blend.rs` L30 (Ed25519 pk bytes), `tests/testing_framework/src/node/configs/wallet.rs` L141, L154 (`"wl" + index` seed; 32 random bytes) | `ZkKey::new(fr_from_mod_bytes(..))` — the same reduction, now spelled out at the call site |
| Small integer literals in tests, benches and codec fixtures (`ZkKey::from(BigUint::from(1u8))`, `ZkPublicKey::from(BigUint::from(42u64))`) | 40 sites in 21 files: `core` (11 files), `ledger` (6), `wallet`, `zone-sdk`, `tools/blockchain-tools`, `tests` | `Fr::from(n)`; the literal is below `p` by construction, so the value is identical and the codec fixtures' expected bytes are unchanged |
| Not affected (the `From` was ark's own `Fr: From<BigUint>`, not a key impl) | `ZkKey::new(BigUint::from_bytes_le(..).into())` in `config/kms/serde.rs` L36 and `backend/preload.rs` L247; `ZkPublicKey::new(BigUint::from(..).into())` in `ledger` tests; `ZkKey::from(zk_key)` with an `UnsecuredZkKey` argument in `c-bindings/src/api/keys.rs` L185 and `cli/keys.rs` L146, L244 | none |

The rewrite leaves `tools/config`, `tests` and `tests/testing_framework` without any `BigUint` use, so their `num-bigint` dependencies are removed as well (cargo-machete would otherwise fail CI). No production code path outside the parser and the sampler used the lossy impl; the two `tools/config` and two testing-framework sites are provisioning helpers whose reduction is intentional and now explicit.

### 4.3 Item 6 — `VoucherSecret` and `X25519PrivateKey`

`VoucherSecret` (`leader_claim.rs` L45-L46) keeps `Clone`, `Copy`, `Serialize`, `Deserialize` and loses `Default`, `Debug`, `PartialEq`, `Eq`, `Hash` as derives:

- `Debug` is hand-written and prints `VoucherSecret(<redacted>)` (test `voucher_secret_tests::debug_redacts_the_secret`).
- `PartialEq` is `ct_eq` over the scalar's limbs, the pattern of `SecretKey` (`private.rs` L75-L79) and `X25519PrivateKey` (`x25519.rs` L31-L35); `core` gains `subtle` as a dependency (already in the workspace). `Hash` is hand-written over the scalar so it stays consistent with the manual `Eq` and clippy's `derived_hash_with_manual_eq` does not fire (test `equality_compares_the_scalar`).
- `Default` is dropped: the zero scalar is a valid-looking secret whose commitment and nullifier compute fine, and nothing constructed one (`grep VoucherSecret::default` empty; the workspace still compiles with all targets).
- Kept, with the reason: `Copy`, because a non-`Copy` zeroizing newtype is the #123 change and touches the wallet/KMS hand-off (`services/wallet/src/lib.rs` L1174-L1187); `Serialize`/`Deserialize`, because the type is `serde(with = "serde_fr")` on purpose and gating them behind a feature the way `ZkKey` does belongs with #122's rework of that feature. Both are noted in the type's doc comment.

`X25519PrivateKey` (`x25519.rs` L8-L9) loses `Serialize` and `Deserialize` outright rather than gaining a feature gate: it is only ever produced by `derive_x25519()` (`ed25519/private.rs` L53, `ed25519/mod.rs` L63) and consumed by `derive_shared_key` (`blend/message/src/encap/validated.rs` L245, `blend/provers/src/crypto/mod.rs` L27, which already hides it from `Debug` with `educe`), never configured or persisted; removing the derives compiles across the workspace, which is the proof that no serialisation consumer existed.

### 4.4 Verification

| Check | Result |
|---|---|
| `cargo check --workspace --all-targets` | clean, 0 warnings |
| `cargo clippy --all-targets` on `core`, `pol`, `poc`, `key-management-system-keys`, `groth16`, `node`, `config`, `tools`, `testing-framework`, `ledger`, `zone-sdk`, `wallet`, `tests` (workspace lints) | clean, 0 warnings |
| `cargo +nightly fmt --all` | no changes beyond the patched files |
| `cargo machete` (via the repository's pre-commit hook) | flagged `num-bigint` as unused in `tools/config`, `tests` and `tests/testing_framework` once the rewrite removed their last `BigUint` use; the three dependency lines are dropped and the check is clean |
| `cargo test --lib -p logos-blockchain-{pol,poc,key-management-system-keys,groth16}` | 8 + 2 + 10 + 28 passed, 0 failed (includes the 5 new redaction tests) |
| `cargo test --lib -p logos-blockchain-core` | 296 passed, 0 failed, 3 ignored (includes the 4 new tests) |
| `cargo test --lib -p logos-blockchain-ledger` | 162 passed, 0 failed, 1 ignored |
| `cargo test --lib -p logos-blockchain-node zk_key_parser_tests` | 3 passed |

### 4.5 Not done, and why

- `SecretKey::from_rng` still reduces 32 random bytes modulo `p` (#128 S-002). Unifying the two samplers was the item; changing the distribution is a separate, spec-adjacent decision.
- `UnsecuredZkKey` still derives `Debug` and `Serialize` and `ZkKey` still delegates to it under `unsafe` (#128 LB-003). #122 owns that.
- The witness types are not zeroized (#123).
- `PolWitnessInputs` still derives `Serialize`; it must, for the prover.

## 5. Suggestions (non-security)

### S-001 · Apply the patch, and add a workspace-wide redaction test

Appendix B is a `git diff` against `a805329f8` and applies cleanly with `git apply`. A follow-up worth its own small PR: one test module that formats every secret-bearing newtype in `kms/keys` and `core` and asserts `<redacted>`, so the next type added to the family is caught by the same test rather than by the next audit.

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

## Appendix B — The patch

Proposed patch as `git diff` against `a805329f8a186eb6989f09a7c49dee4a0e07473b` (42 files, +440/−103; `Cargo.lock` gains the `subtle` edge for `core` and loses three `num-bigint` edges).

```diff
diff --git a/Cargo.lock b/Cargo.lock
index 0ecedfd37..73ad6a565 100644
--- a/Cargo.lock
+++ b/Cargo.lock
@@ -4490,7 +4490,6 @@ dependencies = [
  "logos-blockchain-node",
  "logos-blockchain-tracing",
  "logos-blockchain-utils",
- "num-bigint",
  "rand 0.8.6",
  "time",
 ]
@@ -4527,6 +4526,7 @@ dependencies = [
  "serde",
  "serde_json",
  "strum",
+ "subtle",
  "thiserror 2.0.18",
  "time",
  "tokio",
@@ -5133,7 +5133,6 @@ dependencies = [
  "logos-blockchain-zksign",
  "logos-blockchain-zone-sdk",
  "logos-sql",
- "num-bigint",
  "rand 0.8.6",
  "reqwest 0.12.28",
  "rpds",
@@ -8288,7 +8287,6 @@ dependencies = [
  "logos-blockchain-time-service",
  "logos-blockchain-tx-service",
  "logos-blockchain-utils",
- "num-bigint",
  "rand 0.8.6",
  "reqwest 0.12.28",
  "serde",
diff --git a/core/Cargo.toml b/core/Cargo.toml
index 9fbc174fe..878fc7318 100644
--- a/core/Cargo.toml
+++ b/core/Cargo.toml
@@ -37,6 +37,7 @@ num-bigint                    = { workspace = true }
 rpds                          = { workspace = true }
 serde                         = { features = ["derive"], workspace = true }
 strum                         = { features = ["derive"], workspace = true }
+subtle                        = { workspace = true }
 thiserror                     = { workspace = true }
 time                          = { workspace = true }
 tracing                       = { workspace = true }
diff --git a/core/src/block/genesis.rs b/core/src/block/genesis.rs
index 73287bb10..d47e4dcce 100644
--- a/core/src/block/genesis.rs
+++ b/core/src/block/genesis.rs
@@ -1290,7 +1290,6 @@ mod tests {
     use lb_codec::BinaryEncode as _;
     use lb_groth16::{AdditiveGroup as _, Fr};
     use lb_key_management_system_keys::keys::{Ed25519PublicKey, ZkPublicKey};
-    use num_bigint::BigUint;
 
     use super::*;
     use crate::{
@@ -1343,7 +1342,7 @@ mod tests {
     }
 
     fn make_note(value: u64) -> Note {
-        Note::new(value, ZkPublicKey::from(BigUint::from(value + 1)))
+        Note::new(value, ZkPublicKey::from(Fr::from(value + 1)))
     }
 
     fn make_sdp_decl(id: u8) -> SDPDeclareOp {
@@ -1352,7 +1351,7 @@ mod tests {
         SDPDeclareOp {
             service_type: ServiceType::BlendNetwork,
             service_note_id: NoteId(Fr::from(u64::from(id))),
-            zk_id: ZkPublicKey::from(BigUint::from(u64::from(id) + 1)),
+            zk_id: ZkPublicKey::from(Fr::from(u64::from(id) + 1)),
             provider_id: ProviderId(Ed25519PublicKey::from_bytes(&[0; 32]).unwrap()),
             locators: "/ip4/1.1.1.1/udp/0".parse::<Locator>().unwrap().into(),
         }
diff --git a/core/src/mantle/batch.rs b/core/src/mantle/batch.rs
index e8edb9fef..e33592286 100644
--- a/core/src/mantle/batch.rs
+++ b/core/src/mantle/batch.rs
@@ -102,7 +102,6 @@ mod tests {
     use lb_groth16::Fr;
     use lb_key_management_system_keys::keys::{ZkKey, public_inputs_from_pks};
     use lb_mmr::MerkleMountainRange;
-    use num_bigint::BigUint;
 
     use super::*;
     use crate::{
@@ -180,7 +179,7 @@ mod tests {
     /// verifier checks it against: those of `msg_for_input`.
     /// If `msg != msg_for_input`, an invalid proof will be produced.
     fn zk_sig(msg: u64, msg_for_input: u64) -> DeferredZkpVerification {
-        let key = ZkKey::from(BigUint::from(1u8));
+        let key = ZkKey::from(Fr::from(1u8));
         let signature = ZkKey::multi_sign(std::slice::from_ref(&key), &Fr::from(msg)).unwrap();
         let inputs =
             public_inputs_from_pks(Fr::from(msg_for_input).into(), &[key.to_public_key()]).unwrap();
diff --git a/core/src/mantle/fixtures/ledger.rs b/core/src/mantle/fixtures/ledger.rs
index 1c5d1fa32..1c4283f5e 100644
--- a/core/src/mantle/fixtures/ledger.rs
+++ b/core/src/mantle/fixtures/ledger.rs
@@ -1,10 +1,11 @@
 use lb_codec::codec_fixtures;
+use lb_groth16::Fr;
 use lb_key_management_system_keys::keys::ZkPublicKey;
 use num_bigint::BigUint;
 
 use crate::mantle::ledger::{Inputs, Note, NoteId, Outputs};
 
 codec_fixtures!(NoteId, Self(BigUint::from(123u64).into()) => "7b00000000000000000000000000000000000000000000000000000000000000");
-codec_fixtures!(Note, Self::new(1000, ZkPublicKey::from(BigUint::from(42u64))) => "e8030000000000002a00000000000000000000000000000000000000000000000000000000000000");
+codec_fixtures!(Note, Self::new(1000, ZkPublicKey::from(Fr::from(42u64))) => "e8030000000000002a00000000000000000000000000000000000000000000000000000000000000");
 codec_fixtures!(Inputs, Self::new([NoteId(BigUint::from(123u64).into())]) => "017b00000000000000000000000000000000000000000000000000000000000000");
-codec_fixtures!(Outputs, Self::new([Note::new(1000, ZkPublicKey::from(BigUint::from(42u64)))]) => "01e8030000000000002a00000000000000000000000000000000000000000000000000000000000000");
+codec_fixtures!(Outputs, Self::new([Note::new(1000, ZkPublicKey::from(Fr::from(42u64)))]) => "01e8030000000000002a00000000000000000000000000000000000000000000000000000000000000");
diff --git a/core/src/mantle/ops/leader_claim.rs b/core/src/mantle/ops/leader_claim.rs
index 5099dd88f..7cf08daf7 100644
--- a/core/src/mantle/ops/leader_claim.rs
+++ b/core/src/mantle/ops/leader_claim.rs
@@ -6,6 +6,7 @@ use lb_key_management_system_keys::keys::ZkPublicKey;
 use lb_poc::PoCVerifierInput;
 use lb_poseidon2::{Digest, Fr, ZkHash};
 use serde::{Deserialize, Serialize};
+use subtle::ConstantTimeEq as _;
 use thiserror::Error;
 
 use crate::{
@@ -42,9 +43,32 @@ static VOUCHER_NF: LazyLock<Fr> = LazyLock::new(|| {
 #[derive(Clone, Copy, Debug, Eq, PartialEq, Hash, Default, Serialize, Deserialize, BinaryCodec)]
 pub struct RewardsRoot(#[serde(with = "serde_fr")] ZkHash);
 
-#[derive(Clone, Copy, Debug, Default, Eq, PartialEq, Hash, Serialize, Deserialize)]
+/// Preimage of a voucher commitment and nullifier: whoever holds it can claim
+/// the voucher's reward. No `Default` (the zero scalar is a valid-looking
+/// secret), redacted `Debug`, constant-time equality.
+#[derive(Clone, Copy, Serialize, Deserialize)]
 pub struct VoucherSecret(#[serde(with = "serde_fr")] pub Fr);
 
+impl core::fmt::Debug for VoucherSecret {
+    fn fmt(&self, f: &mut core::fmt::Formatter<'_>) -> core::fmt::Result {
+        f.write_str("VoucherSecret(<redacted>)")
+    }
+}
+
+impl PartialEq for VoucherSecret {
+    fn eq(&self, other: &Self) -> bool {
+        self.0.0.0.ct_eq(&other.0.0.0).into()
+    }
+}
+
+impl Eq for VoucherSecret {}
+
+impl core::hash::Hash for VoucherSecret {
+    fn hash<H: core::hash::Hasher>(&self, state: &mut H) {
+        self.0.hash(state);
+    }
+}
+
 #[derive(Clone, Copy, Debug, Default, Eq, PartialEq, Hash, Serialize, Deserialize, BinaryCodec)]
 pub struct VoucherNullifier(#[serde(with = "serde_fr")] ZkHash);
 
@@ -498,3 +522,25 @@ mod tests {
         ));
     }
 }
+
+#[cfg(test)]
+mod voucher_secret_tests {
+    use super::*;
+
+    #[test]
+    fn debug_redacts_the_secret() {
+        let secret = Fr::from(12_648_430u64);
+        let rendered = format!("{:?}", VoucherSecret::from(secret));
+        assert_eq!(rendered, "VoucherSecret(<redacted>)");
+        assert!(!rendered.contains(&secret.to_string()));
+    }
+
+    #[test]
+    fn equality_compares_the_scalar() {
+        let a = VoucherSecret::from(Fr::from(7u64));
+        let b = VoucherSecret::from(Fr::from(7u64));
+        let c = VoucherSecret::from(Fr::from(8u64));
+        assert_eq!(a, b);
+        assert_ne!(a, c);
+    }
+}
diff --git a/core/src/mantle/ops/op_proof.rs b/core/src/mantle/ops/op_proof.rs
index 806919085..d31b7d8ae 100644
--- a/core/src/mantle/ops/op_proof.rs
+++ b/core/src/mantle/ops/op_proof.rs
@@ -212,9 +212,8 @@ pub mod placeholders {
 #[cfg(test)]
 mod tests {
     use lb_codec::BinaryEncode as _;
-    use lb_groth16::{COMPRESSED_PROOF_SIZE, CompressedGroth16Proof};
+    use lb_groth16::{COMPRESSED_PROOF_SIZE, CompressedGroth16Proof, Fr};
     use lb_key_management_system_keys::keys::ZkPublicKey;
-    use num_bigint::BigUint;
 
     use crate::{
         mantle::{
@@ -229,7 +228,7 @@ mod tests {
         let op = Op::LeaderClaim(LeaderClaimOp {
             rewards_root: RewardsRoot::default(),
             voucher_nullifier: VoucherNullifier::default(),
-            pk: ZkPublicKey::from(BigUint::from(0u64)),
+            pk: ZkPublicKey::from(Fr::from(0u64)),
         });
 
         let proof_bytes: [u8; 128] = core::array::from_fn(|i| i as u8);
diff --git a/core/src/mantle/ops/sdp/active.rs b/core/src/mantle/ops/sdp/active.rs
index dc182cd63..1982185f9 100644
--- a/core/src/mantle/ops/sdp/active.rs
+++ b/core/src/mantle/ops/sdp/active.rs
@@ -139,7 +139,6 @@ mod tests {
     use lb_cryptarchia_engine::Epoch;
     use lb_groth16::{AdditiveGroup as _, CompressedGroth16Proof, Fr};
     use lb_key_management_system_keys::keys::{Ed25519Key, ZkKey, ZkSignature};
-    use num_bigint::BigUint;
 
     use super::{SDPActiveOp, SDPActiveValidationContext, SdpError};
     use crate::{
@@ -181,7 +180,7 @@ mod tests {
                 .try_into()
                 .unwrap(),
             provider_id: signing_key.public_key().into(),
-            zk_id: ZkKey::from(BigUint::from(1u64)).to_public_key(),
+            zk_id: ZkKey::from(Fr::from(1u64)).to_public_key(),
             service_note_id: Fr::ZERO.into(),
         };
         let mut declaration = Declaration::new(Epoch::new(0), &declare_op);
diff --git a/core/src/mantle/ops/sdp/declare.rs b/core/src/mantle/ops/sdp/declare.rs
index 79703125a..62823259d 100644
--- a/core/src/mantle/ops/sdp/declare.rs
+++ b/core/src/mantle/ops/sdp/declare.rs
@@ -284,7 +284,6 @@ mod tests {
     use lb_cryptarchia_engine::Epoch;
     use lb_groth16::{AdditiveGroup as _, Fr};
     use lb_key_management_system_keys::keys::{Ed25519Key, ZkKey};
-    use num_bigint::BigUint;
 
     use super::{SDPDeclareOp, SdpError, validate_service_scoped_uniqueness};
     use crate::{
@@ -299,7 +298,7 @@ mod tests {
             provider_id: Ed25519Key::from_bytes(&[provider_sk; 32])
                 .public_key()
                 .into(),
-            zk_id: ZkKey::from(BigUint::from(zk_sk)).to_public_key(),
+            zk_id: ZkKey::from(Fr::from(zk_sk)).to_public_key(),
             service_note_id: Fr::ZERO.into(),
         }
     }
diff --git a/core/src/mantle/transactions/codec.rs b/core/src/mantle/transactions/codec.rs
index ee2b4ccfe..3a1196e09 100644
--- a/core/src/mantle/transactions/codec.rs
+++ b/core/src/mantle/transactions/codec.rs
@@ -295,7 +295,7 @@ mod tests {
     #[test]
     fn test_encode_decode_roundtrip_with_transfer() {
         // Create a MantleTx with ledger inputs and outputs
-        let pk = ZkPublicKey::from(BigUint::from(42u64));
+        let pk = ZkPublicKey::from(Fr::from(42u64));
         let note = Note::new(1000, pk);
         let note_id = NoteId(BigUint::from(123u64).into());
         let transfer_op = TransferOp::new(Inputs::new([note_id]), Outputs::new([note]));
@@ -403,7 +403,7 @@ mod tests {
         let locator1: Multiaddr = "/ip4/127.0.0.1/tcp/8080".parse().unwrap();
         let locator2: Multiaddr = "/ip6/::1/tcp/9090".parse().unwrap();
 
-        let service_note_sk = ZkKey::from(BigUint::from(1u64));
+        let service_note_sk = ZkKey::from(Fr::from(1u64));
         let service_note = Utxo {
             op_id: [1u8; 32],
             output_index: 12,
@@ -574,8 +574,8 @@ mod tests {
 
     #[test]
     fn test_minimum_signed_mantle_tx_size_with_ledger_inputs_outputs() {
-        let pk1 = ZkPublicKey::from(BigUint::from(100u64));
-        let pk2 = ZkPublicKey::from(BigUint::from(200u64));
+        let pk1 = ZkPublicKey::from(Fr::from(100u64));
+        let pk2 = ZkPublicKey::from(Fr::from(200u64));
 
         let note1 = Note::new(1000, pk1);
         let note2 = Note::new(2000, pk2);
@@ -627,7 +627,7 @@ mod tests {
             transfer_threshold: 0,
         };
 
-        let service_note_sk = ZkKey::from(BigUint::from(1u64));
+        let service_note_sk = ZkKey::from(Fr::from(1u64));
         let transfer_op = TransferOp {
             inputs: Inputs::new([NoteId(BigUint::from(777u64).into())]),
             outputs: Outputs::new([Note::new(5000, service_note_sk.to_public_key())]),
@@ -685,7 +685,7 @@ mod tests {
         let leader_claim_op = LeaderClaimOp {
             rewards_root: RewardsRoot::default(),
             voucher_nullifier: VoucherNullifier::default(),
-            pk: ZkPublicKey::from(BigUint::from(0u64)),
+            pk: ZkPublicKey::from(Fr::from(0u64)),
         };
 
         let mantle_tx = Ops::new_unchecked(vec![Op::LeaderClaim(leader_claim_op)]);
@@ -711,7 +711,7 @@ mod tests {
         let leader_claim_op = LeaderClaimOp {
             rewards_root: RewardsRoot::default(),
             voucher_nullifier: VoucherNullifier::default(),
-            pk: ZkPublicKey::from(BigUint::from(0u64)),
+            pk: ZkPublicKey::from(Fr::from(0u64)),
         };
         let op = Op::LeaderClaim(leader_claim_op);
 
@@ -1022,7 +1022,7 @@ mod tests {
 
     #[test]
     fn test_encode_decode_max_outputs() {
-        let note = Note::new(1000, ZkPublicKey::from(BigUint::from(42u64)));
+        let note = Note::new(1000, ZkPublicKey::from(Fr::from(42u64)));
         let outputs = [note; u8::MAX as usize];
         let outputs = BoundedOutputs::from(outputs);
 
diff --git a/core/src/mantle/transactions/genesis_tx.rs b/core/src/mantle/transactions/genesis_tx.rs
index 057a45963..f2995eeff 100644
--- a/core/src/mantle/transactions/genesis_tx.rs
+++ b/core/src/mantle/transactions/genesis_tx.rs
@@ -442,7 +442,7 @@ mod tests {
 
     // Helper function to create a test note
     fn create_test_note(value: Value) -> Note {
-        Note::new(value, ZkPublicKey::from(BigUint::from(123u64)))
+        Note::new(value, ZkPublicKey::from(Fr::from(123u64)))
     }
 
     // Helper function to build a proof of the variant expected for a given op
diff --git a/core/src/mantle/transactions/tx_list/signed_ops.rs b/core/src/mantle/transactions/tx_list/signed_ops.rs
index e58415d85..4909be866 100644
--- a/core/src/mantle/transactions/tx_list/signed_ops.rs
+++ b/core/src/mantle/transactions/tx_list/signed_ops.rs
@@ -709,7 +709,7 @@ mod tests {
 
     #[test]
     fn test_signed_mantle_tx_new_rejects_zero_value_transfer_output() {
-        let input_sk = ZkKey::from(BigUint::from(1u8));
+        let input_sk = ZkKey::from(Fr::from(1u8));
         let input_utxo = Utxo {
             op_id: [1u8; 32],
             output_index: 0,
diff --git a/core/src/mantle/transactions/verified_ops.rs b/core/src/mantle/transactions/verified_ops.rs
index f42d8a435..c01397eab 100644
--- a/core/src/mantle/transactions/verified_ops.rs
+++ b/core/src/mantle/transactions/verified_ops.rs
@@ -73,8 +73,8 @@ impl From<SignedOps<Preverified, StandardMode>> for VerifiedOperations {
 
 #[cfg(test)]
 mod tests {
+    use lb_groth16::Fr;
     use lb_key_management_system_keys::keys::{Ed25519Key, ZkKey};
-    use num_bigint::BigUint;
 
     use crate::mantle::{
         Note, Utxo, VerificationError,
@@ -94,7 +94,7 @@ mod tests {
         let key1 = Ed25519Key::from_bytes(&[9; 32]);
         let keys = Keys::new_unchecked(vec![key0.public_key(), key1.public_key()]);
 
-        let input_sk = ZkKey::from(BigUint::from(1u8));
+        let input_sk = ZkKey::from(Fr::from(1u8));
         let utxo = Utxo {
             op_id: [1u8; 32],
             output_index: 0,
diff --git a/core/src/proofs/leader_claim_proof.rs b/core/src/proofs/leader_claim_proof.rs
index 96b16b2b2..d55874d91 100644
--- a/core/src/proofs/leader_claim_proof.rs
+++ b/core/src/proofs/leader_claim_proof.rs
@@ -121,11 +121,19 @@ impl LeaderClaimPublic {
     }
 }
 
-#[derive(Debug, Clone)]
+#[derive(Clone)]
 pub struct LeaderClaimPrivate {
     input: lb_poc::PoCWitnessInputsData,
 }
 
+impl core::fmt::Debug for LeaderClaimPrivate {
+    fn fmt(&self, f: &mut core::fmt::Formatter<'_>) -> core::fmt::Result {
+        f.debug_struct("LeaderClaimPrivate")
+            .field("input", &self.input)
+            .finish()
+    }
+}
+
 impl LeaderClaimPrivate {
     pub fn try_new(
         public: LeaderClaimPublic,
@@ -180,3 +188,28 @@ mod proof_serde {
         Ok(lb_poc::PoCProof::from_bytes(&proof_array))
     }
 }
+
+#[cfg(test)]
+mod redaction_tests {
+    use ark_ff::AdditiveGroup as _;
+
+    use super::*;
+
+    #[test]
+    fn leader_claim_private_debug_redacts_secret_voucher() {
+        let secret = Fr::from(12_648_430u64);
+        let path = MerklePath::try_new(0, vec![Fr::ZERO; lb_poc::VOUCHER_MERKLE_PATH_LEN])
+            .expect("path of the circuit length is valid");
+        let private = LeaderClaimPrivate::try_new(
+            LeaderClaimPublic::new(Fr::from(1u64), Fr::from(2u64), Fr::from(3u64)),
+            &path,
+            VoucherSecret::from(secret),
+        )
+        .expect("witness must build");
+        let rendered = format!("{private:?}");
+
+        assert!(rendered.contains("<redacted>"), "{rendered}");
+        assert!(!rendered.contains(&format!("{secret:?}")), "{rendered}");
+        assert!(!rendered.contains(&secret.to_string()), "{rendered}");
+    }
+}
diff --git a/core/src/proofs/leader_proof.rs b/core/src/proofs/leader_proof.rs
index cbaab0702..72b781bdb 100644
--- a/core/src/proofs/leader_proof.rs
+++ b/core/src/proofs/leader_proof.rs
@@ -278,12 +278,21 @@ impl LeaderPublic {
 static LEAD_V1: LazyLock<Fr> =
     LazyLock::new(|| fr_from_bytes(b"LEAD_V1").expect("BigUint should load from constant string"));
 
-#[derive(Debug, Clone)]
+#[derive(Clone)]
 pub struct LeaderPrivate {
     input: lb_pol::PolWitnessInputsData,
     pk: Ed25519PublicKey,
 }
 
+impl Debug for LeaderPrivate {
+    fn fmt(&self, f: &mut core::fmt::Formatter<'_>) -> core::fmt::Result {
+        f.debug_struct("LeaderPrivate")
+            .field("input", &self.input)
+            .field("pk", &self.pk)
+            .finish()
+    }
+}
+
 impl LeaderPrivate {
     #[must_use]
     pub fn new(
@@ -473,3 +482,33 @@ mod tests {
         });
     }
 }
+
+#[cfg(test)]
+mod redaction_tests {
+    use lb_key_management_system_keys::keys::Ed25519Key;
+    use lb_utxotree::UtxoTree;
+
+    use super::*;
+    use crate::{crypto::ZkHasher, mantle::ledger::Note};
+
+    #[test]
+    fn leader_private_debug_redacts_secret_key() {
+        let secret_key = Fr::from(12_648_430u64);
+        let utxo = Utxo {
+            op_id: [0u8; 32],
+            output_index: 0,
+            note: Note::new(1, ZkPublicKey::from(Fr::ZERO)),
+        };
+        let tree = UtxoTree::<_, _, ZkHasher>::new().insert(utxo.id(), utxo).0;
+        let path = tree.path(&utxo.id()).expect("note must exist in tree");
+        let public = LeaderPublic::new(tree.root(), tree.root(), Fr::ZERO, 0, Fr::ZERO, Fr::ZERO);
+        let pk = Ed25519Key::from_bytes(&[0; 32]).public_key();
+
+        let private = LeaderPrivate::new(public, utxo, &path, &path, secret_key, &pk);
+        let rendered = format!("{private:?}");
+
+        assert!(rendered.contains("<redacted>"), "{rendered}");
+        assert!(!rendered.contains(&format!("{secret_key:?}")), "{rendered}");
+        assert!(!rendered.contains(&secret_key.to_string()), "{rendered}");
+    }
+}
diff --git a/core/src/sdp/service_notes.rs b/core/src/sdp/service_notes.rs
index 5954939ec..f294f34c8 100644
--- a/core/src/sdp/service_notes.rs
+++ b/core/src/sdp/service_notes.rs
@@ -121,8 +121,8 @@ impl ServiceNotes {
 
 #[cfg(test)]
 mod tests {
+    use lb_groth16::Fr;
     use lb_key_management_system_keys::keys::ZkKey;
-    use num_bigint::BigUint;
     use rand::{RngCore as _, thread_rng};
 
     use super::*;
@@ -131,7 +131,7 @@ mod tests {
     fn utxo() -> Utxo {
         let mut op_id = [0u8; 32];
         thread_rng().fill_bytes(&mut op_id);
-        let zk_sk = ZkKey::from(BigUint::from(0u64));
+        let zk_sk = ZkKey::from(Fr::from(0u64));
         Utxo {
             op_id,
             output_index: 0,
diff --git a/kms/keys/src/keys/ed25519/x25519.rs b/kms/keys/src/keys/ed25519/x25519.rs
index 8f535ca7f..0bd3a8150 100644
--- a/kms/keys/src/keys/ed25519/x25519.rs
+++ b/kms/keys/src/keys/ed25519/x25519.rs
@@ -5,7 +5,10 @@ use zeroize::ZeroizeOnDrop;
 
 pub const X25519_SECRET_KEY_LENGTH: usize = 32;
 
-#[derive(Clone, ZeroizeOnDrop, Deserialize, Serialize)]
+/// Derived from an Ed25519 key on demand and never configured or persisted, so
+/// it carries no `Serialize`/`Deserialize`, unlike the two KMS key types whose
+/// export is gated behind the `unsafe` feature.
+#[derive(Clone, ZeroizeOnDrop)]
 pub struct X25519PrivateKey(StaticSecret);
 
 impl X25519PrivateKey {
diff --git a/kms/keys/src/keys/zk/mod.rs b/kms/keys/src/keys/zk/mod.rs
index be5f65cb6..b9a4a1ce1 100644
--- a/kms/keys/src/keys/zk/mod.rs
+++ b/kms/keys/src/keys/zk/mod.rs
@@ -91,9 +91,11 @@ impl From<Fr> for ZkKey {
     }
 }
 
-impl From<BigUint> for ZkKey {
-    fn from(value: BigUint) -> Self {
-        Self(value.into())
+impl TryFrom<BigUint> for ZkKey {
+    type Error = lb_groth16::FrFromBytesError;
+
+    fn try_from(value: BigUint) -> Result<Self, Self::Error> {
+        UnsecuredZkKey::try_from(value).map(Self)
     }
 }
 
diff --git a/kms/keys/src/keys/zk/private.rs b/kms/keys/src/keys/zk/private.rs
index 01bf4f3c9..b64ebedb9 100644
--- a/kms/keys/src/keys/zk/private.rs
+++ b/kms/keys/src/keys/zk/private.rs
@@ -1,6 +1,9 @@
 use std::sync::LazyLock;
 
-use lb_groth16::{AdditiveGroup as _, Field as _, Fr, fr_from_bytes_unchecked, fr_from_mod_bytes};
+use lb_groth16::{
+    AdditiveGroup as _, Field as _, Fr, FrFromBytesError, fr_from_bytes_unchecked,
+    fr_from_mod_bytes, fr_try_from_biguint,
+};
 use lb_poseidon2::{Digest, Poseidon2Bn254Hasher};
 use lb_zksign::{ZkSignError, ZkSignPrivateKeysData, ZkSignWitnessInputs};
 use num_bigint::BigUint;
@@ -80,9 +83,13 @@ impl PartialEq for SecretKey {
 
 impl Eq for SecretKey {}
 
-impl From<BigUint> for SecretKey {
-    fn from(value: BigUint) -> Self {
-        Self(value.into())
+impl TryFrom<BigUint> for SecretKey {
+    type Error = FrFromBytesError;
+
+    /// Fails for values `>= p` instead of reducing them, so a key given as
+    /// bytes is exactly the scalar the caller supplied.
+    fn try_from(value: BigUint) -> Result<Self, Self::Error> {
+        fr_try_from_biguint(value).map(Self)
     }
 }
 
diff --git a/kms/keys/src/keys/zk/public.rs b/kms/keys/src/keys/zk/public.rs
index 8650517c3..ffa7b6b1c 100644
--- a/kms/keys/src/keys/zk/public.rs
+++ b/kms/keys/src/keys/zk/public.rs
@@ -74,9 +74,11 @@ impl From<PublicKey> for Fr {
     }
 }
 
-impl From<BigUint> for PublicKey {
-    fn from(value: BigUint) -> Self {
-        Fr::from(value).into()
+impl TryFrom<BigUint> for PublicKey {
+    type Error = lb_groth16::FrFromBytesError;
+
+    fn try_from(value: BigUint) -> Result<Self, Self::Error> {
+        lb_groth16::fr_try_from_biguint(value).map(Self)
     }
 }
 
diff --git a/ledger/benches/zk_batch.rs b/ledger/benches/zk_batch.rs
index c7b6a042a..a91b08503 100644
--- a/ledger/benches/zk_batch.rs
+++ b/ledger/benches/zk_batch.rs
@@ -37,6 +37,7 @@ use lb_core::{
     sdp::{MinStake, ServiceParameters, ServiceType},
 };
 use lb_cryptarchia_engine::EpochConfig;
+use lb_groth16::Fr;
 use lb_key_management_system_keys::keys::ZkKey;
 use lb_utils::math::{NonNegativeRatio, PositiveF64};
 use lb_zksign::verify;
@@ -45,7 +46,6 @@ use logos_blockchain_ledger::{
     config::{BlendPoWConfig, ModulusShift, PoWConfig, RewardPoWConfig},
     mantle::sdp::{ServiceRewardsParameters, rewards},
 };
-use num_bigint::BigUint;
 
 fn main() {
     divan::main();
@@ -176,7 +176,7 @@ static TX_POOL: LazyLock<TxPool> = LazyLock::new(|| {
     let n_txs = *single_block::TXS_PER_BLOCK.last().unwrap();
 
     let keys: Vec<ZkKey> = (0..n_txs)
-        .map(|i| ZkKey::from(BigUint::from(i as u64 + 1)))
+        .map(|i| ZkKey::from(Fr::from(i as u64 + 1)))
         .collect();
     let utxos: Vec<Utxo> = keys
         .iter()
diff --git a/ledger/src/cryptarchia/mod.rs b/ledger/src/cryptarchia/mod.rs
index 1d8955548..4d80f2e09 100644
--- a/ledger/src/cryptarchia/mod.rs
+++ b/ledger/src/cryptarchia/mod.rs
@@ -902,7 +902,7 @@ pub mod tests {
     pub fn utxo_with_sk() -> (ZkKey, Utxo) {
         let mut op_id = [0u8; 32];
         thread_rng().fill_bytes(&mut op_id);
-        let zk_sk = ZkKey::from(BigUint::from(0u64));
+        let zk_sk = ZkKey::from(Fr::from(0u64));
         let utxo = Utxo {
             op_id,
             output_index: 0,
@@ -1991,8 +1991,8 @@ pub mod tests {
 
     #[test]
     fn test_invalid_double_spend_transfer() {
-        let note_sk = ZkKey::from(BigUint::from(1u8));
-        let output_note_sk = ZkKey::from(BigUint::from(2u8));
+        let note_sk = ZkKey::from(Fr::from(1u8));
+        let output_note_sk = ZkKey::from(Fr::from(2u8));
         let input_note = Note::new(100, note_sk.to_public_key());
         let input_utxo = Utxo {
             op_id: [1u8; 32],
@@ -2017,9 +2017,9 @@ pub mod tests {
 
     #[test]
     fn test_tx_processing_valid_transaction() {
-        let note_sk = ZkKey::from(BigUint::from(1u8));
-        let output_note1_sk = ZkKey::from(BigUint::from(2u8));
-        let output_note2_sk = ZkKey::from(BigUint::from(3u8));
+        let note_sk = ZkKey::from(Fr::from(1u8));
+        let output_note1_sk = ZkKey::from(Fr::from(2u8));
+        let output_note2_sk = ZkKey::from(Fr::from(3u8));
         let input_note = Note::new(11000, note_sk.to_public_key());
         let input_utxo = Utxo {
             op_id: [1u8; 32],
@@ -2084,7 +2084,7 @@ pub mod tests {
 
     #[test]
     fn test_tx_processing_invalid_input() {
-        let input_sk = ZkKey::from(BigUint::from(1u8));
+        let input_sk = ZkKey::from(Fr::from(1u8));
         let input_note = Note::new(1000, input_sk.to_public_key());
         let input_utxo = Utxo {
             op_id: [1u8; 32],
@@ -2133,7 +2133,7 @@ pub mod tests {
 
     #[test]
     fn test_tx_processing_insufficient_balance() {
-        let input_sk = ZkKey::from(BigUint::from(1u8));
+        let input_sk = ZkKey::from(Fr::from(1u8));
         let input_note = Note::new(1, input_sk.to_public_key());
         let input_utxo = Utxo {
             op_id: [1u8; 32],
@@ -2171,7 +2171,7 @@ pub mod tests {
 
     #[test]
     fn test_tx_processing_no_outputs() {
-        let input_sk = ZkKey::from(BigUint::from(1u8));
+        let input_sk = ZkKey::from(Fr::from(1u8));
         let input_note = Note::new(10000, input_sk.to_public_key());
         let input_utxo = Utxo {
             op_id: [1u8; 32],
diff --git a/ledger/src/cryptarchia/test/tsi_simulation.rs b/ledger/src/cryptarchia/test/tsi_simulation.rs
index c33995837..a13035c43 100644
--- a/ledger/src/cryptarchia/test/tsi_simulation.rs
+++ b/ledger/src/cryptarchia/test/tsi_simulation.rs
@@ -39,7 +39,6 @@ use lb_cryptarchia_engine::{Branch, Cryptarchia, Slot, State, UncleSlots};
 use lb_groth16::{AdditiveGroup as _, Fr};
 use lb_key_management_system_keys::keys::ZkKey;
 use lb_utils::math::NonNegativeRatio;
-use num_bigint::BigUint;
 use rand::{Rng as _, SeedableRng as _, rngs::StdRng};
 
 use crate::{
@@ -199,10 +198,7 @@ fn leader_utxo() -> Utxo {
     Utxo {
         op_id: [0u8; 32],
         output_index: 0,
-        note: Note::new(
-            TOTAL_STAKE,
-            ZkKey::from(BigUint::from(0u64)).to_public_key(),
-        ),
+        note: Note::new(TOTAL_STAKE, ZkKey::from(Fr::from(0u64)).to_public_key()),
     }
 }
 
diff --git a/ledger/src/lib.rs b/ledger/src/lib.rs
index 9d0ec5b48..f74fe0b9f 100644
--- a/ledger/src/lib.rs
+++ b/ledger/src/lib.rs
@@ -1261,7 +1261,7 @@ mod tests {
     fn test_ledger_try_update_with_transaction() {
         let (mut ledger, genesis_id, utxo) = create_test_ledger();
         let mut output_note = Note::new(1, ZkPublicKey::new(BigUint::from(1u8).into()));
-        let sk = ZkKey::from(BigUint::from(0u8));
+        let sk = ZkKey::from(Fr::from(0u8));
         // determine fees
         let ops = Ops::from([Op::Transfer(TransferOp::new(
             Inputs::try_new(vec![utxo.id()]).unwrap(),
@@ -2580,7 +2580,7 @@ mod tests {
         update_ledger_prices(&mut ledger, 1, 1);
 
         let mut output_note = Note::new(1, ZkPublicKey::new(BigUint::from(0u8).into()));
-        let sk = ZkKey::from(BigUint::from(0u8));
+        let sk = ZkKey::from(Fr::from(0u8));
         let tx = create_tx(
             vec![utxo.id()],
             vec![output_note],
@@ -2625,7 +2625,7 @@ mod tests {
         let mut ledger = LedgerState::from_utxos([utxo], &config);
 
         let mut output_note = Note::new(1, ZkPublicKey::new(BigUint::from(0u8).into()));
-        let sk = ZkKey::from(BigUint::from(0u8));
+        let sk = ZkKey::from(Fr::from(0u8));
         let tx = create_tx(
             vec![utxo.id()],
             vec![output_note],
@@ -2697,7 +2697,7 @@ mod tests {
         // No outputs: the whole input covers the gas cost and the remainder is
         // tipped. We only assert on the storage counter, not the tip, so the
         // exact balance is irrelevant.
-        let sk = ZkKey::from(BigUint::from(0u8));
+        let sk = ZkKey::from(Fr::from(0u8));
         let tx = create_tx(vec![utxo.id()], vec![], std::slice::from_ref(&sk))
             .preverify()
             .unwrap();
diff --git a/ledger/src/mantle/sdp/mod.rs b/ledger/src/mantle/sdp/mod.rs
index debfedd4e..d5965a007 100644
--- a/ledger/src/mantle/sdp/mod.rs
+++ b/ledger/src/mantle/sdp/mod.rs
@@ -700,7 +700,6 @@ mod tests {
     use lb_groth16::{AdditiveGroup as _, CompressedGroth16Proof, Fr};
     use lb_key_management_system_keys::keys::{Ed25519Key, Ed25519Signature, ZkKey, ZkSignature};
     use lb_utils::math::PositiveF64;
-    use num_bigint::BigUint;
 
     use super::*;
     use crate::{
@@ -730,7 +729,7 @@ mod tests {
     }
 
     fn create_zk_key(sk: u64) -> ZkKey {
-        ZkKey::from(BigUint::from(sk))
+        ZkKey::from(Fr::from(sk))
     }
 
     fn create_signing_key() -> Ed25519Key {
diff --git a/ledger/src/update.rs b/ledger/src/update.rs
index 1bb8bd8d1..d27d022cc 100644
--- a/ledger/src/update.rs
+++ b/ledger/src/update.rs
@@ -53,7 +53,6 @@ mod tests {
     };
     use lb_groth16::Fr;
     use lb_key_management_system_keys::keys::{ZkKey, public_inputs_from_pks};
-    use num_bigint::BigUint;
 
     use super::*;
     use crate::cryptarchia::tests::{config, utxo};
@@ -111,7 +110,7 @@ mod tests {
     /// If `msg == msg_for_input`, a valid sig is produced.
     /// Otherwise, an invalid sig is produced.
     fn zk_sig(msg: u64, msg_for_input: u64) -> DeferredZkpVerification {
-        let key = ZkKey::from(BigUint::from(1u8));
+        let key = ZkKey::from(Fr::from(1u8));
         let signature = ZkKey::multi_sign(std::slice::from_ref(&key), &Fr::from(msg)).unwrap();
         let inputs =
             public_inputs_from_pks(Fr::from(msg_for_input).into(), &[key.to_public_key()]).unwrap();
diff --git a/nodes/node/binary/src/cli/config/keystore.rs b/nodes/node/binary/src/cli/config/keystore.rs
index 3ee9fb281..291cef108 100644
--- a/nodes/node/binary/src/cli/config/keystore.rs
+++ b/nodes/node/binary/src/cli/config/keystore.rs
@@ -7,7 +7,6 @@ use lb_key_management_system_service::{
         Ed25519Key, Key, UnsecuredEd25519Key, UnsecuredZkKey, ZkKey, secured_key::SecuredKey as _,
     },
 };
-use num_bigint::BigUint;
 use rand::rngs::OsRng;
 use serde::{Deserialize, Serialize};
 use thiserror::Error;
@@ -182,7 +181,5 @@ fn key_id(key: &Key) -> KeyId {
 }
 
 fn generate_zk_key_from_random_bytes() -> ZkKey {
-    let mut bytes = [0u8; 32];
-    rand::RngCore::fill_bytes(&mut OsRng, &mut bytes);
-    ZkKey::from(BigUint::from_bytes_le(&bytes))
+    ZkKey::from(UnsecuredZkKey::from_rng(&mut OsRng))
 }
diff --git a/nodes/node/binary/src/config/mod.rs b/nodes/node/binary/src/config/mod.rs
index 4329dad42..d2669a2cd 100644
--- a/nodes/node/binary/src/config/mod.rs
+++ b/nodes/node/binary/src/config/mod.rs
@@ -17,7 +17,6 @@ use lb_tracing::{
     filter::envfilter::{default_envfilter_config, parse_filter_directives},
     logging::local::{AppenderType, CompressionType, RetentionType, RollingConfig, RotationType},
 };
-use num_bigint::BigUint;
 use serde::{Deserialize, Serialize};
 
 pub use crate::config::{
@@ -564,9 +563,16 @@ pub fn parse_hex_public_key(key: &str) -> Result<ZkPublicKey, String> {
 pub fn parse_hex_zk_key(s: &str) -> Result<UnsecuredZkKey, String> {
     let bytes = hex::decode(s).map_err(|e| format!("Invalid hex string for ZK key: {e}"))?;
 
-    let big_uint = BigUint::from_bytes_le(&bytes);
+    let expected_len = size_of::<lb_groth16::FrBytes>();
+    if bytes.len() != expected_len {
+        return Err(format!(
+            "Invalid ZK key length: expected {expected_len} bytes, got {}",
+            bytes.len()
+        ));
+    }
 
-    Ok(UnsecuredZkKey::from(big_uint))
+    let fr = fr_from_bytes(&bytes).map_err(|e| format!("Invalid ZK key: {e}"))?;
+    Ok(UnsecuredZkKey::new(fr))
 }
 
 pub fn parse_hex_ed25519_key(key: &str) -> Result<SecretKey, String> {
@@ -575,3 +581,31 @@ pub fn parse_hex_ed25519_key(key: &str) -> Result<SecretKey, String> {
     SecretKey::try_from_bytes(key_bytes.as_mut_slice())
         .map_err(|e| format!("Failed to deserialize ed25519 key from bytes: {e}"))
 }
+
+#[cfg(test)]
+mod zk_key_parser_tests {
+    use lb_groth16::{Fr, fr_to_bytes};
+
+    use super::parse_hex_zk_key;
+
+    #[test]
+    fn accepts_a_32_byte_scalar_below_the_modulus() {
+        let hex = hex::encode(fr_to_bytes(&Fr::from(7u64)));
+        let key = parse_hex_zk_key(&hex).expect("in-range key parses");
+        assert_eq!(*key.as_fr(), Fr::from(7u64));
+    }
+
+    #[test]
+    fn rejects_a_value_at_or_above_the_modulus() {
+        let error = parse_hex_zk_key(&hex::encode([0xffu8; 32])).expect_err("must not reduce");
+        assert!(error.contains("Invalid ZK key"), "{error}");
+    }
+
+    #[test]
+    fn rejects_short_and_long_keys() {
+        for len in [0usize, 1, 31, 33, 64] {
+            let error = parse_hex_zk_key(&hex::encode(vec![1u8; len])).expect_err("wrong length");
+            assert!(error.contains("length"), "{error}");
+        }
+    }
+}
diff --git a/tests/Cargo.toml b/tests/Cargo.toml
index 3538c0d9a..4cf177390 100644
--- a/tests/Cargo.toml
+++ b/tests/Cargo.toml
@@ -53,7 +53,6 @@ lb-zksign                        = { workspace = true }
 lb-zone-sdk                      = { workspace = true }
 libp2p                           = { workspace = true }
 logos-sql                        = { workspace = true }
-num-bigint                       = { workspace = true }
 rand                             = { workspace = true }
 reqwest                          = { features = ["json"], workspace = true }
 rpds                             = { workspace = true }
diff --git a/tests/src/tests/mantle/sdp/ops.rs b/tests/src/tests/mantle/sdp/ops.rs
index fab4c9b8d..a7618a815 100644
--- a/tests/src/tests/mantle/sdp/ops.rs
+++ b/tests/src/tests/mantle/sdp/ops.rs
@@ -22,6 +22,7 @@ use lb_core::{
         WithdrawMessage,
     },
 };
+use lb_groth16::Fr;
 use lb_key_management_system_service::keys::{Ed25519Key, Ed25519Signature, ZkKey};
 use lb_node::config::{
     RunConfig, blend::deployment::MinimumNetworkSize, cryptarchia::deployment::EpochConfig,
@@ -43,7 +44,6 @@ use logos_blockchain_tests::{
     },
     cucumber::defaults::E2E_ARTIFACTS_DIR,
 };
-use num_bigint::BigUint;
 use testing_framework_core::scenario::{DynError, StartNodeOptions};
 use tokio::time::{sleep, timeout};
 
@@ -91,7 +91,7 @@ async fn sdp_ops_e2e() {
     );
 
     let provider_signing_key = Ed25519Key::from_bytes(&[7u8; 32]);
-    let provider_zk_key = ZkKey::from(BigUint::from(7u64));
+    let provider_zk_key = ZkKey::from(Fr::from(7u64));
     let provider_id = ProviderId::try_from(provider_signing_key.public_key().to_bytes())
         .expect("provider signing key should yield a provider id");
     let zk_id = provider_zk_key.to_public_key();
diff --git a/tests/testing_framework/Cargo.toml b/tests/testing_framework/Cargo.toml
index f96d73af0..ca1e231dc 100644
--- a/tests/testing_framework/Cargo.toml
+++ b/tests/testing_framework/Cargo.toml
@@ -38,7 +38,6 @@ lb-node                          = { features = ["testing"], workspace = true }
 lb-time-service                  = { workspace = true }
 lb-tx-service                    = { workspace = true }
 lb-utils                         = { workspace = true }
-num-bigint                       = { workspace = true }
 rand                             = { workspace = true }
 reqwest                          = { features = ["json"], workspace = true }
 serde                            = { workspace = true }
diff --git a/tests/testing_framework/src/node/configs/wallet.rs b/tests/testing_framework/src/node/configs/wallet.rs
index 864bd4733..f2775c05e 100644
--- a/tests/testing_framework/src/node/configs/wallet.rs
+++ b/tests/testing_framework/src/node/configs/wallet.rs
@@ -2,8 +2,8 @@ use std::{collections::HashSet, num::NonZeroUsize};
 
 use hex::ToHex as _;
 use lb_core::codec::SerializeOp as _;
+use lb_groth16::fr_from_mod_bytes;
 use lb_key_management_system_service::keys::{ZkKey, ZkPublicKey};
-use num_bigint::BigUint;
 use rand::Rng as _;
 use thiserror::Error;
 
@@ -138,7 +138,7 @@ impl WalletAccount {
         seed[..2].copy_from_slice(b"wl");
         seed[2..10].copy_from_slice(&index.to_le_bytes());
 
-        let secret_key = ZkKey::from(BigUint::from_bytes_le(&seed));
+        let secret_key = ZkKey::new(fr_from_mod_bytes(&seed));
         Self::new(
             format!("wallet-user-{index}"),
             secret_key,
@@ -151,7 +151,7 @@ impl WalletAccount {
         let mut seed = [0u8; 32];
         rand::thread_rng().fill(&mut seed);
 
-        let secret_key = ZkKey::from(BigUint::from_bytes_le(&seed));
+        let secret_key = ZkKey::new(fr_from_mod_bytes(&seed));
         let index = u64::from_le_bytes(seed[..8].try_into().expect("seed has len 32"));
         Self::new(format!("wallet-r-user-{index}"), secret_key, 0, true)
     }
diff --git a/tools/blockchain-tools/src/genesis/distribution.rs b/tools/blockchain-tools/src/genesis/distribution.rs
index 9ffb0298f..498620aa3 100644
--- a/tools/blockchain-tools/src/genesis/distribution.rs
+++ b/tools/blockchain-tools/src/genesis/distribution.rs
@@ -124,12 +124,12 @@ where
 #[cfg(test)]
 mod tests {
     use lb_core::sdp::Locator;
-    use num_bigint::BigUint;
+    use lb_groth16::Fr;
 
     use super::*;
 
     fn mock_zk_pk(byte: u8) -> ZkPublicKey {
-        ZkPublicKey::from(BigUint::from(byte))
+        ZkPublicKey::from(Fr::from(byte))
     }
 
     fn mock_ed_pk(byte: u8) -> Ed25519PublicKey {
diff --git a/tools/config/Cargo.toml b/tools/config/Cargo.toml
index 326d333b5..29000c625 100644
--- a/tools/config/Cargo.toml
+++ b/tools/config/Cargo.toml
@@ -27,6 +27,5 @@ lb-libp2p                        = { workspace = true }
 lb-node                          = { workspace = true }
 lb-tracing                       = { workspace = true }
 lb-utils                         = { workspace = true }
-num-bigint                       = { workspace = true }
 rand                             = { workspace = true }
 time                             = { workspace = true }
diff --git a/tools/config/src/blend.rs b/tools/config/src/blend.rs
index 358126896..b5810984a 100644
--- a/tools/config/src/blend.rs
+++ b/tools/config/src/blend.rs
@@ -1,9 +1,9 @@
 use std::str::FromStr as _;
 
+use lb_groth16::fr_from_mod_bytes;
 use lb_key_management_system_service::keys::{Ed25519Key, ZkKey};
 use lb_libp2p::Multiaddr;
 use lb_node::config::blend::serde as blend;
-use num_bigint::BigUint;
 
 use crate::kms::key_id_for_preload_backend;
 
@@ -26,8 +26,7 @@ pub fn create_blend_configs_with_listening_host(
         .zip(ports)
         .map(|(id, port)| {
             let private_key = Ed25519Key::from_bytes(id);
-            let secret_zk_key =
-                ZkKey::from(BigUint::from_bytes_le(private_key.public_key().as_bytes()));
+            let secret_zk_key = ZkKey::new(fr_from_mod_bytes(private_key.public_key().as_bytes()));
             let mut base_config = blend::Config::with_required_values(blend::RequiredValues {
                 non_ephemeral_signing_key_id: key_id_for_preload_backend(
                     &private_key.clone().into(),
diff --git a/tools/config/src/consensus.rs b/tools/config/src/consensus.rs
index 1bcd0d708..2764a98e1 100644
--- a/tools/config/src/consensus.rs
+++ b/tools/config/src/consensus.rs
@@ -17,12 +17,11 @@ use lb_core::{
     },
     sdp::{DeclarationMessage, Locator, ProviderId, ServiceType},
 };
-use lb_groth16::{AdditiveGroup as _, CompressedGroth16Proof, Fr};
+use lb_groth16::{AdditiveGroup as _, CompressedGroth16Proof, Fr, fr_from_mod_bytes};
 use lb_key_management_system_service::keys::{
     Ed25519Key, Ed25519Signature, ZkKey, ZkPublicKey, ZkSignature,
 };
 use lb_node::{Hashable as _, SignedOps};
-use num_bigint::BigUint;
 
 use crate::unique::unique_test_context;
 
@@ -395,7 +394,7 @@ fn create_utxos(
 
     for &id in ids {
         let sk_data = derive_key_material(LEADER_KEY_PREFIX, &id);
-        let sk = ZkKey::from(BigUint::from_bytes_le(&sk_data));
+        let sk = ZkKey::new(fr_from_mod_bytes(&sk_data));
         let pk = sk.to_public_key();
         regular_note_keys.push(sk);
         utxos.push(Utxo {
@@ -406,7 +405,7 @@ fn create_utxos(
         output_index += 1;
 
         let sk_blend_data = derive_key_material(BLEND_KEY_PREFIX, &id);
-        let sk_blend = ZkKey::from(BigUint::from_bytes_le(&sk_blend_data));
+        let sk_blend = ZkKey::new(fr_from_mod_bytes(&sk_blend_data));
         let pk_blend = sk_blend.to_public_key();
         let note_blend = Note::new(BLEND_NOTE_VALUE, pk_blend);
         let utxo = Utxo {
@@ -425,7 +424,7 @@ fn create_utxos(
         output_index += 1;
 
         let sk_sdp_data = derive_key_material(SDP_KEY_PREFIX, &id);
-        let sk_sdp = ZkKey::from(BigUint::from_bytes_le(&sk_sdp_data));
+        let sk_sdp = ZkKey::new(fr_from_mod_bytes(&sk_sdp_data));
         let pk_sdp = sk_sdp.to_public_key();
         for sdp_note_index in 0..sdp_notes_per_node {
             let note_value = base_sdp_note_value + u64::from(sdp_note_index < sdp_value_remainder);
diff --git a/wallet/src/lib.rs b/wallet/src/lib.rs
index a6928c431..3f4ea235b 100644
--- a/wallet/src/lib.rs
+++ b/wallet/src/lib.rs
@@ -907,7 +907,7 @@ mod tests {
     use super::*;
 
     fn pk(v: u64) -> ZkPublicKey {
-        ZkPublicKey::from(BigUint::from(v))
+        ZkPublicKey::from(Fr::from(v))
     }
 
     fn tx_hash(v: u8) -> Hash {
diff --git a/zk/groth16/src/lib.rs b/zk/groth16/src/lib.rs
index f1c6a57c2..3c798f39c 100644
--- a/zk/groth16/src/lib.rs
+++ b/zk/groth16/src/lib.rs
@@ -57,7 +57,12 @@ pub struct FrFromBytesError {
 }
 
 pub fn fr_from_bytes(fr: &[u8]) -> Result<Fr, impl Error + use<>> {
-    let n = BigUint::from_bytes_le(fr);
+    fr_try_from_biguint(BigUint::from_bytes_le(fr))
+}
+
+/// Convert into `Fr` only when the value is below the modulus `p`; never
+/// reduces.
+pub fn fr_try_from_biguint(n: BigUint) -> Result<Fr, FrFromBytesError> {
     if n >= fr_modulus() {
         return Err(FrFromBytesError {
             parsed_bytes: n.to_string(),
diff --git a/zk/proofs/poc/src/wallet_inputs.rs b/zk/proofs/poc/src/wallet_inputs.rs
index 1b5d612a8..d3a25bcd3 100644
--- a/zk/proofs/poc/src/wallet_inputs.rs
+++ b/zk/proofs/poc/src/wallet_inputs.rs
@@ -1,3 +1,5 @@
+use core::fmt;
+
 use lb_groth16::{AdditiveGroup as _, Field as _, Fr, Groth16Input, Groth16InputDeser};
 use serde::Serialize;
 
@@ -10,12 +12,24 @@ pub struct PoCWalletInputs {
     voucher_merkle_path_and_selectors: [(Groth16Input, Groth16Input); VOUCHER_MERKLE_PATH_LEN],
 }
 
-#[derive(Clone, Debug)]
+#[derive(Clone)]
 pub struct PoCWalletInputsData {
     pub secret_voucher: Fr,
     pub voucher_merkle_path_and_selectors: VoucherPathAndSelector,
 }
 
+impl fmt::Debug for PoCWalletInputsData {
+    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
+        f.debug_struct("PoCWalletInputsData")
+            .field("secret_voucher", &"<redacted>")
+            .field(
+                "voucher_merkle_path_and_selectors",
+                &self.voucher_merkle_path_and_selectors,
+            )
+            .finish()
+    }
+}
+
 #[derive(Serialize)]
 pub struct PoCWalletInputsJson {
     secret_voucher: Groth16InputDeser,
@@ -63,3 +77,26 @@ impl From<PoCWalletInputsData> for PoCWalletInputs {
         }
     }
 }
+
+#[cfg(test)]
+mod tests {
+    use super::*;
+
+    #[test]
+    fn wallet_inputs_data_debug_redacts_secret_voucher() {
+        let secret = Fr::from(12_648_430u64);
+        let data = PoCWalletInputsData {
+            secret_voucher: secret,
+            voucher_merkle_path_and_selectors: [(Fr::ZERO, false); VOUCHER_MERKLE_PATH_LEN],
+        };
+        let rendered = format!("{data:?}");
+        assert!(rendered.contains("<redacted>"), "{rendered}");
+        for repr in [
+            format!("{secret:?}"),
+            secret.to_string(),
+            format!("{:?}", Groth16Input::new(secret)),
+        ] {
+            assert!(!rendered.contains(&repr), "{rendered} leaks {repr}");
+        }
+    }
+}
diff --git a/zk/proofs/pol/src/inputs.rs b/zk/proofs/pol/src/inputs.rs
index 6db1e15d5..d9e1c2354 100644
--- a/zk/proofs/pol/src/inputs.rs
+++ b/zk/proofs/pol/src/inputs.rs
@@ -1,3 +1,5 @@
+use core::fmt;
+
 use lb_groth16::{Fr, Groth16Input, Groth16InputDeser};
 use serde::{Deserialize, Serialize};
 
@@ -8,13 +10,22 @@ use crate::{
 };
 
 /// The inputs to the circuit prover.
-#[derive(Clone, Serialize, Debug)]
+#[derive(Clone, Serialize)]
 #[serde(into = "PolInputsJson", rename_all = "snake_case")]
 pub struct PolWitnessInputs {
     pub wallet: PolWalletInputs,
     pub chain: PolChainInputs,
 }
 
+impl fmt::Debug for PolWitnessInputs {
+    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
+        f.debug_struct("PolWitnessInputs")
+            .field("wallet", &self.wallet)
+            .field("chain", &self.chain)
+            .finish()
+    }
+}
+
 impl TryFrom<PolWitnessInputs> for lbc_pol_sys::PolWitnessInput<'_> {
     type Error = lbp_error::Error;
 
@@ -26,12 +37,21 @@ impl TryFrom<PolWitnessInputs> for lbc_pol_sys::PolWitnessInput<'_> {
     }
 }
 
-#[derive(Clone, Debug)]
+#[derive(Clone)]
 pub struct PolWitnessInputsData {
     pub wallet: PolWalletInputsData,
     pub chain: PolChainInputsData,
 }
 
+impl fmt::Debug for PolWitnessInputsData {
+    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
+        f.debug_struct("PolWitnessInputsData")
+            .field("wallet", &self.wallet)
+            .field("chain", &self.chain)
+            .finish()
+    }
+}
+
 impl PolWitnessInputsData {
     #[must_use]
     pub const fn from_chain_and_wallet_data(
@@ -164,3 +184,60 @@ impl PolVerifierInput {
         }
     }
 }
+
+#[cfg(test)]
+mod tests {
+    use lb_groth16::AdditiveGroup as _;
+
+    use super::*;
+    use crate::wallet_inputs::{AGED_NOTE_MERKLE_TREE_HEIGHT, LATEST_NOTE_MERKLE_TREE_HEIGHT};
+
+    const SECRET: u64 = 12_648_430;
+
+    fn witness_data(secret_key: Fr) -> PolWitnessInputsData {
+        let chain = PolChainInputsData {
+            slot_number: 0,
+            epoch_nonce: Fr::ZERO,
+            lottery_0: Fr::ZERO,
+            lottery_1: Fr::ZERO,
+            aged_root: Fr::ZERO,
+            latest_root: Fr::ZERO,
+            leader_pk: (Fr::ZERO, Fr::ZERO),
+        };
+        let wallet = PolWalletInputsData {
+            note_value: 1,
+            transaction_hash: Fr::ZERO,
+            output_number: 0,
+            aged_path: [Fr::ZERO; AGED_NOTE_MERKLE_TREE_HEIGHT],
+            aged_selectors: [false; AGED_NOTE_MERKLE_TREE_HEIGHT],
+            latest_path: [Fr::ZERO; LATEST_NOTE_MERKLE_TREE_HEIGHT],
+            latest_selectors: [false; LATEST_NOTE_MERKLE_TREE_HEIGHT],
+            secret_key,
+        };
+        PolWitnessInputsData::from_chain_and_wallet_data(chain, wallet)
+    }
+
+    fn assert_redacted(rendered: &str, secret: Fr) {
+        assert!(rendered.contains("<redacted>"), "{rendered}");
+        for repr in [
+            format!("{secret:?}"),
+            secret.to_string(),
+            format!("{:?}", Groth16Input::new(secret)),
+        ] {
+            assert!(!rendered.contains(&repr), "{rendered} leaks {repr}");
+        }
+    }
+
+    #[test]
+    fn witness_inputs_data_debug_redacts_secret_key() {
+        let secret = Fr::from(SECRET);
+        assert_redacted(&format!("{:?}", witness_data(secret)), secret);
+    }
+
+    #[test]
+    fn witness_inputs_debug_redacts_secret_key() {
+        let secret = Fr::from(SECRET);
+        let inputs = PolWitnessInputs::from(witness_data(secret));
+        assert_redacted(&format!("{inputs:?}"), secret);
+    }
+}
diff --git a/zk/proofs/pol/src/wallet_inputs.rs b/zk/proofs/pol/src/wallet_inputs.rs
index 11c074beb..dd4db479a 100644
--- a/zk/proofs/pol/src/wallet_inputs.rs
+++ b/zk/proofs/pol/src/wallet_inputs.rs
@@ -1,7 +1,12 @@
+use core::fmt;
+
 use lb_groth16::{AdditiveGroup as _, Field as _, Fr, Groth16Input, Groth16InputDeser};
 use num_bigint::BigUint;
 use serde::Serialize;
 
+/// Placeholder written by the `Debug` impls in place of the leader secret key.
+const REDACTED: &str = "<redacted>";
+
 // TODO: These consts should live in a common crate shared among circuits.
 pub const AGED_NOTE_MERKLE_TREE_HEIGHT: usize = 32;
 pub type AgedNotePath = [Fr; AGED_NOTE_MERKLE_TREE_HEIGHT];
@@ -11,7 +16,7 @@ pub type LatestNotePath = [Fr; LATEST_NOTE_MERKLE_TREE_HEIGHT];
 pub type LatestSelectorPath = [bool; LATEST_NOTE_MERKLE_TREE_HEIGHT];
 
 /// Public inputs of the POL cirmcom circuit as circuit field values.
-#[derive(Clone, Debug)]
+#[derive(Clone)]
 pub struct PolWalletInputs {
     note_value: Groth16Input,
     transaction_hash: Groth16Input,
@@ -23,8 +28,19 @@ pub struct PolWalletInputs {
     secret_key: Groth16Input,
 }
 
+impl fmt::Debug for PolWalletInputs {
+    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
+        f.debug_struct("PolWalletInputs")
+            .field("note_value", &self.note_value)
+            .field("transaction_hash", &self.transaction_hash)
+            .field("output_number", &self.output_number)
+            .field("secret_key", &REDACTED)
+            .finish_non_exhaustive()
+    }
+}
+
 /// Private inputs of the POL cirmcom circuit to be provided by the wallet.
-#[derive(Clone, Debug)]
+#[derive(Clone)]
 pub struct PolWalletInputsData {
     pub note_value: u64,
     pub transaction_hash: Fr,
@@ -36,6 +52,17 @@ pub struct PolWalletInputsData {
     pub secret_key: Fr,
 }
 
+impl fmt::Debug for PolWalletInputsData {
+    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
+        f.debug_struct("PolWalletInputsData")
+            .field("note_value", &self.note_value)
+            .field("transaction_hash", &self.transaction_hash)
+            .field("output_number", &self.output_number)
+            .field("secret_key", &REDACTED)
+            .finish_non_exhaustive()
+    }
+}
+
 #[derive(Serialize)]
 pub struct PolWalletInputsJson {
     #[serde(rename = "v")]
@@ -107,3 +134,47 @@ impl From<PolWalletInputsData> for PolWalletInputs {
         }
     }
 }
+
+#[cfg(test)]
+mod tests {
+    use super::*;
+
+    const SECRET: u64 = 12_648_430;
+
+    fn wallet_data(secret_key: Fr) -> PolWalletInputsData {
+        PolWalletInputsData {
+            note_value: 1,
+            transaction_hash: Fr::ZERO,
+            output_number: 0,
+            aged_path: [Fr::ZERO; AGED_NOTE_MERKLE_TREE_HEIGHT],
+            aged_selectors: [false; AGED_NOTE_MERKLE_TREE_HEIGHT],
+            latest_path: [Fr::ZERO; LATEST_NOTE_MERKLE_TREE_HEIGHT],
+            latest_selectors: [false; LATEST_NOTE_MERKLE_TREE_HEIGHT],
+            secret_key,
+        }
+    }
+
+    fn assert_redacted(rendered: &str, secret: Fr) {
+        assert!(rendered.contains(REDACTED), "{rendered}");
+        for repr in [
+            format!("{secret:?}"),
+            secret.to_string(),
+            format!("{:?}", Groth16Input::new(secret)),
+        ] {
+            assert!(!rendered.contains(&repr), "{rendered} leaks {repr}");
+        }
+    }
+
+    #[test]
+    fn wallet_inputs_data_debug_redacts_secret_key() {
+        let secret = Fr::from(SECRET);
+        assert_redacted(&format!("{:?}", wallet_data(secret)), secret);
+    }
+
+    #[test]
+    fn wallet_inputs_debug_redacts_secret_key() {
+        let secret = Fr::from(SECRET);
+        let inputs = PolWalletInputs::from(wallet_data(secret));
+        assert_redacted(&format!("{inputs:?}"), secret);
+    }
+}
diff --git a/zone-sdk/src/sequencer/actor.rs b/zone-sdk/src/sequencer/actor.rs
index f99c98a03..0f71763c9 100644
--- a/zone-sdk/src/sequencer/actor.rs
+++ b/zone-sdk/src/sequencer/actor.rs
@@ -692,8 +692,8 @@ mod tests {
             transactions::{OpProofs, Ops, states::Unverified},
         },
     };
+    use lb_groth16::Fr;
     use lb_key_management_system_service::keys::{Ed25519Key, ZkKey};
-    use num_bigint::BigUint;
     use rand::{RngCore as _, thread_rng};
     use tokio::sync::{mpsc, watch};
 
@@ -714,7 +714,7 @@ mod tests {
     pub fn utxo_with_sk() -> (ZkKey, Utxo) {
         let mut op_id = [0u8; 32];
         thread_rng().fill_bytes(&mut op_id);
-        let zk_sk = ZkKey::from(BigUint::from(0u64));
+        let zk_sk = ZkKey::from(Fr::from(0u64));
         let utxo = Utxo {
             op_id,
             output_index: 0,
```
