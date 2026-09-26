# CI 失败分析报告

## 基本信息
- PR: #4536 — 【自动升级】rabitq-library容器镜像升级至0.3.9版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式02（下载 URL 硬编码版本路径错误 / 软件包版本不存在；含上游 Git tag 不存在的历史案例）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#9 [4/4] RUN git clone https://github.com/VectorDB-NTU/RaBitQ-Library.git &&     cd RaBitQ-Library && git checkout 0.3.9 &&     cp -r include /usr/local/include/rabitq
#9 0.241 Cloning into 'RaBitQ-Library'...
#9 2.102 error: pathspec '0.3.9' did not match any file(s) known to git
#9 ERROR: process "/bin/sh -c git clone https://github.com/VectorDB-NTU/RaBitQ-Library.git &&     cd RaBitQ-Library && git checkout ${VERSION} &&     cp -r include /usr/local/include/rabitq" did not complete successfully: exit code: 1
------
Dockerfile:9
   8 |     WORKDIR /workspace
   9 | >>> RUN git clone https://github.com/VectorDB-NTU/RaBitQ-Library.git && \
  10 | >>>     cd RaBitQ-Library && git checkout ${VERSION} && \
  11 | >>>     cp -r include /usr/local/include/rabitq
ERROR: failed to solve: process ... exit code: 1
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: `Others/rabitq-library/0.3.9/24.03-lts-sp4/Dockerfile:10`
- 失败原因: `git clone` 上游仓库 `VectorDB-NTU/RaBitQ-Library` 成功后，`git checkout ${VERSION}`（VERSION=0.3.9）报 `pathspec '0.3.9' did not match any file(s) known to git`，即上游仓库中不存在名为 `0.3.9` 的 tag/分支/ref，checkout 失败并导致构建退出（exit code 1）。

### 与 PR 变更的关联
强关联。本 PR 新增 `Others/rabitq-library/0.3.9/24.03-lts-sp4/Dockerfile`，其中硬编码 `ARG VERSION=0.3.9`，并在 `RUN` 中以 `${VERSION}` 作为 git checkout 的 ref。失败正是由该新增行的 ref 在上游不存在引起，与 PR 改动直接对应，属于新版本镜像引入的错误版本标签。

### 补充说明（排除其他可能）
- 日志末尾为 `Finished: FAILURE`，且含明确 `ERROR` + 退出码 1，属于真实构建失败，不适用"日志成功但状态失败"的 infra-error 场景。
- 日志中 `dnf install git`、`WORKDIR` 等前置步骤均成功（`#7 DONE 40.2s`、`#8 DONE 0.3s`），排除依赖安装/基础镜像问题。
- 失败发生在 `git checkout` 而非 `git clone`，clone 本身成功，故排除网络不可达问题。

## 修复方向

### 方向 1（置信度: 高）
核对上游仓库 `VectorDB-NTU/RaBitQ-Library` 在 0.3.9 版本实际使用的 ref 名称，并将 Dockerfile 中 `ARG VERSION` 或 checkout 使用的 ref 修正为上游真实存在的 tag/分支名（例如若上游 tag 带 `v` 前缀则应写 `v0.3.9`），确保 `git checkout ${VERSION}` 能匹配到 ref。同时同步更新 `meta.yml`、`README.md`、`doc/image-info.yml` 中与该版本一致的标识（若版本号本身有误，需整体一致修正）。

### 方向 2（置信度: 中）
若上游 0.3.9 未发布独立 tag 而以 commit hash 标注，则应改为使用该版本对应的 commit hash 作为 checkout 目标（注意参考模式18：避免 `--depth 1` 浅克隆导致 hash 无法 checkout；本 Dockerfile 当前未使用 `--depth 1`，故直接 checkout hash 可行）。

## 需要进一步确认的点
1. 上游仓库 `https://github.com/VectorDB-NTU/RaBitQ-Library.git` 中 0.3.9 对应的真实 ref 名称（tag 是 `0.3.9`、`v0.3.9` 还是其他，或仅有 commit hash）。
2. 该版本是否已在上游正式发布；若 tag 命名历史规律与现有 0.3.8/0.3.6 目录不同，需确认正确的版本号或 ref。
3. `include` 目录在目标 ref 下是否仍位于仓库根目录（若目录结构变动会影响后续 `cp -r include`）。
