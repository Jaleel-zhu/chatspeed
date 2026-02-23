# 工作流模块重构规划 (Vercel AI SDK 替换方案)

## 1. 目标
使用 **Vercel AI SDK (Core)** 彻底替换现有的手动 ReAct 引擎 (`engine.ts`, `llm.ts`, `stateMachine.ts`)。
*   **代码精简**: 预计缩减 60% 以上的样板代码。
*   **稳定性**: 利用 SDK 内置的 `maxSteps` 自动处理 ReAct 循环。
*   **类型安全**: 使用 **Zod** 替代手写的 JSON Schema 进行工具参数校验。
*   **高性能流式**: 使用 SDK 标准的流处理机制，提升 UI 响应速度。

## 2. 技术栈
*   **核心库**: `ai` (Vercel AI SDK Core)
*   **适配器**: `@ai-sdk/openai` (连接本地 `ccproxy` 11435 端口)
*   **校验**: `zod`

## 3. 核心重构策略

### A. 基础设施层
1.  **安装依赖**: `yarn add ai zod @ai-sdk/openai`
2.  **Provider 配置**: 在 `llm.ts` 或新文件中定义指向 `ccproxy` 的 OpenAI 兼容适配器。
    ```typescript
    import { createOpenAI } from '@ai-sdk/openai';
    const chatspeedProxy = createOpenAI({
      baseURL: 'http://localhost:11435/v1',
      apiKey: 'your-proxy-key',
    });
    ```

### B. 工具定义层 (`src/pkg/workflow/tools/`)
*   将现有的 `ToolDefinition` 转换为 AI SDK 的 `tool` 格式。
*   使用 `z.object({...})` 替换 `inputSchema`。
*   **示例**:
    ```typescript
    const getWeather = tool({
      description: 'Get weather for a location',
      parameters: z.object({ loc: z.string() }),
      execute: async ({ loc }) => { /* 调用现有 Rust 或 TS 逻辑 */ }
    });
    ```

### C. 引擎层 (`src/pkg/workflow/engine.ts`)
*   **废弃**: 手动的 `switch-case` 状态跳转逻辑。
*   **采用**: `streamText` 的 `maxSteps` 模式。
    ```typescript
    const result = await streamText({
      model: chatspeedProxy('model-id'),
      tools: { getWeather, ... },
      maxSteps: 10, // 自动 ReAct 循环
      onStepFinish: async ({ text, toolCalls, toolResults }) => {
        // 在每一步结束时，通过 api.ts 将消息同步到 Tauri 后端数据库
      },
      onFinish: async ({ text }) => {
        // 最终任务完成处理
      }
    });
    ```

### D. 状态管理层
*   简化 `stateMachine.ts`。由于 AI SDK 接管了中间步骤，状态机只需维护：`IDLE` -> `RUNNING` -> `PAUSED (等待审批)` -> `FINISHED` -> `ERROR`。

## 4. 实施步骤

### 第一阶段: 环境与基础
1.  安装 `ai`, `zod`, `@ai-sdk/openai`。
2.  在 `types.ts` 中更新类型定义，兼容 SDK 的 `LanguageModel` 接口。

### 第二阶段: 工具迁移 (关键)
1.  编写一个适配器，将现有的 TypeScript 工具封装为 SDK 可用的 `tool` 对象。
2.  重点重构 `todoList` 和 `webAnalytics` 两个复杂工具。

### 第三阶段: Engine V2 开发
1.  实现 `engine_v2.ts`，使用 `streamText` 替换原本的递归/循环调用。
2.  对接 `api.ts`，确保 SDK 产生的每一个 `step` 都能正确持久化到本地 SQLite。

### 第四阶段: 审批逻辑集成
1.  利用 SDK 的 `onStepFinish` 拦截需要人工审批的工具。
2.  实现工作流的暂停与恢复（利用 SDK 的 `initialMessages` 恢复上下文）。

## 5. 注意事项
*   **端口一致性**: 确保 SDK 始终连接 `localhost:11435`。
*   **Token 统计**: AI SDK 返回的 `usage` 信息非常精准，需将其同步到现有的统计模块。
*   **错误处理**: 针对网络超时或模型拒绝调用工具的情况，利用 SDK 内置的重试机制。

## 6. 架构演进与高级优化 (2026.02 补充)

### A. 全工具驱动 (Everything is a Tool)
*   **理念**: 模型不应直接输出文本结束任务，每一步操作（包括最终回复）必须通过调用工具完成。
*   **实现**:
    *   **强制收尾**: 在 System Prompt 中明确要求使用 `TaskComplete` 工具交付最终结果，杜绝无意义的结尾文本。
    *   **状态闭环**: `TaskComplete` 工具的调用将作为状态机从 `RUNNING` 切换至 `FINISHED` 的唯一信号。

### B. 上下文管理与子代理 (Sub-Agents & Context Optimization)
*   **痛点**: 长文本（如网页抓取、代码阅读）极易挤爆 Context Window，且会导致主代理注意力分散。
*   **解决方案**:
    1.  **智能 Observation 压缩器**:
        *   在 `handleToolResult` 层增加拦截器。
        *   当工具返回结果过大（如 >2000 字符）时，触发后台轻量级模型（Summary Model）进行摘要。
        *   仅将“摘要 + 原文索引”喂回给主代理，避免 Token 爆炸。
    2.  **意图注入 (Intent Injection)**:
        *   在调用子代理（或摘要工具）时，必须传入主代理当前的**任务背景**。
        *   避免“无灵魂摘要”，确保提取的信息是主代理真正需要的。
    3.  **专用子代理**:
        *   针对复杂任务（如搜索、代码库扫描），将其封装为独立的子工作流。
        *   主代理只与子代理的“高层结论”交互，而非原始数据。

### C. 结果包装与反馈循环 (Feedback Loop)
*   **理念**: 将“冷冰冰的报错”转化为“有指导意义的提示”。
*   **实现**:
    *   **错误拦截**: 当工具执行失败（如搜索无果、JSON 解析错误）时，不直接返回 Error。
    *   **动态建议**: 包装成 `Observation`，并附带 `System Hint`（例如：“搜索未找到结果，建议尝试减少关键词或翻译成英文后重试”）。
    *   **增强模型自愈力**: 帮助弱模型在下一轮 ReAct 中自我修正参数。

---
**准备就绪**: 您可以随时启动新会话，只需告诉下一个 AI 助手“按照根目录的 workflow_plan.md 开始重构工作流模块”即可。