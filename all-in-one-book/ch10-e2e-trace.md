# 第十章 端到端追踪 — 一次真实编码任务的完整旅程

第一章我们用黑盒视角走了一遍数据流。现在，经过九章的深入学习，我们有了打开每个黑盒的知识。这一章用**白盒视角**追踪一次真实的编码任务，串联全书所有模块。

---

## 场景

用户在一个 Node.js 项目的根目录执行：

```bash
pi "src/utils.ts 中的 formatDate 函数有 bug，月份少了 1，修一下"
```

我们追踪从用户按下 Enter 到文件被修改的完整过程。

---

## 阶段一：CLI 启动与 AgentSession 创建

**1. 进程启动**（`cli.ts`）

```
process.title = "pi"
设置 HTTP 代理 (undici EnvHttpProxyAgent)
调用 main(["src/utils.ts 中的 formatDate 函数有 bug，月份少了 1，修一下"])
```

**2. 参数解析**（`main.ts` → `parseArgs()`）

没有 `--model`、`--provider` 等标志，只有一个位置参数（用户的消息）。

**3. 模型解析**（`resolveCliModel()`）

未指定模型 → 检测环境中可用的 API Key → 找到 `ANTHROPIC_API_KEY` → 使用 Anthropic 的默认模型 `claude-opus-4-6`。

**4. 创建 AgentSession**（`createAgentSession()`）

- 加载全局设置（`~/.pi/agent/settings.json`）和项目设置（`.pi/settings.json`）
- 加载扩展（`~/.pi/agent/extensions/` 和 `.pi/extensions/`）
- 注册内置工具：read、bash、edit、write
- 构建系统提示词：默认模板 + 项目的 `AGENTS.md` + 技能
- 创建 Agent 实例（pi-agent-core）
- 创建 SessionManager → 新的 JSONL 会话文件

**5. 进入交互模式**

无 `--mode` 标志 → 启动 TUI。但用户提供了初始消息，所以 Agent 立即开始处理（不等待 Editor 输入）。

---

## 阶段二：第一轮 LLM 调用

**6. AgentSession.prompt()**

用户的字符串被包装为 UserMessage：

```typescript
{
  role: "user",
  content: [{ type: "text", text: "src/utils.ts 中的 formatDate 函数有 bug，月份少了 1，修一下" }],
  timestamp: 1711567820628
}
```

**7. Agent._runLoop() → runAgentLoop()**

创建 AgentContext，配置 AgentLoopConfig（包含 convertToLlm、getSteeringMessages 等回调）。

**8. agentLoop 内层循环 → streamAssistantResponse()**

- `convertToLlm()` 过滤自定义消息 → 只剩 UserMessage
- 构建 Context：系统提示词 + [UserMessage] + 4 个工具
- 调用 `streamSimple(model, context, options)`

**9. pi-ai 分发**（`stream.ts`）

- `resolveApiProvider("anthropic-messages")` → 从 API Registry 查找
- 延迟加载：首次调用 → `import("./anthropic.js")` → 缓存

**10. Anthropic Provider 处理**

- `transformMessages()`：归一化消息（这是首次调用，没有跨模型问题）
- 工具名转换：`read` → `Read`（Claude Code 兼容）
- 构建 Anthropic SDK 请求 → 发起流式 HTTP 请求

**11. 流式事件**

LLM 思考后决定先读取文件。事件序列：

```
start → text_delta("我") → text_delta("来看看") → text_delta(" formatDate 函数")
→ toolcall_start → toolcall_delta({"path":"src/utils.ts"})
→ toolcall_end({id:"toolu_01", name:"Read", arguments:{path:"src/utils.ts"}})
→ done(reason: "toolUse")
```

Provider 把 `Read` 转回 `read`（工具名反转换）。

**12. Agent 事件发射**

```
message_start(assistant) → message_update(text_delta) × N
→ message_update(toolcall_end) → message_end(assistant)
```

**13. TUI 渲染**

InteractiveMode 收到 `message_update` 事件 → 追加文本到 Markdown 组件 → `tui.requestRender()` → 差分渲染到终端。用户实时看到"我来看看 formatDate 函数"逐字出现。

**14. SessionManager 保存**

UserMessage 和 AssistantMessage 各写入 JSONL 文件一行。

---

## 阶段三：工具执行

**15. agentLoop 检测到 toolCall**

`stopReason: "toolUse"` → 提取 `toolCall` → 进入工具执行。

**16. prepareToolCall()**

- 在 `context.tools` 中查找 `read` → 找到
- 验证参数 `{path: "src/utils.ts"}` → 通过 TypeBox 验证
- `beforeToolCall()` → 扩展没有拦截 → 放行

**17. executePreparedToolCall()**

