# Audit Report — HTTP API: bind address, auth, CORS/TLS, request limits, information disclosure

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/66`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `b8c3c54ff6e808f247521befd7d103d9daa361aa` — component(s): `nodes/node/binary/src/api` (backend, routes, handlers, errors), `nodes/api-common/src/settings.rs`, `services/api/src/http`
Specs: `https://github.com/logos-co/logos-lips` @ `4b9d1ca1794f1e1b47aa40582f7793e2f84ed999` — read: `wallet-technical-standard.md` (in full); `mantle-transaction-encoding.md` and `bedrock-v1.1-mantle-specification.md` consulted for the submit/sign endpoints' structures; `overview-cryptoeconomics.md`, `bedrock-architecture-overview.md` (core)
Date: 2026-09-10 — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: The HTTP API has **no authentication of any kind on any endpoint**, and the router exposes wallet signing, fund transfer, and reward-claim operations that the node performs with its own keys. Anyone who can reach the port can move the operator's funds. The default CORS policy is a wildcard, so a web page the operator merely visits can drive these endpoints against the localhost-bound server. Request-size, timeout and concurrency limits are set and block ranges are clamped, so those are not issues.
- Findings: 0 critical · 1 high · 1 medium · 2 low · 0 informational
- Key themes: unauthenticated wallet control; wildcard CORS turning a local API into a browser-reachable one; cleartext transport; verbatim internal errors in bodies.
- Must-fix before launch: LB-001 (authenticate the privileged/wallet endpoints). LB-002 (do not default CORS to `Any`) compounds it.

The default binding is `127.0.0.1:8080` (`nodes/api-common/src/settings.rs`, `nodes/node/binary/src/config/api/serde.rs`, `nodes/node/standalone-node-config.yaml:170`), which limits the blast radius to the operator's own host — but LB-002 removes that limit in the browser case, and an operator that binds a routable interface (no guard prevents it) turns LB-001 into a remote, unprivileged loss-of-funds, i.e. Critical.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `nodes/node/binary/src/api/backend.rs` | axum server assembly: CORS, body limit, timeout, concurrency, bind, TLS |
| `nodes/node/binary/src/api/routes.rs` | the full route table (every endpoint) |
| `nodes/node/binary/src/api/handlers.rs:1921-2010` (`post_transactions_transfer_funds`), `:2018-2230` (`sign_tx_ed25519`, `sign_tx_zk`, `fund`), `:1831` (`get_balance`) | wallet signing/transfer endpoints |
| `nodes/node/binary/src/api/errors.rs` | error → response mapping |
| `nodes/api-common/src/settings.rs`, `nodes/node/binary/src/config/api/serde.rs` | server settings and defaults |
| `nodes/node/binary/src/api/handlers.rs` block-range/stream handlers | range bounding (ruled out) |

**Out of scope**

