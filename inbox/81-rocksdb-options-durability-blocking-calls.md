# Audit Report — Storage: RocksDB options, write durability, and blocking calls on the tokio runtime

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/81`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `3088316773684127ab4e06097838f7d16a1acc67` — component(s): `services/storage` (RocksDB backend and service loop), `services/chain/chain-service` (block-apply callers), `services/api/src/http/mantle.rs` and `nodes/node/binary/src/api` (HTTP range scan), `nodes/node/binary` (storage config, startup), `deployment/systemd`
Specs: `https://github.com/logos-co/logos-lips` @ `d788723992a805b395f377de6e6cf59859b47168` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md` (both in full; the issue states that no specification covers this area). Consulted by section: `overview-cryptoeconomics.md` § Permanent Storage Fee Market (1 MiB block cap, roughly 1 TB of ledger data per year) for the sizing in LB-001.
Date: `2026-09-24` — author: `claude-fable-5.1` — status: `final`

The issue was filed against `19353c619`, which is no longer in the repository history; the storage crate has since been restructured (`services/storage/src/backends/rocksdb.rs` is now `services/storage/src/rocksdb/{mod,handlers}.rs`, the typed API lives in `services/storage/src/api/mod.rs`). All line numbers below are at the commit in the header. RocksDB option semantics are taken from the RocksDB sources pinned by `librocksdb-sys 0.17.3+10.4.2` (`Cargo.lock:3871-3873`), i.e. RocksDB v10.4.2 `include/rocksdb/options.h`, `include/rocksdb/advanced_options.h`, `options/options.cc`, and from `rust-rocksdb 0.24.0` `src/db.rs` and `librocksdb-sys/build.rs`.

---

## 1. Summary

