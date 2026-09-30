# CI 失败分析报告

## 基本信息
- PR: #4723 — 【自动升级】milvus容器镜像升级至3.0.2版本.
- 失败类型: `build-error`
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: 上游脚本不支持发行版
- 新模式症状关键词: Unsupported Linux distribution, openEuler, install_deps.sh, milvus, exit code: 1

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
------
Dockerfile:22
  22 | >>> RUN git clone -b v${VERSION} https://github.com/milvus-io/milvus.git && \
  23 | >>>     cd milvus/ && \
  24 | >>>     ./scripts/install_deps.sh && \
  25 | >>>     CXXFLAGS="-I/usr/include/openblas" make build-cpp && \
  26 | >>>     make build-go
ERROR: failed to solve: process "... install_deps.sh ..." did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile:22-26`（`RUN git clone … && ./scripts/install_deps.sh && make build-cpp && make build-go` 步骤）
- 失败原因: milvus v3.0.2 仓库自带的 `scripts/install_deps.sh` 通过发行版探测仅识别 Ubuntu、Rocky Linux、Amazon Linux、CentOS，遇到 openEuler 时直接打印 `[ERROR] Unsupported Linux distribution: openEuler` 并 `exit 1`，导致其后的 `make build-cpp` / `make build-go` 根本没有执行。

### 与 PR 变更的关联
本 PR 新增了 `Database/milvus/3.0.2/24.03-lts-sp4/Dockerfile`（并在 README.md、image-info.yml、meta.yml 中登记）。该 Dockerfile 直接调用上游 milvus 的 `install_deps.sh` 安装编译依赖，而上游脚本不支持 openEuler，因此是本次 PR 新增文件直接引入的构建失败，与其它改动无关。

补充观察（非根因，但显示脚本行为差异）：Dockerfile 中虽设定 `ARG GOLANG_VERSION=1.24.2`，但 `install_deps.sh` 自行下载了 `go1.26.6` 并打印 `Go version: 1.26`，说明上游脚本内部自带 Go 版本管理，与 Dockerfile 的 GOLANG_VERSION 参数不一致；构建中断点是发行版检查，发生在此之后。

## 修复方向

### 方向 1（置信度: 高）
绕过/绕过上游 `install_deps.sh` 的发行版限制：不再依赖该脚本进行环境探测，改为在 Dockerfile 中直接按 openEuler 方式安装 milvus 编译所需的依赖（或让发行版探测识别为受支持发行版，使其跳过 unsupported 分支），随后继续执行 `make build-cpp` / `make build-go`。

### 方向 2（置信度: 中）
在 Dockerfile 的 `git clone` 之后、执行 `install_deps.sh` 之前，对克隆下来的上游脚本做适配修补（如增加 openEuler 分支 / 调整发行版判断逻辑），再继续原有构建流程。

### 方向 3（置信度: 低）
参考仓库中已有的 milvus 旧版本 Dockerfile（如 `Database/milvus/2.5.14/24.03-lts-sp2`、`2.6.0/24.03-lts-sp2`）的依赖安装方式，确认是否存在可复用的、不依赖 `install_deps.sh` 发行版探测的构建路径。

## 需要进一步确认的点
- 上游 milvus v3.0.2 的 `scripts/install_deps.sh` 中发行版探测的具体实现（探测依据是 `/etc/os-release` 的 `ID`、`lsb_release` 还是其它字段），以确定采用方向 1 还是方向 2。
- openEuler 上 milvus 3.0.2 编译实际需要的系统包清单（对照脚本中 Ubuntu/CentOS 分支安装的包），确保绕过后依赖完整。
- 仓库内既有 milvus 版本 Dockerfile 是否已经处理过同类 openEuler 兼容问题，可直接沿用其做法。
- 运行时阶段（`FROM $BASE` 后的 etcd/minio 下载与 `COPY --from=builder` 产物路径）本次尚未执行到，修复构建阶段后需一并验证。

## 修复验证要求
若修复方向中包含"对上游 milvus 源码/脚本文件做正则或文本替换"（如修改克隆后的 `scripts/install_deps.sh`），code-fixer 在提交前必须：从 milvus v3.0.2（以 Dockerfile `ARG VERSION=3.0.2` 为准）获取 `scripts/install_deps.sh` 的实际内容，确认发行版探测分支的真实写法，验证替换规则能精确匹配后再提交，不得假设脚本结构一定与预期一致。
