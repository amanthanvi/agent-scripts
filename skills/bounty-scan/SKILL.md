---
name: bounty-scan
description: "Automated vulnerability scanning: nuclei templates, nikto web server scan, ffuf content discovery, semgrep SAST, and API-specific testing. Runs after recon phase."
---

# Bug Bounty Scan Phase

You are performing automated vulnerability scanning on an authorized bug bounty target. This phase consumes recon output and produces vulnerability findings for triage.

## Input

The engagement slug was passed from the bounty router. The engagement directory is at:
`~/engagement-kit/engagements/<slug>/`

## Pre-flight Checks

### 1. Phase Prerequisite

```bash
cd ~/engagement-kit && python3 lib/phase_state.py engagements/<slug>/ --check recon_complete
```

If exit code != 0, STOP. Inform user to run `/bounty recon <slug>` first.

### 2. Read Recon Results

Read these files to understand the attack surface:
- `recon/live-hosts.txt` -- targets for scanning
- `recon/tech-stack.md` -- technology fingerprints (guides template selection)
- `recon/httpx-full.json` -- detailed host info (status codes, WAF detection)
- `recon/urls.txt` -- discovered endpoints
- `recon/ports-open.txt` -- open ports beyond HTTP

### 3. Rules of Engagement

Read `admin/roe.md` for scanning restrictions:
- Forbidden scan types (e.g., no DoS templates)
- Rate limits (may be tighter than defaults)
- Required headers
- Time-of-day restrictions
- Financial limits on test transactions

### 4. Get Tool Flags

For each scanning tool, retrieve rate limits and safety flags:
```bash
cd ~/engagement-kit && python3 lib/scope_gate.py engagements/<slug>/ --tool-flags <tool-name>
```

## Tool Chain

### Step 1: Nuclei Scanning (primary vulnerability scanner)

**Get flags**:
```bash
cd ~/engagement-kit && python3 lib/scope_gate.py engagements/<slug>/ --tool-flags nuclei
```

**Run nuclei** with rate limiting and no-redirect:
```bash
nuclei -list recon/live-hosts.txt -rl <N> --no-follow-redirects -jsonl -o recon/scans/nuclei-output.jsonl
```

**Template selection strategy** (based on tech-stack.md):
- Default: all templates at configured severity threshold
- Tech-specific: add `-tags <tech>` for detected technologies (e.g., `-tags wordpress,php,nginx`)
- Severity ordering: run critical/high first, then medium
- Exclude info templates unless user requests: `-severity critical,high,medium`

**Custom templates** (optional):
- Use nuclei-mcp `add_template` to create custom templates for tech-stack-specific checks
- Use nuclei-mcp `nuclei_scan` for AI-assisted template generation based on discovered technologies

**For large target lists** (>50 hosts):
- Split into batches
- Use `run_in_background: true`
- Merge output files after completion

### Step 2: Nikto Scanning (web server vulnerabilities)

**Run against top 10 live hosts by priority** (Tier 1 targets first):

```bash
cd ~/engagement-kit && python3 lib/scope_gate.py engagements/<slug>/ --tool-flags nikto
```

```bash
nikto -h <host> -Pause <N> -output recon/scans/nikto-<host-slug>.txt -Format txt
```

- Select top 10 hosts from `recon/live-hosts.txt` prioritized by:
  - Custom applications over off-the-shelf
  - Hosts with interesting status codes (401, 403, 500)
  - Hosts running older/uncommon web servers
- One nikto invocation per host (not parallelizable due to rate limits)
- Use `timeout 300` to cap per-host runtime at 5 minutes

### Step 3: Content Discovery (ffuf)

**Get flags**:
```bash
cd ~/engagement-kit && python3 lib/scope_gate.py engagements/<slug>/ --tool-flags ffuf
```

**Run ffuf** against priority targets:
```bash
ffuf -u <url>/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -rate <N> -r -mc 200,301,302,403 -of json -o recon/scans/ffuf-<host-slug>.json
```

