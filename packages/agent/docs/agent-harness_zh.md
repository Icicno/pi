# AgentHarness 生命周期

`AgentHarness` 是位于底层代理循环之上的编排层，负责会话持久化、运行时配置、资源解析、操作锁以及面向扩展的变更语义。

## 生命周期目标

Harness listener 和 hook 应能够捕获 `AgentHarness` 实例，并在文档允许的事件中调用公共 API。这些调用不能损坏正在执行的 turn 快照、打乱持久化 transcript、丢失待处理写入、阻塞结算或使 harness 停留在错误阶段。

规则如下：结构性操作在忙碌时拒绝；队列操作在规定的 turn 安全点接受；运行时配置 setter 更新未来快照而不修改当前提供商请求；忙碌期间的会话写入以确定顺序持久化并刷新；getter 返回最新 harness 配置而不是进行中的快照。当前 listener/hook 不接收 facade；如果它们捕获原始 harness 并在运行中调用 `waitForIdle()`，可能死锁，未来 facade 应提供 `runWhenIdle()`。

`AssistantMessageStream` 已将 SSE/WebSocket 等提供商传输读取与下游事件消费解耦。因此 harness 可以等待 listener、扩展 hook、持久化和保存点工作，而不会阻塞传输读取器。生命周期代码应在 harness 边界使用显式的 await 顺序，而不是 fire-and-forget。

## 错误处理

底层能力和辅助函数在预期失败可被包含且不应抛出时使用 `Result<TValue, TError>`，例如 `ExecutionEnv`、文件系统/ shell 操作、输出捕获、资源加载和压缩辅助函数。`Session` 和 `AgentHarness` 等高层变更/编排 API 应拒绝或抛出，不返回可能被忽略的裸结果。公共 harness 失败在可行时统一为 `AgentHarnessError`，子系统错误保留为 `cause`。

Harness 事件观察的是已提交状态。公共变更器在提交前校验输入并尽量完成持久化，再等待通知。若提交后 hook 或订阅者失败，状态不会回滚，公共方法以 `AgentHarnessError` 的 `"hook"` code 拒绝。

## 状态模型

### Harness 配置

最新运行时配置包括 model、thinking level、tools、活动工具名称、resources、stream options 以及 system prompt/provider。getter 返回 harness 配置，不返回进行中请求使用的快照。setter 即使在 turn 中也立即更新配置；变更影响下一个 turn 快照，不影响当前请求。`setResources()` 每次调用都发出 `resources_update`，并对当前/旧资源做浅复制；`getResources()` 返回当前资源的浅复制。

### Turn 快照

turn 快照由 `createTurnState()` 创建，包含持久化会话消息、解析后的资源和系统提示词、model、thinking level、全部工具、活动工具、流选项和派生 session id。静态值直接使用；系统提示词 provider 每次创建快照只调用一次。资源数组和流选项会浅复制，`headers`/`metadata` map 也只浅复制。`getApiKeyAndHeaders()` 按提供商请求解析凭据，以便刷新过期 token。

### Session 与待处理写入

Session 只包含已持久化条目；读取不包含排队写入。`buildContextEntries()` 返回考虑压缩的上下文条目；`buildContext()` 从完整活动分支派生运行时状态，再投影为 `AgentMessage[]`。自定义条目默认不进入模型上下文，可通过 `entryProjectors` 和 `entryTransforms` 控制。

Session 存储必须把叶节点变化持久化为 `leaf` 条目。`setLeafId()` 不是仅更新内存游标，而是追加带 `targetId` 的持久条目；重新打开存储时必须从最新影响叶节点的条目恢复当前叶节点。

操作活动期间的会话写入会成为待处理写入，使用不含 `id`、`parentId`、`timestamp` 的条目形状。它们必须持久化，并在保存点、操作结算和失败清理时刷新。公共待处理写入/session facade API 尚未实现。

## 操作阶段

```ts
type AgentHarnessPhase = "idle" | "turn" | "compaction" | "branch_summary" | "retry";
```

