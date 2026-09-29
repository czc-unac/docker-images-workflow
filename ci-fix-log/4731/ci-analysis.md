# CI 失败分析报告

## 基本信息
- PR: #4731 — 【自动升级】blat容器镜像升级至2.5.1版本.
- 失败类型: build-error
- 置信度: 高（根因定位）/ 中（正确版本号需上游核实）
- 知识库匹配: 模式22（Git分支名构造错误，症状高度匹配）；根因机理与模式02（版本/上游 ref 不存在）同源
- 新模式标题: (无需填写，已有模式可覆盖)
- 新模式症状关键词: (无需填写)

## 根因分析

### 直接错误
```
#8 [3/6] RUN git clone -b 2.5.1 https://github.com/icebert/pblat-cluster.git /blat-cluster
#8 0.169 Cloning into '/blat-cluster'...
#8 0.886 fatal: Remote branch 2.5.1 not found in upstream origin
#8 ERROR: process "/bin/sh -c git clone -b ${VERSION} https://github.com/icebert/pblat-cluster.git /blat-cluster" did not complete successfully: exit code: 128
------
Dockerfile:9
   9 | >>> RUN git clone -b ${VERSION} https://github.com/icebert/pblat-cluster.git /blat-cluster
ERROR: failed to solve: process "/bin/sh -c git clone -b ${VERSION} ..." did not complete successfully: exit code: 128
```

### 根因定位
- 失败位置: `HPC/blat/2.5.1/24.03-lts-sp4/Dockerfile:9`（新增文件中 `ARG VERSION=2.5.1` 配合 `git clone -b ${VERSION}`）
- 失败原因: 新增 Dockerfile 将 `VERSION` 硬编码为 `2.5.1`，但上游仓库 `https://github.com/icebert/pblat-cluster.git` 中**不存在名为 `2.5.1` 的分支或标签**，`git clone -b 2.5.1` 在克隆阶段即报 `fatal: Remote branch 2.5.1 not found in upstream origin`（exit code 128），构建在 `[3/6]` 步骤终止。

### 与 PR 变更的关联
- 直接相关。该 PR 为“自动升级”新增 `HPC/blat/2.5.1/24.03-lts-sp4/Dockerfile`，其中 `ARG VERSION=2.5.1` 是本 PR 新引入的值，正是该值导致 `git clone -b 2.5.1` 失败。
- 同时注意版本元数据与克隆地址存在不一致：`HPC/blat/doc/image-info.yml` 的 `upstream.version_url` 为 `icebert/blat-cluster`（无 `p`），而 Dockerfile 克隆的是 `icebert/pblat-cluster`（有 `p`）。自动升级脚本可能从错误的上游/错误的版本来源推导出 `2.5.1`。
- `2.5.1` 更像是 pblat 软件版本号，而上游 `pblat-cluster` 的构建分支命名规则很可能不是纯语义版本号（历史 `1.1` 分支可用即是反例）。需要确认 `2.5.1` 是否真实存在对应的上游 ref。

## 修复方向

### 方向 1（置信度: 高）
核实上游 `icebert/pblat-cluster` 实际存在的分支/标签后，将 `ARG VERSION` 改为真实存在的分支名（保持与上游命名规则一致），再重新生成 README/meta.yml/image-info.yml 中对应的版本描述。切勿凭空保留不可用的 `2.5.1`。

### 方向 2（可选，置信度: 中）
若确认“升级到 2.5.1”本身有误（自动升级读取了错误的上游或错误字段），应撤回该版本新增，改以真实可达的上游版本为准；并修正 `image-info.yml` 中 `version_url: icebert/blat-cluster` 与 Dockerfile 克隆地址之间的不一致，避免后续自动升级再次推导出错误版本。

## 需要进一步确认的点
1. 上游 `https://github.com/icebert/pblat-cluster.git` 实际包含哪些分支/标签（可执行 `git ls-remote --tags https://github.com/icebert/pblat-cluster.git` 与 `--heads`），确认是否存在与 `2.5.1` 对应的 ref、以及分支命名格式。
2. `2.5.1` 是 pblat 的软件版本号还是 pblat-cluster 的分支名？二者是否为同一命名体系。
3. `HPC/blat/doc/image-info.yml` 的 `upstream.version_url`（`icebert/blat-cluster`）与 Dockerfile 克隆源（`icebert/pblat-cluster`）为何不一致，自动升级依据的是哪一个来源。
4. 历史可用版本 `1.1` 对应的上游 ref 名称，作为确认新版本应采用的命名格式的参照。

## 修复验证要求
本修复若涉及“修改版本号/分支名以匹配上游 ref”，code-fixer 在提交前必须：
1. 通过 `git ls-remote https://github.com/icebert/pblat-cluster.git` 拉取上游真实的分支与标签列表，确认所选 `VERSION` 值与其中某个 ref 完全一致（尤其确认目标是否需要用 tag 而非分支，或需带 `v`/其它前缀）。
2. 确认自动升级引入的 `2.5.1` 与上游 ref 的实际对应关系，避免再次提交一个上游不存在的 ref。
3. 如同时调整 `image-info.yml` 的 `version_url`，需核对与 Dockerfile 克隆地址一致，保证后续自动升级来源正确。
