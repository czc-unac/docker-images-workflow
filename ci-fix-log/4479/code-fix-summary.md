# 修复摘要

## 修复的问题
在 CP2K 2026.2 构建阶段的 yum 依赖列表中补充 `xz` 包，使 `tar` 能解压 LIBINT 的 `.tar.xz` 源码包，解决 `xz: Cannot exec: No such file or directory` 导致的工具链安装失败。

## 修改的文件
- `HPC/cp2k/2026.2/24.03-lts-sp4/Dockerfile`: 第 7 行 `yum install` 列表在 `bzip2` 后新增 `xz` 包。

## 修复逻辑
CI 日志显示构建在 `install_cp2k_toolchain.sh` 调用 `scripts/stage3/install_libint.sh:53` 时失败，`tar` 解压 `libint-v2.13.1-cp2k-lmax-5.tar.xz` 时无法调用 `xz`。根因是 build 阶段首个 `yum install` 只安装了 `bzip2`（用于 OpenMPI 的 `.tar.bz2`），缺少 `xz`（用于 LIBINT 的 `.tar.xz`）。本次仅补全该缺包，属纯 Dockerfile 依赖补全，不涉及第三方源文件正则修改，无需上游正则核对。改动与 PR 新增文件 `HPC/cp2k/2026.2/24.03-lts-sp4/Dockerfile` 直接对应。

## 潜在风险
无。新增 `xz` 包仅提供解压工具，不影响既有构建流程；该修复未改动与 CI 失败无关的其他文件。