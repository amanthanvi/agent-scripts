# Hunting Session Artifact Schema

The `hunting-session.md` file is the cross-session handoff document. Each phase updates it.
The writeup skill reads it in a separate clean-context session.

## Location

`~/engagement-kit/engagements/<slug>/hunting-session.md`

## YAML Frontmatter

```yaml
---
session_id: "<uuid>"
engagement: "<slug>"
platform: "hackerone"
phase_completed: "exploit"
started_at: "2026-02-18T10:00:00"
last_updated: "2026-02-18T18:00:00"
findings_confirmed: 3
findings_rejected: 7
estimated_bounty_usd: 4500
---
```

## Sections

### Scope Summary
Root domains, allow/deny rule counts, key constraints, required headers, financial limits.

### Recon Results
Counts + file pointers: all-subdomains.txt, live-hosts.txt, ports-open.txt, urls.txt, params.txt.

### Scan Results
Counts + file pointers: nuclei-output.jsonl, nikto-output.txt, ffuf-*.json, semgrep-results.json.

### Triage Decisions
Table: Finding | Confidence | Severity | Decision | Reasoning

### Confirmed Findings
Table: # | Finding ID | Title | Severity | CVSS | Target | Est. Bounty
Each finding maps to `findings/FINDING-NNN-<slug>/finding.md`.

### Failed Attempts
Table: Finding | Reason | Notes
Documents what didn't work (for dedup and cross-reference).

### Environment Notes
WAF, CDN, auth mechanism, rate limiting observations.

## Update Protocol

Each phase skill appends to or updates the relevant section.
Never delete existing content — only add or update counts.
The writeup skill reads the final state of this file.
