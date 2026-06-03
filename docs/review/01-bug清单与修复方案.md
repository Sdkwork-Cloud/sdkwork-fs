# Bug 清单与修复方案

## RV-001 应用可绑定不存在仓库

- 严重级别：P1
- 影响范围：控制面应用创建、后续构建与发布链路
- 根因：`ApplicationService.create(...)` 未校验仓库存在性。
- 修复方案：
  - 服务层显式加载 `RepositoryRecord`
  - 不存在时返回 `404 NOT_FOUND`
- 验证：
  - `ApplicationApiTest.testCreateApplicationRejectsMissingRepository`
- 状态：已修复

## RV-002 跨租户资源引用未拦截

- 严重级别：P1
- 影响范围：多租户隔离、安全边界、审计可信度
- 根因：服务层未校验请求租户与目标资源租户的一致性。
- 修复方案：
  - 应用创建时校验 `request.tenantId == repository.tenantId`
  - Release 创建时校验 `request.tenantId == application.tenantId`
  - 不一致时返回 `409 CONFLICT`
- 验证：
  - `ApplicationApiTest.testCreateApplicationRejectsCrossTenantRepository`
  - `ReleaseApiTest.testCreateReleaseRejectsCrossTenantApplication`
- 状态：已修复

## RV-003 自动发布 releaseId 在长应用 ID 下碰撞

- 严重级别：P1
- 影响范围：Git sync 自动候选发布、快速切换、版本追踪
- 根因：自动发布直接对 `releaseId` 做长度截断，丢失 commit 熵。
- 修复方案：
  - 引入 `AutoReleaseIdGenerator`
  - 短 ID 兼容旧格式
  - 长 ID 改为 `auto-<appPrefix>-<envPrefix>-<commit12>-<hash8>`
- 验证：
  - `RepositoryApiTest.testSyncRepositoryAutoBuildsDistinctReleasesForDifferentCommitsWithLongAppId`
  - `AutoReleaseIdGeneratorTest`
- 状态：已修复

## RV-004 对象磁盘缓存缺少真正的缓存治理

- 严重级别：P2
- 影响范围：节点磁盘占用、长期运行稳定性、运维治理
- 根因：`NodeObjectCacheService` 初版只有命中与写入，没有访问时间刷新、空闲过期和容量淘汰。
- 修复方案：
  - 为 `objectDisk` 增加 `highWatermarkPercent`、`lowWatermarkPercent`、`maxIdleSeconds`
  - 命中时刷新 `lastModified`
  - 先按空闲时间淘汰，再按双水位回收到低水位
  - 保护当前访问对象不被同轮 GC 淘汰
- 验证：
  - `NodeObjectCacheServiceTest.testEnsureCachedEvictsLeastRecentlyUsedObjectsWhenOverCapacity`
  - `NodeObjectCacheServiceTest.testEnsureCachedEvictsExpiredObjectsBeforeCapacityGc`
  - `SdkworkFsPropertiesTest`
- 状态：已修复

## RV-005 多实例下自动发布去重不完整

- 严重级别：P2
- 影响范围：高可用控制面、并发 sync、幂等性
- 根因：自动发布原先仅依赖单 JVM 内 `ConcurrentHashMap` 加锁，缺少数据库级唯一约束。
- 修复方案：
  - 在 `operation` 表增加 `dedupe_key` 唯一索引
  - 仅对 `BUILD_RELEASE` 活跃任务写入 `build-release:<releaseId>`
  - 终态任务清空 `dedupe_key`，允许失败后重试
  - 自动发布链路将重复排队审计为 `SKIPPED`
- 验证：
  - `OperationServiceTest.testCreatePendingRejectsDuplicateDedupeKey`
  - `OperationServiceTest.testTerminalStateReleasesDedupeKeyForRetry`
  - `CoreSchemaMigrationTest`
  - `RepositorySyncAutoReleaseServiceTest.testTriggerTreatsDuplicateBuildQueueAsSkipped`
