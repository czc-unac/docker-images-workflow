# 修复摘要

## 修复的问题
ceph 21.3.0 镜像构建失败：CMake 配置阶段报 `Could NOT find Protobuf`，原因是 21.3.0 新增的 NVMe-oF Gateway Monitor Client 组件强制依赖 Protobuf / gRPC，而 Dockerfile 未安装相应构建依赖。

## 修改的文件
- `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile`: 在 `dnf install` 依赖列表中新增 `protobuf-devel protobuf-compiler grpc-devel grpc-plugins jq`（仅 1 行）。

## 修复逻辑

### 1. 获取并定位真实失败根因（分析报告要求的确认项 1、5 已完成）
分析报告因日志缺失无法定位。本 agent 通过 GitCode v5 API 读取 PR #4870 的评论，提取到真实失败构建链接（原分析脚本的 Jenkins URL 正则只匹配 `ci.openeuler.openatom.cn`，实际域名为 `log-ci.openeuler.openatom.cn`，故日志抓取为空）：
- x86-64: `https://log-ci.openeuler.openatom.cn/job/multiarch/openeuler/x86-64/openeuler-docker-images/4984/`
- aarch64: `https://log-ci.openeuler.openatom.cn/job/multiarch/openeuler/aarch64/openeuler-docker-images/5080/`

两架构日志末尾均出现同一致命错误（非分析报告猜测的 license / UndefinedVar）：
```
CMake Error at /usr/share/cmake/Modules/FindPackageHandleStandardArgs.cmake:233 (message):
  Could NOT find Protobuf (missing: Protobuf_LIBRARIES Protobuf_INCLUDE_DIR)
Call Stack (most recent call first):
  ...
  src/CMakeLists.txt:1029 (find_package)
#12 ERROR: process "git clone -b v${VERSION} ... && ./do_cmake.sh ..." did not complete successfully: exit code: 1
```
`UndefinedVar`、`check_package_license` 在日志中仅为 warning，并非失败原因（`check_build` 才是 FAILED 的 job）。

### 2. 核对上游依赖（分析报告要求的确认项 4 已完成）
- 已由上游实际源文件验证：从 `https://raw.githubusercontent.com/ceph/ceph/v21.3.0/src/CMakeLists.txt` 拉取，确认新增的 `WITH_NVMEOF_GATEWAY_MONITOR_CLIENT`（默认 ON）块内执行 `find_package(Protobuf REQUIRED)`，随后通过 `find_package(gRPC CONFIG QUIET)` / `pkg_check_modules(GRPCPP REQUIRED ... grpc++)` 以及 `find_program(grpc_cpp_plugin REQUIRED)` 强制要求 gRPC 开发包与 protoc 插件。
- 已由上游 `ceph.spec.in`（v21.3.0）验证：`BuildRequires: grpc-devel`（第 259 行）、`BuildRequires: protobuf-devel` / `protobuf-compiler`（第 397-398 行）、`BuildRequires: jq`（第 340 行，make check 相关）。
- 已核对 openEuler 24.03-LTS-SP4 官方仓包元数据（primary.sqlite），确认 x86_64 与 aarch64 均存在且版本满足：`protobuf-devel/protobuf-compiler 25.1`、`grpc-devel/grpc-plugins 1.60.0`、`jq 1.8.0`。其中 `grpc-plugins` 提供 `/usr/bin/grpc_cpp_plugin`（`grpc-devel` 并不依赖它，必须显式安装）；`protobuf-devel` 提供 `cmake(protobuf)`/`pkgconfig(protobuf)` 与 `libprotobuf`/`protoc`。
- 上游 `v21.3.0` tag 经 `git ls-remote` 确认真实存在，排除分析报告候选根因 C。

### 3. 最小化修复
仅在配置阶段补齐触发失败的新增构建依赖，不改动 clone 逻辑、entrypoint 逻辑或其他文件。`jq` 因在失败构建日志中实际出现 `jq: command not found` 且为上游 BuildRequires，一并补齐以降低后续阶段再次失败的风险。

> 注：分析报告候选根因 A（缺少 Copyright/SPDX 头）与 B（ENV 自引用 UndefinedVar）经真实日志证实均非本次失败原因，按最小化原则未做改动；且新增项目级 Copyright 文件会违反"禁止创建新文件"约束。

## 潜在风险
- 修复的是配置阶段第一个致命缺包。配置通过后编译阶段仍可能暴露 21.3.0 新增的其他问题（如新版本 API 变更、编译器告警），需以重跑日志为准。
- 新增依赖会增大构建镜像体积与下载时间（仅构建阶段，不影响最终运行镜像的运行时依赖）。
- `UndefinedVar`（`ENV LD_LIBRARY_PATH=...:$LD_LIBRARY_PATH` 第 47 行）与文件末尾缺少换行仍为 warning/格式问题，本次未处理，不影响构建结果。