# CI 失败分析报告

## 基本信息
- PR: #4732 — 【自动升级】ceph容器镜像升级至21.3.0版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式10（缺少构建依赖 CMake 找不到系统库）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 前置检查（日志与状态一致性）
`ci.logs` 末尾为 `Build step 'Execute shell' marked build as failure` / `Finished: FAILURE`，未出现 `Finished: SUCCESS` 或 `Build successful`，日志与 PR 的 CI 失败状态一致，可继续分析。失败发生在 Docker 构建阶段（镜像构建 job），属于提供的日志范围内。

## 根因分析

### 直接错误
```
#12 314.7 CMake Error at /usr/share/cmake/Modules/FindPackageHandleStandardArgs.cmake:233 (message):
#12 314.7   Could NOT find Protobuf (missing: Protobuf_LIBRARIES Protobuf_INCLUDE_DIR)
#12 314.7 Call Stack (most recent call first):
#12 314.7   /usr/share/cmake/Modules/FindPackageHandleStandardArgs.cmake:603 (_FPHSA_FAILURE_MESSAGE)
#12 314.7   /usr/share/cmake/Modules/FindProtobuf.cmake:772 (FIND_PACKAGE_HANDLE_STANDARD_ARGS)
#12 314.7   src/CMakeLists.txt:1029 (find_package)
#12 314.8 -- Configuring incomplete, errors occurred!
#12 314.8 + exit 1
#12 ERROR: process "/bin/sh -c git clone -b v${VERSION} --recursive --depth 1 https://github.com/ceph/ceph.git ... && ./do_cmake.sh ... && ninja -j$(nproc) && ninja install" did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile:40`（ceph 源码构建 RUN 步骤内的 cmake 配置阶段 `src/CMakeLists.txt:1029`）
- 失败原因: 构建环境缺少 Protobuf 开发库/头文件，Ceph 的 CMake 在 `find_package(Protobuf)` 处报 `Could NOT find Protobuf (missing: Protobuf_LIBRARIES Protobuf_INCLUDE_DIR)`，配置阶段直接中止（exit code 1）。

### 与 PR 变更的关联
强关联。本 PR 新增 `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile`，其第一个 `dnf install` 的软件包清单（diff 第 5-20 行）未包含 `protobuf` 相关的 `-devel`/编译包。Ceph 21.3.0 的 CMake 在配置阶段强制要求 Protobuf（`src/CMakeLists.txt:1029` 调用了 `find_package(Protobuf)` 且为必需项），因此该 Dockerfile 首次构建即在该步骤失败。这是新增文件本身遗漏构建依赖所致，与代码改动直接相关。

## 修复方向

### 方向 1（置信度: 高）
在 Dockerfile 的 `dnf install` 清单中补充 Protobuf 开发依赖，使 CMake 能定位到 `Protobuf_LIBRARIES` 与 `Protobuf_INCLUDE_DIR`。openEuler 中对应的包为 `protobuf-devel`（同时建议提供 `protobuf-compiler` 以满足 `protoc` 需求）。参照仓库内其他 ceph 版本目录的 Dockerfile 依赖清单进行对齐确认。

### 方向 2（可选，置信度: 低）
日志中同时出现 `bash: line 1: jq: command not found` 及 `Could NOT find Curses (missing: CURSES_LIBRARY CURSES_INCLUDE_PATH)`。这两条在本次日志中均非致命（构建在 Protobuf 处才报 Error 中止），但如需在 ceph 构建中启用相关功能，可一并确认是否需补充 `jq`、`ncurses-devel`。当前不应将其作为根因处理。

> 注：`CMake Warning`（tcmalloc/jemalloc not found）、`Performing Test ... - Failed`（如 HAVE_CXX11_ATOMIC、HAVE_IBV_EXP）均为探测类非致命信息，不是本次失败根因。

## 需要进一步确认的点
1. 对照仓库内既有的 ceph 20.3.0（`Storage/ceph/20.3.0/24.03-lts-sp4/Dockerfile`）依赖清单，确认其是否已包含 `protobuf-devel`/`protobuf-compiler`，以对齐新版本依赖，避免遗漏同类构包。
2. 确认 Ceph 21.3.0 是否还引入了其他新增的强制 `find_package` 依赖（本次日志在 Protobuf 处即中止，后续是否还有别的缺失依赖无法从当前日志判断）。
3. 日志中 `Could NOT find Curses`、`jq: command not found` 是否需要一并补齐，取决于构建配置是否需要启用对应特性（非本次失败根因）。
4. 本次日志仅包含单一架构（x86_64）构建 job；aarch64 构建 job 是否同样失败及是否还有其他架构相关问题，无法从当前日志判断。
