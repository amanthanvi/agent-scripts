---
name: bounty-kickoff
description: "Initialize and verify a bug bounty engagement: scope review, authorization confirmation, safety gate configuration, and scanning enablement. Use when starting a new engagement or re-verifying scope."
---

# Bug Bounty Kickoff Phase

You are initializing a bug bounty engagement for active hunting. This is the ONLY phase where scanning gets enabled.

## Input

The engagement slug was passed from the bounty router. The engagement directory is at:
`~/engagement-kit/engagements/<slug>/`

## Steps

### 1. Validate Structure

```bash
cd ~/engagement-kit && python3 lib/validators.py engagements/<slug>/
```

If validation fails, report the errors and STOP.

### 2. Read Engagement Config

Read `engagements/<slug>/engagement.yaml`. Note:
- Program name, platform, program URL
- Current scanning_enabled status
- Rate limits
- Phase (should be "new" for first kickoff)

### 3. Read Authorization & Rules

Read these files and summarize their key points to the user:

- `admin/authorization.md` — authorization basis, safe harbor, program constraints
- `admin/roe.md` — rules of engagement, special restrictions, required headers
- `admin/scope.md` — asset tiers, reward ranges
- `admin/contacts.md` — researcher handle, program contacts

**STOP if authorization.md is empty or template-only.** Inform the user they need to populate it first.

### 4. Verify Scope Files

```bash
cd ~/engagement-kit && python3 lib/scope_check.py engagements/<slug>/scope_allowlist.txt --verify
```

Report: number of allow rules, number of deny rules.

If scope files are template-only (placeholder content), offer to populate them:
- Search the web for the program's scope page on the relevant platform
- Extract in-scope and out-of-scope assets
- Write to scope_allowlist.txt and out_of_scope.txt
- Re-run --verify

### 5. Extract Program Constraints

From the authorization and ROE files, identify and document:
- **Rate limits** — update engagement.yaml if program specifies different limits
- **Required headers** — update `engagement.yaml` auth.custom_headers (e.g., `X-Bug-Bounty: True`)
- **Financial limits** — note in ROE summary
- **Testing restrictions** — no DoS, no data modification, no social engineering, etc.
- **Special rules** — defense contractor caution (Anduril), sandbox restrictions, etc.

### 6. Update Tech Stack

Read `recon/tech-stack.md`. If it's template-only, populate with known technologies from the program description.

### 7. Generate Test Checklist

Read `testing/test-checklist.md`. Add program-specific test items based on:
- Asset types (web, API, mobile)
- Technology stack
- High-value testing areas from the program description
- Known vulnerability categories for the platform/technology

### 8. APPROVAL GATE

Present a summary to the user:

```
## Engagement Kickoff Summary

**Program**: <name> (<platform>)
**Scope**: <N> allow rules, <N> deny rules
**Root domains**: <list>
**Key Constraints**:
- Rate limit: <N> req/s
- Required headers: <list or "none">
- Financial limit: <amount or "none">
- Restrictions: <list>

**Ready to enable scanning?**
This will set scanning_enabled: true and allow active testing tools to run.
```

**WAIT for explicit user approval before proceeding.**

### 9. Enable Scanning (on approval)

After user confirms:

1. Update `engagement.yaml`:
   - Set `scanning_enabled: true`
   - Set `phase: kickoff_complete`
   - Update `updated_at` timestamp

2. Initialize `hunting-session.md` from template:
   - Copy from `~/engagement-kit/templates/hunting-session.md.tpl`
   - Fill in scope summary section
   - Set session_id (generate a UUID)

3. Update the Makefile DRY_RUN if user explicitly approves:
   - Only if user says "enable live scanning" or equivalent

### 10. Report

```
Kickoff complete for <program>.
- Scanning: ENABLED
- Phase: kickoff_complete
- Next: run `/bounty recon <slug>` or `/bounty <slug>` to start reconnaissance
```

## Error Recovery

If the user denies the approval gate:
- Leave scanning_enabled: false
- Set phase to "new"
- Inform user what they need to review/fix before re-running kickoff
