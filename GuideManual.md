# Pi 项目学习指导手册

本文档基于项目真实代码、README、package 配置、git 历史、changelog、测试目录和 GitHub issue/PR 当前状态整理。信息读取时间为 2026-07-09。

## 1. 项目定位

Pi 是一个 TypeScript monorepo，核心目标是构建一个可扩展的 AI coding agent harness。用户实际使用的是 `pi` CLI，它可以在终端里和模型交互，读取/修改文件、执行 bash、保存会话、恢复会话、加载扩展、运行 RPC 模式等。

推荐先阅读这些顶层文件：

- `README.md`
- `package.json`
- `CONTRIBUTING.md`
- `AGENTS.md`

这些文件分别回答：

- 项目是什么。
- 如何安装、构建、检查和发布。
- 如何贡献 issue/PR。
- 开发时必须遵守哪些项目规则。

## 2. 技术栈

- 语言：TypeScript，ESM。
- 运行时：Node.js `>=22.19.0`。
- 包管理：npm workspaces。
- 类型检查：`tsgo --noEmit`。
- 格式和 lint：Biome。
- 测试：Vitest、Node test runner、项目自定义 `./test.sh`。
- CLI/TUI：自研 terminal UI，不依赖 React/Ink。
- 模型 API：自研统一 LLM provider 抽象。
- 会话存储：JSONL，append-only tree。
- 发布：lockstep versioning，所有 package 同版本发布。

常用命令：

```bash
npm install --ignore-scripts
npm run check
./test.sh
./pi-test.sh
npm run release:local
npm run release:patch
npm run release:minor
```

开发时最重要的是：

- 改代码后运行 `npm run check`。
- 非 e2e 测试优先运行 `./test.sh`。
- 不要随意运行全量 `npm test`，因为其中可能包含依赖真实 endpoint/auth 的 e2e 测试。
- 不要直接修改 `packages/ai/src/models.generated.ts`，需要修改生成脚本后重新生成。

## 3. Monorepo 包结构

项目主要由以下 package 组成：

| Package | 作用 |
| --- | --- |
| `packages/ai` | 统一 LLM API、provider、model catalog、auth、streaming protocol |
| `packages/agent` | 无 UI 的 agent runtime，负责 agent loop、状态、工具调用 |
| `packages/coding-agent` | `pi` CLI 产品层，负责 session、tools、modes、extensions、settings |
| `packages/tui` | 自研 terminal UI 库，负责终端渲染、输入、overlay、markdown、图片 |
| `packages/orchestrator` | 实验性 orchestrator 包，当前 API/CLI 不稳定 |

学习时建议先把 `packages/orchestrator` 放到最后。它的 README 明确说明该包仍在实验中，可能改变或移除。

## 4. `packages/ai`：模型和 Provider 层

`packages/ai` 是底层模型 API 兼容层。它负责把不同厂商、不同 API 风格统一成 Pi 内部使用的消息、模型、工具调用和流事件格式。

核心文件：

- `packages/ai/src/types.ts`
- `packages/ai/src/models.ts`
- `packages/ai/src/providers/all.ts`
- `packages/ai/src/models.generated.ts`

关键概念：

### 4.1 `Model<TApi>`

`Model` 描述一个模型的元数据，包括：

- `id`
- `name`
- `provider`
- `api`
- `baseUrl`
- `reasoning`
- `thinkingLevelMap`
- `input`
- `cost`
- `contextWindow`
- `maxTokens`
- `headers`
- `compat`

Pi 不是只保存模型名称，而是把模型能力、成本、上下文窗口、reasoning 支持等信息都作为运行时决策依据。

### 4.2 `Provider`

`Provider` 是具体运行时 provider。它负责：

- 暴露 provider id/name。
- 提供 auth 规则。
- 返回当前已知模型列表。
- 刷新动态模型列表。
- 执行 `stream()` 和 `streamSimple()`。

### 4.3 `Models`

`Models` 是 provider 集合。它负责：

- 注册 provider。
- 查找 provider。
- 查找 model。
- 刷新 model catalog。
- 统一解析 auth。
- 把请求分发给对应 provider。

新架构中，`Models` 是显式依赖，不再依赖旧式全局 provider registry。这是 0.80 系列的重要架构变化。

### 4.4 统一流事件

所有 provider 最终都要产出统一的 `AssistantMessageEventStream`。事件包括：

