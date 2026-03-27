# Pi Monorepo 实现原理全解 — 写作进度

## 章节规划

| # | 章节标题 | 文件名 | 核心覆盖 | 状态 |
|---|---------|--------|---------|------|
| 0 | 序言 — 全书地图 | ch00-preface.md | 项目定位 / 架构全景图 / 核心概念词典 / 代码库地图 / 一次典型交互极简全流程 | ✅ |
| 1 | 数据流全景 — 一次完整交互的生命周期 | ch01-data-flow.md | 用户输入→CLI 解析→Agent 循环→LLM 流式响应→工具调用→结果回传→渲染输出，每步数据形态变化 | ✅ |
| 2 | pi-ai — 统一的 LLM 通信层 | ch02-pi-ai.md | Provider 适配器模式 / 流式事件协议 / 消息转换 / 模型注册 / 跨 Provider 兼容 / 成本计算 | ✅ |
| 3 | pi-agent-core — Agent 运行时 | ch03-agent-core.md | Agent 类 vs agent-loop / 工具执行生命周期 / 事件流 / 状态管理 / Steering 与 Follow-up / Proxy 模式 | ⏳ |
| 4 | pi-coding-agent 核心 — 从 CLI 到 AgentSession | ch04-coding-agent-core.md | CLI 架构 / AgentSession 抽象 / 会话管理(JSONL 树结构) / 模型解析 / 设置系统 / Compaction | ⏳ |
| 5 | 工具系统与扩展机制 | ch05-tools-and-extensions.md | 7 个内置工具实现 / 扩展加载器(Jiti) / 事件钩子 / 自定义工具注册 / Skills 系统 | ⏳ |
| 6 | pi-tui — 终端 UI 框架 | ch06-tui.md | 差分渲染 / Component 模型 / 键盘输入 / 内置组件(Editor, Markdown, SelectList) / 图像协议 | ⏳ |
| 7 | 交互模式 — TUI + Agent 的协奏 | ch07-interactive-mode.md | Interactive Mode 架构 / 事件到 UI 的映射 / 主题系统 / 斜杠命令 / 快捷键 | ⏳ |
| 8 | RPC 协议与 Web UI | ch08-rpc-and-web-ui.md | RPC JSON-line 协议 / Web Components / IndexedDB 存储 / 沙箱 Iframe / Artifacts | ⏳ |
| 9 | Mom 与 Pods — Slack 机器人与 GPU 管理 | ch09-mom-and-pods.md | Mom: Slack 适配→Agent 委托→沙箱执行 / Pods: SSH 部署→vLLM 编排→端口管理 | ⏳ |
| 10 | 端到端追踪 — 一次真实编码任务的完整旅程 | ch10-e2e-trace.md | 从 `pi "修复这个 bug"` 到代码修改完成，串联全书所有模块，验证理解 | ⏳ |

## 章节规划说明

### 认知路径依赖链

```
ch00 序言（全景地图，建立心智模型）
  ↓
ch01 数据流全景（黑盒级别的完整流程，标注各模块角色）
  ↓
ch02 pi-ai（最底层：LLM 通信，理解消息/流/模型这些基本积木）
  ↓
ch03 pi-agent-core（在 pi-ai 之上：Agent 循环如何编排 LLM 调用和工具执行）
  ↓
ch04 pi-coding-agent 核心（在 agent-core 之上：CLI 如何创建 AgentSession、会话如何持久化）
  ↓
ch05 工具与扩展（coding-agent 的两大扩展维度：内置工具实现 + 扩展机制）
  ↓
ch06 pi-tui（独立的 UI 层：理解终端渲染原理，为下一章做准备）
  ↓
ch07 交互模式（将 agent 事件与 TUI 组件连接起来）
  ↓
ch08 RPC 与 Web UI（另一种 UI 方案：无头模式 + 浏览器组件）
  ↓
ch09 Mom 与 Pods（两个独立应用：展示上述核心模块的不同集成方式）
  ↓
ch10 端到端追踪（综合验收：串联全书，闭合认知环路）
```

### 设计原则
- **自底向上**：先理解积木（pi-ai 消息类型、流），再理解搭建方式（agent 循环），再理解产品形态（CLI/TUI/Web）
- **先黑盒后白盒**：ch01 先给出完整流程的黑盒理解，后续章节逐个打开
- **主干优先**：pi-ai → agent-core → coding-agent 是主干，tui/web-ui/mom/pods 是支线
- **端到端闭合**：ch10 回到 ch01 的场景，但此时读者已理解每个环节的内部实现

## 术语约定

| 术语 | 类型 | 说明 |
|------|------|------|
| Provider | 行业通用 | LLM 服务提供商（OpenAI、Anthropic、Google 等） |
| Stream / 流式 | 行业通用 | LLM 逐 token 返回响应的方式 |
| Tool Calling / 工具调用 | 行业通用 | LLM 请求调用外部函数的能力 |
| Agent Loop / Agent 循环 | 行业通用 | LLM 调用→工具执行→结果回传的循环 |
| Context Window / 上下文窗口 | 行业通用 | LLM 单次请求能处理的最大 token 数 |
| EventStream | 项目特有 | pi-ai 的异步可迭代事件队列，类似 AsyncIterator + Promise 的混合体 |
| AgentSession | 项目特有 | coding-agent 的核心抽象，封装了 Agent + 会话持久化 + 扩展集成 |
| Compaction / 压缩 | 项目特有 | 当上下文接近窗口限制时，用 LLM 总结旧消息来腾出空间 |
| Steering / 转向 | 项目特有 | Agent 运行中插入新指令（类似行业的"中断"），在当前工具轮次结束后注入 |
| Follow-up / 后续 | 项目特有 | Agent 完成后追加新任务（排队等 Agent 空闲时处理） |
| Session Entry | 项目特有 | 会话 JSONL 文件中的一条记录，带 id/parentId 构成树结构 |
| Extension | 项目特有 | coding-agent 的插件系统，通过事件钩子和 API 注册扩展功能 |
| Skill | 项目特有 | 遵循 Agent Skills 规范的 Markdown 文件，为 Agent 提供领域知识 |
| transformMessages | 项目特有 | pi-ai 的消息转换函数，处理跨 Provider 的工具调用 ID 归一化和思考块兼容 |

## 下次续写指引

### 从哪里继续
从 ch00（序言）开始写作。所有探索工作已完成，对七个包的架构、数据流、核心类型有了完整理解。

### 交接备忘
- 项目版本：v0.63.1
- 核心依赖链：pi-ai → pi-agent-core → pi-coding-agent
- pi-tui 是独立的 UI 库，不依赖其他包
- web-ui 依赖 pi-ai 和 pi-tui
- mom 依赖 pi-coding-agent（完整的 agent 能力）
- pods 依赖 pi-agent-core（轻量 agent）
- 所有包使用 ES Module，Node.js >=20
- TypeBox 是核心的 schema 定义库（替代 Zod/JSON Schema）
- 会话文件是 JSONL 格式，用 id/parentId 构成树结构
- 扩展通过 jiti（自定义 fork）动态加载 TypeScript

### 待验证项
- [ ] compaction 的具体切割算法（cut point 选择策略）
- [ ] proxy.ts 中带宽优化的具体实现（partial field 剥离与重建）
- [ ] web-ui 的 mini-lit 框架与标准 Lit 的具体差异
- [ ] pods 的 GPU 分配算法细节
