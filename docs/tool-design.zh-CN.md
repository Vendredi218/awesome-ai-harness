# 工具设计

*工作笔记。[English](tool-design.md) · [返回首页](../README.zh-CN.md)*

---

## 一句话版本

工具定义是 prompt，工具返回值是 prompt，而工具的错误信息是你会写的最重要的那个 prompt —— 因为那是模型在**已经出错的时候**读到的东西。

## 心智模型：你在为一个不睡觉、也从不发问的同事设计界面

[SWE-agent 论文](https://arxiv.org/abs/2405.15793)仿照 HCI 造了 **agent-computer interface（ACI）** 这个词，并展示了一个让人不太舒服的结果：重新设计界面带来的 benchmark 提升，超过了换模型。同一个模型，换成一个「写入时会报语法错误」而不是「默默接受坏代码」的文件编辑器，表现就大幅变好。

这个结果本身就是认真对待工具设计的全部理由。你以为模型缺的能力，其实是界面缺的。

和人类同事不同，agent 不会问你某个参数是什么意思，不会注意到文档过期了，也不会随时间积累直觉。它关于你这个工具的全部认知，都来自 schema 和之前那些调用返回了什么。

## 原则

### 1. 工具要更少、更强

每个工具定义在每一轮都要花 token，而且是永远。更重要的是，每多一个就多一次挑错的机会。三个职责清晰、互不重叠的工具，好过十二个边界模糊的。

判断标准：如果你没法用一句话说清「什么时候该用 A 而不是 B」，模型也说不清。要么合并，要么把边界磨锋利。

### 2. 围绕 workflow 设计，而不是围绕你的 API 表面

错误的直觉是：手上有几个 endpoint 就包几个工具。正确的直觉是问：**agent 实际上要完成什么**，然后为那件事造一个工具。

如果完成一件常见任务需要串四次调用、agent 还得在中间手工传 ID，那就是四次失败机会，外加一堆中间垃圾塞进 context。直接出一个把整个 workflow 做完的工具。

### 3. 返回值要考虑读者的 token 预算

一个返回 5 万 token JSON 的工具已经把井水搞脏了，哪怕这次调用是「成功」的。想清楚模型要决定下一步实际需要什么，然后只返回那些。

具体来说：
- 要截断，而且**明说自己截断了**，并告诉怎么拿更多。静默截断会让模型对着它根本没收到的数据自信推理。
- 优先返回 ID 和路径，而不是内联大块内容 —— 让 agent 想要时自己去取。
- 对于模型需要**推理**而不是**解析**的东西，自然语言比深层嵌套 JSON 更好。
- 提供一个详细程度参数（`concise` / `full`），默认给 concise。

### 4. 错误信息就是指令

大多数工具设计翻车的地方就在这里。对比一下：

```
Error: 400 Bad Request
```

```
Error: 'user_id' 必须是 UUID，收到的是 "alice"。
请先用 search_users(name="alice") 查到对应的 UUID。
```

第一条终结了 agent 这一轮的生产力 —— 它会重试、瞎猜，或者放弃。第二条是一条能直接导向恢复的、可执行的指令。**写错误信息时，就当它是 system prompt 的下一行** —— 因为在功能上它就是。

好的错误信息要说清楚：错在哪、期望是什么、下一步该做什么。参数校验失败是这里投入产出比最高的地方。

### 5. 命名和描述在干实事

模型是靠它们做路由的。用明确、带命名空间的名字（`calendar_create_event`，不要 `create`）。在描述里明确写出**什么时候不该用这个工具** —— 反向指引挡下的误用比正向指引更多。

避免那种「含义依赖于模型没有的上下文」的参数。如果一个字段能装两种不同的 ID，它就一定会收到错的那种。

### 6. 让危险的路比安全的路更难走

如果一个工具能毁掉东西，界面本身就该让「毁掉」变成一个刻意动作：要求显式 flag、要求目标必须先被读过、提交前先返回预览。**不要指望 system prompt 里的嘱咐能拦住它。结构胜过嘱咐** —— 有压力的 agent 会走阻力最小的路，那就把阻力最小的那条路修成安全的那条。

### 7. 像评估代码一样评估工具

准备一组真实任务，跑 agent，然后读 transcript。不是看通过率，是**读 transcript**。你要找的是：它第一个伸手去够的是哪个工具、它在哪里困惑了、它重试了什么、它用什么畸形参数调用过。这里面每一条都是工具的设计 bug，不是模型的失败。

然后改工具，再跑一遍。这个循环的效果好得不合理，而且几乎没人做。

## 规模变大之后：token 墙

一旦工具数量到了几十个 —— 接了几个 MCP server 就是这个状态 —— 光是定义就能在开工之前吃掉一大块 context。目前有两条缓解路线：

- **延迟加载 / 可搜索的 tool schema。** 只呈现名字，schema 按需加载。
- **Tools as code。** 把工具暴露成一套 API 让 agent 写代码去调，这样过滤和组合都发生在执行环境里，而不是在 context 里。见 [Code Execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp)。

第二条更强，也更不成熟。它同时改变了失败面：现在你得去沙箱化代码执行了。

## 一份 checklist

工具上线前：

- [ ] 有一句话说清楚什么时候用它、什么时候不用
- [ ] 名字明确且有命名空间
- [ ] 没有任何参数有可能收到错误种类的值
- [ ] 典型返回值能舒服地装进几百个 token
- [ ] 大返回值会「大声地」截断，并写明怎么拿更多
- [ ] 每条错误信息都指出了修复方法
- [ ] 破坏性操作需要一个显式的、刻意的步骤
- [ ] 你已经读过至少十份 agent 真实使用它的 transcript

## 来源

- [Writing Effective Tools for Agents](https://www.anthropic.com/engineering/writing-tools-for-agents) —— Anthropic
- [SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering](https://arxiv.org/abs/2405.15793)
- [Advanced Tool Use](https://www.anthropic.com/engineering/advanced-tool-use) —— Anthropic
- [Code Execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp) —— Anthropic
- [The "Think" Tool](https://www.anthropic.com/engineering/claude-think-tool) —— Anthropic
