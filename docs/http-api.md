# HTTP API

## 1. 总体约定

- 所有接口都以 `/api` 为前缀
- 动作类接口建议使用 `POST`
- 资源类接口建议使用 `GET`
- 除 `query` 与 `resource` 外，大多数接口返回 JSON
- `query` 直接返回 `text/event-stream`

## 2. 动作类接口

### 2.1 `POST /api/query`

发起一次对话请求。当前实现固定以 SSE 返回实时事件流。

#### Request

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `requestId` | `string` | 前端生成；不填时可等于 `runId` |
| `chatId` | `string` | 可选；缺省则网关创建 chat |
| `agentKey` | `string` | 可选，如 `auto`、`default` |
| `teamId` | `string` | 可选 |
| `role` | 枚举 | 消息角色 |
| `message` | `string` | 必填 |
| `references` | `Reference[]` | 可选 |
| `params` | `object` | 可选 |
| `scene` | `object` | 可选，建议至少带 `url` |
| `stream` | `boolean` | 保留兼容字段 |
| `hidden` | `boolean` | 可选 |

#### Response

- `Content-Type: text/event-stream`
- 业务事件统一使用 `event: message`
- `data` 是单行 JSON，包含 `seq/type/timestamp`
- 传输保活时会出现 `event: heartbeat`
- 结束时追加 `data:[DONE]`

#### 关键说明

- `query` 不返回独立 JSON body
- `request.query` 是 SSE 流中的回看事件，不是 HTTP 响应对象

### 2.2 `POST /api/upload`

浏览器本地文件一步上传。上传完成后，前端可把返回的 `upload` 映射为 `Reference` 参与后续 `query`。

#### Request

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `requestId` | `string` | 前端生成，用于幂等和重试 |
| `chatId` | `string` | 可选；缺省则网关创建 chat |
| `file` | `multipart file` | 必填 |
| `sha256` | `string` | 可选 |

#### Response

返回统一 JSON 包装，`data.upload` 至少包含：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | `string` | chat 内短 ID，如 `r01` |
| `type` | `"file" \| "image"` | 资源类型 |
| `name` | `string` | 文件名 |
| `mimeType` | `string` | MIME 类型 |
| `sizeBytes` | `number` | 文件大小 |
| `url` | `string` | 后续资源地址 |
| `sha256` | `string` | 可选 |

### 2.3 `POST /api/submit`

提交前端工具交互结果。HTTP 请求体不是流内 `request.submit` 事件的镜像对象。

#### Request

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `runId` | `string` | 当前运行 ID |
| `toolId` | `string` | 必填；当前交互关联的 tool |
| `params` | `any` | 必填；表单值、按钮结果、选择结果等 |

#### Response

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `accepted` | `boolean` | 是否接受 |
| `status` | `string` | 接收状态 |
| `runId` | `string` | 关联 run |
| `toolId` | `string` | 关联 tool |
| `detail` | `string` | 说明文字 |

#### 关键说明

- 提交确认是同步响应
- 实际结果继续通过原 SSE 流返回，常见表现是 `tool.result` 或后续 `content.*`

### 2.4 `POST /api/steer`

向进行中的 run 注入新的用户指令。流里对应的回看事件是 `request.steer`。

#### Request

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `requestId` | `string` | 可选 |
| `chatId` | `string` | 可选 |
| `runId` | `string` | 必填 |
| `steerId` | `string` | 可选；缺省由服务端生成 |
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

获取工具或渲染对应的视图载荷。历史 `ViewRequest / ViewResponse` 已统一为 `viewport`。

#### Request

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `viewportKey` | `string` | 必填 |

#### Response

返回统一 `ApiResponse<Object>`，`data` 即视图 payload：

- 本地 HTML viewport 常包装为 `{ "html": "..." }`
- 其他类型通常直接透传 payload

### 3.2 `GET /api/resource`

获取资源内容，包括附件和图片。成功时直接返回文件流，而不是 JSON 包装。

#### Request

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `file` | `string` | 必填；相对 data 根目录的文件路径 |
| `download` | `boolean` | 可选；`true` 时强制下载 |
| `t` | `string` | 可选；resource ticket |

#### Response

- 成功时返回资源二进制流
- `Content-Type` 由文件名推断
- `Content-Disposition`：图片默认 `inline`，其他默认 `attachment`
- `Cache-Control: private, no-store`

### 3.3 `GET /api/agents`

获取智能体列表。

#### Request

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `includeHidden` | `boolean` | 可选，暂未实现 |
| `tag` | `string` | 可选，按标签过滤 |

#### Response

返回 `agents[]`，每个元素通常包含：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `key` | `string` | 如 `agent_researcher` |
| `name` | `string` | 展示名称 |
| `description` | `string` | 可选 |
| `capabilities` | `string[]` | 可选，暂未实现 |
| `meta` | `object` | 可选，暂未实现 |

### 3.4 `GET /api/agent`

获取单个智能体详情。

#### Request

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `agentKey` | `string` | 必填 |

#### Response

返回 `agent` 对象，常见字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `key` | `string` | Agent 唯一键 |
| `name` | `string` | 展示名称 |
| `description` | `string` | 可选 |
| `instructions` | `string` | 可选 |
| `capabilities` | `string[]` | 可选，暂未实现 |
| `meta` | `object` | 可选，暂未实现 |

### 3.5 `GET /api/chats`

获取聊天列表。

#### Request

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `cursor` | `string` | 可选，分页游标 |
| `limit` | `number` | 可选 |
| `q` | `string` | 可选，按名称或内容检索 |

#### Response

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `items` | `object[]` | 聊天列表 |
| `nextCursor` | `string` | 下一页游标，可选 |

`items[]` 常见字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `chatId` | `string` | 会话 ID |
| `chatName` | `string` | 会话名称 |
| `updatedAt` | `number` | 更新时间戳 |

### 3.6 `GET /api/chat`

获取聊天详情，包括会话内容、历史消息、关联 run 和聚合态。

#### Request

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `chatId` | `string` | 必填 |
| `includeEvents` | `boolean` | 可选，是否返回事件流快照 |
| `cursor` | `string` | 可选 |
| `limit` | `number` | 可选 |

#### Response

`chat` 详情结构可按产品扩展，常见字段如下：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `chatId` | `string` | 会话 ID |
| `chatName` | `string` | 会话名称 |
| `chatImageToken` | `string` | chat 作用域内图片访问 token，可选 |
| `rawMessages` | `object[]` | 原始消息，可选 |
| `plan` | `object` | 当前最新计划状态，可选 |
| `artifact` | `object` | 当前最新产物聚合态，可选 |
| `references` | `object[]` | 当前可用引用池，可选 |
| `events` | `BaseEvent[]` | 历史事件列表 |

#### 历史事件折叠规则

- `start/delta/args/end` 通常会折叠为 `snapshot`
- `plan.update` 不再出现在历史 `events` 中，而是进入 chat 级 `plan`
- `artifact.publish` 不再出现在历史 `events` 中，而是进入 chat 级 `artifact`

## 4. 边界提醒

- `query` 是公开 HTTP 入口，但响应是 SSE 流
- `submit`、`steer`、`interrupt` 都是同步确认 + 异步结果的组合模式
- `request.submit`、`request.steer` 属于流内回看事件，不是 HTTP body 定义
- `interrupt` 的流层结果是 `run.cancel`
