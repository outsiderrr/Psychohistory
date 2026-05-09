# Psychohistory V0.3 — L1 Master Plan

**Status**: L1 阶段完成（2026-05-08 → 2026-05-09 增补 CEO review 战略决策）
**Project Version**: V0.3
**TDD Reference**: [docs/TDD-V0.30.md](TDD-V0.30.md) → 升级至 [docs/TDD-V0.31.md](TDD-V0.31.md) → 待 L2-T0.D 升级至 V0.32
**Strategic decisions**: [DECISIONS.md](DECISIONS.md) 的 ADR-0001 ~ ADR-0014
**Project standards**: [PROPOSITION_STANDARD.md](PROPOSITION_STANDARD.md)（QPS）
**Deferred items**: [TODOS.md](TODOS.md)
**Audience**: L2 阶段规划对话 / L3 子任务实现对话 / 后期审计 review

---

## Project Thesis — Forecaster Scoreboard

> **Psychohistory 是一个 forecaster 的 on-chain scoreboard**——一个让校准能力（calibration）被科学测量、被长期累积、被链上验证的市场。
>
> 金融市场无法提供这件事：一笔交易的动机是多元的（投机/对冲/平仓/止损），盈亏归因含糊（运气还是技能？时点对错？），没有清晰的"我作为 forecaster 是不是真的有 alpha"信号。Metaculus / Good Judgment Open 提供 forecaster reputation 但没有 skin in the game。Polymarket 是粗粒度的 binary 市场，不奖励 calibration 精度。
>
> Psychohistory 的差异化由三条结构性选择支撑：
>
> 1. **Brier-based 校准评分**——奖励概率分布精度，不奖励"押对方向"的二元正确（参 §5.2 Numerical / Discrete scoring）
> 2. **跨 bounty 累计的 forecaster rating**（ADR-0010）——单题表现可能波动，长期 track record 是真实信号
> 3. **Top-50% 硬切**——清晰区分 calibrated forecaster 与 noise，scoreboard 有意义的前提
>
> 这套设计把目标用户群锁定为 **niche pro-sumer**（量化分析师 / forecasting 爱好者 / macro 研究员 / Metaculus 老用户群），不追求消费级大众化。GTM 节奏是 **right-fit growth**，不是 aggressive growth。
>
> 详细 thesis 来源: ADR-0010, L1 CEO Review (2026-05-09)。

---

## 0. 文档用途

这份文档是 Psychohistory V0.3 项目 L1 主规划阶段的最终交付物。它锁定：

1. 项目分级体系（L1 / L2 / L3 + 每级 ABC 阶段）
2. 开发 DAG（任务依赖图 + 并行机会）
3. 22 个 L3 叶子任务的上下文切片表
4. L1.C 阶段 ratify 的 5 项核心决策
5. L2-T0 Spec Lock 任务范围（产出 TDD V0.31 的 patch 清单）

L2 / L3 子对话开工后应**反复引用本文档**。如发现本文档与代码或 TDD 冲突，回到 L1 主对话评估影响，不要在子对话里就地修改架构决策。

---

## 1. 项目分级体系

### 1.1 任务层级

- **L1** — 项目级主规划：架构、子任务拆分、上下文切片决策。本文档归属此层级。
- **L2** — 阶段级规划：按 TDD §15 Tier 1~7 切分，每个 Tier 一个 L2 对话，负责本阶段 L3 拆分细节、跨 L3 依赖、契约测试方案。
- **L3** — 叶子执行任务：实际写代码的子对话，每个 L2 下若干 L3。

### 1.2 ABC 阶段（每个 L1 / L2 / L3 任务都默认走完）

- **A 阶段**：与 Claude Opus 4.7 讨论或实现
- **B 阶段**：Codex / GPT-5.5 review
- **C 阶段**：
  - **L1.C / L2.C**：Opus 4.7 与 GPT-5.5 辩论 2-3 轮收敛架构歧义，用户主持
  - **L3.C**：Opus 4.7 接收 GPT-5.5 review 反馈并改代码