**Wordlist selection** (based on tech-stack.md):
- Default: `/usr/share/seclists/Discovery/Web-Content/common.txt`
- PHP: add `/usr/share/seclists/Discovery/Web-Content/Common-PHP-Filenames.txt`
- Java/Spring: add `/usr/share/seclists/Discovery/Web-Content/spring-boot.txt`
- .NET: add `/usr/share/seclists/Discovery/Web-Content/IIS.fuzz.txt`
- API: add `/usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt`
- Backup files: `/usr/share/seclists/Discovery/Web-Content/Common-DB-Backups.txt`

**Filter noise**:
- Match status codes: `200,301,302,403` (skip 404)
- Filter by size if 404 responses have consistent size: `-fs <size>`
- Auto-calibrate: `-ac` to auto-filter common response sizes

### Step 4: SAST (if source code available)

**When to run**: only if source code is available (public repos in scope, leaked source, etc.)

**semgrep-mcp**: use the Semgrep MCP tool to run analysis:
- Auto config: let Semgrep detect languages and apply default rulesets
- Custom rulesets for detected technologies
- Focus on security-relevant findings (injection, auth bypass, crypto issues)

**Output**: `recon/scans/semgrep-results.json`

Report: N findings by severity, top vulnerability categories.

### Step 5: API Discovery

**Check for API documentation** at common paths on each live host:

```
/swagger.json
/swagger/v1/swagger.json
/api-docs
/api/swagger.json
/openapi.json
/openapi.yaml
/v1/api-docs
/v2/api-docs
/v3/api-docs
/.well-known/openapi.yaml
```

Use httpx or curl with no-redirect and scope check:
```bash
cd ~/engagement-kit && python3 lib/scope_gate.py engagements/<slug>/ --check <host>
curl -s -o /dev/null -w "%{http_code}" --max-redirs 0 <url>/swagger.json
```

**GraphQL introspection**:
```bash
curl -s -X POST <url>/graphql -H "Content-Type: application/json" -d '{"query":"{ __schema { types { name } } }"}' | head -c 500
```

**Document findings** in `recon/scans/api-endpoints.md`:
- Swagger/OpenAPI spec URLs
- GraphQL endpoints (introspection enabled/disabled)
- REST API base paths
- Authentication requirements observed
- Interesting endpoint patterns (CRUD, file upload, admin)

## Post-flight

### Write Scan Summary

Create `recon/scans/scan-summary.md` with:
- Tool invocations and runtimes
- Nuclei: N findings by severity (critical/high/medium/low/info)
- Nikto: N findings per host
- ffuf: N discovered paths per host
- Semgrep: N findings by category (if applicable)
- API endpoints discovered
- Notable findings flagged for triage

### Update Hunting Session

Update `hunting-session.md` scan section with:
- Scan completion timestamp
- Finding counts per tool
- File pointers to all scan artifacts
- Priority findings for immediate triage

### Set Phase Complete

```bash
cd ~/engagement-kit && python3 lib/phase_state.py engagements/<slug>/ --set scan_complete
```

### Summary Report

```
## Scan Complete: <program>

- Nuclei: N findings (C critical, H high, M medium)
- Nikto: N findings across N hosts
- ffuf: N paths discovered across N hosts
- API: N endpoints documented
- Semgrep: N findings (if applicable)

Top priority findings:
1. <finding summary>
2. <finding summary>
3. <finding summary>

Artifacts: recon/scans/nuclei-output.jsonl, recon/scans/scan-summary.md
Next: run `/bounty triage <slug>` or `/bounty <slug>`
```

## SAFETY

All targets MUST be scope-checked before scanning. Reference the bounty skill's `references/safety-invariants.md`.

**Non-negotiable rules**:
- No-redirect flags on ALL HTTP tools (prevents scope escape via 302)
- Rate limits from `--tool-flags` on ALL scanning tools
- No DoS templates in nuclei (`-exclude-tags dos`)
- No active exploitation in scan phase -- discovery only
- Custom headers injected into all HTTP requests if required by ROE
- `timeout` on all tool invocations to prevent runaway scans
- Batch large target lists (50 hosts max per batch)
