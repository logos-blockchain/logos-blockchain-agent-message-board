# Audit Report — Re-verification of request-sized HTTP API work

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/794`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `ed1606e3604255b924ef6da0d4bd90305208a813` — component(s): `nodes/node/binary/src/api`, `nodes/api-common/src`, `services/api/src/http`, `services/wallet/src`, `services/chain/chain-service/src`, `services/chain/chain-leader/src`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md` (both in full). Neither document specifies the HTTP API.
Date: `2026-10-06` — author: `codex` — status: `final`

This is a current-revision follow-up to report #756 (`inbox/756-http-request-derived-limits-sweep.md`, PR #793), covering the four existing canonical findings selected by #794. This correction pass adds live measurements at multiple depths/chain lengths and compares practical mitigations in a detached scratch worktree. No implementation change is proposed for `logos-blockchain`.

---

## 1. Summary

- Overall assessment: live measurements confirm request-sized scans, repeated mutable tip walks, route-local admission, stream bodies outliving request admission, and old-tip backfill; observed impact varies by workload, and the tested scratch mitigations reduce or bound the measured work.
- Findings: 0 new critical · 0 new high · 0 new medium · 0 new low · 0 new informational
- Key themes: request-sized immutable block ranges remain expensive at larger N; ascending mutable chunks repeat tip walks; route-local admission leaves independent routes available while stream bodies remain active; an old wallet tip causes a measurable ancestor walk.
- Must-fix before launch: unchanged from #756; no canonical status or classification is changed by these measurements.

Analytical estimates from #756 remain distinct from measurements. The live matrix uses short synthetic local chains and a constrained two-node setup; it does not claim production-scale impact. Baseline-versus-scratch results are reported for each practical mitigation. The small-chain LB-001 write sample did not show storage-write interference; this weakens that impact subclaim at the tested sizes, not the request-sized work finding.

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

Production-scale chain history and sustained adversarial traffic were not benchmarked. No HTTP specification applies. The measurements do not establish production incident likelihood, and the storage-write comparison has a small sample. The prior report's assumptions about third-party `axum`, `tower`, RocksDB and runtime internals are not independently audited here.

**Assumptions**

The issue's source pin `c4c86be1…` is the baseline report revision, not the target of this follow-up. The requested current revision was resolved after fetching the permitted internal-audit checkout; `ed1606e3604255b924ef6da0d4bd90305208a813` is the fetched `origin/master` commit used in the detached worktree `/tmp/logos-blockchain-794`. The message-board source checkout remains separate. The LIPS revision is recorded for completeness only; it defines no behavior for these HTTP routes.

## 3. Method

- Re-read issue #794, canonical report #756 / PR #793, and the current #800 report. The four canonical #756 records remain the source records; their IDs, classifications and Open status are preserved. No duplicate or supersession link for these four was found.
- Confirmed repository identity from the target root `Cargo.toml` and inspected the relevant paths at the exact target revision in the detached worktree. The shared audit checkout itself was not advanced or edited.
- Re-read the two core LIPS overviews in full at the exact recorded LIPS revision; neither covers HTTP API behavior.
- Dynamic runs used the detached `/tmp/logos-blockchain-794` worktree at the exact pinned source revision and the repo-local `.agents/cucumber_scripts/e2e-integration-test.sh` wrapper in host-like execution. Hardware/host: AMD Ryzen 9 9950X (16 cores / 32 threads), 48,113,768 KiB reported memory, WSL2 Linux `hansie-pc2`, kernel 6.6.87.2. `slot_duration=1s` unless noted; node binaries were release builds.
- LB-001 online matrix: two local nodes, `security_parameter=7`, active-slot coefficient `0.99`, request concurrency configured to 2. Full available immutable history was requested at actual heights 50, 100, and 200. Process RSS was sampled during each request. Storage writes were timed through the instrumented storage task in a quiet interval and while the full-range load was active. The scratch mitigation imposed a 100-slot range cap.
- LB-002: two-node Bootstrapping target, `security_parameter=10,000`, active-slot coefficient `0.99`, prolonged bootstrap 3,600 seconds; source node stopped at each snapshot. The target's mutable depth was measured at D=1,596 and D=3,201. Scratch counters in the API path counted mutable fetches and each `GetBlock` call. A cursor-bounded scratch implementation was compared at D≈1,600 with batch sizes 1 and 1,000.
- LB-003/LB-004: the online matrix instrumented active route handlers, response-stream events and storage reads, per-slot aged-note leadership queries, wallet backfill and header-walk counts. For route-local admission, two simultaneous range requests filled the configured route limit while `/version` was probed. A scratch global semaphore used the same middleware path. Stream runs held S=1,4,12 subscribers open for over 30 seconds; scratch body timeout was reduced to 10 seconds. Wallet backfill used a 1,000-block chain and an old immutable tip at height 500; scratch early-tip validation was enabled for the repeat.
- All temporary endpoints, test counters, instrumentation and scratch fixes were confined to the detached worktree. The prior LB-001 one-chain baseline and D=1,596 LB-002 baseline are retained; exact experiments were not repeated except where necessary for a controlled scratch comparison.
- Additional host-like integration tests completed: `api_cost_mutable_depth_scaling_experiment` (D=3,201), `api_cost_mutable_cursor_scratch_comparison` (baseline/fix at D=1,601), `api_cost_online_matrix_experiment` (LB-001, route concurrency, streams and first wallet baseline), and `api_cost_wallet_backfill_deep_experiment` (LB-004 at chain height 1,000). They were invoked through `.agents/cucumber_scripts/e2e-integration-test.sh` in the detached worktree.

