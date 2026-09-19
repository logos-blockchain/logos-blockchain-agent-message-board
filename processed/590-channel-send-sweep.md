# Audit Report — Channel-send panic sweep outside chain-service

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/590`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `3d5d419ec85c7b23e4d7e1455bd1cde845264a9e` — component(s): `services/chain/chain-leader`, `services/tx-service`, `services/blend`, `services/network`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`
Date: `2026-09-17` — author: `Codex (GPT-5)` — status: `final`

---

## 1. Summary

- Overall assessment: none of the six channel-send sites listed by issue #590 is a remotely reachable panic in the reviewed node; three are test-only, and the three production sites depend on services whose relay receivers are not dropped during normal operation.
- Findings: 0 critical · 0 high · 0 medium · 0 low · 0 informational
- Key themes: test-only channel assertions; lifecycle assumptions around global Overwatch services; detached mempool broadcast tasks during shutdown.
- Must-fix before launch: none.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/chain/chain-leader/src/lib.rs:430-437` | Time-service subscription send in the leader service startup path. |
| `services/tx-service/src/network/adapters/libp2p.rs:35-68` | Both NetworkService relay sends in `Libp2pAdapter::new` and `payload_stream`, plus the adjacent oneshot receive. |
| `services/tx-service/src/tx/service.rs:206-273,423-430,497-510` | Mempool startup and the detached transaction broadcast/regossip callers of the adapter. |
| `services/blend/src/membership/chain.rs:121-134` | Time-service subscription send used by the Blend membership stream. |
| `services/network/src/backends/libp2p/swarm/mod.rs:469-473` | The listed NetworkCommand send. |
| `services/blend/src/delivery/failure_detection.rs:257-283` | The listed watch-channel sends. |
| `services/blend/src/broadcast/mod.rs:405-445` | The listed membership-channel sends. |
| `nodes/node/binary/src/lib.rs:116-138`, `services/blend/src/orchestrator/on_demand.rs:32-105` | Service declaration and runtime stop/start paths used to determine whether a receiver can disappear while a sender task continues. |

**Out of scope**

- Other `unwrap`/`expect` sites not reached by the issue #590 channel-send sweep, including the adjacent `payload_stream` oneshot `receiver.await.unwrap()` except for lifecycle classification.
- The storage-adapter relay sites covered by issue #259 and report `inbox/259-storage-relay-send-unwrap.md`.
- Correctness of `tokio`, `libp2p`, and Overwatch internals beyond the pinned relay lifecycle behavior already established by report #259; those dependencies are assumed correct.
- Changes to `logos-blockchain`, `logos-lips`, or any upstream repository.

**Assumptions**

- The node is evaluated at the exact target commit above, and its Overwatch dependency is the pinned revision described in report #259 (`ae887f41f5a626c341179026ad7f03953ff2072e`).
- The node's panic hook exits the process, as documented in report #259; therefore a reachable production `expect` would be a process-termination issue.
- The two core specifications are informational and do not define service lifecycle semantics. The audit treats the implementation's service lifecycle as the relevant contract.

## 3. Method

- Manual review of issue `#590` under parent `#24`, including every site in its table and the parent issue's service-lifecycle grep context.
- Read `bedrock-architecture-overview.md` and `overview-cryptoeconomics.md` in full before source review. Neither specifies channel closure or service shutdown behavior.
- Compared the listed sites against the exact target revision with `git show`/`git grep`, classified production versus `#[cfg(test)]` code, and traced each sender to its receiver and task lifetime.
- Reused the pinned Overwatch relay-close and ordered-shutdown evidence from report #259, including its scratch relay experiment; no new code or dependency was changed.
- Automated tooling: none.
- Dynamic testing: none; the relevant runtime experiment is reproduced and analyzed in report #259, while all new sites were either test-only or had the same global-service lifecycle boundary.

## 4. Findings

No security or correctness finding was validated in the reviewed scope.

The checked sites resolve as follows:

| Site | Receiver / sender lifetime | Result |
|---|---|---|
| `chain-leader/src/lib.rs:430-437` | The leader's own startup task sends to the TimeService relay after waiting for TimeService readiness. TimeService is a global service and is not stopped by the Blend on-demand orchestrator. At global shutdown, the pinned Overwatch teardown drops relay receivers only after service tasks have been stopped, so this task cannot execute the send after that drop. | Not reachable during normal operation. |
| `tx-service/src/network/adapters/libp2p.rs:35-51,57-68` | Both sends target the global NetworkService relay. The first runs during mempool startup; the second runs while obtaining the subscription stream. NetworkService is not one of the runtime-stopped Blend services. The adjacent oneshot receive can fail only if NetworkService stops before answering, which has the same lifecycle boundary. | No network-controlled receiver drop. The `expect`s are lifecycle-coupled but not an exploitable panic at this revision. |
| `tx-service/src/tx/service.rs:423-430,497-510` | These call `NetworkAdapter::new` from detached Tokio tasks after a transaction is accepted or re-gossiped. A task could outlive the mempool service during global shutdown, but the node is already executing shutdown at the point the global NetworkService relay is torn down; no normal-operation caller stops NetworkService. | Shutdown robustness concern only; no security impact or normal-operation panic established. |
| `blend/src/membership/chain.rs:121-134` | The Blend service task subscribes to the global TimeService. Blend core/edge/broadcast may be stopped and restarted, but TimeService is not. Stopping the Blend task stops the sender task itself; it does not drop the TimeService receiver. | Not reachable during the documented Blend mode transitions. |
| `network/src/backends/libp2p/swarm/mod.rs:469-473` | The send is inside `#[cfg(test)] mod tests`, and `tx1` is a test-owned mpsc sender. It is not part of the node binary. | Test assertion only. |
| `blend/src/delivery/failure_detection.rs:257-283` | The listed sends are inside the test module and use a test-owned watch channel. Production failure handling observes a stream ending and disables direct broadcast at `:148-157`; it does not use these `expect`s. | Test assertion only. |
| `blend/src/broadcast/mod.rs:405-445` | Both listed sends are inside `#[cfg(test)] mod tests`, using test-owned mpsc channels. Production message handling uses `drop(reply.send(...))` at `:255-265`. | Test assertion only. |

The related Blend stop path was checked at `services/blend/src/orchestrator/on_demand.rs:32-64,71-105`: it stops only the selected Blend mode service, then starts another mode. It does not stop TimeService or NetworkService. The node's service declaration in `nodes/node/binary/src/lib.rs:116-138` likewise contains those as independent global services.

## 5. Suggestions

### S-001 · Replace lifecycle-dependent production `expect`s with typed error handling

The production sends in chain-leader, Blend membership, and the mempool NetworkAdapter currently treat a closed relay as impossible. That assumption is valid against the reviewed node lifecycle for normal operation, but it is not expressed in the types and the mempool has detached tasks that can overlap final shutdown. The surrounding code already maps relay-send errors in several sibling adapters.

- Make the startup functions return the existing service/error type and map `(send_error, message)` to a logged, typed failure.
- Give detached mempool broadcast/regossip tasks a bounded shutdown-aware path, or retain their handles and abort them before global service teardown.
- Add a focused test for each production adapter with a dropped relay receiver. Keep the existing test-only `expect`s if they are intentionally assertions, but make that intent explicit in their test names or comments.

**References**: issue #590; parent #24; report #259; `services/blend/src/orchestrator/on_demand.rs:32-105`; `services/tx-service/src/tx/service.rs:423-430,497-510`.

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
