# Triage Scoring Rubric

Reference for bounty-triage skill. Scoring system for prioritizing findings.

## Confidence Scoring (0-100)

Points are additive. Start at 0, add applicable modifiers:

### Tool Agreement (+20 max)
- Single tool detection: +0
- Two tools agree: +10
- Three or more tools agree: +20

### Response Analysis (+30 max)
- Generic template match only: +0
- Status code/header anomaly consistent with vuln: +10
- Actual vulnerability indicator in response body: +20
- Exploitable payload reflected/executed: +30

### Technology Consistency (+20 max)
- Tech stack unknown: +5 (benefit of doubt)
- Tech stack does NOT support the vuln class: -20 (likely FP)
- Tech stack matches vuln (e.g., SQLi on PHP+MySQL): +20

### Reproducibility (+30 max)
- Not yet tested: +0
- Automated replay confirms: +15
- Manual verification confirms: +30

## Severity: CVSS 3.1 Base Score

### Program-Specific Modifiers
- Behind authentication: -1.0 from base score
- Requires user interaction: -1.0 from base score
- Affects financial data or transactions: +1.0 to base score
- Affects PII or health data: +1.0 to base score
- Chained with another finding: use combined impact score

### CVSS Quick Reference for Common Web Vulns

| Vulnerability Class | Typical CVSS | Vector (common) |
|---------------------|-------------|-----------------|
| Unauthenticated RCE | 9.8 | AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H |
| SQL Injection (data exfil) | 8.6 | AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:N/A:N |
| Stored XSS (admin context) | 8.4 | AV:N/AC:L/PR:N/UI:R/S:C/C:H/I:H/A:N |
| SSRF (internal access) | 7.5 | AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N |
| IDOR (data access) | 6.5 | AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N |
| Reflected XSS | 6.1 | AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N |
| Open redirect | 4.7 | AV:N/AC:L/PR:N/UI:R/S:C/C:N/I:L/A:N |
| Info disclosure (minor) | 3.7 | AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N |
| Missing security header | 2.0 | Informational -- program dependent |

## Asset Tier Weighting

| Tier | Description | Weight | Examples |
|------|-------------|--------|----------|
| 1 | Primary domains, core app, API | 3x | `*.robinhood.com`, `api.example.com` |
| 2 | Secondary services, subdomains | 2x | `blog.example.com`, `status.example.com` |
| 3 | Low-value, informational assets | 1x | `docs.example.com`, static CDN hosts |

Tier assignment: read `engagement.yaml` for explicit tiers. If not defined, infer from asset criticality.

## Expected Value Calculation

```
expected_value = (confidence / 100) * cvss_score * tier_weight
```

Sort findings by expected_value descending. This prioritizes high-confidence, high-severity findings on critical assets.

## Decision Thresholds

| Category | Expected Value | Action |
|----------|---------------|--------|
| Confirmed | >= 15.0 | Exploit immediately |
| Probable | 8.0 - 14.9 | Manual verification, then exploit |
| Possible | 3.0 - 7.9 | Low priority, verify if time permits |
| Discarded | < 3.0 or FP match | Log reason, skip |
