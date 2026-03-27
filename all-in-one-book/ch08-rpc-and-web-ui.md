# 第八章 RPC 协议与 Web UI

交互模式把 Agent 绑定在了终端里。但有很多场景需要 Agent 的能力却不需要终端——Web 应用、IDE 插件、自动化脚本。Pi 用 **RPC 模式**和 **Web UI 组件**解决了这个需求。

---

## 8.1 RPC 模式：无头的 Agent

RPC（Remote Procedure Call）模式把 AgentSession 的全部能力暴露为一个 **JSON-line 协议**。每行一个 JSON 对象，通过 stdin/stdout 通信。

```bash
pi --mode rpc     # 启动 RPC 模式
```

协议有三种消息方向：

**命令**（stdin → Agent）：客户端发送操作请求

```json
{"type":"prompt","message":"读取 package.json"}
{"type":"steer","message":"改用另一种方式"}
{"type":"set_model","provider":"openai","modelId":"gpt-5.4"}
{"type":"compact"}
{"type":"get_state"}
```

**响应**（Agent → stdout）：每个命令对应一个响应

```json
{"type":"response","command":"prompt","success":true}
{"type":"response","command":"get_state","success":true,"data":{"model":"claude-opus-4-6","isStreaming":false,...}}
```

**事件**（Agent → stdout）：AgentSession 事件的 JSON 序列化

```json
{"type":"agent_start"}
{"type":"message_update","assistantMessageEvent":{"type":"text_delta","delta":"版本号是","contentIndex":0}}
{"type":"tool_execution_start","toolCallId":"toolu_01","toolName":"read","args":{"path":"package.json"}}
{"type":"agent_end"}
```

RPC 模式支持 20+ 种命令，覆盖了交互模式的所有操作（prompt、steer、follow-up、模型切换、会话管理、压缩等）。

**扩展 UI 交互**也通过 RPC 传递。当扩展调用 `ctx.ui.confirm("允许删除？")` 时，RPC 模式向客户端发送 UI 请求，等待客户端响应：

```json
{"type":"extension_ui_request","method":"confirm","title":"允许删除？","message":"..."}
// 客户端响应
{"type":"extension_ui_response","requestId":"req_1","result":true}
```

这个设计让扩展在 RPC 模式下也能拥有 UI 交互能力——只是 UI 由客户端（比如 Web 应用）实现。

---

## 8.2 Web UI 组件库

pi-web-ui 是一套用 Web Components 构建的浏览器端 AI 聊天界面。它可以直接嵌入任何 Web 应用：

```html
<pi-chat-panel
  .agent=${agent}
  .storage=${appStorage}>
</pi-chat-panel>
```

### 核心组件

**ChatPanel** 是最高层的组件——一个完整的聊天界面，包含消息列表、输入框、Artifact 面板：

```
┌─────────────────────────────────────────────┐
│                 ChatPanel                    │
│ ┌─────────────────────┐ ┌─────────────────┐ │
│ │    MessageList       │ │  ArtifactsPanel │ │
│ │  ┌───────────────┐  │ │  ┌────────────┐ │ │
│ │  │ User Message   │  │ │  │ Sandboxed  │ │ │
│ │  │ Assistant Reply │  │ │  │ Iframe     │ │ │
│ │  │ Tool Result     │  │ │  │ (HTML/SVG) │ │ │
│ │  │ Thinking Block  │  │ │  └────────────┘ │ │
│ │  └───────────────┘  │ └─────────────────┘ │
│ └─────────────────────┘                      │
│ ┌─────────────────────────────────────────┐  │
│ │              Input Bar                   │  │
│ │  [消息输入...]            [📎 附件]       │  │
│ └─────────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

**AgentInterface** 是更底层的组件——只有消息列表和输入，不包含 Artifact 面板。适合需要自定义布局的场景。

### 存储系统

Web UI 用 **IndexedDB** 持久化数据，通过一个三层架构管理：

```
AppStorage（统一门面）
  ├─ SettingsStore     → 应用设置（代理、主题等）
  ├─ ProviderKeysStore → API Key 存储
  ├─ SessionsStore     → 聊天会话历史
  └─ Backend           → IndexedDB 实现
```

为什么用 IndexedDB 而不是 localStorage？因为 localStorage 有 5MB 大小限制且只支持字符串，对于包含图像附件的长对话远远不够。IndexedDB 支持结构化数据和大体量存储。

### 沙箱 Iframe

Web UI 的一个特色功能是 **Artifact**——LLM 生成的 HTML/SVG/JavaScript 代码可以在一个**沙箱化的 iframe** 中实时预览。

沙箱通过 `sandbox` 属性限制 iframe 的能力，防止恶意代码：
- 不允许访问父页面的 DOM
- 不允许发起网络请求（除非明确允许）
- 不允许执行 `top.location` 等跳转

同时，Web UI 通过 `postMessage` 建立了安全的通信通道，让 iframe 中的代码可以访问 Artifact 数据和附件。

### 消息类型扩展

与 pi-agent-core 的自定义消息类似，Web UI 也扩展了消息类型：

```typescript
interface UserMessageWithAttachments {   // 带附件的用户消息
  role: "user-with-attachments";
  content: string;
  attachments: Attachment[];            // PDF、DOCX、图片等
}

interface ArtifactMessage {             // Artifact 创建/更新
  role: "artifact";
  action: "create" | "update" | "delete";
  filename: string;
  content: string;
}
```

`convertToLlm()` 函数把这些自定义消息转换成 LLM 能理解的格式——附件被提取为文本或图像内容块，Artifact 消息被过滤掉（它们只服务 UI）。

---

## 8.3 小结

RPC 模式和 Web UI 代表了 Pi 的**无头化**和**浏览器化**两个方向：

- **RPC 模式**用 JSON-line 协议暴露 AgentSession 的全部能力，让任何语言/平台的程序都能驱动 Agent。
- **Web UI** 用 Web Components + IndexedDB 构建了浏览器端的聊天体验，支持附件、Artifact 预览、API Key 管理。
- 两者共享同一个 AgentSession 后端——这是 Pi "业务逻辑与 I/O 分离"设计原则的又一次体现。

主线到此结束。最后两章我们看看 Pi 的两个独立应用——Mom（Slack 机器人）和 Pods（GPU 管理），它们如何以不同的方式集成 Pi 的核心能力。

---

### 质检报告

**讲解节奏**
- [x] RPC 先讲协议设计（三种消息），再讲扩展 UI
- [x] Web UI 先讲整体架构，再讲存储和沙箱

**周边知识**
- [x] 解释了 JSON-line 协议的选择理由（简单、流式、跨语言）
- [x] 解释了 IndexedDB vs localStorage（容量和数据类型限制）
- [x] 解释了沙箱 iframe 的安全模型

**讲透了吗**
- [x] RPC 命令/响应/事件三种消息类型完整
- [x] Web UI 组件层次清晰
- [x] 存储和沙箱机制覆盖

**过渡自然吗**
- [x] 章头衔接上一章
- [x] 章尾引出下一章

**准确吗**
- [x] RPC 协议类型和 Web UI 架构已验证

**勘误建议**
无
