# Audit Report — Codec: trailing bytes, versioning of unknown variants, bincode limits

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/56`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `19353c61963d4ef8c37ad00d24fdf0f08a482887` — component(s): `codec`, `codec/macros`, `core/src/codec` (bincode wrapper), and every hand-written `BinaryDecode` / binary `serde::Deserialize` impl outside `codec/`
Date: `2026-09-07` — author: `Claude (Fable 5.1), operated by davidrusu` — status: `final`

---

## 1. Summary

- Overall assessment: the wire codec is in good shape; every network ingress rejects trailing bytes, length prefixes are checked before any allocation, and unknown discriminants are rejected everywhere. The gaps found are a latent inconsistency in one serde bridge, an unlimited bincode configuration that is currently saved by its callers, and one trusted-input decoder that ignores leftovers.
- Findings: `0` critical · `0` high · `0` medium · `1` low · `2` informational
- Key themes: "trailing-byte checks live at call sites, not in the trait", "bincode safety depends on transport caps", "no negotiated upgrade path for wire versions".
- Must-fix before launch: none.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `codec/src/{lib,bounded_vec,array,numbers,boolean,error,fixtures}.rs` | The `BinaryEncode`/`BinaryDecode` traits, the primitive codecs, length-prefix handling, allocation behaviour, fixture machinery. |
| `codec/macros/src/lib.rs` | `#[derive(BinaryCodec)]` and `codec_fixtures!` expansions. |
| `core/src/codec/{mod,bincode/mod,errors}.rs` | The bincode 1.3 wrapper (`SerializeOp` / `DeserializeOp`) used for sync messages, gossiped transactions, blend payloads, and storage. |
| Every `impl BinaryDecode` outside `codec/` (37 impls) | `core/src/{header,mantle,sdp,proofs}`, `blend/message`, `blend/proofs`, `kms/keys`, `consensus/cryptarchia-engine/src/time.rs`, `logos_sql/src/protocol/codec.rs`. Read for truncation, width, discriminant, and canonicality. |
| Binary `serde::Deserialize` bridges | `RawMantleTx`, `SignedMantleTx`, `Op`, `Version`, `Block<Tx>`, `KnownBlocks`, `PaddedPayloadBody`, `BoundedVec`, fixed-array helpers in `utils/src/lib.rs`. |
| Network ingress call sites | `services/chain/chain-network/src/network/adapters/libp2p.rs`, `services/tx-service/src/network/adapters/libp2p.rs`, `services/blend/src/core/dispatcher/libp2p.rs`, `blend/network/src/core/with_{core,edge}/behaviour`, `consensus/cryptarchia-sync/src/libp2p/{packing,messages}.rs`, `logos_sql/src/protocol/mod.rs`. |
| Third-party behaviour relied on | `bincode 1.3.3` `SliceReader` (`src/de/read.rs`), `serde_core 1.0.228` `size_hint::cautious`, `unsigned-varint 0.7.2` minimality check, `multiaddr 0.18.2` `TryFrom<Vec<u8>>`. Read, not audited. |

**Out of scope**

