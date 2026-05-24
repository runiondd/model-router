---
name: model-router
description: Use when deciding whether to delegate a task to a cheaper-model subagent for cost efficiency — code/symbol location, log scans, mechanical renames, boilerplate, writing tests, focused diff review, or single-file edits. Provides the break-even gate, the haiku/sonnet/opus routing taxonomy, and hard delegation guardrails. Always-on cost discipline.
---

# Model Router — Cost-Routing Discipline

A skill CANNOT switch the running model. It routes *work* to cheaper-model
subagents via the `Agent` tool's `model` param (`haiku` | `sonnet` | `opus`).
The main Opus thread stays the orchestrator and keeps reasoning work.

## Break-Even Gate (run FIRST, every time)

Delegation overhead is 300–800+ tokens per hand-off (prompt + isolation +
summarized result + coordination). Delegating small work LOSES money.

**Only delegate when BOTH hold:**
- Estimated mechanical work is **> ~1.5k tokens**, AND
- Coordination cost is **< 30%** of the expected savings.

If either fails → **do it inline on Opus. Stop.**

## Routing Taxonomy

Only truly mechanical, no-judgment work is haiku-eligible. Anything touching
product behavior, logic, or coupling stays on Opus regardless of file count.

| Tier | Use for | Dispatch to |
|------|---------|-------------|
| **haiku** (only when gate passes) | "where is X" code/symbol location; log/output scans for a known pattern; mechanical renames; pure format/whitespace edits; reading files to extract specific values; template boilerplate with no logic | `cavecrew-investigator`, `cavecrew-builder`, or `Agent(model: haiku)` |
| **sonnet** (default for light judgment) | writing tests for existing code; focused diff review; single-file edits that touch logic but have a clear, unambiguous spec; doc/markdown from a fixed outline | `cavecrew-builder`, `cavecrew-reviewer`, `code-simplifier`, or `Agent(model: sonnet)` |
| **opus / inline** (do NOT delegate) | architecture & design; multi-file refactors with coupling; debugging an unknown root cause; brainstorming/planning; any edit touching product behavior or cross-file coupling; the routing decision itself | keep on main thread |

**Hard rule:** if a change touches product behavior or coupling, stay on Opus —
even if it looks like a single-file edit. Tests and diff review default to
sonnet minimum, NEVER haiku.

## Decision Procedure

1. **Break-even gate** (above). Fail → inline, stop.
2. Bounded + low-judgment per taxonomy? No → inline.
3. Needs full conversation context a fresh subagent lacks? → inline.
4. Multiple cheap items? → batch into ONE subagent call.
5. Else → dispatch to the matching tier, then **summarize the result back into
   main context** before continuing.

## Guardrails (hard rules)

- **Max 2 subagent calls per coherent task.** More = do it inline.
- **Never delegate when the task description is < 100 tokens** — overhead can't pay off.
- **Always summarize a subagent's result back into context** — summaries are lossy.
- Don't fragment one coherent change across many subagents.
- Don't route work whose context lives only in this conversation.
- The main thread is the orchestrator; routing is never itself routed.

## Worked Examples

- "rename a var across one file" → inline (< 100-token task)
- "rename a symbol across 30 files" → haiku (gate passes)
- "scan 3k lines of prod logs for a 500" → haiku
- "find where a value is computed (2-grep job)" → inline; (large surface) → haiku
- "write tests for an existing engine" → sonnet (never haiku)
- "fix a known off-by-one in one file" → sonnet (touches logic)
- "debug an intermittent hang" → opus inline
- "review a 1-file diff" → sonnet
- "design a data model" / "split a god-object service" → opus inline
