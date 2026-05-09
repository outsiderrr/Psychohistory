# Architecture Decision Records — Psychohistory V0.3

**Source of truth** for ratified architectural decisions. L1 conversation is the only entity authorized to add or modify ADRs. L3 task conversations may **propose** an ADR but cannot commit one without L1 sign-off.

Each ADR is immutable once Status is `Accepted`. Subsequent revisions become new ADRs that supersede the prior; the prior is marked `Superseded by ADR-XXXX` but its body is preserved for archaeological context.

---

## Format

```
## ADR-XXXX — <Title>

**Status**: <Proposed | Accepted | Superseded by ADR-YYYY>
**Date**: YYYY-MM-DD
**Decided by**: L1.C / L2.C / spike outcome / etc.
**Related**: TDD §X.Y, prior ADR-YYYY

### Context
<situation forcing the decision>

### Options considered
- A: <name> — <consequences>
- B: <name> — <consequences>
- ...

### Decision
<what was chosen>

### Rationale
<why this option>

### Consequences
<positive, negative, follow-up tasks>
```

---

## ADR-0001 — Privacy route: V0.3 transparent + V0.4 architectural hooks

**Status**: Accepted
**Date**: 2026-05-08
**Decided by**: L1.C
**Related**: TDD V0.31 §1, §8.2, §12.4; L1-PLAN §4 decision 1

### Context

V0.30 §12.4 stated "predictions are private (not readable in any UI designed for this protocol)". The claim is technically accurate at the UI layer but on-chain prediction storage is publicly readable by anyone running an indexer. This invalidates the sponsor-bidding tiered-access mechanism's core economic assumption (sponsors pay for early exclusive information). Without addressing it, sponsor auction equilibrium price collapses to zero.

Three V0.3-feasible privacy routes were debated in L1.B rounds 2–3, plus a fourth "accept transparency and reposition" route:

- **A2 (oracle custody)**: ciphertext encrypted to oracle pubkey, oracle decrypts off-chain at close. Trust: oracle single point of failure. ~1.5–3 person-months, 3–6 weeks delay.
- **A3 (self-reveal commit-reveal)**: commitments on-chain, predictors self-reveal post-close. Issues: reveal-time gaming, no-reveal liveness problem, two-tx UX. ~2–4 person-months, 4–8 weeks delay.
- **A\* (threshold encryption)**: t-of-n operators jointly decrypt. Cleanest trust model, but requires DKG ceremony, operator coordination, key rotation. ~5–8+ person-months, 8–12+ weeks delay.
- **B (transparent + analytics SLA repositioning)**: accept on-chain transparency; sponsor value redefined as analytics convenience, schema, SLA, service priority. ~0.5–1 person-month, 0–2 weeks delay.

### Options considered

- A2 / A3 / A\* — privacy-preserving, varying trust and engineering cost
- B — accept transparency, reposition sponsor value
- Defer all privacy work to V0.4 — same as B but explicit about future work

### Decision

**V0.3 ships route B** (transparent + analytics SLA reposition). Privacy routes are deferred to V0.4 with **A\* (threshold encryption)** as the preferred V0.4 target; A2 may serve as a faster trusted-tier interim if needed.

V0.3 ships **5 mandatory architectural hooks** so that V0.4 upgrade is a focused change set, not a protocol rewrite:

| Hook | Where | V0.3 enforcement | V0.4 use |
|---|---|---|---|
| A | `IPredictionEngine.submitPrediction` `bytes encryptedPayload` parameter | `require(encryptedPayload.length == 0)` | Carries ciphertext per `Bounty.privacyMode` |
| B | `Bounty.privacyMode` enum (`Transparent` / `OracleEncrypted` / `ThresholdEncrypted`) | `require(privacyMode == Transparent)` at `createBounty` | Allows other modes for high-value bounties |
| C | `uint256[50] private __gap` on every upgradeable contract | Reserved | Storage extension for encryption-related fields |
| D | `_getPrediction(bountyId, predictor)` internal helper indirection | Returns plaintext directly | Routes through decryption based on privacyMode |
| E | No batch-public-prediction view functions | `getPrediction` is single-record only | Aggregate views remain off-chain in both versions |

### Rationale

- A\* has the cleanest decentralized-trust property but exceeds V0.3 scope by an order of magnitude in time and complexity.
- A2 introduces a centralized trust point that contradicts the "decentralized" framing.
- A3 forces a UX (commit + reveal) that conflicts with the "lower the barrier to predict" philosophy and creates gaming incentives.
- B is the only route that ships within V0.3 launch budget while preserving optionality. The cost is rewriting sponsor narrative — sponsors now buy "structured analytics + standardized API + timeliness/format SLA + service priority" not "exclusive cryptographic early access". Auction equilibrium price drops from "information edge premium" (~$5K–$50K) to "convenience premium" (~$500–$5K), but the protocol economy remains viable.

### Consequences

**Positive**:
- V0.3 launches in weeks, not quarters
- §12.4 wording aligns with on-chain reality; no false security claims
- 5 hooks make V0.4 a contained upgrade

**Negative**:
- Sponsor pool size will be smaller in V0.3 than the original V0.30 vision
- §1 / §8 narrative had to be rewritten away from "information arbitrage moat"
- Oracle preemption risk explicitly acknowledged as a V0.3 trust assumption (similar to Polymarket-at-launch)

**Follow-up**:
- V0.4 tracking issue when V0.3 launches
- Sponsor outreach copy must reflect new value proposition

---

## ADR-0002 — Buyback formula: rolling geometric smoothing

**Status**: Accepted
**Date**: 2026-05-08
**Decided by**: L1.C
**Related**: TDD V0.31 §6.5, §10.5; L1-PLAN §4 decision 2

### Context

V0.30 §6.5 contained internally inconsistent language: text claimed "each dollar enters the buyback queue and is spent linearly over 12 weeks", but the formula said "spend `1/12` of Treasury's current buyback balance per epoch". The two are different: the latter is geometric decay (half-life ~8 epochs, ~65% spent in 12 epochs), not strict 12-epoch linear consumption.

### Options considered

