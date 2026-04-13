# 接入用例

本页展示 9 个典型场景。每个场景都尽量保留“公开 API 请求 + 对应 SSE 事件流”的最小闭环。

## 1. 基础问答：Query + Content

最小场景，不涉及 Tool、Action 或多任务。

```json
// POST /api/query
{
  "requestId": "req_001",
  "chatId": "chat_123",
  "agentKey": "auto",
  "role": "user",
  "message": "用一句话解释事件流模型是什么？并说明关键事件有哪些。",
  "scene": { "url": "https://app.example.com/chat", "title": "Chat" },
  "stream": true
}
```

```text
event: message
data: {"seq":1,"type":"request.query","timestamp":1707000000,"requestId":"req_001","chatId":"chat_123","agentKey":"auto","role":"user","message":"用一句话解释事件流模型是什么？并说明关键事件有哪些。","scene":{"url":"https://app.example.com/chat","title":"Chat"},"stream":true}

event: message
data: {"seq":2,"type":"chat.start","timestamp":1707000001,"chatId":"chat_123","chatName":"Minimal QA"}

event: message
data: {"seq":3,"type":"run.start","timestamp":1707000002,"runId":"run_001","chatId":"chat_123","agentKey":"auto"}

event: message
data: {"seq":4,"type":"content.start","timestamp":1707000003,"contentId":"ctx_001","runId":"run_001"}

event: message
data: {"seq":5,"type":"content.delta","timestamp":1707000004,"contentId":"ctx_001","delta":"事件流模型：后端把一次请求拆成多个可订阅事件，按序推送给前端。"}

event: message
data: {"seq":6,"type":"content.end","timestamp":1707000007,"contentId":"ctx_001"}

event: message
data: {"seq":7,"type":"run.complete","timestamp":1707000008,"runId":"run_001","finishReason":"end_turn"}

event: message
data: [DONE]
```

## 2. Thinking + Tooling + Content

先推理，再调用工具，再输出正文。

```json
// POST /api/query
{
  "requestId": "req_002",
  "chatId": "chat_123",
  "agentKey": "auto",
  "role": "user",
  "message": "今天北京天气如何？请先查，再给出穿衣建议与注意事项。",
  "scene": { "url": "https://app.example.com/chat" },
  "stream": true
}
```

```text
event: message
data: {"seq":1,"type":"request.query","timestamp":1707000100,"requestId":"req_002","chatId":"chat_123","agentKey":"auto","role":"user","message":"今天北京天气如何？请先查，再给出穿衣建议与注意事项。","scene":{"url":"https://app.example.com/chat"},"stream":true}

event: message
data: {"seq":4,"type":"reasoning.start","timestamp":1707000103,"reasoningId":"re_002_1","runId":"run_002"}

event: message
data: {"seq":5,"type":"reasoning.delta","timestamp":1707000104,"reasoningId":"re_002_1","delta":"先检索天气要素，再给出穿衣建议。"}

event: message
data: {"seq":9,"type":"tool.start","timestamp":1707000108,"toolId":"tool_002_1","runId":"run_002","toolName":"web_search"}

event: message
data: {"seq":10,"type":"tool.args","timestamp":1707000109,"toolId":"tool_002_1","delta":"{\"query\":\"Beijing weather today\"}","chunkIndex":0}

event: message
data: {"seq":14,"type":"tool.result","timestamp":1707000113,"toolId":"tool_002_1","result":"{\"summary\":\"Sunny\",\"tempC\":25,\"wind\":\"NW 3\",\"aqi\":\"Good\"}"}

event: message
data: {"seq":15,"type":"content.start","timestamp":1707000114,"contentId":"ctx_002","runId":"run_002"}

event: message
data: {"seq":16,"type":"content.delta","timestamp":1707000115,"contentId":"ctx_002","delta":"北京今天晴，约 25°C，适合短袖或薄长袖。"}

event: message
data: {"seq":20,"type":"run.complete","timestamp":1707000119,"runId":"run_002","finishReason":"end_turn"}
```

## 3. 计划与多任务：Query + Plan + Tasks

适合“先检索，再写总结”这类分步骤任务。

```json
// POST /api/query
{
  "requestId": "req_003",
  "chatId": "c_01",
  "agentKey": "auto",
  "role": "user",
  "message": "帮我先查资料，再写一段总结。要求：先列出检索关键词，再给 5 条要点，最后给 1 段总结。",
  "scene": { "url": "https://app.example.com/chat" },
  "stream": true
}
```

