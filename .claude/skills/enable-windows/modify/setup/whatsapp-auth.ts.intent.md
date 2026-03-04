# Intent: setup/whatsapp-auth.ts modifications

## What changed
Added `shell: true` on Windows for child process spawning.

## Key sections
- **spawn options** (line 190): Added `shell: process.platform === 'win32'`. On Windows, `npx` is a `.cmd` file requiring shell execution. On other platforms, direct execution is used for security and performance.

## Invariants
- All auth flow logic unchanged (QR browser, pairing code, terminal)
- HTML templates unchanged
- Polling intervals and timeouts unchanged
- Cleanup handler unchanged

## Must-keep
- The entire auth flow for all methods
- The statusFile/qrFile polling mechanism
- The cleanup handler on process exit
- All emitAuthStatus calls
