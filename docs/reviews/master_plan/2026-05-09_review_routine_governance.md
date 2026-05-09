# Psychohistory V0.3 — Review Routine Governance

**Status**: Active source of truth
**Effective from**: 2026-05-09
**Applies to**: L2-T1 onward (L2-T0 ran informally; see §10)
**Owner**: L1 master planning
**Predecessor**: Internal governance from prior project (V0.4.1) — patterns adapted to this project's L1/L2/L3 hierarchy.

---

## 1. Purpose

Single source of truth for how each L3 task moves from idea to merged code. Defines ABC three-stage flow, L2 acceptance, exceptions to BC, file save paths, and tested workflow blind spots.

When this document conflicts with anything else in repo, **this document wins**. Conflicts must be resolved by editing this document with explicit changelog entry, not by deviating in practice.

---

## 2. Hierarchy Recap

```
L1   Project master planning conversation
        ├── docs/L1-PLAN.md (canonical L1 deliverable)
        ├── docs/DECISIONS.md (ADRs)
        └── docs/reviews/master_plan/...governance.md (this file)

L2   Stage / module planning conversations (one per Tier in TDD §15)
        ├── L2-T0 Spec Lock — special, no L3 children, produces TDD V0.31 (informal ABC)
        ├── L2-T1 Foundations
        ├── L2-T2 Math + Token
        ├── L2-T3 Core Modules
        ├── L2-T4 PredictionEngine
        ├── L2-T5a Router
        ├── L2-T5b BuybackExecutor
        ├── L2-T6 Deploy + Integration
        └── L2-T7 Security
              └── docs/STAGE_X_TASKS.md (paste-ready L3 list, per stage)

L3   Leaf execution conversations (typed code work, one per task)
        └── docs/prompts/stage_N/T-N.X.md (paste-ready L3 starter prompts)
```

L2 may also be referred to as "L2 integration planner" when its primary work is producing STAGE_X_TASKS.md and the per-L3 paste-ready prompts. Both labels (L2 / L2 integration planner) refer to the same conversation level; the distinction is functional, not hierarchical.

---

## 3. ABC Three-Stage Flow (each L3 must complete the full loop)

### 3.1 A Stage — Development (Claude Code)

**Author**: starts a new Claude Code session.

- Working directory: `/Users/outsider/Desktop/Psychohistory-v3` (worktree spawned automatically by Claude Code on a `claude/T-N.X-<topic>` branch).
- **First message** to the new session: `请按 /docs/prompts/stage_N/T-N.X.md 的指示执行任务` (verbatim — the actual prompt body lives in the file, the starter message just points to it).
- Session work: develop + write tests + run tests + commit + push + open PR (`base=main`, `head=claude/T-N.X-<topic>`).
- Output: PR URL + final commit hash.

The author may converse with the L3 conversation for clarification, scope confirmation, or test failure debugging during A. The L3 conversation is the executor; the author is the human in the loop, not a parallel reviewer.

### 3.2 B Stage — Cross-LLM Review (Codex / GPT-5.5)

**Author**: starts a new Codex session in an **independent codex worktree** (NOT reusing the A-stage worktree).

#### Pre-flight cleanup (mandatory)

Codex switches to `main` to read the PR diff. If multiple worktrees are present, this can collide.

```bash
git worktree list
git worktree remove <临时目录>     # for any temp worktree no longer needed
```

The cleanup is needed because git refuses to check out the same branch in two worktrees. If Codex's worktree tries to enter `main` while another worktree already occupies `main`, it fails.

#### Review prompt assembly

