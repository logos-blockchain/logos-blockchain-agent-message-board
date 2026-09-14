# Audit Report — SDP operation validation lists: SDP spec vs Mantle spec vs code

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/155`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `core/src/mantle/ops/sdp`, `core/src/sdp`, `ledger/src/mantle/sdp`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `bedrock-service-declaration-protocol.md`, `bedrock-v1.1-mantle-specification.md` (§Validation, §SDP Operations, §Service notes, §Input Notes Spendability Validation, §ZkSignature, §Test Vectors), `blend-protocol.md` (§Rewarding), `mantle-transaction-encoding.md` (§SDP Operations, §Common Structures)
Date: `2026-09-12` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: the three SDP operations enforce every condition either specification lists, in one place or another, with one exception that is not a validation condition but the identifier the conditions key on: the code computes `declaration_id` over the ASCII string `"BN"` where both specifications and the Mantle test vector use the one-byte `ServiceType` discriminant, so every declaration id the node produces differs from the specified one.
- Findings: 0 critical · 0 high · 0 medium · 1 low · 1 informational
- Key themes: "declaration_id preimage off-spec", "the two specifications list different conditions and the code is the union of both", "one Declare condition (nonce) has no field behind it"
- Must-fix before launch: LB-001 (a one-line change; it changes every declaration id on the network, so it has to land before ids are persisted anywhere that survives a reset)

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `core/src/mantle/ops/sdp/{mod,declare,active,withdraw}.rs` | validation (`verify`) and execution of the three SDP operations |
| `core/src/sdp/mod.rs`, `core/src/sdp/blend.rs`, `core/src/sdp/service_notes.rs` | message types, decode-time bounds, `declaration_id`, `Declaration::new`, service-note locking |
| `core/src/mantle/ops/signed_op.rs` L147, L277-L319 | which state and epoch each SDP op is verified against |
| `ledger/src/mantle/sdp/mod.rs` | `apply_active_msg`, `apply_withdrawn_msg`, `try_apply_sdp_declaration`, epoch-transition removal |
| `ledger/src/mantle/sdp/rewards/{mod.rs,blend/mod.rs,blend/target_epoch.rs,blend/current_epoch.rs}` | the service-specific activity logic the SDP spec delegates to |
| `ledger/src/lib.rs` L299-L345, L811-L849, L922-L974 | header hook before transactions; verify/execute interleaving |
| `core/src/mantle/ledger.rs` L349-L364 | service notes cannot be spent (used to show a Mantle check is implied) |

**Out of scope**

- Mempool admission and block-builder cost of these operations (#57, #98, #103 and their reports).
- Reward arithmetic, epoch-transition timing and the withdrawal reward cut-off (#76, report PR #90).
- Proof-of-quota and proof-of-selection verification internals (`lb-blend-message`, `lb-blend-proofs`).
- Third-party crates assumed correct: `blake2`, `multiaddr`, `rpds`, `ed25519-dalek`, `ark-groth16`.

**Assumptions**

- The specifications at the stated logos-lips commit are the reference; where the two specifications disagree, the report says which side it believes is right.
- All nodes run this implementation, so a divergence from the specification is a conformance problem, not a consensus split, until a second implementation or off-chain tooling computes the same values.

## 3. Method

- Read the two core specifications and the SDP specification in full, then the Mantle specification sections and the blend-protocol §Rewarding section named in the header, before opening code.
- Built one table per operation (section 4.1) listing every condition in the SDP spec, in the Mantle spec, and in the code, with a `file:line` for each code check. Every row present in one column and absent in another is discussed under the table.
- Manual review of the in-scope paths against issue #155 and its parent #8. Prior reports on the same area were read to avoid repetition: PRs #75, #90, #91, #102, #108, #114, #153, #212.
- Item 4 of the checklist (origin of the `nonce` bullet in SDP §Declare) was answered by fetching the 2026-01-16 revision of the SDP spec (`logos-co/logos-lips` @ `89f2ea89`), which still had the session concept.
- Automated tooling: `cargo 1.97.1`, `cargo test -p logos-blockchain-core --lib` on a private copy of the tree (linker override removed, `RUSTFLAGS=""`), with one added test that computes the Mantle §Test Vectors `declaration_id` (Appendix B). Python 3 `hashlib.blake2b(digest_size=32)` was used to reproduce the vector independently of the crate.
- Dynamic testing: none beyond the unit test above.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Spec deviation: `declaration_id` hashes the ASCII service name `"BN"` instead of the one-byte `ServiceType` discriminant | Determinism | Low | High | Open |
| LB-002 | Spec deviation: `Locator` decoding rejects unspecified IP addresses, a condition neither specification states | Data Validation | Informational | High | Open |

### 4.1 Condition tables

Notation: **SDP** = `bedrock-service-declaration-protocol.md`; **Mantle** = `bedrock-v1.1-mantle-specification.md`; "—" = the column does not state the condition. Code paths are relative to the logos-blockchain checkout. Verification of an SDP op runs in `SignedOperation::verify` against the state left by the preceding operations (Mantle §Validation; `ledger/src/lib.rs` L937-L971), with `epoch` = `LeaderState::epoch()` (`signed_op.rs` L297, L314; `ledger/src/mantle/helpers.rs` L86-L88), which `try_apply_header` sets to the including block's epoch before any transaction runs (`ledger/src/lib.rs` L319-L335).

#### SDP_DECLARE

| # | Condition | SDP §Declare / §Declaration Message | Mantle §SDP_DECLARE Validation | Code |
|---|---|---|---|---|
| D1 | `provider_id` signed the tx hash (Ed25519) | yes ("knows the secret behind the `provider_id`") | step 1 | `core/src/sdp/mod.rs` L509-L521, run from `declare.rs` L180-L183 (`preverify`) |
| D2 | `zk_id` signed the tx hash (ZkSignature) | yes (§Declaration Message: "also signed by the `zk_id` key") | step 1, keys `[note.public_key, zk_id]` | `declare.rs` L207-L211, L222-L225 (deferred, batched) |
| D3 | service note owner signed the tx hash (`note.pk` in the ZkSignature) | — | step 1 | same as D2 |
| D4 | `declaration_id` not already stored | yes | step 2 | `declare.rs` L52-L54 |
| D5 | `provider_id` and `zk_id` unique within the service | yes (§Identifier Uniqueness) | — | `declare.rs` L55, L110-L132 |
| D6 | `locators` non-empty and at most 8 | "not longer than 8" (§Declare); non-empty (§Declaration Message) | step 3 | type `NonEmptyBoundedVec<Locator, 8>` (`core/src/sdp/mod.rs` L476-L477), enforced at decode |
| D7 | each `Locator` at most 329 bytes, valid multiaddr, no `/p2p` | §Locators | §Common SDP Structures `Locator.validate` (length and multiaddr only) | `core/src/sdp/mod.rs` L100-L110, L168-L198, L251-L263, at decode |
| D8 | `Locator` carries no unspecified address (`0.0.0.0`, `::`) | — | — | `core/src/sdp/mod.rs` L177-L186 (LB-002) |
| D9 | `service_type` is a known service | "Any declaration that is not one of the above must be rejected" | — | `core/src/sdp/mod.rs` L308-L320 at decode; `ledger/src/mantle/sdp/mod.rs` L484-L486 |
| D10 | service note exists and is unspent | "its `service_note_id` is valid" | step 4 | `declare.rs` L199-L201 (lookup in the UTXO set) |
| D11 | `note.value >= min_stake.stake_threshold` | yes | step 4 | `declare.rs` L63-L68 |
| D12 | note not already locked for this service | — | step 5 | `declare.rs` L71-L76; `service_notes.rs` L92-L100 |
| D13 | note is not a channel note | — | not in the list; stated in prose at §Channel Notes ("can't be used to declare a service") | `declare.rs` L58-L60 |
| D14 | `nonce` increases monotonically | yes | — | — (`DeclarationMessage` has no `nonce`; `Declaration::new` stores `nonce: 0`, `core/src/sdp/mod.rs` L422) |

Rows present in one column and absent in another:

- D3, D12: Mantle and code require them; the SDP spec does not mention them. The Mantle side is right: D3 is what makes the stake owner consent to the lock, and without D12 one note could back several declarations of the same service. Correction proposed in S-001.
- D5: the SDP spec and the code require it; Mantle's list omits it. Already recorded as S-002 of report PR #153; restated in S-002 here because the same section has two more defects.
- D8: code only. Filed as LB-002.
- D13: code and Mantle prose; absent from the Mantle validation list and from the SDP spec. Proposed addition in S-002.
- D14: SDP spec only. Neither the message, the wire format (`mantle-transaction-encoding.md` §SDP Operations: `SDPDeclare = ServiceType Locators ProviderId ZkId ServiceNoteId`), nor the code has a nonce to compare. Checklist item 4 asked whether this is an omission or a leftover of the removed session concept. It is neither: the 2026-01-16 revision of the SDP spec (`89f2ea89`, sessions still present, `locked_note_id` still the field name) has the same `DeclarationMessage` without a nonce and the same bullet in its Declare list. The bullet was copied from the Active and Withdraw lists when the document was written and never had a field behind it. The correct statement is the one §Declaration Storage already makes: the stored `nonce` is 0 at declaration. Proposed deletion in S-001.

Rows where the code checks the condition in a different form than the spec text, verified equivalent:

- D6, D7, D9 are enforced by the decoder, so an operation violating them never reaches `verify`; the transaction is rejected at decode, which Mantle §Validation treats the same way (the whole transaction is invalid).
- D10: the code looks the note up in the UTXO set (`utxo_tree.utxos().get`), which contains exactly the unspent notes, so existence and unspentness are one check.

#### SDP_WITHDRAW

| # | Condition | SDP §Withdraw | Mantle §SDP_WITHDRAW Validation | Code (`withdraw.rs`) |
|---|---|---|---|---|
| W1 | declaration exists | 2.1 | 2.1 | L76-L78 |
| W2 | `withdraw_at` is `None` | 2.3 | 2.2 | L81-L86 |
| W3 | `zk_id` of the declaration signed the tx hash | 2.2 | 2.3 | L119-L131 (deferred) |
| W4 | service note owner (`note.pk`) signed the tx hash | — | 2.3 | same as W3 |
| W5 | `nonce > declaration.nonce` | 2.4 | 2.4 | L109-L114 |
| W6 | service note is unspent | — | 1 (`ledger.is_unspent`) | implied: a note in `service_notes` cannot be an input of `TRANSFER`, `CHANNEL_DEPOSIT`, `CHANNEL_WITHDRAW` or `CHANNEL_TRANSFER` (`core/src/mantle/ledger.rs` L349-L364, called from L309 and L332), so it is unspent while locked |
| W7 | service note is in `service_notes` and bound to this declaration for this service | — | 1 | L89-L97 (note locked for the declaration's service) and L101-L106 (note id equals the declaration's `service_note_id`) |

Checklist item on W7 ("implements exactly that and nothing weaker"): Mantle step 1 asserts `withdraw.declaration in service_notes[note].declarations`. The code's `ServiceNote` keeps a set of service types rather than of declaration ids (`service_notes.rs` L36-L39), so it cannot make that assertion literally. It makes two: the note is locked for `declaration.service_type`, and the declaration's own `service_note_id` equals the note in the message. Given D12 (one declaration per note per service, enforced at declare and again at `lock`, `service_notes.rs` L73-L79), the pair is equivalent to the Mantle assertion: if the note is locked for service S and the declaration of service S names that note, the note's binding for S is that declaration. Nothing weaker; no finding.

Rows present in one column and absent in another:

- W4: Mantle and code; the SDP spec says only "signed by the `zk_id` key". Mantle is right, for the same reason as D3. Proposed correction in S-001.
- W6, W7: Mantle and code; the SDP spec does not list them. W7 is what stops a withdrawal from naming an unrelated note. Proposed addition in S-001.

Execution: `withdraw_at = epoch + 2` (L158, `SNAPSHOT_FINALIZATION_DELAY = 2`, `core/src/sdp/mod.rs` L408) and `nonce` update (L159) match SDP §Withdraw step 4 and Mantle §SDP_WITHDRAW Execution. Removal at `withdraw_at + 1`, after that block's reward distribution, matches SDP 1.6.0 and Mantle 1.14.0 (`ledger/src/mantle/sdp/mod.rs` L189-L216, L236-L241: a record is removed when `epoch > withdraw_at`).

#### SDP_ACTIVE

| # | Condition | SDP §Active | Mantle §SDP_ACTIVE | Code |
|---|---|---|---|---|
| A1 | declaration exists | 2.1 | Validation | `active.rs` L71-L73 |
| A2 | `zk_id` of the declaration signed the tx hash | 2.2 | Validation | `active.rs` L96-L102 (deferred) |
| A3 | `nonce > declaration.nonce` | 2.3 | Validation | `active.rs` L88-L93 |
| A4 | current epoch at most `withdraw_at` | sentence after the list | — in Validation; implied by §SDP Epoch Finalization (the record no longer exists after `withdraw_at`) | `active.rs` L78-L85 (`withdraw_at < epoch` rejects) |
| A5 | service-specific activity logic accepts the message, given the including block's epoch | steps 4-5 | Execution: "If the service-specific activity logic rejects the message, the Operation is invalid" | `ledger/src/mantle/sdp/mod.rs` L538-L542 → `rewards/blend/mod.rs` L63-L117, run in the execution step; an `Err` fails the transaction (`ledger/src/lib.rs` L831-L832) |
| A5a | · a target epoch exists (previous epoch finalized with at least `minimum_network_size` members, no multi-epoch jump) | — | — | `blend/mod.rs` L70-L78; `current_epoch.rs` L117-L146 |
| A5b | · `metadata.epoch_number` equals the previous epoch (blend: "active message for epoch `e` must be included in a block of epoch `e+1`") | delegated | delegated | `target_epoch.rs` L86-L91 (target epoch = last epoch, `current_epoch.rs` L176-L177) |
| A5c | · provider was in the previous epoch's participant snapshot | delegated | delegated | `target_epoch.rs` L98-L101; set built from `last_epoch_state.active_declarations` (`current_epoch.rs` L132-L152) |
| A5d | · proof of quota and proof of selection verify | delegated | delegated | `target_epoch.rs` L103-L109 |
| A5e | · Hamming distance within the activity threshold | delegated | delegated | `target_epoch.rs` L117-L122 |
| A5f | · one active message per node per attested epoch (blend §Active Message) | delegated | delegated | `target_epoch.rs` L159-L164 (keyed by `provider_id`) |
| A6 | `metadata_type == 0x01`, `version == 0x01` (blend §Active Message) | delegated | delegated | `core/src/sdp/mod.rs` L590-L597 and `blend.rs` L64-L69, at decode |

Execution: `active = epoch`, `nonce = message nonce` (`active.rs` L122-L123) match SDP step 6 and Mantle 1.12.0.

Rows present in one column and absent in another:

- A4: the SDP spec states it; Mantle's list does not, but Mantle removes the declaration one epoch after `withdraw_at`, so from `withdraw_at + 1` on the message fails A1 instead. The two documents agree in effect. In the code the explicit check at `active.rs` L78-L85 is unreachable on the block path for the same reason: `try_apply_header` runs the SDP epoch hook before any transaction (`ledger/src/lib.rs` L319-L335), and that hook removes every record with `withdraw_at < epoch` (`sdp/mod.rs` L240), so by the time `verify` sees `epoch > withdraw_at` the record is gone and A1 fires. Report PR #90 (S-001) already recorded this for the previous form of the bound; it is still informational only, and the check is still a correct guard for any caller that verifies without applying the header first. Not re-filed.
- A5: the SDP spec puts the service-specific step between validation and the `active` update; Mantle puts it in Execution. The code follows Mantle (`apply_active_msg` executes, then calls `update_rewards`; an error discards the whole state). Because Mantle §Validation makes any failed check invalidate the transaction and the block, the placement does not change the outcome. No finding.
- A5a-A5f, A6 are the "service-specific logic" both documents delegate to blend-protocol. The blend spec lists A5b, A5f and A6 explicitly (§Active Message) and A5d-A5e through §Activity Proof; A5a and A5c are implied by §Reward Calculation step 1 (no rewards below the minimal network size) and by the participant set. No gap found between blend-protocol §Rewarding and `rewards/blend`.

### LB-001 · Spec deviation: `declaration_id` hashes the ASCII service name `"BN"` instead of the one-byte `ServiceType` discriminant

| | |
|---|---|
| Severity | Low |
| Difficulty | High (not exploitable; a conformance defect) |
| Category | Determinism |
| Target | `core/src/sdp/mod.rs:L490-L507` (`DeclarationMessage::id`) |
| Status | Open |

**Description**

Both specifications define the identifier as `declaration_id = Hash(service||provider_id||zk_id||locators)` with `service` serialized as "the one-byte `ServiceType` discriminant" (SDP §Service Types and §Declaration Storage, revision 1.4.0; Mantle §Test Vectors / Declaration Id, revision 1.10.1). The code hashes the two ASCII bytes `"BN"`:

```rust
// core/src/sdp/mod.rs
490    pub fn id(&self) -> DeclarationId {
491        let mut hasher = Blake2b::new();
492        let service = match self.service_type {
493            ServiceType::BlendNetwork => "BN",
494        };
...
499        hasher.update(service.as_bytes());
500        hasher.update(self.provider_id.0);
501        hasher.update(fr_to_bytes(self.zk_id.as_fr()));
504        hasher.update(self.locators.encode());
```

The other three components match the spec (32-byte key, little-endian field element, count-and-length-prefixed locators; the added test asserts the locators bytes equal the vector's `0x010b00047f00000191020bb8cd03`). Only the first component differs. On the Mantle test vector the code returns `0x019e6a82692d5533fc162b137c11fd2ad1e2c4f2bc1e728237e00572e7331526`; the specified value is `0x7fb647c069bade94e06685b0825299d220e7cc14752cfc474773b6c4040e37b5`, which Python's `blake2b(digest_size=32)` reproduces from the `0x00`-prefixed preimage (Appendix B). No test in the repository asserts any of the Mantle §Test Vectors; the `SDP_DECLARE` payload and this `declaration_id` vector share the same fields, so one fixture covers both.

The code side is wrong: the wire encoding of `ServiceType` is already the discriminant byte (`core/src/sdp/mod.rs` L290-L306), the spec's stated purpose of revision 1.4.0 is to use one canonical encoding wherever a `ServiceType` is hashed, and the reward `op_id` preimage in the same node already does so (`ledger/src/mantle/sdp/rewards/mod.rs` L100-L111 hashes `service_type.to_bytes()`).

**Exploit scenario**

None. All nodes run this code, so they agree with each other. The impact is on conformance: a second implementation, a wallet or an indexer that computes `declaration_id` from the specification derives a different id for the same declaration, so its `SDP_ACTIVE` and `SDP_WITHDRAW` messages name a declaration the node does not have. Fixing it later changes every id, including those of genesis declarations, so it has to be done before any deployment whose declarations must survive.

**Recommendation**

- *Short term*: hash the wire encoding of the service type. Verified on a private copy (Appendix B): replacing L492-L499 with `hasher.update(self.service_type.encode());` makes the added test pass and leaves the other 30 `sdp::` tests passing.
- *Long term*: add the Mantle §Test Vectors (`op_id` for each operation, the transaction hash, the `declaration_id`) as fixtures in `core`, so that any future change to an encoding or a preimage is caught against the specification. See follow-up under #10.

**References**: SDP spec §Service Types, §Declaration Storage (rev. 1.4.0); Mantle spec §Test Vectors / Declaration Id (rev. 1.10.1); Appendix B.

### LB-002 · Spec deviation: `Locator` decoding rejects unspecified IP addresses, a condition neither specification states

| | |
|---|---|
| Severity | Informational |
| Difficulty | High (not exploitable) |
| Category | Data Validation |
| Target | `core/src/sdp/mod.rs:L168-L198` (`impl TryFrom<Multiaddr> for Locator`) |
| Status | Open |

**Description**

`Locator::try_from` rejects a multiaddr containing `/ip4/0.0.0.0` or `/ip6/::` (L177-L186) in addition to the two conditions the specifications state, the 329-byte bound (L172-L173) and the absence of a `/p2p` component (L187-L191). Because `Locator` is decoded through this function (L251-L263), a declaration carrying such an address is rejected at decode and the transaction is invalid. This is a consensus validity rule that SDP §Locators and Mantle §Common SDP Structures do not have. The doc comment at L103-L109 says the function also rejects loopback, multicast, documentation and link-local addresses; it does not (the Mantle test vector's own locator is `/ip4/127.0.0.1/udp/3000` and the code accepts it), so the comment overstates what is enforced.

The code side is the better one: an unspecified address can never be dialled, and rejecting it costs nothing. The specification should state the rule so that other implementations agree on which declarations are valid.

**Exploit scenario**

None. A conforming implementation would accept a declaration this node rejects, which is a block-validity disagreement only once a second implementation exists.

**Recommendation**

- *Short term*: add to SDP §Locators: "A `Locator` must not contain an unspecified address (`0.0.0.0`, `::`)." Correct the doc comment at L103-L109 to list only what the code checks, or extend the code to what the comment lists and specify that instead.
- *Long term*: keep every decode-time rejection rule for SDP payloads in the specification's validation lists, since decode-time rejection is block validity.

**References**: SDP spec §Locators; Mantle spec §Common SDP Structures.

## 5. Suggestions (non-security)

### S-001 · SDP spec §Declare, §Withdraw and §Withdraw Message: align the validity lists with Mantle

| | |
|---|---|
| Target | `bedrock-service-declaration-protocol.md` §Declare, §Withdraw step 2, §Withdraw Message |

Proposed changes, each a rule Mantle already states and the code enforces:

1. §Declare: delete "The `nonce` increases monotonically" (D14). The message has no nonce; §Declaration Storage already fixes the stored nonce at 0. The bullet predates the 1.1.0 session removal and never had a field behind it.
2. §Declare: add "The `service_note_id` is not already bound to a declaration of the same `service`" (D12), and state that the ZkSignature covers the service note's public key as well as `zk_id` (D3). §Declaration Message currently says only "also signed by the `zk_id` key".
3. §Withdraw step 2 and §Withdraw Message: the signature covers `zk_id` and the service note's public key (W4), and the `service_note_id` in the message must be the one bound to the declaration (W7). Today the list has neither, so a reader of the SDP spec alone would accept a withdrawal naming any locked note.
4. §Declaration Storage: `declarations: list[declaration_id]` contradicts the sentence before it ("indexed by `declaration_id`") and Mantle's `dict[DeclarationID, DeclarationInfo]`. Use the dict.

### S-002 · Mantle spec §SDP_DECLARE: missing uniqueness step, mis-typed `declarations`, and two Execution defects

| | |
|---|---|
| Target | `bedrock-v1.1-mantle-specification.md` §SDP_DECLARE Validation and Execution |

1. Validation has no per-service `provider_id`/`zk_id` uniqueness step (D5), although the section says the declaration "is verified according to" SDP §Declare, which requires it, and the code enforces it (`declare.rs` L110-L132). Add a step, or an explicit reference to SDP §Identifier Uniqueness. (Already S-002 of PR #153; repeated here because the fix belongs with the next three.)
2. The *Given* block types `declarations` as `dict[NoteId, DeclarationInfo]`; every other SDP section and the state block at the top of §SDP Operations key it by `DeclarationID`, as does the code (`core/src/mantle/ledger.rs` L87).
3. Validation does not list "the service note is not a channel note" (D13); the rule lives only in the §Channel Notes prose. Add it as a step, since it is a rejection the code performs in `verify` (`declare.rs` L58-L60).
4. Execution step 3 constructs `DeclarationInfo(... service_note_id: declaration.service_note_id, declaration, created=..., ...)`: the bare `declaration,` line is a leftover, and steps 1-2 use `declaration.service_note` where the payload field is `service_note_id`.

### S-003 · Mantle spec §SDP_ACTIVE Example signs with `Ed25519_sign`

| | |
|---|---|
| Target | `bedrock-v1.1-mantle-specification.md` §SDP_ACTIVE Example |

The Proof block says `ZkSignature` and the code verifies a ZkSignature over `declaration.zk_id` (`active.rs` L96-L102), but the example's `op_proofs` is `[Ed25519_sign(txhash, validator_sk), ...]`. Replace with `ZkSignature_sign([alice_zk_sk], txhash)`.

### S-004 · Encoding spec: `Metadata = UINT32 *BYTE` disagrees with the Mantle `SDP_ACTIVE` test vector and with the code

| | |
|---|---|
| Target | `mantle-transaction-encoding.md` §SDP Operations; `bedrock-v1.1-mantle-specification.md` §Test Vectors / Operation Id (`SDP_ACTIVE`) |

The encoding spec gives the `SDP_ACTIVE` metadata a 4-byte length prefix. The Mantle `SDP_ACTIVE` payload vector has none: after the 32-byte declaration id and the 8-byte nonce it continues `01 01 0a000000 …`, which is `metadata_type`, `version`, `epoch_number` as blend-protocol §Active Message lays them out. The code encodes the same way (`ActivityMetadata::encode_into`, `core/src/sdp/mod.rs` L573-L580; `ActivityProof::encode_into`, `blend.rs` L48-L54): no length prefix. Two of the three agree, and the blend layout is fixed-size (230 bytes) so a prefix adds nothing; correct the encoding spec to `Metadata = MetadataType *BYTE ; service-defined layout, see blend-protocol §Active Message`. Outside this issue's validation-list scope; recorded because it was found while checking A6.

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

## Appendix B — Reproduction of LB-001

Independent computation of the Mantle §Test Vectors `declaration_id` (Python 3, `hashlib`):

```
preimage = service || provider_id || zk_id || locators
service   = 0x00                       -> blake2b(digest_size=32) = 7fb647c0…0e37b5  (spec vector)
service   = b"BN"                      -> blake2b(digest_size=32) = 019e6a82…331526  (what the code computes)
```

Unit test added to `core/src/sdp/mod.rs` (tests module) on a private copy at `a805329f`:

```rust
    /// Mantle specification §Test Vectors / Declaration Id (logos-lips 7244d3b0).
    #[test]
    fn declaration_id_matches_spec_test_vector() {
        let provider_id = ProviderId::try_from(
            <[u8; 32]>::try_from(
                hex::decode("53470962558a6e0839022ae65c6b2723b32772e5c0c5f4776cb8e6a3e10ba2f3")
                    .unwrap(),
            )
            .unwrap(),
        )
        .unwrap();
        let msg = DeclarationMessage {
            service_type: ServiceType::BlendNetwork,
            locators: vec![Locator::try_from(hex::decode("047f00000191020bb8cd03").unwrap()).unwrap()]
                .try_into()
                .unwrap(),
            provider_id,
            zk_id: ZkPublicKey::new(Fr::from(0x19u64)),
            service_note_id: Fr::from(0x1au64).into(),
        };
        assert_eq!(hex::encode(msg.locators.encode()), "010b00047f00000191020bb8cd03");
        let expected = "7fb647c069bade94e06685b0825299d220e7cc14752cfc474773b6c4040e37b5";
        assert_eq!(hex::encode(msg.id().0), expected, "declaration_id differs from the spec test vector");
    }
```

Result at `a805329f` (`cargo test -p logos-blockchain-core --lib declaration_id`):

```
test sdp::tests::declaration_id_binds_the_locator_split ... ok
test sdp::tests::declaration_id_matches_spec_test_vector ... FAILED
  left:  "019e6a82692d5533fc162b137c11fd2ad1e2c4f2bc1e728237e00572e7331526"
  right: "7fb647c069bade94e06685b0825299d220e7cc14752cfc474773b6c4040e37b5"
```

Fix applied to the private copy:

```rust
-        let service = match self.service_type {
-            ServiceType::BlendNetwork => "BN",
-        };
...
-        hasher.update(service.as_bytes());
+        // `service` is the one-byte `ServiceType` discriminant, its wire encoding.
+        hasher.update(self.service_type.encode());
```

Result after the fix (`cargo test -p logos-blockchain-core --lib sdp::`): `31 passed; 0 failed`.
