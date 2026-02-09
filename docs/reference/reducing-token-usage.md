---
summary: "Practical guide for reducing token consumption in OpenClaw agent sessions"
read_when:
  - Agent is burning through tokens too quickly
  - You want to optimize API costs
  - You need to tune context window usage
title: "Reducing Token Usage"
---

# Reducing Token Usage

This guide covers practical ways to reduce how many tokens your OpenClaw agent
consumes per session. Changes are ordered by impact — start at the top.

## 1. Diagnose first

Before tuning, see where tokens actually go:

```
/context detail
```

This shows a per-component breakdown: system prompt size, injected workspace
files (with raw vs truncated sizes), tool schema sizes, and skills overhead.
Also run `/status` to see current context window fill percentage.

## 2. Switch to a cheaper or smaller model

The single biggest lever. Smaller models use fewer tokens per response and cost
less per token.

```json5
{
  agent: {
    model: { primary: "anthropic/claude-sonnet-4-5" },  // instead of opus
  },
}
```

Use `sessions_spawn` with a cheaper model for sub-tasks, keeping the expensive
model for the main session only:

```json5
{
  agent: {
    subagents: {
      model: "anthropic/claude-haiku-3-5",
      thinking: "off",
    },
  },
}
```

## 3. Trim injected workspace files

Workspace files (`AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`)
are injected into **every** system prompt. Large files are the #1 source of
system prompt bloat.

**Check sizes:**

```
/context list
```

**Reduce the per-file cap:**

```json5
{
  agent: {
    bootstrapMaxChars: 10000,  // default: 20000
  },
}
```

**Keep files lean:**

- Remove verbose instructions from `AGENTS.md` — the model doesn't need a
  novel; bullet points work.
- Split rarely-needed content into separate files the agent can `read` on
  demand instead of injecting everything.
- Delete `TOOLS.md` entries for tools that are rarely used — the model already
  sees tool schemas.

## 4. Enable context pruning (cache-ttl mode)

This trims old tool results from the context before each LLM call, preventing
long sessions from ballooning:

```json5
{
  agent: {
    contextPruning: {
      mode: "cache-ttl",
      ttl: "5m",
      keepLastAssistants: 3,
    },
  },
}
```

For Anthropic models, this also reduces cache-write costs after idle periods.

## 5. Tune compaction aggressively

Compaction summarizes older messages. Lower the history share to compact earlier:

```json5
{
  agent: {
    compaction: {
      maxHistoryShare: 0.3,          // default: 0.5 (50% of context)
      reserveTokensFloor: 30000,     // default: 20000
    },
  },
}
```

Use `/compact` manually when a session feels bloated, optionally with focus
instructions:

```
/compact Focus on decisions and open questions only
```

## 6. Limit DM history depth

For messaging channels (WhatsApp, Telegram, etc.), limit how many past turns
are sent to the model:

```json5
{
  channels: {
    whatsapp: {
      dmHistoryLimit: 20,       // only last 20 user turns
    },
    telegram: {
      dmHistoryLimit: 15,
    },
  },
}
```

Per-user overrides are also supported:

```json5
{
  channels: {
    whatsapp: {
      dms: {
        "+15555550123": { historyLimit: 10 },
      },
    },
  },
}
```

## 7. Reduce tool count via policy

Every tool schema counts toward context (often 8-30K tokens total). Disable
tools your agent doesn't need:

```json5
{
  agent: {
    tools: {
      policy: {
        deny: ["browser", "canvas", "nodes", "image", "cron"],
      },
    },
  },
}
```

Check which tools consume the most with `/context detail` — `browser` and
`exec` schemas are typically the largest.

## 8. Minimize skills overhead

Skills metadata is injected into the system prompt even when not used. Disable
skills you don't need:

```json5
{
  skills: {
    disabled: ["notion", "obsidian", "linear", "jira"],
  },
}
```

Or restrict to an explicit allowlist by only installing the skills you need.

## 9. Turn off thinking for routine work

Extended thinking (reasoning) adds significant token overhead. Use it only when
needed:

```json5
{
  agent: {
    thinkingDefault: "off",   // or "minimal" for light reasoning
  },
}
```

You can still toggle per-message with `/think high` when complex reasoning is
needed.

## 10. Use prompt caching effectively (Anthropic)

Anthropic prompt caching makes repeated system prompts nearly free on cache
hits. Keep your cache warm with heartbeat:

```json5
{
  agent: {
    model: {
      primary: "anthropic/claude-sonnet-4-5",
    },
    heartbeat: {
      every: "55m",            // just under the 1h cache TTL
    },
  },
}
```

**Cache stability tips:**

- Avoid rapidly changing workspace files (timestamps, counters) — they
  invalidate the cache.
- The system prompt deliberately excludes the current time for this reason.

## 11. Cap the context window

If your model reports a large context window (e.g. 200K), you can cap it to
force earlier compaction:

```json5
{
  agent: {
    contextTokens: 64000,     // force compaction at ~64K instead of 200K
  },
}
```

This trades history depth for lower per-request token consumption.

## Summary table

| Technique                    | Typical savings    | Effort   |
|------------------------------|--------------------|----------|
| Smaller model                | 30-70% cost        | Config   |
| Trim workspace files         | 5-50K tokens/req   | Content  |
| Enable context pruning       | 10-40% on long sessions | Config |
| Lower `maxHistoryShare`      | Fewer overflow retries  | Config |
| Limit DM history             | 20-60% for chat channels | Config |
| Reduce tool count            | 5-30K tokens/req   | Config   |
| Disable unused skills        | 1-10K tokens/req   | Config   |
| Turn off thinking            | 20-50% output tokens | Config |
| Prompt caching + heartbeat   | 80-90% input cost  | Config   |
| Cap context window           | Forces earlier compaction | Config |

See also: [Token Use & Costs](/reference/token-use),
[Compaction](/concepts/compaction),
[Session Pruning](/concepts/session-pruning),
[Context](/concepts/context).
