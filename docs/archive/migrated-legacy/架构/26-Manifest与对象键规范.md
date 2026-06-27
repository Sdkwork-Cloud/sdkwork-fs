# Manifest 与对象键规范

## 1. 目标

统一定义 `sdkworkfs` 的 Manifest 格式、对象键命名、哈希标准、压缩规则和对象分类，确保 Git、对象存储、本地缓存和节点物化都围绕同一套内容寻址规则运行。

## 2. Manifest 角色

Manifest 是 Release 的内容清单，是 Git 版本、对象版本和运行时物化目录之间的桥梁。Manifest 必须不可变、可校验、可重放、可比对。

## 3. Manifest 最小结构

建议最小结构如下：

```json
{
  "schemaVersion": "v1",
  "tenantId": "100001",
  "appId": "app-demo",
  "releaseId": "rel_20260406_001",
  "manifestDigest": "sha256:abcd...",
  "source": {
    "type": "git",
    "version": "refs/tags/v1.0.0",
    "commit": "abcdef123456"
  },
  "entries": [
    {
      "path": "dist/index.js",
      "kind": "static",
      "contentHash": "sha256:1234...",
      "size": 1024,
      "storageEncoding": "zstd",
      "contentType": "application/javascript",
      "mode": "0644"
    }
  ],
  "createdAt": "2026-04-06T00:00:00Z"
}
```

## 4. Manifest 规则

- `entries` 必须按规范化路径排序，避免同一内容生成不同哈希
- `path` 必须使用相对路径，不允许绝对路径和 `..`
- `contentHash` 必须基于原始内容计算，不受压缩方式影响
- `manifestDigest` 必须基于规范化后的 Manifest 内容计算
- 相同对象可被多个 Release 复用，但同一路径在同一 Release 内只允许出现一次

## 5. 对象键命名规则

推荐对象键格式：

```text
objects/sha256/ab/cd/abcdef1234567890...
```

推荐 Manifest 键格式：

```text
manifests/<tenantId>/<appId>/<releaseId>/<manifestDigest>.json
```

说明：

- 对象以内容哈希寻址，便于去重和缓存复用
- 前缀分片应至少保留两级目录，降低单目录热点
- 不建议把分支名、环境名直接作为对象内容键的一部分

## 6. 哈希、压缩与内容类型

- 默认内容哈希标准为 `SHA-256`
- `contentHash` 表示原始内容哈希
- `storageEncoding` 允许 `none`、`gzip`、`zstd`
- `contentType` 应保留 MIME 类型，用于节点物化与网关透传
- 压缩策略不得改变内容哈希，只影响存储占用和回源解压方式

## 7. 对象分类

推荐 `kind` 枚举：

- `static`
- `binary`
- `config`
- `media`
- `model`
- `data`

不同 `kind` 可对应不同缓存、回收和预热策略。

## 8. 校验与版本约束

- 上传阶段必须校验 `contentHash`、大小和 `contentType`
- 物化阶段必须校验 Manifest 中的路径、权限和对象存在性
- Schema 版本升级必须提供显式迁移工具、版本门禁和回滚方案

## 9. 正式要求

- 每个 Release 必须绑定唯一 Manifest
- Manifest 与对象键规则必须稳定、可审计、可重建
- 内容哈希、存储编码、对象类型必须进入 Manifest

## 10. 实施要求

- `POST /manifests` 与 `GET /manifests/{manifestDigest}` 必须围绕同一份规范化 `entries` 工作
- Manifest `entries` 必须按路径排序后持久化到中心库或对象存储索引
- `release_object` 必须基于 Manifest `entries` 派生落库
- `Release verify` 必须至少校验：
  - `schemaVersion == v1`
  - `objectCount > 0`
  - `storageEncoding` 属于 `none` / `gzip` / `zstd`
  - `entries.size == objectCount`
  - `sum(entries.size) == totalSize`
  - `release_object` 与 Manifest `entries` 的路径、哈希和大小一致
- Builder、Upload 与 S3 Import 三类入口都必须最终引用同一套 Manifest 规则
