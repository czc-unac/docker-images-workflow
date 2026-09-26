# CI 失败分析报告

## 基本信息
- PR: #4576 — 【自动升级】ceph容器镜像升级至21.3.0版本.
- 失败类型: build-error
- 置信度: 高
- 知识库匹配: 模式10（缺少构建依赖 CMake 找不到系统库）
- 新模式标题: (不适用)
- 新模式症状关键词: (不适用)

## 根因分析

### 直接错误
```
#12 231.6 CMake Error at /usr/share/cmake/Modules/FindPackageHandleStandardArgs.cmake:233 (message):
#12 231.6   Could NOT find Protobuf (missing: Protobuf_LIBRARIES Protobuf_INCLUDE_DIR)
#12 231.6 Call Stack (most recent call first):
#12 231.6   /usr/share/cmake/Modules/FindPackageHandleStandardArgs.cmake:603 (_FPHSA_FAILURE_MESSAGE)
#12 231.6   /usr/share/cmake/Modules/FindProtobuf.cmake:772 (FIND_PACKAGE_HANDLE_STANDARD_ARGS)
#12 231.6   src/CMakeLists.txt:1029 (find_package)
#12 231.6
#12 231.6
#12 231.6 -- Configuring incomplete, errors occurred!
#12 231.6 + exit 1
#12 ERROR: process "/bin/sh -c git clone -b v${VERSION} ... && ./do_cmake.sh ... && ninja ..." did not complete successfully: exit code: 1
```

日志末尾为 `Finished: FAILURE`，与 PR 的失败状态一致（非 trigger 层伪成功场景）。

### 根因定位
- 失败位置: `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile:40-45`（`./do_cmake.sh` 配置阶段），实际报错点在上游 `src/CMakeLists.txt:1029` 的 `find_package(Protobuf)`
- 失败原因: 构建镜像的 `dnf install` 依赖清单中未包含 Protobuf 开发包（`protobuf-devel` / `protobuf-compiler`），CMake 配置阶段在 `src/CMakeLists.txt:1029` 调用 `find_package(Protobuf)` 时找不到 `Protobuf_LIBRARIES` 与 `Protobuf_INCLUDE_DIR`，导致 `Configuring incomplete`，`do_cmake.sh` 以 exit 1 终止。

### 与 PR 变更的关联
本次 PR 新增了 `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile`（全新文件，54 行），其 `dnf install` 列表新增/沿用了 ceph 21.3.0 的构建依赖。Ceph 21.x 相比 20.x 在上游 CMake 配置中新增了对 Protobuf 的强依赖（`src/CMakeLists.txt:1029`），而该 Dockerfile 的依赖列表缺少 protobuf 相关包，因此首次引入该版本 Dockerfile 时直接触发配置失败。失败由 PR 改动直接引起。

## 修复方向

### 方向 1（置信度: 高）
在 `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile` 的 `dnf install` 依赖列表中补充 Protobuf 开发依赖（openEuler 包名通常为 `protobuf-devel`、`protobuf-compiler`、`protobuf-c` 等），使 `find_package(Protobuf)` 能找到库与头文件，然后重新触发构建。

### 方向 2（可选，置信度: 中）
若 Ceph 该版本允许在不需要相关组件时关闭 Protobuf，可在 `./do_cmake.sh` 的 CMake 参数中显式关闭依赖 Protobuf 的组件（如 `-DWITH_RADOSGW_*` 或对应开关）；但需确认 ceph 21.3.0 是否提供可关闭该依赖的选项，否则仍应回到方向 1 补齐依赖。注意：直接禁用可能导致构建出的镜像功能缺失，应优先选择方向 1。

## 需要进一步确认的点
- 确认 openEuler 24.03-LTS-SP4 仓库中 Protobuf 相关包的确切包名（`protobuf-devel`、`protobuf-compiler`、`protobuf-c-devel` 等），避免包名不存在导致新的 `dnf install` 失败。
- 日志中另有非致命项：`Could NOT find Curses`、`Could NOT find gperftools`、`Could NOT find JeMalloc`、`Could NOT find xfs`、`bash: line 1: jq: command not found`。这些在 ceph 构建中通常为可选依赖（tcmalloc/jemalloc 缺失时已回退 libc，属 CMake Warning），**不是本次失败根因**，但可一并评估是否需要补齐以减少告警。

## 修复验证要求
本次修复涉及在 Dockerfile 依赖列表补充系统包名，建议 code-fixer 在提交前确认 openEuler 24.03-LTS-SP4 仓库中对应 Protobuf 包的真实存在与命名（可通过仓库索引/已构建镜像中 `dnf list` 验证），确保补充的包名可被 `dnf install` 正确解析；如选择方向 2（关闭组件），需先确认 ceph 21.3.0 上游 CMake 提供对应开关，不能假设一定存在。
