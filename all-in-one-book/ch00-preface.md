# 序言 — 全书地图

## Pi 是什么

Pi 是一套用 TypeScript 编写的开源工具集，用于**构建 AI Agent 并管理 LLM 部署**。它最广为人知的产品形态是一个终端里的交互式编码 Agent（coding agent）——你在终端输入一句自然语言指令，它调用大语言模型理解你的意图，然后自主地读文件、写代码、执行命令，直到任务完成。

但 Pi 不仅仅是一个 CLI 工具。它是一个**分层的 monorepo**，从底层的 LLM 通信协议到上层的终端渲染引擎，每一层都是独立的 npm 包，可以单独使用。你可以只用它的 LLM 通信层和三个不同的 Provider 对话，也可以用它的 Agent 运行时搭建自己的 AI 应用，或者直接用它的 coding agent 日常写代码。

这本书要讲的，是这整套系统**怎么实现的**。

---

## 架构全景图

Pi monorepo 包含 7 个包，它们之间的依赖关系构成了一棵清晰的树：

```
┌─────────────────────────────────────────────────────────────────┐
│                        应用层（产品形态）                          │
│                                                                 │
│   pi-coding-agent        pi-mom           pi-pods               │
│   (交互式编码 Agent)    (Slack 机器人)    (GPU 集群管理)           │
│         │                  │                 │                   │
│         ├──────────────────┤                 │                   │
│         ▼                  ▼                 │                   │
│   ┌──────────────────────────────┐           │                   │
│   │      pi-agent-core           │◄──────────┘                   │
│   │  (Agent 运行时：循环+工具+状态) │                              │
│   └──────────────┬───────────────┘                               │
│                  ▼                                               │
│   ┌──────────────────────────────┐                               │
│   │         pi-ai                │                               │
│   │  (统一 LLM API：流式+多Provider)│                             │
│   └──────────────────────────────┘                               │
│                                                                 │
│         pi-tui                    pi-web-ui                     │
│   (终端 UI 框架，独立)         (Web 聊天组件)                      │
│                                    │                             │
│                                    ├── 依赖 pi-ai（模型数据）      │
│                                    └── 依赖 pi-tui（部分复用）      │
└─────────────────────────────────────────────────────────────────┘
```

从下往上读这张图：

- **pi-ai** 是地基。它把 OpenAI、Anthropic、Google、Mistral、AWS Bedrock 等 10+ 个 LLM Provider 统一到一套流式 API 背后。所有上层包通过它与大模型通信。
- **pi-agent-core** 在 pi-ai 之上搭建了 Agent 运行时——一个"LLM 思考 → 调用工具 → 拿到结果 → 继续思考"的循环引擎。
- **pi-coding-agent** 是面向用户的产品。它在 agent-core 之上加了 CLI 解析、会话持久化、扩展系统、交互式 TUI，构成了完整的编码 Agent。
- **pi-tui** 是独立的终端 UI 框架，提供差分渲染、组件模型、键盘处理。coding-agent 的交互模式用它来渲染界面。
- **pi-web-ui** 是一套 Web Components，用于构建浏览器端的 AI 聊天界面。
- **pi-mom** 是一个 Slack 机器人，把 Slack 消息委托给 coding-agent 处理。
- **pi-pods** 是管理 GPU 服务器上 vLLM 部署的 CLI 工具。

**本书的主线**是中间那条纵向链路：pi-ai → pi-agent-core → pi-coding-agent。这是 Pi 的骨架。tui、web-ui、mom、pods 是围绕骨架生长的四肢——我们也会讲，但优先级排在主线之后。

---

## 核心概念词典

在深入任何一个模块之前，先认识这些贯穿全书的核心概念。每个概念在后续章节会详细展开，这里只需要建立初步印象。

### LLM 通信层的概念（pi-ai）

| 概念 | 解释 |
|------|------|
| **Provider** | LLM 服务提供商。Pi 把 OpenAI、Anthropic 等各家 API 的差异藏在统一接口后面。 |
| **Model** | 一个具体的模型配置——包含模型 ID、所属 Provider、API 类型、上下文窗口大小、费用信息等。不是一个"空洞的名字"，而是一个带完整元数据的对象。 |
| **Context** | 发送给 LLM 的全部输入——系统提示词（system prompt）+ 消息历史 + 可用工具列表。 |
| **Message** | 对话中的一条消息。有三种角色：`user`（用户说的）、`assistant`（LLM 回复的）、`toolResult`（工具执行结果）。 |
| **AssistantMessageEventStream** | Pi 定义的流式响应容器。LLM 的回复不是一次性返回的，而是一个 token 一个 token 地"流"出来。这个对象既是 AsyncIterator（可以 `for await` 逐事件消费），又是 Promise（可以 `await` 拿最终结果）。 |
| **transformMessages** | 跨 Provider 消息转换函数。不同 Provider 对工具调用 ID 的格式、思考块的处理方式各不相同，这个函数负责归一化。 |

