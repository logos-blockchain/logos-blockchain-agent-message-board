# Audit Report — Wire format versioning and upgrade path across header, blend, sync, and gossip

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/70`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `core/src/header, blend/message, blend/network, consensus/cryptarchia-sync, libp2p, services/network, services/storage, services/chain, services/tx-service, logos_sql, tools/config, deployment/ceremony`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md, overview-cryptoeconomics.md, network-wire-format.md, mantle-transaction-encoding.md, message-formatting.md, payload-formatting.md` (in full); `cryptarchia-v1-protocol.md, bedrock-v1.1-block-construction.md, cryptarchia-v1-bootstr-sync.md, blend-protocol.md, message-encapsulation.md, bedrock-v1.1-mantle-specification.md, p2p-network-bootstrapping.md` (by section)
Date: `2026-09-11` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: every version surface fails closed on an unknown value, which is correct for launch, but nothing in the node or the specifications lets two versions coexist: protocol identifiers are namespaced by release, each libp2p behaviour negotiates exactly one protocol, the blend network classifies a newer-version peer as spammy, and the on-disk formats are the wire formats, so the first post-launch codec change is a coordinated flag day that also invalidates every node's database. One surface has already been bumped without a compatibility path: λSQL inscriptions moved from payload version 1 to 2 on 2026-08-27, and a replica reading a version it does not know records the write as a deterministic rejection, so replicas at different versions diverge silently.
- Findings: 0 critical · 0 high · 1 medium · 3 low · 3 informational
- Key themes: "no negotiated upgrade path", "on-disk format is the wire format", "unknown version is indistinguishable from malformed input"
- Must-fix before launch: none. Before the first post-launch codec change: LB-001 (λSQL legacy readers), LB-002 and LB-004 (multi-protocol negotiation and a blend version transition), LB-003 (storage format version).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `core/src/header/mod.rs` | `Version` enum, encode/decode, serde paths, use in `Header::id()` and signing |
| `core/src/block/mod.rs`, `core/src/codec/` | `Block` serde, `SerializeOp`/`DeserializeOp`, bincode options |
| `core/src/sdp/mod.rs`, `core/src/sdp/blend.rs` | `ActivityMetadata` type byte and `ActivityProof` version byte |
| `blend/message/src/message/public_header.rs`, `blend/message/src/encap/encapsulated.rs`, `blend/message/src/message/payload.rs` | message version byte, signature coverage, payload type |
| `blend/network/src/core/with_core/`, `blend/network/src/core/with_edge/` | protocol negotiation, handling of undecodable messages |
| `consensus/cryptarchia-sync/src/` | protocol registration, message enums, framing |
| `libp2p/src/protocol_name.rs`, `libp2p/src/config/{kademlia,identify,gossipsub}.rs`, `libp2p/src/behaviour/gossipsub/swarm_ext.rs` | protocol names, identify, topics |
| `services/network/src/backends/libp2p/swarm/identify.rs` | what identify information is acted on |
| `services/chain/chain-network/src/network/adapters/libp2p.rs`, `services/tx-service/src/network/adapters/libp2p.rs`, `services/blend/src/core/dispatcher/libp2p.rs` | gossip ingress decode and failure handling |
| `services/storage/src/api/backend/rocksdb/chain.rs`, `services/storage/src/recovery.rs`, `services/chain/chain-service/src/states.rs`, `services/chain/chain-service/src/storage/adapters/storage.rs`, `services/tx-service/src/storage/adapters/rocksdb.rs` | stored-at-rest formats |
| `logos_sql/src/protocol/mod.rs`, `logos_sql/src/applier.rs` | inscription payload version and its handling |
| `tools/config/src/{release,deployment}.rs`, `deployment/ceremony/genesis/*/deployment-template.yaml`, `nodes/node/binary/src/config/deployment/settings.yaml`, `.github/workflows/genesis-ceremony.yml` | how protocol identifiers and topics are produced |

**Version surfaces inventory** (the deliverable of the issue; "today" is the target commit)

