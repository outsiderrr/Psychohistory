# Psychohistory — Technical Design Document

**Document Version:** 3.0  
**Project Codename:** Psychohistory  
**Supersedes:** Internal V1 (macro scenario engine), V2 (sponsor-only bounty protocol)  
**Audience:** Claude Code master planning conversation. This document is the single source of truth for architecture. Use it to decompose work into subtasks, each of which will be developed in independent conversations.

---

## 1. Design Philosophy & Positioning

Psychohistory is a decentralized prediction protocol that combines three elements absent from existing predictions markets:

1. **Unified discrete-and-quantitative framework.** Predictors always submit a structured prediction (probability distribution for discrete events, numeric value for quantitative events) paired with a free-form wager amount. Polymarket's AMM architecture cannot natively express quantitative predictions. This is the primary technical moat.
2. **Score-ranked settlement with top-50% cutoff.** Not "correct direction wins" (Polymarket), not "proportional to confidence" (soft ranking). Instead: only predictors in the top-50% score bracket share Pool 1 and Pool 3. Bottom-50% receives only a consolation slice of the sponsor pool. This enforces hard differentiation between good and mediocre predictions.
3. **Sponsor-bidding tiered data access.** Sponsors fund a secondary prize pool and receive staggered post-close access to aggregated prediction data. Bidding for earlier access creates an auction mechanism whose equilibrium price approximates the market value of the aggregated signal.

**What Psychohistory is NOT:**

- Not an AMM. No market-making, no continuous price discovery during the prediction window, no liquidity pools. Predictions lock at submission and settle once at resolution.
- Not a pure PvP casino. Sponsor pool creates a third source of funds beyond loser stakes, enabling sustainable value flow even in low-volume markets.
- Not a fully decentralized protocol at launch. Proposition creation is centralized (team-curated). Decentralized challenge mechanisms are deferred to future phases.

**Primary user personas:**

- **Predictors:** Individual users submitting predictions with real USDC wagers. Motivated by (a) winning other predictors' wagers, (b) sponsor pool bonuses, (c) earning $PSYH tokens through prediction mining.
- **Sponsors:** Entities that fund a prediction with USDC in exchange for tiered early access to aggregated predictions. Motivated by information arbitrage (e.g., using the aggregated signal to trade on Polymarket or other markets).
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
- Scoring: normalized absolute error. Given WAD-scaled predicted value `p` and resolved value `a`, raw error is `|p - a|`. Scoring depends on ranking, not absolute value (see §5.3).

### 3.3 Unification

Both types resolve into a single sortable score per prediction. The settlement pipeline is type-agnostic once scores are computed. `BrierMath` exposes distinct scoring functions but a common interface for downstream ranking.

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

**Pool 1 — Predictor Wager Pool**
- Source: sum of all predictor wagers for this proposition.
- Recipients: Top-50% predictors only.
- Distribution rule: detailed in §5.4.

**Pool 2 — Sponsor Pool**
- Source: sum of all sponsor deposits for this proposition.
- Split: `30% / 30% / 20% / 10% / 10%`
  - **30% consolation:** bottom-50% predictors, weighted by `score × wager`.
  - **30% victory bonus:** top-50% predictors, weighted by `score × wager`.
  - **20% buyback allocation:** transferred to Treasury, earmarked for $PSYH buybacks.
  - **10% team allocation:** transferred to team wallet.
  - **10% DAO fund allocation:** transferred to Treasury's DAO-controlled sub-account.

**Pool 3 — Token Reward Pool**
- Source: `RewardDistributor` mints from the 40% mining allocation.
- Total tokens per proposition: `propositionTokenAllocation = (Pool 1 total + Pool 2 total) × K(t)`
  - `K(t)` is a time-decay coefficient (see §6.4).
- Split: `50% / 50%`
  - **50% amount pool:** top-50% predictors, weighted by their Pool 1 winnings.
  - **50% quality pool:** top-50% predictors, weighted by `score × wager`.
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

### 5.4 Pool 1 Distribution (Principal Protection + Remainder)

This is the rule you specified explicitly. It differs from the naive "weighted by score × wager" approach.

Let:
- `P1_total` = sum of all predictor wagers.
- `T` = set of top-50% predictors (by score cutoff rule in §5.3).
- `P1_principal = Σ (wager_i for i ∈ T)` = sum of top-group wagers.
- `P1_remainder = P1_total - P1_principal` = sum of bottom-group wagers, to be redistributed.

