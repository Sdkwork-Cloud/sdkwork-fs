> Migrated from `docs/架构/32-配置示例与环境模板.md` on 2026-06-24.
> Owner: SDKWork maintainers

# 配置示例与环境模板

## 1. 目标

在 [12-配置模型与默认值.md](12-配置模型与默认值.md) 的默认规则基础上，给出可直接照抄修改的模板，降低首次部署和多环境治理的配置歧义。

## 2. 本地开发模板

```yaml
platform:
  mode: local-dev

git:
  workflow:
    mode: trunk-based
    mainBranch: main

storage:
  type: s3
  endpoint: http://127.0.0.1:9000
  region: local
  bucket: sdkworkfs-dev
  accessKey: sdkworkfs
  secretKey: sdkworkfs123
  pathStyle: true
  presignExpirySeconds: 900
  autoCreateBucket: true
  failFastOnMissingBucket: true
  verifyOnComplete: true

cache:
  metadata:
    maxEntries: 50000
  memory:
    maxBytes: 256MB
    maxFileSize: 1MB
  objectDisk:
    maxBytes: 50GB
  materialized:
    keepReleases: 2

cluster:
  mode: single-primary
  nodeRole: primary

node:
  sync:
    mode: eager
```

说明：

- 本地开发默认保持 `sdkworkfs.build.node.enabled=false`、`sdkworkfs.build.java.enabled=false` 与 `sdkworkfs.build.python.enabled=false`
- 若需要在本地验证 Node.js / React、Java 或 Python 真实构建，请配合下文的“真实构建模板”显式开启，并确保宿主机或虚拟环境已安装对应工具链

## 3. 单主生产模板

```yaml
platform:
  mode: production

git:
  workflow:
    mode: trunk-based
    mainBranch: main
    prodSource: tag
  replication:
    mode: mirror
    enabled: true

storage:
  type: s3
  endpoint: https://s3.company.internal
  region: ap-east-1
  bucket: sdkworkfs-prod
  accessKey: ${S3_ACCESS_KEY}
  secretKey: ${S3_SECRET_KEY}
  pathStyle: false
  presignExpirySeconds: 900
  autoCreateBucket: false
  failFastOnMissingBucket: true
  verifyOnComplete: true

cache:
  metadata:
    maxEntries: 200000
  memory:
    maxBytes: 1GB
    maxFileSize: 2MB
  objectDisk:
    maxBytes: 500GB
  materialized:
    keepReleases: 4

cluster:
  mode: single-primary
  releasePropagation: event-driven
  failoverPolicy: manual

limits:
  maxConcurrentPrefetch: 8
  maxConcurrentWriteback: 16
```

## 4. 双区域容灾模板

```yaml
platform:
  mode: production-dr

git:
  replication:
    mode: mirror
    enabled: true
    loopPrevention: true

storage:
  type: s3
  region: ap-east-1
  primaryBucket: sdkworkfs-prod-a
  mirrorBucket: sdkworkfs-prod-b
  presignExpirySeconds: 900
  autoCreateBucket: false
  failFastOnMissingBucket: true
  verifyOnComplete: true

cluster:
  mode: single-primary
  failoverPolicy: controlled
  leaderLeaseTtl: 15s
  heartbeatInterval: 5s
  heartbeatTimeout: 20s

disasterRecovery:
  enabled: true
  metadataBackupInterval: 5m
  objectReplication: async
```

## 5. Node.js / React 真实构建模板

```yaml
sdkworkfs:
  build:
    workspaceRootDir: /var/lib/sdkworkfs/build/workspaces
    commandPathPrepend: /opt/node/bin
    node:
      enabled: true
      installTimeoutSeconds: 300
      buildTimeoutSeconds: 300
```

说明：

- 仅在控制面宿主机或自定义镜像已提供 `pnpm` / `npm` / `yarn` 时开启
- `commandPathPrepend` 用于把工具目录前置到 `PATH`
- 标准 Docker Compose 示例默认保持 `enabled=false`，且默认镜像不内置 Node.js toolchain

