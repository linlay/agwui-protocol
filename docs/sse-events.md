# SSE 事件模型

## 1. 传输约定

### 业务事件

- 使用 `event: message`
- `data` 是一行 JSON
- 实时业务事件统一包含 `seq`、`type`、`timestamp`

示例：

```text
event: message
data: {"seq":3,"type":"run.start","timestamp":1707000002,"runId":"run_001","chatId":"chat_123","agentKey":"auto"}
```

### 传输层保活

```text
event: heartbeat
```

`heartbeat` 没有 JSON `data`，仅用于传输保活。

### 流结束标志

```text
data:[DONE]
```

这不是业务事件，也不会进入历史 `events`。

## 2. Base Event

所有事件共享的基础字段：

| 字段 | 说明 |
| --- | --- |
| `seq` | 实时事件序号 |
| `type` | 事件类型 |
| `timestamp` | 事件创建时间戳 |
| `rawEvent` | 调试用途原始事件对象，可选 |

## 3. Request 回看事件

这些事件会真实出现在 SSE 流里，用于记录请求层动作。

### `request.query`

对应 `POST /api/query`。

常见字段：

- `requestId`
- `chatId`
- `agentKey`
- `teamId`
- `role`
- `message`
- `references`
- `params`
- `scene`
- `stream`
- `hidden`

### `request.upload`

对应 `POST /api/upload` 的内部回看对象。

常见字段：

- `requestId`
- `chatId`
- `upload.type`
- `upload.name`
- `upload.sizeBytes`
- `upload.mimeType`
- `upload.sha256`

### `request.submit`

对应 `POST /api/submit` 的回看事件。

常见字段：

- `requestId`
- `chatId`
- `runId`
- `toolId`
- `payload`
- `viewId`

### `request.steer`

对应 `POST /api/steer` 的回看事件。

常见字段：

- `requestId`
- `chatId`
- `runId`
- `steerId`
- `message`
- `role`，固定为 `user`

### 不存在的事件

- 没有独立的 `request.interrupt`
- `POST /api/interrupt` 的结果由 `run.cancel` 表达

## 4. 会话与计划事件

### `chat.start`

表示聊天会话开始。

| 字段 | 说明 |
| --- | --- |
| `chatId` | 会话 ID |
| `chatName` | 会话名称 |

当前实时流只发开始事件，不额外发独立的会话名称更新事件。

### `plan.update`

表示 chat 级计划状态更新。

| 字段 | 说明 |
| --- | --- |
| `planId` | 计划 ID |
| `chatId` | 关联 chat |
| `plan` | 任务列表 |

当前实时流统一使用 `plan.update` 表达首次创建与后续修改。

## 5. Run 事件

### `run.start`

| 字段 | 说明 |
| --- | --- |
| `runId` | 运行实例 ID |
| `chatId` | 所属 chat |
| `agentKey` | 运行使用的智能体 |

### `run.complete`

| 字段 | 说明 |
| --- | --- |
| `runId` | 已完成 run |
| `finishReason` | 完成原因，常见为 `end_turn` |

### `run.cancel`

表示 run 被取消或中断。

| 字段 | 说明 |
| --- | --- |
| `runId` | 被取消的 run |

### `run.error`

| 字段 | 说明 |
| --- | --- |
| `runId` | 出错 run |
| `error` | 错误信息 |

## 6. Task 事件

### `task.start`

| 字段 | 说明 |
| --- | --- |
| `taskId` | 任务 ID |
| `runId` | 所属 run |
| `taskName` | 任务名称 |
| `description` | 任务描述 |

### `task.complete`

仅表示任务完成，核心字段为 `taskId`。

### `task.cancel`

仅表示任务取消，核心字段为 `taskId`。

### `task.fail`

| 字段 | 说明 |
| --- | --- |
| `taskId` | 任务 ID |
| `error` | 错误信息 |

## 7. Reasoning 事件

用于承载推理过程。

### 实时事件

- `reasoning.start`
- `reasoning.delta`
- `reasoning.end`

### 历史快照

- `reasoning.snapshot`

### 常见字段

