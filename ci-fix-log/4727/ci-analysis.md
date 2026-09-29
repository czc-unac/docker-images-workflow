# CI 失败分析报告

## 基本信息
- PR: #4727 — 【自动升级】cp2k容器镜像升级至2026.2版本.
- 失败类型: dependency-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 缺少xz解压工具
- 新模式症状关键词: xz, Cannot exec: No such file or directory, tar (child), install_libint.sh, .tar.xz

## 根因分析

### 直接错误
```
#11 1459.7 ==================== Installing LIBINT ====================
#11 1459.8 wget  --quiet https://www.cp2k.org/static/downloads/libint-v2.13.1-cp2k-lmax-5.tar.xz -O libint-v2.13.1-cp2k-lmax-5.tar.xz
#11 1465.6 libint-v2.13.1-cp2k-lmax-5.tar.xz: OK
#11 1465.6 Checksum of libint-v2.13.1-cp2k-lmax-5.tar.xz Ok
#11 1465.6 Installing from scratch into /opt/cp2k/tools/toolchain/install/libint-v2.13.1-cp2k-lmax-5
#11 1465.6 tar (child): xz: Cannot exec: No such file or directory
#11 1465.6 tar (child): Error is not recoverable: exiting now
#11 1465.6 tar: Child returned status 2
#11 1465.6 tar: Error is not recoverable: exiting now
#11 1465.6 ERROR: (./scripts/stage3/install_libint.sh, line 53) Non-zero exit code detected.
```

### 根因定位
- 失败位置: `HPC/cp2k/2026.2/24.03-lts-sp4/Dockerfile:5`（build 阶段的 yum install 指令），触发点在 CP2K toolchain 的 `./scripts/stage3/install_libint.sh:53`
- 失败原因: 构建阶段安装的软件包列表缺少 `xz`（含 `/usr/bin/xz`），而 CP2K toolchain 下载的 LIBINT 源码包为 `.tar.xz` 格式。`tar` 调用子进程 `xz` 解压时找不到可执行文件，报 `xz: Cannot exec: No such file or directory`，`tar` 返回非零退出码，toolchain 脚本行 53 检测到非零退出码后中止整个 `install_cp2k_toolchain.sh`，最终 Docker build 失败（exit code 1）。

### 与 PR 变更的关联
本 PR 新增了 `HPC/cp2k/2026.2/24.03-lts-sp4/Dockerfile`（全新文件）。该 Dockerfile build 阶段的依赖安装为：
```
yum install -y gcc g++ gfortran openssh-clients bzip2 ca-certificates git make patch pkgconfig unzip wget zlib-devel m4
```
列表中包含 `bzip2`，但**未包含 `xz`**。因此该失败由本 PR 新增的 Dockerfile 直接引起，而非历史遗留或基础设施问题。日志末尾为 `Finished: FAILURE`，且失败发生在 Docker 构建（build 5/8）阶段，状态与日志一致，无成功标志矛盾。

## 修复方向

### 方向 1（置信度: 高）
在 build 阶段（`FROM ${BASE} AS build` 之后的第一个 `yum install`）中补充 `xz` 软件包，使 `tar` 能解压 LIBINT 的 `.tar.xz` 源码包。可参照同一 toolchain 在其它 CP2K 版本镜像中的依赖列表，确认既有版本是否已包含 `xz`。这是本日志中可唯一确认的直接缺失项。

### 方向 2（置信度: 低）
若补充 `xz` 后 toolchain 继续在后续 stage 报其它压缩/解压工具缺失（如 `zstd`），则需一并核对 CP2K toolchain 对这些下载包的完整解压依赖。但当前日志仅能证明 `xz` 缺失，其它工具的缺失属推测，不应作为主修复项。

## 需要进一步确认的点
- 确认 openEuler 24.03-LTS-SP4 中提供 `/usr/bin/xz` 的可执行文件的包名（预期为 `xz`；`xz-devel` 仅提供头文件/开发库，通常不提供命令）。
- 对照 `HPC/cp2k/2025.2/24.03-lts-sp4/Dockerfile` 等已有版本，确认其 yum 依赖列表是否包含 `xz`，以验证修复方向与本仓库既有实践一致。
- 确认该镜像是否需在 amd64/arm64 两架构均构建；本日志仅展示了 x86_64 构建过程（`OpenBLAS detected ARCH = x86_64`），aarch64 构建是否同样缺 `xz` 需并行确认（判断依据相同，风险低）。

## 修复验证要求（仅当修复涉及正则 patch 外部源文件时填写）
不适用。本次修复方向为在 Dockerfile 依赖列表中补充系统软件包，不涉及对第三方/上游源文件的正则 patch。
