# 修复摘要

## 修复的问题
为 ceph 21.3.0 镜像的构建阶段补充 protobuf / gRPC 开发依赖，修复 `do_cmake.sh` 配置阶段
`Could NOT find Protobuf` 导致 x86_64 与 aarch64 构建均失败的问题。

## 修改的文件
- `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile`: 在首个 `dnf install` 列表中新增
  `protobuf-devel protobuf-compiler grpc-devel grpc-plugins` 四个构建依赖包。

## 修复逻辑

### 实际根因（已获取真实 CI 日志确认）
分析报告中日志缺失，因此先按流程要求拉取了下游失败 job 的完整日志：

- x86_64 job：`https://log-ci.openeuler.openatom.cn/job/multiarch/openeuler/x86-64/openeuler-docker-images/5036/`
- aarch64 job：`https://log-ci.openeuler.openatom.cn/job/multiarch/openeuler/aarch64/openeuler-docker-images/5132/`

（PR 评论中的门禁结果：`check_sca` SUCCESS；`check_build` x86_64 / aarch64 均 FAILED；
`check_package_license` 仅为 WARNING（缺少项目级 Copyright 声明文件），非失败项。）

两个架构的日志在 `Dockerfile:40` 处报同一错误：

```
#12 225.8 CMake Error at /usr/share/cmake/Modules/FindPackageHandleStandardArgs.cmake:233 (message):
#12 225.8   Could NOT find Protobuf (missing: Protobuf_LIBRARIES Protobuf_INCLUDE_DIR)
#12 225.8 Call Stack (most recent call first):
#12 225.8   src/CMakeLists.txt:1029 (find_package)
```

### 为什么 20.3.0 能过、21.3.0 不能
对比上游 `ceph/ceph` 源码（已从 GitHub 拉取 `v20.3.0` 与 `v21.3.0` 的 `src/CMakeLists.txt` 验证）：

- `v20.3.0`：NVMeOF 网关监控客户端受平台条件保护，仅在 RHEL/Fedora 上默认开启：
  ```cmake
  if(EXISTS "/etc/redhat-release" OR EXISTS "/etc/fedora-release")
    option(WITH_NVMEOF_GATEWAY_MONITOR_CLIENT "build nvmeof gateway monitor client" ON)
  else()
    option(WITH_NVMEOF_GATEWAY_MONITOR_CLIENT "build nvmeof gateway monitor client" OFF)
  endif()
  ```
  在 openEuler 上该组件被关闭，因此不需要 Protobuf / gRPC。
- `v21.3.0`：改为**无条件默认开启**：
  ```cmake
  option(WITH_NVMEOF_GATEWAY_MONITOR_CLIENT "build nvmeof gateway monitor client" ON)
  if(WITH_NVMEOF_GATEWAY_MONITOR_CLIENT)
    find_package(Protobuf REQUIRED)   # v21.3.0 src/CMakeLists.txt:1029
    ...
    pkg_check_modules(GRPCPP REQUIRED IMPORTED_TARGET grpc++)
    find_program(_GRPC_CPP_PLUGIN_EXECUTABLE grpc_cpp_plugin REQUIRED)
  ```

原 Dockerfile 的 `dnf install` 列表缺失 protobuf 与 gRPC 的 `-devel` 包，故 21.3.0 配置阶段直接报错。
上游 `ceph.spec.in`（v21.3.0）的 BuildRequires 也明确要求：
`grpc-devel`、`protobuf-devel`、`protobuf-compiler`。

### 依赖可用性验证
已确认 openEuler 24.03-LTS-SP4 仓库提供所需包（解析其 `primary.xml` 确认）：
- `protobuf-devel`（提供 `cmake(protobuf)`、`pkgconfig(protobuf)`）
- `protobuf-compiler`（提供 `/usr/bin/protoc`）
- `grpc-devel`（提供 `pkgconfig(grpc++)`）
- `grpc-plugins`（提供 `/usr/bin/grpc_cpp_plugin`）

### 版本 tag 校验
分析报告的方向 2（`v21.3.0` 是否为上游真实 tag）经 `git ls-remote` 核实：`ceph/ceph`
上游**存在** `v21.3.0` tag，构建日志也已成功 `git clone -b v21.3.0`（仅提示该 tag 为 annotated tag），
故不属于「版本不存在」问题，未改动版本号。

> 说明：本修复直接编辑仓库内 Dockerfile，未涉及对上游源文件的正则 patch，因此不适用正则 patch 验证条款。

## 潜在风险
- 新增依赖会拉入 `abseil-cpp-devel` 等传递依赖，略微增大构建体积与时间；不影响现有功能。
- 日志中另有两条非致命告警：`bash: line 1: jq: command not found`（来自 ceph
  `src/pybind/mgr/dashboard/frontend/CMakeLists.txt` 的 `execute_process`，其输出变量
  `default_lang` 未被后续使用，属无害告警）与 BuildKit `UndefinedVar: $LD_LIBRARY_PATH`
  （模式20，仅告警）。二者均非本次失败根因，按最小化原则未做改动。