用户可授权跳过或修改某阶段。

---

## 2. 开发 DAG

### 2.1 依赖图

```
[L1: 主规划]（本文档）
      │
      ↓
[L2-T0 Spec Lock]（特殊 L2，无 §15 Tier 对应，无 L3 子任务，仅产出 TDD V0.31）
      │
      ↓
[L2-T1 基础]               T1.1 → T1.2 → T1.3
      ↓
[L2-T2 数学+代币]          T2.1 ‖ T2.2
      ↓
[L2-T3 核心模块]           T3.1 ‖ T3.2 ‖ T3.3
      │
      ├──────────────────────────────────────┐
      ↓                                       ↓
[L2-T4 PredictionEngine]              [L2-T5b Buyback]
   T4.1 → T4.2 →                        T5.2（与 L2-T4 完全并行）
   T4.3a → T4.3b →
   T4.3c → T4.3d → T4.3e → T4.4
      │                                       
      ↓                                       
[L2-T5a Router]                       
   T5.1（依赖 T4.4）                  
      │                                       
      └─────────────────┬─────────────────────┘
                        ↓
                  [L2-T6 部署+集成]    T6.1 → T6.2 → T6.3
                        ↓
                  [L2-T7 安全]          T7.1 ‖ T7.2
```

### 2.2 关键并行机会

- **L2-T2 内**：T2.1 BrierMath ‖ T2.2 PsychohistoryToken
- **L2-T3 内**：T3.1 Treasury ‖ T3.2 RewardDistributor ‖ T3.3 BountyManager
- **跨 L2**：T5.2 BuybackExecutor 仅依赖 T3.1 + T2.2，可与整个 L2-T4 并行 — 这是降低关键路径长度最大的杠杆
- **L2-T7 内**：T7.1 Slither ‖ T7.2 Invariants

### 2.3 必须串行

- **T1.x 三任务**（T1.1 → T1.2 → T1.3）
- **L2-T4 内部所有 T4.x**（共享 storage + 状态机）
- **T6.x 三任务**（T6.1 → T6.2 → T6.3）

### 2.4 T4.3 拆分（对 §15 原始 T4.3 的修改）

原 §15.T4.3 覆盖 §11 全部 4 pass + Pool 1/2/3 三池数学，过重。L1.B 阶段拆为 5 段：

- **T4.3a**：Pass 1（score 计算 + 分页）
- **T4.3b**：Pass 2（cutoff hint 提交 + 链上分页验证）
- **T4.3c**：Pass 3 + Pass 4 累加器（聚合 + 全局 amount commitment）
- **T4.3d**：Pass 4 单户 USDC payout 分配
- **T4.3e**：终态资金转账（fee/buyback/team/dao）+ 分页 token reward 承诺

### 2.5 L2-T5 拆分

原 §15 L2-T5 包含 Router (T5.1) + BuybackExecutor (T5.2) 但两者无共同依赖。L1.B 阶段拆为：

- **L2-T5a Router**：依赖 T4.4
- **L2-T5b BuybackExecutor**：依赖 T3.1 + T2.2，与 L2-T4 完全并行

---

## 3. L3 任务清单与上下文切片

### 3.1 切片表

每个 L3 子任务接到的 TDD 切片应当"够用且仅够用"。下表是各 L3 必传 TDD 章节、选传章节、契约依赖。

