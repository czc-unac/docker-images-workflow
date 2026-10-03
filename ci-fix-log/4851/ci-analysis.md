# CI 失败分析报告

## 基本信息
- PR: #4851 — 【自动升级】milvus容器镜像升级至3.0.2版本.
- 失败类型: build-error
- 置信度: 低
- 知识库匹配: 模式42（日志缺失无法定位）
- 新模式标题: (不适用，已匹配模式42)
- 新模式症状关键词: (不适用，已匹配模式42)

> **前置说明（证据状态）**：本次上下文 `ci.logs` 为 `(not available — analyze based on PR diff only)`，
> `ci.run_info` 为 `(not available)`。**没有任何 CI 日志可供分析**，因此无法确定失败发生的阶段、命令与
> 第一条真实错误。按角色约束，本报告将标注"证据不足"，所有根因均只能来自 `pr.diff` 的静态推断，
> 不能作为确诊结论。下文各方向均为待验证假设，请以获取实际日志后为准。

## 根因分析

### 直接错误
无。上下文未提供任何日志，不存在可引用的错误信息。

### 根因定位
- 失败位置: 未知（无日志）
- 失败原因: 无法确定。仅能确认本 PR 新增了一个全新镜像构建单元，属于"新增镜像自动升级"类改动，典型失败点集中在：镜像许可证头校验、上游版本/tag 是否存在、Dockerfile 构建阶段依赖与产物路径。

### 与 PR 变更的关联
本 PR 共改动 4 个文件：

1. **新增** `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`（53 行，全新文件，多阶段构建）。
2. **修改** `Database/milvus/README.md`：新增 `3.0.2-oe2403sp4` 表格行。
3. **修改** `Database/milvus/doc/image-info.yml`：新增同一 tag 行。
4. **修改** `Database/milvus/meta.yml`：新增 `3.0.2-oe2403sp4: path: 3.0.2/24.03-lts-sp4/Dockerfile`。

该改动本身是元数据与 Dockerfile 的配套新增，无删除逻辑。若 CI 失败，最可能由以下 diff 可观察特征触发（均为待验证假设）：

- **许可证头缺失（对照模式17）**：新增的 `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile` 正文
  直接以 `ARG BASE=openeuler/openeuler:24.03-lts-sp4` 开头，**未包含** `# Copyright ... All rights reserved.`
  与 `# SPDX-License-Identifier: MulanPSL-2.0` 头。CI 的 `check_package_license` 校验对新增文件逐个检查，缺失时预检阶段即失败。
- **上游版本/tag 不存在（对照模式02/19/42）**：Dockerfile 使用 `git clone -b v${VERSION} https://github.com/milvus-io/milvus.git`
  （`VERSION=3.0.2`），要求上游存在 `v3.0.2` tag。若该 tag 不存在或为空，构建会以 `exit 128` 失败。
  本仓库历史上有 milvus `3.0-beta` 被视为不稳定版本的先例（模式11/历史 PR #2269）。
- **构建阶段依赖/产物路径**：Dockerfile 中 `./scripts/install_deps.sh`、`make build-cpp`、`make build-go`，
  以及后续 `COPY --from=builder /milvus/internal/core/output/lib64/`、`/milvus/internal/core/output/lib/*.so*`
  等路径依赖 Milvus 3.0.2 上游实际产物布局。若上游布局与 2.x 不同，会出现路径不存在或编译错误（对照模式10/12）。
- **`ENV PATH=$PATH:/milvus/bin/` 自引用**：BuildKit 可能产生 `UndefinedVar`（对照模式20），但只属警告，一般非致命失败原因。

以上特征都只能解释"可能失败"，无法替代真实日志。**不能据此断定根因。**

## 修复方向

### 方向 1（置信度: 低）— 补齐新增文件的版权/SPDX 头
参照模式17，为新增的 `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile` 添加 openEuler 仓库要求格式的
Copyright 与 SPDX-License-Identifier 头。该方向只能解释预检类失败，若日志显示失败发生在 Docker build
阶段则不成立。

### 方向 2（置信度: 低）— 核实上游 milvus `v3.0.2` tag/tag 名格式
若日志出现 `Remote branch ... not found`、`couldn't find remote ref`、`exit code: 128` 等，则为版本/tag 问题
（模式02/19/42）。需确认 `v3.0.2` 是否为上游正确 tag 名。

### 方向 3（置信度: 低）— 核对构建依赖与产物路径
若日志出现 `Could NOT find`、`No such file or directory`、`failed to calculate checksum`、编译报错等，
需按实际报错补齐 `-devel` 包或修正 `COPY` 源路径（模式10/12/6）。

> 上述三个方向互斥，**在拿到真实日志前不应同时套用**。禁止在无日志情况下直接修改 Dockerfile。

## 需要进一步确认的点
1. **获取本次 CI 的实际失败日志**（trigger/编排层之外的架构构建 job，如 x86-64 / aarch64 的 build job）。
2. 确认失败发生在哪个阶段：许可证预检、YAML/元数据预检，还是 Docker build。
3. 若为 Docker build：确认第一条 error 出现在哪个 `RUN` 步骤。
4. 核实上游 `milvus-io/milvus` 是否存在 `v3.0.2` tag，以及 3.0.2 是否为稳定发布版本。
5. 确认 `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile` 是否为唯一需要补版权头的新增文件。

## 修复验证要求
证据不足（置信度低），code-fixer **必须先获取并阅读真实 CI 日志**，确认失败阶段与第一条错误后，
再选择对应方向修改；**不得**仅凭本报告假设直接改动。若日志确认失败发生在 Docker build 阶段，
需对照上游 milvus 3.0.2 源码实际目录布局验证 Dockerfile 中的依赖安装命令与 `COPY` 产物路径。
