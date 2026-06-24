> Migrated from `docs/架构/73-控制面输入校验与错误映射补充规范.md` on 2026-06-24.
> Owner: SDKWork maintainers

# 控制面输入校验与错误映射补充规范

## 1. 目标

本规范补充控制面输入边界上的统一要求，解决“请求体、路径变量、查询参数校验口径不一致”和“验证异常返回模型不统一”的问题。

## 2. 资源 ID 约束

统一约束以下标识：

- `tenantId`
- `repositoryId`
- `appId`
- `releaseId`

统一格式：

- 最大长度：`64`
- 允许字符：小写字母、数字、`.`、`_`、`-`
- 不允许空格、斜杠、反斜杠和其他路径敏感字符
- 首尾字符必须为字母或数字

建议正则：

```text
^[a-z0-9](?:[a-z0-9._-]{0,62}[a-z0-9])?$
```

该约束同时满足：

- 与数据库主键列 `varchar(64)` 保持一致
- `releaseId` / `appId` 可直接安全用于本地目录名
- 避免控制面接收明显异常的资源标识

## 3. 校验落点

要求在以下三个入口同时生效：

### 3.1 请求体 DTO

- `ApplicationCreateRequest`
- `ApplicationBuildReleaseRequest`
- `RepositoryCreateRequest`
- `ReleaseCreateRequest`

### 3.2 路径变量

- `/api/v1/applications/{appId}`
- `/api/v1/applications/{appId}:build-release`
- `/api/v1/repositories/{repositoryId}`
- `/api/v1/repositories/{repositoryId}:sync`
- `/api/v1/releases/{releaseId}`
- `/api/v1/releases/{releaseId}/objects`
- `/api/v1/releases/{releaseId}/materialization-plan`
- `/api/v1/releases/{releaseId}:publish`
- `/api/v1/releases/{releaseId}:activate`
- `/api/v1/releases/{releaseId}:verify`
- `/api/v1/releases/{releaseId}:rollback`

### 3.3 查询参数

- `/api/v1/applications?tenantId=...`
- `/api/v1/repositories?tenantId=...`
- `/api/v1/releases?appId=...`

## 4. 错误映射

控制面必须把以下异常统一映射为标准错误响应：

- `MethodArgumentNotValidException`
- `BindException`
- `ConstraintViolationException`
- `HandlerMethodValidationException`

统一返回：

- HTTP `400`
- `code = INVALID_ARGUMENT`
- `message = request validation failed`

这样可以保证：

- 非法路径参数不会误落为 `404`
- 非法查询参数不会绕过 controller 直接进入 service
- 客户端可以稳定按 `INVALID_ARGUMENT` 做重试外处理

## 5. 当前边界

本轮仅收敛资源 ID 这一层的通用安全约束，暂未覆盖：

- `RepositorySourceConfig` 的按 `sourceType` 跨字段校验
- `RepositoryReplicationConfig` 的 remote 完整性校验
- `manifestDigest`、`objectHash` 等内容地址字段的语义格式校验

上述内容将在下一轮 Schema 强化中继续补齐。

## 6. 测试与验收

本规范对应的最小验收用例如下：

- `ApplicationApiTest.testCreateApplicationRejectsInvalidAppId`
- `ApplicationApiTest.testGetApplicationRejectsInvalidAppIdPath`
- `ApplicationApiTest.testListApplicationsRejectsInvalidTenantId`
- `RepositoryApiTest.testCreateRepositoryRejectsInvalidRepositoryId`
- `ReleaseApiTest.testCreateReleaseRejectsInvalidReleaseId`
- `ReleaseApiTest.testBuildReleaseRejectsInvalidReleaseId`
- `ReleaseApiTest.testListReleasesRejectsInvalidAppId`

验收通过标准：

- 非法资源 ID 均返回 `400`
- 返回体统一为 `INVALID_ARGUMENT`
- 合法请求不受影响，现有主流程测试继续通过

