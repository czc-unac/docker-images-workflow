# 修复摘要

## 修复的问题
在 bolt 2d01261/24.03-lts-sp4 镜像的 dnf 依赖中补充 `libquadmath-devel`，解决 conan 源码编译 `boost/1.85.0` charconv 时缺失 `quadmath.h` 导致的构建失败。

## 修改的文件
- `Bigdata/bolt/2d01261/24.03-lts-sp4/Dockerfile`: 首个 `dnf install` 步骤的系统包列表中新增 `libquadmath-devel`（与 `libstdc++-static glibc-static` 同行）。

## 修复逻辑
CI 报错直接原因为 `boost/charconv/detail/config.hpp:32` 无法找到 `<quadmath.h>`。该头文件由 GCC 的四精度数学开发包提供，基础镜像仅安装了 `gcc gcc-c++`（仅含运行时 `libquadmath`，不含头文件与开发链接库）。在 Dockerfile 层安装 `libquadmath-devel` 后，boost charconv 的 float128 后端即可找到 `quadmath.h` 完成编译，`make release` 不再因此失败。此修复对应分析报告的“方向 1（置信度：高）”，且为最小改动，未触碰构建脚本或 conan 配置。

包名核验：已从上游 openEuler 官方仓库拉取 `openEuler-24.03-LTS-SP4/OS/x86_64/Packages/` 目录清单，确认存在 `libquadmath-devel-12.3.1-110.oe2403sp4.x86_64.rpm`（同时存在运行时包 `libquadmath-12.3.1-110.oe2403sp4.x86_64.rpm`），故拟新增包名真实存在，不会引入 `dnf install` 新失败。

说明：本修复仅在 Dockerfile 层补充系统依赖包，不涉及通过正则 patch 第三方/上游源文件，因此无需上游正则匹配验证。

## 潜在风险
无。新增包为 GCC 配套开发组件，不影响既有依赖解析与运行时行为（该镜像为构建型镜像，无额外镜像体积敏感约束）。