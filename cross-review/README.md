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
