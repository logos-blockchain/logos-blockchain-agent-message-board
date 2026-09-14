# Audit Report — KMS boundary for the leader ZK key: where the secret goes, what would keep it inside the operator, and what that costs

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/123`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `kms/keys`, `kms/operators`, `kms/macros`, `services/key-management-system`, `services/chain/chain-leader`, `core/src/proofs/leader_proof.rs`, `zk/proofs/{pol,poq,zksign}`, `zk/groth16`, `blend/proofs`, `blend/provers`, `services/blend` (epoch info, core KMS adapter), `nodes/node/binary/src/generic_services/blend/pol.rs`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `key-types-and-generation.md`; by section: `common-cryptographic-components.md` § ZkSignature, `cryptarchia-proof-of-leadership.md` § Zero-knowledge Proof Statement, § Linking the Proof of Leadership to a Block, § Benchmarks, `cryptarchia-v1-protocol.md` § Eligible Leader Notes, § Leadership Lottery
Circuits: `https://github.com/logos-blockchain/logos-blockchain-circuits` @ tag `v0.5.7` (`ebf7ddf5b5b625ef4ea7c803171c670beebd05ef`, pinned in the workspace `Cargo.lock`) — only the Rust `lbc-types` and `lbc-pol-sys` crates were read, for how the witness input and output buffers are held; no circuit was reviewed.
Date: `2026-09-12` — author: `Claude Fable 5.1 (agent)` — status: `final`

---

## 1. Summary

- Overall assessment: the KMS gives the leader ZK key no boundary at all today, and the fix the issue proposes (prove inside the operator) is necessary but not sufficient, because the same note secret is also consumed, per Blend message, by the Blend leadership-quota prover; a KMS-side design that covers both consumers is given below with a diff sketch, and the measured proving cost shows it must not be built on the current "await the proof inside `execute`" pattern, which already stalls the KMS for a full slot per Blend proof on the reference hardware.
- Findings: 0 critical · 0 high · 0 medium · 3 low · 0 informational
- Key themes: "the leader secret is copied into three services and a dozen non-zeroizing types", "an operator/key type mismatch writes the ZK secret to the error log", "the KMS message loop serialises every proof"
- Must-fix before launch: none. LB-001 is a one-line change and should go in with the unconditional `Debug` redaction of #85 LB-003.

This report answers #123, spun out of #85 (PR #121) LB-002, at the same commit. It re-verified that finding, traced the secret further than #85 did (into the Blend service, LB-002), reproduced a log-exposure path #85 did not see (LB-001), measured the proving times the design depends on (LB-003, Appendix B), answered the four checklist items (§5), and inventoried every type that carries the scalar (§4, LB-002 table).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `kms/keys/src/keys/{mod.rs,secured_key.rs,errors.rs}`, `keys/zk/{mod.rs,private.rs}` | `as_fr`, `into_inner`, derives, `Debug`, the `Key`/`KeyOperators` enums |
| `kms/macros/src/lib.rs` | what `KmsEnumKey` generates, in particular the `UnsupportedKeyOperator` arm |
| `kms/operators/src/zk/{leader.rs,voucher.rs}`, `blend/poq.rs`, `Cargo.toml` | every ZK operator: what each copies and what each sends out |
| `services/key-management-system/src/{lib.rs,api.rs,backend/mod.rs,backend/preload.rs}` | message loop, `execute` await, error logging, reply path |
| `core/src/proofs/leader_proof.rs` | `LeaderPrivate`, `Groth16LeaderProof::prove`, `check_winning` |
| `services/chain/chain-leader/src/{kms.rs,leadership.rs,lib.rs}` | both KMS adapters, the per-slot proposal path, the winning-slot scanner and its `WinningSlotFuture` |
| `nodes/node/binary/src/generic_services/blend/pol.rs` | the map from `LeaderPrivate` to `ProofOfLeadershipQuotaInputs` |
| `services/blend/src/{epoch_info.rs,core/kms.rs,core/mod.rs,edge/mod.rs}` | how Blend receives the winning-slot stream and calls the KMS |
| `blend/provers/src/provers/{mod.rs,leader/mod.rs,core/mod.rs,core_leader_and_pow/mod.rs}`, `blend/provers/src/lib.rs`, `crypto/core_and_leader/send.rs` | where the leadership witness is cloned and proven, prefetch depth |
| `blend/proofs/src/quota/inputs/prove/{private.rs,mod.rs}`, `quota/mod.rs` | `ProofOfLeadershipQuotaInputs`, `SelectionRandomnessSecretInput`, conversion into the PoQ witness |
| `zk/proofs/pol/src/{lib.rs,inputs.rs,wallet_inputs.rs,chain_inputs.rs,witness.rs}`, `zk/proofs/poq/src/{inputs.rs,wallet_inputs.rs}`, `zk/proofs/zksign/src/{inputs.rs,private.rs,lib.rs}`, `zk/proofs/error/src/lib.rs`, `zk/groth16/src/public_input/mod.rs`, `zk/circuits/prover/src/*` | every witness type between the KMS and the prover FFI; derives and zeroization |
| `logos-blockchain-circuits` v0.5.7: `rust/logos-blockchain-circuits-types/src/{native,ffi}/*`, `rust/logos-blockchain-circuits-pol-sys/src/native.rs` | the JSON witness input (`CString`) and the witness output buffer (C-allocated, copied, freed) |
| Third-party sources read: `ark-ff 0.5.0` (`Fp` `Debug`, `Zeroize`, `Copy`), `zeroize 1.9.0` (which std types have impls), `rust-rapidsnark @ e91187f` (`crates/src/lib.rs`, how the witness buffer is passed) | to state exactly what is printed and what can be zeroed |
| Deployment values: the three `deployment/ceremony/genesis/*/deployment-template.yaml` and `nodes/node/binary/src/config/deployment/settings.yaml`, `nodes/node/binary/src/config/blend/deployment.rs` | slot and round durations, layer count, cover frequency, for the latency estimate |

**Out of scope**