```text
event: message
data: {"seq":1,"type":"request.query","timestamp":1707000200,"requestId":"req_003","chatId":"c_01","agentKey":"auto","role":"user","message":"帮我先查资料，再写一段总结。要求：先列出检索关键词，再给 5 条要点，最后给 1 段总结。","scene":{"url":"https://app.example.com/chat"},"stream":true}

event: message
data: {"seq":3,"type":"plan.update","timestamp":1707000202,"planId":"p_01","chatId":"c_01","plan":[{"taskId":"task_search","description":"检索资料并整理要点","status":"pending"},{"taskId":"task_write","description":"基于检索结果输出结构化要点与总结","status":"pending"}]}

event: message
data: {"seq":5,"type":"task.start","timestamp":1707000204,"taskId":"task_search","runId":"r_01","taskName":"Search","description":"检索资料并整理要点"}

event: message
data: {"seq":11,"type":"tool.start","timestamp":1707000210,"toolId":"t_search_1","runId":"r_01","taskId":"task_search","toolName":"web_search"}

event: message
data: {"seq":17,"type":"task.complete","timestamp":1707000216,"taskId":"task_search"}

event: message
data: {"seq":18,"type":"task.start","timestamp":1707000217,"taskId":"task_write","runId":"r_01","taskName":"Write","description":"基于检索结果输出结构化要点与总结"}

event: message
data: {"seq":20,"type":"content.delta","timestamp":1707000219,"contentId":"ctx_write_1","delta":"检索关键词：keyword1、keyword2"}

event: message
data: {"seq":25,"type":"run.complete","timestamp":1707000224,"runId":"r_01","finishReason":"end_turn"}
```

## 4. 文件 + 图片引用：Upload -> Query

先上传资源，再在 query 中通过 `#{{refid}}` 和 `references[]` 使用它们。

```json
// 1) POST /api/upload
{
  "requestId": "req_up_0001",
  "chatId": "7b09c3b3-8e6a-4df2-a7c4-3a6c2a1dc2c1",
  "file": "(binary requirements.md)",
  "sha256": "b9c3...e21a"
}
```

```json
// 1) Response
{
  "requestId": "req_up_0001",
  "chatId": "7b09c3b3-8e6a-4df2-a7c4-3a6c2a1dc2c1",
  "upload": {
    "id": "r01",
    "type": "file",
    "name": "requirements.md",
    "mimeType": "text/markdown",
    "sizeBytes": 48291,
    "url": "/api/resource?file=7b09c3b3-8e6a-4df2-a7c4-3a6c2a1dc2c1%2Frequirements.md"
  }
}
```

```json
// 2) POST /api/query
{
  "requestId": "req_0002",
  "chatId": "7b09c3b3-8e6a-4df2-a7c4-3a6c2a1dc2c1",
  "agentKey": "agent_researcher",
  "role": "user",
  "message": "请总结 #{{r01}} 的关键需求，并结合 #{{r02}} 指出协议中可能遗漏的字段。",
  "references": [
    { "id": "r01", "type": "file", "name": "requirements.md", "sandboxPath": "/workspace/requirements.md" },
    { "id": "r02", "type": "image", "name": "ui.png", "sandboxPath": "/workspace/ui.png" }
  ],
  "scene": { "url": "https://app.example.com/workspace/protocol" },
  "stream": true
}
```

关键点：

- 上传结果可以直接映射为 `Reference`
- `message` 中的 `#{{r01}}` 和 `references[]` 可以同时存在

## 5. 工作区选区引用：`type="file"` + `meta`

工作区代码片段不需要新类型，继续使用 `type="file"`，差异放进 `meta`。

```json
{
  "requestId": "req_0003",
  "chatId": "c40a08d0-b61d-4be0-99b5-3a9e8c7466c2",
  "agentKey": "agent_code_review",
  "role": "user",
  "message": "请审查我选中的代码片段（见 #{{f1}}），指出潜在 bug、边界条件与改进建议。",
  "references": [
    {
      "id": "f1",
      "type": "file",
      "name": "app/src/main.ts",
      "mimeType": "text/plain",
      "url": "https://workspace.example.com/ws/proj/file/app%2Fsrc%2Fmain.ts?selection=120:1-180:1",
      "meta": {
        "origin": "workspace",
        "filePath": "app/src/main.ts",
        "range": { "startLine": 120, "startCol": 1, "endLine": 180, "endCol": 1 },
        "textExcerpt": "function boot() { /* selected code */ }"
      }
    }
  ],
  "params": { "switch": { "strictMode": true }, "select": { "severity": "high" } },
  "scene": { "url": "https://app.example.com/ide/editor?file=app/src/main.ts" },
  "stream": true
}
```

关键点：

- 仍然用 `type="file"`
- `filePath`、`range`、`textExcerpt` 放在 `meta`

## 6. 图片为主：带 OCR 与 caption

