# diagnose-cursor-crash

A [Cursor Agent Skill](https://docs.cursor.com/) that helps a regular Cursor user diagnose IDE crashes, hangs, freezes, renderer-gone errors, and MCP / Node subprocess failures — without needing access to the Cursor source code.

The agent walks the user through:

1. **Capturing** the right logs the first time (Developer Tools console open before reproducing, version block, local timestamp).
2. **Collecting** session logs, Crashpad dumps, and native OS crash dumps (Windows WER / macOS DiagnosticReports / Linux coredumps).
3. **Analyzing** them against a catalog of known failure modes: renderer OOM, MCP host failure with in-renderer fallback, GPU process crash, extension host crash, individual MCP server crash, hangs, listener leaks, agent-loop correlation.
4. **Acting on the result**, with a clear split between:
   - **What the user can fix themselves** right now — e.g. fully quit and reopen Cursor, disable unused MCP servers in Settings → MCP, retry with Max Mode off, relaunch with `--disable-gpu`, bisect extensions with `--disable-extensions`.
   - **When to email Cursor** — if self-fix steps don't resolve it, the crash points at a first-party Cursor component, or the data would help Cursor fix the underlying bug, the skill produces a pre-filled email body and tells the user to send the logs zip to **hi@cursor.com**.

## What's inside

| File | Purpose |
|---|---|
| [`SKILL.md`](SKILL.md) | Entry point — phased workflow (Capture / Environment / Version check / Reproduce / Save DevTools / Collect / Analyze + report), safety rules, cost & mode guidance, recommendation hygiene, and evidence calibration so the agent doesn't invent causes. |
| [`log-locations.md`](log-locations.md) | Exact paths per OS for session logs, Crashpad, WER, DiagnosticReports, coredumps. Includes the elevated-PowerShell snippet to enable Windows user-mode dumps. |
| [`analysis-patterns.md`](analysis-patterns.md) | 12 concrete log-line patterns with example lines, file locations, what they mean, and what to do. Plus a noise-to-skip table and a report template. |
| [`anonymize-logs.md`](anonymize-logs.md) | Optional, on user request: per-OS scripts to scrub usernames / hostnames / workspace paths / emails / repo URLs from a copy of the logs without the agent ever reading the log content. |

## Install

Clone into your personal Cursor skills directory:

```bash
git clone https://github.com/tibor-src/diagnose-cursor-crash.git \
  ~/.cursor/skills/diagnose-cursor-crash
```

The skill is then auto-discovered by Cursor. To trigger it explicitly:

```
Run /diagnose-cursor-crash on this logs zip: /path/to/logs.zip
```

It also auto-loads when the agent sees keywords like *crash, freeze, renderer process gone, MCP failure, plan tool crash, agent loop crash,* or when handed a Cursor logs zip.

## License

MIT.
