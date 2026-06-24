> Migrated from `docs/架构/72-对象缓存治理与构建幂等补充规范.md` on 2026-06-24.
> Owner: SDKWork maintainers

# 对象缓存治理与构建幂等补充规范

## 1. 目标

本规范补充对象缓存治理与构建幂等的 3 条关键控制能力：

- L2 对象磁盘缓存的水位治理与空闲回收
- `BUILD_RELEASE` 的数据库级幂等排队
- 构建命令执行的超时回收与日志尾部保留

目标是让控制面在多节点、长时间运行、跨平台构建场景下具备基本稳定性边界。

## 2. L2 对象磁盘缓存治理

### 2.1 配置项

建议使用以下配置：

- `sdkworkfs.cache.objectDisk.rootDir`
- `sdkworkfs.cache.objectDisk.maxBytes`
- `sdkworkfs.cache.objectDisk.highWatermarkPercent`
- `sdkworkfs.cache.objectDisk.lowWatermarkPercent`
- `sdkworkfs.cache.objectDisk.maxIdleSeconds`

默认值：

- `maxBytes = 200GB`
- `highWatermarkPercent = 85`
- `lowWatermarkPercent = 70`
- `maxIdleSeconds = 604800`

### 2.2 访问与回收流程

`NodeObjectCacheService.ensureCached(...)` 的执行顺序如下：

1. 根据 `objectHash` 解析对象路径。
2. 若命中且文件大小匹配，则刷新访问时间。
3. 先执行空闲过期清理。
4. 若缓存总量超过高水位，则按最近最少访问顺序淘汰到低水位。
5. 当前请求正在读取或刚下载的对象属于受保护对象，不能在同一轮 GC 中被回收。

### 2.3 实现边界

- 当前 L2 访问热度直接复用文件 `lastModified` 作为最近访问时间。
- 当前 GC 是“随访问触发”，尚未引入独立后台定时任务。
- 当前策略针对单节点本地磁盘缓存，不维护分布式缓存目录索引。

## 3. BUILD_RELEASE 数据库幂等

### 3.1 数据模型

`operation` 表新增：

- `dedupe_key varchar(128) null`
- 唯一索引 `uq_operation_dedupe_key`

由于 `dedupe_key` 可为空，因此普通操作不受影响；仅带幂等键的活动任务参与唯一约束。

### 3.2 幂等键规则

- 仅 `BUILD_RELEASE` 活跃任务使用幂等键。
- 统一格式：`build-release:<releaseId>`
- 创建构建任务时写入 `dedupe_key`
- 任务进入 `SUCCEEDED`、`FAILED`、`CANCELLED` 后清空 `dedupe_key`

这样可以满足“同一时刻只能有一个构建排队或运行，同时失败后允许重新构建”的要求。

### 3.3 集群语义

- `ApplicationService.buildRelease(...)` 统一经由 `OperationService.createPending(..., dedupeKey)` 创建构建任务。
- 数据库唯一索引是最终正确性边界，应用内存锁只作为本地优化。
- `RepositorySyncAutoReleaseService` 若遇到重复排队冲突，应记录为 `SKIPPED`，而不是 `FAILED`。

## 4. 构建命令执行韧性

### 4.1 日志策略

- 构建日志先落临时文件，避免子进程输出堵塞内存。
- 返回结果或异常时只读取日志尾部。
- 当前尾部上限为 `64KiB`。

### 4.2 超时处理

超时时执行以下步骤：

1. 先尝试终止整个进程树。
2. 在短暂宽限期内等待退出。
3. 对仍存活的进程执行强制终止。
4. 读取日志尾部并拼接进异常消息。

### 4.3 跨平台要求

- 不能依赖只对 Linux 有效的 kill 方式。
- 需要处理 Windows 下日志文件句柄尚未释放的场景。
- 临时日志文件使用 best-effort 删除，不允许因清理失败覆盖主业务异常。

## 5. 测试与验收

本规范对应的最小验收用例如下：

- `NodeObjectCacheServiceTest`
- `SdkworkFsPropertiesTest`
- `OperationServiceTest`
- `CoreSchemaMigrationTest`
- `RepositorySyncAutoReleaseServiceTest`
- `BuildWorkspaceCommandRunnerTest`

验收通过标准：

- 缓存命中可刷新访问时间
- 过期对象可被回收
- 超高水位可回收到低水位
- 相同 `releaseId` 的活跃构建不会重复排队
- 终态任务允许重新发起构建
- 构建超时异常包含日志尾部且不会因日志文件删除失败而掩盖主异常

