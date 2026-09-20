# Audit Report — Re-verification of Blend core-connection epoch binding

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/116`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `7b5e48b0fda2d5001cb8e62d6e2923f8306a7b3e` — component(s): `blend/network` core behaviour and handler, `services/blend` epoch membership and libp2p backend, `blend/message` PoQ verifier, chain epoch-state query
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `proof-of-quota.md`, `message-formatting.md`, `payload-formatting.md` in full; `blend-protocol.md` §Core Network (Bootstrapping), §Connection Details, §Connectivity Maintenance, §Transition Period
Date: `2026-09-20` — author: `Codex` — status: `draft`

## 1. Summary

- Overall assessment: the #116 mechanism and its three canonical observations are unchanged at the issue-pinned commit; no independent new finding was identified.
- Findings: `0` critical · `0` high · `1` medium (re-verified) · `1` low (re-verified) · `1` informational observation (re-verified)
- Key themes: receiver-local epoch assignment, invalid PoQ treated as peer evidence, and a slot-based epoch-state retry window.
- Must-fix before launch: the existing canonical `116-LB-001` remains the material issue; the devnet measurement requested by #116 is still outstanding.

This is a re-verification of `processed/116-epoch-bound-core-connections.md`, not a second identifier for the same defects. The surviving canonical records are #322 (`116-LB-001`), #323 (`116-LB-002`), and the informational observation #324 (`116-LB-003`). The supporting anchors were checked independently against the issue-pinned tree. #324 is now closed as a low-value observation; that tracker disposition does not change the observation or the classification.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `blend/network/src/core/with_core/behaviour/mod.rs` | epoch rotation, connection upgrade, forwarding, PoQ failure handling |
| `blend/network/src/core/with_core/behaviour/handler/mod.rs` | inbound and outbound stream protocol negotiation |
| `blend/network/src/core/poq_verification.rs` | PoQ outcome and diagnostic path |
| `services/blend/src/membership/chain.rs` | epoch-state latch and failed-query retry |
| `services/blend/src/core/backends/libp2p/swarm.rs` | epoch handoff, peer blocking, and retry behaviour |
| `services/chain/chain-service/src/lib.rs` | state source used by the epoch query |
| `ledger/src/cryptarchia/mod.rs` | slot validity and epoch-state update precondition |
| `blend/message/src/crypto/proofs.rs`, `blend/message/src/encap/mod.rs` | verifier inputs and the stale fallback comment |

**Out of scope**

The devnet measurement itself, the implementation of the proposed fixes, the blocklist lifetime covered by the originating reports, observation-window threshold calibration, edge connections, PoQ circuit soundness, and third-party correctness of `libp2p`, `multistream-select`, `tokio`, `arkworks`, and `rust-rapidsnark`.

**Assumptions**

The issue-pinned source and spec revisions are authoritative. The later `a805329f` revision referenced by the existing report is comparison evidence only, not a replacement target. Core peers are authenticated by the transport handshake; the concern is a false application-layer verdict between honest peers.

## 3. Method

- Read the two core Logos LIPs overview documents in full before claiming the issue.
- Read `proof-of-quota.md`, `message-formatting.md`, and `payload-formatting.md` in full, then read the requested `blend-protocol.md` sections before opening source code.
- Read issue #13, issue #116, the prior report, and the canonical records #322–#324. Reused the existing identifiers and classifications rather than creating duplicates.
- Re-read the pinned source anchors with `git show`/`git blame` at `7b5e48b0`. Compared the pinned tree with the later `a805329f` only to validate evidence provenance; no upstream repository was modified.
- Automated tooling: none.
- Dynamic testing: none. The requested multi-node devnet measurement of epoch-latch spread and PoQ failures remains outstanding.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Core connections carry no epoch: honest cross-boundary traffic is classified as spam | Data Validation | Medium | Low | Re-verified; canonical `116-LB-001` / #322 |
| LB-002 | A boundary-slot block can hold the Blend node in the old epoch for one slot | Denial of Service | Low | High | Re-verified; canonical `116-LB-002` / #323 |
| LB-003 | `RealProofsVerifier` retains a stale previous-epoch fallback comment | Data Validation | Informational | — | Re-verified; canonical `116-LB-003` / #324, closed observation |

### LB-001 · Core connections carry no epoch: honest cross-boundary traffic is classified as spam

| | |
|---|---|
| Severity | Medium (unchanged; canonical #322) |
| Difficulty | Low |
| Category | Data Validation |
| Target | `blend/network/src/core/with_core/behaviour/mod.rs:L285-L319`, `L924-L954`, `L960-L992`, `L1076-L1163`; `blend/network/src/core/with_core/behaviour/handler/mod.rs:L160-L167`, `L326-L330`; `services/blend/src/core/backends/libp2p/swarm.rs:L422-L424`, `L628-L644`; `services/blend/src/membership/chain.rs:L145-L221` |
| Status | Open; re-verified supporting anchor, not an independent new finding |

**Description**

At `start_new_epoch`, the behaviour moves existing negotiated peers and their message cache into `OldEpoch` and installs the new verifier (`mod.rs:L285-L319`). New inbound and outbound core upgrades still pass the same deployment `protocol_name` to the handler (`mod.rs:L1103-L1114`, `L1152-L1163`); the handler offers and requests that one protocol name (`handler/mod.rs:L160-L167`, `L326-L330`). The stream and public message header carry no epoch identifier, while PoQ inputs such as the epoch nonce, aged-ledger root, core root, and Blend difficulty are receiver-selected.

The receiver therefore records a connection under whichever local epoch it is serving when the upgrade completes. Current-epoch forwarding sends to all non-spammy current peers (`mod.rs:L924-L954`), and a failed PoQ is converted directly into `SpamReason::InvalidProofOfQuota` (`mod.rs:L984-L992`). The swarm then adds a peer in the `Spammy` state to its blocked-peer set (`swarm.rs:L422-L424`).

**Exploit scenario**

Let node `P` process the epoch transition before node `L`, and let their connection upgrade during the gap. `P` verifies `L`'s old-epoch proof against new inputs; `L` verifies `P`'s new-epoch proof against old inputs. Both fail, so both honest peers can close and block one another. The issue does not require an attacker. The spec's transition rule requires validation against both old and new inputs (`blend-protocol.md` §Transition Period), but a current-epoch connection has only its current verifier in this path.

**Recommendation**

- *Short term*: during the transition window, treat an invalid PoQ on a current connection as an untrusted epoch mismatch and close without a spam verdict, or retry it against the other epoch's verifier before blocking.
- *Long term*: include the epoch, wire-format version, and layer count in the negotiated protocol name, optionally with a digest of the public PoQ inputs. A mismatch then fails negotiation and enters the existing retry path before any message-level verdict.

**References**: `proof-of-quota.md` public inputs and nullifier rules; `message-formatting.md` public header; `blend-protocol.md` §Core Network, §Connection Details, §Connectivity Maintenance, and §Transition Period; canonical issue #322.

### LB-002 · A boundary-slot block can hold the Blend node in the old epoch for one slot

| | |
|---|---|
| Severity | Low (unchanged; canonical #323) |
| Difficulty | High |
| Category | Denial of Service |
| Target | `services/blend/src/membership/chain.rs:L145-L149`, `L217-L221`; `services/chain/chain-service/src/lib.rs:L508-L513`; `ledger/src/cryptarchia/mod.rs:L257-L269` |
| Status | Open; re-verified supporting anchor, not an independent new finding |

**Description**

On the first tick for a new epoch, the membership stream queries the chain for the epoch state at that slot (`chain.rs:L145-L149`). The chain service reads the tip state and asks the ledger to derive the requested state (`chain-service/src/lib.rs:L508-L513`). The ledger rejects a slot that is not strictly after its current slot (`cryptarchia/mod.rs:L264-L269`). If the boundary-slot block has already reached the tip before the query, the query returns `InvalidSlot`; the membership stream logs the failure and retries only on the next slot (`chain.rs:L217-L221`).

**Exploit scenario**

A boundary-slot block that arrives before a node's local tick leaves that node on the previous epoch for one full slot. Combined with LB-001, the node can form cross-epoch connections and generate false PoQ failures during that slot. The effect is bounded per transition and was not dynamically reproduced in this pass.

**Recommendation**

- *Short term*: return the already-applied tip's epoch state when the requested slot equals the tip slot, or retry immediately with the state for the current epoch rather than waiting for the next tick.
- *Long term*: make the query take an epoch rather than a slot; the Blend membership stream needs an epoch snapshot, not a future-slot derivation.

**References**: `blend-protocol.md` §Core Network (Bootstrapping); canonical issue #323.

### LB-003 · `RealProofsVerifier` retains a stale previous-epoch fallback comment

| | |
|---|---|
| Severity | Informational |
| Difficulty | — |
| Category | Data Validation |
| Target | `blend/message/src/crypto/proofs.rs:L67-L103`; `blend/message/src/encap/mod.rs:L17-L29` |
| Status | Re-verified observation; canonical #324 was closed as low value, not reclassified |

**Description**

`RealProofsVerifier` stores one `current_inputs` value and verifies the PoQ against that value only (`proofs.rs:L67-L103`). The trait constructor also takes one epoch-bound input set (`encap/mod.rs:L17-L29`). The nearby comment says verification should try previous inputs during the transition period, but no previous input is stored or attempted. This remains an inaccurate implementation comment; its operational impact is already covered by LB-001.

**Exploit scenario**

None independently. The observation can mislead a reviewer into believing that the transition-period fallback exists, but it adds no impact beyond LB-001.

**Recommendation**

Delete the stale comment or move the fallback into the behaviour, where the connection's epoch and both verifiers are available. Keep the observation informational, consistent with canonical issue #324's closure.

**References**: `blend-protocol.md` §Transition Period; canonical issue #324.

## 5. Suggestions

### S-001 · Complete the requested devnet measurement of the epoch-latch spread

No multi-node devnet run was performed in this pass. The existing diagnostics at the pinned commit expose `blend_epoch_state_latched` with `clock_epoch`, `clock_slot`, `epoch_state_epoch`, and chain-tip fields (`services/blend/src/membership/chain.rs:L174-L194`), and `blend_poq_verification_failed` with the epoch, peer, message, and sender key (`blend/network/src/core/poq_verification.rs:L87-L103`). Run at least four core nodes across several transitions, join latch timestamps by `clock_epoch`, separate slot retries from sub-slot clock/relay skew using `clock_slot` and the retry warning, and count PoQ failures in the transition window. This is the remaining empirical item from #116; static re-verification does not establish its distribution.

## Appendix A — Classification definitions

Severity and difficulty use the definitions in `docs/REPORT_TEMPLATE.md`. Existing canonical identifiers and ratings are intentionally preserved; this pass did not justify a reclassification.