- 状态：已修复

## RV-006 构建命令超时后缺少回收与日志控制

- 严重级别：P2
- 影响范围：跨平台构建稳定性、节点资源泄漏、问题排查效率
- 根因：`BuildWorkspaceCommandRunner` 超时时仅强杀主进程，未处理子进程树，也未限制日志读取大小。
- 修复方案：
  - 只读取日志尾部，默认限制 `64KiB`
  - 超时时按“优雅终止 -> 强制终止”处理整个进程树
  - 超时异常附带输出尾部
  - 日志文件删除改为 best-effort
- 验证：
  - `BuildWorkspaceCommandRunnerTest.testRunTimeoutIncludesOutputTail`
  - `BuildWorkspaceCommandRunnerTest.testRunCapturesOutputTailWithinBound`
- 状态：已修复

## RV-007 控制面资源 ID 缺少统一格式校验与错误映射

- 严重级别：P2
- 影响范围：控制面 API 一致性、错误可诊断性、目录与主键安全边界
- 根因：
  - 请求 DTO 大多只有 `@NotBlank`，无法拦截带空格或非法分隔符的 ID
  - controller 未校验路径变量与查询参数
  - `ApiExceptionHandler` 未统一映射 Bean Validation / method validation 异常
- 修复方案：
  - 新增 `ResourceIdentifier` 注解，统一约束 `tenantId`、`repositoryId`、`appId`、`releaseId`
  - 在 `ApplicationController`、`RepositoryController`、`ReleaseController` 中补齐路径和查询参数校验
  - 在 `ApiExceptionHandler` 中统一把验证失败映射为 `400 + INVALID_ARGUMENT`
- 验证：
  - `ApplicationApiTest.testCreateApplicationRejectsInvalidAppId`
  - `ApplicationApiTest.testGetApplicationRejectsInvalidAppIdPath`
  - `ApplicationApiTest.testListApplicationsRejectsInvalidTenantId`
  - `RepositoryApiTest.testCreateRepositoryRejectsInvalidRepositoryId`
  - `ReleaseApiTest.testCreateReleaseRejectsInvalidReleaseId`
  - `ReleaseApiTest.testBuildReleaseRejectsInvalidReleaseId`
  - `ReleaseApiTest.testListReleasesRejectsInvalidAppId`
- 状态：已修复

## RV-008 仓库来源与复制配置允许延迟失败

- 严重级别：P2
- 影响范围：仓库创建、Git/S3 来源治理、同步与自动发布链路
- 根因：
  - `RepositoryService.create(...)` 只负责序列化配置，不校验来源与复制配置的组合是否合法
  - `RepositoryCreateRequest` 缺少 cross-field 约束
  - remote 配置字段未声明最小完整性要求
- 修复方案：
  - 在 `RepositoryCreateRequest` 上增加按 `sourceType` / `replication.enabled` 生效的 `@AssertTrue` 校验
  - `GIT` 来源要求 `source.gitUrl`
  - `S3` 来源要求 `source.bucket`
  - 复制开启时要求 `replication.remotes` 非空
  - 在 `RepositoryRemoteConfig` 上增加 `name/url/role/priority` 必填约束
- 验证：
  - `RepositoryApiTest.testCreateGitRepositoryRejectsMissingGitUrl`
  - `RepositoryApiTest.testCreateS3RepositoryRejectsMissingBucket`
  - `RepositoryApiTest.testCreateRepositoryRejectsEnabledReplicationWithoutRemotes`
  - `RepositoryApiTest.testCreateRepositoryRejectsIncompleteRemoteConfig`
  - `RepositoryApiTest.testCreateRepositoryRejectsInvalidGitConfigBeforeSync`
  - `RepositoryApiTest.testRejectedInvalidRepositoryDoesNotAutoBuildCandidateRelease`
- 状态：已修复
