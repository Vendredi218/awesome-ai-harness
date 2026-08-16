# Contributing to Open-Source Harnesses

*A working note. [中文版](contributing-to-open-harnesses.zh-CN.md) · [back to index](../README.md)*

---

## The one-sentence version

The fastest way to learn harness engineering is to extend a harness someone else runs in production — and the right way to do that depends entirely on whether the project's core is open or its ecosystem is.

## Why contribute at all

Reading about context budgets is one thing. Writing a Pi extension that hooks `session_before_compact` and watching your code decide what survives compaction is another. One real plugin forces you through session state, context budgeting, tool dispatch, and provider abstraction — four of the six runtime responsibilities that [the harness-design survey](https://arxiv.org/abs/2606.20683) says every harness must answer (permissions and recovery round out the list). There is no faster curriculum.

There's also a selfish career argument: harness engineering has almost no credentialing yet. A merged benchmark task or a published plugin *is* the credential.

## First, read the governance map

Harness projects split into two clusters, and mistaking one for the other wastes months:

| | Core-open | Ecosystem-first |
|---|---|---|
| **Projects** | OpenHands, Cline, Continue, Gemini CLI, Goose, Aider | Claude Code, DeepSeek Harness (dsh), Codex CLI |
| **Engine PRs** | Welcome, with normal OSS friction | Closed core, or invited-only |
| **Intended surface** | The code itself | Plugins, skills, extensions, registries |
| **What "good contribution" means** | A merged PR | A published extension, a validated skill, a root-cause analysis |

The ecosystem-first posture is not hostility — it's maintainer-attention economics. Codex's contributing doc says it plainly: review cost exceeded implementation cost, so what they want from outsiders is *diagnosis*, not diffs. A reproducible root-cause analysis in an issue thread is a first-class contribution there.

Before picking a target, check which game it plays. Then pick your pathway.

## The six pathways, ranked by learning density

### 1. Programmatic plugins on deep-API harnesses — learn the loop itself

The highest learning-per-hour route. Three harnesses expose deep enough APIs to teach real internals:

- **Pi** ([extensions.md](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/extensions.md)) — TypeScript extensions against an explicit event surface: `session_before_compact`, `before_provider_request`, tool interception. Small enough to read entirely; the events map one-to-one onto harness concepts.
- **dsh** ([Cordis tutorial](https://deepseek-harness.github.io/deepseek-harness/en/develop/cordis-tutorial/)) — the deepest plugin API in the field: dependency injection, typed events, hot module reload, and *everything* is replaceable, including the agent loop. Publish by tagging your repo `dsh-plugin` — the registry is decentralized.
- **Cline** ([plugins](https://docs.cline.bot/sdk/plugins)) — the `AgentPlugin` design with hook policies (blocking vs. async, timeouts, `failureMode`) is the best available study text for the question "how do you safely run third-party code inside an agent loop?"

What it teaches: the loop, message assembly, tool dispatch, lifecycle. What it doesn't: nothing — this is the full curriculum. Its only cost is that your work is tied to one harness.

### 2. Contribute an eval task — the most underrated route

Terminal-bench accumulated 229 tasks from 93 contributors; benchmark task authorship is now a recognized, reviewed contribution with your name on it. The [Harbor task format](https://harborframework.com/docs/task-format) is the craft in miniature — every task is four parts:

1. an instruction,
2. a Dockerized environment,
3. a verifier that emits `reward.json`,
4. an oracle `solve.sh` proving the task is solvable.

Writing the *verifier* is where the learning is: you will discover exactly how agents cheat, shortcut, and technically-satisfy. [cline-bench](https://cline.ghost.io/cline-bench-initiative/) takes an even more direct route — contribute the real task that broke your agent.

What it teaches: verification design, agent failure modes, environment reproducibility. Nothing teaches you what breaks harnesses faster than writing a task designed to break them.

### 3. Ship an MCP server — maximum portability

One artifact works in every major harness. The [registry](https://github.com/modelcontextprotocol/registry) gives it discovery; [Goose's extension docs](https://goose-docs.ai/docs/tutorials/custom-extensions) demonstrate the punchline — a Goose extension *is* an MCP server, no adaptation needed.

What it teaches: tool-surface ergonomics, schema design, token-cost tradeoffs — the entire [tool design](tool-design.md) doc, by doing. What it doesn't: you stay *outside* the loop. You'll learn nothing about context management or the agent lifecycle from inside an MCP server. Know that going in.

### 4. Skills, recipes, and blocks — the lowest-friction real PR

The declarative tier: no code, just well-structured instructions.

- **OpenHands** ([extensions registry](https://github.com/OpenHands/extensions)): a `SKILL.md` directory plus one JSON entry — plausibly the lowest-friction merged PR in the field.
- **Claude Code** ([plugin docs](https://code.claude.com/docs/en/plugins)): `claude plugin validate` → community marketplace review → SHA-pinned catalog. The pipeline itself is worth studying as supply-chain design.
- **Continue** ([blocks](https://docs.continue.dev/customize/overview)), **Goose** (recipes), and the cross-harness [skills.sh](https://www.skills.sh/) directory.

What it teaches: progressive disclosure — writing instructions a model loads on demand — and the discipline of writing for a reader with no context. A skill is a tiny system prompt; writing a good one is prompt engineering with a merge gate.

### 5. Core PRs — for the core-open cluster only

The classic route still works where it's welcome: OpenHands, Cline, Continue, Gemini CLI, Goose, and Aider all run good-first-issue labels. Highest prestige, slowest feedback loop, most subject to maintainer bandwidth. Start with issues where you've already done pathway 6.

### 6. Root-cause analysis — the closed-core contribution

On ecosystem-first projects, a reproducible diagnosis is the contribution: minimal repro, trajectory excerpt, hypothesis, suggested fix. This is trajectory debugging — arguably *the* core skill of harness engineering — practiced on production systems you didn't build. Read [Codex's policy](https://github.com/openai/codex/blob/main/docs/contributing.md) for the explicit version.

## The boring parts that bite

- **CLAs and DCOs.** Google (Gemini CLI, ADK), AWS (Strands), and ByteDance (Trae) require contributor license agreements; many others require DCO sign-off (`git commit -s`). Check before you write code, not after.
- **License asymmetries.** Crush is FSL-1.1-MIT (source-available, converts to MIT after two years); Skyvern is AGPL with cloud-only features. Both are fine places to *learn* — but understand what your plugin can legally do and where your contribution ends up before investing.
- **Security etiquette.** Scan third-party MCP servers and skills before wiring them into anything ([mcp-scan](https://github.com/invariantlabs-ai/mcp-scan)); never publish a skill that embeds credentials or fetches remote instructions at runtime.
- **Money is entering the loop.** Goose has paid recipes, Cline has Builder Credits. Ecosystem contribution is starting to pay — which also means marketplaces will get spammier and review bars higher.

## An honest caveat

This guide maps the *doorways* — it is compiled from the projects' own contributor docs, current as of August 2026. It does not yet include acceptance-rate or time-to-merge data, and doc'd policy is not always lived policy. Before committing serious effort to any one pathway, skim that project's recently merged PRs and see what actually gets accepted. (If you do this systematically, that analysis would itself make a great contribution to this repo.)

## 中文生态

dsh 的去中心化注册模式（给仓库打 `dsh-plugin` topic 即完成发布）让中文开发者可以完全绕过英文 PR 流程参与生态；[awesome-deepseek-harness](https://github.com/libukai/awesome-deepseek-harness) 是这个生态的地图。AgentScope（阿里）和 Youtu-Agent（腾讯）的 issue 区都活跃接受中文交流。用中文写 harness engineering 的一手实践笔记，本身就是这个领域稀缺的贡献。

## Sources

- [From Question Answering to Task Completion: A Survey on Agent System and Harness Design](https://arxiv.org/abs/2606.20683)
- Contributor docs of Claude Code, Gemini CLI, Codex, dsh, Pi, Cline, OpenHands, Goose, Continue (linked inline above)
- [Harbor task format](https://harborframework.com/docs/task-format) · [cline-bench](https://cline.ghost.io/cline-bench-initiative/)
- [DeepSeek Harness: Everything Is a Plugin](https://saikodev.ai/blog/deepseek-harness) — SaikoDev, for the dsh/Pi governance contrast
