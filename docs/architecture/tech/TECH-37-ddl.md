> Migrated from `docs/架构/37-数据库DDL与索引示例.md` on 2026-06-24.
> Owner: SDKWork maintainers

# 数据库 DDL 与索引示例

## 1. 目标

把数据模型从概念表结构进一步细化为接近实现的 PostgreSQL / SQLite DDL 示例，帮助后续快速落数据库 schema、索引和约束。

## 2. PostgreSQL 示例

### 2.1 tenant

```sql
create table tenant (
  tenant_id varchar(64) primary key,
  name varchar(128) not null,
  status varchar(32) not null,
  created_at timestamptz not null default now()
);
```

### 2.2 repository

```sql
create table repository (
  repository_id varchar(64) primary key,
  tenant_id varchar(64) not null references tenant(tenant_id),
  name varchar(128) not null,
  default_branch varchar(128) not null,
  workflow_mode varchar(32) not null,
  replication_mode varchar(32) not null,
  source_type varchar(32) not null,
  source_settings text,
  replication_settings text,
  status varchar(32) not null,
  resource_version bigint not null default 0,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  last_sync_status varchar(32),
  last_sync_remote varchar(128),
  last_sync_remote_url varchar(512),
  last_sync_source_ref varchar(256),
  last_sync_resolved_version varchar(128),
  last_sync_fingerprint varchar(256),
  last_sync_operation_id varchar(64),
  last_sync_error_code varchar(64),
  last_sync_summary varchar(512),
  last_sync_at timestamptz,
  unique (tenant_id, name)
);
```

`repository` 既保存来源与复制配置，也保存最近一次同步结果投影。对于 `source_type = 'GIT'` 的仓库，控制面应把最近一次同步命中的：

- `last_sync_remote_url`
- `last_sync_source_ref`
- `last_sync_resolved_version`

直接持久化，供后续 Release Builder、审计和运维诊断复用。

### 2.3 application

```sql
create table application (
  app_id varchar(64) primary key,
  tenant_id varchar(64) not null references tenant(tenant_id),
  repository_id varchar(64) not null references repository(repository_id),
  runtime_type varchar(32) not null,
  delivery_mode varchar(32) not null,
  write_policy varchar(32) not null,
  current_release_id varchar(64),
  resource_version bigint not null default 0,
  created_at timestamptz not null default now(),
  unique (tenant_id, app_id)
);
```

### 2.4 manifest

```sql
create table manifest (
  manifest_digest varchar(128) primary key,
  schema_version varchar(16) not null,
  object_count bigint not null,
  total_size bigint not null,
  storage_encoding varchar(32) not null,
  payload_json text,
  created_at timestamptz not null default now()
);
```

控制面应登记 Manifest 元数据和 `payload_json` 正文；`Release verify` 会基于该表校验 `manifest_digest` 是否已注册，并校验：

- `schema_version = 'v1'`
- `object_count > 0`
- `storage_encoding` 属于 `none` / `gzip` / `zstd`
- `payload_json` 中的 `entries` 数量与大小汇总必须与元数据一致

### 2.5 release

```sql
create table release_record (
  release_id varchar(64) primary key,
  tenant_id varchar(64) not null references tenant(tenant_id),
  app_id varchar(64) not null references application(app_id),
  environment varchar(32) not null,
  channel varchar(32) not null,
  status varchar(32) not null,
  source_type varchar(32) not null,
  source_version varchar(256) not null,
  manifest_digest varchar(128) not null references manifest(manifest_digest),
  stable boolean not null default false,
  created_at timestamptz not null default now(),
  published_at timestamptz,
  verified_at timestamptz,
  activated_at timestamptz,
  rollback_of varchar(64)
);
```

`release_record` 既可来自 Upload 型 `POST /api/v1/releases`，也可来自 `POST /api/v1/applications/{appId}:build-release`：

- Git Builder 入库时会写入 `source_type = 'GIT'`
- `source_version` 当前格式为 `<last_sync_source_ref>@<last_sync_resolved_version>`
- 初始状态为 `CREATED`，后续继续沿用同一套 `verify -> publish -> activate -> rollback` 生命周期

### 2.6 release_object

```sql
create table release_object (
  release_id varchar(64) not null references release_record(release_id),
  path text not null,
  object_hash varchar(128) not null,
  size bigint not null,
  storage_size bigint not null,
  kind varchar(32) not null,
  content_type varchar(128),
  mode varchar(16),
  primary key (release_id, path)
);
```

`release_object` 由 Manifest `entries` 派生，并作为 `GET /api/v1/releases/{releaseId}/objects` 的查询来源。

### 2.7 upload_session

```sql
create table upload_session (
  upload_id varchar(64) primary key,
  tenant_id varchar(64) not null references tenant(tenant_id),
  object_hash varchar(128) not null,
  bucket varchar(128) not null,
  object_key varchar(512) not null,
  size bigint not null,
  storage_size bigint not null,
  storage_encoding varchar(32) not null,
  content_type varchar(128),
  status varchar(32) not null,
  etag varchar(128),
  created_at timestamptz not null default now(),
  completed_at timestamptz
);
```

