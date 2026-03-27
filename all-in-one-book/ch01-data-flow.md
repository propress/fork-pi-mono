# 第一章 数据流全景 — 一次完整交互的生命周期

上一章我们用最粗的粒度走了一遍"用户输入 → Agent 完成任务"的全过程。这一章我们把放大镜对准数据——在这个过程中，**数据长什么样、经过了什么变换、变成了什么**。

我们将追踪一个具体场景：用户执行 `pi "读取 package.json 并告诉我版本号"`。这是一个只需要一次工具调用的简单场景，刚好能展示完整的数据流而不至于太复杂。

---

## 1.1 起点：用户的一句话

用户在终端输入的字符串 `"读取 package.json 并告诉我版本号"` 是原始输入。它首先经过 CLI 参数解析（详见第 4 章），最终到达 `AgentSession.prompt()` 方法时，变成了一个函数调用参数：

```typescript
agentSession.prompt("读取 package.json 并告诉我版本号")
```

AgentSession 内部将这个字符串传递给 `Agent.prompt()`。在 Agent 类中，字符串被包装成一条标准的 **UserMessage**：

```typescript
// packages/agent/src/agent.ts — prompt() 方法内部
const userMessage: UserMessage = {
  role: "user",
  content: [{ type: "text", text: "读取 package.json 并告诉我版本号" }],
  timestamp: 1711567820628   // Date.now()
};
```

这里发生了第一次数据变换：**字符串 → 结构化的 UserMessage 对象**。

`UserMessage` 是 Pi 消息系统的三种基本消息类型之一。来看看完整的消息类型体系：

```typescript
// packages/ai/src/types.ts
type Message = UserMessage | AssistantMessage | ToolResultMessage;

interface UserMessage {
  role: "user";
  content: string | (TextContent | ImageContent)[];  // 文本或混合内容
  timestamp: number;
}

interface AssistantMessage {
  role: "assistant";
  content: (TextContent | ThinkingContent | ToolCall)[];  // 文本、思考、工具调用
  api: Api;              // 使用的 API 类型
  provider: Provider;    // Provider 名
  model: string;         // 模型 ID
  usage: Usage;          // token 用量和费用
  stopReason: StopReason;
  timestamp: number;
}

interface ToolResultMessage {
  role: "toolResult";
  toolCallId: string;    // 对应哪个工具调用
  toolName: string;
  content: (TextContent | ImageContent)[];
  isError: boolean;
  timestamp: number;
}
```

注意 `AssistantMessage` 的 `content` 数组可以包含三种内容：**文本**（LLM 的自然语言回复）、**思考**（推理过程，某些模型支持）、**工具调用**（LLM 请求执行某个工具）。这个设计让一条助手消息可以同时包含"我的想法"和"我要调用的工具"。

---

## 1.2 消息进入 Agent 循环

`Agent.prompt()` 拿到 UserMessage 后，调用内部的 `_runLoop()` 方法，启动 **Agent 循环**（agent loop）。

Agent 循环是整个系统的心脏。它的逻辑可以用一句话概括：**不停地问 LLM，直到 LLM 说"我做完了"**。具体来说：

```
发送消息给 LLM ──→ 收到回复 ──→ 回复里有工具调用吗？
                                     │
                            ┌────────┴────────┐
                            │ 有               │ 没有
                            ▼                  ▼
                    执行工具，把结果        循环结束，
                    追加到消息历史，        返回最终回复
                    回到开头重新发送
```

但在发送给 LLM 之前，需要做一步关键的**数据变换**——构建 LLM 能理解的 Context。

---

## 1.3 构建 Context：从 AgentMessage 到 LLM 消息

Agent 内部维护的消息历史不只有 `UserMessage`、`AssistantMessage`、`ToolResultMessage` 这三种 LLM 原生消息。coding-agent 还定义了自己的自定义消息类型：

