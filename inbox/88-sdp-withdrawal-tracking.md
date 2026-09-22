# Audit Report — SDP withdrawal intent tracking independently re-verified

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/88`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `c3ff08e4a9cbc8344c58a18ef7992a41403c7f9c` — component(s): `services/sdp`, `services/tx-service`, `services/wallet`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `bedrock-service-declaration-protocol.md`, `bedrock-service-reward-distribution.md`, `bedrock-anonymous-leaders-reward.md`
Date: `2026-09-21` — author: `Hansie Odendaal` — status: `final`

---

## 1. Summary

- Overall assessment: No new finding. Issue #88 independently re-verifies and expands canonical `76-LB-002` / issue `#368`: the SDP service treats mempool acceptance of a withdrawal as confirmation and discards the only local declaration state needed to retry it. The added evidence covers mempool eviction, restart recovery, and wallet reservations.
- Findings: `0` new critical · `0` new high · `0` new medium · `0` new low · `0` new informational
- Canonical outcome: `76-LB-002` / `#368` independently re-verified and expanded; its `Low / Low / Economic / Incentive` classification is preserved. This is not a separate DoS finding.
- Key themes: withdrawal intent is not tracked to ledger application or finality; persisted state has no withdrawal record.
- Must-fix before launch: none. Withdrawal-intent tracking remains the recommended remediation and follow-up for the canonical finding.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/sdp/src/lib.rs` | Withdrawal submission, declaration state, activity tracking, and restart restoration. |
| `services/sdp/src/state.rs` | Persisted SDP state. |
| `services/tx-service/src/backend/pool.rs` | Mempool transaction lifetime and eviction trigger. |
| `services/wallet/src/states.rs` | Funding-note reservation and expiry for an in-flight transaction. |
| `services/api/src/http/sdp.rs` | API-to-service withdrawal path. |

**Out of scope**

The ledger's SDP operation validation and epoch-finalization implementation were not re-audited beyond the specification and the service-facing calls needed to establish the withdrawal lifecycle. The wallet's cryptographic signing, RocksDB, libp2p, and Overwatch recovery implementation were assumed correct except for the state fields and reservation behavior directly referenced here.

**Assumptions**

The pinned SDP specification is the intended protocol: a withdrawal included in epoch `e` sets `withdraw_at = e+2`, remains service-relevant through `withdraw_at - 1`, and is removed and unlocked at `withdraw_at + 1`. “Accepted by the mempool” is not treated as inclusion in a block.

## 3. Method

- Read the parent review direction `#8` and the complete checklist issue `#88`.
- Read the required core specifications in full: `bedrock-architecture-overview.md` and `overview-cryptoeconomics.md`.
- Read the required area specifications in full: `bedrock-service-declaration-protocol.md`, `bedrock-service-reward-distribution.md`, and `bedrock-anonymous-leaders-reward.md`. The relevant SDP timing rules are the `Withdraw message`, `Message Timing`, `Withdraw`, and `SDP Epoch Finalization` sections.
- Inspected the pinned `logos-blockchain` revision with revision-qualified `git show` and `git grep`, following the withdrawal from the HTTP handler through `SdpService`, the persisted state, the mempool, and wallet reservations.
- Automated tooling: none.
- Dynamic testing: none; the finding follows directly from the state transitions and the generic mempool eviction policy.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| 76-LB-002 / #368 | Withdrawal state is discarded before the withdrawal is applied or finalized | Economic / Incentive | Low | Low | Open; independently re-verified and expanded |

### 76-LB-002 / #368 — Withdrawal state is discarded before the withdrawal is applied or finalized

No new finding is filed here. Issue #88 was spun out of canonical `76-LB-002` / issue `#368`, which already records this behavior as the same root cause. This report independently re-verifies that canonical finding and expands its evidence with mempool-eviction, restart-recovery, and wallet-reservation analysis.

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Economic / Incentive |
| Target | `services/sdp/src/lib.rs:746-799` (`handle_post_withdrawal`), `services/sdp/src/state.rs:12-27` (`SdpState`) |
| Status | Open; canonical finding `76-LB-002` / `#368`, independently re-verified and expanded |

**Description**