| # | Surface | Where the value lives | Unknown value today | Recommended mechanism | Spec sections it touches |
|---|---|---|---|---|---|
| 1 | Block header `bedrock_version` (1 byte, value 1) | `core/src/header/mod.rs:20,49-74,127-139`; validated only at decode, in both the `BinaryDecode` and the serde path | `DecodeError::unknown_discriminant`; the proposal (gossip, `chain-network/.../libp2p.rs:232-238`) or the synced block (`libp2p.rs:373-377`) is dropped with a debug log; no peer penalty; nothing tells the operator the chain has moved on | Keep decode fail-closed for consensus. Add a height-activation table `(version, activation_height)` to the deployment config, and make rule 1 of header validation `version == active_version(height)`. At gossip and sync ingress, peek the first byte before full decode: a value greater than any known version raises an "upgrade required" alarm (metric + error log) and stops the node from proposing; it must not be recorded as a verdict on any block id. Relay needs no change: gossipsub forwards the raw frame before the application decodes it (`validate_messages: false`), so old nodes already propagate newer proposals | `cryptarchia-v1-protocol.md` "Versioning and Protocol Upgrades", "Block Header Validation" rule 1; `bedrock-v1.1-block-construction.md` "Header" (`fixed to 0x01`), "Canonical Encoding", "Block Proposal Validation" step 2; `cryptarchia-v1-bootstr-sync.md` "Downloading Blocks" |
| 2 | Blend `PublicHeader.version` (1 byte, value 1) | `blend/message/src/message/public_header.rs:10,128-133` (BinaryDecode), `:27-39` (serde) | core neighbour: `ReceiveError::UndeserializableMessage` → `SpamReason::UndeserializableMessage` → connection marked spammy and closed (`with_core/behaviour/mod.rs:1028-1045,729-740`); edge sender: message ignored at trace level (`with_edge/behaviour/mod.rs:202-206`) | Two-phase rollout keyed on the epoch boundary that already drives the Transition Period: release A accepts `{1,2}` and emits 1; once every core node in the SDP set runs A (observable from identify `agent_version`), release B emits 2; release C drops 1. During the window a version that is unknown but greater than the current one is discarded without the spammy classification. Edge nodes emit the lowest version accepted by the whole core set, which the node must expose (`/blend/info`) | `blend-protocol.md` "Relaying" 1.1 and "Relaying" step 1.3, "Transition Period", "Connection Details"; `message-formatting.md` "Public Header"; `message-encapsulation.md` "Message Structure" |
| 3 | SDP `ActivityMetadata` type byte and blend `ActivityProof` version byte | `core/src/sdp/mod.rs:583-598`, `core/src/sdp/blend.rs:35,57-69` | `unknown_discriminant` / `invalid_value`; the transaction does not decode | These are consensus data (part of `SDP_ACTIVE`, executed by the ledger): a new version is a Mantle change and follows surface 1's height activation. Keep both bytes distinct as the spec already requires | `blend-protocol.md` "Active Message"; `bedrock-service-declaration-protocol.md` "Active Message" |
| 4 | libp2p stream protocol identifiers: kademlia, identify, chainsync, blend | produced as `/logos-blockchain-<env>-<version>/<name>/1.0.0` by the genesis ceremony (`deployment-template.yaml:5,17-19`, `genesis-ceremony.yml:14-16,35-38`) or `ProtocolIdentity` (`tools/config/src/release.rs:21-38,51-53`); each behaviour accepts exactly one (`cryptarchia-sync/src/libp2p/behaviour.rs:160-165`, `libp2p/src/config/kademlia.rs:71-72`, `blend/network/.../with_core/behaviour/handler/mod.rs:160-166`) | multistream-select fails; the peer is simply not usable for that protocol. Identify only uses the peer's protocol list to decide whether to add it to the Kademlia table (`services/network/.../identify.rs:25-52`) | Put the chain identity (genesis id or `chain_id`) in the namespace and the protocol version in the trailing segment: `/logos-blockchain/<chain-id>/chainsync/1.1.0`. Register every supported version (`Control::accept` per protocol; `kad::Config::set_protocol_names`; a `Vec<StreamProtocol>` upgrade for blend) and let multistream-select pick the highest common one. Use identify's `protocol_version`/`agent_version` to report the peer's release, which today is `None` (`libp2p/src/config/identify.rs:43-50`) | `cryptarchia-v1-bootstr-sync.md` "Downloading Blocks" (protocol ID); `blend-protocol.md` "Connection Details"; `p2p-network-bootstrapping.md` step 2 ("verify ... protocol compatibility" is not defined) |
| 5 | Gossipsub topics: cryptarchia proposals, mempool transactions | plain `IdentTopic` strings (`libp2p/src/behaviour/gossipsub/swarm_ext.rs:15,56`), namespaced the same way as surface 4 (`deployment-template.yaml:85,104`); gossipsub's own protocol id is the libp2p default (no `protocol_id` customisation in `libp2p/src/config/gossipsub.rs`) | a node on the other topic never sees the message; on the same topic, a message that does not decode is dropped with a debug log (`chain-network/.../libp2p.rs:232-238`, `tx-service/.../libp2p.rs:71-77`) and, because `validate_messages` is false, was already forwarded | Topics are cheap to run in parallel: during a transition subscribe to both `<name>/N` and `<name>/N+1`, publish on the old one until activation height, then on the new one, then unsubscribe from the old one. The topic string itself does not need a version if surface 1 versions the payload | no spec names the topics; `cryptarchia-v1-bootstr-sync.md` "Listening for New Blocks" is the natural place |
| 6 | bincode enum tags in sync messages | `cryptarchia-sync/src/libp2p/messages.rs:14-20,131-139`, `cryptarchia-sync/src/messages.rs:14-24`, `cryptarchia-sync/src/lib.rs:12-18`; fixint little-endian `u32` tags (`core/src/codec/bincode/mod.rs:23-29`) | bincode error → `PackingError::Serialization` → the stream is closed and the download attempt fails (`packing.rs:75`, `downloader.rs:108-114`, `provider.rs:31-33`); no peer penalty | Version the protocol name (surface 4) rather than the messages, and apply the compatibility rules in LB-007 to any message type kept across versions | `cryptarchia-v1-bootstr-sync.md` "Downloading Blocks"; `network-wire-format.md` "Encoding and Decoding" |
| 7 | λSQL inscription `PAYLOAD_VERSION` (u16, value 2) | `logos_sql/src/protocol/mod.rs:18-20,261-262,284-286` | `Error::InvalidPayload("protocol version is not supported")`, treated as a deterministic rejection and skipped (`applier.rs:363-366,419-421`); on live-suffix rebuild it is `InvalidLocalState` and the rebuild aborts (`applier.rs:327-328`) | Keep one decoder per historical version, dispatch on the version, never re-encode stored payloads. A version greater than the highest known is "upgrade required" (halt), never a rejection. See LB-001 | none: the λSQL payload has no specification (S-003) |
| 8 | Stored-at-rest formats: blocks, block events, consensus state (with `LedgerState`), mempool items, λSQL live suffix | `Block::to_bytes()` is the value stored under the block id (`core/src/block/mod.rs:381-389`, `services/storage/src/api/backend/rocksdb/chain.rs:45-70`); `CryptarchiaConsensusState` is bincode via `SerializeOp` (`services/storage/src/recovery.rs:106,122-127`, `chain-service/src/states.rs:10-30`); mempool items via `Item::to_bytes()` (`tx-service/src/storage/adapters/rocksdb.rs:51,91`); λSQL keeps the raw payload (`applier.rs:305-310`) | not applicable: a stored value never carries a version, so after a wire change the old value fails to decode and the block, state or item is silently absent or the node fails to recover | A `meta/schema_version` key per database plus a startup migration step, and a storage-specific type for the recovered consensus state. See LB-003 | none: storage is unspecified, which is fine; the constraint belongs in the node's own docs |