```typescript
// packages/coding-agent/src/core/messages.ts
interface BashExecutionMessage {        // Bash 命令执行记录
  role: "bashExecution";
  command: string;
  output: string;
  exitCode: number | undefined;
  timestamp: number;
}

interface CompactionSummaryMessage {    // 上下文压缩摘要
  role: "compactionSummary";
  summary: string;
  tokensBefore: number;
  timestamp: number;
}

interface BranchSummaryMessage {        // 分支跳转摘要
  role: "branchSummary";
  summary: string;
  fromId: string;
  timestamp: number;
}
```

这些自定义消息对 UI 展示很有价值（比如在终端里渲染 Bash 输出），但 LLM 不认识 `bashExecution` 这种角色。所以在发送给 LLM 之前，需要做一次转换。

这个转换分两步：

**第一步：`transformContext()`**（可选）—— 对消息列表做任意预处理，比如注入额外上下文、裁剪过长的消息。

**第二步：`convertToLlm()`**（必须）—— 把所有消息转换成 LLM 能理解的三种标准类型。

```typescript
// packages/agent/src/agent-loop.ts — streamAssistantResponse() 内部

// 第一步：可选的上下文变换
let messages = context.messages;
if (config.transformContext) {
  messages = await config.transformContext(messages, signal);
}

// 第二步：转换为 LLM 消息
const llmMessages = await config.convertToLlm(messages);

// 构建最终的 LLM Context
const llmContext: Context = {
  systemPrompt: context.systemPrompt,
  messages: llmMessages,
  tools: context.tools,
};
```

在我们的场景中，消息历史只有一条 UserMessage，没有自定义消息，所以 `convertToLlm()` 基本原样通过。最终构建出的 **Context** 对象长这样：

```typescript
const context: Context = {
  systemPrompt: "You are an interactive coding agent...",  // 系统提示词
  messages: [
    {
      role: "user",
      content: [{ type: "text", text: "读取 package.json 并告诉我版本号" }],
      timestamp: 1711567820628
    }
  ],
  tools: [
    {
      name: "read",
      description: "Read the contents of a file...",
      parameters: { /* TypeBox schema */ }
    },
    {
      name: "bash",
      description: "Execute a bash command...",
      parameters: { /* TypeBox schema */ }
    },
    {
      name: "edit",
      description: "Edit a file...",
      parameters: { /* TypeBox schema */ }
    },
    {
      name: "write",
      description: "Write content to a file...",
      parameters: { /* TypeBox schema */ }
    }
  ]
};
```

这就是发送给 LLM 的全部输入。它包含三个部分：**系统提示词**告诉 LLM 它的角色和行为准则；**消息历史**是到目前为止的对话；**工具列表**告诉 LLM 有哪些工具可用、每个工具的参数格式。

---

## 1.4 进入 pi-ai：流式请求 LLM

Context 构建好后，Agent 循环调用 pi-ai 的 `streamSimple()` 函数：

```typescript
// packages/ai/src/stream.ts
function streamSimple(model, context, options): AssistantMessageEventStream {
  const provider = resolveApiProvider(model.api);
  return provider.streamSimple(model, context, options);
}
```

这个函数做了两件事：

1. **查找 Provider**：根据 Model 的 `api` 字段（比如 `"anthropic-messages"`）从注册表中找到对应的 Provider 实现。
2. **委托给 Provider**：调用该 Provider 的 `streamSimple()` 方法。

返回的是一个 `AssistantMessageEventStream`——这是 Pi 设计的核心数据结构之一。它**既是 AsyncIterator 又是 Promise**：你可以 `for await` 逐个消费流式事件，也可以 `await stream.result()` 直接等最终结果。

以 Anthropic 为例，Provider 内部发生了这些事：

```
Context (Pi 格式)
    │
    ▼
transformMessages()          ← 消息格式归一化
    │                           - 工具调用 ID 标准化
    │                           - 思考块跨 Provider 兼容
    │                           - 补齐缺失的工具结果
    ▼
Anthropic SDK 格式的请求
    │
    ▼
发送 HTTP 流式请求到 Anthropic API
    │
    ▼
逐 chunk 解析响应
    │
    ▼
推送 AssistantMessageEvent 到 EventStream
```

