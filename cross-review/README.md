# cross-review

Adversarial multi-reviewer code analysis with structured debate. Four specialized reviewer personas review code in parallel, then debate their findings over up to three rounds, producing a consolidated report with consensus findings, preserved disagreements, and mandatory integration warnings.

## File Structure

```
cross-review/
  SKILL.md                         ← orchestrator brain
  references/
    persona-claude-code.md         ← architecture & naming reviewer
    persona-codex.md               ← logic & edge-case reviewer
    persona-coderabbit.md          ← security & robustness reviewer
    persona-integration.md         ← API contracts & cross-cutting reviewer
    finding-schema.md              ← canonical FINDING: block format
    debate-protocol.md             ← AGREE/DISAGREE/ABSTAIN rules + elevation
    report-template.md             ← final report structure
```

## How It Works

1. **Input** — accepts a PR number, GitHub URL, `.diff` file, or source file path(s)
2. **Phase 1** — 4 reviewer sub-agents run in parallel, each with a distinct domain focus and bias, producing globally numbered findings (F1, F2, ...)
3. **Debate rounds (up to 3)** — sequential rounds where each reviewer reads the full ledger and responds per-finding with `AGREE`/`DISAGREE`/`ABSTAIN` + confidence + reasoning. Only unresolved findings re-enter each round
4. **Automatic detection** of mind changes (position reversals) and bug elevations (≤70% confidence findings validated to ≥85% by 2+ reviewers get severity upgraded)
5. **Final report** — consensus findings sorted by severity, preserved disagreements, debate highlights, and integration warnings always surfaced regardless of other reviewers' consensus

## Reviewers

| Persona | Domain |
|---------|--------|
| **Claude Code** | Naming conventions, architectural consistency, code duplication, structural debt |
| **Codex** | Logic correctness, off-by-one errors, edge cases, type coercions, concurrency |
| **CodeRabbit** | Injection vulnerabilities, auth bypass, crypto misuse, error handling robustness |
| **Integration Analyzer** | API contract changes, cross-cutting regressions, test coverage gaps, DB migration risks |

## Usage

```
/cross-review 42                          # PR number
/cross-review https://github.com/org/repo/pull/42
/cross-review path/to/changes.diff
/cross-review src/auth/middleware.py
```

## Report Structure

The final report contains:

- **Executive Summary** — overall risk level and verdict (BLOCKED / NEEDS WORK / CONDITIONAL APPROVAL / APPROVED)
- **Consensus Findings** — findings where 3+ reviewers agreed, sorted critical → high → medium → low → info
- **Preserved Disagreements** — unresolved findings after all rounds; represent genuine ambiguity worth escalating
- **Debate Highlights** — mind changes and bug elevations produced by the debate process
- **Integration Warnings** — all Integration Analyzer findings, surfaced unconditionally
- **Actionable Fix Summary** — priority-ordered fix table for the developer

---

## Sample Run

```
❯ /cross-review https://github.com/run-llama/llama_index/pull/21340
```

**PR:** run-llama/llama_index#21340 — Falsy-value guard fixes in MMR embeddings and fusion retriever

### Phase 1: Parallel Reviews

Four reviewers run simultaneously and produce 11 globally numbered findings:

