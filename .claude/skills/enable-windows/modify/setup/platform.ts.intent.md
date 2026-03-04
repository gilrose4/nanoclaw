# Intent: setup/platform.ts modifications

## What changed
Added Windows to platform detection and adapted command-line utilities for Windows.

## Key sections
- **Platform type union** (line 8): Added `'windows'` to the `Platform` type
- **getPlatform()** (line 15): Detects `os.platform() === 'win32'` and returns `'windows'`
- **getNodePath()** (line 105): Uses `where node` on Windows instead of `command -v node`. Splits on newlines and takes the first result since Windows `where` may return multiple paths.
- **commandExists()** (line 117): Uses `where <name>` on Windows instead of `command -v <name>`

## Invariants
- All macOS and Linux behavior unchanged
- `getServiceManager()` returns `'none'` for Windows (no launchd or systemd)
- `openBrowser()` has no Windows case (not needed for current setup flow)
- `isWSL()`, `isRoot()`, `isHeadless()`, `hasSystemd()` all unchanged

## Must-keep
- All existing platform detection functions
- WSL detection logic
- Headless environment detection
- systemd detection via `/proc/1/comm`
- `getNodeVersion()` and `getNodeMajorVersion()` utilities
