> Migrated from `docs/架构/24-资源Schema与状态机规范.md` on 2026-06-24.
> Owner: SDKWork maintainers

# 资源 Schema 与状态机规范

## 1. 目标

统一定义核心资源的 schema、必填字段、可选字段、通用元字段和状态流转，避免不同模块各自理解 `Release`、`Operation`、`Node`、`WritebackTask` 的生命周期。

## 2. 通用资源字段

所有核心资源建议统一包含以下元字段：

| 字段 | 说明 |
| --- | --- |
| `resourceId` | 资源逻辑主键 |
| `tenantId` | 所属租户 |
| `resourceVersion` | 乐观锁版本 |
| `labels` | 轻量标签 |
| `annotations` | 扩展元数据 |
| `createdAt` | 创建时间 |
| `updatedAt` | 更新时间 |
| `createdBy` | 创建人或系统主体 |
| `updatedBy` | 更新人或系统主体 |

## 3. Release Schema

### 3.1 必填字段

| 字段 | 说明 |
| --- | --- |
| `releaseId` | 发布唯一 ID |
| `appId` | 应用 ID |
| `environment` | 目标环境 |
| `channel` | 发布通道 |
| `status` | Release 状态 |
| `sourceType` | 来源类型 |
| `sourceVersion` | 分支、tag、commit 或对象版本 |
| `manifestDigest` | Manifest 哈希 |
| `objectCount` | 对象数量 |
| `createdAt` | 创建时间 |

### 3.2 可选字段

| 字段 | 说明 |
| --- | --- |
| `publishedAt` | 发布到通道时间 |
| `activatedAt` | 最后激活时间 |
| `rollbackOf` | 回滚来源版本 |
| `stable` | 是否为稳定版本 |
| `approvalSummary` | 审批摘要 |

### 3.3 Release 状态机

```text
CREATED
-> OBJECTS_INDEXED
-> VERIFIED
-> PUBLISHED
-> PREFETCHING
-> ACTIVATED
-> RETIRED
```

失败状态：

- `FAILED_BUILD`
- `FAILED_VERIFY`
- `FAILED_PREFETCH`
- `FAILED_ACTIVATE`

### 3.4 非法状态流转

- 不允许 `ACTIVATED -> CREATED`
- 不允许 `RETIRED -> ACTIVATED`，必须通过新的激活操作实现
- 不允许 `FAILED_VERIFY -> ACTIVATED`
- 不允许缺少 `manifestDigest` 的 Release 进入 `VERIFIED`

## 4. Operation Schema

### 4.1 必填字段

| 字段 | 说明 |
| --- | --- |
| `operationId` | 操作唯一 ID |
| `type` | `publish` / `prefetch` / `switch` / `rollback` / `sync` / `gc` |
| `status` | 当前状态 |
| `targetResourceType` | 目标资源类型 |
| `targetResourceId` | 目标资源 ID |
| `progress` | 进度百分比 |
| `createdAt` | 创建时间 |

### 4.2 可选字段

| 字段 | 说明 |
| --- | --- |
| `startedAt` | 开始时间 |
| `finishedAt` | 完成时间 |
| `errorCode` | 失败错误码 |
| `result` | 结果摘要 |
| `retryable` | 是否可重试 |

### 4.3 Operation 状态机

```text
PENDING -> RUNNING -> SUCCEEDED
PENDING -> RUNNING -> FAILED
PENDING -> CANCELLED
RUNNING -> CANCELLED
```

阶段性约束：

- 首期实现至少支持 `PENDING -> CANCELLED`
- `RUNNING -> CANCELLED` 作为扩展状态机保留，待协作式取消或分布式执行器阶段实现

非法状态：

- `SUCCEEDED -> RUNNING`
- `FAILED -> SUCCEEDED`
- `CANCELLED -> RUNNING`

## 5. Node Schema

### 5.1 必填字段

| 字段 | 说明 |
| --- | --- |
| `nodeId` | 节点唯一 ID |
| `role` | 节点角色 |
| `region` | 地域 |
| `lifecycleStatus` | 生命周期状态 |
| `healthStatus` | 健康状态 |
| `heartbeatAt` | 最近心跳时间 |
| `daemonVersion` | 节点守护进程版本 |

### 5.2 Node 状态

生命周期状态：

- `REGISTERING`
- `READY`
- `DRAINING`
- `FENCED`
- `OFFLINE`

健康状态：

- `HEALTHY`
- `DEGRADED`
- `UNAVAILABLE`

### 5.3 非法状态

- `FENCED` 节点不得继续接受新的激活任务
- `OFFLINE` 节点不得被选择进入金丝雀范围
- `REGISTERING` 节点不得直接承载正式 `prod` 激活

## 6. WritebackTask Schema

### 6.1 必填字段

| 字段 | 说明 |
| --- | --- |
| `taskId` | 任务 ID |
| `appId` | 应用 ID |
| `path` | 写入路径 |
| `operation` | `put` / `delete` / `rename` |
| `status` | 当前状态 |
| `expectedVersion` | 冲突检测版本 |
| `createdAt` | 创建时间 |

### 6.2 可选字段

| 字段 | 说明 |
| --- | --- |
| `retryCount` | 重试次数 |
| `lastErrorCode` | 最近错误码 |
| `resolvedBy` | 冲突处理人或系统主体 |
| `resolvedAt` | 冲突处理时间 |

### 6.3 Writeback 状态机

```text
PENDING
-> PACKING
-> UPLOADING
-> VERIFYING
-> DONE
```

异常状态：

- `FAILED_RETRYABLE`
- `FAILED_PERMANENT`
- `CONFLICT`
- `CANCELLED`

非法状态：

- `DONE -> UPLOADING`
- `CONFLICT -> DONE`，除非形成新的重试任务
- `FAILED_PERMANENT -> RUNNING`

## 7. 通用状态机约束

- 所有资源状态变更必须产生审计记录
- 状态推进必须包含 `updatedAt` 和触发人或系统主体
- 失败状态必须附带错误码与摘要
- 资源扩展状态时必须先更新文档和消费者升级策略

## 8. 正式要求

- 核心资源必须定义稳定 schema
- 任何状态机扩展都必须通过显式版本升级或同步发布计划落地
- 不允许核心资源只有隐式状态、没有明确状态字段
- 所有核心资源都必须支持 `resourceVersion` 或等价并发控制字段

## 9. 补充约束：自动候选发布

- `Application` 资源增加 `autoReleaseEnabled`、`autoReleaseEnvironment`、`autoReleaseChannel` 三个字段，用于声明同步成功后是否自动排队候选 Release
- 自动排队本身不引入新的 `Operation` 类型，仍复用现有 `BUILD_RELEASE`；但会新增一条 `AUTO_TRIGGER_BUILD_RELEASE` 审计轨迹，用于区分“排队成功”和“构建执行结果”
- 自动排队采用确定性 `releaseId`：优先使用 `auto-<appId>-<environment>-<commit12>`，若超过 64 字符则退化为 `auto-<appPrefix>-<envPrefix>-<commit12>-<hash8>`；同一应用、同一环境、同一已解析 commit 仍共享同一个候选 Release 标识

