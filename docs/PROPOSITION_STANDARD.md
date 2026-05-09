# Qualified Proposition Standard (QPS)

**Source ADR**: [ADR-0014](DECISIONS.md#adr-0014--qualified-proposition-standard-qps)
**Scope**: Project-level proposition curation principle. Applies to all Psychohistory bounties (team-curated in V0.3, sponsor self-service in V0.4+).
**Audience**: Curator (team in V0.3), sponsors creating bounties (V0.4+), front-end / SDK reviewing proposed bounties.

---

## 1. Why QPS Exists

Psychohistory's design philosophy is **Forecaster Scoreboard** — a place where calibration is measured cleanly and reputation accumulates over time. This breaks down completely if proposition resolutions are subjective: an "interpretation latitude" lets the oracle (and by extension the protocol owner) tilt outcomes, undermining both fairness and the Schelling-point oracle paradigm we're moving toward.

**QPS is the proposition-design discipline that makes interpretation latitude impossible by construction.** Every proposition is structured so any honest observer, given the same data source, reaches identical resolution.

This means:
- **Owner participation in own bounties is safe** (S7 in [L1-PLAN.md](L1-PLAN.md) §4 strategic decisions) — there's nothing to interpret
- **Oracle role is reduced from "judge" to "reader"** — any actor can verify resolution
- **Future oracle decentralization is straightforward** — drop in a multisig / optimistic / Reality.eth-style oracle, the propositions are already designed for them

---

## 2. The 5 Standards

A proposition is *qualified* if and only if it satisfies all 5:

### Standard 1 — Single Authoritative Public Source

**Rule**: Resolution comes from exactly one publicly accessible data source. The source has authority over the data type (e.g., the Nasdaq exchange has authority over NASDAQ closing prices).

**Examples of qualified sources**:
- Nasdaq.com closing auction
- China National Bureau of Statistics (NBS) official press release
- US Bureau of Labor Statistics (BLS) employment data
- Lloyd's List Intelligence weekly Hormuz Strait transit report
- OpenAI's official Twitter/X account `@OpenAI` for product announcements
- Anthropic's official website blog post
- Etherscan transaction count for a specific contract
- A specific blockchain's block height at a specific timestamp
- Major exchanges (Binance, Coinbase) for crypto closing prices, *if* the proposition specifies which exchange

**Examples of unqualified sources**:
- "What people are saying on Twitter" (no authoritative source)
- "Wikipedia" (mutable, no version control by default)
- "Major news outlets" (which one? what if they disagree?)
- "Industry consensus" (subjective)
- "Reasonable interpretation" (subjective)

### Standard 2 — Source Named Explicitly in Metadata

**Rule**: The proposition's `metadataURI` JSON must include a structured `resolution` field naming the source, the URL, and the data point.

**Required schema fragment**:

```json
{
  "resolution": {
    "source": "Nasdaq.com",
    "sourceUrl": "https://www.nasdaq.com/market-activity/index/comp",
    "dataPoint": "NASDAQ Composite Index closing value",
    "publicationTrigger": "End-of-day closing auction completion"
  }
}
```

**Why**: Predictors and observers must be able to verify the resolution path before staking. An informally referenced source ("from NBS") is ambiguous and litigable.

### Standard 3 — Value Unambiguous

**Rule**: The numerical or categorical value is precisely defined. No interpretation latitude.

**Qualified value definitions**:
- ✅ "NASDAQ Composite Index closing value on 2026-12-31, rounded to 2 decimal places"
- ✅ "China NBS-published 2026 full-year GDP growth rate (annualized %), rounded to 1 decimal place"
- ✅ "Number of vessel transits through the Strait of Hormuz, week of 2026-XX-XX, per Lloyd's List Weekly Report"
- ✅ "Categorical: 'Bitcoin closing price on 2026-12-31 falls in bin: A=<$50K, B=$50K-$70K, C=$70K-$90K, D=$90K-$110K, E=>$110K'"

**Unqualified value definitions** (need rephrasing or rejection):
- ❌ "Whether AI replaced 50% of jobs" — what's "AI", what's "replaced", what's "50%"
- ❌ "Did the economy do well?" — subjective
- ❌ "Is Hormuz Strait blockaded?" — what counts as blockade?
  - **Reframe**: "Did weekly Hormuz transit count fall below threshold X per Lloyd's List?" — qualified
- ❌ "Will tensions escalate in Middle East?" — subjective
  - **Reframe**: "Did US State Department issue Level 4 advisory for [specific country] by [date]?" — qualified

### Standard 4 — Resolution Timing Unambiguous

**Rule**: The proposition specifies an exact resolution time anchor.

**Acceptable patterns**:
- "Closing value on 2026-12-31 (UTC)" (specific date)
- "First publication of [data point] by [source] in [time window]" (handles revisions)
- "As-of-2027-01-31 final published value of [data point], whichever publication comes first by source's standard schedule"
- "[event] occurring before 2026-12-31 UTC midnight, as confirmed by [source]"

**Resolution timing must specify**:
- Date (with timezone)
- Whether "first published" or "final" or "as-of-X"
- What happens if source delays its standard publication schedule

**Unqualified timing**:
- ❌ "By the end of the year" (which timezone? exact moment?)
- ❌ "When NBS releases the data" (releases when? what if delayed?)

### Standard 5 — Failure Clause Prebuilt

**Rule**: The proposition specifies what happens if the named source is unavailable, publishes differently than expected, or fails to publish by `resolutionDeadline`.

**Standard failure clauses**:
- "If [source] does not publish [data point] by [resolutionDeadline], proposition resolves as `Invalidated` and all wagers + sponsor deposits are refunded per ADR-0006"
- "If [source] publishes a value but with explicit caveats or labels it 'preliminary' rather than 'final', oracle uses the labeled value as resolution; if no value labeled by deadline, Invalidated"
- "If [source] is acquired, renamed, or discontinued before resolution, oracle attempts to use successor source; if no successor exists, Invalidated"

**Why**: Without prebuilt failure clauses, any source-publication anomaly becomes an interpretation event, breaking Standard 1 retroactively.

---

## 3. Qualified Proposition Examples (Full Schema)

### Example A: Numerical, financial

```json
{
  "title": "NASDAQ Composite Index closing value on 2026-12-31",
  "type": "Numerical",
  "predictedDecimals": 2,
  "resolution": {
    "source": "Nasdaq.com",
    "sourceUrl": "https://www.nasdaq.com/market-activity/index/comp",
    "dataPoint": "NASDAQ Composite Index closing value (USD), end-of-day closing auction",
    "publicationTrigger": "Trading day close on 2026-12-31, normally published within 30 minutes after 4:00 PM EST",
    "resolutionTime": "2026-12-31 21:30 UTC (= 2026-12-31 4:30 PM EST)",
    "failureClause": "If Nasdaq is closed on 2026-12-31 (e.g., holiday) or fails to publish closing value by 2027-01-02 EOD, proposition resolves as Invalidated."
  },
  "qualifiedBy": "QPS-v1"
}
```

### Example B: Numerical, signed (negative possible)

```json
{
  "title": "US 2026 midterm election: net change in Democratic House seats vs 2024",
  "type": "Numerical",
  "predictedDecimals": 0,
  "predictedSign": true,
  "resolution": {
    "source": "Associated Press election results",
    "sourceUrl": "https://apnews.com/projects/election-results-2026/",
    "dataPoint": "Democratic House seats final certified count after 2026 midterm minus the 2024 Democratic count of [N]; positive = Democrats gained, negative = Republicans gained",
    "publicationTrigger": "AP race calls for all 435 House districts",
    "resolutionTime": "2026-12-15 23:59 UTC (or earlier if all races called)",
    "failureClause": "If AP has not called all 435 races by 2026-12-15, proposition resolves as Invalidated. If a race is contested via legal challenge, the AP-projected winner at 2026-12-15 cutoff is used."
  },
  "qualifiedBy": "QPS-v1"
}
```

### Example C: Discrete, categorical from a numerical underlying

```json
{
  "title": "Bitcoin closing price on 2026-12-31 — which bin?",
  "type": "Discrete",
  "optionsCount": 5,
  "options": [
    "<$50,000",
    "$50,000 – $70,000 (inclusive of $50,000, exclusive of $70,000)",
    "$70,000 – $90,000",
    "$90,000 – $110,000",
    "≥$110,000"
  ],
  "resolution": {
    "source": "Coinbase BTC-USD spot",
    "sourceUrl": "https://www.coinbase.com/price/bitcoin",
    "dataPoint": "BTC-USD spot price at 2026-12-31 23:59:59 UTC, as recorded by Coinbase Pro / Advanced exchange API closing tick",
    "publicationTrigger": "End of UTC day, immediate availability via API",
    "resolutionTime": "2026-12-31 23:59:59 UTC",
    "failureClause": "If Coinbase BTC-USD market is halted at the resolution timestamp, fall back to Binance BTC-USDT 23:59:59 UTC. If both halted or unavailable, Invalidated."
  },
  "qualifiedBy": "QPS-v1"
}
```

### Example D: Geopolitical via objective metric

```json
{
  "title": "Hormuz Strait weekly vessel transit count, week of 2026-06-22",
  "type": "Discrete",
  "optionsCount": 5,
  "options": [
    "<50 transits",
    "50–100 transits",
    "100–200 transits",
    "200–300 transits",
    "≥300 transits"
  ],
  "resolution": {
    "source": "Lloyd's List Intelligence Weekly Hormuz Report",
    "sourceUrl": "https://lloydslist.maritimeintelligence.informa.com/",
    "dataPoint": "Total vessel transits Strait of Hormuz, week 2026-06-22 to 2026-06-28 inclusive",
    "publicationTrigger": "Lloyd's standard Wednesday weekly report, expected publication 2026-07-02",
    "resolutionTime": "2026-07-09 23:59 UTC (resolution deadline; publication expected 2026-07-02)",
    "failureClause": "If Lloyd's does not publish the relevant week's report by 2026-07-09, proposition resolves as Invalidated. If the report uses a different metric definition (e.g., excluding small craft), oracle uses the report's headline transit figure."
  },
  "qualifiedBy": "QPS-v1"
}
```

---

## 4. Anti-Examples (Reject These)

| Proposition (verbatim user submission) | Reason for rejection | Reframing path |
|---|---|---|
| "Will Bitcoin be doing well by end of year?" | "doing well" subjective | Reframe to numerical (closing price) or discrete bins |
| "Did GPT-5 launch this year?" | "launch" ambiguous (announce vs API GA vs paid GA) | Specify source + event ("OpenAI official Twitter announcement of GPT-5 model availability before 2026-12-31 UTC") |
| "Will there be a recession in 2026?" | "recession" has multiple definitions (NBER, technical 2-quarters, etc.) | Specify which definition + which authority ("NBER announces recession start in 2026" or "BEA reports 2 consecutive quarters of negative GDP") |
| "What will be the most popular AI model in 2026?" | "most popular" by what metric? | Reframe to specific metric ("Highest API revenue per OpenAI/Anthropic public earnings"; or unqualified) |
| "Will the war end?" | What war, what counts as ending | Specify event ("UN Security Council resolution X passes by date Y") |

---

## 5. Curator Review Checklist (V0.3 team-curated)

Before posting any proposition on chain, the curator (team / owner / sponsor) verifies:

- [ ] **S1**: Single source identified, source has authority over the data type
- [ ] **S2**: `metadataURI` JSON includes structured `resolution` block with all required fields
- [ ] **S3**: Value definition has zero interpretation latitude (test: ask 3 honest people independently → same answer)
- [ ] **S4**: Resolution timing is exact (date + timezone + first/final/as-of qualifier)
- [ ] **S5**: Failure clause covers (a) source unavailable, (b) source publishes differently, (c) source schedule slipped
- [ ] Curator can articulate the resolution procedure to a stranger in 30 seconds
- [ ] Resolution does NOT require human judgment, taste, or expertise — only data lookup

If any item fails, **reject or reframe**. Do not "ship and figure it out."

---

## 6. Edge Cases & Decisions

### 6.1 Source Revisions (e.g., GDP revisions)

GDP, employment data, and similar economic statistics are commonly revised after first publication. QPS S4 addresses this:

- Use **first published** (often called "advance estimate") for fast resolution
- Or use **as-of-[future date] final revision** for accuracy at cost of resolution delay
- The proposition MUST pick one; never leave to oracle interpretation

### 6.2 Multi-Source Disagreement

If you're tempted to use multiple sources ("BLS or Reuters reports"), don't. Pick one:

- One primary source (e.g., BLS)
- Failure clause names the fallback only if the primary is *unavailable* (e.g., BLS website hacked, BLS report not published by deadline)
- Multi-source averaging or majority vote is *not* QPS-compliant; it adds interpretation surface

### 6.3 Source Acquisition / Renaming

If the named source is acquired, renamed, or merged before resolution:

- Default failure clause: "If [source] is no longer available under this identity, oracle attempts to use successor source if unambiguous; otherwise Invalidated"
- Curator should consider source longevity when proposing long-horizon (>1 year) bounties

### 6.4 Numerical Decimals Mismatch (Predictor vs Oracle)

V0.31 §3.2: predictor specifies `predictedDecimals ≤ 18`. The oracle's `resolvedDecimals` should match the source's published precision.

QPS S3 implies: the proposition metadata pre-declares the *expected* `resolvedDecimals` so predictors know how precise to be. Post-resolution mismatch (oracle publishes more or fewer decimals than expected) requires WAD normalization, which is BrierMath's job — but the proposition framing should set expectations.

### 6.5 Truly Unqualifiable Questions

Some interesting questions cannot meet QPS:

- "Will AGI be achieved by 2030?" (definition + authority)
- "Will [country] have a coup?" (definition)
- "What will the next breakthrough technology be?" (open-ended)

**Decision**: These are out of scope for Psychohistory. The protocol is designed for *measurable* events with *single authoritative sources*. Refer enthusiasts to Metaculus / Manifold for unqualifiable speculative questions.

---

## 7. Future Versioning

This is **QPS v1**. Future revisions (v1.1, v2) may:

- Add Standard 6 for cross-chain proposition oracles
- Refine multi-source semantics (cryptographically attested aggregation)
- Add structured `metadataURI` JSON schema validation tooling

QPS revisions are recorded in [DECISIONS.md](DECISIONS.md) as ADRs superseding ADR-0014.
