# 第二章 pi-ai — 统一的 LLM 通信层

上一章我们看到，Agent 循环通过调用 `streamSimple()` 与 LLM 通信，拿到一个 `AssistantMessageEventStream`。这一章我们打开这个黑盒，看看 pi-ai 内部是怎么把十几个不同的 LLM Provider 统一到一套接口背后的。

---

## 2.1 要解决的问题

假设你想让应用同时支持 OpenAI、Anthropic、Google 三个 Provider。直觉做法是给每个 Provider 写一套调用代码。但问题很快出现：

- **API 格式不同**：OpenAI 用 `chat.completions.create()`，Anthropic 用 `messages.create()`，Google 用 `generateContent()`。请求和响应的 JSON 结构完全不一样。
- **流式协议不同**：OpenAI 用 Server-Sent Events，Google Gemini CLI 用 WebSocket。
- **工具调用格式不同**：OpenAI 的工具调用 ID 可能有 450+ 个字符，Anthropic 要求最多 64 个字符且只允许 `[a-zA-Z0-9_-]`。
- **思考/推理支持不同**：OpenAI 用 `reasoning_effort` 参数，Anthropic 用 `budget_tokens`，Google 用 `thinkingConfig`。

如果每个 Provider 的差异都暴露给上层，Agent 循环就需要知道"我现在用的是哪个 Provider"才能正确构造请求。这违背了 Agent 循环应该是 Provider 无关的设计原则。

pi-ai 的方案是**适配器模式**（Adapter Pattern）：定义一套统一的消息类型和流式协议，让每个 Provider 实现各自负责翻译。上层只和统一接口打交道。

---

## 2.2 四个核心组件

pi-ai 的内部架构由四个核心组件构成：

```
┌──────────────────────────────────────────────────────────┐
│                      公共 API                             │
│   stream() / streamSimple() / complete() / completeSimple()│
└───────────────────────┬──────────────────────────────────┘
                        │ 查找 Provider
                        ▼
┌──────────────────────────────────────────────────────────┐
│                  API Registry（注册表）                    │
│   Map<api名 → Provider 实现>                              │
│   "anthropic-messages" → { stream, streamSimple }         │
│   "openai-completions" → { stream, streamSimple }         │
└───────────────────────┬──────────────────────────────────┘
                        │ 委托
                        ▼
┌──────────────────────────────────────────────────────────┐
│              Provider 实现（每家一个文件）                   │
│   anthropic.ts / openai-completions.ts / google.ts / ...  │
│   负责：消息格式转换 → SDK 调用 → 流式事件发射               │
└───────────────────────┬──────────────────────────────────┘
                        │ 承载事件
                        ▼
┌──────────────────────────────────────────────────────────┐
│              EventStream（事件流容器）                      │
│   异步可迭代 + Promise 的混合体                             │
│   生产者推事件，消费者逐个消费或等最终结果                     │
└──────────────────────────────────────────────────────────┘
```

接下来逐个展开。

---

## 2.3 Model：不只是一个名字

在调用任何 Provider 之前，你需要一个 `Model` 对象。它不是一个简单的字符串标识符，而是包含完整元数据的配置对象：

```typescript
// packages/ai/src/types.ts
interface Model<TApi extends Api> {
  id: string;              // "claude-sonnet-4-20250514"
  name: string;            // "Claude Sonnet 4"
  api: TApi;               // "anthropic-messages" — 决定用哪个 Provider
  provider: Provider;      // "anthropic" — Provider 厂商名
  baseUrl: string;         // "https://api.anthropic.com"
  reasoning: boolean;      // 是否支持扩展思考
  input: ("text" | "image")[];  // 支持的输入类型
  cost: {
    input: number;         // 每百万 token 的美元价格
    output: number;
    cacheRead: number;
    cacheWrite: number;
  };
  contextWindow: number;   // 上下文窗口大小（token 数）
  maxTokens: number;       // 最大输出 token 数
}
```

所有模型配置预先存储在 `models.generated.ts` 文件中——这是一个由脚本从各 Provider 的模型列表自动生成的静态文件。通过 `getModel()` 函数查找：

