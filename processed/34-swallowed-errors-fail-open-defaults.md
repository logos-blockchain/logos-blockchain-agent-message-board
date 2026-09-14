# Audit Report — Sweep: swallowed errors and fail-open defaults on validation paths

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/34`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): workspace-wide sweep; findings land in `nodes/node/binary` (config, HTTP API), `services/tx-service`, `services/storage`, `services/chain/*`, `blend/network`, `consensus/cryptarchia-sync`, `c-bindings`
Date: `2026-09-12` — author: `claude-fable-5.1` — status: `final`

Specifications: no specification covers this sweep (parent #24). `logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` was the head at the time of writing; only the two overviews named in #24 were consulted.

---

## 1. Summary

- Overall assessment: the consensus-critical validation paths (ledger, `core/src/mantle`, block and transaction preverification, blend message verification) fail closed; every `map_err(|_| …)` there maps to a typed rejection and no `Err` arm on those paths is turned into `Ok`, a default, or a `continue`. The fail-open patterns are one layer out: operator-facing configuration defaults that silently disable a safety limit or a consensus parameter, a mempool/storage seam where a write is assumed to have happened and a read error is indistinguishable from "absent", and error paths that log and carry on where the surrounding code expects an abort.
- Findings: 0 critical · 0 high · 0 medium · 6 low · 4 informational
- Key themes: `#[serde(default)]` on values that are limits or consensus inputs; storage errors flattened to `None`/empty streams; asymmetric spam handling between epoch paths in blend; internal error text forwarded to remote peers and HTTP clients.
- Must-fix before launch: none; LB-001 and LB-002 are the two worth doing before a public deployment configuration is frozen.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| whole workspace (excluding `tests/`, `zone-sdk/`, `logos_sql/`, `tools/`, `deployment/`) | grep-driven sweep for `let _ =`, `.ok()`, `unwrap_or_default()`, `unwrap_or(`, `map_err(\|_\|`, `if let Ok` / `let Ok … else`, `Err(_) =>`, `is_none_or`, `map_or(true/false`, `#[serde(default)]` |
| `ledger/src`, `consensus/`, `core/src` | every `Err` arm and `.is_ok()/.is_err()` outside tests read in context |
| `services/chain/{chain-service,chain-network,chain-leader}` | every `Err` arm, `if let Err`, `let Ok … else` outside tests |
| `services/tx-service`, `services/storage`, `services/network`, `services/blend`, `blend/*`, `services/sdp`, `services/pow`, `services/time`, `services/key-management-system`, `kms/*` | `Err` arms filtered for swallow shapes (`continue`, `return;`, `None`, `Ok(())`, `true/false`) |
| `nodes/node/binary/src/config/**`, `libp2p/src/config/**`, `services/*/…/config.rs` | every `#[serde(default)]` read for what the default disables |
| `nodes/node/binary/src/api/{errors,handlers,responses}`, `services/api/src/http` | error `Display` reaching HTTP clients |
| `consensus/cryptarchia-sync/src/libp2p/{provider,downloader,behaviour}.rs`, `services/chain/chain-service/src/sync/block_provider.rs` | error `Display` reaching remote peers |
| error enums in `kms/`, `services/key-management-system`, `wallet/`, `services/wallet`, `blend/crypto`, `core/src/crypto.rs` | `#[error(…)]` format strings checked for secret-bearing fields |

**Out of scope**

`tests/`, `zone-sdk/`, `logos_sql/`, `tools/`, `deployment/`, `wallet-http-client/` and `tracing/` were only grepped, not read; the `unwrap_or_default` hits there (roughly 60 of the 70 in the workspace) are in test harnesses, diagnostics and the zone sequencer and were not assessed. Panics on these paths are PR #126's subject and are not repeated here. Mempool admission ordering is PR #179 / #113; gossipsub forward-before-validate is #144; the rejected-block cache is #143 / #184; NTP fallback behaviour is #46. Third-party crates assumed correct: `serde`, `serde_ignored`, `serde_yaml`, `libp2p` (gossipsub, request-response), `rocksdb`, `tokio`, `thiserror`.

**Assumptions**