Distribution:
1. Each `i ∈ T` first receives back their own `wager_i` (principal protection).
2. `P1_remainder` is distributed among `T` proportionally to `score_i × wager_i`:
   ```
   remainderShare_i = P1_remainder × (score_i × wager_i) / Σ(score_j × wager_j for j ∈ T)
   ```
3. Each `i ∈ T` receives `payout_i = wager_i + remainderShare_i`.

**Invariant:** `Σ payout_i = P1_total` (sum of all wagers conserved, minus rounding dust which goes to Treasury).

**Invariant:** Every top-group predictor receives at least `wager_i` back, so they cannot lose money in Pool 1. This addresses the concern that a low-ranked top-50% predictor could be wiped out by a high-weight peer.

### 5.5 Pool 2 Distribution

Let `P2_total` = sum of all sponsor deposits for this proposition.

**Slice A — Consolation (30% of P2_total):**
- Distributed to bottom-50% predictors.
- Let `B` = bottom group, `P2_A_amount = P2_total × 0.30`.
- For each `i ∈ B`:
  ```
  consolation_i = P2_A_amount × (score_i × wager_i) / Σ(score_j × wager_j for j ∈ B)
  ```
- If `B` is empty (all predictors are in top group due to ties), Slice A goes to Treasury.

**Slice B — Victory Bonus (30% of P2_total):**
- Distributed to top-50% predictors.
- Let `P2_B_amount = P2_total × 0.30`.
- For each `i ∈ T`:
  ```
  victoryBonus_i = P2_B_amount × (score_i × wager_i) / Σ(score_j × wager_j for j ∈ T)
  ```

**Slice C — Buyback (20% of P2_total):**
- Transferred to `Treasury` with a label/event indicating buyback allocation.

**Slice D1 — Team (10% of P2_total):**
- Transferred directly to team wallet address (configurable, set via `DEFAULT_ADMIN_ROLE`).

**Slice D2 — DAO Fund (10% of P2_total):**
- Transferred to `Treasury`'s DAO-controlled sub-account.
- Before DAO exists: admin multisig custody. Documented in whitepaper and on-chain metadata.

### 5.6 Pool 3 Distribution

Total token allocation for this proposition:
```
propositionTokenAllocation = (P1_total + P2_total) × K(t)
```
where `K(t)` follows the schedule in §6.4. `K` has units of "$PSYH per USDC."

**Slice A — Amount Pool (50% of allocation):**
- Recipients: top-50% predictors.
- Weight: each predictor's Pool 1 winnings (`payout_i` as computed in §5.4).
- For each `i ∈ T`:
  ```
  amountTokens_i = (propositionTokenAllocation / 2) × payout_i / Σ(payout_j for j ∈ T)
  ```

**Slice B — Quality Pool (50% of allocation):**
- Recipients: top-50% predictors.
- Weight: `score_i × wager_i`.
- For each `i ∈ T`:
  ```
  qualityTokens_i = (propositionTokenAllocation / 2) × (score_i × wager_i) / Σ(score_j × wager_j for j ∈ T)
  ```

Total tokens for predictor `i ∈ T`: `amountTokens_i + qualityTokens_i`.  
Total tokens for predictor `i ∈ B`: `0`.

### 5.7 Platform Fee

A flat **1%** fee is deducted from `P1_total` before the rule in §5.4 is applied. The fee goes to `Treasury`. The effective `P1_total` used in §5.4 is `0.99 × raw total wager sum`.

No fee on Pool 2 (sponsors already pay via the 20% buyback + 10% team + 10% DAO allocations, totaling 40% platform take).

### 5.8 Invalidation and Total Refund

If the oracle calls `resolveAsInvalid()`:

- All predictors get 100% wager refund. No ranking, no scoring, no $PSYH minting.
- Sponsors get 100% deposit refund via `BountyManager.claimSponsorshipRefund()`.
- No fees collected.
- `BountyState` transitions to `Invalidated` then `Settled` after refunds processed.

### 5.9 Edge Cases

