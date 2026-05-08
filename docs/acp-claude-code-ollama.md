# ACP Claude Code + Ollama Integration

## Date
2026-05-08 (rewrite — supersedes the 2026-05-07 version)

## Author
Gobling (Ariel's assistant), corrected after the 2026-05-07/08 ACP debugging session.

## Context
Ariel wanted Claude Code to run as an ACP harness backed by Ollama models (`kimi-k2.6:cloud` initially, then `deepseek-v4-pro:cloud`). The 2026-05-07 version of this doc said `ollama launch claude --model X --yes` was the correct ACP command. **That was wrong.** This rewrite documents what actually works and why the previous instructions produced 17-minute hung `sessions_spawn` calls.

## What the Previous Version Got Wrong

The previous instruction was:
```json
"claude": { "command": "ollama launch claude --model kimi-k2.6:cloud --yes" }
```

This runs interactive Claude Code (the ELF binary at `/root/.local/bin/claude`). **Interactive Claude Code does not speak the ACP wire protocol on stdio.** It reads stdin as user chat messages and replies conversationally. When acpx pipes JSON-RPC `initialize` into stdin, Claude Code treats it as a question:

```
$ echo '{"jsonrpc":"2.0","id":1,"method":"initialize",...}' \
    | ollama launch claude --model deepseek-v4-pro:cloud --yes

Launching Claude Code with deepseek-v4-pro:cloud...
I see you're sending a JSON-RPC `initialize` request — this looks like a
Model Context Protocol (MCP) or Language Server Protocol handshake. I'm
Claude Code, an interactive CLI agent, not an MCP/LSP server. I don't
speak the LSP/MCP wire protocol directly.
```

That's why `sessions_spawn` hung for 17+ minutes with kimi-k2.6:cloud — the model never produced its conversational reply (slow + unstable cloud routing). With deepseek-v4-pro:cloud the chat reply came fast, but ACP was still broken: acpx kept waiting for a JSON-RPC `result`, the subprocess kept producing natural-language sentences. Wrong protocol, wrong layer.

## The Correct Approach

Two pieces compose:

### 1. Use `@agentclientprotocol/claude-agent-acp` as the ACP harness

The npm package `@agentclientprotocol/claude-agent-acp@0.33.1` IS the ACP-speaking adapter for Claude Code. It wraps `@anthropic-ai/claude-agent-sdk` (which IS the engine that powers Claude Code — the SDK bundles the actual Claude Code binary as an optional dep) and bridges ACP JSON-RPC over stdio to/from the Claude Code agent loop.

Smoke test (verified 2026-05-08 on the VPS):

```
$ INIT='{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":1,"clientCapabilities":{"fs":{"readTextFile":false,"writeTextFile":false},"terminal":false},"clientInfo":{"name":"probe","version":"1.0"}}}'

$ echo "$INIT" | env \
    ANTHROPIC_BASE_URL=http://127.0.0.1:11434 \
    ANTHROPIC_AUTH_TOKEN=ollama \
    ANTHROPIC_MODEL=deepseek-v4-pro:cloud \
    npx -y @agentclientprotocol/claude-agent-acp

{"jsonrpc":"2.0","id":1,"result":{
  "protocolVersion":1,
  "agentCapabilities":{
    "_meta":{"claudeCode":{"promptQueueing":true}},
    "promptCapabilities":{"image":true,"embeddedContext":true},
    "mcpCapabilities":{"http":true,"sse":true},
    "loadSession":true,
    "sessionCapabilities":{"fork":{},"list":{},"resume":{},"close":{}}
  },
  "agentInfo":{
    "name":"@agentclientprotocol/claude-agent-acp",
    "title":"Claude Agent",
    "version":"0.33.1"
  }
}}
```

That's a real ACP `initialize` response. Wire protocol confirmed.

### 2. Point the SDK at Ollama's Anthropic Messages endpoint

Ollama 0.14+ serves an Anthropic Messages API at `/v1/messages`. It accepts the full Anthropic format — model, max_tokens, messages, system, tools, thinking, streaming via SSE — and translates internally to Ollama's native chat. Both `anthropic-version` header and `Authorization: Bearer <token>` are accepted-but-ignored — pass any string for the token.

Smoke test against the live VPS Ollama (0.20.4):

```
$ curl -X POST http://127.0.0.1:11434/v1/messages \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{"model":"deepseek-v4-pro:cloud","max_tokens":50,
         "messages":[{"role":"user","content":"reply with just PONG"}]}'

{"id":"msg_3e63b0f11069b79f4be20df2","type":"message","role":"assistant",
 "model":"deepseek-v4-pro",
 "content":[
   {"type":"thinking","thinking":"..."},
   {"type":"text","text":"PONG"}
 ],
 "stop_reason":"end_turn",
 "usage":{"input_tokens":9,"output_tokens":34}}
```

Byte-compatible with Anthropic's API, including `thinking` blocks. The SDK appends `/v1/messages` to `ANTHROPIC_BASE_URL`, so the base must be the bare origin (`http://127.0.0.1:11434`), not `http://127.0.0.1:11434/v1`.

OpenClaw already uses this endpoint internally — `~/.openclaw/openclaw.json` lists `kimi-k2.6:cloud` and `minimax-m2.5:cloud` with `"api": "anthropic-messages"`, so the same routing path is reused.

## Required Config

In `/root/.openclaw/openclaw.json`:

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
              "command": "/usr/bin/env ANTHROPIC_BASE_URL=http://127.0.0.1:11434 ANTHROPIC_AUTH_TOKEN=ollama ANTHROPIC_MODEL=deepseek-v4-pro:cloud npx -y @agentclientprotocol/claude-agent-acp"
            }
          }
        }
      }
    }
  }
}
```

Why `/usr/bin/env` prefix: the acpx plugin's command schema accepts only a single `command` string — no separate `env` block. `/usr/bin/env VAR=val ...` is the portable way to set env vars for one subprocess invocation, working whether acpx exec's the command directly or shells it.

The `plugins.entries.acpx.config.agents.claude.command` key is **hot-reloadable**. Changing it triggers `config hot reload applied` in the gateway log within ~3 seconds, no gateway restart needed.

## Why the Old Path "Worked Interactively"

`ollama launch claude --model deepseek-v4-pro:cloud` does work — just not as an ACP harness. It launches the same Claude Code REPL with Ollama configured as the model provider. From a terminal, that gives a fine interactive chat session with deepseek as the brain. It just isn't ACP-spawnable, so OpenClaw can't drive it programmatically.

## You Can Have All Three

You can pick TWO of {Claude Code agent loop, ACP wire protocol, Ollama models}. Until 2026-05-08 we thought you couldn't have all three. With `claude-agent-acp` + Ollama's `/v1/messages` compat, you can:

| Want | Command |
|---|---|
| **Claude Code agent loop + ACP + Ollama models** (this doc) | `/usr/bin/env ANTHROPIC_BASE_URL=http://127.0.0.1:11434 ANTHROPIC_AUTH_TOKEN=ollama ANTHROPIC_MODEL=<ollama-model-id> npx -y @agentclientprotocol/claude-agent-acp` |
| Claude Code agent loop + ACP + Anthropic API | `npx -y @agentclientprotocol/claude-agent-acp` (with `ANTHROPIC_API_KEY` in env) |
| Claude Code interactive REPL + Ollama (NO ACP) | `ollama launch claude --model <model>` (cannot be spawned by acpx) |

