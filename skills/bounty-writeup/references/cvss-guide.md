# CVSS 3.1 Scoring Reference

Reference for bounty-writeup skill. Quick reference for calculating CVSS vectors.

## Vector String Format

```
CVSS:3.1/AV:<val>/AC:<val>/PR:<val>/UI:<val>/S:<val>/C:<val>/I:<val>/A:<val>
```

## Base Metrics

### Attack Vector (AV)
| Value | Code | Score Weight |
|-------|------|-------------|
| Network | N | Highest -- remote exploitation |
| Adjacent | A | Requires adjacent network |
| Local | L | Requires local access |
| Physical | P | Requires physical access |

### Attack Complexity (AC)
| Value | Code | Score Weight |
|-------|------|-------------|
| Low | L | No special conditions |
| High | H | Race condition, specific config, etc. |

### Privileges Required (PR)
| Value | Code | Score Weight |
|-------|------|-------------|
| None | N | No auth needed |
| Low | L | Basic user account |
| High | H | Admin/privileged account |

### User Interaction (UI)
| Value | Code | Score Weight |
|-------|------|-------------|
| None | N | No user action needed |
| Required | R | Victim must click/visit |

### Scope (S)
| Value | Code | Score Weight |
|-------|------|-------------|
| Unchanged | U | Impact limited to vulnerable component |
| Changed | C | Impact extends beyond vulnerable component |

### Confidentiality / Integrity / Availability (C/I/A)
| Value | Code | Score Weight |
|-------|------|-------------|
| None | N | No impact |
| Low | L | Limited impact |
| High | H | Total loss |

## Common Web Vulnerability Vectors

### Critical (9.0-10.0)

**Unauthenticated RCE**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H = 9.8
```

**Auth Bypass to Admin**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H = 9.8
```

### High (7.0-8.9)

**SQL Injection (data exfiltration)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:N/A:N = 8.6
```

**Stored XSS (admin context)**
```
CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:C/C:H/I:H/A:N = 8.4
```

**SSRF (internal resource access)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N = 7.5
```

**Account Takeover via password reset**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N = 9.1
```

### Medium (4.0-6.9)

**IDOR (data access)**
```
CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N = 6.5
```

**Reflected XSS**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N = 6.1
```

**CSRF (state-changing action)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:U/C:N/I:H/A:N = 6.5
```

### Low (0.1-3.9)

**Open Redirect**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:C/C:N/I:L/A:N = 4.7
```

**Information Disclosure (minor)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N = 5.3
```

**Missing Security Header (with context)**
```
CVSS:3.1/AV:N/AC:H/PR:N/UI:R/S:U/C:L/I:N/A:N = 3.1
```

## Justification Checklist

When assigning CVSS in a report, justify each metric:

1. **AV**: How does the attacker reach the vulnerable component? (almost always Network for web)
2. **AC**: Are there prerequisites beyond basic access? (race conditions, specific configs)
3. **PR**: What level of authentication is needed? (none, user, admin)
4. **UI**: Does the victim need to do something? (click link, visit page)
5. **S**: Does exploitation affect other components? (XSS in iframe = Changed scope)
6. **C/I/A**: What's the actual demonstrated impact on each? Base on PoC evidence, not theory.

## Common Mistakes

- Scoring Scope:Changed when impact stays within the same application
- Using C:H for information disclosure of non-sensitive data (use C:L)
- Scoring UI:N for XSS (reflected XSS always requires UI:R)
- Forgetting PR:L when auth is required to reach the endpoint
- Over-scoring A (availability) for non-DoS vulnerabilities
