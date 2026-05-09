# Stage 1 — L2-T1 Foundations: Paste-Ready L3 Tasks

**Stage scope**: Initialize the project skeleton — Foundry + OpenZeppelin + directory layout + shared types + interfaces. Sequential L3s; no parallelism in this stage.

**Author**: L2-T1 integration planner (this conversation)
**Effective from**: 2026-05-09
**Status**: Active. T1.1 ready to start; T1.2 / T1.3 prompts will be authored after T1.1 PR is merged.

**Sequence**: T1.1 → T1.2 → T1.3 (each subject to full ABC + L2 acceptance per `docs/reviews/master_plan/2026-05-09_review_routine_governance.md`).

**Stage 1 specific note**: T1.1 is the **first L3 to run formal ABC**. It serves double-duty as the dry-run for the entire governance pipeline (worktree management, PR mechanics, B-stage Codex review, C-stage fix loop, L2 acceptance). Bias toward strict adherence to governance even where shortcuts may seem tempting.

---

## T-1.1 Project scaffolding

**Paste-ready prompt**: [`docs/prompts/stage_1/T-1.1.md`](prompts/stage_1/T-1.1.md)
**Branch**: `claude/T-1.1-scaffolding`
**Estimated effort**: 1–2 hours of L3 conversation work
**Dependencies**: none

### Goal
Bootstrap the Foundry project with OpenZeppelin v5 dependencies and the directory layout from TDD §2 conventions. No protocol code yet — only the skeleton.

### Acceptance criteria

1. `forge init --no-commit` was run; project root contains `foundry.toml`, `remappings.txt`, `lib/`, `src/`, `test/`, `script/`.
2. OpenZeppelin v5.x and OZ-Upgradeable v5.x installed as git submodules under `lib/`. Each submodule pinned to a specific commit hash (no floating tags).
3. Directory structure created (empty placeholder `.gitkeep` files allowed):
   - `src/core/`
   - `src/libraries/`
   - `src/interfaces/`
   - `src/mocks/`
   - `test/`
   - `script/`
4. `foundry.toml` profile sets:
   - `solc_version = "0.8.27"` (per TDD §2)
   - `optimizer = true`, `optimizer_runs = 200`
   - `via_ir = false` (default; can be toggled later if needed)
   - Sensible test profile (`test = "test"`, `out = "out"`, `cache_path = "cache"`)
5. `remappings.txt` includes:
   - `@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/`
   - `@openzeppelin/contracts-upgradeable/=lib/openzeppelin-contracts-upgradeable/contracts/`
   - `forge-std/=lib/forge-std/src/`
6. `.gitmodules` is committed.
7. `forge build` succeeds on the empty project.
8. PR opened: `base=main`, `head=claude/T-1.1-scaffolding`. PR title: `T-1.1 Project scaffolding`.

### Context slice (per `L1-PLAN.md` §3.1)
- TDD §2 (Tech stack & conventions)
- TDD §15.T1.1 (acceptance recap)
- L1-PLAN.md §2 (DAG context), §3 (slicing rules)

### Out of scope
- Any contract source files in `src/core/` (those are T1.2 onward)
- Any `script/` contents (T6.1)
- Any `test/` contents beyond the auto-generated `Counter.t.sol` example (which should be deleted)
- README content (a one-line README is fine; full README is a separate task)

### L2-视角 audit checklist (for B-stage `{{REVIEW_TARGET}}`)

```
PR: #<N> (filled at B-stage prompt assembly)
Task: T-1.1 Project scaffolding (Foundry init + OZ v5 + dirs)

L2 视角关注方向 (4 条):
1. forge-std 与 OpenZeppelin v5 的依赖版本是否锁定到具体 commit hash（避免后
   续不可复现）
2. 目录结构是否完全对齐 TDD §2 约定（src/core / src/libraries / src/interfaces /
   src/mocks / test / script），无多余目录
3. remappings.txt 是否就位 + foundry.toml 的 solc_version / optimizer / 路径配
   置是否合理
4. .gitmodules 是否提交、submodule 是否锁到 commit hash（不是 floating tag /
   branch）
```

---

## T-1.2 Shared types and constants (placeholder — to be authored after T1.1 merge)

