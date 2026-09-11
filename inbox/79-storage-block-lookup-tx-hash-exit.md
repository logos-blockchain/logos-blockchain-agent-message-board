# Audit Report — Storage: reproduce the block-lookup-of-a-tx-hash exit; audit every expect/unwrap on bytes read back from RocksDB

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/79`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `services/storage`, `services/api` (HTTP storage adapter), `services/tx-service` (mempool storage adapter), `services/chain/chain-service` (storage adapter, sync block provider), `nodes/node/binary` (block/transaction handlers, panic hook)
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md` (both in full; no specification covers the storage service, per parent #15 and this issue)
Date: `2026-09-12` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: the process exit described but not executed in PR #78 (LB-001) and PR #126 (LB-004) is real and reproducible. One unauthenticated `GET /cryptarchia/blocks/<tx hash>` for any transaction currently in the mempool terminates the whole node. I reproduced it two ways: a storage-service unit test that fires the exact `expect`, and an end-to-end test against a running release node that observes the process dying.
- Findings: 0 critical · 0 high · 1 medium · 0 low · 0 informational
- Key themes: a single `expect` on data read back from RocksDB (`services/storage/src/lib.rs:98`); the block and transaction key spaces alias because both use the raw 32-byte hash; the panic hook turns any handler panic into `std::process::exit(1)`.
- Must-fix before launch: LB-001 (delete the `expect` in `StorageReplyReceiver::recv`; it alone removes the crash).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/storage/src/lib.rs` | `StorageReplyReceiver::recv`, `StorageMsg`, `handle_load` |
| `services/storage/src/api/backend/rocksdb/chain.rs` | key scheme for blocks and transactions |
| `services/storage/src/backends/rocksdb.rs` | `load`, `bulk_store`, blocking-join site |
| `services/storage/src/recovery.rs` | recovery-state decode path |
| `services/api/src/http/storage/adapters/rocksdb.rs` | HTTP `get_block` / `get_transactions` adapters |
| `services/tx-service/src/storage/adapters/rocksdb.rs`, `src/backend/pool.rs` | mempool persistence, TTL and grace period |
| `services/chain/chain-service/src/storage/adapters/storage.rs`, `src/sync/block_provider.rs` | typed storage requests, block loads |
| `nodes/node/binary/src/api/handlers.rs`, `src/api/routes.rs`, `src/panic.rs`, `src/lib.rs` | block/transaction handlers, panic hook, hook install |

**Out of scope**: RocksDB internals, crash consistency of write batches (covered by parent #15 / PR #78), wire-codec correctness beyond the block-vs-transaction decode. Third-party crates assumed correct: `rocksdb`, `bincode`, `serde`, `axum`, `hyper`, `reqwest`, `tokio`.

**Assumptions**: specs at the commit above are correct; the node binds its HTTP API to localhost by default and has no authentication layer (issue #66 / PR #118), so an operator who exposes it exposes a fully-trusted surface. Repo-level facts from issue #19: `expect_used`, `unwrap_used`, and `panic` are allow-listed in `Cargo.toml` lints, so these patterns are not surfaced by clippy and were checked by hand.

## 3. Method

- Worked issue #79 (all three checklist items) and read parent #15 and issue #19.
- Read both prior reports on this line and built on them rather than repeating: PR #78 (`report/63-storage`, LB-001) and PR #126 (`report/29-panic-sweep`) plus its second pass (`report/29-panic-sweep-second-pass`, LB-004). Both derived this exit from the code; neither executed it. This report executes it.
- Item 1 (reproduce): wrote two tests in a private copy of the node at the target commit.
  - A storage-service integration test driving the exact `StorageMsg` sequence the node uses (`services/storage/tests/issue79_block_lookup_of_tx_hash.rs`). Run: `cargo test -p logos-blockchain-storage-service --features rocksdb-backend`. Result: `block_lookup_of_a_pending_tx_hash_panics_in_storage_reply_receiver` panics at `services/storage/src/lib.rs:98:48` with `Recovery from storage should never fail: Deserialize(Custom("Invalid version [73]"))`; the control test with an unknown key returns `None` and does not panic.
  - An end-to-end test against a running node (`tests/src/tests/mantle/issue79_block_lookup_exit.rs`), built release with `--features testing`. It starts a one-node cluster, submits a real transfer over HTTP (which the mempool writes to RocksDB), confirms the node answers `consensus_info`, then issues `GET /cryptarchia/blocks/<tx hash>`. Result: that request returns a transport error (`hyper::Error(IncompleteMessage)` — the connection is dropped as the process exits), and every subsequent `consensus_info` call fails; the test asserts the node never recovers, and passes. `cargo test` toolchain: rustc/cargo 1.98.1.
- Item 2 (audit expect/unwrap on RocksDB read-back): enumerated by hand every `.expect(` / `.unwrap(` in the 52 non-test files that touch `StorageMsg`, `StorageChainApi`, `RocksBackend`, or the storage adapters, and classified each as reachable-from-input or not (see LB-001).
- Item 3 (other generic-`Load` callers with a caller-controlled key): grepped `new_load_message`, `StorageMsg::Load`, and `.recv::<`. Only one non-test caller exists.
- Automated tooling beyond `cargo test`: none. Dynamic testing: the e2e run above.
- The repo's `.cargo/config.toml` forces `rust-lld`, which fails on this machine; per parent guidance I removed that override in the private copy only and built with `RUSTFLAGS=""` in a separate target directory.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Block lookup of a mempool transaction hash exits the node; reproduced | Denial of Service / Data Validation | Medium | Low | Open |

### LB-001 · `Block lookup of a mempool transaction hash exits the node (reproduced)`

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Denial of Service / Data Validation |
| Target | `services/storage/src/lib.rs:98` (`StorageReplyReceiver::recv`); `services/api/src/http/storage/adapters/rocksdb.rs:46-54` (`RocksAdapter::get_block`); `services/storage/src/api/backend/rocksdb/chain.rs:39-42` (`get_block`), `:53-54` (block key), `:195-197` (transaction key); `nodes/node/binary/src/api/handlers.rs:1530` (`block`), `routes.rs:71`, `paths.rs:38`; `nodes/node/binary/src/panic.rs:35`; hook installed at `nodes/node/binary/src/lib.rs:228` |
| Status | Open |

**Description**

Blocks and transactions share one unprefixed 32-byte key space in the default column family. A block is stored under its raw header id (`chain.rs:53-54`, `store_block_data`), a transaction under its raw hash (`chain.rs:195-197`, `store_transactions`; `From<TxHash> for Bytes` at `core/src/mantle/transactions/hash.rs:69-72` is the raw 32 bytes). Every other record type carries a string prefix (`block_parent/`, `block_events/`, `immutable_block/slot/`, `recovery/`), so only these two alias.

`GET /cryptarchia/blocks/:id` parses `:id` as a `HeaderId` and loads it through the generic `Load` message:

```rust
// services/api/src/http/storage/adapters/rocksdb.rs:46-53
let key: [u8; 32] = id.into();
let (msg, receiver) = StorageMsg::new_load_message(Bytes::copy_from_slice(&key));
storage_relay.send(msg).await.map_err(|(e, _)| e)?;
receiver.recv().await.map_err(|e| Box::new(e) as crate::http::DynError)
```

`recv` decodes the reply with an `expect`:

```rust
// services/storage/src/lib.rs:96-99
.map(|maybe_bytes| {
    maybe_bytes.map(|bytes| {
        Output::from_bytes(&bytes).expect("Recovery from storage should never fail")
```

Here `Output` is `Block<SignedOps<Unverified, StandardMode>>`. When the 32 bytes name a transaction rather than a block, the stored value is a bincode-encoded `SignedOps`; decoding it as a `Block` fails (`reject_trailing_bytes`/version byte mismatch, `core/src/codec/bincode/mod.rs`), so the `expect` panics. The node installs `log_and_exit_hook` (`lib.rs:228`), whose last line is `std::process::exit(1)` (`panic.rs:35`), so the panic in the axum handler task exits the entire process.

The `recv` decode is the only reachable-from-input panic on data read back from RocksDB. The full classification of every `expect`/`unwrap` in files touching the storage service:

- Reachable from unauthenticated input: `services/storage/src/lib.rs:98` — this finding. It is the only one.
- Relay-send `.unwrap()`s that panic only if the storage service relay is already closed (node shutdown), not attacker-triggerable: `services/chain/chain-service/src/storage/adapters/storage.rs:82,135,149` and the sibling typed requests. These decode the returned value safely (`try_into().ok()` / `unwrap_or_else`). Noted as S-001.
- Not on a read-back path: `services/storage/src/backends/rocksdb.rs:162` (`expect` on a `spawn_blocking` join handle), relay/`NonZero`/`try_from` expects across `chain-leader`, `chain-network`, `pow`, `wallet`, and the `NonZeroUsize::new(...).expect(...)` constants in `block_provider.rs`.
- Test-only: everything under `services/storage/src/recovery.rs` `#[cfg(test)]` and the `chain.rs` test module.

Every other consumer of block/transaction bytes read from storage decodes safely and does not panic:

- `services/chain/chain-service/src/storage/adapters/storage.rs:84-86` (`get_block`): `block.try_into().ok()` → `None` on failure.
- `services/chain/chain-service/src/sync/block_provider.rs:509-529` (`load_block`): `try_into` mapped to `ConversionError`; `load_block_bytes:533-548` returns the raw bytes without decoding.
- `services/tx-service/src/storage/adapters/rocksdb.rs:91` (`get_items`): `Self::Item::from_bytes(&bytes).ok()` → filtered out on failure.

Item 3 result: the only non-test caller of `new_load_message` (a generic `Load` with a caller-controlled key) is `services/api/src/http/storage/adapters/rocksdb.rs:47`, which is exactly this path. All other storage reads use the typed `StorageChainApi` requests (`get_block_request`, `get_block_parent_request`, `get_transactions_request`, `get_immutable_block_id_request`), whose callers decode with the safe patterns above. So the crash is unique to `GET /cryptarchia/blocks/:id`.

**Exploit scenario**

A client that can reach the HTTP API submits a transaction via `POST /mempool/add/tx` (or the wallet `transfer-funds` endpoint) and reads back the transaction hash, then issues `GET /cryptarchia/blocks/<that hash>`. The storage service returns the transaction bytes, `recv` panics, and the process exits. One unauthenticated request, no special resources, repeatable on every restart while the transaction stays in RocksDB (24-hour default TTL, `pool.rs:30`; kept a further 10-minute grace period after removal, `pool.rs:23`). Reproduced end to end: the malicious request drops the connection mid-response and the node never answers again.

Rated Medium because the API binds to localhost by default and has no authentication, so an exposed API is already fully trusted; where the API is reachable by untrusted parties this is High. This finding reproduces and confirms PR #78 LB-001 and PR #126 LB-004, which described the exit from the code without executing it.

**Recommendation**
- *Short term*: honour the existing `// TODO` in `recv` — make it return `Result<Option<Output>, _>` with the decode error propagated, delete the `expect`, and map the error to HTTP 500 in `RocksAdapter::get_block`. This alone removes the crash.
- *Long term*: prefix block keys (`block/`) and transaction keys (`tx/`) in `chain.rs` so the two record types cannot alias (matches PR #78 LB-001), and version stored values (issue #181) so a schema mismatch is a typed error at open time. Consider linting for any `expect`/`unwrap` applied to data read back from storage, since issue #19 confirms these patterns are otherwise silent.

**References**: parent #15; issue #19 (`expect_used`/`panic` allow-listed); PR #78 (`report/63-storage`) LB-001; PR #126 (`report/29-panic-sweep`) and its second pass LB-004; issue #181 (storage schema versioning).

## 5. Suggestions (non-security)

### S-001 · Storage-relay `send().await.unwrap()` panics on relay closure

`services/chain/chain-service/src/storage/adapters/storage.rs:82,135,149` (and the other typed request methods) call `.send(...).await.unwrap()` on the storage relay. If the storage service has shut down or its inbound relay is closed, these panic instead of returning `None`/an error, and — through the same exit hook — take the process down. Not attacker-triggerable through input, but a fail-hard during shutdown or service-restart races. Return `None` or a typed error, as the surrounding methods already do for the receive half.

### S-002 · `GET /cryptarchia/transaction/:id` decodes bincode-stored bytes with `serde_json`

`services/api/src/http/storage/adapters/rocksdb.rs:78` decodes transactions with `serde_json::from_slice`, but the mempool writes them with the bincode wire codec (`services/tx-service/src/storage/adapters/rocksdb.rs:51`, `Tx::to_bytes`). The decode always fails, so the endpoint returns 500 for any stored transaction. This is a functional bug, not a crash (the error is mapped, not `expect`ed), and overlaps the note in PR #78's appendix; it belongs to the API area (issue #18 / #66). Decode with the same codec used to store, or route the read through the typed `StorageChainApi::get_transactions`.
