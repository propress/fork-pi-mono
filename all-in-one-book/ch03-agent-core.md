# 第三章 pi-agent-core — Agent 运行时

上一章我们深入了 pi-ai，理解了它如何把十几个 LLM Provider 统一到一套流式接口背后。但 pi-ai 只管"发消息、收回复"——它不关心回复里有没有工具调用，不关心工具执行后要不要再问 LLM，不关心对话的状态。

这些"编排"工作，就是 pi-agent-core 的职责。

---

## 3.1 Agent 运行时要解决什么问题

设想你拿到了 pi-ai 的 `streamSimple()` 函数。要实现一个能调用工具的 Agent，你需要自己写一个循环：

1. 把用户消息发给 LLM
2. 检查回复里有没有工具调用
3. 如果有，执行工具，把结果追加到消息历史，回到步骤 1
4. 如果没有，结束

这个循环看起来简单，但真实场景里有很多棘手的问题：

- **并行工具调用**：LLM 一次可能返回多个工具调用，怎么决定是串行还是并行执行？
- **用户中断**（Steering）：Agent 正在执行工具时，用户想修改指令，怎么插入新消息？
- **任务追加**（Follow-up）：Agent 完成后，用户想追加新任务，怎么不重新初始化？
- **实时反馈**：UI 需要知道"现在在执行哪个工具""LLM 在输出什么"，需要细粒度的事件。
- **工具拦截**：扩展可能想在工具执行前审查、在执行后修改结果。
- **错误恢复**：LLM 请求失败或工具报错时，怎么让循环不崩溃？

pi-agent-core 用**两层架构**来解决这些问题。

---

## 3.2 两层架构

```
┌───────────────────────────────────────────────────────────┐
│                   Agent 类（高层封装）                       │
│                                                           │
│  · 状态管理（AgentState）                                   │
│  · 事件订阅（subscribe/emit）                               │
│  · Steering / Follow-up 队列                               │
│  · prompt() / continue() / abort() 公共 API                │
│  · 事件到状态的同步更新                                      │
│                                                           │
│              ┌────────────────────────────┐               │
│              │   agentLoop（核心循环）      │               │
│              │                            │               │
│              │  · 嵌套 while 循环          │               │
│              │  · LLM 流式调用             │               │
│              │  · 工具执行（串行/并行）      │               │
│              │  · 事件发射（AgentEvent）     │               │
│              └────────────────────────────┘               │
└───────────────────────────────────────────────────────────┘
```

**底层 `agentLoop`**（`agent-loop.ts`）是一个纯函数——输入消息和配置，输出事件流。它不持有状态、不管 UI。它只关心："拿到消息 → 问 LLM → 执行工具 → 循环"。

**高层 `Agent` 类**（`agent.ts`）在 agentLoop 之上加了状态管理和控制能力。它持有 `AgentState`（当前模型、消息历史、工具列表等），维护 steering/follow-up 队列，并把 agentLoop 发射的事件同步更新到状态中。

为什么分两层？因为不同的使用场景需要不同的控制粒度：

- **coding-agent 的交互模式**用 Agent 类——需要状态管理、需要 steering 能力、需要事件订阅来驱动 UI。
- **嵌入场景**（比如一个简单的 Node.js 脚本）可能直接用 `agentLoop`——不需要状态管理，只需要一个事件流。

---

## 3.3 Agent 循环的核心：嵌套 while 循环

让我们看看 agentLoop 的核心结构。这是整个 Agent 系统的心脏：

