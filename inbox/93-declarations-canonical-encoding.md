# Audit Report — `Declarations` byte encoding: canonical or removed?

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/93`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `core/src/sdp`, `core/src/codec`, `ledger/src/cryptarchia`, `ledger/src/mantle/sdp`, `services/chain/chain-service/src/states.rs`, `services/storage/src/recovery.rs`
Date: `2026-09-12` — author: `claude-fable-5.1` — status: `final`

Specifications read (logos-lips `master` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd`): `bedrock-service-declaration-protocol.md` (v1.4.0 RFC), `bedrock-service-reward-distribution.md` (v1.3.1 RFC), `bedrock-anonymous-leaders-reward.md`.

---

## 1. Summary

- Overall assessment: the `Declarations` encoding is still not canonical, the `Bytes` conversions still have no caller, and nothing hashes, signs, or byte-compares the encoding; but since the #77 report the same encoding has become part of every LIB recovery record written to RocksDB (twice per record), so the "unused impl" framing no longer holds. Removing the conversions and switching the maps to `BTreeMap` is a small, safe change; a correction to the issue text is that `ServiceType` does not derive `Ord`, so it needs one added.
- Findings: 0 critical · 0 high · 0 medium · 0 low · 3 informational
- Key themes: latent non-determinism in serialised consensus state; decoders that accept non-canonical input; dead conversion impls that invite misuse.
- Must-fix before launch: none.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `core/src/sdp/mod.rs` L265-L269, L375-L378, L427-L474 | `ServiceType`, `DeclarationId`, `Declarations`, the `From`/`FromIterator` constructors and the two `TryFrom` `Bytes` conversions |
| `core/src/codec/mod.rs` L22-L30, L62-L66; `core/src/codec/bincode/mod.rs` L12-L29 | the blanket `SerializeOp`/`DeserializeOp` impls that `to_bytes`/`from_bytes` resolve to, and the bincode options |
| `ledger/src/cryptarchia/mod.rs` L64-L107, L208-L229 | `EpochState.active_declarations: Arc<Declarations>`; `LedgerState.epoch_state` / `next_epoch_state` |
| `ledger/src/lib.rs` L246-L251; `ledger/src/mantle/sdp/mod.rs` L40, L168-L175, L293-L299, L589-L636 | outer `LedgerState`, `SdpLedger.services`, `ServiceState.declarations`, the two builders that produce `lb_core::sdp::Declarations` |
| `services/chain/chain-service/src/states.rs` L10-L30; `services/storage/src/recovery.rs` L98-L133 | the recovery record and how it is written and read |
| `services/chain/chain-service/src/service/mod.rs` L345-L382, L629-L636; `services/api/src/http/mantle.rs` L881-L935; `nodes/node/binary/src/api/handlers.rs` L1321-L1332; `tests/src/cucumber/steps/nodes/steps/blend.rs` L398-L420 | every consumer of a `Declarations` value found by `grep -rn Declarations --include='*.rs'` outside `core` and test modules |
| `bincode 1.3.3` `src/ser/mod.rs` L178-L182, `src/de/mod.rs` L353-L392; `serde 1.0.228` `src/core/de/impls.rs` L1540-L1543; `rpds 1.2.1` `src/map/hash_trie_map/mod.rs` L117-L127, L1152-L1160, `src/map/red_black_tree_map/mod.rs` L1494-L1502, `src/utils/mod.rs` L4 | read from the vendored registry sources to pin down map ordering and duplicate-key behaviour |

**Out of scope**

