# 第九章 Mom 与 Pods — Slack 机器人与 GPU 管理

前几章我们完整地解剖了 Pi 的核心——从 LLM 通信到 Agent 运行时，从 CLI 到 UI。这一章看两个**独立应用**，它们以不同的方式集成 Pi 的核心能力，展示了这套架构的灵活性。

---

## 9.1 Mom：Slack 中的编码 Agent

Mom（名字取自"management of machines"）是一个 Slack 机器人——你在 Slack 频道里发消息，它就像一个远程的编码助手，能读文件、写代码、执行命令，然后把结果发回 Slack。

### 架构概览

```
Slack Channel
    │
    ▼ (Socket Mode)
┌────────────────────┐
│    SlackBot         │
│  · 消息路由         │
│  · 频道隔离         │
│  · 消息适配         │
└────────┬───────────┘
         ▼
┌────────────────────┐
│   AgentRunner       │
│  · pi-coding-agent  │
│    的 AgentSession   │
│  · 工具注册         │
│  · 技能加载         │
└────────┬───────────┘
         ▼
┌────────────────────┐
│    Sandbox          │
│  · Docker 容器      │
│  · 文件系统隔离     │
│  · 命令执行隔离     │
└────────────────────┘
```

### 关键设计

**频道隔离**：每个 Slack 频道有独立的状态——独立的 AgentRunner、独立的存储目录、独立的会话。频道 A 的对话不会影响频道 B。

**SlackContext 适配器**：Mom 不使用 TUI 或 RPC 模式，它自己实现了一个**SlackContext**——把 Slack 的消息 API 适配到 AgentSession 需要的接口：

```typescript
interface SlackContext {
  message: { text, user, channel, ts, attachments };
  respond(text): Promise<void>;           // 回复消息
  replaceMessage(text): Promise<void>;    // 更新消息（用于"正在输入..."）
  respondInThread(text): Promise<void>;   // 在线程中回复（工具详情）
  uploadFile(path, title?): Promise<void>;// 上传文件
  setTyping(isTyping): Promise<void>;     // 设置输入指示器
}
```

**沙箱执行**：Mom 不直接在宿主机上执行命令——那太危险了。它把工具操作委托给 Docker 容器。还记得第 5 章提到的工具可插拔操作层吗？Mom 正是利用了这个设计：

```typescript
// Mom 的 bash 工具：在 Docker 容器中执行
const bashOperations = {
  exec: (command, cwd, options) =>
    dockerExec(containerId, command, cwd, options),
};
const bashTool = createBashTool(cwd, { operations: bashOperations });
```

**记忆系统**：Mom 用 Markdown 文件管理记忆——工作区级别的 `MEMORY.md` 和频道级别的 `{channelId}/MEMORY.md`。这些文件会被注入到系统提示词中，让 Agent 记住跨会话的上下文。

**消息更新模式**：Agent 开始处理时，Mom 发送一条"正在处理..."的消息。Agent 完成后用 `replaceMessage()` 更新为最终回复。工具执行的详细输出则通过线程（thread）发送，不污染主频道。

---

## 9.2 Pods：GPU 上的 vLLM 编排

Pods 是一个命令行工具，用于管理远程 GPU 服务器上的 **vLLM** 部署。vLLM 是一个高性能的 LLM 推理引擎——你给它一个模型文件，它提供 OpenAI 兼容的 API 端点。Pods 自动化了"把模型部署到 GPU 服务器"的全过程。

### 核心概念

**Pod**：一台 GPU 服务器。Pods 通过 SSH 连接和管理。

```typescript
interface Pod {
  id: string;                    // 用户定义的名字，如 "dc1"
  sshCmd: string;                // SSH 连接命令
  modelsPath: string;            // 模型存储路径
  gpus: Array<{                  // GPU 状态
    id: number;
    model: string;               // 正在运行的模型
    memory: number;              // 显存大小
  }>;
  models: Record<string, ModelConfig>;  // 已部署的模型
}
```

**模型配置注册表**：Pods 预定义了热门模型的 vLLM 启动参数：

