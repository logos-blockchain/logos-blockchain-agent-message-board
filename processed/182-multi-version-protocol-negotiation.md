# Audit Report — Multi-version protocol negotiation: libp2p capability check, affected config, and a naming rule

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/182`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `3d5d419ec85c7b23e4d7e1455bd1cde845264a9e` — component(s): `libp2p`, `consensus/cryptarchia-sync`, `blend/network`, `services/network`, `services/blend`, `nodes/node/binary/src/config`, `tools/config`, `deployment/ceremony`, `.github/workflows/genesis-ceremony.yml`
Specs: `https://github.com/logos-co/logos-lips` @ `75d3d0382604d4a0d8e246c268935dd386ffc8ea` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `network-wire-format.md` (in full); `cryptarchia-v1-bootstr-sync.md` §Downloading Blocks, `blend-protocol.md` §Connection Details, `p2p-network-bootstrapping.md` §Protocol step 2 and §Details, `bedrock-genesis-block.md` §Cryptarchia Parameters (by section)
Date: `2026-09-15` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: the rollout the #70 report asked for (several protocol versions negotiated per behaviour, chain identity in the namespace, version in the trailing segment) is possible with the pinned libp2p crates for chain-sync, Blend and both gossip topics, but not for Kademlia: `libp2p-kad` 0.48.0 takes exactly one protocol name and the setter that once accepted several was removed, so the DHT identifier must be version-stable. The `identify_protocol_name` field is not a protocol identifier at all: it is passed as identify's advertised `protocol_version` string, identify itself always runs `/ipfs/id/1.0.0`, and the node never reads the string from peers, so today it gates nothing. Six deployment fields carry protocol names or topics; three become lists, one stays single, one is renamed, one is derived. The committed embedded deployment still carries the literal `X.Y.Z` placeholder in all six. A naming rule is proposed with the two spec edits it needs.
- Findings: `0` critical · `0` high · `0` medium · `0` low · `2` informational
- Key themes: "the version is in the wrong segment", "one name per behaviour", "an identifier that identifies nothing"
- Must-fix before launch: none; the rule should be settled before the first release that changes a wire format, because after it the two networks cannot see each other (#377).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `libp2p-stream` 0.4.0-alpha, `libp2p-kad` 0.48.0, `libp2p-identify` 0.47.0, `libp2p-swarm` 0.47.1, `multistream-select` 0.13.0 (registry sources at the `Cargo.lock` pins) | checklist item 1: what each can negotiate |
| `consensus/cryptarchia-sync/src/libp2p/behaviour.rs`, `blend/network/src/core/with_core/behaviour/handler/mod.rs`, `blend/network/src/core/with_edge/behaviour/handler/mod.rs`, `services/blend/src/edge/backends/libp2p/swarm.rs`, `libp2p/src/config/kademlia.rs`, `libp2p/src/config/identify.rs`, `libp2p/src/behaviour/gossipsub/swarm_ext.rs`, `services/network/src/backends/libp2p/swarm/identify.rs` | where each name is registered and used |
| `nodes/node/binary/src/config/{network,blend,cryptarchia,mempool}/deployment.rs` and the `mod.rs` that consume them, `tools/config/src/{release,deployment}.rs`, `deployment/ceremony/genesis/*/{deployment-template,inscribe}.yaml`, `nodes/node/standalone-deployment-config.yaml`, `nodes/node/binary/src/config/deployment/settings.yaml`, `.github/workflows/genesis-ceremony.yml` | checklist items 2 to 4 |
| `nodes/version/src/lib.rs`, `nodes/node/binary/src/api/{handlers,routes}.rs`, `services/network/src/backends/libp2p/command.rs` (`Libp2pInfo`) | checklist item 5 |
| `core/src/mantle/transactions/genesis_tx.rs:241-320` (`ChainId`) | what a chain identity can contain |

**Out of scope**

The wire formats themselves and their versioning (#70 LB-001, LB-003, LB-007); the Blend message version byte (#70 LB-006); gossipsub validation and scoring (#144); the wrong-network marker in storage (#80). Third-party crates assumed correct beyond the API facts quoted from their sources.

**Assumptions**

- The specification at the commit above is correct unless stated otherwise; the spec strings are the ones in `cryptarchia-v1-bootstr-sync.md:210-211` and `blend-protocol.md:539`.
- The node stays on the pinned libp2p versions; where a later upstream release changes an API this report says so.
- Release build facts of issue #19 re-verified at this commit; none bears on this issue.

## 3. Method

- Manual review of the in-scope paths, working through issue `#182` under parent `#10`, all five checklist items. The in-scope node files are unchanged between `a805329f8` (the #70 report) and the target commit.
- Spec conformance against `cryptarchia-v1-bootstr-sync.md` §Downloading Blocks (protocol id), `blend-protocol.md` §Connection Details, `p2p-network-bootstrapping.md` §Protocol step 2 ("verify network identity and protocol compatibility") and §Details, `bedrock-genesis-block.md` §Cryptarchia Parameters (`chain_id`).
- Automated tooling: none. Every capability claim is a quotation of the pinned crate's source, with line numbers.
- Dynamic testing: none.

**Checked and ruled out**

- multistream-select 0.13.0 picks the dialer's first supported protocol. `dialer_select_proto` takes an iterator, proposes the first item with the header (`src/dialer_select.rs:114-117`), and on `NotAvailable` proposes the next (`:186-193`); the listener answers each proposal in turn. The dialer's list order is therefore the preference order, and a dialer that lists newest first gets the newest version both ends support. Confirmed as the issue asked.
- `libp2p-stream::Control::accept` can be called once per distinct protocol on one behaviour: `Shared::accept` keys a `HashMap<StreamProtocol, Sender>` and returns `AlreadyRegistered` only for a name already present (`src/shared.rs:52-68`); the inbound upgrade advertises every registered name (`src/handler.rs:54-61`, `src/upgrade.rs:9-20`). The outbound side is one name per stream: `Control::open_stream(peer, protocol)` (`src/control.rs:44-64`) produces an outbound `Upgrade` with `supported_protocols: vec![protocol]` (`src/handler.rs:70-80`), so a dialer cannot hand multistream-select a version list; it gets `OpenStreamError::UnsupportedProtocol` (`src/control.rs:81-86`) and must retry with the next version itself, or pick the version up front from what identify reported the peer supports (`info.protocols`, already consulted for Kademlia at `services/network/src/backends/libp2p/swarm/identify.rs:25-33`). Confirmed with that caveat.
- `libp2p-kad` 0.48.0 cannot carry more than one protocol name. `Config::new(protocol_name: StreamProtocol)` builds `ProtocolConfig { protocol_names: vec![protocol_name], .. }` (`src/behaviour.rs:223-227`, `src/protocol.rs:156-162`); the field is private; the only public accessors are `protocol_names()` (`:165-167`, `src/behaviour.rs:471-473`) and, on `Config`, setters for timeouts, replication, TTLs, packet size, k-bucket parameters and mode (`src/behaviour.rs:247-435`, `:1118`), none for names. The changelog records `set_protocol_names` being added in 0.40.0 (`CHANGELOG.md:277`), `set_protocol_name` removed in 0.41.0 (`:263-264`), and `Config::new(StreamProtocol)` made mandatory in 0.46.0 (`:55-57`); at 0.48.0 the list setter is gone. The issue's item 1 assumption is wrong at the pinned version: LB-001.
- Kademlia does advertise all of its names to multistream-select (`ProtocolConfig::protocol_info` returns the vector, `src/protocol.rs:189-191`), so if a future `libp2p-kad` restores a list API nothing else in the node needs to change; the node already reads the list back through `protocol_names()` (`libp2p/src/behaviour/kademlia/behaviour_ext.rs:77-79`).
- `libp2p-identify` 0.47.0 negotiates fixed protocol ids, `/ipfs/id/1.0.0` and `/ipfs/id/push/1.0.0` (`src/protocol.rs:35-37`). `Config::new(protocol_version, public_key)` (`src/behaviour.rs:166-168`) takes the string the node will *advertise* in its `Info.protocol_version` (`src/protocol.rs:46`, `:65-66`); it is never negotiated. The node passes `identify_protocol_name` there (`libp2p/src/config/identify.rs:44`) and, on `identify::Event::Received`, reads only `info.protocols` and `info.listen_addrs` (`services/network/src/backends/libp2p/swarm/identify.rs:16-52`); nothing reads `info.protocol_version` or `info.agent_version`. LB-002.
- Gossipsub subscribes per `IdentTopic` and a swarm may hold any number of subscriptions (`libp2p/src/behaviour/gossipsub/swarm_ext.rs:11-16`); publishing takes one topic (`:18-30`). Two topics per transition window is a config and a loop, no library limit.
- The Blend core handler advertises one `ReadyUpgrade<StreamProtocol>` inbound and outbound (`with_core/behaviour/handler/mod.rs:160-167`); `ReadyUpgrade` yields a single `Info` (`libp2p-swarm` 0.47.1 `src/upgrade.rs`, `ReadyUpgrade<P>::protocol_info` is `iter::once`). The edge handler does the same (`with_edge/behaviour/handler/mod.rs:143-149`), and the edge dialer opens its stream through `libp2p-stream` with one name (`services/blend/src/edge/backends/libp2p/swarm.rs:429-440`). A list needs a small custom upgrade of the kind `libp2p-stream` already ships (`src/upgrade.rs:9-45`, `UpgradeInfo` over a `Vec<StreamProtocol>`).
- Where the six config fields go: `network.{kademlia,identify,chain_sync}_protocol_name` (`config/network/deployment.rs:6-8`) into the swarm config (`config/network/mod.rs:28-30`) and from there into `kad::Config::new` (`libp2p/src/config/kademlia.rs:72`), `identify::Config::new` (`identify.rs:44`) and `cryptarchia_sync::Behaviour::new` (`behaviour.rs:160-165`); `blend.common.protocol_name` (`config/blend/deployment.rs:93`) into the core and edge Blend backends (`config/blend/mod.rs:106`, `:143`); `cryptarchia.gossipsub_protocol` (`config/cryptarchia/deployment.rs:28`) as the proposal topic for chain-network (`config/cryptarchia/mod.rs:133`) and for the Blend broadcast fallback (`config/blend/mod.rs:73`); `mempool.pubsub_topic` (`config/mempool/deployment.rs:11`) as the transaction topic (`config/mempool/mod.rs:33`). The integration config derives all but Blend's from `ProtocolIdentity` (`tools/config/src/deployment.rs:96`, `:132-137`) with the suffixes at `:45-48`, and Blend's from a literal `/blend/integration-tests` (`:36`, `:105`).
- The committed embedded deployment still carries `X.Y.Z` in all six strings (`config/deployment/settings.yaml:5,17,18,19,85,122`), and `ProtocolIdentity::from_env` only affects the integration tooling (`release.rs:13-41`), not the binary. A node built from the repository without the ceremony therefore listens on `/logos-blockchain/kad/X.Y.Z`, `/logos-blockchain/chainsync/X.Y.Z` and `/logos-blockchain/blend/X.Y.Z`, subscribes to `/logos-blockchain/cryptarchia/X.Y.Z` and `/logos-blockchain/mempool/X.Y.Z`, and advertises `/logos-blockchain/identify/X.Y.Z` as its identify `protocol_version`. That is #380's fourth paragraph re-verified at this commit; checklist item 4 is answered there and not filed again.
- The chain identity available today. `DeploymentSettings::chain_id()` reads the `ChainId` out of the genesis inscription (`config/deployment/mod.rs:28-32`, `config/cryptarchia/deployment.rs:36-43`); a `ChainId` is 1 to 255 bytes of UTF-8 with no alphabet restriction (`genesis_tx.rs:241-250`, #106), and the ceremony templates put a release string in it: `"X.Y.Z"` (testnet), `"X.Y.Z-rc.N"` (devnet), `"standalone/X.Y.Z"` (standalone, with a slash) (`deployment/ceremony/genesis/*/inscribe.yaml`). It can therefore contain `/`, spaces and non-ASCII, none of which a multistream path segment may hold, and on the current templates it changes on every release exactly like the namespace it would replace. The genesis block id (Blake2b-256 of the genesis header, `cryptarchia-v1-protocol.md` §Block ID) is the identity that is derived rather than typed, so S-001 uses it.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | `libp2p-kad` 0.48.0 accepts exactly one protocol name, so the DHT cannot bridge two protocol versions and its identifier must not carry one | Configuration | Informational | Low | Open |
| LB-002 | `identify_protocol_name` is not a libp2p protocol identifier: it is identify's advertised `protocol_version` string, which the node never reads from peers | Configuration | Informational | Low | Open |

### LB-001 · `libp2p-kad` 0.48.0 accepts exactly one protocol name, so the DHT cannot bridge two protocol versions and its identifier must not carry one

| | |
|---|---|
| Severity | Informational |
| Difficulty | Low |
| Category | Configuration |
| Target | `libp2p/src/config/kademlia.rs:71-72`; `libp2p-kad` 0.48.0 `src/behaviour.rs:223-227`, `:247-435`, `src/protocol.rs:146-167`, `CHANGELOG.md:55-57`, `:263-264`, `:277` |
| Status | Open |

**Description**

The plan in #70 LB-002 (#377) and in this issue's item 1 is to register every supported version of every protocol so that multistream-select picks the newest common one. That works for the stream-based protocols and for gossip (§3), but the Kademlia behaviour at the pinned version takes one `StreamProtocol` in `Config::new` and has no way to add another: `ProtocolConfig.protocol_names` is private, and the `set_protocol_names` that 0.40.0 added and 0.41.0 made the only setter is absent from 0.48.0's `Config`. The node builds it with exactly the one name from `network.kademlia_protocol_name` (`kademlia.rs:72`). Peers only enter each other's routing tables when identify reports a Kademlia protocol they share (`identify.rs:25-33`), so if the DHT name ever carries a version, the two halves of a rollout cannot discover each other at all, whatever the other protocols do.

The DHT's own wire format is libp2p's, not this project's, so there is no reason for its identifier to change when a Logos wire format does. The consequence for the naming rule (S-001) is that the Kademlia identifier is scoped by chain only, with a fixed trailing segment, and is the one identifier the ceremony never bumps.

**Exploit scenario**

None; a design constraint. Its cost if ignored is the flag day of #377 in its worst form: no discovery between the two sides.

**Recommendation**

- *Short term*: fix the Kademlia identifier as `/logos-blockchain/<chain>/kad/1.0.0` and document that the trailing segment is not a Logos protocol version. Keep `network.kademlia_protocol_name` a single value.
- *Long term*: if a future `libp2p-kad` restores a list API, the node already reads `protocol_names()` as a list; nothing else changes.

**References**: `p2p-network-bootstrapping.md` §Protocol step 2, §Details ("Currently, the Logos Blockchain network uses `kademlia`"); #377.

### LB-002 · `identify_protocol_name` is not a libp2p protocol identifier: it is identify's advertised `protocol_version` string, which the node never reads from peers

| | |
|---|---|
| Severity | Informational |
| Difficulty | Low |
| Category | Configuration |
| Target | `libp2p/src/config/identify.rs:37-50`; `services/network/src/backends/libp2p/swarm/identify.rs:16-58`; `nodes/node/binary/src/config/network/deployment.rs:7`; `libp2p-identify` 0.47.0 `src/behaviour.rs:161-168`, `src/protocol.rs:35-37`, `:46-49`, `:65-69` |
| Status | Open |

**Description**

`identify::Config::new(protocol_version, public_key)` takes the string the local node advertises as `protocol_version` in its identify `Info`; the identify protocol itself is always negotiated as `/ipfs/id/1.0.0`. The node passes its `identify_protocol_name` deployment field there (`identify.rs:44`), so the value `/logos-blockchain-<env>-<version>/identify/1.0.0` never appears in any multistream negotiation. It is sent to every peer and received from every peer, and the receiving side discards it: `handle_identify_event` uses `info.protocols` (to find a shared Kademlia name) and `info.listen_addrs`, nothing else. Two nodes with different `identify_protocol_name` values peer exactly as if they had the same one, so the field, despite its name and its place next to the two real protocol identifiers, does not separate networks or versions.

This is also the cheapest place to do what `p2p-network-bootstrapping.md` §Protocol step 2 asks, "verify network identity and protocol compatibility": identify runs once per connection before anything else, `Info.protocol_version` is one string compare away, and `Info.agent_version` is the field libp2p reserves for the software version (currently `None`, `identify.rs:48-50`, so the libp2p default is advertised).

**Exploit scenario**

None. Impact is a misleading configuration surface: an operator or a second implementation may believe the field selects a network.

**Recommendation**

- *Short term*: rename the field to `identify_protocol_version`, set it to the chain identity of S-001 rather than a per-release string, and on `identify::Event::Received` log at `warn` and disconnect when a peer's `protocol_version` differs (the wrong-network peer check parent #26 asks for, and a cheaper gate than #80's storage marker).
- *Long term*: S-003.

**References**: `p2p-network-bootstrapping.md` §Protocol step 2; parent #26 "wrong-network peers are disconnected"; #80.

## 5. Suggestions (non-security)

### S-001 · The naming rule: chain identity in the namespace, Logos protocol version in the trailing segment, and the two spec edits

| | |
|---|---|
| Target | `tools/config/src/deployment.rs:45-48`, `src/release.rs:7-54`; `deployment/ceremony/genesis/*/deployment-template.yaml:5,17-19,85,104`; `.github/workflows/genesis-ceremony.yml:14-16,35-38`; `cryptarchia-v1-bootstr-sync.md:210-211`; `blend-protocol.md:539` |

Checklist item 3. Rule:

```
/logos-blockchain/<chain>/<protocol>/<version>
```

- `<chain>` is the first 16 hex characters of the genesis block id (`block_id(GENESIS_HEADER)`, `cryptarchia-v1-protocol.md` §Block ID). It is derived from the genesis block every node already holds, so no ceremony placeholder, no environment variable and no hand-typed release string can drift from it; it is a valid single multistream segment, which the `ChainId` string is not (`standalone/X.Y.Z` carries a slash, and the type allows any UTF-8); and two networks with different genesis blocks get disjoint identifiers by construction, which is the property the spec's `-testnet` suffix is for. `DeploymentSettings` can compute it from `cryptarchia.genesis_block` at load time, so the six strings stop being configuration at all.
- `<protocol>` is the spec's name for the protocol: `cryptarchia/sync`, `blend`, `kad`, `cryptarchia/proposals` and `mempool` for the two topics. The code's `chainsync` and `cryptarchia/proto` go (#380).
- `<version>` is the Logos wire-format version of that protocol, `MAJOR.MINOR.PATCH`, bumped in the release that changes the format, and only there. A node registers every version it can speak, newest first, for `cryptarchia/sync` (one `Control::accept` per version, `open_stream` newest first with fallback on `UnsupportedProtocol`, or the version chosen from the peer's identify `protocols`), for `blend` (a `Vec<StreamProtocol>` upgrade in place of `ReadyUpgrade`), and for both topics (subscribe to every supported version, publish on the newest, drop the old subscription a release later). `kad` stays at `1.0.0` for the reason in LB-001.

The two spec edits. `cryptarchia-v1-bootstr-sync.md` §Downloading Blocks replaces its mainnet and testnet literals with the rule above, states that a node offers every version it supports newest first, and says what a version bump means (any change to the request or response encoding of §Downloading Blocks). `blend-protocol.md` §Connection Details does the same for `/blend/<version>` and notes that during the Transition Period of §Transition Period a node keeps accepting the previous version. A third, smaller edit: `p2p-network-bootstrapping.md` §Protocol step 2 can name the mechanism it means by "verify network identity" (LB-002's identify check).

Why the version does not belong in the namespace, restated from #377 with the numbers from this pass: multistream-select matches whole strings, so a version inside the namespace makes every protocol of the node incompatible at once, Kademlia included; a version in the trailing segment lets each protocol move independently and lets the one protocol that cannot move (LB-001) stay put.

### S-002 · The config fields that change

| | |
|---|---|
| Target | `nodes/node/binary/src/config/network/deployment.rs:5-9`, `blend/deployment.rs:88-95`, `cryptarchia/deployment.rs:28`, `mempool/deployment.rs:10-17`; their consumers in `config/network/mod.rs:28-30`, `config/blend/mod.rs:73,106,143`, `config/cryptarchia/mod.rs:133`, `config/mempool/mod.rs:33`; `tools/config/src/deployment.rs:36,45-48,96-137`; `libp2p/src/protocol_name.rs` |

Checklist item 2. Today, six fields, all single strings, all set by the ceremony from one `<env>-<version>` substitution:

| Field | Today | Under S-001 |
|---|---|---|
| `network.chain_sync_protocol_name` | one `StreamProtocol` | `Vec<StreamProtocol>`, newest first; `cryptarchia_sync::Behaviour::new` takes the list, one `accept` each |
| `blend.common.protocol_name` | one `StreamProtocol` | `Vec<StreamProtocol>`; core and edge handlers take the list |
| `cryptarchia.gossipsub_protocol` | one topic `String` | `Vec<String>`: subscribe all, publish first; used by chain-network and the Blend broadcast fallback |
| `mempool.pubsub_topic` | one topic `String` | `Vec<String>`, same |
| `network.kademlia_protocol_name` | one `StreamProtocol` | unchanged type; value fixed by the rule (LB-001) |
| `network.identify_protocol_name` | one `StreamProtocol` | renamed `identify_protocol_version: String`, value is `<chain>` (LB-002) |

Or, better, none of them: derive all six from `cryptarchia.genesis_block` plus a per-protocol version list that lives in the binary, since the version list is a property of the code, not of the deployment. The `StreamProtocol` serde wrapper (`protocol_name.rs:6-34`) already rejects malformed names at load time and works unchanged inside a `Vec`. `tools/config` replaces `ProtocolIdentity` and the four suffix constants with the same derivation, and the Blend literal `/blend/integration-tests` (`deployment.rs:36`) with it too, so the integration tests exercise the production rule.

### S-003 · Populate identify `agent_version` from `lb-version` and expose peer versions in the libp2p info endpoint

| | |
|---|---|
| Target | `libp2p/src/config/identify.rs:48-50`; `nodes/version/src/lib.rs:19-35` (`BuildVersionInfo`), `nodes/node/binary/src/api/handlers.rs:492`; `services/network/src/backends/libp2p/command.rs:39-54` (`Libp2pInfo`); `services/network/src/backends/libp2p/swarm/identify.rs:16-58` |

Checklist item 5. `BuildVersionInfo` already exists and is served on the node's `/version` route. Set identify's `agent_version` to `logos-blockchain/<version>[+<commit>]` from it (`with_agent_version`, `identify.rs:48-50`, replacing the `None` default), keep the received `agent_version` and `protocol_version` per connected peer in the swarm handler when `identify::Event::Received` arrives, and add them to `Libp2pInfo` (`command.rs:39-54`) so the libp2p info route shows, per peer, the software version and the chain identity it advertises. With S-001 the same route can show which protocol versions each peer offers (`info.protocols`), which is the signal an operator needs to know when the old version can be dropped.

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
