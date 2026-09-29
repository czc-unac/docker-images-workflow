# 修复摘要

## 修复的问题
milvus v3.0.2 上游 `scripts/install_deps.sh` 的发行版白名单不识别 openEuler，脚本主动 `exit 1` 导致 Docker 构建失败；已通过 patch 该脚本的发行版检测分支，让 openEuler 走通受支持分支后继续执行官方依赖安装。

## 修改的文件
- `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`:
  - 在 `git clone` 之后、`./scripts/install_deps.sh` 之前新增一行 `sed -i 's/amzn)/amzn|openEuler|openeuler)/' scripts/install_deps.sh`，把上游脚本 `main()` 中 `amzn)` 分支扩展为 `amzn|openEuler|openeuler)`，恢复 `./scripts/install_deps.sh` 调用。
  - （分支既有）构建依赖与工具版本已对齐 milvus v3.0.2 官方 builder：Go 1.26.6、Conan 2.25.1、Rust 1.92，并补齐 `autoconf/automake/libtool/m4/ccache/python3-devel/libstdc++-static/libaio-devel/openssl-devel/zlib-devel/perl-Digest-SHA` 等系统包。

## 修复逻辑
分析报告根因：v3.0.2 的 `scripts/install_deps.sh` 通过 `detect_linux_distro()` 读取 `/etc/os-release` 的 `$ID`，`main()` 的 `case` 仅匹配 `ubuntu|debian`、`rocky|almalinux`、`amzn`、`centos|rhel`，openEuler 落入 `*)` 分支后打印 `Unsupported Linux distribution: openEuler` 并 `exit 1`。

本修复采用分析报告**方向 1（置信度: 高）**：保留上游依赖安装脚本，仅适配其发行版检测逻辑。选择将 openEuler 归入 `amzn` 分支而非 `centos`/`rocky`，原因是 openEuler 24.03 自带 `dnf` 且软件包为 RHEL 系命名，而 `install_centos_deps` 依赖 `devtoolset-11`/`llvm-toolset-11.0`（openEuler 无此 SCL 包）、`install_rocky_deps` 依赖 `epel-release`/`crb` 仓库（openEuler 无），都会失败；`install_amazon_linux_deps` 仅使用 `dnf` + 通用包名，是最兼容的受支持分支。

验证过程（上游实际文件）：
- 已从 GitHub 拉取 milvus v3.0.2 的 `scripts/install_deps.sh`（共 523 行），确认 `detect_linux_distro()` 读取 `/etc/os-release` 的 `$ID`，`main()` 的 `case` 中 `amzn)` 位于第 503 行且全文仅出现一次。
- 已在内存中用 Python `re.sub` 及 `sed -i 's/amzn)/amzn|openEuler|openeuler)/'` 分别验证正则，均匹配 1 处，执行后第 503 行变为 `amzn|openEuler|openeuler)`，`install_amazon_linux_deps` 保持被调用。
- 已确认 `install_amazon_linux_deps` 安装的包（`ninja-build`、`gcc-c++`、`gcc-gfortran`、`libaio`、`libuuid-devel`、`ccache`、`libtool`、`m4`、`autoconf`、`automake`、`openssl-devel`、`zlib-devel` 等）在 openEuler 仓库均存在（仓库内其它 openEuler Dockerfile 已在用同名包）。
- 已拉取上游 `build/docker/builder/cpu/amazonlinux2023/Dockerfile` 交叉核对：官方基线为 Go 1.26.6、Conan 2.25.1、Rust 1.92，与本 Dockerfile 一致；`install_deps.sh` 会自动执行 `conan profile detect` 并校验 conan/rust，无需额外配置。
- 已确认除 `install_deps.sh` 外，构建链路中的 `scripts/core_build.sh`（仅按 `$ID` 决定是否开启 DiskANN，不报错）和 `scripts/3rdparty_build.sh`（仅打印 OS 名）不存在发行版白名单阻塞。

## 潜在风险
- 若 openEuler 仓库未来移除某个被 `install_amazon_linux_deps` 引用的包名，`dnf install` 会失败；本次已核对当前包名存在。
- openEuler 不在 `ubuntu/rocky/amzn` 之列时 `core_build.sh` 不开启 `BUILD_DISK_ANN`，属功能裁剪，不影响构建失败问题的修复。
- 本环境无法实际运行 openEuler 容器做端到端构建验证，结论基于上游 v3.0.2 源码、官方 builder 镜像与 openEuler 仓库包清单的交叉核对，最终以 CI 构建结果为准。