| L3 子任务 | 必传 TDD 章节 | 选传章节 | 契约依赖 |
|---|---|---|---|
| **T1.1** scaffolding | §2, §15.T1.1 | §4.1 | — |
| **T1.2** types/constants | §3, §9 全部, §11 (pass 字段), §5.3–§5.7, §5.9 | §6.4 | — |
| **T1.3** interfaces | §10 全部, §4.2, §4.3, §8, §11, §13 role grants | §3 | T1.2 |
| **T2.1** BrierMath | §3, §5.2, §9 Constants | §5.3 | T1.2 |
| **T2.2** PsychohistoryToken | §6.1–§6.3, §4.2 | §6.6, §7 | T1.3 |
| **T3.1** Treasury | §5.1, §5.5(C/D1/D2), §5.7, §6.5, §10.4, §4.2, §13, §2 conventions, §12.1, §12.7 | — | T1.3 |
| **T3.2** RewardDistributor | §5.6, §6.2, §6.3, §6.4, §10.3, §11 final formula, §12.6 | — | T1.3, T2.2 |
| **T3.3** BountyManager | §3, §8 全部, §9 (Bounty/Sponsorship), §10.1, §15.T3.3, §5.8 | — | T1.3 |
| **T4.1** PE-submit | §3, §9 (Prediction), §10.2 (submit), §4.3, §10.1, §12.3, §14 submit/lifecycle 测试 | — | T3.3 |
| **T4.2** PE-resolve | §10.2 (resolve/invalid), §5.8, §12.2 | — | T4.1 |
| **T4.3a** PE-Pass1 | §5.2, §11 (Pass 1), §9 Constants | §3 | T4.2, T2.1 |
| **T4.3b** PE-Pass2 | §5.3, §11 (Pass 2), §12.5, §10.2 (含 hint 接口), §9 SettlementState | — | T4.3a |
| **T4.3c** PE-Pass3-4-accum | §5.4, §5.5, §11 (Pass 3+4), §5.7, §5.9, §14 invariants | — | T4.3b |
| **T4.3d** PE-payout | §5.4, §5.5, §11 (Pass 4 per-predictor) | — | T4.3c |
| **T4.3e** PE-finalize | §5.5(C/D1/D2), §5.6, §5.7, §11(final), §13 | — | T4.3d, T3.1, T3.2 |
| **T4.4** PE-claim | §10.2 (claim), §10.3, §12.1 | — | T4.3e |
| **T5.1** Router | §4.3, §10.1, §10.2 | — | T3.3, T4.4 |
| **T5.2** BuybackExecutor | §6.5, §10.5, §7.3, §10.4 | §6.4, §13 | T3.1, T2.2 |
| **T6.1** Deploy | §13 全部, §4.2, §15.T6.1, §7 | — | 全部前序 |
| **T6.2** Integration | §14 全部, §5 (用于不变式), §15.T6.2 | — | 全部前序 |
| **T6.3** Simulation | §15.T6.3, §7 | §13 | T6.1 + T6.2 fixtures |
| **T7.1** Static analysis | §12, §15.T7.1 | — | T1–T5 完成 |
| **T7.2** Invariant testing | §14, §11, §8 refunds, §12 | §5 | T6.2 |

### 3.2 切片硬性约束

1. **每个切片头部附说明**："如有歧义可引用 TDD 完整版（[docs/TDD-V0.31.md](TDD-V0.31.md)），但优先在切片内实现。"
2. **接口契约不可变**：T1.3 完成后，下游所有 L3 以那个版本的 .sol 接口为准。如发现需要改，**必须回 L1 主对话**评估影响，不能在子对话里就地修改。
3. **每个 L3 必须明确"出口契约"**：其代码会被哪些下游 L3 / L2 依赖（在 L3.A 阶段第一项就声明）。例如 T3.3 必须知道 T4.x 会读 `getBounty / getSponsorship / getAllSponsors` 三个函数。

### 3.3 单测拆分约定

- **T3.3 BountyManager**：单 L3 实现，但单测拆为 `BountyLifecycle.t.sol` + `SponsorMechanics.t.sol` 两份文件
- **T4.3 系列**：每个 sub-pass 独立单测文件 `PESettlement.PassN.t.sol`

---

## 4. L1.C 五项决策（2026-05-08 ratified）

经过 L1.B 三轮 Opus 4.7 / GPT-5.5 辩论后由用户拍板。完整 spec 见 TDD V0.31。

### 决策 1 — Privacy 路线

