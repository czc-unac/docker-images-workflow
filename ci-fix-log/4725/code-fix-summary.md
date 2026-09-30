# 修复摘要

## 修复的问题
LAMMPS 2025.07.22 镜像构建时按 `stable_${VERSION}.tar.gz` 下载 `stable_2025.07.22.tar.gz` 返回 404（LAMMPS 上游 release tag 使用 `DDMonYYYY` 格式，对应 tag 为 `stable_22Jul2025`）。

## 修改的文件
- 无新增修改。当前 `fix/4725` 分支已包含正确修复，且该修复已通过真实 CI 验证：
  - `HPC/lammps/2025.07.22/24.03-lts-sp4/Dockerfile`：
    - `ARG LAMMPS_TAG=22Jul2025`（上游真实 tag）；
    - 下载/解压/WORKDIR 全部改用 `stable_${LAMMPS_TAG}`，即 `stable_22Jul2025.tar.gz` / `/opt/lammps-stable_22Jul2025`；
    - 保留 `ARG VERSION=2025.07.22` 作为镜像版本标识，与目录名、`meta.yml`、`README.md`、`doc/image-info.yml` 中的 `2025.07.22-oe2403sp4` 保持一致。
- `HPC/lammps/README.md`、`HPC/lammps/doc/image-info.yml`、`HPC/lammps/meta.yml`：均与修正后的版本标识一致，无需改动。

## 修复逻辑
1. 根因是自动升级生成的版本串 `2025.07.22` 被直接拼进下载 URL，而 LAMMPS 上游 tag 为 `22Jul2025`，导致 GitHub 返回 404（exit code 8）。
2. 将“镜像版本标识”与“上游下载 tag”解耦：`VERSION` 保持面向用户的日期版本 `2025.07.22`，新增 `LAMMPS_TAG=22Jul2025` 专用于构造下载地址。这样既不改变镜像对外版本（`2025.07.22-oe2403sp4`），又能命中上游真实 tag。
3. 已做上游核实：
   - `stable_2025.07.22.tar.gz` → HTTP 404（复现问题）；
   - `stable_22Jul2025.tar.gz` → HTTP 200；实际下载后 tar 顶层目录为 `lammps-stable_22Jul2025`，与 Dockerfile 的 `WORKDIR /opt/lammps-stable_${LAMMPS_TAG}` 完全一致；`examples/melt/in.melt` 与 `src/Makefile` 均存在。
4. 说明：本分支早前的修复尝试 `c539c8ce9`（直接把 `VERSION` 改成 `22Jul2025`）在 CI 的 aarch64 构建上失败；当前提交 `64500a8c0` 的形态在真实 CI 中已通过（见下），因此不应对其再做改动。

## CI 验证结论
- GitCode 修复 PR **#4751**（head sha `64500a8c0`，即当前 fix 分支 HEAD）的构建结果：x86_64 ✅ SUCCESS、aarch64 ✅ SUCCESS，标签 `ci_successful`。
- 对照 PR **#4746**（head sha `c539c8ce9`）：x86_64 ✅ SUCCESS、aarch64 ❌ FAILED。
- 因此当前源码状态即为 CI 验证通过的最终修复，本次不再产生新的代码修改，以免回归已验证通过的构建。

## 潜在风险
无。改动仅影响 `HPC/lammps/2025.07.22/24.03-lts-sp4/Dockerfile` 内部的下载 tag 与工作目录变量，镜像对外版本标识（目录名 / meta / README / image-info 中的 `2025.07.22-oe2403sp4`）保持不变。