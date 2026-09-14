# Audit Report — SDP declaration lifecycle, locked stake, declarer authorisation

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/53`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `19353c61963d4ef8c37ad00d24fdf0f08a482887` — component(s): `core/src/sdp`, `core/src/mantle/ops/sdp`, `ledger/src/mantle/sdp`, `ledger/src/cryptarchia` (epoch snapshot), `services/sdp`
Date: `2026-09-07` — author: `claude-fable-5-1` — status: `final`

---

## 1. Summary

- Overall assessment: the declare → active → withdraw state machine is sound on the consensus path; stake lock, activation timing, and withdrawal timing all hold, and every op is bound to its declaration and to the tx hash. The one defect is that the SDP nonce is only required to increase, so a single hot-key-signed Active message can push the nonce to `u64::MAX` and make the declaration unwithdrawable, permanently locking the stake note.
- Findings: 0 critical · 0 high · 0 medium · 1 low · 2 informational
- Key themes: "hot key can veto the cold-key withdrawal path", "lapsed declarations have no re-entry other than withdraw + redeclare", "genesis declarations skip the ZK ownership proof"
- Must-fix before launch: none; LB-001 should be fixed before stake and operator keys are held by different parties.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `core/src/sdp/mod.rs`, `service_notes.rs` | `Declaration`, `DeclarationId`, message types, `ServiceNotes` lock/unlock |
| `core/src/mantle/ops/sdp/{declare,active,withdraw}.rs` | op verification and execution |
| `core/src/mantle/ledger.rs` L324-L388 | input spendability checks (service-note lock enforcement) |
| `core/src/mantle/transactions/signed_mantle_tx.rs` L292-L323, `mantle_tx.rs` L96-L111 | verify dispatch, tx hash construction |
| `ledger/src/mantle/sdp/mod.rs` | `SdpLedger`, `is_active`, withdrawal removal, `active_declarations` |
| `ledger/src/mantle/sdp/rewards/blend/{current_epoch,target_epoch}.rs` | only where they gate the Active op (provider set, proof epoch) |
| `ledger/src/cryptarchia/mod.rs` L95-L170, L325-L440, L640-L656; `ledger/src/config.rs` L110 | active-declaration snapshot timing |
| `ledger/src/lib.rs` L291-L330, L474-L510, L883-L919 | header-before-txs order, gas check, per-op verify/apply loop |
| `ledger/src/mantle/helpers.rs` | epoch used for op verification |
| `services/sdp/src/lib.rs` L600-L660, L730-L760; `nodes/node/binary/src/generic_services/sdp/wallet.rs` | nonce selection and tx funding on the node side |

**Out of scope**
Reward arithmetic and distribution (parent #8 questions 1 and 5), the activity-proof circuit and `lb_blend_message::reward::ActivityProof::verify_and_build`, the ZK signature circuit (`ZkSignVerifierInputs`, Groth16 verification), Ed25519 verification, the SDP mempool, blend membership construction from the snapshot (`services/blend/src/membership/service.rs`). Third-party crates assumed correct: `rpds`, `blake2`, `ark-*`/`lb_groth16`, `ed25519-dalek`, `multiaddr`.

**Assumptions**
Ledger config `execution_base_fee` is non-zero at genesis (it then stays ≥ 1 via `div_ceil`, `ledger/src/cryptarchia/mod.rs` L464-L468), so every tx must spend at least one input to pay gas. The blend proving/verification keys are the ones the ledger expects. Consensus epoch numbers never approach `u32::MAX` (all epoch additions use `strict_add`, which would panic).

## 3. Method

- Manual static review of the in-scope paths, working through sub-issue #53 with parent #8 as context. Every item below was traced to code at the stated commit; nothing was taken from documentation or memory.
- Spec conformance: `DeclarationId` derivation checked against the comment citing the SDP spec (`core/src/sdp/mod.rs` L492-L500); no other spec text was consulted.
- Automated tooling: none (no builds run; the checkout was shared with other reviewers).
- Dynamic testing: none. Existing unit tests in `ledger/src/mantle/sdp/mod.rs` (L777, L826, L930, L1009, L1221, L1298) and `ledger/src/cryptarchia/mod.rs` (L1685) were read as evidence of covered behaviour.

### Lifecycle as implemented

| State | Stored as | Entered by | Leaves via |
|---|---|---|---|
| Declared | `created = C`, `active = C+2`, `withdraw_at = None`, `nonce = 0` (`core/src/sdp/mod.rs` L413-L424) | `SDPDeclare` op in a block of epoch `C` | snapshot inclusion, `SDPWithdraw`, lapse |
| In snapshot (member) | `is_active(d, E)`: `active + inactivity_period >= E && withdraw_at.is_none_or(> E)` (`ledger/src/mantle/sdp/mod.rs` L269-L279) | snapshot for epoch `E` built at the `E-1` boundary from the ledger at end of `E-2` (`ledger/src/cryptarchia/mod.rs` L400-L440) | lapse or withdrawal |
| Refreshed | `active = submission epoch`, `nonce = msg.nonce` (`active.rs` L115-L116) | `SDPActive` op; requires provider in the previous epoch's snapshot (`target_epoch.rs` L98-L101) | — |
| Lapsed | same record, `active + inactivity_period < E` | no Active accepted within `inactivity_period` epochs | `SDPWithdraw` only (see LB-002) |
| Withdraw pending | `withdraw_at = Some(W+2)` (`withdraw.rs` L148) | `SDPWithdraw` op in epoch `W` | first header of an epoch `>= W+2` |
| Removed, note unlocked | record deleted, `ServiceNotes::unlock` (`ledger/src/mantle/sdp/mod.rs` L218-L257) | epoch transition | terminal; `provider_id`/`zk_id`/note reusable (test L1221) |

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Non-sequential nonce lets a `zk_id`-only signer make a declaration permanently unwithdrawable | Access Controls | Low | High | Open |
| LB-002 | A lapsed declaration has no re-activation path other than withdraw and redeclare | Consensus | Informational | — | Open |
| LB-003 | Genesis-mode `SDPDeclare` skips the ZK ownership proof of the service note | Authentication | Informational | — | Open |

### LB-001 · Non-sequential nonce lets a `zk_id`-only signer make a declaration permanently unwithdrawable

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Access Controls |
| Target | `core/src/mantle/ops/sdp/active.rs:L82-L88, L115-L116` (`SDPActiveOp::verify`, `execute`); `core/src/mantle/ops/sdp/withdraw.rs:L102-L108` (`SDPWithdrawOp::verify`) |
| Status | Open |

**Description**
Both Active and Withdraw only require the message nonce to be strictly greater than the stored one, and Active stores whatever value was sent:

```rust
// active.rs
82  // Check the nonce is increasing
83  if self.nonce <= declaration.nonce {
84      return Err(SdpError::InvalidNonce { .. });
...
115 declaration.active = context.epoch;
116 declaration.nonce = self.nonce;
```

```rust
// withdraw.rs
103 if self.nonce <= declaration.nonce {
104     return Err(SdpError::InvalidNonce { .. });
```

`Nonce` is `u64` (`core/src/sdp/mod.rs` L363). An Active message with `nonce = u64::MAX` is accepted, after which no `nonce > u64::MAX` exists and every later Active or Withdraw fails verification. The declaration then has no exit: `unlock_and_remove_withdrawn_declarations` only removes records with `withdraw_at` set (`ledger/src/mantle/sdp/mod.rs` L232), and the service note stays in `ServiceNotes`, so it can never be spent (`core/src/mantle/ledger.rs` L377-L379).

The authorisation asymmetry is what makes this a security issue rather than a footgun. Active needs only a ZK signature from `declaration.zk_id` (`active.rs` L91-L93); Withdraw needs a signature from both the note owner `note.pk` and `zk_id` (`withdraw.rs` L117-L121). The `zk_id` key is the node's operating key, held in the node KMS and used every epoch to post activity (`nodes/node/binary/src/generic_services/sdp/wallet.rs` L121-L165). So a hot key that was never supposed to be able to touch the stake can veto the cold-key withdrawal path forever. The node-side client always sends `declaration.nonce + 1` (`services/sdp/src/lib.rs` L619, L739), so nothing legitimate relies on gaps.

**Exploit scenario**
An attacker who obtains a provider's node `zk_id` key (node compromise, or a malicious operator running a node on behalf of a separate staker) builds a MantleTx with `SDPActive { declaration_id, nonce: u64::MAX, metadata: <any valid activity proof for the target epoch> }`, signs it with the `zk_id` key, funds it from any wallet, and submits it. The tx is accepted; `declaration.nonce` becomes `u64::MAX`. The staker's `SDPWithdraw` is now rejected with `InvalidNonce` in every future block. The staked note is locked for the life of the chain. No funds are gained by the attacker; the staker's collateral is destroyed. If the staker and operator are the same entity this is griefing after a key compromise; if they are different parties it is a loss-of-funds vector for the staker, and the rating should be raised to Medium.

**Recommendation**
- *Short term*: require `self.nonce == declaration.nonce + 1` (checked) in both `verify` functions, or accept any greater nonce but store `declaration.nonce.saturating_add(1)`-style sequential values. Either closes the `u64::MAX` dead end.
- *Long term*: do not gate Withdraw on the same counter Active uses. Withdraw is authorised by the note key; give it its own replay protection (e.g. `withdraw_at.is_none()` already prevents double-withdraw, so the nonce check on Withdraw can be dropped entirely), so a hot key can never block the cold-key path. Add a unit test that Withdraw succeeds after an Active with `nonce = u64::MAX`.

**References**: none.

### LB-002 · A lapsed declaration has no re-activation path other than withdraw and redeclare

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Consensus |
| Target | `ledger/src/mantle/sdp/rewards/blend/current_epoch.rs:L132-L152` (`finalize`), `target_epoch.rs:L98-L101` (`verify_proof`), `core/src/sdp/mod.rs:L397-L405` (doc comment on `Declaration::active`) |
| Status | Open |

**Description**
The comment on `Declaration::active` says the Idle → Active transition "must be handled by the `EpochState` snapshot logic". The snapshot logic only admits a declaration whose `active + inactivity_period >= epoch` (`is_active`, `ledger/src/mantle/sdp/mod.rs` L269-L279), and `active` is only refreshed by an Active op. An Active op is only accepted if the provider is in the previous epoch's snapshot:

```rust
// current_epoch.rs
132 let maybe_declarations = last_epoch_state
133     .active_declarations
134     .for_service(&ServiceType::BlendNetwork);
...
148 let (providers, zk_root) = Self::providers_and_zk_root(..)
// target_epoch.rs
 98 let &(zk_id, index) = self.providers.get(provider_id)
 99     .ok_or_else(|| Error::UnknownProvider(..))?;
```

`apply_active_msg` propagates this error and the whole tx fails (`ledger/src/mantle/sdp/mod.rs` L436, test L777). So once a declaration drops out of the snapshot it can never get back in: the only transition out of "lapsed" is Withdraw (2 epochs) followed by a fresh Declare (2 more epochs). With the deployed `inactivity_period: 2` (`nodes/node/binary/src/config/deployment/settings.yaml` L80 and the three ceremony templates), a provider that last refreshed at epoch `A` is a member through `A+2`, must submit the `A+2` activity proof during `A+3`, and is otherwise stuck from `A+4`. Two consecutive epochs offline therefore cost roughly four epochs of enforced downtime plus two extra txs. This is a liveness/UX property, not a safety one; nothing lets a lapsed provider be selected or rewarded.

**Exploit scenario**
Not exploitable. Impact is operational: a provider recovering from an outage longer than `inactivity_period - 1` epochs cannot resume by posting activity and must cycle its declaration, during which its stake remains locked and its `provider_id`/`zk_id` cannot be reused until the old record is removed (`declare.rs` L107-L129).

**Recommendation**
- *Short term*: document the behaviour next to `Declaration::active` and in the operator docs; the current comment describes a transition that does not exist.
- *Long term*: decide whether a re-entry path is wanted (e.g. accept an Active op with a proof for the current epoch from a non-member and treat it as re-declaration, or let a Declare op reuse an existing lapsed record). If not wanted, remove the "Idle->Active" comment.

**References**: none.

### LB-003 · Genesis-mode `SDPDeclare` skips the ZK ownership proof of the service note

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Authentication |
| Target | `core/src/mantle/ops/sdp/declare.rs:L234-L259` (`VerifiableOperation<GenesisMode> for SDPDeclareOp`) |
| Status | Open |

**Description**
In `StandardMode` a Declare is authorised by an Ed25519 signature from `provider_id` plus a ZK signature from both `note.pk` and `zk_id` over the tx hash (`declare.rs` L199-L217). In `GenesisMode` the same op runs `preverify` (Ed25519 only, L225-L232) and then `verify` returns `Ok(None)` without touching `proof.zk_sig` (L238-L259). Genesis declarations therefore never prove control of the staked note or of `zk_id`. Genesis is operator-authored configuration, so this is not attacker-reachable, but it means a genesis file can lock a note whose owner never consented, and a ceremony participant can register a `zk_id` they do not hold.

**Exploit scenario**
None from the network. A mistaken or malicious genesis author can pre-lock arbitrary genesis notes into declarations.

**Recommendation**
- *Short term*: none required for the node; note the gap in the genesis-ceremony checklist so the ceremony verifies signatures out of band.
- *Long term*: verify the ZK signature in `GenesisMode` too, or build genesis declarations from `SignedOp` values so the pairing is enforced by type (the `TODO` at `ledger/src/mantle/sdp/mod.rs` L312-L313 already points there).

**References**: none.

## 5. Suggestions (non-security)

### S-001 · Withdraw reports `NoteNotUsedForService` for a merely mismatched note

| | |
|---|---|
| Target | `core/src/mantle/ops/sdp/withdraw.rs:L82-L100` |

`verify` checks `service_notes.is_used_for_service(&self.service_note_id, ..)` (L83-L91) before checking `declaration.service_note_id != self.service_note_id` (L95-L100). A Withdraw naming an unrelated, unlocked note fails on the first check with a misleading error; swapping the two checks gives `InvalidServiceNote`, which is the actual problem. No behavioural impact.

### S-002 · Withdrawal cut-off silently forfeits the last epoch's activity reward

| | |
|---|---|
| Target | `core/src/mantle/ops/sdp/active.rs:L71-L80`; `ledger/src/mantle/sdp/mod.rs:L269-L279` |

A provider that withdraws in epoch `W` is still a member for `W+1` (`withdraw_at = W+2 > W+1`) and is expected to serve. Its `W+1` activity proof can only be submitted in `W+2`, when Active is rejected (`withdraw_at <= epoch`) and the record is already removed. The last served epoch is never rewarded. This belongs to parent #8's reward questions; flagged here because it is a lifecycle boundary. Consider allowing Active for `withdraw_at == epoch` or documenting the forfeit.

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

## Appendix B — Checklist items verified

| # | Item (from #53) | Evidence | Result |
|---|---|---|---|
| B1 | No unreachable states | Every row of the lifecycle table is reachable by an op or an epoch transition; `Declared` and `Withdraw pending` exercised by tests `ledger/src/mantle/sdp/mod.rs` L1298, L930; lapse by `ledger/src/cryptarchia/mod.rs` L1685 | holds |
| B2 | No absorbing states | Lapsed and withdraw-pending both exit via Withdraw / epoch transition. Exception: a nonce of `u64::MAX` closes both exits | **LB-001**; re-entry gap noted as **LB-002** |
| B3 | Activation timing vs. epoch boundary | Snapshot for `E+1` is built at the `E` boundary from the ledger as of end of `E-1` (`ledger/src/lib.rs` L301-L313 applies the header before txs; `ledger/src/cryptarchia/mod.rs` L417-L420); `stake_distribution_snapshot(E) = (E-1)·L` (`config.rs` L110-L112) so `update_from_ledger` never recomputes it. A declaration in any slot of epoch `C`, including the last, first appears in the `C+2` snapshot, matching `active = C+2` | holds, no off-by-one |
| B4 | Declaration in the last slot of an epoch | Same as B3: slot position inside `C` is irrelevant because the snapshot reads the state at the boundary | holds |
| B5 | Stake bound to the declaration | `Declaration.service_note_id` stored (`core/src/sdp/mod.rs` L385, L416); `ServiceNotes::lock` on execute (`declare.rs` L92-L100); Withdraw must name the same note (`withdraw.rs` L95-L100) | holds |
| B6 | Lock enforced by the ledger | `Inputs::validate_not_in_service_and_present` rejects `service_notes.contains(input)` (`core/src/mantle/ledger.rs` L373-L388), called from Transfer (`transfer.rs` L126), ChannelDeposit (`deposit.rs` L115), ChannelTransfer (`channel_transfer.rs` L119), ChannelWithdraw (`channel/withdraw.rs` L108). No other path calls `Inputs::execute` | holds |
| B7 | Same-tx lock-then-spend / spend-then-lock | Ops are verified and applied one at a time with a fresh helper per op (`ledger/src/lib.rs` L899-L916). Declare then Transfer → `InputsError::ServiceNote`; Transfer then Declare → `SdpError::InexistingNote` (`declare.rs` L191-L193) | holds |
| B8 | Unlock timing | `unlock_and_remove_withdrawn_declarations` runs only when `last_epoch_state.epoch() < epoch_state.epoch()` and removes records with `withdraw_at <= epoch` (`ledger/src/mantle/sdp/mod.rs` L181-L185, L232); test L930 | holds |
| B9 | Withdraw and still be selected | Withdraw in `W` → `withdraw_at = W+2`; `is_active(d, W+2)` is false so the `W+2` snapshot excludes it; the `W+1` snapshot was frozen at the `W` boundary and the note is still locked through `W+1`. Provider is never a member of an epoch in which its note is spendable | holds |
| B10 | Withdraw and still be rewarded | Active is rejected once `withdraw_at <= epoch` (`active.rs` L73-L80) | holds; forfeits last epoch instead, see S-002 |
| B11 | Min stake | `note.value < min_stake.threshold` checked at declare (`declare.rs` L60-L65) and again in `lock` (`service_notes.rs` L66-L71); `min_stake` is a static config value so no later re-check is needed | holds |
| B12 | Arithmetic on stake / epochs with `overflow-checks` off | Only comparisons on `note.value`; epoch math uses `Epoch::strict_add` which panics regardless of profile (`consensus/cryptarchia-engine/src/time.rs` L45-L53); inputs are consensus epochs, not attacker values; service-side nonce bump uses `checked_add` (`services/sdp/src/lib.rs` L619, L739) | holds |
| B13 | Only the declarer can Withdraw | ZK signature over tx hash with `[note.pk, declaration.zk_id]` (`withdraw.rs` L113-L125); keys come from ledger state, not from the message | holds |
| B14 | Only the declarer can Active | ZK signature with `[declaration.zk_id]` (`active.rs` L91-L97); `zk_id` from ledger state | holds |
| B15 | Declare authorisation | Ed25519 from `provider_id` over tx hash (`core/src/sdp/mod.rs` L504-L514) + ZK from `[note.pk, zk_id]` (`declare.rs` L199-L203); `note.pk` read from the UTXO set | holds (genesis exception: **LB-003**) |
| B16 | Binding to declaration ID | `declaration_id` is a signed field of Active/Withdraw; tx hash = `H("MANTLE_TXHASH_V1" ‖ encode(tx))` (`mantle_tx.rs` L96-L111) so a signature cannot be moved to another declaration | holds |
| B17 | Replay of Active / Withdraw | Nonce must increase (`active.rs` L83, `withdraw.rs` L103); Withdraw also rejected once `withdraw_at` is set (`withdraw.rs` L75-L80) | holds |
| B18 | Replay of Declare | Rejected while the record exists (`declare.rs` L49, L107-L129). After removal, a byte-identical Declare tx would re-verify, but its funding inputs are spent (node always funds via `fund_tx`, `wallet.rs` L48-L56; gas > 0 enforced at `ledger/src/lib.rs` L497). Relies on the fee input, not on the op | holds under the base-fee assumption |
| B19 | Chain binding | Tx hash carries no chain or genesis identifier; a note with the same `NoteId` on two networks (genesis-derived) would accept the same Declare on both. Tracked under #49 (chain-id replay), not rated here | see #49 |
| B20 | `DeclarationId` collisions | `Blake2b(service ‖ provider_id ‖ zk_id ‖ encode(locators))` (`core/src/sdp/mod.rs` L488-L502); locator list length-prefixed (test L706); `provider_id` and `zk_id` unique per service (`declare.rs` L107-L129) | holds |
| B21 | Panics on the consensus path | `expect` at `active.rs` L113, `withdraw.rs` L140, `declare.rs` L89 are guarded by the preceding `verify`; `unlock(...).expect` at `ledger/src/mantle/sdp/mod.rs` L238-L240 guarded by `is_used_for_service` on the line before | holds |
| B22 | Failed Active leaves state untouched | `apply_active_msg` consumes `self` and returns `Err`; caller discards (`ledger/src/mantle/mod.rs` L232-L245); test `failed_activity_does_not_return_partially_updated_ledger` L777 | holds |
| B23 | Determinism of per-declaration data | Ledger `Declarations` is `RedBlackTreeMapSync` (`ledger/src/mantle/sdp/mod.rs` L35); provider indices are assigned after sorting (`current_epoch.rs` L195-L212). Snapshot `Declarations` is a `HashMap` (`core/src/sdp/mod.rs` L428) consumed by blend membership; ordering there is out of scope (see #33, #62) | holds here |
| B24 | State growth | Records are never pruned unless withdrawn; each costs one locked note ≥ `min_stake`, ≤ 8 locators × 329 B. Bounded by supply / `min_stake`; no unbounded-growth path | holds |
