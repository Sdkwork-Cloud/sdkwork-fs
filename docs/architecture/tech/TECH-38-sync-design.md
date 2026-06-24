> Migrated from `docs/架构/38-节点守护进程与挂载同步详细设计.md` on 2026-06-24.
> Owner: SDKWork maintainers

# 节点守护进程与挂载同步详细设计

## 1. 目标

`sdkworkfsd` 是运行时性能核心，负责：

- 节点注册与心跳
- 多级缓存
- 本地物化
- 版本切换
- 挂载模式
- 同步模式
- 回源对象存储
- 写回和冲突处理

它必须保证应用真正访问的是本地真实目录，而不是中心控制面或远端对象存储。

## 2. 进程内模块

### 2.1 `node-control`

- 注册
- 心跳
- 租约续租
- 配置拉取
- 任务拉取与上报

### 2.2 `node-cache`

- L1 内存缓存
- L2 对象磁盘缓存
- 命中统计
- 预热
- 淘汰

### 2.3 `node-materialize`

- 读取 Manifest
- 拉取对象
- 校验哈希
- 展开到真实目录
- 恢复权限与元数据

### 2.4 `node-switch`

- `current / previous` 管理
- 原子切换
- 回滚
- 切换状态持久化

### 2.5 `node-mount`

- 挂载入口
- FUSE 请求适配
- 路径解析
- 读写策略分流

### 2.6 `node-sync`

- 把目标 Release 同步到本地真实目录
- 管理增量更新
- 维护本地可见运行目录

### 2.7 `node-writeback`

- 收集可写目录变化
- 形成写回队列
- 幂等重试
- 冲突检测和副本隔离

## 3. 多级缓存模型

| 层级 | 介质 | 作用 |
| --- | --- | --- |
| `L1` | 内存 | 热点对象元数据、目录项、句柄、短期索引 |
| `L2` | 本地磁盘对象缓存 | 保存压缩或原始对象内容 |
| `L3` | 物化目录 | 应用实际读取的真实文件目录 |
| `L4` | 挂载/同步暴露层 | 面向应用的访问入口 |

缓存策略要求：

- 先查 L1
- 再查 L2
- L2 miss 才回源 S3
- 物化到 L3 后再对业务暴露
- `L4` 只能暴露 `L3` 中已经 ready 的目录

## 4. 目录结构

建议目录：

```text
<root>/
  cache/
    objects/
  materialized/
    <appId>/
      releases/
      current
      previous
  writable/
    <appId>/
      data/
      workspace/
      logs/
  metadata/
    tasks/
    switches/
    cache/
  state/
    node.db
```

## 5. 本地 SQLite 状态归属

`node.db` 至少包含以下表或等价结构：

| 表 | 说明 |
| --- | --- |
| `node_task_lease` | 当前持有的任务租约、`leaseEpoch`、过期时间 |
| `materialized_release` | 本地 release ready 状态、目录路径、校验摘要 |
| `switch_pointer` | `current`、`previous`、最后一次切换结果 |
| `cache_object_index` | L2 对象命中、大小、最后访问时间、引用计数 |
| `writeback_queue` | 待写回记录、重试次数、冲突状态 |

规则：

- 这些表只由 `sdkworkfsd` 写入。
- 控制面只能接收上报结果，不能直接改本地状态。

## 6. 读取流程

```text
应用请求路径
-> 解析到 current 指向的 release
-> 检查 L3 是否已有真实文件
-> 若不存在，则检查 L2 对象缓存
-> 若 L2 miss，则回源 S3
-> 校验 object hash
-> 写入 L2
-> 展开到 L3
-> 返回本地真实文件
```

## 7. 物化流程

```text
接收 materialization-plan
-> 创建目标 release 目录
-> 并发拉取对象
-> 校验大小与 hash
-> 解压/解码
-> 写入真实文件
-> fsync / 校验
-> 生成物化索引
-> 标记 release ready
```

## 8. 切换流程

```text
接收 activate/switch
-> 校验目标 release ready
-> current -> previous
-> target release -> current
-> 原子更新
-> 落盘 switch state
-> 上报结果
```

切换必须满足：

- 对应用可见的入口始终只有一个 `current`
- `previous` 始终可用于快速回滚
- 切换失败不得污染现有 `current`

## 9. 任务租约模型

节点任务必须使用租约而不是裸“抢任务”：

- `tasks:pull` 返回 `taskId`、`leaseId`、`leaseEpoch`、`leaseExpiresAt`
- 节点确认任务时必须带回 `leaseEpoch`
- 续租失败后不得继续执行副作用步骤
- lease 过期后其他节点才允许接管

推荐任务状态：

```text
PENDING -> LEASED -> RUNNING -> SUCCEEDED / FAILED / CANCELLED
```

推荐租约续期规则：

- 续租间隔小于 `leaseTtl / 3`
- 同一 `taskId` 只允许一个有效 `leaseEpoch`
- 完成上报必须携带 `taskId + leaseEpoch + attemptDigest`

## 10. 挂载模式与同步模式并存规则

### 10.1 默认策略

- 生产环境默认 `sync`
- `mount` 主要用于调试、透明路径访问和特定 POSIX 挂载场景

### 10.2 并存约束

- 同一 `appId + environment + node` 同一时刻只能有一个活动暴露模式。
- `mount` 不允许绕过 `current` 直接指向半物化目录。
- `sync` 目录始终是最终运行时真相。
- 若启用 `mount`，其底层内容仍优先读取 `L3/L2`，而不是控制面。

## 11. 写回策略

默认规则：

- `release/`、`bin/`、`static/` 等发布内容只读
- `data/`、`workspace/`、`logs/` 可写
- 写回通过后台队列处理
- 发生冲突时生成冲突副本并上报

补充要求：

- 写回永远不直接改写 Release 物化目录
- 写回目标必须是受策略允许的独立可写区域
- 写回失败不能阻塞当前版本继续运行

## 12. 崩溃恢复

守护进程重启后必须恢复：

- 当前 `current / previous` 指针
- release ready 状态
- 未完成写回任务
- 未完成预热任务
- 当前任务租约与续租信息
- 缓存索引和磁盘占用统计

## 13. 跨平台策略

### 13.1 Linux

- 第一优先平台
- 同步模式和挂载模式都要支持

### 13.2 macOS

- 先支持同步模式
- 挂载模式逐步增强

### 13.3 Windows

- 优先同步模式、物化目录和快速切换
- 挂载模式单独适配，不影响主链路

## 14. 正式要求

- `sdkworkfsd` 只能消费控制面下发的 Release/Manifest/Task，不直接篡改中心真相。
- 缓存、物化、切换、写回必须可审计、可恢复、可重试。
- 节点数据面必须独立于管理 API 存活。
- 任务执行必须基于 lease，不能使用无保护的“先到先得”模型。