```typescript
// packages/agent/src/agent-loop.ts — runLoop()（简化）
async function runLoop(context, newMessages, config, signal, emit) {
  let pendingMessages = await config.getSteeringMessages?.() || [];

  // 外层循环：处理 follow-up 消息
  while (true) {
    let hasMoreToolCalls = true;

    // 内层循环：处理工具调用和 steering 消息
    while (hasMoreToolCalls || pendingMessages.length > 0) {
      // 1. 注入 pending 消息（steering 或 follow-up）
      if (pendingMessages.length > 0) {
        for (const msg of pendingMessages) {
          emit({ type: "message_start", message: msg });
          emit({ type: "message_end", message: msg });
          context.messages.push(msg);
        }
        pendingMessages = [];
      }

      // 2. 调用 LLM，获取助手回复
      const message = await streamAssistantResponse(context, config, signal, emit);

      // 3. 错误/中止 → 直接结束
      if (message.stopReason === "error" || message.stopReason === "aborted") {
        emit({ type: "turn_end", message, toolResults: [] });
        emit({ type: "agent_end", messages: newMessages });
        return;
      }

      // 4. 检查工具调用
      const toolCalls = message.content.filter(c => c.type === "toolCall");
      hasMoreToolCalls = toolCalls.length > 0;

      // 5. 执行工具
      if (hasMoreToolCalls) {
        const toolResults = await executeToolCalls(context, message, config, signal, emit);
        for (const result of toolResults) {
          context.messages.push(result);
        }
      }

      emit({ type: "turn_end", message, toolResults });

      // 6. 检查 steering 消息（用户可能在工具执行期间发了新指令）
      pendingMessages = await config.getSteeringMessages?.() || [];
    }
    // 内层循环退出：没有工具调用，也没有 steering 消息

    // 7. 检查 follow-up 消息
    const followUps = await config.getFollowUpMessages?.() || [];
    if (followUps.length > 0) {
      pendingMessages = followUps;
      continue;  // 回到外层循环
    }

    break;  // 没有 follow-up → 彻底结束
  }

  emit({ type: "agent_end", messages: newMessages });
}
```

用一个具体的场景来理解这两层循环：

**内层循环**处理的是"LLM 想干活"的情况——LLM 说"我要调用 read 工具"，执行完后 LLM 又说"我要调用 edit 工具"，再执行，再问 LLM……直到 LLM 说"我做完了"（没有工具调用）。在这个过程中，如果用户通过 steering 插入了新指令，也在内层循环中处理。

**外层循环**处理的是"用户还有话说"的情况——内层循环结束（LLM 做完了），但用户通过 follow-up 队列追加了新任务。外层循环把 follow-up 消息设为 pending，然后重新进入内层循环。

---

## 3.4 事件体系：Agent 循环的"可观测性"

Agent 循环在每个关键步骤都发射事件，让外部订阅者能够实时了解"Agent 在做什么"。完整的事件类型：

```typescript
type AgentEvent =
  // Agent 生命周期
  | { type: "agent_start" }
  | { type: "agent_end"; messages: AgentMessage[] }

  // Turn（轮次）生命周期——一个 turn = 一次 LLM 回复 + 后续工具执行
  | { type: "turn_start" }
  | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }

  // 消息生命周期
  | { type: "message_start"; message: AgentMessage }
  | { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
  | { type: "message_end"; message: AgentMessage }

  // 工具执行生命周期
  | { type: "tool_execution_start"; toolCallId: string; toolName: string; args: any }
  | { type: "tool_execution_update"; toolCallId: string; toolName: string; partialResult: any }
  | { type: "tool_execution_end"; toolCallId: string; toolName: string; result: any; isError: boolean };
```

一个完整的"有工具调用"场景会产生这样的事件序列：

```
agent_start
├─ turn_start
│  ├─ message_start  (用户消息)
│  ├─ message_end    (用户消息)
│  ├─ message_start  (助手消息开始流式输出)
│  ├─ message_update (text_delta: "我来读取...")
│  ├─ message_update (toolcall_start)
│  ├─ message_update (toolcall_delta: '{"path":...')
│  ├─ message_update (toolcall_end)
│  ├─ message_end    (助手消息完成)
│  ├─ tool_execution_start (read 工具)
│  ├─ tool_execution_end   (read 工具完成)
│  ├─ message_start  (toolResult 消息)
│  └─ message_end    (toolResult 消息)
├─ turn_end
│
├─ turn_start        (LLM 再次回复)
│  ├─ message_start  (助手最终回复)
│  ├─ message_update (text_delta: "版本号是...")
│  ├─ message_end
├─ turn_end
│
└─ agent_end
```

注意 `message_update` 事件携带了原始的 `AssistantMessageEvent`（来自 pi-ai），这让 UI 层可以直接拿到 `text_delta` 来实时渲染。Agent 层的事件是 pi-ai 事件的**包装**，不是替代。

---

## 3.5 工具执行：三阶段生命周期

