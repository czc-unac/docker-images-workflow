# CI 失败分析报告

## 基本信息
- PR: #4534 — 【自动升级】diskann容器镜像升级至5.0.3版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式22（Git分支名构造错误）
- 新模式标题: (非新模式，不适用)
- 新模式症状关键词: (非新模式，不适用)

## 根因分析

### 直接错误
```
#9 [builder 4/6] RUN git clone -b v5.0.3 --depth 1 https://github.com/microsoft/DiskANN.git /build
#9 0.869 Cloning into '/build'...
#9 1.615 fatal: Remote branch v5.0.3 not found in upstream origin
#9 ERROR: process "/bin/sh -c git clone -b v${VERSION} --depth 1 https://github.com/microsoft/DiskANN.git /build" did not complete successfully: exit code: 128
------
Dockerfile:13
--------------------
  13 | >>> RUN git clone -b v${VERSION} --depth 1 https://github.com/microsoft/DiskANN.git /build
--------------------
ERROR: failed to solve: process "/bin/sh -c git clone -b v${VERSION} --depth 1 ..." did not complete successfully: exit code: 128
```

日志末尾为 `Build step 'Execute shell' marked build as failure` 与 `Finished: FAILURE`，日志与失败状态一致，非 infra-error。

### 根因定位
- 失败位置: `AI/diskann/5.0.3/24.03-lts-sp4/Dockerfile:13`
- 失败原因: Dockerfile 用 `git clone -b v${VERSION}` 拉取 `microsoft/DiskANN`，在 `VERSION=5.0.3` 时构造出的 ref 为 `v5.0.3`，该 ref 在上游仓库 `https://github.com/microsoft/DiskANN.git` 中不存在，git 返回 `fatal: Remote branch v5.0.3 not found in upstream origin`，exit code 128。

### 与 PR 变更的关联
直接相关。本 PR 新增了 `AI/diskann/5.0.3/24.03-lts-sp4/Dockerfile`（`new_file: True`），其中 `ARG VERSION=5.0.3` 与 `RUN git clone -b v${VERSION} ...` 为本 PR 引入。失败正是该新增 Dockerfile 的第 13 行在构建时触发的，与既有镜像无关。日志中第 7、8 步（dnf 安装依赖、安装 Rust 1.92.0）均成功（`#7 DONE` / `#8 DONE`），唯一失败步骤是第 9 步的 `git clone`。

## 修复方向

### 方向 1（置信度: 高）
修正 git clone 的目标 ref，使其与 `microsoft/DiskANN` 上游仓库在 5.0.3 版本时实际存在的 tag/branch 一致。需先确认上游 tag 命名规则：
- 若上游 tag 不带 `v` 前缀，则 `v${VERSION}` 应改为 `${VERSION}`；
- 若上游 5.0.3 版本使用其他命名（如 `5.0.3`、`v5.0.3.0` 等），应改为对应实际 ref。
（同时建议核对该 `VERSION` 是否在上游确实发布了 5.0.3，避免模式02 的“版本不存在”情况。）

### 方向 2（置信度: 中）
若上游 `microsoft/DiskANN` 并未发布 `5.0.3` 这一版本（即版本号本身有误），则应改用上游真实存在且与升级目标对应的 tag；否则需回退到存在有效 ref 的版本，或改用 `HEAD`/默认分支（不推荐用于可复现的镜像构建）。

## 需要进一步确认的点
1. 需确认 `https://github.com/microsoft/DiskANN` 在 5.0.3 对应的实际 tag 名称（是否带 `v` 前缀，或是否存在该版本）。
2. 需确认该镜像历史版本（如 `0.59.0/24.03-lts-sp4/Dockerfile`）中对 DiskANN 使用的 clone ref 规则，以判断本 PR 的 `v${VERSION}` 是否与既有约定一致。
3. 需确认 PR 标题“升级至 5.0.3”的目标版本号本身是否正确（DiskANN 版本号序列与 5.0.3 是否匹配）。

## 修复验证要求
本次修复为修改 Dockerfile 中 git 的 ref/分支名，不涉及对第三方源码正则 patch。但 code-fixer 在提交前仍须执行以下验证：
1. 从上游 `https://github.com/microsoft/DiskANN` 查询确认 5.0.3 实际存在的 tag/ref（如 `git ls-remote --tags https://github.com/microsoft/DiskANN.git`），验证新的 `-b` 参数能命中真实 ref。
2. 若保留 `VERSION=5.0.3`，必须确认该版本目录/镜像标签与上游版本语义一致，不可凭猜测套用 `v` 前缀。
