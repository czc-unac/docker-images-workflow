# CI 失败分析报告

## 基本信息
- PR: #4896 — 【自动升级】milvus容器镜像升级至3.0.2版本.
- 失败类型: 证据不足（无法归类；diff 层面可确认的候选偏 `lint-error`/`build-error`）
- 置信度: 低
- 知识库匹配: 模式42（日志缺失无法定位）为主，兼顾 模式19（证据不足）、模式17（Copyright/SPDX 缺失）、模式02/22/28（上游版本/分支不存在）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
（无可用日志）

上下文 `ci.logs` 为 `"(not available — analyze based on PR diff only)"`，`ci.run_info` 为 `"(not available)"`。
本次分析没有任何失败 job 的日志可供引用，因此**无法从日志中提取最早出现的错误信息**，也无法确定失败发生在哪个
阶段（预检 / 构建 / 推送）。按核心约束，判定为**证据不足**，不得凭 diff 直接下结论。

### 根因定位
- 失败位置: 未知（无日志；仅能给出候选文件）
- 失败原因: 无法从日志确认。仅能从 PR diff 静态识别出若干**可疑项**，需日志验证后定性。

从 diff 可静态识别的可疑项（均非已验证根因）：

1. **新增 Dockerfile 缺少 Copyright / SPDX 版权头**（对应 模式17）
   `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile` 为全新文件，第 1 行直接是
   `ARG BASE=openeuler/openeuler:24.03-lts-sp4`，文件内**没有** `# Copyright ...` 与
   `# SPDX-License-Identifier:` 声明。历史案例 PR #2516 表明新增 Dockerfile 缺版权头会触发
   `check_package_license` 预检失败（`lint-error` 类）。

2. **上游 git tag `v3.0.2` 是否真实存在未经验证**（对应 模式02 / 模式22 / 模式28）
   构建阶段执行 `git clone -b v${VERSION} https://github.com/milvus-io/milvus.git`，`VERSION=3.0.2`，
   实际 clone 分支为 `v3.0.2`。若上游无该 tag，构建会以
   `fatal: Remote branch v3.0.2 not found in upstream origin`（exit code 128）失败。
   历史案例（PR #2269）显示该仓库曾存在并被移除 `3.0-beta` 版本条目，说明 Milvus 3.0 系列版本号
   本身存在可用性风险；模式19/42 也记录了多起自动升级 PR 指向不存在版本的情况。

3. **Milvus 3.0.2 的构建方式/产物路径可能与该 Dockerfile 的 2.x 构建指令不匹配**（推断）
   Dockerfile 使用 `./scripts/install_deps.sh && make build-cpp && make build-go`，并
   `COPY --from=builder /milvus/internal/core/output/lib64/`、`/milvus/internal/core/output/lib/`。
   若 3.0 系列的目录结构、Makefile 目标或产物路径发生变化，会在构建阶段失败。此为推断，无日志佐证。

### 与 PR 变更的关联
本 PR 为纯新增自动升级：新增 `3.0.2/24.03-lts-sp4/Dockerfile`，并同步更新 `meta.yml`、
`README.md`、`doc/image-info.yml` 的版本条目。`meta.yml` 新增
`3.0.2-oe2403sp4: path: 3.0.2/24.03-lts-sp4/Dockerfile`，路径为规范的
`{image-version}/{os-version}/Dockerfile` 两级结构，未见 YAML/路径规范错误。
两个 `README.md` / `image-info.yml` 表格新增行格式与现有行一致，未见明显格式问题。
因此，若失败发生在构建阶段，最可能由上述新增 Dockerfile 自身（版权头 / 上游版本 / 构建指令）
引起；但**无日志时不能确认**。

## 修复方向

### 方向 1（置信度: 低）
**补充新增 Dockerfile 的 Copyright / SPDX 头**（若失败发生在预检 `check_package_license`）。
需先确认 CI 是否报 `check_package_license` 类失败；如日志缺失，应先补日志再动手。

### 方向 2（置信度: 低）
**核验上游 `v3.0.2` tag 是否存在**（若失败发生在 `git clone` 阶段）。
在动代码前，必须从上游仓库以 Dockerfile 的 `ARG VERSION=3.0.2` 为准，确认存在可用于 clone 的
tag/分支；若不存在，应改用上游真实存在的版本或修正版本号。

### 方向 3（置信度: 低）
**核验 Milvus 3.0.2 的实际构建指令与产物路径**（若失败发生在 `install_deps.sh` / `make build-cpp` /
`make build-go` / `COPY` 阶段）。对照上游该版本 `scripts/`、Makefile 目标与 `internal/core/output`
路径确认是否与 Dockerfile 一致。

> 说明：以上均为候选方向，无日志不能确定优先级，禁止直接据此提交修改。

## 需要进一步确认的点
1. **获取失败 job 的完整日志**：当前上下文的 `ci.logs` 为空，`ci.run_info` 也为空。必须提供真正失败的
   构建 job 日志（含 `#N RUN ...` 失败步骤与 exit code），才能定性根因。
2. 该 PR 是否携带 `ci_failed` 标签，以及失败发生在预检（license/路径校验）还是 Docker 构建阶段。
3. `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile` 对应的 CI 检查是否包含版权头校验（`check_package_license`）。
4. 上游 `milvus-io/milvus` 是否存在 tag `v3.0.2`；若存在，其 `scripts/install_deps.sh`、
   `make build-cpp`、`make build-go` 目标及产物路径是否与 Dockerfile 一致。
5. 失败是否为单架构（如仅 aarch64）还是双架构，以判断是否为架构相关问题。

## 修复验证要求（仅当修复涉及正则 patch 外部源文件时填写）
本次诊断未给出涉及正则 patch 外部源文件的修复方向；若后续修复方向确认为“修改正则以匹配上游
`install_deps.sh`/Makefile 内容”，则 code-fixer 必须先从 `milvus-io/milvus` 的 `v3.0.2`（以 Dockerfile
`ARG VERSION=3.0.2` 为准）拉取对应文件，验证正则匹配后再提交。