- **Zero predictors at resolution:** no Pool 1 distribution. Pool 2 splits: Slice A and B both empty → redirected to Treasury. Slices C/D1/D2 execute normally. Sponsors cannot reclaim — they bought signal access; no signal was generated, but no refund either (business risk). *Alternative: grant `claimSponsorshipRefund()` when `predictorCount == 0`. We enable this by treating `predictorCount == 0` as an implicit invalidation for refund purposes. See interface spec.*
- **All predictors tie:** if all `n` predictors have identical scores, they are all in the top group (ties include). Bottom group is empty. Pool 1 principal-protection step still works (each gets wager back); remainder is 0, so no additional distribution. Pool 2 consolation → Treasury; Pool 2 victory bonus distributed evenly by wager.
- **Only one predictor:** top group has 1 member. Bottom group is empty. Same handling as "all tie."
- **Rounding dust:** always rounded down. Remaining dust after all distributions is swept to Treasury at end of `settle()`.

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
| Prediction Mining | 40% | 400,000,000 | Released over 4 years per §6.4 |
| Team | 15% | 150,000,000 | 1-year cliff, 3-year linear vesting after |
| DAO Treasury | 20% | 200,000,000 | Locked; unlocked by DAO governance post-formation |
| Liquidity Reserve | 10% | 100,000,000 | Used for initial Uniswap pool seeding in Phase 3 |
| Airdrop / Marketing | 10% | 100,000,000 | Allocated over 4 years; early airdrops at milestones |
| Reserve | 5% | 50,000,000 | Contingency; no predefined schedule |

### 6.3 Transfer Gating

`PsychohistoryToken` has a `_transfersEnabled` boolean flag, initially `false`. Only `TRANSFER_CONTROLLER_ROLE` can flip it. Minting (by `RewardDistributor`) is always allowed. During Phase 1 (0–3 months post-launch), transfers are disabled; users accumulate but cannot move tokens.

See §7 for phased rollout schedule.

### 6.4 Mining Schedule

The K coefficient determines how many PSYH are minted per USDC of proposition volume.

```
K(t) by month since token contract deployment:
  Month 1–3:   K = 10    (aggressive early incentive)
  Month 4–6:   K = 5
  Month 7–12:  K = 2
  Month 13–24: K = 1
  Month 25–36: K = 0.5
  Month 37–48: K = 0.25
  Month 49+:   K = 0     (mining ends)
```

**Monthly emission cap:** each month has a hard cap derived from the 40% mining allocation divided by 48 months, with front-loading:
```
Month 1–12:   total release = 200M PSYH (50% of mining allocation)
Month 13–24:  total release = 120M PSYH (30%)
Month 25–36:  total release = 50M PSYH (12.5%)
Month 37–48:  total release = 30M PSYH (7.5%)
```

The `RewardDistributor` contract tracks cumulative emissions and automatically reduces K if the monthly cap is exceeded. Exact cap enforcement logic is implementation detail.

### 6.5 Buyback & Burn

**Source of buyback funds:**
- 1% platform fee from Pool 1
- 20% of Pool 2
- Accumulated in `Treasury` as USDC

**Buyback execution:**
- `BuybackExecutor` runs weekly. Each week, it spends `1/12` of the Treasury's buyback balance.
- Net effect: each dollar enters the buyback queue and is spent linearly over 12 weeks.
- Execution uses Uniswap V3 or CoW Protocol with TWAP; exact integration is implementation detail (see subtasks §11).
- All bought-back PSYH is **burned** (100% burn). Treasury does not retain buyback PSYH.

**Before Phase 3 (DEX listing):**
- Buyback queue accumulates USDC. No buybacks execute until `BuybackExecutor.activate()` is called by admin after DEX liquidity exists.

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

Sponsors deposit USDC at any time between `openTimestamp` and `closeTimestamp`. Deposits are **additive** — a sponsor may call `addSponsorship()` multiple times, each call increases their total contribution.

At `closeTimestamp`, sponsor contributions are frozen. The ranking of sponsors by total contribution is locked and becomes immutable.

### 8.2 Tiered Data Access

After `closeTimestamp` but before `resolutionTimestamp`, the oracle (or a designated off-chain service) progressively releases aggregated prediction data:

- **Tier 1 (day 1):** top 1 sponsor by contribution
- **Tier 2 (day 2):** next 2 sponsors (ranks 2–3)
- **Tier 3 (day 3):** next 3 sponsors (ranks 4–6)
- **Tier N (day N):** next N sponsors
- **Final tier:** all remaining sponsors, no cap (day N+1 or final day)
- **Public:** everyone else, at `resolutionTimestamp`

**Rule:** Tier `k` has capacity for `k` sponsors. If fewer sponsors exist than capacity, the sequence collapses — sponsors fill tiers 1, 2, 3 in order until all are placed. Tier timing still uses day-indexing (tier `k` opens on day `k`).

