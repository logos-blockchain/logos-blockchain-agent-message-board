# Audit Report — Debug and profiling HTTP routes: parameter validation and shipped-build exposure

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/208` (parent #24; overlaps API parent #18)
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `nodes/api-common` (`pprof.rs`, `lib.rs`, `Cargo.toml`), `nodes/node/binary` (`src/api/backend.rs`, `src/config/api/serde.rs`, `src/global_allocators/`, `src/panic.rs`, `Cargo.toml` features), `services/tracing` (`console.rs`, `lib.rs`), the build and deployment surface (`Dockerfile`, `deployment/`, `flake.nix`, `.github/workflows/`, `tests/testing_framework/src/framework/local/provisioning.rs`, `tests/testing_framework/src/node/cfgsync.rs`)
Date: `2026-09-12` — author: `claude-fable-5.1` — status: `final`

Deployment repositories checked alongside the node: `logos-blockchain/logos-blockchain-testing` @ `1b336c2c080715b0873bdb5b36bd526dd0e74baf` and `logos-blockchain/logos-blockchain-module` @ `f787d0413511e0815eaa405f74cc6998c17a485d` (local checkouts, read only). Third-party sources read at the pinned versions: `pprof` 0.15.0 (`src/timer.rs`, `src/profiler.rs`, `src/lib.rs`), `axum` 0.7.9 (`src/docs/routing/layer.md`), `tokio` 1.52.3 (`src/time/sleep.rs`).

No protocol specification applies to this issue: the routes are an operational surface of the node binary, not part of the protocol. The two logos-lips overviews were not re-read for this round.

---

## 1. Summary

- Overall assessment: no artefact the node repository ships carries the `/debug/pprof/profile` route, but the route is one opt-in flag away in the testing repository's bundle builder, which also binds the API to every interface and prints the profiling URL per node; the handler itself passes both numeric query parameters to C and OS APIs unvalidated and sits outside the API's timeout, concurrency and metrics middleware. The panic from PR #206 is reproduced at handler level, and a patch that turns every bad input into a 400 (and a concurrent profile into a 409) is verified with the same tests.
- Findings: `0` critical · `0` high · `1` medium · `2` low · `1` informational
- Key themes: "feature-gated routes join the unauthenticated listener", "parameters crossing into `setitimer` unchecked", "profiling router merged after the middleware stack", "testing-framework deployments expose the route on `0.0.0.0`"
- Must-fix before launch: none. LB-001 plus the LB-002 bounds should land before any `profiling` build is run with a reachable API port, which today means any bundle built with `--features profiling` through `logos-blockchain-testing`.
- Follow-ups filed: #261 (upstream the patch and tests, CI job; under #18), #260 (loopback-only debug listener, `tokio-console` bind check; under #18), #262 (testing-repo bundle builder and cfgsync; under #26).

Status of the four task-list items of #208 at this commit:

| Item | Result |
|---|---|
| 1. Shipped builds with `profiling` / `release-profiling` | **Ruled out for every shipped artefact** (§4.0). `release-profiling` is used by the node's own testing framework for `tokio-console` builds only, which do not include the pprof route. The testing repository's bundle builder accepts `--features profiling` as an opt-in and its config generator binds the API to `0.0.0.0` (LB-003). |
| 2. Inventory of gated routes and parameters reaching C/OS APIs | Done (§4.0, table). One HTTP route is feature-gated; its two numeric parameters reach `setitimer(2)` and `tokio::time::sleep` unvalidated. The other gated surfaces (`tokio-console`, `dhat-heap`, `jemalloc`) take no request parameters. |
| 3. Prototype fix | Done (Appendix B): bounds on `frequency` and `seconds` returning 400, an in-flight guard returning 409, and a design for a loopback-only listener (§LB-002 recommendation). |
| 4. Verify `frequency=0` reproduces and the patch returns 400 | Done at handler level (Appendix C): the unmodified handler panics with `attempt to divide by zero`; the patched handler returns 400 / 409 and still serves a valid profile. A full `--features profiling` node build was not run (see §3). |

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `nodes/api-common/src/pprof.rs`, `src/lib.rs`, `Cargo.toml` | the handler, its parameters, the `profiling` feature and the Windows guard |
| `nodes/node/binary/src/api/backend.rs` | where the profiling router is merged and which middleware applies to it |
| `nodes/node/binary/Cargo.toml` (`[features]`), `src/global_allocators/`, `src/main.rs`, `src/panic.rs` | the `profiling`, `tokio-console`, `dhat-heap`, `jemalloc` features and what each compiles in |
| `nodes/node/binary/src/config/api/serde.rs`, `nodes/node/standalone-node-config.yaml`, `tools/config/src/node.rs` | default API bind address |
| `services/tracing/src/console.rs`, `src/lib.rs`, `nodes/node/binary/src/config/tests.rs` | the `tokio-console` endpoint and its config |
| `Cargo.toml` (`[profile.release-profiling]`), `.cargo-deny.toml` | the profiling profile and the recorded assumption that no release build enables `pprof` |
| `Dockerfile`, `deployment/Dockerfile`, `deployment/compose*.yml`, `deployment/.env.*`, `deployment/systemd/`, `flake.nix`, `.github/workflows/*.yml`, `tests/testing_framework/assets/runtime/Dockerfile.*`, `README.md`, `tests/README.md` | every build command the node repository runs or documents |
| `tests/testing_framework/src/framework/local/provisioning.rs`, `src/node/cfgsync.rs` | the node repository's own testing framework: build profiles and API bind rewriting |
| `logos-blockchain-testing` @ `1b336c2c` | `scripts/build/build-bundle.sh`, `scripts/build/build-linux-binaries.sh`, `testing-framework/assets/stack/scripts/docker/prepare_binaries.sh`, `testing-framework/tools/cfgsync/src/{config/builder.rs,server.rs}`, `testing-framework/deployers/{compose,k8s}/src/deployer/orchestrator.rs`, `.github/workflows/` |
| `logos-blockchain-module` @ `f787d041` | `flake.nix`, `justfile`, `CMakeLists.txt`: how the module obtains the node library |
| `pprof` 0.15.0 | `Timer::new`, `ProfilerGuardBuilder::build`, `Profiler::{start,stop}`, `perf_signal_handler`, `MAX_DEPTH` |

**Out of scope**

- The contents of published container images. Listing the organisation's packages needs the `read:packages` scope, which the reviewing token lacks (`gh api /orgs/logos-blockchain/packages` returned 403). The conclusion for images rests on the Dockerfiles and the publishing workflow, not on inspecting a pulled image.
- Kubernetes manifests and Helm values in `logos-blockchain-testing` beyond the orchestrator that prints the profiling URLs.
- The authentication layer design for the HTTP API (#119, PR #130): this report only says where the debug routes should sit relative to it.
- `pprof` internals other than the timer, the profiler lifecycle and the signal handler; `inferno` (flamegraph SVG writer) and the `quick-xml` advisories recorded in `.cargo-deny.toml`; `console-subscriber` internals; `axum`, `tower-http`, `tokio` beyond the two documented behaviours cited.

**Assumptions**

- The global panic hook installed at `nodes/node/binary/src/panic.rs:35` (PR #126 LB-001, re-verified unchanged by PR #206) is in place, so a panic on a request task is a process exit, not a dropped connection.
- PR #118's finding that the HTTP API has no authentication still holds at this commit (nothing in `backend.rs:229-284` adds any).
- The Linux kernel delivers `ITIMER_PROF` at tick resolution; the report does not measure the CPU cost of high sampling frequencies.

## 3. Method

- Manual review of the in-scope paths, working through the four task-list items of issue `#208`, with the repo facts from `#19` and the exit-hook context from PR #126 / PR #206.
- Build-surface enumeration: every `cargo build`/`cargo install` invocation in the node repository's Dockerfiles, workflows, `flake.nix`, README files and testing framework; the same in the two deployment repositories; `gh search code` across the `logos-blockchain` organisation for `features profiling`, `release-profiling` and `debug/pprof`.
- Handler-level verification instead of a full node build: a scratchpad crate depending on `nodes/api-common` by path with `features = ["profiling"]` (rustc/cargo 1.98.1, `--release`), driving the unmodified `cpu_profile` and the patched copy through `axum::Router::oneshot`. The full `logos-blockchain-node --features profiling` binary was not built: it pulls RocksDB and the ZK stack for no additional evidence, since `pprof.rs:86-93` is the only code between the query string and `pprof`, and `backend.rs:261-272` only mounts the router.
- Dynamic testing: the eight tests in Appendix C (two against the original handler, six against the patch). No devnet.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | `/debug/pprof/profile?frequency=0` exits the node in `profiling` builds (PR #206 LB-001, now reproduced) | Data Validation | Medium | Low | Open |
| LB-002 | The profiling router is merged after the API middleware stack, so `seconds` is unbounded and no timeout, concurrency limit or metrics apply to it | Denial of Service | Low | Low | Open |
| LB-003 | `logos-blockchain-testing` builds `profiling` bundles on request and binds the API to `0.0.0.0`, so every framework-deployed profiling node exposes the route on all interfaces | Configuration | Low | Low | Open |
| LB-004 | The `tokio-console` endpoint accepts any bind address and any recording path from config; only the README asks for loopback | Configuration | Informational | High | Open |

### 4.0 Inventory: what each build feature compiles in, and what ships

**Item 1: build surface.** Every build command found, with the features it enables:

| Where | Command | Features / profile | pprof route? |
|---|---|---|---|
| `deployment/Dockerfile:18` (multi-node compose image, `deployment/compose.run.yml`) | `cargo build --locked --release` | none | no |
| `Dockerfile` (root; `publish-node-image.yml:63-78` builds and pushes it) | no `cargo` at all: downloads the prebuilt `blockchain_module` `.lgx` for `LB_NODE_VERSION` via `lgpd` | n/a | no |
| `.github/workflows/prepare-release.yml:88` (release binaries) | `cargo build -p logos-blockchain-node --release --locked --target …` | none | no |
| `flake.nix:79` (consumed by `logos-blockchain-module/flake.nix:46,120` as `logos-blockchain-c`) | `cargoExtraArgs = "-p logos-blockchain-c"` | none (`c-bindings/Cargo.toml` has no `[features]`) | no |
| `tests/testing_framework/assets/runtime/Dockerfile.node:13,27-29` | `cargo build --locked --release [--features testing]` | `testing` | no |
| `.github/workflows/{code-check,end-to-end-integration-tests,nightly-cluster-fork-detector}.yml`, `.github/actions/run-cucumber-suite/action.yml` | `--features testing` / `testing-disable-proposal-publish` / `mantle_sdp` / `tip_poll_self_heal` / `testing_framework` / `cucumber` | test features only | no |
| `tests/testing_framework/src/framework/local/provisioning.rs:450-470` (`NodeBinaryProfile::TokioConsole`, local runs only; CI uses the plain release binary, `:451-452`) | `cargo build --locked --profile release-profiling -p logos-blockchain-node --features testing,tokio-console` | `release-profiling` + `tokio-console` | **no**: `tokio-console` (`Cargo.toml:106-120`) does not include `lb-http-api-common/profiling`; only the `profiling` umbrella (`Cargo.toml:95`) does |
| `README.md:195` (heap profiling), `README.md:215-219` (tokio console) | `--profile release-profiling --features=dhat-heap` / `--features tokio-console` | documented manual builds | no |
| `deployment/systemd/logos-blockchain-node.service:13` | runs a prebuilt `/home/<user>/logos-blockchain-node` | operator-supplied binary | depends on how it was built |
| `logos-blockchain-testing/scripts/build/build-bundle.sh:352-361,398-403` (via `build-linux-binaries.sh:106-114` `--features`) | `cargo build --features "${FEATURES}"` where `FEATURES` = `testing` plus whatever `--features` the caller passes | **`profiling` on request**; when present the script prints `curl "http://<node-host>:8722/debug/pprof/profile…"` | **yes, opt-in** |
| `logos-blockchain-testing/testing-framework/assets/stack/scripts/docker/prepare_binaries.sh:57` | `cargo build --features "testing"` | `testing` | no |
| `logos-blockchain-testing/.github/workflows/{build-binaries,deploy-pages,lint}.yml` | `lint.yml:99` runs clippy with `--all-features` (compile only, nothing deployed) | n/a | no |

So `release-profiling` (`Cargo.toml` `[profile.release-profiling]`: `debug = true`, `strip = "none"`, inherits release) answers #19's open question: it is a *profile*, it changes symbols only, and the only in-repo user is the local `tokio-console` provider. The route is compiled in exactly when the `profiling` Cargo *feature* is on, and the only place that turns it on is a manual `--features profiling` to the testing repository's bundle builder (LB-003). `.cargo-deny.toml:20-21` records the same conclusion ("`pprof` is absent from default node builds and no CI or release build enables it") as the justification for ignoring two `quick-xml` advisories.

**Item 2: gated surfaces and their parameters.**

| Surface | Gate | Transport | Request / config parameters | Reaches | Validation at the boundary |
|---|---|---|---|---|---|
| `GET /debug/pprof/profile` (`nodes/api-common/src/pprof.rs:46,86`) | `feature = "profiling"` (`api-common/Cargo.toml:36`, mounted at `backend.rs:261-272`); `compile_error!` on Windows (`lib.rs:9-12`) | HTTP, **same listener and port as the whole API** (`backend.rs:274`), no authentication (PR #118) | `seconds: Option<u64>`, `frequency: Option<i32>`, `format: Option<ProfileFormat>` (`pprof.rs:27-31`) | `frequency` → `pprof::ProfilerGuard::new` → `Timer::new` → `1e6 / frequency` → `setitimer(ITIMER_PROF)` (`pprof-0.15.0/src/timer.rs:35-50`) and the `SIGPROF` handler rate (`profiler.rs:285-380`); `seconds` → `tokio::time::sleep` (`pprof.rs:93`) while the process-wide profiler is held | **none** for either number; `format` is a two-variant enum, safe |
| `tokio-console` gRPC endpoint (`services/tracing/src/console.rs:8-32`) | `feature = "tokio-console"` + `RUSTFLAGS=--cfg tokio_unstable` + config `tracing.console: !Console` (`Cargo.toml:99-105`, `lib.rs:288-291`) | gRPC on `bind_address:port`, default `127.0.0.1:6669` (`config/tests.rs:67-72`) | config only: `bind_address`, `port`, `recording_path` | `TcpListener` bind inside `ConsoleLayer::builder().server_addr(..).spawn()`; `fs::create_dir_all` + `OpenOptions::create().append()` on `recording_path` (`console.rs:35-60`) | `bind_addr.parse::<SocketAddr>()` only; an unparsable address silently disables the layer (`console.rs:19-20`); no loopback check, any path (LB-004) |
| `dhat-heap` (`nodes/node/binary/src/global_allocators/dhat_heap.rs`, `main.rs:13-14`, `panic.rs:32-33`) | `feature = "dhat-heap"` | none; writes `dhat-heap.json` to the working directory on drop | none | allocator hooks, one file write at exit | n/a |
| `jemalloc` (`global_allocators/jemalloc.rs`) | `feature = "jemalloc"` | none | none | allocator | n/a |
| `PUT /admin/tracing/filter` (`api-common/src/paths.rs:52-53`) | **not gated**: always mounted | HTTP, same listener | filter strings | `tracing-subscriber` reload handles | covered by PR #217 LB-003; listed here because the issue asked for "debug flag" routes and this is the one debug-shaped route that ships in every build |
| `GET /mantle/metrics` (`paths.rs:1`) | not gated | HTTP | none | read-only counters | n/a |

There is no `/metrics` scrape route, no `/debug/*` besides pprof, and no `cfg(debug_assertions)`-gated handler anywhere under `nodes/node/binary/src/api/` or `nodes/api-common/src/` (`grep -rn "cfg(" nodes/node/binary/src/api/*.rs` hits only `#[cfg(test)]` and the one `profiling` gate).

### LB-001 · `/debug/pprof/profile?frequency=0` exits the node in `profiling` builds (PR #206 LB-001, now reproduced)

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Data Validation |
| Target | `nodes/api-common/src/pprof.rs:86-93` (`cpu_profile`); `pprof-0.15.0/src/timer.rs:35` (`Timer::new`), reached from `profiler.rs:142` after `profiler.start()` at `:139` |
| Status | Open (carried from PR #206 LB-001; reproduced in Appendix C) |

**Description**

Unchanged from PR #206: the handler takes `frequency` from the query string, defaults it to 1000 (`pprof.rs:88`), and passes it to `pprof::ProfilerGuard::new` (`:91`). `ProfilerGuardBuilder::build` first registers the `SIGPROF` handler and sets `running = true` (`profiler.rs:139`, `:419-427`), then constructs the timer:

```rust
// pprof-0.15.0/src/timer.rs
pub fn new(frequency: c_int) -> Timer {
    let interval = 1e6 as i64 / i64::from(frequency);   // line 35: panics for 0
```

The scratchpad test in Appendix C.1 drives the unmodified `cpu_profile` with `?frequency=0&seconds=1` and asserts that the task panics with a payload containing `divide by zero`; it does. In the node the panic reaches the global hook and the process exits (PR #126 LB-001). Two side effects noted for completeness: the panic happens *after* `profiler.start()`, so even without the exit hook the global profiler would stay `running` with the signal handler registered and every later profile would fail with `Error::Running`; and the `spin::RwLock` write guard (`profiler.rs:10,123`) is released on unwind, so the lock itself is not poisoned.

The second test (Appendix C.2) shows the other unchecked direction: `?frequency=-1&seconds=1` is accepted. `Timer::new` computes a negative interval, `setitimer` rejects it with `EINVAL`, `pprof` discards the return value (`timer.rs:41-48`), and the request holds the profiler for `seconds` and returns 200 with no samples.

**Exploit scenario**

As in PR #206: against a node built with `profiling` whose API port is reachable, one unauthenticated `GET /debug/pprof/profile?frequency=0` stops the node. §4.0 establishes that no shipped artefact is such a node; LB-003 establishes which nodes are.

**Recommendation**

- *Short term*: the validation block in Appendix B (`frequency` in `1..=10_000`, `seconds` in `1..=300`, 400 otherwise). It is 30 lines in `pprof.rs` and needs no change elsewhere.
- *Long term*: see LB-002.

**References**: PR #206 LB-001; PR #126 LB-001 (exit hook); issue #19 (`release-profiling`); `.cargo-deny.toml:20-21`.

### LB-002 · The profiling router is merged after the API middleware stack, so `seconds` is unbounded and no timeout, concurrency limit or metrics apply to it

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `nodes/node/binary/src/api/backend.rs:231-272` (layer order vs. `merge`); `nodes/api-common/src/pprof.rs:87,93` (`seconds`); `tokio-1.52.3/src/time/sleep.rs:126-129` |
| Status | Open |

**Description**

The API's middleware is applied to the main router before the profiling router exists:

```rust
// nodes/node/binary/src/api/backend.rs
let app = app
    .with_state(handle.clone())
    .layer(axum::Extension(self.settings.chain_id.clone()))            // 236
    .layer(axum::middleware::from_fn(http_metrics_middleware))         // 237
    .layer(axum::extract::DefaultBodyLimit::max(..))                   // 238
    .layer(TimeoutLayer::with_status_code(REQUEST_TIMEOUT, timeout))   // 241
    .layer(RequestBodyLimitLayer::new(..))                             // 245
    .layer(ConcurrencyLimitLayer::new(max_concurrent_requests))        // 246
    .layer(TraceLayer::..);                                            // 249
let app = app.layer(cors_layer.clone());                               // 259
#[cfg(feature = "profiling")]
let app = {
    let pprof_routes = lb_http_api_common::pprof::create_pprof_router()
        .layer(TraceLayer::..)                                         // 264
        .layer(cors_layer);                                            // 269
    app.merge(pprof_routes)                                            // 271
};
```

`axum` 0.7.9 documents that `Router::layer` "is only applied to existing routes … routes added after `layer` is called will not have the middleware added" (`src/docs/routing/layer.md:6-8`). The profiling router therefore gets only its own trace and CORS layers. Consequences at this commit:

- `seconds` is not bounded by anything. `tokio::time::sleep(Duration::from_secs(u64::MAX))` does not panic: `sleep` falls back to `Instant::far_future()` when the deadline overflows (`sleep.rs:126-129`). One request with a huge `seconds` therefore holds the global profiler, with `SIGPROF` firing at `frequency` Hz on every thread, until the connection drops; the 30-second `TimeoutLayer` the operator configured for the API (`settings.rs:32-34`) never sees it.
- `pprof` allows one profiler per process (`Profiler::start` returns `Error::Running`, `profiler.rs:421-422`), so while that request is held every legitimate profile attempt gets a 500 `Failed to start profiler`. The operator's own profiling is denied for as long as the attacker keeps the connection open.
- The `ConcurrencyLimitLayer` and `http_metrics_middleware` do not count these requests, so the hold is invisible in the API's request metrics.

The CPU cost of the hold is real but modest: each `SIGPROF` walks up to `MAX_DEPTH` frames (128 on this platform set, `pprof/src/lib.rs:44-50`) and pushes a sample; `ITIMER_PROF` is delivered at kernel tick resolution, so `frequency` above the tick rate does not multiply the cost. This is why the finding is Low and not a remote CPU exhaustion.

**Exploit scenario**

Same precondition as LB-001. `curl 'http://<node>:<api-port>/debug/pprof/profile?seconds=4000000000'` and keep the socket open: the node keeps sampling, no other profile can start, and nothing in the API's limits or metrics reflects it. Combined with LB-001 the attacker chooses between exiting the node and pinning its profiler.

**Recommendation**

- *Short term*: the bounds and in-flight guard in Appendix B (`seconds` capped at 300, a second concurrent request answered with 409 rather than left to `pprof`'s 500).
- *Long term*: do not let a build-time feature widen the unauthenticated listener. Either (a) serve the profiling router on its own listener bound to loopback, configured separately from `AxumBackendSettings.address`, or (b) keep it on the main listener but behind the authentication layer from #119 and mounted *before* the middleware stack so timeout and concurrency limits apply. Sketch for (a), uncompiled:

```rust
// nodes/api-common/src/settings.rs
pub struct AxumBackendSettings {
    ...
    /// Loopback-only listener for debug/profiling routes; None disables them
    /// even in `profiling` builds.
    #[serde(default)]
    pub debug_address: Option<SocketAddr>,   // default 127.0.0.1:<port+1> in Config::default
}

// nodes/node/binary/src/api/backend.rs, inside serve()
#[cfg(feature = "profiling")]
if let Some(debug_addr) = self.settings.debug_address {
    if !debug_addr.ip().is_loopback() {
        tracing::warn!(target: LOG_TARGET, %debug_addr, "debug listener is not loopback");
    }
    let debug_app = lb_http_api_common::pprof::create_pprof_router()
        .layer(TimeoutLayer::with_status_code(StatusCode::REQUEST_TIMEOUT,
               Duration::from_secs(lb_http_api_common::pprof::MAX_SECONDS + 5)))
        .layer(ConcurrencyLimitLayer::new(1));
    let debug_listener = TcpListener::bind(debug_addr).await?;
    tokio::spawn(axum::serve(debug_listener, debug_app.into_make_service()));
}
```

  Operators who profile remote nodes then forward the port over SSH, which is what `README.md:293` already tells them to do for `tokio-console`.

**References**: PR #118 (unauthenticated API); PR #130 / #119 (auth layer); PR #217 LB-003 (the other debug-shaped route on the same listener); `axum` 0.7.9 `docs/routing/layer.md`.

### LB-003 · `logos-blockchain-testing` builds `profiling` bundles on request and binds the API to `0.0.0.0`, so every framework-deployed profiling node exposes the route on all interfaces

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Configuration |
| Target | `logos-blockchain-testing` @ `1b336c2c`: `scripts/build/build-bundle.sh:330-331,352-361,398-403`, `scripts/build/build-linux-binaries.sh:106-114`, `testing-framework/tools/cfgsync/src/config/builder.rs:263`, `testing-framework/tools/cfgsync/src/server.rs:278`, `testing-framework/deployers/compose/src/deployer/orchestrator.rs:200-236`, `testing-framework/deployers/k8s/src/deployer/orchestrator.rs:341-355`; node repo `tests/testing_framework/src/node/cfgsync.rs:109-116` |
| Status | Open |

**Description**

The node's default API bind address is loopback (`config/api/serde.rs:39-41`: `127.0.0.1:8080`; `standalone-node-config.yaml:170`), which on its own would keep a profiling build's route off the network. Both testing frameworks rewrite it:

```rust
// logos-blockchain (node repo) tests/testing_framework/src/node/cfgsync.rs:109-116
const fn apply_launch_ready_bind_addresses(config: &mut RunConfig) {
    config.user.api.backend.listen_address.set_ip(IpAddr::V4(Ipv4Addr::UNSPECIFIED));
}
```

```rust
// logos-blockchain-testing testing-framework/tools/cfgsync/src/config/builder.rs:263
let address_value = format!("0.0.0.0:{}", host.api_port);
// …/server.rs:278
*address = json!(format!("0.0.0.0:{api_port}"));
```

The testing repository's bundle builder appends any caller-supplied features to the base `testing` set (`build-bundle.sh:330-331`) and builds with them (`:352-361`); when the set contains `profiling` it prints ready-made `curl` lines against `http://<node-host>:8722/debug/pprof/profile` (`:398-403`). Both deployers print or log the same URL for every validator and executor (`compose …/orchestrator.rs:200-236`, `k8s …/orchestrator.rs:341-355`, the latter behind `TESTNET_PRINT_ENDPOINTS`). On compose the API ports are published to the Docker host (`deployment/compose.run.yml:50-53,116-119` in the node repo; the testing repo's compose deployer maps `node.api` to a host port, `orchestrator.rs:205-206`).

None of this runs in CI: the testing repository's workflows build with `testing` only, and `prepare_binaries.sh:57` hard-codes `--features "testing"`. The exposure is therefore exactly the set of nodes an engineer builds with `--features profiling` and deploys through the framework, and on those nodes LB-001 is a one-request crash and LB-002 a one-request hold from anything that can reach the published port. The bundle builder still clones `logos-co/nomos-node` and builds `nomos-node` packages (`build-bundle.sh:340-345,356-361`), so it has not been used against a `logos-blockchain`-named checkout recently; the `--path` override (`build-linux-binaries.sh:108-109`) makes that irrelevant to the exposure once it is.

**Exploit scenario**

An engineer runs `scripts/build/build-linux-binaries.sh --features profiling` and deploys the bundle to a shared devnet with the k8s deployer to take flamegraphs. Every node in that devnet now answers `/debug/pprof/profile` on its Service address with no authentication. Anyone on the cluster network sends `?frequency=0` to each and the devnet is down; with `?seconds=4000000000` they keep the profiler busy so the engineer cannot take the profile they built the bundle for.

**Recommendation**

- *Short term*: in `build-bundle.sh`, when `FEATURES` contains `profiling`, print a warning next to the `curl` lines that the route is unauthenticated and is bound to `0.0.0.0` by cfgsync; land the Appendix B bounds in the node so the crash at least is gone.
- *Long term*: with LB-002's loopback listener the framework's `0.0.0.0` rewrite stops mattering for the debug routes (the rewrite touches `listen_address`, not `debug_address`). Until then, have cfgsync leave the API on loopback for profiling bundles and reach it through `kubectl port-forward` / a compose sidecar, which is what the framework's own `README.md:293` recommends for `tokio-console`.

**References**: #19 (which artefacts ship); PR #118; the testing repository's `nomos-bundle-meta.env` (`build-bundle.sh:378-386`) records `features=` per bundle, so an existing deployment can be checked for `profiling` after the fact.

### LB-004 · The `tokio-console` endpoint accepts any bind address and any recording path from config; only the README asks for loopback

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Configuration |
| Target | `services/tracing/src/console.rs:8-32` (`create_console_layer`), `:35-60` (`prepare_recording_path`); `nodes/node/binary/src/config/tests.rs:67-72` (defaults); `README.md:222-235,293` |
| Status | Open |

**Description**

The second feature-gated surface is the `tokio-console` gRPC server. It takes no request parameters, so it is not a parameter-validation issue like LB-001; it is listed because item 2 asked for every gated route. Its config is `bind_address`, `port` and `recording_path` (`TokioConsoleConfig`); the layer builder formats them into a `SocketAddr` and binds there (`console.rs:17-30`). There is no check that the address is loopback, and the README carries the only guard: "Keep the console bound to loopback. When profiling a remote node, forward the port over SSH" (`README.md:293`). `console-subscriber` 0.5's server is unauthenticated and streams task names, span fields and waker events, which PR #217 LB-002 rates as identity-linking material when they carry Blend fields. `recording_path` is created with `create_dir_all` and opened append (`:37-60`), so a config pointing it at an existing file appends telemetry to that file; that is the operator's own config, hence Informational.

An unparsable `bind_address:port` disables the layer silently (`console.rs:19-20` maps the parse error to `Ok(None)`), which is a fail-quiet rather than fail-open default and would fit #204's list.

**Recommendation**

- *Short term*: reject a non-loopback `bind_address` at config load unless an explicit `allow_remote: true` is set, and turn the parse failure into a config error.
- *Long term*: same as LB-002: one loopback-only debug listener with one policy, rather than per-feature conventions.

**References**: PR #217 LB-002 (identity fields in tracing output); #204 (silent config defaults).

## 5. Suggestions (non-security)

### S-001 · Document the `profiling` route where the other profiling features are documented

`README.md:190-300` documents heap profiling (`dhat-heap`) and Tokio task profiling (`tokio-console`), each with its build command and a loopback warning. The CPU profiling route has no README entry; its only documentation is the doc comment in `pprof.rs:49-85`, and the `curl` examples there and in the testing repository use port `8722`, which is neither the node default (`8080`, `serde.rs:34-36`) nor the compose ports (`18080-18083`, `deployment/.env.devnet:12-27`). Add a "CPU profiling" section with the build command, the parameter bounds from Appendix B, and the same loopback/SSH-forward instruction as the console section.

### S-002 · Add `profiling` to the feature matrix CI compiles

`nightly-cargo-hack.yml` exists but nothing in the node repository compiles `-p logos-blockchain-node --features profiling` (the testing repository's `lint.yml:99` does, with `--all-features`, as a by-product). A `cargo check --features profiling` job would have caught the Windows `compile_error!` earlier and would compile the Appendix B tests if they land in `api-common`.

### S-003 · The `.cargo-deny.toml` justification should point at the build inventory

`.cargo-deny.toml:20-21` ignores two `quick-xml` advisories on the grounds that no CI or release build enables `profiling`. §4.0 confirms that today; the note would age better if it named the build files it depends on (`deployment/Dockerfile`, `prepare-release.yml`, `flake.nix`) so a future `--features profiling` in any of them is recognised as invalidating the ignore.

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

## Appendix B — Patch for `nodes/api-common/src/pprof.rs` (LB-001, LB-002 short term)

Applies to `a805329f8a186eb6989f09a7c49dee4a0e07473b`. The constants are `pub` so the node binary can reuse `MAX_SECONDS` for a debug-listener timeout (LB-002 long term). The in-flight guard is released on drop, so a client that disconnects mid-profile frees the slot when axum drops the handler future.

```diff
--- a/nodes/api-common/src/pprof.rs
+++ b/nodes/api-common/src/pprof.rs
@@ -1,4 +1,7 @@
-use std::time::Duration;
+use std::{
+    sync::atomic::{AtomicBool, Ordering},
+    time::Duration,
+};
 
 use axum::{
     Router,
@@ -13,6 +16,39 @@
 use serde::Deserialize;
 
 const LOG_TARGET: &str = api::http::PPROF;
+
+/// Lowest sampling frequency accepted by the endpoint, in Hz.
+///
+/// `pprof::Timer::new` computes `1_000_000 / frequency` and hands the result to
+/// `setitimer(2)`; zero divides by zero and a negative value produces an
+/// interval `setitimer` rejects with `EINVAL`, which `pprof` ignores.
+pub const MIN_FREQUENCY_HZ: i32 = 1;
+/// Highest sampling frequency accepted by the endpoint, in Hz.
+///
+/// Every `SIGPROF` delivery walks the interrupted thread's stack up to
+/// `pprof::MAX_DEPTH` frames, so the frequency bounds the CPU the profiler
+/// itself burns. 10 kHz is an order of magnitude above the 1 kHz default.
+pub const MAX_FREQUENCY_HZ: i32 = 10_000;
+/// Shortest profile accepted, in seconds.
+pub const MIN_SECONDS: u64 = 1;
+/// Longest profile accepted, in seconds.
+///
+/// The profiling router is merged after the API's `TimeoutLayer`, so nothing
+/// else bounds how long a single request may hold the process-wide profiler.
+pub const MAX_SECONDS: u64 = 300;
+
+/// Only one CPU profile can run per process (`pprof` keeps one global
+/// profiler and returns `Error::Running` for a second one). Refuse a second
+/// request up front with a status a client can act on instead of a 500.
+static PROFILE_IN_FLIGHT: AtomicBool = AtomicBool::new(false);
+
+struct InFlightGuard;
+
+impl Drop for InFlightGuard {
+    fn drop(&mut self) {
+        PROFILE_IN_FLIGHT.store(false, Ordering::Release);
+    }
+}
 
 #[derive(Deserialize, Debug, Clone)]
 #[serde(rename_all = "lowercase")]
@@ -66,10 +102,14 @@
 /// - `proto`: Google pprof protobuf format for analysis tools
 ///
 /// # Query Parameters
-/// - `seconds` (default: 30): Profiling duration in seconds
-/// - `frequency` (default: 1000): Sampling frequency in Hz (samples per second)
+/// - `seconds` (default: 30): Profiling duration in seconds, `1..=300`
+/// - `frequency` (default: 1000): Sampling frequency in Hz, `1..=10000`
 /// - `format` (default: proto): Output format (svg|proto)
 ///
+/// Out-of-range values are rejected with `400 Bad Request` before anything
+/// is handed to `pprof`; a request that arrives while a profile is already
+/// running is rejected with `409 Conflict`.
+///
 /// # Usage Examples
 /// ```bash
 /// # Generate flamegraph (viewable in browser)
@@ -88,6 +128,30 @@
     let frequency = params.frequency.unwrap_or(1000);
     let format = params.format.unwrap_or_default();
 
+    if !(MIN_FREQUENCY_HZ..=MAX_FREQUENCY_HZ).contains(&frequency) {
+        return (
+            StatusCode::BAD_REQUEST,
+            format!(
+                "frequency must be within {MIN_FREQUENCY_HZ}..={MAX_FREQUENCY_HZ} Hz, got {frequency}"
+            ),
+        )
+            .into_response();
+    }
+    if !(MIN_SECONDS..=MAX_SECONDS).contains(&duration_secs) {
+        return (
+            StatusCode::BAD_REQUEST,
+            format!("seconds must be within {MIN_SECONDS}..={MAX_SECONDS}, got {duration_secs}"),
+        )
+            .into_response();
+    }
+    if PROFILE_IN_FLIGHT
+        .compare_exchange(false, true, Ordering::AcqRel, Ordering::Acquire)
+        .is_err()
+    {
+        return (StatusCode::CONFLICT, "a CPU profile is already running").into_response();
+    }
+    let _in_flight = InFlightGuard;
+
     match pprof::ProfilerGuard::new(frequency) {
         Ok(guard) => {
             tokio::time::sleep(Duration::from_secs(duration_secs)).await;
```

## Appendix C — Verification (item 4)

Harness: a scratchpad crate with `lb-http-api-common = { path = "…/nodes/api-common", features = ["profiling"] }` (the tree at `a805329f8`, unmodified), `axum` 0.7.9, `tokio` 1.52, `tower` 0.4 (`ServiceExt::oneshot`), built with rustc 1.98.1 in `--release`. The patched handler is the Appendix B file compiled as a module of the harness. Tests C.1 and C.2 each live in their own test binary because the panic in C.1 leaves `pprof`'s global profiler in the `running` state for the rest of that process.

### C.1 Unmodified handler, `frequency=0` (reproduces LB-001)

```rust
#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
async fn frequency_zero_panics_inside_pprof() {
    let app = Router::new().route("/debug/pprof/profile",
        get(lb_http_api_common::pprof::cpu_profile));
    let handle = tokio::spawn(async move {
        app.oneshot(Request::get("/debug/pprof/profile?frequency=0&seconds=1")
            .body(Body::empty()).unwrap()).await
    });
    let err = handle.await.expect_err("the original handler must panic on frequency=0");
    assert!(err.is_panic());
    let payload = err.into_panic();
    let msg = payload.downcast_ref::<&str>().map(|s| (*s).to_owned())
        .or_else(|| payload.downcast_ref::<String>().cloned()).unwrap_or_default();
    assert!(msg.contains("divide by zero"), "unexpected panic message: {msg}");
}
```

### C.2 Unmodified handler, `frequency=-1`

`GET /debug/pprof/profile?frequency=-1&seconds=1&format=svg` → `200 OK` after ≥ 1 s (the `EINVAL` from `setitimer` is discarded by `pprof`; no samples).

### C.3 Patched handler

| Request | Expected | Result |
|---|---|---|
| `?frequency=0` | 400, body names `frequency` | ok |
| `?frequency=-1` | 400 | ok |
| `?frequency=10001` | 400 | ok |
| `?seconds=0`, `?seconds=301`, `?seconds=18446744073709551615` | 400 each | ok |
| `?frequency=abc` | 400 (axum `Query` rejection, unchanged) | ok |
| `?frequency=99&seconds=2&format=svg`, then `?frequency=99&seconds=1` 300 ms later, then a third after the first completes | 200, 409, 200 | ok |

### C.4 Output

```
     Running tests/original_negative.rs
test negative_frequency_is_accepted_by_original_handler ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.04s

     Running tests/original_panics.rs
test frequency_zero_panics_inside_pprof ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.02s

     Running tests/patched.rs
test rejects_negative_frequency ... ok
test non_numeric_parameter_is_still_a_400_from_the_extractor ... ok
test rejects_frequency_above_cap ... ok
test rejects_seconds_zero_and_above_cap ... ok
test rejects_frequency_zero ... ok
test valid_request_is_served_and_concurrent_one_is_refused ... ok
test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 3.06s
```

What this does and does not show: the panic and the 400s are established at the handler, which is the whole code path between the query string and `pprof` (`backend.rs:261-272` only mounts the router). It does not exercise the node's panic hook (that the panic becomes `exit(1)` is PR #126 LB-001, re-verified by PR #206), and it does not build the node binary with `--features profiling`; follow-up #261 covers landing the tests upstream where CI can run them under that feature.

## Appendix D — Ruled out

- **Any shipped artefact with the route.** `deployment/Dockerfile:18`, the root `Dockerfile` (downloads a prebuilt module), `prepare-release.yml:88`, `flake.nix:79` (`-p logos-blockchain-c`, and the module repository consumes that package: `logos-blockchain-module/flake.nix:46,120`), the two testing-framework Dockerfiles, every workflow in both repositories: none passes `profiling`. `c-bindings/Cargo.toml` has no `[features]` table, so the FFI library cannot enable it either, and the FFI path pins the API to `127.0.0.1:0` (`c-bindings/src/api/lifecycle.rs:272`).
- **`release-profiling` as a shipping profile.** Used only by `provisioning.rs:455-466` for local `tokio-console` runs; CI takes the plain release binary (`:451-452`). The profile does not enable any feature.
- **`tokio-console` pulling in pprof.** Its feature list (`nodes/node/binary/Cargo.toml:106-120`) contains `lb-tracing-service/tokio-console` and `tokio-task-names` features only.
- **`format` parameter.** Two-variant enum with `rename_all = "lowercase"`; an unknown value is a 400 from the `Query` extractor.
- **Large `frequency`.** Values above 1,000,000 make `interval` zero, and `setitimer` with a zero `it_value` disarms the timer (`timer.rs:35-48`): no samples, no cost, a 200 with an empty report. Not a crash.
- **`seconds` overflow.** `Duration::from_secs(u64::MAX)` is valid and `tokio::time::sleep` clamps the deadline (`sleep.rs:126-129`); no panic (this is LB-002's hold, not a crash).
- **Second concurrent profile crashing the node.** `Profiler::start` returns `Error::Running` (`profiler.rs:419-427`); the handler maps it to a 500 (`pprof.rs:111-116`). The patch turns this into a 409 before `pprof` is touched.
- **Other `cfg`-gated handlers.** None under `nodes/node/binary/src/api/` or `nodes/api-common/src/` besides the `profiling` gate; `dhat-heap` and `jemalloc` have no request surface.
- **`SIGPROF` handler re-entrancy.** `perf_signal_handler` uses `PROFILER.try_write()` and returns if the lock is held (`profiler.rs:292`), so a signal landing during report generation is dropped, not deadlocked.
