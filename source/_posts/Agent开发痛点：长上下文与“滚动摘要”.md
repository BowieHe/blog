---
title: Agent开发痛点：长上下文与“滚动摘要”
date: 2025-07-25 11:02:44
updated: 2025-07-25 12:02:44
categories:
    - 教程
tags:
    - LangGraph
comments: true
description: Agent对话长易卡死，源于“长上下文悖论”。破局之道：引入“滚动摘要”架构，仅处理增量信息。实现无限对话，高效经济，彻底解决LLM上下文超限问题。
keywords:
top_image: https://blog-1321748307.cos.ap-shanghai.myqcloud.com/img/langgraph-cover.jpeg
cover: https://blog-1321748307.cos.ap-shanghai.myqcloud.com/img/langgraph-cover.jpeg
---

我目前正在使用 **LangGraph** 来开发一个旅行相关的 Agent。在构建过程中，我遇到了一个非常普遍但又棘手的问题：随着对话的深入和图中节点（node）之间消息的不断流转，`messages`（消息历史）会不断累积，很容易导致整个 Agent 的上下文超出大语言模型（LLM）的限制。

我们都知道，LLM 的上下文窗口是有限的，一旦消息量过大，Agent 就会“失忆”或“卡壳”。在不考虑使用外部存储（如数据库、Memory Bank 或向量数据库）的情况下，我一直在探索如何仅通过 Agent 内部的机制，特别是通过一个**总结节点（summarizer node）**来最小依赖化地实现上下文的压缩和管理。

然而，在这个探索过程中，我发现了一个非常隐蔽但又致命的缺陷——一个关于“长上下文”的悖论。

