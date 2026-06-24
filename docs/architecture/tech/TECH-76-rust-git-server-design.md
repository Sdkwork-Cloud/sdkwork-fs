> Migrated from `docs/架构/76-Rust版Git-Server完整能力设计.md` on 2026-06-24.
> Owner: SDKWork maintainers

# Rust 版 Git Server 完整能力设计

## 1. 目标

`sdkworkfs-git-server` 必须成为完整 Git server，而不是简单的 Git 仓库代理。它需要覆盖：

- 仓库托管
- Smart HTTP
- Git over SSH
- `clone/fetch/push`
- `upload-pack/receive-pack`
- Git LFS
- hook 与保护规则
- push 收据输出
- 与镜像、双向同步、候选 Release 链路打通

## 2. Rust 模块拆分

### 2.1 `git-http-front`

- Smart HTTP 路由
- 认证上下文
- 请求解析
- 协议执行层分发

### 2.2 `git-ssh-front`

- SSH 握手
- 公钥认证
- 仓库授权
- Git 子命令路由

### 2.3 `git-repository-service`

- 受管 bare repo 路径管理
- 仓库本地只读/可写状态
- push 收据 outbox

### 2.4 `git-protocol-engine`

- refs 读取
- receive-pack
- upload-pack
- pack/objects 处理

### 2.5 `git-policy-engine`

- 分支保护
- 标签保护
- 主干分支规则
- 只读模式校验

### 2.6 `git-lfs-service`

- LFS batch
- 票据生成
- 与对象存储打通

## 3. 开源复用策略

### 3.1 主体依赖

- `gix`
- `gix-transport`
- `gix-protocol`
- `russh`
- `axum`

### 3.2 设计原则

- 生产实现以 Rust 库集成为主。
- 外部 Git CLI 只作为互操作测试参考，不作为生产主依赖。
- 所有 server-side Git 细节必须收敛到 `sdkworkfs-git` 和 `sdkworkfs-git-server`。

## 4. 仓库模型

每个仓库至少有以下实体：

- `repository`
- `managed bare repo`
- `mainlineRef`
- `auth policy`
- `branch protection policy`
- `mirror relations`
- `lfs policy`
- `push receipt outbox`

推荐目录：

```text
var/sdkworkfs/git/repos/<tenant>/<repo>.git
var/sdkworkfs/git-server/state/push-outbox.db
var/sdkworkfs/git-server/tmp/
```

## 5. Push 接受链路

```text
git push
-> git-server 获取仓库配置快照
-> 鉴权与保护规则校验
-> receive-pack 更新 bare repo refs
-> 写入 push receipt outbox
-> 返回客户端成功
-> 异步上报 control
-> control 幂等更新 resolved revision 与候选 Release
```

关键约束：

- push 成功后，control 不可用不能导致协议回滚。
- 若配置快照不可用且本地缓存已过期，写操作必须失败。
- `pushReceiptId` 必须可幂等重放。

## 6. Fetch / Clone 链路

```text
git clone/fetch
-> gateway / ssh front 鉴权
-> git-server 解析仓库归属
-> upload-pack
-> 读取 refs 和 objects
-> 输出 pack 数据
```

## 7. 授权与保护

必须支持：

- 读写权限分离
- 保护分支
- 禁止 `force push`
- fast-forward only
- 标签保护
- 指定角色写权限
- 只读仓库模式

仓库默认主干：

- 默认 `refs/heads/main`
- 可配置改为 `refs/heads/dev` 或其他分支
- 所有候选 Release 触发规则都必须基于 `mainlineRef`，而不是写死 `main`

## 8. Git LFS 与对象边界

- LFS 元数据归 Git server 所有。
- LFS 大对象存储在 S3 的 Git LFS 命名空间。
- Release 运行时对象使用独立命名空间。
- 构建阶段允许内容摘要去重，但 LFS 与 Release 不能共用写入权限。

## 9. 镜像与双向同步归属

平台必须支持：

- `pull-only`
- `push-only`
- `mirror`
- `bidirectional`

但职责边界必须保持：

- `sdkworkfs-git-server` 负责协议入口与仓库访问能力。
- `sdkworkfs-job-runner` 负责镜像、双向同步、冲突重试和补偿。
- `sdkworkfs-git` 负责共用 Git 操作封装。

## 10. 集群部署要求

- 单仓库同一时刻只能有一个写主 Git 节点。
- 仓库归属信息由 control 统一分配。
- gateway 或前置 SSH 入口必须按仓库归属路由。
- 切主必须带 `placementEpoch` 或等价 fencing 机制。

## 11. 与控制面的边界

Git server 不负责：

- 发布审批
- 激活到节点
- 运行时缓存
- 版本切换
- 长时间镜像重试

Git server 负责：

- 仓库内容与 Git 协议
- push 收据产出
- 仓库策略执法
- LFS 鉴权

## 12. 正式要求

- `sdkworkfs-git-server` 必须作为独立服务存在。
- Git 协议实现、仓库管理、镜像执行和发布编排不可混写。
- push/fetch/clone 必须可审计、可限流、可鉴权。

