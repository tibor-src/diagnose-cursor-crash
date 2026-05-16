# Analysis patterns

Concrete log lines, what they mean, and what to do. Use **ripgrep** (`rg`) or any grep across the unzipped session folder.

## Noise to skip (don't quote as evidence, don't burn tokens reading)

These show up constantly in healthy sessions too. Filter them out unless they're the **only** thing happening at the crash time:

| Pattern | Where | Why it's noise |
|---|---|---|
| `[otel.error] ... OTLPExporterError: Bad Request ... Trace spans collection is not enabled for this user` | `renderer.log` | Telemetry endpoint expected response for users not opted in. |
| `updateWindowsJumpList#setJumpList unexpected result: customCategoryAccessDeniedError` | `main.log` | Windows jump-list permission, unrelated to crashes. |
| `Missing property "rpcFileLoggerFolder" in oldValue. Filling with value from initValue.` | `renderer.log` | Internal config migration warning. |
| `Via 'product.json#extensionEnabledApiProposals' extension '...' wants API proposal '...' but that proposal DOES NOT EXIST.` | `renderer.log` | API surface drift; not a crash signal. |
| `Cannot register two commands with the same id: workbench.action.show...` | `renderer.log` | Cosmetic startup warning. |
| `update#setState idle/checking/downloading/downloaded` | `main.log` | Updater state — see Pattern 11 for the rare case it matters. |
| `extHostTelemetry.log` entirely | per-window | Telemetry plumbing, no diagnostic value. |
| `[lifecycle]: Process participant started (id: ...)` (info, not error) | `renderer.log` | Normal startup chatter. |

If your `rg` matches are dominated by these, narrow the pattern. Quoting them in the verdict wastes user-attention and tokens.

## 1. Renderer process gone (the crash event)

**File:** `main.log`

```
[error] CodeWindow: renderer process gone (reason: oom, code: -536870904)
```

`reason=` is the first-class diagnosis:

| reason | Meaning | Common cause |
|---|---|---|
| `oom` | Out of memory | Heavy agent loop, large workspace, many extensions, in-renderer MCP fallback |
| `crashed` | Native crash (C++ / V8) | Specific extension bug, Electron bug — pair with Crashpad dump |
| `killed` | OS killed the process | Windows OOM-killer-equivalent, macOS jetsam, Linux OOM killer, AV/EDR termination |
| `launch-failed` | Renderer never started | Sandbox / GPU init issue at startup |

`code=` decoding:

- `-536870904` = `0xE0000008` = `STATUS_NO_MEMORY` (Windows)
- `-1073741819` = `0xC0000005` = access violation (Windows)
- `-9` = `SIGKILL` (Unix, OS killed it)
- `-11` = `SIGSEGV` (Unix segfault)

## 2. GPU process

**File:** `main.log`

```
[error] gpu-process-crashed
```

Have user relaunch with `--disable-gpu`:

- Windows: `"C:\Program Files\cursor\Cursor.exe" --disable-gpu`
- macOS: `open -a Cursor --args --disable-gpu`
- Linux: `cursor --disable-gpu`

If stable → GPU driver / Chromium-GPU issue. Collect `dxdiag` (Windows) or `system_profiler SPDisplaysDataType` (macOS).

## 3. Hang / unresponsive (no `process gone`)

**File:** `main.log`

```
[error] CodeWindow: recovered from unresponsive
```

Or in `renderer.log`:

```
[info] Extension host (LocalProcess pid: <PID>) is unresponsive.
[info] Extension host (LocalProcess pid: <PID>) is responsive.
```

A single unresponsive/responsive cycle is normal under load. **Repeated cycles or "recovered from unresponsive" lines** indicate a real hang. Need a DevTools Performance recording or heap snapshot to root-cause.

## 4. MCP host (out-of-process) failure → in-renderer fallback

**File:** `main.log`

```
[error] [McpProcess] ipcReady wait failed [McpProcess] timed out waiting for ipcReady after 10000ms
[warning] [McpProcess] crash detected (attempt N), retrying in <ms>ms
[error] [McpProcess] giving up after 6 crashes within 600000ms; falling back to legacy in-window MCP
```

**File:** `renderer.log`

```
[error] [lifecycle]: Process participant start failed (id: mcp-process-connection, ms: ...)
  Failed to acquire MessagePort for response channel 'vscode:createMcpProcessChannelConnectionResult'
```

**What it means:** the separate MCP process Cursor uses to isolate MCP servers can't start. Every MCP server gets loaded inside the renderer instead. With a large MCP fleet, this dramatically raises renderer memory and crash risk.

**Remediation:** fully quit Cursor from the tray / dock and reopen. If it recurs every launch, ask for the user's antivirus / EDR (often blocks the helper process spawn) and any sandboxing settings.

## 5. MCP fleet size

**File:** `window*/exthost/anysphere.cursor-agent-exec/*.log`

```
[info] fetchAndWrite: lease returned 96 tools across 15 clients
[info] Incremental write: ... "totalServerCount":15 ...
```

`clients` = MCP servers. `tools` = total tool count exposed across them.

Heuristics:

- **≤ 5 servers, ≤ 30 tools** — normal.
- **6–10 servers, 30–60 tools** — heavy but OK if MCP host is working.
- **11+ servers or 60+ tools** — if MCP host has failed (Pattern 4), this is the OOM accelerator.