### 2.8 storage_object

```sql
create table storage_object (
  object_hash varchar(128) primary key,
  bucket varchar(128) not null,
  object_key varchar(512) not null,
  size bigint not null,
  storage_size bigint not null,
  storage_encoding varchar(32) not null,
  content_type varchar(128),
  etag varchar(128),
  status varchar(32) not null,
  source_type varchar(32) not null,
  created_at timestamptz not null default now(),
  completed_at timestamptz
);
```

`upload_session` 用于记录上传请求和去重命中结果，`storage_object` 是 Upload 与 Git Builder 共用的对象中心登记表：

- Upload 完成接口会把对象登记为 `source_type = 'UPLOAD'`
- Git Builder 会把 Git 快照文件登记为 `source_type = 'GIT'`
- Upload 类型的 `Release verify` 会继续依赖该表，确保 `release_object.object_hash` 都已经完成登记
- Git Builder 则直接复用该表支撑 `materialization-plan`、节点预取与本地物化

### 2.9 node / operation / approval_record / audit_log

```sql
create table node (
  node_id varchar(64) primary key,
  role varchar(32) not null,
  region varchar(64) not null,
  lifecycle_status varchar(32) not null,
  health_status varchar(32) not null,
  daemon_version varchar(64) not null,
  heartbeat_at timestamptz,
  created_at timestamptz not null default now()
);

create table operation (
  operation_id varchar(64) primary key,
  type varchar(32) not null,
  status varchar(32) not null,
  target_resource_type varchar(32) not null,
  target_resource_id varchar(64) not null,
  progress integer not null default 0,
  error_code varchar(64),
  started_at timestamptz,
  updated_at timestamptz not null default now(),
  summary varchar(512),
  retry_count integer not null default 0,
  created_at timestamptz not null default now(),
  finished_at timestamptz
);

create table approval_record (
  approval_id varchar(64) primary key,
  target_type varchar(32) not null,
  target_id varchar(64) not null,
  environment varchar(32) not null,
  decision varchar(32) not null,
  approver_role varchar(64) not null,
  approved_at timestamptz
);

create table audit_log (
  audit_id varchar(64) primary key,
  resource_type varchar(32) not null,
  resource_id varchar(64) not null,
  action varchar(64) not null,
  status varchar(32) not null,
  operator varchar(128) not null,
  request_id varchar(64),
  operation_id varchar(64),
  detail_summary varchar(512),
  created_at timestamptz not null default now()
);
```

## 3. 推荐索引

```sql
create index idx_release_app_channel_created
  on release_record(app_id, channel, created_at desc);

create index idx_manifest_created
  on manifest(created_at desc);

create index idx_release_status_created
  on release_record(status, created_at desc);

create index idx_release_app_env_stable_activated
  on release_record(app_id, environment, stable, activated_at desc);

create index idx_release_object_hash
  on release_object(object_hash);

create index idx_upload_session_object_hash
  on upload_session(object_hash);

create index idx_upload_session_status_created
  on upload_session(status, created_at desc);

create unique index uq_storage_object_key
  on storage_object(object_key);

create index idx_storage_object_status_completed
  on storage_object(status, completed_at desc);

create index idx_node_region_role_heartbeat
  on node(region, role, heartbeat_at desc);

create index idx_operation_target_created
  on operation(target_resource_type, target_resource_id, created_at desc);

create index idx_audit_log_resource_created
  on audit_log(resource_type, resource_id, created_at desc);
```

## 4. SQLite 本地表示例

### 4.1 local_release

```sql
create table local_release (
  release_id text primary key,
  app_id text not null,
  environment text not null,
  status text not null,
  manifest_digest text not null,
  materialized_root text not null,
  last_access_at text
);
```

### 4.2 local_object_cache

```sql
create table local_object_cache (
  object_hash text primary key,
  cache_path text not null,
  storage_encoding text not null,
  size_bytes integer not null,
  storage_size_bytes integer not null,
  last_hit_at text,
  pin_state integer not null default 0
);
```

### 4.3 writeback_task

```sql
create table writeback_task (
  task_id text primary key,
  app_id text not null,
  path text not null,
  operation text not null,
  status text not null,
  retry_count integer not null default 0,
  expected_version integer,
  last_error_code text,
  updated_at text
);
```

推荐索引：

```sql
create index idx_local_release_app_env_status
  on local_release(app_id, environment, status);

create index idx_local_object_cache_last_hit
  on local_object_cache(last_hit_at desc);

create index idx_writeback_task_status_updated
  on writeback_task(status, updated_at);
```

## 5. 正式要求

- 中心库与节点本地库都必须有显式 schema 与索引
- DDL 示例必须与数据模型文档保持一致
- 实现时允许字段扩展，但不得破坏核心唯一约束、索引与引用关系

