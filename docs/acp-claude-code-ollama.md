# ACP Claude Code + Ollama Integration

## Date
2026-05-07

## Author
Gobling (Ariel's assistant)

## Context
Ariel wanted to run Claude Code as an ACP harness backed by ollama models (specifically `kimi-k2.6:cloud`). I initially claimed this was impossible. I was wrong.

## What I Got Wrong
- Claimed Claude Code can only use Anthropic API models
- Claimed `ollama launch claude` is not a real command
- Failed to read the code before making claims

## The Truth

### 1. Ollama supports Claude Code
**Source:** https://docs.ollama.com/integrations/claude-code

```
ollama launch claude --model kimi-k2.6:cloud
```

Ollama provides an Anthropic-compatible API shim. Claude Code connects to `http://localhost:11434` instead of Anthropic's API.

### 2. The ollama CLI confirms this
```bash
$ ollama launch --help
...
Supported integrations:
  claude    Claude Code
...
Flags:
  --model string   Model to use
  -y, --yes        Automatically answer yes to confirmation prompts
```

### 3. acpx supports custom agent commands
**Source:** `/opt/openclaw-dev-mode/extensions/acpx/src/config-schema.ts:36`
```typescript
agents?: Record<string, { command: string }>;
```

**Source:** `/opt/openclaw-dev-mode/extensions/acpx/src/runtime.ts:362-371`
```typescript
function resolveAgentCommand(params: {
  agentName: string | undefined;
  agentRegistry: AcpAgentRegistry;
}): string | undefined {
  const normalizedAgentName = normalizeAgentName(params.agentName);
  const resolvedCommand = params.agentRegistry.resolve(normalizedAgentName);
  return typeof resolvedCommand === "string" ? resolvedCommand.trim() || undefined : undefined;
}
```

When you spawn `runtime: "acp", agentId: "claude"`, acpx looks up `"claude"` in the agent registry. If you configured a custom command, it uses that instead of the default `claude` binary.

### 4. How acpx loads the agent registry
**Source:** `/opt/openclaw-dev-mode/extensions/acpx/src/service.ts:63`
```typescript
agentRegistry: module.createAgentRegistry({
  overrides: params.pluginConfig.agents,
}),
```

The `agents` object from `plugins.entries.acpx.config.agents` is passed as overrides to the agent registry.

## Required Config

Add to `/root/.openclaw/openclaw.json`:

```json
{
  "plugins": {
    "entries": {
      "acpx": {
        "enabled": true,
        "config": {
          "permissionMode": "approve-all",
          "nonInteractivePermissions": "fail",
          "timeoutSeconds": 120,
          "agents": {
            "claude": {
              "command": "ollama launch claude --model kimi-k2.6:cloud --yes"
            }
          }
        }
      }
    }
  }
}
```

## What Happens When You Spawn

1. You call `sessions_spawn({ runtime: "acp", agentId: "claude", ... })`
2. OpenClaw routes to acpx plugin
3. acpx resolves agent `"claude"` via `resolveAgentCommand()`
4. Finds custom command: `"ollama launch claude --model kimi-k2.6:cloud --yes"`
5. Spawns that process instead of raw `claude`
6. Claude Code starts, connects to ollama's Anthropic-compatible API at `http://localhost:11434`
7. ollama routes requests to the `kimi-k2.6:cloud` model
8. Full Claude Code functionality (filesystem, git, shell) works with kimi model

## Bossy Cron Pattern (Shopping App)

```json
{
  "name": "dark-factory-v2:shopmate-bossy-ui",
  "schedule": {
    "kind": "every",
    "everyMs": 3600000
  },
  "payload": {
    "kind": "agentTurn",
    "message": "SOURCE OF TRUTH: /root/.openclaw/workspace/shopping-app/TODO.md...",
    "model": "ollama/kimi-k2.6:cloud",
    "runtime": "acp",
    "agentId": "claude",
    "cwd": "/root/.openclaw/workspace/shopping-app",
    "mode": "run",
    "timeoutSeconds": 2700
  },
  "sessionTarget": "isolated",
  "delivery": {
    "mode": "announce",
    "channel": "whatsapp",
    "to": "+972542634114"
  }
}
```

## Cron Spec Rules

When writing cron specs that use ACP Claude Code:

1. **Use `agentId: "claude"`** — this maps to the acpx agent registry key
2. **Custom command override** goes in `plugins.entries.acpx.config.agents.claude.command`
3. **Model** in the cron payload is for OpenClaw's routing, not Claude Code's model
4. **Claude Code's actual model** is set via the `--model` flag in the custom command
5. **`runtime: "acp"`** tells OpenClaw to use the acpx plugin
6. **`mode: "run"`** means one-shot execution (not persistent chat)

## Correct Way to Write the Bossy Spec

```
- At most one ACP Claude Code coding worker: runtime=acp, agentId=claude, model=ollama/kimi-k2.6:cloud (OpenClaw routing), cwd=/root/.openclaw/workspace/shopping-app, mode=run, timeout <=45 min. (Claude Code runs via `ollama launch claude --model kimi-k2.6:cloud` set in acpx config)
```

Or more explicitly:

```json
{
  "payload": {
    "kind": "agentTurn",
    "message": "Fix bugs in shopping app...",
    "model": "ollama/kimi-k2.6:cloud",
    "runtime": "acp",
    "agentId": "claude",
    "cwd": "/root/.openclaw/workspace/shopping-app",
    "mode": "run",
    "timeoutSeconds": 2700
  }
}
```

With this in `openclaw.json`:

```json
{
  "plugins": {
    "entries": {
      "acpx": {
        "enabled": true,
        "config": {
          "agents": {
            "claude": {
              "command": "ollama launch claude --model kimi-k2.6:cloud --yes"
            }
          }
        }
      }
    }
  }
}
```

## Lessons

1. **Read the code before claiming something is impossible**
2. **Check ollama docs** — they integrate with more tools than expected
3. **acpx agent registry** is the correct abstraction for custom harness commands
4. **Model selection happens at two levels:**
   - OpenClaw routing: `model` field in spawn/payload
   - Harness execution: `--model` flag in the command override

## References
- Ollama Claude Code docs: https://docs.ollama.com/integrations/claude-code
- acpx config schema: `/opt/openclaw-dev-mode/extensions/acpx/src/config-schema.ts`
- acpx runtime: `/opt/openclaw-dev-mode/extensions/acpx/src/runtime.ts`
- acpx service: `/opt/openclaw-dev-mode/extensions/acpx/src/service.ts`
