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

---

## End-to-end smoke test results (2026-05-08)

Three ACP adapters tested directly against Ollama (not through OpenClaw `sessions_spawn`) using a Node test harness that runs the full ACP wire-protocol sequence: `initialize` → `session/new` → `session/prompt`. The prompt is *"Reply with just the single word PONG and nothing else"*. The harness watches stdout for the model's reply and reports the result, timing, and adapter capabilities.

The harness lives at `/tmp/acp-pong.mjs` on the VPS. Invoke as:

```
env <env-vars> node /tmp/acp-pong.mjs <agent-command...>
```

The agent commands come from acpx's built-in registry (`extensions/acpx/node_modules/acpx/skills/acpx/SKILL.md` in our fork — also visible upstream at `openclaw/acpx`).

### Agent: `claude` → `npx -y @agentclientprotocol/claude-agent-acp`

**Status: WORKS.** PONG received end-to-end via Ollama.

Env required:
```
ANTHROPIC_BASE_URL=http://127.0.0.1:11434
ANTHROPIC_AUTH_TOKEN=ollama
ANTHROPIC_MODEL=deepseek-v4-pro:cloud
```

Result: `ok: true`, `durationMs: 14861`, `chunkCount: 27`. Model output included reasoning ("The user wants me to reply with just \"PONG\" and nothing else.") followed by `PONG`.

Adapter capabilities (from `initialize` response):

```json
{
  "_meta": { "claudeCode": { "promptQueueing": true } },
  "promptCapabilities": { "image": true, "embeddedContext": true },
  "mcpCapabilities": { "http": true, "sse": true },
  "loadSession": true,
  "sessionCapabilities": { "fork": {}, "list": {}, "resume": {}, "close": {} }
}
```

OpenClaw integration note: this adapter does NOT advertise `session/set_mode` in its capabilities (no `controls` field at all in the initialize response). OpenClaw's main agent must call `sessions_spawn` without a `mode` parameter, otherwise `src/acp/control-plane/manager.runtime-controls.ts` throws `AcpRuntimeError: Could not apply ACP runtime options before turn execution.` before the prompt is ever sent. Same applies to the `codex` adapter below.

### Agent: `codex` → `npx @zed-industries/codex-acp`

**Status: WORKS.** PONG received end-to-end via Ollama. Fastest and terstest of the three.

Env required:
```
OPENAI_BASE_URL=http://127.0.0.1:11434/v1
OPENAI_API_KEY=ollama
OPENAI_MODEL=deepseek-v4-pro:cloud
```

Result: `ok: true`, `durationMs: 8517`, `chunkCount: 7`. Model output: `PONG` — no reasoning preamble, no extra text. Roughly 2x faster than claude on the same model.

Adapter capabilities:

```json
{
  "loadSession": true,
  "promptCapabilities": { "image": true, "audio": false, "embeddedContext": true },
  "mcpCapabilities": { "http": true, "sse": false },
  "sessionCapabilities": { "list": {}, "close": {} },
  "auth": { "logout": {} }
}
```

This is Zed's adapter that wraps the OpenAI Codex CLI via stdio and routes underlying chat completions to whatever `OPENAI_BASE_URL` points at. Because Ollama serves an OpenAI-compatible `/v1/chat/completions` endpoint at `http://127.0.0.1:11434/v1`, no protocol translation is needed — codex-acp talks to it like it would talk to api.openai.com.

OpenClaw fork already has special-case handling for the `codex` agent ID in `extensions/acpx/src/runtime.ts`: `CODEX_ACP_THINKING_ALIASES` (maps OpenClaw thought-level values to codex-acp `reasoning_effort`), `CODEX_ACP_REASONING_EFFORTS`, and OpenAI-prefix model normalization. None of the other adapters get that treatment — codex is the most thoroughly integrated ACP path in our fork.

### Agent: `pi` → `npx pi-acp`

**Status: DOES NOT WORK with Ollama.** Wrong product.

Env: tried with no overrides.

Result: `ok: false`, `durationMs: 1807`, error: `Authentication required: Configure an API key or log in with an OAuth provider.`

The `pi-acp` npm package is **not** OpenClaw's embedded PI runtime — it is a separate product (Inflection's Pi assistant) that requires its own API key or OAuth. It cannot be redirected to Ollama or any other model backend.

The acpx alias `pi → npx pi-acp` exists for the original Pi product. If you want "OpenClaw's own runtime served as an ACP harness" instead, the right alias is `openclaw → openclaw acp` — that loops back to the local OpenClaw gateway and inherits whichever models the gateway is configured to use (already Ollama in this setup, no extra config). Untested in this round but logically the cleaner OpenClaw-native path.

### Recommendation