**Out of scope**

Codec safety on untrusted input (bounds, panics, canonical encoding), which the report for #56 covers; the correctness of the blend cryptography and proofs; the content of the ledger rules that a version bump would carry; `libp2p` (multistream-select, gossipsub, kademlia, identify), `bincode`, `rocksdb`, `rusqlite`, `serde` are assumed correct.

**Assumptions**

The specifications at the stated logos-lips commit are the reference. Nodes are built with the workspace release profile (`Cargo.toml:11-19`: `lto = "fat"`, `strip = true`, no `overflow-checks`), so nothing here relies on a panic surfacing to an operator. Operators run the binary produced from the committed `settings.yaml` of a genesis ceremony, so the protocol identifiers in the templates are the ones on the wire. No item in this report is triggered by an attacker; they are triggered by the project's own first upgrade.

## 3. Method

- Manual review of the in-scope paths, working through issue `#70` (all six items) under parent `#10`, with the repo-level facts of `#19` re-verified at the target commit (`Cargo.toml:11-19,308-348`).
- Spec conformance against `cryptarchia-v1-protocol.md` §"Block Header", §"Block Header Validation", §"Versioning and Protocol Upgrades"; `bedrock-v1.1-block-construction.md` §"Block Proposal", §"Header", §"Canonical Encoding", §"Block", §"Block Proposal Reconstruction", §"Block Proposal Validation"; `cryptarchia-v1-bootstr-sync.md` §"Listening for New Blocks", §"Downloading Blocks", §"Checkpoint Provider HTTP API"; `blend-protocol.md` §"Core Network", §"Edge Network", §"Transition Period", §"Connection Details", §"Relaying" (both tiers), §"Processing", §"Message Structure", §"Active Message"; `message-encapsulation.md` §"Message Structure"; `bedrock-v1.1-mantle-specification.md` §"Opcodes", §"CHANNEL_INSCRIBE"; `p2p-network-bootstrapping.md` §"Protocol", §"Details"; plus `network-wire-format.md`, `mantle-transaction-encoding.md`, `message-formatting.md`, `payload-formatting.md` and the two core overviews in full. A grep of every `raw/*.md` for "version" found no further version surface.
- ⚑ repo items re-verified at `a805329f`: `core/src/header/mod.rs:49-74` (one variant, decode-time check in both codec paths), `blend/message/src/message/public_header.rs:128-133` (unchanged), `libp2p/src/protocol_name.rs` (a serde wrapper only; the names come from config), `consensus/cryptarchia-sync/src/libp2p/messages.rs` (unchanged), `logos_sql/src/protocol/mod.rs:19,284-286` — changed: `PAYLOAD_VERSION` went from 1 to 2 in `c33f4d2b3` (2026-08-27, "capture nondeterministic function results (#3361)"), which is the trigger for LB-001.
- Ruled out: gossipsub uses a custom `protocol_id` (it does not; only topics are configured); identify compares `protocol_version` or `agent_version` (it does not; `agent_version` defaults to `None`); any second `Version` check after decode (none: `version()` has no caller outside the module); any decode path for a stored block other than `Block::from_bytes` (none); any existing legacy decoder in `logos_sql` (none); a version segment on the λSQL marker (`LOGOS_SQL` is fixed and `is_logos_sql_payload` checks only the marker, `protocol/mod.rs:341-343`, so a future version is still routed to the λSQL applier and rejected there).
- Automated tooling run: none.
- Dynamic testing: none.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | λSQL inscriptions of another payload version are recorded as deterministic rejections, and a stored replay payload of an old version aborts the live rebuild | Data Validation | Medium | Low | Open |
| LB-002 | Every wire change is a network-wide flag day: protocol identifiers are namespaced by release and each behaviour negotiates exactly one protocol | Configuration | Low | Low | Open |
| LB-003 | Stored-at-rest formats are the wire formats, carry no format version, and fail silently after a wire change | Denial of Service | Low | Low | Open |
| LB-004 | A blend message of a newer version marks the sending core node as spammy and closes the connection, so a version bump partitions the mix network | Denial of Service | Low | Low | Open |
| LB-005 | Spec deviation: libp2p protocol identifiers on the wire differ from the strings the specifications fix | Configuration | Informational | Low | Open |
| LB-006 | The blend public-header version byte is not covered by the header signature | Cryptography | Informational | Medium | Open |
| LB-007 | Sync message enum tags are positional bincode discriminants with no stated compatibility rule | Data Validation | Informational | Low | Open |

