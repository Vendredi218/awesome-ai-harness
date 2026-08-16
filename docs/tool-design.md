# Tool Design

*A working note. [中文版](tool-design.zh-CN.md) · [back to index](../README.md)*

---

## The one-sentence version

A tool definition is a prompt, a tool result is a prompt, and a tool error is the most important prompt you will write — because it is the one the model reads when it is already wrong.

## The mental model: you are designing an interface for a colleague who never sleeps and never asks

The [SWE-agent paper](https://arxiv.org/abs/2405.15793) coined **agent-computer interface** (ACI) for this, by analogy with HCI, and demonstrated something uncomfortable: redesigning the interface moved benchmark scores more than swapping the model. The same model, given a file editor that reported syntax errors on write instead of silently accepting broken code, got dramatically better.

That result is the whole case for taking tool design seriously. Capability you thought was missing from the model was actually missing from the interface.

Unlike a human colleague, the agent will not ask you what a parameter means, will not notice that the docs are stale, and will not develop intuition over time. Everything it knows about your tool, it knows from the schema and from what previous calls returned.

## Principles

### 1. Fewer, more capable tools

Every tool definition costs tokens on every turn, forever. More importantly, each one adds a chance of picking wrong. Three tools with clear, disjoint purposes beat twelve with overlapping ones.

The test: if you can't state in one sentence when to use tool A instead of tool B, the model can't either. Merge them or sharpen the boundary.

### 2. Design around workflows, not around your API surface

The wrong instinct is to wrap each endpoint you happen to have. The right instinct is to ask *what does the agent actually need to accomplish*, and build a tool for that.

If completing a common task requires four chained calls where the agent has to thread IDs between them, that's four chances to fail and a lot of intermediate junk in context. Ship one tool that does the workflow.

### 3. Return results for a reader with a token budget

A tool that returns 50,000 tokens of JSON has poisoned the well, even if the call succeeded. Ask what the model actually needs to decide the next step, and return that.

Concretely:
- Truncate, and **say that you truncated**, with how to get more. Silent truncation causes the model to reason confidently about data it didn't receive.
- Prefer identifiers and paths over inlined blobs — let the agent fetch the blob if it wants it.
- Natural language beats deeply nested JSON for anything the model needs to *reason about* rather than *parse*.
- Offer a response-detail parameter (`concise` / `full`) and default to concise.

### 4. Errors are instructions

This is where most tool design goes wrong. Compare:

```
Error: 400 Bad Request
```

```
Error: 'user_id' must be a UUID, got "alice". Look up the UUID
with search_users(name="alice") first.
```

The first ends the agent's productive turn — it will retry, guess, or give up. The second is a functioning instruction that leads directly to recovery. **Write error messages as if they are the next line of the system prompt**, because functionally they are.

Good error messages should say what was wrong, what was expected, and what to do next. Validation failures are the highest-value place to invest here.

### 5. Names and descriptions do real work

The model routes on them. Use unambiguous, namespaced names (`calendar_create_event`, not `create`). Spell out in the description **when not to use this tool** — negative guidance prevents more misfires than positive guidance.

Avoid parameters whose meaning depends on context the model doesn't have. If a field can hold two different kinds of identifier, it will receive the wrong one.

### 6. Make the dangerous path harder than the safe path

If a tool can destroy something, the interface should make destruction deliberate: require an explicit flag, require the target to have been read first, return a preview before committing. Do not rely on instructions in the system prompt to prevent it. **Structure beats instruction** — an agent under pressure follows the path of least resistance, so make that path the safe one.

### 7. Evaluate tools like you evaluate code

Build a set of realistic tasks, run the agent, and read the transcripts. Not the pass rate — the transcripts. You are looking for: which tool did it reach for first, where did it get confused, what did it retry, what did it call with malformed arguments. Every one of those is a design bug in the tool, not a model failure.

Then fix the tool and re-run. This loop is unreasonably effective and almost nobody does it.

## At scale: the token wall

Once you have dozens of tools — the normal situation with MCP servers connected — definitions alone can consume a large chunk of context before any work happens. Two mitigations are emerging:

- **Deferred / searchable tool schemas.** Present names only, load full schemas on demand.
- **Tools as code.** Expose tools as an API the agent writes code against, so filtering and composition happen in the execution environment instead of in context. See [Code Execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp).

The second is more powerful and less mature. It also changes the failure surface: now you're sandboxing code execution.

## A checklist

Before shipping a tool:

- [ ] One sentence says exactly when to use it, and when not to
- [ ] Name is unambiguous and namespaced
- [ ] No parameter can plausibly receive the wrong kind of value
- [ ] Typical result fits comfortably in a few hundred tokens
- [ ] Large results truncate loudly, with a documented way to get more
- [ ] Every error message names the fix
- [ ] Destructive actions require an explicit, deliberate step
- [ ] You have read at least ten real transcripts of an agent using it

## Sources

- [Writing Effective Tools for Agents](https://www.anthropic.com/engineering/writing-tools-for-agents) — Anthropic
- [SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering](https://arxiv.org/abs/2405.15793)
- [Advanced Tool Use](https://www.anthropic.com/engineering/advanced-tool-use) — Anthropic
- [Code Execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp) — Anthropic
- [The "Think" Tool](https://www.anthropic.com/engineering/claude-think-tool) — Anthropic
