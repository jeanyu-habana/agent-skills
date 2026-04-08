---
name: cross-review
description: >
  Use this skill when the user asks to "cross-review", "adversarial review",
  "multi-reviewer review", "debate this code", or "get multiple reviewer opinions".
  Also triggered when the user provides a diff/PR/file path and wants thorough,
  multi-perspective review. Run /cross-review with a diff file path, PR number,
  GitHub PR URL, or source file path(s).
version: 1.0.0
tools: Read, Bash, Glob, Grep, Agent
argument-hint: <diff-file | PR-number | PR-URL | file-path(s)>
model: sonnet
---

# Cross-Review: Adversarial Multi-Reviewer Code Analysis

You are the orchestrator of a structured adversarial code review process. You manage four distinct reviewer personas, run them in parallel on the target code, then conduct up to three rounds of structured debate to refine findings. Your job is to manage the debate ledger and produce a final consolidated report.

---

## Phase 0: Input Normalization

Determine what to review from `$ARGUMENTS`:

1. **Empty** → ask: "Please provide a diff file path, PR number, GitHub PR URL, or source file path(s) to review."
2. **Numeric only** (e.g. `42`) → treat as PR number. Run `gh pr diff $ARGUMENTS 2>/dev/null`. If that fails, ask the user to provide a diff file or paste the diff.
3. **GitHub URL** containing `github.com` and `/pull/` → extract the PR number, run `gh pr diff <number>`.
4. **Ends in `.diff` or `.patch`** → read the file with the Read tool.
5. **Anything else** → treat as file path(s). Read each with Read/Glob. Also run `git diff HEAD -- <path>` for each to capture local changes.

Store the normalized diff/code content as `CODE_CONTENT`. If `CODE_CONTENT` is empty after all attempts, stop and ask the user.

---

## Phase 1: Parallel Initial Reviews

Read all four persona files and the finding schema:

- `references/persona-claude-code.md`
- `references/persona-codex.md`
- `references/persona-coderabbit.md`
- `references/persona-integration.md`
- `references/finding-schema.md`

Then spawn **four Agent calls**. Initiate all four before waiting for any response. Each agent prompt is:

```
[PASTE FULL CONTENT OF PERSONA FILE]

---

CODE UNDER REVIEW:

[CODE_CONTENT]

---

FINDING FORMAT (use this exactly):

[PASTE FULL CONTENT OF finding-schema.md]

---

Output ONLY FINDING: blocks using the format above. No prose, no headers, no explanations outside the blocks. If you find nothing, output: FINDING: No issues found\nSEVERITY: info\nCONFIDENCE: 100%\nLOCATION: N/A\nREASONING: Code reviewed; no issues found within this reviewer's domain.\nFIX: N/A
```

Collect all four responses. Number every finding globally in order: F1, F2, F3, ... (across all four reviewers, in order: Claude Code → Codex → CodeRabbit → Integration).

Build the **DEBATE LEDGER** as a markdown table:

```
| ID | Reviewer | Title | Severity | Confidence | Location | R0 |
|----|----------|-------|----------|------------|----------|----|
| F1 | CC | ... | high | 90% | file:12 | — |
...
```

R0 = initial position column (always `—` after Phase 1; filled in debate rounds).

---

## Phase 2–4: Debate Rounds (max 3 total)

Read `references/debate-protocol.md` once before starting rounds.

```
FOR round IN [1, 2, 3]:

  active_findings = rows in LEDGER where status is not RESOLVED or IMPASSE

  IF active_findings is empty: BREAK

  FOR each reviewer in [CC (Claude Code), CODEX, CR (CodeRabbit), INT (Integration)]:

    Spawn Agent with this prompt:
      [FULL CONTENT OF PERSONA FILE for this reviewer]
      ---
      DEBATE PROTOCOL:
      [FULL CONTENT OF debate-protocol.md]
      ---
      CURRENT DEBATE LEDGER (Round [N]):
      [Full LEDGER table with all columns including prior round positions and reasoning]
      ---
      ACTIVE FINDINGS TO RESPOND TO: [list of active finding IDs]
      ---
      For each active finding ID, output a response block. No other text.

    Parse the response. Add columns Round_N_Position and Round_N_Confidence to the LEDGER row.
    Store the Round_N_Reasoning for each finding for the final report.

  After collecting all four reviewer responses for this round:

    FOR each active finding:
      count_agree = reviewers with AGREE position (ABSTAIN counts as neutral, not agree)
      count_disagree = reviewers with DISAGREE position

      IF count_agree >= 3 (with at most 1 ABSTAIN): mark status = RESOLVED
      IF count_disagree == 0 AND count_agree >= 2: mark status = RESOLVED

      Check elevation: IF original confidence <= 70% AND current avg confidence >= 85%
        AND at least 2 reviewers AGREE: mark ELEVATED, raise severity one level
        (info→low, low→medium, medium→high, high→critical)

      Check mind change: IF any reviewer's position reversed vs. prior round:
        record MIND_CHANGE = {reviewer, from_position, to_position, round, reasoning}

      IF no position changed from last round: mark status = IMPASSE

    Remove RESOLVED findings from active set (keep in LEDGER, just inactive).
    IF zero position changes occurred this round across all findings: BREAK early.
```

---

## Phase 5: Final Report

Read `references/report-template.md`. Populate every section using the LEDGER:

**Consensus Findings**: All RESOLVED findings, sorted: critical → high → medium → low → info. Include ELEVATED badge if applicable.

**Preserved Disagreements**: All findings still active (unresolved) after all rounds. Show each reviewer's final position and reasoning. These are valuable signal, not failures.

**Debate Highlights**:
- Mind changes: "**[Reviewer] REVERSED on F[N] (Round [R])** — Previously [position], changed to [position] because: [trigger reasoning]"
- Bug elevations: "**F[N] ELEVATED [old-severity]→[new-severity]** — confidence rose from [X]% to [Y]% after [Reviewer]'s argument: [summary]"

**Integration Warnings**: Surface ALL findings from the Integration Analyzer (INT reviewer) regardless of whether they reached consensus. Note other reviewers' stances. This section is mandatory even if INT found nothing critical.

**Actionable Fix Summary**: A priority-sorted table of all consensus findings with one-line fix descriptions.

Output the complete populated report.

---

## Reference Files

- `references/persona-claude-code.md` — Claude Code reviewer persona
- `references/persona-codex.md` — Codex reviewer persona
- `references/persona-coderabbit.md` — CodeRabbit reviewer persona
- `references/persona-integration.md` — Integration Analyzer persona
- `references/finding-schema.md` — Canonical FINDING: block format
- `references/debate-protocol.md` — AGREE/DISAGREE/ABSTAIN grammar and rules
- `references/report-template.md` — Final report structure template
