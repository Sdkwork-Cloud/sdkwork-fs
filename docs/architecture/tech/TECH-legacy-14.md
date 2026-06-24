> Migrated from `docs/架构/14-目录布局与运行时路径规范.md` on 2026-06-24.
> Owner: SDKWork maintainers

# 目录布局与运行时路径规范

## 1. 目标

统一定义 `sdkworkfs` 在不同操作系统上的默认目录、节点本地布局、Git 仓库存储位置、对象缓存位置和运行时物化目录，避免后续实现和运维各自发明路径规则。

## 2. 默认根目录

### 2.1 用户级默认目录

| 操作系统 | 默认目录 |
| --- | --- |
| Linux | `~/.local/share/sdkworkfs` |
| macOS | `~/Library/Application Support/sdkworkfs` |
| Windows | `%LOCALAPPDATA%\\sdkworkfs` |

### 2.2 系统级默认目录

| 操作系统 | 默认目录 |
| --- | --- |
| Linux | `/var/lib/sdkworkfs` |
| macOS | `/Library/Application Support/sdkworkfs` |
| Windows | `C:\\ProgramData\\sdkworkfs` |

## 3. 节点本地目录布局

推荐结构：

```text
<SDKWORKFS_HOME>/
  config/
  logs/
  tmp/
  git/
    repos/<repoId>.git
    mirrors/<repoId>/
  objects/
    sha256/ab/cd/<full-hash>
  apps/
    <appId>/
      releases/<releaseId>/
      current/
      previous/
      data/
      workspace/
      queue/
      conflicts/
      meta/
        metadata.db
  mounts/
    <appId>/
```

### 3.1 推荐落盘实例

推荐按以下运行时路径语义落盘：

```text
<git-managed-root>/
  <tenantId>/
    <repositoryId>.git

<object-cache-root>/
  sha256/ab/cd/<digest>.blob

<materialized-root>/
  <appId>/
    releases/<releaseId>/
    current
    previous

<metadata-root>/
  prefetch/<nodeId>/<releaseId>.json
  materialized/<nodeId>/<releaseId>.json
  switches/<nodeId>/<appId>.json
```

- `releases/<releaseId>/` 为不可变 L3 真实目录
- `current` / `previous` 为节点当前阶段的运行别名入口
- `switches/<nodeId>/<appId>.json` 记录 `currentReleaseId`、`previousReleaseId`、目标目录、别名路径、别名类型和切换时间
- 控制面默认通过 `sdkworkfs.git.managed.root-dir=./var/sdkworkfs/git/repos` 管理本地 bare repo，生产部署可显式覆盖到系统级数据目录

## 4. 路径职责

- `git/repos`：受管裸仓库存储，由 Git 同步、镜像和构建链路复用
- `git/mirrors`：外部镜像或上游同步工作目录
- `objects`：内容寻址对象缓存
- `releases`：已物化的不可变版本目录
- `current`：当前运行版本指针，优先使用符号链接，回退时为 JSON 指针文件
- `previous`：上一个稳定回滚点，优先使用符号链接，回退时为 JSON 指针文件
- `data`：用户或业务可写数据
- `workspace`：工作区、临时构建或开发同步目录
- `queue`：待同步与待写回任务
- `conflicts`：冲突副本
- `meta/metadata.db`：节点本地 SQLite 元数据

## 5. 默认挂载点

| 操作系统 | 默认挂载点 |
| --- | --- |
| Linux | `/mnt/sdkworkfs/<appId>` |
| macOS | `/Volumes/sdkworkfs/<appId>` |
| Windows | `S:\\<appId>` 或目录挂载回退 |

## 6. 路径规范

- 目录名使用 ASCII，避免平台编码差异
- `releaseId` 应可直接作为目录名使用
- 运行时目录与写回目录必须隔离
- 任何用户可写路径不得直接落在不可变 `releases/` 下
- `current` / `previous` 必须始终解析到 `releases/` 下的已物化目录，不能指向可写路径
- Windows 或不支持符号链接的环境应回退为别名路径上的 JSON 指针文件，而不是复制整份 release 目录

## 7. 配置覆盖

这些路径都必须支持：

- 平台默认
- 节点覆盖
- 应用覆盖

## 8. 正式要求

- 节点必须能在配置中显式声明本地根目录
- 路径变更必须通过启动前校验
- Windows、Linux、macOS 的路径规则必须有自动化测试覆盖

