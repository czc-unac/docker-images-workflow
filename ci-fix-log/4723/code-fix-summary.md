# 修复摘要

## 修复的问题
milvus 3.0.2 构建阶段直接调用上游 `scripts/install_deps.sh`，该脚本显式拒绝 openEuler 发行版（`Unsupported Linux distribution: openEuler`），导致 Docker 构建失败。已改为在 Dockerfile 中自行安装完整构建依赖并直接执行构建，不再调用该脚本。

## 修改的文件
- `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`:
  - 删除 `./scripts/install_deps.sh` 调用（该脚本 v3.0.2 的发行版检测仅支持 Ubuntu/Rocky/Amazon Linux/CentOS，检测到 `openEuler` 时 `exit 1`）。
  - 按 milvus 3.0.2 上游 builder 镜像的实际需求补齐 yum 依赖：新增 `curl autoconf automake libtool m4 ccache python3-devel tar libstdc++-static libaio-devel openssl-devel zlib-devel perl-Digest-SHA`，并将 `gfortran` 归入主依赖行。
  - 升级构建工具版本以匹配 milvus 3.0.2 要求：`GOLANG_VERSION` 1.24.2 → 1.26.6，Rust toolchain 1.73 → 1.92，Conan 1.61.0 → 2.25.1。

## 修复逻辑
根因是新增 Dockerfile 依赖上游 `install_deps.sh`，而该脚本在 v3.0.2 重写后按 `/etc/os-release` 的 `ID` 做白名单校验，openEuler 不在白名单内直接退出（对应分析报告"上游脚本不支持系统"模式、失败位置 `Dockerfile:22-26`）。

修复采用分析报告的高置信度方向 1：跳过 `install_deps.sh`，由 Dockerfile 自行提供依赖并直接 `make build-cpp` / `make build-go`。

验证依据（均为上游实际文件）：
- 已从 GitHub 拉取 milvus v3.0.2（tag 对应 commit `3c4448a2aee506ccd14851e742aba85f1c83188a`）的 `scripts/install_deps.sh`，确认其发行版检测逻辑为 `detect_linux_distro()` 读取 `/etc/os-release` 的 `$ID`，`main()` 的 case 仅匹配 `ubuntu|debian`、`rocky|almalinux`、`amzn`、`centos|rhel`，其余走 `*)` 分支打印 `Unsupported Linux distribution: ${distro}` 并 `exit 1`。
- 已拉取上游 `build/docker/builder/cpu/amazonlinux2023/Dockerfile` 与 `build/docker/builder/cpu/rockylinux9/Dockerfile`，确认 milvus 3.0.2 官方 builder 的基线为：Go 1.26.6、Conan 2.25.1、Rust 1.92、CMake 3.27、以及 `gcc/g++/gcc-gfortran/ninja-build/autoconf/automake/python3-devel/perl-IPC-Cmd/perl-Digest-SHA/libstdc++-static/libaio-devel` 等系统包。本修复据此对齐。
- 已确认 `scripts/3rdparty_build.sh`（v3.0.2）会在构建时自动校验并安装 `conan==2.25.1`、自动执行 `conan profile detect`，因此移除 `install_deps.sh` 不会缺失 Conan 配置。
- 已核对 openEuler 24.03-LTS-SP4 仓库（OS/everything/EPOL）中新增的所有包名均存在：`curl, autoconf, automake, libtool, m4, ccache, python3-devel, tar, libstdc++-static, libaio-devel, openssl-devel, zlib-devel, perl-Digest-SHA` 等；且 openEuler 自带 `cmake 3.31.12`（>= 脚本要求的 3.26）与 `gcc 12.3`（在要求的 11-14 范围内）。
- 已确认 Go 1.26.6 在 `golang.google.cn` 与 `go.dev` 均可用。

## 潜在风险
- 移除 `install_deps.sh` 后，系统依赖完全由 Dockerfile 的 yum 列表保证。若 milvus 3.0.2 后续补丁引入新的系统依赖，需同步维护该列表（本次已按上游 builder 镜像对齐）。
- `scripts/core_build.sh` 依据 `/etc/os-release` 的 `$ID` 判断是否开启 `BUILD_DISK_ANN`（仅 ubuntu/rocky/amzn 开启）。openEuler 下磁盘索引（DiskANN）不会启用；该行为属于功能裁剪，不影响本次构建失败问题的修复。
- 未能在此环境实际执行 Docker 构建（无法运行 openEuler 容器做端到端验证），结论基于上游源码、上游 builder 镜像与 openEuler 仓库包清单的交叉核对，建议以 CI 实际构建结果为准。