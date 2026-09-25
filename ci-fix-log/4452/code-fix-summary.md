# 修复摘要

## 修复的问题
修正 cps_public 5.2.5 Dockerfile 中 git clone 的分支名，使其命中上游真实 tag `v5_2_5`，解决 `fatal: Remote branch v5.2.5 not found in upstream origin` 构建失败。

## 修改的文件
- `HPC/cps_public/5.2.5/24.03-lts-sp4/Dockerfile`: 第 4 行 `ARG VERSION=5.2.5` 改为 `ARG VERSION=5_2_5`（单个字符 `.` → `_`）。

## 修复逻辑
分析报告的根因：新增的 `5.2.5/24.03-lts-sp4/Dockerfile` 第 4 行 `ARG VERSION=5.2.5` 与第 13 行 `git clone --depth 1 --branch v${VERSION} ...` 组合展开为 `v5.2.5`，该 ref 在上游 `RBC-UKQCD/CPS_public` 不存在，导致退出码 128。

修复前已按要求从上游核实真实 tag 列表（`git ls-remote --tags https://github.com/RBC-UKQCD/CPS_public.git`），确认 5.2.5 对应的真实 tag 为下划线形式 `v5_2_5`（解析结果：`ef4453256b6f49ffa2751dc15f73121996d034a2 refs/tags/v5_2_5`）。仓库中已合并的同版本 `HPC/cps_public/5_2_5/24.03-lts-sp4/Dockerfile` 也正是使用 `ARG VERSION=5_2_5`，本次改动与之保持一致。`ARG VERSION` 仅用于构造 clone ref，不参与镜像 tag 命名（镜像 tag 由 meta.yml/README 定义），因此将值改为上游真实 tag 不会影响镜像发布名 `5.2.5-oe2403sp4`。

## 潜在风险
无。改动仅影响该 Dockerfile 内部 clone 所使用 ref，且已验证该 ref 在上游可解析；未改动 meta.yml、README.md、doc/image-info.yml，镜像 tag 与文档保持不变。