# SDK 核心业务逻辑分析

## 1. 库定位

`@google/gemini-cli-sdk`
是面向开发者的轻量封装层，目标是以编程 API 复用 Core 能力，而不是重写一套 agent 引擎。

导出入口：`src/index.ts`

- `GeminiCliAgent`
- `GeminiCliSession`
- 工具定义辅助（`tool` / `SdkTool`）
- skills 引用与基础类型

## 2. 对象模型

1. `GeminiCliAgent`（`src/agent.ts`）
   - 负责创建新会话 `session()`
   - 支持 `resumeSession(sessionId)`：从 Core `Storage` 恢复历史会话

2. `GeminiCliSession`（`src/session.ts`）
   - 管理单会话生命周期：初始化、发送消息流、工具回合循环
   - 内部持有一个精简配置版 Core `Config`

3. `SdkTool`（`src/tool.ts`）
   - 将开发者定义的函数工具适配为 Core 可识别的 DeclarativeTool
   - 负责 Zod schema → JSON schema 以及执行结果/错误格式化

## 3. 会话初始化链路

关键路径：`GeminiCliSession.initialize()`

- 组装 `ConfigParameters`（默认关闭 hooks/mcp/extensions，保持 SDK 轻量）
- 执行 `config.refreshAuth(...)` 与 `config.initialize()`
- 加载外部 skills 目录（`loadSkillsFromDir`）
- 若有技能则重注册 `ActivateSkillTool` 以刷新枚举 schema
- 将 SDK 自定义工具封装为 `SdkTool` 注册进 `ToolRegistry`
- 如果是恢复会话，转换历史消息并 `client.resumeChat(...)`

这一步的核心思想：SDK 只做“最小装配”，把复杂执行交给 Core。

## 4. 推理与工具闭环

关键路径：`GeminiCliSession.sendStream(prompt)`

1. 首轮请求
   - 初始 request = `[{ text: prompt }]`
   - 若 `instructions` 是函数，每轮基于 `SessionContext` 动态生成 system memory

2. 消息流消费
   - 调用 `client.sendMessageStream(...)`
   - 将事件直接 `yield` 给 SDK 使用方
   - 收集模型发出的 `ToolCallRequest`

3. 工具执行与回灌
   - 为 SDK 工具绑定上下文（`fs/shell/transcript/session`）
   - 调用 Core `scheduleAgentTools(...)` 执行工具
   - 提取 function response parts 作为下一轮 request
   - 直到无工具调用，循环结束

这是标准 ReAct 样式的“模型思考-工具执行-结果回灌”循环，SDK 只是把流程封装成易用 API。

## 5. SDK 提供的运行时上下文能力

- 文件能力：`SdkAgentFilesystem`（`src/fs.ts`）
  - 基于 Core 的路径访问校验，提供 `readFile/writeFile`
- Shell 能力：`SdkAgentShell`（`src/shell.ts`）
  - 先走 `ShellTool` 策略确认，再用 `ShellExecutionService` 执行
- 工具上下文：`SessionContext`（`src/types.ts`）
  - 提供 `sessionId/transcript/cwd/timestamp/fs/shell/agent/session`

## 6. 与 Core 的边界

- SDK 不直接处理底层模型协议、上下文压缩、策略引擎实现
- SDK 负责开发者体验：会话对象、工具定义、上下文注入、流式迭代接口
- 复杂能力全部委托给 Core，保证 CLI 与 SDK 行为一致性

## 7. 关键文件

- `src/index.ts`：对外导出面
- `src/agent.ts`：Agent 与会话恢复
- `src/session.ts`：主业务链路（初始化 + sendStream）
- `src/tool.ts`：自定义工具适配层
- `src/fs.ts`、`src/shell.ts`、`src/types.ts`：上下文能力与类型
