# Audit Report — Blend message version transition: epoch-keyed accepted version, edge emission rule, and the unsigned version byte

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/183`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `blend/message, blend/network, blend/scheduling, blend/membership, services/blend, core/src/sdp, nodes/node/binary/src/config/blend, nodes/api-common`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md, overview-cryptoeconomics.md, message-formatting.md, payload-formatting.md` (in full); `blend-protocol.md, message-encapsulation.md, bedrock-service-declaration-protocol.md` (by section)
Date: `2026-09-12` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: the Blend network already keeps exactly one verification context per epoch and selects it by the connection a message arrives on, so the accepted message version does not need a set: it is a function of the epoch, `version(e)`, switched at an activation epoch `E_V`, and the existing Transition Period is the only overlap window needed; the edge node emits `version(e)` for the epoch it is on and needs no negotiation, no `/blend/info` field and no SDP field; the unsigned version byte is not exploitable under this rule but should be signed in the next format revision. The design supersedes the three-phase accept-set rollout proposed for surface 2 in the #70 report and is compatible with the epoch-bound protocol name of #116/#235, which should carry `version(e)` rather than a constant.
- Findings: 0 critical · 0 high · 0 medium · 1 low · 2 informational
- Key themes: "version is a compile-time constant at every construction site", "no transition period on the edge path", "version byte outside every signature"
- Must-fix before launch: none. Before the first Blend format change: the plumbing in LB-002 and the decode rule in §2, together with #235; the signed version byte (LB-003) when the format is next revised.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `blend/message/src/message/public_header.rs`, `blend/message/src/encap/encapsulated.rs`, `blend/message/src/codec.rs`, `blend/message/src/fixtures/` | version constant, decode contexts, signing body, every site that constructs a public header, golden fixtures |
| `blend/network/src/core/with_core/behaviour/{mod.rs,old_epoch.rs,utils.rs}`, `blend/network/src/core/with_edge/behaviour/mod.rs`, `blend/network/src/lib.rs` | epoch transition machinery, connection replacement, spam verdicts, edge receive path, framing |
| `blend/scheduling/src/epoch.rs`, `services/blend/src/core/epoch_stages/*.rs`, `services/blend/src/membership/chain.rs`, `services/blend/src/core/backends/libp2p/swarm.rs` | what drives a rotation, the transition period timer, service-level old/new epoch pairing, `/blend/info` data |
| `services/blend/src/edge/{mod.rs,current_epoch.rs,handlers.rs}`, `services/blend/src/edge/backends/libp2p/{swarm.rs,settings.rs}`, `services/blend/src/delivery/mod.rs` | how an edge node learns its epoch, emits, and what it can observe about delivery |
| `core/src/sdp/mod.rs`, `blend/membership/src/lib.rs`, `services/blend/src/membership/service.rs` | what a declaration and a membership entry carry |
| `nodes/node/binary/src/config/blend/{deployment.rs,mod.rs}`, `nodes/node/binary/src/config/deployment/settings.yaml`, `deployment/ceremony/genesis/testnet/deployment-template.yaml` | where a protocol parameter such as an activation epoch would live; the transition period value |
| `nodes/api-common/src/paths.rs`, `nodes/node/binary/src/api/handlers.rs`, `services/blend/src/message.rs` | `/blend/info` |

**Transition design** (the deliverable of the issue)

*Notation.* `e` is the Blend epoch (the consensus epoch, `blend-protocol.md` §Time). `V` is the message version defined in `message-formatting.md` §Public Header, today `1`. `T` is the Transition Period in rounds (`blend-protocol.md` §Transition Period, 30) and `T_M` the message traversal time (15). `E_V` is the activation epoch of version `V`, a deployment parameter with the same status as `blend.common.protocol_name` (`settings.yaml:5`).

*Rule.* `version(e) = V` if `e ≥ E_V`, else `V − 1`. A node generates every message of epoch `e` with `version(e)` and accepts on a connection of epoch `e` only `version(e)`. Nothing else changes.

*Why one version per epoch rather than a set.* Item 1 of the issue asked whether the epoch machinery can hold an accepted-version set `{N, N+1}`. It can, but it does not need to, because it already holds two complete contexts during `T` and selects one of them by connection, not by message content:

