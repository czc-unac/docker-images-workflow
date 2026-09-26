# 修复摘要

## 修复的问题
Dockerfile 中 `git checkout` 的 ref 缺少 `v` 前缀，导致上游不存在 `0.3.9` tag 而 checkout 失败。

## 修改的文件
- `Others/rabitq-library/0.3.9/24.03-lts-sp4/Dockerfile`: 将第 10 行 `git checkout ${VERSION}` 修正为 `git checkout v${VERSION}`。

## 修复逻辑
CI 失败根因是 `pathspec '0.3.9' did not match any file(s) known to git`。通过 `git ls-remote --tags https://github.com/VectorDB-NTU/RaBitQ-Library.git` 及 GitHub API 核实，上游实际 tag 名为 `v0.3.9`（commit `f4be243ab7be4d79e56b0047622a9e0279c35de7`），不存在无前缀的 `0.3.9`。同目录下既有的 0.3.6/0.3.8 Dockerfile 均使用 `git checkout v${VERSION}`，本次新增的 0.3.9 因漏写 `v` 前缀导致构建失败。修改后 `ARG VERSION=0.3.9` 保持版本号语义不变，checkout 目标变为上游存在的 `v0.3.9`。另核实 `v0.3.9` 的仓库根目录仍包含 `include/`，故后续 `cp -r include /usr/local/include/rabitq` 无需调整。

`meta.yml`、`README.md`、`doc/image-info.yml` 中的版本号标识 `0.3.9` 与上游一致，无需改动。

## 潜在风险
无。改动仅补全 ref 前缀，不改变版本号与镜像对外标识。