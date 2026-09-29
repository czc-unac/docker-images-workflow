# CI 失败分析报告

## 基本信息
- PR: #4723 — 【自动升级】milvus容器镜像升级至3.0.2版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 上游脚本不支持系统
- 新模式症状关键词: Unsupported Linux distribution, openEuler, install_deps.sh, milvus, Supported distributions

## 根因分析

### 直接错误
```
#13 [builder 4/4] RUN git clone -b v3.0.2 https://github.com/milvus-io/milvus.git &&     cd milvus/ &&     ./scripts/install_deps.sh && ...
#13 32.01 [INFO] Milvus Development Dependencies Installer
#13 32.01 [INFO] ==========================================
#13 32.04 go: downloading go1.26.6 (linux/amd64)
#13 40.68 [INFO] Go version: 1.26
#13 40.68 [ERROR] Unsupported Linux distribution: openEuler
#13 40.68 [INFO] Supported distributions: Ubuntu, Rocky Linux, Amazon Linux, CentOS
#13 ERROR: process "/bin/sh -c git clone -b v${VERSION} https://github.com/milvus-io/milvus.git &&     cd milvus/ &&     ./scripts/install_deps.sh &&     CXXFLAGS=\"-I/usr/include/openblas\" make build-cpp &&     make build-go" did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile:22-26`（`RUN git clone ... && ./scripts/install_deps.sh && ...` 步骤）
- 失败原因: milvus 3.0.2 上游 `scripts/install_deps.sh` 在系统发行版检测阶段遇到 `openEuler` 时显式报错退出（脚本仅声明支持 Ubuntu、Rocky Linux、Amazon Linux、CentOS），导致该 RUN 指令以 exit code 1 失败，Docker 构建中止。

### 与 PR 变更的关联
本 PR 新增 `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`，其中直接调用上游 `milvus` 仓库的 `./scripts/install_deps.sh`。该脚本不支持 openEuler 发行版，因此该新增文件引入的构建步骤必然失败。README.md / image-info.yml / meta.yml 的改动为元数据登记，非失败来源。失败类型与 PR 改动直接相关，而非基础设施问题。

## 修复方向

### 方向 1（置信度: 高）
新增 Dockerfile 已通过 `yum install` 自行安装了编译依赖（gcc/g++/cmake/openblas-devel/conan 等）。可考虑跳过或替换上游 `./scripts/install_deps.sh`（该脚本会在不支持的系统上直接退出），改为直接执行 `make build-cpp` 与 `make build-go`，并补齐脚本原本负责安装的缺失依赖。需先确认 milvus 3.0.2 的 C++ 构建实际所需依赖，避免因跳过脚本而遗漏组件。

### 方向 2（置信度: 中）
若必须保留 `install_deps.sh`，则需要使其识别 openEuler（例如让脚本检测到的发行版映射到 CentOS/Rocky 分支）。验证方式：以 Dockerfile 中 `ARG VERSION=3.0.2` 对应的上游 tag（日志显示实际 checkout 到 commit `3c4448a2`）为准，拉取该版本的 `scripts/install_deps.sh`，确认其发行版检测逻辑与可接受的替代分支。

## 需要进一步确认的点
- 需确认 `scripts/install_deps.sh`（v3.0.2 / commit 3c4448a2）中发行版检测的具体实现：是否仅按 `/etc/os-release` 的 ID 判断，以及是否存在可用的 CentOS/Rocky 兼容分支可直接复用。
- 需确认跳过 `install_deps.sh` 后，milvus 的 `make build-cpp` / `make build-go` 是否依赖脚本安装的系统包（如第三方 C++ 库、protobuf、grpc 等），以决定是否需要在 Dockerfile 的 `yum install` 步骤补包。
- 日志中仅捕获到 x86_64 构建（`go/src/...`、`linux/amd64`）的失败；若该镜像声明 `amd64, arm64`，需确认 aarch64 构建是否同样因该发行版检测失败。

## 修复验证要求
- code-fixer 必须从 milvus v3.0.2（上游 tag / commit `3c4448a2aee506ccd14851e742aba85f1c83188a`）获取 `scripts/install_deps.sh` 的实际内容，确认其发行版检测分支及支持列表，再决定是"跳过脚本"还是"补丁脚本使其支持 openEuler"。
- 若采用补丁/Sed 方式修改上游脚本，必须对照上述实际文件内容验证匹配目标存在且能生效。
- 修复后本地/CI 至少需验证 `make build-cpp` 与 `make build-go` 能在 openEuler 24.03-LTS-SP4 上完成。