- `start`
- `text_start`
- `text_delta`
- `text_end`
- `thinking_start`
- `thinking_delta`
- `thinking_end`
- `toolcall_start`
- `toolcall_delta`
- `toolcall_end`
- `done`
- `error`

这让上层 agent loop 不需要关心 OpenAI、Anthropic、Google、Bedrock 等厂商的原始协议差异。

### 4.5 推荐阅读顺序

1. `packages/ai/src/types.ts`
2. `packages/ai/src/models.ts`
3. `packages/ai/src/providers/all.ts`
4. 任意一个 provider，例如 `packages/ai/src/providers/openai.ts`
5. 对应 API adapter，例如 `packages/ai/src/api/openai-responses.ts`

学习目标：

- 能解释模型元数据如何组织。
- 能解释 provider 如何注册。
- 能解释 auth 如何解析。
- 能解释厂商 stream 如何变成统一事件流。

## 5. `packages/agent`：无 UI Agent Runtime

`packages/agent` 是 agent 的核心运行时。它不知道 CLI、TUI、session 文件、扩展系统长什么样，只负责 agent loop、消息状态、工具调用和事件生命周期。

核心文件：

- `packages/agent/src/types.ts`
- `packages/agent/src/agent-loop.ts`
- `packages/agent/src/agent.ts`

### 5.1 核心流程

一次用户 prompt 的主流程是：

1. 用户消息进入 `Agent.prompt()`。
2. `runAgentLoop()` 发出 `agent_start`、`turn_start`。
3. prompt message 被加入上下文。
4. `streamAssistantResponse()` 调用 `streamSimple()` 请求模型。
5. assistant message 通过事件流持续更新。
6. 如果 assistant 返回 tool calls，agent loop 执行工具。
7. tool result 写回上下文。
8. 如果还有 tool call、steering message 或 follow-up message，继续下一轮。
9. 无更多工作后发出 `agent_end`。

### 5.2 工具执行模型

工具默认并行执行。相关配置：

- `toolExecution: "parallel"`
- `toolExecution: "sequential"`
- 单个工具也可以声明 `executionMode: "sequential"`。

重要 hook：

- `beforeToolCall`
- `afterToolCall`
- `prepareNextTurn`
- `prepareNextTurnWithContext`
- `shouldStopAfterTurn`

这些 hook 是 `coding-agent` 和扩展系统接入底层 agent loop 的关键点。

### 5.3 队列机制

Agent 支持两种队列：

- steering queue：当前 assistant turn 完成工具调用后插入。
- follow-up queue：agent 本来准备结束时再插入。

这使得用户或扩展可以在 agent 工作中继续追加消息。

### 5.4 截断工具调用保护

如果 assistant message 的 `stopReason` 是 `length`，Pi 不会执行该消息里的 tool calls。原因是 tool call 参数可能被截断，即使 JSON 能被解析，也可能语义不完整。此时 Pi 会为每个 tool call 生成错误 tool result，让模型重新发起完整工具调用。

这是一个非常重要的安全边界。

## 6. `packages/coding-agent`：CLI 产品层

`packages/coding-agent` 是用户实际使用的 `pi` CLI。它把 `agent-core`、`pi-ai`、TUI、session、扩展、工具、配置等能力组装成完整应用。

核心文件：

- `packages/coding-agent/src/cli.ts`
- `packages/coding-agent/src/main.ts`
- `packages/coding-agent/src/core/agent-session.ts`
- `packages/coding-agent/src/core/agent-session-runtime.ts`
- `packages/coding-agent/src/core/agent-session-services.ts`

### 6.1 CLI 启动流程

`cli.ts` 很薄，主要做：

- 设置 `process.title`。
- 设置 `PI_CODING_AGENT` 环境变量。
- 配置 HTTP dispatcher。
- 调用 `main(process.argv.slice(2))`。

真正的启动逻辑在 `main.ts`。

`main.ts` 负责：

1. 解析 CLI 参数。
2. 处理 `install`、`update`、`config` 等 package command。
3. 处理 `--help`、`--version`、`--export`。
4. 判断运行模式：interactive、print、json、rpc。
5. 创建或恢复 session。
6. 处理 project trust。
7. 加载 settings、auth、extensions、skills、themes、prompt templates、context files。
8. 解析 model 和 scoped models。
9. 创建 `AgentSessionRuntime`。
10. 分流到 `InteractiveMode`、`runPrintMode()` 或 `runRpcMode()`。

### 6.2 运行模式