| 字段 | 说明 |
| --- | --- |
| `reasoningId` | 推理段标识 |
| `runId` | 所属 run |
| `taskId` | 可选，所属 task |
| `delta` | 增量文本，仅 `delta` 事件出现 |
| `text` | 完整文本，仅 `snapshot` 出现 |

`reasoning.snapshot` 主要用于历史记录，不是当前 `/api/query` 实时流主路径。

## 8. Content 事件

用于承载对用户可见的正文输出。

### 实时事件

- `content.start`
- `content.delta`
- `content.end`

### 历史快照

- `content.snapshot`

### 常见字段

| 字段 | 说明 |
| --- | --- |
| `contentId` | 正文段标识 |
| `runId` | 所属 run |
| `taskId` | 可选，所属 task |
| `delta` | 增量文本，仅 `delta` 事件出现 |
| `text` | 完整文本，仅 `snapshot` 出现 |

## 9. Tool 事件

用于表示工具渲染、参数增量和工具结果。

### 实时事件

- `tool.start`
- `tool.args`
- `tool.end`
- `tool.result`

### 历史快照

- `tool.snapshot`

### `tool.start` 常见字段

| 字段 | 说明 |
| --- | --- |
| `toolId` | 工具调用 ID |
| `runId` | 所属 run |
| `taskId` | 可选 |
| `toolName` | 工具名称 |
| `toolType` | 可选，前端工具常见为 `html/qlc/dqlc` |
| `toolLabel` | 可选，展示名 |
| `toolDescription` | 可选 |
| `viewportKey` | 前端工具相关视图键 |
| `toolTimeout` | 前端工具超时毫秒数 |

### `tool.args`

| 字段 | 说明 |
| --- | --- |
| `toolId` | 工具调用 ID |
| `delta` | 参数增量文本 |
| `chunkIndex` | 同一 `toolId` 下的分片序号 |

### `tool.result`

| 字段 | 说明 |
| --- | --- |
| `toolId` | 工具调用 ID |
| `result` | 工具运行结果 |

### 关键说明

- 前端交互提交走 `POST /api/submit`
- 运行流中会记录为 `request.submit`
- 不会因为提交动作额外生成一个独立的“工具参数事件”

## 10. Artifact 事件

### `artifact.publish`

向当前 chat 资产池发布单个产物。

| 字段 | 说明 |
| --- | --- |
| `artifactId` | 产物 ID |
| `chatId` | 所属 chat |
| `runId` | 来源 run |
| `artifact` | 精简产物对象，常见含 `type/name/mimeType/sizeBytes/url/sha256` |

一条 `_artifact_publish_` 工具调用可以发布多个文件，因此常见结果是：

1. 一次 `tool.result`
2. 连续多条 `artifact.publish`

## 11. Action 事件

用于表示动作渲染和动作执行结果。

### 实时事件

- `action.start`
- `action.args`
- `action.end`
- `action.result`

### 历史快照

- `action.snapshot`

### 常见字段

| 字段 | 说明 |
| --- | --- |
| `actionId` | 动作 ID |
| `runId` | 所属 run |
| `taskId` | 可选 |
| `actionName` | 动作名称 |
| `description` | 动作描述 |
| `delta` | 参数增量，仅 `action.args` |
| `arguments` | 完整参数，仅 `snapshot` |
| `result` | 动作结果，仅 `action.result` |

## 12. Source 事件

### `source.snapshot`

当前仍未实现实时 SSE 输出，主要表示来源信息块快照。

常见字段：

| 字段 | 说明 |
| --- | --- |
| `icon` | 来源图标 |
| `title` | 来源标题 |
| `url` | 来源地址 |

## 13. 层级关系速览

```text
chat
├── plan
├── run
│   ├── task
│   │   ├── reasoning
│   │   ├── content
│   │   ├── tool
│   │   └── action
│   └── artifact.publish
└── source(snapshot, optional)
```

## 14. 需要重点记住的边界

- `query` 对应 `request.query`
- `submit` 对应 `request.submit`
- `steer` 对应 `request.steer`
- `interrupt` 对应 `run.cancel`
- `heartbeat` 和 `data:[DONE]` 都是传输层概念，不是业务 JSON 事件
