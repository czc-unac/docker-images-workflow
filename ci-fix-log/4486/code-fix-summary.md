# 修复摘要

## 修复的问题
修正 openEuler 24.03-LTS-SP4 构建阶段中不存在的 gcc-toolset-14 C++ 包名，解决 `yum install` 报 `Unable to find a match: gcc-toolset-14-c++*` 导致的 dependency-error。

## 修改的文件
- `AI/onnxruntime/1.30.0/24.03-lts-sp4/Dockerfile`: 将 yum 安装列表中的 `gcc-toolset-14-c++*` 改为 `gcc-toolset-14-gcc-c++*`。

## 修复逻辑
分析报告根因指向 `AI/onnxruntime/1.30.0/24.03-lts-sp4/Dockerfile:14` 的 `gcc-toolset-14-c++*` 在 openEuler 24.03-LTS-SP4 仓库中不存在，yum 通配符无匹配直接报错退出。

已从 openEuler 官方仓库实际拉取包清单验证：
- SP4 `everything` 仓库 x86_64 与 aarch64 均提供 `gcc-toolset-14-gcc-c++`（14.3.1-18.oe2403sp4），且不存在 `gcc-toolset-14-c++` 包；同时 `gcc-toolset-14-gcc`、`gcc-toolset-14-binutils` 均存在（`gcc-toolset-14-binutils*` 通配符可匹配到 `gcc-toolset-14-binutils`/`-devel`/`-gold`）。
- 旧版本 `1.22.1/24.03-lts-sp2` 使用相同写法能构建成功，是因为 SP2 仓库确实存在 `gcc-toolset-14-c++-14.2.1-8.oe2403sp2`；该包在 SP4 中已不再提供，改由 `gcc-toolset-14-gcc-c++` 提供，因此这是新增 SP4 镜像触发的真实包名变更问题。

修复采用分析报告方向 1：改用 SP4 正确的 C++ 编译器包名 `gcc-toolset-14-gcc-c++*`，保留显式安装 C++ 编译器的意图（`*` 与其他条目风格保持一致）。该包名与 `gcc-toolset-14-gcc*` 通配符有重叠，但重复安装幂等，无副作用。

## 潜在风险
无。仅修正了一个 yum 包名，未改动构建流程与其它逻辑；包名已在 SP4 x86_64/aarch64 两个架构的官方仓库确认存在。