### LB-001 · λSQL inscriptions of another payload version are recorded as deterministic rejections, and a stored replay payload of an old version aborts the live rebuild

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Data Validation |
| Target | `logos_sql/src/protocol/mod.rs:L19,L284-L286` (`ChannelInscription::decode`), `logos_sql/src/applier.rs:L363-L366,L419-L421` (`apply_inscription`, `is_rejected_write`), `logos_sql/src/applier.rs:L327-L328` (`rebuild_live_from_suffix`) |
| Status | Open |

**Description**

An inscription is permanent chain history, but the decoder accepts exactly one payload version and the version has already moved once. `PAYLOAD_VERSION` was 1 when the protocol landed (`d94703cd7`) and is 2 since `c33f4d2b3` (2026-08-27); no reader for version 1 remains:

```rust
// logos_sql/src/protocol/mod.rs:19
const PAYLOAD_VERSION: u16 = 2;
// logos_sql/src/protocol/mod.rs:284-286
if version != PAYLOAD_VERSION {
    return Err(Error::InvalidPayload("protocol version is not supported"));
}
```

`applier.rs:419-421` classifies every `Error::InvalidPayload` as a rejection "that every replica must reject" (`applier.rs:383`), so `apply_inscription` records the write as rejected and moves on (`applier.rs:363-366,400-417`). That classification is only true when every replica runs the same version. A replica still on version 1 records every version-2 write as rejected and continues; a replica on version 2 applies it. Both believe they are in sync with the channel and their `LIVE.db` diverge with no error. Conversely, every version-1 inscription that is already on chain is now permanently rejected by the current code, which is a loss of zone history rather than a migration.

The second path is worse. `collect_adopted_suffix` keeps the raw payload of every adopted write (`applier.rs:305-310`), and `rebuild_live_from_suffix` re-decodes those payloads later (`applier.rs:327-328`), returning `Error::InvalidLocalState("stored live suffix payload is malformed")` on failure. After a node upgrade that bumps the version, every payload written before the upgrade is "malformed", and the rebuild that runs on the next channel branch change fails permanently.

**Exploit scenario**

Not attacker-triggered. Operator A upgrades a λSQL replica to a release with `PAYLOAD_VERSION = 3`; operator B does not. The sequencer on release A inscribes a version-3 write. B's replica records it as rejected and serves stale query results; A's replica applies it. Neither logs above `warn`. When B later upgrades, B's stored suffix still holds version-2 payloads; the first reorg after the upgrade aborts `rebuild_live_from_suffix`, and the replica cannot recover without a manual rebuild from genesis.

**Recommendation**
- *Short term*: keep `decode_v1` alongside `decode_v2` and dispatch on the version read at `mod.rs:282`. Treat a version greater than the highest known as a fatal "upgrade required" error, not as `InvalidPayload`, so it is neither recorded as a rejection nor skipped. Re-decode stored suffix payloads with the same dispatch.
- *Long term*: specify the λSQL payload (S-003) with the rule that a version is never removed from readers, and add a test that decodes a fixture from every historical version.

**References**: `bedrock-architecture-overview.md` "Mantle Channels" (inscriptions are permanent); `bedrock-v1.1-mantle-specification.md` "CHANNEL_INSCRIBE".