The C FFI surface (`c-bindings`, parent #18's second half) is not covered here. Handler-internal correctness of the wallet, SDP, PoW and blend services they call is assumed as reviewed elsewhere. `axum`, `tower-http`, `tower`, `utoipa` assumed correct. Transaction wire-format validation (`mantle-transaction-encoding`) is covered by the codec issues (#10, #69); this report treats the submit path only at the HTTP layer.

**Assumptions**

Repo-level facts from issue #19 hold (panic lints allowed, `overflow-checks` off). The node is assumed to hold spendable wallet keys (the wallet service is wired into the router).

## 3. Method

- Manual review of the axum backend assembly (`backend.rs`), the complete route table (`routes.rs`), the wallet handlers, and the error envelope, tracing which endpoints are unauthenticated and which perform privileged or funds-moving work.
- Configuration review of both `AxumBackendSettings` definitions and the shipped `*.yaml` to establish default bind address, CORS, body limit, timeout and concurrency.
- Spec: `wallet-technical-standard.md` read in full (to understand what key material the wallet endpoints operate on); `mantle-transaction-encoding.md` / `bedrock-v1.1-mantle-specification.md` consulted for the submit/sign request shapes. The HTTP-surface findings do not turn on the transaction wire format.
- No dynamic testing; no tooling run. Exploit steps below are from reading the router and handlers.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | No authentication on any endpoint; wallet transfer/sign/fund move funds unauthenticated | Authentication | High | Low | Open |
| LB-002 | CORS defaults to wildcard, exposing the local API to any visited web page | Access Controls | Medium | Low | Open |
| LB-003 | API served over plain HTTP, no TLS | Cryptography | Low | Medium | Open |
| LB-004 | Internal error strings returned verbatim in response bodies | Data Exposure | Low | Low | Open |

### LB-001 · `No authentication on any endpoint; wallet transfer, signing and fund endpoints move funds unauthenticated`

| | |
|---|---|
| Severity | High (Critical if bound to a routable interface) |
| Difficulty | Low |
| Category | Authentication / Access Controls |
| Target | `nodes/node/binary/src/api/routes.rs` (route table), `nodes/node/binary/src/api/handlers.rs:1921-2010` (`post_transactions_transfer_funds`), `:2138-2230` (`fund`), `:2018-2130` (`sign_tx_ed25519`/`sign_tx_zk`), `backend.rs` (`serve` — no auth layer) |
| Status | Open |

**Description**

`AxumBackend::serve` (`backend.rs`) assembles the router with CORS, body-limit, timeout, concurrency and trace layers, and **no authentication layer**. A search for any auth mechanism (`Authorization`, bearer, API key, basic auth, token) across `nodes/node/binary/src/api`, `nodes/api-common` and `services/api` returns nothing. Every row of `routes.rs` is therefore reachable by any client that can open a connection to the port.

Several of those rows are privileged and operate on the node's own keys:

- `POST …/transactions/transfer-funds` → `post_transactions_transfer_funds` (`handlers.rs:1921`) takes a JSON body with `recipient_public_key`, `funding_public_keys` and `amount`, calls `wallet_api.transfer_funds(...)` — which signs with the node's wallet keys — and submits the signed transaction to the mempool. The recipient and amount are entirely caller-supplied.
- `POST …/fund` → `fund` (`:2138`) and `POST …/sign/tx/{ed25519,zk}` → `sign_tx_ed25519` / `sign_tx_zk` (`:2018`, `:2078`) likewise sign with node-held keys (`wallet.sign_tx_with_ed25519`, `wallet.sign_tx_with_zk`).
- `POST …/leader/claim` → `leader_claim`, the PoW `PUT …/mining/*` and `POST …/pow/claim`, the SDP `POST …/sdp/{declaration,activity,withdrawal,set-declaration-id}`, `POST …/peers/dial` (`dial_peer`), and `PUT …/admin/tracing-filter` (`reload_tracing_filter`) are all state-changing and all unauthenticated.
- `GET …/wallet/balance`, `GET …/leader/claim/vouchers`, `GET …/network/info` disclose wallet and peer state.

There is no confirmation step, no allowlist, no capability token — the presence of a signing wallet behind an unauthenticated HTTP endpoint is the whole vulnerability.

**Exploit scenario**

With the node bound to its default `127.0.0.1:8080`, any process on the host (any local user, any compromised or sandboxed application) issues:

```
POST http://127.0.0.1:8080/<transfer-funds path>
Content-Type: application/json
{ "recipient_public_key": "<attacker>", "funding_public_keys": [...], "amount": <all of it>, ... }
```

The node signs the transfer with its wallet keys and broadcasts it. The operator's funds are gone. If the operator has bound the API to a LAN or public interface — nothing in the code or config warns against it — any network peer does the same remotely, which is the Critical case. LB-002 extends the localhost case to any web page the operator visits.

**Recommendation**

- *Short term*: put an authentication layer in front of the router (a bearer token or local unix-socket-only transport), and at minimum gate the wallet (`transfer-funds`, `fund`, `sign/*`), `leader/claim`, `pow/*`, `sdp/*`, `peers/dial` and `admin/*` endpoints behind it. Fail closed when no credential is configured, rather than serving them open.
- *Long term*: split the API into a public read surface and an authenticated control surface on separate listeners, so the signing/admin endpoints are never reachable on the same socket as public reads. Document that the control listener must never be bound to a routable interface without auth.

**References**: `wallet-technical-standard.md` (the key material `sign_tx_*` operates on); template Appendix A (Critical = loss of funds by an unprivileged participant).

### LB-002 · `CORS defaults to a wildcard, exposing the local API to any visited web page`

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Access Controls |
| Target | `nodes/node/binary/src/api/backend.rs` (`serve`, CORS construction); default `cors_origins: []` in `nodes/node/standalone-node-config.yaml:171` and `config/api/serde.rs` |
| Status | Open |

**Description**

`serve` builds the CORS layer as:

```rust
let mut builder = CorsLayer::new();
if self.settings.cors_origins.is_empty() {
    builder = builder.allow_origin(Any);
}
// ... per-configured-origin otherwise ...
let cors_layer = builder.allow_headers(vec![CONTENT_TYPE, USER_AGENT]).allow_methods(Any);
```

When `cors_origins` is empty — the shipped default (`standalone-node-config.yaml:171`, and the `Default` impl in `config/api/serde.rs`) — the API answers every origin with `Access-Control-Allow-Origin: *` and `allow_methods(Any)`. Combined with LB-001 (no auth), any web page the operator opens can script cross-origin `fetch` calls to `http://127.0.0.1:8080/...`: the preflight for a JSON `POST` passes (any origin, any method, `Content-Type` allowed), so the state-changing POSTs in LB-001 execute, and because the allowed origin is reflected as `*` without credentials the page can also read the responses (wallet balance, peer list). The localhost bind, normally a containment boundary, is bypassed because the victim's own browser originates the request.

**Exploit scenario**

An operator running a node browses to any attacker-influenced page (an ad, a forum post, a compromised site). The page's JavaScript POSTs a `transfer-funds` request to `127.0.0.1:8080`; the node signs and broadcasts it; funds are drained. No local code execution is needed — only that the node runs on a machine that also browses the web.

**Recommendation**

- *Short term*: do not default to `allow_origin(Any)`. Default to no allowed origins (same-origin only) and require the operator to name origins explicitly; never pair a wildcard origin with the privileged endpoints. This alone blunts the browser drive-by even before LB-001 is fixed.
- *Long term*: once the control surface is authenticated and/or on a separate listener (LB-001), keep wildcard CORS (if wanted) only on the public read listener.

**References**: Fetch/CORS semantics; LB-001.

### LB-003 · `API served over plain HTTP, no TLS`

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Cryptography / Data Exposure |
| Target | `nodes/node/binary/src/api/backend.rs` (`axum::serve(listener, app)` over a plain `TcpListener`) |
| Status | Open |

**Description**

The server terminates plain HTTP — `TcpListener::bind` then `axum::serve`, with no TLS acceptor and no TLS settings in `AxumBackendSettings`. On the localhost default this is moot, but the moment the API is reached over any network (a bound LAN/public interface, or a reverse proxy configured without TLS), the signing-request bodies, wallet balances, and any future auth credential travel in cleartext and are open to interception and tampering.

**Recommendation**

- *Short term*: document that the API must sit behind a TLS-terminating proxy and must not be exposed in cleartext beyond loopback.
- *Long term*: support TLS termination in-process (a rustls acceptor configured from `AxumBackendSettings`) for deployments without a proxy.

### LB-004 · `Internal error strings returned verbatim in response bodies`

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Data Exposure / Error Reporting |
| Target | `nodes/node/binary/src/api/errors.rs` (`ApiError::Internal(error) => error_response(500, error.to_string())`) |
| Status | Open |

**Description**

`ApiError::Internal(DynError)` is rendered into the 500 response body as `error.to_string()`. Internal errors bubbled up from services (storage, mempool, relay, wallet) are therefore returned verbatim to an unauthenticated client. These are not stack traces, but they leak internal state and implementation detail — storage-layer messages, relay failures, service wiring — useful for fingerprinting and for probing the node's internals. The `BadRequest`/`NotFound` variants also echo caller-influenced strings, which is benign, but the `Internal` path is the disclosure. Note `ApiError::InternalServerError` (the opaque variant) exists and is the right default for the `Internal` case too.

**Recommendation**

- *Short term*: return a generic body for `Internal` (as `InternalServerError` already does) and log the detail server-side with a correlation id, rather than sending `error.to_string()` to the client.
- *Long term*: make the error envelope distinguish client-safe messages from internal detail at the type level, so an internal error cannot be serialised into a body by construction.

## 5. Suggestions (non-security)

- **S-001 · Malformed `cors_origins` panics the node at startup.** `origin.as_str().parse::<HeaderValue>().expect("fail to parse origin")` in `serve` aborts the process if any configured origin is not a valid header value. Config-input panic (issue #19 notes panic lints are allowed); startup-only and operator-controlled, so not a security finding, but it should return a configuration error instead of panicking.
- **S-002 · Ruled out, recorded for completeness.** Request-body size (`max_body_size`, default 10 MiB, applied via both `DefaultBodyLimit` and `RequestBodyLimitLayer`), request timeout (`TimeoutLayer`, default 30 s) and a global `ConcurrencyLimitLayer` (default 500) are all configured, so oversized-body, slow-response and unbounded-concurrency vectors are handled. Block-range/stream handlers clamp `slot_to` to the tip/LIB anchor and validate `slot_from <= slot_to` (`handlers.rs` `resolve_*_window`, `errors.rs` `BlocksStreamWindowError`), so range queries are bounded to chain length rather than unbounded. There is, however, no **per-client rate limit** (only global concurrency); worth considering alongside the auth work.

---

## Appendix A — Definitions

Severity, difficulty and category ratings use the definitions in `docs/REPORT_TEMPLATE.md` Appendix A. No new scales were introduced.
