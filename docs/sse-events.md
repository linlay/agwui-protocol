# SSE 事件模型

## 1. 传输约定

### 业务事件帧

- 使用 `event: message`
- `data` 是一行 JSON
- 当前 live SSE 业务事件统一包含 `seq`、`type`、`timestamp`

示例：

```text
event: message
data: {"seq":3,"type":"run.start","runId":"run_001","chatId":"chat_123","agentKey":"default","timestamp":1707000002000}
```

### 心跳帧

```text
: heartbeat
```

心跳是 SSE 注释帧，不是业务 JSON 事件。

### 流结束帧

```text
event: message
data: [DONE]
```

`[DONE]` 是传输层结束哨兵，不是业务事件，也不会进入历史 `events`。

## 2. Base Event

所有 live 业务事件共享的基础字段：

| 字段 | 说明 |
| --- | --- |
| `seq` | 客户端可见的实时事件序号 |
| `type` | 事件类型 |
| `timestamp` | 事件创建时间戳（毫秒） |

## 3. 事件分层

当前实现需要区分三类内容：

- live SSE 事件：会通过 `POST /api/query` 实时发给客户端
- snapshot / persisted 事件：只进入历史持久化，不走 live SSE
- 传输层帧：心跳与 `[DONE]`，不属于业务 JSON 事件

## 4. Live SSE 事件

### 4.1 请求与运行生命周期

#### `request.query`

对应 `POST /api/query` 发起时的回看事件。

常见字段：

- `requestId`
- `runId`
- `chatId`
- `role`
- `message`

说明：当前事件 payload 不应等同于 HTTP request body；只有实现实际写入流中的字段才属于 live SSE 协议。

#### `chat.start`

表示 chat 在本次请求中被新建。

| 字段 | 说明 |
| --- | --- |
| `chatId` | 会话 ID |
| `chatName` | 会话名称 |

#### `run.start`

| 字段 | 说明 |
| --- | --- |
| `runId` | 运行实例 ID |
| `chatId` | 所属 chat |
| `agentKey` | 运行使用的 agent |

#### `run.complete`

| 字段 | 说明 |
| --- | --- |
| `runId` | 已完成 run |
| `finishReason` | 完成原因；取值由当前运行流决定 |

#### `run.cancel`

表示 run 被取消或中断。

| 字段 | 说明 |
| --- | --- |
| `runId` | 被取消的 run |

#### `run.error`

| 字段 | 说明 |
| --- | --- |
| `runId` | 出错 run |
| `error` | 规范化错误对象 |

#### `request.submit`

对应 `POST /api/submit` 在流中的回看事件。

常见字段：

- `requestId`
- `chatId`
- `runId`
- `toolId`
- `payload`

说明：live SSE 里当前使用 `toolId` 字段名；HTTP 提交接口使用 `awaitingId`。

#### `request.steer`

对应 `POST /api/steer` 的回看事件。

常见字段：

- `requestId`
- `chatId`
- `runId`
- `steerId`
- `message`
- `role`，固定为 `user`

说明：当前不存在独立的 `request.interrupt`；`POST /api/interrupt` 的流内效果由 `run.cancel` 表达。

### 4.2 Task / Plan / Artifact

#### `plan.update`

表示 chat 级计划状态更新。

| 字段 | 说明 |
| --- | --- |
| `planId` | 计划 ID |
| `chatId` | 关联 chat |
| `plan` | 当前计划内容 |

说明：当前实现只实际发出 `plan.update`，不把 `plan.create` 作为现行 live 事件。

#### `task.start`

| 字段 | 说明 |
| --- | --- |
| `taskId` | 任务 ID |
| `runId` | 所属 run |
| `taskName` | 任务名称 |
| `description` | 任务描述 |

#### `task.complete`

仅表示任务完成，核心字段为 `taskId`。

#### `task.cancel`

仅表示任务取消，核心字段为 `taskId`。

#### `task.fail`

| 字段 | 说明 |
| --- | --- |
| `taskId` | 任务 ID |
| `error` | 规范化错误对象 |

#### `artifact.publish`

向当前 chat 资产池发布单个产物。

| 字段 | 说明 |
| --- | --- |
| `artifactId` | 产物 ID |
| `chatId` | 所属 chat |
| `runId` | 来源 run |
| `artifact` | 产物对象 |

### 4.3 Reasoning

用于承载推理过程的实时增量。

- `reasoning.start`
- `reasoning.delta`
- `reasoning.end`

常见字段：

