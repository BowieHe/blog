---
title: LangGraph vs. LangChain：怎么选
date: 2025-08-05 13:56:04
updated: 2025-08-05 13:56:04
categories:
    - 教程
tags:
    - LangGraph
comments: true
description: Agent开发过程中LangChain和LangGraph的选择
top_img: https://blog-1321748307.cos.ap-shanghai.myqcloud.com/img/4863ed8b-895d-41ca-9b61-e4c9ed36877a.png
cover: https://blog-1321748307.cos.ap-shanghai.myqcloud.com/img/4863ed8b-895d-41ca-9b61-e4c9ed36877a.png
---

最近，我正在进行一个有趣的项目：将我之前用 NodeJS 开发的 Agent 框架，迁移到更适合桌面应用的 Electron 上。在这个过程中，一个经典的问题浮现在我脑海里：

> _在当下的 AI Agent 浪潮中，到底哪款框架才是最合适的？它们各自的优缺点是什么？尤其是在 LangChain 和新秀 LangGraph 之间，我该如何抉择？_

带着这个疑问，我做了一番调研。市面上有不少小型框架，但要论规模和成熟度，除了 LangChain 和 LangGraph，另外两个重量级选手就是 LlamaIndex 和微软的 AutoGen。

在正式对决之前，我们先快速过一下这两位“备选”选手，以及为什么它们没有进入我的最终选择。

### 初步筛选：为什么不是 LlamaIndex 和 AutoGen？

#### LlamaIndex

-   **核心定位：** 专注于数据处理，特别是数据的摄取、索引和检索。如果你的 Agent 核心任务是围绕海量非结构化数据进行问答（RAG），LlamaIndex 无疑是顶级的选择。
-   **Agent 能力：** 它也提供了基于检索和工具的 Agent 框架，但其核心优势依旧牢牢地扎根在 RAG 上。
-   **适用场景：** 当你的 Agent 需要一个极其庞大且复杂的知识库，并要求极致的检索优化时，LlamaIndex 会大放异彩。

#### AutoGen (Microsoft)

-   **核心定位：** 专注于构建“多代理协作系统”。它允许你创建多个拥有不同角色、能力和目标的 Agent，让它们通过对话与合作来解决复杂任务。
-   **优点：** 强大的多代理协作机制、自动化的对话流、支持人类随时介入指导。它非常适合需要任务分解、角色扮演、代码生成与执行等复杂场景。

**我的选择：** AutoGen 对 Python 的依赖较重，考虑到我希望尽可能保持纯 NodeJS 技术栈，它首先被排除了。而 LlamaIndex 的核心优势在于 RAG，我的项目重心则更多地放在 Agent 的决策与执行流上。因此，这两位优秀的选手也被我暂时搁置。

现在，舞台中央只剩下两位主角：**LangChain** 和 **LangGraph**。

## ![graph_vs_chain](https://blog-1321748307.cos.ap-shanghai.myqcloud.com/img/4863ed8b-895d-41ca-9b61-e4c9ed36877a.png)

### 主角登场：LangGraph 与 LangChain 的终极对决

最早，LangChain 是 Agent 领域的绝对王者，几乎所有场景都能用它实现。但随后，LangGraph 登场了，很多人（包括我）都对这两个框架的设计理念和使用场景感到困惑。

今天，我们就用举例子的方式，让你体会两者设计理念的不同，选择时不再迷茫。

### 先聊聊 LangChain：你的全能工具箱 🧰

你可以把 LangChain.js 想象成一个**超级豪华的乐高工具箱**。

**它的设计哲学是：** “我为你提供所有必需的零件——模型接口、提示词模板、外部工具……你则像一个装配工，将它们‘咔嚓’一下拼接起来，一个任务流就完成了。”

它极其适合处理那种“一条道走到黑”的线性任务。

**优点：**

-   **上手神速：** 对于简单的想法，几行代码就能跑通，堪称原型开发神器。
-   **生态完备：** 社区极其庞大，你想要集成的各种模型、工具，基本上都有现成的模块。

**缺点：**

-   **天生“一根筋”，不会拐弯：** 如果你的任务需要“如果 A 失败了，就去尝试 B，如果 B 还不行，就退回到 A”，LangChain 本身是无法原生支持这种循环的。你必须在它的外部套上一个 `while` 循环，像个操心的老妈一样时刻监督和指挥它。

