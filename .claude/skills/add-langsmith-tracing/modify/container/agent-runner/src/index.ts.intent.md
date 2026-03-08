# Intent: Wrap Claude Agent SDK with LangSmith tracing

Three changes to the agent-runner entry point:

1. **Import changes**: Replace direct `query` import with namespace import of the SDK,
   plus imports of `wrapClaudeAgentSDK` from `langsmith/experimental/anthropic` and
   `RunTree` from `langsmith`. Declare `query` as mutable (`let`) and add a
   `tracingEnabled` flag.

2. **SDK wrapping in main()**: After the `sdkEnv` loop, inject LangSmith env vars into
   `process.env` (the tracing SDK reads credentials from there at runtime). Then
   conditionally call `wrapClaudeAgentSDK()` to replace the `query` function with a
   traced version. The wrapper name is set to `containerInput.groupFolder` so traces
   are identifiable per group.

3. **Flush before exit**: After the query loop's try/catch, call
   `RunTree.getSharedClient().flush()` to ensure all trace uploads complete before the
   container exits. Without this, the Node.js process terminates before async HTTP
   requests to LangSmith finish, leaving traces stuck in `pending` status. Replace
   `process.exit(1)` in the catch with `exitCode = 1` so the flush always runs.

All changes are opt-in: when `LANGSMITH_TRACING` is not `'true'` or `LANGSMITH_API_KEY`
is missing, the agent-runner behaves identically to before.
