# CI 失败分析报告

## 基本信息
- PR: #4723 — 【自动升级】milvus容器镜像升级至3.0.2版本.
- 失败类型: `build-error`
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 安装脚本不识别openEuler
- 新模式症状关键词: Unsupported Linux distribution, openEuler, install_deps.sh, exit code: 1, Supported distributions

## 根因分析

### 直接错误
```
#13 [builder 4/4] RUN git clone -b v3.0.2 https://github.com/milvus-io/milvus.git &&     cd milvus/ &&     ./scripts/install_deps.sh &&     CXXFLAGS="-I/usr/include/openblas" make build-cpp &&     make build-go
#13 32.01 [INFO] Milvus Development Dependencies Installer
#13 32.01 [INFO] ==========================================
#13 32.04 go: downloading go1.26.6 (linux/amd64)
#13 40.68 [INFO] Go version: 1.26
#13 40.68 [ERROR] Unsupported Linux distribution: openEuler
#13 40.68 [INFO] Supported distributions: Ubuntu, Rocky Linux, Amazon Linux, CentOS
#13 ERROR: process "/bin/sh -c git clone -b v${VERSION} ..." did not complete successfully: exit code: 1
Dockerfile:22
  22 | >>> RUN git clone -b v${VERSION} https://github.com/milvus-io/milvus.git && \
  23 | >>>     cd milvus/ && \
  24 | >>>     ./scripts/install_deps.sh && \
  25 | >>>     CXXFLAGS="-I/usr/include/openblas" make build-cpp && \
  26 | >>>     make build-go
ERROR: failed to solve: ... did not complete successfully: exit code: 1
Finished: FAILURE
```

### 根因定位
- 失败位置: `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile:22`（`./scripts/install_deps.sh` 调用）
- 失败原因: milvus 上游 `${VERSION}`(v3.0.2) 自带的 `scripts/install_deps.sh` 只识别 Ubuntu / Rocky Linux / Amazon Linux / CentOS，检测到 openEuler 后直接打印 `Unsupported Linux distribution: openEuler` 并以 exit code 1 终止，导致后续 `make build-cpp` / `make build-go` 从未执行。

### 与 PR 变更的关联
本 PR 新增了 `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`（new_file），该 Dockerfile 在 `openeuler/openeuler:24.03-lts-sp4` 基础镜像中克隆 milvus v3.0.2 并调用其 `install_deps.sh`。失败与 PR 新增内容直接相关——这是一个全新的镜像版本，首次在 openEuler 上执行 milvus 的依赖安装脚本，脚本的发行版白名单不含 openEuler。

## 修复方向

### 方向 1（置信度: 高）
让 milvus 的依赖安装流程绕开/适配 openEuler 检测：在调用 `install_deps.sh` 前对其发行版识别逻辑做处理（例如使脚本将 openEuler 视为已支持的 RHEL 系发行版，或直接跳过其发行版检查、改为在 Dockerfile 中用 yum 手工安装编译所需依赖）。核心是让 openEuler 被脚本接受，或不再依赖该脚本的发行版分支。

### 方向 2（置信度: 中）
参考仓库内其他基于 RHEL 系发行版的 milvus 构建方式：如果同场景已有可在 openEuler 上成功构建的 milvus 版本，可复用其依赖安装做法，而非直接调用上游 `install_deps.sh`。

### 需注意的次要问题（非本次根因）
日志出现 `go: downloading go1.26.6`，与 Dockerfile 中 `ARG GOLANG_VERSION=1.24.2` 不一致（实际拉取了 1.26.6）。此为次要现象，不是本次构建终止的直接原因，但说明上游脚本可能在自行下载 Go，修复时应一并确认最终使用的 Go 版本是否符合预期。

## 需要进一步确认的点
- milvus v3.0.2 `scripts/install_deps.sh` 中发行版检测的具体判断逻辑（读取 `/etc/os-release` 的哪个字段、如何匹配），以确定是"补一个 openEuler 分支"还是"绕过检测"。
- 需要确认 openEuler 上是否有与脚本所安装依赖等价、可用的 yum/dnf 包名（如脚本在 Rocky/CentOS 分支安装的包在 openEuler 是否同名可用）。
- 失败发生在 x86_64 与 aarch64 哪个架构（日志中 `go: downloading go1.26.6 (linux/amd64)` 表明当前片段为 amd64 构建），需确认两架构是否同因此失败。

## 修复验证要求（仅当修复涉及正则 patch 外部源文件时填写）
若修复方向采用"用 sed/正则修改 milvus 上游 `scripts/install_deps.sh` 的发行版判断逻辑"，code-fixer 在提交前必须从上游仓库拉取 milvus `v3.0.2`（以 Dockerfile ARG `VERSION=3.0.2` 为准）的 `scripts/install_deps.sh` 实际内容，确认 `Unsupported Linux distribution` 判断所在行及格式，验证修改后的规则确实能让 openEuler 分支通过，再行提交。
