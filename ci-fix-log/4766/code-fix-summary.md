# 修复摘要

## 修复的问题
修正 onnxruntime 1.30.0 在 openEuler 24.03-LTS-SP4 上构建时 `yum install` 失败的问题：将 SP4 已不存在的包名 `gcc-toolset-14-c++*` 替换为 SP4 中的实际包名 `gcc-toolset-14-gcc-c++`。

## 修改的文件
- `AI/onnxruntime/1.30.0/24.03-lts-sp4/Dockerfile`: 第 14 行 `gcc-toolset-14-c++*` → `gcc-toolset-14-gcc-c++`

## 修复逻辑
分析报告本身置信度为“低”且未附日志，但我从 PR #4766 的 CI 评论中定位到失败 job 的真实日志：

- x86_64: https://ci.openeuler.openatom.cn/job/multiarch/job/openeuler/job/x86-64/job/openeuler-docker-images/4877/consoleText
- aarch64: https://ci.openeuler.openatom.cn/job/multiarch/job/openeuler/job/aarch64/job/openeuler-docker-images/4973/consoleText

两个架构均在同一个 `RUN yum install` 步骤（Dockerfile:8）失败，错误完全一致：

```
No match for argument: gcc-toolset-14-c++*
Error: Unable to find a match: gcc-toolset-14-c++*
exit code: 1
```

根因：基础镜像由 `24.03-lts-sp2` 升级到 `24.03-lts-sp4` 后，工具链 C++ 支持包被重命名。
- SP2 仓库中存在包 `gcc-toolset-14-c++`（summary 为 “C++ support for GCC”，提供 `gcc-toolset-14-g++` / `gcc-toolset-14-gcc-g++`）。
- SP4 仓库中该名字已不存在，等价包改名为 `gcc-toolset-14-gcc-c++`（提供 `gcc-toolset-14-g++` / `gcc-toolset-14-gcc-g++`，summary/描述与 SP2 的同名包一致）。

验证过程：
1. 下载并解析 `https://repo.openeuler.org/openEuler-24.03-LTS-SP2/everything/x86_64/repodata/...-primary.xml.zst`，确认存在 `<name>gcc-toolset-14-c++</name>`。
2. 解析 SP4 x86_64 / aarch64 的 primary 元数据，确认均不存在 `gcc-toolset-14-c++`，但存在 `<name>gcc-toolset-14-gcc-c++</name>`。
3. 确认 `gcc-toolset-14-gcc-c++` 提供 `gcc-toolset-14-g++`、`gcc-toolset-14-gcc-g++`，与原包功能等价。

同时，仓库内其他镜像（如 `AI/vllm-cpu/*`）也采用 `gcc-toolset-12-gcc-c++` 的命名方式，本修复与仓库既有约定一致。

## 潜在风险
- 无。`gcc-toolset-14-gcc-c++` 是 SP4 中 C++ 编译器的官方包名，功能与 SP2 的 `gcc-toolset-14-c++` 等价；改动仅替换一个包名，未改变构建流程或其他文件。
- 说明：CI 中另有 `check_package_license` 的 WARNING（仓库缺少项目级 Copyright 声明文件），属警告而非本次构建失败原因，且不在本 PR 变更文件范围内，未做改动。
- 该 `RUN` 之后（如 `source /opt/openEuler/gcc-toolset-14/enable`、`git clone v1.30.0`、`build.sh`）的实际执行结果尚无日志（构建在此步骤即中止），本次仅修复已确证的第一条错误。