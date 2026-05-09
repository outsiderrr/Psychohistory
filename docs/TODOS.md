# TODOS — Deferred Items & Future Triggers

**Scope**: Items that are not blockers for any current task but must not be lost. Curated by L1 / L2 strategic discussions; updated as new deferred items surface.

**Index conventions**:
- `[V0.3]` = address before V0.3 mainnet launch
- `[V0.4]` = address as part of V0.4 privacy / decentralization upgrade
- `[Launch]` = address at protocol launch event
- `[Trigger]` = listed condition triggers immediate action; do not wait for "later"
- `[Continuous]` = monitoring item; re-evaluate periodically

---

## V0.3 Pre-Launch — Operational

### `[V0.3]` MAX_OPTIONS bump
Spec constant `MAX_OPTIONS = 5` may be too tight for complex categorical bounties (e.g., 6+ Hormuz scenario bins). Recommended bump to 8 or 10. Single line spec patch in next spec lock cycle. Owner: L2-T0.E or L2-T1.2 implementation.

### `[V0.3]` First bounty topic finalization
Strategic decision S3: A-1 numerical first bounty. Specific topic + date + source still TBD before launch. Candidates: BTC closing price, NASDAQ closing, GPT-5 launch event with specific source.

### `[V0.3]` Audit pipeline operationalization (per S8)
- Enumerate AI multi-round prompts (6 perspectives × 4 models per ADR audit-strategy notes)
- Configure Slither + Aderyn + Mythril + Echidna in CI
- Foundry invariant test framework setup (T7.2)
- Immunefi bug bounty pool sizing (PSYH-denominated, ~5% supply earmark)
- Disclosure / responsible-disclosure email or PGP key
- Owner: L2-T7

### `[V0.3]` Open source readiness checklist
- README clearly states "AI-multi-round audited; not professionally audited"
- Vulnerability disclosure policy
- License (already MIT per TDD §2)
- CONTRIBUTING.md
- SECURITY.md with disclosure path
- Owner: L2-T6 / L2-T7

---

## V0.3 Pre-Launch — Strategic

### `[Trigger]` Jurisdiction / entity registration triggers
Per S10 (phased entity strategy: testnet personal mode → mainnet 8-week-prior Cayman/BVI foundation registration). Hard triggers requiring **immediate** start of foundation registration:

| # | Trigger | Window |
|---|---|---|
| **T1** | First external sponsor inquires about invoice / contract / KYC owner | Immediate |
| **T2** | Mainnet deployment scheduled within 8 weeks | Counted backwards |
| **T3** | PSYH transfer enable scheduled within 8 weeks (Phase 2 → Phase 3) | Counted backwards |
| **T4** | Any regulatory inquiry (CFTC, SEC, MiCA, etc.) | Immediate (already late) |
| **T5** | Total Treasury balance > $10K | Soft trigger |
| **T6** | Total predictor count > 100 | Soft trigger |
| **T7** | Launch + 6 months elapsed | Soft trigger |

Cost anchor: Cayman/BVI foundation registration ~$10-15K + annual maintenance $3-10K. Path: Cayman foundation > BVI > Wyoming DAO LLC > Liechtenstein Token Act foundation.

### `[V0.3]` Legal disclaimer / Terms of Service
Public-facing notice that protocol operates under personal mode (until S10 entity registration) and what users agree to. Owner: L2-T6 / L2-T7.

---

## V0.4 Candidates — Demand-Driven

### `[V0.4]` `[Trigger]` Privacy upgrade evidence-driven monitoring
Per S1 (V0.4 privacy hooks pre-staged). Monitor at launch:

- Is sponsor demand for "encrypted bounty" mode arising? (Hedge fund / 画像 A type sponsors asking about confidentiality)
- Is "competitor monitoring sponsor activity" a real pain point in observed data?

If yes → promote V0.4 A2-oracle-encrypted privacy mode (1.5-3 person-month effort) to active development. If no → defer further.