| 字段 | 说明 |
| --- | --- |
| `reasoningId` | 推理段标识 |
| `runId` | 所属 run，仅 `start` 保证出现 |
| `taskId` | 可选，所属 task |
| `reasoningLabel` | 可选，推理标签 |
| `delta` | 增量文本，仅 `reasoning.delta` |

### 4.4 Content

用于承载对用户可见的正文输出。

- `content.start`
- `content.delta`
- `content.end`

常见字段：

| 字段 | 说明 |
| --- | --- |
| `contentId` | 正文段标识 |
| `runId` | 所属 run，仅 `start` 保证出现 |
| `taskId` | 可选，所属 task |
| `delta` | 增量文本，仅 `content.delta` |

### 4.5 Tool

用于表示工具调用过程。

- `tool.start`
- `tool.args`
- `tool.end`
- `tool.result`

#### `tool.start` 常见字段

| 字段 | 说明 |
| --- | --- |
| `toolId` | 工具调用 ID |
| `runId` | 所属 run |
| `taskId` | 可选 |
| `toolName` | 工具名称 |
| `toolLabel` | 可选，展示名 |
| `toolDescription` | 可选 |

#### `tool.args`

| 字段 | 说明 |
| --- | --- |
| `toolId` | 工具调用 ID |
| `delta` | 参数增量文本 |
| `chunkIndex` | 同一 `toolId` 下的分片序号 |

#### `tool.result`

| 字段 | 说明 |
| --- | --- |
| `toolId` | 工具调用 ID |
| `result` | 工具结果 |

关键说明：

- 当前 `tool.start` 不应写入未由实现提供的 `toolType`、`viewportKey`、`toolTimeout`
- 某些前端交互会在 `tool.start` 后看到 `awaiting.ask` / `awaiting.payload`
- `clientVisible=false` 的工具，其 `tool.*` 事件可能被 normalizer 抑制，不会出现在客户端 live SSE

### 4.6 Awaiting / 前端交互

#### `awaiting.ask`

告诉前端当前 run 进入等待用户交互状态。

常见字段：

- `awaitingId`
- `viewportType`
- `viewportKey`
- `mode`
- `toolTimeout`
- `runId`
- `questions`（仅部分模式出现）

#### `awaiting.payload`

向前端提供待回答问题列表。

| 字段 | 说明 |
| --- | --- |
| `awaitingId` | 当前 awaiting ID |
| `questions` | 问题列表 |

关键说明：

- `clientVisible=false` 的工具，其 `awaiting.ask` / `awaiting.payload` 也可能被抑制
- `POST /api/submit` 对应的流内回看事件是 `request.submit`，不是额外的 awaiting 专用提交事件

### 4.7 Action

用于表示 action 调用过程。

- `action.start`
- `action.args`
- `action.end`
- `action.result`

常见字段：

| 字段 | 说明 |
| --- | --- |
| `actionId` | 动作 ID |
| `runId` | 所属 run，仅 `start` 保证出现 |
| `taskId` | 可选 |
| `actionName` | 动作名称 |
| `description` | 动作描述 |
| `delta` | 参数增量，仅 `action.args` |
| `result` | 动作结果，仅 `action.result` |

## 5. Snapshot / Persisted 事件

下列事件会进入历史持久化，但不会通过 `/api/query` live SSE 发给客户端：

- `reasoning.snapshot`
- `content.snapshot`
- `tool.snapshot`
- `action.snapshot`

这些事件通常出现在 `GET /api/chat` 的 `events` 中，用于保存完整文本或完整参数。

## 6. 当前不存在的现行事件

以下名称不应作为当前 live SSE 协议的一部分：

- `request.upload`
- `source.snapshot`
- `plan.create`

## 7. 层级关系速览

```text
chat
├── plan.update
├── run
│   ├── task
│   │   ├── reasoning (live) / reasoning.snapshot (history)
│   │   ├── content (live) / content.snapshot (history)
│   │   ├── tool (live) / tool.snapshot (history)
│   │   └── action (live) / action.snapshot (history)
│   └── artifact.publish
└── request.* / awaiting.*
```

## 8. 需要重点记住的边界

- 当前唯一 SSE endpoint 是 `POST /api/query`
- `submit` 对应 `request.submit`
- `steer` 对应 `request.steer`
- `interrupt` 对应 `run.cancel`
- `: heartbeat` 和 `[DONE]` 都是传输层概念，不是业务 JSON 事件
- snapshot 事件属于历史持久化，不属于 live SSE