For new ACP-driven sub-agent work, **prefer `codex`** (Zed's adapter):

- Faster: ~8.5s vs ~14.8s to first reply on the same Ollama model
- Terser: model emits the answer directly without thinking preamble
- Most-tested in our fork: explicit `CODEX_ACP_THINKING_ALIASES` etc. in `extensions/acpx/src/runtime.ts`
- Simpler env: just `OPENAI_BASE_URL` + `OPENAI_API_KEY=ollama`, no Anthropic header gymnastics
- Capability surface advertised in initialize is similar to claude's — same `session/set_mode`-not-advertised story

Suggested ACP config block for `~/.openclaw/openclaw.json` (replaces the previous `claude` agent override):

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
            "codex": {
              "command": "/usr/bin/env OPENAI_BASE_URL=http://127.0.0.1:11434/v1 OPENAI_API_KEY=ollama OPENAI_MODEL=deepseek-v4-pro:cloud npx -y @zed-industries/codex-acp"
            }
          }
        }
      }
    }
  }
}
```

Then call `sessions_spawn` with `agentId: "codex"` instead of `agentId: "claude"`.

### Open question — Path A through the gateway

Both adapters that worked (claude, codex) omit the `session/set_mode` control from their advertised capabilities. OpenClaw will throw `AcpRuntimeError: Could not apply ACP runtime options before turn execution.` whenever `sessions_spawn` is called with a non-empty `mode`. The proven-to-work path is to call `sessions_spawn` **without** a `mode` parameter — the agent's prompt or the cron payload should not set it.

This was not yet verified end-to-end through the OpenClaw gateway in this round (`openclaw message send` is outbound-only and bypasses the auto-reply pipeline; an inbound test message from the user is needed). The adapter side is fully proven via the direct-pipe smoke tests above.

### Reproduction

The test harness `/tmp/acp-pong.mjs`:

```js
import { spawn } from "node:child_process";

const args = process.argv.slice(2);
const child = spawn(args[0], args.slice(1), { stdio: ["pipe", "pipe", "pipe"], env: process.env });

const responses = new Map();
const allChunks = [];
let buffer = "";

child.stdout.on("data", (data) => {
  buffer += data.toString();
  const lines = buffer.split("\n");
  buffer = lines.pop();
  for (const line of lines) {
    if (!line.trim()) continue;
    try {
      const msg = JSON.parse(line);
      allChunks.push(msg);
      if (msg.id !== undefined && responses.has(msg.id)) {
        const resolve = responses.get(msg.id);
        responses.delete(msg.id);
        resolve(msg);
      }
    } catch {}
  }
});

const send = (msg) => new Promise((resolve) => {
  if (msg.id !== undefined) responses.set(msg.id, resolve);
  child.stdin.write(JSON.stringify(msg) + "\n");
  if (msg.id === undefined) resolve();
});

const timed = (p, ms, label) => Promise.race([
  p,
  new Promise((_, rej) => setTimeout(() => rej(new Error(`${label} timeout`)), ms)),
]);

const result = { ok: false, sessionId: null, modelOutput: "", chunkCount: 0 };

try {
  const initResp = await timed(send({
    jsonrpc: "2.0", id: 1, method: "initialize",
    params: { protocolVersion: 1, clientCapabilities: { fs: { readTextFile: false, writeTextFile: false }, terminal: false }, clientInfo: { name: "pong-test", version: "1.0" } },
  }), 20000, "initialize");
  if (initResp.error) throw new Error(initResp.error.message);

  const sessResp = await timed(send({
    jsonrpc: "2.0", id: 2, method: "session/new",
    params: { cwd: "/tmp", mcpServers: [] },
  }), 30000, "session/new");
  if (sessResp.error) throw new Error(sessResp.error.message);
  result.sessionId = sessResp.result?.sessionId;

  await timed(send({
    jsonrpc: "2.0", id: 3, method: "session/prompt",
    params: { sessionId: result.sessionId, prompt: [{ type: "text", text: "Reply with just the single word PONG and nothing else." }] },
  }), 120000, "session/prompt");

  let collected = "";
  for (const c of allChunks) {
    const tryGetText = (obj) => {
      if (!obj || typeof obj !== "object") return;
      if (typeof obj.text === "string") collected += obj.text + " ";
      if (Array.isArray(obj)) obj.forEach(tryGetText);
      else for (const v of Object.values(obj)) tryGetText(v);
    };
    tryGetText(c);
  }
  result.modelOutput = collected.slice(0, 600);
  result.ok = true;
} catch (e) {
  result.error = e.message;
}
result.chunkCount = allChunks.length;
console.log(JSON.stringify(result, null, 2));
child.stdin.end();
child.kill("SIGTERM");
```

Run any of:

```bash
# claude
env ANTHROPIC_BASE_URL=http://127.0.0.1:11434 ANTHROPIC_AUTH_TOKEN=ollama ANTHROPIC_MODEL=deepseek-v4-pro:cloud \
  node /tmp/acp-pong.mjs npx -y @agentclientprotocol/claude-agent-acp

# codex (recommended)
env OPENAI_BASE_URL=http://127.0.0.1:11434/v1 OPENAI_API_KEY=ollama OPENAI_MODEL=deepseek-v4-pro:cloud \
  node /tmp/acp-pong.mjs npx -y @zed-industries/codex-acp

# pi (will fail unless you have inflection.ai auth)
node /tmp/acp-pong.mjs npx -y pi-acp
```
