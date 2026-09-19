# Audit Report — cfgsync `debug_address` in `logos-blockchain-testing`: the repository no longer configures nodes, and the per-node loopback debug port already lives in the unmerged #260 patch

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/281` (parent `#18`)
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `3d5d419ec85c7b23e4d7e1455bd1cde845264a9e` — component(s): `tests/testing_framework/src/node/cfgsync.rs`, `tests/testing_framework/src/node/configs/dynamic.rs`, `tools/config/src/{api.rs, tracing.rs}`, `nodes/node/binary/src/api/{backend.rs, routes.rs}`, `nodes/node/binary/src/config/api/serde.rs`, `nodes/node/binary/src/config/tracing/serde/console.rs`, `nodes/api-common/src/{paths.rs, pprof.rs}`, `Cargo.toml` (testing-framework pins), `deployment/compose.run.yml`
Second target: `https://github.com/logos-blockchain/logos-blockchain-testing` @ `1fdd8b96b03c7d034a5641ad42a9c9aea514d3da` (HEAD, 2026-09-14), compared with `1b336c2c080715b0873bdb5b36bd526dd0e74baf` (the commit the issue cites, 2026-01-12) and `db880d61155db3a4f32177ef16b0f1e4bf22b1f6` (the revision the node pins, 2026-08-04) — component(s): `cfgsync/{core,runtime,adapter}/src`, `testing-framework/deployers/{compose,k8s,local}/src`, `scripts/`, `book/src`, `README.md`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md` (core), `wallet-technical-standard.md`, `mantle-transaction-encoding.md` (area, per #18); `bedrock-v1.1-mantle-specification.md` not consulted: no section of it governs a test framework's bind addresses, and nothing below rests on it
Date: 2026-09-14 — author: `agent (Claude)` — status: `final`

---

## 1. Summary

- Overall assessment: none of the three items of #281 can be done in the repository the issue names, and two of them are already done elsewhere. `logos-blockchain-testing` stopped assembling node configuration on 2026-01-19 (`77ae507` deleted `testing-framework/tools/cfgsync/src/config/builder.rs`, seven days after the commit the issue cites) and deleted `scripts/build/build-bundle.sh` on 2026-03-29 (`79037d7`); at HEAD it is an application-agnostic framework with no knowledge of the node's `api` section, which the node consumes as a git dependency. The node's own config assembly, including the `0.0.0.0` rewrite, is `tests/testing_framework/src/node/cfgsync.rs:108-115` in the node repository. The per-node loopback debug port (item 1) and the `launch_ready_rewrite_opens_the_api_listener_but_not_the_debug_listener` test (item 3) are both in the #260 patch against that file and `tools/config/src/api.rs` (`processed/260-loopback-debug-listener.md`, Appendix B) — and that patch is not upstream: the string `debug_address` occurs nowhere in `logos-blockchain` at `3d5d419e`. Item 2's script is gone; what remains of it are the pprof URL printers already tracked as #290, with one interaction with #260 recorded in S-002.
- Findings: 0 critical · 0 high · 0 medium · 0 low · 0 informational.
- Key themes: "the issue's target moved before the issue was filed", "the fix exists but has not been upstreamed", "a generic framework still carries one node-specific URL".
- Must-fix before launch: none from this issue.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `logos-blockchain-testing` @ `1fdd8b96` — whole tree | Searched for every place that sets, rewrites or prints a node API address or a debug/pprof URL; read `cfgsync/core/src/{render.rs, server.rs}`, `cfgsync/runtime/src/{server.rs, client.rs}`, `testing-framework/deployers/compose/src/provisioner/mod.rs:595-634`, `testing-framework/deployers/compose/src/deployer/orchestrator.rs:270-282`, `testing-framework/deployers/k8s/src/env.rs:181-235`, `testing-framework/deployers/k8s/src/deployer/orchestrator.rs:690-792`, `README.md`, `book/src/{deployer-compose.md, deployer-k8s.md, environment-variables.md}` |
| `logos-blockchain-testing` @ `1b336c2c` and `db880d61` | The lines the issue cites (`builder.rs:263`, `server.rs:278`, `build-bundle.sh:398-403`) confirmed at `1b336c2c`; the pprof printers confirmed present at `db880d61` |
| `logos-blockchain` @ `3d5d419e` — `tests/testing_framework/src/node/{cfgsync.rs, configs/dynamic.rs}`, `tools/config/src/{api.rs, tracing.rs}` | Where node configuration is assembled for the local, compose and k8s deployers; which fields the `0.0.0.0` rewrite touches; how per-node ports are reserved |
| `logos-blockchain` @ `3d5d419e` — `nodes/node/binary/src/api/{backend.rs, routes.rs}`, `nodes/node/binary/src/config/api/serde.rs`, `nodes/api-common/src/{paths.rs, pprof.rs}` | Which listener carries `/debug/pprof/profile` and `PUT /admin/tracing/filter` today |
| `logos-blockchain` @ `3d5d419e` — `nodes/node/binary/src/config/tracing/serde/console.rs`, `tools/config/src/tracing.rs`, `deployment/compose.run.yml`, `Cargo.toml:104-105,178-181` | The other fixed-port listener (Tokio console), the node's own compose networking, the testing-repo pin |
| `processed/260-loopback-debug-listener.md` Appendix B | Read for the `tools/config/src/api.rs` and `tests/testing_framework/src/node/cfgsync.rs` hunks and the test named by item 3 |

**Out of scope**

- Building or running either framework; no deployment was started. The dynamic evidence for the #260 patch is in that report's §4.5 and is not re-run here.
- The cfgsync server's own bind and authentication (`cfgsync/core/src/server.rs:126`, `cfgsync/runtime/src/server.rs:222`): #289.
- Removing the pprof URL printers and the stale `nomos-node` boundary path: #290. This report records only how #260 changes what #290 should do (S-002).
- The pprof handler's parameter bounds: #261. The authentication layer for the API address: #119.
- Third-party crates assumed correct: `axum`, `tokio`, `serde_yaml`, `kube`, `bollard`.

**Assumptions**

- The #260 patch (`processed/260-loopback-debug-listener.md`, Appendix B, 25 files against `a805329f8`) is the shape the debug listener will take when it is upstreamed. If it is upstreamed differently, S-001 and S-002 need re-reading, the rest of this report does not.
- Deployments of the node through the testing framework go through the node repository's `tests/testing_framework` crate, which is the only consumer of the testing repository found (`Cargo.toml:104-105,178-181`; `tests/testing_framework/Cargo.toml:17`).

## 3. Method

- Working through the three items of `#281` (parent `#18`, spun out of the #260 report §4.6 and PR #263 LB-003) against fresh clones of both repositories, with `git show <rev>:<path>` to compare the issue's commit, the pinned revision and HEAD.
- Full-text search of `logos-blockchain` at `3d5d419e` for `debug_address`, `pprof`, `profiling`, `listen_address`, `UNSPECIFIED`, `TRACING_FILTER`, `create_api_configs`, `launch_ready`; of `logos-blockchain-testing` at `1fdd8b96` for `0.0.0.0`, `127.0.0.1`, `debug_address`, `api_port`, `listen`, `backend`, `profiling`, `pprof`, `network_mode`, `hostNetwork`, `sidecar`; `git log --diff-filter=D` on the two deleted files the issue cites and `git log -S'0.0.0.0:{'` on the whole tree.
- Upstream pull-request search in both repositories for `debug_address`, `debug listener`, `loopback`, `tracing filter`, `pprof`, `profiling`, `cfgsync` (open and closed): no PR carries the #260 patch.
- Spec conformance: not applicable; no specification governs bind addresses of a test harness. The four documents listed in the header were read in full before the code as the README requires.
- Automated tooling: none. Dynamic testing: none (see Out of scope).

