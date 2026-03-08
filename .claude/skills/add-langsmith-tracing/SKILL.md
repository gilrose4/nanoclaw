# Add LangSmith Tracing

Adds LangSmith observability to NanoClaw's containerized agents. Wraps the Claude Agent SDK with `wrapClaudeAgentSDK` from `langsmith/experimental/anthropic` to automatically trace agent queries, tool calls, and MCP operations.

## Phase 1: Pre-flight

### Check if already applied

Grep `container/agent-runner/src/index.ts` for `wrapClaudeAgentSDK`. If present, skip to Phase 3 (Setup) — the code changes are already in place.

### Verify langsmith is installed

```bash
cd container/agent-runner && node -e "require.resolve('langsmith/experimental/anthropic')" 2>&1
```

If not found, install it:

```bash
cd container/agent-runner && npm install langsmith@^0.5.8
```

## Phase 2: Apply Code Changes

### 2a. Pass LangSmith secrets to containers

**File**: `src/container-runner.ts` — `readSecrets()` function

Add these to the allowlist array:

```typescript
'LANGSMITH_API_KEY',
'LANGSMITH_PROJECT',
'LANGSMITH_ENDPOINT',
'LANGSMITH_TRACING',
```

### 2b. Update .env.example

Append to `.env.example`:

```
# LangSmith Tracing (optional)
LANGSMITH_TRACING=true
LANGSMITH_API_KEY=
LANGSMITH_PROJECT=nanoclaw-production
LANGSMITH_ENDPOINT=https://api.smith.langchain.com
```

### 2c. Wrap the SDK in agent-runner

**File**: `container/agent-runner/src/index.ts`

Change the import:

```typescript
// Before:
import { query, HookCallback, PreCompactHookInput, PreToolUseHookInput } from '@anthropic-ai/claude-agent-sdk';

// After:
import * as claudeAgentSdk from '@anthropic-ai/claude-agent-sdk';
import type { HookCallback, PreCompactHookInput, PreToolUseHookInput } from '@anthropic-ai/claude-agent-sdk';
import { wrapClaudeAgentSDK } from 'langsmith/experimental/anthropic';
import { RunTree } from 'langsmith';

let { query } = claudeAgentSdk;
let tracingEnabled = false;
```

In `main()`, after the `sdkEnv` loop and before the query loop, add:

```typescript
// Inject LangSmith vars into process.env so the tracing SDK can read them.
for (const key of ['LANGSMITH_API_KEY', 'LANGSMITH_PROJECT', 'LANGSMITH_ENDPOINT', 'LANGSMITH_TRACING']) {
  if (sdkEnv[key]) process.env[key] = sdkEnv[key];
}

if (process.env.LANGSMITH_TRACING === 'true' && process.env.LANGSMITH_API_KEY) {
  query = wrapClaudeAgentSDK(claudeAgentSdk, { name: containerInput.groupFolder }).query;
  tracingEnabled = true;
  log('LangSmith tracing enabled');
}
```

### 2d. Flush traces before exit

**Critical**: Without this, the container exits before async trace uploads complete, leaving traces stuck in `pending` status.

Add a `let exitCode = 0;` before the try/catch query loop. Change `process.exit(1)` in the catch block to `exitCode = 1;`. After the try/catch, add:

```typescript
if (tracingEnabled) {
  try {
    await RunTree.getSharedClient().flush();
    log('LangSmith traces flushed');
  } catch (e) {
    log(`LangSmith flush failed: ${e instanceof Error ? e.message : String(e)}`);
  }
}

if (exitCode !== 0) process.exit(exitCode);
```

### Validate

```bash
cd container/agent-runner && npx tsc --noEmit
```

## Phase 3: Setup

### Collect API key

AskUserQuestion: Do you have a LangSmith API key? If not, create one at https://smith.langchain.com/settings

### Configure environment

Add to `.env`:

```bash
LANGSMITH_TRACING=true
LANGSMITH_API_KEY=<their-key>
LANGSMITH_PROJECT=nanoclaw-production
LANGSMITH_ENDPOINT=https://api.smith.langchain.com
```

Sync to container environment:

```bash
mkdir -p data/env && cp .env data/env/env
```

### Rebuild container

```bash
./container/build.sh
```

## Phase 4: Verify

1. Send a test message to any registered group
2. Check https://smith.langchain.com for traces under the `nanoclaw-production` project
3. Each trace should show: agent query, tool calls, and model interactions
4. Verify graceful degradation: set `LANGSMITH_TRACING=false` in `.env`, sync, restart — agent should work normally with no errors

## Troubleshooting

### Traces stuck in `pending` status

The flush step (2d) is missing. Without `RunTree.getSharedClient().flush()` before exit, the container process terminates before async trace uploads complete. Verify container logs show "LangSmith traces flushed".

### No traces appearing

1. Check `LANGSMITH_API_KEY` is set in `.env` AND synced to `data/env/env`
2. Check container logs for "LangSmith tracing enabled" message
3. Verify the API key is valid at https://smith.langchain.com/settings
4. Check `LANGSMITH_ENDPOINT` matches your LangSmith instance (default: `https://api.smith.langchain.com`)

### Container build fails

Ensure `langsmith` >= 0.5.8 is in `container/agent-runner/package.json`. Run:

```bash
cd container/agent-runner && npm install langsmith@^0.5.8
```

Then rebuild: `./container/build.sh`
