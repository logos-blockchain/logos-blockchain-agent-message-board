# Audit Report — FFI ABI hygiene (`c-bindings`)

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/67`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `19353c61963d4ef8c37ad00d24fdf0f08a482887` — component(s): `c-bindings` (5,325 lines), with the callees it hands host data to: `nodes/node/binary/src/cli/keys.rs`, `nodes/node/binary/src/config/mod.rs`, `kms/keys`
Date: `2026-09-07` — author: `claude-fable-5.1` — status: `final`

Re-verified against `origin/master` @ `c3ff08e4a9cbc8344c58a18ef7992a41403c7f9c` (nine commits later). The commits touch `c-bindings` only to add `get_chain_id` and rename transaction types. Every finding below is still present there; the counts move from 142 to 146 `unsafe` blocks and from 49 to 50 exported functions.

---

## 1. Summary

- Overall assessment: the crate is careful about null checks and ownership documentation at each entry point, but it has no panic barrier at the C boundary, tells embedders nothing about which thread their callbacks run on, and has opted out of the one lint that would force `SAFETY:` justifications.
- Findings: `0` critical · `0` high · `2` medium · `4` low · `1` informational
- Key themes: panics reach the host as process aborts; callback re-entrancy is a guaranteed abort; unchecked values crossing by value (enum, function pointer, one path pointer); `unsafe` is undocumented by policy.
- Must-fix before launch: LB-001 (document and guard callback re-entrancy) and LB-002 (catch panics at every `extern "C"` function). Both turn a host bug or an internal `unwrap` into an abort of the embedding application, which for a wallet or desktop node is a data-loss event.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `c-bindings/src/lib.rs`, `macros.rs`, `errors.rs`, `result.rs`, `callbacks.rs`, `node.rs`, `logging.rs` | Crate-wide lint policy, the two null-check/early-return macros, result and status layouts, callback wrapping, node handle ownership |
| `c-bindings/src/api/*.rs`, `api/types/*.rs` | All 49 `extern "C"` functions and all 20 `#[repr(C)]` types, read in full |
| `c-bindings/logos_blockchain.h`, `cbindgen.toml`, `build.rs` | The generated C header as the ABI of record: enum sizes, `uintptr_t`, `bool`, symbol names |
| `nodes/node/binary/src/cli/keys.rs`, `nodes/node/binary/src/config/mod.rs:564-577` | What `generate_key`, `add_key`, `remove_key` do with host-supplied secret material |
| `kms/keys/src/keys/**` | Whether the key types that receive host bytes zeroize on drop, and what they `Debug`-print |
| `Cargo.toml` (workspace lints, profiles), `.github/workflows/code-check.yml` | Which lints and dynamic checks actually run on this crate |

**Out of scope**

The HTTP API (parent issue #18's other half). The behaviour of the node services the bindings call into, except where a panic inside them can reach the C boundary. The embedder's own C code. Assumed correct: `cbindgen` 0.x layout emission, `tokio`, `ed25519-dalek`, `zeroize`, `libp2p-identity`, `serde_json`.

**Assumptions**

The host is not malicious toward itself: findings rated below assume an embedder that follows the header comments but makes ordinary mistakes (a null argument, an out-of-range enum, calling back into the API from a callback). The node is built with the workspace `[profile.release]` (issue #19): no `overflow-checks`, `panic = "unwind"` (not set to `abort`), `strip = true`.

## 3. Method

- Manual review of every file in `c-bindings/src`, working through issue #67's three items and the FFI questions in parent #18 (owner of each returned pointer, `catch_unwind`, callback threading and re-entrancy, null and length checks).
- Cross-checked each `#[repr(C)]` type and each function signature against the committed `logos_blockchain.h`, which `build.rs` regenerates on every build (the header is clean in `git status`, so it matches the source).
- Followed host-supplied secrets (`add_key`'s `key_hex`) through `parse_key_hex`, `parse_hex_ed25519_key`, `parse_hex_zk_key`, `AddKeyArgs::new`, `run_add_key`, and into `kms/keys` to see where copies live and whether they zeroize.
- Confirmed the tokio re-entrancy panic path by reading `tokio-1.52.3/src/runtime/handle.rs:342-370` and `runtime/context/runtime.rs:68-73` in the vendored registry source (the version pinned in `Cargo.lock`).
- Automated tooling: `grep` counts over the crate (`unsafe {`: 142; `unsafe fn`: 56; `SAFETY:`: 1; `extern "C" fn`: 49; `catch_unwind`: 0; `zeroiz`: 0). `cargo clippy` is already run on this crate in CI (`code-check.yml:108-112`, `--workspace`) and cannot flag undocumented unsafe because of LB-006. Miri and AddressSanitizer: **not run**, see LB-006 for why the existing tests cannot be run under Miri and what would need to change.
- Dynamic testing: built the `cdylib` and reproduced LB-003 from C (see LB-003 and the note at the end of §4). The crate's three existing unit tests were run and pass; one of them logs a caught node-side task panic (`services/network/.../swarm/mod.rs:80` `unwrap` on a failed listen), which is exactly the swallowed-in-a-spawned-task shape LB-002 describes.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Any FFI call made from inside a subscription callback aborts the host process | Denial of Service | Medium | Low | Open |
| LB-002 | No panic barrier at any of the 49 exported functions; every reachable panic aborts the host | Denial of Service | Medium | Medium | Open |
| LB-003 | `start_lb_node` dereferences `config_path` without a null check and is not marked `unsafe` | Data Validation | Low | Low | Open |
| LB-004 | Callback parameters are non-nullable function pointers; a NULL callback is UB when the first event fires | Data Validation | Low | Low | Open |
| LB-005 | `KeyType` crosses by value as a C `enum`; any value other than 0 or 1 is UB inside `generate_key`/`add_key` | Undefined Behavior / Memory safety | Low | Low | Open |
| LB-006 | `undocumented_unsafe_blocks` is allowed crate-wide; 142 `unsafe` blocks carry one `SAFETY:` comment, and no Miri/ASan job exists | Auditing and Logging | Low | — | Open |
| LB-007 | `add_key` with `KeyType::Zk` accepts any hex length and silently reduces it modulo the field, including the empty string | Data Validation | Informational | Low | Open |

### LB-001 · Any FFI call made from inside a subscription callback aborts the host process

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `c-bindings/src/api/subscriptions.rs:86-121`, `:203-209`, `:267-284` (callbacks invoked from `runtime_handler.spawn`); every `*_sync` helper that calls `Handle::block_on`, e.g. `c-bindings/src/api/cryptarchia.rs:103`, `storage.rs:37`, `wallet.rs:77` |
| Status | Open |

**Description**

All three subscription functions spawn a task on the node's tokio runtime and invoke the embedder's callback from that task:

```rust
// subscriptions.rs:203-209
runtime_handler.spawn(async move {
    let mut stream = Box::pin(stream);
    while let Some(event) = stream.next().await {
        emit_json(&ApiProcessedBlockEventOwned::from(event), &mut on_event);
    }
    on_event(std::ptr::null());
});
```

So the callback executes on a tokio worker thread. Every other exported function reaches the node through `node.get_runtime_handle().block_on(...)`. tokio's `Handle::block_on` calls `context::enter_runtime`, which panics when the current thread is already driving that runtime (`tokio-1.52.3/src/runtime/context/runtime.rs:68-73`, message "Cannot start a runtime from within a runtime"). That panic unwinds out of the `extern "C"` function the embedder called from its callback. Since Rust 1.81 an `extern "C"` function that unwinds aborts the process, and the crate has no `catch_unwind` anywhere (LB-002).

The only guidance the header gives is "must be thread-safe" (`subscriptions.rs:147-148`, `:230`, `:302-303`). Nothing says which thread runs the callback, that it is a runtime worker, or that re-entering the API is forbidden. The natural embedder pattern, "on each new block, fetch the block's events / my balance / the chain info", is exactly the forbidden one.

**Exploit scenario**

An embedder registers `subscribe_to_processed_blocks(node, on_block)` and inside `on_block` calls `get_cryptarchia_info(node)` to read the tip. The first block after subscription triggers `Handle::block_on` on a worker thread, tokio panics, the panic hits the `extern "C"` frame of `get_cryptarchia_info`, and the whole host process aborts with "panic in a function that cannot unwind". The same happens for `shutdown_node` called from the NULL-sentinel callback that the documentation encourages the embedder to react to. No network input is required; this is a deterministic crash of any embedder that follows the documented API in the obvious way.

**Recommendation**
- *Short term*: state in every subscription doc comment that the callback runs on a node worker thread and that calling any function of this library from it is not allowed. In each `*_sync` helper, check `tokio::runtime::Handle::try_current()` and return `OperationStatusCode::RuntimeError` ("called from a node callback") instead of calling `block_on`.
- *Long term*: deliver callbacks from a dedicated non-runtime thread (spawn once per subscription, feed it through an `mpsc` channel), so that re-entrancy is legal. Together with LB-002 this removes the abort path entirely.

**References**: tokio `Handle::block_on` docs ("panics if called within an asynchronous execution context"); Rust 1.81 release notes on abort-on-unwind for `extern "C"`.

### LB-002 · No panic barrier at any of the 49 exported functions; every reachable panic aborts the host

| | |
|---|---|
| Severity | Medium |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | All `#[unsafe(no_mangle)] pub … extern "C" fn` items in `c-bindings/src/api/*.rs` and `errors.rs:47-57`; concrete panics at `lifecycle.rs:96`, `config.rs:143`, `errors.rs:41-43`, `node.rs:38-56` |
| Status | Open |

**Description**

`grep -rn catch_unwind c-bindings/` returns nothing. The workspace does not set `panic = "abort"`, so panics unwind, and each of the 49 exported functions is `extern "C"` rather than `extern "C-unwind"`. The compiler therefore inserts an abort guard at each boundary: any panic that reaches it terminates the host process. Because every service call goes through `Handle::block_on` (which re-raises a panic from the future on the calling thread), a panic anywhere in the node code executed on behalf of an FFI call also lands on this boundary. Issue #19 records that `unwrap_used`, `expect_used`, `panic`, `unreachable` and `indexing_slicing` are all allowed workspace-wide, so the surface beneath is large.

Panics that live in the binding crate itself:

- `lifecycle.rs:96` and `config.rs:143`: `Runtime::new().expect("Failed to create Tokio runtime")`. Fails when the process is at its thread or file-descriptor limit, which is precisely when a library should return an error.
- `errors.rs:41-43`: `CString::new(message).expect("Message contained an interior NUL byte.")` runs on the *error* path for every function. The message is built with `format!` from host and file data (`participate` echoes the host string at `config.rs:341`; config, keystore and deployment parse errors echo YAML content). C strings cannot carry a NUL, but YAML can (`"\0"` escapes), so an error while parsing a crafted config file becomes an abort instead of a `ConfigurationError`.
- `node.rs:38-56`: `.as_ref().expect(...)` on the two opaque pointers. Only reachable via a corrupted `LogosBlockchainNode`, but see S-002.
- `subscriptions.rs:104,107,173-175` and `types/block.rs:21-24`: `expect`s inside spawned tasks. These do not abort (tokio catches task panics) but silently end the subscription without the promised NULL sentinel.

**Exploit scenario**

A desktop wallet embeds the node. It calls `start_lb_node` on a config whose `state` block contains a string with a `\0` escape that fails validation; the error message is built, `CString::new` fails, the wallet aborts before it can show the error. Or, with no crafted input at all: the wallet calls `get_balance` while the wallet service is in a state that trips one of the allowed `unwrap`s beneath `WalletApi::get_balance`; the panic propagates through `block_on` and aborts the GUI process. The HTTP server that wraps the same service calls would have returned 500.

**Recommendation**
- *Short term*: wrap the body of every exported function in `std::panic::catch_unwind(AssertUnwindSafe(|| …))` via a single macro or helper, mapping a caught panic to `OperationStatusCode::RuntimeError` with the panic payload as the message. Replace the two `Runtime::new().expect` with `map_err`. Make `OperationStatus::error` replace or strip interior NULs rather than `expect`.
- *Long term*: keep `catch_unwind` as defence in depth but also switch the exported functions to `extern "C-unwind"` only if the embedder contract wants unwinding (it does not for C). Add a test that injects a panic behind one entry point and asserts an error status is returned.

**References**: Rust reference, "FFI and unwinding"; Rust 1.81.0 release notes.

### LB-003 · `start_lb_node` dereferences `config_path` without a null check and is not marked `unsafe`

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Data Validation |
| Target | `c-bindings/src/api/lifecycle.rs:41-50` (`start_lb_node`), `:127-129` (`get_user_config`); also `c-bindings/src/errors.rs:47-57` (`is_ok`, `is_error`) |
| Status | Open |

**Description**

Every other exported function begins with `return_error_if_null_pointer!` on each pointer argument. `start_lb_node` does not:

```rust
// lifecycle.rs:42-50
pub extern "C" fn start_lb_node(config_path: *const c_char, custom_deployment_path: *const c_char)
    -> FfiInitializedLogosBlockchainNodeResult {
    initialize_lb_node(config_path, custom_deployment_path).map_or_else(…)
}
// lifecycle.rs:128
let user_config_path = unsafe { std::ffi::CStr::from_ptr(config_path) }
```

`custom_deployment_path` is null-checked at `:91` and `:150`; `config_path` is not, and `CStr::from_ptr(NULL)` is undefined behaviour (a segfault in practice). The function is also the only exported one declared as a safe `fn`, so its own tests call it without `unsafe` while it dereferences a raw pointer.

`is_ok` and `is_error` take `&self` (`const struct OperationStatus *self` in the header). A NULL from C is UB before any check can run; a `*const` parameter with a null check would match the rest of the crate.

**Exploit scenario**

An embedder reads the config path from its own settings, gets NULL on a missing key, and calls `start_lb_node(NULL, NULL)`. Instead of `NullPointer` in the returned status, the process segfaults. **Confirmed dynamically** at this commit: a three-line C program linked against `liblogos_blockchain.dylib` calling `start_lb_node(NULL, NULL)` terminates with SIGSEGV (exit 139) before returning a status, whereas every other entry point returns `NullPointer` for the same input.

**Recommendation**
- *Short term*: add `return_error_if_null_pointer!(config_path)` at the top of `start_lb_node`, mark it `unsafe extern "C"` like its siblings, and change `is_ok`/`is_error` to take `*const OperationStatus`.
- *Long term*: a unit test per exported function that passes NULL for every pointer argument and asserts `NullPointer`, run under Miri (LB-006).

### LB-004 · Callback parameters are non-nullable function pointers; a NULL callback is UB when the first event fires

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Data Validation |
| Target | `c-bindings/src/callbacks.rs:2` (`CCallback<T> = unsafe extern "C" fn(T)`), `subscriptions.rs:158-166`, `:241-248`, `:314-321` |
| Status | Open |

**Description**

`CCallback<T>` is a bare Rust function-pointer type, which the compiler assumes is never null. The header renders it as `void (*)(const char*)`, which C code can pass as NULL. `subscribe_to_*` copy the value into a `Box<dyn FnMut>` and return `OK`; the first block event then calls through the null pointer on a worker thread. Nothing in the function can detect it because a null `fn` is already UB at the parameter.

**Exploit scenario**

`subscribe_to_lib_blocks(node, NULL)` returns success; the next finalised block crashes the process on a runtime thread, with a stack that does not point at the embedder's mistake.

**Recommendation**
- *Short term*: declare the parameters as `Option<CCallback<T>>` (cbindgen emits the same C type) and `return_error_if_null_pointer!`-style reject `None`.
- *Long term*: cover with the NULL-argument test from LB-003.

### LB-005 · `KeyType` crosses by value as a C `enum`; any value other than 0 or 1 is UB inside `generate_key`/`add_key`

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Undefined Behavior / Memory safety |
| Target | `c-bindings/src/api/keys.rs:14-19` (type), `:74-79` and `:132-138` (by-value parameters), `:21-28` and `:173-187` (`match`es on it) |
| Status | Open |

**Description**

```c
typedef enum KeyType { Ed25519 = 0, Zk = 1 } KeyType;   // logos_blockchain.h:33-36
FfiGenerateKeyResult generate_key(const char*, const char*, enum KeyType key_type, const char*);
```

A C `enum` is an `int`; C permits any value. On the Rust side `KeyType` is a two-variant `#[repr(C)]` enum, and producing a value outside `{0, 1}` is immediate UB; the `match` at `keys.rs:23` and `:174` may be lowered to a jump table indexed by the value. There is no validation because, by the time Rust sees it, the value is already assumed valid. The other two `repr(C)` enums (`State`, `OperationStatusCode`) only travel Rust → C and are fine.

**Exploit scenario**

An embedder writes `add_key(cfg, ks, 2, hex, NULL)` (an off-by-one against a future third variant, or an uninitialised local). Behaviour is undefined; in practice an out-of-bounds jump or a mis-parsed key written to the keystore.

**Recommendation**
- *Short term*: take `key_type: u32` (or `c_int`) and convert with a `TryFrom` that returns `ValidationError`; keep the C `enum` in the header for readability via a cbindgen `[enum] … ` rename or a documented constant list.
- *Long term*: policy that no Rust `enum` is ever a by-value *input* parameter across the boundary.

### LB-006 · `undocumented_unsafe_blocks` is allowed crate-wide; 142 `unsafe` blocks carry one `SAFETY:` comment, and no Miri/ASan job exists

| | |
|---|---|
| Severity | Low |
| Difficulty | — |
| Category | Auditing and Logging |
| Target | `c-bindings/src/lib.rs:1-4`; `.github/workflows/code-check.yml:108-112`, `:166-170` |
| Status | Open |

**Description**

```rust
// lib.rs:1-4
#![allow(
    clippy::undocumented_unsafe_blocks,
    reason = "Well, this is gonna be a shit show of unsafe calls..."
)]
```

The workspace turns on the whole `restriction` group (`Cargo.toml:324`) and does not allow `undocumented_unsafe_blocks`, so CI's `cargo clippy --workspace` would have flagged every block; this attribute switches it off for the largest `unsafe` surface in the repository (issue #19). Counts at the audited commit: 142 `unsafe {` blocks, 56 `unsafe fn`, one `SAFETY:` comment, and that one is in a test (`lifecycle.rs:333`). Most blocks are of four mechanical shapes (`&*node`, `CStr::from_ptr(p)`, `slice::from_raw_parts(p, 32)`, `Box::from_raw(p)`), so the fix is small: four documented helpers replace most of them, and the rest get a one-line justification.

Dynamic checking: CI runs `nextest --workspace --lib …` and clippy; there is no Miri or sanitizer job (`grep -rn 'miri\|sanitizer' .github/` is empty). The crate's three tests (`lifecycle.rs:297-349`, `config.rs:380-475`) all start a tokio runtime, open RocksDB, and write files, which Miri cannot execute. The code Miri *could* check, the by-value `free_*` functions that rebuild boxed slices from host `len` fields, the `validate()` methods that walk host pointer arrays, `parse_public_key`, `GenerateConfigArgs → EmbeddedInitArgs`, has no unit tests at all. I did not run ASan either: the local nightly (1.97) predates the pinned 1.98 toolchain and a sanitizer build of the full workspace was not feasible in this session, so this item is reported as unverified rather than clean.

**Exploit scenario**

Not directly exploitable. The consequence is that the next edit to `free_known_addresses` or `TransferFundsArguments::validate` has no test and no lint to catch a wrong length or a missing null check, and the reviewer has no stated invariant to check the change against.

**Recommendation**
- *Short term*: remove the `allow`, add `// SAFETY:` to each block (or replace with the four helpers), and add pure unit tests for every `free_*`, `validate`, and parse helper with `len == 0`, `len == N`, and NULL inputs.
- *Long term*: a CI job `cargo +nightly miri test -p logos-blockchain-c -- <pure tests>` and a Linux `RUSTFLAGS=-Zsanitizer=address cargo test -p logos-blockchain-c` job for the integration tests.

### LB-007 · `add_key` with `KeyType::Zk` accepts any hex length and silently reduces it modulo the field, including the empty string

| | |
|---|---|
| Severity | Informational |
| Difficulty | Low |
| Category | Data Validation |
| Target | `c-bindings/src/api/keys.rs:183-186` → `nodes/node/binary/src/config/mod.rs:564-570` (`parse_hex_zk_key`) → `kms/keys/src/keys/zk/private.rs:83-87` |
| Status | Open |

**Description**

```rust
// config/mod.rs:564-570
pub fn parse_hex_zk_key(s: &str) -> Result<UnsecuredZkKey, String> {
    let bytes = hex::decode(s).map_err(…)?;
    let big_uint = BigUint::from_bytes_le(&bytes);
    Ok(UnsecuredZkKey::from(big_uint))      // Fr::from(BigUint): reduce mod r
}
```

The Ed25519 branch (`:572-577`) rejects anything but 32 bytes. The ZK branch accepts `""` (key = 0), `"01"` (key = 1), or 64 bytes (reduced), and `add_key` returns `OK` and persists it. The CLI `--zk` flag shares the parser, so this is not FFI-specific, but the FFI is the path where the hex is most likely to be produced programmatically from a truncated or empty buffer. Listed as informational because the key is the host's own; the damage is a weak or wrong key silently accepted rather than an attack.

**Recommendation**
- *Short term*: require exactly 32 bytes and reject non-canonical values (compare `Fr::from_bytes` result back to the input, as `parse_public_key` in `wallet.rs:816-824` already does for public keys).

**Note on dynamic reproduction.** The `cdylib` was built at this commit and LB-003 reproduced from C (SIGSEGV on a null `config_path`), confirming that path directly. LB-001 and LB-002 were established by code reading and by the pinned tokio panic path; a full end-to-end reproduction of LB-001 additionally needs a running node with genesis, which was not built in this session. The `test_basic_lifecycle` unit test run does exercise the swallowed-task-panic behaviour LB-002 describes (a `libp2p` listen `unwrap` fails and is logged, not returned).

## 5. Suggestions (non-security)

### S-001 · `usize` in ten ABI struct fields

`ClaimableVouchers.len`, `KnownAddresses.len`, `WalletNotes.len`, `PoWClaimableRewards.{claimable_tickets,len}`, `TransferFundsArguments.funding_public_keys_len`, `ChannelDepositWithNotesArguments.{input_note_ids_len,metadata_len,funding_public_keys_len}`, `ChannelDepositArguments.metadata_len` are `usize`, rendered `uintptr_t` (`logos_blockchain.h:208-478`). This is not an ABI mismatch on any Rust target (`usize == uintptr_t == size_t`) and 32-bit layout of the `u64` fields that follow them matches C, but C callers expect `size_t` for lengths. cbindgen's `usize_is_size_t = true` makes the header say so. Checked and fine for 32-bit: `get_blocks` guards `u64 → usize` with `try_from` (`storage.rs:311-322`); `initial_peers_count` is `*const u32`; no `u128`; `bool` goes through `<stdbool.h>`.

### S-002 · `LogosBlockchainNode` is not opaque

`node.rs:15-24` is `#[repr(C)]` with two `void*` fields, so the header exposes the layout (`logos_blockchain.h:66-69`) and a C embedder can copy the struct by value and later pass both copies to `shutdown_node`, double-freeing. Drop the `repr(C)` (or make the fields private and let cbindgen emit a forward declaration) so C only ever sees `struct LogosBlockchainNode*`. Related: `Drop` (`node.rs:89-105`) logs when a pointer is null and then calls `Box::from_raw` on it anyway.

### S-003 · `Block(CString)` is `#[repr(C)]` but not FFI-safe and never crosses

`types/block.rs:7-8`. The attribute is misleading; the type is `pub(crate)` and absent from the header. Remove it.

### S-004 · Inconsistent `free_*` contracts

`free_wallet_notes` returns `OK` on a null pointer (`wallet.rs:601-603`); `free_known_addresses`, `free_claimable_vouchers`, `free_pow_claimable_rewards` return `NullPointer`. `free_known_addresses` returns early from inside its loop (`wallet.rs:244-247`), leaking the remaining 32-byte boxes if one entry is null. Pick one contract (null is a no-op is the C convention) and free everything before reporting.

### S-005 · Library code writes to the host's stdout/stderr

`run_add_key` and `run_remove_key` `println!` on success (`cli/keys.rs:330,369`), reached from `add_key`/`remove_key`. `logging.rs` writes errors with `eprintln!` instead of `tracing`. A library embedded in a GUI or a service should not print; route through `tracing` and let the embedder install a subscriber, or return the text in the status.

### S-006 · Secret-handling copies (checked, mostly fine)

No exported function returns secret material: outputs are public keys, note ids, hashes, JSON of blocks, transactions, channel state, blend info, and one ZK signature (`wallet_fund_tx`). Inbound secrets arrive only via `add_key`. Along that path `Ed25519Key`, `ZkKey`, `UnsecuredEd25519Key`, `UnsecuredZkKey` are `ZeroizeOnDrop` (`kms/keys/src/keys/{mod,ed25519/mod,ed25519/private,zk/mod,zk/private}.rs`), `parse_hex_ed25519_key` passes `as_mut_slice()` to `libp2p_identity`'s `try_from_bytes`, which zeroes its input, and the error strings echo only the offending hex character, never the key. Two gaps: the `[u8; 32]` stack copy at `keys.rs:177-181` is not zeroized (wrap in `Zeroizing`), and `AddKeyArgs` derives `Debug` while holding `UnsecuredEd25519Key`, which also derives `Debug` (`private.rs:16`), so any future `{args:?}` prints the key. The FFI path never formats it today; the CLI's `run_generate_key` does print `Key: {key:?}` by design when the user declines to persist (`cli/keys.rs:297`).

### S-007 · Subscription contract on shutdown is unverified

Docs promise the NULL sentinel "exactly once" when the stream ends, including on node shutdown (`subscriptions.rs:227-229`). `shutdown_node` awaits Overwatch shutdown, then drops the runtime (`node.rs:75-85`, `:103-104`), which cancels the spawned task at its next await point. Whether the task is polled after the stream ends and before the runtime drops is a race I did not test; if it loses, the sentinel never fires. Worth a test.

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
