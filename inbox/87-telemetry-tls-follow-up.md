# Audit Report — Telemetry exporter transport security

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/87`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `tracing`, `services/tracing`, and node tracing configuration
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `bedrock-genesis-block.md`, `p2p-network-bootstrapping.md`
Date: `2026-09-16` — author: `Codex` — status: `draft`

---

## 1. Summary

- Overall assessment: no new findings; the review re-verifies canonical #464 / 36-LB-003 for OTLP and re-verifies/extends canonical #460 / 37-LB-004 for Loki with new runtime evidence.
- Findings: `0` critical · `0` high · `0` medium · `0` low · `0` informational
- Key themes: unsupported HTTPS configuration; plaintext telemetry export; asynchronous exporter failure; evidence completeness.
- Must-fix before launch: none from issue #87 alone; preserve the canonical recommendations and ratings under #464 and #460.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `tracing/Cargo.toml` | OTLP and Loki dependency feature selections |
| `tracing/src/logging/{loki,otlp}.rs` | Loki and OTLP log exporter construction |
| `tracing/src/{tracing,metrics}/otlp.rs` | OTLP span and metric exporter construction |
| `services/tracing/src/lib.rs` | Assembly and error propagation for exporter layers |
| `nodes/node/binary/src/config/tracing/serde/{logger,tracing,metrics}.rs` | Endpoint parsing and absence of scheme validation |
| Resolved dependency graph and relevant third-party source | `cargo tree -e features`; reqwest, tonic, opentelemetry-otlp, and tracing-loki transport behavior |

**Out of scope**

The node's full devnet startup, collector interoperability, and GELF transport were not tested. The requested production-node checks for OTLP HTTP, OTLP gRPC, and Loki were not completed; in particular, no collector or production node was started. Third-party cryptographic correctness and the security of a separately deployed collector are assumed correct. No source repository was modified.

**Assumptions**

The target and specification revisions above are authoritative. The operator controls telemetry configuration, and the collector may be remote or reachable over an untrusted network. A configured `http://` endpoint is intentionally plaintext; this report does not treat it as an automatic downgrade from HTTPS.

## 3. Method

- Manual review of the in-scope paths, working through issue `#87`, parent issue `#26`, and the repository-wide security context in `#19`.
- Canonical-finding comparison against #464 / 36-LB-003 and #460 / 37-LB-004. The OTLP evidence matches #464's core defect; the Loki evidence extends #460's exporter-content and transport analysis.
- Read the four issue-mandated LIPS documents in full. They establish the node's network and deployment context but do not specify a telemetry TLS policy; the transport decision therefore remains an implementation/configuration requirement.
- Resolved features at the target revision with `cargo tree -e features -p logos-blockchain-tracing --depth 5 --format '{p} {f}'`. The graph contains OTLP HTTP/gRPC transport features, but no OTLP TLS feature, no reqwest TLS feature, no tonic TLS feature, and only `tracing-loki`'s compatibility feature.
- Inspected the resolved crate manifests and connector code for `opentelemetry-otlp`, `opentelemetry-http`, `tracing-loki`, `tonic`, and `reqwest`.
- Dynamic testing: a temporary offline probe built with `cargo +nightly-2026-07-05` against the cached `reqwest 0.12.28` and `tracing-loki 0.2.7` dependencies. It sent an HTTPS request and emitted ten Loki events; no production node or collector was started.
- Evidence qualification: issue #87 requested a dynamic node test covering OTLP, Loki, and gRPC, including the exact error and whether the node survives. That requested check remains unperformed. The new runtime evidence exercises cached reqwest and Loki only.

## 4. Re-verification

No new findings are created by issue #87. The following existing canonical findings are re-verified or extended.

### #464 / 36-LB-003 · OTLP HTTPS endpoints are accepted without a usable TLS transport

| | |
|---|---|
| Canonical severity | Low |
| Canonical difficulty | High |
| Canonical category | Configuration |
| Target | `tracing/Cargo.toml:27-35,45`; `tracing/src/logging/{loki,otlp}.rs`; `tracing/src/{tracing,metrics}/otlp.rs`; `nodes/node/binary/src/config/tracing/serde/{logger,tracing,metrics}.rs` |
| Re-verification status | Re-verified with new runtime evidence |

**Description**

The node parses exporter endpoints as generic `Url` values and does not reject `https://`. The exporter constructors then pass the URL string directly to clients:

```rust
// tracing/src/logging/otlp.rs:45-72
.with_tonic().with_endpoint(config.service.url.to_string())
.with_http().with_endpoint(config.service.url.to_string())
```

