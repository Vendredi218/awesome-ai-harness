# 给开源 Harness 做贡献

*工作笔记。[English](contributing-to-open-harnesses.md) · [返回首页](../README.zh-CN.md)*

---

## 一句话版本

学 harness engineering 最快的方式，是去扩展一个别人跑在生产环境里的 harness —— 而正确的参与方式，完全取决于这个项目开放的是核心代码，还是生态。

## 为什么要去贡献

读 context 预算的文章是一回事。亲手写一个 Pi extension、hook 住 `session_before_compact`、看着自己的代码决定什么能在 compaction 里幸存，是另一回事。一个真实的 plugin 会逼你走完 session 状态、context 预算、tool dispatch、provider 抽象 —— [harness 设计综述](https://arxiv.org/abs/2606.20683)说每个 harness 都必须回答六项运行时职责，这就是其中四项（剩下两项是权限和恢复）。没有比这更快的课程了。

还有一个自利的职业论证：harness engineering 这个领域几乎还没有任何 credential 体系。一个被 merge 的 benchmark task，或者一个发布出去的 plugin，**就是** credential 本身。

## 先看清 governance 地图

Harness 项目分成两个阵营，认错阵营会浪费你几个月：

| | Core-open（核心开放） | Ecosystem-first（生态优先） |
|---|---|---|
| **项目** | OpenHands、Cline、Continue、Gemini CLI、Goose、Aider | Claude Code、DeepSeek Harness（dsh）、Codex CLI |
| **给引擎提 PR** | 欢迎，伴随正常的 OSS 摩擦 | 核心闭源，或仅限受邀 |
| **官方期望的参与面** | 代码本身 | Plugin、skill、extension、registry |
| **「好贡献」意味着什么** | 一个被 merge 的 PR | 一个发布的 extension、一个验证过的 skill、一份 root-cause 分析 |

Ecosystem-first 的姿态不是敌意 —— 是 maintainer 注意力的经济学。Codex 的 contributing 文档说得很直白：review 的成本超过了实现的成本，所以他们想从外部人那里得到的是**诊断**，不是 diff。在那里，一份可复现的 root-cause 分析就是 first-class 贡献。

选定目标之前，先确认它在玩哪一局。然后再挑路线。

## 六条路线，按学习密度排序

### 1. 在深 API 的 harness 上写 programmatic plugin —— 学 loop 本身

单位小时学习密度最高的路线。三个 harness 暴露的 API 深到足以教真正的内部机制：

- **Pi**（[extensions.md](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/extensions.md)）—— 用 TypeScript 对着一个显式的事件面写 extension：`session_before_compact`、`before_provider_request`、工具拦截。小到可以从头读到尾；事件和 harness 概念一一对应。
- **dsh**（[Cordis tutorial](https://deepseek-harness.github.io/deepseek-harness/en/develop/cordis-tutorial/)）—— 全领域最深的 plugin API：依赖注入、typed events、hot module reload，而且**一切**都可替换，包括 agent loop 本身。发布方式是给你的仓库打 `dsh-plugin` topic —— registry 是去中心化的。
- **Cline**（[plugins](https://docs.cline.bot/sdk/plugins)）—— `AgentPlugin` 的设计带 hook 策略（blocking vs async、超时、`failureMode`），是「怎么在 agent loop 里安全地跑第三方代码」这个问题目前能找到的最好教材。

它教什么：loop、消息组装、tool dispatch、生命周期。它不教什么：没有 —— 这条路线就是完整课程。唯一的代价是你的工作绑死在一个 harness 上。

### 2. 贡献一个 eval task —— 最被低估的路线

Terminal-bench 从 93 个贡献者那里积累了 229 个任务；benchmark task 的作者身份如今是一种被正式承认、有人 review、署你名字的贡献。[Harbor task format](https://harborframework.com/docs/task-format) 就是这门手艺的微缩版 —— 每个 task 都是四件东西：

1. 一条 instruction，
2. 一个 Docker 化的环境，
3. 一个输出 `reward.json` 的 verifier，
4. 一个证明任务确实可解的 oracle `solve.sh`。

学习发生在写 **verifier** 的时候：你会精确见识到 agent 怎么作弊、怎么抄近道、怎么「技术上满足了要求」。[cline-bench](https://cline.ghost.io/cline-bench-initiative/) 走了条更直接的路 —— 把真实弄崩过你 agent 的那个任务直接贡献出来。

它教什么：verification 设计、agent 的失败模式、环境可复现性。想知道什么能弄崩 harness，没有比亲手写一个以弄崩它为目的的任务更快的了。

### 3. 发布一个 MCP server —— 可移植性最大化

一个 artifact，所有主流 harness 通吃。[registry](https://github.com/modelcontextprotocol/registry) 给它提供 discovery；[Goose 的 extension 文档](https://goose-docs.ai/docs/tutorials/custom-extensions)把这个卖点直接演出来了 —— 一个 Goose extension **就是**一个 MCP server，不需要任何适配。

它教什么：tool surface 的 ergonomics、schema 设计、token 成本权衡 —— 整篇[工具设计](tool-design.zh-CN.md)，用动手的方式过一遍。它不教什么：你始终待在 loop **外面**。在 MCP server 里面，你学不到任何关于 context 管理或 agent 生命周期的东西。进场之前把这一点想明白。

### 4. Skill、recipe、block —— 摩擦最低的真实 PR

声明式那一层：不写代码，只写结构良好的指令。

- **OpenHands**（[extensions registry](https://github.com/OpenHands/extensions)）：一个 `SKILL.md` 目录加一条 JSON —— 大概率是全领域摩擦最低的、能被 merge 的 PR。
- **Claude Code**（[plugin docs](https://code.claude.com/docs/en/plugins)）：`claude plugin validate` → 社区 marketplace review → SHA-pinned catalog。这条流水线本身就值得当作 supply-chain 设计来研究。
- **Continue**（[blocks](https://docs.continue.dev/customize/overview)）、**Goose**（recipes），以及跨 harness 的 [skills.sh](https://www.skills.sh/) 目录。

它教什么：progressive disclosure —— 写那种模型按需加载的指令 —— 以及为一个毫无上下文的读者写作的纪律。一个 skill 就是一个微型 system prompt；写好它，等于带 merge 门槛的 prompt engineering。

### 5. Core PR —— 只适用于 core-open 阵营

经典路线在欢迎它的地方依然管用：OpenHands、Cline、Continue、Gemini CLI、Goose、Aider 都维护着 good-first-issue 标签。声望最高，反馈循环最慢，最受 maintainer 带宽制约。从那些你已经做完路线 6 功课的 issue 开始。

### 6. Root-cause 分析 —— 闭核项目的贡献方式

在 ecosystem-first 项目上，一份可复现的诊断本身就是贡献：最小 repro、trajectory 摘录、假设、修复建议。这就是 trajectory debugging —— 可以说是 harness engineering **的**核心技能 —— 而且是在你没参与搭建的生产系统上练。想看白纸黑字的版本，读 [Codex 的政策](https://github.com/openai/codex/blob/main/docs/contributing.md)。

## 无聊但会咬人的部分

- **CLA 和 DCO。** Google（Gemini CLI、ADK）、AWS（Strands）、字节（Trae）要求签 contributor license agreement；很多其他项目要求 DCO sign-off（`git commit -s`）。动手写代码之前先查，不要写完才查。
- **License 的不对称。** Crush 是 FSL-1.1-MIT（source-available，两年后转 MIT）；Skyvern 是 AGPL 且部分功能 cloud-only。两个都是不错的**学习**场所 —— 但投入之前，先搞清楚你的 plugin 在法律上能做什么、你的贡献最终归到哪里。
- **安全礼仪。** 把第三方 MCP server 和 skill 接进任何东西之前先扫一遍（[mcp-scan](https://github.com/invariantlabs-ai/mcp-scan)）；永远不要发布内嵌 credential、或在运行时拉取远程指令的 skill。
- **钱开始进场了。** Goose 有付费 recipe，Cline 有 Builder Credits。生态贡献开始能赚钱了 —— 这也意味着 marketplace 会变得更垃圾、review 门槛会更高。

## 一句诚实的注脚

这份指南画的是**门开在哪儿** —— 内容整理自各项目自己的 contributor 文档，时点是 2026 年 8 月。它还不包含 acceptance rate 或 time-to-merge 数据，而且写在文档里的政策不等于实际执行的政策。往任何一条路线上投入认真精力之前，先翻翻那个项目最近 merge 的 PR，看看实际被接受的长什么样。（如果你系统性地做了这件事，这份分析本身就会是对本 repo 的一个很好的贡献。）

## 中文生态

dsh 的去中心化注册模式（给仓库打 `dsh-plugin` topic 即完成发布）让中文开发者可以完全绕过英文 PR 流程参与生态；[awesome-deepseek-harness](https://github.com/libukai/awesome-deepseek-harness) 是这个生态的地图。AgentScope（阿里）和 Youtu-Agent（腾讯）的 issue 区都活跃接受中文交流。用中文写 harness engineering 的一手实践笔记，本身就是这个领域稀缺的贡献。

## 来源

- [From Question Answering to Task Completion: A Survey on Agent System and Harness Design](https://arxiv.org/abs/2606.20683)
- Claude Code、Gemini CLI、Codex、dsh、Pi、Cline、OpenHands、Goose、Continue 各自的 contributor 文档（已在正文内链接）
- [Harbor task format](https://harborframework.com/docs/task-format) · [cline-bench](https://cline.ghost.io/cline-bench-initiative/)
- [DeepSeek Harness: Everything Is a Plugin](https://saikodev.ai/blog/deepseek-harness) —— SaikoDev，dsh/Pi 的 governance 对比取自此文