The repo-level facts in #19 hold at this commit and were re-verified: `Cargo.toml:308-415` allows `map_err_ignore`, `let_underscore_must_use`, `let_underscore_untyped`, `unwrap_used`, `expect_used`, `wildcard_enum_match_arm`; `Cargo.toml:416-421` allows `unused_results`; `let_underscore_drop` is `warn` (`Cargo.toml:429`). So none of the patterns in this sweep are caught by CI and each had to be found by hand. Release profile (`Cargo.toml:11-19`) still does not enable overflow checks. Toolchain `1.98.1` (`rust-toolchain.toml:11`).

## 3. Method

- Manual review of the in-scope paths, working through issue #34 (both checklist items) with #19 and #24 as context.
- Automated tooling: `rg 14.x` for the pattern inventory listed under Scope; no cargo tooling was run (the sweep is static).
- Dynamic testing: none.
- Every claim below was read in context at the stated commit; line numbers are at that commit.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Leader-claim and SDP fee caps default to `Value::MAX` when the key is omitted | Configuration | Low | Low | Open |
| LB-002 | `faucet_pk` silently defaults to `None`, changing genesis `total_stake` and leader eligibility on the node that omits it | Consensus | Low | High | Open |
| LB-003 | Blend: invalid messages on the old-epoch path close substreams but never mark the peer spammy | Denial of Service | Low | Medium | Open |
| LB-004 | Mempool admits a transaction before storage acknowledges it, and storage read errors are indistinguishable from "absent" | Error Reporting | Low | High | Open |
| LB-005 | Chain recovery replay continues past a block that fails to apply | Error Reporting | Low | High | Open |
| LB-006 | C bindings drop malformed host-supplied peers, HTTP address and external address without an error | Data Validation | Low | Low | Open |
| LB-007 | Block-sync provider forwards internal error `Display`/`Debug` text to the requesting peer | Data Exposure | Informational | Low | Open |
| LB-008 | HTTP 500 bodies carry the raw internal error and NDJSON streams end silently on error | Error Reporting | Informational | Low | Open |
| LB-009 | The whole `network` section is optional; an omitted `node_key` yields a fresh identity per start | Configuration | Informational | Low | Open |
| LB-010 | `should_process_block` / `is_after_lib` fail open when the chain-service query errors | Error Reporting | Informational | High | Open |

### LB-001 · Leader-claim and SDP fee caps default to `Value::MAX` when the key is omitted

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Configuration |
| Target | `nodes/node/binary/src/config/cryptarchia/serde/leader.rs:13-24`, `nodes/node/binary/src/config/sdp/serde.rs:23-24`; enforced at `services/wallet/src/lib.rs:1441-1447` and `services/sdp/src/wallet.rs:21-31` |
| Status | Open |

**Description**

Both wallets that spend on the operator's behalf without a human in the loop carry a "hard cap" on the transaction fee:

```rust
// nodes/node/binary/src/config/cryptarchia/serde/leader.rs:11-24
pub struct WalletConfig {
    // Hard cap on the ransaction fee for LEADER_CLAIM
    #[serde(default = "default_max_tx_fee")]
    pub max_tx_fee: GasCost,
    ...
}
pub const fn default_max_tx_fee() -> GasCost {
    GasCost::new(Value::MAX)
}
```

`nodes/node/binary/src/config/sdp/serde.rs:23-24` uses the same default for SDP declare/withdraw/active transactions. The only place the cap is checked is the comparison `tx_fee > request.max_tx_fee` (`services/wallet/src/lib.rs:1442`, and the SDP equivalent), so an omitted key is a cap that can never trigger. The field is documented as a hard cap, and the config loader rejects *unknown* keys (`OnUnknownKeys::Fail`, `nodes/node/binary/src/main.rs:58-81`) but a *missing* key is filled in silently, so a typo in the key name is the one case that is reported while forgetting it entirely is not.

**Exploit scenario**

Not attacker-triggered directly. The fee the wallet pays is `minimum_gas_cost` at the current ledger gas prices plus the priority reserve (`services/wallet/src/lib.rs:1430-1441`). A storage-market price step, a mis-set `priority_fee_percent`, or a builder bug in gas accounting produces a leader-claim or SDP transaction that pays whatever the arithmetic says, out of `funding_pk`, with nothing to stop it. The operator who intended to be protected by the documented cap finds out from their balance.