| ID  | Reviewer | Title                                                              | Severity | Confidence | Location                    |
|-----|----------|--------------------------------------------------------------------|----------|------------|-----------------------------|
| F1  | CC       | Inconsistent `or` vs `is not None` style within same diff         | low      | 85%        | fusion_retriever.py:~186    |
| F2  | CC       | `similarity_fn or default_similarity_fn` not updated to `is not None` | low  | 75%        | embedding_utils.py:~118     |
| F3  | CC       | `existing_score = score or 0.0` unchanged context line            | info     | 70%        | fusion_retriever.py:~194    |
| F4  | CODEX    | `mmr_threshold=0` fix is correct (positive confirmation)          | info     | 100%       | embedding_utils.py:118      |
| F5  | CODEX    | `score or 0.0` silently masks None, produces wrong normalization  | high     | 92%        | fusion_retriever.py:186-188 |
| F6  | CODEX    | `existing_score = score or 0.0` (unchanged) has same falsy flaw  | medium   | 88%        | fusion_retriever.py:191     |
| F7  | CR       | Score corruption: `score or 0.0` conflates None and 0.0          | high     | 85%        | fusion_retriever.py:186     |
| F8  | CR       | Downstream `or` patterns remain inconsistent post-fix            | medium   | 72%        | fusion_retriever.py:190     |
| F9  | INT      | `threshold=0` is now a silent breaking change for callers passing 0 | high  | 82%        | embedding_utils.py:118      |
| F10 | INT      | None→0.0 substitution mid-formula may push score out of [0,1]   | medium   | 75%        | fusion_retriever.py:186     |
| F11 | INT      | No regression test for `mmr_threshold=0` boundary               | low      | 78%        | embedding_utils.py:118      |

### Debate Round 1

All four reviewers respond to every active finding simultaneously:

| ID  | CC           | Codex        | CR           | INT          | AGREEs | DISAGREEs | Status   |
|-----|--------------|--------------|--------------|--------------|--------|-----------|----------|
| F1  | DISAGREE 70% | DISAGREE 70% | DISAGREE 60% | DISAGREE 60% | 0      | 4         | active   |
| F2  | AGREE 65%    | DISAGREE 72% | DISAGREE 65% | DISAGREE 70% | 1      | 3         | active   |
| F3  | AGREE 72%    | AGREE 80%    | AGREE 72%    | AGREE 72%    | 4      | 0         | RESOLVED |
| F4  | AGREE 100%   | AGREE 100%   | AGREE 100%   | AGREE 100%   | 4      | 0         | RESOLVED |
| F5  | AGREE 90%    | AGREE 90%    | AGREE 88%    | AGREE 88%    | 4      | 0         | RESOLVED |
| F6  | AGREE 85%    | AGREE 85%    | AGREE 82%    | AGREE 82%    | 4      | 0         | RESOLVED |
| F7  | AGREE 88%    | AGREE 88%    | AGREE 90%    | AGREE 85%    | 4      | 0         | RESOLVED |
| F8  | AGREE 75%    | AGREE 75%    | AGREE 70%    | AGREE 68%    | 4      | 0         | RESOLVED |
| F9  | DISAGREE 80% | DISAGREE 85% | DISAGREE 80% | DISAGREE 75% | 0      | 4         | active   |
| F10 | AGREE 78%    | AGREE 78%    | AGREE 70%    | AGREE 70%    | 4      | 0         | RESOLVED |
| F11 | AGREE 82%    | AGREE 82%    | AGREE 82%    | AGREE 78%    | 4      | 0         | RESOLVED |

**Mind changes detected:**
- CC REVERSED on F1 (own finding) — DISAGREE 70%
- INT REVERSED on F9 (own finding) — DISAGREE 75%

Active after round 1: F1, F2, F9 → running Round 2.

### Debate Round 2

| ID  | CC-R2        | Codex-R2     | CR-R2        | INT-R2       | Change from R1? | Status  |
|-----|--------------|--------------|--------------|--------------|-----------------|---------|
| F1  | DISAGREE 78% | DISAGREE 80% | DISAGREE 75% | DISAGREE 72% | No              | IMPASSE |
| F2  | AGREE 60%    | DISAGREE 74% | DISAGREE 68% | DISAGREE 68% | No              | IMPASSE |
| F9  | DISAGREE 88% | DISAGREE 88% | DISAGREE 85% | DISAGREE 90% | No              | IMPASSE |

Zero position changes → early break. Generating final report.

### Final Report

**Reviewed:** run-llama/llama_index#21340
**Date:** 2026-04-08
**Rounds completed:** 2 of 3
**Total findings:** 11 (8 consensus | 3 impasse | 3 integration warnings)

