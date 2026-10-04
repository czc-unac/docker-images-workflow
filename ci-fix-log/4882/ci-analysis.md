# CI 失败分析报告

## 基本信息
- PR: #4882 — 【自动升级】openfoam容器镜像升级至20260907版本.
- 失败类型: build-error（下载/构建阶段失败；根因方向为上游版本号不存在）
- 置信度: 低（`ci.logs` 未提供，仅能依据 `pr.diff` 与历史模式推断）
- 知识库匹配: 模式02（下载 URL 版本路径错误 / 软件包版本不存在）；同时与模式19历史案例 PR #4838 高度对应
- 新模式标题: （不适用，命中已有模式）
- 新模式症状关键词: （不适用）

## 根因分析

### 直接错误
`ci.logs` 为 "(not available — analyze based on PR diff only)"，**没有任何可引用的实际报错日志**。
无法给出真实的错误片段。以下是 diff 中唯一可推断的失败触发点（非日志证据）：

```
+ARG VERSION=20260907
+RUN wget https://sourceforge.net/projects/openfoam/files/v${VERSION}/ThirdParty-v${VERSION}.tgz && \
+    wget https://sourceforge.net/projects/openfoam/files/v${VERSION}/OpenFOAM-v${VERSION}.tgz && \
```

### 根因定位
- 失败位置: `HPC/openfoam/20260907/24.03-lts-sp4/Dockerfile`（新增文件）中 `ARG VERSION=20260907` 及基于该变量的 sourceforge 下载 URL 构造（约第 2、10-11 行）
- 失败原因: 推断为上游 OpenFOAM 不存在 `v20260907` 这一版本，导致 `ThirdParty-v20260907.tgz` / `OpenFOAM-v20260907.tgz` 下载 404，Docker 构建失败。OpenFOAM 的版本号采用 `YYMM`（如 `v2412`、`v2506`、`v2606`），而 `20260907` 形似日期（2026-09-07），并非合法的上游发布版本号。

### 与 PR 变更的关联
强关联。本 PR 为自动升级 PR，新增了 `20260907` 版本目录及 Dockerfile，并同步在 `README.md`、`doc/image-info.yml`、`meta.yml` 中登记 `20260907-oe2403sp4`。
- `HPC/openfoam/meta.yml` 新增 `20260907-oe2403sp4: path: 20260907/24.03-lts-sp4/Dockerfile` 会驱动 CI 对该 Dockerfile 发起构建。
- 一旦 `VERSION=20260907` 在上游不存在，下载步骤即失败。

历史佐证（知识库模式19）: PR #4838 记录了完全相同的路径 `HPC/openfoam/20260907/24.03-lts-sp4/Dockerfile`，结论为"将 openfoam Dockerfile 中不存在的版本 `20260907` 修正为上游真实存在的版本 `2606`"。本 PR 是同一错误版本的再次出现（或自动升级机制复现），因此根因方向高度一致。

## 修复方向

### 方向 1（置信度: 中）
将 `VERSION` 从 `20260907` 修正为上游 OpenFOAM 实际发布的版本（历史模式指向 `2606`），并同步更新：
- `HPC/openfoam/20260907/...` 目录名（若采用版本目录命名，需与 `meta.yml`、`README.md`、`doc/image-info.yml` 的 tag 保持一致）
- `HPC/openfoam/meta.yml` 中的条目名与 path
- `HPC/openfoam/README.md` 与 `HPC/openfoam/doc/image-info.yml` 中的 tag/链接

注意：由于目录层级与 tag 规范（`{image-version}/{os-version}/Dockerfile`）受 CI 校验约束，目录重命名需整体一致，避免触发模式29（路径层级/校验失败）。

### 方向 2（可选，置信度: 低）
若上游 sourceforge 对应目录确实存在但文件名/路径规则不同（例如版本前缀、归档路径差异），则需按真实发布结构调整下载 URL。此方向需先确认上游发布列表，当前无日志/网络证据支持。

## 需要进一步确认的点
1. **必须获取失败 job 的实际日志**：当前 `ci.logs` 缺失，无法确认失败究竟是下载 404、下载超时，还是下载成功后的 `./Allwmake` 编译失败。在拿到日志前，以上根因属于推断而非确证。
2. 确认上游 `https://sourceforge.net/projects/openfoam/files/` 下实际存在的版本列表，核实 `v20260907` 是否存在。若不存在，确认真实的最新版本号（历史模式给出 `2606`）。
3. 确认 `sourceforge.net` 在 CI 构建环境是否可达（历史模式33 显示部分源在 CI 环境网络不通，可能产生超时而非 404），以区分"版本不存在"与"网络不可达"。
4. 确认 `meta.yml` 中新增条目的 tag 名（`20260907-oe2403sp4`）与 CI 规范、`image-list.yml` 校验是否一致。
5. 确认新 Dockerfile 是否缺少项目规范要求的 Copyright / SPDX 头（知识库模式17）；diff 中新增的 Dockerfile 未见版权头，若 CI 有 `check_package_license` 检查也可能导致失败。

## 修复验证要求（仅当修复涉及正则 patch 外部源文件时填写）
不适用（本失败不涉及对上游源文件的正则 patch）。
