# 第四章 pi-coding-agent 核心 — 从 CLI 到 AgentSession

上一章我们理解了 Agent 运行时的内部机制——循环、工具执行、事件体系。但 Agent 运行时只是一个"引擎"。用户在终端输入 `pi "修复这个 bug"` 时，从字符串到引擎启动之间，还有大量的工作要做：解析命令行参数、选择模型、加载会话历史、构建系统提示词、初始化扩展……

这些工作由 pi-coding-agent 完成。它是整个 monorepo 中最复杂的包——不是因为它的算法复杂，而是因为它要处理大量的**用户侧关注点**：配置、持久化、交互、可扩展性。

这一章我们聚焦于 coding-agent 的核心层：从 CLI 入口到 `AgentSession` 的创建和运行。

---

## 4.1 CLI 启动链

当用户执行 `pi "修复这个 bug"` 时，进程启动的调用链是：

```
pi (可执行文件 → dist/cli.js)
  │
  ├─ 设置进程标题：process.title = "pi"
  ├─ 禁用 Node.js 警告
  ├─ 配置 HTTP 代理（通过 undici 的 EnvHttpProxyAgent）
  │
  └─ 调用 main(process.argv.slice(2))
       │
       ├─ parseArgs()：解析 CLI 参数
       ├─ 各种前置检查（--help, --version, --list-models 等）
       ├─ selectConfig()：选择配置目录
       ├─ 创建核心服务：
       │   ├─ SettingsManager（设置管理）
       │   ├─ AuthStorage（认证存储）
       │   ├─ ModelRegistry（模型注册表）
       │   ├─ ResourceLoader（资源发现）
       │   └─ SessionManager（会话管理）
       │
       ├─ resolveCliModel()：解析 --model/--provider 指定的模型
       │
       ├─ createAgentSession()：创建核心的 AgentSession
       │   ├─ 加载扩展
       │   ├─ 注册工具
       │   ├─ 构建系统提示词
       │   └─ 创建 Agent 实例
       │
       └─ 进入运行模式：
           ├─ interactive（默认）→ TUI 交互
           ├─ rpc → JSON-line 协议
           └─ print → 纯文本输出
```

`main.ts` 的注释很好地概括了它的职责：*"处理 CLI 参数解析并将其转换为 `createAgentSession()` 选项。SDK 做真正的重活。"*

---

## 4.2 参数解析：30+ 个 CLI 标志

`parseArgs()` 把 `process.argv` 解析为一个结构化的 `Args` 对象。Pi 的 CLI 支持 30+ 个标志，按功能分组：

| 类别 | 标志 | 作用 |
|------|------|------|
| **模型选择** | `--provider`, `--model`, `--models` | 指定 Provider、模型或可循环的模型列表 |
| **会话控制** | `-c`/`--continue`, `-r`/`--resume`, `--session`, `--fork`, `--no-session` | 继续上次会话、交互选择会话、指定会话路径 |
| **工具** | `--tools`, `--no-tools` | 指定可用工具子集 |
| **扩展** | `-e`/`--extension`, `--no-extensions` | 加载扩展模块 |
| **思考** | `--thinking` | 设置推理深度（off/minimal/low/medium/high/xhigh） |
| **输出** | `--mode text\|json\|rpc`, `--print`, `--export` | 控制输出格式 |
| **系统提示词** | `--system-prompt`, `--append-system-prompt` | 自定义或追加系统提示词 |

最常用的参数之一是 `--models`（注意是复数），它支持**作用域模型循环**：

```bash
pi --models "claude-opus-4-6:high,gpt-5.4:medium,gemini-2.5-pro"
```

这让用户可以在交互过程中通过快捷键（Ctrl+P）在预选的模型之间快速切换。冒号后的后缀指定每个模型的思考深度。

---

## 4.3 模型解析：从字符串到 Model 对象

用户通过 `--model` 传入的是一个字符串（比如 `"claude-opus-4-6"` 或 `"anthropic/claude-opus-4-6:high"`），但系统需要一个带完整元数据的 `Model` 对象。`resolveCliModel()` 负责这个转换。

解析逻辑比看起来复杂，因为它要处理多种格式：

```
"claude-opus-4-6"                    → 在所有 Provider 中搜索匹配的模型
"anthropic/claude-opus-4-6"          → 指定 Provider + 模型 ID
"claude-opus-4-6:high"               → 模型 + 思考级别
"openai/gpt-5.1-codex:extended"      → Provider + 模型 + OpenRouter 后缀
```

