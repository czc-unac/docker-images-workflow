# CI 失败分析报告

## 基本信息
- PR: #4723 — 【自动升级】milvus容器镜像升级至3.0.2版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: milvus 脚本不支持 openEuler
- 新模式症状关键词: Unsupported Linux distribution, openEuler, install_deps.sh, exit code: 1

## 根因分析

### 直接错误
```
#13 [builder 4/4] RUN git clone -b v3.0.2 https://github.com/milvus-io/milvus.git && cd milvus/ && ./scripts/install_deps.sh && CXXFLAGS="-I/usr/include/openblas" make build-cpp && make build-go
#13 32.01 [INFO] Milvus Development Dependencies Installer
#13 32.01 [INFO] ==========================================
#13 32.04 go: downloading go1.26.6 (linux/amd64)
#13 40.68 [INFO] Go version: 1.26
#13 40.68 [ERROR] Unsupported Linux distribution: openEuler
#13 40.68 [INFO] Supported distributions: Ubuntu, Rocky Linux, Amazon Linux, CentOS
#13 ERROR: process "/bin/sh -c git clone -b v${VERSION} ... ./scripts/install_deps.sh ..." did not complete successfully: exit code: 1
ERROR: failed to solve: process "/bin/sh -c git clone ..." did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile:22-26`（`RUN ... ./scripts/install_deps.sh ...` 步骤）；实际报错来自上游 milvus v3.0.2 仓库的 `scripts/install_deps.sh`
- 失败原因: milvus v3.0.2 的依赖安装脚本 `install_deps.sh` 内做发行版检测，仅识别 Ubuntu / Rocky Linux / Amazon Linux / CentOS，检测到 openEuler 时直接打印 `Unsupported Linux distribution: openEuler` 并以 exit code 1 退出，导致 `make build-cpp` / `make build-go` 从未执行。

### 与 PR 变更的关联
PR 新增了 `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`（全新增，53 行），构建流程为 `git clone -b v3.0.2 milvus` 后直接调用上游 `scripts/install_deps.sh`。该脚本不支持 openEuler 发行版，故此次失败由本 PR 新增的 Dockerfile 直接触发。旧版本（2.5.14/2.6.0）Dockerfile 若采用相同方式调用 install_deps.sh，也可能存在同类问题，但不在本 PR 变更范围内。

## 修复方向

### 方向 1（置信度: 高）
在 Dockerfile 中绕过或适配 `install_deps.sh` 的发行版检测：openEuler 24.03-LTS-SP4 属 RHEL/CentOS 系兼容发行版，可让脚本把 openEuler 识别为 CentOS/Rocky，或直接跳过该检测逻辑，再继续 `make build-cpp` / `make build-go`。需确认篡改发行版标识后脚本使用的包名（dnf/yum 包）在 openEuler 仓库中均存在。

### 方向 2（置信度: 中）
不使用上游 `install_deps.sh`，改为在 Dockerfile 中显式安装 milvus 3.0.2 所需的编译依赖（依据脚本中对应 CentOS/Rocky 分支的包清单），再执行 `make build-cpp` 与 `make build-go`。

## 需要进一步确认的点
1. milvus v3.0.2 上游 `scripts/install_deps.sh` 的发行版检测逻辑（如何判定 distro，是否可通过环境变量或 `ID` 覆盖），以及它对 CentOS/Rocky 分支安装的具体包清单。
2. openEuler 24.03-LTS-SP4 的 dnf/yum 源是否提供该包清单中的全部依赖（如 conan/cmake/ninja 等所需包），是否存在包名差异导致后续步骤仍失败。
3. 下游架构 job（x86-64、aarch64）是否同样报错，确认该失败为两架构共有还是仅单架构。

## 修复验证要求（仅当修复涉及正则 patch 外部源文件时填写）
若修复方向选择在 Dockerfile 中通过 sed/python 正则 patch 上游 milvus 仓库的 `scripts/install_deps.sh`（或其它源文件），code-fixer 在提交前必须：
- 从 milvus `v${VERSION}`（即 v3.0.2）实际源码中拉取 `scripts/install_deps.sh`，核对目标发行版检测代码段的真实文本与语法（如 `lsb_release` / `/etc/os-release` 的 `ID` 判定分支），验证新正则确实能匹配并生效；
- 同步验证 patch 后脚本在 openEuler 上走通的安装分支所引用的包在 openEuler 源中可用。
