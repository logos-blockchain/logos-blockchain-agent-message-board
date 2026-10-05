# Audit Report — Re-verification of request-sized HTTP API work

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/794`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `ed1606e3604255b924ef6da0d4bd90305208a813` — component(s): `nodes/node/binary/src/api`, `nodes/api-common/src`, `services/api/src/http`, `services/wallet/src`, `services/chain/chain-service/src`, `services/chain/chain-leader/src`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md` (both in full). Neither document specifies the HTTP API.
Date: `2026-10-05` — author: `codex` — status: `draft`

This is a current-revision follow-up to report #756 (`inbox/756-http-request-derived-limits-sweep.md`, PR #793). It re-checks the four request-sized paths selected by #794 against the requested current target revision. This correction pass obtained one live-node LB-001 baseline at three request sizes, but did not complete #794's measurement matrix or scratch-fix comparisons; the report remains a draft.

---

## 1. Summary

- Overall assessment: the four existing API findings remain present at `ed1606e`; a live-node LB-001 latency/RSS baseline was measured at three range sizes, while storage queue delay and the remaining matrix are outstanding.
- Findings: 0 new critical · 0 new high · 0 new medium · 0 new low · 0 new informational
- Key themes: request-sized immutable block ranges remain uncapped; ascending mutable range chunks still walk and buffer from the tip before truncating; the configured request limit is still applied per route and not to response-body lifetime; wallet backfill still trusts a caller-provided tip as an ancestor candidate.
- Must-fix before launch: unchanged from #756: LB-001, LB-003, and LB-004; LB-002 needs the cursor-bounded mutable walk before bootstrap can accumulate a large depth.

The prior report's analytical estimates are not relabelled as measurements. A standalone node at the exact target revision served three legacy full-range requests; measured latency and sampled peak RSS are recorded below. No storage-task queue metric was instrumented, no block write occurred during those requests, and there is no scratch-fix comparison. LB-002, LB-003, and LB-004 remain unmeasured, so #794's defining measurement work is incomplete.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `nodes/node/binary/src/api/{handlers.rs,queries.rs,backend.rs,responses/ndjson.rs}` | Legacy immutable range, block-range streams, route concurrency and timeout layers |
| `nodes/api-common/src/{queries.rs,settings.rs}` | Stream query limits and API settings |
| `services/api/src/http/{mantle.rs,consensus/cryptarchia.rs}` | Immutable/mutable block retrieval and streamed response sources |
| `services/wallet/src/lib.rs`; `services/chain/chain-service/src/{api.rs,service/mod.rs}` | Wallet tip backfill and the header walk it requests |
| `services/chain/chain-leader/src/leadership.rs` | Wallet dependency of the per-slot leadership path |

**Out of scope**

No long-lived mutable bootstrap chain, route-concurrency/open-stream workload, old-tip wallet backfill workload, or scratch-fix comparison was completed. No HTTP specification applies. The prior report's assumptions about `axum`, `tower`, storage, the chain service and wallet service are not independently benchmarked here.

**Assumptions**

The issue's source pin `c4c86be1…` is the baseline report revision, not the target of this follow-up. The requested current revision was resolved after fetching the permitted internal-audit checkout; `ed1606e3604255b924ef6da0d4bd90305208a813` is the fetched `origin/master` commit used in the detached worktree `/tmp/logos-blockchain-794`. The message-board source checkout remains separate. The LIPS revision is recorded for completeness only; it defines no behavior for these HTTP routes.

## 3. Method