**Implementation:** the sponsor ranking and tier timestamps are recorded on-chain via `BountyManager`. Actual data delivery happens off-chain (email, dashboard, API) and is outside the contract's responsibility. The contract's role is to produce an immutable record that off-chain systems can reference.

### 8.3 Aggregated Data Content

The "aggregated data" given to sponsors consists of:

- `wagerWeightedDistribution`: for each option (discrete) or value histogram bucket (numerical), the sum of wagers placed. This is the market-implied probability distribution weighted by skin-in-the-game.
- `predictionCount`: number of unique predictors.
- `totalWager`: sum of all wagers.

Individual predictor identities and raw predictions are **not** revealed. Only aggregated statistics.

### 8.4 Sponsor Refund

Sponsors can reclaim deposits in these cases:
- `BountyState.Invalidated` (oracle calls `resolveAsInvalid`)
- `BountyState.Cancelled` (admin cancels before any sponsorship or prediction exists)
- `predictorCount == 0` at resolution (no signal generated)

Sponsors cannot reclaim when settlement proceeds normally. The 40% of Pool 2 they pay in platform take (20% buyback + 10% team + 10% DAO) is the cost of the service.

---

## 9. Data Structures

Full Solidity definitions. These should be in `src/libraries/PsychohistoryTypes.sol` or equivalent shared location.

```solidity
enum BountyState {
    Open,              // accepting predictions and sponsorships
    Closed,            // prediction window ended; tiered data release in progress
    Resolved,          // oracle has submitted outcome
    Settled,           // settlement complete; funds distributed
    Invalidated,       // oracle voided the bounty
    Cancelled          // admin cancelled before predictions
}

enum PropositionType {
    Discrete,          // N options, 2 ≤ N ≤ 5
    Numerical          // continuous value
}

struct Bounty {
    uint256 bountyId;
    address creator;                    // team address in Phase 1
    string metadataURI;                 // IPFS CID: question text, option labels, units
    PropositionType propositionType;
    uint8 optionsCount;                 // Discrete: 2–5. Numerical: 0.
    uint64 openTimestamp;
    uint64 closeTimestamp;
    uint64 resolutionDeadline;
    BountyState state;

    uint256 resolvedValue;              // Discrete: winning option index. Numerical: scaled integer.
    uint8 resolvedDecimals;

    uint256 totalPredictors;
    uint256 totalWagerAmount;           // Pool 1 raw total (before fee)
    uint256 totalSponsorAmount;         // Pool 2 total
    uint256 winnersCount;               // set during settlement
}

struct Prediction {
    address predictor;
    uint256 bountyId;
    uint256 wagerAmount;
    uint256 submittedAt;                // for deterministic tiebreaking if needed

    // Discrete fields (unused for Numerical)
    // confidenceBpsArray stored in separate mapping for gas

    // Numerical fields (unused for Discrete)
    uint256 predictedValue;
    uint8 predictedDecimals;

    // Computed during settlement
    uint256 score;                      // WAD-scaled, filled by settle()
    bool inTopGroup;                    // true if passed top-50% cutoff
    bool processed;                     // true if settlement visited this prediction

    // Claimable amounts (filled by settle())
    uint256 usdcPayout;                 // Pool 1 payout + Pool 2 slice (consolation or victory)
    uint256 tokenReward;                // Pool 3 $PSYH to mint on claim
    bool claimed;
}

struct Sponsorship {
    address sponsor;
    uint256 bountyId;
    uint256 contributionAmount;         // cumulative; increments on addSponsorship()
    uint256 rank;                       // set at closeTimestamp; 0 = highest
    uint64 tier;                        // set at closeTimestamp; 1 = earliest access
    uint64 accessUnlockTimestamp;       // set at closeTimestamp
    bool refunded;                      // true if invalidation refund was claimed
}

struct SettlementState {
    bool resolved;
    bool isInvalidated;
    uint256 resolvedValue;
    uint8 resolvedDecimals;

    uint256 topGroupCount;
    uint256 topGroupCutoffScore;        // minimum score to be in top group
    uint256 settledUpTo;                // pagination cursor
    bool fullySettled;

    // Sums used during settlement (computed in passes)
    uint256 p1SumTopScoreWager;         // Σ(score × wager) for top group
    uint256 p1TopPrincipal;              // Σ(wager) for top group
    uint256 p1Remainder;                 // = totalWager - totalFee - topPrincipal
    uint256 p2SumBottomScoreWager;      // Σ(score × wager) for bottom group
    uint256 p2SumTopScoreWager;         // Σ(score × wager) for top group
    uint256 totalFeeCollected;
}
```

