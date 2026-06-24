> Migrated from `docs/架构/22-数据模型与持久化规范.md` on 2026-06-24.
> Owner: SDKWork maintainers

# 数据模型与持久化规范

## 1. 目标

统一定义 `sdkworkfs` 的中心持久化模型、节点本地持久化模型、缓存索引、审计存储和保留规则，确保控制面元数据、节点执行状态和缓存索引都有明确的落库边界，而不是散落在对象存储路径或本地隐式文件中。

接近实现的 PostgreSQL / SQLite DDL 示例见 `37-数据库DDL与索引示例.md`。

## 2. 持久化总原则

- 中心数据库保存控制面真相，包括配置、Release、审批、操作、审计
- 对象存储保存内容真相，包括 Manifest、对象内容、压缩元信息
- 节点本地 SQLite 保存执行面真相，包括本地缓存索引、写回任务、冲突记录、预热状态
- 内存状态只作为加速层，不得成为唯一状态来源

## 3. 中心数据库模型

推荐使用 PostgreSQL。核心实体至少包括：

- `tenant`
- `repository`
- `application`
- `environment`
- `release`
- `release_source`
- `manifest`
- `release_object`
- `upload_session`
- `storage_object`
- `release_activation`
- `node`
- `node_heartbeat`
- `git_replication_relation`
- `operation`
- `approval_record`
- `audit_log`

### 3.1 关键表说明

`release`

| 字段 | 说明 |
| --- | --- |
| `release_id` | 发布唯一 ID |
| `tenant_id` | 所属租户 |
| `app_id` | 应用 ID |
| `channel` | 发布通道 |
| `status` | Release 状态 |
| `source_type` | `upload` / `git` / `s3` / `mixed` |
| `source_version` | 分支、tag、commit 或对象版本 |
| `manifest_digest` | Manifest 哈希 |
| `created_at` | 创建时间 |
| `published_at` | 发布到通道时间 |

`manifest`

| 字段 | 说明 |
| --- | --- |
| `manifest_digest` | Manifest 唯一哈希 |
| `schema_version` | Schema 版本 |
| `object_count` | 对象数量 |
| `total_size` | 内容总大小 |
| `storage_encoding` | Manifest 存储编码 |
| `payload_json` | 规范化后的 `entries` 正文快照 |
| `created_at` | 创建时间 |

`release_object`

| 字段 | 说明 |
| --- | --- |
| `release_id` | 关联 Release |
| `object_hash` | 对象内容哈希 |
| `path` | 物化路径 |
| `size` | 逻辑大小 |
| `storage_size` | 实际存储大小 |
| `kind` | `static` / `binary` / `config` / `media` / `model` / `data` |
| `content_type` | MIME 类型 |
| `mode` | 文件权限或执行位信息 |

`release_object` 由已登记的 Manifest `entries` 派生并落库，作为节点物化、本地缓存和对象存储下载的最小对象清单来源。

`upload_session`

| 字段 | 说明 |
| --- | --- |
| `upload_id` | 上传会话唯一 ID |
| `tenant_id` | 所属租户 |
| `object_hash` | 目标对象哈希 |
| `bucket` | 对象存储 bucket |
| `object_key` | 内容寻址对象键 |
| `size` | 逻辑大小 |
| `storage_size` | 预计存储大小 |
| `storage_encoding` | `none` / `gzip` / `zstd` |
| `content_type` | MIME 类型 |
| `status` | `INITIATED` / `COMPLETED` |
| `etag` | 对象存储返回的 etag |
| `created_at` | 创建时间 |
| `completed_at` | 完成时间 |

`storage_object`

| 字段 | 说明 |
| --- | --- |
| `object_hash` | 内容寻址对象哈希 |
| `bucket` | 对象所在 bucket |
| `object_key` | 对象键 |
| `size` | 逻辑大小 |
| `storage_size` | 存储大小 |
| `storage_encoding` | 对象存储编码 |
| `content_type` | MIME 类型 |
| `etag` | 对象存储 etag |
| `status` | `AVAILABLE` / `DELETED` / `CORRUPTED` |
| `source_type` | `UPLOAD` / `S3_IMPORT` / `GIT_BUILD` |
| `created_at` | 首次登记时间 |
| `completed_at` | 完成登记时间 |