**Recommendation**
- *Short term*: remove the `default = "default_max_tx_fee"` on both fields so the key is required, or make the default a small multiple of the mandatory fee at genesis prices. Either way, log the effective cap at startup.
- *Long term*: a policy for `#[serde(default)]` in `nodes/node/binary/src/config`: a default is acceptable for tuning knobs, not for anything that is a limit, a key, or a consensus parameter. LB-002 and LB-009 are the other instances.

### LB-002 · `faucet_pk` silently defaults to `None`, changing genesis `total_stake` and leader eligibility on the node that omits it

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Consensus |
| Target | `nodes/node/binary/src/config/cryptarchia/deployment.rs:30-31`, `ledger/src/config.rs:15-16`; consumed at `ledger/src/cryptarchia/mod.rs:743-749` and `services/chain/chain-leader/src/leadership.rs:442-449` |
| Status | Open |

**Description**

`faucet_pk` is a deployment-level consensus input: at genesis the ledger excludes the faucet's notes from `total_stake`, which feeds the lottery thresholds,

```rust
// ledger/src/cryptarchia/mod.rs:743-752
let total_stake = utxos
    .utxos()
    .iter()
    .filter(|(_, (utxo, _))| config.faucet_pk.is_none_or(|fpk| utxo.note.pk != fpk))
    .map(|(_, (utxo, _))| utxo.note.value)
    .sum::<Value>()
    .max(1);
let (lottery_0, lottery_1) = config.lottery_constants().compute_lottery_values(total_stake);
```

and the leader excludes them from its eligible set (`leadership.rs:442-449`). Both the deployment `Settings` (`deployment.rs:30-31`) and the ledger `Config` (`config.rs:15-16`) carry `#[serde(default)]` on the field, and the value is not part of the genesis inscription, so it is not covered by the genesis hash. A deployment file that omits it parses without any diagnostic and produces a node whose `lottery_0`/`lottery_1` differ from every node that has it set. The tooling that writes the deployment file does patch the key in (`tools/blockchain-tools/src/bin/genesis.rs:260-261, 396-399`), and the release binary embeds its deployment (`nodes/node/binary/src/config/deployment/mod.rs:47`), which is why this is rated Low; hand-edited or third-party deployment files are the exposure. #175 (bind and validate the consensus schedule parameters) and #177 (restored LIB state not validated against the running config) are the same class of problem for the other parameters; this one is called out because the default is silent rather than merely unvalidated.

**Exploit scenario**

An operator bootstraps from a deployment file that predates the field or was written by hand without it. Their node computes a larger `total_stake`, so it applies stricter thresholds than the network and rejects some valid leadership proofs (`LedgerError`) while accepting its own faucet-funded notes as eligible. It forks off silently; its wallet and API users see a different chain. No attacker is needed, but a deployment file distributed with the key stripped would do the same to every node that uses it.

