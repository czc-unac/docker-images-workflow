# 修复摘要

## 修复的问题
修复 milc 6b9b8a0 镜像 Dockerfile 将 commit 短哈希 `6b9b8a0` 误作 `git clone --branch` 参数导致上游 ref 不存在、clone 退出码 128 的构建失败。

## 修改的文件
- `HPC/milc/6b9b8a0/24.03-lts-sp4/Dockerfile`: 将第 18 行 `git clone --depth 1 --branch 6b9b8a0 ...` 改为完整克隆后 `git checkout ${VERSION}`（`VERSION` 已由第 4 行 `ARG VERSION=6b9b8a0` 定义）。

## 修复逻辑
- 根因：分析报告指出 `6b9b8a0` 是版本号/commit 短哈希，而 `git clone --branch` 只接受分支名或标签名，不能接受 commit hash；上游 `milc-qcd/milc_qcd` 不存在名为 `6b9b8a0` 的分支或标签，故 clone 立即失败（exit code 128）。
- 已从上游 `https://github.com/milc-qcd/milc_qcd` 验证：
  - `git ls-remote` 确认 `6b9b8a06eec5746187bbfd197eac2629ab8d8e72` 为当前 `develop`/`HEAD` 指向的真实 commit，且上游**无任何 tag**、无名为 `6b9b8a0` 的分支。
  - 实测完整 `git clone` 后 `git checkout 6b9b8a0` 可正确解析并检出该 commit（HEAD 指向 `6b9b8a06 Use FORSOMPARITY_OMP instead of FOREVENSITES_OMP`），短哈希无歧义。
- 采用 `git clone`（去掉 `--depth 1` 浅克隆，浅克隆无法 checkout 任意历史 commit）+ `git checkout ${VERSION}` 的方式，与本仓库既有 commit 版本镜像的惯例一致（如 `Storage/3fs/22fca04`、`Others/multiwfn/cb37c53`、`Others/monolith/135c491`、`Others/rabitq-library/7c2d0d7` 均采用 clone 后 checkout 短哈希）。
- `meta.yml`、`README.md`、`doc/image-info.yml` 中的 `6b9b8a0` 版本目录与 tag 名称本身正确，无需改动，保持改动最小化。

## 潜在风险
- 改为完整克隆会略微增加构建时网络传输量（原 `--depth 1` 为浅克隆），这是 checkout 指定历史 commit 所必需的；构建时间可能小幅增加，不影响构建结果。
- 该修复依赖上游仓库长期保留 commit `6b9b8a0`；若上游后续 force-push 删除该 commit 则会失效，但当前该 commit 为上游主线可达历史，正常不会被回收。
- 未验证点：无法在本环境实际执行 `docker build`（openEuler 基础镜像与 cmake 编译需容器环境），但导致失败的 clone/checkout 步骤已用等价 git 命令在本地成功复现验证。