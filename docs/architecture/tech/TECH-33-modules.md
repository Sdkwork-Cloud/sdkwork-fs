> Migrated from `docs/架构/33-模块拆分与代码落地映射.md` on 2026-06-24.
> Owner: SDKWork maintainers

# 模块拆分与代码落地映射

## 1. 目标

建立一套一眼能看懂职责和写入权边界的 Rust workspace 结构，避免控制面、Git、缓存、构建、挂载、节点逻辑再次混写。

## 2. 顶层目录

```text
sdkwork-fs/
  apps/
    sdkworkfs-gateway/
    sdkworkfs-control/
    sdkworkfs-job-runner/
    sdkworkfs-git-server/
    sdkworkfsd/
    sdkworkfs/
  crates/
    sdkworkfs-core/
    sdkworkfs-config/
    sdkworkfs-contracts/
    sdkworkfs-domain/
    sdkworkfs-persistence-pg/
    sdkworkfs-persistence-sqlite/
    sdkworkfs-orchestrator/
    sdkworkfs-scheduler/
    sdkworkfs-object/
    sdkworkfs-git/
    sdkworkfs-process/
    sdkworkfs-build-core/
    sdkworkfs-build-nodejs/
    sdkworkfs-build-java/
    sdkworkfs-build-python/
    sdkworkfs-cache/
    sdkworkfs-materialize/
    sdkworkfs-mount/
    sdkworkfs-security/
    sdkworkfs-observability/
  deploy/
  scripts/
  docs/
```

## 3. 二进制职责

### 3.1 `apps/sdkworkfs-gateway`

职责：

- 对外 HTTP 入口
- OpenAPI 与 SDK 暴露面
- 认证、限流、幂等、审计上下文注入
- `/git/*` 到 `sdkworkfs-git-server` 的 HTTP 路由

禁止承担：

- 长任务执行
- 中心业务状态持久化
- 节点文件操作
- Git 策略真相维护

### 3.2 `apps/sdkworkfs-control`

职责：

- 资源 CRUD
- 发布、回滚、激活状态机
- 审批、策略、配额、审计
- 查询投影与仓库归属分配
- `Operation` 聚合与目标态管理

禁止承担：

- Git pack 处理
- 语言构建执行
- 节点本地文件读写

### 3.3 `apps/sdkworkfs-job-runner`

职责：

- 执行同步、镜像、双向同步、构建、发布、激活、GC
- 处理重试、去重、租约、超时和补偿
- 维护 `operation_attempt` 与 `worker_lease`

禁止承担：

- 管理 API 对外暴露
- 修改控制面聚合真相
- 直接接收 Git 客户端协议流量

### 3.4 `apps/sdkworkfs-git-server`

职责：

- Git Smart HTTP
- Git over SSH
- receive-pack / upload-pack
- 受管 bare repo 读写
- LFS 鉴权与 batch 协议
- push 收据落盘和异步上报

禁止承担：

- 发布审批与发布状态机
- 长时间镜像重试编排
- 直接写控制面业务表

说明：

- 镜像和双向同步是平台 Git 能力的一部分，但其长任务执行权属于 `sdkworkfs-job-runner`，`sdkworkfs-git-server` 只负责协议入口、策略校验和仓库读写能力。

### 3.5 `apps/sdkworkfsd`

职责：

- 节点注册、心跳、任务拉取
- 本地缓存、物化、切换、挂载、同步、写回
- 节点本地 SQLite 状态恢复

禁止承担：

- 直接修改中心元数据真相
- 直接访问控制面数据库

### 3.6 `apps/sdkworkfs`

职责：

- 运维命令
- 发布命令
- 节点本地控制
- 诊断与日志导出

## 4. 共享 crate 职责

### 4.1 纯领域

- `sdkworkfs-core`
- `sdkworkfs-domain`
- `sdkworkfs-contracts`

### 4.2 基础设施

- `sdkworkfs-config`
- `sdkworkfs-persistence-pg`
- `sdkworkfs-persistence-sqlite`
- `sdkworkfs-object`
- `sdkworkfs-git`
- `sdkworkfs-security`
- `sdkworkfs-observability`
- `sdkworkfs-process`

### 4.3 业务执行

- `sdkworkfs-orchestrator`
- `sdkworkfs-scheduler`
- `sdkworkfs-build-core`
- `sdkworkfs-build-nodejs`
- `sdkworkfs-build-java`
- `sdkworkfs-build-python`
- `sdkworkfs-cache`
- `sdkworkfs-materialize`
- `sdkworkfs-mount`

### 4.4 建议的 crate 内部子模块

- `sdkworkfs-build-core`
  - `adapters/nodejs`
  - `adapters/java`
  - `adapters/python`
  - `artifact-select`
- `sdkworkfs-git`
  - `repo-store`
  - `policy-check`
  - `protocol`
  - `mirror`
  - `lfs`
- `sdkworkfs-scheduler`
  - `operation-dispatch`
  - `attempt-store`
  - `lease`
  - `retry-policy`

## 5. 数据归属矩阵