**Paste-ready prompt**: `docs/prompts/stage_1/T-1.2.md` (TBD)
**Branch**: `claude/T-1.2-types-constants`
**Dependencies**: T1.1 merged

### Goal (preview)
Implement `src/libraries/PsychohistoryTypes.sol` (enums + structs from TDD §9) and `src/libraries/Constants.sol` (constants from TDD §9). No contract logic, only type definitions.

### Notable elements (from TDD V0.31 §9)
- `Bounty` struct including `PrivacyMode privacyMode` (V0.4 hook B), `resolvedAt`, `totalRawWagerAmount` / `totalEffectiveWagerAmount` / `totalFeeCollected`
- `Prediction` struct including `rawWager / effectiveWager / feeAmount` and `bytes encryptedPayload` (V0.4 hook A)
- `SettlementState` with `currentPass / passCompletedFlags / cutoffHintSubmitted / cutoffHintScore`
- New enums `PrivacyMode { Transparent, OracleEncrypted, ThresholdEncrypted }`, `RefundReasonCode { INVALIDATED, CANCELLED, NO_SIGNAL }`
- Constants: `MAX_SPONSORS_PER_BOUNTY = 100`, `K_WAD_PHASE1..7` schedule, Treasury category constants including `CAT_P2_UNALLOCATED`
- `__reservedForV04` storage gaps on `Bounty` / `Prediction` / `SettlementState`

### Acceptance criteria (preview)
- `forge build` succeeds; no warnings.
- Every struct field documented with `///` NatSpec.
- Constants file is internal/private (no public API surface) — types library is `library`-keyword, constants is `library` or `abstract contract` per Foundry idioms.

### Context slice (preview)
- TDD §3, §9, §11 (pass fields), §5.3–§5.7, §5.9
- Optional: §6.4 (K schedule rationale), §10 (interface signatures using these types)

---

## T-1.3 All interface files (placeholder — to be authored after T1.2 merge)

**Paste-ready prompt**: `docs/prompts/stage_1/T-1.3.md` (TBD)
**Branch**: `claude/T-1.3-interfaces`
**Dependencies**: T1.2 merged

### Goal (preview)
Author all six interface files from TDD §10 with full NatSpec:
- `IBountyManager.sol`
- `IPredictionEngine.sol`
- `IRewardDistributor.sol`
- `ITreasury.sol`
- `IBuybackExecutor.sol`
- `IPsychohistoryToken.sol`

### Notable elements (from TDD V0.31 §10)
- `IBountyManager`: 5 PE-only mutators (`recordPrediction / closeBounty / markResolved / markInvalidated / markSettled`); `addSponsorship` documents 100-cap; `SponsorRefundClaimed` event with `RefundReasonCode`
- `IPredictionEngine`: `submitPrediction` includes `bytes encryptedPayload`; new `submitCutoffHint`
- `IRewardDistributor`: 4-function API (`reserveRewards / assignRewards / finalizeRewards / claimTokens`)
- `ITreasury`: `CAT_P2_UNALLOCATED` category; per-category balance views; `pullBuybackForEpoch` replaces `scheduleBuyback`
- `IBuybackExecutor`: `Activated` event; `isActivated()` view

### Acceptance criteria (preview)
- `forge build` succeeds against mock implementations.
- Every external/public function has `@notice / @param / @return` NatSpec.
- Events match TDD §10 specifications exactly.
- Interfaces import only `@openzeppelin` IERC20 / IERC20Permit / IVotes (no other source dependencies).

### Context slice (preview)
- TDD §10 (all subsections), §4.2 (roles), §4.3 (interaction flow), §8 (sponsor mechanics for IBountyManager context), §11 (settlement context for IPredictionEngine), §13 (role grants)

---

## Stage 1 exit criteria

Stage 1 is complete when:

1. T1.1, T1.2, T1.3 all merged to `main` after L2 acceptance.
2. `forge build` succeeds on the integrated state.
3. L1 has signed off the per-stage acceptance report at `docs/reviews/master_plan/<日期>_STAGE_1_acceptance.md`.

After Stage 1 closes, L2-T2 (math + token, with T2.1 ‖ T2.2) begins. Stage 2 paste-ready tasks will be authored at that point.
