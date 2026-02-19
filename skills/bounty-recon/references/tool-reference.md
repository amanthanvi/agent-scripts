# Tool Reference for Bug Bounty Recon & Scan

Quick reference for all available tools and their invocation patterns.

## MCP Tools (available via Claude Code tool discovery)

### bbot-mcp
- `start_bbot_scan` — Start subdomain enumeration scan
- `get_scan_status` — Check scan progress
- `get_scan_results` — Retrieve scan output
- `list_active_scans` — List running scans

### nuclei-mcp
- `nuclei_scan` — Run nuclei with custom targets and templates
- `add_template` — Add custom nuclei template
- `list_templates` — List available templates

### semgrep-mcp
- SAST analysis on source code
- Use for code review of in-scope repositories

### hexstrike-ai (150+ tools)
- `nmap_scan` / `nmap_advanced_scan` — Port scanning
- `ffuf_scan` — Content discovery
- `nikto_scan` — Web server scanning
- `httpx_probe` — HTTP probing
- `subfinder_scan` — Subdomain enumeration
- `nuclei_scan` — Template-based scanning
- `katana_crawl` — Web crawling
- `wafw00f_scan` — WAF detection
- `sqlmap_scan` — SQL injection testing
- `dalfox_xss_scan` — XSS testing
- `http_repeater` — Replay HTTP requests
- `http_intruder` — Parameterized HTTP attacks

## CLI Tools (invoked via Bash)

### Subdomain Enumeration
```bash
# subfinder (passive)
subfinder -d <domain> -silent -o recon/subfinder.txt

# bbot (recursive, active+passive)
bbot -t <domain> -p subdomain-enum -o recon/bbot/
```

### Port Scanning
```bash
# naabu (fast, SYN scan)
~/.pdtm/go/bin/naabu -list targets.txt -p 80,443,8080,8443 -rate <N> -silent -o recon/ports-open.txt

# nmap (detailed, service detection)
timeout 600 nmap -iL targets.txt -sV -T3 --max-rate <N> -oA recon/nmap-scan
```

### HTTP Probing
```bash
# httpx (PD — status, title, tech-detect)
~/.pdtm/go/bin/httpx -list targets.txt -silent -status-code -title -tech-detect -json -max-redirects 0 -rl <N> -o recon/httpx-full.json
```

### URL Discovery
```bash
# gau (historical URLs from Wayback/CommonCrawl/URLScan)
gau <domain> --subs --o recon/gau-urls.txt

# katana (live crawling, rate-limited)
katana -list recon/live-hosts.txt -rl <N> -silent -o recon/katana-urls.txt
```

### Secret Scanning
```bash
# trufflehog (verified secrets only)
~/go/bin/trufflehog filesystem <path> --only-verified

# gitleaks (git-focused)
gitleaks detect -s <repo-path> -v
```

### Vulnerability Scanning
```bash
# nuclei (template-based)
nuclei -list recon/live-hosts.txt -rl <N> --no-follow-redirects -jsonl -o recon/scans/nuclei-output.jsonl

# nikto (web server)
nikto -h <host> -Pause <N> -output recon/scans/nikto-output.txt

# ffuf (content discovery)
ffuf -u <url>/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -rate <N> -r -mc 200,301,302,403 -of json -o recon/scans/ffuf-<host>.json
```

## Wordlists

- General: `/usr/share/wordlists/`
- SecLists: `/usr/share/seclists/`
  - Web content: `Discovery/Web-Content/common.txt`, `big.txt`, `raft-medium-words.txt`
  - DNS: `Discovery/DNS/subdomains-top1million-5000.txt`
  - Parameters: `Discovery/Web-Content/burp-parameter-names.txt`
  - API: `Discovery/Web-Content/api/`

## Rate Limit Injection

Always get rate limit flags before running tools:
```bash
cd ~/engagement-kit && python3 lib/scope_gate.py engagements/<slug>/ --tool-flags <tool-name>
```
