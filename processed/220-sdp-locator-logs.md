# Audit Report — SDP locator validation and log-surface re-verification

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/220`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `a805329f8a186eb6989f09a7c49dee4a0e07473b` — component(s): SDP locator parsing, Blend membership and dial-error logging, chain-sync error strings
Specs: `https://github.com/logos-co/logos-lips` @ `7244d3b05ddec91a4a7b565bd5a9340ab77ededd` — read in full: `bedrock-service-declaration-protocol.md`, `bedrock-service-reward-distribution.md`, `bedrock-anonymous-leaders-reward.md`; core context: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`
Date: `2026-09-18` — author: `codex` — status: `final`

---

## 1. Summary

- Overall assessment: Re-verification confirms the existing SDP locator log-injection path, but finds no distinct new defect beyond canonical finding `37-LB-005` (issue #461).
- Findings: `0 new findings; canonical 37-LB-005 re-verified — Low / Medium / Auditing and Logging.`
- Key themes: `unvalidated DNS multiaddr labels`, `unescaped logging`, `spec migration risk`
- Must-fix before launch: none new; retain the existing #461 remediation plan and classification.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `core/src/sdp/mod.rs` | `Locator` length, protocol, string, and binary deserialisation. |
| `services/blend/src/core/mod.rs`, `services/blend/src/membership/*` | Membership logging that formats locators. |
| `services/blend/src/{core,edge}/backends/libp2p/swarm.rs` | Dial-error logging that formats multiaddrs. |
| `consensus/cryptarchia-sync`, `services/chain/chain-service`, `services/chain/chain-network` | Candidate peer/error strings from the issue's sweep. |
| `services/tracing` and pinned `multiaddr` sources | Whether message formatting escapes control characters and whether DNS components validate labels. |
| Channel inscriptions and wire/on-chain string surfaces | Whether byte payloads, channel errors, identify metadata, and chain-sync reasons become log/error-display sinks. |

**Out of scope**

The full SDP state/reward implementation, unrelated on-chain strings, exporter transport security, and fixes in `logos-blockchain` were not reviewed. Third-party behavior outside the cited `multiaddr` parsing/display and `tracing-subscriber` formatting paths was assumed correct.

**Assumptions**

The pinned source and specification revisions in the header are authoritative. The current SDP specification defines the canonical binary multiaddr form and a 329-byte limit, but does not define LDH hostname grammar, label limits, or whether `/dnsaddr/` is permitted. Adding such restrictions would therefore require a specification decision and a migration/compatibility policy for already stored declarations.

## 3. Method

- Read issue `#220`, parent `#8`, and repo-level context issue `#19`.
- Read the three area specifications named by parent #8 in full, plus both core overview specifications in full.
- Re-read the current `Locator::try_from`, `FromStr`, `Display`, Blend membership, and dial-error paths at the target commit.
- Inspected pinned `multiaddr-0.18.2` parsing and display code: DNS components retain any UTF-8 string and write it directly during display; no LDH validation is applied.
- Searched the requested candidate paths for channel inscriptions, generic `String`/`Option<String>` wire fields, `agent_version`, identify addresses, `BlocksUnavailableReason::Unknown(String)`, error `Display` implementations, and logging sinks. The locator path matches existing canonical finding `37-LB-005`; the other candidates did not produce a separate peer-controlled INFO/ERROR log injection finding.
- Ran a scratch-only parser/formatter experiment against the pinned dependency behavior. `/dns4/a\nb/tcp/1` was accepted and displayed with a literal newline. A local escaping wrapper changed that newline to `\\n`; 100,000 iterations over a 75-byte sample took 91,856 µs (about 918 ns/iteration) on this host. This was not an upstream code change or a production benchmark.

## 4. Findings

No new `LB-NNN` finding is opened. The confirmed defect is the existing canonical finding `37-LB-005`, titled “An SDP locator's DNS label is written unescaped into every core node's log,” tracked at [issue #461](https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/461), classified **Low / Medium / Auditing and Logging**, and reported in [processed/37-logging-tracing-leaks-log-volume.md](../processed/37-logging-tracing-leaks-log-volume.md).

### Re-verification of `37-LB-005`

`core/src/sdp/mod.rs:168-196` checks the locator byte length, rejects unspecified IP addresses, and rejects `/p2p/`; it does not constrain `/dns/`, `/dns4/`, `/dns6/`, or `/dnsaddr/` labels. The pinned `multiaddr-0.18.2` parser accepts those components as UTF-8 strings (`src/protocol.rs:169-181, 303-319`) and its display implementation writes them raw (`src/protocol.rs:712-714`).

The value reaches a core node's INFO log at `services/blend/src/core/mod.rs:675-679`, where the complete current membership is formatted with `{:?}`. It also reaches ERROR/WARN dial-failure logs at `services/blend/src/edge/backends/libp2p/swarm.rs:298-302, 480-486` and `services/blend/src/core/backends/libp2p/swarm.rs:345-354, 575-583`. A DNS label containing a newline or terminal escape sequence therefore remains an unescaped log message field. This is the same root cause, impact, and trust boundary documented by #461; no second identifier is warranted.

The specification point is narrower than the implementation recommendation: the SDP document requires a multiaddr location in canonical binary form and limits it to 329 bytes, but is silent on hostname grammar. Rejecting control characters or enforcing LDH labels is a reasonable hardening choice, not a demonstrated current spec deviation. Before applying it to consensus parsing, the specification should define the grammar and the handling of declarations already accepted under the current rule.

The requested sweep found no additional canonical finding at this revision. Channel inscriptions are `UpperBoundedVec<u8, MAX_BYTES>` and are serialized as bytes; the inscription payload itself is not formatted into the reviewed logging paths. The channel error `UnauthorizedSigner` stores a locally formatted public-key string, while channel IDs, parents, and signer values are fixed-size typed values; no free-form inscription reaches an error `Display` sink. `agent_version` and identify addresses are emitted only on TRACE paths already covered by the logging review. The `BlocksUnavailableReason::Unknown(String)` values inspected are constructed from local chain/provider errors before being returned, not copied directly from a peer-supplied wire string. The general formatter hardening recommendation remains useful defense in depth, but it duplicates #461's long-term recommendation rather than creating a new finding.

The scratch experiment confirms the exact regression case requested by #220: the current parser admits `/dns4/a\nb/tcp/1`, and the default formatter writes the newline as a record-breaking byte. The local `FormatEvent`-equivalent wrapper escaped the control character with the measured small per-event cost above. This is evidence for the recommendation, not an upstream implementation or a claim that the cost is representative of all deployments.

The issue's older instructions to implement validation and file a specification issue are converted here into recommendations because this audit workflow permits source inspection and local experiments but prohibits mutating the audited upstream repository. No upstream validation, formatter, or specification change was made.

## 5. Suggestions

### S-001 · Define locator hostname grammar before tightening consensus parsing

If the protocol wants DNS locators, specify the allowed variants, ASCII/LDH rules, per-label and total limits, and `/dnsaddr/` policy in `bedrock-service-declaration-protocol.md`. Define whether existing declarations are grandfathered, rejected only on new admission, or migrated before changing `Locator` consensus deserialisation.

### S-002 · Add a regression test and retain formatter-level escaping

Add a regression test covering `/dns4/a\nb/tcp/1` at both `Locator` deserialisation and declaration admission. Independently, escape control characters in file/stdout log message formatting so future remote strings cannot forge records even if another parser accepts them. The scratch result above is a local prototype and cost measurement only; it should not be treated as an upstream patch.