Pi 有三种主要运行模式：

#### Interactive Mode

文件：

- `packages/coding-agent/src/modes/interactive/interactive-mode.ts`

这是完整终端 UI 模式。它负责：

- 输入框。
- 消息流式渲染。
- 工具调用渲染。
- model selector。
- settings selector。
- session selector。
- tree selector。
- extension UI。
- theme。

#### Print Mode

文件：

- `packages/coding-agent/src/modes/print-mode.ts`

用于单次运行：

```bash
pi -p "prompt"
pi --mode json "prompt"
```

文本模式只输出最后 assistant 文本。JSON 模式会输出 session header 和事件流。

#### RPC Mode

文件：

- `packages/coding-agent/src/modes/rpc/rpc-mode.ts`

RPC 模式用 stdin/stdout JSON 协议暴露 agent 能力，适合外部应用嵌入。它会把 session events、extension UI requests、command responses 都序列化成 JSON line。

### 6.3 `AgentSession`

文件：

- `packages/coding-agent/src/core/agent-session.ts`

`AgentSession` 是产品层最重要的类。它封装：

- agent state。
- session persistence。
- model 管理。
- thinking level 管理。
- queue 管理。
- compaction。
- retry。
- bash execution。
- session tree navigation。
- extension binding。
- system prompt rebuild。
- tool registry。

可以把它理解成：`Agent` 负责“如何跑一轮 agent”，`AgentSession` 负责“作为 coding agent 产品时如何管理一整个会话”。

## 7. 内置工具系统

工具定义在：

- `packages/coding-agent/src/core/tools/index.ts`
- `packages/coding-agent/src/core/tools/read.ts`
- `packages/coding-agent/src/core/tools/bash.ts`
- `packages/coding-agent/src/core/tools/edit.ts`
- `packages/coding-agent/src/core/tools/write.ts`
- `packages/coding-agent/src/core/tools/grep.ts`
- `packages/coding-agent/src/core/tools/find.ts`
- `packages/coding-agent/src/core/tools/ls.ts`
- `packages/coding-agent/src/core/tools/file-mutation-queue.ts`

默认激活工具：

- `read`
- `bash`
- `edit`
- `write`

额外内置工具：

- `grep`
- `find`
- `ls`

### 7.1 `read`

`read` 支持读取文本和图片。

文本读取特点：

- 支持 `offset`。
- 支持 `limit`。
- 默认按最大行数和最大字节数截断。
- 输出会提示下一次应该从哪个 offset 继续。

图片读取特点：

- 支持 jpg、png、gif、webp、bmp 等。
- 可自动 resize。
- 如果当前模型不支持图片，会返回提示文本。

### 7.2 `bash`

`bash` 用于执行 shell 命令。

特点：

- 支持 timeout。
- stdout/stderr 合并输出。
- 支持流式 partial update。
- 输出过大时保留尾部，并把完整输出保存到临时文件。
- 支持 command prefix 和自定义 shell。
- 会在 abort 或 timeout 时尝试杀掉进程树。

### 7.3 `edit`

`edit` 用精确文本替换修改单个文件。

特点：

- 输入是 `edits[]`。
- 每个 `oldText` 必须唯一匹配。
- 多个 edit 基于原始文件匹配，不是按顺序增量匹配。
- 不允许重叠或嵌套 edit。
- 会生成 diff 和 patch。
- 渲染层会在工具参数完整后预览 diff。

这是 Pi 修改文件时最重要的工具。

### 7.4 `write`

`write` 用于创建或完整覆盖文件。

特点：

- 自动创建父目录。
- 适合新文件或完整重写。
- 不适合小范围修改；小范围修改应该用 `edit`。

### 7.5 文件写入串行化

`edit` 和 `write` 都通过 `withFileMutationQueue()` 串行化同一文件的写入。

这样可以允许不同文件并行修改，同时避免多个并行 tool calls 同时写同一个文件。

## 8. 会话系统

核心文件：

- `packages/coding-agent/src/core/session-manager.ts`

Pi 的 session 是 JSONL append-only tree。

### 8.1 文件结构

第一行是 session header：

- `type`
- `version`
- `id`
- `timestamp`
- `cwd`
- `parentSession`

后续每行是 session entry。每个 entry 都有：

- `type`
- `id`
- `parentId`
- `timestamp`

### 8.2 Entry 类型

主要 entry 类型：

