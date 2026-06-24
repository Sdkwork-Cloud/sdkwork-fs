> Migrated from `docs/架构/08-Git托管与双向同步标准.md` on 2026-06-24.
> Owner: SDKWork maintainers

# Git 托管与双向同步标准

## 1. 目标

`sdkworkfs` 必须提供成熟的 Git 接入能力，包括：

- HTTP(S) Git
- SSH Git
- 仓库托管
- 上游拉取
- 下游镜像
- 可选双向同步

更细的仓库生命周期、push 流程、分支保护和 Git Gateway 实现边界见 `34-Git Gateway与仓库管理详细设计.md`。

## 2. 协议标准

- HTTP(S)：支持 Smart HTTP
- SSH：支持标准 Git SSH 工作流
- Push、Fetch、Clone、Mirror 必须完整可用

## 3. Git server 实现建议

- 控制面负责仓库注册、权限、审计和同步配置
- 数据面优先复用成熟 Git 协议处理能力，而不是自定义协议
- 推荐通过 Git Gateway 包装原生 Git 协议处理程序

## 4. 同步模式

| 模式 | 说明 |
| --- | --- |
| `pull-only` | 只从上游拉取 |
| `push-only` | 只推送到镜像仓库 |
| `mirror` | 单向镜像与灾备 |
| `bidirectional` | 双向同步，需显式启用 |

## 5. 默认策略

- 默认目标：镜像、灾备、上游同步
- 默认不启用双向同步
- 默认集群角色下，Git 主写入口位于主节点

## 6. 双向同步约束

必须支持：

- 分支过滤
- 标签过滤
- 循环防护
- 优先级定义
- 冲突策略
- 历史改写限制

## 7. 双向同步冲突规则

推荐策略：

- 默认禁止自动处理 `force push`
- 默认禁止无过滤全量双向同步
- 默认要求定义主优先级远端
- 冲突事件必须记录审计

## 8. Git 配置模型

```yaml
git:
  workflow:
    mode: trunk-based
    mainBranch: main
    devSource: main
    prodSource: tag
  replication:
    mode: mirror
    enabled: true
    branchFilter:
      - main
      - release/*
    tagFilter:
      - v*
    loopPrevention: true
```

### 8.1 双向同步高级样例

```yaml
git:
  replication:
    mode: bidirectional
    enabled: true
    remotes:
      upstream:
        url: git@upstream.example.com:team/app.git
        role: upstream
        priority: 100
      dr:
        url: git@dr.example.com:team/app.git
        role: mirror
        priority: 80
    branchFilter:
      - main
      - release/*
    tagFilter:
      - v*
    conflictPolicy: reject
    loopPrevention: true
    rewriteProtection: true
```

### 8.2 循环防护要求

循环防护至少支持：

- 远端唯一标识
- 同步来源标签
- 最近同步指纹
- 相同提交重复传播抑制
- 强制推送保护

### 8.3 同步冲突处理

推荐策略：

- `reject`
- `manual-review`
- `prefer-primary`

默认值：`reject`

## 9. 与发布系统关系

- Git push 不直接等于生产切换
- Git 事件只负责产生或更新候选 Release
- 生产环境切换仍然由发布策略和 Release 激活控制

