# False Positive Patterns

Reference for bounty-triage skill. Known FP patterns by tool.

## Nuclei

| Pattern | Why It's FP | Action |
|---------|-------------|--------|
| Generic header checks (X-Frame-Options, CSP) | Info-severity, not exploitable alone | Discard unless chained |
| Version detection matching ANY version | Template matches presence, not vulnerable version | Verify exact version vs CVE range |
| SSL certificate info findings | Certificate metadata, not a vulnerability | Discard |
| `tech-detect` templates reported as vulns | Technology detection != vulnerability | Reclassify as recon data |
| CNAME/DNS findings without takeover | DNS record exists but no dangling reference | Verify with manual subdomain takeover check |
| Default page detection | Server running != misconfigured | Discard unless sensitive info exposed |
| Generic WAF bypass templates | Many trigger on normal WAF responses | Verify payload actually bypasses |

## Nikto

| Pattern | Why It's FP | Action |
|---------|-------------|--------|
| OSVDB references for ancient vulns | References CVEs from 2003-2010 on modern stacks | Discard unless version confirmed old |
| Generic "interesting" headers | `X-Powered-By: Express` is info, not vuln | Reclassify as recon |
| `robots.txt` entries | Having robots.txt is normal | Only flag if sensitive paths revealed |
| "Retrieved X-Powered-By header" | Technology disclosure, rarely bounty-eligible | Discard |
| "Cookie created without httponly flag" | Only relevant for session cookies | Check if cookie is actually a session cookie |
| "Server leaks inodes via ETags" | Ancient Apache vuln, rarely exploitable | Discard on modern servers |

## ffuf / Feroxbuster

| Pattern | Why It's FP | Action |
|---------|-------------|--------|
| WAF block pages returning 200 | WAF serves block page with 200 status | Check response body for block indicators |
| WAF block pages returning 403 | Standard WAF behavior, not hidden content | Discard unless content differs |
| Redirect chains (301/302) | Redirects to login/home, not hidden content | Follow redirect; check final destination |
| Soft 404s | Custom 404 page returns 200 | Compare response size/content to known 404 |
| Rate-limit pages (429 disguised as 200) | Rate limiting, not content discovery | Check for rate-limit headers |
| Catch-all routes (wildcard handlers) | Framework returns same page for any path | Baseline with random path first |

## sqlmap

| Pattern | Why It's FP | Action |
|---------|-------------|--------|
| "Parameter might be injectable" (no confirmation) | Heuristic match without exploitation | Requires manual verification |
| Time-based blind on slow endpoints | Slow response != time-based SQLi | Compare baseline response time |
| Boolean-based on variable content | Dynamic content != boolean oracle | Test with known true/false conditions |

## General Cross-Tool Patterns

| Pattern | Why It's FP | Action |
|---------|-------------|--------|
| Info-severity findings | Not exploitable, not bounty-eligible | Discard unless program explicitly accepts |
| Cookie flags on non-session cookies | Analytics/tracking cookies don't need httponly | Identify cookie purpose first |
| Missing headers without exploitable impact | No CSP but no XSS vector = no impact | Only flag if paired with exploitable finding |
| Findings on out-of-scope assets | Scan hit adjacent hosts | Cross-reference `out_of_scope.txt` |
| Duplicate finding from multiple tools | Same vuln reported by nuclei + nikto | Dedup; keep highest-detail report; note multi-tool agreement |
| Self-signed cert on internal service | Expected for internal infrastructure | Discard unless program says otherwise |

## FP Detection Heuristics

When uncertain, apply these checks:

1. **Response body check**: Does the response actually contain vulnerability indicators, or just a generic page?
2. **Version check**: Is the detected version actually in the vulnerable range for the CVE?
3. **Technology check**: Does the tech stack support the vulnerability class? (e.g., SQLi on a static site = FP)
4. **Baseline comparison**: Compare the "vulnerable" response to a normal response. If identical, likely FP.
5. **Rate-limit check**: Did the tool get rate-limited and misinterpret the response?
