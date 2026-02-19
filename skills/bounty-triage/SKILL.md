---
name: bounty-triage
description: "Intelligent finding triage: deduplication, false-positive filtering, severity scoring, attack chain discovery. Uses triage_engine.py for pre-processing. Runs after scan phase."
---

# Bug Bounty Triage Phase

You are triaging scan results for an authorized bug bounty engagement. This phase runs after scanning and produces a ranked list of findings for exploitation.

## Input

The engagement slug was passed from the bounty router. The engagement directory is at:
`~/engagement-kit/engagements/<slug>/`

## Pre-flight Checks

### 1. Phase Prerequisite

```bash
cd ~/engagement-kit && python3 lib/phase_state.py engagements/<slug>/ --check scan_complete
```

If exit code != 0, STOP. Inform user to run `/bounty scan <slug>` first.

### 2. Verify Scan Output

Check that scan output exists in `recon/scans/`. If the directory is empty or missing, STOP.

## Steps

### Step 1: Pre-process Scan Output

**CRITICAL** -- raw scan output is too large for context. Pre-process first:

```bash
cd ~/engagement-kit && python3 lib/triage_engine.py engagements/<slug>/
```

This generates `recon/triage-summary.md` (max 200 lines) from raw nuclei/nikto/ffuf output.

If `triage_engine.py` is not available, manually extract key findings from scan files, but limit to the top 50 unique findings by severity.

### Step 2: Read Triage Summary

Read `recon/triage-summary.md` (NOT raw scan files).

### Step 3: AI Analysis

For each finding in the summary:

- **Exploitability**: Assess real-world exploitability, not just template match
- **Tech consistency**: Check if the technology stack (from `recon/tech-stack.md`) supports the vulnerability
- **Corroboration**: Look for corroborating evidence from multiple tools
- **Attack chains**: Identify potential chains (e.g., open redirect + SSRF, info disclosure + auth bypass)

### Step 4: Cross-reference Exclusions

- Read `out_of_scope.txt` for vulnerability class exclusions
- Remove findings matching excluded types (e.g., DoS, clickjacking, self-XSS)
- Log removed findings with reason

### Step 5: Score Findings

For each remaining finding:

- **Confidence** (0-100): tool agreement, response analysis, tech consistency
- **Severity**: CVSS 3.1 base score
- **Expected value**: confidence x severity x asset tier weight
- Tier 1 assets (primary domains, e.g., `*.robinhood.com`) get 3x weight
- Tier 2 assets (secondary services) get 2x weight
- Tier 3 assets (low-value, informational) get 1x weight

Read `references/triage-criteria.md` for the full scoring rubric.
Read `references/false-positive-patterns.md` for known FP patterns.

### Step 6: Write Triage Report

Write to `recon/triage-report.md`:

```markdown
# Triage Report -- <Program Name>

## Summary
- Raw findings: N
- After dedup: N
- After filtering: N
- Recommended for exploitation: N

## Confirmed (exploit next)
| # | Title | Severity | Confidence | Target | CWE | Est. Value |
|---|-------|----------|------------|--------|-----|------------|
| 1 | ...   | ...      | ...        | ...    | ... | ...        |

## Probable (verify manually)
| # | Title | Severity | Confidence | Target | CWE | Est. Value |
|---|-------|----------|------------|--------|-----|------------|

## Possible (low priority)
| # | Title | Severity | Confidence | Target | CWE | Est. Value |
|---|-------|----------|------------|--------|-----|------------|

## Discarded (with reasoning)
| # | Title | Reason |
|---|-------|--------|
```

### Step 7: Update Hunting Session

Update `hunting-session.md` triage section with the decisions table:
- Findings triaged count
- Confirmed / probable / possible / discarded breakdown
- Top 5 findings by expected value
- Identified attack chains

### Step 8: Set Phase Complete

```bash
cd ~/engagement-kit && python3 lib/phase_state.py engagements/<slug>/ --set triage_complete
```

## Post-flight Summary

```
## Triage Complete: <program>

- Raw findings: N
- After dedup: N
- After filtering: N
- Confirmed for exploit: N
- Top finding: <title> (CVSS X.X, confidence N%)

Next: run `/bounty exploit <slug>` or `/bounty <slug>`
```

## SAFETY INVARIANT

Cross-reference every finding against scope before recommending exploitation:

```bash
cd ~/engagement-kit && python3 lib/scope_gate.py engagements/<slug>/ --check <TARGET>
```

Do NOT recommend exploiting out-of-scope targets, even if scan tools found them.
