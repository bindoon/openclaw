# OpenClaw 架构深度解析

本文档提供 OpenClaw 代码库的全面架构分析，包括目录结构、LLM 集成、核心循环实现、技能系统和系统提示词。

## 目录

1. [详细目录结构](#1-详细目录结构)
2. [LLM 厂商对接架构](#2-llm-厂商对接架构)
3. [核心 LLM 循环与递归调用](#3-核心-llm-循环与递归调用)
4. [Skills 底层实现](#4-skills-底层实现)
5. [所有系统提示词](#5-所有系统提示词)

---

## 1. 详细目录结构

### 1.1 顶层目录概览

```
openclaw/
├── src/                    # 核心应用源代码 (TypeScript)
├── skills/                 # AI 自动化技能 (47+ 个技能)
├── extensions/             # 频道集成与插件 (30+ 个扩展)
├── apps/                   # 原生移动/桌面应用
├── docs/                   # 综合文档 (Mintlify)
├── packages/               # 内部包 (clawdbot, moltbot)
├── scripts/                # 构建与工具脚本
├── test/                   # 测试工具
├── ui/                     # Web UI 组件
└── vendor/                 # 第三方依赖
```

### 1.2 核心源码目录 (`src/`) 详解

OpenClaw 的核心代码位于 `src/` 目录，包含以下主要模块：

#### **入口文件**
- `index.ts` - CLI 主入口
- `entry.ts` - 网关启动器与初始化
- `extensionAPI.ts` - 插件 SDK 接口

#### **核心模块详解**

| 模块路径 | 文件数 | 主要功能 |
|---------|--------|---------|
| **gateway/** | 100+ | 核心服务器：WebSocket、HTTP、认证、聊天管理、模型发现、钩子系统 |
| **cli/** | 70+ | 命令行界面：守护进程、向导、节点、插件、模型、频道管理 |
| **agents/** | 100+ | 代理运行时：认证配置、沙箱、技能、系统提示、模型选择、工具执行 |
| **channels/** | 多个子模块 | 消息路由：slack、discord、telegram、signal、imessage、line、whatsapp |
| **providers/** | 多个子模块 | OAuth/模型提供商：GitHub Copilot、Google、Qwen |
| **plugins/** | 多个子模块 | 插件系统：发现、注册、加载器、运行时、钩子 |
| **config/** | 多个文件 | 配置管理与会话存储 |
| **memory/** | 多个文件 | 对话记忆与嵌入向量 |
| **acp/** | 多个文件 | Air Command Protocol（设备控制） |
| **infra/** | 工具模块 | 基础设施：环境、端口、二进制文件、运行时守卫 |
| **utils/** | 工具模块 | 共享工具与辅助函数 |

#### **专用模块**
- **media-understanding/** - 图像/视频处理
- **link-understanding/** - 网页链接解析
- **browser/** - 无头浏览器自动化
- **canvas-host/** - 实时画布渲染
- **tts/** - 文本转语音集成
- **terminal/** - 终端集成
- **markdown/** - Markdown 处理
- **security/** - 安全与沙箱

### 1.3 技能目录 (`skills/`)

包含 **47+ 个内置技能**，每个技能为独立目录，包含：
- `SKILL.md` - 技能文档（带 YAML 元数据）
- 实现代码（TypeScript/Python）

**技能分类：**

| 类别 | 技能示例 |
|------|---------|
| **通讯** | discord, slack, imsg, telegram, signal, whatsapp |
| **生产力** | notion, obsidian, things-mac, trello, apple-reminders |
| **开发** | github, coding-agent, npm |
| **实用工具** | weather, healthcheck, tmux, video-frames, voice-call |
| **内容** | gifgrep, songsee, summarize, blogwatcher, openai-whisper |
| **专用** | 1password, bear-notes, spotify-player, oracle, peekaboo |

### 1.4 扩展目录 (`extensions/`)

**30+ 个频道与功能扩展：**

**消息频道：**
- bluebubbles, copilot-proxy, device-pair, discord, feishu
- googlechat, imessage, line, matrix, mattermost, msteams
- nextcloud-talk, signal, slack, telegram, whatsapp, zalo

**服务与 API：**
- google-antigravity-auth, google-gemini-cli-auth
- llm-task, lobster, memory-core, memory-lancedb
- minimax-portal-auth, open-prose, qwen-portal-auth
- talk-voice, voice-call, twitch, tlon, nostr

### 1.5 应用目录 (`apps/`)

```
apps/
├── ios/         # iOS 应用 (Swift)
├── android/     # Android 应用 (Gradle)
├── macos/       # macOS 应用
└── shared/      # 跨平台共享代码
```

**用途：** 为 macOS、iOS 和 Android 提供原生应用，实现本地设备集成、语音控制和系统通知。

### 1.6 文档目录 (`docs/`)

基于 Mintlify 的文档结构：

```
docs/
├── start/         # 快速开始、入门、展示、FAQ
├── install/       # 安装指南（Docker、macOS、Linux、Windows）
├── concepts/      # 模型、故障转移、安全
├── cli/           # 命令参考
├── channels/      # 频道设置指南
├── gateway/       # 网关配置
├── plugins/       # 插件开发
├── platforms/     # 平台特定指南（iOS、Android、macOS）
└── zh-CN/         # 中文文档（自动生成）
```

---

## 2. LLM 厂商对接架构

### 2.1 架构概览

OpenClaw **并非完全自行实现** LLM API 对接，而是采用 **抽象层 + SDK 组合** 的方式：

1. **核心依赖：** 基于 `@mariozechner/pi-coding-agent` SDK
2. **抽象层：** 统一的 `ModelApi` 接口类型
3. **提供商配置：** 声明式配置 + 自动发现
4. **凭证管理：** 多源认证（环境变量、OAuth、AWS SDK）

### 2.2 模型 API 类型定义

**文件：** `/src/config/types.models.ts`

```typescript
// 支持的 API 类型
type ModelApi = 
  | "openai-completions"      // OpenAI Chat Completions API
  | "openai-responses"        // OpenAI 实时响应 API
  | "anthropic-messages"      // Anthropic Messages API
  | "google-generative-ai"    // Google Gemini API
  | "github-copilot"          // GitHub Copilot API
  | "bedrock-converse-stream" // AWS Bedrock Converse API

// 提供商配置结构
interface ModelProviderConfig {
  baseUrl: string              // API 基础 URL
  apiKey?: string              // API 密钥
  auth?: "api-key" | "aws-sdk" | "oauth" | "token"
  api?: ModelApi               // API 类型
  models: ModelDefinitionConfig[]  // 模型列表
}
```

### 2.3 支持的厂商列表

OpenClaw 通过统一接口支持以下厂商：

| 厂商 | API 类型 | 认证方式 | 自动发现 |
|------|---------|---------|---------|
| **OpenAI** | openai-completions | API Key | ✅ |
| **Anthropic/Claude** | anthropic-messages | API Key | ✅ |
| **Google Gemini** | google-generative-ai | API Key | ✅ |
| **GitHub Copilot** | github-copilot | Token 交换 | ✅ |
| **AWS Bedrock** | bedrock-converse-stream | AWS SDK | ✅ 动态发现 |
| **Ollama** | openai-completions | 本地 | ✅ |
| **MiniMax** | openai/anthropic 变体 | API Key | ✅ |
| **Moonshot/Kimi** | openai-completions | API Key | ✅ |
| **Qwen/千帆** | openai-completions | API Key | ✅ |
| **Venice** | openai-completions | API Key | - |
| **Together AI** | openai-completions | API Key | - |
| **Xiaomi** | openai-completions | API Key | - |
| **Cloudflare AI** | openai-completions | API Key | - |

### 2.4 提供商配置与自动发现

**文件：** `/src/agents/models-config.providers.ts`

**核心函数：**

```typescript
// 规范化提供商配置
normalizeProviders(providers: ModelProviderConfig[]): NormalizedProvider[]

// 自动发现隐式提供商（基于环境变量）
resolveImplicitProviders(): ModelProviderConfig[]

// GitHub Copilot 令牌交换
resolveImplicitCopilotProvider(): Promise<ModelProviderConfig | null>

// AWS Bedrock 自动发现（含动态模型列表）
resolveImplicitBedrockProvider(): Promise<ModelProviderConfig | null>
```

**自动发现机制：**

1. **环境变量检测：**
   - `OPENAI_API_KEY` → OpenAI 提供商
   - `ANTHROPIC_API_KEY` → Anthropic 提供商
   - `GOOGLE_API_KEY` → Google Gemini 提供商
   - `AWS_PROFILE` / `AWS_ACCESS_KEY_ID` → AWS Bedrock

2. **OAuth 配置文件：**
   - `{agentDir}/auth.json` - 存储 OAuth 令牌

3. **GitHub Copilot 特殊处理：**
   - 读取 GitHub 令牌 → 交换 Copilot 端点访问权限
   - 自动设置 baseUrl 和模型列表

4. **AWS Bedrock 特殊处理：**
   - 使用 AWS SDK 动态查询可用模型
   - 自动构建模型列表（Claude、Llama 等）

### 2.5 认证与凭证管理

**文件：** `/src/agents/model-auth.ts`

**多源认证解析顺序：**

```typescript
// 1. 环境变量（最高优先级）
process.env.OPENAI_API_KEY
process.env.ANTHROPIC_API_KEY
process.env.GOOGLE_API_KEY

// 2. OAuth 配置文件
{agentDir}/auth.json

// 3. AWS SDK 默认凭证链
~/.aws/credentials
AWS_PROFILE, AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY

// 4. Token 交换（GitHub Copilot）
GitHub Token → Copilot Endpoint
```

### 2.6 模型发现与注册

**文件：** `/src/agents/pi-model-discovery.ts`

**核心组件：**

```typescript
// 模型注册表（来自 pi-coding-agent）
class ModelRegistry {
  // 目录化所有可用模型
  static async load(providersConfig: ModelProviderConfig[]): Promise<ModelRegistry>
  
  // 查找特定模型
  find(provider: string, modelId: string): ModelDefinition | null
}

// 认证存储
class AuthStorage {
  // 管理凭证位置：{agentDir}/auth.json
  static create(agentDir: string): AuthStorage
}
```

**模型解析来源：**
- `{agentDir}/models.json` - 用户配置的模型
- 环境变量 - 提供商 API 密钥
- OAuth 配置 - 基于令牌的认证

### 2.7 模型选择与故障转移

**文件：** `/src/agents/model-selection.ts`

**功能：**
- 每会话模型选择，支持故障转移链
- 提供商别名规范化（例如 "gpt-4" → "openai/gpt-4o"）
- 模型特定兼容性配置（`compat`）处理 API 差异

### 2.8 嵌入式代理运行器

**文件：** `/src/agents/pi-embedded-runner/model.ts`

**集成模型到代理执行：**

```typescript
// 1. 查找模型配置
const model = ModelRegistry.find(provider, modelId)

// 2. 传递模型 + 认证到 pi-ai
const session = createSession({
  model: model,
  auth: authConfig,
  tools: tools,
  systemPrompt: systemPrompt
})

// 3. 处理提供商特定行为
if (provider === "google") {
  // 修复 Google Gemini 的回合顺序问题
  session.fixTurnOrdering()
}

if (model.compat?.thinking) {
  // 处理思考/推理令牌
  session.enableThinking()
}
```

### 2.9 配置加载流程

**文件：** `/src/agents/models-config.ts`

```
loadConfig()
  ↓
ensureOpenClawModelsJson()
  ↓
resolveImplicitProviders()     [环境变量自动发现]
  ↓
normalizeProviders()           [验证与凭证注入]
  ↓
合并显式 models.json 配置
  ↓
loadModelCatalog()             [通过 ModelRegistry]
```

### 2.10 通用性实现总结

OpenClaw 的 LLM 对接采用 **提供商无关抽象** 模式：

1. **统一接口：** 每个提供商映射到标准 `ModelApi` 类型
2. **凭证抽象：** 环境变量、OAuth、AWS SDK、API 密钥统一管理
3. **动态发现：** 通过 `pi-coding-agent` 的 ModelRegistry 动态发现模型
4. **运行时委托：** 执行委托给 `@mariozechner/pi-ai` 库
5. **提供商特定处理：** 通过 `compat` 配置标志和辅助函数处理差异

**添加新提供商的步骤：**
1. 创建提供商构建器函数（参考现有提供商）
2. 在 `resolveImplicitProviders()` 中添加自动发现逻辑
3. 注册凭证解析规则
4. 无需修改核心代码

---

## 3. 核心 LLM 循环与递归调用

### 3.1 主代理循环入口

**文件：** `/src/agents/pi-embedded-runner/run.ts` (第 357-600 行)

**核心循环函数：** `runEmbeddedPiAgent()`

```typescript
// 第 392 行：多回合循环，支持认证配置故障转移
while (true) {
  // 使用当前认证配置尝试一个回合
  const attempt = await runEmbeddedAttempt({
    session,
    model,
    authProfile,
    systemPrompt,
    tools,
    // ... 其他参数
  });
  
  // 处理上下文溢出错误
  if (contextOverflowError) {
    // 压缩上下文并重试
    await compactSession();
    continue;
  }
  
  // 处理中止
  if (aborted) break;
  
  // 成功时退出
  if (lastAssistant?.stopReason === "end_turn") break;
  
  // 处理认证配置故障转移
  if (authFailure) {
    await advanceAuthProfile();
    continue;
  }
}
```

**循环特点：**
- **外层循环：** 处理认证失败、上下文溢出的重试
- **中止条件：** `stopReason === "end_turn"` 或用户中止
- **自动压缩：** 上下文接近限制时自动触发记忆刷写 + 压缩

### 3.2 单回合尝试（LLM 调用）

**文件：** `/src/agents/pi-embedded-runner/run/attempt.ts` (第 140-400+ 行)

**核心流程：**

```typescript
async function runEmbeddedAttempt(params) {
  // 1. 创建会话，包含工具、系统提示、会话历史
  const activeSession = createAgentSession({
    model: params.model,
    authProfile: params.authProfile,
    systemPrompt: params.systemPrompt,
    tools: params.tools,
    sessionHistory: params.session.history,
  });
  
  // 2. 订阅 pi-agent-core 事件
  params.session.subscribe(eventHandler);
  
  // 3. 流式调用 LLM
  const subscription = subscribeEmbeddedPiSession({
    session: activeSession.session,
    onBlockReply: params.onBlockReply,      // 处理助手回复块
    onToolResult: params.onToolResult,      // 处理工具结果
    // ... 其他事件处理器
  });
  
  // 4. 等待流式完成
  await subscription.waitForCompletion();
  
  // 5. 返回结果
  return {
    assistantTexts: subscription.assistantTexts,
    toolMetas: subscription.toolMetas,
    usage: subscription.getUsageTotals(),
  };
}
```

### 3.3 工具调用与执行循环

**文件：** `/src/agents/pi-embedded-subscribe.handlers.ts` (第 1-250 行)

**事件驱动工具处理器：**

```typescript
// 工具执行开始事件
export function handleToolExecutionStart(ctx, evt) {
  const {toolName, toolCallId, input} = evt;
  
  // 记录工具调用
  ctx.state.toolMetas.push({
    toolName,
    toolCallId,
    input,
    startTime: Date.now(),
  });
  
  // 发出工具摘要
  emitAgentEvent({
    stream: "tool",
    data: {
      phase: "start",
      name: toolName,
      input: truncate(input, 8192),  // 截断大输入
    }
  });
}

// 工具执行更新事件（流式部分结果）
export function handleToolExecutionUpdate(ctx, evt) {
  const {toolCallId, partialResult} = evt;
  
  // 流式输出工具结果
  emitAgentEvent({
    stream: "tool",
    data: {
      phase: "update",
      toolCallId,
      content: partialResult,
    }
  });
}

// 工具执行结束事件
export function handleToolExecutionEnd(ctx, evt) {
  const {toolName, toolCallId, isError, result} = evt;
  
  // 清理和截断结果（最大 8KB）
  const sanitizedResult = sanitizeToolResult(result);
  
  // 存储结果供上下文使用
  ctx.state.toolMetas.push({
    toolName,
    toolCallId,
    result: sanitizedResult,
    endTime: Date.now(),
    isError,
  });
  
  // 消息工具跟踪
  if (!isError && isMessagingTool(toolName)) {
    ctx.state.messagingToolSentTexts.push(extractText(result));
  }
  
  // 发出结果事件返回给消费者
  emitAgentEvent({
    stream: "tool",
    data: {
      phase: "result",
      name: toolName,
      result: sanitizedResult,
      error: isError,
    }
  });
}
```

### 3.4 事件订阅桥接

**文件：** `/src/agents/pi-embedded-subscribe.ts` (第 31-618 行)

**创建上下文并订阅 pi-agent-core 生命周期：**

```typescript
export function subscribeEmbeddedPiSession(params) {
  // 创建上下文
  const ctx = createEmbeddedPiSessionContext({
    session: params.session,
    onBlockReply: params.onBlockReply,
    onToolResult: params.onToolResult,
  });
  
  // 订阅所有 pi-agent-core 事件（第 585 行）
  const unsubscribe = params.session.subscribe(
    createEmbeddedPiSessionEventHandler(ctx)
  );
  
  // 返回收集的数据供调用者使用
  return {
    assistantTexts,           // 累积的助手回复
    toolMetas,                // 工具执行元数据
    getUsageTotals(),         // 令牌使用统计
    waitForCompactionRetry(), // 自动压缩同步
    unsubscribe,              // 取消订阅函数
  };
}
```

### 3.5 消息流概览

| 阶段 | 处理器 | 文件 | 功能 |
|------|--------|------|------|
| **LLM 流开始** | `handleMessageStart` | `.handlers.messages.ts` | 初始化消息上下文 |
| **助手增量** | `handleMessageUpdate` | 同上 | 累积文本到 `assistantTexts[]` |
| **工具调用** | `handleToolExecutionStart` | `.handlers.tools.ts` | 发出 `tool_execution_start` |
| **工具结果** | `handleToolExecutionEnd` | 同上 | 清理、存储、发出结果返回会话 |
| **消息完成** | `handleMessageEnd` | `.handlers.messages.ts` | 最终化助手回复，计算使用量 |

### 3.6 递归/多回合机制

**三层循环结构：**

1. **外层循环（第 357 行，`run.ts`）：**
   - 处理认证失败 → 切换认证配置
   - 处理上下文溢出 → 压缩会话
   - 处理中止 → 退出
   
2. **中层循环（隐式，在 `pi-agent-core` 中）：**
   - LLM 流式响应 → 助手文本块
   - 工具调用块 → 执行工具 → 工具结果块
   - 工具结果反馈给 LLM → LLM 继续流式响应
   - 循环直到 `stopReason !== "tool_use"`
   
3. **压缩重试循环（第 202-225 行，`pi-embedded-subscribe.ts`）：**
   - 上下文溢出 → 触发记忆刷写回合
   - 会话压缩 → 移除旧消息
   - 重试原始请求

**示例流程：**

```
用户: "查询天气并发送消息"
  ↓
LLM: [工具调用: weather_get]
  ↓
工具执行: {"temp": 20, "condition": "晴"}
  ↓ [结果反馈给 LLM]
LLM: [工具调用: message_send, text="今天20度，晴天"]
  ↓
工具执行: {sent: true}
  ↓ [结果反馈给 LLM]
LLM: [文本回复: "已发送天气消息"] [stopReason: "end_turn"]
  ↓
循环结束
```

### 3.7 工具策略与清理

**相关文件：**
- `tool-policy.ts` - 定义代理可访问的工具
- `pi-embedded-subscribe.tools.ts` - 将结果截断至 8KB，清理负载
- `session-tool-result-guard.ts` - 持久化前守卫工具结果

**关键机制：**
- **结果截断：** 工具结果最大 8KB，防止上下文爆炸
- **敏感信息清理：** 移除长令牌、大型二进制数据
- **错误处理：** 工具失败时优雅降级

循环继续直到 `stopReason === "end_turn"`（第 456 行），然后最终化并返回。

---

## 4. Skills 底层实现

### 4.1 技能目录结构

OpenClaw 包含 **47+ 个内置技能**，位于 `/skills/` 目录：

```
skills/
├── github/          # GitHub 集成
│   └── SKILL.md     # 技能文档 + YAML 元数据
├── weather/         # 天气查询
│   └── SKILL.md
├── discord/         # Discord 集成
│   └── SKILL.md
├── slack/           # Slack 集成
│   └── SKILL.md
├── notion/          # Notion 集成
│   └── SKILL.md
├── obsidian/        # Obsidian 集成
│   └── SKILL.md
├── tmux/            # Tmux 会话管理
│   └── SKILL.md
├── coding-agent/    # 代码编写代理
│   └── SKILL.md
├── spotify-player/  # Spotify 播放器控制
│   └── SKILL.md
├── openai-image-gen/  # OpenAI 图像生成
│   └── SKILL.md
├── nano-pdf/        # PDF 处理
│   └── SKILL.md
├── sherpa-onnx-tts/ # 本地 TTS
│   └── SKILL.md
└── [40+ 更多技能...]
```

**每个技能包含：**
- `SKILL.md` - 技能文档，包含 YAML frontmatter 元数据
- 可选的实现代码（TypeScript/Python/Shell）

### 4.2 核心实现文件

技能系统在 `/src/agents/skills/` 中实现：

| 文件 | 功能 |
|------|------|
| **workspace.ts** | 加载、过滤和构建技能提示；主入口点 |
| **types.ts** | 核心 TypeScript 接口（SkillEntry、SkillSnapshot 等） |
| **config.ts** | 解析技能资格、配置路径、平台检查 |
| **frontmatter.ts** | 解析 YAML frontmatter，提取 OpenClaw 元数据 |
| **bundled-dir.ts** | 解析捆绑技能目录 |
| **refresh.ts** | 监视技能目录变化，发出事件 |
| **plugin-skills.ts** | 从插件扩展加载技能 |
| **env-overrides.ts** | 应用环境变量覆盖到技能 |

### 4.3 技能加载、注册与执行

#### **加载流程** (`workspace.ts` 第 100-189 行)

```typescript
function loadWorkspaceSkillEntries(workspaceDir: string, options: LoadOptions) {
  const entries: SkillEntry[] = [];
  
  // 1. 从捆绑包加载 (最低优先级)
  entries.push(...loadSkillsFromBundled());
  
  // 2. 从额外目录加载
  entries.push(...loadSkillsFromExtraDirs(config.skills.load.extraDirs));
  
  // 3. 从托管目录加载
  entries.push(...loadSkillsFromManaged("~/.openclaw/skills/"));
  
  // 4. 从工作区加载 (最高优先级)
  entries.push(...loadSkillsFromDir(`${workspaceDir}/skills/`));
  
  // 优先级顺序：extra < bundled < managed < workspace
  return deduplicateByName(entries, "workspace-first");
}
```

使用 `@mariozechner/pi-coding-agent` 包中的 `loadSkillsFromDir()` 解析 SKILL.md 文件。

#### **过滤逻辑** (`workspace.ts` 第 44-63 行)

```typescript
function filterSkills(entries: SkillEntry[], config: Config) {
  return entries.filter(entry => {
    // 1. 资格检查（OS 兼容性、所需二进制文件、环境变量）
    if (!isSkillEligible(entry, config)) return false;
    
    // 2. 配置允许列表/拒绝列表过滤
    if (config.skills.denylist.includes(entry.skill.name)) return false;
    if (config.skills.allowlist.length > 0 && 
        !config.skills.allowlist.includes(entry.skill.name)) return false;
    
    // 3. 可选的技能过滤器白名单
    if (options.skillFilter && 
        !options.skillFilter.includes(entry.skill.name)) return false;
    
    return true;
  });
}
```

#### **构建技能提示** (`workspace.ts` 第 228-254 行)

```typescript
function buildWorkspaceSkillsPrompt(
  workspaceDir: string,
  options: PromptOptions
): string {
  // 1. 加载并过滤技能
  const entries = loadWorkspaceSkillEntries(workspaceDir, options);
  
  // 2. 格式化为提示文本
  return formatSkillsForPrompt(entries.map(e => e.skill));
  // 输出：格式化的技能列表，包含名称、描述、位置
}
```

#### **系统提示集成** (`system-prompt.ts` 第 16-37 行)

```markdown
## Skills (mandatory)
Before replying: scan <available_skills> <description> entries.
- If exactly one skill clearly applies: read its SKILL.md at <location> with `read`
- If multiple could apply: choose the most specific one
- If none clearly apply: do not read any SKILL.md

Constraints:
- never read more than one skill up front
- only read after selecting
```

#### **执行流程**

1. **代理读取技能列表：** 系统提示中包含 `<available_skills>` 列表
2. **代理选择技能：** 根据任务扫描描述，选择最相关的技能
3. **代理读取 SKILL.md：** 使用 `read` 工具获取完整技能文档
4. **代理遵循指令：** 按照技能文档中的步骤执行任务
5. **用户可调用技能：** 创建 Discord/Telegram 命令供直接调用

### 4.4 技能元数据结构

#### **核心数据结构** (`types.ts`)

```typescript
// 核心技能条目
interface SkillEntry {
  skill: Skill;                          // 名称、描述、文件路径、基础目录
  frontmatter: Record<string, string>;   // 原始 YAML 值
  metadata?: OpenClawSkillMetadata;      // 解析的 OpenClaw 特定配置
  invocation?: SkillInvocationPolicy;    // 用户可调用、禁用模型调用
}

// 运行时快照
interface SkillSnapshot {
  prompt: string;                        // 格式化的技能供代理使用
  skills: Array<{
    name: string;
    primaryEnv?: string;                 // 主要环境变量
  }>;
  resolvedSkills?: Skill[];              // 完整技能对象
  version?: number;                      // 版本号（用于刷新）
}

// OpenClaw 元数据（从 YAML 解析）
interface OpenClawSkillMetadata {
  emoji?: string;                        // 技能图标
  requires?: {
    bins?: string[];                     // 所需二进制文件
    anyBins?: string[];                  // 任一所需二进制文件
    env?: string[];                      // 所需环境变量
    config?: string[];                   // 所需配置路径
    os?: ("darwin" | "linux" | "win32")[]; // 所需操作系统
  };
  install?: SkillInstallSpec[];          // 安装说明
  primaryEnv?: string;                   // 主要环境变量名称
  homepage?: string;                     // 主页 URL
  always?: boolean;                      // 始终包含（忽略资格检查）
}

// 调用策略
interface SkillInvocationPolicy {
  userInvocable?: boolean;               // 可作为命令调用
  disableModelInvocation?: boolean;      // 隐藏代理上下文
}
```

### 4.5 技能 YAML Frontmatter 示例

#### **GitHub 技能** (`skills/github/SKILL.md`)

```yaml
---
name: github
description: "使用 `gh` CLI 与 GitHub 交互..."
metadata:
  openclaw:
    emoji: "🐙"
    requires:
      bins: ["gh"]
    install:
      - kind: brew
        formula: gh
        label: "Install GitHub CLI"
---
```

#### **天气技能** (`skills/weather/SKILL.md`)

```yaml
---
name: weather
description: 获取当前天气和预报（无需 API 密钥）
metadata:
  openclaw:
    emoji: "🌤️"
    requires:
      bins: ["curl"]
---
```

#### **Notion 技能** (`skills/notion/SKILL.md`)

```yaml
---
name: notion
description: "使用 Notion API 管理页面和数据库"
metadata:
  openclaw:
    emoji: "📝"
    requires:
      env: ["NOTION_API_KEY"]
    primaryEnv: "NOTION_API_KEY"
    homepage: "https://www.notion.so/my-integrations"
---
```

### 4.6 关键函数 API

| 函数 | 输入 | 输出 | 用途 |
|------|------|------|------|
| `loadWorkspaceSkillEntries()` | workspaceDir | SkillEntry[] | 加载所有可用技能 |
| `buildWorkspaceSkillsPrompt()` | workspaceDir, opts | string | 生成系统提示的技能列表 |
| `buildWorkspaceSkillSnapshot()` | workspaceDir, opts | SkillSnapshot | 创建版本化技能快照 |
| `resolveSkillsPromptForRun()` | snapshot, entries | string | 获取当前运行的技能文本 |
| `buildWorkspaceSkillCommandSpecs()` | workspaceDir | SkillCommandSpec[] | 创建 Discord/Telegram 命令 |
| `formatSkillsForPrompt()` | Skill[] | string | 格式化技能（来自 pi-coding-agent） |

### 4.7 技能资格与过滤

技能通过以下方式过滤：

**平台检查：**
```typescript
requires.os: ["darwin"]  // 仅 macOS
requires.os: ["linux"]   // 仅 Linux
requires.os: ["win32"]   // 仅 Windows
```

**二进制文件检查：**
```typescript
requires.bins: ["gh"]         // 必须有 gh 命令
requires.anyBins: ["python3", "python"]  // 必须有其中之一
```

**配置检查：**
```typescript
requires.config: ["~/.gitconfig"]  // 必须有配置文件
```

**特殊标志：**
```typescript
metadata.always: true              // 始终包含（忽略资格）
user-invocable: false              // 隐藏命令
disable-model-invocation: true     // 隐藏代理上下文
```

### 4.8 监视/刷新系统

**文件：** `refresh.ts`

**功能：**
- 监视技能目录：workspace/skills、~/.openclaw/skills、额外目录、插件目录
- 去抖动（可配置）
- 变化时发出版本增量
- 允许运行中代理的远程技能更新

**使用示例：**
```typescript
const watcher = watchSkillsDirectories({
  workspaceDir: "/path/to/workspace",
  extraDirs: ["/custom/skills"],
  debounceMs: 1000,
  onRefresh: (newVersion) => {
    console.log(`Skills updated: version ${newVersion}`);
  }
});
```

### 4.9 提示词模板

#### **技能部分模板** (在 `system-prompt.ts` 中)

```
## Skills (mandatory)
Before replying: scan <available_skills> <description> entries.
- If exactly one skill clearly applies: read its SKILL.md at <location> with `read`
- If multiple could apply: choose the most specific one
- If none clearly apply: do not read any SKILL.md

Constraints:
- never read more than one skill up front
- only read after selecting
```

#### **技能列表格式** (通过 `formatSkillsForPrompt()` 生成)

```xml
<available_skills>
<skill>
  <name>github</name>
  <description>使用 gh CLI 与 GitHub 交互</description>
  <location>file:///path/to/skills/github/SKILL.md</location>
</skill>
<skill>
  <name>weather</name>
  <description>获取当前天气和预报</description>
  <location>file:///path/to/skills/weather/SKILL.md</location>
</skill>
...
</available_skills>
```

---

## 5. 所有系统提示词

OpenClaw 在代码库中有 **4 个主要系统提示系统**：

### 5.1 主代理系统提示

**文件：** `/src/agents/system-prompt.ts`

**用途：** 运行 OpenClaw 的主代理核心提示。

**关键部分：**

#### **1. 工具说明**
列出所有可用工具（read、write、edit、exec、browser 等）及其用法。

#### **2. 工具调用风格**
```markdown
## Tool Calling Style
- Narrate tool usage for user-facing operations
- Execute silently for internal/background tasks
- Be concise; don't over-explain routine actions
```

#### **3. 安全性**
```markdown
## Safety
- No self-preservation instincts; prioritize user requests
- Don't bypass safeguards or escalate privileges without consent
- Ask before irreversible/destructive actions
```

#### **4. OpenClaw CLI 快速参考**
```markdown
## OpenClaw CLI Quick Reference
- Start: `openclaw gateway start`
- Stop: `openclaw gateway stop`
- Restart: `openclaw gateway restart`
- Status: `openclaw gateway status`
```

#### **5. Skills（技能）**
```markdown
## Skills (mandatory)
Before replying: scan <available_skills> <description> entries.
- If exactly one skill clearly applies: read its SKILL.md at <location> with `read`
- If multiple could apply: choose the most specific one
- If none clearly apply: do not read any SKILL.md

Constraints:
- never read more than one skill up front
- only read after selecting
```

#### **6. Memory Recall（记忆召回）**
```markdown
## Memory Recall
Before answering questions about:
- Prior work, conversations, or events
- Dates, appointments, or time-sensitive info

Use:
- `memory_search(query)` - semantic search in memory store
- `memory_get(path)` - retrieve specific memory file
- Store new memories in `memory/YYYY-MM-DD.md`
```

#### **7. User Identity（用户身份）**
```markdown
## User Identity
Owner: {{OWNER_NAME}}
Phone: {{OWNER_PHONE}}

Access control:
- Verify identity for sensitive operations
- Use phone numbers for authentication when needed
```

#### **8. Workspace（工作区）**
```markdown
## Workspace
Working directory: {{WORKSPACE_DIR}}
- All file paths relative to workspace root
- Use `read`, `write`, `edit` for file operations
```

#### **9. Sandbox Info（沙箱信息）**
```markdown
## Sandbox Info
Mode: {{SANDBOX_MODE}}
- Docker: limited system access, use `exec` for commands
- Native: full system access, elevated permissions available
```

#### **10. Messaging（消息传递）**
```markdown
## Messaging
Use `message` tool to send messages across sessions/channels:
- `message(to: "phone_number", text: "...")`
- `message(to: "channel_name", text: "...")`
- Reply routing: messages sent to origin by default
```

#### **11. Silent Replies（静默回复）**
```markdown
## Silent Replies
Reply with `◇` (SILENT_REPLY_TOKEN) when:
- No meaningful output needed
- Operation completed successfully without user-facing result
- Background task finished
```

#### **12. Heartbeats（心跳）**
```markdown
## Heartbeats
Periodic check-ins for scheduled tasks and reminders.
- Read HEARTBEAT.md if it exists
- Reply `HEARTBEAT_OK` if nothing needs attention
```

#### **13. Runtime Info（运行时信息）**
```markdown
## Runtime Info
Agent ID: {{AGENT_ID}}
Host: {{HOSTNAME}}
OS: {{OS_PLATFORM}}
Model: {{MODEL_NAME}}
Shell: {{SHELL_TYPE}}
Channel: {{CHANNEL_NAME}}
Capabilities: {{CHANNEL_CAPABILITIES}}
```

#### **14. Project Context（项目上下文）**
注入 SOUL.md 和上下文文件（如果存在）。

**模式：**
- `"full"` (主代理): 所有部分
- `"minimal"` (子代理): 减少的工具/工作区/运行时部分
- `"none"` (轻量级): 仅基本身份行

### 5.2 子代理系统提示

**文件：** `/src/agents/subagent-announce.ts`

**用途：** 为具有特定任务的生成子代理提供轻量级提示。

**内容：**
```markdown
You are a **subagent** spawned by the main agent for a specific task.
- You were created to handle: {{TASK_DESCRIPTION}}
- Complete this task. That's your entire purpose.

Rules:
- Stay focused on assigned task only
- Don't initiate heartbeats/proactive actions
- No external messaging unless explicitly tasked
- Return output concisely to main agent
```

**使用场景：**
- 并行任务分解（例如，多个文件搜索）
- 隔离的上下文执行（避免污染主会话）
- 专门的工具链（例如，仅浏览器自动化）

### 5.3 记忆刷写系统提示

**文件：** `/src/auto-reply/reply/memory-flush.ts`

**用途：** 上下文溢出前保存持久记忆的代理回合。

**默认用户提示：**
```markdown
Pre-compaction memory flush.
Store durable memories now (use memory/YYYY-MM-DD.md; create memory/ if needed).
If nothing to store, reply with ◇.
```

**默认系统提示：**
```markdown
Pre-compaction memory flush turn.
The session is near auto-compaction; capture durable memories to disk.
You may reply, but usually ◇ is correct.
```

**触发配置：**
- 当令牌超过 `(contextWindow - reserveTokens - softThreshold)` 时触发
- 默认软阈值：4000 令牌

**流程：**
1. 检测上下文接近限制
2. 触发记忆刷写回合
3. 代理将重要信息写入 `memory/YYYY-MM-DD.md`
4. 代理回复 `◇` 或简短确认
5. 系统压缩会话历史
6. 继续原始任务

### 5.4 心跳提示

**文件：** `/src/auto-reply/heartbeat.ts`

**用途：** 定期检查计划任务和提醒。

**默认提示：**
```markdown
Read HEARTBEAT.md if it exists (workspace context). Follow it strictly.
Do not infer or repeat old tasks from prior chats.
If nothing needs attention, reply HEARTBEAT_OK.
```

**时间表：** 默认每 30 分钟。如果没有任务，回复 `HEARTBEAT_OK`。

**使用场景：**
- 定期提醒（例如，"每天下午 3 点提醒我喝水"）
- 计划任务（例如，"每周一检查 GitHub 问题"）
- 后台监控（例如，"如果 CPU 使用率 > 80% 则通知我"）

**HEARTBEAT.md 示例：**
```markdown
# 心跳任务

## 每天下午 3 点
- 检查天气并发送消息
- 提醒喝水

## 每周一上午 9 点
- 查询 GitHub 开放 PR
- 总结并发送报告

## 持续监控
- 检查系统 CPU 使用率
- 如果 > 80% 则发送警报
```

### 5.5 关键提示构建函数

| 函数 | 文件 | 用途 |
|------|------|------|
| `buildAgentSystemPrompt()` | system-prompt.ts | 主代理系统提示生成器 |
| `buildEmbeddedSystemPrompt()` | pi-embedded-runner/system-prompt.ts | 嵌入式 Pi 代理包装器 |
| `buildSubagentSystemPrompt()` | subagent-announce.ts | 子代理上下文注入 |
| `buildSystemPromptParams()` | system-prompt-params.ts | 运行时元数据（时区、主机、仓库） |
| `buildSystemPromptReport()` | system-prompt-report.ts | 提示组成的遥测报告 |
| `resolveMemoryFlushSettings()` | memory-flush.ts | 记忆刷写配置解析器 |
| `resolveHeartbeatPrompt()` | heartbeat.ts | 心跳提示加载器 |

### 5.6 技能提示

**50+ 个捆绑技能**（在 `/skills/` 中）每个都有一个 `SKILL.md` 文件，代理按需读取：

**技能示例：**
- **github** - GitHub 集成（`gh` CLI）
- **slack** - Slack 消息传递
- **discord** - Discord bot 控制
- **notion** - Notion API 集成
- **obsidian** - Obsidian vault 管理
- **weather** - 天气查询
- **trello** - Trello 看板管理
- **spotify-player** - Spotify 播放器控制
- **apple-reminders** - Apple 提醒事项集成
- **custom tools** - video-frames、nano-pdf、coding-agent 等

**技能按需读取，不烘焙到主系统提示中。**

### 5.7 提示组成流程

```
buildAgentSystemPrompt()
  ↓
1. 加载基础部分（工具、安全、CLI）
  ↓
2. 注入技能列表 (buildWorkspaceSkillsPrompt())
  ↓
3. 添加记忆召回说明
  ↓
4. 注入用户身份信息
  ↓
5. 添加工作区上下文
  ↓
6. 注入沙箱信息
  ↓
7. 添加消息传递规则
  ↓
8. 注入运行时信息 (buildSystemPromptParams())
  ↓
9. 注入项目上下文（SOUL.md、上下文文件）
  ↓
最终系统提示 → LLM
```

### 5.8 提示定制

**环境变量覆盖：**
```bash
OPENCLAW_SYSTEM_PROMPT_MODE="minimal"  # full | minimal | none
OPENCLAW_HEARTBEAT_PROMPT="自定义心跳提示"
OPENCLAW_MEMORY_FLUSH_PROMPT="自定义记忆刷写提示"
```

**配置文件覆盖：**
```json
{
  "agent": {
    "systemPrompt": {
      "mode": "full",
      "sections": {
        "tools": true,
        "skills": true,
        "memory": true,
        "messaging": true
      }
    }
  }
}
```

---

## 总结

本文档全面分析了 OpenClaw 的架构：

1. **目录结构：** 模块化设计，包含核心（src/）、技能（skills/）、扩展（extensions/）、应用（apps/）和文档（docs/）
2. **LLM 集成：** 基于提供商无关抽象层，支持 10+ 个主流 LLM 厂商，通过环境变量和配置文件统一管理
3. **核心循环：** 三层循环结构（外层重试、中层工具调用、内层 LLM 流式），支持自动压缩和故障转移
4. **技能系统：** 47+ 个内置技能，基于 Markdown 文档 + YAML 元数据，代理按需读取和执行
5. **系统提示：** 4 个主要提示系统（主代理、子代理、记忆刷写、心跳），模块化设计支持灵活定制

OpenClaw 提供了一个强大、可扩展的个人 AI 助手平台，具有企业级的架构设计和本地优先的隐私保护理念。
