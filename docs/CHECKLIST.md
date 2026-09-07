# Audit Checklist — Logos Blockchain node

What to look for when auditing the logos-blockchain repository (https://github.com/logos-blockchain/logos-blockchain). Each item is a question the reviewer answers with a finding, a "verified OK" note, or "not in scope". Items marked **⚑ repo** are grounded in something observed in the logos-blockchain codebase at the time of writing (2026-09-07) and should be checked first.

All paths mentioned below (`c-bindings/`, `zk/`, `consensus/`, `services/`, `Cargo.toml`, `flake.nix`, etc.) and the commands under "Quick grep starters" are relative to a checkout of the logos-blockchain repository (https://github.com/logos-blockchain/logos-blockchain), not to the message-board repository this file lives in.

Sources: Trail of Bits public reports and code-maturity framework, Least Authority (Zebra, Namada, Core DAO), Oak Security (Nym mixnet), zkSecurity / Zellic ZK-audit methodology, Sherlock Rust auditing guide, RustSec advisories (`RUSTSEC-2022-0084` libp2p resource management, gossipsub subscription flood `CVE-2026-46679`), Groth16 malleability disclosures (snarkjs #383, Sui blog).

---

## 0. Repo-level facts to carry into every section  ⚑ repo

- [ ] `[profile.release]` does **not** set `overflow-checks = true` → integer arithmetic wraps silently in production builds. Every `+ - * <<` on amounts, slots, epochs, lengths, indexes is a candidate.
- [ ] Clippy `restriction` group is on, but these are explicitly **allowed** (`Cargo.toml` lints): `arithmetic_side_effects`, `as_conversions`, `cast_possible_truncation`, `cast_possible_wrap`, `indexing_slicing`, `unwrap_used`, `expect_used`, `unwrap_in_result`, `panic`, `todo`, `unimplemented`, `unreachable`, `map_err_ignore`, `missing_panics_doc`. The lints that would normally surface panic/overflow risks are therefore silent — grep for these patterns manually.
- [ ] `c-bindings/` contains ~300 `unsafe` occurrences across ~5.3k lines. This is the largest memory-safety surface.
- [ ] Proving is done through `rust-rapidsnark` (C++ prover/verifier behind FFI, pinned git rev, must stay in sync with `flake.nix`). Circuits and keys live outside the logos-blockchain repository (`circuits.env` points at local `bin/` paths).
- [ ] No `fuzz/` directory or `cargo-fuzz` targets exist; `proptest`/`quickcheck` are dependencies — check where they're actually used.
- [ ] No `deny.toml` / `cargo audit` configuration found; `multiple_crate_versions` is allowed (duplicate crate versions tolerated).
- [ ] Git dependencies pinned by rev: `jellyfish` (Poseidon2), `rust-rapidsnark`, `logos-blockchain-testing`. Verify each rev is what's reviewed.
- [ ] `strip = true` + `lto = "fat"` in release: crash reports from operators will lack symbols. Decide if `release-profiling` is what gets shipped.
- [ ] CI has `nightly-cluster-fork-detector.yml` and `genesis-ceremony.yml` — read both; they encode operational assumptions worth auditing.
- [ ] Config parsing uses `serde_ignored` — confirm unknown/misspelled config keys are rejected or at least logged loudly.

---

## 1. Consensus — Cryptarchia (`consensus/`, `ledger/src/cryptarchia`, `services/chain/*`, `services/time`)

### Slot / epoch arithmetic
- [ ] Slot → epoch conversion, epoch length, and epoch-boundary handling: off-by-one at boundary, wrap on `u64`, behaviour at slot 0 / genesis.
- [ ] Which epoch's **stake distribution** and **nonce** feed the current epoch's lottery? Is the snapshot taken far enough back (≥ stability window) that an adversary can't influence it at the last moment?
- [ ] Security parameters (`k`, `s`/truncation window, stability/settlement depth) are per-network config: mainnet vs testnet values, validation that they're sane, and that all nodes agree (part of chain-id / genesis hash?).

### Leader election / Proof of Leadership (PoL)
- [ ] Threshold derivation from stake (`φ(α)`): no floating point in the consensus path; fixed-point precision; rounding direction; `u128` where stake × constant can exceed `u64`.
- [ ] Zero stake, total-stake = 0, single-staker, and stake > total-stake (inconsistent snapshot) edge cases.
- [ ] Stake **relativisation** / fork-choice interplay (see Logos blog on preventing wealth concentration): is the implemented rule the specified one?
- [ ] Can a leader produce **multiple blocks for one slot** (equivocation)? What does fork choice do with them? Is there any slashing / evidence?
- [ ] Leader secret handling in `chain-leader`: never logged, zeroized, not exposed via API/FFI.
- [ ] Linkability: does a PoL proof or block header let an observer link a leader's blocks across slots/epochs (privacy of stakers)?

### Randomness / grinding
- [ ] Epoch nonce derivation: inputs, hash domain separation, how many bits of adversarial influence per block (withholding, ordering, choosing among own eligible slots).
- [ ] Is the nonce fixed before the adversary can see the resulting eligibility? Compare against Ouroboros Praos/Genesis and CIP-0161-style grinding analysis.

### Fork choice
- [ ] Rule implemented (longest chain with `k` truncation / density) matches spec; tie-breaking is deterministic and identical on all nodes.
- [ ] Reorg depth bound enforced; what happens on a fork deeper than `k` (halt? follow? alert?).
- [ ] Blocks with slot in the **future** (beyond clock tolerance) and far **past** are rejected/held correctly; clock-skew tolerance is configured and bounded.
- [ ] Orphan / pending-block pools are bounded in count and bytes and evict deterministically.
- [ ] Per-fork ledger states: memory growth with many forks, pruning of abandoned forks, identical state after applying the same blocks in a different arrival order.

### Validation ordering (DoS)
- [ ] Cheap checks (parent known, slot sane, size limits, signature) happen **before** expensive ones (Groth16 verification, ledger application).
- [ ] Duplicate block / duplicate header detection before any work; cache keyed by hash, bounded.
- [ ] Blocks are not **relayed** before they are validated.

### Sync / bootstrap (`cryptarchia-sync`, `chain-network`)
- [ ] Bootstrapping from untrusted peers: long-range / "fake long chain" attacks — is there a checkpoint, genesis trust, or density-based comparison when far behind?
- [ ] Sync requests and responses bounded (batch size, bytes, timeouts); a slow or malicious peer can't stall sync indefinitely; sync spreads across peers.
- [ ] Blocks received during sync are validated before persisting; partial sync state survives restart or is safely discarded.
- [ ] Backfill / historical download cannot be turned into an amplification vector against the serving node.

### Time (`services/time`)
- [ ] Time source (system clock, NTP?) and what happens on backwards jumps; slot timer uses monotonic time; no consumption of peer-provided time for validation.
- [ ] Startup before genesis time, and behaviour when the node has been offline for many epochs.

### Determinism (consensus-critical code)
- [ ] No `HashMap`/`HashSet` iteration order reaches block content, ledger state, or fork choice (use `BTreeMap` or sort).
- [ ] No `f32`/`f64`; no `usize` in serialised or hashed data (32-bit FFI hosts); `sort` uses total orders.
- [ ] No `SystemTime::now()` inside validation functions.

### Finality semantics
- [ ] What the API/wallet/FFI report as "confirmed"/"final" matches what consensus actually guarantees (depth `k`? probabilistic?).

---

## 2. Ledger & state transition (`ledger/`, `core/src/mantle`, `merkle/*`, `mmr/`, `codec/`)

### Notes / UTXO model
- [ ] Double spend: same nullifier twice in a tx, in a block, across a reorg; nullifier set is per-fork and persisted atomically with the block.
- [ ] Commitment tree: append-only, root recomputed correctly, Merkle proofs verified against an **accepted** root (which roots are accepted — only latest, or a window?).
- [ ] Value conservation: Σinputs = Σoutputs + fees with checked arithmetic; zero-value and dust notes; max supply.
- [ ] Note commitment / nullifier derivation matches the circuits' derivation bit-for-bit (domain tags, field encoding).

### Mantle operations / transactions
- [ ] Every op type: full validation, authorisation (who may execute), ordering effects inside a tx and inside a block.
- [ ] Transaction hash/ID covers **all** fields (no malleable bytes, e.g. proof bytes or signatures excluded → ID-based dedup bypass).
- [ ] Replay: across forks, after reorg, across networks — chain-id is bound into signatures/proofs (**⚑ repo**: branch `expose-chain-id` recently made chain-id non-static; check every signing/proof domain includes it and that a malformed/mismatched chain-id fails closed).
- [ ] Fees: who pays, when deducted, underflow when fee > balance, fee-less ops as DoS vector.
- [ ] Tx and block size limits enforced before decode where possible; max ops per tx; max txs per block.

### SDP — service declaration protocol (`services/sdp`, `core/src/sdp`)
- [ ] Declaration lifecycle (declare → active → withdraw): state machine has no unreachable/absorbing states; timing of activation vs. epoch boundaries.
- [ ] Locked stake accounting; withdrawal timing vs. reward/penalty windows; can a provider withdraw and still be selected?
- [ ] Only the declarer can update/withdraw; signature/proof binds declaration ID and chain.
- [ ] Rewards/penalties: double-claim, claim for someone else, inflation via rounding.
- [ ] SDP mempool (`services/sdp/src/mempool.rs`): bounds, eviction, conflicting declarations.

### Reorg handling
- [ ] Ledger, mempool, wallet, and blend membership all roll back consistently; nothing keeps effects from an orphaned block.

### Codec (`codec/`, `core/src/codec`)
- [ ] Canonical encoding: `decode(encode(x)) == x` **and** `encode(decode(b)) == b` for all accepted `b`; trailing bytes rejected; length prefixes bounded **before** allocation; nested depth bounded.
- [ ] Versioning: how unknown versions/variants are treated; upgrade path.
- [ ] Any `bincode` use on untrusted input has a size limit (`bincode 1.x` default is unlimited).

### Merkle / MMR / UTXO tree (`merkle/*`, `mmr/`)
- [ ] Leaf vs internal node domain separation (second-preimage / CVE-2012-2459-style); empty tree root; proof with out-of-range index; proof length vs tree depth; verification uses caller-supplied root only when that root is trusted.
- [ ] `blake2btree` / `dynamic-merkle`: insertion order determinism, hash of `usize` positions.

---

## 3. Zero-knowledge (`zk/`, `blend/proofs`, `blend/provers`)

### Circuit-side (separate repo — confirm whether in scope)
- [ ] Under-constrained signals: every public input and every intermediate signal is bound; witness-only computations have a matching constraint.
- [ ] Range checks on all non-field values (bytes, amounts, indices, bits); Merkle path index bits constrained boolean; comparisons don't alias past the field modulus.
- [ ] Nullifier binds secret key **and** note; commitment binds all note fields; no way to open a commitment two ways.
- [ ] Poseidon2 parameters (t, rounds, MDS, constants) equal the Rust `zk/poseidon2` / jellyfish parameters; domain separation between hash uses.
- [ ] Over-constraint (completeness): honest inputs that can't be proven (boundary values, max amounts).

### Integration (the logos-blockchain repository)
- [ ] Public-input vector construction (`*_inputs.rs`) matches the circuit's ordering and encoding exactly; a swapped pair of inputs would be a soundness break.
- [ ] Field elements parsed from network/config: reject values ≥ p (non-canonical) rather than reducing; endianness consistent with rapidsnark/circom.
- [ ] Verification keys embedded in `zk/proofs/*/verification_key`: provenance documented, hash pinned and checked at startup, matches the trusted-setup output; who can swap them (`circuits.env` / env vars / file paths — file permissions, TOCTOU).
- [ ] Proof deserialisation: curve-point validation and subgroup checks (arkworks `deserialize_compressed` vs `_unchecked`); rapidsnark verifier's own checks on `A`, `B`, `C` < q.
- [ ] **Groth16 malleability**: the same statement has infinitely many valid proofs. Nothing uses proof bytes/hash as a unique ID, nullifier, or dedup key.
- [ ] Proof **binding**: PoL → slot, epoch nonce, parent/chain-id, leader commitment; PoQ → session/epoch, quota index, blend identity; zksign → message hash + chain-id; PoC → claim target. Replay of a valid proof in another context must fail.
- [ ] Verification happens **before** expensive or state-changing work and is not skippable via a config flag in release builds; a "verified" cache is keyed by (proof, public inputs) not proof alone.
- [ ] Prover FFI (`zk/circuits/prover/src/rapidsnark.rs`): buffer lengths passed correctly, error codes checked, no panic across the boundary, thread safety of the C++ prover, witness/secret buffers zeroized after use, memory freed on every path.
- [ ] Witness generation timing: proof generation must fit in a slot; failure path when it doesn't (leader skips slot silently? logs?).
- [ ] Proving/verification key binaries downloaded by scripts: integrity (sha256 pinned), source, and behaviour when files are missing/corrupt.
- [ ] PoQ / blend quota: quota can't be reused across sessions; selection proofs can't be pre-computed to bias selection; total quota accounting overflow.

---

## 4. Cryptography & key management (`core/src/crypto.rs`, `blend/crypto`, `kms/`, `services/key-management-system`)

- [ ] RNG: `OsRng`/`getrandom` for keys and nonces; any `rand_chacha` seeded RNG is test-only or seeded from OS entropy; `fixtures.rs` modules never reachable in release paths.
- [ ] Ed25519: `verify_strict` or explicit handling of small-order/non-canonical points if signatures affect consensus; batch verification failure isolation.
- [ ] X25519 / cipher (`blend/crypto/src/cipher.rs`): AEAD with unique nonces (how derived? counter vs random), key commitment where multiple keys could decrypt, no reuse across directions.
- [ ] Domain separation on every hash and signature (tags include purpose + chain-id + version).
- [ ] Constant-time comparison (`subtle`) for MACs, tags, keys; no secret-dependent indexing/branching; no `PartialEq` derive on secret types.
- [ ] Secret types: `Zeroize`/`ZeroizeOnDrop`, redacted `Debug`, not `Clone` unless necessary, not `Serialize` by accident.
- [ ] KMS: key storage at rest (`keystore.yaml`, `kms.yaml`) — encryption, file permissions, plaintext in config; operator model (`kms/operators`) — which operations each key type permits, and that signing requests can't be coerced into a different domain.
- [ ] Key export/import via HTTP API, FFI, or logs; key derivation paths documented and versioned.
- [ ] Poseidon2 Rust implementation: constants and MDS match reference; sponge padding; inputs reduced mod p exactly once.

---

## 5. Blend — mixnet & privacy (`blend/*`, `services/blend`)

### Message format & processing
- [ ] Fixed message size and padding: no length or structure leaks between layers; cover and real messages indistinguishable on the wire and in timing.
- [ ] Decapsulation errors: no oracle (different error/timing for wrong key vs malformed vs replayed).
- [ ] Replay protection for messages/encapsulations; bounded replay cache.
- [ ] Recent commit clamps PoW claims to payload size (**⚑ repo**, `d37bde7a5`) — verify all other payload producers respect the same bound and that over-size input is rejected, not truncated silently.

### Membership & selection (`blend/membership`, `blend/scheduling`)
- [ ] Membership set derives from on-chain SDP state deterministically; epoch transition handling; a node with stale membership can't be tricked into routing through attacker-only paths.
- [ ] Node selection randomness: source, bias, ability of an adversary to predict or steer path selection (Sybil / family-style coercion as found in the Nym audit).
- [ ] Scheduling delays: distribution, deterministic seeds that would let an observer correlate; cover-traffic rate and what happens when a node is idle.
- [ ] Graceful shutdown/restart doesn't reveal in-flight messages or drop them observably.

### Rewards (`blend/message/src/reward`)
- [ ] Reward claims can't be forged, inflated, or double-claimed; binding to actual work; who verifies.

### Metadata & side channels
- [ ] Logs/tracing/metrics never include message contents, session IDs, per-message timing, or peer↔identity mappings.
- [ ] Identity reuse: libp2p peer ID, blend identity, SDP declaration key, leader key, wallet — any linkable pair is a deanonymisation path.
- [ ] Error paths and rate limiting don't distinguish sender behaviour to an observer.

### Network layer (`blend/network`, `services/blend/src/core/backends`)
- [ ] Per-peer connection/stream/message limits; malformed message handling; memory bounds on queues; flood resistance.
- [ ] One `unsafe` site in `services/blend/src/core/backends/libp2p/tokio_provider.rs` (**⚑ repo**) — justify and document.

---

## 6. P2P networking (`libp2p/`, `services/network`, `services/chain/chain-network`, `services/chain/broadcast-service`)

- [ ] Resource limits: max connections (in/out), per-peer streams, max message size per protocol, decode limits on every `Vec`/`String` length, gossipsub `max_transmit_size`, mesh params, peer scoring enabled and tuned.
- [ ] Subscription/topic flood, IHAVE/IWANT amplification, duplicate-message cache size (libp2p advisories RUSTSEC-2022-0084, gossipsub CVE-2026-46679 class).
- [ ] Kademlia (if used): routing-table poisoning, eclipse via many peer IDs from one IP, bootstrap-node trust.
- [ ] Custom NAT state machine (`libp2p/src/behaviour/nat/state_machine`): every transition on peer-controlled input; no panics; timers bounded.
- [ ] Transport security: only authenticated encrypted transports enabled; no plaintext fallback; QUIC/TCP/WebSocket configs; peer identity pinned for bootstrap nodes?
- [ ] Message validation before forwarding for blocks, txs, and blend messages; invalid-message penalties.
- [ ] Backpressure: bounded channels between network service and consumers; behaviour when a consumer is slow (drop vs. unbounded growth); one task per message patterns.
- [ ] `libp2p-stream` is an `alpha` dependency (**⚑ repo**) — review its usage sites for correctness under stream resets/timeouts.
- [ ] Peer address handling: private/loopback addresses from peers, address amplification, dial-back attacks.

---

## 7. Mempool, leader & PoW (`services/tx-service`, `services/chain/chain-leader`, `services/pow`, `services/sdp`)

- [ ] Admission: size, per-sender count, fee floor, signature/proof check **before** insertion, dedup by full tx hash, conflicting nullifiers rejected.
- [ ] Eviction policy is deterministic and not steerable by an attacker to drop honest txs; stale tx expiry; re-injection after reorg.
- [ ] Mempool exposure: HTTP endpoint and p2p ingestion have the same policy; no privileged path around it.
- [ ] Block building: only currently-valid txs, deterministic ordering (or documented MEV/ordering policy), block size/op limits respected, fee accounting correct, no panics on pathological mempool contents (a panic here kills the leader's slot).
- [ ] PoW service: difficulty calculation/adjustment overflow, ticket validation cost ≪ generation cost, ticket replay, target comparison endianness.
- [ ] Proof generation for leadership runs off the runtime worker threads (`spawn_blocking`) and has a deadline.

---

## 8. Storage (`services/storage`, `logos_sql`, RocksDB, SQLite)

- [ ] Multi-key updates use write batches / transactions; crash between writes can't leave block-without-state or state-without-block.
- [ ] Startup recovery: detects and handles a partially applied block, corrupt DB, DB from a different network (chain-id/genesis hash stored and checked — **⚑ repo** relevant to current branch).
- [ ] Key/prefix scheme: no collisions between column families or key types; keys use fixed-width big-endian ints if range-scanned.
- [ ] Unbounded growth: what is never pruned (forks, old proofs, blend caches, tx history); disk-full behaviour.
- [ ] `logos_sql`/`rusqlite`: parameterised queries only; schema migrations versioned; WAL/fsync settings vs durability claims.
- [ ] State-dir permissions; secrets (wallet, keys) encrypted at rest or clearly documented as not.
- [ ] Blocking DB calls not on async worker threads.

---

## 9. HTTP API, wallet & FFI (`services/api`, `nodes/*`, `wallet*`, `c-bindings`, `zone-sdk`)

### HTTP API
- [ ] Default bind address (localhost vs 0.0.0.0), auth on privileged endpoints (submit tx, key/leader/config/peer management), CORS policy, TLS.
- [ ] Request body/size limits, JSON depth, rate limiting, timeouts, slow-loris.
- [ ] Information disclosure: peer lists, wallet balances, key material, internal errors/stack traces, config dumps; chain-id endpoint (**⚑ repo**) returns nothing sensitive alongside.
- [ ] Any file path or query parameter reaching the filesystem or DB.

### Wallet (`wallet`, `wallet-http-client`, `services/wallet`)
- [ ] Note selection / change outputs don't create linkable patterns; fee estimation can't be manipulated by the node; balance vs. finality.
- [ ] HTTP client validates TLS, pins nothing insecure, and doesn't trust the node for validity of proofs it could check.
- [ ] Wallet key storage and derivation.

### FFI (`c-bindings`, ~300 unsafe sites — **⚑ repo**)
- [ ] Every pointer argument: null check, length check, alignment, lifetime documented; every `*const c_char` validated UTF-8 before use.
- [ ] Ownership: allocation/free pairs (`api/memory.rs`), no double free, no free of non-owned memory, returned buffers' lifetimes documented.
- [ ] Callbacks (`callbacks.rs`, `subscriptions.rs`): invoked from tokio threads — thread-safety contract, reentrancy, callback lifetime vs. subscription lifetime (use-after-free when host frees context).
- [ ] Panics never cross the boundary (`catch_unwind` at every extern "C" fn, or `panic = "abort"`); error codes complete and stable.
- [ ] `#[repr(C)]` on every struct crossing; enum discriminant sizes; `usize` avoided in ABI; 32-bit targets.
- [ ] Key/leader/wallet APIs over FFI (`api/keys.rs`, `api/leader.rs`, `api/wallet.rs`): secret export surface and zeroization of buffers handed to the host.
- [ ] `SAFETY:` comment on every `unsafe` block (`undocumented_unsafe_blocks` lint), Miri/ASan run on the crate's tests.

---

## 10. Rust-specific review items (apply everywhere)

### Arithmetic
- [ ] `+ - * / % << >>` on `u64` amounts, stakes, slots, epochs, timestamps, lengths → `checked_*`/`saturating_*` or proven-bounded; `u128` for products. (**⚑ repo**: release overflow-checks off; `arithmetic_side_effects` allowed.)
- [ ] `as` casts: `usize→u32`, `u64→usize`, `i64→u64` truncation/wrap on attacker-influenced values (`as_conversions`, `cast_*` allowed — **⚑ repo**).
- [ ] Division by zero (total stake, epoch length, difficulty), `Duration`/`Instant` subtraction panics, `abs_diff` vs. `-`.

### Panics = remote DoS
- [ ] `unwrap`/`expect`/index `[]`/`split_at`/`copy_from_slice`/`from_*_bytes` on data derived from network, config, DB, or FFI (`unwrap_used`, `indexing_slicing`, `panic` allowed — **⚑ repo**).
- [ ] `todo!`/`unimplemented!`/`unreachable!` reachable in release (allowed — **⚑ repo**).
- [ ] `RefCell` double borrow, `Mutex` poisoning propagating, `assert!` on external input.
- [ ] A panic inside a `tokio::spawn`ed task doesn't crash the node — it silently kills that service. Every spawned task's `JoinHandle` is awaited/monitored or the service supervises restarts.

### `unsafe`
- [ ] Inventory with `cargo geiger`; each block has a `SAFETY:` justification; Miri on crates that have any; no `unsafe` to bypass borrowck for convenience; `transmute`, `from_raw_parts`, `set_len`, `get_unchecked`, `static mut` audited individually.

### Deserialisation of untrusted input
- [ ] `serde` derives on network/config types: `#[serde(default)]` making missing fields mean something privileged; `untagged` enums with ambiguous variants; `deny_unknown_fields` on config; duplicate keys; numbers exceeding target width; NaN floats.
- [ ] Length-prefixed collections allocate only after bounds check; `serde-big-array`/`serde_arrays` sizes fixed; `bincode` limits; YAML/JSON nesting depth; string sizes.
- [ ] Custom `Deserialize`/`Decode` impls: total-consumption, canonical form, error on trailing bytes.

### Async / tokio
- [ ] `std::sync::Mutex`/`RwLock` guard held across `.await`; lock ordering across services (deadlock); `RwLock` writer starvation.
- [ ] Unbounded `mpsc::unbounded_channel` / `broadcast` with lagging receivers; channel capacities documented; behaviour on `send` failure.
- [ ] CPU-heavy (proof gen/verify, hashing large batches) and blocking (RocksDB, file IO, rapidsnark) work off the async runtime via `spawn_blocking`/dedicated threads.
- [ ] `select!` cancellation safety — a cancelled branch loses a partially read message or a popped queue item; `tokio::time::timeout` on every network/FFI wait.
- [ ] Service shutdown (`services/system-sig`, overwatch lifecycle): tasks cancelled, storage flushed, no double-init on restart.

### Determinism (for anything hashed, signed, agreed on)
- [ ] `HashMap`/`HashSet` iteration; `f32/f64`; `usize`/`isize` in serialised data; `Instant`/`SystemTime` in logic; `rand::thread_rng` in consensus paths; sort with non-total `PartialOrd`; `Debug` formatting used as a canonical string.

### Error handling
- [ ] `let _ = fallible()`; `.ok()` discarding; `map_err(|_| ...)` (`map_err_ignore` allowed — **⚑ repo**) hiding root causes; errors converted into defaults (`unwrap_or_default`) on validation paths — fail closed, not open.
- [ ] Error types don't carry secrets into logs; `Display` impls of errors for network responses don't disclose internals.

### Type & trait hygiene
- [ ] `Debug`/`Display`/`Serialize` derived on secret-bearing structs (log leakage); `Clone`/`Copy` on secrets; `PartialEq` on secrets (non-constant-time); `Hash` on types with floats or pointers; `Default` producing valid-looking-but-wrong values (zero key, epoch 0).
- [ ] `Ord` derives used by consensus/ledger where field order in the struct silently defines protocol ordering.
- [ ] `From`/`Into` conversions that lose information or truncate; `TryFrom` where `From` is used.

### Features / cfg
- [ ] `fixtures`/test helper modules gated with `#[cfg(test)]` or a feature that is **off** in release; feature unification can enable them via another crate.
- [ ] `default-features = false` everywhere (**⚑ repo** style) → confirm required features (e.g. `rustls`, `tokio` runtime features) are enabled explicitly rather than by accident through another crate.

### Logging & tracing
- [ ] `tracing::*!("{:?}", self)` on structs with secrets/notes/peers; log injection via peer-supplied strings (newlines, ANSI); per-peer log volume as disk-fill DoS; OTLP/exporter endpoints and whether telemetry leaves the host unencrypted.

### Dependencies & supply chain
- [ ] `cargo audit`, `cargo deny` (advisories, licences, sources, bans), `cargo vet` or review of every git dependency at the pinned rev; yanked crates; `alpha`/`rc` crates (`libp2p-stream`).
- [ ] Duplicate crate versions with security relevance (`rand`/`rand_core` 0.6 vs newer used by dalek/arkworks; `getrandom` backends).
- [ ] `build.rs` scripts and proc-macros in the tree (`codec/macros`, `kms/macros`, `tracing/targets/macros`): no network access, deterministic.
- [ ] Reproducible build via `flake.nix` actually matches the `cargo build` artefact shipped in Docker (`publish-node-image.yml`); base image; runs as non-root; minimal exposed ports.
- [ ] Downloaded circuit artefacts (`zk/proofs/*/bin`, `resources/`) verified by hash; download script uses HTTPS and fails closed.
- [ ] `rust-toolchain.toml` pinned; nightly usage confined to fmt.

---

## 11. Configuration, genesis & deployment (`tools/config`, `nodes/`, `deployment/`, `testnet/`, `.github/workflows`)

- [ ] Config precedence (`deployment_config.yaml` vs `user_config.yaml` vs env vars vs CLI): documented, no silent override of security-relevant values (bind address, security params, chain-id, peers).
- [ ] Secure defaults: API bound to loopback, telemetry off, debug endpoints off, sane resource limits.
- [ ] Secrets in config files (`keystore.yaml`, `kms.yaml`, `provider.yaml`, `stakeholder.yaml`) — file permission checks, no secrets echoed in logs at startup, no secrets in `deployment_config`.
- [ ] Chain-id / genesis-hash consistency: config vs genesis files vs stored DB vs peers; mismatch fails at startup with a clear error; wrong-network peers are disconnected (**⚑ repo**, current branch).
- [ ] Genesis inputs (`genesis/declarations.yaml`, `notes.yaml`, `inscription.yaml`): schema validation, total supply, duplicate notes/declarations, and the `genesis-ceremony.yml` workflow's trust assumptions (who can run it, how outputs are verified).
- [ ] Testnet vs mainnet parameter sets can't be mixed; `standalone` mode can't be started against mainnet peers.
- [ ] `faucet` (`deployment/faucet`): rate limiting, key isolation — never deployable against mainnet.
- [ ] Docker: image provenance, non-root, no build secrets baked in, healthchecks.

---

## 12. Testing & verification (for the maturity rating)

- [ ] Negative tests exist for every consensus rule (invalid PoL, wrong parent, future slot, double spend, bad proof binding, bad chain-id).
- [ ] Property tests (`proptest`/`quickcheck`) cover codec round-trips, Merkle/MMR proofs, arithmetic helpers; where they exist vs. where they're only a dependency (**⚑ repo**).
- [ ] Fuzz targets (`cargo-fuzz`) for: block/header/tx decode, blend message decode, sync protocol messages, HTTP JSON bodies, config parsing, FFI entry points (**⚑ repo**: none present).
- [ ] Miri on `c-bindings`, `codec`, any `unsafe` crate; ASan/UBSan for the rapidsnark link.
- [ ] Concurrency: `loom` or targeted tests for service channels/shutdown; e2e fork-detector (`nightly-cluster-fork-detector.yml`) results are reviewed and alert someone.
- [ ] Differential tests against the executable spec / reference implementation where one exists.
- [ ] CI runs clippy with workspace lints as errors; `cargo audit`/`cargo deny` in CI; test coverage number for `ledger`, `consensus`, `zk/proofs`.

---

## Quick grep starters

```sh
# panic surface
rg -n "\.unwrap\(\)|\.expect\(|panic!|unreachable!|todo!|unimplemented!" --type rust -g '!**/tests/**' -g '!**/fixtures.rs'
# raw arithmetic on 64-bit ints (needs eyeballing)
rg -n "as u32|as usize|as u64|as i64" --type rust
# unsafe inventory
rg -n "unsafe " --type rust -c | sort -t: -k2 -nr
# hashmap iteration in consensus/ledger
rg -n "HashMap|HashSet" consensus ledger core/src/mantle
# locks across await (manual): find guards then check for .await in scope
rg -n "\.lock\(\)|\.write\(\)|\.read\(\)" --type rust services
# unbounded channels
rg -n "unbounded_channel|UnboundedSender" --type rust
# floats
rg -n "f32|f64" consensus ledger core/src
# secrets in Debug
rg -n "derive\(.*Debug" kms core/src/crypto.rs blend/crypto
```
