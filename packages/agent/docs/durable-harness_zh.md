# 持久化 AgentHarness 与会话设计

<!-- 从 jot zmnps2zu 同步而来。后续请直接在仓库中编辑本文件。 -->

持久化 AgentHarness / 会话设计说明。

## 总体定位

单靠 AgentHarness 无法实现完全持久化，因为工具实现、模型和认证提供商、扩展与 hook、资源加载器以及系统提示词回调等关键依赖由宿主应用以运行时 JavaScript 提供。

实际目标是半持久化 harness：会话是持久化的追加式状态树；harness 将自己拥有的状态写入会话条目；宿主应用负责恢复兼容的不可持久化依赖；恢复从持久化边界重新开始，而不是从正在进行的提供商流中继续。

## 会话拥有持久化状态

应将会话视为所有持久化的代理状态，而不只是对话记录。现有状态已经包含模型、思考级别、活动工具、叶节点、标签、压缩、分支摘要、自定义消息和自定义条目。因此应继续使用一个持久化会话日志，而不是增加 harness 旁路存储。大型数据可以使用旁路存储，但会话条目仍应保存事实来源引用。

## 恢复时应用必须提供的内容

应用必须重新创建兼容的模型注册表、工具注册表、扩展集合及其版本和顺序、资源加载器、系统提示词提供器/hook、认证提供器及应用专用 hook。Harness 可以在可用时验证稳定 ID、版本或 hash，但不能自行序列化这些依赖。

## 运行时配置与恢复

构造函数选项仍表示显式运行时配置，不读取会话状态。未来的异步 builder/factory 应负责持久化恢复：

```ts
const harness = await AgentHarness.builder()
  .env(env)
  .session(session)
  .model(defaultModel)
  .tools(runtimeTools)
  .defaultActiveTools(["read", "edit"])
  .restore({ missingActiveTools: "fail" });
```

`restore()` 应读取活动分支、归约持久化 harness 配置、为缺失条目应用默认值、根据应用提供的运行时依赖进行验证、构造 harness，并可在构造后发出 `source: "restore"` 更新事件。

活动工具名称必须唯一，工具注册表名称必须唯一。默认情况下，恢复时缺失的活动工具应导致失败；宽松的丢弃/禁用策略应显式添加。具体工具永远不会从会话恢复，宿主应用必须提供兼容工具。

## Harness 应持久化的内容

至少应持久化：分支范围的活动工具名称、排队的 steer/followUp/nextTurn 消息、队列消费与 turn 的关联、活动操作期间接受的待处理会话写入、写入应用状态、操作开始/结束/中断、turn 开始/结束，以及必要时的提供商请求和工具调用开始/结束。

```ts
type DurableHarnessEntry =
  | QueueEnqueuedEntry
  | QueueConsumedEntry
  | PendingWriteEnqueuedEntry
  | PendingWriteAppliedEntry
  | OperationStartedEntry
  | OperationFinishedEntry
  | OperationInterruptedEntry
  | TurnStartedEntry
  | TurnFinishedEntry
  | ProviderRequestStartedEntry
  | ProviderRequestFinishedEntry
  | ToolCallStartedEntry
  | ToolCallFinishedEntry;
```

每个已接受的变更都必须在公共 API 返回前持久化。

## 恢复模型

启动时依次执行：宿主应用注册工具、模型、扩展、资源、认证和 hook；harness 打开会话；将条目归约为当前叶节点、对话分支、harness 配置、队列、待处理写入以及活动操作/turn/工具状态；验证运行时依赖；协调未完成的操作。提供商流不可恢复，只能从持久化边界重试，或将操作标记为中断。

默认的保守策略是：未完成的代理 turn 标记为中断并保持队列/待处理写入；未完成的提供商请求标记为中断且不自动重试；未完成的工具调用追加中断或错误结果，只有声明可安全重试/幂等时才重试；没有最终条目时才重新运行未完成的压缩或分支摘要操作。

```ts
recovery: "mark_interrupted" | "retry_unfinished"
```

`retry_unfinished` 必须针对非幂等工具调用设置保护。

## 关键场景

### 队列

在 `queue_enqueued` 前崩溃表示消息未被接受；之后崩溃表示消息应恢复。队列排空后、持久化 turn 记录前崩溃可能造成丢失或重复。因此，已消费的队列 ID 必须记录在 `turn_started` 或等价条目中，之后才能视为已消费。

### 待处理写入

入队前崩溃表示写入未被接受；入队后、应用前崩溃时恢复应应用它；应用后、标记前崩溃时，可通过确定性的目标条目 ID 检测条目已存在并标记为已应用。

### Agent loop turn

提供商请求前崩溃可重试或中断；请求期间默认中断；提供商返回后、助手消息持久化前崩溃会丢失响应，除非响应已写入日志；助手消息持久化后可从该消息恢复。

### 工具调用

工具开始后崩溃时，外部副作用可能已经发生。默认不应重新运行非幂等工具。工具调用需要稳定 ID 和重试安全元数据。

### 压缩与树导航

在摘要或最终条目写入前崩溃时，可安全地重新运行准备/摘要；最终条目已存在时操作完成，只需在缺失时补充结束标记。树导航同理：摘要条目存在但叶条目缺失时补写叶条目。

## 最小可行验证

1. 添加持久化队列条目。
2. 添加带确定性目标 ID 的待处理写入条目。
3. 添加操作开始/结束/中断条目。
4. 在 turn 开始条目中记录已消费队列 ID。
5. 通过归约会话日志实现恢复。
6. 默认将未完成的代理 turn 标记为中断。
7. 只有不存在最终条目时才重新运行未完成的压缩/树操作。
8. 除非工具元数据声明可安全重试，否则不重试未完成的工具调用。

## 待解决问题

哪些 harness 配置应优先进入会话（资源、流选项、系统提示词引用）？是否应为审计/调试按 turn 保存已解析的系统提示词？恢复时是否要求严格匹配依赖 ID/版本？需要记录多少提供商请求数据？中断是否应追加用户可见的助手消息？恢复时存储是否应支持截断最后一行不完整的 JSONL？
