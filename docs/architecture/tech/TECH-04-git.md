> Migrated from `docs/架构/04-Git与对象存储双源机制.md` on 2026-06-24.
> Owner: SDKWork maintainers

# Git 与对象存储双源机制

## 1. 设计原则

`sdkworkfs` 必须同时支持 Git-first 和 S3-first 两种内容入口，但两者都要在发布前统一收敛为 Release。

## 2. Git-first 模式

适用场景：

- React、Node.js、Java、Python 源码仓库
- 基于主干开发的持续集成
- 需要 tag 发布和代码审计的环境

流程：

```text
Git push / Git pull mirror
-> 仓库识别与权限校验
-> 规则匹配
-> 可选构建
-> 过滤与打包
-> 生成 Manifest
-> 写入对象存储
-> 生成 Release
```

## 3. S3-first 模式

适用场景：

- 已有构建产物
- 静态资源包
- 大型数据集
- 二进制发布包

流程：

```text
对象上传 / 目录导入
-> 对象索引
-> 生成 Manifest
-> 生成 Release
```

## 4. 统一收敛规则

- Git 是协作入口之一，不是运行时格式
- 对象存储是对象承载层，不是唯一版本语义层
- Release 是唯一运行时发布单元

## 5. Git 工作流规则

支持三种模式：

- `trunk-based`
- `branch-based`
- `mixed`

默认值：

- 开发主干：`main`
- 生产发布：`tag/release`

## 6. 仓库与对象映射

Git-first 发布必须记录：

- 仓库地址
- 分支
- tag
- commit
- 构建规则
- 过滤规则
- 生成的 ReleaseId

S3-first 发布必须记录：

- 存储地址
- bucket
- prefix
- 导入批次
- 生成的 ReleaseId

## 7. Git 与对象版本差异

| 维度 | Git | 对象存储 |
| --- | --- | --- |
| 主标识 | commit / tag / ref | object key / version / hash |
| 主要作用 | 协作、审计、触发发布 | 内容存放、分发、回源 |
| 是否可直接运行 | 否 | 否 |
| 是否可直接切换 | 否 | 否 |
| 是否需要收敛为 Release | 是 | 是 |

## 8. 双源冲突原则

- 同一环境同一应用同一时刻只能激活一个 Release
- Git-first 和 S3-first 可同时存在，但不能在同一发布流水线上混淆真相源
- 必须显式声明每个发布通道使用哪种入口

## 9. Git 同步与对象同步的边界

- Git 同步负责仓库内容和引用传播
- 对象同步负责 Release 对象和清单传播
- 不允许把 Git refs 与对象版本切换逻辑耦合成一套隐藏规则

## 10. S3 标准要求

- 使用 SigV4
- 支持 `HeadObject`
- 支持 `GetObject` 与 `Range`
- 支持 `PutObject`
- 支持 Multipart Upload
- 支持 `ListObjectsV2`
- 支持自定义 `endpoint` 与 `path-style`

## 11. 正式要求

- Git-first 和 S3-first 的发布结果必须使用同一套 Manifest 模型
- 任一 Release 的对象集合必须可由对象哈希完整重建
- 任一节点不得直接把 Git 仓库目录当成最终运行目录

