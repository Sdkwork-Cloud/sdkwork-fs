> Migrated from `docs/架构/35-Release Builder与构建发布流水线详细设计.md` on 2026-06-24.
> Owner: SDKWork maintainers

# Release Builder 与构建发布流水线详细设计

## 1. 目标

定义 `sdkworkfs` 如何把 Git、对象存储、本地上传等不同来源统一收敛为标准 Release，包括内容解析、过滤、构建、Manifest 生成、对象上传、校验和发布入库全流程。

## 2. 输入来源

支持以下输入：

- Git 仓库指定 `branch/tag/commit`
- 对象存储指定前缀或对象清单
- 本地目录上传
- 构建产物上传

每种输入最终都必须被标准化为：

- `source metadata`
- `normalized file tree`
- `Manifest`
- `Release`

## 3. 参与服务与 crate

- `sdkworkfs-control`
  - 持有 Repository、Application、Release、Manifest、Policy 真相
- `sdkworkfs-job-runner`
  - 执行长任务、重试、超时控制与审计回报
- `sdkworkfs-build-core`
  - Builder 总编排、过滤、Manifest 生成与验证
- `sdkworkfs-build-nodejs`
  - Node.js / React 受控构建
- `sdkworkfs-build-java`
  - Java 受控构建
- `sdkworkfs-build-python`
  - Python 受控构建
- `sdkworkfs-git`
  - Git snapshot、bare repo 访问、mirror 只读视图
- `sdkworkfs-object`
  - 对象上传、去重、预签名、HEAD 校验

## 4. Builder 子阶段

### 4.1 Source Resolver

职责：

- 解析 Git / S3 / Upload 来源
- 拉取或接收输入内容
- 记录来源元信息

要求：

- Git 来源必须固定到明确 revision，不允许对浮动分支直接构建正式发布
- Git 构建优先读取受管 bare repo，再按策略决定是否访问上游远端
- Upload 与 S3 Import 进入 Builder 前必须完成对象登记或来源扫描

### 4.2 Filter Engine

职责：

- 应用多语言排除规则
- 应用应用级覆盖规则
- 输出最终候选文件树

要求：

- 支持 `contextDir`、`include`、`exclude`、`buildOutput`
- 默认过滤 `.git/`、语言缓存目录和本地依赖目录
- 路径必须规范化、稳定排序、禁止越界路径

### 4.3 Build Adapter

职责：

- 对需要构建的项目执行受控构建
- 识别构建产物
- 生成可发布目录

要求：

- 只允许固定模板和白名单工具链，不接受仓库内任意 shell 指令
- 构建必须运行在隔离工作目录，不得污染 bare repo 与运行目录
- 构建日志、超时、退出码、工具链版本必须可审计

### 4.4 Manifest Generator

职责：

- 规范化路径
- 计算对象哈希
- 分类对象类型
- 生成 Manifest

要求：

- 对象哈希统一使用 `SHA-256`
- Manifest digest 只能基于规范化后内容计算
- `entries` 必须稳定排序，避免相同输入得到不同 digest

### 4.5 Object Registrar

职责：

- 上传对象
- 写入对象索引
- 做内容去重

要求：

- 相同对象哈希必须复用同一 `storage_object`
- 上传失败不能产生“对象缺失但 Release 已可见”的状态
- 大对象、压缩对象和 LFS 对象必须走同一内容寻址规则

### 4.6 Release Registrar

职责：

- 写入 Manifest 元数据
- 创建候选 Release
- 绑定来源、环境与策略

要求：

- Release 必须记录 source type、source revision、builder profile、toolchain fingerprint
- 候选 Release 创建后必须进入统一 `verify -> publish -> activate -> rollback` 生命周期

## 5. 多语言构建规则

### 5.1 Node.js / React

- 识别 `pnpm`、`npm`、`yarn`
- 默认构建目标：`dist/`、`build/`、`.next/standalone`
- 固定模板：
  - `pnpm install --frozen-lockfile && pnpm build`
  - `npm ci && npm run build`
  - `yarn install --frozen-lockfile && yarn build`
- 不允许把 `node_modules/` 作为正式上传主体

### 5.2 Java

- 识别 Maven / Gradle
- 默认产物：`jar` / `war`
- 固定模板：
  - `mvn -B -DskipTests package`
  - `gradle --no-daemon assemble -x test`
- 必须保留受控配置目录和启动脚本

### 5.3 Python

- 识别 `pyproject.toml` 或 `setup.py`
- 固定模板：`python -m build --wheel --sdist`
- 受支持产物：`dist/*.whl`、`dist/*.tar.gz`
- 仅有 `requirements.txt` 不构成受控构建条件

## 6. 标准流水线

```text
source resolve
-> filter
-> build
-> artifact normalize
-> object hash
-> manifest generate
-> object upload
-> release persist
-> verify
```

Git 构建推荐链路：

```text
repository revision resolve
-> open managed bare repo
-> create isolated workspace
-> optional controlled build
-> select buildOutput or runtime-default artifacts
-> object dedupe / upload
-> manifest register
-> release persist
```

## 7. 可重复构建要求

- 相同输入、相同构建配置应尽量生成相同 Manifest
- Builder 环境版本应可追踪
- Release 必须记录 Builder 版本、构建开始时间、结束时间、输入版本
- Git 快照 checkout 前应显式固定行尾、文件模式和路径规范化规则

## 8. 校验要求

至少校验：

- 文件路径合法性
- 对象哈希
- 大小与 MIME 类型
- 必要产物是否存在
- 入口文件是否存在
- 受控构建是否满足工具链和模板条件

发布前缺少必要产物时必须失败，不允许拖到节点运行阶段暴露。

## 9. 构建隔离与安全

- 构建步骤必须运行在隔离工作目录
- 不应复用正式运行目录作为构建目录
- 构建输出必须再次经过过滤与规范化
- 可选集成恶意内容扫描、许可证扫描和 SBOM 生成
- 敏感凭据只能通过短期票据或进程级 secret 注入，不得写回仓库

## 10. 失败处理

常见失败类型：

- 来源不可达
- 构建失败
- 产物缺失
- 哈希校验失败
- 对象上传失败
- 元数据写入失败

要求：

- 每个阶段失败都必须进入 `operation` 记录
- 必须保留失败摘要与错误码
- 可重试阶段与不可重试阶段要区分清楚
- 自动触发与人工触发必须共享同一条 Builder 审计模型

## 11. 正式要求

- Release Builder 必须把不同来源统一收敛到 `Release + Manifest`
- 多语言规则、构建适配、路径过滤、哈希校验必须标准化
- 任何构建失败、产物缺失或校验失败都不得进入正式发布

