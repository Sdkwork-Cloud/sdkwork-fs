> Migrated from `docs/架构/77-Rust版模块与依赖选型矩阵.md` on 2026-06-24.
> Owner: SDKWork maintainers

# Rust 版模块与依赖选型矩阵

## 1. 目标

把目标二进制、共享 crate 和主要开源依赖矩阵化，并进一步消除“过宽 crate”风险，便于后续建立 Cargo workspace 与分阶段实施计划。

## 2. 二进制矩阵

| 模块 | 主要职责 | 关键依赖 |
| --- | --- | --- |
| `sdkworkfs-gateway` | REST、OpenAPI、认证、限流、Git HTTP 路由 | `axum`, `tower-http`, `utoipa`, `rustls` |
| `sdkworkfs-control` | 资源状态、策略、查询投影、仓库放置 | `sqlx`, `tonic`, `async-nats` |
| `sdkworkfs-job-runner` | 构建、发布、激活、回滚、镜像、GC | `tokio`, `async-nats`, `gix`, `aws-sdk-s3` |
| `sdkworkfs-git-server` | Smart HTTP、SSH、LFS、push receipt | `gix`, `russh`, `axum` |
| `sdkworkfsd` | 缓存、物化、切换、挂载 | `moka`, `fuser`, `aws-sdk-s3`, `notify` |
| `sdkworkfs` | CLI、诊断、运维 | `clap`, `reqwest` |

## 3. 共享 crate 矩阵

### 3.1 基础与契约

| crate | 主要职责 | 不应依赖 |
| --- | --- | --- |
| `sdkworkfs-core` | 基础工具、错误模型、ID、时间 | DB、HTTP、S3 |
| `sdkworkfs-config` | 配置模型与默认值 | HTTP handler |
| `sdkworkfs-contracts` | REST DTO、gRPC、事件契约 | DB 实现 |
| `sdkworkfs-domain` | 领域模型和状态机 | `sqlx`, `aws-sdk-s3` |
| `sdkworkfs-observability` | tracing、metrics、health | 业务领域 |
| `sdkworkfs-security` | 鉴权、授权、令牌、密钥治理 | UI、挂载逻辑 |

### 3.2 持久化

| crate | 主要职责 | 不应依赖 |
| --- | --- | --- |
| `sdkworkfs-persistence-pg` | control / runner 的 PostgreSQL 仓储 | daemon、本地挂载 |
| `sdkworkfs-persistence-sqlite` | git-server / daemon 的本地 SQLite 状态 | 网关、控制面状态机 |

说明：

- 不再推荐单个 `sdkworkfs-persistence` 同时承载 PostgreSQL 和 SQLite。
- 中心真相和本地状态必须通过不同 crate 隔离，避免错误依赖方向。

### 3.3 资源与执行

| crate | 主要职责 | 不应依赖 |
| --- | --- | --- |
| `sdkworkfs-object` | S3 抽象、预签名、对象校验 | Gateway handler |
| `sdkworkfs-git` | Git 抽象、repo-store、policy-check、LFS、mirror helper | OpenAPI、CLI UI |
| `sdkworkfs-orchestrator` | 发布、激活、回滚、同步编排 | HTTP handler |
| `sdkworkfs-scheduler` | 队列、去重、lease、retry-policy | Git 协议层 |
| `sdkworkfs-cache` | L1/L2/L3 缓存能力 | 控制面数据库 |
| `sdkworkfs-materialize` | 物化、校验、切换辅助 | OpenAPI |
| `sdkworkfs-mount` | 挂载适配 | 控制面状态机 |
| `sdkworkfs-process` | 外部命令执行、超时、日志收集、工具链探测 | REST DTO |

### 3.4 构建适配器

| crate | 主要职责 | 不应依赖 |
| --- | --- | --- |
| `sdkworkfs-build-core` | 构建任务协议、产物抽象、公共步骤 | Node/Java/Python 专属依赖 |
| `sdkworkfs-build-nodejs` | Node.js/React/Vite/PNPM/NPM/Yarn 构建适配 | Java、Python 工具链 |
| `sdkworkfs-build-java` | Maven/Gradle/JAR/WAR 构建适配 | Node、Python 工具链 |
| `sdkworkfs-build-python` | venv/pip/poetry/uv/打包适配 | Node、Java 工具链 |

说明：

- 不再推荐单个 `sdkworkfs-build` 承载所有语言适配器。
- 多语言构建工具链必须物理拆开，避免依赖爆炸和跨语言污染。

## 4. 推荐依赖组合

### 4.1 网关

- `axum`
- `tower`
- `tower-http`
- `rustls`
- `utoipa`

### 4.2 控制面

- `sqlx`
- `serde`
- `tonic`
- `async-nats`
- `casbin`

### 4.3 Git

- `gix`
- `gix-transport`
- `gix-protocol`
- `russh`

### 4.4 节点

- `moka`
- `fuser`
- `notify`
- `sha2`
- `zstd`

### 4.5 可观测性

- `tracing`
- `tracing-subscriber`
- `opentelemetry`
- `tracing-opentelemetry`

## 5. 依赖约束

- 共享 crate 必须先于二进制稳定。
- 所有第三方库只通过内部封装层向外暴露。
- 不允许各二进制直接各自实现一套 Git、S3、权限或缓存逻辑。
- `sdkworkfs-git-server` 与 `sdkworkfsd` 只能依赖 `sdkworkfs-persistence-sqlite`。
- `sdkworkfs-control` 与 `sdkworkfs-job-runner` 只能依赖 `sdkworkfs-persistence-pg`。
- 语言构建适配器只能通过 `sdkworkfs-build-core` 接入 runner。

## 6. 当前结论

后续建仓和编码时，推荐先落以下基础 crate：

1. `sdkworkfs-core`
2. `sdkworkfs-config`
3. `sdkworkfs-contracts`
4. `sdkworkfs-domain`
5. `sdkworkfs-observability`
6. `sdkworkfs-security`
7. `sdkworkfs-persistence-pg`
8. `sdkworkfs-persistence-sqlite`

随后再推进 `gateway`、`control`、`git-server`、`daemon`，最后接入各语言构建适配器。

