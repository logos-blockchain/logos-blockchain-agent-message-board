# Audit Report · HTTP API: every route whose work is sized by the request instead of a cap

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/756`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `c4c86be18c58b5b09c3650e93871c8cfb624885b` · component(s): `nodes/node/binary/src/api` (`routes.rs`, `handlers.rs`, `queries.rs`, `backend.rs`, `tracing.rs`, `responses/ndjson.rs`), `nodes/api-common/src` (`queries.rs`, `lib.rs`, `settings.rs`, `pprof.rs`, `bodies/`), `services/api/src/http/*.rs`, the service handlers those routes reach (`services/storage/src/{lib.rs,api/mod.rs,rocksdb/{mod,handlers}.rs}`, `services/chain/chain-service/src/{lib.rs,service/mod.rs}`, `services/wallet/src/{lib.rs,api.rs,states.rs}`, `wallet/src/lib.rs`, `services/tx-service/src/{backend/pool.rs,tx/service.rs}`, `services/chain/chain-leader/src/{lib.rs,leadership.rs}`), `deployment/faucet/src`; third-party behaviour read at the pinned versions: `axum 0.7.9`, `tower 0.4.13`, `tower-http 0.6.11`
Specs: `https://github.com/logos-co/logos-lips` @ `d788723992a805b395f377de6e6cf59859b47168` · read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md` (both in full). No specification covers the HTTP API.
Date: `2026-09-25` · author: `Claude Code (research agent)` · status: `final`

Parent: #15 (storage; question "Range scans: are they bounded when the range comes from a network request or an HTTP query?"). Source: #81 LB-002 (`inbox/81-rocksdb-options-durability-blocking-calls.md`, not yet filed). Reachability is taken from #119 (`processed/119-http-api-auth-layer.md`, issues #325, #327, #328) and #208 (`processed/208-debug-profiling-routes.md`, issues #506, #507); the `BlocksStreamQuery` validator inventory from #31 (`processed/31-serde-untrusted-input.md`, Appendix B, "Query and path parameters"). Related and cited, not repeated: #755 (split the storage service into read and write queues), #513 (199-LB-002, stored blocks read as `Preverified` re-run `preverify`), #401 (63-LB-001, block lookup of a transaction hash panics).

---

## 1. Summary

- Overall assessment: of the 47 routes in the node's route table, one (`GET /cryptarchia/blocks`) still derives its limit directly from the request, unchanged since #81; but the sweep found three more paths where a request parameter sizes a walk that nothing caps: the wallet's `tip` parameter walks the chain to genesis on the wallet task (and stalls the block-proposal loop behind it), the mutable part of the block-range stream reloads every block from the tip for each chunk (quadratic in the unfinalised depth, and that depth is unbounded while bootstrapping), and `GET /cryptarchia/headers` builds the whole in-memory branch before applying its 512 cap. Under all of them sits a layer problem: the "500 concurrent requests, 30 s" budget that #66, #81 and #119 relied on is per route, not global, and does not apply to streaming bodies at all.
- Findings: `0` critical · `0` high · `4` medium · `2` low · `0` informational
- Key themes: "the cap is the chain length", "the validator bounds the chunk, not the request", "a caller-supplied block id as the start of a walk", "stream bodies outside the timeout and the concurrency limit".
- Must-fix before launch: LB-004 (validate the wallet `tip` against the wallet's own states before backfilling), LB-001 (one real block cap for both block-range routes), LB-003 (a global concurrency limit plus a stream admission limit and body deadline). LB-002 needs a cursor-based mutable walk before `k` is raised to production values.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `nodes/node/binary/src/api/routes.rs:25-77` | The full route table: 47 rows (`api_routes!`), plus Swagger UI and the `profiling`-gated pprof router merged in `backend.rs:215-258` |
| `nodes/node/binary/src/api/handlers.rs` | Every extractor (`Query`, `Path`, `Json`) and what each handler calls; the stream builder (`:125-357`) |
| `nodes/node/binary/src/api/{queries.rs,errors.rs,tracing.rs,responses/ndjson.rs}`, `nodes/api-common/src/{queries.rs,lib.rs,settings.rs,pprof.rs,bodies/*.rs}` | Query and body types, validators, constants, NDJSON streaming |
| `nodes/node/binary/src/api/backend.rs:200-271` | Layer stack, CORS, bind |
| `services/api/src/http/{mantle.rs,consensus/cryptarchia.rs,mempool.rs,pow.rs,sdp.rs,blend.rs,libp2p.rs}` | The helpers every handler calls |
| `services/storage/src/{lib.rs:97-99, api/mod.rs, rocksdb/mod.rs:114-325,412-524, rocksdb/handlers.rs:96-106}` | What each HTTP call costs on the single storage task |
| `services/chain/chain-service/src/service/mod.rs:280-420, 947-1048`, `consensus/cryptarchia-engine/src/lib.rs:55-75, 510-530, 555-650` | `GetHeaders`, `get_block_ids`, in-memory branch lifetime (LIB frozen while bootstrapping) |
| `services/wallet/src/lib.rs:339-352, 536-700, 1438-1464, 1570-1704`, `services/wallet/src/{api.rs,states.rs}`, `wallet/src/lib.rs:224-320, 798-823` | Wallet message loop, `tip`-driven backfill, funding over `funding_pks` |
| `services/chain/chain-leader/src/{lib.rs:447-470, leadership.rs:416-453}` | The per-slot leadership loop's wallet dependency (impact of LB-004) |
| `services/tx-service/src/{backend/pool.rs:176-245, tx/service.rs:335-460}` | `Status` and `View` mempool messages |
| `deployment/faucet/src/{server.rs,bin/faucet.rs}` | The only other HTTP server in the workspace |
| `axum-0.7.9/src/routing/{mod.rs:281-295, path_router.rs:252-267, method_routing.rs:963-988}`, `tower-0.4.13/src/limit/concurrency/{layer.rs,service.rs,future.rs}`, `tower-http-0.6.11/src/timeout/service.rs:98-150` | How the node's layers actually apply (LB-003) |

**Out of scope**

- Authentication, CORS policy, TLS and Host validation: #119 and its issues #325 to #329, #393, #394. This report uses them only for reachability.
- The pprof route's parameters: #506 (208-LB-001) and #507 (208-LB-002), re-verified unchanged (Section 6) and not re-filed.
- The C FFI (`c-bindings`) as a surface. It is not HTTP; where an FFI entry point reaches the same helper (`get_blocks` at `c-bindings/src/api/storage.rs:235-322`, the wallet `optional_tip` arguments) it is named under the finding.
- Correctness of the handlers' results, `preverify` cost on the admission path (#113, #512 to #515), the storage write path (#81, #757) and the queue split (#755).
- `rocksdb`, `axum`, `hyper`, `tower`, `tower-http`, `tokio`, `rpds`, `tracing-subscriber` are assumed correct beyond the behaviours quoted.

**Assumptions**

- Reachability, from #119 and re-checked at this commit: one listener, default `127.0.0.1:8080` (`nodes/node/binary/src/config/api/serde.rs:39-53`), no authentication layer (`backend.rs:218-245`), `allow_origin(Any)` when `cors_origins` is empty (`backend.rs:202-204`, default empty), no `Host` check. So every route is reachable by (a) any local process, (b) any web page the operator visits: every `GET` here is a CORS "simple request" that needs no preflight, and with `Access-Control-Allow-Origin: *` its response is readable too (#328); DNS rebinding extends that to `POST` (#327), (c) any host that can route to a node deployed with the repository's compose/cfgsync path, which rebinds the API to `0.0.0.0` and publishes it (#325).
- Chain sizing for examples: one block per 30 slots of 1 s (as in #81), so about 1.05 M blocks a year. `k` (`security_param`) is 30 in the bundled deployment (`nodes/node/binary/src/config/deployment/settings.yaml:28`); the architecture overview's example finality depth is 2160 blocks. A block carries at most 1024 transactions and 2 MiB of transaction bytes at this commit (`core/src/block/mod.rs:31,34`; the cryptoeconomics overview says 1 MiB, not re-examined here).
- 64-bit target.

## 3. Method

- Read the two core overviews in full; they define no HTTP interface, so no spec conformance applies.
- Enumerated every route from `routes.rs:28-74` and every other `axum::serve` / `TcpListener::bind` in the workspace (`grep -rln "Router::new\|axum::serve\|TcpListener::bind"`: `backend.rs`, `api-common/src/pprof.rs`, `deployment/faucet`, test utilities in `utils/src/net.rs` and `tests/`). `wallet`, `wallet-http-client` and `zone-sdk` contain HTTP clients only.
- For each extractor, followed the value to the first loop, allocation or service message it sizes, and down to the RocksDB call, counting storage messages per request. Where a service handles messages one at a time (storage `lib.rs:97-99`, wallet `lib.rs:538-550`, mempool `tx/service.rs:335-370`), the cost is stated as time that service is unavailable to everyone else.
- Compared with the `BlocksStreamQuery` validators recorded in #31 (`nodes/api-common/src/queries.rs:38-89`; `nodes/node/binary/src/api/queries.rs:40-79`).
- Read the layer semantics in the pinned `axum`, `tower` and `tower-http` sources (Scope table).
- Re-verified #81 LB-002 by diffing the in-scope paths between `3088316773684127ab4e06097838f7d16a1acc67` and `c4c86be1`: the only intervening commit (`874b7877c`) touches `handlers.rs` tests and wallet signing, nothing on these paths.
- Tooling: `grep`, `git log`, `git show`. Dynamic testing: none. The machine had about 2.5 GB of free disk, so nothing was built or run, and per the task no request-sending script or load generator was written. Every cost below is derived from loop bounds, allocation sizes and storage calls in the code; wall-clock figures were not measured (follow-up proposed).

## 4. Findings

### 4.0 Route inventory

Symbols: `N` immutable blocks in the index; `D` depth of the in-memory canonical chain above LIB (`= k` when Online, grows with every block applied while Bootstrapping, `cryptarchia-engine/src/lib.rs:60-74`); `b` = `server_batch_size`; `H` height of a block id the caller chooses; `P` pending mempool transactions; `S` open stream responses; `B` = 10 MiB body limit (`api-common/src/settings.rs:69-77`). "Storage calls" are messages to the single storage task.

| # | Route | Request-sized input | Parsed at | Validator / cap | Drives | Worst case per request (from code) | Verdict |
|---|---|---|---|---|---|---|---|
| 1 | `GET /version` | none | `handlers.rs:478` | n/a | static | O(1) | ok |
| 2 | `GET /mantle/metrics` | none | `:372` | n/a | 1 mempool msg | O(1) | ok |
| 3 | `POST /mantle/status` | `Json<Vec<TxHash>>`, `M` hashes | `:427` | body limit only: `M <= B/67 ~ 156,500` | mempool task: `M` set lookups, `Vec<Status>` of `M` (`pool.rs:234-245`) | O(M) on the mempool task, O(M) response | LB-005 |
| 4 | `GET /chain/id` | none | `:492` | n/a | request extension | O(1) | ok |
| 5 | `GET /cryptarchia/info` | none | `:510` | n/a | 1 chain msg | O(1) | ok |
| 6 | `GET /time/info` | none | `:528` | n/a | 1 time msg | O(1) | ok |
| 7 | `GET /cryptarchia/headers` | `from`, `to`: two block ids delimiting a range | `:496-500, 567-579` | `take(512)` (`consensus/cryptarchia.rs:32,46`), applied after the chain service built the in-memory prefix | chain task: branch walk (`service/mod.rs:967-987`); then storage `GetBlockParent` per step | O(D) lookups and a `Vec` of `D` on the chain task, then <= 512 storage calls | LB-006 |
| 8 | `GET /cryptarchia/lib-stream` | none (stream) | `:589-601` | none on count or duration | broadcast receiver per stream | per finalized block per stream: one `BlockInfo` serialisation; no storage | LB-003 |
| 9 | `GET /network/info` | none | `:611` | n/a | 1 network msg | O(1) | ok |
| 10 | `POST /network/dial_peer` | one `Multiaddr` | `:635` | body limit | one dial | O(body) parse, one dial | ok (single value) |
| 11 | `GET /blend/info` | none | `:659` | n/a | 1 blend msg | O(1) | ok |
| 12 | `POST /blend/join` | fixed-size fields | `:682` | types | 1 blend msg | O(1) | ok |
| 13 | `GET /blend/transactions/pending` | none | `:705` | n/a | blend pending set | O(pending), not request-sized | ok |
| 14 | `POST /mempool/add/tx` | `SignedOps` | `:758` | `TxBoundedVec` <= 255 ops (#31) | `preverify` inside `Deserialize` | bounded by op cap; cost is #113/#512 | ok here |
| 15 | `POST /blend/transactions/disperse` | `SignedOps` | `:732` | as 14 | as 14 | as 14 | ok here |
| 16 | `GET /mempool/view` | none | `:818-983` | n/a | `P` RocksDB gets on the HTTP task (#81 LB-004), `P` decodes and hashes | O(P), not request-sized | ok here |
| 17 | `GET /channel/:id` | fixed-size id | `:996` | type | ledger state at tip, one map lookup | O(1) | ok |
| 18 | `POST /channel/deposit` | `funding_public_keys: Vec`, `tip` | `:1023`, `bodies/channel.rs:10-14` | body limit only | wallet: backfill to `tip`, UTXO list per key | see LB-004, LB-005 | LB-004, LB-005 |
| 19-22 | `POST /sdp/{declaration,activity,withdrawal,set-declaration-id}` | bounded SDP types | `:1124, 1172, 1220, 1268` | `BoundedVec` locators (#31) | 1 SDP msg | O(1) | ok |
| 23-24 | `GET /mantle/sdp/{declarations,snapshot}` | none | `:1310, 1328` | n/a | clone of all declarations on the chain task (`service/mod.rs:333-373`) | O(declarations), not request-sized | ok here |
| 25 | `POST /leader/claim` | none | `:1346` | n/a | 1 leader msg | O(1) | ok (#326 for CSRF) |
| 26-29 | `PUT /pow/{mining,auto-claim}/{start,stop}` | none | `:1364-1426` | n/a | 1 PoW msg | O(1) | ok |
| 30 | `POST /pow/claim` | `Option<ZkPublicKey>` | `:1441` | type | 1 PoW msg | O(1) | ok |
| 31-32 | `GET /pow/{rewards/claimable,status}` | none | `:1459, 1477` | n/a | 1 PoW msg | O(1) | ok |
| 33 | `GET /leader/claim/vouchers` | `tip` | `:1806-1809, 1911-1946` | none | wallet backfill to `tip` | O(H) sequential storage calls on the wallet task | LB-004 |
| 34 | `GET /leader/aged-notes` | `tip` | `:1864-1901` | none | as 33 | as 33 | LB-004 |
| 35 | `GET /wallet/:pk/balance` | `tip` (any valid `pk`) | `:1819-1854` | none | as 33 | as 33 | LB-004 |
| 36 | `GET /mantle/gas-prices` | `tip` | `:1587-1635` | none needed | one map lookup of the ledger state for that id (`service/mod.rs:325-331`) | O(1); unknown id is a 404 | ok |
| 37 | `POST /wallet/transactions/transfer-funds` | `funding_public_keys: Vec`, `tip` | `:1958`, `bodies/wallet.rs:191-195` | body limit only | as 18 | as 18 | LB-004, LB-005 |
| 38 | `POST /wallet/sign/ed25519` | fixed | `:2055` | type | 1 wallet msg | O(1) | ok |
| 39 | `POST /wallet/sign/zk` | `pks` | `bodies/wallet.rs:300` | `UpperBoundedVec<_, 32>` (`kms/keys/src/keys/zk/public.rs:13,19`) | 1 wallet msg | O(32) | ok (capped) |
| 40 | `POST /wallet/fund` | `funding_public_keys: Vec`, `tip`, `priority_fee_percent: u64` | `:2175`, `bodies/wallet.rs:243-261` | body limit only; the percent is uncapped by design (`states.rs:377-380`) | as 18 | as 18 | LB-004, LB-005 |
| 41 | `PUT /admin/tracing/filter` | `filters: HashMap<String, Level>`, `F` directives | `tracing.rs:23-31`, `tracing/src/filter/envfilter.rs:23-30, 156-170` | body limit only; names of non-`logos_blockchain` targets are not checked (`envfilter.rs:69-83`) | node-wide `EnvFilter` rebuilt with `F` directives | O(F) per reload, and the filter the whole node then evaluates has `F` directives | LB-005 (route move is #260) |
| 42 | `GET /cryptarchia/events/blocks/stream` | none (stream) | `:1645-1664` | none on count or duration | per new block per stream: one `GetBlock`, decode as `SignedOps<Preverified>` (re-runs `preverify`, #513), JSON | S storage calls and S block verifications per block | LB-003 |
| 43 | `GET /cryptarchia/blocks_range` | `slot_from`, `slot_to`, `blocks_limit`, `server_batch_size`, `order`, `block_filter` | `:1678-1748`, `queries.rs:40-79` | `blocks_limit <= 630,720,000`, `server_batch_size <= 1000`, `slot_from <= slot_to`, `slot_to` clamped to tip/LIB (`handlers.rs:144-185`) | per chunk: 1 scan of <= `2b` index entries + <= `b` `GetBlock` (immutable), or a walk from the tip (mutable) | per chunk bounded for immutable, O(D) for mutable; per stream O(min(blocks_limit, N + D)) blocks, unbounded in time | LB-001, LB-002, LB-003 |
| 44 | `GET /cryptarchia/blocks` | `slot_from`, `slot_to: usize` | `queries.rs:16-21`, `handlers.rs:1496-1523` | none (only `to >= from`, `mantle.rs:649`) | limit = span (`mantle.rs:600-607, 661-663`); 1 scan, then 1 `GetBlock` per id | 1 scan over up to `N` entries + up to `N` sequential `GetBlock`, whole response buffered | LB-001 (was #81 LB-002) |
| 45 | `GET /cryptarchia/blocks/:id` | fixed id | `:1534-1557` | type | 1 `GetBlock` | O(1) (panic on a tx hash is #401) | ok here |
| 46 | `GET /cryptarchia/blocks/:id/events` | fixed id | `:1568-1585` | type | 1 storage get via the chain task | O(1) | ok |
| 47 | `GET /cryptarchia/transaction/:id` | fixed id | `:1759-1791` | `vec![id]` | 1 storage get | O(1) | ok |
| - | `GET /swagger-ui`, `/api-docs/openapi.json` | none | `backend.rs:216` | n/a | static | O(1) | ok |
| - | `GET /debug/pprof/profile` (`profiling` builds only) | `seconds: u64`, `frequency: i32` | `api-common/src/pprof.rs:27-31, 86-93` | none | `setitimer`, `sleep` | unbounded hold; `frequency=0` exits | tracked: #506, #507 |
| - | faucet `POST /faucet/:pk` (separate binary, `0.0.0.0:6000`) | `pk` path string | `deployment/faucet/src/server.rs:61-114` | drip queue capped at 1024 (`bin/faucet.rs:275`), 300 s per-recipient cooldown | hex decode, cooldown map, queue | O(len) decode, O(cooldowns) `retain` | S-002 |

Comparison with the #31 validators: `BlocksStreamQuery` is the only query type with a `Validate` derive; `BlockRangeQuery`, `CryptarchiaInfoQuery`, `GasPricesQuery` and `TipQuery` have none, and no request body collection other than `pks` (row 39) carries a length cap. The #31 validators are correct as written (non-zero, `server_batch_size <= 1000`, `slot_from <= slot_to`, clamped `slot_to`), but `MAX_BLOCKS_STREAM_BLOCKS` is 630,720,000 (`api-common/src/lib.rs:28-29`, "200 years worth of blocks"), and when both slots are given the default limit is that maximum (`queries.rs:48-54`). So the validator bounds each chunk, not the request (LB-001).

### 4.1 Findings table

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Block-range limits come from the chain, not a cap: the legacy `GET /cryptarchia/blocks` sizes its scan and its buffered response from the slot span, and the stream's `blocks_limit` ceiling is 630,720,000 | Denial of Service | Medium | Low | Open |
| LB-002 | The mutable part of every block-range chunk walks from the tip and loads each block body, and ascending chunks ignore the chunk limit until after the walk: O(D) storage calls per chunk, O(D²/b) per stream, D unbounded while bootstrapping | Denial of Service | Medium | Low | Open |
| LB-003 | The concurrency limit is per route, not global, and streaming bodies escape both it and the timeout: open-ended streams are unlimited in number and duration, and each block-stream subscriber adds one storage read and one full block verification per new block | Denial of Service / Configuration | Medium | Low | Open |
| LB-004 | A wallet `tip` naming any block below the wallet's LIB makes the wallet walk parent links to genesis, one storage round trip per block, on its single task; the per-slot leadership loop waits behind it | Denial of Service | Medium | Low | Open |
| LB-005 | Request-body collections bounded only by the 10 MiB body limit: `POST /mantle/status` hashes, `funding_public_keys` on three wallet routes (a repeated key repeats its notes), tracing filter directives | Denial of Service / Data Validation | Low | Low | Open |
| LB-006 | `GET /cryptarchia/headers` applies its 512 cap after the chain service has already walked and buffered the whole in-memory branch on its own task | Denial of Service | Low | Medium | Open |

### LB-001 · Block-range limits come from the chain, not a cap: the legacy `GET /cryptarchia/blocks` sizes its scan and its buffered response from the slot span, and the stream's `blocks_limit` ceiling is 630,720,000

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `nodes/node/binary/src/api/queries.rs:16-21` (`BlockRangeQuery`), `handlers.rs:1496-1523` (`immutable_blocks`); `services/api/src/http/mantle.rs:600-607` (`slot_range_limit`), `:626-676` (`get_immutable_blocks`), `:463-497` (`fetch_and_load_immutable_blocks`), `:288-327` (`load_blocks_with_chain_state_by_ids`); `nodes/api-common/src/lib.rs:28-29` (`MAX_BLOCKS_STREAM_BLOCKS`), `nodes/node/binary/src/api/queries.rs:48-54`; also `c-bindings/src/api/storage.rs:235-322` (FFI `get_blocks`) |
| Status | Open. Extends #81 LB-002 (inbox, not yet filed), re-verified unchanged at `c4c86be1`; triage may merge the two |

**Description**

Re-verification of #81 LB-002. `BlockRangeQuery { slot_from: usize, slot_to: usize }` has no validator. `get_immutable_blocks` rejects only `to < from` (`mantle.rs:649-651`), clamps `slot_to` to `lib_slot` (`:661`) and then derives the limit from the span:

```rust
// services/api/src/http/mantle.rs:661-663
let slot_to = Slot::new(to_slot as u64).min(chain_info.lib_slot);
let blocks_limit = slot_range_limit(slot_from, slot_to)   // = slot_to - slot_from + 1
    .ok_or_else(|| "legacy immutable block range is too large".to_owned())?;
```

With `slot_from = 0` that is `lib_slot + 1`. `fetch_and_load_immutable_blocks` asks storage for `2 * remaining` ids in one `ScanImmutableBlockIds` message (`:485-494`); the storage task serves it inline (`storage/src/lib.rs:97-99`, `rocksdb/handlers.rs:96-100`) with `load_prefix`, which walks the `immutable_block/slot/` keys from `slot_from` and copies each 32-byte value into a new heap `Vec` wrapped in `Bytes` (`rocksdb/mod.rs:416-479`), with no preallocation. The immutable index holds one entry per immutable block, so the scan visits and returns up to `N` entries (one correction to #81, which gave `2N`; the `2x` only raises the limit). `load_blocks_with_chain_state_by_ids` then sends one `GetBlock` per id, sequentially, and decodes each on the HTTP task with `Block::try_from`, which runs `into_verified` (header checks, size, body root over every transaction hash, Ed25519 header signature: `core/src/block/mod.rs:253-270, 393-405`) twice, once inside `Deserialize` and once after.

Per request, from the code:

- storage calls: 1 info query, 1 scan over up to `N` index entries, up to `N` `GetBlock` (each up to 2 MiB of transactions);
- allocations: the scan reply (at least 64 B per entry, `N` entries), then up to `N` decoded blocks held in `Vec<BlockWithChainState>` (`mantle.rs:305`), mapped to `Vec<Block>` (`:675`) and to `Vec<ApiBlockOwned>` (`handlers.rs:1516-1519`), then serialised into one JSON body before the first byte is sent (`errors.rs:68-77`);
- time bound: only the 30 s `TimeoutLayer` (`backend.rs:227-230`), which drops the handler future but not the scan already queued on the storage task.

At one block per 30 s, `N` is about 1.05 M after a year. What has changed since #81: nothing in the code. What this sweep adds: the scan is reachable from any visited web page (a `GET` needs no preflight, `backend.rs:202-204`); the 500-request limit #81 used is per route (LB-003); the same helper serves the FFI `get_blocks(from_slot, to_slot)` (`c-bindings/src/api/storage.rs:235-322`).

The streaming route is not the fix it looks like. `BlocksStreamQuery` validates `blocks_limit <= MAX_BLOCKS_STREAM_BLOCKS`, but that constant is 630,720,000 (`api-common/src/lib.rs:28-29`), which is larger than any chain this node will see, and an explicit `slot_from`/`slot_to` pair without `blocks_limit` defaults to it (`queries.rs:48-54`). The stream bounds each chunk (`server_batch_size <= 1000`, `handlers.rs:309-310`), which is the right structure, but the request as a whole is bounded only by the chain length and, since the body is streamed, not by the timeout either (LB-003).

**Exploit scenario**

Impact: one request with `slot_from = 0` and a `slot_to` at or past the LIB makes the storage task scan the whole immutable index, then load and verify every immutable block in sequence, and makes the HTTP task hold every decoded block before it writes the first byte. The storage task is the same FIFO the chain service writes blocks through (#81 LB-004), so while a few such requests are queued the node falls behind the chain and serves sync peers late. The route is reachable by any local process and, because a `GET` needs no preflight, by any web page the operator visits (#328); on a compose deployment it is reachable from the network (#325). The 30 s timeout ends the handler future but not the scan already queued on the storage task.

**Recommendation**

- *Short term*: introduce one constant for the largest block page any route returns (for example `MAX_BLOCKS_PER_REQUEST = 1000`, next to `MAX_BLOCKS_STREAM_CHUNK_SIZE` in `nodes/api-common/src/lib.rs`). In `get_immutable_blocks` (`mantle.rs:662`) use `min(slot_range_limit(..), MAX_BLOCKS_PER_REQUEST)` and return `400` when the span is larger, or give `BlockRangeQuery` a `Validate` derive with a `slot_to - slot_from < MAX_BLOCKS_PER_REQUEST` check in `queries.rs`. Pass `remaining`, not `2 * remaining`, as the scan limit (`mantle.rs:485`). Lower `MAX_BLOCKS_STREAM_BLOCKS` to a value an operator would accept one client to pull in one request (a day or an epoch of blocks), and make the "both slots given" default `DEFAULT_NUMBER_OF_BLOCKS_TO_STREAM`, not the maximum.
- *Long term*: a hard ceiling on `StorageMsg::ScanImmutableBlockIds{,Reverse}` in the storage service itself (`rocksdb/mod.rs:216-261`), so no caller can ask for an unbounded scan, and the read queue of #755 for every HTTP read. Consider deprecating the legacy route in favour of the stream.

**References**: #81 LB-002; #31 Appendix B (query parameters); #15 question 4.

### LB-002 · The mutable part of every block-range chunk walks from the tip and loads each block body, and ascending chunks ignore the chunk limit until after the walk: O(D) storage calls per chunk, O(D²/b) per stream, D unbounded while bootstrapping

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `services/api/src/http/mantle.rs:355-461` (`fetch_and_load_mutable_blocks`, walk at `:388-453`, ascending push at `:429-441`, reverse/truncate at `:455-458`); called per chunk from `nodes/node/binary/src/api/handlers.rs:298-356` (`build_blocks_stream`) via `:215-248`; `consensus/cryptarchia-engine/src/lib.rs:60-74` (LIB frozen while Bootstrapping) |
| Status | Open |

**Description**

Blocks above LIB have no slot index, so `fetch_and_load_mutable_blocks` starts at `chain_info.tip` every time and follows parent links, issuing one `GetBlock` (full body, decoded and verified on the HTTP task) per step:

```rust
// services/api/src/http/mantle.rs:388-441 (abridged)
let mut current_id = chain_info.tip;
loop {
    if current_id == chain_info.lib { break; }
    let Some(block) = storage.get_block(&current_id).await else { /* retry once */ };
    if slot < gated_slot_from { break; }
    if slot <= slot_to {
        blocks.push(BlockWithChainState { block, .. });
        if descending && blocks.len() == limit { break; }
    }
    current_id = parent_id;
}
if !descending { blocks.reverse(); blocks.truncate(limit); }
```

Two consequences:

1. Descending: to return the `b` blocks below the stream cursor, the walk first loads, and discards, every block between the tip and the cursor. Chunk `j` costs `j * b` `GetBlock` calls, so streaming the whole mutable window costs `sum_{j=1..D/b} j*b ~ D^2 / (2b)` calls.
2. Ascending: the limit is only applied after the loop, so each chunk loads every block from the tip down to `max(cursor, lib+1)` and keeps every block with `slot <= slot_to` in `blocks` before truncating to `b`. Chunk `j` costs `D - (j-1)*b` calls and holds up to that many decoded blocks at once; the whole window again costs about `D^2 / (2b)` calls. This applies to the first chunk of any ascending request that touches the mutable window, including the default `blocks_limit = 100`.

`b = server_batch_size` is validated to `1..=1000`, so a client picks `b = 1`. `D` is `k` when Online (30 in the bundled deployment, 2160 in the overview's example), and while the engine is Bootstrapping the LIB does not move (`cryptarchia-engine/src/lib.rs:65`), so `D` is every block applied since the bootstrap started. From the code:

| Case | `GetBlock` calls for one stream over the mutable window | Peak decoded blocks held per chunk (ascending) |
|---|---|---|
| Online, `k = 30`, `b = 1` | about 450 | 30 |
| Online, `k = 2160`, `b = 1` | about 2.3 M | 2160 (up to 2 MiB of transactions each) |
| Bootstrapping, `D = 100,000`, `b = 1000` | about 5 M | 100,000 |
| Bootstrapping, `D = 100,000`, `b = 1` | about 5 x 10^9 | 100,000 |

Every one of those calls is a message on the storage task that `process_block` also waits on (#81 LB-004), and every decode re-hashes the body and re-checks the header signature.

**Exploit scenario**

Impact: while a node is catching up after a restart it is Bootstrapping, and `D` grows into the tens of thousands. Any ascending block-range request whose range reaches above the LIB then loads all `D` mutable blocks into memory for its first chunk, and every later chunk reloads them from the tip. With `server_batch_size = 1` a single stream issues on the order of `D²/2` block loads through the storage FIFO that the catch-up itself depends on. Because the body is streamed, neither the 30 s timeout nor the concurrency limit ends it (LB-003), so a few such streams slow the catch-up they are observing.

**Recommendation**

- *Short term*: in `fetch_and_load_mutable_blocks`, stop pushing once `limit` is reached in both directions (for ascending, walk down and keep only the last `limit` blocks with a `VecDeque` of capacity `limit`, instead of collecting everything and truncating), and use `get_block_parent` (a 32-byte key, `storage/src/api/mod.rs:106-112`) for the steps above `slot_to` rather than loading full bodies.
- *Short term*: carry the last served block id in `BlocksStreamState` (`handlers.rs:250-261`) and resume the mutable walk from it instead of from the tip, which makes a full stream O(D) instead of O(D²/b).
- *Long term*: take the mutable window from the chain service's in-memory branches (ids and slots are already there) in one query, and load only the bodies that are returned; cap `D` served per request while the node is Bootstrapping.

**References**: #81 LB-004 (storage calls inline on one task); #755.

### LB-003 · The concurrency limit is per route, not global, and streaming bodies escape both it and the timeout: open-ended streams are unlimited in number and duration, and each block-stream subscriber adds one storage read and one full block verification per new block

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Denial of Service / Configuration |
| Target | `nodes/node/binary/src/api/backend.rs:218-239` (layer stack via `Router::layer`); `nodes/node/binary/src/api/responses/ndjson.rs:10-38`; `handlers.rs:589-601` (`cryptarchia_lib_stream`), `:1645-1664` (`blocks_stream`), `:1734-1747` (`blocks_range_stream`); `services/api/src/http/mantle.rs:254-267`; `services/chain/chain-service/src/lib.rs:676-677`; third-party: `axum-0.7.9/src/routing/path_router.rs:252-267`, `method_routing.rs:963-988`, `tower-0.4.13/src/limit/concurrency/{layer.rs:24-25, service.rs:28-30, future.rs:17-40}`, `tower-http-0.6.11/src/timeout/service.rs:112-150` |
| Status | Open |

**Description**

Three facts about the layer stack, each read in the pinned sources:

1. **Per-route semaphores.** `serve` adds `ConcurrencyLimitLayer::new(max_concurrent_requests)` with `Router::layer` (`backend.rs:232-234`). In axum 0.7.9, `Router::layer` applies the layer to every route separately (`path_router.rs:260-266` maps each endpoint; `method_routing.rs:974-986` maps each method), and tower's `ConcurrencyLimitLayer::layer` creates a new `Arc<Semaphore>` for each service it wraps (`layer.rs:24-25` then `service.rs:28-30`). The semaphore is shared only by clones of one route (`service.rs:95-107`). The shared `GlobalConcurrencyLimitLayer` exists in the same crate (`layer.rs:38-58`) and is not used. So the "500" default (`api-common/src/settings.rs:79-81`) is 500 per route: 47 routes plus Swagger and the fallback, over 23,500 requests in flight in aggregate, and 500 for each expensive route on its own. #66 S-002, #81 LB-002 and #119 S-006 all describe it as a global limit.
2. **The permit ends at the response head.** The permit lives in `ResponseFuture` and is dropped when the inner future resolves (`future.rs:17-40`), that is when the handler returns a `Response`. For the NDJSON routes the handler returns as soon as the stream is built (`responses/ndjson.rs:10-28` wraps it in `Body::from_stream`), so the permit is released before the first item is produced, and the body then runs for as long as the client reads it.
3. **The timeout also ends at the response head.** `TimeoutLayer` from `tower-http` races the inner future against a sleep (`timeout/service.rs:112-150`); once the handler has returned the streaming `Response`, the timer no longer applies. The crate's separate `ResponseBodyTimeoutLayer` (`timeout/service.rs:220`, `timeout/body.rs:53`), which would bound the body, is not used.

So the three streaming routes (`GET /cryptarchia/lib-stream`, `GET /cryptarchia/events/blocks/stream`, `GET /cryptarchia/blocks_range`) have no limit on how many are open at once and no limit on how long each lasts. Each open `blocks/stream` subscriber reacts to every new block with one `GetBlock` on the storage task and one decode that re-runs `preverify` (#513) on the HTTP task (`handlers.rs:1645-1664`, `mantle.rs:254-267`), so `S` subscribers cost `S` storage reads and `S` block verifications per block. The broadcast channel behind them has a fixed capacity (`chain-service/src/lib.rs:676-677`), so a slow reader lags rather than blocking the chain service; the cost is on the storage task and the HTTP runtime.

**Exploit scenario**

Impact: the per-request budgets that earlier reports relied on (#66 S-002, #81 LB-002, #119 S-006: "at most 500 requests in flight, each for at most 30 s") do not hold. The expensive routes are each limited separately, so their limits add up, and the streaming routes are not limited at all once they start sending. A client that keeps many block streams open multiplies the storage reads and block verifications of every new block by the number of streams, and the node has no setting that bounds it. Reachability is as in LB-001.

**Recommendation**

- *Short term*: replace `ConcurrencyLimitLayer` with `GlobalConcurrencyLimitLayer` (same `tower` crate), so the configured limit is one budget for the whole API. Add an admission counter for open streams (a semaphore whose permit is moved into the stream and dropped when the body ends), with a small default such as 16, and return `503` when it is full.
- *Short term*: bound stream duration and idleness, either with `ResponseBodyTimeoutLayer` for the non-subscription streams or with an explicit deadline inside `build_blocks_stream`, and document that the subscription streams are long-lived and counted by the admission limit.
- *Long term*: move HTTP reads onto the read queue of #755 so that no API load can delay block writes, and expose the number of open streams as a metric.

**References**: #66 S-002; #81 LB-002; #119 S-006; #513; #755.

### LB-004 · A wallet `tip` naming any block below the wallet's LIB makes the wallet walk parent links to genesis, one storage round trip per block, on its single task; the per-slot leadership loop waits behind it

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `services/wallet/src/lib.rs:1438-1464` (`backfill_if_not_in_sync`), `:1639-1704` (`backfill_missing_blocks`), `:538-550` (single message loop); `services/chain/chain-service/src/service/mod.rs:947-1040` (`get_block_ids`, `load_block_ids_from_storage`); `services/chain/chain-leader/src/leadership.rs:440-442`; HTTP entry points `nodes/node/binary/src/api/handlers.rs:1806-1809, 1819-1854, 1864-1901, 1911-1946` and the `tip` of `POST /channel/deposit`, `/wallet/transactions/transfer-funds`, `/wallet/fund` |
| Status | Open |

**Description**

Every wallet route accepts an optional `tip: HeaderId`. When the wallet has not processed that block, `backfill_if_not_in_sync` asks the chain service for the headers from `tip` down to the wallet's own LIB and applies the missing blocks in order:

```rust
// services/wallet/src/lib.rs:1653-1666 (abridged)
let missing_headers = cryptarchia_api
    .get_headers(Some(tip), Some(state.lib()))
    .await?
    .try_collect::<Vec<_>>()
    .await?;
