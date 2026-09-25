# 修复摘要

## 修复的问题
为 bolt 2d01261 (openEuler 24.03-LTS-SP4) 镜像补齐 `libquadmath-devel` 构建依赖，解决 boost/1.85.0 charconv 编译时 `quadmath.h: No such file or directory` 错误。

## 修改的文件
- `Bigdata/bolt/2d01261/24.03-lts-sp4/Dockerfile`: 在第一个 `dnf install` 步骤的库依赖行（`libstdc++-static glibc-static`）末尾追加 `libquadmath-devel`。

## 修复逻辑
CI 分析报告指出根因是构建依赖 boost/1.85.0 的 charconv 组件在编译 `to_chars.cpp` 时 `#include <quadmath.h>` 失败，该头文件由 GCC 的 libquadmath 开发包提供。原 Dockerfile 的 `dnf install` 只安装了 `gcc gcc-c++ make cmake ninja-build patch libstdc++-static glibc-static curl python3-pip`，遗漏了 `libquadmath-devel`，导致 conan 构建 boost 失败、`make release` 返回 Error 2。

修复采用分析报告方向 1（置信度高）：在系统包安装阶段补充 `libquadmath-devel`，使 `quadmath.h` 可被找到，boost/1.85.0 的 charconv 组件即可正常编译。

已确认 openEuler 24.03-LTS-SP4 官方仓库存在该包：
- `https://repo.openeuler.org/openEuler-24.03-LTS-SP4/OS/x86_64/Packages/libquadmath-devel-12.3.1-110.oe2403sp4.x86_64.rpm`
- `https://repo.openeuler.org/openEuler-24.03-LTS-SP4/everything/x86_64/Packages/libquadmath-devel-12.3.1-110.oe2403sp4.x86_64.rpm`

同时确认 bolt 仓库 `2d01261` 的 `scripts/install-bolt-deps.sh`（已从上游拉取核对）仅负责下载 conan recipes 并配置本地 remote，不安装任何系统包，因此系统依赖只能在 Dockerfile 中补齐，修复位置正确。

## 潜在风险
改动仅为追加一个官方开发包，不改变构建逻辑与运行镜像内容，无已知副作用。安装 `libquadmath-devel` 后 boost b2 的 libquadmath 探测可能由 no 变为 yes，使 charconv 走原生 float128 实现，属于预期行为改善。

## 验证情况
未涉及正则 patch 外部源文件，无上游正则验证要求。已从上游 openEuler 镜像仓库确认包存在；实际构建需由 CI 复核。