其中最棘手的是**冒号的歧义**：冒号可能是思考级别的分隔符（`:high`），也可能是模型 ID 的一部分（OpenRouter 的 `:extended` 后缀）。`parseModelPattern()` 用递归策略处理——先尝试把整个字符串当作模型 ID，如果匹配不到，再按最后一个冒号拆分，检查后缀是否是合法的思考级别。

如果用户没有指定模型，Pi 会根据检测到的 API Key 自动选择默认模型。每个 Provider 都有预设的默认模型：

```typescript
const defaultModelPerProvider = {
  anthropic: "claude-opus-4-6",
  openai: "gpt-5.4",
  google: "gemini-2.5-pro",
  mistral: "devstral-medium-latest",
  // ... 20+ providers
};
```

---

## 4.4 AgentSession：coding-agent 的中枢

`AgentSession` 是 coding-agent 最重要的抽象。它不是 Agent 运行时的 `Agent` 类——它是在 `Agent` 之上的更高层封装，负责把 Agent 运行时和所有"用户侧"功能连接在一起。

```
┌────────────────────────────────────────────────────┐
│                   AgentSession                     │
│                                                    │
│  ┌───────────────────────────────────────┐        │
│  │           Agent (pi-agent-core)        │        │
│  │  · Agent Loop                          │        │
│  │  · Tool Execution                      │        │
│  │  · Events                              │        │
│  └───────────────────────────────────────┘        │
│                                                    │
│  ┌─────────────┐  ┌──────────────────┐            │
│  │ SessionMgr  │  │ ExtensionRunner  │            │
│  │ (会话持久化) │  │ (扩展集成)       │            │
│  └─────────────┘  └──────────────────┘            │
│                                                    │
│  ┌─────────────┐  ┌──────────────────┐            │
│  │ ModelRegistry│  │ SettingsManager  │            │
│  │ (模型+认证) │  │ (用户设置)       │            │
│  └─────────────┘  └──────────────────┘            │
│                                                    │
│  · 自动 Compaction（上下文溢出管理）                │
│  · 自动重试（网络错误恢复）                         │
│  · 模型循环切换                                    │
│  · Bash 执行器集成                                 │
│  · 会话分支与切换                                   │
└────────────────────────────────────────────────────┘
```

`AgentSession` 的创建通过 `createAgentSession()` 工厂函数：

```typescript
const { agentSession, extensionsResult } = await createAgentSession({
  cwd: process.cwd(),
  agentDir: "~/.pi/agent",
  modelRegistry,
  resourceLoader,
  authStorage,
  // ... 更多选项
});
```

这个函数内部做了大量初始化工作：发现并加载扩展、注册内置工具和扩展工具、构建系统提示词、创建底层 Agent 实例。

---

## 4.5 会话管理：JSONL 文件中的树

Pi 的会话持久化方案是一个经过深思熟虑的设计决策。

**问题**：需要保存对话历史，支持继续对话（`-c`）、回溯到历史某一点、创建分支。

**常见方案对比**：
- **数据库**（SQLite/PostgreSQL）：功能强大但引入外部依赖。Pi 的设计哲学是本地优先、零外部依赖。
- **JSON 文件**：每次保存需要重写整个文件。对于长对话（几百条消息）性能不好，且崩溃时可能丢失全部数据。
- **JSONL 文件**：每条记录一行，追加写入。崩溃只丢最后一条记录，之前的数据安全。

Pi 选择了 **JSONL + 树结构**的方案。每条记录（entry）有 `id` 和 `parentId`，构成一棵树：

```json
{"type":"session","version":3,"id":"abc-123","timestamp":"2026-03-27T19:10:20Z","cwd":"/project"}
{"type":"message","id":"e1","parentId":null,"message":{"role":"user",...},"timestamp":"..."}
{"type":"message","id":"e2","parentId":"e1","message":{"role":"assistant",...},"timestamp":"..."}
{"type":"message","id":"e3","parentId":"e2","message":{"role":"toolResult",...},"timestamp":"..."}
{"type":"message","id":"e4","parentId":"e3","message":{"role":"assistant",...},"timestamp":"..."}
```

为什么要树结构而不是简单的线性列表？因为 Pi 支持**分支**——用户可以回到历史中的某一点，从那里开始一条新的对话线路，而不丢失原来的对话。

比如用户在 `e4` 之后觉得 LLM 的方向不对，想回到 `e2` 重新来：

```json
...前面的 e1-e4 不变...
{"type":"message","id":"e5","parentId":"e2","message":{"role":"user","content":"换一种方式..."},"timestamp":"..."}
{"type":"message","id":"e6","parentId":"e5","message":{"role":"assistant",...},"timestamp":"..."}
```

