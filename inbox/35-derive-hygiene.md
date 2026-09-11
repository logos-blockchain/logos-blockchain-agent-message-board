# Audit Report — Derive hygiene on secret-bearing and consensus types (issue #35)

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/35`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `kms/keys, kms/operators, core/src/proofs, core/src/mantle, zk/proofs/{pol,poq,poc}, blend/proofs, consensus/cryptarchia-engine, nodes/node/binary/src/config`
Specs: `https://github.com/logos-co/logos-lips` @ `7c2ec59fadcdc2ba0ae74ff7600a14b55d90efcd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `cryptarchia-v1-protocol.md`
Date: `2026-09-11` — author: `Claude Fable 5.1 (agent, session 2e5d7afe)` — status: `final`

---

## 1. Summary

- Overall assessment: the KMS key types themselves are careful (constant-time equality, redacting `Debug`, `Hash` over the public key, `ZeroizeOnDrop`), but the secret material that leaves the KMS to feed the ZK provers is wrapped in types whose `Debug`, `Serialize` and `Default` are derived, in contrast with the Blend proof crate which redacts the same kind of inputs by hand; the CLI key parser also uses a lossy `From<BigUint>` where the public-key parser uses a checked conversion.
- Findings: 0 critical · 0 high · 0 medium · 3 low · 1 informational
- Key themes: derived `Debug` on prover witness types carrying the leader secret key and the voucher secret; `From` conversions that reduce modulo the field order silently; secret newtypes with unconditional `Serialize`/`Default`.
- Must-fix before launch: none. LB-001 and LB-002 are cheap to close and should be done together with the follow-ups already tracked in #122 and #123.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `kms/keys/src/keys/**` | `Key`, `ZkKey`, `UnsecuredZkKey`, `ZkPublicKey`, `Ed25519Key`, `UnsecuredEd25519Key`, `Ed25519PublicKey`, `Signature`, `X25519PrivateKey`, `X25519PublicKey`, `SharedKey`: every derive, `Debug`, `PartialEq`, `Hash`, `From` impl |
| `kms/operators/src/zk/{leader,voucher}.rs` | operators that move secrets out of the KMS |
| `services/key-management-system/src/{lib.rs,backend/preload.rs}` | settings type, what the service logs |
| `core/src/proofs/{leader_proof,leader_claim_proof}.rs` | `LeaderPrivate`, `LeaderClaimPrivate`, `Groth16LeaderProof` |
| `zk/proofs/pol/src/{inputs,wallet_inputs,witness}.rs`, `zk/proofs/poq/src/{inputs,wallet_inputs,blend_inputs}.rs`, `zk/proofs/poc/src/wallet_inputs.rs` | prover witness types |
| `blend/proofs/src/quota/{mod.rs,inputs/prove/*.rs}`, `blend/proofs/src/selection/mod.rs` | PoQ/PoSel private inputs and proof types |
| `blend/message/src/{input.rs,crypto/key_ext.rs}`, `blend/crypto/src/{lib.rs,cipher.rs}`, `blend/membership/src/lib.rs` | ephemeral keys, shared keys, `Node` |
| `consensus/cryptarchia-engine/src/{lib.rs,time.rs,config.rs}` | `Slot`, `Epoch`, `Branch`, `Config`, uncle ordering |
| `core/src/{header/mod.rs,block/mod.rs,sdp/mod.rs,sdp/blend.rs}` | `HeaderId`, `Header`, `DeclarationId`, `ProviderId`, `Locator`, `ActivityProof` |
| `core/src/mantle/{ledger.rs,gas.rs,channel.rs,transactions/hash.rs,ops/leader_claim.rs,ops/channel/mod.rs}` | `NoteId`, `Note`, `Utxo`, `Gas*`, `TxHash`, `VoucherSecret`, `VoucherCm`, `SlotTimeframe` |
| `ledger/src/{lib.rs,config.rs,cryptarchia/mod.rs}` (first ~300 lines of each) | `EpochState`, `LedgerState`, `Config`, `RewardPoWConfig` |
| `utils/src/math.rs`, `zk/groth16/src/lib.rs`, `zk/proofs/pol/src/lottery.rs` | float wrappers, `Fr` conversions |
| `nodes/node/binary/src/{config/mod.rs,config/kms/serde.rs,config/network/serde/mod.rs,cli/mod.rs,main.rs}` | config/CLI types that hold keys, the key parsers |
| `services/chain/chain-leader/src/{lib.rs,leadership.rs,wallet.rs}`, `services/blend/src/{settings/mod.rs,core/settings.rs}`, `services/wallet/src/{lib.rs,states.rs}` (headers) | where the secret-bearing types are handled and logged |