- Overall assessment: the RocksDB layer runs on every library default, and three of those defaults matter operationally: `max_open_files = -1` (every SST file is opened at startup and kept open, so a default systemd install stops being able to open its own database at about 1,000 SST files), no `sync` on any write (an acknowledged block or recovery record survives a process crash but not a kernel crash or power loss), and no compression at all (the crate is built without a compression library). Every RocksDB call except `bulk_store` still runs inline on the single storage-service task, which the block-apply path hits three to N+3 times per block. The `limit: None` path of `load_prefix` that the issue asked about is unreachable, but the legacy `GET /cryptarchia/blocks` endpoint derives its limit from the requested slot range and can make one request scan the whole immutable index and load every immutable block through that same task.
- Findings: `0` critical · `0` high · `1` medium · `3` low · `0` informational
- Key themes: "library defaults never reviewed", "one storage task, everything inline", "limits derived from the request rather than capped"
- Must-fix before launch: LB-001 (bound `max_open_files` or ship a `LimitNOFILE` in the unit file), LB-002 (cap the legacy range endpoint like the streaming one).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/storage/src/rocksdb/mod.rs` | `RocksBackend::new` (L340-375), every `StorageBackend` method (L377-544), `store_block_data` / `remove_block` / scans / `get_transactions` (L114-325), `load_prefix_entries` (L95-113) |
| `services/storage/src/rocksdb/handlers.rs` | message dispatch, which calls run inline (L20-122) |
| `services/storage/src/lib.rs` | the single-task service loop (L78-104), metrics (L44-55) |
| `services/storage/src/api/{mod,requests}.rs` | the typed API and message set, to enumerate producers of every `StorageMsg` |
| `services/storage/src/recovery.rs` | recovery key, `load_recovery_data` (L36-46), `save_state` (L111-125) |
| `services/storage/Cargo.toml`, `Cargo.toml:262`, `Cargo.lock:3871, 7287` | `rocksdb` crate features and version |
| `services/chain/chain-service/src/service/mod.rs` | `process_block` (L757-885), `process_block_and_update_state` (L199-236), `delete_blocks_from_storage` (L1088-1120), `persist_recovery_state` (L1167-1181) |
| `services/chain/chain-service/src/sync/block_provider.rs` | the sync-provider scan limit (L339-343, L404-431) |
| `services/api/src/http/mantle.rs` | `get_immutable_block_ids_in_slot_range` (L272-286), `load_blocks_with_chain_state_by_ids` (L288-327), `fetch_and_load_immutable_blocks` (L463-497), `get_blocks_in_slot_range_with_snapshot` (L499-595), `slot_range_limit` (L600-607), `get_immutable_blocks` (L626-676) |
| `nodes/node/binary/src/api/{queries,routes,handlers,backend}.rs`, `nodes/api-common/src/{paths,queries}.rs` | `BlockRangeQuery` (queries.rs L16-21), route (routes.rs L71), `immutable_blocks` handler (handlers.rs L1496-1523), timeout layer (backend.rs L227-229), streaming-endpoint limits (api-common queries.rs L15-69) |
| `nodes/node/binary/src/config/storage/{serde,mod}.rs`, `nodes/node/binary/src/lib.rs:182-187`, `nodes/node/standalone-node-config.yaml:170-174` | storage settings, defaults, the startup open |
| `services/utils/src/overwatch/recovery/operators.rs:61-66`, `services/pow/src/service.rs:319-340`, `services/tx-service/src/storage/adapters/rocksdb.rs` | what is persisted through the storage service and how (for LB-003 impact) |
| `deployment/systemd/logos-blockchain-node.service`, `deployment/systemd/logrotate-logos-blockchain-node.conf` | resource limits and log rotation of the shipped deployment |
| Overwatch @ `8f06c685948620d5d9931ed8f5bdb86e6cbffcd9` (`Cargo.lock:6251`) | `utils/runtime.rs:9-15`, `overwatch/runner.rs:70`, `services/runner/service_runner.rs:170`: services run as tasks on one shared multi-thread tokio runtime with the default worker count |

**Out of scope**

- The size and write rate of the recovery record and the cost of the recovery `put` itself: PR #209 (issue #188) LB-001 to LB-004. This report only cites them.
- Key scheme, network identity, error feedback on fire-and-forget writes, orphaned blocks, the unused `blocks` column family: PR for issue #63 (LB-001 to LB-006). Schema versioning: issue #181. Relay `unwrap`s: issue #259.
- The streaming blocks endpoint (`BlocksStreamQuery`) whose `blocks_limit` and `server_batch_size` are validated: issue #31 covered its validators; it is only used here as the comparison for LB-002.
- RocksDB internals beyond the option semantics quoted from its headers; `rocksdb 0.24.0`, `librocksdb-sys 0.17.3+10.4.2`, `tokio`, `overwatch` are assumed correct. Linux page-cache writeback behaviour is taken from the kernel defaults (`vm.dirty_expire_centisecs = 3000`, `vm.dirty_writeback_centisecs = 500`) and not measured.
- Whether the wallet, SDP or Blend services can re-derive their persisted state from the chain after losing it (relevant to LB-003's impact list, marked as unverified there).

**Assumptions**

- Repo-level facts from issue #19 hold: `[profile.release]` sets neither `overflow-checks` nor a `panic` strategy (`Cargo.toml:11-14`), the `unwrap`/`expect`/`panic` lints are allowed.
- The HTTP API binds to `127.0.0.1` by default; findings that need API access (LB-002) assume the operator exposed it or the attacker is local, as in the #63 report.
- The node is deployed as the shipped systemd unit or a container; container runtimes vary in their default file-descriptor limit and LB-001 is stated for systemd, whose default soft limit for services is 1024.
- Block interval for the sizing in LB-001: 1 s slots and `f = 1/30`, one block per 30 s on average, as in the #188 report; block size cap 1 MiB from the cryptoeconomics overview.

## 3. Method

- Manual review of the in-scope paths, working through issue `#81` (three items) and the questions of parent `#15` that they correspond to ("Are RocksDB calls made on the tokio runtime directly? Which ones are on hot paths?", "Range scans: are they bounded when the range comes from a network request or an HTTP query?", "Column family and options: anything obviously unsafe (no fsync, huge write buffer, unbounded WAL)?"). Every producer of every `StorageMsg` variant was enumerated with `grep` across the workspace (Section 4, LB-004 and Section 6).
- Spec conformance: none applicable; the issue states no specification covers this area. The two core overviews were read in full.
- Automated tooling: `cargo 1.94.1` / `rustc 1.98.1` (the toolchain pinned by `rust-toolchain.toml`), `strace 6.x`. No clippy or audit run (nothing in scope is lint-shaped).
- Dynamic testing: three `#[ignore]` tests appended to `services/storage/src/rocksdb/tests.rs` in a scratch checkout of the audited commit (Appendix B; no other change to the tree). (a) `audit_options_dump` opens a database exactly as the node does (`RocksBackend::new` with the node-default settings from `config/storage/serde.rs:19-26`) and prints the effective values RocksDB records in its `OPTIONS-*` file. (b) `audit_write_path_syscalls` issues 200 × (`store_block_data` of a 100 KiB block + 4 KiB events + one immutable-index entry, then a 256 KiB recovery `store`) under `strace -f -c` counting `write`, `pwrite64`, `fsync`, `fdatasync`, `sync_file_range`. (c) `audit_fd_exhaustion`, under `ulimit -n 1024`: phase 1 keeps writing and flushing into a database opened with the default `max_open_files` until an operation fails; phase 2 builds a 1,200-file database with a bounded builder, then opens it with the node's settings and again with `max_open_files = 256`. Results are quoted in the findings.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | `max_open_files = -1` with no `LimitNOFILE` in the shipped unit: the node opens every SST file at startup and stops being able to open its own database at about 1,000 files (roughly 60 GB uncompressed) on a default systemd install | Configuration / Denial of Service | Low | High | Open |
| LB-002 | Legacy `GET /cryptarchia/blocks?slot_from=&slot_to=` derives its limit from the slot range: one request makes the single storage task scan the whole immutable index and then load every immutable block, ahead of block application | Denial of Service | Medium | Low | Open |
| LB-003 | Every write uses `WriteOptions::default()` (`sync = false`) and no `bytes_per_sync`: an acknowledged block or recovery record survives a process crash but not a kernel crash or power loss; what is lost is a prefix-consistent tail of up to the page-cache writeback window | Configuration | Low | High | Open |
| LB-004 | Every RocksDB call except `bulk_store` runs inline on the single storage-service task (and on the consumer's task for `get_transactions`); the block-apply path issues three to N+3 such calls per block, so one slow write, one RocksDB write stall, or one large scan stalls block application, sync serving, HTTP reads and mempool persistence together | Denial of Service / Timing | Low | High | Open |

### LB-001 · `max_open_files = -1` with no `LimitNOFILE` in the shipped unit: the node opens every SST file at startup and stops being able to open its own database at about 1,000 files on a default systemd install

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Configuration / Denial of Service |
| Target | `services/storage/src/rocksdb/mod.rs:358-369` (`RocksBackend::new`, read-write arms); `deployment/systemd/logos-blockchain-node.service:8-30` (no `LimitNOFILE`) |
| Status | Open |

**Description**

`RocksBackend::new` builds `Options::default()` and sets only `create_if_missing` / `create_missing_column_families` (`mod.rs:358-369`). In RocksDB v10.4.2 the default is `max_open_files = -1` (`options.h:771`), documented as "files opened are always kept open" and, three lines later, "If max_open_files is -1, DB will open all files on DB::Open()" (`options.h:773-776`). The number of SST files grows with the data: level compaction with the default `target_file_size_base = 64 MiB` and `target_file_size_multiplier = 1` (`advanced_options.h:472`) keeps files at about 64 MiB on every level, so a database of `S` bytes holds about `S / 64 MiB` files, plus level-0 files of memtable size (also 64 MiB, `options.h:188`) and transient compaction outputs. Nothing compresses the data (Suggestion S-001), so the file count tracks the raw block bytes.

The shipped systemd unit sets no `LimitNOFILE` (`deployment/systemd/logos-blockchain-node.service`, whole file). systemd's default for services is a soft limit of 1024 file descriptors. RocksDB does not raise the soft limit itself; when `open(2)` fails with `EMFILE` during `DB::Open` the open returns an `IOError`, which `RocksBackend::new` propagates, `StorageService::init` propagates (`lib.rs:63-76`), and the node does not start. Before that point, the running node hits the limit first: a flush or WAL rollover that needs a new descriptor fails with the same error (phase 1 below shows a WAL open failing at 1,014 files), which RocksDB reports as a background error and, with the default `paranoid_checks = true` (`options.h:620`), turns into the DB refusing further writes until reopened; `store_block_data` then fails and `process_block` returns `Error::Storage` for every block.

Sizing at the parameters in the Assumptions: one 1 MiB block per 30 s is 2,880 MiB/day, 45 files/day, so about 1,000 files after 22 days of full blocks; at 10 % full blocks, after about 7 months; the cryptoeconomics overview's "roughly 1 Terabyte of data per year" is about 16,400 files a year. The node's other file descriptors (libp2p sockets, HTTP connections, the info log, WAL, MANIFEST) come out of the same 1024.

The secondary effect is memory: with `max_open_files = -1` every table reader stays resident, and its index and filter blocks are held outside the 32 MiB default block cache (`options.h:764-766` warns "A high value or -1 for this option can cause high memory usage").

Measured (Appendix B, test c), under `ulimit -n 1024`, with 16 KiB values and one flush per put so that every flush leaves one SST file:

```
AUDIT RLIMIT_NOFILE soft limit in this process: 1024
AUDIT phase1 (max_open_files=-1, running): write/flush #1014 failed with 1014 SST files on disk: IO error: While open a file for appending: /tmp/.tmpkBQKqa/002036.log: Too many open files
AUDIT phase2 database holds 1200 SST files
AUDIT phase2 open with node defaults FAILED after 90.193396ms: IO error: While open a file for random read: /tmp/.tmpDbm3js/001377.sst: Too many open files
AUDIT phase2 open with max_open_files=256 succeeded in 76.254489ms
```

Phase 1 is the running node: with the default options the database keeps every SST open as it creates them, and the 1,015th flush fails while RocksDB tries to open a new WAL file. Phase 2 is the restart: a database of 1,200 files does not open with the node's settings, and opens in 76 ms with `max_open_files = 256`.

The database is also opened twice at every start, first by `load_recovery_data` (`nodes/node/binary/src/lib.rs:187`, `recovery.rs:36-39`) and then by `StorageService::init` (`lib.rs:68-73`); each open walks every SST file (Suggestion S-003).

**Exploit scenario**

No attacker is required: an operator who installs the shipped unit on a stock Debian or Ubuntu host and syncs a chain of a few tens of GB finds that after a restart the node exits with `Too many open files` from `RocksBackend::new`, and stays down through `Restart=on-failure` (`logos-blockchain-node.service:14-15`) until the operator raises the limit by hand. An adversary who can afford to fill blocks (the 1 MiB cap, priced by the storage market) brings the day forward but pays for it; the growth is otherwise organic. The failure is a liveness loss of individual nodes, correlated across every node deployed the same way, at the same chain size.

**Recommendation**

- *Short term*: set `opts.set_max_open_files(n)` to a bounded value (RocksDB's own tuning guide suggests a few thousand, sized against the deployment's descriptor budget) in all four arms of `RocksBackend::new`, and add `LimitNOFILE=65536` (or similar) to `deployment/systemd/logos-blockchain-node.service`. Mention the requirement in `deployment/systemd/README.md`.
- *Long term*: make the RocksDB options an explicit, reviewed struct in `RocksBackendSettings` (open files, write buffers, WAL bound, compression, info-log rotation) with node defaults, instead of `Options::default()`; add a startup check that logs the soft descriptor limit against the current SST count.

**References**: RocksDB v10.4.2 `include/rocksdb/options.h:758-776` (`max_open_files`), `advanced_options.h:472` (`target_file_size_base`); parent #15 ("Column family and options"); `overview-cryptoeconomics.md` § Permanent Storage Fee Market.

### LB-002 · Legacy `GET /cryptarchia/blocks?slot_from=&slot_to=` derives its limit from the slot range: one request makes the single storage task scan the whole immutable index and then load every immutable block, ahead of block application

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `services/api/src/http/mantle.rs:600-607` (`slot_range_limit`), `:662-663` (`get_immutable_blocks`), `:485-497` (`fetch_and_load_immutable_blocks`), `:288-327` (`load_blocks_with_chain_state_by_ids`); `nodes/node/binary/src/api/queries.rs:16-21` (`BlockRangeQuery`); `nodes/node/binary/src/api/routes.rs:71` |
| Status | Open |

**Description**

Item 3 of the issue asks whether `load_prefix` is reachable with `limit: None`. It is not (Section 6). The live equivalent is a limit that is `Some` but derived from the request. The legacy route `paths::BLOCKS = "/cryptarchia/blocks"` (`nodes/api-common/src/paths.rs:37`, registered at `routes.rs:71`) deserialises `BlockRangeQuery { slot_from: usize, slot_to: usize }` with no validator (`queries.rs:16-21`; compare `BlocksStreamQuery`, whose `blocks_limit` is capped by `validate_blocks_limit`, `nodes/api-common/src/queries.rs:38-69`). The handler calls `get_immutable_blocks` (`handlers.rs:1510-1514`), which clamps `slot_to` to the LIB slot and then sets

```rust
// mantle.rs:661-663
let slot_to = Slot::new(to_slot as u64).min(chain_info.lib_slot);
let blocks_limit = slot_range_limit(slot_from, slot_to)      // = slot_to - slot_from + 1
```

so `slot_from = 0` gives a limit of `lib_slot + 1`. `get_blocks_in_slot_range_with_snapshot` passes it as `remaining` to `fetch_and_load_immutable_blocks`, which asks storage for twice that many index entries in one message:

```rust
// mantle.rs:485-493
let limit = NonZeroUsize::new(remaining.saturating_mul(2).max(1))…;
let header_ids = get_immutable_block_ids_in_slot_range::<RuntimeServiceId>(handle, slot_from, immutable_slot_to, limit, descending).await?;
```

`ScanImmutableBlockIds` is served by `load_prefix` on the storage task, which iterates from `immutable_block/slot/<from>` up to `<to>` or `limit`, copying every value into a `Vec` (`rocksdb/mod.rs:443-478`), inline (LB-004). For a chain of `N` immutable blocks that is one uninterrupted pass over `N` entries with the storage task doing nothing else. `load_blocks_with_chain_state_by_ids` then issues one `GetBlock` per header, sequentially, decoding each block (`mantle.rs:305-322`), and stops only at `blocks_limit` or when the handler is dropped.

The only backstop is the HTTP `TimeoutLayer` at 30 s (`backend.rs:227-229`, `config/api/serde.rs:49`), which drops the handler future. That does not cancel work already queued: the scan message runs to completion on the storage task whatever happened to its reply channel, and each `GetBlock` already sent is served. The API allows 500 concurrent requests (`standalone-node-config.yaml:169`), and a client that reissues on timeout keeps the storage queue populated with full-index scans. Because the storage loop is FIFO (`lib.rs:97-99`), every `StoreBlockData` from `process_block` (`service/mod.rs:798-807`), which the chain service awaits before applying the block, waits behind every scan queued before it.

**Exploit scenario**

Anyone who can reach the HTTP API (loopback by default; any client if the operator exposed it, or a local process) sends `GET /cryptarchia/blocks?slot_from=0&slot_to=18446744073709551615` repeatedly, up to the 500-request concurrency cap. Each request costs the storage task one pass over the whole immutable index (`N` entries, 32 B values) plus up to 30 s of sequential block loads; with a few hundred in flight the storage task never drains, block application stalls behind the queue, the node stops following the chain, and the sync provider (which serves peers through the same task, `block_provider.rs:404-431`) stops serving. Memory grows by `2N × 32 B` per in-flight scan reply plus the decoded blocks loaded within the timeout. No crash is needed; liveness of that node is lost for as long as the requests continue.

**Recommendation**

- *Short term*: cap `blocks_limit` in `get_immutable_blocks` (`mantle.rs:662-663`) to the same `MAX_BLOCKS_STREAM_BLOCKS` the streaming endpoint enforces, or route `BlockRangeQuery` through the same validator; pass the capped value, not `2 ×`, as the scan limit, since the immutable index has at most one entry per slot.
- *Long term*: give `StorageMsg::ScanImmutableBlockIds` a hard server-side ceiling in the storage service itself, so no caller can request an unbounded scan, and move scans and per-block loads for HTTP and sync off the block-apply queue (LB-004, separate priority or a separate task).

**References**: parent #15 ("Range scans: are they bounded when the range comes from a network request or an HTTP query?"); issue #31's validator inventory for `BlocksStreamQuery` (`processed/31-serde-untrusted-input.md`, Query parameters row), which the legacy route lacks; #63 LB-001 for the exposure assumption.

### LB-003 · Every write uses `WriteOptions::default()` (`sync = false`) and no `bytes_per_sync`: an acknowledged block or recovery record survives a process crash but not a kernel crash or power loss

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Configuration |
| Target | `services/storage/src/rocksdb/mod.rs:377-383` (`store`), `:134-143` (`store_block_data`), `:406` (`bulk_store`), `:538-544` (`execute`); `RocksBackend::new` `:358-369` |
| Status | Open |

**Description**

Item 1 of the issue asks what durability the layer actually provides. Every write path ends in `rocks.put` (`mod.rs:382`) or `db.write(batch)` (`:140, :161, :197, :319, :406`). In rust-rocksdb 0.24.0 both forward to `*_opt(…, &WriteOptions::default())` (`src/db.rs:865-867`, `:1808-1814`), and the C++ defaults are `sync = false`, `disableWAL = false` (`options.h:2109, 2117`). Nothing sets `use_fsync`, `bytes_per_sync`, `wal_bytes_per_sync` or `manual_wal_flush` (`options.h:812, 1146, 1156, 1379`, all off).

What that gives, from the RocksDB comment on `sync` (`options.h:2093-2109`): each write is appended to the WAL with `write(2)` and is immediately visible to the process, so "if it is just the process that crashes… no writes will be lost"; "if the machine crashes, some recent writes may be lost." RocksDB issues `fdatasync` only when it creates or retires files (SST flush, MANIFEST edits, WAL rollover). The kernel writes dirty pages back within `vm.dirty_expire_centisecs` (30 s by default), so the exposure is the writes of the last roughly 30 s, or fewer if a flush happened in between.

Measured (Appendix B, test b). 200 rounds of `store_block_data` (one `WriteBatch` of a 100 KiB block, 4 KiB events, one parent and one index entry) followed by a 256 KiB recovery `store`, under `strace -f -c`:

```
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 50.78    0.035209          78       449           write
 48.43    0.033581        4197         8           fdatasync
  0.80    0.000553          79           7           fsync
------ ----------- ----------- --------- --------- ----------------
100.00    0.069343         149       464           total
AUDIT wrote 200 x (store_block_data 104 KiB batch + recovery put 256 KiB) in 442.2149ms
```

400 acknowledged writes (72 MB) produced 449 `write` calls and no per-write sync; the 8 `fdatasync` and 7 `fsync` calls belong to the database open (MANIFEST, directory) and to the single memtable flush that the 72 MB triggered (SST file, MANIFEST, WAL switch, directory). Between flushes, an acknowledged write exists only in the WAL's page-cache pages.

What the loss looks like. The default `wal_recovery_mode = kPointInTimeRecovery` (`options.h:1312`) replays the WAL up to the first torn or missing record and discards everything after it, so the surviving database is a prefix of the write sequence. Consequences that follow from the code:

1. A block's `store_block_data` batch is one `WriteBatch` (`mod.rs:134-143`), so the block, its parent link, its events and its immutable-index entries are lost or kept together; a crash cannot leave a block without its index (parent #15, question 1).
2. The chain service writes the block first and awaits the reply (`service/mod.rs:798-807`), then updates the recovery record through the state operator (`:233`, `operators.rs:61-66`, `recovery.rs:121-124`, a later `put` on the same WAL). A surviving recovery record therefore never points at a lost block. The reverse (blocks newer than the record) is the orphan case of #63 LB-005.
3. Every service that persists state does so through the same storage service and the same WAL (`RECOVERY_KEY_SUFFIX` producers: `cryptarchia`, `mempool`, `sdp`, `wallet`, `blend/core`, `pow`), so after a power loss all of them roll back to the same point. Inside the node the state is consistent; it is simply older.
4. What the node has already told others before the loss: `ProcessedBlockEvent` to API and FFI subscribers, `LibUpdate` to the broadcast service, the mempool and the PoW service, and any block it gossiped. After restart those blocks are re-fetched from peers; a finalized block is finalized by the network, not by this node's disk, so re-sync converges to the same chain.
5. What is actually lost: the PoW service's mined tickets. `PoWServiceState` keeps `ready_to_claim` and `pending_to_claim` "so the tickets survive restarts" (`services/pow/src/service.rs:326-340`); a ticket mined in the writeback window before a power loss is gone with the record, and its reward with it. Pending mempool transactions (re-gossiped by peers). The wallet's, SDP's and Blend's recent state (whether each re-derives it from the chain is not verified here).

**Exploit scenario**

Not attacker-triggered. A kernel panic or power loss on a node that mined a PoW ticket, or that finalized and announced blocks, within the last writeback window: the node restarts from an older recovery record, replays and re-syncs (PR #187 LB-002 covers the replay), and the ticket is not in `ready_to_claim` any more. The impact is bounded to that window and to the operator's own rewards; chain safety is unaffected because nothing in Cryptarchia relies on this node remembering what it wrote.

**Recommendation**

- *Short term*: decide the durability contract explicitly and write it down next to `RocksBackend::new`. If "process-crash durable" is the intended contract (it is a common choice for Nakamoto-style nodes), say so and document that PoW tickets and pending state can be lost on power loss. If the recovery record and PoW state should survive power loss, write those keys with `WriteOptions::set_sync(true)` (one `fdatasync` per recovery write, which is rare in Online mode) and leave block writes unsynced, or set `wal_bytes_per_sync` to bound the window.
- *Long term*: keep the durability contract testable: an `#[ignore]` test that writes, kills the process with `SIGKILL`, and reopens (process-crash durability), and a note in the deployment README about power-loss semantics.

**References**: RocksDB v10.4.2 `options.h:2093-2118` (`WriteOptions::sync`, `disableWAL`), `:1312` (`wal_recovery_mode`), `:812, 1146, 1156, 1379`; rust-rocksdb 0.24.0 `src/db.rs:865-867, 1808-1814`; #63 report (Section 5 table, "RocksDB options … Deferred to #15"); #188 LB-003; #636 (PoW claims that never settle) for the ticket lifecycle.

### LB-004 · Every RocksDB call except `bulk_store` runs inline on the single storage-service task; the block-apply path issues three to N+3 such calls per block

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Denial of Service / Timing |
| Target | `services/storage/src/lib.rs:97-99` (service loop); `services/storage/src/rocksdb/mod.rs:377-383, 412-414, 416-479, 481-524, 526-536, 538-544` (inline methods), `:385-410` (the one `spawn_blocking`), `:276-309` (`get_transactions`) |
| Status | Open |

**Description**

Item 3 of the issue asks which storage calls run on the tokio runtime on the hot block-apply path. The storage service is one task that receives and handles one message at a time (`lib.rs:97-99`), on the shared multi-thread runtime that Overwatch builds with the default worker count (`overwatch/runner.rs:70`, `utils/runtime.rs:9-15`, `service_runner.rs:170`). Inside it, the inventory at this commit is:

| `RocksBackend` method | RocksDB call | Where it runs |
|---|---|---|
| `store` (L377-383) | `put` | inline on the storage task (#188 LB-003) |
| `bulk_store` (L385-410) | `write(batch)` | `spawn_blocking` (the only one) |
| `load` (L412-414) | `get` | inline |
| `load_prefix` / `load_prefix_reverse` (L416-524) | iterator | inline |
| `remove` (L526-536) | `get` + `delete` | inline |
| `execute` (L538-544), used by `store_block_data` (L134-143), `remove_block` (L156-164), `store_immutable_block_ids` (L193-199), `remove_transactions` (L314-323) | `write(batch)` | inline |
| `get_transactions` (L276-309) | `get` per hash | inline on whichever task polls the stream: the mempool service (`tx-service/src/storage/adapters/rocksdb.rs:62`) or the HTTP `transaction` handler (`handlers.rs:1773`), outside the storage service and its metrics |
| `load_prefix_entries` (L95-113) | iterator | startup only, on the main thread before services start (`nodes/node/binary/src/lib.rs:187`) |

On the block-apply path (`services/chain/chain-service/src/service/mod.rs`) each applied block sends, in order: `StoreBlockData` (L798-807, awaited; one `write(batch)` of up to 1 MiB of block plus events and index entries); one `GetBlock` per newly canonical block other than the applied one, when INFO logging is on (L809-814, `:912-914`); one `GetBlock` per reorged block (L874-882, all queued at once through `join_all`); one `RemoveBlock` per stale block pruned by a LIB advance plus any left over from earlier failures (L227-231, `:1088-1100`, `api/mod.rs:268-275`; each a `get` then a `write(batch)`); and the recovery `Store` through the state operator (L233, #188 LB-003). That is three messages per ordinary block and `3 + reorged + stale` on a reorg or LIB advance, each executed inline.

Two things make "inline" cost more than the call's own latency. First, RocksDB write stalls: with the default `level0_slowdown_writes_trigger = 20` / `level0_stop_writes_trigger = 36` (`advanced_options.h:451, 458`), `max_background_jobs = 2` (`options.h:871`) and `no_slowdown = false` (`options.h:2128`), a `put` or `write` that arrives while level 0 is full sleeps inside RocksDB until a flush or compaction frees space; the #188 report's full-state recovery values (one memtable flush each) are exactly the workload that fills level 0 during IBD. That sleep happens on the storage task and on the runtime worker running it. Second, FIFO: every other producer shares the queue, so a sync-provider batch (up to `batch_size + 1 = 1001` index entries then one `GetBlock` per served block, `block_provider.rs:339-343, 404-431`), an HTTP scan (LB-002), a wallet `get_block` (`wallet/src/lib.rs:1605`) or an FFI subscription read (`c-bindings/src/api/subscriptions.rs:84`) sits between one block's `StoreBlockData` and the next.

**Exploit scenario**

A peer that requests sync batches continuously, or an HTTP client as in LB-002, keeps the queue full of reads; every block the node applies waits behind them, and a single write stall (organic, or induced by the #188 write pattern during IBD) pauses everything the node persists, including the reply that `process_block` is awaiting. The node falls behind the chain for as long as the pressure lasts. Difficulty is High because reaching a stall or a long queue needs either a large state or sustained traffic; the sync and HTTP paths are the levers.

**Recommendation**

- *Short term*: route `store`, `execute` and `remove` through `spawn_blocking` as `bulk_store` already does (#188 LB-003's recommendation, extended to the block batch), and give `get_transactions` its reads on the storage task or a blocking thread rather than the consumer's task.
- *Long term*: split the storage service into a write path (block-apply and recovery, bounded queue) and a read path (sync, HTTP, wallet, FFI, own queue and its own blocking pool), so reads cannot delay a block's write; set `WriteOptions::set_no_slowdown(true)` on latency-critical writes and surface `Incomplete` as a metric instead of sleeping inside RocksDB.

**References**: #188 LB-001 and LB-003 (measured 0.3 to 0.9 s inline `put` at 1 M UTXOs, and 12 to 21 % of IBD time stalled behind it); parent #15 ("Are RocksDB calls made on the tokio runtime directly? Which ones are on hot paths?"); RocksDB v10.4.2 `advanced_options.h:451-458`, `options.h:871, 2128`.

## 5. Suggestions (non-security)

### S-001 · Compression is compiled out, so nothing in the database is compressed

`Cargo.toml:262` declares `rocksdb = { default-features = false, version = "0.24" }` and `services/storage/Cargo.toml:26` enables only `bindgen-runtime`. rust-rocksdb's default feature set is `["snappy", "lz4", "zstd", "zlib", "bzip2", "bindgen-runtime"]` (its `Cargo.toml`), and `librocksdb-sys/build.rs:58-60` defines `SNAPPY` only under the `snappy` feature. RocksDB's default `compression` is `Snappy_Supported() ? kSnappyCompression : kNoCompression` (`options/options.cc:126`), so the node runs with `kNoCompression`. Measured (Appendix B, test a): the `OPTIONS-*` file records `compression=kNoCompression` and `bottommost_compression=kDisableCompressionOption`. Block bodies are mostly proofs and hashes, but the recovery records (#188: 360 B per UTXO, three copies of the UTXO set) are structured and would compress well; enable `lz4` (or `zstd` for the bottommost level) and set it in `RocksBackend::new`, or record that the omission is deliberate.

### S-002 · The RocksDB info log is unbounded and outside the shipped log rotation

`max_log_file_size = 0` ("all logs will be written to one log file", `options.h:927-930`) with `stats_dump_period_sec = 600` and every flush and compaction logged, so `<state>/db/LOG` grows for the life of the database; `deployment/systemd/logrotate-logos-blockchain-node.conf` rotates only the node's own log file. Set `set_max_log_file_size` / `set_keep_log_file_num` (or `set_log_level`) in `RocksBackend::new`.

### S-003 · The database is opened twice at startup, and `read_only: true` is accepted but makes the node unable to apply blocks

`load_recovery_data` opens the database in read-write mode to read the `recovery/` prefix (`nodes/node/binary/src/lib.rs:187`, `recovery.rs:36-39`) and drops it; `StorageService::init` opens it again (`lib.rs:68-73`). With `max_open_files = -1` each open walks every SST (LB-001), and the first open replays and, with `avoid_flush_during_recovery = false`, flushes the WAL, leaving one more SST per start. Consider reading the recovery prefix from the service's own handle, or opening once and handing the `Arc<DB>` to the service. Separately, `storage.backend.read_only: true` (`config/storage/serde.rs:15`) opens the DB with `open_for_read_only`, after which every `put`/`write` returns "Not supported operation in read only mode", `store_block_data` fails, and `process_block` returns `Error::Storage` for every block; nothing validates or warns about the combination. Reject `read_only` for the node binary or document what it is for.

---

## 6. Checked and ruled out

- **`load_prefix` with `limit: None` (issue item 3).** `StorageMsg::LoadPrefix` has no producer outside `services/storage/src/rocksdb/tests.rs`; `StorageApi` exposes no prefix method; both scan wrappers pass `Some(limit)` (`mod.rs:229, 252`); `handle_scan_immutable_block_ids*` take `NonZeroUsize` (`handlers.rs:362, 378`). The only unbounded prefix walk is `load_prefix_entries` over `recovery/` at startup (six keys at this commit). Not reachable with `None`; the derived-limit case is LB-002.
- **`StorageMsg::Execute`.** No producer outside the crate; `txn` is used only by the recovery tests. Arbitrary closures cannot reach the storage task from other services.
- **Write-batch atomicity (parent #15, question 1).** Block, parent link, events and immutable-index entries are one `WriteBatch` (`mod.rs:134-143`); `remove_block` deletes its three keys in one batch (`:156-164`). Holds.
- **WAL bound.** `max_total_wal_size = 0` resolves to `[sum of write_buffer_size × max_write_buffer_number] × 4` (`options.h:778-799`); with two column families (`default` and the unused `blocks`, #63 LB-006) that is `(64 MiB × 2) × 2 × 4 = 1 GiB`, and since only `default` is written the practical bound is the 64 MiB memtable. Not unbounded.
- **Write buffers.** `write_buffer_size = 64 MiB`, `max_write_buffer_number = 2`, `db_write_buffer_size = 0` (`options.h:188, 1077`; `advanced_options.h:175`): at most 128 MiB of memtables, plus the 32 MiB default block cache. A single value larger than the memtable (the #188 recovery record) is admitted and flushed immediately; that is #188 LB-002's territory.
- **Read-only open path.** `open_for_read_only(…, error_if_log_file_exist = false)` (`mod.rs:351, 356`) is correct for a reader coexisting with a writer; only the config foot-gun in S-003 applies.
- **`bulk_store` join.** `.expect("Failed to join the blocking task")` (`mod.rs:409`) fires only if the closure panics; `WriteBatch::put` and `DB::write` do not panic on errors. Not rated.
- **Column family `blocks`.** Created, never written (#63 LB-006). Its only effect here is the WAL formula above.

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

## Appendix B — Experiments

Three `#[ignore]` tests appended to `services/storage/src/rocksdb/tests.rs` in a scratch checkout of `3088316773684127ab4e06097838f7d16a1acc67` (no other change; the circuits build script's prebuilt download was pre-seeded into its cache directory because the sandbox's TLS interception is not trusted by `ureq`). All use `RocksBackendSettings { read_only: false, column_family: Some("blocks") }`, the node defaults from `nodes/node/binary/src/config/storage/serde.rs:19-26`, in a `tempfile::TempDir`.

- **(a) `audit_options_dump`**: `RocksBackend::new`, drop, then print selected `key = value` lines from the `OPTIONS-*` file RocksDB writes on open. Run: `cargo test -p logos-blockchain-storage-service audit_options_dump -- --ignored --nocapture`.
- **(b) `audit_write_path_syscalls`**: 200 × (`store_block_data` with a 100 KiB block, 4 KiB events, one parent and one immutable-index entry; then `store("recovery/cryptarchia", 256 KiB)`). Run under `strace -f -c -e trace=write,pwrite64,fsync,fdatasync,sync_file_range <test binary> audit_write_path_syscalls --ignored --nocapture`.
- **(c) `audit_fd_exhaustion`**: phase 1 opens a database with `Options::default()` plus `disable_auto_compactions` (so each `flush()` leaves one SST) and does one 16 KiB `put` + `flush()` per iteration until an operation fails, then closes it and counts SST files; phase 2 builds a 1,200-file database the same way but with `max_open_files = 64`, then opens it with `RocksBackend::new` and the node settings, then with `DB::open_cf` and `max_open_files = 256`. Run under `sh -c 'ulimit -n 1024; exec <test binary> audit_fd_exhaustion --ignored --nocapture'`.

Output (a), effective options with the node's settings (`OPTIONS-000007`, both column families identical; the `blocks` family is omitted):

```
[DBOptions] max_open_files = -1            max_file_opening_threads = 16
[DBOptions] max_total_wal_size = 0         WAL_ttl_seconds = 0   WAL_size_limit_MB = 0
[DBOptions] db_write_buffer_size = 0       max_background_jobs = 2   max_subcompactions = 1
[DBOptions] wal_recovery_mode = kPointInTimeRecovery   paranoid_checks = true
[DBOptions] use_fsync = false   bytes_per_sync = 0   wal_bytes_per_sync = 0   manual_wal_flush = false
[DBOptions] avoid_flush_during_recovery = false   avoid_flush_during_shutdown = false   atomic_flush = false
[DBOptions] track_and_verify_wals_in_manifest = false
[DBOptions] max_log_file_size = 0   keep_log_file_num = 1000   log_file_time_to_roll = 0
[DBOptions] info_log_level = INFO_LEVEL   stats_dump_period_sec = 600
[CFOptions "default"] write_buffer_size = 67108864   max_write_buffer_number = 2
[CFOptions "default"] target_file_size_base = 67108864   target_file_size_multiplier = 1   max_bytes_for_level_base = 268435456
[CFOptions "default"] level0_file_num_compaction_trigger = 4   level0_slowdown_writes_trigger = 20   level0_stop_writes_trigger = 36
[CFOptions "default"] soft_pending_compaction_bytes_limit = 68719476736   hard_pending_compaction_bytes_limit = 274877906944
[CFOptions "default"] compression = kNoCompression   bottommost_compression = kDisableCompressionOption
[CFOptions "default"] prefix_extractor = nullptr
[TableOptions/BlockBasedTable "default"] cache_index_and_filter_blocks = false
```

The database directory after the open holds `LOG` (44 KB already), `OPTIONS-000007`, `MANIFEST-000005`, `CURRENT`, `IDENTITY`, `LOCK` and an empty `000004.log`.
