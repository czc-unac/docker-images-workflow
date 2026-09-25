# CI 失败分析报告

## 基本信息
- PR: #4492 — 【自动升级】ceph容器镜像升级至21.3.0版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式10（缺少构建依赖：CMake 找不到系统库）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#12 307.9 -- Found Python3: /usr/bin/python3.11 (found suitable version "3.11.6", minimum required is "3.6.0") found components: Interpreter
#12 308.0 -- Could NOT find Curses (missing: CURSES_LIBRARY CURSES_INCLUDE_PATH)
#12 308.1 CMake Error at /usr/share/cmake/Modules/FindPackageHandleStandardArgs.cmake:233 (message):
#12 308.1   Could NOT find Protobuf (missing: Protobuf_LIBRARIES Protobuf_INCLUDE_DIR)
#12 308.1 Call Stack (most recent call first):
#12 308.1   /usr/share/cmake/Modules/FindPackageHandleStandardArgs.cmake:603 (_FPHSA_FAILURE_MESSAGE)
#12 308.1   /usr/share/cmake/Modules/FindProtobuf.cmake:772 (FIND_PACKAGE_HANDLE_STANDARD_ARGS)
#12 308.1   src/CMakeLists.txt:1029 (find_package)
#12 308.1 -- Configuring incomplete, errors occurred!
#12 308.1 + exit 1
#12 ERROR: process "/bin/sh -c git clone -b v${VERSION} ... && ./do_cmake.sh ... && ninja ..." did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile:40-45`（ceph 构建 RUN 步骤），触发点位于 ceph 源码 `src/CMakeLists.txt:1029` 的 `find_package(Protobuf)`
- 失败原因: 新增 Dockerfile 的第一个 `dnf install` 依赖列表中**没有安装 protobuf 相关的开发包**（`protobuf-devel` / `protobuf-compiler`），导致 ceph 21.3.0 在 `./do_cmake.sh` 配置阶段找不到 Protobuf 的库与头文件，CMake 配置失败并以 exit code 1 终止。

### 与 PR 变更的关联
本 PR 新增 `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile`（新文件，54 行）并同步 README/image-info.yml/meta.yml。失败完全由该新 Dockerfile 的依赖清单缺项引起：该 Dockerfile 自建的 `dnf install` 列表覆盖了 thrift、boost、libnbd、rdma、lttng 等大量编译依赖，但遗漏了 ceph 构建必需的 Protobuf 开发包。属于本次 PR 改动**直接触发**的失败，与上游 ceph 21.3.0 代码本身无关。

### 补充说明（非根因，勿作为失败依据）
日志中另有以下非致命信息，均**不是**本次失败的根因：
- `-- Could NOT find Curses / gperftools / JeMalloc / xfs`：CMake 对这些可选组件缺失仅告警，配置继续。
- `bash: line 1: jq: command not found`：do_cmake.sh 内的非致命提示。
- BuildKit 警告 `UndefinedVar: Usage of undefined variable '$LD_LIBRARY_PATH' (line 47)`（对应模式20）与 `FromAsCasing` 大小写警告：仅为 Lint 警告，不影响构建结果。
- 末尾 `Finished: FAILURE` 表明为真实构建失败，日志与状态一致，无需按"证据不足"处理。

## 修复方向

### 方向 1（置信度: 高）
在 `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile` 的第一个 `dnf install` 列表中补充 protobuf 相关开发包，使 CMake 的 `find_package(Protobuf)` 能定位到库与头文件。openEuler/RPM 体系中通常需要安装 `protobuf-devel`（提供 `libprotobuf.so` 与 `google/protobuf/*.h`）以及 `protobuf-compiler`（提供 `protoc`，ceph 构建期需要生成代码）。可参照同场景下 ceph 20.3.0 同 OS 版本 Dockerfile 的依赖清单进行对齐。

### 方向 2（可选，置信度: 中）
若确认 ceph 21.3.0 的构建脚本对 protobuf 有其他版本/组件要求（如 `protobuf-lite-devel` 或需 `-DWITH_SYSTEM_PROTOBUF` 行为调整），需进一步核对上游 `do_cmake.sh`/`src/CMakeLists.txt:1029` 的 `find_package` 参数与 `WITH_*` 开关，判断是否应改为使用内置（submodule）protobuf 而非系统包。此方向需先获取实际构建脚本内容确认，不能直接假设。

## 需要进一步确认的点
1. 同仓库 `Storage/ceph/20.3.0/24.03-lts-sp4/Dockerfile` 是否安装了 protobuf 相关包；若安装了具体哪些包名，可直接对齐（受"禁止探索文件系统"约束，本次未读取）。
2. openEuler 24.03-LTS-SP4 仓库中 protobuf 包的确切名称与可用版本（`protobuf-devel`、`protobuf-compiler` 是否均存在且版本满足 ceph 21.3.0 要求）。
3. 是否存在除 Protobuf 之外被本次日志截断/掩盖的后续缺包（当前 `-- Configuring incomplete` 发生在 Protobuf 处，后续依赖尚未验证），修复 Protobuf 后需重新构建确认。

## 修复验证要求
不适用（修复方向为补充 `dnf install` 系统包，不涉及对第三方/上游源文件应用正则 patch）。
