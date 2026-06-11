---
name: codex-desktop-tools
description: Repair, migrate, import, export, and recover local Codex Desktop session/project state on Windows using the codex-desktop-tools PowerShell scripts. Use when Codex needs to move sessions after a project path change, import exported rollout JSONL files, rebuild missing Codex Desktop project UI state, attach a projectless thread to a project, set or clear Codex proxy environment variables, create a proxy-launch shortcut, or launch MSIX Codex Desktop through AppActivation with Chromium proxy arguments.
---

# Codex Desktop Tools

Use this skill to choose and run the `codex-desktop-tools` PowerShell utilities for local Codex Desktop maintenance on Windows. The upstream tools repository is `https://github.com/GImDX/codex-desktop-tools`; on this machine the local working copy is commonly `C:\Users\91571\Desktop\codex\codex-session-tools`.

## Ground Rules

- Prefer the repository scripts over ad hoc edits to `$HOME\.codex`.
- Read Chinese docs or JSONL-adjacent text with explicit UTF-8 in PowerShell.
- Start with `-DryRun` whenever a script supports it.
- Before write operations that touch `$HOME\.codex`, tell the user what will change and keep generated backups until Codex Desktop has been reopened and checked.
- Before running `move-codex-thread-to-project.ps1` without `-DryRun`, require the user to fully exit Codex Desktop, including tray and background processes.
- Treat `start-codex-proxy-appactivation.ps1` as a live launcher, not a validation helper. By default it stops existing `Codex` processes before launching unless `-KeepExisting` is passed.
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
- A generated proxy shortcut must work with a newer MSIX/WindowsApps Codex install: use AppActivation mode or the auto fallback.

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

Use `create-codex-proxy-shortcut.ps1` when environment variables are not enough for the desktop UI bootstrap and Codex needs Chromium's `--proxy-server` flag at process startup.

Preview shortcut creation. In `Auto` mode the script resolves the installed Codex Desktop executable; if the resolved executable is under `WindowsApps`, it falls back to an AppActivation launcher:

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

Choose a launch mode explicitly only when the default `Auto` decision is wrong:

```powershell
.\create-codex-proxy-shortcut.ps1 `
  -ProxyServer "http://127.0.0.1:<port>" `
  -LaunchMode AppActivation `
  -DryRun

.\create-codex-proxy-shortcut.ps1 `
  -ProxyServer "http://127.0.0.1:<port>" `
  -LaunchMode Direct `
  -CodexPath "C:\path\to\app\Codex.exe" `
  -DryRun
```

Use `-LaunchMode AppActivation` for newer MSIX installs where directly executing the `WindowsApps` executable is brittle. The shortcut targets Windows PowerShell and calls `start-codex-proxy-appactivation.ps1` with `-AppId` and `-ProxyServer`.

Use `-LaunchMode Direct` and `-CodexPath` only when direct executable launch is known to work and the path ends in `\app\Codex.exe`.

If automatic AppActivation setup cannot find the launcher script beside `create-codex-proxy-shortcut.ps1`, pass `-AppActivationLauncherPath`.

## AppActivation Proxy Launch

Use `start-codex-proxy-appactivation.ps1` only when the user actually wants to launch Codex Desktop through the Windows AppUserModelID with proxy arguments. It has no `-DryRun`.

```powershell
.\start-codex-proxy-appactivation.ps1 `
  -AppId "OpenAI.Codex_2p2nqsd0c76g0!App" `
  -ProxyServer "http://127.0.0.1:<port>"
```

By default it stops existing `Codex` processes first, then launches with `--proxy-server=<proxy>`. Pass `-KeepExisting` only when the user explicitly wants to avoid stopping the current Codex process.

## Validation

For repository maintenance or script edits, validate touched scripts through representative dry runs before finishing. Useful coverage includes:

- `set-codex-proxy-env.ps1 -DryRun`
- `create-codex-proxy-shortcut.ps1 -DryRun`
- `create-codex-proxy-shortcut.ps1 -LaunchMode AppActivation -DryRun`
- `export-codex-sessions.ps1 -DryRun`
- `import-codex-sessions.ps1 -DryRun` with a minimal export package
- `restore-codex-projects-and-sessions.ps1 -DryRun`
- `move-codex-thread-to-project.ps1 -DryRun` against a real local thread ID when available

Do not run `start-codex-proxy-appactivation.ps1` as a routine validation step; it can stop or launch Codex Desktop.