`transformMessages()` 是一个容易被忽视但极其重要的函数。不同 Provider 对消息格式有不同的要求——比如 OpenAI 的工具调用 ID 可以有 450+ 个字符和特殊字符，但 Anthropic 要求 ID 最多 64 个字符且只允许字母数字。`transformMessages()` 负责把消息归一化到目标 Provider 能接受的格式。我们在第 2 章会详细展开。

---

## 1.5 流式事件：LLM 的回复一个 token 一个 token 地到来

LLM 的回复不是一次性返回的，而是以**流式事件**（streaming events）的方式逐步到达。Pi 定义了一套标准化的事件协议：

```typescript
type AssistantMessageEvent =
  | { type: "start"; partial: AssistantMessage }
  | { type: "text_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "text_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "text_end"; contentIndex: number; content: string; partial: AssistantMessage }
  | { type: "thinking_start"; ... }
  | { type: "thinking_delta"; ... }
  | { type: "thinking_end"; ... }
  | { type: "toolcall_start"; ... }
  | { type: "toolcall_delta"; contentIndex: number; delta: string; ... }
  | { type: "toolcall_end"; contentIndex: number; toolCall: ToolCall; ... }
  | { type: "done"; reason: StopReason; message: AssistantMessage }
  | { type: "error"; reason: StopReason; error: AssistantMessage };
```

为什么需要流式？因为 LLM 生成回复可能需要几秒甚至几十秒，如果等全部生成完再返回，用户会感到界面"卡住了"。流式让前端可以逐 token 渲染，用户能实时看到 LLM 在"打字"。

在我们的场景中，LLM 的回复包含两部分：一段文本（"我来读取 package.json"）和一个工具调用（`read`）。事件序列大致如下：

```
event: { type: "start", partial: { role: "assistant", content: [], ... } }
  │
  ▼
event: { type: "text_start", contentIndex: 0 }
event: { type: "text_delta", contentIndex: 0, delta: "我来" }
event: { type: "text_delta", contentIndex: 0, delta: "读取" }
event: { type: "text_delta", contentIndex: 0, delta: " package.json" }
event: { type: "text_end", contentIndex: 0, content: "我来读取 package.json" }
  │
  ▼
event: { type: "toolcall_start", contentIndex: 1 }
event: { type: "toolcall_delta", contentIndex: 1, delta: '{"path":' }
event: { type: "toolcall_delta", contentIndex: 1, delta: '"package.json"}' }
event: { type: "toolcall_end", contentIndex: 1, toolCall: {
    type: "toolCall",
    id: "toolu_01ABC",
    name: "read",
    arguments: { path: "package.json" }
  }
}
  │
  ▼
event: { type: "done", reason: "toolUse", message: { /* 完整的 AssistantMessage */ } }
```

注意最终的 `done` 事件的 `reason` 是 `"toolUse"` 而不是 `"stop"`——这告诉 Agent 循环：LLM 没有说完，它需要先执行工具调用。

每个事件都携带一个 `partial` 字段，它是**当前时刻**的 AssistantMessage 快照。随着新的 delta 到达，快照不断更新。这个设计让消费者随时都能拿到"到目前为止的完整消息"，而不需要自己拼接 delta。

---

## 1.6 Agent 循环接收事件并追踪状态

回到 Agent 循环。`streamAssistantResponse()` 函数消费这些流式事件，同时做两件事：

**一、维护消息历史**：收到 `start` 事件时，把 partial AssistantMessage 追加到 `context.messages` 数组。后续的 delta 事件更新数组中的最后一条消息。

**二、发射 AgentEvent**：把 pi-ai 的事件包装成 Agent 层的事件，通知上层订阅者（UI、会话管理器等）。

```typescript
// 简化的事件流转
pi-ai event: "start"        → AgentEvent: "message_start"
pi-ai event: "text_delta"   → AgentEvent: "message_update" (包含原始 pi-ai event)
pi-ai event: "done"         → AgentEvent: "message_end"
```

当 `streamAssistantResponse()` 返回时，我们拿到了完整的 AssistantMessage。此时 `context.messages` 数组变成了：