## 4. Findings

None. The three items, what was checked, and the result:

### 4.1 Item 1 — a per-node loopback `debug_address` in the cfgsync config builder and the `server.rs` rewrite path

**The cited code no longer exists.** At `1b336c2c` the testing repository did template the node's API address by string (`testing-framework/tools/cfgsync/src/config/builder.rs:263` `format!("0.0.0.0:{}", host.api_port)`; `server.rs:278` `json!(format!("0.0.0.0:{api_port}"))` under `/http/backend_settings/address`, a config path the node has not used since the `api.backend.listen_address` rename). Both files were deleted on 2026-01-19 by `77ae507` ("feat: refactor for using external cucumber (#6)"); `b62e712` (2026-02-16, "Decouple nomos and core") and `7da3df4` (2026-03-09, "Extract cfgsync into standalone crates") finished the split. At `1fdd8b96` the only address templating in the testing repository is `CfgsyncConfigOverrides::port` (`cfgsync/core/src/render.rs:59-62`), which is the cfgsync *server's* own HTTP port, and the only `0.0.0.0` strings outside test fixtures are the two cfgsync server binds (#289). The `README.md` at HEAD states the model: "Application repositories provide their node configuration, clients, readiness checks, and backend-specific launch settings."

**Where the rewrite actually lives.** The node repository consumes the testing repository as a git dependency at `db880d61` (`Cargo.toml:104-105,178-181`) and assembles node configuration itself:

```rust
// logos-blockchain @ 3d5d419e, tests/testing_framework/src/node/cfgsync.rs:108-115
const fn apply_launch_ready_bind_addresses(config: &mut RunConfig) {
    config
        .user
        .api
        .backend
        .listen_address
        .set_ip(IpAddr::V4(Ipv4Addr::UNSPECIFIED));
}
```

called from the three artifact builders at `cfgsync.rs:49,68,93`. Per-node ports for the local deployer are reserved on loopback in `tools/config/src/api.rs:14-23` (`create_api_configs`, one `get_reserved_available_tcp_port()` per node id), consumed by `tests/testing_framework/src/node/configs/dynamic.rs:58`.

**The field does not exist upstream.** `AxumBackendSettings` at `nodes/node/binary/src/config/api/serde.rs:17-31` has `listen_address`, `cors_origins`, `timeout`, `max_body_size`, `max_concurrent_requests` and nothing else; `GeneralApiConfig` at `tools/config/src/api.rs:8-10` has `address` only; the whole tree has no `debug_address`. The #260 report's status is `fix-review` and its Summary says the four items "are delivered as one proposed patch against the target commit (Appendix B)"; no pull request in `logos-blockchain/logos-blockchain` carries it. The issue's sentence "the node also has `api.backend.debug_address`" is therefore true of the patch, not of `main`.

**The patch already does what item 1 asks, in the right repository.** Appendix B of `processed/260-loopback-debug-listener.md` changes `tools/config/src/api.rs` to

```rust
pub struct GeneralApiConfig {
    pub address: SocketAddr,
    /// Loopback address of the node's debug listener; each node on a host
    /// needs its own port.
    pub debug_address: SocketAddr,
}
// create_api_configs: address: reserved_local_address(), debug_address: reserved_local_address()
```

and narrows `apply_launch_ready_bind_addresses` to the `api` section so it can be unit-tested; the report's §4.5 runs two `profiling` nodes on one host with node 0 on a reserved loopback debug port and node 1 with `debug_address: ~`. Nothing is left for the testing repository to do, and nothing in the testing repository could do it: it has no node config to add a field to.

### 4.2 Item 2 — `build-bundle.sh` `curl` hints for `profiling` bundles

`scripts/build/build-bundle.sh` was deleted on 2026-03-29 by `79037d7` ("Refactor generic compose and k8s runners"); `scripts/build/` at HEAD holds `build-rapidsnark.sh` only. The hints the issue quotes (`http://<node-host>:8722/debug/pprof/profile`, `build-bundle.sh:398-403` at `1b336c2c`) are gone with it.

What survives of the same idea are two printers in the generic deployers, both present at the pinned `db880d61` and at HEAD:

- compose: `testing-framework/deployers/compose/src/provisioner/mod.rs:632-634` builds `http://{host}:{api_port}/debug/pprof/profile?seconds=15&format=proto` from the node's *published API port*; `log_profiling_urls` (`:612-620`) emits it at `info!` for every node on every deploy, unconditionally (`provisioner/mod.rs:329`, `deployer/orchestrator.rs:278`); `print_profiling_urls` (`:622-630`) prints it as `TESTNET_PPROF node_N=` under `TESTNET_PRINT_ENDPOINTS`.
- k8s: `testing-framework/deployers/k8s/src/deployer/orchestrator.rs:780-790` prints the same path appended to each node's base URL, under the same variable (`:690,700`).
- `book/src/deployer-compose.md:63`, `deployer-k8s.md:86`, `environment-variables.md:67` document both.

At `3d5d419e` these URLs are *correct*: the node mounts the pprof router on the API listener (`nodes/node/binary/src/api/backend.rs:261-272`, `app.merge(pprof_routes)` under `#[cfg(feature = "profiling")]`, one `TcpListener::bind(&self.settings.address)` at `:274`), so a `profiling` build deployed through the framework answers on the published port, which is exactly PR #263 LB-003's exposure. The port-forward the issue asks for is the shape the URL must take *after* the #260 patch, when the route moves to `127.0.0.1:<debug port>` inside the container and the published API port stops answering it. That change belongs with the removal of the printers, which is #290's task; S-002 records the dependency so #290 is not done in a way #260 immediately invalidates.

### 4.3 Item 3 — a test mirroring `launch_ready_rewrite_opens_the_api_listener_but_not_the_debug_listener`

The test exists only in the #260 patch (Appendix B, `tests/testing_framework/src/node/cfgsync.rs`), not at `3d5d419e`: `grep -rn launch_ready tests tools` finds the three call sites and the function. There is no counterpart to write in the testing repository because there is no node config there to rewrite; a test of "the rewrite leaves `debug_address` alone" can only live next to the rewrite, which is where the patch puts it.

### 4.4 Ruled out

- **Several nodes in one network namespace.** The issue names "k8s pods with sidecar nodes, host-network compose". Neither exists: the k8s deployer renders one `Deployment` with `replicas: 1` and a single container per node (`testing-framework/deployers/k8s/src/env.rs:220`), with no sidecar; the compose deployer has no `network_mode` anywhere (`grep -rn network_mode testing-framework/deployers/compose/src` is empty; `docker/platform.rs:3` is an `extra_hosts` comment), and the node's own `deployment/compose.run.yml` uses bridge networking with published ports. The one place several nodes share a namespace is the local deployer, where every node already gets a reserved loopback API port (`tools/config/src/api.rs:14-23`) and, with the #260 patch, a reserved loopback debug port the same way.
- **A second fixed-port loopback listener that would collide today.** The Tokio console defaults to `bind_address: 127.0.0.1` (`nodes/node/binary/src/config/tracing/serde/console.rs:38`) with a fixed port, but the framework sets `console: Layer::None` (`tools/config/src/tracing.rs:55`), so no framework-launched node opens it. Nothing else in the node binds a fixed port besides the API listener.
- **`PUT /admin/tracing/filter` on the API address.** Registered at `nodes/node/binary/src/api/routes.rs:67` (`lb_http_api_common::paths::admin::TRACING_FILTER`, `nodes/api-common/src/paths.rs:53`) on the same router the framework rewrites to `0.0.0.0`; unauthenticated, in every build. Already PR #217 LB-003 and moved to the debug listener by the #260 patch; not re-filed.
- **Does any shipped build carry `profiling`?** No `Dockerfile`, workflow, or script in `logos-blockchain` at `3d5d419e` mentions `profiling`; the exposure of §4.2 needs an engineer to build with `--features profiling` (PR #263 §4.0 inventory still holds).

## 5. Suggestions (non-security)

### S-001 · Upstream the #260 patch; it is the whole of items 1 and 3

| | |
|---|---|
| Category | Configuration |
| Target | `processed/260-loopback-debug-listener.md` Appendix B → `logos-blockchain` `tools/config/src/api.rs`, `tests/testing_framework/src/node/cfgsync.rs`, `nodes/node/binary/src/config/api/serde.rs` |

The debug listener, the reserved per-node loopback debug port and the rewrite test are written and dynamically checked, but only exist in a report appendix. Until a `logos-blockchain` PR carries them, every issue downstream of #260 (#281, #290's port-forward shape, #261's "compile `profiling` in CI") is reasoning about a node that does not exist on `main`. The patch was made against `a805329f8`; `3d5d419e` has moved on, so it needs a rebase, not a rewrite: none of the files it touches changed shape in the meantime as far as this review looked (`api.rs`, `cfgsync.rs:108-115`, `serde.rs` are byte-identical to the `-` sides of the Appendix B hunks).

### S-002 · Do #290 with #260 in mind: the pprof URL printers point at the listener the patch removes the route from

| | |
|---|---|
| Category | Configuration |
| Target | `logos-blockchain-testing` @ `1fdd8b96`: `testing-framework/deployers/compose/src/provisioner/mod.rs:612-634`, `testing-framework/deployers/k8s/src/deployer/orchestrator.rs:780-790`, `book/src/{deployer-compose.md:63, deployer-k8s.md:86, environment-variables.md:67}` |

Today the printed URL is right and the exposure it advertises is real (§4.2). After the #260 patch the same URL is wrong in both deployers: the route answers only on `127.0.0.1:<debug port>` inside the container, so `http://<host>:<published api port>/debug/pprof/profile` returns 404, while the printer still logs it at `info!` on every compose deploy. Since the framework is application-agnostic and this is its last node-specific string, the change that survives both states is to delete `profiling_url`, `log_profiling_urls`, `print_profiling_urls`, `print_node_pprof_endpoints` and the three book sentences from the testing repository (that is #290), and, if a hint is wanted at all, print it from the node repository's `tests/testing_framework` as `kubectl port-forward <pod> <local>:<debug port>` / `docker exec <container> curl http://127.0.0.1:<debug port>/debug/pprof/profile`, which is also what the #208 report's long-term recommendation and the framework's README already said for `tokio-console`. The k8s `Service` template (`env.rs:231`) exposes the API port only, so no manifest change is needed either way.

### S-003 · Issues that cite `logos-blockchain-testing` should cite the revision the node pins

| | |
|---|---|
| Category | Auditing and Logging |
| Target | `logos-blockchain` `Cargo.toml:104-105,178-181` (`rev = "db880d61…"`); this repository's issues #281, #289, #290 |

#281's observed state is eight months and a full repository redesign old; the node pins `db880d61` (2026-08-04) and the testing HEAD `1fdd8b96` (2026-09-14) rewrote compose provisioning again ("provision clusters and container stacks through the app model"). The pinned revision is the one that can affect a node deployment; HEAD is the one an upstream PR lands on. A testing-repository issue should state both, and a report should verify against both, as this one did. Of the two sibling issues, #290's printers exist at both revisions; #289's server bind exists at both.

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
