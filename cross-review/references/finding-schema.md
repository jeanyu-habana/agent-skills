# Finding Schema

Each finding MUST use this exact format. No prose outside these blocks. Each field on its own line.

```
FINDING: [Short title, max 80 characters]
SEVERITY: critical | high | medium | low | info
CONFIDENCE: [0-100]%
LOCATION: [file:line-range] or [N/A if applies to overall design]
REASONING: [1-3 sentences explaining the problem and why it matters]
FIX: [Concrete suggestion — can be a code snippet or clear description]
```

## Severity Definitions

| Level | When to use |
|-------|-------------|
| **critical** | Exploitable security flaw, data loss risk, crash in normal operation |
| **high** | Bug in realistic edge case, significant security smell, broken API contract |
| **medium** | Non-obvious logic error, missing validation, moderate code duplication |
| **low** | Style inconsistency, minor duplication, non-critical missing test |
| **info** | Observation worth noting, no action required, or confirming no issues found |

## Confidence Guidelines

| Range | When to use |
|-------|-------------|
| 90-100% | You can trace the exact code path to the bug or have high certainty |
| 75-89% | You are confident but the code context is incomplete |
| 60-74% | Probable issue; requires knowing runtime behavior or full codebase |
| 40-59% | Possible issue; depends on use context or calling conventions |
| Below 40% | Speculative; flag only if the potential impact is very high |

## Examples

**Security Example:**
```
FINDING: SQL injection via unsanitized user_id parameter
SEVERITY: critical
CONFIDENCE: 95%
LOCATION: src/db/users.py:142-145
REASONING: The user_id value from request.args is interpolated directly into
  the SQL string without parameterization. An attacker can terminate the query
  and append arbitrary SQL statements, enabling data exfiltration or deletion.
FIX: Use parameterized query: cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
```

**Logic Example:**
```
FINDING: Off-by-one in pagination offset calculation
SEVERITY: medium
CONFIDENCE: 88%
LOCATION: src/api/list.ts:67
REASONING: offset = page * pageSize skips the first page when pages are
  1-indexed (page=1 gives offset=pageSize, not 0). Users on page 1 will
  see page 2's results.
FIX: const offset = (page - 1) * pageSize;
```

**No Issues Example:**
```
FINDING: No issues found within reviewer domain
SEVERITY: info
CONFIDENCE: 100%
LOCATION: N/A
REASONING: Code reviewed thoroughly within this reviewer's domain. No issues found.
FIX: N/A
```