```typescript
// packages/ai/src/models.ts
const model = getModel("anthropic", "claude-sonnet-4-20250514");
// 返回一个 Model<"anthropic-messages"> 对象，包含完整的价格、窗口大小等信息
```

为什么要把价格、窗口大小这些信息放在 Model 里？因为上层（Agent 循环、会话管理）需要这些信息做决策——比如判断上下文是否即将溢出需要压缩（compaction），或者在 UI 里显示本次对话的费用。把这些信息绑定在 Model 上，避免了到处传递配置。

`api` 字段是连接 Model 和 Provider 的桥梁。Pi 目前注册了 10 种 API 类型：

| API 类型 | 对应 Provider | 说明 |
|----------|-------------|------|
| `anthropic-messages` | Anthropic | Claude 系列 |
| `openai-completions` | OpenAI + 兼容端点 | GPT 系列、Groq、DeepSeek 等 |
| `openai-responses` | OpenAI | Responses API（新版） |
| `openai-codex-responses` | OpenAI | Codex 模型专用 |
| `azure-openai-responses` | Azure OpenAI | Azure 部署的 OpenAI 模型 |
| `google-generative-ai` | Google | Gemini 系列 |
| `google-vertex` | Google Cloud | Vertex AI 部署 |
| `google-gemini-cli` | Google | Gemini CLI（WebSocket） |
| `mistral-conversations` | Mistral | Mistral 系列 |
| `bedrock-converse-stream` | AWS | Bedrock 上的各种模型 |

注意同一个 Provider 可能有多个 API 类型（比如 Google 有三个、OpenAI 有三个），因为同一厂商可能提供不同的 API 端点。

---

## 2.4 API Registry：Provider 的注册与查找

API Registry 是一个简单的 Map，key 是 API 类型名，value 是该 API 的 `stream` 和 `streamSimple` 函数：

```typescript
// packages/ai/src/api-registry.ts
const apiProviderRegistry = new Map<string, RegisteredApiProvider>();

function registerApiProvider(provider: ApiProvider): void {
  apiProviderRegistry.set(provider.api, {
    provider: {
      api: provider.api,
      stream: wrapStream(provider),         // 加了类型安全检查
      streamSimple: wrapStreamSimple(provider),
    },
    sourceId: provider.sourceId,
  });
}

function getApiProvider(api: string): ApiProvider | undefined {
  return apiProviderRegistry.get(api)?.provider;
}
```

`wrapStream()` 做了一件小但重要的事：在调用实际的 Provider 函数前，验证 `model.api` 是否和注册时的 API 类型匹配。这是一道运行时安全网——防止把 Anthropic 的 Model 误传给 OpenAI 的 Provider。

当用户调用 `stream(model, context)` 时，流程是：

```typescript
// packages/ai/src/stream.ts
function stream(model, context, options): AssistantMessageEventStream {
  const provider = resolveApiProvider(model.api);  // 从注册表查找
  return provider.stream(model, context, options);  // 委托给 Provider
}
```

一行代码完成分发。上层不需要知道 Provider 的存在。

---

## 2.5 延迟加载：为什么不一开始就导入所有 Provider？

Pi 支持 10 个 Provider，每个 Provider 都依赖各自的 SDK（`@anthropic-ai/sdk`、`openai`、`@google/genai` 等）。如果在启动时就导入所有 Provider，会拖慢启动速度——用户可能只用 Anthropic，但要为加载 OpenAI、Google、AWS 的 SDK 付出时间。

Pi 的解决方案是**延迟加载**（lazy loading）：Provider 模块只在第一次被调用时才通过 `import()` 动态导入。

```typescript
// packages/ai/src/providers/register-builtins.ts — 简化后的核心模式

// 1. 缓存变量：每个 Provider 一个
let anthropicPromise: Promise<LazyProviderModule> | undefined;

// 2. 加载函数：用 ||= 确保只导入一次
function loadAnthropic(): Promise<LazyProviderModule> {
  anthropicPromise ||= import("./anthropic.js").then((module) => ({
    stream: module.streamAnthropic,
    streamSimple: module.streamSimpleAnthropic,
  }));
  return anthropicPromise;
}

// 3. 延迟包装：创建一个立即返回 EventStream 的函数
//    内部异步加载模块，加载完后把真实的事件流转发到外层
function createLazyStream(loadModule): StreamFunction {
  return (model, context, options) => {
    const outer = new AssistantMessageEventStream();
    loadModule()
      .then((module) => {
        const inner = module.stream(model, context, options);
        // 把 inner 的事件逐个转发到 outer
        (async () => {
          for await (const event of inner) { outer.push(event); }
        })();
      })
      .catch((error) => {
        // 导入失败 → 发射错误事件
        outer.push({ type: "error", reason: "error", error: ... });
      });
    return outer;  // 立即返回，不等待导入完成
  };
}
```

