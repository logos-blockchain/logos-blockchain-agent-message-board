# Audit Report — FFI host-abort paths: reproduction and fix verification

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/95`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `7b5e48b0fda2d5001cb8e62d6e2923f8306a7b3e` — component(s): `c-bindings` (the `logos_blockchain` cdylib)
Specs: `https://github.com/logos-co/logos-lips` @ `d6e824293579e5f53b8d278d0f710467fcae5fb8` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `wallet-technical-standard.md`, `mantle-transaction-encoding.md`; consulted by section: `bedrock-v1.1-mantle-specification.md` ("Zero Knowledge Signature Scheme (ZkSignature)")
Date: 2026-09-08 — author: Claude (Fable 5.1), session `01PncZM22n2hxaU837ZkqroT` — status: `fix-review`

---

## 1. Summary

- Overall assessment: both host-abort paths from report #94 reproduce end-to-end against the built `logos_blockchain` library at this commit, and the running node makes the situation worse than #94 described: once `start_lb_node` has returned, *any* panic anywhere in the host process ends it with exit status 1, because the node library installs a process-global exit-on-panic hook. A fix for all three paths is provided (`inbox/95-ffi-host-abort-paths.patch`) and was verified with the same programs and three new regression tests.
- Findings: 0 critical · 0 high · 3 medium · 1 low · 0 informational
- Key themes: panics crossing the C boundary; the embedded node behaving as if it owned the process; a checked-in genesis that the node decodes into garbage without complaint, which also stops any embedded standalone node from producing blocks.
- Must-fix before launch: LB-001, LB-002 and LB-003 (one patch); LB-004 before anyone relies on the standalone or default deployment files.


## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `c-bindings/src/api/lifecycle.rs`, `api/config.rs` | Node start and config commands: the two `Runtime::new().expect` sites and the config-parse error path that LB-002 goes through |
| `c-bindings/src/api/subscriptions.rs` | The three subscription functions and the thread their callbacks run on (LB-001) |
| `c-bindings/src/api/cryptarchia.rs` | `get_cryptarchia_info`, the API called from inside the callback in the LB-001 reproduction |
| `c-bindings/src/errors.rs`, `macros.rs`, `node.rs`, `callbacks.rs` | `OperationStatus::error`, the early-return macros and `FfiReturn`, the node handle and its `expect`s |
| `c-bindings/src/api/*.rs` (all 48 exported functions) | Wrapped in the panic barrier by the fix; no other behaviour change |
| `utils/src/yaml.rs` | How a YAML value reaches an error message (`deserialize_value_at_path`, `serde_ignored`) |
| `nodes/node/standalone-node-config.yaml`, `standalone-deployment-config.yaml` | The configuration the reproductions run the embedded node with |

**Out of scope**

The HTTP API and every node service the bindings call into, except for the exact panic and re-entrancy paths named below. The findings of report #94 other than LB-001 and LB-002 (LB-003 to LB-006, S-001 to S-007) were not re-examined. Assumed correct: `tokio` 1.52.3, `serde` 1.0.228 / `serde_core`, `serde_yaml` 0.9.34, `serde_ignored`, `cbindgen`, the C compiler.

**Assumptions**

