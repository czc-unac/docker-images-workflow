# 修复摘要

## 修复的问题
在 ceph 21.3.0 镜像构建的依赖清单中补装 Protobuf 开发包，解决 CMake 配置阶段 `find_package(Protobuf)` 找不到 `Protobuf_LIBRARIES` / `Protobuf_INCLUDE_DIR` 导致的构建失败。

## 修改的文件
- `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile`: 在首个 `dnf install -y` 依赖清单中新增 `protobuf-devel protobuf-compiler` 两项（位于 `thrift-devel libcap-devel lua-devel` 之后）。

## 修复逻辑
分析报告根因定位为 `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile` 第一个 `dnf install` 步骤遗漏 Protobuf 开发包，导致 ceph 源码 `src/CMakeLists.txt:1029` 的 `find_package(Protobuf ...)` 配置失败并 exit 1。修复方式即补齐 `protobuf-devel`（提供 `Protobuf_LIBRARIES` 与 `Protobuf_INCLUDE_DIR`，含头文件与库）与 `protobuf-compiler`（提供 ceph 构建所需的 `protoc`）。

包名已通过 openEuler 24.03-LTS-SP4 官方仓库元数据核验：拉取 `https://repo.openeuler.org/openEuler-24.03-LTS-SP4/everything/x86_64/repodata/...-primary.xml.zst`，确认存在 `protobuf`、`protobuf-devel`、`protobuf-compiler`、`protobuf-lite`、`protobuf-lite-devel` 等包，版本均为 `25.1-13.oe2403sp4`。同时参照本仓库既有做法（`AI/onnx`、`Others/brpc` 等）使用 `protobuf-devel` + `protobuf-compiler` 组合，包名真实存在、可正常解析，不会引入 dependency-error。

未采纳可选的 BuildKit 警告清理项（`ENV LD_LIBRARY_PATH` 的 `${LD_LIBRARY_PATH:-}` 写法），因其非本次失败根因，遵循最小化修改原则。

## 潜在风险
- 本次修复为补装系统依赖包，改动仅限 21.3.0 的 Dockerfile，不影响其他版本与镜像；但 ceph 21.3.0 是否还依赖其他本次未安装的库无法从现有日志判断，构建通过 Protobuf 环节后仍可能在后续 cmake 检查中报出下一个缺失依赖，需重新构建观察。
- openEuler SP4 仓库中的 Protobuf 为 25.1，若 ceph 21.3.0 对 protobuf 版本上限存在约束，理论上可能出现版本兼容问题；不过日志中的失败是“找不到 Protobuf”而非版本不匹配，补包后可先以实际构建结果为准。