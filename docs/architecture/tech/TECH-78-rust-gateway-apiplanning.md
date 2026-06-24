> Migrated from `docs/架构/78-Rust版Gateway与管理-非管理API规划.md` on 2026-06-24.
> Owner: SDKWork maintainers

# Rust 版 Gateway 与管理/非管理 API 规划

## 1. 目标

明确 `sdkworkfs-gateway` 的路由边界，防止后续把管理 API、运行时 API、节点接口和 Git 协议混成一锅。

## 2. API 分域

### 2.1 管理 API

前缀：`/admin/v1`

面向：

- 控制台
- 运维系统
- 发布平台
- CI 管理流程

特点：

- 强鉴权
- 强审计
- 资源 CRUD 与治理动作

### 2.2 运行时 API

前缀：`/runtime/v1`

面向：

- 构建系统
- 发布脚本
- 上传客户端
- 运行时查询侧

特点：

- 偏资源消费
- 偏只读或上传相关
- 不承载审批与治理逻辑

### 2.3 节点 API

前缀：`/node/v1`

面向：

- `sdkworkfsd`

特点：

- 机器到机器
- 强约束、低噪音
- 优先使用 gRPC/Protobuf

### 2.4 Git 协议入口

前缀：`/git/*` 与 SSH 端口

特点：

- Git 客户端直连
- 独立于资源 API
- 独立限流和审计

## 3. 路由到后端服务

```text
/admin/v1/*    -> sdkworkfs-control
/runtime/v1/*  -> sdkworkfs-control
/node/v1/*     -> sdkworkfs-control 的 node-facing API
/internal/v1/* -> sdkworkfs-control
/git/*         -> sdkworkfs-git-server
SSH            -> sdkworkfs-git-server
```

说明：

- `sdkworkfs-gateway` 只负责入口协议、认证、限流、审计上下文和路由。
- `sdkworkfs-control` 在 v1 阶段直接承载 node-facing API，不额外引入新的 public binary。
- 后续若节点规模显著上升，可在不改变协议前提下拆出独立 node API service。

## 4. 资源组

### 4.1 管理 API

- 仓库管理
- 应用管理
- 环境管理
- 发布管理
- 策略管理
- 节点管理
- 审计查询
- 异步任务

### 4.2 运行时 API

- 上传会话
- Manifest
- Release 只读查询
- 物化计划
- 下载票据
- 运行时版本查询

## 5. 典型接口

### 5.1 管理 API

- `POST /admin/v1/repositories`
- `PATCH /admin/v1/repositories/{repositoryId}`
- `POST /admin/v1/repositories/{repositoryId}:sync`
- `POST /admin/v1/releases`
- `POST /admin/v1/releases/{releaseId}:publish`
- `POST /admin/v1/releases/{releaseId}:activate`
- `POST /admin/v1/releases/{releaseId}:rollback`

### 5.2 运行时 API

- `POST /runtime/v1/uploads`
- `POST /runtime/v1/uploads/{uploadId}:complete`
- `POST /runtime/v1/manifests`
- `GET /runtime/v1/manifests/{manifestDigest}`
- `GET /runtime/v1/releases/{releaseId}`
- `GET /runtime/v1/releases/{releaseId}/materialization-plan`

## 6. 鉴权区别

| API 域 | 推荐鉴权 |
| --- | --- |
| 管理 API | JWT / OIDC / RBAC |
| 运行时 API | Service Token / JWT |
| 节点 API | mTLS / 节点短期凭据 |
| Git HTTP | PAT / Bearer / 短期仓库票据 |
| Git SSH | SSH 公钥 |

## 7. 版本化与 SDK 生成规则

- OpenAPI 只覆盖 `/admin/v1` 与 `/runtime/v1`。
- 节点协议使用 Protobuf/gRPC，不参与外部 SDK 生成。
- Git 协议入口不进入 OpenAPI，也不生成 SDK。
- `/internal/v1` 只作为服务间契约，不暴露给外部调用方。
- `/admin/v1` 与 `/runtime/v1` 的 breaking change 只能通过 `v2` 新前缀引入。

## 8. 网关职责边界

`sdkworkfs-gateway` 负责：

- TLS 终止
- 身份鉴别
- 请求 ID 注入
- 限流、幂等、基础审计
- 协议路由

`sdkworkfs-gateway` 不负责：

- 业务状态持久化
- Git pack 处理
- 节点计划编排
- 长任务执行

## 9. 不允许的混用

- 不允许把 `git push` 设计成普通资源 API。
- 不允许把节点物化任务设计成外部管理接口同步执行。
- 不允许让运行时 API 承担审批和策略治理。
- 不允许让 Git server 直接改控制面数据库状态机。
- 不允许把 `/node/v1` 和 `/admin/v1` 的权限模型混用。

## 10. 正式要求

- 管理 API、运行时 API、节点 API、Git 协议必须独立演进。
- 管理 API 与运行时 API 的权限、限流和审计策略必须分开配置。
- 后续所有 SDK 生成、接口变更评审都以本分域规则为准。

