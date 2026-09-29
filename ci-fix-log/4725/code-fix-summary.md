# 修复摘要

## 修复的问题
修正 LAMMPS 下载地址中错误的 Git tag（`stable_2025.07.22` 不存在导致 wget 404），改用上游实际存在的 `stable_22Jul2025`。

## 修改的文件
- `HPC/lammps/2025.07.22/24.03-lts-sp4/Dockerfile`: 第 4 行 `ARG VERSION=2025.07.22` 改为 `ARG VERSION=22Jul2025`。`wget`/`tar`/`WORKDIR` 均基于 `${VERSION}` 拼接，故一处改动即修正了下载 URL 与解压目录。

## 修复逻辑
- 分析报告根因：新 Dockerfile 以点分日期版本串拼接下载地址 `stable_${VERSION}.tar.gz`，生成的 `stable_2025.07.22` 在 `github.com/lammps/lammps` 不存在，302 跳转 codeload 后返回 404（exit code 8），构建中断。
- LAMMPS 上游 stable tag 采用「日-月缩写-年」命名（如 `stable_29Aug2024`、`stable_22Jul2025`）。已在提交前按分析报告的验证要求实际核验：`curl -I -L https://github.com/lammps/lammps/archive/refs/tags/stable_22Jul2025.tar.gz` 返回 **200**（非 404）；下载归档顶层目录为 `lammps-stable_22Jul2025/`，与 `WORKDIR /opt/lammps-stable_${VERSION}` 完全对应，解压路径不会错位。
- 该 tag 与 `22Jul2025/24.03-lts-sp4/Dockerfile` 中 ARG VERSION 的前缀 `22Jul2025` 一致，且符合 image-info.yml 中 `version_prefix: stable_` 的设定；`version_filter: patch;update` 表明自动化工具选取基础 stable tag，故采用基础 tag `22Jul2025`。
- `README.md`、`doc/image-info.yml`、`meta.yml` 仅登记镜像 tag（`2025.07.22-oe2403sp4`）及路径，属于目录/镜像标识而非下载地址，不影响构建，未作改动以避免与既有 `22Jul2025-oe2403sp4` 条目产生重复键/重复行。

## 潜在风险
- 本次修复属于「修正下载 tag」，镜像对外 tag 仍为规范化目录名 `2025.07.22-oe2403sp4`，其内容实际来源为 `stable_22Jul2025`（同一发布日期，语义一致），不改动对外展示。
- 根本诱因可能是自动化升级流程 `doc/image-info.yml` 中 `version_scheme: RPM` 将 `22Jul2025` 归一化为 `2025.07.22`。本次未改动该配置（不影响本次构建，且缺少该字段可选值的权威定义，避免误改）；若后续自动化再次生成点分日期 tag，建议单独评估修正 `version_scheme`。