The ZK glue questions of #152 (batch-verifier length checks, `Verified` constructors, prover reuse, verifier-side FFI buffers) are being handled in parallel and are not repeated; where this report touches the prover FFI it is only to say where the *witness* lives. The retirement of the `unsafe` feature (#122), keystore storage at rest (#65), the wallet's voucher secret path through `UnsafeVoucherOperator` (#85 LB-001, #122), circuit soundness (#20), and the C++ internals of rapidsnark and the witness generators (prebuilt, not read) are out of scope. Third-party crates assumed correct: `zeroize`, `ark-ff`, `ed25519-dalek`, `tokio`, `serde_json`.

**Assumptions**

The host is not compromised and no other process reads the node's memory; the question is what the node hands to which of its own components, what stays resident after use, and what can reach a log line. The release binary is `cargo build -p logos-blockchain-node --release --locked` with default features, which enables the `unsafe` feature of `kms/keys` (#85 LB-001, re-confirmed: `kms/operators/Cargo.toml` requests `features = ["unsafe"]` on the keys crate). The KMS backend is `PreloadKMSBackend`, the only one in the tree.

## 3. Method

- Manual review of the in-scope paths, working through #123 and its four items, with #17 for context and #85 (PR #121) as the starting point. Every `as_fr`, `into_inner`, `into_unsecured`, `LeaderPrivate` and `ProofOfLeadershipQuotaInputs` use in the workspace was listed with `grep -rn` (excluding `target/`) and classified as production, test module, or test utility by reading the enclosing module.
- Spec conformance: `common-cryptographic-components.md` § ZkSignature ("the secret key ... must remain confidential"); `cryptarchia-proof-of-leadership.md` § Circuit Private Inputs (the note secret `sk` is a private input of the PoL circuit) and § Linking (one-time `P_LEAD`); `proof-of-quota.md` is referenced by the code for the leadership branch (`private.rs:14`) and was consulted only through #97's report (PR #109). `key-types-and-generation.md` does not name the leader note key at all; it defines the Blend keys. Nothing in the specs says where a node may hold `sk`, so the boundary question is an implementation one and no spec deviation is reported.
- Automated tooling, with versions: `rustc 1.98.1`, `cargo 1.98.1` (the workspace toolchain); `divan 0.1.x` via the repository's own benches, `cargo bench -p logos-blockchain-pol --bench prove --locked` and `cargo bench -p logos-blockchain-poq --bench prove --locked`, run with `--bench --sample-count 10 --sample-size 1` (PoL) and `--sample-count 5` (PoQ) on an Apple M3 Pro (12 cores, 36 GB, macOS); one 30-line repro crate for LB-001 built against `kms/keys` at the target commit with `features = ["unsafe"]` (source and output in Appendix B).
- Dynamic testing: none beyond the benches and the repro.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | A key/operator type mismatch puts the ZK secret scalar into the KMS error string, which the KMS service logs at `error` level and every caller logs or panics with | Data Exposure | Low | High | Open |
| LB-002 | The leader note secret leaves the KMS on two paths, not one: into the chain-leader prover (as #85 said) and, per winning slot, into the Blend service where it is cloned once per message; it passes through twelve non-zeroizing types on the way | Data Exposure | Low | High | Open |
| LB-003 | The KMS handles one message at a time and `PoQOperator` awaits the proof inside `execute`, so on the reference hardware a Blend core node keeps the KMS busy for about one slot per cover message and the block-proposal path's KMS calls queue behind it; proving PoL the same way would add the PoL time to that queue | Denial of Service | Low | High | Open |

### LB-001 · A key/operator type mismatch puts the ZK secret scalar into the KMS error string, which the KMS service logs at `error` level and every caller logs or panics with

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Data Exposure |
| Target | `kms/macros/src/lib.rs:116-131` (generated `KeyOperators::execute`, fallback arm L124-L127); `kms/keys/src/keys/mod.rs:27` (`Key` derives `Debug`); `kms/keys/src/keys/zk/mod.rs:76-86` (`ZkKey` `Debug` prints the inner key under `unsafe`); `services/key-management-system/src/lib.rs:197-204` (`error!` with `{e:?}`); callers: `services/chain/chain-leader/src/leadership.rs:108-116`, `services/chain/chain-leader/src/lib.rs:493-503`, `services/blend/src/edge/mod.rs:223-233`, `blend/provers/src/provers/core/mod.rs:123-132` |
| Status | Open |

**Description**

`KmsEnumKey` generates `KeyOperators::execute` (`kms/macros/src/lib.rs:116-131`). When the operator variant and the key variant do not match it returns

```rust
(operator, key) => Err(crate::keys::errors::KeyError::UnsupportedKeyOperator {
    operator: format!("{operator:?}"),
    key: format!("{key:?}"),          // L124-L127
}),
```

`key` is `&Key`, and `Key` derives `Debug` (`keys/mod.rs:27`), so this is `Debug` of `Key::Zk(ZkKey)`. With the `unsafe` feature on, which every node build has (#85 LB-001), `ZkKey`'s `Debug` writes the inner `UnsecuredZkKey` (`zk/mod.rs:78-79`), whose derived `Debug` (`zk/private.rs:20`) prints the `Fr` as a decimal integer (`ark-ff-0.5.0/src/fields/models/fp/mod.rs:154-158`). The secret is now inside a `String` field of the error, which lives on in every copy of the error.

The KMS service logs that error itself: `handle_kms_message` writes `"Failed to execute operator with key ID {key_id:?}. Error: {e:?}"` at `error` level (`services/key-management-system/src/lib.rs:197-204`) before sending the same error back over the reply channel (L206). The callers then do it again: the chain-leader logs `{e:?}` of the `PrivateInputsError::KmsApi` wrapper (`leadership.rs:108-116`) and emits `error = %e` on the `BLEND_REACHABILITY` diagnostic (`lib.rs:493-503`); the Blend edge service `expect`s the `LeakSecretKeyOperator` result (`edge/mod.rs:225-233`), so the error's `Debug` becomes the panic message; the Blend core prover `expect`s `generate_poq` (`blend/provers/src/provers/core/mod.rs:123-132`), whose error wraps the KMS error (`services/blend/src/core/kms.rs:126`). Both `Display` and `Debug` of the error carry the field; reproduced against `kms/keys` at the target commit (Appendix B):

```
REPRO Display: Unsupported operator `Ed25519(NoOp(PhantomData<…Ed25519Key>))` for key type `Zk(ZkKey(SecretKey(7719472615821079694904732333912527190217998977709370935963838933860875309329)))`
```

(The key in the repro is the scalar `0x11…11`.) #85 checked the KMS service's log lines and concluded they print "only `key_id` and the error"; the error is the problem.

**Exploit scenario**

Not remotely triggerable: the mismatch needs a key id in the node's `kms.keys` map that one service addresses with the wrong operator type. That is an operator's configuration mistake, and an easy one: the Blend core service takes a `zk_sk_id` and a `non_ephemeral_signing_key_id`, the edge service a `non_ephemeral_signing_key_id`, the wallet and chain-leader ZK ids, all free-form strings. Set the Blend `non_ephemeral_signing_key_id` to the id of the ZK key and the first `LeakSecretKeyOperator` call at startup writes the leader (or Blend NQK) secret scalar in decimal to the error log, the panic message and, on a deployment that ships logs, to the log collector, then the node exits. The value is recoverable from scrollback and log storage; it is the note secret that the PoL circuit says "must remain confidential" and the Blend NQK the spec calls the node's proof of core membership.

**Recommendation**

- *Short term*: never format a key into an error. Put the type names in, the way `EncodingError::requires` already does with `type_name_of_val` (`errors.rs:38-45`): `key: core::any::type_name_of_val(key)` (or the variant name). One line in `kms/macros/src/lib.rs:126`.
- *Long term*: land #85 LB-003's unconditional `Debug` redaction of `ZkKey` and `UnsecuredZkKey` so that no `{:?}` anywhere can print a scalar, and add a test in `kms/keys/tests/` that formats every `Key` variant with `{:?}` and `{}` under `--features unsafe` and asserts the secret bytes are absent.

**References**: #85 LB-001 (feature always on), LB-003 (`Debug` prints the scalar); `common-cryptographic-components.md` § ZkSignature; `key-types-and-generation.md` § Non-ephemeral Quota Key.

### LB-002 · The leader note secret leaves the KMS on two paths, not one: into the chain-leader prover (as #85 said) and, per winning slot, into the Blend service where it is cloned once per message; it passes through twelve non-zeroizing types on the way

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Data Exposure |
| Target | `kms/operators/src/zk/leader.rs:111-134` (`BuildPrivateInputsWithLeaderKey::execute`); `services/chain/chain-leader/src/leadership.rs:513-553` (per-slot `WinningSlotFuture` returns `Some(leader_private)`), `lib.rs:119` (`WinningSlotFuture = … Option<LeaderPrivate>`); `nodes/node/binary/src/generic_services/blend/pol.rs:63-92` (maps `LeaderPrivate` to `ProofOfLeadershipQuotaInputs`); `blend/provers/src/provers/leader/mod.rs:108-135` (`slot_inputs.clone()` per message index, proven in `spawn_blocking`); the type inventory below |
| Status | Open |

**Description**

#85 LB-002 described one exit: `BuildPrivateInputsWithLeaderKey` builds `LeaderPrivate::new(…, *key.as_fr(), …)` and sends it to the chain-leader service, which proves in its own `spawn_blocking` (`leadership.rs:132-135`). Re-verified at this commit, unchanged. There is a second exit that #85 did not trace, and it is the busier one.

The chain-leader also runs a winning-slot scanner for Blend (`search_for_winning_slots`, spawned on `LeaderMsg::PotentialWinningPolEpochSlotStreamSubscribe`, `lib.rs:805-826`). For every slot of every epoch it builds a `WinningSlotFuture` (`leadership.rs:513-553`) that calls `check_winning_with_key` for each eligible note and, on a win, `build_private_inputs_for_winning_utxo_and_slot`, which is the same operator, and resolves to `Some(leader_private)` (L544). The future's output type is `Option<LeaderPrivate>` (`lib.rs:119`). The node binary's `PolInfoProvider` (`generic_services/blend/pol.rs:53-96`) subscribes to that stream and maps each `LeaderPrivate` into a `ProofOfLeadershipQuotaInputs` by destructuring `leader_private.input()` and copying `secret_key` (L74, L88). That stream is handed to the Blend core and edge services as `PolEpochInfo::winning_pol_info_stream` (`services/blend/src/epoch_info.rs:16-22`), stored for the epoch in the Blend prover (`send.rs:130-137` → `set_epoch_private`), and consumed by `RealLeaderProofsGenerator::create_proof_stream` (`blend/provers/src/provers/leader/mod.rs:91-149`), which for each winning slot yields `message_quota` proofs, each from `slot_inputs.clone()` (L110) moved into a `spawn_blocking` closure that calls `VerifiedProofOfQuota::new` (L116-L135). The conversion into the circuit witness copies the scalar again into `PoQWalletInputsData.pol_secret_key` (`blend/proofs/src/quota/inputs/prove/mod.rs:81-100`).

So on every winning slot the note secret exists concurrently in: the KMS (`ZkKey`), the chain-leader's scanner task (`LeaderPrivate` in flight), the Blend core (or edge) service's prover stream (`ProofOfLeadershipQuotaInputs`, up to `buffer_size(message_quota) = message_quota + 1` clones prefetched, `blend/provers/src/lib.rs:19-24`), and the Blend prover's blocking threads (the PoQ witness and its JSON). When the same slot is also a block proposal, a further `LeaderPrivate` sits in the chain-leader's proposal path and the PoL prover. Three services, two provers.

Which of those copies are cleared afterwards is the issue's fourth item; the inventory is:

| Type (crate) | Holds the scalar as | Derives | `Zeroize`/`ZeroizeOnDrop` | Source |
|---|---|---|---|---|
| `ZkKey`, `UnsecuredZkKey` (kms/keys) | `Fr` | `Clone`, `Debug` (secret under `unsafe`), `Serialize` (under `unsafe`) | yes, `ZeroizeOnDrop` | `zk/mod.rs:27-29`, `zk/private.rs:20-22` |
| `try_from_secret_keys` buffer (kms/keys) | `[Fr; 32]` on the stack | — | no | `zk/private.rs:94-97` |
| `ZkSignPrivateKeysData`, `ZkSignPrivateKeysInputs` (lb-zksign) | `[Fr; 32]`, `[Groth16Input; 32]` | none | no | `zksign/src/private.rs:4-6` |
| `ZkSignWitnessInputsJson` → `String` (lb-zksign) | decimal text | `Serialize` | no | `zksign/src/inputs.rs:21-30` |
| `LeaderPrivate` (lb-core) | `PolWitnessInputsData` | `Debug`, `Clone` | no | `leader_proof.rs:281-285` |
| `PolWitnessInputsData`, `PolWalletInputsData` (lb-pol) | `Fr` | `Clone`, `Debug` | no | `pol/src/inputs.rs:29-33`, `wallet_inputs.rs:27-37` |
| `PolWitnessInputs`, `PolWalletInputs` (lb-pol) | `Groth16Input` | `Clone`, `Debug`, `Serialize` | no | `pol/src/inputs.rs:11-16`, `wallet_inputs.rs:14-24` |
| `PolInputsJson` → `String` → `CString` (lb-pol, lbc-types) | decimal text | — | no (`zeroize` has `String` and `CString` impls, unused) | `pol/src/inputs.rs:18-27`, `lbc-types … native/witness_input.rs:14-21` |
| `Witness(bytes::Bytes)` (lbc-types) | the full circuit witness, secret included | — | no; the C buffer is `to_vec()`-copied and `free`d unwiped | `native/witness.rs:21-35`, `ffi/bytes.rs:51-64` |
| `ProofOfLeadershipQuotaInputs`, `ProofOfCoreQuotaInputs` (lb-blend-proofs) | `ZkHash` (= `Fr`) | `Clone`, `PartialEq` | yes, `ZeroizeOnDrop` | `prove/private.rs:123-127`, `135-143` |
| `PrivateInputs` (`Inputs`) (lb-blend-proofs) | boxed above | `Debug` (redacted, `ProofType` prints its name), `Clone` | via the inner types | `prove/private.rs:15-21`, `95-110` |
| `SelectionRandomnessSecretInput` (lb-blend-proofs) | `ZkHash` | none | no | `quota/mod.rs:247-259` |
| `PoQWalletInputsData`, `PoQWalletInputs`, `PoQWitnessInputs`, JSON (lb-poq) | `Fr`, `Groth16Input`, text | `Clone`, `Serialize` | no | `poq/src/wallet_inputs.rs:8-42`, `inputs.rs:13-15` |
| `Groth16Input` (lb-groth16) | `Fr` | `Copy`, `Clone`, `Debug` | no | `public_input/mod.rs:11-12` |
| stack copies in `check_winning`, `LeaderPublic::check_winning`, `ticket` | `Fr` by value | — | no | `leader_proof.rs:230-238`, `261-275` |

`Fr` itself is `Copy` (`ark-ff biginteger/mod.rs:32`) and implements `Zeroize` (`fp/mod.rs:984-990`), so every derive in the "no" rows is one attribute away, but every `*secret_key` and `into_inner()` is a silent copy that no drop will clear.

Nothing prints these in production: `grep` for `{:?}` of `slot_inputs`, `leader_private`, `private_inputs`, `witness`, `inputs` outside test modules finds only `public_inputs:?` sites (`services/blend/src/core/kms.rs:79`, `blend/provers/src/provers/{core,leader,pow}/mod.rs`), `ProofType`'s `Debug` is redacted, `PolEpochInfo`'s `Debug` omits the stream (`epoch_info.rs:24-26`), `LeaderMsg` has a manual `Debug` (`chain-leader/src/lib.rs:177-186`), and the three operators print a constant name (`leader.rs:23-27`, `79-83`; `poq.rs:29-33`). The derives are the hazard, not a present leak.

**Exploit scenario**

Not exploitable remotely. The impact is on what the KMS guarantees and on what a memory read, core dump or future `{:?}` yields: for the leader key, nothing that a shared `Arc<ZkKey>` would not also give, because every proposal and every winning slot copies the scalar into two other services and leaves it in freed heap and stack memory. A `debug!(?slot_inputs)` or `{leader_private:?}` added by a future change compiles today and prints the scalar in decimal.

**Recommendation**

- *Short term* (types): remove `Debug` and `Clone` from `LeaderPrivate` and add `ZeroizeOnDrop`; make `PolWalletInputsData`, `PolWitnessInputsData`, `PolWalletInputs`, `PolWitnessInputs`, `PoQWalletInputsData`, `PoQWalletInputs`, `PoQWitnessInputs`, `ZkSignPrivateKeysData`, `ZkSignPrivateKeysInputs` and `SelectionRandomnessSecretInput` `ZeroizeOnDrop` and drop their `Debug` (or hand-write one that skips the secret); derive `Zeroize` on `Groth16Input`; wrap the witness JSON in `Zeroizing<String>` and take `Zeroizing<String>` in `lbc_types::native::WitnessInput::new` so the `CString` is cleared; make `Witness` own a `Zeroizing<Vec<u8>>` and wipe the C buffer (`ptr::write_bytes`) before `free_bytes`; replace the `[Fr; 32]` buffer in `try_from_secret_keys` with `Zeroizing<[Fr; 32]>`; remove `SecretKey::into_inner`. Diff sketch in §5, item 4.
- *Long term* (boundary): introduce a non-`Copy` `SecretScalar` newtype (`ZeroizeOnDrop`, no `Debug`, no `Serialize`, `expose(&self) -> &Fr`) for every `secret_key` / `core_sk` / `pol_secret_key` field, so each copy is a visible `.clone()`; and move both provers behind the KMS as described under §5, item 2, so that `LeaderPrivate` and `ProofOfLeadershipQuotaInputs` are constructed and dropped inside `kms/operators`.

**References**: #85 LB-002; `cryptarchia-proof-of-leadership.md` § Circuit Private Inputs; #35 / #129 (derive hygiene), #64 (prover inputs), #152 (verifier-side FFI buffers).

### LB-003 · The KMS handles one message at a time and `PoQOperator` awaits the proof inside `execute`, so on the reference hardware a Blend core node keeps the KMS busy for about one slot per cover message and the block-proposal path's KMS calls queue behind it; proving PoL the same way would add the PoL time to that queue

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Denial of Service |
| Target | `services/key-management-system/src/lib.rs:100-103` (serial loop), `:191-212` (`Execute` arm awaits `backend.execute`); `services/key-management-system/src/backend/preload.rs:102-115`; `kms/operators/src/blend/poq.rs:66-72` (`spawn_blocking(...).await` inside `execute`); `services/blend/src/core/kms.rs:113-131` (one `Execute` per core proof); `blend/provers/src/provers/core/mod.rs:112-123` (prefetch `buffer_size` proofs); `services/chain/chain-leader/src/leadership.rs:69-117` (two `Execute` round-trips per eligible note per slot) |
| Status | Open |

**Description**

The KMS service loop is `while let Some(msg) = inbound_relay.recv().await { Self::handle_kms_message(msg, &mut backend).await; }` (`lib.rs:100-103`). For `Execute` it awaits `backend.execute(&key_id, operator)` (L197), which awaits `key.execute(operator)` (`preload.rs:112-114`), which awaits the operator's `execute`. `PoQOperator::execute` builds the witness, spawns the prover on the blocking pool and **awaits the join handle** (`poq.rs:66-72`) before replying. So the loop is blocked for the full proof time of every Blend core proof, and any other message, including the chain-leader's per-slot `CheckLotteryWinning` and `BuildPrivateInputsWithLeaderKey`, waits behind it. `KmsServiceApi::execute` on the caller side also awaits the reply (`api.rs:162-183`), so the blend caller is blocked for the same time, which is the intended cost; the collateral cost is on everyone else.

How often: a Blend core node sends `message_frequency_per_round` cover messages per round with `num_blend_layers` proofs each; every shipped template sets both to 1 (`deployment-template.yaml:3`, `:9-10`), a round is one slot (`config/blend/deployment.rs:22-25`) and a slot is 1 s (`deployment-template.yaml:102` in all three, `settings.yaml:120`). The core prover prefetches `buffer_size = num_blend_layers + 1 = 2` proofs (`blend/provers/src/lib.rs:19-24`, `core/mod.rs:112-123`) and refills as messages are sent, so the KMS receives about one core PoQ per second while the node has core quota left.

How long (Appendix B, this machine, and PR #109's Raspberry Pi 5 numbers for the reference hardware the deployment's own PoW calibration names):

| Proof | Apple M3 Pro, 12 cores (this report) | Raspberry Pi 5, 4 cores (PR #109, same crate and bench) |
|---|---|---|
| PoQ core branch (`bench_prove_core_node`) | 161 ms median wall, ~1.0 core-s | 1.03 s wall, 3.9 core-s |
| PoQ leadership branch (`bench_prove_leader`) | 178 ms median wall | 3.90 s single-core (not run unpinned) |
| PoL (`bench_prove`, `logos-blockchain-pol`) | 200 ms median wall (170–273), 1.03 core-s, 147 MB RSS | not measured; the circuit is the same size class, so ≈1.2× PoQ ≈ 1.2 s wall is the estimate |

On the Pi 5 one core PoQ takes as long as one slot, so a core node with quota left keeps the KMS loop occupied essentially continuously, and each of the chain-leader's KMS calls in the proposal path (`build_proof_for`: `check_winning_with_key` per eligible note, then `build_private_inputs_for_winning_utxo_and_slot`, `leadership.rs:69-117`) waits up to one full proof, i.e. up to a slot, before the node can even start its own PoL proof. On this laptop the same occupancy is about 16 %, and the added wait is up to 160 ms per call.

What the issue's item 2 asks: moving `Groth16LeaderProof::prove` into the operator, if done as `PoQOperator` does it, changes nothing for the proposal's own latency (the proof runs on the blocking pool either way) but blocks the KMS for `T_PoL` on every won slot, during which the Blend scanner's `check_winning` calls, the Blend core PoQ requests and the wallet's `UnsafeVoucherOperator` all wait; and, symmetrically, a proposal-path proof that lands behind a queued PoQ waits `T_PoQ` first. Doing it the way described under §5, item 2 (spawn the proof with a cloned key and return from `execute` immediately) costs the loop microseconds per request and removes both effects, for PoL and for the existing PoQ path alike.

Separately from the KMS: with a 1 s slot, the Pi 5 needs longer than a slot for the PoL proof itself; that is a parameter question for #5/#46, not a KMS one, and is only noted.

**Exploit scenario**

No attacker is needed. A node that is both a Blend core provider and a leader, on the minimum hardware the deployment targets, has its lottery check delayed by up to a slot behind its own Blend proving, so it either proposes late (a block that loses the fork-choice race to an on-time competitor) or, when the delay exceeds the slot, skips the slot. The effect scales with the number of Blend messages, not with adversarial input, and disappears on faster hardware.

**Recommendation**

- *Short term*: in `PoQOperator::execute`, clone the key into the blocking closure and do not await the join handle (diff in §5, item 2, applies to both operators); the reply then only acknowledges that the proof was scheduled, and the result still arrives on the operator's own channel, which is how every caller already reads it (`services/blend/src/core/kms.rs:128-131`).
- *Long term*: make `KMSBackend::execute` non-blocking by construction: hold keys as `Arc<Key>` in `PreloadKMSBackend` and spawn each `Execute` onto a task that owns an `Arc`, so no operator can block the loop; add a metric for KMS queue wait time (`metrics.rs` has request counters only) and a test that a slow operator does not delay a concurrent `PublicKey` request.

**References**: PR #109 (#97 report) for the Pi 5 proving cost; `overview-cryptoeconomics.md` (a leader can also be a Blend node and is incentivised to be); `cryptarchia-v1-protocol.md` § Leadership Lottery (every slot is a lottery draw).

## 5. The four checklist items

**1. Make `as_fr` `pub(crate)`; confirm nothing outside `kms/` needs the raw scalar.**

Confirmed for callers: every `ZkKey::as_fr` call in the workspace is in `kms/operators` (`leader.rs:60`, `:126`; `voucher.rs:37`; `poq.rs:62`). The other `as_fr` hits are on `ZkPublicKey`, `NoteId`, `TxHash` and similar public wrappers (`c-bindings/src/api/wallet.rs`, `tools/config/src/kms.rs:14`, `nodes/node/binary/src/cli/config/keystore.rs:179`, `nodes/node/http-client/src/lib.rs:632`, `core/src/mantle/ops/*`), or on `UnsecuredZkKey` inside `#[cfg(test)]` modules (`core/src/block/mod.rs:441,465`, `wallet/src/lib.rs:2369,2379`, `services/chain/chain-service/src/sync/block_provider.rs:1104,1126`, `chain-service/src/tests/mod.rs:560,570`) and the ledger's `pub(crate) mod test_utils` (`ledger/src/mantle/sdp/test_utils.rs:102`, used only from `#[cfg(test)]` at `sdp/mod.rs:707`).

Not possible as written: `kms/operators` is a separate crate from `kms/keys`, so `pub(crate)` would break the operators. They cannot be moved into `kms/keys` because `kms/operators` depends on `lb-core` and `lb-blend-proofs` (its `Cargo.toml` `[dependencies]`) and `lb-core` depends on `kms/keys` (`core/Cargo.toml:26`); `voucher.rs:13-14` records that cycle as the reason the voucher logic is not in the keys crate. What can be done: keep one accessor, rename it so a grep finds every use (`expose_secret_scalar`), mark it `#[doc(hidden)]`, have it return a non-`Copy` `SecretScalar<'_>` guard (LB-002, long term) rather than `&Fr`, and add it to a `disallowed-methods` entry in the workspace `clippy.toml` with an `#[expect(clippy::disallowed_methods)]` at the three operator sites. A cargo feature (`operators`) would look tighter but is unified across the build like `unsafe` is (#85 LB-001) and would gate nothing.

**2. Move `Groth16LeaderProof::prove` into the KMS operator; measure the effect on proposal latency.**

Measured in LB-003 (Appendix B): PoL costs 200 ms wall / 1.03 core-s here and an estimated 1.2 s wall on the Pi 5. The move is right but has to cover both consumers found in LB-002, and must not await inside `execute`. `kms/operators` already depends on `lb-core` (for `LeaderPrivate`) and `lb-blend-proofs` (for `PoQOperator`), so both provers are reachable without a new dependency. Sketch:

```rust
// kms/operators/src/zk/leader.rs — replaces BuildPrivateInputsWithLeaderKey
pub struct ProveLeadership {
    result: oneshot::Sender<Result<Groth16LeaderProof, lb_core::proofs::leader_proof::Error>>,
    utxo: Utxo, leader_public: LeaderPublic,
    aged_path: MerklePath<Fr>, latest_path: MerklePath<Fr>,
    leader_pk: Ed25519PublicKey, voucher_cm: VoucherCm,
}
#[async_trait]
impl SecureKeyOperator for ProveLeadership {
    type Key = ZkKey; type Error = KeyError;
    async fn execute(self: Box<Self>, key: &ZkKey) -> Result<(), KeyError> {
        let key = key.clone();                       // ZkKey: Clone + ZeroizeOnDrop
        let Self { result, utxo, leader_public, aged_path, latest_path, leader_pk, voucher_cm } = *self;
        spawn_blocking("logos/kms/leader-proof", move || {
            let witness = LeaderPrivate::new(leader_public, utxo, &aged_path, &latest_path,
                                             key.expose_secret_scalar(), &leader_pk);
            let proof = Groth16LeaderProof::prove(witness, voucher_cm);   // witness dropped (zeroized) here
            drop(key);
            drop(result.send(proof));
        });
        Ok(())                                       // the loop is free again in microseconds
    }
}

// kms/operators/src/blend/poq.rs — the leadership branch, mirroring PoQOperator
pub struct LeaderPoQOperator {
    public_inputs: PublicInputs,                     // includes the caller's ephemeral signing pk
    slot: WinningSlot,                               // utxo, slot, aged path + selectors: all public
    message_release_index: KeyIndex,
    response: oneshot::Sender<Result<(VerifiedProofOfQuota, Fr), quota::Error>>,
}
// execute(): clone key, spawn_blocking { build ProofOfLeadershipQuotaInputs { secret_key: key…, ..slot },
//            VerifiedProofOfQuota::new(&public_inputs, PrivateInputs::new_proof_of_leadership_quota_inputs(idx, inputs)) },
//            return Ok(()) without awaiting.
```

Consumers then change as follows. `WinningSlotFuture` resolves to `Option<WinningSlot { key_id, utxo, slot, aged_path_and_selectors }>` instead of `Option<LeaderPrivate>` (`chain-leader/src/lib.rs:119`, `leadership.rs:544`); `PolInfoProvider` (`blend/pol.rs:63-92`) forwards `WinningSlot` values, `WinningPolInfoStream` (`blend/provers/src/provers/mod.rs:30`) carries them, and `RealLeaderProofsGenerator::create_proof_stream` (`leader/mod.rs:108-135`) calls a `LeaderProofOfQuotaGenerator` per message index the way the core prover already calls `CoreProofOfQuotaGenerator` (`core/mod.rs:112-140`), keeping its ephemeral signing key generation where it is (the operator only needs the public key, in `PublicInputs.signing_key`). `build_proof_for` (`leadership.rs:97-135`) obtains `voucher_cm` first (it already does, L119) and sends `ProveLeadership`; `PoQOperator::execute` (`poq.rs:66-72`) drops its `.await` on the join handle the same way. `LeaderPrivate::new` and `ProofOfLeadershipQuotaInputs` then only ever exist inside `kms/operators` blocking closures, and `BuildPrivateInputsWithLeaderKey` is deleted, together with the `From<LeaderPrivate> for PolWitnessInputsData` and `input()` accessors.

Latency effect of the sketch: zero on the loop (spawn only), zero on the proposal path (the proof runs on the same blocking pool it runs on today), and it removes the queueing already present (LB-003). Latency effect of the literal move with an awaited join, for comparison: `+T_PoL` (0.2–1.2 s) on every other KMS user per won slot, and `+T_PoQ` (0.16–1.03 s) on the proposal path whenever a Blend proof is in flight.

**3. Remove `Debug` and `Clone` from `LeaderPrivate`; add `ZeroizeOnDrop`.**

Both derives are unused in production: no `{:?}` of a `LeaderPrivate` exists (LB-002) and the only `.clone()` sites are test code; the KMS operator moves it, the chain-leader moves it into `spawn_blocking`, the Blend map borrows it. `ZeroizeOnDrop` on `LeaderPrivate` needs `PolWitnessInputsData: Zeroize`, which needs `zeroize` in `lb-pol` and a derive on `PolWitnessInputsData`, `PolWalletInputsData`, `PolChainInputsData` (all fields are `Fr`, `[Fr; 32]`, `[bool; 32]`, `u64`, `(Fr, Fr)`, for which `zeroize 1.9.0` has impls: arrays L342, tuples L495, `Fr` in ark-ff). `lb-core` already has no `zeroize` dependency (`core/Cargo.toml`), so add it there too. Under the design in item 2 the type becomes `pub(crate)` to `kms/operators`' view and the question mostly disappears; the derives should still go.

**4. Audit the witness types in `lb_pol`, `lb_poc`, `lb_zksign` for `Zeroize`; consider a `Zeroizing<Fr>`-style newtype.**

Done in the LB-002 table. None of the three crates depends on `zeroize` (`zk/proofs/{pol,poc,zksign}/Cargo.toml`); the only zeroizing witness types in the tree are the three `blend/proofs` input structs. For completeness, `lb_poc`'s `PoCWalletInputsData.secret_voucher` (`poc/src/wallet_inputs.rs:13`, derives `Clone, Debug`) is the voucher secret from #85 LB-001 and has the same gaps, out of this issue's scope. The diff for the PoL side:

```diff
--- zk/proofs/pol/Cargo.toml
+zeroize = { workspace = true }          # features = ["derive"] at the workspace entry
--- zk/proofs/pol/src/wallet_inputs.rs
-#[derive(Clone, Debug)]
+#[derive(Clone, Zeroize, ZeroizeOnDrop)]
 pub struct PolWalletInputsData { …, pub secret_key: Fr }
-#[derive(Clone, Debug)]
+#[derive(Clone, Zeroize, ZeroizeOnDrop)]
 pub struct PolWalletInputs { …, secret_key: Groth16Input }     // needs Zeroize on Groth16Input
--- zk/proofs/pol/src/inputs.rs
-        let inputs_str: String = serde_json::to_string(&inputs_json)?;
+        let inputs_str = Zeroizing::new(serde_json::to_string(&inputs_json)?);
         let witness_input = lbc_pol_sys::PolWitnessInput::new(inputs_str)?;   // lbc-types: take Zeroizing<String>, keep the CString in Zeroizing
--- zk/groth16/src/public_input/mod.rs
-#[derive(Copy, Clone, Debug)]
+#[derive(Copy, Clone, Debug, Zeroize)]
 pub struct Input<E: Pairing>(<E as Pairing>::ScalarField);
```

and in `lbc-types` (circuits repo): `Witness` holds `Zeroizing<Vec<u8>>` instead of `bytes::Bytes`, and `From<ffi::Bytes>` does `ptr::write_bytes(data, 0, size)` before `free_bytes`. `Bytes` is immutable and cannot be zeroed in place, which is why the type has to change; the only consumer is `Rapidsnark::prove(&[u8])` (`zk/circuits/prover/src/rapidsnark.rs:10-19`), which takes a slice. The `Zeroizing<Fr>` newtype: `zeroize::Zeroizing<Fr>` works as a field type today (`Fr: Zeroize`) but is `Clone` and derefs to a `Copy` value, so `*x` still copies silently; a purpose-built `SecretScalar(Fr)` with no `Deref`, no `Copy`, no `Debug`, `expose(&self) -> &Fr`, and `ZeroizeOnDrop`, is what makes every copy a reviewable `.clone()`.

## 6. Suggestions (non-security)

### S-001 · `UnsafeVoucherOperator` is gated by the `unsafe` feature and therefore never gated

`kms/operators/src/zk/mod.rs:2-3` puts the voucher operator behind `#[cfg(feature = "unsafe")]`, and the wallet uses it unconditionally (`services/wallet/src/lib.rs:1256-1273`). Since the feature is always on (#85 LB-001) the gate documents intent only. Under the design in §5 item 2 it becomes a `DeriveVoucher` operator that returns `(VoucherCm, VoucherNullifier)`, which is what the `TODO` at `voucher.rs:13-14` and `wallet/src/lib.rs:1256` already ask for; tracked by #122.

### S-002 · `SecretKey::into_inner` and `ZkKey::multi_sign` clone whole keys to build a stack array

`multi_sign` (`zk/mod.rs:51-56`, `:139-147`) clones every `ZkKey` into a `Vec<UnsecuredZkKey>` to call `UnsecuredZkKey::multi_sign`, which then clones each again through `into_inner` into `[Fr; 32]` (`zk/private.rs:94-97`). Take `&[&Self]` down to `try_from_secret_keys` and fill a `Zeroizing<[Fr; 32]>` from `as_fr()` directly; two allocations and one unzeroized buffer fewer per ZK signature.

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

## Appendix B — Measurements and repro

### B.1 Proving benchmarks

Machine: Apple M3 Pro (12 cores), 36 GB, macOS; `rustc 1.98.1`; the repository's own `divan` benches at the target commit, production `prebuilt` proving keys (circuits v0.5.7). Commands, from the node checkout:

```sh
cargo bench -p logos-blockchain-pol --bench prove --locked --no-run
cargo bench -p logos-blockchain-poq --bench prove --locked --no-run
/usr/bin/time -l target/release/deps/prove-<pol>  --bench --sample-count 10 --sample-size 1
/usr/bin/time -l target/release/deps/prove-<poq>  --bench --sample-count 5  --sample-size 1
```

Output (the `--bench` flag matters: without it `divan` runs each bench once in test mode and prints no timings):

```
prove           fastest       │ slowest       │ median        │ mean          │ samples │ iters
╰─ bench_prove  169.8 ms      │ 272.9 ms      │ 199.8 ms      │ 211.4 ms      │ 10      │ 10
        2.17 real        10.30 user         0.34 sys        146653184 maximum resident set size

prove                     fastest       │ slowest       │ median        │ mean          │ samples │ iters
├─ bench_prove_core_node  150.7 ms      │ 167.2 ms      │ 160.9 ms      │ 158.7 ms      │ 5       │ 5
╰─ bench_prove_leader     159.3 ms      │ 200.9 ms      │ 177.8 ms      │ 177.4 ms      │ 5       │ 5
        1.73 real         9.60 user         0.30 sys        119685120 maximum resident set size
```

`user / samples` gives about 1.0 core-second per PoL proof and 0.96 per PoQ proof; rapidsnark uses every core, so wall time is what a `spawn_blocking` caller sees. The Raspberry Pi 5 figures quoted in LB-003 are PR #109's (`bench_prove_core_node` pinned: 3.89 s; unpinned: 1.03 s wall, 3.88 core-s), same crate and bench, not re-run here.

### B.2 LB-001 repro

A standalone crate depending on `kms/keys` at the target commit by path with `features = ["unsafe"]` (the node's effective feature set), `async-trait`, `tokio`, `num-bigint`:

```rust
use std::marker::PhantomData;
use lb_key_management_system_keys::keys::{
    Ed25519Key, Key, KeyOperators, ZkKey, errors::KeyError,
    secured_key::{SecureKeyOperator, SecuredKey},
};
use num_bigint::BigUint;

#[derive(Debug)]
struct NoOp<K>(PhantomData<K>);
#[async_trait::async_trait]
impl<K: Send + Sync + 'static> SecureKeyOperator for NoOp<K> {
    type Key = K; type Error = KeyError;
    async fn execute(self: Box<Self>, _key: &Self::Key) -> Result<(), Self::Error> { Ok(()) }
}

#[tokio::main(flavor = "current_thread")]
async fn main() {
    let zk = Key::Zk(ZkKey::new(BigUint::from_bytes_le(&[0x11u8; 32]).into()));
    let op = KeyOperators::Ed25519(Box::new(NoOp::<Ed25519Key>(PhantomData)));
    let err = zk.execute(op).await.unwrap_err();
    println!("REPRO Display: {err}");
    println!("REPRO Debug:   {err:?}");
}
```

Output:

```
REPRO Display: Unsupported operator `Ed25519(NoOp(PhantomData<logos_blockchain_key_management_system_keys::keys::ed25519::Ed25519Key>))` for key type `Zk(ZkKey(SecretKey(7719472615821079694904732333912527190217998977709370935963838933860875309329)))`
REPRO Debug:   UnsupportedKeyOperator { operator: "Ed25519(NoOp(PhantomData<…Ed25519Key>))", key: "Zk(ZkKey(SecretKey(7719472615821079694904732333912527190217998977709370935963838933860875309329)))" }
```

`7719472615821079694904732333912527190217998977709370935963838933860875309329` is `0x1111…11` (32 bytes) read little-endian, i.e. the test key.

### B.3 Ruled out

| Checked | Where | Result |
|---|---|---|
| Any production caller of `ZkKey::as_fr` outside `kms/operators` | workspace grep, every hit read | none; other hits are public-key wrappers or `#[cfg(test)]` / `test_utils` uses of `UnsecuredZkKey` |
| Any `{:?}` / `{:#?}` of a witness-bearing value outside tests | grep for `slot_inputs`, `leader_private`, `private_inputs`, `witness`, `wallet_data`, `inputs` with `:?` | none; only `public_inputs:?` sites |
| `Debug` impls that could print the scalar indirectly | `ProofType` (`private.rs:102-110`), `PolEpochInfo` (`epoch_info.rs:24-26`), `LeaderMsg` (`chain-leader/src/lib.rs:177-186`), the three operators, `Groth16LeaderProof` (`leader_proof.rs:35-47`) | all redacted or constant |
| KMS reply channels carrying key material | `KMSMessage` variants (`message.rs:18-43`) and their handlers (`lib.rs:122-190`) | `Register`/`PublicKey`/`Sign` return public keys or signatures only; secrets leave only through operators' own channels |
| `CheckLotteryWinning` and `PoQOperator` | `leader.rs:49-67`, `poq.rs:57-77` | keep the scalar inside `execute` (stack copy in `check_winning`, `ProofOfCoreQuotaInputs` zeroized on drop) |
| Whether the witness generator's C status message can echo input values | `lbc-types native/status.rs:31-53` (256-byte message from C) | not verifiable without the C++ source; noted, not claimed |
| `lbp_error::Error` embedding witness values | `zk/proofs/error/src/lib.rs:4-16` | wraps `serde_json`/io/FFI errors; the `Groth16JsonInput` variant formats a parse error of a public *output* signal, not the witness |
