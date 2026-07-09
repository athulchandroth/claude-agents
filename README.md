# claude-agents

Multi-agent patterns for Claude Code: a strong model (Fable 5) supplies judgment, a fast model (Sonnet) does the mechanical work.

| Pattern | Files | Shape |
|---|---|---|
| **Executor + Advisor** | `sonnet-executor.md`, `fable-advisor.md` | Sonnet does the work and consults Fable at decision points — before committing to an approach, when stuck, and before declaring done |
| **Orchestrator + Workers** | `fable-orchestrator.md`, `sonnet-worker.md` | Fable reads, plans, and validates; fans independent sub-tasks out to parallel Sonnet workers |

## Install

Copy the `.md` files into `.claude/agents/` in your project (or `~/.claude/agents/` for all projects).

`sonnet-executor` and `fable-orchestrator` are entry points — run them as your main agent (they need the `Task` tool to spawn the others). `fable-advisor` and `sonnet-worker` are the agents they spawn; you don't normally invoke those directly.

## Design notes

The prompting in these agents adapts the measured guidance from Anthropic's official [advisor tool documentation](https://platform.claude.com/docs/en/agents-and-tools/tool-use/advisor-tool) to Claude Code subagents:

- **Call timing:** consult *before substantive work* (orientation reads don't count), once before committing to an approach and once before declaring done. Make the deliverable durable before the done-check.
- **Advice weight:** follow the advice unless it fails empirically; on a conflict between gathered evidence and the advice, do one reconcile consult instead of silently switching.
- **Advisor brevity:** advisor output length is the pattern's main cost driver, so the advisor is instructed to stay under ~200 words.

One structural difference from the API advisor tool is load-bearing: **the API advisor automatically receives the executor's entire transcript; a Claude Code subagent sees only the prompt it's handed.** That's why `sonnet-executor` is required to pass task + intent + evidence + file paths on every consult, and why `fable-advisor` reads the named files itself before advising.

Other Claude Code realities baked in:

- Subagents share nothing — no history, no prompt cache, no knowledge of sibling sub-tasks. The orchestrator writes self-contained worker prompts and owns cross-file consistency.
- Parallel workers must not edit the same file; same-file sub-tasks run sequentially.

## Building on the API instead?

If you're building your own agent on the Messages API rather than Claude Code, use the server-side **advisor tool** — the executor pauses mid-generation, the advisor sees the full transcript automatically, and everything happens in one request:

```python
response = client.beta.messages.create(
    model="claude-sonnet-4-6",              # executor
    max_tokens=4096,
    betas=["advisor-tool-2026-03-01"],
    tools=[{
        "type": "advisor_20260301",
        "name": "advisor",
        "model": "claude-opus-4-8",          # advisor — must be ≥ executor capability
        "max_tokens": 2048,                  # caps advisor output (recommended starting point)
    }],
    messages=[{"role": "user", "content": "..."}],
)
```

Notes from the official docs: the advisor model must be at least as capable as the executor (invalid pairs 400); a Fable 5 executor can only be advised by Fable 5, and Fable/Mythos advisors return `advisor_redacted_result` (encrypted — round-trip it verbatim); pass `advisor_tool_result` blocks back on subsequent turns, and if you later drop the advisor tool, strip those blocks from history too or the request 400s; advisor usage is billed at the advisor model's rates and reported per-iteration in `usage.iterations[]`; enable the tool-level `caching` option only when you expect ≥3 advisor calls per conversation.
