# 修复摘要

## 修复的问题
ceph 21.3.0 在 openEuler 24.03-LTS-SP4 上构建时，NVMeOF gateway monitor client 在 v21.3.0 中变为无条件默认开启，而其依赖的 protobuf / gRPC 开发库未在 Dockerfile 中安装，导致 `do_cmake.sh`（cmake 配置阶段）因 `find_package(Protobuf REQUIRED)` / `pkg_check_modules(GRPCPP REQUIRED ...)` 失败而中断。

## 修改的文件
- `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile`: 在 `./do_cmake.sh` 参数中追加 `-DWITH_NVMEOF_GATEWAY_MONITOR_CLIENT=OFF`，显式关闭该可选组件以跳过缺失的依赖。

## 修复逻辑
原始 PR 仅把 `ARG VERSION` 由 `20.3.0` 升级为 `21.3.0`，Dockerfile 其余内容（含 dnf 安装清单）与已验证可构建的 20.3.0 保持一致。对上游源码进行对比后确认根因：

- 已从上游 v21.3.0 获取 `src/CMakeLists.txt` 验证，第 1022 行存在
  `option(WITH_NVMEOF_GATEWAY_MONITOR_CLIENT "build nvmeof gateway monitor client" ON)`，
  **无条件默认 ON**；其后的 `if(WITH_NVMEOF_GATEWAY_MONITOR_CLIENT)` 块要求
  `find_package(Protobuf REQUIRED)` 及 gRPC（cmake config 或 `pkg_check_modules(GRPCPP REQUIRED grpc++)`）。
- 对比上游 v20.3.0 `src/CMakeLists.txt`（第 915-921 行），该选项在旧版本中是**条件默认**：
  仅当存在 `/etc/redhat-release` 或 `/etc/fedora-release` 时才 ON，否则 OFF。openEuler 不含这两个文件，
  因此 20.3.0 天然关闭该组件，构建正常；21.3.0 去掉条件后默认开启，才暴露出本问题。
- 本 Dockerfile 的 `dnf install` 清单中并无 `protobuf-devel` / `grpc-devel`，故默认开启必然配置失败。
- 追加 `-DWITH_NVMEOF_GATEWAY_MONITOR_CLIENT=OFF` 后，该可选目标及其 proto/gRPC 代码生成被跳过，
  与 v20.3.0 在 openEuler 上的实际行为对齐，属最小化改动，不触碰 dnf 依赖与其它构建逻辑。

说明：CI 分析报告置信度为“低”且日志缺失，本修复并非基于日志猜测，而是基于上游源码（v20.3.0 vs v21.3.0）可验证的 option 默认值变更得出，方向与候选方向2（版本升级引入的差异）一致。

## 潜在风险
关闭 `WITH_NVMEOF_GATEWAY_MONITOR_CLIENT` 后，镜像中不再构建 `ceph-nvmeof-monitor-client` 工具。
该工具专用于 NVMe-oF gateway 监控，与本镜像“单节点 ceph-mon 集成验证”的用途无关，不影响
`ceph-mon`、`ceph-osd` 等核心组件及入口脚本逻辑。若后续确需该工具，应另行在 dnf 清单中补充
protobuf/gRPC 开发依赖后再开启。无其它已知风险。