```typescript
[
  { role: "user", content: [...], timestamp: ... },           // 用户消息
  { role: "assistant", content: [                             // LLM 回复
      { type: "text", text: "我来读取 package.json" },
      { type: "toolCall", id: "toolu_01ABC", name: "read",
        arguments: { path: "package.json" } }
    ],
    stopReason: "toolUse",
    usage: { input: 1205, output: 42, ... },
    ...
  }
]
```

---

## 1.7 工具执行：从 ToolCall 到 ToolResultMessage

Agent 循环检查 AssistantMessage 的 `stopReason`：如果是 `"toolUse"`，说明有工具需要执行。它从 `content` 数组中提取所有 `type: "toolCall"` 的条目，然后执行工具调用。

工具执行经过三个阶段：

### 阶段一：准备（Prepare）

```typescript
// packages/agent/src/agent-loop.ts — prepareToolCall()
// 1. 在已注册工具列表中查找 "read" 工具
// 2. 验证参数：{ path: "package.json" } 是否符合工具的 TypeBox schema
// 3. 调用 beforeToolCall() 钩子（扩展可以在这里拦截）
```

### 阶段二：执行（Execute）

```typescript
// 调用 read 工具的 execute() 函数
const result = await readTool.execute(
  "toolu_01ABC",                    // toolCallId
  { path: "package.json" },         // 经过验证的参数
  signal,                           // AbortSignal
  onUpdate                          // 进度回调
);

// result 的形态：
{
  content: [{ type: "text", text: '{\n  "name": "pi-monorepo",\n  "version": "0.63.1",\n  ...\n}' }],
  details: { truncated: false, totalLines: 61, totalBytes: 2048 }
}
```

工具返回的 `AgentToolResult` 包含两部分：**content** 是要发回给 LLM 的内容（LLM 需要看到文件内容才能提取版本号）；**details** 是给 UI 展示的元数据（比如文件是否被截断、有多少行）。

### 阶段三：终结（Finalize）

```typescript
// 1. 调用 afterToolCall() 钩子（扩展可以修改结果）
// 2. 构建 ToolResultMessage
const toolResult: ToolResultMessage = {
  role: "toolResult",
  toolCallId: "toolu_01ABC",       // 对应哪个工具调用
  toolName: "read",
  content: [{ type: "text", text: '{\n  "name": "pi-monorepo", ...}' }],
  isError: false,
  timestamp: Date.now()
};

// 3. 追加到消息历史
context.messages.push(toolResult);
```

此时 `context.messages` 变成了：

```typescript
[
  { role: "user", ...用户消息... },
  { role: "assistant", ...LLM 回复（含 toolCall）... },
  { role: "toolResult", toolCallId: "toolu_01ABC", ...文件内容... }
]
```

---

## 1.8 第二轮：LLM 看到工具结果，给出最终答案

工具执行完毕后，Agent 循环回到开头——再次构建 Context 并调用 LLM。这次 Context 的 `messages` 数组多了两条消息（assistant 的工具调用 + toolResult），LLM 看到了 package.json 的内容。

LLM 这次的回复不包含工具调用：

```typescript
{
  role: "assistant",
  content: [
    { type: "text", text: "package.json 中的版本号是 0.63.1。" }
  ],
  stopReason: "stop",     // ← 注意：不再是 "toolUse"
  usage: { input: 1892, output: 18, ... },
  ...
}
```

`stopReason: "stop"` 告诉 Agent 循环：LLM 认为任务完成了。循环退出。

---

## 1.9 收尾：会话保存与 UI 渲染

Agent 循环结束后，按顺序发生以下事情：

**1. 发射结束事件**：`agent_end` 事件携带完整的消息列表。

**2. AgentSession 保存会话**：通过 SessionManager 把消息历史写入 JSONL 文件（详见第 4 章）。每条消息变成文件中的一行：

