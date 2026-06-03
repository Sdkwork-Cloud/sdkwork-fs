# 当前实现 Review 总览

## 当前阶段

本轮在上一轮已经完成的对象缓存治理、构建幂等和构建执行韧性基础上，继续收口 2 个控制面一致性问题：

- RV-007：控制面 API 对资源 ID 的输入约束和错误映射不统一
- RV-008：仓库来源与复制配置缺少创建时 fail-fast 校验

本轮结束后，当前代码已经形成“资源完整性 + 自动发布幂等 + 构建执行韧性 + 控制面输入校验 + 仓库配置前置校验”的一轮完整闭环。

## 结论摘要

当前累计确认 8 类问题，8 类均已完成代码修复并有自动化回归覆盖，暂无阻断当前阶段交付的 P1/P2 缺陷。

| 编号 | 严重级别 | 主题 | 当前状态 |
| --- | --- | --- | --- |
| RV-001 | P1 | 应用创建允许绑定不存在仓库 | 已修复 |
| RV-002 | P1 | 应用/发布允许跨租户引用资源 | 已修复 |
| RV-003 | P1 | 长应用 ID 场景下自动发布 `releaseId` 截断碰撞 | 已修复 |
| RV-004 | P2 | 节点对象磁盘缓存缺少容量上限、TTL、淘汰实现 | 已修复 |
| RV-005 | P2 | 自动发布幂等锁仅为单实例内存锁，集群态不可全局去重 | 已修复 |
| RV-006 | P2 | 构建命令超时后缺少子进程回收与日志截断 | 已修复 |
| RV-007 | P2 | 控制面资源 ID 缺少统一格式校验和标准错误映射 | 已修复 |
| RV-008 | P2 | 仓库来源与复制配置允许非法组合在创建后延迟失败 | 已修复 |

## 本轮新增完成修复

### 1. 控制面资源 ID 统一校验

- 新增 `ResourceIdentifier` 组合注解和 `ValidationPatterns.RESOURCE_IDENTIFIER`。
- 当前统一约束 `tenantId`、`repositoryId`、`appId`、`releaseId` 仅允许小写字母、数字、`.`、`_`、`-`，并保持最大长度 `64`。
- 该约束同时应用到请求体、路径变量和查询参数，不再只覆盖 `@RequestBody` 的非空校验。

### 2. API 错误映射闭环

- `ApiExceptionHandler` 现在统一处理 `MethodArgumentNotValidException`、`BindException`、`ConstraintViolationException`、`HandlerMethodValidationException`。
- 所有输入校验失败统一返回：
  - HTTP `400`
  - `code = INVALID_ARGUMENT`
  - `message = request validation failed`
- 非法路径参数不再落入 service 层后误报 `404 NOT_FOUND`。

### 3. 仓库配置创建时 fail-fast

- `RepositoryCreateRequest` 新增按 `sourceType` / `replication.enabled` 驱动的 cross-field 校验。
- `GIT` 来源现在要求 `source.gitUrl` 必填；`S3` 来源现在要求 `source.bucket` 必填。
- 复制开启时 `replication.remotes` 不能为空。
- `RepositoryRemoteConfig` 新增 `name/url/role/priority` 完整性校验，非法 remote 不再在 `sync` 阶段才报错。
- 两个历史用例已迁移为“创建即拒绝”的行为回归，避免后续又退回延迟失败路径。

### 4. 回归测试补齐

- 新增并通过以下输入约束回归：
  - `ApplicationApiTest.testCreateApplicationRejectsInvalidAppId`
  - `ApplicationApiTest.testGetApplicationRejectsInvalidAppIdPath`
  - `ApplicationApiTest.testListApplicationsRejectsInvalidTenantId`
  - `RepositoryApiTest.testCreateRepositoryRejectsInvalidRepositoryId`
  - `RepositoryApiTest.testCreateGitRepositoryRejectsMissingGitUrl`
  - `RepositoryApiTest.testCreateS3RepositoryRejectsMissingBucket`
  - `RepositoryApiTest.testCreateRepositoryRejectsEnabledReplicationWithoutRemotes`
  - `RepositoryApiTest.testCreateRepositoryRejectsIncompleteRemoteConfig`
  - `ReleaseApiTest.testCreateReleaseRejectsInvalidReleaseId`
  - `ReleaseApiTest.testBuildReleaseRejectsInvalidReleaseId`
  - `ReleaseApiTest.testListReleasesRejectsInvalidAppId`

## 本轮验证结果

已完成并通过以下验证：

- `mvn test "-Dtest=ApplicationApiTest,RepositoryApiTest,ReleaseApiTest"`
- `mvn test "-Dtest=RepositoryApiTest"`
- `mvn test "-Dtest=ReleaseApiTest#testBuildReleaseRunsRealGradleBuildWhenEligible"`
- `mvn test`
- `mvn package -DskipTests`
- `docker compose config`

其中全量测试结果为：

- `Tests run: 111, Failures: 0, Errors: 0, Skipped: 0`

## 当前剩余风险

当前剩余项主要属于增强项，而非阻断项：

- `RepositoryUpdateRequest` 仍未与创建接口保持完全一致的 cross-field 校验口径，理论上仍可通过 patch 写入非法配置组合。
- `manifestDigest`、`objectHash` 等内容地址字段仍缺少语义格式校验。
- 全量测试期间曾出现一次 `ReleaseApiTest.testBuildReleaseRunsRealGradleBuildWhenEligible` 的瞬时失败，单测重跑与全量重跑均已通过，说明该用例存在环境敏感性，后续仍应继续稳定化。
- JDK 25 下测试期仍会出现 Mockito 动态 agent 警告，属于工具链兼容性提醒，不影响当前应用运行。