- **A — Change wording**: keep the `currentBalance × 1/12` formula. Update §6.5 wording to "rolling geometric smoothing mechanism, spend rate proportional to accumulated reserves".
- **B — Change formula**: implement tranche queue. Per-inflow tranche state, active tranche cleanup, gas grows with inflow count. Achieves strict 12-epoch linear consumption.

### Decision

**Option A**. §6.5 wording updated to reflect the geometric model. Implementation remains `pendingBuybackBalance × 1/12` per epoch.

### Rationale

- The geometric model achieves the product goal (smooth buy pressure, no single-epoch market shock) with one state variable.
- Tranche queue requires per-inflow storage and cleanup, gas overhead grows with bounty count, and adds significant test surface.
- "12 weeks" was always a round-number heuristic, never a hard requirement — the actual goal is "long enough to avoid market impact, short enough to recycle revenue meaningfully". Geometric decay's effective half-life of ~8 weeks satisfies this.

### Consequences

**Positive**:
- Implementation reduced to a single state variable update
- No tranche cleanup logic, no gas growth with inflow count
- L2-T5b (BuybackExecutor) implementation simplified

**Negative**:
- Asymptotic decay means a small residual balance persists indefinitely (mathematically; in practice rounding rounds to zero)
- "Each dollar takes 12 weeks to spend" is no longer true; communication to community must be careful

**Follow-up**:
- §6.5 wording rewritten in TDD V0.31
- L2-T5b acceptance includes a test: starting with 1200 USDC and no further inflows, after 12 epochs balance is ~421 USDC (~65% spent), not 0

---

## ADR-0003 — Slice A (consolation) destination when bottom group is empty

**Status**: Accepted
**Date**: 2026-05-08
**Decided by**: L1.C
**Related**: TDD V0.31 §5.5, §10.4; L1-PLAN §4 decision 3

### Context

§5.5 Pool 2 Slice A (30% of sponsor pool) is allocated to bottom-50% predictors as consolation. In all-tie or single-predictor bounties, the bottom group is empty, so Slice A has no recipient. The original §5.5 said "goes to Treasury" without specifying which Treasury category.

### Options considered

- **Buyback (CAT_BUYBACK)** — automatically purchase and burn PSYH
- **Fee (CAT_FEE)** — accumulate as protocol revenue
- **Dust (CAT_DUST)** — semantically incorrect; dust is rounding residue
- **DAO (CAT_DAO)** — accumulate in DAO sub-account, alongside the regular Slice D2
- **New: P2_UNALLOCATED → DAO sub-account** — distinguish accounting category but route to DAO

### Decision

Slice A in empty-bottom-group case is transferred to **Treasury under the new category `CAT_P2_UNALLOCATED`**, accumulated in the DAO-controlled sub-account. Pre-DAO: admin multisig custody, identical to Slice D2.

### Rationale

- Auto-buyback would distort token monetary policy at the worst moments (frequent all-tie scenarios early in protocol life would silently accelerate burns).
- Permanent lock-up wastes funds.
- Routing to DAO is monetary-policy-neutral and preserves future governance flexibility (airdrop, infrastructure grant, eventual buyback, etc.).
- The new category `P2_UNALLOCATED` separates these funds from regular Slice D2 (10% direct DAO allocation) for accounting clarity.

### Consequences

**Positive**:
- Empty-bottom-group bounties resolve cleanly without policy distortion
- Future DAO has an additional discretionary pool

**Negative**:
- New `bytes32` constant added to Treasury; minor storage / API surface increase
- Pre-DAO multisig has discretion over an additional fund stream

**Follow-up**:
- §10.4 ITreasury adds `CAT_P2_UNALLOCATED` category constant
- §5.5 spells out the empty-bottom-group routing
- L2-T3.1 Treasury implementation must support the new category in `receivePoolFunds`

---

## ADR-0004 — Sponsor ranking gas: hard cap of 100 sponsors per bounty

**Status**: Accepted
**Date**: 2026-05-08
**Decided by**: L1.C (user product call)
**Related**: TDD V0.31 §8.1, §10.1; L1-PLAN §4 decision 4

### Context

§8.1 sponsors deposit additively (cumulative `contributionAmount`). At `closeTimestamp`, `BountyManager.finalizeSponsorRanking` sorts sponsors descending by contribution to assign rank, tier, and access timestamp. With unbounded sponsor count, a 1000-sponsor bounty would exceed block gas limits during sort.

### Options considered

- **A — Hard cap**: `MAX_SPONSORS_PER_BOUNTY = 100`. New sponsor #101 reverts at `addSponsorship`. Existing sponsors may continue topping up after cap.
- **B — Off-chain hint + on-chain pagination verification**: identical pattern to settlement cutoff hint. Permissionless submitter provides sorted order, contract verifies pairwise descending in pages. No upper bound.

### Decision

**Option A**. `MAX_SPONSORS_PER_BOUNTY = 100`.

### Rationale

- V0.3 launch is unlikely to attract single bounties exceeding 100 distinct sponsors. The cap is generous for early-product reality.
- Option B doubles the off-chain-hint infrastructure (cutoff hint already exists for settlement; sponsor ranking would add a parallel system with similar but distinct semantics).
- Cap is a one-line constant; in-memory sort of ≤ 100 elements fits comfortably in block gas budget.
- If V0.3 traffic demonstrates demand for >100 sponsor bounties, V0.4 can upgrade to hint-pagination model. The current V0.3 storage layout does not preclude this upgrade.

### Consequences

**Positive**:
- Implementation trivial; gas safe by design
- No off-chain hint infrastructure needed for sponsor ranking

**Negative**:
- A hypothetical viral bounty hitting 100 sponsors blocks subsequent enrollment
- Some marketing discomfort if cap is publicly visible

**Follow-up**:
- §9 Constants adds `MAX_SPONSORS_PER_BOUNTY = 100`
- §10.1 `addSponsorship` reverts with `SponsorCapReached` for new addresses past 100
- §14 test added: `addSponsorship` from 101st distinct address reverts; existing sponsors may still top up

---

## ADR-0005 — RewardDistributor token holding model: on-demand mint via MINTER_ROLE

**Status**: Accepted
**Date**: 2026-05-08
**Decided by**: L1.C
**Related**: TDD V0.31 §6.2, §10.3, §10.6; L1-PLAN §4 decision 5

