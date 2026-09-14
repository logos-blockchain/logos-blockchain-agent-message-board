# Audit Report — Logging and tracing: leaks, injection, and log-volume DoS

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/37`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `tracing`, `services/tracing`, `nodes/node/binary/src/api`, and every `tracing::*!` call site in `blend/*`, `services/blend`, `services/chain/*`, `consensus/cryptarchia-sync`, `services/network`, `libp2p`, `services/sdp`, `services/pow`, `services/wallet`, `wallet`, `kms/*`, `services/key-management-system`, `c-bindings`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`
Date: `2026-09-12` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: no secret key, seed, note or witness is formatted into any log call, and the default filter policy is sound; the weaknesses are (a) ERROR-level lines any unauthenticated peer can emit at will into a lossy, size-unbounded log pipeline, (b) TRACE-level lines that bind the node to its Blend messages and to its winning leader UTXO, live in release builds and shipped verbatim by the plaintext exporters, (c) an unauthenticated HTTP endpoint that rewrites the log filters, and (d) an on-chain locator string that reaches every core node's log unescaped.
- Findings: 0 critical · 0 high · 0 medium · 5 low · 0 informational
- Key themes: per-peer ERROR spam with silent drop of genuine lines; identity-bearing TRACE lines against the spec's proposer-confidentiality goal; remotely rewritable filters; log injection through the SDP locator's DNS component.
- Must-fix before launch: none; LB-001 and LB-003 should be closed with #119 (API authentication), LB-002 before any operator ships TRACE logs to a collector.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `tracing/` (`lb-tracing`) | filter policy, file/stdout/Loki/GELF/OTLP layers, compressed appender |
| `services/tracing/` | subscriber assembly, global level cap, filter reload |
| `nodes/node/binary/src/api/{tracing,backend}.rs`, `nodes/api-common/src/paths.rs` | the `/admin/tracing/filter` route and HTTP request tracing |
| `nodes/node/binary/src/config/tracing/serde/*` | shipped logging defaults |
| every `trace!/debug!/info!/warn!/error!` site in the crates listed in the header (grep-complete, about 860 sites outside tests; the 71 sites that format a value with `{:?}` or `?field` were read individually) | what is logged, at which level, and whether a remote party controls the trigger or the content |
| `c-bindings/src/logging.rs` | the FFI stderr logger |
| pinned crates `tracing-subscriber 0.3.23`, `tracing-appender 0.2.5`, `tracing-loki 0.2.7`, `tracing-gelf 0.7.1`, `multiaddr 0.18.2`, `libp2p-swarm 0.47.1` | formatting, buffering, transport and `Display` behaviour cited below |

**Out of scope**

`zone-sdk`, `logos_sql`, `tools/`, `tests/`, `deployment/` (grepped for secrets, not read); the OTLP TLS question (already #87 / PR 84 LB-003); `Debug` derives on secret types (PR 128, #129); the content of HTTP error bodies (PR 202 LB-007, PR 118 LB-004). `tracing`, `tracing-subscriber`, `opentelemetry-*` and `libp2p` are assumed correct except for the specific formatting and transport facts quoted.

**Assumptions**

The operator runs the shipped defaults: `level: INFO`, file sink rolling hourly with 10 files retained, stdout on, no exporter (`nodes/node/binary/src/config/tracing/serde/logger.rs:20-42`, `mod.rs:22-35`). The host is not compromised. "Remote" means a libp2p peer with no stake, or an HTTP client that can reach the API port (bound to `127.0.0.1:8080` by default, see PR 118).

## 3. Method

- Manual review of the in-scope paths, working through issue `#37` (sole checklist item: `{:?}` on structs with secrets/notes/peers; log injection via peer-supplied strings; per-peer log volume as disk-fill DoS; exporter endpoints and encryption) under parent `#24`, with `#19` for the repo facts (no compile-time `max_level_*` feature is set anywhere in the workspace, so every `trace!` site is live in release builds).
- Spec conformance: `bedrock-architecture-overview.md` ("breaking the link between a proposal and its proposer … extends block proposer confidentiality to the network layer") and `overview-cryptoeconomics.md` ("we must not link leaders to their blocks and rewards") are the only statements that bear on logging; both are used to rate LB-002.
- Automated tooling: `rg` over the workspace for every logging macro, for `{:?}` / `?field` arguments, for `println!/eprintln!`, and for `config`/`settings` Debug-formatting; `grep` over the pinned crate sources in the cargo registry. No cargo build.
- Dynamic testing: none. Every trigger described below is established from the code path, not reproduced.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Unauthenticated peers emit ERROR lines at will into a lossy, size-unbounded log pipeline | Denial of Service | Low | Low | Open |
| LB-002 | TRACE lines bind the node to its Blend messages and to its winning leader UTXO | Privacy / Anonymity | Low | Medium | Open |
| LB-003 | `PUT /admin/tracing/filter` lets any API client silence or reshape logging | Access Controls | Low | Low | Open |
| LB-004 | Exporters ship every event and span field, including identity fields, in plaintext | Data Exposure | Low | Medium | Open |
| LB-005 | An SDP locator's DNS label is written unescaped into every core node's log | Auditing and Logging | Low | Medium | Open |

### LB-001 · Unauthenticated peers emit ERROR lines at will into a lossy, size-unbounded log pipeline

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `consensus/cryptarchia-sync/src/libp2p/behaviour.rs:258,297,506`; `blend/network/src/core/with_edge/behaviour/handler/receiving.rs:69`; `services/chain/chain-network/src/lib.rs:611,630,684`; `tracing/src/logging/local.rs:114-116`; `nodes/node/binary/src/config/tracing/serde/logger.rs:26-33` |
| Status | Open |

**Description**

ERROR passes every filter the node can be configured with (the global cap is `config.level`, and `tracing::Level` has no `OFF`, so ERROR is also the floor of any reload, see LB-003). The following sites log at ERROR once per unit of input that a peer with no stake and no prior relationship can produce at whatever rate the transport allows:

| Site | Trigger | Bound on the attacker |
|---|---|---|
| `cryptarchia-sync/.../behaviour.rs:258` `"Received excessive number of additional blocks"` | one chain-sync request stream carrying more than `MAX_ADDITIONAL_BLOCKS` known blocks | none; the stream is closed and the peer opens the next |
| `behaviour.rs:297` `"Rejected excess pending incoming request"` | the 11th concurrent inbound stream (`max_inbound_requests` is a concurrency limit, `libp2p/src/config/mod.rs:98`, not a rate) | none; every extra stream is one line |
| `behaviour.rs:506` `"Error while processing incoming request: {e}"` | any inbound stream whose first bytes do not decode as `RequestMessage` (`provider.rs:31-33`) | none |
| `with_edge/.../receiving.rs:69` `"Failed to receive message. Error {error:?}"` | an edge connection (accepted from any non-member peer while under `max_incoming_connections`, `with_edge/behaviour/mod.rs:273-293`) that closes or sends a truncated frame | one QUIC/TCP handshake per line |
| `chain-network/src/lib.rs:611` `"Proposal header failed the checks that need the header alone"` | a gossipsub proposal with a bad Ed25519 signature; `verify_header_alone` checks only the slot and the signature (`core/src/block/mod.rs:337-351`), and gossipsub runs with `ValidationMode::None` (`libp2p/src/behaviour/mod.rs:75`) so the publisher needs no key at all | none; the block id is also entered into the rejected cache (#143) |
| `lib.rs:630` `"Failed to reconstruct block from proposal"` | a validly signed proposal whose transaction references are unknown; the signature is over a header the attacker builds, so it costs one keypair | none |
| `lib.rs:684` `"Error processing reconstructed block"` | any proposal that reconstructs but fails apply | none |

Two properties of the pipeline turn the volume into harm beyond disk usage:

1. The file, stdout and stderr sinks are wrapped in `tracing_appender::non_blocking(writer)` with the builder defaults (`local.rs:114`): a 128,000-line ring and `is_lossy: true` (`tracing-appender-0.2.5/src/non_blocking.rs:67,231-233`). When a flood fills the ring, *subsequent* lines are dropped silently, whichever component emitted them. A peer that sustains the spam therefore suppresses the operator's own WARN/ERROR evidence for the duration, with only an internal dropped-line counter that nothing reads.
2. The shipped file sink rolls hourly and keeps 10 files (`logger.rs:29-33`), but `tracing-appender` has no size-based rotation (`rolling.rs` exposes `max_log_files` only). One hour of a single peer at 5,000 lines/s of ~200-byte ERROR lines is 3.6 GB per file, 36 GB retained, plus the same bytes on stdout (`stdout: true`) and on any exporter.

Peer-triggered *validation* failures elsewhere are already at DEBUG and structured (`blend/network/src/core/poq_verification.rs:88-99`; `chain-network/src/network/adapters/libp2p.rs:236`; `tx-service/src/network/adapters/libp2p.rs:74`; `services/network/src/backends/libp2p/swarm/mod.rs:205-217`), which is the right level. The sites above are the exceptions.

**Exploit scenario**

A node with no stake connects to a victim, opens chain-sync streams in a loop, writes one garbage byte per stream and closes it. Each stream costs the victim one ERROR line at `behaviour.rs:506` including the peer id and the decode error. After 128,000 lines outrun the appender thread the victim's own security logging goes dark until the flood stops; over hours the state directory's disk fills at the rate above. No blocklist or peer score is applied on this path.

**Recommendation**
- *Short term*: demote every peer-triggered line in the table to DEBUG with structured fields (as `poq_verification.rs:88-99` already does), and count them in a metric instead. Build the non-blocking writers with `.lossy(false)` for the file sink, or export the dropped-line counter as a metric so a flood is visible.
- *Long term*: a per-peer token bucket on inbound chain-sync streams and edge connections (also closes the concurrency-only limit at `behaviour.rs:297`), and a size-based rotation or `max_log_files` on bytes rather than count. A lint or review rule: no `warn!`/`error!` whose trigger is a remote message.

**References**: PR 126 / 206 (panic sweeps), #143 (rejected-block cache), #144 (gossipsub validation), #57.

### LB-002 · TRACE lines bind the node to its Blend messages and to its winning leader UTXO

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Privacy / Anonymity |
| Target | `blend/provers/src/crypto/core_and_leader/send.rs:244`; `blend/provers/src/provers/{core/mod.rs:90,136, leader/mod.rs:86,143, pow/mod.rs:88,197}`; `services/chain/chain-leader/src/leadership.rs:533` |
| Status | Open |

**Description**

At TRACE the Blend provers write, per outgoing message and per layer, the message type, the destination node index, the ephemeral signing public key, the hex key nullifier of the proof-of-quota, and the sender's own index in the membership:

```
send.rs:244  "Encapsulating layer {layer:?} of message type {payload_type:?} for node at index {node_index:?}
              with proof with public key and key nullifier: ({:?}, {:?}). Local node index: {:?}"
core/mod.rs:136 "Generated core PoQ ... with key nullifier {:?} and public key {:?}."
```

The key nullifier and the ephemeral public key are exactly the values every relay on the path sees on the wire. A reader of the log can therefore match "node *i* originated the message carrying nullifier *N* at *t*" against observations at any hop, which is the linkage the Blend layer exists to prevent (`bedrock-architecture-overview.md`, Cryptarchia section: proposer confidentiality is "extended to the network layer by routing proposals through the Blend Network").

The leader path does the same for the block: `leadership.rs:533` logs `"Found winning utxo with ID {:?} for slot {slot}"`. The header carries a proof and a nullifier, not the UTXO id; the log line is the only place the (slot → winning UTXO → wallet note) link is materialised. `overview-cryptoeconomics.md` §Anonymous Leaders Reward Protocol: "we must not link leaders to their blocks and rewards".

Why this is not a test-only concern: no crate sets a `tracing` `max_level_*` / `release_max_level_*` feature (workspace `Cargo.toml:292`, grep of every `Cargo.toml`), so the sites are compiled into the release binary and are one config line away (`tracing.level: trace`). They cannot be switched on remotely (LB-003 is capped by `config.level`), but once on, LB-004 ships them off the host in plaintext.

**Exploit scenario**

An operator raises `level` to `trace` to debug a Blend reachability problem and has Loki configured (LB-004). The collector, and anyone on the path to it, now holds a per-message list of (local index, layer indices, nullifier, ephemeral key) for the node's own traffic, and a per-slot list of the node's winning notes. Deanonymisation of that operator's proposals and Blend origination is then a lookup, not an attack.

**Recommendation**
- *Short term*: remove the nullifier, the ephemeral public key, the local index and the winning UTXO id from these lines, keeping timings and counts; or gate them behind an explicit `privacy-unsafe-logs` cargo feature that is off in release.
- *Long term*: the same treatment PR 128 asked for on `Debug`: a `Redacted` wrapper for wire-visible identifiers so they cannot be formatted by accident, and a CI grep that fails on `nullifier`/`utxo.id()`/`local_index` inside a logging macro.

**References**: `bedrock-architecture-overview.md` §Cryptarchia; `overview-cryptoeconomics.md` §Anonymous Leaders Reward Protocol; PR 128 (Debug hygiene), #44 (leader linkability), #22.

### LB-003 · `PUT /admin/tracing/filter` lets any API client silence or reshape logging

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Access Controls |
| Target | `nodes/node/binary/src/api/tracing.rs:15-31`; `nodes/api-common/src/paths.rs:53`; `services/tracing/src/lib.rs:380-403` (`reload_filters`); `tracing/src/filter/envfilter.rs:69-83,141-146` |
| Status | Open |

**Description**

The route takes a JSON `EnvFilterConfig` and replaces the per-sink `EnvFilter` on every logger sink (`lib.rs:393-400`). It sits on the same unauthenticated router as everything else (PR 118 LB-001 lists it by name; the router is built at `backend.rs:230-253` with no auth layer). Validation only rejects *logos* targets that are not in the catalogue (`envfilter.rs:69-83`); any level parseable as `tracing::Level` is accepted (`envfilter.rs:141-146`), and any non-logos target string is accepted verbatim.

What a caller can do:

- **Silence**: `{"filters": {"*": "error", "logos_blockchain": "error"}}` drops every INFO and WARN line on every sink, including the exporters. `Level` has no `OFF`, so ERROR is the floor, which is why LB-001's lines survive but the Blend spam warnings, `"Orphan block ignored due to queue size limit"`, the SDP and PoW state transitions, and every WARN in the workspace do not.
- **Amplify within the cap**: raise `*` (all third-party crates, WARN by default, `envfilter.rs:50`) and `libp2p_gossipsub` (ERROR by default, `envfilter.rs:9`) to `config.level`. With the shipped INFO this is a modest volume increase; it cannot reach DEBUG/TRACE because the registry is assembled with a global `LevelFilter::from(config.level)` (`lib.rs:321,331`), and `LevelFilter` as a layer answers `Interest::never()` for anything above it (`tracing-subscriber-0.3.23/src/filter/level.rs:11-27`). Verified: the cap holds.
- **Inconsistent state**: reloads are applied sink by sink and the first failure returns early (`lib.rs:393-400`), so a directive that one sink rejects leaves earlier sinks on the new filter and later ones on the old.

**Exploit scenario**

With the default bind this needs a local process or the CORS path from PR 118 LB-002 (any web page the operator opens can `PUT` to `127.0.0.1:8080`). The page sets the filters to `error` before driving the wallet endpoints from the same PR; the operator's log shows nothing of the transfer, because the wallet and mempool lines that would have recorded it are INFO/DEBUG.

**Recommendation**
- *Short term*: put the route behind whatever #119 introduces, and refuse to *lower* a logos target below the configured level unless the request carries an explicit flag.
- *Long term*: apply the reload atomically (build all filters, then swap), and emit one WARN line on the sinks *before* the swap recording the caller address and the new directives, so a silence request is itself logged.

**References**: PR 118 (#66) LB-001/LB-002, #119.

### LB-004 · Exporters ship every event and span field, including identity fields, in plaintext

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Data Exposure |
| Target | `tracing/src/logging/loki.rs:13-27`; `tracing/src/logging/gelf.rs:13-33`; `tracing/src/logging/otlp.rs:19-43`; `tracing/Cargo.toml:45`; workspace `Cargo.toml:296` |
| Status | Open |

**Description**

This answers the last item of #87 ("audit what the exporters send in plaintext today so #37 can rate the exposure"). All three exporters receive the same filtered event stream as the file sink (`services/tracing/src/lib.rs:246-265`). `tracing-loki` serialises the message, every event field, every field of every enclosing span, and the module/file/line (`tracing-loki-0.2.7/src/lib.rs:314-349`), labelled with the configured `host_identifier`. Its transport is `reqwest` with the crate's TLS features removed (`default = ["compat-0-2-1", "native-tls"]` in its `Cargo.toml:33-38`; the workspace requests only `compat-0-2-1`, `tracing/Cargo.toml:45`). GELF uses `connect_tcp` (`gelf.rs:18`), not the crate's `connect_tls`. OTLP is #87.

What leaves the host at the shipped INFO level, from the sites read:

| Site | Fields |
|---|---|
| `libp2p/src/swarm.rs:48`, `services/network/.../swarm/mod.rs:480-491` | own peer id, listen and bootstrap multiaddrs |
| `services/blend/src/lib.rs:333` | own SDP `locator` and `service_note_id` (the note that locks the stake) |
| `services/sdp/src/lib.rs:548-556, 725-733` | `provider_id`, `declaration_id`, `zk_id`, `tx_id`, epochs; these are public on chain but the line ties them to `host_identifier` |
| `services/blend/src/edge/mod.rs:338-343` | `local_node_index` in the membership |
| `services/blend/src/core/mod.rs:675-678` | the entire membership (`Node { id, address, public_key }` for every core node) each epoch |
| `wallet/src/lib.rs:669-675, 701-709` | LIB id, number of known keys and vouchers |
| `services/pow/src/service.rs:600-603, 1069-1073` | counts of winning tickets and claims |

At DEBUG the Blend PoQ failure line adds `sender_signing_key`, `message_id` and `peer_id` per failed message (`poq_verification.rs:88-99`), and at TRACE the LB-002 lines. Nothing at any level carries key material; the exposure is metadata and linkage, proportional to the level.

**Exploit scenario**

An operator points `logger.loki` at a hosted collector over the public internet. A passive observer on the path learns the node's peer id, addresses, stake note id and provider id under one host label, and (at DEBUG) which peers' Blend messages the node rejected and when. The host label plus the membership dump identifies the operator's node within the Blend graph without touching the node.

**Recommendation**
- *Short term*: document that the exporters are plaintext and that INFO already carries the identifiers above; refuse `http://` collectors that are not loopback or RFC 1918 unless an `allow_plaintext_export` flag is set (aligns with #87's "decide the supported transport").
- *Long term*: a redaction layer between the filter and the exporters that strips the identity fields listed in LB-002/LB-004 unless a `full_export` setting is on.

**References**: #87, PR 84 LB-003.

### LB-005 · An SDP locator's DNS label is written unescaped into every core node's log

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Auditing and Logging |
| Target | `core/src/sdp/mod.rs:168-196` (`Locator::try_from`); `services/blend/src/core/mod.rs:675-678`; `services/blend/src/edge/backends/libp2p/swarm.rs:298,486`; `services/blend/src/core/backends/libp2p/swarm.rs:353,580` |
| Status | Open |

**Description**

`Locator::try_from` bounds the byte length and rejects unspecified IPs and `/p2p/` components, nothing else. A `/dns/`, `/dns4/`, `/dns6/` or `/dnsaddr/` component is any UTF-8 string (`multiaddr-0.18.2/src/protocol.rs:300-303`), so a declaration can carry a label containing `\n`, `\r`, or `\x1b[` sequences within the 329-byte cap. `Multiaddr`'s `Debug` delegates to `Display` (`lib.rs:219-223`) and `Display` writes the label raw (`protocol.rs:711`); `libp2p-swarm`'s `DialError` `Display` writes the address the same way (`libp2p-swarm-0.47.1/src/lib.rs:1575-1583`); and `tracing-subscriber`'s formatter writes the `message` field without escaping (`fmt/format/mod.rs:1264-1272`, only non-message `&str` fields go through `Debug`).

The string reaches logs on two paths:

- Every core node, every epoch, at INFO, without any dial: `core/mod.rs:675` formats the whole `CoreEpochPublicInfo` with `{:?}`, which includes `Membership { core_nodes: { … Node { address: <locator> … } } }` (`blend/membership/src/lib.rs:16-25,28-36`).
- Every node that dials the declaring node, at ERROR, on failure: `edge/.../swarm.rs:486` (`{error}`), `:298` (`{:?}` of the address and `{error}`), `core/.../swarm.rs:353` (`{e:?}`), `:580` (`{error:?}` at WARN). A label with a control character never resolves, so the dial always fails and the line is always written.

Compare the network service, which logs dial failures only at DEBUG (`services/network/.../swarm/mod.rs:205-217`) and identify addresses only at TRACE (`identify.rs:19-56`): the unauthenticated multiaddr sources are already quiet. The locator is the one remote-controlled string that reaches INFO and ERROR, and it costs one declaration (minimum stake, refundable on withdrawal).

**Exploit scenario**

A provider declares locator `/dns4/x\n2026-09-12T10:00:00Z ERROR logos_blockchain::chain: consensus split detected at slot 1\n/tcp/1`. On the next epoch every core node writes the forged line, indistinguishable from a genuine one, into its file, stdout and collectors; SIEM rules keyed on message text fire on every node at once. The same payload with `\x1b]0;…\x07` retitles the terminal of anyone tailing the log.

**Recommendation**
- *Short term*: in `Locator::try_from`, reject DNS labels that are not valid hostnames (LDH rule: ASCII letters, digits, hyphen, dot, ≤253 bytes), which also removes the never-resolvable declarations; log the membership as a count at `core/mod.rs:675`.
- *Long term*: a formatter that escapes control characters in the message field for the file/stdout sinks (a thin `FormatEvent` wrapper), so no future remote string can forge a line.

**References**: #93 (Declarations encoding), #62 (membership derivation), PR 202 (raw error text in peer-facing messages, the mirror image of this finding).

## 5. Suggestions (non-security)

### S-001 · Membership dump at INFO grows with the network

`services/blend/src/core/mod.rs:675-678` formats every core node each epoch. At the 2^20 leaf cap of the membership tree (PR 206 LB-002) that is a single log line in the hundreds of MB. Log `membership.size()` and the epoch instead.

### S-002 · Two different logging defaults

`lb_tracing_service::TracingSettings::default()` writes a never-rotating `Simple` file named by unix timestamp in `.` (`services/tracing/src/lib.rs:154-178`); the node binary's `logger::Layers::default()` rolls hourly with 10 files (`logger.rs:20-42`). Anything constructing the service settings directly (tests, embedders via `c-bindings`) gets the unbounded variant. Keep one default.

### S-003 · Dropped-line counter is invisible

`NonBlocking` counts dropped lines internally; nothing reads it. Export it through `lb_tracing::metrics` so LB-001's silent drop is at least observable.

### S-004 · FFI logger is unfiltered

`c-bindings/src/logging.rs:19-23` writes to stderr with no level or target filter and ignores the host's tracing configuration. Route it through `tracing` so the embedder's filter and sinks apply.

### S-005 · Filter directive strings are interpolated

`envfilter_directives` joins map keys with `,` and `=` (`envfilter.rs:106-120`); a key containing either changes the parsed directive set. Harmless today because the level cap holds (LB-003), but validate the target syntax (identifier segments joined by `::`) at the boundary.

## 6. Checked and ruled out

- **Secrets in log calls.** Grep of every logging macro in `kms/*`, `services/key-management-system`, `wallet`, `services/wallet`, `blend/provers`, `core` for `key`, `secret`, `seed`, `note`, `nullifier`, `witness`: the KMS logs key *ids* and error `Debug` only (`services/key-management-system/src/lib.rs:198-203`); the wallet logs counts, block ids and voucher ids (`wallet/src/lib.rs:669,701,855`); the operators log channel failures without payloads (`kms/operators/src/ed25519/exfiltrate_secret_key.rs:38`, `derive_x25519.rs:32`, `zk/voucher.rs:39`). No call formats a `SecretKey`, `ZkKey`, `LeaderPrivate`, `VoucherSecret` or note value. `Debug` on those types is PR 128's finding, not exercised by any log site.
- **Config in logs.** No site formats a config or settings struct (`rg` for `?config|?settings|{config:?}|{:?} … config` over the workspace: only the three Blend prover lines, which format proofs). `OtlpServiceConfig::authorization_header` therefore never reaches a log. `serde_ignored` unknown keys go to stderr as key paths (`utils/src/yaml.rs:55-70,94`).
- **`println!` outside tests and tools.** `ledger/src/cryptarchia/mod.rs:1040,1046` are inside `#[cfg(test)] pub mod tests` (line 851). `nodes/node/binary/src/cli/keys.rs:297` prints a freshly generated key on the operator's explicit request when they decline to persist it (#122/#85 territory, by design). `libp2p/src/config/mod.rs:110` is a test.
- **Peer-supplied strings on the quiet paths.** Identify `listen_addrs` and `agent_version` are TRACE only (`identify.rs:19-56`); Kademlia add/remove at DEBUG or, for `remove_address`, WARN but only after a `WrongPeerId` dial (`swarm/mod.rs:205-210`), which requires the address to resolve and connect, so a control-character label cannot reach it; chain-sync error strings are built locally (`errors.rs`); gossip decode failures are DEBUG (`adapters/libp2p.rs:236`, `tx-service … libp2p.rs:74`).
- **Default filter policy.** `default_log_filter` sets `*=warn`, `logos_blockchain=<level>`, and quiets `libp2p_gossipsub` to ERROR and the HTTP/OTLP stacks to WARN (`envfilter.rs:8-19,49-62`); `tower_http` request/response lines are configured at TRACE (`backend.rs:250-253,265-268`) and fall under the `*=warn` default. The pprof routes exist only under the `profiling` feature (PR 206 LB-001).
- **Level cap.** The `LevelFilter` layer is a hard global cap (`filter/level.rs:11-27`); the reload cannot exceed `config.level`. Verified in source, cited in LB-003.
- **ANSI.** `tracing-subscriber` is built with `env-filter, std` only (`tracing/Cargo.toml:47`); whether another crate unifies the `ansi` feature in was not established (`nu-ansi-term` is in `Cargo.lock`), so no claim is made about colour codes in file logs.

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
