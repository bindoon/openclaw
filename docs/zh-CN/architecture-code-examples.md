# OpenClaw 架构代码示例

本文档提供 OpenClaw 关键架构组件的实际代码示例，补充架构分析文档。

## 目录

1. [LLM 提供商配置代码](#1-llm-提供商配置代码)
2. [核心 LLM 循环代码](#2-核心-llm-循环代码)
3. [工具调用处理代码](#3-工具调用处理代码)
4. [系统提示构建代码](#4-系统提示构建代码)
5. [技能加载代码](#5-技能加载代码)

---

## 1. LLM 提供商配置代码

### 1.1 模型 API 类型定义

**文件：** `/src/config/types.models.ts`

```typescript
// 支持的 API 类型
export type ModelApi =
  | "openai-completions"      // OpenAI Chat Completions API
  | "openai-responses"        // OpenAI 实时响应 API
  | "anthropic-messages"      // Anthropic Messages API
  | "google-generative-ai"    // Google Gemini API
  | "github-copilot"          // GitHub Copilot API
  | "bedrock-converse-stream" // AWS Bedrock Converse API

// 模型兼容性配置
export type ModelCompatConfig = {
  supportsStore?: boolean;              // 是否支持会话存储
  supportsDeveloperRole?: boolean;      // 是否支持开发者角色
  supportsReasoningEffort?: boolean;    // 是否支持推理强度
  maxTokensField?: "max_completion_tokens" | "max_tokens"; // 最大令牌字段名
};

// 提供商认证模式
export type ModelProviderAuthMode = "api-key" | "aws-sdk" | "oauth" | "token";

// 模型定义配置
export type ModelDefinitionConfig = {
  id: string;                    // 模型 ID (例如 "gpt-4o")
  name: string;                  // 模型显示名称
  api?: ModelApi;                // API 类型（可继承自提供商）
  reasoning: boolean;            // 是否为推理模型
  input: Array<"text" | "image">; // 支持的输入类型
  cost: {
    input: number;               // 输入令牌成本（每百万令牌美元）
    output: number;              // 输出令牌成本
    cacheRead: number;           // 缓存读取成本
    cacheWrite: number;          // 缓存写入成本
  };
  contextWindow: number;         // 上下文窗口大小
  maxTokens: number;             // 最大输出令牌
  headers?: Record<string, string>; // 自定义 HTTP 头
  compat?: ModelCompatConfig;    // 兼容性配置
};

// 提供商配置
export type ModelProviderConfig = {
  baseUrl: string;               // API 基础 URL
  apiKey?: string;               // API 密钥
  auth?: ModelProviderAuthMode;  // 认证模式
  api?: ModelApi;                // 默认 API 类型
  headers?: Record<string, string>; // 自定义 HTTP 头
  authHeader?: boolean;          // 是否使用认证头
  models: ModelDefinitionConfig[]; // 模型列表
};

// Bedrock 自动发现配置
export type BedrockDiscoveryConfig = {
  enabled?: boolean;             // 是否启用自动发现
  region?: string;               // AWS 区域
  providerFilter?: string[];     // 提供商过滤器
  refreshInterval?: number;      // 刷新间隔（秒）
  defaultContextWindow?: number; // 默认上下文窗口
  defaultMaxTokens?: number;     // 默认最大令牌
};

// 模型配置
export type ModelsConfig = {
  mode?: "merge" | "replace";    // 配置模式
  providers?: Record<string, ModelProviderConfig>; // 提供商映射
  bedrockDiscovery?: BedrockDiscoveryConfig; // Bedrock 自动发现
};
```

### 1.2 提供商自动发现示例

**文件：** `/src/agents/models-config.providers.ts`

```typescript
// OpenAI 提供商自动发现
export function resolveOpenAIProvider(): ModelProviderConfig | null {
  const apiKey = process.env.OPENAI_API_KEY;
  if (!apiKey) return null;
  
  return {
    baseUrl: "https://api.openai.com/v1",
    apiKey,
    auth: "api-key",
    api: "openai-completions",
    models: [
      {
        id: "gpt-4o",
        name: "GPT-4o",
        reasoning: false,
        input: ["text", "image"],
        cost: { input: 2.5, output: 10, cacheRead: 1.25, cacheWrite: 3.75 },
        contextWindow: 128000,
        maxTokens: 16384,
      },
      {
        id: "gpt-4o-mini",
        name: "GPT-4o mini",
        reasoning: false,
        input: ["text", "image"],
        cost: { input: 0.15, output: 0.6, cacheRead: 0.075, cacheWrite: 0.3 },
        contextWindow: 128000,
        maxTokens: 16384,
      },
      {
        id: "o1",
        name: "O1",
        reasoning: true,
        input: ["text", "image"],
        cost: { input: 15, output: 60, cacheRead: 7.5, cacheWrite: 22.5 },
        contextWindow: 200000,
        maxTokens: 100000,
        compat: { supportsReasoningEffort: true },
      },
    ],
  };
}

// Anthropic 提供商自动发现
export function resolveAnthropicProvider(): ModelProviderConfig | null {
  const apiKey = process.env.ANTHROPIC_API_KEY;
  if (!apiKey) return null;
  
  return {
    baseUrl: "https://api.anthropic.com",
    apiKey,
    auth: "api-key",
    api: "anthropic-messages",
    models: [
      {
        id: "claude-3-5-sonnet-20241022",
        name: "Claude 3.5 Sonnet",
        reasoning: false,
        input: ["text", "image"],
        cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 3.75 },
        contextWindow: 200000,
        maxTokens: 8192,
        compat: { supportsStore: true },
      },
      {
        id: "claude-3-5-haiku-20241022",
        name: "Claude 3.5 Haiku",
        reasoning: false,
        input: ["text", "image"],
        cost: { input: 0.8, output: 4, cacheRead: 0.08, cacheWrite: 1 },
        contextWindow: 200000,
        maxTokens: 8192,
      },
    ],
  };
}

// GitHub Copilot 提供商（需要令牌交换）
export async function resolveGitHubCopilotProvider(): Promise<ModelProviderConfig | null> {
  // 1. 读取 GitHub 令牌
  const githubToken = await readGitHubToken();
  if (!githubToken) return null;
  
  // 2. 交换 Copilot 端点访问权限
  const copilotAuth = await exchangeCopilotToken(githubToken);
  if (!copilotAuth) return null;
  
  return {
    baseUrl: copilotAuth.endpoints.api,
    apiKey: copilotAuth.token,
    auth: "token",
    api: "github-copilot",
    models: [
      {
        id: "gpt-4o",
        name: "GPT-4o (Copilot)",
        reasoning: false,
        input: ["text"],
        cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }, // 免费
        contextWindow: 128000,
        maxTokens: 8192,
      },
      {
        id: "o1-preview",
        name: "O1 Preview (Copilot)",
        reasoning: true,
        input: ["text"],
        cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }, // 免费
        contextWindow: 128000,
        maxTokens: 32768,
      },
    ],
  };
}

// AWS Bedrock 提供商（动态模型发现）
export async function resolveBedrockProvider(): Promise<ModelProviderConfig | null> {
  // 1. 检查 AWS 凭证
  const hasAwsCredentials = process.env.AWS_PROFILE || 
                           process.env.AWS_ACCESS_KEY_ID;
  if (!hasAwsCredentials) return null;
  
  // 2. 使用 AWS SDK 动态查询可用模型
  const bedrockClient = new BedrockClient({ region: "us-east-1" });
  const { modelSummaries } = await bedrockClient.send(
    new ListFoundationModelsCommand({})
  );
  
  // 3. 转换为 OpenClaw 模型配置
  const models: ModelDefinitionConfig[] = modelSummaries
    .filter(m => m.responseStreamingSupported)
    .map(m => ({
      id: m.modelId,
      name: m.modelName,
      reasoning: m.modelId.includes("reasoning"),
      input: m.inputModalities.includes("IMAGE") ? ["text", "image"] : ["text"],
      cost: estimateCost(m.modelId), // 根据模型 ID 估算成本
      contextWindow: m.maxTokens || 200000,
      maxTokens: m.maxOutputTokens || 4096,
    }));
  
  return {
    baseUrl: "https://bedrock-runtime.us-east-1.amazonaws.com",
    auth: "aws-sdk",
    api: "bedrock-converse-stream",
    models,
  };
}

// 统一的隐式提供商解析
export async function resolveImplicitProviders(): Promise<ModelProviderConfig[]> {
  const providers: ModelProviderConfig[] = [];
  
  // 按优先级顺序解析
  const openai = resolveOpenAIProvider();
  if (openai) providers.push(openai);
  
  const anthropic = resolveAnthropicProvider();
  if (anthropic) providers.push(anthropic);
  
  const google = resolveGoogleProvider();
  if (google) providers.push(google);
  
  const copilot = await resolveGitHubCopilotProvider();
  if (copilot) providers.push(copilot);
  
  const bedrock = await resolveBedrockProvider();
  if (bedrock) providers.push(bedrock);
  
  return providers;
}
```

---

## 2. 核心 LLM 循环代码

### 2.1 主代理循环

**文件：** `/src/agents/pi-embedded-runner/run.ts` (第 390-470 行)

```typescript
export async function runEmbeddedPiAgent(params: RunEmbeddedPiAgentParams) {
  // 初始化使用量累加器
  const usageAccumulator = createUsageAccumulator();
  let autoCompactionCount = 0;
  
  try {
    // 多回合循环，支持认证配置故障转移
    while (true) {
      // 记录已尝试的思考级别
      attemptedThinking.add(thinkLevel);
      
      // 确保工作区存在
      await fs.mkdir(resolvedWorkspace, { recursive: true });
      
      // 处理 Anthropic 特定的提示清理
      const prompt = provider === "anthropic" 
        ? scrubAnthropicRefusalMagic(params.prompt) 
        : params.prompt;
      
      // 执行单次尝试
      const attempt = await runEmbeddedAttempt({
        sessionId: params.sessionId,
        sessionKey: params.sessionKey,
        messageChannel: params.messageChannel,
        agentAccountId: params.agentAccountId,
        messageTo: params.messageTo,
        workspaceDir: resolvedWorkspace,
        agentDir,
        config: params.config,
        skillsSnapshot: params.skillsSnapshot,
        prompt,
        images: params.images,
        disableTools: params.disableTools,
        provider,
        modelId,
        model,
        authStorage,
        modelRegistry,
        agentId: workspaceResolution.agentId,
        thinkLevel,
        verboseLevel: params.verboseLevel,
        reasoningLevel: params.reasoningLevel,
        toolResultFormat: resolvedToolResultFormat,
        timeoutMs: params.timeoutMs,
        runId: params.runId,
        abortSignal: params.abortSignal,
        onPartialReply: params.onPartialReply,
        onBlockReply: params.onBlockReply,
        onToolResult: params.onToolResult,
        onAgentEvent: params.onAgentEvent,
        extraSystemPrompt: params.extraSystemPrompt,
        ownerNumbers: params.ownerNumbers,
      });
      
      // 提取尝试结果
      const { aborted, promptError, timedOut, lastAssistant } = attempt;
      
      // 累积令牌使用量
      mergeUsageIntoAccumulator(
        usageAccumulator,
        attempt.attemptUsage ?? normalizeUsage(lastAssistant?.usage)
      );
      
      // 累积自动压缩次数
      autoCompactionCount += Math.max(0, attempt.compactionCount ?? 0);
      
      // 处理助手错误
      const formattedAssistantErrorText = lastAssistant
        ? formatAssistantErrorText(lastAssistant, {
            cfg: params.config,
            sessionKey: params.sessionKey ?? params.sessionId,
          })
        : undefined;
      
      const assistantErrorText = 
        lastAssistant?.stopReason === "error"
          ? lastAssistant.errorMessage?.trim() || formattedAssistantErrorText
          : undefined;
      
      // 处理中止
      if (aborted) {
        return {
          success: false,
          aborted: true,
          usage: usageAccumulator,
          autoCompactionCount,
        };
      }
      
      // 处理超时
      if (timedOut) {
        return {
          success: false,
          timedOut: true,
          usage: usageAccumulator,
          autoCompactionCount,
        };
      }
      
      // 处理上下文溢出错误
      if (isContextOverflowError(assistantErrorText)) {
        // 触发记忆刷写 + 会话压缩
        await compactSession({
          session: params.session,
          agentDir,
          workspaceDir: resolvedWorkspace,
        });
        
        // 重试
        continue;
      }
      
      // 成功完成，退出循环
      if (lastAssistant?.stopReason === "end_turn") {
        return {
          success: true,
          usage: usageAccumulator,
          autoCompactionCount,
          assistantTexts: attempt.assistantTexts,
          toolMetas: attempt.toolMetas,
        };
      }
      
      // 处理认证失败 → 切换认证配置
      if (isAuthError(assistantErrorText)) {
        const advanced = await advanceAuthProfile({
          provider,
          authStorage,
          currentProfile: authProfile,
        });
        
        if (advanced) {
          authProfile = advanced;
          continue; // 重试新的认证配置
        } else {
          // 没有更多认证配置可用
          return {
            success: false,
            authError: true,
            usage: usageAccumulator,
            autoCompactionCount,
          };
        }
      }
      
      // 其他错误，退出
      return {
        success: false,
        error: assistantErrorText,
        usage: usageAccumulator,
        autoCompactionCount,
      };
    }
  } catch (error) {
    // 捕获未处理的异常
    return {
      success: false,
      error: error.message,
      usage: usageAccumulator,
      autoCompactionCount,
    };
  }
}
```

### 2.2 单回合尝试

**文件：** `/src/agents/pi-embedded-runner/run/attempt.ts`

```typescript
async function runEmbeddedAttempt(params: AttemptParams) {
  // 1. 创建会话，包含工具、系统提示、会话历史
  const activeSession = await createAgentSession({
    model: params.model,
    authProfile: params.authProfile,
    systemPrompt: buildSystemPrompt({
      workspaceDir: params.workspaceDir,
      agentId: params.agentId,
      skillsSnapshot: params.skillsSnapshot,
      ownerNumbers: params.ownerNumbers,
      extraSystemPrompt: params.extraSystemPrompt,
    }),
    tools: buildToolset({
      workspaceDir: params.workspaceDir,
      disableTools: params.disableTools,
      config: params.config,
    }),
    sessionHistory: loadSessionHistory(params.sessionFile),
  });
  
  // 2. 添加用户消息到会话
  activeSession.addUserMessage({
    text: params.prompt,
    images: params.images,
  });
  
  // 3. 订阅 pi-agent-core 事件
  const subscription = subscribeEmbeddedPiSession({
    session: activeSession,
    onBlockReply: params.onBlockReply,
    onToolResult: params.onToolResult,
    onAgentEvent: params.onAgentEvent,
  });
  
  // 4. 流式调用 LLM
  try {
    await activeSession.streamResponse();
  } catch (error) {
    return {
      aborted: false,
      promptError: error.message,
      lastAssistant: null,
      assistantTexts: [],
      toolMetas: [],
    };
  }
  
  // 5. 等待流式完成
  await subscription.waitForCompletion();
  
  // 6. 保存会话历史
  await saveSessionHistory(params.sessionFile, activeSession.history);
  
  // 7. 返回结果
  return {
    aborted: params.abortSignal?.aborted || false,
    promptError: null,
    lastAssistant: activeSession.lastAssistantMessage,
    assistantTexts: subscription.assistantTexts,
    toolMetas: subscription.toolMetas,
    attemptUsage: subscription.getUsageTotals(),
  };
}
```

---

## 3. 工具调用处理代码

### 3.1 工具执行事件处理器

**文件：** `/src/agents/pi-embedded-subscribe.handlers.ts`

```typescript
// 工具执行开始事件
export function handleToolExecutionStart(
  ctx: EmbeddedPiSessionContext,
  evt: ToolExecutionStartEvent
) {
  const { toolName, toolCallId, input } = evt;
  
  // 记录工具调用开始时间
  const startTime = Date.now();
  
  // 存储工具元数据
  ctx.state.toolMetas.push({
    toolName,
    toolCallId,
    input,
    startTime,
    phase: "start",
  });
  
  // 截断大输入（最大 8KB）
  const truncatedInput = truncate(JSON.stringify(input), 8192);
  
  // 发出工具开始事件
  emitAgentEvent(ctx, {
    stream: "tool",
    data: {
      phase: "start",
      name: toolName,
      callId: toolCallId,
      input: JSON.parse(truncatedInput),
      timestamp: startTime,
    },
  });
  
  // 特殊处理：消息工具跟踪
  if (isMessagingTool(toolName)) {
    ctx.state.pendingMessagingTools.set(toolCallId, {
      toolName,
      text: input.text || "",
      to: input.to,
    });
  }
}

// 工具执行更新事件（流式部分结果）
export function handleToolExecutionUpdate(
  ctx: EmbeddedPiSessionContext,
  evt: ToolExecutionUpdateEvent
) {
  const { toolCallId, partialResult } = evt;
  
  // 查找对应的工具元数据
  const meta = ctx.state.toolMetas.find(m => m.toolCallId === toolCallId);
  if (!meta) return;
  
  // 流式输出工具结果
  emitAgentEvent(ctx, {
    stream: "tool",
    data: {
      phase: "update",
      name: meta.toolName,
      callId: toolCallId,
      content: partialResult,
      timestamp: Date.now(),
    },
  });
  
  // 调用用户回调
  if (ctx.callbacks.onToolResult) {
    ctx.callbacks.onToolResult({
      phase: "update",
      name: meta.toolName,
      callId: toolCallId,
      content: partialResult,
    });
  }
}

// 工具执行结束事件
export function handleToolExecutionEnd(
  ctx: EmbeddedPiSessionContext,
  evt: ToolExecutionEndEvent
) {
  const { toolName, toolCallId, isError, result, error } = evt;
  
  // 查找对应的工具元数据
  const meta = ctx.state.toolMetas.find(m => m.toolCallId === toolCallId);
  if (!meta) return;
  
  // 更新工具元数据
  meta.endTime = Date.now();
  meta.duration = meta.endTime - meta.startTime;
  meta.phase = "end";
  meta.isError = isError;
  
  // 清理和截断结果（最大 8KB）
  const sanitizedResult = sanitizeToolResult(result, {
    maxLength: 8192,
    stripLongTokens: true,
    stripBinaryData: true,
  });
  
  meta.result = sanitizedResult;
  
  // 消息工具跟踪
  if (!isError && isMessagingTool(toolName)) {
    const pending = ctx.state.pendingMessagingTools.get(toolCallId);
    if (pending) {
      ctx.state.messagingToolSentTexts.push(pending.text);
      ctx.state.pendingMessagingTools.delete(toolCallId);
    }
  }
  
  // 发出结果事件返回给消费者
  emitAgentEvent(ctx, {
    stream: "tool",
    data: {
      phase: "result",
      name: toolName,
      callId: toolCallId,
      result: sanitizedResult,
      error: isError ? error : undefined,
      duration: meta.duration,
      timestamp: meta.endTime,
    },
  });
  
  // 调用用户回调
  if (ctx.callbacks.onToolResult) {
    ctx.callbacks.onToolResult({
      phase: "result",
      name: toolName,
      callId: toolCallId,
      result: sanitizedResult,
      error: isError ? error : undefined,
      duration: meta.duration,
    });
  }
}

// 工具结果清理函数
function sanitizeToolResult(
  result: unknown,
  options: {
    maxLength: number;
    stripLongTokens: boolean;
    stripBinaryData: boolean;
  }
): unknown {
  if (typeof result === "string") {
    // 截断长字符串
    if (result.length > options.maxLength) {
      return result.slice(0, options.maxLength) + "\n[...truncated]";
    }
    
    // 移除长令牌（JWT、API 密钥等）
    if (options.stripLongTokens) {
      result = result.replace(/[A-Za-z0-9_-]{64,}/g, "[REDACTED_TOKEN]");
    }
    
    return result;
  }
  
  if (typeof result === "object" && result !== null) {
    // 移除二进制数据
    if (options.stripBinaryData && Buffer.isBuffer(result)) {
      return "[BINARY_DATA]";
    }
    
    // 递归清理对象
    if (Array.isArray(result)) {
      return result.map(item => sanitizeToolResult(item, options));
    }
    
    const sanitized: Record<string, unknown> = {};
    for (const [key, value] of Object.entries(result)) {
      sanitized[key] = sanitizeToolResult(value, options);
    }
    return sanitized;
  }
  
  return result;
}
```

---

## 4. 系统提示构建代码

### 4.1 系统提示主构建函数

**文件：** `/src/agents/system-prompt.ts`

```typescript
/**
 * 控制哪些硬编码部分包含在系统提示中
 * - "full": 所有部分（默认，用于主代理）
 * - "minimal": 减少的部分（工具、工作区、运行时）- 用于子代理
 * - "none": 只有基本身份行，无部分
 */
export type PromptMode = "full" | "minimal" | "none";

// 构建技能部分
function buildSkillsSection(params: {
  skillsPrompt?: string;
  isMinimal: boolean;
  readToolName: string;
}) {
  if (params.isMinimal) {
    return [];
  }
  const trimmed = params.skillsPrompt?.trim();
  if (!trimmed) {
    return [];
  }
  return [
    "## Skills (mandatory)",
    "Before replying: scan <available_skills> <description> entries.",
    `- If exactly one skill clearly applies: read its SKILL.md at <location> with \`${params.readToolName}\`, then follow it.`,
    "- If multiple could apply: choose the most specific one, then read/follow it.",
    "- If none clearly apply: do not read any SKILL.md.",
    "Constraints: never read more than one skill up front; only read after selecting.",
    trimmed,
    "",
  ];
}

// 构建记忆部分
function buildMemorySection(params: {
  isMinimal: boolean;
  availableTools: Set<string>;
  citationsMode?: MemoryCitationsMode;
}) {
  if (params.isMinimal) {
    return [];
  }
  if (!params.availableTools.has("memory_search") && 
      !params.availableTools.has("memory_get")) {
    return [];
  }
  
  const lines = [
    "## Memory Recall",
    "Before answering anything about prior work, decisions, dates, people, preferences, or todos: run memory_search on MEMORY.md + memory/*.md; then use memory_get to pull only the needed lines. If low confidence after search, say you checked.",
  ];
  
  if (params.citationsMode === "off") {
    lines.push(
      "Citations are disabled: do not mention file paths or line numbers in replies unless the user explicitly asks.",
    );
  } else {
    lines.push(
      "Citations: include Source: <path#line> when it helps the user verify memory snippets.",
    );
  }
  
  lines.push("");
  return lines;
}

// 构建用户身份部分
function buildUserIdentitySection(
  ownerLine: string | undefined,
  isMinimal: boolean
) {
  if (!ownerLine || isMinimal) {
    return [];
  }
  return ["## User Identity", ownerLine, ""];
}

// 构建时间部分
function buildTimeSection(params: { userTimezone?: string }) {
  if (!params.userTimezone) {
    return [];
  }
  return [
    "## Current Date & Time",
    `Time zone: ${params.userTimezone}`,
    "",
  ];
}

// 构建回复标签部分
function buildReplyTagsSection(isMinimal: boolean) {
  if (isMinimal) {
    return [];
  }
  return [
    "## Reply Tags",
    "To request a native reply/quote on supported surfaces, include one tag in your reply:",
    "- [[reply_to_current]] replies to the triggering message.",
    "- Prefer [[reply_to_current]]. Use [[reply_to:<id>]] only when an id was explicitly provided (e.g. by the user or a tool).",
    "Whitespace inside the tag is allowed (e.g. [[ reply_to_current ]] / [[ reply_to: 123 ]]).",
    "Tags are stripped before sending; support depends on the current channel config.",
    "",
  ];
}

// 构建消息传递部分
function buildMessagingSection(params: {
  isMinimal: boolean;
  availableTools: Set<string>;
  messageChannelOptions: string;
}) {
  if (params.isMinimal || !params.availableTools.has("message")) {
    return [];
  }
  
  return [
    "## Messaging",
    "Use `message` tool to send messages across sessions/channels:",
    "- `message(to: \"phone_number\", text: \"...\")`",
    "- `message(to: \"channel_name\", text: \"...\")`",
    "- Reply routing: messages sent to origin by default",
    "",
    "Available channels:",
    params.messageChannelOptions,
    "",
  ];
}

// 主系统提示构建函数
export function buildAgentSystemPrompt(params: SystemPromptParams): string {
  const isMinimal = params.mode === "minimal";
  const isNone = params.mode === "none";
  
  if (isNone) {
    return `You are an AI assistant running in OpenClaw.`;
  }
  
  const sections: string[] = [];
  
  // 1. 基本身份
  sections.push(
    `You are an AI assistant running in OpenClaw.`,
    `Agent ID: ${params.agentId}`,
    `Host: ${params.hostname}`,
    `OS: ${params.osPlatform}`,
    `Model: ${params.modelName}`,
    ``,
  );
  
  // 2. 工具列表
  sections.push(...buildToolsSection({
    availableTools: params.availableTools,
    isMinimal,
  }));
  
  // 3. 技能
  sections.push(...buildSkillsSection({
    skillsPrompt: params.skillsPrompt,
    isMinimal,
    readToolName: "read",
  }));
  
  // 4. 记忆召回
  sections.push(...buildMemorySection({
    isMinimal,
    availableTools: params.availableTools,
    citationsMode: params.citationsMode,
  }));
  
  // 5. 用户身份
  sections.push(...buildUserIdentitySection(
    params.ownerLine,
    isMinimal
  ));
  
  // 6. 时间
  sections.push(...buildTimeSection({
    userTimezone: params.userTimezone,
  }));
  
  // 7. 回复标签
  sections.push(...buildReplyTagsSection(isMinimal));
  
  // 8. 消息传递
  sections.push(...buildMessagingSection({
    isMinimal,
    availableTools: params.availableTools,
    messageChannelOptions: params.messageChannelOptions || "",
  }));
  
  // 9. 工作区
  sections.push(...buildWorkspaceSection({
    workspaceDir: params.workspaceDir,
    isMinimal,
  }));
  
  // 10. 沙箱信息
  sections.push(...buildSandboxSection({
    sandboxMode: params.sandboxMode,
    isMinimal,
  }));
  
  // 11. 静默回复
  sections.push(...buildSilentReplySection(isMinimal));
  
  // 12. 心跳
  sections.push(...buildHeartbeatSection(isMinimal));
  
  // 13. 额外系统提示
  if (params.extraSystemPrompt) {
    sections.push(params.extraSystemPrompt, "");
  }
  
  // 14. 项目上下文（SOUL.md 等）
  sections.push(...buildProjectContextSection({
    contextFiles: params.contextFiles,
    isMinimal,
  }));
  
  return sections.join("\n");
}
```

---

## 5. 技能加载代码

### 5.1 技能加载主函数

**文件：** `/src/agents/skills/workspace.ts`

```typescript
/**
 * 加载工作区技能条目
 * 优先级顺序：extra < bundled < managed < workspace
 */
export function loadWorkspaceSkillEntries(
  workspaceDir: string,
  options: LoadSkillsOptions
): SkillEntry[] {
  const entries: SkillEntry[] = [];
  const config = options.config || getDefaultConfig();
  
  // 1. 从捆绑包加载（最低优先级）
  const bundledDir = resolveBundledSkillsDir();
  if (bundledDir && fs.existsSync(bundledDir)) {
    const bundledEntries = loadSkillsFromDir(bundledDir, {
      source: "bundled",
    });
    entries.push(...bundledEntries);
  }
  
  // 2. 从额外目录加载
  if (config.skills?.load?.extraDirs) {
    for (const extraDir of config.skills.load.extraDirs) {
      const resolvedDir = resolvePath(extraDir);
      if (fs.existsSync(resolvedDir)) {
        const extraEntries = loadSkillsFromDir(resolvedDir, {
          source: "extra",
        });
        entries.push(...extraEntries);
      }
    }
  }
  
  // 3. 从托管目录加载
  const managedDir = path.join(os.homedir(), ".openclaw", "skills");
  if (fs.existsSync(managedDir)) {
    const managedEntries = loadSkillsFromDir(managedDir, {
      source: "managed",
    });
    entries.push(...managedEntries);
  }
  
  // 4. 从工作区加载（最高优先级）
  const workspaceSkillsDir = path.join(workspaceDir, "skills");
  if (fs.existsSync(workspaceSkillsDir)) {
    const workspaceEntries = loadSkillsFromDir(workspaceSkillsDir, {
      source: "workspace",
    });
    entries.push(...workspaceEntries);
  }
  
  // 5. 从插件加载
  if (options.includePluginSkills) {
    const pluginEntries = loadPluginSkills({
      config,
      pluginRegistry: options.pluginRegistry,
    });
    entries.push(...pluginEntries);
  }
  
  // 6. 去重（工作区优先）
  const deduplicated = deduplicateByName(entries, "workspace-first");
  
  // 7. 解析元数据
  const withMetadata = deduplicated.map(entry => ({
    ...entry,
    metadata: parseOpenClawMetadata(entry.frontmatter),
  }));
  
  // 8. 应用环境变量覆盖
  const withEnvOverrides = applyEnvOverrides(withMetadata, {
    envPrefix: "OPENCLAW_SKILL_",
  });
  
  // 9. 过滤不符合条件的技能
  const filtered = filterSkillsByEligibility(withEnvOverrides, {
    config,
    osPlatform: process.platform,
    checkBinaries: true,
    checkEnvVars: true,
    checkConfig: true,
  });
  
  // 10. 应用配置的允许/拒绝列表
  const final = applyAllowDenyLists(filtered, {
    allowlist: config.skills?.allowlist,
    denylist: config.skills?.denylist,
  });
  
  return final;
}

/**
 * 构建工作区技能提示
 */
export function buildWorkspaceSkillsPrompt(
  workspaceDir: string,
  options: BuildSkillsPromptOptions
): string {
  // 1. 加载并过滤技能
  const entries = loadWorkspaceSkillEntries(workspaceDir, {
    config: options.config,
    includePluginSkills: options.includePluginSkills,
    pluginRegistry: options.pluginRegistry,
  });
  
  // 2. 应用技能过滤器白名单
  const filtered = options.skillFilter
    ? entries.filter(e => options.skillFilter!.includes(e.skill.name))
    : entries;
  
  // 3. 格式化为提示文本
  return formatSkillsForPrompt(filtered.map(e => e.skill));
}

/**
 * 格式化技能为提示（来自 pi-coding-agent）
 */
function formatSkillsForPrompt(skills: Skill[]): string {
  if (skills.length === 0) {
    return "";
  }
  
  const lines = ["<available_skills>"];
  
  for (const skill of skills) {
    lines.push("<skill>");
    lines.push(`  <name>${skill.name}</name>`);
    lines.push(`  <description>${escapeXml(skill.description)}</description>`);
    lines.push(`  <location>file://${skill.filePath}</location>`);
    lines.push("</skill>");
  }
  
  lines.push("</available_skills>");
  
  return lines.join("\n");
}

/**
 * 检查技能资格
 */
function filterSkillsByEligibility(
  entries: SkillEntry[],
  options: EligibilityOptions
): SkillEntry[] {
  return entries.filter(entry => {
    const metadata = entry.metadata;
    
    // 始终包含的技能
    if (metadata?.always) {
      return true;
    }
    
    // 操作系统检查
    if (metadata?.requires?.os) {
      if (!metadata.requires.os.includes(options.osPlatform)) {
        return false;
      }
    }
    
    // 二进制文件检查
    if (options.checkBinaries && metadata?.requires?.bins) {
      for (const bin of metadata.requires.bins) {
        if (!commandExists(bin)) {
          return false;
        }
      }
    }
    
    // 任一二进制文件检查
    if (options.checkBinaries && metadata?.requires?.anyBins) {
      const hasAny = metadata.requires.anyBins.some(bin => 
        commandExists(bin)
      );
      if (!hasAny) {
        return false;
      }
    }
    
    // 环境变量检查
    if (options.checkEnvVars && metadata?.requires?.env) {
      for (const envVar of metadata.requires.env) {
        if (!process.env[envVar]) {
          return false;
        }
      }
    }
    
    // 配置路径检查
    if (options.checkConfig && metadata?.requires?.config) {
      for (const configPath of metadata.requires.config) {
        const resolved = resolvePath(configPath);
        if (!fs.existsSync(resolved)) {
          return false;
        }
      }
    }
    
    return true;
  });
}

/**
 * 解析 OpenClaw 元数据
 */
function parseOpenClawMetadata(
  frontmatter: Record<string, unknown>
): OpenClawSkillMetadata | undefined {
  const metadata = frontmatter.metadata as Record<string, unknown> | undefined;
  if (!metadata?.openclaw) {
    return undefined;
  }
  
  const openclaw = metadata.openclaw as Record<string, unknown>;
  
  return {
    emoji: openclaw.emoji as string | undefined,
    requires: openclaw.requires as {
      bins?: string[];
      anyBins?: string[];
      env?: string[];
      config?: string[];
      os?: ("darwin" | "linux" | "win32")[];
    } | undefined,
    install: openclaw.install as SkillInstallSpec[] | undefined,
    primaryEnv: openclaw.primaryEnv as string | undefined,
    homepage: openclaw.homepage as string | undefined,
    always: openclaw.always as boolean | undefined,
  };
}
```

---

## 总结

本文档提供了 OpenClaw 关键架构组件的实际代码示例：

1. **LLM 提供商配置：** 类型定义、自动发现、认证管理
2. **核心 LLM 循环：** 多回合循环、单次尝试、错误处理
3. **工具调用处理：** 事件处理器、结果清理、流式更新
4. **系统提示构建：** 模块化部分、动态组合、定制化
5. **技能加载：** 多源加载、资格检查、提示格式化

这些代码示例展示了 OpenClaw 如何实现一个强大、可扩展的 AI 代理平台，具有清晰的抽象层和模块化设计。
