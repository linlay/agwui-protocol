# 协议定位

## 1. 协议是什么

AGW UI Interaction Protocol 定义的是前端应用如何与智能体平台通信，而不是前端直接与某个具体 Agent 的私有协议。

这里的 Agent Platform 是协议层的统一服务端主体。Gateway 可以是其中一种部署模式，但协议本身不要求中间必须存在 Gateway。

平台通常承担三类职责：

- 对外暴露统一的 `/api/*` HTTP API
- 把请求路由到具体运行实例、单智能体或多智能体协作链路
- 通过统一的实时传输把过程和结果持续回传给前端

推荐理解方式是：前端只对接协议层的 Agent Platform，至于内部是否以 Gateway、Runner 或其他服务编排实现，不影响协议本身。

## 2. 默认阅读主线

如果你第一次阅读这套文档，建议按下面顺序：

1. 本页：理解协议主体与目标
2. [术语与边界](terminology-and-boundaries.md)：理解 HTTP / SSE / WebSocket 的分层
3. [架构与交互图](architecture.md)：理解主流程与参与方
4. [HTTP API](../reference/http-api.md)、[SSE 事件模型](../reference/sse-events.md)、[WebSocket 协议](../reference/websocket-protocol.md)
5. [交互时序图](../guides/interaction-sequences.md) 与 [接入用例](../guides/use-cases.md)

## 3. 传输方式

当前协议的默认讲解主路径仍然是：

- 公开 HTTP API
- `POST /api/query` 返回 live SSE
- 运行中通过 `submit / steer / interrupt` 继续交互

如果服务端启用 WebSocket 扩展，也可以通过 `/ws` 承载同一套业务语义；但这不会改变 HTTP API 和事件语义本身，只是新增一种传输方式。

## 4. 文档分区职责

- `overview/`：讲协议定位、术语边界、架构
- `reference/`：讲协议定义
- `guides/`：讲时序、用例、资源
- `visuals/`：讲辅助理解页面，不属于正式规范正文
