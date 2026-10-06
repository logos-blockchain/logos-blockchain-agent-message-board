# Audit Report — Re-verification of request-sized HTTP API work

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/794`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `ed1606e3604255b924ef6da0d4bd90305208a813` — component(s): `nodes/node/binary/src/api`, `nodes/api-common/src`, `services/api/src/http`, `services/wallet/src`, `services/chain/chain-service/src`, `services/chain/chain-leader/src`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md` (both in full). Neither document specifies the HTTP API.
Date: `2026-10-06` — author: `codex` — status: `draft`

This is a current-revision follow-up to report #756 (`inbox/756-http-request-derived-limits-sweep.md`, PR #793). It re-checks the four request-sized paths selected by #794 against the requested current target revision. This correction pass adds a live Bootstrapping-node LB-002 baseline with per-request storage-fetch instrumentation. It does not complete #794's full measurement matrix or any scratch-fix comparisons; the report remains a draft.

---

## 1. Summary

- Overall assessment: the four existing API findings remain present at `ed1606e`; live-node LB-001 latency/RSS and LB-002 mutable-range baselines are measured, while storage queue delay and the remaining matrix are outstanding.
- Findings: 0 new critical · 0 new high · 0 new medium · 0 new low · 0 new informational
- Key themes: request-sized immutable block ranges remain uncapped; ascending mutable range chunks still walk and buffer from the tip before truncating; the configured request limit is still applied per route and not to response-body lifetime; wallet backfill still trusts a caller-provided tip as an ancestor candidate.
- Must-fix before launch: unchanged from #756: LB-001, LB-003, and LB-004; LB-002 needs the cursor-bounded mutable walk before bootstrap can accumulate a large depth.

The prior report's analytical estimates are not relabelled as measurements. A standalone node at the exact target revision served three legacy full-range requests; measured latency and sampled peak RSS are recorded below. A second two-node run measured an ascending mutable range on a Bootstrapping target, including actual per-request `GetBlock` counters at batch sizes 1 and 1,000. No storage-task queue metric was instrumented, no block write overlapped the LB-001 requests, LB-003 and LB-004 remain unmeasured, and no scratch-fix comparison was run. #794's defining measurement work is therefore incomplete.

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

No route-concurrency/open-stream workload, old-tip wallet backfill workload, or scratch-fix comparison was completed. LB-002 used a bounded live chain rather than a production-length bootstrap history. No HTTP specification applies. The prior report's assumptions about `axum`, `tower`, storage, the chain service and wallet service are not independently benchmarked here.

**Assumptions**

The issue's source pin `c4c86be1…` is the baseline report revision, not the target of this follow-up. The requested current revision was resolved after fetching the permitted internal-audit checkout; `ed1606e3604255b924ef6da0d4bd90305208a813` is the fetched `origin/master` commit used in the detached worktree `/tmp/logos-blockchain-794`. The message-board source checkout remains separate. The LIPS revision is recorded for completeness only; it defines no behavior for these HTTP routes.

## 3. Method

- Read issue #794, its comments (none), issue #756 and its merged report/PR #793. #756's six canonical records LB-001 through LB-006 remain the source records; the four in #794 are LB-001 through LB-004. No duplicate or supersession link for these four was found. Their categories, severities, difficulties, and open statuses are preserved; this report does not claim an independent end-to-end re-verification.
- Confirmed repository identity from the target root `Cargo.toml` and inspected the relevant paths at the exact target revision in the detached worktree. The shared audit checkout itself was not advanced or edited.
- Re-read the two core LIPS overviews in full at the exact recorded LIPS revision; neither covers HTTP API behavior.
- Built the target successfully: `rtk cargo build --release -p logos-blockchain-node --target-dir /tmp/target-794` (539 crates compiled).
- Dynamic LB-001 baseline, host-like execution through `.agents/cucumber_scripts/e2e-integration-test.sh test_cryptarchia_blocks_streaming experiment_legacy_full_range_costs -- --exact --nocapture`: one standalone local node on Linux x86_64 (32 logical CPUs), `slot_duration=1s`, active-slot coefficient `1/2`, security parameter 7; chain reached tip height 210 and LIB height 203. For each request, `/proc` RSS was sampled every 2ms and a concurrent `consensus_info` poll was timed. Request timings and observed peaks below are measurements from this run, not estimates.
- Dynamic LB-002 baseline, host-like execution through `.agents/cucumber_scripts/e2e-integration-test.sh test_cryptarchia_blocks_streaming api_cost_mutable_ascending_bootstrap_experiment -- --exact --nocapture`, on the same Linux x86_64 host (32 logical CPUs). Two nodes used `slot_duration=1s`, active-slot coefficient `99/100`, and security parameter 10,000; node-0 was online and node-1 had a 3,600s prolonged-bootstrap period. The node was built at the exact target revision with scratch-only instrumentation in `fetch_and_load_mutable_blocks` that counted each `storage.get_block` call and timed that walk. At the snapshot, node-1 was `Bootstrapping`, tip height 1,596 / slot 1,756, LIB slot 0, and the API parent walk counted 1,596 mutable blocks. Node-0 was stopped to hold the measured tip constant before the two range runs. The successful integration test took 1,844.95s wall time, mostly chain production/synchronization.
- The first LB-002 pass used an older release binary and measured timing only; after detecting that its timestamp predated the scratch instrumentation, the exact-target release node was rebuilt with `rtk cargo build --release -p logos-blockchain-node` and the same host wrapper was rerun. Only the rebuilt-binary run is used for the `GetBlock` counter results below. The instrumented per-request records were printed in full by the temporary harness and confirm request/counter ranges; the harness output itself did not persist an aggregate sum after its temporary log directory was cleaned up, so no aggregate `GetBlock` total is claimed for batch size 1.
- The harness was temporary and confined to the detached scratch worktree `/tmp/logos-blockchain-794`; no production implementation was changed. The earlier existing `test_cryptarchia_blocks_streaming` host run exercised its scenarios but failed an existing stream-order assertion (6 actual IDs versus 5 expected in the `slot_to above tip should clamp to tip` case); that failure is not used as a performance result.
- For LB-003 and LB-004, re-checked code structure only. The LB-002 analytical loop-count examples below remain arithmetic from the source shape; the one measured depth does not validate extrapolations to other chain depths.