| 模型 | GPU 需求 | 特性 |
|------|---------|------|
| Qwen2.5-Coder-32B | 1-2 GPU | 代码生成 |
| Qwen3-Coder-480B | 8 GPU | 大型代码模型 |
| GLM-4.5 | 2-4 GPU | 支持思考模式 |
| GPT-OSS-120B | 4-8 GPU | 开源 GPT |

每个配置包含 vLLM 命令行参数、工具调用解析器、tensor 并行度、上下文窗口等。

### 工作流

```bash
# 1. 注册一个 Pod（GPU 服务器）
pi pods setup dc1 "ssh root@gpu-server.example.com" \
  --mount "sudo mount -t nfs nfs-server:/models /mnt/models"

# 2. 在 Pod 上安装 vLLM
pi pods setup dc1  # 自动安装 vLLM 和依赖

# 3. 启动模型
pi start Qwen/Qwen2.5-Coder-32B-Instruct --name qwen --context 64k

# 4. 使用模型
pi agent qwen "用 Python 写一个 HTTP 服务器"
```

### 智能 GPU 分配

当 Pod 上有多个 GPU 时，Pods 会智能分配：

1. **查询当前 GPU 使用情况**：检查每个 GPU 上已运行的模型
2. **选择空闲 GPU**：优先使用没有模型运行的 GPU
3. **计算 tensor 并行度**：大模型需要多个 GPU 协同工作
4. **分配端口**：每个模型一个端口（从 8001 开始递增），避免冲突

### 内置 Agent

Pods 还内置了一个轻量的 Agent，用于与部署好的模型交互。它使用 pi-agent-core（不是完整的 coding-agent），提供文件系统工具（read、write、bash 等）。这让你可以直接在终端中和自部署的模型对话。

---

## 9.3 Mom 与 Pods 的对比

| 维度 | Mom | Pods |
|------|-----|------|
| **集成层次** | pi-coding-agent（完整） | pi-agent-core（轻量） |
| **用户界面** | Slack | 终端 CLI |
| **运行环境** | Docker 沙箱 | 远程 GPU 服务器 |
| **状态管理** | 文件 + MEMORY.md | config.json |
| **核心价值** | 把 Agent 带到团队协作平台 | 把开源模型带到本地 Agent |

两个应用展示了 Pi 分层架构的灵活性——Mom 需要完整的 Agent 能力（会话、技能、扩展），所以它集成了 coding-agent；Pods 只需要基本的循环和工具调用，所以它只用 agent-core。

---

## 9.4 小结

Mom 和 Pods 从两个方向扩展了 Pi 的应用边界：

- **Mom** 把编码 Agent 带到了 Slack——团队协作平台。它展示了工具可插拔操作层（Docker 沙箱）、自定义 UI 适配（SlackContext）、记忆管理这些设计如何在不同场景中发挥作用。
- **Pods** 把 LLM 推理搬到了用户自己的 GPU 上——降低成本、保护隐私。它展示了 Pi 的分层设计允许在不同的层次进行集成。

到这里，我们已经理解了 Pi 的每一个包和模块。但真正的考验是：能不能把所有这些知识串起来，完整追踪一次真实的交互？

最后一章，我们做一次端到端追踪。

---

### 质检报告

**讲解节奏**
- [x] Mom 先讲"是什么"（Slack 中的编码 Agent），再讲架构
- [x] Pods 先讲"是什么"（GPU vLLM 管理），再讲工作流

**周边知识**
- [x] 解释了 vLLM 是什么
- [x] 解释了为什么需要 Docker 沙箱（安全性）
- [x] 解释了 tensor 并行度的概念

**讲透了吗**
- [x] Mom 的频道隔离、沙箱执行、记忆系统完整覆盖
- [x] Pods 的 Pod/模型/GPU 概念和工作流
- [x] 两者的集成层次对比

**过渡自然吗**
- [x] 章头衔接主线（"两个独立应用展示架构灵活性"）
- [x] 章尾引出最后一章

**准确吗**
- [x] Mom 的 SlackContext 接口已验证
- [x] Pods 的 Pod 接口和模型列表已验证

**勘误建议**
无
