# Audit Report · Time source, backwards jumps, startup before genesis, long offline periods

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/46` (parent `#5`)
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `c4c86be18c58b5b09c3650e93871c8cfb624885b` · component(s): `services/time` (`lib.rs`, `backends/common.rs`, `backends/system_time.rs`, `backends/ntp/{mod.rs,async_client.rs}`), `consensus/cryptarchia-engine/src/time.rs`, `services/chain/chain-service` (`lib.rs`, `states.rs`, `bootstrap/state.rs`, `service/phases/*`), `services/chain/chain-network/src/{lib.rs,sync/tip_poll.rs}`, `services/chain/chain-leader/src/{lib.rs,leadership.rs}`, `nodes/node/binary/src/config/time`, third-party `sntpc 0.5.2`
Specs: `https://github.com/logos-co/logos-lips` @ `d788723992a805b395f377de6e6cf59859b47168` · read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `cryptarchia-proof-of-leadership.md`, `bedrock-anonymous-leaders-reward.md` (in full); `cryptarchia-v1-protocol.md`, `bedrock-v1.1-block-construction.md`, `cryptarchia-v1-bootstr-sync.md` (by section, listed in §3)
Date: `2026-09-25` · author: `Claude Code (research agent)` · status: `final`

---

## 1. Summary

- Overall assessment: the slot clock is NTP-derived and runs on tokio's monotonic timers, never reads peer-provided time, and handles startup before genesis correctly; its weakness is the correction policy. `NtpStream` adopts any forward NTP step at once and answers any backward step by freezing the tick stream until real time catches up, so the node's current slot is the most advanced time it has ever been told, and one bad sample (a host clock that is ahead at start, a single spoofed or unsynchronised NTP reply, or a host clock step during an exchange) moves the node ahead by that much and silences every slot consumer for as long. Each case was reproduced in unit tests on the audited commit (Appendix B).
- Findings: `0` critical · `0` high · `2` medium · `3` low · `0` informational
- Key themes: "the slot clock ratchets forward and freezes on backward correction", "one unauthenticated NTP reply is trusted with no validation or bound", "NTP failure is silent and can be total", "three clocks: NTP for slots, the host clock for the genesis wait and the offline grace period, the monotonic clock for timers".
- Must-fix before launch: LB-001 and LB-002 (bound and validate NTP samples against each other and against the monotonic clock, slew rather than freeze on backward correction, and derive every tick from the clock instead of a counter).

Answers to the two checklist items:

