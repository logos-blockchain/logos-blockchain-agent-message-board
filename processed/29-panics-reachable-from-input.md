# Audit Report — Panics reachable from network, config, DB, or FFI input

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/29`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `nodes/node/binary`, `services/network`, `services/chain/chain-network`, `consensus/cryptarchia-sync`, `consensus/cryptarchia-engine`, `ledger`, `core/src/mantle`, `codec`, `services/storage`, `services/wallet`, `services/blend`, `c-bindings`
Specs: `https://github.com/logos-co/logos-lips` @ `accd97d9bfd94ce1d542c41d0f95582f278e57db` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md` (the two core overviews; issue #29 states no spec covers this area)
Date: 2026-09-10 — author: `agent (Claude)` — status: `final`

> Note on commit: the sub-issue names `19353c619` as the head to review. That commit is not on the default branch of the reviewed checkout; the current `main` head is `a805329f8`, which is what this report was read against.

---

## 1. Summary

- Overall assessment: the reviewed input-facing paths are hardened against panics — length prefixes are capped before allocation, collections are bounded types, and decoders return `Result` — so the sweep found no unauthenticated remote panic. The one material finding is architectural: a global panic hook turns **every** panic on **every** task into an immediate whole-node `exit(1)`. That inverts the fault model this sub-issue asks about (a panicking spawned task crashes the whole node rather than silently killing one service) and raises the blast radius of any future panic bug to a full-node crash.
- Findings: 0 critical · 0 high · 0 medium · 1 low · 1 informational.
- Key themes: a fail-stop panic policy amplifies every panic; detached tasks that *end* (rather than panic) stop silently with no restart; a few latent `expect()`s are held safe only by an invariant, not by construction.
- Must-fix before launch: none. Recommended: reconsider the process-wide `exit(1)` panic hook (LB-001), and convert the latent `expect()`s to errors (S-001…S-003).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `nodes/node/binary/src/panic.rs`, `.../lib.rs`, `.../main.rs` | the process-wide panic hook, where it is installed, config parsing |
| `consensus/cryptarchia-sync/src/libp2p/` | block/tip request-response wire protocol (packing, provider, downloader, messages) |
| `services/network/src/backends/libp2p/` | swarm, gossipsub, dial/retry tasks |
| `services/chain/chain-network/` | proposal ingest, orphan downloader, IBD, tip-poll |
| `consensus/cryptarchia-engine/`, `ledger/`, `core/src/mantle` | header/block/tx decode and apply, epoch/slot arithmetic |
| `codec` (spot checks only) | length-prefixed and bounded-vec decode; the deep codec review is issue #56 / PR #68 |
| `services/storage/src/backends/rocksdb.rs`, `.../api/backend/rocksdb/chain.rs` | DB read-back and key parsing |
| `services/wallet`, `services/blend`, `c-bindings` | spawned-task inventory and message handling |

**Out of scope**

- The `rust-rapidsnark` C++ prover/verifier behind FFI and the Groth16 proof verifiers (`ark-groth16`, `jellyfish`/Poseidon2): assumed correct, per issue #19. Whether a malformed proof byte string can panic inside these is not resolved here (see Notes).
- The `c-bindings` FFI host-abort surface and stale genesis config: covered by PR #105 (LB-001…LB-004) and PR #94; not re-reported.
- HTTP API authentication / CORS / TLS and SDP verification ordering: covered by PR #118 and PR #108.
- `libp2p`, `rocksdb`, `serde`, `axum`, `tokio` internals: assumed correct.

**Assumptions**

