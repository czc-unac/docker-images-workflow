# CI 失败分析报告

## 基本信息
- PR: #4479 — 【自动升级】cp2k容器镜像升级至2026.2版本.
- 失败类型: dependency-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 缺少xz解压工具
- 新模式症状关键词: xz: Cannot exec, tar (child), not recoverable, install_libint.sh, .tar.xz, No such file or directory

## 根因分析

### 直接错误
```
#11 1551.7 ==================== Installing LIBINT ====================
#11 1551.7 wget  --quiet https://www.cp2k.org/static/downloads/libint-v2.13.1-cp2k-lmax-5.tar.xz -O libint-v2.13.1-cp2k-lmax-5.tar.xz
#11 1557.6 libint-v2.13.1-cp2k-lmax-5.tar.xz: OK
#11 1557.6 Checksum of libint-v2.13.1-cp2k-lmax-5.tar.xz Ok
#11 1557.6 Installing from scratch into /opt/cp2k/tools/toolchain/install/libint-v2.13.1-cp2k-lmax-5
#11 1557.6 tar (child): xz: Cannot exec: No such file or directory
#11 1557.6 tar (child): Error is not recoverable: exiting now
#11 1557.6 tar: Child returned status 2
#11 1557.6 tar: Error is not recoverable: exiting now
#11 1557.6 ERROR: (./scripts/stage3/install_libint.sh, line 53) Non-zero exit code detected.
ERROR: failed to solve: process "/bin/sh -c /bin/bash -c -o pipefail     \"./install_cp2k_toolchain.sh ...\"" did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `HPC/cp2k/2026.2/24.03-lts-sp4/Dockerfile:14`（`RUN ... ./install_cp2k_toolchain.sh ... --install-all ...` 步骤），实际失败点为 CP2K 工具链脚本 `/opt/cp2k/tools/toolchain/scripts/stage3/install_libint.sh:53`
- 失败原因: LIBINT 的源码包为 `libint-v2.13.1-cp2k-lmax-5.tar.xz`（xz 压缩格式），但 build 阶段首个 `yum install` 步骤安装的依赖列表中包含 `bzip2` 却**缺少 `xz`**。容器内没有 `xz` 可执行文件，`tar` 在解压 `.tar.xz` 时无法调用 `xz`，报 `xz: Cannot exec: No such file or directory`，导致 install_libint.sh 解压失败并以非零码退出，整个 `install_cp2k_toolchain.sh` 随之失败。

### 与 PR 变更的关联
本 PR 新增了 `HPC/cp2k/2026.2/24.03-lts-sp4/Dockerfile`（新文件，PR diff 中 `new_file: True`），失败完全发生在新文件第 14 行的工具链安装步骤中。新 Dockerfile 的 yum 依赖清单为：
`gcc g++ gfortran openssh-clients bzip2 ca-certificates git make patch pkgconfig unzip wget zlib-devel m4`
其中只有 `bzip2`（用于 OpenMPI 的 `.tar.bz2`），没有 `xz`（用于 LIBINT 的 `.tar.xz`）。该失败由本次 PR 新增文件直接触发，属确定性问题。

## 修复方向

### 方向 1（置信度: 高）
在 `HPC/cp2k/2026.2/24.03-lts-sp4/Dockerfile` 第一个 `yum install` 步骤的包列表中加入 `xz` 包（用于提供 `xz`/`xz-devel` 解压工具），使 `tar` 能解压 LIBINT 的 `.tar.xz` 源码包。可同时参考同类 CP2K 版本 Dockerfile 的依赖清单，确认是否还缺少其他解压工具。

### 方向 2（可选）
若希望减少对系统工具的逐个补全依赖，可在工具链调用前显式检查/安装 xz 解压能力（例如安装 `xz` 或 `xz-devel`）。但根因仍是缺包，方向 1 即可解决，方向 2 为加固措施。

## 需要进一步确认的点
- 确认 openEuler 24.03-LTS-SP4 仓库中提供 `xz` 包名（openEuler 通常为 `xz`，提供 `xz`/`xzgrep` 等命令；也可考虑 `xz-devel`）。日志显示此前 159 个包安装成功，缺失的是 xz 命令，属可修复缺包。
- 建议对照现有 `HPC/cp2k/2025.2/24.03-lts-sp4/Dockerfile`（同 OS 版本）的 yum 包列表，确认是否同样包含 `xz` 及其他本次新 Dockerfile 遗漏的解压/构建工具，避免后续工具链步骤再次因缺包失败（如 libxc、dbcsr 等仍可能有其他压缩格式源码）。

## 修复验证要求
本次修复为纯 Dockerfile 依赖补全（新增 `xz` 包），不涉及对第三方源文件正则的修改，因此无需从上游核对正则匹配。验证方式：重新触发 CI 构建，确认 `install_libint.sh` 能成功解压 `libint-v2.13.1-cp2k-lmax-5.tar.xz` 并继续通过后续工具链步骤。