1. Open `/docs/REVIEW_PROMPT_CODE_GPT.md`. The file contains a single ` ```text ` block — this is the canonical prompt body.
2. **Recommended source for the prompt body**: macOS Notes (plain-text mode, `Cmd+Shift+T`). Reason: chat platforms auto-linkify URLs, mangle nested ```fences, and corrupt invisible characters. The repo file is the source of truth; macOS Notes is the daily-use copy. When the repo prompt updates, manually re-copy to Notes.
3. **Find & Replace** `{{REVIEW_TARGET}}` with the L2-supplied content (PR # + L2-视角 audit checklist 4-5 lines under the literal header `L2 视角补充上下文(不替 finding;仅作 review 关注方向):`).
4. Paste the assembled prompt as the **first message** to the Codex session.

#### What Codex does after review (automated steps in the prompt body)

After completing the review report:

```bash
git checkout main && git pull origin main
git add docs/reviews/<日期>_T-N.X_<topic>_review.md
git commit -m "docs(review): T-N.X cross-LLM review report (B-phase output for PR #N)"
git push origin main
```

Output: review report at `/docs/reviews/<日期>_T-N.X_<topic>_review.md` on `main`, in its own commit. Not on the PR branch. Not just on Codex's local worktree. **Physical location on `main` is what L2 verifies in step 1 of acceptance.**

### 3.3 C Stage — Fix (Claude Code, A-stage worktree, A-stage branch)

**Author**: starts a **new Claude Code session** but pointed at the same worktree and same `claude/T-N.X-<topic>` branch as A.

- First message: `Read main 上 B 报告路径 docs/reviews/<日期>_T-N.X_<topic>_review.md` + literal text `吃 review 报告改代码`.
- Session work: process every B finding — accept 🔴 (mandatory) and 🟡 (recommended) by default; rejected 🟢 must include explicit rationale committed alongside. Append commits to the same PR (do **not** open a new PR).
- Output: C-stage commit hash(es) on the same PR.

If C reveals that B was wrong (e.g., the reviewer misread the spec), C may push back: commit the explanatory rationale and `gh pr comment` on the PR with the reasoning. L2 sees both at acceptance time.

---

## 4. L2 Acceptance

L2 verifies the loop without needing the author to paste anything. L2 uses `gh api` to fetch ground truth.

### 4.1 Five-point verification checklist

1. **Module boundary respected** — PR diff stays within the L3 task's declared scope (`docs/STAGE_X_TASKS.md` row + `T-N.X.md` prompt). No drive-by edits to unrelated modules.
2. **ABC loop is closed**:
   - PR exists (A output: `gh pr view <N> --json url,headRefName,state`)
   - B report exists at `docs/reviews/<日期>_T-N.X_<topic>_review.md` on `main` (`gh api repos/outsiderrr/Psychohistory/contents/docs/reviews?ref=main`)
   - C commit exists on the PR after B report's commit timestamp (`gh pr view <N> --json commits`)
3. **Every B finding addressed** — Each 🔴/🟡 either fixed in C (commit referenced) or rejected with rationale (commit message or PR comment). No ignored findings.
4. **Completion criteria met** — Tests pass (`gh pr checks <N>`), forge build succeeds, acceptance criteria from `docs/STAGE_X_TASKS.md` met, NatSpec on every external/public.
5. **L2 audit checklist** — The 4-5 task-specific checks L2 supplied during B-stage prompt assembly. L2 confirms each was addressed in either implementation or B-finding response.

### 4.2 L2 acceptance output

L2 writes its own report to `docs/reviews/master_plan/<日期>_T-N.X_L2_acceptance.md`. Format:

```markdown
# L2 Acceptance — T-N.X

PR: #N
A author session: <ref>
B report: <path>
C commits: <hashes>
Decision: PASS / REJECT / SECOND_B_NEEDED

## Five-point check
1. Module boundary: ✅/❌ + 1 line
2. ABC closure: ✅/❌ + 1 line
3. B finding handling: ✅/❌ + count of 🔴/🟡/🟢 disposition
4. Completion: ✅/❌ + ci status
5. L2 audit checklist: ✅/❌ + per-item status

## Action
PASS  → notify author to merge (or L2 merges if author preauthorized)
REJECT → return to C with notes
SECOND_B_NEEDED → trigger second B round (rare; only if C introduced new structural questions)
```

### 4.3 Merge

PR merge happens **after** L2 acceptance PASS. L2 may merge directly if the author has explicitly preauthorized for this task; otherwise L2 notifies and the author merges.

`--squash` is the default merge strategy (cleaner history); `--merge` (commit-preserving) for L3s where commit granularity matters for future bisect.

---

## 5. 跳 BC Exceptions — Five Default-Authorized Categories

These task types do NOT require cross-LLM review. Author fixes directly + L2 quick-check + merge.

| # | Category | Examples |
|---|---|---|
| 1 | **R-class follow-up** | Tooling / content bugs surfaced by empirical work, where the fix is mechanical |
| 2 | **Baseline batch findings** | Findings from running protocol baselines, where fix is substitution-shaped |
| 3 | **Playtest batch findings** | Findings from end-to-end simulation runs |
| 4 | **UI ergonomic adjustments** | Front-end-only copy / layout / view tweaks not touching backend schema or algorithm |
| 5 | **Stage acceptance reports** | Per-stage summary documents, written after a stage closes |

For these:

- A: author commits directly to `main` (no PR; or trivial PR self-merged) or to a feature branch with single-author signoff.
- L2: spot-checks the change (typically <5 min), greenlights.
- Documentation: a one-line note in `docs/reviews/master_plan/<日期>_<topic>_skip_bc.md` explaining which category and why.

The BC-skip is a **default-authorized** path. L2 does NOT need the author's per-task request. If a category-1-5 task is in progress, the BC-skip applies automatically. The author writes the skip note, not the L2.

---

## 6. File Save Paths

Mandatory canonical locations:

| Document type | Path | Author |
|---|---|---|
| L1 master plan | `/docs/L1-PLAN.md` | L1 conversation |
| TDD (versioned) | `/docs/TDD-V0.XX.md` | L1 / L2-T0 |
| ADRs (architecture decisions) | `/docs/DECISIONS.md` | L1 conversation; L3 amends only with L1 sign-off |
| Governance source-of-truth | `/docs/reviews/master_plan/<日期>_review_routine_governance.md` | L1 conversation |
| B-stage prompt template | `/docs/REVIEW_PROMPT_CODE_GPT.md` | L1 conversation |
| L1 paste-ready stage tasks | `/docs/STAGE_X_TASKS.md` | L2 integration planner |
| L3 paste-ready prompts | `/docs/prompts/stage_N/T-N.X.md` | L2 integration planner |
| B review reports | `/docs/reviews/<日期>_T-N.X_<topic>_review.md` (on main) | Codex B stage |
| L2 acceptance reports | `/docs/reviews/master_plan/<日期>_T-N.X_L2_acceptance.md` | L2 acceptance conversation |
| BC-skip notes | `/docs/reviews/master_plan/<日期>_<topic>_skip_bc.md` | author |

L2 conversations may write only to `/docs/reviews/master_plan/` and to their own `~/.claude/projects/...memory/` (per-conversation memory). L2 must NOT modify L1 documents (`L1-PLAN.md`, `DECISIONS.md`, `TDD-V0.XX.md`, `STAGE_X_TASKS.md`, paste-ready prompts) without L1 authorization.

---

## 7. Key Governance Rules (Tested Hard Rules)

These five rules are non-negotiable and must remain explicit in this document. Each was learned the hard way in a prior project.

### Rule 1 — B-stage review report MUST commit + push to `main` as an independent commit

Not on the PR branch. Not in a `git stash`. Not just on Codex's local worktree. The first thing L2 does is `gh api repos/outsiderrr/Psychohistory/contents/docs/reviews?ref=main` to verify physical location. If the report is anywhere else, L2 acceptance fails at step 1 with zero hit.

### Rule 2 — L2-视角 audit checklist as supplementary context (default operation)

When the author assembles the B prompt, they fill `{{REVIEW_TARGET}}` with PR # plus 4-5 L2-supplied task-specific concerns under the literal header:

> `L2 视角补充上下文(不替 finding;仅作 review 关注方向):`

These are **focus areas, not findings**. The reviewer must still produce findings independently; the L2 hints just bias attention. Explicitly NOT a "give us 5 findings on these 5 things".

### Rule 3 — B-stage commit message uses the literal template

```
docs(review): T-N.X cross-LLM review report (B-phase output for PR #N)
```

Author transcribes verbatim. Free-form Codex commit messages drift over multiple reviews and break L2's regex-based traceability. Tested in prior project.

### Rule 4 — C stage uses the same A-stage worktree and branch

C does not start from `main`. C does not open a new PR. C appends commits to the same `claude/T-N.X-<topic>` branch and the same PR.

If C cannot reuse the A worktree (e.g., it was removed mid-flow), recreate the worktree from `claude/T-N.X-<topic>` rather than starting fresh.

### Rule 5 — L2's ONLY allowed write locations

`/docs/reviews/master_plan/` plus per-conversation memory. **NOT** `L1-PLAN.md`, `DECISIONS.md`, `TDD-*.md`, `STAGE_X_TASKS.md`, or paste-ready prompts. L2 is verification + planning, never authoring of L1 source-of-truth documents.

---

## 8. Memo Strategy

To minimize transcription error in the B prompt:

1. The repo's `/docs/REVIEW_PROMPT_CODE_GPT.md` is the canonical authoritative source.
2. The author copies the ` ```text ` block from the file into **macOS Notes** in plain-text mode (`Cmd+Shift+T`) once per repo lifetime, then re-copies on prompt updates.
3. Each B-stage invocation: author copies from Notes (not from repo, not from chat), Find & Replace `{{REVIEW_TARGET}}`, paste to Codex.

Reason: chat platforms (Slack, Telegram, Discord, etc.) silently mangle the prompt — auto-linkify URLs, transform code fences, eat unicode. macOS Notes preserves verbatim.

L2 supplies only the `{{REVIEW_TARGET}}` substitution payload (PR # + audit checklist) to the author. The author does the assembly.

---

## 9. Tested Workflow Blind Spots (Prevention List)

From the prior project's empirical phase. Encoded here so this project does not re-learn.

### 9.1 Multi-worktree git checkout main fails

**Symptom**: Codex tries `git checkout main` and gets `fatal: 'main' is already checked out at <path>`.

**Cause**: Another worktree (Claude's A-stage worktree, or a stale temp worktree) still occupies `main`.

**Workaround**: Pre-flight cleanup before starting Codex (§3.2 above). `git worktree list` + `git worktree remove <临时目录>`.

### 9.2 Codex worktree reuse + chat-history pollution

**Symptom**: Same Codex session running two reviews in a row → second review's commit message picks up styling from first.

**Cause**: LLM context contamination between independent review tasks.

**Workaround**: Each L3's B-stage uses a fresh Codex session. No reuse.

### 9.3 Multi-review same-commit packaging

**Symptom**: Codex worktree has multiple untracked review reports → `git add` may package them together.

**Cause**: Codex's `git add docs/reviews/...` matches a glob.

**Workaround**: Author specifies the exact filename in the prompt's `git add` step. Severity: low (reports still traceable individually).

---

## 10. Retroactive Note: L2-T0 Ran Informally

L2-T0 Spec Lock (the task that produced TDD V0.31 from V0.30 + L1.C decisions) ran on **informal ABC** before this governance document existed. Specifically:

- A stage: TDD V0.31 produced in the L1 master conversation, not in a separate Claude Code session with worktree/branch/PR. No PR was opened.
- B stage: GPT-5.5 review prompt sent manually by the author (not via the formal `REVIEW_PROMPT_CODE_GPT.md` template). Review feedback handled inline rather than committed to `main`.
- C stage: in progress as of this document's effective date; will run in the L2-T0 dedicated conversation.

This informality is **deliberate, not a violation**. L2-T0 is a special L2 (no L3 children, document-only output) and its ABC is best mapped to "category 5 — stage acceptance report" of §5 BC-skip exceptions in spirit, even though the actual L2-T0.B is being run formally in spirit.

**Formal governance starts at L2-T1**. T1.1 Project scaffolding is the first task subject to full ABC + this document's rules.

---

## 11. Changelog

| Date | Change | Author |
|---|---|---|
| 2026-05-09 | Initial version. Captures ABC three-stage flow, L2 acceptance protocol, BC-skip 5 categories, file save paths, 5 hard rules, memo strategy, tested blind spots. Adapted from V0.4.1 prior project governance. | L1 master conversation |
