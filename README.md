# SdkworkFS

SdkworkFS 是一个面向多语言应用分发、版本切换、Git 托管、对象存储和跨节点运行的文件系统平台。自本轮架构重设开始，**整个系统的目标实现基线统一切换为全 Rust**。

需要特别说明：

- 后续设计、模块边界、API、部署方式和实现路径，**以 `/docs/架构` 中的 Rust 文档为唯一基线**。
- 本仓库目标是落地 Rust workspace。
- 架构文档只维护当前 Rust 基线。

## 目标能力

SdkworkFS 的目标能力包括：

- 支持 React、Node.js、Java、Python、静态资源、二进制应用的统一发布模型
- 支持 Git、S3 标准对象存储、本地上传三类入口统一收敛为 `Release + Manifest`
- 支持完整 Git server 能力，包括 Smart HTTP、Git over SSH、镜像、双向同步、hook 和 push 触发发布
- 支持本地真实目录物化、挂载访问、快速切换、快速回滚
- 支持多级缓存，包括内存缓存、对象磁盘缓存、物化目录缓存
- 支持多机器分布式集群、节点编排、异步任务、审计与可观测性

## Rust 目标架构

目标实现采用单仓库、多二进制、多 crate 的 Rust workspace：

- `sdkworkfs-gateway`
  - 统一入口，承载管理 API、运行时 API、Git HTTP 入口、认证、限流、幂等
- `sdkworkfs-control`
  - 控制面核心，负责资源状态、发布编排、策略、审计、查询投影
- `sdkworkfs-job-runner`
  - 异步任务执行器，负责同步、镜像、构建、发布、激活、回滚、GC
- `sdkworkfs-git-server`
  - Git 服务，负责 Smart HTTP、SSH、receive-pack、upload-pack、LFS、push 收据
- `sdkworkfsd`
  - 节点守护进程，负责挂载、缓存、物化、切换、回源、写回
- `sdkworkfs`
  - CLI，负责初始化、发布、诊断、运维和本地控制

共享能力收敛到 `crates/` 下的共享库，例如：

- `sdkworkfs-core`
- `sdkworkfs-config`
- `sdkworkfs-contracts`
- `sdkworkfs-domain`
- `sdkworkfs-persistence-pg`
- `sdkworkfs-persistence-sqlite`
- `sdkworkfs-orchestrator`
- `sdkworkfs-scheduler`
- `sdkworkfs-object`
- `sdkworkfs-git`
- `sdkworkfs-process`
- `sdkworkfs-build-core`
- `sdkworkfs-build-nodejs`
- `sdkworkfs-build-java`
- `sdkworkfs-build-python`
- `sdkworkfs-cache`
- `sdkworkfs-materialize`
- `sdkworkfs-mount`
- `sdkworkfs-security`
- `sdkworkfs-observability`

## 关键边界

为避免后续实现再次出现“控制面、Git、任务执行、节点运行时”混写，当前架构明确采用以下边界：

- `sdkworkfs-gateway`
  - 只负责入口协议、认证、限流、幂等和路由，不拥有业务真相。
- `sdkworkfs-control`
  - 负责中心元数据、策略、审批、发布状态机和目标态，是平台业务真相拥有者。
- `sdkworkfs-job-runner`
  - 只负责长任务执行与重试，拥有执行尝试和 worker lease，不直接拥有发布真相。
- `sdkworkfs-git-server`
  - 负责 Git 协议、受管裸仓库、push 收据和 LFS 鉴权，不直接写控制面业务表。
- `sdkworkfsd`
  - 负责节点本地缓存、物化目录、`current/previous` 切换指针、写回队列和本地恢复。

同时明确三类内容分仓治理：

- Git 对象存放在受管 bare repo 中。
- Git LFS 大对象存放在 Git LFS 专用对象前缀中。
- Release 运行时对象存放在 Release 对象存储前缀中。

三者可以共用底层 S3 服务，但逻辑命名空间、权限和生命周期策略必须隔离。

## API 分层

