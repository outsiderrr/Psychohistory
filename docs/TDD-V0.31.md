# Psychohistory — Technical Design Document

**Project Version:** V0.3
**Document Revision:** V0.31 (2026-05-08)
**Supersedes:** V0.30 (2026-05-08, initial V0.3 draft); internal V0.1 / V0.2 designs are deprecated and not referenced.
**Audience:** Claude Code master planning conversation. This document is the single source of truth for architecture. Use it to decompose work into subtasks, each of which will be developed in independent conversations. The companion document [docs/L1-PLAN.md](L1-PLAN.md) records the L1 master plan, L3 task decomposition, and ratified design decisions.

## Revision History (V0.30 → V0.31)

V0.31 incorporates the L1.B / L1.C output:

- **§1, §8** — Sponsor value proposition rewritten from "cryptographic secrecy of early signal" to "analytics SLA + standardized aggregate API + timeliness/format advantage + service priority". V0.3 走 B 路（透明） + 留 5 条 V0.4 升级架构口子。
- **§3, §5.2** — Numerical score direction unified: all settlement uses high-score-is-better.
- **§5.4** — Renamed *principal protection* → *net-of-fee principal protection*; all Pool 1 math based on `effectiveWager = wager × 0.99`.
- **§5.5** — Slice A fallback for empty bottom group routes to DAO sub-account under new category `P2_UNALLOCATED`.
- **§5.6** — `p1RemainderShareExcluded` removed; amount-pool weight is the explicit `pool1Payout = effectiveWager + remainderShare`.
- **§5.7** — Platform fee model: 1% deducted at submission (each predictor's `effectiveWager = wager × 0.99`); no edge-case underflow.
- **§5.9** — Zero-predictor case clarified: sponsors refund via `claimSponsorshipRefund()` (consistent with §8.4); no implicit invalidation. Score-0 fallback added (avoid division by zero).
- **§6.4** — `K` defined as WAD-scaled PSYH-per-USDC (`kWad`); allocation formula `usdcRaw × kWad / 1e6`. Monthly cap reservation moved from settlement-time global mutation to per-bounty `reserveRewards()` lock-in.
- **§6.5** — Buyback formula explicitly geometric ("rolling smoothing", `1/12` of current balance per epoch); no tranche queue.
- **§9** — `Prediction` adds `rawWager / effectiveWager / feeAmount` and `bytes encryptedPayload`. `Bounty` adds `PrivacyMode privacyMode`. `SettlementState` adds `currentPass / passCompletedFlags / cutoffHintSubmitted / cutoffHintScore`. New constants: `MAX_SPONSORS_PER_BOUNTY = 100`, `K_WAD_*` schedule.
- **§10.1** — `IBountyManager` adds PE-only mutators (`recordPrediction / closeBounty / markResolved / markSettled / markInvalidated`); `addSponsorship` enforces 100-sponsor cap; `SponsorRefundClaimed` event carries reason code.
- **§10.2** — `IPredictionEngine.submitPrediction` adds `bytes encryptedPayload` parameter; new `submitCutoffHint()`; reward claim path unified via `claim()`.
- **§10.3** — `IRewardDistributor` rewritten as 4-function reserve/assign/finalize/claim API; cap consumed at reserve, not at claim.
- **§10.4** — `ITreasury` adds `P2_UNALLOCATED` category; per-category balance accounting.
- **§11** — Settlement passes use `effectiveWager`; cutoff hint固化 as permissionless + on-chain paginated verification; final finalization uses commit-then-paginated token reward (matches §10.3 reserve/assign).
- **§12.4** — Removed "predictions are private" claim. Front-end opacity is convention, not security boundary.
- **§12.5** — Cutoff hint trust model fixed to permissionless.
- **§13** — Deploy split into `DeployCore.s.sol` (Phase 1) + `DeployBuyback.s.sol` (Phase 3).
- **§14** — Tests added for sponsor cap, score-0 fallback, effectiveWager invariants, RewardDistributor reserve/assign/finalize, encryptedPayload empty-check.
- **§15** — T4.3 split into a/b/c/d/e (五段); L2-T5 split into Router (T5.1) + BuybackExecutor (T5.2). Operational details, see [docs/L1-PLAN.md](L1-PLAN.md).

---

## 1. Design Philosophy & Positioning

Psychohistory is a decentralized prediction protocol that combines three elements absent from existing predictions markets:

1. **Unified discrete-and-quantitative framework.** Predictors always submit a structured prediction (probability distribution for discrete events, numeric value for quantitative events) paired with a free-form wager amount. Polymarket's AMM architecture cannot natively express quantitative predictions. This is the primary technical moat.
2. **Score-ranked settlement with top-50% cutoff.** Not "correct direction wins" (Polymarket), not "proportional to confidence" (soft ranking). Instead: only predictors in the top-50% score bracket share Pool 1 and Pool 3. Bottom-50% receives only a consolation slice of the sponsor pool. This enforces hard differentiation between good and mediocre predictions.
3. **Sponsor-funded analytics tier.** Sponsors fund a secondary prize pool and receive staggered post-close delivery of curated aggregate analytics — clean schema, machine-readable API, timeliness/format guarantees, and human/AI support priority. The auction equilibrium price reflects the value of analytics convenience and SLA, not cryptographic secrecy. Predictions are accepted to be on-chain transparent in V0.3; sponsors do not pay for exclusive information access. (Optional V0.4 upgrade may add encrypted submission and threshold/oracle decryption to recover an information-asymmetry tier; V0.3 ships with architectural hooks to enable that upgrade without rewrite — see §9 `PrivacyMode`, §10.2 `encryptedPayload`.)

**What Psychohistory is NOT:**

- Not an AMM. No market-making, no continuous price discovery during the prediction window, no liquidity pools. Predictions lock at submission and settle once at resolution.
- Not a pure PvP casino. Sponsor pool creates a third source of funds beyond loser stakes, enabling sustainable value flow even in low-volume markets.
- Not a fully decentralized protocol at launch. Proposition creation is centralized (team-curated). Decentralized challenge mechanisms are deferred to future phases.
- **Not (in V0.3) a privacy-preserving protocol.** All predictions, wagers, and aggregates are derivable from on-chain state by anyone willing to run an indexer. Front-ends must not represent prediction privacy as a security feature. See §12.4.

**Primary user personas:**

- **Predictors:** Individual users submitting predictions with real USDC wagers. Motivated by (a) winning other predictors' wagers (Pool 1 + Pool 2 victory bonus), (b) sponsor pool consolation (bottom-50%), (c) earning $PSYH tokens through prediction mining (top-50% only).
- **Sponsors:** Entities that fund a prediction with USDC in exchange for staggered access to a curated aggregate-analytics service. Motivated by (a) saving the ~$5K–$50K engineering and ongoing-maintenance cost of a private indexer, (b) format/SLA reliability for downstream products, (c) timeliness within minutes of close vs. hours-to-days of DIY indexing, (d) optional analyst/AI support priority. Sponsors are NOT paying for cryptographic exclusivity in V0.3.
- **Arbitrageurs:** A subset of predictors and sponsors who exploit divergences between Psychohistory's aggregated signal and external markets like Polymarket. Psychohistory's signal is valuable whether it is more accurate or less accurate than Polymarket, as long as it is independent — this enables bidirectional arbitrage and removes the pressure to "beat Polymarket" as a cold-start requirement.

---

## 2. Tech Stack & Conventions

- **Solidity:** ^0.8.20 (toolchain configured for 0.8.27)
- **Framework:** Foundry (forge, cast, anvil)
- **Libraries:** OpenZeppelin Contracts Upgradeable v5.x, OpenZeppelin Contracts v5.x
- **Upgradeability:** Transparent Proxy pattern for all stateful contracts. `BrierMath` and other pure libraries are linked at compile time and do not use proxies.
- **Math:** All multiplication-then-division uses `Math.mulDiv` to prevent intermediate overflow. Fixed-point representations use WAD (1e18) for continuous values and BPS (10,000) for probabilities.
- **Token transfers:** `SafeERC20` everywhere. No direct `transfer` / `transferFrom` calls.
- **Reentrancy:** `nonReentrant` on every function that moves tokens. Checks-Effects-Interactions pattern throughout.
- **Storage safety:** `uint256[50] private __gap;` at end of every upgradeable contract.
- **License:** `// SPDX-License-Identifier: MIT` on every file.
- **NatSpec:** Required on every external/public function.

**Token specifications:**

- **USDC:** 6 decimals. Used for all stakes, sponsor pools, and payouts.
- **$PSYH:** 18 decimals. Native governance and reward token.

**Environment assumptions:**

- Target chain: any EVM-compatible chain with USDC support (Ethereum mainnet, Base, Arbitrum, Polygon, etc.)
- Oracle: single `ORACLE_ROLE` held by a multisig or the team at launch. Decentralized oracle is a future upgrade.

---

## 3. Proposition Types

There are exactly two proposition types. This is a deliberate simplification from earlier designs that had three types.

### 3.1 Discrete Propositions

Applicable to any event with `N` mutually exclusive outcomes, where `2 ≤ N ≤ MAX_OPTIONS (5)`.

- Traditional binary (Yes/No) is simply `N=2`, handled by the same code path as `N=3,4,5`.
- Predictor input:
  - `selectedOption`: unused for scoring purposes (see note below).
  - `confidenceBpsArray`: an array of `N` integers in basis points, must sum to exactly `10_000`.
  - `wagerAmount`: USDC amount, minimum 1 USDC (1e6 raw).
- Scoring: Brier distance `BS = Σ(f_j - o_j)²` over all `j ∈ [0, N)`, where `o_j = WAD` if option `j` is the resolved outcome, else `0`. Lower is better. Range `[0, 2×WAD]`.

**Note on `selectedOption`:** In earlier V2 design, discrete predictions had both a `selectedOption` (for win/loss) and a `confidenceBpsArray` (for Brier weighting). In this V3 design, the `selectedOption` concept is eliminated. The Brier score alone determines ranking — a predictor's full probability distribution is always used. This is philosophically cleaner: we reward calibrated probabilistic thinking, not binary correctness. Users in UI may still be guided by a "primary choice" field, but on-chain only the `confidenceBpsArray` matters.

### 3.2 Numerical Propositions

Applicable to any event with a continuous numeric outcome.

- Predictor input:
  - `predictedValue`: scaled integer.
  - `predictedDecimals`: decimal precision (≤18).
  - `wagerAmount`: USDC amount, minimum 1 USDC.
- Scoring (intermediate): normalized absolute error. Given WAD-scaled predicted value `p` and resolved value `a`, raw error is `|p - a|`. The raw error is never used directly for ranking; it is converted to a unified high-score-is-better `score` per §5.2.

### 3.3 Unification

Both types resolve into a **single sortable `score` value per prediction, where higher is always better.** This direction is uniform across all settlement logic — Pass 1 score computation, Pass 2 cutoff determination, Pass 3/4 weighted allocation, Pool 3 token reward weights — there is no place in the protocol where "lower is better" applies. `BrierMath` exposes distinct scoring functions for Discrete and Numerical but normalizes both to high-is-better before returning. The settlement pipeline is therefore type-agnostic once `score` is computed. See §5.2 for the unification formulas.

---

## 4. Contract Architecture

### 4.1 Contract Topology

| Contract | Type | Responsibility |
|---|---|---|
| `BrierMath.sol` | Library (linked) | Pure math: Brier score, normalized error, ranking-based payout helpers |
| `Treasury.sol` | Proxy | Accumulates fees and sponsor-pool buyback allocations; executes TWAP buybacks & burns |
| `PsychohistoryToken.sol` | Non-upgradeable ERC20Votes | $PSYH token. Mint-gated by `MinterRole`. Transfers toggleable by phase. |
| `RewardDistributor.sol` | Proxy | Computes and mints $PSYH rewards post-settlement. Holds the 40% mining allocation. |
| `BountyManager.sol` | Proxy | Proposition lifecycle: creation, sponsor deposits, state machine, sponsor refund logic |
| `PredictionEngine.sol` | Proxy | Prediction submission, resolution, paginated settlement, payout claim |
| `BuybackExecutor.sol` | Proxy | Executes TWAP buybacks on Uniswap V3 / CoW Protocol. Controlled by Treasury. |
| `PsychohistoryRouter.sol` | Proxy | Thin facade for multi-step atomic operations |

### 4.2 Access Control Roles

```
DEFAULT_ADMIN_ROLE            → Multisig (or deployer at launch)
ORACLE_ROLE                   → Oracle multisig / team
PREDICTION_ENGINE_ROLE        → PredictionEngine proxy (granted on Treasury, RewardDistributor, BountyManager)
TREASURY_EXECUTOR_ROLE        → BuybackExecutor proxy (granted on Treasury for withdrawals)
MINTER_ROLE                   → RewardDistributor proxy (granted on PsychohistoryToken)
TRANSFER_CONTROLLER_ROLE      → DEFAULT_ADMIN_ROLE (controls PsychohistoryToken transfer toggle during phased rollout)
```

### 4.3 Interaction Flow

```
Sponsor → BountyManager.createProposition()                      [initial sponsor deposit]
Sponsor → BountyManager.addSponsorship()                         [bidding during open window]
Predictor → PredictionEngine.submitPrediction()                  [wager in USDC]
Oracle → PredictionEngine.resolve() OR resolveAsInvalid()
Anyone → PredictionEngine.settle(startIndex, endIndex)           [paginated]
  ├─ Computes scores, ranks, top-50% cutoff
  ├─ Distributes Pool 1 (predictor wager pool)
  ├─ Distributes Pool 2 (sponsor pool) in 4 slices: 30/30/20/20
  ├─ Sends 20% of Pool 2 to Treasury (buyback allocation)
  ├─ Sends 10% of Pool 2 to team wallet, 10% to DAO fund (via Treasury)
  ├─ Calls RewardDistributor.distributeRewards() for Pool 3 ($PSYH)
  └─ Sets each predictor's claimable payout and token reward
Winner → PredictionEngine.claim()                                [pull USDC payout]
Winner → RewardDistributor.claimTokens()                         [pull $PSYH]
Sponsor → BountyManager.claimSponsorshipRefund()                 [only if Invalidated or no predictors]
```

---

## 5. Three Prize Pools & Settlement Mathematics

This is the mathematical heart of the protocol. Every rule below must be implemented exactly as specified.

### 5.1 Pool Definitions

> **Wager terminology.** Throughout §5 and §11, `wager` (when used as a weight or in distribution math) refers to **`effectiveWager = rawWager × (1 - PLATFORM_FEE_BPS/10000) = rawWager × 0.99`**. The 1% fee is deducted at submission time once and the residual `effectiveWager` is the only quantity that participates in any subsequent math. `rawWager` is preserved for accounting/audit only. See §5.7 for the fee model and §9 `Prediction` struct for storage fields.

**Pool 1 — Predictor Wager Pool**
- Source: sum of all predictor `effectiveWager` for this proposition.
- Recipients: Top-50% predictors only.
- Distribution rule: detailed in §5.4.

**Pool 2 — Sponsor Pool**
- Source: sum of all sponsor deposits for this proposition.
- Split: `30% / 30% / 20% / 10% / 10%`
  - **30% consolation:** bottom-50% predictors, weighted by `score × effectiveWager`.
  - **30% victory bonus:** top-50% predictors, weighted by `score × effectiveWager`.
  - **20% buyback allocation:** transferred to Treasury under category `BUYBACK`.
  - **10% team allocation:** transferred to team wallet.
  - **10% DAO fund allocation:** transferred to Treasury under category `DAO`.

**Pool 3 — Token Reward Pool**
- Source: `RewardDistributor` mints from the 40% mining allocation, gated by per-bounty reservation against the monthly emission cap (see §6.4 and §10.3).
- Total tokens per proposition (reserved at settlement start):
  ```
  propositionTokenAllocation = (P1_rawTotal + P2_total) × kWad / 1e6
  ```
  where `kWad` is the WAD-scaled coefficient (see §6.4) and the divisor `1e6` converts USDC's 6-decimal unit to PSYH's 18-decimal unit. P1 uses raw (pre-fee) total here so that token incentive scales with full economic activity.
- Split: `50% / 50%`
  - **50% amount pool:** top-50% predictors, weighted by their Pool 1 winnings (`pool1Payout = effectiveWager + remainderShare`, see §5.4 and §5.6).
  - **50% quality pool:** top-50% predictors, weighted by `score × effectiveWager`.
- Bottom-50% predictors receive zero $PSYH.

### 5.2 Scoring Functions

**Discrete proposition:**
```
score = (2 × WAD) - computeCategoricalBrierScore(confidenceWadArray, resolvedOptionIndex)
```
Higher score is better. Range `[0, 2×WAD]`.

**Numerical proposition:**
```
rawError = computeAbsoluteError(predictedValueWad, resolvedValueWad)
```
Used directly for ranking. Lower `rawError` is better. For score-weighted allocation (required in Pool 2 consolation/victory and Pool 3 quality slice), convert:
```
score = WAD_SQUARED / max(rawError, MIN_BRIER)
```
This gives higher values for better predictions, symmetric with the discrete case.

### 5.3 Ranking and Top-50% Cutoff

Rank all predictions in descending order by `score`.

Define the cutoff as follows. Let `n` be the total number of predictions.

- `targetCount = ceil(n / 2)`.
- Walk down the sorted list. The cutoff is the score at position `targetCount - 1` (0-indexed).
- **Include all ties:** every prediction with score `>= cutoffScore` is in the top group. This means the top group may exceed `targetCount` if ties span the cutoff.

**Example:** 10 predictors, scores `[100, 95, 95, 95, 80, 70, 60, 50, 40, 30]`. Target count is 5. Position 4 (0-indexed) has score `80`. Cutoff is `80`. Top group includes all scores `>= 80`, which is positions 0–4 = 5 predictors. No ties span the cutoff in this case.

**Example with ties:** 10 predictors, scores `[100, 95, 90, 90, 90, 90, 80, 70, 60, 50]`. Target count is 5. Position 4 has score `90`. Cutoff is `90`. Top group includes all scores `>= 90`, which is 6 predictors (positions 0–5).

### 5.4 Pool 1 Distribution (Net-of-Fee Principal Protection + Remainder)

The rule below is *net-of-fee principal protection*: each top-group predictor is guaranteed back **`effectiveWager_i = rawWager_i × 0.99`**, never less. The 1% fee was already deducted at submission (§5.7) and lives in `Treasury` from the moment of submission; settlement does not re-touch it. This eliminates the V0.30 underflow edge case where `0.99 × rawTotal < topPrincipal` in all-tie / single-predictor scenarios.

Let:
- `P1_effectiveTotal = Σ effectiveWager_i over all i` = sum of all predictor effective wagers (already net of fee).
- `T` = set of top-50% predictors (by score cutoff rule in §5.3).
- `B` = set of bottom-50% predictors. `B` is empty in all-tie / single-predictor cases.
- `P1_principal = Σ effectiveWager_i for i ∈ T`.
- `P1_remainder = Σ effectiveWager_i for i ∈ B = P1_effectiveTotal - P1_principal`. (Always ≥ 0; equals 0 when `B = ∅`.)

Distribution:
1. Each `i ∈ T` first receives back their own `effectiveWager_i` (net-of-fee principal protection).
2. `P1_remainder` is distributed among `T` proportionally to `score_i × effectiveWager_i`:
   ```
   remainderShare_i = P1_remainder × (score_i × effectiveWager_i) / Σ(score_j × effectiveWager_j for j ∈ T)
   ```
   If `Σ(score_j × effectiveWager_j for j ∈ T) == 0` (degenerate: all top-group scores are 0), fall back to weighting by `effectiveWager_i` alone. If that sum is also 0 (impossible — `effectiveWager` is always ≥ `MIN_WAGER × 0.99`), revert.
3. Each `i ∈ T` receives `pool1Payout_i = effectiveWager_i + remainderShare_i`.

**Invariant (conservation):** `Σ pool1Payout_i for i ∈ T = P1_effectiveTotal`, modulo rounding dust which is swept to Treasury under category `DUST` at end of `settle()`.
**Invariant (no-loss):** Every `i ∈ T` receives ≥ `effectiveWager_i`. They never lose net-of-fee principal in Pool 1.
**Invariant (bottom-group):** Every `i ∈ B` receives `pool1Payout_i = 0`. Bottom-group predictors recover nothing from Pool 1; their consolation comes from Pool 2 Slice A only.

### 5.5 Pool 2 Distribution

Let `P2_total` = sum of all sponsor deposits for this proposition.

**Slice A — Consolation (30% of P2_total):**
- Distributed to bottom-50% predictors `B`.
- `P2_A_amount = P2_total × P2_CONSOLATION_BPS / BPS = P2_total × 0.30`.
- For each `i ∈ B`:
  ```
  consolation_i = P2_A_amount × (score_i × effectiveWager_i) / Σ(score_j × effectiveWager_j for j ∈ B)
  ```
- **Score-0 fallback:** if `Σ(score_j × effectiveWager_j for j ∈ B) == 0` (all bottom-group scores are 0; can occur when every bottom-group predictor's Brier score equals `2 × WAD`), fall back to weighting by `effectiveWager_i` alone within `B`. If `Σ effectiveWager_j for j ∈ B` is also 0 (impossible; `B ≠ ∅` implies at least one wager), revert.
- **Empty bottom group:** if `B = ∅` (all predictors are in top group due to ties or single-predictor case), `P2_A_amount` is **not** distributed and **not** routed to buyback or fee. It is transferred to Treasury under the new dedicated category `P2_UNALLOCATED` and settled into the DAO-controlled sub-account, joining Slice D2 funds operationally but tagged separately for accounting clarity. Rationale: auto-buyback would distort token monetary policy in all-tie/single-predictor scenarios; routing to DAO is monetary-neutral and lets future DAO governance decide use (e.g., airdrop, buyback, infrastructure grant). Pre-DAO this remains under admin multisig custody, identical to Slice D2 today.

**Slice B — Victory Bonus (30% of P2_total):**
- Distributed to top-50% predictors `T`.
- `P2_B_amount = P2_total × P2_VICTORY_BPS / BPS = P2_total × 0.30`.
- For each `i ∈ T`:
  ```
  victoryBonus_i = P2_B_amount × (score_i × effectiveWager_i) / Σ(score_j × effectiveWager_j for j ∈ T)
  ```
- **Score-0 fallback:** if `Σ(score_j × effectiveWager_j for j ∈ T) == 0` (degenerate; the entire top group scored 0), fall back to weighting by `effectiveWager_i` alone within `T`. By construction `T ≠ ∅` whenever any prediction exists.

**Slice C — Buyback (20% of P2_total):**
- Transferred to `Treasury` under category `BUYBACK`. Funds enter the geometric buyback queue (§6.5) and are gradually used to purchase and burn $PSYH on the open market once `BuybackExecutor.activate()` has been called (Phase 3 onward).

**Slice D1 — Team (10% of P2_total):**
- Transferred directly to team wallet address (configurable, set via `DEFAULT_ADMIN_ROLE`).

**Slice D2 — DAO Fund (10% of P2_total):**
- Transferred to `Treasury` under category `DAO`, accumulating in the DAO-controlled sub-account.
- Before DAO formation: admin multisig custody. Documented in whitepaper and on-chain metadata.

**Per-bounty Pool 2 invariant:**
```
Σ Slice A + Σ Slice B + Slice C + Slice D1 + Slice D2 = P2_total (modulo rounding dust)
```
Rounding dust is swept to Treasury under category `DUST`.

### 5.6 Pool 3 Distribution

Total token allocation for this proposition is reserved at the start of settlement (after Pass 2 completes and `T` is known) by calling `RewardDistributor.reserveRewards(bountyId, P1_rawTotal + P2_total)`. The reservation locks against the monthly emission cap immediately, preventing late-claim contention with other bounties (see §10.3 and §6.4).

```
propositionTokenAllocation = (P1_rawTotal + P2_total) × kWad / 1e6
```
where `kWad` is the WAD-scaled coefficient at reservation time (see §6.4); the divisor `1e6` converts USDC's 6-decimal raw amount to PSYH's 18-decimal raw amount. P1 raw (pre-fee) total is used so that the token-incentive scales with full economic activity, not net-of-fee.

**Slice A — Amount Pool (50% of allocation):**
- Recipients: top-50% predictors `T`.
- Weight: each predictor's Pool 1 winnings, defined as
  ```
  pool1Payout_i = effectiveWager_i + remainderShare_i      // i ∈ T
  ```
  This is the same quantity computed in §5.4. Sponsor Slice B (victoryBonus) is **not** included in the weight — Pool 3 amount slice rewards Pool-1-economic activity only.
- For each `i ∈ T`:
  ```
  amountTokens_i = (propositionTokenAllocation / 2) × pool1Payout_i / Σ(pool1Payout_j for j ∈ T)
  ```

**Slice B — Quality Pool (50% of allocation):**
- Recipients: top-50% predictors `T`.
- Weight: `score_i × effectiveWager_i`.
- For each `i ∈ T`:
  ```
  qualityTokens_i = (propositionTokenAllocation / 2) × (score_i × effectiveWager_i) / Σ(score_j × effectiveWager_j for j ∈ T)
  ```
- **Score-0 fallback:** if `Σ(score_j × effectiveWager_j for j ∈ T) == 0`, fall back to weighting by `effectiveWager_i` alone within `T`.

Total tokens for predictor `i ∈ T`: `amountTokens_i + qualityTokens_i`.
Total tokens for predictor `i ∈ B`: `0`.

**Note on V0.30's `p1RemainderShareExcluded`:** Earlier draft §11 referenced a variable `p1RemainderShareExcluded` for this slice. That variable is **removed in V0.31**; the explicit `pool1Payout_i` defined in §5.4 is the single source of truth for Slice A's weight.

### 5.7 Platform Fee

A flat **1%** platform fee is collected on each prediction at submission time, not at settlement. This is the **per-predictor pre-deduction model** ratified in L1.C.

When a predictor submits a prediction with `rawWager` USDC:
```
feeAmount      = rawWager × PLATFORM_FEE_BPS / BPS = rawWager × 0.01   // rounded down
effectiveWager = rawWager - feeAmount                                    // exactly equals rawWager × 0.99 modulo rounding
```

`feeAmount` is transferred directly to `Treasury` under category `FEE` during `submitPrediction()`. `effectiveWager` is the only quantity that enters Pool 1 / Pool 2 / Pool 3 math thereafter; `rawWager` is preserved on the `Prediction` struct for accounting/audit but never participates in distribution.

Consequences (compared to V0.30's settlement-time fee deduction):
- **No principal-protection underflow.** §5.4 net-of-fee principal protection is mathematically tight; `effectiveTotal == Σ effectiveWager` by construction, so `effectiveTotal ≥ topPrincipal` always.
- **Protocol revenue is exactly 1% of `rawTotal`**, regardless of all-tie / single-predictor edge cases.
- **No branching in distribution code.** All formulas use `effectiveWager` uniformly.
- **Submission flow** must `transferFrom(predictor, Treasury, feeAmount)` and `transferFrom(predictor, address(this), effectiveWager)` atomically (or one combined transfer with internal split).

No fee on Pool 2 (sponsors already pay via the 20% buyback + 10% team + 10% DAO allocations, totaling 40% platform take).

**Invariant (per-predictor):** `rawWager_i = effectiveWager_i + feeAmount_i` always, exactly, with `feeAmount_i = rawWager_i × 100 / 10000` rounded down. Storage MUST persist all three for audit.

**Invariant (protocol total):** `Σ feeAmount_i for all i == platformFeeBalance(bountyId)` at any time.

### 5.8 Invalidation and Total Refund

If the oracle calls `resolveAsInvalid()`:

- All predictors get 100% wager refund. No ranking, no scoring, no $PSYH minting.
- Sponsors get 100% deposit refund via `BountyManager.claimSponsorshipRefund()`.
- No fees collected.
- `BountyState` transitions to `Invalidated` then `Settled` after refunds processed.

### 5.9 Edge Cases

- **Zero predictors at resolution (`totalPredictors == 0`).** No Pool 1 distribution (no Pool 1 funds — the fee was never collected because no predictions were ever submitted). Pool 2 is **fully refunded** to sponsors via `BountyManager.claimSponsorshipRefund()`. Slices A/B/C/D1/D2 do **not** execute; the bounty's terminal state is `Settled` with sponsor refund predicate. The refund event carries `reasonCode: NO_SIGNAL` (distinct from `INVALIDATED` and `CANCELLED` — see §10.1). This rule is consistent with §8.4 and supersedes V0.30 §5.9's contradictory "no refund either" prose.
- **All predictors tie.** If all `n` predictors have identical scores, they all enter `T` (ties include rule, §5.3); `B = ∅`. Pool 1 net-of-fee principal protection still works — each `i ∈ T` recovers `effectiveWager_i`; `P1_remainder == 0` so no additional Pool 1 distribution. Pool 2 Slice A → Treasury under `P2_UNALLOCATED` (§5.5). Pool 2 Slice B distributed via `effectiveWager_i` weighting (the score-0 fallback if all scores happen to be 0).
- **Only one predictor.** `T = {only}`, `B = ∅`. Same handling as "all tie."
- **All scores are 0** (degenerate). Possible for Discrete when every predictor's Brier score equals `2 × WAD`. The score-0 fallback rules in §5.4 / §5.5 / §5.6 apply (weight by `effectiveWager` alone). The bounty still settles deterministically.
- **Rounding dust.** All `mulDiv` operations round down. Residual dust at end of `settle()` is swept to Treasury under category `DUST`.
- **Encrypted payload (V0.4 path).** V0.3 settlement reads `confidenceBpsArray` / `predictedValue` directly from storage. V0.4 will route this through `_getPrediction(bountyId, predictor)` which can decrypt `encryptedPayload` based on `Bounty.privacyMode`. V0.3 implementations MUST already use the helper indirection so that V0.4 is a single helper-body change rather than a settlement rewrite (see §12.4 hooks list).

---

## 6. Token Economics

### 6.1 Token Specification

- **Name:** Psychohistory Token
- **Symbol:** PSYH
- **Standard:** ERC-20 + ERC20Votes (OpenZeppelin) + ERC20Permit
- **Decimals:** 18
- **Total Supply:** 1,000,000,000 PSYH (1 billion)
- **Upgradeable:** No. Deployed once, immutable. Proxy not used.

### 6.2 Allocation

| Bucket | Percentage | Amount | Vesting / Release |
|---|---|---|---|
| Prediction Mining | 40% | 400,000,000 | Cap, **not** pre-mint. Released on-demand via `RewardDistributor.MINTER_ROLE`, gated by §6.4 monthly cap and per-bounty reservation. |
| Team | 15% | 150,000,000 | 1-year cliff, 3-year linear vesting after |
| DAO Treasury | 20% | 200,000,000 | Locked; unlocked by DAO governance post-formation |
| Liquidity Reserve | 10% | 100,000,000 | Used for initial Uniswap pool seeding in Phase 3 |
| Airdrop / Marketing | 10% | 100,000,000 | Allocated over 4 years; early airdrops at milestones |
| Reserve | 5% | 50,000,000 | Contingency; no predefined schedule |

**Mining bucket model (decision 5).** The 400 M PSYH for Prediction Mining is **not pre-minted into `RewardDistributor`**. Instead `RewardDistributor` holds `MINTER_ROLE` on `PsychohistoryToken` and mints **on demand** when predictors call `claimTokens()`, accounting against an internal `mintingCapRemaining` counter initialized to `400_000_000 × 1e18` and decreasing with every successful claim. Consequences:
- `PsychohistoryToken.totalSupply()` exactly equals the cumulative quantity of PSYH that has flowed to predictors / liquidity / team / etc. — no inflated supply due to held-but-undistributed mining bucket.
- Compromise of the `RewardDistributor` proxy exposes at most the *next batch* of mints, not the entire 400 M reserve.
- The mining cap is enforced by `mintingCapRemaining` plus the §6.4 monthly cap; the token contract does **not** itself know about the 40% cap.

### 6.3 Transfer Gating

`PsychohistoryToken` has a `_transfersEnabled` boolean flag, initially `false`. Only `TRANSFER_CONTROLLER_ROLE` can flip it. Minting (by `RewardDistributor`) is always allowed. During Phase 1 (0–3 months post-launch), transfers are disabled; users accumulate but cannot move tokens.

See §7 for phased rollout schedule.

### 6.4 Mining Schedule

The `K` coefficient defines how many PSYH are minted per USDC of proposition volume. **`K` is stored on-chain WAD-scaled** as `kWad` (i.e., the human-readable value `K = 10` is encoded as `kWad = 10 × 1e18`). The Pool 3 allocation formula (§5.6) is:

```
propositionTokenAllocation = (P1_rawTotal + P2_total) × kWad / 1e6
```

The divisor `1e6` converts the USDC raw amount (6 decimals) so the result is in PSYH raw units (18 decimals). Worked example: `P1+P2 = 1 USDC = 1e6 raw`, `kWad = 10e18` → `allocation = 1e6 × 10e18 / 1e6 = 10e18 = 10 PSYH`.

**Schedule (`kWad` by month since `PsychohistoryToken` deployment):**

```
Month  1–3:   kWad = 10  × 1e18    (aggressive early incentive)
Month  4–6:   kWad =  5  × 1e18
Month  7–12:  kWad =  2  × 1e18
Month 13–24:  kWad =  1  × 1e18
Month 25–36:  kWad =  0.5 × 1e18
Month 37–48:  kWad =  0.25 × 1e18
Month 49+:    kWad =  0               (mining ends)
```

The schedule constants are baked in at deploy time (immutable; see §9 Constants). `RewardDistributor.currentK()` returns the schedule value for `block.timestamp` (no admin override).

**Monthly emission cap (front-loaded):**
```
Month  1–12:  monthly cap chosen so cumulative ≤ 200M PSYH (50% of mining allocation)
Month 13–24:  cumulative ≤ 320M PSYH (+30%)
Month 25–36:  cumulative ≤ 370M PSYH (+12.5%)
Month 37–48:  cumulative ≤ 400M PSYH (+7.5%, completing the bucket)
```

Per-month caps are derived by the `RewardDistributor` contract from these tier totals (e.g., months 1–12 each get `200M / 12 ≈ 16.67M PSYH`). Implementation detail.

**Reservation model (decision 7).** Cap enforcement is **at reservation time, not at claim time**. When settlement starts (Pass 3 finalization), `PredictionEngine` calls:

```solidity
RewardDistributor.reserveRewards(bountyId, P1_rawTotal + P2_total)
  → returns (uint256 reservedAmount, uint256 effectiveKWad)
```

The function reads `currentK()`, computes the requested allocation, clamps it to whatever monthly-cap headroom remains, and **decrements the month's cap accounting in storage immediately**. The returned `effectiveKWad` may be less than `currentK()` if the cap was approached — but is **the same `kWad` for every predictor in this bounty**, locked at reservation. Subsequent `assignRewards()` and `claimTokens()` calls do not re-check the cap; they only consume against `reservedAmount`.

This eliminates V0.30's race condition where a late `claimTokens()` for an old bounty could be reduced because a newer high-volume bounty later in the same month consumed the cap. Each bounty's `kWad` is fixed at reservation; the cap is always evaluated against the bounty that arrived first.

If `reserveRewards` is called and the month's cap is fully exhausted (`effectiveKWad == 0`), the bounty proceeds to settlement with **zero Pool 3 reward** (Pool 1 and Pool 2 still distribute normally). The `KCoefficientUpdated(0)` event is emitted for off-chain monitoring.

### 6.5 Buyback & Burn

**Source of buyback funds:**
- 1% platform fee from Pool 1 (collected per-prediction at submission, see §5.7)
- 20% of Pool 2 (transferred at settlement)
- Accumulated in `Treasury` as USDC under categories `FEE` and `BUYBACK` respectively. For buyback purposes both categories aggregate into the **buyback pool** balance returned by `Treasury.pendingBuybackBalance()`.

**Buyback execution — rolling geometric smoothing (decision 2):**

`BuybackExecutor` runs weekly. Each epoch spends `1/BUYBACK_EPOCH_COUNT == 1/12` of the Treasury's *current* buyback-pool balance:

```
spendThisEpoch = pendingBuybackBalance() × 1 / BUYBACK_EPOCH_COUNT      // i.e. /12
```

This is a **rolling geometric smoothing mechanism**, not a strict tranche queue. New inflows simply add to `pendingBuybackBalance` and are consumed at the same proportional rate. The model has no per-tranche state and no cleanup logic. Properties:

- **Half-life of any incoming dollar is approximately 8 epochs** (≈ 8 weeks) — `(11/12)^8 ≈ 0.50`.
- **After 12 epochs, approximately 65 % of any one inflow has been spent** — `1 - (11/12)^12 ≈ 0.65`.
- The remainder asymptotes toward zero as further epochs run.
- Long-run effect: each USDC of inflow is fully consumed in PSYH burns over many epochs, but no individual epoch is dominated by any single inflow.

L1.C ratified this model over an alternative tranche-queue accounting that would achieve strict 12-epoch linear consumption. Tranche queue was rejected as over-engineered for V0.3 — it would require per-tranche storage, active tranche cleanup, and gas growth proportional to inflow count. The simpler `currentBalance × 1/12` model achieves the product goal of "smooth buy pressure, no single-epoch market shock" with a single state variable.

Execution uses Uniswap V3 or CoW Protocol with TWAP; exact venue choice and slippage parameters are decided in L2-T5b BuybackExecutor (subtask T5.2). All PSYH bought back is **burned** (100% burn). Treasury does not retain buyback PSYH.

**Before Phase 3 (DEX listing):**
- Buyback pool accumulates USDC. No buybacks execute until `BuybackExecutor.activate()` is called by `DEFAULT_ADMIN_ROLE` after DEX liquidity is seeded. Phase 1 / Phase 2 thus build up a buyback war chest that is consumed once Phase 3 begins.

### 6.6 Token Utility Summary

Phase 1: reward only (accumulation, no transfer).  
Phase 2: transfer enabled (gift, send, but no on-chain liquidity).  
Phase 3: DEX liquidity seeded. Buybacks activate.  
Phase 4 (future): governance voting, staking, challenge mechanisms.

---

## 7. Phased Rollout

### 7.1 Phase 1 — Closed Accumulation (Month 0–3)

- Core protocol live: propositions, predictions, resolution, settlement, claim.
- `PsychohistoryToken` deployed. Minting active (via `RewardDistributor`). Transfers **disabled**.
- Users accumulate tokens as internal score balance. Front-end shows a "locked tokens" indicator.
- Treasury accumulates USDC from fees and Pool 2 allocations. Buybacks inactive.
- Team manually curates propositions. No sponsor self-service yet (optional).

### 7.2 Phase 2 — Transfer Unlock (Month 3–6)

- `TRANSFER_CONTROLLER_ROLE` flips `_transfersEnabled = true`.
- Tokens can be sent wallet-to-wallet. No DEX yet.
- OTC trades and informal markets may emerge. This is acceptable.
- Treasury continues accumulating USDC. No buybacks yet (no DEX liquidity to buy from).

### 7.3 Phase 3 — DEX Launch (Month 6+)

- Team creates initial Uniswap V3 liquidity pool: `PSYH / USDC`.
  - Liquidity seeded from: Liquidity Reserve bucket (10% of supply = 100M PSYH) + Treasury USDC balance.
  - Initial price set by team to be below expected market clearing price, allowing upward discovery.
- `BuybackExecutor.activate()` called.
- Weekly TWAP buyback + burn cycle begins.
- Sponsor self-service creation may be opened (subject to product readiness).

### 7.4 Phase 4 — Decentralization (Month 12+)

- Deferred. Covered by future TDDs. Includes:
  - Decentralized challenge mechanisms for proposition quality and fact submission
  - On-chain governance activation (DAO formation)
  - Deprecation of team-curated proposition creation in favor of permissionless creation with challenge periods
  - Oracle decentralization

---

## 8. Sponsor Mechanics

### 8.1 Lifecycle

Sponsors deposit USDC at any time between `openTimestamp` and `closeTimestamp`. Deposits are **additive** — a sponsor may call `addSponsorship()` multiple times, each call increases their `contributionAmount` cumulatively.

**Sponsor count cap (decision 4).** Each bounty enforces a hard limit of `MAX_SPONSORS_PER_BOUNTY = 100` distinct sponsor addresses. The 101st distinct address calling `addSponsorship()` reverts with `SponsorCapReached`. Existing sponsors may still increase their contribution after the cap is reached; only new sponsor enrollment is blocked. Rationale: V0.3 sorts sponsors on-chain at `closeTimestamp` (§8.2); a 100-element sort fits comfortably within block gas limits, while removing the need for an off-chain hint + verification protocol. If/when V0.3 launch traffic demonstrates demand for >100-sponsor bounties, V0.4 may upgrade to an off-chain sorted hint with on-chain pagination verification (analogous to §11 cutoff hint).

At `closeTimestamp`, sponsor contributions are frozen. `BountyManager.finalizeSponsorRanking(bountyId)` (callable by `PREDICTION_ENGINE_ROLE` or any caller after close) sorts the ≤ 100 sponsors descending by `contributionAmount`, assigns `rank` (0 = top), `tier`, and `accessUnlockTimestamp` per §8.2. Sort uses an in-memory pass plus `mstore` of sorted addresses; ties broken by earlier `submittedAt`, falling back to address numeric order.

### 8.2 Tiered Data Access — Service Tier, Not Information Tier

In V0.3 prediction data is on-chain transparent (see §1, §12.4). Tiered "access" therefore refers to **service-priority delivery of curated aggregate analytics**, not exclusive cryptographic access to information. The on-chain ranking + timestamps record an immutable commitment by the protocol team / data service to deliver per-tier service within the listed window.

After `closeTimestamp` but before `publicAccessTimestamp` (= bounty's `resolutionDeadline` for V0.3), the off-chain analytics service progressively delivers aggregate-statistics packages to sponsors:

- **Tier 1 (day 1):** top 1 sponsor by contribution
- **Tier 2 (day 2):** next 2 sponsors (ranks 2–3)
- **Tier 3 (day 3):** next 3 sponsors (ranks 4–6)
- **Tier N (day N):** next N sponsors
- **Final tier:** all remaining sponsors, no per-tier cap (day N+1 or earliest reached when all sponsors are seated)
- **Public:** everyone, at `publicAccessTimestamp`

**Rule:** Tier `k` has capacity for `k` sponsors. If fewer sponsors exist than capacity, the sequence collapses — sponsors fill tiers 1, 2, 3 in order until all are placed. Tier timing still uses day-indexing (tier `k` opens on day `k`). With the 100-sponsor cap, the deepest possible tier is `k = 13` (1 + 2 + ... + 13 = 91 < 100 ≤ 1 + 2 + ... + 14 = 105), so settlements complete within ~2 weeks of close in worst case.

**What sponsors actually receive (the "service" they pay for):**

- Pre-computed `wagerWeightedDistribution` ready in clean schema (JSON / CSV / Parquet) on standardized API endpoint
- `predictionCount`, `totalWager`, plus per-option / per-bucket sub-aggregates
- Optional: time-series aggregates by sub-window, percentile breakdowns, predictor cohort analysis (premium tiers)
- SLA on freshness ("within X minutes of close for Tier 1, etc.")
- Optional: human/AI analyst Q&A priority (highest tiers)

A sponsor who refuses the service can replicate the on-chain aggregation themselves (see §1 persona note: ~$5K–$50K engineering + maintenance). The service tier's value is convenience and SLA; the auction equilibrium price reflects this convenience premium.

**Implementation:** the on-chain `BountyManager` records sponsor ranking + tier timestamps immutably. The **off-chain service** is operated by the team in V0.3 (later may be opened to multiple service providers). On-chain accountability for service delivery is out of scope for this TDD.

### 8.3 Aggregated Data Content

The aggregate-statistics delivered by the service consists of (at minimum):

- `wagerWeightedDistribution`: for each option (discrete) or value histogram bucket (numerical), the sum of `effectiveWager` placed. This is the market-implied probability distribution weighted by skin-in-the-game.
- `predictionCount`: number of unique predictors.
- `totalWager`: sum of `effectiveWager` (or `rawWager`, both versions provided).
- `submittedAt` distribution: temporal breakdown of when wagers entered.

Individual predictor identities are visible on-chain (V0.3, decision 1) but the off-chain service does NOT highlight them; sponsors interested in pseudonym-level data can derive it from chain themselves.

Higher-tier service may include richer aggregates (cohort, percentile, time-series) as a product differentiator — these are not in the on-chain spec.

### 8.4 Sponsor Refund

Sponsors can reclaim deposits in exactly these cases:

| Trigger | `BountyState` precondition | Refund event `reasonCode` |
|---|---|---|
| Oracle calls `resolveAsInvalid` (proposition was malformed / unknowable) | `Invalidated` | `INVALIDATED` |
| Admin cancels before any prediction or sponsorship exists | `Cancelled` | `CANCELLED` |
| `totalPredictors == 0` at `resolutionDeadline` (no signal generated) | `Settled` (terminal, with refund predicate) | `NO_SIGNAL` |

Sponsors **cannot** reclaim when settlement proceeds normally with at least one predictor. The 40% of Pool 2 they pay in platform take (20% buyback + 10% team + 10% DAO) is the cost of the analytics service plus protocol overhead.

The `NO_SIGNAL` case is operationally distinct from `INVALIDATED`: the proposition was *resolvable* (oracle could have submitted an outcome) but no predictor staked, so no aggregate signal was produced. We treat it as refund-eligible because the sponsor's purpose (buying signal) was unmet. The `BountyState` does **not** need a new enum value for this; the refund predicate `(state == Invalidated) || (state == Cancelled) || (state == Settled && totalPredictors == 0)` is sufficient and the event `reasonCode` carries the semantic distinction.

**Sponsor refund supersedes V0.30 §5.9 ambiguity.** V0.30 §5.9 contained contradictory language — the lead paragraph said sponsors get no refund on zero-predictor bounties, then the footnote contradicted by enabling refund. V0.31 unifies on **refund** per the table above; this is also consistent with V0.30 §8.4. See §5.9 for the corresponding edge-case settlement flow.

---

## 9. Data Structures

Full Solidity definitions. These should be in `src/libraries/PsychohistoryTypes.sol` or equivalent shared location.

```solidity
enum BountyState {
    Open,              // accepting predictions and sponsorships
    Closed,            // prediction window ended; tiered data release in progress;
                       //   reached via explicit BountyManager.closeBounty() at closeTimestamp
    Resolved,          // oracle has submitted outcome
    Settled,           // settlement complete; funds distributed (may carry a refund predicate
                       //   if totalPredictors == 0; see §8.4)
    Invalidated,       // oracle voided the bounty; sponsors + predictors refundable
    Cancelled          // admin cancelled before predictions / sponsorships
}

enum PropositionType {
    Discrete,          // N options, 2 ≤ N ≤ 5
    Numerical          // continuous value
}

enum PrivacyMode {
    Transparent,           // V0.3 default and only allowed value at launch.
    OracleEncrypted,       // V0.4 candidate: ciphertext encrypted to oracle pubkey
    ThresholdEncrypted     // V0.4 candidate: ciphertext encrypted to threshold operator group
}

enum RefundReasonCode {
    INVALIDATED,           // BountyState.Invalidated path
    CANCELLED,             // BountyState.Cancelled path
    NO_SIGNAL              // BountyState.Settled with totalPredictors == 0
}

struct Bounty {
    uint256 bountyId;
    address creator;                    // team address in Phase 1
    string metadataURI;                 // IPFS CID: question text, option labels, units
    PropositionType propositionType;
    uint8 optionsCount;                 // Discrete: 2–5. Numerical: 0.

    uint64 openTimestamp;
    uint64 closeTimestamp;
    uint64 resolutionDeadline;          // hard upper bound for oracle to call resolve(); also used as publicAccessTimestamp
    uint64 resolvedAt;                  // 0 until oracle calls resolve(); set to block.timestamp at that call
    BountyState state;

    PrivacyMode privacyMode;            // V0.3: enforced == Transparent at createBounty(); V0.4 may relax

    uint256 resolvedValue;              // Discrete: winning option index. Numerical: scaled integer.
    uint8 resolvedDecimals;

    uint256 totalPredictors;
    uint256 totalRawWagerAmount;        // Σ rawWager, audit only
    uint256 totalEffectiveWagerAmount;  // Σ effectiveWager = Σ rawWager × 0.99 (Pool 1 effective total)
    uint256 totalFeeCollected;          // Σ feeAmount, transferred to Treasury at submission time (§5.7)
    uint256 totalSponsorAmount;         // Pool 2 total
    uint256 sponsorCount;               // monotonically increasing; capped at MAX_SPONSORS_PER_BOUNTY (§8.1)
    uint256 winnersCount;               // |T|, set during settlement Pass 2

    uint256[20] __reservedForV04;       // reserved storage for V0.4 privacy / encryption fields
}

struct Prediction {
    address predictor;
    uint256 bountyId;
    uint256 submittedAt;                // for deterministic tiebreaking if needed

    // Wager fields (§5.7 net-of-fee model)
    uint256 rawWager;                   // amount user transferred
    uint256 effectiveWager;             // == rawWager × 0.99 (rounded down); used in all distribution math
    uint256 feeAmount;                  // == rawWager - effectiveWager; transferred to Treasury at submission
    // Invariant: rawWager == effectiveWager + feeAmount, exactly.

    // Discrete fields (unused for Numerical)
    // confidenceBpsArray stored in separate mapping for gas and variable-length safety; see below

    // Numerical fields (unused for Discrete)
    uint256 predictedValue;
    uint8 predictedDecimals;

    // V0.3 privacy hook (decision 1, hook A): always empty in V0.3
    bytes encryptedPayload;             // V0.3 require length == 0; V0.4 stores ciphertext

    // Computed during settlement
    uint256 score;                      // WAD-scaled, high-is-better (§3.3); filled in Pass 1
    bool inTopGroup;                    // set in Pass 2 after cutoff hint verified
    bool processed;                     // settlement Pass 4 visited this prediction

    // Claimable amounts (filled by settle())
    uint256 usdcPayout;                 // Pool 1 payout + Pool 2 slice (consolation or victory)
    uint256 tokenReward;                // Pool 3 $PSYH owed to predictor on claim (mint at claim time)
    bool claimed;

    uint256[10] __reservedForV04;       // reserved storage for V0.4 encryption / commitment fields
}

struct Sponsorship {
    address sponsor;
    uint256 bountyId;
    uint256 contributionAmount;         // cumulative; increments on addSponsorship()
    uint256 rank;                       // set at finalizeSponsorRanking; 0 = highest
    uint64 tier;                        // set at finalizeSponsorRanking; 1 = earliest access
    uint64 accessUnlockTimestamp;       // set at finalizeSponsorRanking
    bool refunded;                      // true once refund claimed
}

struct SettlementState {
    bool resolved;                      // mirrors Bounty.state == Resolved
    bool isInvalidated;
    uint256 resolvedValue;
    uint8 resolvedDecimals;

    // Pass machinery (§11)
    uint8 currentPass;                  // 1, 2, 3, or 4
    uint8 passCompletedFlags;           // bit i set when pass i complete (i ∈ {1,2,3,4})
    uint256 settledUpTo;                // pagination cursor; reset to 0 when advancing pass

    // Cutoff hint (Pass 2, §11)
    bool cutoffHintSubmitted;
    uint256 cutoffHintScore;
    uint256 cutoffStrictlyAboveCount;   // running count for hint verification
    uint256 cutoffAtCutoffCount;        // running count for hint verification

    uint256 topGroupCount;              // |T|, frozen after Pass 2 verification
    uint256 topGroupCutoffScore;        // == cutoffHintScore once verified
    bool fullySettled;

    // Pool 1 / Pool 2 accumulators (Pass 3)
    uint256 p1TopPrincipal;             // Σ effectiveWager for i ∈ T
    uint256 p1RemainderAmount;          // = totalEffectiveWagerAmount - p1TopPrincipal
    uint256 p1SumTopScoreEffWager;      // Σ(score × effectiveWager / WAD) for T
    uint256 p2SumTopScoreEffWager;      // Σ(score × effectiveWager / WAD) for T (used in Pool 2 Slice B)
    uint256 p2SumBottomScoreEffWager;   // Σ(score × effectiveWager / WAD) for B (used in Pool 2 Slice A)
    uint256 sumPool1PayoutTop;          // Σ pool1Payout_i for T (used in Pool 3 Slice A)

    // Pool 3 (Pass 4 finalization, §10.3)
    uint256 reservedTokenAllocation;    // returned by RewardDistributor.reserveRewards()
    uint256 effectiveKWad;              // returned by RewardDistributor.reserveRewards()

    uint256[20] __reservedForV04;
}
```

**Separate mappings (for gas and variable-length array safety):**
```solidity
mapping(uint256 => mapping(address => uint256[])) public confidenceArrays;
// bountyId → predictor → confidenceBpsArray (Discrete only). For Numerical, predictedValue
// lives directly on Prediction struct.

mapping(uint256 => address[]) public predictorList;
// bountyId → ordered list of predictors (used to iterate during settlement)

mapping(uint256 => address[]) public sponsorList;
// bountyId → ordered list of sponsors (used in finalizeSponsorRanking sort)
```

**Constants (`src/libraries/Constants.sol`):**
```solidity
// Token / unit fundamentals
uint256 constant WAD          = 1e18;
uint256 constant WAD_SQUARED  = 1e36;
uint256 constant BPS          = 10_000;
uint256 constant MIN_BRIER    = 1e12;
uint256 constant MAX_BRIER_SCORE = 2e18;

// USDC scale
uint256 constant USDC_DECIMALS_DIVISOR = 1e6;       // converts USDC raw → 1.0 USDC for kWad multiplication

// Wager bounds
uint256 constant MIN_WAGER    = 1e6;                // 1 USDC

// Discrete bounds
uint8   constant MAX_OPTIONS  = 5;
uint8   constant MIN_OPTIONS  = 2;

// Pool math fractions (basis points)
uint256 constant PLATFORM_FEE_BPS    = 100;         // 1%
uint256 constant P2_CONSOLATION_BPS  = 3000;        // 30%
uint256 constant P2_VICTORY_BPS      = 3000;        // 30%
uint256 constant P2_BUYBACK_BPS      = 2000;        // 20%
uint256 constant P2_TEAM_BPS         = 1000;        // 10%
uint256 constant P2_DAO_BPS          = 1000;        // 10%
uint256 constant TOP_GROUP_BPS       = 5000;        // 50%

// Sponsor cap (decision 4)
uint256 constant MAX_SPONSORS_PER_BOUNTY = 100;

// Buyback (decision 2: rolling geometric, NOT linear-12-week)
uint256 constant BUYBACK_EPOCH_COUNT    = 12;       // each epoch spends balance × 1/12
uint256 constant BUYBACK_EPOCH_DURATION = 7 days;

// K(t) schedule, WAD-scaled (§6.4)
uint256 constant K_WAD_PHASE1 = 10 * 1e18;          // months 1–3
uint256 constant K_WAD_PHASE2 =  5 * 1e18;          // months 4–6
uint256 constant K_WAD_PHASE3 =  2 * 1e18;          // months 7–12
uint256 constant K_WAD_PHASE4 =  1 * 1e18;          // months 13–24
uint256 constant K_WAD_PHASE5 = 0.5 * 1e18 / 1;     // months 25–36 (encoded as 5e17 in actual code)
uint256 constant K_WAD_PHASE6 = 0.25 * 1e18 / 1;    // months 37–48 (encoded as 2.5e17)
uint256 constant K_WAD_PHASE7 = 0;                  // months 49+

// Mining bucket bound (decision 5: cap, not pre-mint)
uint256 constant MINING_CAP = 400_000_000 * 1e18;   // 40% of total supply

// Treasury categories (keccak256 hashes, see §10.4)
bytes32 constant CAT_FEE             = keccak256("FEE");
bytes32 constant CAT_BUYBACK         = keccak256("BUYBACK");
bytes32 constant CAT_DAO             = keccak256("DAO");
bytes32 constant CAT_DUST            = keccak256("DUST");
bytes32 constant CAT_P2_UNALLOCATED  = keccak256("P2_UNALLOCATED");   // V0.31 NEW
```

> **Note on K_WAD_PHASE5/6 encoding.** Solidity does not allow non-integer literal expressions like `0.5 * 1e18` directly. Implementations should declare `K_WAD_PHASE5 = 5e17` and `K_WAD_PHASE6 = 25e16` literally; the decimals comments above are documentation. The library's `currentK()` selects from these constants by month.

> **Storage layout note.** All upgradeable contracts must reserve `uint256[50] private __gap;` at end of storage per §2; combined with the explicit `__reservedForV04` slots in `Bounty` / `Prediction` / `SettlementState`, V0.4 can extend storage without disturbing layout. See §12.7.

---

## 10. Interface Specifications

### 10.1 IBountyManager

```solidity
interface IBountyManager {
    // ─── Events ───────────────────────────────────────────────────────────────

    event BountyCreated(
        uint256 indexed bountyId,
        address indexed creator,
        PropositionType propositionType,
        uint8 optionsCount,
        PrivacyMode privacyMode
    );
    event SponsorshipDeposited(uint256 indexed bountyId, address indexed sponsor, uint256 totalContribution);
    event SponsorCapReached(uint256 indexed bountyId, uint256 sponsorCount);  // emitted on the call that fills the 100th slot
    event BountyStateChanged(uint256 indexed bountyId, BountyState oldState, BountyState newState);
    event SponsorTierAssigned(uint256 indexed bountyId, address indexed sponsor, uint256 rank, uint64 tier, uint64 accessUnlock);
    event SponsorRefundClaimed(
        uint256 indexed bountyId,
        address indexed sponsor,
        uint256 amount,
        RefundReasonCode reasonCode
    );
    event BountyResolvedAt(uint256 indexed bountyId, uint256 resolvedAt);  // mirror to PE event for off-chain indexers

    // ─── Public lifecycle (sponsor / admin) ───────────────────────────────────

    /// @notice Create a new bounty. V0.3 enforces privacyMode == PrivacyMode.Transparent.
    /// V0.4 may relax this to allow OracleEncrypted / ThresholdEncrypted modes.
    function createBounty(
        PropositionType propositionType,
        uint8 optionsCount,
        string calldata metadataURI,
        uint64 openTimestamp,
        uint64 closeTimestamp,
        uint64 resolutionDeadline,
        PrivacyMode privacyMode  // V0.3 require == PrivacyMode.Transparent
    ) external returns (uint256 bountyId);

    /// @notice Add to sponsor's contribution. Reverts with SponsorCapReached when a NEW
    /// sponsor address attempts to enroll past MAX_SPONSORS_PER_BOUNTY.
    /// Existing sponsors may continue to top up after the cap is reached.
    function addSponsorship(uint256 bountyId, uint256 amount) external;

    /// @notice Refund predicate is (state == Invalidated) || (state == Cancelled) ||
    /// (state == Settled && totalPredictors == 0). Emits SponsorRefundClaimed with reason code.
    function claimSponsorshipRefund(uint256 bountyId) external;

    /// @notice Admin-only. Only valid when no predictions AND no sponsorships exist on the bounty.
    /// Transitions Open → Cancelled.
    function cancelBounty(uint256 bountyId) external;

    // ─── Sponsor ranking ──────────────────────────────────────────────────────

    /// @notice Sort sponsors descending by contributionAmount and assign rank, tier,
    /// accessUnlockTimestamp. Callable by anyone after closeTimestamp; idempotent thereafter.
    /// Internally bounded by MAX_SPONSORS_PER_BOUNTY = 100, so on-chain sort is gas-safe.
    function finalizeSponsorRanking(uint256 bountyId) external;

    // ─── PE-only role-gated mutators (V0.31 NEW) ──────────────────────────────
    //
    // These functions are callable ONLY by PREDICTION_ENGINE_ROLE (granted to the PE proxy
    // at deployment, see §4.2 / §13). They allow PE to update Bounty struct fields without
    // exposing them to public mutation.

    /// @notice Increment Bounty.totalPredictors / totalRawWagerAmount / totalEffectiveWagerAmount /
    /// totalFeeCollected. Called from PredictionEngine.submitPrediction() after successful
    /// USDC transfer. MUST be atomic with the predictor list append (predictorList[bountyId].push).
    function recordPrediction(
        uint256 bountyId,
        address predictor,
        uint256 rawWager,
        uint256 effectiveWager,
        uint256 feeAmount
    ) external;

    /// @notice Transition Open → Closed. Callable by PE (typically when first action arrives
    /// after closeTimestamp) or by anyone at/after closeTimestamp via an explicit closeBounty()
    /// passthrough. Idempotent.
    function closeBounty(uint256 bountyId) external;

    /// @notice Transition Closed → Resolved. Stores resolvedValue/resolvedDecimals/resolvedAt.
    /// Caller is PredictionEngine.resolve().
    function markResolved(
        uint256 bountyId,
        uint256 resolvedValue,
        uint8 resolvedDecimals
    ) external;

    /// @notice Transition any pre-Settled state → Invalidated. Caller is PredictionEngine.resolveAsInvalid().
    function markInvalidated(uint256 bountyId) external;

    /// @notice Transition Resolved → Settled when settlement fully completes.
    /// May also be called from the no-signal refund path with totalPredictors == 0.
    /// Caller is PredictionEngine.settle() final batch (or PE on no-signal exit).
    function markSettled(uint256 bountyId, uint256 winnersCount) external;

    // ─── Views ────────────────────────────────────────────────────────────────

    function getBounty(uint256 bountyId) external view returns (Bounty memory);
    function getSponsorship(uint256 bountyId, address sponsor) external view returns (Sponsorship memory);
    function getAllSponsors(uint256 bountyId) external view returns (address[] memory);
    function isRefundable(uint256 bountyId) external view returns (bool, RefundReasonCode);
}
```

> **Note on V0.30 → V0.31 interface delta.** V0.30 listed only `createBounty` / `addSponsorship` / `finalizeSponsorRanking` / `claimSponsorshipRefund` / `cancelBounty` plus views. The PE-mutator block (`recordPrediction` / `closeBounty` / `markResolved` / `markInvalidated` / `markSettled`) is new in V0.31 — V0.30 implicitly assumed these but never specified them, leaving the BountyManager / PredictionEngine boundary undefined. T1.3 must add the corresponding `PREDICTION_ENGINE_ROLE` modifier on each.

### 10.2 IPredictionEngine

```solidity
interface IPredictionEngine {
    // ─── Events ───────────────────────────────────────────────────────────────

    event PredictionSubmitted(
        uint256 indexed bountyId,
        address indexed predictor,
        uint256 rawWager,
        uint256 effectiveWager,
        uint256 feeAmount,
        bool encrypted             // == (encryptedPayload.length > 0); always false in V0.3
    );
    event PredictionResolved(uint256 indexed bountyId, uint256 resolvedValue, uint256 resolvedAt);
    event BountyInvalidated(uint256 indexed bountyId);
    event CutoffHintSubmitted(uint256 indexed bountyId, uint256 cutoffScore, address submitter);
    event CutoffHintVerified(uint256 indexed bountyId, uint256 cutoffScore, uint256 topGroupCount);
    event SettlementProgressed(uint256 indexed bountyId, uint8 currentPass, uint256 settledUpTo, uint256 totalPredictors);
    event SettlementComplete(uint256 indexed bountyId, uint256 topGroupCount);
    event NoSignalSettled(uint256 indexed bountyId);  // emitted when settle() reaches final state with totalPredictors == 0
    event PayoutClaimed(uint256 indexed bountyId, address indexed predictor, uint256 usdcAmount, uint256 tokenAmount);

    // ─── Submission ───────────────────────────────────────────────────────────

    /// @notice Submit a prediction with USDC wager. Fee deducted at submission (§5.7).
    /// V0.3: encryptedPayload MUST be empty (require length == 0). V0.4 will relax based
    /// on Bounty.privacyMode.
    function submitPrediction(
        uint256 bountyId,
        uint256[] calldata confidenceBpsArray,  // Discrete: length == optionsCount, sum == 10000. Numerical: empty.
        uint256 predictedValue,                 // Numerical only
        uint8 predictedDecimals,                // Numerical only
        uint256 wagerAmount,                    // rawWager USDC; min 1e6
        bytes calldata encryptedPayload         // V0.3 MUST be empty; V0.4 carries ciphertext
    ) external;

    // ─── Resolution (oracle) ──────────────────────────────────────────────────

    /// @notice Only ORACLE_ROLE. Calls BountyManager.markResolved internally.
    /// Transitions Closed → Resolved.
    function resolve(
        uint256 bountyId,
        uint256 resolvedValue,
        uint8 resolvedDecimals
    ) external;

    /// @notice Only ORACLE_ROLE. Calls BountyManager.markInvalidated. Allows total refund
    /// of predictor wagers and sponsor deposits. May be called pre- or post-close.
    function resolveAsInvalid(uint256 bountyId) external;

    // ─── Settlement (paginated, see §11) ──────────────────────────────────────

    /// @notice Submit the off-chain-computed cutoff score for top-50% determination.
    /// Permissionless. Must be called once after Pass 1 completes and before Pass 2 verification
    /// can proceed. Idempotent only if the same score was previously accepted; otherwise reverts
    /// (re-submission requires a state-machine reset path not exposed in V0.3).
    function submitCutoffHint(uint256 bountyId, uint256 cutoffScore) external;

    /// @notice Permissionless. Multi-pass paginated settlement; see §11.
    /// `currentPass` and `settledUpTo` in SettlementState track progress across calls.
    function settle(uint256 bountyId, uint256 startIndex, uint256 endIndex) external;

    /// @notice Pull pattern. Transfers usdcPayout to msg.sender (the predictor).
    /// Internally calls RewardDistributor.claimTokens(bountyId, msg.sender) so the predictor
    /// receives both USDC and PSYH in a single transaction. Reverts on double-claim.
    /// Predictors who only want PSYH (not the USDC) MAY call RewardDistributor.claimTokens()
    /// directly; the USDC payout still requires this function.
    function claim(uint256 bountyId) external;

    // ─── Views ────────────────────────────────────────────────────────────────

    function getPrediction(uint256 bountyId, address predictor) external view returns (Prediction memory);
    function getPredictorCount(uint256 bountyId) external view returns (uint256);
    function getSettlementState(uint256 bountyId) external view returns (SettlementState memory);
}
```

### 10.3 IRewardDistributor

V0.31 rewrites this interface around a **reserve / assign / finalize / claim** lifecycle. The cap is consumed at reservation time (when settlement begins), guaranteeing each predictor in a given bounty sees the **same** `effectiveKWad`. Late claims do not contend with newer bounties.

```solidity
interface IRewardDistributor {
    // ─── Events ───────────────────────────────────────────────────────────────

    event RewardsReserved(
        uint256 indexed bountyId,
        uint256 volumeUsdcRaw,         // P1_rawTotal + P2_total fed in
        uint256 reservedAmount,        // PSYH (raw, 18-decimal) committed against monthly cap
        uint256 effectiveKWad          // K_wad value clamped to remaining cap
    );
    event RewardsAssigned(uint256 indexed bountyId, address indexed predictor, uint256 amount);
    event RewardsFinalized(uint256 indexed bountyId, uint256 totalAssigned, uint256 unassignedReturned);
    event TokensClaimed(uint256 indexed bountyId, address indexed claimant, uint256 minted);
    event KCoefficientObserved(uint256 month, uint256 kWad);  // emitted on bounds checks for monitoring

    // ─── Reserve / Assign / Finalize (caller: PredictionEngine settlement) ───

    /// @notice Called once per bounty at settlement Pass 3 finalization, BEFORE per-predictor
    /// rewards are computed. Locks emission against the monthly cap; returns the actual
    /// (possibly clamped) values to use when computing per-predictor amounts.
    /// @param volumeUsdcRaw  P1_rawTotal + P2_total, in USDC raw (6-decimal) units.
    /// @return reservedAmount  PSYH (18-decimal) reserved for this bounty.
    /// @return effectiveKWad   The K_wad effectively applied (may be < currentK() if cap was tight).
    function reserveRewards(uint256 bountyId, uint256 volumeUsdcRaw)
        external
        returns (uint256 reservedAmount, uint256 effectiveKWad);

    /// @notice Paginated assignment of reserved rewards to specific predictors. Called from
    /// PredictionEngine settlement Pass 4 (or finalization). Σ amounts may NOT exceed
    /// reservedAmount (revert otherwise). Multiple calls accumulate.
    function assignRewards(
        uint256 bountyId,
        address[] calldata predictors,
        uint256[] calldata amounts
    ) external;

    /// @notice Marks the bounty's reward assignment as complete. Any unassigned remainder
    /// (reservedAmount - Σ assigned) is RELEASED back to the unallocated mining pool, restoring
    /// the monthly cap headroom for future bounties. Idempotent after first call.
    function finalizeRewards(uint256 bountyId) external;

    // ─── Claim (caller: predictor directly OR PredictionEngine.claim) ────────

    /// @notice Mint assigned PSYH to predictor (one-shot). Reverts if already claimed for
    /// this (bountyId, predictor) pair. No cap check — cap was consumed at reserve time.
    /// Calls PsychohistoryToken.mint(predictor, amount). Returns minted amount.
    function claimTokens(uint256 bountyId, address predictor) external returns (uint256 minted);

    // ─── Views ────────────────────────────────────────────────────────────────

    /// @notice K_wad valid for the current block.timestamp's month bucket. Pure schedule lookup.
    function currentK() external view returns (uint256 kWad);

    /// @notice The PSYH amount assigned to (bountyId, predictor) but not yet claimed.
    function pendingTokensOf(uint256 bountyId, address predictor) external view returns (uint256);

    /// @notice Reservation snapshot for a bounty (reservedAmount, effectiveKWad, totalAssigned, finalized).
    function reservationOf(uint256 bountyId)
        external
        view
        returns (uint256 reservedAmount, uint256 effectiveKWad, uint256 totalAssigned, bool finalized);

    /// @notice Cap headroom remaining for the current month, AFTER all current reservations.
    function monthlyCapRemaining() external view returns (uint256);
}
```

> **Note on V0.30 → V0.31 delta.** V0.30 had a single `distributeRewards` taking `(bountyId, predictors[], amounts[])` plus `claimTokens`. The cap check happened lazily inside `claimTokens`. That model created a race where late claimers could be reduced because newer bounties consumed the month's cap first. V0.31's reserve-at-start model fixes this. Migration from V0.30 implementations is non-trivial and is captured in T0 spec lock and T3.2 redesign.

### 10.4 ITreasury

```solidity
interface ITreasury {
    // ─── Events ───────────────────────────────────────────────────────────────

    event FundsReceived(
        uint256 indexed bountyId,
        address source,
        uint256 amount,
        bytes32 indexed category    // CAT_FEE | CAT_BUYBACK | CAT_DAO | CAT_DUST | CAT_P2_UNALLOCATED
    );
    event BuybackPulled(uint256 amount);   // amount pulled by BuybackExecutor for this epoch
    event BuybackExecuted(uint256 amountIn, uint256 tokensBurned);
    event DAOFundWithdrawal(address to, uint256 amount, bytes32 indexed category, string reason);

    // ─── Inflows (caller: PE settlement, PE submission for FEE) ───────────────

    /// @notice Receive funds tagged with a category. Caller MUST hold PREDICTION_ENGINE_ROLE.
    /// Increments per-category running balance. Categories include the V0.31 NEW
    /// CAT_P2_UNALLOCATED for empty-bottom-group Slice A redirects (§5.5).
    function receivePoolFunds(uint256 bountyId, uint256 amount, bytes32 category) external;

    // ─── Buyback queue (caller: BuybackExecutor) ──────────────────────────────

    /// @notice Total balance available for buyback. Combines CAT_FEE and CAT_BUYBACK
    /// per-category accounting; CAT_DAO, CAT_P2_UNALLOCATED, and CAT_DUST are excluded.
    function pendingBuybackBalance() external view returns (uint256);

    /// @notice Pull `pendingBuybackBalance × 1/BUYBACK_EPOCH_COUNT` USDC for this epoch's
    /// buyback. Caller MUST hold TREASURY_EXECUTOR_ROLE (= BuybackExecutor proxy).
    /// Decrements both CAT_FEE and CAT_BUYBACK proportionally. Returns the amount pulled.
    /// Reverts if called more than once per epoch (epoch tracked here, not by caller).
    function pullBuybackForEpoch(uint256 epochIndex) external returns (uint256 amountPulled);

    // ─── DAO withdrawals (caller: DEFAULT_ADMIN_ROLE / future DAO governor) ──

    /// @notice Withdraw from CAT_DAO or CAT_P2_UNALLOCATED sub-balances. Cannot withdraw
    /// from CAT_FEE / CAT_BUYBACK (those flow to buyback) or CAT_DUST (negligible).
    function daoWithdraw(address to, uint256 amount, bytes32 category, string calldata reason) external;

    // ─── Views ────────────────────────────────────────────────────────────────

    function categoryBalance(bytes32 category) external view returns (uint256);
    function totalBalance() external view returns (uint256);
}
```

> **V0.31 delta.** New `CAT_P2_UNALLOCATED` category for Slice A fallback (decision 3); per-category balance accounting (was implicitly sum-only); `pullBuybackForEpoch` replaces `scheduleBuyback` to make the buyback queue's epoch tracking authoritative on the Treasury side rather than the BuybackExecutor side; `daoWithdraw` now takes a `category` parameter so admin can distinguish DAO-bucket vs. P2_UNALLOCATED-bucket withdrawals.

### 10.5 IBuybackExecutor

```solidity
interface IBuybackExecutor {
    event Activated(uint256 firstEpochTimestamp);
    event BuybackEpochExecuted(
        uint256 indexed epoch,
        uint256 usdcSpent,        // == Treasury.pullBuybackForEpoch return value
        uint256 psyhBought,
        uint256 psyhBurned        // == psyhBought (100% burn invariant)
    );

    /// @notice DEFAULT_ADMIN_ROLE. Starts weekly epochs. Called once after Phase 3 DEX
    /// liquidity has been seeded. Idempotent: subsequent calls revert.
    function activate() external;

    /// @notice Permissionless. Spends pendingBuybackBalance × 1/12 (§6.5 rolling geometric model).
    /// Executes via Uniswap V3 / CoW Protocol with TWAP per L2-T5b decisions. Burns 100% PSYH
    /// received. Reverts if called more than once per BUYBACK_EPOCH_DURATION (= 7 days).
    function executeEpoch() external;

    function currentEpoch() external view returns (uint256);
    function lastExecutedEpoch() external view returns (uint256);
    function isActivated() external view returns (bool);
}
```

> **V0.31 delta.** Behavioural unchanged from V0.30; spec wording adjusted to match the §6.5 geometric model (was "linearly over 12 weeks" — incorrect — now "× 1/12 per epoch"). DEX venue (Uniswap V3 vs. CoW vs. both) is decided in L2-T5b BuybackExecutor task.

### 10.6 IPsychohistoryToken

```solidity
interface IPsychohistoryToken is IERC20, IERC20Permit, IVotes {
    event TransfersEnabled();

    function mint(address to, uint256 amount) external;
    // Only MINTER_ROLE (= RewardDistributor).

    function burn(uint256 amount) external;
    // Standard ERC20 burn.

    function enableTransfers() external;
    // TRANSFER_CONTROLLER_ROLE only. One-way flag; once enabled, cannot disable.

    function transfersEnabled() external view returns (bool);
}
```

---

## 11. Settlement Algorithm (Paginated)

The most complex function in the protocol. Specification is normative — implementation must match exactly.

**Input:** `bountyId`, `startIndex`, `endIndex`. Each `settle()` call processes the predictor range `[startIndex, endIndex)` within the *current* pass.

**Precondition:** `SettlementState.resolved == true` and `SettlementState.isInvalidated == false`. Bounty state must be `Resolved` (not yet `Settled`).

**Zero-predictor short-circuit.** If `Bounty.totalPredictors == 0`, the very first `settle()` call after `resolve()` skips all 4 passes, transitions Bounty to `Settled`, emits `NoSignalSettled`, and returns. Sponsors can then call `claimSponsorshipRefund()` per §8.4.

**High-level: a four-pass algorithm split across multiple `settle()` calls.** Because Solidity has no cheap sort and unbounded loops are gas-hazardous, settlement is paginated within each pass and gated between passes by `currentPass` / `passCompletedFlags`.

### State machinery (mirrors §9 SettlementState)

```
currentPass        // 1, 2, 3, or 4 — which pass settle() is currently advancing
settledUpTo        // pagination cursor within currentPass; reset to 0 when pass advances
passCompletedFlags // bit i set when pass i complete
```

Pass advancement: when `settledUpTo == totalPredictors` for a given pass, the contract:
1. Sets the corresponding bit in `passCompletedFlags`.
2. Increments `currentPass` to the next pass.
3. Resets `settledUpTo` to 0.
4. Performs any once-per-pass aggregate finalization (e.g., Pass 2 hint verification, Pass 3 reservation).

The next `settle()` call enters the next pass.

### Pass 1 — Score Computation

For each prediction in `[startIndex, endIndex)` of `predictorList[bountyId]`:

1. Compute `score`:
   - Discrete: `score = (2 × WAD) - computeCategoricalBrierScore(confidenceWadArray, resolvedOptionIndex)`
   - Numerical: `score = WAD_SQUARED / max(computeAbsoluteError(predictedWad, resolvedWad), MIN_BRIER)`
   Both yield high-is-better scores in `[0, 2 × WAD]` (Discrete) or `[0, WAD_SQUARED / MIN_BRIER]` (Numerical, much larger range but same direction).
2. Store `score` in the `Prediction` struct via `_getPrediction(bountyId, predictor)` indirection (decision 1, hook D).
3. Increment `settledUpTo`.

When `settledUpTo == totalPredictors`, set `passCompletedFlags |= 0b0001`, advance to Pass 2, reset cursor.

### Pass 2 — Cutoff Hint Submission and Verification

This pass uses a **permissionless off-chain hint + on-chain paginated verification** model (decision 4 of L1.B; cutoff hint trust is fixed to permissionless in V0.31, no oracle gate, no bond).

**Step 2A — Hint submission (anyone, called once between Pass 1 completion and Pass 2 verification start).** `submitCutoffHint(bountyId, cutoffScore)` records the proposed cutoff in `SettlementState.cutoffHintScore` and sets `cutoffHintSubmitted = true`. If a hint already exists for this bounty, revert (V0.3 does not support hint revision; if a hint fails verification, the bounty enters a stuck state requiring oracle invalidation — acceptable for V0.3, may be improved in V0.4).

**Step 2B — Paginated verification.** For each prediction in `[startIndex, endIndex)`:

1. Read the prediction's `score` from storage (set in Pass 1).
2. If `score > cutoffHintScore`: increment `cutoffStrictlyAboveCount`.
3. If `score == cutoffHintScore`: increment `cutoffAtCutoffCount`.
4. If `score >= cutoffHintScore`: also set `prediction.inTopGroup = true`.
5. Increment `settledUpTo`.

When `settledUpTo == totalPredictors`, perform the verification check:

```
n = totalPredictors
targetCount = ceil(n / 2) = (n + 1) / 2

// Count of predictions in the top group under the proposed hint
topCount = cutoffStrictlyAboveCount + cutoffAtCutoffCount

require(cutoffStrictlyAboveCount <= targetCount, "hint too low");
require(topCount >= targetCount, "hint too high");
```

If both pass: set `topGroupCutoffScore = cutoffHintScore`, `topGroupCount = topCount`, set `passCompletedFlags |= 0b0010`, emit `CutoffHintVerified(bountyId, cutoffScore, topCount)`, advance to Pass 3.

If either fails: revert (the hint was invalid). The transaction's effects (`inTopGroup` flags set during this pagination) must be in-memory or rolled back by revert; `settledUpTo` is also rolled back. The submitter must call `submitCutoffHint` again with a corrected score. (V0.3 caveat: if no correct hint is ever submitted, bounty stays stuck. V0.4 may add a hint-replace path.)

### Pass 3 — Accumulator Pass + Reservation

For each prediction in `[startIndex, endIndex)`:

1. If `inTopGroup`:
   - `p1TopPrincipal += effectiveWager`
   - `p1SumTopScoreEffWager += (score × effectiveWager) / WAD`
   - `p2SumTopScoreEffWager += (score × effectiveWager) / WAD`
2. Else (in bottom group):
   - `p2SumBottomScoreEffWager += (score × effectiveWager) / WAD`
3. Increment `settledUpTo`.

When `settledUpTo == totalPredictors`, perform once-per-bounty finalization:

**Compute global amounts (using §9 Bounty / §5.7 fields):**

```
p1RawTotal       = bounty.totalRawWagerAmount
p1EffectiveTotal = bounty.totalEffectiveWagerAmount
p1Remainder      = p1EffectiveTotal - p1TopPrincipal       // ≥ 0 by construction (§5.4)

p2Total          = bounty.totalSponsorAmount
p2Consolation    = mulDiv(p2Total, P2_CONSOLATION_BPS, BPS)
p2Victory        = mulDiv(p2Total, P2_VICTORY_BPS, BPS)
p2Buyback        = mulDiv(p2Total, P2_BUYBACK_BPS, BPS)
p2Team           = mulDiv(p2Total, P2_TEAM_BPS, BPS)
p2Dao            = p2Total - p2Consolation - p2Victory - p2Buyback - p2Team   // exact remainder, no rounding loss
```

**Reserve Pool 3 allocation against monthly cap:**

```
(reservedTokenAllocation, effectiveKWad) =
    RewardDistributor.reserveRewards(bountyId, p1RawTotal + p2Total)
```

This consumes the month's cap headroom *now*, before any per-predictor amounts are computed. See §6.4 / §10.3 reserve-at-start model.

Set `passCompletedFlags |= 0b0100`, advance to Pass 4.

### Pass 4 — Per-Predictor Payout Assignment

For each prediction in `[startIndex, endIndex)`:

If `inTopGroup`:
```
remainderShare = mulDiv(p1Remainder,
                         score × effectiveWager,
                         p1SumTopScoreEffWager × WAD)         // score-0 fallback if denominator == 0

victoryBonus   = mulDiv(p2Victory,
                         score × effectiveWager,
                         p2SumTopScoreEffWager × WAD)         // score-0 fallback if denominator == 0

pool1Payout_i  = effectiveWager + remainderShare              // §5.4

usdcPayout     = pool1Payout_i + victoryBonus
```

Else:
```
consolation    = mulDiv(p2Consolation,
                         score × effectiveWager,
                         p2SumBottomScoreEffWager × WAD)      // score-0 fallback as in §5.5

usdcPayout     = consolation
pool1Payout_i  = 0
```

Accumulate `sumPool1PayoutTop += pool1Payout_i` (only for top group). Store `prediction.usdcPayout = usdcPayout`. Set `prediction.processed = true`. Increment `settledUpTo`.

### Final Finalization (triggered on the call that completes Pass 4)

When `settledUpTo == totalPredictors` in Pass 4:

1. **Transfer Pool 2 slices and dust:**
   - `Treasury.receivePoolFunds(bountyId, p2Buyback,   CAT_BUYBACK)`
   - `Treasury.receivePoolFunds(bountyId, p2Dao,       CAT_DAO)`
   - Direct USDC `transfer` to team wallet for `p2Team`
   - If `B = ∅` (i.e., `topGroupCount == totalPredictors`), `Treasury.receivePoolFunds(bountyId, p2Consolation, CAT_P2_UNALLOCATED)` — Slice A redirect (§5.5).
   - Note: the FEE category was already credited per-predictor at submission time (§5.7); not re-touched here.
   - Sweep accumulated rounding dust: `Treasury.receivePoolFunds(bountyId, dustAmount, CAT_DUST)`.

2. **Compute and assign Pool 3 rewards (paginated allowed):**

```
totalAlloc      = reservedTokenAllocation
amountSlice     = totalAlloc / 2
qualitySlice    = totalAlloc - amountSlice                    // exact remainder

for each i in T:
    amountTokens_i  = mulDiv(amountSlice,
                              pool1Payout_i,
                              sumPool1PayoutTop)
                      // score-0 fallback: weight by effectiveWager_i alone if denominator 0

    qualityTokens_i = mulDiv(qualitySlice,
                              score_i × effectiveWager_i,
                              p1SumTopScoreEffWager × WAD)
                      // score-0 fallback: weight by effectiveWager_i alone if denominator 0

    tokenReward_i   = amountTokens_i + qualityTokens_i
```

Then call `RewardDistributor.assignRewards(bountyId, predictors[T], amounts[T])`. This **may be paginated** in the same `settle()` call OR split across multiple final-finalization invocations if `|T|` is large; each call accumulates against `reservedTokenAllocation`. When all top-group predictors have been assigned, call `RewardDistributor.finalizeRewards(bountyId)`.

3. **Mark settled:**
   - `passCompletedFlags |= 0b1000`, `fullySettled = true`
   - `BountyManager.markSettled(bountyId, topGroupCount)`
   - Emit `SettlementComplete(bountyId, topGroupCount)`

### Gas Pagination Strategy

Each `settle()` call processes a caller-specified range. Callers (oracle, keeper bot, or any user) should batch in groups of **~50–100 predictors** per call to stay within block gas limit. The contract resumes from `settledUpTo` across calls and transparently advances `currentPass` when each pass completes.

Empirical batch tuning is implementation-time work in T4.3a–T4.3e.

### Removed / Renamed (V0.30 → V0.31)

- **Removed:** the `p1RemainderShareExcluded` term from V0.30 §11 Pass 4. The Pool 3 amount slice now weights by the explicit `pool1Payout_i = effectiveWager_i + remainderShare_i` defined in §5.4. See §5.6 note.
- **Removed:** the V0.30 hand-wave "or permissionless with bond" alternative on cutoff hint trust. V0.31 fixes this to permissionless (no bond).
- **Renamed:** `p1SumTopScoreWager` → `p1SumTopScoreEffWager` and analogous `p2*` accumulators, to make explicit that all multiplications use `effectiveWager` not `rawWager`.
- **Added:** Pass 3 reservation step (`RewardDistributor.reserveRewards`).
- **Added:** Final-finalization paginated assignment via `assignRewards` / `finalizeRewards` (replaces V0.30's monolithic `distributeRewards` array call).
- **Added:** explicit zero-predictor short-circuit at start of `settle()`.
- **Added:** Slice A → `CAT_P2_UNALLOCATED` redirect when `B = ∅`.

---

## 12. Security Considerations

### 12.1 Reentrancy
- Every function that moves tokens uses `nonReentrant`.
- CEI pattern throughout: state updates before external calls.
- `claim()` zeroes out `usdcPayout` and `tokenReward` before transfers.

### 12.2 Oracle Trust
- Single `ORACLE_ROLE` at launch. Future: oracle decentralization, dispute windows.
- Oracle can invalidate a proposition. This is a privileged operation; must be logged and monitored.

### 12.3 Sybil Attacks
- 1 USDC minimum wager partially mitigates spam.
- Same address cannot submit twice (enforced by `hasPredicted` mapping).
- Wallet-level Sybil (one user, many addresses) remains possible. Mitigated by:
  - Top-50% cutoff limits rewards to genuinely skilled predictions.
  - Token quality pool uses `score × effectiveWager` (§5.6), so a Sybil army with identical small wagers gets little per-address.
  - Future: optional Worldcoin / Gitcoin Passport integration.

### 12.4 On-Chain Transparency of Predictions (V0.3)

**V0.3 explicitly accepts that predictions are NOT private.** All fields of `Prediction` — `confidenceBpsArray`, `predictedValue`, `rawWager`, `effectiveWager`, `submittedAt` — live in unencrypted contract storage and are readable by anyone running an EVM RPC node or indexer. Front-ends MUST NOT represent prediction privacy as a security feature; the convention "the protocol's official UIs do not surface raw individual predictions" is a UX choice, not a security boundary.

**Front-running is still not a concern**, however. Brier scoring rewards calibrated probability distributions. A mempool observer who copies another predictor's distribution gains no advantage over that predictor — they end up at the same score. There is no "first-mover" or "alpha-leak" attack on prediction submission.

**The genuinely affected concern is sponsor auction value.** The V0.3 model intentionally trades cryptographic exclusivity for protocol simplicity (decision 1). Sponsors pay for the analytics service tier (§1, §8.2), not for information asymmetry. Sponsor auction equilibrium prices in V0.3 will reflect convenience and SLA value (estimated $5K–$50K range based on persona analysis), not the much larger prediction-information premium that an A-route protocol could capture.

**V0.4 upgrade hooks (decision 1, hooks A–E):**

V0.31 ships with the following architectural hooks so that an A-route privacy upgrade in V0.4 does not require protocol rewrite:

| Hook | Where | V0.3 behavior | V0.4 expectation |
|---|---|---|---|
| **A** `bytes encryptedPayload` parameter | `IPredictionEngine.submitPrediction` (§10.2), `Prediction` struct (§9) | `require(encryptedPayload.length == 0)` | Stores ciphertext encrypted to `Bounty.privacyMode`'s key/keyset |
| **B** `PrivacyMode` enum on `Bounty` | §9 `Bounty.privacyMode` | `require(privacyMode == PrivacyMode.Transparent)` at `createBounty` | May be `OracleEncrypted` or `ThresholdEncrypted` for high-value bounties |
| **C** `__gap` storage reservation | All upgradeable contracts (§2) | reserved | New encryption-related storage fields appended without breaking layout |
| **D** `_getPrediction(bountyId, predictor)` internal helper | `PredictionEngine` (§5.9, §11) | Returns plaintext directly | Routes through decryption based on `privacyMode` |
| **E** No batch-public predicate views | `IPredictionEngine` (§10.2) | `getPrediction` is single (bountyId, predictor) only; no `getAllPredictions` | Front-end behavior unchanged on V0.4; aggregate views remain off-chain |

These hooks are MANDATORY for V0.3 compliance. Skipping them turns V0.4 into a protocol rewrite rather than an upgrade.

**V0.3 oracle preemption risk.** Because predictions are transparent on-chain, the oracle (`ORACLE_ROLE`) can read all predictions before invoking `resolve()` and could in principle trade against them in external markets, or selectively `resolveAsInvalid()` based on which bounties are unfavorable to its pre-knowledge. Mitigation in V0.3 is a) operational — oracle held by team multisig with audit logging, b) trust assumption — protocol launches with declared centralized oracle and migrates toward decentralized oracle in Phase 4 (§7.4). This is the same trust assumption Polymarket and most prediction markets ship with at v1; not novel risk.

### 12.5 Settlement Manipulation

**Cutoff hint trust model (decision 4 of L1.B's settlement track,固化 in V0.31).** `submitCutoffHint` is permissionless. Anyone may submit a hint; the on-chain paginated verification (§11 Pass 2) catches any wrong hint deterministically by counting `score > hint` and `score >= hint`. The verification reverts the entire transaction if either bound is violated, returning the contract to the pre-Pass-2 state and consuming only the gas the prover paid.

V0.30 mentioned "oracle-only or permissionless with bond" — V0.31 removes both alternatives: no oracle trust dependency, no bond. Rationale: bond design is a non-trivial economic system (slash conditions, slash beneficiary, bond size) and offers no real protection beyond the on-chain verification's guarantee. Wrong hints simply waste their submitter's gas.

**Stuck-bounty edge case.** If no correct hint is ever submitted (e.g., due to UI bugs), the bounty hangs in Pass 2. In V0.3, the only recovery path is `ORACLE_ROLE.resolveAsInvalid` followed by full refund — same as if the resolution itself were flawed. V0.4 may add a hint-replace flow to recover gracefully.

**Settlement front-running is benign.** Anyone can call `settle()` and `claim()` on any bounty (after their own preconditions are met). Front-running these calls cannot extract value, only spend the front-runner's gas. `claim()` is already CEI-pattern protected against reentrancy (§12.1).

### 12.6 Token Inflation
- Mining schedule is hardcoded. Cannot be increased by admin.
- Monthly cap enforced at contract level.
- Team and DAO buckets have vesting or DAO-gate, preventing dumping.

### 12.7 Upgrade Safety
- All upgradeable contracts use OpenZeppelin's TransparentProxy with a dedicated ProxyAdmin.
- Storage layout: existing variables are never removed or reordered. New variables append at the end. `__gap` reduced accordingly on upgrade.
- Upgrade process documented in `docs/UPGRADE.md` (subtask).

---

## 13. Deployment Dependency Graph

V0.31 splits deployment into two scripts to mirror the §7 phased rollout. `BuybackExecutor` is **not** deployed in Phase 1; it is deployed only after Phase 3 DEX liquidity is seeded.

### 13.1 `script/DeployCore.s.sol` (Phase 1)

```
Step 1.  Deploy BrierMath (library, no proxy)
Step 2.  Deploy PsychohistoryToken (non-upgradeable ERC20 + ERC20Votes + ERC20Permit)
Step 3.  Deploy Treasury implementation + ProxyAdmin + TransparentProxy
           Initializer args: (USDC_ADDRESS, teamWallet, adminMultisig)
Step 4.  Deploy RewardDistributor implementation + proxy
           Initializer args: (PSYH_ADDRESS, adminMultisig, deployTimestamp)
Step 5.  Deploy BountyManager implementation + proxy
           Initializer args: (USDC_ADDRESS, adminMultisig)
Step 6.  Deploy PredictionEngine implementation + proxy (linked against BrierMath)
           Initializer args: (BountyManager, RewardDistributor, Treasury, USDC_ADDRESS, oracleMultisig)
Step 7.  Deploy PsychohistoryRouter implementation + proxy
           Initializer args: (BountyManager, PredictionEngine)

Role grants (atomic, in same script):
  - Treasury.grantRole(PREDICTION_ENGINE_ROLE, PredictionEngine)
  - RewardDistributor.grantRole(PREDICTION_ENGINE_ROLE, PredictionEngine)
  - BountyManager.grantRole(PREDICTION_ENGINE_ROLE, PredictionEngine)
  - PsychohistoryToken.grantRole(MINTER_ROLE, RewardDistributor)
  - PsychohistoryToken.grantRole(TRANSFER_CONTROLLER_ROLE, adminMultisig)
  - PredictionEngine.grantRole(ORACLE_ROLE, oracleMultisig)
  - All contracts: DEFAULT_ADMIN_ROLE → adminMultisig

Output: deployments-core.json with all addresses, ProxyAdmin owners, role-grant txn hashes.
```

T6.1 acceptance criteria for `DeployCore`:

- All proxy initializer values verified against `deployments-core.json` post-deploy.
- ProxyAdmin contract owner equals `adminMultisig` (not the deployer EOA).
- Each role grant verified via `hasRole` view in a separate verification step.
- No leftover `DEFAULT_ADMIN_ROLE` on the deployer EOA (must transfer or revoke before script exits).

### 13.2 `script/DeployBuyback.s.sol` (Phase 3)

Run after DEX liquidity is seeded (Phase 3 onset):

```
Step 1.  Deploy BuybackExecutor implementation + proxy
           Initializer args: (Treasury, PSYH_ADDRESS, USDC_ADDRESS, dexRouter, slippageBps, adminMultisig)
Step 2.  Treasury.grantRole(TREASURY_EXECUTOR_ROLE, BuybackExecutor)
Step 3.  BuybackExecutor.activate()  ← starts the weekly epoch counter

Output: deployments-buyback.json appended to deployments-core.json.
```

T6.1 acceptance criteria for `DeployBuyback`:

- BuybackExecutor activated successfully (`isActivated() == true`).
- DEX router address sanity-checked against published Uniswap V3 / CoW Protocol router.
- First epoch correctly anchored to `block.timestamp`.

### 13.3 Cross-cutting deployment invariants

- Every upgradeable contract uses OpenZeppelin's `TransparentUpgradeableProxy` with a dedicated `ProxyAdmin` per contract (not a shared global admin).
- ProxyAdmin owner = `adminMultisig`, never the deployer.
- All initializer functions have `initializer` modifier, callable once.
- Total `MINTER_ROLE` holders on PsychohistoryToken at end of `DeployCore` = exactly 1 (RewardDistributor).
- Total `ORACLE_ROLE` holders on PredictionEngine at end of `DeployCore` = exactly 1 (oracleMultisig).
- Phase 1 ends with NO BuybackExecutor address granted any role anywhere.

---

## 14. Testing Requirements

Each subtask (see §15) includes its own tests. Integration-level requirements:

### 14.1 Coverage and base scenarios

- **>90% line coverage** on all contracts (`forge coverage`).
- **Full settlement lifecycle test:** propose → deposit (multiple sponsors) → predict (≥ 20) → close → resolve → submit cutoff hint → paginated settle (multiple `settle()` calls across all 4 passes) → claim (USDC + PSYH for all winners + bottom-group consolation) → verify totals against invariants in §14.4.

### 14.2 Edge cases and refund paths

- **Invalidation test:** full flow ending in `resolveAsInvalid` with all predictor wagers + sponsor deposits refunded; `RefundReasonCode == INVALIDATED`.
- **Cancel test:** `cancelBounty` in Open state with no predictions / sponsorships — confirm cancellation; subsequent `addSponsorship` reverts.
- **No-signal test (V0.31 NEW):** create bounty + sponsor, no predictions submitted, `closeBounty` then `resolve()`, then `settle()` short-circuits to `Settled` with `NoSignalSettled` event; sponsors refundable with `reasonCode == NO_SIGNAL`.
- **Tie-handling test:** ≥ 20% of predictors score exactly at cutoff; verify all ties enter top group; `topGroupCount` exceeds `ceil(n/2)` by the tie span.
- **Edge: 1 predictor.** Must not revert. Single predictor enters top group. Pool 1 → return effectiveWager. Pool 2 Slice A → CAT_P2_UNALLOCATED. Pool 2 Slice B → all to predictor (or fallback to effectiveWager weight if score = 0).
- **Edge: all-tie.** All predictors enter top group. `B = ∅`. Pool 1 remainder = 0. Pool 2 Slice A → CAT_P2_UNALLOCATED.

### 14.3 V0.31-specific invariants

- **Per-predictor fee invariant (V0.31 NEW):** for every prediction, `rawWager == effectiveWager + feeAmount`, exact. Property test (≥ 1000 random submissions).
- **Effective-wager-only math invariant (V0.31 NEW):** in any settled bounty, `Σ pool1Payout_i for i ∈ T == Σ effectiveWager_i for all i` (modulo dust). Property: `rawWager` never appears as a coefficient in any pool math.
- **Score-0 fallback test (V0.31 NEW):** craft a Discrete bounty where every predictor's `confidenceBpsArray` puts 100% on the wrong option (so all Brier scores = 2 × WAD → score = 0). Verify Pool 2 Slice A and Pool 3 quality slice both fall back to effectiveWager weighting and complete settlement without revert.
- **Sponsor cap test (V0.31 NEW):** add 100 sponsors (each `addSponsorship` once); 101st `addSponsorship` from a NEW address reverts with `SponsorCapReached`; existing sponsor calling `addSponsorship` again succeeds.
- **`finalizeSponsorRanking` gas test (V0.31 NEW):** with exactly 100 sponsors, measure gas; assert under 12M (well within block limit). With 101+ sponsors the test cannot reach this state because `addSponsorship` rejects.
- **`encryptedPayload` empty enforcement (V0.31 NEW):** all V0.3 `submitPrediction` calls with non-empty `encryptedPayload` revert. Property test with random byte arrays.
- **`PrivacyMode == Transparent` enforcement (V0.31 NEW):** `createBounty` with `OracleEncrypted` or `ThresholdEncrypted` reverts.
- **RewardDistributor reserve/assign/finalize (V0.31 NEW):**
  - Reserve consumes monthly cap headroom immediately.
  - Two bounties in the same month with combined `volumeUsdcRaw × kWad` exceeding cap: first bounty gets full `kWad`; second gets clamped `effectiveKWad < kWad` reflecting only remaining cap.
  - `assignRewards` cumulative sum cannot exceed `reservedAmount` (reverts).
  - `finalizeRewards` releases unassigned remainder back to monthly headroom; subsequent reservations in same month see the freed cap.
- **`claimTokens` after `finalizeRewards`** continues to work; cap check is NOT re-applied at claim time.
- **K-schedule tier transitions (V0.31 NEW):** `currentK()` returns correct `kWad` value at month boundaries (test `block.timestamp` exactly at month 4, month 7, month 13, etc.).
- **K = 0 at month 49+:** `reserveRewards` returns `effectiveKWad == 0` and `reservedAmount == 0`; settlement proceeds with zero Pool 3.

### 14.4 Conservation invariants

For any non-invalidated, non-cancelled, non-no-signal settled bounty:

```
Σ_predictors usdcPayout_i
+ p2Buyback (to Treasury CAT_BUYBACK)
+ p2Team    (to teamWallet)
+ p2Dao     (to Treasury CAT_DAO)
+ slice_A_redirect_if_B_empty (to Treasury CAT_P2_UNALLOCATED, else 0)
+ dust      (to Treasury CAT_DUST)
==
totalEffectiveWagerAmount + totalSponsorAmount
```

Plus the FEE invariant:

```
Σ_predictors feeAmount_i == bounty.totalFeeCollected
                          == Treasury.categoryBalance(CAT_FEE) attributable to this bounty
```

(Off-bounty FEE accumulation also possible from other bounties; the per-bounty assertion uses event-stream tracing, not pure balance.)

### 14.5 Buyback test (decision 2 model)

- **Buyback geometric model test:** seed Treasury with 1200 USDC under CAT_BUYBACK, no further inflows, run 12 epochs:
  - Epoch 1: spends 100, balance 1100.
  - Epoch 2: spends `1100/12 ≈ 91.67`, balance ≈ 1008.33.
  - …
  - Verify balance after 12 epochs ≈ `1200 × (11/12)^12 ≈ 421` (i.e. ~ 65% spent), NOT zero. This is the V0.31 expected behavior (decision 2).
  - All bought-back PSYH burned (verify `totalSupply` decreased by `psyhBought` per epoch).
- **Buyback inflow during decay test:** seed 1200, epoch-1 spend 100, then add 1200 more at epoch-2; verify epoch-2 spend == `(1100 + 1200)/12 ≈ 191.67`. Confirms inflows enter the same rolling pool.

### 14.6 State machine and access control

- **Open → Closed transition:** `closeBounty` callable at/after `closeTimestamp`; idempotent.
- **PE-only mutators:** every BountyManager mutator from §10.1 reverts when called by non-`PREDICTION_ENGINE_ROLE` accounts.
- **Refund predicate:** `claimSponsorshipRefund` rejects when bounty is in any non-refundable state (Open, Closed, Resolved with predictors > 0, Settled with predictors > 0).

---

## 15. Subtask Decomposition

These are the recommended subtasks for parallel development in independent conversations. Each subtask should have this TDD attached as context. Dependencies flow downward; siblings at the same level can be developed in parallel.

### Tier 1 — Foundations (sequential)

**T1.1 Project scaffolding**
- Initialize Foundry project, install OpenZeppelin dependencies
- Set up remappings and compiler config
- Directory structure: `src/core/`, `src/libraries/`, `src/interfaces/`, `src/mocks/`, `test/`, `script/`
- Acceptance: `forge build` succeeds on empty project

**T1.2 Shared types and constants**
- `src/libraries/PsychohistoryTypes.sol`: all enums and structs from §9
- `src/libraries/Constants.sol`: all constants from §9
- Acceptance: `forge build` succeeds; no warnings

**T1.3 All interface files**
- Complete `IBountyManager`, `IPredictionEngine`, `IRewardDistributor`, `ITreasury`, `IBuybackExecutor`, `IPsychohistoryToken`
- Full NatSpec
- Acceptance: `forge build` succeeds; interfaces compile against mock implementations

### Tier 2 — Math and Token (parallel)

**T2.1 BrierMath library**
- Implement all functions from §5.2
- Comprehensive unit tests + fuzz tests
- Acceptance: `forge test --match-path test/BrierMath.t.sol` passes with >95% coverage

**T2.2 PsychohistoryToken**
- ERC20 + ERC20Votes + ERC20Permit (OpenZeppelin)
- Non-upgradeable
- Transfer gating with `_transfersEnabled` flag
- `MINTER_ROLE` for minting; `TRANSFER_CONTROLLER_ROLE` for flag toggle
- Total supply hard cap enforced in `mint()`
- Acceptance: transfer gating test, minting permission test, supply cap test, vote delegation test

### Tier 3 — Core Modules (partial parallelism)

**T3.1 Treasury**
- Proxy-upgradeable
- Category-labeled fund receipt (§10.4)
- DAO sub-account accounting
- `pendingBuybackBalance()` and `scheduleBuyback()` hooks
- Depends on: T1.1, T1.2, T1.3

**T3.2 RewardDistributor**
- Proxy-upgradeable
- Tracks pending mints per `(bountyId, predictor)`
- Implements K schedule (§6.4)
- Monthly cap enforcement with auto-reduction
- `claimTokens()` mints from PsychohistoryToken
- Depends on: T1.1, T1.2, T1.3, T2.2 (token address)

**T3.3 BountyManager**
- Proxy-upgradeable
- Proposition lifecycle and state machine
- Sponsor ranking and tier assignment (§8)
- `claimSponsorshipRefund()` for invalidation and zero-predictor cases
- Depends on: T1.1, T1.2, T1.3

### Tier 4 — PredictionEngine (heaviest)

**T4.1 PredictionEngine — Storage and submission**
- Proxy-upgradeable
- **Acceptance, first criterion (V0.31 NEW):** Before implementing submission logic, define and compile the full PredictionEngine storage layout required by T4.1–T4.4, including settlement pass state and reward accounting fields. Later T4.x tasks may append only with L2 approval.
- `submitPrediction()` with Discrete/Numerical validation; per-predictor 1% fee deduction (§5.7); `encryptedPayload` length-zero enforcement
- Calls `BountyManager.recordPrediction` via PE→BM mutator
- Depends on: T3.3 (BountyManager) finalized

**T4.2 PredictionEngine — Resolution**
- `resolve()` with outcome validation; calls `BountyManager.markResolved`
- `resolveAsInvalid()`; calls `BountyManager.markInvalidated`
- Explicit `closeBounty()` passthrough (Open → Closed)
- Depends on: T4.1

**T4.3 PredictionEngine — Settlement (V0.31: 5-segment split)**

Original V0.30 T4.3 covered §11 all 4 passes + Pool 1/2/3 — too heavy for one L3 conversation. V0.31 splits into 5 sub-tasks:

- **T4.3a Pass 1 (score computation + pagination)** — depends on T4.2, T2.1
- **T4.3b Pass 2 (cutoff hint submission + paginated verification + top group flagging)** — depends on T4.3a
- **T4.3c Pass 3 (accumulator pass + global amount commitment + reserveRewards call)** — depends on T4.3b
- **T4.3d Pass 4 (per-predictor USDC payout assignment)** — depends on T4.3c
- **T4.3e Final finalization (Pool 2 transfers + paginated assignRewards + finalizeRewards + markSettled)** — depends on T4.3d, T3.1, T3.2

**T4.4 PredictionEngine — Claim**
- `claim()`: transfers USDC payout AND calls `RewardDistributor.claimTokens` for atomic USDC + PSYH delivery
- Double-claim protection
- Depends on: T4.3e

### Tier 5 — Router and Buyback (V0.31: split L2-T5a + L2-T5b)

V0.30's L2-T5 grouped Router + BuybackExecutor with no shared dependency. V0.31 splits:

**L2-T5a — PsychohistoryRouter**
- T5.1: atomic multi-step operations (`createBountyAndPredict`, `predictAndSponsor`, etc.)
- Depends on: T4.4, T3.3

**L2-T5b — BuybackExecutor (parallel to entire L2-T4)**
- T5.2: epoch-based geometric buyback (§6.5 model: `currentBalance × 1/12` per epoch)
- L2-T5b.A first decision: DEX venue (Uniswap V3 vs. CoW Protocol vs. both)
- Burn on receipt (100%)
- Depends on: T3.1, T2.2

### Tier 6 — Deployment and Simulation

**T6.1 Deployment script**
- `script/Deploy.s.sol` following §13
- Supports local Anvil and Sepolia
- Outputs `deployments.json`

**T6.2 Integration tests**
- `test/Integration.t.sol`: all scenarios from §14
- Depends on: all Tier 1–5 tasks

**T6.3 Simulation script**
- `script/Simulation.s.sol`: full end-to-end on testnet
- Depends on: T6.1

### Tier 7 — Security Review (final)

**T7.1 Static analysis**
- Slither + Aderyn configured
- Report reviewed, false positives documented

**T7.2 Invariant testing**
- Foundry invariants: total supply, pool sum conservation, top-group cardinality
- Handler-based stateful fuzzing

---

## 16. Open Questions and Deferred Decisions

The following decisions are intentionally deferred to avoid premature complexity:

1. **Sponsor data delivery mechanism.** The contract records tier and timestamp, but actual data hand-off (email, dashboard, API) is off-chain. Off-chain system design is not part of this TDD.
2. **Proposition metadata standard.** `metadataURI` points to IPFS JSON. Schema (question text, option labels, units, source oracle URL) should be defined in a separate product spec.
3. **Front-end architecture.** Not part of this TDD. Will need a Next.js app or similar that integrates with Privy (embedded wallets) and the contracts.
4. **Challenge and dispute mechanisms.** Phase 4 scope. Not addressed here.
5. **Cross-chain strategy.** Single chain at launch. Multi-chain via LayerZero or Wormhole is a future upgrade.
6. **KYC for high-value sponsors.** Not enforced on-chain. Could be required off-chain as a condition of team curation service.

---

## 17. Changelog vs V2

This section is for historical context, in case existing V2 code is referenced.

| Concern | V2 | V3 (this doc) |
|---|---|---|
| Prediction types | Binary, Categorical, Numerical (3 types) | Discrete (N ≥ 2), Numerical (2 types) |
| Prediction input | `selectedOption + confidenceBps` (Categorical) | `confidenceBpsArray` only |
| Stake | Fixed 10 USDC per prediction | Free-form wager, min 1 USDC |
| Pool structure | Single pool (sponsor bounty only) | Three pools: predictor wager, sponsor, token |
| Bottom-50% handling | Total slash to Treasury | Consolation slice from sponsor pool |
| Sponsor mechanics | Static bounty, no bidding | Cumulative bidding with tiered access |
| Token mechanics | vePSYH boost, delegation | Prediction mining + buyback/burn |
| Philosophy | Pure knowledge-purchase | Hybrid PvP + sponsor signal market |

Existing V2 code should not be incrementally modified into V3. This is a ground-up rewrite. V2 is preserved as a separate repository for historical reference.

---

**End of TDD**

The Claude Code master planning conversation should use this document as the authoritative reference when decomposing work into subtask conversations. Each subtask conversation should be given:
1. The relevant sections of this TDD (not the whole thing, to preserve context budget)
2. The specific acceptance criteria from §15
3. Permission to reference this TDD when encountering ambiguity
