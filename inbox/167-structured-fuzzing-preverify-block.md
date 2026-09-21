# Audit Report — Structured fuzzing past the signature checks: generated transactions and blocks for `preverify`, `body_root` and `into_verified`

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/167`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `3b7730a7c0cea9c367b29b63b4d768586b90419a` — component(s): `core/src/mantle/ops/*` (`preverify` of every op), `core/src/mantle/transactions/tx_list/signed_ops.rs`, `core/src/block/{mod,uncle}.rs`, `core/src/header/mod.rs`, `core/src/proofs/leader_proof.rs`
Specs: `https://github.com/logos-co/logos-lips` @ `75d3d0382604d4a0d8e246c268935dd386ffc8ea` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `network-wire-format.md`, `mantle-transaction-encoding.md` (in full); `bedrock-v1.1-block-construction.md` (Hash, Block Proposal, Header, References, Proof of Leadership, Canonical Encoding, Block, Block Proposal Reconstruction, Block Proposal Validation, Block Execution) and `bedrock-v1.1-mantle-specification.md` (Mantle Transaction, Mantle Transaction Hash, Validation, Opcodes, and the Payload/Proof/Validation sections of CHANNEL_INSCRIBE, CHANNEL_CONFIG, CHANNEL_DEPOSIT, CHANNEL_WITHDRAW, CHANNEL_TRANSFER, SDP_DECLARE, SDP_WITHDRAW, SDP_ACTIVE, LEADER_CLAIM, CLAIM_POW_REWARD, TRANSFER, Multiple Ed25519 Signatures Verification) by section
Date: `2026-09-21` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: the paths behind the signature checks are reachable with structure-aware generators without any change to the node (the crate-private `Groth16LeaderProof::from_parts` the issue expected to need is unnecessary, because the public `BinaryDecode` impl builds a proof of leadership with any `leader_key`). Two generator targets ran for one hour each (498 031 generated transactions and blocks in total) and every property the issue asks for held: generated transactions decode canonically and pass `preverify`, every flip of a verified signature byte fails it, generated blocks pass `Block::try_from(Bytes)` for both transaction types, and every independent mutation of the header, signature, uncles or transaction ops is rejected; no panic, timeout or assertion failure occurred. The review of the code those generators target found one disagreement with the specification: the `CLAIM_POW_REWARD` operation carries no proof at all in the node, while the specification (encoding rev 1.8.0, Mantle § CLAIM_POW_REWARD) requires a ZK signature by the beneficiary key and verifies it. The double verification in `Block::try_from` (#69 S-002) costs 0.2 to 12 ms per block.
- Findings: `0` critical · `0` high · `1` medium · `0` low · `0` informational.
- Key themes: "the generated-input properties hold", "which proofs `preverify` actually verifies at this commit differs from the issue's premise", "the wire format of a whole operation class disagrees with the spec".
- Must-fix before launch: LB-001 (the transaction wire format is not interoperable for `CLAIM_POW_REWARD`, and the ownership proof the spec relies on is absent).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `core/src/mantle/ops/signed_op.rs:116-176` | `SignedOp::into_preverified`: which ops get a `tx_hash_view` (can verify a signature) and which get `()`. |
| `core/src/mantle/ops/channel/inscribe.rs:103-118`, `core/src/sdp/mod.rs:509-521` (via `ops/sdp/declare.rs:174-184`), `core/src/mantle/ops/leader_claim.rs:199-220` | The three `preverify` impls that verify a signature or proof against the transaction hash. |
| `core/src/mantle/ops/channel/{config.rs:80-99, deposit.rs:84-94, withdraw.rs:74-86, channel_transfer.rs:82-99}`, `ops/transfer.rs:102-119`, `ops/sdp/active.rs:49-58`, `ops/pow.rs:286-295` | The structural-only `preverify` impls. |
| `core/src/mantle/transactions/tx_list/signed_ops.rs:105-112, 181-237, 345-371` | `SignedOps::preverify`, the columnar wire encoding, `Hashable` (hash over ops only), `StorageSize`, and the `Preverified` `Deserialize` that runs `preverify`. |
| `core/src/block/mod.rs:169-389`, `core/src/block/uncle.rs`, `core/src/header/mod.rs:141-219`, `core/src/utils/merkle.rs:14-30` | `Block::create`, `reconstruct`, `into_verified`, `validate_total_transactions_size`, `validate_body_root`, `verify_header_alone`, `verify_header_signature`, `body_root`, `TryFrom<Bytes>`; header encoding and signing. |
| `core/src/proofs/leader_proof.rs:68-89, 140-152` | `Groth16LeaderProof` decoder (copies bytes, no curve parsing) and the crate-private `from_parts`. |
| `fuzz/` (Appendix C) | Two generator-based libFuzzer targets, a proof-of-claim seed generator and a double-verification benchmark. |

**Out of scope**

- Stateful validation (`into_verified` of operations against a ledger, channel or SDP state), block execution, mempool admission policy, and the chain service's handling of a rejected synced block (covered by #313, #554, #520, #143, #184).
- The proof systems themselves (`lb-pol`, `lb-poc`, `lb-zksign`, `ark-*`, `rust-rapidsnark`), `ed25519-dalek`, `bincode 1.3`, `serde`, `libfuzzer-sys`, `arbitrary`: assumed correct.
- Byte-level fuzzing of the decoders: done by the report for #69 (`processed/69-fuzz-harness-ingress-decoders.md`); not repeated.

**Assumptions**

- Facts from #19 hold at this commit: `[profile.release]` sets no `overflow-checks` (`Cargo.toml:11-14`); the workspace allows `arithmetic_side_effects`, `indexing_slicing`, `unwrap_used`, `expect_used`, `panic`. The harness therefore builds with `-Coverflow-checks -Cdebug-assertions` (cargo-fuzz's defaults) so that an overflow in the code under test surfaces as a panic.
- No `fuzz/` directory and no `arbitrary` dependency exist at this commit (`ls`; `grep -n arbitrary Cargo.toml` hits only `arbitrary-int` at `:185` and the lint name at `:330`).
- The deployed transaction type is `SignedOps<Preverified, StandardMode>` (`nodes/node/binary/src/generic_services/mod.rs:26-65`), so a block downloaded during sync is `Block<SignedOps<Preverified, StandardMode>>` and its `Deserialize` runs `preverify` on every transaction (`signed_ops.rs:360-371`, `block/mod.rs:106-132`).

## 3. Method

- Manual review of the in-scope paths, working through the five items of #167 under parent #10. The report for #69 was read as the origin of the issue; #113, #199, #313 and #554 were read for what they already establish about `preverify` and proof malleability.
- Spec conformance: the generators write the ABNF of `mantle-transaction-encoding.md` (rev 1.8.0) and the `Header` / `ProofOfLeadership` layout of `bedrock-v1.1-block-construction.md` § Canonical Encoding, and hand the bytes to the node's decoders; a refusal is a spec/code disagreement by construction. The `preverify` sections of `bedrock-v1.1-mantle-specification.md` were compared op by op with the code (Appendix B, table 1).
- Harness design (Appendix C). `fuzz/src/lib.rs` turns `arbitrary` input into a `SignedMantleTx`: it picks 0–6 opcodes, writes each payload with valid field values (real Ed25519 public keys from four fixed test keys where a payload names a signer, BN254 scalars below the modulus, non-empty inputs, non-zero output values, non-zero thresholds, well-formed multiaddr locators), decodes the `MantleTx` prefix with `Ops::decode_all`, hashes it as the node does (`Hashable for TxList`, `signed_ops.rs:222-231`), signs the hash with the named test key for every `Ed25519SigProof`, `ZkAndEd25519SigsProof` and `ChannelMultiSigProof` entry, and fills ZK proofs with arbitrary bytes (they are not verified by `preverify`). It records which byte ranges `preverify` verifies and which it does not. `fuzz_targets/signed_ops_gen.rs` asserts, per generated transaction: `SignedOps::decode_all` accepts it; `encode(decode(x)) == x`; `encoded_length() == storage_size() == encode().len()`; `preverify()` succeeds (or, for a garbage proof of claim, fails with `LeaderClaimVerificationError`); the gossip `bincode` envelope decodes to the same `Preverified` value; flipping any byte of a verified signature makes `preverify` fail and leaves the transaction hash unchanged; flipping a byte of an unverified proof leaves the verdict unchanged; flipping an op byte changes the hash and, when a verified signature is present, fails `preverify`. `fuzz_targets/block_gen.rs` builds 0–6 preverifiable transactions, 0–4 signed uncle headers, a proof of leadership whose `leader_key` is a test key, and calls `Block::create` with that key; it asserts `Block::try_from(Bytes)` accepts the bytes for both `Unverified` and `Preverified`, re-serialises them identically, and rejects independent flips of the version, slot, `body_root`, `leader_key`, signature, any uncle byte, any op byte of any transaction, a dropped uncle, a reordered transaction list and a trailing byte; and that flips of transaction *proof* bytes are accepted with the same block ID (the #313/#554 property), except that a flipped verified signature is rejected by the `Preverified` type only. `examples/gen_poc.rs` proves four real proofs of claim (one `LeaderClaim` transaction each, since the proof binds the transaction hash) for the transaction target; `examples/bench_block.rs` times `Block::from_bytes` (one `into_verified`) against `Block::try_from(Bytes)` (two) for six block shapes.
- Two generator corrections came out of the first runs, both places where the ABNF alone does not describe what the node accepts: `SDP_ACTIVE` metadata is a typed Blend activity proof rather than `UINT32 *BYTE` (already filed as #245; the generator now follows the code), and a `Locator` whose IPv4 address is `0.0.0.0` is rejected as unspecified by `Locator::try_from` (`core/src/sdp/mod.rs:168-190`), a rule the SDP specification carries and the encoding ABNF does not (the generator avoids it). Neither is a node defect.
- The harness is a self-contained nested workspace (`fuzz/Cargo.toml` carries its own `[workspace]` and path dependencies), so the node's root manifest, lock file and visibility are untouched; its release profile disables fat LTO to keep the build tractable on a 4-core machine.
- Automated tooling: `cargo-fuzz 0.13.2`; `rustc 1.98.1` (workspace pin) for the examples and the four proof-of-claim seeds (each proved and preverified in about 550 ms); `rustc 1.100.0-nightly (2026-09-10)` for the fuzz build with `-s none -a` (no sanitizer, since the properties are logical and #69 already ran AddressSanitizer on the decoders; debug assertions and overflow checks on). Both targets were first run for 3 000 iterations as a smoke test, which is where the two generator corrections above were found.
- Dynamic testing: one libFuzzer process per target for 3 600 s on a Raspberry Pi 5 (4 cores, 8 GB), in parallel, with `-max_len=8192 -timeout=25` (transactions) and `-max_len=16384 -timeout=60` (blocks), `-rss_limit_mb=2048`; results in Section 6.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Spec deviation: `CLAIM_POW_REWARD` carries no proof; the specification requires a ZK signature by the beneficiary key and verifies it | Authentication | Medium | Medium | Open |

### LB-001 · Spec deviation: `CLAIM_POW_REWARD` carries no proof; the specification requires a ZK signature by the beneficiary key and verifies it

| | |
|---|---|
| Severity | Medium |
| Difficulty | Medium |
| Category | Authentication |
| Target | `core/src/mantle/ops/pow.rs:275-278` (`impl ProvableOperation for ClaimPowRewardOp { type Proof = NoOpProof; const CODE: u8 = 0x40; }`), `:286-295` (`preverify` returns `Ok(())`), `:303-315` (`verify` checks pool, window, epoch nonce, ticket and double claim, and nothing about the key); `core/src/mantle/ops/proof_noop.rs:7-24` (encodes to zero bytes) |
| Status | Open |

**Description**

`mantle-transaction-encoding.md` rev 1.8.0 (dated 2026-09-08, thirteen days before the audited checkout) adds the `ClaimPowReward` payload and states "its proof is a `ZkSigProof`"; its `Op Proofs` production lists `ZkSigProof = ZkSignature = Groth16 = 128BYTE`. `bedrock-v1.1-mantle-specification.md` § CLAIM_POW_REWARD, *Proof*, says: "A ZkSignature by the secret key corresponding to `public_key`, over the transaction's `mantle_txhash`. The signature proves knowledge of that secret key, so a solution cannot be found by searching over public keys directly", and its *Validate* block ends with step 6, `assert ZkSignature_verify(mantle_txhash, claim_proof, [claim.public_key])`.

At `3b7730a7` the node binds the op to `NoOpProof` (`pow.rs:276`), which encodes and decodes as zero bytes (`proof_noop.rs:7-24`). `OpProof::decode_for_op` (`op_proof.rs:33-47`) therefore consumes nothing after a `ClaimPowReward` op, `preverify` does nothing (`pow.rs:292-294`), and `verify` (`pow.rs:303-315`) never reads a signature: the only binding between `public_key` and the claim is that the puzzle ticket is `Poseidon2(public_key, block_hash, epoch_nonce)` (`pow.rs:97-103`).

Two consequences follow. First, the wire format of every transaction that contains this op differs from the specification by 128 bytes: a spec-conformant encoder produces bytes the node rejects (`SignedOps::decode_all` sees 128 trailing bytes, or misparses them as the next proof), and the node's own bytes are rejected by a spec-conformant decoder, so the operation is not interoperable across implementations and the test vectors of #245/#641 cannot pin it. Second, the ownership check the specification introduced the proof for does not exist.

The generator in Appendix C follows the code (zero proof bytes) so that its positive cases pass; the deviation was found while deriving the proof table from the ABNF, before the harness ran, and is verified by reading the three locations above.

**Exploit scenario**

The specification's stated purpose for the signature is that a solution "cannot be found by searching over public keys directly". Without it, a miner grinds `public_key` values as raw field elements: each attempt costs one Poseidon2 compression (the ticket) instead of two (key derivation `pk = Poseidon2(KDF, sk)`, `kms/keys/src/keys/zk/private.rs:63-65`, then the ticket), so per unit of work the miner finds about twice as many winning tickets as a miner who needs a spendable key. The reward note is paid to a key nobody holds (`pow.rs:337-339`), so the miner gains nothing directly; what it does is drain the capped epoch pool (`are_pow_reward_enabled`, `InsufficientPoolBalance`, `pow.rs:309`, `:109-110`) ahead of honest claimants and push the reward difficulty up for everyone, at half the cost per claim of an honest miner. Separately, a second implementation that follows the specification cannot exchange these transactions with the node at all.

**Recommendation**

- *Short term*: `type Proof = ZkSignature` for `ClaimPowRewardOp`; in `verify`, defer the check like `SDPDeclare` does (`declare.rs:207-211, 222-225`): `DeferredZkpVerification::ZkSig(*proof.as_proof(), public_inputs_from_pks(tx_hash_view.as_fr().into(), &[operation.public_key]))`, and pass `tx_hash_view` in `ClaimPoWRewardVerificationContext`. Regenerate the `codec_fixtures!` vectors that contain the op.
- *Long term*: add a conformance test that derives the `(opcode, proof variant, proof length)` table from the encoding specification's ABNF and checks it against `ProvableOperation::Proof` for every op (the placeholder machinery at `op_proof.rs:142-160` already enumerates them), so a spec revision that changes a proof type fails CI instead of silently forking the wire format.

**References**: `mantle-transaction-encoding.md` § Proof of work operations, § Op Proofs (rev 1.8.0); `bedrock-v1.1-mantle-specification.md` § CLAIM_POW_REWARD (Proof, Validation step 6); #245 and #641 (test vectors) for the missing row.

## 5. Suggestions (non-security)

### S-001 · The issue's premise about multi-signature ops does not hold at this commit

#167 asks for generators "for the ops whose `preverify` checks a signature (`ChannelInscribe`, `SDPDeclare`, `LeaderClaim`, the channel multi-sig ops)". At `3b7730a7` only the first three receive the transaction hash in `into_preverified` (`signed_op.rs:124-129, 146-151, 160-165`); `ChannelConfig`, `ChannelWithdraw` and `ChannelTransfer` are preverified with the unit context (`:130-145`) and their `ChannelMultiSigProof` is verified only against the channel's accredited keys and threshold in `verify` (`config.rs:126-150`, `channel/verification.rs:12-52`), which is what `bedrock-v1.1-mantle-specification.md` § Multiple Ed25519 Signatures Verification requires (the keys come from the channel state). The generator still signs the multi-signature entries with the test keys at the indices it lists, so the values are ready for a stateful follow-up (a `Channels` with the four test keys accredited), but the harness asserts that flipping those bytes does *not* change the `preverify` verdict. The issue text should be read with that correction; nothing to fix in the code.

### S-002 · No `test-utils` constructor is needed for a hand-built proof of leadership

The issue anticipated a `test-utils`-gated `Groth16LeaderProof::from_parts` (crate-private at `leader_proof.rs:140-152`). The public `BinaryDecode` impl (`:68-89`) copies the 128 proof bytes without curve parsing and accepts any `leader_key`, so `Groth16LeaderProof::decode_all(&[proof ‖ entropy ‖ leader_key ‖ voucher_cm], &())` builds the same value from outside the crate, and `Header::decode_all` likewise builds a header with an arbitrary `body_root`. The harness uses both and needs no visibility change. The same is true of the header signature: `Header::sign` (`header/mod.rs:195-198`) is public.

### S-003 · `Block::try_from(Bytes)` still verifies twice (#69 S-002)

Unchanged at this commit: `block/mod.rs:382` deserialises through `reconstruct` → `into_verified` (`:124-131`, `:232`) and `:384-386` calls `into_verified` again. The benchmark in Appendix C measures the second pass (Section 6.3): 0.2 ms on an empty block, 10 to 12 ms on a 2 MiB body, at most a ×1.05 to ×1.3 overhead on the deployed `Preverified` type, whose per-transaction `preverify` dominates. Note also the order inside `into_verified` (`:242-251`): `body_root` (a Blake2b over the uncle list plus a Merkle root over up to 1 024 transaction hashes, each a Blake2b over that transaction's ops) is computed before the header signature is checked. For a synced block the specification imposes no order ("applies to it as written, with no ordering constraint", § Block Proposal Validation, last paragraph), so this is not a deviation; it is the cost an unauthenticated peer can impose per block before the 64-byte signature is rejected, which #553 and #512 already quantify.

## 6. Campaign and benchmark results

Everything below ran from a scratch checkout of `logos-blockchain` at `3b7730a7` with the `fuzz/` directory of Appendix C added and no other file modified. Commands, for reproduction:

```sh
cd logos-blockchain/fuzz
export CARGO_TARGET_DIR=$HOME/.cache/lb-audit-167/target
cargo build --release --examples
$CARGO_TARGET_DIR/release/examples/gen_poc $HOME/.cache/lb-audit-167/poc_txs 4     # four LeaderClaim txs with valid proofs of claim
$CARGO_TARGET_DIR/release/examples/bench_block 20                                    # Section 6.3
cargo +nightly fuzz build -s none -a
POC_TXS_DIR=$HOME/.cache/lb-audit-167/poc_txs cargo +nightly fuzz run signed_ops_gen -s none -a corpus/signed_ops_gen -- -max_total_time=3600 -max_len=8192 -timeout=25 -rss_limit_mb=2048 -print_final_stats=1
cargo +nightly fuzz run block_gen -s none -a corpus/block_gen -- -max_total_time=3600 -max_len=16384 -timeout=60 -rss_limit_mb=2048 -print_final_stats=1
```

### 6.1 Campaign results (audited commit `3b7730a7`, 3 601 s wall-clock per target)

| Target | Generator | Executions | exec/s | Edges (`cov`) | Features (`ft`) | Corpus units (final) | Longest input (B) | Peak RSS | Crashes / timeouts / assertion failures |
|---|---|---|---|---|---|---|---|---|---|
| `signed_ops_gen` | valid `SignedMantleTx` (0–6 ops, 11 opcodes, 4 test keys, 4 fixed PoC txs) | 212 257 | 58 | 3 247 | 11 943 | 1 074 (63 KB) | 241 | 60 MB | 0 / 0 / 0 |
| `block_gen` | valid `Block` (0–6 txs, 0–4 uncles, test-key leader) | 285 774 | 79 | 2 511 | 8 327 | 1 143 (117 KB) | 360 | 60 MB | 0 / 0 / 0 |

No artifact was written by either process (`artifacts/` empty), no `-timeout` or `rss_limit` alarm fired, and no assertion in either target fired once the two generator corrections of Section 3 were in. Coverage was still growing slowly at the end of the hour on both targets (`signed_ops_gen`: 2 639 → 2 894 → 3 122 → 3 202 → 3 247 edges at runs 64, 30k, 69k, 116k, 178k; `block_gen`: 1 920 → 2 228 → 2 411 → 2 483 → 2 511 at runs 160, 41k, 82k, 140k, 208k). The execution rate is low by fuzzing standards because every input costs several Ed25519 signatures and verifications, a Blake2b Merkle root, up to two block signature verifications and, one input in eight on the transaction target, a Groth16 proof-of-claim verification (about 1 ms), so the campaign's weight is in the properties checked per input rather than in the count.

One limitation to read the numbers with: libFuzzer's length control grew the input cap to only 254 B (transactions) and 365 B (blocks) within the hour at this rate, so the generators were fed at most 241 and 360 bytes; `arbitrary` fills every field it cannot read from the input with the lowest value (zero counts, zero bytes), so the six-op transactions and 1 024-transaction blocks the generators can express were not reached, and multi-op inputs were mostly short. A longer run, or `-len_control=0` with a seed corpus of full-size inputs, is the way to exercise the upper bounds (`MAX_OPS_PER_TX`, `MAX_BLOCK_TRANSACTIONS`, the 1.75 MiB inscription), which this campaign did not.

### 6.2 Properties asserted per input

| Property (from #167) | Where asserted | Result |
|---|---|---|
| Generated tx decodes through `SignedOps::decode_all`, and `decode` agrees with `decode_all` | `signed_ops_gen.rs` | Held on every input: the node's decoders accept every transaction the encoding spec's ABNF describes, with the two documented exceptions (`SDP_ACTIVE` metadata, #245; unspecified IPv4 locators). |
| `encode(decode(x)) == x`; `encoded_length() == storage_size() == encode().len()` | `signed_ops_gen.rs` | Held. |
| The transaction hash covers the `MantleTx` prefix only (`Ops::decode_all(prefix).hash() == tx.hash()`) | `signed_ops_gen.rs` | Held (by construction of `Hashable for TxList`, `signed_ops.rs:222-231`). |
| `preverify()` succeeds on every generated tx; a garbage proof of claim fails with `LeaderClaimVerificationError` | `signed_ops_gen.rs` | Held. |
| Flipping any byte of a signature `preverify` verifies (inscribe signer, declare provider, valid proof of claim) fails `preverify`, leaves the tx hash unchanged, and fails the `Preverified` gossip decode | `signed_ops_gen.rs` | Held, for the first, last and one middle byte of every such signature. |
| Flipping a ZK-signature or multi-signature byte leaves the `preverify` verdict unchanged | `signed_ops_gen.rs` | Held (S-001: those proofs are verified statefully or deferred). |
| Flipping an op byte changes the hash, so a tx with a verified signature no longer preverifies | `signed_ops_gen.rs` | Held. |
| Gossip `bincode` envelope (`SignedOps<Preverified>::from_bytes`) agrees with `preverify` | `signed_ops_gen.rs` | Held. |
| `Block::create` rejects a signing key that is not the `leader_key` (`KeyMismatch`) and a genesis slot (`Header(GenesisSlot)`) | `block_gen.rs` | Held. |
| `Block::create` + `Block::try_from(Bytes)` succeed for `Unverified` and `Preverified`; both re-serialise byte for byte; `reconstruct` agrees | `block_gen.rs` | Held. |
| The bincode layout has no length prefix on fixed-size fields, so `signature`, uncle and transaction offsets are computable | `block_gen.rs` | Held (`block/deser.rs:74-107` states the same). |
| Flipped version, slot, `body_root`, `leader_key`, signature, any uncle byte, any op byte of any transaction; a dropped uncle; a reversed transaction list; a trailing byte: all rejected by both types | `block_gen.rs` | Held. |
| A flipped *verified* signature inside a transaction is rejected by the `Preverified` type and accepted by the `Unverified` type; a flipped *unverified* proof byte is accepted by both with the same block ID and different bytes | `block_gen.rs` | Held: `body_root` does not commit to transaction proofs (#313, #554, #520). |

### 6.3 Double verification cost (`Block::try_from` vs `Block::from_bytes`)

Medians of 20 iterations, `examples/bench_block.rs`, release build without LTO, Raspberry Pi 5 (Cortex-A76, 4 cores), single thread. `from_bytes` runs `Deserialize` → `reconstruct` → `into_verified` once; `try_from` adds the second `into_verified` (`block/mod.rs:384-386`). The `Preverified` columns include one `preverify` per transaction inside `Deserialize`.

| Block shape | Bytes | Unverified `from_bytes` (1 verify) | Unverified `try_from` (2 verifies) | Preverified `from_bytes` | Preverified `try_from` |
|---|---|---|---|---|---|
| 0 tx | 377 | 215.6 µs | 416.0 µs (×1.93) | 216.0 µs | 415.9 µs (×1.93) |
| 64 `SDPActive` (nothing verified at `preverify`) | 26 489 | 1.55 ms | 1.85 ms (×1.19) | 1.67 ms | 1.97 ms (×1.18) |
| 1 024 `Transfer` (ZK signature deferred) | 307 257 | 7.07 ms | 8.63 ms (×1.22) | 8.83 ms | 10.40 ms (×1.18) |
| 1 024 `ChannelInscribe`, 32 B (Ed25519 at `preverify`) | 211 321 | 18.59 ms | 20.20 ms (×1.09) | 224.6 ms | 226.3 ms (×1.01) |
| 1 `ChannelInscribe`, 1.75 MiB (max inscription) | 1 835 559 | 23.19 ms | 33.72 ms (×1.45) | 33.47 ms | 43.97 ms (×1.31) |
| 1 024 `ChannelInscribe`, 1 850 B (1.95 MiB body) | 2 072 953 | 42.65 ms | 54.82 ms (×1.29) | 258.4 ms | 270.7 ms (×1.05) |

Reading: the second `into_verified` costs one Ed25519 verification (about 200 µs here) plus one `body_root` (a Blake2b over the uncle bytes and the Merkle root over the transaction hashes, each hash a Blake2b over that transaction's ops). It is therefore visible on small blocks (×1.93 on an empty block) and on large bodies (+10 to +12 ms per 2 MiB, ×1.3 to ×1.45 for the `Unverified` type), and it is dwarfed by the per-transaction `preverify` cost on the deployed `Preverified` type, where 1 024 inscriptions cost 206 ms of Ed25519 verification before the block's own signature is checked (#512). Dropping the second pass saves 0.2 to 12 ms per synced block; it is a clean-up, not a bottleneck.

---

## Appendix B — What was checked and ruled out (static)

**Table 1. What `preverify` verifies per op at `3b7730a7`, against the spec's stateless checks**

| Op (opcode) | Proof type (code) | Proof type (spec 1.8.0) | `preverify` checks (file:lines) | Stateless checks the spec has that `preverify` lacks |
|---|---|---|---|---|
| `Transfer` (0x00) | `ZkSignature` | `ZkSigProof` | inputs non-empty, no zero-value output (`transfer.rs:108-118`) | none (ZK signature is deferred to the block batch, per § Batch verification) |
| `ChannelConfig` (0x10) | `ChannelMultiSigProof` | `ChannelConfigOpProof` | thresholds ≠ 0, keys non-empty (`config.rs:86-97`) | `configuration_threshold <= len(keys)`: known, #317 / #185 |
| `ChannelInscribe` (0x11) | `Ed25519Signature` | `Ed25519SigProof` | signer's signature over the tx hash (`inscribe.rs:110-114`) | none |
| `ChannelDeposit` (0x12) | `ZkSignature` | `ZkSigProof` | inputs non-empty (`deposit.rs:88-93`) | none |
| `ChannelWithdraw` (0x13) | `ChannelMultiSigProof` | `ChannelWithdrawOpProof` | inputs non-empty (`withdraw.rs:80-85`) | none (signatures need the channel state) |
| `ChannelTransfer` (0x14) | `ChannelMultiSigProof` | `ChannelTransferOpProof` | inputs non-empty, outputs valid (`channel_transfer.rs:88-98`) | none |
| `SDPDeclare` (0x20) | `ZkAndEd25519Proof` | `ZkAndEd25519SigsProof` | provider's Ed25519 signature (`sdp/mod.rs:514-518`); 1–8 locators enforced by the `NonEmptyBoundedVec<_, 8>` decoder (`sdp/mod.rs:476-477`) | none |
| `SDPWithdraw` (0x21) | `ZkSignature` | `ZkSigProof` | nothing (unit context, `signed_op.rs:152-155`) | none |
| `SDPActive` (0x22) | `ZkSignature` | `ZkSigProof` | nothing (`active.rs:55-57`) | none. The harness's first run found the payload itself off-spec: `Metadata = UINT32 *BYTE` in the encoding spec, a typed Blend activity proof (`0x01`, version `0x01`, epoch `u32`, signing key, proof of quota, proof of selection; `sdp/mod.rs:583-599`, `sdp/blend.rs:58-80`) in the code. Already filed as #245; the generator follows the code. |
| `LeaderClaim` (0x30) | `Groth16LeaderClaimProof` | `ProofOfClaimProof` | full Groth16 verification of the proof of claim with the op's nullifier and root and the tx hash (`leader_claim.rs:205-219`) | none |
| `ClaimPowReward` (0x40) | **`NoOpProof`** | **`ZkSigProof`** | nothing (`pow.rs:292-294`) | the ZK signature by `public_key`: **LB-001** |

**Table 2. Block path**

| Property | Result | Evidence |
|---|---|---|
| `Block::create` requires `leader_key == signing key public key` and a non-genesis slot | Holds. | `block/mod.rs:181-190`. Asserted by `block_gen.rs`. |
| `into_verified` checks version, non-genesis slot, total transaction size ≤ 2 MiB, `body_root`, header signature | Holds, in that order. | `:237-254`, `:343-361`. |
| `body_root` commits to the uncle list bytes (signatures included) and to the Merkle root of the transaction hashes; the transaction hash covers ops only | Holds; proofs are outside every commitment. | `:365-374`, `merkle.rs:14-30`, `signed_ops.rs:222-231`; spec § Block ("committing the uncle headers here is what makes two blocks with the same ID identical byte for byte" is therefore only true of the uncle list, not of the body): known, #313, #554. |
| `Block::try_from(Bytes)` verifies twice | Still true. | `:382` and `:384-386`; S-003. |
| A hand-built `Groth16LeaderProof` / `Header` needs no crate change | Holds. | S-002. |
| The bincode layout of `Block` carries no length prefix for fixed-size fields, so mutation offsets are computable | Holds. | `block/deser.rs:74-107` (test `test_bincode_fixed_size_fields_have_no_length_prefix`), used by `block_gen.rs`. |

## Appendix C — Harness source

Files are relative to the `logos-blockchain` root. Nothing else in the node is modified; `fuzz/` is its own workspace.

`fuzz/Cargo.toml`

```toml
[package]
name    = "logos-blockchain-fuzz"
version = "0.0.0"
edition = "2024"
publish = false

[package.metadata]
cargo-fuzz = true

# Its own workspace: the node's root manifest is not modified. The path
# dependencies below are members of the root workspace and inherit from it.
[workspace]
members = ["."]

[profile.release]
# The node's release profile (fat LTO, one codegen unit) is not needed for a
# harness; this keeps the fuzz and bench builds tractable on a 4-core machine.
codegen-units = 16
lto           = false
strip         = false

[dependencies]
arbitrary                     = { version = "1.4" }
bytes                         = { version = "1.3" }
lb-codec                      = { package = "logos-blockchain-codec", path = "../codec" }
lb-core                       = { package = "logos-blockchain-core", path = "../core" }
lb-cryptarchia-engine         = { package = "logos-blockchain-cryptarchia-engine", path = "../consensus/cryptarchia-engine" }
lb-groth16                    = { package = "logos-blockchain-groth16", path = "../zk/groth16" }
lb-key-management-system-keys = { package = "logos-blockchain-key-management-system-keys", path = "../kms/keys" }
lb-mmr                        = { package = "logos-blockchain-mmr", path = "../mmr" }
lb-utils                      = { package = "logos-blockchain-utils", path = "../utils" }
libfuzzer-sys                 = "0.4"

[[bin]]
name  = "signed_ops_gen"
path  = "fuzz_targets/signed_ops_gen.rs"
test  = false
doc   = false
bench = false

[[bin]]
name  = "block_gen"
path  = "fuzz_targets/block_gen.rs"
test  = false
doc   = false
bench = false
```

`fuzz/src/lib.rs`

```rust
//! Structure-aware generators for the paths behind the signature checks
//! (issue #167): transactions that pass `preverify()` and blocks that pass
//! `Block::try_from(Bytes)`.
//!
//! The generators write the canonical wire bytes described by
//! `mantle-transaction-encoding.md` (rev 1.8.0) from `arbitrary` input, decode
//! them through the node's own decoders, compute the transaction hash the node
//! computes, and sign it with fixed test keys. A generated value that the node
//! refuses to decode is therefore a disagreement between the encoding
//! specification (as read here) and the code, and the harness panics on it.

use std::sync::LazyLock;

use arbitrary::{Result as ArbResult, Unstructured};
use lb_codec::{BinaryDecode as _, BinaryEncode as _};
use lb_core::mantle::{
    ledger::verification_mode::StandardMode,
    traits::Hashable as _,
    transactions::{Ops, SignedOps, states::Unverified},
};
use lb_key_management_system_keys::keys::Ed25519Key;

pub const N_KEYS: usize = 4;

/// Fixed Ed25519 test keys: secret bytes `[i; 32]` for `i` in `1..=N_KEYS`.
pub static KEYS: LazyLock<Vec<Ed25519Key>> = LazyLock::new(|| {
    (1..=N_KEYS as u8)
        .map(|i| Ed25519Key::from_bytes(&[i; 32]))
        .collect()
});

pub static PUBKEYS: LazyLock<Vec<[u8; 32]>> =
    LazyLock::new(|| KEYS.iter().map(|k| k.public_key().to_bytes()).collect());

/// Transactions carrying a `LeaderClaim` with a valid proof of claim, produced
/// offline by `examples/gen_poc.rs` into the directory named by `POC_TXS_DIR`
/// (one `*.bin` file per transaction). The proof binds the transaction hash,
/// so these op lists cannot vary. Empty when the variable is unset.
pub static POC_TXS: LazyLock<Vec<Vec<u8>>> = LazyLock::new(|| {
    let Ok(dir) = std::env::var("POC_TXS_DIR") else {
        return Vec::new();
    };
    let mut files: Vec<_> = std::fs::read_dir(&dir)
        .map(|rd| rd.filter_map(Result::ok).map(|e| e.path()).collect())
        .unwrap_or_default();
    files.sort();
    files
        .into_iter()
        .filter(|p| p.extension().is_some_and(|e| e == "bin"))
        .filter_map(|p| std::fs::read(p).ok())
        .filter(|b| !b.is_empty())
        .collect()
});

/// Opcodes, `bedrock-v1.1-mantle-specification.md` § Opcodes.
pub const OP_TRANSFER: u8 = 0x00;
pub const OP_CHANNEL_CONFIG: u8 = 0x10;
pub const OP_CHANNEL_INSCRIBE: u8 = 0x11;
pub const OP_CHANNEL_DEPOSIT: u8 = 0x12;
pub const OP_CHANNEL_WITHDRAW: u8 = 0x13;
pub const OP_CHANNEL_TRANSFER: u8 = 0x14;
pub const OP_SDP_DECLARE: u8 = 0x20;
pub const OP_SDP_WITHDRAW: u8 = 0x21;
pub const OP_SDP_ACTIVE: u8 = 0x22;
pub const OP_LEADER_CLAIM: u8 = 0x30;
pub const OP_CLAIM_POW_REWARD: u8 = 0x40;

pub const ALL_OPCODES: [u8; 11] = [
    OP_TRANSFER,
    OP_CHANNEL_CONFIG,
    OP_CHANNEL_INSCRIBE,
    OP_CHANNEL_DEPOSIT,
    OP_CHANNEL_WITHDRAW,
    OP_CHANNEL_TRANSFER,
    OP_SDP_DECLARE,
    OP_SDP_WITHDRAW,
    OP_SDP_ACTIVE,
    OP_LEADER_CLAIM,
    OP_CLAIM_POW_REWARD,
];

/// What kind of proof an op takes and which test keys sign it.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum ProofKind {
    /// `Ed25519SigProof`, signed by `KEYS[key]`; checked by `preverify`.
    Ed25519 { key: usize },
    /// `ZkSigProof`, arbitrary bytes; not checked by `preverify`.
    Zk,
    /// `ZkAndEd25519SigsProof`: arbitrary ZK bytes + Ed25519 by `KEYS[key]`.
    ZkAndEd { key: usize },
    /// `ChannelMultiSigProof`: one signature per listed key index, strictly
    /// increasing; not checked by `preverify` at the audited commit.
    MultiSig { keys: Vec<usize> },
    /// `ProofOfClaimProof`, arbitrary bytes: `preverify` must reject it.
    PoCGarbage,
    /// No proof bytes (the code's `NoOpProof` for `ClaimPowReward`).
    NoOp,
}

/// A byte range of the encoded transaction.
pub type Range = (usize, usize);

#[derive(Debug, Clone)]
pub struct GeneratedTx {
    /// Canonical `SignedMantleTx` bytes: `OpCount *Op *OpProof`.
    pub bytes: Vec<u8>,
    /// Number of bytes covered by `MantleTx` (`OpCount *Op`), i.e. the
    /// transaction-hash preimage.
    pub ops_len: usize,
    pub opcodes: Vec<u8>,
    pub proofs: Vec<ProofKind>,
    /// Signature / proof ranges that `preverify` verifies (inscribe signer,
    /// SDP declare provider, a valid proof of claim).
    pub checked_sig_ranges: Vec<Range>,
    /// Proof ranges that `preverify` does not verify (ZK signatures, channel
    /// multi-signatures).
    pub unchecked_proof_ranges: Vec<Range>,
    /// Whether `preverify` is expected to accept the transaction.
    pub expect_preverify_ok: bool,
    /// Whether the transaction is one of the fixed `POC_TXS`.
    pub fixed_poc: bool,
}

pub fn fr32(u: &mut Unstructured<'_>) -> ArbResult<[u8; 32]> {
    // A BN254 scalar, little-endian. Masking the top byte keeps it below the
    // modulus (p < 2^254), which is what the `Fr` decoder requires.
    let mut b: [u8; 32] = u.arbitrary()?;
    b[31] &= 0x1F;
    Ok(b)
}

fn bytes32(u: &mut Unstructured<'_>) -> ArbResult<[u8; 32]> {
    u.arbitrary()
}

fn key_index(u: &mut Unstructured<'_>) -> ArbResult<usize> {
    u.int_in_range(0..=(N_KEYS - 1))
}

fn short_bytes(u: &mut Unstructured<'_>, max: usize) -> ArbResult<Vec<u8>> {
    let len = u.int_in_range(0..=max)?;
    let mut v = vec![0u8; len];
    u.fill_buffer(&mut v)?;
    Ok(v)
}

/// `Inputs = InputCount *NoteId`, non-empty (`preverify` rejects empty
/// inputs).
fn inputs(u: &mut Unstructured<'_>, out: &mut Vec<u8>) -> ArbResult<()> {
    let n = u.int_in_range(1..=4u8)?;
    out.push(n);
    for _ in 0..n {
        out.extend_from_slice(&fr32(u)?);
    }
    Ok(())
}

/// `Outputs = OutputCount *Note`, every value non-zero (`Outputs::validate`
/// rejects zero-value notes).
fn outputs(u: &mut Unstructured<'_>, out: &mut Vec<u8>) -> ArbResult<()> {
    let n = u.int_in_range(0..=4u8)?;
    out.push(n);
    for _ in 0..n {
        let value: u64 = u.int_in_range(1..=u64::MAX)?;
        out.extend_from_slice(&value.to_le_bytes());
        out.extend_from_slice(&fr32(u)?);
    }
    Ok(())
}

/// `Locator = 2Byte *BYTE`, multiaddr binary form: `/ip4/a.b.c.d/tcp/p`.
fn locator(u: &mut Unstructured<'_>, out: &mut Vec<u8>) -> ArbResult<()> {
    let mut ip: [u8; 4] = u.arbitrary()?;
    // `Locator::try_from` rejects the unspecified address 0.0.0.0
    // (`core/src/sdp/mod.rs:172-178`).
    ip[0] = ip[0].max(1);
    let port: [u8; 2] = u.arbitrary()?;
    let mut addr = vec![0x04];
    addr.extend_from_slice(&ip);
    addr.push(0x06);
    addr.extend_from_slice(&port);
    out.extend_from_slice(&(addr.len() as u16).to_le_bytes());
    out.extend_from_slice(&addr);
    Ok(())
}

/// Writes one `Op = Opcode OpPayload` and returns the proof it requires.
pub fn gen_op(u: &mut Unstructured<'_>, opcode: u8, out: &mut Vec<u8>) -> ArbResult<ProofKind> {
    out.push(opcode);
    match opcode {
        OP_TRANSFER => {
            inputs(u, out)?;
            outputs(u, out)?;
            Ok(ProofKind::Zk)
        }
        OP_CHANNEL_CONFIG => {
            out.extend_from_slice(&bytes32(u)?); // ChannelId
            out.extend_from_slice(&bytes32(u)?); // Parent
            let n_keys = u.int_in_range(1..=6u16)?;
            out.extend_from_slice(&n_keys.to_le_bytes());
            for _ in 0..n_keys {
                out.extend_from_slice(&PUBKEYS[key_index(u)?]);
            }
            out.extend_from_slice(&u.arbitrary::<u32>()?.to_le_bytes()); // PostingTimeframe
            out.extend_from_slice(&u.arbitrary::<u32>()?.to_le_bytes()); // PostingTimeout
            // `preverify` only requires the thresholds to be non-zero.
            out.extend_from_slice(&u.int_in_range(1..=u16::MAX)?.to_le_bytes());
            out.extend_from_slice(&u.int_in_range(1..=u16::MAX)?.to_le_bytes());
            Ok(ProofKind::MultiSig {
                keys: multisig_keys(u)?,
            })
        }
        OP_CHANNEL_INSCRIBE => {
            out.extend_from_slice(&bytes32(u)?); // ChannelId
            let inscription = short_bytes(u, 256)?;
            out.extend_from_slice(&(inscription.len() as u32).to_le_bytes());
            out.extend_from_slice(&inscription);
            let parent = if u.arbitrary::<bool>()? {
                [0u8; 32]
            } else {
                bytes32(u)?
            };
            out.extend_from_slice(&parent);
            let key = key_index(u)?;
            out.extend_from_slice(&PUBKEYS[key]); // Signer
            Ok(ProofKind::Ed25519 { key })
        }
        OP_CHANNEL_DEPOSIT => {
            out.extend_from_slice(&bytes32(u)?);
            inputs(u, out)?;
            let metadata = short_bytes(u, 64)?;
            out.extend_from_slice(&(metadata.len() as u32).to_le_bytes());
            out.extend_from_slice(&metadata);
            Ok(ProofKind::Zk)
        }
        OP_CHANNEL_WITHDRAW => {
            out.extend_from_slice(&bytes32(u)?);
            inputs(u, out)?;
            Ok(ProofKind::MultiSig {
                keys: multisig_keys(u)?,
            })
        }
        OP_CHANNEL_TRANSFER => {
            out.extend_from_slice(&bytes32(u)?);
            inputs(u, out)?;
            outputs(u, out)?;
            Ok(ProofKind::MultiSig {
                keys: multisig_keys(u)?,
            })
        }
        OP_SDP_DECLARE => {
            out.push(0); // ServiceType BN
            let n = u.int_in_range(1..=3u8)?;
            out.push(n);
            for _ in 0..n {
                locator(u, out)?;
            }
            let key = key_index(u)?;
            out.extend_from_slice(&PUBKEYS[key]); // ProviderId
            out.extend_from_slice(&fr32(u)?); // ZkId
            out.extend_from_slice(&fr32(u)?); // ServiceNoteId
            Ok(ProofKind::ZkAndEd { key })
        }
        OP_SDP_WITHDRAW => {
            out.extend_from_slice(&bytes32(u)?); // DeclarationId
            out.extend_from_slice(&u.arbitrary::<u64>()?.to_le_bytes()); // Nonce
            out.extend_from_slice(&fr32(u)?); // ServiceNoteId
            Ok(ProofKind::Zk)
        }
        OP_SDP_ACTIVE => {
            out.extend_from_slice(&bytes32(u)?);
            out.extend_from_slice(&u.arbitrary::<u64>()?.to_le_bytes());
            // The encoding spec says `Metadata = UINT32 *BYTE`; the code
            // (`core/src/sdp/mod.rs:583-599`, `sdp/blend.rs:58-80`) decodes a
            // typed Blend activity proof instead (known: #245):
            // type 0x01, version 0x01, epoch u32, signing key, proof of quota
            // (key nullifier Fr + 128 proof bytes), proof of selection (Fr).
            out.push(0x01);
            out.push(0x01);
            out.extend_from_slice(&u.arbitrary::<u32>()?.to_le_bytes());
            out.extend_from_slice(&PUBKEYS[key_index(u)?]);
            out.extend_from_slice(&fr32(u)?);
            let poq: [u8; 128] = u.arbitrary()?;
            out.extend_from_slice(&poq);
            out.extend_from_slice(&fr32(u)?);
            Ok(ProofKind::Zk)
        }
        OP_LEADER_CLAIM => {
            out.extend_from_slice(&fr32(u)?); // RewardsRoot
            out.extend_from_slice(&fr32(u)?); // VoucherNullifier
            out.extend_from_slice(&fr32(u)?); // PublicKey
            Ok(ProofKind::PoCGarbage)
        }
        OP_CLAIM_POW_REWARD => {
            out.extend_from_slice(&fr32(u)?); // EpochNonce
            out.extend_from_slice(&bytes32(u)?); // BlockHash
            out.extend_from_slice(&fr32(u)?); // PublicKey
            // The code pairs this op with `NoOpProof` (0 bytes); the
            // encoding specification (rev 1.8.0) says `ZkSigProof`.
            Ok(ProofKind::NoOp)
        }
        _ => unreachable!("unknown opcode"),
    }
}

fn multisig_keys(u: &mut Unstructured<'_>) -> ArbResult<Vec<usize>> {
    let mut keys = Vec::new();
    for i in 0..N_KEYS {
        if u.arbitrary::<bool>()? {
            keys.push(i);
        }
    }
    Ok(keys)
}

/// Appends the proof for `kind` over `msg` (the transaction hash bytes) and
/// records the ranges the harness will later flip.
fn gen_proof(
    u: &mut Unstructured<'_>,
    kind: &ProofKind,
    msg: &[u8],
    out: &mut Vec<u8>,
    checked: &mut Vec<Range>,
    unchecked: &mut Vec<Range>,
) -> ArbResult<()> {
    match kind {
        ProofKind::Ed25519 { key } => {
            let start = out.len();
            out.extend_from_slice(&KEYS[*key].sign_payload(msg).to_bytes());
            checked.push((start, out.len()));
        }
        ProofKind::Zk => {
            let start = out.len();
            let zk: [u8; 128] = u.arbitrary()?;
            out.extend_from_slice(&zk);
            unchecked.push((start, out.len()));
        }
        ProofKind::ZkAndEd { key } => {
            let start = out.len();
            let zk: [u8; 128] = u.arbitrary()?;
            out.extend_from_slice(&zk);
            unchecked.push((start, out.len()));
            let start = out.len();
            out.extend_from_slice(&KEYS[*key].sign_payload(msg).to_bytes());
            checked.push((start, out.len()));
        }
        ProofKind::MultiSig { keys } => {
            let start = out.len();
            out.extend_from_slice(&(keys.len() as u16).to_le_bytes());
            for key in keys {
                out.extend_from_slice(&KEYS[*key].sign_payload(msg).to_bytes());
                out.extend_from_slice(&(*key as u16).to_le_bytes());
            }
            if !keys.is_empty() {
                // Only the signature bytes: flipping an index byte changes
                // the structure (ordering) and is a different property.
                unchecked.push((start + 2, start + 2 + 64));
            }
        }
        ProofKind::PoCGarbage => {
            let zk: [u8; 128] = u.arbitrary()?;
            out.extend_from_slice(&zk);
        }
        ProofKind::NoOp => {}
    }
    Ok(())
}

/// Generates one signed transaction. With `allow_leader_claim`, a garbage
/// proof of claim may be produced (`expect_preverify_ok == false`) and one of
/// the fixed valid `POC_TXS` may be returned instead of a fresh transaction.
pub fn gen_tx(u: &mut Unstructured<'_>, allow_leader_claim: bool) -> ArbResult<GeneratedTx> {
    if allow_leader_claim && !POC_TXS.is_empty() && u.ratio(1u8, 8u8)? {
        let bytes = u.choose(POC_TXS.as_slice())?;
        let ops = ops_len_of(bytes);
        return Ok(GeneratedTx {
            bytes: bytes.clone(),
            ops_len: ops,
            opcodes: vec![OP_LEADER_CLAIM],
            proofs: vec![ProofKind::PoCGarbage],
            checked_sig_ranges: vec![(ops, ops + 128)],
            unchecked_proof_ranges: vec![],
            expect_preverify_ok: true,
            fixed_poc: true,
        });
    }

    let n_ops = u.int_in_range(0..=6u8)?;
    let mut ops_bytes = vec![n_ops];
    let mut opcodes = Vec::with_capacity(n_ops as usize);
    let mut proofs = Vec::with_capacity(n_ops as usize);
    let mut expect_preverify_ok = true;
    for _ in 0..n_ops {
        let mut opcode = *u.choose(&ALL_OPCODES)?;
        if opcode == OP_LEADER_CLAIM && !allow_leader_claim {
            // Substitute an op that always preverifies.
            opcode = OP_SDP_ACTIVE;
        }
        let kind = gen_op(u, opcode, &mut ops_bytes)?;
        if kind == ProofKind::PoCGarbage {
            expect_preverify_ok = false;
        }
        opcodes.push(opcode);
        proofs.push(kind);
    }

    let ops = Ops::decode_all(&ops_bytes, &()).unwrap_or_else(|e| {
        panic!("spec-encoded MantleTx rejected by Ops::decode_all: {e:?} (opcodes {opcodes:x?})")
    });
    let msg = ops.hash().as_signing_bytes();

    let mut bytes = ops_bytes.clone();
    let mut checked = Vec::new();
    let mut unchecked = Vec::new();
    for kind in &proofs {
        gen_proof(u, kind, &msg, &mut bytes, &mut checked, &mut unchecked)?;
    }

    Ok(GeneratedTx {
        bytes,
        ops_len: ops_bytes.len(),
        opcodes,
        proofs,
        checked_sig_ranges: checked,
        unchecked_proof_ranges: unchecked,
        expect_preverify_ok,
        fixed_poc: false,
    })
}

/// Length of the `MantleTx` prefix of an encoded signed transaction.
pub fn ops_len_of(bytes: &[u8]) -> usize {
    let (rest, _) = Ops::decode(bytes, &()).expect("transaction prefix must decode");
    bytes.len() - rest.len()
}

/// Decodes a generated transaction through the node's decoder.
pub fn decode_signed(bytes: &[u8]) -> SignedOps<Unverified, StandardMode> {
    SignedOps::<Unverified, StandardMode>::decode_all(bytes, &()).unwrap_or_else(|e| {
        panic!("spec-encoded SignedMantleTx rejected by SignedOps::decode_all: {e:?}")
    })
}

/// The bincode envelope the transaction gossip topic carries: a `Vec<u8>`
/// (u64 little-endian length + bytes).
pub fn gossip_envelope(bytes: &[u8]) -> Vec<u8> {
    let mut out = (bytes.len() as u64).to_le_bytes().to_vec();
    out.extend_from_slice(bytes);
    out
}

/// A copy of `bytes` with one bit of the byte at `at` flipped.
pub fn flipped(bytes: &[u8], at: usize) -> Vec<u8> {
    let mut v = bytes.to_vec();
    v[at] ^= 0x01;
    v
}

/// Chooses a few offsets inside `range` to flip: first, last and one in
/// between.
pub fn flip_offsets(u: &mut Unstructured<'_>, (start, end): Range) -> ArbResult<Vec<usize>> {
    let mut v = vec![start, end - 1];
    if end - start > 2 {
        v.push(u.int_in_range(start + 1..=end - 2)?);
    }
    Ok(v)
}

pub fn encoded(tx: &SignedOps<Unverified, StandardMode>) -> Vec<u8> {
    tx.encode_to_vec()
}
```

`fuzz/fuzz_targets/signed_ops_gen.rs`

```rust
#![no_main]
//! Issue #167, items 1 and 2: generated transactions must decode, be
//! canonical, pass `preverify()`, and fail it when any verified signature
//! byte is flipped. Flips of proof bytes that `preverify` does not verify
//! (ZK signatures, channel multi-signatures) must leave the verdict unchanged.

use arbitrary::Unstructured;
use lb_codec::{BinaryDecode as _, BinaryEncode as _};
use lb_core::{
    codec::DeserializeOp as _,
    mantle::{
        VerificationError,
        ledger::verification_mode::StandardMode,
        traits::{Hashable as _, StorageSize as _},
        transactions::{
            Ops, SignedOps,
            states::{Preverified, Unverified},
        },
    },
};
use libfuzzer_sys::fuzz_target;
use logos_blockchain_fuzz::{
    decode_signed, encoded, flip_offsets, flipped, gen_tx, gossip_envelope,
};

fuzz_target!(|data: &[u8]| {
    let mut u = Unstructured::new(data);
    let Ok(tx) = gen_tx(&mut u, true) else {
        return;
    };

    // Decoding and canonicality.
    let decoded = decode_signed(&tx.bytes);
    assert_eq!(encoded(&decoded), tx.bytes, "encode(decode(x)) != x");
    assert_eq!(
        decoded.encoded_length(),
        tx.bytes.len(),
        "encoded_length() != encode().len()"
    );
    assert_eq!(
        decoded.storage_size(),
        tx.bytes.len(),
        "storage_size() != encode().len()"
    );
    let (rest, partial) = SignedOps::<Unverified, StandardMode>::decode(&tx.bytes, &())
        .expect("decode agrees with decode_all");
    assert!(rest.is_empty());
    assert_eq!(partial, decoded);
    assert_eq!(decoded.len(), tx.opcodes.len());

    // The transaction hash covers the ops only; the proofs are outside it.
    let hash = decoded.hash();
    assert_eq!(
        hash,
        Ops::decode_all(&tx.bytes[..tx.ops_len], &())
            .unwrap()
            .hash()
    );

    // preverify on the value and through the gossip (bincode) envelope.
    let envelope = gossip_envelope(&tx.bytes);
    let verdict = decoded.clone().preverify();
    let via_envelope = SignedOps::<Preverified, StandardMode>::from_bytes(&envelope);
    if tx.expect_preverify_ok {
        let pre = verdict.unwrap_or_else(|e| {
            panic!(
                "preverify rejected a generated transaction: {e:?} (opcodes {:x?})",
                tx.opcodes
            )
        });
        let env = via_envelope.expect("gossip envelope decode must agree with preverify");
        assert_eq!(env, pre);
        assert_eq!(pre.storage_size(), tx.bytes.len());
        assert_eq!(pre.encode_to_vec(), tx.bytes);
        assert_eq!(pre.hash(), hash);
    } else {
        assert!(
            matches!(
                verdict,
                Err(VerificationError::LeaderClaimVerificationError(_))
            ),
            "a garbage proof of claim must be rejected: {verdict:?}"
        );
        assert!(via_envelope.is_err());
        return;
    }

    // Flipping any byte of a signature preverify verifies must fail it, and
    // the flipped bytes must still decode (a signature is an opaque 64-byte
    // string; a 128-byte proof of claim likewise).
    for range in &tx.checked_sig_ranges {
        for at in flip_offsets(&mut u, *range).unwrap_or_default() {
            let mutated = flipped(&tx.bytes, at);
            let value = decode_signed(&mutated);
            assert_eq!(value.hash(), hash, "a proof flip must not change the tx hash");
            let verdict = value.preverify();
            assert!(
                verdict.is_err(),
                "flipped verified signature byte {at} still preverifies (opcodes {:x?})",
                tx.opcodes
            );
            assert!(
                SignedOps::<Preverified, StandardMode>::from_bytes(&gossip_envelope(&mutated))
                    .is_err()
            );
        }
    }

    // Flipping a byte of a proof preverify does not verify leaves the verdict
    // unchanged: the ZK signatures are deferred to block validation and the
    // channel multi-signatures need the channel state.
    for range in &tx.unchecked_proof_ranges {
        for at in flip_offsets(&mut u, *range).unwrap_or_default() {
            let mutated = flipped(&tx.bytes, at);
            let value = decode_signed(&mutated);
            assert_eq!(value.hash(), hash);
            assert!(
                value.preverify().is_ok(),
                "flip of an unverified proof byte {at} changed the preverify verdict"
            );
        }
    }

    // Flipping a byte of the MantleTx prefix changes the hash, so every
    // signature preverify verifies must fail (or the bytes must fail to
    // decode); a transaction with no verified signature is accepted as a
    // different transaction.
    if tx.ops_len > 1 && !tx.fixed_poc {
        let at = u.int_in_range(1..=tx.ops_len - 1).unwrap_or(1);
        let mutated = flipped(&tx.bytes, at);
        if let Ok(value) = SignedOps::<Unverified, StandardMode>::decode_all(&mutated, &())
            && value.hash() != hash
            && !tx.checked_sig_ranges.is_empty()
        {
            assert!(
                value.preverify().is_err(),
                "ops byte {at} flipped, hash changed, but the signatures still verify"
            );
        }
    }
});
```

`fuzz/fuzz_targets/block_gen.rs`

```rust
#![no_main]
//! Issue #167, item 3: generated blocks signed by a test key whose public key
//! is the `leader_key` of a hand-built proof of leadership must pass
//! `Block::create` and `Block::try_from(Bytes)` for both transaction types;
//! independent mutations of the header, the signature, the uncle headers and
//! the transactions must be rejected, except for transaction *proof* bytes,
//! which `body_root` does not commit to (#313, #554).

use arbitrary::Unstructured;
use bytes::Bytes;
use lb_codec::{BinaryDecode as _, BinaryEncode as _};
use lb_core::{
    block::{Block, BlockTransactions, Error as BlockError, SignedHeader, UncleHeaders},
    codec::SerializeOp as _,
    header::Header,
    mantle::{
        ledger::verification_mode::StandardMode,
        transactions::{
            SignedOps,
            states::{Preverified, Unverified},
        },
    },
    proofs::leader_proof::Groth16LeaderProof,
};
use lb_cryptarchia_engine::{MAX_UNCLES, Slot};
use lb_utils::bounded::UpperBoundedVec;
use libfuzzer_sys::fuzz_target;
use logos_blockchain_fuzz::{KEYS, N_KEYS, PUBKEYS, decode_signed, fr32, gen_tx, ops_len_of};

const HEADER_LEN: usize = 297;
const SIG_LEN: usize = 64;
const BODY_ROOT: (usize, usize) = (41, 73);
const LEADER_KEY: (usize, usize) = (73 + 128 + 32, 73 + 128 + 32 + 32);

type UTx = SignedOps<Unverified, StandardMode>;
type PTx = SignedOps<Preverified, StandardMode>;

fn pol_bytes(u: &mut Unstructured<'_>, leader: usize) -> arbitrary::Result<Vec<u8>> {
    let mut v = Vec::with_capacity(224);
    let proof: [u8; 128] = u.arbitrary()?;
    v.extend_from_slice(&proof);
    v.extend_from_slice(&fr32(u)?); // entropy contribution
    v.extend_from_slice(&PUBKEYS[leader]); // leader key
    v.extend_from_slice(&fr32(u)?); // voucher cm
    Ok(v)
}

fn uncle(u: &mut Unstructured<'_>) -> arbitrary::Result<SignedHeader> {
    let key = u.int_in_range(0..=N_KEYS - 1)?;
    let mut h = vec![0x01];
    h.extend_from_slice(&u.arbitrary::<[u8; 32]>()?); // parent
    h.extend_from_slice(&u.int_in_range(1..=u64::MAX)?.to_le_bytes()); // slot
    h.extend_from_slice(&u.arbitrary::<[u8; 32]>()?); // body root
    h.extend_from_slice(&pol_bytes(u, key)?);
    assert_eq!(h.len(), HEADER_LEN);
    let header = Header::decode_all(&h, &()).expect("spec-encoded header must decode");
    let signature = header.sign(&KEYS[key]).expect("signing cannot fail");
    Ok(SignedHeader::new(header, signature))
}

fn rejects(bytes: &[u8]) -> bool {
    let b = Bytes::copy_from_slice(bytes);
    Block::<UTx>::try_from(b.clone()).is_err() && Block::<PTx>::try_from(b).is_err()
}

fuzz_target!(|data: &[u8]| {
    let mut u = Unstructured::new(data);
    let Ok(n_tx) = u.int_in_range(0..=6usize) else {
        return;
    };
    let mut txs: Vec<UTx> = Vec::with_capacity(n_tx);
    let mut ranges: Vec<(Vec<(usize, usize)>, Vec<(usize, usize)>)> = Vec::new();
    for _ in 0..n_tx {
        let Ok(tx) = gen_tx(&mut u, false) else {
            return;
        };
        assert!(tx.expect_preverify_ok);
        ranges.push((
            tx.checked_sig_ranges.clone(),
            tx.unchecked_proof_ranges.clone(),
        ));
        txs.push(decode_signed(&tx.bytes));
    }
    let Ok(n_uncles) = u.int_in_range(0..=MAX_UNCLES) else {
        return;
    };
    let mut uncles = Vec::with_capacity(n_uncles);
    for _ in 0..n_uncles {
        let Ok(h) = uncle(&mut u) else {
            return;
        };
        uncles.push(h);
    }
    let uncle_headers =
        UncleHeaders::new(UpperBoundedVec::<SignedHeader, MAX_UNCLES>::try_from(uncles).unwrap());
    let Ok(leader) = u.int_in_range(0..=N_KEYS - 1) else {
        return;
    };
    let Ok(pol) = pol_bytes(&mut u, leader) else {
        return;
    };
    let pol = Groth16LeaderProof::decode_all(&pol, &())
        .expect("spec-encoded proof of leadership must decode");
    let Ok(slot) = u.int_in_range(1..=u64::MAX) else {
        return;
    };
    let Ok(parent) = u.arbitrary::<[u8; 32]>() else {
        return;
    };
    let transactions = BlockTransactions::<UTx>::try_from(txs.clone()).unwrap();

    // Negative construction cases.
    let wrong = (leader + 1) % N_KEYS;
    assert!(matches!(
        Block::create(
            parent.into(),
            Slot::from(slot),
            uncle_headers.clone(),
            pol.clone(),
            transactions.clone(),
            &KEYS[wrong]
        ),
        Err(BlockError::KeyMismatch)
    ));
    assert!(matches!(
        Block::create(
            parent.into(),
            Slot::genesis(),
            uncle_headers.clone(),
            pol.clone(),
            transactions.clone(),
            &KEYS[leader]
        ),
        Err(BlockError::Header(_))
    ));

    let block = Block::create(
        parent.into(),
        Slot::from(slot),
        uncle_headers.clone(),
        pol.clone(),
        transactions.clone(),
        &KEYS[leader],
    )
    .expect("a block signed by the leader key must be created");
    let bytes = block.to_bytes().expect("serialize");

    // Both ingress types accept it and re-serialize byte for byte.
    let as_unverified =
        Block::<UTx>::try_from(bytes.clone()).expect("try_from must accept the block (Unverified)");
    assert_eq!(as_unverified.to_bytes().unwrap(), bytes);
    assert_eq!(as_unverified.header().id(), block.header().id());
    assert_eq!(as_unverified.transactions().len(), n_tx);
    assert_eq!(as_unverified.uncle_headers().len(), n_uncles);
    let as_preverified = Block::<PTx>::try_from(bytes.clone())
        .expect("try_from must accept the block (Preverified)");
    assert_eq!(as_preverified.to_bytes().unwrap(), bytes);
    // `reconstruct` from the parts agrees.
    let re = Block::<UTx>::reconstruct(
        block.header().clone(),
        uncle_headers.clone(),
        transactions.clone(),
        *block.signature(),
    )
    .expect("reconstruct");
    assert_eq!(re.to_bytes().unwrap(), bytes);

    // Layout of the bincode encoding (fixed-size fields carry no prefix).
    let sig_at = HEADER_LEN;
    let uncles_count_at = sig_at + SIG_LEN;
    let uncles_at = uncles_count_at + 8;
    let uncles_end = uncles_at + n_uncles * (HEADER_LEN + SIG_LEN);
    let tx_count_at = uncles_end;
    let txs_at = tx_count_at + 8;
    assert_eq!(
        &bytes[uncles_count_at..uncles_at],
        &(n_uncles as u64).to_le_bytes()
    );
    assert_eq!(&bytes[tx_count_at..txs_at], &(n_tx as u64).to_le_bytes());

    let flip = |at: usize| {
        let mut v = bytes.to_vec();
        v[at] ^= 0x01;
        v
    };

    // 1. Header: version, slot, body_root, leader key: all rejected.
    assert!(rejects(&flip(0)), "flipped version accepted");
    assert!(rejects(&flip(33)), "flipped slot accepted");
    let Ok(at) = u.int_in_range(BODY_ROOT.0..=BODY_ROOT.1 - 1) else {
        return;
    };
    assert!(rejects(&flip(at)), "flipped body_root byte {at} accepted");
    let Ok(at) = u.int_in_range(LEADER_KEY.0..=LEADER_KEY.1 - 1) else {
        return;
    };
    assert!(rejects(&flip(at)), "flipped leader_key byte {at} accepted");
    // 2. Signature.
    let Ok(at) = u.int_in_range(sig_at..=sig_at + SIG_LEN - 1) else {
        return;
    };
    assert!(rejects(&flip(at)), "flipped signature byte {at} accepted");
    // 3. Uncle headers: any byte, including an uncle's own signature.
    if n_uncles > 0 {
        let Ok(at) = u.int_in_range(uncles_at..=uncles_end - 1) else {
            return;
        };
        assert!(rejects(&flip(at)), "flipped uncle byte {at} accepted");
        // A dropped uncle: body root mismatch through `reconstruct`.
        let fewer = UncleHeaders::new(
            UpperBoundedVec::<SignedHeader, MAX_UNCLES>::try_from(
                uncle_headers.iter().skip(1).cloned().collect::<Vec<_>>(),
            )
            .unwrap(),
        );
        assert!(matches!(
            Block::<UTx>::reconstruct(
                block.header().clone(),
                fewer,
                transactions.clone(),
                *block.signature()
            ),
            Err(BlockError::BodyRootMismatch)
        ));
    }
    // 4. Transactions: op bytes are committed by body_root; proof bytes are not.
    let mut cursor = txs_at;
    for (i, tx) in txs.iter().enumerate() {
        let enc = tx.encode_to_vec();
        let len_at = cursor;
        let body_at = cursor + 8;
        assert_eq!(&bytes[len_at..body_at], &(enc.len() as u64).to_le_bytes());
        assert_eq!(&bytes[body_at..body_at + enc.len()], &enc[..]);
        let ops_len = ops_len_of(&enc);
        // An op byte flip changes the tx hash (or breaks decoding): rejected.
        if ops_len > 1 {
            let Ok(off) = u.int_in_range(1..=ops_len - 1) else {
                return;
            };
            assert!(
                rejects(&flip(body_at + off)),
                "flipped op byte of tx {i} accepted"
            );
        }
        // A verified-signature flip: rejected by the Preverified type only
        // (its `Deserialize` runs `preverify`), accepted by the Unverified one.
        let (checked, unchecked) = &ranges[i];
        if let Some((s, _)) = checked.first() {
            let m = flip(body_at + s);
            assert!(
                Block::<PTx>::try_from(Bytes::from(m.clone())).is_err(),
                "flipped verified signature of tx {i} accepted as Preverified"
            );
            assert!(
                Block::<UTx>::try_from(Bytes::from(m)).is_ok(),
                "flipped verified signature of tx {i} rejected as Unverified: body_root does not cover proofs"
            );
        }
        // An unverified-proof flip: accepted by both (same block ID, different bytes).
        if let Some((s, _)) = unchecked.first() {
            let m = flip(body_at + s);
            let accepted = Block::<UTx>::try_from(Bytes::from(m.clone()))
                .expect("proof-mangled copy accepted (Unverified): body_root does not cover proofs");
            assert_eq!(accepted.header().id(), block.header().id());
            assert_ne!(accepted.to_bytes().unwrap(), bytes);
            assert!(
                Block::<PTx>::try_from(Bytes::from(m)).is_ok(),
                "proof-mangled copy rejected (Preverified)"
            );
        }
        cursor = body_at + enc.len();
    }
    assert_eq!(cursor, bytes.len(), "trailing bytes in the block encoding");
    // A reordered transaction list: body root mismatch.
    if n_tx >= 2 && txs[0] != txs[n_tx - 1] {
        let mut rev = txs.clone();
        rev.reverse();
        assert!(matches!(
            Block::<UTx>::reconstruct(
                block.header().clone(),
                uncle_headers.clone(),
                BlockTransactions::try_from(rev).unwrap(),
                *block.signature()
            ),
            Err(BlockError::BodyRootMismatch)
        ));
    }
    // Trailing byte: rejected.
    let mut trailing = bytes.to_vec();
    trailing.push(0);
    assert!(rejects(&trailing), "trailing byte accepted");
});
```

`fuzz/examples/gen_poc.rs`

```rust
//! Produces transactions carrying one `LeaderClaim` with a valid proof of
//! claim, for the `signed_ops_gen` target (`POC_TXS_DIR`). Usage:
//! `gen_poc <out_dir> [count]`.

use lb_codec::BinaryEncode as _;
use lb_core::{
    crypto::ZkHasher,
    mantle::{
        Op, OpProof,
        ledger::verification_mode::StandardMode,
        ops::leader_claim::{
            LeaderClaimOp, RewardsRoot, VoucherCm, VoucherNullifier, VoucherSecret,
        },
        traits::Hashable as _,
        transactions::{OpProofs, Ops, SignedOps},
    },
    proofs::leader_claim_proof::{Groth16LeaderClaimProof, LeaderClaimPrivate, LeaderClaimPublic},
};
use lb_groth16::Fr;
use lb_key_management_system_keys::keys::ZkPublicKey;
use lb_mmr::MerkleMountainRange;

fn main() {
    let mut args = std::env::args().skip(1);
    let out_dir = args.next().expect("usage: gen_poc <out_dir> [count]");
    let count: u64 = args.next().map(|c| c.parse().unwrap()).unwrap_or(4);
    std::fs::create_dir_all(&out_dir).unwrap();

    for i in 0..count {
        let started = std::time::Instant::now();
        let secret = VoucherSecret::from(Fr::from(1000 + i));
        let cm = VoucherCm::from_secret(secret);
        let (mmr, path) = MerkleMountainRange::<VoucherCm, ZkHasher>::new()
            .push_with_paths(cm, &mut [])
            .expect("mmr");
        let root = RewardsRoot::from(mmr.frontier_root());
        let nf = VoucherNullifier::from_secret(secret);
        let op = LeaderClaimOp {
            rewards_root: root,
            voucher_nullifier: nf,
            pk: ZkPublicKey::from(Fr::from(7 + i)),
        };
        let ops = Ops::from([Op::LeaderClaim(op)]);
        let tx_hash = ops.hash();
        let private = LeaderClaimPrivate::try_new(
            LeaderClaimPublic::new(nf.into(), root.into(), tx_hash.to_fr()),
            &path,
            secret,
        )
        .expect("voucher path");
        let proof = Groth16LeaderClaimProof::prove(private).expect("prove");
        let signed = SignedOps::<_, StandardMode>::from_parts(ops, OpProofs::from([OpProof::PoC(proof)]))
            .expect("from_parts");
        let bytes = signed.encode_to_vec();
        signed.clone().preverify().expect("the generated proof of claim must preverify");
        std::fs::write(format!("{out_dir}/{i}.bin"), &bytes).unwrap();
        println!(
            "{i}: {} bytes, proved and preverified in {:.2?}",
            bytes.len(),
            started.elapsed()
        );
    }
}
```

`fuzz/examples/bench_block.rs`

```rust
//! Issue #167, item 4: the cost of `Block::try_from(Bytes)` verifying the
//! block twice (`Deserialize` -> `reconstruct` -> `into_verified`, then
//! `into_verified` again) against `Block::from_bytes` alone, for both
//! transaction types and several block shapes.

use std::time::{Duration, Instant};

use arbitrary::Unstructured;
use bytes::Bytes;
use lb_codec::BinaryDecode as _;
use lb_core::{
    block::{Block, BlockTransactions, UncleHeaders},
    codec::{DeserializeOp as _, SerializeOp as _},
    mantle::{
        ledger::verification_mode::StandardMode,
        transactions::{
            SignedOps,
            states::{Preverified, Unverified},
        },
    },
    proofs::leader_proof::Groth16LeaderProof,
};
use lb_cryptarchia_engine::Slot;
use logos_blockchain_fuzz::{
    KEYS, OP_CHANNEL_INSCRIBE, OP_SDP_ACTIVE, OP_TRANSFER, PUBKEYS, decode_signed, gen_op,
};

type UTx = SignedOps<Unverified, StandardMode>;
type PTx = SignedOps<Preverified, StandardMode>;

/// One transaction of a fixed shape, signed by the test keys.
fn tx(seed: u64, opcode: u8, inscription_len: usize) -> UTx {
    let mut data = vec![0u8; 4096 + inscription_len];
    data.iter_mut()
        .enumerate()
        .for_each(|(i, b)| *b = (seed as usize).wrapping_mul(31).wrapping_add(i * 7) as u8);
    let mut u = Unstructured::new(&data);
    let mut bytes = vec![1u8];
    let kind = if opcode == OP_CHANNEL_INSCRIBE {
        // ChannelInscribe with an inscription of the requested length.
        bytes.push(opcode);
        bytes.extend_from_slice(&[seed as u8; 32]);
        bytes.extend_from_slice(&(inscription_len as u32).to_le_bytes());
        bytes.extend(std::iter::repeat_n(0xABu8, inscription_len));
        bytes.extend_from_slice(&[0u8; 32]);
        bytes.extend_from_slice(&PUBKEYS[0]);
        logos_blockchain_fuzz::ProofKind::Ed25519 { key: 0 }
    } else {
        gen_op(&mut u, opcode, &mut bytes).unwrap()
    };
    use lb_core::mantle::traits::Hashable as _;
    let ops = lb_core::mantle::transactions::Ops::decode_all(&bytes, &()).unwrap();
    let msg = ops.hash().as_signing_bytes();
    match kind {
        logos_blockchain_fuzz::ProofKind::Ed25519 { key } => {
            bytes.extend_from_slice(&KEYS[key].sign_payload(&msg).to_bytes());
        }
        logos_blockchain_fuzz::ProofKind::Zk => bytes.extend_from_slice(&[0x11u8; 128]),
        other => panic!("unsupported shape {other:?}"),
    }
    decode_signed(&bytes)
}

fn block(txs: Vec<UTx>) -> Bytes {
    let mut pol = vec![0u8; 128];
    pol.extend_from_slice(&[0u8; 32]);
    pol.extend_from_slice(&PUBKEYS[0]);
    pol.extend_from_slice(&[0u8; 32]);
    let pol = Groth16LeaderProof::decode_all(&pol, &()).unwrap();
    let block = Block::create(
        [9u8; 32].into(),
        Slot::from(42u64),
        UncleHeaders::empty(),
        pol,
        BlockTransactions::try_from(txs).unwrap(),
        &KEYS[0],
    )
    .unwrap();
    block.to_bytes().unwrap()
}

fn median(mut v: Vec<Duration>) -> Duration {
    v.sort();
    v[v.len() / 2]
}

fn measure(bytes: &Bytes, iters: usize) -> [Duration; 4] {
    let mut one_u = Vec::with_capacity(iters);
    let mut two_u = Vec::with_capacity(iters);
    let mut one_p = Vec::with_capacity(iters);
    let mut two_p = Vec::with_capacity(iters);
    for _ in 0..iters {
        let t = Instant::now();
        let b = Block::<UTx>::from_bytes(bytes).unwrap();
        one_u.push(t.elapsed());
        std::hint::black_box(b);
        let t = Instant::now();
        let b = Block::<UTx>::try_from(bytes.clone()).unwrap();
        two_u.push(t.elapsed());
        std::hint::black_box(b);
        let t = Instant::now();
        let b = Block::<PTx>::from_bytes(bytes).unwrap();
        one_p.push(t.elapsed());
        std::hint::black_box(b);
        let t = Instant::now();
        let b = Block::<PTx>::try_from(bytes.clone()).unwrap();
        two_p.push(t.elapsed());
        std::hint::black_box(b);
    }
    [median(one_u), median(two_u), median(one_p), median(two_p)]
}

fn main() {
    let iters: usize = std::env::args()
        .nth(1)
        .map(|s| s.parse().unwrap())
        .unwrap_or(20);
    println!(
        "| Block shape | Bytes | Unverified `from_bytes` (1 verify) | Unverified `try_from` (2 verifies) | Preverified `from_bytes` | Preverified `try_from` |"
    );
    println!("|---|---|---|---|---|---|");
    let shapes: Vec<(&str, Vec<UTx>)> = vec![
        ("0 tx", vec![]),
        (
            "64 SDPActive (no signature at preverify)",
            (0..64).map(|i| tx(i, OP_SDP_ACTIVE, 0)).collect(),
        ),
        (
            "1024 Transfer (ZK sig deferred)",
            (0..1024).map(|i| tx(i, OP_TRANSFER, 0)).collect(),
        ),
        (
            "1024 ChannelInscribe, 32 B (Ed25519 at preverify)",
            (0..1024).map(|i| tx(i, OP_CHANNEL_INSCRIBE, 32)).collect(),
        ),
        (
            "1 ChannelInscribe, 1.75 MiB (max inscription)",
            vec![tx(0, OP_CHANNEL_INSCRIBE, 2 * 1024 * 1024 * 7 / 8)],
        ),
        (
            "1024 ChannelInscribe, 1850 B (~1.95 MiB body)",
            (0..1024).map(|i| tx(i, OP_CHANNEL_INSCRIBE, 1850)).collect(),
        ),
    ];
    for (name, txs) in shapes {
        let bytes = block(txs);
        let [one_u, two_u, one_p, two_p] = measure(&bytes, iters);
        println!(
            "| {name} | {} | {one_u:.2?} | {two_u:.2?} (x{:.2}) | {one_p:.2?} | {two_p:.2?} (x{:.2}) |",
            bytes.len(),
            two_u.as_secs_f64() / one_u.as_secs_f64().max(1e-9),
            two_p.as_secs_f64() / one_p.as_secs_f64().max(1e-9),
        );
    }
}
```