关键技巧在第 3 步：`createLazyStream` 返回的函数**同步返回**一个 EventStream，然后在后台异步加载 Provider 模块。加载完成后，把真实 Provider 产生的事件逐个转发到外层的 EventStream。对调用者来说，这个过程是透明的——它只看到一个 EventStream 在产出事件。

`||=` 运算符保证了模块只会被 `import()` 一次。第二次调用时直接复用第一次的 Promise。

---

## 2.6 EventStream：流式事件的管道

`EventStream` 是 pi-ai 最巧妙的数据结构之一。它解决了一个常见问题：生产者（Provider）异步地产生事件，消费者（Agent 循环）需要逐个处理这些事件，还要在所有事件结束后拿到最终结果。

```typescript
// packages/ai/src/utils/event-stream.ts — 核心实现
class EventStream<T, R = T> implements AsyncIterable<T> {
  private queue: T[] = [];                    // 缓冲区：事件推入但还没人消费
  private waiting: ((value) => void)[] = [];  // 等待者：消费者在等下一个事件
  private done = false;
  private finalResultPromise: Promise<R>;
  private resolveResult!: (value: R) => void;

  constructor(isComplete?, extractResult?) {
    this.finalResultPromise = new Promise((resolve) => {
      this.resolveResult = resolve;
    });
    // isComplete / extractResult 用于自动检测结束事件
  }

  push(event: T): void {
    // 如果这个事件标志着流结束（比如 "done" 或 "error"）
    if (this.isComplete?.(event)) {
      this.done = true;
      this.resolveResult(this.extractResult!(event));
    }
    // 如果有人在等 → 直接交给它
    if (this.waiting.length > 0) {
      const resolve = this.waiting.shift()!;
      resolve({ value: event, done: false });
    } else {
      // 没人在等 → 放进缓冲区
      this.queue.push(event);
    }
  }

  async *[Symbol.asyncIterator]() {
    while (true) {
      if (this.queue.length > 0) {
        yield this.queue.shift()!;             // 有缓冲 → 立即产出
      } else if (this.done) {
        return;                                // 已结束 → 退出
      } else {
        // 既没缓冲也没结束 → 等待下一次 push
        const result = await new Promise<IteratorResult<T>>((resolve) => {
          this.waiting.push(resolve);
        });
        if (result.done) return;
        yield result.value;
      }
    }
  }

  result(): Promise<R> {
    return this.finalResultPromise;
  }
}
```

这是一个**无锁的生产者-消费者队列**。工作原理：

- 生产者调用 `push(event)` 推入事件。如果此时有消费者在等，直接把事件交给它（零拷贝）。否则放进 `queue` 缓冲。
- 消费者通过 `for await (const event of stream)` 消费。如果 `queue` 里有数据，立即拿走。如果没有，创建一个 Promise 等待下次 `push`。
- 流结束时（`done` 事件或显式调用 `end()`），`finalResultPromise` 被 resolve，消费者的循环退出。

`AssistantMessageEventStream` 是 `EventStream` 的特化版，它知道 `type: "done"` 和 `type: "error"` 是结束事件，并能自动从中提取最终的 `AssistantMessage`：

```typescript
class AssistantMessageEventStream extends EventStream<AssistantMessageEvent, AssistantMessage> {
  constructor() {
    super(
      (event) => event.type === "done" || event.type === "error",  // 结束条件
      (event) => event.type === "done" ? event.message : event.error  // 提取结果
    );
  }
}
```

这个设计的优雅之处在于，同一个对象既能做 `for await` 逐事件消费（实时渲染 UI），又能 `await stream.result()` 等最终结果（Agent 循环需要完整的 AssistantMessage）。两种用法可以同时进行。

