# Audit Report — Silent `serde(default)` values on limits, keys and consensus inputs

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/204`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `3d5d419ec85c7b23e4d7e1455bd1cde845264a9e` — component(s): `nodes/node/binary/src/config`, `libp2p/src/config`, `ledger/src/config.rs`, `ledger/src/cryptarchia/mod.rs`, `services/chain/chain-network/src/bootstrap`, `services/chain/chain-service/src/bootstrap`, `services/sdp`, `services/blend`, `tools/blockchain-tools/src/genesis`
Specs: `https://github.com/logos-co/logos-lips` @ `75d3d0382604d4a0d8e246c268935dd386ffc8ea` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `bedrock-genesis-block.md`, `p2p-network-bootstrapping.md` (in full); `cryptarchia-proof-of-leadership.md` §Circuit Public Inputs, `cryptarchia-v1-protocol.md` §Epoch State, §Clocks (by section)
Date: `2026-09-15` — author: `claude-fable-5.1` — status: `final`

---

## 1. Summary

- Overall assessment: of the 115 `#[serde(default)]` sites under the node's config tree, `libp2p/src/config` and the services' settings structs, six sit on values that are a limit, an identity or a consensus input; four of those were already filed (#474, #475, #488) and this report sharpens one of them and adds two. Omitting `faucet_pk` is not the mild threshold drift #34 LB-002 (#475) described: because the genesis faucet note is `u64::MAX` and the genesis stake sum is an unchecked `u64` addition, the node that omits the key computes a total stake that wrapped to one token less than everyone else's, gets different lottery constants, and can never verify another node's Proof of Leadership. The faucet exclusion itself is a spec deviation: `bedrock-genesis-block.md` sets the initial total stake to the total distributed at genesis and has no faucet. Separately, omitting the IBD peer list silently skips Initial Block Download and the node goes Online an hour later on whatever it received by gossip.
- Findings: `0` critical · `0` high · `0` medium · `2` low · `0` informational
- Key themes: "a missing key is filled in silently while a misspelled one is rejected", "consensus inputs outside the genesis block", "the same peer list in two config sections"
- Must-fix before launch: none for a network whose deployment files come from the ceremony tool; the `faucet_pk` and IBD-peer defaults should be removed before third parties write deployment or user files by hand.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `nodes/node/binary/src/config/**` | every `#[serde(default)]` (81 sites), the `Default` impl each resolves to, the CLI overrides in `config/mod.rs`, the `init`/`update` generators in `cli/config/` |
| `libp2p/src/config/**` | 24 sites (`SwarmConfig`, kademlia, identify, NAT) |
| `services/**` settings structs | 10 sites (pow, chain-network sync, chain-service bootstrap, wallet, sdp state, api request body, tx pool, network backend) |
| `utils/src/yaml.rs` | the loader: `serde_ignored` with `OnUnknownKeys::Fail` for user and deployment files |
| `ledger/src/config.rs`, `ledger/src/cryptarchia/mod.rs:738-800` | `faucet_pk` and the genesis `total_stake` |
| `services/chain/chain-leader/src/leadership.rs:440-450`, `services/wallet/src/lib.rs:288-300` | leader-side faucet exclusion |
| `services/chain/chain-network/src/bootstrap/ibd.rs`, `services/chain/chain-service/src/bootstrap/state.rs`, `src/service/phases/pbp.rs` | what an empty IBD peer set does |
| `tools/blockchain-tools/src/bin/genesis.rs`, `src/genesis/distribution.rs` | how the ceremony tool emits `faucet_pk` and the faucet note |
| `deployment/ceremony/genesis/*/`, `nodes/node/standalone-deployment-config.yaml`, `nodes/node/binary/src/config/deployment/settings.yaml`, `tools/config/src/deployment.rs` | every checked-in deployment and the test deployment builder, for the `faucet_pk` key |

**Out of scope**

Float-typed knobs and their bounds (`sample_ratio`, `gossip_factor`) and the `node_key` question: both are #198's, and `node_key` is filed as #488. Config precedence between deployment file, user file, CLI and environment beyond what the two findings need. Secrets on disk and file permissions (#65). The genesis inscription's own encoding (#106, closed). The Blend delivery-failure fallback mechanism itself (#576, #588); only its default is classified here. Third-party crates assumed correct: `serde`, `serde_yaml`, `serde_ignored`, `clap`.

**Assumptions**

- The specification at the commit above is correct unless stated otherwise.
- Release profile facts of issue #19, re-verified at this commit: `overflow-checks` is not set in `[profile.release]` (`Cargo.toml:11-14`), so `u64` addition wraps in production builds; `arithmetic_side_effects` is allowed.
- Deployment files are normally produced by `blockchain-tools genesis ceremony` and user files by `logos-blockchain-node config init`; the exposure of every finding is a file written or edited by hand, or produced by third-party tooling.

## 3. Method

- Manual review of the in-scope paths, working through issue `#204` under parent `#26`, all four checklist items. Each `#[serde(default)]` was listed with `grep -rn 'serde(default'` over `nodes/node/binary/src/config`, `libp2p/src/config`, `services`, `network`, `blend` (test files excluded): 115 sites at this commit against the 132 the issue counted at `a805329f8`. For each, the field or struct it applies to and the value its `Default` resolves to were read, and the field was classified as tuning knob, limit, identity/key, or consensus input (the table in S-001).
- Spec conformance against `bedrock-genesis-block.md` §Initial Token Distribution, §Cryptarchia Parameters, §Initial Epoch State; `p2p-network-bootstrapping.md` §Protocol step 1; `cryptarchia-proof-of-leadership.md` §Circuit Public Inputs; `cryptarchia-v1-protocol.md` §Clocks.
- Automated tooling: one throw-away unit test in the `ledger` crate, run with `cargo test --release` (`rustc 1.98.1`, workspace release profile): `LedgerState::from_utxos` over three notes (100 T, 200 T and `u64::MAX`) with `faucet_pk` set to the third note's key and with `faucet_pk: None`. Result: `total_stake` 300,000,000,000,000 with the key, 299,999,999,999,999 without, and different `lottery_0`. Reproducible from LB-001's description alone.
- Dynamic testing: none against a running node.

**Checked and ruled out**

- The loader does what #34 said: `deserialize_value_at_path` and `deserialize_value_from_reader` run `serde_ignored` and fail on any unknown key for the user file, a custom deployment file and the embedded deployment (`utils/src/yaml.rs:29-104`, `nodes/node/binary/src/main.rs:54-87`, `config/deployment/mod.rs:45-50`). A missing key on a defaulted field is never reported anywhere; nothing logs which fields were defaulted.
- Every checked-in deployment carries `faucet_pk`: the embedded `settings.yaml:118` and the devnet and testnet templates (`deployment-template.yaml:100`) with the all-zero key, the standalone template with a real key (`:100`), and `nodes/node/standalone-deployment-config.yaml:142` with the key of its `u64::MAX` genesis note (`:114-115`). The ceremony tool always writes the key (`tools/blockchain-tools/src/bin/genesis.rs:260-262`, `:395-408`) and validates the result by deserialising it (`:472-482`), which cannot detect an absent defaulted key. The test deployment builder sets `faucet_pk: None` (`tools/config/src/deployment.rs:169`) with a genesis that has no faucet note, which is consistent. Checklist item 4 is therefore clean at this commit; the exposure is third-party or hand-edited files, as #34 said.
- The all-zero `faucet_pk` in the devnet and testnet templates is a placeholder the ceremony overwrites; in the embedded deployment it stays, and no embedded genesis note has the zero key, so the exclusion is a no-op there.
- Generated user files carry every defaulted key. `config init` serialises the whole `UserConfig` (`nodes/node/binary/src/cli/config/init.rs:55`), and none of the fields in S-001's first class has `skip_serializing_if`, so `max_tx_fee`, `abstain_on_failure`, `declaration_id: null` and the `network`/`cryptarchia.network` sections are all written out. `config init` also copies `initial_peers` into `cryptarchia.network.bootstrap.ibd.peers` unless `--skip-ibd` is passed (`cli/config/update.rs:143-153`), so LB-002 needs a hand-written file.
- `max_tx_fee` (leader and SDP wallets, `Value::MAX`) and the `network` section are unchanged since #34: `config/cryptarchia/serde/leader.rs:13-24`, `config/sdp/serde.rs:23-24, 43-45`, `config/network/serde/mod.rs:16-30, 62-75`, `libp2p/src/config/mod.rs:25`. They are #474 and #488 and are only classified here.
- The API binds to loopback by default (`config/api/serde.rs:39-53`), CORS is empty, the tokio console and the OTLP tracing and metrics layers default to `None` (`tracing/serde/console.rs:8-12`, `tracing.rs:6-11`, `metrics.rs:6-11`), PoW auto-claim is off with no targets (`services/pow/src/service.rs:223-235`), the KMS key map is empty and Blend fails loudly without its keys (`config/mod.rs:112-134`). These are secure defaults and are not findings.
- The SDP `declaration_id` default (`None`) does not cause a new declaration: nothing posts one without an explicit `PostDeclaration` message (`services/sdp/src/lib.rs:288-303`). Its effect is S-003.
- The genesis transfer's output sum is only overflow-checked on the balance path (`Outputs::amount`, `core/src/mantle/ledger.rs:183-190`), which genesis is exempt from (`bedrock-genesis-block.md` §Genesis Validation Exemptions 2); `Outputs::validate` (`:165-172`) checks only for zero-value notes. So a `u64::MAX` faucet note plus any stake is a valid genesis, as the standalone file shows.

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Spec deviation: genesis total stake excludes the faucet note by an unbound, defaultable config key; with the `u64::MAX` faucet note an omitted `faucet_pk` wraps the stake sum and isolates the node | Consensus | Low | High | Open |
| LB-002 | An omitted `cryptarchia.network.bootstrap.ibd.peers` silently skips Initial Block Download; the node goes Online after the bootstrap period on whatever gossip delivered | Configuration | Low | High | Open |

### LB-001 · Spec deviation: genesis total stake excludes the faucet note by an unbound, defaultable config key; with the `u64::MAX` faucet note an omitted `faucet_pk` wraps the stake sum and isolates the node

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Consensus |
| Target | `ledger/src/cryptarchia/mod.rs:738-752` (`from_utxos`, the `sum::<Value>()` and the `faucet_pk` filter); `ledger/src/config.rs:15-16`, `nodes/node/binary/src/config/cryptarchia/deployment.rs:30-31` (the defaults); `tools/blockchain-tools/src/genesis/distribution.rs:29-53` (the faucet note); `deployment/ceremony/genesis/{standalone,testnet}/faucet.yaml` (`funds: 18446744073709551615`); `nodes/node/standalone-deployment-config.yaml:114-115, 142`; `services/chain/chain-leader/src/leadership.rs:444-449` |
| Status | Open |

**Description**

Three facts combine. First, the genesis ceremony gives the faucet one note worth `funds` from `faucet.yaml`, and both checked-in faucet files set `funds` to `u64::MAX` (`distribution.rs:41-53`; the standalone genesis block carries that note at `standalone-deployment-config.yaml:114-115`). Second, the ledger computes the genesis total stake as

```rust
// ledger/src/cryptarchia/mod.rs:743-749
let total_stake = utxos
    .utxos()
    .iter()
    .filter(|(_, (utxo, _))| config.faucet_pk.is_none_or(|fpk| utxo.note.pk != fpk))
    .map(|(_, (utxo, _))| utxo.note.value)
    .sum::<Value>()
    .max(1);
```

where `Value` is `u64` and `sum` is unchecked (`arithmetic_side_effects` allowed, `overflow-checks` off in release). Third, `faucet_pk` is `#[serde(default)]` on both the deployment `Settings` (`deployment.rs:30-31`) and the ledger `Config` (`config.rs:15-16`), and it is not in the genesis block: `bedrock-genesis-block.md` §Cryptarchia Parameters allows exactly `chain_id`, `genesis_time` and `genesis_epoch_nonce` in the inscription, "with no trailing bytes".

A node whose deployment file omits the key therefore includes the faucet note in the sum. In a release build the sum wraps: with the test's 100 T + 200 T + `u64::MAX` it yields 299,999,999,999,999 instead of 300,000,000,000,000, one token less than every correctly configured node. In a debug build it panics at genesis. The lottery constants `lottery_0`, `lottery_1` are functions of the total stake (`:750-752`), and they are public inputs of every Proof of Leadership (`try_apply_proof`, `:503-510`; `cryptarchia-proof-of-leadership.md` §Circuit Public Inputs 3), so every block the node receives fails `InvalidProof`, and every block it proposes fails on every other node. The epoch-to-epoch stake inference starts from this value (`update_epoch_state`, `:304-307`), so the difference never heals. #34 LB-002 (#475) described a stricter threshold and a slow fork; the actual effect on the shipped genesis files is immediate and total isolation.

The mechanism is also a deviation from the specification, independent of the default. §Initial Epoch State of `bedrock-genesis-block.md` sets `D` to "the total tokens distributed at genesis" and the eligible-leader commitment to the root over all genesis notes; the node's `D` excludes one note chosen by an out-of-band config value. The faucet note is in the aged UTXO tree on every node, so the only thing that keeps a `u64::MAX` note from winning every slot (`cryptarchia-proof-of-leadership.md` §Corner Case: Note Value Exceeding Inferred Total Stake) is that the leader of the node holding the faucet key skips it (`leadership.rs:444-449`, `services/wallet/src/lib.rs:294-296`); a modified leader with the key would not. For a testnet this is an accepted convention. For any genesis produced by the tool the faucet is mandatory: `--faucet` is a required argument (`genesis.rs:81-84`, `:236-237`) and `Outputs::validate` rejects a zero-value note (`core/src/mantle/ledger.rs:165-172`), so a mainnet ceremony run with this tool necessarily mints a faucet note and depends on the same exclusion.

Checklist item 3 asked whether `faucet_pk` can move into the genesis inscription. Not without a spec change: the inscription is fixed to three fields with no trailing bytes, and a fourth field would make every existing genesis block undecodable by the rule "must decode to exactly the three parameters". The value can be bound without touching the inscription: the parameter-set hash #175 proposes for `k`, `f`, the phase lengths and the rest is the right place, since `faucet_pk` is exactly the same kind of out-of-band consensus input. Or the concept can be removed from consensus: give the faucet an ordinary-sized note (it only needs to fund test accounts) so that no exclusion is needed and the node follows the spec's `D`.

**Exploit scenario**

Not attacker-triggered. An operator bootstraps a testnet or standalone node from a deployment file written by hand or by tooling other than `blockchain-tools`, without the `faucet_pk` line. The node starts, logs nothing about the missing key, computes a total stake one token below the network's, and rejects every block it receives with `InvalidProof` while producing blocks nobody accepts. The operator sees a node that never syncs. A deployment file distributed with the key stripped does the same to every node that uses it. In a debug build the node panics at genesis instead.

**Recommendation**

- *Short term*: remove `#[serde(default)]` from `deployment.rs:30` and `config.rs:15` so the key must be present, `faucet_pk: null` included; make the genesis sum `checked_add` (or `try_fold`) and fail genesis loudly on overflow; log the effective `faucet_pk` and the genesis `total_stake` in the startup banner next to the chain ID. Make `--faucet` optional in the ceremony tool.
- *Long term*: fold `faucet_pk` into the parameter-set hash of #175 so two nodes that disagree on it cannot peer; raise upstream (S-005) whether the spec should define a faucet or whether the node should stop excluding notes from `D` and give the faucet a note of ordinary size instead.

**References**: `bedrock-genesis-block.md` §Initial Token Distribution, §Cryptarchia Parameters, §Initial Epoch State; `cryptarchia-proof-of-leadership.md` §Circuit Public Inputs, §Corner Case: Note Value Exceeding Inferred Total Stake; #475 (#34 LB-002), #175, #106.

### LB-002 · An omitted `cryptarchia.network.bootstrap.ibd.peers` silently skips Initial Block Download; the node goes Online after the bootstrap period on whatever gossip delivered

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Configuration |
| Target | `nodes/node/binary/src/config/cryptarchia/serde/mod.rs:12-13`, `serde/network.rs:11-54` (`Config`, `BootstrapConfig`, `IbdConfig::default`, `peers: HashSet::new()`); `services/chain/chain-network/src/bootstrap/ibd.rs:131-139` (`run`, "Skipping IBD as no peers are configured"); `services/chain/chain-service/src/bootstrap/state.rs:11-27` (`choose_engine_state`), `src/service/phases/pbp.rs:66-113` (the one-hour timer); `nodes/node/binary/src/cli/config/update.rs:143-153` |
| Status | Open |

**Description**

The node has two peer lists. `network.backend.initial_peers` (multiaddrs the swarm dials) and `cryptarchia.network.bootstrap.ibd.peers` (peer IDs to fetch tips and blocks from during Initial Block Download). The second lives in `cryptarchia.network`, a `#[serde(default)]` section whose default is an empty set (`serde/network.rs:12-17, 43-54`). `config init` derives it from the first list (`update.rs:143-153`), but a user file written by hand, or one written for a node that was later given peers through `--net-initial-peers`/`NET_INITIAL_PEERS`, has the first list and not the second. With an empty set `InitialBlockDownload::run` logs a warning and returns without downloading anything (`ibd.rs:136-139`); the chain service starts in `Bootstrapping` because the LIB is genesis (`state.rs:17-18`), waits `prolonged_bootstrap_period` (default one hour, `serde/service.rs:46-54`) while applying whatever blocks arrive, then switches to Online (`pbp.rs:66-113`).

`p2p-network-bootstrapping.md` §Protocol step 1 has the operator configure bootstrap node addresses once; the node is meant to derive everything else. Here the same information must be given twice, and the omission of the second copy is reported as one `warn!` line among the startup output. Whether the node catches up during the hour depends on the orphan downloader being handed gossiped proposals whose ancestors it can fetch; on a chain more than `k` blocks long that is a long chain of single-parent fetches bounded by `max_orphan_cache_size` (1000), and after the hour the node commits to its tip under the Online rule and rejects any fork that diverges more than `k` blocks back (#39 LB-003), which is exactly what the honest chain looks like to a node that did not finish syncing. `--skip-ibd` exists (`config/mod.rs:281-284`) for operators who want this; an omitted key should not be a second way to get it.

**Exploit scenario**

Not attacker-triggered. An operator writes a user file with `network.backend.initial_peers` pointing at the bootstrap nodes and no `cryptarchia.network` section. The node starts, prints "Skipping IBD as no peers are configured", spends an hour applying gossiped blocks through the orphan path, and, if the chain is deeper than it managed to fetch, switches to Online on a stale tip and stays there, logging `ParentMissing` for every new block. Nothing distinguishes this from a healthy node except the height.

**Recommendation**

- *Short term*: when `ibd.peers` is empty and `initial_peers` is not, derive the IBD peer set from the peer IDs in `initial_peers` at config load time (the same code `config init` runs), and only skip IBD when both are empty or `--skip-ibd` was given; raise the "Skipping IBD" line to `error!` when `initial_peers` is non-empty.
- *Long term*: keep one peer list. The IBD set is a subset of the dialled peers by construction; make it a filter over `initial_peers` rather than a second list.

**References**: `p2p-network-bootstrapping.md` §Protocol; `cryptarchia-v1-bootstr-sync.md` §Setting the Fork Choice Rule (via #39 LB-003); #135, #140.

## 5. Suggestions (non-security)

### S-001 · Inventory and classification of the 115 `#[serde(default)]` sites, and the policy diff

| | |
|---|---|
| Target | every file in `nodes/node/binary/src/config/**`, `libp2p/src/config/**` and the service settings listed in §2 |

Checklist items 1 and 2. Classes: **K** tuning knob (a default is fine), **L** limit, **I** identity or key, **C** consensus input or security-bearing behaviour. Struct-level `#[serde(default)]` is listed once per struct with the fields its `Default` sets.

| Site | Field(s) and default | Class | Verdict |
|---|---|---|---|
| `config/mod.rs:59` `UserConfig.network` | whole section: `SwarmConfig::default()` incl. fresh `node_key`, `0.0.0.0:3000`, empty `initial_peers` | I | required; #488 |
| `config/mod.rs:63,66,68,70,75,77,79` `time`, `api`, `storage`, `kms`, `pow`, `tracing`, `state` | sections; see rows below | K | fine |
| `config/cryptarchia/serde/mod.rs:10,12` `service`, `network` | sections; see rows below | C | see LB-002 |
| `config/cryptarchia/serde/network.rs:12-125` `bootstrap.ibd.peers` empty, `tips_fetch_*`, `round_delay`, `orphan.max_orphan_cache_size` 1000, `max_rejected_cache_size` 1000, `tip_poll.enabled` true, `lag_threshold_blocks` 3, `max_peers_to_sample` 5, `max_*_peers_to_try_download` 16 | C (`ibd.peers`), L (caches), K | `ibd.peers`: LB-002; caches: log effective values |
| `config/cryptarchia/serde/service.rs:8-75` `sync.block_provider.batch_size` 1000, `bootstrap.prolonged_bootstrap_period` 1 h, `force_bootstrap` false, `offline_grace_period.grace_period` 20 min, `state_recording_interval` 1 min | C (bootstrap), L | defaults are the intended ones; log them at startup, since they decide when the Genesis rule applies (#42) |
| `config/cryptarchia/serde/leader.rs:13` `max_tx_fee` = `Value::MAX` | L | required; #474 |
| `config/cryptarchia/deployment.rs:30` `faucet_pk` = `None` | C | required; LB-001, #475 |
| `config/sdp/serde.rs:14` `declaration_id` = `None` | I | keep optional, log; S-003 |
| `config/sdp/serde.rs:17,29-41` `active_message_tracker.status_check_interval_in_tip_changes` 3 | K | fine |
| `config/sdp/serde.rs:23` `max_tx_fee` = `Value::MAX` | L | required; #474 |
| `config/blend/serde/mod.rs:23` `edge` section; `edge.rs:6-27` `max_dial_attempts_per_peer_per_message` 1, `replication_factor` 1 | K | fine |
| `config/blend/serde/mod.rs:25` `abstain_on_failure` = `false` | C | privacy-bearing; S-002 |
| `config/blend/serde/core.rs:10-61` `backend`: `0.0.0.0:3400`, `core_peering_degree` 3..=5, `edge_node_connection_timeout` 1 s, `max_edge_node_incoming_connections` 300, `max_dial_attempts_per_peer` 3, `peering_degree_check_interval` 1 min | L, K | fine; log the limits |
| `config/network/serde/mod.rs:16,22,30` `Config`, `BackendSettings`, `SwarmConfig` | I (`node_key`), K | #488 |
| `config/network/serde/gossipsub.rs:9` all `libp2p-gossipsub` defaults, incl. `validate_messages` false, `gossip_factor` 0.25 | C (`validate_messages`), K | forward-before-validate is #144's; `gossip_factor` is #198's |
| `config/network/serde/kademlia.rs:6`, `identify.rs:4` (`hide_listen_addrs: Some(true)`), `chainsync.rs:9` (`peer_response_timeout` 5 s, `max_inbound_requests` 10), `nat.rs:57-124` (traversal on: autonat, UPnP/NAT-PMP mapping, gateway monitor) | K, L | fine |
| `config/api/serde.rs:10,17` `127.0.0.1:8080`, `cors_origins` empty, `timeout` 30 s, `max_body_size` 10 MiB, `max_concurrent_requests` 500 | L | secure defaults; fine |
| `config/kms/serde.rs:7,13` `keys` empty | I | fails loudly downstream; fine |
| `config/wallet/serde.rs:9,12` `known_keys` empty, `pending_note_expiry_blocks` 10 | K | fine |
| `config/pow/serde.rs:8,13` `mining` (all CPUs, 4 tickets/block), `auto_claim` off | K | fine |
| `config/storage/serde.rs:4,10` `./db`, `read_only` false, `column_family` `blocks` | K | fine |
| `config/time/serde.rs:11,18,41` `server` `pool.ntp.org:123`, `update_interval` 15 s, `timeout` 5 s, `listening_interface` `0.0.0.0` | C (clock source) | S-004 |
| `config/state.rs:6` `./state` | K | fine |
| `config/tracing/serde/**` (13 sites) `level` INFO, file log in `.` + stdout, OTLP/metrics/console `None`, `sample_ratio` 0.5, `authorization_header` `None`, GELF `0.0.0.0:9000` when enabled | K | `sample_ratio` is #198's; GELF's `0.0.0.0` only applies once the layer is configured |
| `config/mempool/deployment.rs:15` `tx_ttl` 24 h | L | fine; deployment-level |
| `libp2p/src/config/mod.rs:25` `node_key` generated | I | #488 |
| `libp2p/src/config/mod.rs:31,40,44,51`, `kademlia.rs` (8), `identify.rs` (5), `nat/*.rs` (9) | K | fine |
| `services/chain/chain-network/src/sync/config.rs:13,20,38`, `services/chain/chain-service/src/bootstrap/config.rs:12,21,25` | mirrors of the node rows | C, L | as above |
| `services/pow/src/service.rs:170,174,230,233,252,255` | mirrors of `config/pow` | K | fine |
| `services/wallet/src/lib.rs:368`, `services/sdp/src/state.rs:16` (`pending_activity`), `services/api/src/http/pow.rs:106` (request body), `services/tx-service/src/backend/pool.rs:34`, `services/network/src/backends/libp2p/config.rs:8`, `command.rs:44-53`, `mock.rs:70` | K, or recovery-state and API-request defaults that are not config | fine |

The policy the classification implies, as a diff against the node crate (the two `max_tx_fee` sites are #474's and are included for completeness):

```diff
--- a/nodes/node/binary/src/config/cryptarchia/deployment.rs
+++ b/nodes/node/binary/src/config/cryptarchia/deployment.rs
@@
     pub genesis_block: GenesisBlock,
-    #[serde(default)]
     pub faucet_pk: Option<ZkPublicKey>,
     pub pow_config: PoWConfig,
--- a/ledger/src/config.rs
+++ b/ledger/src/config.rs
@@
     pub sdp_config: crate::mantle::sdp::Config,
-    #[serde(default)]
     pub faucet_pk: Option<ZkPublicKey>,
--- a/nodes/node/binary/src/config/cryptarchia/serde/leader.rs
+++ b/nodes/node/binary/src/config/cryptarchia/serde/leader.rs
@@
-    #[serde(default = "default_max_tx_fee")]
     pub max_tx_fee: GasCost,
--- a/nodes/node/binary/src/config/sdp/serde.rs
+++ b/nodes/node/binary/src/config/sdp/serde.rs
@@
-    #[serde(default = "default_max_tx_fee")]
     pub max_tx_fee: GasCost,
--- a/nodes/node/binary/src/config/mod.rs
+++ b/nodes/node/binary/src/config/mod.rs
@@ pub fn build_run_config
+    // LB-002: one peer list. Derive the IBD set from the dialled peers when
+    // the operator did not give one, and only skip IBD on an explicit request.
+    if user.cryptarchia.network.bootstrap.ibd.peers.is_empty()
+        && !user.network.backend.initial_peers.is_empty()
+        && !cli_args.cryptarchia.skip_ibd
+    {
+        user.cryptarchia.network.bootstrap.ibd.peers =
+            peer_ids_of(&user.network.backend.initial_peers);
+    }
+    // Effective security-bearing values, once, at startup.
+    info!(
+        chain_id = %deployment.chain_id(),
+        faucet_pk = ?deployment.cryptarchia.faucet_pk,
+        leader_max_tx_fee = ?user.cryptarchia.leader.wallet.max_tx_fee,
+        sdp_max_tx_fee = ?user.sdp.wallet.max_tx_fee,
+        ibd_peers = user.cryptarchia.network.bootstrap.ibd.peers.len(),
+        blend_abstain_on_failure = user.blend.abstain_on_failure,
+        sdp_declaration_id = ?user.sdp.declaration_id,
+        ntp_server = %user.time.backend.server,
+        "effective configuration"
+    );
```

`config init` already writes every one of these keys, so the change costs nothing for generated files and turns a silent omission in a hand-written one into a parse error naming the field. A test that deserialises each checked-in deployment file and the embedded one after the change would catch a stripped key in CI.

### S-002 · `blend.abstain_on_failure` defaults to the privacy-degrading side and is not logged

| | |
|---|---|
| Target | `nodes/node/binary/src/config/blend/serde/mod.rs:25-26`, `config/mod.rs:264-269` (CLI `default_value_t = false`); `services/blend/src/core/mod.rs:465-472`, `src/edge/mod.rs:362-372` |

When the key is absent the Blend failure detector is enabled, and a proposal that Blend fails to deliver in time is broadcast by the node itself ("`None` when the operator has turned the fallback off, which records nothing, watches nothing and can reveal nothing", `edge/mod.rs:362-363`). That is the intended liveness-first default and is documented on the CLI flag, but it means an omitted key selects the setting under which a leader can reveal its own block. The mechanism's own problems are #576 and #588; the suggestion here is only to log the effective value at startup (S-001's banner) and to state in the config template that `false` is the deanonymising side.

### S-003 · `sdp.declaration_id` defaults to `None` and the node then forgets it is a provider unless its state file survives

| | |
|---|---|
| Target | `nodes/node/binary/src/config/sdp/serde.rs:12-15`; `services/sdp/src/lib.rs:177-180`, `:465-467` |

The service resolves its declaration as `state.declaration_id` or, failing that, `settings.declaration_id` (`lib.rs:177-180`). With the key omitted and the state file absent (fresh host, restored from a config-only backup), the node runs with no declaration, sends no activity messages and drops out of the Blend membership after the inactivity period, with its stake still locked, while the SDP ledger has its declaration in plain view. The service could look its own declaration up by `provider_id` (the Blend signing key it already resolves in `config/mod.rs:112-123`) at startup and warn when the ledger has one the config does not mention.

### S-004 · The consensus clock source defaults to a public NTP pool when the `time` section is omitted

| | |
|---|---|
| Target | `nodes/node/binary/src/config/time/serde.rs:11-37` |

`cryptarchia-v1-protocol.md` §Clocks says the protocol relies on NTP-synchronised clocks. An omitted `time` section selects `pool.ntp.org:123` over plain NTP, which is fine for a testnet and a choice an operator should see: log the server in the startup banner and document that the slot clock follows it.

### S-005 · Spec: the genesis has no faucet, the node's genesis has a mandatory one

| | |
|---|---|
| Target | `bedrock-genesis-block.md` §Initial Token Distribution, §Initial Epoch State; `tools/blockchain-tools/src/bin/genesis.rs:81-84` |

LB-001's deviation should be settled on the spec side. Either the spec defines a faucet allocation and states that `D` excludes it (and then the exclusion key belongs in the inscription or the #175 parameter-set hash), or it does not and the node should compute `D` over every genesis note, which the ceremony tool can accommodate by making `--faucet` optional and by giving the faucet a note sized like any other stakeholder's.

### S-006 · Check the genesis output sum for overflow at genesis validation

| | |
|---|---|
| Target | `core/src/mantle/ledger.rs:165-172` (`Outputs::validate`), `:183-190` (`Outputs::amount`); `ledger/src/cryptarchia/mod.rs:743-749` |

The exemption that skips the balance check at genesis (`bedrock-genesis-block.md` §Genesis Validation Exemptions 2) also skips the only overflow check on the outputs, so a genesis whose notes sum past `u64::MAX` is accepted and every later `sum` over them is a wrapping addition. Calling `Outputs::amount` in `validate` for the genesis transfer, or a `checked_add` fold in `from_utxos`, makes such a genesis a load-time error.

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
