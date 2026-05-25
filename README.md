# model-router

An always-on cost-routing skill for [Claude Code](https://docs.claude.com/en/docs/claude-code). It pushes cheap, mechanical work down to lower-cost models and keeps the expensive model for the work that actually needs it.

## The honest version of what this does

A Claude Code "skill" is a markdown file loaded into the model's context. **A skill cannot switch the model the harness is running** — that's set per request by Claude Code, not something the model can reach up and change mid-session.

So model-router doesn't pretend to. Instead it routes *work*: the main thread (your top-tier model) stays the orchestrator and delegates bounded, low-judgment tasks to cheaper-model subagents (`haiku`, `sonnet`) via the `Agent` tool's `model` parameter. You capture most of the cost savings without the magic trick nobody can actually build.

## Why it's not just "always use the cheap model"

Delegation isn't free. Each hand-off costs 300–800+ tokens (the delegation prompt, subagent isolation, the summarized result, coordination). Delegate a small task and the overhead eats the savings.

So the skill gates on a **break-even rule**: only delegate when the mechanical work is estimated above ~1.5k tokens *and* coordination cost is under 30% of the expected savings. Below that, it does the work inline. The result is real savings on long, read- and edit-heavy sessions, and near-zero overhead on short ones.

## Routing taxonomy

| Tier | Use for |
|------|---------|
| **haiku** (cheapest) | code/symbol location, log scans, mechanical renames, pure format edits, extracting values, no-logic boilerplate |
| **sonnet** (light judgment) | writing tests for existing code, focused diff review, single-file edits with a clear spec, docs from an outline |
| **opus / inline** (don't delegate) | architecture & design, multi-file refactors with coupling, unknown-root-cause debugging, anything touching product behavior — and the routing decision itself |

**Hard rule:** if a change touches product behavior or cross-file coupling, it stays on the top model — even if it looks like a one-file edit. Tests and diff review default to sonnet minimum, never haiku.

## Guardrails

- Max 2 subagent calls per coherent task — more than that and you do it inline.
- Never delegate a task whose description is under ~100 tokens; the hand-off can't pay off.
- Always summarize a subagent's result back into the main context — subagent summaries are lossy.
- The main thread is the orchestrator; routing is never itself routed.

## Two outlets, one gate

When work clears the break-even gate, there are two ways to run it cheaper — the skill picks based on shape:

- **Delegate to a subagent** — for a discrete chunk you can offload while you keep working on the main thread. Autonomous; runs on the cheaper model in its own context.
- **Prompt-to-switch the main thread** — for a *sustained* stretch of cheap work, where the cheapest place to run it is the main conversation itself, on a cheaper model. A skill can't run `/model` (only the user can switch the live session model), so it surfaces the exact command — e.g. `/model haiku` — and prompts you once, then prompts again to switch back up when judgment work resumes.

Same gate, never both for the same work. Below the gate: stay inline, no prompt, no subagent.

(No recursion: subagents don't route. In Claude Code a subagent isn't given the dispatch tool, so the orchestration tree stays one level deep — a subagent that uncovers a big new job reports up and the main thread decides. Flat fan-out beats deep nesting.)

## Install

### 1. Drop in the skill

```bash
mkdir -p ~/.claude/skills
cp -r skills/model-router ~/.claude/skills/model-router
```

The skill is now invocable in any Claude Code session.

### 2. (Optional) Make it always-on

Add a tiny `SessionStart` hook so the policy is nudged into context at the start of every session. Edit `~/.claude/settings.json` and add this object to `hooks.SessionStart` (create the array if it doesn't exist):

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "cat <<'EOF'\n{\"hookSpecificOutput\":{\"hookEventName\":\"SessionStart\",\"additionalContext\":\"Cost discipline (model-router skill): for mechanical work est. over 1.5k tokens - code/symbol location, log scans, renames, boilerplate - delegate to haiku or sonnet subagents instead of doing it inline. Below ~1.5k tokens, task description under 100 tokens, or any judgment/behavior/coupling-touching work, stay inline on the top model. Full policy, break-even gate, and guardrails: invoke the model-router skill.\"}}\nEOF"
          }
        ]
      }
    ]
  }
}
```

The injected pointer is deliberately tiny (~3 lines) — a cost-saver that taxes every session would defeat its own purpose. The full policy stays in the skill and only loads when invoked.

## Scope notes

- Installed under `~/.claude`, it applies to **every project** you work in.
- It lives on the machine where `~/.claude` is — a remote/Cloud session won't carry it.

## How it was built

Designed and shipped in a single Claude Code session: brainstorm → design spec → adversarial review (run through Grok across three rounds) → implementation plan → build → verify. I'll be honest about the division of labor — Claude wrote the files. I owned the problem framing, the architecture decisions (work-routing vs. the impossible model-switch, the break-even gate, the guardrails), and the quality bar.

## License

MIT — see [LICENSE](LICENSE).
