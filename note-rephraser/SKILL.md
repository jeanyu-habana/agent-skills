---
name: note-rephraser
description: Use when the user asks to "rephrase", "rewrite", or "reformat" a response or message as a note. Activated by phrases like "rephrase this as a note", "rewrite my response", "clean up my draft", or when the user provides a NOTE: and RESPONSE: block.
version: 1.0.0
---

Rephrase the response provided in the input following general communication guidelines.

## Input Format

The user provides two labeled sections:
- **NOTE:** — Context about the audience, purpose, tone, or any special requirements
- **RESPONSE:** — The draft text to be rephrased

If the input is not structured this way, ask the user to clarify which part is the note and which is the response.

## Rephrasing Guidelines

Follow the principles in [references/general-comms.md](references/general-comms.md):

1. **Lead with the key point** — Put the most important information first
2. **Be brief and direct** — Remove filler, redundancy, and unnecessary hedging
3. **Use active voice** — Prefer "We decided X" over "X was decided"
4. **Match the audience** — Use the NOTE context to calibrate tone and vocabulary
5. **Preserve the intent** — Do not change the meaning of the original response
6. **Include links/citations** — Keep any URLs or references from the original

## Output

Return only the rephrased response. Do not explain the changes or add commentary.

$ARGUMENTS