`handle_post_withdrawal` loads the current declaration and builds a signed withdrawal transaction. If `mempool_adapter.post_tx` fails, the function returns and keeps the declaration state. If it succeeds, however, the success only means that the local mempool accepted the transaction. The service then immediately executes:

```rust
metrics::withdrawal_success_total();

self.declaration_id = None;
self.active_message_tracker = None;
self.update_state();
```

There is no withdrawal intent, transaction hash, or status tracker. The activity tracker is the contrasting implementation: it checks the tip and LIB ledgers, resubmits while the intent is not applied, and only drops the tracker after LIB finalization (`services/sdp/src/lib.rs:316-388`, `services/sdp/src/intent.rs:66-113`). Withdrawal has none of those transitions.

The generic transaction mempool can evict a pending transaction after the configured TTL. At this revision the default is 24 hours (`services/tx-service/src/backend/pool.rs:27-52`), and eviction is selected as part of the next `remove` call (`:207-220`). A withdrawal can therefore be accepted by the mempool, clear the SDP service state, and later disappear without ever entering a block. A reorg that removes an unfinalized withdrawal has the same logical result from the SDP service's perspective: no code retains the operation to resubmit it.

The restart state contains only `declaration_id`, `updated`, and an optional pending activity (`services/sdp/src/state.rs:12-27`). It has no withdrawal intent. After the success path above, `SdpState::new(None, None)` persists no declaration. On startup, `SdpService::init` can fall back to the static `SdpSettings.declaration_id` when the recovered state is not marked updated (`services/sdp/src/lib.rs:168-187`), so a configured deployment may resurrect the identifier. That does not restore the withdrawal transaction or retry it, and the normal test configuration has `declaration_id: None`; restart is therefore not a reliable recovery path.

**Operational scenario**

An operator calls the SDP withdrawal API for an active declaration. The SDP service signs the withdrawal and the local mempool returns success. Before a leader includes it, the transaction remains pending until its 24-hour TTL expires (or is otherwise removed/reorged). The service has already stopped its activity tracker and persisted `declaration_id = None`, so later block events cannot resubmit the withdrawal and later activity requests are rejected as having no declaration. The declaration remains in the ledger, the node is still subject to the protocol's withdrawal timing, and the service note remains locked until a user manually restores the declaration ID and submits another withdrawal. The same loss occurs across a restart unless the deployment happens to retain the ID in static settings, and even then the missing withdrawal intent still requires a manual request. This is an operational recovery and incentive issue, not a separately classified third-party DoS finding.

The wallet does reserve the transaction's funding notes when the transaction is built. Those reservations are deliberately ephemeral: `PendingNotes` is rebuilt empty on restart and expires after the configured LIB-progress bound (10 blocks by default), while a note observed spent is removed from the wallet state (`services/wallet/src/states.rs:130-181`, `:298-335`, `services/wallet/src/lib.rs:271-280`). An abandoned withdrawal therefore does not itself spend the funding note on chain, and a later retry does not double-spend an already confirmed withdrawal. The reservation can simply outlive the lost service-side intent until its expiry; this is a secondary recovery detail, not a mitigation for the missing withdrawal retry.

**Recommendation**

- *Short term*: Store a withdrawal intent containing the declaration ID and the signed transaction (or enough information to rebuild it safely). Keep the declaration and intent in service state after mempool submission. On each relevant tip, check whether the withdrawal is applied; if it is not, resubmit or rebuild it using the current declaration nonce. Treat the intent as complete only after the withdrawal is present in the LIB ledger. Do not emit “withdrawal success” as a terminal state merely because `post_tx` returned `Ok`.
- *Long term*: Generalize the existing `IntentTracker` to cover SDP withdrawal and make recovery state explicit about operation type, transaction identity, and retry status. Add tests covering mempool acceptance followed by eviction, a reorg before LIB inclusion, and process restart at each point. The tests should assert that the declaration eventually receives `withdraw_at` and is removed at the specified epoch, while a confirmed withdrawal is never submitted again.

**References**: canonical `76-LB-002` / issue `#368`; `bedrock-service-declaration-protocol.md`, `Message Timing` and `Withdraw`; `bedrock-service-reward-distribution.md`, `Service Reward Distribution`; `services/sdp/src/intent.rs:66-113` (activity intent lifecycle).

Independent review completed; this report is finalized for the Draft PR handoff.
