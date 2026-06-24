> Migrated from `docs/review/02-下一轮优化计划.md` on 2026-06-24.
> Owner: SDKWork maintainers

# 下一轮优化计划

## 目标

在当前“资源引用正确、自动发布可去重、构建执行更稳、控制面输入约束已闭环、仓库创建可 fail fast”的基础上，下一轮优先推进创建/更新校验口径一致化和测试稳定性治理。

## 优先级顺序

### P0：仓库更新接口与创建接口校验口径对齐

- 为 `RepositoryUpdateRequest` 增加与创建接口一致的 `sourceType` / `replication.enabled` cross-field 校验
- 避免通过 patch 写入缺失 `gitUrl`、缺失 `bucket`、remote 不完整等非法组合
- 目标：创建和更新两个入口对仓库配置采用同一套约束

### P1：内容地址字段与 DTO 语义校验

- 为 `manifestDigest`、`objectHash`、`etag` 等字段增加语义格式校验
- 评估哪些字段必须收敛到标准格式，哪些字段需要保持宽松兼容
- 为非法内容地址补齐标准 `INVALID_ARGUMENT` 返回

### P1：测试稳定性与构建矩阵治理

- 针对 `ReleaseApiTest.testBuildReleaseRunsRealGradleBuildWhenEligible` 这类环境敏感用例补齐稳定化措施
- 增加 Windows / Linux / macOS 的构建命令兼容性验证矩阵
- 补齐 Node、Java、Python 三类真实示例仓库的集成测试

### P2：缓存与构建目录后台治理

- 为对象缓存目录、构建工作区和临时日志目录增加后台清理任务
- 输出缓存命中率、水位、淘汰次数、构建超时次数等指标

### P2：测试工具链与告警收敛

- 处理 Mockito 动态 agent 警告，降低未来 JDK 升级风险
- 评估是否为高风险链路增加 fault injection 与超时重试告警

## 执行方式

1. 继续按“先补失败测试，再改实现”的方式推进。
2. 每完成一项能力都同步更新 `/docs/review` 与 `/docs/架构`。
3. 每轮结束执行定向测试、全量测试、打包与部署配置校验。