**Separate mappings (for gas and variable-length array safety):**
```solidity
mapping(uint256 => mapping(address => uint256[])) public confidenceArrays;
// bountyId → predictor → confidenceBpsArray
```

**Constants (`src/libraries/Constants.sol`):**
```solidity
uint256 constant MIN_WAGER = 1e6;             // 1 USDC
uint256 constant WAD = 1e18;
uint256 constant BPS = 10_000;
uint256 constant MIN_BRIER = 1e12;
uint256 constant MAX_BRIER_SCORE = 2e18;
uint256 constant WAD_SQUARED = 1e36;
uint8   constant MAX_OPTIONS = 5;
uint8   constant MIN_OPTIONS = 2;
uint256 constant PLATFORM_FEE_BPS = 100;      // 1%
uint256 constant P2_CONSOLATION_BPS = 3000;   // 30%
uint256 constant P2_VICTORY_BPS = 3000;       // 30%
uint256 constant P2_BUYBACK_BPS = 2000;       // 20%
uint256 constant P2_TEAM_BPS = 1000;          // 10%
uint256 constant P2_DAO_BPS = 1000;           // 10%
uint256 constant TOP_GROUP_BPS = 5000;        // 50%
uint256 constant BUYBACK_EPOCH_COUNT = 12;    // 12-week TWAP
uint256 constant BUYBACK_EPOCH_DURATION = 7 days;
```

---

## 10. Interface Specifications

### 10.1 IBountyManager

```solidity
interface IBountyManager {
    event BountyCreated(uint256 indexed bountyId, address indexed creator, PropositionType propositionType, uint8 optionsCount);
    event SponsorshipDeposited(uint256 indexed bountyId, address indexed sponsor, uint256 totalContribution);
    event BountyStateChanged(uint256 indexed bountyId, BountyState oldState, BountyState newState);
    event SponsorTierAssigned(uint256 indexed bountyId, address indexed sponsor, uint256 rank, uint64 tier, uint64 accessUnlock);
    event SponsorRefundClaimed(uint256 indexed bountyId, address indexed sponsor, uint256 amount);

    function createBounty(
        PropositionType propositionType,
        uint8 optionsCount,
        string calldata metadataURI,
        uint64 openTimestamp,
        uint64 closeTimestamp,
        uint64 resolutionDeadline
    ) external returns (uint256 bountyId);

    function addSponsorship(uint256 bountyId, uint256 amount) external;

    function finalizeSponsorRanking(uint256 bountyId) external;
    // Called by PredictionEngine at closeTimestamp. Sorts sponsors by contributionAmount
    // and assigns rank, tier, and accessUnlockTimestamp.

    function claimSponsorshipRefund(uint256 bountyId) external;
    // Callable only if state is Invalidated, Cancelled, or predictorCount == 0.

    function cancelBounty(uint256 bountyId) external;
    // Admin-only. Only if no predictions or sponsorships exist.

    function getBounty(uint256 bountyId) external view returns (Bounty memory);
    function getSponsorship(uint256 bountyId, address sponsor) external view returns (Sponsorship memory);
    function getAllSponsors(uint256 bountyId) external view returns (address[] memory);
}
```

### 10.2 IPredictionEngine

```solidity
interface IPredictionEngine {
    event PredictionSubmitted(uint256 indexed bountyId, address indexed predictor, uint256 wagerAmount);
    event PredictionResolved(uint256 indexed bountyId, uint256 resolvedValue);
    event BountyInvalidated(uint256 indexed bountyId);
    event SettlementProgressed(uint256 indexed bountyId, uint256 settledUpTo, uint256 totalPredictors);
    event SettlementComplete(uint256 indexed bountyId, uint256 topGroupCount);
    event PayoutClaimed(uint256 indexed bountyId, address indexed predictor, uint256 usdcAmount, uint256 tokenAmount);

    function submitPrediction(
        uint256 bountyId,
        uint256[] calldata confidenceBpsArray,  // Discrete: length == optionsCount, sum == 10000. Numerical: empty.
        uint256 predictedValue,                 // Numerical only
        uint8 predictedDecimals,                // Numerical only
        uint256 wagerAmount                     // USDC; min 1e6
    ) external;

    function resolve(
        uint256 bountyId,
        uint256 resolvedValue,
        uint8 resolvedDecimals
    ) external;
    // Only ORACLE_ROLE. Transitions Closed → Resolved.

    function resolveAsInvalid(uint256 bountyId) external;
    // Only ORACLE_ROLE. Transitions Closed → Invalidated.

    function settle(uint256 bountyId, uint256 startIndex, uint256 endIndex) external;
    // Anyone can call. Multi-pass algorithm; see §11. On final page, triggers
    // Pool 2 distributions and Pool 3 minting.

    function claim(uint256 bountyId) external;
    // Pull pattern. Transfers USDC payout and triggers token minting.

    function getPrediction(uint256 bountyId, address predictor) external view returns (Prediction memory);
    function getPredictorCount(uint256 bountyId) external view returns (uint256);
    function getSettlementState(uint256 bountyId) external view returns (SettlementState memory);
}
```