外部与内部接口统一按职责拆分：

- `/admin/v1/*`
  - 管理 API，面向运维、控制台、CI 管理端
- `/runtime/v1/*`
  - 运行时 API，面向上传、Manifest、Release 消费、运行时查询
- `/node/v1/*`
  - 节点 API，面向 `sdkworkfsd`
- `/git/*`
  - Git Smart HTTP
- `SSH`
  - Git over SSH
- `/internal/v1/*`
  - 服务间内部接口

详细接口规划见：

- [`/docs/架构/15-接口与事件模型.md`](docs/架构/15-接口与事件模型.md)
- [`/docs/架构/29-控制面Gateway API详细设计.md`](docs/架构/29-控制面Gateway API详细设计.md)
- [`/docs/架构/78-Rust版Gateway与管理-非管理API规划.md`](docs/架构/78-Rust版Gateway与管理-非管理API规划.md)

## 技术底座

当前推荐的 Rust 技术基线：

- HTTP / 网关：`axum` + `tower-http`
- 异步运行时：`tokio`
- OpenAPI：`utoipa`
- 数据库：`sqlx`
- 内部 RPC：`tonic`
- 事件总线：`async-nats`
- Git：`gitoxide` 生态，以 `gix` 为应用主入口
- Git SSH：`russh`
- 对象存储：`aws-sdk-s3`
- 缓存：`moka`
- 挂载：Linux 优先 `fuser`，其余平台走 Rust 适配层
- 可观测性：`tracing` + `opentelemetry`

详细说明见：

- [`/docs/架构/75-Rust技术底座与开源集成策略.md`](docs/架构/75-Rust技术底座与开源集成策略.md)
- [`/docs/架构/76-Rust版Git-Server完整能力设计.md`](docs/架构/76-Rust版Git-Server完整能力设计.md)
- [`/docs/架构/77-Rust版模块与依赖选型矩阵.md`](docs/架构/77-Rust版模块与依赖选型矩阵.md)

## 文档阅读顺序

建议按以下顺序阅读架构文档：

1. [`/docs/架构/00-总体规划.md`](docs/架构/00-总体规划.md)
2. [`/docs/架构/02-技术架构总览.md`](docs/架构/02-技术架构总览.md)
3. [`/docs/架构/33-模块拆分与代码落地映射.md`](docs/架构/33-模块拆分与代码落地映射.md)
4. [`/docs/架构/79-Rust版服务边界与数据归属规范.md`](docs/架构/79-Rust版服务边界与数据归属规范.md)
5. [`/docs/架构/80-Rust版限界上下文与crate细化规范.md`](docs/架构/80-Rust版限界上下文与crate细化规范.md)
6. [`/docs/架构/81-Rust版代码布局与端口适配器规范.md`](docs/架构/81-Rust版代码布局与端口适配器规范.md)
7. [`/docs/架构/75-Rust技术底座与开源集成策略.md`](docs/架构/75-Rust技术底座与开源集成策略.md)
8. [`/docs/架构/76-Rust版Git-Server完整能力设计.md`](docs/架构/76-Rust版Git-Server完整能力设计.md)
9. [`/docs/架构/15-接口与事件模型.md`](docs/架构/15-接口与事件模型.md)
10. [`/docs/架构/29-控制面Gateway API详细设计.md`](docs/架构/29-控制面Gateway API详细设计.md)
11. [`/docs/架构/38-节点守护进程与挂载同步详细设计.md`](docs/架构/38-节点守护进程与挂载同步详细设计.md)

## 当前文档状态

当前文档已统一收敛到全 Rust 基线。后续实现、分阶段计划和代码落地都应以现有 Rust 服务边界、crate 拆分和 API 设计为准。

## Documentation Canon

- [docs/README.md](docs/README.md)
- [docs/product/prd/PRD.md](docs/product/prd/PRD.md)
- [docs/architecture/tech/TECH_ARCHITECTURE.md](docs/architecture/tech/TECH_ARCHITECTURE.md)