---

## 2.7 Provider 内部：以 Anthropic 为例

现在让我们看看一个 Provider 内部做了什么。以 Anthropic Provider 为例，当 Agent 循环调用 `streamSimple(model, context, options)` 时：

### 步骤一：消息转换

首先，Provider 调用 `transformMessages()` 把 Pi 的统一消息格式转换成 Anthropic API 需要的格式。这个函数做了几件重要的事（下一节详细展开）。

### 步骤二：构建 SDK 请求

```typescript
// 简化后的 Anthropic Provider 逻辑
const client = new Anthropic({ apiKey, baseURL: model.baseUrl });

const params: MessageCreateParamsStreaming = {
  model: model.id,
  max_tokens: adjustedMaxTokens,
  system: [{ type: "text", text: context.systemPrompt, cache_control: ... }],
  messages: transformedMessages,     // 转换后的消息
  tools: context.tools?.map(convertTool),  // 工具转 Anthropic 格式
  stream: true,
};

// 如果模型支持思考且请求了思考
if (model.reasoning && thinkingBudget > 0) {
  params.thinking = {
    type: "enabled",
    budget_tokens: thinkingBudget,  // 比如 8192
  };
}
```

### 步骤三：流式读取与事件发射

```typescript
const stream = new AssistantMessageEventStream();

(async () => {
  const output: AssistantMessage = {
    role: "assistant",
    content: [],
    usage: { input: 0, output: 0, ... },
    stopReason: "stop",
    timestamp: Date.now(),
    api: model.api,
    provider: model.provider,
    model: model.id,
  };

  try {
    stream.push({ type: "start", partial: output });

    const response = client.messages.stream(params, { signal });

    for await (const event of response) {
      if (event.type === "content_block_start") {
        // 新的内容块开始：可能是文本、思考或工具调用
        if (event.content_block.type === "text") {
          output.content.push({ type: "text", text: "" });
          stream.push({ type: "text_start", contentIndex: idx, partial: output });
        }
        // ... 类似处理 thinking 和 tool_use
      }

      if (event.type === "content_block_delta") {
        if (event.delta.type === "text_delta") {
          output.content[idx].text += event.delta.text;
          stream.push({
            type: "text_delta",
            contentIndex: idx,
            delta: event.delta.text,
            partial: output,
          });
        }
        // ... 类似处理 thinking_delta 和 input_json_delta
      }

      if (event.type === "message_delta") {
        // 消息级元数据：stop_reason、usage
        output.usage.output = event.usage.output_tokens;
      }
    }

    stream.push({ type: "done", reason: mapStopReason(output), message: output });
  } catch (error) {
    output.stopReason = "error";
    output.errorMessage = error.message;
    stream.push({ type: "error", reason: "error", error: output });
  }
})();

return stream;  // 立即返回，不等异步完成
```

核心模式：**立即返回 EventStream，在后台异步推事件**。这让调用者可以立刻开始 `for await` 消费，而不需要等 HTTP 连接建立。

每个 Provider 都遵循相同的模式：创建 EventStream → 在后台异步调用 SDK → 解析 SDK 的流式响应 → 转换成统一的 `AssistantMessageEvent` → 推入 EventStream。差异在于每个 SDK 的流式协议不同、字段名不同、事件类型不同。Provider 的职责就是吸收这些差异。

---

## 2.8 transformMessages：跨 Provider 兼容的关键

`transformMessages()` 可能是 pi-ai 中最不起眼但最关键的函数。它解决的问题是：**同一段对话历史可能会在不同 Provider 之间传递**（比如用户从 Claude 切换到 GPT），而不同 Provider 对消息格式有不同的要求。

它做了两轮处理：

### 第一轮：内容转换

逐条遍历消息，处理 AssistantMessage 中的每个内容块：

**思考块（ThinkingContent）处理**：
- **被审查的思考**（`redacted: true`）：只在同一模型重放时保留（Provider 用加密签名验证），跨模型时丢弃。
- **带签名的思考**：同一模型保留签名（用于 API 的多轮连续性），跨模型时转为普通文本。
- **空思考**：直接丢弃。
- **普通思考**：跨模型时转为文本块（因为目标 Provider 可能不支持思考块）。