### 10.3 IRewardDistributor

```solidity
interface IRewardDistributor {
    event TokenReward(uint256 indexed bountyId, address indexed predictor, uint256 amount);
    event KCoefficientUpdated(uint256 newK);

    function distributeRewards(
        uint256 bountyId,
        address[] calldata predictors,
        uint256[] calldata amounts
    ) external;
    // Called by PredictionEngine. Tracks pending mints; does not mint immediately.

    function claimTokens(uint256 bountyId, address predictor) external returns (uint256 minted);
    // Called by PredictionEngine (or predictor directly). Mints $PSYH from the 40% bucket
    // up to per-month cap.

    function currentK() external view returns (uint256);
    function pendingTokensOf(uint256 bountyId, address predictor) external view returns (uint256);
}
```

### 10.4 ITreasury

```solidity
interface ITreasury {
    event FundsReceived(uint256 indexed bountyId, address source, uint256 amount, bytes32 category);
    event BuybackExecuted(uint256 amountIn, uint256 tokensBurned);
    event DAOFundWithdrawal(address to, uint256 amount, string reason);

    function receivePoolFunds(uint256 bountyId, uint256 amount, bytes32 category) external;
    // Categories: keccak256("FEE"), keccak256("BUYBACK"), keccak256("DAO"), keccak256("DUST")

    function pendingBuybackBalance() external view returns (uint256);
    function scheduleBuyback(uint256 weekIndex) external;
    // Called by BuybackExecutor weekly. Transfers 1/12 of current buyback balance.

    function daoWithdraw(address to, uint256 amount, string calldata reason) external;
    // DEFAULT_ADMIN_ROLE only. Withdraws from DAO sub-account.
}
```

### 10.5 IBuybackExecutor

```solidity
interface IBuybackExecutor {
    event BuybackEpochExecuted(uint256 indexed epoch, uint256 usdcSpent, uint256 psyhBought, uint256 psyhBurned);

    function activate() external;
    // DEFAULT_ADMIN_ROLE. Starts weekly epochs.

    function executeEpoch() external;
    // Permissionless. Callable once per epoch (checks block.timestamp against last epoch).
    // Spends 1/12 of Treasury's buyback balance, executes TWAP on Uniswap V3 or CoW Protocol,
    // burns all PSYH received.

    function currentEpoch() external view returns (uint256);
    function lastExecutedEpoch() external view returns (uint256);
}
```

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

This is the most complex function and warrants detailed specification.

**Input:** `bountyId`, `startIndex`, `endIndex`.

**Precondition:** `SettlementState.resolved == true` and `SettlementState.isInvalidated == false`.

**High-level: three-pass algorithm split across multiple calls.**

Because Solidity has no cheap sort and unbounded arrays are gas-hazardous, settlement uses an explicit multi-pass approach:

### Pass 1 — Score Computation

For each prediction in `[startIndex, endIndex)`:
1. Compute `score`:
   - Discrete: `score = (2 × WAD) - computeCategoricalBrierScore(confidenceWadArray, resolvedOptionIndex)`
   - Numerical: `score = WAD_SQUARED / max(computeAbsoluteError(predictedWad, resolvedWad), MIN_BRIER)`
2. Store `score` in `predictions[bountyId][predictor]`.
3. Increment `SettlementState.settledUpTo`.

When `settledUpTo == totalPredictors`, Pass 1 is complete. Mark a sub-flag `pass1Complete = true`.

### Pass 2 — Cutoff Determination and Top Group Assignment

Pass 2 cannot be done in-memory sort (gas hazard). Instead, it's done via an **off-chain hint with on-chain verification**, similar to the V2 numerical median mechanism:

