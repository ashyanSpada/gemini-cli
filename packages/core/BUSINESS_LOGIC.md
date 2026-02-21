# Core 核心业务逻辑分析

## 1. 库定位

`@google/gemini-cli-core` 是 Gemini CLI 的业务引擎层，承载：

- 运行时配置与依赖装配（Config）
- 会话与模型交互（GeminiClient / GeminiChat / Turn）
- 工具系统（注册、策略检查、确认、执行、回写）
- 扩展机制（MCP、Extensions、Skills、Hooks、Agents）

对上层（CLI/SDK）暴露统一能力，对下游（模型、工具、策略）进行编排。

## 2. 架构主轴

1. `Config`（`src/config/config.ts`）
   - 统一持有运行参数、状态、服务实例
   - 构造阶段注入 `GeminiClient`、`PolicyEngine`、`MessageBus`、`Storage` 等
   - `initialize()`
     执行真正装配：ToolRegistry、MCP、Extensions、Skills、Hooks、GeminiClient

2. `GeminiClient`（`src/core/client.ts`）
   - 作为单会话编排器，负责发送消息、路由模型、压缩上下文、循环检测、hook 前后置
   - 将 `Turn` 产生的底层流式事件提升为统一 `GeminiEventType` 事件流

3. `GeminiChat`（`src/core/geminiChat.ts`）
   - 管理历史消息、system instruction、tool declarations
   - 与模型 API 流式交互，处理无效响应重试与录制

4. `Scheduler`（`src/scheduler/scheduler.ts`）
   - 事件驱动工具调度器：校验 → 策略 → 确认 → 执行 → 完成态收敛
   - 支持批处理、排队、取消、进度回传

## 3. Config 初始化主链路

关键路径：`Config.initialize()` / `_initialize()`（`src/config/config.ts`）

- 初始化 `Storage`、Workspace context、可选 plans 目录
- 创建 `PromptRegistry` / `ResourceRegistry` / `AgentRegistry`
- `createToolRegistry()`
  注册核心工具（ls/read/grep|ripgrep/edit/write/shell/web 等）
- 启动 `McpClientManager` 与 extension loader
- 发现并装配 skills，必要时重注册 `ActivateSkillTool`
- 初始化 hook system 与可选 JIT context
- 最后 `geminiClient.initialize()` 建立对话引擎

`createToolRegistry()`
是能力拼装关键：按配置与可用性动态选择工具（例如 ripgrep 不可用时回退 grep），并将 subagent 也注册为工具。

## 4. 模型对话主链路

关键路径：`GeminiClient.sendMessageStream()`（`src/core/client.ts`）

1. 会话控制与 hook 前置
   - 处理 prompt_id 切换、loop detector 重置
   - 触发 BeforeAgent hook（可能 stop/block 或注入 additional context）

2. 单轮处理（`processTurn()`）
   - 预算检查：上下文压缩、token 估算、窗口溢出预警
   - IDE 场景下拼接 editor context（全量或增量）
   - 模型选择：路由服务 + 可用性策略 + model stickiness
   - 执行 `Turn.run(...)` 获取流式事件，监控 invalid stream / error / loop

3. 自动续写与后置 hook
   - 无 pending tools 时可做 next-speaker 检查，必要时触发 “Please
     continue” 递归轮次
   - 触发 AfterAgent hook 并处理附加结果

该链路本质是“可恢复的多轮状态机”，不是一次性请求。

## 5. 工具调度主链路

关键路径：`Scheduler.schedule()`（`src/scheduler/scheduler.ts`）

- 接收一批 `ToolCallRequestInfo`
- 将请求转为内部状态对象（含 tool build 校验）
- 不存在工具时生成标准 functionResponse 错误回包
- 依次经过策略检查、确认请求、执行器运行、状态迁移
- 输出 `CompletedToolCall[]`，供上游回灌给模型继续推理

对于 Agent 场景，`scheduleAgentTools()`（`src/agents/agent-scheduler.ts`）通过代理
`Config.getToolRegistry()` 为当前 agent 注入作用域工具集，再复用同一
`Scheduler`。

## 6. 业务价值与边界

- Core 抽象了“模型 + 工具 + 策略 + 生态扩展”的统一执行语义
- CLI/SDK 可以只关心输入输出体验，而复用同一执行内核
- 扩展能力（MCP、skills、hooks、agents）通过注册和策略系统接入，不侵入主链路

## 7. 关键文件

- `src/config/config.ts`：运行时配置与能力装配
- `src/core/client.ts`：会话编排主引擎
- `src/core/geminiChat.ts`：与模型交互及历史管理
- `src/core/turn.ts`：单轮流式事件执行
- `src/scheduler/scheduler.ts`：事件驱动工具调度
- `src/agents/agent-scheduler.ts`：agent 场景下的工具调度入口
