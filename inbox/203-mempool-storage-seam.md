# Audit Report — Transaction persistence and mempool consistency

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/203`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `services/tx-service`, `services/storage`, `services/chain/chain-service`, and `services/chain/chain-network`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`
Date: `2026-09-16` — author: `Codex` — status: `draft`

---

## 1. Summary

- Overall assessment: the transaction storage boundary reports relay acceptance as persistence success and collapses missing, unreadable, and undecodable transactions into the same absent stream item.
- Findings: `0` critical · `0` high · `0` medium · `2` low · `0` informational
- Key themes: asynchronous write acknowledgement gap; mempool status/view divergence; lossy read-side error handling.
- Must-fix before launch: make transaction writes acknowledge backend success before mempool admission, and preserve read errors instead of silently omitting transactions.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/tx-service/src/storage/adapters/rocksdb.rs` | Transaction relay adapter and byte decoding |
| `services/tx-service/src/backend/pool.rs` | Mempool admission, pending index, and view behavior |
| `services/storage/src/api/chain/requests.rs` | Storage request shape and transaction handlers |
| `services/storage/src/api/backend/rocksdb/chain.rs` | RocksDB transaction reads and writes |
| `services/storage/src/lib.rs` | Storage-service error reporting |
| `services/chain/chain-service/src/storage/adapters/storage.rs` | Chain-service transaction storage and decoding |
| `services/chain/chain-network/src/lib.rs` | Proposal reference resolution after a missing transaction |

**Out of scope**

Full-disk/read-only-device reproduction, node/devnet execution, RocksDB durability policy, and storage recovery `expect` paths already tracked by #79 were not re-audited. RocksDB and the transaction codec are treated as the intended backend/serialization implementations; this report concerns the contracts around them. No source repository was modified.

**Assumptions**

The target and specification revisions above are authoritative. A storage backend may fail because of a full or read-only filesystem, I/O failure, or corrupted stored bytes. The consensus protocol treats a locally unavailable transaction as a reconstruction failure rather than accepting a malformed block.

## 3. Method

- Manual review of the in-scope paths, working through issue `#203`, parent issue `#24`, and overlap checks against #79 and #81.
- Spec conformance against the two core LIPS overview documents. They contain no normative storage-contract requirement for this seam.
- Dynamic testing: `cargo test -p logos-blockchain-tx-service --target-dir /tmp/logos-audit-203-target --test mock storage_failure_does_not_mark_tx_pending` — 1 passed. This test injects a failing mempool adapter and confirms the pool itself does not mark a failed add as pending; it does not exercise a backend failure after the storage relay accepts `StoreTransactions`.
- No full-disk or rejecting-RocksDB integration reproduction was available. Static evidence covers the unacknowledged service boundary and the read-side loss of errors.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Transaction writes are acknowledged before the backend write completes | Configuration | Low | Low | Open |
| LB-002 | Transaction read failures are silently converted to missing items | Error Reporting | Low | Low | Open |

### LB-001 · Transaction writes are acknowledged before the backend write completes

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Configuration |
| Target | `services/storage/src/api/chain/requests.rs:67-69,140-142,484-492`; `services/tx-service/src/storage/adapters/rocksdb.rs:49-64`; `services/tx-service/src/backend/pool.rs:153-161` |
| Status | Open |

**Description**

`ChainApiRequest::StoreTransactions` carries only the transaction map; unlike `StoreBlockData`, it has no reply channel. The storage service invokes `backend.store_transactions(...)` and returns only to its own dispatcher, which logs any error. The mempool adapter therefore treats successful relay delivery as successful persistence:

```rust
self.storage_relay
    .send(StorageMsg::store_transactions_request(transactions))
    .await?;
```

`Mempool::add_item` then updates `by_prefix`, the evictor, and `pending_items` after that send returns. `RocksBackend::bulk_store` can still return an I/O error from `rocks_db.write(batch)` after the request has been accepted. The chain-service adapter has the same fire-and-forget transaction write at `storage.rs:224-229`.

**Exploit scenario**

When the RocksDB write fails—for example because the state filesystem becomes full or read-only—the storage service logs `Storage request failed`, but the mempool has already received `Ok(())` from the relay send and records the transaction as pending. `status()` reports `Pending`, while a subsequent `view()` or prefix lookup cannot load the transaction. A proposal referencing that transaction consequently reaches `resolve_reference` with zero candidates and returns `UnresolvedReference`, incrementing `consensus_observe_proposal_missing_txs`. The direct adapter failure test passes, but it does not cover this relay/backend timing boundary.

**Recommendation**

- *Short term*: add a oneshot `Sender<Result<(), String>>` to `StoreTransactions`, send the backend result to the caller, and have both transaction-storage adapters await it. Only update the mempool indexes after backend success; leave the key unknown on failure.
- *Long term*: test the full storage-service path with a backend that rejects writes and assert that add, status, view, and proposal reconstruction agree. Keep the write acknowledgement contract explicit for every persistence operation.

**References**: issue `#203`; parent `#24`; related open issue `#81` (backend durability/runtime behavior); #79 covers a separate storage-recovery panic path.

### LB-002 · Transaction read failures are silently converted to missing items

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Error Reporting |
| Target | `services/storage/src/api/backend/rocksdb/chain.rs:203-230`; `services/tx-service/src/storage/adapters/rocksdb.rs:66-93`; `services/chain/chain-service/src/storage/adapters/storage.rs:232-250` |
| Status | Open |

**Description**

`GetTransactions` returns a stream whose item type is only the stored byte value; it cannot carry a per-key read error. `RocksBackend::get_transactions` logs a database error and yields `None` for that key, exactly as it does for a genuinely absent key. The tx-service and chain-service adapters then apply `from_bytes(...).ok()` and discard decode failures as well. Thus backend unreadability, absent data, and corrupt data all appear to callers as a shorter successful stream.

**Exploit scenario**

For a key still present in `pending_items`, a failed read or invalid stored encoding is omitted from `view()` and prefix resolution. A proposal reconstruction sees no candidate and returns `UnresolvedReference` rather than a storage error, so operators cannot distinguish a missing transaction from a database or integrity failure. The same ambiguity affects any chain-service caller that expects all requested transactions to be returned. This is primarily a liveness/diagnostic failure; no direct remote trigger was established.

**Recommendation**

- *Short term*: change the read contract to preserve errors—either fail the whole request on any backend/decode error or return `Stream<Item = Result<Tx, StorageError>>`—and make callers handle absence separately from unreadability.
- *Long term*: add tests for missing, backend-error, and malformed-byte cases, asserting distinct outcomes and metrics. Include an integrity/repair signal rather than treating corrupt persistence as a normal cache miss.

**References**: issue `#203`; `services/chain/chain-network/src/lib.rs:1128-1131`; #79's separate `StorageReplyReceiver` recovery panic.

