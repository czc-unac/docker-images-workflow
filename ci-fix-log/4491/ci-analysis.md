# CI 失败分析报告

## 基本信息
- PR: #4491 — 【自动升级】xla容器镜像升级至2.9.0版本.
- 失败类型: `build-error`
- 置信度: 高
- 知识库匹配: 新模式（与模式28"Git短哈希无法作为远程ref"、模式02"版本不存在"相关但不完全一致）
- 新模式标题: 版本引用无效
- 新模式症状关键词: `pathspec ... did not match`, `git checkout`, `ARG VERSION`, `openxla/xla`, `exit code: 1`

## 根因分析

### 直接错误
```
#10 [5/5] RUN git clone https://github.com/openxla/xla.git &&     cd xla &&     git checkout 2.9.04f29ee5bb224854bf0e12fc0cbecb6db9 &&     python3 ./configure.py --backend=CPU
#10 0.067 Cloning into 'xla'...
#10 45.28 Updating files: 100% (9435/9435), done.
#10 45.42 error: pathspec '2.9.04f29ee5bb224854bf0e12fc0cbecb6db9' did not match any file(s) known to git
#10 ERROR: process "/bin/sh -c git clone https://github.com/openxla/xla.git && cd xla && git checkout ${VERSION} && python3 ./configure.py --backend=CPU" did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `AI/xla/2.9.0/24.03-lts-sp4/Dockerfile:23`（`git checkout ${VERSION}`），变量定义在 `Dockerfile:5`（`ARG VERSION=2.9.04f29ee5bb224854bf0e12fc0cbecb6db9`）
- 失败原因: `ARG VERSION` 的值 `2.9.04f29ee5bb224854bf0e12fc0cbecb6db9` 不是 openxla/xla 仓库中任何已存在的 ref（tag/branch/commit）。该字符串明显是版本号 `2.9.0` 与提交哈希 `4f29ee5bb224854bf0e12fc0cbecb6db9` 直接拼接而成（缺少分隔符），因此 `git checkout` 报 `pathspec ... did not match any file(s) known to git`，构建在 5/5 步失败。

### 与 PR 变更的关联
直接由本 PR 新增的 Dockerfile 触发。前序步骤均成功（dnf 安装完成 `#7 DONE 87.0s`、bazelisk 下载完成 `#8 DONE 1.4s`），失败仅出现在 `git checkout ${VERSION}` 这一步，属于新增 Dockerfile 中版本参数取值错误。

## 修复方向

### 方向 1（置信度: 高）
修正 `ARG VERSION` 的取值，使其为 openxla/xla 仓库中确实存在的 git ref。若该镜像意图按 commit 构建（参考仓库既有 `3b0ff80` 条目以短哈希命名目录），则 VERSION 应为纯 commit 哈希（去掉前缀 `2.9.0`），而不是"版本号+哈希"拼接值。

### 方向 2（置信度: 中）
若确实希望以 `2.9.0` 作为构建目标，需先确认 openxla/xla 是否发布了名为 `2.9.0` 的 tag；若不存在，应改用上游实际存在的发布 tag/commit。同时需协调版本目录名（当前为 `2.9.0/`）与实际构建 ref，避免版本标识与 git ref 语义不一致。

## 需要进一步确认的点
- `2.9.04f29ee5bb224854bf0e12fc0cbecb6db9` 是否为"版本号 2.9.0"与"哈希 4f29ee5bb224854bf0e12fc0cbecb6db9"的误拼接；拼接后的哈希片段长度为 33 字符，非标准 40 位完整 SHA，需确认正确的完整 commit hash。
- openxla/xla 上游是否存在 `2.9.0` 的 tag 或分支（`git ls-remote --tags https://github.com/openxla/xla` 验证）。
- 该镜像目录命名规范应遵循哪种方案：以版本号（2.9.0）还是以 commit 哈希（如既有 `3b0ff80`）作为 `meta.yml`/目录标识。
- CI 日志末尾为 `Finished: FAILURE`，与失败状态一致，本次并非 trigger/编排层 job 假成功问题。

## 修复验证要求
本次修复方向不涉及修改正则匹配第三方源文件，无需拉取上游 `fetcher.py` 之类的验证。但 code-fixer 在提交前必须验证目标 ref 确实存在于上游，建议：
- 执行 `git ls-remote https://github.com/openxla/xla <ref>`（或等价方式）确认修正后的 VERSION 能在 openxla/xla 中解析为有效 ref，再提交。
