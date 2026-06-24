> Migrated from `docs/架构/40-CI-CD与发布自动化流水线规范.md` on 2026-06-24.
> Owner: SDKWork maintainers

# CI-CD 与发布自动化流水线规范

## 1. 目标

定义 `sdkworkfs` 从代码提交、构建、校验、对象上传、Release 创建、环境晋级到回滚验证的自动化流水线标准，使平台能够稳定支撑多语言项目与多环境部署。

## 2. 流水线分层

至少拆分为：

- CI 校验流水线
- Build / Package 流水线
- Release 创建流水线
- Environment Promote 流水线
- Rollback 验证流水线

## 3. CI 流水线

触发：

- push 到 `main`
- PR / Merge Request
- tag 创建

步骤建议：

1. 代码 checkout
2. 依赖安装
3. 单元测试
4. 静态检查
5. 安全扫描
6. 构建产物
7. 产物归档

## 4. 多语言构建矩阵

| 类型 | 推荐步骤 |
| --- | --- |
| Node.js / React | `install -> lint -> test -> build` |
| Java | `mvn test -> package` 或 Gradle 等价流程 |
| Python | `install -> test -> build wheel` |
| 静态站点 | 资源构建与压缩 |

## 5. Release 创建流水线

建议流程：

```text
构建成功
-> 识别构建产物
-> 上传对象
-> 生成 Manifest
-> 创建 Release
-> 执行 verify
-> 产出 releaseId
```

## 6. 环境晋级流水线

### 6.1 dev

- 可由 `main` 自动触发
- 默认允许无人审批

### 6.2 test / staging

- 建议半自动
- 允许通过流水线参数指定候选 Release

### 6.3 prod

- 必须受审批与环境策略约束
- 建议使用 tag/release 触发
- 建议金丝雀晋级受监控指标驱动

## 7. 建议工具适配

支持但不限于：

- GitHub Actions
- GitLab CI
- Jenkins
- Argo Workflows / Argo CD

要求：

- 流水线工具可替换
- 关键阶段语义保持一致
- 不能把平台规则硬编码到单一 CI 工具脚本中

## 8. 产物与凭据管理

- 构建产物应有唯一版本标识
- 上传对象应使用短期凭据或预签名地址
- 流水线不得明文打印长期密钥
- 产物元信息必须可追踪到 commit、构建时间、构建人或系统主体

## 9. 回滚流水线

至少支持：

- 人工触发回滚
- 告警触发回滚
- 金丝雀失败自动回滚

回滚流水线必须：

- 指定目标稳定版本
- 记录原因
- 验证回滚后健康状态

## 10. 交付物要求

每次正式流水线运行至少要产出：

- 构建日志
- 测试结果
- 安全扫描结果
- Manifest 摘要
- releaseId
- 审批记录或审批引用

## 11. 正式要求

- CI/CD 必须覆盖从提交到 Release 的关键链路
- 环境晋级、审批、金丝雀、回滚必须与平台规则一致
- 流水线脚本、产物和凭据管理必须可审计、可回放、可替换

