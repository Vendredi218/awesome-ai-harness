# Context Engineering

*工作笔记。[English](context-engineering.md) · [返回首页](../README.zh-CN.md)*

---

## 一句话版本

Context window 不是存储空间，而是**你每一轮都要重新花一遍的预算**。harness 的工作就是决定什么东西配占这个位置。

## 为什么这个框架重要

Prompt engineering 假设的是一次性交互：写个好 prompt，拿个好答案。Agent 打破了这个假设。一个 agent 会跑五十轮，而每一轮都会把累积的全部历史重新发一遍 —— system prompt、所有工具定义、读过的每个文件、每条命令的输出、犯过的每个错。Context 不是写一次就完事的，它在**每一次前向传播时都被重新构建**。

这就把工程问题从「我该说什么」变成了「此时此刻什么应该在场，什么应该消失」。

## 四种失败模式

Drew Breunig 的分类是这个领域最有用的词汇表。学会在你的 trace 里认出它们：

| 失败模式 | 表现 |
|---|---|
| **Poisoning（污染）** | 一个幻觉进入 context，之后就被当作事实。agent 在第 3 轮编了个函数名，到第 30 轮还在调它。 |
| **Distraction（分心）** | Context 太长，模型过度关注自己的历史而不是解决任务。症状是：反复重做已经做过的事。 |
| **Confusion（混淆）** | 无关材料影响了输出。挂了 60 个工具，只有 3 个相关，然后它挑错了。 |
| **Clash（冲突）** | Context 里两部分互相矛盾。早期的计划说 X，后来的观察说 not-X，没人去调和。 |

注意其中三种会随着 context 变长而**恶化**。这就是核心张力：信息更多 ≠ 能力更强。

## 为什么「直接上 1M 窗口」解决不了

三个独立的原因，而且会叠加：

1. **注意力随位置衰减。**[Lost in the Middle](https://arxiv.org/abs/2307.03172) 表明，长 context 中间位置的材料检索准确率会明显下降。把关键那行埋在第 40 万个 token，跟没写差不多。
2. **成本和延迟大致随 token 线性增长，而且每轮都算一次。** 一个背着 200k token 的 50 轮 agent，要为这 200k 付五十次钱。
3. **上面那些失败模式是信噪比问题，不是容量问题。** 给分心留出更大的空间，并不能减少分心。

目标是**能让下一个决策正确的、最小的高信号 token 集合**，而不是塞得下的最大集合。

## 具体手段

大致按「单位工程投入的杠杆率」排序。

### 1. Just-in-time 检索（杠杆率最高）

不要预加载。给 agent 工具，让它在需要的时候自己去取。一个文件路径大约 10 个 token，文件本身大约 5000。让 agent 先攥着路径，等它判断这文件真的重要时再花那 5000。

这就是为什么在大多数 coding agent 上，`grep` + `read` 的组合会打败预建的向量索引：agent **在需要的当下、用自己的判断**去评估相关性，比事先算好的相似度分数更准。而且它留下了一条清晰可读的记录，说明每样东西为什么被加载进来。

### 2. Compaction（压缩）

窗口满了就把历史总结掉，带着摘要继续。工程问题不是「要不要压缩」，而是**什么必须在压缩中幸存**：未决的决策、没解决的错误、当前目标。可以丢的：完整文件内容（可以重读）、冗长的工具输出、走进死胡同的探索。

好的压缩保留的是**工作的状态**，差的压缩保留的是「发生了什么」的流水账。

### 3. Offload 到文件系统

最可靠的记忆根本不在 context window 里。让 agent 把笔记、计划、中间结果写进文件，需要时再读回来。这样能扛过 compaction、扛过 session 边界，而且人类可以直接检查。

让 agent 自己维护一个 `TODO.md`、每轮重读一次 —— 相对于它的复杂度，这个机制的效果好得惊人。

### 4. Sub-agent 隔离

把一个边界清晰的任务交给一个 context 全新的 agent，只把结论拿回来。一次读了 40 个文件的搜索，返回一段话。那 40 个文件从来没碰过父 agent 的 context。

代价是真实存在的（见 [Cognition](https://cognition.ai/blog/dont-build-multi-agents)）：sub-agent 缺少父 agent 的上下文，可能做出父 agent 不会做的决定。适合用在**输出远小于输入**、且任务确实自包含的场景。搜索和 review 合适，写实现代码通常不合适。

### 5. 工具集管理（tool loadout）

只有 6 个工具相关的时候，别挂 80 个定义。按需加载 schema，或者按任务挑子集。这直接打击 *confusion*，而且工具定义在被用到之前，每一轮都是纯开销。

### 6. 结构化笔记

让 agent 用固定格式维护一份显式的运行摘要 —— 目标、已做的决定、待解的问题、下一步。这比从历史里重新推导状态便宜，而且能让 clash 浮出水面：矛盾在一份两行的摘要里一眼可见，在 40k token 的对话记录里则永远看不出来。

## 实用经验

- **稳定的内容放前面。** System prompt 和工具定义放最前，逐轮变化的材料放后面。这是 prompt caching 生效的前提，而 caching 是 agent harness 里最大的单项成本杠杆。
- **返回错误，而不是抛异常。** 工具失败时应该返回一条模型能据以行动的消息。这就是 context engineering —— 错误信息**本身就是**下一轮的指令。
- **留意重复读取。** 如果你的 agent 把同一个文件读了三遍，说明你的 context 管理丢掉了它需要的东西。这是一个诊断信号，不是小毛病。
- **显式做预算。** 大致知道你的窗口里有多少比例给了 system prompt、工具、历史。如果工具占了 30% 而且大部分没被用到，问题就找到了。
- **在 review 里报 token 数。** 评估一个 harness 改动时，「能跑」是不够的 —— 它是不是用更少的 token 跑通的？这才是 harness engineering 竞争的那个维度。

## 尚未解决的问题

坦白讲还没定论的部分：

- **什么时候该用 compaction，什么时候该用 sub-agent？** 两者都在做压缩。compaction 保住了一条连贯的线索但丢了细节；sub-agent 在局部保住了细节但丢了连贯性。目前还没有好的判断规则。
- **怎么单独评估 context 管理？** 端到端的任务成功率把它和其他一切混在一起。没人有一个干净的 benchmark 来测「harness 有没有留对东西」。
- **这些东西在更强的模型面前还成立吗？** 一部分是围绕当前弱点搭的脚手架；另一部分 —— 成本论证、信噪比论证 —— 看起来是结构性的。哪些是哪些，还不清楚。

## 来源

- [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) —— Anthropic
- [How Long Contexts Fail](https://www.dbreunig.com/2025/06/22/how-contexts-fail-and-how-to-fix-them.html) 与 [How to Fix Your Context](https://www.dbreunig.com/2025/06/26/how-to-fix-your-context.html) —— Drew Breunig
- [Context Engineering for Agents](https://blog.langchain.com/context-engineering-for-agents/) —— LangChain
- [Lost in the Middle](https://arxiv.org/abs/2307.03172)
- [Don't Build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents) —— Cognition