- Every other `HashMap`/`HashSet` that reaches a `Serialize` derive in the workspace; that is sub-issue #33's sweep. The one such map found inside the LIB state on the way (`SdpLedger.services`) is recorded under LB-001.
- Ordering effects on Blend membership and reward computation from iterating these maps; #77 (PR 91) covers position dependence and its table still holds at this SHA (re-checked the three chain-service call sites).
- Wire-format versioning of the recovery record (PR 174 / #181) and its size and write cost (PR 209 / #188, #211).
- Third-party crates assumed correct: `bincode`, `serde`, `rpds`, `rocksdb`.

**Assumptions**

- The recovery record in RocksDB is written and read only by the same node (`services/storage/src/recovery.rs`); an attacker with write access to the node's database is out of the threat model.
- Spec is correct; the SDP spec defines `declarations` as a list indexed by `declaration_id` (L255-L258) and defines no encoding for the set as a whole and no state commitment over it.

## 3. Method

- Manual review of the in-scope paths, working through sub-issue #93 under parent #8, with the #77 report (PR 91), the #53 report (PR 75), the #56 codec report (PR 68), PR 174 (#70), PR 120 (#104) and PR 209 (#188) read for overlap.
- Spec conformance against the three logos-lips documents listed in the header, for whether any canonical encoding or commitment of the declaration set is specified.
- Automated tooling: none against the node. A 100-line standalone crate (Appendix B) depending on the same pinned `bincode 1.3.3` and `serde 1.0.228`, built with the node's exact bincode options, mirrors the `Declarations` derive and measures the three properties claimed below (ordering, duplicate keys, length prefix). `cargo 1.94.1 --offline --release`, 15 s build.
- Dynamic testing: none.

Checklist items from #93, with the answer:

| Item | Result |
|---|---|
| No new consumer of the `Bytes` conversions | Holds. At `a805329f` the only matches for `Declarations` together with `try_from`/`to_bytes`/`from_bytes`/`Bytes::` are the two impls themselves (`core/src/sdp/mod.rs` L460-L474) and one test helper (`ledger/src/mantle/sdp/rewards/test_utils.rs` L43, `.into()` from a `HashMap`). `c-bindings` uses only `DeclarationId` (`c-bindings/src/api/blend.rs` L23-L27). Storage never sees a `Declarations` value directly. **However**, the same serde derive is exercised indirectly: see LB-001. |
| Switch to `BTreeMap` or delete the conversions | Recommended: both. `DeclarationId` derives `Ord` (`core/src/sdp/mod.rs` L375); `ServiceType` does **not** (L265: `Clone, Copy, Debug, Eq, PartialEq, Hash, Serialize, Deserialize, EnumIter`), and no manual `Ord` impl exists in the workspace, so the issue's "keys are `Ord`" needs one derive added. Diff sketch under LB-001. |
| Is the on-disk LIB state ever compared byte-wise across restarts or nodes | No. `StorageRecoveryBackend::save_state` (`services/storage/src/recovery.rs` L111-L133) stores `state.to_bytes()` under a fixed key; `load_state` (L98-L108) does `State::from_bytes` and nothing else; there is no checksum, no equality check, no `Hash`. The only `LedgerState` byte round-trip elsewhere is the unit test at `services/chain/chain-service/src/states.rs` L452-L453, which compares fields after decoding, not bytes. IBD (`services/chain/chain-network/src/bootstrap/ibd.rs`) references `LedgerState` only in its test module (L370-L380, L938-L948); blocks are synced, never ledger state. The wallet recovery state (`services/wallet/src/states.rs` L33-L40) does not embed `LedgerState`. The e2e suite reads the declarations endpoint (`tests/.../blend.rs` L398-L420) and looks a provider up in the decoded map; it never compares serialised forms. |

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | `Declarations` has no canonical encoding, and that encoding is now written twice into every LIB recovery record | Determinism | Informational | High | Open |
| LB-002 | The `Declarations` decoder accepts duplicate keys and any length prefix, so decoding is not canonical either | Data Validation | Informational | High | Open |
| LB-003 | The unused `TryFrom<Bytes>` / `TryFrom<Declarations> for Bytes` impls are the only public bridge from raw bytes to this type | Determinism | Informational | High | Open |

### LB-001 · `Declarations` has no canonical encoding, and that encoding is now written twice into every LIB recovery record

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Determinism |
| Target | `core/src/sdp/mod.rs:L427-L428` (`struct Declarations`), `ledger/src/cryptarchia/mod.rs:L106` (`active_declarations`), `services/chain/chain-service/src/states.rs:L14` (`lib_ledger_state`) |
| Status | Open |

**Description**

`Declarations` is a newtype over two nested `std::collections::HashMap`s with a derived `Serialize`/`Deserialize` (`core/src/sdp/mod.rs` L427-L428, `use std::collections::HashMap` at L8). `to_bytes()` resolves to the blanket `SerializeOp` impl (`core/src/codec/mod.rs` L22-L30) and hence to `bincode::serialize` with the options at `core/src/codec/bincode/mod.rs` L23-L29 (little-endian, fixint, no limit, reject trailing). bincode's `serialize_map` writes the length and then the entries in the map's iteration order (`bincode-1.3.3/src/ser/mod.rs` L178-L182); `std::HashMap` iterates in an order fixed by its per-instance `RandomState` seed. Two nodes, or one node before and after a restart, therefore encode the same set of declarations as different byte strings.

What changed since #77 was filed at `c3ff08e4`: `EpochState` (derives `Serialize`, `ledger/src/cryptarchia/mod.rs` L64) carries `active_declarations: Arc<Declarations>` (L106), and cryptarchia's `LedgerState` (derives `Serialize`, L208-L210) carries both `next_epoch_state` and `epoch_state` (L228-L229). The outer `lb_ledger::LedgerState` (`ledger/src/lib.rs` L246-L251) is the `lib_ledger_state` field of `CryptarchiaConsensusState` (`services/chain/chain-service/src/states.rs` L14), which `StorageRecoveryBackend::save_state` serialises with `to_bytes()` into RocksDB (`services/storage/src/recovery.rs` L124-L126). So every recovery write now contains two non-canonical `Declarations` encodings. Each is built fresh by `SdpLedger::active_declarations` (`ledger/src/mantle/sdp/mod.rs` L615-L636) via `collect()` into new `HashMap`s, so even consecutive writes on one node differ in byte layout.

The rest of the LIB state's SDP data does not have this property: `ServiceState.declarations` is an `rpds::RedBlackTreeMapSync` (`ledger/src/mantle/sdp/mod.rs` L40, L172), whose `Serialize` walks the tree in key order (`rpds-1.2.1/src/map/red_black_tree_map/mod.rs` L1494-L1502). The one exception is `SdpLedger.services: rpds::HashTrieMapSync<ServiceType, Service>` (L295): `HashTrieMapSync` defaults its hasher to `std::collections::hash_map::RandomState` (`rpds-1.2.1/src/map/hash_trie_map/mod.rs` L117, L127; `src/utils/mod.rs` L4) and serialises by iteration (L1152-L1160). With one service today this cannot reorder, but it is the same class and will matter the day a second `ServiceType` is added.

Measured with the mirror crate in Appendix B, using the node's bincode options: 16 declarations under one service type, encoded once from each of 32 freshly constructed `HashMap`s, produced 32 distinct byte strings of identical length (804 B); two maps with equal contents compared equal as values and unequal as bytes.

**Exploit scenario**

None. At this SHA no code hashes, signs, transmits, or byte-compares any of these encodings (Method table, item 3), so the non-determinism has no observable effect: the record is decoded back into `HashMap`s whose contents are order-independent (`Declarations: PartialEq` derives from `HashMap: PartialEq`). The actual impact is the hazard the issue describes: a future state checksum, snapshot-sync of `LedgerState`, or a debugging comparison of two nodes' recovery records would disagree on identical state, and the derive makes that a silent bug rather than a compile error.

**Recommendation**

- *Short term*: switch both levels to `BTreeMap` and give `ServiceType` an `Ord`. This makes the encoding a pure function of the set with no other change, because `DeclarationId` is already `Ord` and every consumer uses `iter()`, `get()`, `values()` or `len()` (`services/chain/chain-service/src/service/mod.rs` L345-L382, L629-L636; `ledger/src/mantle/sdp/mod.rs` L589-L636) or constructs it with `default()` (`ledger/src/cryptarchia/mod.rs` L775-L785, L1170-L1180; `services/chain/chain-leader/src/leadership.rs` L624, L717). Sketch, against `core/src/sdp/mod.rs`:

  ```diff
  -use std::{collections::HashMap, hash::Hash};
  +use std::{collections::{BTreeMap, HashMap}, hash::Hash};
   ...
  -#[derive(Clone, Copy, Debug, Eq, PartialEq, Hash, Serialize, Deserialize, EnumIter)]
  +#[derive(Clone, Copy, Debug, Eq, PartialEq, Hash, PartialOrd, Ord, Serialize, Deserialize, EnumIter)]
   pub enum ServiceType {
   ...
   #[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq, Default)]
  -pub struct Declarations(HashMap<ServiceType, HashMap<DeclarationId, Declaration>>);
  +pub struct Declarations(BTreeMap<ServiceType, BTreeMap<DeclarationId, Declaration>>);
  ```

  The three `.collect()` sites in `ledger/src/mantle/sdp/mod.rs` (L589-L603, L615-L636) and the test helper at `rewards/test_utils.rs` L43 need their inner `HashMap` type changed to `BTreeMap`; the `From<HashMap<..>>` impl at L446-L450 can stay as a conversion that sorts, or go. Then add a round-trip test in `core/src/sdp` that builds the same set through two differently ordered insert sequences and asserts equal `to_bytes()`.
- *Long term*: treat `SdpLedger.services` the same way (`rpds::RedBlackTreeMapSync<ServiceType, Service>` once `ServiceType: Ord`), and let #33 carry a workspace rule that no `Serialize` type reachable from `LedgerState` contains a `RandomState`-keyed map. If the recovery record is reshaped under #211, make "deterministic encoding" an explicit acceptance criterion there.

**References**: #77 report S-001 (PR 91); SDP spec `Declaration Storage` L216-L258 (no set encoding specified); #33; #211.

### LB-002 · The `Declarations` decoder accepts duplicate keys and any length prefix, so decoding is not canonical either

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Data Validation |
| Target | `core/src/sdp/mod.rs:L460-L466` (`TryFrom<Bytes> for Declarations`), `core/src/codec/bincode/mod.rs:L23-L29` (`with_no_limit`) |
| Status | Open |

**Description**

Canonicality has two directions. LB-001 covers encode; on decode, serde's `HashMap` visitor loops `next_entry()` and `insert`s each pair (`serde-1.0.228/src/core/de/impls.rs` L1540-L1543), so a byte string carrying the same `DeclarationId` twice decodes without error and the last value wins. bincode's `deserialize_map` trusts the length prefix as the entry count (`bincode-1.3.3/src/de/mod.rs` L353-L392) and the node's options set `with_no_limit()`, so an inflated count only fails when the reader hits end of input (serde caps the pre-allocation, so this is a failed decode, not an allocation).

Measured (Appendix B): a hand-built 118-byte record declaring two entries with the same key decoded to a map of length 1 holding the second value; re-encoding it produced 69 bytes, not the input. A record with a `u64::MAX` count failed with `io error: unexpected end of file`.

**Exploit scenario**

None at this SHA. The only bytes this decoder ever sees are the node's own recovery record, read back by `load_state` (`services/storage/src/recovery.rs` L98-L108). The observation matters only for the `TryFrom<Bytes>` entry point, which is public, has no caller, and would let a future HTTP or FFI path feed attacker-controlled bytes to a decoder that silently merges duplicates. Whether the mantle transaction decoder has the same shape is #56's and #31's territory and was not re-checked here.

**Recommendation**

- *Short term*: none beyond LB-003 (remove the entry point). With `BTreeMap` the duplicate-key behaviour is unchanged (serde's `BTreeMap` visitor also `insert`s), so a strict decoder would need a custom `Deserialize` that rejects a repeated or out-of-order key; do that only if the type ever gains an untrusted input path.
- *Long term*: keep `Declarations` off every untrusted-bytes path, and document on the type that its serde form is a storage format, not a wire format.

**References**: #31 report (PR 196) for the serde sweep; #56 report (PR 68) for the codec-level limits.

### LB-003 · The unused `TryFrom<Bytes>` / `TryFrom<Declarations> for Bytes` impls are the only public bridge from raw bytes to this type

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Determinism |
| Target | `core/src/sdp/mod.rs:L460-L474` |
| Status | Open |

**Description**

Both impls delegate to the blanket `from_bytes`/`to_bytes` (L463-L465, L471-L473) and have no caller anywhere in the workspace, including `c-bindings`, `services`, `nodes`, `tests` and `tools` (grep in Method). They are the only place where `Bytes` is spelled next to `Declarations`, and they are exactly the API a future storage adapter or FFI accessor would reach for, at which point LB-001 and LB-002 stop being latent. They also predate the recovery-record change and were presumably written for a storage use that never materialised.

**Exploit scenario**

None; dead code. Impact is the maintenance hazard described in the issue.

**Recommendation**

- *Short term*: delete both impls (a 15-line removal with no fallout, since nothing calls them) together with the now-unused `use bytes::Bytes` if nothing else in the module needs it (`Bytes` is used at L11 only for these impls; check `BoundedMultiaddrBytes` at L102 is a `BoundedVec`, not `bytes::Bytes`). If a bytes bridge is wanted later, add it after LB-001's `BTreeMap` change and name it for its purpose (`to_storage_bytes`).
- *Long term*: none.

**References**: #77 report S-001.

## 5. Suggestions (non-security)

### S-001 · Two `Declarations` builders, two policies

`SdpLedger::declarations()` (`ledger/src/mantle/sdp/mod.rs` L589-L603) keeps empty per-service maps; `active_declarations()` (L615-L636) drops them (`if entries.is_empty() { None }`). Both feed the same type, so `Declarations` equality between "all declarations of an empty service" and "no entry for that service" depends on which builder produced it. Harmless today (one service, and `for_service` returns `Option` either way) but worth unifying when the type is touched for LB-001.

### S-002 · The `sdp/declarations` and `sdp/snapshot` endpoints serve tip state, not finalized state

Both handlers read `self.cryptarchia.tip()` (`services/chain/chain-service/src/service/mod.rs` L346, L365). The SDP spec says "Every query must return information for a finalized state only" (L409). Not a canonicality issue and possibly already noted by the #53 or #66 reports; recorded here because it was seen while enumerating consumers.

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

## Appendix B — Measurement harness

Standalone crate, `bincode = "=1.3.3"` (no default features) and `serde = "=1.0.228"` (`derive`), i.e. the versions in the node's `Cargo.lock`. The type mirrors `Declarations` with `u32` for the `ServiceType` variant index and `[u8; 32]` for `DeclarationId`; the options are copied from `core/src/codec/bincode/mod.rs` L23-L29.

```rust
#[derive(Serialize, Deserialize, PartialEq, Eq, Debug, Default, Clone)]
struct Declarations(HashMap<u32, HashMap<[u8; 32], Declaration>>);

fn opts() -> impl bincode::Options {
    bincode::DefaultOptions::new()
        .with_little_endian().with_no_limit()
        .with_fixint_encoding().reject_trailing_bytes()
}
// [1] 16 entries, two maps with different hasher seeds, then 32 fresh RandomState maps
// [2] hand-built record: outer len 1, variant 0, inner len 2, same key twice
// [3] hand-built record: outer len 1, variant 0, inner len u64::MAX
```

Output:

```
[1] equal values: true
[1] equal bytes : false (len 804 vs 804)
[1] distinct encodings of one 16-entry set over 32 RandomState maps: 32
[2] duplicate-key wire: decoded len 1 nonce 999 (input claimed 2 entries)
[2] re-encoded == input: false (69 vs 118 bytes)
[3] len=u64::MAX prefix -> Err("io error: unexpected end of file")
```