**选**：V0.3 走 B 路（透明 + sponsor 价值重定位为 analytics SLA）+ 强制留 5 条架构口子。
**否**：A2 (oracle custody)、A3 (self-reveal)、A* (threshold)、timelock 全部排除。
**Rationale**：A* 工程量 5–8+ 人月超 V0.3 范围；A2 oracle 单点信任不接受；A3 reveal gaming 复杂；timelock 根本无法实现 sponsor 分级（drand 只发全局时间签名，不能给单 sponsor 早解）。

**5 条强制架构口子（V0.4 升级时不需重写）**：

a. **submit 接口预留 `bytes encryptedPayload` 参数**，V0.3 `require(payload.length == 0)`
b. **Bounty struct 加 `PrivacyMode` enum**（Transparent / OracleEncrypted / ThresholdEncrypted），V0.3 仅允许 Transparent
c. **所有 upgradeable 合约保留 `uint256[50] private __gap`**
d. **settlement 数学走 `_getPrediction(bountyId, predictor)` 内部 helper 读取**，不直接访问 storage
e. **不实现 `getAllPredictions(bountyId)` 等批量公开预测函数**，前端只能按单条 (bountyId, predictor) 查询

**TDD V0.31 修订**：§1 / §8 sponsor 价值主张重写；§12.4 删除 privacy 安全论证。

### 决策 2 — Buyback 公式

**选**：A 改文字，接受几何衰减（每周花余额 1/12，半衰期约 8 周，12 周后烧掉约 65%）。
**否**：B 实现 tranche queue 真正 12 周线性。
**Rationale**：tranche queue 需要 per-tranche storage、active cleanup、gas 随入账增长，给 V0.3 太重。几何衰减实现极简、状态少、对产品目标（平滑买压避免一次性冲击）已经足够。

**TDD V0.31 修订**：§6.5 措辞改为 "rolling geometric smoothing mechanism, spend rate proportional to accumulated buyback reserves"。

### 决策 3 — Slice A 归属（bottom group 为空时）

**选**：进 Treasury 的 DAO sub-account，独立 category `P2_UNALLOCATED`。
**否**：进 buyback、归 fee、永久滞留。
**Rationale**：自动 buyback 会让 all-tie / single-predictor 等罕见情况扰乱代币 monetary policy。DAO 中性，未来 DAO 投票决定用途；DAO 未成立前由 admin multisig 托管。

**TDD V0.31 修订**：§5.5 加 Slice A 的去向说明；§10.4 ITreasury 加 `P2_UNALLOCATED` category。

### 决策 4 — Sponsor ranking gas 模型

**选**：A 硬 cap 每 bounty 100 个 sponsors。`addSponsorship()` 在达到 cap 时 revert。
**否**：B off-chain hint + 链上分页验证。
**Rationale**：用户产品判断 — V0.3 launch 阶段不需为爆款 bounty 优化；先简单上线，碰到瓶颈再升级。

**TDD V0.31 修订**：§9 Constants 加 `MAX_SPONSORS_PER_BOUNTY = 100`；§10.1 `addSponsorship` 加上限检查。

### 决策 5 — RewardDistributor 持币模型

**选**：B 按需 mint via MINTER_ROLE。
**否**：A 预先 mint 4 亿 PSYH 预持有。
**Rationale**：账目 `totalSupply() == 真实流通量`，攻击面只是单笔发奖额而非 4 亿库存。§4.2 已暗示此方案（明确 `MINTER_ROLE → RewardDistributor`）。

**TDD V0.31 修订**：§6.2 文字明确为按需 mint；§10.3 / §10.6 已隐含。

---

## 4.B L1 CEO Review 战略决策（2026-05-09 ratified）

L1.C 五项决策（§4 above）锁定 V0.31 spec 后，**2026-05-09 的 L1 CEO Review** 在 framing shift（从"创业 PMF 假设"修正为"passion infra + option-on-success"）下补 ratify 了 10 项额外战略决策。这些决策驱动 ADR-0008 ~ ADR-0014 + L2-T0.D spec lock + 多份新文档。