### LB-002 · Every wire change is a network-wide flag day: protocol identifiers are namespaced by release and each behaviour negotiates exactly one protocol

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Configuration |
| Target | `deployment/ceremony/genesis/testnet/deployment-template.yaml:L5,L17-L19,L85,L104`; `.github/workflows/genesis-ceremony.yml:L14-L16,L35-L38`; `tools/config/src/release.rs:L21-L38,L51-L53` (`ProtocolIdentity`); `consensus/cryptarchia-sync/src/libp2p/behaviour.rs:L160-L165` (`Behaviour::new`); `libp2p/src/config/kademlia.rs:L71-L72`; `blend/network/src/core/with_core/behaviour/handler/mod.rs:L160-L166`; `libp2p/src/behaviour/gossipsub/swarm_ext.rs:L15` |
| Status | Open |

**Description**

Every protocol identifier and both gossip topics are produced as `/logos-blockchain-<env>-<version>/<name>/1.0.0`, where `<version>` is the "Protocol version string" typed into the genesis ceremony (`genesis-ceremony.yml:14-16`) and substituted into the template (`:35-38`), or derived from `LOGOS_BLOCKCHAIN_PROTOCOL_RELEASE` by `ProtocolIdentity::from_env` (`release.rs:21-38`). The trailing `1.0.0` never changes. The version therefore sits in the namespace, which is the part multistream-select must match exactly, and every behaviour registers a single name:

```rust
// consensus/cryptarchia-sync/src/libp2p/behaviour.rs:163-165
let incoming_streams = control
    .accept(protocol_name.clone())
    .expect("Failed to accept incoming streams for sync protocol");
// libp2p/src/config/kademlia.rs:72
let mut config = kad::Config::new(protocol_name);
```

The blend handler advertises one `ReadyUpgrade<StreamProtocol>` (`handler/mod.rs:160-166`), and gossip topics are single `IdentTopic` strings (`swarm_ext.rs:15`). A node at version N and a node at version N+1 share no Kademlia protocol, so they never enter each other's routing table (`identify.rs:29-33` adds a peer only when a Kademlia protocol name matches), cannot sync, cannot exchange blend messages, and publish on different topics. Upgrading is therefore an all-at-once cut-over: the first operators to upgrade lose every peer until a majority follows, and a node that lags stays on a dead network with no signal other than empty peer lists.

**Exploit scenario**

Not attacker-triggered. A release that changes any wire detail ships with a new `<version>`. During the rollout window the network is two disconnected networks, each with a fraction of the stake; blocks proposed on the smaller one are wasted, and the blend anonymity set is the fraction that has upgraded. If the window is long, the smaller side keeps extending its own chain and must be reorganised away when it rejoins.

**Recommendation**
- *Short term*: register every supported protocol version rather than one: call `Control::accept` for each name in a `Vec<StreamProtocol>` (`behaviour.rs:160-165`), `kad::Config::set_protocol_names` after `new` (`kademlia.rs:72`), and give the blend handler a list of upgrades. Put the chain identity in the namespace and the protocol version in the trailing segment (`/logos-blockchain/<chain-id>/chainsync/1.1.0`), so that the namespace is stable for the life of a chain and multistream-select selects the highest common version. Subscribe to both gossip topics for the transition window.
- *Long term*: set identify `agent_version` to `BuildVersionInfo` (`libp2p/src/config/identify.rs:48-50`, `nodes/version/src/lib.rs`) and expose the peers' versions in `/network/info`, so an operator can see when the network is ready for the next phase of a rollout.

**References**: `cryptarchia-v1-bootstr-sync.md` "Downloading Blocks" (Libp2p Protocol ID); `blend-protocol.md` "Connection Details"; `p2p-network-bootstrapping.md` step 2.

### LB-003 · Stored-at-rest formats are the wire formats, carry no format version, and fail silently after a wire change

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `core/src/block/mod.rs:L366-L389` (`TryFrom<Bytes> for Block`, `TryFrom<Block> for Bytes`); `services/storage/src/api/backend/rocksdb/chain.rs:L45-L70` (`store_block_data`); `services/storage/src/recovery.rs:L106,L122-L127` (`load_state`, `save_state`); `services/chain/chain-service/src/states.rs:L10-L30` (`CryptarchiaConsensusState`); `services/tx-service/src/storage/adapters/rocksdb.rs:L51,L91`; `services/chain/chain-service/src/storage/adapters/storage.rs:L76-L91,L248` |
| Status | Open |

**Description**

The block stored under its header id is exactly `Block::to_bytes()` (`block/mod.rs:386-388`), the same bytes that `SerialisedBlock` carries over the sync stream (`cryptarchia-sync/src/messages.rs:11-12`). The recovered consensus state, which embeds the full `LedgerState` (`states.rs:14`), is written and read with the same bincode `SerializeOp` (`recovery.rs:106,125`). Mempool items use `Item::to_bytes()` (`rocksdb.rs:51`) and are read back with `from_bytes(..).ok()` (`:91`). None of these values is prefixed with a format version, and no database holds a schema version key (`chain.rs:27-29` defines only data prefixes).

