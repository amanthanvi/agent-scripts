---
name: bounty
description: "Bug bounty engagement automation. Routes to phase skills: kickoff, recon, scan, triage, exploit, writeup. Use when starting, resuming, or managing a bug bounty engagement. Invoke with /bounty <slug> or /bounty <phase> <slug>."
---

# Bug Bounty Engagement Router

You are managing a bug bounty engagement using the engagement-kit framework at `~/engagement-kit/`.

## Argument Parsing

Parse the user's input to determine the action:

- `/bounty list` → run `cd ~/engagement-kit && python3 lib/scaffold.py list`
- `/bounty status <slug>` → read engagement.yaml + hunting-session.md, summarize phase + scope
- `/bounty <slug>` → auto-detect current phase from engagement.yaml, resume at next phase
- `/bounty <phase> <slug>` → run the specified phase (kickoff, recon, scan, triage, exploit, writeup)
- `/bounty preflight <slug>` → run `cd ~/engagement-kit && python3 lib/scope_gate.py engagements/<slug>/ --preflight`

## SAFETY INVARIANTS (check every invocation)

Before routing to any phase skill:

1. **Read authorization**: read `~/engagement-kit/engagements/<slug>/admin/authorization.md` — STOP if empty or template-only
2. **Verify scope**: `cd ~/engagement-kit && python3 lib/scope_check.py engagements/<slug>/scope_allowlist.txt --verify`
3. **Check DRY_RUN**: read the engagement Makefile and check DRY_RUN value — inform user if DRY_RUN=1
4. **Check scanning_enabled**: `cd ~/engagement-kit && python3 -c "from lib.config import load_config, is_scanning_enabled; print(is_scanning_enabled(load_config('engagements/<slug>/')))"` — if False, route to kickoff first
5. **Preflight MCP tools**: `cd ~/engagement-kit && python3 lib/scope_gate.py engagements/<slug>/ --preflight`

## Phase Routing

After safety checks pass, route to the appropriate phase skill by description matching:

| Phase | Skill to invoke | Prerequisite |
|-------|----------------|--------------|
| kickoff | bounty-kickoff | phase = new |
| recon | bounty-recon | phase >= kickoff_complete |
| scan | bounty-scan | phase >= recon_complete |
| triage | bounty-triage | phase >= scan_complete |
| exploit | bounty-exploit | phase >= triage_complete |
| writeup | bounty-writeup | phase >= exploit_complete |

Check prerequisites: `cd ~/engagement-kit && python3 lib/phase_state.py engagements/<slug>/ --check <required_phase>`

## Auto-Resume Logic

When no phase is specified (`/bounty <slug>`):
1. Read current phase: `cd ~/engagement-kit && python3 lib/phase_state.py engagements/<slug>/ --get`
2. Route to the NEXT phase after the current one
3. If phase is `new`, route to kickoff
4. If phase is `writeup_complete`, inform user that engagement is complete

## Scope Gate Usage

Before ANY command that targets a host/URL, check scope first:
```bash
cd ~/engagement-kit && python3 lib/scope_gate.py engagements/<slug>/ --check <TARGET>
```
If exit code != 0, STOP. Do not proceed.

For filtering a list of targets:
```bash
cd ~/engagement-kit && python3 lib/scope_gate.py engagements/<slug>/ --check-file <targets-file>
```

For tool-specific safety flags (rate limit, no-redirect, custom headers):
```bash
cd ~/engagement-kit && python3 lib/scope_gate.py engagements/<slug>/ --tool-flags <tool-name>
```

## Key Paths

- Engagement dir: `~/engagement-kit/engagements/<slug>/`
- Scope allowlist: `scope_allowlist.txt`
- Out of scope: `out_of_scope.txt`
- Authorization: `admin/authorization.md`
- Rules of engagement: `admin/roe.md`
- Config: `engagement.yaml`
- Hunting session: `hunting-session.md`
- Audit log: `logs/audit.jsonl`

Read `references/safety-invariants.md` for detailed safety rules.
Read `references/artifact-schema.md` for the hunting-session.md format.
