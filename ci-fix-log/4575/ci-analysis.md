# CI 失败分析报告

## 基本信息
- PR: #4575 — 【自动升级】xla容器镜像升级至2.9.0版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 版本号拼接错误
- 新模式症状关键词: pathspec, did not match any file(s) known to git, git checkout, ${VERSION}, VERSION

## 根因分析

### 直接错误
```
#10 [5/5] RUN git clone https://github.com/openxla/xla.git &&     cd xla &&     git checkout 2.9.04f29ee5bb224854bf0e12fc0cbecb6db9 &&     python3 ./configure.py --backend=CPU
#10 39.12 error: pathspec '2.9.04f29ee5bb224854bf0e12fc0cbecb6db9' did not match any file(s) known to git
#10 ERROR: process "/bin/sh -c git clone https://github.com/openxla/xla.git &&     cd xla &&     git checkout ${VERSION} &&     python3 ./configure.py --backend=CPU" did not complete successfully: exit code: 1
ERROR: failed to solve: process "/bin/sh -c git clone ... " did not complete successfully: exit code: 1
------ 
Dockerfile:21-24
```

### 根因定位
- 失败位置: `AI/xla/2.9.0/24.03-lts-sp4/Dockerfile:21-24`（`RUN git clone ... && git checkout ${VERSION} ...` 步骤）
- 失败原因: `ARG VERSION=2.9.04f29ee5bb224854bf0e12fc0cbecb6db9` 是一个**拼接错误**的版本字符串——它由语义版本号 `2.9.0` 与上游 commit 短哈希 `4f29ee5bb224854bf0e12fc0cbecb6db9` 直接相连而成（缺少分隔符）。该字符串既不是 openxla/xla 的有效 tag，也不是有效的 commit ref，因此 `git checkout` 报 `pathspec ... did not match any file(s) known to git`，Docker 构建在该层退出码 1 失败。

### 与 PR 变更的关联
失败由本次 PR 新增的 `AI/xla/2.9.0/24.03-lts-sp4/Dockerfile` 直接触发：
- 该 Dockerfile 第 5 行 `ARG VERSION=` 的值格式错误（版本号与 commit 哈希粘连），被第 23 行 `git checkout ${VERSION}` 使用。
- 对比同目录历史条目 `AI/xla/3b0ff80/...`（VERSION 为纯 commit 短哈希 `3b0ff80`），本次自动升级脚本生成的版本字符串显然混合了"版本号 2.9.0"与"commit 哈希"两种语义，导致引用不存在。
- 其余改动（README.md、doc/image-info.yml、meta.yml）与该失败无因果关系。

## 修复方向

### 方向 1（置信度: 高）
确认 openxla/xla 上游仓库中该版本对应的**真实可检出引用**，使 Dockerfile 的 `ARG VERSION` 与之匹配：
- 若上游存在 v2.9.0 之类的 tag，则 VERSION 应为该 tag 原始名称（如 `v2.9.0` 或上游实际 tag 格式）；
- 若上游该版本只能通过 commit 检出，则 VERSION 应为**完整 commit SHA**（40 位）或可被解析的短 SHA，不能与版本号 `2.9.0` 拼接；
- 同时核对 `meta.yml` 中 `2.9.0-oe2403sp4` 的路径与 tag 一致性。

### 方向 2（可选）
若自动升级脚本无法正确解析 xla 的版本/tag 规则（`image-info.yml` 中 `version_scheme: RPM`、`version_filter: alpha;rc;candidate;beta;pre`），应修正脚本对 xla 上游 tag 的提取逻辑，避免再次生成"版本号+commit"粘连字符串。

## 需要进一步确认的点
1. openxla/xla 上游仓库中 `2.9.0` 是否存在对应 tag，及其确切 tag 名称（是否为 `v2.9.0`）。
2. 自动升级工具生成 `VERSION` 的规则：为何将 `2.9.0` 与 `4f29ee5bb224854bf0e12fc0cbecb6db9` 直接相连（推测为版本号截断/拼接 bug），需确认是否还有其它镜像条目受同一生成逻辑影响。
3. `AI/xla/doc/image-info.yml` 的 `upstream` 配置（`version_url: openxla/xla`）与 `meta.yml` 中新条目的版本名是否应与实际 git ref 对齐。

## 修复验证要求
- 本次修复涉及的 VERSION 引用来自上游 Git 仓库，code-fixer 在提交前须用 `git ls-remote` 或等效方式确认所填 `VERSION` 值在 `https://github.com/openxla/xla.git` 中确实存在（tag 或 commit 均可被 `git checkout` 解析），并在本地/CI 复现 `git clone && git checkout <VERSION>` 成功后再提交。
