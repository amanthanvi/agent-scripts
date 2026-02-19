# Platform Report Templates

Reference for bounty-writeup skill. Format differences by submission platform.

## HackerOne

### Report Fields

| Field | Notes |
|-------|-------|
| Title | Short, descriptive: "[VulnType] in [Feature] allows [Impact]" |
| Severity | Select: Critical / High / Medium / Low / None |
| Weakness | CWE dropdown -- search by number or name |
| Asset | Select from program's defined asset list |
| CVSS | Calculator or manual vector string entry |
| Description | Markdown body (main report) |
| Impact | Separate impact section |
| Attachments | Upload evidence files |

### Report Structure

```markdown
## Summary
[1-2 sentence description of the vulnerability]

## Steps to Reproduce
1. Navigate to [URL]
2. [Action with exact parameters]
3. [Observe vulnerability indicator]

## Impact
[What an attacker can ACTUALLY do with this vulnerability]

## Supporting Material/References
- [screenshot-1.png] -- Description
- [request-response.txt] -- Full HTTP capture

## Severity
**CVSS**: [score] ([vector string])
```

### Required Fields
- Weakness (CWE dropdown)
- Severity (CVSS calculator)
- Affected asset (host value from scope)
- Steps to reproduce

### Markdown Conventions
- Use `##` for section headers (H1 is the title field)
- Code blocks with language hints: ````http`, ````json`, ````bash`
- Inline code for URLs, parameters, headers
- Ordered lists for reproduction steps
- Bold for emphasis on key observations

### Example Title Formats
- "Stored XSS in comment field allows session hijacking"
- "IDOR in /api/users/{id} exposes other users' PII"
- "SQL Injection in search parameter allows data exfiltration"
- "SSRF via webhook URL allows internal network scanning"

### Researcher Identification
- Handle: `athanvi`
- Required headers on all requests: check `admin/contacts.md`

---

## Bugcrowd

### Report Structure

```markdown
## Title
[Vulnerability Type] -- [Affected Endpoint]

## Description
[Detailed description]

## Steps to Reproduce
1. [Step]
2. [Step]

## Proof of Concept
[Evidence description + attachments]

## Suggested Remediation
[Fix recommendation]
```

### VRT (Vulnerability Rating Taxonomy) Mapping

| Vuln Class | VRT Category | Priority |
|-----------|-------------|----------|
| RCE | Server-Side Injection > Remote Code Execution | P1 |
| SQLi | Server-Side Injection > SQL Injection | P1-P2 |
| Stored XSS | Cross-Site Scripting > Stored | P2 |
| SSRF | Server-Side Injection > SSRF | P2-P3 |
| IDOR | Broken Access Control > IDOR | P2-P3 |
| Reflected XSS | Cross-Site Scripting > Reflected | P3 |
| Open Redirect | Unvalidated Redirects and Forwards | P4 |
| Info Disclosure | Information Disclosure | P3-P4 |

### Priority Rating
P1 (Critical) through P5 (Informational)

### Report Fields

| Field | Notes |
|-------|-------|
| Title | Same format as H1 |
| VRT | Select from taxonomy tree |
| Priority | P1-P5 |
| URL/Target | Specific affected URL |
| Description | Markdown body |
| Steps to Reproduce | Numbered list |
| Attachments | Upload evidence |

---

## Intigriti

### Report Structure

```markdown
## Vulnerability Type
[Select from taxonomy]

## Domain/Target
[Affected asset]

## Description
[Clear description of the issue]

## Steps to Reproduce
1. [Step]

## Impact
[Business impact]

## Evidence
[Attachments]
```

### Report Fields

| Field | Notes |
|-------|-------|
| Title | Descriptive vulnerability title |
| Severity | Critical / High / Medium / Low |
| Domain | Affected domain from program scope |
| Description | Full report in markdown |
| Impact | Separate impact section |
| Proof of Concept | Steps + evidence |
| Attachments | Upload evidence |

### Impact Categories
- Data breach / Data leak
- Account takeover
- Remote code execution
- Privilege escalation
- Information disclosure
- Denial of service (if in scope)

### Severity Guidelines

| Severity | Typical Findings |
|----------|-----------------|
| Critical | RCE, auth bypass on admin, mass data exfil |
| High | SQLi, stored XSS (admin), SSRF to internal |
| Medium | IDOR, reflected XSS, privilege escalation |
| Low | Info disclosure, open redirect, missing headers with impact |

---

## Cross-Platform Best Practices

1. **First paragraph matters most** -- triagers read the first 2-3 sentences to gauge severity. Lead with impact.
2. **Copy-pasteable reproduction** -- include exact URLs, headers, and payloads. Triager should ctrl+c/ctrl+v into their terminal.
3. **One finding per report** -- never bundle multiple vulns. Exception: attack chains where individual findings are low severity but chain is high.
4. **Screenshots annotated** -- highlight the vulnerable response/behavior with arrows or boxes if possible.
5. **Remediation is optional but appreciated** -- shows expertise, helps triage team.
6. **Timeline**: submit within program's SLA expectations. Most programs expect reports within 90 days of discovery.
