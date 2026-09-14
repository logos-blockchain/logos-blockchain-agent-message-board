# Audit Report — `logos-blockchain-testing` profiling bundles: bundle builder, API bind address, live devnet check

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/262` (parent #26; source: PR #263 for #208, LB-003)
Target: `https://github.com/logos-blockchain/logos-blockchain-testing` @ `b32b3651de7548eb4e8fbb233fee702370201c8f` (2026-09-10) — component(s): `scripts/`, `cfgsync/{core,runtime,adapter,artifacts}`, `testing-framework/deployers/{compose,k8s}`, `.github/workflows/`, `book/`
Node repository read alongside: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — `tests/testing_framework/` (the adopter of the testing framework, pinned to testing repo rev `db880d61155db3a4f32177ef16b0f1e4bf22b1f6` in `Cargo.toml:104-105,178-181`), `deployment/`, `Dockerfile`, `nodes/api-common/src/pprof.rs`, `nodes/node/binary/src/api/backend.rs`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `bedrock-genesis-block.md`, `p2p-network-bootstrapping.md`
Date: `2026-09-12` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: the issue was filed against a checkout of `logos-blockchain-testing` from 2026-01-12 (`1b336c2c`). Every file it names was deleted from that repository between 2026-01-19 and 2026-03-27, eight months before the issue. At today's HEAD there is no bundle builder, no `--features profiling` path, no `nomos-bundle-meta.env`, and no cfgsync code that sets a node's API address. The only surviving piece of PR #263 LB-003 is in the node repository: its framework adapter binds the API to `0.0.0.0` inside containers, but its compose backend publishes the port on the Docker host's loopback and its node image is built with `--features testing` only. The public devnet (4 nodes) and testnet (4 nodes) were probed with a request that cannot start a profile; none of the eight nodes carries the pprof route.
- Findings: `0` critical · `0` high · `0` medium · `0` low · `0` informational
- Key themes: "stale premise", "vestigial pprof URLs printed by both deployers", "stale `nomos-node` paths in the boundary check"
- Must-fix before launch: none.
- Follow-ups filed: see the issue comment (cfgsync server exposure on shared compose hosts, under #26).

Status of the four task-list items of #262:

| Item | Result |
|---|---|
| 1. Warn in `build-bundle.sh` when `FEATURES` contains `profiling`; confirm deployers surface `features=` | **Moot.** `scripts/build/build-bundle.sh` and `build-linux-binaries.sh` were deleted in `79037d7` (2026-03-27, "Refactor generic compose and k8s runners"). No script, workflow, Dockerfile or Rust code in the repository passes `profiling` to cargo (§4.1). Nothing writes or reads `nomos-bundle-meta.env`. The deployers still print pprof URLs, unconditionally, for a route no framework build compiles in (S-001). |
| 2. Have cfgsync leave the API on loopback for profiling bundles | **Moot in the testing repo, partially open in the node repo.** The cfgsync that rewrote the API to `0.0.0.0:<api_port>` (`tools/cfgsync/src/config/builder.rs:263`, `server.rs:278`) was deleted in `77ae507` (2026-01-19). The replacement (`cfgsync/{core,runtime,adapter,artifacts}`) is application-agnostic and never touches a node's bind address (§4.2). The rewrite now lives in the node repository's adapter, `tests/testing_framework/src/node/cfgsync.rs:109-115`, which is the node-side half of #208 LB-003 and is already tracked by #260 (loopback-only debug listener). On compose the node repo publishes API ports as `127.0.0.1::<port>` (§4.2), so the container-side `0.0.0.0` is not reachable from outside the Docker host. |
| 3. Check shared devnet and testnet for a `profiling` bundle | **Done, negative.** No bundle metadata exists any more, so the check was made against the running nodes instead: all four devnet nodes (`65.108.203.235:18080-18083`, version `0.3.0-rc.2`) and all four testnet nodes (`65.109.51.37:18080-18083`) answer `404` on `/debug/pprof/profile` while answering `200` on `/cryptarchia/info`; the route is not compiled in (§4.3). |
| 4. Update `nomos-node` references in `build-bundle.sh` or confirm it is retired | **Retired** (item 1). Three unrelated `nomos` references remain: `scripts/run/check-boundaries.sh:10` and `book/src/tf-boundaries.md:62` point at `../nomos-node/tests/testing_framework/lb-topology`, a crate that does not exist in the node repository at `a805329f`, so the boundary check cannot pass; `Cargo.toml:49,52` carry a `nomos` keyword and a placeholder repository URL (S-002). |

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `logos-blockchain-testing` @ `b32b3651`: `scripts/`, `.github/workflows/*.yml`, `examples/*/Dockerfile`, `book/src/` | every build command and feature flag in the repository; the history of `profiling` handling (`git log -S profiling`) |
| `cfgsync/core/src/server.rs`, `cfgsync/runtime/src/server.rs`, `book/src/cfgsync.md` | what cfgsync is now, what it serves, and where it binds |
| `testing-framework/deployers/compose/src/{provisioner/mod.rs,deployer/orchestrator.rs,descriptor/node.rs,docker/config_server.rs}` | the pprof URL logging, the host it names, how ports are published |
| `testing-framework/deployers/k8s/src/{deployer/orchestrator.rs,env.rs}` | the pprof URL printing and where the base URL comes from |
| `testing-framework/assets/stack/scripts/run_logos.sh` | the only node-specific asset left in the repository |
| `logos-blockchain` @ `a805329f`: `tests/testing_framework/src/node/cfgsync.rs`, `src/framework/compose/runtime.rs`, `src/framework/k8s/mod.rs`, `src/framework/local/provisioning.rs`, `assets/runtime/Dockerfile.node` | the adopter: API bind rewrite, port publishing, node image features, binary profiles |
| `logos-blockchain` `deployment/{compose.run.yml,.env.devnet,.env.testnet}`, `Dockerfile`, `README.md:168-170` | how the public devnet and testnet are deployed and where their API ports are |
| `nodes/api-common/src/pprof.rs:86-93`, `nodes/node/binary/src/api/backend.rs:261-272` | re-checked unchanged since PR #263 (the patch has not landed) |

**Out of scope**

- The contents of published container images (`ghcr.io/logos-blockchain/logos-blockchain-node:*`); listing packages needs `read:packages`, which the reviewing token lacks (403). The live probe in §4.3 stands in for it.
- Kubernetes Helm chart values in `testing-framework/deployers/k8s/src/infrastructure/chart_values.rs` beyond the API Service.
- The security of the new cfgsync server itself (unauthenticated `/register`, `/node`, `/node/replace`; bound to `0.0.0.0` and published on every host interface by the node repo's compose runtime). It is outside the four items of #262 and is filed as a follow-up under #26 instead.
- The node's own `tokio-console` and `dhat-heap` features (PR #263 LB-004).
- Third-party crates assumed correct: `axum`, `tokio`, `pprof`.

**Assumptions**

- PR #263 Appendix C.3 is correct that a `profiling` build answers `400` to `/debug/pprof/profile?frequency=abc` from the `Query` extractor before the handler runs. That is what makes the live probe both safe (no profile starts) and discriminating (a non-`profiling` build answers `404`, as its unknown-route fallback).
- The devnet and testnet reached at the `PUBLIC_IP_ADDR` values in `deployment/.env.devnet` and `.env.testnet` are the shared deployments the issue means. The node repo's README (`:168-170`) points operators at the same devnet.

## 3. Method

- Specs: the two core overviews and the two documents parent #26 lists, read in full before any code. None of them constrains an operational bind address or a profiling route; they were read for the model, not for a conformance check, and no spec deviation is reported.
- Manual review of the in-scope paths, working through the four items of #262 against the testing repository at HEAD and the node repository at `a805329f`, with PR #263 (issue #208) as the source of the premise.
- History: `git log --diff-filter=D` for the files the issue names; `git log -S profiling` over `scripts/` and `testing-framework/`; confirmation that `1b336c2c` and the node-pinned `db880d61` are ancestors of HEAD.
- Live check (§4.3): `curl` against the eight public API ports, 2026-09-12 16:19-16:20 UTC, three requests per node: `/version` or `/cryptarchia/info` (reachability), `/debug/pprof/profile?frequency=abc` (route present → `400`, absent → `404`), `/debug/pprof/does-not-exist` (fallback baseline). No request with valid profiling parameters was sent.
- Tooling: none beyond `git`, `grep`, `curl` 8.x. No build.

## 4. Findings

None. The sections below record what was verified for each item.

### 4.1 Item 1 and 4: the bundle builder is gone, and nothing builds with `profiling`

At `1b336c2c` the script did what the issue says (`build-bundle.sh:327-331` appends `NOMOS_EXTRA_FEATURES` to `testing`; `:352-361` builds `nomos-node`, `nomos-executor`, `nomos-cli` with them; `:378-386` writes `features=` into `nomos-bundle-meta.env`; `:398-403` prints the `curl` lines when `FEATURES` contains `profiling`). That whole directory was removed:

```
79037d7 2026-03-27 Refactor generic compose and k8s runners
  scripts/build/build-bundle.sh
  scripts/build/build-linux-binaries.sh
  scripts/build/build_test_image.sh
```

`scripts/build/` now holds `build-rapidsnark.sh` only. Every remaining `cargo build` in the repository is for the framework's own example applications (`examples/{kvstore,queue,pubsub,openraft_kv,metrics_counter,multi_app}/Dockerfile`, `.github/workflows/tests.yml:38,59`); none passes `--features`. `lint.yml:61` runs clippy with `--all-features`, which compiles nothing that is deployed. A repository-wide search for `profiling` finds only the URL printers described in S-001 and their book entries. The repository's README now describes itself as "application-agnostic"; the node is one adopter among the examples, and the only node-specific file left is `testing-framework/assets/stack/scripts/run_logos.sh`, which executes whatever `logos-blockchain-node` binary the image provides.

### 4.2 Item 2: where the API bind address is decided now

The cfgsync the issue cites (`testing-framework/tools/cfgsync/`) was deleted in `77ae507` (2026-01-19). The current cfgsync is a generic artifact server (`book/src/cfgsync.md`): a node container registers, fetches an `ArtifactSet` of files, writes them, and starts the binary. The server has no notion of an API port; it binds its own listener to `0.0.0.0:<port>` (`cfgsync/core/src/server.rs:124-126`, `cfgsync/runtime/src/server.rs:222`) and serves what the adopter rendered.

The adopter is the node repository, and it is where the `0.0.0.0` rewrite survives:

```rust
// logos-blockchain a805329f tests/testing_framework/src/node/cfgsync.rs:109-115
const fn apply_launch_ready_bind_addresses(config: &mut RunConfig) {
    config.user.api.backend.listen_address.set_ip(IpAddr::V4(Ipv4Addr::UNSPECIFIED));
}
```

What that reaches depends on the backend:

| Backend | Node image / binary | API port on the outside | pprof reachable? |
|---|---|---|---|
| compose (node repo `src/framework/compose/runtime.rs:45-52`) | `assets/runtime/Dockerfile.node:27-29`: `cargo build --locked --release -p logos-blockchain-node --features testing` | `NodeDescriptor::with_loopback_ports` → `127.0.0.1::<api_port>` (testing repo `descriptor/node.rs:99-129`): published on the Docker host's loopback with an ephemeral host port | no route in the image; and the port is loopback-published |
| k8s (node repo `src/framework/k8s/mod.rs:77-79`) | same image | a cluster `Service`; `node_base_url` is the client's `base_url()` (a port-forward or the Service address) | no route in the image |
| local (`src/framework/local/provisioning.rs:449-467`) | `NodeBinaryProfile::TokioConsole` builds `--profile release-profiling --features testing,tokio-console`; the default profile builds `--features testing` (`:433`) | local process, whatever `listen_address` says | no: `tokio-console` does not enable `lb-http-api-common/profiling` (PR #263 §4.0) |

So the exposure #262 describes ("every framework-deployed profiling node exposes the route on all interfaces") needs a profiling image that no build path produces, and on compose the host side is loopback anyway. The recommendation to move the rewrite off the API listener is unchanged from PR #263 LB-002/LB-003 and is tracked by #260; nothing in the testing repository needs to change for it.

### 4.3 Item 3: the shared devnet and testnet do not carry the route

`deployment/compose.run.yml:50-53,72-75,94-97,116-119` publishes each node's API port on the host, and `.env.devnet` / `.env.testnet` give the public addresses. Probed 2026-09-12 16:19-16:20 UTC:

| Deployment | Node | `/version` | `/cryptarchia/info` | `/debug/pprof/profile?frequency=abc` | `/debug/pprof/does-not-exist` |
|---|---|---|---|---|---|
| devnet `65.108.203.235` | `:18080` | 200 (`"0.3.0-rc.2"`) | 200 | **404** | 404 |
| devnet | `:18081` | 200 | | **404** | 404 |
| devnet | `:18082` | 200 | | **404** | 404 |
| devnet | `:18083` | 200 | | **404** | 404 |
| testnet `65.109.51.37` | `:18080` | 404 (route predates this build) | 200 | **404** | 404 |
| testnet | `:18081` | 404 | | **404** | 404 |
| testnet | `:18082` | 404 | | **404** | 404 |
| testnet | `:18083` | 404 | | **404** | 404 |

A `profiling` build answers `400` here (the `Query<ProfileParams>` extractor rejects `abc` before `cpu_profile` runs, PR #263 C.3), so `404` with the same headers as the fallback means the router has no such route. This agrees with how the images are made: the root `Dockerfile:28-36` downloads the prebuilt `blockchain_module` for `LB_NODE_VERSION` through `lgpd` and runs no `cargo` at all, and that module is built from `flake.nix` with `-p logos-blockchain-c`, which has no features (PR #263 Appendix D).

Command used, for the next agent:

```sh
for p in 18080 18081 18082 18083; do
  curl -s -m 8 -o /dev/null -w "$p pprof=%{http_code}\n" \
    "http://<PUBLIC_IP_ADDR>:$p/debug/pprof/profile?frequency=abc"
done   # 400 = route present (profiling build), 404 = absent
```

## 5. Suggestions (non-security)

### S-001 · Both deployers still print pprof URLs for a route no framework build compiles in

`testing-framework/deployers/compose/src/provisioner/mod.rs:612-634` logs, at `info` level and on every compose deployment (`:329`, and `deployer/orchestrator.rs:278`), one `node profiling endpoint (profiling feature required)` line per node with `http://<host>:<api_port>/debug/pprof/profile?seconds=15&format=proto`; with `TESTNET_PRINT_ENDPOINTS` set it also prints them as `TESTNET_PPROF node_<i>=…`. The k8s orchestrator does the same under the env var (`deployer/orchestrator.rs:700,780-790`). The book documents both (`book/src/deployer-compose.md:63`, `deployer-k8s.md:86`, `environment-variables.md:67`). Since `79037d7` nothing in this repository can produce a binary that answers those URLs, and the repository no longer knows what application it is deploying, so the URL shape (`/debug/pprof/profile`) is a leftover of the node-specific era. Either drop the two printers, or move them behind an adopter hook (the k8s side already takes `node_base_url` from the `K8sDeployEnv` trait, `env.rs:680-682`) so an application that ships a profiling build opts in and states its own route. The same rev the node pins, `db880d61`, has the same code (`compose/src/deployer/orchestrator.rs:648`, `k8s/src/deployer/orchestrator.rs:730`).

### S-002 · Stale `nomos-node` path in the boundary check, and placeholder workspace metadata

`scripts/run/check-boundaries.sh:10` resolves `../nomos-node/tests/testing_framework/lb-topology` and exits 1 if it is missing (`:12-15`); `book/src/tf-boundaries.md:62` documents that path. The node repository is `logos-blockchain` and has no `lb-topology` crate at `a805329f` (`tests/testing_framework/` is a single crate), so the check cannot pass against the current adopter. Either point it at the current adopter crate or delete it. `Cargo.toml:49` keeps `nomos` in `keywords` and `:52` sets `repository = "https://example.invalid/nomos-testing-local"`; harmless, but it is what `cargo metadata` and any published crate would carry.

### S-003 · Close the loop in PR #263 LB-003

LB-003 in PR #263 cites `logos-blockchain-testing @ 1b336c2c` as the reviewed commit. That checkout was eight months behind `main` when the report was written, and the finding's testing-repo half describes code that no longer existed. The node-repo half (the `0.0.0.0` rewrite in `tests/testing_framework/src/node/cfgsync.rs`) is accurate at `a805329f`. When #263 is revised, LB-003 should cite the node repository as its target and drop the bundle-builder scenario; the recommendation (a loopback debug listener, #260) stands.

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

## Appendix B — Ruled out

- **Any `profiling` build path in `logos-blockchain-testing` at HEAD.** `grep -rn profiling` over `.rs`, `.sh`, `.yml`, `.toml`, `Dockerfile*`, `.md`: only the URL printers of S-001 and their docs. No `--features` on any `cargo build`.
- **Bundle metadata.** No file named or writing `nomos-bundle-meta.env`, `bundle-meta`, or `features=` anywhere.
- **cfgsync setting a node's API address.** `grep -rn 'api_port\|listen_address\|api.backend'` over `cfgsync/` and `testing-framework/{core,app,container}`: no hits. The only `0.0.0.0` in `cfgsync/` is the server's own bind (`core/src/server.rs:126`, `runtime/src/server.rs:222`).
- **The node repo's compose backend publishing API ports on all interfaces.** `with_loopback_ports` → `127.0.0.1::<port>` (`descriptor/node.rs:128`); only the cfgsync container uses `DockerPortBinding::tcp(port, port)` → `<port>:<port>` (`runtime.rs:94`, rendered at `docker/config_server.rs:215`), which is the follow-up filed under #26.
- **The pprof patch from PR #263 having landed.** `nodes/api-common/src/pprof.rs:87-91` and `backend.rs:261-272` are byte-identical to what PR #263 describes; the route is still unbounded when compiled in. Not a testing-repo matter.
- **The `/version` 404 on testnet meaning a non-node listener.** Same CORS headers as the devnet node, `200` on `/cryptarchia/info` and `/mantle/metrics`; the route was added in the node repo after the testnet image was cut.
