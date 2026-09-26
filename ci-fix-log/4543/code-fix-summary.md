# 修复摘要

## 修复的问题
在 bolt 新版本镜像的新 Dockerfile 中补充 `libquadmath-devel` 依赖，修复 boost/charconv 编译期找不到 `quadmath.h` 导致的构建失败。

## 修改的文件
- `Bigdata/bolt/2d01261/24.03-lts-sp4/Dockerfile`: 在 `dnf install` 依赖列表中新增 `libquadmath-devel`（位于 `gcc gcc-c++ ...` 与 `libstdc++-static glibc-static` 之间）。

## 修复逻辑
- 分析报告根因：bolt 通过 conan 从源码构建 boost 1.85.0，其 `boost/charconv/detail/config.hpp` 在检测到 GCC libquadmath/__float128 支持时会 `#include <quadmath.h>`；但新 Dockerfile 只安装了 `gcc gcc-c++`，未安装提供该头文件的 `libquadmath-devel`，导致 `fatal error: quadmath.h: No such file or directory`，boost 构建失败并连带 `make release` 退出码 2。日志前置征兆 `- GCC libquadmath and __float128 support : no` 也印证了该头文件缺失。
- 修复方式：按分析报告方向 1，在新 Dockerfile 的 `dnf install` 列表中补充 `libquadmath-devel`。该包会被安装到系统 include 路径，使 boost 配置阶段正确识别 libquadmath，charconv 编译通过。
- 包可用性验证：已从 openEuler 24.03-LTS-SP4 官方仓库确认该包存在（x86_64 与 aarch64 均有）：
  - `https://repo.openeuler.org/openEuler-24.03-LTS-SP4/everything/x86_64/Packages/libquadmath-devel-14.3.1-18.oe2403sp4.x86_64.rpm`（另有 12.3.1 版本）
  - `https://repo.openeuler.org/openEuler-24.03-LTS-SP4/everything/aarch64/Packages/libquadmath-devel-12.3.1-110.oe2403sp4.aarch64.rpm`
  与实际基础镜像 `openeuler/openeuler:24.03-lts-sp4` 匹配，两种架构均可用。
- 本次修复不涉及对第三方/上游源文件的正则 patch（分析报告已注明不适用）。

## 潜在风险
无。仅在构建阶段依赖列表中增加一个官方开发包，不改变 bolt 源码、构建脚本或运行逻辑；`libquadmath-devel` 会随基础镜像仓库正常解析安装。该改动仅作用于新增的 `2d01261/24.03-lts-sp4` 镜像，不影响历史版本目录。