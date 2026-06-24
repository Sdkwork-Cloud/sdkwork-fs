# Rust 版限界上下文与 crate 细化规范

## 1. 目标

在已经完成“服务级拆分”的基础上，继续明确二进制内部的限界上下文和 crate 拆分规则，防止 `sdkworkfs-control`、`sdkworkfs-job-runner`、`sdkworkfs-git-server`、`sdkworkfsd` 继续长成超大模块。

## 2. 设计原则

- 先按真相归属拆服务，再按限界上下文拆模块。
- 一个上下文只回答一类问题。
- 能作为可复用能力沉淀的，优先拆 crate。
- 只在一个二进制内部使用且不适合复用的，保留为内部模块。
- 若一个模块同时依赖 PostgreSQL、Git 协议和本地文件系统，说明边界已经失控。

## 3. `sdkworkfs-control` 内部上下文

建议固定为以下上下文：

| 上下文 | 责任 |
| --- | --- |
| `repository-admin` | 仓库元数据、主干配置、保护规则、放置归属 |
| `application-admin` | 应用、环境、默认发布策略 |
| `release-admin` | Release 生命周期、门禁、审批 |
| `node-admin` | 节点注册投影、状态视图、选择器 |
| `policy-admin` | 权限、配额、同步策略、缓存策略 |
| `operation-admin` | Operation 聚合、取消、查询 |
| `audit-query` | 审计查询与检索 |

禁止把这些上下文再堆进一个通用 `service.rs`。

## 4. `sdkworkfs-job-runner` 内部上下文

建议固定为以下 worker 族：

| worker | 责任 |
| --- | --- |
| `repository-sync-worker` | 外部 Git 来源同步到受管仓库 |
| `mirror-worker` | 单向镜像、双向同步、冲突处理 |
| `release-build-worker` | 构建、产物选择、Manifest 生成 |
| `release-verify-worker` | 对象校验、签名、完整性检查 |
| `activation-worker` | 节点预热、发布、切换编排 |
| `rollback-worker` | 回滚任务编排 |
| `gc-worker` | Git GC、对象 GC、缓存清理 |

所有 worker 只通过 `Operation + Attempt + Lease` 接入，不得自行维护一套独立任务状态机。

## 5. `sdkworkfs-git-server` 内部上下文

建议固定为以下模块：

| 模块 | 责任 |
| --- | --- |
| `http-front` | Smart HTTP 入口 |
| `ssh-front` | Git over SSH 入口 |
| `protocol-engine` | `upload-pack` / `receive-pack` |
| `repo-store` | bare repo 路径和本地仓库访问 |
| `policy-check` | 分支保护、标签保护、只读校验 |
| `lfs-gateway` | LFS batch、票据、对象命名空间映射 |
| `push-outbox` | push receipt 持久化与补偿上报 |

说明：

- `mirror` 不属于 git-server 内部长期上下文。
- `mirror` 的长任务应由 runner 执行，只复用 `sdkworkfs-git`。

## 6. `sdkworkfsd` 内部上下文

建议固定为以下模块：

| 模块 | 责任 |
| --- | --- |
| `node-agent` | 注册、心跳、拉任务、续租 |
| `cache-manager` | L1/L2/L3 缓存索引和淘汰 |
| `materializer` | Manifest 到真实目录 |
| `switcher` | `current/previous` 原子切换 |
| `mount-adapter` | FUSE 或平台挂载适配 |
| `writeback-agent` | 可写区域监控、写回队列、冲突副本 |
| `local-state` | SQLite、本地恢复、磁盘布局管理 |

## 7. crate 与内部模块的拆分规则

满足以下任一条件时，必须优先拆成独立 crate：

- 被两个及以上二进制复用
- 需要独立第三方依赖树
- 需要单独稳定测试夹具
- 需要单独版本策略或明显的编译裁剪

满足以下全部条件时，可保留为二进制内部模块：

- 只被一个二进制使用
- 与外部协议无共享契约
- 不需要独立依赖树
- 不承载跨服务真相

## 8. 当前推荐的 crate 细化

相比第一轮规划，当前推荐额外细化：

- `sdkworkfs-persistence-pg`
- `sdkworkfs-persistence-sqlite`
- `sdkworkfs-process`
- `sdkworkfs-build-core`
- `sdkworkfs-build-nodejs`
- `sdkworkfs-build-java`
- `sdkworkfs-build-python`

当前不建议继续拆分为独立 crate 的部分：

- `sdkworkfs-contracts`
- `sdkworkfs-git`
- `sdkworkfs-cache`

原因：

- 它们已有清晰职责，但进一步按协议或算法拆 crate 会过早增加复杂度。
- 现阶段更适合在 crate 内部按模块保持清晰边界。

## 9. 正式要求

- 后续任何新增 crate 都必须说明它解决了哪个“过宽模块”问题。
- 后续任何二进制内部模块命名都必须体现上下文，而不是使用 `common`、`misc`、`manager` 这类模糊名字。
- 若一个模块同时需要了解控制面聚合、Git 协议和节点文件路径，必须立即重拆边界。
- 具体目录与层次规范见 `81-Rust版代码布局与端口适配器规范.md`。