**工具调用 ID 归一化**：

这是一个现实中很棘手的问题。OpenAI Responses API 生成的工具调用 ID 可以有 450+ 个字符，且包含 `|` 等特殊字符。但 Anthropic 要求 ID 最多 64 个字符，只允许 `[a-zA-Z0-9_-]`。如果用户从 OpenAI 切换到 Anthropic 继续对话，这些 ID 就会导致 API 报错。

`transformMessages()` 通过一个回调函数 `normalizeToolCallId()` 来处理这个问题——调用者（Provider）提供 ID 归一化逻辑，函数内部维护一个 `oldId → newId` 的映射表，确保 ToolCall 和对应的 ToolResult 使用一致的 ID。

**文本签名处理**：
- 同一模型重放时保留 `textSignature`（用于 OpenAI 的消息元数据验证）。
- 跨模型时移除签名，只保留纯文本。

### 第二轮：修补孤立的工具调用

一个 AssistantMessage 可能包含工具调用（`toolCall`），但后续消息中没有对应的 `toolResult`——这在用户中途中断、或者跨模型切换时可能发生。大多数 Provider 的 API 要求每个工具调用都必须有对应的结果。

`transformMessages()` 在第二轮扫描中检测这种情况，为孤立的工具调用插入合成的错误结果：

```typescript
const syntheticResult: ToolResultMessage = {
  role: "toolResult",
  toolCallId: orphanedToolCall.id,
  toolName: orphanedToolCall.name,
  content: [{ type: "text", text: "Tool call was not executed." }],
  isError: true,
  timestamp: Date.now(),
};
```

它还处理一种特殊情况：当用户消息紧跟在助手的工具调用之后（用户打断了工具执行），同样需要补上合成结果，否则 API 会因为"工具调用后面应该是工具结果而不是用户消息"而报错。

---

## 2.9 stream vs streamSimple：两层 API

Pi 提供了两层流式 API：

**底层 `stream()`**：接受 Provider 特定的选项类型。如果你需要精细控制某个 Provider 的参数（比如 Anthropic 的 `cache_control` 配置），用这个。

**高层 `streamSimple()`**：接受统一的 `SimpleStreamOptions`，最重要的是 `reasoning` 参数：

```typescript
streamSimple(model, context, {
  reasoning: "high",           // "off" | "minimal" | "low" | "medium" | "high" | "xhigh"
  thinkingBudgets: { ... },    // 可选：自定义每个级别的 token 预算
  temperature: 0.7,
  maxTokens: 4096,
});
```

`reasoning` 参数让上层不需要知道每个 Provider 怎么配置思考——`streamSimple()` 内部负责翻译：

| 参数 | Anthropic | OpenAI | Google |
|------|-----------|--------|--------|
| `reasoning: "high"` | `thinking.budget_tokens: 16384` | `reasoning_effort: "high"` | `thinkingConfig.thinkingBudget: 16384` |

token 预算的默认映射是：
- `minimal` → 1024 tokens
- `low` → 2048 tokens
- `medium` → 8192 tokens
- `high` → 16384 tokens
- `xhigh` → 对支持的模型使用最大预算，否则降级为 `high`

Agent 循环统一使用 `streamSimple()`，避免了与 Provider 细节的耦合。

---

## 2.10 成本计算

每个 Model 对象都携带价格信息（每百万 token 的美元价格）。当 Provider 返回 `usage` 数据（输入/输出/缓存读/缓存写的 token 数）后，pi-ai 用一个简单的公式计算费用：

```typescript
// packages/ai/src/models.ts
function calculateCost(model: Model, usage: Usage): void {
  usage.cost.input = (model.cost.input / 1_000_000) * usage.input;
  usage.cost.output = (model.cost.output / 1_000_000) * usage.output;
  usage.cost.cacheRead = (model.cost.cacheRead / 1_000_000) * usage.cacheRead;
  usage.cost.cacheWrite = (model.cost.cacheWrite / 1_000_000) * usage.cacheWrite;
  usage.cost.total = usage.cost.input + usage.cost.output
                   + usage.cost.cacheRead + usage.cost.cacheWrite;
}
```

