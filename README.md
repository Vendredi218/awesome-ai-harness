# Awesome AI Harness [![Awesome](https://awesome.re/badge.svg)](https://awesome.re)

> The model is the engine. The **harness** is the car.

A curated list of resources on **harness engineering** — the discipline of building the runtime scaffolding that turns a language model into a working agent.

[中文版 →](README.zh-CN.md)

---

## Why this list

Most "awesome agent" lists catalogue *frameworks*. This one is about the layer underneath: the concrete engineering decisions that determine whether an agent works or flails.

A frontier model dropped into a bad harness behaves like a bad model. It loops, forgets, calls the wrong tool, burns its context on noise, and gives up. The same model in a good harness plans, recovers from errors, and finishes. **The gap between those two outcomes is harness engineering**, and almost none of it is in the model weights.

The craft is young, the knowledge is scattered across engineering blogs and leaked system prompts, and there is no textbook. This is an attempt at a map.

## What is a harness?

Everything between the user's intent and the model's next token:

```mermaid
flowchart TB
    U([User intent]) --> H
    subgraph H[The Harness]
        direction TB
        SP[System prompt<br/>role, rules, environment]
        CTX[Context assembly<br/>budget · retrieval · compaction]
        TOOL[Tool layer<br/>definitions · results · errors]
        LOOP[Agent loop<br/>plan → act → observe → repeat]
        MEM[Memory &amp; state<br/>files · sessions · scratchpads]
        GATE[Permissions &amp; sandbox<br/>allowlists · HITL gates]
    end
    H <--> M{{Model}}
    H --> W([Effects on the world])
```

The model is a stateless function. Every bit of statefulness, safety, memory, and competence an "agent" appears to have was engineered into the loop around it.

---

## Contents

