# Awesome AI Harness [![Awesome](https://awesome.re/badge.svg)](https://awesome.re)

> 模型是引擎，**harness** 是整辆车。

关于 **harness engineering**（智能体运行时脚手架工程）的资源精选 —— 即把一个语言模型变成一个能干活的 agent 所需要的全部工程。

[English →](README.md)

---

## 为什么做这个列表

大部分 "awesome agent" 列表收集的是 *framework*。这个列表关注的是更底层的一层：那些真正决定 agent 是能干活还是原地打转的具体工程决策。

把一个前沿模型放进糟糕的 harness，它的表现就像一个糟糕的模型：死循环、遗忘、调错工具、把 context 烧在噪音上，然后放弃。同一个模型放进好的 harness，它会规划、会从错误中恢复、会把事情做完。**这两种结果之间的差距就是 harness engineering**，而它几乎完全不在模型权重里。

这门手艺还很年轻，知识散落在各家的工程博客和泄露的 system prompt 里，没有教科书。这个仓库试着画一张地图。

## 什么是 harness？

用户意图和模型下一个 token 之间的一切：

```mermaid
flowchart TB
    U([用户意图]) --> H
    subgraph H[Harness]
        direction TB
        SP[System prompt<br/>角色 · 规则 · 环境]
        CTX[Context 组装<br/>预算 · 检索 · 压缩]
        TOOL[工具层<br/>定义 · 返回值 · 错误]
        LOOP[Agent loop<br/>plan → act → observe → 循环]
        MEM[记忆与状态<br/>文件 · session · 草稿区]
        GATE[权限与沙箱<br/>allowlist · 人工确认]
    end
    H <--> M{{模型}}
    H --> W([对世界产生的影响])
```

模型是一个无状态函数。一个 "agent" 看起来具备的所有状态、安全性、记忆和能力，都是被工程化进它外面那个循环里的。

---

## 目录