`storage_object` 是 Upload、S3 Import 与 Git Build 共用的对象真相源，`upload_session` 用于记录每次上传请求及去重命中结果。Upload 类型的 `Release verify` 应在 `manifest` 与 `release_object` 一致性之外，进一步校验所有引用对象都已在 `storage_object` 中完成登记。

`operation`

| 字段 | 说明 |
| --- | --- |
| `operation_id` | 操作唯一 ID |
| `type` | 操作类型 |
| `status` | 操作状态 |
| `target_resource_type` | 目标资源类型 |
| `target_resource_id` | 目标资源 ID |
| `progress` | 进度百分比 |
| `error_code` | 失败错误码 |
| `started_at` | 开始执行时间 |
| `updated_at` | 最近状态更新时间 |
| `summary` | 执行摘要 |
| `retry_count` | 已执行重试次数 |
| `created_at` | 创建时间 |
| `finished_at` | 完成时间 |

`audit_log`

| 字段 | 说明 |
| --- | --- |
| `audit_id` | 审计记录唯一 ID |
| `resource_type` | 资源类型，如 `REPOSITORY` |
| `resource_id` | 资源 ID |
| `action` | 动作类型，如 `SYNC_REPOSITORY` |
| `status` | 动作状态，如 `STARTED` / `SUCCEEDED` / `FAILED` |
| `operator` | 操作主体 |
| `request_id` | 请求链路 ID |
| `operation_id` | 关联异步操作 ID |
| `detail_summary` | 摘要信息 |
| `created_at` | 创建时间 |

`approval_record`

| 字段 | 说明 |
| --- | --- |
| `approval_id` | 审批记录 ID |
| `target_type` | 目标类型，如 `release` |
| `target_id` | 目标资源 ID |
| `environment` | 审批生效环境 |
| `decision` | `approved` / `rejected` / `expired` |
| `approver_role` | 审批人角色 |
| `approved_at` | 审批时间 |

### 3.2 索引建议

中心库建议至少建立：

- `release(app_id, channel, created_at desc)`
- `release(status, created_at desc)`
- `release(tenant_id, source_type, source_version)`
- `manifest(created_at desc)`
- `release_object(release_id, path)`
- `release_object(object_hash)`
- `upload_session(object_hash)`
- `upload_session(status, created_at desc)`
- `storage_object(object_key)`
- `storage_object(status, completed_at desc)`
- `release_activation(app_id, environment, activated_at desc)`
- `operation(status, updated_at desc)`
- `operation(target_resource_type, target_resource_id, created_at desc)`
- `node(region, role, heartbeat_at desc)`
- `node_heartbeat(node_id, heartbeat_at desc)`
- `approval_record(target_type, target_id, created_at desc)`
- `audit_log(resource_type, resource_id, created_at desc)`

### 3.3 唯一约束与引用关系

- `tenant.tenant_id` 全局唯一
- `repository(tenant_id, name)` 唯一
- `application(tenant_id, app_id)` 唯一
- `release.release_id` 全局唯一
- `manifest.manifest_digest` 全局唯一
- `release_object(release_id, path)` 唯一
- `upload_session.upload_id` 全局唯一
- `storage_object.object_hash` 全局唯一
- `storage_object.object_key` 全局唯一
- `node.node_id` 全局唯一
- `git_replication_relation(repository_id, remote_name, direction)` 唯一
- `approval_record(approval_id)` 唯一

引用建议：

- `release.app_id -> application.app_id`
- `release.manifest_digest -> manifest.manifest_digest`
- `release_object.release_id -> release.release_id`
- `upload_session.object_hash -> storage_object.object_hash` 建议采用逻辑关联
- `operation.target_resource_id` 采用逻辑关联，并在应用层校验目标存在

### 3.4 分区、归档与冷热分层

- `audit_log`、`node_heartbeat` 建议按月分区
- 高容量表建议采用 `created_at` 时间分区或哈希分区
- 热数据保留在主表，冷数据归档到分区或历史表
- 大批量心跳与操作日志不建议长期占用主业务索引

## 4. 节点本地持久化模型

推荐使用 SQLite。核心表至少包括：

