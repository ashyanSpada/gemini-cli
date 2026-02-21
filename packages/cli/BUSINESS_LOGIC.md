# CLI 核心业务逻辑分析

## 1. 库定位

`@google/gemini-cli` 是终端应用层，负责：

- 启动流程编排（参数、设置、鉴权、沙箱、会话恢复）
- 交互式 UI（Ink）与非交互模式（文本/JSON/流式 JSON）
- 将用户输入转交 `@google/gemini-cli-core`，并把 Core 事件渲染为终端输出

核心入口为 `src/gemini.tsx` 的 `main()`。

## 2. 启动主链路（main）

关键路径：`src/gemini.tsx`

1. 初始化基础运行环境
   - 监听 admin controls IPC（`setupAdminControlsListener`）
   - patch stdio、注册清理钩子、注册 unhandled rejection 处理
   - 加载 settings / trusted folders、清理旧 checkpoint 与会话产物

2. 解析参数并进行前置校验
   - `parseArguments()` 解析命令行
   - 检查参数组合合法性（例如 `--prompt-interactive` + pipe stdin）
   - 根据 settings 修正 auth 默认类型（例如 Cloud Shell → `COMPUTE_ADC`）

3. 构建部分配置并前置鉴权
   - 先 `loadCliConfig()` 得到 partial config
   - 在进入沙箱前尝试 `refreshAuth()`，并获取远端 admin settings
   - 执行 deferred command（依赖 admin settings）

4. 沙箱/子进程重启策略
   - 若启用沙箱：拼接 stdin 与参数后 `start_sandbox(...)`
   - 否则按内存参数执行 `relaunchAppInChildProcess(...)`

5. 重建完整 Config 并分流执行
   - 再次 `loadCliConfig()`（完整初始化）+ `config.storage.initialize()`
   - 处理管理型命令：`--list-extensions`、`--list-sessions`、`--delete-session`
   - `initializeApp()` 完成主题/鉴权/IDE 连接前置初始化
   - 分流：
     - 交互式：`startInteractiveUI(...)`
     - 非交互：`runNonInteractive(...)`

## 3. 交互式业务逻辑

关键路径：`startInteractiveUI()`（`src/gemini.tsx`）

- 决定是否进入 alternate buffer、启停鼠标事件、设置窗口标题
- 使用 `ConsolePatcher` 将 console 日志转为 Core 事件
- 通过多层 Provider（Settings/Keypress/Mouse/Session/Terminal 等）注入 UI 运行时上下文
- 渲染 `AppContainer`，UI 内部再驱动会话、工具确认与流式展示

本质上，交互式模式是“状态管理 + 事件渲染壳”，模型与工具调度能力主要来自 Core。

## 4. 非交互式业务逻辑

关键路径：`src/nonInteractiveCli.ts` 的 `runNonInteractive()`

1. 环境准备
   - patch console、建立输出器 `TextOutput`
   - 注册 user feedback 监听与 Ctrl+C 取消（`AbortController`）

2. 输入预处理
   - 先处理 slash command（`handleSlashCommand`）
   - 再处理 `@` 引用（`handleAtCommand`）

3. 模型-工具循环
   - 调用 `geminiClient.sendMessageStream(...)` 获取流式事件
   - 收集 `ToolCallRequest` 后交给 `Scheduler.schedule(...)`
   - 将 tool response parts 回灌为下一轮输入，直到无工具调用

4. 输出协议
   - 支持 TEXT / JSON / STREAM_JSON 三种输出协议
   - 针对终止、阻断、错误、最大轮数等事件有明确分支

该流程体现了 CLI 的核心职责：将 Core 的事件流编排为可消费的终端行为。

## 5. 与 Core 的边界

- CLI 负责：启动、参数、UI、I/O、进程控制、运行模式分流
- Core 负责：模型请求、上下文管理、路由、工具注册与调度、策略与钩子

因此 CLI 是“应用编排层”，Core 是“智能执行引擎层”。

## 6. 关键文件

- `src/gemini.tsx`：总入口、启动编排、交互/非交互分流
- `src/nonInteractiveCli.ts`：批处理模式的模型-工具循环
- `src/core/initializer.ts`：UI 渲染前初始化（鉴权/主题/IDE）
- `src/config/*`：参数与 settings 加载