## 4. Findings

No new security finding is filed. The existing canonical findings retain their classifications:

| ID | Prior canonical classification | Current-code anchor at `ed1606e` | Dynamic status |
|---|---|---|---|
| LB-001 | Denial of Service · Medium · Low | `nodes/node/binary/src/api/queries.rs:16-21`; `nodes/node/binary/src/api/handlers.rs:1491-1518`; `services/api/src/http/mantle.rs:600-681` | Three request latency/RSS baselines measured; no storage queue metric, and no scratch-fix comparison |
| LB-002 | Denial of Service · Medium · Low | `services/api/src/http/mantle.rs:355-457` | One live Bootstrapping baseline measured; per-request fetch counters confirm the repeated tip walk, but no scratch-fix comparison or multi-depth scaling run |
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

`fetch_and_load_mutable_blocks` starts from `chain_info.tip`, loads each block body while traversing toward the requested lower bound, and only stops at `limit` in the descending case. For ascending order it reverses the collected vector and truncates it after traversal. Thus each ascending chunk can revisit the tip-to-cursor suffix even when the requested batch is small. The prior `D(D+1)/2` and `D + (D-b) + …` examples for `D = 10,000` remain source-derived loop-count estimates, not measurements of RocksDB or wall time.

The live baseline used a two-node local chain at the exact target revision on Linux x86_64 (32 logical CPUs), with one-second slots, active-slot coefficient 0.99, security parameter 10,000, and the target held in Bootstrapping. At the measured snapshot, the target had height 1,596, tip slot 1,756, LIB slot 0, and a 1,596-block tip-to-LIB parent walk. Node-0 was stopped after the snapshot. Requests were ascending over slots 1–1,756 with `blocks_limit=1,596` and `BlockFilter::MutableAndImmutable`:

| `server_batch_size` | Events/blocks returned | Server mutable-fetch requests | Instrumented `GetBlock` observations | Request + body elapsed |
|---:|---:|---:|---|---:|
| 1 | 1,596 | 1,596 | Per-request counter began at 1,596 (the first two records were 1,596) and decreased with the ascending cursor to 2 on the last recorded request; all requests emitted an instrumented record. Aggregate sum was not retained by the temporary harness. | 79,853 ms |
| 1,000 | 1,596 | 2 | First request: 1,596 calls; second request: 597 calls; total 2,193 instrumented calls. | 188 ms |

The large elapsed-time difference and per-request counter progression directly confirm repeated tip walks for batch size 1 at this measured depth. The batch-1,000 run makes two walks rather than one bounded walk; measured calls were 1,596 + 597. This single depth does not establish a general scaling curve across `D`, and no aggregate total for batch size 1 is claimed because its per-request trace was not retained after test completion. The first timing-only pass, before rebuilding the instrumented node, returned the same 1,596 events in 78,846 ms (batch 1) and 184 ms (batch 1,000); the rebuilt-binary values above are the counter-enabled run used for the finding assessment.

The #756 short-term recommendation—use parent IDs above the returned range, bound the collected window, and resume from the last served block—was not applied in a scratch branch, so no before/after comparison is available. Assessment: the repeated-tip-walk claim is confirmed at this depth; the predicted behavior after the short-term fix is unmeasured, and the broader `O(D²/b)` scaling claim remains only partially validated because no second depth or scratch-fix run was measured.

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

This report does not complete the measurements requested by #794. Outstanding work is: repeat the legacy full-range measurement at two or three actual chain lengths and instrument storage-task queue delay while a block write overlaps; apply the LB-002 cursor-bounded recommendation in a scratch worktree and repeat the measured range; empirically test route concurrency, stream lifetime and resource cost for several open-subscriber counts; measure old-immutable-tip wallet backfill while observing wallet-task delay and leadership slots; and run the relevant short-term recommendations as scratch-fix comparisons. The LB-002 batch-1 aggregate fetch total was not retained, and no second depth was measured. Record hardware, database/chain size, configuration, request parameters, storage queue delay, RSS, latency, block interval, and exact code revision. No scratch-fix result is available for any finding. Keep #794 open until the requested matrix and comparisons have been completed to a defensible level.

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