| # | 决定 | 来源 |
|---|---|---|
| **S1** | **Mode = HOLD SCOPE**（V0.4 privacy 作为 evidence-driven 可选扩张，不预先开发） | Delta CEO mode reselection |
| **S2** | **项目 thesis = "Forecaster Scoreboard"**（Brier-based on-chain calibration reputation, niche pro-sumer 用户群） | Hook #1 用户的"心理回报"论证升华 |
| **S3** | **首个 bounty 类型 = A-1 numerical**（连续值，第一性原理；比 A-2 discrete bin 更 truthful for 数值预测） | Hook #1 用户 push back |
| **S4** | **题目分布 70% A / 20% B / 10% E**（QPS 下 E 类也可早做，不必等 sponsor curate 阶段） | Hook #1 + ADR-0014 |
| **S5** | **冷启动 = $200/周 self-sponsor，约 $9K/年** | Hook #4-A |
| **S6** | **Stop signal = 反应式**（不预定 KPI；按外部信号灵活停止自资助） | Hook #4-B |
| **S7** | **Owner 不禁止参与预测**——由 QPS（ADR-0014）保证机制层面无解读空间，不需要机制限制 | Hook #4-C + ADR-0014 |
| **S8** | **Audit 路径 = AI multi-round + 静态分析（Slither/Aderyn/Mythril）+ Immunefi（PSYH 计价）+ open source + testnet 6-12 月** | Q4 audit 决策 |
| **S9** | **QPS（Qualified Proposition Standard）作为项目级 curation 原则** | ADR-0014 |
| **S10** | **Jurisdiction 策略 = phased**（testnet 期个人 + 无实体；mainnet 前 8 周启动 Cayman/BVI 基金会注册，本名露面）。Hard triggers 见 [TODOS.md](TODOS.md) | Hook #6 + Q4 jurisdiction |

每条决策的完整 rationale 和 trade-off 分析见 [DECISIONS.md](DECISIONS.md) 对应 ADR。

### S1-S10 的 spec / 文档影响汇总

| 影响层 | 体现 |
|---|---|
| Spec patches → V0.32 | ADR-0008（K(t) Brier alignment）、ADR-0009（int256）、ADR-0010（forecaster stats）、ADR-0011（TVL cap）、ADR-0012（Pausable）、ADR-0013（withdrawal time-lock） |
| 新文档（不动 spec） | [PROPOSITION_STANDARD.md](PROPOSITION_STANDARD.md)（QPS 完整定义 + 例子 + 反例）、[TODOS.md](TODOS.md)（监控项 + jurisdiction 触发条件 + V0.4 candidate） |
| L1-PLAN.md（本文件） | 新增 Project Thesis 章节 + §4.B 本节 |

---

## 5. L2-T0 Spec Lock 任务范围

**层级位置**：L2 级任务。非典型 L2——不对应 §15 的任何 Tier，没有 L3 子任务，输出物是规范文档而不是合约代码。可以理解为"在所有 §15 Tier-aligned L2 之前的 spec gate L2"。

是 L1.C 收敛后、L2-T1（T1.1 / T1.2 / T1.3）之前**必须执行的 spec patch 任务**。

- **输入**：[docs/TDD-V0.30.md](TDD-V0.30.md) + L1.C 5 项决策 + L1.B 累积 30+ 项接口/数学/状态机修订
- **输出**：[docs/TDD-V0.31.md](TDD-V0.31.md)
- **ABC 阶段**：
  - **L2-T0.A**：Opus 应用所有 patch 产出 V0.31 草稿
  - **L2-T0.B**：GPT-5.5 review patch（粒度细，QA 性质）
  - **L2-T0.C**：用户 ratify 或修订

### 5.1 TDD V0.31 必修章节清单

