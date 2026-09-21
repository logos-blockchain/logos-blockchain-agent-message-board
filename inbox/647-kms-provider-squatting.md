# Audit Report — KMS provider-id squatting experiment and libp2p identity-signature inventory

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/647`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `9ffddb30b9e6cf79465802953caedd010ff1cecd` — component(s): `services/wallet`, `services/sdp`, `core/src/mantle/ops/sdp`, `nodes/node/binary`, `services/blend`, `libp2p`
Specs: `https://github.com/logos-co/logos-lips` @ `75d3d0382604d4a0d8e246c268935dd386ffc8ea` — read: `bedrock-service-declaration-protocol.md`, `key-types-and-generation.md`, relevant membership and connection sections of `blend-protocol.md`, `bedrock-v1.1-mantle-specification.md` § “Zero Knowledge Signature Scheme (ZkSignature)”
Date: `2026-09-21` — author: `Codex (research agent)` — status: `draft`

---

## 1. Summary

- Overall assessment: the requested local experiment independently confirms the canonical #65 `LB-001`: an unauthenticated caller can obtain a valid Ed25519 signature from the KMS-backed Blend provider key and submit a declaration that claims that provider id with attacker-chosen locators and a different ZK key.
- Findings: `0` critical · `0` high · `0` medium · `0` new low · `0` informational; existing `LB-001` from issue #65 independently re-verified and remains open/Low.
- Key themes: provider-id authorization is delegated to an unrestricted wallet signing endpoint; declaration uniqueness prevents a later same-service duplicate but does not protect the first squatting declaration; libp2p transport identity signatures are not the same as the SDP provider signature.
- Must-fix before launch: the existing #65 `LB-001` and its API-reachability prerequisite (#325) remain applicable; no new independent finding is filed here.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/wallet/src/lib.rs:919-949` | SDP declaration proof construction: ZK signature over the service-note and requested `zk_id`, plus Ed25519 signature over the transaction hash and requested `provider_id`. |
| `services/sdp/src/lib.rs:492-548`, `nodes/node/binary/src/api/handlers.rs:1125-1171` | Declaration-message HTTP submission and transaction forwarding. |
| `core/src/mantle/ops/sdp/declare.rs:109-132`, `:174-225` | Declaration uniqueness, provider-signature preverification, service-note and ZK checks. |
| `nodes/node/binary/src/api/handlers.rs:2068-2180`, wallet API body/path definitions | Direct Ed25519 and ZK signing endpoints. |
| `services/blend/src/core/backends/libp2p/settings.rs`, `services/blend/src/core/backends/libp2p/swarm.rs` | Blend libp2p identity construction and transport setup. |
| `libp2p/src/swarm.rs`, `libp2p/src/behaviour/mod.rs` | Main network identity, QUIC transport, Identify, and gossipsub configuration. |
| `blend/message/src/encap/**`, `blend/message/src/message/**` | Blend message signing fields and signed-body sizes. |

**Out of scope**

- Fixes, source changes, or shared-network experiments; all runtime changes were confined to `/tmp` scratch configuration and state directories.
- A second live node and packet capture. The absent second-node locator was deliberately used to demonstrate that declaration storage accepts location data independently of the provider key; no shared network was started.
- Correctness of third-party `libp2p`, `rustls`, `quinn`, `ed25519-dalek`, Groth16, and RocksDB implementations, except for reading the pinned libp2p 0.56.0 dependency sources to establish the signed-message layouts and configuration semantics.

**Assumptions**

- The issue’s API threat model and the canonical #65 classification are retained: the HTTP wallet API is reachable by the attacker, and the attacker does not already possess the protected KMS secrets.
- The pinned LIPs revision is authoritative for SDP provider-id/zk-id uniqueness and the ZkSignature input rules.

## 3. Method

