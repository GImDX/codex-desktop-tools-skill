---
name: codex-desktop-tools
description: Repair, migrate, import, export, and recover local Codex Desktop session/project state on Windows using the codex-desktop-tools PowerShell scripts and thread tools. Use when Codex needs to move sessions after a project path change, import exported rollout JSONL files, rebuild missing Codex Desktop project UI state, attach a projectless thread to a project, set or clear Codex proxy variables in ~/.codex/.env, create a proxy-launch shortcut, or repair Codex Desktop sessions shown as "New chat" or "新对话".
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
- For session-title repair, use Codex app thread tools such as `list_threads`, `read_thread`, and `set_thread_title`; do not edit SQLite, rollout JSONL, or `session_index.jsonl` directly.

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
- Codex Desktop needs proxy variables in `~\.codex\.env` or those variables should be cleared: use set proxy env.
- Codex Desktop needs Chromium `--proxy-server` at launch: use create proxy shortcut.
- Older sessions display as `New chat` or `新对话` after restart but reveal meaningful titles/content when opened: use session title repair.

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

Useful export controls:

- `-IncludeArchived` also scans `$HOME\.codex\archived_sessions`.
- `-IncludeTextMatch "text"` exports sessions whose file content contains the text even when `cwd` is not `OldCwd`.
- `-NoPathRewrite` copies matched sessions without replacing `OldCwd` with `NewCwd`.

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

Use `-OnlyIds` or `-OnlyFiles` for narrow imports. When `-RegisterProjectRoots` is used, missing project directories throw unless `-SkipMissingProjectRoots` is also passed. Use `-RestoreBackup` only when the user explicitly wants to roll back an import backup.

## Restore Missing Project UI State

Use explicit project roots when the user knows them:

```powershell
.\restore-codex-projects-and-sessions.ps1 `
  -ProjectRoots "C:\path\to\project-a", "C:\path\to\project-b" `
  -DryRun
```

Use session metadata discovery when the user wants to recover all discoverable roots. Only `cwd` values that still exist as directories are registered:

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

Set proxy variables in `~\.codex\.env` for Codex Desktop:

```powershell
.\set-codex-proxy-env.ps1 `
  -ProxyUrl "http://127.0.0.1:<port>" `
  -DryRun
```

Clear the same variables:

```powershell
.\set-codex-proxy-env.ps1 -Clear -DryRun
```

The script creates `~\.codex` when needed, updates existing proxy keys, appends missing proxy keys, and leaves unrelated `.env` entries unchanged. Its output marks each key as `Add`, `Update`, `Keep`, `Remove`, or `Absent`.

After the dry run, repeat without `-DryRun` if the user approves the change. Restart Codex Desktop after setting or clearing proxy variables so it reloads `~\.codex\.env`.

## Proxy Shortcut

Use `create-codex-proxy-shortcut.ps1` when environment variables are not enough for the desktop UI bootstrap and Codex needs Chromium's `--proxy-server` flag at process startup.

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

Create the shortcut in an existing specific directory:

```powershell
.\create-codex-proxy-shortcut.ps1 `
  -ProxyServer "http://127.0.0.1:<port>" `
  -OutputDirectory "C:\path\to\shortcuts"
```

Use `-ShortcutName` to change the generated `.lnk` filename. Use `-AppId` only when the Windows AppsFolder app ID must be overridden.

The shortcut targets Windows PowerShell and runs `Start-Process -FilePath shell:AppsFolder\<AppId> -ArgumentList '--proxy-server=...'`. It does not directly target a versioned `WindowsApps\...\app\Codex.exe` path.

## Session Title Repair

Use this workflow when Codex Desktop shows older sessions as `New chat` or `新对话` after restart even though opening the thread reveals the real conversation.

Use Codex app thread tools, not local database or JSONL edits:

1. Discover affected threads with `list_threads`, or use thread IDs provided by the user.
2. Read every candidate with `read_thread`.
3. Prefer an existing non-placeholder title when available; otherwise derive a concise title from the earliest concrete user request or first meaningful task statement.
4. Rename through `set_thread_title`.
5. Re-read each renamed thread and report thread ID, old visible title, and new title.

Do not treat `thread.preview` or the first user message as proof that the visible title is repaired after restart. Ask the user to fully restart Codex Desktop for final verification because the bug is restart-time sidebar state.

## Validation

For repository maintenance or script edits, validate touched scripts through representative dry runs before finishing. Useful coverage includes:

- `set-codex-proxy-env.ps1 -DryRun`
- `create-codex-proxy-shortcut.ps1 -DryRun`
- `export-codex-sessions.ps1 -DryRun`
- `import-codex-sessions.ps1 -DryRun` with a minimal export package
- `restore-codex-projects-and-sessions.ps1 -DryRun`
- `move-codex-thread-to-project.ps1 -DryRun` against a real local thread ID when available