```json
{"type":"session","version":3,"id":"abc-123","timestamp":"2026-03-27T19:10:20.628Z","cwd":"/project"}
{"type":"message","id":"msg-1","parentId":null,"message":{"role":"user","content":[...],"timestamp":1711567820628}}
{"type":"message","id":"msg-2","parentId":"msg-1","message":{"role":"assistant","content":[...],"timestamp":1711567821100}}
{"type":"message","id":"msg-3","parentId":"msg-2","message":{"role":"toolResult","content":[...],"timestamp":1711567821500}}
{"type":"message","id":"msg-4","parentId":"msg-3","message":{"role":"assistant","content":[...],"timestamp":1711567822200}}
```

注意每条记录都有 `id` 和 `parentId`——这不是简单的线性日志，而是一棵**树**。这个设计支持分支和回溯，我们在第 4 章详细讲。

**3. TUI 渲染最终回复**：交互模式订阅了 Agent 事件，`message_update` 事件中的 `text_delta` 被实时渲染到终端，`message_end` 时完成最终布局。

---

## 1.10 全流程数据变换总结

让我们把整个过程中数据的形态变化串起来：

```
"读取 package.json 并告诉我版本号"     ← 原始字符串
         │
         ▼
UserMessage { role: "user", content: [...] }     ← 结构化消息
         │
         ▼ convertToLlm()
Message[] (标准 LLM 消息格式)                     ← 过滤非 LLM 消息
         │
         ▼ 组装 Context
Context { systemPrompt, messages, tools }        ← LLM 输入
         │
         ▼ transformMessages()
Provider 特定格式的请求                            ← 消息归一化
         │
         ▼ HTTP 流式请求
AssistantMessageEvent 序列                        ← 流式事件
  (start → text_delta... → toolcall_end → done)
         │
         ▼ 聚合
AssistantMessage { content: [text, toolCall], stopReason: "toolUse" }
         │
         ▼ 工具执行
AgentToolResult { content: [...], details: {...} }
         │
         ▼ 包装
ToolResultMessage { role: "toolResult", ... }
         │
         ▼ 追加到 messages，再次调用 LLM
AssistantMessage { content: [text], stopReason: "stop" }
         │
         ▼ 持久化
JSONL 文件中的一行行 JSON 记录
         │
         ▼ 渲染
终端上的 Markdown 格式文本
```

每一次变换都有明确的目的：字符串→消息是为了结构化；convertToLlm 是为了过滤自定义消息类型；transformMessages 是为了跨 Provider 兼容；流式事件是为了实时渲染；JSONL 是为了持久化和分支。

---

现在我们有了完整的数据流画面。但这张画面里有几个黑盒还没打开：pi-ai 内部的 Provider 机制到底怎么工作？`transformMessages()` 做了什么具体的归一化？流式事件是怎么在 Provider 和 Agent 之间传递的？

下一章，我们打开第一个黑盒——pi-ai，Pi 的 LLM 通信层。

---

### 质检报告

**讲解节奏**
- [x] 每个模块先讲"它是什么"再讲"里面有什么"
- [x] 没有上来就讲实现的情况

**周边知识**
- [x] 解释了为什么需要流式（LLM 生成慢，避免界面卡顿）
- [x] 解释了为什么需要 transformMessages（不同 Provider 格式差异）
- [x] 解释了为什么工具结果分 content 和 details（LLM vs UI 需要不同信息）

**讲透了吗**
- [x] 核心流程每一步解释了数据变化（从字符串到 JSONL 的完整链条）
- [x] 没有跳步
- [x] transformMessages、会话持久化、TUI 渲染标注"详见第 N 章"

**过渡自然吗**
- [x] 章头衔接上一章（"上一章我们用最粗的粒度走了一遍..."）
- [x] 章尾引出下一章（"下一章，我们打开第一个黑盒——pi-ai"）
- [x] 章内小节之间用场景进展自然衔接

**准确吗**
- [x] 类型定义均已对照源码验证
- [x] 事件序列符合实际实现
- [x] 项目特有术语（AgentSession、Compaction 等）已在序言中解释

**读得下去吗**
- [x] 用具体场景驱动（而非抽象描述）
- [x] 每个代码块有上下文解释
- [x] 每张图有文字讲解

**勘误建议**
无
