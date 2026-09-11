# Audit Report — Fuzz harness for the network ingress decoders

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/69`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): `codec`, `core/src/block`, `core/src/mantle/transactions/tx_list`, `core/src/codec` (bincode wrapper), `blend/message`, `logos_sql/src/protocol`, `consensus/cryptarchia-sync/src/libp2p`
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `network-wire-format.md`, `mantle-transaction-encoding.md` (in full); `bedrock-v1.1-mantle-specification.md` (Mantle Transaction, Opcodes), `bedrock-v1.1-block-construction.md` (Block Proposal, Canonical Encoding, Block), `message-encapsulation.md` (Message Structure), `payload-formatting.md` (Body) by section
Date: `2026-09-11` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: seven libFuzzer targets covering every network ingress decoder ran for one hour each under AddressSanitizer (about 378 million executions in total) with no panic, no sanitizer report, no timeout and no canonicality violation on any `lb_codec` type or on the block and sync-response bincode types. The one invariant violation found is a determinism gap in a sync request, not a memory-safety or parsing flaw.
- Findings: `0` critical · `0` high · `0` medium · `1` low · `1` informational
- Key themes: "the decoders hold up under fuzzing", "the sync frame cap is not per message type", "`HashSet` on the wire is not byte-stable".
- Must-fix before launch: none.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/chain/chain-network/src/network/adapters/libp2p.rs:232` | `Proposal::decode_all` on gossipsub proposal bytes. Fuzzed as target `proposal`. |
| `services/tx-service/src/network/adapters/libp2p.rs:71` | `Item::from_bytes` on gossipsub transaction bytes; `Item` is `SignedOps<Preverified, StandardMode>` in the node (`nodes/node/binary/src/generic_services/mod.rs:24-28`). Fuzzed as `signed_ops_gossip` (bincode envelope, both `Unverified` and `Preverified`) and `signed_ops_codec` (the canonical bytes inside the envelope, `core/src/mantle/transactions/tx_list/signed_ops.rs:204-220`). |
| `blend/message/src/codec.rs:41-50` | `deserialize_encapsulated_message`, called from `blend/network/src/core/with_core/behaviour/utils.rs:103` and `with_edge/behaviour/mod.rs:203`. Fuzzed as `blend_message` with 1 and 3 layers. |
| `logos_sql/src/protocol/mod.rs:268-290` | `ChannelInscription::decode` on inscription bytes read from the chain. Fuzzed as `channel_inscription`. |
| `core/src/block/mod.rs:366-378` | `Block::<Tx>::try_from(Bytes)`, the sync path (`services/chain/chain-network/src/network/adapters/libp2p.rs:375`). Fuzzed as `block` for `Tx = SignedOps<Unverified, StandardMode>` and `SignedOps<Preverified, StandardMode>`. |
| `consensus/cryptarchia-sync/src/libp2p/packing.rs:58-76`, `libp2p/messages.rs`, `src/messages.rs` | `unpack_from_reader` and the three framed message types (`RequestMessage`, `DownloadBlocksResponse`, `GetTipResponse`). Fuzzed as `sync_messages`. |
| `codec/src/bounded_vec.rs:76-140` | The zero-length-element guard (checklist item 6). Reviewed, not fuzzed. |

**Out of scope**

- Everything after a successful decode: mempool admission, ledger validation, proof verification, blend decapsulation, SQL application. The `Preverified` targets do run `preverify()` (Ed25519 signature and structural checks, `core/src/mantle/ops/signed_op.rs:119-172`) because the node runs it inside `Deserialize`, but its correctness is not assessed here.
- Human-readable (JSON) serde paths (HTTP API, config).
- Third-party crates assumed correct: `bincode 1.3.3`, `serde`, `libfuzzer-sys 0.4`, `ed25519-dalek`, `ark-*`, `multiaddr`, `unsigned-varint`, `rusqlite`.
- Findings already recorded by the report for #56 (`inbox/56-codec-trailing-bytes-versioning-bincode.md`, PR #68): unbounded bincode `OPTIONS`, the `Op` serde bridge, the genesis parameter decoder. Not repeated; the transaction bridge it examined has since been replaced (see Appendix B).

**Assumptions**

- Facts from #19 re-verified at this commit: `[profile.release]` sets no `overflow-checks` (`Cargo.toml:11-14`); `arithmetic_side_effects`, `as_conversions`, `indexing_slicing`, `unwrap_used`, `expect_used`, `panic` are allowed workspace-wide (`Cargo.toml:326-360`); no `fuzz/` directory and no `arbitrary` dependency exist (`proptest`/`quickcheck` are declared at `Cargo.toml:249-252`; the harness does not use them). The fuzz build enables `-Coverflow-checks` and `-Cdebug-assertions`, so an arithmetic overflow in a decoder would have surfaced as a panic even though release builds wrap.
- 64-bit target. Transport caps hold as documented in the #56 report: gossipsub `max_transmit_size` = 16 MiB (`libp2p/src/behaviour/mod.rs:22,77`), sync frames ≤ `MAX_MSG_LEN` = 16 MiB (`consensus/cryptarchia-sync/src/libp2p/mod.rs:9`).
- The deployed blend layer count is 1 (`deployment/ceremony/genesis/*/deployment-template.yaml:3`, `nodes/node/binary/src/config/deployment/settings.yaml:3`); the issue's value of 3 was also run.