read 工具执行：
- 解析路径：`src/utils.ts` → `/project/src/utils.ts`
- 检查权限 → 可读
- 非图像文件 → 读取文本内容
- 添加行号前缀（`1. import ...`, `2. export function formatDate(...) {`, ...）
- 检查截断 → 文件较小，不截断

返回：

```typescript
{
  content: [{ type: "text", text: "1. import ...\n2. export function formatDate(date: Date): string {\n3.   const month = date.getMonth();\n..." }],
  details: { truncated: false, totalLines: 25, totalBytes: 580 }
}
```

**18. finalizeExecutedToolCall()**

- `afterToolCall()` → 扩展没有修改
- 构建 ToolResultMessage
- 发射 `tool_execution_end` 事件
- TUI 渲染工具结果（文件内容带语法高亮）

---

## 阶段四：第二轮 LLM 调用

**19. 内层循环继续**

有工具调用 → `hasMoreToolCalls = true` → 回到循环开头。

**20. 再次 streamAssistantResponse()**

此时消息历史：`[UserMessage, AssistantMessage(含 toolCall), ToolResultMessage(文件内容)]`

LLM 看到了代码，发现第 3 行 `date.getMonth()` 确实缺少 `+ 1`。

回复包含 edit 工具调用：

```json
{
  "type": "toolCall",
  "name": "edit",
  "arguments": {
    "path": "src/utils.ts",
    "oldText": "const month = date.getMonth();",
    "newText": "const month = date.getMonth() + 1;"
  }
}
```

**21. edit 工具执行**

- 读取 `src/utils.ts` 原始内容
- 检测行尾格式 → `\n`
- 搜索 `"const month = date.getMonth();"` → 找到唯一匹配
- 替换为 `"const month = date.getMonth() + 1;"`
- 写回文件
- 生成 diff 作为 details

```diff
- const month = date.getMonth();
+ const month = date.getMonth() + 1;
```

TUI 渲染彩色的 diff 预览。

---

## 阶段五：最终回复与收尾

**22. 第三轮 LLM 调用**

消息历史现在有 5 条。LLM 看到编辑成功，回复：

```
"已修复 `src/utils.ts` 中 `formatDate` 函数的 bug。JavaScript 的 `Date.getMonth()` 返回 0-11，需要 +1 才能得到实际月份。"
```

`stopReason: "stop"` → 没有工具调用 → 内层循环退出。

**23. 检查 follow-up**

`getFollowUpMessages()` → 空 → 外层循环退出。

**24. agent_end 事件**

- AgentSession 保存最终状态到 JSONL
- TUI 恢复 Editor 焦点
- 状态栏更新：显示本次对话的 token 用量和费用

用户看到完整的对话、彩色的 diff、清晰的解释。文件已经修改完毕。整个过程大约 5-10 秒。

---

## 完整的调用链路

```
用户按 Enter
  → cli.ts → main.ts → AgentSession.prompt()
    → Agent._runLoop() → agentLoop()
      → streamAssistantResponse()
        → streamSimple() → resolveApiProvider() → anthropic.stream()
          → transformMessages() → Anthropic SDK → HTTP 流式请求
          → AssistantMessageEvent 流
        ← 返回 AssistantMessage (stopReason: "toolUse")
      → executeToolCalls()
        → prepareToolCall() → read.execute() → finalizeExecutedToolCall()
      → streamAssistantResponse()  (第二轮)
        → edit 工具调用
      → executeToolCalls()
        → edit.execute() → 文件修改完成
      → streamAssistantResponse()  (第三轮)
        ← AssistantMessage (stopReason: "stop")
    ← agentLoop 退出
  ← AgentSession 保存会话
  ← TUI 恢复用户输入
```

从 pi-ai 的消息类型，到 EventStream 的推/拉机制，到 agentLoop 的嵌套循环，到工具的三阶段执行，到 TUI 的差分渲染——每一层都在这 5 秒钟里各司其职。这就是 Pi 的实现原理。

---

### 质检报告

**讲解节奏**
- [x] 按时间顺序追踪，每一步标注参与的模块
- [x] 没有跳步

**周边知识**
- [x] 解释了 `getMonth()` 返回 0-11 的 JavaScript 特性
- [x] 前序章节的概念在这里直接使用，不重复解释

**讲透了吗**
- [x] 从 CLI 启动到文件修改完成的完整链路
- [x] 每一步的数据形态变化
- [x] 三轮 LLM 调用的完整过程

**过渡自然吗**
- [x] 章头衔接第一章（"第一章用黑盒，这一章用白盒"）
- [x] 章尾闭合全书

**准确吗**
- [x] 调用链路与前九章的分析一致
- [x] 工具名转换（read → Read → read）流程正确

**读得下去吗**
- [x] 用连续的时间线讲述（5 个阶段 24 步）
- [x] 最后用一个简洁的调用链路图总结

**勘误建议**
无
