# Audit Report — ChannelConfig threshold bounds: the missing `threshold <= len(keys)` checks, what a threshold-priced operation can really cost, and the channels that lock themselves

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/185`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `273658c765e0be1eb373883c19e7f1fb0320170a` — component(s): `core/src/mantle/ops/channel` (`config.rs`, `withdraw.rs`, `channel_transfer.rs`, `verification.rs`, `mod.rs`), `core/src/mantle/channel.rs`, `core/src/mantle/gas.rs`, `core/src/mantle/transactions/thresholds.rs`, `core/src/proofs/channel_multi_sig_proof.rs`, `core/src/block/mod.rs`, `ledger/src/lib.rs`, `ledger/src/gas_and_fees.rs`, `kms/keys/src/keys/ed25519`, `services/tx-service/src/tx/service.rs`, `services/chain/chain-leader/src`, `zone-sdk/src/sequencer/tx_builder.rs`
Specs: `https://github.com/logos-co/logos-lips` @ `d788723992a805b395f377de6e6cf59859b47168` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `bedrock-v1.1-mantle-specification.md` (all three in full); `analysis-gas-cost-determination.md` §Gas table, §Channel Withdraw, §Channel Transfer, §Channel Config, §Benchmarks (by section); `mantle-transaction-encoding.md` §OpProof (by section); `cryptarchia-v1-protocol.md` §Parameters (by section); `bedrock-v1.1-block-construction.md` §Block, §Block Proposal Validation (by section)
Date: `2026-09-24` — author: `Claude Code (research agent)` — status: `final`

---

## 1. Summary

- Overall assessment: the spec deviation reported by #113 LB-005 is still present at `273658c7`, but half of that finding does not hold. The node still accepts a `CHANNEL_CONFIG` whose `configuration_threshold` or `transfer_threshold` exceeds the number of accredited keys, which the specification (since its revision 1.7.0) forbids for the configuration threshold; such a channel can never be reconfigured, and if the transfer threshold is the one out of reach its channel notes can never be withdrawn or reassigned. The second half of #113 LB-005, that a threshold-priced operation can exceed the block execution-gas limit on its own, is wrong at the current parameters: a proof carries 66 bytes per signature and a block body is capped at 2 MiB, so no operation can carry more than 31,774 signatures, priced at 1,779,344 gas, 56 % of the limit. The gas limit is unreachable by this route, by a margin that a future 4 MiB block body would erase. What is unpriced is the other list in the payload: the keys. A 2 MiB `CHANNEL_CONFIG` that creates a channel carries about 65,530 Ed25519 public keys, each of which is decompressed and small-order-checked at decode, for zero execution gas.
- Findings: 0 critical · 0 high · 0 medium · 2 low · 0 informational
- Key themes: "the spec's invariant is not enforced, and existing state may already violate it", "byte limits bound what gas limits do not, silently", "list-shaped payloads priced as constants".
- Must-fix before launch: none. LB-001 should ship before channels hold user deposits, because the lock is permanent and the wallet-side builder does not prevent it either.

Answers to the four checklist items, in order:

