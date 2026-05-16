# Cursor log locations (per OS)

All paths assume the **Cursor** product. If the user is on **Cursor Nightly** or a side-by-side install, replace the `Cursor` folder name with the matching install (e.g. `Cursor Nightly`). The structure inside is identical.

## Cursor session logs (always collect — most recent dated subfolder)

The session logs folder contains `main.log`, `renderer.log`, per-window subfolders (`window1/`, `window2_wb0/`, ...), and per-extension logs under `exthost/`. **Grab the whole dated subfolder**, not individual files.

| OS | Path | How to open |
|---|---|---|
| Windows | `%APPDATA%\Cursor\logs\` | Win+R → paste path → Enter |
| macOS | `~/Library/Application Support/Cursor/logs/` | Finder → Cmd+Shift+G → paste path |
| Linux | `~/.config/Cursor/logs/` | File manager or `xdg-open` |

The folder contains one timestamped subfolder per Cursor session, e.g. `20260516T112350/`. Pick the one matching the crash time.

**Shortcut from inside Cursor (any OS):** Command Palette → `Developer: Open Logs Folder`. Opens the most recent session directly.

## Crashpad dumps (Electron / Chromium renderer crashes)

Written automatically when the renderer or GPU process crashes. Small (`.dmp` files), worth always grabbing if present.

| OS | Path |
|---|---|
| Windows | `%APPDATA%\Cursor\Crashpad\completed\` |
| macOS | `~/Library/Application Support/Cursor/Crashpad/completed/` |
| Linux | `~/.config/Cursor/Crashpad/completed/` |

If `completed/` is empty, also check `Crashpad/pending/` and `Crashpad/reports/`.

## Native OS crash dumps (whole app died — no Cursor logs cover the crash)

Only needed when the entire process tree dies, not when only the renderer crashes (those are in Crashpad above).

### Windows

1. `%LOCALAPPDATA%\CrashDumps\Cursor.exe.*.dmp`
2. `%LOCALAPPDATA%\Microsoft\Windows\WER\ReportArchive\` — folders starting with `AppCrash_Cursor.exe_...` and `AppHang_Cursor.exe_...`
3. `%LOCALAPPDATA%\Microsoft\Windows\WER\ReportQueue\` — same naming
4. Event Viewer → Windows Logs → Application → filter by Source `Application Error`, `Application Hang`, or `.NET Runtime` around the crash time. Export the matching events as `.evtx`.

If WER isn't writing user-mode dumps, run this once in **elevated PowerShell** and reproduce again:

```powershell
$key = 'HKLM:\SOFTWARE\Microsoft\Windows\Windows Error Reporting\LocalDumps\Cursor.exe'
New-Item -Path $key -Force | Out-Null
New-ItemProperty -Path $key -Name DumpFolder -Value "$env:LOCALAPPDATA\CrashDumps" -PropertyType ExpandString -Force | Out-Null
New-ItemProperty -Path $key -Name DumpType -Value 2 -PropertyType DWord -Force | Out-Null
New-ItemProperty -Path $key -Name DumpCount -Value 5 -PropertyType DWord -Force | Out-Null
```

`DumpType=2` = full dump (large but most useful for a native crash). Reproduce, then collect the new `.dmp` from `%LOCALAPPDATA%\CrashDumps\`.

### macOS

1. `~/Library/Logs/DiagnosticReports/Cursor*.ips` (newer macOS) or `Cursor*.crash` (older). One file per crash.
2. `~/Library/Logs/DiagnosticReports/Cursor Helper*.ips` — renderer / GPU helpers.
3. Also check `/Library/Logs/DiagnosticReports/` for any system-wide entries.
4. Console.app → Crash Reports tab → filter "Cursor" — same data, easier to browse.

### Linux

1. `coredumpctl list | grep -i cursor` — if systemd-coredump is enabled.
2. `coredumpctl info <PID>` to inspect, `coredumpctl dump <PID> > cursor.core` to extract.
3. Fallback: check `/var/lib/systemd/coredump/`, `~/.cache/abrt/`, or distro-specific crash dirs.
4. If nothing exists, the user may need to enable coredumps: `ulimit -c unlimited` in the shell they launch Cursor from, plus a writable `kernel.core_pattern`.

## Inside a session logs folder — what each file is

```
20260516T112350/
├── main.log                              ← MAIN PROCESS: crashes, McpProcess host, lifecycle
├── network-shared.log                    ← shared network process
├── ptyhost.log                           ← terminal subprocess host
├── sharedprocess.log                     ← shared utility process
├── telemetry.log
├── window1/, window2_wb0/, ...           ← one folder per window/workbench
│   ├── renderer.log                      ← RENDERER (per window): UI, extension activation, MessagePort errors
│   ├── output_<timestamp>/               ← Output panel channels for that window
│   │   └── *.log
│   └── exthost/                          ← extension host process for that window
│       ├── exthost.log                   ← extension host main log
│       ├── extHostTelemetry.log
│       ├── output_logging_<timestamp>/   ← extension-emitted output channels
│       ├── anysphere.cursor-agent-exec/  ← agent execution + MCP fleet inventory
│       ├── anysphere.cursor-agent-worker/
│       ├── anysphere.cursor-mcp/         ← per-MCP-server logs
│       │   ├── MCP Logs.*.log            ← MCP orchestrator
│       │   ├── MCP <server-name>.*.log   ← one per MCP server
│       │   └── ...
│       ├── anysphere.cursor-retrieval/   ← indexing, grep, git
│       ├── anysphere.remote-ssh/
│       ├── anysphere.remote-wsl/
│       └── vscode.git/
```

Most useful for crash triage: `main.log`, every `renderer.log`, `exthost.log`, all of `anysphere.cursor-mcp/`, and `anysphere.cursor-agent-exec/` for MCP fleet sizing.

## When the app won't launch at all

Cursor logs won't exist for that session. Skip straight to:

1. Native OS dumps (above).
2. Try launching from a terminal so stderr is visible:
   - Windows: open `cmd.exe`, run `"%LOCALAPPDATA%\Programs\cursor\Cursor.exe"` (system install path may differ — check Start menu shortcut target).
   - macOS: `/Applications/Cursor.app/Contents/MacOS/Cursor`
   - Linux: `cursor` from terminal (or the AppImage / `.deb` install path).
3. Capture the terminal output. Native init crashes (missing system libs, sandbox failures, GPU init) appear there.

## Privacy note before zipping

The session logs include workspace paths, extension activity, and request metadata. They do **not** include source code contents or chat prompts/responses by default, but operators forwarding logs to a third party should still scan for usernames, internal hostnames, or repo names in `main.log` and `exthost.log` before sharing.
