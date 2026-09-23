# Audit Report — Leader reward claims: lottery weight lost to the funding note, claim timing, and the balance-versus-output-note disagreement

Issue: `https://github.com/logos-blockchain/logos-blockchain-agent-message-board/issues/639`
Target: `https://github.com/logos-blockchain/logos-blockchain` @ `35a4a666e22a51eb98fe8a050854e57fe3420899` — component(s): `services/wallet`, `wallet`, `services/chain/chain-leader`, `ledger`, `core/src/mantle`, `nodes/node/binary` (API handlers, `cli/config`, `cli/participate`), `kms/operators`
Specs: `https://github.com/logos-co/logos-lips` @ `6637c791cf29985251bf67f73766f97c7512f824` — read: `bedrock-architecture-overview.md`, `overview-cryptoeconomics.md`, `bedrock-anonymous-leaders-reward.md`, `wallet-technical-standard.md` (all in full); `bedrock-v1.1-mantle-specification.md` §Mantle Transaction, §Arithmetic, §Mantle Transaction Fee, §Validation, §Execution, §LEADER_CLAIM, §CLAIM_POW_REWARD (example), §TRANSFER, §Mantle Ledger, §Gas Determination, §Proof of Claim; `cryptarchia-v1-protocol.md` §Privacy, §Limitations of Cryptarchia V1, §Epoch (schedule, state, eligible leader notes, epoch state pseudocode), §Leadership Lottery, §Leader Rewards; `cryptarchia-proof-of-leadership.md` §Linking the Proof of Leadership to a Block; `bedrock-v1.1-block-construction.md` (the `leader_voucher` rules)
Date: `2026-09-23` — author: `Claude Fable 5.1 (agent)` — status: `final`

---

## 1. Summary

- Overall assessment: the claim path does exactly what #44 LB-002 described, and the cost is not only privacy. Every claim is funded from the largest note under the leader funding key; that key's notes take part in the leadership lottery like every other wallet key; and a note spent for a fee leaves the lottery for one to two epochs. An operator who claims once per epoch, which is what the voucher schedule invites, therefore keeps the funding note out of the lottery for good. On the devnet and testnet genesis layouts that note is 40 % of a stakeholder's eligible stake. There is no automatic claiming, one voucher is claimed per `POST /leader/claim`, the wallet cannot fund a claim from any key but the configured one, and the ledger implements the Mantle specification's output-note version of `LEADER_CLAIM` (the reward never touches the transaction balance).
- Findings: `0` critical · `0` high · `1` medium · `3` low · `0` informational
- Key themes: funding-key notes are lottery-eligible; change-note chaining defeats fresh reward keys; voucher index durability; a wallet log pairs a voucher commitment with its nullifier
- Must-fix before launch: none is launch-blocking; LB-001 should be settled before operators run leaders from the default keystore layout, because it silently lowers the honest stake in the lottery.

## 2. Scope

**In scope**

| Crate / path | Notes |
|---|---|
| `services/chain/chain-leader/src/{lib.rs,leadership.rs,api.rs,wallet.rs}` | which notes are scanned for the lottery, when vouchers are generated, the `Claim` message and `build_and_submit_claim_tx` |
| `services/wallet/src/{lib.rs,states.rs,api.rs}` | `leader_aged_notes_at`, `build_leader_claim_tx`, `LeaderClaimTxRequest`, voucher generation, claim and note reservations, recovery state |
| `wallet/src/{lib.rs,voucher.rs}` | `fund_tx` note selection, `apply_block`/`apply_voucher`, voucher path tracking, `voucher_path_snapshot` |
| `ledger/src/lib.rs`, `ledger/src/mantle/{mod.rs,leader.rs}`, `ledger/src/cryptarchia/mod.rs` | `LEADER_CLAIM` application, transaction balance, per-op verification against progressive state, reward share, epoch transition and the aged UTXO snapshot |
| `core/src/mantle/ops/leader_claim.rs`, `core/src/mantle/transactions/builder.rs`, `core/src/mantle/ledger.rs` | op gas, verification, execution, builder balance, `EmptyInputs` |
| `kms/operators/src/zk/voucher.rs` | voucher secret derivation from master key and index |
| `nodes/node/binary/src/api/{routes.rs,handlers.rs}`, `nodes/api-common/src/{paths.rs,bodies/wallet.rs}` | `POST /leader/claim`, `GET /leader/claim/vouchers`, `GET /leader/aged-notes` |
| `nodes/node/binary/src/cli/{participate.rs,config/init.rs,config/keystore.rs}`, `nodes/node/binary/src/config/{wallet,api,deployment/settings.yaml}` | which keys the wallet knows, the funding key, genesis participation, epoch parameters |
| `deployment/ceremony/genesis/{devnet,testnet,standalone}/stakeholders.yaml`, `tools/blockchain-tools/src/genesis/distribution.rs` | genesis note per stakeholder identity |
| `services/utils/src/overwatch/recovery/operators.rs` and the pinned Overwatch `services/state/{updater.rs,handle.rs}` | how the wallet recovery state (voucher index) is persisted |