### Context

40% of total PSYH supply (400M tokens) is allocated to prediction mining rewards. Two implementation models:

- **Pre-mint (option A)**: `PsychohistoryToken` mints 400M PSYH to `RewardDistributor` at deployment. Subsequent rewards are `ERC20.transfer` calls. `totalSupply()` is constant 1B from day 1.
- **On-demand mint (option B)**: `RewardDistributor` holds `MINTER_ROLE` on `PsychohistoryToken`. Rewards trigger `mint()` calls. `totalSupply()` grows with cumulative rewards. The 400M cap is enforced internally by `RewardDistributor`'s own counter.

### Options considered

- A: pre-mint
- B: on-demand mint

### Decision

**Option B** (on-demand mint via `MINTER_ROLE`).

### Rationale

- `totalSupply()` accurately reflects circulating supply at all times (option A inflates `totalSupply()` to 1B even when only 50M has been distributed).
- `RewardDistributor` proxy compromise exposes at most the next batch of mints, not 400M reserve (option A would mean a single proxy compromise drains 40% of supply).
- §4.2 already specifies `MINTER_ROLE → RewardDistributor`, indicating this was the original design intent; option A would have left MINTER_ROLE unused.
- Cap is enforced by both `mintingCapRemaining` (RewardDistributor internal counter) and the §6.4 monthly cap reservation logic. Two layers of inflation control.

### Consequences

**Positive**:
- `totalSupply()` matches circulating reality
- Smaller attack surface on RewardDistributor proxy
- Token contract stays simpler (no special handling for the 400M reserve)

**Negative**:
- Slight per-claim gas overhead (mint vs transfer); negligible in practice
- The cap enforcement is now distributed across RewardDistributor logic and is auditable but not naively visible from `totalSupply()`

**Follow-up**:
- §10.6 `IPsychohistoryToken.mint` enforces only the 1B total supply cap
- §10.3 `IRewardDistributor` rewrites to reserve / assign / finalize / claim model with cap consumed at reservation
- L2-T2.2 implementation includes `MINTER_ROLE` admission test
- L2-T3.2 implementation includes `mintingCapRemaining` invariant test

---

## ADR-0006 — Invalidation refunds `effectiveWager` only (1% fee non-refundable)

