# xiaoxun fork of OpenCode

Soft fork adding lifecycle hooks, silent agents, and multi-pass compaction.

**Branch**: `dev` (tracks upstream, rebase-friendly)
**Upstream**: [anomalyco/opencode](https://github.com/anomalyco/opencode)

## Changes (6 files, +446/-66 lines)

### 1. Lifecycle hooks — `packages/plugin/src/index.ts` (+29)

Two new plugin hooks that fire at precise points in the turn lifecycle:

- **`chat.turn.prestart`** — fires before LLM processing begins (step===1 only). Plugins can inject memory recall results and topic context into the system prompt or as synthetic user message text. Supports `syncTasks` for spawning sub-agents synchronously and collecting their output.
- **`chat.turn.end`** — fires after the LLM loop exits. Plugins receive the last user and assistant message text. Supports `tasks` for spawning background sub-agents.

### 2. Hook trigger points + compaction trigger — `packages/opencode/src/session/prompt.ts` (+119)

- **prestart**: fires on step===1 before the LLM call. Runs sync tasks, collects output, prepends context text to user message parts.
- **end**: fires after the while loop exits. Spawns background agent tasks forked to not block response.
- `trigger_tokens`: unified compaction threshold. When set, triggers compaction when total tokens exceed the value — works for both default and multi-pass compaction types. When not set, falls through to OC's native overflow detection.

### 3. Silent task mode — `packages/opencode/src/tool/task.ts` (+11)

Added `silent` parameter to the `task` tool. When `silent: true` with `background: true`:
- No toast notification
- No synthetic result message injection
- Agent runs invisibly — useful for automated maintenance (memory save, topic check, compaction)

### 4. Multi-pass compaction — `packages/opencode/src/session/compaction.ts` (+237/-66)

Configurable via `compaction.type` in opencode.json. When set to `"multi-pass"`, replaces the single LLM summary call with a three-zone pipeline:

1. **Preserved zone** — recent messages within `preserve_recent_tokens` budget (default 80K), kept verbatim
2. **Summary zone** — oldest 50% of head by token count → single LLM call → narrative summary
3. **Compression zone** — newer 50% of head → per-message LLM calls (parallel, concurrency=8) → writes compressed text back via `session.updatePart()`

Compression rules (sentence-level classification):
- Emotional / personal / relational content → kept verbatim
- Technical / tool / code content → summarized, key info preserved

When type is not `"multi-pass"`, OC's original single-pass compaction runs unchanged.

### 5. Queued message endpoint — server handler (+50)

New HTTP endpoint for external message injection with built-in queue:

**`POST /session/{sessionID}/prompt_queued`** — same payload as `prompt_async`, but waits until the session is idle before sending. If the session is busy, polls status every second and delivers as soon as the agent loop finishes. Guarantees messages are delivered in order without concurrent agent loops.

```
POST /session/{id}/prompt_queued
{ "parts": [{"type": "text", "text": "hello"}] }
→ 204 No Content (after session becomes idle)
```

## Configuration

All fields except `type` are upstream-native. `trigger_tokens` is general — it applies to both default and multi-pass compaction.

```json
{
  "compaction": {
    "type": "multi-pass",
    "trigger_tokens": 350000,
    "preserve_recent_tokens": 80000
  }
}
```

| Field | Default | Applies to |
|-------|---------|------------|
| `type` | `"default"` | — |
| `trigger_tokens` | none (use OC overflow) | both |
| `preserve_recent_tokens` | OC default (25% of usable) | both |
| `tail_turns` | 2 | both |
| `prune` | false | both |

## Design principles

- **Soft fork** — all changes are additive, original code preserved in `else` branches
- **Upstream-tracked** — regularly synced via `gh repo sync`
- **Config-gated** — features only activate when configured; no config = stock OC behavior
- **English-only** — no localized strings in source changes
