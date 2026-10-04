# 修复摘要

## 修复的问题
rabitq-library 0.5.2 镜像构建时 `git checkout 0.5.2` 失败（`pathspec '0.5.2' did not match any file(s) known to git`）。原因是上游 `VectorDB-NTU/RaBitQ-Library` 的发行 tag 带 `v` 前缀，实际 tag 为 `v0.5.2`，需按 `v${VERSION}` 检出。

## 修改的文件
- `Others/rabitq-library/0.5.2/24.03-lts-sp4/Dockerfile`: 第 10 行 `git checkout ${VERSION}` 修正为 `git checkout v${VERSION}`，使 `ARG VERSION=0.5.2` 解析为上游实际存在的 `v0.5.2`。
  当前分支已包含该改动（相对原始 PR head `52ba758e5` 的差异）：
  ```diff
  -    cd RaBitQ-Library && git checkout ${VERSION} && \
  +    cd RaBitQ-Library && git checkout v${VERSION} && \
  ```

## 修复逻辑
本次不再依赖低置信度推断，已取得真实 CI 控制台日志确认根因（失败发生在 **build 阶段**，非 license/编排层）：

- x86_64 job #5009:
  `#9 1.300 error: pathspec '0.5.2' did not match any file(s) known to git` → `exit code: 1`
  `https://ci.openeuler.openatom.cn/job/multiarch/job/openeuler/job/x86-64/job/openeuler-docker-images/5009/consoleText`
- aarch64 job #5105:
  `#9 2.506 error: pathspec '0.5.2' did not match any file(s) known to git` → `exit code: 1`
  `https://ci.openeuler.openatom.cn/job/multiarch/job/openeuler/job/aarch64/job/openeuler-docker-images/5105/consoleText`

上游 tag 验证（不依赖精确签名，直接枚举实际 tag）：
- `git ls-remote --tags https://github.com/VectorDB-NTU/RaBitQ-Library.git` 返回的发行 tag 全部为 `v` 前缀，且存在 `refs/tags/v0.5.2` = `d929e30dbb7ded52db1830736a2356fbc7dba99a`：
  `v0.5.0`、`v0.5.1`、`v0.5.2` 均带 `v`，不存在裸 `0.5.2`。
- GitHub API 确认 `v0.5.2` 树中存在 `include/rabitqlib/...`，因此第 11 行 `cp -r include /usr/local/include/rabitq` 可正常执行。

因此根因为方向 1（tag 命名带 `v` 前缀），修复方式与仓库历史同类修复 `d69188725`（0.3.6）、`d9dc7db9a`（0.3.8）以及现有 0.3.6/0.3.8/0.3.9/0.4.0 Dockerfile 完全一致，属该镜像的既有正确范式。

关于分析报告的备选方向：
- 方向 2（缺失 Copyright/SPDX 头）：仓库 `check_package_license` 仅给出 **WARNING**「仓库copyright检查未通过: 缺少项目级Copyright声明文件」，且该检查针对项目级声明文件而非单个 Dockerfile；全仓 2066 个 Dockerfile 中仅 126 个带 SPDX 头，rabitq-library 各版本均无该头且历史上构建通过，故非本次根因，未改动。
- 方向 3（`/usr/local/include` 不存在）：openEuler 基础镜像该目录存在，非根因，未改动。

补充验证（fix 分支修复 PR #4919 的复测）：
- aarch64 #5134 日志中已出现 `[Build] finished` / `[Push] finished`，证明修正后的 Dockerfile 可成功构建、推送；其随后报 `FileNotFoundError: [Errno 2] No such file or directory: 'eulerpublisher'`，为 CI 后处理工具问题。
- x86_64 #5038 报 `curl: (22) The requested URL returned error: 429`（拉取 `build.sh` 被限流），构建脚本未执行。
- 以上两项均为 **infra-error**，与代码修复无关；本次针对原始失败（PR #4890）的代码修复已生效，无需进一步代码改动。

## 潜在风险
无。改动仅涉及新增的 `0.5.2` Dockerfile，不影响其他镜像；`git checkout v${VERSION}` 与仓库内所有既有 rabitq-library 版本行为一致。本次运行未新增/删除文件，未触碰 `pr.changed_files` 之外的任何文件。