1. Off-chain, compute `cutoffScore` (the minimum score that places a prediction in the top 50%, with tie-inclusion).
2. Call `submitCutoffHint(bountyId, cutoffScore)` (oracle-only or permissionless with bond).
3. On-chain verification: iterate all predictions, count how many have `score >= cutoffScore` (→ `topCount`) and how many have `score > cutoffScore` (→ `strictlyAboveCount`).
4. Verify: `strictlyAboveCount ≤ ceil(n/2)` AND `topCount ≥ ceil(n/2)`. If not, revert.
5. If verification passes, store `topGroupCutoffScore = cutoffScore` and `topGroupCount = topCount`.
6. In a final pass (or same pass), set `inTopGroup` flag on each prediction.

**Note:** Pass 2 iteration can be paginated the same way Pass 1 is.

### Pass 3 — Payout Computation

For each prediction in `[startIndex, endIndex)`:
1. If `inTopGroup`:
   - Accumulate `p1TopPrincipal += wager`
   - Accumulate `p1SumTopScoreWager += score × wager / WAD`
   - Accumulate `p2SumTopScoreWager += score × wager / WAD`
2. Else:
   - Accumulate `p2SumBottomScoreWager += score × wager / WAD`

When Pass 3 is complete, compute global amounts:
- `p1RawTotal = totalWagerAmount`
- `totalFee = p1RawTotal × PLATFORM_FEE_BPS / BPS`
- `p1EffectiveTotal = p1RawTotal - totalFee`
- `p1Remainder = p1EffectiveTotal - p1TopPrincipal`
- `p2Total = totalSponsorAmount`
- `p2Consolation = p2Total × P2_CONSOLATION_BPS / BPS`
- `p2Victory = p2Total × P2_VICTORY_BPS / BPS`
- `p2Buyback = p2Total × P2_BUYBACK_BPS / BPS`
- `p2Team = p2Total × P2_TEAM_BPS / BPS`
- `p2Dao = p2Total × P2_DAO_BPS / BPS`

### Pass 4 — Final Payout Assignment

For each prediction:
- If `inTopGroup`:
  ```
  remainderShare = p1Remainder × (score × wager) / (p1SumTopScoreWager × WAD)
  victoryBonus = p2Victory × (score × wager) / (p2SumTopScoreWager × WAD)
  usdcPayout = wager + remainderShare + victoryBonus
  ```
- Else:
  ```
  consolation = p2Consolation × (score × wager) / (p2SumBottomScoreWager × WAD)
  usdcPayout = consolation
  ```

Store in `prediction.usdcPayout`.

### Final Finalization (only on last batch)

1. Transfer `totalFee` to Treasury with category `FEE`.
2. Transfer `p2Buyback` to Treasury with category `BUYBACK`.
3. Transfer `p2Team` to team wallet.
4. Transfer `p2Dao` to Treasury with category `DAO`.
5. Sweep any rounding dust to Treasury with category `DUST`.
6. Compute Pool 3 per predictor:
   ```
   propositionTokenAllocation = (p1RawTotal + p2Total) × currentK() / 1e6
   // Divide by 1e6 to convert USDC to PSYH units: 1 USDC * K = K PSYH

   for each i in top group:
       amountTokens_i = (propositionTokenAllocation / 2) × (usdcPayout_i - p1RemainderShareExcluded) / Σ(payout_j)
       qualityTokens_i = (propositionTokenAllocation / 2) × (score_i × wager_i) / Σ(score_j × wager_j)
       tokenReward_i = amountTokens_i + qualityTokens_i
   ```
   Call `RewardDistributor.distributeRewards(bountyId, predictors, amounts)`.
7. Set `fullySettled = true` and `Bounty.state = Settled`. Emit `SettlementComplete`.

### Gas Pagination Strategy

Each `settle()` call processes a caller-specified range. Callers (typically the oracle or a keeper bot) should batch in groups of ~50–100 predictors to stay within block gas limit. The contract must support resuming from `settledUpTo` across calls and across passes.

State machine flags:
```
settledUpTo          // advances within a pass
currentPass          // 1, 2, 3, or 4
passCompletedFlags   // bitflag for pass 1, 2, 3, 4 completion
```

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
  - Token quality pool uses `score × wager`, so a Sybil army with identical small wagers gets little per-address.
  - Future: optional Worldcoin / Gitcoin Passport integration.

