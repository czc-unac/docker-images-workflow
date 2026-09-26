# CI 失败分析报告

## 基本信息
- PR: #4563 — 【自动升级】cp2k容器镜像升级至2026.2版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 缺少解压工具xz
- 新模式症状关键词: xz, Cannot exec, tar (child), Error is not recoverable, .tar.xz, install_cp2k_toolchain

## 根因分析

### 直接错误
```
#11 1364.5 ==================== Installing LIBINT ====================
#11 1364.5 wget --quiet https://www.cp2k.org/static/downloads/libint-v2.13.1-cp2k-lmax-5.tar.xz -O libint-v2.13.1-cp2k-lmax-5.tar.xz
#11 1370.4 libint-v2.13.1-cp2k-lmax-5.tar.xz: OK
#11 1370.4 Checksum of libint-v2.13.1-cp2k-lmax-5.tar.xz Ok
#11 1370.4 Installing from scratch into /opt/cp2k/tools/toolchain/install/libint-v2.13.1-cp2k-lmax-5
#11 1370.4 tar (child): xz: Cannot exec: No such file or directory
#11 1370.4 tar (child): Error is not recoverable: exiting now
#11 1370.4 tar: Child returned status 2
#11 1370.4 tar: Error is not recoverable: exiting now
#11 1370.4 ERROR: (./scripts/stage3/install_libint.sh, line 53) Non-zero exit code detected.
```

### 根因定位
- 失败位置: `HPC/cp2k/2026.2/24.03-lts-sp4/Dockerfile`（新增文件）第一段 `RUN yum install ...` 的依赖列表；实际报错触发点在工具链脚本 `./tools/toolchain/scripts/stage3/install_libint.sh:53`
- 失败原因: Dockerfile 的构建阶段安装的包列表（`gcc g++ gfortran openssh-clients bzip2 ca-certificates git make patch pkgconfig unzip wget zlib-devel m4`）中**缺少 `xz`**，而 CP2K 2026.2 工具链安装 LIBINT 时需要解压 `libint-v2.13.1-cp2k-lmax-5.tar.xz`（xz 压缩格式），`tar` 调用外部 `xz` 解压器失败（`xz: Cannot exec: No such file or directory`），导致 `install_libint.sh` 返回非零并中断整个 toolchain 安装。

### 与 PR 变更的关联
PR 新增的 Dockerfile 是本次失败的唯一致因：该文件首次引入 cp2k 2026.2 的构建，其依赖安装列表遗漏了 `xz`。日志中 `openmpi-5.0.10.tar.bz2`（bzip2 格式，已安装 `bzip2`）解压成功，而随后 `libint-...tar.xz`（xz 格式）解压失败，进一步印证是缺少 xz 解压工具而非下载或校验问题（xz 包本身下载 OK、Checksum Ok）。

## 修复方向

### 方向 1（置信度: 高）
在 `HPC/cp2k/2026.2/24.03-lts-sp4/Dockerfile` 构建阶段的 `yum install -y` 列表中加入 `xz`（openEuler 中提供 `xz` 命令的包名通常为 `xz`；如后续编译也需要头文件，可一并加入 `xz-devel`）。加入后即可让 `tar` 正常解压 `libint-v2.13.1-cp2k-lmax-5.tar.xz`，使 `install_cp2k_toolchain.sh` 继续执行。

### 方向 2（可选）
对比同目录/同项目其它 CP2K 版本（如 `2025.2/24.03-lts-sp4`）Dockerfile 的依赖列表，确认该版本升级是否还遗漏了 2026.2 新增的其它构建依赖（如上游 toolchain 新增的包）。本报告日志仅在 LIBINT 阶段中断，无法覆盖后续阶段是否还有其他依赖缺失。

## 需要进一步确认的点
- 确认 openEuler 24.03-LTS-SP4 仓库中 `xz` 包的确切包名（`xz` 或 `xz-utils`）可用，且同时满足运行阶段需要（若运行阶段也依赖 xz 解压，需同步补充）。
- 由于日志在 LIBINT 阶段即告失败，无法验证 2026.2 工具链后续阶段（libxc/gauxc/libxsmm 等）是否存在新的依赖或版本问题，修复后需重新触发完整构建确认。

## 修复验证要求
不涉及对第三方/上游源文件的正则 patch，无需额外上游文件核验。修复验证方式：重新运行 CI 构建，确认 `Installing LIBINT` 阶段不再出现 `xz: Cannot exec`，且 `install_cp2k_toolchain.sh` 能继续执行直至完成。