**Verdict: NEEDS WORK**

> The PR correctly fixes one falsy-value bug (`mmr_threshold or 0.5` → `is not None`) but simultaneously introduces a new falsy-value bug in the same style (`node_with_score.score or 0.0`) in `fusion_retriever.py`. All four reviewers reached unanimous consensus that the normalization formula fix is semantically incomplete: substituting `None` with `0.0` mid-formula silently corrupts ranking scores for unscored nodes. Two "breaking change" and "style inconsistency" concerns raised by their original authors were both self-reversed during debate.

#### Consensus Findings

| #   | Severity | Title                                                              | Location                    | Confidence |
|-----|----------|--------------------------------------------------------------------|-----------------------------|------------|
| F5  | HIGH     | `score or 0.0` silently masks None, distorts normalization        | fusion_retriever.py:186-188 | 92%        |
| F7  | HIGH     | Score corruption: `or 0.0` conflates None and legitimate 0.0     | fusion_retriever.py:186     | 90%        |
| F6  | MEDIUM   | `existing_score = score or 0.0` unchanged, same falsy flaw       | fusion_retriever.py:191     | 88%        |
| F8  | MEDIUM   | Downstream `or` patterns remain inconsistent post-fix            | fusion_retriever.py:190     | 75%        |
| F10 | MEDIUM   | None→0.0 substitution may push score outside [0, 1]             | fusion_retriever.py:186     | 78%        |
| F11 | LOW      | No regression test for `mmr_threshold=0` boundary after fix     | embedding_utils.py:118      | 82%        |
| F3  | INFO     | `existing_score = score or 0.0` context line noted for cleanup  | fusion_retriever.py:~194    | 80%        |
| F4  | INFO     | `mmr_threshold=0` fix is correct (positive confirmation)         | embedding_utils.py:118      | 100%       |

#### Preserved Disagreements (3 findings, all effectively withdrawn)

- **F1** — CC self-reversed: `similarity_fn` is a callable, so the two idioms guard semantically different value types. IMPASSE with zero supporting votes.
- **F2** — 1 AGREE (CC, 60%) vs 3 DISAGREE. Non-blocking style note; all dissenting reviewers acknowledged it could be a cosmetic follow-up.
- **F9** — INT self-reversed: the old `mmr_threshold or 0.5` was already broken for callers passing `0`. The fix restores correct semantics; no regression exists. IMPASSE with zero supporting votes.

#### Debate Highlights

**CC REVERSED on F1 — Round 1:** Accepted that `similarity_fn` is a callable where falsy-but-non-None values are not a realistic concern. Effectively withdrew own finding.

**INT REVERSED on F9 — Round 1:** Recognized that the old code was already breaking callers who passed `mmr_threshold=0` by silently substituting `0.5`. The "breaking change" concern was backwards. Prevented a false HIGH-severity finding from reaching the report.

#### Actionable Fix Summary

| Priority | Finding | Severity | Location | One-line Fix |
|----------|---------|----------|----------|--------------|
| 1 | F5/F7 — `score or 0.0` distorts normalization | HIGH | fusion_retriever.py:186-188 | Replace `(score or 0.0)` with `(score if score is not None else 0.0)` |
| 2 | F6 — `existing_score = score or 0.0` unchanged | MEDIUM | fusion_retriever.py:191 | Apply same `is not None` guard |
| 3 | F8 — Inconsistent `or` vs `is not None` post-fix | MEDIUM | fusion_retriever.py:190-191 | Apply `is not None` throughout `_relative_score_fusion` |
| 4 | F10 — None substitution yields out-of-range score | MEDIUM | fusion_retriever.py:186 | Add `max(0.0, min(1.0, normalized))` clamp, or guard None upstream |
| 5 | F11 — No regression test for `mmr_threshold=0` | LOW | embedding_utils.py:118 | Add unit tests for `mmr_threshold=0` and `mmr_threshold=None` |