1. Rotation is driven by the slot clock, not by chain observation: `membership/chain.rs:148` reads `SlotTick { epoch, slot }` and rebuilds the epoch when the epoch number changes, so all honest nodes rotate within clock skew of each other (the residual skew is #116's subject).
2. `EpochEventStream` yields `NewEpoch` and then `TransitionPeriodExpired` after `epoch_transition_period` (`blend/scheduling/src/epoch.rs:98-121`), which the node derives as `slot_duration × average_slots_per_block` (`config/blend/deployment.rs:59-65`, `config/blend/mod.rs:78-80`, `cryptarchia/deployment.rs:56-62`, `cryptarchia-engine/src/config.rs:125-131`): 30 s with the testnet template's `slot_activation_coeff = 1/30` (`deployment-template.yaml:26-28`), 20 s with the repository default `1/20` (`settings.yaml:26-28`).
3. On `NewEpoch`, `Behaviour::start_new_epoch` (`with_core/behaviour/mod.rs:285-319`) closes pending upgrades, replaces `current_epoch_info` and `proofs_verifier`, and moves every negotiated peer with its message cache into `OldEpoch` (`old_epoch.rs:40-69`), which keeps its own `epoch`, `num_blend_layers` and `proofs_verifier`. From then on every upgraded connection is a new-epoch connection.
4. A received message is first offered to the old epoch, which claims it if and only if it arrived on one of its connections (`mod.rs:1004-1026`, `old_epoch.rs:245-276`); otherwise the current context decodes and verifies it (`mod.rs:1028-1046`, `utils.rs:88-135`). The decode context is per epoch already: `num_blend_layers` is passed from whichever context owns the connection (`utils.rs:102-104`, `old_epoch.rs:262`).
5. Outbound, the service pairs the current and the previous processor and scheduler (`epoch_stages/running.rs:172-238`), releases old-epoch messages under the old epoch (`DuringTransitionEvent::PreviousEpochReleaseRound`, `:241-274`), and the behaviour routes them to old-epoch peers only (`mod.rs:866-878`, `:904-922`, `old_epoch.rs:76-117`).
6. `finish_epoch_transition` closes the old substreams (`mod.rs:321-334`, `old_epoch.rs:155-165`). Two connections to the same peer in different epochs coexist; the peer-id ordering rule (`mod.rs:620-725`) only arbitrates duplicates within the current epoch.

A version byte is therefore one more per-epoch input, exactly like the PoQ public inputs the spec already binds to the epoch ("their validity is bound to the epoch in which they were generated", `blend-protocol.md:600`). Putting `version(e)` in each context gives, for free, the only overlap the protocol needs: during `T` of epoch `E_V`, old connections carry `V − 1` and new connections carry `V`. A within-epoch set `{N, N+1}` would add state, would require a second and third coordinated switch (start emitting, stop accepting) with no signal to trigger them, and would open the downgrade window discussed under LB-003. The `T ≥ T_M` constraint that makes the overlap sufficient is the spec's existing one (`blend-protocol.md:596`); the node does not enforce it (#156).

*Epoch arithmetic.* Round `r` of epoch `e` is round `e·E + r` overall.

| Epoch | Connections of epoch `e − 1` (old, alive for rounds `0..T` of `e`) | Connections of epoch `e` | Generated |
|---|---|---|---|
| `e ≤ E_V − 1` | `V − 1` | `V − 1` | `V − 1` |
| `e = E_V`, rounds `0..T` | `V − 1` (old verifier, old cache) | `V` | `V` |
| `e = E_V`, rounds `≥ T` | closed | `V` | `V` |
| `e > E_V` | `V` | `V` | `V` |

A message generated at round `r` of `E_V − 1` is processed by its last hop no later than round `r + T_M`; it crosses the boundary only if `r > E − T_M`, and then it is carried and verified on old connections during `T`, as its PoQ already is. No message of version `V − 1` can be valid after round `T` of `E_V`, because its PoQ is bound to `E_V − 1`. The nullifier cache is per context and is untouched. Blending tokens and rewards are per epoch and are untouched unless the Activity Proof format changes, which has its own version byte (`blend-protocol.md` §Active Message).

*Constraint on `E_V`.* `E_V` must be later than the epoch in which the release implementing `V` is published, by enough epochs for every operator of a core node in the SDP set of `E_V` to deploy it; a core node that still generates `V − 1` at `E_V` has every message discarded and every connection closed by the decode rule below, and cannot decode the network's messages. A node that learns `E_V` from its deployment file cannot run past `E_V` on a binary that does not implement `V`, so the file and the binary must be released together; a compiled-in activation table would remove that coupling.

*Decode rule (issue item 2).* Today `PublicHeader::decode` has `Context = ()` and compares the byte to a constant (`public_header.rs:121-147`); `EncapsulatedMessage::decode` has `Context = NonZeroU64` for the layer count (`encapsulated.rs:143-160`) and forwards `()` to the header. The proposal:

```rust
// blend/message
pub struct DecodeContext { pub version: u8, pub num_blend_layers: NonZeroU64 }
impl BinaryDecode for PublicHeader { type Context = u8; /* expected version */ }
impl BinaryDecode for EncapsulatedMessage { type Context = DecodeContext; }
pub enum Error { /* … */ VersionMismatch { expected: u8, got: u8 }, /* … */ }

pub fn deserialize_encapsulated_message(bytes: &[u8], ctx: &DecodeContext) -> Result<EncapsulatedMessage, Error> {
    match bytes.first() {
        Some(&got) if got == ctx.version => {}
        Some(&got) if (1..=LATEST_KNOWN_VERSION).contains(&got) || got > LATEST_KNOWN_VERSION =>
            return Err(Error::VersionMismatch { expected: ctx.version, got }),
        _ => return Err(Error::MessageDeserializationFailed),
    }
    /* existing decode, with ctx.num_blend_layers */
}
```

`Error::VersionMismatch` maps to a new `ReceiveError::VersionMismatch` in `utils.rs:101-104`, which `handle_received_serialized_encapsulated_message` (`mod.rs:1039-1043`) closes without a `SpamReason`: no block-list entry (`swarm.rs:427-434`), a `warn` log carrying `expected`, `got` and the peer, and a counter. `got > LATEST_KNOWN_VERSION` is the "peer is ahead, upgrade required" case and is logged as such. Every other first byte, including `0`, stays `UndeserializableMessage` and spammy, as `blend-protocol.md:885` requires for a malformed header. The distinction costs an attacker nothing: a core peer that sends a wrong version byte loses the connection on the first message either way; the block list only matters for an honest peer at the other version, which must be able to reconnect once it has upgraded (#101, #117). The same rule applies to the old context (`old_epoch.rs:255-263`, with that epoch's version) and, with a `debug` log and a counter instead of the current `trace` (`with_edge/behaviour/mod.rs:202-207`), to the edge receive path.

With #235's protocol name `<protocol_name>/v<version(e)>/l<β>/epoch/<e>`, a version-skewed core peer fails `multistream-select` before any message is exchanged and never reaches this rule (#116 Appendix B.2); the decode rule remains the defence for a peer that negotiated correctly and then sends the wrong byte, which is a malicious peer or a bug. #235 item 1 writes the version into the name as a constant (`v1`); it must be `version(e)` so that the name and the accepted byte switch together at `E_V`.

*Edge emission rule (issue item 3).* An edge node emits `version(e)` for the epoch `e` it is on. It already derives `e` from the same slot clock as core nodes (`edge/mod.rs:241-256` calls `membership::chain::subscribe`, `:148`), builds a fresh `MessageHandler` and backend per epoch (`edge/current_epoch.rs:125-143`, `:183-232`; `edge/handlers.rs:51-80`), and its encapsulation processor takes the epoch (`handlers.rs:60-67`), so `version(e)` is available where the public header is built. "The lowest version accepted by the whole core set" is `version(e)` by construction: every core node of epoch `e` accepts exactly it, on the entry connection and on every relay hop after it, which is what matters, since the entry node is not the route. What the edge node can observe today:

- It has no negotiation: `open_stream(peer_id, protocol_name)` with one name, one write, one close (`edge/backends/libp2p/swarm.rs:583-620`), and `SendOutcome::Sent` is reported when the bytes are written (`:619`), not when the core node accepted them. The core-to-edge handler sends nothing back (`with_edge/behaviour/handler/receiving.rs:76-84` reports the message and drops the state); a core node that ignores the message does so at `trace` (`with_edge/behaviour/mod.rs:202-207`). The only signal is the delivery-failure detector after `T_M` (`edge/mod.rs:364-371`, `:400`; `delivery/mod.rs`), whose reaction is a direct broadcast, or abstention with `abstain_on_failure`. A wrong version emitted by an edge leader is therefore a lost or deanonymised proposal, which is why the rule must be computable locally.
- `/blend/info` (`paths.rs:12`, `handlers.rs:665-684`) returns `NetworkInfo { node_id, core_info: Option<CoreInfo { current_epoch_peers: Vec<(PeerId, bool)>, old_epoch_peers }> }` (`services/blend/src/message.rs:16-29`, `swarm.rs:437-455`): neither the epoch nor a version. It is a local diagnostic, not something another node can query, so it cannot carry a discovery rule; it should still expose `epoch`, `message_version` and `E_V` for operators (S-002).
- The SDP declaration has no free field: `DeclarationMessage { service_type, locators, provider_id, zk_id, service_note_id }` (`core/src/sdp/mod.rs:479-486`), stored as `Declaration` (`:381-400`), reduced to `Node { id, address, public_key }` in the membership (`blend/membership/src/lib.rs:28-37`, `membership/service.rs:84-118`). Adding a supported-version field changes the `declaration_id` preimage (`:489-508`; spec §Declaration Storage) and the Mantle encoding of `SDP_DECLARE`, that is a consensus change, to publish information that the rule above makes redundant. A `Locator` cannot carry it either: the spec restricts it to the location part of a multiaddr and hashes its binary form (§Locators). Not recommended.
- With #235's name, the edge node does get feedback: `open_stream` with `…/v<version(e)>/…` on a core node at another version fails with `UnsupportedProtocol`, which is `SendOutcome::Failed` (`:590-597`) and enters the retry path (`:446-460`, `:333`). Under `version(e)` that only happens to a misconfigured node, and the log line should say "upgrade required".

*Unsigned version byte (issue item 4).* Assessed in LB-003: not exploitable across epochs under `version(e)`, exploitable in principle under a within-epoch set, and cheap to close by signing it in the next revision. The spec example is symbolic and already omits the version, so no spec test vector changes; two golden fixtures in the node do.

*Spec text (issue item 5).* Drafted in S-001.

**Out of scope**

The correctness of the Blend cryptography and proofs; the epoch-transition race between honest nodes (#116, #234, #236) and the block-list lifetime (#101, #117, #139), which this design relies on but does not re-verify; deployment parameter validation (#156); the header, sync and gossip version surfaces (#70, #180-#182); `libp2p` (`multistream-select`, `libp2p-stream`, `allow_block_list`), `ed25519-dalek`, `serde`, `bincode` are assumed correct.

**Assumptions**

The specifications at the stated logos-lips commit are the reference. Core connections are TLS-authenticated to the `provider_id` (§Connection Details), so the sender of every byte on a core connection is the neighbour itself. Nodes are built with the workspace release profile (`Cargo.toml:11-19`: no `overflow-checks`); nothing here depends on it. The slot clock is NTP-disciplined as `services/time` assumes; residual skew is #116's subject.

## 3. Method

- Manual review of the in-scope paths, working through issue `#183` (all five items) under parent `#10`, building on the reports for #70 (PR #174, LB-004 and LB-006), #101 (PR #115, #136), #116 (PR #233, LB-001 and Appendix B) and #117 (PR #164), and on issues #235 and #156.
- Spec conformance against `blend-protocol.md` §Time, §Network, §Messages (Overview tier), §Core Network, §Edge Network, §Relaying and §Processing (Protocol tier), §Notation, §Global Parameters, §Connection Details, §Connectivity Maintenance, §Transition Period, §Message Structure, §Generation, §Relaying, §Processing, §Failure Detection and Reaction (Details tier); `message-encapsulation.md` §Introduction, §Message Structure, §Message Encapsulation, §Message Decapsulation, §Example; `bedrock-service-declaration-protocol.md` §Service Parameters, §Message Timing, §Identifiers, §Locators, §Declaration Message, §Declaration Storage, §Default Service Parameters; `message-formatting.md`, `payload-formatting.md` and the two core overviews in full.
- Code claims of the issue re-verified at `a805329f`: `public_header.rs:10,128-133` (constant, decode-time check, plus a serde check at `:27-39`); `with_core/behaviour/mod.rs:1028-1045,729-751` (unchanged); `with_edge/behaviour/mod.rs:202-207` (unchanged); `encapsulated.rs:372-380` (unchanged); `paths.rs:12` (unchanged). No ⚑ repo items in this issue.
- Ruled out: a second version check after decode (none; `version` is read only by `into_components` and re-set by every constructor); any construction site that takes a version (none; all go through `PublicHeader::new`, `public_header.rs:42-53`, including decapsulation at `encapsulated.rs:543`); byte-level test vectors in `message-encapsulation.md` (none: the four long hex strings in the file are commit hashes in the timeline; the example at `:640` is symbolic and omits the version); any reply from the core-to-edge handler (none, `receiving.rs:76-84`); any per-epoch clearing of the block list at this commit (none; `swarm.rs:632-637` clears dial bookkeeping only, already reported as #101 LB-001 and fixed upstream in PR #3544); any use of identify `agent_version` or `protocol_version` (none, `config/network/serde/identify.rs:28`; #70 LB-002); a free field in `Declaration`, `ProviderInfo` or `Node` (none).
- Automated tooling run: none.
- Dynamic testing: none.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Spec deviation: the core-to-edge behaviour applies no Transition Period, so an edge message generated under the previous epoch is dropped as soon as the core node rotates | Data Validation | Low | Low | Open |
| LB-002 | The message version is a compile-time constant at every construction site, including the header reconstructed at decapsulation, so a message cannot carry a version other than the binary's | Configuration | Informational | Low | Open |
| LB-003 | The version byte is outside every signature, which is harmless under an epoch-keyed version and a downgrade vector under a within-epoch accept set | Cryptography | Informational | Medium | Open |

### LB-001 · Spec deviation: the core-to-edge behaviour applies no Transition Period, so an edge message generated under the previous epoch is dropped as soon as the core node rotates

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Data Validation |
| Target | `blend/network/src/core/with_edge/behaviour/mod.rs:L126-L140` (`start_new_epoch`), `:L195-L225` (`handle_received_serialized_encapsulated_message`), `:L232-L250` (`handle_poq_verification_outcome`); `blend/network/src/core/with_core/behaviour/mod.rs:L285-L319` for comparison |
| Status | Open |

**Description**

`blend-protocol.md` §Transition Period (`:598-603`) says that when a new epoch begins "the node validates message proofs against both new and past epoch-related public input for the duration of TP", and its §Edge Network (`:347-368`) makes an edge node open a connection, send one message and close. The core-to-core behaviour implements the rule by keeping the old verifier for old connections (`with_core/behaviour/mod.rs:307-316`). The core-to-edge behaviour does not: `start_new_epoch` replaces the verifier and closes every edge connection at once, with the stated intent that "edge nodes can retry with the new membership" (`:131-139`), and the receive path verifies every message against `self.current_epoch` and `self.proofs_verifier` only (`:217-223`). An edge message whose PoQ was generated under epoch `e − 1` and that reaches a core node after the node's slot tick for `e` fails verification and is dropped (`:242-249`).

The window is the difference between the core node's and the edge node's slot ticks plus the dial-and-send latency; the edge node itself drops its old backend and queued proposals on its own tick (`edge/current_epoch.rs:125-143`, `edge/handlers.rs:68-74`), so only messages already handed to the swarm are affected. The edge node observes nothing: the write succeeds (`edge/backends/libp2p/swarm.rs:619`), and after `T_M` the delivery-failure detector broadcasts the proposal directly (`edge/mod.rs:364-400`), which links the leader to its block, or abstains if `abstain_on_failure` is set.

For the version transition the consequence is that the edge path has no overlap window of its own: at `E_V` an edge node behind the core node's clock emits `V − 1` and is dropped for the same reason it is dropped today for its PoQ. The design in §2 does not depend on this window, but the deviation should be closed, or the spec amended to exclude edge connections from the rule, before the two mechanisms are stacked.

**Exploit scenario**

Not attacker-triggered. An edge leader whose clock is 500 ms behind a core node's wins the last slot of an epoch, encapsulates the proposal, dials and sends within that 500 ms; the core node has rotated and drops it; `T_M` later the leader broadcasts the block in the clear.

**Recommendation**
- *Short term*: in `with_edge`, keep the previous epoch's verifier for `epoch_transition_period` after `start_new_epoch` and, when verification against the current one fails during that period, verify once against the previous one and report the message under the old epoch (the swarm already carries `epoch` on `Event::Message`, `:234-239`). One extra Groth16 verification per failed edge message for `T` per epoch.
- *Long term*: with #235's epoch-bound protocol name, offer both `…/epoch/e−1` and `…/epoch/e` for inbound edge streams during `T`, so the epoch of an edge message is known at negotiation and no second verification is needed. State in `blend-protocol.md` §Transition Period whether the rule covers edge connections (S-001).

**References**: `blend-protocol.md` §Transition Period `:598-603`, §Edge Network `:347-368`, §Detection `:448`, §Direct Broadcast `:454`; #116 LB-001 (the core-to-core counterpart); #154 (delivery detector).

### LB-002 · The message version is a compile-time constant at every construction site, including the header reconstructed at decapsulation, so a message cannot carry a version other than the binary's

| | |
|---|---|
| Severity | Informational |
| Difficulty | Low |
| Category | Configuration |
| Target | `blend/message/src/message/public_header.rs:L10` (`LATEST_BLEND_MESSAGE_VERSION`), `:L27-L39` (`deserialize_version_number`), `:L42-L53` (`PublicHeader::new`), `:L121-L147` (`BinaryDecode`), `:L170-L184` and `:L279-L293` (`new` of the verified variants); `blend/message/src/encap/encapsulated.rs:L143-L160` (`EncapsulatedMessage::decode`), `:L543` (reconstructed header in `EncapsulatedPrivateHeader::decapsulate`); `blend/message/src/codec.rs:L41-L50`; `blend/network/src/core/with_core/behaviour/mod.rs:L142`, `old_epoch.rs:L46` (`num_blend_layers` as the only per-context decode input) |
| Status | Open |

**Description**

Every path that produces a `PublicHeader` sets `version: LATEST_BLEND_MESSAGE_VERSION` (`:42-53`), and the two verified variants obtain their version by building a `PublicHeader` first (`:176-177`, `:285-286`). The header a relay reconstructs from the first blending header when it decapsulates a layer is built the same way (`encapsulated.rs:543`), so the version a message leaves a hop with is the relay's constant, not the version it arrived with. Both decode paths compare against the same constant (`:32-38`, `:129-133`). The only per-context decode input the behaviours hold is `num_blend_layers` (`mod.rs:142`, `old_epoch.rs:46`), passed as the `NonZeroU64` context (`codec.rs:41-50`).

This is correct for one version and is not a deviation. It means that the transition in §2 cannot be implemented as a decode-side change alone: the version must become a value the per-epoch processor and the per-epoch network context carry, and the reconstruction at `:543` must copy it from the message rather than from the constant, or the version byte is rewritten at every hop and the signed-version rule of LB-003 cannot hold across hops.

**Exploit scenario**

None. Impact is on the shape of the change: without this plumbing a version bump is a flag day, as #70 LB-004 records.

**Recommendation**
- *Short term*: none for one version.
- *Long term*: with the change in §2, give `PublicHeader::new` and the verified constructors a `version: u8` parameter; carry `version(e)` in `CoreEpochPublicInfo` / `EpochCryptographicProcessor` (`edge/handlers.rs:60-67`) and in the behaviour contexts (`mod.rs:142`, `old_epoch.rs:46`) as the `DecodeContext` of §2; at `:543` reuse the outer header's version; apply the same rule to the serde path (`:27-39`), which has no epoch context and should accept every version the binary knows; add a fixture with version `2` under the new context and a test that a `1` on an epoch-`E_V` connection is `VersionMismatch`, not `UndeserializableMessage`.

**References**: `message-formatting.md` §Public Header `:70`; `message-encapsulation.md` `:467`, `:599` (`version=1` in both pseudocode constructors); #70 LB-004; #235 item 1.

### LB-003 · The version byte is outside every signature, which is harmless under an epoch-keyed version and a downgrade vector under a within-epoch accept set

| | |
|---|---|
| Severity | Informational |
| Difficulty | Medium |
| Category | Cryptography |
| Target | `blend/message/src/encap/encapsulated.rs:L372-L380` (`signing_body`), `:L71-L83` (`verify_header_signature`), `:L206-L240` (`EncapsulatedPart::encapsulate`, signature at `:L215`), `:L294-L297` (`sign`); `blend/message/src/fixtures/mod.rs:L147-L195` (`wire_fixture_message`), `:L197-L224`; `blend/message/src/fixtures/{encapsulated_message.hex,encapsulated_part.hex}` |
| Status | Open |

**Description**

The signature in the public header and the signature stored in each blending header are both computed over `private_header || payload` (`:372-380`, `:215`, `:296`); the version byte precedes the public header on the wire and is covered by nothing: not by these signatures, not by the proof of quota (bound to the signing key) and not by the proof of selection (bound to the key and the node index). #70 LB-006 records the fact; this finding assesses it for the two designs on the table.

*Under `version(e)` (§2).* A relay can rewrite the byte, and the next hop attributes the rewrite to the relay (TLS, §Connection Details) and closes the connection under the decode rule. To make the next hop *parse* the message under the other version's rules, the receiver would have to be in a context whose version is the other one, which during `T` of `E_V` is the old-epoch context on an old connection. A message accepted there must also carry a PoQ valid under the inputs of `E_V − 1`; a message generated at `E_V` does not, and one generated at `E_V − 1` is already a `V − 1` message. Outside `T` there is no context with the other version at all. The rewrite therefore only ever drops the message, which a relay can do by not forwarding it. Not exploitable.

*Under a within-epoch accept set `{N, N+1}`.* Both parsers are live on the same connection with the same PoQ inputs. If the two layouts have the same size and the same `num_blend_layers` (a message of another size fails to decode regardless), a relay that rewrites `N+1 → N` makes the next hop verify and process under the `N` rules; the sender's signature still verifies because it never covered the byte. Whether that is harmful depends on what `N+1` changed: a new PoQ circuit is caught by verification, a changed processing rule is not. This is the reason not to adopt the set.

*Closing it.* Prepend the version byte to `signing_body` in the next format revision: `version || private_header || payload`, for the public header and for every blending header, with the message's version carried through decapsulation (LB-002). Cost: one byte per signature; no size change on the wire. Impact on test vectors: none in the specification, whose example is symbolic and states "we are omitting protocol version in the header for simplicity" (`message-encapsulation.md:640`), and whose only long hex strings are commit hashes; two golden fixtures in the node, `encapsulated_message.hex` (public-header signature, `fixtures/mod.rs:187-191`) and `encapsulated_part.hex` (the blending header signed at `:174-180`), must be regenerated. `PUBLIC_HEADER_HEX` (`:107-120`), the `EncapsulatedPrivateHeader` and `EncapsulatedBlendingHeader` fixtures (`:74-90`) use constant or no signatures and are unchanged. Changing `signing_body` is itself a version bump, so the first signed version is `V = 2` and the fixture for `V = 1` stays as the legacy vector.

**Exploit scenario**

Only under a within-epoch accept set: a malicious core node relays a `V = 2` message with the byte set to `1`; the next hop, still accepting both, parses it as `V = 1` and applies the `V = 1` processing rules to a message the sender built for `V = 2`. Under `version(e)` the message is discarded and the relay's connection closed.

**Recommendation**
- *Short term*: adopt `version(e)` (§2), under which the byte does not need to be signed.
- *Long term*: sign it anyway in the next revision as above, and write it into `message-formatting.md` §Public Header and `message-encapsulation.md` `signing_body` (S-001), so that every later version is covered without a further format change.

**References**: `message-formatting.md` `:73`, `:102`; `message-encapsulation.md` `:506-508` (`signing_body`), `:640`; #70 LB-006.

## 5. Suggestions (non-security)

### S-001 · Specification text for the version transition

| | |
|---|---|
| Target | `blend-protocol.md` §Global Parameters `:502-513`, §Relaying (Protocol tier) `:401`, §Relaying (Details tier) `:885`, §Transition Period `:598-603`, §Generation (Details tier) `:858-876`; `message-formatting.md` §Public Header `:70`, `:73`; `message-encapsulation.md` `:506-508`, `:599` |

Proposed edits, in the house style (rules only; nothing restated across tiers).

`blend-protocol.md` §Global Parameters, append:

> - $`V = 1`$, the message version ([Message Formatting](message-formatting.md#public-header)).
> - $`E_V`$, the epoch from which messages carry version $`V`$. It is a deployment parameter. The version of epoch $`e`$ is $`\text{version}(e) = V`$ if $`e \ge E_V`$, else $`V - 1`$. $`E_V`$ must be later than the epoch in which the release implementing $`V`$ is published. A core node that generates $`\text{version}(e) - 1`$ at epoch $`e`$ has every message discarded and every connection closed ([Relaying](#relaying)).

`blend-protocol.md` §Relaying, Protocol tier, replace step 1.1:

> 1. The version of the message must be $`\text{version}(e)`$ for the epoch $`e`$ of the connection; if not, then discard the message and close the connection.

`blend-protocol.md` §Relaying, Details tier, insert before step 1.3 and renumber:

> 3. If the version of the message is not $`\text{version}(e)`$ for the epoch $`e`$ of the connection ([Transition Period](#transition-period)), then discard the message and close the connection. The neighbor is not marked as malicious.

`blend-protocol.md` §Transition Period, in the list "When a new epoch begins", append:

> - The message version is one of these epoch-bound inputs: a message on a connection of the past epoch must carry $`\text{version}(e-1)`$, and a message on a connection of the new epoch $`\text{version}(e)`$.
> - Edge connections opened during the Transition Period are verified against the new epoch first and, on failure, against the past epoch.

The last bullet is the resolution of LB-001; drop it if the team decides the rule does not cover edge connections, and say so instead.

`blend-protocol.md` §Generation, Details tier, step 4:

> 4. The message is formatted according to the [Message Formatting](message-formatting.md) with `version` set to $`\text{version}(e)`$ for the current epoch $`e`$ ([Global Parameters](#global-parameters)).

`message-formatting.md` §Public Header, replace the `version` bullet:

> - `version` is $`\text{version}(e)`$ for the epoch $`e`$ in which the message is generated ([Global Parameters](blend-protocol.md#global-parameters)); its current value is `0x01`.

If LB-003's signed byte is adopted with the next version, also replace the two `signature` bullets (`:73`, `:102`) with $`\sigma_{K^{n}_{i}}(V|\mathbf{h}|\mathbf{P}_i)`$ and update `message-encapsulation.md` `signing_body` (`:506-508`) to `bytes([version]) + b"".join(private_headers) + payload`, and the reconstructed header at `:599` to carry the received message's version rather than `version= 1`. `message-encapsulation.md` §Message Structure `:179` and `:189` say "set to 1" and would read "set to $`\text{version}(e)`$" with the same link.

Related wording already requested upstream: the connection-per-epoch statement and the protocol-name rule (logos-lips #451, #116 S-003, #235 item 4); the name should carry $`\text{version}(e)`$, not a constant.

### S-002 · Expose the epoch, the message version and the activation epoch in `/blend/info`

| | |
|---|---|
| Target | `services/blend/src/message.rs:L16-L29` (`NetworkInfo`, `CoreInfo`), `services/blend/src/core/backends/libp2p/swarm.rs:L437-L455` (`collect_network_info`), `nodes/node/binary/src/config/blend/deployment.rs:L88-L94` (`CommonSettings`) |

`NetworkInfo` carries peers only. Adding `epoch`, `message_version` (= `version(epoch)`) and `activation: Option<(Epoch, u8)>` to it, and `E_V` to `CommonSettings` next to `protocol_name`, lets an operator see on which side of `E_V` a node is before the boundary, which is the only readiness check available until identify carries a release string (#70 LB-002). Together with the `VersionMismatch` counter of §2 this replaces the `trace`-level silence of the edge path (`with_edge/behaviour/mod.rs:205`).

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
