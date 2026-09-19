# Audit Report — Peer-triggered logging volume follow-up

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/218`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `consensus/cryptarchia-sync`, `blend/network`, `services/chain/chain-network`, `tracing`, `services/tracing`, and the requested ingress logging sweep
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`
Date: `2026-09-19` — author: `Codex` — status: `final`

---

## 1. Summary

- Overall assessment: the prior peer-triggered log-volume observation remains valid at the pinned target; the node still turns several unauthenticated network events into unbounded-rate ERROR records sent through lossy default sinks.
- Findings: `0` critical · `0` high · `0` medium · `1` low · `0` informational
- Key themes: peer-triggered ERROR volume, silent log loss, missing per-event rate limiting
- Must-fix before launch: demote or rate-limit the peer-triggered records and expose the lossy appender's dropped-line counter.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `consensus/cryptarchia-sync/src/libp2p/behaviour.rs` | Excess streams, malformed requests, and response-processing errors |
| `blend/network/src/core/with_edge/behaviour/handler/receiving.rs` | Truncated or failed inbound edge messages |
| `services/chain/chain-network/src/lib.rs` | Proposal reconstruction and apply errors from gossipsub input |
| `tracing/src/logging/local.rs`, `services/tracing`, node tracing serde | Queue behavior, filters, sink construction, rotation defaults |
| `zone-sdk`, `logos_sql`, `services/api`, `c-bindings` | Requested follow-up sweep for network, HTTP, and FFI-reachable `warn!`/`error!` sites |

**Out of scope**

No source changes were made. I did not review third-party networking, storage, or logging implementations beyond the pinned `tracing-appender` behavior relevant to the sink; I did not run a full node or cluster flood test.

**Assumptions**

The operator uses the shipped node logging defaults and the host is not compromised. A remote peer is unauthenticated with respect to application stake or membership unless the specific path performs an additional admission check.

## 3. Method

- Re-read issue `#218`, parent issue `#24`, and prior report `#37`; inspected the target checkout at the exact revision in the header.
- Re-read the two mandatory core specifications in full. Neither specifies logging behavior; the architecture overview's proposer-confidentiality goal and the cryptoeconomics overview's anonymous-leader goal were recorded as context only.
- Manually traced all seven sites listed by the issue and the requested `zone-sdk`, `logos_sql`, `services/api`, and `c-bindings` sweep.
- Ran `cargo test -p logos-blockchain-cryptarchia-sync --lib libp2p::behaviour::tests::test_reject_excess_download_requests -- --exact` — passed.
- Ran `cargo test -p logos-blockchain-cryptarchia-sync --lib libp2p::behaviour::tests::test_reject_protocol_violation_too_many_additional_blocks -- --exact` — passed.
- Ran `cargo test -p logos-blockchain-tracing --lib` — 5 tests passed.
- Ran an isolated pinned `tracing-appender 0.2.5` probe: the default 128,000-line lossy queue accepted a 200,000-line burst of 200-byte records into a sink taking 1 ms per write; after 118 ms, 71,982 lines had been dropped. After a 1-second drain window and guard shutdown, 207,600 bytes had been written. This measures the sink behavior, not a live node's network rate.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Unauthenticated peers can refill ERROR sinks without an application-level rate limit | Denial of Service | Low | Low | Open; re-verified from report #37 |

### LB-001 · Unauthenticated peers can refill ERROR sinks without an application-level rate limit

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `consensus/cryptarchia-sync/src/libp2p/behaviour.rs:258,297,506`; `blend/network/src/core/with_edge/behaviour/handler/receiving.rs:69`; `services/chain/chain-network/src/lib.rs:611-631,668-686`; `tracing/src/logging/local.rs:110-118` |
| Status | Open; same underlying issue as report #37 LB-001 |

**Description**

The target still emits an ERROR event for each of several bounded-but-repeatable peer inputs:

- a chain-sync request with too many additional block identifiers (`behaviour.rs:258`);
- every inbound stream after the configured concurrent-request count is full (`behaviour.rs:290-300`);
- every malformed inbound request that reaches request processing (`behaviour.rs:506`);
- every failed or truncated edge message (`receiving.rs:69`); and
- each gossipsub proposal that fails header validation, reconstruction, or apply processing (`chain-network/src/lib.rs:611-631,668-686`).

