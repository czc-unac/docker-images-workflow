# 修复摘要

## 修复的问题
在 ceph 21.3.0 的构建镜像中补充 Protobuf/gRPC 构建依赖，修复 `do_cmake.sh` 配置阶段 `find_package(Protobuf)` 失败导致的构建错误。

## 修改的文件
- `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile`: 在 `dnf install` 依赖列表中新增 `protobuf-devel protobuf-compiler protobuf-lite-devel` 与 `grpc-devel grpc-plugins`。

## 修复逻辑
- 失败根因: 上游 ceph v21.3.0 的 `src/CMakeLists.txt:1021` 将 `WITH_NVMEOF_GATEWAY_MONITOR_CLIENT` 改为**无条件默认 ON**（v20.3.0 时该选项仅在 `/etc/redhat-release` / `/etc/fedora-release` 存在时才 ON，因此 openEuler 上此前不会触发）。该分支在 `src/CMakeLists.txt:1029` 执行 `find_package(Protobuf REQUIRED)`，其后的 gRPC 查找（`find_package(gRPC CONFIG QUIET)` → pkg-config `grpc++` REQUIRED）也依赖 gRPC 开发包。原 Dockerfile 依赖清单缺少这些包，CMake 配置在 `find_package(Protobuf)` 处报 `missing: Protobuf_LIBRARIES Protobuf_INCLUDE_DIR` 并以 exit 1 终止。
- 修复方式: 按分析报告的**方向 1**，在 `dnf install` 中补齐 protobuf 与 gRPC 开发依赖：
  - `protobuf-devel` / `protobuf-compiler` / `protobuf-lite-devel`：提供 `protobuf-config.cmake`、`libprotobuf`、头文件与 `protoc`。
  - `grpc-devel` / `grpc-plugins`：提供 gRPC 开发库、`grpc++` pkgconfig 及 nvmeof 客户端生成 proto 代码所需的 `grpc_cpp_plugin`。
- 包名已从上游**实际源文件**验证：
  - 从 `https://raw.githubusercontent.com/ceph/ceph/v21.3.0/src/CMakeLists.txt` 获取上游文件，确认第 1021/1029 行确为无条件 `option(WITH_NVMEOF_GATEWAY_MONITOR_CLIENT ... ON)` 与 `find_package(Protobuf REQUIRED)`（对比 v20.3.0 的 redhat/fedora 条件判断）。
  - 从 openEuler 24.03-LTS-SP4 `everything/x86_64` 仓库 primary 元数据确认以下包确实存在：`protobuf-devel`(25.1-13.oe2403sp4)、`protobuf-compiler`(25.1-13.oe2403sp4)、`protobuf-lite-devel`(25.1-13.oe2403sp4)、`grpc-devel`(1.60.0-5.oe2403sp4)、`grpc-plugins`(1.60.0-5.oe2403sp4)，包名可被 `dnf install` 正确解析。

## 潜在风险
- 构建将新增编译 `ceph-nvmeof-monitor-client` 组件，涉及 openEuler 的 protobuf 25.1 / gRPC 1.60 / abseil，若上游代码与这些版本存在编译不兼容，可能出现后续编译错误；但这是补齐上游要求的依赖、保持镜像功能完整的正确做法。
- 改动仅新增构建期依赖，不影响运行时镜像内容与 entrypoint 行为。