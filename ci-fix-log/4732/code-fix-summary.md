# 修复摘要

## 修复的问题
ceph 21.3.0 镜像在 `ninja install` 阶段构建 `cython_rados` Python 模块时失败：pip 执行 `bdist_wheel` 报 `error: invalid command 'bdist_wheel'`，根因是构建环境中缺少 `wheel` 包。

## 修改的文件
- `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile`: 在 pip 安装步骤中追加 `wheel` 包（`python3 -m pip install cython prettytable wheel`）。

## 修复逻辑
本报告不依赖低置信度的 diff 推断，而是直接获取了真实门禁日志定位根因：

- 原始自动升级 PR #4732 的失败日志（x86_64 #4811 / aarch64 #4907）显示 configure 阶段报
  `Could NOT find Protobuf (missing: Protobuf_LIBRARIES Protobuf_INCLUDE_DIR)`（`src/CMakeLists.txt:1029` 的
  `WITH_NVMEOF_GATEWAY_MONITOR_CLIENT` 分支）。该问题已由本分支此前的两次修复提交处理：
  `-DWITH_NVMEOF_GATEWAY_MONITOR_CLIENT=OFF`（禁用引入 Protobuf/gRPC 依赖的可选组件）与
  `-DWITH_MGR_DASHBOARD_FRONTEND=OFF`。
- 加入上述 flag 后重新触发的门禁（fix PR #4745，x86_64 #4858 / aarch64 #4954）已越过 configure 阶段，
  真实第一条错误变为 `ninja install` 安装 Python 绑定时的：
  ```
  Processing /opt/ceph/src/pybind/rados
    Preparing metadata (pyproject.toml): finished with status 'error'
        error: invalid command 'bdist_wheel'
  ...
  CMake Error at src/pybind/rados/cmake_install.cmake:78 (message):
    Failed to build and install cython_rados python module
  ```
  两个架构失败点完全一致，均为缺少 `wheel`（`bdist_wheel` 命令由 `wheel` 包提供，新版 setuptools 已将其拆分）。
- 因此最小修复是在 pip 安装列表中补充 `wheel`。不删除测试、不修改 CI 配置、不触碰 PR 文件列表之外的任何文件。

## 潜在风险
无。仅在构建阶段额外安装 `wheel`，用于 ceph 的 Python 绑定打包；不影响运行镜像及 `entrypoint.sh` 行为。

## 备注
- `check_package_license` 结果为 WARNING（缺少项目级 Copyright 声明文件），非本次构建失败原因，且属仓库级问题，未做修改。
- 修复后应重新触发 x86_64 / aarch64 `check_build` 验证；若仍有后续失败，需以新的首条错误为准继续排查。