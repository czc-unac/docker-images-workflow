# 修复摘要

## 修复的问题
修正 LAMMPS Dockerfile 中非法的上游版本 tag（`stable_2025.07.22` 在 GitHub 上不存在），使镜像下载 URL 恢复可用，构建不再 404。

## 修改的文件
- `HPC/lammps/2025.07.22/24.03-lts-sp4/Dockerfile`: 第 4 行 `ARG VERSION=2025.07.22` 改为 `ARG VERSION=22Jul2025`。

## 修复逻辑
分析报告置信度为低且未附 `ci.logs`，因此我先从 PR #4762 评论中的门禁结果表取得真实构建 job 链接，并抓取了两架构的完整控制台日志：

- x86_64（`.../x86-64/openeuler-docker-images/4873/`）失败点：
  ```
  #9 [4/8] RUN wget https://github.com/lammps/lammps/archive/refs/tags/stable_2025.07.22.tar.gz ...
  #9 0.581 HTTP request sent, awaiting response... 404 Not Found
  #9 ERROR: ... exit code: 8
  ```
  即 Dockerfile 以 `stable_${VERSION}.tar.gz` 拼出的 `stable_2025.07.22` 在 `github.com/lammps/lammps` 不存在（HTTP 404），构建中断。这对应分析报告中的模式02（下载 URL / 版本不存在）与知识库中已存在的判断：LAMMPS 上游 stable tag 采用 `<DDMonYYYY>` 命名（如 `stable_29Aug2024`、`stable_22Jul2025`），并非 `2025.07.22` 这种点分日期。

LAMMPS 上游 2025-07-22 发布版对应的真实 tag 是 `stable_22Jul2025`。已在提交前完成上游核验：

- `curl -I -L https://github.com/lammps/lammps/archive/refs/tags/stable_22Jul2025.tar.gz` → **HTTP 200**（对照原值 `stable_2025.07.22` → **HTTP 404**）；
- 下载归档顶层目录为 `lammps-stable_22Jul2025/`，与 `WORKDIR /opt/lammps-stable_${VERSION}` 完全一致（已用 `tar -tzf` 实测），解压路径不会错位。

由于 `wget` / `tar` / `WORKDIR` 均基于 `${VERSION}` 拼接，改一处即同时修正下载 URL 与解压目录。该 tag 也符合 `image-info.yml` 中 `version_prefix: stable_` 与 `version_filter: patch;update` 的设定（自动化工具选取基础 stable tag）。

`README.md`、`doc/image-info.yml`、`meta.yml` 的改动仅登记镜像标识 `2025.07.22` 与目录路径，不参与下载 URL 构造，且仓库已存在 `22Jul2025-oe2403sp4` 条目（内容为 `22Jul2025_update6`），若改其标识会产生重复 key/重复行，故保持不变。

补充说明：门禁表中 `check_package_license` 仅为 WARNING（缺少仓库级 Copyright 声明文件，非本次 PR 引入），真正失败项为 x86_64/aarch64 的 `check_build`，根因即上述 404；`check_sca` 为 SUCCESS。因此无需改动 SPDX 头。

## 潜在风险
无。改动仅修正下载 tag，未改变构建流程、依赖或目录/镜像标识；`22Jul2025` 的归档目录名与 `WORKDIR` 经实测匹配，x86_64 与 aarch64 均可正常下载构建。