1. **Missing checks (§4.1, LB-001).** Confirmed at `273658c7`. `bedrock-v1.1-mantle-specification.md` §CHANNEL_CONFIG Validation asserts `config.configuration_threshold <= len(config.keys)` (L562, added in revision 1.7.0, L34). `preverify` in `core/src/mantle/ops/channel/config.rs` L109-L121 checks only that both thresholds are non-zero and the key list is non-empty; `verify` (L130-L198) relates neither threshold to `keys.len()`, and `execute` (L207-L246) copies both into the channel state unchecked. The spec itself is silent on `transfer_threshold`, so the code deviates from the spec for the configuration threshold and both code and spec leave the transfer threshold unbounded (S-001). The check belongs in `preverify`: it needs only the payload, and a payload that fails it is invalid against every state. The zone SDK's builder (`zone-sdk/src/sequencer/tx_builder.rs` L213-L233, L282-L303) forwards both thresholds without checking them, and the node's own `ChannelConfigOp::sample()` fixture (`config.rs` L59-L73) holds two keys with thresholds 12 and 13, so nothing in the repository assumes the invariant. An existing channel state can already violate it: the ledger has accepted such configurations since the operation shipped, and a `preverify` fix repairs nothing already on chain (§4.1, Recommendation).
2. **Bounding `56 × threshold` against `limit_Ex` (§4.2).** Not needed as a gas rule at the current parameters, and the reason should be written into the specification. A `ChannelMultiSigProof` encodes as `2 + 66 × t` bytes (`core/src/proofs/channel_multi_sig_proof.rs` L179-L182; `mantle-transaction-encoding.md` §OpProof L191-L195), the mempool rejects any item above `MAX_BLOCK_TRANSACTIONS_SIZE = 2 MiB` (`services/tx-service/src/tx/service.rs` L526-L533) and block decoding rejects a body above the same bound (`core/src/block/mod.rs` L31-L34, L283-L289; `cryptarchia-v1-protocol.md` L102). Hence `t <= 31,774` for any operation that can reach the ledger, worth `56 × 31,774 = 1,779,344` gas against `EXECUTION_GAS_LIMIT = 3,193,460` (`ledger/src/gas_and_fees.rs` L5). Reaching the limit would take `t >= 57,027` signatures, 3,763,784 bytes of proof. The constraint that actually protects the limit is `56 × floor((MAX_BLOCK_SIZE − 2) / 66) <= limit_Ex`, which holds with a factor of 1.8 today and fails at a 4 MiB body (63,550 signatures, 3,558,800 gas). The `u16` range of the thresholds (`core/src/mantle/ops/channel/mod.rs` L16-L18) is not the binding bound, and bounding it to 57,026 would be vacuous. What the ledger should do instead is cheap: price the operation before verifying it. Today `try_apply_tx_operations` verifies every signature (`ledger/src/lib.rs` L950-L951) before it prices the operation (L964-L968), and the per-transaction and per-block limits are checked only once the whole transaction has been verified and executed (`try_apply_block_contents` L542, `gas_and_fees.rs` L30-L44). Gas depends only on the operation and the state it is validated against, so the two lines can be swapped and the running total compared to the limit before any signature is checked (§4.2).
3. **Worst case at each validator (§4.3).** The operation described in the issue, a 2 MiB `CHANNEL_CONFIG` with about 65,500 valid signatures, cannot exist: its proof alone would be 4.3 MB. The largest threshold-priced operation is 31,774 signatures, which at the specification's own measurement of 56,000 cycles per `verify_strict` (`analysis-gas-cost-determination.md` L264) costs 1.78 G cycles, about 0.56 s at 3.2 GHz, and is charged exactly 1,779,344 gas, that is 1.78 G cycles by the definition of execution gas: it is priced correctly, and a block cannot hold two of them. That cost is only paid by the sender when the block is valid; an invalid block that fails on its last signature costs every validator the same 0.56 s for nothing, but so does any invalid block that fails after 3.19 M gas of work, and the block-level rejection cache bounds repeats. I could not measure this in the sandbox this report was written in; the figure rests on the specification's benchmark. What the issue's arithmetic missed is the other list in the same payload: the accredited keys (§4.4, LB-002). A `CHANNEL_CONFIG` that creates a channel is verified against a threshold of 0 and priced at 0 gas (`config.rs` L95-L101, `channel.rs` L145-L157, spec L540), but its `keys` field is `NonEmptyBoundedVec<Ed25519PublicKey, 65535>` of *verified* keys (`mod.rs` L19-L21), and decoding each one decompresses the Edwards point and multiplies it by the cofactor (`kms/keys/src/keys/ed25519/mod.rs` L114-L153, `public.rs` L31-L41, L122-L132). At about 65,530 keys per 2 MiB that is on the order of 0.1 s to 0.3 s of unpriced field arithmetic per operation, paid at every gossip hop's decode and at block decode, priced only by the permanent-storage fee on the bytes.
4. **Channels with unreachable thresholds (§4.5).** A channel whose `transfer_threshold` exceeds `len(accredited_keys)` locks its channel notes permanently: `CHANNEL_WITHDRAW` and `CHANNEL_TRANSFER` require exactly `transfer_threshold` signatures with strictly increasing indices (`channel_multi_sig_proof.rs` L94-L104), each index resolving to an accredited key (`withdraw.rs` L134-L154, `channel_transfer.rs` L164-L185), which is impossible, and the only way to change the threshold is a `CHANNEL_CONFIG` signed by `configuration_threshold` keys. If that threshold is also out of reach, the channel is frozen for good; if it is reachable, the committee can repair the transfer threshold in one reconfiguration. For the sequencers this is self-inflicted and acceptable in the sense that they already have every power over the notes. For depositors it is not a new trust assumption, since the spec already tells them (L382) that the channel can move their notes anywhere without their signature; but it is a new *failure mode*, an honest committee's typo that no one can undo, which is exactly the argument the spec gives (L560-L561) for bounding the configuration threshold and should extend to the transfer threshold.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `core/src/mantle/ops/channel/config.rs` L37-L47, L91-L101, L109-L121, L130-L198, L207-L246 | payload, gas, `preverify`, `verify`, `execute` of `CHANNEL_CONFIG` |
| `core/src/mantle/ops/channel/withdraw.rs` L73-L77, L99-L157; `channel_transfer.rs` L92-L96, L123-L189; `verification.rs` L10-L50 | gas and signature checks of the two operations priced by `transfer_threshold` |
| `core/src/mantle/ops/channel/mod.rs` L16-L21; `core/src/mantle/channel.rs` L102-L157 | `ChannelKeyIndex`, `CHANNEL_MAX_KEYS`, the key list type, `ChannelState`, thresholds of an unknown channel |
| `core/src/proofs/channel_multi_sig_proof.rs` L12-L17, L57-L58, L82-L104, L134-L146, L179-L182 | proof structure, strictly increasing indices, wire size |
| `core/src/mantle/gas.rs` L157-L173; `core/src/mantle/transactions/thresholds.rs` L14-L83 | where an operation reads the threshold that prices it; the wallet-side prediction |
| `ledger/src/lib.rs` L484-L560, L928-L980; `ledger/src/gas_and_fees.rs` L5, L30-L52 | order of verification, pricing, balance check and gas-limit check |
| `core/src/block/mod.rs` L31-L34, L283-L289; `services/tx-service/src/tx/service.rs` L526-L533; `services/chain/chain-leader/src/lib.rs` L842, `tx_selection.rs` L256; `core/src/mantle/transactions/tx_list/ops.rs` L240 | the byte limits that bound a proof: block body, mempool item, proposer selection, transaction decode |
| `kms/keys/src/keys/ed25519/public.rs` L31-L41, L58-L64, L122-L132; `mod.rs` L114-L153 | what decoding a verified Ed25519 public key costs |
| `zone-sdk/src/sequencer/tx_builder.rs` L213-L233, L282-L303 | the builder that emits `CHANNEL_CONFIG` |

