# systemd、Docker、Kubernetes 部署清单

## 1. 目标

把安装部署规范进一步细化为 systemd 服务、容器部署、Compose、Kubernetes 的落地清单与样例，使平台在不同环境下都有统一部署基线。

## 2. systemd 部署建议

适用：

- 物理机
- 虚拟机
- 直接运行节点守护进程

### 2.1 服务划分建议

- `sdkworkfs-control-plane.service`
- `sdkworkfs-git-gateway.service`
- `sdkworkfsd.service`

### 2.2 `sdkworkfsd` unit 示例

```ini
[Unit]
Description=sdkworkfs node daemon
After=network.target

[Service]
ExecStart=/usr/local/bin/sdkworkfsd --config /etc/sdkworkfs/node.yaml
Restart=always
User=sdkworkfs
Group=sdkworkfs
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

## 3. Docker / Compose 建议

### 3.1 容器划分

- Control Plane
- Git Gateway
- Release Builder
- PostgreSQL
- NATS
- S3 API 标准存储

### 3.2 Compose 最低要求

- 持久卷
- 健康检查
- 明确网络
- 密钥使用环境变量或密钥文件挂载

### 3.3 节点容器注意事项

- 挂载模式需要宿主机权限
- 需要挂载缓存目录、物化目录、日志目录
- 需要额外评估 FUSE 能力和 `CAP_SYS_ADMIN` 等权限

## 4. Kubernetes 建议

### 4.1 工作负载类型

- Control Plane：`Deployment`
- Git Gateway：`Deployment`
- Release Builder：`Deployment` 或 `Job`
- PostgreSQL / NATS / MinIO：托管服务或 `StatefulSet`
- 节点守护进程：`DaemonSet`

### 4.2 最低清单要求

- `ConfigMap`
- `Secret`
- `Service`
- `PersistentVolumeClaim`
- `livenessProbe`
- `readinessProbe`
- `resources`
- `securityContext`

### 4.3 节点 DaemonSet 注意事项

- 需要宿主机目录挂载
- 需要节点选择器与容忍度
- 挂载模式需显式确认 FUSE / WinFSP 支持

## 5. Helm 与模板化建议

- 建议为 Control Plane、Git Gateway、Node Daemon 提供独立 chart 或子 chart
- values 中应显式暴露缓存目录、对象存储、数据库、事件总线、认证配置
- 生产 values 不得依赖开发默认值

## 6. 部署前检查清单

1. 存储卷已创建
2. Secret 已配置
3. 数据库可连接
4. 对象存储 bucket 已就绪
5. 事件总线已就绪
6. 节点目录权限正确

## 7. 正式要求

- systemd、容器、Kubernetes 三类部署方式都必须有最小基线
- 挂载模式必须额外检查宿主机权限和驱动支持
- 任何生产部署模板都必须显式配置持久卷、探针、资源限制和 Secret
