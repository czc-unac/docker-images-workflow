# 修复摘要

## 修复的问题
新增的 PyRosetta 3.15 Dockerfile 中 `ARG VERSION` 指向了上游不存在的 ref `v3.15-dev62280`，导致 `git clone --branch` 失败，镜像构建中断。

## 修改的文件
- `HPC/pyrosetta/3.15/24.03-lts-sp4/Dockerfile`: 将 `ARG VERSION=v3.15-dev62280` 改为 `ARG VERSION=v3.15`。

## 修复逻辑
- 分析报告的假设 1（上游版本/分支不存在）经实测证实为真实根因：
  - 通过 `git ls-remote --tags https://github.com/RosettaCommons/rosetta.git` 及 GitHub API 校验，`v3.15-dev62280` **不存在**（API 返回 404，tag 列表中无此 ref）。
  - 同仓库存在合法 tag `v3.15`（commit `df75a9c48e763e52a7aa3f5dfba077f4da88dbf5`，日期 2025-09-03）。
  - 通过 GitHub Contents API 确认 `v3.15` 下存在 `source/src/python/PyRosetta/build.py`，与 Dockerfile 后续执行路径 `cd /opt/rosetta/source/src/python/PyRosetta && python3 build.py` 一致。
- 自动升级脚本疑似机械套用了 `v2026.29-dev62280` 的命名格式，错误拼装出不存在的 `v3.15-dev62280`；实际 3.15 为上游的纯发布 tag `v3.15`。
- 排除分析报告中的其他假设：
  - 假设 2（`-isystem /usr/include/c++/12` 路径）：该行与已在 master 中、构建通过的 `2026.29` Dockerfile 完全一致，说明基础镜像 `openeuler/openeuler:24.03-lts-sp4` 的 GCC 12 路径正确，无需修改。
  - 假设 3（构建依赖缺失）：`dnf install` 列表与可用的 `2026.29` 完全一致，无需修改。
  - 假设 4（元数据一致性）：`meta.yml` / `image-info.yml` / `README.md` 中 tag 均为 `3.15-oe2403sp4`，路径 `3.15/24.03-lts-sp4/Dockerfile` 一致，无问题。
- 本修复不涉及正则 patch 第三方/上游源文件，无正则验证要求；但已实际拉取上游 ref/tag 列表与文件树完成验证。

## 潜在风险
无。改动仅为修正无效的上游 ref 为真实存在的 tag，不改变构建流程、依赖或元数据；其余三个文件未做改动。