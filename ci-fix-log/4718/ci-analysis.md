# CI 失败分析报告

## 基本信息
- PR: #4718 — 【自动升级】rabitq-library容器镜像升级至0.4.0版本.
- 失败类型: `build-error`
- 置信度: 高
- 知识库匹配: 模式02（下载 URL 硬编码版本路径错误 / 软件包版本不存在）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#9 [4/4] RUN git clone https://github.com/VectorDB-NTU/RaBitQ-Library.git &&     cd RaBitQ-Library && git checkout 0.4.0 &&     cp -r include /usr/local/include/rabitq
#9 0.052 Cloning into 'RaBitQ-Library'...
#9 1.208 error: pathspec '0.4.0' did not match any file(s) known to git
#9 ERROR: process "/bin/sh -c git clone ..." did not complete successfully: exit code: 1
...
Dockerfile:9
   9 | >>> RUN git clone https://github.com/VectorDB-NTU/RaBitQ-Library.git && \
  10 | >>>     cd RaBitQ-Library && git checkout ${VERSION} && \
  11 | >>>     cp -r include /usr/local/include/rabitq
ERROR: failed to solve: process "/bin/sh -c git clone ..." did not complete successfully: exit code: 1
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: `Others/rabitq-library/0.4.0/24.03-lts-sp4/Dockerfile:9-11`（对应 PR 新增 Dockerfile 的 `ARG VERSION=0.4.0` 与 git checkout 步骤）
- 失败原因: 上游仓库 `https://github.com/VectorDB-NTU/RaBitQ-Library.git` 中不存在名为 `0.4.0` 的 ref（分支/tag），`git clone` 成功但 `git checkout 0.4.0` 报 `pathspec '0.4.0' did not match any file(s) known to git`，导致 RUN 步骤退出码 1，Docker build 失败。

### 与 PR 变更的关联
本次 PR 新增了 `Others/rabitq-library/0.4.0/24.03-lts-sp4/Dockerfile`，其中新增构建步骤 `git clone ... && git checkout ${VERSION}`，并将 `VERSION` 设为 `0.4.0`。该失败由本 PR 新增的 Dockerfile 直接触发，属于 PR 改动引起（不是历史遗留问题）。

日志中前面 `#7` 的 dnf 安装 `git` 全部成功（`Complete!`、`DONE 39.4s`），说明基础镜像与包安装无问题；错误最早且唯一出现在 `#9` 的 git checkout。`#9 1.208` 为第一条 error，即为根因，而非后续构建网络类问题。

## 修复方向

### 方向 1（置信度: 高）
确认上游 `RaBitQ-Library` 仓库中该版本对应的实际 ref 名称（可能是带前缀的 `v0.4.0`，或版本号本身尚未发布/仅有 commit hash），据此修正 Dockerfile 中 `ARG VERSION` 的值或 checkout 的目标 ref，使其与上游实际存在的标签一致。参考该目录下已存在的旧版本（0.3.6 / 0.3.8 / 0.3.9）所用 ref 命名规律来确定正确写法。

### 方向 2（置信度: 中）
若 0.4.0 在上游确实以非标签形式（如 commit hash）存在，则需按上游实际提供的 ref 类型调整 checkout 逻辑（例如先 fetch 对应 ref 再 checkout），避免 `git checkout <tag>` 直接失败。

## 需要进一步确认的点
- `https://github.com/VectorDB-NTU/RaBitQ-Library.git` 中 0.4.0 对应的实际 tag/分支名（`0.4.0` 还是 `v0.4.0`，或该版本是否已发布）。
- 旧版本 `Others/rabitq-library/0.3.9/24.03-lts-sp4/Dockerfile` 中使用的 `VERSION`/checkout 写法（用于确认本仓库既有约定）。
- 上游 `RaBitQ-Library` 0.4.0 中 `include` 目录是否存在及路径是否为 `include`（用于确认 `cp -r include /usr/local/include/rabitq` 后续步骤是否正确，本次尚未执行到该步）。

## 修复验证要求
本失败方向不涉及"修改正则 patch 外部源文件"，无需上游文件正则匹配验证。但建议 code-fixer 在提交前：从上游 `RaBitQ-Library` 仓库确认目标 ref 名称确实存在（以 Dockerfile 中 `ARG VERSION` 为准），确保 `git checkout <ref>` 不再报 `pathspec ... did not match any file(s) known to git`。