#### §1 / §8 — Sponsor 价值重写
- 旧："cryptographic secrecy of early signal"
- 新："analytics SLA + standardized aggregate API + timeliness/format advantage + service priority tier"

#### §3 — Proposition Types
- §3.2 Numerical 部分：修正 score direction，确保所有 settlement 一律 high-score-is-better 排序

#### §5 — 池数学
- **§5.4 改名**为 *net-of-fee principal protection*；公式基于 `effectiveWager = wager × 0.99`
- **§5.5 Slice A** 加 fallback：bottom group 为空时归 `P2_UNALLOCATED` → DAO sub-account
- **§5.6 Pool 3** 删除 `p1RemainderShareExcluded` 冗余项；amount slice 权重明确为 `pool1Payout = effectiveWager + remainderShare`
- **§5.7 Fee 模型**：`effectiveWager = wager × 0.99` 一次性预扣，所有 Pool 1 数学基于 effectiveWager
- **§5.9 段首** 修订：与 §8.4 对齐，明确 `predictorCount == 0` 时 sponsors 可 refund
- **§5 全章** 加 score = 0 fallback 规则（避免 `Σ(score × wager) = 0` 除零；按 wager 均分或归 Treasury）

#### §6 — Token Economics
- **§6.2** 文字明确 RewardDistributor 用 MINTER_ROLE 按需 mint，4 亿是 cap
- **§6.4 K(t)** 单位明确：`currentK()` 返回 WAD-scaled "PSYH per USDC"，公式 `allocation = usdcRaw × kWad / 1e6`
- **§6.4 月度封顶**：改为 per-bounty `effectiveK` 在 `reserveRewards` 时锁定，settlement 中不隐式改全局 K
- **§6.5 Buyback** 措辞改为 geometric smoothing

#### §9 — 数据结构
- `Prediction` 加 `rawWager / effectiveWager / feeAmount` 三字段
- `Prediction` 加 `bytes encryptedPayload`（V0.3 require empty）
- `Bounty` 加 `PrivacyMode privacyMode` 字段
- `Bounty` 加 explicit `closeBounty()` transition（不依赖 lazy timestamp）
- `SettlementState` 加 `currentPass / passCompletedFlags / cutoffHintState`
- `Constants` 加 `MAX_SPONSORS_PER_BOUNTY = 100`
- `Constants` 整理 K 相关常量为 WAD-scaled

#### §10 — 接口面
- **§10.1 IBountyManager**：
  - 加 PE→BountyManager role-gated mutators：`recordPrediction / closeBounty / markResolved / markSettled / markInvalidated`
  - `addSponsorship` 加 cap 100 检查
  - `claimSponsorshipRefund` predicate 显式三态：`Invalidated || Cancelled || totalPredictors == 0`
  - Refund 事件携带 `reasonCode: INVALIDATED / NO_SIGNAL / CANCELLED`
- **§10.2 IPredictionEngine**：
  - `submitPrediction` 加 `bytes encryptedPayload` 参数
  - 加 `submitCutoffHint(bountyId, cutoffScore)`（permissionless + 链上分页验证）
  - 统一 reward claim 路径
- **§10.3 IRewardDistributor 重写为 4 函数**：
  ```
  reserveRewards(bountyId, volumeUsdcRaw) → (reservedAmount, effectiveK)
  assignRewards(bountyId, predictors, amounts) // 分页
  finalizeRewards(bountyId)
  claimTokens(bountyId, predictor) → minted // 不做 cap 检查
  ```
  K 单位 = WAD-scaled PSYH per USDC，cap 在 reserve 时消耗
- **§10.4 ITreasury**：
  - 新分类 `P2_UNALLOCATED`
  - per-category 余额隔离

#### §11 — 设置算法
- **Pass 1** 加 score = 0 fallback 规则
- **Pass 2 cutoff hint** 固化为 permissionless + 链上分页验证
- **Pass 3-4** 全部基于 `effectiveWager` 而非 `rawWager`
- **Pass 4** 删 `p1RemainderShareExcluded`
- **Final finalization** 改为 commit-then-paginated（配合 §10.3 reserve/assign 模型）

