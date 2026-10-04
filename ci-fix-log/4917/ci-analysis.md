# CI 失败分析报告

## 基本信息
- PR: #4917 — 【自动升级】ceph容器镜像升级至21.3.0版本.
- 失败类型: build-error（待确认）
- 置信度: 低
- 知识库匹配: 模式19（证据不足 / 无法定位根因）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
无可用日志。上下文中 `ci.run_info` 与 `ci.logs` 均为 `(not available)`，
未提供任何构建输出（无 `docker build` 步骤日志、无错误堆栈、无成功/失败标志）。
因此无法从日志中提取"最早出现的错误信息"，也无法确认失败发生在哪个构建阶段。

### 根因定位
- 失败位置: 未知（CI 日志缺失，只能基于 `pr.diff` 推断）
- 失败原因: 无法确认。日志不足以定位具体错误。

### 与 PR 变更的关联
本次 PR 为自动升级单，新增 `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile` 及
`entrypoint.sh`，并更新 `meta.yml`、`README.md`、`doc/image-info.yml`。
在仅有 diff 的前提下，代码层面存在以下**可疑点**（均需日志验证，不能作为定论）：

1. **上游 tag 是否存在（最可疑）**：Dockerfile 中
   `ARG VERSION=21.3.0` + `git clone -b v${VERSION} ... https://github.com/ceph/ceph.git`。
   若上游 `ceph/ceph` 不存在 tag `v21.3.0`，`git clone -b` 会以
   `Remote branch v21.3.0 not found in upstream origin` / `exit code: 128` 失败
   （对应知识库 模式22「Git 分支名构造错误」、模式02/模式19「自动升级使用上游不存在的版本」）。
   Ceph 主版本号历史惯用 `X.2.0`（18.2、19.2、20.2），`21.3.0` 的 `.3` 段位较为反常，
   存在自动升级工具生成不存在版本号的可能（同类案例：PR #4852 jetty 12.1.14、#4845 rabitq-library 0.5.1、#4838 openfoam 20260907）。
2. **libnbd 依赖构建**：`git clone https://gitlab.com/nbdkit/libnbd.git` 后
   `autoreconf -fi && ./configure && make`，若 `gnutls-devel`/`libxml2-devel` 等缺失或
   `autoreconf` 未安装完整，会在 configure 阶段失败（知识库 模式10）。本次 diff 已包含相关
   `-devel` 包，故该点风险较低，但仍需日志确认。
3. **`ENV LD_LIBRARY_PATH=/usr/local/lib64:$LD_LIBRARY_PATH`** 自引用未定义变量，
   会产生 BuildKit `UndefinedVar` **警告**（模式20）。该警告非致命，**不应**被视为根因。
4. **运行时入口**：`entrypoint.sh` 依赖 `$BUILD_DIR/bin/*`，若构建阶段未产出对应二进制，
   可能在 CI 的容器启动检查阶段失败（模式25）。但这属于运行阶段，与构建失败状态未必相关。

由于未提供日志，上述 1–4 均为"候选方向"而非结论。真正的根因必须由失败 job 的日志确定。

## 修复方向

### 方向 1（置信度: 低）
若失败为 `git clone -b v21.3.0` 报 `Remote branch ... not found`：确认上游
`ceph/ceph` 是否存在对应 tag，纠正自动升级使用的版本号为上游真实发布版本
（同时按需同步 `meta.yml` 中的 tag 目录名与文档版本描述）。

### 方向 2（置信度: 低）
若失败为 libnbd / ceph 构建阶段的依赖或编译错误：依据失败 job 日志定位缺失的
`-devel` 包或编译参数，再补充依赖。此方向目前缺少任何日志依据，仅为备选。

> 说明：本次无法确定唯一根因，方向 1、2 均未经日志验证。

## 需要进一步确认的点
1. 获取真正失败 job 的完整日志。README 声明该镜像 `amd64, arm64` 双架构，
   构建通常在架构专属 job（如 `/job/x86-64/…`、`/job/aarch64/…`）中执行，
   而 trigger/编排层 job 不一定携带真实错误。需拉取对应架构构建 job 的日志。
2. 确认 `ci.logs` 末尾是否出现 `Finished: SUCCESS` / `Build successful`。
   若日志实际来自 trigger 层且显示成功，而 PR 仍带 `ci_failed`，则应改判为
   `infra-error`（证据不足），本报告结论需相应调整。
3. 核实上游 `https://github.com/ceph/ceph` 是否存在 tag `v21.3.0`
   （以 Dockerfile `ARG VERSION` 为准），以及 `image-info.yml` 中版本号与 tag 的映射关系。
4. 确认是否为双架构中**单一架构**失败（例如 aarch64 专属问题），以缩小范围。

## 修复验证要求（仅当修复涉及正则 patch 外部源文件时填写）
本 PR 的修复方向暂不涉及对第三方源文件的正则 patch，故不适用。

---
**结论**：证据不足。在未获取失败架构构建 job 日志前，**禁止**将本报告中的任何候选方向
当作确定根因。修复前必须先取得真实失败日志，或先验证上游 tag `v21.3.0` 是否存在。