**Out of scope**
- Third-party crates assumed correct: `ed25519-dalek` (its `SigningKey` `Debug` is assumed to redact the seed, as noted in the #36 report), `x25519-dalek`, `libp2p-identity` (its `ed25519::SecretKey` `Debug` was not checked), `ark-ff`, `zeroize`, `serde`, `subtle`, `rpds`.
- Not reviewed: `tools/blockchain-tools`, `c-bindings`, `tests/`, the mempool and storage services, `blend/scheduling`, `blend/network`, the remainder of `ledger/src/lib.rs` and `ledger/src/mantle/**`. Whether any of those formats a secret-bearing type with `{:?}` was not established; the findings below are about the derives, and the report says explicitly where a formatting site was and was not found.
- Panics, `as` casts and `unsafe` are covered by sibling sweeps (#29, #28, #30); items seen in passing are noted under Suggestions with a pointer.

**Assumptions**
- The KMS `unsafe` feature is enabled in every build of the node, as established by the #36 report (PR #84, LB-001) and re-confirmed here: `kms/operators/Cargo.toml:20` and `services/wallet/Cargo.toml:26` request it from `[dependencies]`, and `services/wallet/src/lib.rs:45` uses `operators::zk::voucher`, which only exists under that feature (`kms/operators/src/zk/mod.rs:2-3`).
- The node is run with the default tracing configuration; log sinks (file, GELF, Loki) are treated as places a secret must not reach.

## 3. Method

- Manual review of the in-scope paths, working through the three items of #35 against every type that holds a secret (KMS keys, prover witnesses, voucher secrets, ephemeral and shared keys) and every type used in consensus ordering (`Slot`, `Epoch`, `HeaderId`, `NoteId`, `TxHash`, `DeclarationId`, `ProviderId`, `Locator`, `Gas`).
- Spec conformance: `bedrock-architecture-overview.md` and `overview-cryptoeconomics.md` in full; `cryptarchia-v1-protocol.md` in full, used for the uncle selection ordering (§Uncle Selection) that the `HeaderId` `Ord` derive feeds.
- Automated tooling: none. The review was done by reading files; no `cargo clippy` run, no grep over the tree, so the coverage is the file list in §2 and not the whole workspace.
- Dynamic testing: none.
- Cross-referenced with the earlier reports on the same code: #36 (PR #84, KMS `unsafe` feature always on), #85 (PR #121, `ZkKey` `Debug` under `unsafe`) and their follow-ups #122 and #123. The `ZkKey`/`UnsecuredZkKey` `Debug` leak is listed here only as a cross-reference (LB-003) so the inventory is complete; it is not a new finding.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | PoL and PoC witness types derive `Debug` (and `Serialize`) around the leader secret key and the voucher secret | Data Exposure | Low | High | Open |
| LB-002 | CLI ZK secret-key parser uses a lossy `From<BigUint>` that reduces the key modulo the field order and accepts any length | Data Validation | Low | Medium | Open |
| LB-003 | `UnsecuredZkKey` derives `Debug`/`Serialize`, and `ZkKey` delegates to it in every build (cross-reference to #36/#85) | Data Exposure | Low | High | Reported (PRs #84, #121; tracked by #122) |
| LB-004 | `VoucherSecret` and `X25519PrivateKey` derive `Serialize`/`Debug`/`Default` unconditionally, unlike the other secret newtypes | Data Exposure | Informational | High | Open |

### LB-001 · PoL and PoC witness types derive `Debug` (and `Serialize`) around the leader secret key and the voucher secret

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Data Exposure |
| Target | `core/src/proofs/leader_proof.rs:281-285` (`LeaderPrivate`), `zk/proofs/pol/src/inputs.rs:11-16` (`PolWitnessInputs`), `zk/proofs/pol/src/inputs.rs:29-33` (`PolWitnessInputsData`), `zk/proofs/pol/src/wallet_inputs.rs:14-24` and `:27-37` (`PolWalletInputs`, `PolWalletInputsData`), `core/src/proofs/leader_claim_proof.rs:124-127` (`LeaderClaimPrivate`), `zk/proofs/poc/src/wallet_inputs.rs:13-17` (`PoCWalletInputsData`) |
| Status | Open |

**Description**

The leader secret key leaves the KMS inside `LeaderPrivate` (`kms/operators/src/zk/leader.rs:120-128` builds it from `*key.as_fr()`), and the voucher secret inside `LeaderClaimPrivate`. Both wrappers, and every layer below them down to the raw witness structs, derive `Debug`:

```rust
// core/src/proofs/leader_proof.rs:281-285
#[derive(Debug, Clone)]
pub struct LeaderPrivate {
    input: lb_pol::PolWitnessInputsData,
    pk: Ed25519PublicKey,
}
// zk/proofs/pol/src/wallet_inputs.rs:27-37
#[derive(Clone, Debug)]
pub struct PolWalletInputsData { ..., pub secret_key: Fr, }
// zk/proofs/pol/src/inputs.rs:11-13
#[derive(Clone, Serialize, Debug)]
#[serde(into = "PolInputsJson", rename_all = "snake_case")]
pub struct PolWitnessInputs { pub wallet: PolWalletInputs, pub chain: PolChainInputs }
// core/src/proofs/leader_claim_proof.rs:124-127
#[derive(Debug, Clone)]
pub struct LeaderClaimPrivate { input: lb_poc::PoCWitnessInputsData }
// zk/proofs/poc/src/wallet_inputs.rs:13-17
#[derive(Clone, Debug)]
pub struct PoCWalletInputsData { pub secret_voucher: Fr, ... }
```

`Fr` is `ark_bn254::Fr`, whose `Debug` prints the scalar in full, so `{:?}` on any of these values prints the leader secret key (`secret_key`) or the voucher secret (`secret_voucher`) in clear. `PolWitnessInputs` additionally derives `Serialize` through `PolInputsJson`, which includes `secret_key` (`wallet_inputs.rs:55`); that serialisation is what the prover consumes, so it is needed, but it means any `serde_json::to_string(&inputs)` in a log line would emit the key too.

The same repository already treats the same secret differently in the Blend proof crate: `ProofOfLeadershipQuotaInputs` (which carries the very same `secret_key`) and `ProofOfCoreQuotaInputs` derive no `Debug` at all and are `ZeroizeOnDrop` (`blend/proofs/src/quota/inputs/prove/private.rs:123-127`, `:135-143`), and `ProofType` has a hand-written `Debug` that prints only the variant name (`private.rs:102-110`). `PoQWalletInputsData` and `PoQBlendInputsData` (`zk/proofs/poq/src/wallet_inputs.rs:18`, `blend_inputs.rs:13`) derive nothing. `RunningBlendConfig` in `services/blend/src/core/settings.rs:30-33` deliberately omits `Debug` with the comment "with the secret key exfiltrated from the KMS". `Groth16LeaderProof` (`leader_proof.rs:35-47`) and `ProofOfQuota` (`blend/proofs/src/quota/mod.rs:59-69`) have manual `Debug` impls. The PoL and PoC types are the odd ones out.

What was checked for an actual formatting site: the consumer path in `services/chain/chain-leader/src/leadership.rs:97-154` logs only `utxo.id()`, the slot and the error on every failure branch, never the `LeaderPrivate`; the operators in `kms/operators/src/zk/leader.rs:55-65` and `:120-132` log a fixed string when the `oneshot` send fails and drop the value; `spawn_blocking` failures surface as `JoinError`, which does not carry the closure. No `{:?}` of these types was found in the files read. The finding is therefore about the derive being available, not about a leak that happens today; the files not read (§2) were not checked.

**Exploit scenario**

None today. The risk is a future or out-of-tree log line, for example `tracing::error!("proof failed for {private:?}")` in a retry path, or a panic message that formats the witness, writing the leader's note secret key to the log sink. With the secret key and the (public) Merkle paths an attacker can produce leadership proofs and spend the note; with the voucher secret and the public voucher MMR path they can claim that leader reward. Because `Fr` is `Copy`, `Clone` on these types is not itself a problem, but it also means nothing is zeroized when they are dropped.

**Recommendation**
- *Short term*: replace the derived `Debug` on `LeaderPrivate`, `LeaderClaimPrivate`, `PolWitnessInputs`, `PolWitnessInputsData`, `PolWalletInputs`, `PolWalletInputsData` and `PoCWalletInputsData` with a manual impl that prints the public fields and `<redacted>` for `secret_key`/`secret_voucher`, following `ProofType` in `blend/proofs/src/quota/inputs/prove/private.rs:102-110`. Keep `Serialize` only on the `*Json` structs the prover needs.
- *Long term*: make the witness structs `ZeroizeOnDrop` like `ProofOfLeadershipQuotaInputs`, and move proving inside the KMS operator so the secret never leaves it, which is the direction #123 already proposes. Add `#![deny(clippy::missing_fields_in_debug)]` style review, or a small test that formats each secret-bearing type and asserts the secret's decimal representation is absent.

**References**: #123 (leader secret leaves the KMS inside `LeaderPrivate`), `blend/proofs/src/quota/inputs/prove/private.rs` for the in-tree precedent.

### LB-002 · CLI ZK secret-key parser uses a lossy `From<BigUint>` that reduces the key modulo the field order and accepts any length

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Data Validation |
| Target | `nodes/node/binary/src/config/mod.rs:564-570` (`parse_hex_zk_key`); the impls `kms/keys/src/keys/zk/private.rs:83-87` (`From<BigUint> for SecretKey`), `kms/keys/src/keys/zk/mod.rs:94-98` (`From<BigUint> for ZkKey`), `kms/keys/src/keys/zk/public.rs:77-81` (`From<BigUint> for PublicKey`) |
| Status | Open |

**Description**

The node's public-key parser is checked: `parse_hex_public_key` (`config/mod.rs:555-562`) goes through `fr_from_bytes`, which returns an error when the value is `>= p` (`zk/groth16/src/lib.rs:59-68`). The secret-key parser next to it is not:

```rust
// nodes/node/binary/src/config/mod.rs:564-570
pub fn parse_hex_zk_key(s: &str) -> Result<UnsecuredZkKey, String> {
    let bytes = hex::decode(s).map_err(|e| format!("Invalid hex string for ZK key: {e}"))?;
    let big_uint = BigUint::from_bytes_le(&bytes);
    Ok(UnsecuredZkKey::from(big_uint))
}
```

`UnsecuredZkKey::from(BigUint)` is `Fr::from(BigUint)` (`private.rs:83-87`), which reduces modulo `p` silently. Two consequences:

1. Any 32-byte value `>= p` is accepted and replaced by `value mod p`. `p ≈ 0.19 · 2^256`, so a uniformly random 32-byte string is `>= p` with probability about 0.81. A key produced by an external tool as "32 random bytes" will, four times out of five, be silently turned into a different scalar by this parser, and the node will derive a different `ZkPublicKey` than the tool did.
2. The length is not checked at all: a 1-byte or 100-byte hex string is accepted, so a weak or malformed key is never rejected. The Ed25519 parser beside it (`config/mod.rs:572-577`) uses `SecretKey::try_from_bytes`, which enforces 32 bytes.

The three `From<BigUint>` impls in `kms/keys` are the root: they are `From` where the conversion is not total, and the checked path (`fr_from_bytes`) already exists. `From<BigUint> for PublicKey` is only used in tests in the files read, but it offers the same silent reduction to any future caller.

**Exploit scenario**

Not attacker-triggerable; the input is the operator's own key. The impact is operational: an operator imports a key generated elsewhere, the node accepts it without complaint, and funds sent to the public key the external tool printed are owned by a scalar this node does not hold (or the reverse). A short key is accepted silently, so a truncated paste becomes a low-entropy signing key.

**Recommendation**
- *Short term*: in `parse_hex_zk_key`, require exactly 32 bytes and use `fr_from_bytes` (error on `>= p`), mirroring `parse_hex_public_key`; do the same wherever `add-key` or `generate-key` (`nodes/node/binary/src/cli/keys.rs`, not read) accept a ZK key.
- *Long term*: replace the three `From<BigUint>` impls with `TryFrom<BigUint>` returning the existing `FrFromBytesError`, and keep `fr_from_mod_bytes` for the one place that wants reduction by design (`SecretKey::from_rng`, see S-002).

**References**: `zk/groth16/src/lib.rs:59-75` (`fr_from_bytes` vs `fr_from_mod_bytes`).

### LB-003 · `UnsecuredZkKey` derives `Debug`/`Serialize`, and `ZkKey` delegates to it in every build (cross-reference)

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Data Exposure |
| Target | `kms/keys/src/keys/zk/private.rs:20-22`, `kms/keys/src/keys/zk/mod.rs:27-29` and `:76-86`, `kms/keys/src/keys/mod.rs:27-32` (`Key`), `services/key-management-system/src/backend/preload.rs:31-35`, `nodes/node/binary/src/config/kms/serde.rs:6-16`, `nodes/node/binary/src/config/mod.rs:57-58` (`UserConfig`) |
| Status | Reported in #36 (PR #84, LB-001) and #85 (PR #121); follow-up tracked in #122 |

**Description**

Listed for completeness of the inventory; the substance is in the two earlier reports. At this commit: `UnsecuredZkKey` derives `Debug, Serialize, Deserialize, Clone` (`private.rs:20-22`) and its `Debug` prints the scalar. `ZkKey`'s manual `Debug` writes `ZkKey(<redacted>)` only under `#[cfg(not(feature = "unsafe"))]` (`zk/mod.rs:76-86`), and `Serialize` is derived under the feature (`zk/mod.rs:28`). The feature is requested unconditionally from `kms/operators/Cargo.toml:20` and `services/wallet/Cargo.toml:26`, and `services/wallet/src/lib.rs:45` cannot compile without it. Consequently `Key`, `PreloadKMSBackendSettings`, `PreloadKmsBackendSettings`, `Config` and `UserConfig`, which all derive `Debug`, print every ZK secret key when formatted. `Ed25519Key` follows the same pattern (`ed25519/mod.rs:86-96`) and is only safe because `ed25519_dalek::SigningKey`'s own `Debug` redacts the seed.

No `{:?}` of `UserConfig`, `KmsConfig` or `PreloadKMSBackendSettings` was found in the files read (`main.rs`, `config/mod.rs`, `services/key-management-system/src/lib.rs` log only key ids and errors).

**Recommendation**: as in #122: drop the feature gate on redaction (always redact), and replace the `unsafe` feature with explicit, named export types for the three legitimate uses.

**References**: PR #84 LB-001, PR #121, issue #122.

### LB-004 · `VoucherSecret` and `X25519PrivateKey` derive `Serialize`/`Debug`/`Default` unconditionally, unlike the other secret newtypes

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Data Exposure |
| Target | `core/src/mantle/ops/leader_claim.rs:45-46` (`VoucherSecret`), `kms/keys/src/keys/ed25519/x25519.rs:8-9` (`X25519PrivateKey`) |
| Status | Open |

**Description**

```rust
// core/src/mantle/ops/leader_claim.rs:45-46
#[derive(Clone, Copy, Debug, Default, Eq, PartialEq, Hash, Serialize, Deserialize)]
pub struct VoucherSecret(#[serde(with = "serde_fr")] pub Fr);
// kms/keys/src/keys/ed25519/x25519.rs:8-9
#[derive(Clone, ZeroizeOnDrop, Deserialize, Serialize)]
pub struct X25519PrivateKey(StaticSecret);
```

`VoucherSecret` is the preimage of a voucher commitment and nullifier (`leader_claim.rs:129-158`): whoever holds it (plus the public MMR path) can submit the `LeaderClaim` for that voucher. It is derived as `Poseidon(sk, index)` in `kms/operators/src/zk/voucher.rs:37`, so leaking it does not leak the key, only that reward. The type derives `Debug` (prints the scalar), `Serialize` (no feature gate, unlike `ZkKey`/`Ed25519Key`), `PartialEq`/`Eq` (derived, so not constant-time, unlike every KMS key which uses `subtle::ct_eq`), and `Default` (the zero scalar, a valid-looking secret whose commitment and nullifier compute fine). In the files read it is only constructed from the KMS oneshot result in the wallet service and consumed by `LeaderClaimPrivate::try_new` (`leader_claim_proof.rs:130-149`); no `Default::default()` or `{:?}` of it was found.

`X25519PrivateKey` derives `Serialize`/`Deserialize` with no feature gate, while the two KMS key types put `Serialize` behind `unsafe`. Its equality is constant-time and it has no `Debug`, so this is an inconsistency rather than a leak: it is only used transiently in `blend/message/src/input.rs:34-36` (`derive_x25519().derive_shared_key(..)`), and no serialised struct embeds it in the files read.

**Recommendation**
- *Short term*: on `VoucherSecret` drop `Default` and `Debug` (or redact), implement `PartialEq` with `ct_eq` like `SecretKey` (`zk/private.rs:75-79`), and consider `ZeroizeOnDrop` (it is `Copy` today, which precludes that; a non-`Copy` newtype would be the fix). On `X25519PrivateKey`, gate `Serialize` the same way as `Ed25519Key`, or document why it must be serialisable.
- *Long term*: one rule for every secret newtype in the workspace (no derived `Debug`, no `Default`, `ct_eq` equality, `Serialize` only through an explicit export type), checked by a unit test that formats each one.

**References**: `kms/keys/src/keys/zk/private.rs:75-79` and `ed25519/x25519.rs:31-37` for the constant-time pattern already used.

## 5. Suggestions (non-security)

### S-001 · Omitting `node_key` from the network config silently generates a fresh libp2p identity on every start

Target: `nodes/node/binary/src/config/network/serde/mod.rs:29-31` and `:62-75`. `SwarmConfig` is `#[serde(default)]` and its `Default` calls `SecretKey::generate()`, so a config file without `node_key` gets a new random key, and therefore a new `PeerId`, on every run. The field's doc says "Default: random", so it is intended, but it is exactly the "`Default` producing a valid-looking value" pattern of #35: peers, Kademlia routing tables and any operator allow-list keyed on the `PeerId` stop matching after a restart with no error. Consider making `node_key` required (or fail when `Default` is reached at run time rather than at `init-config` time).

### S-002 · `SecretKey::from_rng` reduces 32 random bytes modulo `p`, which is not uniform

Target: `kms/keys/src/keys/zk/private.rs:68-72`, `zk/groth16/src/lib.rs:71-75`. `fr_from_mod_bytes` over a 256-bit sample gives residues below `2^256 mod p` six preimages and the rest five (about a 6:5 skew over the low ~29 % of the field). This is not exploitable, but the usual construction is to sample 64 bytes (or reject and resample values `>= p`). `fr_from_bytes_unchecked` (`lib.rs:93-95`) has the same reduction and is documented for random values only; it is also used for domain constants (`KDF`, `NOTE_ID_V1`), where it is fine.

### S-003 · The multi-signature path copies every secret key into an unzeroized array

Target: `kms/keys/src/keys/zk/private.rs:89-99` (`try_from_secret_keys`) and `:46-48` (`into_inner`). `ZkKey::multi_sign` clones each key (`zk/mod.rs:53`, `:144`), then `try_from_secret_keys` copies the scalars into a plain `[Fr; 32]` that becomes `ZkSignPrivateKeysData`; neither is zeroized. Since `Fr` is `Copy`, `ZeroizeOnDrop` on `SecretKey` only clears the one instance and none of the copies made by `as_fr()`/`into_inner()`. This is the same root cause as #123 ("zeroize witness types"); worth handling there.

### S-004 · Slot number truncated to `u32` in the slot timer (for #28)

Target: `consensus/cryptarchia-engine/src/time.rs:322`: `slot_duration * u64::from(...) as u32`. Unreachable in practice (slot `2^32` is 136 years at one-second slots), but it is a silent `as` truncation on a consensus quantity and belongs in the #28 inventory. `EpochConfig::last_slot` (`time.rs:273`) similarly does `epoch.into_inner() + 1` on `u32`, which only matters at `Epoch::MAX`.

## 6. Checked and ruled out

Listed so the next agent does not redo it. All line numbers are at the target commit.

- **KMS key types.** `SecretKey`/`UnsecuredZkKey` `PartialEq` is `ct_eq` (`zk/private.rs:75-79`); `UnsecuredEd25519Key` (`ed25519/private.rs:77-81`), `X25519PrivateKey` (`x25519.rs:31-35`) and `SharedKey` (`x25519.rs:66-70`) likewise. `ZkKey` and `Ed25519Key` derive `PartialEq` but it delegates to those impls; both implement `Hash` over the public key (`zk/mod.rs:70-74`, `ed25519/mod.rs:74-78`), which is consistent with `Eq`. `Key` (`keys/mod.rs:27-32`) delegates everything. `SharedKey` has no `Clone`, `Debug` or `Serialize`. `X25519PublicKey` has no `Debug` at all (a usability gap, not a leak). `UnsecuredEd25519Key::to_bytes` returns `Zeroizing` and `Deserialize` wraps the buffer in `Zeroizing` (`ed25519/private.rs:48-50`, `:67-75`). `ed25519-dalek` is built with `zeroize` (`kms/keys/Cargo.toml:18`).
- **Blend proof private inputs.** `ProofOfCoreQuotaInputs`, `ProofOfLeadershipQuotaInputs`, `ProofOfWorkQuotaInputs` derive `Clone, PartialEq, Eq, ZeroizeOnDrop` and no `Debug`; `ProofType`'s `Debug` prints only the variant (`blend/proofs/src/quota/inputs/prove/private.rs:95-160`). `ProofOfQuota` has a manual hex `Debug` (`quota/mod.rs:59-69`). `ProofOfSelection` derives `Debug` but `selection_randomness` is on the wire anyway. `EncapsulationInput` (`blend/message/src/input.rs:5-16`) derives nothing.
- **Config types.** `RunningBlendConfig` deliberately has no `Debug` (`services/blend/src/core/settings.rs:30-33`). `LeaderSettings`/`LeaderWalletConfig` hold only a public key and a fee cap (`chain-leader/src/wallet.rs:5-13`). `CliArgs`/`NetworkArgs` derive `Debug` and `NetworkArgs.node_key` is `libp2p`'s `ed25519::SecretKey` (`config/mod.rs:226-236`); its `Debug` is third-party and was not checked. `BlendArgs` carries KMS key ids, not keys. `RewardPoWConfig` deliberately has no `Default` (`ledger/src/config.rs:137-139`) and validates on deserialisation.
- **`Hash` on floats or pointers.** `FiniteF64`, `NonNegativeF64`, `PositiveF64` derive `PartialEq, PartialOrd` only, no `Hash`/`Eq` (`utils/src/math.rs:4, 60, 115`). Cryptarchia `Config` derives `PartialEq` only (`consensus/cryptarchia-engine/src/config.rs:9`). `LotteryConstants` is two `BigUint`s (`zk/proofs/pol/src/lottery.rs:21-25`). No `Hash` over a pointer or `Arc` address was found; `EpochState.active_declarations: Arc<Declarations>` derives `PartialEq` (deep), not `Hash`.
- **`Ord` derives in consensus and ledger.** Every derived `Ord` found is on a single-field newtype whose inner order is the intended one: `Slot(u64)`, `Epoch(u32)` (`time.rs:13-32`), `HeaderId([u8;32])` (`header/mod.rs:22`), `DeclarationId([u8;32])` (`sdp/mod.rs:375`), `TxHash`/`TxHashPrefix` (`transactions/hash.rs:12, 23`), `NoteId(Fr)` and `ZkPublicKey(Fr)` (integer order of the scalar), `Gas`/`GasCost(u64)` (`gas.rs:13, 71`), `Locator` (multiaddr bytes, `sdp/mod.rs:112`). `ProviderId` implements `Ord` by hand over the key bytes (`sdp/mod.rs:363-373`). No multi-field struct with a derived `Ord` was found on a consensus path; `Branch`, `Header`, `Declaration`, `Note`, `Utxo` derive `PartialEq`/`Eq` only. The one place a derived `Ord` decides protocol output is uncle selection, `sort_unstable_by_key(|(parent_slot, uncle)| (*parent_slot, uncle.slot, uncle.id))` (`cryptarchia-engine/src/lib.rs:739-740`); the spec orders by `(sl_parent(U), sl_U, block_id(U))` (`cryptarchia-v1-protocol.md`, §Uncle Selection), lexicographic on the id bytes, which is what the derive gives, and the spec states the selection is "a recommendation for filling the entries well, not a consensus rule".
- **`Default` producing valid-looking values.** `Slot` (`time.rs:16`, genesis; used on purpose in `Channels::from_genesis`, `channel.rs:163`), `TxHash`/`TxHashPrefix` (zero hash), `GasPrice` (zero), `VoucherCm`/`VoucherNullifier`/`RewardsRoot` (zero; `VoucherCm::default()` is the deliberate genesis sentinel, `leader_proof.rs:118`). `Epoch` and `EpochConfig` have no `Default`. No `#[serde(default)]`, `unwrap_or_default()` or `mem::take` on any of these was found in the files read; `VoucherSecret` and `SwarmConfig` are the two cases reported (LB-004, S-001).
- **`From`/`TryFrom`.** Checked conversions are used where the domain is narrower: `Epoch: TryFrom<u64>` (`time.rs:204-210`), `InactivityPeriod` (`sdp/mod.rs:76-90`), `ProviderId: TryFrom<[u8;32]>` (`:353-361`), `HeaderId: TryFrom<&[u8]>` (`header/mod.rs:244-255`), `Version: TryFrom<u8>` (`:62-74`), `ServiceType: TryFrom<u8>` (`sdp/mod.rs:279-288`), `Locator: TryFrom<Multiaddr>` (`:168-198`), `PowReward: TryFrom<u128>` (`ledger/src/lib.rs:430-433`), `FiniteF64: TryFrom<u64>` with an exactness check (`math.rs:46-57`), `RewardPoWConfig: TryFrom<RewardPoWConfigFields>`. Lossy `From`s other than LB-002: `Fr::from_le_bytes_mod_order` on the op id and tx hash in `Utxo::id` and `TxHash::to_fr` (`ledger.rs:516`, `hash.rs:103-105`) is the specified derivation; `SlotTimeframe`/`SlotTimeout: From<u32>` are exact. `Utxo.output_index: usize` (`ledger.rs:493`) enters the note id through `to_le_bytes()` then `BigUint::from_bytes_le`, so the value is the same on 32- and 64-bit targets; `serde` encodes `usize` as `u64`. Not a determinism issue (for #40).
- **Logging of keys in the KMS service.** `services/key-management-system/src/lib.rs:120-215` logs key ids and `Backend::Error` values only; the `Backend::Key: Debug` bound is never exercised by a log line there.