1. **Time source and backwards jumps.** The node binary always uses `NtpTimeBackend` (`nodes/node/binary/src/config/time/mod.rs:L23-L59`), default server `pool.ntp.org:123`, re-queried every 15 s (`config/time/serde.rs:L29-L37`). The slot timer is a `tokio::time::interval` (monotonic `Instant`), so a host clock jump between NTP rounds does not move it; only the NTP rebase and the initial value read wall time. `sntpc`'s `sec()` is Unix-epoch seconds: the server's transmit timestamp minus the NTP era offset `2_208_988_800`, truncated to `u32` (sntpc `types.rs:L125-L131`, `lib.rs:L918-L919`), correct until 2106 in release builds. `handle_ntp_update` adds `roundtrip / 2`, the standard estimate of the time at receipt. Backward corrections pause emission; this is LB-001, and its interaction with the tick counter of #38 LB-002 (#454) is LB-003. No peer-provided time is consumed: block slots are only compared against the local tick (`chain-service/src/lib.rs:L441`), peer tip slots are used only to pick which tip to download (`chain-network/src/sync/tip_poll.rs:L105-L131`), and the SDP `timestamp` field is a block number (`core/src/sdp/mod.rs:L45`).
2. **Startup before genesis; long offline.** Before genesis the time service reports slot 0, epoch 0, and the first tick is slot 1 at `genesis + slot_duration` (EXP-5); every block is a `FutureBlock` until then. The chain service waits in `AwaitingGenesisTime`, but that phase reads the host clock, not the NTP-derived slot clock (LB-005), and its rejections are still recorded as permanent verdicts (#143 LB-003, filed as #556, re-verified unchanged). After a long offline period the time service needs no catch-up (the slot is computed from the clock), the ledger folds skipped epochs in a loop (`ledger/src/cryptarchia/mod.rs:L376-L395`), and the offline grace period forces the Bootstrap rule; but the grace period is measured on the host clock, which a host without a battery-backed clock resets to its last saved time at boot, so a node offline for days can restart Online (LB-005).

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/time/src/lib.rs` | service loop, `watch` broadcast of ticks, `CurrentSlot` |
| `services/time/src/backends/common.rs`, `system_time.rs` | tick counter construction, initial slot |
| `services/time/src/backends/ntp/mod.rs`, `async_client.rs` | NTP query, `handle_ntp_update`, `poll_slot_timer`, error handling |
| `consensus/cryptarchia-engine/src/time.rs:L161-L179, L300-L341` | `Slot::from_offset_and_config`, `SlotTimer::slot_interval` |
| `services/chain/chain-service/src/lib.rs:L425-L446, L700-L873` | future-block check, how the chain service obtains its slot at start |
| `services/chain/chain-service/src/service/phases/{awaiting_genesis_time,ibd,pbp,following}.rs` | genesis wait, tick consumption, Prolonged Bootstrap timer |
| `services/chain/chain-service/src/bootstrap/{state.rs,config.rs}`, `states.rs:L44-L72` | offline grace period and its timestamp |
| `services/chain/chain-network/src/lib.rs:L78-L79, L660-L690, L950-L1030`, `sync/tip_poll.rs` | what happens to future blocks; which peer data is used |
| `services/chain/chain-leader/src/lib.rs:L420-L523`, `leadership.rs:L416-L463` | leader slot loop and its epoch context |
| `nodes/node/binary/src/config/time/*` | which backend and which defaults the binary uses |
| `sntpc 0.5.2` (`src/lib.rs`, `src/types.rs`) | response validation, `sec()`, roundtrip computation |

**Out of scope**

- Proposal equivocation, proof generation cost and KMS handling (other sub-issues of #5); Blend's own session logic beyond noting that it consumes the same ticks; fork choice internals (#39, #592).
- Third-party crates assumed correct except where read above: `tokio` (interval and `Instant` semantics as documented), `time`, `sntpc` beyond the functions quoted, `overwatch`.

**Assumptions**

- The specs at the commit above are the reference; `cryptarchia-v1-protocol.md` §Clocks says the protocol relies on NTP, so an attacker who can answer NTP queries is inside the threat model this report evaluates, not a precondition that defeats it.
- Repo-level facts from issue #19: `[profile.release]` sets no `overflow-checks`, which matters only for the `sntpc` era arithmetic noted in item 1.

## 3. Method

- Manual review of the in-scope paths, working through issue `#46` (both items) with parent `#5`, the lead in the issue comment (from #38 LB-002, PR #170), and the prior reports `processed/38-slot-epoch-arithmetic.md` (LB-002, S-003), `processed/42-long-range-bootstrap-restart.md` (LB-004, S-001), `processed/143-rejected-cache-insertion-sites.md` (LB-003), `processed/204-config-silent-defaults.md` (LB-002, S-004), `processed/116-epoch-bound-core-connections.md` (clock-offset analysis), `inbox/634-note-ageing-block-free-windows.md` (LB-001, halted-chain restart) and `inbox/644-bootstrapping-nodes-cold-restart-and-serving.md`, so that this report cites rather than repeats them.
- Specifications read: in full, `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `cryptarchia-proof-of-leadership.md`, `bedrock-anonymous-leaders-reward.md`. By section: `cryptarchia-v1-protocol.md` §Protocol/Constants, §Notation, §Latest Immutable Block, §Slot, §Epoch (intro and §Epoch Schedule, §Epoch State), §Leadership Lottery, §Block Header Validation, §Chain Maintenance, §Commit, §Versioning and Protocol Upgrades, §Annexes (§Proof of Stake vs. Proof of Work, §Clocks, §References); `bedrock-v1.1-block-construction.md` §Block Proposal, §Header, §Proposal Construction, §Block Proposal Reconstruction, §Block Proposal Validation, §Block Execution; `cryptarchia-v1-bootstr-sync.md` §Constants, §Setting the Fork Choice Rule, §Initial Block Download, §Prolonged Bootstrap Period, §Proposing New Blocks, §Bootstrapping from Checkpoint, §Offline Grace Period, §Offline Duration Measurement.
- Automated tooling: `rustc 1.98.1`, `cargo 1.98.1` (pinned toolchain). Baseline on the unmodified crate: `cargo test --release -p logos-blockchain-time-service --features ntp --lib -- --skip real`: 9 passed.
- Dynamic testing: seven unit tests added to `services/time/src/backends/ntp/mod.rs` in a scratch clone (module `exp46`, Appendix B), built with `--release` into the shared target dir. EXP-1, EXP-3 and EXP-4 use a minimal NTP server on `127.0.0.1` that answers with a chosen time, leap indicator 3 and stratum 16. EXP-2, EXP-4 and EXP-5 use `tokio` paused time; EXP-1, EXP-3, EXP-6 and EXP-7 run in real time. Output lines are quoted verbatim in the findings. No node binary or devnet was run (see the follow-up in the tracker).

### Checked and ruled out

- **Monotonic slot timer.** `SlotTimer::slot_interval` builds `tokio::time::interval_at(Instant::now() + delay, slot_duration)` (`time.rs:L329-L341`); `Instant` is monotonic, so a host clock step does not move the tick schedule. The Prolonged Bootstrap timer (`pbp.rs:L73`) and the genesis wait (`awaiting_genesis_time.rs:L177`) are `tokio::time::sleep`, also monotonic. `CLOCK_MONOTONIC` does not advance while the host is suspended; the resulting lag is the "process suspension" case of #38 LB-002 (#454) and is corrected by the next successful NTP round.
- **Duplicate or regressing ticks.** `poll_slot_timer` suppresses any tick `<= last_emitted_slot` (`ntp/mod.rs:L225-L227`) and the system backend's counter only increments (`common.rs:L25-L30`), so no consumer ever sees the same slot twice or a smaller slot. A leader cannot be driven to run the same slot twice by the time source.
- **Peer-provided time.** Covered in §1 item 1. The chain service's `FutureBlock` check (`chain-service/src/lib.rs:L440-L446`) uses its own last tick; chain-network retries it `3 x 500 ms` and then drops the block without recording it (`chain-network/src/lib.rs:L78-L79, L674-L681, L955-L976`); tip-poll uses the local tick for its lag test (`tip_poll.rs:L39-L45, L91-L95`).
- **`sntpc` parsing.** The response is checked for origin-timestamp echo, mode 4/5, version, and stratum `!= 0` (sntpc `lib.rs:L869-L893`), and the source address must match (`lib.rs:L569-L571`). `sec()` is Unix seconds (above). The era subtraction `(v >> 32) - 2_208_988_800` underflows after 2036-02-07; in release it wraps and the `as u32` cast yields the correct Unix time modulo `2^32`, so the node is correct until 2106; debug and test builds would panic in sntpc after 2036. `Duration` arithmetic in `handle_ntp_update` cannot overflow (`sec()` is `u32`) and the `i128`/`OffsetDateTime` conversions are checked (`ntp/mod.rs:L161-L182`), as #29's second pass also found.
- **Fractional slot durations.** Re-verified unchanged from #38 LB-002: `from_offset_and_config` divides whole seconds by `slot_duration.as_secs()` (`time.rs:L173-L177`) while the interval ticks at the full duration; every shipped deployment uses `slot_duration: '1.000000000'` (`nodes/node/binary/src/config/deployment/settings.yaml:L127`, `deployment/ceremony/genesis/*/deployment-template.yaml:L109`). Not re-filed.
- **Startup before genesis (time service).** `from_offset_and_config` clamps negative offsets to slot 0 (`time.rs:L166-L170`); `slot_interval` then delays the first tick to `genesis + 1 slot` (EXP-5: `initial tick before genesis: SlotTick { epoch: Epoch(0), slot: Slot(0) }`, `first tick SlotTick { epoch: Epoch(0), slot: Slot(1) } after 11s of virtual time` for a genesis 10 s ahead). No panic path: `delay` is always positive because the next slot start is strictly after `now`.
- **Many skipped epochs.** When a block (or a leader query) lands several epochs after the parent, `update_epoch_state` runs the zero-density stake inference and the storage-market update once per skipped epoch in a `for` loop (`ledger/src/cryptarchia/mod.rs:L376-L380, L393-L395`), not by recursion; EXP-4's jump of 60,772 epochs is therefore cheap to evaluate. The economic consequence of empty inference windows (the inferred stake collapsing to 1) is #634 LB-001 and is not repeated here.
- **Long offline, time side.** After a restart the time service recomputes the slot from the clock; there is no stored slot to catch up from. Replay of stored blocks uses the chain service's first slot reading, which is taken from the host clock before the first NTP round is applied (`chain-service/src/lib.rs:L862-L870`, `ntp/mod.rs:L82-L89`); a host clock behind the stored tip makes replay fail with `FutureBlock`, which is #42 LB-004 (#447), re-verified.
- **Long offline, proposing.** The leader waits for the Online rule (`chain-leader/src/lib.rs:L434-L439`) and does not otherwise check that its tip is recent (`leadership.rs:L416-L463`). With IBD peers configured, IBD brings the tip up to the peers' tips before the Prolonged Bootstrap Period; without them, the node goes Online on whatever gossip delivered, which is #204 LB-002 (#689). See S-003.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | The slot clock ratchets to the most advanced time it has seen: a forward NTP step is adopted at once, a backward one freezes every slot consumer for the whole excursion | Timing | Medium | Medium | Open |
| LB-002 | One NTP reply from whichever resolved address answers first sets the consensus clock, with no check of its synchronisation status, no second source and no bound | Timing | Medium | High | Open |
| LB-003 | A tick suppressed during a backward-correction pause returns `Pending` without a registered wake-up; the pause outlasts the correction and, with NTP failing, leaves a permanent lag | Timing | Low | Medium | Open |
| LB-004 | An NTP round fails as a whole when any resolved address fails first, and a failing NTP is silent: the node runs on the unsynchronised host clock indefinitely | Error Reporting | Low | Low | Open |
| LB-005 | The genesis wait and the offline grace period read the host clock, not the node's slot clock | Timing | Low | High | Open |

### LB-001 · The slot clock ratchets to the most advanced time it has seen: a forward NTP step is adopted at once, a backward one freezes every slot consumer for the whole excursion

| | |
|---|---|
| Severity | Medium |
| Difficulty | Medium |
| Category | Timing |
| Target | `services/time/src/backends/ntp/mod.rs:L195-L213` (`handle_ntp_update`), `:L220-L233` (`poll_slot_timer`), `:L82-L99` (`tick_stream`, initial slot from the host clock) |
| Status | Open |

**Description**

`NtpStream` keeps `last_emitted_slot` and rebases its timer on every NTP reply. A reply that computes a slot below `last_emitted_slot` only swaps the timer; `last_emitted_slot` stays where it was, and `poll_slot_timer` suppresses every tick up to it:

```rust
// services/time/src/backends/ntp/mod.rs:195-213
if current_slot < this.last_emitted_slot {
    tracing::warn!(target: LOG_TARGET, "NTP resync moved backwards: ...; pausing until NTP catches up", ...);
    this.slot_timer = new_slot_timer;
    return;
}
...
this.slot_timer = new_slot_timer;
this.last_emitted_slot = current_slot;          // forward: adopted immediately

// :225-227
if tick.slot <= this.last_emitted_slot {
    return Poll::Pending;                        // backward: silence until caught up
}
```

The effective clock is therefore `max` over every time sample the process has received, plus elapsed time. An error that puts a sample `D` seconds ahead cannot be corrected by later, better samples; it can only be waited out, for `D` seconds. During those seconds:

- no `SlotTick` is published on the `watch` channel (`services/time/src/lib.rs:L151-L160`), so every subscriber stalls: the leader loop (`chain-leader/src/lib.rs:L451`) runs no lottery and proposes nothing, tip-poll (`chain-network/src/lib.rs:L499-L510`) never fires, Blend membership (`services/blend/src/membership/chain.rs:L124`) and the PoW service (`services/pow/src/service.rs:L681`) do not advance;
- the chain service's `current_slot` stays at the inflated value (`service/phases/following.rs:L66`), so the future-block check (`chain-service/src/lib.rs:L441`) keeps accepting blocks up to `D` slots ahead of real time for the whole pause.

The first sample the node ever uses is the host clock (`ntp/mod.rs:L82-L89`, `last_emitted_slot: current_slot_tick.slot` at `:L98`). The NTP backend exists because the host clock may be wrong, yet a host clock that is ahead is locked in by the first NTP round. EXP-2 reproduces the construction of `tick_stream` with a host clock one hour fast:

```
EXP-2 initial slot Slot(1800003600) (system clock), NTP slot Slot(1800000000); first tick Slot(1800003601) after 3601 s of silence
```

The warning at `:L196-L201` says "pausing until NTP catches up", but the pause is as long as the error, with no bound and no metric.

**Exploit scenario**

Benign trigger: a node is started on a host whose clock is 20 minutes fast (a container on a drifting host, a VM restored from a snapshot, a board without an RTC after a manual date set). For 20 minutes after every start it proposes nothing, its tip-poll is off, Blend membership does not advance, and it accepts blocks up to 1,200 slots in the future. Adversarial trigger: any party that can put one wrong sample in front of the node (LB-002) chooses `D`; an adversary with stake can use the open future window to feed blocks for slots honest nodes have not reached, which the node accepts and builds on (quantified under LB-002). The protocol assumption that honest clocks are "relatively in sync" (`cryptarchia-v1-protocol.md` §Clocks) is exactly what this policy breaks for the affected node.

**Recommendation**

- *Short term*: keep the emitted slot monotonic but do not let `last_emitted_slot` run ahead of the best current estimate: on a backward correction larger than one slot, rebase `last_emitted_slot` to the corrected slot and accept that the next tick repeats or regresses by the correction (consumers already handle non-consecutive slots), or slew (emit ticks slightly slower) for corrections below a bound; export a gauge with the current offset between the emitted slot and the NTP slot. Do not seed `last_emitted_slot` from the host clock: emit nothing until the first NTP round or its timeout, then start from the chosen source.
- *Long term*: make the time service a single clock estimate (`monotonic_now + offset`) whose offset is filtered over several samples (LB-002), and derive each tick from it rather than from a counter (#38 LB-002, #454), so neither freezes nor lags are possible by construction.

**References**: `cryptarchia-v1-protocol.md` §Block Header Validation rule 6, §Clocks; #38 LB-002 (#454).

### LB-002 · One NTP reply from whichever resolved address answers first sets the consensus clock, with no check of its synchronisation status, no second source and no bound

| | |
|---|---|
| Severity | Medium |
| Difficulty | High |
| Category | Timing |
| Target | `services/time/src/backends/ntp/async_client.rs:L65-L78` (`request_timestamp`), `services/time/src/backends/ntp/mod.rs:L148-L213` (`handle_ntp_update`), `nodes/node/binary/src/config/time/serde.rs:L29-L37` |
| Status | Open |

**Description**

Every 15 s the backend resolves the configured name, sends one SNTP request to every address it resolves to, and takes the first reply:

```rust
// services/time/src/backends/ntp/async_client.rs:69-77
let hosts = lookup_host(&pool).await.map_err(Error::Io)?;
let mut checks = hosts.map(|host| self.get_time(host)).collect::<FuturesUnordered<_>>().into_stream();
timeout(self.settings.timeout, checks.select_next_some()).await ...
```

`handle_ntp_update` then turns `sec + fraction + roundtrip/2` into the node's current time. What is checked: `sntpc` requires the origin timestamp to be echoed, mode 4 or 5, matching version, stratum not 0 (kiss-of-death), and a matching source address. What is not checked, by `sntpc` or the node:

- the leap indicator: `LI = 3` ("clock not synchronised") is accepted (sntpc only rejects `li > 3`, which two bits cannot encode, `lib.rs:L883-L885`);
- stratum 16 (unsynchronised) and root dispersion;
- agreement with any other server, with previous samples, or with the monotonic clock: a single reply may move the clock by any amount forward (LB-001 then makes that permanent until waited out);
- the plausibility of the node's own roundtrip: `sntpc` computes it from two readings of the host clock (`SystemTime`, sntpc `types.rs:L350-L355`) as `(t4 - t1) - (t3 - t2)` with `wrapping_sub` (sntpc `lib.rs:L957`). If the host clock steps backwards between sending and receiving (a time daemon stepping at boot, exactly when the node starts), `t4 - t1` wraps to about `2^32` seconds and `handle_ntp_update` adds half of it.

The transport is plain unauthenticated UDP to `pool.ntp.org`, a volunteer pool, and "first of all resolved addresses wins" favours whichever server is fastest, not whichever is most accurate.

Measured (Appendix B):

```
EXP-1 initial tick (system clock): SlotTick { epoch: Epoch(0), slot: Slot(1000) }
EXP-1 first tick after the +1 day response: SlotTick { epoch: Epoch(2), slot: Slot(87401) }
EXP-1 honest NTP replies received during the next 8 s: 2; ticks emitted: [87402, 87403]
EXP-1 honest slot now: Slot(1009); last emitted >= Slot(87401)
```

One reply with `LI = 3`, stratum 16 and a time one day ahead moved the slot clock from 1,000 to 87,401 (two epochs at the testnet parameters) on the next tick; the honest replies that followed were treated as a backward correction and the stream went silent.

```
EXP-4 sntpc result: sec=1790321684 roundtrip=4294967295000027 us (= 4294967295 s)
EXP-4 initial slot Slot(40321684) -> next tick Slot(2187805333) (epoch Epoch(60772)): +68 years
```

A one-second backward step of the host clock between `t1` and `t4` (simulated with a custom `NtpTimestampGenerator`) produced a correct `sec()` and a wrapped roundtrip; the node's clock jumped 68 years ahead, after which (LB-001) it would emit no tick for 68 years and accept any block slot up to that point.

**Exploit scenario**

An attacker able to answer the node's NTP query (on the network path, controlling the node's DNS resolver, or operating a server in the pool zone the node draws from and answering first) sends one reply `D` ahead. The node now accepts blocks up to `D` slots ahead of the honest network. An attacker who also holds a fraction `alpha` of the stake builds, in advance, a chain from the node's current tip whose blocks sit in slots `(now, now + D]`: it can do so because the lottery for those slots uses the epoch nonce and ledger roots of its own chain. That chain holds about `alpha * f * D` blocks; the victim adopts it as soon as it is longer than the honest chain since the fork point, and once it is `k` blocks deep the victim's LIB moves onto it and the honest chain is rejected permanently (Online rule). At the testnet template's `k = 120`, `f = 1/30` (`deployment/ceremony/genesis/testnet/deployment-template.yaml:L28-L31`) that needs `D >= 3600 / alpha` slots: 10 hours of clock shift with 10 % of the stake, 3 hours with a third. Without stake, the same single reply silences the node's leader, tip-poll and Blend membership for `D` (LB-001). The victim is one node, not the network, hence Medium rather than High; difficulty is High because the attacker needs a network or pool position, and stake for the split.

**Recommendation**

- *Short term*: reject replies with `LI = 3`, stratum 0 or `>= 16`, a roundtrip above a small bound (for example 1 s), or a root dispersion above a bound; query at least three distinct servers per round and take the median offset of the valid replies (not the first); cap the per-round adjustment against the monotonic clock (for example to a few seconds) and require several consistent rounds before a larger step. Treat the result as an offset applied to the monotonic clock, not as an absolute time.
- *Long term*: support authenticated time (NTS, RFC 8915) and several independent pools in the default configuration, and consider the Ouroboros Chronos approach the spec already cites for a clock that honest stake agrees on (S-001).

**References**: `cryptarchia-v1-protocol.md` §Clocks, §Block Header Validation rule 6, §Commit; `cryptarchia-v1-bootstr-sync.md` §Offline Grace Period (the `k`-deep commit example); RFC 5905 §7.3 (leap indicator, stratum 16); #204 S-004 (default pool not surfaced).

### LB-003 · A tick suppressed during a backward-correction pause returns `Pending` without a registered wake-up; the pause outlasts the correction and, with NTP failing, leaves a permanent lag

| | |
|---|---|
| Severity | Low |
| Difficulty | Medium |
| Category | Timing |
| Target | `services/time/src/backends/ntp/mod.rs:L220-L233` (`poll_slot_timer`), `consensus/cryptarchia-engine/src/time.rs:L339` (`MissedTickBehavior::Skip`), `services/time/src/backends/common.rs:L25-L30` (tick counter) |
| Status | Open |

**Description**

When the inner `IntervalStream` yields a tick that is `<= last_emitted_slot`, `poll_slot_timer` returns `Poll::Pending` directly. The inner interval has just returned `Ready`, so it has not registered the task's waker for its next tick; the stream is next polled only when something else wakes the time service task. In the node that is the next NTP round (the update interval's own timer, every 15 s) or an inbound `Info`/`Subscribe`/`CurrentSlot` message. When that wake comes, the interval (with `MissedTickBehavior::Skip`) yields one tick and the zipped counter advances by one slot, however much time has passed: the #38 LB-002 lag mechanism, triggered here by the time service itself.

- With NTP working, the next successful round rebases forward and repairs the numbers, but every pause lasts until the next NTP round instead of until the correction is caught up.
- With NTP failing after a backward correction (a flapping server, or an unsynchronised server whose sample was early), the counter advances one slot per 15 s round until it passes `last_emitted_slot`, then resumes one per second from a value that lags the clock by roughly the time spent paused, for as long as NTP stays down.

EXP-6 (a 2 s backward correction, then an NTP source that never wakes the task): no tick in 8 s, where by design emission resumes after 2 to 3 s.

```
EXP-6 last emitted Slot(40321801); after 8.004354167s: Err(Elapsed(()))
```

EXP-7 (the same correction, then failing rounds every 15 s, as with an unreachable server): 16 s of silence, then a lag of 15 slots against the host clock (2 of which are the correction itself) that stays for the rest of the run.

```
EXP-7 t= 16.0s emitted Slot(40321982) clock Slot(40321997) lag 15
EXP-7 t= 36.0s emitted Slot(40322038) clock Slot(40322053) lag 15
```

The existing tests (`test_backward_ntp_update_rebases_ahead_slot_stream`, `test_on_time_forward_on_time_backward_ntp_sequence`, `ntp/mod.rs:L564-L772`) drive the stream with `futures::poll!` after each `advance`, which re-polls regardless of wake-ups, so they cannot observe this.

**Exploit scenario**

Not attacker-specific. A node whose NTP server sends one early sample and then becomes unreachable runs 13 or more slots behind until NTP returns; a lagging node rejects valid gossip blocks as `FutureBlock` and drops them after `3 x 500 ms` (#38 LB-002, #454), and its leader proposes at stale slots.

**Recommendation**

- *Short term*: in `poll_slot_timer`, loop on the inner stream while it yields suppressed ticks (so the final `Pending` comes from the inner interval and carries its waker), or call `cx.waker().wake_by_ref()` before returning `Pending`. Add a real-time test like EXP-6.
- *Long term*: the per-tick clock derivation recommended in LB-001 and #454 removes the counter, and with it the lag.

**References**: `futures::Stream::poll_next` contract (a `Pending` return must arrange a wake-up); #38 LB-002 (#454).

### LB-004 · An NTP round fails as a whole when any resolved address fails first, and a failing NTP is silent: the node runs on the unsynchronised host clock indefinitely

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Error Reporting |
| Target | `services/time/src/backends/ntp/async_client.rs:L58-L78` (`get_time`, `request_timestamp`), `services/time/src/backends/ntp/mod.rs:L63-L80` (`tick_stream`, failed rounds), `nodes/node/binary/src/config/time/serde.rs:L48-L55` (`listening_interface` default) |
| Status | Open |

**Description**

`select_next_some()` over the `FuturesUnordered` returns the first *item*, which is a `Result`: the first address to fail ends the round with that error even when other addresses would answer. Addresses that fail immediately are common. The socket is bound to `listening_interface`, default `0.0.0.0` (IPv4 only), so any IPv6 address in the resolution fails at `send_to` on the first poll, before any IPv4 reply can arrive; with `listening_interface: "::"` the IPv4 addresses fail the same way. Many public time services publish both A and AAAA records. EXP-3, against a local server:

```
EXP-3 [v4] -> ok=true
EXP-3 [[::1]:43843, 127.0.0.1:43843] -> Err(Sntp(Network))
EXP-3 [127.0.0.1:43843, [::1]:43843] -> Err(Sntp(Network))
```

A round that fails is logged at `warn` and dropped (`ntp/mod.rs:L68-L76`); nothing counts failures, exports the time since the last successful sync, or refuses to run. The node then keeps the slot it started from the host clock (`:L82-L89`) and the tick counter, so an operator who configures such a server gets a node that never synchronises, with a warning every 15 s as the only signal. The name lookup (`lookup_host`, `:L69`) is outside the `timeout`, so a hung resolver delays the round beyond the configured timeout. The shipped test `test_dummy_ntp_server_normal_poll` (`ntp/mod.rs:L774-L803`) passes with an unresolvable server, which documents that this fail-open behaviour is expected. #34 deferred the NTP fallback policy to this issue.

**Exploit scenario**

Operational: an operator sets `time.backend.server` to a dual-stack name. Every round fails; the node runs on its host clock, whose drift (or error, LB-001) is never corrected. Adversarial variant: a party that can make one address in the resolved set fail fast (or return a stray datagram from another source, which `sntpc` answers with `ResponseAddressMismatch`) can suppress synchronisation without being able to spoof a reply.

**Recommendation**

- *Short term*: filter the resolved addresses by the socket's address family (or bind one socket per family); collect replies until the timeout and keep the valid ones instead of returning the first item; put `lookup_host` inside the timeout. Export `time_ntp_last_success_seconds`, `time_ntp_failures_total` and the current offset, and log at `error` after N consecutive failures.
- *Long term*: define the node's behaviour without a trusted clock (refuse to propose, or mark the node unhealthy) instead of silently trusting the host clock; see S-001.

**References**: `cryptarchia-v1-protocol.md` §Clocks; #34 (NTP fallback deferred to #46); #204 S-004.

### LB-005 · The genesis wait and the offline grace period read the host clock, not the node's slot clock

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Timing |
| Target | `services/chain/chain-service/src/service/phases/awaiting_genesis_time.rs:L165-L181` (`create_genesis_timer`), `services/chain/chain-service/src/bootstrap/state.rs:L30-L54` (`check_offline_grace_period`), `services/chain/chain-service/src/states.rs:L67-L70` (timestamp) |
| Status | Open |

**Description**

The node has three clocks: the NTP-derived slot clock (time service), the monotonic clock (tokio timers), and the host wall clock. Two consensus-relevant decisions use the third:

- `create_genesis_timer` computes `genesis_time - OffsetDateTime::now_utc()` from the host clock and sleeps that long (`awaiting_genesis_time.rs:L172-L177`). Until it fires, every `ApplyBlock` is answered with `Error::AwaitingGenesisTime` (`:L149-L159`), which chain-network maps to `ApiError::Unexpected` and records as a permanent rejection (#143 LB-003, #556; re-verified at `chain-service/src/api.rs:L392-L404` and `chain-network/src/lib.rs:L955-L976`, where `AwaitingGenesisTime` is still absent from the recoverable set). #143 LB-003 attributes the trigger to "a slow clock"; the finding here is that NTP does not prevent it: a node whose NTP-derived slot clock is correct but whose host clock is `d` seconds behind still rejects, and permanently records, every block of the first `d` seconds after genesis.
- The offline grace period compares `SystemTime::now()` at start with a `SystemTime` recorded every minute (`states.rs:L67-L70`, `bootstrap/state.rs:L34-L35`). A backward step of the host clock between shutdown and restart makes the measured offline time shorter by the step. The code is conservative when the result is negative (it bootstraps, `:L45-L52`), but not when the step is smaller than the offline time. The common case is a host without a battery-backed clock: `fake-hwclock` and similar restore the last saved time at boot, so if the node starts before the time daemon has stepped the clock, the measured offline time is close to zero regardless of how long the host was off. The node then restarts with the Online rule after a week offline. `cryptarchia-v1-bootstr-sync.md` §Setting the Fork Choice Rule only requires the Bootstrap rule when the node is "certain" it was offline too long, so this is conformant, but the node holds a better measure: the slot clock it already trusts for block validity, compared with the slot of its recorded tip or LIB.

**Exploit scenario**

Operational for the genesis case: operators start genesis validators minutes before genesis on hosts whose clocks are off; each records the first blocks it sees as invalid and must be restarted to clear the cache (#556). For the restart case: an RTC-less validator is powered off for several days and boots with its clock at the last saved time; it comes back Online, and the first `k`-deep chain it is served becomes immutable for it: the long-range scenario the Offline Grace Period exists to prevent (`cryptarchia-v1-bootstr-sync.md` §Offline Grace Period). Exploiting it needs an attacker with a long fork ready and a victim with this host setup, hence High difficulty.

**Recommendation**

- *Short term*: take the genesis wait from the time service (end the phase on the first tick with `slot >= 1`, or on `CurrentSlot`); add `AwaitingGenesisTime` to `is_recoverable_apply_error` (#556). For the grace period, also record the last tick slot and, at start, bootstrap if either the host clock or `current_slot - recorded_slot` (taken after the first successful NTP round) exceeds the grace period.
- *Long term*: route every wall-clock decision in consensus services through the time service, and forbid `SystemTime::now` / `OffsetDateTime::now_utc` in `services/chain` with a clippy `disallowed-methods` entry.

**References**: `cryptarchia-v1-bootstr-sync.md` §Setting the Fork Choice Rule, §Offline Grace Period, §Offline Duration Measurement; #143 LB-003 (#556); #42 S-001.

## 5. Suggestions (non-security)

### S-001 · Specification: state what the clock must guarantee and what a node does when it cannot

| | |
|---|---|
| Target | `cryptarchia-v1-protocol.md` §Clocks, §Block Header Validation rule 6 |

§Clocks says only that the protocol relies on NTP. Rule 6 compares the header slot with local time with no tolerance, and every property of this report (LB-001 to LB-004) turns on unstated choices: how many sources, what bound on a single correction, whether a backward correction is stepped, slewed or frozen, what tolerance rule 6 allows (the implementation effectively allows `3 x 500 ms`, `chain-network/src/lib.rs:L78-L79`), and what a node without a trusted clock may do (validate only, or also propose). The section should also say that the offline-duration measurement of `cryptarchia-v1-bootstr-sync.md` should use the same clock as rule 6.

### S-002 · Specification: a network launched from genesis cannot propose for `T_boot`

| | |
|---|---|
| Target | `cryptarchia-v1-bootstr-sync.md` §Setting the Fork Choice Rule, §Prolonged Bootstrap Period, §Proposing New Blocks; `services/chain/chain-service/src/bootstrap/state.rs:L17-L19`, `service/phases/pbp.rs:L55-L69` |

A node starting from genesis always uses the Bootstrap rule, keeps it for `T_boot` after IBD (skipped for genesis nodes), and may only propose afterwards. Read literally, the first block of a new network comes `T_boot` (24 h in the spec, 1 h by the node's default, #42 LB-002) after genesis, and the first stake-inference window starts with that many empty slots, which on short-epoch deployments is the halted-chain regime of #634 LB-001. The spec should state the launch procedure (for example: genesis nodes start with the Online rule and no PBP, or `T_boot` counts from genesis for nodes started before it).

### S-003 · Gate proposals on the tip being recent

| | |
|---|---|
| Target | `services/chain/chain-leader/src/lib.rs:L451-L520`, `leadership.rs:L416-L463` |

The leader proposes on whatever tip it has once the Online rule is in force. A node whose tip lags the clock by more than the tip-poll threshold (it already computes that lag, `chain-network/src/sync/tip_poll.rs:L91-L95`) is either still syncing (#204 LB-002, #689) or has a clock ahead (LB-001, LB-002); in both cases its blocks extend a stale tip, and when the lag spans empty inference windows its own lottery runs with a collapsed total stake (#634 LB-001). Skipping the lottery while `current_slot - tip_slot` exceeds a bound costs nothing on a healthy node and removes these cases.

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

## Appendix B · Experiment code

Appended to `services/time/src/backends/ntp/mod.rs` in a scratch clone at `c4c86be1`; nothing else was changed. Run with:

```sh
CARGO_TARGET_DIR=<target> cargo test --release -p logos-blockchain-time-service --features ntp --lib exp46 -- --test-threads=1 --nocapture
```

Result: `test result: ok. 7 passed` (the tests print the quoted lines; EXP-6 and EXP-7 print rather than assert).

```rust
// ---------------------------------------------------------------------------
// Issue #46 experiments (scratch only, not part of the audited tree).
// ---------------------------------------------------------------------------
#[cfg(test)]
mod exp46 {
    use std::{
        net::{Ipv4Addr, SocketAddr},
        sync::{
            Arc,
            atomic::{AtomicI64, AtomicU32, Ordering},
        },
    };

    use sntpc::{NtpContext, NtpTimestampGenerator};
    use tokio::net::UdpSocket;

    use super::*;

    const NTP_DELTA: u64 = 2_208_988_800;

    fn epoch_config() -> EpochConfig {
        EpochConfig {
            epoch_stake_distribution_stabilization: NonZero::new(3).unwrap(),
            epoch_period_nonce_buffer: NonZero::new(3).unwrap(),
            epoch_period_nonce_stabilization: NonZero::new(4).unwrap(),
        }
    }
    fn base_period() -> NonZero<u64> {
        // testnet template: k = 120, f = 1/30 -> floor(k/f) = 3600
        NonZero::new(3600).unwrap()
    }

    fn to_ntp(unix_nanos: i128) -> u64 {
        let secs = (unix_nanos / 1_000_000_000) as u64 + NTP_DELTA;
        let frac = ((unix_nanos % 1_000_000_000) as u64) * (1u64 << 32) / 1_000_000_000;
        (secs << 32) | frac
    }

    /// Minimal NTP server on 127.0.0.1. Each reply carries
    /// `now + offset_secs` (offset read at reply time), LI = 3 (clock
    /// unsynchronised) and stratum 16 (unsynchronised), i.e. a server that
    /// says it should not be trusted.
    async fn fake_ntp_server(offset_secs: Arc<AtomicI64>, replies: Arc<AtomicU32>) -> SocketAddr {
        let sock = UdpSocket::bind((Ipv4Addr::LOCALHOST, 0)).await.unwrap();
        let addr = sock.local_addr().unwrap();
        tokio::spawn(async move {
            let mut buf = [0u8; 48];
            loop {
                let Ok((n, from)) = sock.recv_from(&mut buf).await else { return };
                if n != 48 {
                    continue;
                }
                let now = OffsetDateTime::now_utc().unix_timestamp_nanos()
                    + i128::from(offset_secs.load(Ordering::SeqCst)) * 1_000_000_000;
                let ts = to_ntp(now).to_be_bytes();
                let version = (buf[0] >> 3) & 0x7;
                let mut resp = [0u8; 48];
                resp[0] = (3 << 6) | (version << 3) | 4; // LI=3 (alarm), mode 4
                resp[1] = 16; // stratum 16: unsynchronised
                resp[24..32].copy_from_slice(&buf[40..48]); // origin = client tx
                resp[32..40].copy_from_slice(&ts); // receive
                resp[40..48].copy_from_slice(&ts); // transmit
                replies.fetch_add(1, Ordering::SeqCst);
                let _ = sock.send_to(&resp, from).await;
            }
        });
        addr
    }

    fn settings(
        genesis: OffsetDateTime,
        server: String,
        update_interval: Duration,
    ) -> TimeServiceSettings<NtpTimeBackendSettings> {
        TimeServiceSettings {
            slot_config: SlotConfig {
                slot_duration: Duration::from_secs(1),
                genesis_time: genesis,
            },
            epoch_config: epoch_config(),
            base_period_length: base_period(),
            backend: NtpTimeBackendSettings {
                ntp_server: server,
                ntp_client_settings: NTPClientSettings {
                    timeout: Duration::from_secs(2),
                    listening_interface: Ipv4Addr::UNSPECIFIED.into(),
                },
                update_interval,
            },
        }
    }

    /// EXP-1: one response from an unsynchronised server (LI=3, stratum 16)
    /// claiming `now + 1 day` is adopted as-is on the next tick; a later
    /// correct response is treated as a backward correction and freezes the
    /// tick stream at the inflated slot.
    #[tokio::test]
    async fn exp1_single_forward_response_is_adopted_then_freezes() {
        let offset = Arc::new(AtomicI64::new(86_400));
        let replies = Arc::new(AtomicU32::new(0));
        let server = fake_ntp_server(Arc::clone(&offset), Arc::clone(&replies)).await;
        let genesis = OffsetDateTime::now_utc() - time::Duration::seconds(1_000);
        let backend = NtpTimeBackend::init(settings(
            genesis,
            server.to_string(),
            Duration::from_secs(3),
        ));
        let (initial, mut stream) = backend.tick_stream();
        println!("EXP-1 initial tick (system clock): {initial:?}");

        let first = tokio::time::timeout(Duration::from_secs(3), stream.next())
            .await
            .expect("tick")
            .unwrap();
        println!("EXP-1 first tick after the +1 day response: {first:?}");
        assert!(first.slot.into_inner() >= initial.slot.into_inner() + 86_400);

        // Server now tells the truth. The next NTP round (<= 3 s) is a
        // backward correction by one day.
        offset.store(0, Ordering::SeqCst);
        let before = replies.load(Ordering::SeqCst);
        let mut emitted = Vec::new();
        let deadline = tokio::time::Instant::now() + Duration::from_secs(8);
        while tokio::time::Instant::now() < deadline {
            if let Ok(Some(t)) =
                tokio::time::timeout_at(deadline, stream.next()).await
            {
                emitted.push(t.slot.into_inner());
            }
        }
        let after = replies.load(Ordering::SeqCst);
        println!(
            "EXP-1 honest NTP replies received during the next 8 s: {}; ticks emitted: {:?}",
            after - before,
            emitted
        );
        assert!(after > before, "the honest reply must have been served");
        // Ticks emitted in the 8 s window come only from before the honest
        // reply was applied; after it, the stream is silent.
        let honest_slot = Slot::from_offset_and_config(OffsetDateTime::now_utc(), SlotConfig {
            slot_duration: Duration::from_secs(1),
            genesis_time: genesis,
        });
        println!("EXP-1 honest slot now: {honest_slot:?}; last emitted >= {:?}", first.slot);
    }

    /// EXP-2: the node's slot clock starts from the system clock. A system
    /// clock that is 1 h ahead of NTP makes the first NTP round a backward
    /// correction: the stream emits nothing for the whole hour, and the
    /// last emitted slot (the one chain-service uses for the future-block
    /// check) stays 3600 slots ahead.
    #[tokio::test(start_paused = true)]
    async fn exp2_system_clock_ahead_freezes_for_the_whole_skew() {
        let genesis = OffsetDateTime::UNIX_EPOCH;
        let slot_config = SlotConfig {
            slot_duration: Duration::from_secs(1),
            genesis_time: genesis,
        };
        let true_now = OffsetDateTime::from_unix_timestamp(1_800_000_000).unwrap();
        let system_now = true_now + time::Duration::hours(1); // host clock 1 h fast
        // Same construction as `NtpTimeBackend::tick_stream`, with the system
        // clock replaced by `system_now`.
        let (initial, timer) = slot_timer(
            slot_config,
            system_now,
            Slot::from_offset_and_config(system_now, slot_config),
            epoch_config(),
            base_period(),
        );
        let mut stream = NtpStream {
            interval: Box::pin(futures::stream::iter(vec![NtpResult::new(
                u32::try_from(true_now.unix_timestamp()).unwrap(),
                0,
                0,
                0,
                1,
                0,
            )]).chain(futures::stream::pending())),
            slot_config,
            epoch_config: epoch_config(),
            base_period_length: base_period(),
            slot_timer: timer,
            last_emitted_slot: initial.slot,
        };
        assert!(matches!(futures::poll!(stream.next()), Poll::Pending));
        let mut silent_secs = 0u64;
        loop {
            tokio::time::advance(Duration::from_secs(1)).await;
            silent_secs += 1;
            if let Poll::Ready(Some(t)) = futures::poll!(stream.next()) {
                println!(
                    "EXP-2 initial slot {:?} (system clock), NTP slot {:?}; first tick {:?} after {silent_secs} s of silence",
                    initial.slot,
                    Slot::from_offset_and_config(true_now, slot_config),
                    t.slot
                );
                assert!(silent_secs >= 3600);
                break;
            }
            assert!(silent_secs < 4000);
        }
    }

    /// EXP-3: a round in which one resolved address fails fast fails as a
    /// whole, even though another address answers. Here the IPv6 address
    /// cannot be reached from the IPv4 socket the default config binds.
    #[tokio::test]
    async fn exp3_mixed_family_resolution_fails_every_round() {
        let offset = Arc::new(AtomicI64::new(0));
        let replies = Arc::new(AtomicU32::new(0));
        let v4 = fake_ntp_server(offset, Arc::clone(&replies)).await;
        let client = AsyncNTPClient::new(NTPClientSettings {
            timeout: Duration::from_secs(2),
            listening_interface: Ipv4Addr::UNSPECIFIED.into(),
        });
        let only_v4 = client.request_timestamp(&[v4][..]).await;
        println!("EXP-3 [v4] -> ok={}", only_v4.is_ok());
        assert!(only_v4.is_ok());
        let v6: SocketAddr = format!("[::1]:{}", v4.port()).parse().unwrap();
        for order in [[v6, v4], [v4, v6]] {
            let mixed = client.request_timestamp(&order[..]).await;
            println!("EXP-3 {order:?} -> {:?}", mixed.as_ref().map(NtpResult::sec));
            assert!(mixed.is_err());
        }
    }

    /// EXP-4: a system-clock step backwards between the request (T1) and the
    /// reply (T4) makes sntpc's roundtrip wrap to ~2^32 s; `handle_ntp_update`
    /// adds half of it, and the slot clock jumps ~68 years ahead.
    #[derive(Clone, Copy, Default)]
    struct SteppingClock {
        now: std::time::Duration,
    }
    static CALLS: AtomicU32 = AtomicU32::new(0);
    impl NtpTimestampGenerator for SteppingClock {
        fn init(&mut self) {
            let real = std::time::SystemTime::now()
                .duration_since(std::time::UNIX_EPOCH)
                .unwrap();
            // second call (T4) happens 1 s "earlier" than the first (T1)
            self.now = if CALLS.fetch_add(1, Ordering::SeqCst) == 0 {
                real
            } else {
                real - std::time::Duration::from_secs(1)
            };
        }
        fn timestamp_sec(&self) -> u64 {
            self.now.as_secs()
        }
        fn timestamp_subsec_micros(&self) -> u32 {
            self.now.subsec_micros()
        }
    }

    #[tokio::test(start_paused = true)]
    async fn exp4_clock_step_during_exchange_jumps_68_years() {
        let offset = Arc::new(AtomicI64::new(0));
        let replies = Arc::new(AtomicU32::new(0));
        let server = fake_ntp_server(offset, replies).await;
        let sock = UdpSocket::bind((Ipv4Addr::LOCALHOST, 0)).await.unwrap();
        let result = sntpc::get_time(server, &sock, NtpContext::new(SteppingClock::default()))
            .await
            .unwrap();
        println!(
            "EXP-4 sntpc result: sec={} roundtrip={} us (= {} s)",
            result.sec(),
            result.roundtrip(),
            result.roundtrip() / 1_000_000
        );
        let slot_config = SlotConfig {
            slot_duration: Duration::from_secs(1),
            genesis_time: OffsetDateTime::from_unix_timestamp(1_750_000_000).unwrap(),
        };
        let now = OffsetDateTime::now_utc();
        let (initial, timer) = slot_timer(
            slot_config,
            now,
            Slot::from_offset_and_config(now, slot_config),
            epoch_config(),
            base_period(),
        );
        let mut stream = NtpStream {
            interval: Box::pin(futures::stream::iter(vec![result]).chain(futures::stream::pending())),
            slot_config,
            epoch_config: epoch_config(),
            base_period_length: base_period(),
            slot_timer: timer,
            last_emitted_slot: initial.slot,
        };
        assert!(matches!(futures::poll!(stream.next()), Poll::Pending));
        tokio::time::advance(Duration::from_secs(1)).await;
        let Poll::Ready(Some(t)) = futures::poll!(stream.next()) else {
            panic!("expected a tick");
        };
        let years = (t.slot.into_inner() - initial.slot.into_inner()) / (365 * 86_400);
        println!("EXP-4 initial slot {:?} -> next tick {:?} (epoch {:?}): +{years} years", initial.slot, t.slot, t.epoch);
        assert!(years >= 60);
    }

    /// EXP-6: after a backward correction, a suppressed tick returns
    /// `Poll::Pending` without the inner timer having registered a waker, so
    /// the stream is only re-polled by some other wake-up (the next NTP round
    /// in the node). Here the NTP source goes silent after one 2 s backward
    /// correction: by design emission should resume after about 2-3 s.
    #[tokio::test]
    async fn exp6_suppressed_tick_loses_wakeup() {
        let slot_config = SlotConfig {
            slot_duration: Duration::from_secs(1),
            genesis_time: OffsetDateTime::from_unix_timestamp(1_750_000_000).unwrap(),
        };
        let now = OffsetDateTime::now_utc();
        let (initial, timer) = slot_timer(
            slot_config,
            now,
            Slot::from_offset_and_config(now, slot_config),
            epoch_config(),
            base_period(),
        );
        let behind = now - time::Duration::seconds(2);
        let mut stream = NtpStream {
            interval: Box::pin(
                futures::stream::iter(vec![NtpResult::new(
                    u32::try_from(behind.unix_timestamp()).unwrap(),
                    0, 0, 0, 1, 0,
                )])
                .chain(futures::stream::pending()),
            ),
            slot_config,
            epoch_config: epoch_config(),
            base_period_length: base_period(),
            slot_timer: timer,
            last_emitted_slot: initial.slot,
        };
        let start = std::time::Instant::now();
        let r = tokio::time::timeout(Duration::from_secs(8), stream.next()).await;
        println!(
            "EXP-6 last emitted {:?}; after {:?}: {:?}",
            initial.slot,
            start.elapsed(),
            r.map(|t| t.map(|t| t.slot))
        );
    }

    /// EXP-7: same as EXP-6, but NTP rounds keep firing (every 3 s) and all
    /// fail, as with an unreachable server. Each round wakes the stream once;
    /// the inner `Skip` interval then yields a single tick, so the counter
    /// crawls one slot per round until it passes `last_emitted_slot`, and
    /// the emitted slot lags the clock from then on.
    #[tokio::test]
    async fn exp7_backward_correction_then_failing_ntp_leaves_lag() {
        let slot_config = SlotConfig {
            slot_duration: Duration::from_secs(1),
            genesis_time: OffsetDateTime::from_unix_timestamp(1_750_000_000).unwrap(),
        };
        let now = OffsetDateTime::now_utc();
        let (initial, timer) = slot_timer(
            slot_config,
            now,
            Slot::from_offset_and_config(now, slot_config),
            epoch_config(),
            base_period(),
        );
        let behind = now - time::Duration::seconds(2);
        let mut rounds = interval(Duration::from_secs(15));
        rounds.set_missed_tick_behavior(MissedTickBehavior::Skip);
        rounds.reset(); // first failing round 15 s from now (node default update_interval)
        let failing = IntervalStream::new(rounds).filter_map(|_| Box::pin(async { None::<NtpResult> }));
        let mut stream = NtpStream {
            interval: Box::pin(
                futures::stream::iter(vec![NtpResult::new(
                    u32::try_from(behind.unix_timestamp()).unwrap(),
                    0, 0, 0, 1, 0,
                )])
                .chain(failing),
            ),
            slot_config,
            epoch_config: epoch_config(),
            base_period_length: base_period(),
            slot_timer: timer,
            last_emitted_slot: initial.slot,
        };
        let start = std::time::Instant::now();
        let deadline = tokio::time::Instant::now() + Duration::from_secs(36);
        while let Ok(Some(t)) = tokio::time::timeout_at(deadline, stream.next()).await {
            let clock = Slot::from_offset_and_config(OffsetDateTime::now_utc(), slot_config);
            println!(
                "EXP-7 t={:>5.1}s emitted {:?} clock {:?} lag {}",
                start.elapsed().as_secs_f32(),
                t.slot,
                clock,
                clock.into_inner() - t.slot.into_inner()
            );
        }
        println!("EXP-7 last emitted before start: {:?}", initial.slot);
    }

    /// EXP-5: startup before genesis. The current tick is slot 0 / epoch 0,
    /// and the first tick is slot 1 at genesis + 1 slot; nothing is emitted
    /// before.
    #[tokio::test(start_paused = true)]
    async fn exp5_start_before_genesis() {
        let now = OffsetDateTime::now_utc();
        let slot_config = SlotConfig {
            slot_duration: Duration::from_secs(1),
            genesis_time: now + time::Duration::seconds(10),
        };
        let (initial, mut timer) = crate::backends::common::slot_timer(
            slot_config,
            now,
            Slot::from_offset_and_config(now, slot_config),
            epoch_config(),
            base_period(),
        );
        println!("EXP-5 initial tick before genesis: {initial:?}");
        assert_eq!(initial.slot, Slot::genesis());
        let start = tokio::time::Instant::now();
        let t = timer.next().await.unwrap();
        let waited = start.elapsed();
        println!("EXP-5 first tick {t:?} after {waited:?} of virtual time");
        assert_eq!(t.slot, Slot::new(1));
        assert!(waited >= Duration::from_secs(10));
    }
}
```
