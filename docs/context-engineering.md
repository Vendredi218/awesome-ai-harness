# Context Engineering

*A working note. [中文版](context-engineering.zh-CN.md) · [back to index](../README.md)*

---

## The one-sentence version

A model's context window is not storage — it is a **budget you spend every single turn**, and the job of a harness is to decide what earns a place in it.

## Why the framing matters

Prompt engineering assumed one shot: you write a good prompt, you get a good answer. Agents broke that assumption. An agent runs for fifty turns, and every turn re-sends the entire accumulated history — the system prompt, every tool definition, every file it read, every command's output, every mistake it made. Context is not written once. It is **rebuilt on every forward pass**.

That changes the engineering problem from *"what should I say"* to *"what should be present, at this moment, and what should be gone."*

## The four failure modes

Drew Breunig's taxonomy is the most useful vocabulary in the field. Learn to recognize these in your traces:

| Failure | What it looks like |
|---|---|
| **Poisoning** | A hallucination enters context and gets treated as fact forever after. The agent invented a function name on turn 3 and is still calling it on turn 30. |
| **Distraction** | Context grows so long the model over-attends to its own history instead of solving the task. Symptom: it keeps re-doing things it already did. |
| **Confusion** | Irrelevant material influences the answer. 60 tools loaded, only 3 relevant, and it picks a wrong one. |
| **Clash** | Two parts of the context contradict each other. An early plan says X, a later observation says not-X, and nothing resolved it. |

Note that three of these get **worse** as context gets longer. This is the core tension: more information is not more capability.

## Why "just use a 1M window" doesn't solve it

Three separate reasons, and they compound:

1. **Attention degrades with position.** [Lost in the Middle](https://arxiv.org/abs/2307.03172) showed retrieval accuracy sags for material in the middle of a long context. Burying the crucial line at token 400,000 is close to not including it.
2. **Cost and latency are linear-ish in tokens, every turn.** A 50-turn agent that carries 200k tokens pays for 200k tokens fifty times.
3. **The failure modes above are about signal-to-noise, not capacity.** Adding room to be distracted in does not reduce distraction.

The goal is the **smallest set of high-signal tokens that makes the next decision correct**. Not the largest set that fits.

## The techniques

Ordered roughly by how much leverage they give per unit of engineering effort.

### 1. Just-in-time retrieval (highest leverage)

Don't pre-load. Give the agent tools to fetch what it needs, when it needs it. A file path is ~10 tokens; the file is ~5,000. Let the agent hold the path and spend the 5,000 only when it decides the file matters.

This is what makes `grep` + `read` beat a pre-built vector index for most coding agents: the agent's *own judgment* about relevance, exercised at the moment of need, is better than a similarity score computed in advance. And it produces a legible trail of why each thing got loaded.

### 2. Compaction

When the window fills, summarize the history and continue with the summary. The engineering question is not *whether* to compact but **what must survive compaction**: open decisions, unresolved errors, and the current goal. What can be dropped: full file contents (re-readable), verbose tool output, exploration that led nowhere.

A good compaction preserves the *state of the work*. A bad one preserves a narrative of what happened.

### 3. Offloading to the file system

The most reliable memory is not in the context window at all. Have the agent write notes, plans, and intermediate results to files, and read them back on demand. This survives compaction, survives session boundaries, and is inspectable by a human.

`TODO.md` written by the agent, re-read each turn, is a shockingly effective planning mechanism for its complexity.

### 4. Sub-agent isolation

Give a bounded task to a fresh agent with its own clean window, and return only its conclusion. A search that reads 40 files returns one paragraph. The 40 files never touch the parent's context.

The trade-off, per [Cognition](https://cognition.ai/blog/dont-build-multi-agents), is real: the sub-agent lacks the parent's context and may make decisions the parent wouldn't. Use it for tasks where **the output is much smaller than the input** and the task is genuinely self-contained. Search and review fit. Implementation usually doesn't.

### 5. Tool loadout management

Don't ship 80 tool definitions when 6 are relevant. Load tool schemas on demand, or select a subset per task. This attacks *confusion* directly, and tool definitions are pure overhead on every turn until used.

### 6. Structured note-taking

Have the agent maintain an explicit running summary in a fixed format — goal, decisions made, open questions, next step. Cheaper than re-deriving the state from history, and it makes clash visible: contradictions show up in a two-line summary in a way they never do across 40k tokens of transcript.

## Practical heuristics

- **Put stable content first.** System prompt and tool definitions at the front, volatile turn-by-turn material at the back. This is what makes prompt caching work, and caching is the single largest cost lever in an agent harness.
- **Return errors, not exceptions.** A tool that fails should return a message the model can act on. This is context engineering — the error message *is* the next turn's instruction.
- **Watch for re-reads.** If your agent reads the same file three times, your context management dropped something it needed. That's a diagnostic signal, not a quirk.
- **Budget explicitly.** Know roughly what fraction of your window goes to system prompt vs. tools vs. history. If tools are 30% and mostly unused, you found your problem.
- **Cite tokens in reviews.** When evaluating a harness change, "it works" is not enough — did it work with fewer tokens? That's the axis harness engineering competes on.

## Open questions

Honest about what isn't settled:

- **When is compaction better than sub-agents?** Both compress. Compaction keeps one coherent thread and loses detail; sub-agents keep detail locally and lose coherence. There's no good rule yet for which to reach for.
- **How do you evaluate context management in isolation?** End-to-end task success confounds it with everything else. Nobody has a clean benchmark for "did the harness keep the right things."
- **Does any of this survive better models?** Some of it is scaffolding around current weaknesses. Some — the cost argument, the signal-to-noise argument — looks structural. Unclear which is which.

## Sources

- [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Anthropic
- [How Long Contexts Fail](https://www.dbreunig.com/2025/06/22/how-contexts-fail-and-how-to-fix-them.html) and [How to Fix Your Context](https://www.dbreunig.com/2025/06/26/how-to-fix-your-context.html) — Drew Breunig
- [Context Engineering for Agents](https://blog.langchain.com/context-engineering-for-agents/) — LangChain
- [Lost in the Middle](https://arxiv.org/abs/2307.03172)
- [Don't Build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents) — Cognition