The consequence is that every wire change listed in the inventory is also a storage change. A new header version (surface 1) or a new `Op` variant changes the bincode layout of `Block`, and every stored block then fails `Block::from_bytes`; `get_block` maps that failure to `None` (`storage.rs:84-87`), so the block "does not exist" as far as the chain service is concerned. A change to `LedgerState` makes `load_state` fail and the node starts from genesis. Mempool items that no longer decode are dropped by `filter_map` (`rocksdb.rs:91`) without a log line.

**Exploit scenario**

Not attacker-triggered. A release adds one variant to `Op`. Every operator who upgrades in place finds that `get_block` returns `None` for the whole chain; the sync provider (`block_provider.rs:68`, `TryInto<Block<Tx>>` bound) serves nothing, and the node either re-downloads the chain from peers that have not upgraded, if any remain (LB-002), or cannot bootstrap at all.

**Recommendation**
- *Short term*: write a `meta/schema_version` key on database creation and refuse to open a database whose version is not the binary's, with a clear error naming the migration step. Prefix stored values with a one-byte format version, or wrap them in a storage-only envelope.
- *Long term*: give `CryptarchiaConsensusState` a storage-specific serialisation, independent of the wire codec, and add a migration framework that runs on startup. Keep a decode test against a fixture database from every released version.

**References**: `network-wire-format.md` "Encoding and Decoding" (bincode is specified for transport framing, not for storage).

### LB-004 · A blend message of a newer version marks the sending core node as spammy and closes the connection, so a version bump partitions the mix network

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `blend/network/src/core/with_core/behaviour/mod.rs:L1028-L1045` (`handle_received_serialized_encapsulated_message_and_update_cache` error arm), `:L729-L751` (`close_spammy_connection`, `set_connection_to_spammy`); `blend/message/src/message/public_header.rs:L128-L133`; `blend/network/src/core/with_edge/behaviour/mod.rs:L202-L206` |
| Status | Open |

**Description**

A public header whose version byte is not `LATEST_BLEND_MESSAGE_VERSION` fails to decode (`public_header.rs:129-133`). In the core-to-core behaviour that is `ReceiveError::UndeserializableMessage`, and the receiver marks the connection `NegotiatedPeerState::Spammy` and closes it (`mod.rs:1039-1044`, `:738-739`). This matches the specification (`blend-protocol.md` "Relaying" 1.3: "discard the message and mark the neighbor as malicious and close the connection"), so it is not a deviation, but it means the mix network has no way to carry two message versions at once.

The blend network already has a mechanism for exactly this shape of problem, the Transition Period, which keeps old-epoch messages valid for 30 rounds after an epoch change (`blend-protocol.md` "Transition Period"). Versions have no equivalent: the moment one core node emits version 2, every version-1 neighbour drops the connection, the sender's peering degree collapses below $`\Phi_{CC}^{Min}`$, and the node reconnects to other version-1 nodes that do the same. Edge nodes are affected in the opposite direction: an edge node on the newer release sends version 2 to a version-1 core node, which ignores it at trace level (`with_edge/behaviour/mod.rs:205`), so the edge node's proposal or transaction is silently lost. Because the last blend hop broadcasts the decapsulated proposal bytes unparsed (`services/blend/src/core/dispatcher/libp2p.rs:276-282`), the payload version is not the problem; the envelope version is.

**Exploit scenario**

Not attacker-triggered. During a blend release rollout, the fraction of core nodes on each version forms its own mix network with a proportionally smaller anonymity set, and leaders whose proposals enter the wrong side never reach a majority of validators.

**Recommendation**
- *Short term*: accept a set of versions in `PublicHeader::decode` (a `Context` carrying the accepted set, as the private header already does for `num_blend_layers`), and separate "unknown but greater than the latest accepted" from "malformed" so that the former is discarded without `SpamReason`.
- *Long term*: specify a version transition in `blend-protocol.md`, keyed to an epoch boundary like the Transition Period: accept `{N, N+1}` for one epoch plus TP, emit `N+1` from the next epoch, drop `N` one epoch later. Tie the emitted version to the protocol negotiated on the connection (LB-002), so a node never emits a version its neighbour did not advertise.

**References**: `blend-protocol.md` "Relaying" 1.1 (Overview tier) and 1.3 (Protocol tier), "Transition Period", "Edge Network"; `message-formatting.md` "Public Header".

### LB-005 · Spec deviation: libp2p protocol identifiers on the wire differ from the strings the specifications fix

| | |
|---|---|
| Severity | Informational |
| Difficulty | Low |
| Category | Configuration |
| Target | `tools/config/src/deployment.rs:L45-L48` (`*_PROTOCOL_SUFFIX` constants); `deployment/ceremony/genesis/testnet/deployment-template.yaml:L5,L17-L19,L85,L104`; `nodes/node/binary/src/config/deployment/settings.yaml:L5,L17-L19,L85,L122` |
| Status | Open |