### 12.4 Front-Running on `submitPrediction`
- Not a concern. Predictions are private (not readable by other users in any UI designed for this protocol — front-ends must not expose them). Even if a mempool observer sees another user's prediction, they cannot use it to construct a "better" prediction because Brier scoring rewards calibrated probability distributions; copying another's distribution provides no edge.

### 12.5 Settlement Manipulation
- Pass 2's off-chain hint pattern requires trust in the hint submitter. Mitigations:
  - Hint validation on-chain (counting above/equal/below) detects wrong hints.
  - Hint submitter could be required to post a bond that is slashed if verification fails.
  - As a fallback, allow fully on-chain sort for small `n` (< 100) using insertion sort.

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

```
Phase 1 deploy:
  1. BrierMath (library)
  2. PsychohistoryToken (immutable ERC20)
  3. Treasury (proxy) ← initialized with USDC address, team wallet, admin
  4. RewardDistributor (proxy) ← initialized with PsychohistoryToken address, admin
  5. BountyManager (proxy) ← initialized with USDC address, admin
  6. PredictionEngine (proxy) ← initialized with refs to BountyManager, RewardDistributor, Treasury, USDC
  7. PsychohistoryRouter (proxy) ← initialized with refs to BountyManager, PredictionEngine

Phase 3 deploy (after DEX liquidity):
  8. BuybackExecutor (proxy) ← initialized with refs to Treasury, PSYH, Uniswap V3 / CoW router

Role grants after deployment:
  - PredictionEngine granted PREDICTION_ENGINE_ROLE on Treasury, RewardDistributor, BountyManager
  - BuybackExecutor granted TREASURY_EXECUTOR_ROLE on Treasury (Phase 3)
  - RewardDistributor granted MINTER_ROLE on PsychohistoryToken
  - Oracle multisig granted ORACLE_ROLE on PredictionEngine
  - Deployer/admin multisig holds DEFAULT_ADMIN_ROLE on all contracts
```

---

## 14. Testing Requirements

Each subtask (see §15) includes its own tests. Integration-level requirements:

- **>90% line coverage** on all contracts. `forge coverage`.
- **Full settlement lifecycle test**: propose → deposit → predict (≥20) → resolve → paginated settle → claim (for all winners) → verify totals.
- **Invalidation test**: full flow ending in `resolveAsInvalid` with all refunds.
- **Tie-handling test**: proposition where 20% of predictors score exactly at cutoff; verify all ties are included.
- **Edge case: 1 predictor**: must not revert, predictor gets own wager back + any sponsor victory bonus.
- **Edge case: 0 predictors**: sponsors refunded, no token minting.
- **Sponsor ranking test**: add 10 sponsors with varying amounts in varying orders, close, verify rank and tier assignments.
- **Multi-sponsor additive test**: same sponsor calls `addSponsorship()` 3 times, verify cumulative contribution is used for ranking.
- **Token mining cap test**: simulate a month of heavy activity, verify K auto-adjusts when cap approached.
- **Buyback TWAP test**: seed Treasury with 1200 USDC of buyback category, run 12 epochs, verify each burns 100 USDC worth of PSYH.
- **Invariant test**: in any settled proposition, `Σ usdcPayout_i + fees + sponsor splits = totalWager + totalSponsor`.

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
- `submitPrediction()` with Discrete/Numerical validation
- Depends on: T3.3 (BountyManager)

**T4.2 PredictionEngine — Resolution**
- `resolve()` with outcome validation
- `resolveAsInvalid()`
- State transition to `Resolved` / `Invalidated`
- Depends on: T4.1

**T4.3 PredictionEngine — Settlement (4-pass)**
- Full paginated settlement algorithm (§11)
- Cutoff hint submission and verification
- Pool 1 distribution (principal protection + remainder)
- Pool 2 distribution (all 5 slices)
- Pool 3 delegation to RewardDistributor
- Depends on: T4.2, T3.1, T3.2, T2.1

**T4.4 PredictionEngine — Claim**
- `claim()` function: transfers USDC payout, triggers token minting
- Double-claim protection
- Depends on: T4.3

### Tier 5 — Router and Buyback (parallel)

**T5.1 PsychohistoryRouter**
- Atomic multi-step operations: `createBountyAndPredict`, `predictAndSponsor`, etc.
- Depends on: T4.4, T3.3

**T5.2 BuybackExecutor**
- Epoch-based TWAP buyback
- Uniswap V3 integration (and/or CoW Protocol)
- Burn on receipt
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
