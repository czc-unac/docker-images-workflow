# CI 失败分析报告

## 基本信息
- PR: #4723 — 【自动升级】milvus容器镜像升级至3.0.2版本.
- 失败类型: `build-error`
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 发行版不支持
- 新模式症状关键词: Unsupported Linux distribution, openEuler, install_deps.sh, exit code: 1

## 根因分析

### 直接错误
```
#13 32.01 [INFO] Milvus Development Dependencies Installer
#13 32.01 [INFO] ==========================================
#13 32.04 go: downloading go1.26.6 (linux/amd64)
#13 40.68 [INFO] Go version: 1.26
#13 40.68 [ERROR] Unsupported Linux distribution: openEuler
#13 40.68 [INFO] Supported distributions: Ubuntu, Rocky Linux, Amazon Linux, CentOS
#13 ERROR: process "/bin/sh -c git clone -b v${VERSION} https://github.com/milvus-io/milvus.git &&     cd milvus/ &&     ./scripts/install_deps.sh &&     CXXFLAGS=\"-I/usr/include/openblas\" make build-cpp &&     make build-go" did not complete successfully: exit code: 1

Dockerfile:22
  22 | >>> RUN git clone -b v${VERSION} https://github.com/milvus-io/milvus.git && \
  23 | >>>     cd milvus/ && \
  24 | >>>     ./scripts/install_deps.sh && \
  25 | >>>     CXXFLAGS="-I/usr/include/openblas" make build-cpp && \
  26 | >>>     make build-go
ERROR: failed to solve: process ... did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile:22-26`（`RUN ... ./scripts/install_deps.sh ...` 步骤）
- 失败原因: milvus v3.0.2 上游的 `scripts/install_deps.sh` 对操作系统做白名单检测（仅支持 Ubuntu、Rocky Linux、Amazon Linux、CentOS），无法识别 openEuler，脚本主动报错并以 exit code 1 退出，导致后续 `make build-cpp`、`make build-go` 均未执行。

### 与 PR 变更的关联
直接相关。本 PR 新增 `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`（全新文件），其构建路径强制在 `openeuler/openeuler:24.03-lts-sp4` 基础镜像上运行 milvus 官方 `install_deps.sh`。该脚本不支持 openEuler，因此失败由本 PR 引入的构建步骤触发，属必现问题，并非历史遗留或基础设施问题。

补充观察（非根因）：日志显示 `go: downloading go1.26.6`、`Go version: 1.26`，与 Dockerfile 中 `GOLANG_VERSION=1.24.2` 不一致，说明 `install_deps.sh` 会自行安装其绑定的 Go 版本；此现象不构成失败原因，仅提示脚本行为与 Dockerfile 预期存在偏差。

## 修复方向

### 方向 1（置信度: 高）
绕过/兼容 milvus `install_deps.sh` 的发行版白名单检测。可在 Dockerfile 中于执行 `./scripts/install_deps.sh` 之前，对该脚本的发行版检测逻辑做适配，使 openEuler 被识别为受支持的同类发行版（如 CentOS/Rocky 分支），或直接将所需开发依赖改由官方 openEuler 仓库手动安装（对应 `dnf install`/`yum install`），从而不再依赖脚本的系统检测分支。

### 方向 2（置信度: 中）
放弃调用 `install_deps.sh`，改为在 Dockerfile 中显式安装 milvus v3.0.2 编译所需的依赖（gcc/g++/cmake/conan/rust/openblas-devel 等已在部分步骤安装），只保留 `make build-cpp && make build-go`。需逐一核对 v3.0.2 的实际依赖清单，避免遗漏。

## 需要进一步确认的点
- milvus v3.0.2（tag v3.0.2，commit 3c4448a2aee506ccd14851e742aba85f1c83188a）`scripts/install_deps.sh` 中发行版检测的具体实现（检测依据是 `/etc/os-release` 的 `ID`、`ID_LIKE` 还是别处），以确定最小且正确的适配方式。
- 该脚本在通过发行版检测后，实际会安装哪些依赖、是否仍会出现在 openEuler 上不可用的包名（参照知识库模式10）。
- 脚本自行下载 Go 1.26 的行为是否会影响构建产物，是否需要在通过检测后另行固定 Go 版本。
- 在 `24.03-lts-sp4` 上 `make build-cpp`/`make build-go` 是否还有后续（当前日志因脚本提前退出而未暴露）其他兼容性问题。

## 修复验证要求
若修复方向采用"patch 上游 `scripts/install_deps.sh` 中的发行版检测逻辑/正则"：
- code-fixer 必须从 milvus 上游对应 tag（v3.0.2）拉取实际的 `scripts/install_deps.sh`，确认待修改的发行版判断代码真实存在且格式如预期，验证适配逻辑确实能让 openEuler 走到受支持分支后，再提交修改。
- 若采用手动安装依赖方案，须对照 v3.0.2 该脚本中的依赖清单逐项确认，不能假设当前 Dockerfile 已安装的包完整。