- [基础入门](#基础入门)
- [Context Engineering](#context-engineering)
- [工具设计](#工具设计)
- [Agent Loop](#agent-loop)
- [记忆与状态](#记忆与状态)
- [多 Agent 与编排](#多-agent-与编排)
- [权限、沙箱与安全](#权限沙箱与安全)
- [Skills 与渐进式披露](#skills-与渐进式披露)
- [评估与可观测性](#评估与可观测性)
- [可参考的 Harness 实现](#可参考的-harness-实现)
- [自己动手做一个](#自己动手做一个)
- [论文](#论文)
- [本仓库的深度笔记](#本仓库的深度笔记)

---

## 基础入门

刚入门从这里开始。

- [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) —— Anthropic。公认的起点。核心观点是大部分所谓 "agent" 其实应该是 workflow，并定义了这个领域现在通用的词汇表（chaining、routing、orchestrator-workers、evaluator-optimizer）。
- [How to Build an Agent](https://ampcode.com/how-to-build-an-agent) —— Thorsten Ball。用大约 300 行代码写一个能用的 coding agent。治疗"agent 很玄"这种幻觉的最好药方：它就是一个循环、一组工具，加上扎实的工程。
- [Agents](https://huyenchip.com/2025/01/07/agents.html) —— Chip Huyen。系统性地拆解 planning、tool use 和失败模式。
- [Don't Build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents) —— Cognition。反方观点：跨 agent 的 context 割裂是最主要的失败来源。建议和下面的多 agent 章节对照着读。
- [A Practical Guide to Building Agents](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf) —— OpenAI（PDF）。

## Context Engineering

Context window 就是 agent 的全部工作记忆，而且是一种稀缺的、需要主动管理的资源。harness 的质量大部分体现在这里。

- [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) —— Anthropic。把 context 当作有限预算来管理；覆盖 just-in-time 检索、compaction 和结构化笔记。
- [Context Engineering](https://simonwillison.net/2025/Jun/27/context-engineering/) —— Simon Willison。让这个词取代 "prompt engineering" 的那篇短文。
- [How Long Contexts Fail](https://www.dbreunig.com/2025/06/22/how-contexts-fail-and-how-to-fix-them.html) —— Drew Breunig。命名了四种失败模式：poisoning、distraction、confusion、clash。非常有用的词汇表。
- [How to Fix Your Context](https://www.dbreunig.com/2025/06/26/how-to-fix-your-context.html) —— Drew Breunig。上一篇的解药：RAG、tool loadout、quarantine、pruning、summarization、offloading。
- [Context Engineering for Agents](https://blog.langchain.com/context-engineering-for-agents/) —— LangChain。write / select / compress / isolate 四分法。
- [The Rise of Context Engineering](https://blog.langchain.com/the-rise-of-context-engineering/) —— LangChain。
- [Context Engineering — What it is, and techniques to consider](https://www.philschmid.de/context-engineering) —— Philipp Schmid。
- [Managing Context on the Claude Developer Platform](https://www.anthropic.com/news/context-management) —— Anthropic。把 context editing 和 memory tool 做成 API 一等公民。

## 工具设计

工具是 agent 通向现实世界的 API。它的命名、schema、返回值和错误信息，本质上都是另一种形式的 prompt engineering。

- [Writing Effective Tools for Agents](https://www.anthropic.com/engineering/writing-tools-for-agents) —— Anthropic。这个主题最好的单篇参考：合并工具、返回 token 高效的结果、让错误信息本身具有指导性、命名空间要清晰。
- [Advanced Tool Use](https://www.anthropic.com/engineering/advanced-tool-use) —— Anthropic。programmatic tool calling、tool search 和并行执行。
- [Code Execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp) —— Anthropic。把工具暴露成代码 API 而不是直接调用，在工具数量大时能极大降低 token 开销。
- [The "Think" Tool](https://www.anthropic.com/engineering/claude-think-tool) —— Anthropic。一个什么也不做的工具，用来在循环中间买到"思考时间"。优雅地证明了：工具塑造的是认知方式，不只是能力边界。
- [Model Context Protocol](https://modelcontextprotocol.io) —— 正在成为标准的工具/数据接入协议。
- [MCP 官方 Server 实现](https://github.com/modelcontextprotocol/servers) —— 当作 tool schema 设计的范例来读。

## Agent Loop

- [ReAct: Synergizing Reasoning and Acting](https://arxiv.org/abs/2210.03629) —— 今天几乎所有 harness 仍然在跑的那个循环。
- [Executable Code Actions Elicit Better LLM Agents](https://arxiv.org/abs/2402.01030) —— CodeAct。用 Python 而不是 JSON 表达动作，组合和控制流就都免费得到了。
- [Reflexion](https://arxiv.org/abs/2303.11366) —— 用语言层面的自我反思代替权重更新。
- [Self-Refine](https://arxiv.org/abs/2303.17651) —— 迭代式自我批评。
- [Claude Code Best Practices](https://www.anthropic.com/engineering/claude-code-best-practices) —— Anthropic。explore→plan→code→commit，以及为什么"先读再写"的 agent 表现更好。

## 记忆与状态

- [MemGPT: Towards LLMs as Operating Systems](https://arxiv.org/abs/2310.08560) —— 把虚拟内存分页的思路搬到 context window 上。"记忆层级"这个框架就是这篇确立的。
- [Generative Agents](https://arxiv.org/abs/2304.03442) —— memory stream，按 recency / importance / relevance 检索。
- [AGENT.md](https://ampcode.com/AGENT.md) —— 每个仓库放一份 agent 指令文件的约定。
- [agents.md](https://agents.md/) —— 推动这份文件跨厂商标准化的尝试。
- [Letta](https://github.com/letta-ai/letta) —— MemGPT 那套思路的工程化实现，一个有状态 agent server。

## 多 Agent 与编排

- [How We Built Our Multi-Agent Research System](https://www.anthropic.com/engineering/multi-agent-research-system) —— Anthropic。难得地坦诚代价：token 开销、协调失败，以及 fan-out 到底在什么场景下才划算。
- [Don't Build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents) —— Cognition。最有力的反方论证。两篇对照着读。
- [LangGraph](https://github.com/langchain-ai/langgraph) —— 图结构编排，状态显式化。
- [OpenAI Agents SDK](https://github.com/openai/openai-agents-python) —— 把 handoff、guardrail、tracing 做成原语。
- [Pydantic AI](https://github.com/pydantic/pydantic-ai) —— 类型安全的 agent，schema 纪律很好。

## 权限、沙箱与安全

如果你的 harness 能执行命令，它就能执行*坏*命令。这一节不是可选项。

- [The Lethal Trifecta](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/) —— Simon Willison。私有数据 + 不可信内容 + 对外通信 = 数据外泄。这个领域最清晰的威胁模型。
- [Prompt Injection 系列](https://simonwillison.net/series/prompt-injection/) —— Simon Willison。多年积累的攻击案例，以及为什么"过滤"这条路走不通。
- [Design Patterns for Securing LLM Agents against Prompt Injections](https://arxiv.org/abs/2506.08837) —— 六种具体的架构模式，每种都摆明了取舍。
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/) —— 拿来对着自己的 harness 过一遍的清单。
- [E2B](https://github.com/e2b-dev/E2B) —— 给 agent 生成代码用的沙箱云运行时。
- [Claude Code Security Review](https://github.com/anthropics/claude-code-security-review) —— 把 agentic 安全审查做成 GitHub Action。

## Skills 与渐进式披露

一次性把所有指令塞进 context 是浪费。按需加载本身就是一种 harness 模式。

- [Equipping Agents for the Real World with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) —— Anthropic。渐进式披露的设计动机。
- [Agent Skills 文档](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) —— Anthropic。格式规范：frontmatter、随附脚本，以及什么条件下会被加载。
- [anthropics/skills](https://github.com/anthropics/skills) —— 官方参考实现。
- [Superpowers](https://github.com/obra/superpowers) —— Jesse Vincent。一个观点鲜明的 skill 库，把"流程纪律"本身做成了可安装的能力。

## 评估与可观测性

测不出来的 harness 是调不动的。轨迹级评估和模型 benchmark 是两个不同的问题。

- [Your AI Product Needs Evals](https://hamel.dev/blog/posts/evals/) —— Hamel Husain。关于如何搭一套能在生产环境里活下来的 eval 流程，最实用的文章。
- [Creating a LLM-as-a-Judge That Drives Business Results](https://hamel.dev/blog/posts/llm-judge/) —— Hamel Husain。
- [SWE-bench](https://arxiv.org/abs/2310.06770) —— 让 harness 质量第一次变得可量化的 benchmark；光改脚手架分数就会动。
- [SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering](https://arxiv.org/abs/2405.15793) —— 提出 "agent-computer interface" 概念，并证明界面设计带来的收益超过换模型。可以说是 harness engineering 的奠基论文。
- [τ-bench](https://arxiv.org/abs/2406.12045) —— 带真实领域规则的 tool-agent-user 交互评测。
- [Langfuse](https://github.com/langfuse/langfuse) —— 开源 LLM tracing 和评估。
- [Arize Phoenix](https://github.com/Arize-ai/phoenix) —— agent 轨迹的 tracing 与评估。

## 可参考的 Harness 实现

读源码。这是学这门手艺最快的路。

- [Claude Code](https://github.com/anthropics/claude-code) —— Anthropic 的 coding agent。
- [Codex CLI](https://github.com/openai/codex) —— OpenAI 的，Rust 写的。
- [Gemini CLI](https://github.com/google-gemini/gemini-cli) —— Google 的，Apache-2.0。
- [Aider](https://github.com/Aider-AI/aider) —— 成熟项目，repo-map 和 edit-format 的工程做得格外扎实。
- [OpenHands](https://github.com/All-Hands-AI/OpenHands) —— 研究级、完全开源，沙箱方案很完整。
- [SWE-agent](https://github.com/SWE-agent/SWE-agent) —— ACI 那篇论文的可运行版本。
- [Cline](https://github.com/cline/cline) —— VS Code agent，plan/act 分离做得很显式。
- [OpenCode](https://github.com/sst/opencode) —— 终端 agent，不绑定 provider。
- [Goose](https://github.com/block/goose) —— Block 出品，可扩展的本地 agent。
- [Continue](https://github.com/continuedev/continue) —— 自建 IDE 内助手的积木。

## 自己动手做一个

- [Claude Agent SDK (Python)](https://github.com/anthropics/claude-agent-sdk-python) —— 把 Claude Code 的 harness 变成一个库。
- [Claude Agent SDK (TypeScript)](https://github.com/anthropics/claude-agent-sdk-typescript)
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) —— 可直接跑的范式代码。
- [OpenAI Cookbook](https://github.com/openai/openai-cookbook)
- [LiteLLM](https://github.com/BerriAI/litellm) —— 统一多家 provider 的接口；做 harness 层 A/B 实验很方便。
- [HumanLayer](https://github.com/humanlayer/humanlayer) —— 把 human-in-the-loop 审批做成基础设施。
- [System Prompts and Models of AI Tools](https://github.com/x1xhlol/system-prompts-and-models-of-ai-tools) —— 收集自各家在售产品的 system prompt。一手材料，研究真实 harness 设计无可替代。
- [Decoding Claude Code](https://minusx.ai/blog/decoding-claude-code/) —— MinusX。对一个生产级 harness 的细读。

## 论文

上文引用之外：

- [Chain-of-Thought Prompting](https://arxiv.org/abs/2201.11903)
- [Toolformer](https://arxiv.org/abs/2302.04761)
- [Tree of Thoughts](https://arxiv.org/abs/2305.10601)
- [Voyager](https://arxiv.org/abs/2305.16291) —— 可以随时间不断组合叠加的 skill library。
- [Lost in the Middle](https://arxiv.org/abs/2307.03172) —— 为什么信息在 context 中的位置很重要，以及为什么"直接上长 context"不管用。
- [AgentBench](https://arxiv.org/abs/2308.03688)
- [WebArena](https://arxiv.org/abs/2307.13854)

---

## 本仓库的深度笔记

不只是罗列链接，而是把上面这些东西消化整合过的长笔记：

- [Context Engineering](docs/context-engineering.zh-CN.md) · [EN](docs/context-engineering.md)
- [工具设计](docs/tool-design.zh-CN.md) · [EN](docs/tool-design.md)

**计划中：** agent loop · 记忆与状态 · 多 agent 编排 · 权限与沙箱 · 评估。[欢迎贡献。](CONTRIBUTING.md)

---

## 贡献

请看 [CONTRIBUTING.md](CONTRIBUTING.md)。标准只有一条：它必须教会读者关于**如何搭脚手架**的东西，而不是关于怎么和聊天机器人对话、或者怎么训模型。每一条都要写一句话说明"从中能学到什么"。

## 许可

[CC0 1.0](LICENSE) —— 公共领域。
