# 修复摘要

## 修复的问题
修复 Ceph 21.3.0 镜像构建时因 CMake 配置阶段 `find_package(Protobuf REQUIRED)` 失败而中断的问题。

## 修改的文件
- `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile`: 在第 42 行 `do_cmake.sh` 调用中新增 `-DWITH_NVMEOF_GATEWAY_MONITOR_CLIENT=OFF`。

## 修复逻辑
分析报告中直接错误为 `Could NOT find Protobuf`，其来源是 Ceph 21.3.0 的 `src/CMakeLists.txt:1029`。经对比上游源码确认了真正的根因：

- 已从上游 `https://raw.githubusercontent.com/ceph/ceph/v20.3.0/src/CMakeLists.txt` 与 `v21.3.0/src/CMakeLists.txt` 获取并对比两个版本。20.3.0 中 `WITH_NVMEOF_GATEWAY_MONITOR_CLIENT` 有平台门槛：
  ```cmake
  if(EXISTS "/etc/redhat-release" OR EXISTS "/etc/fedora-release")
    option(WITH_NVMEOF_GATEWAY_MONITOR_CLIENT ... ON)
  else()
    option(WITH_NVMEOF_GATEWAY_MONITOR_CLIENT ... OFF)
  endif()
  ```
  openEuler 没有 `/etc/redhat-release`/`/etc/fedora-release`，因此该特性在 20.3.0 上默认 **OFF**，不需要 Protobuf/gRPC，所以既有 20.3.0 Dockerfile 才能构建成功。
- 21.3.0 删除了该平台判断，改为无条件 `option(WITH_NVMEOF_GATEWAY_MONITOR_CLIENT ... ON)`，导致 openEuler 上也开启该特性，进而在 `find_package(Protobuf REQUIRED)` 处报错。

因此修复方向 1（仅补 `protobuf-devel`）并不充分：同一特性块在 Protobuf 之后还 `find_package(gRPC ...)`，并在找不到 gRPC CMake config 时走 `pkg_check_modules(GRPCPP REQUIRED ... grpc++)`（openEuler 的 `grpc-devel` 只提供 pkgconfig、不提供 `gRPCConfig.cmake`），且 `grpc++.pc` 的 `Requires` 依赖大量 `absl_*` pkgconfig 模块。仅补 Protobuf 会在下一行继续失败。

本修复通过禁用这个 21.3.0 新增、且在上游本意上仅面向 RPM/RHEL 平台的可选组件，使 21.3.0 的必需依赖集合与已验证可构建的 20.3.0 保持一致，从而消除 Protobuf（及后续 gRPC）缺失导致的构建失败。

### 验证
1. 已从上游 `v21.3.0` 获取 `src/CMakeLists.txt`，确认 `src/CMakeLists.txt:1029` 为 `find_package(Protobuf REQUIRED)`，位于 `if(WITH_NVMEOF_GATEWAY_MONITOR_CLIENT)` 内。
2. 已从上游 `v20.3.0` 获取同文件，确认平台门槛存在且默认 OFF；两版本除该特性块外 `find_package(... REQUIRED)` 集合一致（thrift/fmt/pmdk/ndctl/daxctl/Lua/GTest/GMock/Arrow/Parquet/utf8proc），故禁用后恢复 20.3.0 的配置可行性。
3. 已获取上游 `v21.3.0/do_cmake.sh`，确认 `${CMAKE} $ARGS "$@" ...` 会透传命令行 `-D` 参数，新增的 `-DWITH_NVMEOF_GATEWAY_MONITOR_CLIENT=OFF` 会生效。
4. 备用方案核查：已确认 openEuler 24.03-LTS-SP4 的 `protobuf-devel`/`protobuf-compiler`/`grpc-devel`/`grpc-plugins` 在 x86_64 与 aarch64 均存在，但完整依赖链（含 `absl_*` pkgconfig、`re2`/`c-ares` 等）复杂且引入较大体积与版本兼容风险，故未采用。

## 潜在风险
- 该修复关闭了 21.3.0 新默认开启的 `nvmeof gateway monitor client` 组件，构建产物不包含该可选客户端。这与既有 20.3.0 镜像以及上游对非 RPM 平台（含 openEuler）的原始行为一致，不影响 ceph 核心/存储功能；如后续确需该组件，应改为在 dnf 清单中补齐 `protobuf-devel protobuf-compiler grpc-devel grpc-plugins abseil-cpp-devel` 等完整构建依赖并重新验证。
- 分析报告提到的 `jq: command not found`、`Could NOT find Curses` 均为非致命信息，不是本次失败根因，按最小化原则未改动。
- 不确定 21.3.0 在禁用该特性后是否还有其他新增的强制依赖；本次对比显示 `REQUIRED` 依赖集合与已验证的 20.3.0 一致，风险低。