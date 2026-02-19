# Web Application Scanning Playbook

Strategy guide for vulnerability scanning during bug bounty engagements.

## Nuclei Template Strategy

### Severity-Based Ordering

Run in this order to surface critical issues first:

1. **Critical + High** (always run first):
   ```
   nuclei -severity critical,high -exclude-tags dos
   ```
   Catches: RCE, SQLi, SSRF, auth bypass, sensitive data exposure

2. **Medium** (run second):
   ```
   nuclei -severity medium -exclude-tags dos
   ```
   Catches: XSS, CSRF, IDOR indicators, misconfigurations

3. **Low + Info** (only if user requests):
   ```
   nuclei -severity low,info
   ```
   Catches: version disclosure, headers, informational findings

### Tech-Specific Templates

Select templates based on `recon/tech-stack.md`:

| Technology | Tags | Focus Areas |
|-----------|------|-------------|
| WordPress | `wordpress,wp-plugin` | Plugin vulns, wp-config exposure, xmlrpc |
| PHP | `php` | File inclusion, deserialization, type juggling |
| Java/Spring | `java,spring` | Spring4Shell, actuator exposure, JNDI |
| .NET | `asp,iis` | ViewState deserialization, web.config |
| Node.js | `nodejs` | Prototype pollution, SSRF, template injection |
| Python/Django | `python,django` | Debug mode, SSTI, insecure deserialization |
| Ruby/Rails | `ruby,rails` | Mass assignment, SSTI, secret_key_base |
| nginx | `nginx` | Path traversal, alias misconfiguration |
| Apache | `apache` | .htaccess bypass, mod_status, Struts |

### Template Exclusions

Always exclude:
- `dos` tag: DoS templates are never appropriate for bug bounty
- `fuzz` tag: high-volume fuzzing templates (use ffuf instead for controlled fuzzing)
- `intrusive` tag: unless user explicitly approves

### Custom Templates (nuclei-mcp)

Use nuclei-mcp for:
- Generating templates for program-specific patterns
- Tech-stack-specific checks not covered by default templates
- Custom header injection requirements
- Testing specific vulnerability hypotheses from recon findings

## Nikto Usage

### When to Use

- Targets running traditional web servers (Apache, IIS, nginx)
- Hosts with older technology stacks
- Targets not behind strong WAF/CDN
- When looking for server-level misconfigurations

### When to Skip

- Targets behind aggressive WAFs (Cloudflare, Akamai) -- high false positive rate
- API-only endpoints (nikto is page-oriented)
- Already well-covered by nuclei results

### Rate Limiting

- Use `-Pause <N>` from scope_gate `--tool-flags nikto`
- Cap runtime per host: `timeout 300 nikto ...`
- Run against max 10 hosts per engagement (diminishing returns beyond that)
- Prioritize hosts by: custom apps > default installs, older > newer servers

### Useful Flags

```
nikto -h <host> -Pause <N> -Tuning x6 -output <file> -Format txt
```

- `-Tuning x6`: skip denial-of-service tests
- `-Format txt`: readable output (also supports csv, json, xml)
- `-ssl`: force SSL if target is HTTPS
- `-no404`: skip 404 guessing (faster, less noise)

## ffuf Content Discovery

### Wordlist Selection

**Start small, expand if needed**:

1. **Default** (always run):
   - `/usr/share/seclists/Discovery/Web-Content/common.txt` (~4600 entries)

2. **Tech-specific** (add based on tech-stack.md):
   - PHP: `/usr/share/seclists/Discovery/Web-Content/Common-PHP-Filenames.txt`
   - Java: `/usr/share/seclists/Discovery/Web-Content/spring-boot.txt`
   - .NET: `/usr/share/seclists/Discovery/Web-Content/IIS.fuzz.txt`
   - API: `/usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt`

3. **Deep enumeration** (user-approved, Tier 1 targets only):
   - `/usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt`
   - `/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt`

### Status Code Filtering

```
-mc 200,301,302,403
```

- **200**: direct hits -- highest value
- **301/302**: redirects -- check destination is in scope before following
- **403**: forbidden -- potential access control bypass targets
- **Skip 404**: default not-found (noise)
- **500**: optionally include for error-based discovery

### Noise Reduction

- Auto-calibrate: `-ac` (recommended first pass)
- Filter by response size: `-fs <size>` when 404s have consistent size
- Filter by word count: `-fw <count>` for template-based 404 pages
- Filter by line count: `-fl <count>` as additional filter

### Rate Limiting

- Always use `-rate <N>` from scope_gate
- Never exceed program-specified rate limits
- For WAF-protected targets: reduce rate by 50% to avoid blocks
- Add delay: `-p 0.1` for extra throttling if needed

## API Endpoint Testing

### Discovery Checklist

**OpenAPI/Swagger**:
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
/api/v1/docs
/docs
/redoc
```

**GraphQL**:
```
/graphql
/graphiql
/altair
/playground
/api/graphql
/v1/graphql
```

Test introspection:
```json
{"query":"{ __schema { queryType { name } mutationType { name } types { name fields { name } } } }"}
```

### IDOR Patterns

Look for sequential/predictable identifiers in:
- URL path: `/api/users/123`, `/api/orders/456`
- Query params: `?id=123`, `?user_id=456`
- Request body: `{"userId": 123}`
- Headers: `X-User-Id: 123`

Testing approach:
1. Identify authenticated endpoints from API docs
2. Note parameter patterns (numeric IDs, UUIDs)
3. Flag for triage phase -- do NOT exploit in scan phase
4. Document in `recon/scans/api-endpoints.md`

### Authentication Patterns

Document observed auth mechanisms:
- Bearer tokens (JWT, opaque)
- API keys (header, query param)
- Session cookies
- OAuth flows
- Basic auth
- No auth (unauthenticated endpoints)

## Semgrep Rule Selection

### Auto Config (default)

Let Semgrep detect languages and apply appropriate rulesets:
```
semgrep --config auto
```

Covers: OWASP Top 10, CWE Top 25, language-specific security rules.

### Custom Rulesets (tech-specific)

| Target Stack | Ruleset | Focus |
|-------------|---------|-------|
| Python/Django | `p/django`, `p/python` | ORM injection, SSTI, insecure deserialization |
| JavaScript/Node | `p/nodejs`, `p/javascript` | Prototype pollution, XSS, SSRF |
| Java/Spring | `p/java`, `p/spring` | Injection, deserialization, JNDI |
| Ruby/Rails | `p/ruby`, `p/rails` | Mass assignment, SSTI |
| Go | `p/golang` | Race conditions, command injection |
| PHP | `p/php` | Type juggling, file inclusion |

### Severity Filtering

- Always review: ERROR and WARNING findings
- Skip NOTE findings unless specifically hunting for code quality issues
- Focus on injection, authentication, and authorization findings

## Scan Prioritization

### Phase 1: Critical/High Templates (30 min)

- Nuclei critical+high against all live hosts
- Check for known CVEs matching detected tech stack
- API documentation discovery

### Phase 2: Medium Templates + Content Discovery (1 hr)

- Nuclei medium templates
- ffuf with common.txt on Tier 1 targets
- Nikto on top 5 hosts

### Phase 3: Deep Scanning (user-approved, 2+ hrs)

- ffuf with extended wordlists
- Nikto on remaining hosts
- Tech-specific nuclei templates
- SAST if source available

### When to Stop Scanning

- Rate limit budget exhausted
- Diminishing returns (same finding types repeating)
- Sufficient findings queued for triage
- Time budget reached
- WAF blocking indicates detection -- pause and reassess
