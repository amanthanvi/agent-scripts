# Safety Invariants for Bug Bounty Skills

## Scope Enforcement

Every tool invocation targeting a host/URL MUST be preceded by a scope check:

```bash
cd ~/engagement-kit && python3 lib/scope_gate.py engagements/<slug>/ --check <TARGET>
```

- Exit 0 = in scope, proceed
- Exit 1 = out of scope, STOP immediately
- Exit 2 = config error, fix before proceeding

The scope gate checks BOTH `scope_allowlist.txt` AND `out_of_scope.txt`.

## Scanning Gate

`scanning_enabled` must be `true` in `engagement.yaml` before any active scanning.
The scope_gate.py checks this automatically. If false, the error message says:
"Run /bounty kickoff <slug> to review scope and enable scanning."

## Rate Limiting

Every scanning tool must respect the engagement's rate limits:

```bash
cd ~/engagement-kit && python3 lib/scope_gate.py engagements/<slug>/ --tool-flags <tool-name>
```

Returns flags to inject into the tool command:

| Tool | Rate Flag | No-Redirect Flag |
|------|-----------|-------------------|
| nuclei | `-rl <N>` | `--no-follow-redirects` |
| ffuf | `-rate <N>` | `-r` |
| nmap | `--max-rate <N>` | N/A |
| httpx | `-rl <N>` | `--max-redirects 0` |
| naabu | `-rate <N>` | N/A |
| nikto | `-Pause <N>` | N/A (no redirects by default) |
| katana | `-rl <N>` | N/A |

## No-Redirect Policy

HTTP tools MUST NOT follow redirects to prevent scope escape:
- A 302 from in-scope to out-of-scope domain = scope violation
- Use no-redirect flags on all HTTP scanning tools
- If redirect following is needed, check the redirect target against scope first

## Custom Headers

Some programs require specific headers on all requests:
```bash
cd ~/engagement-kit && python3 lib/scope_gate.py engagements/<slug>/ --tool-flags <tool-name>
```
Returns `-H "X-Bug-Bounty: True"` etc. Inject into all HTTP tool calls.

## Audit Logging

All tool invocations are logged to `<engagement>/logs/audit.jsonl`.
The scope_gate.py `gate_tool()` function handles this automatically.

## DRY_RUN

Check the Makefile: `DRY_RUN ?= 1` means no active testing.
Only the kickoff phase can flip this, and only with explicit user approval.

## Phase State

Never skip phases. The phase_state.py enforces:
- Recon requires kickoff_complete
- Scan requires recon_complete
- Triage requires scan_complete
- Exploit requires triage_complete
- Writeup requires exploit_complete

## Financial Limits

Read `admin/roe.md` for program-specific financial limits.
Example: Robinhood limits test transactions to $1,000 USD.
Do NOT exceed financial limits during exploitation.

## Tool Timeouts

Long-running tools (nmap, subfinder) must use timeouts:
- Bash tool: use `run_in_background: true` for operations > 5 minutes
- Split large target lists into batches of 50 hosts
- Use the `timeout` command: `timeout 600 nmap ...`
