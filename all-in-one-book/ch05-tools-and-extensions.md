# 第五章 工具系统与扩展机制

上一章我们看到 AgentSession 创建时会加载工具和扩展。这一章打开这两个维度，看看 Pi 的 7 个内置工具是怎么实现的，以及扩展系统如何让第三方代码深度介入 Agent 的行为。

---

## 5.1 工具系统全景

Pi 提供 7 个内置工具，分为两个预设组：

```typescript
// packages/coding-agent/src/core/tools/index.ts
const codingTools = [readTool, bashTool, editTool, writeTool];    // 默认激活
const readOnlyTools = [readTool, grepTool, findTool, lsTool];     // 安全子集
```

| 工具 | 作用 | 默认激活 |
|------|------|---------|
| **read** | 读取文件内容（支持偏移和行数限制） | ✅ |
| **bash** | 执行 Bash 命令 | ✅ |
| **edit** | 编辑文件（精确文本替换） | ✅ |
| **write** | 创建新文件 | ✅ |
| **grep** | 搜索文件内容（正则表达式） | ❌ |
| **find** | 按名称模式查找文件 | ❌ |
| **ls** | 列出目录内容 | ❌ |

默认只激活 4 个"编码工具"——这是一个有意的设计选择。`grep` 和 `find` 不在默认集中，因为 LLM 在大多数场景下用 `bash` + `grep`/`find` 命令就能完成搜索任务，单独的工具定义会增加系统提示词的长度和工具选择的复杂性。

每个工具遵循统一的接口——**ToolDefinition**。这是 pi-coding-agent 对 pi-agent-core 的 `AgentTool` 的扩展：

```typescript
interface ToolDefinition<TParams, TDetails> {
  name: string;                    // 工具名
  label: string;                   // UI 显示名
  description: string;             // 给 LLM 看的描述
  promptSnippet?: string;          // 系统提示词中的一句话说明
  promptGuidelines?: string[];     // 系统提示词中的使用指南
  parameters: TParams;             // TypeBox schema
  execute: (toolCallId, params, signal, onUpdate, ctx) => Promise<AgentToolResult<TDetails>>;
  renderCall?: (args, theme, ctx) => Component;    // 自定义调用渲染
  renderResult?: (result, opts, theme, ctx) => Component;  // 自定义结果渲染
}
```

相比 `AgentTool`，`ToolDefinition` 多了 `promptSnippet`（注入系统提示词的片段）、`promptGuidelines`（使用指南）、和自定义渲染方法。这些额外字段让工具不仅能被 LLM 调用，还能影响系统提示词的构建和 UI 的呈现。

---

## 5.2 工具的可插拔操作层

一个值得注意的设计是**每个工具都有一个可插拔的操作接口**（Operations）。以 read 工具为例：

```typescript
// packages/coding-agent/src/core/tools/read.ts
interface ReadOperations {
  readFile: (absolutePath: string) => Promise<Buffer>;
  access: (absolutePath: string) => Promise<void>;
  detectImageMimeType?: (absolutePath: string) => Promise<string | null>;
}
```

默认实现使用 Node.js 的 `fs` 模块读取本地文件。但通过覆盖 `operations`，可以把文件读取重定向到远程系统（比如 SSH）。

Mom（Slack 机器人，第 9 章）正是利用了这个设计——它把工具操作委托给 Docker 容器，在沙箱中执行：

```typescript
const readTool = createReadTool(cwd, {
  operations: {
    readFile: (path) => dockerExec(`cat ${path}`),  // 在容器内读取
    access: (path) => dockerExec(`test -r ${path}`),
  },
});
```

这个抽象让工具的**业务逻辑**（参数验证、输出截断、结果格式化）和**I/O 操作**（实际读文件、执行命令）分离。

---

## 5.3 read 工具：文件读取

read 工具的参数 schema：

```typescript
const readSchema = Type.Object({
  path: Type.String({ description: "Path to the file to read" }),
  offset: Type.Optional(Type.Number({ description: "Line number to start reading from (1-indexed)" })),
  limit: Type.Optional(Type.Number({ description: "Maximum number of lines to read" })),
});
```

执行流程：

