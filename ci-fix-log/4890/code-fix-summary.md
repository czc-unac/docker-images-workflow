# 修复摘要

## 修复的问题
`rabitq-library` 0.5.2 镜像构建时 `git checkout` 使用了缺少 `v` 前缀的版本号（`${VERSION}` = `0.5.2`），而上游 `VectorDB-NTU/RaBitQ-Library` 的发布 tag 均为 `v` 前缀（`v0.5.2`），导致 checkout 失败、镜像构建中断。已改为 `git checkout v${VERSION}`。

## 修改的文件
- `Others/rabitq-library/0.5.2/24.03-lts-sp4/Dockerfile`: 将 `git checkout ${VERSION}` 修正为 `git checkout v${VERSION}`（与同系列 0.3.6 / 0.3.8 / 0.4.0 的历史修复完全一致）。

> 说明：当前所在的 `fix/4890` 分支 HEAD（`e7d3d8dc1`）已包含上述修复，工作区与 HEAD 一致、无未提交改动；因此本轮无需再产生新的文件改动。其余 PR 涉及文件（`README.md`、`doc/image-info.yml`、`meta.yml`）的版本登记内容正确，无需改动。

## 修复逻辑
- 分析报告的"方向 1（低置信度）"指出：与 PR #4845（0.5.1）、#4718（0.4.0）、#4536（0.3.9）相同，风险点是 `git checkout ${VERSION}` 依赖上游存在该 tag。
- 已通过 GitHub API 验证上游仓库确实存在 tag `v0.5.2`（`refs/tags/v0.5.2` → commit `d929e30dbb7ded52db1830736a2356fbc7dba99a`），且该 commit 的 tree 中存在 `include/` 目录，因此 `cp -r include /usr/local/include/rabitq` 的源目录存在。
- 上游所有 tag 均为 `v` 前缀，Dockerfile 中 `ARG VERSION=0.5.2`，故正确写法为 `v${VERSION}`，即 `v0.5.2`。
- 该修复与知识库中同类历史修复模式（"upstream tags are v-prefixed"）一致，属最小化、已验证的修复。

## 验证结果
- 上游 tag：已获取 `https://api.github.com/repos/VectorDB-NTU/RaBitQ-Library/git/refs/tags/v0.5.2`，确认 tag 存在，匹配成功。
- 上游文件：已获取 tag `v0.5.2`（commit `d929e30`）的仓库 tree，确认包含 `include` 目录，`cp` 步骤源路径有效。
- 基础镜像路径风险：同目录下 0.3.6/0.3.8/0.4.0 均使用完全相同的 `cp -r include /usr/local/include/rabitq`，且这些版本已在 master 上合入，说明基础镜像 `openeuler/openeuler:24.03-lts-sp4` 中存在 `/usr/local/include`，`cp` 目标父目录无风险。

## 潜在风险
无。改动仅修正 checkout 的 tag 前缀，与已合入的历史版本保持一致；不改动构建步骤顺序、基础镜像或安装路径。