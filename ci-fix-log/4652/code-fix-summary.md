# 修复摘要

## 修复的问题
移除 onnxruntime 1.30.0 构建 Dockerfile 中不存在且多余的 yum 包 glob `gcc-toolset-14-c++*`，解决 `No match for argument` 导致的 `yum install` exit code 1 构建失败。

## 修改的文件
- `AI/onnxruntime/1.30.0/24.03-lts-sp4/Dockerfile`: 删除第 14 行 `gcc-toolset-14-c++* \` 参数。

## 修复逻辑
CI 日志显示 `yum install` 步骤中 `gcc-toolset-14-gcc*`、`gcc-toolset-14-binutils*` 均能解析，唯独 `gcc-toolset-14-c++*` 报 `No match for argument: gcc-toolset-14-c++*`，导致整条命令退出码为 1，构建在 `[builder 2/4]` 层失败。

已从 openEuler 24.03-LTS-SP4 官方仓库（`https://repo.openeuler.org/openEuler-24.03-LTS-SP4/everything/x86_64/`）的 `primary.xml` 元数据中确认：该版本不存在任何以 `gcc-toolset-14-c++` 开头的 RPM 包名；C++ 编译器实际包名为 `gcc-toolset-14-gcc-c++`，且该包已被同文件中已有的 `gcc-toolset-14-gcc*` glob 覆盖（glob `*` 可匹配 `-c++` 后缀）。因此 `gcc-toolset-14-c++*` 属于前缀错误且完全冗余的参数，直接删除即可（对应分析报告修复方向 1，置信度高）。

该修复为最小化改动，未触碰其他文件、未重构，也未修改无关的 `FromAsCasing` 警告项。

## 潜在风险
无。删除后 C++ 编译器仍由 `gcc-toolset-14-gcc*` 提供，功能不受影响。