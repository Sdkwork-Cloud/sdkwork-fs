# Rust 技术底座与开源集成策略

## 1. 目标

在保证全 Rust 主体架构的前提下，优先复用成熟开源库，减少重复开发、降低协议实现风险并提升稳定性。

## 2. 选型原则

- 优先选择社区活跃、文档完整、版本迭代稳定的 Rust 库
- 优先选库级集成，而不是外部命令拼装
- 对高风险协议和系统能力保持清晰封装层
- 不把平台差异扩散到领域层

## 3. 推荐依赖

### 3.1 网关与 API

- `tokio`
- `axum`
- `tower`
- `tower-http`
- `rustls`
- `serde`
- `utoipa`

### 3.2 数据与事件

- `sqlx`
- `tonic`
- `prost`
- `async-nats`

### 3.3 Git

- `gitoxide` 生态
- `gix`
- `gix-transport`
- `gix-protocol`
- `russh`

说明：

- `gitoxide` 是项目生态名
- `gix` 是应用程序集成主入口
- `russh` 用于 Git over SSH

### 3.4 对象存储

- `aws-sdk-s3`
- `aws-config`
- 可选：`opendal`

建议：

- 面向 S3 协议、预签名、multipart、ETag 校验，优先 `aws-sdk-s3`
- 需要统一抽象多后端读写时，再考虑在内部引入 `opendal`

### 3.5 缓存与节点

- `moka`
- `dashmap`
- `notify`
- `sha2`
- `zstd`
- `async-compression`
- `fuser`

### 3.6 安全与权限

- `casbin`
- `argon2`
- `jsonwebtoken`
- `secrecy`
- `zeroize`

### 3.7 可观测性

- `tracing`
- `tracing-subscriber`
- `opentelemetry`
- `tracing-opentelemetry`

### 3.8 CLI

- `clap`
- `reqwest`

## 4. 不建议自研的部分

以下能力不建议从零手写：

- Git pack 协议完整实现
- SSH 协议栈
- S3 签名和 multipart 细节
- OpenAPI 生成
- 权限引擎
- 结构化 tracing / metrics 基础能力

## 5. 建议封装层

为了降低三方库直接扩散，建议增加以下内部封装：

- `sdkworkfs-git`
  - 屏蔽 `gix` 细节
- `sdkworkfs-object`
  - 屏蔽 `aws-sdk-s3`
- `sdkworkfs-security`
  - 屏蔽鉴权、授权和凭据实现
- `sdkworkfs-observability`
  - 屏蔽 tracing 和 metrics 配置

## 6. 当前结论

本项目的 Rust 技术底座推荐为：

- API：`axum`
- DB：`sqlx`
- Event：`async-nats`
- Git：`gitoxide/gix`
- SSH：`russh`
- S3：`aws-sdk-s3`
- Cache：`moka`
- Mount：`fuser`
- RPC：`tonic`
- Observability：`tracing + opentelemetry`
