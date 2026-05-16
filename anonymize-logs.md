# Anonymize logs before sending to Cursor

Only run this workflow when the user explicitly asks to scrub identifying information from the log zip before sending it. The goal is **token-efficient redaction without the agent reading the logs**.

## Core rule

The agent must **never** read the original log content into its own context to find what to redact. It asks the user for the substitution map, generates a script that runs locally, and reads back only the script's diff summary if needed. The user re-zips the redacted directory and sends that.

This keeps the user's data on their machine and keeps the run cheap.

## Step 1 — Ask the user what to scrub

Default categories to offer (pick which to enable):

| Category | Why it leaks | Replacement |
|---|---|---|
| Username | Appears in `C:\Users\<name>`, `/Users/<name>`, `/home/<name>` paths everywhere | `<USER>` |
| Machine / hostname | Network errors, telemetry, some agent logs | `<HOST>` |
| Workspace folder names | May reveal company or product name | `<WORKSPACE>` (or a short pseudonym they choose) |
| Email addresses | Anywhere they appear | `<EMAIL>` |
| Repo URLs | `github.com/<org>/<repo>`, `gitlab…`, internal git hosts | `<REPO_URL>` |
| Internal MCP server names | If they reveal the company (e.g. `mcp-acme-internal`) | `<MCP_X>`, `<MCP_Y>` |
| Internal hostnames in network errors | Corporate proxies, internal services | `<INTERNAL_HOST>` |

Be honest with the user: this is **best-effort scrubbing of the common identifiers above**, not comprehensive PII removal. Logs can contain arbitrary text and they should still spot-check before sending.

## Step 2 — Get the substitution values from the user

Ask the user to paste exactly the values to substitute for each category they enabled. Examples:

```
username:        benpr
hostname:        DESKTOP-AB12CD3
workspace path:  C:\dev\acme-frontend
email:           ben@acme.example
repo URL:        https://github.com/acme-corp/frontend
```

Don't try to derive these from the logs. The user knows their own identifiers; pulling them from the logs both costs tokens and risks missing aliases (e.g. they might have multiple workspaces).

## Step 3 — Work on a copy

The script always operates on a **copy** of the unzipped logs directory so the user keeps the original (un-redacted) version locally. They may need it later for their own debugging.

## Step 4 — Generate and run the per-OS script

Fill in the substitution values the user provided. Run the script on the user's machine; the agent never needs to read the resulting log content.

### macOS / Linux (bash, GNU or BSD sed)

```bash
SRC=/path/to/session/20260516T112350
DEST="${SRC}.redacted"
cp -R "$SRC" "$DEST"

# Choose sed in-place flag for portability across GNU and BSD sed:
case "$(uname)" in
  Darwin) SED_INPLACE=(-i '') ;;
  *)      SED_INPLACE=(-i)    ;;
esac

LC_ALL=C find "$DEST" -type f \( -name '*.log' -o -name '*.txt' -o -name '*.json' \) -print0 \
  | xargs -0 sed "${SED_INPLACE[@]}" \
      -e 's|<actual-username>|<USER>|g' \
      -e 's|<actual-hostname>|<HOST>|g' \
      -e 's|<actual-workspace-path>|<WORKSPACE>|g' \
      -e 's|<actual-email>|<EMAIL>|g' \
      -e 's|<actual-repo-url>|<REPO_URL>|g' \
      -e 's|[A-Za-z0-9._%+-]\+@[A-Za-z0-9.-]\+\.[A-Za-z]\{2,\}|<EMAIL>|g'

echo "Redacted copy at: $DEST"
```

Add `-e` lines for each extra category the user enabled. Keep the literal values out of any shell history if the user is concerned — they can `export VAR=…; sed -e "s|$VAR|<TOKEN>|g"` instead.

### Windows (PowerShell)

```powershell
$Src  = 'C:\path\to\session\20260516T112350'
$Dest = "$Src.redacted"
Copy-Item -Recurse -Force $Src $Dest

$subs = [ordered]@{
  '<actual-username>'       = '<USER>'
  '<actual-hostname>'       = '<HOST>'
  '<actual-workspace-path>' = '<WORKSPACE>'
  '<actual-email>'          = '<EMAIL>'
  '<actual-repo-url>'       = '<REPO_URL>'
}

Get-ChildItem -Recurse $Dest -Include *.log,*.txt,*.json -File | ForEach-Object {
  $text = Get-Content $_.FullName -Raw
  foreach ($k in $subs.Keys) { $text = $text.Replace($k, $subs[$k]) }
  # Generic email regex
  $text = [regex]::Replace($text, '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}', '<EMAIL>')
  Set-Content -Path $_.FullName -Value $text -NoNewline
}

Write-Host "Redacted copy at: $Dest"
```

PowerShell `Replace` is a literal string replace (faster, no regex surprises) for the named values; `[regex]::Replace` is used only for the generic email pattern. Add entries to `$subs` for each extra category.

### Notes on what the scripts cover

- Both scripts only touch `.log`, `.txt`, and `.json` files. Binary artifacts like Crashpad `.dmp` files contain raw memory and **cannot be safely redacted with sed/regex**. Either include them as-is (a Cursor engineer needs the raw bytes to symbolicate) or omit them from the zip if the user is uncomfortable. Tell the user this choice exists.
- Native OS dump folders (Windows WER, macOS `~/Library/Logs/DiagnosticReports`) are outside the session folder. If the user wants those scrubbed too, copy them into a parallel directory and run the same script there — but note the dumps themselves still cannot be regex-scrubbed.
- Email regex is intentionally simple. It will miss obfuscated forms (e.g. `name [at] domain`) and may over-match a string that *looks* like an email. The user should still spot-check.

## Step 5 — Verify and re-zip

Ask the user to:

1. Open the `.redacted` directory and spot-check 1-2 files (e.g. `main.log`, the relevant `renderer.log`) for any leftover identifier they care about. They can `rg -i 'their-name|their-host|their-company'` to sanity-check.
2. Zip the **redacted** directory.
3. Attach that zip to the email to `hi@cursor.com` — not the original.
4. Mention in the email body: "Logs have been anonymized: `<list categories>` replaced with placeholders." This lets the Cursor engineer know what's been stripped so they don't waste time looking for IDs that won't be there.

## What the agent does and doesn't read

- **Reads:** the user's substitution-map values (one short message), the script output (a single line confirming the destination path), and optionally a count of replacements per category if the script reports it.
- **Does not read:** the original log content, the redacted log content, individual lines or matches from the scripts.

If the user asks "is field X still in the logs?", do not open the log to find out. Suggest they run `rg -c '<value>' <DEST>` themselves and paste only the count.
