# Audit Report — Telemetry exporter transport security

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/87`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `tracing`, `services/tracing`, and node tracing configuration
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `bedrock-genesis-block.md`, `p2p-network-bootstrapping.md`
Date: `2026-09-16` — author: `Codex` — status: `draft`

---

## 1. Summary

- Overall assessment: telemetry URLs accept `https://`, but the shipped OTLP and Loki exporter feature sets do not provide an HTTP-client TLS backend, so HTTPS export fails only when the background exporter attempts to send.
- Findings: `0` critical · `0` high · `0` medium · `1` low · `0` informational
- Key themes: unsupported HTTPS is accepted as valid configuration; exporter failures are asynchronous; selecting HTTP sends telemetry without transport encryption.
- Must-fix before launch: decide and document the supported telemetry transport, then either enable TLS for every exporter or reject unsupported HTTPS endpoints during configuration validation.

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

The node's full devnet startup, collector interoperability, exporter payload schema, and GELF transport were not tested. Third-party cryptographic correctness and the security of a separately deployed collector are assumed correct. No source repository was modified.

**Assumptions**

The target and specification revisions above are authoritative. The operator controls telemetry configuration, and the collector may be remote or reachable over an untrusted network. A configured `http://` endpoint is intentionally plaintext; this report does not treat it as an automatic downgrade from HTTPS.

## 3. Method

- Manual review of the in-scope paths, working through issue `#87`, parent issue `#26`, and the repository-wide security context in `#19`.
- Read the four issue-mandated LIPS documents in full. They establish the node's network and deployment context but do not specify a telemetry TLS policy; the transport decision therefore remains an implementation/configuration requirement.
- Resolved features at the target revision with `cargo tree -e features -p logos-blockchain-tracing --depth 5 --format '{p} {f}'`. The graph contains OTLP HTTP/gRPC transport features, but no OTLP TLS feature, no reqwest TLS feature, no tonic TLS feature, and only `tracing-loki`'s compatibility feature.
- Inspected the resolved crate manifests and connector code for `opentelemetry-otlp`, `opentelemetry-http`, `tracing-loki`, `tonic`, and `reqwest`.
- Dynamic testing: a temporary offline probe built with `cargo +nightly-2026-07-05` against the cached `reqwest 0.12.28` and `tracing-loki 0.2.7` dependencies. It sent an HTTPS request and emitted ten Loki events; no production node or collector was started.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | HTTPS telemetry endpoints are accepted without a usable TLS transport | Configuration | Low | Low | Open |

### LB-001 · HTTPS telemetry endpoints are accepted without a usable TLS transport

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Configuration |
| Target | `tracing/Cargo.toml:27-35,45`; `tracing/src/logging/{loki,otlp}.rs`; `tracing/src/{tracing,metrics}/otlp.rs`; `nodes/node/binary/src/config/tracing/serde/{logger,tracing,metrics}.rs` |
| Status | Open |

**Description**

The node parses exporter endpoints as generic `Url` values and does not reject `https://`. The exporter constructors then pass the URL string directly to clients:

```rust
// tracing/src/logging/otlp.rs:45-72
.with_tonic().with_endpoint(config.service.url.to_string())
.with_http().with_endpoint(config.service.url.to_string())
```

The same direct endpoint construction is used for OTLP traces and metrics, while `tracing/src/logging/loki.rs:13-26` passes the URL directly to `tracing_loki::layer`. However, `tracing/Cargo.toml` requests `opentelemetry-otlp`'s HTTP/gRPC features without any TLS feature, uses workspace `tonic` with default features disabled, and requests only `tracing-loki`'s `compat-0-2-1` feature. The resolved graph consequently has no HTTP-client TLS backend for these exporters. The node's unrelated libp2p QUIC rustls dependency does not configure these HTTP clients.

This is an accepted configuration that is not rejected at startup with a useful scheme-specific error. The effect is most likely an exporter initialization or background send error rather than a process crash, depending on the exporter path.

**Exploit scenario**

An operator configures `logger.otlp`, `tracing.otlp`, `metrics.otlp`, or `logger.loki` with an HTTPS collector URL. The configuration parses and the layer can be constructed, but the exporter cannot establish an HTTPS connection. In the focused runtime probe, `reqwest 0.12.28` returned:

```text
invalid URL, scheme is not http
```

The Loki layer accepted the HTTPS URL, then its background task repeatedly logged `couldn't send logs to loki` while attempting `/loki/api/v1/push`. The node may therefore appear healthy while logs, traces, or metrics silently fail to reach the operator's TLS-only collector. If the operator changes the endpoint to `http://` to restore delivery, the telemetry—including identity-bearing fields discussed in issue `#37`—travels without transport encryption. There is no evidence of an automatic HTTPS-to-HTTP downgrade.

**Recommendation**

- *Short term*: explicitly choose the supported policy. If HTTPS is supported, enable and test a TLS backend for each path: reqwest rustls for HTTP OTLP and Loki, plus the corresponding `opentelemetry-otlp`/tonic TLS feature for OTLP gRPC. If HTTPS is not supported, reject `https` endpoints during config validation with an actionable error and document that HTTP is plaintext.
- *Long term*: centralize scheme-aware exporter validation and add startup/unit tests covering HTTP and HTTPS for logger OTLP, tracing OTLP, metrics OTLP, and Loki. Surface background exporter failures through a health/diagnostic signal so a configured collector cannot fail silently.

**References**: issue `#87`; parent issue `#26`; prior feature-gating report `processed/36-feature-cfg-gating-sweep.md` LB-003; logging exposure report `processed/37-logging-tracing-leaks-log-volume.md` LB-004.