1. **路径解析**：相对路径 → 绝对路径（基于 cwd）
2. **权限检查**：调用 `operations.access()` 确认文件可读
3. **图像检测**：如果文件是图像（PNG/JPEG/GIF/WebP），返回 base64 编码的 `ImageContent`，不作为文本读取
4. **文件读取**：调用 `operations.readFile()` 获取文件内容
5. **截断处理**：如果文件超过限制（默认 2000 行或 250KB），从头部截断并标记
6. **行号添加**：给每行加上行号前缀（`1. `、`2. `），方便 LLM 引用

返回值中的 `details` 包含截断信息（是否被截断、总行数、总字节数），UI 用这些信息决定是否显示"文件被截断"的提示。

---

## 5.4 bash 工具：命令执行

bash 工具是 Pi 最强大也最危险的工具——它能执行任意 Bash 命令。

```typescript
const bashSchema = Type.Object({
  command: Type.String({ description: "Bash command to execute" }),
  timeout: Type.Optional(Type.Number({ description: "Timeout in seconds" })),
});
```

执行流程：

1. **创建子进程**：使用 Node.js 的 `spawn()` 启动一个 shell 子进程
2. **流式输出**：通过 `onData` 回调实时推送输出，调用者（Agent 循环）通过 `tool_execution_update` 事件把输出展示给用户
3. **输出保存**：同时把完整输出写入临时文件（用于超长输出的回看）
4. **截断**：返回给 LLM 的输出会被截断（默认尾部 2000 行或 250KB），但完整输出保存在临时文件中
5. **超时处理**：如果指定了 timeout，超时后杀掉进程树
6. **中止支持**：通过 AbortSignal 支持用户中途取消

bash 工具的一个重要细节是**环境变量隔离**——它不会把 Pi 进程的 API Key 环境变量（`ANTHROPIC_API_KEY` 等）传递给子进程，防止 LLM 编写的脚本无意中泄露密钥。

---

## 5.5 edit 工具：精确文本替换

edit 工具不是"重写整个文件"——它做的是**精确的文本替换**。这个设计选择很重要：对于大文件，让 LLM 输出整个文件内容既浪费 token 又容易出错。精确替换只需要提供要修改的部分。

```typescript
const editSchema = Type.Object({
  path: Type.String({ description: "Path to the file to edit" }),
  oldText: Type.Optional(Type.String({ description: "Exact text to replace" })),
  newText: Type.Optional(Type.String({ description: "Replacement text" })),
  edits: Type.Optional(Type.Array(replaceEditSchema, {
    description: "Multiple separate, disjoint regions to edit simultaneously"
  })),
});
```

edit 工具支持两种模式：
- **单替换模式**：`oldText` + `newText`，替换一处
- **多替换模式**：`edits` 数组，同时替换多个不相邻的区域

执行时的关键步骤：
1. 读取原始文件内容
2. 检测行尾格式（`\n` vs `\r\n`），统一为 `\n` 处理
3. 在原始内容中查找每个 `oldText` 的精确匹配位置
4. 验证匹配唯一性——如果 `oldText` 在文件中出现多次，报错（避免歧义替换）
5. 应用替换
6. 恢复原始行尾格式
7. 写入文件
8. 返回 diff（差异）作为 details，UI 用来渲染彩色的变更预览

---

## 5.6 文件变更队列

当 LLM 同时调用多个 edit/write 操作作用于同一个文件时，并行执行可能导致竞态条件（两个编辑都基于同一个原始版本，后写入的覆盖先写入的修改）。

`withFileMutationQueue()` 解决了这个问题——它为每个文件路径维护一个队列，确保同一个文件的写操作串行执行：

```typescript
const queuedTools = withFileMutationQueue(toolDefinitions);
// 现在 edit("file.ts", ...) 和 write("file.ts", ...) 不会并行执行
// 但 edit("a.ts", ...) 和 edit("b.ts", ...) 仍然可以并行
```

---

## 5.7 扩展系统：全景

工具系统让 Agent 有了"手"，扩展系统让它有了"可定制的大脑"。

Pi 的扩展系统允许第三方 TypeScript 模块通过**事件钩子**和 **API 注册**深度介入 Agent 的行为。扩展可以：