**Recommendation**
- *Short term*: drop `#[serde(default)]` on `deployment.rs:30` so a deployment file must state `faucet_pk: null` explicitly, and log its value in the startup banner next to the chain ID.
- *Long term*: move the faucet exclusion into the genesis inscription (or fold `faucet_pk` into whatever #175 binds into the genesis hash) so the value cannot differ between nodes that agree on genesis.

### LB-003 · Blend: invalid messages on the old-epoch path close substreams but never mark the peer spammy

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | `blend/network/src/core/with_core/behaviour/mod.rs:1011-1045` (`handle_received_serialized_encapsulated_message`), `blend/network/src/core/with_core/behaviour/old_epoch.rs:245-275` |
| Status | Open |

**Description**

A received blend message is first offered to the old epoch, then to the current one. The two error arms are not symmetric:

```rust
// blend/network/src/core/with_core/behaviour/mod.rs:1011-1045
if let Some(old_epoch) = &mut self.old_epoch {
    match old_epoch.handle_received_serialized_encapsulated_message(...) {
        Ok(handled) => { if handled { return; } }
        Err(_) => { return; }                                  // <- no spam accounting
    }
}
if let Err(receive_error) = handle_received_serialized_encapsulated_message_and_update_cache(...) {
    let spam_reason = match receive_error { ... };
    self.close_spammy_connection((from_peer_id, from_connection_id), spam_reason);   // <- current epoch
}
```

`close_spammy_connection` (`mod.rs:729-740`) records the connection as spammy (`set_connection_to_spammy`) before closing it, which is what feeds the blocklist and failure detector that #101, #117 and #139 are about. The old-epoch path (`old_epoch.rs:264-272`) only pushes `FromBehaviour::CloseSubstreams` and returns `Err`, which the caller discards with `Err(_) => return`. The same three `ReceiveError` variants (duplicate, bad header signature, undeserialisable) that earn a spam verdict on the current-epoch path earn nothing on the old-epoch path, and the error itself is dropped without a log at the caller.

**Exploit scenario**

A core peer that was negotiated in epoch *e* keeps its connection through the transition into *e+1*. During the grace window it sends malformed or replayed messages on that connection. `is_negotiated` (`old_epoch.rs:251`) routes them to the old-epoch handler, whose substreams get closed but whose peer is never marked spammy, so it is not blocked and can reconnect and repeat, whereas the identical bytes one epoch earlier would have got it blocked. The window is one epoch transition, so this is a per-transition nuisance rather than a sustained attack.

**Recommendation**
- *Short term*: in `mod.rs:1022-1024`, map the `ReceiveError` to a `SpamReason` and call `close_spammy_connection` exactly as the current-epoch arm does; have `old_epoch.rs:264-272` stop closing substreams itself so the two paths share one exit.
- *Long term*: make `ReceiveError` handling a single function called by both epoch handlers, so a future variant cannot be treated differently by accident.

### LB-004 · Mempool admits a transaction before storage acknowledges it, and storage read errors are indistinguishable from "absent"

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Error Reporting |
| Target | `services/tx-service/src/storage/adapters/rocksdb.rs:49-64` (`store_item`) and `:91` (`get_items`); `services/tx-service/src/backend/pool.rs:153-161` (`add`); `services/storage/src/api/backend/rocksdb/chain.rs:211-233` (`get_transactions`); `services/chain/chain-service/src/storage/adapters/storage.rs:210-230` (`store_transactions`) and `:247-248` (`get_transactions`) |
| Status | Open |

**Description**

Write side: `store_item` sends `StorageMsg::store_transactions_request` and returns `Ok(())` as soon as the relay accepted the message (`rocksdb.rs:58-64`); the request carries no reply channel (`services/storage/src/api/chain/requests.rs:67-69`), so a backend write failure is only ever logged by the storage service (`services/storage/src/lib.rs:198-200`). `Pool::add` then inserts the key into `pending_items` and the prefix index unconditionally (`pool.rs:153-161`). The chain-service adapter has the same shape (`storage.rs:224-229`).

Read side: the RocksDB backend turns a database error into `None` with an `error!` (`chain.rs:223-229`) and a decode failure into a dropped stream item (`tx-service/…/rocksdb.rs:91`, `chain-service/…/storage.rs:248`, both `filter_map(… .ok())`). A consumer cannot tell "not stored" from "stored but unreadable".

Consequences of the two together, all verified in code:

- `status()` reports the key `Pending` (`pool.rs:234-240`) while `view()` (`pool.rs:180-182`, used for block building) and `get_transactions_by_prefix` (`services/tx-service/src/tx/service.rs:486-495`) omit the transaction. Proposal reconstruction then fails with `UnresolvedReference` (`services/chain/chain-network/src/lib.rs:1128-1131`), which is deliberately not recorded against the block (`lib.rs:635-640`), so the node simply cannot apply any proposal that references that transaction until it is evicted.
- Reorg handling collects reorged transactions via `get_block(id)` (`services/chain/chain-service/src/service/mod.rs:893-903`), where a decode failure or relay error is `None` (`storage.rs:84-90`) and is `flatten()`ed away, so those transactions are silently not re-inserted into the mempool.

Each individual `None` is logged, so this is Error Reporting rather than a correctness bug; the point is that the log is the only trace and nothing upstream changes behaviour.

**Exploit scenario**

Not remotely triggerable. Disk-full, a RocksDB corruption, or a codec change that leaves older stored bytes undecodable makes the node advertise transactions it cannot produce and drop reorged transactions on the floor, with only per-item `error!` lines to show for it. If the affected transaction is one the network's leaders keep including, this node stops following proposals and falls back to full-block sync for every block.

**Recommendation**
- *Short term*: give `StoreTransactions` a reply channel like `StoreBlockData` has (`requests.rs` already has the pattern) and make `Pool::add` return the storage error instead of admitting the key; in `get_transactions` return `Result<Option<_>>` per item or fail the whole stream on a backend error, so callers can distinguish the cases.
- *Long term*: make `MempoolError::StorageError` an admission failure surfaced to the HTTP and gossip submitters, and add a test that a failing storage backend leaves `pending_items` unchanged.

### LB-005 · Chain recovery replay continues past a block that fails to apply

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Error Reporting |
| Target | `services/chain/chain-service/src/lib.rs:1035-1057`; `services/chain/chain-service/src/service/mod.rs:1235-1251` (`persist_recovery_state`) |
| Status | Open |

**Description**

On startup the service replays the stored blocks in `(LIB, tip]`:

```rust
// services/chain/chain-service/src/lib.rs:1038-1056
for (i, block) in blocks.into_iter().enumerate() {
    match process_block(&mut cryptarchia, block, ...).await {
        Ok(outcome) => { ...; pruned_blocks.extend(&outcome.pruned_blocks); }
        Err(e) => {
            error!(target: LOG_TARGET, "Error processing block: {:?}", e);
        }
    }
}
```

The loader is careful when a block is *missing* (`load_recovery_blocks_or_fall_back_to_lib`, `lib.rs:913-`, falls back to LIB), but when a block is present and fails to apply the loop logs and moves to the next one, whose parent is now missing, and so on to the end. The node then reports itself initialised at whatever prefix did apply, with the stored recovery state still naming the original tip until the next `persist_recovery_state`, which itself swallows its own failure (`mod.rs:1247-1249`). Recovery is a place where the code already has a fail-closed alternative (fall back to LIB, or refuse to start as `lib.rs:942-945` does for a storage load error) and does not take it.

**Exploit scenario**

Not remotely triggerable. A node restarted after a configuration change that alters validation (see #177), or with a stored block that decodes but no longer validates, starts on a truncated branch with a stale recovery record, and the operator learns of it from `Error processing block` lines rather than a refusal to start. If the truncation crosses an epoch boundary the node's epoch state differs from the network's until it re-syncs.

**Recommendation**
- *Short term*: on the first `Err` in the loop, discard the partial replay and take the existing fall-back-to-LIB path, or abort startup with the error, mirroring `lib.rs:942-945`.
- *Long term*: persist the recovery state only after a successful full replay, so a stale record cannot point past what the node actually holds.

### LB-006 · C bindings drop malformed host-supplied peers, HTTP address and external address without an error

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Data Validation |
| Target | `c-bindings/src/api/config.rs:55-67, 88-93, 95-99` |
| Status | Open |

**Description**

The FFI config conversion silently discards values it cannot parse:

```rust
// c-bindings/src/api/config.rs:57-66
.filter_map(|&pointer| {
    if pointer.is_null() { return None; }
    unsafe { CStr::from_ptr(pointer) }.to_str().ok()
        .and_then(|string| Multiaddr::from_str(string).ok())
})
// :90-92  if let Ok(addr) = http_address.to_string_lossy().parse() { init_args.http_addr = addr; }
// :98     init_args.external_address = external_address.to_string_lossy().parse().ok();
```

A peer list with one bad entry loses that entry; a bad `http_addr` keeps the default (`127.0.0.1:8080` per `nodes/node/binary/src/config/api/serde.rs:34-40`); a bad `external_address` becomes "none". The host receives `OK`. The native CLI path for the same values (`nodes/node/binary/src/cli/config/init.rs`) goes through clap and rejects them. PR #94 (the #67 report) covered ABI shape and the ZK key hex parser (its LB-007) but not these three.

**Exploit scenario**

A mobile host passes its bootstrap list with one mistyped multiaddr and an `http_addr` meant to bind a Unix-socket style path. The node starts with fewer peers than intended and its HTTP API bound to the default loopback port rather than where the host expects it. Nothing in the return value says so.

**Recommendation**
- *Short term*: return an error status from the config conversion on the first unparsable value, naming the field; the FFI already has an error-code convention for `init`.
- *Long term*: share the clap value parsers between the CLI and the FFI so the two paths cannot diverge.

### LB-007 · Block-sync provider forwards internal error `Display`/`Debug` text to the requesting peer

| | |
|---|---|
| Severity | Informational |
| Difficulty | Low |
| Category | Data Exposure |
| Target | `services/chain/chain-service/src/sync/block_provider.rs:106-124`; `consensus/cryptarchia-sync/src/libp2p/provider.rs:95-121` |
| Status | Open |

**Description**

This is the checklist's second item. On a failed `DownloadBlocks` request the provider sends the remote peer

```rust
// services/chain/chain-service/src/sync/block_provider.rs:107-121
BlocksUnavailableReason::Unknown(format!("Failed to create a block stream: {e:?}"))   // any DynError, Debug-formatted
...
other => BlocksUnavailableReason::Unknown(other.to_string()),                          // GetBlocksError::{InvalidState(String), SendError(String), ChannelDropped, ConversionError}
```

and on a mid-stream failure `provider.rs:116-118` sends `ChainSyncError::to_string()`, which embeds `"Failed to receive block from stream: {e}"` for whatever the storage stream returned. `GetTipResponse::Failure(reason)` (`provider.rs:54-57`) carries a `String` from the same family. The strings are the node's internal error chain, including RocksDB error text when the backend is the source, `Debug`-formatted. No secret was found in any of these types, and the API-facing variant in `services/chain/chain-service/src/service/mod.rs:1259` is a fixed string, so this is informational: it discloses backend and version fingerprints to an unauthenticated peer, nothing more.

**Exploit scenario**

A peer issues `DownloadBlocks` for header IDs it knows are absent, or times its request to a node under storage pressure, and reads the backend's error text to fingerprint the implementation and its state. No further impact identified.

**Recommendation**
- *Short term*: make `BlocksUnavailableReason::Unknown` carry a fixed message (or an enum code) on the wire and keep the formatted error in the local log line that already exists at `provider.rs:115`.
- *Long term*: a `#[serde]`-level rule that wire-format error enums do not contain `String` payloads derived from `Display` of internal errors.

### LB-008 · HTTP 500 bodies carry the raw internal error and NDJSON streams end silently on error

| | |
|---|---|
| Severity | Informational |
| Difficulty | Low |
| Category | Error Reporting |
| Target | `nodes/node/binary/src/api/errors.rs:61-63`; `nodes/node/binary/src/api/responses/ndjson.rs:30-38`; `nodes/node/http-client/src/lib.rs:228-229, 485-486` |
| Status | Open |

**Description**

`ApiError::Internal(error)` responds with `error.to_string()` in the JSON body (`errors.rs:61-63`), and every `ApiError::internal(...)` site in `handlers.rs` (`:551, :556, :569, :613, :1024, :1619, :1635, :1671, :1866, :1913, :1958, :2050, :2055`) feeds it an arbitrary `DynError` from the relay, storage, wallet or mempool. The `InternalServerError` variant that hides the message exists (`errors.rs:17-18`) and is used once (`handlers.rs:1791`). Separately, `from_stream_result` (`ndjson.rs:36`) drops every `Err` item, so a streaming endpoint whose source fails mid-way returns a well-formed but truncated NDJSON body with status 200 already sent; the bundled HTTP client does the same on its side (`http-client/src/lib.rs:228-229, 485-486`), so neither end can observe the failure. PR #118 covers the authentication and CORS side of the same surface; this is the error-path side.

**Exploit scenario**

A client hits an endpoint while storage is failing and reads the RocksDB error text; a block-stream consumer misses the tail of a range and has no signal to retry. Neither leaks a secret; both make incidents harder to diagnose from the client side.

**Recommendation**
- *Short term*: route `Internal` through `InternalServerError` for the body and keep the formatted error in the server log; in `from_stream_result`, end the stream with a final NDJSON error object instead of dropping the `Err`.
- *Long term*: a typed `ApiError` per handler family so `DynError` never reaches the response layer.

### LB-009 · The whole `network` section is optional; an omitted `node_key` yields a fresh identity per start

| | |
|---|---|
| Severity | Informational |
| Difficulty | Low |
| Category | Configuration |
| Target | `nodes/node/binary/src/config/mod.rs:59-60`; `nodes/node/binary/src/config/network/serde/mod.rs:15-27`; `libp2p/src/config/mod.rs:24-26` |
| Status | Open |

**Description**

`UserConfig.network` is `#[serde(default)]` (`config/mod.rs:59`), `BackendSettings` and `SwarmConfig` are `#[serde(default)]` (`network/serde/mod.rs:16, 22, 30`), and the libp2p key defaults to `ed25519::SecretKey::generate` (`libp2p/src/config/mod.rs:25`, documented "Default: random"). A user config with no `network:` section, or with the section but no `node_key`, starts a node whose peer ID changes on every restart, with an empty `initial_peers`. Kademlia routing tables and gossipsub peer scores held by other nodes are keyed on that ID, so a restart looks like a new peer. The empty peer list at least fails loudly at IBD (`fetch_tips` → `AllPeersFailed` → shutdown, `services/chain/chain-network/src/lib.rs:328-338`), which is why this is informational; the rotating identity does not fail at all. Blend's signing identity is separate (`config/blend/serde/mod.rs:19-21`, required) and unaffected.

**Exploit scenario**

None; operational only. An operator who relies on `initial_peers` on other nodes pointing at this node's `/p2p/<id>` address finds those entries stale after each restart.

**Recommendation**
- *Short term*: log the peer ID and whether the key was generated at startup; consider requiring `node_key` when `initial_peers` is non-empty on peers that expect to be dialed.
- *Long term*: same `#[serde(default)]` policy as LB-001.

### LB-010 · `should_process_block` / `is_after_lib` fail open when the chain-service query errors

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Error Reporting |
| Target | `services/chain/chain-network/src/lib.rs:845-868, 877-908` |
| Status | Open |

**Description**

Both pre-checks on a gossiped proposal return "go ahead" when the cryptarchia query fails: `get_ledger_state` error → `Ok(())` with the comment that processing is idempotent (`lib.rs:862-866`); `info()` error → `true` (`lib.rs:903-906`). The comment is accurate for `AlreadyApplied`; for the LIB check it means a block at or before LIB is handed to reconstruction and then to `process_block`, which rejects it, and `handle_proposal_processing_error` decides whether that rejection is final. This is by design and each branch logs at `error!`; it is recorded here because the sweep asked for defaults on validation paths and this is one, and because the outcome depends on #143's answer to which apply errors condemn a block ID.

**Exploit scenario**

None on its own; the chain-service relay erroring is a local fault.

**Recommendation**
- *Short term*: none needed beyond the existing logs.
- *Long term*: once #143/#184 settle the rejected-cache policy, confirm a pre-LIB block that slipped through here cannot be recorded as rejected against its ID.

## 5. Suggestions (non-security)

### S-001 · Enable `let_underscore_must_use`; the current `let _ =` inventory is clean

The 15 non-test `let _ =` sites (`rg "let _ = " --type rust`, minus `tests/`) are all benign: three discard a `name` under `cfg` in `utils/src/tokio/mod.rs:79,107,136`, two are `oneshot::Sender::send` results whose receiver going away is expected (`logos_sql/src/runtime.rs:323`, `kms/operators/src/ed25519/exfiltrate_secret_key.rs:35-38` and `derive_x25519.rs:29`, both with a `debug!` in the `map_err`), one is a TLS counter in a test allocator (`codec/src/bounded_vec.rs:502`), two are in `#[cfg(test)]` (`ledger/src/config.rs:542,549`, `ledger/src/mantle/pow/difficulty.rs:211`) and one is a feature-gated warning (`services/chain/chain-leader/src/lib.rs:779-782`). Turning `let_underscore_must_use` from `allow` to `warn` (`Cargo.toml:347`) would cost eight `#[expect]`s and keep it that way.

### S-002 · Chain-service SDP queries answer "empty" when the tip's ledger state is missing

`Query::GetSdpDeclarations` and `Query::GetSdpSnapshot` (`services/chain/chain-service/src/service/mod.rs:345-386`) use `.unwrap_or_default()` when `ledger.state(&tip)` is `None`. The tip always has a state, so this is unreachable today, but the reply type makes "no declarations" and "no state" the same answer. Returning `Result` would keep a future bug visible.

### S-003 · Zero thresholds for unknown channels are a pricing default, not a verification one

`ThresholdSource` returns `0` for a channel that does not exist (`core/src/mantle/gas.rs:159-165`, `core/src/mantle/channel.rs:143-153`, `core/src/mantle/transactions/thresholds.rs:67-82`). This was checked because it looked like a fail-open default: it is only used to price execution gas (`ops/channel/config.rs:73-77`, `withdraw.rs:69-71`); signature verification goes through `get_channel_transfer_threshold`, which fails closed with `ChannelNotFound` (`ledger/src/mantle/helpers.rs:102-113`). The effect is that ops against a nonexistent channel are priced at zero execution gas until they are rejected, which #185 already discusses.

### S-004 · Two liveness features degrade to "off" on a startup error

The tip-poll watchdog is disabled when its parameters cannot be derived (`services/chain/chain-network/src/lib.rs:390-393`, `error!` then `None`), and the leader's per-slot context fetch (`services/chain/chain-leader/src/leadership.rs:322-326`) logs at `debug!` when the wallet or chain-service query fails and skips the slot. A leader whose wallet relay is broken produces no blocks and, at the default log level, no lines saying why. Promote the second to `warn!` with a rate limit.

### S-005 · `wallet::WalletState::channel_notes` is `#[serde(default)]`

`wallet/src/lib.rs:160-161` lets a persisted wallet state written before the field existed deserialise with an empty `channel_notes`, which is the set that gates channel-owned notes out of wallet spending. If old state files are expected in the field, a migration step that rebuilds the set from `utxos` is safer than an empty default.

**Checked and ruled out**

- `ledger/src`, `consensus/`, `core/src`: no `Err` arm outside tests becomes `Ok`, `None`, a default or `continue`; the `is_none_or` uses (`ledger/src/cryptarchia/mod.rs:746`, `ledger/src/mantle/sdp/mod.rs:286`) have the intended semantics; `kms/keys/src/keys/zk/public.rs:59-60` returns `false` on a verifier-input error (fail closed).
- Every `map_err(|_| …)` in `core/src/sdp/mod.rs`, `blend/message`, `blend/proofs`, `blend/network/…/utils.rs`, `ledger/src/mantle/sdp/rewards/blend/target_epoch.rs`, `services/chain/chain-service/src/uncle.rs`, `zk/proofs/*` maps to a typed rejection; the root cause is lost from the log but the decision is correct.
- `unwrap_or_default()` outside tests: `awaiting_genesis_time.rs:187` cannot fail (the duration is positive by the guard on `:185`); `swarm.rs:316`, `pprof.rs:89`, `openapi.rs:154`, `tracing/src/compressed_appender.rs:87,136`, `nodes/version/build.rs:95,98` are non-validation.
- `unwrap_or(`: `ledger/src/mantle/pow/mod.rs:245` and `ledger/src/mantle/leader.rs:182` saturate deliberately and are documented; `core/src/mantle/channel.rs:253` is a slot fallback with the intended meaning.
- Error `Display` in `kms/`, `services/key-management-system`, `wallet/`, `services/wallet`, `blend/crypto`: no `#[error(…)]` string formats a key, seed or witness. `services/wallet/src/api.rs:36` `Debug`-formats a `WalletMsg` on relay failure; the enum (`services/wallet/src/lib.rs:150-`) carries public keys, hashes and a `MantleTxBuilder`, no secrets. Derive hygiene of the types themselves is PR #128 / #129.
- Config loading rejects unknown keys on every node path (`OnUnknownKeys::Fail` at `main.rs:58-81`, `deployment/mod.rs:47`, `c-bindings/src/api/lifecycle.rs:140,167,268`); only the `participate` and `get-peer-id` CLI subcommands warn (`cli/participate.rs:38`, `cli/get_peer_id.rs:14`).
- NTP failures keep the last offset with a `warn!` (`services/time/src/backends/ntp/mod.rs:68-76`); that policy is #46's subject.
- Gossipsub runs with `ValidationMode::None` and application-level `validate_messages` off by default (`libp2p/src/behaviour/mod.rs:72-79`, `libp2p/src/config/gossipsub.rs:118-120`); undecodable gossip is dropped at `debug!` with no peer penalty (`services/chain/chain-network/src/network/adapters/libp2p.rs:235-238`, `services/tx-service/src/network/adapters/libp2p.rs:73-76`). These are #144 / #57, not repeated here.
- `services/blend/src/core/dispatcher/libp2p.rs:146,217` drop payloads that do not fit a blend message with `.ok()`; the comment explains why that is the correct filter (the sender could not have used blend for them).

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