![面对上下文过重的痛苦](https://blog-1321748307.cos.ap-shanghai.myqcloud.com/img/headache-langchain.png)

### 致命的悖论：用“长上下文”解决“长上下文问题”

你可能遇到过这样的场景：你的 Agent 聊着聊着，突然在打印完类似 `---SUMMARIZER---` 的提示后就陷入了漫长的等待，甚至直接没有响应。这并不是偶然现象，而是我们早期 Agent 设计中一个非常隐蔽但又致命的缺陷。

这个缺陷可以用一句话来概括：**我们试图让一个“摘要员”来解决对话太长的问题，结果它自己却先被“对话太长”给累垮了。**

**举个例子：** 在 LangGraph 的教程中，我们可能会看到这样的策略——通过一个**条件边（conditional edge）**来判断是否需要触发总结。例如，设定每 5 次对话就进行一次总结，或者当消息的 Token 长度超过某个阈值（比如 1500 Token）时就进入总结流程。

**但即使是这样，问题依然存在，甚至更加突出：**

1.  **“卡在阈值上”的风险：** 想象一下，当前对话已经累积了 1490 个 Token，新的用户输入又增加了 20 个 Token，总数达到了 1510。此时，虽然触发了总结，但`summarizer`接收到的上下文已经是接近或超出 LLM 极限的 1510 个 Token。它在尝试总结这个**完整的、超长**的对话历史时，依然可能因为请求过大、处理时间过长而“卡住”或超时。
2.  **消息累积的盲区：** 即使我们设定每 5 轮总结一次，但在第 1 轮到第 5 轮之间，消息依然在累积。如果用户在第 3 轮就输入了超长的内容，那么在总结触发之前，Agent 的上下文可能就已经爆炸了。

具体来说，当我们的 Agent 与用户聊得越来越多，对话历史越来越长时，系统会意识到上下文信息（也就是我们说的“Token”）超出了大模型的处理范围（比如超过了 2000 个 Token）。这时，一个被称为 `entryRouter` 的“守门员”会把任务交给专门负责“总结”的 `summarizer`（摘要员）节点。

问题就出在这里：这个 `summarizer` 接收到的，竟然是**全部的、超长的对话历史**（比如 6000+ Token）。然后，它会把这一大坨信息，连同“请帮我总结一下”的复杂指令，一股脑地塞给后端的大模型（比如 OpenAI 的 `gpt-4.1-mini`）。你想想，一个模型要处理如此庞大且复杂的请求，需要多长时间？它很可能会超过某个隐藏的超时限制，导致请求“卡住”，既不成功也不报错，就那么一直等着。

**简而言之，我们犯了一个根本性错误：用一个本身会消耗大量上下文的大模型调用，去解决上下文过长的问题。这就像是让一个感冒的医生，通过接触更多病人来治好自己的感冒一样，逻辑上完全行不通** **。**

### 破局之道：引入“滚动摘要（Rolling Summary）”架构

要彻底解决这个“卡死”问题，我们需要对 Agent 的记忆管理机制进行一次根本性的升级，引入一种更智能、更经济、更健壮的**“滚动摘要”**机制 。

“滚动摘要”的核心思想非常简单：**我们永远不让 Agent 的任何一个部分去处理完整的、超长的原始对话历史。** 相反，我们只关注**增量信息**。想象一下，你有一本笔记，每次你只记录最新的内容，然后定期把旧的、零散的笔记整理成一个精炼的总结，并且这个总结会不断更新，涵盖你所有的旧知识。这样，你永远只需要看最新的笔记和最新的总结，而不需要从头翻阅所有旧笔记。

### 如何实现“滚动摘要”？——技术细节与伪代码

下面，我们来看看如何一步步地实现这个“滚动摘要”架构。

#### 1. 升级 Agent 的“状态蓝图”（AgentState）

首先，我们需要重新定义 Agent 存储对话状态的方式。以前，我们可能只有一个 `messages` 列表来存放所有对话。现在，我们需要把它拆分成两部分：

-   `summary`: 一个字符串，用来保存**到目前为止所有对话的精炼总结**。
-   `messages`: 一个列表，只存放**自上次总结以来，最新发生的几条消息**。

**伪代码示例：**

```
// src/state.ts

// 定义Agent的对话状态结构
class AgentState {
    summary: string;     // 存储历史对话的滚动摘要
    messages: Array<Message>; // 存储自上次摘要以来的最新消息
    memory: Object;      // 结构化记忆（例如关键实体、用户偏好等）
    // ... 其他可能的状态字段
}

// 示例：一条消息的结构
class Message {
    sender: 'user' | 'agent';
    content: string;
    timestamp: Date;
}
```

![上下文被正确的总结](https://blog-1321748307.cos.ap-shanghai.myqcloud.com/img/summary-context-langgraph.png)

#### 2. 重构“摘要员”（Summarizer）的逻辑

现在，我们的 `summarizer` 不再是那个“笨重”的总结者了。它变得更聪明、更高效。当它被触发时，它接收到的上下文将不再是全部历史，而是：

-   **旧的摘要** (`state.summary`)
-   **最新的几条消息** (`state.messages`)

它的任务也变得简单清晰：将“旧摘要”和“最新消息”合并，生成一个**全新的、更新后的摘要**。同时，它还可以根据最新对话更新一些结构化的`memory`（比如用户提到的关键人物、地点等）。最最关键的一步是，完成总结后，它会**清空** `messages` 数组，为下一次的“滚动”做好准备。

**伪代码示例：**

```
// src/graph.ts (createSummarizer 内部逻辑)

function createSummarizer(state: AgentState): AgentState {
    // 1. 准备大模型输入：结合旧摘要和最新消息
    const prompt = `
        你是一个对话摘要员。
        这是之前的对话总结：
        ---
        ${state.summary}
        ---
        这是最新的对话内容：
        ---
        ${state.messages.map(msg => `${msg.sender}: ${msg.content}`).join('\n')}
        ---
        请结合上述信息，生成一个更新后的精炼总结。
        同时，从最新对话中提取任何需要更新的结构化记忆（如用户偏好、重要实体等）。
        你的输出格式应该是：
        <SUMMARY>新的总结内容</SUMMARY>
        <MEMORY>{"key": "value", ...}</MEMORY>
    `;

    // 2. 调用大模型进行摘要
    // 这里的 LLM_CALL 是对大模型API的抽象调用
    const llm_response = LLM_CALL(prompt, { model: 'gpt-4.1-mini', max_tokens: 500 });

    // 3. 解析大模型响应
    const new_summary = extractContentBetweenTags(llm_response, '<SUMMARY>');
    const updated_memory = JSON.parse(extractContentBetweenTags(llm_response, '<MEMORY>'));

    // 4. 更新 Agent 状态：生成新的摘要，清空最新消息，更新记忆
    return {
        ...state,
        summary: new_summary,
        memory: mergeMemory(state.memory, updated_memory), // 合并新旧记忆
        messages: [] // 清空最新消息，为下一次滚动做准备
    };
}
```

#### 3. 升级“协调器”（Orchestrator）的提示词

`orchestrator` 是 Agent 的“大脑”，它负责根据当前上下文来决定下一步做什么。现在，它的决策依据不再是可能超长的原始对话，而是两个清晰、精炼的部分：

-   `summary`：包含了所有必要的历史背景和长期记忆。
-   `messages`：只包含了用户最新的、需要立即处理的输入。

这样，`orchestrator` 就能更高效、更准确地做出判断，因为它收到的信息总是“刚刚好”，既有宏观背景，又有微观细节。

**伪代码示例（概念性）：**

```
// src/agents/orchestrator.ts

function createOrchestrator(state: AgentState): Action {
    // 1. 构造给大模型的决策提示
    const decision_prompt = `
        你是一个Agent协调器，根据对话历史和最新用户输入决定下一步行动。
        历史背景（总结）：${state.summary}
        最新用户输入：${state.messages[state.messages.length - 1].content} // 只关注最新一条消息
        你的可用工具和能力有：[工具列表]
        请决定下一步应该做什么，例如：
        - 回答用户问题
        - 调用某个工具
        - 寻求用户澄清
        - ...
    `;

    // 2. 调用大模型获取决策
    const llm_decision = LLM_CALL(decision_prompt, { model: 'gpt-4-turbo' });

    // 3. 解析决策并执行相应的行动
    return parseDecisionAndExecute(llm_decision);
}
```

#### 4. 调整图的“管道”（Graph Pipeline）

最后，我们需要确保整个 Agent 的运行流程（或者说“图”的“管道”）能够正确地处理新的状态结构。这意味着在 Agent 状态流转的各个节点，都要确保数据是按照 `summary` 和 `messages` 分离的格式进行传递和处理。这更多是配置和数据流的调整，而非具体的伪代码实现。

### 新架构带来的巨大好处

这个“滚动摘要”的新架构，为我们的 Agent 带来了多方面质的飞跃：

1.  **彻底告别“卡死”：** 因为任何一个大模型调用都不会再收到超长的上下文，从根本上解决了 API 调用挂起或超时的顽疾。
2.  **实现无限对话：** 理论上，你的 Agent 现在可以进行无限轮次的对话，而不用担心上下文窗口的限制，因为它总是在处理增量信息 。
3.  **高效且经济：** 每一次大模型调用都只处理少量、精炼的信息，大大降低了 Token 消耗，从而节省了大量的 API 费用 。
4.  **更加健壮和专业：** 这种设计是构建企业级、生产级对话 Agent 的标准实践，让你的 Agent 系统更加稳定、可扩展。

通过引入“滚动摘要”机制，我们不仅解决了长上下文的痛点，更让 Agent 真正具备了“长期记忆”和“无限对话”的能力.
