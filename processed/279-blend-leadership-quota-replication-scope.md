# Audit Report — Blend leadership quota: what `R_D` replicates, and which quota a transaction is drawn against

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/279`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `3d5d419ec85c7b23e4d7e1455bd1cde845264a9e` — component(s): `services/blend/src/core`, `services/blend/src/edge`, `services/blend/src/pending.rs`, `blend/provers`, `blend/proofs`, `ledger/src/mantle/sdp/rewards/blend`, `nodes/node/binary/src/config/blend`, `deployment/ceremony/genesis`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `blend-protocol.md` (in full); `proof-of-quota.md` › Construction, › Zero-Knowledge Proof Statement (by section)
Date: `2026-09-15` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: the accounting mismatch that issue #279 describes does not exist in the code. A transaction is never drawn against the leadership quota: it is encapsulated with Proof of Work quota, whose per-solution allowance `Q_W = ß_max` is one single-copy message, exactly as `blend-protocol.md` › Proof of Work Quota states. The leadership quota `Q_L = ß_D · (1 + R_D)` is spent only on block proposals, at `1 + R_D` copies of `ß_D` layers each, so one won slot buys exactly one replicated proposal and nothing is left over to spend on single-copy messages. This was already the case at `a805329f8`, the commit the issue cites; the premise of `#248` LB-010 was that a transaction shares the leadership quota, and it does not. What remains is a wording problem in the specification, which defines `R_D` over "data messages" while its own quota arithmetic and the code replicate proposals only, and never assigns `R_D` or `R_C` a value.
- Findings: 0 critical · 0 high · 0 medium · 0 low · 0 informational
- Key themes: quota arithmetic is consistent between provers, verifiers and the ledger; the spec's `R_D` definition is imprecise; a transaction sender gets no redundancy and, with the fallback disabled, no signal that the copy was lost.
- Must-fix before launch: none.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/blend/src/core/settings.rs`, `services/blend/src/edge/settings.rs` | how `Q_L`, `Q_W` and `Q_C` are computed from the deployment settings |
| `services/blend/src/core/mod.rs` (`handle_service_message`, `queue_transaction_for_encapsulation`, `encapsulate_next_local_message`, `schedule_local_encapsulated_message`, epoch public inputs) | copies queued per payload kind; which branch backs each; cover-message skipping; verifier inputs |
| `services/blend/src/edge/mod.rs`, `services/blend/src/edge/current_epoch.rs`, `services/blend/src/edge/handlers.rs` | same on the edge side |
| `services/blend/src/pending.rs` | the proposal copy counter and the transaction queue |
| `blend/provers/src/crypto/core_and_leader/send.rs`, `blend/provers/src/crypto/leader/send.rs` | payload type to quota branch mapping; proofs drawn per message |
| `blend/provers/src/provers/leader/mod.rs`, `blend/provers/src/provers/pow/mod.rs` | proofs minted per won slot and per PoW solution |
| `blend/proofs/src/quota/inputs/verify.rs` | the `leader_quota` and `pow_quota` public inputs a verifier uses |
| `ledger/src/mantle/sdp/rewards/blend/mod.rs` (`RewardsParameters`) | the ledger-side quota values used to verify activity proofs |
| `nodes/node/binary/src/config/blend/deployment.rs`, `deployment/ceremony/genesis/*/deployment-template.yaml`, `nodes/node/binary/src/config/deployment/settings.yaml` | the shipped `data_replication_factor` and `num_blend_layers` values and how they reach the services and the ledger |
| `services/blend/src/delivery/`, `services/blend/src/core/delivery.rs`, `services/blend/src/core/dispatcher/libp2p.rs`, `services/api/src/http/blend.rs` | what happens to a transaction the network loses |
| `core/src/blend/mod.rs` | `Q_C` and the `R_C = 0` assumption |

**Out of scope**

- The PoQ circuit itself (`zk/circuits`) and its Groth16 verifier: the report reads `proof-of-quota.md` for what the circuit constrains and takes the Rust bindings in `zk/proofs/poq` as a faithful port.
- PoW mining rate, `d_blend` derivation and the cost of a transaction in mining time (`proof-of-work.md`, issue #243).
- The failure detector's deadline arithmetic and the mempool's handling of a directly broadcast transaction, covered by the #276 and #277 reports.
- Third-party crates assumed correct: `rayon`, `tokio`, `futures`, `libp2p`.

**Assumptions**

- The specification at the commit above is the reference; where it is silent, the code's behaviour is judged against the spec's stated purpose.
- The deployment templates under `deployment/ceremony/genesis/` are the values that will ship for each network.

## 3. Method

- Manual review of the in-scope paths, working through issue `#279` under parent `#12`, with the `#248` report (LB-010, S-001) as the starting point.
- Spec conformance against `blend-protocol.md` › Quota (Core Quota, Leadership Quota, Proof of Work Quota, Quota Application), › Messages, › Generation, › Releasing, › Failure Detection and Reaction, › Notation and › Global Parameters, and `proof-of-quota.md` › Public values, › Witness, › Constraints, › Pseudocode.
- History check with `git log -S` to date the PoW-backed transaction path (`14c114a52`, 2026-08-19, "support tx blending"; `b03e60432`, 2026-08-26, "improve PoQ usage on message encapsulations") and `git show a805329f8:…` to confirm the branch mapping at the commit the issue cites.
- Automated tooling: none.
- Dynamic testing: none. The arithmetic under review is `const fn` integer code and the branch selection is a three-arm `match`; both were read rather than executed.

### 3.1 Item-by-item

**Item 1 — the quota arithmetic.** Both services compute the same three allowances from one settings struct.

```rust
// services/blend/src/core/settings.rs:54-77 (edge/settings.rs:45-66 is identical)
pub fn epoch_pow_quota(&self) -> Quota {
    self.num_blend_layers.get().try_into()…              // Q_W = ß_max
}
pub const fn epoch_leadership_quota(&self) -> Quota {
    let additional_encapsulations = num_blend_layers.checked_mul(self.data_replication_factor)…;
    let quota = num_blend_layers.checked_add(additional_encapsulations)…;   // Q_L = ß_D + ß_D·R_D
```

`num_blend_layers` plays both `ß_max` and `ß_D` (and `ß_C`, in `core/src/blend/mod.rs:12-36`). The provers mint exactly these amounts: the leader stream yields `message_quota` proofs per winning slot, indexed `0..message_quota`, with the nullifier keyed by `(slot, index)` (`blend/provers/src/provers/leader/mod.rs:99-105`); the PoW stream yields `pow_quota` proofs per mined solution, keyed by `(nonce, index)` (`blend/provers/src/provers/pow/mod.rs:117-125`).

What one message consumes is `num_blend_layers` proofs of one branch, chosen by payload type:

```rust
// blend/provers/src/crypto/core_and_leader/send.rs:279-285 (leader/send.rs:243-248 has the same two data arms)
match payload_type {
    PayloadType::Cover         => self.proofs_generator.get_next_core_proof().await,
    PayloadType::BlockProposal => self.proofs_generator.get_next_leader_proof().await,
    PayloadType::Transaction   => self.proofs_generator.get_next_pow_proof().await,
}
```

and the number of messages per payload is set where the payload is queued: `1 + data_replication_factor` copies for a proposal (`services/blend/src/core/mod.rs:1260-1264`, `services/blend/src/edge/mod.rs:411-413`, counted down by `PendingProposals::mark_copy_as_sent` in `services/blend/src/pending.rs:140-152`), one for a transaction (`core/mod.rs:1253-1258` and `:1474-1487`, `edge/mod.rs:408-410`). So:

| Payload | Branch | Messages | Proofs consumed | Proofs one grant yields |
|---|---|---|---|---|
| block proposal | leadership | `1 + R_D` | `ß · (1 + R_D)` | `Q_L = ß · (1 + R_D)` per won slot |
| transaction | PoW | `1` | `ß` | `Q_W = ß` per solution |
| cover | core | `⌊Q_C / ß⌋` per epoch | `ß` each | `Q_C` per epoch |

For the shipped templates (`num_blend_layers: 1` everywhere; `data_replication_factor: 0` standalone, `1` devnet and testnet):

| Network | `Q_L` per slot | proposal copies × layers | `Q_W` per solution | transaction copies × layers |
|---|---|---|---|---|
| standalone | 1 | 1 × 1 | 1 | 1 × 1 |
| devnet, testnet | 2 | 2 × 1 | 1 | 1 × 1 |

One won slot buys exactly one replicated proposal; one PoW solution buys exactly one transaction. Nothing is over- or under-allocated, and the "`(1 + R_D)` times as many distinct transactions" effect in `#248` LB-010 cannot occur because the leadership branch is never offered to a transaction. The same mapping is in place at `a805329f8`, the commit `#248` and `#279` cite (`git show a805329f8:blend/provers/src/crypto/core_and_leader/send.rs`, lines 279-285 are identical), so LB-010's premise was wrong when filed, not overtaken since.

**Item 2 — do edge, core and the SDP side agree?** Yes, by construction. The three quotas are computed from the same `CommonSettings` (`nodes/node/binary/src/config/blend/deployment.rs:89-95`), which `BlendDeploymentSettings::rewards_params` (`:67-86`) also copies into the ledger's `RewardsParameters`, whose `encapsulations_per_message` (`ledger/src/mantle/sdp/rewards/blend/mod.rs:265-269`) is the same `ß + ß·R_D` formula. The verifier feeds `leader_quota` and `pow_quota` straight from those values (`blend/proofs/src/quota/inputs/verify.rs:45-46`; core service epoch inputs at `services/blend/src/core/mod.rs:618-632`; edge at `services/blend/src/edge/current_epoch.rs:201-218`). Because both quotas are public inputs of every PoQ regardless of which branch it proves (`proof-of-quota.md` › Public values; the comment at `ledger/…/blend/mod.rs:281-289` says the same), any disagreement would reject every message and every activity proof outright rather than let one node under-count. The circuit bounds `index < leader_quota` per `(slot, index)` and `index < pow_quota` per `(nonce, index)`; it cannot see the payload type, so no verifier, SDP-side or otherwise, could distinguish a replicated proposal from `1 + R_D` single-copy messages. That is inherent to the design, not a gap: the spec says so itself ("the leader is not limited by this assumption", › Leadership Quota).

**Item 3 — does a transaction sender get any redundancy, and what is the recourse?** No redundancy: one message, one PoW solution, `num_blend_layers` layers (or fewer if the PoW branch runs dry mid-draw, `blend/provers/src/crypto/core_and_leader/send.rs:201-235`, which is logged at `warn`). The recourse is the direct-broadcast fallback of › Failure Detection and Reaction: both services register the payload with the failure detector when it goes out (`edge/mod.rs:439-441`; core `schedule_local_encapsulated_message` → `mark_payload_as_encapsulated` → `mark_encapsulated_payload_as_released`, `services/blend/src/core/delivery.rs:47-72`), and if the transaction's hash is not observed on the mempool's accepted stream within `T_M = ß · (Δmax + η)` rounds (`services/blend/src/settings/mod.rs:153-166`) the payload is handed to the dispatcher, which submits it to the local mempool as this node's own transaction (`services/blend/src/delivery/mod.rs:30-50`, `services/blend/src/core/dispatcher/libp2p.rs:151-190`). That costs the sender its unlinkability, as the issue anticipated. With `abstain_on_failure: true` the detector is `None` (`core/mod.rs:465`, `edge/mod.rs:364`), the fallback never fires, and the transaction, already popped from `PendingTransactions` and removed from the recovery state (`core/mod.rs:1561-1600`), is gone without trace; see S-002.

**Item 4 — which side is wrong.** The code is internally consistent and matches every normative sentence of the spec that has a number in it: `Q_L = ß_D + ß_D·R_D`, `Q_W = ß_max` with "one solution pays for exactly one message", and › Releasing's rule that a proposal removes a cover message and a transaction does not (`core/mod.rs:2013-2020` skips one cover per proposal *copy*, which is what keeps the node's emitted-message count constant when `R_D > 0`). The only sentence the code contradicts is the definition of `R_D` in › Notation and › Leadership Quota ("a redundancy parameter for data messages"), and that sentence cannot be read literally anyway once › Proof of Work Quota makes a PoW-backed transaction single-copy. The fix is a spec edit, recorded as S-001. Two narrower spec points came out of the same reading: `R_D` and `R_C` never receive a value (› Global Parameters lists `ß_max`, `ß_C`, `ß_D` but neither `R`), and › Generation allows a transaction to spend "an unused quota allowance" of any kind, which the code narrows to PoW only. The narrowing is a permitted implementation choice, not a deviation.

**Item 5 — raise in logos-lips.** Not done in this iteration: the operator running this agent instructed it not to push anything to another repository. The text for the issue is S-001 below; it is left on the source issue as the remaining work.

### 3.2 Ruled out

- **A leader spending leadership keys on transactions.** Not reachable: `next_proof_for` has no path from `PayloadType::Transaction` to `get_next_leader_proof`, and `PartialDraws` keeps the branches' half-drawn proofs apart (`core_and_leader/send.rs:37-67`) so a cancelled proposal draw cannot be resumed as a transaction.
- **Leftover leadership proofs.** If a proposal is discarded after some copies went out (`PendingProposals::discard_head`), or a slot's proofs were minted but the proposal never arrived, the surplus stays in the leader stream and backs the *next* proposal. The nullifier is per `(slot, index)`, the circuit does not bind a proof to the slot of the proposal it carries, so this is valid and consumes no more than the total `x · Q_L` for `x` won slots. At epoch rotation the surplus is dropped with the generator.
- **Cover skipping per copy vs. per proposal.** › Motivations says "for every block proposal a node generates it must generate one less cover message"; the code skips one per copy. Per copy is the reading that keeps the emitted count constant, since each copy is one released message; the other would leak `R_D` extra messages per won slot. Consistent with › Releasing, which is phrased per data message.
- **A verifier with a different `R_D`.** Would reject everything, including its own cover traffic, on the first message of the epoch; there is no silent path.
- **Overflow and width.** `epoch_leadership_quota` uses `checked_mul`/`checked_add` and `Quota::try_new` (20-bit, `zk/proofs/poq/src/lib.rs:35`), so an absurd `data_replication_factor` fails at service start, on the operator's own configuration, not from network input.
- **Edge and core disagreeing on copy count.** Both read `data_replication_factor + 1`; the only difference is `strict_add` versus `checked_add(…).expect(…)`, which behave identically.

## 4. Findings

None. The behaviour `#248` LB-010 reported as a spec deviation is a spec wording defect with no code-side counterpart; it is not re-filed here, and the resolution above should be attached to the existing `248-LB-010` finding when it is triaged.

## 5. Suggestions (non-security)

### S-001 · `blend-protocol.md`: state that `R_D` replicates block proposals only, and give `R_D` and `R_C` values

| | |
|---|---|
| Target | `blend-protocol.md` › Notation (`R_C`, `R_D`), › Global Parameters, › Leadership Quota, › Generation |

Three edits, to be raised as one logos-lips issue (item 5 of #279, still open):

1. › Notation and › Leadership Quota: define `R_D` as "the number of additional copies of a block proposal a leader releases", and say explicitly that a transaction is sent once. › Proof of Work Quota already implies this ("one solution pays for exactly one message"), and › Releasing already treats the two payloads differently, so the document has the vocabulary; only the definition lags.
2. › Global Parameters: list `R_D` and `R_C` with a value, or say they are deployment parameters and point at where a deployment records them. The node ships `R_D = 1` on devnet and testnet and `0` on standalone, and assumes `R_C = 0` (`core/src/blend/mod.rs:23-25`). A reader of the spec alone cannot compute `Q_L` or `Q_C`.
3. › Generation, trigger 3 ("holds an unused quota allowance"): either keep it general and note that an implementation may restrict transactions to the PoW branch, or say that a transaction is PoW-backed. The code does the latter; `D_Avg = L_Avg · Q_L` in › Leadership Quota then counts blending operations of proposals only, which is what the `α ≈ 1.03` correction in › Releasing assumes.

### S-002 · A transaction lost by the network is unobservable when `abstain_on_failure` is set

| | |
|---|---|
| Target | `services/blend/src/core/mod.rs:1561-1600` (`handle_local_transaction`), `services/blend/src/edge/mod.rs:439-447`, `services/api/src/http/blend.rs:56-80` (`blend_transaction`), `:85-110` (`blend_pending_transactions`) |

The HTTP route's own doc says an id back "means the transaction was accepted for blending, not that it was sent", and the pending-transactions query reports only those still waiting for a PoW solution. Once encapsulated, a transaction leaves both the queue and the recovery state, and with the fallback disabled nothing records whether the single copy ever reached a mempool. A proposal sender in the same configuration at least learns from the chain; a wallet has to poll the mempool for its own hash. Consider keeping the hash of a released transaction in the failure detector's bookkeeping even when the fallback is off, and exposing "released, unconfirmed after `T_M`" through the pending-transactions query, so an `abstain_on_failure` operator can resubmit deliberately rather than never learn. This is a robustness gap in the sender's recourse (item 3), not a security issue.

### S-003 · The `data_replication_factor` doc comments name the wrong parameter and the wrong payload

| | |
|---|---|
| Target | `services/blend/src/core/settings.rs:24`, `services/blend/src/edge/settings.rs:21`, `services/blend/src/settings/common.rs:20`, `nodes/node/binary/src/config/blend/deployment.rs:94` |

The field is documented as "`R_c`: replication factor for data messages". `R_C` is the cover-message redundancy, which the code assumes to be zero; the field is the spec's `R_D`, and it applies to block proposals only. This is the note `#248` LB-010 asked for so that "the next reader does not restore symmetry by accident"; writing "`R_D`: additional copies of each block proposal; transactions are sent once, PoW-backed" at the four declaration sites closes it.

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
