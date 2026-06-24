> Migrated from `docs/架构/18-对象生命周期与回收规范.md` on 2026-06-24.
> Owner: SDKWork maintainers

# 对象生命周期与回收规范

## 1. 目标

统一定义对象从创建、索引、引用、保留、删除到恢复的规则，避免缓存回收和对象删除破坏 Release 可用性。

## 2. 对象生命周期阶段

```text
CREATED
-> INDEXED
-> REFERENCED
-> CACHED
-> MATERIALIZED
-> RETAINED
-> TOMBSTONED
-> PURGED
```

## 3. 引用关系

对象可能被以下实体引用：

- Release Manifest
- 当前运行版本
- previous 回滚版本
- 预热中的目标版本
- 写回任务
- Git 导入批次

## 4. 删除前提

对象只有在以下条件全部满足时才允许物理删除：

- 不被任何活跃 Release 引用
- 不被当前与 previous 版本引用
- 不在进行中的预热任务中
- 不在进行中的写回任务中
- 已超过保留期

## 5. 墓碑机制

删除建议分两步：

1. 写入墓碑 `tombstone`
2. 延迟物理清理

墓碑至少记录：

- objectHash
- deleteReason
- deleteTime
- operator
- recoverableUntil

## 6. 版本保留

对象保留策略至少支持：

- 按 Release 数量保留
- 按时间保留
- 按通道保留
- 按租户配额保留

## 7. 恢复能力

在墓碑有效期内应支持：

- 取消删除
- 恢复对象引用
- 重建缓存索引

## 8. 垃圾回收策略

支持：

- 标记-清除
- 引用计数辅助校验
- 分阶段清理
- 低峰期清理

## 9. 正式要求

- 任何物理删除都必须先通过引用检查
- 墓碑不得绕过审计
- 当前版本所需对象不得被清理
- 回滚点所需对象不得在保留期内被清理

