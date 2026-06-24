> Migrated from `docs/架构/79-Rust版服务边界与数据归属规范.md` on 2026-06-24.
> Owner: SDKWork maintainers

# Rust 版服务边界与数据归属规范

## 1. 目标

把“模块拆分”从目录层面进一步收敛为“写入权与真相归属”规则，避免实现阶段再次出现多个服务同时改同一份状态、多个服务同时拥有同一份磁盘目录、或长任务与协议入口混写的问题。

## 2. 平面划分

平台固定拆为五个平面：

- `Access Plane`
  - `sdkworkfs-gateway`
- `Control Plane`
  - `sdkworkfs-control`
- `Execution Plane`
  - `sdkworkfs-job-runner`
- `Git Plane`
  - `sdkworkfs-git-server`
- `Node Plane`
  - `sdkworkfsd`

每个平面只对自己的真相负责，其他平面只能通过 API、命令或事件访问。

## 3. 真相归属矩阵

| 数据/状态 | 真相拥有者 | 可写介质 | 其他平面的访问方式 |
| --- | --- | --- | --- |
| Repository/Application/Release/Policy/Node/Quota 聚合 | `sdkworkfs-control` | PostgreSQL | 管理 API、内部 gRPC、事件投影 |
| 仓库放置、Git 主节点归属 | `sdkworkfs-control` | PostgreSQL | git-server 配置快照 |
| bare repo、Git refs、pack objects | `sdkworkfs-git-server` | 本地文件系统 / 共享存储 | Git 协议、只读 clone/fetch |
| push receipt outbox | `sdkworkfs-git-server` | 本地 SQLite | 内部摄取接口 |
| Operation 聚合状态 | `sdkworkfs-control` | PostgreSQL | 管理 API 查询 |
| OperationAttempt / WorkerLease | `sdkworkfs-job-runner` | PostgreSQL | control 通过结果回报读取 |
| 节点本地缓存索引、物化状态、切换指针、写回队列 | `sdkworkfsd` | 本地 SQLite + 本地磁盘 | 节点状态上报 |
| Release 对象二进制 | `sdkworkfs-object` | S3 命名空间 | ticket/plan 消费 |
| Git LFS 二进制 | `sdkworkfs-git-server` + `sdkworkfs-object` | S3 命名空间 | Git LFS 协议 |

## 4. 写入规则

- 非真相拥有者不得直写真相存储。
- `sdkworkfs-git-server` 不得直接写控制面业务表。
- `sdkworkfs-job-runner` 不得直接改写 `Release`、`Policy`、`Node` 聚合。
- `sdkworkfs-control` 不得直接操作 bare repo 或节点物化目录。
- `sdkworkfsd` 不得绕过控制面 API 直接篡改中心元数据。

## 5. 命令、事件与查询规则

### 5.1 查询

适合：

- 配置快照获取
- 运行时只读查询
- 节点计划拉取

要求：

- 幂等
- 无副作用
- 可缓存

### 5.2 命令

适合：

- 触发构建、发布、镜像、物化、GC
- 摄取 push receipt
- 下发节点执行计划

要求：

- 指定唯一拥有者
- 带 `commandId`
- 可重试、可超时、可拒绝

### 5.3 事件

适合：

- 发布事实广播
- 节点执行完成广播
- 缓存回收完成广播
- Git push 接受/拒绝广播

要求：

- 至少一次投递
- 使用 `eventId` 去重
- 可重放

## 6. 关键链路的写入边界

### 6.1 Git Push

```text
git client
-> sdkworkfs-git-server 更新 bare repo
-> git-server 写 push receipt outbox
-> sdkworkfs-control 摄取 receipt 并更新 repository 投影
-> control 创建 operation
-> sdkworkfs-job-runner 执行后续构建/镜像任务
```

规则：

- Git push 的协议成功不依赖 control 同步写成功。
- 但 control 必须最终摄取 push receipt，才能产生候选 Release。

### 6.2 发布任务

```text
sdkworkfs-control 创建 Operation
-> sdkworkfs-job-runner 领取并执行 Attempt
-> job-runner 上报结果
-> control 决定 Operation 聚合状态
```

规则：

- control 拥有 Operation 聚合状态。
- runner 拥有 Attempt 与 lease。
- 两者不得写同一张“状态主表”。

### 6.3 节点切换

```text
control 发布目标态
-> sdkworkfsd 拉取计划并持有 task lease
-> daemon 本地物化与切换
-> daemon 上报结果
-> control 更新节点投影
```

规则：

- 节点切换真相在节点本地目录和本地 SQLite。
- control 只维护期望状态和结果投影。

## 7. 失效与恢复边界

- control 不可用时：
  - 既有节点应继续运行当前版本。
  - git-server 只能在拥有有效策略快照时接受写入。
- git-server 不可用时：
  - 只影响 Git 协议入口，不影响既有节点运行。
- job-runner 不可用时：
  - 只影响长任务推进，不影响当前已运行版本。
- daemon 不可用时：
  - 只影响所在节点，不得污染中心状态。

## 8. 反模式清单

- 一个服务既处理协议入口，又执行长时间重试任务。
- 多个服务同时直接更新同一份业务状态。
- 用共享数据库表代替服务边界。
- 用共享磁盘目录代替归属规则。
- 节点直接把本地状态回写为中心真相。

## 9. 正式要求

- 先定义写入权边界，再定义 crate 和目录结构。
- 任何新增服务或 crate，都必须先说明它拥有哪份真相。
- 没有真相归属说明的模块，不得进入正式架构。

