# 修复摘要

## 修复的问题
构建 bolt 镜像时 boost/1.85.0 的 charconv 组件因缺少 `quadmath.h` 头文件导致编译失败，通过在 Dockerfile 中安装 `libquadmath-devel` 修复。

## 修改的文件
- `Bigdata/bolt/2d01261/24.03-lts-sp4/Dockerfile`: 在 `dnf install` 的依赖行中追加 `libquadmath-devel`，提供 `/usr/include/quadmath.h`。

## 修复逻辑
分析报告根因（方向 1，置信度高）指出：`boost/charconv/detail/config.hpp:32` 会 `#include <quadmath.h>`，而基础镜像 `openeuler/openeuler:24.03-lts-sp4` 未预装提供该头文件的开发包，导致 boost 编译 `fatal error: quadmath.h: No such file or directory`，`make release` 退出码 2。

已完成以下验证：
1. 通过 openEuler 24.03-LTS-SP4 官方仓库确认包真实存在：`libquadmath-devel-12.3.1-110.oe2403sp4.x86_64.rpm` 与 `...aarch64.rpm` 均存在，覆盖 x86_64/aarch64 两种架构。
2. 从上游 `https://raw.githubusercontent.com/bytedance/bolt/2d01261/scripts/install-bolt-deps.sh` 拉取并确认 `install-bolt-deps.sh` 仅克隆 conan-center-index 配方、升级 conan 并配置 remote，不会卸载或覆盖系统包，因此 Dockerfile 中预装的 `libquadmath-devel` 不会被脚本抵消。
3. 采用最小改动，仅在原有 `dnf install` 包列表中新增一个包，未改动任何其他文件或无关代码。

## 潜在风险
无。新增仅为构建期头文件开发包，不改变运行时产物与 bolt 功能；所需磁盘/体积增量可忽略。