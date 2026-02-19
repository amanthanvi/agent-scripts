# Web Application Recon Playbook

Strategy guide for web app reconnaissance during bug bounty engagements.

## Subdomain Enumeration

### Passive Sources (no target interaction)

| Tool | Coverage | Notes |
|------|----------|-------|
| subfinder | Certificate transparency, DNS datasets, web archives | Fast, broad passive coverage |
| bbot (subdomain-enum preset) | Combines multiple passive + light active techniques | Comprehensive but slower |

Run both in parallel for maximum coverage. Merge + dedup results.

### Active Enumeration (touches target DNS)

- DNS brute-force: only if ROE permits active DNS queries
- Use focused wordlists: `/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt`
- Full wordlist (`subdomains-top1million-110000.txt`) only for high-value targets with user approval
- Rate limit DNS queries to avoid resolver throttling

### Dedup Strategy

1. Merge all sources into single file
2. `sort -u` to dedup
3. Scope-filter with `scope_gate.py --check-file` immediately
4. Work only with in-scope results from this point forward

## Port Scanning

### Top Ports (default -- fast)

Web-focused port set: `80,443,8080,8443,8000,3000,9443`

Covers:
- Standard HTTP/HTTPS (80, 443)
- Common dev/admin ports (8080, 8443, 8000, 3000)
- Alternative TLS (9443)

### Extended Ports (Tier 2 -- user-approved)

Add: `81,300,591,593,832,981,1010,1311,2082,2087,2095,2096,2480,3128,3333,4243,4567,4711,4712,4993,5000,5104,5108,5800,6543,7000,7396,7474,8001,8008,8014,8042,8069,8081,8088,8090,8091,8118,8123,8172,8222,8243,8280,8281,8333,8443,8500,8834,8880,8888,8983,9000,9043,9060,9080,9090,9091,9200,9443,9800,9981,9999,10000,11371,12443,16080,18091,18092,20720,28017`

### Full Range (Tier 3 -- rare, needs explicit approval)

`naabu -p - ` (all 65535 ports). Only for high-value single targets. Use `run_in_background: true`.

### Timing

- Default rate: use `--tool-flags` from scope_gate
- Large lists (>50 hosts): batch into groups of 50
- Background execution for scans expected to take >5 minutes

## HTTP Probing

### Best Practices

1. **No redirects**: always use `-max-redirects 0` to prevent scope escape
2. **Tech detection**: enable `-tech-detect` for automatic fingerprinting
3. **JSON output**: use `-json` for machine-parseable results
4. **Status codes**: capture with `-status-code` for prioritization
5. **Titles**: capture with `-title` for quick asset identification

### Prioritization from httpx Results

- **200 OK**: live targets, highest priority
- **401/403**: protected resources, potential bypass targets
- **301/302**: redirect destinations need scope verification before following
- **500/502/503**: error pages may leak information

### Output Files

- `recon/httpx-full.json`: complete httpx output (JSON lines)
- `recon/live-hosts.txt`: URLs only (extracted from JSON)

## URL Discovery

### Historical URLs (gau)

- Sources: Wayback Machine, Common Crawl, Open Threat Exchange, URLScan
- No target interaction -- purely passive
- High volume; expect many dead/changed URLs
- Filter for interesting extensions: `.json`, `.xml`, `.yaml`, `.env`, `.bak`, `.sql`, `.conf`

### Live Crawling (katana)

- Active crawling -- respects rate limits from scope_gate
- Discovers JavaScript-rendered content
- Finds API endpoints in JS bundles
- Rate limit is critical -- use `--tool-flags katana`
- Depth: default 3 levels; increase only for specific targets

### URL Processing Pipeline

1. Merge gau + katana output
2. Dedup with `sort -u`
3. Filter interesting patterns:
   - Parameters: URLs with `?` (potential injection points)
   - API paths: `/api/`, `/v1/`, `/graphql`
   - Admin paths: `/admin/`, `/dashboard/`, `/manage/`
   - Config files: `.env`, `.json`, `.xml`, `.yaml`
4. Write to `recon/urls.txt`

## Secret Scanning

### Priority Order

1. **Verified secrets only** (default): trufflehog `--only-verified` flag
2. Unverified secrets: only if user explicitly requests; high false-positive rate

### When to Run

- GitHub repos listed in scope
- Source code obtained through other recon
- Public S3 buckets / blob storage
- JavaScript bundles (extract with katana, scan extracted files)

### Tools

| Tool | Strength | Use When |
|------|----------|----------|
| trufflehog | Verifies secrets are live | GitHub repos, filesystem |
| gitleaks | Git history analysis | Cloned repos with commit history |

### Output

Report only:
- Verified findings (confirmed live/valid)
- Secret type (AWS key, GitHub token, Slack webhook, etc.)
- Location (file and line)
- Do NOT include the actual secret value in reports

## Target Prioritization

### Tier 1 (scan first)

- Main application domains
- API endpoints
- Authentication flows
- Payment/financial features
- Admin panels

### Tier 2 (scan second)

- Subdomains with unique functionality
- Development/staging environments
- Internal tools exposed externally
- Legacy applications

### Tier 3 (scan if time permits)

- Static content hosts
- CDN endpoints
- Marketing/blog subdomains
- Third-party integrations

### Prioritization Signals

- Custom application code > off-the-shelf software
- Dynamic content > static pages
- Authenticated areas > public pages
- Newer deployments > legacy (more likely to have unfixed bugs)
- Higher bounty tiers > lower tiers