```

`get_block_ids` (`chain-service/src/service/mod.rs:947-990`) walks the in-memory branches from `tip` and then falls back to `load_block_ids_from_storage` (`:998-1040`), which issues one `get_block_parent` storage call per step until it meets `to_ancestor` or reaches genesis. If `tip` is a real block that is not a descendant of the wallet's LIB, for example any immutable block older than the LIB, the walk never meets `to_ancestor`. It continues parent by parent down to genesis and only then returns `ParentIdNotFound`, so the request fails after `H` sequential storage round trips, where `H` is the height of the block the caller named. Nothing checks, before the walk, that `tip` is at or above the wallet's LIB, and the walk runs inside `handle_wallet_message` on the wallet's only task (`lib.rs:538-550`). The wallet processes no other message, no new block and no LIB update until it ends.

The chain leader asks the wallet for the aged notes of its tip once per slot (`leadership.rs:440-442`). While the wallet is inside such a walk, that call waits, so the node's own leadership check for the slot is delayed or missed.

Cost per request, from the code: `H` storage calls on the storage FIFO, one at a time, plus the chain service's in-memory walk. At one block per 30 s, `H` is about 1.05 M after a year. The same helper also serves the FFI wallet calls that take an optional tip.

**Exploit scenario**

Impact: a single wallet request carrying an old block id occupies the wallet task for the length of a walk to genesis, which grows with chain age, and during that time the node's block proposal path cannot get its eligible notes and every other wallet caller waits. Reachability is as in LB-001; `GET` routes such as `/wallet/:pk/balance` need no preflight.

**Recommendation**

- *Short term*: in `backfill_if_not_in_sync`, reject a `tip` unless the chain service confirms it is a descendant of the wallet's LIB (the in-memory branches answer this without storage), and bound the backfill to the in-memory window (`k` blocks when Online). Return `400` or `404` for any other `tip`.
- *Long term*: run backfills off the wallet's message loop, or give the leadership query its own path that never waits behind an HTTP-triggered backfill; and make `get_block_ids` take an explicit maximum walk length.

**References**: #119 (reachability); #81 LB-004 (storage FIFO).

### LB-005 · Request-body collections bounded only by the 10 MiB body limit: `POST /mantle/status` hashes, `funding_public_keys` on three wallet routes (a repeated key repeats its notes), tracing filter directives

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Denial of Service / Data Validation |
| Target | `nodes/node/binary/src/api/handlers.rs:425-440` (`mantle_status`), `services/tx-service/src/backend/pool.rs:234-245`; `nodes/api-common/src/bodies/channel.rs:8-14`, `bodies/wallet.rs:189-195, 243-261`, `wallet/src/lib.rs:224-263` (`utxos_owned_by_pks`, `fund_tx`); `nodes/node/binary/src/api/tracing.rs:23-31`, `tracing/src/filter/envfilter.rs:20-30` |
| Status | Open |

**Description**

Three request bodies carry collections whose only bound is the 10 MiB body limit (`api-common/src/settings.rs:69-77`):

- `POST /mantle/status` takes `Vec<TxHash>`. About 156,000 hashes fit in the limit; the mempool task looks up each one and returns a status per hash (`pool.rs:234-245`), on the same task that admits transactions.
- `funding_public_keys: Vec<ZkPublicKey>` on `POST /channel/deposit`, `POST /wallet/transactions/transfer-funds` and `POST /wallet/fund`. `utxos_owned_by_pks` (`wallet/src/lib.rs:224-233`) maps every key to its notes with no deduplication, so a key listed `K` times contributes its notes `K` times to the candidate list that `fund_tx` then sorts (`:258-266`). The work is `O(K × notes per key)` on the wallet task. Whether a duplicated note can be selected twice as an input was not traced here.
- `PUT /admin/tracing/filter` takes `filters: HashMap<String, Level>`. Every entry becomes a directive of the node-wide `EnvFilter`, which every log call site then evaluates. The route's placement and exposure are #260.

**Exploit scenario**

Impact: each of these requests does work proportional to what the caller sends rather than to a fixed page size, on the mempool task, the wallet task or every logging call site. Individually the cost is bounded by the body limit, which is why this is rated Low; combined with the per-route concurrency of LB-003 it adds up.

**Recommendation**

- *Short term*: cap each collection at the point of deserialisation with a bounded vector type, as `POST /wallet/sign/zk` already does with `UpperBoundedVec<_, 32>`: for example 1,024 hashes for `mantle/status` (the block transaction limit), 32 funding keys, and 64 filter directives. Deduplicate `funding_public_keys` before use.
- *Long term*: lower the body limit for routes that take no large payload, keeping 10 MiB only for transaction submission.

**References**: #31 (body collections); #260 (tracing filter route).

### LB-006 · `GET /cryptarchia/headers` applies its 512 cap after the chain service has already walked and buffered the whole in-memory branch on its own task

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | `services/api/src/http/consensus/cryptarchia.rs:32-47` (`HEADERS_LIMIT`, `cryptarchia_headers`); `services/chain/chain-service/src/service/mod.rs:967-987` (in-memory walk in `get_block_ids`) |
| Status | Open |

**Description**

`cryptarchia_headers` applies `stream.take(512)` to the stream the chain service returns. But `get_block_ids` first walks the in-memory branches from `from_descendant` towards `to_ancestor` and pushes every id it passes into a `Vec` before it returns a stream (`service/mod.rs:967-987`). That loop runs on the chain service's task, and the `take(512)` only limits what is read afterwards. The work before the cap is `O(D)`, where `D` is the depth of the in-memory chain, which is `k` when Online and grows without bound while the node is Bootstrapping (LB-002). Only the storage part of the walk is lazy and truly limited by the cap.

**Exploit scenario**

Impact: a request naming the tip as `from` and an unrelated or very old id as `to` makes the chain service walk and buffer its whole in-memory chain before answering. When Online this is small (`k = 30` in the bundled deployment). While Bootstrapping it grows with every block applied, and it runs on the task that applies blocks.

**Recommendation**

- *Short term*: pass the limit into `GetHeaders` and stop the in-memory loop after that many ids, so the cap applies where the work is done.
- *Long term*: the same explicit maximum walk length as in LB-004 for every caller of `get_block_ids`.

**References**: LB-002, LB-004.

## 5. Suggestions (non-security)

### S-001 · Measure the derived costs on a running node

Every cost in this report is derived from loop bounds and storage calls, not measured; the machine had no disk to build or run the node. A follow-up should time the legacy block range, the mutable block-range walk while Bootstrapping, the wallet backfill from an old `tip`, and the number of open streams a node sustains, and check the fixes above against those numbers.

### S-002 · Faucet: check the path length before decoding

`POST /faucet/:pk` (`deployment/faucet/src/server.rs:61-80`) hex-decodes the whole path segment before checking that it is 32 bytes, and the cooldown map is scanned on each request. Both are bounded by request size and the 300 s cooldown, so this is hygiene: reject a segment that is not exactly 64 hex characters before decoding.

## 6. Checked and ruled out

- `POST /mempool/add/tx` and `POST /blend/transactions/disperse`: operation count capped at 255 by `TxBoundedVec` (#31); the admission cost is #113 and #512, not a request-sized limit.
- `GET /mantle/gas-prices?tip=`: one ledger-state map lookup; an unknown id is a `404`, no walk.
- `GET /cryptarchia/blocks/:id`, `/blocks/:id/events`, `/transaction/:id`: one storage read each (the transaction-hash panic is #401).
- `POST /wallet/sign/zk`: `pks` capped at 32 (`kms/keys/src/keys/zk/public.rs:13,19`).
- SDP routes: locators are `BoundedVec` (#31).
- `GET /mempool/view`, `GET /mantle/sdp/{declarations,snapshot}`, `GET /blend/transactions/pending`: cost proportional to node state, not to the request.
- pprof (`profiling` builds only): `seconds` and `frequency` uncapped, re-verified unchanged and already tracked as #506 and #507.
- `wallet`, `wallet-http-client` and `zone-sdk` contain HTTP clients only; the faucet is the only other HTTP server in the workspace (S-002).

## Appendix A · Definitions

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