### `[V0.4]` Multi-winner Discrete propositions (K-of-N selection)
Per Hook #1 user feedback: events like "California's 2 House seats from 5 candidates" require multi-winner Discrete. V0.31 only supports 1-of-N. V0.4 candidate.

Workaround in V0.3: split as multiple binary propositions ("Candidate X wins one of the 2 seats? Yes/No").

### `[V0.4]` A* threshold-encryption privacy mode
Long-term V0.4 candidate per ADR-0001. Engineering ~5-8+ person-months including DKG ceremony. Replaces A2-oracle-encrypted if demand justifies.

### `[V0.4]` Cutoff hint griefing refundable bond
Per ADR-0007 / current "no bond" stance. If V0.3 production observes griefing attacks (attacker spam-submitting wrong cutoff hints), introduce refundable bond with slashing for verification failure.

### `[V0.4]` Sponsor self-service proposition creation
V0.3 = team curated. V0.4 = enable sponsor to create their own propositions. Requires:
- Frontend / curator review tool that enforces QPS (ADR-0014)
- KYC pathway for high-value sponsors
- Per-sponsor proposition rate limits
- Quality vouching / staking mechanism

---

## Continuous / Monitoring

### `[Continuous]` Revere flywheel ("hot then cold") soft signal
Per Q6 soft version. No commercial flywheel pressure but if active forecaster count drops > X% over Y weeks after a peak, evaluate community communication / cool-down narrative. Owner: protocol owner (you).

### `[Continuous]` Self-sponsor stop signal (S6 reactive)
Per S5 / S6: $200/week self-funded sponsor. No predefined stop signal — react to:
- "No external traction" → reduce frequency / pause
- "External sponsor relays" → pause as relay completes
- "Self-running predictor activity without sponsor pool" → consider reduction

Owner: protocol owner (you).

### `[Continuous]` Forecaster rating drift
Per ADR-0010: cumulative forecaster stats on chain. Watch for suspicious rating accumulation patterns (sybil farming, score manipulation). If observed, may need rating system v2.

### `[Continuous]` Oracle response latency
Per S8 audit substitute. Oracle (you) response time on `resolve()` calls is part of the protocol SLA — long latency = "proto-abandonment" signal to users. No specific threshold yet; monitor manually.

---

## Out-of-Spec — Future Sub-projects

These are valid initiatives but explicitly outside V0.3 / V0.4 spec:

- DEX venue choice for buyback (Uniswap V3 vs CoW Protocol vs both) — L2-T5b implementation decision
- Cross-chain strategy (LayerZero vs Wormhole vs single-chain) — TDD §16 deferred
- Front-end / wallet integration (Privy vs WalletConnect-native vs other) — TDD §16 deferred
- KYC framework for high-value sponsors — TDD §16 deferred
- Sponsor data delivery infrastructure (off-chain analytics service) — TDD §16 deferred; delivery contracts, SLA dashboard, etc.
- Subgraph / indexer for protocol-wide leaderboard — feeds front-end leaderboard from on-chain `forecasterStats`
- DAO formation (Phase 4) — TDD §7.4 deferred

---

## Resolved (moved out of TODOS)

This section captures items previously deferred but now resolved by ADR or strategic decision. Kept for archaeological reference.

| Date | Item | Resolution |
|---|---|---|
| 2026-05-09 | int256 negative numerical predictions | ADR-0009 |
| 2026-05-09 | K(t) mining vs Brier alignment | ADR-0008 |
| 2026-05-09 | Long-term forecaster rating | ADR-0010 (the "Scoreboard thesis on chain") |
| 2026-05-09 | Launch-time TVL cap | ADR-0011 |
| 2026-05-09 | Emergency pause switch | ADR-0012 |
| 2026-05-09 | Launch-period withdrawal time-lock | ADR-0013 |
| 2026-05-09 | Qualified Proposition Standard | ADR-0014 + `docs/PROPOSITION_STANDARD.md` |

---

**Maintenance**: When a new strategic / engineering decision surfaces an item that's "important but not urgent," add it here with proper category. When an item is resolved, move to the "Resolved" table with date and pointer.