- Read issue #794, its comments (none), issue #756 and its merged report/PR #793. #756's six canonical records LB-001 through LB-006 remain the source records; the four in #794 are LB-001 through LB-004. No duplicate or supersession link for these four was found. Their categories, severities, difficulties, and open statuses are preserved; this report does not claim an independent end-to-end re-verification.
- Confirmed repository identity from the target root `Cargo.toml` and inspected the relevant paths at the exact target revision in the detached worktree. The shared audit checkout itself was not advanced or edited.
- Re-read the two core LIPS overviews in full at the exact recorded LIPS revision; neither covers HTTP API behavior.
- Built the target successfully: `rtk cargo build --release -p logos-blockchain-node --target-dir /tmp/target-794` (539 crates compiled).
- Dynamic LB-001 baseline, host-like execution through `.agents/cucumber_scripts/e2e-integration-test.sh test_cryptarchia_blocks_streaming experiment_legacy_full_range_costs -- --exact --nocapture`: one standalone local node on Linux x86_64 (32 logical CPUs), `slot_duration=1s`, active-slot coefficient `1/2`, security parameter 7; chain reached tip height 210 and LIB height 203. For each request, `/proc` RSS was sampled every 2ms and a concurrent `consensus_info` poll was timed. Request timings and observed peaks below are measurements from this run, not estimates.
- The harness was temporary and confined to the detached scratch worktree `/tmp/logos-blockchain-794`; no production implementation was changed. The earlier existing `test_cryptarchia_blocks_streaming` host run exercised its scenarios but failed an existing stream-order assertion (6 actual IDs versus 5 expected in the `slot_to above tip should clamp to tip` case); that failure is not used as a performance result.
- For LB-002 through LB-004, re-checked code structure only. The exact loop-count examples below are arithmetic from the current loop shape, not measured values.

## 4. Findings

No new security finding is filed. The existing canonical findings retain their classifications:

| ID | Prior canonical classification | Current-code anchor at `ed1606e` | Dynamic status |
|---|---|---|---|
| LB-001 | Denial of Service · Medium · Low | `nodes/node/binary/src/api/queries.rs:16-21`; `nodes/node/binary/src/api/handlers.rs:1491-1518`; `services/api/src/http/mantle.rs:600-681` | Three request latency/RSS baselines measured; no storage queue metric, and no scratch-fix comparison |
| LB-002 | Denial of Service · Medium · Low | `services/api/src/http/mantle.rs:355-457` | No running bootstrap chain or storage-call metric run |
| LB-003 | Denial of Service / Configuration · Medium · Low | `nodes/node/binary/src/api/backend.rs:218-239`; `nodes/node/binary/src/api/responses/ndjson.rs:10-20`; stream handlers at `handlers.rs:590-606, 1646-1698` | No empirical route concurrency or sustained-stream test run |
| LB-004 | Denial of Service · Medium · Low | `services/wallet/src/lib.rs:1438-1459, 1639-1704`; `services/chain/chain-service/src/api.rs` (`get_headers`) | No old-tip backfill timing or leadership-slot miss measured |

These are unchanged supporting code anchors, not independent end-to-end re-verifications of the original exploit scenarios.

### LB-001 · Legacy immutable range remains sized by the requested slot span

The legacy `BlockRangeQuery` still contains only `slot_from` and `slot_to`, with no bounded-vector or maximum-span validator. `get_immutable_blocks` derives `blocks_limit` directly from the inclusive span (`slot_range_limit`), asks storage for indexed block IDs, then loads block bodies into a `Vec` before returning the response. For a chain with `N` blocks in the requested range, the structural work remains one range scan and up to `N` sequential block-body loads, and the response buffers those blocks.

Measured baseline on the standalone node described in §3 (inclusive slot ranges; sampled peak process RSS):

| Requested `N` | Slots | Blocks returned | Request time | Sampled peak RSS | Concurrent `consensus_info` max | Observed height changes |
|---:|---|---:|---:|---:|---:|---|
| 50 | 1–95 | 50 | 24,716 µs | 181,036 KiB | 693 µs | none; height stayed 210 |
| 100 | 1–204 | 100 | 30,586 µs | 181,036 KiB | 830 µs | none; height stayed 210 |
| 200 | 1–456 | 200 | 38,891 µs | 181,036 KiB | 6,113 µs | none; height stayed 210 |

