# CI 失败分析报告

## 基本信息
- PR: #4618 — 【自动升级】rabitq-library容器镜像升级至0.4.0版本.
- 失败类型: build-error
- 置信度: 中
- 知识库匹配: 新模式
- 新模式标题: Git标签不存在
- 新模式症状关键词: pathspec, did not match any file(s) known to git, git checkout, tag, git clone

## 根因分析

### 直接错误
```
#9 [4/4] RUN git clone https://github.com/VectorDB-NTU/RaBitQ-Library.git &&     cd RaBitQ-Library && git checkout 0.4.0 &&     cp -r include /usr/local/include/rabitq
#9 0.066 Cloning into 'RaBitQ-Library'...
#9 1.642 error: pathspec '0.4.0' did not match any file(s) known to git
#9 ERROR: process "/bin/sh -c git clone ... && cd RaBitQ-Library && git checkout ${VERSION} && ..." did not complete successfully: exit code: 1
Dockerfile:9
   9 | >>> RUN git clone https://github.com/VectorDB-NTU/RaBitQ-Library.git && \
  10 | >>>     cd RaBitQ-Library && git checkout ${VERSION} && \
  11 | >>>     cp -r include /usr/local/include/rabitq
ERROR: failed to solve: ... exit code: 1
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: `Others/rabitq-library/0.4.0/24.03-lts-sp4/Dockerfile:9`（`git checkout ${VERSION}` 步骤，`VERSION=0.4.0`）
- 失败原因: clone 上游仓库 `VectorDB-NTU/RaBitQ-Library` 后，`git checkout 0.4.0` 找不到名为 `0.4.0` 的 ref（tag/分支），`pathspec` 匹配失败，容器构建中断。

### 与 PR 变更的关联
直接相关。本 PR 新增了 `Others/rabitq-library/0.4.0/24.03-lts-sp4/Dockerfile`，其中 `ARG VERSION=0.4.0` 并执行 `git checkout ${VERSION}`。该 ref 在上游仓库中不存在，是本次失败的直接触发点。README.md、doc/image-info.yml、meta.yml 的条目新增仅为元数据同步，不影响构建结果。

## 修复方向

### 方向 1（置信度: 中）
确认上游仓库 `VectorDB-NTU/RaBitQ-Library` 中 0.4.0 对应的**实际 ref 名称**：可能是带前缀的 tag（如 `v0.4.0`），或该版本尚未发布。若为命名差异，将 Dockerfile 中 `ARG VERSION` 的值修正为上游实际存在的 tag；若上游仅以 commit 提供，可参照本仓库既有 `7c2d0d7-oe2403sp4` 条目的做法改用具体 commit。

### 方向 2（可选，置信度: 低）
若上游确实没有 0.4.0 对应的任何制品/发布，则本次"自动升级"目标本身不成立，需要回退到上游实际已发布的最新版本（如同步为 0.3.8），或等待上游打 tag。此方向需要上游发布信息佐证。

## 需要进一步确认的点
- 上游 `VectorDB-NTU/RaBitQ-Library` 是否存在 `0.4.0`（或 `v0.4.0` 等）tag：需通过 `git ls-remote --tags <repo>` 或上游 releases 页面确认。当前日志只证明 `0.4.0` 这个 ref 在默认 clone 后不可解析，无法区分"tag 命名带前缀"与"tag 尚未发布"两种情况。
- 既有 0.3.8 / 0.3.6 条目当初使用的 ref 形式为何（是否为 `0.3.8`），可据此判断 0.4.0 的命名规律。
- 上游仓库的默认分支是否包含 `include` 目录，以评估回退方案的可行性（与本次失败无直接关系，但影响替代版本选择）。

## 修复验证要求
本次置信度为"中"，修复前 code-fixer 必须执行以下验证，不得假定标签命名规律：
1. 从上游仓库 `VectorDB-NTU/RaBitQ-Library` 获取实际 tag/ref 列表（以本 PR 目标 `VERSION=0.4.0` 为准），例如执行 `git ls-remote --tags https://github.com/VectorDB-NTU/RaBitQ-Library.git`，确认 `0.4.0` 或 `v0.4.0` 等确切名称是否存在。
2. 依据实际存在的 ref 修正 Dockerfile 中的版本值；若不存在任何 `0.4.0` 对应 ref，则需与上游版本策略核对后再决定回退或更换版本，不能凭推断提交。