**Status**: Accepted
**Date**: 2026-05-09
**Decided by**: L2-T0.C (proposed); L1 ratification in this commit
**Related**: TDD V0.31 §5.7, §5.8, §10.2 (resolveAsInvalid), §14.2 (invalidation test); ADR-0001 (effective wager term originated in L1.C decision 1's adjacent fee model work, ratified in TDD §5.7)

### Context

ADR-0001's privacy decision and the L1.B fee model work landed `effectiveWager = rawWager × 0.99` as the per-predictor pre-deduction model (§5.7). The 1% fee is transferred to `Treasury.CAT_FEE` at submission time, not at settlement. This is settled.

What was NOT settled by L1.C was the invalidation refund semantics. V0.30 §5.8 said "All predictors get 100% wager refund. No ranking, no scoring, no $PSYH minting. Sponsors get 100% deposit refund. No fees collected." Two parts of that text became inconsistent with the §5.7 effective-wager model:

- "100% wager refund" is now ambiguous — refund 100% of `rawWager` (including the fee already in Treasury), or refund 100% of `effectiveWager` (only what's still in PE escrow)?
- "No fees collected" is impossible: fees were already collected at submission. They cannot be un-collected without a Treasury fee clawback path.

L2-T0.B GPT-5.5 review flagged this as a decision-resolution gap. L2-T0.C proposed two options:

- **A — Refund `rawWager` (100% of original deposit)**: requires PE to call a new Treasury function `clawbackFee(bountyId)` that reverses CAT_FEE balance for invalidated bounties. New role-gated outbound path on Treasury, new accounting subtlety, additional attack surface (e.g., a malicious oracle invalidating to drain CAT_FEE later).
- **B — Refund `effectiveWager` (the post-fee principal in PE escrow)**: zero new Treasury surface; `Treasury.CAT_FEE` is a one-way protocol revenue stream with simple semantics. User perception cost: "99% refund on invalidation" instead of "100% refund".

### Decision

**Option B — invalidation refunds `effectiveWager` only**.

- Predictors receive back exactly `effectiveWager_i = rawWager_i × 0.99`, paid out of PE-held escrow.
- The 1% fee already committed to `Treasury.CAT_FEE` at submission is **non-refundable** under any path.
- Sponsors continue to receive 100% deposit refund (their funds were never charged a fee; Pool 2 platform take only triggers under successful settlement, not invalidation).

### Rationale

- The 1% fee is conceptually "protocol usage charge" — paid at the moment the prediction enters the system, not contingent on outcome. This is consistent with how protocol fees behave in Polymarket, Augur, and most prediction markets at launch.
- A fee-clawback path on Treasury introduces a new role-gated mutator (`PREDICTION_ENGINE_ROLE` or similar) that can move funds out of CAT_FEE. This expands the attack surface of the most concentrated fund pool in the protocol.
- The user-perception cost of "99% refund" is small — and easily disclosed in front-end submission flow. Most users will never encounter invalidation; for the subset who do, transparent disclosure is preferable to design complexity.
- This decision is **forward-compatible** with V0.4 privacy paths: if encrypted submissions still pay a 1% fee at submission, the same non-refundable-on-invalidation rule applies uniformly.

### Consequences

**Positive**:
- Treasury keeps a one-way fee stream — simple accounting, no clawback complexity.
- Zero new role-gated outbound path on Treasury.
- Symmetry with normal-settlement protocol-take semantics.
- Implementation simpler: PE-held escrow holds exactly `Σ effectiveWager`; refund just transfers that.

**Negative**:
- Front-end must disclose the 1% non-refundable fee at submission (no surprise).
- Invalidated bounties leave `Σ feeAmount` in CAT_FEE, attributable on-chain to a now-cancelled bounty. Auditors must understand this is by design, not residue.

**Follow-up**:
- §5.8 spec rewritten with invariant: `Σ effectiveWager_refunded == Σ effectiveWager_at_submission`; `CAT_FEE` unaffected by invalidation
- §10.2 `resolveAsInvalid` natspec clarifies fee non-refundability
- §14.2 invalidation test asserts `effectiveWager` refund and `CAT_FEE` invariance
- Front-end work item (out of TDD scope): submission flow displays "1% protocol fee, non-refundable" disclosure

---

## ADR-0007 — Cutoff hint replaceable + lazy top-group computation

**Status**: Accepted
**Date**: 2026-05-09
**Decided by**: L2-T0.C (proposed); L1 ratification in this commit
**Related**: TDD V0.31 §9 (Prediction struct, `inTopGroup` removed), §10.2 (submitCutoffHint), §11 (Pass 2 rewrite, Pass 3/4 lazy lookup), §14.3 (cutoff hint replacement test); supersedes one paragraph of §11 in V0.30 (first-write-wins hint behavior)

### Context

L1.C decision 4 (sponsor cap = 100, ADR-0004) and the broader L1.B settlement-track work fixed cutoff hint trust as **permissionless, no bond, on-chain paginated verification**. What was NOT specified was failure recovery: V0.30 §11 said `submitCutoffHint` was first-write-wins, and a failed verification meant "the bounty enters a stuck state requiring oracle invalidation".

L2-T0.B GPT-5.5 review identified this as inadequate:

- A single bad hint submission DoSes the bounty until ORACLE_ROLE manually invalidates.
- Oracle invalidation is heavy-handed (refunds all sides) and abuses the privileged path for what is fundamentally a settlement mechanic.
- A wrong hint is detectable in O(seconds) (the verification arithmetic is simple); recovery should also be O(seconds), not require oracle escalation.

L2-T0.C proposed two combinations:

- **Default (V0.30) — first-write-wins, no replacement**: keeps Pass 2 simple but creates the stuck-bounty failure mode.
- **D2-b + Opt-α (proposed)** — hint is replaceable; top-group membership is computed lazily, not stored:
  - **D2-b**: `submitCutoffHint(bountyId, score)` overwrites a pending failed hint AND resets Pass 2 counters (`cutoffStrictlyAboveCount`, `cutoffAtCutoffCount`, `settledUpTo` → 0). Idempotent if same score is re-submitted. Reverts if Pass 2 has already verified successfully.
  - **Opt-α**: remove `Prediction.inTopGroup` storage field. Pass 3 and Pass 4 compute membership lazily as `score >= SettlementState.topGroupCutoffScore`. Cost: ~few hundred extra gas per prediction in Pass 3/4 (one comparison + one storage read of `topGroupCutoffScore` that's hot after first access).

D2-b alone (without Opt-α) is impractical — replacing a hint while `inTopGroup` flags are persisted requires a per-prediction rollback pass, which costs more gas than the savings.

### Decision

**Adopt D2-b + Opt-α as a combined unit.** Cutoff hint is replaceable on verification failure; top-group membership is computed lazily.

### Rationale

- **Failure recovery in O(minutes), not O(oracle escalation)**: any party submits a corrected hint; counters reset; Pass 2 re-runs. No human-in-the-loop, no privileged path. The DoS asymmetry — attacker pays full gas for failed hint + partial Step 2B verification before revert; defender's correct hint resolves the bounty permanently — is acceptable for V0.3 launch.
- **Pass 2 idempotence**: removing `inTopGroup` storage writes makes the verification pass a pure read-and-count operation. A failed hint costs zero per-prediction storage rollback.
- **Lazy top-group computation is cheap**: one storage read of `topGroupCutoffScore` per `settle()` (cached after first access by Solidity's transient warm-slot pricing) plus one per-prediction comparison. Across the four-pass settlement, total amortized gas is comparable to the saved Pass 2 writes.
- **Forward-compatibility with bond model**: V0.4 may add a refundable bond on hint submission to deter griefing. The replaceable-hint design supports this naturally — a bond reverts on revert, refunds on success. Adding bond logic later does not require restructuring.

### Consequences

**Positive**:
- No bounty-stuck-on-bad-hint failure mode.
- Pass 2 implementation simpler (no per-prediction writes).
- ORACLE_ROLE invalidation is reserved for genuinely unknowable propositions, not settlement mishaps.
- Forward-compatible with V0.4 bond addition.

**Negative**:
- ~few hundred gas/prediction cost in Pass 3 and Pass 4 (lazy comparison). For a 1000-predictor bounty across two passes, ~200K extra gas total. Negligible relative to the rest of settlement.
- Pass 2 verification logic is a small bit more complex (counter reset on overwrite, idempotent same-score check).
- Griefing window: an attacker can spam wrong hints. Each costs the attacker full gas + partial Step 2B work. In normal use this is self-limiting; if observed in production, V0.4 bond addresses it.

**Follow-up**:
- §9 `Prediction.inTopGroup` removed; spec note added
- §9 K_WAD_PHASE5/6 cleaned to compilable Solidity literals (housekeeping fix landed in same commit as Opt-α)
- §10.2 `submitCutoffHint` natspec updated for overwrite + counter-reset semantics
- §10.2 `closeBounty` passthrough added (not part of D2-b but discovered in same review pass; permissionless force Open → Closed)
- §11 Pass 2 rewritten with overwrite path; Pass 3/Pass 4 use lazy top-group lookup; State Initialization and Events Emitted subsections added
- §14.3 cutoff hint replacement test + idempotent re-submission test + closeBounty passthrough test added
- L2-T4.3b implementation must include hint-replacement state-machine tests as part of acceptance

---

## ADR-0008 — K(t) mining alignment with Brier philosophy (Pool 3 amount slice score-weighted)

**Status**: Accepted
**Date**: 2026-05-09
**Decided by**: L1 CEO review (round 2, "scoreboard thesis" deepening); L1 ratification in this commit
**Related**: TDD V0.31 §5.6 (Pool 3 distribution), §11 Pass 4 / Final Finalization; supersedes the V0.31 amount-slice formula by adding score weighting; **subordinate to ADR-0010** (long-term forecaster rating, which makes the scoreboard thesis a project-level principle)

### Context

V0.31 Pool 3 splits 50/50 between **amount slice** (weight = `pool1Payout_i = effectiveWager_i + remainderShare_i`) and **quality slice** (weight = `score_i × effectiveWager_i`).

The CEO review identified an internal contradiction: the protocol's design philosophy is **"reward calibration, not stake size"** (Brier scoring's whole point), and the project's positioning is **"Forecaster Scoreboard"** (ADR-0010). Yet the amount slice rewards predictors purely by Pool 1 winnings — which is dominated by stake size, not calibration quality.

Concrete consequence: a predictor who pushes a large `effectiveWager` with a mediocre Brier score gets disproportionately more PSYH from the amount slice than a predictor with a smaller wager but better calibration. **This is the opposite of what the Brier philosophy and Scoreboard thesis demand.**

A future PSYH holder grievance ("the protocol claims calibration-first but rewards stake-size") becomes a real PR / governance liability.

### Decision

**Change the Pool 3 amount slice formula to add `score` weighting**, identical in spirit to the quality slice but using `pool1Payout` instead of pure `effectiveWager`:

```
old (V0.31):
    amountTokens_i = (alloc/2) × pool1Payout_i / Σ(pool1Payout_j for j ∈ T)

new (V0.32):
    amountTokens_i = (alloc/2) × (score_i × pool1Payout_i) / Σ(score_j × pool1Payout_j for j ∈ T)
```

The two slices retain a meaningful difference:
- **Amount slice**: `score × pool1Payout` — rewards holistic predictor success (calibration × actual won-from-pool-1 amount, including the remainder share)
- **Quality slice**: `score × effectiveWager` — rewards calibration on submitted stake (excludes remainder share)

Both now align with Brier philosophy.

### Rationale

- **Internal coherence**: every distribution math now multiplies by `score`, making "calibration" the primary signal across all rewards
- **Scoreboard thesis alignment**: PSYH miners are not "people who pushed big stakes" but "calibrated forecasters" (with stake as a secondary multiplier)
- **PR / governance protection**: no future "the math contradicts the marketing" attack surface
- **Implementation cost**: single formula change, no new storage, no new state machine

### Consequences

**Positive**:
- Brier philosophy fully load-bearing across all reward channels
- Mining incentive aligned with ADR-0010 forecaster rating (future)
- Score-0 fallback rule from §5.6 already covers the degenerate `Σ(score × pool1Payout) == 0` case (fall back to weighting by `pool1Payout` alone)

**Negative**:
- Mining incentive at low-score participants drops further (they already get little; now get even less from amount slice). **Acceptable** under Scoreboard framing — calibration is the protocol's value statement.
- Implementation slightly less symmetric than "amount = winnings" (a clean intuition); now amount = winnings × calibration

**Follow-up**:
- §5.6 Pool 3 amount slice formula updated in V0.32
- §11 Pass 4 / Final Finalization computation updated in V0.32
- L2-T4.3e (Final Finalization) implementation must use new formula

---

## ADR-0009 — int256 for negative-value numerical predictions

**Status**: Accepted
**Date**: 2026-05-09
**Decided by**: L1 CEO review (Hook #1 strategic discussion surfaced concrete user examples); L1 ratification in this commit
**Related**: TDD V0.31 §9 (`Prediction.predictedValue`), §11 Pass 1 numerical scoring

### Context

CEO review Hook #1 surfaced concrete user examples of qualified propositions, including:

- "美国 2026 中期选举民主党净席位变化 (vs 共和党)" — a **signed integer** (positive = Dem net gain, negative = Rep net gain)
- "中国 2026 全年汽车出口增量 vs 2025" — could be signed if exports decline

V0.31 declares `Prediction.predictedValue` as `uint256` and `Bounty.resolvedValue` as `uint256`, which **cannot represent negative values**. Workarounds (offset trick: store `value + LARGE_OFFSET`) are unergonomic and error-prone for predictors and oracles.

### Decision

Change `Prediction.predictedValue` and `Bounty.resolvedValue` from `uint256` to `int256` for `PropositionType.Numerical`. `Bounty.resolvedValue` for `PropositionType.Discrete` (winning option index) remains effectively unsigned (use `int256` and assert non-negative on resolve).

`BrierMath` numerical scoring uses `abs(predictedValue - resolvedValue)` for `rawError`, which works on `int256` directly via OpenZeppelin `SignedMath.abs` returning `uint256`.

### Rationale

- **Real user need**: signed numerical events are common (margins, deltas, year-over-year changes)
- **Storage cost identical**: `int256` and `uint256` both 32 bytes
- **Library support**: OpenZeppelin `SignedMath` provides safe abs/min/max
- **No spec ABI risk**: V0.3 has not shipped any contract; ABI change is free

### Consequences

**Positive**:
- Signed numerical predictions natively supported
- No offset hacks
- `predictedDecimals` semantics unchanged (decimals describes scaling, sign is independent)

**Negative**:
- BrierMath input parameter type change (uint256 → int256 in numerical path)
- Slight code complexity in `__getRawError` helper (handle signed subtraction)
- Discrete proposition `resolvedValue` (option index) requires non-negative assert in resolve()

**Follow-up**:
- §9 Prediction struct: `int256 predictedValue` for Numerical
- §9 Bounty struct: `int256 resolvedValue`
- §11 Pass 1 numerical scoring: `rawError = SignedMath.abs(predictedValue - resolvedValue)` returns `uint256`
- BrierMath library: numerical functions accept `int256` predicted, `int256` resolved
- §10.2 `IPredictionEngine.submitPrediction` parameter type: `int256 predictedValue`
- §10.2 `IPredictionEngine.resolve` parameter type: `int256 resolvedValue`
- `IBountyManager.markResolved` parameter type: `int256 resolvedValue`

---

## ADR-0010 — Long-term forecaster rating system (Scoreboard thesis on-chain materialization)

**Status**: Accepted
**Date**: 2026-05-09
**Decided by**: L1 CEO review (Hook #1 "scoreboard psychology" thesis); L1 ratification in this commit
**Related**: ADR-0008 (Brier alignment subordinate to this); TDD V0.31 §5.2 (scoring), §9 (data structures); supersedes implicit assumption that score is per-bounty only

### Context

The L1 CEO review surfaced a project-level positioning thesis that had been implicit but never articulated:

> **Psychohistory's unique value proposition is being a *Forecaster Scoreboard* — a place where someone's calibration ability is measured, accumulated, and verifiable on-chain. Financial markets cannot deliver this (multiple trade motivations + no clean win/lose anchor); pure prediction sites like Metaculus / Good Judgment Open deliver reputation but no skin-in-the-game.**

V0.31 stores per-bounty `score` on each `Prediction`, but **does not aggregate forecaster performance across bounties**. Every Brier score, win/loss, calibration metric is stranded per-bounty. A user's "long-term track record" exists only as something an off-chain indexer could compute, not as authoritative on-chain state.

This is a **core feature gap** under the Scoreboard thesis. Without on-chain cumulative rating:
- Front-end / SDK consuming "forecaster reputation" must run their own indexer (forking the protocol's value proposition off-chain)
- Sponsor analytics SLA (Pool 2 service offering) has no canonical predictor identity to bind to
- Future feature (e.g., reputation-gated high-value bounties) has no on-chain hook
- The thesis remains a marketing claim rather than a protocol-level fact

### Decision

Add an on-chain cumulative forecaster rating maintained by `PredictionEngine` and updated during Pass 4 / Final Finalization:

```solidity
struct CumulativeBrierStats {
    uint256 totalBountiesParticipated;     // count of bounties this address submitted to and that fully settled
    uint256 totalRawWagerSubmitted;        // Σ rawWager
    uint256 totalEffectiveWagerSubmitted;  // Σ effectiveWager
    uint256 totalScoreSum;                 // Σ score (WAD-scaled)
    uint256 totalScoreWeightedByWager;     // Σ (score × effectiveWager / WAD), the calibration-weighted activity
    uint256 winsCount;                     // count of bounties where address was in top group
    uint256 lastUpdatedAt;                 // block.timestamp of last update (deterministic ordering aid)
}

mapping(address => CumulativeBrierStats) public forecasterStats;
```

Updated by `PredictionEngine` Pass 4 per-predictor as part of the existing per-prediction processing loop. Read via:

```solidity
function getForecasterStats(address predictor) external view returns (CumulativeBrierStats memory);
function getForecasterAverageScore(address predictor) external view returns (uint256);  // totalScoreSum / totalBountiesParticipated, score-0 fallback if 0 bounties
function getForecasterWinRate(address predictor) external view returns (uint256);  // winsCount × WAD / totalBountiesParticipated
```

These reads enable front-end leaderboards, reputation-aware sponsor analytics, and future protocol-level features (gated bounties, fee discounts for high-rated forecasters, etc.).

### Rationale

- **Scoreboard thesis becomes load-bearing on-chain fact**, not a marketing claim
- **Single source of truth** for forecaster reputation (no indexer divergence)
- **Foundation for future features** without retroactive migration (gated bounties, reputation-weighted sponsor analytics, etc.)
- **Storage cost minimal**: 7 uint256 + 1 timestamp per active forecaster = 8 storage slots; cold init 320 gas, warm update ~5K gas — negligible vs settlement gas
- **Update happens during existing Pass 4 loop**: no new pass, no new state machine

### Consequences

**Positive**:
- Forecaster reputation queryable on-chain
- Scoreboard thesis backed by deterministic state
- Future features (reputation-gating, leaderboards, badges) buildable without spec changes

**Negative**:
- Storage write per predictor per settlement (extra ~5K gas warm / ~20K gas cold per predictor in Pass 4)
- For 1000-predictor bounty: extra ~5M-20M gas across pagination, ~$1-4 mainnet — acceptable
- One-time consideration: on bounty invalidation / abandonment refund (ADR-0006 / future ADR), do the stats update? **Decision: stats update only on successful settlement (Bounty.state == Settled with totalPredictors > 0)**, NOT on Invalidated / Abandoned / NoSignal paths. Invalidated bounties have no Brier scoring ran, so no stats to update.

**Follow-up**:
- §9 add `CumulativeBrierStats` struct + `forecasterStats` mapping
- §10.2 `IPredictionEngine` add 3 view functions
- §11 Pass 4 / Final Finalization: per-predictor update of forecaster stats (only on successful settlement)
- L2-T4.3d / T4.3e implementation includes this update
- Front-end / SDK roadmap: leaderboard view consumes these views

---

## ADR-0011 — Launch-time TVL cap per bounty

**Status**: Accepted
**Date**: 2026-05-09
**Decided by**: L1 CEO review (Q4 audit decision: AI-only + compensating controls); L1 ratification in this commit
**Related**: ADR-0012 (emergency pause), ADR-0013 (withdrawal time-lock); TDD V0.31 §9 (Bounty struct), §10.1 (createBounty), §10.2 (submitPrediction)

### Context

Q4 in the L1 CEO review settled the audit strategy: **AI multi-round review + static analysis + Immunefi PSYH bounty + open source + testnet period**, with **no paid third-party audit**.

This is a defensible passion-infra decision but leaves residual security risk. Compensating controls are required to bound the maximum loss from any single undiscovered vulnerability.

The cleanest control: **cap Total Value Locked (TVL) per bounty**. If a single-bounty vulnerability is exploited, the maximum loss is bounded by the cap, not the full protocol-wide TVL.

### Decision

Add a `tvlCap` field on `Bounty`. Both `submitPrediction` and `addSponsorship` MUST verify:

```
bounty.totalRawWagerAmount + bounty.totalSponsorAmount + (incoming amount) ≤ bounty.tvlCap
```

If exceeded, revert with `BountyTvlCapExceeded(bountyId, attempted, cap)`.

`tvlCap` defaults to **10,000 USDC raw (= $10,000)** at protocol launch, settable per-bounty by `createBounty` caller (with `DEFAULT_ADMIN_ROLE` or curator review for higher caps in V0.4).

Cap can be raised post-deployment via a contract upgrade (Transparent Proxy pattern) once protocol shows stable operation (e.g., 6 months without security incidents).

### Rationale

- **Bounded blast radius**: an undiscovered vulnerability in settlement / claim / pool distribution affects at most one bounty's $10K
- **Slow user onboarding aligned with passion-mode launch**: $10K cap forces predictors to spread across bounties, naturally pacing growth
- **Reversible**: cap is a number, can be raised; not a structural decision
- **No contract logic change for the math**: it's just a check at deposit time

### Consequences

**Positive**:
- Maximum single-bounty loss bounded
- Forces protocol to grow horizontally (more bounties) before vertically (larger bounties), which spreads risk
- Easy to reason about in audit substitute

**Negative**:
- Sophisticated users may want to push more than $10K into a single bounty — initially blocked
- A wealthy "whale" predictor cannot dominate one bounty (some users might see this as feature, not bug — anti-whale fairness)
- Sponsor depositing >$10K into one bounty must split across multiple bounties

**Follow-up**:
- §9 Bounty struct add `uint256 tvlCap`
- §9 add `MAX_BOUNTY_TVL_CAP` constant or default
- §10.1 `IBountyManager.createBounty` signature add optional `tvlCap` parameter (default to constant if zero / unspecified)
- §10.1 add error `BountyTvlCapExceeded(uint256 bountyId, uint256 attempted, uint256 cap)`
- §10.2 `submitPrediction` and `IBountyManager.addSponsorship` enforce cap check
- L2-T3.3 / T4.1 implementations include cap test
- L2-T0.E or future spec lock: review cap value after 6 months operation

---

## ADR-0012 — Emergency pause switch (PausableUpgradeable)

**Status**: Accepted
**Date**: 2026-05-09
**Decided by**: L1 CEO review (Q4 audit compensating controls); L1 ratification in this commit
**Related**: ADR-0011 (TVL cap), ADR-0013 (withdrawal time-lock); TDD V0.31 §4.2 (access control), §10 (interfaces)

### Context

Companion compensating control to ADR-0011 (TVL cap) for the AI-only audit decision. Even with bounded blast radius per bounty, an exploited vulnerability requires human intervention to halt further damage.

OpenZeppelin's `PausableUpgradeable` is the standard mechanism — well-audited, widely deployed, low complexity.

### Decision

All upgradeable contracts (`Treasury`, `RewardDistributor`, `BountyManager`, `PredictionEngine`, `PsychohistoryRouter`, `BuybackExecutor`) inherit `PausableUpgradeable`.

- `DEFAULT_ADMIN_ROLE` may call `pause()` and `unpause()` (no separate `PAUSER_ROLE` for V0.3 — this can be added in V0.4 if multisig governance refines roles)
- All **user-facing state-changing functions** are gated by `whenNotPaused`:
  - `BountyManager.createBounty`, `addSponsorship`, `cancelBounty`
  - `PredictionEngine.submitPrediction`, `submitCutoffHint`, `settle`
  - `BuybackExecutor.executeEpoch`
- **Exit-path functions** are NOT gated — users can always withdraw what they're owed:
  - `PredictionEngine.claim` (claim USDC payout + PSYH from settled bounty)
  - `BountyManager.claimSponsorshipRefund` (sponsor refund on Invalidated / Cancelled / NoSignal)
  - `RewardDistributor.claimTokens` (claim PSYH after settlement)
  - `Treasury.daoWithdraw` (DEFAULT_ADMIN_ROLE; emergency withdrawal possible)

### Rationale

- **Standard pattern**: deviation from norm would itself raise audit concerns
- **Halt new activity, allow exit**: best practice for emergency response — users can rescue funds while admin investigates
- **Single role for V0.3**: governance complexity is a future problem (V0.4 may split out PAUSER_ROLE for multisig delegation)

### Consequences

**Positive**:
- Vulnerability response capability without contract upgrade
- Users retain claim/refund paths during pause
- Standard pattern, easily audited

**Negative**:
- DEFAULT_ADMIN_ROLE has unilateral pause power. **Centralization risk acknowledged** — passion launch operates under single-multisig trust.
- Pause is a censorship surface (admin could pause to delay specific predictors) — mitigation: log all pause/unpause events for community visibility, ADR-0013 time-lock applies to admin withdrawal during pause.

**Follow-up**:
- All upgradeable contracts inherit `PausableUpgradeable`
- §10 each interface has corresponding `Paused` / `Unpaused` events documented
- Specific function-level `whenNotPaused` per the list above
- L2-T3 / T4 / T5 implementations include pause test (paused contract rejects target functions, exit-paths still work)

---

## ADR-0013 — Launch-period withdrawal time-lock (Treasury)

**Status**: Accepted
**Date**: 2026-05-09
**Decided by**: L1 CEO review (Q4 audit compensating controls); L1 ratification in this commit
**Related**: ADR-0011 (TVL cap), ADR-0012 (pause switch); TDD V0.31 §10.4 (Treasury), §13 (deployment)

### Context

Final compensating control for the AI-only audit decision. Even with TVL cap and pause switch, a compromised admin key (one of multisig signers) could rapidly drain Treasury of accumulated DAO sub-account / FEE / BUYBACK balances.

Time-lock pattern: admin actions are *scheduled*, not *executed* immediately. A delay (7 days) gives community / oracle / observers time to detect and trigger emergency pause (ADR-0012) before the action takes effect.

### Decision

Treasury admin-only outflow functions (`daoWithdraw`, `pullBuybackForEpoch` if admin-callable) are gated by a 7-day time-lock during the launch period.

- `Treasury.scheduleDaoWithdrawal(address to, uint256 amount, bytes32 category, string reason) returns (bytes32 withdrawalId)`: schedules a withdrawal, executable after 7 days
- `Treasury.executeDaoWithdrawal(bytes32 withdrawalId)`: callable by anyone after the 7-day delay; transfers funds
- `Treasury.cancelDaoWithdrawal(bytes32 withdrawalId)`: callable by `DEFAULT_ADMIN_ROLE` (or governance vote in V0.4); cancels pending withdrawal

Sunset: A flag `launchPeriodActive` defaults `true`. After ≥ 6 months from deployment, `DEFAULT_ADMIN_ROLE` may call `endLaunchPeriod()` (one-way; cannot be re-enabled). After end-launch:

- New `daoWithdraw` calls execute immediately (no time-lock)
- Pending scheduled withdrawals remain time-locked

`pullBuybackForEpoch` is not time-locked (it's automated and bounded by `1/12 × balance` per epoch — limited blast radius).

### Rationale

- **Compromised admin key buys 7 days** before exfiltration completes — community has detection window
- **Sunset prevents permanent operational friction**: 6 months of stable operation = trust earned, time-lock removed for ergonomics
- **Limited scope**: only admin outflows time-locked, not buyback automation or user claims

### Consequences

**Positive**:
- Admin-key-compromise loss bounded (7-day delay → emergency pause + key rotation possible)
- Bounded by sunset (not eternal friction)
- Compatible with future governance: V0.4 DAO can replace admin role with governance vote

**Negative**:
- Operational friction during launch: legitimate admin withdrawals delayed 7 days
- "launchPeriodActive" sunset is a one-way flag; if admin lost, sunset can't be triggered without governance migration
- Sponsor analytics service operations may need pre-funding to avoid mid-stream withdrawal delays

**Follow-up**:
- §9 / §10.4 Treasury add `pendingWithdrawals` mapping, schedule/execute/cancel functions
- §10.4 add `launchPeriodActive` flag + `endLaunchPeriod()` admin function
- §13 deployment script initializes `launchPeriodActive = true`
- L2-T3.1 implementation includes time-lock tests

---

## ADR-0014 — Qualified Proposition Standard (QPS)

**Status**: Accepted
**Date**: 2026-05-09
**Decided by**: L1 CEO review (Hook #4-C → owner-as-oracle subjective resolution risk dissolved by proposition design rigor); L1 ratification in this commit
**Related**: ADR-0007 (cutoff hint replaceability — also a Schelling-point oracle move); TDD V0.31 §3 (proposition types) — *no spec change*; new doc `docs/PROPOSITION_STANDARD.md`

### Context

The L1 CEO review Hook #4-C surfaced a concern about owner participating in predictions while also serving as oracle — a potential conflict where owner could "interpret" categorical / subjective proposition outcomes to favor their own positions.

The user's response reframed this from a mechanism problem to a **proposition design** problem:

> "理论上可以调整解读、让自己赢——这是我们的预测事件上面的严谨性要求啊。预测事件一个合格的预测事件应该要求就是没有调整解读的空间。"

This insight is structurally important. If the protocol's proposition curation (curator review for V0.3, sponsor self-service for V0.4) enforces a quality bar where every proposition has **zero interpretation latitude**, then:

- Owner-as-oracle has nothing to interpret — they can only read the published source
- No need for mechanism-layer restrictions on owner participation
- Future oracle decentralization is much simpler (any anonymous actor can verify resolution against the named source)
- Proposition design itself becomes the security control, not contract-level multi-sig oracle

This is essentially a **Schelling-point oracle** philosophy applied to proposition design.

### Decision

Establish the **Qualified Proposition Standard (QPS)** as a project-level proposition curation principle:

Every Psychohistory proposition (whether team-curated, sponsor-curated, or self-service) must satisfy 5 conditions:

1. **Single authoritative public source**: resolution comes from one publicly accessible data source (NASDAQ, China NBS, Lloyd's List Intelligence, OpenAI official Twitter, etc.)
2. **Source explicitly named in proposition metadata**: the source URL/identifier is stored in `Bounty.metadataURI` JSON, not informally referenced
3. **Numerical / categorical value unambiguous**: "GDP growth rate" not "economy good/bad"; "weekly transit count" not "is Hormuz blockaded"
4. **Resolution timing unambiguous**: specific date + "first published" / "as-of-X" / "Q-X release"
5. **Failure clause prebuilt**: if source unavailable / publishes differently than expected by `resolutionDeadline`, the proposition resolves as `Invalidated` (refund path)

Detailed standard with examples, anti-examples, edge cases, and curator review checklist is documented in `docs/PROPOSITION_STANDARD.md`.

### Rationale

- **Eliminates owner-as-oracle subjectivity attack surface** at the proposition layer rather than the mechanism layer (see Hook #4-C resolution)
- **Aligns with Schelling-point oracle paradigm** — any honest observer reaches identical resolution
- **Reduces oracle trust load** — paving way for decentralized oracle in V0.4+
- **Provides curator review checklist** — sponsor-self-service proposition (V0.4 candidate) gets a quality gate
- **Clean E-class proposition design**: previously deferred E-type "categorical scenarios" (Hormuz blockade, etc.) becomes feasible at launch by reframing to objective measures (transit count buckets, etc.)

### Consequences

**Positive**:
- Owner participation in own bounties no longer requires mechanism limit (S7 in L1 strategic decisions)
- E-class propositions feasible at launch (S4 amended)
- Future oracle decentralization simplified
- Curation quality bar explicit

**Negative**:
- Some real-world questions cannot be qualified (e.g., "did war start" without a specific source) — these are deliberately out of scope
- Curator workload during V0.3 launch (team is curator, must enforce standard manually)

**Follow-up**:
- New file `docs/PROPOSITION_STANDARD.md` with full standard + examples
- TDD V0.32 §3 references QPS for proposition design discipline (no spec change)
- L2-T3.3 (BountyManager) implementation: `metadataURI` JSON schema must accommodate QPS source/timing/failure fields (off-chain schema, no contract enforcement)
- Future V0.4: front-end / curator review tool enforces QPS at proposition creation flow

---

## Future ADRs

The next ADR slot is **ADR-0015**. Candidates that may need ADRs in the future:

- DEX venue choice (Uniswap V3 vs CoW Protocol vs both) — to be decided in L2-T5b
- Cutoff hint griefing mitigation (refundable bond) — V0.4 candidate; depends on V0.3 production observations
- Cross-chain strategy (LayerZero vs Wormhole vs single-chain) — explicitly deferred per TDD §16
- Oracle decentralization path — Phase 4 scope per TDD §7.4
- Front-end privy / authentication choice — out of scope of TDD per §16
- KYC for high-value sponsors — out of scope of TDD per §16
- Multi-winner Discrete proposition support (K-of-N selection) — V0.4 candidate per `docs/TODOS.md`
- A2-oracle-encrypted privacy mode (V0.4 candidate per ADR-0001 hooks; demand-driven by画像 A sponsor presence)
- A* threshold-encryption privacy mode (longer-term per ADR-0001)
- Jurisdiction strategy formalization (currently per `docs/TODOS.md` triggers; full ADR pending entity registration)
