---
name: bounty-writeup
description: "Clean-context report generation from hunting session artifacts. Reads finding reports and evidence, formats for platform submission. Run this in a SEPARATE Claude session with clean context."
---

# Bug Bounty Writeup Phase

You are generating polished vulnerability reports for platform submission. This phase reads finding reports and evidence from the exploit phase and produces submission-ready writeups.

**IMPORTANT**: This skill is designed for a SEPARATE Claude session with clean context. The hunting session accumulates tool output that pollutes context. Start fresh for writeups.

## Input

The engagement slug was passed from the bounty router. The engagement directory is at:
`~/engagement-kit/engagements/<slug>/`

## Steps

### Step 1: Load Hunting Context

Read these files to understand the engagement:

- `hunting-session.md` -- full session summary, confirmed findings list
- `engagement.yaml` -- program metadata (platform, program name, contacts)
- `admin/authorization.md` -- submission requirements, researcher handle

### Step 2: Load Each Finding

For each finding directory in `findings/`:

1. Read `finding.md` -- title, severity, CVSS, CWE, steps to reproduce, impact, remediation
2. Read `evidence-index.md` -- evidence inventory with descriptions
3. View evidence files -- screenshots (via Read tool for images), HTTP captures (via Read for text)

### Step 3: Quality Checks

For each finding, verify:

| Check | Criteria | Fix If Failing |
|-------|----------|----------------|
| Steps to reproduce | Clear, numbered, reproducible by a triager with no prior context | Rewrite with explicit URLs, parameters, headers |
| Impact | Concrete and demonstrated, not theoretical | Rewrite based on actual PoC results |
| CVSS | Correct vector, justified by evidence | Recalculate with proper vector components |
| Evidence | Complete set (request, response, screenshot), properly redacted | Flag missing evidence; do NOT fabricate |
| CWE | Accurate classification for the vulnerability | Look up correct CWE from cwe.mitre.org |
| Remediation | Specific, actionable, technically accurate | Research correct fix for the tech stack |

### Step 4: Platform-Specific Formatting

Read `references/platform-templates.md` for format details.

**HackerOne**:
- Markdown format with H1 headers
- Severity dropdown: Critical/High/Medium/Low/None
- CVSS 3.1 vector string in dedicated field
- Weakness (CWE) selection
- Asset selection from program's defined assets
- Structured impact section

**Bugcrowd**:
- VRT (Vulnerability Rating Taxonomy) alignment
- Priority rating: P1-P5
- Target/URL field
- Steps to reproduce in numbered format

**Intigriti**:
- Severity: Critical/High/Medium/Low
- Impact categorization
- Domain field for affected asset
- Markdown body

### Step 5: Redaction Checklist

Before finalizing each report, verify:

- [ ] No personal access tokens in evidence or report text
- [ ] No other users' PII (mask emails, names, phone numbers)
- [ ] No internal infrastructure details beyond what's necessary for the finding
- [ ] Researcher handle is correct (`athanvi` on HackerOne)
- [ ] No session tokens or API keys visible in screenshots
- [ ] Cookie values masked in HTTP captures

### Step 6: Generate Polished Reports

One report per finding. Write to: `reporting/FINDING-NNN-<slug>-report.md`

Report structure:

```markdown
# <Title>

**Severity**: Critical/High/Medium/Low
**CVSS 3.1**: X.X (vector string)
**CWE**: CWE-XXX (<Name>)
**Affected Asset**: <URL or host>

## Summary

<1-2 sentence overview of the vulnerability and its impact>

## Steps to Reproduce

1. Navigate to `<URL>`
2. <Specific action with exact parameters>
3. <Specific action>
4. Observe <vulnerability indicator>

### HTTP Request (copy-pasteable)

```http
POST /api/v1/endpoint HTTP/1.1
Host: target.example.com
Content-Type: application/json
X-Bug-Bounty: True

{"param": "payload"}
```

## Impact

<Concrete impact statement: what an attacker can do, who is affected, data at risk>

## Evidence

1. **Screenshot**: <description> (attached: <filename>)
2. **HTTP Response**: <description> (attached: <filename>)

## Remediation

<Specific technical recommendation for fixing the vulnerability>

## References

- CWE-XXX: <link>
- Related advisories or documentation
```

### Step 7: Present for Review

Display each report to the user. For each:

1. Show the full report text
2. List attached evidence files
3. Ask: "Approve for submission? (yes / edit / skip)"

Wait for explicit approval before marking complete.

### Step 8: Mark Complete

After all reports are approved:

```bash
cd ~/engagement-kit && python3 lib/phase_state.py engagements/<slug>/ --set writeup_complete
```

Update `hunting-session.md` with:
- Reports generated count
- Reports approved count
- Total estimated bounty for approved submissions
- File pointers to all reports in `reporting/`

## Post-flight Summary

```
## Writeup Complete: <program>

- Reports generated: N
- Reports approved: N
- Total estimated bounty: $X,XXX - $X,XXX

Reports ready for submission:
1. reporting/FINDING-001-<slug>-report.md -- <title> (CVSS X.X)
2. reporting/FINDING-002-<slug>-report.md -- <title> (CVSS X.X)

Next: submit via platform UI or `/bounty submit <slug>` if available
```

## QUALITY STANDARD

Reports must be triager-friendly:
- A platform triager with NO context about the target should be able to reproduce the finding in under 5 minutes using only the report
- Every claim must be backed by evidence
- No speculation or theoretical impact -- only what was demonstrated
- Professional tone, no editorializing
