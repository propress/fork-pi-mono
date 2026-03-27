# Pi Monorepo 实现原理全解 — 写作进度

## 章节规划

| # | 章节标题 | 文件名 | 核心覆盖 | 状态 |
|---|---------|--------|---------|------|
| 0 | [序章：全景鸟瞰](ch00-panorama.md) | ch00-panorama.md | 项目定位 / 架构全景图 / 核心概念词典 / 代码库地图 / 一次典型交互极简全流程 | ✅ |
| 1 | [数据流全景：一次完整的编程对话](ch01-data-flow.md) | ch01-data-flow.md | 从用户输入到 LLM 响应到工具执行，完整数据流每步拆解 | ✅ |
| 2 | [pi-ai：统一的 LLM 抽象层](ch02-pi-ai.md) | ch02-pi-ai.md | Provider 适配器 / 消息类型 / 流式事件 / 工具调用协议 / 模型注册 | ✅ |
| 3 | [pi-agent-core：Agent 运行时引擎](ch03-agent-core.md) | ch03-agent-core.md | Agent 类 / 事件循环 / 工具执行 / 转向与追问队列 / 状态管理 | ✅ |
| 4 | [coding-agent：从命令行到 Agent 会话](ch04-coding-agent.md) | ch04-coding-agent.md | CLI 入口 / main() 流程 / AgentSession 编排 / 运行模式分发 | ⏳ |
| 5 | [工具系统深入：read/write/edit/bash](ch05-tools.md) | ch05-tools.md | 6 个内置工具的实现 / 模糊匹配编辑 / Bash 执行安全 / 文件变更队列 | ⏳ |
| 6 | [扩展系统：不 Fork 就能扩展一切](ch06-extensions.md) | ch06-extensions.md | 扩展发现与加载 / 事件钩子 / 自定义工具与命令 / 扩展 API | ⏳ |
| 7 | [会话管理与上下文压缩](ch07-sessions.md) | ch07-sessions.md | JSONL 持久化 / 分支与切换 / Compaction 原理 / 上下文重建 | ⏳ |
| 8 | [pi-tui：终端 UI 渲染引擎](ch08-tui-engine.md) | ch08-tui-engine.md | 差分渲染 / 组件系统 / Overlay / 键盘协议 / 文本宽度计算 | ⏳ |
| 9 | [交互模式：30+ 组件构建的终端 IDE](ch09-interactive-mode.md) | ch09-interactive-mode.md | InteractiveMode 编排 / 编辑器 / 消息渲染 / 主题 / 斜杠命令 | ⏳ |
| 10 | [应用层：web-ui、mom、pods](ch10-applications.md) | ch10-applications.md | 浏览器聊天组件 / Slack 机器人 / GPU Pod vLLM 部署管理 | ⏳ |
| 11 | [端到端追踪：三个关键场景](ch11-end-to-end.md) | ch11-end-to-end.md | 场景1: 编辑文件 / 场景2: 扩展拦截 / 场景3: 上下文溢出压缩 | ⏳ |

## 章节规划说明

认知路径设计：

```
第 0 章：建立全局画面（"这是什么"、整体长什么样）
    ↓
第 1 章：建立端到端直觉（一次完整交互，数据怎么流动）
    ↓
第 2 章：打开底层黑盒 —— LLM 通信层（第 1 章中"调 LLM"那一步）
    ↓
第 3 章：打开中间层黑盒 —— Agent 运行时（第 1 章中"循环执行"那一步）
    ↓
第 4 章：打开顶层黑盒 —— coding-agent 入口与编排（第 1 章中"启动"那一步）
    ↓
第 5-7 章：逐个深入关键子系统（工具、扩展、会话）
    ↓
第 8-9 章：UI 渲染体系（从底层 TUI 库到上层交互模式）
    ↓
第 10 章：应用层变体（web-ui、mom、pods 如何复用核心）
    ↓
第 11 章：端到端验收（串联全书，完整场景追踪）
```

先建全局直觉，再自底向上打开黑盒，最后端到端串联验证。每章只在读者已有足够上下文时才展开。

## 状态说明
- ✅ 已完成
- 🔄 进行中
- ⏳ 待开始

## 术语约定

### 行业标准术语（直接使用）
- **LLM** (Large Language Model)：大语言模型
- **Provider**：LLM 服务提供商（如 OpenAI、Anthropic）
- **Streaming**：流式响应，逐 token 返回
- **Tool Calling / Function Calling**：工具调用，LLM 请求执行外部操作
- **Token**：LLM 的文本计量单位
- **Context Window**：上下文窗口，模型单次能处理的最大 token 数
- **System Prompt**：系统提示词，定义 Agent 的行为和角色
- **JSON Schema**：描述 JSON 数据结构的规范
- **Monorepo**：单仓库多包管理模式
- **JSONL** (JSON Lines)：每行一个 JSON 对象的文本格式
- **RPC** (Remote Procedure Call)：远程过程调用
- **SDK** (Software Development Kit)：软件开发工具包
- **TUI** (Terminal User Interface)：终端用户界面
- **OAuth**：开放授权协议
- **SSE** (Server-Sent Events)：服务端推送事件

### 项目特有术语（首次出现时解释）
- **Pi**：本项目名称，一个用于构建 AI Agent 的工具链 monorepo
- **Agent**（Pi 语境）：自主执行工具调用的 AI 对话体——类似行业标准 Agent 概念，但 Pi 中特指 `pi-agent-core` 中由事件驱动的状态机
- **AgentSession**：coding-agent 中的会话编排器——在 Agent 之上增加了持久化、扩展、压缩等能力
- **Steering（转向消息）**：在 Agent 正在执行工具时插入的高优先级用户消息，可以改变 Agent 的执行方向
- **Follow-up（追问消息）**：在 Agent 完成当前轮次后自动注入的后续消息
- **Compaction（上下文压缩）**：当对话历史接近上下文窗口限制时，用 LLM 总结旧消息以释放空间
- **Turn（轮次）**：Agent 的一次完整"思考-行动"循环——从收到消息到所有工具执行完毕
- **Skill（技能）**：Markdown 格式的能力描述文件，注入 System Prompt 扩展 Agent 能力
- **Extension（扩展）**：TypeScript 代码文件，通过事件钩子和自定义工具深度定制 Agent 行为
- **ThinkingLevel（思考级别）**：控制 LLM 推理深度的分级，从 "off" 到 "xhigh"

## 下次续写指引
### 从哪里继续
从第 0 章（ch00-panorama.md）开始写作。

### 交接备忘
- 已完成全部 7 个包的深度源码阅读
- 架构已充分理解，可以直接开始写作
- 注意：pi-ai 是最底层，pi-agent-core 依赖 pi-ai，coding-agent 依赖前两者
- TUI 包是独立的渲染引擎，被 coding-agent 的交互模式使用
- web-ui、mom、pods 是三个独立应用，复用核心层

### 待验证项
- [ ] pi-ai 中 `transform-messages.ts` 的跨 Provider 消息转换具体规则
- [ ] coding-agent 中 `model-resolver.ts` 的模型解析优先级
- [ ] pi-tui 差分渲染中 `compositeLineAt()` 的具体合成逻辑
