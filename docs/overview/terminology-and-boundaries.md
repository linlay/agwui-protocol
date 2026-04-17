# 术语与边界

## 1. 三层理解

阅读 AGW 时，建议把协议拆成三层：

### 公开请求语义层

- `/api/query`
- `/api/upload`
- `/api/submit`
- `/api/steer`
- `/api/interrupt`
- `/api/viewport`
- `/api/resource`
- `/api/agents`
- `/api/agent`
- `/api/chats`
- `/api/chat`

这一层描述“前端可以发什么业务动作”。

### 实时事件语义层

- `request.*`
- `chat.*`
- `plan.*`
- `run.*`
- `task.*`
- `reasoning.*`
- `content.*`
- `tool.*`
- `artifact.*`
- `action.*`

这一层描述“运行过程中会发生什么事件”。

### 传输承载层

- HTTP JSON
- SSE
- WebSocket

这一层描述“这些请求和事件是如何在线上传输的”。

## 2. 关键映射关系

这几层不是 1:1 对应关系：

- `POST /api/query` 会在流里出现 `request.query`
- `POST /api/submit` 会在流里出现 `request.submit`
- `POST /api/steer` 会在流里出现 `request.steer`
- `POST /api/interrupt` 不会出现 `request.interrupt`，最终效果由 `run.cancel` 表达

## 3. 三条关键边界

### `query` 的 HTTP 形态直接返回 SSE

- `POST /api/query` 当前固定返回 `text/event-stream`
- 不会返回独立 JSON body
- 业务事件统一走 `event: message`
- 每条业务事件的 `data` 都是一行 JSON

### `steer` 是公开请求，流里体现为 `request.steer`

- `POST /api/steer` 是公开 HTTP API
- 其同步确认来自 HTTP 响应
- 其运行回看事件在原 live 流中体现为 `request.steer`

### `interrupt` 在流里体现为 `run.cancel`

- `POST /api/interrupt` 是公开 HTTP API
- 流层不会产生独立的 `request.interrupt`
- 最终停止结果由 `run.cancel` 表达

## 4. 典型执行生命周期

一个最小闭环通常长这样：

1. 前端调用 `POST /api/query`
2. 服务端直接返回 `text/event-stream`
3. 流里出现 `request.query`
4. 如果 chat 不存在，出现 `chat.start`
5. 出现 `run.start`
6. 运行过程中持续输出 `reasoning.*`、`content.*`、`tool.*`、`action.*`
7. 正常结束时出现 `run.complete`
8. 流结束时追加 `[DONE]`

如果运行中有人工干预：

- 追加指令用 `POST /api/steer`
- 提交前端工具交互结果用 `POST /api/submit`
- 强制中断用 `POST /api/interrupt`

## 5. 共享核心对象

- `Reference`：统一表示文件、图片、工作区选区、截图等引用对象
- `requestId`：用于幂等、重试和链路追踪
- `chatId`：聊天会话标识
- `runId`：一次执行过程的标识
- `taskId`：某次 run 中的任务标识
- `viewportKey`：前端视图载荷的检索键

## 6. 关键事实

- `POST /api/query` 当前固定返回 SSE，不返回独立 JSON body
- 实时业务事件统一包含 `seq`、`type`、`timestamp`
- SSE 传输层可能出现心跳注释帧
- 结束标志是 `[DONE]`，它不是业务事件，也不会进入历史 `events`
- `Snapshot` 类事件主要用于历史记录，不是当前 `/api/query` 实时流主路径
- WebSocket 若启用，承载的是同一套业务语义，不是另一套独立协议名词体系