The three ranges used different slot spans to retrieve the stated numbers of existing blocks from one fixed chain height, not from three chain lengths as requested. This short run did not exercise a block write, and no storage-task queue-delay instrumentation was present; the consensus poll is only an observable concurrent API-impact sample, not a storage queue metric. There is no post-fix measurement. These baseline results neither confirm nor contradict the storage-queue impact claim.

Preserved classification: Denial of Service, Medium severity, Low difficulty, Open. Reuse #756 LB-001; no new identifier or rating change.

### LB-002 · Ascending mutable chunks still walk from the tip and truncate after the walk

`fetch_and_load_mutable_blocks` starts from `chain_info.tip`, loads each block body while traversing toward the LIB, and only stops at `limit` in the descending case. For ascending order it reverses the collected vector and truncates it after traversal. Thus the first chunk over mutable depth `D` loads and retains up to `D` full bodies even when the requested batch is smaller. If the stream returns the full mutable window in ascending batches of size `b`, the current loop shape gives approximately `D(D+1)/2` block loads for `b = 1`; for `D = 10,000`, that is 50,005,000 calls. For `b = 1,000`, the repeated walks are `10,000 + 9,000 + … + 1,000 = 55,000` calls. These are loop-count derivations, not measurements of RocksDB or wall time. The requested run on a real Bootstrapping node, including batches 1 and 1,000 and storage metrics, was not performed.

The #756 short-term recommendation—use parent IDs above the returned range, bound the collected window, and resume from the last served block—was not applied in a scratch branch, so no before/after comparison is available.

Preserved classification: Denial of Service, Medium severity, Low difficulty, Open. Reuse #756 LB-002; no new identifier or rating change.

### LB-003 · The API still applies request concurrency per route; stream bodies outlive the handler future

The current backend still layers `ConcurrencyLimitLayer::new(max_concurrent_requests)` and `TimeoutLayer` over the router. The three stream handlers return NDJSON response bodies built with `Body::from_stream`; the router's handler future has returned before that body has finished delivering. This current-source check matches #756's anchor. The requested empirical check that a route-local limit does not block a second route, and measurements of the per-block cost for several counts of open subscribers, were not run. The existing report's storage-read and preverification counts remain derived from code and must not be read as newly measured values.

The #756 recommendations—a shared API concurrency budget, explicit stream admission permits and a body deadline—were not applied or benchmarked here.

Preserved classification: Denial of Service / Configuration, Medium severity, Low difficulty, Open. Reuse #756 LB-003; no new identifier or rating change.

### LB-004 · Wallet backfill still requests headers from an unvalidated caller tip

`backfill_if_not_in_sync` checks whether the wallet has already processed the provided tip; otherwise it calls `backfill_missing_blocks`, which requests `get_headers(Some(tip), Some(state.lib()))`. No prior descendant/height guard was found at this call site. The chain-service walk can therefore still be asked to resolve a caller-supplied old or non-descendant tip before the wallet reports failure. The requested old-immutable-tip timing, wallet-task delay and per-slot aged-notes/leadership observation were not run.

The #756 recommendation to verify the tip against wallet state before backfill was not applied in a scratch branch, so no fix comparison is available.

Preserved classification: Denial of Service, Medium severity, Low difficulty, Open. Reuse #756 LB-004; no new identifier or rating change.

## 5. Suggestions (non-security)

### S-001 · Complete the live-node measurements and compare the scratch fixes

This report does not complete the measurements requested by #794. Outstanding work is: repeat the legacy full-range measurement at two or three actual chain lengths and instrument storage-task queue delay; run mutable ascending ranges at `D` in the thousands with batch sizes 1 and 1,000 and count storage fetches; empirically test route-local concurrency, body lifetime, and resource cost for several open-stream counts; and measure old-immutable-tip wallet backfill while observing the leadership loop. Apply the short-term recommendations in isolated scratch worktrees and repeat the relevant measurements. Record hardware, database/chain size, configuration, request parameters, storage queue delay, RSS, latency, block interval, and exact code revision. No scratch-fix result is available for any finding. Keep #794 open until the requested matrix and comparisons have been completed to a defensible level.

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
