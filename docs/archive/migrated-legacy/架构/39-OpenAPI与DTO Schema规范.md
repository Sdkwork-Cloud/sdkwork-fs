# OpenAPI 与 DTO Schema 规范

## 1. 目标

把控制面 API 从“路径存在”进一步细化为可生成 SDK、可校验契约、可前后端联调的 OpenAPI 与 DTO 标准，确保字段命名、枚举、错误模型、分页模型和示例响应保持一致。

## 2. OpenAPI 总体要求

- 外部 HTTP API 必须产出 OpenAPI 3.0+ 文档
- 文档必须与实际实现同步发布
- 所有资源模型、错误模型、分页模型、异步操作模型必须进入 `components/schemas`
- 破坏性变更必须通过新版本路径或显式弃用流程处理

## 3. 命名规范

### 3.1 路径

- 使用复数资源名，如 `/releases`
- 动作型接口使用 `:verb` 后缀，如 `/releases/{releaseId}:publish`
- 内部节点接口使用 `/internal/v1/...`

### 3.2 字段命名

- JSON 字段统一 `camelCase`
- 枚举值统一 `UPPER_SNAKE_CASE` 或固定小写模式，单个 schema 内不得混用
- 时间字段统一 ISO-8601 UTC，如 `2026-04-06T01:00:00Z`

## 4. 通用 Envelope Schema

建议统一：

```yaml
ApiResponse:
  type: object
  required: [requestId, code]
  properties:
    requestId:
      type: string
    code:
      type: string
    message:
      type: string
    data:
      nullable: true
```

分页响应建议：

```yaml
PageResponse:
  type: object
  required: [items, page, pageSize, total]
  properties:
    items:
      type: array
      items: {}
    page:
      type: integer
    pageSize:
      type: integer
    total:
      type: integer
```

## 5. 核心 DTO 建议

### 5.1 ReleaseDto

```yaml
ReleaseDto:
  type: object
  required:
    - releaseId
    - appId
    - environment
    - status
    - sourceType
    - sourceVersion
    - manifestDigest
  properties:
    releaseId:
      type: string
    appId:
      type: string
    environment:
      type: string
    status:
      $ref: '#/components/schemas/ReleaseStatus'
    sourceType:
      type: string
    sourceVersion:
      type: string
    manifestDigest:
      type: string
```

### 5.2 OperationDto

```yaml
OperationDto:
  type: object
  required: [operationId, type, status, targetResourceType, targetResourceId]
  properties:
    operationId:
      type: string
    type:
      $ref: '#/components/schemas/OperationType'
    status:
      $ref: '#/components/schemas/OperationStatus'
    targetResourceType:
      type: string
    targetResourceId:
      type: string
    progress:
      type: integer
      minimum: 0
      maximum: 100
```

### 5.3 ErrorResponse

```yaml
ErrorResponse:
  type: object
  required: [requestId, code, message]
  properties:
    requestId:
      type: string
    code:
      type: string
    message:
      type: string
    retryable:
      type: boolean
    details:
      type: object
      additionalProperties: true
```

## 6. 枚举规范

建议把所有关键枚举集中定义，例如：

- `ReleaseStatus`
- `OperationStatus`
- `NodeHealthStatus`
- `ApprovalDecision`
- `DeliveryMode`
- `SourceType`

要求：

- 枚举文案稳定
- 新增值必须评估 SDK 版本影响并同步更新生成产物
- 不允许同一语义在不同接口中使用不同字符串

## 7. 参数规范

### 7.1 分页参数

- `page`
- `pageSize`
- `cursor`
- `sortBy`
- `sortOrder`

### 7.2 过滤参数

- `tenantId`
- `appId`
- `environment`
- `status`
- `updatedAfter`
- `updatedBefore`

### 7.3 Header 参数

- `X-Request-Id`
- `Idempotency-Key`
- `If-Match`

## 8. 示例与示例值要求

- 关键请求和响应必须带 `example`
- 错误响应至少要覆盖权限拒绝、资源缺失、冲突、校验失败
- 异步操作接口必须给出 `operationId` 响应示例

## 9. 代码生成要求

- Java、TypeScript、Python SDK 必须能从 OpenAPI 生成基础 DTO 和客户端
- 手写 DTO 与生成 DTO 不应长期并存为两套模型
- 契约变更应优先更新 OpenAPI，再驱动 SDK 与服务实现

## 10. 正式要求

- OpenAPI 必须成为控制面 API 的契约标准
- DTO、错误模型、分页模型、枚举必须统一治理
- 所有外部接口都必须有 schema、示例和版本说明
