# Windows disposable Herdr session commands for ticket 014

## Scope

Read-only research against the installed binary. No session, pane, directory, or process was created while preparing this procedure.

Installed authority: `C:\Users\uosmi\AppData\Local\Programs\Herdr\bin\herdr.exe`, version `0.8.0-preview.2026-08-04-d78e3d3b5126`.

## Canonical PowerShell procedure

Run these from the existing Herdr-managed parent pane. Keep `$sessionName`, `$reproDir`, and `$outerPaneId` for cleanup. Parse every returned identifier; do not construct pane IDs.

```powershell
if ($env:HERDR_ENV -ne '1') { throw 'This procedure must run inside Herdr.' }

$stamp = Get-Date -Format 'yyyyMMdd-HHmmss-fff'
$sessionName = "repro-kiro-v3-$stamp"
$reproDir = Join-Path ([IO.Path]::GetTempPath()) $sessionName
New-Item -ItemType Directory -Path $reproDir | Out-Null

$split = herdr pane split --current --direction right --cwd $reproDir --no-focus | ConvertFrom-Json
$outerPaneId = $split.result.pane.pane_id
if (-not $outerPaneId) { throw 'pane split did not return result.pane.pane_id' }
```

Launch the named TUI in that new outer pane. `pane run` sends the supplied PowerShell command plus Enter to the pane's existing shell. Clearing inherited selection, socket, and caller variables prevents the nested runtime from addressing the parent session.

```powershell
$launch = @'
Remove-Item Env:HERDR_SOCKET_PATH,Env:HERDR_CLIENT_SOCKET_PATH,Env:HERDR_SESSION,Env:HERDR_WORKSPACE_ID,Env:HERDR_TAB_ID,Env:HERDR_PANE_ID -ErrorAction SilentlyContinue; herdr --session SESSION_NAME
'@ -replace 'SESSION_NAME', $sessionName

herdr pane run $outerPaneId $launch | ConvertFrom-Json
```

For every parent-side command targeting the disposable session, use this wrapper. It clears inherited socket/caller selection for the call and restores the parent environment exactly afterward.

```powershell
function Invoke-ReproHerdr {
    param([Parameter(ValueFromRemainingArguments)] [string[]] $HerdrArgs)

    $names = 'HERDR_SOCKET_PATH', 'HERDR_CLIENT_SOCKET_PATH', 'HERDR_SESSION', 'HERDR_WORKSPACE_ID', 'HERDR_TAB_ID', 'HERDR_PANE_ID'
    $saved = @{}
    foreach ($name in $names) {
        $saved[$name] = [Environment]::GetEnvironmentVariable($name, 'Process')
    }
    try {
        Remove-Item Env:HERDR_SOCKET_PATH,Env:HERDR_CLIENT_SOCKET_PATH,Env:HERDR_WORKSPACE_ID,Env:HERDR_TAB_ID,Env:HERDR_PANE_ID -ErrorAction SilentlyContinue
        $env:HERDR_SESSION = $sessionName
        & herdr @HerdrArgs
    } finally {
        foreach ($name in $names) {
            [Environment]::SetEnvironmentVariable($name, $saved[$name], 'Process')
        }
    }
}
```

Wait for readiness with bounded polling of `pane list`. Do not use a fixed sleep as proof of readiness.

```powershell
$deadline = (Get-Date).AddSeconds(30)
do {
    $raw = Invoke-ReproHerdr pane list 2>$null
    if ($LASTEXITCODE -eq 0) { $panes = $raw | ConvertFrom-Json; break }
    Start-Sleep -Milliseconds 250
} while ((Get-Date) -lt $deadline)
if ($LASTEXITCODE -ne 0 -or -not $panes) { throw "Named session $sessionName did not become ready." }

$rootPaneId = $panes.result.panes[0].pane_id
if (-not $rootPaneId) { throw 'pane list did not return a root pane ID' }
```

Before using the root pane, inspect its current state:

```powershell
Invoke-ReproHerdr pane get $rootPaneId | ConvertFrom-Json
Invoke-ReproHerdr pane process-info --pane $rootPaneId | ConvertFrom-Json
```

Drive ordinary commands only through the disposable pane:

```powershell
Invoke-ReproHerdr pane run $rootPaneId '<command>' | ConvertFrom-Json
Invoke-ReproHerdr pane wait-output $rootPaneId --match '<literal>' --timeout 30000 | ConvertFrom-Json
Invoke-ReproHerdr pane read $rootPaneId --source recent-unwrapped --lines 120 | ConvertFrom-Json
Invoke-ReproHerdr pane send-text $rootPaneId '<literal text>' | ConvertFrom-Json
Invoke-ReproHerdr pane send-keys $rootPaneId enter | ConvertFrom-Json
```

Use `pane run` for a command plus Enter. `send-text` is literal input without Enter. Use `send-keys` only for supported logical keys. For ticket 014, inspect `herdr agent --help` and its nested command help before using agent commands after Kiro is recognized.

## Cleanup

First stop test applications cleanly inside `$rootPaneId` if practical. Then stop and delete exactly the generated session name. Session management commands take the name explicitly and therefore do not need the disposable-session environment wrapper.

```powershell
herdr session stop $sessionName --json | ConvertFrom-Json
herdr session delete $sessionName --json | ConvertFrom-Json

$remaining = (herdr session list --json | ConvertFrom-Json).sessions
if ($remaining.name -contains $sessionName) { throw "Disposable session $sessionName still exists." }

herdr pane wait-output $outerPaneId --regex 'PS [^>]*>' --timeout 10000 | ConvertFrom-Json
herdr pane close $outerPaneId | ConvertFrom-Json

Remove-Item -LiteralPath $reproDir -Recurse -Force
```

Only remove `$reproDir` after resolving it to the exact directory created above. If ticket evidence must be retained, copy the intended evidence into the repository research artifact first. Never stop/delete another session or close another pane because it appears stale.

## Sources

- [L1:verified] Installed `herdr --version`, `herdr --help`, `herdr session --help`, and nested `session list|stop|delete --help` output.
- [L1:verified] Installed `herdr pane --help` and nested `split|list|run|wait-output|read|send-text|send-keys|get|process-info|close --help` output.
- [L1:verified] Installed `herdr --skill` output: creation responses expose `.result.pane.pane_id`; control commands return JSON; explicit IDs/`--current` are required; CLI server errors use exit 1 and syntax errors exit 2.
- [L2:verified] `.agents/skills/herdr-throwaway-repro/SKILL.md`: named-session isolation, environment clearing, readiness, explicit addressing, evidence, and cleanup requirements.
- [L1:verified] `src/cli.rs`: `session list --json` returns `{ "sessions": [...] }`; stop and delete are distinct operations and accept explicit session names.

## Verification caveat

The command syntax and output field expectations are supported by installed help, installed skill text, and source. Per the research constraint, this exact Windows sequence has not yet been executed; ticket 014's run should record the actual JSON and adjust only if the installed observable output differs.
