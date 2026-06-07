---
name: codex-desktop-tools
description: Repair, migrate, import, export, and recover local Codex Desktop session/project state on Windows using the codex-desktop-tools PowerShell scripts. Use when Codex needs to move sessions after a project path change, import exported rollout JSONL files, rebuild missing Codex Desktop project UI state, attach a projectless thread to a project, set or clear Codex proxy environment variables, or create a proxy-launch shortcut.
---

# Codex Desktop Tools

Use this skill to choose and run the `codex-desktop-tools` PowerShell utilities for local Codex Desktop maintenance on Windows. The upstream tools repository is `https://github.com/GImDX/codex-desktop-tools`; on this machine the local working copy is commonly `C:\Users\91571\Desktop\codex\codex-session-tools`.

## Ground Rules

- Prefer the repository scripts over ad hoc edits to `$HOME\.codex`.
- Read Chinese docs or JSONL-adjacent text with explicit UTF-8 in PowerShell.
- Start with `-DryRun` whenever a script supports it.
- Before write operations that touch `$HOME\.codex`, tell the user what will change and keep generated backups until Codex Desktop has been reopened and checked.
- Before running `move-codex-thread-to-project.ps1` without `-DryRun`, require the user to fully exit Codex Desktop, including tray and background processes.
- Use `C:\Users\91571\AppData\Local\Programs\Python\Python313\python.exe` by default when a script needs Python. For `move-codex-thread-to-project.ps1`, pass `-PythonExe` or set `CODEX_SESSION_TOOLS_PYTHON` if needed.

## Locate The Tools

Use the local repository if it exists:

```powershell
Set-Location -LiteralPath "C:\Users\91571\Desktop\codex\codex-session-tools"
```

If the local copy is missing, use the checked-out `GImDX/codex-desktop-tools` repository path provided by the user. Do not assume a generated shortcut or backup directory is tracked by Git.

## Pick The Workflow

- Project path changed and old sessions still point to the old `cwd`: use export then import.
- Session files exist but Codex Desktop UI does not show projects or thread roots: use restore projects and sessions.
- One known thread ID is projectless or attached to the wrong project: use move thread to project.
- Codex Desktop needs proxy environment variables or those variables should be cleared: use set proxy env.
- Codex Desktop needs Chromium `--proxy-server` at launch: use create proxy shortcut.

## Export Then Import Sessions

Preview matching sessions:

```powershell
.\export-codex-sessions.ps1 `
  -OldCwd "C:\path\to\old-project" `
  -NewCwd "C:\path\to\new-project" `
  -OutputDir "C:\path\to\new-project\.codex-conversations" `
  -DryRun
```

Export after checking the preview:

```powershell
.\export-codex-sessions.ps1 `
  -OldCwd "C:\path\to\old-project" `
  -NewCwd "C:\path\to\new-project" `
  -OutputDir "C:\path\to\new-project\.codex-conversations" `
  -Force
```

Preview import and project registration:

```powershell
.\import-codex-sessions.ps1 `
  -ExportDir "C:\path\to\new-project\.codex-conversations" `
  -ExpectedCwd "C:\path\to\new-project" `
  -RegisterProjectRoots `
  -DryRun
```

Import after checking the preview:

```powershell
.\import-codex-sessions.ps1 `
  -ExportDir "C:\path\to\new-project\.codex-conversations" `
  -ExpectedCwd "C:\path\to\new-project" `
  -RegisterProjectRoots `
  -Force
```

Use `-OnlyIds` or `-OnlyFiles` for narrow imports. Use `-RestoreBackup` only when the user explicitly wants to roll back an import backup.

## Restore Missing Project UI State

Use explicit project roots when the user knows them:

```powershell
.\restore-codex-projects-and-sessions.ps1 `
  -ProjectRoots "C:\path\to\project-a", "C:\path\to\project-b" `
  -DryRun
```

Use session metadata discovery when the user wants to recover all discoverable roots:

```powershell
.\restore-codex-projects-and-sessions.ps1 -ProjectRootsFromSessions -DryRun
```

After checking the dry run, run the same command without `-DryRun`. This updates Codex global project state and creates a backup under `$HOME\.codex`.

## Move One Thread To A Project

Use a dry run first:

```powershell
.\move-codex-thread-to-project.ps1 `
  -ThreadId "00000000-0000-0000-0000-000000000000" `
  -TargetProjectRoot "C:\path\to\project" `
  -PythonExe "C:\Users\91571\AppData\Local\Programs\Python\Python313\python.exe" `
  -DryRun
```

For the real run, stop and tell the user to fully exit Codex Desktop before continuing. Then run the same command without `-DryRun`.

This workflow can update `state_5.sqlite`, the rollout JSONL `session_meta`, and `.codex-global-state.json`; do not hand-edit those files unless the script cannot be used.

## Proxy Setup

Set user-level proxy variables:

```powershell
.\set-codex-proxy-env.ps1 `
  -ProxyUrl "http://127.0.0.1:<port>" `
  -DryRun
```

Clear the same variables:

```powershell
.\set-codex-proxy-env.ps1 -Clear -DryRun
```

After the dry run, repeat without `-DryRun` if the user approves the change. Restart Codex Desktop after setting or clearing proxy variables.

## Proxy Shortcut

Preview shortcut creation:

```powershell
.\create-codex-proxy-shortcut.ps1 `
  -ProxyServer "http://127.0.0.1:<port>" `
  -DryRun
```

Create or update the shortcut after checking the preview:

```powershell
.\create-codex-proxy-shortcut.ps1 `
  -ProxyServer "http://127.0.0.1:<port>"
```

Pass `-CodexPath` only when automatic Codex Desktop executable discovery fails.

## Validation

For repository maintenance or script edits, validate touched scripts through representative dry runs before finishing. Useful coverage includes:

- `set-codex-proxy-env.ps1 -DryRun`
- `create-codex-proxy-shortcut.ps1 -DryRun`
- `export-codex-sessions.ps1 -DryRun`
- `import-codex-sessions.ps1 -DryRun` with a minimal export package
- `restore-codex-projects-and-sessions.ps1 -DryRun`
- `move-codex-thread-to-project.ps1 -DryRun` against a real local thread ID when available
