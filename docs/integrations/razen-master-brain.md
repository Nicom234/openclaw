# Integration: Razen Master Brain

[Razen](https://github.com/Nicom234/razen-master-brain) is a TanStack Start + Supabase app that ports OpenClaw's agent-core architecture into a private, localized framework under `/backend/agents`. This doc records what was carried over, what was deliberately dropped, and where the surfaces map.

## What was mirrored

| OpenClaw concept | Razen surface | Notes |
|---|---|---|
| `packages/sdk/src/types.ts` event union | `backend/agents/core/types.ts` | Trimmed to ten event types Razen actually emits (`run.started`, `assistant.delta`, `tool.call.*`, `skill.activated`, etc). |
| `packages/sdk/src/event-hub.ts` | `backend/agents/core/event-hub.ts` | Collapsed to a single `EventHub` class plus `makeEvent()` factory. |
| `packages/sdk/src/transport.ts` | `backend/agents/core/transport.ts` | Replaced gateway protocol with direct SSE against Lovable's OpenAI-compatible endpoint. |
| `packages/sdk/src/client.ts` agent-loop | `backend/agents/core/runtime.ts` | `AgentRuntime.run()` is an async generator yielding the same shape of events, with up to N tool-call rounds. |
| `skills/*/SKILL.md` markdown manifests | `backend/agents/skills/types.ts` + `builtin.ts` | Skills are TypeScript objects rather than parsed YAML — same fields (`whenToUse`, `whenNotToUse`, `toolNames`, `systemPrompt`). |
| OpenAI-style tool descriptors | `backend/agents/tools/types.ts` + `registry.ts` | `ToolRegistry.asOpenAI()` returns the exact shape Lovable / OpenAI / Anthropic accept. |
| Supabase persistence | `backend/agents/supabase/adapter.ts` | Memories, credits, conversations — pure data-access functions over the existing schema. |

## What was deliberately dropped

OpenClaw is a multi-channel, multi-runtime, multi-environment platform. Razen needed roughly 10% of that surface. The following were excluded by design:

- **Gateway protocol** — Razen runs are short-lived inside a Supabase edge function. No long-lived sessions, no remote runtime.
- **Plugin loader / SDK boundaries** — there is no third-party plugin surface in Razen, so `plugin-sdk` and the `extensions/` model were skipped.
- **Managed runtimes (Testbox, Cloud, ACP harnesses)** — the runtime is always the same Lovable gateway.
- **Approvals / idempotency machinery** — every Razen tool is read-only or trivially safe (calculator, memory recall). No approval prompts.
- **Channel adapters** — Razen is a web app; no Discord, iMessage, BlueBubbles, etc.

## Edge function wiring

```
client (RazenAssistant.tsx)
   │  POST /functions/v1/agent  (SSE)
   ▼
supabase/functions/agent/index.ts
   │  imports ../../../backend/agents/index.ts
   ▼
AgentRuntime.run(input, ctx)
   ├── SkillRegistry.match() → activates web-research, memory-recall, …
   ├── ToolRegistry.asOpenAI() → calculator, recall_memory
   └── streamLovable() → ai.gateway.lovable.dev
```

Events surface as two SSE channels:

- `data:` — OpenAI-shaped `{choices[0].delta.content}` chunks for the chat stream.
- `event: agent\ndata:` — control events (`skill.activated`, `tool.call.completed`, …) for the UI.

## Why a copy and not a dependency

OpenClaw is GPL-coupled, Node-targeted, and an order of magnitude larger than Razen needs. Razen's edge functions run Deno; the OpenClaw SDK targets Node. Vendoring the surface area Razen actually uses (~700 lines) keeps the dependency graph shallow and the audit trivial.

If OpenClaw's event-shape or tool-descriptor format ever changes in a way Razen needs, the contracts to re-sync are:

- `packages/sdk/src/types.ts` (event types)
- `packages/sdk/src/normalize.ts` (event normalization)
- `skills/<id>/SKILL.md` frontmatter (skill manifest fields)

## Maintainer notes

- Keep `backend/agents/**` dependency-free. New runtime deps belong in the edge function or the frontend.
- Skill prompts are spliced into the system prompt at runtime. Treat them as part of the contract — changing them shifts model behaviour.
- Tool execution runs inside the edge function process. Anything expensive (LLM calls, network) should be paid for via the credit deduction in the edge function, not buried inside a tool.