The same log lists each server in `Incremental write` lines with `serverIdentifier` — use that to recommend which ones to disable.

## 6. Individual MCP server crashing

**File:** `window*/exthost/anysphere.cursor-mcp/MCP <server-name>.*.log`

Signs of a misbehaving server:

- Repeated stderr / stack traces.
- Frequent restarts (timestamps clustered seconds apart).
- `exit code 1` / `exit code 137` (OOM) / signal numbers.
- Auth / 401 / 403 spam.

Remediation: disable that one server in Settings → MCP. Capture the full server log for whoever maintains it.

## 7. Extension host crash

**File:** `main.log`

```
[info] Extension host with pid <PID> exited with code: <N>, signal: <S>.
```

`code: 0, signal: unknown` after a window close is normal. Non-zero code or non-`unknown` signal **before** the crash event matters. Cross-reference `window*/exthost/exthost.log` for the stack trace, and per-extension subfolders for which extension was active.

## 8. Listener leaks — usually Cursor-internal

**File:** `renderer.log`

```
[error] [00c] potential listener LEAK detected, having 200 listeners already.
  at <stack frames, top-down>
```

By itself: not the crash. Followed by an OOM (Pattern 1): the leak is a meaningful contributor or root cause.

**Identify the source.** Read the stack frames immediately following the leak warning. The top *meaningful* frame — typically the first one whose function name describes a feature (e.g. something ending in `loadInlineDiffs`, `renderModel`, `resolveReferences`, etc.) rather than generic event-emitter plumbing — tells you which subsystem is leaking.

**Classify.**

- **Frame is inside Cursor's own code paths** (any minified or named function under the Cursor workbench, agent, or core extensions): treat as Cursor-side. The user has no lever to disable normal Cursor rendering or model behavior. Move this to "email Cursor" with the top stack frame quoted as evidence.
- **Frame is inside a clearly-third-party extension** (recognizable extension id in the stack or path): treat as that extension's bug; user can reproduce with `--disable-extensions` to confirm, then disable or report to that extension's author.

In neither case should you tell the user to "avoid" the underlying Cursor feature as a workaround — see SKILL.md "Recommendation hygiene".

## 9. Agent loop correlation

**File:** `renderer.log`

```
[info] [ComposerWakelockManager] Acquired wakelock id=0 reason="agent-loop" composerId=<uuid>
[info] [buildRequestedModel] composerId=<uuid> catalogModelId=gpt-5.5 ... maxMode=true ...
```

If the crash time is between a wakelock acquire and (missing) release, the crash happened **during** an agent loop. Note `maxMode=true` — Max Mode roughly doubles the working memory footprint vs normal mode and is a common contributor to OOM.

## 10. OTel exporter "Bad Request"

**File:** `renderer.log`

```
[error] [Extension Host] [otel.error] {"stack":"OTLPExporterError: Bad Request ...
  "data":"{\"error\":\"Trace spans collection is not enabled for this user\"}"}
```

**Noise** — not a crash signal. Ignore unless it's the only error and replaces actual content.

## 11. Updater chatter

**File:** `main.log`

```
[info] update#setState checking for updates
[info] update#setState downloading
[info] update#setState downloaded
```

Informational. Not a crash signal, but worth noting if `downloaded` happens immediately before a crash — a pending restart may have been involved.

## 12. Crashpad dump present but no log explanation

If `Crashpad/completed/*.dmp` exists with a timestamp matching the crash but `main.log` only shows `renderer process gone reason: crashed`, the dump itself is the next artifact a developer needs. The user can't symbolicate it; forward as-is.

## Report template

When summarising to the user, use this shape. The first three blocks are the diagnosis; the last two split self-fix from "send to Cursor".

```markdown
**Verdict:** <one sentence, e.g. "Renderer out-of-memory during a Max Mode agent loop on Cursor 3.4.17 (Windows)">.

**Evidence:**
- `main.log` 11:34:41 — `CodeWindow: renderer process gone (reason: oom, code: -536870904)`
- `main.log` 11:25:39 — `[McpProcess] giving up after 6 crashes within 600000ms; falling back to legacy in-window MCP`
- `anysphere.cursor-agent-exec/*.log` — `lease returned 96 tools across 15 clients`
- `renderer.log` 11:26:42 — `[buildRequestedModel] ... gpt-5.5 ... maxMode=true`

**Contributing factors (priority order):**
1. MCP host failed to start → all 15 MCP servers loaded in renderer.
2. Heavy agent loop with Max Mode pushed memory over the limit.
3. <other, e.g. specific extension, large workspace>.

**Try yourself (in order):**
1. Fully quit Cursor from the tray / dock and reopen — restores the MCP host on most machines.
2. If still crashing, open Settings → MCP and disable servers you don't need right now (heaviest first: <list from Pattern 5>).
3. Retry the run with Max Mode off.

**If that doesn't fix it, email Cursor.** Send the logs zip to **hi@cursor.com** with this body:

> Subject: Cursor crash - renderer OOM during agent loop on Windows
>
> Hi Cursor team,
>
> I'm hitting a reproducible crash. Logs attached.
>
> Version: <paste Help > About block>
> OS: <OS + version>
> Crash time: <local time + timezone>
>
> What I did: <2-3 lines>
> What I observed: renderer process gone (oom)
> What I tried: <list of "try yourself" steps and result>
>
> Thanks,
> <name>
```

Keep it that short. Users are scanning, not reading.