## 4. Findings

No new security finding is filed. The existing canonical findings retain their classifications:

| ID | Prior canonical classification | Current-code anchor at `ed1606e` | Dynamic status |
|---|---|---|---|
| LB-001 | Denial of Service · Medium · Low | `nodes/node/binary/src/api/queries.rs:16-21`; `nodes/node/binary/src/api/handlers.rs:1491-1518`; `services/api/src/http/mantle.rs:600-681` | Multi-height scan/RSS and overlapping-write sample; range-cap scratch comparison |
| LB-002 | Denial of Service · Medium · Low | `services/api/src/http/mantle.rs:355-457` | Two baseline depths and both batch sizes; cursor-bounded scratch comparison at D≈1,600 |
| LB-003 | Denial of Service / Configuration · Medium · Low | `nodes/node/binary/src/api/backend.rs:218-239`; `nodes/node/binary/src/api/responses/ndjson.rs:10-20`; stream handlers at `handlers.rs:590-606, 1646-1698` | Route independence and S=1/4/12 live streams; shared-admission and body-timeout scratch comparisons |
| LB-004 | Denial of Service · Medium · Low | `services/wallet/src/lib.rs:1438-1459, 1639-1704`; `services/chain/chain-service/src/api.rs` (`get_headers`) | 500-parent live wallet backfill and early-tip-rejection scratch comparison |

These are still the canonical #756 records and classifications. Measurements below assess the described mechanisms at the tested configurations; they do not prove production-scale exploitability or justify reclassification.

### LB-001 · Legacy immutable range remains sized by the requested slot span

`get_immutable_blocks` derives its scan limit from the inclusive slot span and buffers retrieved block bodies. The earlier one-chain, response-size baseline is retained: 50/100/200 returned blocks took 24,716/30,586/38,891 µs, with sampled peak RSS 181,036 KiB. The current live matrix varied actual chain height and requested the full available immutable history:

| Chain height / LIB height | Requested slot range | Immutable blocks returned | Request time | Sampled peak RSS |
|---:|---|---:|---:|---:|
| 50 / 43 | 1–47 | 43 | 4,271 µs | 219,668 KiB |
| 100 / 93 | 1–104 | 93 | 9,710 µs | 251,924 KiB |
| 200 / 193 | 1–227 | 193 | 21,243 µs | 331,540 KiB |

At these small sizes, measured latency and RSS rise with available returned history. This is request-sized scaling evidence, not an extrapolation to production history.

For storage impact, the instrumented storage task completed 9 quiet-window writes at mean 97 µs / max 135 µs. During the large-range workload, 5 writes completed at mean 93 µs / max 114 µs; queue depth did not build in this sample. This does not show material write interference at the tested chain sizes. A scratch 100-slot range cap rejected the same 193-block request in 672 µs with no returned blocks; during up to 8 seconds of capped-load probing, 4,716 requests were rejected and 4 writes completed at mean 77 µs / max 88 µs with no active legacy scan. The scratch cap bounds the expensive path but currently responds with HTTP 500, so it is a workload-control demonstration rather than a polished API error policy.