| 状态/数据 | 真相拥有者 | 只读投影/消费者 |
| --- | --- | --- |
| Repository/Application/Release/Policy/Node/Quota | `sdkworkfs-control` | gateway、job-runner、git-server、daemon |
| Git refs、pack objects、受管 bare repo 目录 | `sdkworkfs-git-server` | job-runner 以只读 clone/fetch 使用 |
| Git push 收据 outbox | `sdkworkfs-git-server` | control 通过摄取接口消费 |
| Operation 聚合状态 | `sdkworkfs-control` | job-runner 报告结果，gateway 查询 |
| Operation attempt / worker lease | `sdkworkfs-job-runner` | control 仅读结果 |
| 节点本地缓存索引、切换指针、写回队列 | `sdkworkfsd` | control 只看状态报告 |

完整规则见 `79-Rust版服务边界与数据归属规范.md`。

## 6. 依赖方向

必须遵循：

- `apps/*` 只依赖 `crates/*`，不相互直接依赖实现细节。
- `sdkworkfs-domain` 不依赖任何外部基础设施 crate。
- `sdkworkfs-contracts` 不依赖数据库实现。
- `sdkworkfs-gateway` 不直接依赖 `sdkworkfs-cache`、`sdkworkfs-materialize`、`sdkworkfs-mount`。
- `sdkworkfsd` 不依赖控制面数据库实现。
- `sdkworkfs-git-server` 不依赖发布状态机实现。
- `sdkworkfs-job-runner` 不直接写 Git 协议入口状态。
- `sdkworkfs-control` 与 `sdkworkfs-job-runner` 不依赖 `sdkworkfs-persistence-sqlite`。
- `sdkworkfs-git-server` 与 `sdkworkfsd` 不依赖 `sdkworkfs-persistence-pg`。

推荐依赖方向：

```text
gateway -> contracts, security, observability
control -> domain, persistence-pg, orchestrator, scheduler, contracts, security
job-runner -> orchestrator, scheduler, object, git, process, build-core, build-nodejs, build-java, build-python, materialize, observability, persistence-pg
git-server -> git, security, contracts, observability, persistence-sqlite
sdkworkfsd -> cache, materialize, mount, object, contracts, persistence-sqlite
cli -> contracts, config, security
```

## 7. 反耦合硬约束

- 不允许 `sdkworkfs-git-server` 直接更新 `Release`、`Operation`、`Policy` 中心表。
- 不允许 `sdkworkfs-control` 直接操作 bare repo 文件或节点物化目录。
- 不允许 `sdkworkfs-job-runner` 持有管理 API 对外协议层。
- 不允许 `sdkworkfsd` 绕过控制面协议直接篡改中心状态。
- 不允许把 Git server、缓存、控制面、挂载逻辑堆进一个大 crate。

## 8. 与文档的映射

| 文档 | 主要落地模块 |
| --- | --- |
| `15-接口与事件模型.md` | `sdkworkfs-contracts`, `sdkworkfs-gateway`, `sdkworkfs-control` |
| `29-控制面Gateway API详细设计.md` | `sdkworkfs-gateway`, `sdkworkfs-control` |
| `34-Git Gateway与仓库管理详细设计.md` | `sdkworkfs-git-server`, `sdkworkfs-git`, `sdkworkfs-control`, `sdkworkfs-job-runner` |
| `38-节点守护进程与挂载同步详细设计.md` | `sdkworkfsd`, `sdkworkfs-cache`, `sdkworkfs-materialize`, `sdkworkfs-mount` |
| `79-Rust版服务边界与数据归属规范.md` | 全部二进制与核心 crate |
| `80-Rust版限界上下文与crate细化规范.md` | `sdkworkfs-control`, `sdkworkfs-job-runner`, `sdkworkfs-git-server`, `sdkworkfsd` 内部模块划分 |
| `81-Rust版代码布局与端口适配器规范.md` | 各二进制与共享 crate 的目录模板、层次与依赖规则 |

## 9. 二进制内部目录模板

推荐每个 `apps/*` 二进制采用统一模板：

```text
apps/<binary>/
  src/
    main.rs
    bootstrap/
    api/
    contexts/
    workers/           # 仅 job-runner
    adapters/
    diagnostics/
```

说明：

- `main.rs`
  - 只做启动与 wiring。
- `bootstrap/`
  - 装配配置、日志、路由、依赖注入。
- `api/`
  - 只放 HTTP/gRPC/SSH/FUSE 等协议层。
- `contexts/`
  - 业务上下文入口，内部再拆 `commands`、`queries`、`ports`。
- `adapters/`
  - PostgreSQL、SQLite、S3、Git、本地文件系统等实现。
- `workers/`
  - 只允许出现在 `sdkworkfs-job-runner`。

## 10. Rust workspace 落地要求

- Rust workspace 落到 `apps/` 与 `crates/`
- 服务目录、共享 crate、部署资产必须按职责直接映射
- 工程结构不得再引入与当前服务边界无关的遗留层次

## 11. 正式要求

- 模块命名必须直接反映职责。
- 一个模块只承担一种主要责任。
- 写入权边界必须先于代码目录结构确定。
- 长任务执行权和同步协议入口权必须分离。
- 代码目录必须体现协议层、应用层、端口层、适配器层。