**Out of scope**

- `ed25519-dalek 2.2.0` and `curve25519-dalek 4.1.3` are assumed correct; `rpds`, `bincode`, the binary codec and `tokio` likewise. The Groth16 and ZK signature paths are not reviewed. The round-robin sequencing logic of `channel.rs` L218-L290 was read for context only.
- Channel deposits and inscriptions beyond what the thresholds touch; the mempool admission pipeline as such (#113); the gas values themselves (#630 covers the per-note costs).
- Whether any deployed network holds a channel that already violates the invariant: that needs a ledger dump, not source.

**Assumptions**

- Facts from issue #19, re-verified at `273658c7` where used: `[profile.release]` sets `lto = "fat"` and `strip = true` and does not enable `overflow-checks`; nothing in this report depends on integer overflow, all gas arithmetic here is `checked_*` (`gas.rs` L27-L33). The `arithmetic_side_effects`, `indexing_slicing` and `unwrap_used` lints are allowed, which is why `accredited_keys.get(index)` rather than indexing is what keeps an out-of-range signature index from panicking (`config.rs` L162-L169).
- The specification at `d788723` is taken as the reference; where it is silent (the transfer threshold) that silence is treated as a spec gap, not as code freedom.
- CPU figures are the specification's own (`analysis-gas-cost-determination.md` L252-L264, an i9-13980HX) and the definition of one execution gas as 1,000 cycles (`overview-cryptoeconomics.md` L82). Nothing in this report was measured on the audit host, which could not build or run the node during this session.

## 3. Method

- Manual review of the in-scope paths, working through issue #185 (parent #6) after reading the three documents listed in the header in full and the named sections of the other four. Every line number cited in the issue (taken at `a805329f`) was re-derived at `273658c7`; the operation files have been rewritten since, which is why the numbers differ throughout.
- Spec conformance against `bedrock-v1.1-mantle-specification.md` §CHANNEL_CONFIG (Payload, Proof, Execution Gas, Validation, Execution), §CHANNEL_WITHDRAW, §CHANNEL_TRANSFER, §Multiple Ed25519 Signatures Verification and §Gas Determination, and against the parameter tables of `cryptarchia-v1-protocol.md` and `overview-cryptoeconomics.md` §Execution Fee Market.
- Prior reports read: `processed/113-mempool-admission-validation.md` (LB-002, LB-005 and §"What admission actually checks"), `processed/155-sdp-op-validation-lists.md` and `processed/110-activity-threshold-vs-quota.md` for overlap; none covers the byte bound or the key-decode cost.
- Automated tooling: none. Dynamic testing: none. Item 3 of the issue asks for a measurement; §4.3 and §4.4 give the analytic bound instead and say so.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Spec deviation: `CHANNEL_CONFIG` accepts thresholds above the accredited key count, and such a channel can never be reconfigured or emptied | Data Validation | Low | Low | Open |
| LB-002 | `CHANNEL_CONFIG` key lists are decompressed and small-order-checked at decode for zero execution gas, up to about 65,530 keys per operation | Denial of Service | Low | Low | Open |

### 4.1 Item 1 and LB-001: the missing checks

What the specification requires, `bedrock-v1.1-mantle-specification.md` L553-L562:

```python
assert config.configuration_threshold > 0
assert config.transfer_threshold > 0
assert len(config.keys) > 0
assert len(config.keys) < 2^16
# The configuration threshold must be reachable with the accredited keys,
# otherwise the channel would be locked out of any future reconfiguration
assert config.configuration_threshold <= len(config.keys)
```

These assertions precede the `if config.channel in channels` branch, so they apply to a channel being created and to one being reconfigured alike. Revision 1.7.0 of the document (L34) is the one that "added a validation step in channel config to check the new config threshold is lower or equal than the number of accredited keys".

What the node checks, `core/src/mantle/ops/channel/config.rs` L109-L121:

```rust
fn preverify(&self, _context: &Self::Context<'_>) -> Result<(), Self::Error> {
    let operation = self.operation();

    // Check config is well-formed
    if operation.configuration_threshold == 0
        || operation.transfer_threshold == 0
        || operation.keys.is_empty()
    {
        return Err(Error::InvalidChannelConfig);
    }

    Ok(())
}
```

The first two of the spec's five assertions are here. The third and fourth are enforced by the type of `keys`, `NonEmptyBoundedVec<Ed25519PublicKey, CHANNEL_MAX_KEYS>` with `CHANNEL_MAX_KEYS = u16::MAX` (`mod.rs` L18-L21), which the decoder rejects outside `1..=65535`; the `is_empty()` test is therefore dead code (S-003). The fifth assertion is absent. `verify` (L130-L198) reads `channel.configuration_threshold` and `channel.accredited_keys` from the *current* state to check the proof (L151-L175) and never looks at `operation.configuration_threshold`, `operation.transfer_threshold` or `operation.keys.len()` together. `execute` (L217-L226 for an existing channel, L228-L242 for a new one) then stores both thresholds and the key list as given.

The wallet side does not compensate. The only builder in the repository, `build_channel_config` and `build_and_sign_channel_config` in `zone-sdk/src/sequencer/tx_builder.rs` L213-L233 and L282-L303, takes both thresholds as `u16` parameters and forwards them into `ChannelConfigOp` unchecked; the wallet crate itself only tracks `CHANNEL_CONFIG` events (`wallet/src/lib.rs` L614-L617) and builds none. The repository's own fixture, `ChannelConfigOp::sample()` at `config.rs` L59-L73, is a two-key configuration with `configuration_threshold: 12` and `transfer_threshold: 13`, and it round-trips through `preverify` in the tests at L385-L418, which documents that the invariant is not part of anyone's mental model of the type.

**Where the check belongs.** `preverify`. It needs the payload only, its outcome does not depend on the state, and a configuration that fails it is invalid in every block, which is the property `preverify` exists for (it runs at decode, `core/src/mantle/transactions/tx_list/signed_ops.rs`, so a bad configuration never enters a mempool). Putting it in `verify` would work too but would let the item sit in mempools and be gossiped first.

**Can an existing state already violate it?** Yes, in principle. The ledger has accepted these payloads for as long as the operation has existed, and a `preverify` check does not touch state that is already on chain. On a network where this has happened the channel is stuck (§4.5) whatever the fix does; on a network still to be launched the fix is enough. I could not inspect a deployed ledger from this sandbox. The genesis path cannot produce one: `Channels::from_genesis` (`channel.rs` L160-L170) only runs a genesis `InscriptionOp`, which creates a channel with one key and both thresholds at 1 (`thresholds.rs` L46-L53, spec L307-L318).

### LB-001 · Spec deviation: `CHANNEL_CONFIG` accepts thresholds above the accredited key count, and such a channel can never be reconfigured or emptied

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Data Validation |
| Target | `core/src/mantle/ops/channel/config.rs:L109-L121` (`preverify`), `L130-L198` (`verify`), `L207-L246` (`execute`); `zone-sdk/src/sequencer/tx_builder.rs:L213-L233` |
| Status | Open |

**Description**

The specification requires `configuration_threshold <= len(keys)` for every `CHANNEL_CONFIG` (L562) and gives the reason: a channel whose configuration threshold cannot be met is locked out of any future reconfiguration. The node does not enforce it. It enforces neither the analogous bound on `transfer_threshold`, which the specification does not state but which follows from the same reasoning (S-001): `CHANNEL_WITHDRAW` and `CHANNEL_TRANSFER` demand exactly `transfer_threshold` signatures with strictly increasing key indices (`channel_multi_sig_proof.rs` L94-L104), each of which must resolve to an accredited key (`withdraw.rs` L144-L154; `channel_transfer.rs` L175-L185; `verification.rs` L38-L46), so at most `len(keys)` distinct signatures can ever be presented.

The consequence of the missing check on the configuration threshold is permanent: the only operation that changes a channel's thresholds is `CHANNEL_CONFIG`, and it needs `configuration_threshold` signatures from the *current* state (`config.rs` L151-L158). A threshold above the key count can be neither met nor lowered.

**Exploit scenario**

There is no third-party victim; the committee that signs the configuration is the one that loses. Two ways it happens:

1. A sequencer committee posts a `CHANNEL_CONFIG` with `keys = [k1, k2]`, `configuration_threshold = 3` (a typo for 2, or a threshold copied from a larger key list). The operation is accepted and executed. Every later `CHANNEL_CONFIG` on that channel fails with `ThresholdUnmet` or `InvalidSignatureIndex` (L151-L169). If `transfer_threshold` was also set above 2, every `CHANNEL_WITHDRAW` and `CHANNEL_TRANSFER` fails the same way, and every note deposited into that channel (by the committee or by any user, `CHANNEL_DEPOSIT` needs no channel signature) is frozen. The notes still count toward Proof of Stake for their `ZkPublicKey` holders (spec §Bridging), so the value is not destroyed, only immobilised.
2. A hostile or careless sequencer with enough signatures for the *current* configuration threshold pushes such a configuration deliberately, for instance to make a channel's deposits unspendable before a dispute. The spec already grants sequencers the power to move deposits anywhere, so this adds no capability, but it turns a reversible custody into an irreversible freeze that no later committee can lift.

Listed as Low, in line with #113 LB-005: self-inflicted, permanent, no funds lost to a third party.

**Recommendation**
- *Short term*: add to `preverify` (`config.rs` L112-L118) the two conditions `operation.configuration_threshold as usize > operation.keys.len()` and `operation.transfer_threshold as usize > operation.keys.len()`, returning `Error::InvalidChannelConfig`, and add the two rejecting tests next to `preverify_rejects_a_zero_configuration_threshold` (L338-L351). Fix the `sample()` fixture (L70-L71) so that it satisfies the invariant, and make the zone SDK builder (`tx_builder.rs` L213-L233) reject the same inputs before signing, so that a client learns about it before paying fees. Decide, per deployed network, whether any channel already holds a threshold above its key count; if one does, a ledger-level migration (or a governance `CHANNEL_CONFIG` rule that lets `min(threshold, len(keys))` signatures reconfigure such a channel) is the only repair.
- *Long term*: make the invariant part of the type: a `ChannelConfiguration { keys, configuration_threshold, transfer_threshold }` constructor that returns `Err` when either threshold exceeds `keys.len()`, used by the payload, the state and the builder, so that a state that violates it is unrepresentable rather than merely rejected at one entry point. Raise S-001 upstream so the spec states the bound on `transfer_threshold` too.

**References**: `bedrock-v1.1-mantle-specification.md` §CHANNEL_CONFIG Validation (L553-L589), §CHANNEL_WITHDRAW Validation (L834-L853), §Multiple Ed25519 Signatures Verification (L2056-L2072), revision 1.7.0 (L34); #113 LB-005; parent #6.

### 4.2 Item 2: bounding the gas of a threshold-priced operation

**The bound the issue asked for is already implied by the byte limits.** A `ChannelMultiSigProof` is `SignatureCount *IndexedSignature` with `IndexedSignature = Ed25519Signature SignerIndex`, a `u16` count followed by 64 + 2 bytes per entry (`mantle-transaction-encoding.md` L191-L195; the node's own `calculate_channel_multi_sig_proof_byte_size` at `channel_multi_sig_proof.rs` L179-L182 is `2 + t × 66`). Every route by which an operation reaches the ledger is bounded at `MAX_BLOCK_TRANSACTIONS_SIZE = 2 × 1024 × 1024` bytes (`core/src/block/mod.rs` L34): the transaction decoder (`tx_list/ops.rs` L240), the mempool (`tx-service/src/tx/service.rs` L526-L533, matching the gossip payload bound at `network/adapters/libp2p.rs` L24-L25), the proposer's selection (`chain-leader/src/tx_selection.rs` L256, `lib.rs` L842) and the block body decoder (`block/mod.rs` L283-L289), which is rule 2 of §Block Header Validation, `bytes(transactions) <= MAX_BLOCK_SIZE`, at the spec's 2 MiB (`cryptarchia-v1-protocol.md` L102, revision 1.1.1). Therefore:

| Quantity | Value |
|---|---|
| proof bytes for `t` signatures | `2 + 66 t` |
| largest `t` in a 2 MiB body | 31,774 (2,097,150 / 66, before the rest of the transaction) |
| gas of that operation at 56 per signature | 1,779,344 (55.7 % of 3,193,460) |
| smallest `t` whose gas exceeds `limit_Ex` | 57,027 (3,193,460 / 56 = 57,026.07) |
| bytes such a proof would need | 3,763,784, 1.8 × the block body |
| gas at the `u16` maximum threshold 65,535 | 3,669,960 (the issue's arithmetic; unreachable) |
| largest `t` at a 4 MiB body | 63,550, worth 3,558,800 gas, above the limit |

So at the current parameters no single threshold-priced operation, and no combination of them in one block (the body bound applies to their sum), can exceed the execution-gas limit: the byte limit is 1.8 times tighter than the gas limit for this operation class. That is a coincidence of parameters, not a design: nothing in either specification ties `MAX_BLOCK_SIZE`, the 66-byte signature entry and `EXECUTION_CHANNEL_*_GAS` together, and the memory-safety comment in `core/src/mantle/ledger.rs` L29-L35 already contemplates a 4 MiB transaction bound, at which the gas limit becomes reachable. The constraint to state in the parameters section of the spec is `EXECUTION_CHANNEL_CONFIG_GAS × floor((MAX_BLOCK_SIZE − 2) / (ED25519_SIGNATURE_SIZE + 2)) <= limit_Ex`, and what breaks if it is violated is not the limit itself (the ledger still rejects the block, `gas_and_fees.rs` L31-L36) but the accounting order below (S-001).

**Bounding the thresholds themselves is the wrong tool.** `u16` (`mod.rs` L16) already caps them at 65,535, and a cap at 57,026 would only encode today's gas constant. The bound that has a reason is LB-001's `threshold <= len(keys)`; with it, `t` is bounded by the key list, which is itself bounded by the same 2 MiB (about 65,530 keys), so nothing changes in the gas arithmetic, but the state can no longer demand a proof larger than a block.

**Check the gas before verifying the proof.** This is the part of item 2 that is worth doing, and it is small. `try_apply_tx_operations` (`ledger/src/lib.rs` L943-L977) does, per operation: verify it against the current state (L950-L951, which for the channel operations runs every Ed25519 `verify_strict`, not deferred: `config.rs` L160, `withdraw.rs` L143), then price it (L964-L968), then execute it (L970-L976). The per-transaction limit and the per-block limit are only applied in `try_apply_block_contents` after the whole transaction has been verified and executed (L540-L542, `gas_and_fees.rs` L30-L44), and the balance check that decides whether the sender pays at all comes after that (L510). Gas is a function of the operation and the state it is verified against, exclusively (`gas.rs` L167-L173, spec L168): it can be computed first. A minimal reordering:

```rust
// ledger/src/lib.rs, try_apply_tx_operations: price against the same state the
// operation is about to be verified against, and stop before verifying anything
// that cannot fit in a block.
let op_gas = pending_op.operation().execution_gas::<Profile>(self.mantle_ledger.channels())?;
execution_gas = execution_gas.checked_add(op_gas)?;
if execution_gas > EXECUTION_GAS_LIMIT {
    return Err(LedgerError::TooMuchTransactionExecutionGas { gas: execution_gas, limit: EXECUTION_GAS_LIMIT });
}
let (remaining, (signed_op, deferred_zkp)) = verified_operations.next(&helper).transpose()?...
```

The `verified_operations.next` iterator hands out the operation only once verified, so the pricing needs a peek at the next unverified operation; `SignedOps` already exposes `op_refs()` (used for hashing), which is enough. With the byte bound in place this reordering saves nothing today; it is what keeps the ledger honest if the bound moves, and it also lets `try_apply_block_contents` stop at the first transaction that overflows the block rather than after verifying it.

### 4.3 Item 3: the worst case at each validator

The operation the issue describes cannot be built: 65,500 signatures are 4,323,002 bytes of proof, twice the block body. The worst case that can be built is one operation with 31,774 signatures on a channel whose relevant threshold is 31,774 (which needs at least 31,774 accredited keys, about 1 MiB of key list, set by an earlier configuration). Its cost, using the specification's own measurement:

| | |
|---|---|
| signatures verified, one `verify_strict` each, not batched | 31,774 |
| cycles at 56,000 per verification (`analysis-gas-cost-determination.md` L264) | 1.78 × 10^9 |
| wall time on the spec's benchmark machine, single core | about 0.4 s at the i9-13980HX's clock; about 0.56 s at 3.2 GHz |
| gas charged | 1,779,344 = 1.78 × 10^9 cycles by definition |
| share of the leader's 1 s budget (`overview-cryptoeconomics.md` L102) | 56 % |

So the operation is priced exactly as the gas model intends, and a block cannot hold two of them. The one asymmetry is who pays. The fee is only collected when the block is valid; an invalid transaction whose last signature is wrong, or whose balance is short (checked at L510 after everything else), costs every validator the same 1.78 G cycles and the attacker nothing beyond building it. That is the same exposure as any invalid block that fails after the gas budget is spent, and it is bounded by the block-level rejection cache (#3) and by the proposer having to be a leader. It is not a new finding and I do not rate it; #113 LB-002 and LB-004 cover the mempool side, where a 2 MiB item is decoded and `preverify`d before admission.

The sandbox this report was written in could not build the node, so nothing above is measured. The 56,000-cycle figure is from `tests/src/benchmarks/eddsa.rs` at the spec's cited commit; the node calls `ed25519_dalek::VerifyingKey::verify_strict` (`kms/keys/src/keys/ed25519/public.rs` L58-L64) one signature at a time (`config.rs` L161-L175), so batch verification, which would roughly halve the cost, is not in play, and should not be introduced without checking that its acceptance set matches `verify_strict` (it does not in general; `verify_strict` rejects small-order components that batch verification accepts).

### 4.4 The cost the gas table does not see: LB-002

`CHANNEL_CONFIG` carries two variable-length lists, and only one of them is priced. The proof (signatures) is priced at 56 gas per entry against the state's threshold. The payload's `keys` list is priced at nothing: a configuration that creates a channel is verified against a threshold of 0 and charged `56 × 0 = 0` execution gas (`config.rs` L95-L101 with `Channels::configuration_threshold` returning 0 for an unknown channel, `channel.rs` L145-L150; spec L540 says the same), and a reconfiguration is charged by the *old* threshold, whatever the size of the *new* key list. `analysis-gas-cost-determination.md` §Channel Config (L170-L176) prices only "the verification of the configuration_threshold Ed25519 signatures" and calls "modification of the state of the channel" negligible; L93 says that "comparison, list searching, hashes and operation in small fields are neglected", which is true for a handful of keys and false for 65,530 of them.

Decoding is not free for this type. `keys` is `VerifiedChannelKeys = NonEmptyBoundedVec<Ed25519PublicKey, 65535>` (`mod.rs` L19-L21), and `Ed25519PublicKey` is the *verified* key type: its decoder (`kms/keys/src/keys/ed25519/mod.rs` L138-L153) decodes an `UnverifiedPublicKey`, which calls `VerifyingKey::from_bytes` (`public.rs` L44-L46) and therefore decompresses the Edwards point (one field inversion-free square root, an exponentiation of about 250 squarings), then `PublicKey::try_from` (`public.rs` L122-L132) multiplies the point by the cofactor to reject small-order keys. Both happen at every decode of the transaction: at gossip in the mempool adapter (`Item::from_bytes`, #113 §"What admission actually checks", step 2), at block decode, and again wherever the bytes are re-parsed. None of it is deferred, batched or cached.

How much: I could not measure it here. Decompression is the same operation `verify_strict` performs twice (on `A` and on `R`) before its dominant double-scalar multiplication, so it is a fixed fraction of the spec's 56,000-cycle verification, on the order of a tenth of it, say 4,000 to 8,000 cycles per key including the cofactor multiplication. For the 65,530 keys that fit in 2 MiB (65,535 × 32 = 2,097,120 bytes leaves no room for the rest of the operation) that is 0.26 to 0.52 G cycles, 8 % to 16 % of the block's whole execution budget, for 0 execution gas, at every hop.

### LB-002 · `CHANNEL_CONFIG` key lists are decompressed and small-order-checked at decode for zero execution gas, up to about 65,530 keys per operation

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `core/src/mantle/ops/channel/config.rs:L42` (`keys: VerifiedChannelKeys`), `L95-L101` (gas from the state's threshold only); `kms/keys/src/keys/ed25519/mod.rs:L138-L153` and `public.rs:L44-L46, L122-L132` (decompression and cofactor check at decode); `core/src/mantle/channel.rs:L145-L150` (threshold 0 for an unknown channel) |
| Status | Open |

**Description**

The execution gas of `CHANNEL_CONFIG` is `EXECUTION_CHANNEL_CONFIG_GAS × configuration_threshold` with the threshold taken from the channel state, `0` for a channel that does not exist yet (spec L540; `config.rs` L95-L101). The operation's own `keys` list, up to 65,535 verified Ed25519 public keys, is not part of the price. Decoding a verified key decompresses a curve point and multiplies it by the cofactor, work that the gas analysis neglects as "operation in small fields" (L93) because it assumed short lists. A 2 MiB configuration for a fresh channel therefore costs every node that decodes it on the order of 0.26 to 0.52 G cycles (estimate, §4.4) and is charged 0 execution gas; the only fee it pays is the permanent-storage fee on its 2 MiB, which is the fee the storage market sets for bytes, not for CPU.

**Exploit scenario**

An attacker with enough tokens for the storage fee of a 2 MiB transaction builds, for each block, one `CHANNEL_CONFIG` creating a fresh channel (fresh `ChannelId`, `parent = ZERO`, empty proof, which `verify` accepts at L176-L195) with 65,530 distinct valid keys. Each such transaction is decoded once at every gossip hop before the mempool looks at its size (#113, step 2 precedes step 4), and once more by every validator at block decode, adding 8 % to 16 % of a block's execution budget of unpriced CPU per block for as long as the attacker keeps paying storage. The execution market never sees the load and never raises the base fee; the storage market does, which is the mitigation that exists today. The keys need not be usable: any 65,530 points on the curve will do, and an attacker can precompute them once. The impact is a degradation of every node's block validation time, not a crash, hence Low.

**Recommendation**
- *Short term*: price the key list. Either charge `CHANNEL_CONFIG` a per-key term (the analysis document would put it at the cost of one decompression plus a cofactor multiplication, on the order of 5 gas per key, so a full list would cost about 330,000 gas, a tenth of the block), or cap `CHANNEL_MAX_KEYS` at a value for which the neglect is honest (a few hundred keys makes the whole list cheaper than one signature verification, and no channel design in the specifications needs thousands of sequencers). The cap is the smaller change and also shrinks the largest possible `transfer_threshold` under LB-001's bound.
- *Long term*: in the gas model, treat every variable-length list in a payload as priced per element unless the per-element work is bounded by a constant that the analysis names, and say so in `analysis-gas-cost-determination.md` §Channel Config. Consider decoding channel keys as `UnverifiedEd25519PublicKey` (the state already stores them unverified, `channel.rs` L110-L113) and deferring the small-order check to `verify`, after the operation has been priced, so that the decode of a gossiped item stays linear in bytes.

**References**: `analysis-gas-cost-determination.md` §Channel Config (L170-L176), L93; `bedrock-v1.1-mantle-specification.md` §CHANNEL_CONFIG Execution Gas (L540), §Mantle Transaction Fee (L146-L168); `overview-cryptoeconomics.md` §Gas (L76-L85), §Execution Fee Market; #113 LB-002 (decode-time work on the mempool loop); #630 (per-note work priced as negligible, the same pattern on the ledger path).

### 4.5 Item 4: what an unreachable threshold locks

The signature-count and index rules are the same in the three threshold-checked operations. `CHANNEL_WITHDRAW` (`withdraw.rs` L134-L154), `CHANNEL_TRANSFER` (`channel_transfer.rs` L164-L185) and `CHANNEL_CONFIG` on an existing channel (`config.rs` L151-L175) all require `signatures.len() == threshold` exactly, and every `channel_key_index` must be strictly greater than the previous one (`channel_multi_sig_proof.rs` L94-L104, enforced at construction, at serde and at wire decode, L134-L146) and must resolve through `accredited_keys.get(index)`. With `n = len(accredited_keys)`, the largest proof any signer set can produce has `n` signatures, so a threshold `t > n` can never be met.

| state | `CHANNEL_INSCRIBE` | `CHANNEL_WITHDRAW` / `CHANNEL_TRANSFER` | `CHANNEL_CONFIG` | outcome |
|---|---|---|---|---|
| `configuration_threshold <= n`, `transfer_threshold > n` | works | impossible | works | recoverable: one reconfiguration lowers the transfer threshold |
| `configuration_threshold > n`, `transfer_threshold <= n` | works | works | impossible | frozen configuration: the committee, its rotation parameters and its transfer policy can never change; the channel keeps working until a key is lost |
| both `> n` | works | impossible | impossible | frozen for good: every channel note (the committee's and every depositor's) stays a channel note forever, still earning PoL for its `ZkPublicKey` holder, never spendable |

`CHANNEL_INSCRIBE` is unaffected in all three rows: it needs one signature from the round-robin sequencer (spec L424-L439), never a threshold.

Is that acceptable for a self-inflicted configuration? For the committee's own funds, yes in the same sense that sending tokens to an unspendable key is acceptable: the protocol does not owe anyone a safety net against their own signed transaction. Two things argue for closing it anyway. First, depositors: `CHANNEL_DEPOSIT` needs no channel signature (spec L714-L728), and although the spec warns that a deposit is a transfer of custody (L382), the custody it describes is one that can *move* the notes, not one that can make them immovable by mistake; a committee that means to freeze deposits can do so today by a typo it cannot reverse. Second, the specification already made this call for the configuration threshold (L560-L562) with exactly this rationale; leaving the transfer threshold out is an oversight, not a decision. LB-001's check closes both rows at the cost of one comparison in `preverify`.

## 5. Suggestions (non-security)

### S-001 · Spec: bound `transfer_threshold` by the key count, and tie `MAX_BLOCK_SIZE`, the signature entry size and the channel gas constants together

`bedrock-v1.1-mantle-specification.md` §CHANNEL_CONFIG Validation (L553-L562) asserts `configuration_threshold <= len(keys)` and says why; `transfer_threshold` gets no such assertion although `CHANNEL_WITHDRAW` and `CHANNEL_TRANSFER` are subject to the same `MultiEd25519_verify` and the same impossibility (§4.5). Add `assert config.transfer_threshold <= len(config.keys)` with the same comment. In the parameters section of `cryptarchia-v1-protocol.md` (or wherever `limit_Ex` is stated), add the constraint that keeps threshold-priced operations under the execution limit by construction, `EXECUTION_CHANNEL_CONFIG_GAS × floor((MAX_BLOCK_SIZE − 2) / 66) <= limit_Ex` (1,779,344 <= 3,193,460 today), and say what breaks if it is violated: the ledger's gas-limit check then rejects a block only after verifying up to `limit_Ex / 56` signatures (§4.2). The three channel gas constants are equal today; if they diverge the constraint applies to the largest.

### S-002 · Spec: the ordering rule does not bound the indices

The comment in `MultiEd25519_verify` (L2063-L2065) says that being strictly increasing "guarantees every index stays within bounds". It guarantees uniqueness and order; the bound comes from `keys[idx]` raising on an out-of-range index, which the pseudocode relies on implicitly. The node makes it explicit with `accredited_keys.get(index)` and a dedicated error (`config.rs` L162-L169). Say `assert idx < len(keys)` in the loop, or reword the comment; a reader implementing the routine in a language whose indexing does not raise would otherwise read out of bounds. `ChannelWithdrawOpProof` and `ChannelTransferOpProof` also declare `indexes: list[int]` (L812, L917) where `ChannelConfigOpProof` has `list[u16]` (L533); align them.

### S-003 · Dead check and an invariant-violating fixture in `config.rs`

`operation.keys.is_empty()` at `config.rs` L115 can never be true: `VerifiedChannelKeys` is `NonEmptyBoundedVec` (`mod.rs` L19-L21) and the decoder rejects an empty list, as the test `preverify_rejects_an_empty_accredited_key_set` (L370-L383) has to construct one with `new_unchecked` to exercise it. Either drop the check or keep it with a comment that it is defence against `new_unchecked`. `ChannelConfigOp::sample()` (L59-L73) should satisfy the invariant LB-001 adds, or the tests that use it as a valid configuration (L385-L418, and the `RunningThresholds` and gas tests that price it) start failing for the wrong reason once `preverify` is fixed.

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