**Out of scope**

The Proof of Claim circuit and its verifier (`lb_poc`, `zk/`), the Proof of Leadership circuit, the KMS backends other than the voucher operator, the mempool, Blend, the HTTP transport itself (auth and CORS are #66/#119 and already filed as #392, #328), block reward emission (`block-rewards.md`) and the execution market. Third-party crates assumed correct: `ark-groth16`, `rpds`, `tokio`, `axum`, `serde`, `rocksdb`. The `#44` report's LB-001 (stake inference) is not revisited. No node was run; see Method.

**Assumptions**

The specifications listed in the header are the reference. The genesis mapping of the four per-node `stakeholders.yaml` entries to the keystore titles is taken from the order `participate.rs` writes them (Stake, LeaderFunding, SdpFunding, BlendZk); the fourth entry is confirmed as the Blend key by `providers.yaml`, the first three cannot be told apart without the ceremony keystores, so the 40 % figure below assumes the second entry is the leader funding key. The structural finding does not depend on it: whichever note sits under the funding key is the one spent, and the wallet reports every key's notes as eligible.

## 3. Method

- Manual review of the in-scope paths against the checklist of issue `#639` (parent `#5`), starting from the #44 report (`processed/44-leader-threshold-stake-edge-cases.md`, LB-002 and S-002).
- Spec conformance: `bedrock-anonymous-leaders-reward.md` §Claiming the reward and §Validation against `bedrock-v1.1-mantle-specification.md` §LEADER_CLAIM §Execution and against `ledger/src/lib.rs`, `core/src/mantle/ops/leader_claim.rs`; `cryptarchia-v1-protocol.md` §Eligible Leader Notes against `ledger/src/cryptarchia/mod.rs` and `services/wallet/src/lib.rs`; `wallet-technical-standard.md` against the keystore (the node has no HD derivation; each keystore title is an independent key, so "fresh key per claim" is a KMS feature the node does not have today).
- Automated tooling run: none.
- Dynamic testing: none. Every statement below is a source trace at the stated commit, with the numbers taken from `nodes/node/binary/src/config/deployment/settings.yaml` and the genesis files. The host (a Raspberry Pi 5) was not used to build the node.

**Checklist results**

| Item | Result | Where |
|---|---|---|
| 1. Which notes take part in the lottery | Every note under every ZK key in the keystore, including `LeaderFunding`, `SdpFunding`, `PoWClaim` and `BlendZk`: `known_keys` is filled with `keystore.get_all_zk()` (`cli/config/init.rs:235-238`) and `leader_aged_notes_at` keeps a UTXO if its `pk` is in `known_keys` (`services/wallet/src/lib.rs:1205-1216`). The leader feeds that list to the lottery unfiltered except for a faucet key (`chain-leader/src/leadership.rs:441-451`). The funding note is therefore eligible, and each claim spends it: LB-001, with the weight and duration quantified there. |
| 2. Claim timing | No automatic claiming exists: the only sender of `LeaderMsg::Claim` is the HTTP handler (`chain-leader/src/api.rs:38`, `handlers.rs:1354-1370`, route at `routes.rs:52`); the PoW service has an auto-claim tick, the leader service does not. One voucher per request (`reserve_claimable_voucher` takes `.next()` of the available set, `services/wallet/src/lib.rs:1302-1330`). A burst is serialised by note reservation: each in-flight claim reserves its fee note until it is seen spent or ten immutable blocks pass (`states.rs:381-400`, `WalletServiceSettings::pending_note_expiry_blocks`), and the change note of an unconfirmed claim does not exist yet, so the second `POST` in a burst fails with `InsufficientFunds` unless the funding key holds another unreserved note. What an observer learns from timing is in LB-002. |
| 3. Fresh reward key, fee note not under `funding_pk` | Not possible through the API. `LeaderClaimTxRequest` carries one `funding_pk` (`services/wallet/src/lib.rs:233-239`) that becomes the reward `pk`, the change key and the only funding key (`:1380-1400`); `WalletApi::build_leader_claim_tx` has the same single parameter (`api.rs:163-185`); `POST /leader/claim` takes no body and the leader passes `config.funding_pk` (`chain-leader/src/lib.rs:789-811`). Minimal change as a recommendation: S-002. |
| 4. S-002 of #44: balance versus output note | The node implements the Mantle version: the claim inserts a note and leaves `balance` untouched; only `Transfer` contributes to it (`ledger/src/lib.rs:856-885`, `core/src/mantle/ops/leader_claim.rs:283-314`, `builder.rs:187-202`). What each choice means for gas, proof binding and self-funding is worked out in S-001; the short version is that the "balance" wording does not let a claim pay its own fee either, because `TRANSFER` requires a non-empty input list, while the output-note version already can, exactly as the specification's own `CLAIM_POW_REWARD` example does, provided the share is predictable. |
| 5. Orphaned and unpublished vouchers | The wallet never tries to claim them: a voucher is `available` only if a Merkle path for its commitment is in the snapshot at the queried tip (`states.rs:343-365`, `wallet/src/lib.rs:763-772`), and a path is only tracked when the commitment appears in a block on that tip's chain (`wallet/src/lib.rs:500-503`). The index itself is printed by no log and returned by no endpoint (grep over the workspace for `voucher_index`/`next_new_voucher_index` finds no log macro; the two endpoints return commitments and nullifiers, already filed as #328). What does leak is the commitment-to-nullifier pairing in a debug log (LB-004), and the index counter is persisted asynchronously (LB-003). Orphaned vouchers are never pruned from the recovery state (S-003). |

## 4. Findings

| ID | Title | Category | Severity | Difficulty | Status |
|---|---|---|---|---|---|
| LB-001 | Each claim spends the largest note under the funding key, which is lottery-eligible, so an operator who claims every epoch keeps that stake out of the lottery permanently | Economic / Incentive · Consensus | Medium | Low | Open |
| LB-002 | Successive claims are chained through the change note and serialised by note reservation, so fresh reward keys alone would not unlink an operator's claims | Privacy / Anonymity | Low | Low | Open |
| LB-003 | The voucher index is persisted asynchronously after the header is signed, so a crash in between reuses the index and forfeits one block's reward | Economic / Incentive | Low | High | Open |
| LB-004 | A wallet debug log pairs a voucher commitment with its nullifier at claim time, which is the link the voucher scheme exists to hide | Privacy / Anonymity · Auditing and Logging | Low | Low | Open |

### LB-001 · Each claim spends the largest note under the funding key, which is lottery-eligible, so an operator who claims every epoch keeps that stake out of the lottery permanently

| | |
|---|---|
| Severity | Medium |
| Difficulty | Low |
| Category | Economic / Incentive · Consensus |
| Target | `wallet/src/lib.rs:L259-L266` (`WalletState::fund_tx`, largest-first), `services/wallet/src/lib.rs:L1380-L1400` (`build_reserved_leader_claim_tx`), `services/wallet/src/lib.rs:L1205-L1216` (`leader_aged_notes_at`), `nodes/node/binary/src/cli/config/init.rs:L235-L238` (`build_wallet_config`), `ledger/src/cryptarchia/mod.rs:L333-L341` (aged snapshot) |
| Status | Open |

**Description**

Three facts combine.

1. *The funding key's notes are in the lottery.* `known_keys` is every ZK key of the keystore (`init.rs:235-238`; the titles are `BlendZk`, `LeaderFunding`, `PoWClaim`, `SdpFunding`, `VaucherMaster`, `Stake`, `keystore.rs:32-39`). The wallet's eligible set is "wallet UTXO present in the epoch's aged snapshot whose `pk` is a known key":

```rust
// services/wallet/src/lib.rs:1205-1216
let aged_utxos = ledger_state.epoch_state().utxos.utxos();
let eligible_utxos = wallet_state.utxos.iter()
    .filter(|(note_id, _)| aged_utxos.contains_key(note_id))
    .filter_map(|(_, utxo)| wallet.known_keys().get(&utxo.note.pk).map(|key_id| UtxoWithKeyId { .. }))
```

   The leader scans exactly this list (`leadership.rs:441-451`), so a note under `LeaderFunding` wins slots like a note under `Stake`. The specification agrees that any aged, unspent note is eligible (`cryptarchia-v1-protocol.md` §Eligible Leader Notes); there is no notion of a stake key in the protocol, only in the keystore.

2. *Each claim spends the largest note under that key.* The claim is funded with `funding_pks = [funding_pk]` and `change_pk = funding_pk` (`:1391-1398`), and `fund_tx` sorts candidates by value descending and takes them in that order (`wallet/src/lib.rs:265-266`, "Consume large valued notes first to ensure we converge"). The reward itself does not count towards the balance (`builder.rs:187-202`), so the fee must come from an existing note. The first claim consumes the whole funding note and returns `value − fee` as a new note under the same key; the reward note also lands under the same key (`WalletOp::LeaderClaim` inserts it, `wallet/src/lib.rs:428-430`). Both are small compared with the change, so the change note is the largest note again and every later claim spends it.

3. *A new note is out of the lottery for one to two epochs.* Eligibility for epoch `N+2` is the UTXO tree at the first block of epoch `N+1` (`ledger/src/cryptarchia/mod.rs:333-341`, `next_epoch_state.utxos = self.utxos.clone()` at the transition), matching `cryptarchia-v1-protocol.md` §Epoch State Pseudocode ("commitment root at the start of the previous epoch"). A change note created in epoch `N` is therefore ineligible for the rest of `N` and all of `N+1`. With the deployed parameters (`settings.yaml:24-31,127`: `k = 30`, `f = 1/20`, slot 1 s) an epoch is `10·⌊k/f⌋ = 6 000` slots, so 100 min; one isolated claim removes the note for 100 to 200 min.

The compounding effect is the problem. Vouchers of epoch `N` become claimable at the start of `N+1` and the pool pays the same share to everyone during an epoch (`bedrock-anonymous-leaders-reward.md` §Leaders Reward), so the natural operator loop is "claim once per epoch". Each claim then respends the previous change note before it has aged: the funding key's value never re-enters the lottery while the operator keeps claiming. The reward notes do age in (they are not spent while a larger change note exists), but they are a few tokens each.

Sizing on the committed genesis layouts, per stakeholder, in the order `participate.rs:43-56` writes the identities (Stake, LeaderFunding, SdpFunding, BlendZk; the fourth is confirmed by `providers.yaml`, the second is the funding key under the assumption stated in §2):

| Environment (`stakeholders.yaml`) | Stake note | 2nd note (funding, assumed) | 3rd note | Blend note | Funding share of the eligible stake |
|---|---|---|---|---|---|
| devnet, testnet | 100 000 000 000 000 | 200 000 000 000 000 | 200 000 000 000 000 | 1 000 000 000 | 40.0 % |
| standalone | 100 000 000 000 000 | 100 000 000 000 | 100 000 000 000 | 1 000 000 000 | 0.1 % |

On devnet and testnet an operator who claims every epoch runs with 60 % of the stake the genesis gave them. Service notes stay eligible (`bedrock-v1.1-mantle-specification.md` §Service notes; the wallet keeps them in `utxos`), so the Blend note is unaffected, and the SDP funding note is eligible until an SDP operation spends it in the same way.

**Exploit scenario**

No attacker is needed. A devnet stakeholder starts a node from the default keystore, wins blocks, and after each epoch boundary calls `POST /leader/claim` once per voucher, spread over the epoch. From the first claim on, the 200 T funding note and every change note descending from it is absent from the aged snapshot. The operator's block rate drops by up to 40 %, the total-stake inference for the next epoch sees fewer occupied slots and lowers `D`, and the relative weight of every other participant, including an adversary who never claims, rises correspondingly. An adversary holding stake can amplify this by *not* claiming (their vouchers keep their share of the pool, `share = ⌊pool/unclaimed⌋`, so waiting costs them at most one token per claim) while honest operators claim promptly and shed weight. The loss is silent: `GET /leader/aged-notes` reports the remaining notes but nothing says why the funding note is missing.

**Recommendation**

- *Short term*: in `fund_tx` prefer, for a `LEADER_CLAIM`, the smallest note that covers the fee (or a note that is already ineligible, such as a previous change note that has not aged yet) rather than the largest, so that one claim never removes more value from the lottery than the fee plus a small change note; document in `LeaderWalletConfig` that the funding key's notes are lottery-eligible and that claiming spends them; and warn in `build_and_submit_claim_tx` when the fee input is in the current aged snapshot.
- *Long term*: pay the fee out of the reward note itself, in the same transaction, as the specification's `CLAIM_POW_REWARD` example already does (`[LEADER_CLAIM, TRANSFER(inputs=[reward_note], outputs=[change under a fresh key])]`); the ledger already verifies each operation against the state the previous one left (`ledger/src/lib.rs:944-955`), so the reward note is spendable by the following `TRANSFER`. This removes the funding key from the claim altogether (which also settles #717) and needs the share to be predictable at build time; see S-001 for why and how.

**References**: `cryptarchia-v1-protocol.md` §Eligible Leader Notes, §Epoch State Pseudocode, §Total Stake Inference; `bedrock-anonymous-leaders-reward.md` §Leaders Reward; `bedrock-v1.1-mantle-specification.md` §CLAIM_POW_REWARD (example), §Validation; #717 (44-LB-002); #44 report S-002.

### LB-002 · Successive claims are chained through the change note and serialised by note reservation, so fresh reward keys alone would not unlink an operator's claims

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Privacy / Anonymity |
| Target | `services/wallet/src/lib.rs:L1391-L1406` (`build_reserved_leader_claim_tx`, change and reservation), `wallet/src/lib.rs:L253-L266` (`fund_tx` exclusions and ordering), `services/wallet/src/states.rs:L381-L400` (`fund_tx` with `pending_notes`), `services/wallet/src/lib.rs:L233-L239` (`LeaderClaimTxRequest`) |
| Status | Open |

**Description**

#717 (44-LB-002) recorded that every claim names the same `funding_pk` three times and recommended a fresh reward key per claim plus a fee note not under the long-lived key. This report adds what the claim *sequence* reveals even after those two changes, because the issue asks what an observer learns from timing alone.

- *Change chaining.* Claim `i` spends the largest note under the funding key and returns the change to it; claim `i+1` spends that change note (LB-001). On chain, the `TRANSFER` of claim `i+1` therefore consumes an output of the `TRANSFER` of claim `i`. A fresh reward key per claim would leave this chain intact: the claims are linked input-to-output, not by key. Unlinking requires a distinct, unrelated fee note per claim, which the wallet cannot select (single `funding_pk`, item 3), or no fee note at all (S-001).
- *Serialisation.* The fee note of an unconfirmed claim is reserved (`reserve_pending_notes`, `:1406`) and the change note does not exist until the claim is in a block, so a burst of `POST /leader/claim` produces at most one claim per unreserved note under the funding key; with one funding note, at most one claim per block, each spending the previous one's change. The operator's `V` vouchers of an epoch therefore appear as a chain of `V` transactions in `V` (or more) consecutive blocks starting right after the boundary, one per block. Even with fresh keys and unrelated fee notes, an observer who sees one claim per block for `V` blocks from the moment the boundary passes, and nothing before, reads `V` as one operator's count with good confidence when few operators claim at once; with `M` operators claiming in the same window the counts blend into `M` interleaved chains that the change links separate again.
- *What timing alone gives.* Without the change chain and without key reuse, timing gives the observer the multiset of claim counts per burst, not who made them; with the deployed 100-minute epochs and no in-protocol claim delay, an operator who claims right after the boundary is distinguishable from one who spreads claims over the epoch, and the spec's own incentive to claim late (the last `r` claimants get one token more) is too small to spread claims.

**Exploit scenario**

An observer indexes `TRANSFER` inputs of claim transactions. Each operator's claims form one chain per funding note; the chain length per epoch is the operator's block count for the previous epoch, exactly the statistic `cryptarchia-v1-protocol.md` §Privacy says must not be inferable from on-chain activity. With a fresh reward key per claim (the #717 short-term fix) the chain persists; with the reward note itself paying the fee (S-001) it disappears, since each claim consumes only what it created.

**Recommendation**

- *Short term*: when a fee note is unavoidable, fund each claim from a distinct note under a distinct key and never return change to a key that funds claims; expose `funding_pks`, `change_pk` and `reward_pk` in the claim request (S-002) so an operator can do so.
- *Long term*: S-001 (self-funded claim, no input note), together with a client-side claim scheduler that spreads claims uniformly over the epoch rather than bursting at the boundary.

**References**: `cryptarchia-v1-protocol.md` §Privacy; `cryptarchia-proof-of-leadership.md` §Linking the Proof of Leadership to a Block ("reusing the same one could allow multiple PoLs to be linked to the same identity"); #717.

### LB-003 · The voucher index is persisted asynchronously after the header is signed, so a crash in between reuses the index and forfeits one block's reward

| | |
|---|---|
| Severity | Low |
| Difficulty | High |
| Category | Economic / Incentive |
| Target | `services/wallet/src/states.rs:L287-L295` (`add_next_known_voucher`), `:L409-L421` (`update_state`), `services/chain/chain-leader/src/leadership.rs:L122-L137` (voucher generated before the proof), Overwatch `services/state/updater.rs:L34-L38`, `services/utils/src/overwatch/recovery/operators.rs:L61-L66` |
| Status | Open |

**Description**

A voucher secret is `Poseidon2(master_key, index)` (`kms/operators/src/zk/voucher.rs:37`) with `index = next_new_voucher_index` (`states.rs:292-294`), so the index is the only thing that makes vouchers distinct. `add_next_known_voucher` increments the counter and calls `update_state`, which hands a `RecoveryState` to the Overwatch `StateUpdater`; `update` only sends it on a watch channel (`updater.rs:34-38`) and the `RecoveryOperator` saves it to storage later on its own task (`operators.rs:61-66`). Nothing awaits the save. The leader meanwhile builds the proof with that commitment and publishes the block (`leadership.rs:122-137` then `apply_and_publish_block_proposal`). If the process dies after the block is out and before the operator's save lands, the restored counter is one behind: the next win derives the same secret, the same `voucher_cm` goes into a second header, and the two leaves share one nullifier. Only one `LEADER_CLAIM` can ever succeed for the pair (`core/src/mantle/ops/leader_claim.rs:255-257`), so one reward is lost. The same rewind also loses the path of the first block's commitment if the wallet restarts from a persisted `WalletState` that predates it; #85 asked exactly for this confirmation ("that `next_new_voucher_index` is persisted and never rewinds") and the answer is that it is persisted, but not before the commitment is public.

**Exploit scenario**

Not attacker-triggerable. An operator's node is killed (OOM, power loss, a deploy) within the save latency after a proposal; the window is the storage write, normally milliseconds, so this is a rare operational loss of one block reward per occurrence. It matters more on a node that wins often and restarts often, for example under the testing harness.

**Recommendation**

- *Short term*: persist the incremented index before returning the commitment to the leader (await the recovery save, or write the counter through the storage adapter directly in `generate_new_voucher_secret`), and on startup skip forward past any index whose commitment already appears in the chain the wallet has applied.
- *Long term*: make the voucher secret depend on something the block already fixes (for example hash the slot and the parent id into the derivation) so that an index reuse cannot yield an equal commitment, and reject a duplicate `voucher_cm` in header validation as defence in depth.

**References**: `bedrock-anonymous-leaders-reward.md` §Voucher creation and inclusion (one-time random secret), §Preventing Double Claims Without Breaking Privacy; #85 report (deterministic derivation observation).

### LB-004 · A wallet debug log pairs a voucher commitment with its nullifier at claim time, which is the link the voucher scheme exists to hide

| | |
|---|---|
| Severity | Low |
| Difficulty | Low |
| Category | Privacy / Anonymity · Auditing and Logging |
| Target | `services/wallet/src/lib.rs:L1310-L1318` (`reserve_claimable_voucher`), `services/wallet/src/states.rs:L44-L50` (`PendingClaims::reserve`), `:L58-L66` (`release`) |
| Status | Open |

**Description**

```rust
// services/wallet/src/lib.rs:1310-1318
debug!(target: LOG_TARGET, nf = ?voucher.nullifier, cm = ?voucher.commitment,
       "Found and reserved claimable voucher");
```

The commitment is public in a block header and the nullifier becomes public in the claim; the protocol's anonymity property is precisely that the two cannot be linked (`bedrock-anonymous-leaders-reward.md` §Unlinking Block Rewards from Proposals). This line writes the pair to the log at the moment the claim is built. `PendingClaims::reserve`/`release` log the nullifier alone with a timestamp, which links the nullifier to the node by time rather than by commitment. The voucher index is not printed anywhere (checked for item 5), and the block-proposal path logs only counts per slot (`leadership.rs:162-169`) and a note id on failure (`:126-131`), never the commitment, so this is the one place where the header side and the claim side meet in a log line.

**Exploit scenario**

Logs at `debug` are shipped by the tracing exporters with every field (#460, 37-LB-004) or kept on disk with the state files (#648). Whoever reads them maps each of the operator's nullifiers to its block header, and through the header to the slot, with no chain analysis at all.

**Recommendation**

- *Short term*: log the commitment only, or a truncated hash of the nullifier, and never both in one event; do the same in `PendingClaims` and in the claim-build debug line at `:1429-1437` if a nullifier is ever added there.
- *Long term*: a `tracing` field-redaction layer for the identity types (`VoucherCm`, `VoucherNullifier`, `NoteId`, `ZkPublicKey`) so the pairing cannot reappear through a new log statement.

**References**: `bedrock-anonymous-leaders-reward.md` §Unlinking Block Rewards from Proposals, §Preventing Double Claims Without Breaking Privacy; #460; #648; #328 (the same pair is returned by `GET /leader/claim/vouchers`).

## 5. Suggestions (non-security)

### S-001 · Specification: resolve the `LEADER_CLAIM` balance-versus-output-note disagreement in favour of the output note, add the self-funded example, and make the share predictable within an epoch

The two texts (`bedrock-anonymous-leaders-reward.md` §Claiming the reward, §Validation: "increase the balance of the Mantle Transaction by the share amount"; `bedrock-v1.1-mantle-specification.md` §LEADER_CLAIM §Execution: "construct a single output note with value leader_reward under the public key defined in the payload") still disagree at the commit read, and the node implements the second (`ledger/src/lib.rs:856-873`; `balance` is only ever changed by `Transfer`, `:877-885`). Working through the issue's three questions:

- *Gas.* Both texts keep `EXECUTION_LEADER_CLAIM_GAS = 580` (`core/src/mantle/ops/leader_claim.rs:213-215`). Under either wording a claim that keeps its money needs a `TRANSFER` (590): under the output note, to pay the fee (the reward is not balance); under the balance wording, to turn the surplus into a note, since any unspent balance is a tip that returns to the pool (`§Mantle Transaction Fee`). So both cost 580 + 590 plus storage. The balance wording would save the ledger the note insertion, so its 580 is arguably too high, and `analysis-gas-cost-determination.md` should say which model it priced.
- *Binding.* Both bind the Proof of Claim to `mantle_txhash` (`§Proof of Claim`, `leader_claim.rs:225-244`). Under the output note the destination is the `public_key` of the payload, hashed into the transaction; under the balance wording the destination is the `TRANSFER` outputs, also hashed. Neither is weaker.
- *Self-funding.* The balance wording cannot pay out without an input note, because `TRANSFER` requires a non-empty input list (`§Input Notes Spendability Validation`, `assert len(inputs) > 0`; `core/src/mantle/ledger.rs:293`), so a fee note is needed anyway. The output-note version *can* self-fund: `[LEADER_CLAIM, TRANSFER(inputs=[reward_note_id], outputs=[...])]` is valid because operation `i` is validated against the state operations `0..i−1` left (`§Validation`; `ledger/src/lib.rs:944-955` rebuilds the helper each iteration), exactly the pattern the specification's `CLAIM_POW_REWARD` example uses. The obstacle is that `reward_note_id` commits to the value (`derive_note_id` hashes `note.value`) and the value is `share = ⌊pool/unclaimed⌋` at execution, which moves by one token as other leaders claim in the same epoch and arbitrarily at the boundary (`ledger/src/mantle/leader.rs:173-182`). A self-funded claim built for `q` is invalid if it executes at `q+1`.

Suggested resolution, to raise upstream: keep the Mantle text as normative and rewrite `bedrock-anonymous-leaders-reward.md` to match; add the self-funded example under §LEADER_CLAIM; and freeze the share for the epoch at the boundary (`share_N = ⌊pool_N / |cm_N| − |nf_N|⌋`, remainder carried), which the reward document currently rejects on the ground that the one-token difference is "small enough not to justify freezing the share", an argument that does not consider that a predictable value is what makes a claim self-funding and therefore funding-key-free (LB-001, LB-002, #717).

### S-002 · Minimal API change so a claim can use a fresh reward key and a fee note not under the funding key

`LeaderClaimTxRequest` (`services/wallet/src/lib.rs:233-239`) would carry `reward_pk: ZkPublicKey`, `change_pk: ZkPublicKey` and `funding_pks: Vec<ZkPublicKey>`, defaulting to today's `funding_pk`; `WalletApi::build_leader_claim_tx` (`api.rs:163-185`) and `WalletMsg::BuildLeaderClaimTx` (`:699-720`) pass them through; `POST /leader/claim` accepts an optional JSON body with the same three optional fields (`handlers.rs:1354-1370`, body types in `nodes/api-common/src/bodies`), and `build_and_submit_claim_tx` (`chain-leader/src/lib.rs:789-811`) fills the defaults from `LeaderWalletConfig`. A `self_funded: bool` (S-001 pattern) would set `funding_pks = []` and extend the builder's ledger inputs with `LeaderClaimOp::utxo(reward_amount)` (`core/src/mantle/ops/leader_claim.rs:69-79` already builds that UTXO). The node has no per-claim key derivation (the keystore is a fixed set of titles, `keystore.rs:32-39`; `wallet-technical-standard.md` describes a hierarchy the node does not implement), so "fresh key per claim" also needs a KMS operator that derives child keys, which is the larger piece of work.

### S-003 · Vouchers of orphaned or unpublished blocks are never pruned from the wallet's recovery state

`known_vouchers` grows by one entry per generated voucher (`states.rs:291-295`) and shrinks only for nullifiers seen in an immutable `LEADER_CLAIM` (`services/wallet/src/lib.rs:1608-1624`, `states.rs:331`). A voucher whose block was orphaned, or whose proof failed after the voucher was drawn (`leadership.rs:122-160` draws the voucher before proving and continues to the next winning note on failure), stays in the persisted `RecoveryState` forever, is re-serialised on every `update_state`, and is iterated on every `claimable_vouchers` call. Suggest pruning vouchers whose commitment is absent from the chain once the LIB has passed the epoch in which they were drawn.

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
