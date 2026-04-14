# HTTP API

## 1. 总体约定

- 所有接口都以 `/api` 为前缀
- 动作类接口通常使用 `POST`
- 资源类接口通常使用 `GET`
- 除 `POST /api/query` 与 `GET /api/resource` 外，当前接口默认返回 JSON 包装：`{ "code": number, "msg": string, "data": ... }`
- 当前 `/api` 下唯一返回 SSE 的接口是 `POST /api/query`

## 2. 动作类接口

### 2.1 `POST /api/query`

发起一次对话请求。当前实现固定返回 `text/event-stream`。

#### Request

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `requestId` | `string` | 可选；缺省时服务端自动生成 |
| `runId` | `string` | 可选；缺省时服务端自动生成 |
| `chatId` | `string` | 可选；缺省时服务端创建 chat |
| `agentKey` | `string` | 可选；缺省时回退默认 agent |
| `teamId` | `string` | 可选 |
| `role` | `string` | 可选 |
| `message` | `string` | 必填 |
| `references` | `Reference[]` | 可选 |
| `params` | `object` | 可选 |
| `scene` | `object` | 可选 |
| `stream` | `boolean` | 可选，当前为兼容字段 |
| `hidden` | `boolean` | 可选 |

说明：当前 HTTP 请求体会接收上述字段，但 live SSE 回看事件 `request.query` 只保证包含当前实现实际写入流中的字段，不应把 HTTP request body 视为同形响应对象。

#### Response

- `Content-Type: text/event-stream`
- 业务事件统一使用 `event: message`
- `data` 为单行 JSON，业务事件至少包含 `seq`、`type`、`timestamp`
- 心跳保活使用 SSE 注释帧：`: heartbeat`
- 结束时发送：

```text
event: message
data: [DONE]
```

#### 关键说明

- `POST /api/query` 不返回独立 JSON body
- `request.query` 是 SSE 流中的业务事件，不是 HTTP 响应对象
- `reasoning.snapshot`、`content.snapshot`、`tool.snapshot`、`action.snapshot` 不走 live SSE，只进入历史事件持久化

### 2.2 `POST /api/upload`

浏览器本地文件一步上传。上传完成后，前端可把返回的 `data.upload` 映射为 `Reference` 参与后续 `query`。

#### Request

`multipart/form-data`，当前实现读取：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `requestId` | `string` | 可选；缺省时服务端生成 |
| `chatId` | `string` | 可选；缺省时服务端生成 |
| `name` | `string` | 可选；用于新建 chat 时的名称 |
| `file` | `multipart file` | 必填 |

#### Response

返回统一 JSON 包装，`data` 结构为：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `requestId` | `string` | 本次上传请求 ID |
| `chatId` | `string` | 文件所属 chat |
| `upload` | `object` | 上传票据 |

`data.upload` 当前包含：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | `string` | 当前固定为 chat 内资源短 ID |
| `type` | `string` | 当前实现固定返回 `file` |
| `name` | `string` | 文件名 |
| `mimeType` | `string` | 来自 multipart header，可为空 |
| `sizeBytes` | `number` | 文件大小 |
| `url` | `string` | 后续资源地址 |
| `sha256` | `string` | 文件摘要 |

### 2.3 `POST /api/submit`

提交前端工具交互结果。HTTP 请求体不是流内 `request.submit` 事件的镜像对象。

#### Request

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `runId` | `string` | 必填；当前运行 ID |
| `awaitingId` | `string` | 必填；当前交互关联的 awaiting/tool ID |
| `params` | `any` | 必填；表单值、按钮结果、选择结果等 |

#### Response

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `accepted` | `boolean` | 是否接受 |
| `status` | `string` | 接收状态 |
| `runId` | `string` | 关联 run |
| `awaitingId` | `string` | 关联 awaiting |
| `detail` | `string` | 说明文字 |

#### 关键说明

- HTTP 接口同步返回 ack
- 实际处理结果继续通过原 SSE 流返回，常见表现是 `request.submit`、`tool.result` 或后续 `content.*`
- 当前 live SSE 中 `request.submit` 事件 payload 使用的是 `toolId` 字段名；HTTP 接口字段名是 `awaitingId`

### 2.4 `POST /api/steer`

向进行中的 run 注入新的用户指令。流里对应的回看事件是 `request.steer`。

