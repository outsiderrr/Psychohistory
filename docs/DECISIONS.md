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

## Future ADRs

The next ADR slot is **ADR-0008**. Candidates that may need ADRs in the future:

- DEX venue choice (Uniswap V3 vs CoW Protocol vs both) — to be decided in L2-T5b
- Cutoff hint griefing mitigation (refundable bond) — V0.4 candidate; depends on V0.3 production observations
- Cross-chain strategy (LayerZero vs Wormhole vs single-chain) — explicitly deferred per TDD §16
- Oracle decentralization path — Phase 4 scope per TDD §7.4
- Front-end privy / authentication choice — out of scope of TDD per §16
- KYC for high-value sponsors — out of scope of TDD per §16
