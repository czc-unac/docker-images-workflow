# 修复摘要

## 修复的问题
修正 scann Dockerfile 中错误的 pip 安装版本，将上游 Git commit 短哈希 `d36068b` 改为 PyPI 实际存在的发行版本 `1.4.2`，解决 `Could not find a version that satisfies the requirement scann==d36068b` 导致的 Docker 构建失败。

## 修改的文件
- `Others/scann/d36068b/24.03-lts-sp4/Dockerfile`: 第 4 行 `ARG VERSION=d36068b` 改为 `ARG VERSION=1.4.2`。

## 修复逻辑
CI 失败根因是依赖版本不存在（`dependency-error`）：第 21 行 `pip3 install scann==${VERSION}` 要求 `${VERSION}` 为 PyPI 上的 scann 发行版本号，而 `d36068b` 是 `image-info.yml` 中 `upstream.version_url: google-research/google-research` 自动升级流程从上游源码仓库取到的 git commit 短哈希，并非 PyPI 发行版本。PyPI（清华镜像）上 scann 可用版本仅为 `1.2.2 ~ 1.4.2`，因此构建失败。

本次按分析报告「方向 1（置信度 高）」修复：使用 PyPI 实际存在的发行版本。选择 `1.4.2` 是因为：
1. 该版本在 CI 日志列出的可用版本列表上限内，确认可被 pip 解析安装；
2. 仓库中已存在 `Others/scann/1.4.2/` 目录且同样使用 `scann==1.4.2`，证明该版本可成功构建；
3. master 分支上针对同一失败的历史修复（commit `51306eada`，PR #4555）也采用了完全相同的改法，本次改动与其结果一致（diff blob 相同）。

未修改 `README.md` / `meta.yml` / `image-info.yml`：镜像目录名与 tag（`d36068b-oe2403sp4`）保持不变，无法在「不新增文件、不重命名」的约束下改为 1.4.2（会与已存在的 `1.4.2-oe2403sp4` 条目冲突），因此仅修改直接决定构建成败的 Dockerfile 版本，符合最小化原则。

## 潜在风险
- 该目录/dtag 名称 `d36068b` 与实际安装版本 `1.4.2` 在语义上不一致，文档中仍显示 "scann d36068b"，属可读性/一致性问题，不影响构建。
- 根因修复不彻底：`image-info.yml` 的 `upstream.version_url` 指向无发行 tag 的源码仓库 `google-research/google-research`，自动升级流程仍可能再次生成 commit 哈希版本（本次失败在 master 上已有先例，历史修复曾被下一次自动升级覆盖）。建议后续由人工确认 scann 的正确版本来源（PyPI 发行版本）并调整自动升级配置，但当前分析报告对该配置的正确写法未给出确定结论，故本次不盲目修改，仅在此记录。
- 日志中 `ffi.h: No such file or directory` 导致 `_ctypes` 模块未编译为已知非致命告警，非本次失败根因，未做处理。