- `message`
- `thinking_level_change`
- `model_change`
- `compaction`
- `branch_summary`
- `custom`
- `custom_message`
- `label`
- `session_info`

### 8.3 Tree 结构

每个 entry 有 `parentId`，所以一个 session 文件可以表示树状历史。

这支持：

- resume。
- fork。
- tree navigation。
- branch summary。
- label。
- 在旧节点处分叉继续对话。

### 8.4 Context 构建

`buildSessionContext()` 会从当前 leaf 反向追溯到 root，再整理成发给模型的 messages。

它会处理：

- 当前分支路径。
- model/thinking 状态。
- compaction summary。
- branch summary。
- custom_message。

普通 `custom` entry 不进入模型上下文，主要用于扩展持久化状态。

## 9. Compaction 和上下文管理

相关文件：

- `packages/coding-agent/src/core/agent-session.ts`
- `packages/coding-agent/src/core/compaction/index.ts`

Pi 支持几种压缩场景：

- 手动 `/compact`。
- 上下文超过阈值。
- provider 返回 context overflow。
- tree navigation 时生成 branch summary。

### 9.1 Manual Compaction

用户手动触发时：

1. 先 abort 当前 agent operation。
2. 根据当前 branch 准备 compaction。
3. 允许扩展通过 hook 接管或取消。
4. 生成 summary。
5. append compaction entry。
6. 重建 agent state messages。

### 9.2 Auto Compaction

自动压缩主要分两种：

- `overflow`：模型报上下文溢出。
- `threshold`：上下文接近配置阈值。

overflow 场景下可能会自动 retry。threshold 场景通常只压缩，不自动继续。

### 9.3 Compaction 的关键设计

Pi 不直接删除旧消息，而是插入 compaction entry。构建上下文时，最新 compaction 会代表之前一段历史，同时保留 `firstKeptEntryId` 之后的必要 entry。

这让 session 文件仍然保留完整历史，同时发给模型的上下文被压缩。

## 10. 扩展系统

相关文件：

- `packages/coding-agent/src/core/extensions/types.ts`
- `packages/coding-agent/src/core/extensions/runner.ts`
- `packages/coding-agent/src/core/extensions/loader.ts`

扩展是 Pi 架构中的重要部分。贡献规则也强调：Pi core 要保持 minimal，很多功能应优先做成 extension。

扩展可以：

- 注册 slash command。
- 注册工具。
- 注册 provider。
- 注册 CLI flags。
- 拦截 input。
- 监听 agent/session/tool/model/thinking events。
- 修改系统提示词。
- 增加 skills、prompt templates、themes。
- 参与 compaction。
- 参与 tree navigation。
- 提供 interactive/RPC/print 下不同的 UI 行为。

建议阅读：

- `packages/coding-agent/examples/extensions/README.md`
- `packages/coding-agent/examples/extensions/plan-mode/README.md`
- `packages/coding-agent/examples/extensions/subagent/README.md`
- `packages/coding-agent/examples/extensions/custom-provider-anthropic/README.md`

## 11. 配置、资源和系统提示词

相关文件：

- `packages/coding-agent/src/core/settings-manager.ts`
- `packages/coding-agent/src/core/resource-loader.ts`
- `packages/coding-agent/src/core/skills.ts`
- `packages/coding-agent/src/core/system-prompt.ts`

Pi 会加载多类资源：

- global settings。
- project settings。
- extensions。
- skills。
- prompt templates。
- themes。
- `AGENTS.md` / `CLAUDE.md`。
- CLI `--system-prompt`。
- CLI `--append-system-prompt`。

### 11.1 Project Trust

项目本地资源可能影响 agent 行为，因此 Pi 有 project trust 机制。是否加载 project-local resources，会受 trust 状态影响。

相关文件：

- `packages/coding-agent/src/core/project-trust.ts`
- `packages/coding-agent/src/core/trust-manager.ts`
- `packages/coding-agent/src/cli/project-trust.ts`

### 11.2 System Prompt 重建

`AgentSession` 会根据当前工具、skills、context files、extensions、prompt snippets、prompt guidelines 重建 system prompt。

这意味着启用/禁用工具，或 reload extensions/resources，会影响下一轮发给模型的系统提示词。

## 12. `packages/tui`：终端 UI 库

核心文件：

- `packages/tui/src/tui.ts`
- `packages/tui/src/terminal.ts`
- `packages/tui/src/components/editor.ts`
- `packages/tui/src/components/markdown.ts`

