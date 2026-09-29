# 修复摘要

## 修复的问题
删除新增 Dockerfile 中不存在的包名 `gcc-toolset-14-c++*`，解决 `yum install` 报 `No match for argument` 导致的构建失败。

## 修改的文件
- `AI/onnxruntime/1.30.0/24.03-lts-sp4/Dockerfile`: 删除 builder 阶段 `yum install` 列表中的 `gcc-toolset-14-c++*` 一行（原第 14 行）。

## 修复逻辑
- 根因：openEuler 24.03-LTS-SP4 仓库中不存在名为 `gcc-toolset-14-c++` 的包，`gcc-toolset-14-c++*` 通配匹配不到任何包，使整个 yum 事务失败（exit code 1）。
- 验证：已从上游仓库 `https://repo.openeuler.org/openEuler-24.03-LTS-SP4/everything/x86_64/Packages/` 拉取包列表确认，SP4 实际提供的是 `gcc-toolset-14-gcc`、`gcc-toolset-14-gcc-c++`、`gcc-toolset-14-binutils` 等，**不存在** `gcc-toolset-14-c++`。
- 修复依据：C++ 编译器由 `gcc-toolset-14-gcc-c++` 提供，而该包名以 `gcc-toolset-14-gcc` 开头，已被保留的 `gcc-toolset-14-gcc*` 通配匹配覆盖，因此删除无效行即可，无需替换为其他包名。这与分析报告"方向 1：直接移除该行（通过通配 `gcc-toolset-14-gcc*` 获得 C++ 编译器）"一致。

## 潜在风险
无。删除的包名本身不存在，不能提供任何功能；C++ 编译器仍由 `gcc-toolset-14-gcc*` 通配符匹配到的 `gcc-toolset-14-gcc-c++` 提供。runtime 阶段未使用该工具链，不受影响。