- [Foundations](#foundations)
- [Context Engineering](#context-engineering)
- [Tool Design](#tool-design)
- [The Agent Loop](#the-agent-loop)
- [Memory & State](#memory--state)
- [Multi-Agent & Orchestration](#multi-agent--orchestration)
- [Permissions, Sandboxing & Safety](#permissions-sandboxing--safety)
- [Skills & Progressive Disclosure](#skills--progressive-disclosure)
- [Evaluation & Observability](#evaluation--observability)
- [Reference Harnesses](#reference-harnesses)
- [Building Your Own](#building-your-own)
- [Papers](#papers)
- [Deep Dives in This Repo](#deep-dives-in-this-repo)

---

## Foundations

Start here if you are new to the field.

- [Harness Engineering for Self-Improvement](https://lilianweng.github.io/posts/2026-07-04-harness/) — Lilian Weng. The most complete map of the discipline to date: three design patterns (workflow automation, filesystem-as-memory, sub-agents and backend jobs), a coding-agent case study, where the harness layer ends and core intelligence begins, and the case for harnesses that optimize themselves through evolutionary search. If you read one thing on this list, read this.
- [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) — Anthropic. The canonical starting point. Argues most "agents" should be workflows, and defines the vocabulary (chaining, routing, orchestrator-workers, evaluator-optimizer) the rest of the field now uses.
- [How to Build an Agent](https://ampcode.com/how-to-build-an-agent) — Thorsten Ball. A working code agent in ~300 lines. The single best cure for the belief that agents are magic; it is a loop, a list of tools, and good plumbing.
- [Agents](https://huyenchip.com/2025/01/07/agents.html) — Chip Huyen. Systematic breakdown of planning, tool use, and failure modes.
- [Don't Build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents) — Cognition. The dissenting case: context fragmentation across agents is the dominant failure mode. Read alongside the multi-agent section below.
- [A Practical Guide to Building Agents](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf) — OpenAI (PDF).

## Context Engineering

The context window is the agent's entire working memory, and it is a scarce, actively-managed resource. This is where most harness quality lives.

- [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Anthropic. Treats context as a finite budget; covers just-in-time retrieval, compaction, and structured note-taking.
- [Context Engineering](https://simonwillison.net/2025/Jun/27/context-engineering/) — Simon Willison. Short piece that popularized the term over "prompt engineering."
- [How Long Contexts Fail](https://www.dbreunig.com/2025/06/22/how-contexts-fail-and-how-to-fix-them.html) — Drew Breunig. Names four failure modes: poisoning, distraction, confusion, clash. Essential vocabulary.
- [How to Fix Your Context](https://www.dbreunig.com/2025/06/26/how-to-fix-your-context.html) — Drew Breunig. The companion remedies: RAG, tool loadout, quarantine, pruning, summarization, offloading.
- [Context Engineering for Agents](https://blog.langchain.com/context-engineering-for-agents/) — LangChain. The write/select/compress/isolate taxonomy.
- [The Rise of Context Engineering](https://blog.langchain.com/the-rise-of-context-engineering/) — LangChain.
- [Context Engineering — What it is, and techniques to consider](https://www.philschmid.de/context-engineering) — Philipp Schmid.
- [Managing Context on the Claude Developer Platform](https://www.anthropic.com/news/context-management) — Anthropic. Context editing and the memory tool as first-class API primitives.

## Tool Design

Tools are the agent's API to reality. Their names, schemas, return values, and error messages are prompt engineering by another name.

- [Writing Effective Tools for Agents](https://www.anthropic.com/engineering/writing-tools-for-agents) — Anthropic. The best single reference: consolidate tools, return token-efficient results, make errors instructive, namespace clearly.
- [Advanced Tool Use](https://www.anthropic.com/engineering/advanced-tool-use) — Anthropic. Programmatic tool calling, tool search, and parallel execution.
- [Code Execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp) — Anthropic. Presenting tools as code APIs instead of direct calls, cutting token overhead dramatically at scale.
- [The "Think" Tool](https://www.anthropic.com/engineering/claude-think-tool) — Anthropic. A no-op tool that buys deliberation mid-loop. Elegant demonstration that tools shape cognition, not just capability.
- [Model Context Protocol](https://modelcontextprotocol.io) — The emerging standard for wiring tools and data into harnesses.
- [MCP Reference Servers](https://github.com/modelcontextprotocol/servers) — Read these as worked examples of tool schema design.

## The Agent Loop

- [ReAct: Synergizing Reasoning and Acting](https://arxiv.org/abs/2210.03629) — The loop nearly every harness still runs.
- [Executable Code Actions Elicit Better LLM Agents](https://arxiv.org/abs/2402.01030) — CodeAct. Actions as Python instead of JSON; composition and control flow come free.
- [Reflexion](https://arxiv.org/abs/2303.11366) — Verbal self-reflection as a substitute for weight updates.
- [Self-Refine](https://arxiv.org/abs/2303.17651) — Iterative self-critique.
- [Claude Code Best Practices](https://www.anthropic.com/engineering/claude-code-best-practices) — Anthropic. Explore→plan→code→commit, and why an agent that reads before writing outperforms one that doesn't.

## Memory & State

- [MemGPT: Towards LLMs as Operating Systems](https://arxiv.org/abs/2310.08560) — Virtual-memory paging applied to the context window. The paper that made "memory hierarchy" the default frame.
- [Generative Agents](https://arxiv.org/abs/2304.03442) — Memory stream with recency/importance/relevance retrieval.
- [AGENT.md](https://ampcode.com/AGENT.md) — The convention for per-repo agent instruction files.
- [agents.md](https://agents.md/) — The cross-vendor push to standardize that file.
- [Letta](https://github.com/letta-ai/letta) — The MemGPT ideas as a working stateful-agent server.

## Multi-Agent & Orchestration

- [How We Built Our Multi-Agent Research System](https://www.anthropic.com/engineering/multi-agent-research-system) — Anthropic. Honest about the costs: token overhead, coordination failure, and where fan-out actually pays.
- [Don't Build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents) — Cognition. The strongest counterargument. Read both.
- [LangGraph](https://github.com/langchain-ai/langgraph) — Graph-structured orchestration with explicit state.
- [OpenAI Agents SDK](https://github.com/openai/openai-agents-python) — Handoffs, guardrails, and tracing as primitives.
- [Pydantic AI](https://github.com/pydantic/pydantic-ai) — Type-safe agents; strong schema discipline.

## Permissions, Sandboxing & Safety

If your harness can run commands, it can run *bad* commands. This section is not optional.

- [The Lethal Trifecta](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/) — Simon Willison. Private data + untrusted content + external communication = exfiltration. The clearest threat model in the field.
- [Prompt Injection series](https://simonwillison.net/series/prompt-injection/) — Simon Willison. Years of accumulated attacks and why filtering does not work.
- [Design Patterns for Securing LLM Agents against Prompt Injections](https://arxiv.org/abs/2506.08837) — Six concrete architectural patterns with real trade-offs.
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/) — The checklist to review your harness against.
- [E2B](https://github.com/e2b-dev/E2B) — Sandboxed cloud runtimes for agent-generated code.
- [Claude Code Security Review](https://github.com/anthropics/claude-code-security-review) — Agentic security review as a GitHub Action.

## Skills & Progressive Disclosure

Loading every instruction upfront wastes context. Loading them on demand is a harness pattern in its own right.

- [Equipping Agents for the Real World with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) — Anthropic. The design rationale for progressive disclosure.
- [Agent Skills documentation](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) — Anthropic. The format spec: frontmatter, bundled scripts, and what triggers loading.
- [anthropics/skills](https://github.com/anthropics/skills) — Reference skill implementations.
- [Superpowers](https://github.com/obra/superpowers) — Jesse Vincent. An opinionated skill library that treats process discipline itself as an installable capability.

## Evaluation & Observability

You cannot tune a harness you cannot measure. Trajectory-level evaluation is a distinct problem from model benchmarking.

- [Your AI Product Needs Evals](https://hamel.dev/blog/posts/evals/) — Hamel Husain. The best practical writing on building an eval loop that survives contact with production.
- [Creating a LLM-as-a-Judge That Drives Business Results](https://hamel.dev/blog/posts/llm-judge/) — Hamel Husain.
- [SWE-bench](https://arxiv.org/abs/2310.06770) — The benchmark that made harness quality legible; scores move on scaffolding alone.
- [SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering](https://arxiv.org/abs/2405.15793) — Coins "agent-computer interface" and shows interface design is worth more than model swaps. Arguably the foundational harness-engineering paper.
- [τ-bench](https://arxiv.org/abs/2406.12045) — Tool-agent-user interaction with real domain rules.
- [Langfuse](https://github.com/langfuse/langfuse) — Open-source LLM tracing and evals.
- [Arize Phoenix](https://github.com/Arize-ai/phoenix) — Tracing and evaluation for agent traces.

## Reference Harnesses

Read the source. This is the fastest way to learn the craft.

- [Claude Code](https://github.com/anthropics/claude-code) — Anthropic's coding agent.
- [Codex CLI](https://github.com/openai/codex) — OpenAI's, in Rust.
- [DeepSeek Harness (dsh)](https://github.com/deepseek-ai/deepseek-harness) — Everything is a plugin, including the agent loop itself. There is deliberately no privileged core; the shipped coding agent is just one composition of swappable parts.
- [Pi](https://github.com/earendil-works/pi) — The minimalist counterpoint: a thin core of four tools (`read`, `write`, `edit`, `bash`) on the bet that frontier models already know what a coding agent is. Everything else lives in extensions you add deliberately.
- [Gemini CLI](https://github.com/google-gemini/gemini-cli) — Google's, Apache-2.0.
- [Aider](https://github.com/Aider-AI/aider) — Mature, with unusually rigorous repo-map and edit-format engineering.
- [OpenHands](https://github.com/All-Hands-AI/OpenHands) — Research-grade, fully open, strong sandboxing story.
- [SWE-agent](https://github.com/SWE-agent/SWE-agent) — The ACI paper as running code.
- [Cline](https://github.com/cline/cline) — VS Code agent with an explicit plan/act split.
- [OpenCode](https://github.com/sst/opencode) — Terminal agent, provider-agnostic.
- [Goose](https://github.com/block/goose) — Block's extensible on-machine agent.
- [Continue](https://github.com/continuedev/continue) — Building blocks for custom in-IDE assistants.

Comparative reading:

- [DeepSeek Harness: Everything Is a Plugin](https://saikodev.ai/blog/deepseek-harness) — SaikoDev. A hands-on dsh walkthrough that turns into the sharpest architectural comparison available: dsh versus Pi as two opposite bets — structured addition against subtraction, and whether control over extending the system belongs to the developer ahead of time or to the agent at runtime.

## Building Your Own

- [Claude Agent SDK (Python)](https://github.com/anthropics/claude-agent-sdk-python) — The Claude Code harness as a library.
- [Claude Agent SDK (TypeScript)](https://github.com/anthropics/claude-agent-sdk-typescript)
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Runnable patterns.
- [OpenAI Cookbook](https://github.com/openai/openai-cookbook)
- [LiteLLM](https://github.com/BerriAI/litellm) — One interface across providers; useful for harness-level A/B tests.
- [HumanLayer](https://github.com/humanlayer/humanlayer) — Human-in-the-loop approval as infrastructure.
- [System Prompts and Models of AI Tools](https://github.com/x1xhlol/system-prompts-and-models-of-ai-tools) — Collected system prompts from shipping products. Primary sources, unmatched for studying real harness design.
- [Decoding Claude Code](https://minusx.ai/blog/decoding-claude-code/) — MinusX. Close reading of a production harness.

## Papers

Beyond those cited above:

- [Chain-of-Thought Prompting](https://arxiv.org/abs/2201.11903)
- [Toolformer](https://arxiv.org/abs/2302.04761)
- [Tree of Thoughts](https://arxiv.org/abs/2305.10601)
- [Voyager](https://arxiv.org/abs/2305.16291) — Skill libraries that compose over time.
- [Lost in the Middle](https://arxiv.org/abs/2307.03172) — Why position in context matters, and why "just use a long context" fails.
- [AgentBench](https://arxiv.org/abs/2308.03688)
- [WebArena](https://arxiv.org/abs/2307.13854)

---

## Deep Dives in This Repo

Longer notes that synthesize the above rather than just linking it:

- [Context Engineering](docs/context-engineering.md) · [中文](docs/context-engineering.zh-CN.md)
- [Tool Design](docs/tool-design.md) · [中文](docs/tool-design.zh-CN.md)

**Planned:** the agent loop · memory & state · multi-agent orchestration · permissions & sandboxing · evaluation. [Contributions welcome.](CONTRIBUTING.md)

---

## Contributing

Read [CONTRIBUTING.md](CONTRIBUTING.md). The bar: it must teach something about **building the scaffolding**, not about prompting a chatbot or training a model. Every entry needs a sentence saying what you actually learn from it.

## License

[CC0 1.0](LICENSE) — public domain.