The same direct endpoint construction is used for OTLP traces and metrics. However, `tracing/Cargo.toml` requests `opentelemetry-otlp`'s HTTP/gRPC features without any TLS feature, uses workspace `tonic` with default features disabled, and requests only `tracing-loki`'s `compat-0-2-1` feature. The resolved graph consequently has no HTTP-client TLS backend for these exporters. The node's unrelated libp2p QUIC rustls dependency does not configure these HTTP clients.

This is an accepted configuration that is not rejected at startup with a useful scheme-specific error. The effect is most likely an exporter initialization or background send error rather than a process crash, depending on the exporter path. The focused runtime probe confirmed the reqwest behavior:

```text
invalid URL, scheme is not http
```

The canonical #464 rating remains `Low / High / Configuration`. High difficulty is appropriate because the demonstrated path requires privileged, operator-controlled deployment configuration, as provided for by Appendix A.

**Exploit scenario**

An operator configures `logger.otlp`, `tracing.otlp`, or `metrics.otlp` with an HTTPS collector URL. The configuration parses, but the exporter cannot establish an HTTPS connection. The node may therefore appear healthy while logs, traces, or metrics silently fail to reach the operator's TLS-only collector. If the operator changes the endpoint to `http://` to restore delivery, the telemetry travels without transport encryption. There is no evidence of an automatic HTTPS-to-HTTP downgrade.

**Canonical recommendation**

- *Short term*: explicitly choose the supported policy. If HTTPS is supported, enable and test a TLS backend for each path: reqwest rustls for HTTP OTLP, plus the corresponding `opentelemetry-otlp`/tonic TLS feature for OTLP gRPC. If HTTPS is not supported, reject `https` endpoints during config validation with an actionable error and document that HTTP is plaintext.
- *Long term*: centralize scheme-aware exporter validation and add startup/unit tests covering HTTP and HTTPS for logger OTLP, tracing OTLP, and metrics OTLP. Surface background exporter failures through a health/diagnostic signal so a configured collector cannot fail silently.

### #460 / 37-LB-004 · Loki exports identity-bearing telemetry without transport encryption

| | |
|---|---|
| Canonical severity | Low |
| Canonical difficulty | Medium |
| Canonical category | Data Exposure |
| Target | `tracing/src/logging/loki.rs:13-27`; `services/tracing/src/lib.rs:246-265`; `tracing/Cargo.toml:45`; `tracing/src/logging/{otlp,gelf}.rs` |
| Re-verification status | Re-verified and extended for Loki |

**Description**

The Loki layer receives the same filtered event stream as the file sink. `tracing-loki` serialises the message, every event field, every field of every enclosing span, and the module/file/line, labelled with the configured `host_identifier`. Its transport is `reqwest` with the crate's TLS features removed: the workspace requests only `compat-0-2-1` rather than the crate's default `native-tls` feature.

At the shipped INFO level, the exporter can send the node's peer id, listen and bootstrap multiaddrs, SDP `locator` and `service_note_id`, provider/declaration/transaction identifiers and epochs, Blend membership and local-node index, wallet state counts, and PoW claim counts. At DEBUG or TRACE, additional Blend and leadership linkage fields are included. The canonical #460 report documents these contents and their identity-bearing relationship to `host_identifier`; this review confirms that the current Loki construction still has the same behavior.

The new runtime probe emitted ten Loki events using the cached exporter. With an HTTPS endpoint, the Loki layer accepted the URL but its background task repeatedly logged `couldn't send logs to loki` while attempting `/loki/api/v1/push`; this provides runtime evidence of failed transport, not evidence of successful TLS.

**Exploit scenario**

An operator points `logger.loki` at a hosted collector over the public internet. If HTTPS delivery fails and the operator switches to plaintext HTTP, a passive observer on the path learns the node's peer id, addresses, stake note id, and provider id under one host label, as well as the other identity-bearing fields described above. Nothing in this re-verification indicates that key material is exported; the exposure is metadata and linkage.

**Canonical recommendation**

- *Short term*: document that the exporters are plaintext and that INFO already carries the identifiers above; refuse `http://` collectors that are not loopback or RFC 1918 unless an `allow_plaintext_export` flag is set (aligns with #87's transport decision).
- *Long term*: add a redaction layer between the filter and the exporters that strips the identity fields listed in #460 unless a `full_export` setting is enabled.

### Evidence completeness

The issue checklist requested a dynamic node test for OTLP, Loki, and gRPC that recorded the exact error and whether the node survived. That remains outstanding. This report dynamically exercised cached reqwest and Loki only; it did not run the node, a collector, or the OTLP gRPC path. The partial evidence supports the re-verifications above but must not be presented as completion of that requested checklist.

**References**: #464 / 36-LB-003; #460 / 37-LB-004; issue `#87`; parent issue `#26`; #19.