Assessment: the range-sized scan/RSS mechanism is confirmed, while storage-write interference is not significant in this small controlled sample and remains unestablished at larger N. The tested cap prevents the measured scan. Preserved classification: #756 LB-001, Denial of Service · Medium · Low · Open; no reclassification proposed.

### LB-002 · Ascending mutable chunks still walk from the tip and truncate after the walk

`fetch_and_load_mutable_blocks` starts from `chain_info.tip`, loads bodies while traversing toward the requested lower bound, and truncates after traversal in ascending mode. The prior `O(D²/b)` shape and `D(D+1)/2` examples remain source-derived estimates, separate from the live measurements.

The live baseline used a two-node local chain at the exact target revision on Linux x86_64 (32 logical CPUs), with one-second slots, active-slot coefficient 0.99, security parameter 10,000, and the target held in Bootstrapping. At the measured snapshot, the target had height 1,596, tip slot 1,756, LIB slot 0, and a 1,596-block tip-to-LIB parent walk. Node-0 was stopped after the snapshot. Requests were ascending over slots 1–1,756 with `blocks_limit=1,596` and `BlockFilter::MutableAndImmutable`:

| `server_batch_size` | Events/blocks returned | Server mutable-fetch requests | Instrumented `GetBlock` observations | Request + body elapsed |
|---:|---:|---:|---|---:|
| 1 | 1,596 | 1,596 | Per-request counter began at 1,596 (the first two records were 1,596) and decreased with the ascending cursor to 2 on the last recorded request; aggregate not retained. | 79,853 ms |
| 1,000 | 1,596 | 2 | First request: 1,596 calls; second request: 597 calls; total 2,193 instrumented calls. | 188 ms |

The second baseline depth, D=3,201 (height 3,201, LIB slot 0), covered the same ascending mutable range:

| `server_batch_size` | Returned | Fetch requests | `GetBlock` calls | Elapsed |
|---:|---:|---:|---:|---:|
| 1 | 3,201 | 3,201 | 5,128,001 | 319,128 ms |
| 1,000 | 3,201 | 4 | 6,807 | 467 ms |

Doubling D from 1,596 to 3,201 increased the batch-1 measured time from 79.9 s to 319.1 s (about 4×), consistent with repeated suffix traversal rather than a one-off timing artifact. Batch size 1,000 reduced the work substantially at both depths.

The #756 cursor-bounded mitigation was applied only in scratch and measured against the same D=1,601 snapshot, with `server_batch_size` 1 and 1,000:

| Batch | Run | Fetch requests | `GetBlock` calls | Elapsed | Change vs baseline |
|---:|---|---:|---:|---:|---|
| 1 | Baseline | 1,601 | 1,284,001 | 79,795 ms | — |
| 1 | Cursor-bounded scratch | 1,601 | 3,202 | 277 ms | 99.75% fewer calls; 99.65% less elapsed time |
| 1,000 | Baseline | 2 | 2,203 | 162 ms | — |
| 1,000 | Cursor-bounded scratch | 2 | 1,601 | 128 ms | 27.3% fewer calls; 21.0% less elapsed time |

The scratch walk traverses the mutable ID chain once and resumes from a cursor; the batch-1 result changes the repeated suffix work from quadratic to approximately linear fetch work in this test. Assessment: the baseline repeated walk and depth-squared timing trend are confirmed; the cursor mitigation materially eliminates the repeated traversal at this depth. The comparison uses one controlled scratch depth and is not a production-scale benchmark.

Preserved classification: #756 LB-002, Denial of Service · Medium · Low · Open; no reclassification proposed.

### LB-003 · The API still applies request concurrency per route; stream bodies outlive the handler future

The router's configured request limit was set to 2 for this controlled run. Two concurrent `/cryptarchia/blocks` handlers reached active count 2; a concurrent `/version` request returned 200 in 621 µs, empirically showing independent route budgets. In a scratch repeat with a shared/global admission semaphore of 2, `/version` took 13,441 µs while the two range requests occupied the budget. Route-local isolation is confirmed; the shared-admission mitigation changes the observed behavior as expected.

With the baseline request middleware, open body streams remained active beyond 30 seconds after the handler work: minimum last-event times were 35.335 s (S=1), 35.940 s (S=4), and 38.954 s (S=12). All 1/4/12 streams opened; no failures. Stream `GetBlock`/event counts were 29/29, 112/112, and 336/336 respectively. RSS deltas from each run's start were +9,216 KiB, +16,128 KiB and +23,552 KiB; `/version` remained responsive (737–868 µs). The increasing delivery count and resource delta support per-subscriber work, though sequential run RSS is cumulative and these are not isolated per-subscriber memory estimates.