### Agent 运行时的概念（pi-agent-core）

| 概念 | 解释 |
|------|------|
| **Agent Loop（Agent 循环）** | Agent 的核心运行逻辑：发消息给 LLM → LLM 回复（可能包含工具调用）→ 执行工具 → 把结果发回 LLM → 重复，直到 LLM 不再调用工具。 |
| **AgentTool** | Agent 可以调用的工具。每个工具有名字、描述、参数 schema、执行函数。LLM 通过 tool calling 能力来决定调用哪个工具。 |
| **Steering（转向）** | 在 Agent 运行过程中插入新指令。类似"打断"——但不是立即打断，而是在当前工具轮次结束后注入新的用户消息。 |
| **Follow-up（后续）** | 在 Agent 完成当前任务后追加新任务。消息会排队，等 Agent 空闲后再处理。 |

### 编码 Agent 的概念（pi-coding-agent）

| 概念 | 解释 |
|------|------|
| **AgentSession** | coding-agent 的核心抽象。它包装了 Agent 实例，加上了会话持久化、扩展集成、自动压缩等能力。一次 `pi` 命令执行就是一个 AgentSession 的生命周期。 |
| **Session（会话）** | 用 JSONL 文件存储的对话历史。每条记录有 `id` 和 `parentId`，构成**树结构**——支持分支、回溯，不只是线性历史。 |
| **Compaction（压缩）** | 当对话历史太长、即将超出 LLM 的上下文窗口时，用 LLM 总结旧消息来"腾出空间"。原始历史仍保存在 JSONL 文件中。 |
| **Extension（扩展）** | 插件系统。TypeScript 模块通过事件钩子和 API 注册来扩展 Agent 的能力——添加新工具、拦截工具调用、修改系统提示词等。 |
| **Skill（技能）** | 遵循 Agent Skills 规范的 Markdown 文件，为 Agent 提供特定领域的知识和操作指南。比 Extension 更轻量——不需要写代码，只需要写 Markdown。 |

---

## 代码库地图

```
pi-mono/
├── packages/
│   ├── ai/                          ← LLM 通信层
│   │   └── src/
│   │       ├── types.ts             ← 所有核心类型定义
│   │       ├── stream.ts            ← stream() / complete() 入口
│   │       ├── models.ts            ← getModel() 模型注册表
│   │       ├── api-registry.ts      ← Provider 注册中心
│   │       ├── providers/           ← 各 Provider 实现
│   │       │   ├── anthropic.ts
│   │       │   ├── openai-completions.ts
│   │       │   ├── google.ts
│   │       │   ├── transform-messages.ts  ← 跨 Provider 消息转换
│   │       │   └── register-builtins.ts   ← Provider 延迟注册
│   │       └── utils/               ← 工具函数
│   │           └── event-stream.ts  ← EventStream 实现
│   │
│   ├── agent/                       ← Agent 运行时
│   │   └── src/
│   │       ├── agent.ts             ← Agent 类（高层封装）
│   │       ├── agent-loop.ts        ← agentLoop()（核心循环）
│   │       ├── types.ts             ← AgentEvent、AgentTool 等类型
│   │       └── proxy.ts             ← 浏览器代理模式
│   │
│   ├── coding-agent/                ← 编码 Agent CLI
│   │   └── src/
│   │       ├── cli.ts               ← 进程入口
│   │       ├── main.ts              ← 主编排逻辑
│   │       ├── cli/                 ← 参数解析、配置选择
│   │       ├── core/
│   │       │   ├── agent-session.ts ← AgentSession 核心抽象
│   │       │   ├── session-manager.ts ← JSONL 会话持久化
│   │       │   ├── model-resolver.ts  ← 模型选择逻辑
│   │       │   ├── extensions/      ← 扩展加载与运行
│   │       │   ├── tools/           ← 7 个内置工具
│   │       │   ├── compaction/      ← 上下文压缩
│   │       │   └── sdk.ts           ← 编程接口
│   │       └── modes/
│   │           ├── interactive/     ← TUI 交互模式
│   │           ├── rpc/             ← JSON-line 无头模式
│   │           └── print-mode.ts    ← 纯文本输出模式
│   │
│   ├── tui/                         ← 终端 UI 框架
│   │   └── src/
│   │       ├── tui.ts              ← 差分渲染核心
│   │       ├── terminal.ts         ← 终端抽象
│   │       └── components/         ← 内置组件
│   │
│   ├── web-ui/                      ← Web 聊天组件
│   │   └── src/
│   │       ├── ChatPanel.ts        ← 主聊天面板
│   │       ├── components/         ← 消息、输入、沙箱等
│   │       ├── storage/            ← IndexedDB 存储
│   │       └── tools/              ← JS REPL、文档提取
│   │
│   ├── mom/                         ← Slack 机器人
│   │   └── src/
│   │       ├── main.ts             ← 入口
│   │       ├── agent.ts            ← Agent 运行器
│   │       ├── slack.ts            ← Slack 适配器
│   │       └── tools/              ← 沙箱工具
│   │
│   └── pods/                        ← GPU Pod 管理
│       └── src/
│           ├── cli.ts              ← 命令分发
│           ├── commands/           ← 各子命令
│           └── model-configs.ts    ← 模型配置库
│
├── package.json                     ← monorepo 根配置
├── tsconfig.base.json              ← 共享 TypeScript 配置
└── biome.json                      ← 代码风格配置
```

