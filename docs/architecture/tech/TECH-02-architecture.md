> Migrated from `docs/架构/02-技术架构总览.md` on 2026-06-24.
> Owner: SDKWork maintainers

# 技术架构总览

## 1. 技术选型原则

- 全部核心服务、共享库和 CLI 统一使用 Rust。
- 入口层、控制面、任务执行面、Git 面、节点运行面必须拆开。
- 运行时热路径优先走本地真实目录与缓存，不走中心控制面。
- Git、Upload、对象存储三类来源统一沉淀为 `Release + Manifest`。
- 长任务不进入同步请求路径，统一转为 `Operation + Attempt`。
- 复用成熟 Rust 生态，不自行重写 Git pack、SSH、S3 协议细节。
- 代码落地统一遵循 ports/adapters 分层，禁止 handler 直连数据库或对象存储。

## 2. 目标二进制

| 二进制 | 所属平面 | 写入权边界 | 核心职责 |
| --- | --- | --- | --- |
| `sdkworkfs-gateway` | Access Plane | 不拥有业务真相 | 管理 API、运行时 API、Git HTTP 路由、认证、限流、幂等、OpenAPI |
| `sdkworkfs-control` | Control Plane | 中心元数据与目标态 | Repository/Application/Release/Policy/Node/Operation 聚合、审批、审计、查询投影 |
| `sdkworkfs-job-runner` | Execution Plane | 执行尝试与 worker lease | Git 同步、镜像、构建、发布、激活、回滚、GC、重试与补偿 |
| `sdkworkfs-git-server` | Git Plane | 受管 bare repo 与 push receipt outbox | Smart HTTP、SSH、receive-pack、upload-pack、LFS 鉴权、策略校验、push 收据落盘 |
| `sdkworkfsd` | Node Plane | 节点本地状态 | 缓存、物化、挂载、同步、切换、写回、节点恢复 |
| `sdkworkfs` | Operator Plane | 本地命令上下文 | 初始化、诊断、运维、发布辅助、节点本地控制 |

## 3. 共享 crate

| crate | 职责 | 说明 |
| --- | --- | --- |
| `sdkworkfs-core` | 基础类型、错误模型、ID、时间、工具 | 不含业务基础设施依赖 |
| `sdkworkfs-config` | 配置模型、默认目录、环境映射、配置校验 | 统一控制默认值与多环境覆盖 |
| `sdkworkfs-contracts` | REST DTO、gRPC/Protobuf、事件契约、OpenAPI schema | 只承载契约，不承载实现 |
| `sdkworkfs-domain` | Repository、Application、Release、Node、Operation 状态机 | 平台统一业务模型 |
| `sdkworkfs-persistence-pg` | PostgreSQL 访问、迁移、仓储实现 | control / runner 的中心持久化 |
| `sdkworkfs-persistence-sqlite` | SQLite 访问、迁移、仓储实现 | git-server / daemon 的本地持久化 |
| `sdkworkfs-orchestrator` | 发布、激活、回滚、同步等编排规则 | 只定义编排，不直接跑重任务 |
| `sdkworkfs-scheduler` | 去重、队列、租约、超时、补偿策略 | 供 control 与 runner 共享 |
| `sdkworkfs-object` | S3 访问、预签名、对象登记、完整性校验 | Release 对象与 Git LFS 逻辑隔离 |
| `sdkworkfs-git` | 仓库访问、refs、mirror、hook、LFS 适配 | git-server 和 job-runner 共用 |
| `sdkworkfs-process` | 外部命令执行、超时、日志收集 | 构建工具链统一进程适配 |
| `sdkworkfs-build-core` | 构建任务协议、产物抽象、公共步骤 | 各语言构建共用能力 |
| `sdkworkfs-build-nodejs` | Node.js/React 构建适配器 | Node 生态专属依赖 |
| `sdkworkfs-build-java` | Maven/Gradle 构建适配器 | Java 生态专属依赖 |
| `sdkworkfs-build-python` | Python/uv/pip/poetry 构建适配器 | Python 生态专属依赖 |
| `sdkworkfs-cache` | L1/L2/L3 缓存、预热、淘汰、索引 | 节点运行时共享能力 |
| `sdkworkfs-materialize` | Manifest 物化、校验、原子切换辅助 | 只处理文件与目录级逻辑 |
| `sdkworkfs-mount` | FUSE 或平台适配挂载层 | 保持薄封装 |
| `sdkworkfs-security` | 鉴权、授权、令牌、密钥与凭据治理 | 隔离具体安全库实现 |
| `sdkworkfs-observability` | tracing、metrics、health、diagnostics | 全服务统一观测出口 |

## 4. 真相归属