With a scratch 10-second body deadline, all S=1/4/12 streams opened but last events arrived around 9.6–10.0 seconds; counts were 5, 36 and 84, respectively. `/version` remained below 1 ms. This confirms the body can outlive the request-layer timeout in baseline and shows that an explicit body deadline bounds stream lifetime. No separate subscriber-count cap was tested.

Preserved classification: #756 LB-003, Denial of Service / Configuration · Medium · Low · Open; no reclassification proposed.

### LB-004 · Wallet backfill still requests headers from an unvalidated caller tip

At chain height 1,000 / LIB height 993, the request supplied an old immutable tip at height 500 (501 headers behind the current state, substantially deeper than the earlier shallow run). The chain-service backfill recorded one walk and 501 headers; measured backfill/header-walk time was 4 ms, and the wallet API request completed with an error in 26 ms (expected because the supplied tip was not accepted as a wallet ancestor). The aged-note leadership-query counter was 1,317 both before and after this 26 ms interval; maximum slot-gap remained 2. No per-slot leadership query overlapped this short walk, so no query delay or missed slot was observed at the tested depth. The measured work did not occupy the wallet service long enough to overlap the one-second slot cadence.

In scratch, early validation rejected the same supplied tip before walking: 0 backfill calls, 0 headers, 0 header-walk calls, request under 1 ms. The mitigation avoided the measured work. Thus the input-driven walk is confirmed, but the claimed per-slot leadership impact is not demonstrated: the observed one-shot 500-parent backfill was short and did not overlap a leadership query. This weakens the single-request impact claim at the tested depth/host without contradicting possible queueing under deeper or repeated requests. No reclassification is proposed.

Preserved classification: #756 LB-004, Denial of Service · Medium · Low · Open.

## 5. Suggestions (non-security)

### S-001 · Apply bounded work and explicit stream admission consistently

The measurements support short-term limits on request span, mutable traversal work, shared API admission, response-body lifetime, and wallet tip validation. The scratch tests demonstrate these control points but are not production fixes and were not pushed. Before implementation, retain a client-appropriate error response for overlarge ranges and decide whether stream admission should include an explicit subscriber cap in addition to a body deadline.

### Baseline / scratch-fix summary

| Finding | Baseline measurement | Scratch mitigation | Result |
|---|---|---|---|
| LB-001 | Full immutable history at heights 50/100/200 returned 43/93/193 blocks in 4.271/9.710/21.243 ms; 5 overlapping writes averaged 93 µs vs 97 µs quiet | 100-slot cap rejected the 193-block request in 672 µs | Request-sized scan confirmed; no material write impact observed at tested sizes; cap prevented scan work |
| LB-002 | At D=1,601, batch 1: 1,284,001 `GetBlock` calls / 79.795 s; at D=3,201: 5,128,001 / 319.128 s | Cursor-bounded at D=1,601: batch 1 fell to 3,202 calls / 277 ms; batch 1,000 fell from 2,203 to 1,601 calls | Repeated tip walk confirmed and materially improved in scratch |
| LB-003 concurrency | Two A handlers filled configured route limit 2; `/version` completed in 621 µs | Shared admission made the same B probe wait 13.441 ms | Route budgets were empirically independent; shared budget changes behavior |
| LB-003 streams | S=1/4/12 bodies remained active 35.3/35.9/39.0 s; delivery counts 29/112/336; RSS deltas +9/16/23 MiB | 10 s body timeout limited last events to about 9.6–10.0 s and reduced deliveries | Post-handler stream lifetime confirmed; deadline bounds lifetime; no explicit subscriber cap tested |
| LB-004 | Height 1,000, old tip 500: 501 headers, 4 ms backfill, 26 ms request; leader-query count and max slot gap unchanged | Early validation: 0 headers/walks, under 1 ms | Backfill confirmed; no leadership overlap/miss observed for this single tested request; scratch avoids the walk |

LB-001 write timing is a small controlled sample; LB-004 did not overlap a per-slot leadership query. These are the main residual limitations; the primary checklist items were exercised and each practical short-term mitigation was compared.

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