![LangChain](https://blog-1321748307.cos.ap-shanghai.myqcloud.com/img/langchain-toolbox.png)

### 再看看 LangGraph：给你的 Agent 装个“导航大脑” 🧠

如果说 LangChain 是工具箱，那 LangGraph.js 就是一张**自带导航和智能规划的流程图**。

**它的设计哲学是：** “别急着动手！先坐下来，把你 Agent 要做的所有事、所有可能的岔路口、所有需要‘返工’的环节，都清晰地画在一张图上。图画好了，你只需按下‘启动’，它自己就知道该怎么走，哪怕撞了南墙也知道该如何回头。”

**优点：**

-   **天生会“循环”和“决策”：** 像“自我修正”、“重试”、“条件分支”这类复杂逻辑，对它来说是基本功，因为图（Graph）结构本身就支持闭环和条件跳转。
-   **逻辑清晰，不易失控：** Agent 每一步在做什么，下一步要去哪，在图上都一目了然。一旦出问题，你一眼就能定位到是哪个节点卡住了，调试体验极佳。

**缺点：**

-   **前期准备稍显繁琐：** 你需要预先定义好节点（步骤）和边（流程），把整个图的结构设计出来。对于非常简单的任务来说，这有点“杀鸡用牛刀”。

## ![LangGraph](https://blog-1321748307.cos.ap-shanghai.myqcloud.com/img/langgraph-progress.png)

### 实战演练：让 Agent 写个笑话，不好笑就重写！

假设我们要让 Agent 写个笑话，并且**如果笑话不好笑，就必须重写**。看看这两种框架的“玩法”有何天壤之别。

#### LangChain 的玩法：我在场外当“监工”

```typescript
// LangChain 伪代码：我，应用代码，是总指挥

// 1. 搞一个只会写一次笑话的 Agent
const jokeAgent = createJokeAgent();

// 2. 我在外面用一个 while 循环控制它
let isFunny = false;
while (!isFunny) {
	// 让它干活！
	const aJoke = await jokeAgent.run("讲个笑话");

	// 我来判断它干得好不好
	isFunny = await checkIfJokeIsFunny(aJoke);

	if (!isFunny) {
		console.log("太尬了，重来！"); // 我来决定要不要循环
	}
}

console.log("嗯，这个可以，收工！");
```

**看出来了吗？** LangChain Agent 本身就像一个只会执行单次任务的“工具人”。循环、判断、重试的“大脑”逻辑，完全由我们这些外部代码来承担。

#### LangGraph 的玩法：Agent 自带“自我修养”

```typescript
// LangGraph 伪代码：图自己就是总指挥

// 1. 先定义好两个工作站（节点）
function writeJokeNode(state) {
	/* ...写笑话... */
}
function judgeJokeNode(state) {
	/* ...判断好不好笑... */
}

// 2. 画一张流程图
const jokeWorkflow = new Graph();
jokeWorkflow.addNode("writer", writeJokeNode); // “写作站”
jokeWorkflow.addNode("judge", judgeJokeNode); // “评审站”

// 3. 规定好流水线方向
jokeWorkflow.setEntryPoint("writer"); // 从“写作站”开始
jokeWorkflow.addEdge("writer", "judge"); // 写完送到“评审站”

// 4. 这就是魔法！设置一个“岔路口”
jokeWorkflow.addConditionalEdge("judge", (state) => {
	if (state.isFunny) {
		return END; // 好笑？下班！
	} else {
		return "writer"; // 不好笑？打回去重写！这就形成了循环
	}
});

// 5. 启动这条全自动流水线
const app = jokeWorkflow.compile();
await app.invoke({ input: "start" }); // 喊一嗓子“开工”，然后就不用管了
```

**感受到这种“自治”的魅力了吗？** 我们不再是那个手把手指挥的“监工”，而更像一个设计了自动化工厂流水线的“架构师”。整个“重写”的循环是在 Agent **内部**自动完成的。它从一个被动的“工具”，进化成了一个自主的“员工”。

---

### 结论：我该怎么选？一张图看懂

一句话总结：

-   **选 LangChain.js：** 如果你的任务像**一条直线**（例如：获取数据 -> 总结内容 -> 输出结果），或者你只是想快速验证一个想法。它简单、直接、高效。
-   **选 LangGraph.js：** 如果你的任务像个**带岔路的迷宫**（例如：写代码 -> 运行测试 -> 若不通过则调试 -> 再测试），需要 Agent 具备规划、反思和自我修正的复杂能力。

回到我最初的项目，虽然初期可能用不到太复杂的流程，但考虑到未来我要做的是一个多功能的 Electron 桌面 Agent，它很可能需要处理各种复杂的工作流。为了让它变得更“聪明”、更有韧性，从一开始就采用 LangGraph 的图状思维来构建，无疑是一个更具前瞻性、也更省心的选择。

希望这篇文章能帮你拨开迷雾，为你的下一个 AI Agent 项目做出最明智的选择！
