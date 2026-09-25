# Audit Report · cfgsync deployment hygiene: what the served node config contains, who can read or replace it, and where the port is published

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/289`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `c4c86be18c58b5b09c3650e93871c8cfb624885b` · component(s): `tests/testing_framework/src/node/cfgsync.rs`, `tests/testing_framework/src/framework/{compose,k8s,local/provisioning.rs}`, `tests/testing_framework/src/node/configs/{dynamic,postprocess}.rs`, `tests/testing_framework/assets/runtime/`, `tools/config/src/{kms,blend,network,consensus}.rs`, `nodes/node/binary/src/config/{mod.rs,kms,network,wallet,blend}`, `kms/keys/src/keys/`, `libp2p/src/config/mod.rs`
Target: `https://github.com/logos-blockchain/logos-blockchain-testing` @ `8c3fe4664e1485bb47659220b38f4e028ab334ad` (HEAD, 2026-09-25), compared with `db880d61155db3a4f32177ef16b0f1e4bf22b1f6` (the revision the node pins, `Cargo.toml:105-106,180-183`) · component(s): `cfgsync/{core,adapter,runtime}/src`, `testing-framework/core/src/cfgsync/mod.rs`, `testing-framework/deployers/compose/src/{docker/config_server.rs,descriptor/node.rs,env.rs,infrastructure/environment.rs,container_stack.rs}`, `testing-framework/deployers/compose/assets/docker-compose.yml.tera`, `testing-framework/deployers/k8s/{src/manual.rs,assets/helm/tf-runner/templates/}`
Specs: `https://github.com/logos-co/logos-lips` @ `d788723992a805b395f377de6e6cf59859b47168` · read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md` (both in full); `key-types-and-generation.md` by section (listed in Method)
Date: `2026-09-25` · author: `Claude Code (research agent)` · status: `final`

Parent: #26 (configuration, genesis and deployment defaults). Source: the #262 report (PR #288, `processed/262-testing-profiling-bundles.md`, Scope and Appendix B). Related, not repeated: #260 (loopback debug listener), #281 (`processed/281-testing-cfgsync-debug-address.md`), #290 (pprof URL printers), #404 (63-LB-004, secret keys in plaintext YAML), #648 (secrets at rest outside the keystore), #85 report (`processed/85-kms-unsafe-feature.md`).

---

## 1. Summary

- Overall assessment: the rendered `/config.yaml` that cfgsync serves to a framework node is a full `RunConfig` with every secret key of that node inline (libp2p node key, Blend signing and quota keys, leader/voucher key, Blend and SDP note keys, and every wallet-account key of the deployment), and the cfgsync router serves it, and accepts replacements for it, with no authentication at all. On Kubernetes this path is live: the server is a `ClusterIP` Service reachable from any pod in the cluster, and the same secrets sit in a ConfigMap that is also mounted into every node pod. On compose, the code that would publish the port on every host interface is present in the node repository, but at `c4c86be1` it is not reached: the node's compose environment never enables the cfgsync sidecar (`ComposeConfigServerMode::Disabled` is the default and `LbcEnv` does not override it). Both repositories should switch to a loopback-published port before the compose path is turned back on. All keys involved are per-run test keys, so no production network is affected.
- Findings: `0` critical · `0` high · `0` medium · `3` low · `1` informational
- Key themes: "test-harness secrets served without authentication", "a write route that controls what a node runs on its next start", "the only all-interfaces publication in the framework", "latent in compose, live in k8s"
- Must-fix before launch: none for the node itself. Before re-enabling the compose cfgsync sidecar: LB-001 (loopback publish) and LB-003 (restrict the paths the client writes).

Answers to the four items of #289:

| Item | Result |
|---|---|
| 1. What does a rendered `/config.yaml` contain? | All of the node's secret keys, inline. `serialize_node_config` is `serde_yaml::to_string(&RunConfig)` (`tests/testing_framework/src/node/cfgsync.rs:55-59`). `RunConfig` is serializable only under the `testing` feature, which also turns on the KMS `unsafe` feature that derives `Serialize` for secret keys. There is no keystore path: the KMS backend is the inline preload map `kms.backend.keys`. Field list in §4 LB-002 and Appendix A. |
| 2. Is `POST /node` for `node-<i>` answerable by anyone who reaches the port? | Yes. The router has no auth layer (`cfgsync/core/src/server.rs:102-108`). The `registration` source accepts `/register` for any identifier with no check (`cfgsync/adapter/src/sources.rs:71-79`), and `/node` then returns that identifier's files (`:81-120`). Identifiers are `node-<i>` (compose, `infrastructure/ports.rs:114-117`) or `node-<i>` via `CFG_HOST_IDENTIFIER` (k8s, `framework/k8s/mod.rs:197`). |
| 2b. Does `POST /node/replace` let such a caller change what a node receives on its next start? | Yes. `replace_node_artifacts` stores an override for any identifier (`sources.rs:122-132`), which then wins over the materialized files on every later `/node` (`:106-110`). Node containers fetch on every start (`run_logos.sh:78-88`). The client writes each served file at the path the payload names when no route matches (`cfgsync/runtime/src/client.rs:70-80`), so the override is not limited to `/config.yaml` (LB-003). |
| 3. Publish on `127.0.0.1` or not at all? k8s? | **Loopback**, not "not at all": the compose provisioner's readiness probe connects from the host to `127.0.0.1:<port>` (`infrastructure/environment.rs:36,486-502`), so an unpublished port would fail every deployment, while node containers use the compose network alias `cfgsync` (`runtime.rs:92`, `CFG_SERVER_ADDR` at `:143-146`). `DockerPortBinding` cannot express a host IP today (`docker/config_server.rs:12-26,212-216`), so the fix is in the testing repository (diff in LB-001). On k8s, keep the `ClusterIP` Service (the runner reaches it through `kubectl port-forward`, `manual.rs:738-746`) and add a NetworkPolicy plus a Secret for the artifacts (LB-004). |
| 4. Minimal guard if it must stay published | A per-run bearer token checked by an axum layer on all three routes and sent by the client from an env var (Appendix B.2). A token inside `CFG_REGISTRATION_METADATA_JSON` would only cover `/register` and `/node`: `ReplaceNodeArtifactsRequest` has no metadata field (`cfgsync/core/src/protocol.rs:24-32`). |

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `logos-blockchain-testing` `cfgsync/core/src/{server.rs,source.rs,protocol.rs,bundle.rs,compat.rs}` | routes, handlers, bind address, `StaticConfigSource`/`BundleConfigSource`, request types |
| `cfgsync/adapter/src/{sources.rs,materializer.rs}` | `RegistrationConfigSource`: registration, resolve, override precedence |
| `cfgsync/runtime/src/{server.rs,client.rs,bin/*.rs}` | `cfgsync-server` (config-driven, bind), `cfgsync-client` (what it writes and where) |
| `testing-framework/core/src/cfgsync/mod.rs` | `build_static_artifacts`, `render_and_write_registration_server`, the `registration` server config |
| `testing-framework/deployers/compose/src/{docker/config_server.rs,descriptor/node.rs,env.rs,infrastructure/environment.rs,container_stack.rs,lifecycle/cleanup.rs}`, `assets/docker-compose.yml.tera` | `-p` rendering, node port publishing, `ComposeConfigServerMode`, port allocation, readiness probe, preserve mode, container capabilities |
| `testing-framework/deployers/k8s/src/manual.rs`, `assets/helm/tf-runner/templates/{bootstrap-service,bootstrap-deployment,configmap,node-services,node-deployments}.yaml` | Service type, where the artifacts live, how the runner reaches cfgsync |
| `logos-blockchain` `tests/testing_framework/src/node/cfgsync.rs`, `src/framework/{compose/{mod,runtime}.rs,k8s/{mod,assets}.rs,local/provisioning.rs,deployment_artifacts.rs,constants.rs}`, `src/node/configs/{dynamic,postprocess,deployment,wallet}.rs`, `assets/runtime/{Dockerfile.node,Dockerfile.cfgsync,scripts/*.sh}` | the adopter: config assembly, serializer, compose and k8s wiring, container entrypoints |
| `nodes/node/binary/src/config/{mod.rs,kms/serde.rs,network/serde/mod.rs,wallet/serde.rs,blend/serde/*.rs,cryptarchia/serde/leader.rs}`, `nodes/node/binary/Cargo.toml`, `kms/keys/src/keys/`, `libp2p/src/config/mod.rs`, `tools/config/src/{kms,blend,network,consensus}.rs` | which config fields hold secrets and how they serialize |

Every cfgsync file and `docker/config_server.rs` are byte-identical between the pinned `db880d61` and HEAD `8c3fe466` (`git diff --quiet db880d61 HEAD -- cfgsync testing-framework/deployers/compose/src/docker/config_server.rs`). Where other cited testing-repo files differ, both line numbers are given.

**Out of scope**

- Running anything: no cfgsync server was started, no request was sent to any endpoint, no container or cluster was brought up, and no Rust was built. Everything below is from code and configuration.
- The node's production keystore and `user_config.yaml` handling (#404, #648, #65 report). This report is about the framework's rendered test configs only.
- Third-party crates and tools assumed correct: `axum`, `tokio`, `serde_yaml`, Docker's port publishing semantics, Kubernetes Service/NetworkPolicy semantics, `kubectl port-forward`.

**Assumptions**

- Keys in framework configs are generated per run from a random 32-byte node id (`tests/testing_framework/src/node/configs/deployment.rs:409-431`) unless a `DeploymentSeed` is given. They protect a test network only.
- Docker's documented default: `-p <host>:<container>` with no host IP binds all host interfaces, and published ports are reached through Docker's own NAT rules, which host firewalls such as ufw do not filter by default.

## 3. Method

- Read issue #289, its comment, parent #26, and the #262 report (PR #288) and the #281 report, which cover the adjacent API-bind questions. Every file:line in #289 was re-located at the two target commits. Changes since the issue:
  - `cfgsync/core/src/server.rs:110-118` is now `build_legacy_cfgsync_router`. The router the server actually runs is `:102-108`, and it has no `/init-with-node` route.
  - The `0.0.0.0` bind is at `server.rs:126` and `runtime/src/server.rs:222`.
  - `descriptor/node.rs:99-129` is HEAD. The pinned revision has the same `127.0.0.1::{port}` binding at `:76,104-105`.
  - #289 cited `nodes/node/binary/src/config/mod.rs:71` as "`RunConfig` includes `kms`". At `c4c86be1`, `kms` is at `:71` of `UserConfig`, which `RunConfig` flattens (`:538-547`).
- Specifications, read before code. In full: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`. `key-types-and-generation.md` by section: Introduction; Overview; Construction › Non-ephemeral Quota Key, Non-ephemeral Signing Key, Ephemeral Signing Key, Proof of Work Nonce, Non-ephemeral Encryption Key, Ephemeral Encryption Key. These give what each leaked key would let its holder do (Appendix A). Following the orchestrator's scoping, I did not read the area specs that parent #26 lists (`bedrock-genesis-block.md`, `p2p-network-bootstrapping.md`) for this pass. Neither governs a test harness's config server, and nothing below rests on them. No spec deviation is reported. One test-fixture property that would be a deviation in production code is recorded as S-003.
- Static tracing, with `git`, `grep` and `git grep <rev>`: from `serialize_node_config` back to every key-bearing field; from each route handler to the `NodeConfigSource` implementation the server actually loads; from the compose and k8s environment traits to the container and Service specs. `git log -S` and `git merge-base` were used for the `ComposeConfigServerMode` history.
- Duplicate check for follow-ups: `search_issues` for cfgsync/compose, container capabilities, test-key derivation and secrets-at-rest. Only #289 itself, #404 and #648 matched, and they are referenced, not duplicated.
- Automated tooling: none. Dynamic testing: none (see Out of scope). The one behavioural claim that would need a run, whether the node's compose deployer can start nodes at all at `c4c86be1`, is S-001 and a follow-up.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | The compose runtime publishes the cfgsync port on every host interface, the only all-interfaces publication in the framework | Configuration | Low | Low | Open |
| LB-002 | cfgsync has no authentication: any caller that reaches it can register as `node-<i>` and receive that node's rendered config, with all of the node's secret keys inline | Data Exposure | Low | Low | Open |
| LB-003 | Unauthenticated `/node/replace` sets what a node receives on its next start, and the client writes each served file at whatever path the payload names | Access Controls | Low | Medium | Open |
| LB-004 | k8s: cfgsync is a `ClusterIP` Service with no NetworkPolicy, and the artifact set holding every node's keys is stored in a ConfigMap | Configuration | Informational | Medium | Open |

### LB-001 · The compose runtime publishes the cfgsync port on every host interface, the only all-interfaces publication in the framework

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Configuration |
| Target | `logos-blockchain` `tests/testing_framework/src/framework/compose/runtime.rs:71-98` (`build_cfgsync_container_spec`, `:94`); `logos-blockchain-testing` `testing-framework/deployers/compose/src/docker/config_server.rs:12-26,212-216` (`DockerPortBinding`, `build_docker_run_command`) |
| Status | Open (latent at `c4c86be1`, see below) |

**Description**

The node repository's compose environment builds the cfgsync sidecar spec with:

```rust
// tests/testing_framework/src/framework/compose/runtime.rs:92-94
    .with_network_alias("cfgsync".to_owned())
    .with_workdir("/etc/logos".to_owned())
    .with_ports(vec![DockerPortBinding::tcp(port, port)])
```

and the testing framework renders every binding as `-p <host_port>:<container_port>` with no host IP (`docker/config_server.rs:212-216`). `DockerPortBinding` has only `host_port` and `container_port` (`:12-26`), so a caller cannot ask for loopback. Docker binds such a port on all host interfaces. Inside the container the server binds `0.0.0.0:<port>` (`cfgsync/core/src/server.rs:126`, via `serve_from_config` → `serve_cfgsync`, `runtime/src/server.rs:146-154`). That in-container bind is needed for the compose network and is not the problem.

Every other port the framework publishes is loopback-only:

| What | Binding | Where |
|---|---|---|
| node API (node repo compose) | `127.0.0.1::<api_port>` (ephemeral host port) | `runtime.rs:45` → `descriptor/node.rs:99-129` (pin `:76,104-105`) |
| generic container-stack service ports | `127.0.0.1:<host_port>:<container_port>` | `container_stack.rs:475` |
| cfgsync sidecar | `<port>:<port>`, all interfaces | `runtime.rs:94` → `config_server.rs:215` |

Nothing on the host needs more than loopback. Node containers reach the server by the network alias: `CFG_SERVER_ADDR=http://cfgsync:<port>` (`runtime.rs:143-146`; default in `assets/runtime/scripts/run_logos.sh:63`). The only host-side consumer is the provisioner's readiness probe, a TCP connect to `127.0.0.1:<port>` (`infrastructure/environment.rs:36,486-502`; pin `:38,527-531`). The port is picked by binding `0.0.0.0:0` on the host (`environment.rs:294-310`; pin `:281-283`), so it is ephemeral, not the fixed 4400. That makes it harder to guess, but a port scan still finds it. With `COMPOSE_RUNNER_PRESERVE` set, a failed run keeps the sidecar running after the test ends (`lifecycle/cleanup.rs:126`, `DockerConfigServerHandle::mark_preserved`, `config_server.rs:165-167`).

**Why it is latent at `c4c86be1`.** `start_cfgsync_stage` returns before starting anything when `E::cfgsync_server_mode()` is `Disabled` (`infrastructure/environment.rs:230-240`; pin `:217-227`). The trait default is `Disabled` (`env.rs:175-177`; pin `:170-172`, introduced in testing-repo `36d7f3a`, 2026-04-10). The node's `impl ComposeDeployEnv for LbcEnv` (`framework/compose/mod.rs:13-59`) overrides `compose_descriptor`, `enrich_cfgsync_artifacts` and `cfgsync_container_spec`, but not `cfgsync_server_mode` or `prepare_compose_configs`. No code in either repository selects `ComposeConfigServerMode::Docker` (`grep` over both trees). So `build_cfgsync_container_spec` is only called from the image-presence check, which is itself gated on `Docker` mode (`environment.rs:408`), and the `-p` line is never executed for the node today. The binding is still the code that will run as soon as someone restores the sidecar, which the node's own entrypoint expects (S-001).

**Exploit scenario**

Once the sidecar is enabled, on a developer workstation or a self-hosted runner with a reachable interface, anyone on that network who finds the ephemeral port can use the unauthenticated routes of LB-002 and LB-003 for as long as the run lasts, or indefinitely with `COMPOSE_RUNNER_PRESERVE`. Host firewalls do not help by default, because Docker's published ports bypass them. The node API ports on the same host are not reachable from outside, so an operator who checked those would reasonably assume the stack is loopback-only.

**Recommendation**

- *Short term*: publish on loopback. The change belongs in the testing repository, because the node repository cannot express a host IP through `DockerPortBinding`. Make loopback the default, so the node repository only needs a pin bump:

```diff
--- a/testing-framework/deployers/compose/src/docker/config_server.rs
+++ b/testing-framework/deployers/compose/src/docker/config_server.rs
@@ -12,15 +12,23 @@
 #[derive(Clone, Debug)]
 pub struct DockerPortBinding {
+    pub host_ip: std::net::Ipv4Addr,
     pub host_port: u16,
     pub container_port: u16,
 }
 
 impl DockerPortBinding {
+    /// Publishes on the Docker host's loopback only. Containers on the compose
+    /// network reach the service by its network alias, not through this port.
     #[must_use]
     pub fn tcp(host_port: u16, container_port: u16) -> Self {
         Self {
+            host_ip: std::net::Ipv4Addr::LOCALHOST,
             host_port,
             container_port,
         }
     }
 }
@@ -212,5 +220,5 @@ fn build_docker_run_command(spec: &DockerConfigServerSpec) -> Command {
     for port in &spec.ports {
         command
             .arg("-p")
-            .arg(format!("{}:{}", port.host_port, port.container_port));
+            .arg(format!("{}:{}:{}", port.host_ip, port.host_port, port.container_port));
     }
--- a/testing-framework/deployers/compose/src/infrastructure/environment.rs
+++ b/testing-framework/deployers/compose/src/infrastructure/environment.rs
@@ -294,5 +294,5 @@
 pub fn allocate_cfgsync_port() -> Result<u16, ConfigError> {
     let listener =
-        StdTcpListener::bind((Ipv4Addr::UNSPECIFIED, 0)).map_err(|source| ConfigError::Port {
+        StdTcpListener::bind((Ipv4Addr::LOCALHOST, 0)).map_err(|source| ConfigError::Port {
```

  The readiness probe (`127.0.0.1:<port>`) and the node containers (`http://cfgsync:<port>`) keep working unchanged. "Not published at all" would need the readiness probe to move into the compose network (for example `docker exec` into the sidecar, or a probe container on the network). That is a larger change and gains nothing over loopback on a single host.
- *Long term*: add a unit test in `config_server.rs` that asserts every rendered `-p` argument starts with `127.0.0.1:`, matching the existing loopback assertion style for node descriptors. If a remote-Docker use case ever needs a non-loopback bind, make it an explicit opt-in (`DockerPortBinding::tcp_on(ip, …)`) that requires the token guard of Appendix B.2.

**References**: #289; #262 report Appendix B; Docker documentation on `docker run -p` host-IP defaults and on packet filtering of published ports.

### LB-002 · cfgsync has no authentication: any caller that reaches it can register as `node-<i>` and receive that node's rendered config, with all of the node's secret keys inline

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Data Exposure |
| Target | `logos-blockchain-testing` `cfgsync/core/src/server.rs:40-70,102-108` (`node_config`, `register_node`, `build_cfgsync_router`); `cfgsync/adapter/src/sources.rs:71-120` (`RegistrationConfigSource::{register,resolve}`); `logos-blockchain` `tests/testing_framework/src/node/cfgsync.rs:55-59` (`serialize_node_config`) |
| Status | Open |

**Description**

*What the server runs.* Both deployers render a `registration`-kind server config (`testing-framework/core/src/cfgsync/mod.rs:292-305`) and a precomputed `cfgsync.artifacts.yaml` built by `build_static_artifacts`, which puts one `/config.yaml` per node identifier (`:144-175`). The node repository calls this path through `render_and_write_registration_server` (`framework/k8s/assets.rs:124-150`). `cfgsync-server` loads it as `RegistrationConfigSource::new(MaterializedArtifacts)` (`runtime/src/server.rs:108-113,146-154`) and serves `build_cfgsync_router`:

```rust
// cfgsync/core/src/server.rs:102-108
pub fn build_cfgsync_router(state: CfgsyncServerState) -> Router {
    Router::new()
        .route("/register", post(register_node))
        .route("/node", post(node_config))
        .route("/node/replace", post(replace_node_artifacts))
        .with_state(Arc::new(state))
}
```

There is no middleware, no header or token check, and no check of the caller's address. `NodeRegistration` carries an `ip` field and adapter metadata (`protocol.rs:121-130`), but nothing compares them with the caller's address or with any secret. `register` inserts whatever identifier it is given (`sources.rs:71-79`, "Registered" for any string). `resolve` only requires that the identifier was registered (`:82-89`), and `MaterializedArtifacts` ignores the snapshot and returns the precomputed set (`:14-21`), so `/node` returns the node's files (`:112-119`) plus the shared `/deployment.yaml` (`framework/deployment_artifacts.rs:38-58`). The identifiers are predictable: `node-0`, `node-1`, … (`compose/src/infrastructure/ports.rs:114-117`; k8s `CFG_HOST_IDENTIFIER=node-<i>`, node repo `framework/k8s/mod.rs:197`). The legacy `/init-with-node` alias exists only in `build_legacy_cfgsync_router` (`server.rs:112-119`, re-exported by `compat.rs:12`), which neither binary serves.

*What `/config.yaml` contains.* `serialize_node_config` is `serde_yaml::to_string(config)` on the node's `RunConfig` (`cfgsync.rs:55-59`), after `rewrite_for_hostnames` has only changed the API bind, the peers, the NAT address and the Blend listen address (`:40-53,108-132`). `RunConfig` gets `Serialize` from the `testing` feature (`nodes/node/binary/src/config/mod.rs:538-547`). The framework enables that feature (`tests/testing_framework/Cargo.toml:38`), and it also enables `lb-key-management-system-service/unsafe` (`nodes/node/binary/Cargo.toml:97`). The node binary already turns `unsafe` on unconditionally (`:34`, the #85 report's subject). `unsafe` is what derives `Serialize` for `Key`, `Ed25519Key` and `ZkKey` (`kms/keys/src/keys/mod.rs:28-29`, `ed25519/mod.rs:30-31`, `zk/mod.rs:27-28`). Those serialize the raw secret (`ed25519/private.rs:61-68`, the 32 secret bytes; `zk/private.rs:20-22`, the field element). The KMS backend is the preload map, inline by design: `kms.backend.keys: HashMap<KeyId, Key>` (`nodes/node/binary/src/config/kms/serde.rs:6-16`). There is no keystore file or path anywhere in `RunConfig`.

The framework fills it with (`framework/local/provisioning.rs:707-795`, `kms` at `:780-784`; key sets from `tools/config/src/kms.rs:20-57` for the static topology, `tests/testing_framework/src/node/configs/dynamic.rs:102-135` for dynamic nodes, and `postprocess.rs:218-224` / `provisioning.rs:645-650` for wallet accounts):

| Config field in `/config.yaml` | Holds | Inline secret? | Role (key-types spec) |
|---|---|---|---|
| `network.backend.swarm.node_key` | libp2p Ed25519 secret, hex (`network/serde/mod.rs:36-38`, `libp2p/src/config/mod.rs:59-65`), bytes = node `id` (`tools/config/src/network.rs:37-42`) | yes | network identity (PeerId) |
| `kms.backend.keys[<blend NSK id>]` | `Key::Ed25519`, from the same `id` (`tools/config/src/blend.rs:28`) | yes | NSK: authenticates the node on the network, `provider_id` on ledger, derives the NEK |
| `kms.backend.keys[<blend NQK id>]` | `Key::Zk` (`blend.rs:29-30`) | yes | NQK: proves core-node membership for PoQ, `zk_id` on ledger |
| `kms.backend.keys[<blend note id>]` | `Key::Zk`, Blend service note (`tools/config/src/consensus.rs:408-409`) | yes | spends the Blend service note |
| `kms.backend.keys[<known_key id>]` | `Key::Zk`, regular (leader-stake) note (`consensus.rs:397-398`); also `wallet.voucher_master_key_id` (`provisioning.rs:772-776`) | yes | leader stake / PoL witness; voucher master key for leader-reward claims |
| `kms.backend.keys[<funding id>]` | `Key::Zk`, SDP funding notes (`consensus.rs:427-428`) | yes | pays SDP and leader-claim fees |
| `kms.backend.keys[<wallet account ids>]` | `Key::Zk` for **every** wallet account of the deployment, added to **every** node (`postprocess.rs:218-224`) | yes | wallet funds used by workloads |
| `blend.non_ephemeral_signing_key_id`, `blend.core.zk.secret_key_kms_id`, `wallet.voucher_master_key_id` | KMS key ids (hex of the public key, `tools/config/src/kms.rs:11-17`) | no, references into `kms.backend.keys` | |
| `wallet.known_keys`, `cryptarchia.leader.wallet.funding_pk`, `sdp.wallet.funding_pk` | public keys | no | |
| `deployment` (flattened `DeploymentSettings`), shared `/deployment.yaml` | genesis, protocol parameters | no | |

No ephemeral keys (ESK, EEK) and no PoW nonce appear. The node derives those at runtime.

**Exploit scenario**

A process that can reach the port (LB-001 when compose is enabled; any pod in the cluster on k8s, LB-004) posts a registration for `node-0` and then fetches `/node`. It receives node 0's full key set and the keys of every wallet account. The code places no limit on how many identifiers it can do this for. On the test network this lets the caller:

- impersonate the node's libp2p identity and Blend provider (NSK);
- produce proofs of quota as that core node (NQK);
- spend its stake, Blend and SDP notes and claim its leader vouchers;
- drain the workload wallets.

That can silently break the test run's results, for example make a liveness or reward scenario pass or fail for reasons outside the code under test. It also leaks nothing that protects a real network, since all keys are random per run (Assumptions).

**Recommendation**

- *Short term*: close the reach first: LB-001 for compose, LB-004 for k8s. Then add the per-run token of Appendix B.2 to the router, so reach alone is not enough. It is about 30 lines across `server.rs` and `cfgsync/core/src/client.rs`, with the token passed as an env var to the sidecar and to each node container.
- *Long term*: serve only what a node cannot generate itself. The framework could let each node generate its own keys on first start and register only public keys through `RegistrationPayload`, with the materializer building configs from those (`RegistrationSnapshotMaterializer`, `cfgsync/adapter/src/materializer.rs:12-23`, is already registration-driven). Then `/config.yaml` would carry no secrets. This is a design change and is not needed for the finding to be closed.

**References**: `key-types-and-generation.md` § Overview, § Construction (NSK, NQK); #404, #648 (secrets at rest in the production node, different path); #85 report (`unsafe` enabled unconditionally in the node binary).

### LB-003 · Unauthenticated `/node/replace` sets what a node receives on its next start, and the client writes each served file at whatever path the payload names

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Access Controls |
| Target | `logos-blockchain-testing` `cfgsync/core/src/server.rs:72-83,106` (`replace_node_artifacts`); `cfgsync/adapter/src/sources.rs:106-110,122-132`; `cfgsync/core/src/source.rs:80-93`; `cfgsync/runtime/src/client.rs:70-80,229-239,295-316` (`OutputMap::resolve_path`, `write_file`, `build_output_map`) |
| Status | Open |

**Description**

*Which routes change a later start.* Three code paths decide what a node receives the next time its container starts. Node containers run `cfgsync-client` on every start and then exec the binary on the fetched `/config.yaml` (`assets/runtime/scripts/run_logos.sh:78-88`; compose `restart: on-failure`, k8s pod restarts):

| Route | What the code permits | Effect on a later start |
|---|---|---|
| `POST /node/replace` | `RegistrationConfigSource::replace_node_artifacts` inserts `ArtifactSet::new(request.files)` for **any** identifier, existing or not, and always returns `Ok` (`sources.rs:122-132`). The `static` source does the same (`source.rs:80-93`). The request is `{identifier, files: [{path, content}]}` (`protocol.rs:24-32`) and needs no prior registration. | The override replaces the materialized node files on every later `/node` for that identifier, and the shared files are still appended (`sources.rs:106-110`). It persists for the server's lifetime. |
| `POST /register` | Overwrites the stored `NodeRegistration` (ip, metadata) for any identifier (`sources.rs:71-79`). | For this adopter, none: the precomputed `MaterializedArtifacts` ignores the snapshot (`sources.rs:14-21`). For an app that uses a snapshot-driven materializer (`runtime/src/server.rs:165-171`), a forged registration would change what every node receives. |
| `POST /node` | Read-only. | None. |

The framework's own legitimate caller of `/node/replace` is the k8s manual cluster, which pushes override artifacts through a `kubectl port-forward` before restarting a node with options (`deployers/k8s/src/manual.rs:717-754`; pin `:390-419`). Compose has no caller.

*What the client does with the files.* `run_client_from_env` builds an `OutputMap` with explicit routes for `/config.yaml` and `/deployment.yaml` only, and **no fallback** (`client.rs:295-316`). For every other file, `resolve_path` falls through to the path as served:

```rust
// cfgsync/runtime/src/client.rs:70-80
    fn resolve_path(&self, file: &NodeArtifactFile) -> PathBuf {
        self.routes
            .get(&file.path)
            .cloned()
            .or_else(|| { self.fallback.as_ref().map(|fallback| fallback.resolve(&file.path)) })
            .unwrap_or_else(|| PathBuf::from(&file.path))
    }
```

`write_file` creates parent directories and writes the content there (`:229-254`). No path is refused, no directory confines it, and duplicate paths are not rejected. So whoever controls the served payload chooses both the node's configuration and arbitrary other files in the container's filesystem.

In the node's images the client runs as root. Neither `Dockerfile.node` nor `Dockerfile.cfgsync` sets `USER` (`assets/runtime/Dockerfile.node:32-41`). In compose, `/etc/logos` is a read-write bind mount of the host's workspace stack directory (`runtime.rs:118-119`, `./stack:/etc/logos` with no `:ro`). That directory holds the node entrypoint, `/etc/logos/scripts/run_logos_node.sh` (`runtime.rs:21`), which runs on each start. The containers also get `SYS_ADMIN`, `SYS_PTRACE` and `seccomp=unconfined` by default (S-002). In k8s the asset mount is read-only (`node-deployments.yaml:45-46`), but the rest of the container filesystem is writable.

**Exploit scenario**

A process that can reach cfgsync can install an override for `node-2`. The next time `node-2`'s container starts, its client writes whatever files the override lists, at the paths the override names, as root, and then starts the node on the overridden `/config.yaml`. In compose those paths can include files in the host workspace directory that the entrypoint executes. The node's configuration can also be changed silently (peers, keys, genesis via the shared `/deployment.yaml`), which falsifies the test run without any error. The difficulty is Medium because the change only takes effect on a restart, which on compose (`on-failure`) needs the node to exit with an error, and on k8s needs a pod restart.

**Recommendation**

- *Short term*:
  1. Restrict the client to the files it expects, so a served payload cannot name other paths:

  ```diff
  --- a/cfgsync/runtime/src/client.rs
  +++ b/cfgsync/runtime/src/client.rs
  @@ -229,5 +229,11 @@
   fn write_file(file: &NodeArtifactFile, outputs: &OutputMap) -> Result<()> {
  -    let path = outputs.resolve_path(file);
  +    let Some(path) = outputs.try_resolve_path(file) else {
  +        bail!("cfgsync payload file {} has no configured output route", file.path);
  +    };
  ```

     Here `try_resolve_path` is `resolve_path` without the final `unwrap_or_else(|| PathBuf::from(&file.path))`. When `FallbackRoute` is used, it should also reject `..` components.
  2. Gate `/node/replace` behind the same per-run token as LB-002. Better still, mount the route only when the deployer needs it: k8s manual clusters need it; compose never calls it.
  3. Make `replace_node_artifacts` refuse identifiers that were never materialized, so it matches `register` in `StaticConfigSource` (`source.rs:51-63`).
- *Long term*: sign the artifact set per run (the runner holds a key; the client verifies before writing), so a compromised or reachable server cannot change what nodes run. Also drop root and `SYS_ADMIN` from node containers unless a test asks for them (S-002).

**References**: `book/src/cfgsync.md` (describes `replace_node_artifacts` as an administrative, k8s-only operation).

### LB-004 · k8s: cfgsync is a `ClusterIP` Service with no NetworkPolicy, and the artifact set holding every node's keys is stored in a ConfigMap

| | |
|---|---|
| Severity | Informational |
| Difficulty | Medium |
| Category | Configuration |
| Target | `logos-blockchain-testing` `testing-framework/deployers/k8s/assets/helm/tf-runner/templates/bootstrap-service.yaml:1-17` (`type: ClusterIP`, `:9`), `templates/configmap.yaml:14-19`, `templates/node-deployments.yaml:43-64`; `logos-blockchain` `tests/testing_framework/src/framework/k8s/mod.rs:81-83,141-162` |
| Status | Open |

**Description**

The node repository enables the k8s sidecar (`shared_service: Some(SharedServiceSpec::new("cfgsync", …, cfgsync_port(), cfgsync_yaml, artifacts_yaml, …))`, `framework/k8s/mod.rs:141-162`) and names the Service `logos-runner-cfgsync` (`:81-83`). The chart publishes it as `type: ClusterIP` (`bootstrap-service.yaml:9`, same at the pin). That is the right type: it is not exposed outside the cluster, node pods resolve it by Service DNS, and the runner reaches it through `kubectl port-forward` (`manual.rs:738-746`). The node API Services, by contrast, are `NodePort` (`node-services.yaml:29`), reachable on every cluster node's IP. That part is outside this issue.

Two gaps remain:

1. The chart has no NetworkPolicy (`grep -ri networkpolicy` over `deployers/k8s`: none). So any pod in any namespace of the cluster can reach `logos-runner-cfgsync.<ns>:4400` and use LB-002 and LB-003. When `LOGOS_K8S_NAMESPACE` is set, the namespace is shared across runs. Otherwise it is per run (`framework/k8s/mod.rs:57-67,96-102`).
2. The full `cfgsync.artifacts.yaml`, with every node's keys (LB-002), is rendered into the `tf-runner` ConfigMap as `bootstrap.artifacts.yaml` (`configmap.yaml:14-19`). Anyone with `get configmaps` in the namespace can read it without going through cfgsync. The same ConfigMap is mounted read-only into every node pod at the asset mount path (`node-deployments.yaml:43-46`). When bootstrap is enabled, the node pods' `items` list includes `bootstrap.artifacts.yaml` (`node-deployments.yaml:56` at HEAD, same block at the pin). So every node pod holds every other node's keys on disk, whether or not anyone calls cfgsync.

**Exploit scenario**

On a shared development cluster, a workload in another namespace, or any user with read access to ConfigMaps in the test namespace, obtains the run's keys or installs a replacement config (LB-003). The impact is limited to the test run.

**Recommendation**

- *Short term*: keep `ClusterIP`. Add a NetworkPolicy that admits ingress to the bootstrap pod only from the release's node pods:

```yaml
# templates/bootstrap-networkpolicy.yaml (new)
{{- if .Values.bootstrap.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "tf-runner.fullname" . }}-{{ .Values.bootstrap.serviceName }}
spec:
  podSelector:
    matchLabels:
      {{- include "tf-runner.selectorLabels" . | nindent 6 }}
      testing-framework/component: bootstrap
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              {{- include "tf-runner.selectorLabels" . | nindent 14 }}
      ports:
        - port: http
{{- end }}
```

  With common CNIs, `kubectl port-forward` enters the pod's network namespace through the kubelet and is not subject to pod-to-pod NetworkPolicy, so the runner's replace path keeps working. Confirm this on the target CNI. Also move `bootstrap.artifacts.yaml` from the ConfigMap into a `Secret` mounted only into the bootstrap pod, and remove it (and `bootstrap.primary.yaml`) from the node pods' `items` list (`node-deployments.yaml:53-64`). Node pods only need the two runner scripts.
- *Long term*: the per-run token (Appendix B.2), stored in the same Secret, so reaching the Service alone is not enough.

## 5. Suggestions (non-security)

### S-001 · The node's compose deployer never starts cfgsync, yet its node entrypoint requires it

`LbcEnv`'s compose impl (`framework/compose/mod.rs:13-59`) keeps two trait defaults:

- `cfgsync_server_mode() = Disabled` (testing `env.rs:175-177`; pin `:170-172`);
- `prepare_compose_configs() = Ok(())` (pin `env.rs:91-98`).

Because of these, no `cfgsync.yaml` or `cfgsync.artifacts.yaml` is written, and no sidecar is started (`environment.rs:230-240`). No code calls `write_registration_server_compose_configs` (testing `env.rs:488-518`). Contrary to `book/src/cfgsync.md:70`, the compose deployer does not call it; an app has to call it from `prepare_compose_configs`. The node containers still run `run_logos.sh`, which retries `cfgsync-client` against `http://cfgsync:<port>` 30 times at 3-second intervals and then exits 1 (`run_logos.sh:78-87`). Read from the code, `CUCUMBER_DEPLOYER_COMPOSE=1` cannot start a node at `c4c86be1`. Whether that holds in practice was not measured here (no Docker run). CI uses the local deployer (`tests/cucumber_tests/cucumber.rs:198-215`, no deployer env var in `.github/actions/run-cucumber-suite`), so CI would not notice. Either restore the two overrides, together with LB-001 and LB-003, or delete the compose integration and its dead `build_cfgsync_container_spec`. Filed as a follow-up.

### S-002 · Node containers get `SYS_ADMIN`, `SYS_PTRACE` and `seccomp=unconfined` by default, and a read-write mount of the host stack directory

Pinned template: unconditional (`docker-compose.yml.tera` at `db880d61`, lines 28-32). HEAD: conditional on `debug_capabilities`, which `NodeDescriptor::new` sets to `true` (`descriptor/node.rs:82`, template `:33-39`), so the node repository, which builds its descriptors through `with_loopback_ports` → `new`, still gets them. The node repository mounts `./stack:/etc/logos` read-write (`runtime.rs:119`). `book/src/cfgsync.md:88` says the rendered `cfgsync.artifacts.yaml`, with every node's keys, "remains in the stack directory". Once the sidecar is on, every node container would therefore see every other node's keys. Mount the stack read-only (`:ro`), write per-node files into a per-node directory, and make the debug capabilities opt-in (`TF_DEBUG_CAPABILITIES=1`).

### S-003 · Framework key fixtures make one leaked value equivalent to the node's whole key set, and the Blend NQK is computed from public data

All of a framework node's keys are functions of its 32-byte `id`:

- the libp2p key and the Blend NSK are the `id` itself (`tools/config/src/network.rs:37-42`, `blend.rs:28`);
- the leader, Blend-note and SDP ZK secrets are `prefix || id[..14]` (or `id[..13]`) read as a 128-bit scalar (`consensus.rs:35-38,373-381,397,408,427`), so they have at most 112 bits;
- the Blend NQK is `ZkKey::from(BigUint::from_bytes_le(nsk.public_key()))` (`blend.rs:29-30`), a function of the NSK *public* key, which is the on-ledger `provider_id`.

`key-types-and-generation.md` § Non-ephemeral Quota Key treats the NQK as a key the node generates and holds secret. Computing it from the public `provider_id` would be a spec deviation in node code. It is confined to the test-only `lb_config` crate (its only non-test consumer, `tools/blockchain-tools/src/genesis/inscription.rs:2`, imports two constants). This is acceptable for tests. Add a doc comment on `create_blend_configs_with_listening_host` and `create_utxos` saying these keys are deliberately weak and derivable, so the helpers are not reused for devnet or testnet key generation.

### S-004 · `kind: static` cfgsync silently drops `shared_files`

`BundleConfigSource::from_yaml_file` builds its map with `payload_from_bundle_node` (`cfgsync/core/src/source.rs:180-184,211-216`), which ignores `bundle.shared_files`. `bundle_to_payload_map` (`:114-129`) and the registration source (`adapter/src/sources.rs:106-119`) both append them. A `static` server configured with shared files serves nodes without them. Use `bundle_to_payload_map` in `from_yaml_file`.

---

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

Ratings. All four findings concern test tooling whose keys are random per run and protect no real network, so none reaches Medium. LB-001 to LB-003 are "defence-in-depth gaps" (Low). LB-001 is latent at `c4c86be1` and would become reachable the moment the compose sidecar is re-enabled. LB-004 is Informational because the Service is already cluster-internal.

## Appendix B · Supporting material

### B.1 Key-by-key trace for one compose/k8s node

`DeploymentPlan` → `build_node_config` (`cfgsync.rs:27-38`) → `build_node_run_config` (`provisioning.rs:669-685`) → `build_run_config` (`:707-795`), with `config.kms_config` from `create_kms_configs` (`tools/config/src/kms.rs:20-57`) plus wallet accounts (`postprocess.rs:218-224`) → `rewrite_for_hostnames` (`cfgsync.rs:40-53`) → `serialize_node_config` (`:55-59`) → `ArtifactFile("/config.yaml")` (`testing-framework/core/src/cfgsync/mod.rs:163-170`) → `cfgsync.artifacts.yaml` (`:249-262`) → k8s ConfigMap / compose stack dir → served by `/node`. `build_node_artifacts_for_options` (`cfgsync.rs:61-105`) serializes the same `RunConfig` (optionally a caller-supplied override) for the k8s replace path. It also carries the full key set.

### B.2 Minimal per-run token guard (if a port must stay reachable)

Server side (`cfgsync/core/src/server.rs`). Apply the layer to the router that `serve_cfgsync` and `cfgsync_runtime::serve_router` use:

```rust
use axum::{extract::Request, http::{StatusCode, header::AUTHORIZATION}, middleware::{self, Next}, response::Response};

async fn require_token(expected: Arc<str>, req: Request, next: Next) -> Result<Response, StatusCode> {
    let presented = req.headers().get(AUTHORIZATION).and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "));
    // constant-time compare, e.g. subtle::ConstantTimeEq
    match presented {
        Some(t) if bool::from(t.as_bytes().ct_eq(expected.as_bytes())) => Ok(next.run(req).await),
        _ => Err(StatusCode::UNAUTHORIZED),
    }
}

pub fn build_cfgsync_router(state: CfgsyncServerState) -> Router {
    let router = Router::new() /* routes as today */ .with_state(Arc::new(state));
    match std::env::var("CFGSYNC_TOKEN") {
        Ok(t) if !t.is_empty() => { let t: Arc<str> = t.into(); router.layer(middleware::from_fn(move |r, n| require_token(t.clone(), r, n))) }
        _ => router,
    }
}
```

Client side (`cfgsync/core/src/client.rs`): read `CFGSYNC_TOKEN` and add `bearer_auth` to all three requests. Deployer side: generate 32 random bytes per run and pass them as `CFGSYNC_TOKEN`:

- compose: in `build_cfgsync_container_spec`'s `env` (`runtime.rs:83`) and in each node's `base_environment` (`:134-155`);
- k8s: in the Secret proposed in LB-004, referenced by the bootstrap and node pods.

Rejected alternative: a token in `CFG_REGISTRATION_METADATA_JSON` checked by the source. It covers `/register` and `/node` but not `/node/replace`, whose request has no metadata (`protocol.rs:24-32`).

### B.3 Checked and ruled out

- **In-container `0.0.0.0` bind as a problem in itself.** It is needed so that node containers on the compose network and pods behind the Service can connect. Exposure is decided by publication (LB-001) and Service type (LB-004), not by the bind.
- **A second all-interfaces publication in compose.** `grep` for `-p`, `DockerPortBinding`, `ports:` over `deployers/compose`: node descriptors use `127.0.0.1::`, `container_stack.rs:475` uses `127.0.0.1:`. The cfgsync sidecar is the only other one.
- **The `/init-with-node` route being served.** Only `build_legacy_cfgsync_router` has it. Neither `cfgsync-server` nor `cfgsync_runtime::serve*` uses that router.
- **`/register` changing another node's config for this adopter.** `MaterializedArtifacts` is a constant materializer (`sources.rs:14-21`). Registrations gate `/node` but do not feed the output.
- **Keystore paths or file references in `RunConfig`.** None. The KMS config has one backend, `PreloadKmsBackendSettings`, and it is inline.
- **Secrets in `/deployment.yaml`.** It is `DeploymentSettings` (genesis block and protocol parameters). No key material.
- **Ephemeral Blend keys or the PoW nonce in the config.** Absent. They are derived at runtime.
- **The local deployer.** It writes the same `RunConfig` into each node's working directory and never starts cfgsync. That is file-at-rest exposure on the developer's machine, and it is covered by the #404 and #648 line for the production node.