结构性操作要求 `phase === "idle"`，并在第一个 `await` 前同步设置阶段：`prompt`、`skill`、`promptFromTemplate`、`compact`、`navigateTree`。非 idle 时再次启动结构性操作会以 `"busy"` 拒绝。turn 中可用的操作包括 `steer`、`followUp`、`nextTurn`、`abort` 和运行时配置 setter。阶段/结算语义仍需完整生命周期审查。

## Turn 执行与队列

`prompt`、`skill`、`promptFromTemplate` 都会断言 idle、设置 `turn`、创建快照、从快照派生调用文本，再调用 `executeTurn()`；skill 和模板从同一快照解析资源。`steer`、`followUp`、`nextTurn` 接受文本和可选图片并内部创建用户消息，`nextTurn` 消息会插入下一次用户 turn 的新消息之前。

队列模式是实时的而不是快照化的：`getSteeringMode()`/`setSteeringMode()` 和 `getFollowUpMode()`/`setFollowUpMode()`。运行时变更影响下一次排空，队列只在安全点排空。

## 保存点、hook 与事件

保存点发生在助手 turn 及其工具结果消息完成后。Harness 先在本 turn 的 agent 消息之后刷新待处理写入；若底层循环可能继续，则创建新快照，并在下一次提供商请求前应用新的 context、model、thinking level、stream options 和 session id。这样 turn 中的配置变更可影响同一次运行的下一 turn，但不会修改进行中的请求。

低层循环在提供商边界将 `ThinkingLevel` 转为 reasoning：`"off"` 为 `undefined`，其他值原样传递。`agent_end` 不需刷新状态，只需刷新剩余写入并清除操作阶段；`settled` 的确切时序仍在审查。

目标 hook 系统见 [hooks.md](./hooks.md)。`AgentHarness` 发出类型化 hook 事件并消费类型化结果；单一 hooks 实现负责注册、清理、来源和 reducer。观察型和变更型 hook 使用同一个事件专用 `on()` API；结果事件通过类型化 reducer 表归约。Hook context 应是 facade 的普通对象，而非原始内部对象或迟绑定 getter 集合。

## Session facade、终止与结构性操作

未来扩展应使用 harness 范围的 `HarnessSession` facade。idle 时写入立即持久化，busy 时进入待处理队列；读取只读取已持久化状态。agent 消息在 `message_end` 持久化，保存点在这些消息之后刷新扩展/session 写入。

turn 中允许 abort；它终止低层运行并清除 steer/follow-up 队列，但不清除 `nextTurn` 消息，也不丢弃待处理写入。压缩和树导航是只允许 idle 的结构性会话变更，不排队；下一次 prompt 会创建新快照。AgentHarness 尚未实现自动压缩和 retry 决策点。

## 测试组织

测试按领域拆分：`agent-harness.test.ts` 覆盖核心生命周期和公共 API；`agent-harness-stream.test.ts` 覆盖流选项和提供商 hook。未来可拆分资源、工具和生命周期测试。使用 `pi-ai` faux provider（`registerFauxProvider`、`fauxAssistantMessage`）进行确定性测试，不访问真实提供商或网络。

```bash
npm run test:harness
npm run coverage:harness
```

`coverage:harness` 覆盖 `test/harness/**/*.test.ts`，并将 `src/harness/**/*.ts` 及直接测试到的 `src/agent.ts`、`src/agent-loop.ts` 报告到 `coverage/harness`。

## 实现待办

当前主要待办包括：完善 per-`AgentHarness` model registry；完成 phase、save-point、settled、重入和 abort 语义审查；实现通用 hook/event 扩展机制；设计半持久化恢复；增加全面的 listener/hook 重入测试；以及将 coding-agent 资源、扩展、stream/auth/retry/header 行为迁移到 harness 配置和 provider hook。

已完成的核心工作包括：harness 直接调用 `runAgentLoop()`；拥有运行生命周期、队列排空、provider stream 配置、事件归约、会话持久化和保存点快照；支持流配置、认证合并、provider hook；提供低层类型化 `Result`、文件/ shell 能力拆分、结构化错误、Node 专用实现隔离，以及对应测试。
