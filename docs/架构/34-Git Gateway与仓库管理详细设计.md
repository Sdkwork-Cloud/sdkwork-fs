# Git Gateway 与仓库管理详细设计

## 1. 目标

把 `sdkworkfs` 的 Git 能力从“支持 Git”细化为可实现、可运维、可审计的 Git Gateway 标准，明确仓库生命周期、协议处理、push 时序、分支保护、镜像同步和审计边界。

## 2. 组件职责划分

| 组件 | 职责 | 不负责 |
| --- | --- | --- |
| `sdkworkfs-gateway` | `/git/*` HTTP 入口路由、TLS、审计上下文注入 | Git pack 解析、仓库状态机 |
| `sdkworkfs-git-server` | Smart HTTP、SSH、receive-pack、upload-pack、LFS batch、push 收据落盘 | 发布审批、长时间镜像重试 |
| `sdkworkfs-control` | 仓库元数据、主干配置、保护规则、仓库放置归属、push 收据摄取 | 受管 bare repo 文件写入 |
| `sdkworkfs-job-runner` | 仓库同步、镜像、双向同步、仓库 GC、候选 Release 构建 | Git 客户端会话接入 |

## 3. 仓库生命周期

仓库至少支持以下状态：

- `CREATING`
- `READY`
- `READ_ONLY`
- `ARCHIVED`
- `ERROR`

生命周期动作至少包括：

- 创建空仓库
- 导入已有仓库
- 设置 `mainlineRef`
- 配置保护分支与保护标签
- 配置上游 / 镜像远端
- 归档、恢复、只读切换

说明：

- `mainlineRef` 默认值为 `refs/heads/main`。
- 是否允许 `refs/heads/dev` 或其他分支作为主干，必须由仓库级配置显式决定。

## 4. 协议与接入模式

### 4.1 HTTP

- 必须支持 Smart HTTP。
- 必须支持基础 `clone` / `fetch` / `push`。
- 必须支持大仓库超时、限速和并发保护配置。

### 4.2 SSH

- 必须支持标准 Git SSH 工作流。
- 必须支持公钥指纹校验与仓库级授权。
- 必须支持租户级或用户级 SSH key 绑定。

## 5. 鉴权与授权矩阵

| 操作 | 协议 | 最低权限 | 额外校验 |
| --- | --- | --- | --- |
| clone/fetch | HTTP | `repo:read` | 仓库状态必须允许读取 |
| clone/fetch | SSH | `repo:read` | key 与主体映射有效 |
| push | HTTP | `repo:write` | 分支保护、tag 保护、只读状态 |
| push | SSH | `repo:write` | 分支保护、tag 保护、只读状态 |
| lfs upload/download | HTTP | 继承仓库权限 | 对象大小、配额、票据有效期 |
| mirror push/pull | 内部任务 | `repo:replicate` | 远端配置、冲突策略、循环防护 |

## 6. Push 处理时序

标准流程如下：

```text
git push
-> git-server 拉取仓库配置快照
-> 认证、仓库状态校验、分支/标签保护预检查
-> receive-pack 接收 pack 并更新 bare repo refs
-> 生成 push receipt 与审计记录
-> receipt 先持久化到 git-server 本地 outbox
-> 向客户端返回 push 成功
-> 异步把 receipt 摄取到 control
-> control 幂等更新 repository resolved revision
-> control 按策略创建 release candidate / operation
-> job-runner 执行镜像、构建、发布链路
```

关键原则：

- push 是否成功只由 git-server 的协议写入与策略校验决定。
- control 不可用时，push 收据必须先落本地 outbox，待恢复后补偿摄取。
- 若仓库策略快照不可获得且缓存已过期，写操作必须拒绝，不能盲写。

## 7. 分支与标签保护

至少应支持：

- 保护分支禁止 `force push`
- 保护标签禁止覆盖与删除
- 指定角色才可推送到 `mainlineRef`、`release/*`
- 生产 tag 必须受保护
- 可配置是否要求 fast-forward only

推荐配置项：

```yaml
git:
  mainlineRef: refs/heads/main
  protection:
    branches:
      - pattern: refs/heads/main
        denyForcePush: true
        requireRole: repo:write
        fastForwardOnly: true
      - pattern: refs/heads/release/*
        denyDeletion: true
    tags:
      - pattern: refs/tags/v*
        immutable: true
```

保护规则执行顺序：

1. 仓库是否只读
2. 主体是否具备写权限
3. ref 级规则是否允许
4. 特定环境门禁是否需要审批

## 8. 仓库放置与集群路由

- 每个仓库同一时刻只能有一个可写主 Git 节点。
- `sdkworkfs-control` 维护 `RepositoryPlacement`，至少包含 `repositoryId`、`primaryGitNodeId`、`placementEpoch`。
- HTTP Git 流量由 gateway 按仓库归属路由到主节点。
- SSH Git 流量由 Git 前置入口按仓库归属转发到主节点。
- 故障切主必须具备 fencing 机制，避免双主写入。

## 9. 镜像与双向同步

### 9.1 单向镜像

- push 完成后可异步镜像到目标远端。
- 镜像失败不得回滚本地主仓库写入。
- 镜像结果必须可审计、可重试。

### 9.2 双向同步

- 默认关闭。
- 必须显式指定主优先级远端。
- 必须支持循环防护、重放抑制、`force push` 保护。

### 9.3 执行归属

- 镜像与双向同步任务由 `sdkworkfs-job-runner` 执行。
- `sdkworkfs-git-server` 只提供仓库访问、策略校验和 push 收据能力。
- 具体 Git 操作由共享 crate `sdkworkfs-git` 复用。

### 9.4 冲突策略

支持：

- `reject`
- `manual-review`
- `prefer-primary`

默认：`reject`

## 10. Git 对象、LFS 与 Release 对象边界

- Git pack/refs 保存在受管 bare repo 中。
- Git LFS 大对象保存在 Git LFS 专用对象前缀中。
- Release 运行时对象保存在 Release 专用对象前缀中。
- 三者可以使用同一 S3 服务，但必须逻辑隔离、权限隔离、生命周期隔离。
- build/release 阶段可基于内容摘要做去重，但不允许共享写入控制权。

## 11. 仓库存储建议

- 服务端优先使用 bare repo。
- 仓库工作区不应长期作为正式运行目录。
- Git 仓库存储与 Release 物化目录必须分离。
- 必须支持定期仓库完整性检查与垃圾回收。

推荐默认目录：

- bare repo：`./var/sdkworkfs/git/repos/<tenantId>/<repositoryId>.git`
- push outbox：`./var/sdkworkfs/git-server/state/push-outbox.db`
- 临时 pack 缓冲：`./var/sdkworkfs/git-server/tmp`

## 12. 审计要求

必须记录：

- push / fetch / clone / mirror
- 仓库创建、导入、归档、只读切换
- 分支保护与标签保护变更
- 上游与镜像远端变更
- 双向同步冲突事件

审计字段至少包含：

- `tenantId`
- `repositoryId`
- `operator`
- `protocol`
- `remoteIp`
- `refUpdates`
- `requestId`
- `pushReceiptId`
- `timestamp`

## 13. 正式要求

- Git Gateway 必须是受控入口，而不是裸暴露 Git 仓库目录。
- push 收据必须先本地持久化，再异步摄取 control。
- push 只能触发候选版本或后续任务，不得绕过发布门禁直接激活生产。
- Git 集群必须保证单仓库单写主。