- Release profile as recorded in issue #19: `overflow-checks` is off, so ordinary `+ - * << as` do not panic; only true panics count (unwrap/expect on `Err`/`None`, out-of-bounds index/slice, division/remainder by zero, explicit `panic!`/`unreachable!`/`unimplemented!`/`assert!`, `Duration` arithmetic overflow, huge `Vec::with_capacity`).
- The clippy `unwrap_used`, `expect_used`, `indexing_slicing`, `panic`, `unreachable`, `todo`, `unimplemented` lints are allowed workspace-wide (issue #19), so these are found by hand.

## 3. Method

- Manual review of the in-scope paths, working through issue `#29` (all four bullets) with the repo-facts context from `#19`.
- Traced each untrusted-input entry point (libp2p request-response codec, gossipsub message handler, block/tx `Deserialize`/`decode`, config `deserialize_value_at_path`, rocksdb `load`/key parsing, FFI `extern "C"` params) forward to every panic site reachable with a value derived from that input.
- Enumerated every non-test `tokio::spawn` / `spawn_blocking` / `spawn_on` / `JoinHandle` and classified each by whether its handle is awaited or monitored, and by what happens if the task ends or errors.
- Automated tooling: none run this pass; findings are from source reading.
- Dynamic testing: none.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Process-wide `exit(1)` panic hook makes any reachable panic a whole-node crash | Denial of Service | Informational | Low | Open |
| LB-002 | Detached retry / subscription tasks stop silently when they end or error | Denial of Service | Low | High | Open |

### LB-001 · `Process-wide exit(1) panic hook makes any reachable panic a whole-node crash`

| | |
|---|---|
| Severity | Informational |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `nodes/node/binary/src/panic.rs:35` (`log_and_exit_hook`); installed at `nodes/node/binary/src/lib.rs:228` (`run_node_from_config`) |
| Status | Open |

**Description**

`run_node_from_config` installs a global panic hook before starting Overwatch:

```rust
// nodes/node/binary/src/lib.rs
set_hook(Box::new(log_and_exit_hook));            // line 228

// nodes/node/binary/src/panic.rs
pub fn log_and_exit_hook(panic_info: &PanicHookInfo) {
    // ... log the payload/location/backtrace ...
    std::process::exit(1);                         // line 35
}
```

A panic hook runs on **every** panic, on any thread or task, before unwinding and regardless of any surrounding `catch_unwind` or `spawn_blocking` boundary. Because this hook calls `std::process::exit(1)`, the node is fail-stop: any panic anywhere terminates the whole process.

This directly answers the sub-issue's fourth bullet, and inverts its premise. The bullet asks whether "a panic inside a `tokio::spawn`ed task doesn't crash the node — it silently kills that service." For this node the opposite holds: a panic in any spawned task (consensus, mempool, blend swarm, storage blocking pool, a detached retry task) does **not** silently kill one service — it kills the entire node. So there is no per-service fault isolation, and there is nothing to monitor a `JoinHandle` for: the process is gone before a supervisor could observe the task.

The consequence for the rest of this sweep is what matters: it means **any** panic that is reachable from network, config, DB, or FFI input — anywhere in the process — is a full-node denial of service, not a degraded-service condition. The sweep did not find such a reachable panic on the reviewed paths (see "Checked and ruled out"), but this hook is why the bar for a High finding here is a single reachable panic.

**Exploit scenario**

Not directly exploitable on its own; fail-stop is a defensible choice for a consensus node (halting is safer than continuing with a dead consensus task). The impact is that it removes the safety margin the checklist assumes: were a reachable panic found in, say, block validation, one crafted block from one unauthenticated peer would take every node that processes it offline simultaneously (a correlated network-wide halt), rather than disabling one subsystem on one node.

**Recommendation**

- *Short term*: document the fail-stop policy explicitly as a security property, and make sure operators run under a supervisor that restarts on exit code 1.
- *Long term*: consider a hook that logs and aborts only for genuinely unrecoverable faults, and lets Overwatch observe and restart a failed service for recoverable ones, so a single panicking task degrades rather than halts the node. If fail-stop is kept deliberately, treat every reachable-panic finding elsewhere as High by default.

**References**: issue #29 bullet 4; PR #105 LB-003 reported the same hook from the FFI-host angle (a panic anywhere in a host process that embedded the node ends the host).

### LB-002 · `Detached retry / subscription tasks stop silently when they end or error`

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Denial of Service |
| Target | `services/network/src/backends/libp2p/swarm/mod.rs:359` (dial retry) and `.../swarm/gossipsub.rs:92` (broadcast retry); `c-bindings/src/api/subscriptions.rs:87,204,268` (FFI block-stream tasks) |
| Status | Open |

**Description**

Several tasks are spawned with their `JoinHandle` dropped (detached). Because of LB-001 a *panic* in them kills the whole node, so the residual risk is the other case: a detached task that simply **ends or returns without panicking** is neither restarted nor observed.

- `spawn("logos/network/dial-retry", …)` and `spawn("logos/network/gossipsub-retry", …)` are self-contained one-shot retries, so ending is correct behaviour — noted only for completeness.
- The FFI block-subscription tasks (`subscribe_to_new_blocks_sync`, `subscribe_to_processed_blocks_sync`, `subscribe_to_lib_blocks_sync`) run `runtime_handler.spawn(async move { while let Some(event) = stream.next().await { … } … })` and end when the underlying broadcast stream closes; the callback is invoked once with a terminal value and no further events arrive. That is documented behaviour (the caller must re-subscribe), but it is a silent stop rather than a supervised one.

No non-test spawned task on a network-input path was found that ends early in a way that silently disables consensus, mempool, or blend while leaving the node apparently healthy. The `ChainNetwork` service is the counter-example of the correct pattern: on IBD failure it calls `overwatch_handle.shutdown()` and returns `Err` (`services/chain/chain-network/src/lib.rs:333-345`) rather than leaving a zombie.

**Exploit scenario**

Not remotely triggerable. Recorded as defence-in-depth: an FFI consumer that does not re-subscribe after stream closure silently stops receiving blocks with no error surfaced beyond a log line.

**Recommendation**

- *Short term*: none required.
- *Long term*: for any future long-lived detached task on a service path, prefer a monitored handle or an Overwatch-supervised service so an unexpected early return is observable.

**References**: issue #29 bullet 4.

## 5. Suggestions (non-security)

### S-001 · Uncle proof-of-leadership `expect` relies on chain/ledger-state coupling

`services/chain/chain-service/src/uncle.rs:131-134`, in `verify_uncle_pol`, looks up the ledger state of a received block's uncle's parent:

```rust
let parent_state = self
    .ledger
    .state(&uncle.header().parent())
    .expect("ledger state of a block on the chain must exist");
```

This runs while validating a block received from a peer that carries uncle headers. It is safe today because `verify_uncles_ancestry` has already confirmed the uncle's parent is on the chain within the uncle-reference window, and ledger states for on-chain blocks in that window are retained (states are pruned only for blocks pruned from the engine). The safety therefore rests on the coupling between the consensus branch set and the retained ledger-state set staying in lock-step across that window. If that invariant were ever broken (e.g. a future change to pruning), this becomes a remote panic — and, via LB-001, a remote node crash. Convert to a returned `Error::ParentIdNotFound`-style error.

### S-002 · Latent `expect`/`unreachable` on slot and epoch arithmetic

- `consensus/cryptarchia-engine/src/time.rs:263` — `EpochConfig::epoch` does `(u64::from(slot) / epoch_length).try_into().expect("Epoch should build from a correct configuration")`, narrowing `u64 → u32`. The divisor is `>= 1` by construction (sum of three `NonZero<u8>` times a `NonZero<u64>`), so no division-by-zero; the `expect` fires only if the slot number divided by the epoch length exceeds `u32::MAX`. A received block's slot is bounded to the node's wall-clock slot before this runs (`FutureBlock` check in `ledger/src/lib.rs`), so this is unreachable at any realistic uptime (~10^12 slots). Recommend returning an error rather than `expect`, as defence-in-depth against a future caller that passes an unbounded slot (e.g. an epoch-state query).
- `consensus/cryptarchia-engine/src/time.rs:166` — `Slot::from_offset_and_config` `expect`s that `slot_duration >= 1s`; this is enforced at config load by `MinimalBoundedDuration<1, SECOND>` on `SlotConfig::slot_duration`, so it holds by construction. No change strictly needed; a comment linking the two would help.

### S-003 · FFI block-stream `expect`s on stored/serialized blocks

`c-bindings/src/api/subscriptions.rs:106-108` (`.expect("Block should always build from valid block")`, twice) and `:173-177` (`emit_json`'s `expect("Serialization … should always succeed")` / `expect("Event JSON should not contain NUL bytes")`) unwrap on data read back from storage and re-serialized. These are safe today — stored blocks were validated on the way in, `BlockTransactions` is bounded to 1024, and JSON escapes control bytes so no interior NUL reaches `CString::new` — but they sit on the FFI subscription path and would be full-node crashes if the storage/serialization invariants ever slipped. This is in the FFI area already tracked by PR #105; folding these into that review is the natural home.

## Appendix — Checked and ruled out

A clean result is still a result. The following input-facing patterns were examined and found safe at this commit.

- **Sync wire protocol (`consensus/cryptarchia-sync/src/libp2p/`).** The length prefix is bounded before allocation: `unpack_from_reader` rejects any frame whose declared length exceeds `MAX_MSG_LEN` (16 MiB) *before* `vec![0u8; data_length]` (`packing.rs:63-73`), closing the ~4 GiB-prefix OOM. `additional_blocks` is capped at `MAX_ADDITIONAL_BLOCKS = 5` both at the behaviour (`behaviour.rs:257`) and in a custom `Deserialize` that also rejects duplicates (`messages.rs:46-110`). Request/response bodies are `UpperBoundedVec`-typed. `GetTipResponse`/`DownloadBlocksResponse` decode returns `Result`.
- **Block / transaction / proposal decode (`core/src/mantle`, `core/src/block`).** Op counts are bounded (`MAX_OPS_PER_TX = 255`), transaction inputs/outputs bounded (`MAX_TRANSACTION_INPUTS/OUTPUTS = 255`), block transactions bounded (`MAX_BLOCK_TRANSACTIONS = 1024`) and total size checked with `checked_add` (`block/mod.rs:251-274`). `Ops`/`OpProofs` columnar decode allocates `Vec::with_capacity(ops.len())` only after the op count passed the bounded-vec limit (`tx_list/op_proofs.rs:19`, `tx_list/array` decode uses the const `N`). Gossiped proposals decode into the same bounded `References` type; the fan-out reconstruct allocates `Vec::with_capacity(proposal.mempool_transactions().len())` which is `<= 1024` (`chain-network/src/lib.rs:1086`). Trailing bytes are rejected. The `unreachable!`s in `genesis_tx.rs`/`block/genesis.rs` are guarded by a preceding structural match or a bounded-construction invariant, not by peer input.
- **Ledger apply arithmetic (`ledger/`).** Balances, fees and gas use `checked_add`/`checked_sub`/`saturating_*` and return `LedgerError`/`GasOverflow` on overflow (`ledger/src/lib.rs:509-544`, `mantle/leader.rs`, `mantle/pow/mod.rs`). `reward_amount` uses `checked_div(...).unwrap_or(0)` so a zero voucher count does not divide by zero (`mantle/leader.rs:175-183`). Plain `+ - *` that could overflow *wrap* in release rather than panicking — those belong to arithmetic sweep #28, not here.
- **Config parsing.** Both the binary (`main.rs:56,79`) and the FFI (`c-bindings/src/api/lifecycle.rs:140,167`) parse config with `OnUnknownKeys::Fail`, so unknown/misspelled keys are rejected loudly — answering issue #19's `serde_ignored` question. Divisor-shaped config fields are `NonZero` types (`security_param: NonZeroU32`, `base_period_length: NonZero<u64>`, epoch stabilisation `NonZero<u8>`), and `slot_duration` carries a `MinimalBoundedDuration<1, SECOND>` bound, so the runtime-latent "zero divisor from config" class is closed at load time. `DeploymentSettings::default()` `expect`s the *embedded* default deployment parses — an internal constant, not operator input.
- **RocksDB read-back (`services/storage/`).** Block/tx values are opaque `Bytes`; header-id keys are parsed with `bytes.as_ref().try_into()` mapped to an error, not `unwrap` (`api/backend/rocksdb/chain.rs:104,140,163`), so a corrupt/truncated key yields an error rather than a panic. The one `expect` (`backends/rocksdb.rs:162`, `expect("Failed to join the blocking task")`) fires only if the bulk-store blocking task itself panicked — which under LB-001 has already exited the process.
- **Blend network ingest (`services/blend`, `blend/message`, `blend/network`).** `deserialize_encapsulated_message` bounds its allocation by the config `num_blend_layers`, not peer input, and rejects trailing bytes. Incoming-message PoQ verification runs on the blocking pool and its outcome is handled as `Verified`/`Failed` (`blend/network/src/core/poq_verification.rs`); the `unreachable!`s in `services/blend/src/core/mod.rs:1358` and `edge/current_epoch.rs:322` are guarded by the local encapsulation state machine, not by a received message.
- **Overwatch service failure.** The `ChainNetwork` service demonstrates the intended pattern: it shuts Overwatch down and returns `Err` on unrecoverable IBD failure rather than leaving a half-dead service (`chain-network/src/lib.rs:316-346`).

Not resolved (out of scope, flagged for a future pass): whether a malformed Groth16 proof byte string, or a malformed `PoQ`/leader proof, can panic inside `rust-rapidsnark` / `ark-groth16` / `jellyfish` when verifying an attacker-supplied proof. Given LB-001, a panic there would be a remote node crash, so a targeted review of the proof-verification FFI boundary against adversarial proof bytes is worthwhile.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
