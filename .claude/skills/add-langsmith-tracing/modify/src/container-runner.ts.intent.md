# Intent: Add LangSmith secrets to container allowlist

Add `LANGSMITH_API_KEY`, `LANGSMITH_PROJECT`, `LANGSMITH_ENDPOINT`, and `LANGSMITH_TRACING`
to the `readSecrets()` allowlist so they are passed to containers via stdin.

This is an append-only change to the array in `readSecrets()`. Existing secret
entries must be preserved.
