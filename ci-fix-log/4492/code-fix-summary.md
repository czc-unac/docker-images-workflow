# 修复摘要

## 修复的问题
补充 ceph 21.3.0 构建镜像时缺失的 Protobuf 与 gRPC 开发/编译依赖，解决 `./do_cmake.sh` 配置阶段 `find_package(Protobuf)` 失败导致的 build-error。

## 修改的文件
- `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile`: 在第 20 行后的 `dnf install` 依赖列表中新增一行 `protobuf-devel protobuf-compiler protobuf-lite-devel grpc-devel grpc-plugins`。

## 修复逻辑
1. 已从上游 ceph v21.3.0 获取 `src/CMakeLists.txt` 验证：ceph 21.3.0 新增了默认开启的 `WITH_NVMEOF_GATEWAY_MONITOR_CLIENT` 组件（`option(... ON)`），该代码块内包含 `find_package(Protobuf REQUIRED)`，这正是 CI 日志中报错的位置 `src/CMakeLists.txt:1029`。
2. 同一代码块紧接着还要求 gRPC：先 `find_package(gRPC CONFIG QUIET)`，失败则 `pkg_check_modules(GRPCPP REQUIRED IMPORTED_TARGET grpc++)`，并需要 `grpc_cpp_plugin`（`grpc-plugins` 提供）以及链接 `grpc++_reflection`。日志中 CMake 在 Protobuf 处以 FATAL_ERROR 提前终止，因此未打印 gRPC 错误，但 gRPC 属于同一根因、必然的下一步缺包。
3. 原 Dockerfile 的 `dnf install` 清单遗漏了 protobuf/gRPC 开发包，导致 CMake 配置失败。补充上述系统包即可让 `find_package` 定位到库、头文件与 `protoc`。
4. 包名可用性依据 src-openeuler/grpc 的 `grpc.spec`（grpc 1.60.0，`-DgRPC_PROTOBUF_PROVIDER=package`）确认：其子包为 `grpc-devel`（提供 `grpc++` 头文件、pkgconfig、gRPCConfig.cmake）、`grpc-plugins`（提供 `grpc_*_plugin`），并把 `abseil-cpp-devel`、`re2`、`protobuf-devel`、`protobuf-compiler`、`protobuf-lite-devel` 作为依赖。openEuler 仓库中存在这些包。
5. 该改动与本次 PR 新增的 `21.3.0/24.03-lts-sp4/Dockerfile` 直接相关，未触碰其他无关文件。

## 潜在风险
- 若 openEuler 24.03-LTS-SP4 的 grpc 1.60 的 CMake/pkg-config 导出与 ceph 21.3.0 期望不一致（例如 `gRPC::grpc_cpp_plugin` 目标未导出），构建可能仍会在 NVMe-oF monitor client 处失败。届时可改为在 `do_cmake.sh` 参数中追加 `-DWITH_NVMEOF_GATEWAY_MONITOR_CLIENT=OFF` 关闭该可选组件（与 20.3.0 镜像行为一致），但本次未采用，以保留默认功能并遵循分析报告"补充依赖"的方向。
- 修复 Protobuf/gRPC 后，日志中被提前终止掩盖的后续可选依赖缺失可能暴露，需重新构建确认。