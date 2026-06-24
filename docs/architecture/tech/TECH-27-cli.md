> Migrated from `docs/架构/27-CLI与节点命令规范.md` on 2026-06-24.
> Owner: SDKWork maintainers

# CLI 与节点命令规范

## 1. 目标

统一定义 `sdkworkfs` 的命令行工具、节点守护进程接口、通用参数、输出格式和退出码，使本地运维、自动化流水线和节点自检都围绕一致命令集执行。

## 2. 推荐二进制角色

- `sdkworkfs`：控制面与本地节点 CLI
- `sdkworkfsd`：节点守护进程

`sdkworkfs` 应优先调用本地守护进程完成节点操作，仅在需要控制面资源管理时访问远端 API。

## 3. 核心命令

建议至少支持：

- `upload`
- `publish`
- `mount`
- `sync`
- `prefetch`
- `switch`
- `rollback`
- `gc`
- `status`
- `logs`

## 4. 通用参数约定

建议所有命令统一支持：

- `--config`
- `--profile`
- `--request-id`
- `--timeout`
- `--output text|json`
- `--node-id`
- `--app-id`
- `--environment`

参数风格建议使用长参数，避免位置参数歧义。

## 5. 守护进程通信

本地 CLI 与守护进程推荐支持：

- Unix domain socket
- Windows named pipe
- loopback gRPC 或 HTTP

通信要求：

- 默认本地访问，不对公网暴露
- 命令必须返回结构化错误
- 超时、重试和取消必须有明确边界

## 6. 命令样例

发布：

```text
sdkworkfs publish --app-id app-demo --environment staging --source git --ref refs/tags/v1.0.0 --output json
```

切换：

```text
sdkworkfs switch --app-id app-demo --environment prod --release-id rel_20260406_001 --mode canary
```

节点状态：

```text
sdkworkfs status --node-id node-a --output json
```

## 7. 输出模型

文本输出适合人工阅读，JSON 输出适合自动化系统。JSON 输出建议最小结构如下：

```json
{
  "requestId": "req-001",
  "code": "OK",
  "data": {
    "operationId": "op-123",
    "status": "RUNNING"
  }
}
```

## 8. 退出码建议

| 退出码 | 含义 |
| --- | --- |
| `0` | 成功 |
| `2` | 参数错误 |
| `3` | 认证或授权失败 |
| `4` | 资源不存在 |
| `5` | 冲突或状态不允许 |
| `6` | 网络或上游依赖失败 |
| `7` | 节点执行失败 |
| `8` | 超时 |

## 9. 正式要求

- 节点命令必须支持结构化 JSON 输出
- 关键命令必须支持 `requestId` 与幂等能力
- 本地 CLI 与守护进程接口必须与控制面 API 错误码保持可映射关系

