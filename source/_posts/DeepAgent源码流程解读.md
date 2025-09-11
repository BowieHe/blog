---
title: DeepAgent源码流程解读
date: 2025-09-10 17:23:25
categories:
    - 教程
tags:
    - LangGraph
comments: true
description: 深探DeepAgent的源码实现，揭示其如何包装LangGraph的创建反应智能体，提供更可控的任务执行框架。
keywords:
top_image: https://blog-1321748307.cos.ap-shanghai.myqcloud.com/img/DeepAgent-summary.png
cover: https://blog-1321748307.cos.ap-shanghai.myqcloud.com/img/DeepAgent-summary.png
---

## DeepAgents. Js 架构与流程

这篇就是带你了解一边 DeepAgents. Js 的底层实现：你可以把它理解成：在 LangGraph 的 createReactAgent 上，包了一层更好用、更可控的 Agent 工厂，顺手送你一套常用工具、子代理机制（task 工具）、以及“Human-in-Loop”的中断审批。

![](https://blog-1321748307.cos.ap-shanghai.myqcloud.com/img/DeepAgent-summary.png)

## 小科普：LangGraph Node、ReAct、DeepAgent 分别是什么? 怎么区分和选择

| 特性     | Node（基元）                | ReAct（智能体模式）        | DeepAgent（工程化套件）        |
| -------- | --------------------------- | -------------------------- | ------------------------------ |
| 定义     | 图里的单步函数              | 先思考再行动，动态调用工具 | ReAct 的工程化封装             |
| 控制流   | 静态、固定流程              | 动态选择工具，多轮循环     | 动态 + 审批/白名单可控         |
| 粒度     | 最小步骤                    | 控制器，负责多步任务       | ReAct + 工具箱/子代理/审批     |
| 上手成本 | 低（要自己编排）            | 中等                       | 最快（默认电池齐全）           |
| 可控性   | 最强（全手写）              | 需要额外约束               | 内置 interrupt 和状态通道增强  |
| 适用场景 | ETL、固定流水线、结构化处理 | 搜索-阅读-总结等开放式任务 | 产品化落地，需安全、审计、协作 |

设计理念一句话：

-   Node：显式可控，工程化流水线
-   ReAct：让模型“在行动中推理”
-   DeepAgent：让 ReAct 真正可用、可控、可审计、可落地

---

好, 上面大概梳理和介绍了一下 Node, ReAct 和 DeepAgent 的区别, 下面从 DeepAgent 的实现来介绍一下是怎么实现的

![](https://blog-1321748307.cos.ap-shanghai.myqcloud.com/img/DeepAgent-createAgent.png)

## 运行流程（配伪代码更直观）

### 1) 主入口：`createDeepAgent`（见 `graph.ts`）

思路很简单：先拼“工具”和“提示词”，再决定要不要放 `task` 工具（子代理），最后挂上中断钩子（或自定义 postModelHook），然后交给 LangGraph。
其中 DeepAgent 默认的 model 是 Claude sonnet 4, 如果不需要可以自己设置

```TypeScript
function createDeepAgent(params = {}) {
	const stateSchema = params.stateSchema

	// 1) 选内置工具（可用 builtinTools 白名单筛）+ 合并外部工具
	let allTools = [BUILTIN_TOOLS, ...params.tools]

	// 2) 如配置了 subagents，就注入一个 `task` 工具
	if (params.subagents.length > 0) {
		const taskTool = createTaskTool({ subagents: params.subagents, tools: toolsMap, model, stateSchema })
		allTools = [...allTools, taskTool]
	}

	// 3) 拼提示词（instructions + BASE_PROMPT）
	const messageModifier = params.instructions + BASE_PROMPT

	// 4) 中断/后处理二选一
	if (params.postModelHook && params.interruptConfig) throw Error("mutually exclusive")
	const postModelHook = params.postModelHook ?? (
		params.interruptConfig ? createInterruptHook(params.interruptConfig) : undefined
	)

	// 5) 交给 LangGraph
	return createReactAgent({ llm: model, tools: allTools, stateSchema, messageModifier, contextSchema: params.contextSchema, postModelHook })
}
```

你可以把它想成“乐高组装”：把零件（工具、提示、状态、钩子）按需装一装，最后交给 LangGraph 开跑。
从 DeepAgent 创建也可以看出和上面所说的, 实际返回的对象也是一个 ReAct node

---

### 2) 工具体系：`tools.ts`（内置工具都很“规矩”）

![](https://blog-1321748307.cos.ap-shanghai.myqcloud.com/img/DeepAgent-tool.png)

下面我们先介绍一下里面的预置的工具.
这里借鉴了 Claude Code 的 Todo 思路：把任务与中间产物都“文件化”。本库并不真的写磁盘，而是把一切放在内存里的“模拟文件系统”（`state.files`），通过 `Command.update` 来更新。为什么要用“文件式”抽象？有什么好处？

**统一抽象，心智简单**：- 不同工具产出的内容（计划、结果、临时数据）都变成“路径 → 文本”的键值对，行为可预测、接口一致。
**可审计、可回溯**：- 每次写入都会伴随一条 `ToolMessage`；再配合中断（interrupt），关键写操作可以先审批。- 自然支持“看差异”（diff 思维）和变更历史记录，方便代码评审式的审计。
**并发与合并更稳**：- 通过 reducer（`files` 合并、`todos` 覆盖）定义清晰的合并规则，适合多工具并行、反复迭代的场景。
**快照与回滚容易**：- 整个 `files` 就是状态快照，一把抓；需要回滚直接恢复上一份即可，做“时光旅行”很自然。
**低耦合、易组合**：- 工具之间只约定“读哪个路径、写哪个路径”，无需共享复杂对象结构，替换/扩展工具更轻松。
**更好测试体验**：- 单测无需真实磁盘，直接构造/断言 `state.files`；行为确定、可重复，CI 也更快更稳。- 随时 `ls` + `read_file`，就能把当前“上下文现场”打印给人看，排查快。
**更贴合 LLM 的工具心智**：- `read_file` / `edit_file` 这类能力和提示非常直观；还能按 offset/limit 分块读取，节省上下文。

需要注意的地方：

-   这只是“内存文件系统”：进程重启会丢失；如需长期保存，要自己把 `files` 同步到真实存储。
-   路径命名需要约定（按功能/任务分目录），否则容易乱。
-   超长文本会有行截断（每行 2000 字符），大规模文档场景要考虑分片或外部存储。

---

下面看怎么实现 sub-task 的

![](https://blog-1321748307.cos.ap-shanghai.myqcloud.com/img/deepagent-subagent.png)

### 3) 子代理：`createTaskTool`（见 `subAgent.ts`）

`task` 的核心是把“调用一个子代理一次”封装成一个工具（类似 ToolNode）：

-   预创建 subAgent：启动时先为每个 subAgent 生成一个独立的 ReAct Agent（带它自己的提示词与工具集）。
-   包装成可调用节点：对外只暴露一个名为 `task` 的工具，入参包含 `subagent_type`（用哪个子代理）和 `description`（要做什么）。
-   运行时交互：主 Agent 调用 `task` 时，会把 `description` 作为一条用户消息投递给对应的子代理；子代理独立执行后，产出的结果会以“文件变更（files）+ 一条工具消息（ToolMessage）”的形式回填到主状态。

这套设计的巧妙点：

-   动态“接子图”而不是画条件边：传统 conditional edge 要在图构建期把所有可能路径画好；这里把“去另一个子图做事”抽象成工具调用，模型在运行时按需选择哪个子代理、调用几次、是否并行，更像“动态图按需生成”。
-   统一心智：子代理也是“工具的一种”，和普通工具共享同样的调用与观测接口（参数描述、消息回填、权限白名单等）。

与 ToDo（任务面板）的衔接：

-   计划：用 `write_todos` 把复杂需求拆成可执行的 todo（pending）。
-   派工：把某条 todo 设为 `in_progress`，并将其描述/执行说明作为 `description` 交给 `task`，通过 `subagent_type` 指定合适的子代理。
-   回填：子代理完成后把结果写入 `files`，并在消息里产出总结；主 Agent 随后把该 todo 标为 `completed`，必要时追加新的 follow-up todo。
-   双向赋能：主 Agent 与子代理都可以（如果被授予）使用 `write_todos`，在执行过程中细化/拆分任务，形成“计划 → 派工 → 产出 → 回填 → 更新计划”的闭环。

一句话：`task` 就像“分包商调度器 + 子图插槽”，让主图在运行时把合适的子代理接上去干活，做完再把结果规范地回流到同一套状态里。

极简伪代码（从实现提炼，辅助理解）：

```TypeScript
// 1) 预创建子代理（带各自工具与提示）
allTools = { ...BUILTIN_TOOLS, ...userTools }
agentsMap = {}
for each a in subagents:
	agentsMap[a.name] = createReactAgent({ llm: model, tools: toolset, stateSchema, messageModifier: a.prompt })

// 2) task 工具主体
task({ description, subagent_type }, config):
	agent = agentsMap[subagent_type]
	if !agent: return "Agent not found"
	current = getCurrentTaskInput()
	next = { ...current, messages: [{ role: 'user', content: description }] }
	result = agent.invoke(next, config)
	return Command.update({
        files: result.files ?? {},
		messages: [ToolMessage(content = last(result.messages)?.content ?? 'Task completed')] })
```

![](https://blog-1321748307.cos.ap-shanghai.myqcloud.com/img/deepagent-human-in-loop.png)

### 4) 中断/审批：`createInterruptHook`（见 `interrupt.ts`）

这个钩子把“审批/等待/中断等行为”接进来：当 LLM 打算用某个被标记为需审批的工具时，先打个断点问你要不要放行、要不要改参数、还是直接回复。直白说，它就是“工具要上车先验票”。

使用方法（最小示例）：

```TypeScript
const agent = createDeepAgent({
	interruptConfig: {
		// 为 edit_file 开启审批，并允许三种操作
		edit_file: {
			allow_accept: true,
			allow_edit: true,
			allow_respond: true,
		},
		// 也可直接写 true，表示用默认配置（允许 accept / edit / respond）
		write_file: true,
	},
});
```

配置怎么写？

-   形态：`Record<toolName, HumanInterruptConfig | boolean>`。
-   写 `true`：使用默认配置 `{ allow_accept: true, allow_edit: true, allow_respond: true, allow_ignore: false }`。
-   写对象：自定义人类可用的操作按钮（`allow_accept/allow_edit/allow_respond`）。
-   不支持：`allow_ignore`（写了会报错）。
-   想禁用某工具的审批：不要把它写进 `interruptConfig`（写 `false` 也会被视为开启，因实现细节会走默认配置）。

触发条件 & 结果是什么？

-   条件：当模型产出了 `tool_calls`，且其中某个工具名出现在 `interruptConfig` 里。
-   结果（人类可做三选一）：
    -   Accept：原样放行这个工具调用，继续执行。
    -   edit：人类可修改 `action/args`，用修改后的参数执行工具。
    -   respond：不执行工具，直接把人类填写的文本作为 `ToolMessage` 返回给模型（等于“人工回复”）。
-   其他同时产出的工具调用（不在 `interruptConfig` 内的）会自动放行。

适合什么时候开审批？

-   有副作用/破坏性操作：比如 `edit_file`、对关键“文件路径”的 `write_file`。
-   成本较高或有风险的外部请求：如批量写、删除、或者会触发外部系统变更的工具。
-   合规/流程要求必须留痕并二次确认的步骤。

限制与注意事项：

-   一次仅支持“至多一个”需要审批的工具调用；多个会报错（当前版本约束）。
-   `postModelHook` 与 `interruptConfig` 互斥（只能二选一）。
-   审批弹窗内容包含：工具名、参数 JSON 和可用操作按钮；你可以通过可选的 `messagePrefix`（默认“Tool execution requires approval”）自定义提示开头。

小技巧：

-   只在关键工具上开启审批，降低交互打断频率（例如只对 `edit_file`、`write_file` 开启）。
-   若你希望“只允许人工回复，不允许直接执行”，就把 `allow_accept` 设为 false、保留 `allow_respond` 即可（在自定义对象里配置）。
-   审批提示里尽量让参数清晰易读（例如在生成调用时就保证 args 是干净的结构化对象），人类更容易做判断。

## 上手一眼明白的小例子

```TypeScript
const agent = createDeepAgent({
	builtinTools: ['write_todos', 'read_file', 'edit_file'],
	instructions: '你是一个谨慎靠谱的代码助手。',
	subagents: [
		{
			name: 'research-analyst',
			description: '做深度检索和资料梳理',
			prompt: '请给出可溯源的研究结论',
			tools: ['read_file', 'write_todos']
		}
	],
	interruptConfig: {
		edit_file: { allow_accept: true, allow_edit: true, allow_respond: true }
	}
})

// 后面直接：agent.invoke({ messages: [{ role: 'user', content: '...' }] })
```

## 典型一次对话会怎么走

1. 用户说话 -> 进 `messages`。
2. LLM 思考 -> 可能产出 `tool_calls`。
3. 命中审批 -> 先问你（放行/改参数/直接回）。
4. 真正执行工具 -> 可能用 Command 改 `todos/files/messages`。
5. 用了 `task` -> 子代理干活 -> `ToolMessage` 把结果带回主线。
6. 基于最新 `messages` 继续推理 -> 给出最终回答。

## 最后一句话

把它当“拼装式 Agent 工厂”就对了：

-   `createDeepAgent` 负责编排；
-   `createTaskTool` 负责子代理派工；
-   内置工具让过程可观察；
-   中断钩子让关键操作更可控。

先从默认配置跑起来，然后按需加子代理、收紧工具权限、打开审批、扩展状态，你的深度代理就有了清晰的骨架和肌肉。