`pi-tui` 是自研组件系统。核心接口是：

```ts
interface Component {
  render(width: number): string[];
  handleInput?(data: string): void;
  invalidate(): void;
}
```

它负责：

- 差量渲染。
- 焦点管理。
- overlay。
- 输入处理。
- markdown 渲染。
- Kitty 图片。
- 终端颜色探测。
- 硬件光标。
- CJK/宽字符处理。

学习 TUI 时建议先看：

1. `packages/tui/src/tui.ts`
2. `packages/tui/src/components/text.ts`
3. `packages/tui/src/components/editor.ts`
4. `packages/tui/src/components/select-list.ts`
5. `packages/coding-agent/src/modes/interactive/interactive-mode.ts`

## 13. CLI 参数和用户功能

参数解析在：

- `packages/coding-agent/src/cli/args.ts`

重要能力包括：

- `--provider`
- `--model`
- `--models`
- `--thinking`
- `--print`
- `--mode json`
- `--mode rpc`
- `--continue`
- `--resume`
- `--session`
- `--session-id`
- `--fork`
- `--no-session`
- `--tools`
- `--exclude-tools`
- `--no-tools`
- `--no-builtin-tools`
- `--extension`
- `--skill`
- `--prompt-template`
- `--theme`
- `--export`
- `--list-models`
- `--approve`
- `--no-approve`
- `--offline`

理解这些参数和 `main.ts` 的对应逻辑，可以快速掌握用户功能入口。

## 14. 测试体系

测试覆盖很广，尤其是 `coding-agent/test/suite/regressions` 中有大量 issue-specific regression tests。

推荐先读：

- `packages/agent/test/agent-loop.test.ts`
- `packages/agent/test/agent.test.ts`
- `packages/coding-agent/test/tools.test.ts`
- `packages/coding-agent/test/session-manager/build-context.test.ts`
- `packages/coding-agent/test/agent-session-compaction.test.ts`
- `packages/coding-agent/test/rpc.test.ts`
- `packages/tui/test/tui-render.test.ts`

回归测试目录：

- `packages/coding-agent/test/suite/regressions`

这些测试文件名通常带 issue 编号，例如：

- `1717-2113-agent-session-event-settlement.test.ts`
- `2023-queued-slash-command-followup.test.ts`
- `2835-tools-allowlist-filters-extension-tools.test.ts`
- `5303-bash-output-truncation.test.ts`

很适合反向学习真实 bug、修复方式和系统边界。

## 15. Git 历史和当前维护热点

从最近 git log、changelog、开放 issue/PR 看，当前维护重点集中在：

- provider 兼容性。
- OpenAI Responses / Azure OpenAI / Anthropic / Bedrock / Copilot 等 API 细节。
- reasoning/thinking replay。
- thinking block 处理。
- session tree。
- compaction。
- custom message。
- branch summary。
- extension hooks。
- RPC event 语义。
- Windows/TUI 输入和渲染问题。
- tool call 边界。
- edit 工具成功率。
- 模型 catalog 刷新。
- OAuth/login/auth 可靠性。

截至 2026-07-09，通过 GitHub CLI 读取到开放 issue 约 56 个。开放 issue 标签里 `bug` 最多，其次有 `inprogress` 和 `to-discuss`。

开放 PR 方向包括：

- prompt cache miss tracking。
- constrained sampling。
- 新 provider。
- extension event。
- Anthropic/Vertex/Bedrock 相关 provider 支持。

## 16. 学习路线

### 阶段 1：跑起来并理解包边界

目标：知道项目是什么、怎么启动、包之间怎么分工。

建议：

1. 读 `README.md`。
2. 读顶层 `package.json`。
3. 读各 package 的 `package.json`。
4. 运行：

```bash
npm install --ignore-scripts
./pi-test.sh --help
./pi-test.sh --list-models
./pi-test.sh -p "Say exactly: ok"
```

如果没有配置模型 auth，部分命令可能无法真正调用模型，但仍然可以帮助理解 CLI 行为。

### 阶段 2：读 Agent 主链路

按顺序读：

1. `packages/coding-agent/src/cli.ts`
2. `packages/coding-agent/src/main.ts`
3. `packages/coding-agent/src/core/agent-session.ts`
4. `packages/agent/src/agent.ts`
5. `packages/agent/src/agent-loop.ts`

目标：能解释“一条用户消息如何变成模型请求、工具调用、最终回答”。

