# OpenClaw多Agent协作机制研究

> 基于官方文档和搜索结果的整理

---

## 一、什么是多Agent？

### Agent的定义

一个Agent是一个**完全独立的大脑**，拥有自己的：
- **工作空间**（workspace）- 文件、AGENTS.md、SOUL.md等
- **状态目录**（agentDir）- 认证配置、模型注册
- **会话存储**（sessions）- 聊天历史和路由状态

### 关键路径

```
配置文件: ~/.openclaw/openclaw.json
状态目录: ~/.openclaw
工作空间: ~/.openclaw/workspace（或 ~/.openclaw/workspace-<agentId>）
Agent目录: ~/.openclaw/agents/<agentId>/agent
会话存储: ~/.openclaw/agents/<agentId>/sessions
```

---

## 二、多Agent架构

### 单Agent模式（默认）

- `agentId` 默认为 **`main`**
- 会话键为 `agent:main:<mainKey>`
- 工作空间默认 `~/.openclaw/workspace`

### 多Agent模式

每个 `agentId` 是一个**完全隔离的角色**：
- **不同的电话号码/账号**（每个通道的 accountId）
- **不同的个性**（每个Agent有自己的 AGENTS.md 和 SOUL.md）
- **独立的认证和会话**（不会互相干扰）

---

## 三、如何创建多个Agent

### 使用向导

```bash
openclaw agents add work
```

### 验证

```bash
openclaw agents list --bindings
```

---

## 四、路由规则（消息如何选择Agent）

绑定规则是**确定性的**，**最具体的优先**：

1. `peer` 匹配（精确的 DM/群组/频道 ID）
2. `parentPeer` 匹配（线程继承）
3. `guildId + roles`（Discord 角色路由）
4. `guildId`（Discord）
5. `teamId`（Slack）
6. `accountId` 匹配
7. 频道级匹配（`accountId: "*"`）
8. 回退到默认Agent

---

## 五、配置示例

### 两个WhatsApp → 两个Agent

```json5
{
  agents: {
    list: [
      {
        id: "home",
        default: true,
        name: "Home",
        workspace: "~/.openclaw/workspace-home",
        agentDir: "~/.openclaw/agents/home/agent",
      },
      {
        id: "work",
        name: "Work",
        workspace: "~/.openclaw/workspace-work",
        agentDir: "~/.openclaw/agents/work/agent",
      },
    ],
  },

  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
  ],

  tools: {
    agentToAgent: {
      enabled: false,  // 默认关闭，需要显式开启
      allow: ["home", "work"],
    },
  },

  channels: {
    whatsapp: {
      accounts: {
        personal: {},
        biz: {},
      },
    },
  },
}
```

### 一个WhatsApp号码，多人使用（DM分离）

```json5
{
  agents: {
    list: [
      { id: "alex", workspace: "~/.openclaw/workspace-alex" },
      { id: "mia", workspace: "~/.openclaw/workspace-mia" },
    ],
  },
  bindings: [
    {
      agentId: "alex",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551230001" } },
    },
    {
      agentId: "mia",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551230002" } },
    },
  ],
}
```

### 同一频道，不同人用不同模型

```json5
{
  agents: {
    list: [
      {
        id: "chat",
        name: "Everyday",
        model: "anthropic/claude-sonnet-4-5",
      },
      {
        id: "opus",
        name: "Deep Work",
        model: "anthropic/claude-opus-4-6",
      },
    ],
  },
  bindings: [
    { agentId: "chat", match: { channel: "whatsapp" } },
    { agentId: "opus", match: { channel: "telegram" } },
  ],
}
```

---

## 六、Agent间通信（agentToAgent）

### 默认关闭

```json5
tools: {
  agentToAgent: {
    enabled: false,  // 默认关闭
    allow: ["home", "work"],  // 允许通信的Agent列表
  },
}
```

### 注意事项

- **agentToAgent 只适用于单Bot多Agent架构**
- **多Bot架构**（如多个Telegram Bot）需要用 `sessions_spawn/sessions_send`
- Telegram平台限制：**Bots之间无法互相响应消息**

---

## 七、多Bot架构

### 什么是多Bot架构？

- 每个Bot是**独立的账号**
- 例如：4个独立的Telegram Bots
- `sessions_spawn/sessions_send` **无法跨Bot调用**

### 多Bot通信方案

1. **共享消息存储** - 所有Bot写入同一个数据库
2. **外部协调服务** - 通过API协调
3. **群聊协作** - Bot在同一个群里响应

---

## 八、群聊协作场景

### 多个Bot在同一个群

1. **角色分工**
   - Bot A：处理代码问题
   - Bot B：处理设计问题
   - Bot C：处理翻译

2. **提及触发**
   - `@botA 帮我写代码`
   - `@botB 帮我设计`

3. **轮询响应**
   - Bot轮流处理消息

### 配置示例

```json5
{
  agents: {
    list: [
      {
        id: "coder",
        name: "Code Bot",
        groupChat: {
          mentionPatterns: ["@coder", "@codebot"],
        },
      },
      {
        id: "designer",
        name: "Design Bot",
        groupChat: {
          mentionPatterns: ["@designer", "@designbot"],
        },
      },
    ],
  },
}
```

---

## 九、关键陷阱

### 陷阱1：共享Workspace导致人设混乱

**问题**：多个Agent共享同一个workspace
**后果**：所有AI助手人设身份混乱
**解决**：为每个Agent设置独立的workspace

### 陷阱2：Telegram Bots无法互相响应

**问题**：Telegram平台限制
**后果**：Bots之间无法互相响应消息
**解决**：使用群聊或外部协调

### 陷阱3：agentToAgent配置错误

**问题**：agentToAgent只适用于单Bot多Agent
**后果**：多Bot架构下无法跨Bot调用
**解决**：使用sessions_spawn/sessions_send

### 陷阱4：Allowlist不灵活

**问题**：设置了allowlist，新群无法使用
**解决**：使用更灵活的策略

---

## 十、总结

### 多Agent核心概念

| 概念 | 说明 |
|------|------|
| `agentId` | 一个"大脑"（workspace + auth + sessions） |
| `accountId` | 一个通道账号（如WhatsApp账号"personal" vs "biz"） |
| `binding` | 路由规则（将消息路由到agentId） |

### 架构选择

| 需求 | 方案 |
|------|------|
| 多人共用一个Bot | 多Agent + DM绑定 |
| 多个电话号码 | 多Agent + accountId |
| 同一群多Bot | 多Agent + mentionPatterns |
| Agent间通信 | agentToAgent（单Bot）或sessions_send（跨Bot） |

---

*整理日期: 2026-02-17*