| 数据/状态 | 真相拥有者 | 其他服务如何使用 |
| --- | --- | --- |
| 中心资源元数据、策略、审批、目标态 | `sdkworkfs-control` | 通过内部 API、投影或事件读取 |
| 受管 bare repo、Git refs、push 收据 outbox | `sdkworkfs-git-server` | control 通过收据摄取更新投影 |
| 执行尝试、worker lease、重试计数 | `sdkworkfs-job-runner` | control 仅读结果回报，不直接执行 |
| 节点本地缓存索引、`current/previous`、写回队列 | `sdkworkfsd` | control 只接收状态报告 |
| Release 对象二进制内容 | `sdkworkfs-object` + S3 | 节点通过 ticket/plan 读取 |
| Git LFS 二进制内容 | `sdkworkfs-git-server` + `sdkworkfs-object` | 仅 Git LFS 流程访问 |

完整约束见 `79-Rust版服务边界与数据归属规范.md`，内部上下文与 crate 细化规则见 `80-Rust版限界上下文与crate细化规范.md`，代码布局规则见 `81-Rust版代码布局与端口适配器规范.md`。

## 5. 外部基础设施

| 组件 | 作用 | 说明 |
| --- | --- | --- |
| PostgreSQL | 中心元数据存储 | 强事务、聚合状态机、审计、投影 |
| SQLite | git-server 与节点本地状态 | push receipt outbox、节点缓存索引、恢复信息 |
| S3 API 标准对象存储 | Release 对象与 Git LFS 内容 | 逻辑命名空间必须隔离 |
| NATS JetStream | 事件与异步命令总线 | `Operation` 调度、事件广播、节点通知 |
| TLS / mTLS | 安全传输 | 外部访问、服务间通信、节点认证 |

## 6. 核心开源依赖基线

- HTTP：`axum` + `tower-http`
- Async runtime：`tokio`
- OpenAPI：`utoipa`
- gRPC：`tonic`
- DB：`sqlx`
- Event bus：`async-nats`
- Git：`gitoxide` 生态，以 `gix` 为集成入口
- Git SSH：`russh`
- S3：`aws-sdk-s3`
- Cache：`moka`
- Mount：`fuser`
- Observability：`tracing` + `opentelemetry`

## 7. 关键链路

### 7.1 Git Push 到发布

```text
git push
-> sdkworkfs-git-server 鉴权与保护规则校验
-> receive-pack 更新受管 bare repo
-> 写入 push receipt outbox
-> 异步摄取到 sdkworkfs-control
-> control 创建 release candidate / operation
-> sdkworkfs-job-runner 执行构建、校验、发布
-> sdkworkfsd 预热、物化、原子切换
```

### 7.2 Upload / Import 到发布

```text
upload/import
-> sdkworkfs-gateway
-> sdkworkfs-control 创建 upload/import 会话
-> sdkworkfs-object 写入 Release 对象命名空间
-> control 生成 manifest / release
-> job-runner 执行 verify / publish / activate
```

### 7.3 节点运行时读取

```text
业务进程读取文件
-> sdkworkfsd 先查 L3 真实目录
-> miss 时查 L2 对象缓存
-> miss 时回源 S3
-> 回填 L2 并物化到 L3
-> 业务进程始终只读本地真实目录
```

## 8. 部署拓扑

### 8.1 单区域标准部署

```text
User / CI / Console
-> sdkworkfs-gateway
-> sdkworkfs-control
-> sdkworkfs-job-runner
-> sdkworkfs-git-server
-> PostgreSQL + NATS + S3
-> sdkworkfsd nodes
```

### 8.2 多节点 Git 集群

- 每个仓库同一时刻只允许一个写主 Git 节点。
- `sdkworkfs-control` 维护仓库放置与主节点归属。
- HTTP Git 流量由 gateway 按仓库归属路由到主节点。
- SSH Git 流量由前置 SSH 入口按仓库归属转发。
- 镜像复制与双向同步由 `sdkworkfs-job-runner` 异步执行，不阻塞 push 返回。

## 9. 跨平台策略

- Linux
  - 同步模式与挂载模式都必须支持。
- macOS
  - 同步模式优先，挂载模式后续增强。
- Windows
  - 优先保障同步模式、物化目录、快速切换与回滚。
  - 挂载能力通过独立适配层逐步补齐，不得影响主链路。

## 10. 当前结论

从当前基线开始，`sdkworkfs` 的推荐架构是：

- 全 Rust
- 多二进制
- 多 crate
- Git server 与文件系统运行面并列为一等能力
- API 按管理、运行时、节点、Git、内部五类拆分
- 真相归属按 control、git-server、job-runner、daemon 明确分治