`e5` 的 `parentId` 是 `e2` 而不是 `e4`——它从 `e2` 分叉出一条新路径。通过 `/tree` 命令，用户可以在这些分支之间导航。

`SessionManager` 是管理会话文件的核心类。它在内存中维护一个 `byId` 索引（Map<id, entry>），追踪当前的 `leafId`（最新条目），并提供方法来：

- **追加消息**：`appendMessage(message)` → 写一行 JSONL
- **创建新会话**：`newSession()` → 新文件 + header
- **切换会话**：`setSessionFile(path)` → 加载并索引
- **构建上下文**：`buildSessionContext()` → 从 leaf 沿 parentId 链向上回溯，收集路径上的消息
- **获取树**：`getTree()` → 返回完整的树结构供 UI 渲染

---

## 4.6 会话条目类型

JSONL 文件中不只有消息，还有多种条目类型，每种记录不同的状态变化：

```typescript
type SessionEntry =
  | SessionMessageEntry          // 用户/助手/工具结果消息
  | ThinkingLevelChangeEntry     // 思考级别变更
  | ModelChangeEntry             // 模型切换
  | CompactionEntry              // 上下文压缩记录
  | BranchSummaryEntry           // 分支摘要
  | CustomEntry                  // 扩展自定义数据（不参与 LLM 上下文）
  | CustomMessageEntry           // 扩展自定义消息（参与 LLM 上下文）
  | LabelEntry                   // 用户书签
  | SessionInfoEntry;            // 元数据（会话名称等）
```

`CompactionEntry` 特别值得注意——它记录了一次上下文压缩的边界。当 `buildSessionContext()` 重建上下文时，遇到 `CompactionEntry` 就知道：这个点之前的消息已经被总结了，用总结替代原始消息。

---

## 4.7 上下文压缩（Compaction）

LLM 的上下文窗口有大小限制（比如 Claude 的 200K tokens）。长时间的对话会逐渐填满窗口。当接近限制时，有两个选择：要么报错终止，要么想办法"腾出空间"。

Pi 选择了**压缩**（compaction）：用 LLM 对旧消息生成摘要，用摘要替代原始消息。原始消息仍保存在 JSONL 文件中（`/tree` 可以回溯），但发送给 LLM 的上下文只包含摘要 + 最近的消息。

压缩有三种触发方式：

| 触发方式 | 场景 |
|----------|------|
| **手动** | 用户执行 `/compact` 命令 |
| **主动** | 上下文使用率超过阈值（默认 90%），在下一次 LLM 调用前自动触发 |
| **被动** | LLM 返回上下文溢出错误（stopReason 为特定错误），紧急压缩后重试 |

压缩的核心流程：

```
1. 找到切割点：保留最近 N 个 token 的消息，其余待压缩
2. 序列化待压缩的消息为对话文本
3. 提取文件操作记录（哪些文件被读取/修改过）
4. 调用 LLM 生成摘要（用独立的 completeSimple() 调用，不走 Agent 循环）
5. 在 JSONL 中写入 CompactionEntry（包含摘要和切割点）
6. 重新加载会话上下文
```

压缩后的上下文结构变成：

```
[系统提示词]
[上下文压缩摘要]："之前的对话中，用户请求修改 X 文件，我读取了 Y 和 Z..."
[最近的几轮消息]
```

这是一种**有损压缩**——信息必然会丢失。但对于长对话来说，这个 trade-off 是值得的：与其让对话因为窗口溢出而终止，不如丢失一些早期细节。

---

## 4.8 系统提示词构建

系统提示词是影响 LLM 行为的关键因素。Pi 的系统提示词不是一个静态字符串——它由多个部分动态组装：

```typescript
function buildSystemPrompt(options): string {
  // 1. 如果用户提供了自定义系统提示词（--system-prompt），直接使用
  if (options.customPrompt) {
    return customPrompt + appendSection;
  }

  // 2. 否则，用默认模板构建
  return [
    defaultBasePrompt,                    // 角色定义和基本行为准则
    toolSnippets,                         // 每个工具的一句话说明
    guidelines,                           // 使用指南（来自工具和扩展）
    contextFiles,                         // 项目上下文文件（AGENTS.md）
    skills,                               // 技能说明
    appendSection,                        // 用户追加的内容（--append-system-prompt）
  ].join("\n\n");
}
```

**项目上下文文件**是一个巧妙的设计：Pi 会沿着工作目录向上查找 `AGENTS.md`（或 `CLAUDE.md`）文件，把它们拼接起来。这样项目的特定规则（代码风格、测试要求等）会自动注入到系统提示词中。

**技能**（Skills）是遵循 Agent Skills 规范的 Markdown 文件。它们提供领域知识——比如一个 "kubernetes" 技能会告诉 LLM 常见的 kubectl 命令和最佳实践。

