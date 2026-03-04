---
name: enable-windows
description: Enable Windows platform support for NanoClaw. Adds Windows detection, Git Bash-based service launcher, HKCU Registry autostart, and log-based service verification.
---

# Enable Windows Support

This skill adds Windows platform support to NanoClaw's setup, service management, and verification using the skills engine for deterministic code changes.

## Phase 1: Pre-flight

### Check if already applied

Read `.nanoclaw/state.yaml`. If `enable-windows` is in `applied_skills`, skip to Phase 5 (Verify).

### Check Git for Windows

Git for Windows is required for the bash-based service launcher.

```bash
where git
```

If Git is not found:

1. Try installing via winget:
   ```bash
   winget install Git.Git --accept-package-agreements --accept-source-agreements
   ```
2. If winget is not available, tell the user:
   > Git for Windows is required. Install from https://git-scm.com/download/win
   > After installing, restart your terminal and re-run this skill.

Wait for Git to be available before proceeding.

## Phase 2: Apply Code Changes

Run the skills engine to apply this skill's code package.

### Initialize skills system (if needed)

If `.nanoclaw/` directory doesn't exist yet:

```bash
npx tsx scripts/apply-skill.ts --init
```

### Apply the skill

```bash
npx tsx scripts/apply-skill.ts .claude/skills/enable-windows
```

This deterministically:
- Three-way merges Windows platform detection into `setup/platform.ts` (`'windows'` type, `win32` detection, `where` command usage)
- Three-way merges Windows service setup into `setup/service.ts` (`findGitBash()`, `setupWindows()` with bash launcher, HKCU registry autostart)
- Three-way merges Windows verification into `setup/verify.ts` (log file mtime check)
- Three-way merges Windows shell option into `setup/whatsapp-auth.ts` (`shell: true` on win32)
- Records the application in `.nanoclaw/state.yaml`

If the apply reports merge conflicts, read the intent files:
- `modify/setup/platform.ts.intent.md`
- `modify/setup/service.ts.intent.md`
- `modify/setup/verify.ts.intent.md`
- `modify/setup/whatsapp-auth.ts.intent.md`

### Validate code changes

```bash
npm run build
```

Build must be clean before proceeding.

## Phase 3: Setup

No additional configuration needed. Windows support is automatically detected by `getPlatform()` when running on Windows.

Run the normal setup flow to configure the service:

```bash
npx tsx setup/service.ts
```

This will:
1. Find Git Bash (or attempt to install Git via winget)
2. Generate `start-nanoclaw.sh` (bash script running node with log redirection)
3. Generate `start-nanoclaw.ps1` (PowerShell launcher delegating to bash)
4. Register `NanoClaw` in the HKCU Registry Run key for autostart at logon
5. Launch the service immediately

## Phase 4: Configuration

No manual configuration needed. The Windows platform is auto-detected.

## Phase 5: Verify

### Check the build

```bash
npm run build
```

### Check service status (on Windows)

```bash
npx tsx setup/verify.ts
```

The Windows verification checks `logs/nanoclaw.log` — if modified within the last 5 minutes, the service is considered running.

### Manual checks

Check the Registry Run key:
```powershell
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v NanoClaw
```

Check log output:
```powershell
Get-Content -Tail 20 logs\nanoclaw.log
```

Start manually if needed:
```powershell
powershell -File start-nanoclaw.ps1
```

## Troubleshooting

### Service not starting

1. Check Git Bash exists: `where git` — if missing, install Git for Windows
2. Check the generated scripts exist: `start-nanoclaw.ps1` and `start-nanoclaw.sh`
3. Run the PS1 manually to see errors: `powershell -File start-nanoclaw.ps1`
4. Check error log: `Get-Content logs\nanoclaw.error.log`

### Autostart not working

1. Verify registry entry: `reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v NanoClaw`
2. If missing, re-run setup: `npx tsx setup/service.ts`

### "npx not found" during WhatsApp auth

On Windows, `npx` is a `.cmd` file that requires shell execution. The skill adds `shell: true` to the spawn options. If you still see issues, ensure Node.js is in your PATH.

## Removal

To remove Windows support:

1. Revert `setup/platform.ts` — remove `'windows'` from Platform type and `win32` detection
2. Revert `setup/service.ts` — remove `findGitBash()`, `setupWindows()`, and the Windows branch in `run()`
3. Revert `setup/verify.ts` — remove the Windows log file mtime check
4. Revert `setup/whatsapp-auth.ts` — remove `shell: process.platform === 'win32'`
5. Delete generated scripts: `start-nanoclaw.ps1`, `start-nanoclaw.sh`
6. Remove registry entry: `reg delete "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v NanoClaw /f`
