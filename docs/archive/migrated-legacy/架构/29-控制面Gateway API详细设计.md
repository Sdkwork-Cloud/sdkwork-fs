# 控制面 Gateway API 详细设计

## 1. 目标

`sdkworkfs-gateway` 是系统的统一外部入口，负责：

- 管理 API
- 非管理 API
- Git Smart HTTP 入口
- 统一认证与租户识别
- 限流、幂等、审计字段补齐
- OpenAPI 暴露

它不是业务真相源，不负责保存资源状态，也不直接执行长任务。

## 2. 网关职责边界

### 2.1 必须承担

- TLS 终止
- 请求路由
- 认证、租户识别、权限注入
- 幂等键处理
- 请求大小和频率限制
- 统一错误映射
- `X-Request-Id` 生成与透传

### 2.2 不承担

- 长任务执行
- 构建与发布
- 对象上传大流量代理
- 文件系统挂载
- 节点物化
- Git 仓库内容真相维护

## 3. 推荐技术

- `axum`
- `tower`
- `tower-http`
- `rustls`
- `utoipa`
- `tracing`

## 4. 路由分组

### 4.1 管理 API

前缀：`/admin/v1`

主要资源：

- `/repositories`
- `/applications`
- `/environments`
- `/releases`
- `/nodes`
- `/operations`
- `/policies`
- `/audit-logs`
- `/quotas`

### 4.2 非管理 API

前缀：`/runtime/v1`

主要资源：

- `/uploads`
- `/manifests`
- `/releases`
- `/download-tickets`
- `/release-artifacts`

### 4.3 Git HTTP

前缀：`/git`

主要路径：

- `/git/{tenant}/{repo}.git/info/refs`
- `/git/{tenant}/{repo}.git/git-upload-pack`
- `/git/{tenant}/{repo}.git/git-receive-pack`
- `/git/{tenant}/{repo}.git/objects/...`
- `/git/{tenant}/{repo}.git/lfs/*`

### 4.4 内部接口

前缀：`/internal/v1`

仅用于服务间通信，不暴露给公网。

## 5. 鉴权模型

### 5.1 主体类型

- 平台管理员
- 租户管理员
- 应用发布账号
- CI 服务账号
- 节点服务账号
- Git 用户 / 机器人

### 5.2 鉴权机制

- 管理与运行时 API：JWT / OIDC
- Git HTTP：Bearer Token、PAT、临时凭据
- Git SSH：SSH 公钥 + 仓库授权
- 节点 API：mTLS 或短期节点凭据

### 5.3 权限域

- `repo:read`
- `repo:write`
- `repo:mirror`
- `release:read`
- `release:write`
- `release:publish`
- `release:rollback`
- `node:read`
- `node:operate`
- `audit:read`

## 6. 幂等与并发控制

以下接口必须支持 `Idempotency-Key`：

- `POST /admin/v1/releases`
- `POST /admin/v1/releases/{releaseId}:publish`
- `POST /admin/v1/releases/{releaseId}:activate`
- `POST /admin/v1/releases/{releaseId}:rollback`
- `POST /admin/v1/repositories/{repositoryId}:sync`

以下接口必须支持资源版本并发控制：

- `PATCH /admin/v1/repositories/{repositoryId}`
- `PATCH /admin/v1/applications/{appId}`
- `PATCH /admin/v1/policies/{policyId}`

推荐通过 `If-Match` 或请求体中的 `resourceVersion` 实现。

## 7. 限流策略

推荐限流维度：

- 用户维度
- 租户维度
- 仓库维度
- 节点维度
- Git 协议维度

建议重点限制：

- Git push / receive-pack
- 上传初始化
- 节点心跳
- 激活、回滚等高风险管理操作

## 8. 管理 API 代表性接口

### 8.1 仓库

- `POST /admin/v1/repositories`
- `GET /admin/v1/repositories/{repositoryId}`
- `PATCH /admin/v1/repositories/{repositoryId}`
- `POST /admin/v1/repositories/{repositoryId}:sync`
- `POST /admin/v1/repositories/{repositoryId}:mirror`

### 8.2 应用

- `POST /admin/v1/applications`
- `GET /admin/v1/applications/{appId}`
- `PATCH /admin/v1/applications/{appId}`

### 8.3 发布

- `POST /admin/v1/releases`
- `GET /admin/v1/releases/{releaseId}`
- `POST /admin/v1/releases/{releaseId}:verify`
- `POST /admin/v1/releases/{releaseId}:publish`
- `POST /admin/v1/releases/{releaseId}:activate`
- `POST /admin/v1/releases/{releaseId}:rollback`

### 8.4 节点与任务

- `GET /admin/v1/nodes`
- `GET /admin/v1/nodes/{nodeId}`
- `GET /admin/v1/operations`
- `GET /admin/v1/operations/{operationId}`
- `POST /admin/v1/operations/{operationId}:cancel`

## 9. 非管理 API 代表性接口

- `POST /runtime/v1/uploads`
- `POST /runtime/v1/uploads/{uploadId}:complete`
- `POST /runtime/v1/manifests`
- `GET /runtime/v1/manifests/{manifestDigest}`
- `GET /runtime/v1/releases/{releaseId}`
- `GET /runtime/v1/releases/{releaseId}/objects`
- `GET /runtime/v1/releases/{releaseId}/materialization-plan`

## 10. Git 网关处理规则

### 10.1 Smart HTTP

Gateway 负责：

- 鉴权
- 仓库路径解析
- 租户与仓库上下文注入
- trace / request-id
- 转发到 `sdkworkfs-git-server`

Gateway 不负责：

- packfile 解析
- refs 更新
- receive-pack 执行

### 10.2 LFS

Git LFS 可以通过同一 gateway 统一暴露，但逻辑归 `sdkworkfs-git-server` 和 `sdkworkfs-object`。

## 11. 错误映射

网关层统一输出：

```json
{
  "requestId": "req-1001",
  "code": "INVALID_ARGUMENT",
  "message": "request validation failed",
  "data": null
}
```

错误源可以来自：

- 参数校验
- 认证失败
- 权限不足
- 分支保护拒绝
- 资源不存在
- 状态冲突
- 上游 Git 或对象存储不可用

## 12. OpenAPI 要求

- `/admin/v1` 和 `/runtime/v1` 必须产出 OpenAPI 3
- Git 协议入口不纳入常规 OpenAPI，但要有专门协议文档
- 错误码、枚举、异步操作响应必须可生成 SDK

## 13. 正式要求

- `sdkworkfs-gateway` 必须保持薄网关，不得沉淀业务真相
- 管理 API 和非管理 API 必须共享统一错误模型与审计头
- Git 协议入口必须与资源管理 API 分离实现
