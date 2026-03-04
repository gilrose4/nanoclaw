# Intent: setup/service.ts modifications

## What changed
Added Windows service setup using Git Bash launcher and Windows Registry autostart.

## Key sections
- **Import** (line 14): Added `commandExists` from `./platform.js`
- **run() switch** (line 58): Added `else if (platform === 'windows')` branch calling `setupWindows(projectRoot, nodePath)`
- **findGitBash()** (new function): Locates Git Bash by checking `where git`, deriving `bash.exe` path, trying common install locations. Attempts `winget install Git.Git` if not found.
- **setupWindows()** (new function): Generates `start-nanoclaw.sh` (bash script running node with log redirection) and `start-nanoclaw.ps1` (PowerShell launcher delegating to bash). Registers in HKCU Run key for autostart. Launches immediately.

## Invariants
- `setupLaunchd`, `setupLinux`, `setupSystemd`, `setupNohupFallback` all unchanged
- `killOrphanedProcesses`, `checkDockerGroupStale` all unchanged
- The emitStatus pattern is followed consistently
- Build step at the start of `run()` is unchanged

## Must-keep
- All existing service setup functions (launchd, systemd, nohup)
- The build-first pattern in `run()`
- Error handling and logging patterns
- The `emitStatus` calls with consistent field naming
