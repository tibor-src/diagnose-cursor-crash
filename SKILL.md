---
name: diagnose-cursor-crash
description: >-
  Diagnose any Cursor IDE crash, hang, freeze, renderer-gone error, MCP / Node
  subprocess failure, GPU crash, extension host crash, agent loop crash, or
  startup failure from a regular user's machine (no Cursor source access
  required). Read-only and advisory: collects environment info, verifies the
  user is on the latest published Cursor version (the team only supports
  the latest) before doing anything else, walks the user through capturing
  logs, analyzes them, and produces a verdict plus a split between
  user-actionable mitigations and "email hi@cursor.com with the logs".
  Never modifies, moves, or deletes user files; never auto-edits Cursor
  settings; always backs up and exports chats before any destructive-
  sounding step. Use when the user reports a Cursor crash,
  freeze, unresponsive window, "Aw, snap" / "renderer process gone", plan
  or agent crash, MCP unavailable, repeated reloads, Cursor failing to
  start, or wants help interpreting a Cursor logs zip they already
  collected.
---

# Diagnose Cursor crash

A regular Cursor user (no access to Cursor source) can run this end-to-end. You either help them **collect** logs from a live crash, or **analyze** a logs zip they already sent.

## Cost / mode guidance (read first)

Log analysis is a tight, linear procedure — find the crash, walk backwards, name the factors, recommend. It does **not** need plan mode and does **not** need Max Mode. Both just burn credits for no quality gain.

- **Run in normal agent mode** with the default model.
- **Do not switch to plan mode.** There is no design decision to scope; this skill already prescribes the steps.
- **Do not enable Max Mode.** The synthesis is short; the reasoning lift is small.

Token discipline during analysis (Phase 5):

1. **Grep first, Read second.** Use `rg` (or `grep -E`) to find specific patterns. Only `Read` a file once you've confirmed it contains the lines you need, and only the relevant range with `offset` / `limit`.
2. **Three files cover ~90% of cases**: the session's `main.log`, the crashing window's `renderer.log`, and one `anysphere.cursor-agent-exec/*.log` for MCP fleet size. Start there. Pull more only if the verdict isn't clear.
3. **Bound large files**: `rg -n -B2 -A5 '<pattern>' path/to/file` returns just the matches with context — usually a few hundred bytes.
4. **Stop when you have a verdict**. The report template only needs 2–4 evidence lines. Once you have them, write the report. Don't keep grepping.
5. **Skip noise**: `OTLPExporterError`, `updateWindowsJumpList`, `extHostTelemetry`, repeated `Missing property "rpcFileLoggerFolder"` warnings — see [analysis-patterns.md](analysis-patterns.md) for the noise list. Don't quote them as evidence.

A typical run of this skill on a real crash zip should fit comfortably in a single normal-mode turn.

## Safety rules (read before any action)

This skill is **read-only and advisory**. Its job is to read logs and tell the user what's happening, not to change the user's machine.

### Rules for the agent running this skill

1. **Never modify, move, rename, or delete any file on the user's machine.** Reading logs, copying logs into a separate scratch directory for analysis, and creating a new analysis report file in a path you proposed (and the user accepted) are all fine. Touching anything inside the user's Cursor data directory, settings, extensions, workspaces, profile, or home folder is **not**.
2. **Never run destructive shell commands** (`rm`, `del`, `rmdir /s`, `Remove-Item`, `mv` over an existing target, `> file` redirections that truncate, `git clean`, `git reset --hard`, etc.) against any path under the user's home directory or Cursor install location.
3. **Never auto-apply settings changes**, edit `settings.json`, change MCP config files, uninstall extensions, or change file associations. If a setting change is part of the recommendation, instruct the user to do it from the Cursor UI and explain how to reverse it.
4. **Never advise the user to delete a folder, profile, or cache as a "quick fix"** without first stepping them through the backup-and-recovery plan below. Cursor's user-data directory contains chat history, settings, extensions, and per-workspace state that the user almost always wants back.

### Rules for advice given to the user

Some genuine fixes (reinstalling Cursor, starting from a fresh user-data directory, resetting an extension) involve removing or replacing files. When you recommend any of these:

1. **Tell the user to export their chats first.** See "Export your chats before destructive actions" below for the exact steps.
2. **Tell the user to back up the relevant folder first**, not just trust they can re-download it. Give the exact backup command for their OS (e.g. `cp -R "$HOME/Library/Application Support/Cursor" "$HOME/Cursor-backup-$(date +%Y%m%d)"` on macOS; `Copy-Item -Recurse "$env:APPDATA\Cursor" "$env:USERPROFILE\Cursor-backup-$(Get-Date -Format yyyyMMdd)"` on Windows).
3. **Tell them how to restore** if the fix doesn't help (delete the new folder, copy the backup back in place, restart Cursor).
4. **Prefer non-destructive variants first**: launching with `--user-data-dir <empty temp dir>` lets the user test a clean profile *without touching* their real one. Only escalate to "replace the real one" if the clean-profile test confirms the real profile is the problem.

