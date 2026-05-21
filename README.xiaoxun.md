# xiaoxun fork of OpenCode

Soft fork adding lifecycle hooks, silent agents, and multi-pass compaction.

**Branch**: `dev` (tracks upstream, rebase-friendly)
**Upstream**: [anomalyco/opencode](https://github.com/anomalyco/opencode)

## Changes (4 files, +369/-66 lines)

### 1. Lifecycle hooks — `packages/plugin/src/index.ts` (+29)

Two new plugin hooks that fire at precise points in the turn lifecycle:

- **`chat.turn.prestart`** — fires before LLM processing begins (step===1 only). Plugins can inject memory recall results and topic context into the system prompt or as synthetic user message text. Supports `syncTasks` for spawning sub-agents synchronously and collecting their output.
- **`chat.turn.end`** — fires after the LLM loop exits. Plugins receive the last user and assistant message text. Supports `tasks` for spawning background sub-agents.

### 2. Hook trigger points — `packages/opencode/src/session/prompt.ts` (+103)

- **prestart**: fires on step===1 before the LLM call. Runs sync tasks, collects output, prepends context text to user message parts.
- **end**: fires after the while loop exits. Spawns background agent tasks forked to not block response.
- Captures user/assistant message text for both hooks.

### 3. Silent task mode — `packages/opencode/src/tool/task.ts` (+11)

Added `silent` parameter to the `task` tool. When `silent: true` with `background: true`:
- No toast notification
- No synthetic result message injection
- Agent runs invisibly — useful for automated maintenance (memory save, topic check, compaction)

### 4. Multi-pass compaction — `packages/opencode/src/session/compaction.ts` (+292/-66)

Configurable via `compaction.type` in opencode.json:

```json
{ "compaction": { "type": "multi-pass" } }
```

When enabled (default preserves original single-pass behavior):

1. **Preserved zone** — recent messages within `preserve_recent_tokens` budget (default 80K), kept verbatim
2. **Summary zone** — oldest 50% of head messages by token count → single LLM call → narrative summary
3. **Compression zone** — newer 50% of head messages → per-message LLM calls (parallel, concurrency=8) → writes compressed text back to original message parts via `session.updatePart()`

Compression rules (sentence-level classification):
- Emotional/personal/relational content → kept verbatim
- Technical/tool/code content → summarized, key info preserved

## Configuration

```json
{
  "compaction": {
    "type": "multi-pass",
    "preserve_recent_tokens": 80000
  }
}
```

Only `type` is new. All other fields are upstream-native.

## Design principles

- **Soft fork** — all changes are additive, original code preserved in `else` branches
- **Upstream-tracked** — regularly synced via `gh repo sync`
- **Config-gated** — multi-pass features only activate when configured
- **English-only** — no localized strings in source changes

## Companion files (not in this repo)

- `~/.config/opencode/agents/memory-recall.md` — memory recall sub-agent
- `~/.config/opencode/agents/memory-save.md` — memory save sub-agent
- `~/.config/opencode/agents/topic-inject.md` — topic injection sub-agent
- `~/.config/opencode/agents/topic-check.md` — topic detection sub-agent
- `~/.config/opencode/command/smart-compact.md` — manual compaction command
- `xiaoxun/plugin/index.ts` — lifecycle plugin using the new hooks