当 Agent 循环检测到 LLM 回复中有工具调用时，进入工具执行流程。这个流程分三个阶段：

### 阶段一：准备（Prepare）

```typescript
const preparation = await prepareToolCall(context, assistantMessage, toolCall, config, signal);
```

准备阶段做三件事：
1. **查找工具**：在 `context.tools` 中按名字找到对应的 `AgentTool` 对象。
2. **验证参数**：用 TypeBox schema 验证 LLM 提供的参数是否合法。如果参数不合法（LLM 有时会生成错误的参数），不执行工具，直接返回一个错误结果。
3. **调用 `beforeToolCall()` 钩子**：扩展可以在这里拦截工具调用——返回 `{ block: true, reason: "..." }` 可以阻止执行。

准备阶段的结果有两种可能：
- **`"immediate"`**：工具不需要执行（被拦截或参数错误），直接有了结果。
- **`"prepared"`**：工具准备就绪，可以执行。

### 阶段二：执行（Execute）

```typescript
const executed = await executePreparedToolCall(preparation, signal, emit);
```

调用工具的 `execute()` 函数。工具在执行过程中可以通过 `onUpdate` 回调推送进度更新（比如 bash 工具可以推送实时输出），这些更新会作为 `tool_execution_update` 事件发射。

### 阶段三：终结（Finalize）

```typescript
const result = await finalizeExecutedToolCall(context, assistantMessage, preparation, executed, config, signal, emit);
```

终结阶段做两件事：
1. **调用 `afterToolCall()` 钩子**：扩展可以修改工具结果（改 content、改 details、改 isError 标志）。
2. **构建 ToolResultMessage**：把工具结果包装成标准消息格式，追加到消息历史。

这三个阶段的分离让扩展有了丰富的拦截点——可以在执行前审查（安全检查）、在执行后修改（结果增强）。

---

## 3.6 串行 vs 并行：工具执行的两种模式

LLM 可能在一次回复中返回多个工具调用（比如同时读取两个文件）。pi-agent-core 提供两种执行模式：

**串行模式**（`toolExecution: "sequential"`）：

```
prepare(tool1) → execute(tool1) → finalize(tool1)
  → prepare(tool2) → execute(tool2) → finalize(tool2)
```

每个工具完整走完三个阶段，下一个才开始。适用于工具之间有依赖的场景。

**并行模式**（`toolExecution: "parallel"`，默认）：

```
prepare(tool1) → prepare(tool2)    ← 准备阶段串行（因为 beforeToolCall 可能需要顺序决策）
  ↓                ↓
execute(tool1) ┃ execute(tool2)    ← 执行阶段并行
  ↓                ↓
finalize(tool1) → finalize(tool2)  ← 终结阶段按原始顺序串行
```

并行模式的核心代码：

```typescript
// packages/agent/src/agent-loop.ts — executeToolCallsParallel()
async function executeToolCallsParallel(context, assistantMessage, toolCalls, config, signal, emit) {
  const runnableCalls: PreparedToolCall[] = [];

  // 串行准备
  for (const toolCall of toolCalls) {
    const preparation = await prepareToolCall(...);
    if (preparation.kind === "immediate") {
      results.push(await emitToolCallOutcome(...));
    } else {
      runnableCalls.push(preparation);
    }
  }

  // 并行执行
  const runningCalls = runnableCalls.map(prepared => ({
    prepared,
    execution: executePreparedToolCall(prepared, signal, emit),  // 不 await → 并行
  }));

  // 按原始顺序收集结果
  for (const running of runningCalls) {
    const executed = await running.execution;  // 等待完成
    results.push(await finalizeExecutedToolCall(...));
  }

  return results;
}
```

关键设计决策：准备阶段**必须串行**，因为 `beforeToolCall()` 钩子可能会弹出 UI 确认对话框（"是否允许执行 rm -rf？"），用户需要按顺序审查。但执行阶段可以并行——两个文件读取操作之间没有依赖。终结阶段按原始顺序串行，保证 ToolResult 消息在消息历史中的顺序与 ToolCall 一致（某些 LLM Provider 对此有要求）。

---

## 3.7 Agent 类：状态管理层

