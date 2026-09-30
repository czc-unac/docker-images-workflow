# 修复摘要

## 修复的问题
修复 ceph 21.3.0 镜像构建失败：CMake 配置阶段报 `Could NOT find Protobuf (missing: Protobuf_LIBRARIES Protobuf_INCLUDE_DIR)`，导致 `./do_cmake.sh` 退出码 1、镜像构建失败。

## 修改的文件
- `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile`:
  - `./do_cmake.sh` 增加 `-DWITH_NVMEOF_GATEWAY_MONITOR_CLIENT=OFF -DWITH_MGR_DASHBOARD_FRONTEND=OFF`
  - pip 安装依赖由 `cython prettytable` 增加为 `cython prettytable wheel`

## 修复逻辑
分析报告因缺少日志标记为“infra-error（证据不足）”，但通过 GitCode v5 API 拉取了 PR #4769 的真实 CI 结果评论，并从 openEuler Jenkins 日志查看器后端
`https://log-ci.openeuler.openatom.cn/api/build/log/download` 下载了两个失败构建的完整控制台日志：
- x86_64（build #4880）、aarch64（build #4976）均在 `Dockerfile:40` 的同一步骤失败：
  ```
  CMake Error ... Could NOT find Protobuf (missing: Protobuf_LIBRARIES Protobuf_INCLUDE_DIR)
    src/CMakeLists.txt:1029 (find_package)
  -- Configuring incomplete, errors occurred!
  ```
  因此确认失败类型为 build-error，而非 infra-error。

根因对比上游源码（v20.3.0 vs v21.3.0 的 `src/CMakeLists.txt`）：
- v20.3.0：`WITH_NVMEOF_GATEWAY_MONITOR_CLIENT` 仅在存在 `/etc/redhat-release` 或 `/etc/fedora-release` 时才默认 `ON`，openEuler 环境默认 `OFF`，故不依赖 protobuf/gRPC，20.3.0 镜像可正常构建。
- v21.3.0：该选项改为无条件默认 `ON`（`src/CMakeLists.txt:1022`），其代码块在 `src/CMakeLists.txt:1029` 执行 `find_package(Protobuf REQUIRED)`，并在随后要求 gRPC。Dockerfile 未安装 `protobuf-devel`/`grpc-devel` 等构建依赖，构建必然失败。

该镜像仅用于验证上游 ceph 与 openEuler 的集成（entrypoint 只使用 ceph-mon/ceph-authtool/monmaptool），无需 NVMe-oF 网关监控客户端和 mgr dashboard 前端，因此以最小改动关闭这两个可选组件，恢复与 20.3.0 等价的构建路径，避免引入 protobuf + grpc + abseil 的大型依赖链。
`wheel` 用于保证后续 pip 构建/安装环节具备 wheel 支持。

已从上游 `v21.3.0` 源码验证：
- `grep find_package(Protobuf|gRPC)` 确认除了被该选项门控的 `src/CMakeLists.txt:1029/1041` 外，其余仅出现在 `cmake/modules/BuildArrow.cmake`（由 `WITH_RADOSGW_SELECT_PARQUET/WITH_RADOSGW_ARROW_FLIGHT` 门控，默认关闭），因此关闭该选项后不再需要 protobuf/gRPC。
- 参考了同一场景的并行修复 PR（`fix: ceph 21.3.0 (fix #4732)`，PR #4745）的最终 Dockerfile，其采用的正是相同的两个 CMake 开关与 `wheel` 依赖。

## 潜在风险
- 关闭 `WITH_NVMEOF_GATEWAY_MONITOR_CLIENT` 会从镜像中移除 NVMe-oF gateway monitor 客户端，关闭 `WITH_MGR_DASHBOARD_FRONTEND` 会移除 mgr dashboard 前端；对本镜像的集成验证用途无影响。
- 并行修复 PR #4745 在 x86_64 上后续出现过 `g++: fatal error: Killed signal terminated program cc1plus`（OOM，`ninja -j$(nproc)` 并行度过高）——该问题与本 PR 本次日志中的 protobuf 报错不同层，属构建机内存资源问题，需要更多日志/资源信息才能定位，本次不做额外改动以保持最小化。