---

## 4.9 设置系统：分层覆盖

Pi 的设置分两层：

```
全局设置：~/.pi/agent/settings.json
  ↓ 被覆盖
项目设置：.pi/settings.json
```

`SettingsManager` 负责合并这两层设置。项目设置覆盖全局设置的同名字段。

关键设置包括：

```typescript
interface Settings {
  // 压缩相关
  compaction: {
    enabled: boolean;              // 是否启用自动压缩（默认 true）
    reserveTokens: number;         // 压缩 LLM 调用的 token 预留（默认 16384）
    keepRecentTokens: number;      // 压缩后保留的最近消息 token 数（默认 20000）
  };

  // 重试相关
  retry: {
    enabled: boolean;              // 是否启用自动重试（默认 true）
    maxRetries: number;            // 最大重试次数（默认 3）
    baseDelayMs: number;           // 基础延迟，指数退避（默认 2000ms）
  };

  // 终端相关
  terminal: {
    showImages: boolean;           // 是否显示内联图像（默认 true）
  };

  // 思考预算
  thinkingBudgets: {
    minimal: number;               // minimal 级别的 token 预算
    low: number;
    medium: number;
    high: number;
  };
}
```

---

## 4.10 三种运行模式

AgentSession 创建完成后，`main()` 根据参数选择运行模式：

**交互模式**（默认）：全功能的 TUI 体验。用户在终端中实时看到 LLM 的流式输出、工具执行过程、斜杠命令菜单。这是 Pi 的主要使用方式。详见第 7 章。

**RPC 模式**（`--mode rpc`）：无头的 JSON-line 协议。每行一个 JSON 对象，支持命令/响应/事件三种消息类型。适用于被其他程序集成——比如 web-ui 通过 RPC 模式和 coding-agent 通信。详见第 8 章。

**Print 模式**（`--print` 或管道输入）：最简单的模式，只输出文本。适合脚本和 CI 环境。不接受交互输入。

三种模式共享同一个 `AgentSession`——模式只影响输入来源和输出渲染，不影响核心逻辑。这是一个重要的设计决策：**业务逻辑与 I/O 分离**。

---

## 4.11 小结

pi-coding-agent 的核心层做了这些事：

- **CLI 解析**把用户的命令行输入结构化为配置选项。
- **模型解析**支持多种格式的模型指定，包括 Provider 前缀、思考级别后缀、模糊匹配。
- **AgentSession** 是核心抽象，把 Agent 运行时和会话管理、扩展、设置连接在一起。
- **会话管理**用 JSONL + 树结构实现持久化，支持分支和回溯。
- **上下文压缩**在对话过长时自动触发，用 LLM 摘要替代旧消息，有损但必要。
- **系统提示词**动态组装，自动包含项目上下文和技能。
- **三种运行模式**共享同一个 AgentSession，业务逻辑与 I/O 分离。

但 coding-agent 的另外两大维度——**内置工具**和**扩展系统**——还没展开。用户执行 `read`、`bash`、`edit` 这些工具时，内部到底发生了什么？扩展是怎么加载的，事件钩子是怎么串联的？

下一章，我们深入工具系统与扩展机制。

---

### 质检报告

**讲解节奏**
- [x] 先讲 CLI 启动的全景（调用链），再逐个展开关键组件
- [x] 每个组件先讲"它是什么"再讲内部实现

**周边知识**
- [x] 会话存储方案对比（数据库 vs JSON vs JSONL），解释了为什么选 JSONL
- [x] 解释了为什么需要树结构（支持分支和回溯）
- [x] 解释了为什么需要上下文压缩（窗口限制 + 有损 trade-off）

**讲透了吗**
- [x] CLI 启动链完整覆盖
- [x] 模型解析的冒号歧义问题详细解释
- [x] JSONL 树结构的分支机制用具体例子说明
- [x] 压缩的三种触发方式和流程
- [x] 复杂节点标注"详见第 N 章"

**过渡自然吗**
- [x] 章头衔接上一章（"Agent 运行时只是引擎，用户侧关注点由 coding-agent 处理"）
- [x] 章尾引出下一章（"下一章深入工具系统与扩展机制"）
- [x] 章内小节用 CLI 启动流程自然串联

**准确吗**
- [x] 代码和类型定义已对照源码验证
- [x] 默认模型列表已验证
- [x] 会话条目类型已验证

**读得下去吗**
- [x] 用调用链图示展示启动过程
- [x] 用具体 JSON 展示 JSONL 文件结构
- [x] 术语首次出现有解释

**勘误建议**
无