`Agent` 类在 `agentLoop` 之上提供了状态管理和控制 API。它的核心是 `AgentState`：

```typescript
interface AgentState {
  systemPrompt: string;
  model: Model<any>;
  thinkingLevel: ThinkingLevel;       // "off" | "minimal" | "low" | "medium" | "high" | "xhigh"
  tools: AgentTool<any>[];
  messages: AgentMessage[];           // 完整的消息历史
  isStreaming: boolean;               // Agent 是否正在运行
  streamMessage: AgentMessage | null; // 当前正在流式输出的消息
  pendingToolCalls: Set<string>;      // 正在执行的工具调用 ID
  error?: string;
}
```

Agent 类通过 `_processLoopEvent()` 方法把 agentLoop 发射的事件同步到状态中：

```typescript
private _processLoopEvent(event: AgentEvent): void {
  switch (event.type) {
    case "message_start":
      this._state.streamMessage = event.message;  // 记录正在流式的消息
      break;
    case "message_end":
      this._state.streamMessage = null;
      this.appendMessage(event.message);           // 追加到消息历史
      break;
    case "tool_execution_start":
      this._state.pendingToolCalls.add(event.toolCallId);  // 记录执行中的工具
      break;
    case "tool_execution_end":
      this._state.pendingToolCalls.delete(event.toolCallId);
      break;
    case "agent_end":
      this._state.isStreaming = false;
      this._state.streamMessage = null;
      break;
  }
  this.emit(event);  // 转发给外部订阅者
}
```

这里有一个重要的设计选择：**Agent 类在事件处理中设置了一个"屏障"（barrier）**——它 await 了 `_processLoopEvent()`，确保状态更新完成后才继续循环。而 agentLoop 的低层 API（直接使用 `EventStream`）是"观察性"的——事件处理不会阻塞循环。

为什么需要这个屏障？因为 `beforeToolCall()` 钩子可能需要读取最新的状态（比如检查消息历史中是否已有类似的工具调用）。如果状态还没更新就开始下一步，钩子看到的状态是滞后的。

---

## 3.8 Steering 与 Follow-up：对话中的"插队"

Steering 和 Follow-up 是 Pi 的两种消息注入机制，它们的时机不同：

**Steering（转向）**：Agent 运行期间注入。消息会在当前工具轮次结束后（内层循环检查 `getSteeringMessages()`）被注入到下一轮 LLM 调用的上下文中。

```typescript
agent.steer("停一下，先别改那个文件，改另一个");
// → 消息进入 steering 队列
// → 内层循环在下次 LLM 调用前检查队列
// → 注入为新的 user message
```

**Follow-up（后续）**：Agent 完成后注入。消息在内层循环退出后（外层循环检查 `getFollowUpMessages()`）被处理。

```typescript
agent.followUp("顺便也把测试跑一下");
// → 消息进入 follow-up 队列
// → Agent 完成当前任务后检查队列
// → 开始新的内层循环处理这个任务
```

两者都支持两种出队模式：
- **`"one-at-a-time"`**：每次只取一条消息。适合交互模式——用户排了多条消息，一次处理一条，中间可以看到结果。
- **`"all"`**：一次取完所有消息。适合批量处理。

---

## 3.9 自定义消息类型：AgentMessage

pi-ai 的 `Message` 类型只有三种角色（user/assistant/toolResult）——这是 LLM 能理解的。但 Agent 层面需要更多的消息类型来记录"不发给 LLM 但对 UI 有意义"的信息。

Pi 用 TypeScript 的**声明合并**（declaration merging）实现了可扩展的消息类型系统：

```typescript
// packages/agent/src/types.ts
interface CustomAgentMessages {}  // 空接口，等待扩展

type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];

// packages/coding-agent/src/core/messages.ts — coding-agent 扩展了它
declare module "@mariozechner/pi-agent-core" {
  interface CustomAgentMessages {
    bashExecution: BashExecutionMessage;      // Bash 命令执行记录
    compactionSummary: CompactionSummaryMessage;  // 上下文压缩摘要
    branchSummary: BranchSummaryMessage;      // 分支跳转摘要
    custom: CustomMessage;                    // 扩展自定义消息
  }
}
```

