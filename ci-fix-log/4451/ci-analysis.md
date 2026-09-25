# CI 失败分析报告

## 基本信息
- PR: #4451 — 【自动升级】diskann容器镜像升级至5.0.3版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式22（Git分支名构造错误）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#9 [builder 4/6] RUN git clone -b v5.0.3 --depth 1 https://github.com/microsoft/DiskANN.git /build
#9 0.190 Cloning into '/build'...
#9 0.711 fatal: Remote branch v5.0.3 not found in upstream origin
#9 ERROR: process "/bin/sh -c git clone -b v${VERSION} --depth 1 https://github.com/microsoft/DiskANN.git /build" did not complete successfully: exit code: 128
------
Dockerfile:13
  11 |     ENV PATH=/root/.cargo/bin:$PATH
  12 |     
  13 | >>> RUN git clone -b v${VERSION} --depth 1 https://github.com/microsoft/DiskANN.git /build
  14 |     WORKDIR /build
  15 |     
ERROR: failed to solve: process "/bin/sh -c git clone -b v${VERSION} --depth 1 https://github.com/microsoft/DiskANN.git /build" did not complete successfully: exit code: 128
```

### 根因定位
- 失败位置: `AI/diskann/5.0.3/24.03-lts-sp4/Dockerfile:13`
- 失败原因: Dockerfile 中 `ARG VERSION=5.0.3` 配合 `git clone -b v${VERSION}` 构造出的 ref 为 `v5.0.3`，该分支/tag 在上游仓库 `microsoft/DiskANN` 中不存在，git clone 返回 exit code 128。

### 与 PR 变更的关联
本 PR 为新增 Dockerfile（`new_file: True`），该文件首次引入 diskann 5.0.3 构建逻辑，其中 `git clone -b v${VERSION}` 的 VERSION 取值为 `5.0.3`。日志中 `git clone` 前面所有步骤（dnf 安装、rustup 安装 1.92.0）均成功，唯一失败点即 clone 步骤，因此失败由本 PR 新增内容直接引起，与基础镜像或旧版本无关。

补充：日志末尾为 `Finished: FAILURE`，非“日志成功但 PR 失败”情形，故不适用 infra-error（证据不足）判定。

## 修复方向

### 方向 1（置信度: 高）
确认 `microsoft/DiskANN` 仓库中 5.0.3 对应的实际 ref 名称。可能为以下之一：
- 上游 tag 未加 `v` 前缀（如 `5.0.3`），此时应调整 `git clone -b` 的 ref 构造方式；
- 上游根本不存在 5.0.3 版本（该号可能是待发布/错误的自动升级版本），需回退或改用上游真实存在的版本号。

由于当前证据只能证明 `v5.0.3` 不存在，无法区分“前缀错误”还是“版本号错误”，两种根因的修复动作不同。

### 方向 2（可选）
若自动升级工具误将非上游 release 版本号写入，应核对升级来源版本号与 `microsoft/DiskANN` 的实际 tag 命名规范，修正 `VERSION` 与 ref 模板的一致性（同时更新 `meta.yml`、`README.md`、`doc/image-info.yml` 中引用的 5.0.3 版本号）。

## 需要进一步确认的点
1. `microsoft/DiskANN` 上游仓库中是否存在 5.0.3 对应的 tag/branch，及其确切名称（是否带 `v` 前缀）。需在修复前实际查询上游 refs，不能假设。
2. 参考已存在的 `AI/diskann/0.59.0/24.03-lts-sp4/Dockerfile` 与 `0.52.0` 版本，确认历史上 `git clone -b` 使用的 ref 格式（是否带 `v`），以判断本 PR 的 ref 模板是否偏离既有约定。
3. 若 5.0.3 并非上游真实版本，需确认该版本号的来源与正确版本号，并同步修正 `meta.yml`、`README.md`、`doc/image-info.yml` 中的版本条目。

## 修复验证要求
本失败不涉及正则 patch 外部源文件，无需此类验证。但 code-fixer 在提交前必须实际验证上游 ref 存在性：
- 拉取/查询 `microsoft/DiskANN` 仓库，确认修改后的 `git clone -b <ref>` 能成功解析到 5.0.3（或替代版本）的真实 ref。
- 若 ref 前缀或版本号发生变化，需同步确认 `README.md`、`meta.yml`、`doc/image-info.yml` 中新增的 `5.0.3-oe2403sp4` 条目与实际构建版本一致。