---

## 一次典型交互的极简全流程

在深入任何模块之前，先用最粗的粒度看一遍"用户输入一句话，到 Agent 完成任务"的全过程。后续每一章都在展开这个流程的某个环节。

**场景**：用户在终端执行 `pi "把 README.md 里的拼写错误修一下"`。

```
用户输入
  │
  ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. CLI 启动                                                  │
│    cli.ts 设置 HTTP 代理 → 调用 main()                        │
│    main.ts 解析参数 → 解析模型 → 创建 AgentSession              │
│    AgentSession 加载会话/扩展/工具 → 进入交互模式                 │
└────────────────────────┬────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Agent 循环开始                                             │
│    AgentSession.prompt("把 README.md 里的拼写错误修一下")        │
│      → Agent.prompt() → agentLoop()                          │
│                                                              │
│    agentLoop 构建 Context:                                    │
│      systemPrompt + messages + tools                          │
│      → convertToLlm() 过滤非 LLM 消息                         │
│      → 调用 pi-ai 的 streamSimple()                           │
└────────────────────────┬────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. LLM 通信（pi-ai）                                         │
│    streamSimple() → resolveApiProvider() → provider.stream()  │
│    Provider 实现（如 Anthropic）：                              │
│      transformMessages() 归一化消息                             │
│      调用 Anthropic SDK 发起流式请求                            │
│      逐 chunk 解析 → 推送 AssistantMessageEvent                │
│                                                              │
│    LLM 回复："我需要先读取 README.md 的内容。"                    │
│    + 工具调用：read({ path: "README.md" })                    │
└────────────────────────┬────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. 工具执行                                                   │
│    agentLoop 检测到 toolCall → prepareToolCall()              │
│      验证参数 → beforeToolCall() 钩子                          │
│      → tool.execute() 读取文件                                │
│      → afterToolCall() 钩子                                   │
│    生成 ToolResultMessage（文件内容）                            │
│    → 追加到消息历史，回到步骤 2 重新调用 LLM                      │
└────────────────────────┬────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. LLM 再次响应                                               │
│    这次 LLM 看到了文件内容，发现拼写错误                          │
│    回复：调用 edit({ path: "README.md", old: "...", new: "..." })│
│    → 工具执行 → 文件被修改                                      │
│    → 再次调用 LLM                                              │
│                                                              │
│    LLM 回复："已修复 README.md 中的拼写错误。"                    │
│    这次没有工具调用 → stopReason: "stop"                        │
│    → Agent 循环结束                                            │
└────────────────────────┬────────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. 结果呈现                                                   │
│    AgentSession 收到 agent_end 事件                            │
│    → 会话保存到 JSONL 文件                                      │
│    → TUI 渲染最终的助手消息                                     │
│    用户看到修复完成的反馈                                        │
└─────────────────────────────────────────────────────────────┘
```

这就是整个过程。六个步骤，三个核心组件参与：**pi-ai** 负责与 LLM 通信，**pi-agent-core** 负责编排循环和工具执行，**pi-coding-agent** 负责 CLI 解析、会话管理和 UI 呈现。

接下来的章节会按照"从底层积木到上层产品"的顺序，逐个打开这些黑盒。下一章我们先用一个更详细的视角重走这个流程——追踪每一步数据的形态变化。

---

### 质检报告

**讲解节奏**
- [x] 每个模块先讲"它是什么"再讲"里面有什么"
- [x] 没有上来就讲实现的情况

**周边知识**
- [x] 设计决策处是否提供了足够的背景（序言层面不涉及具体设计决策）
- [x] 是否有段落间跨度过大、读者可能缺前置知识的地方（概念词典提供了所需前置知识）

**讲透了吗**
- [x] 核心流程每一步解释了数据变化（极简全流程覆盖了关键步骤）
- [x] 没有跳步
- [x] 复杂节点已标注"后续章节展开"

**过渡自然吗**
- [x] 章头：作为序言，直接开篇
- [x] 章尾引出下一章（"下一章我们先用更详细的视角重走这个流程"）
- [x] 章内小节之间有衔接（从"是什么"→"架构"→"概念"→"代码地图"→"流程"层层递进）

**准确吗**
- [x] 行业标准术语使用正确
- [x] 项目特有术语已在概念词典中解释
- [x] 未确认项：无（均已通过源码验证）

**读得下去吗**
- [x] 术语首次出现有解释
- [x] 每张图有文字讲解

**勘误建议**
无