#### §12 — 安全
- **§12.4 重写**：删除 "predictions are private" 安全论证；改为 "front-end opacity is convention not security boundary; on-chain transparency is accepted; sponsor value reposed in analytics service"
- **§12.5** cutoff hint 信任模型固化为 permissionless

#### §13 — 部署
- 拆 `DeployCore.s.sol`（Phase 1）+ `DeployBuyback.s.sol`（Phase 3）
- T6.1 acceptance 包含 initializer / ProxyAdmin / role grant 验证

#### §14 — 测试
- 加 sponsor ranking gas-bound 测试（达到 cap 100 + 1 应 revert）
- 加 score = 0 fallback 测试
- 加 effectiveWager / feeAmount 一致性 invariant
- 加 RewardDistributor reserve / assign / finalize 流程测试
- 加 PrivacyMode = Transparent 时 encryptedPayload 必须为 empty 测试

---

## 6. 阶段计划

```
2026-05-08  L1 阶段完成（本文档落盘）
   ↓
L2-T0 Spec Lock        → 产出 docs/TDD-V0.31.md（特殊 L2，无 L3 子任务）
   ↓
L2-T1（基础）          → T1.1 → T1.2 → T1.3
   ↓
L2-T2（数学+代币）     → T2.1 ‖ T2.2
   ↓
L2-T3（核心模块）      → T3.1 ‖ T3.2 ‖ T3.3
   ↓
L2-T4（PredictionEngine）  ┐  并行：
   T4.1 → T4.2 →           │  L2-T5b BuybackExecutor (T5.2)
   T4.3a → T4.3b →         │
   T4.3c → T4.3d →         │
   T4.3e → T4.4            │
   ↓                       ↓
L2-T5a Router (T5.1)
   ↓
L2-T6（部署+集成）     → T6.1 → T6.2 → T6.3
   ↓
L2-T7（安全）           → T7.1 ‖ T7.2
   ↓
V0.3 Launch
   ↓
（V0.4 升级路径：privacy A* 等）
```

---

## 7. 附录：L1.B 辩论历史摘要

L1.B 阶段 GPT-5.5 共提交 3 轮 review，与 Opus 4.7 在以下几条上达成收敛：

### Round 1（GPT-5.5 主动 review）
GPT-5.5 提出 10 条 TDD 内部歧义 + 漏掉的关键议题（含 §12.4 prediction privacy 错误假设、PE→BountyManager 接口缺口、numerical score direction、§5.7/§5.4 fee 与本金保护冲突等），并对 22 条 L3 切片提出"必传/选传"升级建议。

### Round 2（GPT-5.5 反驳 Opus 的 5 点 pushback）
- T3.3 拆分：接受 Opus 反提议，保持单 L3，单测拆双文件
- T4.0 新增：接受 Opus 反提议，不新增任务，把 storage freeze 放进 T4.1 acceptance
- NoSignal 新状态：接受 Opus 反提议，不加新状态，refund predicate 显式
- Fee 模型：接受 Opus `effectiveWager` 模型，但要求改名为 *net-of-fee principal protection*
- 提供 RewardDistributor `reserve/assign/finalize/claim` 四函数 API 设计

### Round 3（聚焦最后 2 条）
- Privacy A 路具体形态：GPT-5.5 量化 4 条路线（A2 / A3 / A* / B）的 liveness、信任、工程量；最终与 Opus 一致建议 V0.3 走 B 路 + V0.4 升级 A*
- Buyback 公式：与 Opus 一致建议 A（改文字接受几何衰减）

L1.C 阶段用户对 5 项最终决策逐项 ratify（含决策 4 sponsor cap = 100 这条用户产品判断）。

---

**End of L1-PLAN**

下一步：L2-T0 Spec Lock（特殊 L2，无 L3 子任务），产出 docs/TDD-V0.31.md。
