# 修复摘要

## 修复的问题
LAMMPS 新镜像 Dockerfile 用“点分日期”版本号拼出上游不存在的 tag `stable_2025.07.22`，导致 wget 下载 404、Docker 构建失败；已将其修正为上游真实存在的 `stable_22Jul2025`。

## 修改的文件
- `HPC/lammps/2025.07.22/24.03-lts-sp4/Dockerfile`: 第 4 行 `ARG VERSION=2025.07.22` 改为 `ARG VERSION=22Jul2025`。

## 修复逻辑
- 分析报告根因：`HPC/lammps/2025.07.22/24.03-lts-sp4/Dockerfile:13` 的下载逻辑用 `ARG VERSION` 拼出 URL `.../refs/tags/stable_${VERSION}.tar.gz`，而 `VERSION=2025.07.22` 拼出的 tag 在上游 `github.com/lammps/lammps` 不存在（302 重定向到 codeload 后返回 404，exit code 8）。
- 上游 LAMMPS stable tag 采用 `stable_<DDMonYYYY>` 命名，2025-07-22 发布版对应 tag 为 `stable_22Jul2025`。已从上游核实：
  - `https://github.com/lammps/lammps/archive/refs/tags/stable_22Jul2025.tar.gz` → HTTP 200（存在）
  - `https://github.com/lammps/lammps/archive/refs/tags/stable_2025.07.22.tar.gz` → HTTP 404（不存在）
- 因此将 Dockerfile 中 `VERSION` 改为 `22Jul2025`，使 wget 构造出 `stable_22Jul2025.tar.gz` 正常下载；随后 `WORKDIR /opt/lammps-stable_${VERSION}` = `lammps-stable_22Jul2025`，与 GitHub 归档解压目录名一致。
- 该修复只改动触发失败的唯一变量，未改动 README.md、doc/image-info.yml、meta.yml：三者的 tag/路径均由目录名 `2025.07.22` 派生，而约束禁止改名/新增文件，目录名保持不变即无构建影响。
- 本修复不涉及对第三方/上游源文件做正则 patch，故无需正则验证流程；已按要求从上游核实所选 tag 确实存在（HTTP 200）。

## 潜在风险
- 镜像目录名仍为 `2025.07.22`，镜像 tag 仍为 `2025.07.22-oe2403sp4`，但实际构建的是上游 `22Jul2025`（基础发布版），与已有的 `22Jul2025-oe2403sp4`（`_update6`）存在版本标识不一致，属于命名/展示层面，不影响本次构建通过。
- `doc/image-info.yml` 的 `upstream.version_scheme: RPM` / `version_prefix: stable_` 会让自动升级持续产出“点分日期”形式的非法 tag，后续可能再次触发同类 404；但该配置调整属于预防性改动、且需上游版本解析行为确认，超出本次最小修复范围，未改动。