- 注册新工具（类似内置工具）
- 拦截和修改工具调用（安全审查、行为增强）
- 修改输入和输出（在消息到达 LLM 之前变换）
- 注册斜杠命令（交互模式的命令）
- 注册自定义 Provider（本地模型、代理服务）
- 自定义 UI（添加 widget、修改编辑器）

```typescript
// 一个最小的扩展示例
import { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { Type } from "@sinclair/typebox";

export default function (pi: ExtensionAPI) {
  // 注册工具
  pi.registerTool({
    name: "greet",
    label: "Greet",
    description: "Say hello",
    parameters: Type.Object({ name: Type.String() }),
    async execute(toolCallId, params) {
      return {
        content: [{ type: "text", text: `Hello, ${params.name}!` }],
        details: {},
      };
    },
  });

  // 拦截危险操作
  pi.on("tool_call", async (event, ctx) => {
    if (event.toolName === "bash" && event.input.command?.includes("rm -rf /")) {
      return { block: true, reason: "Blocked dangerous command" };
    }
  });
}
```

---

## 5.8 扩展加载：Jiti 的魔法

扩展是 TypeScript 文件，但 Pi 在运行时需要加载和执行它们。Node.js 不能直接运行 TypeScript——通常需要先编译成 JavaScript。

Pi 使用了 **jiti**（一个 just-in-time TypeScript 转译器的定制 fork）来解决这个问题。jiti 在 `import()` 时动态转译 TypeScript，无需预编译步骤。

```typescript
// packages/coding-agent/src/core/extensions/loader.ts
import { createJiti } from "@mariozechner/jiti";

const jiti = createJiti(import.meta.url, {
  virtualModules: {
    "@sinclair/typebox": _bundledTypebox,
    "@mariozechner/pi-agent-core": _bundledPiAgentCore,
    "@mariozechner/pi-ai": _bundledPiAi,
    "@mariozechner/pi-tui": _bundledPiTui,
    "@mariozechner/pi-coding-agent": _bundledPiCodingAgent,
  },
});
```

注意 `virtualModules` 配置——这是 jiti fork 的关键特性。当扩展代码 `import { Type } from "@sinclair/typebox"` 时，jiti 不会去 node_modules 中查找（那里可能没有），而是直接使用 Pi 预先导入的包。

这个设计解决了两个问题：
1. **Bun 编译场景**：Pi 可以编译成 Bun 二进制文件，此时没有 node_modules 目录。virtualModules 让扩展仍然能导入这些包。
2. **版本一致性**：扩展使用和 Pi 完全相同版本的依赖，避免版本不兼容。

---

## 5.9 扩展发现与加载

Pi 在两个位置自动发现扩展：

```
~/.pi/agent/extensions/     ← 全局扩展
.pi/extensions/             ← 项目扩展（当前工作目录）
```

还可以通过 CLI 参数 `-e ./my-extension.ts` 显式加载。

每个扩展是一个导出默认函数的 TypeScript 模块。加载过程：

```
发现扩展文件路径
    ↓
jiti.import() 动态转译并执行
    ↓
获取默认导出函数 (ExtensionFactory)
    ↓
调用 factory(extensionAPI)
    ↓
扩展通过 API 注册工具/命令/事件处理器
```

扩展还支持**热重载**——用户执行 `/reload` 命令时，Pi 会重新加载所有扩展，替换 ExtensionRunner 实例。扩展可以在重载前保存状态（通过 `CustomEntry`），重载后恢复。

---

## 5.10 事件钩子体系

扩展通过 `pi.on(eventType, handler)` 注册事件处理器。Pi 的事件体系覆盖了 Agent 生命周期的关键节点：

**Agent 事件**：

| 事件 | 时机 | 可以做什么 |
|------|------|-----------|
| `before_agent_start` | Agent 循环启动前 | 注入额外上下文、修改系统提示词 |
| `agent_start` / `agent_end` | Agent 开始/结束 | 日志、状态记录 |
| `turn_start` / `turn_end` | 每轮 LLM 交互 | 统计、审计 |
| `input` | 用户输入到达、发给 LLM 之前 | 修改输入、完全拦截 |

**工具事件**：

