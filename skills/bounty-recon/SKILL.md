---
name: bounty-recon
description: "Automated reconnaissance for bug bounty targets: subdomain enumeration, port scanning, HTTP probing, technology fingerprinting, URL/endpoint discovery, and secret scanning. Runs after kickoff phase."
---

# Bug Bounty Recon Phase

You are performing automated reconnaissance on an authorized bug bounty target. This phase runs after kickoff and produces the asset inventory that feeds into scanning.

## Input

The engagement slug was passed from the bounty router. The engagement directory is at:
`~/engagement-kit/engagements/<slug>/`

## Pre-flight Checks

### 1. Phase Prerequisite

```bash
cd ~/engagement-kit && python3 lib/phase_state.py engagements/<slug>/ --check kickoff_complete
```

If exit code != 0, STOP. Inform user to run `/bounty kickoff <slug>` first.

### 2. Scanning Gate

```bash
cd ~/engagement-kit && python3 lib/scope_gate.py engagements/<slug>/ --preflight
```

Verifies `scanning_enabled: true` in engagement.yaml. STOP if false.

### 3. Rules of Engagement

Read `admin/roe.md` for scanning restrictions:
- Rate limits (override defaults if program specifies tighter limits)
- Required headers (inject into all HTTP tools)
- Restricted asset types or testing methods
- Time-of-day restrictions

## Tool Chain

### Step 1: Subdomain Enumeration (parallelizable)

Run these concurrently:

**bbot-mcp** (passive + active enum):
- Use `start_bbot_scan` MCP tool with target domain from `scope_allowlist.txt`
- Preset: `subdomain-enum`
- Wait for completion with `wait_for_scan_completion`
- Collect results with `get_scan_results`

**subfinder CLI** (passive sources):
```bash
subfinder -d <domain> -silent -o recon/subfinder-subs.txt
```

**Merge and dedup**:
```bash
cat recon/subfinder-subs.txt recon/bbot-subs.txt | sort -u > recon/all-subdomains.txt
```

**Scope filter** (critical -- removes out-of-scope results):
```bash
cd ~/engagement-kit && python3 lib/scope_gate.py engagements/<slug>/ --check-file recon/all-subdomains.txt > recon/all-subdomains-inscope.txt
```

Report: N total subdomains found, N in-scope after filtering.

### Step 2: Port Scanning (depends on step 1)

**Get tool flags**:
```bash
cd ~/engagement-kit && python3 lib/scope_gate.py engagements/<slug>/ --tool-flags naabu
```

**Run naabu** in batches of 50 hosts (use `run_in_background: true` for large lists):
```bash
~/.pdtm/go/bin/naabu -list recon/all-subdomains-inscope.txt -p 80,443,8080,8443,8000,3000,9443 -rate <N> -silent -o recon/ports-open.txt
```

- Use the rate from `--tool-flags` output
- For lists > 50 hosts, split into batches: `split -l 50 recon/all-subdomains-inscope.txt /tmp/naabu-batch-`
- Merge batch outputs into `recon/ports-open.txt`

### Step 3: HTTP Probing (depends on step 2)

**Get tool flags**:
```bash
cd ~/engagement-kit && python3 lib/scope_gate.py engagements/<slug>/ --tool-flags httpx
```

**Run httpx** with tech detection and no redirects:
```bash
~/.pdtm/go/bin/httpx -list recon/all-subdomains-inscope.txt -silent -status-code -title -tech-detect -json -max-redirects 0 -rl <N> -o recon/httpx-full.json
```

**Extract live hosts** (URLs only):
```bash
cat recon/httpx-full.json | jq -r '.url' > recon/live-hosts.txt
```

Report: N live hosts responding, status code distribution.

### Step 4: Technology Fingerprinting (depends on step 3)

Parse `recon/httpx-full.json` for technology data:

```bash
cat recon/httpx-full.json | jq -r '.tech[]?' | sort | uniq -c | sort -rn
```

Update `recon/tech-stack.md` with:
- Web servers (nginx, Apache, IIS, etc.)
- Frameworks (React, Angular, Django, Rails, etc.)
- CDN/WAF presence (Cloudflare, Akamai, AWS WAF, etc.)
- CMS detection (WordPress, Drupal, etc.)
- Programming languages and versions

Note WAF/CDN presence -- this affects scan phase tool selection and rate limits.

### Step 5: URL/Endpoint Discovery (depends on step 3)

**Historical URLs** (gau):
```bash
gau <domain> --subs --o recon/gau-urls.txt
```

**Live crawling** (katana, rate-limited):
```bash
cd ~/engagement-kit && python3 lib/scope_gate.py engagements/<slug>/ --tool-flags katana
```
```bash
katana -list recon/live-hosts.txt -rl <N> -silent -o recon/katana-urls.txt
```

**Merge and dedup**:
```bash
cat recon/gau-urls.txt recon/katana-urls.txt | sort -u > recon/urls.txt
```

Report: N unique URLs discovered.

### Step 6: Secret Scanning (independent -- run in parallel with steps 2-5)

**GitHub repos** (if in scope):
```bash
~/go/bin/trufflehog filesystem <path> --only-verified --json > recon/trufflehog-results.json
```

**gitleaks**:
```bash
gitleaks detect -s <path> -v --report-path recon/gitleaks-results.json
```

Merge findings into `recon/secrets-scan.txt` with:
- Source (trufflehog/gitleaks)
- Type (API key, password, token, etc.)
- Location (file:line)
- Verification status (verified/unverified)

Only report verified secrets unless user requests unverified.

## Post-flight

### Update Hunting Session

Update `hunting-session.md` recon section with:
- Subdomain count (total / in-scope)
- Live host count
- Open port count
- URL count
- Technologies discovered
- Secrets found (count, not details)
- File pointers to all recon artifacts

### Set Phase Complete

```bash
cd ~/engagement-kit && python3 lib/phase_state.py engagements/<slug>/ --set recon_complete
```

### Summary Report

```
## Recon Complete: <program>

- Subdomains: N total, N in-scope
- Live hosts: N responding
- Open ports: N across N hosts
- URLs discovered: N unique
- Technologies: <top 5>
- Secrets: N verified findings

Artifacts: recon/all-subdomains-inscope.txt, recon/live-hosts.txt, recon/ports-open.txt, recon/urls.txt
Next: run `/bounty scan <slug>` or `/bounty <slug>`
```

## SAFETY INVARIANT

Before every tool invocation, check scope. Before every HTTP request, verify the target is in-scope:

```bash
cd ~/engagement-kit && python3 lib/scope_gate.py engagements/<slug>/ --check <TARGET>
```

Reference `references/safety-invariants.md` from the bounty skill for full safety rules.

All HTTP tools MUST use no-redirect flags to prevent scope escape via 302 redirects.
All scanning tools MUST use rate limits from `--tool-flags`.
All long-running tools MUST use `run_in_background: true` or `timeout`.