## 3. Method

- Manual review of the in-scope paths, working through issue `#69` (all six checklist items) under parent `#10`; the #56 report was read as the source of S-002/S-003.
- Spec conformance against `mantle-transaction-encoding.md` (ABNF) for the transaction targets and `bedrock-v1.1-block-construction.md` Canonical Encoding for the proposal target; `network-wire-format.md` for the bincode framing of blocks and sync messages.
- Harness: a `fuzz/` crate (`cargo-fuzz` 0.13.2, `libfuzzer-sys` 0.4) with seven targets, added as a workspace member on a private copy of the audited commit and built with `cargo +nightly-2026-05-22 fuzz build` (AddressSanitizer, `-Cdebug-assertions`, `-Coverflow-checks`, sancov level 4). The full source is in Appendix C. Three one-line visibility changes were needed and belong in the upstream PR: `mod libp2p;` → `pub mod libp2p;` (`consensus/cryptarchia-sync/src/lib.rs:2`), `mod packing;` → `pub mod packing;` (`consensus/cryptarchia-sync/src/libp2p/mod.rs:5`), `mod protocol;` → `pub mod protocol;` (`logos_sql/src/lib.rs:13`).
- Invariants per target (`fuzz/src/lib.rs`): no panic; if `decode_all` accepts the input, `encode` reproduces it byte for byte; `decode` and `decode_all` agree on the consumed length; when `decode` accepts a strict prefix, re-encoding the partial value reproduces exactly the consumed prefix. For bincode types: `from_bytes` then `to_bytes` reproduces the input (relaxed to same-length-and-decodes-again for `RequestMessage`, see LB-001). For the framed sync path: `unpack_from_reader` and `from_bytes` on the framed bytes agree.
- Seed corpora were generated from every relevant `codec_fixtures!` golden vector through the `CodecExamples::fixtures()` trait (`fuzz/examples/seed.rs`): `Proposal` (plus a variant with no uncles), `SignedOps`, `Ops`, `EncapsulatedMessage`, `ChannelInscription`, and hand-built bincode envelopes for the block and the seven sync message shapes, framed and unframed. 27 seed files; every target was run over its seeds alone (`-runs=0`) before the campaign.
- Automated tooling: `cargo-fuzz 0.13.2` on `rustc nightly-2026-05-22`; `rustc 1.98.1` for the seed generator (built with `RUSTFLAGS=""` because the repository's `.cargo/config.toml` forces `linker=rust-lld` on macOS, which cannot link build scripts against `libSystem` on this machine); `cargo test -p logos-blockchain-codec`: 35 passed, 0 failed, so the runtime zero-length guard tests still hold at this commit.
- Dynamic testing: the seven campaigns below, one libFuzzer process each, run in parallel on an Apple M-series machine (14 cores) with `-max_total_time=3600 -timeout=25 -rss_limit_mb=4096` and per-target `-max_len` (24 576 for proposal and blend, 65 536 for the transaction and inscription targets, 131 072 for block and sync).

## 4. Findings

**Campaign results** (audited commit `a805329f`, 3 600 s wall-clock per target):

| Target | Ingress | Executions | exec/s | Edges (`cov`) | Features (`ft`) | Corpus units (seeds → final) | Peak RSS | Crashes / timeouts / canonicality violations |
|---|---|---|---|---|---|---|---|---|
| `proposal` | `Proposal::decode_all` | 78 096 452 | 21 687 | 726 | 1 920 | 2 → 113 | 579 MB | 0 / 0 / 0 |
| `signed_ops_codec` | `SignedOps<Unverified>::decode_all` (canonical bytes) | 36 979 767 | 10 269 | 2 813 | 10 033 | 5 → 1 171 | 982 MB | 0 / 0 / 0 |
| `signed_ops_gossip` | `SignedOps<{Unverified,Preverified}>::from_bytes` (bincode envelope) | 69 549 371 | 19 313 | 5 314 | 12 599 | 2 → 1 428 | 182 MB | 0 / 0 / 0 |
| `blend_message` | `deserialize_encapsulated_message`, 1 and 3 layers | 65 944 739 | 18 312 | 729 | 906 | 1 → 39 | 511 MB | 0 / 0 / 0 |
| `channel_inscription` | `ChannelInscription::decode` | 40 552 310 | 11 261 | 492 | 1 951 | 1 → 392 | 594 MB | 0 / 0 / 0 |
| `block` | `Block::<SignedOps<{Unverified,Preverified}>>::try_from(Bytes)` | 25 270 438 | 7 017 | 2 473 | 7 547 | 2 → 635 | 347 MB | 0 / 0 / 0 |
| `sync_messages` | `unpack_from_reader` + `from_bytes` for the three sync types | 61 436 800 | 17 061 | 543 | 1 088 | 14 → 154 | 159 MB | 0 / 0 / 0 |

Total: 377 829 877 executions over 7 × 3 601 s. Coverage was still growing at the end of the hour on the transaction, block and inscription targets; it plateaued within minutes on `proposal`, `blend_message` and `sync_messages`, whose formats are fixed-layout and whose decoders are small. The `blend_message` corpus stayed at 39 units because every accepted message must carry an exactly `MAX_PAYLOAD_BODY_SIZE`-sized payload and a fixed-size private header per layer, so almost all mutations are rejected at a fixed offset; that is the decoder working as specified, not a harness gap, but a longer run adds little there. Note that `preverify()` cannot succeed on fuzzer-generated input (it needs valid Ed25519 signatures over the transaction hash), so the `Preverified` variants in `signed_ops_gossip` and `block` exercise the decode-then-reject path only; likewise no fuzzer-generated block passes `verify_header_alone`, so `body_root` validation is reached only by the seeds.

No crash artifacts, no `-timeout` alarms, no `rss_limit` hits and no assertion failures were produced by any target once the LB-001 seed was understood. The per-target execution counts above are the `stat::number_of_executed_units` lines of the logs.

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | `DownloadBlocksRequest` is not byte-stable: `additional_blocks` is a `HashSet` serialised in `RandomState` order | Determinism | Informational | Low | Open |
| LB-002 | Sync frame allocation is bounded by the transport cap, not by the message type: a request stream may declare 16 MiB | Denial of Service | Low | Low | Open |

### LB-001 · `DownloadBlocksRequest` is not byte-stable: `additional_blocks` is a `HashSet` serialised in `RandomState` order

| | |
|---|---|
| Severity | Informational |
| Difficulty | Low |
| Category | Determinism |
| Target | `consensus/cryptarchia-sync/src/libp2p/messages.rs:37-44` (`struct KnownBlocks`), `:46-70` (`impl Deserialize for KnownBlocks`) |
| Status | Open |

**Description**

`KnownBlocks` carries `additional_blocks: HashSet<HeaderId>` (`messages.rs:43`) and derives `Serialize`, so bincode writes the set in `HashSet` iteration order, which depends on the process's `RandomState`. The hand-written `Deserialize` (`:46-70`) rejects duplicates and caps the count at `MAX_ADDITIONAL_BLOCKS`, but the type has no canonical byte form: `to_bytes(from_bytes(b)) != b` across processes, and two nodes serialising the same request produce different bytes.

The harness found this on its own seed: the `download-request` seed (172 bytes, two additional blocks) decoded and re-serialised to 172 bytes with the two `HeaderId`s swapped:

```
thread '<unnamed>' panicked at fuzz/src/lib.rs:72:5:
non-canonical bincode: input is 172 bytes, re-serialization is 172 bytes
```

`network-wire-format.md` (Introduction) asks for "a stable encoding and decoding processes that do not depend on interpretation". Every other wire type in scope satisfies encode-after-decode identity; this is the only one that does not.

**Exploit scenario**

None today. The request is neither hashed, signed, deduplicated nor logged by its bytes; the provider only reads the set (`behaviour.rs:257,280`). The gap matters the moment a request identifier, a replay cache or a test vector is keyed on the encoded bytes, and it makes cross-implementation test vectors for the sync protocol impossible to pin.

**Recommendation**

- *Short term*: store `additional_blocks` as `BTreeSet<HeaderId>` (the deserializer already builds the set element by element, `:60-66`), or add a `Serialize` impl that sorts before writing, mirroring the custom `Deserialize`.
- *Long term*: add a `codec_fixtures!`-style golden vector for `RequestMessage` so the sync wire format is pinned like the `lb_codec` types are.

**References**: `network-wire-format.md` Introduction; harness `fuzz/src/lib.rs` (`check_bincode`).

### LB-002 · Sync frame allocation is bounded by the transport cap, not by the message type: a request stream may declare 16 MiB

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Denial of Service |
| Target | `consensus/cryptarchia-sync/src/libp2p/packing.rs:58-76` (`unpack_from_reader`); `libp2p/messages.rs:22-24,141-143` and `src/messages.rs:26-28` (`BoundedSerializeOp` impls) |
| Status | Open |

**Description**

The send side is bounded per type: `pack_to_writer` requires `Message: BoundedSerializeOp` and asserts at compile time that `Message::Bytes::MAX <= MAX_MSG_LEN` (`packing.rs:32-35`). The receive side is not: `unpack_from_reader` requires only `DeserializeOwned + Serialize` (`:60`), checks the peer-supplied length against the global `MAX_MSG_LEN` (`:67`) and then allocates it in full before any payload byte has arrived (`:73`, `vec![0u8; data_length]`). All three message types declare `Bytes = UpperBoundedVec<u8, MAX_MSG_LEN>`, so even if the receiver used the type's bound it would be 16 MiB for a `RequestMessage` whose largest legal encoding is 268 bytes (tag 4 + `target_block` 32 + `local_tip` 32 + `latest_immutable_block` 32 + count 8 + 5 × 32).

Checklist item 5 asked to fuzz this path because it is the only bincode path with a peer-supplied frame length. The fuzzer confirms the cap is enforced before allocation (every prefix above `MAX_MSG_LEN` is rejected with `MessageTooLarge`) and that the framed and direct decoders agree on every complete frame. The residual issue is the size of the cap relative to the message.

**Exploit scenario**

A peer opens up to `max_inbound_requests` (10 by default, `nodes/node/binary/src/config/network/serde/chainsync.rs:25`; streams beyond that are closed before reading, `behaviour.rs:285-300`) sync streams, writes the 4-byte prefix `ff ff ff 00` on each and stalls. The provider allocates ten 16 MiB zeroed buffers (160 MiB virtual; physical pages are committed only as bytes arrive) and holds them until the stream times out. On the downloader side the roles are reversed for responses, where 16 MiB is legitimate because a block body may reach 2 MiB and the envelope adds little. The impact is memory pressure proportional to the stream cap, not a crash; it is recorded as Low because the bound exists and is small.

**Recommendation**

- *Short term*: give `RequestMessage` a tight bound (for example `UpperBoundedVec<u8, 1024>`) and make `unpack_from_reader` generic over `Message: BoundedSerializeOp`, rejecting any prefix above `Message::Bytes::MAX` (the same constant `pack_to_writer` already uses). The provider then never allocates more than 1 KiB for a request.
- *Long term*: read into a bounded buffer incrementally (`take(len)` + `read_to_end` with a pre-checked capacity) so the allocation grows with bytes received rather than with the declared length.

**References**: #56 report LB-002 (unbounded bincode options), which this cap currently compensates for.

## 5. Suggestions (non-security)

### S-001 · Make the zero-length-element guard a compile-time property (checklist item 6)

`codec/src/bounded_vec.rs:129-137` refuses an element that decodes without consuming input only when `MAX > ZERO_LENGTH_ELEMENT_MAX` (1 024), only at runtime, and only after the first element has been decoded. The `TODO` at `:129-131` wants a compile-time check. At this commit the guard is never exercised by a real wire type: no `BinaryDecode` or bincode wire type contains a collection of zero-sized elements (`grep -rnE "Vec<\(\)>|Vec<PhantomData|HashSet<\(\)>|BoundedVec<\(\)|Vec<NoOpProof>"` over the workspace: no hits), and the bincode side has no equivalent guard at all.

A `const` assertion on `size_of::<T>()` is only a partial substitute, because "zero-sized" and "decodes without consuming input" are different properties (`[u8; 0]` is both; a struct whose decoder reads nothing but stores a default value is only the second). Two options, in increasing strength:

1. In `impl BinaryDecode for BoundedVec<T, MIN, MAX>`, add `const { assert!(size_of::<T>() > 0 || MAX <= ZERO_LENGTH_ELEMENT_MAX, "a zero-sized element under a large bound makes the decode loop input-independent") }` at the top of `decode`. This catches the `[u8; 0]` shape at compile time and keeps the runtime check for the rest.
2. Add `const MIN_ENCODED_LEN: usize` to `BinaryDecode` (0 for the `()`-like types, `N` for `[u8; N]`, the sum of fields for the derive) and assert `T::MIN_ENCODED_LEN > 0 || MAX <= ZERO_LENGTH_ELEMENT_MAX` in the same `const` block. This makes the property exact and removes the runtime branch. The derive macro at `codec/macros/src/lib.rs:48-155` already walks the fields, so it can emit the sum.

For the bincode side, a workspace test that walks the wire types is not expressible without reflection; the practical equivalent is a `deny`-level lint on `Vec<()>`/`Vec<PhantomData<_>>` in `#[derive(Deserialize)]` types, or the `with_limit` on `OPTIONS` recommended by the #56 report, which bounds the loop by bytes instead.

### S-002 · `Block::try_from(Bytes)` verifies the block twice

`core/src/block/mod.rs:371-377`: `Self::from_bytes(&bytes)?` goes through `Deserialize for Block<Tx>` (`:104-130`), which calls `reconstruct` (`:215-233`) and therefore `into_verified` (`:235-249`): header signature, transaction size total and body root. The result is then passed to `into_verified()` again (`:374`). Every synced block therefore costs two Ed25519 verifications and two `body_root` computations (a Blake2b over the uncle headers plus a Merkle root over up to 1 024 transaction hashes). Drop the second call, or make `try_from` construct the block without going through `Deserialize`.

### S-003 · The `SignedOps` binary serde bridge reads an unbounded byte string while `Ops` bounds its own

`core/src/mantle/transactions/tx_list/signed_ops.rs:354-355` reads `Vec<u8>` and calls `decode_all`, so trailing bytes are rejected (the #56 report's LB-001 concern no longer applies to the transaction type). `Ops` at `ops.rs:211` reads the same kind of byte string through `deserialize_bounded_bytes::<MAX_BLOCK_TRANSACTIONS_SIZE, _>`. With bincode's slice reader the difference is harmless, but the two bridges should use the same bounded helper so that the bound is visible on the type that is actually on the wire.

### S-004 · Expose the encode side of `EncapsulatedMessage` outside `cfg(test)` for tooling

`blend/message/src/encap/encapsulated.rs:126-140` gates `impl BinaryEncode for EncapsulatedMessage` behind `#[cfg(test)]` so that unverified messages cannot be sent. That is the right production default, but it means a harness or a test vector tool cannot assert canonicality on the whole message and has to re-encode the components (`fuzz/fuzz_targets/blend_message.rs`). A `test-utils` or `fuzzing` feature (the crate already has `unsafe-test-functions`) would keep the production restriction and unblock tooling.

### S-005 · Specification: two documents still carry superseded constants

- `overview-cryptoeconomics.md` (Fee Markets, Permanent Storage Fee Market) states "Blocks are limited to 1MiB"; `cryptarchia-v1-protocol.md` (Constants) and `bedrock-v1.1-block-construction.md` (revision 1.1.3) set `MAX_BLOCK_SIZE` to 2 MiB, and the code agrees (`core/src/block/mod.rs:31`, `MAX_BLOCK_TRANSACTIONS_SIZE = 2 MiB`). The overview should read the constant from the protocol document rather than restate it.
- `message-encapsulation.md` (Message Structure) declares `PAYLOAD_BODY_SIZE = 34 * 1024`; `payload-formatting.md` (revision 1.1.1) sets `Max_Body_Length = 18192` and the code agrees (`blend/message/src/message/payload.rs:18`, `MAX_PAYLOAD_BODY_SIZE = 18_192`). The encapsulation document should reference `Max_Body_Length` instead of carrying its own value.

### S-006 · Ship the harness

The crate in Appendix C is self-contained: add it as a workspace member (one line in `Cargo.toml`), apply the three visibility changes listed in Method, and run `cargo run -p logos-blockchain-fuzz --example seed` once to populate `fuzz/corpus/`. A nightly CI job running each target for a few minutes over the checked-in corpus would keep the "no panic, canonical, consumed-length agreement" invariants enforced as the codecs change. The seed generator should be re-run whenever a `codec_fixtures!` vector changes.

---

## Appendix B — What was checked and ruled out

| Property (from #69 / #10) | Result | Evidence |
|---|---|---|
| Panics reachable from any of the seven ingress decoders | None found. | Campaign table in Section 4; ASan and `-Cdebug-assertions -Coverflow-checks` were on, so overflow, slice indexing and `unwrap` paths would all have reported. |
| Canonicality of every `lb_codec` type on the wire (`Proposal`, `SignedOps`, `EncapsulatedMessage`, `ChannelInscription`) | Holds: for every accepted input, `encode(decode(b)) == b`; for every accepted strict prefix, `encode(decode(b)) == consumed prefix`. | `fuzz/src/lib.rs` `check_binary_codec`; no assertion fired. |
| `decode` and `decode_all` agree on consumed length | Holds for all four `lb_codec` types. | Same helper; `Proposal::decode_all` at `codec/src/lib.rs:89-95` is the shared default. |
| bincode canonicality of block, `DownloadBlocksResponse`, `GetTipResponse` | Holds. `RequestMessage` does not, see LB-001. | `check_bincode` in `fuzz/src/lib.rs`. |
| `unpack_from_reader` frame cap enforced before allocation | Holds; the fuzzer never got the reader to allocate above `MAX_MSG_LEN`, and framed vs. direct decoders agreed on every complete frame. Cap is coarse, see LB-002. | `packing.rs:63-73`; `fuzz/fuzz_targets/sync_messages.rs` `check_framed`. |
| `Preverified` transaction and block decoders accept only what `Unverified` accepts | Holds (asserted per input in `signed_ops_gossip` and `block`). | `core/src/mantle/transactions/tx_list/signed_ops.rs:360-371` runs `preverify()` after the `Unverified` decode. |
| Blend decode independent of the layer count where it should be | Holds: the public header (`PublicHeader::decode`) and payload parse identically for 1 and 3 layers; only the private header consumes `layers × BLENDING_HEADER_ENCODED_SIZE`. | `blend/message/src/encap/encapsulated.rs:142-158,618-632`. |
| Zero-sized-element collections in any wire type | None at this commit. | `grep -rnE "Vec<\(\)>|Vec<PhantomData|HashSet<\(\)>|BoundedVec<\(\)|Vec<NoOpProof>"` over the workspace: no hits. Runtime guard tests pass (`codec/src/bounded_vec.rs:340-397`, 35/35). |
| Trailing bytes at every ingress | Rejected everywhere fuzzed: `decode_all` (proposal, inscription body), `deserialize_encapsulated_message` (`codec.rs:45-48`), `SignedOps::decode_all` inside the serde bridge (`signed_ops.rs:355`), bincode `reject_trailing_bytes` for block and sync messages (`core/src/codec/bincode/mod.rs:28`). | Harness invariants; the `(Err, Ok(rest non-empty))` arm of `check_binary_codec` was exercised and never contradicted. |
| Seeds from `codec_fixtures!` decode under the harness invariants | All 27 seeds pass `-runs=0` on every target after LB-001 was accounted for. | Section 3. |
| Sync inbound streams above `max_inbound_requests` | Closed before any read, so LB-002 is bounded by that setting. | `consensus/cryptarchia-sync/src/libp2p/behaviour.rs:286-300`. |
| `Groth16LeaderProof` and `PoLProof` decoders | Copy bytes without curve parsing (`leader_proof.rs:68-90`, `PoLProof::from_bytes`), so no panic surface and canonical by construction; `Fr` fields reject values ≥ the modulus (`codec/src/numbers.rs`, `zk/groth16/src/serde.rs:20-24`). | Read; covered by the `proposal` and `block` targets. |

## Appendix C — Harness source

Files are relative to the `logos-blockchain` root. Add `"fuzz"` to `[workspace] members` in the root `Cargo.toml`, and apply the three visibility changes listed in Method.

`fuzz/Cargo.toml`

```toml
[package]
name    = "logos-blockchain-fuzz"
version = "0.0.0"
edition = "2024"
publish = false

[package.metadata]
cargo-fuzz = true

[dependencies]
bytes                         = { features = ["serde"], workspace = true }
futures                       = { features = ["executor", "std"], workspace = true }
lb-blend-message              = { workspace = true }
lb-codec                      = { workspace = true }
lb-core                       = { workspace = true }
lb-cryptarchia-engine         = { workspace = true }
lb-cryptarchia-sync           = { workspace = true }
lb-key-management-system-keys = { workspace = true }
lb-utils                      = { workspace = true }
libfuzzer-sys                 = "0.4"
logos-sql                     = { workspace = true }
serde                         = { features = ["derive"], workspace = true }

# One [[bin]] per target, all with test = false, doc = false, bench = false:
# proposal, signed_ops_codec, signed_ops_gossip, blend_message,
# channel_inscription, block, sync_messages (path = "fuzz_targets/<name>.rs").
```

`fuzz/src/lib.rs`

```rust
use lb_codec::{BinaryDecode, BinaryEncode};
use lb_core::codec::{DeserializeOp, SerializeOp};

/// Runs the `lb_codec` invariants for `T` on `data`. Returns the decoded value
/// when `decode_all` accepted the whole input.
pub fn check_binary_codec<T>(data: &[u8], context: &T::Context) -> Option<T>
where
    T: BinaryEncode + BinaryDecode,
{
    let all = T::decode_all(data, context);
    let partial = T::decode(data, context);

    match (all, partial) {
        (Ok(value), Ok((rest, prefix_value))) => {
            assert!(rest.is_empty(), "decode_all accepted the input but decode left {} bytes", rest.len());
            let encoded = value.encode();
            assert!(&*encoded == data, "non-canonical: input is {} bytes, re-encoding is {} bytes", data.len(), encoded.len());
            assert!(&*prefix_value.encode() == data, "decode and decode_all produced values with different encodings");
            Some(value)
        }
        (Err(_), Ok((rest, prefix_value))) => {
            assert!(!rest.is_empty(), "decode_all rejected the input but decode consumed all of it");
            let consumed = &data[..data.len() - rest.len()];
            let encoded = prefix_value.encode();
            assert!(&*encoded == consumed, "non-canonical prefix: consumed {} bytes, re-encoding is {} bytes", consumed.len(), encoded.len());
            None
        }
        (Ok(_), Err(error)) => panic!("decode_all accepted the input but decode failed: {error:?}"),
        (Err(_), Err(_)) => None,
    }
}

/// Runs the bincode (`DeserializeOp`) invariants for `T` on `data`.
pub fn check_bincode<T>(data: &[u8]) -> Option<T>
where
    T: SerializeOp + DeserializeOp,
{
    let value = T::from_bytes(data).ok()?;
    let encoded = value.to_bytes().expect("a value that was deserialized must serialize");
    assert!(encoded.as_ref() == data, "non-canonical bincode: input is {} bytes, re-serialization is {} bytes", data.len(), encoded.len());
    Some(value)
}

/// Like `check_bincode`, for types whose encoding is legitimately not
/// byte-stable across processes (a `HashSet` field serializes in
/// `RandomState` order). Requires the re-serialization to have the input's
/// length and to decode again.
pub fn check_bincode_reorderable<T>(data: &[u8]) -> Option<T>
where
    T: SerializeOp + DeserializeOp,
{
    let value = T::from_bytes(data).ok()?;
    let encoded = value.to_bytes().expect("a value that was deserialized must serialize");
    assert_eq!(encoded.len(), data.len(), "re-serialization changed the length");
    assert!(T::from_bytes(&encoded).is_ok(), "re-serialization does not decode");
    Some(value)
}
```

`fuzz/fuzz_targets/proposal.rs`

```rust
#![no_main]
use lb_core::block::Proposal;
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    logos_blockchain_fuzz::check_binary_codec::<Proposal>(data, &());
});
```

`fuzz/fuzz_targets/signed_ops_codec.rs`

```rust
#![no_main]
use lb_core::mantle::{SignedOps, ledger::verification_mode::StandardMode, transactions::states::Unverified};
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    logos_blockchain_fuzz::check_binary_codec::<SignedOps<Unverified, StandardMode>>(data, &());
});
```

`fuzz/fuzz_targets/signed_ops_gossip.rs`

```rust
#![no_main]
use lb_core::mantle::{SignedOps, ledger::verification_mode::StandardMode, transactions::states::{Preverified, Unverified}};
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    let unverified = logos_blockchain_fuzz::check_bincode::<SignedOps<Unverified, StandardMode>>(data);
    let preverified = logos_blockchain_fuzz::check_bincode::<SignedOps<Preverified, StandardMode>>(data);
    if preverified.is_some() {
        assert!(unverified.is_some(), "preverified decode accepted bytes the unverified decode rejected");
    }
});
```

`fuzz/fuzz_targets/blend_message.rs`

```rust
#![no_main]
use core::num::NonZeroU64;
use lb_blend_message::{deserialize_encapsulated_message, encap::encapsulated::EncapsulatedMessage};
use lb_codec::{BinaryDecode as _, BinaryEncode as _};
use libfuzzer_sys::fuzz_target;

// `EncapsulatedMessage: BinaryEncode` is cfg(test)-only; re-encode the parts.
fn reencode(message: EncapsulatedMessage) -> Vec<u8> {
    let (public_header, part) = message.into_components();
    let mut out = public_header.encode_to_vec();
    part.encode_into(&mut out);
    out
}

fuzz_target!(|data: &[u8]| {
    for layers in [1u64, 3] {
        let layers = NonZeroU64::new(layers).expect("non-zero");
        let via_ingress = deserialize_encapsulated_message(data, &layers);
        let partial = EncapsulatedMessage::decode(data, &layers);
        match (via_ingress, partial) {
            (Ok(message), Ok((rest, _))) => {
                assert!(rest.is_empty());
                assert!(reencode(message) == data, "non-canonical");
            }
            (Err(_), Ok((rest, message))) => {
                assert!(!rest.is_empty());
                let consumed = &data[..data.len() - rest.len()];
                assert!(reencode(message) == consumed, "non-canonical prefix");
            }
            (Ok(_), Err(error)) => panic!("ingress accepted the input but decode failed: {error:?}"),
            (Err(_), Err(_)) => {}
        }
    }
});
```

`fuzz/fuzz_targets/channel_inscription.rs`

```rust
#![no_main]
use libfuzzer_sys::fuzz_target;
use logos_sql::protocol::ChannelInscription;

const PAYLOAD_HEADER_LEN: usize = 9 + 2; // marker + version

fuzz_target!(|data: &[u8]| {
    if let Ok(value) = ChannelInscription::decode(data) {
        let encoded = value.encode().expect("a decoded payload must encode");
        assert!(encoded == data, "non-canonical");
    }
    if let Some(body) = data.get(PAYLOAD_HEADER_LEN..) {
        logos_blockchain_fuzz::check_binary_codec::<ChannelInscription>(body, &());
    }
});
```

`fuzz/fuzz_targets/block.rs`

```rust
#![no_main]
use bytes::Bytes;
use lb_core::{block::Block, codec::SerializeOp as _, mantle::{SignedOps, ledger::verification_mode::StandardMode, transactions::states::{Preverified, Unverified}}};
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    let bytes = Bytes::copy_from_slice(data);
    let unverified = Block::<SignedOps<Unverified, StandardMode>>::try_from(bytes.clone());
    if let Ok(block) = &unverified {
        let encoded = block.to_bytes().expect("a decoded block must serialize");
        assert!(encoded.as_ref() == data, "non-canonical block");
    }
    let preverified = Block::<SignedOps<Preverified, StandardMode>>::try_from(bytes);
    if preverified.is_ok() {
        assert!(unverified.is_ok(), "preverified block decode accepted bytes the unverified decode rejected");
    }
});
```

`fuzz/fuzz_targets/sync_messages.rs`

```rust
#![no_main]
use futures::{executor::block_on, io::Cursor};
use lb_cryptarchia_sync::{GetTipResponse, libp2p::{messages::{DownloadBlocksResponse, RequestMessage}, packing::unpack_from_reader}};
use libfuzzer_sys::fuzz_target;
use logos_blockchain_fuzz::{check_bincode, check_bincode_reorderable};
use serde::{Serialize, de::DeserializeOwned};

const LEN_PREFIX: usize = 4;

fn check_framed<Message>(data: &[u8], direct: impl Fn(&[u8]) -> Option<Message>)
where
    Message: DeserializeOwned + Serialize,
{
    let framed = block_on(unpack_from_reader::<Message, _>(&mut Cursor::new(data)));
    if let Some(len) = data.get(..LEN_PREFIX).map(|p| u32::from_le_bytes(p.try_into().expect("4 bytes")) as usize)
        && let Some(payload) = data.get(LEN_PREFIX..LEN_PREFIX + len)
    {
        assert_eq!(framed.is_ok(), direct(payload).is_some(), "framed and direct decoders disagree on a complete frame");
    } else {
        assert!(framed.is_err(), "an incomplete frame must be rejected");
    }
}

fuzz_target!(|data: &[u8]| {
    check_bincode_reorderable::<RequestMessage>(data);
    check_bincode::<DownloadBlocksResponse>(data);
    check_bincode::<GetTipResponse>(data);
    check_framed::<RequestMessage>(data, check_bincode_reorderable::<RequestMessage>);
    check_framed::<DownloadBlocksResponse>(data, check_bincode::<DownloadBlocksResponse>);
    check_framed::<GetTipResponse>(data, check_bincode::<GetTipResponse>);
});
```

`fuzz/examples/seed.rs` (abridged: the full file also builds the seven sync message shapes)

```rust
use lb_codec::{BinaryEncode as _, CodecExamples as _};
// ... imports of Proposal, UncleHeaders, Header, SignedOps, Ops, EncapsulatedMessage,
//     ChannelInscription, BoundedVec, Ed25519Signature, sync message types ...

type Tx = SignedOps<Unverified, StandardMode>;

/// Mirrors the field order of `lb_core::block::Block`, whose constructor
/// demands a valid signature. The seed only needs the shape.
#[derive(Serialize)]
struct SeedBlock { header: Header, signature: Ed25519Signature, uncle_headers: UncleHeaders, transactions: BoundedVec<Tx, 0, 1024> }

fn write(target: &str, name: &str, bytes: &[u8]) { /* fuzz/corpus/<target>/<name> */ }
fn framed(payload: &[u8]) -> Vec<u8> { /* u32 LE length prefix ++ payload */ }

fn main() {
    for (i, f) in Proposal::fixtures().into_iter().enumerate() { write("proposal", &format!("fixture-{i}"), &f.bytes); }
    let p = Proposal::fixtures().into_iter().next().unwrap().value;
    let mut no_uncles = p.clone(); no_uncles.uncle_headers = UncleHeaders::empty();
    write("proposal", "no-uncles", &no_uncles.encode());
    for (i, f) in Tx::fixtures().into_iter().enumerate() {
        write("signed_ops_codec", &format!("fixture-{i}"), &f.bytes);
        write("signed_ops_gossip", &format!("fixture-{i}"), &f.value.to_bytes().unwrap());
    }
    for (i, f) in Ops::fixtures().into_iter().enumerate() { write("signed_ops_codec", &format!("ops-{i}"), &f.bytes); }
    for (i, f) in EncapsulatedMessage::fixtures().into_iter().enumerate() { write("blend_message", &format!("fixture-{i}"), &f.bytes); }
    for (i, f) in ChannelInscription::fixtures().into_iter().enumerate() { write("channel_inscription", &format!("fixture-{i}"), &f.value.encode().unwrap()); }
    let block = SeedBlock { header: p.header.clone(), signature: p.signature.clone(), uncle_headers: p.uncle_headers.clone(),
        transactions: BoundedVec::try_from(Tx::fixtures().into_iter().map(|f| f.value).collect::<Vec<_>>()).unwrap() };
    write("block", "proposal-header-with-fixture-txs", &block.to_bytes().unwrap());
    // sync_messages: GetTip, DownloadBlocksRequest, Block/NoMoreBlocks/Failure responses,
    // GetTipResponse::{Tip, Failure}; each written raw and framed.
}
```

Campaign driver (`run-campaign.sh`): for each target, `cargo +nightly fuzz run <target> -- -max_total_time=3600 -max_len=<per target> -timeout=25 -rss_limit_mb=4096 -print_final_stats=1`, all seven in parallel.

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
