# Audit Report — API transaction lookup re-verification

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/258`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `services/api` transaction lookup, transaction storage adapters, HTTP route wiring
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md` (the parent issue states that no area specification covers storage/API)
Date: `2026-09-18` — author: `codex` — status: `final`

---

## 1. Summary

- Overall assessment: Re-verification confirms that the transaction-by-hash HTTP endpoint cannot decode a transaction stored by either current writer; the defect is already tracked by canonical finding `31-LB-001` (issue #487).
- Findings: `0` critical · `0` high · `0` medium · `0` low · `0` informational
- Key themes: `binary storage codec versus JSON HTTP decoding`, `missing endpoint coverage`
- Must-fix before launch: none new; see the existing canonical finding below.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/api/src/http/storage/adapters/rocksdb.rs` | HTTP storage adapter and transaction byte decoding. |
| `services/api/src/http/mantle.rs`, `nodes/node/binary/src/api/{handlers,routes}.rs` | Generic lookup, response mapping, and route wiring. |
| `services/tx-service/src/storage/adapters/rocksdb.rs` | Mempool transaction storage writer and matching reader. |
| `services/chain/chain-service/src/storage/adapters/storage.rs`, `services/storage/src/api/chain/requests.rs` | Consensus transaction writer, typed reader, and storage request path. |
| `nodes/node/http-client`, `tests/`, `nodes/` | Search for a client or integration test exercising the endpoint. |

**Out of scope**

The broader storage backend, API authentication, transaction validation, historical retention policy, and all unrelated HTTP endpoints were not reviewed. Third-party behavior of `serde_json`, `bincode`, RocksDB, Axum, and Overwatch was assumed correct.

**Assumptions**

The pinned `logos-blockchain` and `logos-lips` revisions in the header are the review source of truth. The parent issue's statement that no area specification covers this API/storage seam is accepted; the two core overview specifications were read as required context. This report does not duplicate an existing canonical finding.

## 3. Method

- Read issue `#258`, its parent issue `#15`, and repo-level context issue `#19`.
- Read `bedrock-architecture-overview.md` and `overview-cryptoeconomics.md` in full at the pinned `logos-lips` revision.
- Traced every transaction-by-hash path from `nodes/node/binary/src/api/routes.rs` through `handlers.rs`, `services/api/src/http/mantle.rs`, and the RocksDB adapter.
- Compared both storage writers (`Tx::to_bytes`) with the API reader (`serde_json::from_slice`) and the typed chain-service reader (`Tx::from_bytes`).
- Searched the node's clients and tests for requests to `/cryptarchia/transaction/:id`; no endpoint test or client method was found.
- Automated tooling: none. Dynamic testing: none; the mismatch is established from the storage and decode paths.

## 4. Findings

No new `LB-NNN` finding is opened. The confirmed defect is the existing canonical finding `31-LB-001`, titled “HTTP transaction lookup decodes bincode-stored transactions with `serde_json`, so the endpoint can never return one,” tracked at [issue #487](https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/487) and originally reported in [processed/31-serde-untrusted-input.md](../processed/31-serde-untrusted-input.md).

### Re-verification of `31-LB-001`

At `services/api/src/http/storage/adapters/rocksdb.rs:76-82`, the API maps each byte value returned by `StorageMsg::get_transactions_request` through `serde_json::from_slice`. The mempool writer at `services/tx-service/src/storage/adapters/rocksdb.rs:49-56` calls `item.to_bytes()`, and the chain-service writer at `services/chain/chain-service/src/storage/adapters/storage.rs:214-222` calls `Tx::to_bytes(&tx)`. The blanket `SerializeOp` implementation in `core/src/codec/mod.rs:22-25` uses the bincode serializer, so the API's JSON decoder rejects an existing stored transaction and propagates the error to the HTTP handler.

The parallel typed reader at `services/chain/chain-service/src/storage/adapters/storage.rs:232-250` calls `Tx::from_bytes`, confirming the intended decoder for the same storage values. The route is wired at `nodes/node/binary/src/api/routes.rs:73`, and the handler calls the adapter at `nodes/node/binary/src/api/handlers.rs:1788-1793`. A repository-wide search found no test or HTTP-client method that exercises this route.

The prior canonical report's recommendation remains applicable: decode with `Tx::from_bytes` (and add an endpoint test that stores a transaction through the storage request and fetches it through the route), or expose a typed storage accessor that owns the encoding. No new impact, root cause, or trust boundary was identified, so assigning a second LB identifier would create a duplicate finding.

## 5. Suggestions

### S-001 · Add regression coverage for the transaction-by-hash route

Add a handler or integration test that writes a transaction using `StorageMsg::store_transactions_request`, calls `GET /cryptarchia/transaction/:id`, and asserts that the response contains the original transaction. This would prevent another human-readable/binary codec mismatch and would close the coverage gap identified in this re-verification.

