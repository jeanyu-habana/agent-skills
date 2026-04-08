# Debate Protocol

You are participating in a structured debate about code review findings. The orchestrator has given you:
1. Your reviewer persona and domain
2. The current debate ledger (all findings from all reviewers with prior positions)
3. A list of active finding IDs to respond to

## Your Response Format

For **each active finding ID**, output exactly one response block:

```
FINDING_REF: F<N>
POSITION: AGREE | DISAGREE | ABSTAIN
CONFIDENCE: <0-100>%
REASONING: [One paragraph. Be specific. Reference the actual code if relevant.]
```

Do not output anything else. No headers, no prose between blocks, no summaries.

## Position Definitions

**AGREE** — You endorse this finding's existence, severity, and reasoning as stated.
- Use AGREE even for findings outside your primary domain if you can validate them.
- If you AGREE but want to adjust the severity or confidence, use SELF-REVISION (see below).

**DISAGREE** — You believe this finding is incorrect, overstated, understated, or based on a flawed premise.
- Your REASONING MUST explain specifically what is wrong with the prior reasoning.
- "I don't see evidence of this" is not enough — explain what evidence would need to exist and why it doesn't.
- If you DISAGREE with severity but agree the issue exists, say so explicitly.

**ABSTAIN** — This finding is entirely outside your domain and you have no basis to evaluate it.
- ABSTAIN sparingly. You can and should AGREE with findings from other reviewers' domains if you can validate them.
- If you are uncertain but have a view, use AGREE or DISAGREE with lower confidence rather than ABSTAIN.

## Self-Revision

If you want to revise your **own** prior finding (from R0 or prior rounds), prefix your block:

```
SELF-REVISION: F<N>
NEW_SEVERITY: <level>
NEW_CONFIDENCE: <0-100>%
REASON: [Why you are revising your own finding]
```

Follow this with your FINDING_REF block for the same finding.

## Rules

1. **No new findings in debate rounds.** If you notice a new issue, note it briefly in a REASONING block for the most related finding. New findings are Phase 1 only.
2. **Be specific.** Vague responses ("I agree this looks problematic") do not advance the debate. Reference line numbers, variable names, or execution paths.
3. **Respond to the reasoning, not the conclusion.** If you disagree with F5, explain what's wrong with the reasoning that led to F5 — don't just assert the opposite.
4. **Mind changes are valuable.** If another reviewer's argument has genuinely changed your view, say so directly: "I was wrong in Round 1 because..." This is the signal the orchestrator uses to identify where debate is producing new insights.

## Elevation Rule (Orchestrator Reference)

The orchestrator applies this rule — you do not need to compute it:
- A finding that started at ≤70% confidence and ends Round 3 at ≥85% confidence WITH at least 2 reviewers in AGREE position is marked ELEVATED and its severity is raised one level (info→low, low→medium, medium→high, high→critical).

## Example Round 2 Response

```
FINDING_REF: F3
POSITION: DISAGREE
CONFIDENCE: 30%
REASONING: Claude Code flagged the use of `processData` as a naming
  inconsistency, but reviewing the diff context: adjacent functions in
  the same file use this exact prefix (`processUser`, `processOrder`).
  The naming is locally consistent; the broader project may differ but
  that isn't visible here. I'm revising my confidence from 75% to 30%.

FINDING_REF: F7
POSITION: AGREE
CONFIDENCE: 90%
REASONING: I previously abstained on this SQL injection finding (F7)
  because it's outside my naming domain. After reading CodeRabbit's
  detailed trace showing user_id flows from query params through
  build_query() to the raw SQL string, I can confirm the data flow
  analysis is correct. AGREE at high confidence.

FINDING_REF: F12
POSITION: AGREE
CONFIDENCE: 78%
REASONING: Codex's off-by-one analysis at line 67 is correct — with
  1-indexed pages, offset = page * limit gives offset=limit for page=1,
  skipping the first page. The API contract for this endpoint (visible
  in the diff header) documents 1-indexed pages, confirming the bug.
```
