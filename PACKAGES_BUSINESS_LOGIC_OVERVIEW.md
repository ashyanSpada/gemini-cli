# Gemini CLI 三库横向对比总览

本文用于横向对比 `packages/cli`、`packages/core`、`packages/sdk`
的职责边界与协作方式。

## 1. 三库职责矩阵

| 维度                         | `packages/cli`                | `packages/core`                 | `packages/sdk`                     |
| ---------------------------- | ----------------------------- | ------------------------------- | ---------------------------------- |
| 主要定位                     | 终端应用层（交互/非交互入口） | 智能执行引擎层                  | 开发者编程接口层                   |
| 目标用户                     | CLI 终端用户                  | CLI/SDK 内部能力提供方          | 业务开发者/集成方                  |
| 入口形态                     | 命令行主程序（`main`）        | 类库（`Config`/`GeminiClient`） | 类库（`GeminiCliAgent`/`Session`） |
| 是否负责 UI                  | 是（Ink UI、终端输出）        | 否                              | 否                                 |
| 是否负责模型会话编排         | 间接（调用 Core）             | 是（`GeminiClient` + `Turn`）   | 间接（复用 Core）                  |
| 是否负责工具调度             | 间接（触发 Scheduler）        | 是（`Scheduler`）               | 间接（`scheduleAgentTools`）       |
| 扩展生态（MCP/hooks/skills） | 参与配置与启动                | 核心承载与执行                  | 选择性启用，默认轻量               |
| 典型输出                     | 终端文本/JSON/流事件          | 统一业务事件流与执行结果        | 可编程流式事件与上下文接口         |

## 2. 端到端调用关系

### 2.1 CLI 场景

`packages/cli`（参数/鉴权/UI/模式分流）→
`packages/core`（会话、路由、工具、策略）→ 模型与工具执行 →
`packages/cli`（终端渲染与输出）

### 2.2 SDK 场景

业务代码→ `packages/sdk`（Agent/Session/Tool 封装）→
`packages/core`（同一执行内核）→ 模型与工具执行 →
`packages/sdk`（流式回传给业务代码）

## 3. 关键边界（避免职责混淆）

- CLI 不实现模型内核，只做应用编排与 I/O 体验。
- Core 不关心 UI 展示，只关心“请求-推理-工具-策略”的执行语义。
- SDK 不重复造轮子，只做 Core 的开发者友好封装（API、上下文、工具适配）。

## 4. 变更影响面参考

- 改启动参数/交互体验：优先看 `packages/cli`
- 改模型行为/工具策略/会话链路：优先看 `packages/core`
- 改嵌入式用法/API 设计：优先看 `packages/sdk`

## 5. 关联文档

- CLI 详版：[packages/cli/BUSINESS_LOGIC.md](packages/cli/BUSINESS_LOGIC.md)
- Core 详版：[packages/core/BUSINESS_LOGIC.md](packages/core/BUSINESS_LOGIC.md)
- SDK 详版：[packages/sdk/BUSINESS_LOGIC.md](packages/sdk/BUSINESS_LOGIC.md)