这样，消息历史（`AgentState.messages`）中可以包含任意自定义消息。但在发送给 LLM 前，`convertToLlm()` 函数会过滤掉所有非标准消息类型——LLM 只看到 user/assistant/toolResult。

这种"宽进严出"的设计让消息历史既能服务 UI 展示（Bash 输出、压缩摘要等），又不会干扰 LLM 通信。

---

## 3.10 Proxy 模式：为浏览器应用而生

pi-agent-core 还提供了一个 `streamProxy()` 函数，用于浏览器应用把 LLM 请求路由到后端服务器：

```typescript
const agent = new Agent({
  streamFn: (model, context, options) =>
    streamProxy(model, context, {
      ...options,
      authToken: "jwt-token",
      proxyUrl: "https://api.example.com/proxy",
    }),
});
```

为什么需要代理？浏览器环境有两个限制：API Key 不能暴露在客户端代码中；部分 Provider 的 API 不支持浏览器直连（CORS 限制）。通过后端代理，API Key 在服务端管理，浏览器只需要一个认证 token。

Proxy 模式还做了一个**带宽优化**：服务端发送事件时去掉了每个事件中的 `partial` 字段（完整的 AssistantMessage 快照）。`partial` 在每次文本增量更新时都会被完整发送，对于长消息会占用大量带宽。客户端的 `streamProxy()` 在收到增量事件后，自己在内存中重建 `partial`。

代理事件（`ProxyAssistantMessageEvent`）相比标准事件去掉了 `partial`，只保留增量数据。客户端收到后做本地拼接：

```
服务端发送: { type: "text_delta", contentIndex: 0, delta: "hello" }  // 无 partial
客户端重建: { type: "text_delta", contentIndex: 0, delta: "hello", partial: { ...完整消息... } }
```

---

## 3.11 小结

pi-agent-core 的设计可以概括为：

- **核心循环**（agentLoop）是一个纯粹的嵌套 while 循环，内层处理工具调用和 steering，外层处理 follow-up。它不持有状态，只通过事件回调通知外部。
- **Agent 类**在循环之上加了状态管理、事件订阅、控制 API。它通过事件屏障保证状态的一致性。
- **工具执行**有三个阶段（准备→执行→终结），每个阶段都有钩子点，支持串行和并行两种模式。
- **事件体系**从 Agent 级到消息级到工具级，提供了完整的可观测性。
- **自定义消息**通过声明合并实现可扩展性，`convertToLlm()` 保证了 LLM 只看到标准消息。

但 Agent 运行时只是"引擎"——它不关心怎么解析命令行参数、怎么持久化会话、怎么加载扩展。这些"用户侧"的关注点，由下一层来处理。

下一章，我们进入 pi-coding-agent 的核心——看看它如何从一个 CLI 命令出发，经过参数解析、模型选择、会话加载，最终创建并驱动一个 AgentSession。

---

### 质检报告

**讲解节奏**
- [x] 先讲 agent 运行时要解决什么问题（编排 LLM 和工具的复杂性），再讲怎么解决
- [x] 先讲两层架构的全景，再分别展开每层

**周边知识**
- [x] 解释了为什么分两层（不同使用场景需要不同控制粒度）
- [x] 解释了为什么准备阶段必须串行（beforeToolCall 可能需要用户交互）
- [x] 解释了为什么需要事件屏障（状态一致性）
- [x] 解释了为什么需要 Proxy 模式（浏览器安全限制）

**讲透了吗**
- [x] 嵌套 while 循环的逻辑完整展开
- [x] 工具执行三阶段每个阶段做什么、为什么这么分
- [x] 事件序列的完整示例
- [x] 串行 vs 并行的具体区别和代码

**过渡自然吗**
- [x] 章头衔接上一章（"pi-ai 只管发消息收回复，编排工作是 agent-core 的职责"）
- [x] 章尾引出下一章（"下一章进入 coding-agent 的核心"）
- [x] 章内小节用问题驱动衔接

**准确吗**
- [x] 所有代码均对照源码验证
- [x] 事件类型定义和事件序列已验证
- [x] Steering/Follow-up 机制已验证

**读得下去吗**
- [x] 用具体场景解释抽象概念（如嵌套循环的意义）
- [x] 每段代码有上下文解释

**勘误建议**
无
