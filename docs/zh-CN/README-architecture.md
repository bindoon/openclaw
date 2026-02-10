# OpenClaw 架构文档索引

本目录包含 OpenClaw 代码库的完整架构分析文档。

## 文档列表

### 1. [架构深度解析](./architecture-analysis.md)

**完整的架构概览文档，涵盖：**

- **详细目录结构** - 说明每个目录和主要文件的作用
- **LLM 厂商对接架构** - 如何集成各大语言模型提供商
- **核心 LLM 循环与递归调用** - 代理执行的核心逻辑
- **Skills 底层实现** - 技能系统的实现机制
- **所有系统提示词** - 各类系统提示及其用途

### 2. [架构代码示例](./architecture-code-examples.md)

**实际代码示例文档，包含：**

- **LLM 提供商配置代码** - 类型定义、自动发现、认证管理
- **核心 LLM 循环代码** - 多回合循环、单次尝试、错误处理
- **工具调用处理代码** - 事件处理器、结果清理、流式更新
- **系统提示构建代码** - 模块化部分、动态组合
- **技能加载代码** - 多源加载、资格检查、提示格式化

## 快速导航

### 想了解目录结构？
→ 查看 [架构深度解析 - 1. 详细目录结构](./architecture-analysis.md#1-详细目录结构)

### 想了解 OpenClaw 如何对接各厂商的 LLM？
→ 查看 [架构深度解析 - 2. LLM 厂商对接架构](./architecture-analysis.md#2-llm-厂商对接架构)

→ 查看 [架构代码示例 - 1. LLM 提供商配置代码](./architecture-code-examples.md#1-llm-提供商配置代码)

### 想了解核心 LLM 循环的实现？
→ 查看 [架构深度解析 - 3. 核心 LLM 循环与递归调用](./architecture-analysis.md#3-核心-llm-循环与递归调用)

→ 查看 [架构代码示例 - 2. 核心 LLM 循环代码](./architecture-code-examples.md#2-核心-llm-循环代码)

### 想了解 Skills 的底层实现？
→ 查看 [架构深度解析 - 4. Skills 底层实现](./architecture-analysis.md#4-skills-底层实现)

→ 查看 [架构代码示例 - 5. 技能加载代码](./architecture-code-examples.md#5-技能加载代码)

### 想查看所有系统提示词？
→ 查看 [架构深度解析 - 5. 所有系统提示词](./architecture-analysis.md#5-所有系统提示词)

→ 查看 [架构代码示例 - 4. 系统提示构建代码](./architecture-code-examples.md#4-系统提示构建代码)

## 关键发现

### OpenClaw 的 LLM 集成方式

OpenClaw **并非完全自行实现** LLM API 对接，而是采用 **抽象层 + SDK 组合** 的方式：

1. **核心依赖：** 基于 `@mariozechner/pi-coding-agent` SDK
2. **抽象层：** 统一的 `ModelApi` 接口类型
3. **提供商配置：** 声明式配置 + 自动发现
4. **凭证管理：** 多源认证（环境变量、OAuth、AWS SDK）

支持的厂商包括：
- OpenAI、Anthropic/Claude、Google Gemini
- GitHub Copilot、AWS Bedrock、Ollama
- MiniMax、Moonshot/Kimi、Qwen/千帆
- Venice、Together AI、Xiaomi、Cloudflare AI

### 核心循环架构

三层循环结构：

1. **外层循环：** 处理认证失败、上下文溢出的重试
2. **中层循环（隐式）：** LLM 流式响应 → 工具调用 → 工具结果反馈 → 继续响应
3. **压缩重试循环：** 上下文溢出时触发记忆刷写 + 会话压缩

### Skills 系统

- **47+ 个内置技能**，每个技能为独立的 Markdown 文档
- 基于 YAML frontmatter 的元数据系统
- 代理按需读取技能文档，不烘焙到系统提示中
- 支持资格检查（OS、二进制文件、环境变量、配置）

### 系统提示

四个主要系统提示系统：

1. **主代理系统提示** - 完整的工具、技能、记忆、消息传递等指令
2. **子代理系统提示** - 轻量级任务特定提示
3. **记忆刷写提示** - 上下文溢出前保存记忆
4. **心跳提示** - 定期检查计划任务

## 源码位置参考

### 核心文件路径

| 功能 | 文件路径 |
|------|---------|
| **模型类型定义** | `/src/config/types.models.ts` |
| **提供商配置** | `/src/agents/models-config.providers.ts` |
| **模型认证** | `/src/agents/model-auth.ts` |
| **主代理循环** | `/src/agents/pi-embedded-runner/run.ts` |
| **工具处理** | `/src/agents/pi-embedded-subscribe.handlers.ts` |
| **系统提示** | `/src/agents/system-prompt.ts` |
| **技能加载** | `/src/agents/skills/workspace.ts` |
| **技能类型** | `/src/agents/skills/types.ts` |

### 技能位置

- **内置技能：** `/skills/` (47+ 个技能)
- **托管技能：** `~/.openclaw/skills/`
- **工作区技能：** `<workspace>/skills/`

## 贡献与反馈

如果您在阅读文档时发现任何问题或有改进建议，欢迎：

1. 提交 Issue 到 [GitHub Issues](https://github.com/bindoon/openclaw/issues)
2. 提交 Pull Request 改进文档
3. 在 Discord 社区讨论

---

**文档版本：** 2026-02-10

**基于代码版本：** OpenClaw main 分支

**文档作者：** GitHub Copilot (代码分析) + 人工审核
