# CI 失败分析报告

## 基本信息
- PR: #4645 — 【自动升级】cp2k容器镜像升级至2026.2版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式（与模式10「缺少构建依赖」同类）
- 新模式标题: 缺少xz解压工具
- 新模式症状关键词: `xz: Cannot exec`, `tar.xz`, `install_libint.sh`, `tar (child)`, `libint`

## 根因分析

### 直接错误
```
#11 611.6 ==================== Installing LIBINT ====================
#11 611.6 wget  --quiet https://www.cp2k.org/static/downloads/libint-v2.13.1-cp2k-lmax-5.tar.xz -O libint-v2.13.1-cp2k-lmax-5.tar.xz
#11 615.1 libint-v2.13.1-cp2k-lmax-5.tar.xz: OK
#11 615.1 Checksum of libint-v2.13.1-cp2k-lmax-5.tar.xz Ok
#11 615.1 Installing from scratch into /opt/cp2k/tools/toolchain/install/libint-v2.13.1-cp2k-lmax-5
#11 615.1 tar (child): xz: Cannot exec: No such file or directory
#11 615.1 tar (child): Error is not recoverable: exiting now
#11 615.1 tar: Child returned status 2
#11 615.1 tar: Error is not recoverable: exiting now
#11 615.1 ERROR: (./scripts/stage3/install_libint.sh, line 53) Non-zero exit code detected.
ERROR: failed to solve: process "...install_cp2k_toolchain.sh..." did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `HPC/cp2k/2026.2/24.03-lts-sp4/Dockerfile:14`（`install_cp2k_toolchain.sh` 阶段）；直接失败点为上游 `tools/toolchain/scripts/stage3/install_libint.sh:53`
- 失败原因: 容器内缺少 `xz` 可执行文件，`tar` 无法解压 `libint-v2.13.1-cp2k-lmax-5.tar.xz`（`.tar.xz` 格式需调用 `xz`）。Dockerfile 第一个 `yum install` 列表安装了 `bzip2`（对应 `.bz2`）却未安装 `xz`，因此工具链在其他包（OpenMPI/OpenBLAS/FFTW/Eigen，均为 `.tar.gz`/`.tar.bz2`）阶段正常通过，直到 LIBINT 这一 `.tar.xz` 包才暴露。

### 与 PR 变更的关联
本 PR 新增了 `HPC/cp2k/2026.2/24.03-lts-sp4/Dockerfile`（new_file，88 行），其 `yum install` 依赖清单为：
`gcc g++ gfortran openssh-clients bzip2 ca-certificates git make patch pkgconfig unzip wget zlib-devel m4`
——缺 `xz`。因此失败由本 PR 新增的 Dockerfile 直接触发，属于新增文件的依赖遗漏，而非原有历史问题。

## 修复方向

### 方向 1（置信度: 高）
在 `HPC/cp2k/2026.2/24.03-lts-sp4/Dockerfile` 构建阶段第一个 `yum install -y ...` 的包列表中补充 `xz`（提供 `/usr/bin/xz`，供 `tar` 解压 `.tar.xz`）。参考对照：CP2K 旧版本（如 2025.2 / 2024.3）Dockerfile 的同类依赖清单，确认其是否已含 `xz`，若无则同步补齐。这是与本日志证据最直接对应的修复。

### 方向 2（可选）
若 `xz` 包名在 openEuler 24.03-LTS-SP4 中仅由 `xz-devel` / `xz-libs` 间接提供，需确认提供 `xz` 可执行文件的准确 RPM 包名后再加入安装列表；可通过 `dnf provides '*/xz'` 在目标基础镜像中核实（由 code-fixer 在构建环境验证）。

## 需要进一步确认的点
- 工具链阶段是否还有其他被禁用的可选包（日志显示 `libtorch`、`libgint` 等已被禁用/跳过）后续还会触发额外缺失依赖；本次日志仅到 LIBINT 即失败，无法确认后续步骤是否还有其它缺失包。建议修复后在构建中继续观察下游阶段。
- openEuler 24.03-LTS-SP4 中提供 `xz` 可执行文件的精确包名，需以目标基础镜像实际 `dnf` 元数据为准。

## 修复验证要求
本次修复方向为「在 Dockerfile 的 yum 安装列表补充系统包」，不涉及对第三方/上游源文件的正则 patch，故无需执行上游正则可匹配性验证。由 code-fixer 完成修改后，需实际触发一次 Docker 构建，确认 LIBINT 及后续工具链步骤可正常解压 `.tar.xz` 并继续构建。
