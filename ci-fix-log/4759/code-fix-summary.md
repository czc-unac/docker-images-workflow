# 修复摘要

## 修复的问题
milvus v3.0.2 镜像在 openEuler 24.03-LTS-SP4 上构建失败：上游 v3.0.2 的构建体系已换代（Go 1.26.6 / Rust 1.92 / Conan 2.25.1 / CMake≥3.26，且 `install_deps.sh` 发行版白名单不含 openEuler），同时 MinIO 旧下载路径 `dl.min.io/server/minio/release/...` 被上游永久下架返回 HTTP 410。按上游 v3.0.2 实际要求对齐构建环境并修正失效下载源。

## 修改的文件
- `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`:
  - `GOLANG_VERSION` 1.24.2 → 1.26.6（对齐上游 `go.mod` 的 `go 1.26.6`）。
  - `yum install` 补齐构建依赖：`curl autoconf automake libtool m4 ccache python3-devel tar libstdc++-static libaio-devel openssl-devel zlib-devel perl-Digest-SHA`，并移除发行版自带 `cmake`。
  - 新增安装官方预编译 CMake 3.31.8 到 `/usr/local` 并前置 `PATH`（满足 v3.0.2 的 CMake≥3.26 要求，同时避免系统 cmake 动态链接 libldap/OpenSSL 引发符号冲突）。
  - Rust toolchain 1.73 → 1.92、Conan 1.61.0 → 2.25.1（对齐上游 v3.0.2 `scripts/install_deps.sh` 基线）。
  - 在 `./scripts/install_deps.sh` 前用 `sed` 将发行版分支 `amzn)` 扩展为 `amzn|openEuler|openeuler)`，使 openEuler 被脚本识别为受支持的 dnf/RHEL 系发行版。
  - MinIO 下载地址 `https://dl.min.io/server/minio/release/linux-$TARGETARCH/minio` → `https://dl.min.io/aistor/minio/release/linux-$TARGETARCH/minio`。

## 修复逻辑
1. **取得真实失败证据**：分析报告因无日志判定为 infra-error/证据不足，因此直接抓取 PR #4759 门禁失败构建的架构日志定位根因：
   - x86_64：`https://log-ci.openeuler.openatom.cn/job/multiarch/openeuler/x86-64/openeuler-docker-images/4870/`
   - aarch64：`https://log-ci.openeuler.openatom.cn/job/multiarch/openeuler/aarch64/openeuler-docker-images/4966/`
   两架构均在 `stage-1 4/9 RUN curl -fSL -o minio https://dl.min.io/server/minio/release/linux-$TARGETARCH/minio` 处失败：`curl: (22) The requested URL returned error: 410`，`dl.min.io` 已公告 OSS 制品永久停止分发（含 archive 路径），故改为官方现存分发路径 `aistor`。
2. **对齐上游 v3.0.2 构建要求**（从 `milvus-io/milvus` tag `v3.0.2` 拉取实际文件核对）：
   - `go.mod` 声明 `go 1.26.6`；`scripts/install_deps.sh` 要求 `CONAN_VERSION=2.25.1`、`RUST_VERSION=1.92`、`MIN_CMAKE_VERSION=3.26`。
   - 该脚本的 `detect_linux_distro()` 读取 `/etc/os-release` 的 `ID`（openEuler 为 `openEuler`），`main()` 的 case 白名单仅含 ubuntu/rocky/almalinux/amzn/centos/rhel，openEuler 落入 `*)` 分支打印 `Unsupported Linux distribution: openEuler` 并 `exit 1`，导致后续 `make build-cpp`/`make build-go` 从未执行。
3. **正则 patch 外部源文件验证**：已从上游 `https://raw.githubusercontent.com/milvus-io/milvus/v3.0.2/scripts/install_deps.sh` 获取实际文件（523 行），在内存中验证 `sed -i -E 's/^([[:space:]]*)amzn[[:space:]]*\)/\1amzn|openEuler|openeuler)/'`，确认唯一匹配第 503 行 `                amzn)` 并正确改写为 `                amzn|openEuler|openeuler)`，正则匹配成功（`bash -n` 语法检查通过）。选择将该分支归入 `amzn`（Amazon Linux）分支，因其仅通过 dnf 安装标准包，不执行 openEuler 上不可用的 `epel-release`/`crb` 配置。
4. **失效下载源验证**：`https://dl.min.io/aistor/minio/release/linux-amd64/minio` → HTTP 200、`content-length: 278278306`；`linux-arm64` → HTTP 200、`content-length: 256114850`。旧路径 `https://dl.min.io/server/minio/release/linux-amd64/minio` → HTTP 410，确认旧路径失效。
5. `README.md`、`doc/image-info.yml`、`meta.yml` 的新增条目与既有 2.x 条目格式一致，无需改动。

## 潜在风险
- 归入 `amzn` 分支后脚本会执行 `sudo dnf install`，所安装包名需在 openEuler 24.03-LTS-SP4 仓库存在；Dockerfile 首个 `yum install` 已预装大部分等价依赖，风险较低。
- `sed` 依赖上游脚本中 `amzn)` 分支标签存在；当前已针对固定 tag `v3.0.2` 验证匹配成功，后续 milvus 版本若重写发行版检测逻辑需同步调整。
- 该修复涉及的 v3.0.2 构建链较长（Conan 第三方依赖、Rust/Arrow 等），若后续日志仍在编译/链接阶段失败，需依据新日志继续定位。