The embedder is not malicious toward itself: it follows the header comments but makes ordinary mistakes (calls back into the library from a callback, ships a config file it did not fully vet). The library is built with the workspace profile, which leaves `panic = "unwind"` (issue #19); with `panic = "abort"` a panic barrier can catch nothing and both findings stay aborts.

## 3. Method

- Read the specifications listed in the header first, then parent issue #18, sub-issue #95, and the LB-001 / LB-002 sections of report #94 (`inbox/67-ffi-abi-hygiene.md`, PR #94).
- Read every file of `c-bindings/src` in full at the target commit, plus `utils/src/yaml.rs` and the two standalone config files.
- Confirmed the two panic paths in the vendored sources pinned by `Cargo.lock`: tokio's `Handle::block_on` → `context::enter_runtime` panic at `tokio-1.52.3/src/runtime/context/runtime.rs:68-73`; serde's `unknown_variant` message at `serde_core-1.0.228/src/de/mod.rs:252-264`, which interpolates the offending variant name with `{}` (no escaping).
- Built the `logos_blockchain` cdylib at the target commit (`cargo build -p logos-blockchain-c`, rustc 1.98.1 from `rust-toolchain.toml`, debug profile, aarch64 Linux) and wrote two C programs against `c-bindings/logos_blockchain.h` (Appendix B). Ran them first against the unmodified library, then against the library rebuilt with the fix (Appendix C).
- Ran the crate's unit tests before and after the fix, including three new regression tests.
- Automated tooling: `cargo test -p logos-blockchain-c`, `cargo clippy -p logos-blockchain-c` (workspace lint set), `cargo fmt`. Miri / sanitizers: not run.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Calling any API from a subscription callback ends the host process (reproduced) | Denial of Service | Medium | Low | Open, fix proposed |
| LB-002 | No panic barrier at the C boundary; a config parse error with an interior NUL aborts the host (reproduced) | Denial of Service | Medium | Medium | Open, fix proposed |
| LB-003 | `run_node_from_config` installs a process-global panic hook that exits the host on any panic | Denial of Service | Medium | Low | Open, fix proposed |
| LB-004 | Spec deviation: checked-in genesis inscriptions use the pre-1.1.2 encoding; the node decodes them silently into a NUL-prefixed chain ID and a 2029 genesis time | Configuration | Low | Low | Open |

### LB-001 · Calling any API from a subscription callback ends the host process (reproduced)

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `c-bindings/src/api/subscriptions.rs:203-209` (`subscribe_to_processed_blocks_sync`, callback invoked from `runtime_handler.spawn`); `c-bindings/src/api/cryptarchia.rs:103` (`get_cryptarchia_info_sync`, `Handle::block_on`) |
| Status | Open, fix proposed (Appendix C) |

**Description**

Report #94 established this path by reading: the three subscription functions run the embedder's callback on a tokio worker thread, and every other exported function reaches the node through `Handle::block_on`, which tokio refuses on a runtime thread by panicking (`tokio-1.52.3/src/runtime/context/runtime.rs:68-73`). This report ran it. `lb001_reentrancy.c` (Appendix B) starts the node with the standalone config, subscribes with `subscribe_to_processed_blocks`, and calls `get_cryptarchia_info(node)` from inside the callback. Against the unmodified library the first block ends the process:

```
[main] t=10s tip slot 0 height 0 mode 1
INFO logos_blockchain::chain::leader: proposed block HeaderId(eb524a16...) with 0 transactions (0 removed)
[callback] event 0: block json
[callback] calling get_cryptarchia_info from inside the callback...
ERROR logos_blockchain::node: A panic occurred panic_payload="Cannot start a runtime from within a runtime. This happens because a function (like `block_on`) attempted to block the current thread while the thread is being used to drive asynchronous tasks." panic_location="c-bindings/src/api/cryptarchia.rs:103:35" ...
exit 1
```

One detail differs from #94's prediction. The process did not reach the `extern "C"` abort ("panic in a function that cannot unwind", exit 134): it exited with status 1 from inside the panic hook, before any unwinding. That hook is LB-003. The outcome for the embedder is the same, a dead process, but it means a panic barrier in the bindings alone (the #94 recommendation) would not have fixed LB-001: the hook runs first.

The reproduction needed a regenerated genesis, because the checked-in standalone deployment never produces a block (LB-004). With a genesis time of "now", the standalone node proposed its first block 13 seconds after start.

**Exploit scenario**

Unchanged from #94: an embedder that reacts to a block by reading the tip, the balance, or the block's events, which is what the header comments invite, loses its process on the first block. No network input is needed.

**Recommendation**
- *Short term*: run the callbacks on a thread owned by the library, not on a runtime worker, so that re-entering the API is legal. The patch does this with one `std::thread` per subscription fed by an `mpsc` channel (`spawn_callback_thread` in `subscriptions.rs`); the NULL sentinel is delivered by that thread when the channel closes, which keeps the "exactly once" contract on shutdown. Document in the three subscription doc comments which thread runs the callback.
- *Long term*: keep the barrier of LB-002 underneath as defence in depth, and add `Handle::try_current().is_ok()` checks in the `*_sync` helpers so a future path that reaches `block_on` from a runtime thread returns `RuntimeError` instead of panicking.

**Verification after the fix**: same program, same config, library rebuilt with the patch:

```
[main] t=10s tip slot 0 height 0 mode 1
INFO logos_blockchain::chain::leader: proposed block HeaderId(34deb031...) with 0 transactions (0 removed)
[callback] event 0: block json
[callback] calling get_cryptarchia_info from inside the callback...
[callback] re-entrant call returned Ok, tip slot 19 height 1
[main] events=1 reentered=1; shutting down
[callback] event 1: NULL sentinel
[main] shutdown status 0
exit 0
```

The re-entrant call succeeds because the callback now runs on the library's own thread; the sentinel is delivered exactly once, during `shutdown_node`, and the process exits normally.

**References**: tokio `Handle::block_on` documentation ("panics if called within an asynchronous execution context"); report #94 LB-001.

### LB-002 · No panic barrier at the C boundary; a config parse error with an interior NUL aborts the host (reproduced)

| | |
|---|---|
| Severity | Medium |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | `c-bindings/src/errors.rs:41-43` (`OperationStatus::error`, `CString::new(...).expect`); `c-bindings/src/api/lifecycle.rs:42` (`start_lb_node`); all 48 `extern "C"` functions in `c-bindings/src/api/*.rs`; `lifecycle.rs:99` and `config.rs:143` (`Runtime::new().expect`) |
| Status | Open, fix proposed (Appendix C) |

**Description**

`OperationStatus::error` builds the message of every error status with `CString::new(message).expect(...)`. The messages echo parsed content, and serde's `unknown_variant` error interpolates the offending variant name unescaped (`serde_core-1.0.228/src/de/mod.rs:259-263`), so a YAML value with a `\0` escape in an enum field reaches `CString::new` with an interior NUL. `lb002_nul_message.c` (Appendix B) calls `start_lb_node` on a copy of the standalone config whose `tracing.logger.file.appender_type` is `"Sim\0ple"`. Against the unmodified library:

```
[main] calling start_lb_node on bad-config.yaml
thread '<unnamed>' (53713) panicked at c-bindings/src/errors.rs:42:14:
Message contained an interior NUL byte.: NulError(49, [67, 111, 117, 108, 100, ... "Could not parse config file: unknown variant `Sim\0ple`, expected `Simple` or `Rolling`"])
thread '<unnamed>' (53713) panicked at library/core/src/panicking.rs:225:5:
panic in a function that cannot unwind
  11: core::panicking::panic_cannot_unwind
  12: start_lb_node
  13: main at lb002_nul_message.c:10:49
thread caused non-unwinding panic. aborting.
exit code: 134
```

This run happens before any node exists, so it shows the pure C-boundary abort (`SIGABRT`, exit 134) that #94 predicted. The same panic raised after `start_lb_node` has succeeded (for example an `unwrap` in a service reached through `block_on`) goes through LB-003 instead and exits with status 1. Either way the embedder never sees a status.

**Exploit scenario**

A desktop wallet that embeds the node lets the user pick a config file. A file with `appender_type: "Sim\0ple"` (or any enum field with a NUL escape) makes the wallet die instead of showing "unknown variant". More generally, every one of the allowed `unwrap`/`expect` sites beneath the 48 entry points (issue #19) is a host crash rather than an error status.

**Recommendation**
- *Short term*: (1) make `OperationStatus::error` escape interior NULs instead of panicking; (2) wrap the body of every exported function in `catch_unwind` through one helper (`catch_panics` in `macros.rs`), mapping the payload to `OperationStatusCode::RuntimeError`; (3) replace the two `Runtime::new().expect` with error statuses. All three are in the patch.
- *Long term*: a regression test that injects a panic behind an entry point and asserts an error status (added: `node::test::panic_behind_an_entry_point_is_reported_as_a_runtime_error`), plus a clippy configuration that denies `unwrap_used`/`expect_used` in `c-bindings` specifically, since the workspace allows them.

**Verification after the fix**: same program, library rebuilt with the patch:

```
[main] calling start_lb_node on bad-config.yaml
[main] start_lb_node returned status 9, message: Could not parse config file: unknown variant `Sim\0ple`, expected `Simple` or `Rolling`
exit code: 0
```

Status 9 is `InitializationError`; the NUL is rendered as the two characters `\0`. The injected-panic regression test (`node.rs`) returns `RuntimeError` with the message `Panic caught at the FFI boundary: A valid \`tokio::Runtime\` not null pointer`.

**References**: Rust reference, "FFI and unwinding"; Rust 1.81 release notes (abort on unwind out of `extern "C"`); report #94 LB-002.

### LB-003 · `run_node_from_config` installs a process-global panic hook that exits the host on any panic

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `nodes/node/binary/src/lib.rs:226` (`set_hook(Box::new(log_and_exit_hook))` inside `run_node_from_config`); `nodes/node/binary/src/panic.rs:10-36` (`log_and_exit_hook`, ends with `std::process::exit(1)`); called from `c-bindings/src/api/lifecycle.rs:100` |
| Status | Open, fix proposed (Appendix C) |

**Description**

`run_node_from_config` is the function the C bindings call to build the Overwatch application. Before running it, it calls `std::panic::set_hook` with `log_and_exit_hook`, which logs the panic through `tracing` and then calls `std::process::exit(1)`. A panic hook is process-global and runs *before* unwinding starts, on the panicking thread, for every panic in the process. Consequences for an embedder:

- Any panic in the embedder's own Rust code, in another library it links, or in a thread unrelated to the node, ends the process after `start_lb_node` has returned. The host's own panic hook, if it had one, is silently replaced.
- Panics inside tokio tasks spawned by the node, which tokio would otherwise catch and turn into a `JoinError`, also end the process. Report #94 assumed such panics were contained ("these do not abort (tokio catches task panics)"); with the hook installed they are not.
- A `catch_unwind` barrier in the bindings cannot help, because the hook exits before the unwind reaches the barrier. This is what the LB-001 run showed.

The hook is right for the standalone binary, which owns its process, and wrong for a library. Nothing in the bindings' header mentions it.

**Exploit scenario**

The LB-001 run is one: exit status 1, no core dump, one log line if the embedder happened to install a `tracing` subscriber. Another, requiring no callback: the embedder's GUI thread panics on an unrelated bug; before this library was linked, the panic was caught by the embedder's `catch_unwind` around its event loop; after, the process disappears.

**Recommendation**
- *Short term*: move the `set_hook` call from `run_node_from_config` into the binary's `main` (in the patch: `nodes/node/binary/src/main.rs`, right before `run_node_from_config`). The binary keeps its behaviour, the library stops touching the host's hook.
- *Long term*: if the node wants to observe panics when embedded, expose an opt-in (`start_lb_node` flag or a `set_panic_callback` entry point) that forwards the payload to the embedder instead of exiting.

**References**: `std::panic::set_hook` documentation ("The hook is invoked ... before the panic runtime is invoked"); LB-001 run log.

### LB-004 · Spec deviation: checked-in genesis inscriptions use the pre-1.1.2 encoding; the node decodes them silently into a NUL-prefixed chain ID and a 2029 genesis time

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Configuration |
| Target | `nodes/node/standalone-deployment-config.yaml:119` and `nodes/node/binary/src/config/deployment/settings.yaml` (the `inscription` of the genesis `ChannelInscribe` op, identical 60 bytes in both); decoder `core/src/mantle/transactions/genesis_tx.rs:368-381` (`ChainId: BinaryDecode`) and `:410-416` (`CryptarchiaParameter`); `codec/src/bounded_vec.rs:110-119` |
| Status | Open |

**Description**

The genesis inscription carries `CryptarchiaParameter { chain_id, genesis_time, epoch_nonce }`. The specification (`bedrock-genesis-block.md`, "Cryptarchia Parameters", revision 1.1.2 of 2026-07-06) encodes it as a u8 length prefix, the UTF-8 chain ID, a u32 little-endian unix timestamp and the 32-byte nonce. The node's decoder follows the spec (`BoundedVec<u8, 1, 255>` uses a one-byte prefix, `GenesisTime(u32)`), since commits `27bc7e7ce` and `c207cc3f2` of 2026-07-06/07.

Both checked-in deployment files still carry the inscription generated on 2026-06-10 (`3f485b551`) with the previous layout, a u64 length prefix:

```
10 00 00 00 00 00 00 00                              u64 length = 16 (old layout)
73 74 61 6e 64 61 6c 6f 6e 65 2d 6c 6f 63 61 6c      "standalone-local"
91 69 fe 69                                          u32 genesis_time = 1778280849 = 2026-05-08T22:54:09Z
2d 2d df 91 ... 5c 06 05 1a                          32-byte epoch nonce
```

Decoded with the current (spec) layout, the same bytes give length 16 from the first byte, then a chain ID of seven NUL bytes followed by `standalon`, then `genesis_time` from the bytes `e-lo` of the name, 1869360485 = 2029-03-28T02:48:05Z. The decoder accepts all of it: `ChainId::try_from(Vec<u8>)` only checks UTF-8 and the length bound, and NUL is valid UTF-8. The node's own log confirms it on every start with either file:

```
ERROR [new] Chain ID        standalon cannot be represented as a C string: nul byte found in provided data at position: 0. `get_chain_id` will fail for this node.
INFO  logos_blockchain::chain::service: genesis time is in the future genesis_time=2029-03-28 2:48:05.0 +00:00:00
INFO  logos_blockchain::chain::service: entering AwaitingGenesisTime phase
```

Effects at this commit:

- The standalone node described in the repository README, and any embedded node started with either file (the `settings.yaml` one is `DeploymentSettings::default()`, used by `start_lb_node` when no deployment path or `DEPLOYMENT` variable is given), sits in `AwaitingGenesisTime` for two and a half years and never produces a block. The FFI reports `mode = NotStarted (2)` forever. `get_chain_id` returns an error.
- The node stays "healthy" from the outside: services start, `test_basic_lifecycle` passes, CI's `--check-config` step (`code-check.yml:154`) only deserializes the YAML, and `nodes/node/binary/src/config/tests.rs:411` only checks that it deserializes. Nothing decodes the inscription against the ceremony inputs.
- The `13f0c2236` and `b660a7161` edits of 2026-09-08 changed PoW values in these files by hand while leaving the inscription and the genesis notes (`100000`, `100`, `100`, `1`) out of step with `deployment/ceremony/genesis/standalone/stakeholders.yaml` (`100000000000000`, ...) and the SDP `min_stake.threshold` (`1` vs `1000000000`). Regenerating with `scripts/standalone-genesis-ceremony.sh` at this commit produces the spec layout (`10 73 74 ...`, 53 bytes) and a node that proposes blocks (this is how LB-001 was reproduced).

Which side is wrong: the checked-in artifacts. The decoder matches the spec. The spec itself could say what a conforming decoder must reject (see S-003).

**Exploit scenario**

Not an attack. A developer follows the README, runs the standalone node, and sees no block for as long as they care to wait; an embedder tests against the default deployment and concludes the FFI subscriptions never fire.

**Recommendation**
- *Short term*: regenerate both files with the ceremony tool (`scripts/standalone-genesis-ceremony.sh`; the devnet workflow for `settings.yaml`), and stop hand-editing generated files.
- *Long term*: add a test that decodes the inscription of each checked-in deployment file and asserts the chain ID equals the ceremony's `chain_id` and contains no control characters; reject control characters in `ChainId::try_from` (S-003).

**References**: `bedrock-genesis-block.md` § "Cryptarchia Parameters" and revision table entry 1.1.2; `deployment/README.md` § "Genesis ceremony layout".

## 5. Suggestions (non-security)

### S-001 · Unbounded event queue in the callback thread

The fix's `spawn_callback_thread` uses an unbounded `mpsc` channel so that a slow callback never blocks a runtime worker. A callback that is slower than block production for a long time grows the queue without limit. A bounded channel with `try_send` and a logged drop, or a documented "the embedder must return promptly", would bound it. Left unbounded in the patch to keep the "every processed block" contract.

### S-002 · `catch_unwind` cannot help under `panic = "abort"`

The panic barrier relies on the workspace's default `panic = "unwind"`. An embedder building the cdylib with `panic = "abort"` gets the old behaviour back with no warning. A `compile_error!` (or a documented requirement in `c-bindings/README`) would make that explicit.

### S-003 · `ChainId` accepts control characters

`ChainId::try_from(String)` (`genesis_tx.rs:323-331`) checks only length and UTF-8. Rejecting C0 control characters (at least NUL) would have turned LB-004 into a decode error at the first start, and would make `LogosBlockchainNode::new`'s "chain ID not representable as a C string" branch unreachable. Worth raising upstream on the spec as well: "Cryptarchia Parameters" constrains the chain ID's length and encoding but not its alphabet.

### S-004 · The subscription tests do not cover delivery

The crate has no test that a subscription delivers an event or the NULL sentinel, so S-007 of report #94 (sentinel on shutdown) is still untested. With a regenerated standalone genesis, a test can now get a block within about 15 seconds; `lb001_reentrancy.c` is the skeleton.

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

## Appendix B — Reproduction

Environment: Raspberry Pi 5 (aarch64), Linux 6.18, rustc 1.98.1, gcc, `cargo build -p logos-blockchain-c` (debug profile) at the target commit. The link step of this cdylib needs more than 4 GB of RAM with full debug info; on this machine it was linked with `cargo rustc -p logos-blockchain-c --lib -- -C link-arg=-Wl,--strip-debug`, which changes nothing about the behaviour under test.

Configs: `node-config.yaml` is `nodes/node/standalone-node-config.yaml` with `state.base_folder` set to an absolute temporary directory and `api.backend.listen_address` set to `127.0.0.1:0`. `bad-config.yaml` is the same file with `appender_type: Simple` replaced by `appender_type: "Sim\0ple"`. `deployment-config-fresh.yaml` was produced by `scripts/standalone-genesis-ceremony.sh`'s command with `inscribe.yaml` set to `chain_id: "standalone-local"` and `genesis_time:` the current UTC time (LB-004 explains why the checked-in file cannot be used).

Build and run:

```sh
gcc -std=c11 -I c-bindings -o lb001 lb001_reentrancy.c -L target/debug -llogos_blockchain -Wl,-rpath,$PWD/target/debug
gcc -std=c11 -I c-bindings -o lb002 lb002_nul_message.c -L target/debug -llogos_blockchain -Wl,-rpath,$PWD/target/debug
./lb002 bad-config.yaml deployment-config-fresh.yaml            # unfixed: SIGABRT, exit 134
./lb001 node-config.yaml deployment-config-fresh.yaml 300       # unfixed: exit 1 on the first block
```

`lb002_nul_message.c`:

```c
/* LB-002 reproduction: a config whose parse error message carries an
 * interior NUL byte. Expected before the fix: SIGABRT inside start_lb_node.
 * Expected after the fix: an error status is returned and printed. */
#include <stdio.h>
#include "logos_blockchain.h"

int main(int argc, char **argv) {
    if (argc < 3) { fprintf(stderr, "usage: %s <bad-config.yaml> <deployment.yaml>\n", argv[0]); return 2; }
    fprintf(stderr, "[main] calling start_lb_node on %s\n", argv[1]);
    FfiInitializedLogosBlockchainNodeResult r = start_lb_node(argv[1], argv[2]);
    if (r.error.code == Ok) {
        fprintf(stderr, "[main] unexpected: node started\n");
        shutdown_node(r.value);
        return 1;
    }
    fprintf(stderr, "[main] start_lb_node returned status %d, message: %s\n",
            (int)r.error.code, r.error.message ? r.error.message : "(null)");
    if (r.error.message) free_cstring(r.error.message);
    return 0;
}
```

`lb001_reentrancy.c`:

```c
/* LB-001 reproduction: call an API function from inside a subscription
 * callback. Build against c-bindings/logos_blockchain.h and link with
 * -llogos_blockchain. Exit codes: 0 = callback re-entry returned a status
 * (no abort), otherwise the process was killed by SIGABRT (LB-001). */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <stdatomic.h>
#include "logos_blockchain.h"

static struct LogosBlockchainNode *g_node = NULL;
static atomic_int g_events = 0;
static atomic_int g_reentered = 0;

static void on_event(const char *json) {
    int n = atomic_fetch_add(&g_events, 1);
    fprintf(stderr, "[callback] event %d: %s\n", n,
            json ? "block json" : "NULL sentinel");
    fflush(stderr);
    if (json == NULL) return;
    /* The "obvious" embedder pattern: on each block, ask for the tip. */
    fprintf(stderr, "[callback] calling get_cryptarchia_info from inside the callback...\n");
    fflush(stderr);
    FfiCryptarchiaInfoResult r = get_cryptarchia_info(g_node);
    atomic_store(&g_reentered, 1);
    if (r.error.code == Ok) {
        fprintf(stderr, "[callback] re-entrant call returned Ok, tip slot %llu height %llu\n",
                (unsigned long long)r.value->slot, (unsigned long long)r.value->height);
        free_cryptarchia_info(r.value);
    } else {
        fprintf(stderr, "[callback] re-entrant call returned status %d: %s\n",
                (int)r.error.code, r.error.message ? r.error.message : "(null)");
        if (r.error.message) free_cstring(r.error.message);
    }
    fflush(stderr);
}

int main(int argc, char **argv) {
    if (argc < 4) {
        fprintf(stderr, "usage: %s <node-config.yaml> <deployment-config.yaml> <seconds>\n", argv[0]);
        return 2;
    }
    int wait_s = atoi(argv[3]);
    FfiInitializedLogosBlockchainNodeResult start = start_lb_node(argv[1], argv[2]);
    if (start.error.code != Ok) {
        fprintf(stderr, "start_lb_node failed: %d %s\n", (int)start.error.code,
                start.error.message ? start.error.message : "");
        return 3;
    }
    g_node = start.value;
    fprintf(stderr, "[main] node started, subscribing\n");
    struct OperationStatus s = subscribe_to_processed_blocks(g_node, on_event);
    if (s.code != Ok) {
        fprintf(stderr, "subscribe failed: %d %s\n", (int)s.code, s.message ? s.message : "");
        return 4;
    }
    for (int i = 0; i < wait_s; i++) {
        sleep(1);
        if (atomic_load(&g_reentered)) break;
        if (i % 10 == 9) {
            FfiCryptarchiaInfoResult r = get_cryptarchia_info(g_node);
            if (r.error.code == Ok) {
                fprintf(stderr, "[main] t=%ds tip slot %llu height %llu mode %d\n", i + 1,
                        (unsigned long long)r.value->slot, (unsigned long long)r.value->height,
                        (int)r.value->mode);
                free_cryptarchia_info(r.value);
            }
        }
    }
    fprintf(stderr, "[main] events=%d reentered=%d; shutting down\n",
            atomic_load(&g_events), atomic_load(&g_reentered));
    struct OperationStatus sd = shutdown_node(g_node);
    fprintf(stderr, "[main] shutdown status %d\n", (int)sd.code);
    return atomic_load(&g_reentered) ? 0 : 5;
}
```

## Appendix C — The fix

`inbox/95-ffi-host-abort-paths.patch` applies on top of `7b5e48b0fda2d5001cb8e62d6e2923f8306a7b3e` with `git apply`. What it changes:

| File | Change |
|---|---|
| `c-bindings/src/macros.rs` | `catch_panics(body)`: `catch_unwind(AssertUnwindSafe(body))`, mapping a caught payload to a `RuntimeError` status through `FfiReturn` |
| `c-bindings/src/api/*.rs` | Every one of the 48 `extern "C"` functions wraps its body in `crate::macros::catch_panics(\|\| { ... })`; no signature or header change (`logos_blockchain.h` is unchanged after the build) |
| `c-bindings/src/errors.rs` | `OperationStatus::error` escapes interior NULs as `\0` instead of panicking; new `from_panic(payload)`; two unit tests |
| `c-bindings/src/api/lifecycle.rs`, `api/config.rs` | `Runtime::new()` failure becomes `RuntimeError` |
| `c-bindings/src/api/subscriptions.rs` | `spawn_callback_thread`: one named thread per subscription, fed by an `mpsc` channel; the three tasks send serialized events instead of invoking the callback; the NULL sentinel is delivered by the thread when the sender drops; `emit_json`'s two `expect`s replaced by logged skips; doc comments state the thread |
| `c-bindings/src/node.rs` | Regression test: a handle with null inner pointers makes `get_cryptarchia_info` panic behind the barrier; asserts `RuntimeError` and the message prefix |
| `nodes/node/binary/src/lib.rs`, `main.rs` | `set_hook(log_and_exit_hook)` moved from `run_node_from_config` into the binary's `main` |

Not changed, on purpose: `extern "C"` was not switched to `extern "C-unwind"` (the C embedder cannot handle an unwind either); the `expect`s in `subscribe_to_new_blocks_sync`'s task (`Block::reconstruct`) stay, since they run in a spawned task and, without LB-003's hook, are contained by tokio.

Verification: `cargo fmt --check` clean on both crates; the fixed `logos-blockchain-c` lib and its test harness compile with no warnings; `logos_blockchain.h` is byte-for-byte unchanged after the build. Unit tests (`--test-threads=1`):

```
test api::config::test::test_config_and_key_commands_roundtrip ... ok
test api::lifecycle::test::start_applies_environment_overrides ... ok
test api::lifecycle::test::test_basic_lifecycle ... ok
test errors::test::error_with_interior_nul_does_not_panic ... ok            (new)
test errors::test::from_panic_reports_a_runtime_error_with_the_payload ... ok (new)
test node::test::panic_behind_an_entry_point_is_reported_as_a_runtime_error ... ok (new)
test result: ok. 6 passed; 0 failed
```

Not run: `cargo clippy` (a check-profile build of the whole dependency tree did not fit the session's machine), Miri, sanitizers.