**Description**

`cryptarchia-v1-bootstr-sync.md` "Downloading Blocks" fixes the sync protocol id to `/logos-blockchain/cryptarchia/sync/1.0.0` (mainnet) and `/logos-blockchain-testnet/cryptarchia/sync/1.0.0` (testnet). `blend-protocol.md` "Connection Details" fixes `/logos-blockchain/blend/1.0.0` and `/logos-blockchain-testnet/blend/1.0.0`. The node produces `/logos-blockchain-<env>-<version>/chainsync/1.0.0` and `/logos-blockchain-<env>-<version>/blend/1.0.0` (`deployment-template.yaml:5,19`; `deployment.rs:47` uses `chainsync/1.0.0` and `:48` a gossip topic `cryptarchia/proto/1.0.0` that the templates spell `cryptarchia/1.0.0`). The name segment (`chainsync` vs `cryptarchia/sync`) and the namespace (`-<version>` suffix) both differ, and the committed `settings.yaml` still carries the literal placeholder `X.Y.Z` (`settings.yaml:5,17-19,85,122`), so a binary built from the repository without running the ceremony talks on `/logos-blockchain/blend/X.Y.Z`.

The code is the side to keep for the namespace: separating networks by identity is right, and the spec's flat `/logos-blockchain/` prefix would let a testnet and a mainnet node negotiate. The spec is the side to keep for the name segment and for the meaning of the trailing version, which the code never uses. Either way, an alternative implementation written from the spec today does not interoperate with this node.

**Exploit scenario**

None. Impact is interoperability and reviewability: a second implementation, or a monitoring tool written against the spec, uses the wrong protocol ids.

**Recommendation**
- *Short term*: align `deployment.rs:45-48` and the ceremony templates on one set of strings, and update the two spec sections to the same strings with the namespace rule spelled out.
- *Long term*: derive every protocol id and topic from one function in `tools/config` that the spec quotes verbatim, and add a test that compares the ceremony output against it.

**References**: `cryptarchia-v1-bootstr-sync.md` "Downloading Blocks"; `blend-protocol.md` "Connection Details".

### LB-006 · The blend public-header version byte is not covered by the header signature

| | |
|---|---|
| Severity | Informational |
| Difficulty | Medium |
| Category | Cryptography |
| Target | `blend/message/src/encap/encapsulated.rs:L372-L380` (`signing_body`), `:L71-L77,L294-L297` (`verify_header_signature`, signing); `blend/message/src/message/public_header.rs:L55-L69` (`verify_signature`) |
| Status | Open |

**Description**

The public-header signature is computed over the private header and the payload only:

```rust
// blend/message/src/encap/encapsulated.rs:372-380
fn signing_body(private_header: &EncapsulatedPrivateHeader, payload: &EncapsulatedPayload) -> Vec<u8> {
    private_header.iter_bytes().chain(payload.iter_bytes()).collect::<Vec<_>>()
}
```