- Semantic validation after decoding (signature checks, proof verification, ledger rules). Only "bytes in, typed value out" was reviewed.
- The blend replay cache and padding/oracle behaviour (#59), gossip amplification and peer scoring (#57), storage corruption handling (#63), the ZK proof byte parsers behind `PoLProof::from_bytes` etc. (#64).
- Human-readable (JSON) serde paths, which are only used by the HTTP API and config.
- Assumed correct: `libp2p` (gossipsub `max_transmit_size` enforcement), `rocksdb`, `ed25519-dalek`, `ark-*` compressed point parsing, `blake2`.

**Assumptions**

- Facts from #19 verified at this commit: `[profile.release]` sets no `overflow-checks` (`Cargo.toml:11-14`), so release arithmetic wraps; `arithmetic_side_effects`, `as_conversions`, `indexing_slicing`, `unwrap_used`, `expect_used` are allowed workspace-wide (`Cargo.toml:326-347`), so none of those patterns are caught by clippy. No `fuzz/` directory exists.
- 64-bit targets. `decode_length_prefix` is unreachable in its 8-byte branch on 32-bit targets because `MAX_LENGTH <= u32::MAX as usize` is always true there, so the `expect` at `codec/src/bounded_vec.rs:71` cannot fire.
- Transport caps hold: libp2p gossipsub `max_transmit_size` = 16 MiB (`libp2p/src/behaviour/mod.rs:22,77`), sync frames ≤ 16 MiB (`consensus/cryptarchia-sync/src/libp2p/mod.rs:9`, enforced at `packing.rs:67-72`), blend payload bodies ≤ 18 192 bytes (`blend/message/src/message/payload.rs:18`).

## 3. Method

- Manual review of the in-scope paths, working through issue `#56` and its three checklist items; the parent `#10` questions were used as the reading guide but its own deliverable is not claimed here.
- Enumerated every `impl BinaryDecode` outside `codec/` with `grep -rnE "impl(<[^>]*>)? BinaryDecode(<[^>]*>)? for"` (37 hits) and every non-test `bincode` symbol use (`core/src/codec/bincode/mod.rs`, `logos_sql/src/db.rs:951`), then traced each network ingress to the decoder that terminates it.
- Verified the third-party behaviour the codec relies on by reading the vendored sources in `~/.cargo/registry`: bincode's slice reader does `get_byte_slice(len)?.to_vec()` (`bincode-1.3.3/src/de/read.rs:126-128`), so a claimed byte length is checked against the input before allocation; serde caps sequence preallocation at 1 MiB (`serde_core-1.0.228/src/private/size_hint.rs:12-23`); `unsigned-varint` rejects non-minimal varints (`decode.rs:72-77`).
- Automated tooling: `cargo test -p logos-blockchain-codec -p logos-blockchain-codec-macros` on `rustc 1.98.0 (88d9e12ae 2026-08-18)`: 35 passed, 0 failed, including the allocation-counting tests. A two-test proof of concept for LB-001 was compiled against `logos-blockchain-core` (see the finding). No fuzzing; no fuzz harness exists in the repository.
- Dynamic testing: none beyond the unit tests above.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | `Op`'s binary serde bridge accepts trailing bytes and has no size cap, unlike its siblings | Data Validation | Low | Low | Open |
| LB-002 | bincode is configured with an infinite size limit; safety rests on every caller pre-bounding input | Denial of Service | Informational | Medium | Open |
| LB-003 | Genesis `CryptarchiaParameter` decoder ignores leftover inscription bytes | Data Validation | Informational | High | Open |

### LB-001 · `Op`'s binary serde bridge accepts trailing bytes and has no size cap, unlike its siblings

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Data Validation |
| Target | `core/src/mantle/ops/mod.rs:115-129` (`impl<'de> Deserialize<'de> for Op`) |
| Status | Open |

**Description**

Three types bridge the canonical `lb_codec` encoding into serde for binary formats by serialising themselves as a byte string and decoding it on the way back. Two of them check for leftovers and one of them bounds the byte string:

- `RawMantleTx` — `core/src/mantle/transactions/mantle_tx.rs:189-197`: bounded to `MAX_BLOCK_TRANSACTIONS_SIZE`, errors with "contains trailing bytes" if `remaining` is non-empty.
- `SignedMantleTx<Unverified>` — `core/src/mantle/transactions/signed_mantle_tx.rs:515-524`: unbounded `Vec<u8>`, errors with "not all bytes were consumed".
- `Op` — the odd one out:

```rust
// core/src/mantle/ops/mod.rs:122-126
} else {
    let bytes = <Vec<u8>>::deserialize(deserializer)?;
    Self::decode(&bytes, &())
        .map(|(_, op)| op)
        .map_err(serde::de::Error::custom)
}
```

`(_, op)` discards the unconsumed tail, so any byte string whose prefix is a valid op decodes successfully, and there is no cap on the byte string beyond what the outer reader holds. `decode(encode(x)) == x` holds but `encode(decode(b)) == b` does not for this type on the serde path: two different bincode encodings map to one `Op`.

Proof of concept (added under `#[cfg(test)]` in `core/src/mantle/ops/mod.rs`, run with `cargo test -p logos-blockchain-core --lib poc_trailing_bytes_audit`, then removed):

```rust
let op = Op::fixtures().into_iter().next().unwrap().value;
let mut encoded = op.encode().into_vec();
encoded.extend_from_slice(&[0xAA; 7]);
let envelope = bincode::serialize(&encoded).unwrap();
let decoded: Op = bincode::deserialize(&envelope).unwrap();   // succeeds
assert_eq!(decoded, op);
// The same experiment on RawMantleTx returns Err, as intended.
```

**Exploit scenario**

None reachable today. Every untrusted binary ingress terminates in `RawMantleTx` or `SignedMantleTx` (gossip: `services/tx-service/src/network/adapters/libp2p.rs:71`; blend: `services/blend/src/core/dispatcher/libp2p.rs:124`; sync blocks: `core/src/block/mod.rs:104-130` via `BlockTransactions<Tx>`), and the HTTP API takes `Json<SignedMantleTx<Preverified>>` (`nodes/node/binary/src/api/handlers.rs:754`), which uses the human-readable path. `Op` is not a field of any bincode-decoded wire type at this commit. The risk is that the next type to embed an `Op` in a bincode message, or a future direct `Op` RPC, inherits a non-canonical decoder silently; content-addressed dedup (gossipsub message id is `blake2b(data)`, `libp2p/src/behaviour/gossipsub/mod.rs:7-11`) would then be bypassable by appending bytes.

**Recommendation**

- *Short term*: mirror `mantle_tx.rs:189-197` in `Op::deserialize`: bound the byte string (an `Op` cannot legitimately exceed `MAX_BLOCK_TRANSACTIONS_SIZE`) and call `Self::decode_all` (`codec/src/lib.rs:89-95`) instead of `decode`.
- *Long term*: give `lb_codec` one shared helper for the "canonical bytes inside serde" pattern (bounded byte string + `decode_all`) and make the three existing bridges use it, so the leftover check is in one place instead of being re-implemented per type. Add a fixture-driven test in `codec_fixtures!` that, for any type implementing both `BinaryDecode` and binary serde, asserts `bincode::deserialize(fixture_bytes ++ [0xAA])` fails.

**References**: sibling implementations at `mantle_tx.rs:181-200` and `signed_mantle_tx.rs:507-527`; the existing regression test `binary_serde_rejects_trailing_bytes_inside_transaction_envelope` (`mantle_tx.rs:252-259`) shows the intended contract.

### LB-002 · bincode is configured with an infinite size limit; safety rests on every caller pre-bounding input

| | |
|---|---|
| Severity | Informational |
| Difficulty | Medium |
| Category | Denial of Service |
| Target | `core/src/codec/bincode/mod.rs:23-29` (`OPTIONS`), `:66-70` (`deserialize`); also `logos_sql/src/db.rs:951-956` (`checkpoint_options`) |
| Status | Open |

**Description**

Checklist item 3 asks whether every bincode use on untrusted input has a size limit. It does not: the shared options are built with `.with_no_limit()` (`bincode::config::Infinite`) and `deserialize` applies them unchanged. `serialize_bounded` (`:47-56`) does apply `with_limit(MAX)`, but only on the encode side. `logos_sql`'s checkpoint options (`db.rs:951-956`) likewise set no limit.

Why this is not currently exploitable:

1. Every untrusted call goes through `deserialize(data: &[u8])`, i.e. bincode's `SliceReader`. Its `get_byte_buffer` is `self.get_byte_slice(length).map(|x| x.to_vec())` (`bincode-1.3.3/src/de/read.rs:126-128`), so a hostile byte-string length is checked against the remaining input before anything is allocated. There is no `IoReader` use in the workspace.
2. For non-byte sequences, bincode passes the claimed length to serde as `size_hint`, and serde's `Vec`/`HashSet` visitors clamp preallocation to 1 MiB (`serde_core-1.0.228/src/private/size_hint.rs:12-23`). The workspace's own `BoundedVec` serde visitor rejects oversize hints outright (`utils/src/bounded/vec.rs:79-85`) and stops at `MAX` elements (`:89-95`).
3. The transports cap the slice: sync frames at `MAX_MSG_LEN` = 16 MiB checked before `vec![0u8; data_length]` (`consensus/cryptarchia-sync/src/libp2p/packing.rs:63-75`), gossipsub at `DATA_LIMIT` = 16 MiB (`libp2p/src/behaviour/mod.rs:22,77`), blend payload bodies at 18 192 bytes.
4. No bincode-decoded wire type contains a collection of zero-sized elements, which is the one shape where bincode would loop on a claimed length without consuming input (checked with `grep -rnE "Vec<NoOpProof>|Vec<\(\)>|HashSet<\(\)>|Vec<PhantomData"`: no hits). `lb_codec` guards this case explicitly (`codec/src/bounded_vec.rs:84,132-137`); bincode has no such guard.

So the property "bounded before allocation" holds, but it is a property of the callers and of two dependencies' internals, not of the codec configuration. A future call site that reads from an `impl Read`, or a dependency upgrade that changes `SliceReader`'s order of operations, would remove the protection without any compile-time signal.

**Exploit scenario**

Not exploitable at this commit. Impact if a caller regresses: a peer sends an 8-byte length prefix claiming `u64::MAX` bytes; with an `IoReader` bincode would `resize` a buffer to that length (`read.rs:143-146`) and abort the process.

**Recommendation**

- *Short term*: set `.with_limit(MAX_MSG_LEN as u64)` (or a dedicated `MAX_WIRE_MESSAGE_BYTES` in `core`) on `OPTIONS`, and add a unit test that `deserialize::<Vec<u8>>` of an 8-byte prefix claiming more than the limit fails with `SizeLimit`. Do the same in `logos_sql::checkpoint_options`.
- *Long term*: make `DeserializeOp::from_bytes` take a `UpperBoundedVec<u8, MAX>` (the bounded type already exists in `core/src/codec/mod.rs:9,38-50`) so the bound is visible in the signature; the `TODO` at `core/src/codec/mod.rs:2-3` about replacing bincode is the natural moment to do this.

**References**: bincode 1.x documentation notes `DefaultOptions` has no limit and recommends `with_limit` for untrusted input.

### LB-003 · Genesis `CryptarchiaParameter` decoder ignores leftover inscription bytes

| | |
|---|---|
| Severity | Informational |
| Difficulty | High |
| Category | Data Validation |
| Target | `core/src/mantle/transactions/genesis_tx.rs:145-149` (`fn valid_cryptarchia_inscription`) |
| Status | Open |

**Description**

```rust
// core/src/mantle/transactions/genesis_tx.rs:145-149
Ok(
    CryptarchiaParameter::decode(inscription.inscription.as_ref(), &())
        .map_err(|e| Error::InvalidCryptarchiaParameter(format!("Decoding error: {e}")))?
        .1,
)
```

The `.1` drops the unconsumed remainder. An inscription whose prefix is a valid `CryptarchiaParameter` followed by arbitrary bytes is accepted as the genesis parameter. The surrounding checks (`:127-143`) validate parent, channel id, and signer but not exhaustiveness. This is the only `BinaryDecode` entry point outside the codec crate that neither calls `decode_all` nor checks `remaining`; the other three all do (`Proposal::decode_all` at `services/chain/chain-network/src/network/adapters/libp2p.rs:208`, `deserialize_encapsulated_message` at `blend/message/src/codec.rs:41-50`, `ChannelInscription::decode_all` at `logos_sql/src/protocol/mod.rs:288`).

**Exploit scenario**

The genesis transaction comes from operator configuration, so an attacker needs to control the config to trigger this; the impact is then limited to two genesis files with different bytes but identical parameters both being accepted, which does not change any consensus outcome since the tx hash still covers the full inscription. Recorded because the checklist asks for trailing bytes to be rejected everywhere and because genesis material tends to be copy-pasted between tools.

**Recommendation**

- *Short term*: replace `decode(..)?.1` with `CryptarchiaParameter::decode_all(inscription.inscription.as_ref(), &())?`.
- *Long term*: covered by the `decode_all`-only helper suggested in LB-001; consider making `BinaryDecode::decode` `#[doc(hidden)]` or `pub(crate)`-ish behind a `Partial` marker so that top-level callers must opt into partial decoding.

**References**: `codec/src/lib.rs:89-95` (`decode_all`).

## 5. Suggestions (non-security)

### S-001 · Unknown versions are rejected, but there is no negotiated upgrade path

Unknown discriminants and versions fail closed everywhere checked: header `Version` (`core/src/header/mod.rs:62-74,127-139`, one variant `Bedrock`), blend `PublicHeader` (`blend/message/src/message/public_header.rs:128-133`, hard equality with `LATEST_BLEND_MESSAGE_VERSION = 1`), `ActivityProof` (`core/src/sdp/blend.rs:44-49`), `Op` opcodes (`core/src/mantle/ops/mod.rs:168-207`), `PayloadType`, `ServiceType`, `ActivityMetadata`, `SqlParameter`, `CapturedFunction`, `logos_sql` `PAYLOAD_VERSION` (`logos_sql/src/protocol/mod.rs:284-286`), and bincode enum tags for sync messages. That is the right default for a launch. What is missing is any way for two node versions to coexist: sync uses a single configured `StreamProtocol` name (`libp2p/src/protocol_name.rs`), gossip topics are plain config strings, and blend accepts exactly one version byte. Any wire change is therefore a flag day. Before the first post-launch codec change, decide whether protocol names carry a version segment (libp2p supports negotiating several) and whether `Version` should be allowed to be greater-than-known for headers that are only relayed, not validated.

### S-002 · Add a fuzz harness for the four ingress decoders

No `fuzz/` directory or `arbitrary` dependency exists (confirmed from #19). The four network-facing entry points are pure functions of a byte slice and are ideal `cargo-fuzz` targets: `Proposal::decode_all`, `<SignedMantleTx<Unverified> as DeserializeOp>::from_bytes`, `deserialize_encapsulated_message(bytes, &NonZeroU64::new(3).unwrap())`, `ChannelInscription::decode`, plus `Block::<SignedMantleTx<Unverified>>::try_from(Bytes)` for the sync path. Each target should assert (a) no panic, (b) if `Ok`, re-encoding reproduces the input exactly (canonicality), and (c) `decode_all` and `decode` agree on the consumed length. The existing `codec_fixtures!` values make good seed corpora.

### S-003 · Make the zero-length-element guard a compile-time property

`codec/src/bounded_vec.rs:84` refuses zero-length element types only for `MAX > 1024`, at runtime, and only in `lb_codec`; the bincode path has no equivalent. A `const` assertion on `size_of::<T>() > 0 || MAX <= ZERO_LENGTH_ELEMENT_MAX` in the `BinaryDecode for BoundedVec` impl would move this to compile time for the common case (the `TODO` at `:129-131` already wants this), and a workspace test that walks the wire types for `Vec<ZST>` would close the bincode side.

---

## Appendix B — What was checked and ruled out

Recorded so the next reviewer does not repeat it.

| Property (from #56 / #10) | Result | Evidence |
|---|---|---|
| Length prefixes bounded before allocation | Holds. `BoundedVec::decode` checks `len` against `[MIN, MAX]` before the loop and starts from `Vec::new()`, never `with_capacity(len)`. | `codec/src/bounded_vec.rs:114-122`; allocation-counting tests `:508-546`. |
| Prefix width is a function of `MAX`, not of the message | Holds, so the prefix itself is canonical (`MAX = 255` → 1 byte, `256..=65535` → 2, etc.). | `codec/src/bounded_vec.rs:17-37`. Consumers agree: `Ops` (`MAX = 255`, 1 byte), `IndexedSignatures` (`u16::MAX`, 2 bytes), `SqlText`/`BoundedBytes` (`> u16::MAX`, 4 bytes, asserted at `logos_sql/src/protocol/codec.rs:36-37`). |
| Zero-length element loop | Guarded for `MAX > 1024`; test hangs rather than fails on regression. | `codec/src/bounded_vec.rs:84,132-137,340-346`. |
| Nested / recursive depth | Bounded by the types: no `BinaryDecode` or bincode wire type is recursive; the deepest chain is `Block → BoundedVec<Tx> → bytes → RawMantleTx → Ops → Op → op-specific BoundedVecs`. | Enumeration of the 37 impls and the serde bridges above. |
| Truncated input never panics | Holds. Every fixed-width read goes through `take` (`split_at_checked`) or `u8/u16/u32/u64::decode`; no slice indexing in any decoder. `[T; N]::decode` and the bincode fixed-array visitor iterate `N` fallible reads. | `codec/src/lib.rs:118-125`, `array.rs:28-43`, `utils/src/lib.rs:149-160`. |
| Integer decodes width-checked | Holds. Fixed-width little-endian; the only `as` casts are widening (`actual_len as usize`, `context.get() as usize` on a config value). | `codec/src/numbers.rs:17-29`; `blend/message/src/message/payload.rs:263`; `blend/message/src/encap/encapsulated.rs:622`. |
| Trailing bytes rejected at every untrusted ingress | Holds for all four `lb_codec` ingresses and both serde bridges that matter (see LB-001 for the latent exception and LB-003 for the trusted one). bincode itself is built with `reject_trailing_bytes()`. | `chain-network/.../libp2p.rs:208`; `blend/message/src/codec.rs:45-48`; `logos_sql/src/protocol/mod.rs:288`; `mantle_tx.rs:190-197`; `signed_mantle_tx.rs:516-524`; `core/src/codec/bincode/mod.rs:28`. |
| `bool`, `Fr`, field elements, discriminants canonical | Holds. `bool` rejects bytes other than 0/1; `Fr` rejects values ≥ modulus; `ProofOfQuota`/`ProofOfSelection` go through `try_from([u8; N])` which parses and validates; every enum decoder errors on unknown tags. | `codec/src/boolean.rs:22-28`; `codec/src/numbers.rs:61-64` with `zk/groth16/src/lib.rs:59-68`; `blend/proofs/src/{quota,selection}/mod.rs`. |
| `Locator` (multiaddr) canonical | Holds in the byte-preserving sense: `Multiaddr::try_from(Vec<u8>)` validates then stores the input bytes unchanged, and `unsigned-varint` rejects non-minimal varints, so distinct byte strings are distinct values and `encode(decode(b)) == b`. | `multiaddr-0.18.2/src/lib.rs:361-373`; `unsigned-varint-0.7.2/src/decode.rs:66-86`; `core/src/sdp/mod.rs:241-263`. |
| `PaddedPayloadBody` | Intentionally non-canonical: padding is random by spec and preserved through decode; `actual_len` is bounded before the fixed-size `take`. Dedup implications belong to #59. | `blend/message/src/message/payload.rs:82-89,162-185`. |
| bincode `Vec<u8>` / `String` amplification | None with a slice reader: length is checked against remaining input before `to_vec()`. | `bincode-1.3.3/src/de/read.rs:126-128`, `de/mod.rs:93-102`. |
| bincode `Vec<T>` / `HashSet<T>` amplification | Capped at 1 MiB preallocation by serde; `BoundedVec`'s own visitor rejects oversize hints. `KnownBlocks` preallocates a constant 5. | `serde_core-1.0.228/src/private/size_hint.rs:12-23`; `utils/src/bounded/vec.rs:75-100`; `consensus/cryptarchia-sync/src/libp2p/messages.rs:96-104`. |
| `#[derive(BinaryCodec)]` | Generates field-order concatenation and positional decode only; refuses generics, enums, and unit structs, so it cannot introduce a length prefix or a discriminant of its own. | `codec/macros/src/lib.rs:48-155`. |
| Fixture round-trip tests | Every codec must ship a golden vector; the generated test checks encode, decode, no leftovers, and round-trip. 35 tests pass in `codec/`. | `codec/src/fixtures.rs:137-203`; `codec/macros/src/lib.rs:216-277`. |

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