### 阶段 3：读工具和 session

按顺序读：

1. `packages/coding-agent/src/core/tools/index.ts`
2. `packages/coding-agent/src/core/tools/read.ts`
3. `packages/coding-agent/src/core/tools/bash.ts`
4. `packages/coding-agent/src/core/tools/edit.ts`
5. `packages/coding-agent/src/core/tools/write.ts`
6. `packages/coding-agent/src/core/session-manager.ts`

目标：能解释工具如何定义 schema、如何执行、如何渲染、如何写入 session。

### 阶段 4：读 Provider 层

按顺序读：

1. `packages/ai/src/types.ts`
2. `packages/ai/src/models.ts`
3. `packages/ai/src/providers/all.ts`
4. 一个具体 provider，例如 `packages/ai/src/providers/openai.ts`
5. 对应 API adapter，例如 `packages/ai/src/api/openai-responses.ts`

目标：能解释“新 provider 怎么接入”。

### 阶段 5：读 UI 和扩展

按顺序读：

1. `packages/coding-agent/src/modes/interactive/interactive-mode.ts`
2. `packages/tui/src/tui.ts`
3. `packages/coding-agent/src/core/extensions/types.ts`
4. `packages/coding-agent/src/core/extensions/runner.ts`
5. `packages/coding-agent/examples/extensions/*`

目标：能写一个最小 extension。

## 17. 推荐练习

### 练习 1：画出 `pi -p "hello"` 的调用链

从 `cli.ts` 开始，追到：

- `main.ts`
- `createAgentSessionRuntime`
- `runPrintMode`
- `AgentSession.prompt`
- `Agent.prompt`
- `runAgentLoop`
- `streamSimple`

### 练习 2：读一个工具的完整生命周期

建议选 `edit`：

1. 看 schema。
2. 看 prompt guidelines。
3. 看 `prepareArguments`。
4. 看 execute。
5. 看 diff 生成。
6. 看 renderCall/renderResult。
7. 看对应测试。

### 练习 3：读一个 session 回归测试

从 `packages/coding-agent/test/suite/regressions` 任选一个带 issue 编号的测试：

1. 读测试名。
2. 读测试内容。
3. 找对应 issue 或 commit。
4. 解释 bug 原因。
5. 解释当前代码如何防止回归。

### 练习 4：写一个最小 extension

目标：

- 注册一个 slash command。
- command 输出一个 custom message。
- reload 后仍能工作。

推荐从 examples 目录复制思路，但不要一开始就研究复杂扩展。

### 练习 5：接一个最小 faux provider

目标：

- 理解 `Provider`。
- 理解 `Models`。
- 理解 `streamSimple()` 的事件协议。

可以先看测试里的 faux provider，再考虑自己写一个最小 provider。

## 18. 参与开发注意事项

项目规则很严格，尤其要注意：

- 改代码后运行 `npm run check`。
- 创建或修改测试文件后，运行对应测试并修到通过。
- 不要直接运行完整 vitest suite。
- 不要直接修改 generated model 文件。
- 不要随意修改 changelog。
- 不要提交未要求的 unrelated refactor。
- 不要用 `any`，除非确实不可避免。
- 不要添加 inline dynamic import。
- TypeScript 源码要使用 Node strip-only 可擦除语法，避免 enum、namespace、parameter properties 等。
- core 保持 minimal，能做成 extension 的功能优先做 extension。

贡献规则里最重要的一句话是：必须理解自己的代码。使用 AI 写代码可以，但不能提交自己无法解释的改动。

## 19. 总结

Pi 的核心可以理解成四层：

1. `pi-ai`：把不同模型厂商统一成同一套模型、消息和流事件协议。
2. `pi-agent-core`：执行 agent loop、工具调用、事件生命周期和队列。
3. `pi-coding-agent`：把 agent runtime 做成真实 CLI 产品，加入 session、tools、extensions、settings、modes。
4. `pi-tui`：提供自研终端 UI 渲染能力。

如果从 0 开始学习，最有效的路线不是先钻每个 provider，而是先掌握：

1. CLI 如何启动。
2. AgentSession 如何包装 Agent。
3. Agent loop 如何流式请求模型并执行工具。
4. 工具如何定义和执行。
5. session 如何存储和重建上下文。
6. provider 如何把不同厂商协议统一成 Pi 的 stream protocol。

掌握这六点后，再读扩展、TUI、provider 细节和 release 流程，会顺很多。
