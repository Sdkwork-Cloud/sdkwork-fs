> Migrated from `docs/review/03-本轮执行方案.md` on 2026-06-24.
> Owner: SDKWork maintainers

# 本轮执行方案

## 当前阶段

当前阶段已完成：

- 资源引用完整性修复
- 多租户引用边界修复
- 自动候选发布 `releaseId` 截断碰撞修复
- 对象磁盘缓存容量治理
- 自动发布集群态幂等
- 构建命令超时后的进程回收与日志韧性
- 控制面资源 ID 统一校验与错误映射
- 仓库来源与复制配置创建时 fail-fast 校验

下一阶段待完成：

- RV-009 仓库更新接口与创建接口的校验口径对齐
- RV-010 内容地址字段的语义格式校验
- RV-011 构建与测试矩阵稳定性治理

## 下一轮目标

围绕“所有写入口采用一致约束，测试矩阵更稳定”完成下一轮可验证闭环：

1. 为仓库 patch 接口补齐与创建接口一致的 cross-field 校验。
2. 为 `manifestDigest`、`objectHash` 等内容地址字段补齐语义级校验。
3. 对环境敏感的构建集成测试补齐稳定化措施。

## 下一轮实施顺序

### Task 1: 更新接口校验对齐

- 修改 `RepositoryUpdateRequest`
- 视需要补充共享 validator，避免 create/update 规则漂移
- 补充 `RepositoryApiTest`
- 验收：
  - patch 无法写入缺失 `gitUrl` 的 GIT 配置
  - patch 无法写入缺失 remote 的复制配置

### Task 2: 内容地址字段校验

- 修改 upload / release 相关 request DTO
- 明确 `manifestDigest`、`objectHash`、`etag` 的合法格式
- 补充 API 回归测试
- 验收：
  - 非法哈希或摘要直接返回 `400`
  - 合法现有用例不被误伤

### Task 3: 测试稳定性治理

- 排查并收敛 Gradle 构建用例的环境敏感点
- 为多语言真实构建链路补齐更稳定的 fixture 与超时边界
- 验收：
  - 全量测试可稳定重复通过
  - 关键构建链路无显著偶发失败

## 回写要求

- `/docs/review/00-当前实现Review总览.md`：更新本轮发现和状态
- `/docs/review/01-bug清单与修复方案.md`：补充 RV-008 和后续 RV-009 草案
- `/docs/review/02-下一轮优化计划.md`：滚动更新剩余项
- `/docs/架构/`：补充仓库配置校验与内容地址字段约束规范