The per-request and per-connection bounds limit work for an individual input, but none of these paths applies a per-peer log token bucket, coalesces repeated failures, or demotes the event below the shipped root `INFO` threshold. The gossipsub adapter accepts decodable proposals before `verify_header_alone`; a peer can therefore repeatedly publish proposals that reach the ERROR logging branches without needing stake or a valid block.

All local sinks use `tracing_appender::non_blocking(writer)` without a builder override (`tracing/src/logging/local.rs:110-118`). In the pinned `tracing-appender 0.2.5`, that selects a 128,000-line bounded queue in lossy mode. When full, writes succeed to the caller while the queue drops the record and increments an internal counter. The application never reads or exports that counter. The node defaults enable stdout and a file sink (`services/tracing/src/lib.rs:160-177,232-242`; `nodes/node/binary/src/config/tracing/serde/logger.rs:20-42`), so the same event stream can consume both sinks. The node's rolling retention is file-count based, not byte based.

The isolated probe reproduced the queue-loss property: with a 1 ms/write sink, 200,000 200-byte records caused 71,982 dropped records in 118 ms. This does not establish that a real node reaches exactly that rate; it establishes that the production default silently loses records when event production outruns the sink. A sustained 5,000 records/s stream of 200-byte lines would be approximately 1 MB/s, or 3.6 GB/hour per byte-for-byte sink before filesystem and formatting overhead; that figure remains a calculation, not a live-node measurement.

**Exploit scenario**

A remote peer opens a valid chain-sync connection and repeatedly sends malformed request bytes, or keeps opening streams after the inbound-request limit is occupied. Each attempt produces an ERROR record and the stream is closed. Alternatively, the peer repeatedly publishes decodable but invalid proposals to gossipsub. Once the sink queue is saturated, subsequent records—including unrelated security and consensus errors—are silently dropped until the writer catches up. With the shipped file rotation, a sustained flood also grows the current hourly file and can consume the configured retention budget rapidly.

**Recommendation**

- *Short term*: demote expected peer-invalid-input paths to `DEBUG` or structured counters, and add per-peer/event-class coalescing or a token bucket before logging. Export the `NonBlocking::error_counter().dropped_lines()` value through the existing metrics path, at least once per sink.
- *Long term*: make logging policy explicit for network-validation failures: bounded counters for high-rate conditions, sampled diagnostics for representative failures, and byte-based retention or an overall disk quota in addition to file-count retention. Add a review/lint rule for `warn!`/`error!` whose trigger is a remote message.

**References**: report #37 LB-001; `tracing-appender 0.2.5` `NonBlockingBuilder` documentation and implementation; issue #218's requested measurement checklist.

## 5. Suggestions

### S-001 · Keep the requested ingress sweep separated from node-ingress findings

The remaining sites reviewed were not equivalent attack paths: `zone-sdk` logs errors from a client SDK's node responses, `logos_sql` logs local applier/publication state, `services/api` logs locally indexed missing block bodies and is bounded by the API's block limits, and `c-bindings` writes host-side FFI errors. They should not be promoted to the same network-peer DoS finding without a separate deployment and caller model.

## 6. Checked and ruled out

- The two existing chain-sync tests pass and exercise rejection of excess concurrent requests and excess additional blocks; they do not add a log-rate bound.
- The tracing filter tests pass. The default filter sets the Logos root target to the configured level, normally `INFO`, so the listed ERROR events are not filtered by default.
- No additional lower-cost unauthenticated ERROR site was identified in the requested `zone-sdk`, `logos_sql`, `services/api`, or `c-bindings` sweep. SDK errors require a client/zone process; SQL warnings/errors require local service state or an already accepted inscription; the API warning is emitted for locally indexed headers; and c-bindings logging is host-side FFI behavior.
- The full-node flood requested by the issue was not run because the pinned audit checkout had no built node binary and a cluster test would require the repository's host-like integration setup. The dynamic result above is deliberately labeled as an appender probe rather than a node measurement.

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