#### Request

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `requestId` | `string` | 可选 |
| `chatId` | `string` | 可选 |
| `runId` | `string` | 必填 |
| `steerId` | `string` | 可选 |
| `agentKey` | `string` | 可选 |
| `teamId` | `string` | 可选 |
| `message` | `string` | 必填 |
| `planningMode` | `boolean` | 可选 |

#### Response

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `accepted` | `boolean` | 是否接受 |
| `status` | `string` | 接收状态 |
| `runId` | `string` | 当前 run |
| `steerId` | `string` | 当前 steer |
| `detail` | `string` | 说明文字 |

### 2.5 `POST /api/interrupt`

中断进行中的 run。流层不会出现独立的 `request.interrupt` 事件，最终效果由 `run.cancel` 表达。

#### Request

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `requestId` | `string` | 可选 |
| `chatId` | `string` | 可选 |
| `runId` | `string` | 必填 |
| `agentKey` | `string` | 可选 |
| `teamId` | `string` | 可选 |
| `message` | `string` | 可选 |
| `planningMode` | `boolean` | 可选 |

#### Response

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `accepted` | `boolean` | 是否接受 |
| `status` | `string` | 接收状态 |
| `runId` | `string` | 当前 run |
| `detail` | `string` | 说明文字 |

## 3. 资源类接口

### 3.1 `GET /api/viewport`

获取 viewport payload。

#### Request

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `viewportKey` | `string` | 必填 |

#### Response

返回统一 JSON 包装，`data` 为 viewport payload 本体。当前文档不对 payload 内部结构做额外约束。

### 3.2 `GET /api/resource`

获取资源内容。成功时直接返回文件流，而不是 JSON 包装。

#### Request

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `file` | `string` | 必填；资源相对路径 |
| `t` | `string` | 可选；resource ticket |

#### Response

- 成功时直接返回资源内容
- 失败时返回统一 JSON 错误包装
- 当启用 resource ticket 且请求未带登录主体时，可能要求 `t`

### 3.3 `GET /api/agents`

获取智能体列表。

#### Request

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `tag` | `string` | 可选；按标签过滤 |

#### Response

返回统一 JSON 包装，`data` 为 agent 列表。当前列表元素至少包含：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `key` | `string` | Agent 唯一键 |
| `name` | `string` | 展示名称 |
| `icon` | `any` | 可选 |
| `description` | `string` | 可选 |
| `role` | `string` | 可选 |
| `meta` | `object` | 可选 |

### 3.4 `GET /api/agent`

获取单个智能体详情。

#### Request

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `agentKey` | `string` | 必填 |

#### Response

返回统一 JSON 包装，`data` 当前包含：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `key` | `string` | Agent 唯一键 |
| `name` | `string` | 展示名称 |
| `icon` | `any` | 可选 |
| `description` | `string` | 可选 |
| `role` | `string` | 可选 |
| `model` | `string` | 模型标识 |
| `mode` | `string` | 运行模式 |
| `tools` | `string[]` | 工具列表 |
| `skills` | `string[]` | 技能列表 |
| `controls` | `object[]` | 控制项 |
| `meta` | `object` | 元数据 |

### 3.5 `GET /api/chats`

获取聊天列表。

#### Request

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `lastRunId` | `string` | 可选 |
| `agentKey` | `string` | 可选 |

#### Response

返回统一 JSON 包装，`data` 为 chat 列表。每个元素当前包含：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `chatId` | `string` | 会话 ID |
| `chatName` | `string` | 会话名称 |
| `agentKey` | `string` | 可选 |
| `teamId` | `string` | 可选 |
| `createdAt` | `number` | 创建时间 |
| `updatedAt` | `number` | 更新时间 |
| `lastRunId` | `string` | 可选 |
| `lastRunContent` | `string` | 可选 |
| `readStatus` | `number` | 已读状态 |
| `readAt` | `number` | 可选 |

### 3.6 `GET /api/chat`

获取单个聊天详情。

#### Request

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `chatId` | `string` | 必填 |
| `includeRawMessages` | `boolean` | 可选；为 `true` 时返回 `rawMessages` |

#### Response

返回统一 JSON 包装，`data` 当前包含：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `chatId` | `string` | 会话 ID |
| `chatName` | `string` | 会话名称 |
| `chatImageToken` | `string` | 可选 |
| `events` | `EventData[]` | 历史事件 |
| `rawMessages` | `object[]` | 可选 |
| `plan` | `any` | 可选 |
| `artifact` | `any` | 可选 |

说明：`events` 是历史事件集合，可能包含 snapshot 事件；这与 `/api/query` 的 live SSE 事件集合不同。
