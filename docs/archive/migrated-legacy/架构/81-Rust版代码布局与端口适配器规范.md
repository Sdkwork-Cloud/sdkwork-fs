# Rust 版代码布局与端口适配器规范

## 1. 目标

把当前的服务拆分和 crate 拆分进一步落到代码目录与分层模板，确保后续建立 Cargo workspace 时不会再次出现 handler 直连 SQL、worker 直接改聚合、或上下文之间相互横跳的情况。

## 2. 总体分层

推荐每个 Rust 模块都按以下层次理解：

- `protocol`
  - HTTP、gRPC、SSH、FUSE、CLI 输入输出
- `application`
  - use case、command、query、任务编排入口
- `domain`
  - 纯规则、状态机、聚合行为
- `ports`
  - 对外部依赖的抽象接口
- `adapters`
  - PostgreSQL、SQLite、S3、Git、本地文件系统、消息总线实现

规则：

- `protocol` 只能调 `application`
- `application` 可以调 `domain` 和 `ports`
- `domain` 不得依赖 `protocol`、数据库、网络库
- `adapters` 只实现 `ports`，不得反向依赖 `protocol`

## 3. 二进制目录模板

### 3.1 `apps/sdkworkfs-gateway`

```text
src/
  main.rs
  bootstrap/
  api/
    admin/
    runtime/
    git_proxy/
  middleware/
  error/
```

规则：

- `api/*` 只做参数解析、鉴权上下文、错误映射、响应包装。
- 不允许直接引入 `sqlx`、`gix`、`aws-sdk-s3`。

### 3.2 `apps/sdkworkfs-control`

```text
src/
  main.rs
  bootstrap/
  api/
    admin/
    runtime/
    node/
    internal/
  contexts/
    repository_admin/
    application_admin/
    release_admin/
    node_admin/
    policy_admin/
    operation_admin/
    audit_query/
  adapters/
    pg/
    nats/
```

每个 `contexts/<name>/` 推荐包含：

```text
commands/
queries/
ports/
services/
read_models/
```

### 3.3 `apps/sdkworkfs-job-runner`

```text
src/
  main.rs
  bootstrap/
  dispatcher/
  workers/
    repository_sync/
    mirror/
    release_build/
    release_verify/
    activation/
    rollback/
    gc/
  adapters/
    pg/
    nats/
```

规则：

- `dispatcher/` 只负责取任务、租约、分发。
- `workers/*` 只负责某一类 operation attempt 执行。
- worker 之间不得直接互调内部实现。

### 3.4 `apps/sdkworkfs-git-server`

```text
src/
  main.rs
  bootstrap/
  http_front/
  ssh_front/
  protocol_engine/
  repo_store/
  policy_check/
  lfs_gateway/
  push_outbox/
  adapters/
    sqlite/
```

规则：

- `protocol_engine/` 只处理 Git 协议行为。
- `push_outbox/` 单独管理持久化与补偿上报。
- `mirror` 不允许作为 git-server 内部长期模块存在。

### 3.5 `apps/sdkworkfsd`

```text
src/
  main.rs
  bootstrap/
  node_agent/
  cache_manager/
  materializer/
  switcher/
  mount_adapter/
  writeback_agent/
  local_state/
  adapters/
    sqlite/
    fs/
```

规则：

- `node_agent/` 只处理与 control 的交互。
- `local_state/` 只负责 SQLite 和目录状态持久化。
- `mount_adapter/` 不得直接访问控制面接口。

## 4. 共享 crate 模板

推荐共享 crate 采用统一目录：

```text
src/
  lib.rs
  domain/
  ports/
  adapters/   # 仅基础设施 crate 允许
  tests/
```

具体要求：

- `sdkworkfs-domain`
  - 只能有 `domain/`，不应出现 `adapters/`
- `sdkworkfs-contracts`
  - 只放 DTO、schema、protobuf、事件载荷
- `sdkworkfs-object`、`sdkworkfs-git`
  - 可以有 `ports/` 和 `adapters/`
- `sdkworkfs-build-*`
  - 只暴露构建协议与产物结果，不暴露 HTTP 或 DB 细节

## 5. 端口命名规范

推荐统一命名：

- 查询端口：`*Reader`
- 写入端口：`*Writer`
- 仓储端口：`*Repository`
- 外部服务端口：`*Client`
- 任务派发端口：`*Dispatcher`
- 票据签发端口：`*TicketIssuer`

示例：

- `RepositoryPlacementReader`
- `ReleaseRepository`
- `ObjectStoreClient`
- `PushReceiptDispatcher`

## 6. 明确禁止的实现方式

- handler 直接写 SQL
- handler 直接访问 S3
- worker 直接改 `Release` 聚合主表
- git-server 直接生成候选 Release
- daemon 直接调用 PostgreSQL
- 用 `utils.rs`、`common.rs` 收纳跨上下文业务逻辑

## 7. 测试边界建议

- `domain` 层做纯单元测试
- `application` 层使用 port mock 做用例测试
- `adapters` 层做数据库、对象存储、Git 交互测试
- 二进制层做协议集成测试

## 8. 正式要求

- 每个模块都必须能一眼看出属于 `protocol`、`application`、`domain`、`ports`、`adapters` 的哪一层。
- 无法归类到这五层的代码，必须先重审设计再落目录。
- 后续生成 Cargo workspace 时，必须以本模板创建目录，而不是先随意建文件再回头收拾。
