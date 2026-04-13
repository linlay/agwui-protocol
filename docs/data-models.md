# 共享数据模型

## 1. `Reference`

`Reference` 是协议中的统一引用对象。附件、工作区文件、选区、截图、图片等都用它表达，并通过 `reference.id` 与文本标记或显式 `references[]` 关联。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | `string` | 短 ID，要求在当前 chat 内唯一，例如 `f1`、`i1`、`r01` |
| `type` | `"file" \| "image"` | 引用类型 |
| `name` | `string` | 展示名，可选 |
| `mimeType` | `string` | MIME 类型，建议总是提供 |
| `sizeBytes` | `number` | 字节数，可选 |
| `url` | `string` | 网关可访问地址或可下载地址 |
| `sha256` | `string` | 去重、校验或缓存使用，可选 |
| `sandboxPath` | `string` | 沙箱内可访问路径，可选 |
| `meta` | `object` | 来源差异和增强信息，可选 |

### `meta` 常见字段

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `origin` | `"upload" \| "workspace" \| "clipboard" \| "url"` | 引用来源 |
| `filePath` | `string` | 工作区文件路径 |
| `range` | `object` | 选区范围，结构为 `startLine/startCol/endLine/endCol` |
| `textExcerpt` | `string` | 摘录文字，减少必须先拉全文的情况 |
| `caption` | `string` | 图片说明或用户补充描述 |
| `ocrText` | `string` | 图片 OCR 结果 |

## 2. 引用标记规则

协议支持两种并行方式：

- 在 `message` 中写 `#{{refid}}`
- 在请求中显式附带 `references[]`

推荐同时使用，便于渲染、权限控制和执行优化。

示例：

```json
{
  "message": "请总结 #{{f1}} 并解释 #{{i1}} 中的流程",
  "references": [
    { "id": "f1", "type": "file", "name": "requirements.md" },
    { "id": "i1", "type": "image", "name": "ui.png" }
  ]
}
```

## 3. 通用 ID 词汇表

| 字段 | 所属层级 | 说明 |
| --- | --- | --- |
| `requestId` | Request | 前端生成，用于幂等、重试、链路追踪 |
| `chatId` | Chat | 聊天会话 ID，推荐前端生成 UUID |
| `agentKey` | Routing | 指定或提示选择哪个 Agent |
| `teamId` | Routing | 团队或租户路由信息，可选 |
| `runId` | Run | 一次执行过程的唯一标识 |
| `taskId` | Task | 某次 run 中的任务标识 |
| `planId` | Plan | 计划聚合对象的唯一标识 |
| `reasoningId` | Reasoning | 一段推理输出的标识 |
| `contentId` | Content | 一段正文输出的标识 |
| `toolId` | Tool | 一次工具调用的标识 |
| `actionId` | Action | 一次动作渲染或执行的标识 |
| `artifactId` | Artifact | 单个发布产物的标识 |
| `steerId` | Steer | 一次运行中追加指令的标识 |
| `viewportKey` | Viewport | 获取某个前端视图 payload 的键 |

## 4. Query 相关通用字段

`POST /api/query` 常用到的公共字段如下：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `role` | `"user" \| "system" \| "assistant" \| "developer" \| "other"` | 当前消息角色 |
| `message` | `string` | 用户输入正文 |
| `references` | `Reference[]` | 显式引用列表 |
| `params` | `object` | 结构化输入，如 `input/select/datepicker/switch` |
| `scene` | `object` | 页面上下文，建议至少提供 `url` |
| `hidden` | `boolean` | 是否隐藏部分前端展示内容 |
| `stream` | `boolean` | 保留兼容字段；当前 query 固定返回 SSE |

### `scene`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `url` | `string` | 页面 URL，建议提供 |
| `title` | `string` | 页面标题，可选 |

## 5. SSE 事件公共包络

实时业务事件统一包含如下三元组：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `seq` | `number` | 事件序号 |
| `type` | `string` | 事件类型，如 `run.start` |
| `timestamp` | `number` | 事件时间戳 |

此外可能包含调试字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `rawEvent` | `object` | 原始事件对象，仅调试用途 |

## 6. Chat 聚合对象

除了单次事件，协议还会在会话层维护聚合态：

### `plan`

- 表示 chat 当前最新计划状态
- 形如 `{ planId, tasks }`
- 实时流里以 `plan.update` 表达

### `artifact`

- 表示 chat 当前最新产物集合
- 形如 `{ items: [...] }`
- 实时流里每个产物对应一条 `artifact.publish`

### `references`

- 可理解为 query 引用与 chat 资产聚合后的引用池
- 常见于 `GET /api/chat` 的会话详情返回

## 7. 上传对象与资源对象

`POST /api/upload` 成功后，返回的 `upload` 可直接映射为 `Reference` 使用。推荐保留以下字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | `string` | chat 内短 ID，如 `r01` |
| `type` | `"file" \| "image"` | 上传资源类型 |
| `name` | `string` | 文件名 |
| `mimeType` | `string` | MIME 类型 |
| `sizeBytes` | `number` | 文件大小 |
| `url` | `string` | 上传完成后的资源地址 |
| `sha256` | `string` | 校验值，可选 |

## 8. 设计建议

- `id` 尽量短，便于在文本中用 `#{{refid}}` 引用
- 对工作区引用优先补 `meta.filePath`、`meta.range`、`meta.textExcerpt`
- 对图片引用优先补 `meta.caption`、`meta.ocrText`
- `mimeType` 和 `url` 是跨端渲染最关键的两个字段，建议总是提供