## What Happens When You Spawn

1. Caller invokes `sessions_spawn({ runtime: "acp", agentId: "claude", ... })`
2. OpenClaw routes to the acpx plugin
3. acpx looks up `"claude"` in the agent registry → finds custom command
4. Spawns the command as a subprocess; pipes JSON-RPC over its stdin/stdout
5. `claude-agent-acp` receives `initialize`, responds with agent capabilities
6. acpx sends `session/new`, then `session/prompt` with the user message
7. `claude-agent-acp` runs the Claude Code agent loop (read/write/bash/grep/etc.)
8. Each model call goes to `http://127.0.0.1:11434/v1/messages` with the configured model
9. Ollama translates internally, returns Anthropic-format SSE chunks
10. Claude Code processes chunks, emits tool calls, executes tools, iterates
11. Final result streams back via ACP to the parent session

## Bossy Cron Pattern (corrected)

```json
{
  "name": "dark-factory-v2:shopmate-bossy-ui",
  "schedule": { "kind": "every", "everyMs": 3600000 },
  "payload": {
    "kind": "agentTurn",
    "message": "Fix bugs in shopping app...",
    "model": "ollama/deepseek-v4-pro:cloud",
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

The cron payload's `model` field is for OpenClaw's parent-session routing. The actual Claude Code subagent uses whatever `ANTHROPIC_MODEL` env var is in the acpx command override.

## Limitations of the Ollama Anthropic Compat

- No prompt caching (Ollama doesn't implement it)
- No batch processing
- No `/v1/messages/count_tokens` endpoint — token counts are approximations
- Base64 images only (no URLs)
- `tool_choice` is structurally accepted but per Ollama docs not enforced
- Some cloud models are slow/unstable; `kimi-k2.6:cloud` caused 3 stuck-session incidents on 2026-05-07. `deepseek-v4-pro:cloud` has been reliable in testing — preferred for unattended ACP work.

If a session hangs, see `dev-mode/rescue.md` Pattern E (single-session model-call retry loop) in the openclaw-dev-mode repo.

## Lessons (round 2)

1. **An ELF binary at `/root/.local/bin/claude` IS Claude Code; Claude Code IS an interactive REPL, not an ACP server.** Different layer than the SDK.
2. **The npm package `@anthropic-ai/claude-agent-sdk` IS Claude Code's engine** — it bundles the actual binary as an optional dep, so when you install the SDK you pull the same binary. They are not separate codebases.
3. **`@agentclientprotocol/claude-agent-acp` wraps the SDK, gives you ACP wire protocol over stdio**, and shells out to the same Claude Code binary internally. This is the correct ACP harness for "Claude Code in an automated agent context."
4. **Always smoke-test the wire protocol before changing production config.** Pipe an `initialize` JSON-RPC into the configured command. If it returns a `result` object on stdout, ACP works. If it returns natural language ("I see you're sending..."), the command is wrong.
5. **Ollama's Anthropic compat at `/v1/messages` is real and current** — added in 0.14, working in 0.20.4 on this VPS. Returns thinking blocks properly. Auth and version headers are ignored.
6. **`ANTHROPIC_BASE_URL` must be the bare origin** (no `/v1` suffix) — the SDK appends `/v1/messages` itself.

## References

- Ollama Anthropic compat layer: https://docs.ollama.com/api/anthropic-compatibility
- Ollama Claude Code blog: https://ollama.com/blog/claude
- `@agentclientprotocol/claude-agent-acp`: https://github.com/agentclientprotocol/claude-agent-acp
- `@anthropic-ai/claude-agent-sdk`: https://github.com/anthropics/claude-agent-sdk-typescript
- Anthropic Agent SDK overview: https://code.claude.com/docs/en/agent-sdk/overview
- openclaw acpx plugin: https://github.com/openclaw/acpx
- Local fork rescue patterns: `/opt/openclaw-dev-mode/dev-mode/rescue.md` (Pattern E)
