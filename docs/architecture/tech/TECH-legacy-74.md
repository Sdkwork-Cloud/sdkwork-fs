> Migrated from `docs/架构/74-仓库来源与复制配置校验补充规范.md` on 2026-06-24.
> Owner: SDKWork maintainers

# 仓库来源与复制配置校验补充规范

## 1. 目标

本规范补充仓库创建接口上的配置前置校验要求，解决“仓库能创建成功，但在 `sync` 或自动发布阶段才暴露配置错误”的延迟失败问题。

## 2. 生效范围

本规范覆盖 `POST /api/v1/repositories`。

约束对象包括：

- `RepositoryCreateRequest`
- `RepositorySourceConfig`
- `RepositoryReplicationConfig`
- `RepositoryRemoteConfig`

## 3. 来源配置约束

### 3.1 GIT 来源

当 `sourceType = GIT` 时：

- `source` 不能为空
- `source.gitUrl` 必填
- `branch/tag/commit` 允许为空，服务端可回退到 `defaultBranch`

### 3.2 S3 来源

当 `sourceType = S3` 时：

- `source` 不能为空
- `source.bucket` 必填
- `source.prefix` 可选

### 3.3 UPLOAD 来源

当 `sourceType = UPLOAD` 或未显式提供来源配置时：

- 不要求 `source.gitUrl`
- 不要求 `source.bucket`

## 4. 复制配置约束

当 `replication.enabled = true` 时：

- `replication.remotes` 不能为空
- 每个 remote 必须具备：
  - `name`
  - `url`
  - `role`
  - `priority`

当复制未开启或未提供 `replication` 时：

- 不强制要求 remote 列表

## 5. 错误映射

仓库配置非法时，控制面统一返回：

- HTTP `400`
- `code = INVALID_ARGUMENT`
- `message = request validation failed`

这样可保证：

- 非法仓库不会进入持久化层
- 不会在后续 `sync` 阶段才以 `SYNC_CONFIG_INVALID` 暴露
- 自动发布链路不会因为无效仓库被误触发

## 6. 当前边界

本规范当前只保证仓库创建入口 fail fast，仍未覆盖：

- `PATCH /api/v1/repositories/{repositoryId}` 的同口径校验
- remote URL 的协议白名单与更细粒度格式约束
- `buildProfile`、`buildOutput` 的语义合法性

上述内容将在下一轮继续补齐。

## 7. 测试与验收

本规范对应的最小验收用例如下：

- `RepositoryApiTest.testCreateGitRepositoryRejectsMissingGitUrl`
- `RepositoryApiTest.testCreateS3RepositoryRejectsMissingBucket`
- `RepositoryApiTest.testCreateRepositoryRejectsEnabledReplicationWithoutRemotes`
- `RepositoryApiTest.testCreateRepositoryRejectsIncompleteRemoteConfig`
- `RepositoryApiTest.testCreateRepositoryRejectsInvalidGitConfigBeforeSync`
- `RepositoryApiTest.testRejectedInvalidRepositoryDoesNotAutoBuildCandidateRelease`

验收通过标准：

- 非法来源配置在创建阶段直接返回 `400`
- 非法复制配置在创建阶段直接返回 `400`
- 对应仓库记录不会落库
- 对应同步或自动发布操作不会被创建