## 6. Java 真实构建模板

```yaml
sdkworkfs:
  build:
    workspaceRootDir: /var/lib/sdkworkfs/build/workspaces
    commandPathPrepend: /opt/build-tools/bin
    java:
      enabled: true
      mavenTimeoutSeconds: 600
      gradleTimeoutSeconds: 600
```

说明：

- 仅在控制面宿主机或自定义镜像已提供 `mvn` / `gradle` 时开启
- `commandPathPrepend` 可同时服务 Node.js / React、Java 与 Python 的工具目录前置
- 标准 Docker Compose 示例默认保持 `enabled=false`，且默认镜像不内置 Maven / Gradle toolchain

## 7. Python 真实构建模板

```yaml
sdkworkfs:
  build:
    workspaceRootDir: /var/lib/sdkworkfs/build/workspaces
    commandPathPrepend: /opt/python/bin
    python:
      enabled: true
      buildTimeoutSeconds: 600
```

说明：

- 仅在控制面宿主机、自定义镜像或虚拟环境已提供 `python`，且对应解释器已安装 `build` 模块时开启
- 当前只把 `pyproject.toml` 或 `setup.py` 视为真实构建描述文件；`requirements.txt` 只用于依赖声明，不触发真实构建
- `commandPathPrepend` 可与虚拟环境 `Scripts/` 或 `bin/` 目录配合，稳定解析 `python`
- 标准 Docker Compose 示例默认保持 `enabled=false`，且默认镜像不内置 Python toolchain 或 `build` 模块

## 8. 多语言应用模板

```yaml
repositories:
  - repositoryId: web-portal-repo
    source:
      gitUrl: https://git.example.com/web-portal.git
      buildOutput: dist/
  - repositoryId: order-service-repo
    source:
      gitUrl: https://git.example.com/order-service.git
      buildOutput: target/*.jar
  - repositoryId: ml-worker-repo
    source:
      gitUrl: https://git.example.com/ml-worker.git
      buildOutput: dist/*.whl
applications:
  - appId: web-portal
    runtimeType: nodejs
    deliveryMode: sync
    repositoryId: web-portal-repo
  - appId: order-service
    runtimeType: java
    deliveryMode: hybrid
    repositoryId: order-service-repo
  - appId: ml-worker
    runtimeType: python
    deliveryMode: sync
    repositoryId: ml-worker-repo
```

## 7. 环境策略模板

```yaml
environments:
  dev:
    sourcePolicy:
      mode: trunk-head
    approvalPolicy:
      mode: none
  test:
    sourcePolicy:
      mode: protected-branch
    approvalPolicy:
      mode: single
  prod:
    sourcePolicy:
      mode: release-tag
    approvalPolicy:
      mode: dual
    deploymentPolicy:
      mode: canary
```

## 8. 节点模板

```yaml
node:
  nodeId: node-sh-01
  role: edge
  region: cn-shanghai
  mount:
    enabled: true
    mountPoint: /srv/sdkworkfs/mnt
  paths:
    root: /var/lib/sdkworkfs
    cacheRoot: /var/cache/sdkworkfs
    materializedRoot: /var/lib/sdkworkfs/materialized
```

Storage policy note:
- Local development keeps `autoCreateBucket: true` for zero-bootstrap MinIO startup.
- Production and DR templates pin `autoCreateBucket: false` and `failFastOnMissingBucket: true` so bucket readiness is an explicit deployment gate.

## 9. 正式要求

- 配置模板必须和默认值规范、部署规范保持一致
- 模板中的关键字段必须支持平台级、应用级、环境级覆盖
- 所有生产模板必须显式定义缓存、发布来源、审批和故障切换策略
- 生产模板中的 `secretKey` 必须通过密钥管理或环境变量注入，不得硬编码