- Manual review of the pinned target and the issue #65 report, after reading parent issue #17 and the current `docs/REPORT_TEMPLATE.md`.
- Spec conformance review of SDP signatures, declaration uniqueness, Blend connection authentication, key roles, and ZkSignature inputs.
- Static libp2p inventory against the target’s local libp2p wrapper and the locked `libp2p` 0.56.0 / `libp2p-tls` 0.6.2 / `libp2p-gossipsub` 0.49.5 sources.
- Dynamic testing used the existing release binary built from the pinned target at `/tmp/logos-blockchain-audit-9ff/target/release/logos-blockchain-node`. The node ran alone on loopback with API `127.0.0.1:28769`, main swarm UDP port `23004`, and Blend UDP port `23404`; no shared network was used.
- A scratch-only body-root utility was run with `cargo +1.98.1 run --offline` to re-encode the current chain-id format and recompute the modified scratch genesis body root. No source checkout was modified.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| `LB-001` (canonical from #65) | Unrestricted KMS signing permits provider-id squatting | Access Controls / Authentication | Low | Low | Open; independently re-verified |

### Existing `LB-001` from #65 — Unrestricted KMS signing permits provider-id squatting

This is an independent end-to-end re-verification of the surviving canonical finding, not a new finding number and not a reclassification.

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Access Controls; Authentication |
| Target | `services/wallet/src/lib.rs:919-949`, `nodes/node/binary/src/api/handlers.rs:2068-2180`, `core/src/mantle/ops/sdp/declare.rs:174-225` |
| Status | Open; canonical finding from issue #65, independently re-verified here |

**Description**

`POST /wallet/sign/ed25519` accepts a caller-selected Ed25519 public-key id and asks the KMS to sign a caller-selected transaction hash. The SDP wallet adapter uses the same operation for a declaration: it signs the declaration transaction hash using the requested `provider_id`. The request path does not establish that the caller is authorized to use that provider key. The declaration verifier checks that the signature is valid for the declared provider id, not that the declaration was initiated by the owner of that provider id.

The current declaration validator does enforce one `provider_id` and one `zk_id` per service once a declaration exists. That is a useful collision check, but it does not stop the first declaration from claiming an unused provider id. The declaration-message API also accepts locators without a peer id; the SDP specification completes the locator with the declaration’s provider id.

**Exploit scenario and dynamic evidence**

1. The scratch genesis declaration was changed only to use the main network key `dc4aec...` instead of `aa70...`, leaving the Blend provider id `aa70aafc48536ae13168ed4845981a40cbd2dc1c38df88d04c46250e9ad65ce0` available for the experiment. The scratch genesis body root was recomputed using the current target’s chain-id encoding.
2. The first request used provider id `aa70...`, ZK id `e3635f207984ae779cf76b5f20714b514373f61ff96260879fe0a6d71f2dce07`, service note `7302de...`, and locator `/ip4/127.0.0.1/udp/23405/quic-v1`, where no node B was running. The API returned declaration id `730edd37074529674af4506cf24227f8f75dc1c86252ac33dfcf41c3c6c49cd7`.
3. The declaration was included in block `dc15fa52418c71e41a53b91a2692ac9af38d5b2c196082d3d282b4d94adf6073` at slot `28600`, in transaction `8b4f1d910cd31e4cb00a1e1e73e08314d17f2f7981987c3eea2a0958f2a312ee`. The block payload contains the attacker-selected provider id, ZK id, absent-node locator, and a valid `ZkAndEd25519Sigs` proof. `GET /mantle/sdp/declarations` returned the same stored declaration.
4. A second request used the same provider id with ZK id `cce9796339efd968df9cd463bc2248e4b29c86409496e8b2599da8c4c1074d22`, service note `ac541e...`, and locator `/ip4/127.0.0.1/udp/23406/quic-v1`. It returned declaration id `c3ef4e05228740f79a98250bee3af3223fa67216173606d856833d8465caf062` and transaction hash `ed5d39db4dc87b408985fd493f969ff4d058ba64259105c94e7ee28950df640c`.
5. At shutdown, the isolated node’s mempool still listed the second transaction and the SDP ledger still contained only the first `aa70...` declaration. The node stopped advancing after the first inclusion, so this run did not obtain a second block-level rejection record. Static validation at `core/src/mantle/ops/sdp/declare.rs:109-132` identifies the deterministic result if included: `SdpError::DuplicateProviderId` for the same service. This limitation does not affect the first on-chain acceptance that re-verifies the finding.

The same node returned `200` for `POST /wallet/sign/ed25519` with provider id `aa70...` over the 32-byte ASCII message `THIS-IS-NOT-A-MANTLE-TX-HASH-32B`; the returned signature matched the prior #65 evidence. It also returned `200` for `POST /wallet/sign/zk` with Blend ZK id `852efb444db8c3c811625850df39425f43aeffc69571192c0be9f72523256e0a` over that arbitrary hash.

The Blend runtime query returned one local node and `core_info: null`; no peer B was available to complete a dial. The stored locator therefore did not confer network authentication on the absent endpoint. If B serves that address with B’s own libp2p identity, QUIC/TLS authenticates B’s peer id, not A’s provider id, and the connection does not become an authenticated A connection. The chain nevertheless advertises the misleading A-completed locator to consumers of the declaration.

**ZK operation inventory**

The wallet implementation uses ZK signatures as follows:

- `Declare`: `[service_note.pk, declare_op.zk_id]`, plus Ed25519 over the transaction hash under `provider_id`.
- `Active`: the declaration’s `zk_id`.
- `Withdraw`: `[service_note.pk, declaration.zk_id]`.

The generic ZK HTTP endpoint accepts caller-selected `tx_hash` and public-key list. Holding the Blend quota key therefore permits arbitrary proofs under that key, but protocol validation still requires the corresponding declaration/service-note/nonce state. The provider-id squatting path is the Ed25519 half: it lets the caller satisfy the declaration’s provider signature with the Blend identity key.

**Libp2p identity-signature inventory**

The main swarm constructs a libp2p identity from `network.backend.swarm.node_key` and passes its public key to Identify. Its gossipsub behaviour is configured as `MessageAuthenticity::Author(peer_id)` with validation mode `None`; in locked libp2p 0.49.5, `Author` explicitly disables message signing. No gossipsub signature is therefore produced by the main node identity in this target.

The pinned libp2p QUIC/TLS transport does use the identity key. `libp2p-tls` creates a self-signed certificate with an ephemeral certificate key and places an Ed25519 signature by the host identity key in the libp2p certificate extension. The signed bytes are the fixed prefix `libp2p-tls-handshake:` (21 bytes) followed by the DER public key of the ephemeral certificate, not a caller-controlled 32-byte hash. The TLS handshake also signs its normal variable-length transcript with the certificate key; that is transport authentication, not a direct KMS signing endpoint.

Identify is configured with a public key using `identify::Config::new`, not `new_with_signed_peer_record`, so this target does not send a libp2p signed peer record. Identify data is carried over the authenticated transport and is not an additional provider-id signature.

The Blend swarm constructs its libp2p identity directly from `non_ephemeral_signing_key` (the on-chain provider id), so the Blend NSK signs the same QUIC/TLS identity-certificate material. Blend encapsulated messages are different: their public and blending headers carry 64-byte Ed25519 signatures made by ephemeral signing keys, and the public-header signature covers the fixed private-header plus padded-payload body. With one layer, the body is already larger than 32 bytes; the signing key is not the persistent Blend NSK. No inspected libp2p or Blend identity path signs exactly 32 caller-controlled bytes.

**Recommendation**

- *Short term*: apply the existing #65 recommendation: authenticate the wallet API and enforce a KMS key-usage policy so provider/network identity keys cannot be selected by generic wallet signing requests. Do not load the libp2p or Blend identity secrets into an API-reachable generic signing authority.
- *Long term*: represent key roles in the KMS and make each protocol operation request a typed role rather than an arbitrary public-key id. Add an integration test that starts with an unused provider id, submits a declaration with foreign locators, confirms inclusion, and checks that a second same-service provider-id declaration is rejected with `DuplicateProviderId`.

**References**: issue #65 report `processed/65-kms-key-roles-import-derivation.md` LB-001; `bedrock-service-declaration-protocol.md` provider-id signature and identifier-uniqueness sections; `key-types-and-generation.md` NSK/provider-id role; `blend-protocol.md` connection authentication and locator completion; `bedrock-v1.1-mantle-specification.md` ZkSignature section.

## 5. Suggestions (non-security)

- **S-001 — Return the transaction hash from `/sdp/declaration`.** The endpoint currently returns the declaration id, while the transaction hash is only visible in logs or later block data. Returning both would make transaction inclusion and rejection diagnostics reproducible for operators and audit tooling.
- **S-002 — Make provider-id/locator reachability observable.** A declaration can store a locator that is not reachable by the declared provider identity. A diagnostic or asynchronous status should distinguish “stored declaration” from “authenticated provider reachable at locator.”

## Appendix A — Validation evidence

| Check | Result |
|---|---|
| Pinned source/spec resolution | Source `9ffddb30b9e6cf79465802953caedd010ff1cecd`; specs `75d3d0382604d4a0d8e246c268935dd386ffc8ea`. |
| Node startup | Release binary from the pinned source; scratch state `/tmp/kms647-node-5`; loopback-only ports; reached `Online`. |
| First declaration | API accepted; included at block slot 28600; tx `8b4f1d...`; ledger readback confirmed provider id, ZK id, note, and foreign locator. |
| Duplicate declaration | API/mempool accepted tx `ed5d39...`; ledger remained unchanged; block-level rejection was not observed because the isolated tip stopped advancing after the first inclusion. Static validator result is `DuplicateProviderId`. |
| Direct Ed25519 signing | `POST /wallet/sign/ed25519` with Blend provider id returned `200`. |
| Direct ZK signing | `POST /wallet/sign/zk` with Blend ZK id returned `200`. |
| Blend membership | `/blend/info` returned one local node and no core peer information; no shared network was used. |
| Source modifications | None. Scratch deployment/config/state only. |

Source issue disposition: leave issue #647 assigned and open while this report awaits independent review; no tracker or post-merge action is part of this Research Workflow iteration.

Draft pending independent review and explicit approval.
