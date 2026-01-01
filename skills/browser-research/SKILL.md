---
description: Web + browser research loop using built-in web search and/or Chrome DevTools MCP.
argument-hint: QUESTION="<what to research>" USE_BROWSER=true|false
---

Research: $QUESTION

Rules:

-   Prefer built-in web search for docs and up-to-date facts.
-   If USE_BROWSER=true, use Chrome DevTools MCP for anything requiring a real browser session (auth, dynamic UI, screenshots).
-   Summarize with:
    1. key findings
    2. relevant links
    3. recommended next steps / commands
-   Keep output actionable and concise.
