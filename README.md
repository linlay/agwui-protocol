# AGW UI Interaction Protocol

AGW UI Interaction Protocol 用于定义前端应用如何与多智能体网关（Gateway）交互，包括公开 HTTP API、实时 SSE 事件流、共享数据模型，以及典型接入用例。

这个仓库现在是一个纯文档仓库。内容已经从单页 HTML 参考稿拆成多个 Markdown 模块，方便按主题阅读、维护和后续扩展。

## 阅读顺序

1. [协议概览](docs/overview.md)
2. [架构与交互图](docs/architecture.md)
3. [共享数据模型](docs/data-models.md)
4. [HTTP API](docs/http-api.md)
5. [SSE 事件模型](docs/sse-events.md)
6. [交互时序图](docs/interaction-sequences.md)
7. [接入用例](docs/use-cases.md)
8. [资源导航](docs/resources.md)

## 文档导航

| 文档 | 说明 |
| --- | --- |
| [docs/overview.md](docs/overview.md) | 协议定位、分层边界、术语和阅读指引 |
| [docs/architecture.md](docs/architecture.md) | 系统参与方、架构图、交互逻辑图和主流程说明 |
| [docs/data-models.md](docs/data-models.md) | `Reference`、各类 ID、引用标记和通用约定 |
| [docs/http-api.md](docs/http-api.md) | 全部 11 个公开 HTTP 接口 |
| [docs/sse-events.md](docs/sse-events.md) | SSE 传输约定与全部事件家族 |
| [docs/interaction-sequences.md](docs/interaction-sequences.md) | 按交互模式拆分的 live 协议 SVG 时序图 |
| [docs/use-cases.md](docs/use-cases.md) | 9 个典型请求与事件流闭环示例 |
| [docs/resources.md](docs/resources.md) | SDK、前端测试项目、录屏入口占位页 |

## 两张图先看

### 架构图

![AGW Architecture](assets/diagrams/agw-architecture.svg)

### 交互逻辑图

![AGW Interaction Overview](assets/diagrams/agw-interaction-overview.svg)

## 协议边界

- 所有公开请求都以 `/api` 为前缀。
- `/api/query` 直接返回 `text/event-stream`，不是普通 JSON 响应。
- `request.*`、`run.*`、`task.*`、`content.*` 等名称属于 SSE 事件层，不要求和 HTTP API 形成 1:1 命名映射。
- `POST /api/interrupt` 的结果在流层体现为 `run.cancel`，不会额外产生一个 `request.interrupt` 事件。
- 历史文档里的 `ViewRequest / ViewResponse` 已统一收敛为 `viewport`，即 `GET /api/viewport`。

## 仓库结构

```text
.
├── README.md
├── docs/
│   ├── overview.md
│   ├── architecture.md
│   ├── data-models.md
│   ├── http-api.md
│   ├── sse-events.md
│   ├── interaction-sequences.md
│   ├── use-cases.md
│   └── resources.md
└── assets/
    └── diagrams/
        ├── agw-architecture.svg
        ├── agw-interaction-flow.svg
        ├── agw-interaction-overview.svg
        ├── agw-seq-basic-query.svg
        ├── agw-seq-steer.svg
        ├── agw-seq-interrupt.svg
        ├── agw-seq-hitl-approval.svg
        ├── agw-seq-hitl-question.svg
        ├── agw-seq-hitl-bash.svg
        └── agw-seq-artifact-publish.svg
```