- `local_release`
- `materialized_release`
- `local_object_cache`
- `writeback_task`
- `conflict_record`
- `prefetch_task`
- `local_config_snapshot`
- `local_operation_log`

### 4.1 节点表说明

`local_release`

| 字段 | 说明 |
| --- | --- |
| `release_id` | 本地可用 Release |
| `app_id` | 应用 ID |
| `environment` | 环境 |
| `status` | 本地状态 |
| `manifest_digest` | Manifest 哈希 |
| `materialized_root` | 本地物化目录 |
| `last_access_at` | 最近访问时间 |

`local_object_cache`

| 字段 | 说明 |
| --- | --- |
| `object_hash` | 对象哈希 |
| `cache_path` | 本地缓存路径 |
| `storage_encoding` | `none` / `gzip` / `zstd` |
| `size_bytes` | 原始大小 |
| `storage_size_bytes` | 缓存占用大小 |
| `last_hit_at` | 最近命中时间 |
| `pin_state` | 是否被 pin |

`writeback_task`

| 字段 | 说明 |
| --- | --- |
| `task_id` | 任务唯一 ID |
| `app_id` | 应用 ID |
| `path` | 写入路径 |
| `operation` | `put` / `delete` / `rename` |
| `status` | 状态 |
| `retry_count` | 重试次数 |
| `expected_version` | 冲突检测版本 |
| `last_error_code` | 最近失败错误码 |

### 4.2 节点索引建议

- `local_release(app_id, environment, status)`
- `local_release(last_access_at desc)`
- `materialized_release(app_id, environment, active_state)`
- `local_object_cache(object_hash)`
- `local_object_cache(last_hit_at desc)`
- `writeback_task(status, updated_at)`
- `writeback_task(app_id, path, status)`
- `prefetch_task(status, priority)`
- `conflict_record(app_id, created_at desc)`
- `local_operation_log(created_at desc)`

### 4.3 节点约束建议

- `release_id`、`task_id`、`object_hash` 必须唯一
- 同一路径同一时刻仅允许一个活跃强一致写任务
- 已激活 `materialized_release` 同一应用同一环境最多一个
- 被 pin 的对象不得被淘汰任务误删

## 5. 审计存储

审计日志至少要保存：

- 操作人
- 操作时间
- 目标资源
- 资源版本
- 来源 IP、节点或服务账号
- 请求参数摘要与结果状态
- 关联 `requestId`、`operationId`

仓库同步链路的 `STARTED` / `SUCCEEDED` / `FAILED` 审计记录必须落到中心库，并支持按资源维度检索。
Release 校验应依赖 Manifest 正文与 `release_object` 一致性，避免“只有摘要，没有对象列表”的发布记录进入后续阶段。
Upload 链路应记录 `INIT_UPLOAD` / `COMPLETE_UPLOAD` 审计轨迹，便于追踪对象登记和复用行为。

## 6. 保留与归档

建议保留期：

- 发布元数据：长期
- Manifest 元数据：长期
- 节点任务日志：`7-30` 天
- 节点心跳明细：`30-90` 天
- 审计日志：`180-365` 天或合规要求更长
- 冲突记录：至少覆盖最近一个完整发布周期

节点本地建议：

- SQLite 周期性 `VACUUM` 和 `ANALYZE`
- 超过保留期的已完成任务归档后再清理
- 缓存淘汰前先校验引用计数、pin 状态和激活状态

## 7. 事务边界建议

- Release 创建、Source 绑定、Manifest 关联应在中心库同一事务内完成
- 发布审批通过与发布状态推进必须具备明确顺序关系
- Node 心跳写入应轻量化，避免长事务和全表热点
- 写回任务状态更新与本地文件系统变更必须记录同一逻辑操作 ID
- `operation` 状态推进与 `audit_log` 记录必须通过相同的 `requestId` / `operationId` 关联
- Git 同步结果、审计记录和最终资源状态至少要做到最终一致

## 8. 正式要求

- Release 元数据不得只保存在对象存储路径推导中
- 节点关键状态不得只保存在内存中
- 审计数据必须具备持久化与检索能力
- 核心唯一约束、索引与引用关系必须在实现阶段显式定义
- 本地 SQLite 必须能够在节点重启后恢复缓存、任务与当前运行状态

