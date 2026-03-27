# 第七章 交互模式 — TUI + Agent 的协奏

上一章我们理解了 pi-tui 的渲染机制和组件模型。这一章看一个关键问题：当 Agent 循环发射一个 `text_delta` 事件时，终端上怎么就多出了几个字符？

交互模式（Interactive Mode）是 Pi 的默认运行模式，也是它的核心体验。它把 AgentSession 的事件流和 pi-tui 的组件系统连接在一起。

---

## 7.1 交互模式的架构

```
用户键盘输入 → Editor 组件 → 斜杠命令 / AgentSession.prompt()
                                                │
                                    AgentSession 事件流
                                                │
                     ┌──────────────────────────┤
                     ▼                          ▼
              消息渲染组件                 工具执行渲染组件
              (Markdown)                (自定义 renderCall/renderResult)
                     │                          │
                     └──────────────────────────┤
                                                ▼
                                    TUI.requestRender()
                                                │
                                                ▼
                                         差分渲染到终端
```

`InteractiveMode` 类是交互模式的编排者。它负责：

1. **创建 TUI 实例**和核心组件（Editor、消息列表、状态栏）
2. **订阅 AgentSession 事件**，把事件转换为组件状态更新
3. **处理用户输入**，路由到斜杠命令或 AgentSession
4. **管理快捷键**（模型切换、中止、设置等）

---

## 7.2 事件到 UI 的映射

当 AgentSession 发射事件时，InteractiveMode 的事件处理器把事件翻译成 UI 操作：

| AgentSession 事件 | UI 操作 |
|-------------------|---------|
| `message_start`（user） | 在消息列表中添加一个用户消息组件 |
| `message_start`（assistant） | 添加一个流式 Markdown 组件，开始渲染 |
| `message_update`（text_delta） | 追加文本到 Markdown 组件，触发重渲染 |
| `message_update`（thinking_delta） | 更新折叠的"思考"块 |
| `message_update`（toolcall_start） | 在消息区域下方添加工具调用预览 |
| `message_end` | 完成消息渲染，更新状态栏（token 数、费用） |
| `tool_execution_start` | 显示工具执行组件（带动画的加载器） |
| `tool_execution_update` | 更新工具执行输出（如 bash 的实时 stdout） |
| `tool_execution_end` | 完成工具执行渲染，显示结果 |
| `compaction_start` | 显示"正在压缩上下文..."的加载器 |
| `compaction_end` | 显示压缩结果（节省了多少 token） |
| `agent_end` | 恢复 Editor 焦点，用户可以继续输入 |

**实时渲染的关键**在 `text_delta` 的处理：每收到一个 delta（可能只有几个字符），就追加到当前的 Markdown 组件并调用 `tui.requestRender()`。pi-tui 的差分渲染确保只有变化的行被重绘——所以即使每秒更新几十次，也不会闪烁。

---

## 7.3 斜杠命令

在 Editor 组件中输入 `/` 开头的内容会被识别为**斜杠命令**。Pi 内置了多个命令：

| 命令 | 作用 |
|------|------|
| `/model` | 弹出模型选择器（SelectList overlay） |
| `/settings` | 弹出设置面板（SettingsList overlay） |
| `/compact` | 手动触发上下文压缩 |
| `/tree` | 显示会话历史树，支持分支导航 |
| `/fork` | 从当前点创建会话分支 |
| `/new` | 开始新会话 |
| `/resume` | 从历史会话中选择一个继续 |
| `/export` | 导出会话为 HTML 文件 |
| `/copy` | 复制最后一条消息到剪贴板 |
| `/reload` | 热重载扩展和技能 |
| `/hotkeys` | 显示快捷键列表 |

扩展也可以通过 `pi.registerCommand()` 注册自定义斜杠命令。

---

## 7.4 快捷键与主题

交互模式的快捷键通过配置文件自定义（`~/.pi/agent/keybindings.json`），而不是硬编码在代码中。所有快捷键都有默认值：

| 快捷键 | 作用 |
|--------|------|
| Enter | 发送消息 |
| Ctrl+C | 中止 Agent |
| Ctrl+P | 切换到下一个模型 |
| Shift+Ctrl+P | 切换到上一个模型 |
| Ctrl+L | 清屏 |
| Esc | 退出对话框 / 取消输入 |

**主题系统**让用户自定义颜色方案。主题是一个 JSON 文件，定义了各种 UI 元素的颜色：

```
~/.pi/agent/themes/my-theme.json
.pi/themes/my-theme.json
```

主题还支持**文件监听**——修改主题文件后，Pi 自动重新加载，无需重启。

---

## 7.5 小结

交互模式的本质是一个**事件到组件的映射层**。AgentSession 发射事件，InteractiveMode 把事件转换为 pi-tui 组件的状态变化，pi-tui 的差分渲染把变化刷新到终端。这个管道让 LLM 的流式输出、工具的实时执行、用户的即时交互无缝协作。

但交互模式只是 Pi 的三种运行模式之一。对于需要被其他程序集成的场景——比如 web-ui 需要在浏览器中呈现相同的 Agent 能力——Pi 提供了 RPC 模式。

下一章，我们看 RPC 协议和 Web UI 组件。

---

### 质检报告

**讲解节奏**
- [x] 先讲交互模式的职责和架构，再展开事件映射
- [x] 没有上来就讲实现

**周边知识**
- [x] 建立在前两章的基础上（AgentSession 事件 + pi-tui 组件）

**讲透了吗**
- [x] 事件到 UI 的映射表完整
- [x] 斜杠命令列表完整
- [x] 快捷键可配置性说明

**过渡自然吗**
- [x] 章头衔接上一章（"text_delta 事件怎么变成终端字符"）
- [x] 章尾引出下一章（"RPC 模式和 Web UI"）

**准确吗**
- [x] 事件类型和命令列表已验证

**勘误建议**
无
