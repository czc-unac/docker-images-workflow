# 修复摘要

## 修复的问题
在 cp2k 2026.2 镜像 build 阶段的依赖列表中补充 `xz`，解决 CP2K toolchain 解压 LIBINT `.tar.xz` 源码包时缺少 `xz` 可执行文件导致的 Docker 构建失败。

## 修改的文件
- `HPC/cp2k/2026.2/24.03-lts-sp4/Dockerfile`: build 阶段第一个 `yum install` 的软件包列表由 `bzip2` 改为 `bzip2 xz`

## 修复逻辑
CI 失败根因为 `tar (child): xz: Cannot exec: No such file or directory`，触发点在 CP2K toolchain 的 `./scripts/stage3/install_libint.sh:53`。toolchain 下载的 LIBINT 源码包为 `libint-v2.13.1-cp2k-lmax-5.tar.xz`，`tar` 需要调用 `/usr/bin/xz` 解压，但 build 阶段安装的软件包列表缺少 `xz`，导致解压失败、脚本返回非零退出码，最终 `install_cp2k_toolchain.sh` 中止、Docker build 失败。

修复方式与分析报告的"方向 1（置信度: 高）"一致：在 build 阶段 `yum install` 中补充 `xz` 软件包（openEuler 中由 `xz` 包提供 `/usr/bin/xz`）。改动为单行、最小化，不涉及无关重构。

## 潜在风险
无。仅新增一个解压工具软件包，不影响既有构建流程；运行时阶段未改动（运行镜像仅执行已编译好的 cp2k 二进制，无需解压 `.tar.xz`）。若后续 toolchain 继续报其它解压工具缺失（分析报告方向 2，低置信度），需另行核对，属本次修复范围之外。