图片场景建议补充 `caption` 和 `ocrText`，帮助模型少走一步。

```json
{
  "requestId": "req_0004",
  "chatId": "9d81c2c8-2c0c-4f7e-a2f0-5f5fd0a9b6b1",
  "agentKey": "auto",
  "role": "user",
  "message": "请解释截图 #{{i1}} 的报错原因，并给出排查步骤。",
  "references": [
    {
      "id": "i1",
      "type": "image",
      "name": "billing-error.png",
      "mimeType": "image/png",
      "url": "https://files.example.com/chat/9d81.../i1",
      "meta": {
        "origin": "upload",
        "caption": "支付页报错截图（包含错误码与时间戳）",
        "ocrText": "Error: BILLING_403\nRequestId: abcd-1234\nTime: 2026-02-06 10:01:22"
      }
    }
  ],
  "scene": { "url": "https://app.example.com/billing/checkout" },
  "stream": true
}
```

## 7. 运行中引导：Query 之后追加 Steer

`steer` 用于向同一个 `run` 追加新的用户意图。

```json
// 1) POST /api/query
{
  "requestId": "req_steer_001",
  "chatId": "chat_live_001",
  "agentKey": "auto",
  "role": "user",
  "message": "先给我一版三点总结，主题是我们新的 AGW 协议文档。",
  "stream": true
}
```

```json
// 2) POST /api/steer
{
  "requestId": "req_steer_002",
  "chatId": "chat_live_001",
  "runId": "run_live_001",
  "steerId": "steer_001",
  "agentKey": "auto",
  "teamId": "docs",
  "message": "补充一个要求：输出前先列出 HTTP API 和 SSE event 的边界，再继续完成总结。",
  "planningMode": false
}
```

```text
event: message
data: {"seq":6,"type":"request.steer","timestamp":1707000705,"requestId":"req_steer_002","chatId":"chat_live_001","runId":"run_live_001","steerId":"steer_001","message":"补充一个要求：输出前先列出 HTTP API 和 SSE event 的边界，再继续完成总结。","role":"user"}
```

关键点：

- `steer` 是独立 HTTP 请求
- 结果继续体现在原 SSE 流里

## 8. 运行中中断：Interrupt -> `run.cancel`

中断不会映射成独立的 `request.interrupt` 事件。

```json
// POST /api/interrupt
{
  "requestId": "req_interrupt_002",
  "chatId": "chat_live_002",
  "runId": "run_live_002",
  "agentKey": "auto",
  "teamId": "docs",
  "message": "先停止当前回答，我要改需求。",
  "planningMode": false
}
```

```text
event: message
data: {"seq":7,"type":"run.cancel","timestamp":1707000806,"runId":"run_live_002"}

event: message
data: [DONE]
```

关键点：

- 中断的同步确认来自 HTTP 响应
- 流层结果由 `run.cancel` 表达

## 9. 文件产物发布：一次工具调用，多条 `artifact.publish`

一次 `_artifact_publish_` 可以发布多个文件，并在流里拆成多条 `artifact.publish`。

```json
{
  "name": "_artifact_publish_",
  "arguments": {
    "artifacts": [
      {
        "path": "/workspace/reports/final-report.md",
        "description": "最终交付报告"
      },
      {
        "path": "/workspace/reports/summary.md",
        "name": "summary.md"
      }
    ]
  }
}
```

```text
event: message
data: {"seq":9,"type":"tool.start","timestamp":1707000908,"toolId":"tool_publish_1","runId":"run_artifact_001","toolName":"_artifact_publish_"}

event: message
data: {"seq":12,"type":"tool.result","timestamp":1707000911,"toolId":"tool_publish_1","result":"{\"ok\":true,\"artifacts\":[...]}"} 

event: message
data: {"seq":13,"type":"artifact.publish","timestamp":1707000912,"artifactId":"asset_5d4c3b2a1f00","chatId":"chat_artifact_001","runId":"run_artifact_001","artifact":{"type":"file","name":"final-report.md","mimeType":"text/markdown","url":"/api/resource?file=chat_artifact_001%2Fartifacts%2Frun_artifact_001%2Ffinal-report.md"}}

event: message
data: {"seq":14,"type":"artifact.publish","timestamp":1707000913,"artifactId":"asset_5d4c3b2a1f01","chatId":"chat_artifact_001","runId":"run_artifact_001","artifact":{"type":"file","name":"summary.md","mimeType":"text/markdown","url":"/api/resource?file=chat_artifact_001%2Fartifacts%2Frun_artifact_001%2Fsummary.md"}}
```

关键点：

- 一次工具调用可以对应多个产物
- 每个产物都有独立的 `artifact.publish`
- `runId` 表示来源，不改变 chat 级归属
