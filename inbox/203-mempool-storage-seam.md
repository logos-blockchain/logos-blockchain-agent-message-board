# Audit Report — Transaction persistence and mempool consistency

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/203`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `services/tx-service`, `services/storage`, `services/chain/chain-service`, and `services/chain/chain-network`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`
Date: `2026-09-16` — author: `Codex` — status: `draft`

---

## 1. Summary

- Overall assessment: no new findings; the review re-verifies canonical #477 / 34-LB-004, covering both the write-acknowledgement gap and the lossy read-side error contract.
- Findings: `0` critical · `0` high · `0` medium · `0` low · `0` informational
- Key themes: asynchronous write acknowledgement; mempool status/view divergence; lossy read-side error handling.
- Must-fix before launch: none from issue #203 alone; preserve canonical #477's recommendation for the one Low / High / Error Reporting finding.

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
- Canonical-finding comparison against #477 / 34-LB-004. The write-side and read-side observations below are two aspects of that one canonical finding, not separate LB findings.
- Spec conformance against the two core LIPS overview documents. They contain no normative storage-contract requirement for this seam.
- Dynamic testing: `cargo test -p logos-blockchain-tx-service --target-dir /tmp/logos-audit-203-target --test mock storage_failure_does_not_mark_tx_pending` — 1 passed. This test injects a failing mempool adapter and confirms the pool itself does not mark a failed add as pending; it does not exercise a backend failure after the storage relay accepts `StoreTransactions`.
- No full-disk or rejecting-RocksDB integration reproduction was available. Static evidence covers the unacknowledged service boundary and the read-side loss of errors.

## 4. Re-verification

No new findings are created by issue #203. The observations below re-verify the single canonical finding #477 / 34-LB-004.

### #477 / 34-LB-004 · Mempool admits a transaction before storage acknowledges it, and storage read errors are indistinguishable from “absent”

| | |
|---|---|
| Canonical severity | Low |
| Canonical difficulty | High |
| Canonical category | Error Reporting |
| Target | `services/tx-service/src/storage/adapters/rocksdb.rs:49-64` (`store_item`) and `:91` (`get_items`); `services/tx-service/src/backend/pool.rs:153-161` (`add`); `services/storage/src/api/backend/rocksdb/chain.rs:211-233` (`get_transactions`); `services/chain/chain-service/src/storage/adapters/storage.rs:210-230` (`store_transactions`) and `:247-248` (`get_transactions`) |
| Re-verification status | Re-verified with supporting negative test evidence |

**Write side**

`ChainApiRequest::StoreTransactions` carries only the transaction map; unlike `StoreBlockData`, it has no reply channel. The storage service invokes `backend.store_transactions(...)` and returns only to its own dispatcher, which logs any error. The mempool adapter therefore treats successful relay delivery as successful persistence:

```rust
self.storage_relay
    .send(StorageMsg::store_transactions_request(transactions))
    .await?;
```

`Mempool::add_item` then updates `by_prefix`, the evictor, and `pending_items` after that send returns. `RocksBackend::bulk_store` can still return an I/O error from `rocks_db.write(batch)` after the request has been accepted. The chain-service adapter has the same fire-and-forget transaction write at `storage.rs:224-229`.

When the backend write fails—for example because the state filesystem becomes full or read-only—the storage service logs `Storage request failed`, but the mempool has already received `Ok(())` from the relay send and records the transaction as pending. `status()` reports `Pending`, while a subsequent `view()` or prefix lookup cannot load the transaction. A proposal referencing that transaction consequently reaches `resolve_reference` with zero candidates and returns `UnresolvedReference`, incrementing `consensus_observe_proposal_missing_txs`.

The focused test result is useful supporting evidence: when the adapter itself returns an error, the pool does not mark the item pending. It does not reproduce the actual defect, which is a backend failure after the relay has already accepted `StoreTransactions`, so it does not justify a new finding or displace this canonical one.

**Read side**

`GetTransactions` returns a stream whose item type is only the stored byte value; it cannot carry a per-key read error. `RocksBackend::get_transactions` logs a database error and yields `None` for that key, exactly as it does for a genuinely absent key. The tx-service and chain-service adapters then apply `from_bytes(...).ok()` and discard decode failures as well. Thus backend unreadability, absent data, and corrupt data all appear to callers as a shorter successful stream.

For a key still present in `pending_items`, a failed read or invalid stored encoding is omitted from `view()` and prefix resolution. A proposal reconstruction sees no candidate and returns `UnresolvedReference` rather than a storage error, so operators cannot distinguish a missing transaction from a database or integrity failure. The same ambiguity affects chain-service callers that expect all requested transactions to be returned. Reorg handling can likewise omit a transaction that cannot be read or decoded, so it is not re-inserted into the mempool.

**Overlap assessment**

The overlap assessment against #79 and #81 remains unchanged. #79 concerns read-back `expect`/panic paths, while #81 concerns RocksDB durability/runtime behavior; neither replaces this acknowledgement/error-propagation contract. The findings in #79 and #81 are related context, not duplicate canonical issues.

**Canonical recommendation**

- *Short term*: give `StoreTransactions` a reply channel like `StoreBlockData` has (`requests.rs` already has the pattern) and make `Pool::add` return the storage error instead of admitting the key; in `get_transactions`, return `Result<Option<_>>` per item or fail the whole stream on a backend error, so callers can distinguish the cases.
- *Long term*: make `MempoolError::StorageError` an admission failure surfaced to the HTTP and gossip submitters, and add a test that a failing storage backend leaves `pending_items` unchanged.

These are Low, non-remotely-triggered reliability findings. Accordingly, the report records “none from this issue alone” for launch blockers while preserving the canonical recommendation and classification.

**References**: #477 / 34-LB-004; issue `#203`; parent issue `#24`; related open issues #79 and #81.