### Discovery & verification (use built-in lookup, don't guess menu paths)

Cursor menu names, setting paths, and keyboard shortcuts drift between releases. Do not hardcode them in your advice. Use the lookup tools every Cursor user already has:

1. **`cursor-guide` subagent** (the agent's tool — every Cursor install has it): call this whenever you're about to recommend a specific Cursor setting, menu item, command, or recovery flow and you're not 100% sure of the current wording. It reads live Cursor product documentation and returns the current name + path. Cheap; always do it instead of guessing.
2. **`@Docs` in chat** (the user's tool): tell the user they can type `@Docs` in the composer to search Cursor's indexed documentation when they can't find a setting you mentioned.
3. **`@Web` in chat**: tell the user they can type `@Web` for live web search when something is too new for indexed docs.
4. **Command Palette** (`Ctrl/Cmd+Shift+P`): the canonical discovery surface. Prefer "open the Command Palette and type X" over "Settings → Foo → Bar → Toggle" phrasing — it works across versions even when menu names change.
5. **Settings search** (`Ctrl/Cmd+,` → search box) and **Cursor Settings** (`Cursor Settings` in the Command Palette): two separate panels — VSCode-style settings and Cursor-specific settings live in different places.

If `cursor-guide` and the user's own search both fail to find a setting you described, the setting may have been removed or renamed; say so honestly and ask the user what they see in their UI, instead of inventing a path.

### Export your chats before destructive actions

If you are about to recommend reinstalling Cursor, clearing the user-data directory, or anything that could lose chat history, walk the user through exporting their important conversations first.

**Before telling the user how to export, verify the current flow with `cursor-guide`** — the export UI moves between versions, and an out-of-date instruction wastes the user's time. The general shape across versions:

1. In Cursor, open the chat you want to keep.
2. Use the chat's overflow / actions menu to find the export option (currently usually labelled "Export"; `cursor-guide` will confirm the exact wording and location). If the user can't find it, ask for a screenshot of the chat header and the Command Palette results for `export` instead of guessing.
3. Save the export to a folder **outside** the Cursor user-data directory (e.g. `~/Documents/cursor-chats/`).
4. Repeat for every chat they want to preserve.

If **Cursor will not start at all** and the user needs to recover chat content first:

1. Do **not** delete or reinstall yet.
2. Copy the entire Cursor user-data folder to a backup location before doing anything else:
   - Windows: `%APPDATA%\Cursor\` → copy to `%USERPROFILE%\Cursor-backup-YYYYMMDD\`
   - macOS: `~/Library/Application Support/Cursor/` → copy to `~/Cursor-backup-YYYYMMDD/`
   - Linux: `~/.config/Cursor/` → copy to `~/Cursor-backup-YYYYMMDD/`
3. Tell the user this folder contains their chats, settings, and extension state, and that the Cursor team (via `hi@cursor.com`) can guide them through extracting individual conversations from the backup if reinstalling doesn't restore them.

Only after backups are confirmed should you suggest a destructive step.

## When to use which phase

```
User reports a crash, freeze, or MCP failure
        │
        ├── No logs yet ─────────────► run Capture → Reproduce → Collect → Analyze
        ├── Has a logs zip already ──► skip to Analyze
        └── App won't start at all ──► skip to "Native OS crash dumps" in log-locations.md
```

## Phase 1 — Capture (BEFORE reproducing)

Do these **before** reproducing so the crash session is what gets captured. If the crash already happened and they can't reproduce, skip to Phase 4 — the most recent dated logs subfolder still has it.

1. **Open the Developer Tools console** (DevTools is the highest-signal artifact for renderer crashes):
   - Press `Ctrl+Shift+I` (Windows / Linux) or `Cmd+Opt+I` (macOS), **or**
   - Open the Command Palette (`Ctrl/Cmd+Shift+P`) and search for `Developer Tools` — pick the toggle entry.
   - Click the **Console** tab. Leave it open. (If the exact command name has changed in your version, `cursor-guide` or `@Docs` will surface the current name.)
2. **Note the local time** you are about to reproduce (timestamp + timezone).
3. **Capture the environment snapshot** — see Phase 1b below. Do this *before* reproducing so memory / process state reflects the pre-crash baseline. (For the About block specifically, the Command Palette path described in Phase 1b works regardless of where the menu sits in the current Cursor version.)

## Phase 1b — Environment snapshot (mandatory for a useful ticket)

This is the difference between a ticket a Cursor engineer can act on and one that bounces back asking for more info. Collect **all of these** even if you think only one matters — the combination is what disambiguates.

### Verbatim only — do not summarize

Every quoted artifact (About block, log lines, error messages, command output) goes into the ticket **as-is**. Do not paraphrase, abbreviate, redact (unless the user asked — see [anonymize-logs.md](anonymize-logs.md)), reorder fields, or "just send the version number." Summaries lose the exact field values (commit hash, Electron build, OS patch level, etc.) that the team uses to correlate with internal builds and release notes. If the user gives you a summary instead of the raw text, ask for the raw text.

### Cursor identity
- **About block** — open the About dialog, then click the **Copy** button (or select-all + copy if Copy is missing) and paste the whole block verbatim. There are several ways to open the About dialog and the menu wording moves between versions; offer the user **all three** and let them use whichever works in their build:
  1. **Command Palette** (most reliable across versions): `Ctrl/Cmd+Shift+P` → type `About` → pick the entry. Works even when the menu bar is hidden.
  2. **Menu bar — macOS**: `Cursor` (app menu, top-left next to Apple logo) → `About Cursor`.
  3. **Menu bar — Windows / Linux**: `Help` → `About`. If the menu bar is hidden on Windows, press `Alt` once to reveal it, or look for a hamburger / kebab menu in the title bar that contains a `Help` submenu.
  
  Required fields in the pasted block: `Version`, `Commit`, `Electron`, `Chromium`, `Node.js`, `V8`, `Layout`, `Build Type`, `Release Track`, **and the `OS:` line** including the build number. Do **not** trim to just the version — every other field carries triage signal. If the current build's About dialog is missing one of these fields or labels them differently, paste what is shown rather than guessing the field name.
- **Install type**: system install vs user install vs portable. On Windows, look at the Cursor.exe path:
  - `C:\Program Files\cursor\Cursor.exe` → system install
  - `C:\Users\<name>\AppData\Local\Programs\cursor\Cursor.exe` → user install
- **Where they installed from**: cursor.com download, winget, Chocolatey, Homebrew Cask, distro package, AppImage, other.
- **Auto-updated recently?** Did the crash start after a Cursor update in the last few days? `main.log` has `update#setState downloaded` events — they can tell you.

### Machine
- **OS full version + build**: 
  - Windows: `winver` (shows e.g. "Windows 11 24H2 OS Build 26200.xxxx")
  - macOS: Apple menu → About This Mac (e.g. "macOS 15.4 (24E248)")
  - Linux: `cat /etc/os-release` + `uname -r`
- **CPU**: model + cores. Windows: `wmic cpu get name` or Task Manager → Performance. macOS: `sysctl -n machdep.cpu.brand_string`. Linux: `lscpu | head`.
- **Total RAM and free RAM at the time of crash**:
  - Windows: Task Manager → Performance → Memory tab; note **In use** and **Available** numbers.
  - macOS: Activity Monitor → Memory → Memory Pressure + Physical Memory.
  - Linux: `free -h`.
- **Free disk space** on the volume holding the user's home folder.

### Security / sandboxing software (critical for MessagePort / helper-process failures)
- **Antivirus / EDR product** and version: Trend Micro, Symantec, Sophos, Crowdstrike, SentinelOne, Microsoft Defender for Endpoint, ESET, McAfee, etc. Real-time protection on?
- **Has Cursor been excluded** from AV / EDR scanning? (Install folder + `Cursor.exe`.)
- **Corporate-managed device?** MDM / Intune / Jamf? Any application allowlisting (e.g. AppLocker, WDAC on Windows; Gatekeeper / MDM profiles on macOS)?

### What the user was doing
- **Reproducibility**: every time / most attempts / intermittent / one-time.
- **Specific trigger**: which tool, which model, Max Mode on or off, plan mode on or off, agent loop, MCP tool call (which one), pasting a large file, opening a specific workspace, etc.
- **Workspace size hint**: rough file count, is it indexed, is it on a network drive / OneDrive / WSL path?

### Request IDs (mandatory if the crash involved chat / agent activity)

If the crash happened during a chat message, agent loop, plan tool, tool call, model response, or any AI activity — get at least one **request ID** for the failing request. This is the single most useful identifier for the team to find the failed call in backend traces; without it, log triage relies on timestamp + user correlation which is much slower.

- **Where it lives in the UI:** Cursor exposes a request ID on each AI message — currently usually via the message's overflow / actions menu ("Copy Request ID" or similar). The exact wording moves between versions; **verify with `cursor-guide`** before instructing the user, and ask them to screenshot the message header if they can't find it.
- **Composer / chat ID is also useful** but is not a substitute. From `renderer.log` you can grep `[ComposerWakelockManager]` and `[buildRequestedModel]` for the `composerId=` value when the user can't extract it from the UI.
- **DevTools console** captures request IDs in network entries when reproducing with the console open (Phase 1).
- **Multiple IDs are fine and often better** — if several requests failed, include all of them.

For non-chat-related crashes (startup failure, GPU process crash, MCP host failure before any chat ran), request ID is not applicable; say so explicitly in the email instead of leaving the field blank.

### MCP and extension footprint
- **MCP servers enabled** at the time of crash: list them by name (Settings → MCP, screenshot is fine). Don't trust "I disabled them" — Cursor's own logs (`anysphere.cursor-agent-exec/*.log`) record what was actually loaded.
- **Third-party extensions installed**: Command Palette → `Extensions: Show Installed Extensions` → screenshot, or `cursor --list-extensions` in a terminal and paste output.
- **Tried `--disable-extensions`?** Yes / no / and the result.
- **Tried a fresh user-data dir?** (`cursor --user-data-dir <empty temp dir>` to rule out a corrupted profile.) Yes / no / and the result.

### Optional but high-signal
- **Screen recording** of the crash, especially for hangs (~10–30 s, no narration needed). Reveals which UI action triggered it when logs are ambiguous.
- **Other machine test**: does the same workspace / same prompt crash on a *different* machine? If no, it's environment-specific. If yes, it's a Cursor / workspace bug.

## Phase 1c — Version check (hard gate before reproducing)

The Cursor team supports the **latest** version on each release track. If the user is on an outdated build, the crash may already be fixed and the rest of this skill is wasted effort. Always run this gate before proceeding to Phase 2.

### Step 1 — Get the user's current version and release track

From the About block they pasted in Phase 1b, extract three things:

- **Version** (e.g. `3.4.17`).
- **Build Type** (e.g. `Stable`).
- **Release Track** (e.g. `Stable` / `Nightly`).

The Build Type / Release Track combination matters: e.g. `Build Type: Stable, Release Track: Nightly` means the user opted into faster delivery of stable builds — they will move ahead of plain stable but should still be on a recent build.

### Step 2 — Get the latest published version

Fetch `https://cursor.com/download` (or the changelog page if download lists installer URLs only) and identify the current latest **per release track**. Use whichever lookup is available:

- **`WebFetch`** on `https://cursor.com/download` — extract the version string.
- **`cursor-guide`** subagent — ask it for the current Cursor version on the user's release track.
- **`@Web`** in chat — the user can do the lookup themselves and paste the result.

If you cannot reach any of those (offline, restricted network), say so explicitly and ask the user to share what `cursor.com/download` shows in their browser. Do not invent a "latest" version from training data — versions drift weekly.

### Step 3 — Compare and act

| User state | Action |
|---|---|
| On the latest version for their track | Proceed to Phase 2. Note this in the email body ("Confirmed on latest version `<X.Y.Z>` on `<track>` channel as of `<date>`"). |
| One or more versions behind on their track | **Stop. Tell the user to upgrade first**, then re-test. If the crash no longer reproduces, no email is needed. If it still reproduces, proceed with the (now-current) logs. |
| On Nightly / pre-release | Note that bugs on Nightly may already be known or expected; offer them the choice of (a) updating to the latest Nightly and re-testing, or (b) switching to the Stable channel for stability. Either way, re-test after the change before sending logs. |
| Cannot determine latest (offline) | Tell the user the team only supports the latest version; ask them to confirm via `cursor.com/download` before sending logs. |

### Step 4 — How to upgrade (safe & reversible)

Walk the user through upgrading using only reversible steps. Verify the exact menu wording with `cursor-guide` before naming it — the update UI moves between versions.

- **Trigger an update from inside Cursor:** open the Command Palette (`Ctrl/Cmd+Shift+P`) and search for `Check for Updates` (or similar). If an update is available, Cursor downloads it and prompts for a restart. **To revert:** Cursor keeps the previous version installer locally and the team can guide a rollback via `hi@cursor.com` — but updates almost never need to be rolled back, and downgrading is not officially supported.
- **Enable auto-updates if they were turned off:** in the Command Palette, search for `update` and look for settings like `update.mode` (VSCode-style) — the user can switch to `default` for normal auto-updates. **To revert:** change `update.mode` back. `cursor-guide` will confirm the current setting name.
- **Manual download:** if the in-app updater fails, download the latest installer for their OS from `cursor.com/download` and run it over the existing install. Tell them their settings, extensions, and chats are preserved across an in-place upgrade — but still recommend the Phase 6 "Export your chats before destructive actions" workflow as belt-and-suspenders before any reinstall.

### Step 5 — Re-test before sending logs

After upgrading, ask the user to reproduce the crash once more. Two outcomes:

- **Crash no longer reproduces.** The bug was already fixed. No email needed. Tell the user.
- **Crash still reproduces.** Now go to Phase 2 (Reproduce) on the new version. The logs you collect will be from the latest version and will actually be useful to the team.

This gate exists because Cursor moves fast; "I'm on `3.4.17`" can mean "I'm on the version from two days ago" or "I'm on the version from two months ago" depending on the week. Confirming you're testing against the latest build is the single highest-leverage thing the user can do before involving the team.

## Phase 2 — Reproduce

Reproduce the crash **once**. If the app hard-crashes / closes, that's expected — Phase 4 still finds it.

Write down exactly what action triggered the crash so it can be pasted into the "Specific trigger" line of Phase 6's email.

## Phase 3 — Save the DevTools console

If DevTools was open and survived: right-click anywhere in the Console panel → **Save as…** → name it `devtools-console.txt`. If the renderer fully crashed, DevTools dies with it; that's fine, the logs folder still has the renderer log.

## Phase 4 — Collect logs

Read [log-locations.md](log-locations.md) for the full per-OS path list. At minimum collect:

- The **most recent dated subfolder** inside Cursor's `logs/` directory (one folder per session — pick the one whose timestamp matches the crash).
- Anything inside Cursor's `Crashpad/completed/` directory with a timestamp around the crash.
- OS-level crash dumps (Windows WER, macOS DiagnosticReports, Linux coredumps) — only present if the **whole app** died, not just the renderer.
- `devtools-console.txt` from Phase 3 if available.
- The full **Phase 1b environment snapshot** as a text file or pasted into the email body.

Zip everything together. Do **not** prune subfolders inside the session logs directory — `exthost/`, `output_*/`, and per-extension subfolders all matter.

## Phase 5 — Analyze

If the user sent a zip, unzip to a scratch directory first, then `cd` there. Run the commands below verbatim — they finish the triage with ~5 grep calls and 0–1 targeted Read. Don't open whole files until a grep has told you they're worth opening.

### Step 1 — Find the crash event (one grep over main.log)

```bash
rg -n 'renderer process gone|gpu-process-crashed|Extension host with pid .* exited|recovered from unresponsive|is unresponsive' main.log
```

Decode the match:

- `renderer process gone (reason: ...)` — renderer died. The `reason` (`oom`, `crashed`, `killed`, `launch-failed`) is the first diagnosis. The numeric `code:` tells you more — see [analysis-patterns.md](analysis-patterns.md#1-renderer-process-gone-the-crash-event) for the table.
- `gpu-process-crashed` — GPU driver issue.
- `Extension host with pid ... exited with code:` non-zero before the crash time — extension host died; cross-reference its `exthost.log`.
- `recovered from unresponsive` / `is unresponsive` repeated — hang, not a crash.

The first match's timestamp anchors everything that follows.

### Step 2 — Walk backwards ~5 minutes (one grep over main.log + renderer.log)

```bash
rg -n 'McpProcess|MessagePort|ComposerWakelockManager|buildRequestedModel|Extension host.*unresponsive' main.log window*/renderer.log
```

What to extract from the matches:

- `[McpProcess] ipcReady wait failed` / `crash detected (attempt N)` / `giving up after N crashes` (in `main.log`) — the out-of-process MCP host is broken; every MCP server falls back into the renderer process.
- `Failed to acquire MessagePort for response channel 'vscode:createMcpProcessChannelConnectionResult'` (in `renderer.log`) — same root cause, renderer side.
- `[ComposerWakelockManager] Acquired wakelock ... reason="agent-loop"` plus the matching `[buildRequestedModel] ... catalogModelId=... maxMode=true|false` — names the composer (`composerId=`) and model that were running when the crash hit. **Extract the `composerId` value for the email.**
- Repeated `Extension host ... is unresponsive` / `is responsive` cycles — extension host thrashing.

If none of these show up before the crash timestamp, the crash is **not** MCP / agent-loop related — go to Step 4.

#### Step 2b — Extract request IDs (chat / agent crashes only)

If the crash happened during AI activity and the user could not copy a request ID from the UI, pull what you can from the logs:

```bash
rg -nIo 'request[_-]?id"?\s*[:=]\s*"?[A-Za-z0-9_-]+' window*/exthost/anysphere.cursor-agent-*/*.log | sort -u | tail -20
rg -nIo 'composerId=[A-Za-z0-9_-]+' window*/renderer.log | sort -u
```

Report whatever you find. Note explicitly to the user when **no** request ID could be extracted — backend search by request ID is much faster than by timestamp; if the user has any IDs available later (in their UI), they should add them to the email.

### Step 3 — Quantify the MCP fleet (only if Step 2 showed MCP host failure)

```bash
rg -hN 'lease returned' window*/exthost/anysphere.cursor-agent-exec/*.log \
  | awk -F'returned ' '{print $2}' | sort -u | tail -5
```

You want the **peak** `N tools across M clients`. Thresholds:

- ≤ 5 servers, ≤ 30 tools: normal.
- 6–10 / 30–60: heavy but OK if MCP host is working.
- ≥ 11 servers or ≥ 60 tools **plus** Step 2's in-renderer fallback: classic renderer-OOM accelerator.

To name the specific servers (for "disable these first" advice):

```bash
rg -hN 'Incremental write' window*/exthost/anysphere.cursor-agent-exec/*.log \
  | rg -oE 'serverIdentifier":"[^"]+' | sort -u
```

### Step 4 — Per-server crashes (only if Step 2 didn't explain it)

```bash
rg -n 'exit code|stderr|Error|crash' window*/exthost/anysphere.cursor-mcp/MCP*.log | head -50
```

Look for repeated stderr, non-zero exit codes, schema errors, or auth failures from one named server.

### Step 5 — Synthesize (with evidence calibration — mandatory)

You should now have 2–4 quoted log lines and a clear picture. Before writing the verdict, run the evidence-calibration check below. Then write the report using the template in [analysis-patterns.md](analysis-patterns.md#report-template) and stop. **Do not** keep grepping for completeness — extra evidence rarely changes the verdict and always costs tokens.

#### Evidence calibration — be honest about what the logs prove

Logs show what happened and when. They rarely show *why* on their own. Misattributing a crash to Cursor when the cause is the user's environment (AV blocking helper processes, GPU driver, real resource exhaustion, a third-party extension) wastes the team's time and erodes trust. Misattributing it to the user when Cursor is genuinely at fault hides bugs the team would fix. The skill's job is to **help investigate**, not to assign blame quickly.

**Classify every finding before quoting it as evidence:**

- **Confirmed cause** — a single piece of evidence directly explains the crash and the timing matches (e.g. a native dump's faulting module + a renderer-gone line at the same second; a non-zero extension-host exit immediately before the window dies).
- **Strong contributor** — evidence shows this made the crash much more likely, but is not by itself proof (e.g. in-renderer MCP fallback + a large tool fleet + OOM in the same window).
- **Coincident factor** — present at crash time but causal link unclear (e.g. a benign warning, an extension load, a recent settings change).
- **Hypothesis** — a possible explanation that is not yet verified. State the **disambiguating test** that would confirm or rule it out (e.g. "could be AV blocking the helper process — to confirm, ask the user to add Cursor.exe to AV exclusions and re-test").

**Generate at least one alternative hypothesis** before settling on a verdict that implicates Cursor. The usual alternatives:

- **User-environment**: AV / EDR / firewall, GPU driver, low free RAM, low free disk, corporate policy, network / OneDrive / WSL workspace path, locale / encoding, full battery-saver mode.
- **Third-party extension or MCP server** (especially when the user installs many).
- **Specific large workspace / large file / large chat** (a memory cliff that any editor would hit).
- **Cursor itself** — default to this **only** when the other categories don't fit *and* the evidence implicates internal Cursor code paths (e.g. listener leak whose top stack frame is inside Cursor's own bundled code).

**Calibrated language only.** Use "evidence shows", "consistent with", "the logs indicate", "would need X to confirm." Avoid "this is a Cursor bug" or "the user did Y" unless the evidence is unambiguous and the alternatives have been ruled out.

If you cannot pick between two or more explanations from the logs alone, say so in the verdict and name the specific test that would disambiguate them (e.g. "reproduce with `--disable-extensions` to rule out a third-party extension"; "boot with `--disable-gpu` to test GPU"; "test in a clean profile with `--user-data-dir <empty>` to rule out profile corruption").

## Anonymization (only if the user asks)

If the user wants to scrub personal or identifying information from the logs before sending them to Cursor, follow the workflow in [anonymize-logs.md](anonymize-logs.md). Key rule: the agent **never reads log content into its context** to find what to redact. It asks the user for the substitution map (username, hostname, workspace path, etc.) and generates a per-OS script that runs on the user's machine. The user re-zips the redacted directory and sends *that*.

## Common root causes & remediation

Each row has a **Try yourself** column (things the user can do **right now**, on their own machine, that often resolve or work around the issue) and a **Forward to Cursor** column (cases that benefit from a Cursor engineer seeing the artifact).

| Pattern | Try yourself | Forward to Cursor |
|---|---|---|
| Renderer OOM during agent loop + many MCP servers + MCP host failing | (1) Fully quit Cursor from tray / dock (not just close window) and reopen — fixes most MessagePort issues. (2) Settings → MCP → temporarily disable servers not needed right now. (3) Retry with Max Mode off. | If the MessagePort error returns on every fresh launch even after a full restart and antivirus check. |
| Renderer OOM with small MCP footprint | (1) Disable heavy extensions one at a time. (2) Close other large workspace windows. (3) Increase Windows pagefile / check available RAM. | If a specific workspace reliably OOMs Cursor even with extensions disabled and plenty of free RAM. |
| GPU process crashed | Launch with `--disable-gpu` once. If stable, update GPU drivers. | If `--disable-gpu` is the only thing that keeps Cursor stable. Include the GPU info: `dxdiag` (Windows) or `system_profiler SPDisplaysDataType` (macOS). |
| Extension host crash only | Reproduce with `--disable-extensions`. If stable, bisect by re-enabling halves of the extension list until you find the culprit. | If the crashing extension is a first-party `anysphere.*` extension. Third-party extensions: report to that extension's author. |
| Single MCP server crashing | Disable that MCP server in Settings → MCP. | If the crashing server is a first-party Cursor MCP, or you need help reading its server log. |
| Whole app dies with no Cursor logs covering the crash | Native OS dump (WER / DiagnosticReports / coredump) is mandatory — see [log-locations.md](log-locations.md). Re-run with logs from a terminal so stderr is visible. | Almost always — these are the hardest crashes to self-diagnose. |
| Hang ("recovered from unresponsive", no `process gone`) | Capture a renderer heap snapshot via DevTools → Memory tab while hanging, plus DevTools → Performance recording. Close the offending tab/composer if you can identify it. | If the hang repeats and you've captured the heap snapshot + Performance recording. |
| Listener leak / unbounded allocation inside Cursor-internal code before OOM | **Nothing self-fixable** when the leak's top stack frame is inside Cursor itself. Generic OOM pressure reducers (quit-and-reopen, lighter MCP / extension load, Max Mode off, single window) are temporary mitigations, not a fix. | **Always.** Include the leak warning's top stack frame from `renderer.log` as evidence so the team can locate the leaking subsystem. |

## Phase 6 — After analysis: tell the user what to do

End the conversation with a **clear split** so the user knows exactly what's on them vs what to hand off to Cursor.

### What the user can fix themselves (do these first)

Give an ordered list, cheapest first, based on the diagnosis. Every item must be **safe and reversible** (see "Reversibility" below) and must use **discovery-friendly phrasing** (see "Discovery & verification" above) instead of hardcoded menu paths. Examples in the right shape:

- "Fully quit Cursor: right-click the Cursor icon in the system tray (Windows) or dock (macOS) and choose Quit, then reopen. No state changes — fully reversible."
- "Open the Command Palette (`Ctrl/Cmd+Shift+P`) and search for `MCP`. Toggle off the servers you don't need right now (a screenshot of what you see is fine if you're unsure which ones). **To restore:** open the same panel and toggle them back on — credentials and config are preserved."
- "Toggle Max Mode off for this one run (the toggle is in the chat composer). **To restore:** toggle it back on next run."
- "Launch Cursor once from a terminal with `--disable-gpu` to see if the crash goes away. **To restore:** launch normally next time — no settings are changed."
- "Test a clean profile non-destructively: launch Cursor once with `--user-data-dir <path to an empty temp folder>`. Your real profile, chats, and extensions are untouched. **To restore:** close that window and launch Cursor normally."

If you are unsure of the exact current menu name, ping `cursor-guide` before suggesting it. If the user can't find what you described, ask them to share what their Command Palette shows for the keyword instead of insisting on a path.

Stop here if the issue is resolved or there's a clear workaround the user is happy with.

#### Reversibility (mandatory — read before recommending any change)

Every recommendation must be undoable in one step without external help.

**Prefer changes that:**

- Are toggles in the Cursor UI (toggle back to revert).
- Are one-off CLI flags on a single launch (don't pass next time).
- **Disable** rather than uninstall (disabled extensions and MCP servers stay in the user's list; the toggle reverses it).
- Use `--user-data-dir <empty temp dir>` to test a clean profile without touching the real one.

**Do not recommend:**

- Hand-editing `settings.json`, `mcp.json`, or other config files when a UI toggle exists. Raw JSON edits silently break on typos and have no built-in undo button.
- Uninstalling an extension when disabling it would do.
- Deleting any file or folder under the user's home or Cursor install (see Safety rules above; only escalate to this after the backup-and-export workflow).
- Editing the registry, the root `Cursor.exe` install, or system services.
- Changing OS-wide settings (file associations, default browser, security policy) as part of debugging Cursor.

**For every recommended change, include the reverse step in the same bullet.** "Open X, toggle Y off. To restore: open X, toggle Y on." If a recommendation has *no* clean reverse (uninstall-reinstall, factory-reset profile, clear local storage), route it to "When to email Cursor" — the support team can help the user avoid an unnecessary destructive step.

#### Recommendation hygiene (mandatory — read before writing the list)

Recommendations to the user fall into two camps. Be honest about which is which.

**User-fixable** — the user has a direct lever (a menu item, a setting toggle, a keyboard shortcut, a CLI flag, an exclusion list in third-party software, a driver update, a restart). These belong in the try-yourself list.

**Cursor-fixable** — the symptom comes from Cursor's own code paths (leaks inside Cursor internals, IPC / MessagePort races, native asserts, agent loop crashes with normal config, GPU process crashes without a driver smoking gun). The user has no lever. These belong in "email Cursor" — not in try-yourself, even reworded.

**Test before writing each item:** "If the user does this, do they pull a lever that exists on their machine, or are they being asked to **avoid normal Cursor behavior**?" If the latter, it is not a workaround — drop it.

**Categories of advice to never recommend as a workaround** (these are all "avoid normal Cursor behavior" in disguise):

- Telling the user to avoid, limit, or be careful about anything Cursor produces during normal operation (rendered diffs, tool calls per prompt, agent loops, multi-window use, opening files in the editor, multi-root workspaces).
- Telling the user to use shorter prompts, simpler tasks, or fewer files as a workaround for a crash. That is symptom-management, not a fix.
- Telling the user to restart Cursor every N minutes, refresh on a schedule, or use other "tend the process" rituals to mask a leak.
- Telling the user to permanently downgrade Cursor capability (stay off Max Mode forever, never use a specific model, never enable any MCP) when the crash points at a Cursor bug — temporary mitigation while waiting for a fix is fine; framing it as the answer is not.

Generic OOM pressure reducers (quit-and-reopen, lighter MCP fleet, Max Mode off, fewer simultaneous windows) **are** legitimate while waiting for a Cursor fix. Frame them as *temporary mitigations*, not as fixes — and always pair them with the email-Cursor path so the underlying bug actually gets reported.

### When to email Cursor

**Hard prerequisite: confirmed on the latest version.** The team only supports the latest published version on each release track. If you have not already run Phase 1c's version check, run it now. Do not send logs from an outdated build — the team will ask the user to upgrade and re-test, and the round-trip wastes everyone's time.

If **any** of these are true and the user is confirmed on the latest version, send the logs zip to **hi@cursor.com**:

- None of the "try yourself" steps fixed it, or the crash returns within a few minutes / launches.
- The crash points at a first-party Cursor component (renderer, MCP host, `anysphere.*` extension, agent loop) — i.e. the user can't realistically fix it themselves.
- The crash only happens with a specific Cursor feature (plan tool, agent loop, Max Mode, specific model) and that feature is core to their workflow.
- There's a native OS dump but no useful Cursor log content — Cursor engineers can symbolicate the dump.
- The issue looks like it would help Cursor improve the product (e.g. MCP host startup race, OOM under realistic configs, a regression after a recent update).

**What to send.** Produce a complete email body for them, pre-filling **every** Phase 1b field they collected. Skipping fields wastes a round-trip:

```
To: hi@cursor.com
Subject: Cursor crash - <one-line summary, e.g. "renderer OOM during plan/agent loop on Windows">

Hi Cursor team,

I'm hitting a reproducible crash in Cursor. Logs are attached.

## Cursor

About dialog (paste the **entire block verbatim** - do not trim to just the version; every field is used for triage).
How to open: Command Palette (`Ctrl/Cmd+Shift+P` → `About`), or `Cursor > About Cursor` on macOS, or `Help > About` on Windows / Linux (press `Alt` first if the menu bar is hidden).
<paste full About block - Version, Commit, Electron, Chromium, Node.js, V8, Layout, Build Type, Release Track, OS>

Latest version on this release track (from cursor.com/download, checked <date>): <version>
Confirmed reproducing on latest version: <yes / no - and why if no>
Release track: <Stable / Nightly / other>
Install type: <system / user / portable / package manager>
Installed from: <cursor.com / winget / Chocolatey / Homebrew / distro / AppImage>
Auto-updated recently: <yes/no, approximate date if known>
Auto-updates enabled: <yes/no>

## Machine

OS full version: <e.g. Windows 11 24H2 OS Build 26200.xxxx>
CPU: <model + cores>
RAM: <total>; free at crash time: <approx>
Free disk on home volume: <approx>

## Security software

Antivirus / EDR: <product + version, or "none beyond Microsoft Defender">
Real-time protection: <on/off>
Cursor.exe / install folder excluded from scanning: <yes/no/unknown>
Corporate-managed device (MDM, Intune, Jamf, AppLocker, WDAC, etc.): <yes/no, which>

## Crash

Approximate crash time: <local time + timezone>
Crash symptom: <window closed / "renderer process gone" / hang / MCP unavailable / etc.>
Reproducibility: <every time / most attempts / intermittent / one-time>
Specific trigger: <which tool, which model, Max Mode on/off, plan mode on/off, agent loop, MCP tool call, large paste, specific workspace, etc.>
Workspace: <rough file count, indexed yes/no, on local SSD / network drive / OneDrive / WSL>

Request ID(s) of the failing request(s): <paste verbatim from chat message overflow menu, or "n/a - non-chat crash">
Composer / chat ID (if available): <e.g. composerId=a46b9655-... from renderer.log, or "n/a">

## MCP and extensions

MCP servers enabled at crash time:
<list, or screenshot of Settings > MCP>

Third-party extensions installed:
<paste `cursor --list-extensions` output or screenshot>

## What I tried

- <e.g. fully quit from tray and reopened: did/didn't help>
- <e.g. disabled MCP servers X, Y, Z: did/didn't help>
- <e.g. retried with Max Mode off: did/didn't help>
- <e.g. launched with --disable-extensions: did/didn't help>
- <e.g. launched with --user-data-dir <empty>: did/didn't help>
- <e.g. tested on a different machine: did/didn't reproduce>

Thanks,
<name>
```

Attach the **zip from Phase 4** (session logs + Crashpad + OS dumps + `devtools-console.txt` if captured). If they have a screen recording, attach that too — especially for hangs.

Do **not** suggest filing a GitHub issue, posting on the forum, or DMing on Slack as a first option — `hi@cursor.com` is the routed channel and the only one where the logs zip can be safely attached.

## Output expectations

When you summarise back to the user, keep it tight:

1. **One-line verdict** (e.g. "Renderer out-of-memory during a Max Mode agent loop; MCP host was already broken and 15 MCP servers were loaded in-process").
2. **Evidence** — quote the 2–4 log lines that prove it, each with file path and timestamp.
3. **Try yourself** — ordered, cheapest-first steps from the table above.
4. **If that doesn't fix it** — the email-to-hi@cursor.com block above, pre-filled.

Do not paste raw log dumps. Do not speculate beyond what the logs show — if it could be the GPU **or** OOM, say both and how to disambiguate.

## Additional resources

- [log-locations.md](log-locations.md) — exact paths per OS, including OS-level crash dumps and how to enable Windows user-mode dumps.
- [analysis-patterns.md](analysis-patterns.md) — full pattern catalog with example log lines and what each one means.