| 事件 | 时机 | 可以做什么 |
|------|------|-----------|
| `tool_call` | 工具即将执行 | 拦截（返回 `{ block: true }`）、审查参数 |
| `tool_result` | 工具执行完毕 | 修改结果、添加额外信息 |
| `bash` / `read` / `write` / `edit` | 特定工具的调用 | 针对特定工具的拦截和修改 |

**会话事件**：

| 事件 | 时机 | 可以做什么 |
|------|------|-----------|
| `session_start` | 会话创建/加载 | 初始化扩展状态 |
| `session_before_compact` | 压缩前 | 自定义压缩逻辑 |
| `session_before_fork` | 分支前 | 保存分支上下文 |
| `session_before_switch` | 切换会话前 | 清理资源 |

事件处理器的返回值控制后续行为。比如 `tool_call` 事件的处理器可以返回 `{ block: true, reason: "..." }` 来阻止工具执行，或返回 `undefined` 放行。

---

## 5.11 Skills：不写代码的扩展

对于不需要编程能力的用户，Pi 提供了更轻量的扩展方式——**Skills**（技能）。

一个 Skill 就是一个包含 YAML frontmatter 的 Markdown 文件：

```markdown
---
name: kubernetes
description: Kubernetes cluster management
---

# Kubernetes Skill

When working with Kubernetes clusters:
1. Always use `kubectl` with explicit namespace
2. Check pod status before modifying deployments
3. ...
```

Skill 的内容会被注入到系统提示词中——LLM 会"学到"这些知识和操作指南。

Skills 遵循 [Agent Skills](https://agentskills.io) 开放规范，可以在不同的 Agent 框架间共享。Pi 在以下位置发现 Skills：

```
~/.pi/agent/skills/        ← 全局技能
.pi/skills/                ← 项目技能
```

用户也可以通过 `--skill path` 显式加载，或在交互模式中用 `/skill:name args` 调用。

---

## 5.12 小结

Pi 的工具和扩展系统体现了一个清晰的分层设计：

- **7 个内置工具**覆盖了编码 Agent 的基本操作（读、写、编辑、执行）。每个工具有可插拔的操作层，支持本地和远程执行。
- **文件变更队列**防止并发写入同一文件的竞态条件。
- **扩展系统**通过 jiti 动态加载 TypeScript 模块，用 virtualModules 解决依赖问题。
- **事件钩子**覆盖 Agent 生命周期的关键节点，扩展可以拦截、修改、增强。
- **Skills** 是零代码的扩展方式，通过 Markdown 向 LLM 注入领域知识。

到目前为止，我们已经走完了 Pi 的核心主线：pi-ai（通信）→ pi-agent-core（运行时）→ pi-coding-agent（CLI + 会话 + 工具 + 扩展）。但还有一个重要的独立维度我们没有触及——用户看到的东西是怎么渲染出来的？

下一章，我们进入 pi-tui，Pi 的终端 UI 框架。

---

### 质检报告

**讲解节奏**
- [x] 先讲工具系统全景（7 个工具的分类和预设），再逐个展开
- [x] 扩展系统先讲"是什么"和能做什么，再讲内部实现

**周边知识**
- [x] 解释了为什么默认只激活 4 个工具（系统提示词长度和复杂性）
- [x] 解释了为什么用精确文本替换而不是重写文件（token 效率和准确性）
- [x] 解释了为什么需要 jiti（运行时 TypeScript 转译）和 virtualModules（Bun 兼容和版本一致性）

**讲透了吗**
- [x] 可插拔操作层的设计和 Mom 的使用场景
- [x] edit 工具的单替换/多替换模式
- [x] 文件变更队列的竞态条件解决
- [x] 扩展加载的完整流程
- [x] 事件钩子的分类和返回值语义

**过渡自然吗**
- [x] 章头衔接上一章（"上一章看到 AgentSession 加载工具和扩展"）
- [x] 章尾引出下一章（"下一章进入 pi-tui"）
- [x] 章内从工具 → 扩展的过渡自然

**准确吗**
- [x] 代码和类型定义已对照源码验证
- [x] 工具列表和预设已验证
- [x] 事件类型已验证

**读得下去吗**
- [x] 用表格展示工具列表和事件列表
- [x] 用具体代码示例展示扩展写法
- [x] 术语首次出现有解释

**勘误建议**
无