This matches `message-formatting.md` ("a signature of the concatenation of the i-th encapsulation of the payload and the private header"), so it is not a deviation. The version byte, the signing key and the proof of quota are outside the signed body; the key is bound by being the verification key and the proof of quota is bound to the key, but the version byte is bound by nothing. A relay can rewrite it. Today that only lets a relay make the next hop classify the relay itself as spammy (LB-004), which it could achieve by sending garbage anyway. Once two versions coexist (LB-004's recommendation), an unsigned version byte is a downgrade vector: a relay rewrites 2 to 1 and the next hop parses the header under the old rules.

**Exploit scenario**

After a version transition that changes the header layout, a malicious core node rewrites the version byte of a forwarded message to the older value; the next hop parses the header under the old layout and, if the layouts are prefix-compatible, accepts a header the sender never signed. If the layouts are not prefix-compatible the message is merely dropped, which a relay can do regardless.

**Recommendation**
- *Short term*: none required for one version.
- *Long term*: include the version byte in `signing_body` (a one-byte domain separator at the front) when the format is next revised, and say so in `message-formatting.md` "Public Header".

**References**: `message-formatting.md` "Public Header"; `message-encapsulation.md` "Message Structure".

### LB-007 · Sync message enum tags are positional bincode discriminants with no stated compatibility rule

| | |
|---|---|
| Severity | Informational |
| Difficulty | Low |
| Category | Data Validation |
| Target | `consensus/cryptarchia-sync/src/libp2p/messages.rs:L14-L20` (`RequestMessage`), `:L131-L139` (`DownloadBlocksResponse`); `consensus/cryptarchia-sync/src/messages.rs:L14-L24` (`GetTipResponse`); `consensus/cryptarchia-sync/src/lib.rs:L12-L18` (`BlocksUnavailableReason`); `core/src/codec/bincode/mod.rs:L23-L29` (`OPTIONS`) |
| Status | Open |

**Description**

With `with_fixint_encoding` and little endian (`bincode/mod.rs:23-29`), each enum is encoded as a `u32` index in declaration order followed by the variant's fields, and structs are field sequences with no names. Today's tags are: `RequestMessage` 0 = `DownloadBlocksRequest`, 1 = `GetTip`; `DownloadBlocksResponse` 0 = `Block`, 1 = `NoMoreBlocks`, 2 = `Failure`; `GetTipResponse` 0 = `Tip`, 1 = `Failure`; `BlocksUnavailableReason` 0 = `BlockNotFound`, 1 = `StartBlockNotFound`, 2 = `Unknown`. An unknown tag is a bincode error, which `unpack_from_reader` returns as `PackingError::Serialization` (`packing.rs:75`); the downloader ends the stream (`downloader.rs:108-114`) and the provider drops the request (`provider.rs:31-33`). Nothing records that the peer speaks a different version, and nothing distinguishes that from a corrupt frame.

The rules that keep two versions of these types compatible are not written down anywhere, and `#[derive(Serialize, Deserialize)]` makes them easy to break by accident: a variant inserted in the middle, a removed variant, a reordered or added struct field, or a change of a field's type each shift every later tag or offset. `network-wire-format.md` says only that bincode is used.

**Exploit scenario**

None. A peer at another version produces a decode error and the stream closes, which is the safe outcome; the cost is that the same failure is indistinguishable from a malformed frame and cannot be turned into an "upgrade" signal.

**Recommendation**
- *Short term*: document the rules next to the types: append new variants at the end only; never remove or reorder a variant or a struct field, deprecate instead; never change a field's type; no `Option`, `HashMap` or other layout-sensitive type in a message that must stay compatible; and version the protocol name (LB-002) when any of these must be broken. Add a fixture test that pins the encoded bytes of every variant.
- *Long term*: replace the bare enums with a `#[non_exhaustive]`-style envelope carrying an explicit `u8` tag, decoded by hand as `PublicHeader` is, so that an unknown tag is a typed error the behaviour can act on.

**References**: `network-wire-format.md` "Encoding and Decoding"; `cryptarchia-v1-bootstr-sync.md` "Downloading Blocks".

## 5. Suggestions (non-security)

### S-001 · The specifications state that versions will be height-activated but also fix every version to 1, and say nothing about coexistence

| | |
|---|---|
| Target | `cryptarchia-v1-protocol.md` "Versioning and Protocol Upgrades", "Block Header Validation" rule 1; `bedrock-v1.1-block-construction.md` "Header" ("fixed to `0x01`"), "Canonical Encoding" (`Version = Byte ; fixed to 0x01`); `blend-protocol.md` "Relaying" 1.1, "Connection Details"; `message-formatting.md` "Public Header"; `p2p-network-bootstrapping.md` step 2 |

`cryptarchia-v1-protocol.md` "Versioning and Protocol Upgrades" says "We will use block height to schedule the activation of protocol updates. E.g. bedrock version 35 will be active after block height 32000", but rule 1 of "Block Header Validation" is `bedrock_version = 1` and `bedrock-v1.1-block-construction.md` fixes the byte to `0x01` in two places. An implementer cannot tell whether a header with version 2 is invalid forever, invalid until a height, or to be relayed. The recommendation column of the inventory in §2 lists what each section needs: an activation table `(version, height)` as a protocol parameter; a validation rule `version == active_version(height)`; a statement that an unknown greater version is an upgrade signal and not a verdict on the block; the protocol-id naming rule (namespace = chain identity, trailing segment = protocol version, highest common version wins); and a blend version transition keyed to an epoch boundary. `p2p-network-bootstrapping.md` step 2 says nodes "verify network identity and protocol compatibility" without defining either; the protocol-id rule is where that definition belongs.

### S-002 · `message-encapsulation.md` and `payload-formatting.md` disagree on the payload body size and the payload types

| | |
|---|---|
| Target | `message-encapsulation.md` "Message Structure" (`PAYLOAD_BODY_SIZE = 34 * 1024`, `PayloadType { COVER, DATA }`); `payload-formatting.md` "Type" (three types: cover, block proposal, transaction), "Body" (`Max_Body_Length = 18192`) |

The code follows `payload-formatting.md` (`blend/message/src/message/payload.rs:22-26` has three types). The encapsulation document should reference the formatting document instead of restating the values, per the one-derivation rule.

### S-003 · The λSQL inscription payload has no specification

| | |
|---|---|
| Target | `logos_sql/src/protocol/mod.rs:L17-L20,L257-L290` (`PAYLOAD_MARKER`, `PAYLOAD_VERSION`, `encode`, `decode`) |

Inscriptions are permanent chain history (`bedrock-architecture-overview.md` "Mantle Channels"), but the marker, the version field, the body encoding and the rule for reading old versions exist only in code, and the version has already changed once (LB-001). A short standards-track document under `docs/blockchain/raw/` with the ABNF of the payload and the reader rule would give the next bump a reference.

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