为什么要区分 `cacheRead` 和 `cacheWrite`？这涉及到 **Prompt Caching**——某些 Provider（Anthropic、Google）允许缓存系统提示词和早期消息，后续请求如果命中缓存可以大幅降低费用。缓存的首次写入可能收费，但后续读取比正常输入便宜很多（Anthropic 的缓存读取价格是正常输入的 10%）。

---

## 2.11 一个 Provider 的特殊技巧：Claude Code 兼容

值得一提的是 Anthropic Provider 中一个有趣的实现细节。Pi 会把工具名称转换为 Claude Code（Anthropic 官方编码 Agent）的命名风格：

```typescript
// packages/ai/src/providers/anthropic.ts
const claudeCodeTools = [
  "Read", "Write", "Edit", "Bash", "Grep", "Glob",
  "AskUserQuestion", "EnterPlanMode", "ExitPlanMode",
  "KillShell", "NotebookEdit", "Skill", "Task",
  "TaskOutput", "TodoWrite", "WebFetch", "WebSearch",
];

// 例如 Pi 的 "read" 工具 → 发送给 API 时变成 "Read"
const toClaudeCodeName = (name) => ccToolLookup.get(name.toLowerCase()) ?? name;
```

这样做的原因是：Anthropic 对 Claude Code 的工具调用做了针对性优化（更好的 prompt caching、可能的质量调优）。通过伪装成 Claude Code 的工具命名风格，Pi 可以享受这些优化。收到响应后再把工具名转回 Pi 的格式。

---

## 2.12 小结

pi-ai 的设计可以归纳为一句话：**一个注册表 + 一套事件协议 + 一组适配器**。

- **注册表**（API Registry）让上层通过 Model 的 `api` 字段自动找到对应的 Provider，无需硬编码。
- **事件协议**（AssistantMessageEvent）统一了所有 Provider 的流式输出格式，12 种事件类型覆盖了文本、思考、工具调用的开始/增量/结束。
- **适配器**（Provider 实现）各自负责翻译 SDK 的格式差异、处理特殊逻辑（缓存控制、思考预算、工具名映射等）。
- **EventStream** 用一个无锁的生产者-消费者队列优雅地桥接了异步生产和异步消费。
- **延迟加载**确保只有实际用到的 Provider 才会被导入，避免启动时的无谓开销。

有了这层统一抽象，Agent 循环只需要一个 Model 对象就能与任何 Provider 通信。但 Agent 循环本身——如何编排 LLM 调用和工具执行、如何管理状态、如何处理中断和重试——是一个独立的复杂问题。

下一章，我们进入 pi-agent-core，看看 Agent 运行时是怎么实现的。

---

### 质检报告

**讲解节奏**
- [x] 先讲 pi-ai 解决什么问题（不同 Provider 的格式差异），再讲怎么解决
- [x] 每个组件先讲"它是什么"，再展开内部实现

**周边知识**
- [x] 解释了为什么需要适配器模式（API 格式、流式协议、工具调用 ID 格式差异）
- [x] 解释了为什么需要延迟加载（SDK 启动开销）
- [x] 解释了 Prompt Caching 的背景（区分 cacheRead/cacheWrite 的原因）
- [x] 解释了 Claude Code 兼容技巧的动机

**讲透了吗**
- [x] EventStream 的推/拉模型完整展开
- [x] transformMessages 的两轮处理逻辑完整覆盖
- [x] 延迟加载的 ||= 模式和事件转发机制完整解释
- [x] 没有跳步

**过渡自然吗**
- [x] 章头衔接上一章（"上一章我们看到 Agent 循环通过调用 streamSimple() 与 LLM 通信"）
- [x] 章尾引出下一章（"下一章，我们进入 pi-agent-core"）
- [x] 章内小节用问题驱动衔接

**准确吗**
- [x] 所有代码和类型定义已对照源码验证
- [x] Provider 数量和 API 类型已验证
- [x] 思考预算默认值已验证

**读得下去吗**
- [x] 术语首次出现有解释（EventStream、适配器模式、延迟加载等）
- [x] 每段代码有上下文解释
- [x] 每张图有文字讲解

**勘误建议**
无
